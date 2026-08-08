# 19 — Join Algorithms and Execution Strategies
## Phase: Indexes

---

## ELI5 — The Simple Analogy

You have two lists: 500 students, and 40,000 exam results. You must pair them up.

**Nested Loop.** Take student 1, walk the whole results pile looking for theirs. Then student 2. Then student 3. Fine if you have 3 students. Catastrophic for 500 — unless the results are in a filing cabinet sorted by student ID, in which case each lookup is instant and this is the *best* method.

**Hash Join.** Take the *smaller* list — the 500 students — and build a lookup table: student ID → student, pinned to a board. Then walk the 40,000 results once, glancing at the board for each. Two passes total, no sorting.

**Merge Join.** If *both* lists are already sorted by student ID, walk them side by side like a zip. No lookups, no board, one pass each. Unbeatable — but only if they're already sorted, and sorting 40,000 things just to enable this is usually not worth it.

Three algorithms. The right one depends entirely on **how big each side is** and **what's already sorted or indexed**. And the planner chooses using row estimates — which is why Topic 15 matters more than anything here.

---

## Where this fits in the big picture

```
   15 statistics ──▶ 18 the planner
                          │
                          ▼
        ┌──────────────────────────────────────────┐
        │ 19 JOIN ALGORITHMS                       │ ← YOU ARE HERE
        │ the options the planner picks from       │
        └────────────────────┬─────────────────────┘
                             │  ← PHASE 2 ENDS HERE
                             ▼
                    PHASE 3 — DATABASE DESIGN
                    (20–28: now you can design schemas
                     knowing what a join actually costs)
```

This closes Phase 2. You now know the full cost of every relational operation — which is exactly what you need before designing schemas in Phase 3.

---

## What is this?

The three physical algorithms PostgreSQL uses to combine two row sources, plus the supporting nodes (Sort, Hash, Materialize, Memoize) and the **join order** decision that sits above them.

Join *order* matters as much as join *method*: a 5-table query has 5! = 120 orderings, and the difference between the best and worst is routinely 1,000×.

---

## Why does it matter for a backend developer?

Because `EXPLAIN` is unreadable without it, and because the two most common catastrophic plans are both join problems:

```
 1. NESTED LOOP WITH A HUGE OUTER SIDE
    -> Nested Loop (actual rows=8412004 loops=1)
         -> Seq Scan (actual rows=8412004 loops=1)
         -> Index Scan (actual rows=1 loops=8412004)   ← 8.4M index descents
    This is the N+1 problem, expressed in SQL rather than in your ORM.
    Fix: a hash join — which usually means fixing the row estimate.

 2. HASH JOIN SPILLING TO DISK
    -> Hash  (Batches: 64  Memory Usage: 4096kB)
    64 batches means the hash table didn't fit in work_mem, so both
    sides were written to disk and re-read 64 times.
    Fix: work_mem, or fewer rows on the build side.
```

And a design consequence that lands in Phase 3: **a join is not free, but it is far cheaper than most people think.** A hash join over 8M rows is ~400 ms. Fear of joins is the main driver of premature denormalisation (Topic 53), and knowing the real numbers is what lets you push back.

---

## The physical reality

### Nested Loop

```
 for each row R in OUTER:
     for each row S in INNER matching R:
         emit (R, S)

 ┌──────────────┐         ┌──────────────────────────┐
 │  OUTER       │         │  INNER                   │
 │  users       │         │  orders                  │
 │  ┌────────┐  │         │  ┌────────────────────┐  │
 │  │ id=7   │──┼────────▶│  │ idx lookup user=7  │  │  ← re-executed
 │  ├────────┤  │         │  ├────────────────────┤  │    ONCE PER
 │  │ id=8   │──┼────────▶│  │ idx lookup user=8  │  │    OUTER ROW
 │  ├────────┤  │         │  ├────────────────────┤  │
 │  │ id=9   │──┼────────▶│  │ idx lookup user=9  │  │
 │  └────────┘  │         │  └────────────────────┘  │
 └──────────────┘         └──────────────────────────┘

 COST = outer_rows × (cost of one inner lookup)

   inner is an INDEX SCAN     → ~4 page reads each. Viable to ~10k outer rows.
   inner is a SEQ SCAN        → O(n×m). Catastrophic above trivial sizes.

 ✓ STREAMING: emits the first row immediately → great with LIMIT
 ✓ ZERO memory
 ✓ The ONLY algorithm for non-equality joins (<, >, BETWEEN, &&)
 ✓ The only one that preserves the outer side's ordering
 ✗ Cost is LINEAR in outer rows — 8M outer rows = 8M inner executions
```

### Hash Join

```
 PHASE 1 — BUILD: read the SMALLER side entirely, hash it in memory
 PHASE 2 — PROBE: stream the LARGER side, look each row up

 ┌─────────────────────────────────────────────────────────────┐
 │  HASH TABLE (in work_mem)                                    │
 │   bucket 0: [id=7, 'Arjun'] [id=1004, 'Meera']              │
 │   bucket 1: [id=8, 'Ravi']                                  │
 │   ...                                                        │
 └─────────────────────────────────────────────────────────────┘
        ▲                                    │
        │ BUILD (200,000 users)              │ PROBE (8,000,000 orders)
        │ read once                          ▼ read once, hash each
   ┌─────────┐                        ┌─────────────┐
   │  users  │                        │   orders    │
   └─────────┘                        └─────────────┘

 COST = build_rows + probe_rows      (LINEAR, one pass each)

 ✗ BLOCKING on the build side: no output until the hash is complete
 ✗ Needs memory: build_rows × row_width must fit in work_mem
 ✓ Best for large ⋈ large with an equality condition
 ✓ Doesn't care about ordering or indexes at all

 ★ BATCHING (spilling) when the hash doesn't fit:
   PostgreSQL partitions BOTH sides by hash into N batches, writes them
   to temp files, and processes one batch pair at a time.
     Buckets: 65536  Batches: 64  Memory Usage: 4096kB
                     ^^^^^^^^^^^ >1 means it spilled to disk
   ⇒ Both sides written and re-read. Often a 5–20× slowdown.
```

### Merge Join

```
 Both inputs sorted on the join key. Walk them together.

  OUTER (sorted)      INNER (sorted)
    7  ──────────────▶  7   match, emit
    7  ──────────────▶  7   match, emit
    8                   9
    ↓ advance outer     ↑ inner stays
    9  ──────────────▶  9   match, emit
   12                  10
                        ↓ advance inner
   12  ─────────────▶  12   match, emit

 COST = sort(outer) + sort(inner) + one merge pass
      or, if both come from indexes in the right order, JUST the merge pass

 ✓ Cheapest when BOTH sides are ALREADY sorted (index scans on the join key)
 ✓ Constant memory — streams both sides
 ✓ Output is SORTED → a downstream ORDER BY may be free
 ✗ Requires sorting if inputs aren't ordered — often more expensive than hashing
 ✗ Equality (and some range) conditions only
 ★ MATERIALIZE: when the inner has duplicate keys, the executor must
   rewind it. It buffers the inner side in memory to make rewinding cheap.
   That's the `Materialize` node you see under merge joins.
```

### Memoize (PG14+) — nested loops with a cache

```
 ->  Nested Loop
       ->  Seq Scan on orders  (actual rows=8000000)
       ->  Memoize  (actual rows=1 loops=8000000)
             Cache Key: o.user_id
             Hits: 7,960,000  Misses: 40,000  Evictions: 0  Memory: 3.2MB
             ->  Index Scan using users_pkey on users

 ⇒ 8,000,000 loops but only 40,000 actual index lookups. The other
   7.96M were served from an in-memory LRU cache keyed on user_id.

 ★ Makes nested loops viable when the inner side has FEW DISTINCT keys.
   Look at the Hits:Misses ratio — a high ratio means Memoize turned an
   N+1 into something close to a hash join.
```

---

## How it works — step by step

### Costing the three, on the same query

```
 SELECT o.id, u.name FROM orders o JOIN users u ON u.id = o.user_id
 WHERE o.status = 'pending';
   orders: 8,000,000 rows · 98,000 pages · 480,000 match the filter
   users:    200,000 rows ·   2,400 pages

 ── NESTED LOOP (orders outer, users inner via pkey) ──
   outer: seq scan orders          98,000 + 80,000 CPU  = 178,000
   inner: 480,000 × index lookup on users_pkey
          480,000 × ~4 page reads × random_page_cost(1.1) ≈ 2,112,000
   TOTAL ≈ 2,290,000                                       ✗ expensive

 ── HASH JOIN (users build, orders probe) ──
   build: seq scan users            2,400 + 2,000        =   4,400
          hash 200,000 rows         200,000 × 0.0025     =     500
   probe: seq scan orders          98,000 + 80,000       = 178,000
          probe 480,000            480,000 × 0.0025      =   1,200
   TOTAL ≈ 184,100                                          ✓ WINNER

 ── MERGE JOIN ──
   sort orders by user_id (480,000 rows, spills)         ≈ 240,000
   sort users by id — free, use the pkey index scan      ≈  12,000
   merge pass                                            ≈   4,000
   TOTAL ≈ 256,000

 ⇒ HASH JOIN. And note: nested loop lost by 12×, but if the filter
   matched 200 rows instead of 480,000, nested loop would win by 900×.
   ★ THE ROW ESTIMATE IS THE ENTIRE DECISION.
```

### Join order — the other half

```
 SELECT * FROM orders o
   JOIN users u    ON u.id = o.user_id
   JOIN merchants m ON m.id = o.merchant_id
  WHERE u.country = 'IN' AND m.tier = 'gold';

 users:     200,000 (40,000 with country='IN')
 merchants:   4,000 (120 with tier='gold')
 orders:  8,000,000

 ── ORDER A: (orders ⋈ users) ⋈ merchants ──
   orders ⋈ users     → 1,600,000 intermediate rows
   ⋈ merchants        → 48,000 final
   peak intermediate: 1,600,000

 ── ORDER B: (merchants ⋈ orders) ⋈ users ──
   merchants(120) ⋈ orders → 240,000 intermediate rows
   ⋈ users                 → 48,000 final
   peak intermediate: 240,000        ★ 6.7× less work

 ⇒ THE RULE: apply the most selective filter FIRST, so intermediate
   results stay small. The planner does this automatically via dynamic
   programming — CORRECTLY, IF the row estimates are right.
   ⚠ With a bad estimate it picks Order A and does 6.7× the work.
```

### Join *types* vs join *algorithms* — a distinction people conflate

```
 JOIN TYPE (semantics — what SQL you wrote):
   INNER · LEFT/RIGHT/FULL OUTER · SEMI (EXISTS/IN) · ANTI (NOT EXISTS)

 JOIN ALGORITHM (physics — how the executor does it):
   Nested Loop · Hash · Merge

 These are INDEPENDENT. You can have a Hash Anti Join, a Nested Loop
 Semi Join, a Merge Left Join.

 ★ SEMI JOIN (from EXISTS / IN) stops at the FIRST match per outer row.
   This is why `EXISTS` often beats `IN (SELECT …)` on large subqueries —
   it can short-circuit.
 ★ ANTI JOIN (from NOT EXISTS) emits outer rows with NO match.
   ⚠ `NOT IN (SELECT …)` cannot become an anti join if the subquery
     can produce NULL — SQL's NULL semantics force a much worse plan.
     ALWAYS prefer `NOT EXISTS`.
```

---

## Concept breakdown

```
NESTED LOOP
├── cost = outer_rows × inner_cost
├── ✓ streaming · ✓ zero memory · ✓ non-equality joins · ✓ preserves order
├── ✗ linear in outer rows
├── ★ viable ONLY when: outer is small (< ~10k) OR the inner has an index
│      OR Memoize gets a high hit rate
└── the N+1 problem, in plan form

HASH JOIN
├── cost = build_rows + probe_rows        (linear, one pass each)
├── ✓ best for large ⋈ large · ✓ no index or ordering needed
├── ✗ blocking on the build side · ✗ needs work_mem · ✗ equality only
└── ★ WATCH `Batches:` — anything > 1 means it spilled to disk

MERGE JOIN
├── cost = sort(both) + one merge pass
├── ✓ cheapest when both sides ARE ALREADY sorted · ✓ constant memory
├── ✓ output is sorted (a free ORDER BY)
├── ✗ sorting is expensive if not already ordered
└── ★ needs `Materialize` when the inner has duplicate keys

MEMOIZE (PG14+)
└── an LRU cache in front of a nested loop's inner side.
    Check Hits:Misses — high hits means an N+1 was rescued.

SUPPORTING NODES
├── Sort         BLOCKING. Watch `Sort Method` and `Disk:`
├── Hash         BLOCKING (build side). Watch `Batches:`
├── Materialize  buffers a subtree for rewinding
├── Gather/Gather Merge   parallel workers collecting results
└── Append/Merge Append   partitions or UNION branches

JOIN ORDER
├── n tables → n! orderings; dynamic programming below geqo_threshold(12)
├── the goal: keep INTERMEDIATE results small — most selective first
└── ⚠ driven entirely by row estimates (Topic 15)

PARALLELISM
├── Gather: workers run the plan below in parallel, results collected
├── Parallel Hash Join: the hash table is SHARED across workers (PG11+)
├── controlled by max_parallel_workers_per_gather (default 2)
└── ⚠ each worker gets its OWN work_mem. 4 workers × 64MB = 256MB.
```

---

## Diagrams

**Diagram 1 — big picture: which algorithm, by shape**

```
                    JOIN CONDITION IS EQUALITY?
                              │
              ┌───────NO──────┴───────YES──────┐
              ▼                                ▼
      ┌──────────────┐              IS EITHER SIDE SMALL?
      │ NESTED LOOP  │                         │
      │ (only option)│         ┌──────YES──────┴──────NO──────┐
      └──────────────┘         ▼                              ▼
                     IS THERE AN INDEX ON            ARE BOTH ALREADY
                     THE INNER JOIN KEY?             SORTED ON THE KEY?
                              │                              │
                    ┌───YES───┴───NO───┐          ┌────YES───┴───NO───┐
                    ▼                  ▼          ▼                   ▼
             ┌─────────────┐   ┌────────────┐ ┌──────────┐   ┌─────────────┐
             │ NESTED LOOP │   │ HASH JOIN  │ │  MERGE   │   │ HASH JOIN   │
             │ (+ Memoize) │   │            │ │   JOIN   │   │             │
             └─────────────┘   └────────────┘ └──────────┘   └─────────────┘

  ★ "small" ≈ under a few thousand rows, or small enough to hash in work_mem
```

**Diagram 2 — data flow: cost vs outer-row count**

```
  cost
    │                                          Nested Loop
    │                                        ╱  (linear in
    │                                      ╱    outer rows)
    │                                    ╱
    │                                  ╱
    │      ┌──────────────────────────────────────  Hash Join
    │     ╱                                          (flat: one pass each)
    │   ╱
    │ ╱
    └──────────────────────────────────────────────▶ outer rows
      10      1,000      100,000    10,000,000
             ↑
        CROSSOVER (~1,000–10,000 outer rows, depending on
        inner index depth and random_page_cost)

  BELOW the crossover: nested loop wins — it streams, needs no memory,
                       and works beautifully with LIMIT.
  ABOVE it:            hash join wins, and the gap widens linearly.

  ★ A wrong row estimate puts you on the wrong side of this crossover.
    That is what a "catastrophic plan" physically is.
```

**Diagram 3 — before/after: the estimate error, in join terms**

```
 PLANNER BELIEVES: 61 outer rows          REALITY: 2,104,882 outer rows
 ┌──────────────────────────────┐         ┌──────────────────────────────┐
 │ Nested Loop                  │         │ Nested Loop                  │
 │   outer: 61 rows             │         │   outer: 2,104,882 rows      │
 │   inner: index scan × 61     │         │   inner: index scan          │
 │          ≈ 244 page reads    │         │          × 2,104,882 loops   │
 │   cost ≈ 300                 │         │          ≈ 8.4M page reads   │
 └──────────────────────────────┘         │   actual: 41,204 ms          │
                                          └──────────────────────────────┘

 WITH THE CORRECT ESTIMATE, the planner would have chosen:
 ┌──────────────────────────────┐
 │ Hash Join                    │
 │   build: users (200k) → hash │  one pass
 │   probe: 2,104,882 rows      │  one pass
 │   actual: 1,206 ms           │  ← 34× faster
 └──────────────────────────────┘

 ⇒ The join algorithm was never the bug. The ESTIMATE was.
```

---

## Example 1 — basic

```sql
CREATE TABLE users (id bigserial PRIMARY KEY, name text, country text);
INSERT INTO users (name,country) SELECT 'User '||i,
  (ARRAY['IN','US','GB','SG'])[1+(i%4)] FROM generate_series(1,200000) i;

CREATE TABLE orders (id bigserial PRIMARY KEY, user_id bigint NOT NULL,
  status text NOT NULL, total_paise bigint NOT NULL);
INSERT INTO orders (user_id,status,total_paise)
SELECT (random()*200000)::bigint+1,
       CASE WHEN random()<0.94 THEN 'delivered' ELSE 'pending' END,
       (random()*500000)::bigint FROM generate_series(1,8000000);
CREATE INDEX idx_orders_user ON orders(user_id);
VACUUM ANALYZE users; VACUUM ANALYZE orders;
```

**Step 1 — a small outer side: nested loop wins.**

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT o.id, u.name FROM orders o JOIN users u ON u.id=o.user_id
WHERE u.id BETWEEN 1 AND 50;
```
```
Nested Loop  (actual time=0.04..4.12 rows=2012 loops=1)
  ->  Index Scan using users_pkey on users u (actual rows=50 loops=1)
  ->  Index Scan using idx_orders_user on orders o
        (actual time=0.01..0.07 rows=40 loops=50)      ← 50 loops. Fine.
  Buffers: shared hit=2214
Execution Time: 4.3 ms
```

**Step 2 — a large outer side: hash join wins.**

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT o.id, u.name FROM orders o JOIN users u ON u.id=o.user_id
WHERE o.status='pending';
```
```
Hash Join  (actual time=104.2..812.4 rows=479112 loops=1)
  Hash Cond: (o.user_id = u.id)
  ->  Seq Scan on orders o (actual rows=479112 loops=1)
        Filter: (status = 'pending')
        Rows Removed by Filter: 7520888
  ->  Hash  (actual rows=200000 loops=1)
        Buckets: 262144  Batches: 1  Memory Usage: 11266kB   ← 1 batch ✓
        ->  Seq Scan on users u (actual rows=200000 loops=1)
  Buffers: shared hit=2104 read=88204
Execution Time: 841.2 ms
```

**Step 3 — force the nested loop, to see the crossover.**

```sql
SET enable_hashjoin=off; SET enable_mergejoin=off;
EXPLAIN (ANALYZE, BUFFERS)
SELECT o.id, u.name FROM orders o JOIN users u ON u.id=o.user_id WHERE o.status='pending';
```
```
Nested Loop  (actual time=0.05..4102.8 rows=479112 loops=1)
  ->  Seq Scan on orders o (actual rows=479112 loops=1)
  ->  Memoize  (actual time=0.001..0.001 rows=1 loops=479112)
        Cache Key: o.user_id
        Hits: 291204  Misses: 187908  Evictions: 0  Memory Usage: 18412kB
        ->  Index Scan using users_pkey on users u (actual rows=1 loops=187908)
Execution Time: 4132.1 ms
RESET enable_hashjoin; RESET enable_mergejoin;
```
**4,132 ms vs 841 ms — 4.9× slower.** Note Memoize saved 291,204 of the 479,112 lookups; **without it this would be ~9 seconds.**

**Step 4 — merge join, when both sides are pre-sorted.**

```sql
SET enable_hashjoin=off;
EXPLAIN (ANALYZE, BUFFERS)
SELECT o.id, u.name FROM orders o JOIN users u ON u.id=o.user_id
WHERE o.status='pending' ORDER BY u.id;
```
```
Merge Join  (actual time=1204.1..2841.2 rows=479112 loops=1)
  Merge Cond: (u.id = o.user_id)
  ->  Index Scan using users_pkey on users u (actual rows=200000 loops=1)
  ->  Sort  (actual time=1204.0..1412.8 rows=479112 loops=1)
        Sort Key: o.user_id
        Sort Method: external merge  Disk: 18904kB       ← ⚠ spilled
Execution Time: 2861.4 ms
RESET enable_hashjoin;
```
The sort dominates. Merge join only wins when **neither side needs sorting**.

**Step 5 — hash join spilling to disk.**

```sql
SET work_mem='1MB';
EXPLAIN (ANALYZE) SELECT o.id,u.name FROM orders o JOIN users u ON u.id=o.user_id
WHERE o.status='pending';
```
```
Hash Join  (actual time=204.1..3412.8 rows=479112 loops=1)
  ->  Seq Scan on orders o (actual rows=479112 loops=1)
  ->  Hash  (actual rows=200000 loops=1)
        Buckets: 8192  Batches: 32  Memory Usage: 897kB     ← ⚠ 32 BATCHES
Execution Time: 3441.2 ms
```
```sql
SET work_mem='64MB';
-- Buckets: 262144  Batches: 1  Memory Usage: 11266kB
-- Execution Time: 838.4 ms          ← 4.1× faster
RESET work_mem;
```

**Step 6 — `NOT IN` vs `NOT EXISTS`.**

```sql
EXPLAIN (ANALYZE) SELECT count(*) FROM users u
WHERE u.id NOT IN (SELECT user_id FROM orders WHERE status='pending');
```
```
Aggregate (actual time=8412.1..8412.1 rows=1 loops=1)
  ->  Seq Scan on users u (actual rows=1204 loops=1)
        Filter: (NOT (SubPlan 1))
        SubPlan 1
          ->  Materialize (actual rows=241004 loops=200000)   ← ⚠⚠
Execution Time: 8413.9 ms
```
```sql
EXPLAIN (ANALYZE) SELECT count(*) FROM users u
WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.user_id=u.id AND o.status='pending');
```
```
Aggregate (actual time=612.4..612.4 rows=1 loops=1)
  ->  Hash Anti Join (actual rows=1204 loops=1)
        Hash Cond: (u.id = o.user_id)
Execution Time: 613.1 ms
```
**8,414 ms → 613 ms — 13.7×.** `NOT IN` cannot become an anti join because the subquery might yield NULL, and SQL's NULL semantics make `x NOT IN (…, NULL)` never true. **Always use `NOT EXISTS`.**

**Step 7 — join order.**

```sql
CREATE TABLE merchants (id bigserial PRIMARY KEY, name text, tier text);
INSERT INTO merchants (name,tier) SELECT 'M'||i,
  CASE WHEN i%33=0 THEN 'gold' ELSE 'standard' END FROM generate_series(1,4000) i;
ALTER TABLE orders ADD COLUMN merchant_id bigint;
UPDATE orders SET merchant_id=(random()*4000)::bigint+1;
CREATE INDEX idx_orders_merchant ON orders(merchant_id);
VACUUM ANALYZE orders; VACUUM ANALYZE merchants;

EXPLAIN (ANALYZE) SELECT count(*) FROM orders o
JOIN users u ON u.id=o.user_id JOIN merchants m ON m.id=o.merchant_id
WHERE u.country='IN' AND m.tier='gold';
```
Look at which table the planner scans first. It should start from `merchants` (120 gold rows) — the most selective — keeping intermediates small.

---

## Example 2 — production scenario

**The situation.** An order-history endpoint. p99 12 s. The ORM generates:

```sql
SELECT o.id, o.created_at, o.total_paise, u.name, u.email,
       m.name AS merchant, count(oi.id) AS item_count
FROM orders o
JOIN users u ON u.id = o.user_id
JOIN merchants m ON m.id = o.merchant_id
LEFT JOIN order_items oi ON oi.order_id = o.id
WHERE o.created_at >= now() - interval '30 days'
  AND o.status = ANY($1)
GROUP BY o.id, u.name, u.email, m.name
ORDER BY o.created_at DESC
LIMIT 50;
```

```sql
EXPLAIN (ANALYZE, BUFFERS, SETTINGS) <the query with status = ARRAY['paid','shipped']>;
```
```
Limit  (actual time=12104.2..12104.4 rows=50 loops=1)
  ->  GroupAggregate  (actual time=12104.2..12104.3 rows=50 loops=1)
        ->  Sort  (actual time=12104.1..12104.2 rows=142 loops=1)
              Sort Key: o.created_at DESC, o.id, u.name, u.email, m.name
              Sort Method: external merge  Disk: 412880kB          ← ⚠②
              ->  Nested Loop Left Join (actual rows=8412004 loops=1)
                    ->  Nested Loop (actual rows=1402001 loops=1)
                          ->  Nested Loop (actual rows=1402001 loops=1)
                                ->  Index Scan using idx_orders_created on orders o
                                      (actual rows=1402001 loops=1)
                                      Index Cond: (created_at >= …)
                                      Filter: (status = ANY (…))
                                      Rows Removed by Filter: 4812004  ← ⚠③
                                ->  Index Scan using users_pkey on users u
                                      (actual rows=1 loops=1402001)     ← ⚠①
                          ->  Index Scan using merchants_pkey on merchants m
                                      (actual rows=1 loops=1402001)     ← ⚠①
                    ->  Index Scan using idx_oi_order on order_items oi
                                      (actual rows=6 loops=1402001)     ← ⚠①
  Buffers: shared hit=48204112 read=8912004
  Settings: work_mem = '4MB', random_page_cost = '4'
Execution Time: 12105.1 ms
```

**Four distinct problems, in one plan:**

| ⚠ | Problem | Evidence |
|---|---|---|
| ① | Three nested loops with a 1.4M-row outer side | `loops=1402001` × 3 = 4.2M index descents |
| ② | Sort spilled to disk | `external merge Disk: 412880kB` on `work_mem = 4MB` |
| ③ | `status` filtered after the index scan | `Rows Removed by Filter: 4812004` |
| ④ | `LIMIT 50` bought nothing | the `GroupAggregate`+`Sort` are blocking |

**Step 1 — why nested loops?**

```sql
-- estimated vs actual on the driving scan
-- estimated rows=1204, actual rows=1402001  →  1,164× off
SELECT last_analyze, n_mod_since_analyze FROM pg_stat_user_tables WHERE relname='orders';
--  2026-07-02 | 88204112       ← stale (Topic 15's moving window)
ANALYZE orders;
```
```
->  Hash Left Join (actual rows=8412004 loops=1)
      ->  Hash Join (actual rows=1402001 loops=1)
            ->  Hash Join (actual rows=1402001 loops=1)
Execution Time: 3412.8 ms                       ← 12,105 → 3,413 ms
```
**One `ANALYZE`: 3.5×.** The planner switched all three joins to hash joins by itself.

**Step 2 — the spilling sort.**

```sql
SET work_mem = '256MB';   -- session only, for this endpoint's pool
-- Sort Method: quicksort  Memory: 188412kB
-- Execution Time: 1841.2 ms
```
⚠ Do **not** set this globally — `work_mem` is per node per backend (Topic 02). At 200 connections × 3 sort nodes this would be 150 GB.

**Step 3 — the filter.**

```sql
CREATE INDEX CONCURRENTLY idx_orders_status_created
  ON orders (status, created_at DESC) INCLUDE (user_id, merchant_id, total_paise);
VACUUM ANALYZE orders;
-- Rows Removed by Filter: 0
-- Execution Time: 684.1 ms
```
(`status` first because it's the equality predicate, `created_at` last because it's the range — Topic 14.)

**Step 4 — the real fix: restructure the query.**

Even at 684 ms, the plan aggregates 8.4M `order_items` rows to return 50. The `LIMIT` can't push down through the `GROUP BY`. Restructure so the limit applies first:

```sql
WITH page AS (
  SELECT o.id, o.created_at, o.total_paise, o.user_id, o.merchant_id
  FROM orders o
  WHERE o.status = ANY($1) AND o.created_at >= now() - interval '30 days'
  ORDER BY o.created_at DESC
  LIMIT 50                                    -- ★ 50 rows, from the index
)
SELECT p.id, p.created_at, p.total_paise, u.name, u.email, m.name AS merchant,
       (SELECT count(*) FROM order_items oi WHERE oi.order_id = p.id) AS item_count
FROM page p
JOIN users u ON u.id = p.user_id
JOIN merchants m ON m.id = p.merchant_id
ORDER BY p.created_at DESC;
```
```
Sort (actual time=2.14..2.15 rows=50 loops=1)
  ->  Nested Loop (actual rows=50 loops=1)
        ->  Nested Loop (actual rows=50 loops=1)
              ->  Subquery Scan on page (actual rows=50 loops=1)
                    ->  Limit (actual rows=50 loops=1)
                          ->  Index Scan using idx_orders_status_created
                                (actual rows=50 loops=1)     ← ★ 50, not 1.4M
              ->  Index Scan using users_pkey (loops=50)
        ->  Index Scan using merchants_pkey (loops=50)
        SubPlan 1 -> Aggregate (loops=50)
Execution Time: 2.2 ms
```

**684 ms → 2.2 ms.** And the plan is now *correctly* nested loops — with an outer side of 50, that's the right algorithm.

**Step 5 — the cost constant.**

```sql
ALTER SYSTEM SET random_page_cost = 1.1;
ALTER SYSTEM SET effective_cache_size = '96GB';
SELECT pg_reload_conf();
-- Execution Time: 1.8 ms
```

**Summary: 12,105 ms → 1.8 ms (6,725×).** Attribution:

| Fix | Contribution |
|---|---|
| `ANALYZE` (join method corrected itself) | 12,105 → 3,413 ms |
| `work_mem` per session | 3,413 → 1,841 ms |
| Composite covering index | 1,841 → 684 ms |
| **Query restructure (limit before join)** | **684 → 2.2 ms** |
| `random_page_cost` | 2.2 → 1.8 ms |

**The largest single win was restructuring so the `LIMIT` applied before the joins.** No index and no setting can compensate for a query that aggregates 8.4M rows to return 50.

---

## Common mistakes

**1. Blaming the join algorithm instead of the estimate.**
- *Symptom:* "PostgreSQL chose a nested loop, it should have used a hash join."
- *Engine-level why:* the planner chose correctly *for the row count it believed*. The bug is upstream.
- *Fix:* check estimate-vs-actual on the driving scan first (Topic 15).

**2. `SET enable_nestloop = off` as a fix.**
- *Symptom:* it works, until the data shifts and a genuinely-good nested loop becomes impossible.
- *Fix:* diagnosis tool only. Find why the estimate was wrong.

**3. Ignoring `Batches:` on a hash node.**
- *Symptom:* a hash join much slower than expected.
- *Engine-level why:* `Batches: 32` means both sides were partitioned to disk and re-read.
- *Fix:* raise `work_mem` **for that session**, or reduce build-side rows.

**4. `NOT IN` with a subquery.**
- *Symptom:* a query 10–100× slower than the `NOT EXISTS` equivalent.
- *Engine-level why:* if the subquery can produce NULL, `NOT IN` cannot become an anti join — SQL's three-valued logic forbids it. The planner falls back to a correlated SubPlan.
- *Fix:* **always `NOT EXISTS`.** Same result, correct NULL semantics, hash anti join.

**5. Raising `work_mem` globally.**
- *Engine-level why:* per node, per backend. 200 connections × 4 nodes × 256 MB = 200 GB.
- *Fix:* `SET LOCAL work_mem` in the specific transaction, or a dedicated role for reporting.

**6. Expecting `LIMIT` to help with a blocking node above the join.**
- *Symptom:* `LIMIT 50` on a query that still takes 12 s.
- *Engine-level why:* `Sort`, `Hash` (build), and ungrouped `Aggregate` are blocking — they consume everything before emitting.
- *Fix:* restructure so the `LIMIT` applies to an index-ordered scan first (Example 2, step 4).

**7. Not noticing `Memoize`.**
- *Symptom:* a nested loop with millions of loops that's somehow fast.
- *Engine-level why:* PG14+ caches inner results. `Hits: 291204 Misses: 187908` means most lookups never happened.
- *Fix:* nothing — but read the ratio before concluding a nested loop is the problem.

---

## Hands-on proof

**PROVE IT #1 — force each algorithm and compare.**
```sql
\timing on
SET enable_hashjoin=off; SET enable_mergejoin=off;
EXPLAIN (ANALYZE) SELECT count(*) FROM orders o JOIN users u ON u.id=o.user_id;
RESET enable_hashjoin; SET enable_mergejoin=off; SET enable_nestloop=off;
EXPLAIN (ANALYZE) SELECT count(*) FROM orders o JOIN users u ON u.id=o.user_id;
RESET enable_mergejoin; RESET enable_hashjoin; SET enable_nestloop=off; SET enable_hashjoin=off;
EXPLAIN (ANALYZE) SELECT count(*) FROM orders o JOIN users u ON u.id=o.user_id;
RESET ALL;
```

**PROVE IT #2 — find the crossover.**
```sql
-- vary the outer size and watch the plan flip
EXPLAIN SELECT count(*) FROM orders o JOIN users u ON u.id=o.user_id WHERE u.id < 100;
EXPLAIN SELECT count(*) FROM orders o JOIN users u ON u.id=o.user_id WHERE u.id < 5000;
EXPLAIN SELECT count(*) FROM orders o JOIN users u ON u.id=o.user_id WHERE u.id < 100000;
```

**PROVE IT #3 — hash spilling.**
```sql
SET work_mem='1MB';  EXPLAIN (ANALYZE) SELECT count(*) FROM orders o JOIN users u ON u.id=o.user_id;
SET work_mem='64MB'; EXPLAIN (ANALYZE) SELECT count(*) FROM orders o JOIN users u ON u.id=o.user_id;
RESET work_mem;
```

**PROVE IT #4 — Memoize hit ratio.**
```sql
SET enable_hashjoin=off; SET enable_mergejoin=off;
EXPLAIN (ANALYZE) SELECT count(*) FROM orders o JOIN users u ON u.id=o.user_id
WHERE o.user_id < 1000;      -- few distinct keys → high hit rate
RESET ALL;
```

**PROVE IT #5 — `NOT IN` vs `NOT EXISTS`.** (Example 1, step 6.)

**PROVE IT #6 — semi join short-circuits.**
```sql
EXPLAIN (ANALYZE) SELECT count(*) FROM users u
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.user_id=u.id);
-- Hash Semi Join — note it stops at the first match per outer row
```

**PROVE IT #7 — parallel hash join.**
```sql
SET max_parallel_workers_per_gather = 4;
EXPLAIN (ANALYZE) SELECT count(*) FROM orders o JOIN users u ON u.id=o.user_id;
-- Gather → Parallel Hash Join → Parallel Hash (shared build)
RESET max_parallel_workers_per_gather;
```

---

## The design decision framework

```
YOU DO NOT CHOOSE THE ALGORITHM — the planner does. Your job is to
give it correct inputs and good options.

TO GET A NESTED LOOP (when you want one — small outer + LIMIT):
  ✓ make the outer side genuinely small (push filters and LIMIT down)
  ✓ index the inner join key
  ✓ ensure the estimate reflects the small outer side

TO GET A HASH JOIN (large ⋈ large):
  ✓ accurate row estimates so the planner sees the true size
  ✓ enough work_mem that Batches = 1
  ✓ equality join condition

TO GET A MERGE JOIN (rare, but excellent when it fits):
  ✓ both sides arrive sorted from index scans on the join key
  ✓ useful when the output ordering is also needed downstream

THE FOUR JOIN-RELATED FIXES, IN ORDER OF LEVERAGE:
  1. FIX THE ESTIMATE          (Topic 15) — corrects method AND order
  2. RESTRUCTURE THE QUERY     push LIMIT/filters below the joins
  3. INDEX THE JOIN KEY        composite, correctly ordered (Topic 14)
  4. RAISE work_mem            per session, never globally

QUERY SHAPES THAT MATTER:
  NOT IN (subquery)   → ALWAYS rewrite as NOT EXISTS
  IN (subquery)       → usually fine (semi join), but EXISTS is safer
  OR across tables    → often prevents a good plan; try UNION ALL
  LIMIT above GROUP BY→ restructure so LIMIT applies to an ordered scan
  n > 12 tables       → GEQO; split the query or materialise a CTE

THE SIGNAL TO LOOK FOR:
      loops on any inner node
  • loops < 1,000            fine
  • loops 1,000–100,000      check Memoize hits; may still be fine
  • loops > 100,000          ★ wrong join method, almost certainly
                               caused by a wrong estimate

      Batches: on any Hash node
  • Batches: 1               fine
  • Batches: > 1             spilled to disk → raise work_mem for this
                               query, or reduce the build side
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
Force each of the three algorithms on the same two-table join using `enable_*` settings. Record execution time, buffers, and peak memory for each. Then explain, using the cost formulas, why the planner's unforced choice was correct.

### Exercise 2 — medium (apply it)
Find the crossover point between nested loop and hash join on your data by varying the outer-side row count. Then:
(a) change `random_page_cost` from 4 to 1.1 and find it again,
(b) explain which direction it moved and why, from the cost formula,
(c) add `Memoize` to the picture by restricting the inner side's distinct keys, and find the crossover a third time,
(d) state what all three numbers imply about when to trust a nested loop in production.

### Exercise 3 — hard (production simulation)
You are given this plan (p99 regressed 200 ms → 28 s, no deploy):

```
Limit (actual time=28104.1..28104.2 rows=20 loops=1)
  -> GroupAggregate (actual rows=20 loops=1)
       -> Sort (actual rows=4204882 loops=1)
            Sort Key: o.created_at DESC, o.id
            Sort Method: external merge  Disk: 1204880kB
            -> Nested Loop Left Join (actual rows=4204882 loops=1)
                 -> Nested Loop (actual rows=702014 loops=1)
                      -> Seq Scan on orders o (actual rows=702014 loops=1)
                           Filter: (status = ANY ('{paid,shipped}'))
                           Rows Removed by Filter: 11204882
                      -> Index Scan using users_pkey on users u
                           (actual time=0.001..0.002 rows=1 loops=702014)
                 -> Index Scan using idx_oi_order on order_items oi
                           (actual time=0.003..0.008 rows=6 loops=702014)
  Buffers: shared hit=41204882 read=8204112
  Settings: work_mem = '4MB', random_page_cost = '4'
Planning Time: 1.204 ms
Execution Time: 28105.4 ms
```

(a) Identify every problem in this plan and rank them by contribution to the 28 s.
(b) The outer side of the first nested loop is 702,014 rows. Compute the total index descents performed and the approximate page reads.
(c) Why did the planner choose nested loops? Give the exact query that confirms your hypothesis.
(d) `LIMIT 20` returned 20 rows but the plan processed 4.2M. Explain which nodes are blocking and why the limit couldn't push down.
(e) Restructure the query so the `LIMIT` applies before the joins. Show the new SQL and predict the new plan shape.
(f) Design the index your restructured query needs, justifying the column order.
(g) Give all five fixes in deployment order with expected contribution, and explain why that order (not just by size of win).

---

## Mental model checkpoint

1. Give the cost formula for each of the three join algorithms. Which is linear in what?
2. When is a nested loop the *best* choice? Name three conditions.
3. What does `Batches: 32` mean on a Hash node, and what does it cost?
4. Why must a non-equality join (`ON a.x < b.y`) use a nested loop?
5. Explain why `NOT IN (subquery)` is often 10–100× slower than `NOT EXISTS`.
6. In a nested loop, the inner node shows `rows=6 loops=702014`. How many rows did it actually produce, and how many index descents did it do?
7. Why does join *order* matter as much as join *method*? Give the principle in one sentence.

---

## Quick reference card

| Algorithm | Cost | Memory | Streaming | Needs |
|---|---|---|---|---|
| **Nested Loop** | outer × inner_cost | none | ✓ | index on inner, or small outer |
| **Hash Join** | build + probe | build side in `work_mem` | ✗ (build blocks) | equality |
| **Merge Join** | sort both + merge | constant | ✓ | both sorted |

**Join types** (semantics) × **algorithms** (physics) are independent: `Hash Anti Join`, `Nested Loop Semi Join`, `Merge Left Join` are all valid.

| Plan text | Means |
|---|---|
| `loops=N` on an inner node | that node ran N times; multiply everything |
| `Batches: > 1` | hash spilled to disk → `work_mem` |
| `Sort Method: external merge  Disk:` | sort spilled → `work_mem` |
| `Memoize  Hits: / Misses:` | inner-side cache rescued a nested loop |
| `Materialize` | inner subtree buffered for rewinding |
| `Hash Anti Join` | from `NOT EXISTS` — good |
| `SubPlan` + `Materialize` under a filter | from `NOT IN` — bad, rewrite |

**Blocking vs streaming**

| Blocking | Streaming |
|---|---|
| Sort, Hash (build), Materialize, ungrouped Aggregate | Seq/Index Scan, Nested Loop, Append, Limit, Merge Join |

**The rules**
1. `loops > 100,000` → wrong join method → check the estimate.
2. `Batches > 1` → `work_mem`, per session.
3. `NOT IN` → always rewrite as `NOT EXISTS`.
4. `LIMIT` above a blocking node buys nothing — restructure.

---

## When would I use this at work?

1. **Any slow multi-table query.** `loops` on the inner nodes tells you in five seconds whether you have an N+1-shaped plan, and the estimate on the driving scan tells you whether the fix is statistics or structure.

2. **Reviewing ORM-generated SQL.** Spotting a `LIMIT` above a `GROUP BY` over a join means the ORM is aggregating millions of rows to return twenty. Restructuring with a CTE that limits first is routinely a 100–1,000× win.

3. **Pushing back on premature denormalisation.** When someone proposes flattening a schema "because joins are slow," you can show that a hash join over 8M rows is 400 ms and that their actual problem is a stale `ANALYZE` — saving a schema change that would have cost years of write complexity.

---

## Connected topics

**Understand before this:** 12 (scan types), 15 (statistics — the input to every decision here), 18 (the planner and cost model).

**This unlocks — all of Phase 3:**
- **20, 21** — ER modelling and translation, now that you know a join's real cost
- **23** — foreign keys, which imply joins
- **53–55** — denormalisation, priced against the join costs measured here
- **66** — the N+1 problem, which is this plan shape generated by an ORM
- **67** — the performance investigation methodology
