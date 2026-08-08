# 02 — Ticket Booking / Seat Reservation
## When the user picks a *specific* row, and `SKIP LOCKED` stops working

> **Read the brief. Close the file. Design it yourself. Then come back.**

---

## WHY THIS CASE STUDY EXISTS

Case study 01 solved extreme contention with `FOR UPDATE SKIP LOCKED`. It worked because **every unit was interchangeable** — any free phone would do.

Here, it doesn't work. The user is staring at a seat map and has clicked **seat H14**. You cannot hand them H15 because H14 was locked. The whole technique is unavailable, and you need a genuinely different design.

This is the most important lesson in the folder: **techniques are not universal. The property that made `SKIP LOCKED` correct was *interchangeability*, and this problem doesn't have it.**

---

## 1. THE BRIEF

A ticketing platform (think BookMyShow / Ticketmaster).

> **A stadium concert. 68,000 seats. Booking opens at 10:00:00 IST.**
> **~2,000,000 users in the queue. 350,000 hit the seat map in the first minute.**

Business rules:

1. **No double-booking.** One seat, one ticket. Ever.
2. **The user picks specific seats** — up to 6 at a time, from a live seat map.
3. **Hold-then-pay.** Selecting seats creates an 8-minute hold. Payment takes 10–120 s (UPI, netbanking, cards) and often fails.
4. **The seat map must be live-ish.** A user should not spend 30 seconds choosing a seat that was taken 25 seconds ago. Target: under 3 seconds stale.
5. **Groups are atomic.** 6 seats together, or none. A partial booking is worse than a failure.
6. **Abandoned holds must return to the pool** — a permanently locked seat is lost revenue and a support ticket.
7. **There are seat categories** with different prices, and some events have unreserved standing sections (interchangeable — note this: *the same event has both models*).

Infrastructure:
- PostgreSQL 16, primary + 3 replicas, 32 vCPU / 256 GB
- 60 Node.js pods, PgBouncer transaction mode
- Redis cluster
- Peak: ~180,000 req/s at the seat map, ~9,000 req/s at seat selection

---

## 2. ACCESS PATTERN TABLE

| # | Operation | Peak rate | Rows read | Rows written | p99 budget | Staleness tolerated |
|---|---|---|---|---|---|---|
| A | `GET /events/:id/seatmap` — full map, 68k seats | 180,000/s | 68,000 | 0 | 200 ms | **3 s** |
| B | `GET /events/:id/section/:s` — one section, ~800 seats | 40,000/s | 800 | 0 | 100 ms | 3 s |
| C | `POST /hold` — hold 1–6 **specific** seats, atomically | 9,000/s | 6 | 7 | 400 ms | **0** |
| D | `POST /confirm` — payment succeeded | 600/s | 8 | 9 | 600 ms | 0 |
| E | Expiry sweeper | every 5 s | ~2,000 | ~2,000 | n/a | 0 |
| F | `GET /bookings/:id` | 12,000/s | 3 | 0 | 150 ms | 5 s (replica) |
| G | Unreserved standing section — "give me 4 of anything" | 3,000/s | 4 | 5 | 300 ms | 0 |

**What jumps out:**

- **A is 180,000/s reading 68,000 rows each.** That is 12.2 *billion* rows/second if served from PostgreSQL. Obviously impossible. It must be a single cached blob, and the design problem is *how to invalidate it fast enough*.
- **C is the hard one.** 9,000/s, each touching 6 *specific* rows atomically, zero staleness. `SKIP LOCKED` is unusable.
- **G is case study 01's problem** living inside the same system. You will need *both* techniques, chosen per section type.

---

## 3. THE INVARIANTS

```
I1.  A seat has at most ONE active hold or booking at any instant.
     ── the hard one. Must be enforced by the database.

I2.  A hold covers ALL requested seats or NONE (atomicity across 6 rows).

I3.  A hold older than 8 minutes is released, and its seats become
     immediately available.

I4.  A confirmed payment ALWAYS corresponds to confirmed seats.
     No money without a seat; no seat without money.

I5.  Seat state transitions are one-way per cycle:
       free → held → booked          (success)
       free → held → free            (expiry/cancel)
     A booked seat NEVER returns to free except by an explicit,
     audited refund operation.
```

**I1 is the one that decides the whole design.** Note carefully what it is *not*: it is not "count of bookings ≤ 68,000." It is a per-seat uniqueness property. That difference is exactly why case study 01's counter-splitting doesn't apply.

---

## 4. ATTEMPT 1 — THE NAIVE SCHEMA

```sql
CREATE TABLE events (
  id bigserial PRIMARY KEY,
  name text NOT NULL,
  venue_id bigint NOT NULL,
  starts_at timestamptz NOT NULL
);

CREATE TABLE seats (
  id          bigserial PRIMARY KEY,
  event_id    bigint NOT NULL REFERENCES events(id),
  section     text   NOT NULL,        -- 'LOWER_A'
  row_label   text   NOT NULL,        -- 'H'
  seat_no     int    NOT NULL,        -- 14
  category_id bigint NOT NULL,
  status      text   NOT NULL DEFAULT 'free'
              CHECK (status IN ('free','held','booked')),
  held_by     bigint NULL,
  held_until  timestamptz NULL,
  UNIQUE (event_id, section, row_label, seat_no)
);

CREATE TABLE bookings (
  id bigserial PRIMARY KEY,
  event_id bigint NOT NULL,
  user_id  bigint NOT NULL,
  state    text NOT NULL DEFAULT 'held',
  created_at timestamptz NOT NULL DEFAULT now(),
  expires_at timestamptz NOT NULL
);

CREATE TABLE booking_seats (
  booking_id bigint NOT NULL REFERENCES bookings(id),
  seat_id    bigint NOT NULL REFERENCES seats(id),
  PRIMARY KEY (booking_id, seat_id)
);
```

```js
// ATTEMPT 1 — correct at low traffic, fatal at high traffic
async function hold(userId, eventId, seatIds) {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');

    // lock exactly these seats
    const { rows } = await client.query(
      `SELECT id, status FROM seats WHERE id = ANY($1) FOR UPDATE`, [seatIds]);

    if (rows.length !== seatIds.length || rows.some(r => r.status !== 'free')) {
      await client.query('ROLLBACK');
      return { ok: false, reason: 'UNAVAILABLE' };
    }

    const b = await client.query(
      `INSERT INTO bookings (event_id, user_id, expires_at)
       VALUES ($1,$2, now() + interval '8 minutes') RETURNING id`, [eventId, userId]);

    await client.query(
      `UPDATE seats SET status='held', held_by=$1, held_until=now()+interval '8 minutes'
        WHERE id = ANY($2)`, [userId, seatIds]);
    await client.query(
      `INSERT INTO booking_seats (booking_id, seat_id)
       SELECT $1, unnest($2::bigint[])`, [b.rows[0].id, seatIds]);

    await client.query('COMMIT');
    return { ok: true, bookingId: b.rows[0].id };
  } catch (e) { await client.query('ROLLBACK'); throw e; }
  finally { client.release(); }
}
```

This satisfies I1–I5. It is the right *logical* design. Now watch it die.

---

## 5. WHERE IT BREAKS

### 5.1 Failure one: the seat map (pattern A) is impossible

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, section, row_label, seat_no, status FROM seats WHERE event_id = 4102;
```
```
Seq Scan on seats (actual time=0.3..41.2 rows=68000 loops=1)
  Buffers: shared hit=1204 read=3891
Execution Time: 44.8 ms
```

44.8 ms × 180,000 req/s = **8,064 CPU-seconds per second.** You would need 8,000 cores. Even with a perfect index it's ~68,000 tuples deformed per request.

There is no index, no replica count, and no instance size that makes this work. **Pattern A must never reach PostgreSQL.**

### 5.2 Failure two: deadlocks in pattern C

This is the failure that actually surprises people. Two users pick overlapping seats:

```
 User X picks: H14, H15, H16     → seat_ids [4471, 4472, 4473]
 User Y picks: H16, H15          → seat_ids [4473, 4472]

 PostgreSQL locks rows in the order the executor encounters them, which for
 `WHERE id = ANY($1)` depends on the plan — often physical/index order,
 but NOT guaranteed and NOT the array order.

 t1  X: locks 4471
 t2  Y: locks 4473
 t3  X: locks 4472
 t4  Y: waits for 4472  (held by X)
 t5  X: waits for 4473  (held by Y)          ── DEADLOCK
```

```
ERROR:  deadlock detected
DETAIL:  Process 8891 waits for ShareLock on transaction 412009;
         blocked by process 8903.
         Process 8903 waits for ShareLock on transaction 412004;
         blocked by process 8891.
HINT:  See server log for query details.
CONTEXT:  while locking tuple (204,17) in relation "seats"
```

Measure it:

```bash
cat > /tmp/hold.sql <<'EOF'
\set s1 random(1, 400)
\set s2 random(1, 400)
\set s3 random(1, 400)
BEGIN;
SELECT id FROM seats WHERE id IN (:s1, :s2, :s3) FOR UPDATE;
UPDATE seats SET status='held' WHERE id IN (:s1,:s2,:s3) AND status='free';
COMMIT;
EOF
pgbench -f /tmp/hold.sql -c 100 -j 8 -T 30 shop
```
```
number of transactions actually processed: 39104
number of failed transactions: 3187 (7.5%)     ← deadlocks + serialization
tps = 1290.2
latency average = 77.4 ms
```

**7.5% of requests fail with a deadlock error**, which the user sees as "something went wrong." Deadlock detection also isn't free: PostgreSQL waits `deadlock_timeout` (1 s default) before even *checking*, so a deadlocked pair burns a full second of two connections before one is killed.

### 5.3 Failure three: the hot section

Seats are not chosen uniformly. Everyone wants the front. 80% of pattern-C traffic targets ~15% of seats.

```sql
SELECT section, count(*) FILTER (WHERE status <> 'free') AS taken, count(*) AS total
FROM seats WHERE event_id=4102 GROUP BY section ORDER BY 2 DESC;
```
```
   section   | taken | total
-------------+-------+-------
 LOWER_A     |  1180 |  1200      ← 98% gone in 40 seconds
 LOWER_B     |  1104 |  1200
 UPPER_K     |     8 |  2400      ← untouched
```

So the effective contention is far worse than "9,000 requests over 68,000 seats." It's 7,200 requests/sec over ~10,000 desirable seats, with heavy overlap. Deadlock probability rises with the square of overlap.

### 5.4 Failure four: the expiry sweeper stalls everything

```sql
UPDATE seats SET status='free', held_by=NULL, held_until=NULL
WHERE event_id=4102 AND status='held' AND held_until < now();
```

At peak, 8 minutes after open, ~40,000 holds expire in a burst. This single statement:
- takes 40,000 row locks,
- holds them for the ~4 seconds it takes to run,
- blocks every user trying to hold any of those seats,
- and creates 40,000 dead tuples in one shot.

### 5.5 Failure five: MVCC bloat

Each seat is updated at least twice per booking cycle (`free→held`, `held→booked` or `held→free`), and popular seats cycle repeatedly as holds expire.

```sql
SELECT n_tup_upd, n_tup_hot_upd, n_dead_tup, pg_size_pretty(pg_relation_size('seats'))
FROM pg_stat_user_tables WHERE relname='seats';
```
```
 n_tup_upd | n_tup_hot_upd | n_dead_tup | pg_size_pretty
-----------+---------------+------------+----------------
    412000 |         48000 |     364000 | 402 MB
```
68,000 seats occupying 402 MB, with only 12% HOT updates — because `status` is indexed, so almost every update rewrites index entries too.

---

## 6. ATTEMPT 2 — THE FIX THAT ISN'T ENOUGH

**Fix the deadlocks with deterministic lock ordering.** This is the correct, standard answer to deadlocks (Topic 48): if every transaction acquires locks in the same global order, a cycle is impossible.

```js
const ordered = [...seatIds].sort((a, b) => (a < b ? -1 : 1));   // ALWAYS ascending
await client.query(
  `SELECT id, status FROM seats WHERE id = ANY($1) ORDER BY id FOR UPDATE`, [ordered]);
```

⚠ **`ORDER BY id` in the SELECT is essential, not decorative.** Sorting the array in JavaScript is not enough — PostgreSQL locks rows in the order the *executor* produces them. Without `ORDER BY`, a bitmap heap scan returns rows in physical order and your careful sort is ignored.

Measure:

```
number of transactions actually processed: 71204
number of failed transactions: 12 (0.02%)      ← deadlocks essentially gone
tps = 2373.5
latency average = 42.1 ms
```

**Deadlocks solved. Throughput doubled. And it is still nowhere near enough:**

- 2,373 tps against 9,000 req/s of demand.
- Pattern A (the seat map) is completely unaddressed and is the bigger problem by two orders of magnitude.
- The sweeper still stalls.
- Bloat still grows.

And there's a subtler problem the lock ordering *introduced*: users now **queue** instead of failing. A popular seat has a FIFO queue of 400 transactions, each waiting for the one before. Latency for the hot seats climbs to seconds, and every waiter holds a connection.

**The real insight:** the naive design makes the *seat row itself* the coordination point. That's fine for correctness and terrible for concurrency. The production design keeps the seat row as the **source of truth** but stops making 9,000 requests/sec fight over it.

---

## 7. THE PRODUCTION DESIGN

Four structural changes:

```
 PROBLEM                              SOLUTION                        TOPIC
 ────────────────────────────────────────────────────────────────────────
 180k/s reading 68k rows        →  Precomputed bitmap blob in Redis    57
                                   + event-driven invalidation
 Deadlocks on multi-seat holds  →  Deterministic ordering (kept)       48
                                   + short transactions
 I1 enforced by app logic       →  EXCLUSION CONSTRAINT on time range  24
                                   — the DB makes double-booking
                                     structurally impossible
 Sweeper stalls / bloat         →  Batched sweeper + separate hot
                                   table + fillfactor                  04, 47
```

### 7.1 The schema

```sql
-- ═══════════════════════════════════════════════════════════════════
-- SEATS — now IMMUTABLE reference data. Never updated during a sale.
-- ═══════════════════════════════════════════════════════════════════
CREATE TABLE seats (
  id          bigserial PRIMARY KEY,
  event_id    bigint  NOT NULL REFERENCES events(id),
  section_id  bigint  NOT NULL REFERENCES sections(id),
  row_label   text    NOT NULL,
  seat_no     int     NOT NULL,
  category_id bigint  NOT NULL REFERENCES seat_categories(id),
  map_index   int     NOT NULL,          -- 0..67999, position in the bitmap
  UNIQUE (event_id, section_id, row_label, seat_no),
  UNIQUE (event_id, map_index)
);
-- ★ NO `status` COLUMN. This is the key move. Status is derived from
--   seat_holds, so `seats` is never written during a sale → zero bloat,
--   fully cacheable, safe to read from any replica.

-- ═══════════════════════════════════════════════════════════════════
-- SEAT HOLDS — where all the churn lives, isolated from `seats`
-- ═══════════════════════════════════════════════════════════════════
CREATE EXTENSION IF NOT EXISTS btree_gist;

CREATE TABLE seat_holds (
  id          bigserial   PRIMARY KEY,
  seat_id     bigint      NOT NULL REFERENCES seats(id),
  booking_id  bigint      NOT NULL,
  state       smallint    NOT NULL DEFAULT 1,   -- 1=held 2=booked 3=released
  valid       tstzrange   NOT NULL,             -- [held_at, expires_at)
  created_at  timestamptz NOT NULL DEFAULT now(),

  -- ★★★ THE INVARIANT, ENFORCED BY THE ENGINE ★★★
  -- No two ACTIVE holds on the same seat may have overlapping validity.
  -- This is I1, made structurally impossible to violate. Not a check in
  -- application code — an index the engine consults on every insert.
  EXCLUDE USING gist (
    seat_id WITH =,
    valid   WITH &&
  ) WHERE (state IN (1, 2))
) WITH (fillfactor = 80);

CREATE INDEX idx_holds_expiry ON seat_holds (upper(valid))
  WHERE state = 1;
CREATE INDEX idx_holds_booking ON seat_holds (booking_id);

-- ═══════════════════════════════════════════════════════════════════
-- BOOKINGS
-- ═══════════════════════════════════════════════════════════════════
CREATE TABLE bookings (
  id              bigserial   PRIMARY KEY,
  event_id        bigint      NOT NULL REFERENCES events(id),
  user_id         bigint      NOT NULL REFERENCES users(id),
  state           smallint    NOT NULL DEFAULT 1,  -- 1=held 2=paid 3=expired 4=cancelled
  seat_count      smallint    NOT NULL CHECK (seat_count BETWEEN 1 AND 6),
  total_paise     bigint      NOT NULL CHECK (total_paise > 0),
  idempotency_key uuid        NOT NULL,
  created_at      timestamptz NOT NULL DEFAULT now(),
  expires_at      timestamptz NOT NULL,
  payment_id      text        NULL
);
CREATE UNIQUE INDEX uq_bookings_idem ON bookings (idempotency_key);
CREATE INDEX idx_bookings_user ON bookings (user_id, created_at DESC);
CREATE INDEX idx_bookings_expiry ON bookings (expires_at) WHERE state = 1;

-- ═══════════════════════════════════════════════════════════════════
-- UNRESERVED SECTIONS — case study 01's model, in the same system
-- ═══════════════════════════════════════════════════════════════════
CREATE TABLE ga_allocations (
  event_id   bigint   NOT NULL,
  section_id bigint   NOT NULL,
  slot_no    int      NOT NULL,
  shard      smallint NOT NULL,
  state      smallint NOT NULL DEFAULT 0,
  booking_id bigint   NULL,
  held_until timestamptz NULL,
  PRIMARY KEY (event_id, section_id, slot_no)
) WITH (fillfactor = 70);
CREATE INDEX idx_ga_free ON ga_allocations (event_id, section_id, shard, slot_no)
  WHERE state = 0;
-- ★ Interchangeable → FOR UPDATE SKIP LOCKED applies here, and ONLY here.
```

### 7.2 Why the EXCLUSION constraint is the centrepiece

This is the single most valuable thing in this case study, and most developers have never used it.

```sql
EXCLUDE USING gist (seat_id WITH =, valid WITH &&) WHERE (state IN (1,2))
```

Read it as: *"there may not exist two rows where `seat_id` is **equal** AND `valid` **overlaps**, among rows where state is held or booked."*

```
 WHAT IT DOES AT THE ENGINE LEVEL
 ─────────────────────────────────────────────────────────────────────
 It builds a GiST index. On every INSERT/UPDATE, before the row is made
 visible, the engine searches that index for a conflicting entry — inside
 the index insert itself, holding the appropriate page locks.

 There is NO WINDOW. Unlike:
     SELECT ... WHERE seat_id=$1 AND status='free';   ← gap here
     UPDATE seats SET status='held' ...;              ← another txn snuck in

 Two concurrent inserts for seat H14:
     txn A: INSERT hold(seat 4471, [10:00:00, 10:08:00))  → succeeds
     txn B: INSERT hold(seat 4471, [10:00:01, 10:08:01))  → BLOCKS on A,
            then on A's commit:
            ERROR: conflicting key value violates exclusion constraint
                   "seat_holds_seat_id_valid_excl"

 ⇒ I1 CANNOT be violated. Not by a race, not by a bug in a new code path,
   not by someone running a manual UPDATE in psql at 2am. The database
   refuses.
```

**And the time-range dimension is what makes expiry clean.** A released hold's range has already ended, so it no longer overlaps a new hold — you don't even need to delete it. The seat becomes available *by the passage of time*, not by a state update. That's an elegant property worth internalising: **modelling validity as a range turns "expiry" from an operation into a consequence.**

```
 seat 4471 timeline
 ──────────────────────────────────────────────────────────────────▶ t
   hold#1 [10:00 ──────── 10:08)          state=1, expired
                          hold#2 [10:08 ──────── 10:16)   state=1 active
                                                      ↑
                        a new hold starting at 10:09 CONFLICTS (overlaps)
                        a new hold starting at 10:17 does NOT
```

### 7.3 The hold operation

```js
const HOLD_MINUTES = 8;

async function holdSeats({ userId, eventId, seatIds, idempotencyKey }) {
  if (seatIds.length < 1 || seatIds.length > 6) return { status: 400 };

  // ── LAYER 1: Redis pre-check. Cheap rejection of obviously-taken seats.
  //    NOT authoritative — just avoids a doomed database round trip.
  const taken = await redis.eval(CHECK_BITMAP, 1, `event:${eventId}:taken`,
                                 ...seatIds.map(String));
  if (taken.length) return { status: 409, body: { unavailable: taken } };

  const ordered = [...seatIds].map(Number).sort((a, b) => a - b);   // deadlock-free order

  const client = await pool.connect();
  try {
    await client.query('BEGIN');
    await client.query("SET LOCAL lock_timeout = '400ms'");

    const price = await client.query(
      `SELECT sum(c.price_paise) AS total FROM seats s
         JOIN seat_categories c ON c.id = s.category_id
        WHERE s.id = ANY($1) AND s.event_id = $2`, [ordered, eventId]);

    const booking = await client.query(
      `INSERT INTO bookings (event_id, user_id, state, seat_count, total_paise,
                             idempotency_key, expires_at)
       VALUES ($1,$2,1,$3,$4,$5, now() + ($6||' minutes')::interval)
       ON CONFLICT (idempotency_key) DO NOTHING
       RETURNING id, expires_at`,
      [eventId, userId, ordered.length, price.rows[0].total, idempotencyKey, HOLD_MINUTES]);

    if (booking.rowCount === 0) {          // retry of the same request
      await client.query('ROLLBACK');
      const prev = await pool.query(
        'SELECT id, expires_at FROM bookings WHERE idempotency_key=$1', [idempotencyKey]);
      return { status: 200, body: prev.rows[0] };     // idempotent
    }

    const bookingId = booking.rows[0].id;
    const expires   = booking.rows[0].expires_at;

    // ★ ONE statement, seats in ascending order. The EXCLUDE constraint
    //   makes this atomically all-or-nothing: if ANY seat conflicts, the
    //   whole INSERT fails and the transaction rolls back. I2 for free.
    await client.query(
      `INSERT INTO seat_holds (seat_id, booking_id, state, valid)
       SELECT s, $2, 1, tstzrange(now(), $3, '[)')
         FROM unnest($1::bigint[]) WITH ORDINALITY AS t(s, ord)
        ORDER BY s`,
      [ordered, bookingId, expires]);

    await client.query('COMMIT');

    // ── LAYER 3: publish the invalidation. Best-effort; the sweeper reconciles.
    await redis.eval(MARK_TAKEN, 1, `event:${eventId}:taken`,
                     ...ordered.map(String), String(Math.floor(Date.parse(expires)/1000)));
    await redis.publish(`event:${eventId}:changed`, JSON.stringify({ seats: ordered }));

    return { status: 201, body: { bookingId, seats: ordered, expiresAt: expires } };

  } catch (e) {
    await client.query('ROLLBACK').catch(() => {});
    if (e.code === '23P01') {   // exclusion_violation — someone got there first
      return { status: 409, body: { error: 'SEATS_TAKEN' } };
    }
    if (e.code === '55P03' || e.code === '40P01') {   // lock_timeout / deadlock
      return { status: 503, body: { error: 'RETRY' } };
    }
    throw e;
  } finally { client.release(); }
}
```

**Note the error handling.** `23P01` (exclusion violation) is not a bug — it is the *expected* outcome when two users want the same seat, and it is exactly the outcome you want: fast, deterministic, and impossible to get wrong. The application's job is to translate it into a good user message, not to prevent it.

### 7.4 The seat map — 180,000 req/s without touching PostgreSQL

The map is 68,000 seats × 3 states. Encode it as a **bitmap**: 2 bits per seat = 17 KB, or with gzip ~2 KB.

```
 REDIS KEY LAYOUT
 ────────────────────────────────────────────────────────────────
 event:4102:bitmap     a 17 KB binary string; 2 bits per map_index
                       00 = free, 01 = held, 10 = booked
 event:4102:version    monotonically increasing; bumped on every change
 event:4102:taken      a Redis BITMAP for the fast pre-check in holdSeats

 SERVING PATTERN A:
   CDN (2s TTL) ──▶ 60 Node pods (in-process cache, 1s TTL) ──▶ Redis
                     │
                     └─ 99.98% served from the in-process cache.
                        Redis sees ~60 req/s. PostgreSQL sees ZERO.
```

```js
// In-process cache, refreshed by a Redis pub/sub subscription
let bitmapCache = new Map();   // eventId -> { buf, version, at }

redisSub.subscribe('event:*:changed');
redisSub.on('message', async (channel) => {
  const eventId = channel.split(':')[1];
  bitmapCache.delete(eventId);            // next request refetches
});

async function getSeatMap(eventId) {
  const c = bitmapCache.get(eventId);
  if (c && Date.now() - c.at < 1000) return c.buf;      // ≤1 s stale
  const buf = await redis.getBuffer(`event:${eventId}:bitmap`);
  bitmapCache.set(eventId, { buf, at: Date.now() });
  return buf;
}
```

**The reconciler — because Redis is allowed to be wrong, and PostgreSQL is not:**

```sql
-- Every 3 seconds, per active event. Rebuilds the authoritative bitmap.
SELECT s.map_index,
       CASE WHEN h.state = 2 THEN 2
            WHEN h.state = 1 THEN 1
            ELSE 0 END AS st
FROM seats s
LEFT JOIN LATERAL (
  SELECT state FROM seat_holds h
   WHERE h.seat_id = s.id AND h.state IN (1,2) AND h.valid @> now()
   LIMIT 1
) h ON true
WHERE s.event_id = $1
ORDER BY s.map_index;
```

That query runs **20 times/second across all events**, not 180,000. It reads a table that is never updated (`seats`) joined to a small index-driven lookup. On a replica, it costs nothing.

**The staleness budget is explicit:** ≤1 s in-process + ≤3 s reconciler = **≤4 s worst case**, against a requirement of 3 s typical. Users occasionally click a seat that's just been taken and get a clean 409. That is a designed, accepted outcome — not a bug.

### 7.5 The sweeper — batched, and mostly unnecessary

Because validity is a time range, **an expired hold stops blocking new holds automatically.** The sweeper exists only to update `state` for reporting and to publish invalidations.

```sql
-- Every 5 seconds. LIMIT is mandatory. SKIP LOCKED is safe here because
-- expired holds ARE interchangeable from the sweeper's point of view.
WITH victims AS (
  SELECT id, seat_id, booking_id FROM seat_holds
   WHERE state = 1 AND upper(valid) < now()
   ORDER BY upper(valid)
   LIMIT 500
   FOR UPDATE SKIP LOCKED
), released AS (
  UPDATE seat_holds h SET state = 3
    FROM victims v WHERE h.id = v.id
  RETURNING h.seat_id, h.booking_id
)
UPDATE bookings b SET state = 3
  FROM (SELECT DISTINCT booking_id FROM released) r
 WHERE b.id = r.booking_id AND b.state = 1
RETURNING b.id;
```

### 7.6 Confirmation

```js
async function confirmBooking({ bookingId, paymentId, signature }) {
  if (!verifyPaymentSignature(paymentId, signature)) return { status: 401 };

  return withTransaction(async (tx) => {
    // Only a still-valid held booking transitions. Idempotent by construction.
    const b = await tx.query(
      `UPDATE bookings SET state = 2, payment_id = $2
        WHERE id = $1 AND state = 1 AND expires_at > now()
       RETURNING id, event_id, user_id`, [bookingId, paymentId]);

    if (b.rowCount === 0) {
      const cur = await tx.query('SELECT state FROM bookings WHERE id=$1', [bookingId]);
      if (cur.rows[0]?.state === 2) return { status: 200 };            // replay
      return { status: 409, body: { error: 'HOLD_EXPIRED', refund: true } };
    }

    // Extend validity to infinity and mark booked. The EXCLUDE constraint
    // now permanently reserves these seats.
    await tx.query(
      `UPDATE seat_holds
          SET state = 2, valid = tstzrange(lower(valid), 'infinity', '[)')
        WHERE booking_id = $1 AND state = 1`, [bookingId]);

    await tx.query(
      `INSERT INTO tickets (booking_id, seat_id, qr_token)
       SELECT $1, seat_id, encode(gen_random_bytes(16),'hex')
         FROM seat_holds WHERE booking_id = $1 AND state = 2`, [bookingId]);

    await tx.query(
      `INSERT INTO outbox (topic, payload) VALUES ('booking.confirmed', $1)`,
      [JSON.stringify({ bookingId })]);

    return { status: 200 };
  });
}
```

`valid = tstzrange(lower(valid), 'infinity', '[)')` is the neat part: a booked seat's hold never expires, so it permanently excludes any future hold — using the exact same constraint, with no special-casing.

---

## 8. THE NUMBERS

| Metric | Attempt 1 | Attempt 2 (ordered locks) | Production |
|---|---|---|---|
| Seat map (A) served by | PostgreSQL, 45 ms | PostgreSQL, 45 ms | **in-process cache, 0.4 ms** |
| Seat map load on PostgreSQL | 180,000 req/s (impossible) | same | **0** |
| Holds/sec (C) | 1,290 | 2,373 | **8,900** |
| Deadlock/conflict failure rate | 7.5% | 0.02% | **0** deadlocks; 1.8% clean 409s |
| p99 hold latency | 690 ms | 210 ms | **34 ms** |
| Double-bookings | 0 (by luck + locking) | 0 | **0 (structurally impossible)** |
| `seats` table bloat | 402 MB for 68k rows | 402 MB | **11 MB, never updated** |
| Sweeper stall | 4 s blocking | 4 s blocking | **<40 ms, batched** |
| Time to sell 68k seats | n/a — system down | ~9 min | **~2.5 min** |

The 1.8% clean 409s are *the correct behaviour*: two people wanted the same seat, one lost, and they were told immediately and accurately. Attempt 1's 7.5% were `deadlock detected` — a database internal error surfaced to a user who did nothing wrong.

---

## 9. FAILURE MODES AND WHAT YOU MONITOR

| Failure | What happens | Design response |
|---|---|---|
| **Redis dies** | Seat map falls back; pre-check unavailable | Pods serve their last in-process bitmap (stale but usable) and the DB pre-check is skipped — holds still work, just with more 409s. **Never fall back to querying PostgreSQL for the map**; that's the outage |
| **Reconciler dies** | Bitmap freezes; users click taken seats | Alert on `event:*:version` not advancing for 15 s. Users get clean 409s meanwhile — degraded, not broken |
| **Pod cache skew** | Two pods show different maps | Bounded by the 1 s TTL. Acceptable and explicitly in the staleness budget |
| **Payment succeeds after hold expiry** | Money taken, seats gone | `confirmBooking` returns `HOLD_EXPIRED, refund: true`. **Automatic refund path is mandatory** — this WILL happen at scale, and manual handling doesn't survive 600/s |
| **Clock skew between pods** | `now()` differs | ★ All time comes from **PostgreSQL's `now()`**, never from Node's `Date.now()`. Ranges are computed server-side. This is why `expires_at` is returned from the INSERT rather than computed in JS |
| **Duplicate confirm webhook** | Double tickets | `UPDATE ... WHERE state = 1` transitions once; replays return 200 |
| **Sweeper lags during the burst** | Expired holds still shown as held | Harmless — the EXCLUDE constraint already lets a new hold take an expired seat. The bitmap is stale for a few seconds |
| **A section sells out** | Users hammer a dead section | Pre-check bitmap returns "all taken"; zero DB load. Add a `section_available` flag in Redis for an even earlier exit |

**The four alerts:**

```sql
-- 1. Exclusion violations spiking = extreme contention or a bitmap problem
--    (track from application error codes, rate per second)
--    ALERT if 23P01 rate > 15% of hold attempts for 30 s

-- 2. Sweeper backlog
SELECT count(*) FROM seat_holds WHERE state=1 AND upper(valid) < now() - interval '60 seconds';
-- ALERT if > 1000

-- 3. Orphaned money: paid bookings with no booked holds
SELECT count(*) FROM bookings b WHERE b.state=2
  AND NOT EXISTS (SELECT 1 FROM seat_holds h WHERE h.booking_id=b.id AND h.state=2);
-- ALERT if > 0.  This is I4. It should always be zero.

-- 4. Lock waits (should be near zero in this design)
SELECT count(*) FROM pg_stat_activity WHERE wait_event_type='Lock';
-- ALERT if > 15 for 10 s
```

---

## 10. WHAT BREAKS AT 10×

**680,000 seats (a festival with many stages), 20M users, 90,000 holds/sec.**

| Wall | Why | Next design |
|---|---|---|
| The 17 KB bitmap | 680k seats × 2 bits = 170 KB per event; ×1000 concurrent events = too much to push on every change | **Section-level bitmaps.** Clients fetch only the section they're viewing. Invalidate per section, not per event |
| GiST exclusion index writes | 90k inserts/sec into one GiST index — GiST is more expensive to maintain than B-tree | **Partition `seat_holds` by `event_id`** (hash or list). Each partition gets its own exclusion index, so contention and index depth both drop |
| Single primary write capacity | ~20k write txns/sec ceiling | **Shard by `event_id`.** Like case study 01, an event never spans shards, so this is unusually clean sharding (Topic 60) |
| Redis pub/sub fan-out | 60 pods × 1000 events × changes/sec | Move to per-section Redis Streams with consumer groups, or push invalidation via the CDN's purge API |
| The waiting room | 20M users arriving at 10:00:00 | This stops being a database problem. **Virtual queue**: signed admission tokens with a timestamp, issued at a controlled rate. The database only ever sees admitted traffic |

---

## 11. THE SEVEN QUESTIONS

1. Why does `FOR UPDATE SKIP LOCKED` work in case study 01 but not here? Name the exact property that differs, and name the one part of *this* system where it does apply.
2. Explain the `EXCLUDE USING gist (seat_id WITH =, valid WITH &&)` constraint in plain English. Why does it have no race window when `SELECT`-then-`UPDATE` does?
3. Sorting `seatIds` in JavaScript is not sufficient to prevent deadlocks. What else is required, and why?
4. Why does the production design remove the `status` column from `seats`? Name three separate benefits.
5. Modelling validity as a `tstzrange` makes expiry "a consequence rather than an operation." Explain what that means and what it saves you.
6. The seat map is up to 4 seconds stale, and users sometimes get a 409 for a seat that looked free. Why is this the correct design rather than a bug to fix?
7. Why must `now()` come from PostgreSQL and never from Node.js in this design? Give the concrete failure.

---

## 12. TRANSFERABLE LESSONS

**1. `SKIP LOCKED` requires interchangeability. Check for it explicitly.**
Before reaching for it, ask: *"if I hand the caller a different row, is that acceptable?"* Jobs in a queue: yes. A seat, a specific account, a named resource: no. Applying it where items aren't interchangeable silently changes your semantics. → Contrast with **01**; reappears in **09 (job claiming — yes)**, **16 (specific SKU at a specific warehouse — no)**.

**2. Exclusion constraints are the most underused feature in PostgreSQL.**
Any invariant of the form *"no two rows may share X while overlapping in Y"* — seat + time, room + booking window, employee + shift, IP range + tenant — is one `EXCLUDE USING gist` away from being impossible to violate. This is strictly stronger than application logic and strictly stronger than a unique index. → Reappears in **16 (allocation windows)**, **17 (temporal records)**, **04 (billing periods)**.

**3. Separate the immutable from the churning.**
`seats` never changes; `seat_holds` changes constantly. Splitting them means the big table is bloat-free, replica-safe, and infinitely cacheable, while the churn is confined to a small table you can tune aggressively. → Reappears in **12 (order vs order_status)**, **19 (video metadata vs view counts)**.

**4. Model validity as a range, not a state plus a timestamp.**
`state='held' AND held_until > now()` requires you to sweep. `valid tstzrange` expires itself. The sweeper becomes a bookkeeping nicety instead of a correctness requirement — which means a sweeper outage degrades reporting, not the product. → Reappears in **17 (bitemporal records)**, **15 (subscription periods)**.

**5. A designed 409 beats an accidental 500.**
Attempt 1's deadlocks and the production design's exclusion violations are both "you lost the race." One is an internal error leaked to a user; the other is a first-class, fast, accurate business outcome. **Decide which of your failures are expected, and make them clean.**

**6. Time comes from one clock.**
Every timestamp that participates in an invariant must come from the database, not from the application. Sixty pods have sixty slightly different clocks, and NTP drift of 200 ms is enough to create overlapping "non-overlapping" ranges.

**7. The read path and the write path can have completely different architectures.**
180,000 req/s of reads served from an in-process bitmap; 9,000 req/s of writes served by a GiST exclusion constraint. Same data, same product, two entirely separate designs. Trying to serve both from one mechanism is what killed Attempt 1.

---

## FILES TO READ NEXT

- **03 — Payment ledger**: what happens after `confirmBooking`, and how money stays correct
- **24 — Constraints in depth** (curriculum): exclusion constraints, GiST, and the full argument for DB-level enforcement
- **48 — Deadlocks** (curriculum): the lock-ordering discipline used in Attempt 2
- **59 — Partitioning** (curriculum): partitioning `seat_holds` by event, the 10× answer
- **01 — Flash sale** (this folder): re-read section 12 and compare lesson 1 with lesson 3 there
