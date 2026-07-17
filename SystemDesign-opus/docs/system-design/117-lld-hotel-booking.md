# 117 — Design a Hotel Booking System
## Category: LLD Case Study

---

## What is this?

A hotel booking system lets a guest search for available rooms over a date range, reserve one, and later check in and out — while making sure two guests can never book the **same room for overlapping dates**. Think of it like a physical wall calendar pinned behind the front desk: each room has its own row, and you shade in the nights that are already taken. A room is "available" only if the strip of nights you want is still blank.

This is a classic low-level design (LLD) interview problem because the "availability" is not a simple `isBooked = true/false` flag — it is a function of **time intervals**, and getting the interval-overlap check right is the whole game.

---

## Why does it matter?

**Interview angle.** Interviewers love this problem because it separates people who model "availability = boolean" (wrong) from people who realise availability is a **date-range** question. If you model it as a boolean, you cannot answer "is room 101 free next Tuesday to Thursday when it's booked this weekend?" The second thing they test is the **double-booking race**: two requests for the same room and dates arriving at the same instant. Both see "available", both write a reservation, and now one physical room has two angry guests. This is the exact same class of bug as double-selling a concert seat (recall `110 — HLD Ticketmaster`).

**Real-work angle.** If your overlap logic is off by one boundary (`<` vs `<=`), you either block back-to-back stays that should be allowed (checkout Tuesday, check-in Tuesday is fine) or you allow a true overlap. Both cost real money and real trust. Every reservation platform — hotels, meeting rooms, car rentals, restaurant tables — is this same interval-reservation shape underneath.

---

## The core idea — explained simply

### The wall-calendar analogy

Picture a whiteboard behind the front desk. Down the left edge is one row per room: 101, 102, 201… Across the top are the dates. For every confirmed reservation, the clerk draws a shaded bar from the check-in date to the check-out date.

When a guest walks up and asks "a Deluxe room from July 20 to July 23," the clerk:

1. Looks only at the Deluxe rows.
2. For each Deluxe room, checks whether the guest's requested bar `[Jul 20, Jul 23)` **overlaps any existing shaded bar** in that row.
3. Any room whose row has no overlapping bar in that window is offered as available.

That's the entire system. Everything else — pricing, cancellation, check-in — hangs off this one interval-overlap idea.

| Whiteboard thing | Technical concept |
| --- | --- |
| One row per room | A `Room` object |
| A shaded bar | A `Reservation` with `checkIn`/`checkOut` |
| "Is this window blank?" | The **overlap check** across confirmed reservations |
| Rows filtered to Deluxe | `search(checkIn, checkOut, roomType)` |
| Clerk redrawing after a cancel | Reservation status flips to `CANCELLED`, freeing the window |
| Nightly price on a tag | `PricingStrategy.calculatePrice(room, range)` |
| "No-show fee" sign at the desk | `CancellationPolicy` |

The single most important sub-idea: **two date ranges `[a, b)` and `[c, d)` overlap if and only if `a < d AND c < b`.** Memorise that. We treat check-out as *exclusive* (the room is free again on the checkout date), which is why we use `<` and not `<=` — a stay ending July 23 and another starting July 23 do **not** overlap.

---

## Key concepts inside this topic

We follow the standard 7-step LLD method: requirements → objects (nouns) → behaviours (verbs) → class diagram → code → patterns → extensions.

### 1. Requirements & clarifying questions

**Functional requirements**
- Search available rooms by **date range** and **room type**.
- A room has a **type** (Standard / Deluxe / Suite) and a **nightly rate**.
- Book a room for a date range `[checkIn, checkOut)`.
- **Prevent double-booking** of the same room for overlapping dates.
- Cancel a booking, applying a **cancellation policy**.
- Check-in and check-out a guest.
- **Pricing can vary** (seasonal, weekend) — the nightly rate is not fixed.

**Non-functional**
- Correctness under concurrency (no double-book) is more important than raw speed.
- Availability check should be cheap for a single hotel (linear scan over a room's reservations is fine in-memory; a real system indexes by date).

**Clarifying questions to ask the interviewer** (asking these signals seniority):
- One hotel or **multiple hotels / a chain**? (We build for one, but keep `Hotel` as the aggregate root so a `HotelChain` can hold many.)
- Are **group bookings** (N rooms in one reservation) in scope? (We do one room per reservation; note the extension.)
- Is **overbooking** allowed (airlines do it deliberately)? (Default no; show it as a policy extension.)
- Is **payment** in scope? (We stub pricing but leave a payment gateway out.)
- Do we need a temporary **hold** during checkout like Ticketmaster's seat lock? (Note as extension #4.)

### 2. Identify core objects (nouns)

Read the requirements and underline the nouns:

- **`Hotel`** — the aggregate root; owns rooms and all reservations; the orchestrator (also called `HotelService`).
- **`Room`** — has an id, a `RoomType`, and a base nightly rate.
- **`RoomType`** — an enum: `STANDARD`, `DELUXE`, `SUITE`.
- **`Guest`** — id, name, email.
- **`Reservation`** (a.k.a. `Booking`) — ties a guest to a room for `[checkIn, checkOut)` with a `ReservationStatus`. Owns the crucial `overlaps()` method.
- **`ReservationStatus`** — enum: `CONFIRMED`, `CHECKED_IN`, `CHECKED_OUT`, `CANCELLED`.
- **`PricingStrategy`** — computes the price for a room over a range (Strategy pattern).
- **`CancellationPolicy`** — computes the refund/penalty on cancel (Strategy pattern).
- **`SearchService`** — logic to filter rooms by availability (we fold this into `Hotel.search` for a single hotel).

### 3. Behaviours (verbs) + who owns them

| Behaviour | Owner | Notes |
| --- | --- | --- |
| `overlaps(checkIn, checkOut)` | `Reservation` | The interval math lives here — one place, tested once. |
| `search(checkIn, checkOut, type)` | `Hotel` | Returns rooms with **no overlapping active reservation**. |
| `book(guest, room, checkIn, checkOut)` | `Hotel` | **Re-validates availability atomically**, then creates a `CONFIRMED` reservation. |
| `cancel(reservation)` | `Hotel` | Flips status to `CANCELLED`, asks the policy for the penalty. |
| `checkIn(reservation)` / `checkOut(reservation)` | `Hotel` | Status transitions `CONFIRMED → CHECKED_IN → CHECKED_OUT`. |
| `calculatePrice(room, range)` | `PricingStrategy` | Nightly rate can vary by date/season. |

**The crux — availability is a function of date ranges, not a boolean.** A room is available for `[checkIn, checkOut)` only if **no** reservation on that room with an active status (`CONFIRMED` or `CHECKED_IN`) overlaps the range.

The overlap rule, stated precisely:

> Ranges `[a, b)` and `[c, d)` overlap ⟺ `a < d AND c < b`.

Why this works: `a < d` says "my stay starts before yours ends," and `c < b` says "your stay starts before mine ends." Both must hold for the bars to touch. If either fails, one stay is entirely before the other. Because check-out is exclusive, back-to-back stays (`b == c`) give `c < b` → `false`, so they correctly do **not** overlap.

```
        a=Jul20            b=Jul23
  Mine:  |==================|
  Yours:            |==================|
                  c=Jul22            d=Jul25
  a<d? Jul20<Jul25 yes.  c<b? Jul22<Jul23 yes.  => OVERLAP (reject)

  Back-to-back:
  Mine:  |=========|
  Yours:           |=========|
       a       b=c=Jul23     d
  a<d? yes.  c<b? Jul23<Jul23 NO. => no overlap (allowed)
```

### 4. Class diagram (ASCII)

```
                 ┌─────────────────────────────┐
                 │            Hotel             │  (aggregate root / orchestrator)
                 │  - name                      │
                 │  - rooms: Room[]             │
                 │  - reservations: Reservation[]│
                 │  - pricing: PricingStrategy  │◇──────────┐ uses
                 │  + search(in,out,type)       │           │
                 │  + book(guest,room,in,out)   │           ▼
                 │  + cancel(res)               │   ┌───────────────────┐
                 │  + checkIn/checkOut(res)     │   │  PricingStrategy   │ «interface»
                 └───────┬─────────────┬────────┘   │ + calculatePrice() │
             composition │             │            └─────────┬─────────┘
                 (◇ owns)│             │ 1..*                 │ implements
                         ▼             ▼           ┌──────────┼───────────┐
                 ┌──────────────┐ ┌──────────────────┐  ┌──────────┐ ┌──────────┐
                 │     Room     │ │   Reservation    │  │Standard  │ │Seasonal/ │
                 │ - id         │ │ - id             │  │Pricing   │ │Weekend   │
                 │ - type ──────┼─│ - room ──────────┤  └──────────┘ └──────────┘
                 │ - rate       │ │ - guest ─────┐   │
                 └──────────────┘ │ - checkIn    │   │      ┌────────────────────┐
                   type: RoomType │ - checkOut   │   │      │ CancellationPolicy │ «interface»
                   (enum)         │ - status ────┼───┤      │ + penalty(res)     │
                                  │ + overlaps() │   │      └─────────┬──────────┘
                                  └──────────────┘   │                │ implements
                                    status:          │      ┌─────────┼──────────┐
                                    ReservationStatus▼   ┌────────┐┌────────┐┌──────────┐
                                       ┌────────┐        │Free    ││OneNight││NonRefund │
                                       │ Guest  │        │Cancel  ││Penalty ││able      │
                                       └────────┘        └────────┘└────────┘└──────────┘
```

- `Hotel ◇— Room` and `Hotel ◇— Reservation`: **composition** (the diamond ◇). Rooms and reservations don't exist without the hotel.
- `Room 1 —* Reservation`: one room has many reservations over time (each a shaded bar on its calendar row).
- `Reservation *— 1 Guest`: many reservations point to one guest.
- `PricingStrategy` and `CancellationPolicy` are **interfaces** with swappable concrete implementations — this is the Strategy pattern (recall `42 — Strategy`).

### 5. Full JavaScript implementation

Complete and runnable. Save as `hotel.js` and run `node hotel.js`.

```javascript
'use strict';

// ── Enums (frozen so nobody can mutate them) ───────────────────────────────
const RoomType = Object.freeze({
  STANDARD: 'STANDARD',
  DELUXE: 'DELUXE',
  SUITE: 'SUITE',
});

const ReservationStatus = Object.freeze({
  CONFIRMED: 'CONFIRMED',
  CHECKED_IN: 'CHECKED_IN',
  CHECKED_OUT: 'CHECKED_OUT',
  CANCELLED: 'CANCELLED',
});

// A status is "active" (blocks the room) only when confirmed or the guest is in.
const ACTIVE_STATUSES = new Set([
  ReservationStatus.CONFIRMED,
  ReservationStatus.CHECKED_IN,
]);

// ── Small date helper: whole nights between two Date objects ────────────────
const MS_PER_NIGHT = 24 * 60 * 60 * 1000;
function nightsBetween(checkIn, checkOut) {
  // We treat checkOut as exclusive: [checkIn, checkOut). 20th->23rd = 3 nights.
  return Math.round((checkOut.getTime() - checkIn.getTime()) / MS_PER_NIGHT);
}
function d(iso) { return new Date(iso + 'T00:00:00Z'); } // tiny date literal

// ── Core entities ───────────────────────────────────────────────────────────
class Guest {
  constructor(id, name, email) {
    this.id = id;
    this.name = name;
    this.email = email;
  }
}

class Room {
  constructor(id, type, baseRate) {
    this.id = id;           // e.g. "101"
    this.type = type;       // RoomType
    this.baseRate = baseRate; // nightly base price in dollars
  }
}

class Reservation {
  constructor(id, room, guest, checkIn, checkOut, price) {
    this.id = id;
    this.room = room;
    this.guest = guest;
    this.checkIn = checkIn;   // Date, inclusive
    this.checkOut = checkOut; // Date, EXCLUSIVE (room free again this day)
    this.price = price;
    this.status = ReservationStatus.CONFIRMED;
  }

  isActive() {
    return ACTIVE_STATUSES.has(this.status);
  }

  // THE HEART OF THE PROBLEM.
  // Two ranges [a,b) and [c,d) overlap  <=>  a < d  AND  c < b.
  // checkOut is exclusive, so back-to-back stays (b == c) do NOT overlap.
  overlaps(checkIn, checkOut) {
    return this.checkIn < checkOut && checkIn < this.checkOut;
  }
}

// ── Strategy pattern #1: pricing (recall 42 — Strategy) ─────────────────────
// A PricingStrategy computes total price for a room over a date range.
// Swapping the strategy changes pricing WITHOUT touching Hotel or Room.
class PricingStrategy {
  calculatePrice(room, checkIn, checkOut) {
    throw new Error('PricingStrategy.calculatePrice must be implemented');
  }
}

class StandardPricing extends PricingStrategy {
  calculatePrice(room, checkIn, checkOut) {
    return room.baseRate * nightsBetween(checkIn, checkOut);
  }
}

class SeasonalPricing extends PricingStrategy {
  // Peak months get a multiplier; everything else is base.
  constructor(peakMonths = [5, 6, 7], multiplier = 1.5) { // Jun,Jul,Aug (0-idx)
    super();
    this.peakMonths = new Set(peakMonths);
    this.multiplier = multiplier;
  }
  calculatePrice(room, checkIn, checkOut) {
    let total = 0;
    const cursor = new Date(checkIn.getTime());
    while (cursor < checkOut) {
      const isPeak = this.peakMonths.has(cursor.getUTCMonth());
      total += room.baseRate * (isPeak ? this.multiplier : 1);
      cursor.setUTCDate(cursor.getUTCDate() + 1);
    }
    return total;
  }
}

class WeekendPricing extends PricingStrategy {
  // Fri (5) and Sat (6) nights cost more.
  constructor(weekendMultiplier = 1.25) {
    super();
    this.weekendMultiplier = weekendMultiplier;
  }
  calculatePrice(room, checkIn, checkOut) {
    let total = 0;
    const cursor = new Date(checkIn.getTime());
    while (cursor < checkOut) {
      const day = cursor.getUTCDay();
      const isWeekend = day === 5 || day === 6;
      total += room.baseRate * (isWeekend ? this.weekendMultiplier : 1);
      cursor.setUTCDate(cursor.getUTCDate() + 1);
    }
    return total;
  }
}

// ── Strategy pattern #2: cancellation policy ────────────────────────────────
// Returns the penalty (amount NOT refunded) for cancelling a reservation.
class CancellationPolicy {
  penalty(reservation) {
    throw new Error('CancellationPolicy.penalty must be implemented');
  }
}
class FreeCancellation extends CancellationPolicy {
  penalty(_res) { return 0; } // full refund
}
class OneNightPenalty extends CancellationPolicy {
  penalty(res) {
    return res.room.baseRate; // keep one night's base rate
  }
}
class NonRefundable extends CancellationPolicy {
  penalty(res) { return res.price; } // keep everything
}

// ── The orchestrator ────────────────────────────────────────────────────────
class Hotel {
  constructor(name, pricing = new StandardPricing(),
              cancellationPolicy = new FreeCancellation()) {
    this.name = name;
    this.rooms = [];
    this.reservations = [];
    this.pricing = pricing;
    this.cancellationPolicy = cancellationPolicy;
    this._nextResId = 1;
    // A crude in-process lock to model the atomic critical section.
    // In a real system this is a DB transaction / row lock (recall 84).
    this._booking = false;
  }

  addRoom(room) { this.rooms.push(room); return room; }

  // Return every active reservation currently held against a given room.
  _activeReservationsFor(room) {
    return this.reservations.filter(r => r.room.id === room.id && r.isActive());
  }

  // Is this specific room free for [checkIn, checkOut)?  <-- the overlap filter
  _isRoomAvailable(room, checkIn, checkOut) {
    return this._activeReservationsFor(room)
      .every(r => !r.overlaps(checkIn, checkOut));
  }

  // SEARCH: rooms of the requested type with NO overlapping active reservation.
  search(checkIn, checkOut, roomType = null) {
    return this.rooms.filter(room => {
      if (roomType && room.type !== roomType) return false;
      return this._isRoomAvailable(room, checkIn, checkOut);
    });
  }

  // BOOK: the concurrency-sensitive path.
  // Between a user's search and their booking, someone else may have grabbed
  // the room (recall 52 — concurrency; same double-sell risk as 110/118).
  // So we RE-VALIDATE availability inside a critical section, atomically.
  book(guest, room, checkIn, checkOut) {
    if (checkIn >= checkOut) throw new Error('checkOut must be after checkIn');

    // --- begin critical section (atomic in a real DB via a transaction) ---
    if (this._booking) throw new Error('booking busy; retry'); // guard re-entry
    this._booking = true;
    try {
      // The all-important re-check: do NOT trust the earlier search result.
      if (!this._isRoomAvailable(room, checkIn, checkOut)) {
        throw new Error(
          `Room ${room.id} is not available for ` +
          `${checkIn.toISOString().slice(0, 10)} -> ` +
          `${checkOut.toISOString().slice(0, 10)} (would double-book)`
        );
      }
      const price = this.pricing.calculatePrice(room, checkIn, checkOut);
      const res = new Reservation(
        `R${this._nextResId++}`, room, guest, checkIn, checkOut, price
      );
      this.reservations.push(res);
      return res;
    } finally {
      this._booking = false;
    }
    // --- end critical section ---
  }

  cancel(reservation) {
    if (reservation.status === ReservationStatus.CANCELLED) return 0;
    if (reservation.status === ReservationStatus.CHECKED_OUT) {
      throw new Error('cannot cancel a completed stay');
    }
    const penalty = this.cancellationPolicy.penalty(reservation);
    const refund = reservation.price - penalty;
    reservation.status = ReservationStatus.CANCELLED; // frees the date window
    return refund;
  }

  checkIn(reservation) {
    if (reservation.status !== ReservationStatus.CONFIRMED) {
      throw new Error(`cannot check in from status ${reservation.status}`);
    }
    reservation.status = ReservationStatus.CHECKED_IN;
  }

  checkOut(reservation) {
    if (reservation.status !== ReservationStatus.CHECKED_IN) {
      throw new Error(`cannot check out from status ${reservation.status}`);
    }
    reservation.status = ReservationStatus.CHECKED_OUT; // room reusable again
  }
}

// ── Demo ────────────────────────────────────────────────────────────────────
function main() {
  const hotel = new Hotel('Grand Interval Hotel',
    new SeasonalPricing(),      // July is peak -> 1.5x
    new OneNightPenalty());     // cancel = keep one night

  // Set up rooms.
  const std1 = hotel.addRoom(new Room('101', RoomType.STANDARD, 100));
  const std2 = hotel.addRoom(new Room('102', RoomType.STANDARD, 100));
  hotel.addRoom(new Room('201', RoomType.DELUXE, 180));

  const alice = new Guest('G1', 'Alice', 'alice@example.com');
  const bob   = new Guest('G2', 'Bob',   'bob@example.com');
  const carol = new Guest('G3', 'Carol', 'carol@example.com');

  const jul20 = d('2026-07-20');
  const jul23 = d('2026-07-23');
  const jul22 = d('2026-07-22');
  const jul25 = d('2026-07-25');
  const jul23b = d('2026-07-23'); // back-to-back start

  console.log('--- 1. Search STANDARD, Jul 20 -> Jul 23 ---');
  let available = hotel.search(jul20, jul23, RoomType.STANDARD);
  console.log('available:', available.map(r => r.id)); // [101, 102]

  console.log('\n--- 2. Alice books room 101, Jul 20 -> Jul 23 ---');
  const aliceRes = hotel.book(alice, std1, jul20, jul23);
  console.log(`booked ${aliceRes.id} room ${aliceRes.room.id} ` +
    `price $${aliceRes.price} (3 peak nights @ 100*1.5)`); // $450

  console.log('\n--- 3. Bob tries to DOUBLE-BOOK room 101, Jul 22 -> Jul 25 ---');
  try {
    hotel.book(bob, std1, jul22, jul25); // overlaps Alice -> must reject
  } catch (e) {
    console.log('rejected:', e.message);
  }

  console.log('\n--- 4. Search STANDARD again, Jul 20 -> Jul 23 ---');
  available = hotel.search(jul20, jul23, RoomType.STANDARD);
  console.log('available:', available.map(r => r.id)); // [102] only

  console.log('\n--- 5. Back-to-back is allowed: Carol books 101, Jul 23 -> Jul 25 ---');
  const carolRes = hotel.book(carol, std1, jul23b, jul25); // b==c, no overlap
  console.log(`booked ${carolRes.id} room ${carolRes.room.id} ` +
    `Jul 23 -> Jul 25 price $${carolRes.price}`);

  console.log('\n--- 6. Alice cancels (one-night penalty) ---');
  const refund = hotel.cancel(aliceRes);
  console.log(`refund $${refund} (paid $${aliceRes.price}, kept $100)`);

  console.log('\n--- 7. Re-search STANDARD, Jul 20 -> Jul 23 ---');
  available = hotel.search(jul20, jul23, RoomType.STANDARD);
  console.log('available:', available.map(r => r.id)); // [101, 102] again

  console.log('\n--- 8. Check-in / check-out flow for Carol ---');
  hotel.checkIn(carolRes);
  console.log('status:', carolRes.status);   // CHECKED_IN
  hotel.checkOut(carolRes);
  console.log('status:', carolRes.status);   // CHECKED_OUT
}

main();
```

Expected output (abridged):

```
--- 1. Search STANDARD, Jul 20 -> Jul 23 ---
available: [ '101', '102' ]
--- 2. Alice books room 101, Jul 20 -> Jul 23 ---
booked R1 room 101 price $450 (3 peak nights @ 100*1.5)
--- 3. Bob tries to DOUBLE-BOOK room 101, Jul 22 -> Jul 25 ---
rejected: Room 101 is not available for 2026-07-22 -> 2026-07-25 (would double-book)
--- 4. Search STANDARD again, Jul 20 -> Jul 23 ---
available: [ '102' ]
--- 5. Back-to-back is allowed: Carol books 101, Jul 23 -> Jul 25 ---
booked R2 room 101 Jul 23 -> Jul 25 price $360
--- 6. Alice cancels (one-night penalty) ---
refund $350 (paid $450, kept $100)
--- 7. Re-search STANDARD, Jul 20 -> Jul 23 ---
available: [ '101', '102' ]
```

Notice step 3 (double-book rejected), step 4 (fewer rooms after booking), step 5 (back-to-back allowed — the `<` vs `<=` detail), and step 7 (cancel frees the window). That is the availability logic proving itself.

### 6. Design patterns used and WHY

- **Strategy (the star), twice.** `PricingStrategy` lets the nightly rate vary — standard, seasonal, weekend — without the `Hotel` knowing which rule is in force; you inject the strategy at construction. `CancellationPolicy` does the same for refunds. This is textbook `42 — Strategy`: encapsulate a family of interchangeable algorithms behind one interface. Adding "loyalty discount pricing" later means writing one new class, changing nothing else.
- **Aggregate root.** `Hotel` owns rooms and reservations and is the only entry point that mutates them, so all invariants (no double-book) are enforced in one place.
- **Concurrency / consistency — the Ticketmaster parallel.** The `book()` method wraps a **re-check-then-write** in a critical section (`this._booking` flag standing in for a DB transaction / row lock). This is deliberately the same shape as seat reservation in `110 — HLD Ticketmaster` and `118 — Movie Ticket Booking`: the availability you showed the user is stale the instant you show it, so you must re-validate atomically at commit time (recall `52 — Concurrency Fundamentals`). Without this, two overlapping `book()` calls both pass their overlap check and both insert — a double-book. In production the lock is a `SELECT ... FOR UPDATE`, an optimistic version column, or a unique constraint on `(room_id, date)`.

### 7. Extensions the interviewer asks for

| "Now add…" | How the design absorbs it |
| --- | --- |
| **Multiple hotels / a chain** | Add a `HotelChain` holding many `Hotel` aggregates; search fans out across hotels and merges results. `Hotel` already owns its own rooms/reservations, so nothing inside it changes. |
| **Overbooking with a policy** | Give `Hotel` an `OverbookingPolicy` strategy that allows up to N confirmed reservations beyond physical capacity per type/date. The overlap filter counts overlaps instead of forbidding them; capacity check moves into the policy. |
| **Room upgrades** | On check-in, `checkIn()` can consult an availability search for a higher `RoomType` and reassign `reservation.room`. The overlap logic is unchanged — it works per room regardless. |
| **Hold / TTL during checkout (like Ticketmaster)** | Introduce a `HELD` status with an `expiresAt`; `search`/`_isRoomAvailable` treat non-expired holds as blocking. A sweeper releases expired holds. Same TTL-lock idea as the seat hold in `110`. |
| **Loyalty discounts** | Just another `PricingStrategy` (e.g. `LoyaltyPricing` wrapping a base strategy and applying a member %). Strategy makes this a pure add — proof the pattern earned its place. |

---

## Visual / Diagram description

Booking sequence, showing the atomic re-check that prevents the double-book:

```
 Guest            Hotel (orchestrator)          Reservations store
   │  search(in,out,type)  │                            │
   │──────────────────────▶│  filter rooms by overlap   │
   │                       │───────────────────────────▶│
   │   [rooms available]   │◀───────────────────────────│
   │◀──────────────────────│                            │
   │                       │                            │
   │  book(guest,room,in,out)                           │
   │──────────────────────▶│                            │
   │                       │┌── CRITICAL SECTION ──────┐│
   │                       ││ re-check overlap (again!) ││
   │                       ││   available?              ││
   │                       ││   yes -> insert CONFIRMED ││───▶ new Reservation
   │                       ││   no  -> throw (reject)   ││
   │                       │└──────────────────────────┘│
   │   reservation / error │                            │
   │◀──────────────────────│                            │
```

The dashed critical-section box is the whole point: the overlap check that ran during `search` is advisory only; the check that runs inside `book`, holding the lock, is authoritative. Two concurrent `book` calls serialise through that box, so the second one sees the first one's insert and is rejected.

---

## Real world examples

### Booking.com / Expedia (representative)
Large OTAs (online travel agencies) model availability as inventory per `(property, room_type, date)` with a count of remaining rooms per night, not a single boolean. A multi-night booking decrements every night in the range. This is the same interval idea, indexed by date for scale, and they lean heavily on atomic decrements plus overbooking policies negotiated with hotels.

### Airbnb (representative)
Each listing has a calendar of blocked/available nights; a booking request checks the requested nights against that calendar — literally the wall-calendar analogy. Airbnb also implements request holds and instant-book, mirroring the hold/TTL extension above.

### OpenTable / restaurant reservations (representative)
Same interval-reservation shape with much shorter ranges (a 2-hour slot instead of nights) and tables instead of rooms. The overlap check and the concurrency-safe commit are identical in spirit — proof this LLD generalises far beyond hotels.

---

## Trade-offs

| Modelling choice | Pros | Cons |
| --- | --- | --- |
| Availability as date-range overlap | Correct for multi-night stays; no per-night rows needed in-memory | Overlap scan is O(reservations per room); needs a date index at scale |
| Boolean `isBooked` flag (tempting, wrong) | Trivial | Cannot answer future/date-specific availability; breaks immediately |
| Per-night inventory rows | O(1) lookup per night; easy overbooking counts | Many rows; a 30-night stay touches 30 rows in a transaction |

| Concurrency approach | Pros | Cons |
| --- | --- | --- |
| Pessimistic lock (row / `FOR UPDATE`) | Simple, strongly correct | Reduces throughput; risk of lock contention on hot rooms |
| Optimistic (version column, retry) | High throughput when conflicts are rare | Wasted work + retries under high contention |
| Unique constraint on `(room, date)` | Database guarantees no double-book | Requires per-night rows; error handling on conflict |

**The sweet spot:** model availability as **date-range overlap**, keep the overlap check in one `Reservation.overlaps()` method, and make the booking commit **atomic** (transaction + row lock, or a unique per-night constraint). Correctness first; optimise the availability lookup with a date index only when the room count grows.

---

## Common interview questions on this topic

### Q1: "How do you know if a room is available for a date range?"
**Hint:** Not a boolean. A room is available for `[in, out)` iff **no** active reservation on it overlaps that range. State the rule: `[a,b)` and `[c,d)` overlap ⟺ `a < d AND c < b`. Mention that check-out is exclusive so back-to-back stays are allowed.

### Q2: "Two guests book the same room for the same nights at the same instant — what happens?"
**Hint:** The double-booking race. The availability from `search` is stale. You must **re-validate inside a critical section** (DB transaction with a row lock, an optimistic version check, or a unique `(room, date)` constraint) so the two `book` calls serialise; the loser is rejected. Same problem as seat reservation in Ticketmaster (110).

### Q3: "Why is check-out exclusive? Show the boundary case."
**Hint:** So a room freed at checkout can be re-let that same day. Range `[Jul20, Jul23)` and `[Jul23, Jul25)`: `c < b` is `Jul23 < Jul23` = false → no overlap → allowed. Using `<=` would wrongly block back-to-back stays.

### Q4: "How do you support seasonal / weekend pricing without rewriting the booker?"
**Hint:** Strategy pattern. Inject a `PricingStrategy`; iterate the nights and apply a multiplier for peak months or weekends. `Hotel.book` just calls `pricing.calculatePrice(...)` and never learns the rule.

### Q5: "Now add a temporary hold during checkout so the room isn't grabbed while the user pays."
**Hint:** Add a `HELD` status with `expiresAt`; the availability filter treats non-expired holds as blocking, and a background sweeper releases expired ones. This is the TTL-lock idea from Ticketmaster.

---

## Practice exercise

**Build and test the overlap engine (25-35 min).**
Starting from the code above, do the following:
1. Write a standalone `rangesOverlap(a, b, c, d)` function and unit-test it against these cases and predict each before running: identical ranges (overlap), back-to-back `b==c` (no overlap), one fully inside the other (overlap), fully disjoint (no overlap), and a single shared instant at the boundary (no overlap).
2. Add a `SUITE` room and a `WeekendPricing` strategy; book a Fri-Sun stay and confirm the weekend nights cost more.
3. Simulate the race: call `hotel.book(...)` for the same room and overlapping dates twice in a row and confirm the second throws. Then swap `OneNightPenalty` for `NonRefundable` and confirm the refund becomes `$0`.

Produce: the `rangesOverlap` function, its test output, and a one-paragraph note on where the atomic re-check must live in a real database.

---

## Quick reference cheat sheet

- **Availability is a date-range question, not a boolean.** This is the #1 thing the problem tests.
- **Overlap rule:** `[a,b)` and `[c,d)` overlap ⟺ `a < d AND c < b`. Memorise it.
- **Check-out is exclusive** → back-to-back stays (`b == c`) are allowed; use `<`, not `<=`.
- **`Reservation.overlaps()`** holds the interval math — one method, tested once.
- **A room is available** iff no *active* (`CONFIRMED`/`CHECKED_IN`) reservation overlaps the range.
- **Double-booking = a race.** Re-validate availability **atomically** in `book()` (transaction + row lock / unique constraint) — same as Ticketmaster seat reservation.
- **Search result is stale** the moment it's returned; never trust it at commit time.
- **PricingStrategy** = Strategy pattern → standard / seasonal / weekend / loyalty are swappable classes.
- **CancellationPolicy** = Strategy pattern → free / one-night / non-refundable.
- **Status flow:** `CONFIRMED → CHECKED_IN → CHECKED_OUT`, or `CONFIRMED → CANCELLED` (frees the window).
- **`Hotel`** is the aggregate root — the single place invariants are enforced.
- **Scale it** by indexing reservations by date; the model doesn't change, only the lookup.
- **Enums via `Object.freeze`** so `RoomType` / `ReservationStatus` can't be mutated.

---

## Connected topics

| Direction | Topic | Why |
| --- | --- | --- |
| **Previous** | [118 — Movie Ticket Booking](./118-lld-movie-ticket-booking.md) | The sibling reservation LLD — seats instead of rooms, the same double-booking race. |
| **Next** | [111 — LLD Approach Framework](./111-lld-approach-framework.md) | The 7-step method this doc applied; use it on any LLD problem. |
| **Related** | [42 — Strategy Pattern](./42-pattern-strategy.md) | The pattern behind swappable pricing and cancellation policies. |
| **Related** | [52 — Concurrency Fundamentals](./52-concurrency-fundamentals.md) | Why the atomic re-check in `book()` is needed to stop double-booking. |
| **Related** | [110 — HLD Ticketmaster](./110-hld-ticket-master.md) | The distributed-scale version of the same seat/room reservation problem. |
