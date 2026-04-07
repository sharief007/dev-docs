---
title: Database Indexes
weight: 1
type: docs
---

An index is a separate data structure maintained by the database that lets it find rows without scanning the entire table. Every index you add speeds up reads and slows down writes — the art is knowing which tradeoff to make.

{{< callout type="info" >}}
This file covers index types, query compatibility, and when to index. For deeper internals see: [B+ Tree Internals](../b-plus-tree) (pages, fan-out, page splits, write amplification) and [Hash Index](../hash-index) (O(1) mechanics, InnoDB Adaptive Hash Index, Bitcask).
{{< /callout >}}

## The Cost of Not Indexing

```sql
-- Table: orders (50M rows)
SELECT * FROM orders WHERE customer_id = 42;
```

Without an index on `customer_id`, the database reads every row (full table scan, O(n)). With an index, it jumps directly to matching rows (O(log n) for B-tree). At 50M rows the difference is seconds vs milliseconds.

## B-Tree Index

The default index type in PostgreSQL, MySQL (InnoDB), Oracle, and SQL Server. A balanced tree where all leaf nodes are at the same depth, connected by pointers for range traversal.

```
                    [50 | 75]
                   /    |    \
           [20|35]   [60|70]   [80|90]
           / | \     / | \     / | \
         20 35 40  60 65 70  80 85 90   ← leaf nodes (actual row pointers)
          ↔  ↔  ↔   ↔  ↔  ↔   ↔  ↔  ↔  ← linked for range scans
```

**What B-tree supports:**

| Query type | Works? |
|------------|--------|
| Equality (`=`) | ✅ |
| Range (`>`, `<`, `BETWEEN`) | ✅ |
| Prefix LIKE (`LIKE 'abc%'`) | ✅ |
| Suffix LIKE (`LIKE '%abc'`) | ❌ (can't use index — no leading anchor) |
| `ORDER BY` on indexed column | ✅ (rows already sorted — avoids sort step) |
| `IS NULL` | ✅ (PostgreSQL stores NULLs in index) |

**Read cost:** O(log n) to find the first matching leaf, then sequential reads for range results.

**Write cost:** Every INSERT, UPDATE (on indexed column), or DELETE must update the B-tree: split/merge nodes, update pointers. More indexes = more write amplification.

## Hash Index

A hash table mapping `hash(key) → row pointer`. Supported natively in PostgreSQL (`CREATE INDEX ... USING HASH`) and MySQL MEMORY engine.

| | B-Tree | Hash |
|---|---|---|
| Lookup | O(log n) | O(1) average |
| Range queries | ✅ | ❌ |
| ORDER BY | ✅ | ❌ |
| Prefix LIKE | ✅ | ❌ |
| Best for | General-purpose | Exact equality joins on high-cardinality keys |

{{< callout type="info" >}}
In practice, hash indexes are rarely used in OLTP systems. B-tree's O(log n) is fast enough for equality lookups, and the flexibility for range queries and sorting makes B-tree the better default. Hash indexes become relevant in in-memory databases or hash-join operations performed internally by the query engine.
{{< /callout >}}

## Composite Index

An index on multiple columns: `CREATE INDEX idx ON orders (customer_id, status, created_at)`.

The database sorts rows first by `customer_id`, then by `status` within each customer, then by `created_at`. This ordering is the core rule that determines which queries can use the index.

**Leftmost prefix rule:** A query can use the index only if it filters on a contiguous prefix of the index columns, starting from the left.

```sql
-- Index: (customer_id, status, created_at)

-- ✅ Uses index — full prefix
WHERE customer_id = 42 AND status = 'pending' AND created_at > '2024-01-01'

-- ✅ Uses index — partial prefix (leading columns)
WHERE customer_id = 42 AND status = 'pending'

-- ✅ Uses index — leading column only
WHERE customer_id = 42

-- ❌ Cannot use index — skips customer_id (leftmost column missing)
WHERE status = 'pending'

-- ❌ Cannot use index — middle column skipped
WHERE customer_id = 42 AND created_at > '2024-01-01'
-- (database uses index to find customer_id = 42, then must scan all statuses)
```

**Column order rule of thumb:** Place the most selective equality columns first (highest cardinality), then range columns last. Range predicates stop the index from being useful for subsequent columns.

```sql
-- Index for: WHERE status = ? AND created_at BETWEEN ? AND ?
-- ❌ Bad: (created_at, status) — range on leading column kills status filtering
-- ✅ Good: (status, created_at) — equality first, range last
```

## Covering Index

A covering index contains every column a query needs — the `SELECT` list, `WHERE` clause, and `JOIN` conditions. The database satisfies the query entirely from the index without touching the table heap.

```sql
-- Table: orders(id, customer_id, status, total, created_at, ...)
-- Query:
SELECT customer_id, status, total
FROM orders
WHERE customer_id = 42 AND status = 'pending';

-- Non-covering index on (customer_id, status):
--   1. Scan index → get row pointers
--   2. Fetch each row from heap (random I/O) ← expensive

-- Covering index on (customer_id, status, total):
--   1. Scan index → all needed columns are here
--   2. No heap access ← "index-only scan" in EXPLAIN
```

In PostgreSQL `EXPLAIN`, look for `Index Only Scan` instead of `Index Scan`. In MySQL, look for `Using index` in the `Extra` column.

**`INCLUDE` columns (PostgreSQL 11+, SQL Server):** Add non-key columns to the leaf level only — they don't affect sort order but satisfy covering without bloating the B-tree interior.

```sql
CREATE INDEX idx_orders_covering
ON orders (customer_id, status)
INCLUDE (total);        -- total is in leaf pages only
```

## Other Index Types

| Type | Use case | Example |
|------|----------|---------|
| **Partial index** | Index only rows matching a condition — smaller, faster | `CREATE INDEX ON orders (customer_id) WHERE status = 'pending'` |
| **Expression index** | Index the result of a function — query must use same expression | `CREATE INDEX ON users (lower(email))` |
| **Full-text index** | Tokenized text search with ranking | `CREATE INDEX ON posts USING gin(to_tsvector('english', body))` |
| **Spatial index** (GiST/R-tree) | Geospatial queries (point-in-polygon, nearest-neighbor) | `CREATE INDEX ON locations USING gist(coordinates)` |
| **Bitmap index** | Low-cardinality columns (Oracle); PostgreSQL uses bitmap scans internally by combining B-tree indexes | `status`, `country_code` |

## Write Overhead

Every index on a table is a separate structure that must be updated on every write:

```
INSERT into orders → update heap + update every index on orders
UPDATE orders SET status = 'shipped' → update heap + update any index containing status
DELETE from orders → update heap + update every index (or mark as dead)
```

A table with 8 indexes takes ~8× the write I/O of an unindexed table. At high ingestion rates (IoT telemetry, log pipelines), excess indexes become a throughput bottleneck.

**Vacuum / dead tuple cleanup (PostgreSQL):** Deleted rows and old row versions from MVCC are not immediately removed — they become dead tuples. VACUUM reclaims them. Heavily updated indexed columns accumulate dead index entries that bloat the index and slow scans until VACUUM runs.

## When NOT to Index

| Situation | Reason |
|-----------|--------|
| **Low-cardinality column standalone** (boolean, `status` with 3 values) | Index returns a large fraction of rows — full scan is cheaper than random heap I/O per pointer |
| **Write-heavy table with low read rate** (event log, metrics ingest) | Write amplification outweighs read benefit |
| **Small table** (< ~1000 rows) | Full scan fits in a single I/O; index lookup adds overhead |
| **Column rarely in WHERE / JOIN** | Index occupies space and slows writes with no read benefit |
| **Monotonically increasing key with range scans across shards** | B-tree hotspot on the rightmost leaf — consider hash sharding or time-bucketing |

## Reading EXPLAIN

```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT customer_id, status, total
FROM orders WHERE customer_id = 42 AND status = 'pending';
```

Key things to spot:

| Output | Meaning |
|--------|---------|
| `Seq Scan` | Full table scan — missing index or optimizer chose not to use one |
| `Index Scan` | Index used, but heap fetch still needed for non-covered columns |
| `Index Only Scan` | Covering index — no heap access |
| `Bitmap Heap Scan` | Row pointers collected from index scan(s) sorted by physical block location; heap pages read sequentially — avoids random I/O per row; also produced when multiple indexes are OR'd or AND'd together |
| `rows=` estimate vs actual | Large gap → stale statistics; run `ANALYZE` |
| `Buffers: hit=X read=Y` | `hit` = from cache, `read` = from disk — high `read` indicates cache pressure |
