---
title: Search & Geospatial
weight: 12
type: docs
toc: false
sidebar:
  open: true
---

{{< cards >}}
  {{< card link="typeahead" title="Search Autocomplete / Typeahead" subtitle="Trie data structure, top-k per prefix, Redis sorted sets, batch + real-time data pipeline, and personalization" >}}
  {{< card link="geohash" title="GeoHash" subtitle="Lat/lng to string encoding, prefix property, 9-cell proximity search, Redis GEO commands, and limitations at high latitudes" >}}
  {{< card link="quadtree" title="QuadTree" subtitle="Adaptive 2D spatial partitioning, dynamic subdivision, range queries with pruning, and QuadTree vs GeoHash trade-offs" >}}
  {{< card link="location-indexing" title="Uber-Style Location Indexing" subtitle="High write-rate GPS ingestion, GeoHash cells in Redis sorted sets, stale cleanup, H3 hexagonal grid, and hot cell mitigation" >}}
{{< /cards >}}