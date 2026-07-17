# 118 — Design BookMyShow / Movie Ticket Booking
## Category: LLD Case Study

---

## What is this?

This is the class-level design for a movie-ticket booking system — the software behind BookMyShow, Fandango, or the ticket screen at your local multiplex. You browse movies, pick a show, look at the seat map, choose seats A5 and A6, and pay. The system's single hardest job: **when two strangers tap seat A5 for the same show at the same millisecond, exactly one of them gets it.**

Think of it as a theatre box office with one physical ticket per seat. Two people reach for the same ticket; only one hand can close around it. Everything else — movies, screens, pricing — is scaffolding around that one rule.

---

## Why does it matter?

**Interview angle.** This is the canonical "concurrency at the object level" LLD question, the class-level twin of the Ticketmaster HLD (topic 110). Interviewers use it to see whether you can model *shared-but-per-context state* (one physical seat, many shows) and whether you understand that "check if free, then book" is a **race condition**, not a solution. If you write `if (seat.isAvailable()) seat.book()` and move on, you have failed the question.

**Real-work angle.** Double-booking is not a hypothetical. If your code sells one seat twice, someone flies to a city, drives to a cinema, and is turned away — a refund, a support ticket, and a lost customer. The exact same pattern shows up in hotel rooms (117), airline seats, flash-sale inventory, and appointment slots. Master it once here and you own a whole family of problems.

---

## The core idea — explained simply

### Analogy: the coat-check tag

Imagine a coat check with numbered hooks. A **hook** (hook #47) is a permanent fixture — it exists all day, every day. But whether hook #47 is *free right now* depends on the current session. Tonight's 7 PM crowd and tomorrow's matinee crowd both use hook #47, but their occupancy is tracked separately.

So there are two different things:
- The **hook** — a physical fixture, defined once when the coat check is built.
- The **hook's status for this session** — free / reserved / occupied, tracked per session.

A movie seat is exactly a coat-check hook.

| Coat check | Movie system | Why it matters |
|---|---|---|
| Hook (physical fixture) | `Seat` | Defined once per screen; shared across all shows |
| Session (tonight's crowd) | `Show` | A movie on a screen at a specific time |
| Hook's status this session | `ShowSeat` | Availability of *one seat* for *one show* |
| "Reach for the tag" | `holdSeats()` | Atomic AVAILABLE → HELD |
| Walking to the counter to pay | The hold's TTL window | You have N minutes or the tag goes back |
| Handing over your ticket | `confirmBooking()` | HELD → BOOKED |

The single most common modelling mistake is putting `status` directly on `Seat`. Do that and seat A5 can only ever be "booked" or "free" globally — booking it for the 7 PM show would also book it for the matinee. The **`ShowSeat`** — a small object that links `Seat` + `Show` + `status` — is the insight the whole design hangs on.

---

## Key concepts inside this topic

We follow the standard 7-step LLD method: clarify, find nouns, find verbs, draw the classes, solve the hard part, code it, name the patterns.

### 1. Requirements & clarifying questions

**Functional requirements (what it must do):**
- Browse movies, cinemas, and the shows on offer.
- View the seat map for a chosen show (which seats are free).
- Select one or more seats.
- **Hold** those seats briefly while the user pays.
- **Confirm** the booking after payment.
- **Never** let two users book the same seat for the same show.

**Clarifying questions to ask the interviewer** (asking these signals seniority):
- *Seat types and pricing?* Are there REGULAR / PREMIUM / RECLINER tiers at different prices? (Yes — drives a pricing Strategy.)
- *Hold timeout?* How long may a user hold seats before paying? (Assume 5 minutes; after that the hold auto-expires.)
- *Group bookings?* Must a group of seats be adjacent? (Out of scope for the core, an extension.)
- *Payment?* Do we integrate a real gateway? (We stub it — payment is a boundary, not the puzzle.)
- *Scale?* Single node in-memory, or distributed? (We build in-memory but call out exactly where a distributed lock replaces the in-process one — see step 5.)

**Non-functional:** correctness under concurrency (the star), low latency on the seat map, and idempotent confirmation (a retried "confirm" must not double-charge).

### 2. Identify core objects (the nouns)

Read the requirements and underline the nouns:

| Object | What it represents |
|---|---|
| `Movie` | A film: title, duration, language |
| `Cinema` | A physical venue with several screens |
| `Screen` / `Hall` | One auditorium inside a cinema; owns a fixed set of seats |
| `Seat` | A **physical** seat in a screen (row + number + type). Shared across all shows on that screen |
| `Show` | A `Movie` playing on a `Screen` at a `startTime` |
| `ShowSeat` | **The key object.** A `Seat`'s status *for one particular `Show`* (available/held/booked, who holds it, when the hold expires) |
| `Booking` | A confirmed purchase: a `User`, a `Show`, the `ShowSeat`s, a total, a status |
| `User` | Whoever is booking |
| `Payment` | The money side (stubbed) |

**Enums** (fixed sets of values — modelled with `Object.freeze`):
- `SeatType` = REGULAR / PREMIUM / RECLINER
- `SeatStatus` = AVAILABLE / HELD / BOOKED
- `BookingStatus` = PENDING / CONFIRMED / CANCELLED

### 3. Behaviours (the verbs) + who owns them

| Behaviour | Owner | Notes |
|---|---|---|
| `searchShows(movie, city)` | `BookingService` | Find shows for a movie |
| `getAvailableSeats(show)` | `BookingService` | The seat map: which `ShowSeat`s are AVAILABLE |
| `holdSeats(user, show, seatIds)` | `BookingService` | Atomic AVAILABLE → HELD with a TTL. All-or-nothing |
| `confirmBooking(user, holdId)` | `BookingService` | HELD → BOOKED. Idempotent |
| `releaseExpiredHolds()` | `BookingService` | Sweep expired holds back to AVAILABLE |
| `price()` | `PricingStrategy` | Cost of a seat, by type |

The crux of the modelling: **`Seat` vs `ShowSeat`** (a physical thing vs its per-show availability) and the **three-state lifecycle** `AVAILABLE → HELD → BOOKED`. A seat can also travel `HELD → AVAILABLE` (hold expired or released). It can never jump straight from AVAILABLE to BOOKED — the hold is mandatory, because the hold is what buys the user time to pay without a competitor snatching the seat.

### 4. Class diagram

```
        ┌───────────┐         ┌───────────┐
        │   Movie   │         │   User    │
        └─────┬─────┘         └─────┬─────┘
              │ plays in            │ makes
              ▼                     ▼
   ┌──────────────────┐      ┌──────────────┐
   │      Cinema      │      │   Booking    │
   │  ◇──────────┐    │      │  status      │
   └───┬──────────┘   │      │  total       │
       │ owns *       │      └──┬────────┬──┘
       ▼              │  refs   │        │ contains *
  ┌──────────┐        │  Show   │        ▼
  │  Screen  │◄───────┼─────────┘   ┌──────────┐
  │ 1        │        │             │ ShowSeat │
  └────┬─────┘        │             │ status   │
       │ 1..* has     │             │ heldBy   │
       ▼              │             │ expiresAt│
  ┌──────────┐        │             └────┬─────┘
  │   Seat   │◄───────┼──────────────────┘ wraps 1
  │ row,num  │        │
  │ type     │   ┌────┴─────┐
  └──────────┘   │   Show   │  owns * ShowSeats
                 │ startTime│  refs 1 Movie + 1 Screen
                 └──────────┘
```

Read it as: a `Cinema` is composed of (`◇`) `Screen`s; a `Screen` has 1..* physical `Seat`s; a `Show` references one `Screen` and one `Movie` and **owns** a set of `ShowSeat`s (one per physical seat, created when the show is scheduled); a `Booking` references a `Show`, a `User`, and the specific `ShowSeat`s it locked.

### 5. The concurrency problem — the heart of this problem

This is the class-level mirror of the Ticketmaster HLD (110). At the system level, 110 talks about distributed locks and databases. Here we solve the *same* race inside one process, then point at how it scales out.

**The naive race (broken).** Two requests run `holdSeats` for seat A5 at the same time:

```
Time  User A                         User B
 t1   read A5.status → AVAILABLE
 t2                                  read A5.status → AVAILABLE
 t3   set A5.status = HELD (A)
 t4                                  set A5.status = HELD (B)   ← overwrites A!
```

Both saw AVAILABLE (nothing had changed yet), both wrote HELD. Seat A5 is now "held" by B, A thinks they hold it too, and both proceed to pay. **Double booking.** The bug is the gap between *check* and *set* — another thread sneaks in between.

**Fix 1 — make check-and-set atomic.** The read and the write must be a single indivisible step: *"if status is AVAILABLE, set it to HELD, and let no one interleave."* This is compare-and-set (CAS).

- In our **single-node in-memory** model we enforce atomicity with an async mutex (recall topic 52 — a lock the `await`ing task must acquire before touching seat state). Because Node runs JS on one thread, the danger is *interleaving across `await` points*, not true parallelism; the mutex serialises the critical section so only one hold operation mutates seats at a time.
- In a **real distributed system**, "atomic" comes from the database: a `SELECT ... FOR UPDATE` row lock, or a conditional update (`UPDATE show_seat SET status='HELD' WHERE id=? AND status='AVAILABLE'` — succeeds for exactly one caller, the other gets 0 rows affected), or a **distributed lock** across nodes (recall topic 84 — Redis Redlock / a lease). The *shape* is identical; only the lock's implementation changes. This is the one place you must say that out loud in an interview.

**Fix 2 — hold with a TTL.** Grabbing the seat isn't enough; the user needs time to pay without a competitor waiting them out. So a hold carries an **expiry timestamp**. If `confirmBooking` doesn't arrive within N minutes, a sweep (`releaseExpiredHolds`) flips the seat HELD → AVAILABLE. Payment latency is thus decoupled from seat contention: you hold, you pay calmly, you confirm — or you time out and the seat is freed for the next person.

**Fix 3 — idempotent confirmation.** `confirmBooking` moves HELD → BOOKED *only* if the hold is still valid, unexpired, and owned by this user. Called twice (a retried request, a double-click), the second call sees the booking already CONFIRMED and returns the same result instead of charging again. No new side effects — that is what "idempotent" means.

### 6. Full JavaScript implementation

Complete and runnable — save as `movie-booking.js` and run `node movie-booking.js`.

```javascript
'use strict';

// ---------- Enums (fixed value sets, frozen so they can't be mutated) ----------
const SeatType = Object.freeze({ REGULAR: 'REGULAR', PREMIUM: 'PREMIUM', RECLINER: 'RECLINER' });
const SeatStatus = Object.freeze({ AVAILABLE: 'AVAILABLE', HELD: 'HELD', BOOKED: 'BOOKED' });
const BookingStatus = Object.freeze({ PENDING: 'PENDING', CONFIRMED: 'CONFIRMED', CANCELLED: 'CANCELLED' });

// ---------- A tiny async mutex (recall topic 52) ----------
// Node is single-threaded, but code can interleave across `await`. The mutex
// guarantees that the check-and-set critical section runs to completion for one
// caller before the next begins — this is what makes AVAILABLE->HELD atomic.
class Mutex {
  constructor() { this._queue = Promise.resolve(); }
  // runExclusive(fn): fn runs only after all previously-queued fns have finished.
  runExclusive(fn) {
    const result = this._queue.then(() => fn());
    // Keep the chain alive even if fn throws, so the lock never deadlocks.
    this._queue = result.then(() => {}, () => {});
    return result;
  }
}

// ---------- Domain objects ----------
class Movie {
  constructor(id, title, durationMins, language) {
    this.id = id; this.title = title;
    this.durationMins = durationMins; this.language = language;
  }
}

class Cinema {
  constructor(id, name, city) {
    this.id = id; this.name = name; this.city = city;
    this.screens = []; // composition: cinema owns its screens
  }
  addScreen(screen) { this.screens.push(screen); return screen; }
}

class Screen {
  constructor(id, name) {
    this.id = id; this.name = name;
    this.seats = []; // physical seats, defined once for this screen
  }
  addSeat(seat) { this.seats.push(seat); return seat; }
}

// A PHYSICAL seat. Notice: NO status here. Status is per-show, on ShowSeat.
class Seat {
  constructor(id, row, number, type) {
    this.id = id; this.row = row; this.number = number; this.type = type;
  }
  get label() { return `${this.row}${this.number}`; } // e.g. "A5"
}

class User {
  constructor(id, name) { this.id = id; this.name = name; }
}

// ---------- Pricing Strategy (see patterns section) ----------
// Each seat type prices differently; a Strategy lets us swap the rule freely.
class PricingStrategy {
  price(showSeat) { throw new Error('not implemented'); }
}
class TypeBasedPricing extends PricingStrategy {
  constructor(basePrice) { super(); this.base = basePrice; }
  price(showSeat) {
    const multiplier = { REGULAR: 1, PREMIUM: 1.5, RECLINER: 2 }[showSeat.seat.type];
    return Math.round(this.base * multiplier);
  }
}

// A seat's status FOR ONE SHOW. This is the modelling keystone.
class ShowSeat {
  constructor(seat) {
    this.seat = seat;                 // the physical seat it wraps
    this.status = SeatStatus.AVAILABLE;
    this.heldBy = null;               // userId currently holding it
    this.holdExpiresAt = null;        // epoch ms when the hold lapses
  }
  isAvailable() { return this.status === SeatStatus.AVAILABLE; }
}

class Show {
  constructor(id, movie, screen, startTime, pricing) {
    this.id = id; this.movie = movie; this.screen = screen;
    this.startTime = startTime; this.pricing = pricing;
    // On scheduling, create exactly one ShowSeat per physical seat.
    this.showSeats = new Map();       // seatId -> ShowSeat
    for (const seat of screen.seats) this.showSeats.set(seat.id, new ShowSeat(seat));
  }
}

class Booking {
  constructor(id, user, show, showSeats) {
    this.id = id; this.user = user; this.show = show;
    this.showSeats = showSeats;
    this.status = BookingStatus.PENDING;
    this.total = showSeats.reduce((sum, ss) => sum + show.pricing.price(ss), 0);
  }
}

// ---------- The service that orchestrates everything ----------
class BookingService {
  constructor({ holdTtlMs = 5 * 60 * 1000 } = {}) {
    this.holdTtlMs = holdTtlMs;
    this._locks = new Map();          // showId -> Mutex (one critical section per show)
    this._holds = new Map();          // holdId -> { userId, showId, seatIds, expiresAt }
    this._bookings = new Map();       // holdId -> Booking (idempotency: reuse per hold)
    this._holdSeq = 0;
  }

  _lockFor(showId) {
    if (!this._locks.has(showId)) this._locks.set(showId, new Mutex());
    return this._locks.get(showId);
  }

  // Read-only seat map: which seats can still be picked right now.
  getAvailableSeats(show) {
    const now = Date.now();
    const available = [];
    for (const ss of show.showSeats.values()) {
      // A lazily-expired hold still counts as free.
      const expired = ss.status === SeatStatus.HELD && ss.holdExpiresAt <= now;
      if (ss.status === SeatStatus.AVAILABLE || expired) available.push(ss);
    }
    return available;
  }

  // Atomic AVAILABLE -> HELD for a set of seats. ALL-OR-NOTHING.
  async holdSeats(user, show, seatIds) {
    return this._lockFor(show.id).runExclusive(() => {
      const now = Date.now();
      const grabbed = [];
      for (const seatId of seatIds) {
        const ss = show.showSeats.get(seatId);
        if (!ss) { this._rollback(grabbed); throw new Error(`No such seat ${seatId}`); }
        // Treat an expired hold as available (lazy reclaim inside the lock).
        if (ss.status === SeatStatus.HELD && ss.holdExpiresAt <= now) {
          ss.status = SeatStatus.AVAILABLE; ss.heldBy = null; ss.holdExpiresAt = null;
        }
        if (!ss.isAvailable()) {
          // Someone else owns this seat -> abort the WHOLE request.
          this._rollback(grabbed);
          return { ok: false, reason: `Seat ${ss.seat.label} is not available` };
        }
        // The atomic transition — safe because we hold the show's mutex.
        ss.status = SeatStatus.HELD;
        ss.heldBy = user.id;
        ss.holdExpiresAt = now + this.holdTtlMs;
        grabbed.push(ss);
      }
      const holdId = `hold_${++this._holdSeq}`;
      this._holds.set(holdId, {
        userId: user.id, showId: show.id, seatIds: [...seatIds],
        expiresAt: now + this.holdTtlMs,
      });
      return { ok: true, holdId, seats: grabbed.map(ss => ss.seat.label),
               expiresAt: now + this.holdTtlMs };
    });
  }

  // Undo partial holds when an all-or-nothing request fails midway.
  _rollback(showSeats) {
    for (const ss of showSeats) {
      ss.status = SeatStatus.AVAILABLE; ss.heldBy = null; ss.holdExpiresAt = null;
    }
  }

  // HELD -> BOOKED. Idempotent: a repeat call returns the same Booking.
  async confirmBooking(user, show, holdId) {
    return this._lockFor(show.id).runExclusive(() => {
      // Idempotency: already confirmed for this hold -> return the same result.
      if (this._bookings.has(holdId)) {
        return { ok: true, booking: this._bookings.get(holdId), idempotent: true };
      }
      const hold = this._holds.get(holdId);
      if (!hold) return { ok: false, reason: 'Unknown or already-released hold' };
      if (hold.userId !== user.id) return { ok: false, reason: 'Hold not owned by user' };
      if (hold.expiresAt <= Date.now()) return { ok: false, reason: 'Hold expired' };

      const showSeats = hold.seatIds.map(id => show.showSeats.get(id));
      for (const ss of showSeats) {
        if (ss.status !== SeatStatus.HELD || ss.heldBy !== user.id) {
          return { ok: false, reason: `Seat ${ss.seat.label} lost its hold` };
        }
      }
      // Simulated payment would go here. On success, commit HELD -> BOOKED.
      for (const ss of showSeats) { ss.status = SeatStatus.BOOKED; ss.holdExpiresAt = null; }
      const booking = new Booking(`bk_${holdId}`, user, show, showSeats);
      booking.status = BookingStatus.CONFIRMED;
      this._bookings.set(holdId, booking);
      this._holds.delete(holdId);
      return { ok: true, booking, idempotent: false };
    });
  }

  // Background sweep: any hold past its TTL returns its seats to AVAILABLE.
  async releaseExpiredHolds() {
    const now = Date.now();
    let released = 0;
    for (const [holdId, hold] of [...this._holds.entries()]) {
      if (hold.expiresAt > now) continue;
      const show = this._shows?.get(hold.showId);
      await this._lockFor(hold.showId).runExclusive(() => {
        for (const seatId of hold.seatIds) {
          const ss = show?.showSeats.get(seatId);
          if (ss && ss.status === SeatStatus.HELD && ss.heldBy === hold.userId) {
            ss.status = SeatStatus.AVAILABLE; ss.heldBy = null; ss.holdExpiresAt = null;
            released++;
          }
        }
        this._holds.delete(holdId);
      });
    }
    return released;
  }

  registerShow(show) { (this._shows ??= new Map()).set(show.id, show); }
}

// ---------- Demo ----------
async function main() {
  // Build a screen with a 2x3 grid: row A regular, row B premium.
  const screen = new Screen('scr1', 'Audi 1');
  const layout = [['A', SeatType.REGULAR], ['B', SeatType.PREMIUM]];
  let seatNo = 0;
  for (const [row, type] of layout)
    for (let n = 5; n <= 7; n++) screen.addSeat(new Seat(`s${++seatNo}`, row, n, type));

  const cinema = new Cinema('cin1', 'PVR Central', 'Bengaluru');
  cinema.addScreen(screen);
  const movie = new Movie('m1', 'Interstellar', 169, 'English');
  const pricing = new TypeBasedPricing(200); // REGULAR 200, PREMIUM 300, RECLINER 400
  const show = new Show('show1', movie, screen, new Date('2026-07-20T19:00:00'), pricing);

  // Use a SHORT ttl so the demo can watch a hold expire.
  const service = new BookingService({ holdTtlMs: 1000 });
  service.registerShow(show);

  const alice = new User('u1', 'Alice');
  const bob = new User('u2', 'Bob');

  const idOf = (label) => [...show.showSeats.values()].find(ss => ss.seat.label === label).seat.id;
  const printMap = (tag) => {
    const state = [...show.showSeats.values()].map(ss => `${ss.seat.label}:${ss.status}`).join('  ');
    console.log(`  [${tag}] ${state}`);
  };

  console.log('== Initial seat map ==');
  printMap('start');

  console.log('\n== Alice holds A5 + A6 ==');
  const aliceHold = await service.holdSeats(alice, show, [idOf('A5'), idOf('A6')]);
  console.log('  Alice:', aliceHold);
  printMap('after Alice');

  console.log('\n== Bob races for A5 (must FAIL — Alice holds it) ==');
  const bobHold = await service.holdSeats(bob, show, [idOf('A5'), idOf('A7')]);
  console.log('  Bob:', bobHold, '<-- rejected, and A7 was NOT left half-held (all-or-nothing)');
  printMap('after Bob fails');

  console.log('\n== Alice confirms her booking (HELD -> BOOKED) ==');
  const confirm1 = await service.confirmBooking(alice, show, aliceHold.holdId);
  console.log('  Confirm:', { ok: confirm1.ok, total: confirm1.booking.total, status: confirm1.booking.status });
  printMap('after confirm');

  console.log('\n== Alice retries confirm (idempotent — no double charge) ==');
  const confirm2 = await service.confirmBooking(alice, show, aliceHold.holdId);
  console.log('  Retry idempotent?', confirm2.idempotent, '| same booking?', confirm2.booking === confirm1.booking);

  console.log('\n== Bob now holds A7 (free again), then we let it EXPIRE ==');
  const bobHold2 = await service.holdSeats(bob, show, [idOf('A7')]);
  console.log('  Bob holds A7:', bobHold2.ok, bobHold2.seats);
  printMap('Bob holds A7');
  await new Promise(r => setTimeout(r, 1100)); // wait past the 1s TTL
  const released = await service.releaseExpiredHolds();
  console.log(`  Sweep released ${released} seat(s) back to AVAILABLE`);
  printMap('after expiry sweep');

  console.log('\n== Available seats now ==');
  console.log('  ', service.getAvailableSeats(show).map(ss => ss.seat.label).join(', '));
}

main().catch(err => { console.error(err); process.exit(1); });

module.exports = { BookingService, Show, ShowSeat, Seat, Screen, Cinema, Movie, User,
                   SeatType, SeatStatus, BookingStatus, TypeBasedPricing };
```

**Expected output (abridged):**

```
== Alice holds A5 + A6 ==
  Alice: { ok: true, holdId: 'hold_1', seats: [ 'A5', 'A6' ], ... }
== Bob races for A5 (must FAIL) ==
  Bob: { ok: false, reason: 'Seat A5 is not available' } <-- rejected...
== Alice confirms ==
  Confirm: { ok: true, total: 400, status: 'CONFIRMED' }
== Alice retries confirm ==
  Retry idempotent? true | same booking? true
== after expiry sweep ==
  [after expiry sweep] A5:BOOKED  A6:BOOKED  A7:AVAILABLE  B5:AVAILABLE ...
```

Notice the three proofs the concurrency control runs: Bob's race is **rejected**, his half-grab of A7 is **rolled back** (all-or-nothing), the retry is **idempotent**, and the expired hold is **swept back** to AVAILABLE.

### 7. Design patterns used and WHY

| Pattern | Where | Why it fits |
|---|---|---|
| **Strategy** | `PricingStrategy` / `TypeBasedPricing` | Pricing varies (by seat type today, by surge tomorrow) independently of the booking flow. Swap the strategy without touching `BookingService`. Open/Closed. |
| **State** | `SeatStatus` (AVAILABLE→HELD→BOOKED) and `BookingStatus` (PENDING→CONFIRMED→CANCELLED) | The object behaves differently per state, and only certain transitions are legal. Modelling status explicitly makes illegal jumps (AVAILABLE→BOOKED) impossible to reach. |
| **Facade** | `BookingService` | A single, simple entry point (hold/confirm/release) that hides the mutex, holds map, and rollback machinery from callers. |
| **Concurrency control (CAS + lock)** | `Mutex.runExclusive` around every check-and-set | Not a GoF pattern but the true heart: it makes the state transition atomic. |

**Connect this to Ticketmaster (110).** Topic 110 asks the *same question at the system level*: thousands of nodes, a shared database, a virtual queue, distributed locks. This doc asks it at the *class level*: one process, an in-memory mutex, one `ShowSeat` object. The mental model transfers exactly — the in-process `Mutex` here **is** the `SELECT ... FOR UPDATE` row lock or Redis lease (84) there. If you can explain why `holdSeats` must be atomic in this file, you can explain 110's locking design; they are the same idea at two zoom levels.

---

## Visual / Diagram description

Sequence of the winning and losing hold, then confirmation:

```
 Alice            BookingService (mutex on show1)         Bob
   │  holdSeats(A5,A6) │                                   │
   │──────────────────►│  LOCK ──────────────┐            │
   │                   │  A5 AVAILABLE→HELD   │            │
   │                   │  A6 AVAILABLE→HELD   │            │
   │◄── ok, hold_1 ────│  UNLOCK ─────────────┘            │
   │                   │            holdSeats(A5,A7)        │
   │                   │◄──────────────────────────────────│
   │                   │  LOCK ──────────────┐             │
   │                   │  A5 is HELD → REJECT │             │
   │                   │  rollback A7 (none)  │             │
   │                   │  UNLOCK ─────────────┘             │
   │                   │── ok:false, "A5 not available" ──►│
   │  confirm(hold_1)  │                                   │
   │──────────────────►│  LOCK: A5,A6 HELD→BOOKED, UNLOCK  │
   │◄── CONFIRMED ─────│                                   │
```

Every mutation of seat state happens *inside* a LOCK/UNLOCK pair on that show. Because only one caller can be inside the lock at a time, the "check AVAILABLE then set HELD" pair can never interleave — the exact bug the naive version had.

---

## Real world examples

### BookMyShow (India)

Representative: BookMyShow shows a live seat map and, once you pick seats, starts a countdown timer (commonly a few minutes) during which the seats are yours to pay for. That visible countdown is the **hold TTL** in action — the UI surfacing exactly the `holdExpiresAt` mechanism in step 6. If the timer runs out, the seats rejoin the pool.

### Ticketmaster (concerts/events)

Ticketmaster's high-demand on-sales pair the hold with a **virtual waiting room** (the system-level design in topic 110). The waiting room throttles how many users reach the seat-selection stage at once, which reduces contention on any single seat before the hold/lock even engages. Same seat-hold core, plus an admission-control layer in front.

### Airline seat maps (representative)

Airlines model a seat's availability per *flight-leg* the way we model it per *show* — the physical seat 14C exists on the aircraft, but its availability is tracked separately for each dated flight. That per-context availability object is the industry's `ShowSeat`.

---

## Trade-offs

**Hold TTL length:**

| Short TTL (e.g. 2 min) | Long TTL (e.g. 15 min) |
|---|---|
| Seats recycle fast; higher effective availability | Users rarely lose seats mid-payment |
| Users on slow connections lose seats mid-pay | Popular shows look "sold out" while holds sit idle |

**Lock granularity:**

| Per-show lock (used here) | Per-seat lock |
|---|---|
| Simple; one mutex per show | Max concurrency; two users on different seats never wait |
| Two users on *different* seats of one show serialise briefly | More locks to manage; deadlock risk if not ordered |

**In-memory state vs database:**

| In-memory (this doc) | Database rows |
|---|---|
| Fast, simple, great for teaching | Survives restarts; scales to many nodes |
| Lost on crash; single node only | Needs `FOR UPDATE` / conditional updates / a distributed lock (84) |

**The sweet spot:** a **per-show lock (or per-seat DB row lock)** guarding an **atomic AVAILABLE→HELD transition**, plus a **short hold TTL (~5 min)** with a **background expiry sweep**, plus **idempotent confirmation**. That combination prevents double-booking without killing throughput.

---

## Common interview questions on this topic

### Q1: "Why not just put a `status` field on `Seat`?"

**Hint:** Because a physical seat is shared across every show on its screen. A global status would mean booking A5 for the 7 PM show also books it for the matinee. You need a per-show availability object — `ShowSeat` — linking `Seat` + `Show` + `status`.

### Q2: "Two users click the same seat at the same time. Walk me through preventing a double-book."

**Hint:** The bug is the gap between *check* (is it AVAILABLE?) and *set* (mark HELD). Make them one atomic step: a mutex/lock in-process, or a conditional `UPDATE ... WHERE status='AVAILABLE'` / `SELECT FOR UPDATE` / distributed lock (84) in a real system. Exactly one caller's CAS succeeds.

### Q3: "Why hold seats instead of booking immediately?"

**Hint:** Payment takes time and can fail. A hold with a TTL gives the user a private window to pay while blocking competitors, and auto-releases the seat if they abandon — decoupling payment latency from seat contention.

### Q4: "What happens if `confirmBooking` is called twice?"

**Hint:** It must be idempotent. Key the booking by `holdId`; on the second call, detect the existing CONFIRMED booking and return it unchanged — no second charge, no second state transition.

### Q5: "How does this design change for a distributed, multi-node deployment?"

**Hint:** The in-memory `Mutex` becomes a **DB row lock** or a **distributed lock** (84 — Redis lease/Redlock); `ShowSeat` state moves to a database; the expiry sweep becomes a scheduled job or a TTL/queue. The *object model and lifecycle are unchanged* — only the lock's implementation moves out of process. This is the class-level (118) vs system-level (110) distinction.

---

## Practice exercise

### Build and race the booking service (~30 min)

1. Copy the step-6 code into `movie-booking.js` and run it. Confirm Bob's race is rejected and Alice's retry is idempotent.
2. **Add a `cancelBooking(user, bookingId)` method** that flips a CONFIRMED booking to CANCELLED and returns its `ShowSeat`s to AVAILABLE. Make it idempotent.
3. **Simulate a true race:** launch `holdSeats` for A5 from Alice and Bob with `Promise.all([...])` *without awaiting individually*. Verify exactly one wins every run. Then temporarily replace `runExclusive` with a plain function call (no lock) and observe the double-book appear — proving the mutex is load-bearing.
4. **Produce:** the output logs from step 3 with and without the lock, and a one-paragraph explanation of why removing the lock breaks correctness.

---

## Quick reference cheat sheet

- **Seat vs ShowSeat:** `Seat` is physical (defined once per screen); `ShowSeat` is its status *for one show*. This split is the whole design.
- **Three-state lifecycle:** AVAILABLE → HELD → BOOKED; plus HELD → AVAILABLE on expiry/release. Never AVAILABLE → BOOKED directly.
- **The race:** the gap between *check available* and *set held* lets two writers both win. That gap is the bug.
- **The fix:** make check-and-set **atomic** — mutex (in-process), row lock / conditional UPDATE (DB), or distributed lock (distributed).
- **Hold TTL:** a hold carries `holdExpiresAt`; unpaid holds auto-release, decoupling payment time from contention.
- **All-or-nothing:** if any seat in a multi-seat request is unavailable, roll back the ones already grabbed.
- **Idempotent confirm:** key by `holdId`; a repeat call returns the same booking, never double-charges.
- **Expiry sweep:** `releaseExpiredHolds()` reclaims lapsed holds; can be a cron/queue job at scale.
- **Strategy pattern:** pricing by seat type, swappable (surge pricing later).
- **State pattern:** legal transitions encoded so illegal ones are unreachable.
- **110 vs 118:** same problem, system level vs class level. In-memory `Mutex` here ≈ distributed lock / row lock there.
- **Lock granularity:** per-show is simple; per-seat maximises concurrency.

---

## Connected topics

| Direction | Topic | Why |
|---|---|---|
| **Previous** | [117 — LLD Hotel Booking](./117-lld-hotel-booking.md) | The same reserve-with-a-hold pattern for rooms; ShowSeat here mirrors per-date room availability |
| **Next** | [111 — LLD Approach Framework](./111-lld-approach-framework.md) | The general 7-step method this doc applies end to end |
| **Related** | [110 — HLD Ticketmaster](./110-hld-ticket-master.md) | The system-level twin: the identical seat-contention problem at scale |
| **Related** | [52 — Concurrency Fundamentals](./52-concurrency-fundamentals.md) | The async mutex that makes the AVAILABLE→HELD transition atomic |
| **Related** | [84 — Distributed Locking](./84-distributed-locking.md) | What the in-memory lock becomes across many nodes |
