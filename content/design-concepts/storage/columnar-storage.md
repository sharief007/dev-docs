---
title: Columnar Storage
weight: 12
type: docs
---

Row-oriented and columnar storage are two fundamentally different ways to lay out table data on disk. The choice determines which workloads are fast and which are expensive. OLTP systems use row storage; OLAP / analytics systems use columnar.

## Row-Oriented Layout

In a row store, all columns of a row are written and stored contiguously.

```
Disk layout (row store):
[id=1, name="Alice", age=30, country="US"] [id=2, name="Bob", age=25, country="UK"] [id=3, …]
```

**Reads fetch entire rows.** A query for `SELECT * FROM users WHERE id = 42` reads one contiguous block and has everything it needs. An analytics query like `SELECT AVG(age) FROM users` must read all rows and deserialize every column — including `name` and `country` — even though only `age` is needed.

## Columnar Layout

In a column store, each column is stored independently as a contiguous block.

```
Disk layout (columnar):
[id]:      [1, 2, 3, 4, 5, …]
[name]:    ["Alice", "Bob", "Carol", …]
[age]:     [30, 25, 34, 28, …]
[country]: ["US", "UK", "US", "DE", …]
```

**Reads fetch only the columns the query touches.** `SELECT AVG(age)` reads only the `age` column — skipping `id`, `name`, and `country` entirely. At petabyte scale, skipping unused columns is the difference between reading 1 TB and 1 GB.

## Side-by-Side Comparison

| | Row Store | Columnar Store |
|---|---|---|
| **Storage layout** | Entire row contiguous | Each column contiguous |
| **Point lookups (by PK)** | ✅ Fast — one I/O for all columns | ❌ Slow — separate read per column |
| **Full-row INSERT / UPDATE** | ✅ One write per row | ❌ N writes (one per column) |
| **Aggregations (SUM, AVG, COUNT)** | ❌ Must read all columns | ✅ Read only the target column |
| **Projections (SELECT col1, col2)** | ❌ Always reads full row | ✅ Only touched columns loaded |
| **Wide tables (100+ columns)** | ❌ Wastes I/O on unused columns | ✅ Column pruning scales with width |
| **Compression ratio** | Moderate | High (same-type values compress better) |
| **Cache efficiency** | Good for single-row access | Good for column scans |
| **Best workload** | OLTP (transactional, single-row) | OLAP (analytics, large aggregations) |

## Why Columnar Compresses Better

A column contains values of a single type, often with limited cardinality or monotonic patterns. This is ideal for compression algorithms that exploit repetition and predictability.

**Run-Length Encoding (RLE):** Replace consecutive repeated values with (value, count).

```
[US, US, US, US, UK, UK, US, US]  →  [(US,4), (UK,2), (US,2)]
```

Highly effective on low-cardinality columns after sorting (e.g., `country`, `status`).

**Dictionary Encoding:** Replace repeated strings with small integer codes.

```
["engineering", "product", "engineering", "sales", "product"]
→ dict: {0:"engineering", 1:"product", 2:"sales"}
→ data: [0, 1, 0, 2, 1]
```

Reduces string storage to 1–4 bytes per value. Used by Parquet and most columnar stores by default.

**Delta Encoding:** Store differences between consecutive values instead of absolute values.

```
timestamps: [1700000000, 1700000060, 1700000120, 1700000180]
→ base=1700000000, deltas: [0, 60, 60, 60]
```

Effective for monotonically increasing columns — timestamps, auto-increment IDs.

**Bit Packing:** If a column's values fit in fewer than 64 bits, pack them tightly.

```
column "rating" (values 1–5) → 3 bits per value instead of 64
```

**Combined effect:** A 1 TB row-store table commonly compresses to 100–200 GB in Parquet with Snappy, and to 50–100 GB with Zstandard. Savings are both storage cost and I/O bandwidth on every query.

## Vectorized Execution

Columnar storage enables a fundamentally different query execution model. Instead of processing one row at a time, the engine processes a column chunk (e.g., 1024–8192 values) in a tight loop.

```
Row-at-a-time (row store):
  for each row:
    if row.age > 30: sum += row.age   ← branch + field access per row

Vectorized (columnar):
  age_chunk = load_column_chunk(age, offset=0, len=1024)
  mask = age_chunk > 30              ← SIMD: compare 8–16 values per CPU instruction
  sum = masked_sum(age_chunk, mask)  ← SIMD: sum 8–16 values per CPU instruction
```

Modern CPUs execute SIMD instructions (AVX-512) that operate on 512-bit registers — processing 8 × 64-bit integers or 16 × 32-bit integers in a single instruction. Columnar layouts make this trivial; row layouts require scatter/gather to extract a single column's values.

DuckDB, Snowflake, ClickHouse, and BigQuery all use vectorized columnar execution.

## File Formats

| Format | Storage | Used by | Key feature |
|--------|---------|---------|-------------|
| **Parquet** | Columnar | Spark, Hive, BigQuery, Redshift Spectrum, Athena, Pandas | Nested data support (structs, lists); widely supported |
| **ORC** | Columnar | Hive, Presto, Spark (Hive ecosystem) | Better predicate pushdown statistics; lighter than Parquet for Hive |
| **Arrow (IPC)** | Columnar in-memory | In-process exchange (Spark↔Python, DuckDB, Pandas) | Zero-copy inter-process; not an on-disk format |
| **CSV / JSON** | Row (text) | Universal ingestion | No compression, no schema, slow for analytics |
| **Avro** | Row (binary) | Kafka, schema registry | Schema evolution; compact binary row format |

**Parquet row groups:** Parquet files are divided into row groups (default 128 MB). Each row group stores all columns for that row range, with min/max statistics per column chunk. A query filtering on `WHERE year = 2024` can skip entire row groups without decompressing them — this is **predicate pushdown**.

```
Parquet file layout:
  Row group 1 (rows 0–99999):   col: year [min=2023, max=2023] → skip if year=2024
  Row group 2 (rows 100000–…):  col: year [min=2024, max=2024] → read
```

## When to Use Each

| Scenario | Storage choice | Reason |
|----------|---------------|--------|
| Web app database (users, orders, sessions) | Row (PostgreSQL, MySQL) | Single-row lookups, frequent updates |
| Event / clickstream analytics | Columnar (BigQuery, Redshift, ClickHouse) | Aggregations over billions of events across few columns |
| ML feature store (offline) | Parquet on object storage | Column pruning for feature selection; Spark/Pandas read natively |
| Time-series metrics | Columnar (InfluxDB, TimescaleDB, ClickHouse) | High cardinality timestamps, delta encoding, range aggregations |
| Operational reporting on live data | Hybrid (PostgreSQL with columnar extension, Citus, Aurora) | HTAP: transactional writes + analytical reads on same data |
| Data lake (raw → processed) | Parquet or ORC | Open format; any engine can read; schema evolution via Avro for ingest |

{{< callout type="info" >}}
A common architecture at FAANG: OLTP data in MySQL/PostgreSQL → CDC pipeline (Debezium/Kafka) → Parquet on S3 → queried by Athena/Spark/BigQuery. The row store handles transactions; the columnar layer handles analytics. This separation avoids OLAP queries degrading OLTP latency.
{{< /callout >}}

{{< callout type="info" >}}
**Interview tip:** If the question is "how do we run analytics on a billion rows?", I'd say: "Don't run it on the OLTP database — push the data into a columnar store." Columnar layouts read only the columns the query touches, compress 5–10x better than row stores because each column is a single type with high repetition (RLE, dictionary, delta, bit-packing), and enable vectorized SIMD execution that processes thousands of values per CPU instruction. The standard architecture I'd propose is OLTP in PostgreSQL → CDC into Parquet on S3 → queried by Athena, Spark, or ClickHouse. I'd flag that columnar is wrong for point lookups and updates — every row touches N column files — so the row store stays authoritative for transactions and the columnar layer is a derived view.
{{< /callout >}}
