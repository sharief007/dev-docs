---
title: API Pagination
weight: 1
type: docs
---

An API endpoint returns 2 million records. The client doesn't need them all at once — it needs page 1 of 20 results. Pagination splits a large result set into manageable chunks, each retrieved with a separate request. The method you choose determines performance at scale, stability under concurrent writes, and client implementation complexity.

## Offset Pagination

The simplest approach: `LIMIT n OFFSET m`. The client specifies which page to retrieve by calculating the offset.

```sql
-- Page 1 (items 1-20)
SELECT * FROM orders ORDER BY created_at DESC LIMIT 20 OFFSET 0;

-- Page 2 (items 21-40)
SELECT * FROM orders ORDER BY created_at DESC LIMIT 20 OFFSET 20;

-- Page 50 (items 981-1000)
SELECT * FROM orders ORDER BY created_at DESC LIMIT 20 OFFSET 980;
```

**API request:**

```
GET /orders?page=2&page_size=20

Response:
{
  "data": [...],
  "page": 2,
  "page_size": 20,
  "total_count": 15420
}
```

### The Deep Offset Problem

To return page 50, the database must **scan and discard** 980 rows before returning 20. For `OFFSET 1000000`, the database scans a million rows and throws them away.

```
EXPLAIN ANALYZE SELECT * FROM orders ORDER BY created_at DESC
LIMIT 20 OFFSET 1000000;

→ Seq Scan on orders  (actual rows=1000020, filtered=1000000)
   Execution Time: 3240ms
```

| Offset | DB Work | Typical Latency |
|--------|---------|----------------|
| 0 | Scan 20 rows | ~2ms |
| 1,000 | Scan 1,020 rows | ~10ms |
| 100,000 | Scan 100,020 rows | ~200ms |
| 1,000,000 | Scan 1,000,020 rows | ~3s |

**Also broken by concurrent writes:** if a new order is inserted between fetching page 1 and page 2, all items shift — page 2 may repeat an item from page 1 or skip one entirely.

## Cursor-Based Pagination

Encode the **last seen item** as an opaque cursor. The client doesn't know what's inside the cursor — it just passes it back on the next request.

```
GET /orders?limit=20

Response:
{
  "data": [...20 items...],
  "next_cursor": "eyJpZCI6MTAwMCwiY3JlYXRlZF9hdCI6IjIwMjUtMDEtMTV9",
  "has_more": true
}

GET /orders?limit=20&cursor=eyJpZCI6MTAwMCwiY3JlYXRlZF9hdCI6IjIwMjUtMDEtMTV9

Response:
{
  "data": [...next 20 items...],
  "next_cursor": "eyJpZCI6OTgwLCJjcmVhdGVkX2F0IjoiMjAyNS0wMS0xNH0",
  "has_more": true
}
```

The cursor is typically a Base64-encoded JSON object containing the sort key values of the last item:

```python
import base64, json

def encode_cursor(last_item):
    payload = {"id": last_item["id"], "created_at": str(last_item["created_at"])}
    return base64.urlsafe_b64encode(json.dumps(payload).encode()).decode()

def decode_cursor(cursor):
    return json.loads(base64.urlsafe_b64decode(cursor.encode()).decode())
```

**Why opaque?** The client should not construct or modify cursors. If the sort order or cursor encoding changes server-side, old cursors simply return an error — no silent data corruption.

## Keyset Pagination

The underlying mechanism behind cursor-based pagination. Instead of `OFFSET`, use a `WHERE` clause on the sort key to skip directly to the right position.

```sql
-- Page 1
SELECT * FROM orders
ORDER BY created_at DESC, id DESC
LIMIT 20;

-- Page 2 (cursor: created_at='2025-01-15', id=1000)
SELECT * FROM orders
WHERE (created_at, id) < ('2025-01-15', 1000)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

**Performance:** the database uses the index to jump directly to the starting point — no scanning and discarding. Page 50 is as fast as page 1.

```
EXPLAIN ANALYZE SELECT * FROM orders
WHERE (created_at, id) < ('2025-01-15', 1000)
ORDER BY created_at DESC, id DESC LIMIT 20;

→ Index Scan using idx_orders_created_at_id on orders
   Execution Time: 2ms  (same speed regardless of position)
```

### Requirement: Sorted Unique Key

Keyset pagination requires the sort columns to produce a **unique ordering**. If `created_at` has duplicates (multiple orders at the same timestamp), the tie-breaker must be a unique column (like `id`).

```sql
-- Wrong: non-unique sort key → items with same created_at may be skipped
WHERE created_at < '2025-01-15' ORDER BY created_at DESC LIMIT 20;

-- Right: add unique tie-breaker
WHERE (created_at, id) < ('2025-01-15', 1000)
ORDER BY created_at DESC, id DESC LIMIT 20;
```

## Comparison

| Property | Offset | Cursor / Keyset |
|----------|--------|----------------|
| Performance at deep pages | Degrades linearly with offset | Constant — always index scan |
| Random page access | Yes (`?page=50`) | No — must traverse sequentially |
| Stability under writes | Items can repeat or be skipped | Stable — position anchored to sort key |
| Implementation complexity | Trivial | Moderate (cursor encoding, composite WHERE) |
| Total count available | Yes (but `COUNT(*)` is expensive on large tables) | Not naturally — requires separate count query |
| Best for | Admin panels, small datasets, page-number UIs | Infinite scroll, mobile apps, large datasets |

## Page Size Trade-offs

| Page Size | Pros | Cons |
|-----------|------|------|
| Small (10–20) | Low latency, small payload, responsive UI | More round trips, higher overhead per request |
| Medium (50–100) | Balanced; typical default for APIs | — |
| Large (500–1000) | Fewer requests, good for batch processing | Higher latency, larger payload, memory pressure on client |

**Practical defaults:** Stripe uses 10 (max 100). GitHub uses 30 (max 100). Twitter uses 20. It's sensible to let the client choose within a capped range via `?page_size=50` with a server-enforced maximum.

## Consistency During Pagination

Even with cursor pagination, concurrent writes can cause issues:

```
Page 1 request: returns items with id > 1000 (items 1001-1020)
Between requests: item 1015 is deleted
Page 2 request: returns items with id > 1020 (items 1021-1040)

Result: item 1015 disappeared — client never sees it
```

**Mitigation strategies:**
- **Acceptable for most use cases:** pagination is a best-effort view, not a snapshot
- **Snapshot isolation:** use a database transaction with `REPEATABLE READ` for the entire pagination session (impractical for stateless APIs)
- **Versioned cursors:** encode a timestamp in the cursor; query against a read replica that's at least as fresh as that timestamp

{{< callout type="info" >}}
**Interview tip:** When an API needs to return a large list, say: "I'd use cursor-based pagination with keyset queries — the cursor encodes the last item's sort key and is opaque to the client. Unlike offset pagination, performance doesn't degrade for deeper pages because the database uses an index seek instead of scanning. The trade-off is no random page access, which is fine for infinite scroll UIs but not for page-number navigation." This shows you understand the database-level reason for the choice, not just the API pattern.
{{< /callout >}}

## Test Your Understanding

{{< details title="A client uses offset pagination: GET /orders?offset=50000&limit=20. The query is slow even with an index. Why?" closed="true" >}}
**Offset scanning.** `OFFSET 50000` means the database finds the first 50,020 rows using the index, then **discards the first 50,000** and returns 20. The deeper the offset, the more work is wasted. At offset 1M, the DB scans 1M+ index entries just to throw them away.

Cursor/keyset pagination avoids this: `WHERE id > last_seen_id ORDER BY id LIMIT 20` uses an index **seek** to jump directly to the starting point — O(log n) regardless of page depth.
{{< /details >}}

{{< details title="You use cursor-based pagination. A client bookmarks page 5 to return later. Between visits, items are inserted and deleted. The bookmark is broken. Why, and is this fixable?" closed="true" >}}
**Cursors encode position, not page number.** The cursor says "start after item X." If items are inserted before X, the client sees them on a previous page (if they go back) but not on the current cursor's continuation — items are "missed." If X is deleted, the cursor is invalid.

**This is inherent to cursor pagination** — it provides a stable traversal of a moving dataset, not a stable snapshot. Offset pagination has the same problem (inserted items shift offsets).

**If you need stable snapshots:** Use timestamp-based cursors with immutable data (events, audit logs). Or take a point-in-time snapshot and paginate over it (expensive, only for exports).
{{< /details >}}

{{< details title="Your API uses keyset pagination: WHERE created_at > cursor ORDER BY created_at LIMIT 20. Two orders have the same created_at. One is skipped. Why?" closed="true" >}}
**Non-unique sort key.** If two orders share `created_at = '2024-01-15T10:00:00'`, the cursor after the first page points to that timestamp. The next page's `WHERE created_at > cursor` skips the second order with the same timestamp.

**Fix:** Use a **compound cursor** with a tiebreaker: `WHERE (created_at, id) > (cursor_ts, cursor_id) ORDER BY created_at, id`. The `id` is unique and breaks ties. This requires a composite index on `(created_at, id)`.
{{< /details >}}