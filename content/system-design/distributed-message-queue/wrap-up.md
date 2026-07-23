---
title: 'Wrap-Up'
weight: 4
type: docs
---

## Interview Tips

- **Anchor on "a topic is a partitioned append-only log."** Every property — throughput, ordering, retention, replay — falls out of that one idea. Say it first; it frames the whole interview.
- **Explain why sequential disk + page cache beats "keep it in RAM."** Interviewers expect you to know that append-only sequential I/O plus OS page cache and zero-copy is what makes commodity disks sustain GB/s. This is the counter-intuitive insight they probe.
- **Own the durability story: ISR + acks + min.insync.replicas.** Be able to walk through exactly when a message is "committed" and how many failures you tolerate. This is the single most common deep-dive.
- **Partition count = parallelism = ordering boundary.** Explain the trade-off: more partitions → more throughput and consumers, but ordering only holds *within* a partition, and rebalances/metadata cost grows.
- **Delivery semantics ladder:** at-most-once (acks=0), at-least-once (default), exactly-once (idempotent producer + transactions). Know what each costs.
- **Push vs pull:** consumers **pull** (fetch) so they control their rate and can replay — contrast with push systems that can overwhelm slow consumers (backpressure for free).
- **Likely follow-ups:** consumer lag monitoring, hot partitions from a skewed key, compaction vs deletion retention, how the controller stays consistent (Raft), and "how would you do exactly-once end-to-end?"

## Resiliency

- **Replication + ISR:** tolerate `|ISR|-1` broker failures per partition with zero loss of acked data. See {{% relref "/design-concepts/distributed/quorum" %}}.
- **Leader failover from ISR:** guarantees the new leader has all committed messages; controlled by disabling unclean leader election.
- **Consensus-backed metadata:** the controller runs on Raft, so cluster state survives node loss. See {{% relref "/design-concepts/consensus/raft" %}}.
- **Backpressure by design:** pull-based consumers plus retained logs mean a slow consumer never destabilizes the cluster — it just lags. See {{% relref "/design-concepts/reliability/back-pressure" %}}.
- **Rack/AZ-aware replica placement:** spread a partition's replicas across failure domains so one AZ loss can't take all replicas.
- **Multi-region:** async mirroring (e.g. MirrorMaker) for DR; accept cross-region lag. See {{% relref "/design-concepts/distributed/multi-region-design" %}}.
- **Idempotent producers** neutralize retry duplication.

## Observability

- **SLIs:** produce latency (p99), under-replicated partition count, offline partition count, ISR shrink/expand events, consumer group lag per partition, request-handler idle ratio, disk & network utilization per broker.
- **Golden alerts:** any offline partitions, under-replicated partitions > 0 for sustained time, ISR below `min.insync.replicas`, consumer lag growing unbounded, controller elections flapping.
- **Tracing / lineage:** propagate a message id/headers so a record can be followed producer → partition → consumer across pipelines.
- **Dashboards:** throughput (msgs/s & bytes/s) per topic, per-broker load balance (detect hot partitions), retention/disk headroom, lag heatmap by consumer group.
- **Capacity signals:** partition count vs consumer count (are consumers starved or idle?), skew across partitions (hot-key detection — see {{% relref "/design-concepts/storage/hotspot-problems" %}}).

## Concepts Used

- {{% relref "/design-concepts/messaging/kafka" %}} — the reference implementation of this design
- {{% relref "/design-concepts/messaging/queues-vs-streams" %}} — log-based streams vs traditional queues
- {{% relref "/design-concepts/scaling/sharding" %}} & {{% relref "/design-concepts/storage/consistent-hashing" %}} — partitioning
- {{% relref "/design-concepts/replication/leader-based-replication" %}} — partition leaders & followers
- {{% relref "/design-concepts/distributed/quorum" %}} — ISR / durability quorum
- {{% relref "/design-concepts/consensus/raft" %}} & {{% relref "/design-concepts/distributed/leader-election" %}} — controller metadata & failover
- {{% relref "/design-concepts/distributed/idempotency" %}} — exactly-once building block
- {{% relref "/design-concepts/messaging/dlq-and-retry" %}} — poison-message handling
