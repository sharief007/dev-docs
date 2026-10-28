---
title: RAG Ingestion, Retrieval Quality & Freshness
weight: 10
type: docs
---

The ExampleCo assistant answers ordinary questions well but misses a carryover exception. The exception was split from the rule during ingestion. Later, HR updates the policy from version 2 to version 3, but a delayed worker reactivates version 2. Finally, a deleted document still appears through the answer cache.

These are not primarily language-model failures. **RAG quality depends on how evidence is extracted, split, indexed, updated, filtered, and retired.**

## Treat Ingestion as a Data Pipeline

```mermaid
flowchart LR
    U["Approved upload"] --> R["Store immutable document version"]
    R --> E["Extract text and structure"]
    E --> C["Chunk with source locations"]
    C --> B["Embed and build keyword data"]
    B --> V["Validate completeness and quality"]
    V --> A["Activate version atomically"]
    A --> S["Serve current permitted evidence"]
```

An upload succeeding is not the same as the document becoming searchable. Expose states such as `uploaded`, `processing`, `ready`, and `failed`; keep activation as a deliberate step.

## Extraction Errors Become Answer Errors

Check the material before choosing a chunk size:

| Source feature | Typical extraction failure | Consequence |
|----------------|----------------------------|-------------|
| Multi-column PDF | Columns interleaved | Conditions attach to the wrong rule |
| Table | Headers separated from rows | Region A value attributed to Region B |
| Scanned page | Text recognition reads 20 as 28 | Wrong numerical evidence |
| Footnote | Exception lost | Overgeneralized answer |
| Repeated page header | Header embedded into every chunk | Noisy retrieval |

Store enough location information to inspect the original: document version, section, page, or paragraph offsets. Do not silently index an empty extraction as a successful document.

## Chunking: Keep Useful Meaning Together

A **chunk** is a searchable passage. It should be small enough to retrieve precisely but large enough to preserve meaning.

```text
Bad split:
Chunk 1: "Unused leave may be carried forward."
Chunk 2: "Only up to five days, with approval, before March 31."

Better:
Chunk: "Unused leave may be carried forward, up to five days,
        with approval, and must be used before March 31."
```

The dates and numbers here are fictional examples.

| Strategy | Advantage | Risk |
|----------|-----------|------|
| Fixed token count | Simple and predictable size | Cuts across rules and exceptions |
| Paragraph/section boundaries | Preserves document organization | Sections vary greatly in size |
| Limited overlap | Carries nearby context across boundaries | Duplicates evidence and inflates counts |
| Parent-child retrieval | Search small chunks, load their larger parent | Parent context adds size and authorization work |

Start with document-aware boundaries and a bounded size. Treat a size such as 400-600 tokens as an **experiment**, not a universal optimum.

Preserve the heading in the chunk or metadata. "20 days" means little if "Region A full-time staff" was discarded.

## Stable Identity and Idempotent Jobs

A useful chunk identity is derived from:

```text
tenant + document_id + document_version + chunker_version + chunk_ordinal
```

Use a collision-resistant encoding or hash where appropriate. Store a content hash separately if you need change detection.

If a queue retries the same job, it should upsert the same versioned chunks rather than create duplicates. Reuse [idempotency]({{% relref "/design-concepts/distributed/idempotency" %}}) and [retry/DLQ handling]({{% relref "/design-concepts/messaging/dlq-and-retry" %}}).

A worker must check that its document version is still eligible before activation. A stale version-2 job must not overwrite an active version-3 pointer.

## Candidate Retrieval vs Final Context

Use different names for different sets:

```text
50 keyword candidates + 50 vector candidates
-> merge and deduplicate
-> authorize current sources before downstream text processing
-> rerank a bounded candidate set
-> choose perhaps 5 complementary passages
-> pack within the context budget
```

These counts are illustrative. They should be selected using evaluation and latency measurements.

**Top-k** is the number of results requested at a particular stage. A vector top-50, reranker top-10, and prompt containing 5 passages are not contradictory settings.

## Hybrid Search and Rank Fusion

Keyword and vector search are complementary. To combine ranked lists without adding incomparable raw scores, one option is **reciprocal rank fusion (RRF)**:

```text
RRF(document) = sum over result lists of 1 / (c + rank)
```

If `c = 60`, a passage ranked first in one list contributes `1/61`. A passage in both lists receives both contributions. A missing passage contributes zero from that list.

This is a ranking heuristic, not a probability. The constant and candidate depths still need testing.

## Reranking: Spend More Work on Fewer Candidates

A **reranker** examines a query and candidate passage together to estimate relevance. It can notice relationships that a single precomputed embedding misses.

- Retrieval cheaply narrows a large corpus.
- Reranking spends more computation on a small shortlist.
- Context selection removes duplication and preserves evidence coverage.

A reranker cannot recover a passage that never entered its candidate set. It also does not grant authorization or prove factual support.

## Make Updates Atomic From the Reader's Perspective

Use immutable document versions and an authoritative active-version pointer.

```text
PSEUDOCODE
job = load_ingestion_job()
require_document_not_deleted(job.document_id)
chunks = extract_and_chunk(job.immutable_version)
upsert_chunks_and_indexes(job.version, chunks)
verify_expected_chunks_and_index_readiness(job.version)

transaction:
    current = lock_document_state(job.document_id)
    if current.deleted or current.requested_version != job.version:
        mark_job_superseded()
    else:
        activate(job.version)
        increment_source_epoch()

enqueue_obsolete_chunk_and_cache_cleanup()
```

The transaction cannot magically span every external index. The **authoritative pointer** determines which prepared version is eligible; index readiness must be verified before switching it.

If an external index temporarily returns an old version, the retrieval service filters it against authoritative state and may return an explicit "not ready" or no-evidence result rather than using retired text.

## Deletion and Permission Revocation

Separate **blocking use** from **physical cleanup**:

1. Mark the source deleted or revoke access in authoritative state.
2. Make new source-use checks reject it.
3. Invalidate dependent search and answer caches.
4. Remove vectors, text derivatives, and obsolete index records asynchronously.
5. Handle originals, backups, logs, and conversation state according to the retention policy.

Recheck selected source eligibility before returning a buffered answer. A revocation after bytes have already been delivered cannot retroactively make the user forget them; define the exact access-consistency boundary.

## Scheduled Refresh vs Change Data Capture

| Method | Benefit | Limitation |
|--------|---------|------------|
| Scheduled full refresh | Simple for small, infrequently changing collections | Freshness delay and repeated work |
| Incremental polling | Less work for changed sources | Requires reliable change/version detection |
| Change data capture (CDC) | Low-latency source-change events | Ordering, duplicate events, and operational complexity |

See [CDC]({{% relref "/design-concepts/data/change-data-capture" %}}). Regardless of event delivery, current permissions must not depend solely on an eventually consistent index copy.

## Test the Complete Lifecycle

| Test | Expected observation |
|------|----------------------|
| Upload version 1 | Not searchable until preparation succeeds |
| Ask a supported question | Correct applicable passage and citation |
| Publish version 2 with a different value | New requests use version 2 after activation |
| Retry version-1 ingestion late | Version 1 cannot reactivate |
| Revoke one employee's access | Future authorized-use checks reject the source |
| Delete document | No future answer or cache reuse depends on it |
| Change embedding model | Queries switch only with a compatible prepared index |

Track extraction failures, job retries, activation lag, active-version mismatch, and evidence recall. A low model-error rate says nothing about an ingestion pipeline that quietly dropped documents.

## Related Concepts

- [Vector search]({{% relref "/design-concepts/ml/vector-search-and-databases" %}}) covers index and filter behavior.
- [RAG]({{% relref "/design-concepts/ml/retrieval-augmented-generation" %}}) separates ingestion from answering.
- [Evaluation]({{% relref "/design-concepts/ml/generative-ai-evaluation" %}}) measures evidence and answer quality.

## Trade-offs

| Choice | Helps | Costs or risks |
|--------|-------|----------------|
| Small chunks | Precise retrieval | Missing surrounding conditions |
| Large chunks | More local context | Noise and token cost |
| Overlap | Preserves boundary information | Duplicates and extra storage |
| Hybrid retrieval | Combines exact and semantic matching | More tuning |
| Reranking | Better shortlist ordering | Extra processing and latency |
| Immutable versions with activation | Reader-visible consistency | More lifecycle state and cleanup |

## Test Your Understanding

{{< details title="1. The upload API returned success. Why is the document not searchable yet?" closed="true" >}}
Upload can mean durable receipt, not completed extraction, embedding, indexing, or activation. Expose processing state rather than treating every accepted upload as immediately queryable.
{{< /details >}}

{{< details title="2. Why can a smaller chunk make a correct rule dangerous?" closed="true" >}}
It can separate the rule from its exception, scope, or deadline. A retrieved sentence may become misleading when its qualifying context is missing.
{{< /details >}}

{{< details title="3. Does overlap solve every chunk-boundary problem?" closed="true" >}}
No. It only repeats nearby content within a chosen window. Tables, long-distance references, and widely separated exceptions can still require structure-aware processing or parent retrieval.
{{< /details >}}

{{< details title="4. A reranker is excellent, but the needed policy is candidate 500. Will reranking the top 50 help?" closed="true" >}}
Not if the policy never enters those 50. Candidate-generation recall bounds what reranking can recover.
{{< /details >}}

{{< details title="5. Why not add raw keyword and cosine scores?" closed="true" >}}
They may use unrelated scales. One can dominate for accidental numerical reasons. Calibrate them or use a tested rank-fusion method such as RRF.
{{< /details >}}

{{< details title="6. Does an RRF score represent factual confidence?" closed="true" >}}
No. It combines ranked-list positions. It does not establish support, policy applicability, or correctness.
{{< /details >}}

{{< details title="7. Why is a deterministic chunk ID useful with at-least-once delivery?" closed="true" >}}
A retry can upsert the same versioned chunk instead of creating duplicates. Include enough version information to avoid overwriting a different document representation.
{{< /details >}}

{{< details title="8. Version 3 is active when a delayed version-2 job finishes. What must prevent regression?" closed="true" >}}
The activation transaction must compare the job's version with the currently requested version and deletion state. A stale worker should mark itself superseded, not move the active pointer backward.
{{< /details >}}

{{< details title="9. Can deleting vectors alone satisfy document deletion?" closed="true" >}}
No. Text derivatives, originals, keyword indexes, caches, histories, logs, and backups may also contain source information. Apply the documented retention and deletion policy across those surfaces.
{{< /details >}}

{{< details title="10. Why authorize before sending candidates to a hosted reranker?" closed="true" >}}
The reranker receives the source text too. It is a downstream data processor and must not receive documents the request is not allowed to use.
{{< /details >}}

{{< details title="11. What if only half the chunks reached an external index?" closed="true" >}}
Do not activate a version that requires completeness. Verify expected chunks and index visibility, or mark the version unavailable with a clear error while recovery proceeds.
{{< /details >}}

{{< details title="12. Why keep page or section locations after extracting plain text?" closed="true" >}}
They let humans inspect the original evidence, resolve precise citations, and diagnose extraction errors such as a misplaced table value.
{{< /details >}}

{{< details title="13. Is a 500-token chunk universally optimal?" closed="true" >}}
No. Document structure, model behavior, question type, and latency budget differ. Compare a few sensible configurations on the same evaluation set.
{{< /details >}}

{{< details title="14. What happens if an embedding upgrade reuses old vectors?" closed="true" >}}
Queries can be compared against an incompatible coordinate space. Prepare and validate a new matched index, then switch the query encoder and index together.
{{< /details >}}

{{< details title="15. Why can duplicate overlapping chunks inflate retrieval metrics?" closed="true" >}}
Several near-identical passages may appear as multiple relevant hits without adding evidence coverage. Deduplicate and define relevance labels so repeated text does not masquerade as broader recall.
{{< /details >}}

{{< details title="16. A permission event is delayed. Must the assistant wait for the index update?" closed="true" >}}
No. Current authorization should be checked against authoritative state before source use and response release. Index metadata alone should not establish sensitive access.
{{< /details >}}

{{< details title="17. Can a background cleanup job guarantee immediate physical erasure everywhere?" closed="true" >}}
No. Replicas, backups, and other retained systems have their own timelines. Distinguish immediate blocking of future use from the documented physical-deletion process.
{{< /details >}}

{{< details title="18. Why does reranking all two million chunks defeat the architecture?" closed="true" >}}
Reranking is usually more expensive per passage than initial retrieval. Its role is to improve a bounded shortlist, not replace efficient candidate generation for the full corpus.
{{< /details >}}

{{< details title="19. A source is revoked after its answer was already displayed. Can the system undo that disclosure?" closed="true" >}}
No. It can stop future access, reuse, and retrieval, but cannot retract knowledge already delivered. Define and enforce the authorization boundary before release.
{{< /details >}}

{{< details title="20. What lifecycle test catches more than a static question-answer test?" closed="true" >}}
Upload a policy, activate it, answer from it, change its value, retry an old job, revoke access, and delete it. Confirm each later query and cache lookup follows current authoritative state.
{{< /details >}}
