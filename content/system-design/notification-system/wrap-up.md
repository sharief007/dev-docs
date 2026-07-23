---
title: 'Wrap-Up'
weight: 4
type: docs
---

## Interview Tips

- **Frame it as fan-out + reliable delivery**, not "send an email." The interesting engineering is preferences, dedup, burst absorption, and provider failure handling — steer there quickly.
- **Lead with at-least-once + idempotent dedup = effectively-once.** Stating this delivery-semantics contract up front shows you understand the core correctness problem. Interviewers probe "what if it's delivered twice?" — have the `SETNX` answer ready.
- **Separate transactional from promotional early.** This single distinction drives priority queues, quiet-hours policy, and rate-cap behavior. It's the highest-signal design decision.
- **Kafka as a shock absorber** is the key insight for fan-out storms — the backlog drains at a provider-safe rate. Say why you don't push 10M sends synchronously.
- **Bulkheads + circuit breaker per provider** is the resiliency talking point. Explain transient-vs-permanent failure classification (retry vs DLQ).
- **Likely follow-ups:** "how do you not spam users?" (preferences + rate caps), "how do you know it was delivered?" (provider webhooks + status log), "how do you handle a provider outage?" (circuit breaker + retry queue + eventual drain), "how do you localize?" (template + locale).
- **Don't over-index on the database** — the queues and policy engine are where points are won.

## Resiliency

- **Durable-before-ack:** an accepted notification is persisted/enqueued before the 202, so a crash never silently drops it.
- **Kafka retention & replay:** downstream outages become backlogs, not data loss; the DLQ enables replay after a fix.
- **Bulkheads:** per-channel queues and worker pools isolate a slow/broken provider from healthy channels. See {{% relref "/design-concepts/reliability/bulkhead" %}}.
- **Circuit breakers + backoff:** stop hammering a failing provider; exponential backoff with jitter prevents synchronized retry storms. See {{% relref "/design-concepts/reliability/circuit-breaker" %}}.
- **Backpressure:** token buckets shape outbound rate; excess waits in Kafka rather than overrunning providers. See {{% relref "/design-concepts/reliability/back-pressure" %}}.
- **Idempotency everywhere:** dedup keys make retries and duplicate events safe.
- **Multi-region:** stateless ingest/processor tiers run active-active; Kafka and stores replicate across regions.

## Observability

- **SLIs:** ingest acceptance latency & error rate; end-to-end delivery latency per priority (transactional p99 < 5 s); per-channel delivery success rate; provider error rate; queue depth / consumer lag per channel; DLQ rate; dedup hit rate.
- **Golden alerts:** transactional delivery p99 breaching SLO, consumer lag growing (drain slower than intake), provider circuit breaker open, DLQ rate spike, dedup store unavailable.
- **Tracing:** propagate the `notification_id` and upstream trace id from ingest → processor → worker → provider so a single notification's journey is reconstructable.
- **Dashboards:** notifications/sec by category & channel, delivery funnel (accepted → sent → delivered → opened), provider health, backlog burn-down during fan-out storms.
- **Auditing:** immutable notification log for support ("why didn't I get notified?") and compliance (opt-out honored).

## Concepts Used

- {{% relref "/design-concepts/messaging/kafka" %}} — buffering backbone, per-user partitioning, replay
- {{% relref "/design-concepts/specialized/notification-fanout" %}} — fan-out strategies
- {{% relref "/design-concepts/messaging/dlq-and-retry" %}} — dead-lettering and retry policy
- {{% relref "/design-concepts/distributed/idempotency" %}} — dedup / effectively-once
- {{% relref "/design-concepts/rate-limiting/algorithms" %}} — per-provider token-bucket shaping
- {{% relref "/design-concepts/reliability/circuit-breaker" %}} & {{% relref "/design-concepts/reliability/bulkhead" %}} — provider failure isolation
- {{% relref "/design-concepts/reliability/back-pressure" %}} — burst absorption
- {{% relref "/design-concepts/storage/wide-column-stores" %}} — the notification log
