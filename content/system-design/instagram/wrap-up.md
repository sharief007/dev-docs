---
title: 'Wrap-Up'
weight: 5
type: docs
---

## Interview Tips

- **Lead with the media pipeline, not the social graph.** Unlike a URL shortener or Twitter, the defining complexity here is **upload → async process → CDN serve**. Frame this within the first 5 minutes; it shows you understand what makes photo sharing structurally different from text-only social platforms.

- **Quantify the egress math.** Saying "800 Gbps CDN bandwidth" and explaining why it forces a CDN-first architecture (not direct S3 serving) is a senior-level signal. Walk through: 2.5B feed opens × 12 thumbnails × 50 KB = ~1,500 TB/day thumbnails, plus full-size views, then divide by 86,400 s to get GB/s. Interviewers who ask about bandwidth are listening for this reasoning.

- **Name the hybrid fan-out explicitly.** State the fan-out on write (regular users) / fan-out on read (celebrities > 50K followers) hybrid, and note that you fan out **photo IDs** not full content — photo bytes are always CDN-served. This is the same pattern as Twitter but media-focused.

- **Justify Cassandra for likes and comments.** Explain: high write QPS with no complex joins, access pattern is always by `photo_id` — a textbook wide-column fit. Mention counter columns for like counts, and partition bucketing (`photo_id % N`) for viral-post hot-partition mitigation.

- **Volunteer resumable uploads.** Awareness of large-file / mobile-network challenges (S3 multipart, TUS protocol) is a real-world signal that differentiates candidates who have built production media systems.

- **Common follow-ups to rehearse:**
  - *How do you handle a post going viral?* → Redis counters (O(1) INCR), Cassandra bucketed partitions, CDN absorbs photo-serving load.
  - *How does Explore work?* → Two-stage: offline trending-score batch (50K candidates into Redis ZSET) + online ML re-ranking + per-user Bloom filter for seen-post filtering.
  - *How do notifications avoid spam?* → Kafka aggregation windows (30 s tumbling), user preference check, APNs/FCM with in-app Cassandra inbox as fallback.
  - *What if the image processing worker fails?* → Kafka offset not committed → event requeued; S3 PUT is idempotent by key.
  - *How do you paginate the feed?* → Cursor-based on `(rank_score, photo_id)` — stable through re-ranking; not offset-based.
  - *How do you handle a celebrity crossing the 50K follower threshold?* → A background migration job reclassifies them, moves future fan-out to the celebrity_posts table, and optionally backfills existing fan-out entries.

---

## Resiliency

- **Stateless app tier + multi-region.** All services (Upload, Feed, Social Graph, Like, Explore) are stateless and horizontally scalable. Global load balancer routes to the nearest healthy region; a regional failure sheds traffic to others.

- **Object storage durability.** S3 provides 11 nines of durability with cross-region replication (S3 CRR). Processed variants are regenerable from the original — only originals need cross-region replication; thumbnails can be re-derived if a region is permanently lost.

- **Kafka consumer group isolation.** Image Workers, Fan-Out Workers, and Notification Aggregators are independent consumer groups. A backlog in fan-out does not block image processing or notifications; each scales independently. Consumer lag is the leading indicator for scaling decisions.

- **Feed fallback chain.** Redis ranked ZSET → Cassandra `user_feed` scan → (emergency) pull-on-read from follows graph. Each tier is slower but correct. See {{% relref "/design-concepts/storage/caching-patterns" %}}.

- **Dead-letter queue.** Image processing jobs that fail N times (corrupt file, OOM, codec bug) are routed to a DLQ for human review without blocking the pipeline. See {{% relref "/design-concepts/messaging/dlq-and-retry" %}}.

- **Idempotent fan-out.** Cassandra `INSERT IF NOT EXISTS` on `(user_id, score, photo_id)` makes replaying a `photo.published` event safe — no duplicate feed rows.

- **Circuit breaker on ML ranking.** If the ranking service is slow or unavailable, the Feed Service falls back to chronological order (timestamp score) automatically — degraded experience, not an outage. See {{% relref "/design-concepts/reliability/circuit-breaker" %}}.

- **Back-pressure on upload.** If image processing workers are overloaded, Kafka queue depth grows — observable and alertable before users are impacted. The upload acknowledgement (202) is decoupled from processing completion, so back-pressure in the pipeline never stalls client uploads. See {{% relref "/design-concepts/reliability/back-pressure" %}}.

- **Read-your-writes for follow/unfollow.** The Social Graph Service routes the acting user's subsequent reads to the SQL primary for a short window (e.g. 5 s), ensuring they immediately see the effect of their own follow/unfollow action.

---

## Observability

### SLIs / Key Metrics

| Metric | Target | Why it matters |
|---|---|---|
| Feed load p50 / p99 | < 100 ms / < 200 ms | Core user experience |
| Photo thumbnail CDN hit ratio | > 90% | Drives storage cost and origin load |
| Upload 202 response rate | > 99.5% | Proxy for upload availability |
| Image processing lag (upload → published) | p99 < 10 s | Feed freshness for poster |
| Fan-out lag (published → last follower) | p99 < 30 s | Feed freshness SLA |
| Kafka consumer group lag — Image Workers | Alert > 100K msgs | Processing bottleneck |
| Kafka consumer group lag — Fan-Out Workers | Alert > 500K msgs | Fan-out bottleneck |
| Like/comment write error rate | < 0.1% | Data loss for social engagement |
| CDN egress TB/day | Track vs. budget | Dominant cost driver |

### Golden Alerts

- **CDN hit ratio drops below 85%:** possible cache misconfiguration, new photo format not cacheable, or S3 origin throttling. Page immediately — cost and latency both spike.
- **Fan-out lag exceeds 30 s:** fan-out worker pool undersized; auto-scale Kafka consumers and alert on-call if not recovering within 2 minutes.
- **Image processing Kafka lag > 1M messages:** processing bottleneck; page on-call — feed freshness for all new posts is degraded.
- **Feed p99 > 500 ms:** Redis or Cassandra latency spike; check node health and replication lag.
- **Like write error rate > 0.1%:** Cassandra or Redis degraded; potential unacknowledged likes (data loss risk).

### Tracing

Propagate a trace ID through the full photo lifecycle:

```
Mobile upload  →  Upload Service  →  S3 ObjectCreated  →  Kafka  →  Image Worker
→  photo.published Kafka  →  Fan-Out Worker  →  Cassandra user_feed
```

This trace lets on-call engineers answer: "User reports photo not showing in follower feeds after 5 minutes — where in the pipeline did it stall?"

### Dashboards

- **Media pipeline:** uploads/s, processing latency histogram, image worker queue depth, CDN bandwidth and hit ratio, S3 PUT error rate.
- **Feed:** feed reads/s, Redis cache hit ratio, feed freshness distribution (how stale are feeds? — P50/P95 of fan-out lag), ranking service latency histogram.
- **Social engagement:** likes/s, comments/s, Cassandra write latency, hot-partition event rate (bucketing effectiveness).
- **Cost:** CDN egress GB/day (dominant), S3 storage TB total, Cassandra node count and utilisation.

### Logging Strategy

- **Full audit logs** for upload, delete, and follow events (one log line per event, retained 90 days).
- **Sampled logs (1%)** for feed reads and photo views at 86,700/s — full logging at this rate is cost-prohibitive and unnecessary for debugging.
- **Full logs for errors** (4xx, 5xx) regardless of sampling.

---

## Concepts Used

- {{% relref "/design-concepts/storage/object-storage" %}} — storing raw originals and processed variants; S3 multipart upload; cross-region replication
- {{% relref "/design-concepts/networking/cdn" %}} — global photo delivery; 800 Gbps egress; immutable cache-control headers; edge 410 for deleted photos
- {{% relref "/design-concepts/specialized/notification-fanout" %}} — hybrid fan-out (write for regulars, read for celebrities); notification aggregation pipeline
- {{% relref "/design-concepts/storage/key-value-stores" %}} — Redis feed ZSET, like/comment counters, explore trending ZSET
- {{% relref "/design-concepts/storage/wide-column-stores" %}} — Cassandra for likes (bucketed partitions), comments, user_feed, celebrity_posts, notifications
- {{% relref "/design-concepts/ml/recommendation-systems" %}} — ML-ranked feed; two-stage Explore retrieval (trending candidates + per-user re-ranking)
- {{% relref "/design-concepts/storage/caching-patterns" %}} — cache-aside for feed, photo metadata, like counts; fallback chain
- {{% relref "/design-concepts/storage/hotspot-problems" %}} — viral post like storms (Redis INCR + Cassandra bucketing); celebrity feed fan-out; CDN hot keys
- {{% relref "/design-concepts/messaging/kafka" %}} — photo.uploaded / photo.published / social.events event streaming; consumer group isolation
- {{% relref "/design-concepts/api/pagination" %}} — cursor-based feed and comment pagination; cursor encodes (score, photo_id)
- {{% relref "/design-concepts/storage/bloom-filters" %}} — seen-post filtering in Explore; deleted-photo fast-path at CDN edge
- {{% relref "/design-concepts/storage/consistent-hashing" %}} — SQL shard assignment by user_id; adding shards remaps only a fraction of keys
- {{% relref "/design-concepts/security/oauth-oidc" %}} — OAuth 2.0 bearer tokens for all API endpoints
- {{% relref "/design-concepts/reliability/circuit-breaker" %}} — ML ranking fallback to chronological feed on ranking service degradation
- {{% relref "/design-concepts/reliability/back-pressure" %}} — Kafka consumer lag as the back-pressure signal for image processing and fan-out pipelines
- {{% relref "/design-concepts/messaging/dlq-and-retry" %}} — dead-letter queue for persistently failing image processing jobs
