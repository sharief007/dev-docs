# L2: Storage & Databases

> Storage is where data lives. Master the internals to design systems that are fast, durable, and scalable.

---

## 1. Storage Hardware

### HDD Internals
- **Platters**: magnetic disks, multiple per drive
- **Seek time**: moving read/write head to correct track (~3–10 ms)
- **Rotational latency**: waiting for sector to rotate under head (~2–4 ms at 7200 RPM)
- **Transfer time**: actual data read time
- **Total HDD latency**: 5–20 ms (dominated by seek + rotational)
- Best for: sequential access (streaming), cold archival storage

### SSD Internals
- **NAND flash types**: SLC (1 bit/cell, fast), MLC (2), TLC (3), QLC (4) — capacity ↑, endurance ↓
- **Flash pages** (~4–16 KB): smallest read/write unit
- **Erase blocks** (~256 KB–4 MB): must erase before writing
- **Write amplification**: writing more data to flash than actually requested (due to erase blocks)
- **Wear leveling**: distribute writes evenly to extend lifespan
- **FTL** (Flash Translation Layer): handles bad block management, wear leveling
- **P/E cycles**: TLC NAND ~1000–3000 cycles before failure

### NVMe vs SATA vs SAS
| Interface | Bandwidth | Latency |
|-----------|-----------|---------|
| SATA III | 600 MB/s | ~100 μs |
| SAS | 1.2 GB/s | ~100 μs |
| NVMe (PCIe 4.0) | 7+ GB/s | ~20 μs |
| NVMe (PCIe 5.0) | 14+ GB/s | ~20 μs |

### Storage Tiers
- **Hot**: NVMe SSD, in-memory (fastest, expensive)
- **Warm**: SATA SSD (balanced cost/performance)
- **Cold**: HDD, object storage (cheap, slow)
- **Archive**: tape, Glacier (very cheap, hours of retrieval)

### RAID Levels
| Level | Description | Min Disks | Fault Tolerance | Notes |
|-------|------------|-----------|----------------|-------|
| RAID 0 | Striping | 2 | None | Max performance, no redundancy |
| RAID 1 | Mirroring | 2 | 1 disk | Full copy, expensive |
| RAID 5 | Striping + parity | 3 | 1 disk | Good balance |
| RAID 6 | Striping + double parity | 4 | 2 disks | Safer than RAID 5 |
| RAID 10 | Mirroring + striping | 4 | 1 per mirror | High perf + redundancy |

### Object Storage
- No filesystem hierarchy — flat namespace with keys
- **Erasure coding**: stores data across N nodes, recovers from any M failures (more space-efficient than replication)
  - E.g., RS(6,3): 6 data + 3 parity shards — tolerate any 3 failures
- Examples: AWS S3, GCS, Azure Blob
- S3 internals: distributed key-value store, eventual consistency historically (now strong consistency for reads after write)

---

## 2. File Systems

### inode Structure
- Stores metadata: permissions, owner, timestamps, size, block pointers
- Does NOT store filename (filename → inode mapping is in directory entries)
- Direct blocks, single indirect, double indirect, triple indirect pointers
- Hard links: multiple directory entries → same inode
- Soft links: inode pointing to a path string

### Journaling (ext4, XFS)
- Write-ahead log (journal) records intent before metadata updates
- Prevents filesystem corruption on crash
- Journal modes: writeback (metadata only), ordered (data before metadata), journal (safest, slowest)

### Copy-on-Write (ZFS, Btrfs)
- Never overwrite existing blocks — always write new blocks
- Old version intact until transaction committed
- Free snapshots (just keep old block pointers)
- Self-healing (checksums + redundancy detect and repair corruption)

### Distributed File Systems
- **NFS**: client-server, POSIX-like semantics, stateless (NFSv3)
- **HDFS** (Hadoop Distributed File System):
  - NameNode: metadata (namespace tree, block locations)
  - DataNodes: actual blocks (default 128 MB each), 3× replication
  - Rack-aware placement: two replicas on same rack, one on another rack
  - Optimized for large sequential reads (MapReduce workloads)
- **GlusterFS**: peer-to-peer, no metadata server
- **Ceph**: unified object/block/file storage, CRUSH algorithm for placement (no metadata bottleneck)
- **GFS (Google File System)**: original inspiration for HDFS, 2003 paper

---

## 3. Relational Databases (RDBMS)

### ACID Properties

**Atomicity**: All-or-nothing — partial transactions never committed
- Implemented via undo logs (rollback) and WAL

**Consistency**: Transaction brings DB from one valid state to another
- Enforced by constraints, triggers, foreign keys

**Isolation**: Concurrent transactions don't see each other's intermediate state
- Implemented via locking or MVCC

**Durability**: Committed transactions survive crashes
- Implemented via WAL (Write-Ahead Log) + checkpoints

### Isolation Levels and Anomalies

| Isolation Level | Dirty Read | Non-Repeatable Read | Phantom Read |
|-----------------|------------|--------------------|----|
| READ UNCOMMITTED | Possible | Possible | Possible |
| READ COMMITTED | Prevented | Possible | Possible |
| REPEATABLE READ | Prevented | Prevented | Possible |
| SERIALIZABLE | Prevented | Prevented | Prevented |

- **Dirty read**: reading uncommitted data from another transaction
- **Non-repeatable read**: same row returns different value within one transaction
- **Phantom read**: same query returns different rows (due to insert/delete by another transaction)
- **Write skew**: two transactions read overlapping data, write non-overlapping data, violating a constraint (even under REPEATABLE READ)

### B-Tree and B+ Tree Indexes
- B+ Tree: all data in leaf nodes, leaf nodes linked for range scans
- Fanout: typically 100–1000 children per node (page-size dependent)
- Height: 3–4 levels for billions of rows
- Balanced: all leaves at same depth
- **Clustered index**: data physically sorted by index (InnoDB primary key, SQL Server)
- **Secondary indexes**: separate structure pointing to primary key or row location
- **Covering index**: index contains all columns needed by query (no heap/table lookup)

### Other Index Types
- **Hash indexes**: O(1) equality, no range queries (Memory engine in MySQL)
- **GiST** (Generalized Search Tree): PostgreSQL extensible index for geometry, full-text, etc.
- **GIN** (Generalized Inverted Index): for full-text search, JSONB, arrays
- **BRIN** (Block Range Index): tiny index for naturally ordered data (timestamps, serial IDs)
- **Partial indexes**: index subset of rows (e.g., `WHERE status = 'active'`)
- **Expression indexes**: index on `lower(email)` etc.

### Query Execution
- **Query parser** → rewrite rules → **query planner** → **executor**
- EXPLAIN and EXPLAIN ANALYZE
- **Sequential scan**: read all pages — good for large table fraction
- **Index scan**: follow index → heap — good for selective queries
- **Index-only scan**: all data in index (covering index)
- **Bitmap scan**: collect heap locations from index, sort, read heap efficiently
- **Join algorithms**:
  - Nested Loop: O(n×m), good for small inner tables
  - Hash Join: O(n+m), good for large unsorted tables
  - Merge Join: O(n log n + m log m), good for pre-sorted inputs

### MVCC (Multi-Version Concurrency Control)
- Each transaction sees a snapshot of the database as of transaction start
- Old row versions kept for concurrent readers (no read locks!)
- PostgreSQL: stores old versions in heap alongside current data (dead tuples need VACUUM)
- MySQL InnoDB: stores old versions in undo log (UNDO tablespace)
- **VACUUM (PostgreSQL)**: reclaims dead tuple space, updates visibility maps

### Write-Ahead Log (WAL)
- All changes written to WAL before modifying actual data pages
- WAL is sequential → fast (even on HDD)
- On crash recovery: redo from last checkpoint using WAL
- **WAL shipping**: send WAL to replicas for streaming replication
- **Logical replication**: decode WAL into logical changes (INSERT/UPDATE/DELETE)

### Partitioning
- **Horizontal (Sharding)**: split rows across partitions
  - Range: partition by date range (common for time-series)
  - Hash: partition by hash of key (even distribution)
  - List: partition by explicit value list
- **Vertical**: split columns (rarely in same DB, more common in application design)
- **Partition pruning**: query planner skips irrelevant partitions
- Local vs global indexes on partitioned tables

### Replication
- **Streaming replication**: send WAL records to standby in real-time (synchronous or asynchronous)
- **Logical replication**: table-level, supports different schemas on replica
- **Cascading replication**: replica replicates to another replica
- **Replication lag**: async replication may be seconds behind — stale reads possible

---

## 4. NoSQL Databases

### Key-Value Stores

**Redis**
- In-memory, optionally persistent (RDB snapshots, AOF append-only file)
- Data structures: String, Hash, List, Set, Sorted Set (skiplist), HyperLogLog, Stream, Geo
- **Redis Cluster**: hash slots (16384 total) distributed across nodes, primary-replica per shard
- **Redis Sentinel**: HA for non-clustered Redis, monitors, promotes replica on failure
- Lua scripting for atomic multi-key operations
- **Pub/Sub**: message broker built-in (but no persistence, no consumer groups)
- **Redis Streams**: persistent, consumer groups, exactly-once delivery
- **Eviction policies**: noeviction, allkeys-lru, volatile-lru, allkeys-lfu, etc.
- **Use cases**: caching, session storage, rate limiting, distributed locks, leaderboards

**Memcached**
- Purely in-memory cache, no persistence
- Multi-threaded (vs Redis single-threaded main loop)
- Simpler data model: key-string only
- Better for high-throughput simple caching with multiple CPU cores

**DynamoDB**
- AWS fully managed key-value + document store
- Single-digit millisecond performance at any scale
- Provisioned vs on-demand capacity
- Partition key (hash) + optional sort key (range)
- DynamoDB Streams for CDC
- Global Tables for multi-region active-active
- DAX (DynamoDB Accelerator): in-memory cache layer

### Document Stores

**MongoDB**
- BSON (Binary JSON) documents, nested structures
- **WiredTiger** storage engine: MVCC, compression, checkpoint every 60s
- **Aggregation pipeline**: $match, $group, $lookup (join), $unwind, $project
- **Change streams**: watch for data changes (built on oplog)
- **Sharding**: range or hash shard key, mongos router, config servers
- **Replica set**: primary + secondaries + arbiters, automatic failover
- **Atlas**: managed MongoDB with full-text search, vector search

### Wide-Column / Columnar Stores

**Cassandra**
- **Ring topology**: consistent hashing, no single master
- **Vnodes**: virtual nodes per physical node for better distribution
- **Partitioner**: determines which node owns a partition
- **Replication factor**: how many copies (RF=3 typical)
- **Consistency levels**: ONE, QUORUM, LOCAL_QUORUM, ALL
- **Read repair**: reconcile inconsistent replicas on read
- **Hinted handoff**: store writes for temporarily unavailable nodes
- **Compaction**: SSTable merging (size-tiered or leveled)
- **Tunable consistency**: trade-off between consistency and availability per operation
- **Use cases**: time-series, IoT, write-heavy, global distribution

**HBase**
- Column-family based, runs on HDFS
- Master-RegionServer architecture (like HDFS NameNode-DataNode)
- Strong consistency (unlike Cassandra)
- Region splits for scalability
- Used for: random read/write on large datasets (Facebook Messages used HBase)

**Apache Parquet / ORC (file formats)**
- Columnar storage: each column stored together
- **Benefits**: read only needed columns, excellent compression (similar values together)
- **Parquet**: Apache standard, excellent for Spark/Hive/Presto
- **ORC**: Hive-optimized, predicate pushdown, Bloom filters per column
- **Compression codecs**: Snappy (fast), Zstd (better ratio), LZ4

### Graph Databases

**Neo4j**
- Property graph model: nodes + edges + properties
- **Cypher** query language: `MATCH (n:Person)-[:KNOWS]->(m) RETURN m`
- Native graph storage: adjacency lists per node (no join overhead)
- **APOC** library for advanced procedures
- Uses: fraud detection, social graphs, recommendation engines, knowledge graphs

**Graph Query Languages**
- Cypher (Neo4j), Gremlin (TinkerPop standard), SPARQL (RDF/semantic web)
- openCypher: open standard based on Neo4j's Cypher

**When to Use Graph Databases**
- Highly connected data with many relationship traversals
- Depth-first traversal queries (who knows who knows who)
- Not for: simple lookups, analytics on non-connected data

### Time-Series Databases

**InfluxDB**
- Purpose-built for time-series (metrics, events, logs)
- **Measurement + tags (indexed) + fields (values) + timestamp**
- Data sorted by time — excellent compression of monotonic timestamps
- Downsampling with Flux or InfluxQL
- Retention policies: automatic data expiry

**TimescaleDB**
- PostgreSQL extension — full SQL on time-series
- Hypertables: automatic partitioning by time
- Continuous aggregates (materialized views that auto-update)
- Compression: columnar for old data, row-based for recent

**Prometheus**
- Pull-based metrics collection
- **Time-series DB** optimized for sparse data
- PromQL: powerful functional query language
- Alertmanager integration
- Not for long-term storage — use Thanos or Cortex for multi-cluster

### Search Engines

**Elasticsearch / OpenSearch Internals**
- Built on Apache Lucene
- **Inverted index**: maps term → list of document IDs
- **Segments**: immutable Lucene index files, merged periodically
- **Shards**: horizontal partition (primary + replica shards)
- **Indexing**: document → tokenizer → token filters → inverted index
- **Relevance scoring**: BM25 (replaced TF-IDF as default)
- **Query types**: match, bool, range, term, prefix, wildcard, fuzzy, nested, geo
- **Aggregations**: bucketing, metrics, pipeline (analytics without separate system)
- **Mapping**: schema definition, explicit vs dynamic

### Vector Databases

**What They Do**
- Store and retrieve high-dimensional vectors (embeddings)
- Support Approximate Nearest Neighbor (ANN) search
- Core to: semantic search, RAG, recommendation systems

**Key Algorithms**
- **HNSW** (Hierarchical Navigable Small World): graph-based, O(log n) search, high recall
  - Multi-layer graph: upper layers for long-range navigation, bottom layer for fine search
  - Best for: high recall requirements, in-memory indexes
- **IVF** (Inverted File Index): cluster vectors, search only nearest clusters
  - IVF-Flat: exact within cluster, FAISS default
  - IVF-PQ: Product Quantization compression — huge memory savings
- **LSH** (Locality-Sensitive Hashing): hash similar vectors to same bucket
- **PQ** (Product Quantization): compress vectors by quantizing sub-vectors
- **ScaNN** (Google): anisotropic quantization, better recall than PQ

**Options**
- **FAISS** (Facebook AI Similarity Search): library, CPU/GPU, best for research
- **Pinecone**: managed, serverless, easy to use
- **Weaviate**: open-source, built-in ML models, supports hybrid search
- **Qdrant**: Rust-based, fast, payload filtering
- **Chroma**: simple, great for local development
- **pgvector**: PostgreSQL extension — SQL + vectors together

---

## 5. Storage Internals & Algorithms

### LSM Trees (Log-Structured Merge Trees)

Used by: **Cassandra, RocksDB, LevelDB, HBase, InfluxDB, ScyllaDB**

```
Write path:
1. Write to MemTable (in-memory sorted buffer)
2. When full, flush to SSTable (immutable sorted file on disk)
3. Background compaction merges SSTables

Read path:
1. Check MemTable (newest data)
2. Check Bloom filter for each SSTable (skip if key not present)
3. Check SSTable index, read data
```

**Compaction Strategies**:
- **Size-tiered**: compact similar-sized SSTables — good for write-heavy
- **Leveled**: maintain strict level sizes, fewer overlapping SSTables — better read performance
- **FIFO**: for time-series, expire oldest data

**LSM Trade-offs**:
- **Write amplification**: data written multiple times during compaction
- **Read amplification**: may need to check multiple SSTables
- **Space amplification**: stale data exists until compaction

### B-Tree vs LSM Tree Trade-offs

| Property | B-Tree | LSM Tree |
|----------|--------|----------|
| Write performance | O(log n), random I/O | O(1) amortized, sequential I/O |
| Read performance | O(log n), consistent | O(log n) with amplification |
| Write amplification | Low (2–3×) | High (10–30×) |
| Read amplification | Low (1×) | Medium (depends on compaction) |
| Space amplification | Low | Medium (stale data) |
| Best for | Read-heavy, updates in place | Write-heavy, append-mostly |

---

## 6. Caching

### Eviction Policies

- **LRU (Least Recently Used)**: evict least recently accessed — most common
  - Implementation: hash map + doubly linked list
  - Problem: cache pollution by one-time large scans
- **LFU (Least Frequently Used)**: evict least frequently accessed
  - Better for skewed access patterns
  - Implementation: frequency buckets
- **ARC (Adaptive Replacement Cache)**: hybrid LRU + LFU, self-tuning
  - Used in: ZFS, IBM storage
- **2Q (Two-Queue)**: one queue for first access, one for repeated — avoids scan pollution
- **CLOCK / Clock-with-Adaptive-Replacement**: efficient approximation of LRU

### Cache Strategies

- **Cache-aside (Lazy Loading)**: app checks cache → on miss, loads from DB and populates cache
  - Risk: cache miss storm on cold start
- **Read-through**: cache sits in front of DB, populates itself on miss
- **Write-through**: writes go to cache AND DB synchronously
  - Consistent but slower writes
- **Write-behind (Write-back)**: writes go to cache, async flushed to DB
  - Faster writes, risk of data loss on cache failure
- **Write-around**: writes go directly to DB, bypass cache
  - Good for write-once-read-never data

### Cache Invalidation Problems

- **Cache stampede (Thundering Herd)**: many requests hit DB simultaneously after cache expiry
  - Solutions: mutex/lock on cache miss, probabilistic early expiration, jitter on TTLs
- **Stale data**: cache not updated when underlying data changes
  - Solutions: short TTLs, event-driven invalidation, cache tags/purge API
- **Cache coherence in distributed caches**: multiple cache nodes with stale data
  - Use versioned keys or events to invalidate

### Distributed Caching

- **Redis Cluster**: sharded across nodes using hash slots
- **Consistent hashing**: minimize cache misses when nodes are added/removed
- **Replication**: each shard has replicas for HA
- **Local + distributed layers**: L1 (in-process LRU) + L2 (Redis cluster) — reduces Redis RTT
  - Facebook's Memcached uses this pattern (look-aside cache with lease mechanism)

---

## 7. Distributed Database Concepts

### Sharding Strategies

- **Range sharding**: shard by key range (e.g., userID 1–1M on shard 1)
  - Problem: hotspots (recent data, popular ranges)
- **Hash sharding**: shard by hash(key) mod N — even distribution
  - Problem: range queries need all shards, resharding is expensive
- **Directory sharding**: lookup table maps key → shard
  - Flexible but lookup table is single point of failure
- **Consistent hashing**: keys and nodes on a ring, minimal remapping when nodes change
  - Virtual nodes: each physical node owns multiple ring positions (better balance)

### Consistent Hashing Deep Dive

- Hash space: 0 → 2³², nodes placed at positions on ring
- Key assigned to next node clockwise
- Adding/removing node: only adjacent keys remapped
- Virtual nodes (vnodes): each physical node = N virtual positions → better load distribution
- Used in: Amazon DynamoDB, Apache Cassandra, Ceph, load balancers

### Replication Patterns

- **Single-leader (master-slave)**: all writes to leader, replicated to followers
  - Failover: promote follower to leader (manual or automatic)
  - Lag: async replication creates stale reads on followers
- **Multi-leader**: writes accepted by multiple nodes, conflict resolution needed
  - Used for: multi-region active-active, offline clients (CouchDB)
  - Conflicts: last-write-wins (LWW), application-level merge, CRDTs
- **Leaderless (Dynamo-style)**: write to any N nodes, read from R nodes, W+R>N for consistency
  - Quorum: typical W=QUORUM (N/2+1), R=QUORUM → guarantees seeing at least one write
  - Sloppy quorum: accept writes on healthy nodes even if preferred nodes down
  - Read repair: fix stale replicas during reads
  - Anti-entropy: background process syncs replicas using Merkle trees

### Distributed Transactions

**Two-Phase Commit (2PC)**
- Phase 1 (Prepare): coordinator asks all participants to prepare
- Phase 2 (Commit): if all vote YES, coordinator sends COMMIT
- **Problem**: blocking — if coordinator crashes between phases, participants stuck
- Used in: XA transactions, PostgreSQL postgres_fdw

**Three-Phase Commit (3PC)**
- Adds a pre-commit phase to enable non-blocking recovery
- **Problem**: still fails under network partition

**Saga Pattern**
- Break distributed transaction into local transactions with compensating transactions
- **Choreography**: each service publishes events, others react
- **Orchestration**: central saga orchestrator commands each step
- Good for: long-running transactions, microservices
- Example: Book hotel, flight, car. If car booking fails, cancel hotel and flight.

**Google Spanner**
- Globally distributed, externally consistent (linearizable across regions)
- **TrueTime**: atomic clock + GPS to bound clock uncertainty
- Commits wait out clock uncertainty window (ε ~ 7ms) to guarantee commit timestamp ordering
- Paxos groups per shard, 2PC across shards
- Full SQL support with ACID across global shards

**CockroachDB / YugabyteDB**
- Open-source alternatives to Spanner
- Raft (not Paxos) per range
- Hybrid logical clocks (no atomic clock hardware needed)
- SQL compatible (PostgreSQL wire protocol)
