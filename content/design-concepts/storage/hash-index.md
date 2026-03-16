---
title: Hash Index
weight: 3
type: docs
toc: true
sidebar:
  open: true
prev:
next:
params:
  editURL:
---

A hash index maps each key through a hash function to a bucket, then stores a pointer to the row in that bucket. Lookups are O(1) average — there is no tree to traverse. The tradeoff is absolute: a hash index answers only equality predicates. It cannot range-scan, sort, or match prefixes. [Database Indexes](../database-indexes) covers when to choose one over a B+ tree; this file covers how it works and where it appears in real systems.

## Mechanics

```
Index on: users.email

hash("alice@example.com") → bucket 3 → [row_ptr: heap page 41, offset 7]
hash("bob@example.com")   → bucket 7 → [row_ptr: heap page 12, offset 2]
hash("carol@example.com") → bucket 3 → [row_ptr: heap page 88, offset 0]
                                  ↑
                      collision — bucket 3 holds a chain
```

**Hash function:** maps an arbitrary-length key to a fixed-size integer (the bucket index). A good hash function distributes keys uniformly across buckets, minimizing collisions.

**Collision resolution:** Two approaches are common in databases:

| Method | How | Used by |
|--------|-----|---------|
| **Separate chaining** | Each bucket holds a linked list of entries; a collision appends to the list | PostgreSQL hash index |
| **Open addressing / linear probing** | On collision, probe the next empty bucket in sequence | In-memory hash tables, InnoDB AHI |

**Lookup:**
1. Hash the query key → bucket number
2. Read the bucket → get one or more (key, row_ptr) pairs
3. Compare keys for equality (needed because different keys can hash to the same bucket)
4. Follow the matching row pointer to the heap page

**Average O(1), worst-case O(n):** With a uniform hash function and load factor < 0.75, collisions are rare and chaining stays short. A pathological key distribution (many keys hashing to the same bucket) degrades to O(n) — this is why hash function quality matters.

## What Hash Index Supports

| Query | Hash index? | Why |
|-------|:-----------:|-----|
| `WHERE email = 'alice@example.com'` | ✅ | Single hash computation → O(1) bucket lookup |
| `WHERE id IN (1, 5, 9)` | ✅ | Three separate hash lookups |
| `WHERE age > 30` | ❌ | Hash output has no ordering — cannot find "greater than" |
| `WHERE age BETWEEN 20 AND 30` | ❌ | Range requires ordering |
| `WHERE name LIKE 'ali%'` | ❌ | Prefix requires knowing hash of all matching keys upfront |
| `ORDER BY email` | ❌ | Hash output is not sorted |
| `WHERE email IS NULL` | ❌ (PostgreSQL) | NULL is not hashed into the index |

The inability to support range queries or ordering is not a limitation that can be engineered around — it is fundamental to how hashing works. Hashing intentionally destroys key ordering to achieve uniform distribution. If you need ordering, you need a B+ tree.

## B+ Tree vs Hash Index

| | B+ Tree | Hash Index |
|---|---|---|
| **Equality lookup** | O(log n) | O(1) average |
| **Range query** | ✅ | ❌ |
| **ORDER BY** | ✅ | ❌ |
| **Prefix LIKE** | ✅ | ❌ |
| **Composite index** | ✅ (leftmost prefix rule) | ❌ (hashes the whole key) |
| **Index size** | Larger (internal nodes + leaf nodes) | Smaller (flat bucket array + chains) |
| **Write cost** | Moderate (B+ tree node updates, possible splits) | Low (hash + append to chain) |
| **On-disk support** | ✅ (default everywhere) | ✅ PostgreSQL; limited elsewhere |
| **In-memory performance** | Fast | Faster |

**The practical conclusion:** B+ tree's O(log n) vs hash's O(1) is not a meaningful difference in practice — a 3-level B+ tree with a 400-key fan-out reaches any of 64 million rows in 3 I/Os. The real reason to choose a hash index is workload-specific: **pure equality joins on a high-cardinality key where range queries, sorting, and prefix filters are never needed**.

## PostgreSQL Hash Index

PostgreSQL has had hash indexes since version 7.x, but they were not WAL-logged until **PostgreSQL 10 (2017)**. Before v10, a hash index had to be manually rebuilt after a crash — making them unsafe for production. Since v10, they are fully durable and usable.

```sql
-- Create a hash index
CREATE INDEX idx_users_email_hash
ON users USING HASH (email);

-- The query optimizer will use it for equality predicates:
SELECT * FROM users WHERE email = 'alice@example.com';
-- → Index Scan using idx_users_email_hash (cost is lower than B-tree for pure equality)

-- The optimizer will NOT use it for:
SELECT * FROM users WHERE email > 'alice@example.com';  -- range → B-tree fallback
SELECT * FROM users ORDER BY email;                     -- sort → seq scan or B-tree
```

**PostgreSQL hash index internals — overflow pages:**

PostgreSQL implements the hash index as a file of fixed-size 8 KB pages, divided into primary bucket pages and overflow pages. When a bucket fills up, an overflow page is chained to it. Unlike a B-tree, there are no internal routing nodes — the bucket number is computed directly from the hash value and the current number of buckets.

**Splits:** When the load factor exceeds a threshold, the index doubles its bucket count (a split). During a split, half the entries in the overfull bucket are redistributed to a new bucket. This is online and page-level — it does not lock the entire index.

{{< callout type="info" >}}
PostgreSQL's query planner rarely chooses a hash index over a B-tree unless the table is large, the column has very high cardinality, and the workload is strictly equality-only. In practice, B-tree handles equality fast enough that the hash index advantage is marginal except at very high QPS on in-memory datasets.
{{< /callout >}}

## InnoDB Adaptive Hash Index (AHI)

MySQL InnoDB does not let you create hash indexes on user tables. Instead, it builds one **automatically and internally** — the **Adaptive Hash Index (AHI)**.

```
InnoDB Buffer Pool
┌────────────────────────────────────────────────────────┐
│  B+ Tree leaf pages (data)                             │
│  B+ Tree internal pages (routing)                      │
│                                                        │
│  Adaptive Hash Index                                   │
│  ┌──────────────────────────────────────────────┐      │
│  │ hash(index_key) → pointer to B+ tree leaf    │      │
│  │                   page already in buffer pool│      │
│  └──────────────────────────────────────────────┘      │
│         (built and destroyed automatically)            │
└────────────────────────────────────────────────────────┘
```

InnoDB observes which B+ tree leaf pages are accessed repeatedly. For frequently accessed key patterns, it builds an in-memory hash table mapping that key directly to the leaf page that is already in the buffer pool. On the next identical lookup, the engine skips B+ tree traversal entirely and jumps straight to the leaf page.

**Key characteristics:**
- Enabled by default (`innodb_adaptive_hash_index = ON`)
- Entirely in-memory — not persisted; rebuilt from buffer pool contents on restart
- Partitioned into `innodb_adaptive_hash_index_parts` shards (default 8) to reduce lock contention
- Automatically disabled for key patterns that do not benefit
- Can be disabled completely: `SET GLOBAL innodb_adaptive_hash_index = OFF`

**When AHI hurts:** On workloads with high key variety (sequential scans, many distinct keys), the AHI hash table sees constant churn — entries are added and evicted before they are reused. The overhead of maintaining the AHI exceeds its benefit. A symptom is high contention on `rw_lock_x_lock: btr_search_latch` in `SHOW ENGINE INNODB STATUS`.

{{< callout type="warning" >}}
The AHI is a buffer-pool-level optimization, not a storage-level index. It accelerates access to pages that are already in memory. It provides no benefit for cold data that must be read from disk — a B+ tree traversal is always required on a cold read.
{{< /callout >}}

## Bitcask: Hash Indexes in a Storage Engine

Bitcask (the default storage engine in Riak) demonstrates what a hash index looks like as the entire storage engine architecture — not just an auxiliary index on a table.

**Design:**
- All writes are **append-only** to a sequential log file (active file). Random I/O is eliminated entirely.
- An **in-memory hash table** maps every key → `{file_id, byte_offset, value_size}`.
- Reads: hash the key → get file+offset → one disk seek → read value.

```
Write("k1", "v1"):          Active log file         In-memory keydir
  Append to log     →   [k1|v1][k2|v2][k1|v3]   {k1 → (file=1, offset=8)}
  Update keydir                                    {k2 → (file=1, offset=2)}

Read("k1"):
  keydir["k1"] → file=1, offset=8
  pread(file=1, offset=8, len=...)  → "v3"
```

**Compaction:** Log files grow without bound. A background process rewrites each segment, keeping only the latest value per key and discarding all older versions. After compaction, the old files are deleted and the keydir is updated.

**What Bitcask gives you:**
- Write throughput limited only by sequential disk I/O (near disk saturation)
- O(1) reads with exactly one disk seek (assuming keydir fits in RAM)
- Simple crash recovery: re-scan the log, or use the stored keydir snapshot (`hint files`)

**Bitcask's hard constraint:** The entire keydir must fit in RAM. If you have more distinct keys than fit in memory, Bitcask is not usable regardless of how much disk you have. This is not a tunable tradeoff — it is the architectural price of O(1) in-memory hash lookups.

| | Bitcask | LSM Tree | B+ Tree |
|---|---|---|---|
| **Write pattern** | Sequential append | Sequential (memtable → SSTable) | Random in-place page write |
| **Read cost** | 1 disk seek (with hint file) | Multiple SSTable checks | O(log n) tree traversal |
| **Range queries** | ❌ | ✅ (within SSTable) | ✅ (linked leaf list) |
| **RAM requirement** | All keys must fit | Configurable (memtable + Bloom filters) | OS page cache (partial) |
| **Used by** | Riak, simple KV engines | Cassandra, RocksDB, LevelDB | PostgreSQL, MySQL, Oracle |

## When to Use a Hash Index

| Scenario | Use hash index? |
|----------|:--------------:|
| Pure equality lookups on a UUID / token / email column, no range needed | ✅ |
| Very high QPS in-memory workload where O(1) vs O(log n) matters | ✅ |
| Any column that is also ORDER BY, BETWEEN, or LIKE queried | ❌ |
| Composite lookup (`WHERE a = ? AND b = ?`) — need leftmost prefix flexibility | ❌ (use B+ tree composite) |
| Default general-purpose index | ❌ (use B+ tree) |

The default for any new index should be a B+ tree. Reach for a hash index only when profiling confirms that equality-only lookups on a specific high-cardinality column are a measured bottleneck, and range/sort queries on that column will never exist.
