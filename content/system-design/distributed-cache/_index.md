---
title: 'Distributed Cache'
weight: 1
type: docs
---

Large-scale applications — social feeds, e-commerce product pages, ride-sharing dispatch, gaming leaderboards — share a common problem: their databases cannot absorb the full read load at acceptable latency. A **distributed in-memory cache** sits between the application tier and the database, holding the hot working set of data in RAM and serving it in under a millisecond. Facebook's Memcached fleet serves trillions of items across thousands of nodes; Redis Cluster powers similar workloads at GitHub, Twitter/X, and Shopify. The design looks trivial — "put a hash map in front of your database" — but the engineering challenges around sharding keys across nodes, handling node failures without a miss storm, preventing stampedes when a popular key expires, and keeping cache and source of truth consistent are where the real depth lives.

We are designing a horizontally-scalable, multi-node in-memory key-value cache cluster. Application services embed a client library that routes each key to the right cache node, falls back to the backing database on a miss, and populates the cache for subsequent reads. The cluster must handle 1 million read operations per second at sub-millisecond latency.

## Functional Requirements

1. **GET:** Return the stored value for a key, or a cache-miss signal if the key is absent or has expired.
2. **SET:** Store a key-value pair with an optional TTL (time-to-live in seconds). Overwrites any existing value.
3. **DELETE:** Explicitly remove a key and its value from the cache.
4. **Batch reads (MGET):** Fetch multiple keys in a single network round-trip; missing keys return null in the response array.
5. **Batch writes (MSET):** Store multiple key-value pairs atomically per-node (best-effort across nodes).
6. **TTL and expiry:** Keys expire automatically after their TTL elapses; the client is responsible for choosing an appropriate TTL based on acceptable staleness.
7. **Eviction:** When a node's memory is full, evict keys according to a configurable policy (LRU or LFU).

## Out of Scope

- **Durable persistence.** Data loss on a node restart is acceptable; the backing database is the source of truth.
- **Pub/sub and event streaming.** Messaging features are out; this is a cache, not a message broker.
- **Server-side scripting** (e.g., Lua, stored procedures). Computation belongs in the application tier.
- **Cross-datacenter replication.** Each datacenter operates its own independent cache cluster; cross-DC consistency is an application concern.
- **Authentication and multi-tenancy.** Assumed handled at the network boundary (e.g., mTLS, VPC isolation, or an API gateway).
- **Strong consistency guarantees.** The cache and the database will be inconsistent within the TTL window — this is by design.

## Non-Functional Requirements

- **Scale:** 1,000,000 read QPS and 100,000 write QPS at peak (10:1 read:write ratio).
- **Latency:** p50 < 0.3 ms, p99 < 1 ms for GET. The cache engine must not be the bottleneck; network RTT will dominate.
- **Availability:** 99.99% (< 53 minutes downtime/year). A cache cluster outage must degrade performance — not correctness. The backing database remains authoritative.
- **Working set:** ~1 TB of hot data in RAM across the cluster. Average key: 100 B; average value: 1 KB; average entry: ~1.1 KB.
- **Consistency:** Cache-aside best-effort; staleness is bounded by TTL. The cache and backing database may diverge within the TTL window — acceptable for the use cases in scope.
- **No durability requirement:** A cache miss is a performance penalty, not a correctness failure. Node restarts may flush data cold.
