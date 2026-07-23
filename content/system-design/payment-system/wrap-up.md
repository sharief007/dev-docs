---
title: 'Wrap-Up'
weight: 5
type: docs
---

## Interview Tips

- **Lead with the failure model, not the happy path.** Payment systems live or die by "what happens when X fails mid-transaction." Open by naming the partial failure scenarios — processor timeout, ledger write failure, network partition — and show you already have a structured answer: saga + idempotency + outbox. Interviewers who know payments will immediately respect the framing.

- **Explain 2PC vs Saga before being asked.** Proactively state that two-phase commit cannot span an external processor (it has no rollback API) and would serialise writes under a global lock at 10K TPS. Pivot to saga with compensating transactions. This is the single most-asked payment follow-up; don't wait for it.

- **Sell event sourcing for the ledger in one sentence.** "The ledger is an append-only event log; balance is a derived projection. This gives us point-in-time queries and immutable audit trails for free — both legally required." Offer snapshots only if the interviewer asks about read performance.

- **Name the outbox pattern explicitly.** "We write the event into the outbox table in the same transaction as the DB write, then publish asynchronously" is a tier-1 signal. Many candidates say "publish to Kafka after the DB write" without acknowledging the atomicity gap.

- **Don't forget reconciliation.** Most candidates design the happy path and stop. Mentioning nightly reconciliation as the safety net that catches silent failures — even after all the reliability patterns — demonstrates production experience.

- **Common follow-ups to rehearse:** refund and chargeback flow; multi-currency (FX conversion, currency rounding rules); payout batching (aggregating merchant credits before a single bank transfer); rate limiting and abuse prevention (burst control on payment creation); what if the outbox processor falls behind; how do you test the saga compensation path in CI; hot merchant accounts (a single merchant generating millions of ledger entries per day).

## Resiliency

- **Idempotency at every hop.** Client → API (via `payment_intent_id`), API → processor (same ID forwarded as processor idempotency key), outbox → Kafka (transactional producer), Kafka → consumer (consumer-side idempotency on the `payment_intent_id`). Each hop is independently idempotent, so any hop can be retried safely. See {{% relref "/design-concepts/distributed/idempotency" %}}.

- **Saga compensation for partial failures.** Each step has a defined rollback action. The orchestrator persists saga step progress in `payment_intents.status` so a crash mid-saga resumes from the last committed step on restart, not from the beginning. See {{% relref "/design-concepts/distributed/saga-pattern" %}}.

- **Outbox for reliable event delivery.** Kafka publish is decoupled from the DB commit via the outbox. The outbox processor can lag, crash, and restart without data loss — unpublished rows persist in the DB. See {{% relref "/design-concepts/distributed/outbox-pattern" %}}.

- **Circuit breaker on the fraud service.** Prevents a degraded fraud service from stalling the payment critical path. Open circuit falls back to rule-based heuristics. See {{% relref "/design-concepts/reliability/circuit-breaker" %}}.

- **Processor timeout — query before retry.** A timed-out processor call must be resolved by querying the processor's record for the `payment_intent_id` before retrying. Blindly retrying risks double-charging; blindly failing risks a charged-but-unrecorded transaction.

- **Multi-region.** Payment service nodes are stateless and run in multiple regions. The Postgres primary is in a primary region with synchronous cross-region replicas (RPO < 1 s). Payments write to the primary; balance reads can be served from replicas with a bounded staleness window. Regional failover promotes a replica to primary automatically. See {{% relref "/design-concepts/distributed/multi-region-design" %}} and {{% relref "/design-concepts/replication/leader-based-replication" %}}.

- **Tiered storage.** Hot Postgres data migrates to columnar warm storage after 90 days and to object cold storage after 2 years. This keeps the hot tier manageable (~58 TB) while meeting the 7-year retention requirement. Background migration jobs use soft-delete + copy-then-purge so live queries are unaffected.

- **Reconciliation as the final safety net.** Every reliability mechanism above can fail silently. Nightly reconciliation against processor settlement records catches any discrepancy that escaped all prior guards. Discrepancies above threshold automatically page on-call.

## Observability

- **SLIs:** payment success rate (non-5xx and non-failed-fraud ratio); payment p50/p99 latency from confirmation request to ledger write commit; ledger write latency p99; outbox lag (age of oldest unpublished row); saga compensation rate (% of payments triggering a compensating step); reconciliation discrepancy count per day.

- **Golden alerts:**
  - Saga compensation rate > 0.1% → systemic processor or ledger issue.
  - Outbox lag > 60 s → outbox processor stuck or CDC broken.
  - Reconciliation discrepancies > threshold → potential money loss.
  - Payment error rate > 0.5% for 2 consecutive minutes → broad failure.
  - Fraud circuit breaker open → fraud service degraded.
  - Snapshot staleness per account > 30 min → balance query latency at risk.

- **Distributed tracing.** Every payment carries a `trace_id` propagated from API gateway → saga orchestrator → processor HTTP call → ledger writer → Kafka producer → outbox processor. Tail latency can be pinpointed to a specific step in the chain.

- **Logging.** Log every saga step transition (status change, processor call, ledger write commit) with `payment_intent_id` and `trace_id`. Log all compensation events at `WARN` level with full context. Logs are shipped to an append-only sink (S3 + Athena) for compliance — they must be immutable and retained for 7 years alongside ledger entries.

- **Dashboards.** Payment volume (TPS with 1-minute granularity), success/failure breakdown by merchant and payment method, saga step heatmap (which step fails most often), outbox pipeline lag over time, ledger write throughput vs. snapshot freshness, reconciliation daily discrepancy trend, fraud block rate by rule.

## Concepts Used

- {{% relref "/design-concepts/specialized/payment-systems" %}} — payment flow, processor integration patterns
- {{% relref "/design-concepts/distributed/idempotency" %}} — exactly-once semantics at every service boundary
- {{% relref "/design-concepts/distributed/outbox-pattern" %}} — atomic DB write paired with event publish
- {{% relref "/design-concepts/consensus/two-phase-commit" %}} — why 2PC is unsuitable for cross-service money movement
- {{% relref "/design-concepts/distributed/saga-pattern" %}} — multi-step distributed transaction with compensation
- {{% relref "/design-concepts/messaging/cqrs" %}} — account_snapshots as a CQRS read model derived from the ledger event store
- {{% relref "/design-concepts/messaging/event-driven-architecture" %}} — downstream services consuming payment events asynchronously
- {{% relref "/design-concepts/api/idempotency-and-versioning" %}} — idempotency-key pattern at the API layer
- {{% relref "/design-concepts/distributed/consistency-models" %}} — strong consistency within ledger, eventual correctness at processor trust boundary
- {{% relref "/design-concepts/messaging/kafka" %}} — transactional producer for exactly-once event delivery
- {{% relref "/design-concepts/reliability/circuit-breaker" %}} — isolating the fraud service from the payment critical path
- {{% relref "/design-concepts/reliability/back-pressure" %}} — fraud service overload handling
- {{% relref "/design-concepts/distributed/multi-region-design" %}} — cross-region replication and failover
- {{% relref "/design-concepts/replication/leader-based-replication" %}} — Postgres primary-replica replication model
