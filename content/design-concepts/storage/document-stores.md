---
title: Document Stores (MongoDB)
weight: 8
type: docs
---

MongoDB stores data as BSON documents in schemaless collections. The central design decision is not which queries to support — it is whether to **embed** related data inside a document or **reference** it from a separate collection. Getting that decision right eliminates most of the pain that comes from using MongoDB at scale.

## Document Model

A document is a BSON object — a superset of JSON with typed fields.

```json
{
  "_id": ObjectId("65a3f2c1e4b09d3f1a2b3c4d"),   // 12-byte: timestamp + machine + pid + counter
  "user_id": 42,
  "email": "alice@example.com",
  "created_at": ISODate("2024-01-14T10:00:00Z"),  // BSON Date, not a string
  "address": {                                       // embedded sub-document
    "street": "123 Main St",
    "city": "Seattle",
    "zip": "98101"
  },
  "tags": ["premium", "beta"],                      // array
  "balance": NumberDecimal("9999.99")               // Decimal128 — exact for money
}
```

**BSON types that matter:** `ObjectId` (auto-generated unique ID with embedded timestamp), `Date` (millisecond precision, not string), `NumberDecimal` (exact decimal for financial values — use this, not `double`), `BinData` (binary blobs).

### Embedding vs Referencing

This is the core schema decision. There is no JOIN in MongoDB — related data must either be embedded in the same document or fetched with a separate query.

| Scenario | Embed | Reference |
|----------|-------|-----------|
| Data always fetched together | ✅ | ❌ (extra round trip) |
| Sub-document has bounded growth | ✅ | — |
| Sub-document grows unboundedly | ❌ (16 MB document limit) | ✅ |
| Data shared by many documents | ❌ (duplicated across all) | ✅ (one copy, referenced) |
| Sub-document queried independently | ❌ (must load parent) | ✅ |
| Write contention on sub-document | ❌ (all writers contend on parent) | ✅ |

**Example — embed comments in a post:**
```json
// Good if comments are bounded (e.g., max 100 per post)
{ "_id": 1, "title": "Post", "comments": [ {"author": "bob", "text": "..."} ] }

// Bad if a post can have 100,000 comments — document hits 16 MB limit
// Fix: separate "comments" collection, reference post_id
```

**The embedding trap:** When you embed an array expecting it to stay small and it grows without bound, the document eventually exceeds MongoDB's 16 MB limit — at which point you must migrate to a reference model under load.

## Indexes

MongoDB builds indexes on a `mongod` collection level. All index types are B-trees except geospatial (2d/2dsphere use geohash).

| Index type | Declaration | Behavior |
|-----------|-------------|---------|
| **Single-field** | `{field: 1}` | Sort ascending; `-1` for descending |
| **Compound** | `{a: 1, b: -1}` | Leftmost prefix rule identical to SQL compound indexes |
| **Multikey** | `{tags: 1}` | Automatically created when field is an array; one index entry per array element |
| **Text** | `{body: "text"}` | Tokenized full-text; supports `$text` queries with relevance scoring |
| **2dsphere** | `{loc: "2dsphere"}` | GeoJSON points/polygons; `$near`, `$geoWithin`, `$geoIntersects` |
| **Hashed** | `{user_id: "hashed"}` | Used for hash-based sharding; equality only, no range |
| **Wildcard** | `{"$**": 1}` | Index all fields or a subtree; useful for dynamic/user-defined schemas |
| **Partial** | `{status: 1}` with `partialFilterExpression` | Index only documents matching a filter — smaller, faster |
| **TTL** | `{created_at: 1}` with `expireAfterSeconds` | Automatically delete documents after TTL — useful for sessions, logs |

**Multikey index gotcha:** A compound index cannot have more than one multikey field. `{tags: 1, categories: 1}` fails if both are arrays. Also, a multikey index on an array field cannot be used as a shard key.

**Covered query:** If all projected fields are in the index, MongoDB returns results directly from the index without touching the collection — identical to PostgreSQL's index-only scan.

```javascript
// Index: { user_id: 1, status: 1, created_at: 1 }
db.orders.find(
  { user_id: 42, status: "pending" },
  { _id: 0, created_at: 1 }           // project only indexed field
)
// → "indexOnly: true" in explain() — no document fetch
```

## Aggregation Pipeline

The aggregation pipeline transforms documents through a sequence of stages. Each stage receives documents from the previous stage and passes its output to the next.

```javascript
db.orders.aggregate([
  { $match:   { status: "completed", created_at: { $gte: ISODate("2024-01-01") } } },
  { $lookup:  { from: "customers", localField: "customer_id",
                foreignField: "_id", as: "customer" } },
  { $unwind:  "$customer" },
  { $group:   { _id: "$customer.region",
                total_revenue: { $sum: "$amount" },
                order_count:   { $sum: 1 } } },
  { $sort:    { total_revenue: -1 } },
  { $limit:   10 },
  { $project: { _id: 0, region: "$_id", total_revenue: 1, order_count: 1 } }
])
```

**Key stages:**

| Stage | Purpose |
|-------|---------|
| `$match` | Filter documents — place early to reduce pipeline input; uses indexes if first stage |
| `$group` | Aggregate: `$sum`, `$avg`, `$min`, `$max`, `$push` (array), `$addToSet` (distinct array) |
| `$lookup` | Left outer join to another collection — expensive without an index on the foreign field |
| `$unwind` | Deconstruct an array field into one document per element |
| `$project` | Reshape documents: include/exclude/rename/compute fields |
| `$addFields` | Add computed fields without removing existing ones |
| `$sort` + `$limit` | Top-N pattern — MongoDB optimizes `$sort` + `$limit` as a single top-N heap |
| `$facet` | Run multiple sub-pipelines on the same input — for faceted search results |
| `$bucket` | Group documents into ranges (histogram) |
| `$out` / `$merge` | Write pipeline results to a collection — for pre-computed aggregations |

**`$lookup` performance trap:** `$lookup` performs an in-memory join. Without an index on the `foreignField`, it scans the entire foreign collection for every input document — an O(n × m) operation. Always create an index on the foreign field before using `$lookup` in production.

**Memory limit:** Aggregation pipelines have a 100 MB memory limit per stage. Add `{ allowDiskUse: true }` for large aggregations, or use `$out`/`$merge` to materialize intermediate results.

## Replication: Replica Set

A replica set is a group of `mongod` instances that maintain the same data. One node is the primary (accepts writes); the others are secondaries (replicate from primary).

```
          ┌──────────────┐
  Writes ─►    Primary    │◄── Heartbeat (every 2s)
          └──────┬───────┘
           oplog │ async replication
        ┌────────┴────────┐
        ▼                 ▼
  ┌──────────┐      ┌──────────┐
  │Secondary │      │Secondary │  (or Arbiter — votes but holds no data)
  └──────────┘      └──────────┘
```

**Oplog (operations log):** A capped collection on the primary that records every write as an idempotent operation. Secondaries tail the oplog and replay operations to stay in sync. The oplog window (how far back a secondary can lag before needing full resync) is determined by oplog size — typically configured to hold 24–72 hours of writes.

**Automatic failover:**
1. Primary becomes unreachable (heartbeat misses)
2. Secondaries detect failure after `electionTimeoutMillis` (default 10s)
3. An eligible secondary calls an election; majority vote required to become primary
4. Clients with replica set connection strings auto-discover the new primary

**Read preferences:**

| Preference | Reads from | Use case |
|-----------|-----------|---------|
| `primary` (default) | Primary only | Strong consistency — always latest data |
| `primaryPreferred` | Primary; secondary if primary unavailable | Slight availability improvement |
| `secondary` | Any secondary | Offload reads; accepts replication lag |
| `secondaryPreferred` | Secondary; primary if no secondary available | Maximize read scale |
| `nearest` | Lowest latency node | Geo-distributed reads; may read stale data |

**Write concern:**

```javascript
db.orders.insertOne(doc, { writeConcern: { w: "majority", j: true, wtimeout: 5000 } })
// w: "majority" — acknowledged by majority of voting nodes
// j: true — journaled (fsynced) before acknowledgement
// wtimeout — timeout in ms; returns error if not satisfied in time
```

| Write concern | Durability | Latency |
|--------------|-----------|---------|
| `w: 0` | None — fire and forget | Minimal |
| `w: 1` | Primary ack only — lost if primary fails before replicating | Low |
| `w: "majority"` | Survives primary failure — majority has the write | Medium |
| `j: true` | Fsynced to journal — survives process crash | Adds fsync latency |

**Change streams:** Applications can subscribe to a real-time stream of changes (inserts, updates, deletes) on a collection, database, or cluster. Built on top of the oplog; provides a resumable cursor using a resume token — if the consumer disconnects, it can resume from where it left off.

```javascript
const stream = db.orders.watch([{ $match: { "fullDocument.status": "shipped" } }]);
stream.on("change", change => console.log(change.fullDocument));
```

## Sharding

Sharding horizontally partitions a collection across multiple shard servers. Each shard is itself a replica set.

```
          mongos (query router)
         /        |         \
    Shard 1    Shard 2    Shard 3     ← each shard is a replica set
  (RS: 3 nodes) (RS: 3)   (RS: 3)
         \        |         /
             Config Servers
          (RS: stores metadata,
           chunk→shard mapping)
```

### Shard Key Selection

The shard key determines how documents are distributed. It is **immutable after collection creation** — a bad shard key requires re-sharding the entire collection.

**Three properties to evaluate:**

| Property | Goal | Bad example | Good example |
|----------|------|-------------|-------------|
| **Cardinality** | High — enough distinct values to fill all chunks | `status` (3 values → max 3 shards) | `user_id` (millions of distinct values) |
| **Frequency** | Even — no single value dominates | `country_code` (50% US traffic → US chunk is hot) | `user_id` (roughly even distribution) |
| **Monotonic rate of change** | Avoid — always-increasing keys route all writes to one chunk | `created_at`, `ObjectId` | `hashed(user_id)`, `(user_id, created_at)` |

**Range sharding:** Documents with adjacent shard key values are co-located in the same chunk. Supports efficient range queries on the shard key. Risk: monotonically increasing keys (ObjectId, timestamp) create insert hotspots on the last chunk.

**Hashed sharding:** MongoDB hashes the shard key value before distributing. Writes are evenly spread across shards. Range queries on the shard key require scatter-gather (fan out to all shards).

```javascript
// Range sharding — supports range queries, risk of hotspot on monotonic key
sh.shardCollection("db.orders", { customer_id: 1 })

// Hashed sharding — even distribution, no range query support on shard key
sh.shardCollection("db.orders", { customer_id: "hashed" })

// Compound shard key — common pattern: entity ID + time bucket
sh.shardCollection("db.events", { user_id: 1, day: 1 })
```

### Chunk Migration

MongoDB divides each shard's key range into **chunks** (default 128 MB). The balancer automatically migrates chunks between shards to maintain even distribution.

**Migration cost:** Chunk migration moves data between shards over the network while still serving live traffic. Heavy migrations during peak traffic impact query latency. Schedule balancer windows during off-peak hours:

```javascript
sh.setBalancerState(false)              // disable balancer
sh.startBalancer()                      // re-enable
db.settings.updateOne(
  { _id: "balancer" },
  { $set: { activeWindow: { start: "01:00", stop: "05:00" } } }
)
```

### Scatter-Gather

If a query does not include the shard key, `mongos` fans it out to all shards and merges results. For a 10-shard cluster, one query becomes 10 queries. Scatter-gather is acceptable for infrequent admin queries but is a performance anti-pattern on hot paths.

Always include the shard key in query predicates on high-traffic paths.

## Transactions

**Single-document operations are always atomic** — an update to a document with embedded sub-documents and arrays is atomic without any transaction overhead. This is why embedding related data (when appropriate) is preferable to referencing.

**Multi-document transactions (MongoDB 4.0+):**

```javascript
const session = client.startSession();
session.startTransaction({ readConcern: { level: "snapshot" }, writeConcern: { w: "majority" } });
try {
  db.accounts.updateOne({ _id: A }, { $inc: { balance: -100 } }, { session });
  db.accounts.updateOne({ _id: B }, { $inc: { balance: +100 } }, { session });
  await session.commitTransaction();
} catch (e) {
  await session.abortTransaction();
}
```

Multi-document transactions use the same MVCC snapshot isolation as the replica set oplog. They add coordination overhead — prefer single-document atomicity via embedding where possible. Transactions across shards (distributed transactions) add further latency due to two-phase commit coordination across shard replica sets.
