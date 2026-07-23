---
title: 'Wrap-Up'
weight: 4
type: docs
---

## Interview Tips

- **Open by naming the read:write skew.** The single most important framing is "this is ~3000:1 read-heavy," because it justifies caching, replicas, and 302-vs-301 discussion. Interviewers wait for you to say it.
- **Drive the key-generation discussion.** This is the real meat. Compare three approaches out loud: (a) random + collision check, (b) hash the URL (MD5/base62, truncate — but collisions on truncation), (c) **distributed counter + base62** (preferred). Mention the guessability trade-off and the Feistel/XOR fix.
- **Volunteer 301 vs 302.** Explaining why you pick 302 (analytics + flexibility, at the cost of more traffic) signals real-world experience.
- **Right-size the storage.** Showing that 5 years is only ~6 TB reframes the problem from "big data" to "big QPS," which changes the whole solution shape. Don't over-engineer storage.
- **Bloom filter is a strong bonus.** Bringing it up unprompted to kill unknown-key lookups is a classic senior signal.
- **Common follow-ups to rehearse:** custom aliases, link expiration & cleanup, analytics pipeline, preventing abuse/rate-limiting creation, multi-region consistency, and "what if a single key goes viral" (hot-key handling).

## Resiliency

- **Statelessness + multi-region:** app tier holds no session state, so any node/region can serve any request; a regional outage sheds to others via the global LB.
- **KGS range buffering:** each node caches an unused ID range, so short-lived allocator outages don't stop creation.
- **Replication & quorum:** 3× replication on the KV store; redirects can read from the nearest healthy replica. See {{% relref "/design-concepts/replication/leader-based-replication" %}}.
- **Graceful degradation:** if the click pipeline is down, redirects still succeed (analytics is best-effort, decoupled via the queue). If Redis is down, redirects fall through to replicas — slower but correct.
- **Idempotency** on creation prevents duplicate keys on client retries and at-least-once queue delivery.
- **Backpressure:** the click stream buffers bursts; the aggregator consumes at its own pace so a spike never stalls the redirect path.

## Observability

- **SLIs:** redirect p50/p99 latency, redirect availability (non-5xx ratio), cache hit ratio (L1 & L2), DB read latency, KGS range-refill rate, click-pipeline lag.
- **Golden alerts:** cache hit ratio drops below ~90% (stampede risk), redirect p99 > 50 ms, 404 rate spike (possible abuse or bad deploy), KGS allocator errors, replica lag beyond threshold.
- **Tracing:** propagate a trace ID through LB → redirect → cache → DB to attribute tail latency to a tier.
- **Dashboards:** QPS by region, hot-key leaderboard (top keys by request rate), storage growth vs. projection, error budget burn-down for the 99.9% redirect SLO.
- **Logging:** sample redirect logs (full logging at 115k/s is wasteful); log all creation/deletion events fully for audit.

## Concepts Used

- {{% relref "/design-concepts/specialized/id-generation" %}} — distributed counter, base62, guessability trade-offs
- {{% relref "/design-concepts/storage/consistent-hashing" %}} — sharding the KV store
- {{% relref "/design-concepts/storage/key-value-stores" %}} — the durable mapping store
- {{% relref "/design-concepts/storage/caching-patterns" %}} & {{% relref "/design-concepts/storage/cache-eviction" %}} — cache-aside, LRU/LFU, TTLs
- {{% relref "/design-concepts/storage/bloom-filters" %}} — cheap negative lookups
- {{% relref "/design-concepts/storage/hotspot-problems" %}} — viral / celebrity keys
