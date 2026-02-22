# L7: Data Engineering

> The layer that processes, stores, and serves data at scale. Essential for any data-intensive application.

---

## 1. Architecture Paradigms

### Lambda Architecture

```
Raw Data ──→ Batch Layer (Hadoop/Spark) ──→ Serving Layer ──→ Query
         ↘  Speed Layer (Flink/Kafka)  ──→ (merges views)
```

- **Batch layer**: recomputes full historical dataset (accuracy)
- **Speed layer**: processes recent data in real-time (low latency)
- **Serving layer**: merges batch and speed views
- **Problem**: two codebases to maintain (batch + streaming logic often duplicated)

### Kappa Architecture

```
Raw Data ──→ Kafka (immutable log) ──→ Stream Processing ──→ Serving Layer
```

- **Only streaming**: process everything as streams (batch = slow stream)
- Replay from Kafka to recompute if needed
- Simpler: one codebase for real-time and historical
- **Limitation**: replaying years of data is slow (Kafka retention limits)

### Delta Architecture

```
Raw Data ──→ Delta Lake (object storage) ──→ Unified Query Layer ──→ Query
              (ACID transactions, versioning)
```

- Delta Lake enables ACID on data lake (Parquet on S3/ADLS)
- Time travel: query historical versions of data
- Unified batch + streaming (Delta Lake supports both)

### Data Mesh (Zhamak Dehghani)

- **Domain-oriented ownership**: each team owns their data products
- **Data as a product**: data has SLAs, documentation, quality guarantees
- **Self-serve data infrastructure**: platform team provides tooling
- **Federated governance**: global policies, local implementation
- Contrast with: centralized data team (Data Warehouse era), centralized data lake (Data Lake era)

---

## 2. Batch Processing

### Hadoop Ecosystem

**HDFS** (already covered in Storage):
- NameNode (metadata), DataNodes (blocks), 128MB blocks, 3× replication

**YARN** (Yet Another Resource Negotiator):
- Cluster resource manager (CPU, memory)
- ResourceManager: global scheduler
- NodeManager: per-node agent
- ApplicationMaster: per-application scheduler (MapReduce, Spark, etc.)

**MapReduce**:
- Map: process input splits → key-value pairs
- Shuffle: group by key, sort
- Reduce: aggregate per key
- Written entirely to HDFS between stages (slow — Spark replaced this)

### Apache Spark

**Core Concepts**:
- **RDD (Resilient Distributed Dataset)**: immutable distributed collection, lazy evaluation
- **DataFrame**: distributed table with schema (like SQL table), preferred over RDD
- **Dataset**: typed DataFrame (Java/Scala only)
- **Lazy evaluation**: transformations are not computed until action triggered
- **Lineage**: Spark tracks transformations to recover from failures (no replication needed)

**Execution Model**:
```
Job: one Spark action (collect, count, write)
  ↳ Stage: set of transformations between shuffles
      ↳ Task: one partition processed by one executor core
```

**Shuffles** (expensive):
- Operations that require data movement: groupBy, join, repartition, sortBy
- Data written to disk, transferred over network
- Optimization: use broadcast join for small tables (no shuffle for small side)

**Memory Management**:
- Execution memory: for shuffles, sorts, aggregations
- Storage memory: for caching RDDs/DataFrames
- Off-heap memory: for Tungsten binary format (avoids GC)
- Garbage collection: major source of slow jobs; use Kryo serialization, avoid large Java objects

**Catalyst Optimizer** (Spark SQL):
- Parses SQL → logical plan → optimized logical plan → physical plan → code generation
- Predicate pushdown: filter early, reduce data scanned
- Projection pushdown: only read needed columns
- Join reordering: smaller tables joined first

**Broadcast Variables**: send large lookup tables to all executors (avoids shuffle)
**Accumulators**: distributed counters/summers (metrics, error counts)

**Spark Structured Streaming**:
- Treat stream as unbounded table
- Same DataFrame API
- **Output modes**: complete (rewrite all), append (new rows only), update (changed rows)
- **Watermarks**: handle late data — discard data older than watermark
- **Checkpointing**: save streaming state for fault tolerance

### Presto / Trino

- Distributed SQL query engine (not an ETL tool)
- Queries data in-place: S3/HDFS (Parquet, ORC), Hive, databases
- No data movement: pushes queries to data sources
- Good for: ad-hoc interactive analytics on large datasets
- **MPP** (Massively Parallel Processing): splits query into fragments executed in parallel

### Apache Beam

- Unified model for batch + streaming (one API, multiple runners)
- Runners: Dataflow (Google), Spark, Flink, Samza, Apex
- **Windowing**: fixed, sliding, session windows
- **Triggers**: when to fire window results
- **Accumulation**: how to combine results with late data

---

## 3. Stream Processing

### Core Stream Processing Concepts

**Event Time vs Processing Time vs Ingestion Time**:
- **Event time**: when event occurred (on device)
- **Ingestion time**: when event arrived at message broker
- **Processing time**: when event was processed by streaming job
- Use event time for correctness; deal with late data via watermarks

**Watermarks**:
- Tracks progress of event time in the stream
- "I'm confident no event with timestamp < W will arrive"
- **Heuristic watermarks**: estimate based on observed event times - lag
- Late elements: arrive after watermark → either discard or reprocess with allowed lateness

**Windows**:
- **Tumbling**: fixed non-overlapping windows (each event in exactly one window)
- **Sliding**: overlapping windows (event may be in multiple windows)
- **Session**: activity-based gaps (window closes after idle period)
- **Global**: entire stream as one window (with custom triggers)

### Apache Flink

**Architecture**:
- **JobManager**: coordinates execution (like Spark driver)
- **TaskManagers**: execute tasks (like Spark executors)
- **Savepoints**: manual snapshots for versioned restarts
- **Checkpoints**: automatic snapshots for fault recovery

**DataStream API**:
```java
stream
  .keyBy(event -> event.userId)
  .window(TumblingEventTimeWindows.of(Time.minutes(5)))
  .aggregate(new CountAggregate())
```

**Time Semantics**: Flink is the gold standard for event-time processing
- **Watermark generation**: PeriodicWatermarkStrategy, BoundedOutOfOrdernessWatermarks

**State Backends**:
- **HashMapStateBackend**: in heap memory (fast, limited size)
- **EmbeddedRocksDBStateBackend**: RocksDB on disk (large state, slower)
- **State TTL**: expire old state entries automatically

**Checkpointing**:
- Barriers injected into streams, flow through operators
- Operator saves state when barrier passes
- Distributed snapshot of entire streaming job
- On failure: restart from last successful checkpoint

**Exactly-Once Semantics**:
- Flink internal: checkpoint-based exactly-once
- End-to-end: requires transactional sinks (Kafka transactions, idempotent writes)

**Table API / Flink SQL**:
- SQL on streams (with time-aware semantics)
- TUMBLE, HOP, SESSION window functions
- Temporal joins (join with dimension table at event time)

### Kafka Streams

- Library (not a separate cluster) — runs inside your application
- **KTable**: changelog stream as materialized view
- **KStream**: unbounded event stream
- Joins: stream-stream (windowed), stream-table, table-table
- State stores: local RocksDB per partition, changelog topic for recovery
- **Interactive Queries**: query state store via REST API

### Amazon Kinesis

- AWS managed streaming service (Kafka alternative)
- **Shards**: unit of throughput (1MB/s write, 2MB/s read per shard)
- **Kinesis Data Firehose**: load streaming data to S3/Redshift/Elasticsearch
- **Kinesis Data Analytics**: Flink-based managed stream processing

---

## 4. Data Warehouses

### OLAP vs OLTP

| | OLTP | OLAP |
|--|------|------|
| Workload | Many short transactions | Few long analytical queries |
| Data size | GB–TB | TB–PB |
| Schema | Normalized (3NF) | Denormalized (star/snowflake) |
| Queries | Row-based | Column-based |
| Indexes | Many (B-tree) | Few (bitmap, zone maps) |
| Examples | PostgreSQL, MySQL | Redshift, BigQuery, Snowflake |

### Columnar Storage

- Store each column contiguously (vs row store: each row contiguously)
- **Benefits for analytics**:
  - Read only columns needed (projection pushdown)
  - Better compression: same-type data with similar values (run-length, dictionary encoding)
  - Vectorized execution: SIMD operations on column arrays
- **Parquet**: Apache columnar format, widely supported
  - Row groups → column chunks → pages
  - Dictionary encoding, RLE, Bit packing, Delta encoding

### Data Warehouse Design

**Star Schema**:
```
           ┌─ dim_date
           ├─ dim_customer
fact_sales ─┤
           ├─ dim_product
           └─ dim_store
```
- Fact table: metrics (revenue, quantity)
- Dimension tables: descriptive attributes (customer name, product name, date)
- Simple joins, good for most BI queries

**Snowflake Schema**: normalized dimensions (dimensions have sub-dimensions)
- Less redundancy, more complex joins
- Usually not worth the complexity over star schema

**Slowly Changing Dimensions (SCD)**:
- Type 1: overwrite (no history)
- Type 2: add new row with effective date range (most common — keeps full history)
- Type 3: add "previous" column
- Type 4: separate history table
- Type 6: hybrid

### Snowflake (Cloud Data Warehouse)

- **Virtual Warehouses**: separate compute clusters (scale independently per workload)
- **Storage**: shared S3/Azure Blob (all warehouses query same data)
- **Multi-cluster**: auto-scale by adding more clusters in same virtual warehouse
- **Time travel**: query data as it was at any point in past 90 days
- **Zero-copy clone**: clone table/schema/database instantly (just copy metadata)
- **Data sharing**: share data across Snowflake accounts without copying

### Google BigQuery

- Serverless: no cluster management, pay per query
- **Capacitor** format: proprietary columnar (better than Parquet for BQ)
- **Dremel** execution engine: tree of query servers (root → mixers → leaf servers)
- Slot-based pricing: 1 slot = 1 virtual CPU for query execution
- **Partitioning**: by ingestion time, date/timestamp column, integer range
- **Clustering**: sort data within partitions by columns (like an index)
- **Materialized Views**: auto-refresh on base table changes

### Amazon Redshift

- Column-oriented MPP database
- **Distribution styles**: EVEN (round robin), KEY (by column value), ALL (broadcast small tables)
- **Sort keys**: controls physical sort order (like clustered index) — interleaved vs compound
- Redshift Spectrum: query S3 data directly from Redshift
- RA3 nodes: separate compute and storage (like Snowflake)

---

## 5. Data Lakes

### Delta Lake

- Open-source ACID table storage layer on top of Parquet + object storage
- **Transaction log** (JSON files in `_delta_log`): all changes tracked
- **ACID**: optimistic concurrency control via transaction log
- **Time travel**: `SELECT * FROM table VERSION AS OF 5`
- **Schema enforcement**: reject data that doesn't match schema
- **Schema evolution**: add columns, rename columns
- **Merge (UPSERT)**: `MERGE INTO target USING source ON target.id = source.id ...`
- **Optimize**: Z-order clustering for multi-dimensional queries

### Apache Iceberg

- Table format for huge analytic tables (Netflix, Apple)
- **Hidden partitioning**: partition columns hidden from users, auto-applied
- **Partition evolution**: change partition strategy without rewriting data
- **Time travel**: via snapshot history
- **Concurrent writers**: optimistic concurrency with atomic commit
- Supported by: Spark, Flink, Hive, Trino, Dremio

### Apache Hudi (Hadoop Upserts Deletes Incrementals)

- Enables record-level updates and deletes on data lake (hard with plain Parquet)
- **Copy-on-Write (CoW)**: on write, rewrite affected Parquet files — good for reads
- **Merge-on-Read (MoR)**: write delta files, merge on read — good for writes
- **Incremental queries**: read only records changed since given timestamp
- Used by: Uber (original creators), AWS, Alibaba

### Data Catalog

- Central repository of metadata about data assets
- **Apache Atlas**: open-source, lineage tracking, classification
- **AWS Glue Catalog**: managed, integrates with Athena/Redshift/EMR
- Features: schema discovery, column-level lineage, data classification (PII), search

---

## 6. ETL / ELT Pipelines

### Apache Airflow

- **DAG** (Directed Acyclic Graph): workflow definition in Python
- **Operators**: units of work (BashOperator, PythonOperator, SparkSubmitOperator, etc.)
- **Sensors**: wait for external condition (S3KeySensor, ExternalTaskSensor)
- **Hooks**: connection to external systems (database, S3, REST API)
- **XCom**: pass small data between tasks (not for large datasets)
- **Executor**: how tasks run (LocalExecutor, CeleryExecutor, KubernetesExecutor)
- **Scheduler**: checks DAG every ~30s for eligible tasks
- **Catchup**: backfill historical runs automatically

**Airflow Pitfalls**:
- Don't do heavy computation in DAG file (just define workflow)
- Avoid cross-DAG dependencies where possible
- DAG file must import cleanly every 30 seconds (no slow imports)

### dbt (Data Build Tool)

- Transform data already in warehouse using SQL + Jinja
- **Models**: SELECT statements that define transformations (materializations: table, view, incremental, ephemeral)
- **Incremental models**: only process new/changed data (`{{ is_incremental() }}`)
- **Tests**: built-in (unique, not_null, accepted_values, relationships) + custom
- **Documentation**: auto-generates data catalog from model descriptions
- **Lineage**: auto-generates DAG of model dependencies
- Used with: Snowflake, BigQuery, Redshift, Databricks, DuckDB

### Change Data Capture (CDC) with Debezium

- Reads database transaction log (WAL/binlog/oplog) in real-time
- Publishes changes as events to Kafka topics
- **Connectors**: PostgreSQL (logical decoding), MySQL (binlog), MongoDB (oplog), SQL Server (CDC API)
- **Event format**: `before` + `after` state of each row
- **Use cases**: sync to search index (Elasticsearch), populate caches (Redis), event sourcing, CQRS

**PostgreSQL Logical Decoding**:
```sql
-- Enable logical replication
ALTER SYSTEM SET wal_level = logical;
-- Create replication slot
SELECT * FROM pg_create_logical_replication_slot('debezium', 'pgoutput');
```

### Schema Evolution and Migration

- **Backward compatible**: new schema can read old data
- **Forward compatible**: old schema can read new data
- **Full compatible**: both directions
- Avro, Protobuf, JSON Schema support evolution with Schema Registry
- Database migrations: Flyway (versioned), Liquibase (XML/YAML changelogs), Alembic (Python/SQLAlchemy)

---

## 7. Feature Engineering

### Feature Stores

**Problem**: same features computed independently by many ML teams → inconsistency, duplicated effort

**Feature Store Components**:
- **Offline store**: historical features for training (data warehouse, Delta Lake)
- **Online store**: low-latency feature serving for inference (Redis, DynamoDB)
- **Feature transform**: computation logic (registered once, reused everywhere)
- **Point-in-time join**: training data uses feature values as they were at prediction time (no data leakage)

**Point-in-Time Correctness**:
```
Training: for each example at time T, use feature values as they existed at T
Wrong:    use current feature values (different from what model saw at inference time)
Result of bug: training-serving skew → model performance degrades in production
```

**Feature Stores**:
- **Feast**: open-source, flexible, integrates with Spark/Redis
- **Tecton**: managed, production-grade, real-time feature computation
- **Vertex AI Feature Store**: Google managed
- **Databricks Feature Store**: integrated with MLflow

### Feature Types

- **Batch features**: computed in batch jobs (user total purchases last 30 days)
- **Real-time features**: computed on-the-fly from streaming data (last 5 clicks in session)
- **On-demand features**: computed at request time from request context (time of day, device type)
- **Pre-computed features**: computed offline, served from feature store (user embeddings)

### Training-Serving Skew

Most common ML production bug:
- Feature computation differs between training and serving
- Prevention: share feature computation code between training and serving
- Feature store enforces consistency by having one feature definition served in both contexts
