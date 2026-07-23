---
title: 'Web Crawler'
weight: 1
type: docs
---

A web crawler (also called a spider or bot) is the engine behind every large-scale search index, price-comparison feed, and data-pipeline that needs to systematically download and process content from the web. Given a small set of **seed URLs**, it fetches pages, parses HTML to discover outgoing links, and enqueues those links for future visits — repeating this loop continuously until the crawl budget is exhausted.

At the scale of a search engine the challenge is extraordinary: crawling 1 billion pages per month means ~385 fetches per second on average, terabytes of compressed HTML per day, coordination across dozens of worker machines without re-crawling the same URL twice, and the constant discipline of being a *polite guest* — honouring rate limits, respecting `robots.txt`, and not hammering any single origin server with concurrent requests.

## Functional Requirements

1. **Seed ingestion:** Accept a set of seed URLs to bootstrap the crawl.
2. **Fetch:** Download the HTML content of each URL in the frontier queue.
3. **Link extraction:** Parse fetched HTML to discover outgoing hyperlinks; add new, unseen URLs to the frontier.
4. **URL deduplication:** Track which URLs have already been seen; never enqueue the same URL twice.
5. **Content storage:** Persist raw (compressed) page content for downstream indexers.
6. **Politeness:** Respect `robots.txt` directives; enforce per-domain crawl-rate limits so no origin is overwhelmed.
7. **Freshness:** Re-crawl previously-seen pages on a schedule proportional to how frequently each page changes.

## Out of Scope

- Search indexing, ranking (PageRank), and query serving — crawling and indexing are separate systems.
- Non-HTML content (PDFs, images, video) beyond following links that reference them.
- Authenticated or paywalled pages.
- Anti-bot / CAPTCHA circumvention (ethical crawling is assumed throughout).

## Non-Functional Requirements

- **Scale:** 1 billion pages/month (~385 pages/s average; ~1,000 pages/s at 2.5× peak burst).
- **Storage:** ~100 KB average page size compressed ⇒ ~30 TB/month of content after compression.
- **Bandwidth:** 385 pages/s × 100 KB ≈ 38 MB/s sustained; ~100 MB/s at peak.
- **Scheduling latency:** A newly discovered URL should be scheduled within minutes, not hours.
- **Availability:** Crawler workers best-effort 99.5%; URL frontier and content store must be 99.9% durable.
- **Consistency:** At-least-once URL delivery is acceptable; idempotent deduplication prevents duplicate fetches.
- **Dedup accuracy:** Bloom filter false-positive rate ≤ 1% — occasionally skipping a real page is acceptable; a duplicate fetch wastes bandwidth but is not catastrophic.
- **Politeness SLO:** No single domain receives more than 1 request/second (configurable) from the crawler fleet.
