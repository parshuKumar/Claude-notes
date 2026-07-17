# 102 — Design Uber / Ride Sharing
## Category: HLD Case Study

---

## What is this?

Uber is a **matchmaker for moving metal boxes**. A million drivers wander around a city, each one shouting "I'm here! I'm here!" every few seconds. A rider stands on a corner and says "I want a car, right now, right here." The system's whole job is to answer one question, fast: **"which of these million dots is closest to that one dot?"**

Everything else — the fare, the payment, the star rating, the map that shows a little car creeping toward you — is ordinary CRUD software you already know how to build. The hard, interesting, interview-winning part is that one geometric question, asked five thousand times a second, against a dataset that is being rewritten a quarter of a million times a second.

That's the doc. **Geospatial indexing.** Everything else is supporting cast.

---

## Why does it matter?

Because Uber is the canonical **write-heavy, low-latency, geometry-flavoured** system, and it breaks almost every instinct you've built from designing Twitter or a URL shortener.

Nearly every system you've studied so far is **read-heavy**: you write a tweet once and it's read a million times, so you cache aggressively and fan out on write. Uber is the opposite. The dominant load is drivers hammering the system with location pings that almost nobody ever reads. If you walk into the interview with your "add a cache, add a read replica" reflex, you will design the wrong system and the interviewer will watch you do it.

**In interviews:** "Design Uber" (or Lyft, or DoorDash, or Swiggy, or Google Maps' nearby-search) is a top-5 HLD question at every large company. The interviewer is specifically probing whether you know that **2D proximity search is a genuinely hard indexing problem** and whether you've heard of geohashing, quadtrees, or H3. If you say "I'd query Postgres with a lat/lng BETWEEN clause," the interview is effectively over.

**At work:** any time you build "stores near me," "restaurants delivering to this postcode," "which delivery partner should take this order," "show me all sensors within 500m of this alarm" — you are building a scaled-down Uber. The same index choices apply.

---

## The core idea — explained simply

### The Chessboard Analogy

Imagine a giant chessboard laid over a city map. Each square is about a kilometre across. Now imagine every driver in the city is a grain of rice sitting on that board, and every four seconds, every grain jumps to wherever its car actually is.

A rider stands somewhere and asks: **"give me the grains of rice near me."**

The naive way: pick up every one of the million grains, measure its distance to the rider with a ruler, and keep the closest ones. That's a **full scan** — a million distance calculations for one ride request. At 5,000 ride requests a second, that's 5 billion distance calculations a second. Dead on arrival.

The clever way: **label the squares.** Give every square on the chessboard a short name — `gcpvj`, `gcpvk`, and so on. Now, when a grain of rice lands, you don't remember its exact coordinates for search purposes; you remember **which square it's in**. Then the rider's question becomes trivially cheap:

> "What square am I standing in? `gcpvj`. Give me everything labelled `gcpvj` — plus the eight squares touching it, because I might be standing right next to the edge."

You went from *a million distance calculations* to *nine dictionary lookups*. That's the entire idea behind every geospatial index ever built.

**The mapping back to the tech:**

| Chessboard thing | Technical thing |
|---|---|
| The board laid over the city | A **spatial index** — a partitioning of the Earth's surface |
| One square | A **cell** (geohash cell, quadtree node, H3 hexagon) |
| The square's short name | The **cell ID / geohash string** — a 1-dimensional key |
| A grain of rice | A driver's current location |
| Grains re-jumping every 4s | The 250k writes/second location firehose |
| "Give me everything in square X" | A hash-map or sorted-set lookup — **O(1)-ish**, not O(n) |
| Also checking the 8 touching squares | The **boundary problem** — the #1 thing candidates forget |
| Squares near dense downtown being subdivided smaller | A **quadtree** — density-adaptive cells |

The deep insight to say out loud in the interview:

> **A spatial index is a machine for turning a 2-dimensional proximity problem into a 1-dimensional key lookup.** Databases are extremely good at 1D key lookups (B-trees, hash maps, sorted sets). They are terrible at 2D range scans. So we don't make the database smarter — we *reshape the problem* until it becomes something the database is already good at.

---

## Key concepts inside this topic

### 1. Requirements

**Clarifying questions to ask the interviewer first** (always do this — it's scored):

- "Are we designing for a single city or globally?" (Assume global, but sharded by city/region — cities are natural partitions and rides almost never cross them.)
- "Do we need to handle pooled rides / UberPool?" (Say: I'll design solo rides, and mention pooling as an extension.)
- "Is the map/routing engine given to us, or do we build it?" (Assume a Maps service exists that can give us road-network ETAs. Building a routing engine is a separate 45-minute question.)
- "How fresh must driver locations be?" (Push for: a few seconds of staleness is fine. This unlocks the whole design.)

**Functional requirements:**

1. **Drivers continuously broadcast location.** Every active driver's app sends its GPS coordinates every ~4 seconds while online.
2. **A rider requests a ride** from a pickup point to a destination.
3. **The system matches** the rider to a nearby, available driver.
4. **Both parties see each other's live location** on a map during pickup and during the trip.
5. **Fare calculation** — base fare + distance + time + surge multiplier.
6. **Trip lifecycle** — request, match, arrive, start, complete, pay, rate.

**Non-functional requirements** (this is where the design is actually decided):

| Requirement | What it forces |
|---|---|
| **VERY HIGH WRITE THROUGHPUT** | This is the headline. Location updates are the dominant load by 50×. Most systems you've designed are read-heavy — **this one is write-heavy**, and that inverts your usual toolkit. Say this out loud in the interview. |
| **Low-latency matching** (< 1–2 seconds to find candidates) | A rider staring at a spinner for 10 seconds opens a competitor's app. The geo query must be sub-100ms. |
| **High availability** | If matching is down, the business is down. Prefer availability over consistency for location data. |
| **Location data can be stale** | A driver's dot being 2 seconds old is harmless. **Eventual consistency is fine.** This is a gift — take it. |
| **Trip and payment data must be strongly consistent and durable** | You cannot "eventually" charge someone. This half of the system is boring, ACID, transactional. |

Notice the requirements split the system cleanly in two: a **hot, sloppy, in-memory, write-crushed half** (locations, matching) and a **cold, careful, durable half** (trips, payments). Almost every design decision follows from that split.

---

### 2. Capacity estimation — the one number that defines the architecture

Let's assume:

- **1,000,000 active drivers** online at peak (globally, at a busy hour).
- Each sends a location update every **4 seconds**.

**Location write throughput:**

```
1,000,000 drivers ÷ 4 seconds
= 250,000 location writes per second
```

Stop and look at that number. **250,000 writes/second.**

For comparison, a well-tuned single Postgres node handles roughly **5,000–10,000 simple writes/second** before it starts groaning (WAL fsyncs, index maintenance, vacuum). You would need **25–50 fully saturated Postgres primaries** doing nothing but overwriting rows — and each write invalidates an index entry, so it's worse than that in practice.

**Conclusion, stated explicitly:** you cannot put driver locations in a disk-based relational database. Not with sharding, not with clever indexes. The write rate alone kills it. **Location data must live in an in-memory store.** Redis handles ~100k+ ops/sec per node comfortably; five to ten shards absorbs 250k/s with headroom.

**Ride requests:**

```
Say 5,000 ride requests per second at peak.
```

That's **50× fewer** than location writes. Let that sink in: the thing users think of as "the product" (requesting a ride) is 2% of the load. The invisible background chatter is 98%.

**Payload size per location update:**

```
driverId (16B) + lat (8B) + lng (8B) + heading (2B) + timestamp (8B) + overhead
≈ 100 bytes on the wire (with framing/TLS, call it ~200 bytes)

250,000 × 200 bytes = 50 MB/second ingress
                    = 4.3 TB/day of raw location chatter
```

That's a lot of *bandwidth*, but almost none of it needs to be *stored*.

**Storage — and the beautiful simplification:**

Here's the key realisation:

> **The driver location index does not need to be durable.**

If the entire Redis geo-index evaporates, what happens? Nothing much. **Within 4 seconds, every driver re-reports and the index rebuilds itself.** You lose maybe one round of matching for a few seconds. There is no data loss, because the "source of truth" for where a driver is... is the driver, and they're about to tell you again anyway.

This is a genuinely elegant property and worth stating loudly in an interview: **the hot dataset is self-healing.** So: no replication for durability, no AOF fsync, no WAL. Just RAM.

Memory footprint of the live index:

```
1M drivers × ~100 bytes per entry (id + geohash score + metadata)
≈ 100 MB — 200 MB with Redis overhead.
```

**One hundred megabytes.** The entire live position of every driver on Earth fits comfortably in the RAM of a laptop. The problem was never *size*. It was always *write rate* and *query shape*.

**Durable storage (trips):**

```
Say 20 million completed trips/day globally.
Each trip row ≈ 1 KB (ids, timestamps, coords, fare, status, route polyline ref)

20M × 1KB       = 20 GB/day
× 365           = ~7.3 TB/year
× 5 years       = ~36 TB
```

36 TB over five years is very manageable for a sharded Postgres or a Cassandra cluster. **The durable half of the system is not the hard part.**

---

### 3. THE CORE PROBLEM: "find all drivers within 3km of this point"

This is the heart of the doc. Everything above was setup.

#### Why the naive approach fails

Your first instinct is probably this:

```sql
SELECT driver_id
FROM drivers
WHERE lat BETWEEN 37.77 AND 37.80
  AND lng BETWEEN -122.43 AND -122.39
  AND status = 'AVAILABLE';
```

This is wrong in a way that's worth understanding deeply, because the *reason* it's wrong is the reason geohashing exists.

**Problem 1 — without an index, it's a full table scan.** A million rows, every ride request. At 5,000 requests/sec, that's 5 billion row-touches per second. No.

**Problem 2 — and this is the subtle one — a B-tree index doesn't save you.** You might think "fine, I'll add `CREATE INDEX ON drivers (lat, lng)`." But a **B-tree is a 1-dimensional, ordered structure.** It can efficiently answer "give me everything where the key is between A and B" *along one axis*.

With a composite index on `(lat, lng)`, the database can use `lat` to narrow down — but `lat BETWEEN 37.77 AND 37.80` is a **band that wraps the entire planet**. Every driver in that latitude band, from San Francisco to Seville to Seoul, is a candidate. The database pulls that whole band and then filters on `lng` row-by-row. You've turned a 2D search into a 1D range scan of a globe-circling strip.

Draw it:

```
   A B-TREE ON `lat` CAN ONLY GIVE YOU A HORIZONTAL BAND
   ═══════════════════════════════════════════════════════
   
   -180°                                              +180°
     ┌───────────────────────────────────────────────────┐
     │                                                   │
     │                                                   │
   ┌─┼───────────────────────────────────────────────────┼─┐ ← lat 37.80
   │ │ ▓▓▓▓▓▓▓▓▓▓▓ [★] ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓ │ │
   └─┼───────────────────────────────────────────────────┼─┘ ← lat 37.77
     │            ↑                                      │
     │       what you WANT                               │
     │   (a small box around ★)                          │
     └───────────────────────────────────────────────────┘

   ▓ = rows the index actually hands you (the whole band)
   ★ = the rider
   
   You wanted a 3km box. You got a 40,000km strip. Then you filter
   by hand. That is not an index — that's a scan with extra steps.
```

**The fundamental difficulty, stated precisely:**

> **Proximity is a 2-dimensional relationship, but every fast index we own (B-tree, hash map, sorted set) is 1-dimensional.** The job of a geospatial index is to build a *mapping from 2D space to a 1D key* that **preserves locality** — meaning points that are near each other in 2D get keys that are near each other in 1D.

Once you have such a mapping, you can use the boring, battle-tested 1D index you already have. That mapping is called a **space-filling curve**, and geohash is the most famous one.

---

#### Solution A: Geohash

**The idea:** recursively cut the world in half, and record each cut as a bit.

1. Start with the whole world: longitude `[-180, +180]`, latitude `[-90, +90]`.
2. Look at longitude. Is your point in the left half or the right half? Left → write `0`. Right → write `1`. Now your longitude range is half as wide.
3. Now do latitude. Bottom half → `0`. Top half → `1`.
4. Alternate longitude, latitude, longitude, latitude... each bit **doubles your precision**.
5. Take the resulting bit string, chunk it into groups of 5 bits, and encode each group as one character in **base-32**.

The result is a short string like `9q8yyk`.

**The magic property — and this is the whole point:**

> **Two points that are physically near each other share a common string PREFIX.**

Because every character you add is just "more cuts, in the same region," a longer shared prefix means both points survived the same sequence of halvings — which means they're in the same shrinking box.

```
   GEOHASH = RECURSIVE HALVING OF THE WORLD
   
   Level 1 (1 char, ~5000km)        Level 2 (2 chars, ~1250km)
   ┌─────────┬─────────┐            ┌────┬────┬─────────┐
   │         │         │            │ 9q │ 9w │         │
   │    9    │    d    │    ───▶    ├────┼────┤    d    │
   │         │         │            │ 9n │ 9p │         │
   ├─────────┼─────────┤            ├────┴────┼─────────┤
   │    3    │    6    │            │    3    │    6    │
   └─────────┴─────────┘            └─────────┴─────────┘

   Level 6 (6 chars, ~1.2km)        Level 7 (7 chars, ~150m)
   ┌──┬──┬──┬──┬──┬──┐              a single 1.2km cell,
   │  │  │  │  │  │  │              chopped into 32 smaller
   ├──┼──┼──┼──┼──┼──┤     ───▶     boxes, each ~150m across
   │  │9q│8y│yk│  │  │              9q8yyk0, 9q8yyk1, ...
   ├──┼──┼──┼──┼──┼──┤              9q8yykz
   │  │  │  │  │  │  │
   └──┴──┴──┴──┴──┴──┘
   
   SF Ferry Building = 9q8yyk...    Coit Tower = 9q8yyz...
                       ^^^^                       ^^^^
                       shared prefix "9q8yy" = they're within ~1km
```

**Precision table — memorise the middle rows:**

| Geohash length | Cell width × height (approx) | Useful for |
|---|---|---|
| 1 | 5,000 km × 5,000 km | Continent |
| 2 | 1,250 km × 625 km | Large country |
| 3 | 156 km × 156 km | Metro region |
| 4 | 39 km × 19.5 km | City |
| 5 | 4.9 km × 4.9 km | Neighbourhood / district |
| **6** | **1.2 km × 0.6 km** | **Ride matching search radius — the sweet spot** |
| **7** | **153 m × 153 m** | **Street block / dense downtown** |
| 8 | 38 m × 19 m | Building |
| 9 | 4.8 m × 4.8 m | Parking spot |

Now the query is embarrassingly simple. You need drivers near `9q8yyk`? That's:

```sql
SELECT driver_id FROM driver_locations WHERE geohash LIKE '9q8yyk%';
```

A **prefix scan on an ordinary string B-tree index.** No geometry. No trigonometry. No full scan. The database was always good at this — we just had to ask the question in a language it understands.

#### The edge case interviewers love: the boundary problem

Here's the flaw, and if you raise it unprompted you get serious credit.

**Two points can be five metres apart and have completely different geohashes.** Why? Because they sit on opposite sides of a cell boundary. Recursive halving means a boundary at *any* level of the recursion produces a divergence at *that character*, and everything after it.

```
   THE BOUNDARY PROBLEM
   
        cell "9q8yyj"  │  cell "9q8yyk"
                       │
       ┌───────────────┼───────────────┐
       │               │               │
       │               │  ★ rider      │
       │        🚗  ←5m→ │               │
       │       driver   │               │
       │               │               │
       └───────────────┼───────────────┘
                       │
                    boundary
   
   The driver is 5 METRES from the rider.
   Their geohashes differ in the LAST character: ...yyj vs ...yyk.
   
   A naive `WHERE geohash = '9q8yyk'` query MISSES this driver entirely,
   while happily returning a driver 1.1km away at the far corner of the
   same cell.
   
   THE FIX: always query the target cell AND ITS 8 NEIGHBOURS.
   
       ┌──────┬──────┬──────┐
       │  NW  │  N   │  NE  │
       ├──────┼──────┼──────┤
       │  W   │ SELF │  E   │   ← query all 9 cells, union the results,
       ├──────┼──────┼──────┤     THEN compute real distances and filter
       │  SW  │  S   │  SE  │
       └──────┴──────┴──────┘
   
   9 cells × ~1.2km each = a ~3.6km search window. Perfect for a 3km radius.
```

Every real geohash library ships a `neighbors(hash)` function precisely because of this. If a candidate says "I'll just query the one cell," the interviewer will draw this picture and ask them to find the bug.

**Also note:** cells shrink toward the poles (longitude lines converge), so a level-6 cell in Norway is narrower than one in Singapore. Rarely matters for ride-hailing, but it's a nice detail to know.

---

#### Solution B: QuadTree

Geohash uses a **fixed** grid: every level-6 cell is the same size, whether it's Times Square (4,000 drivers) or the Nevada desert (0 drivers). That's wasteful in both directions — Times Square's cell returns far too many candidates, and the desert cell wastes an entry on nothing.

A **quadtree** fixes this by subdividing **only where density demands it**.

- Start with one square node covering the whole city.
- Insert drivers into it.
- When a node exceeds a threshold (say, 100 drivers), **split it into 4 quadrants** (NW, NE, SW, SE) and redistribute its drivers.
- Recurse.

The result: dense Manhattan subdivides 12 levels deep into tiny cells; empty desert stays one enormous node. **Every leaf holds roughly the same number of drivers**, which means every query does roughly the same amount of work regardless of where in the world it's asked. That's a very nice property.

```
   QUADTREE — SPLITS ONLY WHERE IT'S CROWDED
   
   ┌───────────────────────────┬───────────────────────────┐
   │                           │             │             │
   │                           │      •      │  •      •   │
   │                           │             │             │
   │        DESERT             ├──────┬──────┼─────────────┤
   │      (0 drivers)          │ •• │•│      │             │
   │                           ├────┼─┼──────┤  SUBURBS    │
   │     one huge leaf         │•│••│ │  •   │             │
   │                           ├─┴──┴─┴──────┤             │
   │                           │  DOWNTOWN   │             │
   │                           │ (deep split)│             │
   ├───────────────────────────┼─────────────┴─────────────┤
   │                           │                           │
   │        WATER              │        SUBURBS            │
   │                           │                           │
   └───────────────────────────┴───────────────────────────┘
   
   • = a driver.  Notice: crowded areas → many tiny cells.
                          empty areas   → one big cell.
```

**The cost:** a quadtree is a pointer-based tree held **in memory**, and drivers move. Every 4 seconds, a driver may cross a cell boundary, which means **removing it from one leaf and inserting it into another** — and if a leaf drops below threshold, merging nodes back together. At 250k updates/sec, that tree is being surgically rewritten constantly. It works (you keep a `driverId → leafNode` hash map so removal is O(1)), but it's a mutable in-memory structure that you must build, shard, and rebuild after a crash yourself.

Geohash needs none of that machinery — it's a pure function from `(lat, lng) → string`. That simplicity is why geohash wins in practice for the *hot path*.

---

#### Solution C: S2 (Google) and H3 (Uber's actual system)

**S2** (Google) projects the Earth onto the six faces of a cube, then uses a **Hilbert curve** (a better space-filling curve than geohash's Z-order — it has no long "jumps," so locality is preserved more consistently) to number cells within each face. It handles poles and the antimeridian correctly. It's what Google Maps uses.

**H3** is the one Uber actually built and open-sourced — and it uses **hexagons** instead of squares.

**Why hexagons? Here's the detail that wins interviews:**

Take a square cell. It has 8 neighbours: 4 that share an *edge* (N, S, E, W) and 4 that share only a *corner* (NE, NW, SE, SW). The distance from the square's centre to an edge-neighbour's centre is `d`. The distance to a corner-neighbour's centre is `d × √2 ≈ 1.41d`.

So **"adjacent" is ambiguous with squares.** Two different neighbours are two different distances away. Any calculation that means "spread out to my neighbours" — smoothing surge prices, modelling how demand flows across a city, computing a k-ring around a point — has to constantly correct for this, or it produces distorted, blocky, diagonal-biased results.

With hexagons, **all 6 neighbours are exactly equidistant.** There are no corner-neighbours. "Adjacent" means one thing.

```
   SQUARE GRID                      HEXAGONAL GRID (H3)
   
   ┌────┬────┬────┐                    ⬡     ⬡
   │ NW │ N  │ NE │                 ⬡  ┌─────┐  ⬡
   ├────┼────┼────┤                   ╱       ╲
   │ W  │ ★  │ E  │                  │    ★    │
   ├────┼────┼────┤                   ╲       ╱
   │ SW │ S  │ SE │                 ⬡  └─────┘  ⬡
   └────┴────┴────┘                    ⬡     ⬡
   
   N is distance  d                  ALL SIX neighbours are
   NE is distance d×1.41              at distance d. Exactly.
   
   "adjacent" = two               "adjacent" = one thing.
    different distances            Flow, smoothing, and k-ring
    → distorted flow maps           calculations are clean.
```

For a company whose core business is **modelling the flow of supply and demand across a city grid**, that's not a cosmetic choice — it's why the surge-pricing and driver-positioning maths comes out cleaner. Mention this and watch the interviewer's eyebrows go up.

---

#### Solution D: Redis GEO — the practical answer

In a real system you don't hand-roll any of this. Redis has geospatial commands built in, and — delightfully — **they're implemented on top of sorted sets, using a 52-bit geohash as the score.** The theory you just learned *is* the implementation.

```
GEOADD drivers:sf  -122.4194 37.7749  driver:1234
GEOADD drivers:sf  -122.4089 37.7837  driver:5678

# "Who's within 3km of the rider, nearest first, max 20?"
GEOSEARCH drivers:sf FROMLONLAT -122.4194 37.7749 BYRADIUS 3 km ASC COUNT 20 WITHDIST

# Under the hood this is a ZRANGEBYSCORE over geohash prefixes,
# plus neighbour cells, plus a real haversine distance filter.
# Redis handles the boundary problem FOR you.
```

Note `drivers:sf` — the key is **per-city**. That's your shard key. Rides essentially never cross city boundaries, so cities are a natural, perfectly-balanced partition (recall **[75 — Data Partitioning]**). Each city's index is small enough to live entirely on one Redis node, and you scale by adding cities, not by resharding.

---

### 4. High-level architecture

```
 ┌──────────────┐                                   ┌──────────────┐
 │  DRIVER APP  │                                   │  RIDER APP   │
 │              │                                   │              │
 │ GPS ping     │                                   │ "get me a    │
 │ every 4 sec  │                                   │  car"        │
 └──────┬───────┘                                   └──────┬───────┘
        │                                                  │
        │ WebSocket (persistent — recall 82)               │ HTTPS
        │ 250,000 msgs/sec                                 │ 5,000 req/sec
        ▼                                                  ▼
 ┌──────────────────────┐                          ┌──────────────────┐
 │   LOCATION SERVICE   │                          │   RIDE SERVICE   │
 │ (stateless, many     │                          │  (validates,     │
 │  instances, LB'd by  │                          │   creates trip   │
 │  consistent hashing) │                          │   in REQUESTED)  │
 └───┬──────────────┬───┘                          └────────┬─────────┘
     │              │                                       │
     │ hot path     │ cold path (async, OFF the hot path)   │
     │ (sync, fast) │                                       ▼
     ▼              ▼                              ┌──────────────────┐
┌─────────────┐  ┌────────────────┐                │ MATCHING SERVICE │
│    REDIS    │  │     KAFKA      │                │                  │
│  GEO INDEX  │  │ location-events│                │ 1. geo query     │
│             │  │    topic       │                │ 2. filter        │
│ GEOADD      │  │  (recall 67)   │                │ 3. rank by ETA   │
│ GEOSEARCH   │  └───────┬────────┘                │ 4. offer w/ 15s  │
│             │◀─────────┼─────────────────────────┤    timeout       │
│ ~100MB      │  QUERY   │                         │ 5. ATOMIC claim  │
│ EPHEMERAL   │          │                         └────┬─────────┬───┘
│ (self-heals │          ▼                              │         │
│  in 4 sec!) │  ┌────────────────┐                     │         │
└─────────────┘  │ STREAM PROCESS │                     │         │
                 │ (surge, ETA    │                     ▼         ▼
                 │  models, trip  │            ┌────────────┐ ┌──────────┐
                 │  history, ML)  │            │NOTIFICATION│ │   TRIP   │
                 └───────┬────────┘            │ / DISPATCH │ │ SERVICE  │
                         ▼                     │ (WebSocket │ │ (state   │
                 ┌────────────────┐            │  push to   │ │  machine)│
                 │  DATA LAKE /   │            │  driver)   │ └────┬─────┘
                 │  CASSANDRA     │            └────────────┘      │
                 │ (raw traces)   │                                ▼
                 └────────────────┘                        ┌───────────────┐
                                                           │   POSTGRES    │
                                                           │ trips, users, │
                                                           │  payments     │
                                                           │  (ACID, dur-  │
                                                           │   able, cold) │
                                                           └───────┬───────┘
                                                                   ▼
                                                           ┌───────────────┐
                                                           │PAYMENT SERVICE│
                                                           │ (idempotent — │
                                                           │  recall 85)   │
                                                           └───────────────┘
```

**Reading the diagram:**

- **The left column is the firehose.** 250k/sec of driver pings arrive over **WebSockets**, not HTTP. Why? Because opening a fresh TCP + TLS connection for a 100-byte payload every 4 seconds is absurd — the handshake costs more than the data. A persistent connection also lets the server **push** to the driver (the ride offer) without polling (recall **[82 — Content Negotiation and Protocols]**).
- **Two paths diverge at the Location Service.** The **hot path** writes to Redis synchronously — that's a single `GEOADD`, sub-millisecond. The **cold path** fires the same event into **Kafka** (recall **[67 — Message Queues]**) and forgets about it. Analytics, ML training, surge computation, and trip-trace history all consume from Kafka. **Nothing that isn't needed to match a rider in the next second is allowed on the hot path.** This is the single most important architectural discipline in the diagram.
- **The right column is boring, careful software.** Trips and payments are ACID, on disk, in Postgres. Small volume, high stakes.
- **The Matching Service is the bridge** — it reads from the hot half and writes to the cold half.

---

### 5. API design

```
POST /v1/drivers/location          (actually a WebSocket frame, not HTTP)
  { driverId, lat, lng, heading, speed, ts }
  → no response body. Fire and forget.

POST /v1/rides
  { riderId, pickup: {lat,lng}, dropoff: {lat,lng}, vehicleType, idempotencyKey }
  → 202 Accepted { rideId, status: "SEARCHING", etaSeconds: null }

GET  /v1/rides/:rideId
  → { rideId, status, driver: {id,name,car,rating,lat,lng}, etaSeconds, fare }

WS   /v1/rides/:rideId/track       (server pushes driver position ~1/sec)
  ← { lat, lng, heading, etaSeconds }

POST /v1/rides/:rideId/offer/:offerId/accept    (driver accepts)
  → 200 { ok: true }  |  409 Conflict { reason: "OFFER_EXPIRED" }

POST /v1/rides/:rideId/start | /complete
POST /v1/rides/:rideId/cancel
```

Note the **`idempotencyKey`** on ride creation. A rider on a flaky connection will double-tap "Request." Two identical requests must produce one ride (recall **[85 — Idempotency]**).

---

### 6. The matching flow — step by step

This is the second-most-important section. Walk it slowly in an interview.

```
 1. Rider requests ride at (lat, lng)
        │
        ▼
 2. GEO QUERY: find candidate drivers in the pickup cell + 8 neighbours
        │      → ~50-200 candidates in a dense city
        ▼
 3. FILTER: status == AVAILABLE
            vehicleType matches (UberX vs Black vs XL)
            rating >= threshold
            not already holding an outstanding offer
            heading roughly toward the rider (a driver moving away
              at 60km/h on a highway is useless, even if he's 400m away)
        │      → ~20 candidates
        ▼
 4. RANK BY ETA — *NOT* straight-line distance.
        │
        │   THIS IS THE CLASSIC TRAP.
        │
        │      🚗 Driver A: 500m away — but across a river.
        │                   Nearest bridge is 4km north.
        │                   Real ETA: 14 minutes.
        │
        │      🚗 Driver B: 2km away — straight down the same street.
        │                   Real ETA: 5 minutes.
        │
        │      Straight-line distance picks A. It is wrong.
        │      ETA on the ROAD GRAPH picks B. It is right.
        ▼
 5. OFFER to the best driver, with a 15-SECOND TIMEOUT.
        │   Push over WebSocket. Start a timer.
        │   ┌─ accepted within 15s? ──▶ go to step 6
        │   ├─ declined?             ──▶ offer to next-best driver
        │   └─ timed out (no reply)? ──▶ offer to next-best driver
        │                                (and quietly ding the driver's
        │                                 acceptance-rate metric)
        ▼
 6. ON ACCEPT: ATOMICALLY mark BOTH rider and driver as matched.
        │   ⚠️  THIS IS THE CONCURRENCY BUG THAT KILLS CANDIDATES.
        ▼
 7. Trip → MATCHED. Push driver's live location to rider's WebSocket.
```

#### The concurrency problem — the one you must not miss

Two riders, on opposite corners, request a ride at the same millisecond. The geo index returns the **same nearby driver** to both matching workers. Both workers check `driver.status === 'AVAILABLE'` — both see `true`. Both send an offer. The driver's phone buzzes twice, or worse: **both trips get created and two riders are told the same car is coming.**

This is a textbook **race condition / lost update** (recall **[84 — Distributed Locks]** and **[66 — Consistency Models]**). The fix is that **"check availability" and "claim the driver" must be a single atomic operation.**

Two acceptable answers:

1. **Atomic compare-and-swap** — a conditional update that only succeeds if the driver is still `AVAILABLE`. Exactly one of the two racing workers wins; the loser gets `false` back and moves to the next candidate. This is the cheapest and best answer.
2. **A distributed lock** (Redlock / `SET key val NX PX 30000`) held for the duration of the offer window. Heavier, but necessary if the claim spans multiple resources.

The CAS version, in Redis Lua (a Lua script in Redis is **atomic by definition** — Redis is single-threaded, and the script runs to completion without interleaving):

```js
// claim.lua — the entire matching correctness argument lives in these 12 lines.
const CLAIM_DRIVER_LUA = `
  local driverKey = KEYS[1]
  local expectedStatus = ARGV[1]     -- 'AVAILABLE'
  local newStatus = ARGV[2]          -- 'OFFERED'
  local rideId = ARGV[3]
  local ttl = tonumber(ARGV[4])      -- offer window, seconds

  local current = redis.call('HGET', driverKey, 'status')
  if current ~= expectedStatus then
    return 0                          -- someone else got here first. Lose gracefully.
  end

  redis.call('HSET', driverKey, 'status', newStatus, 'rideId', rideId)
  -- Safety net: if this worker crashes mid-offer, the driver un-sticks itself.
  redis.call('HSET', driverKey, 'offerExpiresAt', tostring(redis.call('TIME')[1] + ttl))
  return 1                            -- we won the driver.
`;
```

---

### 7. Node.js implementation — the geo index and the atomic match

```js
// geo-index.js — the hot path. Everything here must be sub-millisecond.
import Redis from 'ioredis';

export class DriverGeoIndex {
  constructor(redis) {
    this.redis = redis;
  }

  // Shard by city. Rides never cross cities, so this partition is
  // perfectly balanced and never needs resharding (recall 75).
  #geoKey(cityId)    { return `drivers:geo:${cityId}`; }
  #driverKey(id)     { return `driver:${id}`; }

  /**
   * Called 250,000 times per second across the fleet.
   * ONE pipelined round trip. No disk. No durability. If Redis dies,
   * every driver re-reports within 4 seconds and the index rebuilds itself.
   */
  async updateLocation({ driverId, cityId, lat, lng, heading }) {
    await this.redis
      .pipeline()
      // GEOADD stores a 52-bit geohash as the sorted-set SCORE.
      // The 2D->1D mapping we spent this whole doc explaining IS this line.
      .geoadd(this.#geoKey(cityId), lng, lat, driverId)
      .hset(this.#driverKey(driverId), {
        lat, lng, heading, cityId,
        updatedAt: Date.now(),
      })
      // If a driver's app dies, we must not keep offering them rides.
      // A 30s TTL means a silent driver falls out of the index on their own.
      .expire(this.#driverKey(driverId), 30)
      .exec();
  }

  /**
   * The query the entire system exists to serve.
   * Redis walks the target geohash cell AND its 8 neighbours, then applies a
   * true haversine filter — so the boundary problem is handled for us.
   */
  async findNearbyDrivers({ cityId, lat, lng, radiusKm = 3, limit = 20 }) {
    const raw = await this.redis.geosearch(
      this.#geoKey(cityId),
      'FROMLONLAT', lng, lat,
      'BYRADIUS', radiusKm, 'km',
      'ASC',                 // nearest first
      'COUNT', limit,
      'WITHDIST',
      'WITHCOORD',
    );

    // [[ 'driver:1234', '0.7421', ['-122.41', '37.77'] ], ...]
    return raw.map(([driverId, distStr, [lngStr, latStr]]) => ({
      driverId,
      distanceKm: Number(distStr),
      lat: Number(latStr),
      lng: Number(lngStr),
    }));
  }

  async goOffline(driverId, cityId) {
    await this.redis
      .pipeline()
      .zrem(this.#geoKey(cityId), driverId)   // GEO sets ARE sorted sets
      .del(this.#driverKey(driverId))
      .exec();
  }
}
```

```js
// matching-service.js — correctness lives here.
import { CLAIM_DRIVER_LUA } from './claim.js';

const OFFER_TIMEOUT_MS = 15_000;
const MAX_CANDIDATES_TO_TRY = 8;

export class MatchingService {
  constructor({ geoIndex, redis, mapsClient, dispatcher, tripRepo }) {
    this.geo = geoIndex;
    this.redis = redis;
    this.maps = mapsClient;
    this.dispatcher = dispatcher;     // pushes over the driver's WebSocket
    this.trips = tripRepo;
    this.redis.defineCommand('claimDriver', {
      numberOfKeys: 1,
      lua: CLAIM_DRIVER_LUA,
    });
  }

  async matchRide(ride) {
    // STEP 1+2: geo query — the cell and its 8 neighbours, handled by Redis.
    const nearby = await this.geo.findNearbyDrivers({
      cityId: ride.cityId,
      lat: ride.pickup.lat,
      lng: ride.pickup.lng,
      radiusKm: 3,
      limit: 50,
    });

    // STEP 3: filter. Cheap, local predicates first — never call Maps for a
    // driver you're about to reject for being the wrong car.
    const eligible = [];
    for (const c of nearby) {
      const d = await this.redis.hgetall(`driver:${c.driverId}`);
      if (d.status !== 'AVAILABLE') continue;
      if (d.vehicleType !== ride.vehicleType) continue;
      if (Number(d.rating) < 4.2) continue;
      if (Date.now() - Number(d.updatedAt) > 10_000) continue;  // stale ghost
      eligible.push({ ...c, ...d });
    }
    if (eligible.length === 0) return { matched: false, reason: 'NO_DRIVERS' };

    // STEP 4: rank by ETA on the ROAD GRAPH, not by straight-line distance.
    // A driver 500m away across a river is worse than one 2km down the street.
    // One BATCHED call — never N calls in a loop.
    const etas = await this.maps.batchEta(
      eligible.map((d) => ({ from: { lat: d.lat, lng: d.lng }, to: ride.pickup })),
    );
    eligible.forEach((d, i) => { d.etaSeconds = etas[i]; });
    eligible.sort((a, b) => a.etaSeconds - b.etaSeconds);

    // STEP 5+6: offer down the ranked list until someone takes it.
    for (const driver of eligible.slice(0, MAX_CANDIDATES_TO_TRY)) {
      // THE ATOMIC CLAIM. Two riders racing for this same driver:
      // exactly ONE gets a 1 back. The other gets 0 and quietly moves on.
      const won = await this.redis.claimDriver(
        `driver:${driver.driverId}`,
        'AVAILABLE', 'OFFERED', ride.id, 30,
      );
      if (won !== 1) continue;   // lost the race. No harm, no double-booking.

      const accepted = await this.#offerWithTimeout(driver, ride);

      if (accepted) {
        // Promote the claim: OFFERED -> ON_TRIP, and write the durable record.
        await this.redis.hset(`driver:${driver.driverId}`, 'status', 'ON_TRIP');
        await this.trips.markMatched(ride.id, driver.driverId, driver.etaSeconds);
        return { matched: true, driverId: driver.driverId, eta: driver.etaSeconds };
      }

      // Declined or timed out — release the claim so others can have them.
      await this.redis.hset(`driver:${driver.driverId}`, 'status', 'AVAILABLE');
    }

    return { matched: false, reason: 'ALL_DECLINED' };
  }

  #offerWithTimeout(driver, ride) {
    return new Promise((resolve) => {
      const timer = setTimeout(() => resolve(false), OFFER_TIMEOUT_MS);
      this.dispatcher.pushOffer(driver.driverId, ride, (accepted) => {
        clearTimeout(timer);
        resolve(Boolean(accepted));
      });
    });
  }
}
```

---

### 8. Data model — the hot/cold split

**HOT — Redis, ephemeral, in-memory, not durable:**

```
drivers:geo:{cityId}        → Sorted Set. member=driverId, score=52-bit geohash
driver:{driverId}           → Hash { lat, lng, heading, status, vehicleType,
                                      rating, rideId, cityId, updatedAt }
                              TTL 30s — a silent driver removes themselves.
```

**COLD — Postgres, durable, ACID:**

```sql
CREATE TABLE trips (
  id              UUID PRIMARY KEY,
  rider_id        UUID NOT NULL,
  driver_id       UUID,                          -- null until matched
  status          TEXT NOT NULL,                 -- REQUESTED|MATCHED|...|PAID
  pickup_lat      DOUBLE PRECISION NOT NULL,
  pickup_lng      DOUBLE PRECISION NOT NULL,
  dropoff_lat     DOUBLE PRECISION NOT NULL,
  dropoff_lng     DOUBLE PRECISION NOT NULL,
  requested_at    TIMESTAMPTZ NOT NULL,
  completed_at    TIMESTAMPTZ,
  distance_km     NUMERIC(6,2),
  surge_multiplier NUMERIC(3,2) DEFAULT 1.00,
  fare_cents      INTEGER,
  idempotency_key TEXT UNIQUE                    -- kills the double-tap bug
);
CREATE INDEX ON trips (rider_id, requested_at DESC);
CREATE INDEX ON trips (driver_id, requested_at DESC);

CREATE TABLE drivers (
  id UUID PRIMARY KEY, name TEXT, phone TEXT,
  vehicle_type TEXT, license_plate TEXT,
  rating NUMERIC(2,1), total_trips INTEGER, home_city TEXT
);
-- riders: same shape, minus vehicle fields.
```

**Why this split?** Locations are written 250k/sec, read rarely, and are worthless 10 seconds later — **RAM, no durability.** Trips are written 230/sec, must never be lost, and involve money — **disk, ACID, replicated.** Same system, two completely opposite storage philosophies, and you should say *exactly that* in the interview.

If trip volume outgrows Postgres, move `trips` to **Cassandra** partitioned by `rider_id` (recall **[75 — Data Partitioning]**) — trips are append-mostly, immutable once complete, and always queried by rider or driver. That's a perfect Cassandra shape.

---

### 9. Deep dives

#### (a) ETA calculation — it's a graph problem, not a geometry problem

Straight-line ("haversine") distance is a **lie** on a map with rivers, one-way streets, motorways, and rush hour. Real ETA requires:

1. **A road graph** — nodes are intersections, edges are road segments. The edge **weight is travel time**, not length: a 2km motorway edge might be 90 seconds while a 400m gridlocked high-street edge is 6 minutes.
2. **A shortest-path search** over that weighted graph — Dijkstra in principle, but at Uber's scale it's a preprocessed hierarchical algorithm (Contraction Hierarchies / A* with landmarks) that answers in microseconds.
3. **Live traffic**, which continuously rewrites the edge weights. And where does the traffic data come from? **From the same 250k/sec location firehose** — a million drivers moving at known speeds along known roads *is* the best real-time traffic sensor network on Earth. That's a lovely closed loop to point out.

The graph nature of this is the direct connection to **[92 — Graph Databases and Relationships]**. Note it's a *specialised* graph engine, not a general graph DB — but the mental model (nodes, weighted edges, shortest path) is identical.

#### (b) Surge pricing — a stream job with a feedback loop

Compute, per geo cell (an H3 hexagon!), over a rolling 5-minute window:

```
supply = count(AVAILABLE drivers in cell)
demand = count(ride requests originating in cell)
ratio  = demand / max(supply, 1)

ratio > 3.0  → 2.0×
ratio > 2.0  → 1.5×
ratio > 1.5  → 1.2×
else         → 1.0×
```

This is a classic **stream-processing job** (Flink/Kafka Streams) consuming the Kafka location and request topics. Results land in Redis as `surge:{cellId} → multiplier`, read at fare-quote time.

**The feedback loop is the interesting part, and it cuts both ways.** Surge is *supposed* to be self-correcting: higher prices attract drivers into the cell, which raises supply, which lowers the ratio, which lowers surge. But it can also **oscillate** — the price spikes, drivers all pile in at once, supply overshoots, surge collapses, drivers leave, surge spikes again. Real systems damp this with hysteresis, smoothing windows, and caps. Also: **never surge instantaneously** — a rider who sees 3.4× because one other person happened to request nearby feels cheated. Smooth over a window.

#### (c) Trip state machine

```
REQUESTED ──match──▶ MATCHED ──driver en route──▶ ARRIVING
                        │                            │
                     (cancel)                   driver arrives
                        │                            ▼
                        ▼                       IN_PROGRESS
                    CANCELLED                        │
                                                 (rider out)
                                                     ▼
                                                 COMPLETED ──charge──▶ PAID
```

Illegal transitions must be **impossible, not merely discouraged**: you cannot go `REQUESTED → COMPLETED`; you cannot cancel a trip that's `IN_PROGRESS` without a fee. Encode the legal transitions in a table and validate every write against it. This is a textbook **State pattern** — forward-reference **[127 — LLD: Uber Matching]**, where you'll build exactly these classes.

Guard the DB write with a conditional update so two concurrent state changes can't both win:

```sql
UPDATE trips SET status = 'IN_PROGRESS'
WHERE id = $1 AND status = 'ARRIVING';   -- 0 rows updated = illegal transition
```

#### (d) Payment and idempotency

The trip ends. You charge the card. The payment gateway times out. **Did the charge go through?** You don't know. If you retry blindly, you might double-charge a customer — the single worst bug in this entire system.

The fix (recall **[85 — Idempotency]**): generate an **idempotency key** (e.g. `trip:{tripId}:charge:v1`) and send it with the charge request. Stripe and every serious gateway will deduplicate on it: the second request with the same key returns *the original result* instead of charging again. Retry as many times as you like — at most one charge exists.

Charge asynchronously via a queue, not in the request path. If the card declines, the trip is still `COMPLETED` — you chase the money afterwards. **Never block the driver from taking their next ride on a payment processor's latency.**

#### (e) Driver disconnection mid-trip

The driver goes into a tunnel. WebSocket drops. Location pings stop.

- **Don't panic.** The driver app **buffers** location points locally and replays them on reconnect. Trips are not cancelled on a brief disconnect.
- **The rider's map** shows the last known position with a subtle "updating..." state — do not freeze, and absolutely do not show the car teleporting backwards when the stale buffered points flood in. Reconcile by timestamp and only move the marker forward.
- **The `driver:{id}` hash TTL (30s)** means a truly-gone driver falls out of the *available* index automatically — but a driver **on a trip** must NOT be evicted. Keep on-trip state in the durable trip record in Postgres, not only in Redis. Redis is for *matching*; Postgres is the truth about *what's happening*.
- **After ~3 minutes of silence mid-trip**, escalate: ping the rider, ping the driver's phone, and if both are dark, flag for support. Safety beats efficiency here.

---

### 10. Bottlenecks, failure modes, and how to scale further

| Bottleneck / failure | Symptom | Fix |
|---|---|---|
| **The 250k/s location firehose** | Location Service CPU pegged; ingestion lag | Stateless Location Service, horizontally scaled behind an L4 LB. Shard Redis **by city**. Batch writes with pipelining. |
| **Redis geo shard hotspot** | One city (a huge metro) saturates one node | Sub-shard the mega-city by **geohash prefix** (SF-north / SF-south). Rides rarely cross the split, and you can query both shards when they do. |
| **Redis dies** | The geo index is gone | **Shrug.** Rebuild from the location firehose in ~4 seconds. Run a hot replica so failover is even faster. **This is a non-event, and saying so confidently is a strong signal.** |
| **Thundering herd on a surge event** (a stadium empties) | 40k requests in one cell in 60s | Queue requests. Widen the search radius progressively (3km → 5km → 8km). Surge, which by design suppresses demand. Communicate honestly: "high demand, 8 min wait." |
| **Maps/ETA service latency** | Matching is slow, riders bail | Cache ETAs per (cell-pair, time-bucket). Fall back to haversine × a road-winding fudge factor (~1.3×) if Maps times out. **Degrade, never block.** |
| **Double-booking a driver** | Two riders, one car, furious support tickets | The **atomic CAS claim** above. This is the correctness bug the whole system is designed around. |
| **Hot Postgres `trips` table** | Write contention as trip volume grows | Shard by `rider_id`, or move to Cassandra. Trips are append-mostly and immutable once completed. |
| **Ghost drivers** (app crashed, entry stale) | Offers sent into the void; matching wastes 15s per ghost | The 30s TTL on `driver:{id}` plus a `updatedAt` freshness check in the filter step. |
| **Cascading failure from Kafka backpressure** | Location Service blocks writing to Kafka; hot path dies | **Kafka must NEVER be on the hot path.** Fire-and-forget with a bounded local buffer; **drop analytics events under pressure**. A lost analytics event costs a rounding error. A lost ride costs a customer. |

---

## Visual / Diagram description

The three diagrams you must be able to redraw from memory on a whiteboard:

1. **The architecture diagram** (Section 4). The critical thing it communicates is the **fork at the Location Service**: a synchronous hot path into Redis, and an asynchronous cold path into Kafka. Draw the fork, and say "nothing goes on the hot path unless a rider needs it in the next second."

2. **The geohash halving diagram** (Section 3, Solution A). Draw one big square, cut it in half, cut it again, and show the string growing one character at a time as the box shrinks. Then write two nearby places and circle their **shared prefix**. This is the single picture that explains the whole doc.

3. **The 3×3 neighbour grid** (the boundary problem). Draw a rider next to a cell boundary with a driver 5 metres away *on the other side*. Then draw the 3×3 grid of cells you actually query. This diagram is your defence against the follow-up question everybody gets asked.

If you can draw those three and narrate them, you can pass this interview.

---

## Real world examples

### Uber — H3

Uber built and open-sourced **H3**, a hexagonal hierarchical geospatial indexing system. Hexagons were chosen because all six neighbours of a hex cell are equidistant from its centre, which makes flow, smoothing, and k-ring analyses cleaner than square grids (where edge-neighbours and corner-neighbours sit at different distances). H3 is used across Uber for demand/supply modelling, surge zones, and marketplace analytics. It's on GitHub and widely adopted outside Uber.

### Google — S2

Google's **S2** library projects the sphere onto a cube and orders cells along a **Hilbert curve**, giving better locality preservation than geohash's Z-order curve (a Hilbert curve never makes a long jump between consecutive cells; a Z-order curve does). S2 handles the poles and the antimeridian gracefully, which naive lat/lng grids do not. Used broadly in Google's geo stack, and by companies like Foursquare for venue proximity.

### Redis — GEO commands

Redis implements `GEOADD` / `GEOSEARCH` on top of **sorted sets**, storing a 52-bit geohash integer as the member's score. Because a sorted set is a skip-list ordered by score, and because a geohash preserves locality, a radius query becomes a small set of `ZRANGEBYSCORE` operations over the target cell's score range and its neighbours' ranges, followed by an exact haversine filter. It is, in effect, this entire doc compiled down into one data structure — and it's the pragmatic answer to give when the interviewer asks "so what would you actually use?"

---

## Trade-offs

**Choosing a spatial index:**

| Index | Pros | Cons | Use when |
|---|---|---|---|
| **Geohash** | Dead simple; a pure function; works with any string index; no rebuild; trivially shardable | Fixed cell size (bad for very uneven density); boundary problem; cells distort near poles | You want the standard answer. **Default choice.** |
| **QuadTree** | Adaptive to density — every leaf holds ~equal drivers; consistent query cost anywhere | In-memory pointer structure you must build, shard, and rebuild yourself; constant mutation under a write firehose | Density varies wildly and you control the whole index |
| **S2** | Excellent locality (Hilbert curve); correct at poles/antimeridian; mature | More complex; heavier library | Global scale, geometric correctness matters |
| **H3** | Equidistant neighbours; superb for flow/aggregation/surge maths | Hexagons can't tile perfectly hierarchically (children don't nest exactly); newer | Marketplace/flow modelling — Uber's own use case |
| **PostGIS (`GiST` index)** | Real SQL, real geometry, ACID | Disk-based. **Will not survive 250k writes/sec.** | Read-heavy geo queries on a *slow-changing* dataset (store locations, delivery zones) |

**Other key trade-offs:**

| Decision | You gain | You give up |
|---|---|---|
| Locations in Redis, not Postgres | 250k writes/sec; sub-ms reads | Durability (which you didn't need — it self-heals in 4s) |
| WebSocket instead of HTTP polling | No handshake cost; server can push offers | Stateful connections; harder load balancing; connection-count limits |
| Eventual consistency on locations | Massive throughput; simple design | A driver's dot may be 2–4s stale (harmless) |
| Rank by ETA, not distance | Correct matches; happy riders | A dependency on a Maps service, on the latency-critical path |
| Kafka for analytics, off the hot path | The firehose can't take down matching | Analytics is eventually consistent, and may drop events under load |
| Atomic CAS claim on drivers | No double-booking, ever | A small amount of contention; some workers lose the race and retry |

**The sweet spot:** **Redis GEO, sharded by city, with an atomic Lua compare-and-swap for the driver claim.** It gives you 250k writes/sec, sub-100ms proximity queries, no double-bookings, and an index that heals itself in four seconds if you drop it on the floor. Reach for a quadtree or H3 only when you can name the specific problem the simpler design failed to solve.

**Rule of thumb:** *if the data is hot, tiny, and reconstructible, it belongs in RAM with no durability guarantees — and you should be loud about the fact that you don't need durability, because that's the insight the interviewer is listening for.*

---

## Common interview questions on this topic

### Q1: "Why can't you just put driver locations in Postgres with an index on (lat, lng)?"

**Hint:** Two independent reasons, and give both. **(1) Write rate:** 1M drivers ÷ 4s = 250k writes/sec; a Postgres node does ~5–10k/sec. You'd need 25–50 saturated primaries doing nothing but overwriting rows. **(2) Query shape:** a B-tree is 1-dimensional. An index on `lat` gives you a latitude band that wraps the entire planet, then filters `lng` row by row. It's a scan with extra steps. You need a structure that maps 2D → 1D while preserving locality. That's a geohash.

### Q2: "Two riders request a ride at the same instant and the geo index returns the same driver to both. What happens?"

**Hint:** This is the money question. Without care, a **lost-update race**: both workers read `status = AVAILABLE`, both offer, and you double-book. The fix is that check-and-claim must be **one atomic operation** — a Redis Lua compare-and-swap (`if status == 'AVAILABLE' then set 'OFFERED'`), which is atomic because Redis is single-threaded and runs scripts to completion. Exactly one worker gets `1` back; the other gets `0` and moves to the next candidate. Mention a TTL on the claim so a crashed matching worker doesn't strand the driver forever.

### Q3: "A driver is 5 metres from a rider but in a different geohash cell. Your query misses them. Why, and how do you fix it?"

**Hint:** The **boundary problem**. Geohash is recursive halving, so a boundary crossing at any level diverges the string from that character onward — two points metres apart can have unrelated hashes. Fix: always query the target cell **plus its 8 neighbours** (every geohash library has `neighbors()`), union the results, then apply a real haversine distance filter. Bonus: Redis `GEOSEARCH` does this for you internally, which is one reason to use it rather than rolling your own.

### Q4: "Why did Uber build H3 with hexagons instead of squares?"

**Hint:** With squares, the 8 neighbours split into 4 edge-neighbours at distance `d` and 4 corner-neighbours at distance `d√2` — so "adjacent" means two different distances, and any flow / smoothing / k-ring calculation acquires a diagonal bias you have to correct for. With hexagons, **all 6 neighbours are exactly equidistant**. For a company whose core business is modelling supply-and-demand flow across a city grid, that makes the maths clean rather than fudged. (Honest caveat: hexagons don't nest perfectly hierarchically — H3 child cells don't tile a parent exactly — which is a real cost H3 accepts.)

### Q5: "Redis goes down and you lose the entire driver location index. What's your recovery plan?"

**Hint:** Answer with a shrug, and mean it. **Every driver re-reports within 4 seconds**, so the index rebuilds itself from the live firehose with no data loss and no backup restore. That's the payoff for the earlier decision that this data is ephemeral. Keep a hot replica so failover is a couple of seconds rather than four. Contrast this loudly with the `trips` table, where losing data means losing money — *that's* the half that gets replication, backups, and ACID.

---

## Practice exercise

### Build a working geo-matcher

**Time: ~35 minutes. You will write code and run it.**

Spin up Redis locally (`docker run -p 6379:6379 redis`) and build a small Node script that does the following:

1. **Seed the index.** Generate **10,000 fake drivers** scattered randomly within roughly 10km of San Francisco (37.7749, -122.4194). `GEOADD` them all to `drivers:geo:sf` and give each a `driver:{id}` hash with `status: 'AVAILABLE'`, a random `vehicleType` (`X` or `XL`), and a random `rating` between 3.8 and 5.0.

2. **Query.** Write `findNearbyDrivers(lat, lng, radiusKm)` using `GEOSEARCH ... WITHDIST ASC COUNT 20`. Print the 10 nearest drivers with their distances. **Time it.** (You should see well under 5 ms. Sit with that: 10,000 drivers, sub-5ms. That is the entire lesson.)

3. **Prove the naive approach is bad.** Now write the dumb version: `HGETALL` every one of the 10,000 drivers, compute haversine distance to the rider in JS, sort, take 20. **Time it too.** Write down both numbers side by side. Then multiply the naive number by 5,000 requests/second and stare at it.

4. **Break it, then fix it.** Write the Lua CAS claim script. Then fire **two concurrent `matchRide()` calls for the same pickup point** with `Promise.all`. First run it **without** the CAS (just a `HGET` status check followed by an `HSET`) — with 10k drivers you may need to shrink the candidate pool to force the collision, so restrict the search to a tiny radius with only one eligible driver. Show that both riders get the same driver. Then add the CAS and show that exactly one wins and the other falls through to `NO_DRIVERS` or the next candidate.

**Produce:** one `matcher.js` file, plus **three numbers written in a comment at the top** — naive query time, Redis query time, and the ratio between them. That ratio is the reason geospatial indexing exists.

---

## Quick reference cheat sheet

- **Uber is WRITE-heavy, not read-heavy.** 250,000 location writes/sec vs 5,000 ride requests/sec — a 50:1 ratio. This inverts every instinct you built designing Twitter. Say it out loud in the interview.
- **The one number:** 1M drivers ÷ 4-second ping = **250k writes/sec**. It alone rules out any disk-based database and mandates an in-memory index.
- **The location index is ephemeral and self-healing.** Lose it entirely and it rebuilds in 4 seconds from the live firehose. No durability needed. This is the most elegant simplification in the whole design.
- **The core problem:** proximity is 2-dimensional; every fast index we own (B-tree, hash, sorted set) is 1-dimensional. **A spatial index is a machine for turning 2D proximity into a 1D key lookup.**
- **Geohash:** recursively halve the world, encode the bits as base-32. **Nearby points share a string PREFIX.** Longer prefix = smaller cell. Search becomes a prefix scan.
- **Precision:** 6 chars ≈ **1.2km** (the ride-matching sweet spot); 7 chars ≈ **150m**; 5 chars ≈ 5km.
- **The boundary problem:** two points metres apart can sit in different cells with unrelated hashes. **Always query the target cell + its 8 neighbours**, then filter by true distance.
- **QuadTree:** subdivides only where density demands it, so every leaf holds a similar number of drivers. Adaptive — but it's a mutable in-memory tree you must maintain under a write firehose.
- **H3 (Uber's, hexagonal):** all **6 neighbours are equidistant**, unlike a square's 4 edge- and 4 corner-neighbours at different distances. Makes flow and surge maths clean. This detail wins interviews.
- **Redis GEO** (`GEOADD` / `GEOSEARCH`) is the practical answer — sorted sets with 52-bit geohash scores, and it handles neighbour cells for you.
- **Shard by city.** Rides never cross cities; it's a naturally balanced partition that never needs resharding.
- **Rank by ETA on the road graph, NOT straight-line distance.** A driver 500m away across a river loses to one 2km down the same street.
- **The killer bug: double-booking.** Check-availability and claim-driver must be **ONE atomic operation** — a Redis Lua compare-and-swap. Exactly one racing worker wins.
- **The hot/cold split:** ephemeral locations in RAM (no durability); durable trips and payments on disk (full ACID, idempotent charges). Two opposite storage philosophies in one system — name the split explicitly.

---

## Connected topics

| Direction | Topic |
|---|---|
| **Previous** | [82 — Content Negotiation and Protocols](./82-content-negotiation-and-protocols.md) — why driver pings and live tracking use persistent WebSockets rather than HTTP polling; the protocol choice is what makes a 250k/sec firehose affordable |
| **Next** | [127 — LLD: Uber Matching](./127-lld-uber-matching.md) — zoom in from the architecture to the actual classes: the trip State pattern, the Matcher, the driver claim, written out as real code |
| **Related** | [75 — Data Partitioning](./75-data-partitioning.md) — sharding the geo index by city, and the `trips` table by rider; how to pick a shard key that never needs to change |
| **Related** | [67 — Message Queues](./67-message-queues.md) — the Kafka stream that carries location events off the hot path to analytics, surge computation, and history |
| **Related** | [92 — Graph Databases and Relationships](./92-graph-databases-and-relationships.md) — ETA is a shortest-path search on a weighted road graph, which is why straight-line distance is the wrong ranking signal |
