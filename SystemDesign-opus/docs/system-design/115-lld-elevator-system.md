# 115 — Design an Elevator System
## Category: LLD Case Study

---

## What is this?

An elevator system is the software brain of a building's lifts: it takes **requests** (someone on floor 7 presses UP, or a rider inside presses "12") and decides **which elevator serves which request, in what order**. The goal is to move people quickly without wasting travel.

Real-world analogy: think of a **pizza delivery driver** with a car full of orders. New orders keep coming in. The driver doesn't drive back to the shop after every drop — they plan a smart route that hits nearby stops on the way. An elevator does exactly this, but on a vertical line of floors.

---

## Why does it matter?

Elevator design is a **classic LLD interview question** because it forces you to combine three hard ideas at once:

- **State** — an elevator is MOVING, STOPPED, or has its DOORS_OPEN, and it can only do certain things in each state.
- **Strategy** — the "which elevator, and in what order do we visit floors" decision is a swappable algorithm. This is the intellectual core.
- **Concurrency** — real elevators move in parallel and requests arrive at the same time from many floors.

**Interview angle:** interviewers watch whether you cleanly separate the *dispatcher* (who picks an elevator) from the *elevator* (which manages its own motion). Mixing those is the most common failure.

**Real-work angle:** the same shape shows up everywhere — assigning ride requests to drivers (Uber), tasks to workers in a queue, print jobs to printers, or I/O requests to a disk head. The SCAN algorithm you'll use here literally comes from **disk scheduling**.

---

## The core idea — explained simply

### The analogy: a shuttle bus on a one-lane hill

Picture a shuttle bus on a **single vertical road** up a hill. It can only go up or down — no shortcuts. People wait at stops (floors) and hold a sign saying which way they want to go (UP or DOWN). People already on the bus shout out their stop.

The dumb driver serves requests in the exact order they arrived: drive to the first caller, then all the way back for the second, then up again for the third. Exhausting and slow.

The smart driver follows one rule: **keep going the same direction, picking up everyone who wants to go that way, until nobody ahead needs you — then reverse.** One smooth sweep up, one smooth sweep down. This is the "elevator algorithm."

Now add a second bus. A **dispatcher** stands at the bottom with a radio. When a new caller appears, the dispatcher picks whichever bus can reach them soonest.

| Analogy piece | Technical concept |
|---|---|
| The vertical road | The shaft; floors are discrete positions |
| A bus | An `Elevator` / `ElevatorCar` |
| Person at a stop with an UP/DOWN sign | **External request** (`requestElevator(floor, direction)`) |
| Rider shouting their stop | **Internal request** (`selectFloor(elevatorId, floor)`) |
| Driver's "keep going, then reverse" rule | The **SCAN / LOOK scheduling strategy** (per-elevator stop ordering) |
| Dispatcher with the radio | The `ElevatorController` + its **dispatch strategy** (which elevator) |
| Bus is driving / parked / doors open | `ElevatorState` MOVING / STOPPED / DOORS_OPEN |

The two decisions are **separate**: (1) *which* elevator answers a hall call — the dispatcher's job; (2) *what order* one elevator visits its own floors — the elevator's job, guided by a strategy. Keep them apart and the design stays clean.

---

## Key concepts inside this topic

We follow the standard **7-step LLD method**: requirements → nouns → behaviours → class diagram → the hard algorithm → code → patterns.

### 1. Requirements & clarifying questions

**Functional requirements**
- A building has **multiple elevators** and **N floors**.
- **External (hall) requests:** a person on a floor presses UP or DOWN. The *system* must dispatch an elevator to that floor.
- **Internal (car) requests:** a rider inside an elevator presses a destination floor. That request belongs to *that specific* elevator.
- The system dispatches an elevator **efficiently** — minimize waiting/travel.
- Elevators move UP / DOWN / stay IDLE, and OPEN / CLOSE doors at stops.

**Non-functional requirements**
- Low average wait time; fair (no request starves forever).
- Extensible scheduling — we should be able to change the dispatch policy without rewriting elevators.

**Clarifying questions to ask the interviewer** (asking these earns points):
1. How many elevators and floors? (Changes whether a simple scan is enough.)
2. Optimize for **wait time** or **energy**? (Different strategy.)
3. Are there **express elevators** (skip some floors, e.g. a sky-lobby)?
4. **Capacity limits** — should a full elevator skip further hall calls?
5. Any **priority** requests — fire service, freight, VIP?
6. Single building or campus? (We'll assume one building.)

We'll assume: 1 building, 2–3 elevators, ~15 floors, optimize for **wait time**, and make the policy swappable so the other goals are one-line changes later.

### 2. Identify the core objects (nouns)

Read the requirements and underline the nouns:

| Noun | Role |
|---|---|
| `Building` | Owns the floors and the controller (glossed — thin wrapper). |
| `ElevatorController` | The **dispatcher**: holds all elevators + a scheduling strategy, routes requests. |
| `Elevator` (`ElevatorCar`) | One physical car: knows its floor, direction, state, and its own target floors. |
| `Request` | A need to be served — either **external** (origin + direction) or **internal** (destination). |
| `Button` / `Panel` | UI that creates requests. We **gloss** these — they just call controller methods. |
| `Direction` (enum) | UP / DOWN / IDLE. |
| `ElevatorState` (enum) | MOVING / STOPPED / DOORS_OPEN. |
| `RequestType` (enum) | EXTERNAL / INTERNAL. |
| `SchedulingStrategy` | Pluggable algorithm that picks the elevator for a hall call. |

### 3. Behaviours (verbs) + who owns them

The single most important design decision here is **ownership**. Write the verbs and assign each to exactly one class.

| Behaviour | Owner | Why |
|---|---|---|
| `requestElevator(floor, dir)` | `ElevatorController` | A hall call has no elevator yet — the controller must **choose** one. |
| `selectFloor(elevatorId, floor)` | `ElevatorController` → routes to that `Elevator` | The car is already known; it just adds an internal target. |
| **dispatch / assignElevator** | `ElevatorController` via `SchedulingStrategy` | Deciding *which* car answers is a policy — keep it swappable. |
| `addTarget(floor)` | `Elevator` | Only the car knows its own sorted stop list. |
| `move()` / `step()` | `Elevator` | The car owns its **motion** and current floor. |
| `openDoors()` / `closeDoors()` | `Elevator` | Part of the car's own state machine. |

The clean split (recall **coupling & cohesion** from LLD fundamentals):

- The **Controller owns DISPATCHING** — the cross-elevator decision. It does *not* know how a car sequences its floors.
- Each **Elevator owns its own MOVEMENT** and its sorted set of target floors. It does *not* know other elevators exist.

This is **high cohesion** (each class does one job well) and **low coupling** (the controller talks to elevators through a tiny interface: `addTarget`, `step`, a few getters).

### 4. Class diagram (ASCII)

```
                 ┌────────────────────┐
                 │      Building      │
                 │  floors: int       │
                 │  controller ───────┼──┐
                 └────────────────────┘  │  owns
                                         ▼
        ┌──────────────────────────────────────────────┐
        │            ElevatorController                 │
        │  elevators: Elevator[]  ◇────────────┐        │
        │  strategy: SchedulingStrategy        │        │
        │  requestElevator(floor, dir)         │        │
        │  selectFloor(id, floor)              │        │
        │  step()   // advance all cars        │        │
        └───────────────┬──────────────────────┼────────┘
                        │ uses                  │ 1..*
                        ▼                       ▼
     ┌─────────────────────────┐    ┌──────────────────────────┐
     │   SchedulingStrategy    │    │        Elevator          │
     │  (abstract)             │    │  id, currentFloor        │
     │  selectElevator(        │    │  direction: Direction    │
     │      request, elevators)│    │  state: ElevatorState    │
     └──────────┬──────────────┘    │  upSet:  SortedSet<int>  │
                │ extends           │  downSet:SortedSet<int>  │
      ┌─────────┴──────────┐        │  addTarget(floor, dir)   │
      ▼                    ▼        │  step()                  │
┌──────────────┐  ┌──────────────┐  │  isIdle()                │
│ LookStrategy │  │  Nearest     │  └──────────────────────────┘
│              │  │  Elevator    │
└──────────────┘  │  Strategy    │        ┌───────────────────┐
                  └──────────────┘        │     Request       │
                                          │  type: EXTERNAL/  │
   ◇── = aggregation (controller          │        INTERNAL   │
        holds elevators)                  │  floor, direction │
                                          └───────────────────┘
```

The controller **aggregates** elevators (◇) and **uses** a strategy (composition of behaviour). A `Request` is a plain value object passed to the strategy.

### 5. The scheduling problem — the intellectual core (use STRATEGY)

This is what the interviewer really wants to see. "Which elevator, and in what floor order?" is an **algorithm**, and different algorithms trade off differently. So we make it a **Strategy** (recall **42 — Strategy Pattern**: encapsulate a family of interchangeable algorithms behind one interface, so the choice is swappable at runtime).

There are two sub-decisions:

**(a) Dispatch policy — which elevator answers a hall call?** This is `SchedulingStrategy.selectElevator(request, elevators)`.

**(b) Stop ordering — in what order does one car visit its floors?** This lives inside the elevator and is where SCAN/LOOK apply.

Let's compare the classic algorithms:

| Algorithm | Rule | Result |
|---|---|---|
| **FCFS** (First-Come-First-Served) | Serve requests strictly in arrival order. | Simple but **terrible** — the car bounces up and down, huge wasted travel, poor average wait. |
| **SCAN** (the "elevator algorithm") | Keep moving in the current direction, serving every request that way, until you hit the **physical end** of the shaft; then reverse and sweep back. | Smooth, few direction changes. Comes straight from **disk-head scheduling**. Slight waste going all the way to the top/bottom even if nobody's there. |
| **LOOK** | Like SCAN, but reverse **as soon as there are no more requests ahead** — don't drive to the physical end for nothing. | The practical winner: all of SCAN's smoothness, none of the pointless end-travel. |

**Why LOOK/SCAN beats FCFS — a worked example.** Car idle at floor 5. Requests arrive: floor 8 (UP), floor 2 (DOWN), floor 10 (UP), floor 3 (UP).

- **FCFS** visits in arrival order: 5→8 (3), 8→2 (6), 2→10 (8), 10→3 (7) = **24 floors** of travel and 3 direction changes.
- **LOOK** sweeps up first (5→8→10), then down (10→3→2): 5→10 (5) then 10→2 (8) = **13 floors**, 1 direction change.

Nearly half the travel, and far less door-reversing wear. That is *why* every real elevator uses a LOOK-style sweep, and why we must **not** hard-code FCFS.

**The dispatch strategy** (`NearestElevatorStrategy`) picks the elevator whose expected travel to the request is smallest — favouring cars already moving toward the caller in the same direction, and idle cars over busy ones. Because it's a Strategy, we can later drop in an `EnergyEfficientStrategy` without touching the elevators or controller.

> **We are using Strategy** so the dispatch/ordering policy is a plug-in. "Optimize for energy instead of wait time" then becomes *"inject a different strategy object"* — the payoff you'll show in Extensions.

### 6. Full JavaScript implementation

Real, complete, runnable. Save as `elevator.js` and run `node elevator.js`.

```javascript
'use strict';

// ---------- Enums (frozen so they can't be mutated) ----------
const Direction = Object.freeze({ UP: 'UP', DOWN: 'DOWN', IDLE: 'IDLE' });
const ElevatorState = Object.freeze({
  MOVING: 'MOVING',
  STOPPED: 'STOPPED',
  DOORS_OPEN: 'DOORS_OPEN',
});
const RequestType = Object.freeze({ EXTERNAL: 'EXTERNAL', INTERNAL: 'INTERNAL' });

// ---------- Request: a value object describing a need ----------
class Request {
  // External: someone on `floor` pressed `direction`.
  // Internal: a rider chose destination `floor` (direction inferred later).
  constructor(type, floor, direction = Direction.IDLE) {
    this.type = type;
    this.floor = floor;
    this.direction = direction;
  }
  static external(floor, direction) {
    return new Request(RequestType.EXTERNAL, floor, direction);
  }
  static internal(floor) {
    return new Request(RequestType.INTERNAL, floor);
  }
}

// ---------- Elevator: owns its motion + its own sorted target floors ----------
class Elevator {
  constructor(id, startFloor = 0) {
    this.id = id;
    this.currentFloor = startFloor;
    this.direction = Direction.IDLE;
    this.state = ElevatorState.STOPPED;
    // Two sorted sets implement LOOK: serve everything UP, then everything DOWN.
    // Using Sets to auto-dedupe; we sort on demand (small N per car).
    this.upSet = new Set();     // target floors above/at us we serve going up
    this.downSet = new Set();   // target floors below us we serve going down
  }

  isIdle() {
    return this.upSet.size === 0 && this.downSet.size === 0;
  }

  // Add a target floor. `dir` is the caller's intended travel direction
  // (for internal requests we compute it from currentFloor).
  addTarget(floor, dir = Direction.IDLE) {
    if (floor === this.currentFloor && this.state !== ElevatorState.MOVING) {
      // Already here and not mid-move — just open doors next tick.
      this._enqueue(floor, dir);
      return;
    }
    this._enqueue(floor, dir);
    // If we were idle, pick an initial direction toward the new work.
    if (this.direction === Direction.IDLE) {
      this.direction = floor > this.currentFloor ? Direction.UP
                     : floor < this.currentFloor ? Direction.DOWN
                     : (dir === Direction.IDLE ? Direction.UP : dir);
    }
  }

  _enqueue(floor, dir) {
    // Decide which sweep this floor belongs to.
    if (floor > this.currentFloor) this.upSet.add(floor);
    else if (floor < this.currentFloor) this.downSet.add(floor);
    else {
      // Same floor: route by requested direction so we visit it on the right sweep.
      if (dir === Direction.DOWN) this.downSet.add(floor);
      else this.upSet.add(floor);
    }
  }

  // Cost estimate used by the dispatcher: how far to reach `floor`.
  // Idle cars = plain distance. Moving-toward = distance. Moving-away = penalty.
  estimateCost(floor, dir) {
    const dist = Math.abs(floor - this.currentFloor);
    if (this.direction === Direction.IDLE) return dist;
    const goingToward =
      (this.direction === Direction.UP && floor >= this.currentFloor) ||
      (this.direction === Direction.DOWN && floor <= this.currentFloor);
    // Same-direction & ahead is cheap; otherwise add a penalty for the U-turn.
    if (goingToward && (dir === this.direction || dir === Direction.IDLE)) return dist;
    return dist + this._pendingSpan() + 2; // rough "finish current sweep first" penalty
  }

  _pendingSpan() {
    const all = [...this.upSet, ...this.downSet];
    if (all.length === 0) return 0;
    return Math.max(...all) - Math.min(...all);
  }

  // The LOOK step: pick the next target in the current direction; if none,
  // reverse; move one floor; open doors on arrival.
  step() {
    // If doors were open last tick, close them and resume moving.
    if (this.state === ElevatorState.DOORS_OPEN) {
      this.state = ElevatorState.STOPPED;
    }
    if (this.isIdle()) {
      this.direction = Direction.IDLE;
      this.state = ElevatorState.STOPPED;
      return;
    }

    // Choose the active sweep set. LOOK: keep current direction if it has work
    // ahead; otherwise flip.
    let target = this._nextTargetInDirection();
    if (target === null) {
      // No more work this way — reverse (this is the LOOK "turn early" rule).
      this.direction = this.direction === Direction.UP ? Direction.DOWN : Direction.UP;
      target = this._nextTargetInDirection();
    }
    if (target === null) { // safety
      this.direction = Direction.IDLE;
      this.state = ElevatorState.STOPPED;
      return;
    }

    if (target === this.currentFloor) {
      this._arrive(target);
      return;
    }

    // Move exactly one floor toward the target.
    this.state = ElevatorState.MOVING;
    this.currentFloor += this.direction === Direction.UP ? 1 : -1;

    if (this.currentFloor === target || this._servesHere()) {
      this._arrive(this.currentFloor);
    }
  }

  _servesHere() {
    // Do we have a target exactly at the floor we just reached, in our set?
    const set = this.direction === Direction.UP ? this.upSet : this.downSet;
    return set.has(this.currentFloor);
  }

  _nextTargetInDirection() {
    const set = this.direction === Direction.UP ? this.upSet : this.downSet;
    const floors = [...set];
    if (floors.length === 0) return null;
    if (this.direction === Direction.UP) {
      const ahead = floors.filter((f) => f >= this.currentFloor).sort((a, b) => a - b);
      return ahead.length ? ahead[0] : null;
    } else {
      const ahead = floors.filter((f) => f <= this.currentFloor).sort((a, b) => b - a);
      return ahead.length ? ahead[0] : null;
    }
  }

  _arrive(floor) {
    this.upSet.delete(floor);
    this.downSet.delete(floor);
    this.state = ElevatorState.DOORS_OPEN; // "open doors" happens here
  }

  describe() {
    const targets = [...new Set([...this.upSet, ...this.downSet])].sort((a, b) => a - b);
    return `E${this.id} @${this.currentFloor} ${this.direction.padEnd(4)} ` +
           `${this.state.padEnd(10)} targets=[${targets.join(',')}]`;
  }
}

// ---------- Scheduling strategies (the swappable policy) ----------
class SchedulingStrategy {
  // Returns the Elevator that should serve this external request.
  selectElevator(request, elevators) {
    throw new Error('selectElevator must be implemented by a subclass');
  }
}

// Pick the elevator with the lowest expected travel cost (best wait time).
class NearestElevatorStrategy extends SchedulingStrategy {
  selectElevator(request, elevators) {
    let best = null;
    let bestCost = Infinity;
    for (const e of elevators) {
      // Prefer idle cars slightly and use the elevator's own cost model.
      const cost = e.estimateCost(request.floor, request.direction) +
                   (e.isIdle() ? 0 : 0.5);
      if (cost < bestCost) { bestCost = cost; best = e; }
    }
    return best;
  }
}

// Alternative policy to demonstrate swappability: minimize direction changes /
// energy by strongly preferring a car already moving the same way, else the
// most idle one. (Simplified illustrative version.)
class EnergyEfficientStrategy extends SchedulingStrategy {
  selectElevator(request, elevators) {
    const sameDir = elevators.filter(
      (e) => e.direction === request.direction &&
             e.estimateCost(request.floor, request.direction) < Infinity);
    const pool = sameDir.length ? sameDir : elevators.filter((e) => e.isIdle());
    const candidates = pool.length ? pool : elevators;
    return candidates.reduce((a, b) =>
      a.estimateCost(request.floor, request.direction) <=
      b.estimateCost(request.floor, request.direction) ? a : b);
  }
}

// ---------- Controller: owns dispatching, delegates motion to elevators ----------
class ElevatorController {
  constructor(numElevators, numFloors, strategy = new NearestElevatorStrategy()) {
    this.numFloors = numFloors;
    this.strategy = strategy;
    this.elevators = Array.from({ length: numElevators }, (_, i) => new Elevator(i));
    this.tick = 0;
  }

  // Swap the dispatch policy at runtime — the Strategy payoff.
  setStrategy(strategy) { this.strategy = strategy; }

  // EXTERNAL request: choose an elevator, then hand it the stop.
  requestElevator(floor, direction) {
    const req = Request.external(floor, direction);
    const chosen = this.strategy.selectElevator(req, this.elevators);
    chosen.addTarget(floor, direction);
    console.log(`  [dispatch] hall call floor ${floor} ${direction} -> E${chosen.id}`);
    return chosen.id;
  }

  // INTERNAL request: the car is already known; just add its destination.
  selectFloor(elevatorId, floor) {
    const e = this.elevators[elevatorId];
    const dir = floor > e.currentFloor ? Direction.UP : Direction.DOWN;
    e.addTarget(floor, dir);
    console.log(`  [rider]    inside E${elevatorId} pressed floor ${floor}`);
  }

  // Advance the whole simulation one tick: every elevator steps once.
  // (In reality these run in parallel — see the concurrency note.)
  step() {
    this.tick++;
    for (const e of this.elevators) e.step();
  }

  allIdle() { return this.elevators.every((e) => e.isIdle()); }

  print() {
    console.log(`t=${String(this.tick).padStart(2)} | ` +
      this.elevators.map((e) => e.describe()).join('  ||  '));
  }
}

// ---------- Demo simulation ----------
function main() {
  console.log('=== Elevator simulation (LOOK ordering, Nearest dispatch) ===\n');
  const ctrl = new ElevatorController(2, 15); // 2 elevators, floors 0..14

  // Fire a burst of external + internal requests.
  ctrl.requestElevator(8, Direction.UP);   // hall call
  ctrl.requestElevator(2, Direction.DOWN); // hall call
  ctrl.requestElevator(10, Direction.UP);  // hall call
  console.log('');

  ctrl.print();
  for (let i = 0; i < 6; i++) {
    ctrl.step();
    ctrl.print();
    // Mid-run, a rider inside E0 presses 12, and a new hall call arrives.
    if (i === 1) {
      console.log('  -- new events at this tick --');
      ctrl.selectFloor(0, 12);
      ctrl.requestElevator(3, Direction.UP);
    }
  }

  // Run to completion.
  console.log('\n...draining remaining requests...');
  let guard = 0;
  while (!ctrl.allIdle() && guard++ < 40) {
    ctrl.step();
    ctrl.print();
  }

  console.log('\n=== Now swap the Strategy to EnergyEfficient — same API ===\n');
  ctrl.setStrategy(new EnergyEfficientStrategy());
  ctrl.requestElevator(6, Direction.UP);
  ctrl.requestElevator(1, Direction.DOWN);
  ctrl.print();
  guard = 0;
  while (!ctrl.allIdle() && guard++ < 40) { ctrl.step(); ctrl.print(); }
  console.log('\nDone.');
}

main();

module.exports = {
  Direction, ElevatorState, RequestType, Request,
  Elevator, SchedulingStrategy, NearestElevatorStrategy,
  EnergyEfficientStrategy, ElevatorController,
};
```

Running it prints each elevator's floor/direction/state every tick, showing cars sweeping up then down (LOOK), picking up mid-sweep floors like 3 and 12, and the dispatcher assigning hall calls to the nearest car — proving the scheduling works and it **runs**.

### 7. Design patterns used and WHY

| Pattern | Where | Why |
|---|---|---|
| **Strategy** (the star — recall **42**) | `SchedulingStrategy` + `NearestElevatorStrategy` / `EnergyEfficientStrategy`, injected into the controller | Dispatch/ordering is a *family of interchangeable algorithms*. Encapsulating each behind one interface lets us swap policy at runtime without touching `Elevator` or `ElevatorController`. |
| **State** (recall **46**) | `ElevatorState` MOVING / STOPPED / DOORS_OPEN drives what `step()` does | An elevator behaves differently in each state (can't move with doors open). Modelled as an explicit enum-driven machine; could grow into full State-pattern classes if transitions get complex. |
| **(implicit) Factory** | `Request.external` / `Request.internal` static builders | Clear construction of the two request flavours. |

**The concurrency angle (recall 52 — concurrency fundamentals).** Our simulation is deliberately **single-threaded**: `controller.step()` advances every elevator in a loop, so there are no data races. But **real** elevators run in parallel and hall calls arrive concurrently from many floors. If this weren't a simulation you'd need synchronization:

- The dispatcher's `selectElevator` + `addTarget` must be **atomic** — otherwise two hall calls could both pick the same "nearest" idle car and one gets starved (a lost-update race). A lock or an actor-style single-writer queue per elevator fixes this.
- Each elevator's `upSet`/`downSet` is **shared mutable state** (rider presses a button while the motion thread reads targets) — guard it with a mutex or make the elevator an actor that processes messages one at a time.
- The cleanest concurrent design makes each `Elevator` an **independent worker** and the controller a **producer** posting requests to per-elevator queues — no shared mutable structure, so no locks.

---

## Visual / Diagram description

Sequence of a single hall call, end to end:

```
Person@F7        Controller            Strategy           Elevator E1
   │                 │                    │                    │
   │ press UP        │                    │                    │
   ├────────────────>│ requestElevator(7,UP)                   │
   │                 │ selectElevator(req,[E0,E1])             │
   │                 ├───────────────────>│                    │
   │                 │   estimateCost(7,UP) on each car        │
   │                 │<───────────────────┤ returns E1 (nearest)
   │                 │ E1.addTarget(7, UP)                     │
   │                 ├────────────────────────────────────────>│
   │                 │                    │        upSet={7}   │
   │                 │                    │                    │
   │   ... each tick: controller.step() -> every elevator.step()
   │                 │                    │   E1: F5→F6→F7      │
   │                 │                    │   arrive: DOORS_OPEN│
   │  rider enters, presses 12            │                    │
   ├────────────────>│ selectFloor(1, 12)                      │
   │                 ├────────────────────────────────────────>│
   │                 │                    │   upSet={12}        │
   │                 │                    │   continue sweep up │
```

**How to read it:** the hall call goes to the **controller**, which asks the **strategy** which car; only then does the chosen **elevator** own the floor and its own motion. The internal "press 12" skips the strategy entirely — the car is already known.

---

## Real world examples

### Otis / KONE (Destination Dispatch)

Modern high-rise elevators from vendors like Otis and KONE use **destination dispatch**: you enter your destination floor at a **lobby kiosk** *before* boarding, and the system **groups** riders going to nearby floors into the same car. This is a richer dispatch **strategy** than "nearest car" — it optimizes for total system throughput, not just one call. (Representative of the published approach; exact algorithms are proprietary.)

### Disk I/O schedulers (the same math)

The Linux kernel historically shipped the **"elevator" I/O scheduler** (and later CFQ, deadline, mq-deadline). The disk head is literally an elevator servicing block requests, and it uses **SCAN/LOOK** to minimize seek time — the identical algorithm you implemented. Elevator LLD and disk scheduling are two faces of the same problem.

### Ride dispatch (Uber / Lyft) — conceptual cousin

Assigning a ride request to the best nearby driver is the same shape as assigning a hall call to the best car: a **dispatcher** scoring candidates by estimated arrival time, behind a swappable policy (nearest, batched, surge-aware). Representative of how matching systems are structured, not a specific published algorithm.

---

## Trade-offs

**Scheduling algorithm**

| Algorithm | Pros | Cons |
|---|---|---|
| FCFS | Trivial, perfectly fair by arrival | Terrible travel/wait; bounces up and down |
| SCAN | Smooth, few reversals, bounded worst case | Wastes travel to the physical shaft ends |
| LOOK | All of SCAN's smoothness, no wasted end-travel | Slightly more bookkeeping (track "any requests ahead?") |

**Dispatch responsibility split**

| Choice | Pros | Cons |
|---|---|---|
| Controller owns dispatch, elevator owns motion (our design) | High cohesion, low coupling, swappable policy | Slightly more classes/indirection |
| Elevator picks its own hall calls | Fewer classes | Cars fight over calls; no global optimization; hard to change policy |

**The sweet spot:** a thin **controller** that dispatches via an **injectable Strategy**, plus **elevators** that each run a **LOOK** sweep over their own sorted target set. You get global dispatch optimization *and* smooth per-car motion, and either half can change independently.

---

## Common interview questions on this topic

### Q1: "Where does the scheduling logic live — in the elevator or the controller?"
**Hint:** Split it. The *controller* decides **which** elevator answers a hall call (dispatch, via a Strategy). Each *elevator* decides **what order** to visit its own floors (LOOK over its sorted target set). Mixing them couples every car to every other car.

### Q2: "How do you handle external vs internal requests differently?"
**Hint:** An **external** (hall) request has no assigned car yet → it must go through the controller's dispatch strategy. An **internal** (car) request already knows its elevator → skip the strategy, just `addTarget` on that car. Both end up as sorted targets.

### Q3: "Why not FCFS? Prove LOOK is better."
**Hint:** FCFS bounces between distant floors, multiplying travel and direction changes. Walk the worked example (5→8→2→10 by FCFS ≈ 24 floors vs LOOK ≈ 13). LOOK minimizes direction changes and total travel by sweeping one way then reversing.

### Q4: "Now optimize for energy instead of wait time. How much do you rewrite?"
**Hint:** Almost nothing — that's the point of Strategy. Inject a different `SchedulingStrategy` (e.g. `EnergyEfficientStrategy` that batches same-direction calls and prefers already-moving cars). `Elevator` and `ElevatorController` are untouched.

### Q5: "How would this work with real concurrent elevators?"
**Hint:** Make each elevator an independent worker with its own request queue; the controller is a producer. Dispatch (`select + addTarget`) must be atomic to avoid two calls grabbing the same car. Prefer per-elevator single-writer queues over shared mutable sets to avoid locks (recall 52).

---

## Practice exercise

### Build and extend the simulator (~30–40 min)

1. Copy the implementation into `elevator.js` and run `node elevator.js`. Confirm each elevator sweeps up then down (LOOK) and hall calls go to the nearest car.
2. **Add a `FCFSStrategy`** that always assigns the first elevator and makes it serve targets in strict arrival order (hint: give the elevator an optional FIFO queue mode). Run the same request burst under FCFS and under `NearestElevatorStrategy`. **Count total floors travelled** for each and confirm FCFS is worse.
3. **Add capacity:** give `Elevator` a `capacity` and a `load`; make the dispatcher **skip full elevators** for new hall calls. Test that a full car is passed over.

Produce: the working file, plus a short note with the two floor-travel totals from step 2 proving LOOK/Nearest wins.

---

## Quick reference cheat sheet

- **Two request types:** **External** (hall call — floor + direction, no car yet) vs **Internal** (car call — destination, car known).
- **Ownership split:** Controller owns **dispatch** (which car); Elevator owns **motion** + its **sorted targets**.
- **Dispatch = Strategy:** `selectElevator(request, elevators)` — swap the policy without touching elevators.
- **LOOK algorithm:** sweep one direction serving all targets ahead; reverse the moment nothing's ahead. Beats SCAN (no end-travel) and crushes FCFS.
- **FCFS is a trap:** simple but bounces between floors — huge wasted travel.
- **SCAN origin:** the disk-head scheduling algorithm; elevator = disk head.
- **State enum:** MOVING / STOPPED / DOORS_OPEN gates what `step()` may do.
- **Direction enum:** UP / DOWN / IDLE; an idle car picks direction toward its first target.
- **Two sorted sets** (`upSet`, `downSet`) per car cleanly implement the LOOK sweep.
- **Strategy payoff:** "optimize for energy" = inject a new strategy, zero rewrite.
- **Concurrency (52):** real cars run in parallel; make dispatch atomic or use per-car worker queues to avoid races.
- **Extensibility:** express floors, capacity limits, and priority all attach as strategy tweaks or elevator fields — the core stays put.

---

## Connected topics

| Direction | Topic | Why |
|---|---|---|
| **Previous** | [114 — ATM Machine](./114-lld-atm-machine.md) | Another LLD case study; ATM leans on State, elevator leans on Strategy — compare how each drives its object model. |
| **Next** | [111 — LLD Approach Framework](./111-lld-approach-framework.md) | Re-read the 7-step method you just applied end to end; use it as your checklist for the next LLD problem. |
| **Related** | [42 — Strategy Pattern](./42-pattern-strategy.md) | The scheduling/dispatch policy is a textbook Strategy — swappable algorithms behind one interface. |
| **Related** | [46 — State Pattern](./46-pattern-state.md) | The elevator's MOVING/STOPPED/DOORS_OPEN machine; grow the enum into State classes when transitions get complex. |
| **Related** | [52 — Concurrency Fundamentals](./52-concurrency-fundamentals.md) | Real elevators run in parallel; understand the races before removing the single-threaded `step()` loop. |
