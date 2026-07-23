---
title: 'Wrap-Up'
weight: 4
type: docs
---

## Interview Tips

- **Lead with the data structure choice.** The entire design pivots on one insight: a Redis sorted set answers `ZADD`, `ZREVRANK`, and `ZREVRANGE` in O(log n) and sub-millisecond — no SQL `COUNT(*) WHERE score > ?` can compete at 250,000 reads/s. State this explicitly and explain the skip-list span-counter trick if pressed (that is what makes `ZREVRANK` O(log n) rather than O(n)).
- **State the scale axes up front.** 50 M players × 25,000 writes/s × 250,000 reads/s × segmented boards gives your interviewer four dimensions to explore. Name each axis early, then show how the design addresses it: Kafka for write decoupling, read replicas for read scale-out, sharding for memory overflow, histogram for cross-shard rank.
- **Drive the tie-breaking discussion.** Interviewers love "how do you break ties?" as a follow-up because it shows you have thought past the happy path. Walk through the float score trick: `composite = raw_score + (1 - ts / MAX_TS)`. Mention the IEEE 754 precision limit and the integer-composite fallback for high-score games.
- **Volunteer the sharding hard problem.** Say unprompted: "you cannot do `ZRANK` across shards." Then offer two solutions — scatter-gather `ZCOUNT` for exact rank, and the bucketed histogram for approximate rank. Offering two distinct solutions to the same hard problem is a strong senior signal.
- **Right-size Redis memory.** Showing that 50 M × 200 B = 10 GB per board, multiplied by the segment count, yields a realistic ~135 GB cluster reframes the problem as manageable infrastructure rather than exotic big-data engineering. Interviewers appreciate when candidates eliminate false complexity.
- **Distinguish real-time from near-real-time.** The synchronous write path (direct `ZADD`) is simpler but couples game servers to Redis health. The Kafka path adds ~100–500 ms lag in exchange for durability and decoupling. Knowing when to make that trade-off signals operational maturity.
- **Common follow-ups to rehearse:** friend-group boards (`ZUNIONSTORE` vs. incremental update), daily/weekly resets (key rotation + TTL), anti-cheat and score validation (upstream pipeline, not leaderboard's problem), percentile display vs. exact rank, "what if the game has 500 M players?" (scale shard count, switch to approximate rank), and Redis memory eviction policies for windowed boards.

## Resiliency

- **Kafka as the durable backbone.** All score events land in Kafka before any acknowledgment to the game server. If the Score Processor crashes, it replays from the last committed Kafka offset — no events are lost. Redis is a derived, recomputable cache; Kafka and ClickHouse are the source of truth.
- **Redis Sentinel / Cluster HA.** Each primary has 2+ replicas managed by Redis Sentinel (for single-primary topology) or Redis Cluster (for sharded topology). Primary failure triggers automatic leader election and failover in < 30 s.
- **Graceful degradation on Redis failure.** If the primary is unavailable: rank reads fall back to a surviving replica (stale by seconds — acceptable under our near-real-time SLO); writes queue in Kafka and are applied when the primary recovers or a new primary is elected.
- **Idempotent ZADD GT.** `ZADD GT` is naturally idempotent: replaying the same score event (same composite) on the same ZSET member is a no-op if the score has not changed. No deduplication table needed on the Redis write path — the event log itself handles replay safety.
- **Histogram consistency.** The Lua script wrapping `HINCRBY + ZADD` ensures atomicity per Score Processor write. A periodic reconciliation job (run against a replica to avoid blocking primary reads) validates histogram bucket counts against `ZRANGEBYSCORE` results and resets any drift.
- **Board rebuild SLA.** Replaying 30 days of events through the Score Processor rebuilds all boards in approximately 3 hours — an acceptable RTO for catastrophic Redis data loss. Prioritise the global and daily boards first; segmented boards can follow.
- **Backpressure.** If the Redis primary saturates, the Score Processor slows its Kafka consumption naturally (bounded consumer lag). Kafka absorbs the backlog without data loss. The Score Processor can also apply **score coalescing**: if a user has two events in the batch, process only the higher score `ZADD GT` in one call.

## Observability

- **SLIs:** top-K read p50/p99 latency, my-rank read p50/p99, score-event end-to-end lag (Kafka publish timestamp → ZADD timestamp), Redis primary write ops/s, Kafka consumer lag per partition, Redis memory utilisation per node.
- **Golden alerts:**
  - Kafka consumer lag > 10,000 events (Score Processor falling behind; investigate CPU, Redis latency, or pipeline batch size).
  - Redis memory > 85% on any node (capacity risk; trigger provisioning of a new replica or shard split).
  - Rank read p99 > 8 ms (approaching SLO; check replica replication lag, app tier bottlenecks).
  - Redis primary failover event (page on-call; validate histogram integrity and consumer lag after promotion).
- **Tracing.** Propagate a `trace_id` from the game server's POST through the Kafka message header → Score Processor → Redis pipeline, so a slow end-to-end score update can be attributed to Kafka lag vs. Redis pipeline stall vs. network jitter between tiers.
- **Dashboards:** events/s by segment (global vs. per-country), top-10 most-updated user IDs (hot-member detection for abuse or bots), Redis ZSET cardinality per board (growth tracking), histogram drift metric (max deviation between approximate rank from histogram and exact ZREVRANK on a 1%-random sample), daily/weekly board rollover success rate, friend-board `ZUNIONSTORE` p99 latency.
- **Logging.** Log every board rollover, histogram reconciliation event, and shard failover in full — these are rare and operationally significant. Sample score-update logs at 0.1% (5,000/s is far too hot for full event logging).

## Concepts Used

- {{% relref "/design-concepts/storage/key-value-stores" %}} — Redis as the leaderboard store; skip-list + hash map internals that make ZADD / ZREVRANK / ZREVRANGE O(log n) and sub-millisecond
- {{% relref "/design-concepts/messaging/kafka" %}} — durable score event ingestion, game-server decoupling, consumer backpressure, and board-rebuild replay
- {{% relref "/design-concepts/data/batch-vs-streaming" %}} — Score Processor pipeline batching vs. near-real-time single-event processing; real-time vs. near-real-time write path trade-off
- {{% relref "/design-concepts/storage/hotspot-problems" %}} — single ZSET primary node as a write hotspot; read replica fan-out to distribute rank-read load
- {{% relref "/design-concepts/storage/caching-patterns" %}} — Redis as a recomputable derived cache; TTL-based eviction on windowed boards; user profile caching for top-K enrichment
- {{% relref "/design-concepts/scaling/sharding" %}} — sharding the ZSET by user_id to overcome single-node RAM and throughput limits; scatter-gather ZCOUNT for exact global rank across shards
- {{% relref "/design-concepts/storage/consistent-hashing" %}} — distributing ZSET shards across Redis nodes so adding a new shard remaps only a fraction of user_id → shard assignments
