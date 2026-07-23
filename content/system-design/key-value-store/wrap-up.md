---
title: 'Wrap-Up'
weight: 5
type: docs
---

## Interview Tips

- **Open by naming the access pattern and scale.** State "we have a 10:1 read:write ratio at 116k reads/s and 11.6k writes/s steady state" — this immediately frames every design decision. The read:write ratio and latency targets are what determine storage engine choice, quorum settings, and caching strategy.

- **Walk the storage engine progression out loud.** Compare three options — (a) in-memory hash table (fast, volatile, RAM-limited), (b) WAL + snapshots (durable, still RAM-limited), (c) **LSM-tree** (write-optimised, disk-backed, industry standard). An interviewer who wants depth will ask you to draw the read and write paths — practice the pseudocode.

- **State N, R, W explicitly.** Say "I'll use replication factor N=3 with W=2 and R=2, so W+R>N gives read-after-write consistency" — this shows familiarity with Dynamo-style systems and invites a rich trade-off discussion. Be ready to adjust: "if we need lower write latency, drop W to 1 and accept eventual consistency."

- **Contrast leader-follower vs leaderless.** Mention that leader-follower (Raft-based) gives total ordering of writes per key range but imposes election latency on leader failure; leaderless quorum has no election but allows concurrent writes that produce conflicts. State which you'd pick for this workload (leaderless for high availability) and why.

- **Bring up vector clocks vs LWW.** Explaining that LWW silently drops concurrent writes while vector clocks surface conflicts to the application is a senior-engineer signal. Note that Dynamo originally used vector clocks and that Cassandra chose LWW for simplicity — both are defensible, but the choice must be deliberate.

- **Common follow-ups to rehearse:** range queries (not supported — need a different data model or a range-index layer on top), TTL expiration cleanup (lazy compaction + background sweeper), cross-datacenter replication (requires multi-region design), cluster rebalancing when a node joins or leaves (vnode migration), and "what if one key receives 1M writes/s" (client-side sharding by appending a random suffix, or hot-key replication).

## Resiliency

- **Replication factor ≥ 3:** the cluster tolerates up to N−1 simultaneous node failures without data loss. Even with W=2, a single-node failure allows writes to proceed using the two surviving replicas.

- **WAL durability before ack:** every write is fsynced to the WAL before the coordinator returns success. Even a process crash between WAL write and memtable update is safe — WAL replay restores the entry on restart.

- **Hinted handoff:** short-term outages (minutes to hours) are absorbed transparently. The write succeeds against healthy nodes; hints are delivered when the target recovers. The client sees no failure.

- **Anti-entropy Merkle sweep:** long-term replica drift — from extended outages, dropped hints, or network partitions — is detected and repaired by the periodic background sweep. The sweep runs at low I/O priority so it doesn't compete with live traffic.

- **Gossip-based failure detection:** each node broadcasts heartbeats via the gossip protocol. A node that misses *k* consecutive heartbeats from a peer (e.g. k=3) is marked suspect, then down. Ring routing is updated automatically. No single point of failure in the failure detector.

- **Graceful degradation under partition (AP mode):** surviving partitions accept reads and writes independently. When the partition heals, Merkle trees reconcile diverged data. Conflicts resolved via LWW or vector clocks depending on configuration.

- **Compaction backpressure:** if SSTable accumulation outpaces compaction (write burst), the write path is rate-limited to protect read latency. See [Back-Pressure]({{% relref "/design-concepts/reliability/back-pressure" %}}).

## Observability

- **SLIs (service-level indicators):**
  - GET p50/p99 latency (target: p99 < 10 ms)
  - PUT p50/p99 latency (target: p99 < 20 ms)
  - Quorum success rate (target: > 99.99%)
  - Bloom filter false-positive rate per level (signals under-sized filter)
  - L0 SSTable count per node (signals compaction falling behind)
  - Replica sync lag (seconds behind coordinator)
  - Hinted handoff queue depth per node

- **Golden alerts:**
  - GET p99 > 10 ms sustained for > 1 minute
  - Quorum failure rate > 0.01% (a replica may be down or partitioned)
  - L0 SSTable count > 20 on any node (compaction stall imminent)
  - Replica sync lag > 60 s (anti-entropy not keeping up)
  - Hinted handoff queue > 10k entries (a node has been down too long)
  - WAL size growing without bound (flush stall — memtable blocked)

- **Distributed tracing:** propagate a trace ID through client → proxy/coordinator → replica nodes. Every hop records its latency contribution. Tail-latency spikes can be attributed to specific replicas (compaction contention, GC pause) or the network.

- **Dashboards:**
  - QPS by operation type (read vs write vs delete) per node
  - Cluster ring health map — vnode ownership, replicas per node
  - SSTable count and compaction throughput per level per node
  - Hinted handoff queue depth and delivery rate
  - Conflict resolution events per second (vector clock vs LWW) — a spike signals a write hotspot
  - Error-budget burn-down for the 99.99% availability SLO

- **Logging:** log every quorum failure (including which replicas did not respond), every hinted handoff creation and delivery, and every compaction start/end with bytes merged. Sample successful reads at 0.1% (logging at 350k reads/s would saturate disk).

## Concepts Used

- {{% relref "/design-concepts/storage/key-value-stores" %}} — core data model and KV store use cases
- {{% relref "/design-concepts/storage/lsm-trees" %}} — LSM-tree architecture, memtable, SSTables, compaction strategies
- {{% relref "/design-concepts/storage/consistent-hashing" %}} — ring-based key distribution with virtual nodes
- {{% relref "/design-concepts/storage/hash-index" %}} — in-memory hash map as the simplest KV engine; comparison baseline
- {{% relref "/design-concepts/storage/b-plus-tree" %}} — alternative disk engine; read-heavy contrast to LSM
- {{% relref "/design-concepts/storage/bloom-filters" %}} — O(1) probabilistic negative lookups on SSTables
- {{% relref "/design-concepts/storage/cache-eviction" %}} — LRU/LFU policies for managing the in-memory hot set
- {{% relref "/design-concepts/storage/hotspot-problems" %}} — hot-key detection and mitigation strategies
- {{% relref "/design-concepts/distributed/gossip-protocol" %}} — decentralised ring state and failure detection
- {{% relref "/design-concepts/distributed/quorum" %}} — N/R/W quorum semantics and the W+R > N invariant
- {{% relref "/design-concepts/distributed/cap-theorem" %}} — CP vs AP trade-off under partition
- {{% relref "/design-concepts/distributed/pacelc" %}} — latency vs consistency even without partitions
- {{% relref "/design-concepts/distributed/logical-clocks" %}} — vector clocks for causality tracking and conflict detection
- {{% relref "/design-concepts/distributed/leader-election" %}} — Raft-based leader election in CP replication mode
- {{% relref "/design-concepts/replication/leader-based-replication" %}} — leader-follower model; ordering guarantees and election overhead
- {{% relref "/design-concepts/replication/replication-implementations" %}} — anti-entropy, Merkle trees, hinted handoff, read repair
- {{% relref "/design-concepts/consensus/raft" %}} — consensus algorithm underpinning CP leader election
- {{% relref "/design-concepts/reliability/back-pressure" %}} — compaction throttling to protect read latency
