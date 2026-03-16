---
title: B+ Tree
weight: 2
type: docs
toc: true
sidebar:
  open: true
prev:
next:
params:
  editURL:
---

A B+ tree is the index structure underneath PostgreSQL, MySQL (InnoDB), Oracle, and SQL Server. [Database Indexes](../database-indexes) covers what queries a B-tree supports and when to use one. This file goes one level deeper: how the structure actually works on disk, why it is fast, and what happens when you write.

## B-Tree vs B+ Tree

The name "B-tree" is used by every vendor, but the implementation is always a **B+ tree**:

| | B-tree | B+ tree |
|---|---|---|
| Data stored at | Every node (internal + leaf) | Leaf nodes only |
| Internal nodes | Keys + data + child pointers | Keys + child pointers (routing only) |
| Leaf nodes | — | Keys + data, linked in a doubly-linked list |
| Range scan | Must traverse tree | Follow leaf linked list — sequential reads |

The linked leaf list is the critical difference. It is what makes range scans and `ORDER BY` on indexed columns efficient.

## Pages: The Unit of I/O

Every node in a B+ tree is exactly one **page** — a fixed-size block that the database engine reads and writes atomically. The engine never reads a single key; it reads an entire page.

| Database | Page size |
|----------|-----------|
| PostgreSQL | 8 KB (configurable at `initdb` time) |
| InnoDB (MySQL) | 16 KB |
| SQL Server | 8 KB |
| Oracle | 8 KB (default; 2–32 KB configurable) |

This has two major consequences:

**1. Random I/O is expensive because it is always a full page.** Updating a single key in an index requires reading the 8 KB page containing it, modifying it in memory, and writing the full 8 KB back. At high write throughput, scattered random page writes become the bottleneck — this is the problem LSM trees solve.

**2. Large pages = high fan-out = short trees.** The more keys that fit in one page, the fewer levels the tree needs to cover the same number of rows.

## Tree Height and Fan-Out

**Fan-out** is the number of child pointers in one internal node — equivalently, the number of keys per page.

For a PostgreSQL B+ tree on a 4-byte integer column with 6-byte child pointers:

$$\text{fan-out} = \left\lfloor \frac{8192 \text{ bytes}}{4 + 6 \text{ bytes}} \right\rfloor \approx 800 \text{ keys per internal node}$$

Real-world fan-out is lower after accounting for page headers, alignment, and the fill factor (typically 90%) — a practical figure is **200–500 keys per internal node**.

| Tree height | Rows covered (fan-out = 400) |
|-------------|------------------------------|
| 1 | 400 |
| 2 | 160,000 |
| 3 | 64,000,000 |
| 4 | 25,600,000,000 |

A 50-million-row table is reachable in **3 I/Os**. This is why O(log n) is fast in practice — the log is base 400, not base 2.

{{< callout type="info" >}}
Most production B+ tree indexes for OLTP tables are 3–4 levels deep regardless of row count. The root and upper internal nodes stay hot in the buffer pool (database page cache), so only the leaf-level I/O is typically paid on a point lookup.
{{< /callout >}}

## Node Structure

```
Internal node (routing only — never contains actual row data)
┌──────────────────────────────────────────────────────┐
│ Page header │ key₁ │ ptr₁ │ key₂ │ ptr₂ │ … │ keyₙ │ ptrₙ │
└──────────────────────────────────────────────────────┘
  ptr₀ → subtree with keys < key₁
  ptr₁ → subtree with key₁ ≤ keys < key₂
  …

Leaf node (contains actual data or row pointers)
┌────────────────────────────────────────────────────────────┐
│ Page header │ prev ptr │ key₁│val₁ │ key₂│val₂ │ … │ next ptr │
└────────────────────────────────────────────────────────────┘
                ↑ linked list pointer to previous leaf page
                                              ↑ linked list pointer to next leaf page
```

`val` in a leaf node is either the row's full column values (clustered index) or a pointer to the row in the heap (secondary index).

## Read Path

```
Point lookup: WHERE id = 42

Root page (in buffer pool — no I/O)
  └── Internal page (level 2 — likely in buffer pool)
        └── Leaf page (I/O if not cached) → row pointer or data
              └── Heap page (I/O for secondary index — random access)
```

**Clustered index:** The leaf node *is* the row. No heap fetch. InnoDB always clusters the primary key; the table data is physically sorted by primary key on disk.

**Secondary (non-clustered) index:** The leaf node holds the primary key value. For InnoDB, finding the full row requires a second B+ tree traversal on the primary key index ("double dip"). For PostgreSQL, it holds a heap tuple ID (page number + offset) for a direct heap fetch.

**Index-only scan:** If all queried columns are in the index (covering index), the heap fetch is skipped entirely. This is the `Index Only Scan` in PostgreSQL's `EXPLAIN` output and `Using index` in MySQL's `EXPLAIN Extra` column.

**Range scan:** The database navigates to the first matching leaf page, then follows the **linked leaf list** sequentially — no tree traversal needed for subsequent pages. Sequential page reads are orders of magnitude faster than random reads.

## Write Path

Writing to a B+ tree is more expensive than reading because it may require restructuring pages.

{{% steps %}}

### Find the target leaf page

Navigate the tree from root to the correct leaf page — same as a read.

### Write to the page (if space available)

Modify the in-memory copy of the leaf page. The page is now **dirty** and will be flushed to disk by the background writer or at checkpoint time.

Also append the change to the **WAL (Write-Ahead Log)** before marking the page dirty — this is what makes the write durable and crash-recoverable.

### Page split (if the page is full)

When a leaf page has no room for the new key:

```
Before split (leaf full):
[10 | 20 | 30 | 40 | 50]

After split into two leaf pages:
[10 | 20 | 30] ↔ [40 | 50]
                    ↑
       Parent internal node gains a new key (40) and pointer
```

The split propagates upward: the parent internal node gains a new key and pointer. If the parent is also full, it splits too — ultimately potentially splitting all the way up to the root. A root split creates a new root, increasing tree height by 1.

{{% /steps %}}

### Fill Factor

Databases leave internal pages partly empty by default (PostgreSQL default: **90% fill factor for leaf pages, 70% for internal**). The spare capacity absorbs insertions and delays splits. For append-only or monotonically increasing keys (e.g., auto-increment IDs), a fill factor of 100% is safe — splits only ever happen at the rightmost page.

{{< callout type="warning" >}}
Monotonically increasing insert patterns (auto-increment, ULIDs, time-sorted IDs) concentrate all writes on the **rightmost leaf page**. At high insert rates this creates a "right-edge hotspot" where that single page is the write bottleneck. This is why some workloads use random UUIDs as primary keys — distributing writes across all pages — at the cost of more page splits and cache fragmentation.
{{< /callout >}}

## Write Amplification

A single logical write can generate several physical disk writes:

| Write | What it touches |
|-------|----------------|
| WAL entry | Sequential append — always, for every write |
| Dirty leaf page | The modified page — 1 page minimum |
| Dirty internal page(s) | On page split, parent page(s) also dirtied |
| New sibling page | Created by the split |
| MVCC page version (PostgreSQL) | PostgreSQL writes a new tuple version to the heap rather than updating in place — old version stays for concurrent readers |

A table with 8 indexes amplifies this across 8 B+ trees — every INSERT must update every index. This is the fundamental write overhead of B+ trees and the core reason write-heavy workloads use [LSM trees](../lsm-trees) instead.

## B+ Tree vs LSM Tree

| | B+ Tree | LSM Tree |
|---|---|---|
| **Write I/O pattern** | Random (in-place page update) | Sequential (append-only memtable → SSTable flush) |
| **Read amplification** | Low — single tree, O(log n) with huge fan-out | Higher — multiple SSTables checked per lookup |
| **Write amplification** | High — page splits, multiple dirty pages, MVCC versions | Low — batched sequential flushes |
| **Range scan** | ✅ Linked leaf list = sequential page reads | ✅ Within an SSTable; cross-SSTable merge adds cost |
| **Space amplification** | Low | Moderate (compaction temp space, tombstones) |
| **Crash recovery** | WAL replay | WAL (memtable replay) |
| **Used by** | PostgreSQL, MySQL (InnoDB), Oracle, SQL Server | Cassandra, RocksDB, LevelDB, HBase, InfluxDB |

The rule of thumb: **B+ tree for read-heavy OLTP, LSM tree for write-heavy time-series or log workloads.** Most OLTP databases are read-dominated (10:1 to 100:1 read/write ratio), which is why B+ trees dominate relational databases.
