---
title: GeoHash
weight: 2
type: docs
---

"Find all restaurants within 1km of my location." The database has 50 million restaurant records with latitude and longitude columns. A naive query computes the Haversine distance from your location to every single restaurant — a full table scan. GeoHash solves this by encoding a 2D coordinate into a 1D string so that **nearby points share a common prefix**, turning a spatial proximity query into a simple string prefix match.

## How GeoHash Encoding Works

GeoHash recursively bisects the world map, alternating between longitude and latitude, encoding each bisection as a bit. The bits are then grouped into characters using Base32.

### Step-by-Step Encoding

Encode latitude **40.7128** (New York City), longitude **-74.0060**.

**Longitude bits** (range: -180 to +180):

```
Step 1: [-180, 180] → midpoint 0     → -74.006 < 0  → bit 0 → range [-180, 0]
Step 2: [-180, 0]   → midpoint -90   → -74.006 > -90 → bit 1 → range [-90, 0]
Step 3: [-90, 0]    → midpoint -45   → -74.006 < -45 → bit 0 → range [-90, -45]
Step 4: [-90, -45]  → midpoint -67.5 → -74.006 < -67.5 → bit 0 → range [-90, -67.5]
Step 5: [-90, -67.5]→ midpoint -78.75→ -74.006 > -78.75→ bit 1 → range [-78.75, -67.5]
...
Longitude bits: 0 1 0 0 1 ...
```

**Latitude bits** (range: -90 to +90):

```
Step 1: [-90, 90]   → midpoint 0    → 40.7128 > 0   → bit 1 → range [0, 90]
Step 2: [0, 90]     → midpoint 45   → 40.7128 < 45  → bit 0 → range [0, 45]
Step 3: [0, 45]     → midpoint 22.5 → 40.7128 > 22.5→ bit 1 → range [22.5, 45]
Step 4: [22.5, 45]  → midpoint 33.75→ 40.7128 > 33.75→ bit 1 → range [33.75, 45]
Step 5: [33.75, 45] → midpoint 39.375→ 40.7128 > 39.375→ bit 1 → range [39.375, 45]
...
Latitude bits: 1 0 1 1 1 ...
```

**Interleave** (longitude bit, latitude bit, longitude bit, ...):

```
Longitude: 0 1 0 0 1 ...
Latitude:  1 0 1 1 1 ...
Interleaved: 01 10 01 01 11 ... = 01100 10111 ...
```

**Base32 encode** each 5-bit group into a character: `01100` = "d", `10111` = "r", ... → GeoHash string: `"dr5ru..."`.

### Precision and Cell Size

Each additional character halves the cell area. More characters = smaller cell = higher precision.

| GeoHash Length | Cell Width | Cell Height | Approximate Area |
|---------------|-----------|------------|-----------------|
| 1 | 5,000 km | 5,000 km | ~25,000,000 km² |
| 2 | 1,250 km | 625 km | ~781,000 km² |
| 3 | 156 km | 156 km | ~24,000 km² |
| 4 | 39 km | 19.5 km | ~760 km² |
| 5 | 4.9 km | 4.9 km | ~24 km² |
| **6** | **1.2 km** | **0.6 km** | **~0.7 km²** |
| 7 | 153 m | 153 m | ~0.023 km² |
| **8** | **38 m** | **19 m** | **~700 m²** |

**Rule of thumb:** 6 characters for ~1km precision queries (nearby restaurants), 8 characters for ~40m precision (street-level).

## The Prefix Property

The key insight: **points in the same GeoHash cell share the same GeoHash prefix**. If two restaurants both have GeoHash `"dr5ru7"`, they are within the same ~1km cell. If they share prefix `"dr5ru"`, they are within the same ~5km cell.

```
Restaurant A: lat=40.7128, lng=-74.0060 → GeoHash = "dr5ru7h3"
Restaurant B: lat=40.7135, lng=-74.0052 → GeoHash = "dr5ru7h8"
                                          Shared prefix: "dr5ru7h"  → within ~150m

Restaurant C: lat=40.7580, lng=-73.9855 → GeoHash = "dr5rueg5"
                                          Shared prefix: "dr5ru"    → within ~5km
```

This means a proximity query becomes a **string prefix lookup** — the database can use a regular B-tree index on the GeoHash column.

```sql
-- Find all restaurants in the same ~1km cell as the user
-- User's GeoHash (6 chars): "dr5ru7"
SELECT * FROM restaurants
WHERE geohash LIKE 'dr5ru7%';
```

## Proximity Queries: The 9-Cell Search

A point near the **edge** of a cell might be closer to points in a neighboring cell than to points in its own cell. Searching only the target cell misses these.

```
┌──────────┬──────────┬──────────┐
│ NW       │ N        │ NE       │
│ dr5ru6   │ dr5ru9   │ dr5rud   │
├──────────┼──────────┼──────────┤
│ W        │ ★ Target │ E        │
│ dr5ru3   │ dr5ru7   │ dr5ruc   │
├──────────┼──────────┼──────────┤
│ SW       │ S        │ SE       │
│ dr5ru1   │ dr5ru4   │ dr5ru8   │
└──────────┴──────────┴──────────┘

★ = user's location in cell "dr5ru7"
Search all 9 cells to cover edge cases.
```

**Algorithm:**
1. Compute user's GeoHash at target precision (e.g., 6 chars)
2. Compute the 8 neighboring GeoHash cells
3. Query all 9 cells: `WHERE geohash IN ('dr5ru7%', 'dr5ru6%', 'dr5ru9%', ...)`
4. Post-filter: compute exact Haversine distance and discard results beyond the desired radius

```python
import geohash2 as geohash

def find_nearby(lat, lng, precision=6):
    center = geohash.encode(lat, lng, precision)
    neighbors = geohash.neighbors(center)  # returns 8 neighboring cells
    
    search_cells = [center] + neighbors  # 9 cells total
    
    # Query DB with prefix match on all 9 cells
    query = "SELECT * FROM restaurants WHERE " + " OR ".join(
        f"geohash LIKE '{cell}%'" for cell in search_cells
    )
    
    candidates = db.execute(query)
    
    # Post-filter by exact distance
    return [
        r for r in candidates
        if haversine(lat, lng, r.lat, r.lng) <= desired_radius_km
    ]
```

{{< callout type="warning" >}}
**Edge-case pitfall:** neighboring GeoHash cells don't always share a common prefix. In the grid above, `dr5ru7` and `dr5rud` are neighbors but differ in the last character. Near the boundaries of large cells (e.g., at the equator or the prime meridian), neighbors can have completely different prefixes. The 9-cell search with computed neighbors handles this correctly — never rely solely on prefix matching for proximity.
{{< /callout >}}

## Redis GEO Commands

Redis implements GeoHash internally and provides high-level geospatial commands. Under the hood, coordinates are stored as GeoHash-encoded scores in a sorted set.

```python
import redis
r = redis.Redis()

# Add locations — Redis encodes lat/lng as GeoHash internally
r.geoadd("restaurants", [
    (-74.0060, 40.7128, "joes-pizza"),      # lng, lat, member
    (-74.0052, 40.7135, "katz-deli"),
    (-73.9855, 40.7580, "central-park-cafe"),
])

# Find all restaurants within 1km of a point
nearby = r.georadius(
    "restaurants",
    longitude=-74.005,
    latitude=40.713,
    radius=1,
    unit="km",
    withcoord=True,    # return coordinates
    withdist=True,     # return distance
    count=10,          # limit results
    sort="ASC"         # nearest first
)

# Result: [("joes-pizza", 0.12, (-74.006, 40.7128)),
#          ("katz-deli", 0.08, (-74.0052, 40.7135))]
```

**How `GEORADIUS` works internally:**
1. Compute GeoHash for the center point at appropriate precision
2. Compute neighboring cells (the 9-cell search)
3. Scan the sorted set for members in those cells (using ZRANGEBYSCORE on GeoHash scores)
4. Filter by exact distance using the Haversine formula
5. Sort by distance and return top-k

| Command | Purpose |
|---------|---------|
| `GEOADD` | Add member with coordinates |
| `GEORADIUS` | Find members within radius of a point |
| `GEORADIUSBYMEMBER` | Find members within radius of another member |
| `GEODIST` | Distance between two members |
| `GEOHASH` | Return the GeoHash string for a member |
| `GEOPOS` | Return coordinates of a member |
| `GEOSEARCH` | (Redis 6.2+) Replaces GEORADIUS with more flexible search box/circle |

## Limitations

### Uneven Cell Sizes at High Latitudes

GeoHash cells are defined by equal angular divisions. Near the poles, 1° of longitude covers much less distance than at the equator. A GeoHash cell at 60°N latitude is roughly half as wide in meters as the same-precision cell at the equator.

```
At equator:  1° longitude ≈ 111 km
At 60°N:     1° longitude ≈ 55 km
At 80°N:     1° longitude ≈ 19 km

Same GeoHash precision = very different physical areas.
```

For most applications (ride-sharing, restaurant search) this doesn't matter because the population density at extreme latitudes is low. For global-scale systems, use **S2 Geometry** (Google) or **H3** (Uber) — both use cells with more uniform area.

### GeoHash vs Other Spatial Indexes

| Property | GeoHash | [QuadTree](../quadtree) | S2 / H3 |
|----------|---------|----------|---------|
| Cell shape | Rectangle | Rectangle (adaptive) | Hexagon (H3) / Sphere cap (S2) |
| Cell size uniformity | Varies with latitude | Adapts to data density | Near-uniform globally |
| Storage | String column with B-tree index | Custom in-memory tree | 64-bit cell ID with B-tree index |
| Range queries | 9-cell prefix search | Tree traversal | Cell union covering |
| Implementation | Simple — works with any DB | Custom server | Library required (S2/H3) |
| Best for | Moderate precision, simple infra | Dense/sparse mixed data | Global-scale, uniform precision |

{{< callout type="info" >}}
**Interview tip:** When asked about proximity search, start with: "I'd encode locations as GeoHash strings and store them with a B-tree index. For a radius query, I compute the 9 neighbouring GeoHash cells at a precision matching the radius (~6 chars for 1km) and query all 9, then post-filter by exact Haversine distance. For a production system at Uber/Google scale, I'd consider H3 or S2 for more uniform cell areas, but GeoHash with Redis GEO commands works well up to millions of points." This shows you understand the encoding, the 9-cell search trick, the post-filter step, and when to upgrade.
{{< /callout >}}