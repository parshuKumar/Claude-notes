# 112 — Design a Parking Lot System
## Category: LLD Case Study

---

## What is this?

A parking lot system is the software that runs a multi-floor parking garage: it hands you a ticket when you drive in, remembers where your car sits, and charges you a fee based on how long you stayed when you leave.

Think of it like a **coat check at a fancy event**. You hand over your coat (park your car), you get a numbered stub (a ticket), the attendant hangs it on the right-sized hook (a spot that fits your vehicle), and when you come back with the stub they find your coat, work out any charge, and hand it over. Our job is to design the *classes* behind that coat check — which objects exist, what each one knows, and what each one does.

---

## Why does it matter?

The parking lot is the **canonical first LLD (Low-Level Design) problem**. It is small enough to hold in your head, but big enough to force every core object-oriented skill: turning a vague problem into classes, choosing inheritance vs. composition, deciding *who owns which behaviour*, and swapping in patterns like Strategy so the design bends instead of breaking.

**Interview angle:** This exact question ("design a parking lot") is asked at Amazon, Google, and dozens of others. The interviewer is not testing whether you know parking — they are watching *how you go from words to classes*. Do you ask clarifying questions? Do you avoid a single god-class stuffed with every method? Can you extend the design when they say "now add EV charging"?

**Real-work angle:** Every feature you will ever build starts as fuzzy requirements that you must carve into objects with clear responsibilities. If you can do it cleanly for a parking lot, you can do it for a booking system, a payment flow, or an inventory service. The muscle is identical.

This document walks the **full 7-step LLD method from [111 — LLD Approach Framework](./111-lld-approach-framework.md)** end to end, visibly, because it is your first complete class-level design. Watch the method, not just the answer.

---

## The core idea — explained simply

### The analogy: a hotel with themed floors

Picture a **hotel that only rents parking, not rooms**.

- The **hotel** (`ParkingLot`) is the whole building. It is the front desk you talk to. You never talk to a room directly — you ask the desk, "I need a room," and it figures out the rest.
- Each **floor** (`ParkingFloor`) is one level of the garage. The hotel *is made of* its floors — tear down the hotel and the floors are gone too.
- Each **room** (`ParkingSpot`) is one parking space. Rooms come in sizes: single, double, suite (Motorcycle, Compact, Large). A big guest cannot squeeze into a tiny room, but a small guest *can* take a bigger room if that is all that is left.
- A **guest** (`Vehicle`) arrives wanting a room. Guests come in sizes too.
- The **key card** (`Ticket`) records which room you got and the exact minute you checked in.
- The **bill** (`Payment` + fee calculation) is worked out at checkout from your check-in time.

The single most important idea: **you talk only to the front desk.** You say `parkVehicle(car)` and the desk orchestrates floors, spots, and tickets for you. That is *encapsulation* — the caller does not need to know the building's internals.

| Real world (hotel) | Our class | What it knows | What it does |
|---|---|---|---|
| The hotel building | `ParkingLot` | its floors, its pricing rule | park a vehicle, unpark, quote a fee |
| A floor / level | `ParkingFloor` | its spots | find a free spot that fits |
| A room | `ParkingSpot` | its size, is it occupied, by whom | can it fit a vehicle, assign, release |
| A guest | `Vehicle` | its type & plate | (mostly data) |
| The key card | `Ticket` | spot, vehicle, check-in time | (mostly data) |
| The bill rule | `FeeStrategy` | how to price | compute a fee from a duration |

Keep this table nearby. Every class below is just one row of it, made concrete.

---

## Key concepts inside this topic

We follow the 7-step method from [111 — LLD Approach Framework](./111-lld-approach-framework.md). Each numbered part below *is* a step.

### 1. Requirements & clarifying questions

Before writing a single class, pin down scope. In an interview you *say these out loud* — it shows maturity and earns you a tighter problem.

**Functional requirements (what it must do):**

- The lot has **multiple floors**.
- Each floor has **multiple spots** of different **sizes**: Motorcycle, Compact, Large.
- A vehicle takes an **appropriately-sized spot**. A Motorcycle fits anywhere; a Truck needs a Large spot.
- On entry, **issue a ticket** stamped with the entry time and assigned spot.
- On exit, **calculate a fee** based on how long the vehicle stayed.
- Support **multiple entry and exit gates**.
- **Track availability** — reject a vehicle when nothing fits.

**Clarifying questions to ask the interviewer** (and sensible default answers):

| Question | Why it matters | Default we assume |
|---|---|---|
| What payment methods? Cash, card, app? | Changes the `Payment` design | Assume pluggable; model one `Payment` type |
| Can a big car take a small spot? | Affects `canFit` logic | No. A vehicle needs a spot ≥ its size |
| Can a small car take a big spot? | Affects spot search | Yes, if nothing smaller is free |
| Handicapped / EV / reserved spots? | Extra `SpotType`s | Out of scope for v1; show as an extension |
| Monthly passes / subscriptions? | Alternate billing path | Out of scope for v1; show as an extension |
| Do we need real-time display boards? | UI concern, not core model | Out of scope; expose a `getAvailability()` |

**Non-functional (quietly important):** correctness under concurrent gate entries (two cars must never get the same spot), and *extensibility* — new spot types and pricing rules should slot in without rewrites.

**Scope discipline:** we deliberately keep v1 tight — three spot sizes, one pricing rule at a time, single-process. Everything else is an *extension* (Part 7). Narrow scope, then expand on request, is exactly what interviewers reward.

### 2. Identify core objects (the nouns)

Underline the nouns in the requirements, then filter to the ones that carry state or behaviour. This is the "noun extraction" step from topic 111.

Raw nouns: *lot, floor, spot (motorcycle/compact/large), vehicle (motorcycle/car/truck), ticket, entry gate, exit gate, payment, fee, rate, time, availability.*

After filtering:

| Noun | Becomes | Why |
|---|---|---|
| Parking lot | `ParkingLot` class | Top-level orchestrator |
| Floor | `ParkingFloor` class | Groups spots |
| Spot | `ParkingSpot` class (+ subtypes) | Has state (occupied?) and behaviour (`canFit`) |
| Vehicle | `Vehicle` class (+ subtypes) | Carries type & plate |
| Ticket | `Ticket` class | Records the parking session |
| Payment | `Payment` (light, extension) | Records a settled charge |
| Rate / fee rule | `FeeStrategy` interface | Pluggable pricing |
| **Vehicle type** | **enum** via `Object.freeze` | Fixed set of constant labels |
| **Spot type** | **enum** via `Object.freeze` | Fixed set of constant labels |
| Time, availability | *attributes*, not classes | They are data on other objects |
| Entry/exit gate | Often just methods on `ParkingLot` | v1 has no gate-specific state |

Note the two nouns that become **enums**, not classes: `VehicleType` and `SpotType`. An enum is the right tool when a value is one of a small, fixed, named set. In JavaScript we fake enums with `Object.freeze` so the values cannot be mutated.

### 3. Behaviours (the verbs) and who owns them

Now the verbs: *park, unpark, find a spot, assign a spot, issue a ticket, calculate a fee, process payment.* The critical skill is assigning each verb to the object that *owns the data it needs*. This is what kills the **anemic model** — the anti-pattern where all data sits in dumb structs and one giant `ParkingManager` class does everything.

| Behaviour | Owner | Why that owner |
|---|---|---|
| `parkVehicle(vehicle)` | `ParkingLot` | It orchestrates: find, assign, ticket |
| `unparkVehicle(ticket)` | `ParkingLot` | It orchestrates: release, price |
| `findAvailableSpot(type)` | `ParkingFloor` | The floor knows its own spots |
| `canFit(vehicle)` / `assign` / `remove` / `isAvailable` | `ParkingSpot` | The spot alone knows its size & occupancy |
| `calculateFee(ticket)` | `FeeStrategy` | Pricing is a swappable policy, not lot internals |
| entry time / duration | `Ticket` | The ticket *is* the record of the session |
| `processPayment(fee)` | `Payment` (extension) | Payment is its own concern |

**Avoid the god class.** A tempting-but-wrong design puts `findSpot`, `assign`, `calculateFee`, `charge`, and every field inside one `ParkingManager`. It compiles, but every change touches that one file and nothing is reusable. Instead: the *spot* decides if it fits, the *floor* searches its spots, the *ticket* remembers the time, the *strategy* prices. The `ParkingLot` only *coordinates* them — it is thin. Distributed responsibility is the whole point of object design. (See [23 — Class Relationships](./23-class-relationships.md) for how these objects associate.)

### 4. Class diagram (ASCII)

```
                          ┌──────────────────────────┐
                          │        ParkingLot         │   ◇ = composition
                          │──────────────────────────│   ◆ = strong ownership
                          │ - floors: ParkingFloor[]  │
                          │ - feeStrategy: FeeStrategy │────────────┐ "has a" (aggregation:
                          │ - activeTickets: Map      │             │  strategy injected)
                          │──────────────────────────│             ▼
                          │ + parkVehicle(v): Ticket  │      ┌───────────────────┐
                          │ + unparkVehicle(t): fee   │      │  «interface»      │
                          │ + getAvailability()       │      │   FeeStrategy     │
                          └─────────────┬────────────┘      │───────────────────│
                                        │ ◆ 1..*             │ + calculateFee(t) │
                                        ▼   (composed of)    └─────────┬─────────┘
                          ┌──────────────────────────┐                 │ implements
                          │       ParkingFloor        │      ┌──────────┴──────────┐
                          │──────────────────────────│      ▼                     ▼
                          │ - floorNumber: int        │  ┌──────────────┐  ┌──────────────┐
                          │ - spots: ParkingSpot[]    │  │HourlyRate    │  │FlatRate      │
                          │──────────────────────────│  │Strategy      │  │Strategy      │
                          │ + findAvailableSpot(type) │  └──────────────┘  └──────────────┘
                          └─────────────┬────────────┘
                                        │ ◆ 1..*  (composed of spots)
                                        ▼
                          ┌──────────────────────────┐        ┌────────────────────┐
                          │   ParkingSpot (base)      │        │       Ticket        │
                          │──────────────────────────│        │────────────────────│
                          │ - id, type: SpotType      │        │ - id               │
                          │ - vehicle: Vehicle | null │◄───────│ - spot: ParkingSpot │ (references)
                          │──────────────────────────│  1  1  │ - vehicle: Vehicle  │
                          │ + canFit(vehicle): bool   │        │ - entryTime: Date   │
                          │ + assign(v) / remove()    │        └─────────┬──────────┘
                          │ + isAvailable(): bool     │                  │ references
                          └───────────△──────────────┘                  ▼
                              ┌───────┼────────┐            ┌────────────────────┐
                              ▼       ▼        ▼            │   Vehicle (base)    │
                        Motorcycle Compact  Large           │────────────────────│
                          Spot      Spot     Spot           │ - plate, type       │
                        (subclasses set their SpotType)     └─────────△──────────┘
                                                             ┌────────┼────────┐
                                                             ▼        ▼        ▼
                                                       Motorcycle    Car     Truck
```

**Reading the relationships (recall [23 — Class Relationships](./23-class-relationships.md)):**

- `ParkingLot ◆— ParkingFloor ◆— ParkingSpot` is **composition** (the filled diamond). A floor does not exist without its lot; a spot does not exist without its floor. They share a lifetime — destroy the lot and everything inside vanishes.
- `ParkingLot ◇— FeeStrategy` is **aggregation** (open diamond). The strategy is *injected* and can be swapped; it has an independent life. This is dependency injection, and it is why pricing is pluggable.
- `Ticket → Vehicle` and `Ticket → ParkingSpot` are **associations** (plain references). A ticket merely *points at* a vehicle and a spot; it does not own them.
- The triangles (△) are **inheritance**: `Motorcycle/Car/Truck` are `Vehicle`s; the spot sizes are `ParkingSpot`s.

Multiplicities: one lot has `1..*` floors; one floor has `1..*` spots; one spot holds `0..1` vehicles; one ticket references exactly `1` spot and `1` vehicle.

### 5. Full JavaScript implementation

Real, complete, runnable. Save as `parking-lot.js` and run `node parking-lot.js`.

```javascript
'use strict';

// ─────────────────────────────────────────────────────────────
// ENUMS — a fixed, named set of constants. Object.freeze makes
// them immutable so nobody can do VehicleType.CAR = 'bike' by
// accident. This is the JS idiom for an enum.
// ─────────────────────────────────────────────────────────────
const VehicleType = Object.freeze({
  MOTORCYCLE: 'MOTORCYCLE',
  CAR: 'CAR',
  TRUCK: 'TRUCK',
});

const SpotType = Object.freeze({
  MOTORCYCLE: 'MOTORCYCLE', // smallest
  COMPACT: 'COMPACT',       // medium
  LARGE: 'LARGE',           // biggest
});

// A numeric "size rank" lets us compare who fits where.
// A vehicle fits a spot when spotRank >= vehicleRank.
const SIZE_RANK = Object.freeze({
  MOTORCYCLE: 1,
  COMPACT: 2,
  LARGE: 3,
  // Vehicle -> minimum spot size it needs:
  CAR: 2,    // a car needs at least a COMPACT spot
  TRUCK: 3,  // a truck needs a LARGE spot
});

// ─────────────────────────────────────────────────────────────
// VEHICLE hierarchy.
// Design choice: base class + thin subclasses vs. one class with a
// `type` field. We use subclasses because it reads clearly and lets
// us attach type-specific behaviour later (e.g. Truck.axleCount).
// Each subclass just fixes its VehicleType — minimal but explicit.
// ─────────────────────────────────────────────────────────────
class Vehicle {
  constructor(licensePlate, type) {
    if (new.target === Vehicle) {
      throw new Error('Vehicle is abstract; instantiate a subclass.');
    }
    this.licensePlate = licensePlate;
    this.type = type;                 // one of VehicleType
    this.requiredRank = SIZE_RANK[type]; // smallest spot rank it can use
  }
}

class Motorcycle extends Vehicle {
  constructor(plate) { super(plate, VehicleType.MOTORCYCLE); }
}
class Car extends Vehicle {
  constructor(plate) { super(plate, VehicleType.CAR); }
}
class Truck extends Vehicle {
  constructor(plate) { super(plate, VehicleType.TRUCK); }
}

// ─────────────────────────────────────────────────────────────
// PARKING SPOT hierarchy. The spot is NOT anemic: it owns the
// answer to "can this vehicle fit?" and "am I free?". Only the
// spot knows its own occupancy, so only the spot decides.
// ─────────────────────────────────────────────────────────────
class ParkingSpot {
  constructor(id, type) {
    if (new.target === ParkingSpot) {
      throw new Error('ParkingSpot is abstract; instantiate a subclass.');
    }
    this.id = id;
    this.type = type;             // one of SpotType
    this.rank = SIZE_RANK[type];  // how big this spot is
    this.vehicle = null;          // null = free, else the parked vehicle
  }

  isAvailable() {
    return this.vehicle === null;
  }

  // A vehicle fits when the spot is at least as big as it needs
  // AND the spot is currently free.
  canFit(vehicle) {
    return this.isAvailable() && this.rank >= vehicle.requiredRank;
  }

  assign(vehicle) {
    if (!this.canFit(vehicle)) {
      throw new Error(`Spot ${this.id} cannot fit ${vehicle.licensePlate}`);
    }
    this.vehicle = vehicle;
  }

  remove() {
    const v = this.vehicle;
    this.vehicle = null;
    return v;
  }
}

class MotorcycleSpot extends ParkingSpot {
  constructor(id) { super(id, SpotType.MOTORCYCLE); }
}
class CompactSpot extends ParkingSpot {
  constructor(id) { super(id, SpotType.COMPACT); }
}
class LargeSpot extends ParkingSpot {
  constructor(id) { super(id, SpotType.LARGE); }
}

// ─────────────────────────────────────────────────────────────
// PARKING FLOOR. Owns its spots and knows how to search them.
// findAvailableSpot picks the SMALLEST spot that still fits, so a
// motorcycle does not waste a large spot. (That "smallest-first"
// choice is itself a strategy — see Part 6.)
// ─────────────────────────────────────────────────────────────
class ParkingFloor {
  constructor(floorNumber, spots) {
    this.floorNumber = floorNumber;
    this.spots = spots; // ParkingSpot[]
  }

  findAvailableSpot(vehicle) {
    // Sort candidates by size so we hand out the tightest fit.
    return this.spots
      .filter((s) => s.canFit(vehicle))
      .sort((a, b) => a.rank - b.rank)[0] || null;
  }

  availableCount() {
    return this.spots.filter((s) => s.isAvailable()).length;
  }
}

// ─────────────────────────────────────────────────────────────
// TICKET. Mostly a record: which spot, which vehicle, when it
// entered. It OWNS the entry time, so it can report a duration.
// ─────────────────────────────────────────────────────────────
let ticketCounter = 0;
class Ticket {
  constructor(spot, vehicle, entryTime = new Date()) {
    this.id = `T-${++ticketCounter}`;
    this.spot = spot;         // reference (association)
    this.vehicle = vehicle;   // reference (association)
    this.entryTime = entryTime;
    this.exitTime = null;
  }

  // Duration in hours, rounded UP — 61 minutes bills as 2 hours,
  // which is how real garages charge (any part of an hour).
  durationHours(now = new Date()) {
    const ms = now - this.entryTime;
    return Math.max(1, Math.ceil(ms / (1000 * 60 * 60)));
  }
}

// ─────────────────────────────────────────────────────────────
// FEE STRATEGY — the STRATEGY pattern (recall 42). Pricing is a
// family of interchangeable algorithms behind one interface. We
// inject the chosen one into ParkingLot, so we can switch pricing
// without touching the lot. calculateFee is the single method of
// the "interface"; JS has no interface keyword, so a base class
// documents the contract.
// ─────────────────────────────────────────────────────────────
class FeeStrategy {
  calculateFee(_ticket) {
    throw new Error('calculateFee must be implemented by a subclass.');
  }
}

// Charge per started hour, with a rate that depends on vehicle type.
class HourlyRateStrategy extends FeeStrategy {
  constructor(hourlyRates = { MOTORCYCLE: 1, CAR: 2, TRUCK: 4 }) {
    super();
    this.hourlyRates = hourlyRates;
  }
  calculateFee(ticket) {
    const hours = ticket.durationHours(ticket.exitTime || new Date());
    const rate = this.hourlyRates[ticket.vehicle.type];
    return hours * rate;
  }
}

// One flat price regardless of duration — e.g. an event lot.
class FlatRateStrategy extends FeeStrategy {
  constructor(flatFee = 10) {
    super();
    this.flatFee = flatFee;
  }
  calculateFee(_ticket) {
    return this.flatFee;
  }
}

// ─────────────────────────────────────────────────────────────
// PARKING LOT — the top-level orchestrator. It is THIN: it finds a
// spot, assigns it, mints a ticket; on exit it releases the spot
// and asks the strategy for a fee. It never re-implements what a
// spot, floor, ticket, or strategy already knows.
// ─────────────────────────────────────────────────────────────
class ParkingLot {
  constructor(name, floors, feeStrategy) {
    this.name = name;
    this.floors = floors;            // ParkingFloor[]
    this.feeStrategy = feeStrategy;  // injected — swappable pricing
    this.activeTickets = new Map();  // ticketId -> Ticket
  }

  // Returns a Ticket, or throws if the lot is full for this vehicle.
  parkVehicle(vehicle) {
    for (const floor of this.floors) {
      const spot = floor.findAvailableSpot(vehicle);
      if (spot) {
        spot.assign(vehicle);               // spot flips to occupied
        const ticket = new Ticket(spot, vehicle);
        this.activeTickets.set(ticket.id, ticket);
        console.log(
          `[IN ] ${vehicle.type} ${vehicle.licensePlate} -> ` +
          `floor ${floor.floorNumber}, spot ${spot.id} (${spot.type}). ` +
          `Ticket ${ticket.id}`
        );
        return ticket;
      }
    }
    throw new Error(`Lot full: no spot fits ${vehicle.type} ${vehicle.licensePlate}`);
  }

  // Returns the fee owed. Frees the spot and closes the ticket.
  unparkVehicle(ticket) {
    if (!this.activeTickets.has(ticket.id)) {
      throw new Error(`Unknown or already-closed ticket ${ticket.id}`);
    }
    ticket.exitTime = new Date();
    const fee = this.feeStrategy.calculateFee(ticket); // policy decides price
    ticket.spot.remove();                              // spot flips to free
    this.activeTickets.delete(ticket.id);
    console.log(
      `[OUT] ${ticket.vehicle.type} ${ticket.vehicle.licensePlate} ` +
      `from spot ${ticket.spot.id}. Fee = $${fee}`
    );
    return fee;
  }

  getAvailability() {
    return this.floors.map((f) => ({
      floor: f.floorNumber,
      free: f.availableCount(),
    }));
  }
}

// ─────────────────────────────────────────────────────────────
// A tiny factory to build a floor of mixed spots quickly.
// (Factory is discussed in Part 6 — creating objects by type.)
// ─────────────────────────────────────────────────────────────
function buildFloor(floorNumber, { motorcycle, compact, large }) {
  const spots = [];
  for (let i = 0; i < motorcycle; i++) spots.push(new MotorcycleSpot(`${floorNumber}-M${i}`));
  for (let i = 0; i < compact; i++)    spots.push(new CompactSpot(`${floorNumber}-C${i}`));
  for (let i = 0; i < large; i++)      spots.push(new LargeSpot(`${floorNumber}-L${i}`));
  return new ParkingFloor(floorNumber, spots);
}

// ─────────────────────────────────────────────────────────────
// DEMO — proves it actually runs.
// ─────────────────────────────────────────────────────────────
function main() {
  // Two floors, each with a mix of spots.
  const floors = [
    buildFloor(1, { motorcycle: 1, compact: 2, large: 1 }),
    buildFloor(2, { motorcycle: 1, compact: 1, large: 1 }),
  ];

  // Inject an hourly pricing strategy. Swap to FlatRateStrategy in one line.
  const lot = new ParkingLot('Downtown Garage', floors, new HourlyRateStrategy());

  console.log('Availability at open:', lot.getAvailability());

  // Park a few vehicles.
  const bike = new Motorcycle('KA-01-BIKE');
  const car1 = new Car('KA-02-CAR1');
  const car2 = new Car('KA-03-CAR2');
  const truck = new Truck('KA-04-TRUK');

  const tBike = lot.parkVehicle(bike);
  const tCar1 = lot.parkVehicle(car1);
  const tCar2 = lot.parkVehicle(car2);
  const tTruck = lot.parkVehicle(truck);

  console.log('Availability after 4 parked:', lot.getAvailability());

  // Simulate time passing for the bike: pretend it entered 3h ago.
  tBike.entryTime = new Date(Date.now() - 3 * 60 * 60 * 1000);

  // Unpark and see fees. Bike: 3h * $1 = $3. Others: ~1h * rate.
  lot.unparkVehicle(tBike);
  lot.unparkVehicle(tCar1);
  lot.unparkVehicle(tTruck);

  console.log('Availability after 3 left:', lot.getAvailability());

  // car2 is still parked; tCar2 stays open.
  console.log(`Open tickets: ${lot.activeTickets.size} (expect 1: ${tCar2.id})`);
}

main();

module.exports = {
  VehicleType, SpotType, Vehicle, Motorcycle, Car, Truck,
  ParkingSpot, MotorcycleSpot, CompactSpot, LargeSpot,
  ParkingFloor, Ticket, FeeStrategy, HourlyRateStrategy,
  FlatRateStrategy, ParkingLot, buildFloor,
};
```

**Expected output (times will vary slightly):**

```
Availability at open: [ { floor: 1, free: 4 }, { floor: 2, free: 3 } ]
[IN ] MOTORCYCLE KA-01-BIKE -> floor 1, spot 1-M0 (MOTORCYCLE). Ticket T-1
[IN ] CAR KA-02-CAR1 -> floor 1, spot 1-C0 (COMPACT). Ticket T-2
[IN ] CAR KA-03-CAR2 -> floor 1, spot 1-C1 (COMPACT). Ticket T-3
[IN ] TRUCK KA-04-TRUK -> floor 1, spot 1-L0 (LARGE). Ticket T-4
Availability after 4 parked: [ { floor: 1, free: 0 }, { floor: 2, free: 3 } ]
[OUT] MOTORCYCLE KA-01-BIKE from spot 1-M0. Fee = $3
[OUT] CAR KA-02-CAR1 from spot 1-C0. Fee = $2
[OUT] TRUCK KA-04-TRUK from spot 1-L0. Fee = $4
Availability after 3 left: [ { floor: 1, free: 3 }, { floor: 2, free: 3 } ]
Open tickets: 1 (expect 1: T-3)
```

Notice how the **motorcycle took the motorcycle spot, not a large one** — the smallest-fit search paid off. And the bike's $3 fee came straight from the injected `HourlyRateStrategy` doing `3 hours × $1`.

### 6. Design patterns used, and WHY

Name your patterns in the interview — it signals fluency.

| Pattern | Where | Why we used it |
|---|---|---|
| **Strategy** ([42](./42-pattern-strategy.md)) | `FeeStrategy` → `HourlyRateStrategy`, `FlatRateStrategy` | Pricing rules change per lot and per business deal. Strategy makes pricing a pluggable object injected into `ParkingLot`, so a new rule is a new class — zero edits to the lot. Spot-assignment (smallest-fit vs. nearest-gate) is a *second* Strategy point. |
| **Factory** ([30](./30-pattern-factory-method.md)) | `buildFloor` (and an optional `VehicleFactory`) | Centralizes the messy "create the right object for this type" logic. A `VehicleFactory.create(type, plate)` keeps `new Motorcycle/Car/Truck` decisions in one place, so gate code does not `switch` on type everywhere. |
| **Singleton** | The `ParkingLot` instance | A garage has exactly one lot, so it is tempting to make `ParkingLot` a singleton. It works, but a hard singleton makes testing painful (global state). **Prefer dependency injection** — create one instance in `main()` and pass it where needed. Mention the singleton, then justify choosing DI instead. |

The star is **Strategy**. Recall from [42 — Strategy Pattern](./42-pattern-strategy.md): it is "define a family of algorithms, encapsulate each one, and make them interchangeable." Pricing is exactly that family. Because we injected `feeStrategy`, switching the whole lot to flat-rate event pricing is one line: `new ParkingLot(name, floors, new FlatRateStrategy(15))`.

### 7. Extensions the interviewer asks for

The real test comes now: "great — now add X." A good design *absorbs* each change with minimal edits. Watch how little we touch.

**"Add EV charging spots."** Add `SpotType.ELECTRIC` and an `ElectricSpot` subclass. If EV spots also charge for electricity, that is a pricing concern — extend the `FeeStrategy`, not the spot. New enum value + new subclass; `ParkingLot` and `ParkingFloor` are untouched because they only ever call `canFit`/`findAvailableSpot`. This is the Open/Closed Principle paying off.

**"Add monthly passes."** Introduce a `PassStrategy` (another Strategy) or a `MonthlyPassFeeStrategy` that returns `0` for pass-holders and defers to the hourly strategy otherwise. The `Ticket` gains an optional `passId`. No change to spots or floors.

**"Add multiple pricing tiers"** (peak/off-peak, weekday/weekend). Because pricing is already a Strategy, add a `TieredRateStrategy` that reads the entry/exit time and picks a rate. `ParkingLot` still just calls `calculateFee(ticket)` — it does not know or care.

**"Make it thread-safe for concurrent gates."** This is the sharp one (recall concurrency, topic 52). Two cars arriving at two gates at the same instant could both be handed the *same* free spot — a race condition. The fix is to make **find-and-assign atomic**: guard the critical section so only one gate can claim a spot at a time.

```javascript
// A minimal async mutex: serialize spot assignment across gates.
class Mutex {
  constructor() { this._chain = Promise.resolve(); }
  runExclusive(fn) {
    const result = this._chain.then(fn, fn);
    // keep the chain alive even if fn throws
    this._chain = result.then(() => {}, () => {});
    return result;
  }
}

// In ParkingLot: wrap the find+assign so only one caller runs it at a time.
async parkVehicleSafe(vehicle) {
  return this._mutex.runExclusive(() => this.parkVehicle(vehicle));
}
```

In a single-threaded Node process the danger is only around `await` boundaries; the mutex serializes the claim so two gates cannot interleave and double-book. In a multi-*instance* deployment you would push this to a database transaction or a distributed lock (see the availability/locking discussion in the fundamentals). The key teaching point: because *only the spot* mutates its own occupancy and *only the lot* orchestrates, the critical section is tiny and easy to guard. A god class would have made this nearly impossible.

Each extension touched one or two classes. **That is the payoff of good LLD** — and the exact thing the interviewer is measuring.

---

## Visual / Diagram description

A sequence diagram of one full park-and-leave cycle:

```
 Driver        ParkingLot        ParkingFloor      ParkingSpot        Ticket      FeeStrategy
   │                │                  │                │               │              │
   │ parkVehicle(v) │                  │                │               │              │
   │───────────────►│                  │                │               │              │
   │                │ findAvailableSpot(v)               │               │              │
   │                │─────────────────►│  canFit(v)?     │               │              │
   │                │                  │───────────────►│               │              │
   │                │                  │◄──── spot ──────│               │              │
   │                │◄──── spot ───────│                │               │              │
   │                │ assign(v)        │                │               │              │
   │                │───────────────────────────────────►│ (occupied)   │              │
   │                │ new Ticket(spot, v, now) ─────────────────────────►│              │
   │◄─── ticket ────│                  │                │               │              │
   │                │                  │                │               │              │
   │ unparkVehicle(ticket)             │                │               │              │
   │───────────────►│ calculateFee(ticket) ─────────────────────────────────────────►│
   │                │◄──────────────────── fee ──────────────────────────────────────│
   │                │ ticket.spot.remove() ─────────────►│ (free again) │              │
   │◄──── fee ──────│                  │                │               │              │
```

Read top to bottom as time. The `Driver` only ever talks to `ParkingLot`. The lot delegates: it asks the *floor* to find a spot, the floor asks each *spot* if it `canFit`, the *spot* flips to occupied on `assign`, a *ticket* is minted, and on exit the *strategy* prices the stay while the *spot* frees itself. No single object does everything — responsibility flows to whoever owns the data. That flow *is* the design.

---

## Real world examples

### Amazon (interview canon)

Amazon uses the parking-lot problem so often in LLD/OOD interview loops that it is practically a rite of passage. Interviewers there specifically probe the extension phase — "add reserved spots," "handle payment failure" — to see whether your class boundaries hold up. This is representative of the broad industry pattern rather than a claim about Amazon's internal garage software.

### SKIDATA / commercial garage vendors

Real commercial parking systems (SKIDATA, Amano, and similar vendors) run barrier gates that print a bar-coded ticket on entry and compute the fee at a pay station on exit — exactly the entry-ticket / timed-fee model here. Their pricing engines are configurable per site (hourly, daily cap, event flat-rate), which is precisely the Strategy pattern in production. Described conceptually; internal architectures are proprietary.

### Airport & mall multi-level garages

Large airport garages show live per-floor availability counts on entry boards — the same `getAvailability()` idea aggregated across floors — and vary rates by duration and day, matching the "multiple pricing tiers" extension. The availability boards are a display layer over the exact spot-tracking model we built.

---

## Trade-offs

**Subclasses per size vs. one class with a `type` field:**

| Approach | Pros | Cons |
|---|---|---|
| Subclass per vehicle/spot size | Reads clearly; room for type-specific behaviour; polymorphism | More classes; overkill if types never differ in behaviour |
| Single class + `type` enum | Fewer files; trivially data-driven | Type-specific logic becomes `switch` statements (a code smell) |

**Strategy for pricing vs. a `calculateFee` method on `ParkingLot`:**

| Approach | Pros | Cons |
|---|---|---|
| Strategy object (ours) | Pluggable, testable, obeys Open/Closed | One more class + injection wiring |
| Method on the lot | Fewer moving parts | Every new pricing rule edits the lot; hard to test in isolation |

**Singleton lot vs. dependency injection:**

| Approach | Pros | Cons |
|---|---|---|
| Singleton | Guarantees one instance; globally reachable | Global state; hard to test; hidden coupling |
| Dependency injection (ours) | Testable; explicit; multiple lots possible | Must pass the instance around |

**The sweet spot:** subclasses when behaviour genuinely differs (spots/vehicles here), a single class when it does not; Strategy for anything that will vary per deployment (pricing, spot-selection); and dependency injection over Singleton unless a true global is unavoidable.

---

## Common interview questions on this topic

### Q1: "How does a vehicle get assigned the right spot, and why not just take the first free one?"

**Hint:** Each `ParkingSpot.canFit(vehicle)` compares size ranks. `ParkingFloor.findAvailableSpot` filters to fitting spots then picks the *smallest* one, so a motorcycle does not occupy a large spot needed by a truck. "First free" wastes big spots — mention that spot-selection is itself a swappable Strategy (smallest-fit vs. nearest-gate).

### Q2: "Where does the fee logic live, and how would you add weekend pricing?"

**Hint:** Fee logic lives in a `FeeStrategy` object injected into `ParkingLot`, not in the lot itself. Weekend pricing is a new `TieredRateStrategy` class that inspects `ticket.entryTime`; the lot is untouched. Name the Strategy pattern explicitly.

### Q3: "Two cars reach two gates at the same moment — how do you stop a double-booking?"

**Hint:** The find-and-assign step is a critical section. Make it atomic with a lock/mutex (single process) or a DB transaction / distributed lock (multi-instance). Point out that only `ParkingSpot` mutates its own occupancy, so the section to guard is tiny (recall concurrency, topic 52).

### Q4: "Why not put all the logic in one ParkingManager class?"

**Hint:** That is the anemic-model / god-class anti-pattern. It couples everything, blocks reuse, and makes concurrency and testing hard. Distribute behaviour to the owner of the data: spot decides fit, floor searches, ticket times, strategy prices, lot orchestrates.

### Q5: "How would you add EV charging spots without breaking existing code?"

**Hint:** Add `SpotType.ELECTRIC` + an `ElectricSpot` subclass; charging cost is a `FeeStrategy` extension. `ParkingLot` and `ParkingFloor` only call `canFit`/`findAvailableSpot`, so they need no changes — the Open/Closed Principle in action.

---

## Practice exercise

**Extend the lot with reserved handicapped spots and a valet.** Starting from the code in Part 5 (copy it into `parking-lot.js`):

1. Add `SpotType.HANDICAPPED` and a `HandicappedSpot` subclass. Give `Vehicle` an optional `hasPermit` flag; make `HandicappedSpot.canFit` return `true` only when `vehicle.hasPermit` is set (in addition to the size check).
2. Add a `NearestGateStrategy` for spot selection: pass a `gateFloor` number into `parkVehicle` and prefer spots on that floor. Refactor `findAvailableSpot` so the selection rule is injectable (a second Strategy).
3. Add a `TieredRateStrategy` that charges 1.5× between 6pm and midnight, 1× otherwise, based on `ticket.entryTime`.
4. In `main()`, park a permit vehicle into a handicapped spot, park a non-permit vehicle and show it is refused that spot, and unpark one during "peak" hours to show the higher fee.

Produce the updated file and its console output. You should find you edited *no existing method bodies of `ParkingLot`* except to accept the injected selection strategy — that is the sign your class boundaries were right.

---

## Quick reference cheat sheet

- **7-step LLD method:** requirements → nouns (objects) → verbs (behaviours) → class diagram → code → patterns → extensions.
- **Nouns become classes; fixed label sets become enums** (`VehicleType`, `SpotType` via `Object.freeze`).
- **Assign each verb to the object that owns its data** — spot decides fit, floor searches, ticket times, strategy prices, lot orchestrates.
- **Anemic model / god class = the trap.** Never dump every method into one `ParkingManager`.
- **Composition (◆):** lot owns floors owns spots — shared lifetime. **Aggregation (◇):** strategy is injected — independent life.
- **`canFit` = spot is free AND spot size ≥ vehicle's required size.** Small vehicles may take bigger spots; big vehicles may not shrink.
- **Smallest-fit search** stops small vehicles hogging large spots; spot-selection is itself a Strategy.
- **Strategy pattern** makes pricing pluggable — new rule = new class, zero edits to the lot.
- **Factory** centralizes "create the right object for this type"; keeps `switch`-on-type out of business code.
- **Prefer dependency injection over Singleton** for the lot — testable, explicit, no global state.
- **Duration billing** rounds up per started hour: `Math.ceil(ms / 3_600_000)`.
- **Concurrency:** make find-and-assign atomic (mutex or DB transaction) so two gates cannot double-book one spot.
- **Extensions are the exam.** EV spots, passes, tiers, thread-safety should each touch one or two classes — if they touch everything, your boundaries were wrong.

---

## Connected topics

| Direction | Topic | Why |
|---|---|---|
| **Previous** | [111 — LLD Approach Framework](./111-lld-approach-framework.md) | The 7-step method this doc applies end to end |
| **Next** | [113 — Library Management System](./113-lld-library-management.md) | The next LLD case study, reusing the same method |
| **Related** | [42 — Strategy Pattern](./42-pattern-strategy.md) | The pattern behind our pluggable `FeeStrategy` |
| **Related** | [30 — Factory Method](./30-pattern-factory-method.md) | Creating vehicles/spots by type without `switch` everywhere |
| **Related** | [23 — Class Relationships](./23-class-relationships.md) | Composition vs. aggregation vs. association used in the class diagram |
