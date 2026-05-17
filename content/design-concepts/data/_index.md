---
title: Data Engineering
weight: 15
type: docs
toc: false
---

{{< cards >}}
  {{< card link="batch-vs-streaming" title="Batch vs Streaming" subtitle="Bounded vs unbounded data, MapReduce to Spark, stream processing with Flink/Kafka Streams, Lambda vs Kappa architecture, and when to choose each" >}}
  {{< card link="stream-processing-engines" title="Stream Processing Engines" subtitle="Flink internals (checkpointing, RocksDB state, watermarks), Kafka Streams (library model, changelog-backed state), Spark Structured Streaming (micro-batch), and engine selection framework" >}}
  {{< card link="data-warehouse" title="Data Warehouse Basics" subtitle="OLTP vs OLAP workloads, columnar storage and compression, star schema with fact/dimension tables, materialized views, and Snowflake vs BigQuery vs Redshift" >}}
  {{< card link="change-data-capture" title="Change Data Capture (CDC)" subtitle="Reading database transaction logs, Debezium with PostgreSQL WAL and MySQL binlog, cache/search/warehouse sync, outbox pattern integration, and schema evolution" >}}
{{< /cards >}}
