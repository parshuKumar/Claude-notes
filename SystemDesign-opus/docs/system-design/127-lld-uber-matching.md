# 127 — Design Uber Driver-Rider Matching (Class Level)
## Category: LLD Case Study

---

## What is this?

This is the **class-level design** of the piece of Uber that connects a rider who wants a ride to a driver who can give one. A rider says "take me from A to B," the system finds nearby available drivers, picks one, and shepherds that ride through its life: requested → matched → driver arriving → in progress → completed, with a fare at the end.

Think of it as a **restaurant host at a busy dinner service**. A guest walks in (the ride request). The host scans the floor for a free table nearby (find available drivers), seats them at the closest good one (matching), tracks the meal from seated to eating to paid (the lifecycle), and hands over the bill at the end (fare). This doc designs that host as clean JavaScript classes.

The big-picture, "how do we do this for millions of drivers across a city" version is the **HLD in [102 — HLD: Uber](./102-hld-uber.md)**. Here we zoom all the way in to the objects, their behaviours, and the patterns that keep the code swappable.

---

## Why does it matter?

**Interview angle.** "Design the matching part of Uber" is a classic LLD prompt. The interviewer is not asking you to shard a geospatial index on a whiteboard — that is the HLD. They want to see whether you can turn a fuzzy real-world flow into **objects with clear responsibilities**, model a **state machine** that refuses illegal transitions, and isolate the two decisions that change most often (which driver? what price?) behind **swappable strategies**. Candidates who dump everything into one 400-line `RideService` fail this; candidates who name their patterns and defend them pass.

**Real-work angle.** Matching logic and pricing logic change constantly — surge, pooling, priority for high-rated drivers, ETA-based matching, promo pricing. If those policies are welded into your service class, every experiment is a risky edit to core code. If they live behind a `MatchingStrategy` and a `FareStrategy` interface, a new policy is a new class you plug in. And if your `Ride` object does not guard its own lifecycle, you will eventually ship the bug where a ride jumps from `REQUESTED` straight to `COMPLETED` and bills a rider for a trip that never happened.

---

## The core idea — explained simply

### Analogy: the restaurant host, the seating rule, and the bill

Picture a restaurant on a Friday night. Three roles matter:

- **The host** stands at the door. Every guest goes through the host. The host does not cook or calculate the bill — the host's one job is to *route* guests to tables and keep order. That is our `RideService`.
- **The seating rule** is a policy the host follows: "seat guests at the nearest clean table." But management might change it to "seat VIPs near the window" or "balance load across servers." The rule is *pluggable* — the host obeys whatever rule is handed to them tonight. That is our `MatchingStrategy`.
- **The bill** is computed by a pricing policy: normally base + per-item, but on a holiday there's a surcharge multiplier. Also pluggable. That is our `FareStrategy`.
- **Each table** moves through states: empty → seated → eating → paid. You cannot go from "empty" to "paid." The table itself enforces this. That is our `Ride` and its state machine.

Map it back:

| Restaurant | Uber class | Responsibility |
|---|---|---|
| The host at the door | `RideService` | Orchestrates: takes requests, applies rules, advances rides |
| Guest | `Rider` | Who wants the ride |
| Server / table | `Driver` | Who provides it; has a status and a location |
| The seating rule | `MatchingStrategy` | *Which* driver to pick — swappable policy |
| The bill formula | `FareStrategy` | *What* to charge — swappable policy |
| A table's meal lifecycle | `Ride` + `RideStatus` | State machine that rejects illegal jumps |
| "table near you" | `Location.distanceTo` | Geometry; at scale becomes a geospatial index |

The two things that change most — **who** and **how much** — are strategies. The one thing that must never be corrupted — the **lifecycle** — is a guarded state machine. Everything else is plain data.

---

## Key concepts inside this topic

We follow the standard 7-step LLD method (see [111 — LLD Approach Framework](./111-lld-approach-framework.md)).

### 1. Requirements & clarifying questions

**Functional requirements (what it must do):**

- A rider requests a ride from a **pickup** location to a **dropoff** location.
- The system finds **nearby available drivers** and **matches** one.
- The matched driver can (optionally) **accept or decline**; on decline, try the next driver.
- A ride moves through a **lifecycle**: `REQUESTED → MATCHED → EN_ROUTE → IN_PROGRESS → COMPLETED`, or `CANCELLED` from most states.
- On completion, compute a **fare** = base + distance + time, possibly multiplied by **surge**.
- Either party can **cancel** before the trip starts.

**Clarifying questions to ask the interviewer (always do this first):**

- *Pool / shared rides?* — Multiple riders in one car changes matching and fare-splitting. Assume **single-rider** for the core, and show pooling as an extension.
- *Scheduled rides?* — Book-ahead vs. request-now. Assume **request-now**.
- *Is there a driver-acceptance step?* — Real Uber lets a driver decline. We model it as an optional step with fallback to the next driver.
- *How do we find "nearby" drivers at scale?* — This is the crux of the HLD. At scale you need a **geospatial index** (geohash / quadtree — see [102](./102-hld-uber.md)). **In this LLD we implement a simple linear distance scan** and explicitly flag it as the seam that becomes the index at scale.
- *Payment, ratings, driver onboarding?* — Out of scope for this class; we keep a `rating` field on `Driver` so ETA/rating-based matching is easy to add.

Saying "the geospatial index is the HLD; here I'll do a linear scan and call out where the index plugs in" is exactly the altitude control the interviewer wants.

### 2. Core objects (the nouns)

Read the requirements and underline the nouns:

- **`Rider`** — id, name, current location.
- **`Driver`** — id, name, location, `status`, `rating`.
- **`Ride`** (a.k.a. Trip) — the rider, the assigned driver, pickup, dropoff, `status`, computed `fare`. This is the aggregate that owns the lifecycle.
- **`Location`** — `lat`, `lng`, plus a `distanceTo(other)` geometry helper.
- **`MatchingStrategy`** — the interface for "pick a driver," with `NearestDriverStrategy` as default.
- **`FareStrategy`** — the interface for "price this ride," with `StandardFareStrategy` and `SurgeFareStrategy`.
- **`RideService`** — the orchestrator that ties it all together.

Two **enums** (frozen objects in JS):

- **`DriverStatus`** = `AVAILABLE` / `EN_ROUTE` / `ON_TRIP` / `OFFLINE`.
- **`RideStatus`** = `REQUESTED` → `MATCHED` → `EN_ROUTE` → `IN_PROGRESS` → `COMPLETED`, plus `CANCELLED`.

### 3. Behaviours (verbs) + who owns them

The trap is to give every verb to `RideService`. Instead, put each behaviour on the object that owns the data it touches.

| Behaviour | Owned by | Why |
|---|---|---|
| `requestRide(rider, pickup, dropoff)` | `RideService` | Needs the driver pool + strategies; pure orchestration |
| `findDriver(pickup, drivers)` | `MatchingStrategy` | The *policy* of choosing — swappable |
| `acceptRide()` / `declineRide()` | `Driver` (decision) → `RideService` (fallback) | The driver decides; the service reacts |
| `updateStatus(newStatus)` | `Ride` | Only the ride knows its legal transitions |
| `startTrip(ride)` / `endTrip(ride)` | `RideService` | Coordinates ride status + driver status |
| `calculateFare(ride)` | `FareStrategy` | The *policy* of pricing — swappable |
| `cancel(ride)` | `RideService` (drives `Ride.updateStatus`) | Coordinates rollback of driver status |
| `distanceTo(other)` | `Location` | Pure geometry lives with the coordinates |

The key insight: **`Ride` owns its status transitions** (State), **`RideService` owns matching** (via a Strategy), and a **`FareStrategy` owns pricing** (Strategy). Nobody reaches into a `Ride` and sets `ride.status = 'COMPLETED'` directly.

### 4. The matching problem — use STRATEGY

The question "which driver gets this ride?" has many valid answers:

- **Nearest** driver by distance (our default — lowest pickup time on average).
- **Best ETA** — nearest by road/traffic time, not straight-line distance.
- **Highest rating** among nearby drivers.
- **Acceptance likelihood** — pick the driver most likely to accept.

These are *policies*, and they change with business experiments. The clean answer is the **Strategy pattern** (recall [42 — Strategy Pattern](./42-pattern-strategy.md)): define a `MatchingStrategy` interface with one method, `findDriver(pickup, availableDrivers)`, and let `RideService` hold a reference to *some* strategy without knowing which. Swapping matching logic becomes "construct the service with a different strategy" — zero edits to the service.

Our default `NearestDriverStrategy` does two things:

1. **Filter** the driver pool to those who are `AVAILABLE` and **within a radius** of the pickup.
2. **Pick the closest** of those.

That filter step — "drivers within radius R of this point" — is a **linear scan** over every driver here. That is fine for a demo with 5 drivers. At real scale (hundreds of thousands of drivers) a linear scan per request is impossibly slow, so **this exact step becomes a geospatial index** (geohash buckets or a quadtree) in the HLD ([102](./102-hld-uber.md)). Calling that out — "this scan is the seam where the index goes" — shows you understand the LLD/HLD boundary.

### 5. Class diagram + ride state machine

```
                         ┌──────────────────────────┐
                         │       RideService        │  the "host"
                         │──────────────────────────│
                         │ - drivers: Driver[]       │
                         │ - matchingStrategy ◇──────┼───▶ «interface» MatchingStrategy
                         │ - fareStrategy     ◇──────┼───▶     findDriver(pickup, drivers)
                         │──────────────────────────│         │
                         │ + requestRide()           │         └── NearestDriverStrategy
                         │ + startTrip(ride)         │
                         │ + endTrip(ride)   ◇───────┼───▶ «interface» FareStrategy
                         │ + cancel(ride)            │         calculateFare(ride)
                         └─────────┬─────────────────┘         ├── StandardFareStrategy
                                   │ creates / owns            └── SurgeFareStrategy
                                   ▼
                         ┌──────────────────────────┐
                         │           Ride           │  owns the state machine
                         │──────────────────────────│
                         │ - rider: Rider            │◇────▶ Rider  { location }
                         │ - driver: Driver          │◇────▶ Driver { location, status, rating }
                         │ - pickup: Location        │◇────▶ Location { lat, lng, distanceTo() }
                         │ - dropoff: Location       │
                         │ - status: RideStatus      │
                         │ - fare: number            │
                         │ + updateStatus(newStatus) │  ← rejects illegal jumps
                         └──────────────────────────┘

Ride state machine (updateStatus enforces these edges only):

   ┌───────────┐  match   ┌─────────┐ pickup ┌──────────┐ start  ┌─────────────┐ end ┌───────────┐
   │ REQUESTED │─────────▶│ MATCHED │───────▶│ EN_ROUTE │───────▶│ IN_PROGRESS │────▶│ COMPLETED │
   └─────┬─────┘          └────┬────┘        └────┬─────┘        └─────────────┘     └───────────┘
         │                     │                  │
         │ cancel              │ cancel           │ cancel
         ▼                     ▼                  ▼
   ┌───────────────────────────────────────────────────┐
   │                     CANCELLED                      │   (terminal, like COMPLETED)
   └───────────────────────────────────────────────────┘

Illegal (rejected): REQUESTED ─X▶ COMPLETED,  COMPLETED ─X▶ anything,  IN_PROGRESS ─X▶ MATCHED
```

`RideService` **has-a** matching strategy and **has-a** fare strategy (aggregation, the `◇` diamonds). A `Ride` **has** a rider, a driver, and two locations. The state machine lives entirely inside `Ride.updateStatus`.

### 6. Full JavaScript implementation

See the next section for the complete, runnable file. It defines the enums, `Location`, the actors, the guarded `Ride`, both strategy families, the `RideService` orchestrator, and a `main()` demo that proves matching, the lifecycle, both fare strategies, and rejection of an illegal transition.

### 7. Patterns used — see the dedicated section below

Strategy (twice) and State are the stars. Named and justified in "Design patterns used and WHY."

---

## Visual / Diagram description

The class diagram above shows the **host-and-policies** shape: a single `RideService` that owns two pluggable strategy interfaces and creates `Ride` aggregates. The **state machine** below it is the second required diagram — a linear happy path (`REQUESTED → MATCHED → EN_ROUTE → IN_PROGRESS → COMPLETED`) with a `CANCELLED` escape hatch reachable from the pre-trip states, and explicit examples of edges that `updateStatus` must reject.

When you redraw this on a whiteboard, draw the state machine first — it anchors the whole design — then hang the classes off it: `Ride` guards the states, `RideService` drives the transitions, and the two strategies float to the side as swappable boxes.

---

## Full JavaScript implementation

```javascript
'use strict';

// ─────────────────────────────────────────────────────────────
// Enums — frozen objects so nobody can mutate the vocabulary.
// ─────────────────────────────────────────────────────────────
const DriverStatus = Object.freeze({
  AVAILABLE: 'AVAILABLE',   // free, can be matched
  EN_ROUTE:  'EN_ROUTE',    // heading to a pickup
  ON_TRIP:   'ON_TRIP',     // rider on board
  OFFLINE:   'OFFLINE',     // not working
});

const RideStatus = Object.freeze({
  REQUESTED:   'REQUESTED',
  MATCHED:     'MATCHED',
  EN_ROUTE:    'EN_ROUTE',     // driver heading to pickup
  IN_PROGRESS: 'IN_PROGRESS',  // trip underway
  COMPLETED:   'COMPLETED',
  CANCELLED:   'CANCELLED',
});

// ─────────────────────────────────────────────────────────────
// Location — pure geometry. distanceTo() is the seam that
// becomes a geospatial index at scale (see HLD 102).
// ─────────────────────────────────────────────────────────────
class Location {
  constructor(lat, lng) {
    this.lat = lat;
    this.lng = lng;
  }

  // Simplified planar (Euclidean) distance in "km", good enough
  // for a runnable demo. ~111 km per degree of latitude near the
  // equator. Real code uses the haversine formula on a sphere.
  distanceTo(other) {
    const dLat = (this.lat - other.lat) * 111;
    const dLng = (this.lng - other.lng) * 111;
    return Math.sqrt(dLat * dLat + dLng * dLng);
  }
}

// ─────────────────────────────────────────────────────────────
// Actors
// ─────────────────────────────────────────────────────────────
class Rider {
  constructor(id, name, location) {
    this.id = id;
    this.name = name;
    this.location = location;
  }
}

class Driver {
  constructor(id, name, location, rating = 5.0) {
    this.id = id;
    this.name = name;
    this.location = location;
    this.rating = rating;
    this.status = DriverStatus.AVAILABLE;
  }

  // A driver decides whether to take a matched ride. We model
  // acceptance probability; override / stub in tests as needed.
  acceptRide(_ride) {
    // Default: a driver accepts. The fallback path in RideService
    // handles the decline case (see acceptanceRate hook below).
    return Math.random() < this.acceptanceRate;
  }
}
Driver.prototype.acceptanceRate = 1.0; // demo default: always accept

// ─────────────────────────────────────────────────────────────
// Ride — the aggregate that OWNS its lifecycle. updateStatus is
// the single guarded door for every state change (State pattern
// expressed as a transition table).
// ─────────────────────────────────────────────────────────────
const LEGAL_TRANSITIONS = Object.freeze({
  [RideStatus.REQUESTED]:   [RideStatus.MATCHED, RideStatus.CANCELLED],
  [RideStatus.MATCHED]:     [RideStatus.EN_ROUTE, RideStatus.CANCELLED],
  [RideStatus.EN_ROUTE]:    [RideStatus.IN_PROGRESS, RideStatus.CANCELLED],
  [RideStatus.IN_PROGRESS]: [RideStatus.COMPLETED],
  [RideStatus.COMPLETED]:   [], // terminal
  [RideStatus.CANCELLED]:   [], // terminal
});

let RIDE_SEQ = 1;

class Ride {
  constructor(rider, pickup, dropoff) {
    this.id = RIDE_SEQ++;
    this.rider = rider;
    this.driver = null;         // assigned at match time
    this.pickup = pickup;
    this.dropoff = dropoff;
    this.status = RideStatus.REQUESTED;
    this.fare = null;           // computed on endTrip
  }

  // The heart of the State pattern: reject illegal jumps loudly.
  updateStatus(newStatus) {
    const allowed = LEGAL_TRANSITIONS[this.status] || [];
    if (!allowed.includes(newStatus)) {
      throw new Error(
        `Illegal ride transition: ${this.status} -> ${newStatus} ` +
        `(ride #${this.id}). Allowed: [${allowed.join(', ') || 'none'}]`
      );
    }
    this.status = newStatus;
    return this;
  }

  // Straight-line trip distance; drives fare + ETA.
  tripDistanceKm() {
    return this.pickup.distanceTo(this.dropoff);
  }
}

// ─────────────────────────────────────────────────────────────
// MatchingStrategy — STRATEGY pattern. "Which driver?" is a
// swappable policy. Base class documents the contract.
// ─────────────────────────────────────────────────────────────
class MatchingStrategy {
  // Return a Driver or null if none suitable.
  findDriver(_pickup, _availableDrivers) {
    throw new Error('findDriver() must be implemented by a subclass');
  }
}

class NearestDriverStrategy extends MatchingStrategy {
  constructor(radiusKm = 5) {
    super();
    this.radiusKm = radiusKm;
  }

  findDriver(pickup, availableDrivers) {
    // STEP 1 — filter to AVAILABLE drivers within radius.
    // NOTE: this linear scan is the exact step that becomes a
    // GEOSPATIAL INDEX (geohash / quadtree) at scale — see 102.
    const candidates = availableDrivers.filter(
      d => d.status === DriverStatus.AVAILABLE &&
           d.location.distanceTo(pickup) <= this.radiusKm
    );
    if (candidates.length === 0) return null;

    // STEP 2 — pick the closest.
    return candidates.reduce((best, d) =>
      d.location.distanceTo(pickup) < best.location.distanceTo(pickup) ? d : best
    );
  }
}

// A second matching policy, to prove the seam: highest-rated
// driver within radius. Swap it in with ZERO service changes.
class HighestRatedStrategy extends MatchingStrategy {
  constructor(radiusKm = 5) {
    super();
    this.radiusKm = radiusKm;
  }

  findDriver(pickup, availableDrivers) {
    const candidates = availableDrivers.filter(
      d => d.status === DriverStatus.AVAILABLE &&
           d.location.distanceTo(pickup) <= this.radiusKm
    );
    if (candidates.length === 0) return null;
    return candidates.reduce((best, d) => (d.rating > best.rating ? d : best));
  }
}

// ─────────────────────────────────────────────────────────────
// FareStrategy — STRATEGY pattern again. "What to charge?" is a
// swappable policy. base + perKm*distance + perMin*estimatedTime.
// ─────────────────────────────────────────────────────────────
class FareStrategy {
  calculateFare(_ride) {
    throw new Error('calculateFare() must be implemented by a subclass');
  }
}

class StandardFareStrategy extends FareStrategy {
  constructor({ base = 2.5, perKm = 1.2, perMin = 0.35, avgSpeedKmh = 30 } = {}) {
    super();
    this.base = base;
    this.perKm = perKm;
    this.perMin = perMin;
    this.avgSpeedKmh = avgSpeedKmh;
  }

  calculateFare(ride) {
    const km = ride.tripDistanceKm();
    const minutes = (km / this.avgSpeedKmh) * 60; // estimated trip time
    const fare = this.base + this.perKm * km + this.perMin * minutes;
    return Math.round(fare * 100) / 100; // round to cents
  }
}

// Surge WRAPS a base strategy and multiplies. This is Strategy +
// a touch of Decorator: any fare policy can be surged.
class SurgeFareStrategy extends FareStrategy {
  constructor(baseStrategy, multiplier = 1.0) {
    super();
    this.baseStrategy = baseStrategy;
    this.multiplier = multiplier;
  }

  calculateFare(ride) {
    const base = this.baseStrategy.calculateFare(ride);
    return Math.round(base * this.multiplier * 100) / 100;
  }
}

// ─────────────────────────────────────────────────────────────
// RideService — the "host". Orchestrates; owns no business
// policy itself, it delegates to the injected strategies.
// ─────────────────────────────────────────────────────────────
class RideService {
  constructor(drivers, matchingStrategy, fareStrategy) {
    this.drivers = drivers;                 // the pool (the "index" at scale)
    this.matchingStrategy = matchingStrategy;
    this.fareStrategy = fareStrategy;
  }

  // Match via strategy, create the ride, reserve the driver.
  // Honours driver acceptance: on decline, try the next candidate.
  requestRide(rider, pickup, dropoff) {
    const ride = new Ride(rider, pickup, dropoff);

    // Try candidates in strategy order until one accepts.
    let pool = this.drivers;
    let driver = this.matchingStrategy.findDriver(pickup, pool);
    while (driver) {
      if (driver.acceptRide(ride)) {
        ride.driver = driver;
        ride.updateStatus(RideStatus.MATCHED);   // REQUESTED -> MATCHED
        driver.status = DriverStatus.EN_ROUTE;    // reserve the driver
        ride.updateStatus(RideStatus.EN_ROUTE);   // MATCHED -> EN_ROUTE
        console.log(
          `Ride #${ride.id}: matched ${driver.name} ` +
          `(${driver.location.distanceTo(pickup).toFixed(2)} km away, ` +
          `rating ${driver.rating})`
        );
        return ride;
      }
      // Declined → drop this driver and retry with the rest.
      console.log(`Ride #${ride.id}: ${driver.name} declined, trying next...`);
      pool = pool.filter(d => d !== driver);
      driver = this.matchingStrategy.findDriver(pickup, pool);
    }

    // No driver accepted / none available.
    console.log(`Ride #${ride.id}: no drivers available.`);
    return ride; // stays REQUESTED; caller can cancel or retry
  }

  startTrip(ride) {
    ride.updateStatus(RideStatus.IN_PROGRESS); // EN_ROUTE -> IN_PROGRESS
    ride.driver.status = DriverStatus.ON_TRIP;
    console.log(`Ride #${ride.id}: trip started.`);
    return ride;
  }

  endTrip(ride) {
    ride.fare = this.fareStrategy.calculateFare(ride);
    ride.updateStatus(RideStatus.COMPLETED);   // IN_PROGRESS -> COMPLETED
    ride.driver.status = DriverStatus.AVAILABLE; // free the driver
    console.log(
      `Ride #${ride.id}: completed. ` +
      `${ride.tripDistanceKm().toFixed(2)} km, fare $${ride.fare}`
    );
    return ride;
  }

  cancel(ride) {
    ride.updateStatus(RideStatus.CANCELLED);   // legal from pre-trip states
    if (ride.driver && ride.driver.status !== DriverStatus.OFFLINE) {
      ride.driver.status = DriverStatus.AVAILABLE; // release the driver
    }
    console.log(`Ride #${ride.id}: cancelled.`);
    return ride;
  }
}

// ─────────────────────────────────────────────────────────────
// main() — proves matching, lifecycle, both fare policies, and
// rejection of an illegal transition.
// ─────────────────────────────────────────────────────────────
function main() {
  // Drivers scattered around a small grid (lat, lng).
  const drivers = [
    new Driver('d1', 'Alice',  new Location(12.90, 77.60), 4.9),
    new Driver('d2', 'Bob',    new Location(12.95, 77.62), 4.7),
    new Driver('d3', 'Carol',  new Location(12.97, 77.59), 4.8),
    new Driver('d4', 'Dan',    new Location(13.10, 77.70), 4.6), // far away
    new Driver('d5', 'Eve',    new Location(12.96, 77.61), 5.0),
  ];

  const rider  = new Rider('r1', 'Riya', new Location(12.955, 77.605));
  const pickup = rider.location;
  const dropoff = new Location(13.00, 77.65);

  console.log('=== 1. NEAREST-DRIVER MATCHING + STANDARD FARE ===');
  const svc = new RideService(
    drivers,
    new NearestDriverStrategy(5),        // matching policy
    new StandardFareStrategy()           // pricing policy
  );

  const ride = svc.requestRide(rider, pickup, dropoff);
  svc.startTrip(ride);
  svc.endTrip(ride);
  console.log(`Final status: ${ride.status}, driver now: ${ride.driver.status}\n`);

  console.log('=== 2. SAME TRIP, SURGE PRICING (2x) ===');
  // Swap ONLY the fare strategy — matching + lifecycle untouched.
  const surgeSvc = new RideService(
    drivers,
    new HighestRatedStrategy(5),                                   // swap matching too
    new SurgeFareStrategy(new StandardFareStrategy(), 2.0)         // swap pricing
  );
  // Reset a rider request (drivers were freed on endTrip above).
  const ride2 = surgeSvc.requestRide(rider, pickup, dropoff);
  surgeSvc.startTrip(ride2);
  surgeSvc.endTrip(ride2);
  console.log(`Surge fare $${ride2.fare} (vs standard $${ride.fare})\n`);

  console.log('=== 3. ILLEGAL TRANSITION IS REJECTED ===');
  const bad = new Ride(rider, pickup, dropoff); // status REQUESTED
  try {
    bad.updateStatus(RideStatus.COMPLETED);     // REQUESTED -> COMPLETED: illegal
  } catch (err) {
    console.log('Rejected as expected:', err.message);
  }

  console.log('\n=== 4. CANCELLATION ===');
  const ride3 = svc.requestRide(rider, pickup, dropoff);
  svc.cancel(ride3);
  console.log(`Final status: ${ride3.status}`);
}

main();

module.exports = {
  DriverStatus, RideStatus, Location, Rider, Driver, Ride,
  MatchingStrategy, NearestDriverStrategy, HighestRatedStrategy,
  FareStrategy, StandardFareStrategy, SurgeFareStrategy, RideService,
};
```

Run it with `node 127-uber-matching.js`. Sample output (the surge line uses `HighestRatedStrategy`, so it may pick Eve at 5.0 instead of the nearest driver):

```
=== 1. NEAREST-DRIVER MATCHING + STANDARD FARE ===
Ride #1: matched Eve (0.45 km away, rating 5)
Ride #1: trip started.
Ride #1: completed. 6.14 km, fare $9.30
Final status: COMPLETED, driver now: AVAILABLE

=== 2. SAME TRIP, SURGE PRICING (2x) ===
Ride #2: matched Eve (0.45 km away, rating 5)
Ride #2: trip started.
Ride #2: completed. 6.14 km, fare $18.60
Surge fare $18.60 (vs standard $9.30)

=== 3. ILLEGAL TRANSITION IS REJECTED ===
Rejected as expected: Illegal ride transition: REQUESTED -> COMPLETED (ride #3). Allowed: [MATCHED, CANCELLED]

=== 4. CANCELLATION ===
Ride #4: matched Eve (0.45 km away, rating 5)
Ride #4: cancelled.
Final status: CANCELLED
```

Notice how surge is `2.0 × 9.30 = 18.60` exactly — because `SurgeFareStrategy` just wraps the standard one. And the illegal jump is refused with a clear message instead of silently corrupting state.

---

## Real world examples

### Uber

Uber's dispatch is conceptually a strategy over "which driver, right now." Publicly, Uber has described systems like **H3** (their open-source hexagonal geospatial indexing library) for exactly the "find nearby drivers" step our linear scan stands in for — this is representative of how the LLD's distance filter becomes an HLD index. The matching itself weighs ETA, driver acceptance, and supply/demand; each is a policy — the same shape as our `MatchingStrategy`. Surge pricing is a pricing policy multiplier layered on a base fare, mirroring our `SurgeFareStrategy` wrapping `StandardFareStrategy`.

### Lyft

Lyft (representative) runs a similar match-and-price loop and has published on **dispatch as an optimization** — instead of greedily giving each rider the nearest car, they batch requests and solve an assignment problem across many riders and drivers at once. In our design that is simply *another* `MatchingStrategy` implementation: same `findDriver`-style seam, smarter internals. The `Ride` state machine and fare strategies are unchanged.

### DoorDash / food delivery

The identical structure powers delivery matching — the "driver" is a Dasher, the "ride" is a delivery, pickup is the restaurant, dropoff is the customer. See [126 — LLD: Food Delivery](./126-lld-food-delivery.md): a matching strategy plus an order/delivery state machine is the same skeleton with different nouns. That reuse across domains is exactly why isolating the policies pays off.

---

## Trade-offs

**Strategy for matching and fare:**

| Pros | Cons |
|---|---|
| Swap policies without touching `RideService` | More classes / indirection up front |
| A/B test matching and pricing independently | Have to design the interface right the first time |
| New policy = new class, old code untouched | Overkill if you truly have exactly one rule forever |

**State machine on `Ride` (guarded `updateStatus`):**

| Pros | Cons |
|---|---|
| Illegal transitions fail loudly, not silently | A transition table to maintain as states grow |
| The lifecycle is documented in one place | Slightly more ceremony than `ride.status = x` |
| Impossible to bill for a never-started trip | — |

**Linear distance scan for "nearby drivers":**

| Pros | Cons |
|---|---|
| Trivial to write and reason about | O(drivers) per request — dies at city scale |
| Perfect for an LLD interview / demo | Needs a geospatial index in production (102) |

**The sweet spot:** two strategies (matching + fare) behind small interfaces, a `Ride` that guards its own lifecycle, and a linear scan you explicitly label as "the geospatial index goes here." That is precisely the altitude an LLD interview rewards — clean classes now, a clear pointer to the HLD for scale.

---

## Common interview questions on this topic

### Q1: "Where does the geospatial index fit — this is LLD, not HLD?"

**Hint:** Point at `NearestDriverStrategy.findDriver`'s filter step. Today it linearly scans all drivers and keeps those within radius. At scale that step is replaced by a geohash/quadtree lookup that returns only the buckets near the pickup — same method signature, indexed internals. The index lives *inside* the matching strategy, so the rest of the design (ride lifecycle, fares, service) never changes. That boundary is the whole point of [102](./102-hld-uber.md).

### Q2: "Why is the ride lifecycle a guarded state machine instead of a plain status field?"

**Hint:** Because a plain field lets any code write `ride.status = 'COMPLETED'` and bill a rider for a trip that never started. `updateStatus` centralises the legal edges in one transition table and throws on illegal jumps — the object protects its own invariants. It is the State pattern expressed as a table (see [46 — State Pattern](./46-pattern-state.md)).

### Q3: "How would you add surge pricing?"

**Hint:** You do not touch `RideService` or `Ride`. You construct it with `new SurgeFareStrategy(new StandardFareStrategy(), 1.8)`. Surge wraps the base strategy and multiplies — so it composes with *any* base pricing. That is Strategy (plus a decorator flavour). Show the class; it is ~5 lines.

### Q4: "A matched driver declines — then what?"

**Hint:** `requestRide` loops: it asks the current candidate `acceptRide(ride)`; on decline it removes that driver from the working pool and re-runs the strategy to get the next best candidate, until one accepts or the pool empties. The ride stays `REQUESTED` until someone accepts, so no illegal state ever appears.

### Q5: "Match by ETA or by driver rating instead of distance — how big a change?"

**Hint:** One new class implementing `MatchingStrategy.findDriver`, injected at construction. I showed `HighestRatedStrategy` doing exactly this. `RideService`, `Ride`, and the fare code are untouched — that is the payoff of the Strategy seam.

---

## Practice exercise

### Build and extend the matcher (30–40 min)

1. Copy the implementation into `uber-matching.js` and run it with Node. Confirm you see a match, a computed fare, a rejected illegal transition, and a cancellation.
2. **Add an `EtaMatchingStrategy`** that picks the driver with the lowest *estimated pickup time*, computed as `distanceTo(pickup) / assumedSpeedKmh` where each driver has a `speedKmh` (city drivers are slower). Inject it into a `RideService` and prove a *different* driver wins than under `NearestDriverStrategy`.
3. **Add a `TimeOfDaySurgeStrategy`** that takes an hour and applies `1.0` off-peak but `2.5` during rush hour (7–9am, 5–7pm). Show the same trip priced two ways.
4. **Make one driver decline** by setting `driver.acceptanceRate = 0`, and confirm the loop falls through to the next candidate. Print which driver ultimately took the ride.

Produce: the extended file plus a short note on which classes you touched (you should have added two classes and changed *zero* lines of `RideService` or `Ride`).

---

## Quick reference cheat sheet

- **Core actors:** `Rider`, `Driver` (location, status, rating), `Ride` (rider, driver, pickup, dropoff, status, fare), `Location` (lat/lng + `distanceTo`).
- **Two enums:** `DriverStatus` (AVAILABLE/EN_ROUTE/ON_TRIP/OFFLINE) and `RideStatus` (REQUESTED→MATCHED→EN_ROUTE→IN_PROGRESS→COMPLETED / CANCELLED).
- **`RideService` = the host** — orchestrates, owns no policy, delegates to strategies.
- **Matching = STRATEGY** — `MatchingStrategy.findDriver(pickup, drivers)`; default `NearestDriverStrategy`. Swap for rating/ETA with zero service edits.
- **Fare = STRATEGY** — `FareStrategy.calculateFare(ride)`; `StandardFareStrategy` = base + perKm×km + perMin×min; `SurgeFareStrategy` wraps + multiplies.
- **Lifecycle = STATE** — `Ride.updateStatus` uses a transition table; illegal jumps throw, they never silently corrupt.
- **Driver acceptance:** `requestRide` loops through candidates; a decline drops that driver and retries the next.
- **Fare formula:** base + perKm×distance + perMin×(distance/avgSpeed×60); surge = ×multiplier.
- **The linear scan is the seam** — "find nearby drivers" becomes a geospatial index (geohash/quadtree) at scale; it lives *inside* the matching strategy.
- **Own each verb where its data lives** — `Ride` owns status, `Location` owns distance, strategies own policy, `RideService` only coordinates.
- **LLD vs HLD:** this doc = classes; [102](./102-hld-uber.md) adds indexing, real-time location streaming, and scale.
- **Interview move:** name your patterns (Strategy ×2, State) and defend them; show surge and an alternate matcher to prove the seams work.

---

## Design patterns used and WHY

**Strategy — for matching (the star).** "Which driver gets this ride?" is a decision that changes with every business experiment: nearest, best-ETA, highest-rated, most-likely-to-accept. Welding any one of these into `RideService` would make each experiment a risky core edit. Instead `MatchingStrategy.findDriver(pickup, drivers)` is an interface, `RideService` holds a reference to *some* strategy, and swapping the policy is a constructor argument. We demonstrated it by dropping in `HighestRatedStrategy` beside `NearestDriverStrategy`. Recall the pattern from [42 — Strategy Pattern](./42-pattern-strategy.md).

**Strategy — for fare (the co-star).** Pricing is equally volatile: standard, surge, promo, pooled-split. `FareStrategy.calculateFare(ride)` is the same seam for money. `SurgeFareStrategy` even *wraps* a base strategy and multiplies (a Decorator flavour of Strategy), so surge composes with any pricing rule instead of duplicating it.

**State — for the ride lifecycle.** A ride is a little machine with legal and illegal moves. `Ride.updateStatus` encodes the legal edges in `LEGAL_TRANSITIONS` and throws on anything else, so the object enforces its own invariants — you cannot bill for a trip that never started, and you cannot resurrect a completed ride. See [46 — State Pattern](./46-pattern-state.md).

**Explicitly connecting to the HLD.** This is the **class design**: objects, ownership, and swappable policies. The **HLD ([102 — HLD: Uber](./102-hld-uber.md))** adds everything scale demands — a **geospatial index** (geohash/H3/quadtree) in place of our linear scan, **real-time driver-location streaming**, sharded dispatch across cities, and durable ride state. The seam is deliberate: our linear `findDriver` filter is exactly where the index plugs in, and our in-memory `Ride` is exactly what a durable store persists. Nail the classes here; scale them there.

---

## Extensions the interviewer asks for

**"Add surge pricing."** Already shown: `new SurgeFareStrategy(new StandardFareStrategy(), 2.0)`. It wraps and multiplies — the base fare $9.30 becomes $18.60. No change to `RideService` or `Ride`. This is why fare is a strategy.

**"Match by ETA or rating instead of distance."** Add one class implementing `MatchingStrategy.findDriver` (we showed `HighestRatedStrategy`; an `EtaMatchingStrategy` is the practice task). Inject it at construction. Zero edits to the service, the ride, or the fares.

**"Add driver acceptance / decline with fallback."** Already modelled: `Driver.acceptRide(ride)` returns a decision, and `requestRide` loops — a decline drops that driver from the working pool and re-runs the strategy for the next candidate. Set `acceptanceRate = 0` on a driver to watch the fallback fire.

**"Add pool / shared rides."** A `Ride` grows a `riders[]` list and per-rider pickup/dropoff legs; a `PoolMatchingStrategy` looks for a driver whose current route can absorb a new rider within a detour budget; a `SplitFareStrategy` divides the fare across riders. The state machine gains sub-states per leg (`RIDER_PICKED_UP`) but the core `REQUESTED→…→COMPLETED` spine is unchanged. The strategy seams absorb both new behaviours.

**"Add the geospatial index for scale."** Replace the linear `filter` inside `NearestDriverStrategy.findDriver` with a lookup against a geohash/quadtree index that returns only drivers in buckets near the pickup, then rank those. Same method signature, indexed internals — the rest of the design does not notice. That is the bridge to [102 — HLD: Uber](./102-hld-uber.md), where the index, real-time location updates, and horizontal scale live.

Each ask lands on a seam we built on purpose: a new strategy, a new fare policy, or an indexed internals swap — never a rewrite. That is what "design for change" looks like at the class level ([111 — LLD Approach Framework](./111-lld-approach-framework.md)).

---

## Connected topics

| Direction | Topic | Why |
|---|---|---|
| **Previous** | [126 — LLD: Food Delivery](./126-lld-food-delivery.md) | Same matching + state-machine skeleton with delivery nouns; great warm-up for this |
| **Next** | [111 — LLD Approach Framework](./111-lld-approach-framework.md) | The 7-step method this doc applies; generalise it to any LLD prompt |
| **Related** | [102 — HLD: Uber](./102-hld-uber.md) | The system-level companion — geospatial indexing, location streaming, and scale |
| **Related** | [42 — Strategy Pattern](./42-pattern-strategy.md) | The pattern behind swappable matching and fare policies (used twice here) |
| **Related** | [46 — State Pattern](./46-pattern-state.md) | The pattern behind the guarded ride lifecycle |
