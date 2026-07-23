---
title: 'Distributed Key-Value Store'
weight: 1
type: docs
---

A **distributed key-value store** is the foundational building block behind nearly every large-scale internet service. Amazon Dynamo powers DynamoDB and S3 metadata; Apache Cassandra stores user activity at Netflix and Instagram; Redis is the industry-standard in-memory KV layer. The interface is deceptively simple — `GET(key)`, `PUT(key, value)`, `DELETE(key)` — but building a version that stays available through node failures, serves hundreds of thousands of operations per second, durably persists data across restarts, and gives operators knobs to tune consistency vs latency is what makes it genuinely hard.

You are asked to design such a store from scratch: start with a single-node in-memory engine, evolve it to a disk-backed LSM-tree, and then distribute keys across a cluster with consistent hashing and Dynamo-style leaderless replication with tunable quorums. The design should resemble what DynamoDB, Cassandra, or Riak expose internally.

## Functional Requirements

1. **GET(key):** Return the value for a key, or a "not found" response if the key does not exist or has expired.
2. **PUT(key, value):** Store or overwrite a key-value pair. Values are arbitrary bytes up to a configured maximum (default 10 MB).
3. **DELETE(key):** Remove a key-value pair from the store. Idempotent — deleting an absent key succeeds silently.
4. **TTL / expiration:** Optionally attach a time-to-live when writing a key; expired keys behave as absent without requiring client-driven cleanup.
5. **Fault tolerance:** The cluster continues to serve reads and writes even when individual nodes fail, without operator intervention.

## Out of Scope

- Rich query language, secondary indexes, or SQL-like filtering — this is a KV store, not a relational database.
- Multi-key transactions or atomic compare-and-swap across keys (single-key conditional writes are in scope as an extension).
- Authentication and authorisation — assumed to be enforced at an API gateway.
- Cross-datacenter / multi-region replication — mentioned as an extension but not fully designed here.
- Schema enforcement or type coercion on values — values are opaque bytes.

## Non-Functional Requirements

- **Scale:** 10 billion reads/day and 1 billion writes/day at steady state; support clusters of 1–1,000 nodes with linear horizontal scaling.
- **Latency:** GET p99 **< 10 ms**, PUT p99 **< 20 ms** end-to-end within a single datacenter.
- **Availability:** **99.99%** — roughly 52 minutes downtime per year. Any single node failure must be transparent to clients.
- **Consistency:** Tunable. Default is **eventual consistency** (AP mode per CAP); operators can configure stricter quorums per-request for read-your-writes or strong consistency guarantees.
- **Durability:** Zero data loss once a write is acknowledged — writes are persisted to the WAL and replicated to ≥ 2 nodes before returning success at the default quorum setting.
- **Replication:** Configurable replication factor *N* (default 3). The cluster tolerates up to *N − 1* simultaneous node failures without data loss.

## Terminology

| Term | Meaning |
|---|---|
| **Key** | Opaque byte string up to 1 KB identifying a record |
| **Value** | Arbitrary byte payload associated with a key |
| **Shard / partition** | A subset of keys assigned to a node |
| **Replication factor (N)** | Number of nodes that hold a copy of each key |
| **Write quorum (W)** | Minimum acknowledgements required before reporting a write as successful |
| **Read quorum (R)** | Minimum responses required before returning a read result |
| **Vnode** | Virtual node — a logical partition position on the consistent-hash ring |
| **SSTable** | Sorted String Table — immutable on-disk file produced by the LSM storage engine |
| **Memtable** | In-memory sorted buffer that absorbs writes before flushing to an SSTable |
| **Tombstone** | A special delete marker written to the log instead of physically removing a key |
