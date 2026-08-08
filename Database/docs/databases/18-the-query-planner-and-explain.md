# 18 — The Query Planner and EXPLAIN
## Phase: Indexes

---

## ELI5 — The Simple Analogy

You want to get across Bengaluru at 6pm.

Google Maps doesn't drive you. It **estimates**: this route is 14 km of clear road, that one is 9 km but through Silk Board. It assigns each a cost in minutes using rules of thumb — average speed on a highway, average speed in traffic — and picks the cheapest.

It is often right and sometimes badly wrong. And when it's wrong, it's almost never because the *rules* are bad. It's because the **input** was bad: it thought the road was clear and there was a protest.

Three things follow, and they're the whole topic:

- The planner **doesn't know** how long anything takes. It computes an arbitrary cost number and compares.
- The cost model has **tunable constants** — "how much slower is a random read than a sequential one?" — and the defaults are from the 1990s.
- Almost every bad plan is a bad **estimate**, not a bad rule. Fix the input, not the algorithm.

`EXPLAIN` shows you the route it picked. `EXPLAIN ANALYZE` shows you what actually happened. **The gap between them is the diagnosis.**

---

## Where this fits in the big picture

```
   10–17 — indexes, and the statistics that describe them
                          │
                          ▼
        ┌──────────────────────────────────────────┐
        │ 18 THE PLANNER AND EXPLAIN               │ ← YOU ARE HERE
        │ how the choice is actually made          │
        └────────────────────┬─────────────────────┘
                             │
              ┌──────────────┴───────────────┐
              ▼                              ▼
        19 join algorithms            67 performance
        (the options it              investigation
         chooses between)            (this, as a method)
```

Topic 15 gave the planner its **inputs**. Topic 19 gives it its **options**. **This topic is the decision procedure in between** — and the skill of reading its output.

---

## What is this?

The **planner** (optimiser) takes a query tree and produces an execution plan. It:

1. Enumerates the possible ways to execute the query — scan methods, join methods, join orders.
2. Estimates the **cost** of each, in arbitrary units, using statistics (Topic 15) and cost constants.
3. Picks the cheapest total-cost path.

`EXPLAIN` prints the chosen plan and its estimates. `EXPLAIN (ANALYZE, BUFFERS)` runs it and prints reality alongside.

The planner is **cost-based**, not rule-based. It has no notion of "always use an index." It compares numbers.

---

## Why does it matter for a backend developer?

Because `EXPLAIN` output is the primary diagnostic instrument for the database, and most people read only the top line.

Three things you will do constantly:

```
 1. Find the ONE node that's wrong.
    A 40-line plan has one problem. Everything else is downstream of it.
    Reading top-down for "the big number" is the wrong method and will
    send you to the wrong fix.

 2. Distinguish "bad plan" from "bad estimate".
    Bad plan       → fix the index, the query, or a cost constant
    Bad estimate   → fix statistics. NO index will help.
    These need completely different work, and the symptom looks identical.

 3. Know which cost constants are lying to you.
    random_page_cost defaults to 4.0 — a 1990s spinning-disk assumption.
    On NVMe the true ratio is ~1.1. Leaving it at 4 is why PostgreSQL
    "refuses to use my index" in a huge number of reported cases.
```

And one broader point: the planner is *usually right*. When it disagrees with you, the default assumption should be that it knows something you don't — most often that your index would cost more than the scan.

---

## The physical reality

### The cost model — the actual constants

```
 postgresql.conf                    default     what it means
 ────────────────────────────────────────────────────────────────────────
 seq_page_cost               1.0    reading one page sequentially
 random_page_cost            4.0    reading one page randomly    ⚠
 cpu_tuple_cost              0.01   processing one row
 cpu_index_tuple_cost        0.005  processing one index entry
 cpu_operator_cost           0.0025 evaluating one operator/function
 parallel_tuple_cost         0.1    passing a row between workers
 parallel_setup_cost         1000   starting parallel workers

 THE UNITS ARE ARBITRARY. `1.0` means "one sequential page read".
 Everything is relative to that. A cost of 4,182 is not milliseconds,
 not I/Os — it is a comparable number and nothing more.

 ⚠ random_page_cost = 4.0 encodes "a random read costs 4× a sequential
   read." True for a 7,200 RPM disk in 1998. On NVMe the ratio is
   ~1.05–1.2. On EBS gp3, ~1.1–1.5.
   ⇒ Leaving it at 4.0 makes every index scan look 4× more expensive
     than it is, biasing the planner toward sequential scans.
```

### How a scan is costed

```
 SEQ SCAN on orders (98,000 pages, 8,000,000 rows)
   I/O:  98,000 × seq_page_cost(1.0)             =  98,000
   CPU:  8,000,000 × cpu_tuple_cost(0.01)        =  80,000
   filter: 8,000,000 × cpu_operator_cost(0.0025) =  20,000
   ─────────────────────────────────────────────────────────
   TOTAL COST = 198,000       (startup 0.00)

 INDEX SCAN on idx_orders_user_id (est. 39 rows)
   descend: ~3 pages × random_page_cost(4.0)     =      12
   index tuples: 39 × cpu_index_tuple_cost       =       0.2
   heap fetches: THIS IS THE INTERESTING PART.
     PostgreSQL interpolates between two extremes using
     pg_stats.correlation (Topic 15):
       correlation ±1 → the 39 rows are on ~1 page  → near-sequential
       correlation  0 → the 39 rows are on 39 pages → fully random
     with correlation = 0.002:
       39 pages × random_page_cost(4.0)            =     156
   ─────────────────────────────────────────────────────────
   TOTAL COST ≈ 168.4         (startup 0.43)

 ⇒ 198,000 vs 168. The planner picks the index by a factor of 1,175.

 ★ AND NOTE: correlation feeds directly into the cost. An index on a
   well-correlated column is genuinely cheaper to scan, and the planner
   knows it. This is the mechanism behind `CLUSTER` helping.
```

### Startup cost vs total cost — the field people ignore

```
 cost=0.43..1284.10
      ^^^^  ^^^^^^^
      │     └── TOTAL: cost to return ALL rows
      └──────── STARTUP: cost before the FIRST row can be returned

 WHY IT MATTERS — LIMIT changes which plan wins:

   Sort:        cost=48200.00..48250.00
                      ^^^^^^^^ must consume and sort EVERYTHING
                               before emitting row 1. BLOCKING.
   Index Scan:  cost=0.43..38210.00
                     ^^^^ first row is nearly free. STREAMING.

   Without LIMIT: Sort's total (48,250) beats Index Scan's (38,210)? No —
                  index wins. But for a big sort with a cheap seq scan
                  underneath, Sort often wins on TOTAL.
   With LIMIT 20: the planner scales cost by the fraction of rows needed.
                  Index Scan → 0.43 + (20/8000000) × 38210 ≈ 0.53
                  Sort       → 48,200 + almost nothing ≈ 48,200
                  ⇒ Index scan wins by 90,000×.

 ★ THIS IS WHY `LIMIT` CAN COMPLETELY CHANGE A PLAN, and why an index
   matching your ORDER BY is so valuable (Topics 09, 14).
```

### What `EXPLAIN` actually prints

```
 EXPLAIN (ANALYZE, BUFFERS, VERBOSE, SETTINGS)
 SELECT ... ;

 Nested Loop  (cost=1.28..842.10 rows=104 width=64)
              (actual time=0.041..8412.9 rows=184220 loops=1)
              └───┬───┘  └──┬───┘ └───┬────┘ └───┬───┘  └──┬──┘
       estimated startup   estimated  estimated  ACTUAL   how many
       and total cost      row count  row width  rows     TIMES this
                                                          node ran
   Buffers: shared hit=8420 read=136891 dirtied=12 written=4
            └───┬───┘ └───┬──┘  └────┬────┘
            from cache  from disk  pages this node modified
   ->  Index Scan using idx_a on orders o
         Index Cond:  (user_id = 4471)      ← narrowed the scan   ✓
         Filter:      (status = 'paid')     ← applied after fetch ✗
         Rows Removed by Filter: 184100     ← the size of the waste
   Planning Time: 0.181 ms                  ← phases 1–4 (Topic 09)
   Execution Time: 8413.4 ms                ← phase 5
```

**`loops` is the field that catches everyone.** In a nested loop, the inner node's `actual rows` and `actual time` are **per loop**, already averaged. Total rows produced by that node = `rows × loops`. A node showing `rows=1 loops=184220` did 184,220 units of work, not 1.

---

## How it works — step by step

### The planning pipeline

```
 1. GENERATE BASE RELATION PATHS
    For each table, enumerate every access path:
      orders:  Seq Scan · Index Scan (idx_a) · Bitmap Scan (idx_a)
               · Index Only Scan (idx_cover) · Parallel Seq Scan
      users:   Seq Scan · Index Scan (users_pkey)
    Cost each. Keep the cheapest per "useful sort order" — a more
    expensive path that arrives already sorted may win later.

 2. GENERATE JOIN PATHS  (dynamic programming over subsets)
    level 2:  {orders,users} via Nested Loop / Hash Join / Merge Join,
              in both directions (orders drives, or users drives)
    level 3:  {orders,users,products} = {orders,users}⋈products, etc.
    ...
    ⚠ The search space is factorial in table count. Guard rails:
       geqo_threshold (12): above 12 relations, switch from exhaustive
       dynamic programming to a GENETIC ALGORITHM (GEQO), which finds a
       good-but-not-guaranteed-optimal plan quickly.
       ⇒ This is why 15-table queries sometimes get erratic plans.

 3. ADD UPPER NODES
    Aggregation, sorting, LIMIT, window functions, DISTINCT.

 4. PICK THE CHEAPEST TOTAL PATH
    …unless there's a LIMIT, in which case cost is interpolated
    between startup and total by the row fraction.

 5. CONVERT the winning path into an executable plan tree.
```

### Prepared statements: custom vs generic plans

```
 PREPARE p(bigint) AS SELECT * FROM orders WHERE user_id = $1;

 EXECUTE p(4471)   #1 → CUSTOM plan. $1 is known → MCV/histogram lookup.
 EXECUTE p(4471)   #2 → custom
 ...               #5 → custom
 EXECUTE p(4471)   #6 → PostgreSQL compares:
                          avg(custom plan cost) + planning cost
                          vs generic plan cost
                        If generic is not clearly worse → USE GENERIC
                        from now on, and stop planning.

 GENERIC PLAN: built with NO parameter values.
   selectivity for `user_id = $1` = 1 / n_distinct  ← THE AVERAGE

 ⇒ For a skewed column, one plan now serves a user with 4 rows and a
   user with 900,000. (Topic 15's production scenario.)

 CONTROL IT:
   SET plan_cache_mode = 'auto';                 -- default
   SET plan_cache_mode = 'force_custom_plan';    -- always re-plan
   SET plan_cache_mode = 'force_generic_plan';   -- never re-plan

 ⚠ node-postgres only creates a prepared statement when you pass a
   `name` to query(). Unnamed queries are re-planned every time — which
   costs planning time but avoids this trap entirely.
```

### When the planner is wrong — the four causes

```
 ① BAD ROW ESTIMATE            (Topic 15)
    signature: estimated vs actual off by >10× on a leaf node
    fix: ANALYZE · statistics_target · CREATE STATISTICS · custom plans
    ⚠ NO index fixes this.

 ② WRONG COST CONSTANTS
    signature: estimates are ACCURATE, but the planner still picks the
               slower plan
    fix: random_page_cost = 1.1 on SSD; effective_cache_size = 75% RAM

 ③ NO GOOD PATH EXISTS
    signature: estimates accurate, plan is the best available, still slow
    fix: this is a genuine index/schema problem (Topics 13, 14)

 ④ PLANNER LIMITATION
    signature: >12 relations (GEQO), correlated subqueries that can't be
               pulled up, functions the planner can't see through
    fix: restructure the query; materialise a CTE; split it

 ★ DIAGNOSE IN THIS ORDER. ① is by far the most common, and the only
   one where adding indexes actively makes things worse.
```

---

## Concept breakdown

```
COST
│  └── An arbitrary unit where 1.0 = one sequential page read.
│      Not milliseconds. Not I/Os. Only comparable to other costs.
│
├── STARTUP COST  work before the first row can be emitted
└── TOTAL COST    work to emit all rows
    ⇒ LIMIT interpolates between them. This is why LIMIT flips plans.

THE COST CONSTANTS  (all tunable, all defaults from a different era)
├── seq_page_cost 1.0 · random_page_cost 4.0    ⚠ set to 1.1 on SSD
├── cpu_tuple_cost 0.01 · cpu_index_tuple_cost 0.005 · cpu_operator_cost 0.0025
└── effective_cache_size 4GB   ⚠ an ESTIMATE the planner uses; allocates
                                 nothing. Set to ~75% of RAM.

EXPLAIN OPTIONS
├── ANALYZE   actually RUNS the query. ⚠ wrap writes in BEGIN…ROLLBACK
├── BUFFERS   page hits/reads/dirtied per node       ★ always use this
├── VERBOSE   output column lists, schema-qualified names
├── SETTINGS  non-default planner settings in effect ★ underused
├── WAL       WAL generated (PG13+), for write statements
└── FORMAT JSON|YAML|XML  machine-readable

THE FIELDS THAT MATTER, IN ORDER
├── ① estimated vs actual rows      the master diagnostic
├── ② loops                          inner-node values are PER LOOP
├── ③ Rows Removed by Filter         I/O you paid for and discarded
├── ④ Buffers: read vs hit           cold cache or too-large working set
├── ⑤ Heap Fetches                   index-only scan falling back (T12)
├── ⑥ Heap Blocks: lossy             bitmap exceeded work_mem (T12)
└── ⑦ Sort Method / Memory / Disk    external merge = work_mem too small

THE `loops` TRAP
│   ->  Index Scan on users  (actual time=0.001..0.002 rows=1 loops=184220)
│       reads as "1 row, 2 microseconds" — harmless-looking.
└── REALITY: 184,220 × 0.002 ms = 368 ms, and 184,220 index descents.
    ★ ALWAYS multiply inner-node numbers by loops.

DISABLING PLAN TYPES — for DIAGNOSIS ONLY
    SET enable_seqscan = off;      -- these set a cost of 1e10,
    SET enable_nestloop = off;     -- they do not truly disable
    SET enable_hashjoin = off;
    SET enable_bitmapscan = off;
    SET enable_indexscan = off;
    ⇒ Use them to ANSWER "would the other plan be faster?"
    ⚠ NEVER leave them set in production. They are a measuring tool,
      not a fix. If the other plan is faster, find out WHY the planner
      didn't choose it — that's your actual bug.
```

---

## Diagrams

**Diagram 1 — big picture: how the choice is made**

```
                     QUERY TREE
                          │
      ┌───────────────────┼────────────────────┐
      ▼                   ▼                    ▼
  ENUMERATE           ESTIMATE              COMPARE
  base paths          selectivity           total cost
      │               (Topic 15)                │
      │                    │                    │
  Seq Scan ─────────┐      │                    │
  Index Scan ───────┼──────┼─── rows ───────────┤
  Bitmap Scan ──────┤      │                    │
  Index Only Scan ──┘      │                    │
      │                    │                    ▼
      ▼                    │              ┌───────────┐
  JOIN PATHS               │              │  CHEAPEST │
  nested loop ─────────────┼──── cost ───▶│   PATH    │
  hash join ───────────────┤              └─────┬─────┘
  merge join ──────────────┘                    │
      │                                          ▼
  (× join ORDER: n! combinations,          PLAN TREE
   GEQO above geqo_threshold=12)                │
                                                 ▼
                                            EXECUTOR
```

**Diagram 2 — data flow: reading a plan bottom-up**

```
 Limit  (cost=48210.44..48210.49 rows=20)
                                                    ⑤ finally, take 20
   ->  Sort  (cost=48210.44..48460.44 rows=100000)
         Sort Key: o.created_at DESC                ④ BLOCKING: sorts
         Sort Method: external merge  Disk: 24MB       everything first
                                                        ⚠ spilled to disk
         ->  Hash Join  (cost=1204.00..44210.00 rows=100000)
               Hash Cond: (o.user_id = u.id)        ③ probe the hash
               ->  Seq Scan on orders o             ② scan 8M rows
                     (cost=0.00..38210.00 rows=8000000)
               ->  Hash  (cost=804.00..804.00 rows=32000)
                     ->  Seq Scan on users u        ① build the hash first
                           (cost=0.00..804.00 rows=32000)

 ★ READ BOTTOM-UP, INSIDE-OUT. Execution starts at the deepest,
   most-indented node. Costs are CUMULATIVE — a parent's cost
   INCLUDES its children's.
 ★ Sort's cost (48,210) minus Hash Join's (44,210) = 4,000 is the
   sort's own cost. That's how you attribute cost to a single node.
```

**Diagram 3 — before/after: the estimate error cascade**

```
 ESTIMATED PLAN                        ACTUAL EXECUTION
 ┌────────────────────────────┐        ┌────────────────────────────┐
 │ Nested Loop      rows=104  │        │ Nested Loop  rows=184,220  │
 │   cost=1.28..842           │        │   actual time=…8412 ms     │
 │   ├─ Index Scan  rows=104  │        │   ├─ Index Scan rows=184220│
 │   └─ Index Scan  rows=1    │        │   └─ Index Scan rows=1     │
 │        (×104 loops)        │        │        (×184,220 LOOPS)    │
 └────────────────────────────┘        └────────────────────────────┘
      planner: "cheap!"                     reality: 8.4 seconds

 THE CASCADE — one wrong number, four wrong decisions:
   ① scan choice   index scan looked right for 104 rows      ✗ for 184k
   ② join method   nested loop is optimal for ~100 rows      ✗ hash wins
   ③ join order    the small side should drive               ✗ it wasn't small
   ④ work_mem      no hash allocated                         ✗ needed one

 ⇒ FIX THE ESTIMATE AND ALL FOUR CORRECT THEMSELVES.
   Adding an index here would give the planner a FIFTH wrong option.
```

---

## Example 1 — basic

```sql
CREATE TABLE users (id bigserial PRIMARY KEY, name text, country text);
INSERT INTO users (name, country)
SELECT 'User '||i, (ARRAY['IN','US','GB','SG'])[1+(i%4)] FROM generate_series(1,200000) i;

CREATE TABLE orders (
  id bigserial PRIMARY KEY, user_id bigint NOT NULL REFERENCES users(id),
  status text NOT NULL, total_paise bigint NOT NULL, created_at timestamptz NOT NULL
);
INSERT INTO orders (user_id, status, total_paise, created_at)
SELECT (random()*200000)::bigint+1,
       CASE WHEN random()<0.94 THEN 'delivered' ELSE 'pending' END,
       (random()*500000)::bigint, now() - (random()*400)::int * interval '1 day'
FROM generate_series(1,8000000);
CREATE INDEX idx_orders_user ON orders(user_id);
VACUUM ANALYZE users; VACUUM ANALYZE orders;
```

**Step 1 — costs are comparable numbers, not time.**

```sql
EXPLAIN SELECT * FROM orders WHERE user_id = 4471;
EXPLAIN SELECT count(*) FROM orders;
```
```
Index Scan using idx_orders_user on orders  (cost=0.43..152.61 rows=40 width=41)
Finalize Aggregate  (cost=86412.11..86412.12 rows=1 width=8)
```
`152.61` vs `86,412.12` — a ratio of 566. Time them and the real ratio is different, because cost units aren't milliseconds. **Cost is only ever meaningful in comparison to another cost for the same query.**

**Step 2 — `random_page_cost`, the most impactful one-line change.**

```sql
SHOW random_page_cost;   -- 4
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE user_id BETWEEN 1000 AND 3000;
```
```
Bitmap Heap Scan on orders  (cost=1421.44..88104.21 rows=79412 width=41)
                            (actual time=18.4..214.1 rows=80112 loops=1)
  Buffers: shared hit=1204 read=27204
Execution Time: 218.4 ms
```
```sql
SET random_page_cost = 1.1;
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE user_id BETWEEN 1000 AND 3000;
```
```
Index Scan using idx_orders_user on orders  (cost=0.43..31204.88 rows=79412 width=41)
                                            (actual time=0.04..142.2 rows=80112 loops=1)
  Buffers: shared hit=8104 read=18204
Execution Time: 146.1 ms
```
**Same query, same data, same index. The plan changed and it's 33% faster** — because the default assumed random reads cost 4× sequential, which is false on SSD.

```sql
RESET random_page_cost;
```

**Step 3 — startup cost and `LIMIT`.**

```sql
EXPLAIN SELECT * FROM orders ORDER BY created_at DESC;
```
```
Sort  (cost=1284102.31..1304102.31 rows=8000000 width=41)
      ^^^^^^^^^^ enormous startup — must sort all 8M first
  Sort Key: created_at DESC
  ->  Seq Scan on orders  (cost=0.00..178204.00 rows=8000000 width=41)
```
```sql
CREATE INDEX idx_orders_created ON orders(created_at DESC);
VACUUM ANALYZE orders;
EXPLAIN SELECT * FROM orders ORDER BY created_at DESC;
```
```
Index Scan using idx_orders_created on orders
  (cost=0.43..412104.88 rows=8000000 width=41)
        ^^^^ startup ≈ 0
```
Note the **total** cost is lower for the Sort in some configurations — but add a `LIMIT`:
```sql
EXPLAIN SELECT * FROM orders ORDER BY created_at DESC LIMIT 20;
```
```
Limit  (cost=0.43..1.46 rows=20 width=41)
  ->  Index Scan using idx_orders_created on orders  (cost=0.43..412104.88 …)
```
**Cost 1.46 instead of 1,284,102.** The planner scaled the index scan's cost by `20/8,000,000`. A blocking Sort cannot be scaled that way — hence the 880,000× difference.

**Step 4 — reading `loops` correctly.**

```sql
EXPLAIN (ANALYZE)
SELECT o.id, u.name FROM orders o JOIN users u ON u.id = o.user_id
WHERE o.status = 'pending';
```
```
Hash Join  (cost=5204.00..214882.00 rows=480000) (actual rows=479112 loops=1)
  ->  Seq Scan on orders o (actual rows=479112 loops=1)
  ->  Hash  (actual rows=200000 loops=1)
        ->  Seq Scan on users u (actual rows=200000 loops=1)
Execution Time: 412.1 ms
```
```sql
SET enable_hashjoin = off; SET enable_mergejoin = off;
EXPLAIN (ANALYZE) SELECT o.id, u.name FROM orders o JOIN users u ON u.id=o.user_id
WHERE o.status='pending';
```
```
Nested Loop  (actual time=0.04..3841.2 rows=479112 loops=1)
  ->  Seq Scan on orders o (actual rows=479112 loops=1)
  ->  Index Scan using users_pkey on users u
        (actual time=0.005..0.006 rows=1 loops=479112)   ← ★ 479,112 LOOPS
Execution Time: 3842.9 ms
```
The inner node reads as "6 microseconds, 1 row." **× 479,112 loops = 2.9 seconds.** Always multiply.

```sql
RESET enable_hashjoin; RESET enable_mergejoin;
```

**Step 5 — `Sort Method` reveals a `work_mem` problem.**

```sql
SET work_mem = '1MB';
EXPLAIN (ANALYZE) SELECT * FROM orders ORDER BY total_paise LIMIT 2000000;
```
```
Limit (actual time=8412.1..9204.4 rows=2000000 loops=1)
  ->  Sort  (actual time=8412.1..8904.2 rows=2000000 loops=1)
        Sort Method: external merge  Disk: 118416kB       ← ⚠ spilled
Execution Time: 9412.8 ms
```
```sql
SET work_mem = '256MB';
-- Sort Method: quicksort  Memory: 204812kB
-- Execution Time: 3104.2 ms          ← 3× faster
RESET work_mem;
```

**Step 6 — the JSON format, for tooling.**

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) SELECT * FROM orders WHERE user_id = 4471;
```
Feed this to https://explain.dalibo.com or `pev2` for a visual tree with per-node cost attribution. **For plans over ~20 lines, use a visualiser** — attributing cost by hand from cumulative numbers is error-prone.

---

## Example 2 — production scenario

**The situation.** A merchant analytics endpoint. p99 was 200 ms for a year; over three weeks it becomes 34 seconds. No deploy, no schema change, traffic flat.

```sql
SELECT m.name, count(*) AS orders, sum(o.total_paise) AS revenue
FROM orders o
JOIN merchants m ON m.id = o.merchant_id
JOIN order_items oi ON oi.order_id = o.id
WHERE o.merchant_id = ANY($1)
  AND o.created_at >= $2
GROUP BY m.name;
```

**Step 1 — get the plan with everything.**

```sql
EXPLAIN (ANALYZE, BUFFERS, SETTINGS)
SELECT m.name, count(*), sum(o.total_paise)
FROM orders o JOIN merchants m ON m.id=o.merchant_id
JOIN order_items oi ON oi.order_id=o.id
WHERE o.merchant_id = ANY(ARRAY[12,88,412]) AND o.created_at >= '2026-07-01'
GROUP BY m.name;
```
```
GroupAggregate  (cost=2841.02..2914.88 rows=3)
                (actual time=34102.4..34104.1 rows=3 loops=1)
  ->  Nested Loop  (cost=1.71..2840.10 rows=88 width=48)
                   (actual time=0.14..31204.8 rows=8412004 loops=1)   ← ⚠⚠
        ->  Nested Loop  (cost=1.14..1204.22 rows=42 width=40)
                         (actual time=0.09..4102.1 rows=1402001 loops=1) ← ⚠
              ->  Index Scan using idx_orders_merchant_created on orders o
                    (cost=0.57..812.10 rows=42 width=32)
                    (actual time=0.04..1841.2 rows=1402001 loops=1)   ← ⚠ HERE
                    Index Cond: ((merchant_id = ANY (…)) AND (created_at >= …))
              ->  Index Scan using merchants_pkey on merchants m
                    (actual time=0.001..0.001 rows=1 loops=1402001)
        ->  Index Scan using idx_oi_order on order_items oi
                    (actual time=0.004..0.018 rows=6 loops=1402001)   ← ⚠⚠⚠
  Buffers: shared hit=41204882 read=8912004
  Settings: random_page_cost = '4'
Planning Time: 0.412 ms
Execution Time: 34104.9 ms
```

**Step 2 — find the deepest divergent node.**

Work bottom-up. The deepest node with an estimate/actual gap:

```
 Index Scan on orders:   estimated 42, actual 1,402,001    → 33,381× off  ★
 Nested Loop (inner):    estimated 42, actual 1,402,001    → inherited
 Nested Loop (outer):    estimated 88, actual 8,412,004    → inherited
 Index Scan on order_items: rows=6, loops=1,402,001        → 8.4M rows
```

**The root cause is one node.** Everything above it is a *consequence*: the planner chose nested loops because it expected 42 rows, and executed the inner scans 1.4 million times.

**Step 3 — why is the estimate 42?**

```sql
SELECT last_analyze, last_autoanalyze, n_mod_since_analyze, n_live_tup
FROM pg_stat_user_tables WHERE relname = 'orders';
```
```
      last_analyze       | last_autoanalyze | n_mod_since_analyze | n_live_tup
------------------------+------------------+---------------------+------------
 2026-06-28 03:12:04+00 | 2026-06-28 …     |           412008841 |  920004112
```

**412 million modifications since the last ANALYZE.** The `created_at` histogram's maximum is 28 June; the query filters `>= 2026-07-01`. That's **above the histogram's range**, so PostgreSQL estimates ~1 row for the date predicate. This is Topic 15's *moving window*, exactly.

Why didn't autovacuum analyse? The threshold:

```
 50 + 0.1 × 920,004,112 = 92,000,461 modifications
 ⇒ it takes 92 MILLION changes to trigger. At 4M/day, that's 23 days.
   The table drifts out of date for three weeks at a time, on a rolling
   basis — which is exactly the observed symptom.
```

**Step 4 — fix the estimate, not the plan.**

```sql
ANALYZE orders;
EXPLAIN (ANALYZE, BUFFERS) <same query>;
```
```
GroupAggregate  (actual time=1204.8..1205.1 rows=3 loops=1)
  ->  Hash Join  (cost=48210.11..102841.44 rows=8408012)
                 (actual time=412.1..1104.2 rows=8412004 loops=1)
        Hash Cond: (oi.order_id = o.id)
        ->  Seq Scan on order_items oi (actual rows=48000000 loops=1)
        ->  Hash  (actual rows=1402001 loops=1)
              ->  Hash Join (actual rows=1402001 loops=1)
                    ->  Index Scan using idx_orders_merchant_created on orders o
                          (actual rows=1402001 loops=1)
                    ->  Hash → Seq Scan on merchants m (actual rows=4102)
  Buffers: shared hit=88204 read=412008
Execution Time: 1206.4 ms
```

**34,104 ms → 1,206 ms. 28×.** One `ANALYZE`. No index, no query change, no hardware.

The planner switched from nested loops to hash joins **by itself**, because it now knows 1.4 million rows will come out. That's the cascade correcting itself.

**Step 5 — the remaining 1.2 seconds.**

`Seq Scan on order_items` reading 48M rows to join 1.4M. Now a genuine index/plan problem (cause ③):

```sql
CREATE INDEX CONCURRENTLY idx_oi_order_covering
  ON order_items (order_id) INCLUDE (quantity, unit_price_paise);
VACUUM ANALYZE order_items;
-- Execution Time: 412.8 ms
```

And the cost constant (cause ②) — the `SETTINGS` line showed `random_page_cost = '4'` on an NVMe instance:

```sql
ALTER SYSTEM SET random_page_cost = 1.1;
ALTER SYSTEM SET effective_cache_size = '96GB';
SELECT pg_reload_conf();
-- Execution Time: 188.4 ms
```

**Step 6 — prevent recurrence.**

```sql
-- The real fix: this table needs far more frequent statistics
ALTER TABLE orders SET (
  autovacuum_analyze_scale_factor = 0.005,   -- 0.5% not 10%
  autovacuum_analyze_threshold = 50000
);
-- now: 50,000 + 0.005 × 920M = 4.65M modifications → ~1× per day

-- And extended statistics, since merchant_id and created_at correlate
-- (a merchant's orders cluster in time)
CREATE STATISTICS stat_orders_merchant_time (dependencies, ndistinct, mcv)
  ON merchant_id, created_at FROM orders;
ANALYZE orders;
```

```sql
-- Monitoring: which tables are drifting?
SELECT relname, n_live_tup, n_mod_since_analyze,
       round(100.0*n_mod_since_analyze/nullif(n_live_tup,0),1) AS pct_stale,
       last_autoanalyze
FROM pg_stat_user_tables
WHERE n_mod_since_analyze > 1000000
ORDER BY pct_stale DESC;
-- ALERT if pct_stale > 20 on any table with > 10M rows
```

**Summary: 34,104 ms → 188 ms.** The breakdown is the lesson:

| Fix | Cause class | Contribution |
|---|---|---|
| `ANALYZE` | ① bad estimate | 34,104 → 1,206 ms (**96% of the win**) |
| Covering index | ③ no good path | 1,206 → 413 ms |
| `random_page_cost` | ② wrong constant | 413 → 188 ms |

**Nobody's first instinct is `ANALYZE`, and it was 96% of the fix.**

---

## Common mistakes

**1. Reading the plan top-down and chasing the biggest number.**
- *Symptom:* optimising a node that's slow only because its input was wrong.
- *Fix:* read **bottom-up**. Find the *deepest* node where estimated and actual diverge. Everything above is downstream.

**2. Ignoring `loops`.**
- *Symptom:* "the inner scan takes 6 microseconds, it's fine."
- *Engine-level why:* inner-node values in a nested loop are per-loop averages. `loops=479112` means it ran 479,112 times.
- *Fix:* always multiply. A visualiser does this for you.

**3. Using bare `EXPLAIN` instead of `EXPLAIN (ANALYZE, BUFFERS)`.**
- *Symptom:* "the plan looks fine" on a slow query.
- *Engine-level why:* bare `EXPLAIN` shows only *estimates*. The estimate-vs-actual gap — the single most diagnostic number — is invisible.
- *Fix:* always `ANALYZE, BUFFERS`. For writes: `BEGIN; EXPLAIN (ANALYZE) …; ROLLBACK;`

**4. Adding indexes to fix a bad estimate.**
- *Symptom:* five indexes added, no improvement, writes slower.
- *Fix:* check estimate-vs-actual first. If it's off by >10×, fix statistics. Indexes give the planner more ways to be wrong.

**5. Leaving `enable_*` settings off in production.**
- *Symptom:* a query is fast until the data shifts, then catastrophically slow.
- *Engine-level why:* `enable_seqscan = off` sets a cost of 1e10 — it doesn't disable the plan, it makes the planner avoid it even when it's genuinely best.
- *Fix:* these are **measurement tools**. Use them to answer "would the other plan be faster?", then fix the real cause.

**6. `random_page_cost = 4.0` on SSD.**
- *Symptom:* "PostgreSQL won't use my index" across many queries.
- *Fix:* `random_page_cost = 1.1` on NVMe, 1.1–1.5 on network storage. Set `effective_cache_size` to ~75% of RAM at the same time — it's an estimate the planner uses, not an allocation.

**7. Not knowing about GEQO.**
- *Symptom:* a 15-table query gets a different plan on different days.
- *Engine-level why:* above `geqo_threshold` (12 relations) the planner switches to a genetic algorithm, which is non-deterministic.
- *Fix:* reduce join count (split the query, materialise a CTE), or raise `geqo_threshold` and accept longer planning.

---

## Hands-on proof

**PROVE IT #1 — costs are relative, not absolute.**
```sql
EXPLAIN SELECT * FROM orders WHERE user_id=4471;
EXPLAIN SELECT count(*) FROM orders;
\timing on
SELECT * FROM orders WHERE user_id=4471;  SELECT count(*) FROM orders;
-- compare the cost ratio to the time ratio. They differ.
```

**PROVE IT #2 — `random_page_cost` flips plans.**
```sql
SET random_page_cost=4;   EXPLAIN SELECT * FROM orders WHERE user_id BETWEEN 1000 AND 3000;
SET random_page_cost=1.1; EXPLAIN SELECT * FROM orders WHERE user_id BETWEEN 1000 AND 3000;
RESET random_page_cost;
```

**PROVE IT #3 — startup cost and LIMIT.**
```sql
EXPLAIN SELECT * FROM orders ORDER BY total_paise;             -- big Sort
EXPLAIN SELECT * FROM orders ORDER BY total_paise LIMIT 10;    -- top-N heapsort
EXPLAIN SELECT * FROM orders ORDER BY created_at DESC LIMIT 10;-- Index Scan, cost≈1
```

**PROVE IT #4 — the `loops` multiplication.**
```sql
SET enable_hashjoin=off; SET enable_mergejoin=off;
EXPLAIN (ANALYZE) SELECT o.id,u.name FROM orders o JOIN users u ON u.id=o.user_id
WHERE o.status='pending';
RESET enable_hashjoin; RESET enable_mergejoin;
```

**PROVE IT #5 — the generic-plan switch, live.**
```sql
PREPARE g(bigint) AS SELECT count(*) FROM orders WHERE user_id=$1;
EXPLAIN EXECUTE g(4471);   -- ×1..5 custom
EXPLAIN EXECUTE g(4471);   -- ×6 → look for `$1` in the Index Cond
SET plan_cache_mode='force_custom_plan';
EXPLAIN EXECUTE g(4471);   -- back to a literal
RESET plan_cache_mode; DEALLOCATE g;
```

**PROVE IT #6 — see non-default settings that affect the plan.**
```sql
EXPLAIN (ANALYZE, SETTINGS) SELECT * FROM orders WHERE user_id=4471;
-- Settings: random_page_cost = '1.1', work_mem = '256MB'
-- ★ invaluable when a query behaves differently in two environments
```

**PROVE IT #7 — stale statistics, reproduced.**
```sql
CREATE TABLE drift AS SELECT i, now()-(i||' s')::interval AS t FROM generate_series(1,2000000) i;
CREATE INDEX ON drift(t); ANALYZE drift;
INSERT INTO drift SELECT i, now()+(i||' s')::interval FROM generate_series(1,500000) i;
EXPLAIN (ANALYZE) SELECT count(*) FROM drift WHERE t > now();   -- rows=1, actual 500000
ANALYZE drift;
EXPLAIN (ANALYZE) SELECT count(*) FROM drift WHERE t > now();   -- accurate
```

---

## The design decision framework

```
THE READING PROCEDURE — every slow query, in this order:

  0. EXPLAIN (ANALYZE, BUFFERS, SETTINGS)
     For writes: BEGIN; …; ROLLBACK;

  1. READ BOTTOM-UP. Find the DEEPEST node where
       actual rows / estimated rows  > 10
     Everything above it is a consequence. Fix this node only.

  2. CLASSIFY THE CAUSE:
     ① estimate off > 10×      → STATISTICS (Topic 15). Not an index.
     ② estimates fine, wrong plan chosen → COST CONSTANTS
     ③ estimates fine, best plan is still slow → INDEX / SCHEMA
     ④ > 12 relations, or odd shapes → PLANNER LIMITATION; restructure

  3. FOR ③, LOOK FOR THESE SPECIFIC SIGNALS:
     Rows Removed by Filter, large  → column belongs in the index (T14)
     Sort under a Limit             → index matching ORDER BY (T14)
     Heap Fetches > 0 on IOS        → autovacuum (T12)
     Heap Blocks: lossy             → raise work_mem (T12)
     Sort Method: external merge    → raise work_mem
     Buffers read ≫ hit             → covering index / RAM (T07)
     loops in the tens of thousands → wrong join method (T19)

  4. VERIFY the fix with the same EXPLAIN. If the plan didn't change,
     you didn't fix anything.

COST CONSTANTS — set these once, correctly:
  random_page_cost      1.1    (NVMe) · 1.1–1.5 (network storage)
  seq_page_cost         1.0
  effective_cache_size  ~75% of RAM   (an ESTIMATE; allocates nothing)
  work_mem              4–16 MB globally; raise per-session for heavy jobs
  ⚠ work_mem is PER NODE PER BACKEND (Topic 02). Never raise it globally
    to fix one report.

PLAN CACHE MODE:
  ✓ force_custom_plan when a filtered column is heavily SKEWED
    (multi-tenant, marketplaces, anything with whales)
  ✓ auto (default) otherwise
  Cost: ~0.2–0.5 ms of planning per execution. Worth it against a
  100× tail-latency risk.

THE SIGNAL TO LOOK FOR:
      the ratio  actual_rows / estimated_rows  on the DEEPEST node
  • ≤ 2×      statistics are fine — look at cost constants or indexes
  • 10–100×   one of the four causes in Topic 15
  • > 1000×   almost always stale ANALYZE or a generic plan
  ★ And ALWAYS check `SETTINGS` output before comparing two environments.
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
Run `EXPLAIN (ANALYZE, BUFFERS)` on a three-table join. For each node, write down: estimated rows, actual rows, the ratio, and `loops`. Identify the deepest divergent node. Then compute that node's *own* cost by subtracting its children's costs from its total, and explain what that number represents.

### Exercise 2 — medium (apply it)
Take a query whose plan is a Hash Join.
(a) Force a Nested Loop with `enable_hashjoin=off` and measure both.
(b) Explain, using `loops` and the cost model, why the planner chose what it chose.
(c) Find the row count at which the two plans cross over, by varying the predicate.
(d) Change `random_page_cost` from 4 to 1.1 and find the crossover again. Explain the direction it moved.
(e) State what you would actually change in production and why — the constant, the index, or neither.

### Exercise 3 — hard (production simulation)
You are given this plan for an endpoint that regressed from 180 ms to 41 s with no deploy:

```
GroupAggregate (cost=8421.10..8424.88 rows=4) (actual time=41204.8..41206.1 rows=4)
  -> Nested Loop (cost=2.14..8420.02 rows=61) (actual rows=12402881 loops=1)
       -> Nested Loop (cost=1.57..4102.11 rows=28) (actual rows=2104882 loops=1)
            -> Index Scan using idx_events_tenant_time on events e
                 (cost=0.70..2841.02 rows=28) (actual rows=2104882 loops=1)
                 Index Cond: ((tenant_id = $1) AND (occurred_at >= $2))
                 Filter: (severity >= 3)
                 Rows Removed by Filter: 8412004
            -> Index Scan using users_pkey on users u
                 (actual time=0.001..0.001 rows=1 loops=2104882)
       -> Index Scan using idx_sessions_user on sessions s
                 (actual time=0.002..0.004 rows=6 loops=2104882)
  Buffers: shared hit=88204112 read=12402881
  Settings: random_page_cost = '4', work_mem = '4MB'
Planning Time: 0.88 ms
Execution Time: 41206.9 ms
```

(a) Identify the deepest divergent node and compute its ratio.
(b) Classify the cause. Give the exact query that would confirm or rule it out.
(c) Explain why *both* nested loops were chosen, and what the planner would have chosen with a correct estimate.
(d) `Rows Removed by Filter: 8412004` — what does this cost, and what specific change eliminates it? Justify the column position.
(e) The plan shows `$1` and `$2`. What does that tell you, and what is the second possible root cause? Give the three-command test that distinguishes it from cause (b).
(f) `Settings` shows `random_page_cost = '4'` on an NVMe box. Estimate the additional improvement from fixing it, and explain the direction.
(g) Rank all your fixes by expected contribution, and give the deployment order with the reasoning for that order.

---

## Mental model checkpoint

1. What are the units of `cost`? What does `1.0` mean?
2. Explain startup cost vs total cost, and why `LIMIT` can change which plan wins by five orders of magnitude.
3. In a nested loop, the inner node shows `rows=1 loops=184220`. How much work did it do?
4. Name the four causes of a bad plan. Which is most common, and which one is made *worse* by adding indexes?
5. Why is `random_page_cost = 4.0` wrong on modern hardware, and what does it bias the planner toward?
6. What's the difference between a custom plan and a generic plan? When does PostgreSQL switch, and when is that switch harmful?
7. Why should you read a plan bottom-up rather than top-down?

---

## Quick reference card

**Cost constants**

| Setting | Default | Set to |
|---|---|---|
| `seq_page_cost` | 1.0 | 1.0 |
| `random_page_cost` | 4.0 | **1.1** (SSD) |
| `cpu_tuple_cost` | 0.01 | — |
| `effective_cache_size` | 4 GB | **~75% of RAM** |
| `work_mem` | 4 MB | 4–16 MB; per-session for heavy jobs |
| `geqo_threshold` | 12 | — |

**EXPLAIN options**

```sql
EXPLAIN (ANALYZE, BUFFERS, SETTINGS) <query>;      -- the standard incantation
BEGIN; EXPLAIN (ANALYZE) <write>; ROLLBACK;         -- for INSERT/UPDATE/DELETE
EXPLAIN (ANALYZE, FORMAT JSON) <query>;             -- for pev2 / dalibo
```

**Fields, in order of diagnostic value**

| Field | Means |
|---|---|
| est vs actual rows | ★ the master diagnostic |
| `loops` | multiply inner-node values by this |
| `Rows Removed by Filter` | I/O paid for and discarded → index column order |
| `Buffers: read` vs `hit` | cold cache or working set too large |
| `Heap Fetches` | index-only scan falling back → VACUUM |
| `Heap Blocks: lossy` | bitmap exceeded `work_mem` |
| `Sort Method: external merge` | `work_mem` too small |
| `Settings:` | non-default planner config — check across environments |

**Diagnosis tools (never leave these set)**

```sql
SET enable_seqscan|indexscan|bitmapscan|nestloop|hashjoin|mergejoin = off;
SET plan_cache_mode = 'force_custom_plan';
```

**The procedure:** bottom-up → deepest divergent node → classify (statistics / constants / index / limitation) → fix → re-EXPLAIN.

---

## When would I use this at work?

1. **Every performance ticket.** `EXPLAIN (ANALYZE, BUFFERS, SETTINGS)` and the bottom-up rule turn a vague report into a specific node and a specific cause in about three minutes — and, crucially, tell you what *not* to work on.

2. **A regression with no deploy.** The estimate-vs-actual ratio on the deepest node points straight at stale statistics. `ANALYZE` was 96% of the fix in the production example above, and it's the last thing most teams try.

3. **"It's fast in staging."** `EXPLAIN (… SETTINGS)` in both environments usually reveals a different `random_page_cost`, `work_mem`, or `effective_cache_size` — a two-minute answer to a question that otherwise eats a day.

---

## Connected topics

**Understand before this:** 09 (the five phases), 12 (scan types), 15 (statistics — the planner's inputs).

**This unlocks:**
- **19** — join algorithms: the options the planner is choosing between
- **28** — migrations, where plan stability matters
- **58** — read replicas, which have their own statistics and plans
- **65** — connection pooling and prepared-statement lifetime
- **66** — the N+1 problem, visible as enormous `loops`
- **67** — the full performance investigation methodology, built on this
