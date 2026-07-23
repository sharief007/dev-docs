---
title: 'Wrap-Up'
weight: 4
type: docs
---

## Interview Tips

- **Open with the working-set-fits-in-RAM insight.** The foundational reason distributed caches work is that the hot 20% of data often fits in a modest amount of RAM and absorbs 80% of reads. Stating this immediately — and sizing the cluster around it — shows you understand the system's purpose.
- **Drive the sharding discussion early.** Most candidates say "shard by key." The follow-up is "what happens when a node dies or you add a node?" If your answer is modulo hashing, you get the catastrophic-rehash story. Come in already knowing consistent hashing with virtual nodes and volunteer the trade-offs (ring O(log N) lookup, vnode count vs. balance, heterogeneous capacity weighting).
- **Stampede prevention is a senior signal.** Naming the thundering herd problem unprompted and offering at least two mitigations (single-flight + probabilistic early expiry + mutex locking) distinguishes senior candidates. Most candidates describe the problem; fewer describe all three solutions.
- **Name all three write policies, pick one, and explain why.** Write-through sounds safe but causes the stale-set race under concurrency. Write-back is tempting for performance but risks data loss. Write-around with explicit DEL is the right default for most use cases — but knowing all three and their failure modes is what matters.
- **LRU vs LFU is a concrete trade-off.** Be ready to give a concrete example where LRU fails (a large sequential scan evicts the entire hot set) and where LFU fails (new hot keys get evicted before they accumulate frequency). Redis's approximated LRU is a good bridge answer.
- **Bring up the hot-key problem.** Even with consistent hashing, a single viral key can saturate one node. Knowing that the solution is K-replica sharding and client-side L1 caching — not just "add more nodes" — demonstrates depth.
- **Common follow-ups:** How do you handle a node that is warming up after a restart? What is the consistency model for write-through vs. write-around? When would you use Redis instead of Memcached (data structures, persistence, Lua)? How do you detect and respond to hot keys in production without prior knowledge?

## Resiliency

- **Replica per primary.** Each primary has a hot replica that is already warmed. On primary failure, the replica promotes in < 5 seconds via a heartbeat-based health check. The consistent hash ring updates to point to the replica, and no miss storm occurs because the replica is already serving the same key range.
- **Rate-limited cache warming.** When a node is provisioned from scratch (horizontal scale-out, hardware replacement), a warming service pre-populates it at a controlled read rate from the database before the node enters the ring at full vnode count. Traffic is incrementally migrated as fill ratio rises.
- **Graceful degradation to the database.** The cache is a performance layer, not a correctness layer. On total cache cluster failure, all traffic falls to the backing database. Ensure the database is provisioned with enough read capacity to absorb at least a fraction of peak cache-miss traffic (or paired with read replicas). Implement circuit breakers in the cache client so a slow cache node does not block the application — fail fast and go to the DB.
- **Bloom filter persistence.** The Bloom filter (for negative caching) should be persisted to disk or rebuilt from the database on restart, otherwise all requests become cache-miss candidates for the filter warm-up window.
- **Back-pressure on DB.** Without {{% relref "/design-concepts/reliability/back-pressure" %}}, a cache miss storm can cascade to the database. The single-flight pattern is the primary mechanism; a secondary circuit-breaker that short-circuits DB calls when the DB latency exceeds threshold prevents complete database overload from turning a cache outage into a database outage.
- **TTL jitter.** If many keys are SET at the same time (e.g., during a nightly batch load), they will expire simultaneously. Add ±10–20% random jitter to TTLs at SET time to spread expiry events across time. This is a simple, zero-code-complexity improvement.
- **Connection pool health.** Cache client connections should have a health-check timeout (e.g., 50ms). If a cache node is slow or unresponsive, the client should detect this quickly and mark the node as unhealthy, removing it from the ring temporarily rather than queuing requests behind it.

## Observability

**SLIs (key metrics to instrument):**
- **Cache hit rate** per cache tier: L1 (in-process), L2 (distributed cache). Target: L2 > 90%, combined > 99%.
- **GET latency p50/p99** at the cache client. Alert if p99 exceeds 1ms.
- **Write latency p99** for SET operations. Alert if > 2ms.
- **Eviction rate** per node. Sustained high eviction = working set overflow; scale the cluster.
- **Miss rate spike** (cache misses/s). A sudden jump signals a node failure, TTL cliff, or new traffic pattern.
- **Single-flight coalescing ratio**: (requests coalesced) / (DB calls made). A ratio < 10× on a hot key indicates the coalescing window is too narrow or the key is not actually hot.
- **Bloom filter false positive rate.** Track keys that passed the filter but missed in both cache and DB.
- **Connection pool saturation** per (app-node, cache-node) pair.

**Alerting:**
- L2 hit rate < 85% → potential miss storm or node failure; page on-call.
- p99 GET > 1ms sustained for > 30s → cache node under load or network issue.
- Eviction rate > 5% of capacity/minute → cluster undersized; schedule scale-out.
- Cache node heartbeat failure → automated replica promotion; alert for human review.
- DB read QPS spike > 3× baseline → cache miss cascade; trigger circuit-breaker runbook.

**Tracing and logging:**
- Propagate a trace ID through the application request into every cache GET and SET call. This lets you attribute tail latency to a specific cache node, network hop, or DB fallback.
- Log cache misses at a sampled rate (e.g. 1%) with the key prefix (not the full key — it may contain PII). This surfaces which key namespaces are generating the most misses for TTL or eviction policy tuning.
- Use structured logs (JSON) for cache events so dashboards can aggregate by key prefix, node, and operation type.

**Dashboards:**
- Request waterfall: L1 hit → L2 hit → DB miss. Visualise the proportion of requests absorbed at each tier.
- Hot-key leaderboard: top-N keys by request rate, updated every 60s. Identifies candidates for K-replica replication or L1 caching.
- Node memory heat map: memory fill % per node. Flag nodes approaching the eviction threshold.
- Error budget burn-down for the 99.99% availability SLO.

## Concepts Used

- {{% relref "/design-concepts/storage/consistent-hashing" %}} — virtual-node hash ring for sharding keys across cache nodes; 1/N rehash on topology change
- {{% relref "/design-concepts/storage/caching-patterns" %}} — cache-aside read pattern; write-through, write-around, write-back policies; read-through and refresh-ahead variants
- {{% relref "/design-concepts/storage/cache-eviction" %}} — LRU, LFU, TTL-only eviction policies; approximated LRU; eviction churn and working-set overflow
- {{% relref "/design-concepts/storage/bloom-filters" %}} — negative caching for never-issued keys; count-min sketch for hot-key frequency detection
- {{% relref "/design-concepts/storage/hotspot-problems" %}} — hot-key replication across K vnodes; client-side L1 caching for top-N keys; thundering herd / stampede
- {{% relref "/design-concepts/storage/key-value-stores" %}} — hash table internals; LRU doubly-linked list; Redis vs Memcached engine trade-offs
- {{% relref "/design-concepts/networking/load-balancing" %}} — cluster-aware client vs proxy (twemproxy, mcrouter) topology trade-offs; sidecar proxy pattern
- {{% relref "/design-concepts/reliability/back-pressure" %}} — circuit breaker on cache miss cascade to DB; single-flight as demand-side back-pressure on database reads
