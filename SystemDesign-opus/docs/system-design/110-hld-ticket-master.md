# 110 — Design Ticketmaster / Event Booking
## Category: HLD Case Study

---

## What is this?

You are designing the backend that sells tickets to events — concerts, sports, theatre. A user browses events, opens a seat map, picks the two front-row seats they want, pays, and gets a confirmation. Simple on the surface.

The nightmare underneath: when a hot concert goes on sale, one hundred thousand people slam the BUY button in the first sixty seconds, all fighting over the same ten thousand seats. Two people must **never** be sold the same seat. This is the definitive **high-contention booking** problem — many humans racing for the same scarce resource at the same instant.

The real-world analogy: a physical box office with one clerk and a mob outside. The clerk can only hand each ticket to one person, and there is a velvet rope managing the line so the mob does not trample the desk.

---

## Why does it matter?

**The interview angle.** Ticketmaster (and its cousins — Uber's surge pricing, StubHub, flash-sale e-commerce like a PS5 drop) is the go-to interview question for testing whether you understand **strong consistency under concurrency**. The interviewer is watching for one thing: do you casually reach for eventual consistency (which you learned is fine for feeds and likes), or do you recognize that selling a seat is a **correctness** problem where "eventually" is a bug? If you say "we'll cache the seat status and sync later" for the *write* path, you have failed the question. Getting the read/write split right is the mark of a senior answer.

**The real-work angle.** Any system where a finite inventory is claimed by concurrent users hits this exact wall: booking systems, limited-edition drops, ad-slot auctions, warehouse stock, appointment scheduling, parking spots. If you don't understand atomic state transitions and hold-with-TTL, you will ship a double-booking bug that costs real money and real trust. Double-selling a wedding-anniversary concert seat is not a rounding error — it is a lawsuit and a viral tweet.

---

## The core idea — explained simply

### Analogy: the coat-check with a numbered ticket and a timer

Imagine a very popular coat-check at a gala. There are exactly 10,000 hooks (seats). A crowd of 100,000 guests arrives at once, each wanting a *specific* hook they saw through the window.

1. **The velvet rope (virtual queue).** You cannot let all 100,000 storm the counter — the two clerks would be crushed. A bouncer forms a line outside and lets people in a few at a time, at a pace the clerks can actually serve.
2. **Claiming a hook (the HOLD).** When a guest reaches the counter and points at hook #47, the clerk immediately flips a little sign on that hook to **RESERVED — expires in 10 minutes** and hands the guest a claim stub. No one else can point at #47 now. But the guest hasn't *paid* yet.
3. **The timer (TTL).** If the guest wanders off and doesn't pay within 10 minutes, the sign flips back to **AVAILABLE** automatically and hook #47 is free again. Holds cannot last forever or the whole coat-check locks up with abandoned claims.
4. **Paying (HELD → BOOKED).** The guest pays, the clerk staples the ticket permanently to the hook — **SOLD**. This must happen *exactly once* even if the guest's card machine hiccups and retries.

Every technical piece maps directly onto this story:

| Coat-check thing | Technical concept |
|---|---|
| Velvet rope + bouncer letting people in batches | **Virtual waiting room / queue service** |
| Flipping the sign to RESERVED atomically | **Atomic compare-and-set** (`AVAILABLE → HELD`) |
| "Expires in 10 minutes" on the sign | **Hold TTL** (Redis key expiry or `hold_expires_at` column) |
| Only one clerk can flip a given sign at a time | **Serialization on the seat row / lock** |
| Stapling the ticket permanently, exactly once | **Idempotent** `HELD → BOOKED` on payment |
| Looking through the window at which hooks are free | **Cached, slightly-stale seat map (read path)** |
| Pointing and the clerk saying "sorry, just taken" | **Authoritative check at reservation time (write path)** |

The whole design is that one story. Everything below is engineering it to survive 100,000 people at once without ever selling hook #47 twice.

---

## Key concepts inside this topic

We execute the standard 5-step HLD framework, but the heart of this problem lives in steps 3, 4, and 5 (the concurrency, the queue, and the read/write split), so we spend most of our time there.

### 1. Requirements

**Functional (what the system does):**

- **Browse events** — search/list concerts, sports, shows by city, date, artist.
- **View a seat map** — for a chosen event, show every seat and whether it is available, in near-real-time.
- **Select and reserve specific seats** — a user claims one or more exact seats (not "any 2 seats" — *these* two).
- **Pay within a time limit** — the user has a countdown (e.g. 10 minutes) to complete payment.
- **Confirm the booking** — on successful payment, the seats become permanently theirs and they get tickets.

**Non-functional (the qualities that make or break it):**

- **STRONG consistency on seat inventory.** This is the cardinal rule. A seat is sold to **exactly one** person, never two. Double-booking is the single worst failure this system can have. We will *not* trade this away for latency or availability on the write path.
- **Handle enormous, spiky demand.** A hot concert is a **flash sale**: ~100,000 users hitting BUY for ~10,000 seats in the first minute, then a long tail. Normal browsing is orders of magnitude lighter.
- **Low latency** on the seat map and on the "is this seat still free?" reservation attempt — users abandon slow pages.
- **Fairness.** Someone who arrived first should generally get served first; the system should not feel like a random lottery (bots aside).
- **High availability** for browsing and for the queue, even while the inventory service is under pressure.

**Clarifying questions to ask the interviewer:**

- Are seats **assigned** (pick exact seats) or **general admission** (just a count)? Assigned is harder — we'll design for it; GA is a simpler decrement-a-counter variant.
- How long is a hold allowed? (Drives contention — longer holds = more seats locked up.)
- One event at a time or many concurrent hot events? (Sharding strategy.)
- Do we need to support resale/transfers? (Out of scope here; adds a whole marketplace.)
- Global or single-region? (Affects where the inventory DB lives — strong consistency is easier in one region.)

### 2. Capacity estimation

Let's separate the two very different loads.

**Normal browse load.** Say the platform has 10M daily active users, mostly browsing.

```
10,000,000 DAU
Each does ~20 browse/search/seat-map requests per day
= 200,000,000 read requests/day
÷ 86,400 sec/day ≈ 2,300 reads/sec average
Peak factor ~5×  ≈ 11,500 reads/sec  (comfortable with caching)
```

Writes (actual purchases) in normal times are a trickle — a few hundred/sec. Nothing scary.

**The flash-sale spike — this is the real design driver.** One mega-event goes on sale:

```
Users hammering this ONE event:      100,000 concurrent
Seats available:                      10,000
Time window (first burst):            ~60 seconds
```

Now the crucial split — most of those 100,000 are **reading**, few are **writing**:

```
READ path (seat map):
  100,000 users each polling/refreshing the seat map,
  say once every 2 seconds during the frenzy
  = 100,000 / 2 = 50,000 seat-map reads/sec  on ONE event
  → massive read amplification, but cacheable

WRITE path (reserve a seat):
  Only ~10,000 seats exist, and each can be reserved once.
  Even if everyone tries, successful writes are bounded by inventory.
  Attempts might spike to thousands/sec, but the DB only needs to
  *commit* ~10,000 reservations total across the burst.
  → small volume, but every one must be strongly consistent
```

**The asymmetry is the whole insight.** The read path is huge (50,000 rps) but tolerates staleness — a cache can absorb it, and it's fine if the map shows a seat as "available" for a second after someone grabbed it. The write path is tiny in volume but demands perfect correctness. **The entire architecture hinges on separating these two paths** so the giant, forgiving read load never touches the small, unforgiving write load.

**Storage** is trivial by comparison. A seat row is maybe 200 bytes. A 60,000-seat stadium × 200B = 12 MB per event. Even 1M events over 5 years is ~12 TB — a rounding error for a relational DB with archival. Bookings/payments add a bit more but nothing exotic. **This is not a storage problem. It is a concurrency problem.**

### 3. The core problem — preventing double-booking under extreme concurrency

Picture it concretely: **5,000 people click the same front-row seat A1 within the same second.** Exactly one must win. The other 4,999 must be told "taken" instantly. Let's see why the obvious code is catastrophically wrong, then build up to the real answer.

**The naive, broken approach (a race condition):**

```js
// DO NOT DO THIS — textbook race condition
async function reserveSeatBROKEN(seatId, userId) {
  const seat = await db.query(
    'SELECT status FROM seats WHERE seat_id = $1', [seatId]
  );
  if (seat.status === 'AVAILABLE') {      // <-- 5,000 threads all read AVAILABLE here
    await db.query(
      'UPDATE seats SET status = $1, held_by = $2 WHERE seat_id = $3',
      ['HELD', userId, seatId]
    );                                     // <-- and all 5,000 happily write
    return { ok: true };
  }
  return { ok: false };
}
```

The gap between the `SELECT` and the `UPDATE` is where the disaster lives. Thousands of requests read `AVAILABLE` before any of them writes, so thousands all believe they won. You just sold seat A1 five thousand times. This is the **read-modify-write race**. We need the check and the write to be **one indivisible operation**.

**Recall from [66 — Database Transactions & Isolation]** the tools that fix this. Three real solutions, in order:

#### (a) Pessimistic DB locking — `SELECT ... FOR UPDATE`

Lock the seat row *before* reading it, inside a transaction. Any other transaction wanting that row must wait.

```js
async function reserveSeatPessimistic(seatId, userId) {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');
    // FOR UPDATE takes a row lock; concurrent txns block here until we commit
    const { rows } = await client.query(
      'SELECT status FROM seats WHERE seat_id = $1 FOR UPDATE', [seatId]
    );
    if (rows[0].status !== 'AVAILABLE') {
      await client.query('ROLLBACK');
      return { ok: false, reason: 'TAKEN' };
    }
    await client.query(
      `UPDATE seats SET status = 'HELD', held_by = $1,
              hold_expires_at = NOW() + INTERVAL '10 minutes'
       WHERE seat_id = $2`, [userId, seatId]
    );
    await client.query('COMMIT');
    return { ok: true };
  } finally {
    client.release();
  }
}
```

**Correct.** But under a flash sale it hurts: 5,000 transactions queue on that one row lock, and if a lock is held while a user *thinks* or *pays* (don't do that, but people try), the queue backs up and connections exhaust. Row locks are for the brief moment of the state flip, never across a human's payment. Good correctness, poor throughput on the hottest rows.

#### (b) Optimistic locking — conditional update on a status/version column

Don't lock. Just make the UPDATE itself the atomic check. Attempt the state change in a single statement whose `WHERE` clause encodes the precondition. The database guarantees each row-update is atomic, so exactly one wins.

```js
async function reserveSeatOptimistic(seatId, userId) {
  // The magic: the WHERE clause is the guard. Only ONE of the 5,000
  // concurrent statements will find status='AVAILABLE' and change it.
  const { rowCount } = await db.query(
    `UPDATE seats
        SET status = 'HELD', held_by = $1, version = version + 1,
            hold_expires_at = NOW() + INTERVAL '10 minutes'
      WHERE seat_id = $2 AND status = 'AVAILABLE'`,
    [userId, seatId]
  );
  // rowCount === 1  -> you won the seat
  // rowCount === 0  -> someone else already flipped it; you lost the race
  return { ok: rowCount === 1, reason: rowCount === 0 ? 'TAKEN' : null };
}
```

`0 rows affected means you lost.` No explicit locks, no read-then-write gap. This is a **compare-and-set (CAS)** at the SQL level. It's excellent for low-to-moderate contention and is the workhorse pattern. Under extreme contention on one row it does more retries/wasted work, but it never double-books.

#### (c) The reservation / HOLD model with a TTL — the real answer

Neither (a) nor (b) alone solves the *product* problem: the user needs **time to pay** without someone stealing their seat, and abandoned attempts must not lock seats forever. So we model the seat as a **state machine** and add a **hold with an expiry (TTL)**.

Three states:

```
        select seat (atomic CAS)              pay (idempotent)
AVAILABLE  ─────────────────────────▶  HELD  ─────────────────────▶  BOOKED
    ▲                                    │
    └──────────  hold TTL expires  ◀──────┘
                 (or user cancels)
```

- **AVAILABLE → HELD** must be an **atomic compare-and-set** (exactly the optimistic update above, or a Redis atomic op). This is where the race is resolved — exactly one winner.
- **HELD** carries a `hold_expires_at`. If the user doesn't pay in time, the hold auto-expires and the seat returns to AVAILABLE. This is what lets the user safely walk into the payment screen.
- **HELD → BOOKED** happens on payment success and must be **idempotent** — **recall [85 — Idempotency]** — because the payment webhook may fire twice, the client may retry, and the network may double-deliver. Booking a seat twice for the same payment must be a no-op the second time.

**Where does the hold + TTL live?** Two common homes, often combined:

- **A distributed lock / Redis key with expiry** — **recall [84 — Distributed Locking]**. `SET seat:{event}:{A1} {userId} NX PX 600000` claims the seat atomically (`NX` = only if not exists) with a 600,000 ms (10 min) auto-expiry. Redis is single-threaded per key, so the CAS is naturally atomic and *blazing* fast — perfect for absorbing the write spike. The TTL cleanup is automatic; no sweeper needed for the lock itself.
- **A DB row with an `hold_expires_at` timestamp**, swept by a background job (or checked lazily). This is the durable source of truth; Redis alone can lose data on failover, so the DB row is what you actually trust for the sale.

A robust design uses Redis as the fast front-line CAS to shed the spike, and the relational DB as the durable, authoritative record that the sale is ultimately committed against.

**Say it out loud in the interview:** *This is a correctness problem. We must NOT rely on eventual consistency for the seat inventory write path.* Contrast it with the social/feed systems earlier in the course: for a like count or a timeline, "eventually consistent, approximately right" is perfectly fine — nobody is harmed if a like count lags by a second. Here, "approximately right" means *two people flew across the country to sit in the same chair.* Different problem, different rules.

### 4. The virtual waiting room / queue — handling the flash-sale spike

You **cannot** let 100,000 people hit the booking/inventory system at once. Even with perfect CAS logic, the sheer connection count, the thundering herd on the hot rows, and the payment fan-out will melt the core. The industry solution — the thing that is *literally happening* when you stare at a "You are number 43,918 in line" screen for a concert — is a **virtual queue** (a.k.a. virtual waiting room).

```
   100,000 users hit "BUY" at sale start
                │
                ▼
     ┌──────────────────────┐
     │   QUEUE SERVICE      │   Everyone gets a queue TOKEN + position.
     │  (Redis sorted set / │   Waiting room page just polls "am I in yet?"
     │   dedicated service) │   — this page is cheap, static, cacheable.
     └──────────┬───────────┘
                │  admits users in CONTROLLED BATCHES
                │  at a rate the backend can handle
                │  e.g. 500 users every 2 seconds
                ▼
     ┌──────────────────────┐
     │   BOOKING FLOW       │   Only admitted users, holding a valid
     │  (seat map + reserve)│   access token, may call reserve/pay.
     └──────────────────────┘
```

**How it works:**

1. At sale start, every arriving user is **enqueued** and handed a token with their position (e.g. a Redis sorted set keyed by arrival timestamp gives natural FIFO ordering).
2. The user's browser sits on a lightweight **waiting-room page** that just polls "is it my turn yet?" — this page hits a tiny, heavily-cached endpoint, not the booking system.
3. An **admission controller** releases users into the real booking flow in **batches**, at a rate the inventory service can comfortably serve (say 500 users per 2 seconds). Admission mints a short-lived **access token** (a signed JWT with an expiry) that the booking endpoints require. No valid token → you're not getting in, no matter how fast your bot clicks.
4. Admitted users get a bounded window to browse the seat map and reserve. If they idle out, their token expires and they may be re-queued.

**Why it's the right move:**

- **Protects the core system** — the inventory DB and payment path only ever see a manageable, throttled trickle, never the 100,000-user tidal wave.
- **Fairness** — FIFO admission means earlier arrivals generally go first, instead of "whoever has the fastest connection and a bot wins."
- **Better UX** — a concrete position ("you are #4,318, ~6 min wait") beats a wall of 503 errors and spinning failures. Users wait calmly instead of retry-storming.

This is the piece beginners forget. The clever CAS logic prevents double-booking; the **queue** is what keeps the whole thing standing up long enough to *use* that logic.

### 5. Splitting the read and write paths

Now we make the read/write asymmetry from step 2 **architectural and explicit** — this is the single most important design decision, and stating it clearly is what separates a strong answer from a mediocre one.

- **The seat MAP is a READ path.** It is served from a **heavily cached, possibly slightly-stale view** (Redis/CDN). Fifty thousand reads/sec on one event? Fine — the cache absorbs it, refreshing every second or two. **It is OK if the map briefly shows a seat as available for a second after someone else grabbed it.** The map is a *hint*, not a promise. Its whole job is to let people browse without hammering the DB.
- **The RESERVE action is a WRITE path.** It goes to the **strongly-consistent inventory service** (Redis CAS front-line + relational DB of record). This is the *authoritative* check. Even if the cached map lied and said A1 was free, the atomic CAS at reserve time is the real judge: exactly one caller flips `AVAILABLE → HELD`, everyone else is told "taken." **The stale map never causes a double-booking, because the map never sells anything — only the write path does.**

The mental model: **the map is a menu; the reservation is the kitchen.** The menu can be a little out of date. The kitchen is the source of truth about what's actually left.

### 6. API design

Concrete endpoints, split cleanly along the read/write line. Notice the queue gate on the write endpoints.

```
# ---- Browsing / read path (cheap, cacheable) ----
GET  /events?city=NYC&date=2026-08-01        -> list events
GET  /events/{eventId}                        -> event details
GET  /events/{eventId}/seatmap                -> seat map snapshot (may be ~1-2s stale)
                                                 200 { seats: [{seatId, section, row, num, status}] }

# ---- Virtual queue ----
POST /events/{eventId}/queue                   -> join queue
                                                 200 { queueToken, position, etaSeconds }
GET  /queue/{queueToken}                        -> poll status
                                                 200 { admitted: false, position: 4318 }
                                              or 200 { admitted: true, accessToken }  # short-lived JWT

# ---- Reserve / write path (requires a valid accessToken) ----
POST /events/{eventId}/holds                   -> atomic hold of specific seats
     body { seatIds:[...], accessToken }        201 { holdId, expiresAt } | 409 { taken:[...] }
DELETE /holds/{holdId}                          -> release early (user changed mind)

# ---- Payment / confirm (idempotent) ----
POST /holds/{holdId}/checkout                  -> begin payment
     header Idempotency-Key: <uuid>             200 { bookingId, status:'CONFIRMED' }
                                              or 409 { reason:'HOLD_EXPIRED' }
GET  /bookings/{bookingId}                      -> booking + tickets
```

The `Idempotency-Key` header on checkout (recall 85) is what makes a retried request safe — the server keys the booking on it and returns the same result instead of charging twice.

### 7. Data model (fields + DB choice)

Relational, normalized, strongly consistent.

| Table | Key fields | Notes |
|---|---|---|
| `events` | `event_id`, `venue_id`, `name`, `starts_at`, `on_sale_at`, `status` | one row per show |
| `venues` | `venue_id`, `name`, `address`, `sections` | reusable across events |
| `seats` | `seat_id`, `event_id`, `section`, `row`, `number`, **`status`**, `held_by`, **`hold_expires_at`**, **`version`** | the hot table; status ∈ AVAILABLE/HELD/BOOKED |
| `reservations` | `hold_id`, `event_id`, `user_id`, `seat_ids[]`, `created_at`, `expires_at` | groups seats held together |
| `bookings` | `booking_id`, `user_id`, `seat_id`, **`payment_id`** (unique), `created_at` | the permanent sale record |
| `payments` | `payment_id`, `booking_ref`, `amount`, `status`, `idempotency_key` | charge lifecycle |

The three bolded seat columns are the machinery: `status` is the state, `hold_expires_at` is the TTL, `version` powers optimistic CAS. Two constraints matter most:

- A **partial unique index** so a seat can be BOOKED at most once: `CREATE UNIQUE INDEX ON seats (event_id, seat_number) WHERE status = 'BOOKED';`
- A **unique `payment_id` on `bookings`** so a replayed payment can't create two bookings.

**DB choice — say this explicitly:** use a **strongly-consistent relational DB (PostgreSQL/MySQL)** for inventory. This is **NOT** a NoSQL eventual-consistency use case. ACID transactions, row-level atomic updates, and unique constraints are exactly the tools selling a scarce seat needs. NoSQL's "eventually consistent, fast writes" is the *opposite* of the guarantee we require. (You *can* put the giant seat-map read cache in Redis/CDN — that's fine, because the cache never sells anything.)

### 8. Bottlenecks, failure modes, and scaling

- **Hot inventory rows during a flash sale.** All the contention lands on one event's seats. **Shard the inventory by `event_id`** so different concerts live on different DB shards and don't interfere. The honest caveat: a *single* mega-event is still a single hot shard — you can't shard one concert's 10,000 seats across the planet and keep strong consistency cheaply. Mitigations: the Redis CAS front-line absorbs most attempts before they reach the DB; you can partition the seat map into sections handled by separate workers; and the virtual queue caps the arrival rate. But a blockbuster on-sale is *genuinely hard* — admit that in the interview rather than pretending sharding makes it free.
- **If the hold store (Redis) fails.** Holds in flight may be lost. Because the **relational DB is the source of truth**, no sale is corrupted — a lost Redis hold just means the seat's durable row still says HELD until the sweeper (or the DB `hold_expires_at`) reclaims it. Run Redis with replication/Sentinel; on failover, reconcile against the DB. Never treat Redis as the authority for a *completed* sale.
- **Payment provider slowness/outage.** Holds keep users' seats during the wait; if payment can't confirm before the TTL, the hold expires and the seat is resold — the checkout must then fail cleanly and refund any late-succeeding charge (the payment-timeout race, handled by re-validating the hold).
- **Thundering-herd cache stampede** on the seat map when a popular event's cache entry expires: use a single-flight refresh (one worker rebuilds, others serve the last snapshot) so 50,000 readers don't all hit the DB at once.
- **Bots and scalpers.** The queue is also a defense line, but add: per-user **seat caps** (max N tickets per account/event), **rate limiting** and device fingerprinting on the reserve endpoint, and **CAPTCHA** at admission to blunt automated clients. None is perfect, so combine them; the seat cap in particular is enforced in the same transaction that flips HELD→BOOKED so it can't be raced.
- **Fairness in the queue.** FIFO by arrival timestamp (a Redis sorted set) is the simplest fair policy; some platforms deliberately *randomize* the initial order at sale start so that raw network speed and geography don't decide winners. Either way, admission is token-gated so a fast bot can't skip the line — it still needs a valid, its-your-turn access token to reach the booking flow.

---

## Visual / Diagram description

**Architecture — read path vs write path, guarded by the virtual queue.** Redraw this on the whiteboard; it is the answer.

```
                         100,000 users at sale start
                                    │
                          ┌─────────▼──────────┐
                          │   VIRTUAL QUEUE    │  token + position;
                          │   (admission ctrl) │  admits in batches
                          └─────────┬──────────┘
                                    │ (only holders of a valid access token pass)
                    ┌───────────────┴───────────────┐
                    │                               │
             READ PATH (huge, stale-OK)      WRITE PATH (small, strict)
                    │                               │
         ┌──────────▼───────────┐        ┌──────────▼────────────┐
         │  Seat-Map Service    │        │  Inventory / Booking  │
         │  reads from CACHE    │        │  Service              │
         └──────────┬───────────┘        └──────────┬────────────┘
                    │                               │ atomic CAS
         ┌──────────▼───────────┐        ┌──────────▼────────────┐
         │  Redis / CDN cache   │◀───────│  Redis (hold+TTL)     │
         │  seat map snapshot   │ invalid│  SET ... NX PX 600000 │
         │  (refresh ~1-2s)     │  ate   └──────────┬────────────┘
         └──────────────────────┘                   │ commit sale
                                          ┌──────────▼────────────┐
                                          │  RELATIONAL DB (SoT)  │
                                          │  seats/reservations/  │
                                          │  bookings/payments    │
                                          │  STRONG consistency   │
                                          └──────────┬────────────┘
                                                     │
                                          ┌──────────▼────────────┐
                                          │  Payment Service      │ (idempotent)
                                          └───────────────────────┘
```

**In prose:** The queue is the bouncer — nothing reaches the core without an admission token. Past the queue, traffic forks. The **read path** (left) serves the seat map from cache and never touches the authoritative store; it soaks up the 50,000 rps. The **write path** (right) is the only thing that can change inventory: it does an atomic hold in Redis, then commits the durable sale to the relational DB, then charges via the idempotent payment service. When a seat's state changes on the write path, the cache is invalidated so the map catches up within a second or two.

**Seat state machine** (the correctness core):

```
                atomic CAS                 payment success (idempotent)
   ┌──────────┐  AVAILABLE→HELD  ┌──────┐  HELD→BOOKED       ┌────────┐
   │AVAILABLE │ ───────────────▶ │ HELD │ ─────────────────▶ │ BOOKED │
   └──────────┘                  └──┬───┘                    └────────┘
        ▲                           │                         (terminal)
        │   TTL expiry / cancel     │
        └───────────────────────────┘
              HELD→AVAILABLE
```

Only three states. Every edge is either an atomic operation or a timer. BOOKED is terminal — once a seat is sold it never goes back.

---

## Real world examples

### Ticketmaster (representative)

Ticketmaster publicly uses a **"Smart Queue" / virtual waiting room** for high-demand on-sales — exactly the batched-admission pattern in step 4. Fans are placed in a randomized/ordered queue at sale start and admitted in waves. Behind it, inventory is managed with holds and timed carts (the classic "you have 10:00 to complete your purchase" countdown), which is the HELD-with-TTL state machine. The countdown you see *is* the hold expiry. (Internal specifics are proprietary; the observable behavior maps cleanly onto this design.)

### Amazon-style flash sales / Lightning Deals (representative)

Limited-quantity, time-boxed deals (and console/GPU drops) hit the identical wall: finite stock, a spike of concurrent buyers, and a "reserved in your cart for N minutes" hold. Large e-commerce platforms front these with **queueing/throttling layers** and per-item atomic inventory decrements so stock never goes negative — the same read/write split (browse the deal page from cache; claim stock through a strict atomic path). See **[104 — Design Amazon](./104-hld-amazon.md)** for the broader commerce architecture this plugs into.

### Airline / hotel booking engines (conceptual)

Seat/room booking systems use **temporary holds** ("seat locked for 15 minutes while you enter passenger details") backed by a strongly-consistent inventory store, plus **overbooking policy** as a *deliberate business* choice — importantly a controlled, accounted-for decision, never the accidental double-booking bug we are preventing here.

---

## Trade-offs

**Concurrency-control strategy:**

| Approach | Pros | Cons |
|---|---|---|
| Pessimistic `FOR UPDATE` | Simple, obviously correct | Lock queues under flash-sale contention; risk if held across payment |
| Optimistic CAS (`WHERE status='AVAILABLE'`) | No locks, high throughput, never double-books | Wasted retries under extreme single-row contention |
| Redis hold + TTL (distributed lock) | Absorbs the spike, auto-expiring holds, very fast | Redis can lose data on failover → must reconcile with durable DB |

**Where the hold lives:**

| Home for the hold | Pros | Cons |
|---|---|---|
| Redis key with `PX` expiry | Automatic cleanup, atomic, fast | Not durable alone; needs DB as source of truth |
| DB row + `hold_expires_at` + sweeper | Durable, single source of truth | Sweeper lag; extra write load; hot rows |

**Consistency choice:**

| Choice | Pros | Cons |
|---|---|---|
| Strong consistency on inventory | No double-booking — the whole point | Lower write throughput; harder to scale a single hot event |
| Eventual consistency on inventory | Scales trivially | **Double-books seats — unacceptable.** Never do this for the write path |

**The sweet spot:** A **virtual queue** to throttle the herd, an **atomic CAS hold with a TTL** (Redis front-line for speed, relational DB as the durable source of truth) for the write path, a **cached, stale-tolerant seat map** for the read path, and **idempotent** payment confirmation — with a **DB unique constraint** as the last-line backstop against overselling.

---

## Common interview questions on this topic

### Q1: "Two users click the same seat at the same millisecond. How do you guarantee only one gets it?"

**Hint:** An atomic compare-and-set — either the SQL conditional update `UPDATE seats SET status='HELD' WHERE seat_id=? AND status='AVAILABLE'` (0 rows affected = you lost) or a Redis `SET key val NX PX ttl`. The check and the write are one indivisible operation, so the read-modify-write race disappears. Explicitly reject the naive SELECT-then-UPDATE.

### Q2: "How do you stop the system from melting when 100,000 people hit BUY in the first minute?"

**Hint:** A virtual waiting room / queue. Enqueue everyone with a token and position, admit them into the real booking flow in controlled batches at a rate the backend can serve, and require a signed access token on the booking endpoints. It protects the core, provides fairness, and gives users a position instead of errors.

### Q3: "A user reserves a seat but never pays. How do you free it — and what if their payment lands right as the hold expires?"

**Hint:** Auto-expiring hold (Redis TTL and/or a `hold_expires_at` swept by a job). For the payment-timeout race: the payment confirmation must **re-validate the hold** before committing HELD→BOOKED. If the hold already expired and the seat was resold, refund/compensate; idempotency ensures a retried webhook doesn't double-book. Never blindly trust that a successful charge still owns the seat.

### Q4: "Why not just cache the seat inventory and sync eventually, like a news feed?"

**Hint:** Because selling a seat is a **correctness** problem, not a freshness problem. Eventual consistency means two people can both read "available" and both buy — a double-booking. The read path (seat map) *can* be eventually consistent; the write path (reserve) must be strongly consistent. That split is the core of the design.

### Q5: "What's your final backstop if every layer of your concurrency logic has a bug?"

**Hint:** A database **unique constraint** — e.g. a partial unique index on `(event_id, seat_id)` for BOOKED rows, so the DB itself physically refuses to record the same seat sold twice. Defense in depth: even if the queue, the CAS, and the hold all failed, the constraint throws and the second booking errors out instead of corrupting inventory.

---

## Practice exercise

### Build a working atomic seat-hold and expiry (~30-40 min)

Write a small Node.js module that simulates the seat state machine — no framework, just an in-memory map plus timers (or a real Redis if you have one).

Produce:

1. A `SeatInventory` class holding seats keyed by `seatId`, each with `{ status, heldBy, holdExpiresAt, version }`.
2. `hold(seatId, userId, ttlMs)` that performs an **atomic** `AVAILABLE → HELD` compare-and-set and sets an expiry. It must return `false` (not throw) if the seat is not AVAILABLE.
3. `confirm(seatId, userId, paymentId)` that performs an **idempotent** `HELD → BOOKED`, and **re-validates** the hold hasn't expired or been reassigned before committing. Calling it twice with the same `paymentId` must not double-book.
4. Expiry that returns HELD seats to AVAILABLE after the TTL.
5. A test harness firing **1,000 concurrent** `hold(A1, ...)` calls and asserting **exactly one** succeeds.

If your test ever shows two winners, you have reproduced the double-booking bug in miniature — fix it until it can't happen. That is the entire lesson of this document in one exercise.

---

## Quick reference cheat sheet

- **Cardinal rule:** a seat is sold to **exactly one** person — double-booking is the worst possible bug.
- **Strong consistency on the write path (reserve); eventual is fine on the read path (seat map).** The whole design hinges on separating these.
- **Read amplification:** ~50,000 seat-map reads/sec vs a few thousand write attempts on one hot event — read path is huge but cacheable, write path is tiny but strict.
- **Naive `SELECT`-then-`UPDATE` races catastrophically** — thousands read AVAILABLE before anyone writes.
- **Atomic CAS is the fix:** `UPDATE ... WHERE status='AVAILABLE'` (0 rows = you lost) or Redis `SET key val NX PX ttl`.
- **Three-state machine:** `AVAILABLE → HELD → BOOKED`, with `HELD → AVAILABLE` on TTL expiry.
- **HOLD with a TTL** gives the user time to pay without letting abandoned carts lock seats forever.
- **HELD → BOOKED must be idempotent** — payment webhooks and client retries fire more than once (recall 85).
- **Virtual queue** throttles the flash-sale herd, admits users in batches with a signed token, and provides fairness.
- **Pessimistic (`FOR UPDATE`)** = correct but lock queues; **optimistic (CAS)** = high throughput; **Redis hold** = absorbs the spike.
- **DB choice:** strongly-consistent relational (Postgres/MySQL) for inventory — **not** NoSQL eventual consistency.
- **Overselling backstop:** a DB **unique constraint** on booked `(event_id, seat_id)` as defense-in-depth.
- **Payment-timeout race:** payment must re-validate the hold before committing; compensate/refund if the seat was lost.
- **Hardest case:** a single mega-event is a genuinely hot shard — sharding by event helps everything *except* the one blockbuster.
- **Bot defense:** per-user seat caps (enforced in the same transaction as HELD→BOOKED), rate limiting, and CAPTCHA at admission — combined, since none alone suffices.
- **API split:** read endpoints (`/seatmap`) are cacheable and open; write endpoints (`/holds`, `/checkout`) require a queue-issued `accessToken` and an `Idempotency-Key`.

---

## Connected topics

| Direction | Topic | Why |
|---|---|---|
| **Previous** | [66 — Database Transactions & Isolation](./66-database-transactions-and-isolation.md) | The `FOR UPDATE` locks, optimistic CAS, and ACID guarantees that make seat reservation correct |
| **Next** | [118 — LLD: Movie Ticket Booking](./118-lld-movie-ticket-booking.md) | Zooms from this HLD into the class-level design of the same booking problem |
| **Related** | [84 — Distributed Locking](./84-distributed-locking.md) | Where the atomic hold-with-TTL lives — Redis `SET NX PX` is the seat lock |
| **Related** | [85 — Idempotency](./85-idempotency.md) | The HELD→BOOKED payment confirmation must be idempotent against retried webhooks |
| **Related** | [104 — Design Amazon](./104-hld-amazon.md) | The broader commerce/flash-sale architecture this inventory pattern plugs into |

---

### Appendix: reference Node.js — atomic hold (CAS) + expiry + idempotent confirm

Two implementations: a **SQL** version (durable source of truth) and a **Redis** version (fast front-line), plus the sweeper and the booking sequence they serve.

```js
// ---------- SQL: atomic AVAILABLE -> HELD via optimistic CAS ----------
// The WHERE clause is the guard. Exactly one of N concurrent callers
// finds status='AVAILABLE' and flips it; the rest get rowCount === 0.
async function holdSeatSQL(db, { seatId, userId, ttlMinutes = 10 }) {
  const { rowCount } = await db.query(
    `UPDATE seats
        SET status = 'HELD',
            held_by = $1,
            version = version + 1,
            hold_expires_at = NOW() + ($2 || ' minutes')::interval
      WHERE seat_id = $3
        AND status = 'AVAILABLE'`,   // <-- the atomic precondition
    [userId, String(ttlMinutes), seatId]
  );
  return rowCount === 1;             // true = you won the seat, false = taken
}

// ---------- SQL: idempotent HELD -> BOOKED on payment success ----------
// Re-validates the hold (still HELD, still yours, not expired) and only
// then commits the sale. Idempotent on paymentId: a retried webhook is a
// no-op the second time.
async function confirmBookingSQL(db, { seatId, userId, paymentId }) {
  const client = await db.connect();
  try {
    await client.query('BEGIN');

    // Idempotency guard: has THIS payment already booked something? If so, done.
    const existing = await client.query(
      'SELECT booking_id FROM bookings WHERE payment_id = $1', [paymentId]
    );
    if (existing.rowCount > 0) {
      await client.query('COMMIT');
      return { ok: true, bookingId: existing.rows[0].booking_id, replayed: true };
    }

    // Re-validate the hold: must still be HELD by THIS user and not expired.
    // The WHERE clause makes the transition atomic and safe against the
    // payment-timeout race (if the hold expired, rowCount === 0 -> compensate).
    const flip = await client.query(
      `UPDATE seats SET status = 'BOOKED'
        WHERE seat_id = $1 AND status = 'HELD'
          AND held_by = $2 AND hold_expires_at > NOW()`,
      [seatId, userId]
    );
    if (flip.rowCount !== 1) {
      await client.query('ROLLBACK');
      // Hold lost (expired or resold). Caller must refund/compensate this payment.
      return { ok: false, reason: 'HOLD_LOST' };
    }

    const booking = await client.query(
      `INSERT INTO bookings (seat_id, user_id, payment_id, created_at)
       VALUES ($1, $2, $3, NOW()) RETURNING booking_id`,
      [seatId, userId, paymentId]
    );
    await client.query('COMMIT');
    return { ok: true, bookingId: booking.rows[0].booking_id, replayed: false };
  } catch (e) {
    await client.query('ROLLBACK');
    throw e;
  } finally {
    client.release();
  }
}

// ---------- Redis: fast front-line hold with automatic TTL ----------
// SET key value NX PX ttl  -> atomic "claim only if not already held",
// with a built-in expiry so abandoned holds clean themselves up.
async function holdSeatRedis(redis, { eventId, seatId, userId, ttlMs = 600000 }) {
  const key = `hold:${eventId}:${seatId}`;
  // NX = only set if key does not exist (the CAS); PX = expiry in ms (the TTL).
  const result = await redis.set(key, userId, 'NX', 'PX', ttlMs);
  return result === 'OK';           // 'OK' = you won; null = already held
}

// Release a hold ONLY if you own it (avoids releasing someone else's seat).
// A tiny Lua script keeps the check-and-delete atomic.
const RELEASE_LUA = `
  if redis.call('get', KEYS[1]) == ARGV[1] then
    return redis.call('del', KEYS[1])
  else
    return 0
  end`;
async function releaseHoldRedis(redis, { eventId, seatId, userId }) {
  return redis.eval(RELEASE_LUA, 1, `hold:${eventId}:${seatId}`, userId);
}

// ---------- Sweeper: reliably release expired DB holds ----------
// Redis TTL cleans the fast lock automatically, but the durable DB rows
// need a periodic sweep (defense-in-depth; also handles Redis-less holds).
async function sweepExpiredHolds(db) {
  const { rowCount } = await db.query(
    `UPDATE seats
        SET status = 'AVAILABLE', held_by = NULL, hold_expires_at = NULL
      WHERE status = 'HELD' AND hold_expires_at <= NOW()`
  );
  if (rowCount > 0) console.log(`Swept ${rowCount} expired holds back to AVAILABLE`);
  return rowCount;
}
// Run every few seconds; keep the interval short so seats return quickly,
// but note it's a backstop — the authoritative check is always at confirm time.
// setInterval(() => sweepExpiredHolds(db).catch(console.error), 5000);
```

**The booking sequence (end to end, with the timeout branch):**

```
User        Queue         SeatMap(cache)   Inventory(CAS)   Payment
 │  join      │                │                │              │
 ├───────────▶│  token+pos     │                │              │
 │◀───────────┤                │                │              │
 │  (admitted in batch)        │                │              │
 ├── view map ────────────────▶│  (stale-OK)    │              │
 │◀── seats ───────────────────┤                │              │
 │  select A1 ─────────────────────────────────▶│ CAS AVAIL→HELD (TTL 10m)
 │◀── held (you have 10:00) ────────────────────┤              │
 │  pay ───────────────────────────────────────────────────── ▶│ charge (idempotent)
 │                             │                │◀── success ──┤
 │                             │   re-validate hold, HELD→BOOKED│
 │◀── confirmed! tickets issued ────────────────┤              │
 │                                              │
 │  ── OR: no payment in 10 min ──▶ TTL expires: HELD→AVAILABLE, seat released
```

That is the full picture: the queue admits you, the cached map lets you browse, the atomic CAS holds your seat with a countdown, the idempotent payment flips it to BOOKED (or the hold expires and the seat goes back on sale). One seat, one buyer, every time.
