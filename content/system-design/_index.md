---
title: System Design Use Cases
weight: 1
type: docs
toc: false
---

Real-world system design case studies, each built to a single, strict framework:
**precise requirements → capacity estimation → high-level architecture → iterative refinement →
deep dives (APIs, schema, data structures, algorithms) → interview tips, resiliency, and observability.**

New designs follow the shared framework template (`_TEMPLATE.md`) in this folder.

{{< cards >}}
    {{< card link="url-shortener" title="URL Shortener" subtitle="Key generation, base62, cache-aside redirects, bloom filters, analytics" >}}
    {{< card link="rate-limiter" title="Rate Limiter Service" subtitle="Token bucket & sliding window, Redis Lua atomicity, fail-open vs fail-closed" >}}
    {{< card link="leaderboard" title="Real-Time Leaderboard" subtitle="Redis sorted sets, top-N & rank queries, tie-breaking, sharded scaling" >}}
    {{< card link="distributed-cache" title="Distributed Cache" subtitle="Consistent hashing, stampede prevention, hot-key replication, eviction" >}}
    {{< card link="typeahead" title="Typeahead / Autocomplete" subtitle="Trie with top-K per node, offline pipeline, trending updates, sharding" >}}
    {{< card link="notification-system" title="Notification System" subtitle="Multi-channel fan-out, DLQ retries, dedup, user preferences" >}}
    {{< card link="key-value-store" title="Key-Value Store" subtitle="Consistent hashing, quorum R/W, gossip failure detection, LSM storage" >}}
    {{< card link="distributed-message-queue" title="Distributed Message Queue" subtitle="Partitioned logs, consumer groups, ISR replication, exactly-once" >}}
    {{< card link="twitter-feed" title="Twitter Home Timeline" subtitle="Fan-out on write vs read, hybrid for celebrities, ranking, media" >}}
    {{< card link="instagram" title="Instagram / Photo Sharing" subtitle="Media pipeline, CDN delivery, hybrid feed fan-out, likes at scale" >}}
    {{< card link="web-crawler" title="Web Crawler" subtitle="URL frontier, bloom-filter dedup, politeness, distributed coordination" >}}
    {{< card link="search-engine" title="Search Engine" subtitle="Inverted index, BM25, sharded query processing, link analysis" >}}
    {{< card link="ride-sharing" title="Uber / Ride-Sharing" subtitle="Geospatial indexing, matching, trip tracking, surge, payments" >}}
    {{< card link="chat-system" title="WhatsApp / Chat System" subtitle="WebSockets at scale, message ordering, delivery receipts, E2E encryption" >}}
    {{< card link="video-streaming" title="YouTube / Video Streaming" subtitle="Transcoding pipeline, adaptive bitrate (HLS/DASH), CDN, metadata" >}}
    {{< card link="google-maps" title="Google Maps / Navigation" subtitle="Road-network graph, A* routing, map tiles, real-time traffic, ETA" >}}
    {{< card link="file-storage" title="Dropbox / Google Drive" subtitle="Chunking, content-addressed dedup, delta sync, conflict resolution" >}}
    {{< card link="payment-system" title="Payment System" subtitle="Idempotency, double-entry ledger, saga/outbox, reconciliation" >}}
{{< /cards >}}
