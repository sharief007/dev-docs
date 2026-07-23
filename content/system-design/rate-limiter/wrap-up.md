---
title: 'Wrap-Up'
weight: 4
type: docs
---

## Interview Tips

- **Lead with the accuracy ↔ latency trade-off triangle.** Every interesting decision in this design maps back to three axes: how accurate you want limits to be, how much latency you can add per request, and how much load you put on Redis. State this triangle early — it gives you a framework to justify every subsequent choice.
- **Walk through all four algorithms before committing.** Naming the fixed-window boundary-burst problem unprompted signals genuine familiarity with the domain. Then eliminate sliding-window log (memory) and token bucket (distributed sync complexity) and land on sliding-window counter with explicit reasoning.
- **Bring up the Lua script.** Explaining why a Redis Lua script gives atomicity without a distributed lock (single-threaded GIL, no round-trip overhead of SETNX) is a strong senior signal. Sketch the pseudocode — interviewers rarely see that level of detail.
- **State the fail-open / fail-closed trade-off and tie it to CAP.** This is one of the few questions in system design with no universal "right" answer — it depends on whether the rate limiter is a fairness control (fail-open is safer) or a security boundary (fail-closed is safer). Articulating this clearly distinguishes senior from mid-level.
- **Common follow-ups to rehearse:**
  - *"How do you handle a sudden spike from one abusive user?"* → local counter catches it first; Lua enforces at sync; circuit breaker isolates Redis; sticky routing makes it exact.
  - *"What if a gateway node crashes?"* → local state is lost; new node starts fresh, granting one window of slightly elevated allowance — acceptable under eventual consistency.
  - *"How do you rate-limit across regions?"* → per-region Redis clusters by default (cross-region sync adds 50–150 ms latency, usually not worth it); global limits require accepting larger over-allowance windows or synchronous cross-region replication.
  - *"How do you test the rate limiter?"* → unit-test the Lua script with `redis-cli EVAL`, load-test the sidecar with synthetic burst traffic, chaos-test Redis failure with Toxiproxy.
  - *"Can the rate limiter be bypassed?"* → yes, if requests are spread across regional PoPs with independent Redis clusters. Accept this or add a global coordination layer at the cost of latency.

## Resiliency

- **No central enforcement process:** the rate-limit sidecar runs *on* each gateway node — the enforcement tier itself has no single process to fail. Redis failure degrades to local-cache-only operation, not a total outage.
- **Redis cluster with replicas:** counter keys are partitioned across Redis shards using consistent hashing. Each shard has at least one read replica; a primary failover promotes a replica within seconds. See {{% relref "/design-concepts/storage/consistent-hashing" %}}.
- **Circuit breaker on Redis calls:** opens after a configurable error threshold, falls back to local token cache, probes for recovery periodically. Prevents Redis latency spikes from cascading into request-path latency. See {{% relref "/design-concepts/reliability/circuit-breaker" %}}.
- **Local token cache as a latency buffer:** even when Redis is fully healthy, the local cache decouples the request path from Redis GC pauses, leader-election delays, and network jitter. Transient Redis slowness does not directly spike request p99.
- **Back-pressure propagation:** if downstream services signal overload, the gateway can tighten rate limits adaptively — reducing traffic before it reaches saturated origins. See {{% relref "/design-concepts/reliability/back-pressure" %}}.
- **Idempotent rule updates:** limit rules are delivered via pub/sub and applied idempotently (`PUT` semantics) — duplicate or out-of-order deliveries are harmless. See {{% relref "/design-concepts/distributed/idempotency" %}}.
- **Multi-region deployment:** each region operates its own Redis cluster and gateway fleet independently. Limits are enforced per-region. Cross-region counter coordination is an opt-in configuration for tenants who need global precision — it adds one cross-region RTT to every sync and is rarely worth the latency cost. See {{% relref "/design-concepts/distributed/cap-theorem" %}} for the consistency vs. availability trade-off in a partitioned multi-region counter store.

## Observability

**SLIs:**
- Rate-limit check latency p50 / p99 — target < 1 ms p99.
- 429 rate as a fraction of total requests — baseline vs. spike detection.
- Redis op latency p99 and error rate — leading indicator of sidecar fallback.
- Local cache hit ratio — high means Redis is healthy and load is well-distributed.
- Sync worker interval lag — time between target sync interval and actual sync execution.

**Golden alerts:**
- Redis error rate > 1% over any 10 s window → circuit breaker nearing open; page on-call.
- Rate-limit check p99 > 2 ms → local cache may be cold or Redis is saturated.
- 429 rate spikes > 10× rolling baseline → possible abuse campaign or misconfigured limit; page and investigate.
- Sync worker lag > 2× target interval on any node → node is overloaded or stalled.
- Redis cluster memory > 80% → headroom shrinking; risk of key eviction; capacity event.

**Logging:**
- Log every 429 event with `{user_id, ip, endpoint, limit, count, window_start, node_id}` — essential for customer support ("why am I being throttled?") and abuse investigation.
- Log every limit-rule change with the before/after values, the operator identity, and the change timestamp for audit.
- Sample allow-path logs at 0.1% — logging every allowed request at 1M req/s is impractical and unnecessary.

**Tracing:** propagate a trace ID from the client through LB → gateway → sidecar → Redis. Tag sidecar spans with `rl.check_result` (allow / deny), `rl.remaining`, `rl.source` (local-cache or Redis), and `rl.redis_latency_ms`. This makes it trivial to identify which tier accounts for tail latency.

**Dashboards:**
- QPS vs. 429 rate per endpoint and per tier (free / paid / internal).
- Top-N users by request rate leaderboard — identifies misbehaving or high-value clients.
- Redis cluster memory utilisation and key-count growth vs. projected capacity.
- Circuit-breaker state timeline per node — shows how often and how long each node operated in degraded mode.
- Sync interval histogram — distribution of actual sync intervals reveals whether workers are keeping up.

## Concepts Used

- {{% relref "/design-concepts/rate-limiting/algorithms" %}} — fixed window, token bucket, sliding window log, sliding window counter comparison
- {{% relref "/design-concepts/distributed/cap-theorem" %}} — fail-open vs. fail-closed under Redis partition; per-region vs. global counter trade-offs
- {{% relref "/design-concepts/distributed/distributed-locking" %}} — why a Lua script avoids needing a distributed lock for atomic check-increment
- {{% relref "/design-concepts/storage/key-value-stores" %}} — Redis as the counter store; INCR, EXPIRE, Lua EVALSHA
- {{% relref "/design-concepts/storage/consistent-hashing" %}} — partitioning counter keys across a Redis cluster
- {{% relref "/design-concepts/storage/hotspot-problems" %}} — high-cardinality IP attacks; single hot-user node saturation
- {{% relref "/design-concepts/distributed/idempotency" %}} — idempotent rule updates; safe at-least-once pub/sub delivery
- {{% relref "/design-concepts/networking/load-balancing" %}} — sticky vs. round-robin routing and the counter accuracy implication
- {{% relref "/design-concepts/networking/http-headers" %}} — X-RateLimit-Limit, X-RateLimit-Remaining, X-RateLimit-Reset, Retry-After
- {{% relref "/design-concepts/networking/http-status-codes" %}} — 429 Too Many Requests semantics
- {{% relref "/design-concepts/reliability/circuit-breaker" %}} — isolating Redis failure from the request path
- {{% relref "/design-concepts/reliability/back-pressure" %}} — adaptive limit tightening under downstream overload
