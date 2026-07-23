---
title: 'Wrap-Up'
weight: 5
type: docs
---

## Interview Tips

- **Open with the two structural observations.** (1) The system has ~15M *persistent* WebSocket connections at any moment — connection management is the first-order infrastructure constraint, not storage. (2) End-to-end encryption makes the server a zero-knowledge routing layer, which changes the data model (ciphertext blobs, per-device payloads) and makes multi-device history sync a non-trivial problem. Lead with both before touching QPS or storage.

- **Drive the durability-before-ack discussion.** The most important correctness property: the server issues a SENT receipt *only after* the message is durably written to Cassandra. This guarantees the sender that the message is safe even if both parties go offline immediately afterward. State this explicitly — interviewers probe whether you understand the ack-semantics of the write path.

- **Walk the receipt state machine precisely.** SENT (Cassandra write acked) → DELIVERED (recipient device websocket acked) → READ (user opened conversation). Each transition is a separate event relayed over WebSocket. Receipts are best-effort: if the sender's gateway crashes, a receipt is dropped but the message is not. Reconciliation happens on reconnect.

- **Volunteer the fan-out trade-off for groups.** Unprompted: "For small groups I use fan-out-on-write: one Kafka publish, fan-out service delivers N tasks. For very large groups (> 500 members), I'd switch to lazy fan-out: store one ciphertext; members fetch on conversation open." This signals awareness of write-amplification vs. read-amplification tension.

- **State the E2EE constraint on history sync.** When asked "how does a new device get message history?", the answer must acknowledge the server *cannot* re-decrypt — valid paths are device-to-device encrypted transfer and out-of-band encrypted cloud backup. This shows you understand E2EE as an architectural constraint, not just a feature label.

- **Know the Snowflake ordering guarantee.** If asked about message ordering, explain that clients display messages by *server-assigned* Snowflake order, never client timestamp, to eliminate clock-skew issues on mobile. Near-simultaneous messages sort by generator ID — a stable if arbitrary tiebreak.

- **Common follow-ups:** How would you handle 100K-member broadcast channels? How do you prevent metadata leakage if the server cannot read content? How do you scale the fan-out service for a viral group? How does key recovery work if a user loses all devices? How would you add voice/video call signalling without breaking E2EE?

## Resiliency

- **Gateway server crash:** Connection registry TTL ensures the crashed gateway's routing entries expire within 240 s. Clients reconnect to any available gateway (L4 LB picks the next healthy server). Undelivered messages are in Kafka; the fan-out consumer reprocesses them after partition reassignment. No message is lost.

- **Cassandra node failure:** RF=3 with quorum writes. A single node failure is transparent. Two simultaneous failures degrade quorum writes to hinted handoff (then repair). Messages are at-least-once durable by the time the SENT receipt is returned to the client.

- **Kafka broker failure:** Kafka replication and consumer group protocol handle broker failures automatically; partition leadership fails over to an in-sync replica within seconds. Fan-out consumer groups reassign dead partitions within the consumer session timeout (default 10 s). No delivery tasks are lost (Kafka persists to disk before returning produce ack).

- **Redis failure (routing + presence):** Routing lookups fail → all deliveries fall through to the push notification path. Presence shows stale data. Redis Cluster provides automatic failover to replica within ~30 s. This is graceful degradation — messages still arrive, with slightly higher latency.

- **Push notification overload:** APNs/FCM are external queues with their own rate limits. Back-pressure from those services is absorbed by the Push Notification Service's internal Kafka-backed queue. Burst spikes in offline deliveries are buffered and consumed at a controlled rate. See {{% relref "/design-concepts/reliability/back-pressure" %}}.

- **Idempotency everywhere:** Client sends carry a `client_msg_id` UUID. The server deduplicates on this ID using a short-lived Redis set (TTL = 5 min). Kafka consumers deduplicate on `server_msg_id`. Receipt updates are idempotent (status only advances: SENT → DELIVERED → READ, never reverses). See {{% relref "/design-concepts/distributed/idempotency" %}}.

- **Multi-region design:** Each region runs a complete stack (gateways, Cassandra, Redis, Kafka). Cross-region messages are routed via the Region Router over the private backbone. Cassandra is replicated cross-region asynchronously (async replication avoids write-latency penalty at the cost of a replication lag window). A regional outage sheds to adjacent regions via the global L4 load balancer.

## Observability

**SLIs:**
- Message delivery latency: p50 / p99 measured from `message.send` received at the gateway to `message.deliver` sent to the recipient gateway.
- End-to-end receipt latency: p99 from send to DELIVERED receipt returning to the sender (includes cross-server routing).
- WebSocket connection count per gateway server (capacity headroom signal).
- Kafka consumer lag per inbox topic partition (fan-out health; high lag = delivery falling behind).
- Push notification delivery rate and APNs/FCM error rate per platform.
- Cassandra write latency p99 per datacenter and replication lag.
- Presence staleness: fraction of reads where `last_seen` differs from actual last-heartbeat by > 60 s.
- Key server: OPK replenishment rate and fraction of sessions that fell back to SPK-only (OPK exhausted).

**Golden alerts:**
- Message delivery p99 > 1 s (SLO breach risk).
- Any gateway server at > 80% of its WebSocket capacity.
- Kafka consumer lag > 100K messages on any inbox topic partition (fan-out is falling behind).
- Cassandra write error rate > 0.01% (durability risk; every write error = a SENT receipt that should not have been issued).
- Push notification error rate > 5% (offline users are missing messages).
- Redis memory usage > 80% on any node (routing + presence degradation risk).
- OPK depletion rate: > 20% of key-fetch requests returning SPK-only fallback.

**Distributed tracing:** Propagate a `trace_id` from the client's send event through gateway → message service → Cassandra → Kafka → fan-out service → recipient gateway. This 5–6 hop trace is essential for diagnosing tail latency: a single slow Cassandra write can push the SENT receipt from 20 ms to 200 ms and it must be identifiable without sampling every request.

**Dashboards:**
- WebSocket connections per server and per region (with capacity ceiling line).
- Message throughput (send/s, deliver/s) broken down by 1-on-1 vs. group.
- Fan-out amplification ratio: delivery tasks issued / messages sent (tracks group activity).
- Kafka topic lag heatmap across all inbox partitions.
- Cassandra compaction queue depth and SSTable count (TWCS health).
- E2EE key-fetch success rate and OPK inventory level per user population.

**Logging:** Sample message-metadata events (sender device type, message type, group size) at 1% for traffic analysis — **never log ciphertext or message content**. Log all key-fetch requests at 100% (low volume; anomaly detection for bulk key enumeration which could indicate a harvesting attack). Log all gateway connection events (connect / disconnect / reason) at 100% — connection events are low-volume relative to message throughput and are essential for debugging session-affinity issues.

## Concepts Used

- {{% relref "/design-concepts/specialized/websocket-at-scale" %}} — persistent WebSocket connections, gateway fleet sizing, connection registry, TTL-based routing
- {{% relref "/design-concepts/storage/wide-column-stores" %}} — Cassandra partition/clustering key design for messages and inbox, TWCS compaction
- {{% relref "/design-concepts/messaging/kafka" %}} — delivery bus, fan-out topics, per-user inbox topics, consumer groups at 3.4M writes/s
- {{% relref "/design-concepts/specialized/notification-fanout" %}} — group fan-out trade-offs, push vs. pull threshold, push notification pipeline
- {{% relref "/design-concepts/specialized/presence-system" %}} — TTL-based presence, Redis pub/sub subscribe-on-demand, debounce
- {{% relref "/design-concepts/storage/object-storage" %}} — media offload, pre-signed upload URLs, CDN delivery
- {{% relref "/design-concepts/networking/realtime-transport" %}} — WebSocket persistent connection characteristics, L4 vs. L7 load balancing
- {{% relref "/design-concepts/security/jwt" %}} — WebSocket session authentication tokens in the upgrade handshake
- {{% relref "/design-concepts/distributed/idempotency" %}} — client_msg_id deduplication, at-least-once with dedup at every layer
- {{% relref "/design-concepts/storage/key-value-stores" %}} — Redis as connection registry and presence store
