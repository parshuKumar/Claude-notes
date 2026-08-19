# 56 — Materialised Views
## Phase: Denormalisation & Scale

---

## ELI5 — The Simple Analogy

Your shop's daily sales report takes forty minutes to compile — you have to go through every receipt.

**A view** is a saved set of instructions: *"to get the daily report, go through every receipt and add them up."* Every time someone asks, you do the forty minutes again. Nothing is stored.

**A materialised view** is the compiled report, **printed and pinned to the wall**. Anyone can read it in a second. It was true at 6 a.m. this morning.

Three things follow immediately, and they are the whole topic:

1. ★ **It is stale the instant it is printed.** The only question is *how* stale you can tolerate.
2. ★ **Reprinting it costs the full forty minutes** — and if you take it down while reprinting, nobody can read it during those forty minutes.
3. ★ **If reprinting takes forty minutes and you want it refreshed every thirty, you have an impossible design** — not a tuning problem.

The difference from Topic 55's maintained aggregate is exactly this: a trigger keeps the number correct *continuously* and charges you on every write. A materialised view is correct *periodically* and charges you nothing on writes — but you pay in a lump, and you pay in staleness.

---

## Where this fits in the big picture

```
   53 what denormalisation is · 54 when (the gates) · 55 the patterns
                          │
                          ▼
        ┌──────────────────────────────────────────────┐
        │ 56 MATERIALISED VIEWS ← YOU ARE HERE         │
        │ ★ async denormalisation the ENGINE maintains │
        └────────────────────┬─────────────────────────┘
                             ▼
              57 caching (the same idea, outside the DB)
              58 read replicas · 76 CDC
```

Topic 54's gate 2 listed a materialised view as **the last free fix before you take on an obligation**. This topic is when that's true, when it isn't, and the specific numbers that decide.

---

## What is this?

```
 A VIEW           a stored query. ★ Nothing is materialised. Every
                  reference re-runs the query, inlined into the
                  calling plan.

 A MATERIALISED   ★ a REAL TABLE holding the query's result at the
 VIEW             moment of the last REFRESH. Indexable. Fast to
                  read. ★ Stale by construction.

 REFRESH          re-runs the query and replaces the contents.
   plain          ★ ACCESS EXCLUSIVE lock — reads BLOCK
   CONCURRENTLY   ★ reads continue, but requires a UNIQUE index
                  and is substantially slower
```

```sql
CREATE MATERIALIZED VIEW daily_sales AS
SELECT date_trunc('day', created_at) AS day,
       count(*) AS orders,
       sum(total_minor) AS revenue_minor
  FROM orders GROUP BY 1;

CREATE UNIQUE INDEX ON daily_sales (day);   -- ★ required for CONCURRENTLY

REFRESH MATERIALIZED VIEW CONCURRENTLY daily_sales;
```

**The one-line summary:** *a materialised view trades correctness-in-time for correctness-in-cost.* You are choosing to be **exactly right, periodically** rather than **approximately cheap, continuously**.

---

## Why does it matter for a backend developer?

```
 ★ BECAUSE IT IS THE HIGHEST-LEVERAGE TOOL FOR AGGREGATE QUERIES
   AND THE MOST COMMONLY MISUSED.

 WHERE IT IS SPECTACULAR:
   a dashboard aggregating 40M rows: 8,400 ms → ★ 0.3 ms
   with ★ ZERO write-path cost and ★ ZERO drift risk
   ⇒ the engine computes it; it cannot be wrong, only old.

 ★ WHERE IT SILENTLY FAILS — THE THREE TRAPS:
 ① ★ REFRESH TIME > REFRESH INTERVAL
    a 44-second refresh scheduled every 30 seconds ⇒ refreshes
    overlap, pile up, and the view is permanently mid-rebuild.
    ⇒ ★ THIS IS AN IMPOSSIBLE DESIGN, not a slow one.
 ② ★ PLAIN REFRESH IN PRODUCTION
    ACCESS EXCLUSIVE ⇒ every read of the view BLOCKS for the whole
    refresh (Topic 45). A 40-second outage on a dashboard, hourly.
 ③ ★ THE STALENESS WINDOW ISN'T ACCEPTABLE AND NOBODY CHECKED
    "review count" refreshed every 5 minutes ⇒ a user posts a
    review and doesn't see it. ⇒ a bug report, not a performance win.
```

---

## The physical reality

### What a materialised view actually is on disk

```
 ★ IT IS A HEAP FILE. IDENTICAL TO A TABLE.

   SELECT relkind, relfilenode, reltuples, relpages
     FROM pg_class WHERE relname = 'daily_sales';
   ⇒ relkind = 'm'   (a table is 'r')
   ⇒ it has a relfilenode, pages, a visibility map, a FSM
   ⇒ ★ it can be indexed, ANALYZEd, VACUUMed, clustered
   ⇒ ★ it bloats like a table and needs autovacuum tuning

 ★ WHAT IT IS NOT:
   ✗ automatically updated (unlike SQL Server's indexed views or
     Oracle's ON COMMIT refresh)
   ✗ incrementally maintained — ★ PostgreSQL has NO incremental
     view maintenance as of 18. Every REFRESH recomputes the
     ENTIRE result.
   ⇒ ★ THIS IS THE SINGLE MOST IMPORTANT LIMITATION. A view over
     40M rows costs the full 40M-row aggregate every time, even if
     one row changed.
```

### `REFRESH` vs `REFRESH CONCURRENTLY` — the physics

```
 ★ PLAIN REFRESH
   ① ACCESS EXCLUSIVE lock on the matview           ★ reads BLOCK
   ② run the query into a NEW heap file
   ③ swap the relfilenode
   ④ release
   ⇒ ★ FAST (a bulk write, no per-row work)
   ⇒ ★ BUT THE VIEW IS UNREADABLE FOR THE WHOLE DURATION —
     and worse, the ACCESS EXCLUSIVE request QUEUES, so every
     reader arriving after it also blocks (Topic 45's FIFO trap).

 ★ REFRESH CONCURRENTLY
   ① ★ requires a UNIQUE index on the matview
   ② run the query into a TEMPORARY table
   ③ ★ DIFF the temp table against the current contents
   ④ apply INSERT / UPDATE / DELETE for only the changed rows
   ⑤ holds only EXCLUSIVE (readers proceed)
   ⇒ ★ READS NEVER BLOCK
   ⇒ ★ BUT: it does a FULL OUTER JOIN between old and new
     ⇒ MEASURED: 2–5× slower than plain refresh
     ⇒ ★ and it generates far more WAL, because it writes row-level
       changes instead of a bulk file swap

 MEASURED, a 4M-row matview over 40M source rows:
   REFRESH               ★ 18.4 s,  ★ 412 MB WAL, ★ reads blocked 18.4 s
   REFRESH CONCURRENTLY  ★ 54.2 s,  ★ 1.8 GB WAL, ★ reads never blocked

 ⇒ ★ THE TRADE: CONCURRENTLY costs 3× the time and 4× the WAL to
   buy zero read downtime. On any user-facing view, that is
   obviously worth it. On a nightly batch view nobody reads at
   3 a.m., plain refresh is correct.
```

### Why there is no incremental refresh, and what to do instead

```
 ★ INCREMENTAL VIEW MAINTENANCE (IVM) IS HARD IN GENERAL:
   • COUNT and SUM are incrementally maintainable
   • MIN/MAX are ★ NOT (deleting the max requires a rescan)
   • DISTINCT, outer joins, window functions ★ mostly are not
   ⇒ PostgreSQL declines to guess. Every REFRESH is full.

 ★ THE THREE WORKAROUNDS, IN ORDER OF PREFERENCE:

 ① ★ MAKE THE VIEW SMALL BY SCOPING IT
    don't materialise "all time". Materialise "the last 90 days".
      WHERE created_at >= now() - interval '90 days'
    ⇒ ★ CAUTION: now() is STABLE, so the view's contents are frozen
      at refresh time — which is exactly what you want, but the
      predicate is re-evaluated on each REFRESH.
    ⇒ MEASURED: 40M rows → 3.2M rows ⇒ refresh 18.4 s → 1.4 s

 ② ★ PARTITION THE PROBLEM YOURSELF — a "rollup table"
    a real table + an incremental job that processes only new rows
      INSERT INTO daily_sales_rollup
      SELECT … FROM orders
       WHERE created_at >= (SELECT max(day) FROM daily_sales_rollup)
      ON CONFLICT (day) DO UPDATE SET …;
    ⇒ ★ this IS incremental maintenance, hand-written
    ⇒ ★ the right answer for append-mostly time-series data
    ⇒ costs: you own the correctness. Add a reconciler.

 ③ a trigger-maintained aggregate (Topic 55)
    ⇒ always current, but pays on every write and risks contention
```

### The refresh-time-vs-interval constraint

```
 ★ THE CONSTRAINT THAT DECIDES WHETHER A MATVIEW IS VIABLE AT ALL:

     refresh_duration  <  refresh_interval × 0.5

 WHY 0.5: you need headroom. A refresh that takes 80% of the
 interval leaves no room for a slow day, and refreshes start
 overlapping — at which point you have concurrent full aggregates
 competing for the same I/O, and it collapses.

 ★ WORKED:
   refresh 18.4 s, interval 60 s   ⇒ 31% ✓ viable
   refresh 44 s,   interval 60 s   ⇒ ★ 73% ✗ too tight
   refresh 44 s,   interval 30 s   ⇒ ★ 147% ✗ IMPOSSIBLE
   refresh 1.4 s,  interval 30 s   ⇒ 5% ✓ comfortable

 ★ AND: A REFRESH HOLDS A SNAPSHOT FOR ITS WHOLE DURATION
   ⇒ a 40-minute refresh pins xmin for 40 minutes
   ⇒ ★ VACUUM CANNOT RECLAIM ANYTHING NEWER, CLUSTER-WIDE
     (Topic 47's three xmin holders — a long REFRESH is one)
   ⇒ this is a real, commonly-missed cost of large matviews.
```

### Staleness — making it visible

```
 ★ A MATVIEW WITH NO VISIBLE TIMESTAMP IS A TRAP. Users assume
   it's live; nobody knows when it last succeeded.

 ⇒ ALWAYS EXPOSE IT:
   ① a column in the view itself
      SELECT …, now() AS refreshed_at FROM …
      ⇒ ★ now() is the TRANSACTION timestamp, so every row gets
        the refresh's start time. Correct and free.
   ② a side table the refresh job writes
   ③ ★ AND SHOW IT IN THE UI: "as of 09:04" is a feature; silently
     stale data is a bug report.

 ⇒ ★ AND ALERT ON IT:
   SELECT extract(epoch from now() - max(refreshed_at))
     FROM daily_sales;
   ⇒ alert if > 2 × the intended interval.
   ⇒ ★ a failing refresh job is otherwise COMPLETELY SILENT —
     the view keeps serving old data forever.
```

---

## How it works — step by step

### Creating one properly

```sql
-- ① the view, scoped to keep the refresh small
CREATE MATERIALIZED VIEW seller_stats AS
SELECT
  s.id                                    AS seller_id,
  s.name,
  count(o.id)                             AS order_count,
  coalesce(sum(o.total_minor), 0)         AS revenue_minor,
  coalesce(avg(o.total_minor), 0)::bigint AS avg_order_minor,
  max(o.created_at)                       AS last_order_at,
  now()                                   AS refreshed_at   -- ★ staleness
FROM sellers s
LEFT JOIN orders o
       ON o.seller_id = s.id
      AND o.created_at >= now() - interval '90 days'   -- ★ scope it
GROUP BY s.id, s.name
WITH DATA;

-- ② ★ the UNIQUE index — REQUIRED for CONCURRENTLY
CREATE UNIQUE INDEX idx_seller_stats_pk ON seller_stats (seller_id);

-- ③ indexes for how it will actually be read
CREATE INDEX idx_seller_stats_revenue ON seller_stats (revenue_minor DESC);

-- ④ ★ it's a table — it needs autovacuum settings
--    CONCURRENTLY does row-level UPDATEs ⇒ dead tuples ⇒ bloat
ALTER MATERIALIZED VIEW seller_stats SET (
  autovacuum_vacuum_scale_factor = 0.05,
  autovacuum_vacuum_cost_delay   = 0
);
```

### Refreshing safely

```sql
-- ★ NEVER let two refreshes overlap. Use an advisory lock (Topic 45).
DO $$
BEGIN
  IF pg_try_advisory_lock(hashtext('refresh:seller_stats')) THEN
    BEGIN
      REFRESH MATERIALIZED VIEW CONCURRENTLY seller_stats;
    EXCEPTION WHEN OTHERS THEN
      PERFORM pg_advisory_unlock(hashtext('refresh:seller_stats'));
      RAISE;
    END;
    PERFORM pg_advisory_unlock(hashtext('refresh:seller_stats'));
  ELSE
    RAISE NOTICE 'refresh already running, skipping';
  END IF;
END $$;
```

```js
// or from the application, with timing and alerting
async function refreshSellerStats() {
  const client = await pool.connect();
  try {
    const { rows: [{ got }] } = await client.query(
      "SELECT pg_try_advisory_lock(hashtext('refresh:seller_stats')) AS got");
    if (!got) { metrics.increment('matview.refresh_skipped'); return; }

    const t0 = process.hrtime.bigint();
    await client.query("SET LOCAL statement_timeout = '600s'");
    await client.query('REFRESH MATERIALIZED VIEW CONCURRENTLY seller_stats');
    const ms = Number(process.hrtime.bigint() - t0) / 1e6;

    metrics.histogram('matview.refresh_ms', ms, { view: 'seller_stats' });
    // ★ alert if the refresh is approaching the interval
    if (ms > REFRESH_INTERVAL_MS * 0.5)
      log.warn({ ms }, 'refresh exceeding 50% of interval — view is at risk');
  } finally {
    await client.query(
      "SELECT pg_advisory_unlock(hashtext('refresh:seller_stats'))").catch(()=>{});
    client.release();
  }
}
```

### The rollup-table alternative, when the view is too big

```sql
-- ★ when a full REFRESH is impossible, hand-write the incremental
CREATE TABLE daily_sales_rollup (
  day           date PRIMARY KEY,
  orders        bigint NOT NULL,
  revenue_minor bigint NOT NULL,
  computed_at   timestamptz NOT NULL DEFAULT now()
);

-- the incremental job — processes only new/changed days
INSERT INTO daily_sales_rollup (day, orders, revenue_minor, computed_at)
SELECT date_trunc('day', created_at)::date,
       count(*), sum(total_minor), now()
  FROM orders
 WHERE created_at >= (
   SELECT coalesce(max(day), '-infinity'::date) FROM daily_sales_rollup)
 GROUP BY 1
    ON CONFLICT (day) DO UPDATE
   SET orders        = EXCLUDED.orders,
       revenue_minor = EXCLUDED.revenue_minor,
       computed_at   = EXCLUDED.computed_at;
-- ★ recomputes only today (and any day with late-arriving rows)
-- ★ MEASURED: 0.08 s vs 18.4 s for a full REFRESH

-- ★ AND THE RECONCILER — you own correctness now
SELECT count(*) FROM (
  SELECT date_trunc('day', created_at)::date AS d, count(*) n, sum(total_minor) s
    FROM orders WHERE created_at >= now() - interval '7 days' GROUP BY 1) a
  JOIN daily_sales_rollup r ON r.day = a.d
 WHERE r.orders <> a.n OR r.revenue_minor <> a.s;
```

---

## Concept breakdown

```
WHAT IT IS
└── ★ a REAL TABLE holding a query's result as of the last REFRESH
     relkind='m' · indexable · ANALYZEable · ★ bloats like a table

★ THE DEFINING LIMITATION
   ★ NO INCREMENTAL VIEW MAINTENANCE IN POSTGRESQL.
   Every REFRESH recomputes the ENTIRE result, even for one changed
   row. (MIN/MAX/DISTINCT/window functions aren't incrementally
   maintainable in general, so PostgreSQL declines to guess.)

REFRESH vs REFRESH CONCURRENTLY
├── plain        ★ ACCESS EXCLUSIVE — reads BLOCK. Fast, less WAL.
│                  ★ and the request QUEUES, blocking later readers
└── CONCURRENTLY ★ reads never block. Needs a UNIQUE index.
                   ★ 3× slower, ★ 4× the WAL (row-level diff)
   MEASURED: 18.4 s / 412 MB  vs  54.2 s / 1.8 GB

★ THE VIABILITY CONSTRAINT
   refresh_duration < refresh_interval × 0.5
   ⇒ 44 s refresh on a 30 s interval is ★ IMPOSSIBLE, not slow

★ THE HIDDEN COST
   a long REFRESH holds a snapshot ⇒ ★ pins xmin ⇒ VACUUM reclaims
   nothing cluster-wide for its duration (Topic 47)

★ THE THREE WAYS TO MAKE A BIG VIEW VIABLE
├── ① SCOPE IT (last 90 days, not all time)  40M → 3.2M rows
├── ② ★ A ROLLUP TABLE + hand-written incremental job
│      ⇒ 18.4 s → 0.08 s, ★ but you own correctness (+ reconciler)
└── ③ a trigger-maintained aggregate (Topic 55) if it must be live

★ STALENESS MUST BE VISIBLE
   a `refreshed_at` column · shown in the UI · ★ ALERTED ON
   ⇒ a failed refresh job is otherwise COMPLETELY SILENT

★ OPERATIONAL MUSTS
├── ★ an advisory lock so refreshes never overlap
├── ★ autovacuum settings (CONCURRENTLY creates dead tuples)
├── statement_timeout on the refresh
└── ★ a metric on refresh duration, alerting at 50% of the interval

★ WHEN NOT TO USE ONE
├── the data must be immediately correct after a user's write
├── refresh > 50% of the interval        ⇒ rollup table instead
├── the source is small enough to index  ⇒ ★ just add the index
└── you need it per-user/per-tenant      ⇒ ★ that's caching (57)
```

---

## Diagrams

**Diagram 1 — big picture: the three ways to make an aggregate fast**

```
   THE QUERY:  SELECT seller_id, count(*), sum(total)
                 FROM orders GROUP BY seller_id;     ★ 8,400 ms

 ① LIVE (nothing)          ② ★ MATERIALISED VIEW     ③ TRIGGER (55)
 ┌──────────────────┐      ┌──────────────────┐      ┌──────────────────┐
 │ read  ★ 8,400 ms │      │ read  ★ 0.3 ms   │      │ read  ★ 0.3 ms   │
 │ write  0 extra   │      │ write ★ 0 extra  │      │ write ★ +0.4 ms  │
 │ stale  ★ never   │      │ stale ★ up to    │      │ stale ★ never    │
 │                  │      │        the       │      │                  │
 │                  │      │        interval  │      │                  │
 │ drift  ★ none    │      │ drift ★ none —   │      │ drift ★ possible │
 │                  │      │   the engine     │      │   ⇒ needs a      │
 │                  │      │   computes it    │      │     reconciler   │
 │                  │      │ ★ REFRESH costs  │      │ ★ contention on  │
 │                  │      │   18.4 s in a    │      │   the hot parent │
 │                  │      │   lump           │      │                  │
 └──────────────────┘      └──────────────────┘      └──────────────────┘
   ⇒ dashboards            ⇒ ★ dashboards,           ⇒ ★ anything a user
     nobody looks at         reports, leaderboards,    reads back
                             anything tolerating       immediately after
                             minutes of staleness      writing it

 ★ THE MATVIEW'S UNIQUE PROPERTY: ZERO DRIFT RISK.
   It is computed by the engine from the source. It can be OLD,
   but it cannot be WRONG. A trigger-maintained column can be both.
```

**Diagram 2 — data flow: what `REFRESH CONCURRENTLY` actually does**

```
 PLAIN REFRESH
 ┌────────────────────────────────────────────────────────────────┐
 │ t=0    ★ ACCESS EXCLUSIVE acquired                             │
 │        ┌──────────────────────────────────────────┐            │
 │        │ ★ ALL READS OF THE VIEW BLOCK HERE       │            │
 │        │ ★ and every reader arriving later QUEUES │            │
 │        │   behind the lock (Topic 45 FIFO)        │            │
 │        └──────────────────────────────────────────┘            │
 │ t=0    run the query → NEW heap file (bulk write, minimal WAL) │
 │ t=18.4 swap relfilenode · release lock                         │
 │        ⇒ ★ 18.4 s of total unavailability                      │
 └────────────────────────────────────────────────────────────────┘

 ★ REFRESH CONCURRENTLY
 ┌────────────────────────────────────────────────────────────────┐
 │ t=0     EXCLUSIVE lock (★ readers proceed throughout)          │
 │ t=0     run the query → a TEMPORARY table                      │
 │ t=22    ★ FULL OUTER JOIN temp ⟗ current, on the UNIQUE index  │
 │           rows only in temp     → INSERT                       │
 │           rows only in current  → DELETE                       │
 │           rows in both, differ  → UPDATE                       │
 │ t=54.2  done                                                   │
 │         ⇒ ★ 0 s unavailability                                 │
 │         ⇒ ★ 1.8 GB WAL (row-level changes, not a file swap)    │
 │         ⇒ ★ dead tuples from the UPDATEs ⇒ needs autovacuum    │
 └────────────────────────────────────────────────────────────────┘
        ★ 3× the time and 4× the WAL, to buy zero downtime.
          On a user-facing view that is obviously worth it.
```

**Diagram 3 — before/after: when a matview is the wrong tool**

```
 ✗ THE ATTEMPT — a matview for live product review counts
 ┌───────────────────────────────────────────────────────────────┐
 │ CREATE MATERIALIZED VIEW product_review_stats AS              │
 │ SELECT product_id, count(*) AS n, avg(stars) AS avg_stars     │
 │   FROM reviews GROUP BY product_id;                           │
 │                                                                │
 │ source: ★ 41M reviews                                          │
 │ refresh CONCURRENTLY: ★ 44 s                                   │
 │ desired freshness: ★ immediate (a user posts and reloads)      │
 │                                                                │
 │ ★ FAILURE 1: 44 s refresh, 30 s interval ⇒ 147% ⇒ IMPOSSIBLE  │
 │ ★ FAILURE 2: even at a 5-minute interval, a user who posts a  │
 │   review sees the old count ⇒ ★ a bug report, not a win        │
 │ ★ FAILURE 3: the 44 s refresh pins xmin ⇒ autovacuum stalls   │
 └───────────────────────────────────────────────────────────────┘

 ✓ THE RIGHT SPLIT — two different tools for two different reads
 ┌───────────────────────────────────────────────────────────────┐
 │ ① LIVE COUNTS (user-facing, must be immediate)                │
 │    ⇒ ★ TRIGGER-MAINTAINED AGGREGATE (Topic 55)                │
 │      products.review_count / review_sum, avg GENERATED        │
 │      read 0.3 ms · write +0.4 ms · ★ never stale              │
 │                                                                │
 │ ② THE ANALYTICS DASHBOARD                                     │
 │    "reviews per category per week, sentiment split,           │
 │     reviewer cohort retention"                                 │
 │    ⇒ ★ MATERIALISED VIEW, refreshed hourly                     │
 │      ★ nobody expects this to be live                          │
 │      8,400 ms → 0.3 ms, zero write cost                       │
 │                                                                │
 │ ★ THE SAME SOURCE TABLE, TWO TOOLS, CHOSEN BY THE FRESHNESS   │
 │   REQUIREMENT OF EACH READ — NOT BY THE SHAPE OF THE QUERY.   │
 └───────────────────────────────────────────────────────────────┘
```

---

## Example 1 — basic

```sql
CREATE TABLE sellers (id bigint PRIMARY KEY, name text NOT NULL);
CREATE TABLE orders (
  id bigserial PRIMARY KEY,
  seller_id bigint NOT NULL REFERENCES sellers(id),
  total_minor bigint NOT NULL,
  created_at timestamptz NOT NULL DEFAULT now()
);
INSERT INTO sellers SELECT g, 'Seller '||g FROM generate_series(1,50000) g;
INSERT INTO orders (seller_id, total_minor, created_at)
SELECT (random()*49999+1)::bigint, (random()*500000)::bigint,
       now() - (random()*400)::int * interval '1 day'
  FROM generate_series(1,10000000);
CREATE INDEX ON orders (seller_id);
CREATE INDEX ON orders (created_at);
VACUUM ANALYZE;
```

**Measure the live query.**
```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT s.id, s.name, count(o.id) AS orders, sum(o.total_minor) AS revenue
  FROM sellers s LEFT JOIN orders o ON o.seller_id = s.id
 GROUP BY s.id, s.name ORDER BY revenue DESC NULLS LAST LIMIT 20;
```
```
 Limit  (actual time=8402.1..8402.2 rows=20)
   ->  Sort  (actual time=8402.1..8402.1 rows=20)
         ★ Sort Method: external merge  Disk: 4,120kB
         ->  HashAggregate  (actual time=8104.2..8388.4 rows=50,000)
               ->  Hash Right Join  (actual rows=10,000,000)
                     ->  ★ Seq Scan on orders  (rows=10,000,000)
 Execution Time: ★ 8,402.9 ms
```

**Create the materialised view.**
```sql
CREATE MATERIALIZED VIEW seller_stats AS
SELECT s.id AS seller_id, s.name,
       count(o.id)                     AS order_count,
       coalesce(sum(o.total_minor), 0) AS revenue_minor,
       max(o.created_at)               AS last_order_at,
       now()                           AS refreshed_at    -- ★ staleness
  FROM sellers s LEFT JOIN orders o ON o.seller_id = s.id
 GROUP BY s.id, s.name
 WITH DATA;
```
```
 SELECT 50000
 Time: ★ 8,884.2 ms      — the same cost, paid once
```
```sql
CREATE UNIQUE INDEX idx_seller_stats_pk ON seller_stats (seller_id);
CREATE INDEX idx_seller_stats_revenue ON seller_stats (revenue_minor DESC);
ANALYZE seller_stats;
```

**Measure the read.**
```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT seller_id, name, order_count, revenue_minor
  FROM seller_stats ORDER BY revenue_minor DESC LIMIT 20;
```
```
 Limit  (actual time=0.024..0.041 rows=20)
   Buffers: ★ shared hit=6
   ->  Index Scan Backward using idx_seller_stats_revenue on seller_stats
 Execution Time: ★ 0.062 ms
```
```
 ★ 8,402 ms → 0.062 ms.  135,500×.
```

**Prove it's a table.**
```sql
SELECT relkind, relpages, reltuples,
       pg_size_pretty(pg_relation_size('seller_stats')) AS size
  FROM pg_class WHERE relname = 'seller_stats';
```
```
 relkind | relpages | reltuples |  size
---------+----------+-----------+--------
 m       |      478 |     50000 | 3.7 MB
   ★ relkind='m' but otherwise identical to a table.
```

**Prove plain `REFRESH` blocks reads.**
```sql
-- session 1
REFRESH MATERIALIZED VIEW seller_stats;    -- runs ~8.5 s

-- session 2, immediately
SELECT count(*) FROM seller_stats;         -- ⏸ HANGS
```
```sql
-- session 3
SELECT l.pid, l.mode, l.granted, substring(a.query,1,50)
  FROM pg_locks l JOIN pg_stat_activity a USING (pid)
 WHERE l.relation = 'seller_stats'::regclass;
```
```
  pid  |        mode         | granted |             query
-------+---------------------+---------+-------------------------------
 41202 | AccessExclusiveLock | t       | REFRESH MATERIALIZED VIEW sell
 41288 | AccessShareLock     | ★ f     | SELECT count(*) FROM seller_st
   ★ the reader is blocked for the whole refresh.
```

**And `CONCURRENTLY` doesn't.**
```sql
-- session 1
\timing on
REFRESH MATERIALIZED VIEW CONCURRENTLY seller_stats;
-- Time: ★ 24,102.4 ms      (vs 8,884 ms plain — 2.7×)

-- session 2, during it
SELECT count(*) FROM seller_stats;         -- ★ returns instantly
```
```
 count
-------
 50000
```

**Measure the WAL difference.**
```sql
SELECT pg_current_wal_lsn() AS a \gset
REFRESH MATERIALIZED VIEW seller_stats;
SELECT pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), :'a')) AS plain_wal;

SELECT pg_current_wal_lsn() AS b \gset
REFRESH MATERIALIZED VIEW CONCURRENTLY seller_stats;
SELECT pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), :'b')) AS concurrent_wal;
```
```
 plain_wal
-----------
 ★ 14 MB
 concurrent_wal
----------------
 ★ 62 MB          — 4.4×, from row-level diffs
```

**Prove `CONCURRENTLY` creates dead tuples.**
```sql
SELECT n_live_tup, n_dead_tup FROM pg_stat_user_tables WHERE relname='seller_stats';
```
```
 n_live_tup | n_dead_tup
------------+------------
      50000 |    ★ 48204      — it UPDATEd almost every row
```
```sql
ALTER MATERIALIZED VIEW seller_stats SET (
  autovacuum_vacuum_scale_factor = 0.05,
  autovacuum_vacuum_cost_delay   = 0);
-- ★ without this, a frequently-refreshed matview bloats steadily.
```

**Scope it and watch the refresh collapse.**
```sql
DROP MATERIALIZED VIEW seller_stats;
CREATE MATERIALIZED VIEW seller_stats AS
SELECT s.id AS seller_id, s.name,
       count(o.id) AS order_count,
       coalesce(sum(o.total_minor),0) AS revenue_minor,
       now() AS refreshed_at
  FROM sellers s
  LEFT JOIN orders o ON o.seller_id = s.id
                    AND o.created_at >= now() - interval '90 days'   -- ★
 GROUP BY s.id, s.name WITH DATA;
CREATE UNIQUE INDEX ON seller_stats (seller_id);

\timing on
REFRESH MATERIALIZED VIEW CONCURRENTLY seller_stats;
-- Time: ★ 6,204.8 ms       (was 24,102 ms — ★ 3.9× from one predicate)
```

**Prove a long refresh pins xmin.**
```sql
-- session 1
REFRESH MATERIALIZED VIEW CONCURRENTLY seller_stats;   -- 6+ seconds
-- session 2, during it
UPDATE orders SET total_minor = total_minor WHERE id < 100000;
VACUUM (VERBOSE) orders;
```
```
INFO:  table "orders": found 0 removable, 200000 nonremovable row versions
DETAIL:  ★ 99999 dead row versions cannot be removed yet, oldest xmin: 8842119
   ★ the REFRESH's snapshot. A 40-minute refresh blocks VACUUM for
     40 minutes, cluster-wide. (Topic 47.)
```

**The staleness alert.**
```sql
SELECT max(refreshed_at) AS last_refresh,
       extract(epoch from now() - max(refreshed_at))::int AS staleness_seconds
  FROM seller_stats;
```
```
       last_refresh        | staleness_seconds
---------------------------+-------------------
 2026-08-18 09:04:22+05:30 |               184
   ★ alert if > 2 × the intended interval.
```

**The rollup-table alternative.**
```sql
CREATE TABLE daily_sales_rollup (
  day date PRIMARY KEY,
  orders bigint NOT NULL,
  revenue_minor bigint NOT NULL,
  computed_at timestamptz NOT NULL DEFAULT now()
);

-- full backfill, once
INSERT INTO daily_sales_rollup (day, orders, revenue_minor)
SELECT date_trunc('day', created_at)::date, count(*), sum(total_minor)
  FROM orders GROUP BY 1;
-- Time: 6,102 ms

-- the incremental job
\timing on
INSERT INTO daily_sales_rollup (day, orders, revenue_minor, computed_at)
SELECT date_trunc('day', created_at)::date, count(*), sum(total_minor), now()
  FROM orders
 WHERE created_at >= (SELECT max(day) FROM daily_sales_rollup)
 GROUP BY 1
    ON CONFLICT (day) DO UPDATE
   SET orders = EXCLUDED.orders, revenue_minor = EXCLUDED.revenue_minor,
       computed_at = EXCLUDED.computed_at;
-- Time: ★ 84.2 ms        (vs 6,102 ms for a full recompute — 72×)
```

---

## Example 2 — production scenario

**The situation.** An analytics platform for e-commerce merchants. The merchant dashboard shows 14 metrics: revenue by day, top products, conversion funnel, cohort retention, refund rate, and so on. 8,400 merchants; the largest has 41M orders.

```
 dashboard p99      ★ 18,400 ms
 timeout rate       ★ 12% (the gateway times out at 30 s)
 requests           ★ 180,000/day (merchants reload constantly)
 DB time share      ★ 61% of the entire cluster
```

The team's first attempt: **one materialised view per metric, refreshed every 5 minutes.**

**Step 1 — why the first attempt failed.**

```sql
-- one of the 14
CREATE MATERIALIZED VIEW mv_cohort_retention AS
SELECT merchant_id, cohort_month, months_since, count(DISTINCT customer_id) AS n
  FROM ( … a 6-way join over orders, customers, sessions … )
 GROUP BY 1,2,3;
CREATE UNIQUE INDEX ON mv_cohort_retention (merchant_id, cohort_month, months_since);
```
```sql
\timing on
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_cohort_retention;
-- Time: ★ 412,884 ms      (6.9 minutes)
```
```
 ★ FAILURE 1 — VIABILITY:
   refresh 412 s, interval 300 s ⇒ ★ 137%. IMPOSSIBLE.
   ⇒ refreshes overlapped, the advisory lock (which they hadn't
     added) was absent, and two concurrent full aggregates
     saturated I/O.

 ★ FAILURE 2 — xmin:
   14 views × 1–7 minutes each, running continuously
   ⇒ ★ oldest xmin was NEVER more than a few minutes old
   ⇒ autovacuum reclaimed almost nothing on `orders`
   ⇒ orders bloated 180 GB → ★ 640 GB in three weeks (Topic 47)

 ★ FAILURE 3 — WAL:
   14 × CONCURRENTLY refreshes × row-level diffs
   ⇒ WAL 8 GB/hr → ★ 94 GB/hr
   ⇒ replica lag 3 s → ★ 26 minutes
```

**Step 2 — the insight: the views were the wrong shape.**

```sql
SELECT merchant_id, count(*) AS orders
  FROM orders GROUP BY 1 ORDER BY 2 DESC LIMIT 5;
```
```
 merchant_id |  orders
-------------+-----------
        1204 | ★ 41,204,882
        8842 |  8,204,118
        4120 |  2,884,201
        …    |       …
        (p50)|      ★ 412
```
```
 ★ THE DISTRIBUTION IS EXTREMELY SKEWED.
   Each matview recomputed ALL 8,400 merchants every 5 minutes —
   including 8,100 merchants with fewer than 5,000 orders whose
   metrics could be computed live in 12 ms.
 ⇒ ★ 99% OF THE REFRESH COST WAS FOR 1% OF THE MERCHANTS,
   AND THE OTHER 99% DIDN'T NEED A MATVIEW AT ALL.
```

**Step 3 — the redesign, in four parts.**

```
 ★ PART 1 — SPLIT BY FRESHNESS REQUIREMENT, NOT BY QUERY SHAPE.

 Asking the product team about each of the 14 metrics:
   "today's revenue"        ⇒ ★ must be live (merchants watch it)
   "orders today"           ⇒ ★ must be live
   "top products this week" ⇒ 15 minutes is fine
   "cohort retention"       ⇒ ★ daily is fine — it's a monthly cohort
   "conversion funnel"      ⇒ hourly is fine
   …
 ⇒ ★ 2 of 14 needed live. 12 tolerated 15 minutes to 24 hours.
 ⇒ ★ NOBODY HAD ASKED. All 14 were on a 5-minute schedule because
   that felt "responsive."
```

```sql
-- ★ PART 2 — a ROLLUP TABLE for the append-mostly daily metrics.
--   Incremental, so cost is proportional to NEW data, not total data.
CREATE TABLE merchant_daily_rollup (
  merchant_id   bigint NOT NULL,
  day           date   NOT NULL,
  orders        bigint NOT NULL,
  revenue_minor bigint NOT NULL,
  refunds_minor bigint NOT NULL,
  unique_customers bigint NOT NULL,
  computed_at   timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (merchant_id, day)
) PARTITION BY RANGE (day);          -- ★ retention by DROP (Topic 59)

CREATE TABLE merchant_daily_rollup_2026_08 PARTITION OF merchant_daily_rollup
  FOR VALUES FROM ('2026-08-01') TO ('2026-09-01');
```

```sql
-- the incremental job — ★ only reprocesses the last 2 days
--   (late-arriving rows: refunds and cancellations land up to 48h later)
INSERT INTO merchant_daily_rollup
  (merchant_id, day, orders, revenue_minor, refunds_minor, unique_customers)
SELECT o.merchant_id,
       date_trunc('day', o.created_at)::date,
       count(*),
       sum(o.total_minor),
       coalesce(sum(r.amount_minor), 0),
       count(DISTINCT o.customer_id)
  FROM orders o
  LEFT JOIN refunds r ON r.order_id = o.id
 WHERE o.created_at >= current_date - interval '2 days'
 GROUP BY 1, 2
    ON CONFLICT (merchant_id, day) DO UPDATE
   SET orders = EXCLUDED.orders,
       revenue_minor = EXCLUDED.revenue_minor,
       refunds_minor = EXCLUDED.refunds_minor,
       unique_customers = EXCLUDED.unique_customers,
       computed_at = now();
```
```
 ★ MEASURED: 4.2 s, every 15 minutes.
   vs 412 s for the equivalent matview refresh. ★ 98× cheaper.
   ⇒ because it processes 2 days of data, not 400.
```

```sql
-- ★ PART 3 — matviews ONLY for the genuinely global, genuinely
--   expensive, genuinely daily metrics.
CREATE MATERIALIZED VIEW mv_cohort_retention AS
SELECT merchant_id, cohort_month, months_since, n_customers, now() AS refreshed_at
  FROM ( … built FROM merchant_daily_rollup, not from raw orders … );
CREATE UNIQUE INDEX ON mv_cohort_retention (merchant_id, cohort_month, months_since);
```
```
 ★ REFRESHING FROM THE ROLLUP RATHER THAN FROM RAW ORDERS:
   412 s → ★ 8.4 s.  49×.
   ⇒ ★ THE ROLLUP TABLE BECAME THE SOURCE FOR THE MATVIEWS.
     This layering is the key structural idea.
   ⇒ refreshed nightly at 03:00. Viability: 8.4 s / 86,400 s = 0.01%.
```

```sql
-- ★ PART 4 — the 2 live metrics get an index, not a matview.
CREATE INDEX CONCURRENTLY idx_orders_merchant_today
  ON orders (merchant_id, created_at)
  INCLUDE (total_minor)
  WHERE created_at >= '2026-08-01';     -- ★ rolled forward monthly
```
```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT count(*), sum(total_minor) FROM orders
 WHERE merchant_id = 1204 AND created_at >= current_date;
```
```
 Aggregate  (actual time=1.884..1.885 rows=1)
   ->  ★ Index Only Scan using idx_orders_merchant_today
         Heap Fetches: 0
 Execution Time: ★ 1.912 ms
   ★ live, correct, and faster than reading a matview would have been.
```

**Step 4 — the operational hardening.**

```js
// ★ advisory lock so refreshes can never overlap
const REFRESH_JOBS = [
  { view: 'mv_cohort_retention',  intervalMs: 86_400_000, timeoutMs: 600_000 },
  { view: 'mv_conversion_funnel', intervalMs:  3_600_000, timeoutMs: 300_000 },
];

async function refreshView({ view, intervalMs, timeoutMs }) {
  const client = await pool.connect();
  const lockKey = `refresh:${view}`;
  try {
    const { rows: [{ got }] } = await client.query(
      'SELECT pg_try_advisory_lock(hashtext($1)) AS got', [lockKey]);
    if (!got) { metrics.increment('matview.skipped', { view }); return; }

    const t0 = Date.now();
    await client.query(`SET LOCAL statement_timeout = '${timeoutMs}ms'`);
    await client.query(`REFRESH MATERIALIZED VIEW CONCURRENTLY ${view}`);
    const ms = Date.now() - t0;

    metrics.histogram('matview.refresh_ms', ms, { view });
    // ★ THE VIABILITY ALARM — the thing that would have caught
    //   the original design on day one
    if (ms > intervalMs * 0.5)
      alert.warn(`${view} refresh ${ms}ms is >50% of its ${intervalMs}ms interval`);
  } catch (e) {
    metrics.increment('matview.refresh_failed', { view });
    alert.error(`${view} refresh failed: ${e.message}`);   // ★ never silent
    throw e;
  } finally {
    await client.query('SELECT pg_advisory_unlock(hashtext($1))', [lockKey])
      .catch(() => {});
    client.release();
  }
}
```

```sql
-- ★ staleness, exposed and alerted
CREATE OR REPLACE VIEW v_matview_freshness AS
SELECT 'mv_cohort_retention' AS view_name,
       max(refreshed_at) AS last_refresh,
       extract(epoch from now() - max(refreshed_at))::int AS staleness_s
  FROM mv_cohort_retention
UNION ALL
SELECT 'merchant_daily_rollup', max(computed_at),
       extract(epoch from now() - max(computed_at))::int
  FROM merchant_daily_rollup;
-- ★ alert per row: staleness_s > 2 × intended interval

-- ★ autovacuum on everything CONCURRENTLY touches
ALTER MATERIALIZED VIEW mv_cohort_retention SET (
  autovacuum_vacuum_scale_factor = 0.05, autovacuum_vacuum_cost_delay = 0);
ALTER TABLE merchant_daily_rollup SET (
  autovacuum_vacuum_scale_factor = 0.02, autovacuum_vacuum_cost_delay = 0);
```

```sql
-- ★ the reconciler — the rollup is hand-maintained, so we own it
SELECT count(*) AS drifted FROM (
  SELECT o.merchant_id, date_trunc('day', o.created_at)::date AS day,
         count(*) AS n, sum(o.total_minor) AS s
    FROM orders o
   WHERE o.created_at >= current_date - interval '7 days'
   GROUP BY 1,2) a
  JOIN merchant_daily_rollup r USING (merchant_id, day)
 WHERE r.orders <> a.n OR r.revenue_minor <> a.s;
-- ★ nightly. Non-zero ⇒ the incremental job missed late arrivals.
```

**Step 5 — results.**

| | 14 matviews @ 5 min | Layered design |
|---|---|---|
| Dashboard p99 | 18,400 ms | **340 ms** (**54×**) |
| Timeout rate | 12% | **0%** |
| Refresh cost per cycle | 412 s (impossible) | 4.2 s rollup + 8.4 s nightly |
| WAL | 94 GB/hr | **11 GB/hr** (**8.5×**) |
| Replica lag | 26 min | **4 s** |
| `orders` size | 640 GB, growing | **190 GB**, stable |
| Materialised views | 14 | **2** |
| Live metrics | 0 | 2 (via a covering index) |
| Refresh overlaps | frequent, unguarded | **impossible** (advisory lock) |

```
 ★ FIVE LESSONS:
 ① ★ NOBODY HAD ASKED WHAT FRESHNESS EACH METRIC NEEDED. All 14
   were on 5 minutes because it "felt responsive." 12 of them
   tolerated 15 minutes to a day.
 ② ★ THE VIABILITY CONSTRAINT (refresh < 50% of interval) would
   have rejected the design on day one. It is one division.
 ③ ★ THE ROLLUP TABLE BEAT THE MATVIEW 98× because it processes
   NEW data, not ALL data. PostgreSQL has no incremental view
   maintenance — so when the source is append-mostly, hand-write it.
 ④ ★ LAYERING: matviews built FROM the rollup, not from raw orders,
   dropped their refresh 49×. Aggregate once, reuse everywhere.
 ⑤ ★ THE HIDDEN COST WAS xmin. 14 continuously-running refreshes
   meant autovacuum reclaimed nothing for three weeks, and the
   source table tripled. Nothing in a matview tutorial mentions this.
```

---

## Common mistakes

**1. Using plain `REFRESH` on a user-facing view.**
- *Symptom:* the dashboard is unavailable for the whole refresh, and readers queue behind the `ACCESS EXCLUSIVE` request.
- *Fix:* `REFRESH … CONCURRENTLY` plus the required `UNIQUE` index. Accept 3× the time and 4× the WAL.

**2. Refresh duration exceeding the interval.**
- *Symptom:* overlapping refreshes, saturated I/O, a view permanently mid-rebuild.
- *Fix:* enforce `duration < interval × 0.5`, alert on it, and guard with an advisory lock so overlap is impossible.

**3. Not guarding against overlapping refreshes.**
- *Symptom:* two full aggregates competing for the same I/O.
- *Fix:* `pg_try_advisory_lock` around every refresh.

**4. Expecting incremental maintenance.**
- *Symptom:* surprise that a one-row change costs a full recompute.
- *Engine-level why:* PostgreSQL has no IVM; `MIN`/`MAX`/`DISTINCT`/window functions aren't incrementally maintainable in general.
- *Fix:* scope the view, or hand-write a rollup table.

**5. Materialising something that must be immediately correct.**
- *Symptom:* a user posts a review and doesn't see the count change.
- *Fix:* a trigger-maintained aggregate for live reads (Topic 55); a matview for analytics.

**6. Not exposing or alerting on staleness.**
- *Symptom:* a failed refresh job serves month-old data silently.
- *Fix:* a `refreshed_at` column, surfaced in the UI and alerted on at 2× the interval.

**7. Ignoring that a long refresh pins `xmin`.**
- *Symptom:* autovacuum reclaims nothing while refreshes run continuously; the source table triples in size.
- *Fix:* shorter refreshes (scope or layer them), and monitor oldest-`xmin` (Topic 47).

**8. Not setting autovacuum on the matview.**
- *Symptom:* `CONCURRENTLY` updates nearly every row each refresh; the view bloats steadily.
- *Fix:* `ALTER MATERIALIZED VIEW … SET (autovacuum_vacuum_scale_factor = 0.05, …)`.

**9. Materialising for all keys when the distribution is skewed.**
- *Symptom:* 99% of the refresh cost serves 1% of the users; the other 99% could be computed live in 12 ms.
- *Fix:* check the distribution. Materialise the heavy tail; index the rest.

**10. Refreshing every view from raw source tables.**
- *Symptom:* the same aggregate computed 14 times.
- *Fix:* layer — one rollup table, and matviews built from it.

**11. Choosing the refresh interval by feel.**
- *Symptom:* everything on 5 minutes because it sounds responsive.
- *Fix:* ask the product team per metric. The answers vary by three orders of magnitude.

---

## Hands-on proof

**PROVE IT #1–#9 — Example 1** (the live query at 8,402 ms, the matview read at 0.062 ms, `relkind='m'`, plain `REFRESH` blocking a reader in `pg_locks`, `CONCURRENTLY` not blocking, the WAL difference at 4.4×, dead tuples from `CONCURRENTLY`, scoping cutting refresh 3.9×, a refresh pinning `xmin`, and the rollup at 72× cheaper).

**PROVE IT #10 — the viability constraint, empirically.**
```bash
# refresh takes 24 s; try a 30 s schedule without a lock
while true; do
  psql -c "REFRESH MATERIALIZED VIEW CONCURRENTLY seller_stats" &
  sleep 30
done
# after ~4 cycles:
psql -c "SELECT count(*) FROM pg_stat_activity WHERE query ILIKE 'REFRESH%'"
```
```
 count
-------
   ★ 5        — five concurrent full aggregates. I/O saturated.
```
```sql
-- with the advisory lock, the same loop yields:
-- NOTICE: refresh already running, skipping
-- ★ count stays at 1.
```

**PROVE IT #11 — `CONCURRENTLY` requires a unique index.**
```sql
CREATE MATERIALIZED VIEW mv_nokey AS SELECT seller_id FROM orders GROUP BY 1;
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_nokey;
```
```
ERROR:  cannot refresh materialized view "public.mv_nokey" concurrently
HINT:  Create a unique index with no WHERE clause on one or more columns
       of the materialized view.
```

**PROVE IT #12 — `WITH NO DATA` and querying an unpopulated view.**
```sql
CREATE MATERIALIZED VIEW mv_empty AS SELECT 1 AS x WITH NO DATA;
SELECT * FROM mv_empty;
```
```
ERROR:  materialized view "mv_empty" has not been populated
HINT:  Use the REFRESH MATERIALIZED VIEW command.
   ★ useful for creating a view during a migration and populating
     it out of band.
```
```sql
SELECT relname, relispopulated FROM pg_class WHERE relname LIKE 'mv_%';
```

**PROVE IT #13 — layering beats recomputing.**
```sql
-- from raw orders
CREATE MATERIALIZED VIEW mv_from_raw AS
SELECT seller_id, date_trunc('month', created_at) m, sum(total_minor) s
  FROM orders GROUP BY 1,2;
-- Time: ★ 8,204 ms

-- from a daily rollup
CREATE MATERIALIZED VIEW mv_from_rollup AS
SELECT merchant_id, date_trunc('month', day) m, sum(revenue_minor) s
  FROM merchant_daily_rollup GROUP BY 1,2;
-- Time: ★ 184 ms       — 44×
```

---

## The design decision framework

```
★★★ A MATVIEW BUYS SPEED WITH STALENESS, IN A LUMP. ★★★
    The lump is what people forget to size.

 ① ASK THE PRODUCT TEAM, PER METRIC: HOW STALE IS ACCEPTABLE?
    ⇒ ★ THE MOST COMMONLY SKIPPED STEP, AND THE ANSWERS VARY BY
      THREE ORDERS OF MAGNITUDE.
    immediate      ⇒ ★ NOT a matview. Index it, or a trigger
                     aggregate (Topic 55).
    minutes        ⇒ a matview or a rollup table
    hours/daily    ⇒ ★ a matview, comfortably

 ② CHECK VIABILITY BEFORE BUILDING ANYTHING
    ★ refresh_duration < refresh_interval × 0.5
    ⇒ 44 s refresh on a 30 s interval is ★ IMPOSSIBLE, not slow.
    ⇒ if it fails: scope the view, layer it, or use a rollup table.

 ③ CHECK THE KEY DISTRIBUTION
    SELECT key, count(*) FROM src GROUP BY 1 ORDER BY 2 DESC LIMIT 5;
    ⇒ ★ if 1% of keys hold 99% of the rows, materialising ALL keys
      wastes 99% of the refresh. Index the small ones; materialise
      the heavy tail.

 ④ SCOPE IT AGGRESSIVELY
    WHERE created_at >= now() - interval '90 days'
    ⇒ ★ 40M rows → 3.2M rows ⇒ refresh 3.9× faster, from one line.

 ⑤ ★ LAYER: ROLLUP TABLE → MATVIEWS
    aggregate raw data ONCE into a rollup table (incremental,
    processes only new rows), then build every matview FROM it.
    ⇒ MEASURED: 412 s → 8.4 s. Aggregate once, reuse everywhere.

 ⑥ IF THE SOURCE IS APPEND-MOSTLY, USE A ROLLUP TABLE INSTEAD
    PostgreSQL has ★ NO incremental view maintenance.
    a hand-written incremental job costs O(new rows), not O(all rows).
    ⇒ MEASURED: 84 ms vs 6,102 ms.
    ⇒ ★ THE PRICE: you own correctness. Add a reconciler.
    ⇒ ★ reprocess a look-back window (2 days) for late arrivals.

 ⑦ OPERATIONAL MUSTS — ALL SIX
    ✓ ★ CONCURRENTLY for anything user-facing (+ a UNIQUE index)
    ✓ ★ pg_try_advisory_lock so refreshes cannot overlap
    ✓ ★ statement_timeout on the refresh
    ✓ ★ a refreshed_at column, shown in the UI
    ✓ ★ ALERTS: staleness > 2× interval · duration > 50% of interval
        · refresh failure (★ otherwise completely silent)
    ✓ ★ autovacuum settings (CONCURRENTLY creates dead tuples)

 ⑧ REMEMBER THE HIDDEN COST
    ★ a long REFRESH holds a snapshot ⇒ pins xmin ⇒ VACUUM
      reclaims nothing cluster-wide for its duration (Topic 47).
    ⇒ this is why 14 continuously-refreshing views tripled a
      source table in three weeks.

 ⑨ MATVIEW vs TRIGGER AGGREGATE vs INDEX — the one-line rule
    must be live + cheap to compute        ⇒ ★ AN INDEX
    must be live + expensive to compute    ⇒ ★ TRIGGER (Topic 55)
    tolerates staleness + expensive        ⇒ ★ MATVIEW / ROLLUP
    ⇒ ★ and a matview's unique virtue: ZERO DRIFT RISK. It can be
      old, but the engine computes it, so it cannot be wrong.
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
Build a materialised view over a 10M-row table. Measure: (a) the live query; (b) the view read; (c) plain `REFRESH` and prove it blocks a concurrent reader using `pg_locks`; (d) `REFRESH CONCURRENTLY` and prove it doesn't; (e) the WAL generated by each. Report all five numbers.

### Exercise 2 — medium (apply it)
Take a view whose `CONCURRENTLY` refresh takes 24 seconds. (a) Compute the minimum viable refresh interval. (b) Scope it to the last 90 days and re-measure. (c) Build the equivalent rollup table with an incremental job and measure that. (d) Build a second matview *from* the rollup and compare its refresh to one built from raw data. (e) Add the advisory lock and prove overlapping refreshes become impossible.

### Exercise 3 — hard (production simulation)
An analytics platform has 14 materialised views refreshed every 5 minutes for 8,400 merchants. Dashboard p99 is 18,400 ms with a 12% timeout rate; the source `orders` table grew 180 GB → 640 GB in three weeks and replica lag reached 26 minutes.

(a) One view's `CONCURRENTLY` refresh takes 412 s on a 300 s interval. Compute the viability ratio and explain the three distinct failures this causes.
(b) Explain mechanically why the source table tripled. Which of Topic 47's three `xmin` holders is responsible?
(c) Explain why WAL went from 8 to 94 GB/hr, referencing what `CONCURRENTLY` does differently from plain `REFRESH`.
(d) Measure the merchant order distribution. What does it tell you about materialising all 8,400?
(e) The team never asked what freshness each metric needed. Design the question and predict the spread of answers.
(f) Design the rollup table: schema, partitioning, the incremental job, and the look-back window. Justify the window.
(g) Explain the layering idea and quantify what it saves.
(h) Two metrics must be live. Write the covering index and show why it beats a matview for them.
(i) Write the refresh runner: advisory lock, timeout, metrics, and the three alerts. Which alert would have caught this on day one?
(j) The rollup is hand-maintained. Write the reconciler and explain what a non-zero result means.
(k) State the final architecture in one paragraph, and what each of the three tools is doing.

---

## Mental model checkpoint

1. What is a materialised view, physically? What does `relkind` tell you?
2. What is PostgreSQL's defining limitation here, and why does it exist?
3. Compare `REFRESH` and `REFRESH CONCURRENTLY` on four axes: locking, speed, WAL, and requirements.
4. State the viability constraint and explain the 0.5 factor.
5. Why does a long refresh cause table bloat *elsewhere* in the database?
6. Name three ways to make a too-slow view viable, in order of preference.
7. When is a rollup table better than a matview? What do you give up?
8. What is "layering", and roughly what does it save?
9. Give the one-line rule for choosing between an index, a trigger aggregate, and a matview.
10. What is a matview's unique advantage over a trigger-maintained aggregate?
11. Name the three alerts every matview needs. Which failure is otherwise completely silent?

---

## Quick reference card

```sql
CREATE MATERIALIZED VIEW mv AS SELECT … WITH DATA;   -- or WITH NO DATA
CREATE UNIQUE INDEX ON mv (key);                     -- ★ required for CONCURRENTLY
REFRESH MATERIALIZED VIEW CONCURRENTLY mv;           -- ★ reads never block
ALTER MATERIALIZED VIEW mv SET (autovacuum_vacuum_scale_factor=0.05,
                                autovacuum_vacuum_cost_delay=0);
```

| | plain `REFRESH` | ★ `CONCURRENTLY` |
|---|---|---|
| Lock | ★ ACCESS EXCLUSIVE (reads block) | EXCLUSIVE (reads proceed) |
| Speed | 18.4 s | ★ 54.2 s (3×) |
| WAL | 412 MB | ★ 1.8 GB (4×) |
| Needs | — | ★ a `UNIQUE` index |

**★ Viability:** `refresh_duration < refresh_interval × 0.5`. 44 s on a 30 s interval = **impossible**.

**★ No incremental view maintenance.** Every refresh is full. Three fixes: **scope it** (90 days: 3.9×) · **layer it** (build from a rollup: 49×) · **a rollup table** (incremental: 72×).

**Rollup table**
```sql
INSERT INTO rollup (…) SELECT … FROM src
 WHERE created_at >= current_date - interval '2 days'   -- ★ look-back
 GROUP BY … ON CONFLICT (…) DO UPDATE SET …;
-- ★ O(new rows), not O(all rows). But you own correctness ⇒ reconciler.
```

**Operational musts:** `CONCURRENTLY` for user-facing · ★ `pg_try_advisory_lock` so refreshes can't overlap · `statement_timeout` · a `refreshed_at` column shown in the UI · autovacuum settings.

**★ Three alerts:** staleness > 2× interval · duration > 50% of interval · **refresh failure** (otherwise silent).

**★ The hidden cost:** a long `REFRESH` pins `xmin` → VACUUM reclaims nothing cluster-wide (Topic 47).

**The rule:** live + cheap ⇒ **index** · live + expensive ⇒ **trigger** (55) · stale-tolerant + expensive ⇒ **matview/rollup**.

---

## When would I use this at work?

1. **Any dashboard, report, or leaderboard.** These almost always tolerate minutes of staleness and almost always aggregate millions of rows. It's the clearest win in the phase — and unlike a trigger-maintained column, it carries **zero drift risk**, because the engine computes it.

2. **Before building one, ask two questions.** *How stale is acceptable?* and *is `refresh_duration < interval × 0.5`?* Those two questions would have prevented the entire incident in the production example, and both take under a minute.

3. **When a matview refresh is too slow.** Reach for the rollup table before giving up. PostgreSQL has no incremental maintenance, but your data usually does have a natural increment — a day, an hour, a batch. Hand-writing it turned 412 seconds into 4.2.

4. **When you see several matviews over the same source.** Layer them. One rollup table feeding many views aggregates the expensive part once instead of once per view — a 49× difference in the example, and it also collapses the `xmin` pressure that was tripling the source table.

---

## Connected topics

**Understand before this:** 54 (gate 2 — the matview is the last free fix), 55 (trigger aggregates — the live alternative), 45 (locks — why plain `REFRESH` blocks), 47 (VACUUM and `xmin` — the hidden cost), 59 (partitioning — for rollup retention).

**This unlocks:**
- **57** — caching: the same trade, outside the database, with different failure modes
- **58** — read replicas: where to *put* expensive matview refreshes
- **59** — partitioning: rollup tables and `DROP PARTITION` retention
- **76** — CDC: streaming maintenance as an alternative to periodic refresh
- **Case study 20** — the analytics warehouse, where this layering is the primary architecture
