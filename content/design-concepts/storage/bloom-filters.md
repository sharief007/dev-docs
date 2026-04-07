---
title: Bloom Filters & HyperLogLog
weight: 12
type: docs
---

Bloom filters and HyperLogLog are probabilistic data structures that trade exact answers for dramatic space savings. Both give you a bounded error guarantee in exchange for near-constant memory — making problems that would otherwise require gigabytes solvable in kilobytes.

## Bloom Filter

A Bloom filter answers one question: **"Is this element in the set?"** It answers with either "definitely not" or "probably yes." It never produces false negatives — if an element was inserted, the filter will always say "probably yes." It can produce false positives — it may say "probably yes" for elements that were never inserted.

### Mechanics

A Bloom filter is a bit array of size `m`, initialized to all zeros, and `k` independent hash functions.

**Insert element `x`:**
1. Compute `h1(x), h2(x), ..., hk(x)` — k positions in the bit array
2. Set all k bits to 1

**Query element `x`:**
1. Compute the same k positions
2. If **any bit is 0** → `x` is **definitely not** in the set (impossible to have been inserted without setting all k bits)
3. If **all bits are 1** → `x` is **probably** in the set (but other insertions may have set those bits)

```
m=10 bits, k=3 hash functions

Insert "alice" → H1=1, H2=4, H3=7
  [0, 1, 0, 0, 1, 0, 0, 1, 0, 0]

Insert "bob" → H1=2, H2=4, H3=6
  [0, 1, 1, 0, 1, 0, 1, 1, 0, 0]

Query "alice" → H1=1, H2=4, H3=7 → bits [1,1,1] → probably in set ✓
Query "carol" → H1=0, H2=4, H3=7 → bits [0,1,1] → bit 0 is 0 → definitely NOT in set ✓
Query "dave"  → H1=2, H2=6, H3=7 → bits [1,1,1] → probably in set ← FALSE POSITIVE
                (dave was never inserted; bob and alice happened to set these bits)
```

### Why False Negatives Are Impossible

Insertion always sets k specific bits to 1. Once set, bits are never cleared (in a standard Bloom filter). If an element was inserted, its k bits are all 1 by definition. A query for that element checks the same k positions and will always find all 1s. There is no mechanism by which a bit set during insertion can become 0 later.

### Why False Positives Occur

Hash collisions. Two different elements may share one or more of their k bit positions with each other or with a combination of previously inserted elements. If, by chance, all k positions for a queried element were set by different prior insertions, the filter wrongly concludes the element is present.

### Tuning: Math for Sizing

Three variables: `m` (bit array size), `n` (number of elements inserted), `k` (number of hash functions), `p` (false positive probability).

**Optimal number of hash functions** — minimizes false positive rate for given m and n:
```
k = (m/n) × ln(2) ≈ 0.693 × (m/n)
```

**False positive probability** — given m, n, and optimal k:
```
p ≈ (1 - e^(-kn/m))^k ≈ (0.5)^k  when k is optimal
```

**Required bit array size** — given n elements and desired false positive rate p:
```
m = -n × ln(p) / (ln 2)²
```

**Worked example:** Index 1 million URLs with a 1% false positive rate.
```
n = 1,000,000
p = 0.01 (1%)

m = -(1,000,000 × ln(0.01)) / (ln 2)²
  = -(1,000,000 × (-4.605)) / 0.480
  = 9,585,058 bits ≈ 1.14 MB

k = 0.693 × (9,585,058 / 1,000,000) ≈ 6.6 → use k = 7
```

1.14 MB to track 1 million URLs with 1% false positives. A hash set storing the same URLs (32 bytes each) would require ~32 MB — 28× larger.

| False positive rate | Bits per element (m/n) | Hash functions (k) |
|--------------------|----------------------|-------------------|
| 1% | 9.6 | 7 |
| 0.1% | 14.4 | 10 |
| 0.01% | 19.2 | 14 |

**Rule of thumb:** roughly 10 bits per element achieves ~1% false positive rate.

### Real-World Use Cases

**Cassandra SSTable negative lookups (the most important FAANG example)**

This is the primary use of Bloom filters in [LSM trees](../lsm-trees). Each SSTable in Cassandra (and RocksDB) has an associated Bloom filter. When a read arrives for a key:
1. Check the Bloom filter for this SSTable
2. "Definitely not here" → skip the SSTable entirely (no disk I/O)
3. "Probably here" → do the actual disk read

Without Bloom filters, a key that does not exist in any SSTable would require reading every SSTable on disk — catastrophic for a read path with 5–10 SSTables per level. With Bloom filters, non-existent key lookups are O(k × num_levels) bit checks, not disk reads.

**Cache penetration prevention**

A cache miss for a key that does not exist in the database is a wasted DB query. Attackers intentionally probe non-existent keys to bypass the cache and hammer the DB directly (cache penetration attack).

```
Request for key K:
  1. Check Bloom filter for K
  2. "Definitely not in DB" → return 404 immediately, no DB hit
  3. "Probably in DB" → check cache → on miss, query DB
```

The Bloom filter must be loaded with all keys that exist in the DB. On DB writes, update the filter. False positives cause unnecessary DB lookups (acceptable); false negatives are impossible (a key that exists is always found).

**Web crawler URL deduplication**

Googlebot has crawled ~50 billion URLs. Checking whether a URL has been visited before cannot use a hash set (50 billion × 32 bytes = 1.6 TB). A Bloom filter with 10 bits/URL = 62.5 GB — still large but ~25× smaller, and distributed across the crawler cluster.

**Chrome Safe Browsing**

Chrome ships a Bloom filter of known-malicious URLs. On each navigation:
1. Check the local Bloom filter — "definitely not malicious" → navigate without network call
2. "Probably malicious" → query Google's API to confirm

This eliminates ~99.9% of network calls while still catching all malicious URLs.

**Database join optimization**

Some query optimizers build a Bloom filter on the smaller side of a join, then use it to filter the larger side before the join executes — eliminating rows that cannot match before the expensive join operation.

### Deletion: The Standard Filter's Limitation

Standard Bloom filters cannot support deletion. Unsetting a bit that was set during insertion could invalidate other elements that share that bit position.

```
Insert "alice" → sets bits {1, 4, 7}
Insert "bob"   → sets bits {2, 4, 6}   ← bit 4 shared with alice
Delete "alice" → would unset bits {1, 4, 7}
Query "bob"    → checks bits {2, 4, 6} → bit 4 is now 0 → FALSE NEGATIVE ❌
```

## Counting Bloom Filter

Replace each bit in the array with a small integer counter (typically 4 bits). Insertions increment counters; deletions decrement them. A membership check verifies all k counters are > 0.

```
Insert "alice" → counters at {1, 4, 7} become [+1, +1, +1]
Insert "bob"   → counters at {2, 4, 6} become [+1, +2, +1]  ← counter 4 = 2
Delete "alice" → counters at {1, 4, 7} become [0, +1, 0]
Query "bob"    → checks {2, 4, 6} → [1, 1, 1] all > 0 → probably present ✓
```

**Cost:** 4× memory (4-bit counter vs 1-bit). Counter overflow (a position incremented more than 2^4 = 16 times) causes incorrect results — use larger counters for high-frequency elements.

**When to use:** cache eviction tracking, network packet deduplication, any use case requiring both insertion and deletion.

## Cuckoo Filter

A more modern alternative that supports deletion with lower false positive rates than Counting Bloom Filters and comparable space to standard Bloom filters.

Instead of a bit array, a Cuckoo filter stores **fingerprints** (short hashes of elements) in a hash table using cuckoo hashing. Deletion removes the fingerprint.

| | Bloom Filter | Counting BF | Cuckoo Filter |
|---|---|---|---|
| **Space** | 1× | ~4× | ~1.05× |
| **False positive rate** | Configurable | Configurable | Lower for same space |
| **Deletion** | ❌ | ✅ | ✅ |
| **Lookup** | O(k) | O(k) | O(1) |
| **Insertion worst case** | O(k) | O(k) | O(n) (cuckoo cycle — rare) |

Use Cuckoo filters when you need deletion and want better space efficiency than Counting Bloom Filters.

---

## HyperLogLog

HyperLogLog (HLL) solves a different problem: **how many unique elements are in a set?** (cardinality estimation). It does not answer membership queries — it counts distinct values with a bounded relative error.

### The Problem It Solves

Counting unique visitors to a website: store each user ID seen today, count at end of day. With 100 million unique visitors, a hash set requires ~800 MB. HyperLogLog counts 100 million unique elements with ~0.81% error using **~1.5 KB** — a 500,000× memory reduction.

### How It Works

The key insight: in a stream of uniformly hashed values, the maximum number of leading zero bits observed is a statistical estimator of the stream's cardinality.

```
hash("user_1") = 0b 0001 1010...  ← 3 leading zeros
hash("user_2") = 0b 1010 0011...  ← 0 leading zeros
hash("user_3") = 0b 0000 0110...  ← 4 leading zeros
hash("user_4") = 0b 0010 1100...  ← 2 leading zeros

Max leading zeros observed = 4
Estimated cardinality ≈ 2^4 = 16  (rough estimate — real HLL corrects this)
```

A single maximum-zeros estimator has high variance. HLL reduces variance by:
1. Using the first `b` bits of each hash to route the element to one of `2^b` registers
2. Each register tracks the max leading zeros seen in its sub-stream
3. Use harmonic mean of 2^(max_zeros) across all registers as the estimate

With `b = 14` → 16,384 registers × 6 bits each = ~12 KB → **0.81% standard error** at any cardinality.

### Properties

| Property | Value |
|----------|-------|
| Memory | ~1.5 KB (standard) to 12 KB (high precision) |
| Error rate | 0.81% with 16,384 registers |
| Merge | Two HLL sketches can be merged (union) without accessing original data |
| Cardinality range | Works for any cardinality from 0 to 2^64 |
| Updates | O(1) per element |

### Redis Implementation

```
PFADD  dau:2024-01-14 user_42 user_99 user_7   # add users to daily HLL
PFCOUNT dau:2024-01-14                           # → estimated unique count
PFMERGE dau:2024-01-week users:day1 users:day2 ... users:day7  # weekly unique users
```

`PFCOUNT` on a key with 1 billion unique users returns in microseconds and uses ~12 KB of memory regardless of cardinality.

### Use Cases

| Use case | Why HLL |
|----------|---------|
| Daily/Monthly Active Users | Exact counts need a set per user — HLL gives 0.81% error in 12 KB |
| Unique search queries per day | Billions of queries; exact uniqueness not required |
| Distinct IPs per URL (CDN analytics) | Per-URL hash sets at web scale would be terabytes |
| Database query planners | PostgreSQL uses HLL internally to estimate `COUNT(DISTINCT col)` for query optimization |
| Network flow cardinality | Count unique src/dst IP pairs in a router without storing each pair |

### HyperLogLog vs Bloom Filter

| | Bloom Filter | HyperLogLog |
|---|---|---|
| **Question answered** | "Is X in the set?" | "How many unique elements?" |
| **Error type** | False positives (membership) | Relative error on count |
| **Supports deletion** | No (standard) | No |
| **Merge two sketches** | Yes (bitwise OR) | Yes (register max) |
| **Memory** | ~10 bits/element | ~12 KB fixed, any cardinality |
