---
title: 'Wrap-Up'
weight: 5
type: docs
---

## Interview Tips

**Open by naming the fan-out problem.** The very first thing to say after clarifying requirements is: "The fundamental trade-off here is fan-out on write vs fan-out on read. Let me work through the math to show why neither extreme works at scale." This immediately signals you understand the core difficulty.

**Show the math before proposing the hybrid.** Compute fan-out write QPS for a celebrity (100 M followers × 1 tweet = 100 M writes → ~100 s at 1 M writes/s) and fan-out read QPS for a typical user (200 following × 34,700 reads/s = 7 M Cassandra queries/s). Only after demonstrating both extremes fail does the hybrid model feel inevitable rather than memorised.

**Drive the threshold discussion.** The interviewer will ask "how do you pick T?" Work through the table: at T = 10,000, the per-tweet fan-out is bounded at 10,000 writes; the average non-celebrity account has ~200 followers, so effective fan-out QPS is ~348,000 /s — manageable. State the trade-off explicitly: a higher T means more accounts on the cheaper write path but more read overhead at merge time.

**Explain the Redis sorted set choice.** Why sorted set? O(log n) insert, O(log n + k) range query, and using the Snowflake tweet_id as score eliminates a separate timestamp field. Contrast with a plain list (O(n) trim) or a hash (no ordering). This is the kind of concrete data-structure justification that distinguishes a senior answer.

**Volunteer tweet deletion and edit design.** Most candidates forget this. For deletion: soft-delete in Cassandra + read-time filter (cheap) vs active ZREM fan-out (consistent but expensive — reserve for legal takedowns). For edits: immutable edit-chain approach avoids distributed cache invalidation entirely.

**Common follow-ups to rehearse:**
- "What happens when a user with 9,999 followers gains their 10,000th?" — transition is eventually consistent; background job flips the flag.
- "How do you serve both chronological and ranked timelines from the same infrastructure?" — same candidate pool (top 500), different Stage 2 (identity function vs ML model).
- "What if Kafka falls behind on fan-out?" — timeline staleness bounded by lag; no data loss since tweets are in Cassandra; horizontal scale-out of workers.
- "How would you add notification fan-out?" — a separate Kafka consumer group reading the same `tweet.created` topic and routing to the notification system (link to [Notification Fan-out]({{% relref "/design-concepts/specialized/notification-fanout" %}})).

---

## Resiliency

**Stateless app tier:** every Write Service and Read Service node holds no local state; any node can serve any request. A node failure is invisible to clients behind the load balancer.

**Kafka as the fan-out buffer:** the tweet is durable in Cassandra before fan-out begins. A total fan-out worker outage (e.g. deployment) produces a stale-but-correct timeline; workers drain the backlog when they recover. Kafka's retention (typically 7 days) prevents message loss even across multi-day outages.

**Redis replication and cold rebuild:** each Redis primary has one replica. If a primary fails, the replica promotes in seconds. On a full cluster failure, the Read Service falls back to cold timeline rebuild from Cassandra — expensive but correct. Cold rebuild on ~1 % of requests is within Cassandra capacity.

**Cassandra multi-region replication:** tweet data is replicated across at least two geographic regions using Cassandra's `NetworkTopologyStrategy`. A regional outage leaves the other region fully readable. Writes in the outaged region queue in Kafka and replay on recovery. See [Leader-Based Replication]({{% relref "/design-concepts/replication/leader-based-replication" %}}).

**Celebrity cache recovery:** if the celebrity Redis cache is cold (new cluster, eviction), a single celebrity tweet fetch will miss. The Read Service falls back to querying `tweets_by_user` in Cassandra for the celebrity's recent tweets — exactly the cold-rebuild path. After one successful fetch the celebrity cache is warm again.

**Idempotent fan-out:** fan-out workers use Kafka consumer offsets with at-least-once delivery. Duplicate `tweet.created` events are safe: `ZADD` with the same score/member is idempotent in Redis (same entry, same score — no double entry). See [Idempotency]({{% relref "/design-concepts/distributed/idempotency" %}}).

**Back-pressure on fan-out:** Kafka consumer group lag is monitored. When lag exceeds a threshold (e.g. 1 M messages, ~30 min of tweets), an alert fires and the operator scales out fan-out worker pods. Because workers are stateless, scale-out is instant. See [Back-pressure]({{% relref "/design-concepts/reliability/back-pressure" %}}).

**Graceful degradation:** if the Ranking Service is down, the Read Service falls back to chronological order (skip Stage 2). If engagement counters are unavailable, tweet bodies are returned without counts. Partial degradation is always preferable to a hard failure on the read path.

---

## Observability

### SLIs / Key Metrics

| Signal | Target | Alert threshold |
|---|---|---|
| Timeline read p99 latency | < 100 ms | > 150 ms for 5 min |
| Timeline read availability | > 99.99 % | < 99.9 % over 1 min |
| Fan-out Kafka consumer lag | < 100 K messages | > 1 M messages |
| Redis timeline cache hit ratio | > 95 % | < 90 % for 10 min |
| Fan-out write QPS | steady ~350 K/s | spike > 1.5 M/s |
| Celebrity cache freshness | < 5 s after tweet | any celebrity tweet older than 30 s in cache |
| Cassandra p99 read latency | < 10 ms | > 25 ms for 5 min |
| Media transcoding queue depth | < 1 min lag | > 10 min lag |

### Logging

- **Tweet write events:** log every `tweet.created` and `tweet.deleted` fully — these are audit-critical.
- **Timeline reads:** sample at 0.1 % (logging 34,700 reads/s in full is ~3 GB/min of logs — cost-prohibitive). Log all 5xx errors in full.
- **Fan-out decisions:** log the celebrity/regular classification for each tweet (useful for threshold tuning).
- **Media upload events:** log `media.uploaded` → `media.ready` latency (transcoding SLA).

### Tracing

Propagate a `trace_id` through: API gateway → Write/Read Service → Kafka event header → Fan-out Worker → Redis → Cassandra. This allows attribution of tail latency to a specific tier. A p99 timeline read at 95 ms should be traceable to "85 ms in Cassandra multiget, 10 ms in ranking service."

### Dashboards

- **Timeline health:** QPS by endpoint, p50/p99/p999 latency, error rate — per region.
- **Fan-out throughput:** Kafka lag per consumer group, fan-out write QPS, celebrity vs regular ratio.
- **Cache efficiency:** Redis hit ratio per shard, memory usage, eviction rate.
- **Hot-account monitor:** top 20 accounts by fan-out QPS; detect unexpected threshold crossings.
- **Media pipeline:** transcoding queue depth, per-format success rate, CDN origin pull rate.
- **Error budget burn-down:** rolling 28-day SLO burn against the 99.99 % availability target.

---

## Concepts Used

- {{% relref "/design-concepts/specialized/notification-fanout" %}} — fan-out pattern; out-of-scope sibling system for push notifications
- {{% relref "/design-concepts/storage/key-value-stores" %}} — Redis as timeline cache and celebrity cache
- {{% relref "/design-concepts/messaging/kafka" %}} — decoupling tweet writes from async fan-out workers
- {{% relref "/design-concepts/storage/object-storage" %}} — immutable media blob storage at petabyte scale
- {{% relref "/design-concepts/networking/cdn" %}} — edge delivery of media; offloads origin bandwidth
- {{% relref "/design-concepts/ml/recommendation-systems" %}} — two-stage ranking pipeline for ML timeline
- {{% relref "/design-concepts/ml/feature-store" %}} — real-time engagement signals for ranking
- {{% relref "/design-concepts/storage/hotspot-problems" %}} — celebrity accounts as write/read hotspots; hybrid threshold
- {{% relref "/design-concepts/storage/wide-column-stores" %}} — Cassandra for tweet store and social graph
- {{% relref "/design-concepts/storage/caching-patterns" %}} — cache-aside, TTLs, cold rebuild, ranked result caching
- {{% relref "/design-concepts/data/batch-vs-streaming" %}} — precomputation vs lazy evaluation (fan-out on write vs read)
- {{% relref "/design-concepts/distributed/idempotency" %}} — idempotent ZADD; at-least-once Kafka delivery
- {{% relref "/design-concepts/reliability/back-pressure" %}} — fan-out worker lag alerting and scale-out
- {{% relref "/design-concepts/replication/leader-based-replication" %}} — Cassandra multi-region replication
- {{% relref "/design-concepts/api/pagination" %}} — cursor-based timeline pagination using Snowflake score
