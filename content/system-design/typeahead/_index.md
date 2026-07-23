---
title: 'Typeahead / Autocomplete'
weight: 1
type: docs
---

When you type "best piz" into a search bar and see "best pizza near me", "best pizza dough recipe", and "best pizza in New York" appear before you finish—that is **typeahead**, also called search-as-you-type or autocomplete. Google Search, Amazon product search, YouTube, Spotify, and IDE symbol-completion all depend on it. Every keystroke fires a request, so the system must return ranked suggestions in under 100 milliseconds at massive scale—billions of requests per day—while staying fresh enough to surface trending topics.

The data structure at the heart of typeahead is a **prefix trie** whose nodes cache their own top-K completions, turning a subtree traversal into an O(prefix-length) point lookup. The surrounding engineering—how the trie is built from query logs, deployed atomically to serving nodes, sharded across a fleet, fronted by a CDN, and kept live with a streaming update layer—is where the real design complexity lives.

## Functional Requirements

1. **Suggest:** Given a query prefix (1–50 characters), return up to **K = 10** ranked completions that begin with that prefix.
2. **Ranking:** Suggestions are ordered by **query frequency** weighted by **recency**—recent queries score higher than stale ones with the same raw count.
3. **Freshness:** The index reflects queries from the past 7 days; a **batch pipeline** rebuilds the trie every 24 hours; a **streaming layer** propagates trending queries within 15 minutes.
4. **Language support:** English (ASCII) and common Unicode scripts (Latin-extended, CJK, Arabic, Cyrillic) with proper normalization.
5. **Typo tolerance (stretch):** Optionally return suggestions for prefixes with 1–2 edit-distance errors (e.g. `"googel"` → suggestions for `"google"`).

## Out of Scope

- Per-user personalized ranking beyond the shared popularity signal (requires a separate user-profile service).
- Full-text body search—this covers prefix-based query completion only, not document retrieval.
- Spell-correction of fully submitted queries.
- Semantic / intent-based suggestions (synonyms, NLP reranking).
- User authentication, rate limiting, and abuse detection (handled by an upstream API gateway).

## Non-Functional Requirements

- **Scale:** 500M DAU; ~3 searches/user/day; ~4 prefix queries per search (keystrokes) → **~6B autocomplete requests/day** (~69 K req/s average, ~207 K req/s peak at 3× burst).
- **Latency:** p50 < 50 ms, p99 < **100 ms** end-to-end. The trie lookup itself must complete in **< 5 ms**; the remainder is network and serialization budget.
- **Availability:** **99.9%** (≤ 44 min downtime/month). Degraded mode: serve stale cached suggestions rather than errors.
- **Freshness SLO:** Batch rebuild ≤ 24 hours; trending query visible across all nodes ≤ 15 minutes via the streaming path.
- **Consistency:** Eventual. Different serving nodes may briefly hold different trie snapshots; this is acceptable.
- **Durability:** Raw query logs retained 90 days (compliance + pipeline retraining).
- **Privacy:** The trie stores aggregated frequencies only—not individual queries. Queries below a minimum frequency threshold (e.g. fewer than 5 occurrences) are suppressed.

## Terminology

| Term | Meaning |
|---|---|
| Prefix | The characters typed so far, e.g. `"how do i"` |
| Completion / suggestion | A full query that begins with the prefix, e.g. `"how do i lose weight"` |
| Top-K cache | Each trie node pre-stores the K highest-scored completions reachable from it |
| Trie | Prefix tree where each root-to-node path spells a string |
| Score | Time-decayed frequency—raw count × recency weight |
| Snapshot | An immutable, serialized trie image swapped atomically onto serving nodes |
