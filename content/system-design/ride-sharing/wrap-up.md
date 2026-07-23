---
title: 'Wrap-Up'
weight: 5
type: docs
---

## Interview Tips

- **Lead with the location firehose.** The first number to write on the whiteboard is **1.25M writes/s** (5M drivers ÷ 4 s). This single constraint justifies Kafka, an in-memory geo-index, and the entire location pipeline. Interviewers look for candidates who identify the dominant constraint before diving into features.

- **Name your geospatial index and defend it.** GeoHash, QuadTree, and H3 are all valid starting points — the point is to compare them. Favour H3 for uniform cell area, 6-neighbour k-ring, and natural fit for both geo-queries and surge pricing aggregation. If pushed, acknowledge that GeoHash (Redis-native) is a simpler first step with lower operational complexity.

- **Walk through the matching cascade explicitly.** Say: "find candidates via geo-index → rank by ETA → offer with 15 s TTL → serial accept/reject cascade → radius expansion on failure." Mention the Redis NX offer-lock as your distributed mutex to prevent double-dispatch.

- **Surge pricing is supply/demand per cell.** State: "count open requests (demand) and available drivers (supply) per H3 resolution-6 cell every 30 s; piecewise-linear formula; lock multiplier at booking." This shows you've covered the full request lifecycle, not just the happy path.

- **Name WebSocket as the tracking transport — and say why.** Polling at 4 s for 333k riders = 83k requests/s of pure waste. WebSocket push + Redis pub/sub fan-out is the right answer. Mention pub/sub: one location event published once, consumed by the gateway subscribed to that ride's channel.

- **Idempotency for payment is non-negotiable.** Always say: "idempotency key = ride_id; outbox pattern commits the payment intent atomically with the ride status update; the payment gateway deduplicates on its side." This is a required senior signal in any payment-adjacent design.

- **Common follow-ups to rehearse:**
  - How does the system handle a driver going offline mid-trip?
  - How do you prevent a "no drivers" situation from becoming a permanent state? (surge pricing as a supply signal)
  - How does radius expansion affect rider experience? (ETA shown to rider updates as radius widens)
  - How would you add ride pooling? (assign multiple riders per driver; matching constraint changes from 1:1 to 1:N)
  - What if a single H3 cell becomes a hotspot (stadium event)? (geo-index keys are per-cell, so no single Redis key is hot; surge pricing attracts supply)
  - What happens if Kafka goes down? (location ingestion buffers in ingestion nodes up to a threshold; geo-index becomes stale; matching degrades gracefully but rides continue)

---

## Resiliency

- **Location ingestion:** Kafka buffers burst writes and provides a durable replay log. If the Redis geo-index fails, the Location Consumer rebuilds it from the last 15 minutes of Kafka history (enough to recover current positions for all 5M drivers). See {{% relref "/design-concepts/messaging/kafka" %}}.

- **Geo-index staleness:** The heartbeat sweeper (every 30 s) removes drivers silent for > 60 s from the available set, ensuring stale entries never permanently pollute the index. Location consumers are idempotent — replaying a Kafka message is safe.

- **Matching Service:** stateless and horizontally scaled; any node can handle any match request. Offer locks in Redis auto-expire, so a crashed Matching Service never permanently blocks a driver from receiving new offers.

- **Ride Service:** the Ride DB is replicated (3× minimum, quorum reads for billing consistency). The outbox pattern ensures payment events survive Payment Service downtime of arbitrary duration without data loss.

- **WebSocket Gateway:** connections are stateful, so node crashes drop sessions. Mitigation: fast client reconnect (< 2 s), consistent-hash routing so reconnects hit the same node, last-known-location replay on reconnect. See {{% relref "/design-concepts/specialized/websocket-at-scale" %}}.

- **Payment:** outbox + idempotency key = at-least-once delivery with exactly-once semantics at the payment gateway. Payment failures enter a retry queue with exponential backoff; after N failures the event moves to a dead-letter queue for human review. See {{% relref "/design-concepts/distributed/idempotency" %}} and {{% relref "/design-concepts/specialized/payment-systems" %}}.

- **Multi-region:** ride-sharing is inherently city-local — a ride in New York has no dependency on servers in London. Each city runs its own Location Service, Matching Service, and Ride DB in a regional data centre; cross-region traffic is limited to driver/rider profile reads and global analytics. A regional outage affects only rides in that region. See {{% relref "/design-concepts/distributed/multi-region-design" %}}.

- **Hotspot handling:** a large venue (stadium, concert) creates a geographic hotspot. Because the geo-index shards by H3 cell, no single Redis key bears the full query load. Surge pricing acts as a self-healing mechanism: a high demand/supply ratio raises the multiplier, attracting drivers into the area. See {{% relref "/design-concepts/storage/hotspot-problems" %}}.

- **Backpressure:** the location ingestion rate limiter (one update per 2 s per driver) bounds the Kafka publish rate. If Kafka is slow, ingestion nodes apply back-pressure at the HTTP layer (429 Too Many Requests) rather than unbounded buffering. See the back-pressure concept in {{% relref "/design-concepts/reliability/back-pressure" %}}.

---

## Observability

### SLIs and Key Metrics

| Signal | What to Measure | Alert Threshold |
|---|---|---|
| **Location freshness** | Kafka consumer lag (seconds) | > 10 s |
| **Match success rate** | % ride requests that result in a driver assignment | < 85% over 5 min |
| **Match latency** | Time from ride request to driver acceptance (p50 / p99) | p99 > 30 s |
| **Offer acceptance rate** | Accepts / (accepts + rejects + timeouts) | < 60% |
| **WebSocket connections** | Active connections per gateway node | > 12k (near saturation) |
| **Payment success rate** | Successful gateway charges / completed rides | < 99.5% |
| **Payment latency** | Time from trip end to charge settled (p99) | > 60 s |
| **Geo-index query latency** | Redis SMEMBERS p99 for k-ring batch | > 5 ms |
| **Surge distribution** | Histogram of active multipliers across all H3 cells | > 20% cells at 2× (demand shock) |
| **Outbox depth** | Number of unprocessed outbox rows | > 1,000 rows |

### Logging

- **Full logging:** every ride state transition (status change + timestamp), every offer dispatch and response, every payment event including gateway response codes.
- **Sampled logging (1–5%):** driver GPS pings. Full GPS logging at 1.25M/s generates > 3 TB/day of log volume — sample for debugging and route the full stream to Kafka / columnar analytics storage.
- **Structured logs (JSON):** mandatory fields on every log line: `ride_id`, `driver_id`, `rider_id`, `region`, `service_name`, `trace_id`. This enables cross-service correlation without a full-text scan.

### Distributed Tracing

Propagate a `trace_id` (= `ride_id` + span suffix) through the full causal chain:

- **Match path:** API Gateway → Ride Service → Matching Service → Geo-Index query → ETA Service → Push Notification → Driver response → Ride Service.
- **Payment path:** Ride Service → Outbox → Outbox Worker → Payment Service → Payment Gateway → Ride DB update.

Trace-level detail allows attributing tail latency to a specific tier (e.g. "ETA Service p99 is 3 s, dominating match latency").

### Dashboards

- **Operational:** Kafka consumer lag per partition, Redis memory utilisation, WebSocket connections per gateway node, active rides by region, Outbox Worker depth.
- **Business:** rides/hour by city, match success rate, average surge multiplier, payment error rate, driver earnings per hour.
- **Geo heatmap:** live driver density and surge multiplier overlaid on a city map using H3 cells at resolution 6. Refreshed every 30 s (aligned with the Surge Service cadence).
- **Error budget:** SLO burn-down chart for the 99.99% matching availability target and 99.9% payment availability target.

---

## Concepts Used

- {{% relref "/design-concepts/specialized/location-indexing" %}} — geo-index design and the location-update ingestion pipeline
- {{% relref "/design-concepts/specialized/geohash" %}} — GeoHash encoding, radius-query algorithm, and comparison with H3
- {{% relref "/design-concepts/specialized/quadtree" %}} — quadrant-recursive spatial index, evaluated and compared against H3
- {{% relref "/design-concepts/messaging/kafka" %}} — absorbing the 1.25M/s location-update firehose; replay for geo-index rebuild
- {{% relref "/design-concepts/networking/realtime-transport" %}} — WebSocket vs SSE vs polling for live trip tracking
- {{% relref "/design-concepts/specialized/websocket-at-scale" %}} — WebSocket Gateway scaling, sticky routing, pub/sub fan-out
- {{% relref "/design-concepts/distributed/idempotency" %}} — offer locks, payment idempotency key, outbox pattern
- {{% relref "/design-concepts/specialized/payment-systems" %}} — payment gateway integration, PCI boundary, settlement flow
- {{% relref "/design-concepts/storage/key-value-stores" %}} — Redis as geo-index, offer store, surge cache, and pub/sub broker
- {{% relref "/design-concepts/storage/hotspot-problems" %}} — geo-cell sharding distributes query load; surge pricing as a supply signal
- {{% relref "/design-concepts/distributed/multi-region-design" %}} — city-level regional deployment; minimal cross-region dependency
- {{% relref "/design-concepts/reliability/back-pressure" %}} — rate limiting on location ingestion to bound Kafka write pressure
