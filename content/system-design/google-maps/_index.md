---
title: 'Google Maps / Navigation'
weight: 1
type: docs
---

Google Maps is a global navigation and mapping platform used by over 2 billion people monthly. A user types an origin and a destination and receives turn-by-turn directions, a live-rendered map, and an estimated arrival time that updates continuously as they drive. Behind that interaction are three fundamentally different subsystems: a **road-network graph traversal engine** that finds the fastest path through hundreds of millions of nodes in milliseconds; a **tile pipeline** that pre-renders the planet's geography into billions of image tiles stored in object storage and delivered via a global CDN; and a **real-time traffic system** that ingests a continuous stream of GPS breadcrumbs from every navigating device, aggregates them into edge-speed estimates, and propagates weight updates back to the routing engine within seconds.

Each sub-problem is independently hard at continental scale, and all three must be tightly integrated so rerouting feels instantaneous when a traffic jam appears ahead.

## Functional Requirements

1. **Route computation:** Given an origin and destination (lat/lng or place name), return a turn-by-turn route with distance, ETA, and manoeuvre steps. Support fastest-time and shortest-distance modes.
2. **Map tile serving:** Render and serve raster or vector map tiles at zoom levels 0–20. Tiles load as users pan and zoom.
3. **Real-time traffic:** Ingest GPS probe data from navigating users, detect congestion, update ETAs within seconds, and reroute when a significantly faster path exists.
4. **Route alternatives:** Return 2–3 ranked alternative routes alongside the primary.
5. **POI search:** Return nearby points of interest given a text query and current location.
6. **ETA estimation:** Predict arrival time using historical traffic patterns, live conditions, and ML-based corrections.
7. **Offline maps:** Allow users to download a region for offline navigation with incremental delta sync on reconnect.

## Out of Scope

- Street View imagery capture and serving (a separate camera-crawl and storage pipeline).
- Public transit routing (requires GTFS feed ingestion — a distinct service).
- 3D building rendering and satellite imagery processing.
- Ride-sharing or driver dispatch integration.
- Business listing management and user reviews.
- Voice synthesis for turn instructions (handled client-side from the route payload).

## Non-Functional Requirements

- **Scale:** 2 B registered users; 50 M daily active navigating users; 20 M route requests/day; 50 B map tile requests/day; ~2 M GPS probe events/second from concurrent navigating sessions.
- **Latency:** Route computation p99 < 500 ms (city-level routes < 100 ms); map tile serving p99 < 50 ms globally; GPS probe ingest p99 < 200 ms.
- **Availability:** 99.99 % for tile serving and active turn-by-turn navigation (users are driving and depend on it); 99.9 % for route computation.
- **Consistency:** Traffic edge weights eventually consistent across all routing servers within ≤ 30 s of observation. Route results may use traffic data up to 30 s stale — acceptable for navigation.
- **Durability:** Road graph and tile data — zero loss tolerated; geo data is expensive to recreate. GPS probe data — best-effort; individual probe loss has no user-visible effect.
- **Privacy:** GPS probes anonymised at ingest (stripped of device ID); raw user location not persisted beyond the active navigation session.

## Terminology

| Term | Meaning |
|---|---|
| **Node** | An intersection or road endpoint in the graph |
| **Edge** | A directed road segment between two nodes, carrying distance and speed data |
| **Live weight** | The current traversal cost of an edge in seconds, updated by the traffic stream |
| **Tile** | A fixed-size square image or vector bundle covering a geographic region at a zoom level |
| **Zoom level** | Integer 0 (entire world = 1 tile) to 20 (building-level detail) |
| **GPS probe** | An anonymised location + speed sample sent by the navigation client |
| **ETA** | Estimated Time of Arrival |
| **CH** | Contraction Hierarchies — the precomputation technique enabling fast continental routing |
