---
title: 'Design Framework Template'
weight: 999
type: docs
draft: true
---

<!--
============================================================================
 SYSTEM DESIGN AUTHORING TEMPLATE  (draft: true — never rendered on the site)
============================================================================
This file is the canonical framework every design under content/system-design/
MUST follow. It is a DRAFT, so Hugo's production build (`hugo --gc --minify`)
skips it. It exists only as a copy-paste guide for humans and AI sessions.

HOW TO ADD A NEW DESIGN
-----------------------
1. Create a folder: content/system-design/<slug>/   (kebab-case, e.g. url-shortener)
2. Create these weight-ordered pages (copy the skeletons in PART 2 below):
       _index.md              weight 1   Problem + Requirements
       high-level-design.md   weight 2   Estimation + API + Data model + v1 architecture
       deep-dive.md           weight 3   Iterative refinement + drill-down
       wrap-up.md             weight 4   Interview tips + Resiliency + Observability + links
   For a complex design, split the deep dive into multiple pages instead of one:
       <area>-deep-dive.md    weight 3,4,5...   (e.g. storage-deep-dive, streaming-deep-dive)
   and bump wrap-up's weight so it stays last.
3. Fill every MANDATORY section in PART 1. Do not skip sections or reorder them.
4. Add a card for the design in content/system-design/_index.md.
5. Reuse existing concepts by LINKING to /design-concepts/... — reference, don't re-explain.

STYLE RULES
-----------
- Practical and interview-focused, not academic.
- Show the naive/first-cut design, then EVOLVE it: each refinement is
  Problem -> Modification -> Justification & trade-offs, with its OWN Mermaid diagram.
- Capacity estimation must SHOW THE ARITHMETIC in tables with explicit units.
- Every architecture and every refinement step needs a Mermaid diagram.
- Prefer tables for trade-offs and comparisons.
- Cross-link concepts, e.g.  [consistent hashing]({{% relref "/design-concepts/storage/consistent-hashing" %}}).
============================================================================
-->

# The Framework (what every design must contain)

Follow these seven parts **in order**. PART 1 is the mandatory content checklist;
PART 2 is the copy-paste page skeletons; PART 3 is a filled micro-example.

## PART 1 — Mandatory Section Checklist

### Page 1 — `_index.md` (weight 1): Problem & Requirements
- [ ] **Real-world scenario** — 1–2 paragraphs framing the problem and who uses it.
- [ ] **Functional Requirements** — numbered, precise, in-scope behaviors only.
- [ ] **Out of Scope** — explicit bullets of what you deliberately will NOT build.
- [ ] **Non-Functional / Technical Requirements** — concrete numbers:
      scale (users, QPS), latency SLOs (p50/p99), availability target (e.g. 99.9%),
      consistency model, durability, security/compliance.
- [ ] *(Optional)* **Terminology** — glossary when the domain is jargon-heavy.

### Page 2 — `high-level-design.md` (weight 2): Estimation + API + Data + v1
- [ ] **Capacity Estimation (show the math)** — traffic (DAU → read QPS & write QPS → peak QPS
      with a stated peak factor), storage (bytes/record × records/day × retention), bandwidth,
      memory/hot-set, derived counts (#servers, #shards, #cache nodes). Use tables + units.
- [ ] **API Design** — endpoints, verbs, request/response shapes, status codes,
      pagination/idempotency where relevant.
- [ ] **Data Model** — core entities, fields, relationships; the chosen store per entity.
- [ ] **High-Level Architecture v1** — **Level 0 context** Mermaid diagram AND
      **Level 1 component/data-flow** Mermaid diagram; per-component responsibility,
      initial justification, and first-order trade-offs.

### Page 3 — `deep-dive.md` (weight 3+): Iterative Refinement + Drill-down
- [ ] **Iterative refinement** — take each weakness of v1 and evolve the design:
      **Problem → Modification → Justification & trade-offs**, each with an **updated Mermaid diagram**.
      Common problems: hotspots, write amplification, read scaling, ordering/consistency,
      availability/failover, cost, cardinality/storage blow-up.
- [ ] **Final architecture** — the consolidated Mermaid diagram after all refinements.
- [ ] **Drill-down into every level**:
      - APIs — detailed contracts with concrete request/response examples.
      - Database schema — tables/collections, partition & clustering keys, indexes, sharding.
      - Data structures — the specific ones used (sorted set, trie, bloom filter, geohash cell,
        count-min sketch, LSM tree, inverted index, …) and WHY.
      - Key algorithms — pseudocode for the core logic.
      - Edge cases & failure handling.

### Page 4 — `wrap-up.md` (last weight): Interview + Ops
- [ ] **Interview Tips** — how to open, what to prioritize, likely follow-ups, tradeoff talking points.
- [ ] **Resiliency** — failure modes, redundancy, failover, backpressure, idempotency, DR.
- [ ] **Observability** — SLIs/metrics, logging, tracing, alerting, dashboards.
- [ ] **Concept Cross-links** — bullet list linking each `/design-concepts/...` page reused here.

---

## PART 2 — Copy-Paste Page Skeletons

### `_index.md`
```markdown
---
title: '<Design Name>'
weight: 1
type: docs
---

<One-paragraph real-world scenario: what are we building and for whom.>

## Functional Requirements
1. ...
2. ...

## Out of Scope
- ...

## Non-Functional Requirements
- **Scale:** <users / QPS / data volume>
- **Latency:** <p50 / p99 targets per operation>
- **Availability:** <e.g. 99.9%>
- **Consistency:** <strong / eventual / read-your-writes, and where>
- **Durability / Security:** <as applicable>
```

### `high-level-design.md`
```markdown
---
title: 'High-Level Design'
weight: 2
type: docs
---

## Capacity Estimation

| Metric | Calculation | Result |
|---|---|---|
| Write QPS | <math> | <value> |
| Read QPS | <math> | <value> |
| Peak QPS | avg × <peak factor> | <value> |
| Storage/day | bytes/record × records/day | <value> |
| Storage @ retention | /day × <days> | <value> |
| Hot-set (cache) | <fraction> × working set | <value> |

## API Design
<endpoints, verbs, request/response, status codes>

## Data Model
<entities, fields, relationships, store per entity>

## Architecture v1

### Level 0 — Context
\`\`\`mermaid
graph TD
  User --> System --> DataStore
\`\`\`

### Level 1 — Components
\`\`\`mermaid
graph TD
  ...
\`\`\`

<Per-component: responsibility, justification, trade-offs.>
```

### `deep-dive.md`
```markdown
---
title: 'Deep Dive'
weight: 3
type: docs
---

## Refinement 1 — <weakness being fixed>
**Problem:** ...
**Modification:** ...
**Justification & trade-offs:** ...
\`\`\`mermaid
graph TD
  ...updated design...
\`\`\`

## Refinement 2 — <next weakness>
... (same Problem / Modification / Justification + diagram) ...

## Final Architecture
\`\`\`mermaid
graph TD
  ...
\`\`\`

## Drill-down
### APIs
### Database Schema
### Data Structures
### Key Algorithms
### Edge Cases & Failure Handling
```

### `wrap-up.md`
```markdown
---
title: 'Wrap-Up'
weight: 4
type: docs
---

## Interview Tips
## Resiliency
## Observability
## Related Concepts
- [<concept>]({{%/* relref "/design-concepts/<path>" */%}})
```

---

## PART 3 — Filled Micro-Example (URL Shortener, abbreviated)

> This is a *shortened* illustration of the tone and structure — real designs go much deeper.

**`_index.md`**

> A URL shortener turns a long URL into a short key (e.g. `sho.rt/aB3xZ9`) and redirects on lookup.
>
> **Functional:** (1) create short URL, (2) redirect short→long, (3) custom alias, (4) expiry.
> **Out of scope:** analytics dashboards, user accounts.
> **NFR:** 100M creates/mo, 10B redirects/day (~116K read QPS), redirect p99 < 50 ms, 99.9% uptime,
> redirects may be eventually consistent.

**`high-level-design.md`** (estimation excerpt)

| Metric | Calculation | Result |
|---|---|---|
| Write QPS | 100M / (30×86400) | ~39 /s |
| Read QPS | 10B / 86400 | ~116K /s |
| Storage @ 5y | 100M×12×5 × 500 B | ~3 TB |

Read:write ≈ **3000:1** → cache-heavy, KV store, 301/302 redirect decision, key from base62(counter/Snowflake).

**`deep-dive.md`** (one refinement)

> **Problem:** hashing the long URL causes collisions.
> **Modification:** generate keys from a distributed counter (Snowflake) + base62, not a hash.
> **Justification:** collision-free by construction; trade-off is sequential-ish keys (mitigate with per-shard offsets).

**`wrap-up.md`**: interview tips (lead with read:write ratio), resiliency (cache-aside + DB fallback),
observability (redirect latency, cache hit rate), links to consistent-hashing / caching-patterns / bloom-filters.
