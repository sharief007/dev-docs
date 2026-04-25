---
title: Capacity Estimation
weight: 1
type: docs
---

System design interviews start with requirements, and requirements include **scale**. "How many users?" is not trivia — it determines whether you need one PostgreSQL instance or a sharded cluster, whether a single Redis node handles the load or you need a distributed cache, and whether your storage fits on one disk or requires an object store. Capacity estimation turns vague requirements into concrete numbers: requests per second, storage in terabytes, bandwidth in Gbps.

## Latency Numbers Every Engineer Should Know

These are order-of-magnitude numbers from 2024. The exact values change with hardware generations, but the relative magnitudes remain stable.

| Operation | Latency | Notes |
|-----------|---------|-------|
| L1 cache reference | ~1 ns | CPU-local, fastest possible |
| L2 cache reference | ~4 ns | Still on-chip |
| L3 cache reference | ~10 ns | Shared across cores |
| RAM access | ~100 ns | Main memory; 10× slower than L2 |
| SSD random read | ~100 µs | 1,000× slower than RAM |
| SSD sequential read (1 MB) | ~1 ms | SSDs shine at sequential I/O |
| HDD random seek | ~10 ms | 100× slower than SSD random |
| Network round trip (same datacenter) | ~0.5 ms | Varies with distance and congestion |
| Network round trip (cross-region) | ~50–150 ms | US-East ↔ EU-West ≈ 80ms |
| Disk seek + read 1 MB (HDD) | ~20 ms | Dominated by seek time |

**Key insight:** RAM is 1,000× faster than SSD which is 100× faster than HDD. Network latency within a datacenter is comparable to SSD latency. Cross-region network latency is comparable to HDD seek. These ratios drive every caching and replication decision.

## The Estimation Framework

Every capacity estimate follows the same structure:

```
1. Users            → DAU (Daily Active Users)
2. User actions     → actions per user per day
3. QPS              → DAU × actions / 86,400 seconds
4. Peak QPS         → average QPS × peak multiplier (2–5×)
5. Storage per unit → size of one record (bytes)
6. Storage growth   → records/day × size × retention period
7. Bandwidth        → QPS × average response size
```

### Step-by-Step Example: Twitter-Scale Service

**Given:** 300M DAU, average user reads 100 tweets/day, writes 0.5 tweets/day.

**Read QPS:**

```
300M × 100 reads / 86,400 seconds
= 30,000,000,000 / 86,400
≈ 350,000 read QPS (350K)

Peak: 350K × 3 = ~1M read QPS
```

**Write QPS:**

```
300M × 0.5 writes / 86,400
= 150,000,000 / 86,400
≈ 1,700 write QPS

Peak: 1,700 × 5 = ~8,500 write QPS
```

**Storage (tweets):**

```
Average tweet body: 280 chars in UTF-8 — assume ~2 bytes/char as
                    a worst-case upper bound for mixed scripts
                    (Latin = 1 byte, CJK / emoji = 3–4 bytes)   ≈ 560 bytes
Metadata (user_id, timestamp, indexes): ~200 bytes
Media references (URLs, not actual media): ~100 bytes
Total per tweet: ~860 bytes ≈ 1 KB

New tweets/day: 300M × 0.5 = 150M tweets
Daily storage: 150M × 1 KB = 150 GB/day
Yearly: 150 GB × 365 = ~55 TB/year
5-year retention: ~275 TB
```

**Media storage:**

```
10% of tweets have an image: 15M images/day
Average image (compressed, multiple sizes): 500 KB
Daily media: 15M × 500 KB = 7.5 TB/day
Yearly: ~2.7 PB/year
```

**Bandwidth:**

```
Read bandwidth: 350K QPS × 1 KB (tweet) = 350 MB/s = 2.8 Gbps
With media loaded on 20% of reads: + 350K × 0.2 × 500 KB = 35 GB/s

Write bandwidth: 1,700 QPS × 1 KB = 1.7 MB/s (negligible)
Media uploads: 1,700 × 0.1 × 2 MB (raw) = 340 MB/s
```

## Common Size References

| Data Type | Typical Size | Notes |
|-----------|-------------|-------|
| Short text (tweet, message) | 200 B – 1 KB | UTF-8/16 + metadata |
| JSON API response | 1 KB – 10 KB | Depends on nesting |
| User profile record | 1 KB – 5 KB | Name, email, preferences |
| Thumbnail image | 10 KB – 50 KB | 150×150 JPEG |
| Compressed image | 100 KB – 1 MB | 1080p JPEG/WebP |
| Audio (1 minute) | ~1 MB | 128 kbps MP3 |
| HD video (1 second) | ~1 MB | 8 Mbps H.264 |
| HD video (1 minute) | ~60 MB | Before CDN caching |
| Database row (typical OLTP) | 100 B – 1 KB | Depends on column count |

## Conversion Cheat Sheet

```
1 KB  = 1,000 bytes      (10³)
1 MB  = 1,000 KB          (10⁶)
1 GB  = 1,000 MB          (10⁹)
1 TB  = 1,000 GB          (10¹²)
1 PB  = 1,000 TB          (10¹⁵)

86,400 seconds in a day
~2.6 million seconds in a month
~31.5 million seconds in a year

1 million requests/day ≈ 12 QPS
1 billion requests/day ≈ 12,000 QPS
```

## Peak Load Estimation

Average QPS is misleading — real traffic is bursty. Size infrastructure for **peak**, not average.

```
Daily traffic pattern (social media):

QPS
 │
 │     ★ peak (lunch break)
 │    ╱ ╲
 │   ╱   ╲     ★ evening peak
 │  ╱     ╲   ╱ ╲
 │ ╱       ╲ ╱   ╲
 │╱         ╲     ╲
 └──────────────────────
  0    6   12   18   24  hour

Peak / Average ratio: typically 2–3× for most apps, 5–10× for event-driven (Super Bowl, flash sales)
```

| Traffic Pattern | Peak/Avg Ratio | Example |
|----------------|---------------|---------|
| Steady (B2B SaaS) | 1.5–2× | Internal tools, enterprise APIs |
| Diurnal (social media) | 2–3× | Twitter, Instagram |
| Weekly (e-commerce) | 3–5× | Weekend shopping spikes |
| Event-driven | 5–100× | Flash sales, live sports, breaking news |

**Rule of thumb:** design for 3× average QPS as your baseline; add auto-scaling for higher peaks.

## Putting It Together: The Interview Worksheet

```
System: ____________________

USERS
  DAU:                    ______
  Peak concurrent:        ______ (typically 10-20% of DAU)

READS
  Actions/user/day:       ______
  Average QPS:            DAU × actions / 86,400 = ______
  Peak QPS:               avg × ____ = ______

WRITES
  Actions/user/day:       ______
  Average QPS:            DAU × actions / 86,400 = ______
  Peak QPS:               avg × ____ = ______

STORAGE
  Size per record:        ______ bytes
  Records per day:        ______
  Daily storage growth:   ______ GB/day
  5-year total:           ______ TB

BANDWIDTH
  Avg read bandwidth:     QPS × response_size = ______ MB/s
  Avg write bandwidth:    QPS × request_size  = ______ MB/s

INFRASTRUCTURE IMPLICATIONS
  - Single DB or sharded?
  - Cache needed? (if read QPS > 10K)
  - CDN needed? (if serving media)
  - Object storage needed? (if media > 1 TB)
```

{{< callout type="warning" >}}
**Don't over-precision your estimates.** The point is order-of-magnitude correctness, not exact numbers. "~350K read QPS" is useful — it tells you a single PostgreSQL instance won't cut it and you need caching. "347,222.22 read QPS" is false precision and wastes time. Round aggressively and focus on the architectural implications.
{{< /callout >}}

{{< callout type="info" >}}
**Interview tip:** Start every system design with a 2-minute capacity estimate: "300M DAU, 100 reads per user per day gives us ~350K read QPS, peaking at ~1M. Each tweet is ~1 KB, so 150M tweets/day is 150 GB of new text data per day. With 10% having images at 500 KB each, that's 7.5 TB/day in media. This tells me we need a distributed cache for reads, object storage for media behind a CDN, and the text data fits in a sharded database." This frames every subsequent design decision.
{{< /callout >}}