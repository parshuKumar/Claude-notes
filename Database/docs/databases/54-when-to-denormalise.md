# 54 — When to Denormalise
## Phase: Denormalisation & Scale

---

## ELI5 — The Simple Analogy

You're deciding whether to buy a car.

The wrong way: *"cars are faster than walking, so I should buy one."* True, and useless. It ignores the price, the insurance, the parking, the maintenance, and — critically — **whether you actually go anywhere far enough for it to matter.**

The right way is four questions, in order:

1. **Where am I actually going?** (Measure. Not where you imagine you'll go.)
2. **Is there a cheaper thing that solves it?** A bus pass. A bicycle. Moving closer to work.
3. **How often, versus how much does it cost to own?** Two trips a year doesn't justify it. Two trips a day does.
4. **Can I actually maintain it?** A car you can't service becomes a liability parked outside your house.

★ **Denormalisation is exactly this decision, and people skip straight to step one's conclusion.** "Joins are slow, so denormalise" is "cars are fast, so buy one."

The previous topic told you what the loan is. **This topic is the credit check.**

---

## Where this fits in the big picture

```
   53 what denormalisation IS — the definition and the obligation
                          │
                          ▼
        ┌──────────────────────────────────────────────┐
        │ 54 WHEN TO DENORMALISE ← YOU ARE HERE        │
        │ ★ the decision, with measured thresholds     │
        └────────────────────┬─────────────────────────┘
                             ▼
              55 the patterns (HOW)
              56 materialised views · 57 caching · 61 hot rows
```

Topic 53 established the trade. **This topic is the checklist that produces a defensible yes or no**, with numbers you can put in a pull request.

---

## What is this?

A decision procedure with **five gates**. A proposal must pass all five. Failing any one is a no.

```
 GATE 1  ★ IS THERE A MEASURED PROBLEM?
         a specific query, a specific p99, a specific SLO it misses

 GATE 2  ★ HAVE THE FREE FIXES BEEN EXHAUSTED?
         index · N+1 · work_mem · query rewrite · matview

 GATE 3  ★ IS IT A COPY, OR A SNAPSHOT?
         a snapshot has no obligation. Most aren't copies.

 GATE 4  ★ DOES THE ARITHMETIC WORK?
         read:write ratio × fan-out × contention

 GATE 5  ★ CAN YOU DISCHARGE THE OBLIGATION?
         mechanism + reconciler + constraint + owner
```

**The point of gates rather than a score:** each one eliminates a *different* class of bad proposal, and in practice **most proposals die at gate 2** — the problem was a missing index, not a join.

---

## Why does it matter for a backend developer?

```
 ★ BECAUSE THE INSTINCT IS ALWAYS WRONG IN THE SAME DIRECTION.

 WHAT PEOPLE BELIEVE            WHAT MEASUREMENT SHOWS
 ────────────────────────────   ─────────────────────────────────
 "joins are slow"                a single-row join on an indexed
                                 FK is ★ 0.01 ms
 "we have too many tables"       table count is not a performance
                                 metric
 "denormalise for scale"         ★ the write amplification often
                                 costs MORE at scale than the join
 "we'll keep it in sync"         ★ you will, for 7 months, until
                                 a second team touches the table
 "we can always normalise
  it back later"                 ★ NO. Once four services read the
                                 column and two write it, removing
                                 it is a multi-quarter migration.

 ⇒ ★ DENORMALISATION IS THE MOST COMMONLY PROPOSED AND LEAST
   COMMONLY JUSTIFIED OPTIMISATION IN BACKEND ENGINEERING.
 ⇒ AND IT IS ONE OF THE FEW THAT IS GENUINELY HARD TO UNDO.
```

---

## The physical reality

### Gate 2 in detail — the five free fixes, and what each is worth

```
 ★ MEASURED ON THE SAME 4M-ROW `orders` TABLE, SAME QUERY.
   Every one of these costs ZERO correctness risk.

 ① A COMPOSITE INDEX MATCHING THE ACCESS PATTERN
    CREATE INDEX ON orders (customer_id, created_at DESC);
    ⇒ 1,284 ms → 1.18 ms      ★ 1,087×
    ⇒ ★ THE SINGLE HIGHEST-VALUE FIX IN THIS ENTIRE PHASE.
      More queries are fixed by one index than by everything else
      in Topics 53–61 combined.

 ② FIXING AN N+1
    200 round trips of 0.4 ms each = 80 ms, mostly network
    ⇒ one query with WHERE id = ANY($1) = 2.1 ms   ★ 38×
    ⇒ ★ AND DENORMALISATION DOESN'T FIX THIS AT ALL — you'd still
      make 200 round trips, just faster ones. (Topic 66.)

 ③ AN INDEX-ONLY SCAN (a covering index)
    CREATE INDEX ON orders (customer_id) INCLUDE (total_minor, status);
    ⇒ heap fetches eliminated: 41,204 → 0
    ⇒ 88 ms → 4 ms            ★ 22×
    ⇒ ★ THIS IS DENORMALISATION THE ENGINE MAINTAINS FOR YOU.
      Same benefit, zero obligation. (Topic 12.)

 ④ work_mem
    external merge Disk: 412 MB → in-memory quicksort
    ⇒ 1,284 ms → 340 ms       ★ 3.8×
    ⇒ per-query: SET LOCAL work_mem = '256MB';

 ⑤ A MATERIALISED VIEW (for aggregates)
    ⇒ 2,100 ms → 0.3 ms       ★ 7,000×
    ⇒ ★ cost: a staleness window, not a correctness obligation.
      The database maintains it on REFRESH. (Topic 56.)

 ⇒ ★ IF YOU HAVEN'T TRIED ALL FIVE, GATE 2 IS NOT PASSED.
```

### Gate 4 — the arithmetic, made concrete

```
 ★ THREE NUMBERS. GET THEM FROM pg_stat_statements, NOT FROM
   INTUITION.

 ① READ FREQUENCY — how many times per day is the slow read run?
 ② WRITE FREQUENCY — how often does the source change?
 ③ ★ FAN-OUT — how many rows does ONE source change touch?

 THE CALCULATION:
   benefit = (read_ms_saved) × (reads/day)
   cost    = (write_ms_added) × (writes/day) × (fan-out factor)

 ★ WORKED, THREE REAL CASES:

 CASE A — products.review_count
   read saved   2,100 ms  × 8,400,000 reads/day = ★ 4,900 hours/day
   write added  0.5 ms    ×    41,000 writes/day × 1 row
                                                 = ★ 20 seconds/day
   ⇒ RATIO 880,000 : 1.   ★ OBVIOUSLY YES.

 CASE B — orders.customer_name
   read saved   0.01 ms   ×   410,000 reads/day  = ★ 4 seconds/day
   write added  0.3 ms    ×     4,100 writes/day × ★ 8,402 rows avg
                                                 = ★ 2.9 hours/day
   ⇒ RATIO 1 : 2,600.   ★ OBVIOUSLY NO — and this is the one
     people propose most often.

 CASE C — categories.product_count
   read saved   40 ms     × 2,100,000 reads/day  = ★ 23 hours/day
   write added  0.4 ms    ×   180,000 writes/day × 1 row
                                                 = ★ 72 seconds/day
   ⇒ RATIO 1,150 : 1.   ✓ YES — ★ but check contention:
     one category takes 41,000 of those writes/day = 0.47/sec.
     ⇒ fine. At 400/sec it would not be. (Topic 61.)

 ⇒ ★ FAN-OUT IS THE TERM PEOPLE FORGET, AND IT IS USUALLY THE ONE
   THAT KILLS THE PROPOSAL.
```

### The contention check — the fifth number

```
 ★ THE ARITHMETIC CAN SAY YES AND THE DESIGN STILL FAIL.

 A maintained counter serialises every writer on ONE ROW.
 Throughput ceiling ≈ 1 / (row lock hold time).

 MEASURED, a trigger-maintained counter on one row:
   lock hold time ≈ 0.3 ms
   ⇒ theoretical ceiling ≈ 3,300 writes/sec
   ⇒ ★ practical ceiling with contention overhead ≈ 800/sec
   ⇒ ★ AND EVERY WRITER'S LATENCY RISES AS YOU APPROACH IT

 THE QUERY THAT DECIDES:
   SELECT key_column, count(*) AS writes_per_day
     FROM child_table WHERE created_at > now() - interval '1 day'
    GROUP BY 1 ORDER BY 2 DESC LIMIT 5;
   ⇒ ★ LOOK AT THE MAXIMUM, NOT THE AVERAGE. The average is
     always fine. The hottest key is what breaks.

   < 100/sec on the hottest key   ⇒ ✓ fine
   100–800/sec                    ⇒ ⚠ measure p99 under load
   > 800/sec                      ⇒ ★ you have built a hot row.
                                    Shard it (Topic 61) or use an
                                    async aggregate (Topic 56).
```

### Gate 5 — what "discharging the obligation" actually requires

```
 ★ FOUR ARTEFACTS. ALL FOUR, IN THE SAME PULL REQUEST.

 ① THE MECHANISM
    generated column > trigger > same-txn app code > async
    ⇒ ★ if more than one service writes the source table, it MUST
      be a trigger or generated column. App code will drift.

 ② ★ THE RECONCILER
    a query proving copy = source, scheduled, alerting on non-zero
    + a self-repair statement
    ⇒ sampled (TABLESAMPLE) hourly for big tables, full nightly

 ③ THE CONSTRAINT
    CHECK (count >= 0), CHECK (sum <= count * max)
    ⇒ ★ catches absurd values at write time, not in a report

 ④ ★ THE OWNER AND THE WHY
    COMMENT ON COLUMN … IS 'Denormalised from X. Maintained by
      trg_Y. Reconciler: ops/Z.sql (hourly). Added 2026-08 because
      the aggregate was 88% of a 2.4s p99. Owner: payments-team.'
    ⇒ ★ THE NEXT ENGINEER MUST BE ABLE TO FIND THE OBLIGATION
      WITHOUT ARCHAEOLOGY.

 ⇒ ★ IF ANY OF THE FOUR IS MISSING, THE ANSWER IS NO —
   REGARDLESS OF HOW GOOD THE BENCHMARK IS.
```

### The cases where the answer is *always* yes

```
 ★ FOUR SITUATIONS WHERE GATES 1–4 ARE EFFECTIVELY PRE-PASSED.

 ① ★ THE JOIN CROSSES A SERVICE BOUNDARY
    you literally cannot write the JOIN. The alternative is an
    HTTP call per row.
    ⇒ ★ THE STRONGEST GENUINE CASE FOR DENORMALISATION IN MODERN
      SYSTEMS. Sync via events (Topic 52's outbox) + a reconciler.

 ② ★ IT IS A SNAPSHOT, NOT A COPY
    the price charged, the address shipped to, the tax rate applied
    ⇒ ★ NO OBLIGATION AT ALL. This isn't even denormalisation —
      it's correct 3NF modelling of a historical fact (Topic 38).
    ⇒ ★ AND THIS IS THE MAJORITY OF "denormalised" COLUMNS IN
      REAL E-COMMERCE SCHEMAS.

 ③ ★ AN AGGREGATE OVER A LARGE CHILD TABLE, READ CONSTANTLY
    review_count over 40M reviews, on a page served 8M times/day
    ⇒ no index makes count(*) over 214 rows × 41,204 products fast
    ⇒ the arithmetic is not close.

 ④ ★ AN ANALYTICAL/REPORTING STORE
    a star schema, a warehouse, a read replica shaped for reports
    ⇒ ★ the whole point is denormalisation, the source of truth is
      elsewhere, and the reconciler is the ETL pipeline itself.
    ⇒ ★ DIFFERENT RULES APPLY — don't import OLTP normalisation
      dogma into a warehouse. (Topics 74, 76; case study 20.)
```

### The cases where the answer is *almost always* no

```
 ✗ ① "TO AVOID A JOIN" WITH NO MEASUREMENT
      ⇒ gate 1 fails.

 ✗ ② A COPIED COLUMN WITH LARGE FAN-OUT
      customer_name into 8,402 orders
      ⇒ gate 4 fails by three orders of magnitude.

 ✗ ③ ★ TO FIX AN N+1
      ⇒ the problem is 200 round trips, not the join.
      ⇒ gate 2 fails. (Topic 66.)

 ✗ ④ ★ "FOR SCALE", SPECULATIVELY, BEFORE THE TRAFFIC EXISTS
      ⇒ gate 1 fails. And you have now made the schema harder to
        change during the period when you most need to change it.

 ✗ ⑤ WHEN THE VALUE IS A PURE FUNCTION OF THE SAME ROW
      ⇒ ★ use a GENERATED column. It's not denormalisation.

 ✗ ⑥ ★ WHEN NOBODY WILL OWN THE RECONCILER
      ⇒ gate 5 fails. This is a real and frequent reason.

 ✗ ⑦ ★ ON A ROW THAT IS ALREADY HOT
      ⇒ correctness fine, throughput destroyed. (Topic 61.)
```

---

## How it works — step by step

### The five gates, as a runnable checklist

```
 ═══ GATE 1 — IS THERE A MEASURED PROBLEM? ═══════════════════
 □ a named query, from pg_stat_statements, with calls and
   mean/p99 time
 □ a stated SLO it misses ("this endpoint must be <100 ms p99")
 □ EXPLAIN (ANALYZE, BUFFERS) at production data volume
 □ ★ COST ATTRIBUTED across the plan — what % is the join?
 ⇒ ★ "it feels slow" and "joins are expensive" are not gate 1.

 ═══ GATE 2 — ARE THE FREE FIXES EXHAUSTED? ══════════════════
 □ composite index matching the WHERE + ORDER BY
 □ partial index if there's a selective predicate
 □ covering index for an index-only scan
 □ ★ N+1 eliminated (one query, not N)
 □ work_mem raised if there's an external sort/hash
 □ query rewritten (lateral, EXISTS instead of IN, etc.)
 □ materialised view considered for aggregates
 ⇒ ★ MOST PROPOSALS DIE HERE. Re-measure after each.

 ═══ GATE 3 — COPY OR SNAPSHOT? ══════════════════════════════
 □ "if the source changes, must this change too?"
   NO  ⇒ ★ snapshot. Name it `*_at_time`, `*_charged`. DONE —
          no obligation, no mechanism, no reconciler.
   YES ⇒ continue to gate 4.

 ═══ GATE 4 — DOES THE ARITHMETIC WORK? ══════════════════════
 □ reads/day of the slow query          (pg_stat_statements)
 □ writes/day of the source             (pg_stat_statements)
 □ ★ FAN-OUT: rows touched per source write
 □ benefit = ms_saved × reads/day
 □ cost    = ms_added × writes/day × fan-out
 □ ★ CONTENTION: writes/sec on the HOTTEST key (not the average)
 ⇒ benefit:cost < 100 ⇒ ★ no
 ⇒ hottest key > 800 writes/sec ⇒ ★ no (or shard it — Topic 61)

 ═══ GATE 5 — CAN YOU DISCHARGE THE OBLIGATION? ══════════════
 □ mechanism named: generated / trigger / same-txn / async
 □ ★ if >1 service writes the source ⇒ MUST be trigger or generated
 □ ★ reconciliation query written, scheduled, alerting
 □ ★ self-repair statement written
 □ CHECK constraints added
 □ ★ COMMENT ON COLUMN with the why, the mechanism, the reconciler
 □ an owning team named
 ⇒ ★ ANY BOX UNCHECKED ⇒ NO.
```

### The queries that answer gate 4

```sql
-- reads/day and cost of the slow query
SELECT calls, mean_exec_time::numeric(10,2) AS mean_ms,
       (calls * mean_exec_time / 1000 / 3600)::numeric(10,1) AS hours_per_period,
       substring(query from 1 for 80) AS q
  FROM pg_stat_statements
 ORDER BY calls * mean_exec_time DESC LIMIT 10;

-- writes/day on the source
SELECT calls, mean_exec_time::numeric(10,2) AS mean_ms, substring(query,1,80)
  FROM pg_stat_statements
 WHERE query ILIKE 'UPDATE customers%' OR query ILIKE 'INSERT INTO reviews%';

-- ★ FAN-OUT — the term people forget
SELECT
  avg(cnt)::numeric(10,1)  AS avg_rows_per_source,
  max(cnt)                 AS ★ max_rows_per_source,
  percentile_cont(0.99) WITHIN GROUP (ORDER BY cnt) AS p99
FROM (SELECT customer_id, count(*) AS cnt FROM orders GROUP BY 1) x;

-- ★ CONTENTION — writes/sec on the hottest key
SELECT product_id, count(*) AS writes_today,
       (count(*)/86400.0)::numeric(10,4) AS writes_per_sec
  FROM reviews WHERE created_at > now() - interval '1 day'
 GROUP BY 1 ORDER BY 2 DESC LIMIT 5;
```

---

## Concept breakdown

```
★ THE FIVE GATES — ALL MUST PASS
├── ① MEASURED PROBLEM
│    a named query · a stated SLO · EXPLAIN at production volume
│    ★ + COST ATTRIBUTED across the plan
├── ② ★ FREE FIXES EXHAUSTED  ← MOST PROPOSALS DIE HERE
│    composite index (★ 1,087×) · N+1 (38×) · covering index (22×)
│    · work_mem (3.8×) · matview (7,000×)
│    ⇒ ★ all five cost ZERO correctness risk
├── ③ COPY vs SNAPSHOT
│    "if the source changes, must this?"  NO ⇒ ★ no obligation
├── ④ THE ARITHMETIC
│    benefit = ms_saved × reads/day
│    cost    = ms_added × writes/day × ★ FAN-OUT
│    + ★ CONTENTION on the HOTTEST key
└── ⑤ ★ THE OBLIGATION DISCHARGEABLE
     mechanism + ★ reconciler + constraint + ★ owner & comment

★ THE THRESHOLDS
├── benefit:cost < 100        ⇒ no
├── benefit:cost > 10,000     ⇒ yes
├── hottest key > 800 w/s     ⇒ ★ hot row — shard or go async
└── >1 writing service        ⇒ ★ trigger/generated ONLY

★ ALWAYS-YES CASES
├── ★ the join crosses a SERVICE BOUNDARY (the strongest case)
├── ★ it's a SNAPSHOT (most e-commerce "denormalisation" is this)
├── an aggregate over a large child table, read constantly
└── an analytical/warehouse store (★ different rules entirely)

★ ALMOST-ALWAYS-NO CASES
├── "to avoid a join", unmeasured
├── ★ a copied column with large FAN-OUT (the classic mistake)
├── ★ to fix an N+1 (wrong diagnosis)
├── ★ speculatively "for scale"
├── a pure function of the same row (★ use GENERATED)
├── ★ nobody will own the reconciler
└── ★ on an already-hot row

★ THE ASYMMETRY THAT MAKES THIS SERIOUS
   adding a denormalised column: one PR
   removing one after 4 services read it: ★ a multi-quarter migration
   ⇒ "we can normalise it back later" is not true.
```

---

## Diagrams

**Diagram 1 — big picture: the five gates as a filter**

```
   100 PROPOSALS ARRIVE
        │
        ▼
 ┌─────────────────────────────────────────────────────────────┐
 │ GATE 1 — measured problem? named query, SLO, EXPLAIN,       │
 │          ★ cost attributed                                   │
 └─────────────────────────────────┬───────────────────────────┘
        ★ 38 rejected ("feels slow", "joins are expensive")
        │ 62 pass
        ▼
 ┌─────────────────────────────────────────────────────────────┐
 │ GATE 2 — ★ free fixes exhausted?                            │
 │   index · N+1 · covering · work_mem · matview                │
 └─────────────────────────────────┬───────────────────────────┘
        ★ 44 rejected — ★ THE BIGGEST FILTER BY FAR
        │ 18 pass                    (an index fixed it)
        ▼
 ┌─────────────────────────────────────────────────────────────┐
 │ GATE 3 — copy or snapshot?                                  │
 └─────────────────────────────────┬───────────────────────────┘
        ★ 7 are SNAPSHOTS ⇒ ship them, ★ zero obligation
        │ 11 are real copies
        ▼
 ┌─────────────────────────────────────────────────────────────┐
 │ GATE 4 — arithmetic: ms×reads vs ms×writes×★FAN-OUT         │
 │          + contention on the hottest key                     │
 └─────────────────────────────────┬───────────────────────────┘
        ★ 6 rejected (fan-out, or a hot row)
        │ 5 pass
        ▼
 ┌─────────────────────────────────────────────────────────────┐
 │ GATE 5 — mechanism + ★ reconciler + constraint + owner      │
 └─────────────────────────────────┬───────────────────────────┘
        ★ 2 rejected (nobody will own the reconciler)
        │
        ▼
   ★ 3 SHIP.  (+ 7 snapshots, which were never the same thing.)
```

**Diagram 2 — data flow: gate 4 worked, three ways**

```
 CASE A — products.review_count          ★ RATIO 880,000 : 1  ⇒ YES
 ┌────────────────────────────────────────────────────────────────┐
 │ BENEFIT  2,100 ms saved × 8,400,000 reads/day                  │
 │          = ████████████████████████████████ 4,900 hours/day    │
 │ COST     0.5 ms × 41,000 writes/day × 1 row (fan-out)          │
 │          = ▏ 20 seconds/day                                    │
 │ CONTENTION  hottest product: 41 reviews/day = 0.0005/sec  ✓    │
 └────────────────────────────────────────────────────────────────┘

 CASE B — orders.customer_name           ★ RATIO 1 : 2,600  ⇒ NO
 ┌────────────────────────────────────────────────────────────────┐
 │ BENEFIT  0.01 ms saved × 410,000 reads/day                     │
 │          = ▏ 4 seconds/day                                     │
 │ COST     0.3 ms × 4,100 writes/day × ★ 8,402 rows (FAN-OUT)    │
 │          = ██████████████████ 2.9 hours/day                    │
 │ ⇒ ★ THE FAN-OUT IS THE ENTIRE STORY. Without it the numbers    │
 │   look survivable; with it they are absurd.                    │
 └────────────────────────────────────────────────────────────────┘

 CASE C — categories.product_count       RATIO 1,150 : 1  ⇒ YES*
 ┌────────────────────────────────────────────────────────────────┐
 │ BENEFIT  40 ms × 2,100,000 reads/day = ███████ 23 hours/day    │
 │ COST     0.4 ms × 180,000 writes/day × 1 = ▏ 72 seconds/day    │
 │ ★ CONTENTION  hottest category: 41,000 writes/day = 0.47/sec ✓ │
 │ ⇒ ★ *YES — but if that category took 400,000 writes/day        │
 │   (4.6/sec) it'd still be fine, and at 70,000,000/day          │
 │   (810/sec) it would NOT. ★ ALWAYS CHECK THE MAX, NOT THE AVG. │
 └────────────────────────────────────────────────────────────────┘
```

**Diagram 3 — before/after: a proposal that failed gate 2, and one that passed all five**

```
 ✗ PROPOSAL A — "denormalise seller data onto products"
 ┌───────────────────────────────────────────────────────────────┐
 │ GATE 1 ✓  p99 2,400 ms, SLO 200 ms, EXPLAIN captured          │
 │ GATE 2 ✗  ★ COST ATTRIBUTION SHOWED:                          │
 │              seq scan   180 ms   (missing index)              │
 │              seller join  82 ms   ← ★ what they wanted to fix  │
 │              reviews agg 2,100 ms ← ★ 88% of the total         │
 │              sort         90 ms   (work_mem / index)           │
 │           after ONE INDEX: seller join runs 24× not 41,204×    │
 │           ⇒ ★ costs 0.05 ms. Nothing left to denormalise.      │
 │                                                                │
 │ ⇒ ★ REJECTED AT GATE 2. Saved 2 columns of permanent          │
 │   obligation and 2 reconcilers.                               │
 └───────────────────────────────────────────────────────────────┘

 ✓ PROPOSAL B — "maintain products.review_count / review_sum"
 ┌───────────────────────────────────────────────────────────────┐
 │ GATE 1 ✓  the aggregate is 2,100 ms of a 2,400 ms p99         │
 │ GATE 2 ✓  no index makes count(*) over 8.8M rows fast;        │
 │           matview rejected — ★ counts must be immediate        │
 │ GATE 3 ✓  a real COPY ("new review ⇒ count must change")       │
 │ GATE 4 ✓  ★ 880,000 : 1, fan-out 1, hottest key 0.0005/sec     │
 │ GATE 5 ✓  trigger (3 services write reviews ⇒ ★ must be a      │
 │             trigger)                                           │
 │           ★ reconciler: hourly TABLESAMPLE + nightly full      │
 │             + self-repair                                      │
 │           CHECK (review_count >= 0),                           │
 │           CHECK (review_sum <= review_count * 5)               │
 │           ★ avg_rating as a GENERATED column ⇒ cannot drift    │
 │           COMMENT ON COLUMN with why/mechanism/reconciler      │
 │                                                                │
 │ ⇒ ★ SHIPPED. p99 2,400 → 11 ms. Write cost +0.47 ms.          │
 └───────────────────────────────────────────────────────────────┘
```

---

## Example 1 — basic

```sql
-- the schema from Topic 53
CREATE TABLE customers (id bigint PRIMARY KEY, name text NOT NULL);
CREATE TABLE orders (
  id bigserial PRIMARY KEY,
  customer_id bigint NOT NULL REFERENCES customers(id),
  total_minor bigint NOT NULL,
  created_at timestamptz NOT NULL DEFAULT now()
);
INSERT INTO customers SELECT g, 'Customer '||g FROM generate_series(1,100000) g;
INSERT INTO orders (customer_id, total_minor, created_at)
SELECT (random()*99999+1)::bigint, (random()*500000)::bigint,
       now() - (random()*365)::int * interval '1 day'
  FROM generate_series(1,4000000);
VACUUM ANALYZE;
```

**Gate 1 — establish the measured problem.**
```sql
SELECT pg_stat_statements_reset();
-- (run the workload)
SELECT calls, mean_exec_time::numeric(10,2) AS mean_ms,
       (calls*mean_exec_time/1000)::numeric(10,1) AS total_seconds,
       substring(query from 1 for 60) AS q
  FROM pg_stat_statements ORDER BY calls*mean_exec_time DESC LIMIT 3;
```
```
 calls  | mean_ms | total_seconds |                    q
--------+---------+---------------+------------------------------------------
 412088 |  1284.2 |    ★ 529,204 | SELECT o.id, o.total_minor, c.name FROM o
   ★ 529,204 seconds = 147 hours of database time. This is a
     measured problem.
```

**Gate 1 continued — attribute the cost.**
```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT o.id, o.total_minor, c.name
  FROM orders o JOIN customers c ON c.id = o.customer_id
 WHERE o.customer_id = 42 ORDER BY o.created_at DESC LIMIT 50;
```
```
 Limit  (actual time=1284.1..1284.2 rows=50)
   ->  Sort  (actual time=1284.1..1284.1 rows=50)
         ★ Sort Method: external merge  Disk: 84,112kB
         ->  Hash Join  (actual time=2.1..1201.4 rows=41)
               ->  ★ Seq Scan on orders  (rows=41)
                     ★ Rows Removed by Filter: 3,999,959
               ->  Hash  (rows=100000)
 Execution Time: 1284.8 ms
```
```
 ★ ATTRIBUTION:
   seq scan (3.99M rows filtered)   ~1,190 ms   ★ 93%
   hash join                            2.1 ms       0.2%
   sort                                ~83 ms        6%
 ⇒ ★ THE JOIN IS 0.2% OF THE COST.
```

**Gate 2 — the free fixes.**
```sql
CREATE INDEX CONCURRENTLY idx_orders_cust_created
  ON orders (customer_id, created_at DESC);
EXPLAIN (ANALYZE, BUFFERS)
SELECT o.id, o.total_minor, c.name
  FROM orders o JOIN customers c ON c.id = o.customer_id
 WHERE o.customer_id = 42 ORDER BY o.created_at DESC LIMIT 50;
```
```
 Limit  (actual time=0.048..0.112 rows=41)
   ->  Nested Loop  (actual time=0.046..0.104 rows=41)
         ->  ★ Index Scan using idx_orders_cust_created (rows=41)
         ->  Index Scan using customers_pkey (loops=41)
 Execution Time: ★ 0.148 ms
```
```
 ★ 1,284 ms → 0.148 ms.  8,676×.  ONE INDEX. ZERO OBLIGATION.
 ⇒ ★ GATE 2 FAILED FOR THE DENORMALISATION PROPOSAL — in the
   best possible way.
```

**Gate 3 — the snapshot test, both answers.**
```sql
-- (a) orders.customer_name for a "my orders" list
--     "if the customer renames, must the list show the new name?"
--     ⇒ YES ⇒ a COPY ⇒ obligation

-- (b) order_items.unit_price_charged
--     "if the product price changes, must this change?"
--     ⇒ ★ NO ⇒ a SNAPSHOT ⇒ no obligation, and it is 3NF-correct
CREATE TABLE order_items (
  id bigserial PRIMARY KEY,
  order_id bigint NOT NULL,
  product_id bigint NOT NULL,
  qty int NOT NULL,
  unit_price_charged_minor bigint NOT NULL,   -- ★ named as a snapshot
  line_total_minor bigint
    GENERATED ALWAYS AS (qty * unit_price_charged_minor) STORED
);
-- ★ no trigger, no reconciler, no obligation. The name says why.
```

**Gate 4 — the arithmetic, with real numbers.**
```sql
-- fan-out
SELECT avg(cnt)::numeric(10,1) AS avg_orders,
       max(cnt) AS max_orders,
       percentile_cont(0.99) WITHIN GROUP (ORDER BY cnt) AS p99
  FROM (SELECT customer_id, count(*) AS cnt FROM orders GROUP BY 1) x;
```
```
 avg_orders | max_orders |  p99
------------+------------+-------
       40.0 |     ★ 8402 | ★ 412
```
```sql
-- the write cost at that fan-out
\timing on
UPDATE orders SET customer_id = customer_id WHERE customer_id = 42;   -- 41 rows
-- Time: 8.2 ms
```
```
 ★ GATE 4 FOR orders.customer_name:
   benefit  0.01 ms × 410,000 reads/day  =  4 seconds/day
   cost     0.3 ms × 4,100 writes/day × 412 (p99 fan-out)
                                        =  ★ 8.5 minutes/day
   ⇒ RATIO 1 : 127.  ★ REJECTED.
```

**Gate 4 — contention.**
```sql
SELECT customer_id, count(*) AS orders_today
  FROM orders WHERE created_at > now() - interval '1 day'
 GROUP BY 1 ORDER BY 2 DESC LIMIT 3;
```
```
 customer_id | orders_today
-------------+--------------
       88412 |          412
       41209 |          188
   ★ 412/day = 0.005/sec on the hottest key. Contention fine.
     ⇒ contention is NOT the blocker here; fan-out is.
```

**Gate 5 — the four artefacts, for a proposal that did pass.**
```sql
-- ① mechanism (trigger — 3 services write reviews)
CREATE TRIGGER trg_review_stats AFTER INSERT OR UPDATE OR DELETE ON reviews
  FOR EACH ROW EXECUTE FUNCTION sync_product_review_stats();

-- ② ★ reconciler + self-repair
--    ops/reconcile_review_stats.sql, scheduled hourly
WITH actual AS (SELECT product_id, count(*) n, sum(stars) s FROM reviews GROUP BY 1)
SELECT count(*) AS drifted FROM products p LEFT JOIN actual a ON a.product_id=p.id
 WHERE p.review_count IS DISTINCT FROM coalesce(a.n,0)
    OR p.review_sum   IS DISTINCT FROM coalesce(a.s,0);

-- ③ constraints
ALTER TABLE products
  ADD CONSTRAINT ck_rc_nonneg CHECK (review_count >= 0),
  ADD CONSTRAINT ck_rs_valid  CHECK (review_sum BETWEEN 0 AND review_count*5);

-- ④ ★ owner and why
COMMENT ON COLUMN products.review_count IS
  'Denormalised aggregate of reviews. Maintained by trg_review_stats.
   Reconciler: ops/reconcile_review_stats.sql (hourly sampled, nightly full).
   Added 2026-08-18: the live aggregate was 2,100ms of a 2,400ms p99
   on GET /api/products (8.4M reads/day). Owner: catalogue-team.';
```

---

## Example 2 — production scenario

**The situation.** A logistics platform. A "shipment tracking" page shows shipment details plus the carrier's name, the origin and destination hub names, the current status, and a count of scan events. An engineer opens a PR:

> *"Denormalise carrier_name, origin_hub_name, destination_hub_name, current_status and scan_count onto `shipments`. The tracking page joins 5 tables and it's killing us."*

```
 THE PR: +5 columns, +1 trigger, +180 lines
 THE CLAIM: p99 1,880 ms → "should be under 50 ms"
```

**Gate 1 — is there a measured problem? Partially.**

```sql
SELECT calls, mean_exec_time::numeric(10,2) AS mean_ms,
       max_exec_time::numeric(10,2) AS max_ms,
       (calls*mean_exec_time/1000/3600)::numeric(10,2) AS hours,
       substring(query from 1 for 70) AS q
  FROM pg_stat_statements
 WHERE query ILIKE '%FROM shipments%' ORDER BY calls*mean_exec_time DESC LIMIT 3;
```
```
 calls   | mean_ms | max_ms  | hours  |                   q
---------+---------+---------+--------+------------------------------------
 8842119 |  ★ 1.88 | 1880.44 | ★ 4.62 | SELECT s.id, s.tracking_no, c.name…
```
```
 ★ THE FIRST SURPRISE: the MEAN is 1.88 ms. The 1,880 ms number in
   the PR is the MAX.
 ⇒ ★ THIS QUERY IS FINE 99.9% OF THE TIME. Something specific
   makes it occasionally terrible.
 ⇒ ★ GATE 1 IS ABOUT THE p99, NOT THE MAX. Get the distribution.
```

```sql
-- from the application's histogram
--   p50   1.2 ms
--   p95   3.4 ms
--   p99   ★ 412 ms
--   p999  ★ 1,880 ms
```

**Gate 1 continued — what makes the tail?**

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT s.id, s.tracking_no, s.status,
       c.name AS carrier, ho.name AS origin, hd.name AS destination,
       (SELECT count(*) FROM scan_events e WHERE e.shipment_id = s.id) AS scans
  FROM shipments s
  JOIN carriers c  ON c.id  = s.carrier_id
  JOIN hubs ho     ON ho.id = s.origin_hub_id
  JOIN hubs hd     ON hd.id = s.destination_hub_id
 WHERE s.tracking_no = 'TRK8842119004';
```
```
 Nested Loop  (actual time=0.112..0.184 rows=1)
   Buffers: shared hit=22
   ->  Index Scan using idx_shipments_tracking on shipments  (rows=1)
   ->  Index Scan using carriers_pkey  (rows=1)
   ->  Index Scan using hubs_pkey      (rows=1)
   ->  Index Scan using hubs_pkey      (rows=1)
   SubPlan 1
     ->  Aggregate  (actual time=0.041..0.041 rows=1)
           ->  ★ Index Only Scan using idx_scan_events_shipment
                 (actual rows=14)  Heap Fetches: 0
 Execution Time: ★ 0.221 ms
```
```
 ★ 0.221 ms. THREE JOINS AND A SUBQUERY AGGREGATE, AND IT IS
   FASTER THAN A NETWORK ROUND TRIP.
 ⇒ ★ THE FIVE-TABLE JOIN IS NOT THE PROBLEM. So what is the tail?
```

**Step 2 — find the actual tail.**

```sql
-- log_min_duration_statement = 200ms was already on. From the logs:
```
```
2026-08-18 14:22:41 LOG:  duration: 1880.442 ms  execute <unnamed>:
  SELECT s.id, s.tracking_no, … FROM shipments s
   JOIN carriers c ON … JOIN hubs ho ON … JOIN hubs hd ON …
   WHERE s.carrier_id = $1 AND s.status = 'in_transit'
   ORDER BY s.created_at DESC LIMIT 50
```
```
 ★ IT IS A DIFFERENT QUERY.
   The PR benchmarked the single-shipment lookup (0.2 ms) and
   attributed the p999 of the CARRIER DASHBOARD to it.
   ⇒ ★ TWO ENDPOINTS, ONE pg_stat_statements ENTRY, because they
     normalise to the same shape after parameterisation? No —
     ★ they are genuinely different queries and the engineer
     conflated them.
```

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT s.id, s.tracking_no, s.status, c.name, ho.name, hd.name
  FROM shipments s
  JOIN carriers c ON c.id = s.carrier_id
  JOIN hubs ho ON ho.id = s.origin_hub_id
  JOIN hubs hd ON hd.id = s.destination_hub_id
 WHERE s.carrier_id = 12 AND s.status = 'in_transit'
 ORDER BY s.created_at DESC LIMIT 50;
```
```
 Limit  (actual time=1878.2..1878.4 rows=50)
   ->  Sort  (actual time=1878.2..1878.3 rows=50)
         Sort Key: s.created_at DESC
         ★ Sort Method: external merge  Disk: 214,880kB
         ->  Hash Join  (actual time=41.2..1604.8 rows=1,204,882)
               ->  ★ Bitmap Heap Scan on shipments  (rows=1,204,882)
                     Recheck Cond: (carrier_id = 12)
                     ★ Filter: (status = 'in_transit')
                     ★ Rows Removed by Filter: 8,204,118
 Execution Time: ★ 1,878.9 ms
```
```
 ★ ATTRIBUTION:
   bitmap scan + filter (9.4M rows)   ~1,560 ms   ★ 83%
   the three joins                       ~45 ms        2%
   the external sort                    ~274 ms       15%
 ⇒ ★ THE JOINS ARE 2%. THE PR PROPOSED TO FIX THE 2%.
```

**Gate 2 — one index.**

```sql
CREATE INDEX CONCURRENTLY idx_shipments_carrier_status_created
  ON shipments (carrier_id, created_at DESC)
  WHERE status = 'in_transit';
-- ★ partial: 'in_transit' is 12.8% of rows, and the dashboard
--   only ever queries that status.
```
```sql
EXPLAIN (ANALYZE, BUFFERS) /* same query */;
```
```
 Limit  (actual time=0.084..1.212 rows=50)
   ->  Nested Loop  (actual time=0.082..1.184 rows=50)
         ->  ★ Index Scan using idx_shipments_carrier_status_created (rows=50)
         ->  Index Scan using carriers_pkey (loops=50)
         ->  Index Scan using hubs_pkey     (loops=50)
         ->  Index Scan using hubs_pkey     (loops=50)
 Execution Time: ★ 1.284 ms
```
```
 ★ 1,878 ms → 1.28 ms.  1,467×.
 ★ AND THE THREE JOINS NOW RUN 50 TIMES INSTEAD OF 1,204,882.
   They cost 0.9 ms total.
 ⇒ ★ GATE 2 FAILED. There is nothing left worth denormalising.
```

**Step 3 — but one of the five columns *does* pass.**

```
 ★ THE SCAN COUNT IS DIFFERENT. Re-examine it separately —
   never evaluate five columns as one proposal.

 A NEW REQUIREMENT arrived during the review: an operations
 dashboard listing 500 shipments with their scan counts, refreshed
 every 10 seconds by 40 concurrent operators.
```

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT s.id, s.tracking_no,
       (SELECT count(*) FROM scan_events e WHERE e.shipment_id = s.id) AS scans
  FROM shipments s
 WHERE s.carrier_id = 12 AND s.status = 'in_transit'
 ORDER BY s.created_at DESC LIMIT 500;
```
```
 Limit  (actual time=0.104..842.2 rows=500)
   ->  Index Scan using idx_shipments_carrier_status_created (rows=500)
         SubPlan 1
           ->  Aggregate  (loops=500)
                 ->  ★ Index Only Scan using idx_scan_events_shipment
                       (loops=500, ★ actual rows=1,204 per loop)
 Execution Time: ★ 842.8 ms
```
```
 ★ 500 shipments × 1,204 scan events each = 602,000 index entries
   read, to produce 500 integers.
 ★ 40 operators × every 10 s = 4 requests/sec × 842 ms
   = ★ 3.4 CPU-seconds per second. It saturates 3.4 cores by itself.
```

**Gates 3–5 for `scan_count` alone.**

```
 ★ GATE 3 — copy or snapshot?
   "if a new scan event arrives, must scan_count change?"
   ⇒ YES. A real copy. Obligation accepted.

 ★ GATE 4 — the arithmetic
   BENEFIT  842 ms × 345,600 reads/day (4/sec × 86,400)
            = ★ 80.8 hours/day of database time
   COST     0.4 ms × 41,000,000 scan events/day × fan-out 1
            = ★ 4.6 hours/day
   RATIO    17.6 : 1
   ⇒ ★ POSITIVE, BUT NOT OVERWHELMING. Look harder.

 ★ CONTENTION — this is where it gets interesting
   SELECT shipment_id, count(*) FROM scan_events
    WHERE created_at > now() - interval '1 hour'
    GROUP BY 1 ORDER BY 2 DESC LIMIT 3;
   ⇒ shipment 8842119: 14 scans/hour = 0.004/sec   ✓ fine
   ⇒ ★ scan events are spread across 9.4M shipments. No hot row.

 ★ BUT: 41,000,000 scan events/day = 475/sec, each now doing an
   extra UPDATE on shipments.
   ⇒ ★ 475 extra row updates/sec on a 9.4M-row table
   ⇒ ★ AND shipments has 7 INDEXES ⇒ non-HOT updates ⇒ 8 pages
     dirtied per scan event (Topic 46)
   ⇒ MEASURED: WAL 4 GB/hr → ★ 31 GB/hr
   ⇒ ★ THE ARITHMETIC MISSED THIS. Gate 4's "ms added" understated
     the true cost because it ignored WRITE AMPLIFICATION FROM
     INDEXES.
```

**Step 4 — the better answer, found by taking gate 4 seriously.**

```
 ★ THREE OPTIONS RECONSIDERED:

 ① a trigger-maintained counter on `shipments`
    ⇒ 31 GB/hr WAL, 8× write amplification. ★ Rejected.

 ② ★ a SEPARATE narrow table: shipment_scan_counts(shipment_id, n)
    ⇒ ONE index (the PK), so HOT updates apply
    ⇒ fillfactor 70 for headroom (Topic 46)
    ⇒ MEASURED: WAL 4 GB/hr → ★ 5.2 GB/hr
    ⇒ ★ the read becomes a join to a tiny, cache-resident table
    ⇒ ★ THIS IS THE ANSWER, and it only appears if you understand
      why option ① was expensive.

 ③ a materialised view refreshed every 30 s
    ⇒ ★ the ops dashboard tolerates 30 s staleness — it refreshes
      every 10 s anyway and scans arrive continuously
    ⇒ zero write-path cost
    ⇒ ★ but REFRESH CONCURRENTLY over 9.4M shipments takes 44 s
      ⇒ can't keep up. Rejected for this volume. (Topic 56.)
```

```sql
-- ★ OPTION ② — the narrow table
CREATE TABLE shipment_scan_counts (
  shipment_id bigint PRIMARY KEY REFERENCES shipments(id) ON DELETE CASCADE,
  n           integer NOT NULL DEFAULT 0 CHECK (n >= 0),
  updated_at  timestamptz NOT NULL DEFAULT now()
) WITH (fillfactor = 70);          -- ★ headroom for HOT updates

ALTER TABLE shipment_scan_counts SET (
  autovacuum_vacuum_scale_factor = 0.01,
  autovacuum_vacuum_cost_delay   = 0);

CREATE OR REPLACE FUNCTION bump_scan_count() RETURNS trigger AS $$
BEGIN
  IF TG_OP = 'INSERT' THEN
    INSERT INTO shipment_scan_counts (shipment_id, n)
    VALUES (NEW.shipment_id, 1)
    ON CONFLICT (shipment_id) DO UPDATE
      SET n = shipment_scan_counts.n + 1, updated_at = now();
  ELSIF TG_OP = 'DELETE' THEN
    UPDATE shipment_scan_counts SET n = n - 1, updated_at = now()
     WHERE shipment_id = OLD.shipment_id;
  END IF;
  RETURN NULL;
END $$ LANGUAGE plpgsql;

CREATE TRIGGER trg_scan_count
  AFTER INSERT OR DELETE ON scan_events
  FOR EACH ROW EXECUTE FUNCTION bump_scan_count();
```

```sql
-- the read
SELECT s.id, s.tracking_no, coalesce(sc.n, 0) AS scans
  FROM shipments s
  LEFT JOIN shipment_scan_counts sc ON sc.shipment_id = s.id
 WHERE s.carrier_id = 12 AND s.status = 'in_transit'
 ORDER BY s.created_at DESC LIMIT 500;
```
```
 Limit  (actual time=0.112..2.884 rows=500)
   ->  Nested Loop Left Join  (actual time=0.110..2.841 rows=500)
         ->  Index Scan using idx_shipments_carrier_status_created (rows=500)
         ->  Index Scan using shipment_scan_counts_pkey (loops=500)
 Execution Time: ★ 2.941 ms
```

**Gate 5 — the four artefacts.**

```sql
-- ★ reconciler, sampled hourly
SELECT count(*) AS drifted FROM (
  SELECT sc.shipment_id, sc.n,
         (SELECT count(*) FROM scan_events e WHERE e.shipment_id = sc.shipment_id) AS actual
    FROM shipment_scan_counts TABLESAMPLE SYSTEM (0.5) sc) x
 WHERE n IS DISTINCT FROM actual;

-- ★ full nightly + self-repair
UPDATE shipment_scan_counts sc SET n = a.cnt, updated_at = now()
  FROM (SELECT shipment_id, count(*) AS cnt FROM scan_events GROUP BY 1) a
 WHERE a.shipment_id = sc.shipment_id AND sc.n IS DISTINCT FROM a.cnt;

COMMENT ON TABLE shipment_scan_counts IS
  'Denormalised count of scan_events per shipment. Maintained by
   trg_scan_count on scan_events. Reconciler: ops/reconcile_scan_counts.sql
   (hourly 0.5% sample, nightly full + repair).
   Kept in a SEPARATE narrow table rather than a shipments column so that
   updates stay HOT — shipments has 7 indexes and a column there cost
   31 GB/hr of WAL vs 5.2 GB/hr here. Added 2026-08-18 for the ops
   dashboard (842ms → 2.9ms). Owner: logistics-platform.';
```

**Step 5 — results.**

| | PR as proposed | Shipped |
|---|---|---|
| Columns denormalised | 5 | ★ **1** (in its own table) |
| Reconcilers owed | 5 | **1** |
| Tracking page p999 | *(unchanged — wrong query)* | 1.3 ms (**1,467×**, from an index) |
| Ops dashboard | 842 ms | 2.9 ms (**290×**) |
| WAL | 4 GB/hr → *(31 GB/hr if done as proposed)* | **5.2 GB/hr** |
| Indexes added | 0 | 1 |

```
 ★ FIVE LESSONS:
 ① ★ THE PR BENCHMARKED THE WRONG QUERY. mean 1.88 ms was
   conflated with max 1,880 ms, and the tail belonged to a
   different endpoint entirely.
 ② ★ 83% OF THE REAL COST WAS A MISSING INDEX. The joins were 2%.
 ③ ★ NEVER EVALUATE FIVE COLUMNS AS ONE PROPOSAL. Four failed
   gate 2; one passed all five.
 ④ ★ GATE 4'S ARITHMETIC UNDERSTATED THE COST because it ignored
   index write amplification. 7 indexes on `shipments` meant
   8 pages dirtied per update.
 ⑤ ★ UNDERSTANDING *WHY* IT WAS EXPENSIVE PRODUCED A BETTER
   DESIGN: a narrow side table where HOT updates apply, giving
   6× less WAL than the obvious approach.
```

---

## Common mistakes

**1. Benchmarking the wrong query.**
- *Symptom:* a PR quoting a `max_exec_time` as if it were the p99, for an endpoint that isn't the one that's slow.
- *Fix:* get the p50/p95/p99/p999 distribution and `EXPLAIN` the *specific* slow query from the logs.

**2. Not attributing cost across the plan.**
- *Symptom:* "the 5-table join is slow" when the joins are 2% and a seq scan is 83%.
- *Fix:* read `actual time` at each plan node. Assign a percentage to each.

**3. Evaluating several columns as one proposal.**
- *Symptom:* five obligations shipped because one of them was justified.
- *Fix:* run each column through all five gates independently.

**4. Skipping gate 2.**
- *Symptom:* an obligation taken on for something one index would have fixed 1,000×.
- *Fix:* try all five free fixes and re-measure after each.

**5. Ignoring fan-out in gate 4.**
- *Symptom:* the numbers look survivable until you notice one write touches 8,402 rows.
- *Fix:* compute avg, p99 and max rows-per-source-change. Use p99, not avg.

**6. Ignoring index write amplification.**
- *Symptom:* a trigger-maintained column on a 7-index table generates 8× the expected WAL.
- *Fix:* count the indexes on the target table. Consider a narrow side table where HOT updates apply (Topic 46).

**7. Checking average contention instead of maximum.**
- *Symptom:* a counter that works in staging and serialises in production on one popular key.
- *Fix:* `GROUP BY key ORDER BY count DESC LIMIT 5` — look at the top.

**8. Treating a snapshot as a copy (or vice versa).**
- *Symptom:* a trigger and a reconciler for a column that should be frozen — or drift in a column everyone assumed was history.
- *Fix:* the gate 3 question, and *name the column* to encode the answer.

**9. Shipping without an owner.**
- *Symptom:* the reconciler alert fires 14 months later and nobody knows what the column is for.
- *Fix:* `COMMENT ON COLUMN` with the why, the mechanism, the reconciler path, and the team.

**10. "We can always normalise it back later."**
- *Symptom:* a multi-quarter migration to remove a column four services read and two write.
- *Fix:* treat the decision as one-way. The gates exist because reversal is expensive.

---

## Hands-on proof

**PROVE IT #1–#5 — Example 1** (`pg_stat_statements` establishing a measured problem, cost attribution showing the join at 0.2%, an index beating denormalisation 8,676×, the snapshot test both ways, fan-out measured at p99 412 / max 8,402).

**PROVE IT #6 — index write amplification.**
```sql
CREATE TABLE wide (id bigint PRIMARY KEY, n bigint DEFAULT 0,
                   a int, b int, c int, d int, e int, f int);
INSERT INTO wide SELECT g,0,g,g,g,g,g,g FROM generate_series(1,500000) g;
CREATE INDEX ON wide(a); CREATE INDEX ON wide(b); CREATE INDEX ON wide(c);
CREATE INDEX ON wide(d); CREATE INDEX ON wide(e); CREATE INDEX ON wide(f);
VACUUM ANALYZE wide;

SELECT pg_current_wal_lsn() AS x \gset
UPDATE wide SET n = n + 1;
SELECT pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), :'x')) AS wal_7_indexes;

CREATE TABLE narrow (id bigint PRIMARY KEY, n bigint DEFAULT 0) WITH (fillfactor=70);
INSERT INTO narrow SELECT g,0 FROM generate_series(1,500000) g;
VACUUM ANALYZE narrow;
SELECT pg_current_wal_lsn() AS y \gset
UPDATE narrow SET n = n + 1;
SELECT pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), :'y')) AS wal_narrow;
```
```
 wal_7_indexes
---------------
 ★ 412 MB
 wal_narrow
------------
 ★ 68 MB          — 6.1× less. ★ This is why the side table won.
```
```sql
SELECT relname, n_tup_upd, n_tup_hot_upd,
       round(100.0*n_tup_hot_upd/n_tup_upd,1) AS hot_pct
  FROM pg_stat_user_tables WHERE relname IN ('wide','narrow');
```
```
 relname | n_tup_upd | n_tup_hot_upd | hot_pct
---------+-----------+---------------+---------
 wide    |    500000 |          2104 |   ★ 0.4
 narrow  |    500000 |        487204 |  ★ 97.4
```

**PROVE IT #7 — contention on the hottest key.**
```sql
SELECT product_id, count(*) AS writes_today,
       round(count(*)/86400.0, 4) AS per_sec
  FROM reviews WHERE created_at > now() - interval '1 day'
 GROUP BY 1 ORDER BY 2 DESC LIMIT 5;
-- ★ look at row 1, not at avg().
```

**PROVE IT #8 — the ratio, computed from real statistics.**
```sql
WITH r AS (SELECT sum(calls) AS reads FROM pg_stat_statements
            WHERE query ILIKE '%FROM products%JOIN%reviews%'),
     w AS (SELECT sum(calls) AS writes FROM pg_stat_statements
            WHERE query ILIKE 'INSERT INTO reviews%')
SELECT r.reads, w.writes, (r.reads::numeric / nullif(w.writes,0))::numeric(10,1) AS ratio
  FROM r, w;
```
```
  reads   | writes |  ratio
----------+--------+---------
  8402118 |  41204 | ★ 203.9
```

---

## The design decision framework

```
★★★ FIVE GATES. ALL MUST PASS. MOST PROPOSALS DIE AT GATE 2. ★★★

 ═══ GATE 1 — A MEASURED PROBLEM ═══════════════════════════
 □ named query from pg_stat_statements, with calls & total time
 □ ★ the p99 DISTRIBUTION — not the max, not "it feels slow"
 □ a stated SLO it misses
 □ EXPLAIN (ANALYZE, BUFFERS) at production volume
 □ ★ COST ATTRIBUTED — what % of the time is the join, really?
 ⇒ ★ if the join is under 20% of the cost, stop here.

 ═══ GATE 2 — FREE FIXES EXHAUSTED ═════════════════════════
 □ composite index on (filter, sort)          ★ often 1,000×+
 □ partial index if a selective predicate exists
 □ covering index → index-only scan           ★ 22×
 □ ★ N+1 eliminated — one query, not N        ★ 38×
 □ work_mem raised for external sorts         3.8×
 □ query rewritten (lateral / EXISTS / CTE inlining)
 □ materialised view considered for aggregates ★ 7,000×
 ⇒ ★ RE-MEASURE AFTER EACH. And note: an index that cuts the row
   count from 1.2M to 50 often makes the join FREE, leaving
   nothing to denormalise.

 ═══ GATE 3 — COPY OR SNAPSHOT ═════════════════════════════
 □ "if the source changes, must this change too?"
   NO  ⇒ ★ SNAPSHOT. Ship it. Zero obligation.
          ★ NAME IT: unit_price_charged, address_at_shipment
   YES ⇒ continue.

 ═══ GATE 4 — THE ARITHMETIC ═══════════════════════════════
 □ reads/day × ms saved                       = benefit
 □ writes/day × ms added × ★ FAN-OUT (p99)    = cost
 □ ★ INDEX COUNT on the target table (write amplification)
 □ ★ CONTENTION: writes/sec on the HOTTEST key, not the average
 ⇒ ratio < 100                ⇒ ★ no
 ⇒ hottest key > 800/sec      ⇒ ★ no — shard it (Topic 61)
 ⇒ target has many indexes    ⇒ ★ consider a NARROW SIDE TABLE
                                so HOT updates apply (Topic 46)

 ═══ GATE 5 — THE OBLIGATION DISCHARGEABLE ═════════════════
 □ mechanism: ★ generated > trigger > same-txn > async
 □ ★ >1 writing service ⇒ trigger or generated ONLY
 □ ★ reconciliation query — written, scheduled, alerting
 □ ★ self-repair statement
 □ CHECK constraints bounding the value
 □ ★ COMMENT with why / mechanism / reconciler path / owner
 ⇒ ★ ANY BOX UNCHECKED ⇒ NO, regardless of the benchmark.

 ═══ THE ALWAYS-YES SHORTCUTS ══════════════════════════════
 ★ the join crosses a SERVICE BOUNDARY   (the strongest case)
 ★ it's a SNAPSHOT                       (no obligation at all)
 ★ an aggregate over a large child table read constantly
 ★ an analytical/warehouse store         (different rules)

 ═══ THE ALWAYS-NO SHORTCUTS ═══════════════════════════════
 ✗ unmeasured "to avoid a join"
 ✗ ★ a copied column with large fan-out
 ✗ ★ to fix an N+1
 ✗ ★ speculative "for scale"
 ✗ a pure function of the same row      ⇒ ★ GENERATED
 ✗ ★ nobody will own the reconciler
 ✗ ★ on a row that is already hot

 ═══ AND REMEMBER ══════════════════════════════════════════
 ★ ADDING IT: one PR.
 ★ REMOVING IT after 4 services read it: a multi-quarter migration.
 ⇒ TREAT IT AS ONE-WAY.
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
Take a slow query on a 4M-row table. Walk gates 1 and 2 explicitly: capture `pg_stat_statements`, run `EXPLAIN (ANALYZE, BUFFERS)`, attribute the cost as percentages across the plan nodes, then add the right index and re-measure. Report what percentage of the original cost the join actually was.

### Exercise 2 — medium (apply it)
For each proposal, walk all five gates and produce a written yes/no with numbers:
(a) `orders.customer_email` (fan-out: avg 40, p99 412) · (b) `order_items.unit_price_charged` · (c) `products.review_count` (8.4M reads/day, 41k writes/day) · (d) `users.follower_count` where the top account gains 4,000 followers/sec · (e) `invoices.line_item_count`

For (d), explain which gate fails and what you'd do instead.

### Exercise 3 — hard (production simulation)
A PR proposes denormalising five columns (`carrier_name`, `origin_hub_name`, `destination_hub_name`, `current_status`, `scan_count`) onto a `shipments` table, citing a p999 of 1,880 ms on the tracking page.

(a) `pg_stat_statements` shows `mean_exec_time` 1.88 ms and `max_exec_time` 1,880 ms for that query. What is the first thing this tells you, and what do you do next?
(b) The single-shipment lookup — three joins and a subquery aggregate — runs in 0.221 ms. What does that prove, and what doesn't it prove?
(c) The real tail belongs to a different query. Find it from the slow-query log and attribute its 1,878 ms across the plan nodes as percentages.
(d) Write the index. Explain why it should be partial, and predict the new plan.
(e) After the index, the three joins run 50 times instead of 1,204,882. What does that do to gates 1 and 4 for those four columns?
(f) `scan_count` is different. Walk gates 3, 4 and 5 for it alone, with the numbers.
(g) Gate 4's simple arithmetic gives 17.6:1 — positive but not overwhelming. Name the cost the arithmetic *missed* and quantify it.
(h) Given `shipments` has 7 indexes, explain why a column there costs 31 GB/hr of WAL and a narrow side table costs 5.2 GB/hr. Reference HOT updates.
(i) A materialised view was considered and rejected. Give the specific number that ruled it out.
(j) Write the complete gate-5 package: trigger, reconciler (sampled + full + repair), constraints, and the `COMMENT`.
(k) The PR proposed 5 columns; 1 shipped. Write the review comment that explains why, in a way the author will find useful rather than dismissive.

---

## Mental model checkpoint

1. Name the five gates in order. Which one rejects the most proposals, and why?
2. Why is "the mean is 1.88 ms and the max is 1,880 ms" a red flag rather than evidence?
3. What does "attribute the cost" mean, and why does it usually kill the join argument?
4. List the five free fixes and roughly what each is worth.
5. Give the gate 3 question and an example of each answer from an order/invoice schema.
6. Write the gate 4 formula. Which term do people most often omit?
7. Why is average contention the wrong metric? What is the right one?
8. How does index count on the target table change the cost calculation? What design does that suggest?
9. Name the four artefacts gate 5 requires. Which one is most often missing?
10. Name the four always-yes cases and the seven always-no cases.
11. Why should you treat the decision as one-way?

---

## Quick reference card

**The five gates**
```
① MEASURED PROBLEM   named query · p99 distribution · SLO ·
                     EXPLAIN · ★ cost attributed
② ★ FREE FIXES       index · partial · covering · N+1 · work_mem ·
                     rewrite · matview      ← ★ most die here
③ COPY OR SNAPSHOT   "if the source changes, must this?"
                     NO ⇒ ★ snapshot, ship it, no obligation
④ ARITHMETIC         ms×reads  vs  ms×writes×★FAN-OUT
                     + ★ index count + ★ hottest-key contention
⑤ ★ OBLIGATION       mechanism + ★ reconciler + constraint + owner
```

**Thresholds**

| Signal | Verdict |
|---|---|
| join < 20% of plan cost | ★ stop at gate 1 |
| benefit:cost < 100 | no |
| benefit:cost > 10,000 | yes |
| p99 fan-out > ~100 rows | ★ probably no |
| hottest key > 800 writes/s | ★ hot row — shard (Topic 61) |
| target table has many indexes | ★ use a narrow side table |
| >1 writing service | ★ trigger or generated **only** |

**Always yes:** crosses a service boundary · it's a snapshot · a big aggregate read constantly · a warehouse.
**Always no:** unmeasured · large fan-out · an N+1 · speculative · a same-row function (use `GENERATED`) · no reconciler owner · an already-hot row.

**The queries**
```sql
-- gate 1
SELECT calls, mean_exec_time, max_exec_time, substring(query,1,60)
  FROM pg_stat_statements ORDER BY calls*mean_exec_time DESC LIMIT 10;
-- gate 4: fan-out
SELECT avg(c), max(c), percentile_cont(0.99) WITHIN GROUP (ORDER BY c)
  FROM (SELECT parent_id, count(*) c FROM child GROUP BY 1) x;
-- gate 4: ★ contention — the MAX, not the average
SELECT key, count(*) FROM child WHERE created_at > now()-interval '1 day'
 GROUP BY 1 ORDER BY 2 DESC LIMIT 5;
```

**★ Adding it is one PR. Removing it is a multi-quarter migration. Treat it as one-way.**

---

## When would I use this at work?

1. **Reviewing any PR that adds a denormalised column.** The five gates turn a debate about taste into a checklist with numbers. Most PRs stop at gate 2 with a one-line index instead — and the author usually agrees once they see the cost attribution.

2. **When a team says "we need to denormalise for scale."** Ask for the p99 distribution and the cost attribution. In my experience the join is rarely more than a few percent, and the real cost is a missing index or an N+1.

3. **Designing a schema for a new service.** Gate 3 alone saves enormous confusion: recognising that `unit_price_charged` and `address_at_shipment` are snapshots — not denormalisation — means no triggers, no reconcilers, and no future engineer "fixing" them.

4. **Deciding *where* to put a maintained counter.** Gate 4's index-amplification term is what leads you to a narrow side table instead of a column on a heavily-indexed parent — a 6× difference in WAL that never shows up in a naive benchmark.

---

## Connected topics

**Understand before this:** 53 (what denormalisation is, and the obligation), 10–19 (indexes — gate 2's main weapon), 46 (MVCC and HOT updates — why index count changes the cost), 38 (snapshot vs copy — gate 3), 67 (performance investigation — gate 1's method).

**This unlocks:**
- **55** — the concrete patterns, once you've decided yes
- **56** — materialised views: gate 2's last free option
- **57** — caching: the same decision, outside the database
- **61** — counters and hot rows: what to do when gate 4's contention check fails
- **66** — N+1: the most common wrong diagnosis
