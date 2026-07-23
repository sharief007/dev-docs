---
title: 'Wrap-Up'
weight: 5
type: docs
---

## Interview Tips

- **Open with the three sub-problems.** A search engine is really three systems bolted together: a crawler, an indexing pipeline, and a query serving stack. State this up front — it frames the rest of the discussion and signals that you see the full scope.
- **Anchor on the inverted index immediately.** The moment you say "inverted index," the interviewer knows you understand the core data structure. Sketch it: `term → [doc_id, freq, positions]`. Then explain *why* it beats a full-table scan (O(|posting list|) vs O(N docs)).
- **Drive the BM25 explanation at the right depth.** You don't need to derive the formula from scratch — name it, state what it corrects over naive TF (saturation + length normalisation), and show the formula. That's the senior signal.
- **Explain PageRank inline — no page exists.** State the random-surfer model in one sentence, give the iterative formula `PR(A) = (1−d)/N + d × Σ PR(B)/L(B)`, and mention it's a daily Spark batch job over the link graph. Interviewers frequently ask "how do you prevent spam from gaming PageRank?" — answer: damping factor (d=0.85) limits rank propagation depth; link-spam detection is a separate offline classifier.
- **Discuss both shard strategies.** Bring up document-partitioning vs. term-partitioning, give the trade-offs, and conclude that document-partitioning is the production choice. Knowing both demonstrates depth.
- **Volunteer the tiered index freshness design.** Near-real-time indexing (hot segment in RAM + warm/cold segments on disk, LSM-style merge) is a strong distinguisher — few candidates go beyond "rebuild the index periodically."
- **Common follow-up questions to rehearse:**
  - "How do you handle a sudden spike in a single query?" → L1 + L2 cache; count-min sketch for detection; short TTL for breaking-news queries.
  - "What about personalised search?" → Out of scope here; would add a learning-to-rank layer above BM25 + PageRank using user-specific features.
  - "How do you deal with web spam / SEO manipulation?" → PageRank damping, trust-rank variants, manual penalties for known spam domains.
  - "How do you keep posting lists small for stop-words?" → Stop-words are removed during tokenisation; they never enter the index.

## Resiliency

- **Crawler failure:** Individual crawler nodes fail silently — their URLs return to the frontier after a timeout. The frontier is itself persisted in a durable queue (Kafka with replication) so a full crawler restart doesn't lose pending URLs.
- **Indexing pipeline crash:** Kafka acts as the durable buffer between crawler and indexing workers. The consumer offset is committed only after a segment is successfully flushed. A crashed pipeline worker re-reads from its last committed offset — no re-crawl needed.
- **Index shard failure:** Every shard has 2 replicas. If the primary fails, reads are automatically served from a replica. A new replica syncs from the surviving replica via segment file copy — no re-indexing required. See {{% relref "/design-concepts/replication/leader-based-replication" %}}.
- **Query node failure:** Stateless query-serving nodes are behind a load balancer; failed nodes are removed from rotation within one health-check interval (typically 5–10 s).
- **Partial shard unavailability:** the query processor uses **deadline-propagated scatter**: if a shard doesn't respond within 25 ms, results are returned from available shards with a `"partial": true` flag. Degraded-but-available is better than a full timeout.
- **Cache stampede on popular results:** single-flight request coalescing at the L2 Redis layer ensures only one backend request is issued for a given cache key even during a miss storm. See {{% relref "/design-concepts/storage/caching-patterns" %}}.
- **Crawler politeness and bans:** per-domain rate limiting (1 req/s default, respecting `Crawl-delay`) prevents the crawler from being blocked. IP rotation across egress pools reduces ban risk for large deployments.

## Observability

**SLIs (Service-Level Indicators):**

| Signal | Measurement | Target |
|---|---|---|
| Query latency p50 | End-to-end from QFE | < 100 ms |
| Query latency p99 | End-to-end from QFE | < 200 ms |
| Query availability | Non-5xx / total requests | 99.99% |
| Cache hit ratio (L1 + L2) | Cache hits / total queries | > 80% for L2 |
| Index freshness lag | Time from publish to first appearance in results | < 4 hours |
| Crawl success rate | HTTP 2xx / total fetches | > 95% |
| Indexing pipeline lag | Kafka consumer lag in bytes | < 100 GB |

**Alerting:**
- Query p99 > 150 ms for > 2 min → page on-call (approaching SLO breach).
- Cache hit ratio drops below 70% → possible cache eviction storm or cold-start event.
- Any shard returns > 1% 5xx errors → possible shard node failure; trigger replica promotion.
- Kafka consumer lag > 500 GB → indexing pipeline is falling behind; auto-scale workers.
- PageRank batch job fails → alert; last successful scores remain in production; job retried.

**Distributed Tracing:**
Propagate a trace ID from the user request through: QFE → spell correction → scatter → each shard response → snippet fetch → response assembly. This isolates tail latency to the specific shard or stage responsible. A 99th percentile trace typically reveals one slow shard response or a cold-cache forward-index lookup.

**Dashboards:**
- QPS by region, latency percentiles (p50/p90/p99), error rate — the "golden signals" board.
- Cache hit rate over time; L1 vs L2 breakdown.
- Per-shard posting list fetch latency distribution (detects hot shards).
- Indexing pipeline throughput: docs/s indexed, Kafka lag, segment flush rate.
- Crawl coverage: pages crawled today vs. target; per-domain crawl success rate.
- PageRank distribution histogram (sanity check after each batch run).

**Logging:**
- Sample query logs at 0.1% for offline relevance analysis (full logging at 100K QPS would be ~50 GB/s).
- Log all indexing pipeline errors (parse failures, oversized documents, encoding errors) at 100%.
- Structured JSON logs with trace ID, shard ID, latency breakdown per stage.

## Related Concepts

- {{% relref "/design-concepts/storage/full-text-search" %}} — inverted index fundamentals and Lucene internals
- {{% relref "/design-concepts/storage/bloom-filters" %}} — URL and content-hash deduplication in the crawler
- {{% relref "/design-concepts/storage/consistent-hashing" %}} — distributing documents across index shards
- {{% relref "/design-concepts/messaging/kafka" %}} — durable decoupling of crawl from indexing pipeline
- {{% relref "/design-concepts/scaling/sharding" %}} — document-partitioning vs. term-partitioning trade-offs
- {{% relref "/design-concepts/storage/lsm-trees" %}} — the tiered segment model for near-real-time index freshness
- {{% relref "/design-concepts/storage/caching-patterns" %}} — L1 in-proc + L2 Redis result caching; cache stampede prevention
- {{% relref "/design-concepts/storage/object-storage" %}} — archiving raw HTML and link-graph edge files
- {{% relref "/design-concepts/data/batch-vs-streaming" %}} — crawl/indexing as streaming; PageRank as daily batch
- {{% relref "/design-concepts/storage/hotspot-problems" %}} — hot shard mitigation in term-partitioned indices
- {{% relref "/design-concepts/specialized/typeahead" %}} — FST-based query autocomplete implementation
- {{% relref "/design-concepts/api/pagination" %}} — cursor-based pagination for search result pages
- {{% relref "/design-concepts/replication/leader-based-replication" %}} — index shard replication and failover
