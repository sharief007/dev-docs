---
title: SQL vs NoSQL Decision Framework
weight: 5
type: docs
sidebar:
  open: true
---

The single biggest mistake in system design interviews — and in real systems — is choosing a database before understanding the access patterns. The database is the consequence of the access pattern, not the starting point.

> **The framework:** Define your access patterns → derive your consistency and scale requirements → select the database that satisfies them with the least operational complexity.

## Step 1: Define Your Access Patterns

Before naming any database, answer these questions about your data:

| Question | Why it matters |
|----------|---------------|
| **What is the primary read pattern?** (point lookup by key, range scan, full-text, geo-proximity, aggregation, graph traversal) | Different read patterns require fundamentally different index structures |
| **What is the write pattern?** (individual transactional writes, high-volume ingest, batch loads) | Determines whether write amplification and transaction coordination cost is acceptable |
| **How are entities related?** (none, parent-child hierarchy, many-to-many graph) | Many-to-many relationships across entities push toward SQL; graph-heavy relationships push toward graph DBs |
| **Is the schema fixed or evolving?** (stable columns, sparse/optional fields, user-defined attributes) | Schema rigidity is a cost in SQL; schema-lessness is a cost in query flexibility |
| **What is the consistency requirement?** (strong consistency for every read, eventual consistency acceptable, tunable per operation) | Strong consistency across distributed nodes is expensive — only pay for it where required |
| **What is the scale target?** (thousands of QPS, millions, hundreds of millions) | Many systems never outgrow a single well-tuned PostgreSQL instance |
| **What are the query patterns?** (known key-based queries, ad-hoc analytical, complex joins, multi-field filters) | Ad-hoc queries need SQL; known fixed queries can use NoSQL |

## SQL (Relational Databases)

SQL databases store data in tables with a fixed schema, enforce referential integrity via foreign keys, and support ACID transactions across multiple rows and tables in a single operation.

**Core strengths:**

| Strength | Why it matters |
|----------|---------------|
| **Multi-entity ACID transactions** | Transfer $100 from account A to B atomically — impossible without coordinated transactions |
| **Complex joins across entities** | Compute a report joining orders, customers, products, and shipping — one query |
| **Ad-hoc queries** | No need to predict every query at schema design time; any column can be filtered, sorted, aggregated |
| **Referential integrity** | Foreign key constraints prevent orphan records — data consistency enforced at the database level |
| **Rich indexing** | B-tree, partial, covering, expression, full-text — supports a wide range of query shapes |
| **Mature tooling** | ORMs, migration tools, query analyzers, replication, PITR — decades of ecosystem maturity |

**When SQL is the right choice:**
- Any system where correctness of writes matters more than write throughput (financial ledgers, order management, user accounts, inventory)
- Systems with complex relationships between entities
- Systems where query requirements are unpredictable or evolving (internal tooling, analytics on transactional data)
- Moderate scale — PostgreSQL handles 10k–100k QPS with read replicas; MySQL at Meta serves billions of rows with custom sharding

**Scale ceiling and how to push it:**

```
Single primary (reads + writes)          → vertical scaling, connection pooling (PgBouncer)
Primary + read replicas                  → scale reads horizontally, writes still bottleneck at primary
Application-level sharding               → partition by user_id / tenant; each shard is an independent DB
Vitess (YouTube/PlanetScale)             → transparent sharding layer over MySQL
Citus (PostgreSQL extension)             → distributed PostgreSQL with coordinator + shards
```

**Best engines:** PostgreSQL (correctness, extensions, community), MySQL (operational maturity, Meta/GitHub scale), SQL Server (Windows/.NET ecosystems), Oracle (legacy enterprise, complex workloads).

## NoSQL: Four Categories

"NoSQL" is not a single thing — it is four fundamentally different data models, each optimized for a different access pattern. Picking "NoSQL" without specifying which category is meaningless.

### Key-Value Stores

Every value is opaque to the database — it stores and retrieves by key only. The application owns the structure of the value.

| | |
|---|---|
| **Access pattern** | `GET key`, `SET key value`, `DEL key` — always by exact key |
| **Consistency** | Single-key operations are atomic; no multi-key transactions |
| **Scale** | Trivially sharded by key hash; linear write and read scale |
| **Latency** | Sub-millisecond (Redis in-memory) |
| **Weakness** | No secondary indexes; no range queries; no relationships; value is opaque |
| **Examples** | Redis, DynamoDB (also supports document model), Memcached, etcd |

**Use for:** session storage, caching, distributed locks, rate-limit counters, feature flags, leaderboards (Redis sorted sets), pub/sub.

**Do not use for:** data that needs to be queried by anything other than the primary key.

### Document Stores

Each record is a self-contained document (JSON/BSON). Related data is embedded inside the document rather than normalized into separate tables. The database understands the document structure and can index into nested fields.

| | |
|---|---|
| **Access pattern** | Fetch a document by ID, query by any indexed field, including nested fields |
| **Consistency** | Single-document operations are ACID (MongoDB 4.0+); multi-document transactions added but expensive |
| **Scale** | Sharded by shard key; horizontal scale for reads and writes |
| **Strength** | Schema flexibility — fields can vary per document; natural fit for user profiles, product catalogs with variable attributes |
| **Weakness** | Joins between collections are expensive (done in application or via `$lookup`); schema-less means no enforcement |
| **Examples** | MongoDB, Firestore, CouchDB, DynamoDB (document mode) |

**Use for:** product catalogs (varying attributes per product type), user profiles, content management, mobile app backends, IoT device state.

**Do not use for:** data with many-to-many relationships that must be queried in both directions; financial transactions; data that requires ad-hoc multi-entity reporting.

**The embedding trap:** embedding all related data in one document (e.g., all comments inside a post document) works until the document grows unbounded. A post with 100,000 comments becomes a 10 MB document that must be fully loaded even when you need only the post title.

### Wide-Column Stores

Data is stored in rows identified by a partition key. Each row can have thousands of columns, and different rows can have different columns. Optimized for high write throughput and time-series access patterns.

| | |
|---|---|
| **Access pattern** | Fetch by partition key (exact), with optional clustering column range scan |
| **Consistency** | Tunable per operation (ONE, QUORUM, ALL in Cassandra) — eventual by default |
| **Scale** | Partitioned across nodes by partition key hash; near-linear horizontal write scale |
| **Strength** | Extremely high write throughput (LSM tree storage), predictable latency, no SPOF |
| **Weakness** | Query pattern must be known at schema design time; no ad-hoc queries; no joins |
| **Examples** | Cassandra, HBase, Bigtable (Google), ScyllaDB |

**Use for:** time-series data (IoT sensor readings, metrics, events), activity feeds, messaging (WhatsApp uses Cassandra for message storage), write-heavy logging.

**Schema design is query-driven:** In Cassandra, you design a table per query. Want to fetch messages by user AND by conversation? You need two tables — one partitioned by `user_id`, one by `conversation_id`. Denormalization is intentional and required.

**Do not use for:** transactional workloads; ad-hoc queries; small datasets (operational overhead not worth it below ~TB scale or millions of writes/sec).

### Graph Databases

Stores entities (nodes) and their relationships (edges) as first-class citizens. Optimized for traversing relationships — "find all friends-of-friends within 3 hops who live in NYC" is a single query.

| | |
|---|---|
| **Access pattern** | Traverse relationships from a starting node, path finding, pattern matching |
| **Consistency** | Typically ACID (Neo4j) |
| **Scale** | Harder to shard — relationship traversal crosses partition boundaries |
| **Strength** | Multi-hop relationship queries that would require recursive CTEs or application-side loops in SQL |
| **Weakness** | Poor fit for non-graph queries; limited horizontal scale; smaller ecosystem |
| **Examples** | Neo4j, Amazon Neptune, TigerGraph, Dgraph |

**Use for:** social graphs (LinkedIn connections, Twitter follows), fraud detection (transaction graph anomalies), recommendation engines (collaborative filtering via graph traversal), knowledge graphs.

**Reality check:** Most systems that think they need a graph database actually have moderate relationship depth and are better served by PostgreSQL with recursive CTEs or adjacency list pattern. Reach for a graph DB when relationship traversal is the primary access pattern, not a secondary one.

### Search Engines

Inverted indexes built for full-text search, relevance scoring, and faceted filtering. Not a general-purpose database — they are a read-optimized query layer, typically fed from another database.

| | |
|---|---|
| **Access pattern** | Full-text search with relevance ranking, filtered aggregations, geo-proximity |
| **Consistency** | Near-real-time (document visible ~1s after write); not strongly consistent |
| **Strength** | Sub-second full-text search across billions of documents; rich aggregations and faceting |
| **Weakness** | Not a primary store — no ACID, limited write throughput, data must be re-indexed from source |
| **Examples** | Elasticsearch, OpenSearch, Solr, Typesense, Meilisearch |

**Use for:** product search with filters and autocomplete, log analytics (ELK stack), document search, any query with `LIKE '%term%'` at scale (never efficient in SQL).

## NewSQL: ACID at Horizontal Scale

NewSQL databases offer the relational model and ACID guarantees of traditional SQL databases with the horizontal scalability of NoSQL systems. They achieve this through distributed consensus (Raft/Paxos) and distributed transactions (2PC or timestamp-based ordering).

| Database | Underlying tech | Compatibility | Notes |
|----------|----------------|--------------|-------|
| **Google Spanner** | TrueTime (GPS/atomic clock), Paxos | Proprietary SQL | Externally consistent global transactions; used for Google Ads, Gmail |
| **CockroachDB** | Raft consensus, MVCC | PostgreSQL wire protocol | Serializable transactions across nodes; geo-partitioning |
| **TiDB** | Raft (TiKV storage), HTAP | MySQL wire protocol | Supports OLTP + OLAP on same cluster (TiFlash columnar engine) |
| **Yugabyte** | Raft, DocDB storage | PostgreSQL + Cassandra | Multi-region active-active |
| **PlanetScale** | Vitess + MySQL | MySQL wire protocol | Horizontal sharding with schema migrations without locks |

**When NewSQL is the right choice:**
- You need multi-entity ACID transactions but have outgrown a single PostgreSQL primary
- You need multi-region active-active writes with strong consistency (Spanner, CockroachDB)
- You are building a global system where data must be co-located with users for latency but consistent globally

**The tradeoff:** Distributed transactions require coordination across nodes — this adds latency (network round trips for consensus). A single PostgreSQL commit takes microseconds; a CockroachDB commit across 3 nodes takes 5–20ms. This is acceptable for transactional writes but not for latency-sensitive paths.

## The Most Common Anti-Pattern

> **"We chose MongoDB/Cassandra/DynamoDB because it scales."**

This reasoning conflates two separate problems:
1. **Does our current system have a scale problem?** (usually: no, not yet)
2. **Will a NoSQL database scale better when we do?** (maybe, but at a cost)

What gets sacrificed when you choose NoSQL prematurely:
- **Joins become application-level logic** — you now own what the database did for free
- **Ad-hoc queries become impossible** — every new query shape may require a schema change or a new table
- **Transactions become distributed** — coordinating writes across two NoSQL "documents" or "items" requires saga patterns, outbox patterns, or 2PC in application code
- **Schema enforcement disappears** — bad data enters silently; data quality bugs are discovered at query time
- **Operational complexity increases** — Cassandra cluster management, rebalancing, compaction tuning is non-trivial

PostgreSQL with proper indexing, connection pooling, and read replicas handles most systems that claim to need NoSQL scale. Instagram ran on PostgreSQL at 1 billion users. GitHub's primary data store is MySQL. Shopify runs on MySQL with Vitess.

**Choose NoSQL when you have a specific, identified access pattern that SQL cannot serve efficiently** — not because you anticipate scale.

## Polyglot Persistence

The answer to "which database?" in a real system is almost always "multiple databases." Each database type is used for the workload it excels at.

**Example: E-commerce platform**

```
┌─────────────────────────────────────────────────────────────────┐
│  Access pattern          │  Database           │  Why           │
├─────────────────────────────────────────────────────────────────┤
│  Orders, payments,       │  PostgreSQL         │  ACID,         │
│  user accounts           │  (primary store)    │  joins, schema │
├─────────────────────────────────────────────────────────────────┤
│  Product catalog         │  MongoDB            │  Variable      │
│  (variable attributes)   │                     │  schema        │
├─────────────────────────────────────────────────────────────────┤
│  Product search,         │  Elasticsearch      │  Full-text,    │
│  autocomplete            │                     │  faceted filters│
├─────────────────────────────────────────────────────────────────┤
│  Session, cart,          │  Redis              │  Sub-ms,       │
│  rate limits, cache      │                     │  TTL, counters │
├─────────────────────────────────────────────────────────────────┤
│  Clickstream events,     │  ClickHouse         │  Columnar,     │
│  analytics               │                     │  aggregations  │
├─────────────────────────────────────────────────────────────────┤
│  Product images, assets  │  S3 / Object store  │  Blob storage  │
└─────────────────────────────────────────────────────────────────┘
```

**Data synchronization:** The source of truth is PostgreSQL. Other stores are derived views — kept in sync via:
- **Change Data Capture (CDC):** Debezium reads PostgreSQL WAL → publishes events to Kafka → consumers update Elasticsearch, ClickHouse
- **Dual writes (avoid):** Writing to two systems in one transaction without distributed coordination risks inconsistency on partial failure
- **Event sourcing:** All state changes are events; any database is materialized from the event log

## Decision Framework

Work through these layers in order. Stop at the first layer where a clear answer emerges.

{{% steps %}}

### Is this a caching, session, or ephemeral data problem?

Key-Value store (Redis, Memcached). No further evaluation needed.

### Does the data require full-text search as the primary access pattern?

Search engine (Elasticsearch, Typesense). Feed from a primary store via CDC.

### Are relationships between entities the primary query pattern (multi-hop graph traversal)?

Graph database (Neo4j, Neptune). Otherwise, continue.

### Is write throughput the primary constraint — millions of writes per second, append-only, time-ordered?

Wide-column store (Cassandra, Bigtable). Schema must be designed per query. Continue if you need transactions.

### Is the schema highly variable or hierarchical with optional nested fields, and are queries always by document ID or indexed field?

Document store (MongoDB, Firestore). Continue if you need cross-document transactions.

### Do you need ACID transactions, complex joins, or ad-hoc queries?

SQL (PostgreSQL, MySQL). Scale with read replicas → connection pooling → sharding (Vitess/Citus) → NewSQL if you need global distribution.

### Do you need multi-entity ACID + horizontal write scale across regions?

NewSQL (CockroachDB, Spanner, TiDB).

{{% /steps %}}

## Futuristic and Emerging Patterns

The decision framework above covers systems built today. These categories are becoming relevant for the next generation of systems:

| Category | What it is | When to reach for it |
|----------|-----------|---------------------|
| **Vector databases** (Pinecone, Weaviate, Qdrant, pgvector) | Index high-dimensional embedding vectors; nearest-neighbor search in embedding space | AI/ML similarity search — semantic search, recommendation via embeddings, RAG (Retrieval-Augmented Generation) pipelines |
| **Multi-model databases** (SurrealDB, ArangoDB, Fauna) | One database serving document, graph, key-value, and relational models | Systems that span multiple access patterns without wanting to run multiple databases |
| **Edge / embedded databases** (Cloudflare D1, Turso, SQLite) | SQLite-compatible, deployed at the edge (CDN PoP) close to the user | Ultra-low-latency reads for personalized, geo-specific data; edge computing workloads |
| **Lakehouse** (Delta Lake, Apache Iceberg, Hudi) | ACID transactions on top of columnar files in object storage (S3) | Unifying the data lake (cheap storage) with data warehouse query capabilities; time travel, schema evolution |
| **HTAP databases** (TiDB, SingleStore, Databricks) | Hybrid transactional + analytical processing on the same data | Eliminating the CDC pipeline between OLTP and OLAP; real-time analytics on live transactional data |
| **AI-native storage** (purpose-built for LLM training) | Optimized for the access patterns of model checkpointing, feature stores, and training data pipelines | ML platforms, LLM training infrastructure |

{{< callout type="info" >}}
In a system design interview, mentioning vector databases for AI-powered features (semantic search, recommendations using embeddings) signals awareness of modern architectures. Pinecone and Weaviate are managed; pgvector (PostgreSQL extension) is the pragmatic choice if you are already on PostgreSQL and embedding count is in the millions, not billions.
{{< /callout >}}
