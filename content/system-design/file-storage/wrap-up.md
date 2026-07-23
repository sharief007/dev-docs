---
title: 'Wrap-Up'
weight: 5
type: docs
---

## Interview Tips

- **Open by naming the two hard problems.** "File storage is fundamentally a block storage system plus a sync protocol. The two hard parts are (1) chunking and deduplication — storing each unique block once — and (2) sync correctness — ensuring every device converges to the latest state." This framing signals that you understand what makes this design interesting.

- **Drive the chunking discussion.** Most candidates say "split the file into chunks." Strong candidates explain *why* fixed-size chunking fails (boundary-shifting on insertions), then introduce content-defined chunking with a rolling hash, and explain the min/max chunk size constraints. This is the signature insight of the design.

- **Content-addressed storage is the lever.** Once you say "the SHA-256 hash IS the storage key," deduplication, resumability, and block immutability all fall out naturally. Dedup: identical blocks from any user share one storage entry. Resumability: `check-blocks` tells you which blocks the server already has. Block immutability: you never update a block, only add new ones. All three are consequences of the same architectural choice.

- **Bloom filter is expected here.** Unlike the URL shortener (where it's a bonus), the Bloom filter for the block existence check is load-critical in file storage. Explain that 1.25 T block lookups at upload time would saturate the block registry without it, and that a false-positive merely triggers an unnecessary registry lookup while a false-negative is impossible.

- **Merkle trees for delta sync.** Explain that comparing two N-block manifests is O(N); a Merkle tree reduces this to O(K log N) where K is the number of changed blocks. This is the argument Git uses — the interviewer will recognise it.

- **State the consistency requirements precisely.** Block existence checks must be strongly consistent (false "already exists" → silent data loss). Cross-device sync is eventually consistent (a few seconds of lag is fine). This distinction drives architecture decisions.

- **Common follow-ups to rehearse:**
  - How do you handle a file deleted on one device while being edited on another?
  - How do you implement folder sharing with hierarchical permissions?
  - What happens if the Bloom filter node crashes?
  - How do you GC blocks whose `ref_count` drops to zero?
  - How would you add real-time collaborative editing (answer: you'd need OT or CRDTs — a fundamentally different architecture)?

---

## Resiliency

- **Stateless app tier + multi-region.** Upload, Download, Sync, and Share services hold no session state. Any node in any region can serve any request; a regional outage is absorbed by the global load balancer rerouting traffic.

- **Object store triple-replication + erasure coding.** Block content is the most durable layer: S3-class object storage achieves 11-nines durability by maintaining replicas across multiple Availability Zones with erasure coding. Losing a single AZ does not lose any data.

- **Block idempotency as a resiliency primitive.** Because every block write is idempotent (`PUT /blocks/{hash}` is safe to retry any number of times), the upload path is naturally resilient to at-least-once delivery. Retries never create duplicate blocks.

- **Upload session persistence.** Upload session state (which blocks were received) is stored durably in the metadata DB, not in app node memory. A crashing upload service node does not lose session progress; the client reconnects to any node and resumes.

- **Bloom filter is non-critical path.** If the Bloom filter cluster is unavailable, the upload service falls back to querying the block registry directly. Throughput drops (more DB queries) but correctness is preserved. The filter is rebuilt from the block registry on restart via a background job.

- **Change log as the safety net.** The Kafka event bus retains events for 7 days (configurable). If the notification service is down, clients miss WebSocket pushes but catch up via `GET /sync/changes?cursor=N` on reconnect. No sync event is permanently lost as long as the Kafka retention window covers the outage duration.

- **Graceful degradation on metadata DB overload.** The sync service reads from Redis-cached manifests. During a metadata DB write spike, read-heavy sync operations continue serving from cache. New uploads degrade gracefully: block writes to object storage succeed independently; manifest commits queue behind DB capacity, and the client retries.

- **GC safety.** Block garbage collection (`ref_count → 0 → delete from object store`) runs asynchronously and conservatively: a block is only deleted after its `ref_count` has been 0 for 24 hours (to allow in-flight manifests to commit). This ensures no block is deleted while an upload is still in progress.

---

## Observability

- **SLIs:** upload block success rate; upload end-to-end latency (p50/p99); download first-byte latency from CDN (p50/p99); sync notification delivery latency (p99); block dedup hit rate (blocks skipped / blocks checked); Bloom filter false-positive rate; Kafka consumer lag (notification service); `ref_count = 0` block backlog (GC health).

- **Golden alerts:**
  - Upload success rate < 99% (indicates storage or network failure).
  - Block dedup hit rate drops below 40% (unexpected data without prior history — potential abuse or misconfigured clients).
  - Sync notification p99 > 10 s (Kafka consumer lag growing; check notification service scaling).
  - Bloom filter false-positive rate > 5% (filter is undersized or stale — schedule refresh).
  - Download p99 from CDN > 500 ms (CDN origin-fill rate too high; check cache hit ratio).
  - Object store write error rate > 0.1% (S3 API errors; page on-call immediately — block writes failing means uploads failing).

- **Tracing.** Propagate a `trace_id` from the client upload SDK through all hops: API gateway → upload service → block registry → object store → manifest commit → Kafka event → notification service → WebSocket push. End-to-end traces enable attribution of tail latency to a specific tier.

- **Dashboards:**
  - Upload funnel: upload sessions created → blocks checked → blocks uploaded (new vs. deduped) → commits succeeded. Drop-offs at each stage indicate bottlenecks.
  - Dedup savings: daily raw bytes received vs. net bytes written to object store; cumulative storage saved.
  - Sync lag: histogram of time between file commit and WebSocket delivery across all active clients.
  - Block registry size vs. Bloom filter freshness: track how far behind the filter is relative to the registry.
  - Object store cost: bytes stored × replication factor, tracked against budget.

- **Logging.** Sample block upload logs at 1% (too high volume to log everything). Log 100% of commit events, share grant/revoke events, and all ACL denials (security audit trail). Do not log presigned URLs (they are bearer tokens).

---

## Concepts Used

- {{% relref "/design-concepts/storage/object-storage" %}} — durable block content storage; key design and cross-AZ replication
- {{% relref "/design-concepts/storage/bloom-filters" %}} — cheap probabilistic block-existence check for the dedup fast path
- {{% relref "/design-concepts/storage/consistent-hashing" %}} — sharding the block registry and metadata DB
- {{% relref "/design-concepts/storage/key-value-stores" %}} — block registry; point lookups by SHA-256 hash
- {{% relref "/design-concepts/storage/caching-patterns" %}} — Redis cache for hot file manifests and ACL results
- {{% relref "/design-concepts/storage/hotspot-problems" %}} — notification fan-out for widely shared files; debounce strategy
- {{% relref "/design-concepts/networking/cdn" %}} — block content delivery; CDN is mandatory at 46 GB/s egress
- {{% relref "/design-concepts/distributed/idempotency" %}} — idempotent block PUTs; at-least-once safe upload and notification delivery
- {{% relref "/design-concepts/data/change-data-capture" %}} — CDC on metadata DB driving Kafka event stream for sync notifications; Bloom filter refresh
- {{% relref "/design-concepts/api/pagination" %}} — cursor-based `/sync/changes` endpoint; monotone sequence cursor for reliable replay
