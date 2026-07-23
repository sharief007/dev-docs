---
title: 'Wrap-Up'
weight: 4
type: docs
---

## Interview Tips

- **Open with the read-to-write skew.** Autocomplete is almost entirely a read problem—the write path is an offline pipeline, not a live write per request. Framing this up front justifies every caching decision you make.
- **Introduce the trie early, explain the top-K cache.** Interviewers want to know *why* you choose a trie over a hash map or inverted index. The answer is that the trie's top-K node cache turns an O(|subtree|) traversal into an O(|prefix|) point lookup—that's the killer property.
- **Walk through the build pipeline.** The batch job (query logs → frequency aggregation → time-decayed scoring → trie annotation → serialization → atomic deploy) is a rich discussion area. Be ready to explain each step and why it's offline.
- **Volunteer the streaming layer as a freshness fix.** Showing the two-tier pipeline (batch correctness + streaming freshness) demonstrates you can balance consistency and latency trade-offs—a senior signal.
- **Mention CDN caching of hot prefixes.** Short prefixes are shared across all users. Caching `?prefix=h` at the edge and serving it to millions of users without touching origin is a simple, high-leverage insight.
- **Discuss sharding trade-offs.** Range partitioning by first character is simple but skewed. Hash-based or two-character sharding is more balanced. Interviewers often probe here.
- **Common follow-ups to rehearse:**
  - How would you add personalization without breaking the shared cache?
  - How do you handle a language the trie doesn't cover yet?
  - What happens when a political/offensive query becomes briefly popular?
  - How do you handle the privacy floor—suppressing rare queries?
  - How do you roll back a bad snapshot?

---

## Resiliency

- **Immutable snapshots + atomic swap.** Serving nodes never mutate the live trie—they swap a pointer from old to new. A bad snapshot can be rolled back by pointing `latest` at the previous version; no data migration needed.
- **Multiple replicas per shard.** Each prefix shard runs 2+ replicas. A node failure shifts traffic to healthy replicas via the load balancer. Replicas share nothing; each holds an identical in-memory trie copy.
- **CDN as a fallback buffer.** If all origin nodes for a shard go down, the CDN continues serving stale cached responses (TTL extends on origin error). Users see slightly stale suggestions rather than errors.
- **Kafka durability.** Search events are durable in Kafka for 24 hours. If the streaming aggregator crashes, it replays from the last committed offset. If the Spark batch job fails, the previous snapshot remains active.
- **Graceful degradation:** if the streaming delta service is unavailable, the serving nodes fall back to the last batch trie—still correct within 24 hours, just not trending-aware.
- **Backpressure on the pipeline.** The Spark batch job and Kafka consumers have independent pacing. A surge in search events never stalls the serving path. See [back-pressure]({{% relref "/design-concepts/reliability/back-pressure" %}}).

---

## Observability

**SLIs (Service Level Indicators):**

| Signal | Measurement |
|---|---|
| Suggestion p50 / p99 latency | End-to-end from client request to response |
| Trie lookup time (in-process) | Time from trie root traversal to result serialization |
| CDN cache hit ratio | Cache hits ÷ total requests at edge |
| Serving availability | Non-5xx responses ÷ total requests |
| Snapshot age | `now − last_successful_snapshot_timestamp` |
| Streaming delta lag | Kafka consumer group lag in seconds |

**Golden alerts:**

- Suggestion p99 > 100 ms → investigate trie shard overload or CDN miss spike.
- CDN cache hit ratio < 50% → cache key misconfiguration or traffic pattern shift.
- Snapshot age > 30 hours → Spark batch job failed; stale trie risk.
- Streaming lag > 20 minutes → aggregator behind; trending queries not surfacing.
- Shard replica count < 2 → failover risk; page on-call.

**Tracing:** propagate a trace ID from the client through CDN → load balancer → trie shard → (optional) personalization re-ranker. Attribute tail latency to the specific tier: edge, network, trie traversal, or re-ranking.

**Logging:** log every snapshot swap event (version, size, load time, errors). Sample suggestion requests at 1% for quality auditing (did the prefix `"how do i"` return sensible results?). Log all Spark batch run outcomes with row counts and timing.

**Dashboards:**
- Top-10 prefixes by QPS and their cache hit ratios.
- Trending query leaderboard (most rapidly rising queries in the last hour).
- Snapshot deployment timeline across shard replicas (are all nodes on the same version?).
- Kafka consumer lag over time per partition.
- Shard load distribution (are shards balanced? are `"s-"` prefixes getting 3× traffic?).

---

## Concepts Used

- {{% relref "/design-concepts/specialized/typeahead" %}} — trie structure, prefix-based completion, top-K node caching
- {{% relref "/design-concepts/messaging/kafka" %}} — durable event log for search events; streaming aggregation source
- {{% relref "/design-concepts/data/batch-vs-streaming" %}} — two-tier pipeline: Spark batch for correctness, streaming for freshness
- {{% relref "/design-concepts/networking/cdn" %}} — edge caching of hot prefix responses
- {{% relref "/design-concepts/storage/key-value-stores" %}} — query frequency store for the streaming delta layer
- {{% relref "/design-concepts/storage/consistent-hashing" %}} — hash-based prefix sharding for balanced shard load
- {{% relref "/design-concepts/storage/caching-patterns" %}} — cache-aside at the CDN; TTL strategy for prefix responses
- {{% relref "/design-concepts/storage/full-text-search" %}} — contrast with inverted-index full-text search; fuzzy matching techniques
- {{% relref "/design-concepts/reliability/back-pressure" %}} — pipeline pacing so event spikes don't stall serving
- {{% relref "/design-concepts/replication/leader-based-replication" %}} — replica strategy for trie serving shards
