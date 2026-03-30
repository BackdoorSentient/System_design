# Case Study: Uber

---

**Q: What are the requirements for an Uber-like ride-sharing system?**

**Functional requirements:**
- Riders request a ride by specifying pickup and drop-off location
- The system matches the rider to a nearby available driver
- Real-time location tracking of the driver while en route and during the trip
- Fare estimation before booking; fare calculation after trip
- Surge pricing: increase fares when demand exceeds supply in an area
- Trip history, receipts, ratings

**Non-functional requirements:**
- **Low matching latency**: A rider should be matched to a driver in < 5 seconds
- **Real-time location accuracy**: Driver location must be updated frequently (every 4–5 seconds)
- **High availability**: Outage = lost revenue for both drivers and Uber. Target 99.99%.
- **Consistency for matching**: A driver must not be matched to two riders simultaneously
- **Scale**: 25M trips/day, 5M active drivers, 100M riders

---

**Q: How do you estimate scale for Uber?**

**Trips:**
- 25M trips/day = 25M / 86400 ≈ **290 trips/sec** average.
- Peak (3× average): ~870 trips/sec.

**Active drivers sending location updates:**
- Assume 2M drivers active simultaneously during peak.
- Each driver's app sends a GPS location update every 4 seconds.
- 2M / 4 = **500,000 location writes/sec** at peak.

**Location read (nearby driver lookups):**
- A rider requesting a ride triggers a geo search: "find all drivers within X km."
- 870 trip requests/sec, but in-app "how far away are drivers?" polls happen more frequently.
- Assume 5M rider apps active, each polling every 5 seconds: 5M / 5 = **1M geo-reads/sec**.

**Key insight from estimation:** The **location write throughput** (500k writes/sec) and **geo-queries** (1M reads/sec) are the hardest problems. Everything else (trip management, payments) is at much lower scale.

---

**Q: How do you store and query driver locations in real-time? This is the core technical challenge.**

You need to answer the query: **"Give me all available drivers within 5 km of this lat/lng"** — extremely fast, from a dataset that changes 500,000 times per second.

**Why standard SQL or naive approaches fail:**
- `SELECT * FROM drivers WHERE lat BETWEEN x1 AND x2 AND lng BETWEEN y1 AND y2` requires a 2D index. B-trees are 1D — they can't efficiently intersect on two columns simultaneously.
- Without spatial indexing, this is a full table scan of 2M rows every query.

**Solution: Geohashing**

Geohash divides the Earth's surface into a hierarchy of rectangular cells, each identified by a short alphanumeric string.

- Each geohash character adds precision. A 6-character geohash covers a ~1.2km × 0.6km area. A 7-character geohash covers ~153m × 153m.
- Nearby locations share a common prefix: `dp3wjz` and `dp3wjz2` are in the same cell.
- Algorithm for finding nearby drivers:
  1. Compute the rider's geohash at precision 6 (or 7).
  2. Identify the 9 geohash cells that cover the rider's cell + 8 neighbours (to avoid edge effects).
  3. Find all drivers whose current geohash prefix matches any of the 9 cells.

**Driver location storage with geohashing (Redis):**

```
Key: geo:drivers:{geohash_prefix_6}
Type: Hash
Field: driver_id
Value: {lat, lng, timestamp, status}
```

On each location update:
1. Compute new geohash (6-char prefix) for the driver.
2. If geohash changed: remove driver from old geohash bucket, add to new one.
3. Update driver's position in the new bucket.

On rider request:
1. Compute rider's geohash.
2. Enumerate 9 neighbouring cells.
3. `HGETALL geo:drivers:{cell}` for each of 9 cells.
4. Post-filter: compute exact haversine distance for each candidate, return those within radius.

**Redis GEOADD / GEORADIUS (native geo commands):**

Redis has first-class geospatial support using an internal geohash-based sorted set:

```
GEOADD drivers_location {lng} {lat} {driver_id}
GEORADIUS drivers_location {lng} {lat} 5 km ASC COUNT 10
```

This is what Uber's early architecture closely resembled. Redis GEORADIUS is backed by a sorted set with a 52-bit geohash as score — O(N+log M) where N is results and M is total entries.

**Scale:** 2M driver entries × ~50 bytes each = 100 MB. Trivially fits in a single Redis instance. With replication for HA, this is a lightweight data store.

---

**Q: How does the matching algorithm work? How do you ensure a driver isn't matched to two riders?**

Finding nearby drivers is step 1. Matching is step 2 — selecting the best driver and locking them.

**Matching criteria (simplified):**
- Distance (primary): closest available driver within acceptable range
- ETA (more accurate than distance — considers traffic)
- Driver acceptance rate (prefer drivers who are likely to accept)
- Match on vehicle type (UberX, UberBlack, etc.)

**The matching service flow:**
```
1. Rider requests ride at location L
2. Matching Service calls Location Service: get drivers within 5 km of L
3. Filter: only AVAILABLE status drivers
4. Score and rank candidates
5. Offer the trip to the top-ranked driver
6. Wait up to 15 seconds for acceptance
7. If declined (or timeout): offer to the next candidate
8. If no drivers accept after N attempts: notify rider of no availability
```

**Preventing double-booking — distributed locking:**

The critical section: between "offer trip to driver X" and "driver X accepts/declines," driver X must not be offered to another rider.

- When a trip is offered to a driver, write `driver:{driver_id}:status = PENDING_OFFER` with TTL = 15s.
- Use `SET driver:{driver_id}:status PENDING_OFFER NX PX 15000` (Redis SET with NX = only set if not exists).
- If the NX SET fails, the driver is already locked — skip to the next candidate.
- On acceptance: change status to `ON_TRIP`.
- On decline or timeout: the TTL expires, driver returns to `AVAILABLE`.

This is a distributed lock via Redis. The 15-second TTL ensures the lock auto-releases if the driver goes offline without responding.

---

**Q: How do you handle real-time location tracking during a trip?**

Once a trip starts, both the rider's app and backend need to track the driver's location continuously for:
1. Display the driver's car moving on the rider's map
2. ETA updates
3. Fraud detection (did the driver take the correct route?)
4. Fare calculation (distance-based pricing)

**Location update pipeline:**

```
Driver app → GPS reading every 4 seconds
           → WebSocket / HTTP POST to Location Ingestion Service
           → Kafka (partitioned by driver_id for ordering)
           → Location Consumer Service
              ↙             ↘
        Redis             Trip DB (record path for fare calc)
   (current position)
           ↓
     WebSocket push to rider app (driver dot moves on map)
```

**Why Kafka between ingestion and consumers?**
- Ingestion is at 500k writes/sec. Kafka buffers spikes, decouples producers from consumers.
- Multiple consumers can process location updates independently (map display, ETA service, audit log).
- Kafka partitioned by `driver_id` guarantees ordered delivery per driver (no out-of-order GPS points).

**Pushing driver location to rider:**
- When a rider has an active trip, they subscribe to a WebSocket channel for their driver.
- The Location Consumer pushes updates to a pub/sub (Redis pub/sub or a push gateway).
- The push gateway fans out to the rider's WebSocket connection.
- Update frequency: every 4–5 seconds during trip. At 25M trips/day, peak concurrent trips ~300k.
- 300k active trips × 1 push/4s = **75,000 location pushes/sec** to rider apps.

---

**Q: How does surge pricing work, and how do you compute it in real-time?**

Surge pricing (Uber calls it "dynamic pricing") increases the fare multiplier in an area when demand exceeds supply.

**Core formula:**
```
surge_multiplier = f(demand / supply) in a geographic cell
```

Where demand = ride requests per minute, supply = available drivers.

**Implementation:**

1. **Define surge zones**: Geohash cells (or manually-drawn polygons for cities) are the pricing units.

2. **Real-time demand signal**: Count ride requests in each geohash cell over a sliding window (last 5 minutes).
   - Ride requests → Kafka → Streaming job (Flink / Spark Streaming) → per-cell demand counter in Redis.

3. **Real-time supply signal**: Count available drivers in each geohash cell.
   - Same driver location index (from the Location Service) provides supply counts.

4. **Surge calculation**:
   - `demand / supply > threshold` → apply surge multiplier.
   - Multiplier is quantised (1.0x, 1.2x, 1.5x, 2.0x, ...) to reduce user confusion.
   - Calculated every 1–5 minutes per zone.

5. **Display surge to user**:
   - When a rider opens the app, their geohash cell's surge multiplier is fetched and shown.
   - Surge is shown before booking and included in the fare estimate.

**Key engineering constraints:**
- Surge multipliers are read by every rider opening the app → cache aggressively in Redis with short TTL (30–60 seconds).
- Must be eventually consistent — a 30-second-stale surge reading is acceptable.
- The demand/supply ratio is approximate; exact real-time counts are not required.

---

**Q: How do you calculate and store trip fares?**

**During the trip:**
- GPS path is recorded in the Trip Service (append to a Cassandra time-series table: `{trip_id, timestamp, lat, lng}`).

**At trip end:**
```
Trip ends
    ↓
Fare Engine:
  distance = sum of haversine distances between consecutive GPS points
  duration = trip_end_time - trip_start_time
  base_fare = base + (per_km × distance) + (per_min × duration)
  surge_multiplier = multiplier applied at time of booking (locked in at booking)
  final_fare = base_fare × surge_multiplier
    ↓
Payment Service → charge rider's saved payment method
    ↓
Receipt generated → push notification + email
```

**GPS noise and fare integrity:**
- Raw GPS has ~5–10m error. A noisy GPS path can add phantom distance.
- Uber applies **map matching**: snap GPS points to road segments using the road network graph (OSRM or proprietary). This both corrects GPS noise and filters impossible paths.
- The matched path is what fare distance is calculated from.

**Dispute handling:**
- Original GPS path and map-matched path are both stored.
- If a rider disputes the fare, the support team can replay the trip on a map.

---

**Q: How does ETA calculation work?**

ETA = time to pick up + time to complete trip. Both depend on real-time traffic.

**Components:**
1. **Road network graph**: A pre-built graph of roads (nodes = intersections, edges = road segments with travel time weights).
2. **Historical travel time**: Statistical baseline for each road segment at each time of day/day of week.
3. **Real-time traffic**: Aggregate the GPS tracks of all Uber drivers (and riders) currently moving on the road network to detect slowdowns.

**Dijkstra / A* shortest path:**
- Compute the fastest route from driver to rider, and from rider to destination.
- ETA = sum of travel times on the fastest path.
- Re-computed every 30–60 seconds as traffic conditions change.

**At Uber's scale:**
- The road graph for a city has millions of nodes and edges.
- Running Dijkstra per trip request, re-computing every minute = enormous compute.
- Uber pre-computes and caches routes for common origin-destination pairs.
- Uses **time-dependent shortest paths**: the edge weight (travel time) varies by time of day — rush-hour routes are pre-computed with rush-hour weights.

---

**Q: What does the full system architecture look like?**

```
Driver App
    ↓ (GPS update every 4s)
Location Ingestion Service (WebSocket / HTTP)
    ↓
Kafka (partitioned by driver_id)
    ↓
Location Consumer
    ↙                   ↘
Redis GEOADD         Trip DB (Cassandra: path recording)
(driver positions)        ↓
                      Fare Engine (on trip end)

Rider App
    ↓ (ride request)
API Gateway → Matching Service
                   ↓
         Location Service (GEORADIUS from Redis)
                   ↓
         Rank candidates → Redis distributed lock (NX SET per driver)
                   ↓
         Trip Service (create trip record in MySQL)
                   ↓
         Notify driver app (WebSocket push)

During trip:
Driver GPS → Kafka → Location Consumer → Redis pub/sub → Rider app (WebSocket, driver dot)

Surge:
Ride requests → Kafka → Flink → Redis (demand counters per geohash)
Driver locations → supply counts → Surge Engine → Redis (multipliers per geohash)
Rider opens app → fetch surge multiplier from Redis → display in UI
```

**Key decisions summarised:**

| Decision | Choice | Reason |
|----------|--------|--------|
| Driver location storage | Redis GEOADD / geohash | O(log N) geo-queries, 500k writes/sec |
| Matching lock | Redis NX SET with TTL | Prevents double-booking; auto-expires |
| Location update pipeline | Kafka → consumers | Decouples ingestion from processing; ordered by driver |
| Surge pricing | Flink streaming + Redis counters | Near-real-time demand/supply per zone |
| Trip path storage | Cassandra (time-series) | Append-only, high write throughput |
| ETA computation | Pre-computed time-dependent graph | Low latency at query time |
| GPS map matching | OSRM / road graph | Corrects GPS noise for accurate fare |
| Rider location push | WebSocket + Redis pub/sub | < 1s update latency for moving car dot |