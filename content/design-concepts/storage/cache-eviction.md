---
title: Cache Eviction
weight: 14
type: docs
---

When a cache reaches its memory limit, it must evict an existing entry to make room for a new one. The eviction policy determines which entry to remove. The right policy depends on the access pattern: recency-based policies work well for most workloads; frequency-based policies handle skewed hot-key access better; adaptive policies automatically tune between the two.

{{< callout type="info" >}}
This file covers what gets evicted when the cache is full. For read/write patterns, invalidation strategies, stampede, and penetration see [Caching Patterns](../caching-patterns).
{{< /callout >}}

## LRU (Least Recently Used)

Evicts the entry that was accessed least recently. The assumption: if a key hasn't been read in a while, it is less likely to be read again soon.

### Implementation: Doubly-Linked List + Hash Map

The textbook LRU achieves O(1) get and O(1) put using two data structures together:

- **Hash map:** `key → node pointer` — O(1) lookup of any key
- **Doubly-linked list:** nodes ordered from most-recently-used (head) to least-recently-used (tail) — O(1) move-to-head, O(1) evict-from-tail

```
HashMap:  { "a": →node_a, "b": →node_b, "c": →node_c }

DLL:  HEAD ←→ [c | val_c] ←→ [a | val_a] ←→ [b | val_b] ←→ TAIL
              most recent                        least recent (eviction candidate)
```

**GET("a"):**
1. Hash map finds node_a in O(1)
2. Unlink node_a from its current position: O(1) (prev/next pointer updates)
3. Reinsert at head: O(1)
4. Return val_a

```
After GET("a"):
HEAD ←→ [a | val_a] ←→ [c | val_c] ←→ [b | val_b] ←→ TAIL
```

**PUT("d") when at capacity:**
1. Check if "d" already in hash map (update case)
2. If at capacity: evict tail node ("b") — O(1) unlink + O(1) hash map delete
3. Create new node_d, insert at head: O(1)
4. Insert `"d": →node_d` into hash map: O(1)

```
After PUT("d"):
HEAD ←→ [d | val_d] ←→ [a | val_a] ←→ [c | val_c] ←→ TAIL
"b" evicted
```

This O(1) get/put is why LRU is the default eviction policy in most cache implementations (Redis `allkeys-lru`, Memcached, CPU TLBs).

### LRU Weakness: Scan Resistance

A sequential scan over a large dataset floods the LRU list with entries that will never be accessed again. Each scanned entry evicts a hot, frequently-accessed entry.

```
Cache capacity = 100 entries. Hot working set = 80 entries.
Scan reads 200 sequential pages, none of which will be re-read.
→ All 100 cache entries replaced by scan results
→ Next 80 requests for the hot working set all miss
```

This is the **cache pollution** or **scan resistance** problem. For workloads that mix random hot-key access with large sequential scans, LRU degrades to zero hit rate.

**Mitigation:** LRU-K (track last K accesses, require K accesses before an entry is considered "frequently used") or segmented LRU (new entries enter a probationary segment; only entries that get a second access are promoted to the protected segment).

## LFU (Least Frequently Used)

Evicts the entry with the lowest access count. The assumption: if a key has been accessed many times historically, it is more likely to be accessed again.

### O(1) LFU Implementation

A naive LFU uses a min-heap: O(log n) per operation. The optimal O(1) LFU uses three data structures:

```
key_map:   { key → (value, freq) }            O(1) key lookup
freq_map:  { freq → LinkedHashSet of keys }   O(1) lookup of all keys at each freq level
min_freq:  integer                             current minimum frequency in the cache
```

**GET("a"):** Find in key_map → increment freq → move from `freq_map[old_freq]` to `freq_map[old_freq+1]` → update min_freq if old bucket is now empty.

**PUT("d") when at capacity:** Evict any key from `freq_map[min_freq]` (take first key from the set — all have equal frequency) → insert "d" into key_map with freq=1 → add to `freq_map[1]` → set min_freq=1.

### LFU Weakness: Aging

Frequency counts never decay. An entry that was popular last week accumulates a high count that protects it from eviction even if it is cold now. A new hot entry starts with freq=1 and gets evicted before the stale high-frequency entry.

```
"home_page" accessed 10,000 times last week, 0 times this week → freq=10,000
"trending_post" accessed 500 times today → freq=500
Cache fills up → "trending_post" evicted before "home_page"  ← wrong
```

**Mitigation:** frequency aging (periodically halve all counters), sliding window frequency counts, or W-TinyLFU (below) which uses a decaying frequency sketch.

## ARC (Adaptive Replacement Cache)

ARC dynamically adapts between LRU and LFU behavior based on real-time access patterns. It maintains four internal lists and automatically tunes how much cache to allocate to recency vs frequency.

### Structure

```
Total cache = T1 + T2  (sum of sizes stays fixed = cache capacity)

T1: "recently seen once"     — entries accessed only once recently
T2: "recently seen twice+"   — entries accessed multiple times (promoted from T1)

B1: ghost list for T1        — keys recently evicted from T1 (no data stored, just keys)
B2: ghost list for T2        — keys recently evicted from T2 (no data stored, just keys)

Parameter p: target size of T1 (starts at 0, adapts between 0 and cache_capacity)
```

### Adaptation Logic

**On a miss where evicted key is found in B1 (T1 ghost):**
"We evicted this from T1 too early — we should grow T1."
→ Increase p (allocate more cache to T1)

**On a miss where evicted key is found in B2 (T2 ghost):**
"We evicted this from T2 too early — we should grow T2."
→ Decrease p (allocate more cache to T2)

**On a clean miss (not in B1 or B2):**
New entry. Insert into T1 (first access). If cache is full, evict from whichever of T1/T2 is larger relative to its target p.

**On a hit in T1 or T2:**
Promote to head of T2 (now "seen multiple times").

The ghost lists are the key insight: they give ARC a memory of what was recently evicted, allowing it to learn from past eviction mistakes without storing the actual evicted data.

**Used by:** ZFS (FreeBSD, Linux ZFS), IBM DB2 buffer pool, some storage controllers.

**Strengths over LRU:** scan resistant (scanned entries stay in T1 and don't displace T2 entries), adapts to workload shift automatically.
**Weakness:** ARC was covered by IBM's US patent 6,996,676 (filed 2002, expired ~2022). Linux kernel page cache and Redis both chose alternatives during the patent's active period — CLOCK and approximate-LRU respectively — and have retained those implementations since.

## W-TinyLFU (Window TinyLFU)

W-TinyLFU is the eviction policy used by **Caffeine** (the default Java in-process cache, used by Spring, Guava successor) and the basis for Redis 4.0+ LFU mode. It combines a frequency sketch (Count-Min Sketch) with a segmented LRU to get LFU-like frequency awareness without the aging problem.

### Structure

```
Window cache  (1% of total capacity) — small LRU for new entries
Main cache    (99% of capacity):
  ├── Protected segment  (80% of main) — entries accessed ≥2 times; not evicted directly
  └── Probationary segment (20% of main) — newly admitted from Window; eviction candidates
```

### Admission Policy

When the Window LRU is full and must evict an entry (the "window victim"), W-TinyLFU compares its estimated frequency against the eviction candidate from the Probationary segment (the "main victim"):

```
if freq(window_victim) > freq(main_victim):
    admit window_victim → Probationary segment
    evict main_victim
else:
    evict window_victim (don't admit to main cache)
```

Frequency is estimated using a **Count-Min Sketch** — a probabilistic frequency counter using a small fixed-size matrix. It returns an upper-bound estimate of how many times a key has been accessed, using O(1) space proportional to a small constant (not proportional to the number of keys).

**Frequency aging:** The Count-Min Sketch halves all counters periodically (every N insertions) — this implements frequency decay. Old high-frequency items gradually lose their advantage over new hot items.

**Why W-TinyLFU beats pure LRU and LFU:**
- Scan resistance: one-time scans only enter the Window (1% of cache), then fail the admission test and are evicted. They never displace protected hot entries.
- No aging problem: Count-Min Sketch decays, so yesterday's popular entries don't block today's trending entries.
- Better hit rate than LRU: studies show 10–40% higher hit rates on real-world traces.

## CLOCK (Clock PRU)

CLOCK is a low-overhead approximation of LRU used in operating system page replacement (Linux uses a CLOCK variant called "second chance"). It avoids the doubly-linked list overhead by using a circular buffer and a single reference bit per entry.

```
Circular buffer of cache entries, each with a reference bit:

  [A:1] → [B:0] → [C:1] → [D:1] → [E:0]
             ↑
           clock hand

Eviction:
  1. Check entry at clock hand
  2. If reference bit = 1: clear bit to 0, advance hand (second chance)
  3. If reference bit = 0: evict this entry
```

**On access:** Set the entry's reference bit to 1.

**On eviction needed:** Advance the clock hand until finding an entry with bit=0 (entries with bit=1 get one "second chance" — their bit is cleared and the hand moves on).

**Why it approximates LRU:** Recently accessed entries have bit=1 and survive one pass of the clock hand. Entries not accessed since the last pass have bit=0 and are evicted — similar to being the least recently used.

**Advantage:** No pointer manipulation, no hash map updates on every access. Just set a bit. Used for OS page tables where O(1) and minimal overhead matters more than optimal eviction decisions.

## FIFO and Random Eviction

**FIFO (First-In, First-Out):** Evicts the oldest entry by insertion time, regardless of access frequency or recency. Simpler than LRU — just a queue.

**When FIFO is appropriate:**
- CDN edge caches for content with uniform freshness requirements (all assets expire at the same time; access frequency is irrelevant to eviction)
- Pre-loaded caches where every entry has the same expected useful life
- When predictability of eviction order matters (e.g., testing, deterministic replay)

**Random eviction:** Evict a randomly selected entry. Surprisingly effective in practice — studies show random eviction achieves within 10–15% of LRU hit rate for many real-world workloads, with significantly lower implementation overhead.

**When random is appropriate:**
- Very high throughput caches where even O(1) pointer updates are too expensive
- Hardware TLBs in some CPU architectures
- Benchmarking baselines

## Comparison

| Policy | Eviction criterion | Implementation complexity | Scan resistant | Aging handled | Used in |
|--------|------------------|--------------------------|---------------|--------------|--------|
| **LRU** | Least recently used | DLL + hashmap, O(1) | ❌ | N/A | Redis, Memcached, CPU caches |
| **LFU** | Least frequently used | 3 hashmaps, O(1) | ✅ | ❌ (naive) | InfluxDB, some CDNs |
| **ARC** | Adaptive LRU + LFU | 4 lists + parameter p | ✅ | Partial | ZFS, IBM DB2 |
| **W-TinyLFU** | Frequency sketch + segmented LRU | Count-Min Sketch + 3 segments | ✅ | ✅ (decay) | Caffeine, Redis LFU mode |
| **CLOCK** | LRU approximation via reference bit | Circular buffer, O(1) | Partial | N/A | Linux page cache, OS kernels |
| **FIFO** | Insertion order | Queue, O(1) | N/A | N/A | CDN assets, simple caches |
| **Random** | Random selection | Array + random, O(1) | N/A | N/A | Hardware TLBs, fallbacks |

## Choosing an Eviction Policy

| Workload | Recommended policy | Reason |
|----------|------------------|--------|
| General web cache (hot keys, long tail) | **LRU** or **W-TinyLFU** | LRU is simple and effective; W-TinyLFU gives 10–40% better hit rate |
| Skewed access (20% of keys get 80% of traffic) | **LFU** or **W-TinyLFU** | Frequency matters more than recency; W-TinyLFU handles aging |
| Mixed hot access + sequential scans | **ARC** or **W-TinyLFU** | Scan resistance protects hot entries from scan pollution |
| Uniform content freshness (CDN assets) | **FIFO** | Access frequency is irrelevant; age is the eviction criterion |
| OS kernel page replacement | **CLOCK** | Minimal overhead, good approximation of LRU |
| Session store (all sessions equally important) | **volatile-ttl** | Evict soonest-to-expire sessions first; access pattern irrelevant |
| Mixed persistent + ephemeral keys (Redis) | **volatile-lru** or **volatile-lfu** | Evict only keys with TTL; preserve no-TTL keys |
