---
title: Wrap-Up
weight: 4
type: docs
---

## Interview Tips

Open with the evidence contract:

> "I would separate document preparation from question answering. Every answer must use current, permitted, applicable evidence. I would start with a simple read-only RAG workflow, validate source references, and buffer responses for a final authorization check."

Then drive the interview toward the difficult parts rather than listing model vendors.

| Time in a 40-minute round | Focus |
|--------------------------|-------|
| First 5 minutes | Scope, caller identity, supported questions, confidentiality |
| Next 5 minutes | Traffic, tokens, storage, and provider quota arithmetic |
| Next 10 minutes | Ingestion path and the first complete answer path |
| Next 15 minutes | Retrieval quality, atomic version activation, and release authorization |
| Final 5 minutes | Outages, evaluation, cost, and trade-offs |

Useful talking points:

- **A vector database is optional.** Retrieval can begin with keyword search or database vector support.
- **RAG is not training.** Current policy arrives as evidence during inference.
- **Access is not a prompt instruction.** The retrieval and release boundaries enforce it.
- **Citations are not proof by themselves.** Check actual source support.
- **Average QPS hides model capacity.** Include tokens and in-flight requests.
- **A cache is another evidence consumer.** Reuse needs current dependencies.
- **Do not add an agent to a fixed workflow.** This case has no autonomous write task.

## Resiliency

| Failure | Protection | User-visible behavior |
|---------|------------|-----------------------|
| Application instance crash | Stateless API tier and durable request state | Retry/reconcile the same logical request |
| Worker crash | Durable queue, lease, deterministic IDs | Processing resumes or reports failure |
| Database/queue dual-write gap | Transactional outbox | Accepted upload is not silently lost |
| Poison document | Bounded retries and dead-letter handling | Administrator sees a failed job |
| Authorization authority unavailable | Fail closed | Explicit temporary error |
| Model rate limit/outage | Admission limits, bounded retries, circuit breaker | Clear retryable service error |
| Source changed during generation | Final consistent source check | No stale answer release from that snapshot |
| Search copy lags | Authoritative source eligibility check | Skip old data or report not-ready state |
| Optional cache unavailable | Run normal path within capacity limits | Slower response or explicit overload |

Avoid "fallbacks" that invent policy answers. An approved alternate model must pass quality and data-handling requirements before it becomes an outage path.

### Disaster Recovery

- Back up authoritative metadata and original files with tested restoration.
- Preserve compatible source versions and preparation configurations.
- Search indexes can be rebuilt from originals and version metadata, but rebuilding takes time and embedding capacity.
- Keep authorization and deletion state authoritative during recovery; do not restore an old index and serve it without current eligibility checks.
- Define recovery point and recovery time objectives with the business. Backups and replication do not eliminate every data-loss scenario.

## Observability

### Service-Level Indicators

- Eligible-request completion within the 12-second deadline.
- p50/p99 time to validated answer.
- Model and embedding quota utilization.
- Source preparation/activation lag.
- Provider failure, refusal, and incomplete-output rates.
- Source-change and final-authorization rejection rates.
- Cost per successful task, including failed attempts.

### Quality Indicators

- Sufficient-evidence recall by question slice.
- Correctness and support on reviewed samples.
- Citation accuracy and coverage.
- Unnecessary refusal and clarification rates.
- Cross-tenant/forbidden-source test failures as release blockers.

Do not treat a low HTTP error rate as answer correctness. A `200` response with an invented policy is still a product failure.

### Tracing and Safe Logging

Propagate a request ID through admission, retrieval, generation, and validation. Record versions, timings, usage, safe source identifiers, and reason codes.

Do not put unrestricted prompts, source excerpts, employee details, or raw conversation content into default logs. Restrict trace access, bound retention, and avoid sensitive/high-cardinality metric labels.

### Alerts

| Signal | Possible explanation |
|--------|----------------------|
| Empty evidence rises | Ingestion breakage, filter bug, or new unsupported questions |
| Active-version mismatch rises | Search lag or lifecycle regression |
| Provider retries rise | Quota pressure or outage |
| Input tokens rise | Redundant retrieval or history growth |
| Validation errors rise after release | Prompt/schema/model incompatibility |
| Contractor slice quality falls | Applicability or retrieval regression |

## Concepts Used

- [LLM fundamentals]({{% relref "/design-concepts/ml/generative-ai-fundamentals" %}}): fluent output is not evidence.
- [Tokens and context]({{% relref "/design-concepts/ml/tokens-context-and-sampling" %}}): explicit input/output budgets.
- [Prompt design]({{% relref "/design-concepts/ml/prompt-design-and-structured-outputs" %}}): typed response states and validation.
- [Embeddings]({{% relref "/design-concepts/ml/embeddings-and-semantic-search" %}}) and [vector search]({{% relref "/design-concepts/ml/vector-search-and-databases" %}}): compatible representation and measured indexes.
- [RAG]({{% relref "/design-concepts/ml/retrieval-augmented-generation" %}}): external evidence at request time.
- [Ingestion and freshness]({{% relref "/design-concepts/ml/rag-ingestion-and-retrieval" %}}): versioned preparation and retirement.
- [Prompting vs tuning]({{% relref "/design-concepts/ml/prompting-vs-rag-vs-fine-tuning" %}}): choose the lever matching the failure.
- [Evaluation]({{% relref "/design-concepts/ml/generative-ai-evaluation" %}}): component metrics and release gates.
- [Safety]({{% relref "/design-concepts/ml/generative-ai-safety" %}}): independent authorization and data minimization.
- [Serving]({{% relref "/design-concepts/ml/llm-serving-latency-and-cost" %}}): quotas, latency, cost, and buffering.
- [Idempotency]({{% relref "/design-concepts/distributed/idempotency" %}}), [outbox]({{% relref "/design-concepts/distributed/outbox-pattern" %}}), and [retry handling]({{% relref "/design-concepts/messaging/dlq-and-retry" %}}): durable processing.

## Final Trade-offs

| Decision | Why chosen | What we give up |
|----------|------------|-----------------|
| Read-only fixed workflow | Smaller, testable control surface | Autonomous task execution |
| Existing database plus search indexes | Fewer synchronization boundaries | Potential shared-resource contention |
| Immutable versions and activation | Complete reader-visible evidence | Extra storage and lifecycle work |
| Authoritative access checks | Defined revocation semantics | Availability depends on authorization |
| Buffered answer release | Final output/source checks | Immediate answer-token streaming |
| No answer cache initially | Simpler correctness baseline | Higher model-call volume |
| Managed model initially if compliant | Lower initial serving ownership | Provider dependency and quota constraints |

## Test Your Understanding

These questions combine architecture, arithmetic, and failure reasoning. Try drawing the relevant request or lifecycle timeline before expanding the answer.

{{< details title="1. Why does this design have separate ingestion and question paths?" closed="true" >}}
Document extraction and embedding happen when sources change; retrieval and generation happen for each question. Separating them avoids repeatedly processing whole documents and allows independent quotas, retries, and freshness tracking.
{{< /details >}}

{{< details title="2. How do 20,000 daily users produce the 30-QPS peak estimate?" closed="true" >}}
20,000 * 10 = 200,000 questions/day. Dividing by 86,400 gives about 2.31/s. Applying the assumed 12x peak factor gives 27.8/s, rounded upward to 30. The factor must be validated against real traffic.
{{< /details >}}

{{< details title="3. Why is five million vectors not automatically a sharding requirement?" closed="true" >}}
Capacity depends on dimensions, indexes, filters, hardware, and load. The raw vectors are 15.36 GB under the stated assumptions. Benchmark a simpler deployment before adding distributed coordination.
{{< /details >}}

{{< details title="4. Does 15.36 GB describe total database storage?" closed="true" >}}
No. It excludes text, metadata, graph/index structures, retained versions, replicas, logs, backups, and rebuild headroom.
{{< /details >}}

{{< details title="5. Why does 30 requests/s also require a token-quota estimate?" closed="true" >}}
At 2,400 average input-plus-output tokens, peak traffic is about 4.32 million tokens/minute. A request quota alone does not establish that the provider can serve the workload.
{{< /details >}}

{{< details title="6. What is the assumed average number of in-flight requests at peak?" closed="true" >}}
30 requests/s * 4 seconds average duration = 120. This uses a planning mean, not p50, and does not establish tail-latency capacity.
{{< /details >}}

{{< details title="7. The upload returns 202. Can the employee immediately rely on the policy being searchable?" closed="true" >}}
No. The upload is accepted, but preparation and activation are asynchronous. Expose status and retain the prior active version until the replacement is ready.
{{< /details >}}

{{< details title="8. What prevents a database write succeeding while its queue event is lost?" closed="true" >}}
Write an outbox event in the same database transaction as the version/job metadata. A publisher retries delivery, and the worker processes the event idempotently.
{{< /details >}}

{{< details title="9. Why cannot a delayed version-2 worker replace active version 3?" closed="true" >}}
Activation checks the current requested version, row revision, and tombstone in an authoritative transaction. A stale job becomes superseded rather than moving the active pointer backward.
{{< /details >}}

{{< details title="10. Can physical index cleanup be asynchronous after deletion?" closed="true" >}}
Yes, if authoritative source eligibility blocks future use immediately under the defined boundary. Physical cleanup and retention still need an explicit tracked process.
{{< /details >}}

{{< details title="11. Why is a copied ACL in the vector index insufficient?" closed="true" >}}
It can lag permission updates. Current source authorization must be established before downstream text use and again at the defined release point.
{{< /details >}}

{{< details title="12. A permission is revoked during model generation. What happens?" closed="true" >}}
If revocation precedes the final authorization snapshot, discard the buffered answer and report the source change or access failure. Do not release text from the now-ineligible source.
{{< /details >}}

{{< details title="13. What if revocation occurs after the final authorization snapshot?" closed="true" >}}
The response may already be authorized and in flight. The design does not claim atomicity between database state and bytes reaching the user, nor retroactive revocation.
{{< /details >}}

{{< details title="14. Why buffer answer text instead of streaming immediately?" closed="true" >}}
The chosen contract requires final source and output checks before answer release. Immediate streaming can disclose text that later fails validation or authorization.
{{< /details >}}

{{< details title="15. Can the UI still show progress while buffering?" closed="true" >}}
Yes. It can show safe application-generated progress events such as searching or validating. These are different from releasing unvalidated model answer content.
{{< /details >}}

{{< details title="16. The model cites a real document but invents a carryover rule. Which checks matter?" closed="true" >}}
Source existence and access pass, but claim support and citation coverage fail. A real citation does not support every statement attached to it.
{{< /details >}}

{{< details title="17. Why return a service error rather than unsupported during a provider outage?" closed="true" >}}
Unsupported describes missing evidence. An outage is an operational failure and must remain visible to users, retries, availability metrics, and alerts.
{{< /details >}}

{{< details title="18. The question is identical but the employee's region changed. Can the cache hit be reused?" closed="true" >}}
Not without checking current applicability and source dependencies. Exact question text is not sufficient identity for a policy answer.
{{< /details >}}

{{< details title="19. Why must idempotency replay recheck source eligibility?" closed="true" >}}
A previously valid result can become stale or forbidden. Idempotency avoids duplicate logical operations; it does not override current access and freshness requirements.
{{< /details >}}

{{< details title="20. Can application idempotency guarantee exactly-once model billing after a crash?" closed="true" >}}
Not without compatible provider semantics. A prior call may have completed with its response lost. Record provider identifiers, reconcile where possible, and bound any duplicate inference cost.
{{< /details >}}

{{< details title="21. Why not send all top candidates to an external reranker before permission filtering?" closed="true" >}}
That would disclose forbidden source text to the reranker. The authorization boundary must precede every downstream text processor, not only the final generator.
{{< /details >}}

{{< details title="22. A perfect reranker cannot find the missing exception. Why?" closed="true" >}}
The exception may not be in the candidate set, may have been lost during extraction, or may have been separated from its rule. Reranking cannot recover absent evidence.
{{< /details >}}

{{< details title="23. Why is generated conversation history not authoritative policy evidence?" closed="true" >}}
It can contain prior hallucinations, stale source summaries, and revoked content. Revalidate dependencies and retrieve current sources rather than treating earlier prose as truth.
{{< /details >}}

{{< details title="24. What is wrong with using stale token group claims for immediate revocation?" closed="true" >}}
The claims can continue granting membership until token expiry. Current membership/permission state must support the promised revocation boundary.
{{< /details >}}

{{< details title="25. How can two incompatible embedding models silently break a working query?" closed="true" >}}
Matching vector dimensions do not imply matching coordinate spaces. Switch the query encoder and prepared index as a compatible versioned pair.
{{< /details >}}

{{< details title="26. Does the hypothetical $720/day include all system costs?" closed="true" >}}
No. It covers average answer-model usage at the stated rates. Add embeddings, retries, discarded generations, search, storage, network, review, and operations.
{{< /details >}}

{{< details title="27. Why do the average 2,000 input tokens and hard 6,000-token cap coexist?" closed="true" >}}
One is a workload estimate for capacity/cost; the other is a per-request boundary. Averages do not replace hard limits, and long-tail requests can cost more.
{{< /details >}}

{{< details title="28. Is passing 95% answer correctness enough if one case leaks another tenant's policy?" closed="true" >}}
No. Cross-tenant exposure is a separate release blocker. It must not be averaged into an otherwise good quality score.
{{< /details >}}

{{< details title="29. What is the safest fallback when current permission cannot be established?" closed="true" >}}
Fail closed with an explicit temporary service error. Do not reuse an old answer or assume copied index metadata is current.
{{< /details >}}

{{< details title="30. What single sentence best summarizes this architecture?" closed="true" >}}
Prepare versioned evidence once, retrieve only current permitted sources for each request, generate within a bounded budget, and release only after explicit output and source checks.
{{< /details >}}
