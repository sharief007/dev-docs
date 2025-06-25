---
title: 'High Level Design'
weight: 4
toc: false
sidebar:
  open: false
prev: 
next:
params:
  editURL: 
---

## Why Two Storage Layers: Time-Series Aggregate DB & Long-Term Storage (S3)

Yes, there are two primary reasons for having both a "Real-time Aggregate Database" (like Druid/ClickHouse/Pinot) and a "Long-term Storage" (like S3) for the aggregated (and potentially raw) data:

1.  **Optimized for Different Query Patterns & Latencies:**
    * **Real-time Aggregate DB (e.g., Druid):**
        * **Purpose:** Designed for extremely fast, low-latency analytical queries over recent, aggregated time-series data. Think "P90 latency in milliseconds to seconds."
        * **Data:** Primarily stores the minute-level pre-aggregated counts (e.g., total clicks per ad_id per minute, top 100 ad_ids per minute).
        * **Query Types:** Ideal for our core requirements: "clicks for ad_id in last M minutes" and "top 100 ads in last M minutes."
        * **Cost:** Generally more expensive per GB than object storage due to specialized indexing and in-memory/SSD optimizations. Data is often kept here for a shorter "hot" window (e.g., last few days to a few months).
    * **Long-term Storage (e.g., S3/HDFS - Data Lake):**
        * **Purpose:** Cost-effective, highly scalable storage for vast amounts of data, often in its raw or semi-processed form. Think "petabytes of data for years." It's not optimized for sub-second query latency.
        * **Data:** Stores the **raw click events** (for auditing, re-processing, ad-hoc deep analysis) and/or the **aggregated data** (as a cost-effective historical archive).
        * **Query Types:** Used for:
            * **Ad-hoc, complex queries:** Deeper dive analytics that might not be pre-aggregated (e.g., "how many distinct users from a specific city clicked on ads from a particular advertiser last quarter?"). This is where tools like Presto/Trino, Spark SQL, or Athena come in.
            * **Historical Analysis:** Queries spanning months or years where latency isn't critical.
            * **Data Reconciliation/Re-processing:** If there's an error in the real-time pipeline, or a very late event arrives outside the watermark window, raw data from S3 can be re-processed in batch to fix or update historical aggregates.
        * **Cost:** Much cheaper per GB, making it suitable for indefinite retention.

2.  **Addressing the Filtering Challenge (IP, User ID, Country):**

In data, **cardinality refers to the number of unique values in a particular column (or dimension/attribute).** It describes the uniqueness of data values in that column.

Think of it like this:

* **Low Cardinality:** A column has a small, limited number of unique values that repeat frequently.
* **High Cardinality:** A column has a very large number of unique values, potentially almost all values being unique.


### Cardinality in Our Ad Click System

Now, let's apply this to our ad click scenario:

* **`country`:**
    * **Number of Unique Values:** Around 200 (number of countries in the world).
    * **Cardinality:** **Low**
    * *Implication:* We *can* easily pre-aggregate clicks by country (`clicks_US`, `clicks_IN`, `clicks_GB`) in our real-time database. The schema doesn't explode because there are only 200 possible columns/combinations.

* **`ad_id`:**
    * **Number of Unique Values:** Roughly 2 million unique ads.
    * **Cardinality:** **Medium to High**
    * *Implication:* This is manageable. We *can* aggregate `total_clicks` per `ad_id` per minute. $2 \text{ million } \times 60 \text{ minutes/hour} \times 24 \text{ hours/day}$ is still a lot of data points, but it's a fixed upper bound and time-series databases are built for this. This is why we can get "Top N Most Clicked Ads" in real-time.

* **`user_id`:**
    * **Number of Unique Values:** Potentially hundreds of millions or billions of unique users over time.
    * **Cardinality:** **Very High**
    * *Implication:* **We cannot pre-aggregate "clicks per user per ad per minute" in our real-time DB.** If you try to store `(timestamp, ad_id, user_id, clicks)`, and you have 2 million ads and 500 million users, the number of possible `(ad_id, user_id)` combinations is $2 \text{ million } \times 500 \text{ million } = 10^{15}$. Storing counts for every such combination, even if most are zero, is simply impossible for a real-time aggregate database. It would explode its storage and make queries incredibly slow.

* **`ip`:**
    * **Number of Unique Values:** Billions of possible IPv4 addresses, even more IPv6. Many users might share IPs (e.g., behind a NAT), but still, the distinct number of IPs seen per minute could be in the millions.
    * **Cardinality:** **Very High**
    * *Implication:* Similar to `user_id`, you cannot pre-aggregate "clicks per IP per ad per minute." The data volume would be astronomical and unmanageable for real-time aggregation.


This is precisely why we store the **raw data (which includes `ip` and `user_id`) in S3**. Then, when a query needs to filter by a specific `ip` or `user_id`, we use a tool like Presto/Trino that can efficiently scan and filter the massive, raw dataset in S3, accepting that these specific queries will have higher latency (seconds to minutes) than the pre-aggregated queries from Druid.


## Long-Term Storage (S3 / Data Lake)

**Primary Goals:** Cost-effective, scalable, durable storage for raw/historical data, enabling flexible ad-hoc queries and batch processing. (e.g., AWS S3).

### 1. Data Organization: Splitting & Partitioning

* **Strategy:** **Time-based partitioning** is crucial for efficiency.
    * **Hierarchy:** `s3://ad-clicks-data-lake/raw_events/year=YYYY/month=MM/day=DD/hour=HH/`
    * **Benefit:** Enables **query pruning** (reading only relevant data) and improves manageability/retention policies.
* **File Size:** Aim for **128 MB to 1 GB per file** to balance parallelism and small file overhead.

### 2. File Format, Data Model, Compression

* **File Format:** **Apache Parquet** or **Apache ORC**.
    * **Why Columnar:**
        * **Query Performance:** Reads *only necessary columns* (reduces I/O).
        * **Compression:** Better ratios (similar data types together).
        * **Predicate Pushdown:** Skips irrelevant data blocks.
        * **Schema Evolution:** Supports adding new columns easily.
* **Data Model:** Store **raw, immutable click events** (source of truth).
    * **Schema:** `click_timestamp (LONG), user_id (STRING), ad_id (LONG), ip (STRING), country (STRING), event_id (STRING/Hash)`.
    * **High Cardinality:** Raw data here allows filtering on high-cardinality `user_id` and `ip` via ad-hoc query engines.
* **Compression:** **Snappy** (good balance) or **Zstd** (better ratio/speed). Gzip for archives.

### 3. Other Key Considerations

* **Ingestion:** Stream processor writes partitioned Parquet/ORC files to S3.
* **Schema Management:** Use a **Schema Registry** (e.g., Confluent Schema Registry) for schema evolution and data quality.
* **Data Catalog:** Use a **Catalog** (e.g., AWS Glue, Hive Metastore) for metadata, allowing query engines (Presto/Trino, Spark SQL) to query S3 data as tables.
* **Idempotency:** Ensure writes from the stream processor to S3 are **idempotent** to prevent duplicates upon retries/failures.
* **Data Lakehouse:** Consider **Apache Iceberg/Delta Lake/Apache Hudi** for future transactional capabilities, schema enforcement, and versioning on top of the lake.