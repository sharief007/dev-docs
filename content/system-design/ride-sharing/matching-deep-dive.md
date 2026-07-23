---
title: 'Matching, Surge, Tracking & Payment Deep Dive'
weight: 4
type: docs
---

With the location layer established, this page tackles the four remaining hard problems: **matching drivers to riders**, **surge pricing**, **real-time trip tracking over WebSockets**, and **payment processing with idempotency**.

---

## Refinement 4 — The Matching Pipeline

**Problem.** Given a list of nearby available drivers from the geo-index, the system must (a) rank them by ETA, (b) send an offer to the best candidate, (c) wait for acceptance or timeout/rejection, and (d) cascade to the next candidate — all within the 5-second rider-experience SLO.

**Modification.** The Matching Service is a per-ride orchestrator. It is stateless itself, but it uses Redis for offer state (with TTL) so any Matching Service node can pick up a retry.

```mermaid
flowchart TB
    RideSvc["Ride Service<br/>(creates ride and triggers match)"]
    MatchSvc["Matching Service<br/>(stateless orchestrator)"]
    GeoIdx[("H3 Geo-Index<br/>(Redis)")]
    ETASvc["ETA Service<br/>(road-graph cache or Maps API)"]
    OfferStore[("Offer Store<br/>Redis, key TTL = 15s)")]
    PushSvc["Push Notification Service"]
    DA["Driver App"]

    RideSvc -->|match ride_id pickup_loc| MatchSvc
    MatchSvc -->|k-ring SMEMBERS| GeoIdx
    GeoIdx --> MatchSvc
    MatchSvc -->|batch ETA estimate| ETASvc
    ETASvc --> MatchSvc
    MatchSvc -->|SET offer:{offer_id} EX 15| OfferStore
    MatchSvc -->|push offer notification| PushSvc --> DA
    DA -->|accept or reject| RideSvc
    RideSvc -->|outcome| MatchSvc
```

### Matching Pseudocode

```python
MAX_INITIAL_RADIUS_KM = 5
EXPAND_STEP_KM        = 2
MAX_RADIUS_KM         = 15
OFFER_TIMEOUT_S       = 15

def match_ride(ride_id, pickup_lat, pickup_lon):
    radius_km = MAX_INITIAL_RADIUS_KM

    while radius_km <= MAX_RADIUS_KM:
        # 1. Query geo-index for nearby available drivers
        candidates = geo_index.nearby_available(pickup_lat, pickup_lon, radius_km)

        if not candidates:
            radius_km += EXPAND_STEP_KM
            continue

        # 2. Rank by ETA (ascending) using cached road-graph estimates
        ranked = sorted(candidates,
                        key=lambda d: eta_service.estimate(d, pickup_lat, pickup_lon))

        for driver in ranked:
            # 3. Atomically claim driver to prevent double-dispatch
            #    NX = only set if key does not exist; EX = TTL in seconds
            claimed = redis.set(f"offer_lock:{driver.id}", ride_id,
                                nx=True, ex=OFFER_TIMEOUT_S)
            if not claimed:
                continue   # driver already has an open offer from another ride

            # 4. Create offer record with TTL
            offer_id = generate_id()
            redis.setex(f"offer:{offer_id}", OFFER_TIMEOUT_S,
                        json_encode({ride_id, driver.id, pickup_lat, pickup_lon}))

            # 5. Push offer to driver app (FCM / APNs)
            push_notification(driver.id, offer_id, timeout_s=OFFER_TIMEOUT_S)

            # 6. Wait for driver response (blocking with timeout)
            response = wait_for_response(offer_id, timeout=OFFER_TIMEOUT_S)
            redis.delete(f"offer_lock:{driver.id}")

            if response == ACCEPT:
                assign_driver(ride_id, driver.id)
                geo_index.mark_unavailable(driver.id)   # SREM from h3:cell:available
                return SUCCESS
            # REJECT or TIMEOUT → try next driver in ranked list

        # No driver accepted in this radius; expand
        radius_km += EXPAND_STEP_KM

    # All radii exhausted
    cancel_ride(ride_id, reason=NO_DRIVERS)
    return FAILURE
```

**Key design decisions and trade-offs:**

| Decision | Choice | Rationale |
|---|---|---|
| **Offer dispatch** | Serial (one at a time) | Simpler; only one driver is ever assigned. Parallel offers require a compare-and-swap race and excess-offer cancellation. |
| **Offer mutex** | Redis NX + TTL | Prevents two rides from offering to the same driver simultaneously. TTL auto-releases if the Matching Service crashes. |
| **Radius expansion** | +2 km per cycle | Automatically broadens search when no one accepts; trades wait time for supply coverage. |
| **ETA ranking** | Cached Maps API call | Calling Maps API per-driver per-request is expensive; cache by (driver grid cell, pickup grid cell) for 60 s. |
| **Assignment** | Write to Ride DB + SREM from geo-index | Ride DB is the source of truth; geo-index update is eventually consistent but lag is bounded by the next Kafka flush. |

---

## Refinement 5 — Surge Pricing

**Problem.** Static fare pricing provides no supply/demand signal. During rush hour or severe weather, demand spikes while supply stays flat. Without a higher fare, queues form, riders wait, and drivers have no incentive to reposition.

**Modification.** Compute a surge multiplier per H3 cell at **resolution 6** (~36 km² per cell — city-district granularity). The Surge Pricing Service runs every 30 seconds.

```mermaid
flowchart LR
    GeoIdx[("Geo-Index<br/>available drivers per res-6 cell")]
    RideSvc["Ride Service<br/>(open request counts per cell)"]
    SurgeSvc["Surge Service<br/>(runs every 30s)"]
    SurgeCache[("Surge Cache<br/>Redis SETEX per H3 cell")]
    RideAPI["Ride Request API<br/>(fare estimate on POST /rides)"]

    GeoIdx -->|supply counts| SurgeSvc
    RideSvc -->|demand counts| SurgeSvc
    SurgeSvc -->|SETEX surge:{cell_id} TTL=90| SurgeCache
    RideAPI -->|GET surge:{cell_id}| SurgeCache
```

### Surge Computation

```python
MAX_SURGE = 3.0

def compute_surge(h3_cell_res6):
    # Supply: available drivers whose current res-9 cell is a child of this res-6 cell
    supply = geo_index.driver_count_in_cell(h3_cell_res6)

    # Demand: ride requests opened in the last 60 s whose pickup is in this cell
    demand = ride_service.open_requests_in_cell(h3_cell_res6, window_s=60)

    if supply == 0:
        return MAX_SURGE

    ratio = demand / supply

    # Piecewise linear ramp
    if ratio < 0.5:
        return 1.0
    elif ratio < 2.0:
        return 1.0 + (ratio - 0.5) / 1.5 * 0.5   # linearly 1.0 → 1.5
    else:
        return min(1.5 + (ratio - 2.0) * 0.25, MAX_SURGE)
```

**Why resolution 6?** Resolution-9 cells (individual city blocks) are too granular — supply in a single block fluctuates wildly and a single parked driver swings the multiplier. Resolution 6 (~36 km²) gives a city-district signal that smooths noise while remaining geographically meaningful.

**Fare lock at booking.** The surge multiplier shown to the rider at booking time is stored in the ride record (`surge_mult`) and never changes after acceptance. This provides the fare guarantee riders expect: "what you see is what you pay."

**Justification & trade-offs.** A 30-second update interval prevents fare instability while staying responsive to real demand shifts. The Redis TTL of 90 s (3× the update interval) ensures a stale multiplier is never served for long even if the Surge Service crashes. See {{% relref "/design-concepts/storage/key-value-stores" %}} for Redis usage patterns.

---

## Refinement 6 — Real-Time Trip Tracking over WebSockets

**Problem.** During pickup and the trip, the rider needs the driver's live position every ~4 s. A polling approach — `GET /rides/{id}` every 4 s for 333k concurrent riders — generates **83k requests/s** of pure polling overhead. Each poll also adds a network round-trip of latency between the GPS update and the rider seeing it on screen.

**Modification.** Use **WebSocket connections** for server-pushed tracking. The rider app opens a WebSocket to the WebSocket Gateway when the ride is accepted. Driver GPS updates flow through the location ingestion path, get published to a per-ride pub/sub channel, and the WebSocket Gateway fan-outs to the rider's socket.

```mermaid
flowchart TB
    DA["Driver App"]
    IGN["Location Ingestion"]
    PubSub[["Redis Pub/Sub<br/>channel: ride:{ride_id}:loc"]]
    WSG["WebSocket Gateway<br/>(~70 nodes, ~10k conns each)"]
    RA["Rider App"]

    DA -->|GPS update every 4s| IGN
    IGN -->|PUBLISH ride:{ride_id}:loc payload| PubSub
    WSG -->|SUBSCRIBE ride:{ride_id}:loc| PubSub
    WSG -->|push JSON to rider socket| RA
    RA -->|WebSocket| WSG
```

### WebSocket Gateway Design

- **Connection routing:** The API Gateway uses consistent hashing on `ride_id` to route both rider and driver connections for the same ride to the same (or adjacent) WebSocket Gateway shard. This means the gateway node is already subscribed to the ride's pub/sub channel before the driver starts moving.
- **Pub/Sub throughput:** 4 s update rate × 333k active rides = **83k messages/s** published through Redis pub/sub — well within Redis's 1M+ messages/s capacity.
- **Reconnect handling:** On reconnect (network hiccup, app background/foreground), the rider app re-opens the WebSocket and receives the last-known location immediately (from `HGETALL driver:{driver_id}` in the geo-index) before live updates resume.
- **Driver-to-gateway path:** The driver app sends GPS to the standard Location Ingestion endpoint. The ingestion node publishes to both Kafka (for geo-index updates) and the ride-specific pub/sub channel (for real-time WebSocket delivery). This gives the fastest possible path to the rider's screen (~10–50 ms) without waiting for the Kafka consumer to update the geo-index first.

**Justification & trade-offs.** WebSockets are the right transport for sustained, low-latency, server-initiated streams. See {{% relref "/design-concepts/networking/realtime-transport" %}} and {{% relref "/design-concepts/specialized/websocket-at-scale" %}}. Server-Sent Events (SSE) would also work for rider-only push but WebSockets are preferred because drivers also receive offers, status updates, and navigation hints on the same connection. The trade-off is statefulness: WebSocket gateway crashes drop all connections. Fast reconnect (< 2 s) plus last-known-location replay on reconnect is the mitigation.

---

## Refinement 7 — Payment Processing with Idempotency

**Problem.** At trip end the system must charge the rider exactly once even when:
- The Payment Service times out and the caller retries.
- The driver's app sends "trip complete" twice (double-tap or network retry).
- The system crashes after starting payment but before recording success.

**Modification.** Use `ride_id` as the natural **idempotency key**. The Ride Service commits a record to an **outbox table** in the same DB transaction as the ride status update. An Outbox Worker polls for unprocessed events and forwards them to the Payment Service — which passes the same idempotency key to the payment gateway.

```mermaid
flowchart TB
    DA["Driver App"]
    RideSvc["Ride Service"]
    RideDB[("Ride DB")]
    Outbox[("Outbox Table<br/>(same DB, same transaction)")]
    OutboxWorker["Outbox Worker<br/>(polls every 1s)"]
    PaySvc["Payment Service"]
    PayGW["Payment Gateway<br/>(Stripe / Braintree)"]
    PayDB[("Payment DB")]

    DA -->|POST /rides/{id}/complete| RideSvc
    RideSvc -->|UPDATE rides SET status=COMPLETING| RideDB
    RideSvc -->|INSERT outbox row| Outbox
    Outbox -.same transaction.-> RideDB
    OutboxWorker -->|poll WHERE processed=false| Outbox
    OutboxWorker -->|POST /charge Idempotency-Key: ride_id| PaySvc
    PaySvc -->|charge with idempotency key| PayGW
    PaySvc -->|INSERT payments| PayDB
    PaySvc -->|UPDATE rides SET status=COMPLETED| RideDB
    PaySvc -->|UPDATE outbox SET processed=true| Outbox
```

### Payment Flow (Step-by-Step)

```
1. Driver POSTs /rides/{ride_id}/complete with {distance_km, duration_min}.

2. Ride Service validates:
   - ride.status == IN_PROGRESS
   - driver_id matches the assigned driver
   - trip_distance_km is plausible (< 500 km)

3. Calculate final fare:
   fare = (BASE_FARE + PER_KM * distance_km + PER_MIN * duration_min) * surge_mult

4. Atomically in one DB transaction:
   a. UPDATE rides
      SET status='COMPLETING', act_fare_usd=fare, completed_at=now()
      WHERE ride_id=? AND status='IN_PROGRESS'   -- guard against double-complete
   b. INSERT INTO outbox (event_type, ride_id, payload, idempotency_key)
      VALUES ('CHARGE_RIDER', ride_id, {rider_id, amount, driver_id}, ride_id)

5. Outbox Worker picks up the row (within ~1 s) and calls Payment Service
   with header Idempotency-Key: {ride_id}.

6. Payment Service calls the payment gateway (Stripe / Braintree) with
   the same idempotency key.
   - If the gateway already processed this key → returns prior result.
   - No double charge, regardless of retries.

7. On gateway success:
   - INSERT INTO payments (payment_id, ride_id, amount, status=succeeded, ...)
   - UPDATE rides SET status='COMPLETED'
   - UPDATE outbox SET processed=true
   - Notify rider (receipt) and driver (payout confirmation).

8. On gateway failure:
   - outbox row stays processed=false; Worker retries with exponential backoff.
   - After N failures: move to dead-letter; alert on-call; ride stuck in COMPLETING.
```

**Justification & trade-offs.**

- **Outbox pattern** commits the payment intent atomically with the ride status change. Even if the Payment Service is down for hours, the outbox row is durable in the same DB; no event is lost. See {{% relref "/design-concepts/distributed/idempotency" %}}.
- **Idempotency key = ride_id** is stable across retries, unique per charge, and meaningful for debugging.
- **COMPLETING state as a distributed lock:** only one "complete" flow can be active at a time per ride, because the UPDATE guard (`WHERE status='IN_PROGRESS'`) is atomic and the status moves to COMPLETING immediately.
- **Trade-off:** outbox polling adds ~1–5 s before the charge is submitted. The rider sees "processing payment" feedback during this window. Eliminate this latency if needed by replacing polling with CDC (Change Data Capture) on the outbox table and streaming directly to the Payment Service.

---

## Final Architecture — Full System

```mermaid
flowchart TB
    subgraph ClientApps["Client Apps"]
      RA["Rider App"]
      DA["Driver App"]
    end

    subgraph EdgeLayer["Edge"]
      APIGW["API Gateway<br/>(Auth, Rate Limit, TLS)"]
      WSG["WebSocket Gateway<br/>(~70 nodes)"]
    end

    subgraph CoreServices["Core Services"]
      LocIng["Location Ingestion<br/>(~20 nodes)"]
      MatchSvc["Matching Service<br/>(stateless)"]
      RideSvc["Ride Service<br/>(state machine)"]
      SurgeSvc["Surge Pricing<br/>(every 30s)"]
      PaySvc["Payment Service"]
    end

    subgraph StreamLayer["Streaming Layer"]
      KF[["Kafka<br/>location-updates"]]
      LocCons["Location Consumers<br/>(~10 nodes)"]
      OutboxW["Outbox Worker"]
    end

    subgraph DataLayer["Data Layer"]
      GeoIdx[("H3 Geo-Index<br/>Redis Sets")]
      DrvHash[("Driver Hash<br/>Redis")]
      SurgeCache[("Surge Cache<br/>Redis")]
      RideDB[("Ride DB<br/>PostgreSQL sharded")]
      PayDB[("Payment DB<br/>PostgreSQL")]
      Outbox[("Outbox Table")]
      PubSub[["Redis Pub/Sub<br/>ride:{id}:loc"]]
    end

    PayGW["Payment Gateway"]

    %% Location ingestion
    DA -->|GPS every 4s| LocIng
    LocIng --> KF
    LocIng -->|PUBLISH ride:{id}:loc| PubSub
    KF --> LocCons
    LocCons -->|SADD or SREM| GeoIdx
    LocCons -->|HSET| DrvHash
    LocCons -->|supply counts| SurgeSvc
    SurgeSvc --> SurgeCache

    %% Ride request
    RA -->|POST /rides| APIGW --> RideSvc
    RideSvc -->|GET surge:{cell}| SurgeCache
    RideSvc -->|match| MatchSvc
    MatchSvc -->|k-ring SMEMBERS| GeoIdx
    MatchSvc -->|HGETALL driver:{id}| DrvHash
    MatchSvc -->|push offer| DA
    DA -->|accept or reject| APIGW --> RideSvc
    RideSvc --> RideDB

    %% Live tracking
    RA -->|WebSocket| WSG
    WSG -->|SUBSCRIBE ride:{id}:loc| PubSub
    WSG -->|push location| RA

    %% Payment
    DA -->|POST /rides/{id}/complete| APIGW --> RideSvc
    RideSvc -->|UPDATE + INSERT outbox| Outbox
    Outbox -.same tx.-> RideDB
    OutboxW -->|poll| Outbox
    OutboxW --> PaySvc
    PaySvc --> PayGW
    PaySvc --> PayDB
    PaySvc -->|UPDATE status=COMPLETED| RideDB
```

---

## Drill-Down

### Detailed Database Schema

```sql
-- Rides (sharded by ride_id; Citus, Cassandra, or DynamoDB)
CREATE TABLE rides (
  ride_id         UUID             PRIMARY KEY,
  rider_id        BIGINT           NOT NULL,
  driver_id       BIGINT,
  status          TEXT             NOT NULL,
  -- status: requested|matching|accepted|en_route|in_progress|completing|completed|cancelled
  pickup_lat      DOUBLE PRECISION NOT NULL,
  pickup_lon      DOUBLE PRECISION NOT NULL,
  dest_lat        DOUBLE PRECISION NOT NULL,
  dest_lon        DOUBLE PRECISION NOT NULL,
  pickup_h3_cell  TEXT,            -- H3 res-6 cell for surge lookup
  surge_mult      NUMERIC(4,2)     NOT NULL DEFAULT 1.0,
  est_fare_usd    NUMERIC(8,2),
  act_fare_usd    NUMERIC(8,2),
  created_at      TIMESTAMPTZ      NOT NULL DEFAULT now(),
  accepted_at     TIMESTAMPTZ,
  pickup_at       TIMESTAMPTZ,
  completed_at    TIMESTAMPTZ
);
CREATE INDEX idx_rides_rider  ON rides(rider_id, created_at DESC);
CREATE INDEX idx_rides_driver ON rides(driver_id, created_at DESC);
CREATE INDEX idx_rides_active ON rides(status)
  WHERE status NOT IN ('completed', 'cancelled');

-- Payments (separate DB for PCI-DSS boundary)
CREATE TABLE payments (
  payment_id        UUID         PRIMARY KEY,
  ride_id           UUID         NOT NULL UNIQUE,  -- one payment per ride
  rider_id          BIGINT       NOT NULL,
  driver_id         BIGINT       NOT NULL,
  amount_usd        NUMERIC(8,2) NOT NULL,
  gateway_charge_id TEXT,
  status            TEXT         NOT NULL,   -- pending|succeeded|failed
  idempotency_key   TEXT         NOT NULL UNIQUE,  -- = ride_id string
  created_at        TIMESTAMPTZ  NOT NULL DEFAULT now()
);

-- Outbox (same DB as rides, committed in same transaction)
CREATE TABLE outbox (
  id              BIGSERIAL    PRIMARY KEY,
  event_type      TEXT         NOT NULL,
  ride_id         UUID         NOT NULL,
  payload         JSONB        NOT NULL,
  idempotency_key TEXT         NOT NULL,
  processed       BOOLEAN      NOT NULL DEFAULT false,
  created_at      TIMESTAMPTZ  NOT NULL DEFAULT now()
);
CREATE INDEX idx_outbox_pending ON outbox(created_at)
  WHERE processed = false;
```

### Data Structures Summary

| Structure | Where Used | Why |
|---|---|---|
| **Redis Set per H3 cell** | Geo-index (available drivers) | O(1) SADD/SREM on move; O(k) batched SMEMBERS for k-ring query |
| **Redis Hash per driver** | Driver current state | Atomic multi-field update; O(1) position fetch |
| **Redis String + NX + TTL** | Offer locks | Distributed mutex with automatic expiry; prevents double-dispatch |
| **Redis Pub/Sub channel** | Real-time trip tracking | Low-latency fan-out of location events to rider WebSocket |
| **Kafka topic** | Location update firehose | Durable ordered buffer; replay-capable; decouples ingestion from consumers |
| **Outbox table** | Payment at-least-once | Transactional guarantee; survives Payment Service downtime |

### Key Algorithms

**ETA Estimation:**

```python
def estimate_eta_seconds(driver_lat, driver_lon, pickup_lat, pickup_lon):
    # Cache key: coarse driver cell + coarse pickup cell (resolution 7, ~5 km²)
    cache_key = (f"eta:{h3.geo_to_h3(driver_lat, driver_lon, 7)}"
                 f":{h3.geo_to_h3(pickup_lat, pickup_lon, 7)}")
    cached = redis.get(cache_key)
    if cached:
        return int(cached)

    # Call Maps API for road-network travel time
    eta_s = maps_api.directions(driver_lat, driver_lon,
                                 pickup_lat, pickup_lon).duration_s
    redis.setex(cache_key, 60, eta_s)   # cache for 60 s
    return eta_s
```

**Fare Calculation:**

```python
BASE_FARE_USD = 1.50
PER_KM_USD    = 0.90
PER_MIN_USD   = 0.25

def calculate_fare(distance_km, duration_min, surge_mult):
    raw = BASE_FARE_USD + (PER_KM_USD * distance_km) + (PER_MIN_USD * duration_min)
    return round(raw * surge_mult, 2)
```

### Edge Cases & Failure Handling

| Scenario | Handling |
|---|---|
| **Driver app crashes mid-trip** | Location updates stop; heartbeat sweeper removes driver from geo-index after 60 s. Ride stays IN_PROGRESS; driver reconnects and resumes. Rider sees last-known position on map. |
| **Driver accepts offer after TTL expires** | Redis offer key has expired → 409 Conflict. Ride Service continues to next candidate. |
| **Payment gateway timeout** | Outbox row remains `processed=false`; Outbox Worker retries with exponential backoff, same `ride_id` idempotency key. |
| **Geo-index Redis node failure** | Matching falls back to Redis replica. If all replicas unavailable: new match requests return 503; in-flight rides continue on WebSocket. |
| **Kafka consumer lag spike** | Location data becomes stale. Alert fires at > 10 s lag. Consumer group auto-scales (Kubernetes HPA). Stale drivers are caught by heartbeat TTL sweeper at 60 s. |
| **Driver double-taps "trip complete"** | Second request hits Ride Service while status is COMPLETING or COMPLETED; the `WHERE status='IN_PROGRESS'` guard produces 0 rows updated → idempotent 200 with existing fare. |
| **Rider cancels after match** | Ride moves to CANCELLED; offer lock deleted from Redis; driver's status set back to AVAILABLE (SADD to geo-index). Cancellation fee applied if past grace period (e.g. 2 min after driver accepted). |
| **No drivers accept in any radius** | Ride cancelled with `NO_DRIVERS` reason. Rider is notified. Surge multiplier in the area is bumped to attract supply signal. |
| **Surge Service down** | Last-written Redis values remain (TTL = 90 s). Ride requests use cached multiplier. Surge Service restarts and recomputes; maximum staleness = 90 s. |
