---
title: 'Wrap-Up'
weight: 6
type: docs
---

## Interview Tips

- **Open with the scale math.** "At 1B pages/month that's ~385 pages/second. Let me size workers and storage first." Anchoring in numbers immediately sets a professional tone and bounds every subsequent design decision.
- **Draw the two-level frontier early.** The priority + per-domain two-level queue is the central data-structure insight of a web crawler design. Put it on the whiteboard first. Explain why a naive FIFO fails on both dimensions — priority and politeness — before proposing the fix.
- **Frame the problem as graph traversal.** "The web is a directed graph; pages are nodes, hyperlinks are edges. We need to traverse it efficiently from seeds." Briefly contrast BFS vs DFS, then explain why production crawlers use priority-weighted BFS. This shows conceptual breadth.
- **Bloom filter is expected at senior level.** Bring it up with the sizing math: 12B URLs at 1% FP ≈ 14.4 GB. Acknowledge false positives (acceptable — we occasionally skip a page) and that false negatives are impossible (correctness preserved). Distinguish from content dedup (SimHash).
- **Politeness is a differentiator.** Many candidates skip it entirely. Mention `robots.txt`, Crawl-delay, exponential backoff on 429/503, per-domain back queues, and the min-heap scheduler. This signals real-world operational awareness.
- **Consistent hashing ties politeness to distribution.** Explain that assigning domains to workers by consistent hash is what makes per-worker politeness schedulers work without distributed locks. This is a strong architectural insight.
- **Common follow-ups to rehearse:**
  - *"How do you handle JS-rendered pages?"* → headless render worker pool; selective routing based on JS-detection heuristics; trade-offs of cost and detectability.
  - *"How do you scale to 10B pages/month?"* → add workers (consistent hash ring rebalance), scale Kafka partitions, distribute the Bloom filter, add object store bandwidth.
  - *"How do you decide when to re-crawl?"* → EMA-based adaptive interval + sitemap `<lastmod>` override + 30-day hard cap.
  - *"What if robots.txt is missing?"* → RFC 9309 says treat as permissive; always retry in background.
  - *"How do you avoid crawl traps?"* → URL normalization (highest leverage), depth limit, per-domain URL cap, redirect chain limit.
  - *"What's your consistency model?"* → at-least-once delivery with idempotent dedup; eventual consistency is acceptable (re-crawling a page twice is wasteful, not incorrect).

## Resiliency

- **Stateless workers beyond local queues.** Worker state consists only of in-process back queues and the robots.txt LRU cache. On crash, the consistent hash ring rebalances; Kafka consumer group reassigns the partition; the replacement worker rebuilds its back queue from the Kafka partition and its robots.txt cache on first access.
- **Bloom filter persistence.** Snapshotted to object storage every 5 minutes. On restart: load snapshot + replay Kafka from the checkpoint offset. The 5-minute window of potential replay creates at-most ~50K duplicate URL checks — a no-op given content hash dedup.
- **At-least-once delivery with idempotent processing.** Kafka offset committed only after full processing. Replayed URLs are suppressed by the Bloom filter (or re-fetched if truly absent); replayed content is suppressed by the content hash dedup. See {{% relref "/design-concepts/distributed/idempotency" %}}.
- **Object store durability.** Raw HTML is written with at-least-3× replication. `crawled_pages` metadata is written only after the object store write is confirmed, preventing orphaned references.
- **robots.txt unavailability.** Treat 5xx as permissive; log for audit; retry background refresh. Never halt a domain's queue because of a single robots.txt fetch failure.
- **Backpressure on the frontier.** If the Kafka frontier topic grows unboundedly (workers consistently falling behind), throttle the Seed Injector and Re-crawl Scheduler to match actual worker throughput. See {{% relref "/design-concepts/reliability/back-pressure" %}}. The Kafka partition consumer lag metric is the backpressure signal.
- **Consistent hash rebalance on worker scale-out/in.** ~1/N of domains remapped; their Kafka partitions are reassigned by the consumer group protocol. In-flight messages for remapped domains are replayed from the last committed offset.

## Observability

- **SLIs:**
  - Pages crawled per second (actual vs target 385/s)
  - Kafka frontier partition lag per worker (leading indicator of worker health)
  - Bloom filter estimated false-positive rate (sampled audit)
  - robots.txt cache hit ratio per worker
  - Per-domain crawl delay compliance (fraction of fetches that respect configured delay)
  - Content store write latency p50 / p99
  - SimHash near-duplicate fraction of crawled pages
  - Re-crawl scheduler lag (how far behind `next_crawl_at ≤ now` is the scan)
  - Headless render worker queue depth and render latency

- **Golden alerts:**
  - Frontier Kafka lag > 10M messages and rising → workers falling behind; scale out or throttle sources
  - Domain error rate (5xx + connection timeouts) > 20% for any worker → possible IP block or origin issue
  - Bloom filter memory usage approaching capacity → needs expansion or partitioning
  - Re-crawl scheduler falling > 24 hours behind `next_crawl_at` → freshness degrading
  - Object store write failure rate > 0.1% → durability risk; investigate immediately

- **Distributed tracing.** Propagate a `crawl_trace_id` from seed ingestion through the Kafka message → Frontier Dispatcher → Fetch Worker → Link Extractor → Content Store. Use it to reconstruct the full lifecycle of any URL for debugging traps, missed pages, or unexpected deduplication.

- **Dashboards:**
  - Real-time pages/s by domain category (news, e-commerce, reference, social)
  - Frontier queue depth per worker over time (leading indicator of worker health)
  - Top 50 domains by crawl budget consumption (spot hotspot domains)
  - Content dedup ratio: exact-dup fraction + SimHash near-dup fraction of total fetches
  - Freshness histogram: distribution of page age since last successful crawl
  - Crawl error breakdown: 4xx, 5xx, timeouts, robots-blocked, trap-discarded

## Concepts Used

- {{% relref "/design-concepts/storage/bloom-filters" %}} — URL deduplication; false-positive analysis and sizing for 12B URLs
- {{% relref "/design-concepts/storage/consistent-hashing" %}} — assigning domains to workers; minimal rehash on fleet scale-out
- {{% relref "/design-concepts/messaging/kafka" %}} — URL frontier as a durable partitioned log; at-least-once delivery via offset management
- {{% relref "/design-concepts/messaging/queues-vs-streams" %}} — why a stream beats a queue for a replayable, durable frontier
- {{% relref "/design-concepts/rate-limiting/algorithms" %}} — token-bucket per-domain rate limiting; exponential backoff with jitter on 429/503
- {{% relref "/design-concepts/networking/dns" %}} — per-worker DNS caching; NXDOMAIN caching to avoid hammering DNS for dead domains
- {{% relref "/design-concepts/storage/object-storage" %}} — raw HTML content store; durability model and cost characteristics
- {{% relref "/design-concepts/storage/hotspot-problems" %}} — popular domains as crawl bottlenecks; sub-queue partitioning by path prefix
- {{% relref "/design-concepts/distributed/idempotency" %}} — at-least-once URL delivery; idempotent re-fetch semantics
- {{% relref "/design-concepts/specialized/job-scheduling" %}} — adaptive re-crawl scheduling; EMA-driven interval + hard-cap fallback
- {{% relref "/design-concepts/reliability/back-pressure" %}} — throttling seed injection and re-crawl scheduler when workers fall behind
- {{% relref "/design-concepts/storage/wide-column-stores" %}} — `crawled_pages` store partitioned by domain for efficient re-crawl scans
