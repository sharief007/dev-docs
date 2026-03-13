---
title: Storage & Databases
weight: 12
type: docs
toc: false
sidebar:
  open: true
prev:
next:
params:
  editURL:
---

{{< cards >}}
    {{< card link="database-indexes" title="Database Indexes" subtitle="B-tree, hash, composite, covering indexes, write overhead, and when not to index" >}}
    {{< card link="lsm-trees" title="LSM Trees" subtitle="Memtable, SSTables, Bloom filters, compaction strategies, and B-tree vs LSM tradeoffs" >}}
    {{< card link="columnar-storage" title="Columnar Storage" subtitle="Row vs columnar layout, compression, vectorized execution, Parquet, and OLAP vs OLTP" >}}
    {{< card link="rdbms-internals" title="RDBMS Internals" subtitle="ACID, MVCC, WAL, buffer pool, and query planner" >}}
    {{< card link="sql-vs-nosql" title="SQL vs NoSQL" subtitle="Decision framework, polyglot persistence, and NewSQL" >}}
    {{< card link="key-value-stores" title="Key-Value Stores (Redis)" subtitle="Data structures, persistence, clustering, eviction, and use cases" >}}
    {{< card link="wide-column-stores" title="Wide-Column Stores (Cassandra)" subtitle="Partition key, consistency levels, compaction, and hot partition avoidance" >}}
    {{< card link="document-stores" title="Document Stores (MongoDB)" subtitle="BSON, aggregation pipeline, replication, and sharding" >}}
    {{< card link="object-storage" title="Object Storage (S3)" subtitle="Flat namespace, multipart upload, consistency model, and durability" >}}
    {{< card link="time-series-databases" title="Time-Series Databases" subtitle="Append-mostly writes, compression, retention, and downsampling" >}}
    {{< card link="consistent-hashing" title="Consistent Hashing" subtitle="Hash ring, virtual nodes, replication, hotspot mitigation, and real-world applications" >}}
    {{< card link="bloom-filters" title="Bloom Filters & HyperLogLog" subtitle="Probabilistic membership testing, false positive tuning, counting variants, and cardinality estimation" >}}
    {{< card link="caching-patterns" title="Caching Patterns" subtitle="Cache-aside, write-through, write-back, write-around, stampede, penetration, and avalanche" >}}
    {{< card link="cache-eviction" title="Cache Eviction" subtitle="LRU, LFU, ARC, W-TinyLFU, CLOCK, FIFO, and choosing the right policy" >}}
    {{< card link="hotspot-problems" title="Hot-Key / Hotspot Problems" subtitle="Root cause, Redis key replication, partition bucketing, Kafka skew, and the celebrity problem" >}}
    {{< card link="full-text-search" title="Full-Text Search (Elasticsearch)" subtitle="Inverted index, BM25, Lucene segments, shard architecture, and relevance tuning" >}}
{{< /cards >}}
