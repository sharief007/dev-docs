---
title: 'Uber / Ride-Sharing'
weight: 1
type: docs
---

Uber, Lyft, and Grab connect riders who need transport with nearby drivers, coordinate pickup, track the live trip, and settle payment — all in real time. A rider opens the app, sees available cars on a map, taps "Request," and within seconds a driver is matched and dispatched. The interesting engineering challenges are not CRUD operations but **physics at scale**: millions of drivers continuously broadcasting their GPS positions, a geospatial index that must answer "nearest available drivers" in milliseconds, a matching pipeline that sequences offers and handles rejections without keeping the rider waiting, and a real-time tracking channel that streams a driver's moving pin to the rider's screen for the entire trip.

The constraint that dominates every design decision is the **location-update firehose**: at 5 million active drivers sending GPS pings every four seconds, the system absorbs roughly **1.25 million location writes per second** before considering peak load. This write pressure, combined with the need for sub-second geo-queries, shapes the entire data tier.

## Functional Requirements

1. **Ride request:** A rider submits a pickup location and destination; the system acknowledges and begins matching.
2. **Nearby-driver discovery:** The system queries available drivers within a configurable radius of the rider's pickup point.
3. **Driver matching:** Candidates are ranked by ETA, offered to the best driver first; if rejected or timed-out, the next candidate is tried.
4. **Driver dispatch:** The accepted driver receives pickup details and navigation context.
5. **Real-time trip tracking:** The rider sees the driver's live position on a map throughout pickup and the trip itself.
6. **Fare estimation and surge pricing:** An estimated fare (including any surge multiplier) is shown before the rider confirms.
7. **Payment processing:** At trip end the rider is charged; the driver's earnings are credited.
8. **Trip status updates:** Both rider and driver receive status transitions (matching → accepted → en-route → in-progress → completed).

## Out of Scope

- Driver onboarding, background checks, and document verification.
- Turn-by-turn navigation (delegated to a third-party maps SDK).
- Fraud detection and anti-money-laundering pipelines.
- Driver incentive programs and dynamic pricing ML model training (we use the output, not the training process).
- Ride scheduling (future rides) — assume on-demand only.
- Multi-stop trips or ride-pooling (UberPool-style seat sharing).

## Non-Functional Requirements

- **Scale:** 5M active drivers; 15M active riders; 8M rides/day; ~111k concurrent rides (average), ~300k at peak.
- **Location write throughput:** 5M drivers × 1 ping/4 s = **1.25M location writes/s** (peak ×2 = 2.5M/s).
- **Latency:** Rider receives first driver offer ≤ 5 s from request; location update ingested and reflected in geo-index ≤ 500 ms; ride-status transitions p99 ≤ 1 s.
- **Availability:** 99.99% for matching and trip-status services; 99.9% for payment (retryable with idempotency).
- **Consistency:** Location data is eventually consistent — up to ~4 s stale is acceptable (one ping interval). Payment is strongly consistent per ride: no double-charge, no missed charge.
- **Durability:** Every completed ride record (billing, regulatory compliance) must survive any single-node failure; replication factor ≥ 3.
- **Security:** Driver GPS and rider identity are sensitive PII; transport over mTLS; location retained per jurisdictional compliance requirements.

## Terminology

| Term | Meaning |
|---|---|
| **GeoHash** | A base-32 string encoding of lat/lon into a hierarchical rectangular grid; adjacent cells share a common prefix |
| **H3** | Uber's open-source hexagonal hierarchical spatial index; divides the globe into hex cells at 16 resolution levels |
| **QuadTree** | A 2-D tree that recursively subdivides space into equal quadrants |
| **Surge multiplier** | A factor (e.g. 1.8×) applied to the base fare when local demand exceeds local supply |
| **Offer** | A short-lived dispatch message sent to one driver; expires after ~15 s |
| **ETA** | Estimated time of arrival of the driver at the rider's pickup point |
| **k-ring** | In H3, the set of all cells within k hexagonal steps of an origin cell |
