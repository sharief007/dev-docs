---
title: RAG Document Assistant
weight: 1
type: docs
---

An ExampleCo employee asks, "How many days of annual leave do I get?" The company has thousands of policy documents, different rules for different regions, and confidential documents that not every employee may read. A fluent answer is not enough: it must use the right current policy, show supporting sources, and respect access rules.

We will design a **retrieval-augmented generation (RAG) document assistant**. It finds relevant permitted passages and uses a language model to explain them. It does not train a model from scratch, act as the policy authority, or submit leave requests.

ExampleCo and all policy rules, prices, and capacity numbers in this case study are fictional teaching assumptions.

## Functional Requirements

1. **Ingest approved documents:** An authorized administrator uploads a text-based PDF or Markdown document and sees its processing status.
2. **Version policies:** Prepare a replacement version without exposing a half-indexed document. Activate it only when ready.
3. **Answer questions:** An authenticated employee asks a text question and receives a supported answer, a specific clarification question, or an explicit evidence limitation.
4. **Cite evidence:** Answers refer to the actual supplied document versions and sections through access-checked links.
5. **Enforce isolation:** A request uses only its tenant's documents and the caller's current document permissions.
6. **Apply policy scope:** Use verified employee region and employment type where needed. Do not equate annual entitlement with remaining balance.
7. **Retire evidence:** Document deletion, replacement, and access revocation prevent future use according to the consistency boundary below.
8. **Support short conversations:** Store owner-scoped conversation state for up to seven days; allow earlier user deletion. Revalidate source-derived history before reuse.
9. **Expose honest failures:** Provider outages, failed ingestion, source changes, and invalid generated output are not disguised as successful answers.

## Out of Scope

- Autonomous agents, write tools, and leave-request submission.
- Remaining-balance calculation or integration with payroll.
- Model pre-training or fine-tuning.
- OCR, audio, images, video, and scanned PDFs without usable text.
- Public web crawling and arbitrary URL ingestion.
- Multi-region active-active writes.
- Legal interpretation or unreviewed changes to official policy.

These are intentional boundaries. The [tool-calling]({{% relref "/design-concepts/generative-ai/tool-calling-and-agents" %}}) and [multimodal]({{% relref "/design-concepts/generative-ai/multimodal-generative-ai" %}}) concepts describe possible later extensions, not hidden requirements for version one.

## Non-Functional Requirements

| Area | Target or assumption |
|------|----------------------|
| Users | 100,000 registered employees; 20,000 daily active users |
| Questions | 10 per active user/day: 200,000/day; provision initially for 30 questions/s peak |
| Corpus | 100,000 active documents; average 50 chunks/document |
| Ingestion | 1,000 new documents + 500 replacements/day |
| Upload limits | 10 MB per document; text-based supported formats only |
| Latency | Answer p50 < 4 s and p99 < 10 s under admitted load; 12 s overall request deadline |
| Availability | 99.9% of eligible question requests return a valid result within the deadline |
| Freshness | Prepared document versions become searchable within five minutes at p95 under the supported ingestion load |
| Confidentiality | No source text is sent downstream without current authorization |
| Durability | Accepted uploads and version metadata survive a worker or application crash |
| Conversation retention | Seven days maximum for stored conversation content; deletion can be requested earlier |
| Operational logs | Safe metadata only by default; 30-day illustrative retention |

These are design targets to validate with load tests and evaluation, not achieved benchmark claims.

An eligible request is authenticated, valid, and within its tenant quota. Provider failures, internal overload, and internal deadlines count against availability. A correct `unsupported` result is a valid domain result, but cannot be used to hide operational failures; evidence coverage is measured separately.

A timely `409 source_changed` is also a valid consistency result. It does not count against the 99.9% target. A timeout or authorization outage while revalidating still does. A timely `422 provider_refused` is a classified provider decision, not missing evidence and not an outage. Track refusal rate on the safety dashboard. `502`, `503`, and `504` count against availability.

### The Access-Consistency Boundary

Source checks use authoritative metadata, not a lagging search replica:

- Check permissions and active source versions **before text is sent to a reranker or model**.
- Buffer answer text and check source dependencies again in a final authorization transaction.
- A revocation completed before that final authorization snapshot must block release from the old source.
- A response already authorized before a concurrent revocation may be in flight. Bytes already delivered cannot be retracted.

This is a precise release boundary, not a promise of retroactive revocation or an atomic transaction spanning the database and the user's screen.

The source policy also applies to citations, cached responses, and source-derived conversation history.

## Quality Requirements

Create a reviewed evaluation set before optimizing. Initial illustrative release gates:

- At least **95%** of answerable cases in the fixed set retrieve sufficient applicable evidence.
- At least **95%** of answerable cases produce a correct, supported answer under the agreed rubric.
- No observed cross-tenant or forbidden-source use in the access test suite; any such failure blocks release.
- Every returned citation resolves to supplied evidence and an access-checked route.
- Unanswerable and ambiguous cases must be evaluated separately, rather than counted as ordinary failures.

A passing finite test set does not prove perfect security or establish production statistical confidence. Enforced boundaries and continuous monitoring remain necessary.

## Terminology

| Term | Meaning in this design |
|------|------------------------|
| Document | Logical policy with permissions and an active version |
| Version | Immutable uploaded content and its prepared representation |
| Chunk | Searchable passage tied to a version and source location |
| Evidence | Selected current, permitted chunks supplied for one answer |
| Source epoch | Tenant-level counter changed by relevant source lifecycle events |
| Authorization epoch | Tenant-level counter changed by permission updates |
| Grounded answer | Claims supported by the evidence actually supplied |

The epochs are coarse invalidation tools. They do not replace caller identity or document-level authorization.

## Reading Order

1. [High-Level Design]({{% relref "high-level-design" %}}): estimates, APIs, data model, and the first complete architecture.
2. [Deep Dive]({{% relref "deep-dive" %}}): evolve a naive model call into a current, access-controlled evidence pipeline.
3. [Wrap-Up]({{% relref "wrap-up" %}}): interview framing, resiliency, observability, and 30 tricky questions.
