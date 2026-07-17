# 111 — How to Approach Any LLD Problem — Step by Step
## Category: LLD Case Study

---

## What is this?

This is the **recipe**. Every LLD problem you will ever be handed — parking lot, elevator, chess, Splitwise, vending machine, ride-hailing, library, ATM — is the *same problem wearing a different costume*: turn a vague paragraph of English into a set of classes with clear responsibilities, then make it run.

Think of it like a chef's *mise en place*. A good chef doesn't improvise the order of operations under pressure — they chop, measure, and lay out every ingredient before the pan gets hot, in the same order, every single time. This doc is your mise en place for machine-coding rounds: seven steps, a time budget, and a checklist so that when the timer starts you are *executing a known procedure*, not *inventing one*.

---

## Why does it matter?

Without a procedure, an LLD round goes like this: you hear "design a parking lot," you panic, you start typing a `ParkingLot` class immediately, and 40 minutes later you have 300 lines of a `ParkingLotManager` god class with fourteen `if` statements, no interfaces, and nothing that runs. The interviewer writes *"jumped to code, no design, not extensible"* and you're out.

**In the interview:** LLD (often called the "machine coding round") is the primary design round for SDE2/L4 and a mandatory round for SDE3/L5 at most product companies. It's 45–90 minutes, usually solo with a laptop or live-shared with the interviewer. The problems repeat — there are maybe 20 canonical ones — so the *procedure* generalizes almost perfectly. Learn the procedure once and you've prepared for all of them.

**At work:** This is exactly how you start any new feature or service. Someone says "we need to add promotional pricing." You clarify scope, extract the entities, decide who owns the behaviour, pick relationships, and *only then* touch the keyboard. The engineers who consistently produce code that survives three quarters of requirement changes are the ones doing steps 1–5 in their head before typing.

The bar is not "did you finish." The bar is **"is your object model right, and does it absorb change?"** That's what you're being scored on.

---

## The core idea — explained simply

### The Screenwriter's Analogy

A screenwriter handed a one-line pitch — *"a heist movie set in a bank"* — does not start writing dialogue. They do this, in order:

1. **Who's in the movie?** (the *cast*) — a thief, a security guard, a bank manager, a getaway driver.
2. **What can each person DO?** (their *actions*) — the thief cracks safes, the guard patrols, the driver drives.
3. **How do they relate?** — the driver *works for* the thief; the guard *is a kind of* bank employee; the bank *contains* vaults.
4. **What are the beats?** (the *plot*) — casing, break-in, alarm, chase.
5. Only then: **write the scenes** (the *code*).

If the writer skips to dialogue, they end up with a script where the thief inexplicably knows the vault combination and the guard has no motivation. It "works" line by line but falls apart as a whole.

LLD is the same job with different words:

| Screenwriting | LLD | What you produce |
|---|---|---|
| The pitch ("a heist in a bank") | The problem statement ("design a parking lot") | Deliberately vague. Ask questions. |
| The cast | The **nouns** → classes/entities | `Vehicle`, `ParkingSpot`, `Ticket` |
| What each character can do | The **verbs** → methods, and *who owns them* | `spot.assign(vehicle)`, `ticket.close()` |
| Relationships between characters | is-a / has-a / uses-a | `Car extends Vehicle`, `Floor` *has* `Spot[]` |
| Recurring narrative devices | **Design patterns** | Strategy for pricing, State for lifecycle |
| The scenes | The **code** | Runnable JS with a `main()` demo |
| The sequel | **Extensions** ("now add EV charging") | Your design absorbs it, or it doesn't |

The whole art is in steps 2 and 3: **find the right nouns, and give each noun the behaviour that belongs to its data.** Everything else is bookkeeping.

---

## Key concepts inside this topic

Here is the full time budget for a 45–60 minute round. Print it. Internalize it.

```
┌──────────────────────────────────────────────────────────┐
│  THE 7-STEP LLD BUDGET  (45–60 min machine coding)       │
├──────────────────────────────────────────────────────────┤
│  Step 1  Clarify requirements & scope .............  5m   │
│  Step 2  Nouns   → core objects ..................  5m   │
│  Step 3  Verbs   → behaviours & ownership ........  5m   │
│  Step 4  Relationships + class diagram ...........  5m   │
│  Step 5  Patterns (only where earned) ............  5m   │
│  ─────────────────────────────────────────────────────── │
│  Step 6  WRITE THE CODE ..................... 20–25m     │
│  ─────────────────────────────────────────────────────── │
│  Step 7  Concurrency, edge cases, extensions .....  5m   │
└──────────────────────────────────────────────────────────┘
   Design ≈ 25 min. Code ≈ 25 min. Never invert this.
```

---

### 1. Step 1 — Clarify requirements and scope (5 min)

**An LLD problem is ALWAYS under-specified on purpose.** "Design a parking lot" is four words. A real parking lot has motorcycles, valet, monthly passes, EV chargers, license-plate cameras, and a payment gateway. The interviewer has *deliberately* withheld the scope because they want to see whether you **ask** or **assume**. Candidates who assume build the wrong system beautifully.

Run this script. Out loud. Every time.

**The LLD question script:**

1. **Who are the actors?** Who touches this system? (Customer, admin, attendant, a background job, another service.)
2. **What are the core use cases?** Make them say 5–8. Force them to be **verbs**: *park a vehicle*, *issue a ticket*, *calculate a fee*, *pay*, *exit*.
3. **What is explicitly OUT of scope?** This is the most valuable question in the round. "Do I need authentication? Payment gateway integration? Persistence to a real DB? A REST API?" Almost always the answer is *no* — and you just saved yourself 20 minutes.
4. **Scale / constraints?** How many floors, spots, users? (Usually irrelevant to LLD, but it tells you whether they want an index/`Map` or a linear scan.)
5. **Concurrency?** "Can two cars try to take the last spot at the same time?" If yes, that changes your design. Ask now, not at minute 40.
6. **Is there a specific algorithm they care about?** "Nearest spot to the entrance?" "Cheapest slot?" If yes, that's a **Strategy** and they're telling you so.

**Then write the agreed use cases as a numbered list at the top of your file, as a comment.** This is not decoration. It is your **test checklist**: at the end, your `main()` demo must exercise every line of it, and you will literally point at the list and say "1 through 7, all demonstrated."

```js
/**
 * AGREED SCOPE — Parking Lot  (confirmed with interviewer)
 *
 * IN SCOPE
 *  1. Park a vehicle (car / bike / truck) at an available spot of a compatible size
 *  2. Issue a ticket with entry timestamp
 *  3. Find the nearest available spot to an entrance
 *  4. Un-park a vehicle and free the spot
 *  5. Calculate the fee (hourly, varies by vehicle type)
 *  6. Take payment (cash or card — simulated, no real gateway)
 *  7. Show free-spot counts per floor on a display board
 *
 * OUT OF SCOPE (confirmed)
 *  - Auth / user accounts, real DB persistence, HTTP API, monthly passes, valet
 *
 * CONSTRAINTS
 *  - Single process, in-memory. Multiple entrances CAN book concurrently → guard the last spot.
 *  - Fee strategy must be swappable (they hinted at weekend/flat pricing later).
 */
```

That comment block alone earns you points before you write a single class.

---

### 2. Step 2 — Identify the core objects (the NOUNS) (5 min)

**Technique:** take the requirements paragraph, underline **every noun**, then **filter aggressively**. Extraction is easy; *culling* is the skill. Most beginners turn every noun into a class and end up with 25 classes, half of which have one field.

**The four buckets every noun falls into:**

| Bucket | Meaning | Example |
|---|---|---|
| **Entity** | Has an identity that persists over time. Two of them with identical fields are still *different things*. | `Ticket`, `Vehicle`, `ParkingSpot` |
| **Value object** | Defined entirely by its values. Immutable. Two with the same values are interchangeable. | `Money`, `TimeRange`, `Coordinates` |
| **Enum** | A closed set of named constants. | `VehicleType`, `SpotSize`, `PaymentStatus` |
| **Not a class** | It's an *attribute* of something else, a duplicate, or out of scope. | "colour", "license plate", "hourly rate" |

**Worked example.** Here is a 3-sentence parking-lot statement:

> *"A parking lot has multiple floors, and each floor has many parking spots. A spot can fit a motorcycle, a car, or a truck depending on its size; the colour of the vehicle doesn't matter. When a customer drives their car up to the entrance gate, an attendant issues a ticket showing the entry time, and on exit the customer pays a fee at the exit gate by cash or card."*

**Raw noun extraction (everything, no judgement):**

```
parking lot, floor, parking spot, motorcycle, car, truck, size, colour,
vehicle, customer, entrance gate, attendant, ticket, entry time, exit,
fee, exit gate, cash, card
```

**Now cull. Say this reasoning OUT LOUD in the interview — this is the part they score:**

| Noun | Verdict | Why |
|---|---|---|
| parking lot | **Entity** | The aggregate root. Owns floors. |
| floor | **Entity** | Owns spots, has a number. |
| parking spot | **Entity** | Has identity + occupancy state. |
| car / motorcycle / truck | **Subclasses of `Vehicle`** *or just an enum* | Do they *behave* differently? If they only differ by size, an enum is enough. Prefer the enum until behaviour diverges. |
| vehicle | **Entity** | Has a plate (identity). |
| size | **Enum** (`SpotSize`) | Closed set. Not a class. |
| **colour** | **CUT** — it's an attribute, and the statement literally says it doesn't matter. | Classic trap noun. |
| customer | **CUT (for now)** | Nothing in scope needs a customer identity. If we add memberships later, promote it. *Say this — it shows judgement, not laziness.* |
| entrance gate / exit gate | **Entity** (`Gate`, with a `GateType` enum) | They have identity (gate #3) and are where tickets are issued. |
| attendant | **CUT** | An actor, not a modelled object. Nothing stores state about them. |
| ticket | **Entity** | The heart of the system. |
| entry time | **Attribute of `Ticket`** | Not a class. |
| fee | **Value object** (`Money`) | Amount + currency, immutable. |
| cash / card | **Enum** (`PaymentMode`) + later a `PaymentStrategy` | |

**From 19 nouns → 6 entities, 1 value object, 3 enums.** That's a good ratio. If you end up with 20 classes, you over-modelled.

**Enums in JavaScript.** JS has no `enum` keyword. Use a frozen object — and *say* why you froze it (accidental mutation of a shared constant is a real bug):

```js
// Object.freeze prevents anyone doing SpotSize.LARGE = 'huge' at runtime.
export const SpotSize = Object.freeze({
  SMALL:  'SMALL',   // motorcycle
  MEDIUM: 'MEDIUM',  // car
  LARGE:  'LARGE',   // truck
});

export const VehicleType = Object.freeze({
  MOTORCYCLE: 'MOTORCYCLE',
  CAR:        'CAR',
  TRUCK:      'TRUCK',
});

export const TicketStatus = Object.freeze({
  ACTIVE: 'ACTIVE',
  PAID:   'PAID',
  CLOSED: 'CLOSED',
});
```

**Value objects: immutable, compared by value.**

```js
export class Money {
  #amount;               // integer minor units (paise/cents) — never float money
  #currency;

  constructor(amount, currency = 'INR') {
    if (!Number.isInteger(amount)) throw new TypeError('Money takes integer minor units');
    this.#amount = amount;
    this.#currency = currency;
    Object.freeze(this);                 // value objects do not mutate
  }

  get amount()   { return this.#amount; }
  get currency() { return this.#currency; }

  // Operations RETURN A NEW Money — they never mutate.
  plus(other) {
    if (other.currency !== this.#currency) throw new Error('Currency mismatch');
    return new Money(this.#amount + other.amount, this.#currency);
  }
  times(n) { return new Money(Math.round(this.#amount * n), this.#currency); }
  equals(other) { return this.#amount === other.amount && this.#currency === other.currency; }
  toString() { return `${(this.#amount / 100).toFixed(2)} ${this.#currency}`; }
}
```

**Naming rules:** classes are singular nouns (`Ticket`, not `Tickets` or `TicketData`). Never `TicketManager`, `TicketHelper`, `TicketUtil`, `TicketInfo` — those names are a confession that you don't know what the class *is*. The one place `Service` is acceptable is a genuine stateless coordinator (`PaymentService`), and even then keep it thin.

---

### 3. Step 3 — Identify the behaviours (the VERBS) and assign ownership (5 min)

Extract the verbs the same way: *park, issue, calculate, pay, exit, free, display, find*.

Now ask the question that **IS** object-oriented design:

> ### **Which class should OWN this behaviour?**

**The rule: behaviour belongs with the data it operates on.** This has a name — **"Tell, Don't Ask."** Don't *ask* an object for its data so you can compute something about it elsewhere; **tell** the object to do the thing, because it already has the data.

**Bad — asking (the data is dragged out and the logic lives somewhere else):**

```js
// ParkingService reaches INTO the ticket and the vehicle, pulls out raw fields,
// and does the thinking itself. The Ticket is a passive data bag.
class ParkingService {
  calculateFee(ticket) {
    const hours = Math.ceil((Date.now() - ticket.entryTime) / 3_600_000);
    if (ticket.vehicle.type === 'CAR')        return hours * 2000;
    if (ticket.vehicle.type === 'MOTORCYCLE') return hours * 1000;
    if (ticket.vehicle.type === 'TRUCK')      return hours * 3500;
  }
}
```

Two things are wrong. First, `Ticket` knows how long it's been open better than anyone — that logic should live on it. Second, that `if/else` chain on type is a **switch on a type code**, which is the loudest possible smell for "you need polymorphism or a Strategy."

**Good — telling:**

```js
class Ticket {
  #entrySpot; #vehicle; #entryTime; #exitTime = null; #status = TicketStatus.ACTIVE;

  constructor(id, vehicle, spot, clock = Date) {
    this.id = id; this.#vehicle = vehicle; this.#entrySpot = spot;
    this.#clock = clock;                       // injected → testable, no real sleeping
    this.#entryTime = clock.now();
  }

  get vehicle() { return this.#vehicle; }
  get status()  { return this.#status; }

  // The Ticket owns its own duration. Nobody else should do this arithmetic.
  durationHours() {
    const end = this.#exitTime ?? this.#clock.now();
    return Math.ceil((end - this.#entryTime) / 3_600_000);
  }

  close(clock = this.#clock) {
    if (this.#status === TicketStatus.CLOSED) throw new Error('Ticket already closed');
    this.#exitTime = clock.now();
    this.#status = TicketStatus.CLOSED;
  }
}

// And the fee itself becomes a Strategy, because it's an interchangeable algorithm.
class HourlyFeeStrategy {
  #ratesByType;
  constructor(ratesByType) { this.#ratesByType = ratesByType; }
  calculate(ticket) {
    const rate = this.#ratesByType[ticket.vehicle.type];
    return rate.times(ticket.durationHours());   // TELL the ticket; don't ask for timestamps
  }
}
```

Notice what happened: `ParkingService.calculateFee` **disappeared**. The knowledge went to the objects that own the data.

#### The anemic domain model — THE most common LLD failure

Name it explicitly in your interview. It buys you enormous credibility.

An **anemic domain model** is when your "classes" are nothing but bags of getters and setters — pure data holders — and *all* the actual logic is marooned in one giant `Service` / `Manager` / `Controller` god class. You have written procedural code, then sprinkled the word `class` on top of it. You get every cost of OOP (indirection, boilerplate) and none of the benefits (encapsulation, polymorphism, extensibility).

**Anemic (what most candidates write under time pressure):**

```js
class Spot { constructor(id, size) { this.id = id; this.size = size; this.vehicle = null; } }
class Vehicle { constructor(plate, type) { this.plate = plate; this.type = type; } }
class Ticket { constructor(id) { this.id = id; this.entry = null; this.exit = null; this.fee = 0; } }

class ParkingLotManager {   // ← 400 lines. Everything lives here. This is the smell.
  park(vehicle) {
    for (const f of this.floors) {
      for (const s of f.spots) {
        if (s.vehicle === null && this.fits(vehicle.type, s.size)) {
          s.vehicle = vehicle;                       // reaching in and mutating another object's field
          const t = new Ticket(this.nextId++);
          t.entry = Date.now();
          t.spotId = s.id;
          this.tickets.set(t.id, t);
          f.free -= 1;                               // and again
          return t;
        }
      }
    }
    throw new Error('Lot full');
  }
  fits(vType, sSize) { /* big if/else */ }
  unpark(ticketId) { /* another 40 lines reaching into everything */ }
  calculateFee(t)  { /* if/else on vehicle type */ }
  pay(t, mode)     { /* if/else on payment mode */ }
}
```

Every field is public. Every invariant ("a spot with a vehicle is occupied") is enforced *by convention* in the manager, and any new code path can violate it. Adding EV charging means editing `ParkingLotManager` in five places.

**Rich (the same system, behaviour pushed onto the data):**

```js
class ParkingSpot {
  #id; #size; #vehicle = null;

  constructor(id, size) { this.#id = id; this.#size = size; }

  get id() { return this.#id; }
  get size() { return this.#size; }
  get isFree() { return this.#vehicle === null; }

  // The spot decides whether it can hold this vehicle. Not a manager.
  canFit(vehicle) { return this.isFree && SPOT_CAPACITY[this.#size].includes(vehicle.type); }

  // The spot enforces its OWN invariant. It is impossible to double-park it,
  // because #vehicle is private and this is the only door in.
  assign(vehicle) {
    if (!this.canFit(vehicle)) throw new Error(`Spot ${this.#id} cannot fit ${vehicle.type}`);
    this.#vehicle = vehicle;
  }

  release() {
    if (this.isFree) throw new Error(`Spot ${this.#id} is already free`);
    const v = this.#vehicle;
    this.#vehicle = null;
    return v;
  }
}

class Floor {
  #number; #spots;
  constructor(number, spots) { this.#number = number; this.#spots = spots; }
  get number() { return this.#number; }
  // The Floor owns the question "where is a spot for this vehicle?"
  findSpotFor(vehicle) { return this.#spots.find(s => s.canFit(vehicle)) ?? null; }
  freeCount(size) { return this.#spots.filter(s => s.isFree && s.size === size).length; }
}
```

Now `ParkingLot.park()` shrinks to ~6 lines of *coordination* — which is all a service should ever be.

| | Anemic | Rich |
|---|---|---|
| Where's the logic? | One god class | Distributed to the owning objects |
| Can you break an invariant from outside? | Yes, trivially (`spot.vehicle = x`) | No — private fields, one door in |
| Add a new vehicle type | Edit 5 `if/else` chains | Add an enum value + a capacity row |
| Unit testable? | Only through the god class | Each class in isolation |
| Interviewer's note | *"procedural code in classes"* | *"good encapsulation"* |

**Ownership heuristics when you're stuck:**
- Does the method only read fields of **one** object? → It belongs **on that object**.
- Does it need fields from **two peers**? → It belongs on the object that **contains** them both (`Floor` coordinates `Spot`s).
- Does it need something **external** (a gateway, a clock, a repository)? → **Service**, with the dependency injected.
- Is it a **plug-in decision** ("how do we price this?") → **Strategy object**, not an `if`.

---

### 4. Step 4 — Define the relationships (5 min)

Recall from [23 — Class Relationships](./23-class-relationships.md): for **every pair** of classes, pick exactly one arrow.

| Relationship | Question to ask | Lifetime | UML | JS |
|---|---|---|---|---|
| **is-a** (inheritance) | "Is a Car *a kind of* Vehicle?" | n/a | `◁──` hollow triangle | `class Car extends Vehicle` |
| **has-a, strong** (composition) | "If the Floor is destroyed, do the Spots stop existing?" **Yes** → composition | Child dies with parent | `◆──` filled diamond | Parent **creates** children in its constructor |
| **has-a, weak** (aggregation) | "The Lot has Vehicles — but a Vehicle exists without the Lot." | Independent | `◇──` hollow diamond | Children **passed in** from outside |
| **uses-a** (association/dependency) | "`ParkingLot` *uses* a `FeeStrategy` to price a ticket." | Transient | `──▶` plain arrow | Passed as a param or held as a field |

**The class diagram (draw this on the board before coding):**

```
                        ┌─────────────────────────────┐
                        │        ParkingLot           │   « aggregate root »
                        ├─────────────────────────────┤
                        │ - floors    : Floor[]       │
                        │ - tickets   : Map<id,Ticket>│
                        │ - feeStrategy : FeeStrategy │◇───────────┐
                        ├─────────────────────────────┤            │ uses-a
                        │ + park(vehicle) : Ticket    │            │
                        │ + unpark(ticketId) : Money  │            ▼
                        └───────────┬─────────────────┘   ┌──────────────────────┐
                              ◆ 1..*│ composition         │  «interface»         │
                                    │ (floors die with    │   FeeStrategy        │
                                    │  the lot)           ├──────────────────────┤
                        ┌───────────▼─────────────┐       │ + calculate(t):Money │
                        │         Floor           │       └──────────▲───────────┘
                        ├─────────────────────────┤                  │ is-a
                        │ - number : number       │        ┌─────────┴─────────┐
                        │ - spots  : ParkingSpot[]│        │                   │
                        ├─────────────────────────┤  ┌─────┴──────┐   ┌────────┴──────┐
                        │ + findSpotFor(v) : Spot │  │HourlyFee   │   │ FlatFee       │
                        │ + freeCount(size): int  │  │Strategy    │   │ Strategy      │
                        └───────────┬─────────────┘  └────────────┘   └───────────────┘
                              ◆ 1..*│ composition
                                    │
                        ┌───────────▼─────────────┐          ┌────────────────────┐
                        │      ParkingSpot        │          │      Ticket        │
                        ├─────────────────────────┤          ├────────────────────┤
                        │ - id      : string      │  0..1    │ - id      : string │
                        │ - size    : SpotSize    │◇─────────│ - vehicle : Vehicle│
                        │ - vehicle : Vehicle|null│  holds   │ - spot    : Spot   │
                        ├─────────────────────────┤          │ - entryTime : ms   │
                        │ + canFit(v)  : boolean  │          ├────────────────────┤
                        │ + assign(v)  : void     │          │ + durationHours()  │
                        │ + release()  : Vehicle  │          │ + close()          │
                        └─────────────────────────┘          └────────────────────┘
                                                                       ◇ 1
                                                                       │ aggregation
                                                             ┌─────────▼──────────┐
                                                             │      Vehicle       │
                                                             ├────────────────────┤
                                                             │ - plate : string   │
                                                             │ - type  : VehicleType
                                                             └────────────────────┘
```

**Narrate the principles as you draw.** This is free credit:

- **SRP (Single Responsibility):** "`Ticket` tracks time and status. It does *not* know how to price itself — that's `FeeStrategy`. One reason to change each."
- **OCP (Open/Closed):** "To add weekend pricing I add a new `FeeStrategy` class. I don't modify `ParkingLot`. Open for extension, closed for modification."
- **Composition over inheritance:** "I did *not* make `CarSpot extends ParkingSpot`. Size is data, not a type. Inheritance for a data difference gives you a class explosion."
- **Program to an interface:** "`ParkingLot` holds a `FeeStrategy`, not an `HourlyFeeStrategy`. It doesn't know or care which."

---

### 5. Step 5 — Apply patterns ONLY where they earn their place (5 min)

Recall from [28 — What Are Design Patterns](./28-what-are-design-patterns.md) that a pattern is a *named solution to a recurring problem*. The mistake is treating them as a shopping list. **You do not "add patterns." You notice a smell, and the pattern is the cure.**

**The LLD pattern cheat-sheet — this problem-smell → pattern mapping covers ~90% of LLD interviews:**

| The smell in the requirements | Pattern | Canonical LLD problems |
|---|---|---|
| Object has **modes / a lifecycle**, and behaviour differs per mode | **State** | ATM, vending machine, order, elevator, traffic light |
| **Interchangeable algorithms** for the same job | **Strategy** | Parking fee, pricing, matching a rider, sorting, Splitwise split types |
| "**X must be notified when Y changes**" | **Observer** | Notifications, display boards, stock tickers, auction bids |
| **Complex creation**, or many variants chosen at runtime | **Factory** | `VehicleFactory`, `PieceFactory` (chess), notification channels |
| **Many optional construction params** | **Builder** | Pizza/burger, `Board` config, HTTP request objects |
| **Exactly one shared instance** | **Singleton** (*sparingly*) | The lot/inventory registry, a logger |
| **Undo / redo**, or queued operations | **Command** | Text editor, chess move history, remote control, job queue |
| A **tree of parts and wholes**, treated uniformly | **Composite** | File system, menus, org charts, UI trees |
| A request that **flows through steps**, any of which may handle or pass it | **Chain of Responsibility** | Express middleware, approval workflows, ATM cash dispensing, logging levels |

**Two more you'll occasionally reach for:** **Decorator** (add responsibilities at runtime — coffee with extra shots, a pizza with toppings) and **Adapter** (bolt a third-party interface onto yours — a payment gateway SDK).

**On Singleton — the honest take.** Interviewers *do* ask for it ("only one parking lot exists"), and you should know it. But say this: *"I can make `ParkingLot` a Singleton, but I'd prefer to just construct one instance in `main()` and inject it. A Singleton is global mutable state — it makes unit tests bleed into each other because they share the instance, and it hides dependencies inside classes instead of declaring them in constructors. If you want the guarantee I'll add it; otherwise DI gives the same 'one instance' with none of the cost."* That single paragraph reliably moves you up a level, because it shows you know the pattern *and* its price.

```js
// If they insist:
class ParkingLot {
  static #instance = null;
  static getInstance(config) {
    if (!ParkingLot.#instance) ParkingLot.#instance = new ParkingLot(config);
    return ParkingLot.#instance;
  }
}
// What you'd rather do — one instance, created once, passed in explicitly:
const lot = new ParkingLot({ floors, feeStrategy: new HourlyFeeStrategy(rates) });
const gate = new EntranceGate(lot);   // dependency is VISIBLE in the signature
```

#### Patternitis: naming a pattern you don't need is WORSE than naming none

If you bolt an `AbstractSpotFactoryProvider` onto a lot with three spot sizes, the interviewer does not think "wow, they know Factory." They think **"this person adds complexity they can't justify — they will do this to our codebase."** That is a *strictly worse* signal than a plain, clean design with no patterns at all.

The test for every pattern you're tempted to add: **"What change does this make cheap, and is that change actually likely?"** If you can't answer in one sentence, don't add it. And if you *do* add it, say the justification out loud — *"I'm making the fee a Strategy because you mentioned weekend pricing is coming; that means pricing is a variation point."*

---

### 6. Step 6 — Write the code (20–25 min)

**Write in this order.** It is the pragmatic order because each layer only depends on the ones above it, so you're never blocked:

```
1. Enums + value objects     (2 min)  — no dependencies, pure data
2. Entities                  (8 min)  — the rich domain objects with their behaviour
3. Interfaces / abstract     (2 min)  — FeeStrategy, PaymentProcessor
   base classes
4. Services / coordinators   (5 min)  — thin! ParkingLot.park() is 6 lines
5. main() demo               (5 min)  — exercise the numbered use-case list from Step 1
```

**The rules of Step 6:**

- **Write RUNNABLE code.** Not pseudocode, not `// TODO: implement`. If it doesn't execute, it doesn't count. Run it. `node lot.js` and show output.
- **Small methods.** If a method exceeds ~15 lines, it's doing two things.
- **Do NOT build a database.** In-memory `Map`s. `#tickets = new Map()`. If they want persistence, a `TicketRepository` *interface* with an `InMemoryTicketRepository` implementation is the perfect answer — it shows you know where the seam goes without spending time on it.
- **Do NOT build a UI, an HTTP server, or auth.** Nobody has ever been rejected for not writing an Express route in a machine-coding round.
- **Handle the obvious errors.** Lot full. Ticket already paid. Spot already occupied. Invalid ID. Throw with a clear message. Skipping error handling entirely reads as careless; five `throw new Error(...)` lines fix it.
- **Interfaces via base classes.** JS has none, so fake them honestly:

```js
// "Interface" in JS: a base class whose methods throw. It documents the contract
// and fails loudly at runtime if a subclass forgets a method.
class FeeStrategy {
  calculate(ticket) { throw new Error('FeeStrategy.calculate() not implemented'); }
}
```

- **Get a working CORE done before flourishes.** This is the single most important sentence in this document:

> ### **A partial-but-running design beats a complete-but-broken one.**

Park → ticket → unpark → fee, running end to end, with a clean object model, at minute 40, beats a half-typed masterpiece with EV charging, a Builder, and an Observer that throws a `TypeError` on line 3. Get the spine working. *Then* add the display board. If you run out of time with a working spine, you can *describe* the rest in 30 seconds — and that describing is nearly as good as building it.

---

### 7. Step 7 — Concurrency, edge cases, extensions (5 min)

**a) Where are the races?** Recall from [52 — Concurrency & Race Conditions] that a race is two operations interleaving on shared state such that the outcome depends on timing. In *every* LLD problem there is a "last one" race:

- Two entrance gates, one free spot.
- Two users, one cinema seat.
- Two riders, one nearby driver.

Node is single-threaded for JS, so a **synchronous** `findSpotFor()` + `assign()` is atomic *for free* — say this, it's a real insight. **But the moment anything in between is `await`ed** (a DB read, a payment call), another request can interleave at that `await` point and you have a genuine double-booking bug.

```js
//  RACE: an await between "check" and "act" is a hole another request drives through.
async park(vehicle) {
  const spot = await this.repo.findFreeSpot(vehicle);   // req A and req B both get spot #7
  await this.repo.markOccupied(spot.id, vehicle);       // both write. One car is now homeless.
}

//  Fix 1 — make check-and-act ONE atomic operation (a conditional/compare-and-set write):
async park(vehicle) {
  const spot = await this.repo.claimFreeSpotAtomically(vehicle);  // UPDATE ... WHERE vehicle_id IS NULL
  if (!spot) throw new LotFullError();                            // lost the race → handle it
  return this.#issueTicket(vehicle, spot);
}

//  Fix 2 (in-process, single Node instance) — serialize the critical section with a mutex.
class Mutex {
  #queue = Promise.resolve();
  runExclusive(fn) {
    const result = this.#queue.then(fn, fn);         // chain onto the tail: strict FIFO
    this.#queue = result.then(() => {}, () => {});   // swallow so one failure doesn't poison the chain
    return result;
  }
}
// const spot = await this.#lock.runExclusive(() => this.#findAndAssign(vehicle));
```

Mention the trade-off in one line: *"A mutex serializes everything and becomes the bottleneck; an atomic conditional write scales, but I have to handle 'I lost the race'. For a single lot the mutex is fine; across gates in separate processes I'd need the DB to arbitrate."* That's a senior answer.

**b) Edge cases to volunteer before they ask:** lot full; vehicle too big for every remaining spot; ticket lost; exit without paying; clock skew (park at 23:59, exit at 00:01 → your duration math must not go negative); zero-duration stay; double payment.

**c) "Now add X."** **The interviewer WILL say this.** It is not an afterthought — **absorbing that request without a rewrite is the entire purpose of your design.** They are testing the object model, and this is the moment it either pays off or exposes itself.

What a good answer sounds like — note that it is *short*, it *names the seam*, and it *changes almost nothing*:

> **"Now add weekend pricing — 1.5× on Saturday and Sunday."**
> *"Pricing is already a Strategy, so this is a new class and one line in `main()`. `class WeekendSurchargeFeeStrategy extends FeeStrategy` wraps the hourly one and multiplies. `ParkingLot` doesn't change at all — that's OCP paying off. Actually, since it *wraps* another strategy rather than replacing it, that's a Decorator, and it means I can stack surcharges later."*

> **"Now add EV charging spots."**
> *"`SpotSize` is about physical fit; charging is a different axis, so I won't add an `EV_SPOT` size — that would conflate two concepts. I'd add a `features: Set<SpotFeature>` to `ParkingSpot` and extend `canFit()` to also check required features. `Vehicle` gets `requiredFeatures()`. Two small edits, no restructuring."*

> **"Now support multiple parking lots in a city."**
> *"`ParkingLot` is already self-contained and I deliberately did **not** make it a Singleton — so I just construct several. I'd add a `ParkingLotRegistry` above them for 'find the nearest lot with a free large spot.' Nothing inside `ParkingLot` changes."*

Every one of those answers is *"my design already has a seam there."* That's the whole game.

---

## Visual / Diagram description

**Diagram 1 — the funnel from English to code.** This is what you're actually doing:

```
   "Design a parking lot."          ← 4 words. Deliberately vague.
             │
             ▼  STEP 1: interrogate
   ┌──────────────────────┐
   │ 7 numbered use cases │  ── becomes your final test checklist ──┐
   │ + explicit OUT-of-   │                                          │
   │   scope list         │                                          │
   └──────────┬───────────┘                                          │
              │                                                      │
     ┌────────┴────────┐                                             │
     ▼ STEP 2          ▼ STEP 3                                      │
 ┌────────┐        ┌────────┐                                        │
 │ NOUNS  │        │ VERBS  │                                        │
 └───┬────┘        └───┬────┘                                        │
     │ filter          │ assign an OWNER                             │
     ▼                 ▼   ("Tell, Don't Ask")                       │
 ┌────────────────────────────────┐                                  │
 │  Entities · Value objects ·    │                                  │
 │  Enums   +  their METHODS      │  ← if verbs pile up in one       │
 └───────────┬────────────────────┘    class, that's your ANEMIC     │
             │ STEP 4                  MODEL warning light           │
             ▼                                                       │
 ┌────────────────────────────────┐                                  │
 │  Class diagram: is-a / has-a / │                                  │
 │  uses-a, with multiplicities   │                                  │
 └───────────┬────────────────────┘                                  │
             │ STEP 5: patterns ONLY where a smell demands one       │
             ▼                                                       │
 ┌────────────────────────────────┐                                  │
 │  STEP 6: CODE (20–25 min)      │                                  │
 │  enums → entities → interfaces │                                  │
 │  → services → main()           │◀─────── verified against ────────┘
 └───────────┬────────────────────┘         the checklist
             │ STEP 7
             ▼
   "Now add X." ──▶ your seams absorb it, or they don't.
```

**Diagram 2 — the anemic vs rich shape.** Draw *both* on the whiteboard and point at the difference. It is instantly legible:

```
   ANEMIC (bad)                          RICH (good)
   ───────────                           ──────────

   ┌───────────────────────┐             ┌──────────────┐
   │   ParkingLotManager   │             │ ParkingLot   │  thin coordinator
   │  ┌─────────────────┐  │             │  park()      │  (~6 lines)
   │  │ park()          │  │             └──┬───────┬───┘
   │  │ unpark()        │  │                │       │
   │  │ calculateFee()  │  │        ┌───────▼──┐ ┌──▼─────────┐
   │  │ findSpot()      │  │        │  Floor   │ │ FeeStrategy│
   │  │ pay()           │  │        │findSpot()│ │ calculate()│
   │  │ notify()        │  │        └───┬──────┘ └────────────┘
   │  │ ... 400 lines   │  │            │
   │  └─────────────────┘  │        ┌───▼────────┐  ┌──────────┐
   └──┬────────┬────────┬──┘        │ParkingSpot │  │  Ticket  │
      │        │        │           │ canFit()   │  │duration()│
      ▼        ▼        ▼           │ assign()   │  │ close()  │
   ┌─────┐ ┌──────┐ ┌────────┐      │ release()  │  └──────────┘
   │Spot │ │Ticket│ │Vehicle │      └────────────┘
   │(data│ │(data)│ │ (data) │
   │only)│ └──────┘ └────────┘      Behaviour sits WITH its data.
   └─────┘                          Each box is independently testable.
   Dumb data bags. All the
   arrows point one way: the
   manager reaches INTO them.
```

The anemic diagram has one fat box and three empty ones, and every arrow points *outward from the manager*. The rich diagram has small boxes that each contain both data *and* verbs, and the arrows are *collaborations*. If your whiteboard looks like the left one, stop and redistribute the behaviour before you write a line of code.

---

## Real world examples

### Domain-Driven Design at large e-commerce and banking systems

Eric Evans coined "anemic domain model" as an *anti-pattern* and Martin Fowler popularized the critique in 2003 — and it is still the most common structural failure in enterprise Java/Node codebases. The industry response, **Domain-Driven Design**, is literally steps 2–4 of this framework given a formal vocabulary: **entities** (identity), **value objects** (immutable, compared by value — our `Money`), **aggregate roots** (the one object outsiders talk to — our `ParkingLot`), and **repositories** (the seam to persistence). When you say "`ParkingLot` is the aggregate root; nobody reaches past it to mutate a `Spot` directly," you are speaking a language every senior backend engineer recognizes.

### Node's own standard library uses the same patterns

You already depend on these daily, which is why the cheat-sheet mapping is worth memorizing:
- **Observer:** `EventEmitter` is Observer, verbatim. `stream.on('data', ...)` is a subscription.
- **Chain of Responsibility:** Express middleware. Each `(req, res, next)` either handles the request or calls `next()` to pass it down the chain. That is the pattern's textbook definition.
- **Decorator:** `stream.pipe(gzip).pipe(cipher)` — each wraps the previous, adding behaviour, preserving the interface.
- **Strategy:** the `compare` function you hand to `Array.prototype.sort()` is an injected algorithm.

Citing these in the interview proves you didn't memorize a pattern catalogue — you recognize the shapes in code you actually use.

### The machine-coding round itself (Flipkart, Swiggy, Uber, Atlassian, and similar)

Many Indian and global product companies run a dedicated 90-minute machine-coding round where you build a runnable system from a one-page problem statement, then walk an interviewer through it. The published rubrics are strikingly consistent, and they are *not* mostly about the algorithm: they score **working code**, **separation of concerns / SRP**, **extensibility**, **no god classes**, and **demonstrable use cases** — and they explicitly penalize over-engineering. The 7-step framework in this doc is reverse-engineered from that rubric.

---

## Trade-offs

**Design time vs coding time (the core tension of the round):**

| Approach | Pros | Cons |
|---|---|---|
| **Code immediately (0 min design)** | Lots of lines on screen early; feels productive | God class almost guaranteed; wrong abstractions calcify; "now add X" forces a rewrite you have no time for. **Most common failure.** |
| **Design for 25 min, code for 25** | Clean model; extensions land in minutes; you narrate SRP/OCP while drawing | Feels terrifyingly slow at minute 20 with no code. **Hold your nerve — this is correct.** |
| **Design for 40 min** | Beautiful UML | You ship nothing runnable. Also a fail. |

**Patterns:**

| | Pros | Cons |
|---|---|---|
| **Use a pattern where a smell exists** | Extension = new class, no edits; you get to *name* the principle | Costs 3–5 min of build time |
| **Use no patterns at all** | Fast, simple, honest; fine for a genuinely simple problem | You'll fail the "now add X" test; reads as junior |
| **Patternitis (patterns everywhere)** | — (there is no pro) | **Worse than none.** Reads as "adds unjustifiable complexity." Actively negative signal. |

**Rich vs anemic:**

| | Rich domain model | Anemic model |
|---|---|---|
| Encapsulation | Invariants enforced by the owning class | Enforced by convention; anyone can break them |
| Testability | Each class in isolation | Only end-to-end through the god class |
| Speed to write | Slightly slower up front | Faster for 20 min, then collapses |
| Honest use case | Always, in LLD | Only for genuine DTOs at a serialization boundary |

**Rule of thumb:** **Spend half your time before you type.** Get the *spine* running end to end before any flourish. Add a pattern only when you can say, in one sentence, *what change it makes cheap* — and if you can't, the absence of a pattern is a feature, not a gap.

---

## Common interview questions on this topic

### Q1: "You've been given the problem. What's the first thing you do?"

**Hint:** Do **not** say "start with the classes." Say: *"First I clarify scope, because these problems are intentionally under-specified. I'd ask who the actors are, get 5–8 core use cases stated as verbs, and — most importantly — nail down what's explicitly out of scope: auth, real persistence, HTTP, payment gateways. Then I'd ask about concurrency and whether any specific algorithm matters. I'll write the agreed use cases as a numbered comment at the top of the file and use it as my test checklist at the end."* That answer alone separates you from most candidates.

### Q2: "What's an anemic domain model, and why is it bad?"

**Hint:** Classes reduced to getters/setters (pure data bags) with all logic hoisted into a `Manager`/`Service` god class. It's procedural code wearing an OOP costume: you pay for the indirection and get none of the encapsulation. Concretely — invariants can't be enforced (any code can do `spot.vehicle = x` and double-book it), you can't unit-test a class in isolation, and adding a variant means editing `if/else` chains in five places. The cure is **"Tell, Don't Ask"**: push each behaviour onto the class that owns the data it reads. Bonus: mention that DTOs at a serialization boundary are the *one* legitimate use of anemic objects.

### Q3: "How do you decide which class a method belongs to?"

**Hint:** Behaviour follows data. If the method reads fields of one object → put it on that object. If it needs two peers → put it on their common container. If it needs something external (clock, gateway, repository) → a thin service with that dependency injected. If it's a *pluggable decision* ("how do we price?") → a Strategy, not an `if/else` on a type code. Name the smell: **a switch on a type code is polymorphism you haven't written yet.**

### Q4: "Which design patterns would you use here, and why?"

**Hint:** The trap is listing patterns. Invert it — start from the *requirement smell*: *"You said the vehicle has states — parked, paid, exited — with different valid transitions, so that's **State**. You hinted pricing will change, so fee is a **Strategy**. The display board must update when a spot frees up, so that's **Observer**. I would **not** add a Factory here — there are three vehicle types and a constructor is enough; adding one would be complexity I can't justify."* **Explicitly declining a pattern scores higher than adding one.**

### Q5: "Now add [feature] to your design."

**Hint:** They are testing your seams, not your typing. Answer in three parts: (1) **which existing abstraction absorbs it** — *"pricing is already a Strategy"*; (2) **exactly what changes** — *"one new class, one line in main(), zero edits to `ParkingLot`"*; (3) **name the principle** — *"that's OCP."* If it genuinely doesn't fit, say so honestly and name the smallest refactor: *"that cuts across my current model — I'd add a `features` set to `Spot` rather than a new subclass, because subclassing on a data difference causes a class explosion."*

---

## Practice exercise

### The Dry Run: Design a Movie Ticket Booking System — steps 1–5 only, no code

**Time-box yourself to exactly 25 minutes.** Set a timer. The discipline is the exercise.

You are given exactly this: *"Design a movie ticket booking system."* That's it.

**Produce a single file `booking-design.md` containing:**

1. **(5 min) Scope.** Write the 5 clarifying questions you'd ask. Then write a numbered list of 6–8 in-scope use cases as **verbs**, plus an explicit OUT-of-scope list. (Hint: is payment in scope? Multiple cities? Seat *selection* or just seat *count*?)

2. **(5 min) Nouns.** Write a 3-sentence problem statement based on your assumed scope. Underline every noun. Then produce a table with columns `Noun | Verdict (Entity / Value Object / Enum / CUT) | Why`. Aim to *cut at least a third of them*. Write the enums as real `Object.freeze({...})` JS.

3. **(5 min) Verbs & ownership.** List every verb. For each, name the owning class and one sentence of justification. **Then find the trap:** someone will want to put `calculateTotalPrice()` on a `BookingService`. Where should it *actually* live, and why? Write the "Tell, Don't Ask" version.

4. **(5 min) Relationships.** Draw the ASCII class diagram with multiplicities. Mark each edge as is-a / composition (◆) / aggregation (◇) / uses-a. Answer explicitly: *is a `Seat` composed into a `Screen`, or aggregated?* Justify with the lifetime test.

5. **(5 min) Patterns.** Name **at most three** patterns, each with a one-sentence justification tied to a specific requirement — and name **two patterns you deliberately rejected**, with why.

**The one thing you must get right:** the last-seat race. Two users click the same seat at the same instant. Write three sentences on where that race lives in your design and how you'd close it.

Do **not** write implementation code. This exercise trains the half of the round that candidates skip — and it's the half that's scored.

---

## Quick reference cheat sheet

- **The 7 steps:** Clarify (5) → Nouns (5) → Verbs + ownership (5) → Relationships (5) → Patterns (5) → **Code (20–25)** → Concurrency/extensions (5).
- **Half your time is design.** Feeling behind at minute 20 with no code is *correct*. Hold your nerve.
- **The problem is under-specified ON PURPOSE.** They're watching whether you ask or assume. Always ask what's OUT of scope.
- **Write the agreed use cases as a numbered comment** at the top of your file. It's your test checklist for `main()`.
- **Nouns → classes, but FILTER.** Attributes ("colour"), actors ("attendant"), and out-of-scope nouns are *not* classes. Sort into Entity / Value Object / Enum / CUT.
- **Enums in JS = `Object.freeze({...})`.** Value objects = immutable, operations return new instances.
- **"Which class OWNS this behaviour?" IS object-oriented design.** Behaviour belongs with the data it operates on.
- **Tell, Don't Ask.** Don't pull data out of an object to compute elsewhere — tell the object to do it.
- **Anemic domain model = the #1 LLD failure.** Data bags + a `Manager` god class = procedural code in an OOP costume. Name it, avoid it, call it out.
- **A switch on a type code is polymorphism you haven't written yet.** Usually it wants Strategy or a subclass.
- **Pattern smells:** modes→**State** · swappable algorithms→**Strategy** · notify-on-change→**Observer** · complex creation→**Factory** · optional params→**Builder** · one instance→**Singleton** (prefer DI) · undo/queue→**Command** · part-whole tree→**Composite** · request through steps→**Chain of Responsibility**.
- **Patternitis is WORSE than no patterns.** Declining a pattern out loud, with a reason, scores higher than adding one you can't justify.
- **Code order:** enums/value objects → entities → interfaces → thin services → `main()` demo. In-memory `Map`s. No DB, no UI, no auth.
- **A partial-but-running design beats a complete-but-broken one.** Always.
- **"Now add X" is the real exam.** Answer with: which abstraction absorbs it, exactly what changes, and the principle's name (usually OCP).
- **Every LLD problem has a "last one" race.** Sync code in Node is atomic for free; an `await` between check and act is a double-booking bug.

---

## The interviewer's actual rubric

You are scored on five things, roughly in this weight order. Know them, and steer toward them.

| # | What they score | What "good" looks like | Instant fail |
|---|---|---|---|
| 1 | **Does it work?** | `node solution.js` runs and prints the use cases executing | Doesn't run; pseudocode; `// TODO` |
| 2 | **Clean responsibilities** | Each class has one job; behaviour sits with its data; no class over ~80 lines | A `Manager` god class; public mutable fields |
| 3 | **Extensible** | "Now add X" = new class + one wiring line | "Now add X" = rewrite |
| 4 | **Trade-offs named** | You said "Strategy here because pricing will change"; "DI over Singleton because tests" | Silent decisions; can't justify anything |
| 5 | **NOT over-engineered** | Patterns only where a smell demanded one; you *declined* one out loud | AbstractFactoryProviderBuilder for 3 spot types |

**How to present.** Don't dump the file. Narrate in dependency order, 60 seconds each:
1. **"Here's the scope we agreed"** — read the numbered comment.
2. **"Here are the enums and value objects"** — 20 seconds, they're trivial.
3. **"Here's the heart of the model"** — the 2–3 entities with the real behaviour. Spend your time here. Point at one method and say *why it lives there*.
4. **"Here's the variation point"** — the Strategy/State interface. This is where you say OCP.
5. **"Here's the coordinator"** — show that `park()` is 6 lines and delegates. That *proves* it isn't anemic.
6. **"Here's it running"** — execute `main()`, then point back at the numbered checklist: *"use cases 1 through 7, all exercised."*
7. **"Here's what I'd add next, and where it plugs in"** — pre-empt their "now add X" by naming your own seams.

---

## Worked example — all 7 steps on a coffee vending machine

Watch the whole procedure execute once, compressed, before you hit topic 112.

**Step 1 — Clarify (asked & answered).** Actors: customer, operator (refills). Use cases: (1) show available drinks, (2) select a drink, (3) insert coins, (4) reject the selection if insufficient funds or out of ingredients, (5) dispense drink + change, (6) cancel and refund, (7) operator refills ingredients. Out of scope: card payments, a real UI, network. Constraint: **the machine has distinct modes** and it can only serve one customer at a time. That word "modes" is a gift — it means **State**.

**Step 2 — Nouns.** *machine, drink, coffee, latte, coin, money, ingredient, water, milk, beans, inventory, customer, change, selection.*

| Noun | Verdict |
|---|---|
| VendingMachine | **Entity** (aggregate root) |
| Drink (coffee/latte/espresso) | **Entity** — recipe + price. Not subclasses: they differ only by *data*. |
| Ingredient (water/milk/beans) | **Enum** `Ingredient` |
| Inventory | **Entity** — owns counts, enforces "can't go negative" |
| Coin | **Enum** `Coin` (5, 10, 20 minor units) |
| Money / change | **Value object** `Money` |
| customer, selection | **CUT** — an actor and a transient variable |

**Step 3 — Verbs & ownership.** *select* → `VendingMachine` (it's a mode transition). *insertCoin* → `VendingMachine` (accumulates the balance). *canMake(drink)* → **`Inventory`** — it owns the counts, so it answers the question. Do NOT ask the inventory for its counts and compare them in the machine; **tell** it. *consume(recipe)* → `Inventory`, and it throws rather than going negative. *dispense* / *refund* → the **State** objects.

**Step 4 — Relationships.**

```
┌──────────────────────────────┐        ┌───────────────────┐
│      VendingMachine          │──────▶ │  «interface»      │
│ - inventory : Inventory      │ uses-a │  MachineState     │
│ - balance   : Money          │        ├───────────────────┤
│ - selected  : Drink|null     │        │ +selectDrink(m,d) │
│ - state     : MachineState   │        │ +insertCoin(m,c)  │
├──────────────────────────────┤        │ +dispense(m)      │
│ + selectDrink(d) / insertCoin│        │ +cancel(m)        │
│ + dispense() / cancel()      │        └─────────▲─────────┘
└───┬────────────────┬─────────┘                  │ is-a
  ◆ │ composition    │◇ aggregation      ┌────────┼────────┬──────────┐
    ▼                ▼                   │        │        │          │
┌───────────┐   ┌──────────┐        ┌────┴───┐┌───┴────┐┌──┴─────┐┌───┴──────┐
│ Inventory │   │  Drink   │        │  Idle  ││Selected││HasMoney││Dispensing│
│ +canMake()│   │ +recipe  │        │ State  ││ State  ││ State  ││  State   │
│ +consume()│   │ +price   │        └────────┘└────────┘└────────┘└──────────┘
└───────────┘   └──────────┘
```

**Step 5 — Patterns.** **State** (the machine's four modes with different valid actions — this is the requirement, verbatim). Everything else: **declined**. No Strategy (one pricing rule). No Factory (drinks are config data, not variants). No Observer (nothing subscribes). *Say the declines out loud.*

**Step 6 — Code.**

```js
const Ingredient = Object.freeze({ WATER: 'WATER', MILK: 'MILK', BEANS: 'BEANS' });
const Coin = Object.freeze({ FIVE: 5, TEN: 10, TWENTY: 20 });

class Inventory {                       // OWNS the counts → owns the questions about them
  #stock = new Map();
  add(ingredient, qty) { this.#stock.set(ingredient, (this.#stock.get(ingredient) ?? 0) + qty); }
  canMake(recipe) { return [...recipe].every(([ing, q]) => (this.#stock.get(ing) ?? 0) >= q); }
  consume(recipe) {                     // enforces its own invariant: never negative
    if (!this.canMake(recipe)) throw new Error('Out of ingredients');
    for (const [ing, q] of recipe) this.#stock.set(ing, this.#stock.get(ing) - q);
  }
}

class Drink {
  constructor(name, priceMinor, recipe) { this.name = name; this.price = priceMinor; this.recipe = recipe; }
}

class MachineState {                    // "interface": fails loudly if a subclass forgets one
  selectDrink() { throw new Error('Cannot select in this state'); }
  insertCoin()  { throw new Error('Cannot insert coins in this state'); }
  dispense()    { throw new Error('Nothing to dispense'); }
  cancel()      { throw new Error('Nothing to cancel'); }
  get name()    { return this.constructor.name; }
}

class IdleState extends MachineState {
  selectDrink(machine, drink) {
    if (!machine.inventory.canMake(drink.recipe)) throw new Error(`${drink.name} is sold out`);
    machine.selected = drink;
    machine.setState(new AwaitingPaymentState());   // the STATE decides the next state
  }
}

class AwaitingPaymentState extends MachineState {
  insertCoin(machine, coin) {
    machine.balance += coin;
    if (machine.balance >= machine.selected.price) machine.setState(new ReadyToDispenseState());
  }
  cancel(machine) { machine.refundAndReset(); }
}

class ReadyToDispenseState extends MachineState {
  insertCoin(machine, coin) { machine.balance += coin; }   // extra coins → more change back
  dispense(machine) {
    machine.inventory.consume(machine.selected.recipe);    // TELL the inventory
    const change = machine.balance - machine.selected.price;
    const drink = machine.selected;
    machine.reset();
    return { drink: drink.name, change };
  }
  cancel(machine) { machine.refundAndReset(); }
}

class VendingMachine {
  #state = new IdleState();
  constructor(inventory, drinks) { this.inventory = inventory; this.drinks = drinks; this.balance = 0; this.selected = null; }
  setState(s) { this.#state = s; }
  get state() { return this.#state.name; }
  reset() { this.balance = 0; this.selected = null; this.setState(new IdleState()); }
  refundAndReset() { const r = this.balance; this.reset(); return r; }

  // The machine is a THIN delegator. Every branch of "what's legal right now"
  // lives in the state objects — there is not a single if/else about mode here.
  selectDrink(name) { this.#state.selectDrink(this, this.drinks.get(name)); }
  insertCoin(coin)  { this.#state.insertCoin(this, coin); }
  dispense()        { return this.#state.dispense(this); }
  cancel()          { return this.#state.cancel(this); }
}

// ---- main(): exercise the numbered use cases from Step 1 ----
function main() {
  const inv = new Inventory();
  inv.add(Ingredient.WATER, 10); inv.add(Ingredient.MILK, 4); inv.add(Ingredient.BEANS, 10);

  const drinks = new Map([
    ['espresso', new Drink('espresso', 20, new Map([[Ingredient.WATER, 1], [Ingredient.BEANS, 2]]))],
    ['latte',    new Drink('latte',    35, new Map([[Ingredient.WATER, 1], [Ingredient.MILK, 2], [Ingredient.BEANS, 2]]))],
  ]);
  const machine = new VendingMachine(inv, drinks);

  machine.selectDrink('latte');                         // UC 2
  console.log('state:', machine.state);                 // AwaitingPaymentState
  machine.insertCoin(Coin.TWENTY);                      // UC 3
  machine.insertCoin(Coin.TWENTY);
  console.log('state:', machine.state);                 // ReadyToDispenseState
  console.log(machine.dispense());                      // UC 5 → { drink: 'latte', change: 5 }

  machine.selectDrink('espresso');                      // UC 6 — cancel path
  machine.insertCoin(Coin.TEN);
  console.log('refunded:', machine.cancel(), '| state:', machine.state);   // 10 | IdleState

  try { machine.dispense(); } catch (e) { console.log('rejected:', e.message); }  // UC 4
}
main();
```

**Step 7 — Concurrency & extensions.** The race: two customers hitting `dispense()` for the last milk. Every mutation here is synchronous, so in a single Node process it's atomic for free — but the moment `dispense()` awaits a hardware call, `canMake` → `consume` becomes a check-then-act hole; I'd move it behind one atomic `Inventory.tryConsume(recipe)` that returns false instead of throwing.

*"Now add card payments."* → `AwaitingPaymentState` gains an `insertCard()`; add a `PaymentMethod` **Strategy** so cash and card both just credit the balance. Nothing else moves.
*"Now add a maintenance mode."* → a new `MaintenanceState` class. **Zero edits to `VendingMachine`.** That's OCP, and it's why State earned its place.

---

## Connected topics

| Direction | Topic |
|---|---|
| **Previous** | [22 — UML Use Case Diagrams](./22-uml-use-case-diagrams.md) — how to formalize the actors and use cases you extract in Step 1 |
| **Next** | [112 — LLD: Parking Lot](./112-lld-parking-lot.md) — the first full case study; run all 7 steps of this framework on it end to end |
| **Related** | [20 — UML Class Diagrams](./20-uml-class-diagrams.md) — the notation for the diagram you draw in Step 4 |
| **Related** | [23 — Class Relationships](./23-class-relationships.md) — is-a vs has-a vs uses-a, the exact decision you make in Step 4 |
| **Related** | [28 — What Are Design Patterns](./28-what-are-design-patterns.md) — the catalogue behind the Step 5 smell→pattern cheat-sheet |
