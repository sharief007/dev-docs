---
title: 'Wrap-Up'
weight: 5
type: docs
---

## Interview Tips

- **Open with the two-plane separation.** This design has two completely independent traffic planes: the upload/transcoding plane (write-heavy, compute-intensive, latency-tolerant) and the streaming/delivery plane (read-extremely-heavy, latency-sensitive, CDN-dominated). Stating this in the first minute immediately shows architectural clarity.

- **Lead with CDN egress as the dominant number.** 40M concurrent streams × 2 Mbps = **80 Tbps** of CDN egress. This single number explains why CDN is not an afterthought — it is the system. Interviewers want to see you do this arithmetic, not cite it from memory.

- **Make the storage multiplier visible.** Show that 5.5× transcoding storage multiplier explicitly. "500 hours uploaded/minute sounds like a storage problem, but the real problem is egress." Reframing the problem shows maturity.

- **The transcoding pipeline is the meat.** Most candidates describe transcoding as "send to a worker." Push it further: GOP-aligned chunking enables parallelism (a 10-min video = 1,800 independent jobs), explain the DAG stages, and mention the fast/slow codec path so H.264 publishes in minutes while AV1 is computed asynchronously.

- **HLS/DASH is often underspecified.** Walk through the two-file structure (master playlist + per-rendition playlist), explain why segments start on IDR boundaries, and sketch the ABR control loop. These details separate a system-design answer from a vague handwave.

- **Volunteer video deduplication.** Bringing up content fingerprinting (pHash of keyframes before transcoding) is a senior signal — it's not obvious but it's real (YouTube does this). Even a one-sentence mention earns credit.

- **Common follow-ups to rehearse:** How would you handle live streaming differently? (Shorter segments, growing playlist, no `#EXT-X-ENDLIST`.) How do you prevent CDN cache stampede when a new video goes viral? (Pre-warm the first 30 s.) How would you implement DRM? (Encrypt segments with AES-128; embed key URL in manifest pointing to a key server.) How do you handle geo-restricted content? (CDN token signing; manifest service embeds time-limited signed URLs.)

## Resiliency

- **Object storage durability.** Raw uploads and segments are stored with 11-nines durability via multi-region replication. Once a segment is written it is immutable — no consistency headache, no cache invalidation problem. The CDN can safely cache segments with 1-year TTLs.

- **Transcoding worker idempotency.** Every encode job targets a deterministic output key and writes to a staging path before atomic rename. At-least-once queue delivery + idempotent jobs = exactly-once effect without distributed transactions. A fleet of spot/preemptible instances can be used for AV1 workers without data loss risk.

- **Upload service statelessness.** The Upload Service brokers presigned URLs and emits events; it holds no binary data. Any instance can fail and the creator retries seamlessly (parts already uploaded are not re-sent — the multipart protocol tracks ETags server-side in object storage).

- **CDN fault isolation.** Segment immutability means CDN nodes never need to validate freshness. A CDN edge PoP failure causes the DNS/Anycast layer to route to an adjacent PoP. No data loss — the segment is at the origin.

- **View counter graceful degradation.** Redis counter increments are best-effort. A Redis node failover (< 30 s with Sentinel) causes a brief under-count window. The Kafka view-event stream captures every event durably for later exact analytics — the Redis counter is purely the "fast approximate display" layer. See {{% relref "/design-concepts/messaging/kafka" %}}.

- **Coordinator fault tolerance.** The Transcoding Coordinator persists the DAG state (per-job status row) to a durable store before emitting to the job queue. A coordinator crash is recoverable: on restart, it reads uncompleted DAGs and re-emits missing jobs (safe because workers are idempotent). See {{% relref "/design-concepts/distributed/idempotency" %}}.

- **Backpressure on upload bursts.** If a viral event causes a spike in uploads, the Kafka topic absorbs the burst naturally. Transcoder workers drain at their own pace. Upload QPS (2/s average) is so low that the queue depth almost never grows. See {{% relref "/design-concepts/reliability/back-pressure" %}}.

## Observability

### SLIs and Golden Signals

| Signal | Metric | Alert threshold |
|---|---|---|
| **Playback availability** | % of manifest and segment requests returning 2xx | < 99.9% → page |
| **Rebuffering ratio** | ratio of stall time to total watch time (reported by player) | > 1% → investigate |
| **Time-to-first-segment** | p50 and p99 of elapsed time from play-press to first segment decode | p99 > 3 s → alert |
| **CDN cache hit ratio** | % of segment requests served from edge cache | < 98% → investigate |
| **Transcoding pipeline lag** | time from upload complete to video READY | p99 > 10 min → alert |
| **Upload error rate** | % of multipart uploads that fail to complete | > 0.5% → investigate |
| **View counter drift** | difference between Redis counter and Kafka-exact count | > 5% → alert |
| **Comment write error rate** | % of comment writes returning non-2xx | > 0.1% → alert |

### Logging and Tracing

- **Trace propagation:** a single `trace_id` is minted at the API Gateway and propagated through the upload service, Kafka message headers, the transcoding coordinator, and each worker. A full trace reconstructs the lifecycle of a single video upload: from initiate → each segment encode job → manifest publish → first viewer segment fetch.
- **Segment fetch logs:** CDN edge logs are sampled at 0.1% for popular content (logging every segment fetch at 13M/s is cost-prohibitive). All errors (4xx, 5xx) are logged fully.
- **Transcoding job logs:** every job logs its start time, worker ID, encode duration, output size, and exit code. These feed an alerting rule on abnormally long encode times (stuck workers).
- **Player telemetry:** the player SDK reports: playback start events, quality switch events (with reason: bandwidth/buffer), stall events, error events. These are the source of the rebuffering ratio SLI.

### Dashboards

- **Streaming health:** real-time CDN request rate, cache hit ratio per region, rebuffering ratio by country, p99 segment delivery latency heatmap.
- **Upload and transcoding:** upload initiations per minute, transcoding queue depth, worker fleet utilisation, p99 time-to-READY, failed video rate.
- **Social signals:** comment write rate, view event throughput, Redis counter vs. Kafka-exact delta.
- **Cost:** CDN egress TB/day (the biggest cost lever), object storage TB added per day vs. projection, transcoder worker-hours per rendition type.

## Related Concepts

- {{% relref "/design-concepts/storage/object-storage" %}} — durable segment and raw video storage with 11-nines durability
- {{% relref "/design-concepts/networking/cdn" %}} — global segment delivery; origin shield; cache-control for immutable segments
- {{% relref "/design-concepts/messaging/kafka" %}} — upload event stream; view event pipeline; decoupling ingestion from transcoding
- {{% relref "/design-concepts/messaging/queues-vs-streams" %}} — job queue (transcoding jobs) vs. event stream (view events) trade-offs
- {{% relref "/design-concepts/storage/wide-column-stores" %}} — Cassandra for comments partitioned by video_id
- {{% relref "/design-concepts/storage/caching-patterns" %}} — Redis view counters; segment caching strategy; comment leaderboard sidecar
- {{% relref "/design-concepts/storage/key-value-stores" %}} — Redis for view counters, like deduplication sets, comment leaderboard
- {{% relref "/design-concepts/storage/hotspot-problems" %}} — viral video cache stampede; view counter write hotspot
- {{% relref "/design-concepts/specialized/job-scheduling" %}} — transcoding DAG coordination; worker fleet autoscaling
- {{% relref "/design-concepts/data/batch-vs-streaming" %}} — approximate view counts (Redis/batch sync) vs. exact analytics (Kafka/Flink)
- {{% relref "/design-concepts/distributed/idempotency" %}} — idempotent segment encode jobs enabling safe at-least-once delivery
- {{% relref "/design-concepts/reliability/back-pressure" %}} — Kafka absorbing upload burst spikes without stalling the ingestion path
