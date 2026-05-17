---
title: Search, Geospatial & Real-Time
weight: 12
type: docs
toc: false
---

{{< cards >}}
  {{< card link="typeahead" title="Search Autocomplete / Typeahead" subtitle="Trie data structure, top-k per prefix, Redis sorted sets, batch + real-time data pipeline, and personalization" >}}
  {{< card link="geohash" title="GeoHash" subtitle="Lat/lng to string encoding, prefix property, 9-cell proximity search, Redis GEO commands, and limitations at high latitudes" >}}
  {{< card link="quadtree" title="QuadTree" subtitle="Adaptive 2D spatial partitioning, dynamic subdivision, range queries with pruning, and QuadTree vs GeoHash trade-offs" >}}
  {{< card link="location-indexing" title="Uber-Style Location Indexing" subtitle="High write-rate GPS ingestion, GeoHash cells in Redis sorted sets, stale cleanup, H3 hexagonal grid, and hot cell mitigation" >}}
  {{< card link="presence-system" title="Presence & Online Status" subtitle="Heartbeat-based detection, Redis TTL auto-expiry, fan-out optimization via scoped pub/sub, flapping prevention, and multi-device presence" >}}
  {{< card link="notification-fanout" title="Notification Fanout Strategies" subtitle="Fan-out on write vs read, hybrid approach for celebrities, APNs/FCM push delivery, deduplication with Redis SETNX, and per-user rate limiting" >}}
  {{< card link="websocket-at-scale" title="WebSocket at Scale" subtitle="Stateful connection management, sticky sessions, cross-server routing via Redis Pub/Sub, session registry externalization, and heartbeat with exponential backoff reconnection" >}}
  {{< card link="mqtt" title="MQTT" subtitle="Lightweight IoT pub/sub protocol, QoS levels (0/1/2), retained messages, Last Will and Testament, persistent sessions, broker clustering, and MQTT vs WebSockets" >}}
  {{< card link="payment-systems" title="Payment System Design" subtitle="Double-entry bookkeeping, idempotency keys for safe retries, outbox pattern for distributed transactions, nightly reconciliation, and PCI scope reduction via tokenization" >}}
  {{< card link="job-scheduling" title="Distributed Job Scheduling" subtitle="Redis sorted set delayed queues, atomic poll with Lua scripts, lease-based at-least-once delivery, leader-based distributed cron, and per-tenant fair scheduling" >}}
  {{< card link="id-generation" title="Unique ID Generation" subtitle="UUID v4 randomness vs B-tree fragmentation, Snowflake 64-bit sortable IDs, ULID timestamp+random, database sequences, and clock skew mitigations" >}}
{{< /cards >}}