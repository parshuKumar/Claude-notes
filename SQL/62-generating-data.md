# Topic 62 — Generating Data
### SQL Mastery Curriculum — Phase 9: Advanced SQL Patterns

---

## 1. ELI5 — The Analogy That Makes It Click

Imagine you run a coffee shop and your manager asks: "Give me sales for every day last month." You pull the receipts, group them by day, and hand over a list:

```
Jan 3   — $420
Jan 4   — $510
Jan 7   — $380
```

The manager frowns: "What happened on Jan 1, 2, 5, and 6? Were we closed? Did we make zero? Or did you just forget them?"

Here is the trap: **your data can only tell you about days something happened.** If nobody bought coffee on Jan 5, there is no receipt for Jan 5, so `GROUP BY day` produces no row for Jan 5. The absence is invisible. A chart drawn from this data would silently skip the empty days and lie about the shape of the month.

The fix is to first lay down a **complete ruler** — a printed calendar with every single day of January already on it, Jan 1 through Jan 31, whether or not anything happened. Then you glue your receipts onto that calendar. Days with a receipt show the sale; days without one show a blank you can fill with `$0`. The ruler comes from nowhere in your sales data — you *manufactured* it.

That manufactured ruler is what **generating data** is about. SQL gives you tools — chiefly `generate_series()` — to conjure rows out of thin air: every integer from 1 to 100, every day of a year, every hour of a shift, a grid of every (store × product) pair. You then LEFT JOIN your real, gappy data onto that complete scaffold. The scaffold guarantees no period, no bucket, no combination is silently missing.

Generating data is the art of manufacturing the rows that *should* exist so you can measure the rows that *do*.

---

## 2. Connection to SQL Internals

Generating data is unusual: most SQL reads rows off disk (heap pages fetched into the buffer pool via B-tree or sequential scans). Set-returning functions like `generate_series()` produce rows **with no disk access at all** — they are computed in memory, row by row, inside the executor.

Internally:

1. **`generate_series()` is a set-returning function (SRF).** In the plan it appears as a `Function Scan` node. The executor calls the function repeatedly; each call materializes the next value. For the integer/bigint form the state is a single counter — O(1) memory regardless of series length. For the timestamp form it holds the current timestamp plus the step interval.

2. **No buffer pool, no WAL, no MVCC.** Because these rows never touch the heap, there are no `shared hit`/`read` buffers for the generation itself, no visibility checks, no tuple headers. This is why generating 10 million integers is dramatically cheaper than reading 10 million rows from a table — you skip the entire storage layer.

3. **Materialization vs streaming.** A bare `Function Scan` streams rows one at a time. But the moment you join a series to real data, or the planner needs the series more than once (e.g. as the inner side of a Nested Loop), it may wrap it in a `Materialize` node, spooling the generated rows into a `tuplestore` — memory up to `work_mem`, then spilling to a temp file on disk.

4. **`random()` and the RNG.** `random()` returns a `double precision` from a per-session pseudo-random generator (a deterministic algorithm seeded once per backend, settable with `setseed()`). It is **volatile** — evaluated once per row — which is exactly why it can't be hoisted out of a loop by the planner. `gen_random_uuid()` (from `pgcrypto`, built-in since PG13) draws from a cryptographically stronger source.

5. **Cost estimation blind spot.** The planner assumes a `Function Scan` returns **1000 rows** by default (the `ROWS` estimate for a generic SRF). If you `generate_series(1, 10000000)`, the planner still thinks 1000 rows — a massive misestimate that can cascade into bad join orders downstream. You can hint the true count with `ROWS` on a custom function, or with the `generate_series(...) WITH ORDINALITY` and materialization tricks discussed later.

Understanding this is what separates "I used generate_series" from "I know why my series-driven report suddenly picked a Nested Loop and blew up."

---

## 3. Logical Execution Order Context

`generate_series()` lives in the **FROM** phase — it is a table source, exactly like a real table or a subquery. This placement drives everything:

```
FROM generate_series(...) AS d(day)   ← series produced here, in FROM
LEFT JOIN orders o ON ...             ← real data joined onto the scaffold
WHERE ...                             ← filters applied AFTER the join
GROUP BY d.day                        ← group by the SCAFFOLD column, not the data column
HAVING ...
SELECT d.day, COALESCE(SUM(...), 0)   ← fill gaps here with COALESCE
ORDER BY d.day
LIMIT ...
```

Three consequences you must internalize:

- **The series is the driving (preserved) side.** In a gap-filling query you write `FROM series LEFT JOIN data`, so the series rows always survive. If you accidentally write `FROM data LEFT JOIN series` or use INNER JOIN, the gaps come back — the whole point is lost. This is the same ON-vs-WHERE / preserved-side reasoning from Topic 12 (LEFT JOIN).

- **GROUP BY must key off the generated column.** If you `GROUP BY o.order_date` (the real column) you only get groups for dates that exist in the data. You must `GROUP BY d.day` (the manufactured column) so empty days still form a group.

- **A WHERE on the joined data can silently re-drop your gaps.** `WHERE o.status = 'completed'` placed after a LEFT JOIN filters out the NULL-extended rows (Topic 12's classic trap), converting your LEFT JOIN back into an INNER JOIN and deleting the empty days. Predicates on the right-hand table must go in the `ON` clause, not `WHERE`.

Because the series is generated in FROM, it is fully available to every later clause — you can join to it, group by it, order by it, and window over it (Topics 40–48).

---

## 4. What Is Generating Data?

**Generating data** means producing result rows from a function or expression rather than reading them from a stored table. The workhorse is `generate_series()`, a set-returning function that emits a sequence of integers, bigints, numerics, or timestamps. The generated rows form a *scaffold* (a calendar, a number line, a grid) that you either return directly (test data) or LEFT JOIN against real data (gap filling).

### 4.1 `generate_series` — integer / bigint / numeric form

```sql
SELECT n
FROM generate_series(1, 10, 2) AS s(n);
--     │              │  │  │        │
--     │              │  │  │        └── alias: table name `s`, column name `n`
--     │              │  │  └────────── step (optional, default 1; may be negative)
--     │              │  └───────────── stop  (INCLUSIVE)
--     │              └────────────────  start (INCLUSIVE)
--     └───────────────────────────────  set-returning function; one row per value
-- Returns: 1, 3, 5, 7, 9   (stops at/ before 10, inclusive of endpoints hit exactly)
```

### 4.2 `generate_series` — timestamp form

```sql
SELECT ts
FROM generate_series(
       TIMESTAMPTZ '2026-01-01 00:00',   -- start (inclusive)
       TIMESTAMPTZ '2026-01-01 23:00',   -- stop  (inclusive)
       INTERVAL '1 hour'                 -- step as an interval; may be '15 minutes', '1 day', '1 month'
     ) AS g(ts);
--   └── one row per timestamp: 00:00, 01:00, ... 23:00  (24 rows)
```

### 4.3 `WITH ORDINALITY` — attach a row number

```sql
SELECT day, idx
FROM generate_series(DATE '2026-01-01', DATE '2026-01-31', INTERVAL '1 day')
       WITH ORDINALITY AS t(day, idx);
--          │                        │    │    │
--          │                        │    │    └── ordinality column: 1,2,3,... (bigint)
--          │                        │    └─────── value column
--          │                        └──────────── alias exposes BOTH columns
--          └───────────────────────────────────── adds a sequential position to any SRF
```

### 4.4 The random-data primitives

```sql
random()                         -- double precision in [0, 1);  VOLATILE, once per row
random() * 100                   -- scale to [0, 100)
floor(random() * 100)::int       -- integer in [0, 99]
(random() * 100)::int            -- integer in [0, 100]  (rounds — endpoints half-weighted!)
md5(random()::text)              -- 32-char hex string; cheap pseudo-random token
gen_random_uuid()                -- random v4 UUID (built-in, PG13+; pgcrypto before)
setseed(0.42)                    -- make random() reproducible within the session
```

**Precise semantics of `generate_series`:**
- Both endpoints are **inclusive** — `generate_series(1,5)` yields 1,2,3,4,5 (five rows).
- If `start > stop` with a positive step (or `start < stop` with a negative step), it returns **zero rows** — no error. This empty-set behavior is important: a backwards range silently vanishes.
- A **zero step** raises `ERROR: step size cannot equal zero`.
- With timestamps, the step is an `interval`; month/year steps respect calendar arithmetic (adding `'1 month'` to Jan 31 lands on Feb 28/29, not a fixed 30 days).
- NULL in any argument yields **zero rows** (not NULL, not error).

---

## 5. Why Generating Data Mastery Matters in Production

1. **Gaps in time-series break dashboards and forecasts.** A revenue chart built straight from `GROUP BY order_date` skips zero-sale days, compressing the x-axis and distorting trends. Moving averages, week-over-week deltas, and anomaly detection all assume evenly spaced rows. Without a date spine, a quiet Sunday looks identical to a missing Sunday — one is a fact, the other is a data-integrity problem, and your query can't tell them apart.

2. **"Zero" is data too.** Regulators, finance teams, and SLA reports need to see that a metric was *zero* for a period, not merely absent. A fraud-monitoring report that omits days with zero flagged transactions can hide an outage in the fraud pipeline itself.

3. **Realistic test data unblocks development.** Before a feature ships you rarely have production-scale data. Generating 10M synthetic orders lets you test index choices, query plans, and pagination under realistic load — catching the Seq Scan that only appears past a few hundred thousand rows.

4. **Matrices catch missing combinations.** A "sales by (region × product)" report driven only by actual sales can't show that Product X had *no* sales in Region Y — the (X,Y) cell simply doesn't exist. Cross-joining two series builds the full grid so every combination is represented, then LEFT JOIN fills the actuals.

5. **Calendar/date-spine tables are foundational infrastructure.** Mature analytics stacks keep a persisted `calendar` dimension (with weekday, is_holiday, fiscal_quarter). Knowing how to generate one — and when to persist it versus generate it inline — is table-stakes data-engineering knowledge.

6. **Avoiding client-side loops.** The naive alternative is a Node.js `for` loop firing thousands of INSERTs or filling gaps in application code. Generating in SQL is one round trip and orders of magnitude faster (Topic 18's N+1 lesson applied to writes).

---

## 6. Deep Technical Content

### 6.1 The Integer Series — Every Form

```sql
SELECT * FROM generate_series(1, 5);            -- 1,2,3,4,5
SELECT * FROM generate_series(0, 10, 2);        -- 0,2,4,6,8,10
SELECT * FROM generate_series(10, 1, -1);       -- 10,9,...,1 (descending)
SELECT * FROM generate_series(1, 10, 3);        -- 1,4,7,10
SELECT * FROM generate_series(5, 5);            -- 5           (single row)
SELECT * FROM generate_series(5, 1);            -- (0 rows — start>stop, positive step)
SELECT * FROM generate_series(1::numeric, 2, 0.5); -- 1.0,1.5,2.0 (numeric supported)
```

The type of the result follows the argument type: `int` args → `integer` column, `bigint` args → `bigint`, `numeric` args → `numeric`. For very large ranges use bigint literals to avoid `integer out of range`:

```sql
SELECT count(*) FROM generate_series(1::bigint, 10000000000);  -- ten billion, no overflow
```

### 6.2 The Timestamp Series and Calendar Arithmetic

```sql
-- Every day of 2026 (365 rows)
SELECT d::date
FROM generate_series(DATE '2026-01-01', DATE '2026-12-31', INTERVAL '1 day') AS g(d);

-- Every 15 minutes across a business day
SELECT slot
FROM generate_series(
  TIMESTAMPTZ '2026-07-18 09:00',
  TIMESTAMPTZ '2026-07-18 17:00',
  INTERVAL '15 minutes'
) AS g(slot);   -- 33 rows (09:00 .. 17:00 inclusive)

-- First day of every month in a year (calendar-aware stepping)
SELECT month_start::date
FROM generate_series(DATE '2026-01-01', DATE '2026-12-01', INTERVAL '1 month') AS g(month_start);
```

**DATE vs TIMESTAMP subtlety:** `generate_series(DATE, DATE, INTERVAL)` actually returns `timestamp` (or `timestamptz`) values at midnight, not `date`. Cast with `::date` if you need pure dates. Mixing a `date` start with a `timestamptz` stop forces the whole series to `timestamptz` and drags in your session `TimeZone` — a common source of off-by-one-day bugs when the server runs UTC but the report is for a different zone.

**DST hazard:** With `timestamptz` and `INTERVAL '1 day'`, PostgreSQL steps by calendar days respecting DST, so a "spring forward" day is 23 hours. With `INTERVAL '24 hours'` it steps by exact duration, drifting off midnight after a DST boundary. For daily buckets always use `'1 day'`, not `'24 hours'`.

### 6.3 The Date-Spine / Calendar Table Pattern

Two variants: **generate inline** (a CTE) or **persist a table**.

Inline spine (no storage, recomputed each query):

```sql
WITH calendar AS (
  SELECT d::date AS day
  FROM generate_series(DATE '2026-01-01', DATE '2026-12-31', INTERVAL '1 day') AS g(d)
)
SELECT day FROM calendar;
```

Persisted calendar dimension (built once, indexed, reused):

```sql
CREATE TABLE calendar (
  day          date PRIMARY KEY,
  day_of_week  int,          -- 0=Sunday .. 6=Saturday (EXTRACT(DOW))
  iso_dow      int,          -- 1=Monday .. 7=Sunday (EXTRACT(ISODOW))
  is_weekend   boolean,
  week_of_year int,
  month        int,
  month_name   text,
  quarter      int,
  year         int,
  is_holiday   boolean DEFAULT false
);

INSERT INTO calendar (day, day_of_week, iso_dow, is_weekend, week_of_year, month, month_name, quarter, year)
SELECT
  d::date,
  EXTRACT(DOW    FROM d)::int,
  EXTRACT(ISODOW FROM d)::int,
  EXTRACT(ISODOW FROM d) IN (6, 7),
  EXTRACT(WEEK   FROM d)::int,
  EXTRACT(MONTH  FROM d)::int,
  to_char(d, 'Month'),
  EXTRACT(QUARTER FROM d)::int,
  EXTRACT(YEAR   FROM d)::int
FROM generate_series(DATE '2000-01-01', DATE '2050-12-31', INTERVAL '1 day') AS g(d);
-- ~18,600 rows covering half a century; built once, joined forever.
```

**When to persist:** if many queries need the spine, or you enrich it with holidays/fiscal periods, persist it — the join becomes a cheap indexed lookup and the enrichment lives in one place. If it's a one-off report over a narrow range, generate inline. The persisted table also gives the planner real statistics (see the cost blind spot in §2).

### 6.4 Gap-Filling — The Core Pattern

The canonical shape:

```sql
SELECT
  c.day,
  COALESCE(SUM(o.total_amount), 0) AS revenue,   -- gap → 0, not NULL
  COUNT(o.id)                      AS order_count -- COUNT(col) is 0 on all-NULL group
FROM generate_series(DATE '2026-06-01', DATE '2026-06-30', INTERVAL '1 day') AS c(day)
LEFT JOIN orders o
       ON o.created_at >= c.day
      AND o.created_at <  c.day + INTERVAL '1 day'   -- half-open range; predicate in ON!
      AND o.status = 'completed'                      -- data filter in ON, not WHERE
GROUP BY c.day
ORDER BY c.day;
```

Why each piece matters:
- **`c(day)` drives the LEFT JOIN** → all 30 days survive.
- **Range join `>= day AND < day+1`** buckets each order into its day. Using `DATE(o.created_at) = c.day` also works but is non-sargable — it can't use an index on `created_at`. The half-open range is sargable (Topic 33).
- **Filters in `ON`, not `WHERE`.** Putting `o.status='completed'` in `WHERE` would drop the NULL-extended empty days.
- **`COALESCE(SUM(...), 0)`** turns the NULL that SUM returns over an empty group into 0.
- **`COUNT(o.id)` vs `COUNT(*)`:** on an empty (all-NULL) group `COUNT(*)` returns 1 (the LEFT JOIN synthesizes one NULL row), while `COUNT(o.id)` correctly returns 0. Always count a non-null column of the right table.

### 6.5 Filling Gaps With Carry-Forward (Last Observation)

Sometimes a gap should show the *previous* value, not zero — e.g. a daily account balance where no transaction means the balance is unchanged. Combine the spine with a window function (Topic 44):

```sql
WITH spine AS (
  SELECT d::date AS day
  FROM generate_series(DATE '2026-06-01', DATE '2026-06-30', INTERVAL '1 day') AS g(d)
),
daily AS (
  SELECT s.day, b.balance
  FROM spine s
  LEFT JOIN balances b ON b.as_of_date = s.day
)
SELECT
  day,
  -- carry the last non-null balance forward over gaps
  COALESCE(
    balance,
    (SELECT b2.balance FROM daily b2
      WHERE b2.day <= daily.day AND b2.balance IS NOT NULL
      ORDER BY b2.day DESC LIMIT 1)
  ) AS balance
FROM daily
ORDER BY day;
```

A cleaner idiom uses a grouped `MAX` over a "value-partition" built from a running count of non-null values — the classic gaps-and-islands carry-forward. Covered fully in Topic 47.

### 6.6 Cross-Join Matrices

CROSS JOIN two series to build a complete grid:

```sql
-- Every (store, day) pair for June — even stores with zero sales on some days
SELECT s.store_id, c.day, COALESCE(SUM(x.amount), 0) AS revenue
FROM (SELECT id AS store_id FROM stores WHERE active) s
CROSS JOIN generate_series(DATE '2026-06-01', DATE '2026-06-30', INTERVAL '1 day') AS c(day)
LEFT JOIN sales x
       ON x.store_id = s.store_id
      AND x.sold_at >= c.day
      AND x.sold_at <  c.day + INTERVAL '1 day'
GROUP BY s.store_id, c.day
ORDER BY s.store_id, c.day;
```

The `stores × 30 days` grid guarantees a row for every combination. This is the foundation for heatmaps, cohort grids, and any "matrix must be dense" reporting. You can cross-join two pure series too:

```sql
-- A multiplication table 1..10 × 1..10
SELECT a.x, b.y, a.x * b.y AS product
FROM generate_series(1,10) a(x)
CROSS JOIN generate_series(1,10) b(y)
ORDER BY a.x, b.y;   -- 100 rows
```

### 6.7 Generating Realistic Test Data

```sql
-- 100,000 synthetic users
INSERT INTO users (id, email, name, created_at, country)
SELECT
  gen_random_uuid(),
  'user_' || i || '@example.test',
  'User ' || i,
  NOW() - (random() * INTERVAL '365 days'),         -- random signup in the last year
  (ARRAY['US','GB','IN','DE','BR'])[floor(random()*5 + 1)]  -- random from a set
FROM generate_series(1, 100000) AS g(i);
```

Techniques in the wild:
- **Random from a set:** `(ARRAY[...])[floor(random()*N + 1)]` picks a uniformly random element. Note the `+1` — Postgres arrays are 1-indexed.
- **Random timestamps in a window:** `start + random() * (stop - start)` for a uniform spread; multiply an interval by `random()`.
- **Random money:** `round((random() * 500 + 5)::numeric, 2)` for values in [5.00, 505.00].
- **Random text tokens:** `md5(random()::text)` for a 32-char hex; `substr(md5(random()::text), 1, 8)` for a short code.
- **Weighted categories:** `CASE WHEN random() < 0.7 THEN 'active' WHEN random() < 0.9 THEN 'churned' ELSE 'banned' END` — 70/20/10 split.
- **Correlated rows (orders per user):** cross join users to a per-user random count.

```sql
-- 1..5 orders per user, each with a random total
INSERT INTO orders (id, user_id, total_amount, status, created_at)
SELECT
  gen_random_uuid(),
  u.id,
  round((random()*400 + 10)::numeric, 2),
  (ARRAY['completed','pending','cancelled'])[floor(random()*3 + 1)],
  u.created_at + (random() * INTERVAL '180 days')
FROM users u
CROSS JOIN LATERAL generate_series(1, floor(random()*5 + 1)::int) AS g(n);
-- LATERAL lets the series length depend on the current user row.
```

`gen_random_uuid()` vs `md5()`: UUIDs are collision-safe and typed (`uuid`), ideal for PKs. `md5(random()::text)` can collide (astronomically rarely) and is just text — fine for opaque tokens, not for guaranteed-unique keys at scale. For truly unique sequential-ish test keys, use the series integer `i` itself.

### 6.8 `random()` Determinism and Reproducible Fixtures

`random()` is seeded once per backend from the system clock, so every run differs. For reproducible test fixtures, seed it:

```sql
SELECT setseed(0.5);   -- must be in [-1, 1]; sets the session RNG state
SELECT random();       -- now deterministic for the rest of the session
```

Caveat: `setseed` affects the whole session and only the classic `random()`; `gen_random_uuid()` uses a separate CSPRNG and is **not** reproducible via `setseed`. For deterministic UUID-like fixtures, derive them: `md5((i)::text)::uuid`-style hashing from the series index.

### 6.9 `generate_series` Beyond Sequences

- **Explode an array to rows:** `unnest(ARRAY[...])` is the array analogue; `generate_subscripts(arr, 1)` yields the indices.
- **Split a range into N buckets:** `width_bucket` plus a series for histogram edges.
- **Densify with ORDINALITY** to number generated rows for pairing (e.g. assigning round-robin values).
- **Recursive CTE as an alternative** when the "next" value depends on the previous non-linearly (e.g. Fibonacci, or stepping by business days skipping weekends) — `generate_series` only does fixed steps.

```sql
-- Business-day spine (skip weekends) via generate_series + filter
SELECT d::date
FROM generate_series(DATE '2026-06-01', DATE '2026-06-30', INTERVAL '1 day') AS g(d)
WHERE EXTRACT(ISODOW FROM d) < 6;   -- Mon..Fri only
```

### 6.10 Common Gotchas Summary

- Backwards range → **0 rows**, silently. Guard your start/stop ordering.
- NULL argument → **0 rows**.
- `date` series returns `timestamp`, not `date` — cast if needed.
- `(random()*N)::int` rounds, so 0 and N are half-weighted; use `floor(random()*N)` for a uniform `[0,N-1]`.
- Planner assumes 1000 rows for a function scan — huge series can mislead join planning.
- Filters on the joined table must live in `ON`, or gaps disappear.
- `'24 hours'` vs `'1 day'` diverge across DST.

---

## 7. EXPLAIN — Function Scan and the Series in the Plan

### 7.1 A bare series

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT n FROM generate_series(1, 1000000) AS s(n);
```

```
Function Scan on generate_series s  (cost=0.00..10.00 rows=1000 width=4)
                                    (actual time=68.2..190.4 rows=1000000 loops=1)
Planning Time: 0.05 ms
Execution Time: 240.7 ms
```

**Reading it:**
- `Function Scan on generate_series` — the SRF node; no `Seq Scan`, no `Buffers` line (nothing read from disk).
- `rows=1000` estimated vs `rows=1000000` actual — the hard-coded 1000-row SRF estimate. A 1000× misestimate. Harmless alone, dangerous when this feeds a join.
- `width=4` — a 4-byte `integer` per row.
- No `shared hit/read` — generation is pure CPU/memory.

### 7.2 The gap-fill join

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT c.day, COALESCE(SUM(o.total_amount),0) AS rev
FROM generate_series(DATE '2026-06-01', DATE '2026-06-30', INTERVAL '1 day') AS c(day)
LEFT JOIN orders o
       ON o.created_at >= c.day
      AND o.created_at <  c.day + INTERVAL '1 day'
GROUP BY c.day;
```

```
GroupAggregate  (cost=... rows=30 ...) (actual time=1.9..22.4 rows=30 loops=1)
  Group Key: c.day
  ->  Nested Loop Left Join  (actual time=0.6..18.1 rows=41200 loops=1)
        ->  Function Scan on generate_series c
              (cost=0.00..0.30 rows=30 width=4) (actual rows=30 loops=1)
        ->  Index Scan using orders_created_at_idx on orders o
              Index Cond: (created_at >= c.day AND created_at < (c.day + '1 day'))
              (actual time=0.02..0.4 rows=1373 loops=30)
Buffers: shared hit=940
Planning Time: 0.3 ms
Execution Time: 22.9 ms
```

**Reading it:**
- The 30-row series is the **outer** side of a `Nested Loop Left Join` — for each of the 30 days it index-probes `orders` by the `created_at` range (`loops=30`).
- `Index Scan ... Index Cond: created_at >= c.day AND < c.day + '1 day'` — the half-open range is **sargable**, so each day is a cheap index range scan (~1373 rows/day). Had we written `WHERE DATE(created_at) = c.day` this would be a `Seq Scan` filtering all rows every loop.
- The planner got the series estimate right here (`rows=30`) because it's a literal-bounded date range small enough that the constant-folding + SRF estimate aligned; large open-ended series often mis-plan.

### 7.3 Materialize appearing under a repeated series

```sql
EXPLAIN
SELECT * FROM generate_series(1,100) a
CROSS JOIN generate_series(1,100) b;
```

```
Nested Loop  (cost=0.00..... rows=1000000 width=8)
  ->  Function Scan on generate_series a  (rows=1000)
  ->  Materialize  (rows=1000)
        ->  Function Scan on generate_series b  (rows=1000)
```

**Reading it:** the inner series `b` is re-scanned once per outer row of `a`. To avoid recomputing the function 100 times, the planner wraps it in `Materialize`, spooling `b`'s rows into an in-memory `tuplestore` (spills to a temp file past `work_mem`). The `rows=1000000` product estimate comes from 1000 (bad `a` estimate) × 1000 (bad `b` estimate) — coincidentally close to the true 10,000? No — it's 1,000,000, a 100× overestimate. This is exactly the misestimate cascade §2 warned about.

### 7.4 What to look for

| Symptom | Meaning | Action |
|---------|---------|--------|
| `Function Scan` with `rows=1000` but huge actual | SRF row estimate blind spot | Persist the series to a temp table + ANALYZE, or add `ROWS` to a wrapper function |
| `Seq Scan` on data side of a gap-fill | Non-sargable date predicate (`DATE(col)=day`) | Rewrite as half-open range `>= day AND < day+1` |
| `Materialize` spilling to disk | Series re-scanned, `work_mem` too small | Raise `work_mem` or reduce series size |
| Empty days missing from output | Filter in `WHERE` instead of `ON`, or INNER JOIN used | Move predicate to `ON`; ensure `series LEFT JOIN data` |

---

## 8. Query Examples

### Example 1 — Basic: A number and a date series

```sql
-- Every integer 1..12 (e.g. month numbers) and every day of one week
SELECT n AS month_number
FROM generate_series(1, 12) AS s(n);

SELECT d::date AS day
FROM generate_series(DATE '2026-07-13', DATE '2026-07-19', INTERVAL '1 day') AS g(d);
-- 7 rows: Mon..Sun of that week
```

### Example 2 — Intermediate: Daily revenue with zero-filled gaps

```sql
-- Complete daily revenue for June 2026, including zero-sale days
SELECT
  c.day,
  to_char(c.day, 'Dy')             AS weekday,
  COUNT(o.id)                      AS order_count,   -- 0 on empty days
  COALESCE(SUM(o.total_amount), 0) AS revenue        -- 0.00 on empty days
FROM generate_series(DATE '2026-06-01', DATE '2026-06-30', INTERVAL '1 day') AS c(day)
LEFT JOIN orders o
       ON o.created_at >= c.day
      AND o.created_at <  c.day + INTERVAL '1 day'
      AND o.status = 'completed'      -- data filter stays in ON to preserve gaps
GROUP BY c.day
ORDER BY c.day;
```

### Example 3 — Production Grade: Dense store × day heatmap with 7-day moving average

```sql
-- CONTEXT:
--   stores:  ~2,000 active rows
--   sales:   ~80M rows over 2 years, index on (store_id, sold_at)
--   Goal:    one row per (active store, day) for the last 30 days — never a gap —
--            with revenue and a trailing 7-day moving average per store.
--   Perf expectation: index range scans per (store, day) bucket; sub-second for a
--                     single store, a few seconds full-fleet; feed a heatmap UI.
WITH spine AS (
  SELECT d::date AS day
  FROM generate_series(CURRENT_DATE - 29, CURRENT_DATE, INTERVAL '1 day') AS g(d)
),
grid AS (                                   -- dense (store, day) matrix
  SELECT s.id AS store_id, sp.day
  FROM stores s
  CROSS JOIN spine sp
  WHERE s.active
),
daily AS (
  SELECT
    g.store_id,
    g.day,
    COALESCE(SUM(x.amount), 0) AS revenue
  FROM grid g
  LEFT JOIN sales x
         ON x.store_id = g.store_id
        AND x.sold_at >= g.day
        AND x.sold_at <  g.day + INTERVAL '1 day'
  GROUP BY g.store_id, g.day
)
SELECT
  store_id,
  day,
  revenue,
  ROUND(AVG(revenue) OVER (
    PARTITION BY store_id ORDER BY day
    ROWS BETWEEN 6 PRECEDING AND CURRENT ROW      -- trailing 7-day window (Topic 45)
  ), 2) AS revenue_7d_avg
FROM daily
ORDER BY store_id, day;
```

```
-- EXPLAIN (ANALYZE) abridged:
WindowAgg  (actual time=... rows=60000 loops=1)
  ->  Sort  Sort Key: daily.store_id, daily.day
        ->  Subquery Scan on daily
              ->  GroupAggregate  (rows=60000)
                    ->  Nested Loop Left Join  (rows=~2.4M)
                          ->  Nested Loop  (grid: 2000 stores × 30 days = 60000)
                                ->  Seq Scan on stores (Filter: active)
                                ->  Function Scan on generate_series
                          ->  Index Scan using sales_store_sold_idx on sales x
                                Index Cond: (store_id = g.store_id
                                  AND sold_at >= g.day AND sold_at < g.day + '1 day')
Execution Time: ~3.8 s
```

The `(store_id, sold_at)` composite index makes each of the 60,000 buckets a tight range probe; the moving average is computed in a single `WindowAgg` pass after the dense grid is built.

---

## 9. Wrong → Right Patterns

### Wrong 1: Grouping by the data column instead of the spine

```sql
-- WRONG: no row exists for days with zero orders
SELECT o.created_at::date AS day, COUNT(*) AS orders
FROM orders o
WHERE o.created_at >= DATE '2026-06-01' AND o.created_at < DATE '2026-07-01'
GROUP BY o.created_at::date
ORDER BY day;
-- RESULT: June 5 and 6 simply absent — the chart silently skips them.
```

```sql
-- RIGHT: spine drives the grouping; gaps become explicit zeros
SELECT c.day, COUNT(o.id) AS orders
FROM generate_series(DATE '2026-06-01', DATE '2026-06-30', INTERVAL '1 day') AS c(day)
LEFT JOIN orders o
       ON o.created_at >= c.day AND o.created_at < c.day + INTERVAL '1 day'
GROUP BY c.day
ORDER BY c.day;
```

**Why:** `GROUP BY` can only form groups from rows that exist. Days with no order produce no row to group. The spine manufactures the missing days.

### Wrong 2: Filter in WHERE collapses the LEFT JOIN

```sql
-- WRONG: WHERE on the right table drops NULL-extended gap rows
SELECT c.day, COALESCE(SUM(o.total_amount),0) AS revenue
FROM generate_series(DATE '2026-06-01', DATE '2026-06-30', INTERVAL '1 day') AS c(day)
LEFT JOIN orders o ON o.created_at >= c.day AND o.created_at < c.day + INTERVAL '1 day'
WHERE o.status = 'completed'          -- ← BUG: NULLs from empty days fail this test
GROUP BY c.day;
-- RESULT: empty days vanish — LEFT JOIN silently became INNER JOIN.
```

```sql
-- RIGHT: predicate on the right table belongs in ON
SELECT c.day, COALESCE(SUM(o.total_amount),0) AS revenue
FROM generate_series(DATE '2026-06-01', DATE '2026-06-30', INTERVAL '1 day') AS c(day)
LEFT JOIN orders o
       ON o.created_at >= c.day AND o.created_at < c.day + INTERVAL '1 day'
      AND o.status = 'completed'
GROUP BY c.day;
```

**Why:** on an empty day the LEFT JOIN emits one row with all `o.*` columns NULL. `o.status = 'completed'` evaluates to UNKNOWN → the row is filtered → the gap returns. This is the Topic 12 LEFT-JOIN-into-INNER trap.

### Wrong 3: Non-sargable date bucketing

```sql
-- WRONG: DATE() wraps the column → index on created_at unusable → Seq Scan each loop
SELECT c.day, COUNT(o.id)
FROM generate_series(DATE '2026-01-01', DATE '2026-12-31', INTERVAL '1 day') AS c(day)
LEFT JOIN orders o ON DATE(o.created_at) = c.day
GROUP BY c.day;
```

```sql
-- RIGHT: half-open range keeps the predicate sargable → index range scan per day
SELECT c.day, COUNT(o.id)
FROM generate_series(DATE '2026-01-01', DATE '2026-12-31', INTERVAL '1 day') AS c(day)
LEFT JOIN orders o
       ON o.created_at >= c.day AND o.created_at < c.day + INTERVAL '1 day'
GROUP BY c.day;
```

**Why:** applying a function to the indexed column (`DATE(o.created_at)`) prevents B-tree use unless an expression index exists. Over 365 loops that is 365 full scans.

### Wrong 4: `COUNT(*)` inflates empty buckets to 1

```sql
-- WRONG: empty days report 1 order, not 0
SELECT c.day, COUNT(*) AS orders     -- counts the synthesized NULL row
FROM generate_series(DATE '2026-06-01', DATE '2026-06-30', INTERVAL '1 day') AS c(day)
LEFT JOIN orders o
       ON o.created_at >= c.day AND o.created_at < c.day + INTERVAL '1 day'
GROUP BY c.day;
```

```sql
-- RIGHT: count a non-null column of the right table
SELECT c.day, COUNT(o.id) AS orders  -- NULL id on empty day is not counted → 0
FROM generate_series(DATE '2026-06-01', DATE '2026-06-30', INTERVAL '1 day') AS c(day)
LEFT JOIN orders o
       ON o.created_at >= c.day AND o.created_at < c.day + INTERVAL '1 day'
GROUP BY c.day;
```

**Why:** `COUNT(*)` counts rows including the LEFT JOIN's synthetic all-NULL row; `COUNT(o.id)` ignores NULLs, correctly yielding 0.

### Wrong 5: Backwards range returns nothing

```sql
-- WRONG: start after stop with default positive step → 0 rows, no error
SELECT * FROM generate_series(DATE '2026-06-30', DATE '2026-06-01', INTERVAL '1 day');
-- RESULT: (0 rows) — the whole report is empty and no one knows why.
```

```sql
-- RIGHT: order the bounds correctly (or use a negative step deliberately)
SELECT * FROM generate_series(DATE '2026-06-01', DATE '2026-06-30', INTERVAL '1 day');
```

**Why:** `generate_series` returns the empty set when the step direction can never reach `stop`. Silent, easy to miss when bounds come from variables. Validate ordering in application code before binding.

### Wrong 6: `(random()*N)::int` skews the endpoints

```sql
-- WRONG: cast rounds → 0 and N appear half as often as interior values
SELECT (random()*10)::int AS bucket FROM generate_series(1,100000);
-- bucket 0 and bucket 10 are under-represented; distribution is not uniform on 0..10.
```

```sql
-- RIGHT: floor for a uniform integer in [0, N-1], then offset if needed
SELECT floor(random()*10)::int AS bucket FROM generate_series(1,100000);  -- uniform 0..9
```

**Why:** rounding maps `[0,0.5)`→0 and `[9.5,10)`→10, half-width bins at both ends. `floor` gives equal-width bins.

---

## 10. Performance Profile

### 10.1 Generation cost

Pure generation is CPU-bound, no I/O. Rough single-core figures on commodity hardware:

| Series size | Bare generation | Notes |
|-------------|-----------------|-------|
| 1K | < 1 ms | negligible |
| 1M integers | ~200–250 ms | O(1) memory, streamed |
| 10M integers | ~2–2.5 s | still O(1) memory |
| 1M with `md5(random()::text)` | ~1.5–3 s | md5 + RNG per row dominates |
| 100K `gen_random_uuid()` | ~150–400 ms | CSPRNG cost per row |

Memory is O(1) for a streamed series. It becomes O(rows) only when a `Materialize`, `Sort`, or `GROUP BY` buffers the generated rows — then it counts against `work_mem` and spills to temp files past that.

### 10.2 Gap-fill scaling

The join, not the series, dominates. With an index supporting the bucket predicate:

| Data table size | Buckets (days) | Pattern | Expectation |
|-----------------|----------------|---------|-------------|
| 1M orders | 30 | Nested Loop, index range/bucket | tens of ms |
| 10M orders | 365 | Nested Loop, index range/bucket | ~100–500 ms |
| 100M orders | 365 | index range per bucket | ~1–5 s; ensure composite index |
| 100M, matrix 2K stores × 30 days | 60K buckets | Nested Loop over grid | seconds; needs `(store_id, ts)` index |

Without a sargable predicate (Wrong 3), each bucket becomes a Seq Scan and cost multiplies by the bucket count — catastrophic.

### 10.3 Optimization techniques specific to generating data

1. **Sargable bucket predicates.** Always `ts >= day AND ts < day+1`, never `DATE(ts)=day`. This is the single biggest lever.
2. **Composite index matching the join.** For a (store × day) matrix, index `(store_id, sold_at)` so each cell is a tight range probe.
3. **Persist wide/reused spines.** A `calendar` table has real statistics (fixing the 1000-row SRF estimate) and can be indexed; inline series cannot.
4. **Pre-aggregate the data side first.** For very large fact tables, `GROUP BY day` the data once, then LEFT JOIN the spine to the small aggregate — turns N bucket probes into one scan + a merge.

```sql
WITH spine AS (
  SELECT d::date AS day FROM generate_series(DATE '2026-01-01', DATE '2026-12-31', INTERVAL '1 day') g(d)
),
agg AS (
  SELECT created_at::date AS day, SUM(total_amount) AS rev
  FROM orders WHERE status='completed'
    AND created_at >= DATE '2026-01-01' AND created_at < DATE '2027-01-01'
  GROUP BY created_at::date
)
SELECT s.day, COALESCE(a.rev, 0) AS rev
FROM spine s LEFT JOIN agg a ON a.day = s.day
ORDER BY s.day;
```

5. **Batch large generated INSERTs.** A single `INSERT ... SELECT FROM generate_series(1, 10000000)` is one statement but writes 10M rows + WAL + index maintenance. For huge loads, drop indexes, `COPY` or insert, then rebuild indexes; or chunk into transactions to bound WAL growth.
6. **`work_mem` for materialized cross joins.** A big series CROSS JOIN spills unless `work_mem` covers the inner tuplestore.

---

## 11. Node.js Integration

### 11.1 Parameterized gap-fill report

```javascript
import pg from 'pg';
const { Pool } = pg;
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

// Daily revenue with zero-filled gaps, bounds passed as params
async function dailyRevenue(startDate, endDate) {
  const { rows } = await pool.query(
    `SELECT
       c.day,
       COUNT(o.id)                      AS order_count,
       COALESCE(SUM(o.total_amount), 0) AS revenue
     FROM generate_series($1::date, $2::date, INTERVAL '1 day') AS c(day)
     LEFT JOIN orders o
            ON o.created_at >= c.day
           AND o.created_at <  c.day + INTERVAL '1 day'
           AND o.status = 'completed'
     GROUP BY c.day
     ORDER BY c.day`,
    [startDate, endDate]           // $1, $2 bound safely — never string-concat dates
  );
  return rows;   // one row per day in [start,end], gaps present as revenue: '0'
}
```

Note: numeric columns (`SUM`, money) come back as **strings** from `pg` to preserve precision — parse with a decimal library, not `parseFloat`, for financial values.

### 11.2 Bulk test-data generation in one round trip

```javascript
// Generate N synthetic users server-side — one statement, no client loop (avoids N+1 writes)
async function seedUsers(n) {
  await pool.query(
    `INSERT INTO users (id, email, name, created_at, country)
     SELECT
       gen_random_uuid(),
       'user_' || i || '@example.test',
       'User ' || i,
       NOW() - (random() * INTERVAL '365 days'),
       (ARRAY['US','GB','IN','DE','BR'])[floor(random()*5 + 1)]
     FROM generate_series(1, $1) AS g(i)`,
    [n]
  );
}
// seedUsers(100000) inserts 100k rows in a single query — vastly faster than a JS for-loop of INSERTs.
```

### 11.3 Guarding the backwards-range trap in application code

```javascript
async function report(start, end) {
  // generate_series returns 0 rows if start > end — validate before querying
  if (new Date(start) > new Date(end)) {
    throw new Error(`Invalid range: start ${start} is after end ${end}`);
  }
  return dailyRevenue(start, end);
}
```

### 11.4 Reproducible fixtures with a seeded RNG

```javascript
// Run inside a single client/transaction so setseed applies to the same backend
async function seedDeterministic() {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');
    await client.query('SELECT setseed(0.42)');   // deterministic random() for this session
    await client.query(
      `INSERT INTO orders (id, user_id, total_amount, status)
       SELECT gen_random_uuid(), $1,
              round((random()*400 + 10)::numeric, 2),
              (ARRAY['completed','pending'])[floor(random()*2 + 1)]
       FROM generate_series(1, 50) g(i)`,
      ['00000000-0000-0000-0000-000000000001']
    );
    await client.query('COMMIT');
  } catch (e) { await client.query('ROLLBACK'); throw e; }
  finally { client.release(); }
}
```

**Do ORMs handle generate_series?** No mainstream ORM has a first-class `generate_series` builder for date spines or gap filling — this is almost always raw SQL. ORMs handle the *reading* of the result set fine, but the generation itself is escape-hatch territory.

---

## 12. ORM Comparison

### Prisma

**Can Prisma do generate_series?** — Not natively. Prisma has no query-builder concept for set-returning functions or date spines. You must use `$queryRaw`.

```typescript
import { PrismaClient, Prisma } from '@prisma/client';
const prisma = new PrismaClient();

const start = new Date('2026-06-01'), end = new Date('2026-06-30');
const rows = await prisma.$queryRaw<Array<{ day: Date; revenue: string }>>`
  SELECT c.day, COALESCE(SUM(o.total_amount), 0) AS revenue
  FROM generate_series(${start}::date, ${end}::date, INTERVAL '1 day') AS c(day)
  LEFT JOIN orders o
         ON o.created_at >= c.day AND o.created_at < c.day + INTERVAL '1 day'
        AND o.status = 'completed'
  GROUP BY c.day
  ORDER BY c.day
`;
```

**Where it breaks:** no typed model maps to a generated series, so results are untyped-by-default; you must annotate the generic. Interpolating an `INTERVAL` string needs `Prisma.raw` since template params are values, not SQL fragments.

**Verdict:** Prisma reads the result fine but the generation is 100% raw SQL. Fine, expected.

---

### Drizzle ORM

**Can Drizzle do generate_series?** — Only through the `sql` template tag; no dedicated helper.

```typescript
import { sql } from 'drizzle-orm';
import { db } from './db';

const rows = await db.execute(sql`
  SELECT c.day, COALESCE(SUM(o.total_amount), 0) AS revenue
  FROM generate_series(${'2026-06-01'}::date, ${'2026-06-30'}::date, INTERVAL '1 day') AS c(day)
  LEFT JOIN orders o
         ON o.created_at >= c.day AND o.created_at < c.day + INTERVAL '1 day'
        AND o.status = 'completed'
  GROUP BY c.day
  ORDER BY c.day
`);
```

You *can* model a persisted `calendar` table in the Drizzle schema and then use the normal typed `.leftJoin(calendar, ...)` builder — that is the cleaner path if you keep a physical date dimension.

**Where it breaks:** inline series generation stays in `sql``` templates; only the persisted-table approach gets full type inference.

**Verdict:** Use a persisted `calendar` table for typed joins; use `sql` for inline series.

---

### Sequelize

**Can Sequelize do generate_series?** — No builder support; use `sequelize.query`.

```javascript
const { QueryTypes } = require('sequelize');
const rows = await sequelize.query(
  `SELECT c.day, COALESCE(SUM(o.total_amount), 0) AS revenue
   FROM generate_series(:start::date, :end::date, INTERVAL '1 day') AS c(day)
   LEFT JOIN orders o
          ON o.created_at >= c.day AND o.created_at < c.day + INTERVAL '1 day'
         AND o.status = 'completed'
   GROUP BY c.day
   ORDER BY c.day`,
  { replacements: { start: '2026-06-01', end: '2026-06-30' }, type: QueryTypes.SELECT }
);
```

**Where it breaks:** Sequelize models can't represent a generated source, so the whole query is raw. Named `:replacements` are the safe binding path.

**Verdict:** Raw query only. Model a `calendar` table if you want an association-based join.

---

### TypeORM

**Can TypeORM do generate_series?** — Not via QueryBuilder for the series itself; use `query()` or wrap a series as a raw FROM.

```typescript
const rows = await dataSource.query(
  `SELECT c.day, COALESCE(SUM(o.total_amount), 0) AS revenue
   FROM generate_series($1::date, $2::date, INTERVAL '1 day') AS c(day)
   LEFT JOIN orders o
          ON o.created_at >= c.day AND o.created_at < c.day + INTERVAL '1 day'
         AND o.status = 'completed'
   GROUP BY c.day
   ORDER BY c.day`,
  ['2026-06-01', '2026-06-30']
);
```

If you persist a `Calendar` entity, `createQueryBuilder('c').leftJoin('orders', 'o', '...')` works with the normal builder.

**Where it breaks:** QueryBuilder has no `.fromFunction()`; inline series must be raw strings with positional params.

**Verdict:** Raw `query()` for inline series; entity + QueryBuilder if you persist the calendar.

---

### Knex.js

**Can Knex do generate_series?** — Yes, more elegantly than the heavier ORMs, via `knex.raw` in `.fromRaw`/`.joinRaw`.

```javascript
const rows = await knex
  .select('c.day')
  .select(knex.raw('COALESCE(SUM(o.total_amount), 0) AS revenue'))
  .fromRaw(
    "generate_series(?::date, ?::date, INTERVAL '1 day') AS c(day)",
    ['2026-06-01', '2026-06-30']
  )
  .leftJoin('orders as o', function () {
    this.on('o.created_at', '>=', 'c.day')
        .andOn('o.created_at', '<', knex.raw("c.day + INTERVAL '1 day'"))
        .andOn('o.status', '=', knex.raw('?', ['completed']));
  })
  .groupBy('c.day')
  .orderBy('c.day');
```

**Where it breaks:** the series still needs `fromRaw`; the range ON clause needs the function form with `knex.raw` for the interval.

**Verdict:** Knex is the most SQL-transparent — series via `fromRaw`, the rest in the builder.

---

### ORM Summary Table

| ORM | generate_series support | Inline series | Persisted calendar path | Verdict |
|-----|------------------------|---------------|-------------------------|---------|
| Prisma | None native | `$queryRaw` | model `calendar`, normal relations | Raw for generation |
| Drizzle | `sql` template only | `sql`\`\` | typed `.leftJoin(calendar)` | Persist for types |
| Sequelize | None native | `sequelize.query` | model + association | Raw only |
| TypeORM | None native | `.query()` | `Calendar` entity + QB | Raw or entity |
| Knex | Via `fromRaw`/`raw` | `fromRaw` | table + builder | Most transparent |

Universal truth: **the generation is raw SQL everywhere; a persisted `calendar` dimension is the only way to get typed, builder-native joins.**

---

## 13. Practice Exercises

### Exercise 1 — Basic

Write a single query that returns every hour of `2026-07-18` (00:00 through 23:00, 24 rows) as a timestamp column named `slot`, plus a second column `is_business_hour` that is TRUE for slots between 09:00 and 17:00 inclusive.

```sql
-- Write your query here
```

---

### Exercise 2 — Medium (combines gap filling + aggregation)

Given `orders(id, customer_id, total_amount, status, created_at)`, produce a complete daily report for the first two weeks of July 2026 (14 rows, no gaps):
- `day`
- `orders` — count of distinct completed orders that day (0 on empty days)
- `revenue` — sum of `total_amount` for completed orders (0.00 on empty days)
- `avg_order_value` — revenue / orders, or 0 when there are no orders (avoid divide-by-zero)

Make sure days with zero completed orders still appear with zeros, and ensure the date predicate is sargable.

```sql
-- Write your query here
```

---

### Exercise 3 — Hard (production simulation, naive answer is wrong)

You must build a **dense cohort retention grid**. Given `users(id, created_at)` and `sessions(id, user_id, started_at)`:
- For each signup **week** in Q2 2026 (13 weeks) and each **week offset** 0..8 (9 offsets), output the number of users from that cohort who had at least one session during that offset week.
- Every (cohort_week, offset) cell must exist even when zero users were retained — a naive `GROUP BY` over sessions will leave holes exactly where retention dropped to zero, which is the most important signal.

Hint: cross join a series of cohort weeks with a series of offsets to build the 13×9 grid, then LEFT JOIN the retention counts.

```sql
-- Write your query here
```

---

### Exercise 4 — Interview Simulation

A teammate's "monthly active users" chart looks jagged and a reviewer says "some months are missing and one month shows a suspiciously round 1." Here is the query:

```sql
SELECT date_trunc('month', s.started_at) AS month, COUNT(*) AS active
FROM generate_series(DATE '2026-01-01', DATE '2026-12-01', INTERVAL '1 month') AS m(month)
LEFT JOIN sessions s ON s.started_at::date = m.month::date
WHERE s.user_id IS NOT NULL
GROUP BY date_trunc('month', s.started_at)
ORDER BY month;
```

1. List every bug (there are at least four: grouping column, WHERE-vs-ON, `COUNT(*)`, and the join predicate).
2. Rewrite it so every month of 2026 appears exactly once, zero-filled, with a **distinct** active-user count, using a sargable month-range join.

```sql
-- Write your query here
```

---

## 14. Interview Questions

### Junior Level

**Q: What does `generate_series(1, 5)` return, and are the endpoints included?**

Junior answer: "It returns numbers from 1 to 5."

Principal answer: "Five rows — 1, 2, 3, 4, 5 — both endpoints inclusive. It's a set-returning function, so in a query it behaves like a table source in the FROM clause, one row per value. The step defaults to 1 and can be any nonzero value including negative for descending sequences. If start > stop with a positive step it returns the empty set silently — no error — which is a common source of mysteriously empty reports when the bounds come from variables."

Interviewer follow-up: "What happens with `generate_series(5, 1)`?" → Zero rows (positive default step can't descend). "And `generate_series(5,1,-1)`?" → 5,4,3,2,1.

---

**Q: Your daily-sales chart is missing days where nothing sold. Why, and how do you fix it?**

Junior answer: "Add the missing days manually" or "use COALESCE."

Principal answer: "`GROUP BY sale_date` can only produce groups for dates that appear in the data; a day with zero sales has no row, so no group, so it's invisible. The fix is a date spine: `generate_series(start, end, INTERVAL '1 day')` produces every day whether or not sales exist, then `spine LEFT JOIN sales` on a half-open date range keeps all spine days. Group by the spine column, and wrap the aggregate in `COALESCE(SUM(...), 0)` so empty days read 0 instead of NULL. Two gotchas: any filter on the sales table must go in the ON clause, not WHERE, or the LEFT JOIN collapses to an INNER JOIN and the gaps return; and count `sales.id`, not `*`, so empty buckets report 0."

Interviewer follow-up: "Why not `WHERE`?" → On an empty day the LEFT JOIN emits an all-NULL sales row; a WHERE predicate on a NULL column is UNKNOWN, filtering the row and re-deleting the gap.

---

**Q: How do you generate 100,000 fake users for load testing?**

Junior answer: "Write a script that loops and inserts."

Principal answer: "One `INSERT ... SELECT ... FROM generate_series(1, 100000)` — a single server-side statement rather than 100,000 client round trips. Use `gen_random_uuid()` for PKs, `'user_' || i || '@x.test'` for unique emails from the series index, `NOW() - random()*INTERVAL '365 days'` for spread-out signup times, and `(ARRAY[...])[floor(random()*N+1)]` to pick random categorical values. For reproducible fixtures, `setseed()` first — though note that only affects `random()`, not `gen_random_uuid()`. For millions of rows, drop non-critical indexes first and rebuild after to avoid per-row index maintenance and WAL bloat."

Interviewer follow-up: "Why `floor` not a cast to int?" → `(random()*N)::int` rounds, half-weighting the endpoints; `floor(random()*N)` gives a uniform `[0, N-1]`.

---

### Principal Level

**Q: EXPLAIN shows a Function Scan estimated at 1000 rows but your series produces 10 million, and the query then picks a terrible join order. Explain the mechanism and your options.**

Principal answer: "PostgreSQL has no statistics for set-returning functions, so it applies a hard-coded default estimate — 1000 rows for a generic SRF. A 10M-row series is therefore under-estimated 10,000×. That misestimate propagates: downstream joins size their hash tables and choose Nested Loop vs Hash Join based on the wrong cardinality, often picking a plan that's catastrophic at the real scale. Options: (1) materialize the series into a temp table and `ANALYZE` it so real stats exist; (2) wrap the generation in a custom SQL/PLpgSQL function declared with an accurate `ROWS n` hint; (3) for date spines, persist a physical `calendar` table — it has stats and indexes, which is why mature analytics stacks keep one rather than regenerating inline; (4) restructure so the huge series isn't on the estimated-sensitive side of a join, e.g. pre-aggregate the fact table first and join a small spine to the small aggregate."

Interviewer follow-up: "Does `Materialize` in the plan fix the estimate?" → No. `Materialize` just caches rows to avoid recomputation on rescans; it doesn't create statistics or correct the cardinality estimate.

---

**Q: Walk me through building a *dense* (region × month) revenue matrix where every combination appears even with zero sales, on a 100M-row fact table. What are the correctness and performance concerns?**

Principal answer: "Correctness: build the grid explicitly by `CROSS JOIN`ing the region dimension with `generate_series` over months, then `LEFT JOIN` the fact table onto the grid — never derive months or regions from the facts, or missing combinations silently disappear, and missing combinations are exactly what a matrix must show. Group by the grid columns, `COALESCE` the aggregate to 0, and keep the fact-side filter in ON. Performance: the join predicate must be sargable — bucket by `sold_at >= month_start AND sold_at < month_start + INTERVAL '1 month'`, never `date_trunc('month', sold_at) = m`, which defeats the index. Build a composite index `(region_id, sold_at)` so each of the `regions × months` cells is a tight index range probe. At 100M rows I'd also consider pre-aggregating the facts to (region, month) once and joining the small dense grid to that small aggregate — turning thousands of per-cell probes into a single grouped scan plus a merge. Watch `work_mem` if the grid CROSS JOIN materializes."

Interviewer follow-up: "Where does the zero come from for an empty cell?" → The LEFT JOIN emits the grid row with NULL facts; `COALESCE(SUM(amount), 0)` renders it as 0. `COUNT` must target a fact column, not `*`.

---

**Q: When would you generate a date spine inline versus persist a calendar table?**

Principal answer: "Inline (`generate_series` in a CTE) when it's a one-off or narrow report and I don't want schema — zero maintenance, always correct bounds. Persist a `calendar` dimension when: many queries need spines (DRY + consistent), I need enrichment the series can't express cheaply (holidays, fiscal quarters, business-day flags), or I'm hitting the planner's SRF-estimate blind spot and want real statistics and indexes on the spine. The persisted table also lets ORMs join it as a normal typed relation instead of raw SQL. The cost is keeping its range current — I'd generate decades ahead (e.g. 2000–2050) so it rarely needs extending."

Interviewer follow-up: "Any downside to a 50-year calendar table?" → Negligible — ~18K rows, a few hundred KB, fully cached; the PK index makes joins trivial.

---

## 15. Mental Model Checkpoint

1. `generate_series(DATE '2026-03-01', DATE '2026-03-01', INTERVAL '1 day')` — how many rows, and what type is the column?

2. You LEFT JOIN a date spine to `orders` and put `AND o.amount > 100` in the `WHERE` clause. What happens to days where every order was ≤ 100? What happens to days with no orders at all?

3. You need every 30-minute slot of a work shift, but the shift crosses a daylight-saving "spring forward." Why does `INTERVAL '30 minutes'` behave differently from stepping in `'0.5 hours'` vs the day-level `'1 day'` vs `'24 hours'` distinction — and which matters here?

4. A report shows `COUNT(*) = 1` for every "empty" day. What single character do you change, and why does it fix it?

5. You cross join `generate_series(1,1000)` with `generate_series(1,1000)`. The plan shows `Materialize` and the query spills to a temp file. What is being materialized, why, and which setting governs the spill?

6. `setseed(0.3)` then two `INSERT ... SELECT ... gen_random_uuid() ... FROM generate_series(...)` in the same session — are the generated UUIDs reproducible across two runs? Why or why not?

7. Your gap-fill query is fast for 30 days but 40× slower for a full year, worse than linearly. EXPLAIN shows a Seq Scan on the fact table inside the loop. What is the likely non-sargable predicate, and how do you make it sargable?

---

## 16. Quick Reference Card

```sql
-- INTEGER / DATE / TIMESTAMP SERIES (endpoints INCLUSIVE)
generate_series(1, 10)                                  -- 1..10
generate_series(0, 100, 5)                              -- 0,5,..,100
generate_series(10, 1, -1)                              -- descending
generate_series(DATE '2026-01-01', DATE '2026-12-31', INTERVAL '1 day')
generate_series(t0, t1, INTERVAL '15 minutes')          -- time slots
-- start>stop with +step, or any NULL arg → 0 ROWS (silent). step=0 → ERROR.
-- date-arg series returns TIMESTAMP; cast ::date if you need pure dates.

-- WITH ORDINALITY (attach a 1-based row number)
FROM generate_series(...) WITH ORDINALITY AS t(value, idx)

-- DATE SPINE + GAP FILL (the core pattern)
SELECT c.day, COALESCE(SUM(o.amt),0) AS v, COUNT(o.id) AS n
FROM generate_series(:start::date, :end::date, INTERVAL '1 day') c(day)
LEFT JOIN orders o
       ON o.created_at >= c.day
      AND o.created_at <  c.day + INTERVAL '1 day'   -- SARGABLE range
      AND o.status = 'completed'                     -- data filter in ON, not WHERE!
GROUP BY c.day ORDER BY c.day;
-- GROUP BY the SPINE column; COUNT(o.id) not COUNT(*); COALESCE the aggregate.

-- MATRIX (dense grid): CROSS JOIN two sources
FROM dim d CROSS JOIN generate_series(...) c(day) LEFT JOIN facts ...

-- TEST DATA PRIMITIVES
gen_random_uuid()                        -- random v4 UUID (PK-safe)
md5(random()::text)                      -- 32-char hex token (may collide)
floor(random()*N)::int                   -- UNIFORM int [0, N-1]  (NOT (random()*N)::int)
(ARRAY['a','b','c'])[floor(random()*3+1)]-- random element (arrays 1-indexed)
start + random()*(stop - start)          -- random point in a range
setseed(0.42)                            -- reproducible random() (NOT gen_random_uuid)

-- BULK SEED (one statement, no client loop)
INSERT INTO users(...) SELECT ... FROM generate_series(1, :n) g(i);

-- PERF RULES OF THUMB
-- • Generation = pure CPU, no I/O, O(1) memory unless buffered by Sort/GroupBy/Materialize.
-- • Planner estimates a Function Scan at 1000 rows → persist big spines for real stats.
-- • Bucket predicate MUST be sargable: ts >= day AND ts < day+1, never DATE(ts)=day.
-- • Composite index (dim_id, ts) for matrix gap fills.
-- • Pre-aggregate huge fact tables, then LEFT JOIN the small spine to the small aggregate.

-- INTERVIEW ONE-LINERS
-- "GROUP BY only groups rows that exist — a spine manufactures the missing ones."
-- "Filter the right table in ON; a WHERE turns LEFT JOIN back into INNER JOIN."
-- "COUNT(o.id), not COUNT(*), or empty buckets read 1."
-- "Backwards range = 0 rows, silently — validate bounds in app code."
-- "floor(random()*N), not a cast — the cast half-weights the endpoints."
```

---

## Connected Topics

- **Topic 12 — LEFT JOIN and RIGHT JOIN**: the preserved-side semantics and the ON-vs-WHERE trap that make gap filling work; the entire spine pattern rests on LEFT JOIN.
- **Topic 33 — Sargable Predicates & Index Usage**: why `ts >= day AND ts < day+1` beats `DATE(ts) = day` in every bucketed gap-fill.
- **Topic 44/45 — Window Functions & Frames**: moving averages, carry-forward (last observation), and cohort math computed over a dense series.
- **Topic 47 — Gaps and Islands**: the deeper pattern for detecting and filling runs of missing/consecutive values.
- **Topic 61 — Pivot and Crosstab**: the previous topic; matrices generated here are frequently pivoted into wide crosstab layouts.
- **Topic 63 — Date and Time Mastery**: the next topic; interval arithmetic, `date_trunc`, DST, and time zones that the timestamp series depends on.
- **Topic 18 — The N+1 Query Problem**: generating rows server-side in one statement is the write-side dual of avoiding per-row client round trips.
- **Internals — Set-Returning Functions & Function Scan**: how the executor materializes SRF rows without touching the buffer pool, WAL, or MVCC.
