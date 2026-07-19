# Topic 63 — Date and Time Mastery
### SQL Mastery Curriculum — Phase 9: Advanced SQL Patterns

---

## 1. ELI5 — The Analogy That Makes It Click

Imagine you run a global conference call. A colleague in Tokyo writes on a sticky note: "Meeting at 3." She hands it to you in New York. You show up at 3 your time. She's furious — she meant 3 *her* time, which was 1 AM for you. The meeting was 14 hours ago.

The sticky note said "3." It did **not** say "3 in Tokyo." That missing piece of information — the time zone — is the difference between a `timestamp` and a `timestamptz` in PostgreSQL.

- A **`timestamp`** (without time zone) is the naive sticky note: "2026-07-19 15:00:00." It's just wall-clock digits. It has no idea where on Earth that clock is. When two people in two cities read it, they each assume it's *their* local time — and they're both wrong about each other.

- A **`timestamptz`** (with time zone) is the sticky note that also records "…and this was 3 PM **in Tokyo**." Internally PostgreSQL immediately converts it to a single universal reference — UTC, the world's master clock — and stores that. Now anyone, anywhere, can ask "what time is that for *me*?" and get the correct answer.

Here's the twist that trips up every engineer: `timestamptz` does **not** store the time zone you gave it. It stores a single absolute instant (as UTC) and *forgets* the original zone. When you read it back, PostgreSQL renders that instant into whatever time zone your session is currently set to. The name is a lie — it should be called "timestamp as an absolute instant." But the behavior is exactly what you want: one true moment, displayed correctly for everyone.

The whole of this topic is: always store the absolute instant (`timestamptz`), never the naive sticky note (`timestamp`), and learn the small toolbox — `AT TIME ZONE`, `date_trunc`, `extract`, `age`, intervals — for slicing time correctly and for writing date filters that the index can actually use.

---

## 2. Connection to SQL Internals

At the storage layer, both `timestamp` and `timestamptz` occupy exactly **8 bytes** in the heap tuple. They are stored identically: a 64-bit signed integer counting **microseconds since the PostgreSQL epoch, 2000-01-01 00:00:00**. There is no time-zone field on disk. This surprises people: `timestamptz` is not "bigger" than `timestamp`.

The difference is entirely in **input/output conversion**, not storage:

- On **INSERT** of a `timestamptz`, PostgreSQL reads the literal, applies the session `TimeZone` (or an explicit offset if the literal carries one), and normalizes to UTC before writing the 8-byte integer.
- On **SELECT** of a `timestamptz`, PostgreSQL reads the UTC integer and renders it in the session `TimeZone`.
- For `timestamp` (naive), **no conversion happens on the way in or out**. The digits you gave are the digits stored; the digits stored are the digits shown.

Consequences for the engine's other subsystems:

1. **B-tree indexes**: Because both types are stored as a monotonic 8-byte integer, a B-tree index on a timestamp column is a dense, perfectly-ordered structure. Range scans (`created_at >= X AND created_at < Y`) are the *canonical* B-tree range-scan workload — the planner loves them. This is why **sargability** (Section 6) matters so much: keep the column bare so the index range scan is usable.

2. **MVCC row headers**: Every heap tuple carries `xmin`/`xmax` transaction IDs, but the *commit timestamp* (if `track_commit_timestamp` is on) and the values in your own `created_at`/`updated_at` columns are ordinary user data. `now()`/`CURRENT_TIMESTAMP` return the **transaction start time**, frozen for the whole transaction — this is an MVCC-snapshot-consistency guarantee, not a wall-clock read. Two rows inserted in the same transaction get the identical `now()`.

3. **The planner and `now()`**: `now()` is `STABLE`, not `IMMUTABLE` — it returns the same value within a statement but can't be constant-folded at plan time across executions. This is why `WHERE created_at > now() - interval '7 days'` can still use an index (the value is fixed once per execution) but can't be cached into a generic plan constant.

4. **Functional immutability and indexes**: `date_trunc('day', col)` and `col AT TIME ZONE 'X'` are only usable in an index if you build an **expression index** on exactly that expression. `timestamptz AT TIME ZONE zone` is `STABLE` (its result depends on nothing mutable when zone is a literal, but the type system treats zone-dependent conversions carefully) — subtleties here decide whether an expression index is even allowed.

---

## 3. Logical Execution Order Context

Date/time expressions appear in nearly every clause, and *where* they appear changes both correctness and performance:

```
FROM / JOIN          ← time-range join conditions (e.g. event within a window)
WHERE                ← sargable range filters live here; keep the column BARE
GROUP BY             ← date_trunc('day', ts) buckets are formed here
HAVING               ← post-aggregate time filters
SELECT               ← extract(), age(), AT TIME ZONE, formatting for display
DISTINCT
window functions     ← range/time-based frames (RANGE BETWEEN interval)
ORDER BY             ← chronological ordering (often matches an index)
LIMIT
```

Key ordering facts specific to time:

- The **`WHERE` clause runs before `GROUP BY`**. So a sargable range filter on a bare `created_at` column shrinks the row set *before* any `date_trunc` bucketing happens. Filter first, bucket second — never bucket the whole table then filter buckets.

- **`date_trunc` in `GROUP BY`** transforms rows into buckets *after* `WHERE`. If you also put `date_trunc(...) = something` in `WHERE`, you've applied a function to the column in the filter and killed sargability (Section 6). Put the bucket in `GROUP BY`/`SELECT`; put a bare-column range in `WHERE`.

- **`now()` / `CURRENT_DATE` evaluate once per statement** (they're STABLE), so their value is consistent across the whole logical pipeline — the row that passes `WHERE created_at < now()` uses the same `now()` the `SELECT now()` column would show.

- **Time-zone conversion in `SELECT`** happens last, at output. Bucketing in `GROUP BY` should already have converted to the *display* zone if you want calendar-day boundaries in local time (`date_trunc('day', ts AT TIME ZONE 'America/New_York')`) — doing it in SELECT after grouping is too late.

---

## 4. What Are Date/Time Types and Operations?

PostgreSQL has a family of temporal types. `timestamptz` stores an **absolute instant** (rendered per session zone); `timestamp` stores **naive wall-clock digits** with no zone; `date` is a calendar day; `time`/`timetz` are times of day; `interval` is a **duration** (a span, not a point).

```sql
column_name  timestamptz    -- 8 bytes; absolute instant, stored as UTC, shown in session zone
             │        └── "with time zone": convert-on-in, convert-on-out. USE THIS in production.
             timestamp      -- 8 bytes; naive digits, NO zone, NO conversion ever. Avoid for events.
             date           -- 4 bytes; calendar day only, no time, no zone
             time           -- 8 bytes; time of day, no date, no zone
             interval       -- 16 bytes; a DURATION (months, days, microseconds) — not an instant
```

Core operations, annotated:

```sql
now()                                  -- current transaction start, as timestamptz (STABLE)
│  └── aliases: CURRENT_TIMESTAMP, transaction_timestamp()
│      clock_timestamp() = real wall clock, changes mid-statement (VOLATILE)
│      statement_timestamp() = start of current statement

CURRENT_DATE                           -- today's date in the SESSION time zone
CURRENT_TIME                           -- time of day with zone offset

ts + interval '1 day'                  -- date arithmetic: add a duration to an instant
ts2 - ts1                              -- subtract two timestamps → an interval

date_trunc('day', ts)                  -- floor ts to start of the unit ('hour','week','month','year'...)
│         │      └── the timestamp to truncate
│         └── the field: microseconds..millennium

extract(hour FROM ts)                  -- pull one numeric field out (also: date_part('hour', ts))
│       │    └── source timestamp/interval
│       └── field name: year, month, day, hour, dow, doy, epoch, ...

age(ts1, ts2)                          -- symbolic interval (years/months/days), calendar-aware
age(ts)                                -- age(CURRENT_DATE, ts): "how long ago", from today

ts AT TIME ZONE 'Asia/Tokyo'           -- ZONE CONVERSION operator (see Section 6.5 — direction matters!)
│  │            └── IANA zone name (preferred) or abbreviation or POSIX offset
│  └── if ts is timestamptz → returns naive timestamp AS SEEN IN that zone
│      if ts is timestamp   → returns timestamptz (interprets the naive digits AS being in that zone)

generate_series(start, stop, interval) -- produce a set of timestamps at fixed steps (time buckets)
make_timestamptz(y,mo,d,h,mi,s,zone)   -- build a timestamptz from parts
```

Two mental anchors:

- `timestamp` − `timestamp` = `interval`. `timestamp` + `interval` = `timestamp`. `interval` + `interval` = `interval`. You cannot add two instants (there is no such operation — adding two points in time is meaningless).
- `AT TIME ZONE` is an **overloaded converter** whose behavior flips depending on the input type. Master this or be forever confused (Section 6.5).

---

## 5. Why Date/Time Mastery Matters in Production

1. **The `timestamp` (naive) production disaster.** The single most expensive temporal mistake is choosing `timestamp` instead of `timestamptz` for event columns (`created_at`, `paid_at`, `deleted_at`). Everything works in dev because dev, the app server, and Postgres all sit in UTC (or all in the same zone). Then you deploy across regions, or a server's `TimeZone` GUC differs, or the app sends local times — and the *same wall-clock digits* now mean different instants on different machines. Reports double-count around midnight, "last 24 hours" silently shifts, and DST transitions create one-hour gaps or duplicates. The data is already corrupt on disk; you can't recover the lost zone. This is a resume-generating bug.

2. **Session `TimeZone` changes what users see.** Two users hitting the same row see different rendered times because their sessions differ. If you don't control the session zone explicitly (or convert in the query), your "8:00 AM report" is 8 AM for nobody in particular.

3. **Non-sargable date filters table-scan your hottest queries.** `WHERE date_trunc('day', created_at) = '2026-07-19'` or `WHERE created_at::date = CURRENT_DATE` wraps the indexed column in a function — the B-tree can't be range-scanned, so Postgres does a Seq Scan on a 100M-row table. The same logic written as a bare-column half-open range (`>= X AND < X+1day`) uses the index and returns in milliseconds. This is the difference between a 20-second and a 2-millisecond query.

4. **DST and month arithmetic are not intuitive.** Adding `interval '1 month'` to Jan 31 gives Feb 28; adding `interval '24 hours'` across a DST boundary is *not* the same as adding `interval '1 day'`. `age()` and calendar-aware intervals exist precisely because "a month" and "a day" are not fixed numbers of seconds.

5. **Time-bucket reports need gap-filling.** A dashboard that groups sales by day will silently omit days with zero sales — the line chart lies by skipping the gap. `generate_series` + `LEFT JOIN` is the correct pattern (built on Topic 62, Generating Data) and it's asked in interviews constantly.

---

## 6. Deep Technical Content

### 6.1 `timestamp` vs `timestamptz` — The Complete Picture

Set up a demonstration. Session zone is `America/New_York` (UTC−4 in July, due to DST).

```sql
SET TimeZone = 'America/New_York';

CREATE TABLE tz_demo (
  naive  timestamp,     -- no zone
  aware  timestamptz    -- with zone
);

INSERT INTO tz_demo (naive, aware)
VALUES ('2026-07-19 15:00:00', '2026-07-19 15:00:00');
```

What was stored?

- `naive`: the integer for the digits `2026-07-19 15:00:00`. No interpretation. Period.
- `aware`: the literal `2026-07-19 15:00:00` was read *as New York time* (session zone), converted to UTC → `2026-07-19 19:00:00 UTC`, and that instant's integer was stored.

Now read it back in the same session:

```sql
SELECT naive, aware FROM tz_demo;
--        naive        |          aware
-- ---------------------+------------------------
--  2026-07-19 15:00:00 | 2026-07-19 15:00:00-04
```

Both look like 15:00. But now change the session zone and read again:

```sql
SET TimeZone = 'Asia/Tokyo';   -- UTC+9
SELECT naive, aware FROM tz_demo;
--        naive        |          aware
-- ---------------------+------------------------
--  2026-07-19 15:00:00 | 2026-07-20 04:00:00+09
```

The **naive** value did not move — it's still 15:00 because it has no zone to convert from. The **aware** value re-rendered: the same absolute instant (19:00 UTC) is 04:00 the *next day* in Tokyo. This is correct and desirable: one instant, displayed per viewer. But watch what it means for the naive column — 15:00 in Tokyo is a *completely different moment* than the 15:00 you thought you stored. If your app is in New York and your reporting runs in Tokyo, every naive timestamp is off by the zone difference and you have no way to know by how much.

**Rule**: For any column representing *when something happened* (an event, an audit entry, a payment), use `timestamptz`. Use naive `timestamp` only for wall-clock concepts that are genuinely zoneless — e.g., "store opens at 09:00 local" where local is defined elsewhere, or a recurring alarm time. Even then, most teams store the zone alongside and still prefer `timestamptz` + a `zone` column.

### 6.2 How Session `TimeZone` Affects Display (and Input)

The session GUC `TimeZone` controls two things for `timestamptz`:

1. **Output rendering** — how the stored UTC instant is displayed.
2. **Input interpretation** — how a zone-less literal assigned to a `timestamptz` is read (which local zone the digits are assumed to be in).

```sql
SHOW TimeZone;                      -- inspect current session zone
SET TimeZone = 'UTC';               -- set for this session
SET TIME ZONE 'America/Los_Angeles';-- alt syntax
RESET TimeZone;                     -- back to default (postgresql.conf / per-role)
```

It does **not** affect naive `timestamp` at all (no conversion happens). And it does **not** change what's stored for `timestamptz` (still UTC on disk) — only how it's rendered/parsed at the boundary.

Where the default comes from, in precedence order: `SET` in the session > `ALTER ROLE ... SET TimeZone` > `ALTER DATABASE ... SET TimeZone` > `postgresql.conf` `timezone` > server OS zone. **In production, pin it.** Many teams run the database in `UTC` and convert explicitly in queries or in the app, so behavior is deterministic regardless of who connects.

`libpq`/node-postgres note: the driver can set the session zone via `PGTZ` or an explicit `SET TimeZone`. node-postgres by default parses `timestamptz` into a JS `Date` (an absolute instant — safe) and parses naive `timestamp` into a JS `Date` *as if it were local to the Node process* (dangerous — see Section 11).

### 6.3 Date Arithmetic and Intervals

Intervals are durations with three independent fields: **months**, **days**, and **microseconds**. They are kept separate on purpose because a month is not a fixed number of days and a day is not always 24 hours (DST).

```sql
SELECT interval '1 year 2 months 3 days 04:05:06';
-- 1 year 2 mons 3 days 04:05:06

SELECT timestamptz '2026-01-31 12:00' + interval '1 month';
-- 2026-02-28 12:00:00  ← clamped to end of Feb; NOT 2026-03-03

SELECT timestamptz '2026-03-08 01:30' + interval '1 day';   -- across US spring-forward DST
-- 2026-03-09 01:30:00  ← same wall-clock next day (calendar day)

SELECT timestamptz '2026-03-08 01:30' + interval '24 hours';-- add exactly 24h of real time
-- 2026-03-09 02:30:00  ← one hour later on the clock, because a DST day was 23h long
```

That last pair is the crux: **`+ interval '1 day'` is calendar-aware; `+ interval '24 hours'` is absolute.** Adding a "day" preserves wall-clock time across DST; adding "24 hours" preserves elapsed physical time. Choose deliberately.

Subtraction of two timestamps yields an interval measured in **days + time** (it does *not* try to be calendar-symbolic — that's `age()`'s job):

```sql
SELECT timestamptz '2026-03-01' - timestamptz '2026-01-01';
-- 59 days   ← raw span (Jan has 31, Feb has 28 in 2026)

SELECT age(timestamptz '2026-03-01', timestamptz '2026-01-01');
-- 2 mons    ← calendar-symbolic: "two months"
```

Multiplying/dividing intervals and mixing with numbers:

```sql
SELECT interval '1 hour' * 3;         -- 03:00:00
SELECT interval '30 days' / 2;        -- 15 days
SELECT 2 * interval '1 day 12:00';    -- 3 days
SELECT extract(epoch FROM interval '1 day 01:00:00'); -- 90000  (seconds in the span)
```

Building intervals from a number (common when the count is a parameter — you can't parameterize inside a literal):

```sql
SELECT now() - make_interval(days => 7);          -- preferred, type-safe
SELECT now() - ($1 || ' days')::interval;         -- string-concat cast (works, less clean)
SELECT now() - $1 * interval '1 day';             -- multiply a bare number by a unit interval
```

### 6.4 `date_trunc` — Flooring to a Unit

`date_trunc(field, source)` floors a timestamp down to the start of the given unit. It's the workhorse of bucketing.

```sql
SELECT date_trunc('hour',    timestamptz '2026-07-19 15:47:33.123');
-- 2026-07-19 15:00:00
SELECT date_trunc('day',     timestamptz '2026-07-19 15:47:33');
-- 2026-07-19 00:00:00
SELECT date_trunc('week',    timestamptz '2026-07-19 15:47:33');  -- weeks start MONDAY (ISO)
-- 2026-07-13 00:00:00
SELECT date_trunc('month',   timestamptz '2026-07-19 15:47:33');
-- 2026-07-01 00:00:00
SELECT date_trunc('quarter', timestamptz '2026-07-19 15:47:33');
-- 2026-07-01 00:00:00
SELECT date_trunc('year',    timestamptz '2026-07-19 15:47:33');
-- 2026-01-01 00:00:00
```

Valid fields: `microseconds, milliseconds, second, minute, hour, day, week, month, quarter, year, decade, century, millennium`.

**Critical zone subtlety.** For a `timestamptz`, `date_trunc('day', ts)` truncates in the **session time zone**, then returns a `timestamptz`. "Start of day" means midnight *in your session zone*. To truncate in a *specific* zone regardless of session, use the three-argument form (PostgreSQL 12+):

```sql
-- Two-arg: floor to midnight in the SESSION zone
SELECT date_trunc('day', timestamptz '2026-07-19 03:30:00+00');

-- Three-arg: floor to midnight in a NAMED zone (session-independent, correct for daily reports)
SELECT date_trunc('day', timestamptz '2026-07-19 03:30:00+00', 'America/New_York');
-- 2026-07-18 04:00:00+00
-- because 03:30 UTC is 23:30 on Jul 18 in New York → floor → 00:00 Jul 19 NY = 04:00 UTC Jul 19...
-- (the returned value is a timestamptz representing local-midnight of that zone)
```

This three-arg form is the correct, DST-safe way to produce "calendar days as experienced in region X."

### 6.5 `AT TIME ZONE` — The Overloaded Converter (Direction Matters)

`AT TIME ZONE` does **two opposite things** depending on the input type. This is the most misunderstood operator in PostgreSQL date handling.

**Case A — input is `timestamptz` (has a zone):** `AT TIME ZONE zone` **strips** the zone and gives you the *naive wall-clock* as seen in that zone. Result type: `timestamp` (naive).

```sql
SET TimeZone = 'UTC';
SELECT timestamptz '2026-07-19 19:00:00+00' AT TIME ZONE 'Asia/Tokyo';
-- 2026-07-20 04:00:00      ← naive: "the wall clock reads 04:00 in Tokyo"
--                             (type is timestamp, no offset shown)
```

**Case B — input is `timestamp` (naive, no zone):** `AT TIME ZONE zone` **attaches** a zone — it interprets the naive digits *as being in* that zone and gives you the absolute instant. Result type: `timestamptz`.

```sql
SELECT timestamp '2026-07-20 04:00:00' AT TIME ZONE 'Asia/Tokyo';
-- 2026-07-19 19:00:00+00   ← "04:00 in Tokyo" is this absolute instant (shown in session UTC)
```

They are inverses. A → B → A round-trips. Mnemonic: **tz IN → naive OUT; naive IN → tz OUT.** The zone you name in Case A is "what clock do you want to read it on"; the zone in Case B is "what clock was it written on."

A frequent real task: "give me the local wall-clock time in the user's zone as a string."

```sql
SELECT to_char(created_at AT TIME ZONE u.timezone, 'YYYY-MM-DD HH24:MI')
FROM orders o JOIN users u ON u.id = o.user_id;
-- created_at (timestamptz) → AT TIME ZONE user's zone → naive local → format
```

### 6.6 `extract` / `date_part` — Pulling Fields Out

`extract(field FROM source)` and its function-call twin `date_part('field', source)` return a `double precision` (numeric) value for one component.

```sql
SELECT extract(year   FROM timestamptz '2026-07-19 15:47:33'); -- 2026
SELECT extract(month  FROM timestamptz '2026-07-19 15:47:33'); -- 7
SELECT extract(day    FROM timestamptz '2026-07-19 15:47:33'); -- 19
SELECT extract(hour   FROM timestamptz '2026-07-19 15:47:33'); -- 15
SELECT extract(dow    FROM timestamptz '2026-07-19');          -- 0=Sunday .. 6=Saturday → 0
SELECT extract(isodow FROM timestamptz '2026-07-19');          -- 1=Monday .. 7=Sunday   → 7
SELECT extract(doy    FROM timestamptz '2026-07-19');          -- day of year → 200
SELECT extract(week   FROM timestamptz '2026-07-19');          -- ISO week number → 29
SELECT extract(quarter FROM timestamptz '2026-07-19');         -- 3
SELECT extract(epoch  FROM timestamptz '2026-07-19 00:00+00'); -- 1784419200 (Unix seconds)
SELECT extract(epoch  FROM interval '1 day 01:00');            -- 90000 (span in seconds)
```

`dow` vs `isodow` is a classic bug: `dow` makes Sunday 0, `isodow` makes Monday 1 and Sunday 7. For "weekends," `extract(isodow ...) IN (6,7)` is clearest. Like `date_trunc`, `extract` on a `timestamptz` uses the **session zone** — extract the hour of a UTC-stored instant and you get the *session-local* hour, not the UTC hour. Convert first (`extract(hour FROM ts AT TIME ZONE 'UTC')`) if you need a specific zone.

`extract(epoch FROM ts)` is the bridge to Unix time; the inverse is `to_timestamp(epoch)` which returns a `timestamptz`.

### 6.7 `age()` — Calendar-Aware Duration

`age(a, b)` returns a **symbolic** interval expressed in years/months/days (calendar-aware), unlike plain subtraction which gives a raw day+time span.

```sql
SELECT age(timestamptz '2026-07-19', timestamptz '2000-01-15');
-- 26 years 6 mons 4 days

SELECT age(timestamptz '2000-01-15');   -- one-arg: age(current_date, arg) → "how old"
-- e.g. 26 years 6 mons 4 days (relative to today)

SELECT timestamptz '2026-07-19' - timestamptz '2000-01-15';
-- 9682 days   ← raw span, not symbolic
```

Use `age()` for human-facing "X years Y months" durations (customer tenure, account age). Use raw subtraction + `extract(epoch...)` when you need a *precise numeric* elapsed time (SLA seconds, latency). Don't use `age()` for exact arithmetic — "2 months" isn't a fixed number of seconds.

### 6.8 Generating Time Buckets with `generate_series`

Building on Topic 62, `generate_series` over timestamps produces a dense set of bucket boundaries — essential for **gap-filling** so zero-activity periods still appear.

```sql
-- Every day in July 2026 (inclusive range, 1-day step)
SELECT generate_series(
  timestamptz '2026-07-01',
  timestamptz '2026-07-31',
  interval '1 day'
) AS day;

-- Hourly buckets for a single day
SELECT generate_series(
  date_trunc('day', now()),
  date_trunc('day', now()) + interval '1 day' - interval '1 second',
  interval '1 hour'
) AS hour_bucket;
```

The gap-filling pattern (LEFT JOIN the real data onto the generated spine):

```sql
SELECT
  d.day::date                         AS day,
  COALESCE(SUM(o.total_amount), 0)    AS revenue,
  COUNT(o.id)                         AS orders
FROM generate_series(
       date_trunc('day', now()) - interval '29 days',
       date_trunc('day', now()),
       interval '1 day'
     ) AS d(day)
LEFT JOIN orders o
  ON o.created_at >= d.day
  AND o.created_at <  d.day + interval '1 day'   -- sargable half-open range
  AND o.status = 'completed'
GROUP BY d.day
ORDER BY d.day;
```

Note the join uses a **bare-column half-open range** (`>= d.day AND < d.day + 1 day`), keeping `o.created_at` sargable so an index on `orders(created_at)` drives the join. Every day appears, zero days show `0` instead of vanishing.

### 6.9 Sargable Date-Range Filters (Avoid Functions on the Column)

**Sargable** = "Search ARGument ABLE" = the predicate can use an index range scan. The rule: **never wrap the indexed column in a function or cast.** Transform the *constant* side instead.

```sql
-- ❌ NON-SARGABLE: function on the column → Seq Scan
WHERE date_trunc('day', created_at) = DATE '2026-07-19'
WHERE created_at::date = CURRENT_DATE
WHERE extract(year FROM created_at) = 2026
WHERE to_char(created_at, 'YYYY-MM') = '2026-07'

-- ✅ SARGABLE: bare column vs computed constants (half-open range)
WHERE created_at >= DATE '2026-07-19'
  AND created_at <  DATE '2026-07-19' + interval '1 day'

WHERE created_at >= date_trunc('day', now())
  AND created_at <  date_trunc('day', now()) + interval '1 day'

WHERE created_at >= DATE '2026-01-01'
  AND created_at <  DATE '2027-01-01'    -- "year 2026" as a range
```

Why the non-sargable forms fail: the B-tree on `created_at` is ordered by the *raw stored value*. `date_trunc('day', created_at)` produces a different value for every row that the index isn't sorted by, so Postgres must compute it for **every** row → full scan. The range form compares the raw column to two constants the planner computes once → a tight index range scan touching only the matching leaf pages.

**Half-open `[start, end)` is mandatory**, not `BETWEEN`. `BETWEEN '2026-07-19 00:00' AND '2026-07-19 23:59:59'` is inclusive on both ends and *misses* rows between 23:59:59.000001 and 23:59:59.999999, and also mishandles the boundary microsecond. Always `>= start AND < next_start`. This composes cleanly across days/months/years and has no fencepost bug.

If you *must* filter on a derived value repeatedly (e.g., "all rows on any Sunday"), build an **expression index**:

```sql
CREATE INDEX idx_orders_created_date
  ON orders ( (created_at AT TIME ZONE 'UTC')::date );
-- now  WHERE (created_at AT TIME ZONE 'UTC')::date = DATE '2026-07-19'  is sargable
```

But prefer the plain range on the bare column when possible — it needs no extra index and stays fast for arbitrary ranges.

### 6.10 Storing and Comparing: Common Pitfalls Recap

- **Comparing `timestamp` to `timestamptz`** forces an implicit cast using the session zone — a silent source of off-by-hours bugs. Keep types consistent.
- **`CURRENT_DATE` is session-zone-dependent.** "Today" for a UTC session differs from "today" for a Tokyo session by up to a day near midnight.
- **`now()` is frozen per transaction.** For a truly moving clock inside one long transaction, use `clock_timestamp()`.
- **DST "spring forward" gaps**: `2026-03-08 02:30` may not exist in `America/New_York`; constructing/interpreting such a naive time can shift it. Storing `timestamptz` from a true instant avoids this.

---

## 7. EXPLAIN — Sargable vs Non-Sargable Date Filters in the Plan

Assume `orders` has 50M rows and a B-tree index `orders_created_at_idx ON orders(created_at)`. We want "orders on 2026-07-19."

### Non-sargable: `date_trunc` on the column → Seq Scan

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, total_amount
FROM orders
WHERE date_trunc('day', created_at) = DATE '2026-07-19';
```

```
Seq Scan on orders  (cost=0.00..1150000.00 rows=250000 width=12)
                    (actual time=0.03..18420.55 rows=8123 loops=1)
  Filter: (date_trunc('day'::text, created_at) = '2026-07-19 00:00:00'::timestamp)
  Rows Removed by Filter: 49991877
  Buffers: shared hit=312 read=849688
Planning Time: 0.14 ms
Execution Time: 18441.02 ms
```

**Reading it**:
- `Seq Scan` — the whole 50M-row heap is read because the index is unusable.
- `Filter: date_trunc('day', created_at) = ...` — the function is evaluated per row.
- `Rows Removed by Filter: 49991877` — ~50M rows read and discarded to find 8,123.
- `Buffers: read=849688` — ~6.5 GB pulled from disk. `Execution Time: 18.4 s`.

### Sargable: bare-column half-open range → Index Range Scan

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, total_amount
FROM orders
WHERE created_at >= DATE '2026-07-19'
  AND created_at <  DATE '2026-07-19' + interval '1 day';
```

```
Index Scan using orders_created_at_idx on orders
    (cost=0.56..402.11 rows=8100 width=12)
    (actual time=0.021..3.842 rows=8123 loops=1)
  Index Cond: ((created_at >= '2026-07-19 00:00:00-04'::timestamptz)
           AND (created_at <  '2026-07-20 00:00:00-04'::timestamptz))
  Buffers: shared hit=128 read=71
Planning Time: 0.16 ms
Execution Time: 4.10 ms
```

**Reading it**:
- `Index Scan using orders_created_at_idx` — the B-tree drives the read.
- `Index Cond` (not `Filter`!) — both bounds are pushed into the index descent; only matching leaf entries are visited. This is the tell-tale of a sargable predicate.
- `read=71` buffers vs 849,688 before. `Execution Time: 4.1 ms` vs 18,441 ms — a **~4,500× speedup** from the identical logical result, purely by keeping the column bare.

The lesson lives in one line of the plan: **`Index Cond` = fast (sargable); `Filter` on the indexed column = slow (non-sargable).**

### Bucketed aggregation plan

```sql
EXPLAIN (ANALYZE)
SELECT date_trunc('day', created_at) AS day, count(*)
FROM orders
WHERE created_at >= DATE '2026-07-01'
  AND created_at <  DATE '2026-08-01'
GROUP BY 1 ORDER BY 1;
```

```
GroupAggregate  (cost=0.56..45210.00 rows=31 width=16) (actual time=2.1..612.7 rows=31 loops=1)
  Group Key: (date_trunc('day'::text, created_at))
  ->  Index Scan using orders_created_at_idx on orders
        Index Cond: ((created_at >= ...) AND (created_at < ...))
        (actual rows=248901 loops=1)
Planning Time: 0.2 ms
Execution Time: 631.4 ms
```

The `WHERE` uses the index (`Index Cond`) to fetch only July's rows; `date_trunc` runs only on those, inside the aggregate — the correct division of labor: **bare-column range in WHERE, bucket in GROUP BY.**

---

## 8. Query Examples

### Example 1 — Basic: "Today's" orders, sargable

```sql
-- Orders created today in the SESSION time zone.
-- Bare column vs computed constants → index-friendly.
SELECT
  o.id,
  o.total_amount,
  o.created_at
FROM orders o
WHERE o.created_at >= date_trunc('day', now())          -- start of today (session zone)
  AND o.created_at <  date_trunc('day', now())
                    + interval '1 day'                    -- start of tomorrow (exclusive)
ORDER BY o.created_at DESC;
```

### Example 2 — Intermediate: Per-user local-day revenue

```sql
-- Group revenue by CALENDAR DAY AS EXPERIENCED IN EACH USER'S ZONE.
-- Convert the instant to the user's local wall clock, then truncate to day.
SELECT
  u.id                                                      AS user_id,
  date_trunc('day', o.created_at, u.timezone)::date         AS local_day,
  --                              └── 3-arg date_trunc: floor in the named zone (DST-safe)
  SUM(o.total_amount)                                       AS revenue,
  COUNT(*)                                                  AS order_count
FROM orders o
JOIN users u ON u.id = o.user_id
WHERE o.created_at >= now() - interval '30 days'            -- sargable, bare column
  AND o.status = 'completed'
GROUP BY u.id, date_trunc('day', o.created_at, u.timezone)
ORDER BY u.id, local_day;
```

### Example 3 — Production Grade: Gap-filled hourly dashboard with zone conversion

```sql
-- CONTEXT:
--   orders: ~80M rows, B-tree index orders_created_at_idx ON orders(created_at) (timestamptz).
--   Goal: last 24 hours of revenue bucketed by HOUR in 'America/New_York',
--         with ZERO-activity hours shown as 0 (gap-filled), not skipped.
--   Perf expectation: index range scan over ~24h of data (a few hundred k rows),
--         aggregated into 24 buckets — well under 100 ms.
WITH spine AS (
  -- 24 hourly bucket starts, in NY local time, as absolute instants
  SELECT
    gs                                             AS bucket_start_utc,
    gs + interval '1 hour'                         AS bucket_end_utc
  FROM generate_series(
         date_trunc('hour', now()) - interval '23 hours',
         date_trunc('hour', now()),
         interval '1 hour'
       ) AS gs
)
SELECT
  (s.bucket_start_utc AT TIME ZONE 'America/New_York') AS bucket_local, -- naive NY wall clock
  COALESCE(SUM(o.total_amount), 0)                     AS revenue,
  COUNT(o.id)                                          AS orders
FROM spine s
LEFT JOIN orders o
  ON o.created_at >= s.bucket_start_utc          -- sargable half-open range
  AND o.created_at <  s.bucket_end_utc
  AND o.status = 'completed'
GROUP BY s.bucket_start_utc
ORDER BY s.bucket_start_utc;
```

```
-- EXPLAIN (ANALYZE) shape:
GroupAggregate  (actual time=0.4..58.9 rows=24 loops=1)
  Group Key: s.bucket_start_utc
  ->  Nested Loop Left Join  (actual rows=214880 loops=1)
        ->  Function Scan on generate_series gs  (actual rows=24 loops=1)
        ->  Index Scan using orders_created_at_idx on orders o
              Index Cond: ((created_at >= gs) AND (created_at < (gs + '01:00:00')))
              Filter: (status = 'completed')
              (actual rows=8953 loops=24)
Execution Time: 61.2 ms
```

The 24-row `generate_series` spine drives a Nested Loop; each iteration does a tight `Index Cond` range scan of one hour. Gap-filling is free because the `LEFT JOIN` keeps empty hours, and `COALESCE` turns their `NULL` sums into `0`.

---

## 9. Wrong → Right Patterns

### Wrong 1: `timestamp` (naive) for an event column

```sql
-- WRONG: naive timestamp for "when the payment happened"
CREATE TABLE payments (
  id          bigserial PRIMARY KEY,
  amount      numeric(12,2),
  paid_at     timestamp        -- ❌ no zone
);

-- App server (in Europe/London, BST = UTC+1) inserts "now" as local wall clock:
INSERT INTO payments (amount, paid_at) VALUES (99.00, '2026-07-19 15:00:00');

-- Reporting job connects with TimeZone = 'UTC' and reads:
SELECT paid_at FROM payments;   -- 2026-07-19 15:00:00
-- Interpreted as 15:00 UTC — but it was actually 14:00 UTC (15:00 BST).
-- Every payment is off by the app-server's offset, and NOTHING records what that was.
```

**Why it's wrong at the execution level**: `timestamp` performs no conversion on input or output. The digits `15:00:00` are stored verbatim and returned verbatim. The information "this clock was BST" existed only in the app server's head and was discarded at the type boundary. Once stored, the true instant is unrecoverable — you cannot fix this with a query, only with out-of-band knowledge of which server wrote which row.

```sql
-- RIGHT: timestamptz; store the instant, let Postgres normalize to UTC
CREATE TABLE payments (
  id       bigserial PRIMARY KEY,
  amount   numeric(12,2),
  paid_at  timestamptz NOT NULL DEFAULT now()   -- ✅ absolute instant
);
INSERT INTO payments (amount) VALUES (99.00);   -- paid_at = now(), unambiguous UTC instant
-- Any reader in any zone sees the correct local rendering of the SAME moment.
```

### Wrong 2: Non-sargable date filter

```sql
-- WRONG: cast the column to date → Seq Scan on 50M rows (~18 s)
SELECT count(*) FROM orders
WHERE created_at::date = CURRENT_DATE;
-- Filter: (created_at)::date = CURRENT_DATE   → function on the column, index unusable
```

**Why it's wrong**: `created_at::date` yields a value the `created_at` B-tree isn't ordered by, so the planner cannot descend the tree — it computes the cast for all 50M rows. Result is correct but 4,000× too slow.

```sql
-- RIGHT: bare-column half-open range → Index Cond (~4 ms)
SELECT count(*) FROM orders
WHERE created_at >= date_trunc('day', now())
  AND created_at <  date_trunc('day', now()) + interval '1 day';
```

### Wrong 3: `BETWEEN` for a day (fencepost / boundary bug)

```sql
-- WRONG: inclusive-inclusive BETWEEN misses part of the last second and mishandles boundaries
SELECT count(*) FROM orders
WHERE created_at BETWEEN '2026-07-19 00:00:00' AND '2026-07-19 23:59:59';
-- Misses rows in [23:59:59.000001 .. 23:59:59.999999]; if you "fix" it by using the next
-- day's 00:00:00 as the upper bound, BETWEEN's inclusivity then double-counts midnight.
```

**Why it's wrong**: `BETWEEN a AND b` is `>= a AND <= b`. With sub-second precision there is always a sliver of time after `23:59:59` and before the next day. And bumping `b` to `00:00:00` next day makes midnight-exactly rows count in *both* adjacent days.

```sql
-- RIGHT: half-open [start, next_start)
SELECT count(*) FROM orders
WHERE created_at >= '2026-07-19'
  AND created_at <  '2026-07-20';   -- exclusive upper bound, no gap, no overlap
```

### Wrong 4: `AT TIME ZONE` in the wrong direction

```sql
-- Goal: show created_at (timestamptz) as Tokyo local wall clock.
-- WRONG: applying AT TIME ZONE twice / assuming it "adds" the zone to a tz value
SELECT (created_at AT TIME ZONE 'Asia/Tokyo') AT TIME ZONE 'Asia/Tokyo' FROM orders;
-- First AT TIME ZONE strips → naive Tokyo wall clock (correct so far).
-- Second AT TIME ZONE (now on a NAIVE value) re-attaches Tokyo → back to a timestamptz,
-- undoing the conversion. Net result: the original instant, not the local wall clock.
```

**Why it's wrong**: the operator's meaning flips with input type (Section 6.5). One application on a `timestamptz` already gives the local naive wall clock; applying it again treats that naive value as Case B and reverses the transform.

```sql
-- RIGHT: single application converts tz → naive local wall clock
SELECT created_at AT TIME ZONE 'Asia/Tokyo' AS tokyo_wall_clock FROM orders;
```

### Wrong 5: Grouping by day in the wrong zone

```sql
-- WRONG: bucket a UTC-stored instant by "day" while session is UTC, but the business
-- reports in New York. Midnight-UTC boundaries split NY days in the wrong place.
SET TimeZone = 'UTC';
SELECT date_trunc('day', created_at) AS day, sum(total_amount)
FROM orders GROUP BY 1;
-- An order at 2026-07-19 02:00 UTC (= 2026-07-18 22:00 New York) lands in the Jul 19 bucket,
-- but for a NY business it belongs to Jul 18. Daily totals are shifted for late-evening orders.
```

**Why it's wrong**: `date_trunc('day', ts)` on a `timestamptz` truncates in the *session* zone. With session = UTC, "day" boundaries are UTC midnights, which don't line up with the business's local midnights.

```sql
-- RIGHT: truncate in the business zone explicitly (3-arg form, DST-safe)
SELECT date_trunc('day', created_at, 'America/New_York')::date AS day,
       sum(total_amount)
FROM orders
GROUP BY 1
ORDER BY 1;
```

---

## 10. Performance Profile

### Storage and CPU

- `timestamp`, `timestamptz` = **8 bytes** each; `date` = 4 bytes; `interval` = 16 bytes; `time` = 8 bytes. `timestamptz` is **not** more expensive to store than `timestamp` — the choice is about correctness, not size.
- Comparisons are integer comparisons on the underlying 8-byte value — extremely cheap. Time-zone rendering happens only at I/O (output formatting), so filtering/sorting/joining on timestamps is as fast as on `bigint`.
- `date_trunc`, `extract`, `age`, `AT TIME ZONE` are per-row function calls. Cheap individually, but on 100M rows they add up — and, more importantly, wrapping the *column* in them destroys sargability (the real cost).

### Scaling the sargable range filter

| Table size | Non-sargable (`::date =`) | Sargable (`>= X AND < Y`) with index |
|-----------|---------------------------|--------------------------------------|
| 1M rows   | Seq Scan ~120 ms          | Index range ~1–3 ms                  |
| 10M rows  | Seq Scan ~1.5 s           | Index range ~2–5 ms                  |
| 100M rows | Seq Scan ~18–30 s         | Index range ~3–10 ms (range-size bound) |

The sargable form's cost scales with the **number of rows returned**, not table size — that's the whole point of a B-tree range scan. The non-sargable form scales with **table size** regardless of how few rows match.

### Index strategy

- A plain B-tree on the timestamp column (`CREATE INDEX ON orders(created_at)`) serves range filters, `ORDER BY created_at`, and `min/max(created_at)` (via index scan) — the highest-leverage single index on an events table.
- **Composite** `(status, created_at)` or `(user_id, created_at)` serves "recent rows for a filter" — put the equality column first, the range column last, so the index descends to the equality key then range-scans the time. This is the canonical "latest N for entity X" index.
- **BRIN** (`CREATE INDEX ... USING brin(created_at)`) is ideal for **append-only, time-ordered** tables (logs, events): tiny index (kilobytes for 100M rows) because it stores per-block min/max ranges. Great when rows are physically clustered by time; poor if inserts are out-of-order. `audit_logs` and `sessions` are classic BRIN candidates.
- **Expression index** `((created_at AT TIME ZONE 'UTC')::date)` — only if you truly must filter on the derived value; costs an extra index and write overhead. Prefer bare-column ranges.
- **Partitioning** by time range (`PARTITION BY RANGE (created_at)`) lets the planner prune whole partitions for a date filter — combine with per-partition indexes for 100M+ row event tables. Partition pruning + BRIN is a common high-scale pattern.

### `now()` planner note

`now()`/`CURRENT_DATE` are `STABLE`: evaluated once per execution, so `WHERE created_at > now() - interval '7 days'` is fully sargable (the right side becomes a constant for that run). But because they're not `IMMUTABLE`, a prepared/generic plan can't fold them at plan time — negligible in practice.

---

## 11. Node.js Integration

### 11.1 Basic sargable range query with pg

```javascript
import pg from 'pg';
const { Pool } = pg;
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

// Recommendation: run the pool in UTC for deterministic behavior.
// PGTZ=UTC in the environment, or SET on connect:
pool.on('connect', (client) => client.query("SET TimeZone = 'UTC'"));

async function ordersInRange(startISO, endISO) {
  // Pass ISO-8601 strings WITH offset (…Z) so timestamptz parses them as absolute instants.
  const { rows } = await pool.query(
    `SELECT id, total_amount, created_at
       FROM orders
      WHERE created_at >= $1
        AND created_at <  $2           -- half-open, sargable, bare column
      ORDER BY created_at DESC`,
    [startISO, endISO]                 // e.g. '2026-07-19T00:00:00Z', '2026-07-20T00:00:00Z'
  );
  return rows;
}
```

### 11.2 The node-postgres `Date` parsing gotcha

```javascript
// pg parses a timestamptz column into a JS Date (an absolute instant) — SAFE.
// pg parses a NAIVE timestamp column into a JS Date AS IF it were the Node process's
// local time — DANGEROUS: the same DB value yields different instants on differently-zoned
// servers. This is the runtime mirror of the storage-level timestamp-vs-timestamptz disaster.

const { rows } = await pool.query('SELECT paid_at FROM payments LIMIT 1');
console.log(rows[0].paid_at instanceof Date); // true
// If paid_at is timestamptz → rows[0].paid_at is the correct instant regardless of TZ env.
// If paid_at is naive timestamp → the JS Date is built using the Node box's local zone. Wrong.

// Defensive: force the Node process to UTC too, so even naive parses are consistent.
//   process.env.TZ = 'UTC'   (set before the process starts)
```

### 11.3 Parameterizing intervals safely

```javascript
// You CANNOT interpolate a number inside an interval literal via $1 directly:
//   `now() - interval '$1 days'`   ❌ $1 is inside a string literal — not a bind param
// Use make_interval, or multiply a unit interval by a bound number:
async function recentOrders(days) {
  const { rows } = await pool.query(
    `SELECT id, created_at
       FROM orders
      WHERE created_at >= now() - make_interval(days => $1)   -- ✅ type-safe
      ORDER BY created_at DESC`,
    [days]
  );
  return rows;
}

// Alternative: cast a concatenated string (works, less clean, ensure $1 is an integer)
//   WHERE created_at >= now() - ($1 || ' days')::interval
```

### 11.4 Gap-filled daily report from Node

```javascript
async function dailyRevenue(userTimezone, days = 30) {
  const { rows } = await pool.query(
    `SELECT
        d.day::date                         AS day,
        COALESCE(SUM(o.total_amount), 0)    AS revenue,
        COUNT(o.id)                         AS orders
     FROM generate_series(
            date_trunc('day', now()) - make_interval(days => $2 - 1),
            date_trunc('day', now()),
            interval '1 day'
          ) AS d(day)
     LEFT JOIN orders o
       ON o.created_at >= d.day
      AND o.created_at <  d.day + interval '1 day'
      AND o.status = 'completed'
     GROUP BY d.day
     ORDER BY d.day`,
    [userTimezone, days]   // $1 reserved for a zone-aware variant; $2 = day count
  );
  return rows;
}
```

**ORM note**: Most ORMs bind JS `Date` objects to `timestamptz` correctly (as absolute instants). Where they fall down: 3-arg `date_trunc`, `AT TIME ZONE`, `generate_series` gap-filling, `age()`, and expression indexes — all need raw SQL. See Section 12.

---

## 12. ORM Comparison

### Prisma

**Can Prisma do it?** — Basic range filters yes; bucketing/zone conversion no (raw SQL).

```typescript
import { PrismaClient } from '@prisma/client';
const prisma = new PrismaClient();

// Prisma maps DateTime to timestamp(3) by DEFAULT — a known footgun. Force timestamptz:
// schema.prisma:
//   paidAt DateTime @db.Timestamptz(6)   // ✅ use @db.Timestamptz, NOT the default
// Without @db.Timestamptz, Prisma creates a NAIVE `timestamp` column → the Section 9 disaster.

// Sargable range (Prisma binds JS Dates as absolute instants):
const rows = await prisma.order.findMany({
  where: {
    createdAt: {
      gte: new Date('2026-07-19T00:00:00Z'),
      lt:  new Date('2026-07-20T00:00:00Z'),   // half-open
    },
    status: 'completed',
  },
  orderBy: { createdAt: 'desc' },
});

// date_trunc bucketing / AT TIME ZONE / generate_series → raw SQL:
const daily = await prisma.$queryRaw`
  SELECT date_trunc('day', created_at, 'America/New_York')::date AS day,
         SUM(total_amount) AS revenue
  FROM orders
  WHERE created_at >= now() - interval '30 days'
  GROUP BY 1 ORDER BY 1`;
```

**Where it breaks**: default `DateTime` → naive `timestamp`; you must add `@db.Timestamptz`. No expression for `date_trunc`, `extract`, `AT TIME ZONE`, or gap-filling.

**Verdict**: Fine for range filters. Always annotate `@db.Timestamptz`. Use `$queryRaw` for any bucketing or zone math.

---

### Drizzle ORM

**Can Drizzle do it?** — Range filters yes, typed; bucketing via the `sql` tag.

```typescript
import { db } from './db';
import { orders } from './schema';
import { and, gte, lt, eq, sql } from 'drizzle-orm';

// Schema: use timestamp with withTimezone true → timestamptz
//   createdAt: timestamp('created_at', { withTimezone: true, mode: 'date' }).notNull()

const rows = await db.select().from(orders).where(
  and(
    gte(orders.createdAt, new Date('2026-07-19T00:00:00Z')),
    lt(orders.createdAt,  new Date('2026-07-20T00:00:00Z')),
    eq(orders.status, 'completed'),
  ),
);

// Bucketed aggregation with the sql template:
const daily = await db
  .select({
    day: sql<string>`date_trunc('day', ${orders.createdAt}, 'America/New_York')::date`,
    revenue: sql<number>`sum(${orders.totalAmount})`,
  })
  .from(orders)
  .where(gte(orders.createdAt, sql`now() - interval '30 days'`))
  .groupBy(sql`1`)
  .orderBy(sql`1`);
```

**Where it breaks**: `date_trunc`, `AT TIME ZONE`, `generate_series` all need the `sql` tag — but it composes cleanly and stays type-checked at the column references.

**Verdict**: Best of the ORMs here. `withTimezone: true` gives real `timestamptz`; the `sql` tag handles the rest without leaving the query builder.

---

### Sequelize

**Can Sequelize do it?** — Range filters yes; use `DataTypes.DATE` (which *is* timestamptz in Postgres).

```javascript
const { Op } = require('sequelize');
// In Sequelize+Postgres, DataTypes.DATE → timestamptz (good), DataTypes.DATEONLY → date.
// (Note DataTypes.DATE is naive on MySQL — Postgres is the safe case.)

const rows = await Order.findAll({
  where: {
    createdAt: {
      [Op.gte]: new Date('2026-07-19T00:00:00Z'),
      [Op.lt]:  new Date('2026-07-20T00:00:00Z'),
    },
    status: 'completed',
  },
  order: [['createdAt', 'DESC']],
});

// Bucketing → raw query:
const daily = await sequelize.query(
  `SELECT date_trunc('day', created_at, 'America/New_York')::date AS day,
          SUM(total_amount) AS revenue
   FROM orders WHERE created_at >= now() - interval '30 days'
   GROUP BY 1 ORDER BY 1`,
  { type: QueryTypes.SELECT }
);
```

**Where it breaks**: no builder support for `date_trunc`/`AT TIME ZONE`/`generate_series`; `fn('date_trunc', ...)` exists but the 3-arg zone form and gap-filling are awkward — raw SQL is cleaner.

**Verdict**: `DataTypes.DATE` = timestamptz on Postgres (safe default here). Use `sequelize.query()` for buckets and zone conversion.

---

### TypeORM

**Can TypeORM do it?** — Range filters via `Between`/`MoreThanOrEqual`; bucketing via QueryBuilder raw fragments.

```typescript
import { MoreThanOrEqual, LessThan } from 'typeorm';

// Entity: use 'timestamptz' explicitly
//   @Column({ type: 'timestamptz' }) createdAt: Date;   // ✅ not 'timestamp'

const rows = await orderRepo.find({
  where: {
    createdAt: MoreThanOrEqual(new Date('2026-07-19T00:00:00Z')),
    // combine with LessThan via an array or QueryBuilder for the half-open upper bound
    status: 'completed',
  },
  order: { createdAt: 'DESC' },
});

// Bucketed aggregation:
const daily = await orderRepo.createQueryBuilder('o')
  .select("date_trunc('day', o.created_at, 'America/New_York')::date", 'day')
  .addSelect('SUM(o.total_amount)', 'revenue')
  .where('o.created_at >= now() - interval \'30 days\'')
  .groupBy('1').orderBy('1')
  .getRawMany();
```

**Where it breaks**: `@Column({ type: 'timestamp' })` creates a naive column — must specify `'timestamptz'`. Combining `>=` and `<` in `find()` is clumsy; QueryBuilder raw strings handle it but lose type safety.

**Verdict**: Explicitly declare `'timestamptz'`. Use QueryBuilder raw fragments for buckets and zone math.

---

### Knex.js

**Can Knex do it?** — Yes; the most SQL-transparent, buckets via `knex.raw`.

```javascript
// Migration: table.timestamp('created_at', { useTz: true })  → timestamptz  ✅
//            (without { useTz: true } older Knex may emit naive timestamp)

const rows = await knex('orders')
  .where('created_at', '>=', new Date('2026-07-19T00:00:00Z'))
  .andWhere('created_at', '<', new Date('2026-07-20T00:00:00Z'))
  .andWhere('status', 'completed')
  .orderBy('created_at', 'desc');

// Bucketed + gap-filled report:
const daily = await knex
  .select(knex.raw("d.day::date AS day"))
  .select(knex.raw("COALESCE(SUM(o.total_amount), 0) AS revenue"))
  .from(knex.raw(`generate_series(
      date_trunc('day', now()) - interval '29 days',
      date_trunc('day', now()), interval '1 day') AS d(day)`))
  .leftJoin('orders as o', function () {
    this.on('o.created_at', '>=', 'd.day')
        .andOn('o.created_at', '<', knex.raw("d.day + interval '1 day'"))
        .andOn('o.status', '=', knex.raw("'completed'"));
  })
  .groupBy('d.day').orderBy('d.day');
```

**Where it breaks**: `generate_series` in FROM and range join conditions need `knex.raw`; the builder itself won't model them.

**Verdict**: Most transparent. `{ useTz: true }` for timestamptz; `knex.raw` for buckets/gap-fill.

---

### ORM Summary Table

| ORM | timestamptz column | Sargable range | Bucketing (`date_trunc`) | Zone / gap-fill | Verdict |
|-----|-------------------|----------------|--------------------------|-----------------|---------|
| Prisma | `@db.Timestamptz` (else naive!) | `gte`/`lt` | `$queryRaw` | `$queryRaw` | Annotate the type; raw for buckets |
| Drizzle | `withTimezone: true` | typed | `sql` tag | `sql` tag | Best builder support |
| Sequelize | `DataTypes.DATE` (=tz on PG) | `Op.gte`/`Op.lt` | raw | raw | Safe default on PG |
| TypeORM | `type: 'timestamptz'` (else naive!) | `MoreThanOrEqual` | QueryBuilder raw | raw | Declare tz explicitly |
| Knex | `{ useTz: true }` | `.where('>=')` | `knex.raw` | `knex.raw` | Most transparent |

The cross-cutting lesson: **every ORM has a default that can silently give you a naive `timestamp`** (Prisma's `DateTime`, TypeORM's `'timestamp'`, old Knex without `useTz`). Explicitly demand `timestamptz` in your schema, always.

---

## 13. Practice Exercises

### Exercise 1 — Basic

Given `orders(id, customer_id, total_amount, status, created_at timestamptz)`:

Write a sargable query returning all `completed` orders created **yesterday** (in the session time zone). Do not wrap `created_at` in any function. Then explain in one sentence why `WHERE created_at::date = CURRENT_DATE - 1` would return the same rows but scan the whole table.

```sql
-- Write your query here
```

---

### Exercise 2 — Intermediate (combines Topics 20, 62)

Given `orders` as above, produce a **7-day daily revenue report** (last 7 calendar days including today, session zone) with these columns: `day` (date), `orders` (count), `revenue` (sum of `total_amount`, completed only). Every day must appear even if it had zero orders. Use `generate_series` for the spine and a sargable half-open range in the join.

```sql
-- Write your query here
```

---

### Exercise 3 — Hard (production simulation; naive answer is wrong)

The `orders.created_at` column is `timestamptz` and the database session runs in `UTC`. The business operates in `America/Chicago` and wants a **monthly revenue report where months are Chicago calendar months** (a sale at 2026-08-01 02:00 UTC is still *July* in Chicago). Requirements:

- Bucket by Chicago calendar month, output the month as a `date` (first of month).
- Only `completed` orders in the trailing 6 Chicago months.
- Must remain sargable on `created_at` in the `WHERE` clause (no function on the column there).
- Handle DST correctly (Chicago changes offset in March and November).

The naive `date_trunc('month', created_at)` answer is wrong because it buckets in UTC. Write the correct version.

```sql
-- Write your query here
```

---

### Exercise 4 — Interview Simulation

You're handed this query, told it's "slow and the numbers look off near midnight":

```sql
SELECT to_char(created_at, 'YYYY-MM-DD') AS day, count(*)
FROM orders
WHERE to_char(created_at, 'YYYY-MM-DD') = '2026-07-19'
GROUP BY 1;
```

1. Name every problem (there are at least three: performance, correctness, zone).
2. Rewrite it to be sargable, zone-correct for a New York business, and boundary-safe.
3. What index would you create, and why does the rewrite let the planner use it?

```sql
-- Write your corrected query here
```

---

## 14. Interview Questions

### Q1 — `timestamp` vs `timestamptz`

**Question**: What's the difference between `timestamp` and `timestamptz` in PostgreSQL, and which should you use for a `created_at` column?

**Junior answer**: "`timestamptz` stores the time zone and `timestamp` doesn't, so `timestamptz` is bigger. I'd use `timestamptz` to keep the zone."

**Principal answer**: "Both are 8 bytes and both store a 64-bit microsecond count — neither stores a zone on disk. The difference is conversion at the I/O boundary. `timestamptz` normalizes the input to UTC using the session zone on insert and renders it back in the session zone on read, so it represents an unambiguous absolute instant. `timestamp` (naive) does zero conversion: the digits in are the digits out, with no zone attached, so the same value means different instants on differently-zoned machines. For `created_at` — anything representing *when something happened* — always `timestamptz`. Naive `timestamp` is only defensible for genuinely zoneless wall-clock concepts, and even then most teams pair it with an explicit zone column. The classic disaster is choosing naive `timestamp`, having it work in dev because everything's UTC, then corrupting data the moment the app writes local times or a reporting box runs in another zone."

**Follow-up**: "If you already have a naive `timestamp` column populated by a UTC app, how do you migrate it to `timestamptz`?" *(Answer: `ALTER COLUMN ... TYPE timestamptz USING col AT TIME ZONE 'UTC'` — the `AT TIME ZONE` on a naive value attaches UTC and yields the correct instant; but only if you're certain every existing value was written in UTC.)*

---

### Q2 — Sargability

**Question**: Why is `WHERE created_at::date = CURRENT_DATE` slow, and how do you fix it without changing the result?

**Junior answer**: "Casting is slow. Maybe add an index on `created_at`."

**Principal answer**: "Casting per row isn't the headline cost — the killer is that wrapping the indexed column in `::date` makes the predicate non-sargable. The B-tree on `created_at` is ordered by the raw stored value; `created_at::date` produces a value the index isn't sorted on, so the planner can't do a range descent and falls back to a Seq Scan, computing the cast for every row. An index on `created_at` alone won't help while the column is wrapped. The fix is a bare-column half-open range: `created_at >= date_trunc('day', now()) AND created_at < date_trunc('day', now()) + interval '1 day'`. Now the plan shows `Index Cond` instead of `Filter`, and it touches only the matching leaf pages — milliseconds instead of seconds. Same logical result, ~1000× faster on a large table. If I genuinely needed to filter on the derived date repeatedly, I'd add an expression index on `((created_at AT TIME ZONE 'UTC')::date)`, but the range is preferable because it also serves arbitrary ranges and ordering."

**Follow-up**: "How do you confirm the rewrite is actually using the index?" *(Answer: `EXPLAIN (ANALYZE, BUFFERS)` — look for `Index Cond` with both bounds pushed into the index, low `Buffers read`, and no `Rows Removed by Filter` on the column.)*

---

### Q3 — Zone-correct daily buckets

**Question**: Your daily revenue report groups by `date_trunc('day', created_at)` with the DB in UTC, but the business is in New York and totals near midnight look wrong. Explain and fix.

**Junior answer**: "The times are in UTC. I'd add or subtract 4 or 5 hours."

**Principal answer**: "`date_trunc('day', ts)` on a `timestamptz` truncates in the *session* zone — UTC here — so day boundaries fall on UTC midnights, which don't align with New York calendar days. An order at 02:00 UTC is 22:00 the previous day in New York and belongs to the prior business day, so it lands in the wrong bucket. Manually subtracting a fixed offset is wrong because New York's offset changes with DST (−5 in winter, −4 in summer) and there are ambiguous/nonexistent hours at the transitions. The correct fix is the 3-argument `date_trunc('day', created_at, 'America/New_York')`, which floors to local midnight in the named zone and is DST-aware. I keep the `WHERE` range on the bare column so it stays sargable, and only apply the zone-aware `date_trunc` in `GROUP BY`/`SELECT` on the already-filtered rows."

**Follow-up**: "The report is served to users in many zones. How do you make it correct per-user?" *(Answer: store each user's IANA zone; pass it as a parameter into the 3-arg `date_trunc(..., u.timezone)` and/or `AT TIME ZONE u.timezone`; never hard-code an offset.)*

---

### Q4 — `AT TIME ZONE` direction

**Question**: A `timestamptz` value is `2026-07-19 19:00:00+00`. What does `... AT TIME ZONE 'Asia/Tokyo'` return, and what's its type?

**Junior answer**: "It converts it to Tokyo time, still a timestamptz."

**Principal answer**: "On a `timestamptz` input, `AT TIME ZONE` *strips* the zone and returns the **naive** wall-clock time as seen in Tokyo: `2026-07-20 04:00:00`, type `timestamp` (no offset). The operator is overloaded and reverses meaning by input type: applied to a naive `timestamp`, it *attaches* the zone and returns a `timestamptz` (the absolute instant of those digits in that zone). tz-in gives naive-out; naive-in gives tz-out — they're inverses. This is the source of the double-application bug where people apply it twice and undo their own conversion."

**Follow-up**: "How would you get a formatted local-time string for display?" *(Answer: `to_char(created_at AT TIME ZONE 'Asia/Tokyo', 'YYYY-MM-DD HH24:MI')` — convert to naive local, then format.)*

---

## 15. Mental Model Checkpoint

1. Both `timestamp` and `timestamptz` are 8 bytes and store microseconds since 2000-01-01. If neither stores a zone on disk, what exactly is different between them, and at what moment does that difference take effect?

2. Your session is `America/New_York`. You insert the literal `'2026-07-19 15:00:00'` into a `timestamptz` column, then a colleague in Tokyo selects it. What do they see, and why is that the *correct* behavior rather than a bug?

3. Why does `WHERE date_trunc('day', created_at) = DATE '2026-07-19'` force a Seq Scan while `WHERE created_at >= '2026-07-19' AND created_at < '2026-07-20'` uses an index — given they return identical rows?

4. When is `+ interval '1 day'` different from `+ interval '24 hours'`, and which one would you use to schedule "same time tomorrow"?

5. You need "calendar days as experienced in Berlin" from UTC-stored timestamps. Which single function call gets you DST-safe day boundaries, and why is subtracting a fixed offset wrong?

6. A dashboard's line chart skips days with no sales, making trends look continuous when they're not. What is the structural fix, and which clause does the real data get attached with?

7. `age(a, b)` and `a - b` both describe the gap between two timestamps. When must you use one and not the other, and what does each actually return?

---

## 16. Quick Reference Card

```sql
-- ═══ TYPE CHOICE ═══
timestamptz   -- ✅ events: absolute instant, stored UTC, shown per session zone (8 bytes)
timestamp     -- ⚠️ naive digits, no zone, NO conversion — only for truly zoneless wall clock
date          -- calendar day (4 bytes)     interval -- DURATION, not an instant (16 bytes)

-- ═══ "NOW" FUNCTIONS ═══
now() / CURRENT_TIMESTAMP   -- txn start, STABLE, frozen per transaction (timestamptz)
clock_timestamp()           -- real wall clock, changes mid-statement (VOLATILE)
CURRENT_DATE                -- today in SESSION zone       CURRENT_TIME -- time + offset

-- ═══ SESSION ZONE ═══
SHOW TimeZone;  SET TimeZone = 'UTC';  RESET TimeZone;
-- affects timestamptz input parsing + output rendering; NEVER affects naive timestamp/storage

-- ═══ ARITHMETIC ═══
ts + interval '1 day'      -- calendar-aware (DST: keeps wall clock)
ts + interval '24 hours'   -- absolute (DST: shifts wall clock by the day's length)
ts2 - ts1                  -- → interval (raw days+time)
age(ts1, ts2)              -- → symbolic interval (years/months/days, calendar-aware)
make_interval(days => $1)  -- build interval from a bound param (parameterizable)

-- ═══ TRUNCATE / EXTRACT ═══
date_trunc('day', ts)                 -- floor in SESSION zone → timestamptz
date_trunc('day', ts, 'Europe/Berlin')-- floor in NAMED zone (DST-safe, 3-arg, PG12+)
extract(hour FROM ts)  date_part('hour', ts)   -- one field (uses session zone)
extract(isodow FROM ts) -- Mon=1..Sun=7    extract(dow ...) -- Sun=0..Sat=6
extract(epoch FROM ts)  -- Unix seconds    to_timestamp(epoch) -- inverse → timestamptz

-- ═══ AT TIME ZONE (overloaded — direction flips by input type) ═══
timestamptz AT TIME ZONE 'X'  -- STRIP: naive wall clock in X   → timestamp
timestamp   AT TIME ZONE 'X'  -- ATTACH: interpret digits as X  → timestamptz
-- they are inverses; applying twice undoes itself

-- ═══ SARGABLE RANGE (index-friendly) ═══
-- ✅ bare column vs constants, HALF-OPEN [start, next_start)
WHERE ts >= DATE '2026-07-19' AND ts < DATE '2026-07-19' + interval '1 day'
-- ❌ NEVER: date_trunc(ts)=..., ts::date=..., extract(...)=..., to_char(ts)=..., BETWEEN a day
-- EXPLAIN tell: "Index Cond" = sargable (fast) ; "Filter" on the column = non-sargable (slow)

-- ═══ TIME BUCKETS + GAP-FILL ═══
generate_series(start, stop, interval '1 day')          -- the spine
FROM generate_series(...) d(day)
LEFT JOIN t ON t.ts >= d.day AND t.ts < d.day + interval '1 day'   -- keeps empty buckets
SELECT COALESCE(SUM(t.x), 0)                            -- 0 instead of NULL for gaps

-- ═══ INDEX RULES OF THUMB ═══
CREATE INDEX ON orders(created_at);              -- ranges, ORDER BY, min/max
CREATE INDEX ON orders(status, created_at);      -- "recent rows for entity" (eq col first)
CREATE INDEX USING brin(created_at);             -- append-only, time-clustered logs (tiny)
PARTITION BY RANGE (created_at)                  -- 100M+ rows: partition pruning by date
```

**Interview one-liners**
- "`timestamptz` isn't bigger — it converts at I/O; naive `timestamp` converts never."
- "`timestamptz` stores an instant and *forgets* the input zone; it renders per session zone."
- "Never wrap the indexed column in a function — transform the constant, keep the column bare."
- "`Index Cond` fast, `Filter` on the column slow — read it straight off EXPLAIN."
- "3-arg `date_trunc` for zone-correct, DST-safe calendar buckets; never subtract a fixed offset."
- "Half-open `[start, next)` ranges, never inclusive `BETWEEN` for a day."
- "`AT TIME ZONE`: tz-in → naive-out, naive-in → tz-out; they're inverses."
- "`+ interval '1 day'` is calendar-aware; `+ interval '24 hours'` is absolute — DST is the difference."
- "`generate_series` + `LEFT JOIN` + `COALESCE` = gap-filled time series."

---

## Connected Topics

- **Topic 62 — Generating Data (previous)**: `generate_series` is the engine behind time-bucket spines; gap-filled dashboards build directly on it.
- **Topic 64 — JSON and JSONB in PostgreSQL (next)**: timestamps inside JSONB lose type and sargability — extracting `(payload->>'ts')::timestamptz` is non-sargable unless expression-indexed, a direct application of Section 6.9.
- **Topic 07 — NULL in Depth**: `LEFT JOIN` gap-filling produces NULLs for empty buckets; `COALESCE` converts them — three-valued logic at the boundary.
- **Topic 11 — INNER JOIN in Depth**: range/time-window joins (`event within [start, end)`) are non-equi-joins; the sargable half-open range keeps them index-driven.
- **Topic 20 — GROUP BY Fundamentals**: `date_trunc` buckets are formed in `GROUP BY` after the sargable `WHERE` filter — filter first, bucket second.
- **Internals — B-tree range scans**: the 8-byte monotonic storage of timestamps makes them the canonical B-tree range-scan workload; sargability is about preserving that access path.
- **Internals — MVCC & `now()`**: `now()` returns the transaction start time (STABLE, frozen per transaction), a snapshot-consistency guarantee, not a wall-clock read.
- **Topic 17 — JOIN Performance Deep Dive**: BRIN vs B-tree and partition pruning for time-ordered event tables at 100M+ rows.
