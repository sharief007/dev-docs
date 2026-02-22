# System Design Interview Prep — 5 Month Roadmap

> **Target:** FAANG / top-tier companies
> **Timeline:** 5 months (~20 weeks)
> **Skipped:** Infrastructure & DevOps (Docker, K8s, IaC, CI/CD, Service Mesh)
> **Reduced:** AI/ML (only recommendation systems + ML platform design)

---

## How to Use This File

Each entry has a file path and subtopics. When you want to generate a post, paste the entry into Claude or Copilot with:

> "Generate a Hugo markdown post for this topic. Follow the existing content format in this repo (YAML frontmatter, real-world scenario, problem/solution structure, Mermaid diagram, code example, tradeoffs table)."

---

## Monthly Overview

| Month | Focus | Topics |
|-------|-------|--------|
| 1 | Networking + Storage & Databases | 1–24 |
| 2 | Distributed Systems Theory | 25–38 |
| 3 | Architecture Patterns | 39–52 |
| 4 | Specialized Systems + Security + Data | 53–72 |
| 5 | System Design Case Studies | 73–90 |

---

## Month 1: Data Layer & Networking (Weeks 1–4)

### Week 1–2: Networking Essentials

#### 1. HTTP/1.1 vs HTTP/2 vs HTTP/3
**Tier:** Must Know | **File:** `content/networking/http-evolution.md`
- HTTP/1.1: persistent connections via keep-alive, pipelining, head-of-line blocking problem
- HTTP/2: multiplexing multiple streams over one TCP connection, binary framing, HPACK header compression, server push
- HTTP/3 / QUIC: built on UDP, per-stream congestion control eliminates TCP HOL blocking, 0-RTT handshake, connection migration
- Practical implications: when API servers, browsers, and mobile clients benefit from each version
- How CDN edge nodes and reverse proxies handle protocol termination and translation

#### 2. Reverse Proxy vs Forward Proxy
**Tier:** Must Know | **File:** `content/networking/reverse-proxy.md`
- Forward proxy: sits in front of clients, used for anonymity, content filtering, and corporate egress control
- Reverse proxy: sits in front of servers, handles SSL termination, load balancing, caching, and compression
- Nginx and HAProxy as production-grade reverse proxies: configuration patterns
- API Gateway as a specialized reverse proxy with routing, authentication, rate limiting, and request aggregation
- When to explicitly include each in a system design diagram and what responsibility it owns

#### 3. TCP vs UDP
**Tier:** Must Know | **File:** `content/networking/tcp-vs-udp.md`
- TCP: 3-way handshake, reliable ordered delivery, flow control (receive window), congestion control (slow start, AIMD)
- UDP: connectionless, no delivery guarantee, no ordering, minimal overhead
- Use cases: TCP for APIs, file transfer, database queries; UDP for DNS, video streaming, gaming, VoIP
- TCP head-of-line blocking vs UDP's lack of ordering guarantees — why QUIC chose UDP
- Half-open connections, TIME_WAIT state, and why they matter at scale

#### 4. DNS
**Tier:** Must Know | **File:** `content/networking/dns.md`
- Resolution chain: recursive resolver → root nameserver → TLD → authoritative nameserver
- Record types: A (IPv4), AAAA (IPv6), CNAME (alias), MX (mail), TXT (verification), SRV (service), NS (delegation)
- TTL and caching: trade-off between freshness and load on authoritative servers
- GeoDNS: returning different IP addresses based on client location for geo-routing
- DNS-based load balancing and failover: weighted routing, health-check-based failover

#### 5. Load Balancing
**Tier:** Must Know | **File:** `content/networking/load-balancing.md`
- L4 (transport layer): routes by IP + TCP port, fast, no content inspection, stateful TCP connections
- L7 (application layer): routes by URL, headers, cookies; supports sticky sessions, A/B routing, content-based routing
- Algorithms: round-robin, least connections, IP hash, consistent hashing, weighted round-robin
- Health checks: active (synthetic probes) vs passive (monitoring real traffic errors), grace periods for deploys
- Global load balancing: Anycast routing, GeoDNS, GSLB (Global Server Load Balancing) for multi-region

#### 6. CDN
**Tier:** Must Know | **File:** `content/networking/cdn.md`
- Edge caching: content cached at Points of Presence (PoPs) close to users, served without hitting origin
- Cache invalidation: TTL expiry, explicit purge APIs, cache-busting via versioned URLs (e.g., `main.abc123.js`)
- Pull CDN vs push CDN: pull fetches on first miss, push requires pre-loading (suitable for large files)
- Dynamic content acceleration: TCP connection reuse, route optimization, TLS session resumption
- CDN for video streaming: HLS/DASH segment caching, multi-bitrate variant selection at the edge

#### 7. WebSockets vs Long Polling vs SSE
**Tier:** Must Know | **File:** `content/networking/realtime-transport.md`
- Long polling: client sends request, server holds it until data is ready or timeout, then client reconnects
- SSE (Server-Sent Events): one-way server→client stream over HTTP, auto-reconnect, HTTP/2 friendly, text-only
- WebSockets: full-duplex bidirectional persistent TCP connection, binary or text frames, requires upgrade handshake
- When to use each: SSE for notifications/feeds, WebSockets for chat/collaboration, long polling as fallback
- Scaling implications: WebSocket connections are stateful, require sticky sessions or external pub/sub to route messages

#### 8. REST vs gRPC vs GraphQL
**Tier:** Must Know | **File:** `content/networking/api-protocols.md`
- REST: stateless, resource-oriented, HTTP verbs, JSON, human-readable, wide tooling support
- gRPC: binary Protocol Buffers, bidirectional streaming, strongly typed contracts, HTTP/2, lower overhead
- GraphQL: client specifies exactly what fields it needs, single endpoint, solves over/under-fetching, schema introspection
- When to choose: REST for public-facing APIs, gRPC for internal microservices, GraphQL for mobile BFF or multi-client APIs
- Trade-offs: payload size, latency, caching (REST easy, GraphQL hard), type safety, developer experience

---

### Week 3–4: Storage & Databases

#### 9. Database Indexes
**Tier:** Must Know | **File:** `content/design-concepts/storage/database-indexes.md`
- B-tree index: balanced tree structure, O(log n) reads, supports range queries and ORDER BY, default in most RDBMS
- Hash index: O(1) equality lookups, no range queries, suitable for exact-match access patterns only
- Composite indexes: column order determines which queries benefit; leading column must appear in WHERE clause
- Covering index: all queried columns are in the index, eliminates the table heap fetch ("index-only scan")
- Trade-offs: indexes speed reads but slow writes (insert/update must maintain all indexes); when NOT to index

#### 10. RDBMS Internals
**Tier:** Must Know | **File:** `content/design-concepts/storage/rdbms-internals.md`
- ACID: atomicity via undo logs, consistency via constraints, isolation via MVCC, durability via WAL
- MVCC (Multi-Version Concurrency Control): each transaction sees a snapshot, readers don't block writers
- WAL (Write-Ahead Log): changes written to log before data pages; enables crash recovery and streaming replication
- Buffer pool / page cache: dirty pages batched and flushed asynchronously; checkpointing strategy
- Query planner: how EXPLAIN works, index selection heuristics, hash join vs nested loop vs merge join

#### 11. SQL vs NoSQL Decision Framework
**Tier:** Must Know | **File:** `content/design-concepts/storage/sql-vs-nosql.md`
- Use SQL when: ACID transactions required, complex joins, well-defined schema, ad-hoc reporting, moderate scale
- Use NoSQL when: horizontal write scale needed, flexible/sparse schema, simple access patterns by key, high ingestion rate
- Not a binary choice: polyglot persistence — use both in one system for different data
- Common anti-pattern: choosing NoSQL because "it scales" without understanding access patterns; creates a distributed monolith
- NewSQL (CockroachDB, Spanner): ACID + horizontal scale; when this is the right middle ground

#### 12. Key-Value Stores (Redis)
**Tier:** Must Know | **File:** `content/design-concepts/storage/key-value-stores.md`
- Data structures: strings, lists, sets, sorted sets (ZSETs), hashes, bitmaps, HyperLogLog, streams
- Persistence: RDB snapshots (point-in-time, compact) vs AOF (every write, large), hybrid mode
- Replication and clustering: Redis Sentinel for HA failover; Redis Cluster with hash slots for horizontal scale
- Use cases: session store, rate limiting counters, pub/sub, distributed locks (SETNX), leaderboards (ZSET), queues
- Memory management: eviction policies (volatile-lru, allkeys-lru, allkeys-lfu); maxmemory configuration

#### 13. Wide-Column Stores (Cassandra)
**Tier:** Must Know | **File:** `content/design-concepts/storage/wide-column-stores.md`
- Data model: partition key (determines node), clustering key (sort order within partition), column values
- Distribution: consistent hashing ring with virtual nodes for even data distribution
- Consistency levels: ONE, QUORUM, ALL — trade-off between latency and consistency
- Compaction strategies: SizeTieredCompaction (write-heavy) vs LeveledCompaction (read-heavy, space-efficient)
- Design patterns: time-series data, write-heavy workloads; hot partition avoidance via partition key design

#### 14. Document Stores (MongoDB)
**Tier:** Good to Know | **File:** `content/design-concepts/storage/document-stores.md`
- BSON documents, schemaless collections, rich nested data model without joins
- Indexes: single-field, compound, multikey (arrays), text, geospatial (2dsphere)
- Aggregation pipeline: $match, $group, $lookup (joins), $unwind, $project for complex transformations
- Replication: replica set with automatic primary election and oplog-based replication
- Sharding: shard key selection (range vs hash), chunk migration, hotspot avoidance

#### 15. Object Storage (S3-like)
**Tier:** Must Know | **File:** `content/design-concepts/storage/object-storage.md`
- Architecture: flat namespace (bucket + key), no real hierarchy; metadata stored separately from object data
- Multipart upload: split large files into parts, upload in parallel, server assembles; enables resume on failure
- Consistency model: strong read-after-write for new objects; eventual consistency for list operations on large buckets
- Durability and replication: erasure coding across availability zones, 11 nines durability
- Use cases vs trade-offs: user-uploaded media, backups, data lakes, static assets; high latency vs low cost vs unlimited scale

#### 16. Time-Series Databases
**Tier:** Good to Know | **File:** `content/design-concepts/storage/time-series-databases.md`
- Characteristics: append-mostly writes, time-ordered data, high cardinality tag sets, aggregation-heavy reads
- Storage optimizations: delta-of-delta encoding for timestamps, Gorilla compression for float values, run-length encoding
- Retention policies: automatic expiry of old data; downsampling — roll up 1-second data into 1-minute averages
- Query patterns: range queries over time, aggregations (avg/max/sum), rate-of-change, percentile calculations
- Tools: InfluxDB, TimescaleDB (Postgres extension), Prometheus; when a regular RDBMS with a time index is enough

#### 17. Consistent Hashing
**Tier:** Must Know | **File:** `content/design-concepts/storage/consistent-hashing.md`
- The problem: naive modulo hashing (`key % N`) causes O(K) key moves when N changes; unacceptable for large clusters
- Hash ring: nodes placed at positions on a circular hash space; key maps to nearest node clockwise
- Virtual nodes: each physical node owns multiple ring positions for even distribution and smoother rebalancing
- Applications: distributed caches (Memcached), Cassandra token ring, load balancers for session affinity
- Hotspot mitigation: virtual nodes help, but explicit shard-splitting needed for truly hot keys

#### 18. Database Sharding
**Tier:** Must Know | **File:** `content/design-concepts/scaling/sharding.md`
- Hash-based sharding: even distribution, O(1) key lookup, cross-shard range queries require scatter-gather
- Range-based sharding: supports range queries on shard key, risk of hotspots on monotonically increasing keys
- Directory-based sharding: lookup table maps key range to shard, flexible but directory is a bottleneck/SPOF
- Resharding strategies: consistent hashing for minimal movement, double-write migration, online split tooling
- Cross-shard operations: scatter-gather for reads, distributed transactions for writes, denormalization to avoid

#### 19. Bloom Filters
**Tier:** Must Know | **File:** `content/design-concepts/storage/bloom-filters.md`
- Probabilistic membership data structure: answers "definitely not in set" or "probably in set" (false positives exist)
- Tuning: false positive rate controlled by bit array size (m) and number of hash functions (k); formula `k = (m/n) ln 2`
- Use cases: cache negative-lookup prevention (avoid DB hit for non-existent keys), Cassandra SSTable lookups, URL deduplication in crawlers
- Counting Bloom Filter: supports deletions by using counters instead of bits, higher memory cost
- HyperLogLog: related probabilistic structure for approximate cardinality counting (how many unique elements?)

#### 20. Caching Patterns
**Tier:** Must Know | **File:** `content/design-concepts/storage/caching-patterns.md`
- Cache-aside (lazy loading): app checks cache → on miss, loads from DB → writes to cache; cache only holds accessed data
- Write-through: write to cache and DB synchronously on every write; cache always consistent, higher write latency
- Write-back (write-behind): write to cache only, async flush to DB; low write latency, risk of data loss on crash
- Write-around: writes bypass cache, go directly to DB; cache populated on reads only; good for write-once data
- Cache warming: pre-populating cache before traffic; probabilistic early expiry (jitter) to prevent simultaneous expiry

#### 21. Cache Eviction
**Tier:** Must Know | **File:** `content/design-concepts/storage/cache-eviction.md`
- LRU (Least Recently Used): doubly-linked list + hash map, O(1) get/put; evicts item not accessed longest
- LFU (Least Frequently Used): tracks access count, evicts least frequently accessed; better for skewed access patterns
- ARC (Adaptive Replacement Cache): adapts dynamically between LRU and LFU based on workload; used in ZFS
- Cache stampede / thundering herd: many requests hit DB simultaneously when a popular key expires; mitigation via mutex lock or probabilistic early expiry
- FIFO, Random eviction: simpler alternatives with specific use cases (e.g., CDN with uniform content freshness)

#### 22. Hot-Key / Hotspot Problems
**Tier:** Must Know | **File:** `content/design-concepts/storage/hotspot-problems.md`
- What causes hotspots: viral content, celebrity accounts, trending topics — single key receives disproportionate traffic
- In caching: single Redis node overwhelmed; solutions: key replication (key → key-1, key-2, key-N with random selection), local in-process L1 cache
- In databases: hot partition in Cassandra or DynamoDB when partition key maps to same shard; solution: add random suffix to partition key, then aggregate on read
- In Kafka: hot partition when message key is skewed; solution: repartition, custom partitioner, or scatter writes
- Detection: per-key access counters, percentile latency spikes on specific nodes, node CPU imbalance

#### 23. Full-Text Search
**Tier:** Must Know | **File:** `content/design-concepts/storage/full-text-search.md`
- Inverted index: maps each term to a posting list (list of document IDs containing that term); core data structure
- TF-IDF and BM25: scoring algorithms for relevance ranking — term frequency, inverse document frequency, field length normalization
- Elasticsearch internals: Lucene segments (immutable), segment merging, near-real-time refresh interval
- Index sharding in Elasticsearch: primary shards (write), replica shards (read HA); shard count fixed at creation
- Search relevance tuning: field boosting, custom scoring functions, fuzzy matching, synonyms, query-time analysis

#### 24. Read Replicas & Replication Lag
**Tier:** Must Know | **File:** `content/design-concepts/replication/read-replicas.md`
- Synchronous replication: primary waits for at least one replica to confirm before ack; durable but higher write latency
- Asynchronous replication: primary acks immediately, replica catches up; faster writes, risk of replication lag
- Replication lag: seconds or more behind primary under heavy write load; causes stale reads
- Read-your-writes consistency: routing reads to primary after a write, or using sticky routing to the replica that confirmed
- When replication lag causes bugs: reading stale counts, stale session state, recommendation feeds showing old data

---

## Month 2: Distributed Systems (Weeks 5–8)

### Week 5–6: Theory

#### 25. CAP Theorem
**Tier:** Must Know | **File:** `content/design-concepts/cap-theorem.md`
- The three properties: Consistency (every read sees latest write), Availability (every request gets a non-error response), Partition Tolerance (system works despite network partitions)
- Why you can only have two during a partition: if partitioned, you must choose to stay consistent (reject requests) or available (return possibly stale data)
- CP systems: ZooKeeper, HBase, etcd — prefer consistency, may be unavailable during partition
- AP systems: Cassandra, DynamoDB, CouchDB — prefer availability, may return stale data during partition
- Common misconceptions: CAP applies only during partitions, not normal operation; "CA" systems don't exist in distributed settings

#### 26. PACELC
**Tier:** Good to Know | **File:** `content/design-concepts/pacelc.md`
- Extends CAP: during Partitions choose A or C; Else (normal operation) choose Latency or Consistency
- Real-world examples: DynamoDB (PA/EL — available under partition, low latency else), Spanner (PC/EC — consistent always, higher latency)
- Why PACELC is more useful in practice: systems are rarely partitioned; the L vs C trade-off happens every request
- Tunable consistency: Cassandra lets you choose per-operation (ONE vs QUORUM vs ALL), sliding L/C trade-off
- Applying PACELC in design interviews: frame storage choices as L/C trade-offs, not just CP vs AP

#### 27. Consistency Models
**Tier:** Must Know | **File:** `content/design-concepts/replication/consistency-models.md`
- Linearizability (strong consistency): behaves as if single copy, operations have global total order, most expensive
- Sequential consistency: each process's operations appear in program order, but no global wall-clock ordering
- Causal consistency: causally related writes are seen in order by all nodes; concurrent writes may be seen in different orders
- Eventual consistency: all replicas converge to the same value eventually if writes stop; weakest model
- Session guarantees: read-your-writes, monotonic reads, monotonic writes — intermediate guarantees useful in practice

#### 28. Idempotency & Exactly-Once Semantics
**Tier:** Must Know | **File:** `content/design-concepts/distributed/idempotency.md`
- Idempotency: applying the same operation N times produces the same result as once; essential for safe retries
- Idempotency keys: client generates unique key per request; server stores result keyed by it, returns cached result on duplicate
- Delivery guarantees: at-most-once (may lose), at-least-once (may duplicate), exactly-once (requires coordination)
- Exactly-once in Kafka: idempotent producers (sequence numbers) + transactional API for atomic multi-partition writes
- Deduplication patterns: Bloom filter for fast negative check, DB unique constraint, Redis SETNX with TTL

#### 29. Distributed Locking
**Tier:** Must Know | **File:** `content/design-concepts/distributed/distributed-locking.md`
- Redis SETNX with expiry: `SET key value NX PX ttl` — simple, works for single Redis node
- Redlock algorithm: acquire lock on majority of N independent Redis nodes; quorum-based, handles single node failure
- Fencing tokens: lock server issues monotonically increasing token; resource server rejects stale tokens — prevents stale lock issues
- ZooKeeper ephemeral sequential znodes: built-in leader election and lock semantics, stronger guarantees than Redis
- When not to use distributed locks: design around idempotency instead; locks add latency and complexity

#### 30. Leader Election
**Tier:** Good to Know | **File:** `content/design-concepts/distributed/leader-election.md`
- Bully algorithm: node with highest ID claims leadership by broadcasting; simple but generates O(n²) messages
- Raft leader election: randomized election timeout, candidates request votes, winner must have majority and latest log
- ZooKeeper-based election: ephemeral sequential znodes; smallest znode is leader; watches notify on leader failure
- Practical systems: Kafka controller (ZooKeeper then KRaft), Elasticsearch master node, etcd leader
- Split-brain prevention: fencing tokens, epoch numbers, STONITH (Shoot The Other Node In The Head)

#### 31. Gossip Protocol
**Tier:** Good to Know | **File:** `content/design-concepts/distributed/gossip-protocol.md`
- Fan-out model: each node selects N random peers per round and exchanges state; O(log N) rounds to reach all nodes
- Uses: failure detection (Cassandra), cluster membership (Consul, Serf), metadata propagation, anti-entropy
- Convergence properties: probabilistic eventual consistency; resistant to node failures during propagation
- Phi accrual failure detector: adaptive threshold based on historical heartbeat intervals; more accurate than fixed timeouts
- Anti-entropy repair: full Merkle tree comparison as a complement to gossip; repairs diverged replicas

#### 32. Quorum Reads & Writes
**Tier:** Must Know | **File:** `content/design-concepts/distributed/quorum.md`
- Quorum rule: R + W > N guarantees at least one node in any read set overlaps with any write set
- Tuning: W = N (all replicas) for max durability; R = 1 for max read performance; QUORUM balances both
- Sloppy quorum: during partition, accept writes on any available nodes (not the designated ones); hinted handoff on recovery
- Read repair: when reading from R replicas, check for staleness and repair in background or synchronously
- Examples: Cassandra (tunable per operation), DynamoDB (eventually consistent vs strongly consistent reads), Riak

---

### Week 7–8: Distributed Transactions & Coordination

#### 33. Two-Phase Commit (2PC)
**Tier:** Must Know | **File:** `content/design-concepts/consensus/two-phase-commit.md`
- Phase 1 (Prepare): coordinator sends prepare to all participants; each locks resources and votes yes/no
- Phase 2 (Commit/Abort): if all voted yes, coordinator sends commit; any no triggers abort to all
- Failure modes: coordinator crash after prepare leaves participants blocked waiting (blocking protocol)
- Recovery: coordinator uses persistent log to replay decision after restart; participants wait or timeout and abort
- When to use vs alternatives: use 2PC within a single database system; prefer Saga for cross-service transactions

#### 34. Saga Pattern
**Tier:** Must Know | **File:** `content/design-concepts/distributed/saga-pattern.md`
- Long-running transactions across services without holding distributed locks; each step has a compensating action
- Choreography: each service publishes events and listens for others' events; decentralized, harder to trace
- Orchestration: central Saga orchestrator sends commands to each service and handles failures; easier to monitor
- Compensating transactions: must be idempotent and always succeed; "undo" the effect of a completed step
- Saga execution log: coordinator persists state so it can resume or compensate on restart after crash

#### 35. Outbox Pattern
**Tier:** Must Know | **File:** `content/design-concepts/distributed/outbox-pattern.md`
- The dual write problem: you can't atomically update a DB and publish an event to Kafka in the same transaction
- Solution: write the event to an outbox table in the same DB transaction as the business data; separate process publishes it
- CDC (Change Data Capture): Debezium reads DB binlog/WAL and publishes outbox rows to Kafka automatically
- At-least-once delivery: consumers must be idempotent since the publisher may retry on failure
- Inbox pattern: consumer stores received event IDs to deduplicate at-least-once delivery on the consumer side

#### 36. Raft Consensus (Conceptual)
**Tier:** Good to Know | **File:** `content/design-concepts/consensus/raft.md`
- Roles: leader (handles all writes), follower (replicates), candidate (during election)
- Leader election: randomized election timeouts prevent simultaneous candidates; term numbers prevent stale leaders
- Log replication: leader appends entry, sends AppendEntries RPC to followers, commits once majority confirms
- Safety: committed entries never lost (leader completeness property); at most one leader per term
- Real-world systems: etcd (Kubernetes), CockroachDB, TiKV, Kafka KRaft mode

#### 37. Logical Clocks
**Tier:** Good to Know | **File:** `content/design-concepts/distributed/logical-clocks.md`
- Lamport timestamps: monotonically increasing counter; on send, increment; on receive, max(local, received) + 1
- Vector clocks: per-process counter array; detects causality and concurrent events; used in Riak, Dynamo
- Version vectors: similar to vector clocks but attached to data objects for conflict detection (vs. conflict resolution)
- Hybrid Logical Clocks (HLC): combines physical clock (NTP) with logical clock; monotonic and close to wall time
- When to use: conflict-free replicated data (CRDTs), causal messaging, debugging distributed event ordering

#### 38. Multi-Region Design & Geo-Replication
**Tier:** Must Know | **File:** `content/design-concepts/distributed/multi-region-design.md`
- Active-passive: traffic served from primary region, secondary is warm standby for DR; simpler, higher RTO
- Active-active: traffic served from multiple regions simultaneously; lower latency globally, harder to keep consistent
- Geo-routing: GeoDNS or Anycast directs users to nearest region; latency reduction of 50–200ms typical
- Cross-region replication lag: async replication introduces seconds of lag; forces choice of read-your-writes strategy
- Conflict resolution in active-active: last-write-wins (by timestamp), CRDTs for counters/sets, application-level merge; data sovereignty and compliance considerations

---

## Month 3: Architecture Patterns (Weeks 9–12)

### Week 9–10: Messaging & Event-Driven

#### 39. Message Queues vs Event Streams
**Tier:** Must Know | **File:** `content/design-concepts/messaging/queues-vs-streams.md`
- Message queue (RabbitMQ, SQS): each message consumed by one consumer; deleted after ack; built for work distribution
- Event stream (Kafka, Kinesis): persistent log; multiple consumer groups each read independently; supports replay
- When to use a queue: background jobs, task distribution, exactly-one consumer semantics (e.g., send one email)
- When to use a stream: audit log, fan-out to multiple downstream systems, event sourcing, analytics pipelines
- Hybrid patterns: Kafka for durable events, SQS for task queues derived from events; trade-off between cost and flexibility

#### 40. Kafka Deep Dive
**Tier:** Must Know | **File:** `content/design-concepts/messaging/kafka.md`
- Architecture: brokers hold partition replicas; topics partitioned for parallelism; leader handles reads+writes per partition
- Partitions: unit of ordering and parallelism; messages within one partition are ordered; across partitions they are not
- Consumer groups: each group gets all messages; within a group, each partition assigned to exactly one consumer
- Offset management: consumers commit offsets to `__consumer_offsets`; enables exactly-where-I-left-off resumption and replay
- Delivery guarantees: at-most-once (no retry), at-least-once (retry + idempotent consumer), exactly-once (producer transactions + idempotent consumer)

#### 41. Event-Driven Architecture
**Tier:** Must Know | **File:** `content/design-concepts/messaging/event-driven-architecture.md`
- Events vs commands: events are immutable facts ("OrderPlaced"), commands are requests ("PlaceOrder")
- Pub/sub: producer publishes without knowing consumers; consumers subscribe independently; loose coupling
- Event sourcing: state is derived by replaying an ordered sequence of events; enables temporal queries and audit log
- Benefits: auditability, easy fan-out to new consumers, temporal decoupling between services
- Challenges: eventual consistency, schema evolution compatibility, complex debugging (no call stack)

#### 42. CQRS
**Tier:** Must Know | **File:** `content/design-concepts/messaging/cqrs.md`
- Separate write model (commands → domain logic → events) from read model (projections optimized for queries)
- Read model is a denormalized view: pre-joined, pre-aggregated, stored in the fastest read store (Redis, Elasticsearch)
- Keeping read and write models in sync: event-driven projections — consume events and update read store
- When to use: high read/write asymmetry, complex query requirements, or read model needs different storage than write model
- Complexity: two models to maintain, eventual consistency between write and read models; only justified at scale

#### 43. Dead Letter Queues & Retry Strategies
**Tier:** Good to Know | **File:** `content/design-concepts/messaging/dlq-and-retry.md`
- DLQ: messages that fail processing after N retries are routed here for manual inspection and reprocessing
- Exponential backoff with jitter: retry after 2^n seconds + random jitter; avoids synchronized retry storms
- Poison message detection: messages that always cause consumer crash; DLQ prevents blocking the main queue
- Idempotent consumers: consumers must handle duplicate delivery safely; use unique message IDs for deduplication
- Monitoring DLQs: DLQ depth alert = production bug; reprocessing workflows after bug fix

---

### Week 11–12: Reliability & API Design

#### 44. Rate Limiting (All Algorithms)
**Tier:** Must Know | **File:** `content/design-concepts/rate-limiting/algorithms.md`
- Fixed window counter: counter per time window; boundary burst problem (2x rate at window boundary)
- Sliding window log: stores timestamp of every request; precise but high memory (O(requests) per user)
- Sliding window counter: weighted average of current and previous window counter; approximation with low memory
- Token bucket: bucket fills at rate R, max capacity B; allows bursts up to B; most flexible for real-world use
- Distributed rate limiting: Redis INCR + EXPIRE for simple counters; Lua scripts for atomic sliding window; centralized vs per-node trade-offs

#### 45. Circuit Breaker Pattern
**Tier:** Must Know | **File:** `content/design-concepts/reliability/circuit-breaker.md`
- States: closed (normal operation), open (fail fast, no real calls), half-open (probe recovery with limited traffic)
- Failure threshold: trip after X% error rate or X consecutive failures over a rolling time window
- Reset timeout: how long circuit stays open before trying half-open; exponential backoff for repeated failures
- Fallback strategies: cached response, static default value, degraded feature, error message — must be explicit
- Libraries and placement: Resilience4j, Polly; sits between service client and HTTP call; works with timeouts and bulkheads

#### 46. Bulkhead Pattern
**Tier:** Good to Know | **File:** `content/design-concepts/reliability/bulkhead.md`
- Named after ship bulkheads: isolate compartments so one failure doesn't sink the ship
- Thread pool bulkhead: separate thread pool per downstream dependency; slow DB won't exhaust all application threads
- Semaphore bulkhead: limit concurrent calls per dependency without a separate thread pool; lower overhead
- Practical example: premium user requests get dedicated pool; free tier requests can be shed without affecting premium
- Combined with circuit breaker: bulkhead limits concurrency, circuit breaker prevents calls when downstream is down

#### 47. Back-Pressure & Load Shedding
**Tier:** Good to Know | **File:** `content/design-concepts/reliability/back-pressure.md`
- Back-pressure: downstream slow consumer signals upstream producer to slow down; prevents unbounded buffer growth
- Reactive Streams: standardized protocol for back-pressure propagation in async pipelines (Java Flow, RxJava)
- Load shedding: intentionally drop requests when system is overloaded rather than queuing indefinitely
- Priority-based shedding: shed lowest-priority traffic first (health checks, analytics) to protect core operations
- Admission control: reject requests at the API gateway level before they reach services; fail fast at the boundary

#### 48. API Pagination
**Tier:** Must Know | **File:** `content/design-concepts/api/pagination.md`
- Offset pagination: `LIMIT n OFFSET m`; simple but slow on large offsets (DB must scan and discard m rows)
- Cursor-based pagination: encode last-seen item identifier as opaque cursor; stable, efficient, no deep scan problem
- Keyset pagination: `WHERE id > last_id LIMIT n`; database-friendly, uses index; requires sorted unique key
- Page size trade-offs: large pages reduce round trips but increase latency and payload; typical default 20–100 items
- Consistency during pagination: items added or deleted between pages cause gaps or duplicates; cursor pagination helps but doesn't fully solve it

#### 49. API Idempotency Keys & Versioning
**Tier:** Must Know | **File:** `content/design-concepts/api/idempotency-and-versioning.md`
- Idempotency keys: client generates UUID per logical operation; server stores (key → response) with TTL; returns cached response on duplicate
- Storage for idempotency: Redis with TTL (fast, volatile) or DB row (durable); trade-off based on risk
- API versioning strategies: URL path (`/v1/`), header (`API-Version: 2`), content negotiation (`Accept: application/vnd.api.v2+json`)
- Breaking vs non-breaking changes: adding fields is safe; removing, renaming, or changing types is breaking
- Sunset and deprecation: `Sunset` response header, deprecation notices, migration guides, grace periods before removal

#### 50. Capacity Estimation
**Tier:** Must Know | **File:** `content/design-concepts/capacity-estimation.md`
- Latency numbers: L1 cache 1ns, RAM 100ns, SSD random read 100µs, network roundtrip 1ms, HDD seek 10ms
- Estimation framework: DAU × avg actions/day → requests/second → storage growth/day → total storage in 5 years
- Peak load: size for 2–3× average QPS; understand daily and weekly traffic patterns
- Storage math: text (bytes), images (100KB–1MB), video (1MB/s for HD), audio (1MB/min); compression ratios
- Practical example walkthrough: Twitter — 300M DAU, 100 tweets/day read → 300K tweets/s read QPS; storage estimation

#### 51. Microservices vs Monolith
**Tier:** Good to Know | **File:** `content/design-concepts/architecture/microservices-vs-monolith.md`
- Monolith benefits: simple deployment, no network overhead, easy cross-module transactions, simple debugging
- Microservices benefits: independent deployability, team autonomy, isolated failure domains, per-service scaling
- Distributed monolith anti-pattern: split into services but tightly coupled; worst of both worlds
- Decomposition strategies: by business capability (order service, inventory service), by data domain (DDD bounded contexts)
- When NOT to split: early-stage products, small teams, lack of operational maturity; start monolith, extract services when clear boundary emerges

#### 52. Service Discovery & API Gateway
**Tier:** Good to Know | **File:** `content/design-concepts/architecture/service-discovery.md`
- Client-side discovery: service queries registry (Consul, Eureka) and load balances itself; adds client complexity
- Server-side discovery: load balancer queries registry and routes; simpler clients, single routing point
- API Gateway: single entry point; handles routing, auth, rate limiting, SSL termination, protocol translation, request aggregation
- BFF (Backend for Frontend): tailored gateway per client type (mobile app, web app, third-party); reduces over-fetching
- Service mesh vs API gateway: service mesh (sidecar per pod, east-west traffic) vs gateway (north-south traffic, external-facing)

---

## Month 4: Specialized Systems + Security + Data (Weeks 13–16)

### Week 13: Search & Geospatial

#### 53. Search Autocomplete / Typeahead
**Tier:** Must Know | **File:** `content/design-concepts/specialized/typeahead.md`
- Trie data structure: compact prefix tree; each node is a character; leaf nodes store completions; memory-heavy at scale
- Top-k with prefix: each trie node stores top-k completions by frequency; avoids full subtree traversal on query
- Redis sorted sets: store (term, score=frequency) pairs; ZRANGEBYLEX for prefix matching; simpler than trie at moderate scale
- Data pipeline: aggregate search query logs offline → compute top-k per prefix → load into cache; refresh periodically
- Personalization layer: blend global popularity ranking with user-specific history at query time

#### 54. GeoHash
**Tier:** Must Know | **File:** `content/design-concepts/specialized/geohash.md`
- Encodes lat/lng as a string: each additional character halves the cell area; 6 chars ≈ 1km², 8 chars ≈ 38m²
- Prefix property: strings sharing a longer prefix are geographically closer (with edge case at cell boundaries)
- Proximity queries: compute 8 neighboring GeoHash cells at target precision + target cell = 9-cell search area
- Redis GEO commands: GEOADD stores members with coordinates; GEORADIUS returns members within radius
- Limitations: uneven cell sizes at high latitudes; edge cases at cell boundaries require neighbor cell queries

#### 55. QuadTree
**Tier:** Must Know | **File:** `content/design-concepts/specialized/quadtree.md`
- 2D spatial partitioning: recursively divide a 2D area into 4 equal quadrants; each node is a region
- Dynamic subdivision: a leaf node splits into 4 children when it holds more than K points (e.g., K=100)
- Lookup: traverse tree from root, follow quadrant containing query point, collect results from leaf node
- Use cases: Yelp/nearby search, map rendering level-of-detail, collision detection in games
- QuadTree vs GeoHash: QuadTree adapts to data density (dense areas subdivide more), GeoHash has fixed-size cells

#### 56. Uber-Style Location Indexing
**Tier:** Good to Know | **File:** `content/design-concepts/specialized/location-indexing.md`
- Write pattern: driver app sends GPS update every 4 seconds; millions of concurrent drivers → very high write rate
- Read pattern: find all available drivers within radius of a pickup point; low latency required (<100ms)
- Design: hash driver location to GeoHash cell → store in Redis sorted set keyed by cell → range query cells in radius
- Alternative: H3 hexagonal grid (Uber's production system); more uniform cell areas than square GeoHash cells
- Stale location cleanup: TTL on driver entries in Redis; heartbeat resets TTL; expired = offline

---

### Week 14: Real-Time & Notifications

#### 57. Presence & Online Status Systems
**Tier:** Must Know | **File:** `content/design-concepts/specialized/presence-system.md`
- Heartbeat approach: client sends heartbeat every 5s; server marks offline if no heartbeat for 15s
- Last-seen storage: Redis hash per user (user_id → last_seen_timestamp); read on profile/chat load
- Fan-out challenge: when user A comes online, notify all of A's contacts; expensive for users with many contacts
- Scaling: connection server tracks local connections; publishes state changes to pub/sub layer (Redis pub/sub or Kafka) for cross-server routing
- Consistency trade-off: strong consistency (all see same state instantly) vs eventual (acceptable for presence; small lag is fine)

#### 58. Notification Fanout Strategies
**Tier:** Must Know | **File:** `content/design-concepts/specialized/notification-fanout.md`
- Fan-out on write (push model): when post created, immediately write to all followers' notification inboxes; fast reads, slow/expensive writes for popular users
- Fan-out on read (pull model): compute notifications at read time by querying who followed; fast writes, slow reads
- Hybrid: fan-out on write for regular users (< 10K followers); fan-out on read for celebrities (> 10K followers)
- Push delivery: APNs (iOS), FCM (Android), Web Push (browsers); handle device token management and expiry
- Deduplication and rate limiting: user should not receive duplicate notifications; cap notification frequency per user

#### 59. WebSocket at Scale
**Tier:** Must Know | **File:** `content/design-concepts/specialized/websocket-at-scale.md`
- Connection management: each server holds N open WebSocket connections in memory; connections are stateful
- Sticky sessions: load balancer routes same user to same WebSocket server (cookie or IP hash); needed for stateful connections
- Cross-server routing: when user A (server 1) sends to user B (server 2), use Redis pub/sub or Kafka to route message
- Horizontal scaling: store session/connection metadata in Redis; allows stateless WebSocket server replicas
- Heartbeat and reconnection: server sends ping frames; client sends pong; on drop, client reconnects with exponential backoff

#### 60. MQTT
**Tier:** Optional | **File:** `content/design-concepts/specialized/mqtt.md`
- Lightweight publish-subscribe protocol over TCP; designed for constrained IoT devices with limited bandwidth
- QoS levels: 0 (fire-and-forget), 1 (at-least-once with ack), 2 (exactly-once with 4-step handshake)
- Retained messages: broker stores last message per topic; new subscribers immediately get current state
- Broker clustering: EMQX, HiveMQ, VerneMQ support horizontal scaling; bridge brokers across regions
- When to use vs WebSockets: MQTT for IoT devices with poor connectivity, battery constraints; WebSockets for browser clients

---

### Week 15: Payments, Scheduling & IDs

#### 61. Payment System Design
**Tier:** Must Know | **File:** `content/design-concepts/specialized/payment-systems.md`
- Double-entry bookkeeping: every debit has a matching credit; ledger is append-only and immutable
- Idempotency: payment retries must not double-charge; use idempotency key tied to client-side payment intent ID
- Distributed transaction challenge: updating internal ledger + calling external payment processor atomically; use outbox pattern
- Reconciliation: periodic job compares internal ledger against payment processor records; detects and alerts discrepancies
- PCI DSS scope reduction: never store raw card data; use tokenization (Stripe/Braintree stores card, gives you a token)

#### 62. Distributed Job Scheduling
**Tier:** Must Know | **File:** `content/design-concepts/specialized/job-scheduling.md`
- Delayed queues: Redis sorted set with score = execution timestamp; worker polls for jobs due now (ZRANGEBYSCORE)
- At-least-once delivery: job may run more than once on worker crash; consumer must be idempotent
- Job leasing: worker acquires lease (lock with TTL) on job; if worker dies, lease expires and another worker retries
- Cron-style scheduling: leader-based distributed cron; only leader fires the job; Kubernetes CronJob for containerized workloads
- Fair scheduling across tenants: per-tenant quotas, priority queues, weighted fair queuing to prevent noisy neighbors

#### 63. Unique ID Generation
**Tier:** Must Know | **File:** `content/design-concepts/specialized/id-generation.md`
- UUID v4: 128-bit random; no coordination required; not sortable; index fragmentation in B-tree; 36-char string
- Snowflake ID (Twitter): 64-bit integer; 41-bit timestamp + 10-bit machine ID + 12-bit sequence; millisecond-sortable; 4096 IDs/ms per machine
- ULID: 128-bit; 48-bit timestamp + 80-bit random; lexicographically sortable; URL-safe Base32 encoded
- Database sequences: simple, strongly ordered, but single point of coordination; works well for single-region
- Trade-offs to discuss: sortability (B-tree index friendliness), coordination overhead, length, global uniqueness guarantees

---

### Week 16: Security + Data Engineering Basics

#### 64. OAuth 2.0 & OIDC
**Tier:** Must Know | **File:** `content/design-concepts/security/oauth-oidc.md`
- OAuth 2.0 grant types: authorization code (user-facing apps), client credentials (service-to-service), device flow (TVs/CLIs)
- PKCE (Proof Key for Code Exchange): prevents auth code interception attacks in public clients (mobile/SPA)
- Token pattern: short-lived access token (15min) + long-lived refresh token (30 days); access token authorizes API calls
- OIDC layer: adds ID token (JWT with user identity claims) on top of OAuth 2.0; `openid` scope triggers it
- Token revocation: access tokens are self-contained (hard to revoke); refresh token revocation via server-side blocklist

#### 65. JWT (JSON Web Tokens)
**Tier:** Must Know | **File:** `content/design-concepts/security/jwt.md`
- Structure: `header.payload.signature`; each section is Base64URL encoded; signature prevents tampering
- Signing algorithms: HS256 (shared HMAC secret — symmetric), RS256 (RSA key pair — asymmetric, preferred for distributed systems)
- Standard claims: `iss` (issuer), `sub` (subject), `exp` (expiry), `iat` (issued at), `aud` (audience)
- Stateless auth: server validates signature without DB lookup; enables horizontal scaling without shared session store
- Pitfalls: cannot revoke before expiry without a blocklist; `alg: none` attack; key confusion between HS256 and RS256

#### 66. API Authentication Patterns
**Tier:** Good to Know | **File:** `content/design-concepts/security/api-auth.md`
- API keys: opaque shared secret in header; simple for server-to-server; store hashed, treat like passwords
- HMAC signing: client signs `method + path + timestamp + body hash` with shared secret; server verifies; prevents replay attacks with timestamp window
- mTLS (mutual TLS): both client and server present certificates; strong identity verification for internal microservices
- JWT Bearer tokens: client presents signed token in `Authorization: Bearer` header; server validates locally without DB call
- Choosing the right scheme: public APIs → API keys + OAuth; internal services → mTLS or JWT with short expiry; third-party integrations → OAuth 2.0 client credentials

#### 67. Batch vs Streaming
**Tier:** Good to Know | **File:** `content/design-concepts/data/batch-vs-streaming.md`
- Batch processing: processes bounded datasets on a schedule; high throughput, minutes-to-hours latency, simpler fault tolerance
- Stream processing: processes unbounded event streams in near-real-time; millisecond-to-second latency, complex exactly-once guarantees
- Lambda architecture: batch layer for accuracy + speed layer for low latency + serving layer merges both; operational complexity
- Kappa architecture: single streaming pipeline replaces both layers; re-processes historical data by replaying the event log
- When to choose: use batch for nightly reports, ML training; use streaming for fraud detection, real-time analytics, live dashboards

#### 68. Data Warehouse Basics
**Tier:** Good to Know | **File:** `content/design-concepts/data/data-warehouse.md`
- OLTP vs OLAP: OLTP optimized for transactions (row store, point reads); OLAP optimized for analytics (columnar, full scans)
- Columnar storage: stores each column separately; better compression (similar values together), faster aggregation scans, slower for row inserts
- Star schema: central fact table (events/transactions) surrounded by dimension tables (users, products, time); optimized for analytical queries
- Materialized views: precomputed query results stored as tables; trades storage and staleness for query speed
- Managed services: Snowflake (auto-scaling, storage-compute separation), BigQuery (serverless, pay-per-query), Redshift (columnar, tight AWS integration)

#### 69. Change Data Capture (CDC)
**Tier:** Good to Know | **File:** `content/design-concepts/data/change-data-capture.md`
- What it does: captures every row-level INSERT/UPDATE/DELETE from a database's binary log in real-time
- Debezium: open-source connector that tails MySQL binlog or Postgres WAL and publishes changes to Kafka
- Use cases: cache invalidation (sync Redis when DB changes), search index sync (Elasticsearch), event-driven pipelines, audit logging
- Outbox + CDC combination: service writes to outbox table → Debezium publishes to Kafka → downstream consumers
- Schema evolution: forward compatibility (new consumers handle old events), backward compatibility (old consumers handle new events)

---

### AI/ML — Minimal (Role-Specific)

#### 70. Recommendation System Design
**Tier:** Good to Know | **File:** `content/design-concepts/ml/recommendation-systems.md`
- Retrieval stage: candidate generation from corpus of millions of items using ANN search or two-tower model embedding similarity
- Ranking stage: score retrieved candidates with richer features (user history, item metadata, context); gradient boosted trees or neural ranker
- Feature engineering: user features (history, demographics), item features (category, popularity), context features (time, device, location)
- A/B testing: online experiments with metric selection (CTR, dwell time, conversion); statistical significance and novelty effect
- Cold start problem: new users (popular/trending items), new items (content-based features before interaction data exists)

#### 71. Feature Store Design
**Tier:** Optional | **File:** `content/design-concepts/ml/feature-store.md`
- Online store: serves precomputed features at low latency for real-time inference (Redis, DynamoDB, Cassandra)
- Offline store: historical feature snapshots for model training (data lake in Parquet, columnar format)
- Point-in-time correctness: training data must only use features available at prediction time; prevents label leakage
- Feature freshness: how often features are computed and updated; trade-off between freshness and compute cost
- Tools: Feast (open-source), Tecton, Vertex AI Feature Store; key design: decouple feature computation from feature serving

#### 72. ML Platform Design
**Tier:** Optional | **File:** `content/design-concepts/ml/ml-platform.md`
- Training pipeline: data ingestion → feature engineering → model training → evaluation → registration
- Model registry: version control for models; stores artifacts, hyperparameters, metrics, lineage (which data was used)
- Model serving: online (low-latency REST/gRPC endpoint), batch (Spark job scoring large datasets), streaming (Kafka consumer)
- Monitoring: data drift (input distribution shifts), concept drift (model accuracy degrades), prediction confidence distribution
- Shadow mode: run new model in parallel without returning results; compare predictions to production before promoting

---

## Month 5: System Design Case Studies (Weeks 17–20)

For each case study entry, when prompting Claude/Copilot include: functional requirements, non-functional requirements (scale, latency, consistency), core design decisions (storage choice, APIs), major components with a diagram, and 2–3 deep dives on the most interesting sub-problems.

#### 73. URL Shortener
**Tier:** Must Know | **File:** `content/system-design/url-shortener.md`
- Core: hash long URL → short key (7 chars = 62^7 possibilities); store key→URL in KV store; redirect with 301 vs 302
- Scale: 100M URLs, 10B redirects/day → 100K redirect QPS; read-heavy, strong cache candidate
- Deep dives: collision handling in key generation, custom aliases, analytics (click counts per short URL), link expiry
- Key concepts: consistent hashing for distributed KV, caching (cache-aside for redirect lookups), bloom filter to avoid DB lookup for non-existent keys

#### 74. Key-Value Store (Redis-like)
**Tier:** Must Know | **File:** `content/system-design/key-value-store.md`
- Core: GET/PUT/DELETE by key; single-node → multi-node with consistent hashing; handle node failures
- Storage engine: in-memory with optional persistence (WAL + snapshots); LSM tree for disk-based variant
- Deep dives: replication protocol (leader-follower), consistent hashing with virtual nodes, eviction policy enforcement
- Key concepts: consistent hashing, LRU eviction, WAL for durability, gossip for failure detection

#### 75. Distributed Cache
**Tier:** Must Know | **File:** `content/system-design/distributed-cache.md`
- Core: cache-aside pattern, consistent hashing across cache nodes, cache invalidation strategies
- Scale: 1M QPS reads, sub-millisecond latency requirement; mostly hot data in memory
- Deep dives: cache stampede prevention, hot-key replication, handling node failures with consistent hashing rehashing
- Key concepts: consistent hashing, bloom filter (negative cache), LRU/LFU eviction, cache-aside vs write-through

#### 76. Twitter Feed (Home Timeline)
**Tier:** Must Know | **File:** `content/system-design/twitter-feed.md`
- Core: when user A posts a tweet, A's followers should see it in their home timeline
- Fan-out on write: precompute and push tweet ID to each follower's Redis list; fast read, slow write for celebrities
- Hybrid approach: fan-out on write for regular users, fan-out on read for celebrities (>10K followers)
- Deep dives: timeline ranking (ML-based vs chronological), media handling (images stored in object storage + CDN), tweet deletion
- Key concepts: fanout strategies, Redis sorted sets, Kafka for async fanout, object storage + CDN for media

#### 77. YouTube / Video Streaming
**Tier:** Must Know | **File:** `content/system-design/video-streaming.md`
- Core: upload video → transcode into multiple bitrates/resolutions → store segments → stream via CDN to clients
- Transcoding pipeline: raw video → message queue → transcoder workers → multiple HLS/DASH segment files → object storage
- Adaptive bitrate streaming: client selects quality based on bandwidth; segment-based (2–10s segments)
- Deep dives: video deduplication (fingerprinting), resumable uploads, comments/likes system design
- Key concepts: object storage, CDN, message queue (async transcoding), Cassandra for metadata, Redis for view counts

#### 78. WhatsApp / Chat System
**Tier:** Must Know | **File:** `content/system-design/chat-system.md`
- Core: 1-on-1 messaging with delivery receipts (sent, delivered, read); group chat; media sharing
- Message delivery: WebSocket connection to chat server; message stored in Cassandra before ack; push notification when offline
- Message ordering: per-conversation sequence numbers; Cassandra partition key = conversation ID, clustering key = message timestamp
- Deep dives: group chat fan-out, end-to-end encryption design, message history sync on new device
- Key concepts: WebSocket at scale, Cassandra for message storage, Kafka for async fan-out, object storage for media

#### 79. Uber / Ride-Sharing
**Tier:** Must Know | **File:** `content/system-design/ride-sharing.md`
- Core: rider requests ride → find nearby available drivers → match → track trip → process payment
- Location service: drivers send GPS every 4s → Redis GeoHash index → radius query for nearby drivers
- Matching service: find candidate drivers → rank by ETA → offer trip → driver accepts/rejects
- Deep dives: surge pricing (supply/demand per GeoHash cell), trip tracking with WebSockets, payment processing
- Key concepts: GeoHash, QuadTree, WebSocket at scale, Redis for location index, Kafka for trip events, payment idempotency

#### 80. Web Crawler
**Tier:** Must Know | **File:** `content/system-design/web-crawler.md`
- Core: seed URLs → fetch page → extract links → deduplicate → queue new URLs → repeat at scale
- Deduplication: Bloom filter to quickly check if URL already crawled (false positive = re-crawl; acceptable)
- Politeness: per-domain crawl rate limits; robots.txt compliance; backoff on 429/503
- Deep dives: URL frontier prioritization (important pages first), handling dynamic content (JavaScript rendering), distributed crawler coordination
- Key concepts: Bloom filter, consistent hashing (assign URLs to crawler workers by domain hash), BFS/DFS frontier, message queue

#### 81. Search Engine
**Tier:** Must Know | **File:** `content/system-design/search-engine.md`
- Core: web crawler → indexing pipeline → inverted index → query processing → ranked results
- Indexing pipeline: raw HTML → content extraction → tokenization → index construction → distributed index shards
- Query processing: parse query → look up posting lists → intersect lists → BM25 score → rank and paginate
- Deep dives: index freshness (near-real-time indexing), PageRank-style link analysis, spell correction, query suggestion
- Key concepts: inverted index, Bloom filter (URL deduplication), consistent hashing (shard assignment), Kafka (crawl queue)

#### 82. Rate Limiter Service
**Tier:** Must Know | **File:** `content/system-design/rate-limiter.md`
- Core: centralized rate limiting service used by API gateway; enforce per-user or per-IP limits
- Algorithms: token bucket for burst allowance, sliding window log for precision, sliding window counter for scale
- Distributed implementation: Redis atomic Lua scripts for sliding window; eventual consistency trade-offs
- Deep dives: handling Redis failure (fail open vs fail closed), rate limiting in a distributed API gateway, per-endpoint vs global limits
- Key concepts: token bucket, Redis Lua scripts, CAP trade-off (fail open = available, fail closed = consistent)

#### 83. Notification System
**Tier:** Must Know | **File:** `content/system-design/notification-system.md`
- Core: trigger notification event → route by channel (push/email/SMS) → deliver via third-party providers → track delivery status
- Fanout: event from upstream service → Kafka → notification consumer → per-channel queue → delivery workers
- Channels: APNs (iOS push), FCM (Android push), SMTP (email), Twilio/SNS (SMS), WebSocket (in-app)
- Deep dives: notification deduplication, user preference management (opt-out per type), delivery retry with DLQ
- Key concepts: Kafka fan-out, DLQ for failed delivery, idempotency keys, Redis for deduplication within time window

#### 84. Leaderboard
**Tier:** Must Know | **File:** `content/system-design/leaderboard.md`
- Core: update user score → maintain global ranking → serve top-K and user rank queries at low latency
- Redis sorted sets: ZADD (O(log n)), ZRANK (O(log n)), ZRANGE (O(log n + k)); all operations sub-millisecond
- Scale: millions of users, thousands of score updates/second; Redis ZSET handles this well in-memory
- Deep dives: segmented leaderboards (per-country, per-friend-group), real-time vs near-real-time update strategies, large-scale tie-breaking
- Key concepts: Redis sorted sets, consistent hashing for sharding across multiple Redis instances, batch aggregation for high-write scenarios

#### 85. Payment System
**Tier:** Must Know | **File:** `content/system-design/payment-system.md`
- Core: initiate payment → authorize with payment processor → debit buyer → credit merchant → record in ledger
- Idempotency: each payment has a unique payment intent ID; retries return same result, never double-charge
- Ledger: append-only, immutable; double-entry bookkeeping; every debit paired with a credit
- Deep dives: reconciliation (compare ledger vs processor records nightly), handling partial failures (outbox pattern), fraud detection integration
- Key concepts: idempotency keys, outbox pattern, 2PC vs Saga, double-entry bookkeeping, event sourcing for ledger

#### 86. Distributed Message Queue (Kafka-like)
**Tier:** Must Know | **File:** `content/system-design/distributed-message-queue.md`
- Core: producers write messages → partitioned topics → consumers read via consumer groups → at-least-once delivery
- Partitioning: hash message key to partition; each partition is an ordered, immutable log stored on disk
- Replication: leader partition on one broker, follower replicas on others; ISR (in-sync replica) set for durability
- Deep dives: exactly-once semantics (idempotent producer + transactions), consumer lag monitoring, compacted topics for changelog use case
- Key concepts: consistent hashing for partition assignment, Raft/ZooKeeper for broker metadata, ISR for durability, offset commit protocol

#### 87. Typeahead / Autocomplete
**Tier:** Must Know | **File:** `content/system-design/typeahead.md`
- Core: user types prefix → return top-K matching suggestions ranked by popularity in <100ms
- Trie approach: each node stores top-K completions; single server can handle millions of prefixes in memory
- Data pipeline: batch job aggregates search logs → compute top-K per prefix → serialize trie → deploy to serving nodes
- Deep dives: real-time trending updates (Kafka stream updates top-K), personalization, multi-language support
- Key concepts: trie, Redis sorted sets (alternative for simpler implementation), consistent hashing (shard by prefix range), CDN caching for common prefixes

#### 88. Google Maps / Navigation
**Tier:** Must Know | **File:** `content/system-design/google-maps.md`
- Core: store road network graph → compute shortest path → render map tiles → real-time traffic updates
- Road network: graph DB or adjacency list; edges have distance + speed limit; Dijkstra / A* for routing
- Map tiles: pre-rendered at multiple zoom levels; stored in object storage; served via CDN; tile key = (zoom, x, y)
- Real-time traffic: GPS data from users → Kafka → stream processing → update edge weights → re-route if needed
- Deep dives: ETA estimation (historical + real-time data), offline maps (delta sync), route alternatives
- Key concepts: graph algorithms (Dijkstra, A*), QuadTree for tile indexing, object storage + CDN for tiles, Kafka for traffic stream

#### 89. Dropbox / Google Drive
**Tier:** Must Know | **File:** `content/system-design/file-storage.md`
- Core: upload file → chunk → deduplicate → store chunks → sync across devices → share with other users
- Chunking: split files into 4MB blocks; hash each block (SHA-256); only upload changed blocks on update (delta sync)
- Deduplication: block hash → check if block already exists in storage; only store unique blocks (content-addressed storage)
- Sync engine: client tracks local file changes; server tracks block state per file version; merge changes on reconnect
- Deep dives: conflict resolution (two users edit same file offline), large file resumable upload, access control and sharing permissions
- Key concepts: object storage (S3), consistent hashing for block placement, Bloom filter for block deduplication check, CDC for sync

#### 90. Instagram / Photo Sharing
**Tier:** Must Know | **File:** `content/system-design/instagram.md`
- Core: upload photo → process/resize → store → display in follower feeds; follow/unfollow; like/comment
- Photo pipeline: upload to object storage → async worker resizes to multiple sizes → CDN-served URLs stored in DB
- Feed generation: hybrid fan-out — fan-out on write for regular users, fan-out on read for celebrities
- Deep dives: photo ranking algorithm (ML-based feed vs chronological), explore page (trending content), notification on like/comment
- Key concepts: object storage + CDN for photos, fan-out strategies, Redis sorted sets for feed, Cassandra for likes/comments at scale

---

## Flat Topic Reference — All 90 Topics by Tier

### Tier 1 — Must Know

| # | Topic | Month |
|---|-------|-------|
| 1 | HTTP/1.1 vs HTTP/2 vs HTTP/3 | 1 |
| 2 | Reverse Proxy vs Forward Proxy | 1 |
| 3 | TCP vs UDP | 1 |
| 4 | DNS | 1 |
| 5 | Load Balancing (L4 vs L7) | 1 |
| 6 | CDN | 1 |
| 7 | WebSockets vs Long Polling vs SSE | 1 |
| 8 | REST vs gRPC vs GraphQL | 1 |
| 9 | Database Indexes | 1 |
| 10 | RDBMS Internals (MVCC, WAL, ACID) | 1 |
| 11 | SQL vs NoSQL Decision Framework | 1 |
| 12 | Key-Value Stores (Redis) | 1 |
| 13 | Wide-Column Stores (Cassandra) | 1 |
| 15 | Object Storage (S3-like) | 1 |
| 17 | Consistent Hashing | 1 |
| 18 | Database Sharding | 1 |
| 19 | Bloom Filters | 1 |
| 20 | Caching Patterns | 1 |
| 21 | Cache Eviction | 1 |
| 22 | Hot-Key / Hotspot Problems | 1 |
| 23 | Full-Text Search | 1 |
| 24 | Read Replicas & Replication Lag | 1 |
| 25 | CAP Theorem | 2 |
| 27 | Consistency Models | 2 |
| 28 | Idempotency & Exactly-Once | 2 |
| 29 | Distributed Locking | 2 |
| 32 | Quorum Reads & Writes | 2 |
| 33 | Two-Phase Commit (2PC) | 2 |
| 34 | Saga Pattern | 2 |
| 35 | Outbox Pattern | 2 |
| 38 | Multi-Region Design | 2 |
| 39 | Message Queues vs Event Streams | 3 |
| 40 | Kafka Deep Dive | 3 |
| 41 | Event-Driven Architecture | 3 |
| 42 | CQRS | 3 |
| 44 | Rate Limiting (All Algorithms) | 3 |
| 45 | Circuit Breaker | 3 |
| 48 | API Pagination | 3 |
| 49 | API Idempotency Keys & Versioning | 3 |
| 50 | Capacity Estimation | 3 |
| 53 | Search Autocomplete / Typeahead | 4 |
| 54 | GeoHash | 4 |
| 55 | QuadTree | 4 |
| 57 | Presence & Online Status | 4 |
| 58 | Notification Fanout Strategies | 4 |
| 59 | WebSocket at Scale | 4 |
| 61 | Payment System Design | 4 |
| 62 | Distributed Job Scheduling | 4 |
| 63 | Unique ID Generation | 4 |
| 64 | OAuth 2.0 & OIDC | 4 |
| 65 | JWT | 4 |
| 73–90 | All Case Studies | 5 |

### Tier 2 — Good to Know

| # | Topic | Month |
|---|-------|-------|
| 14 | Document Stores (MongoDB) | 1 |
| 16 | Time-Series Databases | 1 |
| 26 | PACELC | 2 |
| 30 | Leader Election | 2 |
| 31 | Gossip Protocol | 2 |
| 36 | Raft Consensus (Conceptual) | 2 |
| 37 | Logical Clocks | 2 |
| 43 | Dead Letter Queues & Retry | 3 |
| 46 | Bulkhead Pattern | 3 |
| 47 | Back-Pressure & Load Shedding | 3 |
| 51 | Microservices vs Monolith | 3 |
| 52 | Service Discovery & API Gateway | 3 |
| 56 | Uber-Style Location Indexing | 4 |
| 66 | API Authentication Patterns | 4 |
| 67 | Batch vs Streaming | 4 |
| 68 | Data Warehouse Basics | 4 |
| 69 | Change Data Capture (CDC) | 4 |
| 70 | Recommendation System Design | 4 |

### Tier 3 — Optional (Role-Specific or Bonus)

| # | Topic | Month |
|---|-------|-------|
| 60 | MQTT | 4 |
| 71 | Feature Store Design | 4 |
| 72 | ML Platform Design | 4 |

---

## What Was Intentionally Left Out

| Area | Topics Excluded | Reason |
|------|----------------|--------|
| Infrastructure & DevOps | Docker, Kubernetes, Helm, Terraform, Ansible, CI/CD pipelines, Service Mesh (Istio, Envoy), Chaos Engineering | Ops-level — not tested in system design interview rounds |
| Deep Data Engineering | Spark internals, Flink deep dive, Delta Lake, Iceberg, Hudi, data catalogs, feature engineering pipelines | Too specialized; data engineering interviews are a separate track |
| Deep AI/ML | LLM training, RLHF, LoRA, distributed training (DDP/FSDP), speculative decoding, diffusion models | Beyond scope for general system design rounds; covered minimally for relevant roles |
| Cutting Edge | Blockchain, Web3, EVM, Quantum computing, Federated Learning, post-quantum cryptography | Not tested in system design interviews |
| Deep Consensus Theory | Full Paxos algorithm, PBFT, HotStuff, CRDTs (beyond conceptual) | Raft at a conceptual level is sufficient; deeper theory is academic for interviews |
| Advanced Security | Homomorphic encryption, zero-knowledge proofs, MPC, supply chain security | Advanced cryptography is rarely tested in system design rounds |

---

## Prompt Template

When generating a new topic post, use this prompt:

```
Generate a Hugo markdown post for the following topic. Follow the existing content format in this repo:
- YAML frontmatter: title, weight, type: docs
- Open with a real-world scenario that motivates the topic
- Show the naive/broken approach before introducing the solution
- Include a Mermaid diagram for the architecture or data flow
- Include a code example (Python or pseudocode)
- End with a comparison table of tradeoffs or alternatives
- Tone: practical and interview-focused, not academic

---
Topic: [TOPIC NAME]
File: [FILE PATH]

Key concepts to cover:
[PASTE THE BULLET POINTS FROM THE ENTRY ABOVE]
```
