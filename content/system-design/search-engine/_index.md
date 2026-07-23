---
title: 'Search Engine'
weight: 1
type: docs
---

Every second, hundreds of thousands of users submit queries expecting results from tens of billions of web pages — accurate, ranked, and returned in under 200 milliseconds. A web-scale search engine like Google or Bing solves three fundamentally different problems at once: **crawling** the open web faster than it changes, **indexing** hundreds of terabytes of text so any term can be looked up in microseconds, and **ranking** results so the most relevant document surfaces first from billions of candidates.

The naive approach — download pages into a relational database and run a `LIKE '%query%'` scan — fails at the first dimension. A sequential scan of 10 billion documents, even at SSD speeds, takes hours. The interesting engineering lies in the **inverted index** that reduces any query to a handful of microsecond-scale pointer dereferences, the distributed sharding that spreads that index across hundreds of machines, and the scoring algorithms (BM25 + PageRank) that rank billions of candidates in milliseconds.

## Functional Requirements

1. **Crawl:** Continuously discover and fetch pages from the public web; respect `robots.txt` and per-domain crawl-delay rules.
2. **Index:** Process raw HTML into a searchable inverted index; newly published pages must appear in results within 4 hours.
3. **Search:** Given a text query, return the top-10 ranked results (URL, title, snippet), paginated.
4. **Rank:** Score results by a blend of textual relevance (BM25) and link-graph authority (PageRank).
5. **Spell correction:** Detect misspellings and show "Did you mean: …" when the original query yields sparse results.
6. **Query suggestions:** Return autocomplete candidates as the user types.

## Out of Scope

- Personalised ranking (signed-in users, click history, behavioural signals).
- Vertical indices: image, video, news, shopping.
- Ads auction and placement.
- Learning-to-rank ML pipelines and CTR feedback loops (acknowledged but not designed here).
- Anti-spam and link-farm detection beyond basic link-graph PageRank dampening.
- Sub-minute (real-time) index freshness.

## Non-Functional Requirements

- **Scale:** 10 billion indexed documents; 100 million pages crawled or re-crawled per day.
- **Query throughput:** 100,000 searches/s average; 300,000/s peak (3× burst factor).
- **Latency:** Search p50 < 100 ms, p99 < 200 ms end-to-end.
- **Index freshness:** New pages indexed within 4 hours; top-ranked pages re-crawled within 24 hours.
- **Availability:** 99.99% for query serving (≤ 52 minutes downtime/year); indexing pipeline may degrade gracefully.
- **Consistency:** Eventual — a freshly indexed page may take seconds to minutes to become visible in results.
- **Durability:** All index data replicated 3× across independent failure domains.

## Terminology

| Term | Meaning |
|---|---|
| **Posting list** | Sorted list of doc IDs (plus frequency/position metadata) in which a term appears |
| **Inverted index** | Map from term → posting list; the core data structure of every search engine |
| **Forward index** | Map from doc ID → term list and lengths; used for BM25 scoring and snippet generation |
| **BM25** | Okapi BM25 — a probabilistic relevance model that ranks documents against a query |
| **PageRank** | Link-graph authority score; pages linked to by many quality pages score higher |
| **Crawl frontier** | Priority queue of URLs waiting to be fetched |
| **Segment** | An immutable on-disk chunk of an inverted index; multiple small segments merge into larger ones over time |
| **Shard** | A horizontal partition of the index, stored on one (or a small replica set of) machines |
