---
title: 'Wrap-Up'
weight: 5
type: docs
---

## Interview Tips

- **Open with the three sub-systems and their scale asymmetry.** The problem is not one system — it is three tightly integrated systems with radically different scale profiles: routing is 700 req/s CPU-heavy; tile serving is 580 K req/s bandwidth-heavy; GPS ingestion is 2 M events/s write-heavy. Name this upfront and the interviewer knows you see the full picture.

- **Anchor routing in algorithms, not buzzwords.** Walk through Dijkstra → A* → Contraction Hierarchies in order, explaining *why* each step is needed. Show that A* with haversine is admissible and where it falls short. Explain that CH preprocessing is offline and shortcuts are topology-only — live traffic is a weight overlay at query time. This arc signals genuine depth.

- **Drive the tile caching conversation proactively.** Tiles are pre-rendered, content-addressed by `(zoom, x, y)`, and have near-infinite cache hit ratios at low zoom. Bring up the quadtree/pyramid structure, CDN TTL strategy, and ETag-based conditional requests without being asked.

- **Traffic pipeline is a streaming systems question.** GPS probes → Kafka → Flink window → Redis edge weights → reroute notification. Know each stage's throughput, latency contribution, and failure mode. When the interviewer asks "how do you propagate traffic?", this pipeline is the full answer.

- **Common follow-ups to rehearse:**
  - "How do you handle a major road closure?" → structural graph change requires incremental CH preprocessing plus tile invalidation; interim: add a max-cost edge for the closed segment.
  - "How accurate is your ETA?" → three-layer estimation (free-flow + historical multiplier + ML correction); confidence interval from historical variance.
  - "How do you handle offline maps for a 10 GB region?" → delta sync protocol with content hashes; binary tile diffs; resumable range downloads.
  - "What if a GPS feed from a city goes down?" → routing falls back to historical multipliers; ETA degrades gracefully; no rerouting until feed recovers.
  - "How do you prevent location tracking?" → rotating session tokens, strip device ID at ingest, no persistent GPS trace storage beyond 24 h Kafka retention.

- **State trade-offs explicitly.** CH: millisecond queries vs. multi-hour preprocessing. A*: near-optimal for city routing vs. inadequate for continental. CDN: 95 % cache hit ratio vs. stale tile window on map edits. These are the conversation the interviewer wants to have.

## Resiliency

### Routing Tier
- **In-memory redundancy:** multiple routing servers each hold the full graph. A server crash is transparent; the load balancer routes to healthy replicas. No shared state between routing servers.
- **Graceful graph reload:** rolling restart — one server loads the new CH graph while others serve traffic; LB drains old instance only after new instance passes health check.
- **Redis fallback:** if Redis (live edge weights) is unavailable, routing uses `base_weight` × historical multiplier. Route quality degrades slightly; no outage.

### Tile Serving
- **CDN as resilience layer:** with a 95 % CDN hit ratio, origin servers can absorb a sustained outage of up to 5 minutes without client impact (CDN serves stale with `stale-while-revalidate`). Object storage (e.g. S3) is independently highly available (99.999999999 % durability).
- **Multi-region object storage replication:** tile data replicated across at least two geographic regions; CDN origin falls back to secondary region automatically.

### Traffic Pipeline
- **Kafka replication:** each Kafka partition has 3 replicas across brokers; leader failover is automatic and transparent to producers.
- **Flink checkpointing:** Flink stream jobs checkpoint state to distributed storage every 30 s. On a task-manager failure, job restarts from the last checkpoint and replays Kafka from the saved offset — at-least-once processing; Redis `SETEX` writes are idempotent.
- **GPS probe data loss:** individual probe loss is inconsequential (hundreds of probes per edge per window; one missing event doesn't shift the median). Kafka consumer lag is monitored; if lag exceeds 60 s, the system alerts and scales up Flink tasks.

### Offline Maps
- **Client-side fallback:** navigation continues entirely on cached tiles and graph during connectivity loss. ETA recalculates from the downloaded graph + last known historical speed profiles.
- **Delta sync failure:** if the delta service is unreachable on reconnect, the client retries with exponential backoff. Navigation remains functional on the cached version until sync succeeds.

## Observability

### SLIs and Key Metrics

| Subsystem | SLI | Target |
|---|---|---|
| Routing | p99 route computation latency | < 500 ms |
| Routing | ETA accuracy (MAPE on completed journeys) | < 15 % |
| Tile serving | CDN cache hit ratio | > 95 % |
| Tile serving | p99 tile delivery latency to client | < 50 ms |
| GPS ingestion | Kafka consumer lag (Flink) | < 30 s |
| Traffic propagation | p95 time from probe to Redis weight update | < 15 s |
| Rerouting | p99 reroute notification latency | < 5 s |
| Offline sync | Delta sync success rate | > 99.5 % |

### Logging and Tracing

- **Trace ID propagation:** every route request carries a trace ID from client SDK → load balancer → routing service → CH query → ETA service → response. Distributed trace (OpenTelemetry) attributes latency to: graph load, CH bidirectional Dijkstra, Redis weight fetch, ETA ML inference, turn-step generation.
- **Tile request logs:** sampled at 1 % (580 K/s × 1 % = 5 800/s still meaningful for analysis). Log: tile key, CDN PoP, cache hit/miss, response time, client region.
- **GPS probe logs:** not individually logged (2 M/s); Flink emits per-window aggregate metrics: probes processed, edges updated, median speed changes, reroute triggers fired.

### Alerting

- **Routing p99 > 300 ms for 2 minutes** → page routing team; likely CH graph reload contention or Redis latency spike.
- **CDN cache hit ratio < 90 %** → investigate tile invalidation storm or CDN misconfiguration.
- **Kafka consumer lag > 60 s** → Flink is behind; scale up tasks or investigate map-matching bottleneck.
- **ETA MAPE > 25 % on rolling 1-hour window** → traffic model degraded; check Redis weight freshness and ML model health.
- **Reroute notification p99 > 10 s** → WebSocket fan-out bottleneck or reroute service overload.

### Dashboards

1. **Routing health:** route QPS, p50/p99 latency per region, CH query node expansion count, ETA accuracy (trailing 1-hour MAPE).
2. **Tile serving:** CDN hit/miss ratio by zoom level and region, origin QPS, tile invalidation queue depth, CDN egress TB/day vs. budget.
3. **Traffic pipeline:** Kafka topic lag by partition, Flink job throughput (probes/s, edges updated/s), reroute events/s, Redis memory usage and eviction rate.
4. **GPS probe coverage:** heatmap of probe density by geohash cell; cells with < 3 probes/minute are flagged as "dark" (insufficient data for reliable speed estimates).

## Concepts Used

- {{% relref "/design-concepts/storage/object-storage" %}} — durable tile storage and CH graph persistence
- {{% relref "/design-concepts/networking/cdn" %}} — global tile delivery at 580 K req/s
- {{% relref "/design-concepts/messaging/kafka" %}} — GPS probe ingestion buffer at 2 M events/s
- {{% relref "/design-concepts/data/stream-processing-engines" %}} — Flink window-based probe aggregation to edge weights
- {{% relref "/design-concepts/specialized/quadtree" %}} — tile pyramid spatial indexing and viewport lookup
- {{% relref "/design-concepts/specialized/geohash" %}} — POI spatial prefix indexing and GPS partition routing
- {{% relref "/design-concepts/specialized/location-indexing" %}} — map matching GPS probes to road edges
- {{% relref "/design-concepts/storage/caching-patterns" %}} — CDN tile caching, Redis edge-weight cache, stale-while-revalidate
- {{% relref "/design-concepts/storage/hotspot-problems" %}} — hot geohash partitions in Kafka, high-density tile cells
- {{% relref "/design-concepts/ml/ml-platform" %}} — ETA correction model training, feature store, low-latency inference
- {{% relref "/design-concepts/specialized/websocket-at-scale" %}} — push-based reroute notifications to navigating clients
