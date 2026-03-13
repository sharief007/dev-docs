---
title: Time-Series Databases
weight: 10
type: docs
toc: true
sidebar:
  open: true
prev:
next:
params:
  editURL:
---

A time-series database is not just "a database with a timestamp column." It is a storage engine built around a specific and unusual combination of constraints: **extremely high write throughput, sequential time-ordered access, temporal compression, and automatic data expiry**. Understanding why each design decision exists is what makes this topic answerable at a FAANG interview.

## Why Not Just Use PostgreSQL?

This is the first question you should ask — and be able to answer — before mentioning any time-series database.

A Prometheus installation scraping 1,000 servers every 15 seconds produces ~400,000 data points per minute. An IoT platform with 100,000 sensors reporting every second produces 6 million data points per minute. Consider what happens if you store this in PostgreSQL:

| Problem | At scale |
|---------|---------|
| **Write throughput** | Each INSERT updates the B-tree index on the timestamp column — random I/O into a balanced tree. At millions of inserts/minute, index contention becomes the bottleneck. |
| **Storage volume** | 10,000 servers × 100 metrics × 86,400 seconds/day × 365 days = 315 billion rows/year. At 50 bytes/row = 15 TB/year, for one year of one fleet. |
| **Compression** | PostgreSQL compresses page-by-page. It cannot exploit the fact that `cpu_usage` for server-1 changes by small amounts every second — temporal correlation is invisible to a general-purpose compressor. |
| **Data expiry** | Metrics from 2 years ago are worthless. PostgreSQL has no automatic expiry. You write a cron job that runs `DELETE WHERE time < now() - interval '90 days'` — which generates dead tuples, requires VACUUM, and locks pages during bulk deletes. |
| **Downsampling** | "Show me avg CPU per hour for the last year" requires scanning 315 billion rows or building your own rollup pipeline. |
| **Query model** | Time-series queries are almost always `WHERE time BETWEEN ? AND ?` — range scans on the primary key. PostgreSQL can do this, but the storage engine is not optimized for it. |

Time-series databases solve all six problems by making the timestamp the organizing principle of every design decision.

## The Data Model

### Measurement / Tag / Field (InfluxDB model)

```
measurement: cpu_usage
tags:         host=server-1, region=us-east-1, env=prod     ← indexed, searchable
fields:       usage_idle=17.2, usage_user=63.8, usage_sys=19.0  ← not indexed, the actual values
timestamp:    1705225200000000000  (nanosecond precision)
```

**Measurement** — the name of what you are measuring (analogous to a table name).

**Tags** — metadata dimensions you filter and group by. **Tags are indexed.** Every unique combination of tag values defines one **time series** — the unit of data in a TSDB.

**Fields** — the actual numeric (or string) values being measured. **Fields are not indexed.** Querying on a field value requires a full scan of the time range.

**Why this model?** It matches how metrics are actually collected. A server has stable metadata (hostname, region, environment) — these are tags. The thing that changes every second (CPU usage, memory) — these are fields. Separating indexed dimensions from measured values keeps the index manageable.

### The Series Identity Problem

Every unique combination of tag values creates a separate **series** — its own sorted sequence of (timestamp, value) pairs.

```
cpu_usage,host=server-1,region=us-east-1     ← series 1
cpu_usage,host=server-1,region=us-west-2     ← series 2
cpu_usage,host=server-2,region=us-east-1     ← series 3
```

**Why this matters:** The index that maps series identifiers to their storage locations must fit in memory for fast lookups. If you have 10 tag keys with 100 distinct values each, the theoretical cardinality is 100^10 — impossible. Even 1 million active series can exhaust memory in naive implementations.

## The High Cardinality Problem

High cardinality is the most common operational failure mode in time-series systems. Understanding *why* it causes problems is the interview answer.

**What causes high cardinality:**

```
# Low cardinality tag — 3 possible values → 3 series per measurement ✅
status_code{method="GET", code="200"}

# High cardinality tag — millions of unique values → millions of series ❌
http_requests_total{endpoint="/users/42/orders", request_id="uuid-abc123"}
                                                  ↑
                                   unique per request — never do this
```

**Why it causes memory explosion:**

In InfluxDB, the Time Series Index (TSI) stores the mapping from series keys (tag combinations) to their chunk locations. In Prometheus, each active series requires memory for its WAL (write-ahead log) buffer, chunk head, and label set.

```
1 active series ≈ 1–2 KB memory in Prometheus
1 million series ≈ 1–2 GB
10 million series ≈ 10–20 GB  ← OOM territory on standard nodes
```

**Why high-cardinality series are different from high-cardinality SQL columns:** In a SQL database, a high-cardinality column like `user_id` is just a column — the index entry points to rows. In a TSDB, each unique tag combination is a separate time series with its own storage chunk, WAL buffer, and index entry. The overhead is structural, not just index size.

**The rule:** Tags must be bounded and stable. Values like `user_id`, `request_id`, `session_id`, `IP address`, or `trace_id` must **never** be tags. Store high-cardinality dimensions as fields (unindexed) or handle them in a separate system (distributed tracing for trace IDs, logs for request IDs).

## Storage Engine Internals

### Why Not a B-Tree?

A B-tree is optimized for random access — any key can be found in O(log n). Time-series data is never randomly accessed: writes always append at the current timestamp, and reads always scan a time range. Random-access capability is wasted overhead.

Time-series storage engines use an **LSM tree variant** with time-aware partitioning:

```
Writes → in-memory buffer (current time window)
              │  when full or time boundary crossed
              ▼
         Immutable chunk (e.g., 2-hour block of data, compressed)
              │  chunks accumulate
              ▼
         Retention expiry: drop entire chunks older than threshold
                           ← no row-level deletion needed
```

### Chunk-Based Storage

Data is organized into fixed time-window chunks (Prometheus: 2-hour blocks; TimescaleDB: configurable, default 7 days). Each chunk is a self-contained, compressed, sorted-by-time file.

**Why chunks?**

1. **Query efficiency:** A query for "last 1 hour" loads only the chunk(s) covering that time window — no scan of the full dataset.
2. **Compression:** A chunk contains data from a narrow time window. Within that window, consecutive values of the same metric are highly correlated → compression ratios of 10–100× are achievable.
3. **Retention:** Expiring old data means dropping entire chunk files. No scan, no VACUUM, no dead tuples — the file is simply deleted. O(1) expiry regardless of how many data points were in the chunk.
4. **Immutability:** Once a chunk's time window closes, it is sealed and never modified. Immutable chunks can be offloaded to object storage (S3) without consistency concerns.

### The Write Path

```
New data point arrives (timestamp, series_id, value)
        │
        ▼
WAL (Write-Ahead Log) — append only, for crash recovery
        │
        ▼
In-memory chunk (head block) — the current time window, mutable
        │  chunk time window closes (e.g., 2 hours elapsed)
        ▼
Compress and seal chunk → write to disk as immutable block
        │
        ▼
Index update: series_id → [chunk1_location, chunk2_location, ...]
```

## Compression in Depth

Time-series compression is where the biggest gains come from. General-purpose compression (gzip, Snappy) treats data as arbitrary bytes. Time-series-specific algorithms exploit the **temporal correlation** of the data.

### Delta-of-Delta Encoding for Timestamps

Timestamps in a time series arrive at regular intervals. If a metric is collected every 15 seconds, the timestamps look like:

```
Raw (nanoseconds):  1705225200000000000
                    1705225215000000000
                    1705225230000000000
                    1705225245000000000

Delta (ns):         15000000000, 15000000000, 15000000000
                    ← all identical

Delta-of-delta:     0, 0, 0
                    ← all zeros
```

**Encoding:** Zeros encode to 1 bit. Small values (missed scrape, minor drift) encode to a few bits. Large values (restart, gap) take more bits but are rare.

**Result (Facebook Gorilla paper):** Timestamps compressed from 64 bits to an average of **1.37 bits** for regular collection intervals — a 46× compression ratio.

When intervals are irregular (event data, user actions), deltas are non-zero but still small → still far better than storing 64-bit raw timestamps.

### Gorilla Compression for Float Values

Published by Facebook in 2015 (the Gorilla TSDB paper), this algorithm exploits the fact that consecutive values in a time series are **similar** — CPU usage doesn't jump randomly, it changes gradually.

**Algorithm (XOR-based):**

```
v1 = 63.5  (IEEE 754: 0 10000000101 1111110000000000...)
v2 = 63.8  (IEEE 754: 0 10000000101 1111111001100110...)

XOR:                  0 00000000000 0000001001100110...
                      ← sign and exponent bits are IDENTICAL (zero XOR)
                      ← only the mantissa tail differs
```

Store only the **meaningful bits** of the XOR result: their position, length, and value. If consecutive values are identical, store 1 control bit (0). If they differ, store position + length + bits.

**Result:** Real-world float metrics compress from 8 bytes to an average of **1.37 bytes** per data point — a ~6× compression on values alone. Combined with timestamp compression: raw ~16 bytes/point → compressed ~2.75 bytes/point on typical metrics.

**Why Gorilla fails for random data:** If values jump randomly (e.g., random sensor noise, UUID-like values), consecutive XORs have no leading zeros → no compression. This is why time-series compression is domain-specific.

### Run-Length Encoding for Tags and Low-Cardinality Fields

Tags are repeated for every data point in raw format. An indexed tag like `region=us-east-1` appears identically for every data point from servers in us-east-1.

Within a chunk (all data from one time window, one series): all tag values are constant — the series identity is fixed. Tags are stored once per chunk, not once per data point. This eliminates tag storage overhead entirely within a chunk.

For fields with low cardinality (e.g., `status="ok"` appears 10,000 consecutive times), run-length encoding stores `(value="ok", count=10000)` — constant space regardless of count.

## Retention Policies and Downsampling

### Why Retention Policies Exist

Raw metric data at 1-second resolution:
- 1 server, 50 metrics, 1 year = 1.57 billion data points
- 1,000 servers = 1.57 trillion data points
- At 2.75 bytes/point compressed = ~4.3 TB/year, just for metrics

**After 2 years, nobody queries 2-year-old 1-second CPU samples.** Retention policies automatically drop data older than the configured window, freeing storage without manual intervention.

```
# InfluxDB retention policy
CREATE RETENTION POLICY "90days" ON "metrics" DURATION 90d REPLICATION 1 DEFAULT

# Prometheus storage retention
--storage.tsdb.retention.time=15d    # default; adjust based on disk
--storage.tsdb.retention.size=50GB   # size-based cap
```

**Why chunk-based storage makes retention O(1):** When a 2-hour chunk's newest timestamp exceeds the retention window, the entire chunk file is deleted. No row scanning, no index cleanup, no VACUUM. The chunk is self-contained.

### Downsampling

Downsampling pre-aggregates high-resolution data into coarser granularities and stores the rollups indefinitely while the raw data expires.

```
Raw (1s resolution)   → keep 7 days
1-minute rollup       → keep 90 days
1-hour rollup         → keep 1 year
1-day rollup          → keep indefinitely
```

**Why this enables long-range queries:** "Show me average CPU per day for the last 3 years" hits the 1-day rollup table — ~1,095 data points. Without downsampling, it would scan 94 billion 1-second data points.

**Query routing:** The query engine automatically routes to the appropriate resolution based on the requested time range. Querying the last 5 minutes uses raw data; querying the last year uses pre-aggregated rollups.

**Implementation:**

{{< tabs items="InfluxDB,TimescaleDB,Prometheus" >}}
  {{< tab >}}
```sql
-- InfluxDB Task (continuous query replacement)
option task = {name: "downsample_1m", every: 1m}

from(bucket: "metrics/raw")
  |> range(start: -2m)
  |> aggregateWindow(every: 1m, fn: mean)
  |> to(bucket: "metrics/1m")
```
  {{< /tab >}}
  {{< tab >}}
```sql
-- TimescaleDB continuous aggregate (auto-refreshed materialized view)
CREATE MATERIALIZED VIEW cpu_1min
WITH (timescaledb.continuous) AS
SELECT
  time_bucket('1 minute', time) AS bucket,
  host, region,
  AVG(usage_percent) AS avg_usage,
  MAX(usage_percent) AS max_usage
FROM cpu_metrics
GROUP BY bucket, host, region;

-- Refresh policy
SELECT add_continuous_aggregate_policy('cpu_1min',
  start_offset => INTERVAL '2 minutes',
  end_offset   => INTERVAL '1 minute',
  schedule_interval => INTERVAL '1 minute');
```
  {{< /tab >}}
  {{< tab >}}
```yaml
# Prometheus recording rules (pre-compute expensive queries)
groups:
  - name: cpu_rollups
    interval: 1m
    rules:
      - record: job:cpu_usage:avg1m
        expr: avg by (job, region) (rate(cpu_usage_seconds_total[1m]))
      - record: job:http_requests:rate5m
        expr: sum by (job, status_code) (rate(http_requests_total[5m]))
```
  {{< /tab >}}
{{< /tabs >}}

## Query Patterns

Time-series queries differ from relational queries in important ways. Understanding the canonical patterns is part of what makes answers credible.

| Pattern | Description | Example |
|---------|-------------|---------|
| **Range scan** | All data points in a time window | `WHERE time BETWEEN now()-1h AND now()` |
| **Rate of change** | Per-second rate of a counter over a window | `rate(http_requests_total[5m])` — handles counter resets |
| **Rolling aggregation** | Sliding window avg, max, percentile | `AVG(cpu) OVER (ORDER BY time ROWS BETWEEN 59 PRECEDING AND CURRENT ROW)` |
| **Percentile** | p50/p95/p99 of latency over a period | `histogram_quantile(0.95, rate(request_duration_bucket[5m]))` |
| **Downsampled query** | Aggregate into time buckets | `time_bucket('1 hour', time)` in TimescaleDB |
| **Gap filling** | Fill missing data points | `time_bucket_gapfill` in TimescaleDB; `fill(null)` in InfluxQL |
| **Anomaly / threshold** | Alert when value crosses a threshold | `cpu_usage > 90 for 5m` in Prometheus alerting |
| **Derivative** | Rate of change (non-counter) | `derivative(value, 1s)` |

**Counter vs gauge:** A counter only goes up (total requests, bytes sent). A gauge can go up or down (current connections, memory used). Querying a counter directly is meaningless for "current rate" — you need `rate()` or `increase()` which compute the delta over the window and handle resets (counter reset to 0 on restart).

## Tools

### InfluxDB

Purpose-built TSDB. Uses the TSM (Time Structured Merge tree) storage engine in v1/v2 — an LSM tree variant organized around time series. InfluxDB v3 replaced TSM with Apache Parquet + DataFusion for long-term storage and SQL compatibility.

- **Data model:** measurement / tag / field
- **Query language:** InfluxQL (SQL-like) in v1; Flux (functional) in v2; SQL in v3
- **Strengths:** purpose-built for metrics and events; fast ingest; excellent compression
- **Cardinality limit:** TSI index works well up to ~10M series; beyond that, use purpose-built high-cardinality systems

### TimescaleDB

PostgreSQL extension that adds automatic time-based partitioning (hypertables), continuous aggregates, and time-series functions.

- **Data model:** standard SQL tables; `time` column is the partition key
- **Query language:** Full PostgreSQL SQL + time-series extensions (`time_bucket`, `first`, `last`, `locf`, `interpolate`)
- **Strengths:** full SQL, joins with relational data, PostgreSQL ecosystem (ORM, tooling, extensions), no separate infrastructure
- **When to use:** teams already on PostgreSQL; need joins between metrics and relational data; < ~100M rows/day; don't want to operate a separate database

```sql
-- Create a hypertable (partitioned by time automatically)
SELECT create_hypertable('metrics', 'time');

-- Query with time bucketing
SELECT time_bucket('5 minutes', time) AS bucket,
       host, AVG(cpu_usage) AS avg_cpu
FROM metrics
WHERE time > NOW() - INTERVAL '24 hours'
  AND host = 'server-1'
GROUP BY bucket, host
ORDER BY bucket;
```

### Prometheus

Pull-based metrics system designed for cloud-native environments. Prometheus scrapes HTTP endpoints (`/metrics`) on a configurable interval.

- **Data model:** float64 value + Unix timestamp + label set (key=value pairs)
- **Query language:** PromQL — purpose-built for rate/range queries and aggregations
- **Storage:** local TSDB, short-term (default 15 days). Not designed for long-term storage.
- **Long-term storage backends:** Thanos (object storage + global query), Cortex (horizontally scalable, multitenant), Victoria Metrics (high-performance drop-in), Grafana Mimir

**Pull vs push:** Prometheus pulls from targets rather than targets pushing to Prometheus. This means the infrastructure team controls scrape frequency and target discovery without requiring changes to the services being monitored. The tradeoff: push-based (InfluxDB line protocol, StatsD, OpenTelemetry) works better for short-lived jobs (batch jobs, serverless) — Prometheus Pushgateway is the workaround.

**Cardinality in Prometheus:** The `--storage.tsdb.head-chunks-write-queue-size` and memory limits apply per active series. High-cardinality labels (`user_id`, `request_id`) OOM Prometheus reliably. Use exemplars + distributed tracing (Tempo, Jaeger) for request-level data; Prometheus is for aggregate metrics only.

### ClickHouse (for high-scale time-series analytics)

ClickHouse is a columnar OLAP database, not a purpose-built TSDB — but it is widely used for time-series analytics at extreme scale (Cloudflare, Uber, eBay).

- **Why it works:** columnar storage + vectorized execution + ReplacingMergeTree / SummingMergeTree engines handle time-range aggregations faster than any purpose-built TSDB at billion-row scale
- **When to use:** analytics-heavy time-series (logs, clickstream, billing events) where you need arbitrary SQL aggregations; not for monitoring/alerting loops

## When a Regular RDBMS Is Enough

| Condition | Recommendation |
|-----------|---------------|
| < 1 million data points/day | Plain PostgreSQL with a `time` index |
| Metrics alongside relational data (user events joined to user table) | TimescaleDB (PostgreSQL extension, zero infrastructure change) |
| < 10 million active series | InfluxDB or TimescaleDB |
| Kubernetes / cloud-native monitoring | Prometheus + Grafana |
| > 10 million active series or multi-tenant | Victoria Metrics, Thanos, Grafana Mimir |
| Time-series analytics at petabyte scale | ClickHouse |
| IoT ingest > 1 million writes/sec | InfluxDB v3, TimescaleDB with Citus, or purpose-built pipelines (Kafka → ClickHouse) |

{{< callout type="warning" >}}
The most expensive mistake in time-series systems is storing request-level or user-level data in a TSDB using high-cardinality tags. Use a TSDB for **aggregate** metrics (rate of requests, p99 latency per service). Use distributed tracing (Jaeger, Tempo) for per-request data and use logs (Elasticsearch, Loki) for per-event data. These are three distinct systems solving three distinct problems.
{{< /callout >}}
