# Topic 51 — Optimizer Hints and Interventions
### SQL Mastery Curriculum — Phase 7: Indexes and Query Optimisation

---

## 1. ELI5 — The Analogy That Makes It Click

Imagine you hire a brilliant courier to deliver a package across a city. This courier knows the roads, the traffic patterns, the shortcuts. You could stand over their shoulder and bark turn-by-turn directions — "take Fifth Street, no, turn left here!" — but the whole point of hiring an expert is that they route better than you. Most of the time, when a delivery is slow, it's not because the courier chose a bad road. It's because your *map is out of date*: a bridge that closed last month is still shown as open, so the courier keeps planning routes over it.

In Oracle and MySQL, the database lets you tape directions to the courier's dashboard — "always take Fifth Street." These are called **query hints**, and the courier is obligated to obey them, even when Fifth Street is now flooded.

PostgreSQL took a deliberate, almost stubborn design stance: **it gives you no dashboard to tape directions to.** There is no `/*+ INDEX(orders idx_status) */` you can write into a query and have the planner obey. The PostgreSQL philosophy is: *if the courier is taking a bad route, fix the map, don't override the courier.* Update the statistics. Build the road (index) the courier actually needs. Redraw the neighbourhood boundaries (partition). And in the rare case where you genuinely must override the expert, PostgreSQL hands you a set of blunt instruments — session-level switches that say "you are forbidden from using the highway at all" — which are meant for *diagnosing why the courier avoided the highway*, not for permanent instructions.

This topic is about that whole toolbox: the switches, the map-fixing, the road-building, and the query rewrites that *nudge* the planner without ever *commanding* it. It's the difference between a junior who says "add a hint" and a principal who says "your `n_distinct` estimate is wrong on the `status` column, that's why it's seq-scanning."

---

## 2. Connection to SQL Internals

To intervene in the planner, you have to know what the planner *is*. Remember from Topic 50 (Statistics and the Planner): PostgreSQL uses a **cost-based optimizer**. It does not execute your query as written — it treats your SQL as a *declarative description of the result* and searches a space of physical execution plans for the cheapest one, where "cheapest" is an abstract cost number computed from:

- **`pg_statistic` / `pg_stats`** — the statistics collected by `ANALYZE`: row counts (`reltuples`), per-column distinct-value estimates (`n_distinct`), most-common-values lists (`most_common_vals` / `most_common_freqs`), histograms (`histogram_bounds`), and physical correlation.
- **The cost constants** — `seq_page_cost` (default 1.0), `random_page_cost` (default 4.0), `cpu_tuple_cost` (0.01), `cpu_index_tuple_cost` (0.005), `cpu_operator_cost` (0.0025).
- **`work_mem`** — how much memory a hash table or sort can use before spilling to disk, which changes whether a Hash Join or in-memory Sort even looks affordable.
- **The plan search itself** — a near-exhaustive dynamic-programming search for small joins, switching to the **genetic query optimizer (GEQO)** once the number of join relations exceeds `geqo_threshold` (default 12).

Every intervention in this topic works by changing one of those inputs. The `enable_*` flags don't add a hint — they add a gigantic **`disable_cost`** penalty (historically `1.0e10`) to a node type so the planner picks anything else. Raising `default_statistics_target` makes `ANALYZE` build bigger histograms and MCV lists, which sharpens the cardinality estimates the whole cost model depends on. A **partial index** creates a physical access path that only exists for a subset of rows, so the planner's index-scan cost for that predicate collapses. An **expression index** materialises the statistics for a computed value the planner otherwise has no histogram for. The `OFFSET 0` and CTE tricks manipulate the planner's **subquery-flattening and predicate-pushdown** passes — they build an *optimization fence* that stops the planner from merging a subquery into its parent.

So the mental model is: **you never talk to the executor. You only ever change the numbers the planner minimises over, or you erect walls it isn't allowed to plan across.**

---

## 3. Logical Execution Order Context

Optimizer interventions are unusual for this curriculum because they don't live at one point in the `FROM → JOIN → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → window → ORDER BY → LIMIT` pipeline. They operate one level *above* it — on the **planning stage that happens before any of those clauses execute**.

There are two distinct phases to keep straight:

1. **Planning time** — the planner takes the parsed, rewritten query tree and chooses a physical plan. This is where `enable_seqscan`, `default_statistics_target`, `random_page_cost`, and `pg_hint_plan` all act. Nothing has been read from a heap page yet.
2. **Execution time** — the chosen plan runs, and *now* the logical clause order matters.

That said, the *rewrite* interventions in this topic deliberately manipulate the logical order:

- A **CTE fence** (`WITH x AS (...)` when not inlined) forces the subquery to be fully computed and materialised **before** the outer `WHERE`/`JOIN` can see it — you are pinning a computation earlier in the pipeline than the planner would choose.
- **`OFFSET 0`** on a subquery blocks the planner from pulling the subquery's `WHERE` up into the parent, so the subquery's own `WHERE`/`ORDER BY`/`LIMIT` executes as a self-contained unit.
- **`LATERAL`** changes the `FROM`/`JOIN` phase itself: the right-hand subquery is re-planned and re-executed *per outer row*, which is a per-row `LIMIT`/`ORDER BY` you cannot express any other way.

So while the flags act before the pipeline, the rewrites act *by rearranging where in the pipeline work is allowed to happen*. That distinction — flags change costs, rewrites change fences — is the spine of this entire topic.

---

## 4. What Is an Optimizer Intervention?

An **optimizer intervention** is any deliberate action a developer takes to change the execution plan PostgreSQL chooses, given that PostgreSQL provides **no first-class query-hint syntax**. Interventions fall into three families: (1) *session-level cost switches* that forbid or discourage plan node types, (2) *statistics and schema changes* that correct the planner's cost estimates, and (3) *query rewrites* that erect optimization fences or restructure the logical plan. Only the third-party `pg_hint_plan` extension provides Oracle/MySQL-style directive hints.

```sql
-- Family 1: session cost switches (DIAGNOSTIC — not for production)
SET enable_seqscan = off;
--  │            │      └── boolean: adds ~1e10 "disable_cost" to Seq Scan nodes
--  │            └───────── the plan node type being penalised
--  └────────────────────── session-scoped; reverts on disconnect or RESET

SET enable_hashjoin = off;         -- discourage Hash Join → forces Merge/Nested Loop
SET enable_nestloop = off;         -- discourage Nested Loop
SET enable_indexscan = off;        -- discourage Index Scan → forces Seq Scan/Bitmap
SET enable_material = off;         -- discourage Materialize nodes

-- Family 2: statistics & schema interventions (PRODUCTION-SAFE)
ALTER TABLE orders ALTER COLUMN status SET STATISTICS 1000;
--          │            │           │              └── per-column target: bigger
--          │            │           │                  MCV list + histogram (max 10000)
--          │            └───────────┴───── the column whose estimates are sharpened
--          └──────────────────────────── takes effect on the NEXT ANALYZE
ANALYZE orders;                    -- must re-run to rebuild stats with new target

CREATE INDEX idx_orders_pending ON orders (created_at)
  WHERE status = 'pending';
--       │                        └── partial index predicate: index only these rows
--       └── smaller index, cheaper scan, planner cost for this predicate collapses

CREATE INDEX idx_users_lower_email ON users (lower(email));
--                                          └── expression index: gives the planner
--                                              statistics + an access path for lower(email)

CREATE STATISTICS orders_stat (dependencies, ndistinct)
  ON status, customer_id FROM orders;   -- extended stats for correlated columns

-- Family 3: query-rewrite fences (PRODUCTION-SAFE, subtle)
WITH filtered AS MATERIALIZED (          -- explicit fence: compute once, then join
  SELECT * FROM orders WHERE status = 'shipped'
)
SELECT * FROM filtered f JOIN users u ON u.id = f.customer_id;

SELECT * FROM (
  SELECT * FROM big_table WHERE tenant_id = $1 OFFSET 0   -- OFFSET 0 = optimization fence
) t WHERE t.score > 90;

SELECT u.id, recent.*                     -- LATERAL: per-row subquery (top-N-per-group)
FROM users u
CROSS JOIN LATERAL (
  SELECT * FROM orders o WHERE o.customer_id = u.id
  ORDER BY o.created_at DESC LIMIT 3
) recent;

-- The one true "hint" system (third-party extension)
/*+ IndexScan(orders idx_orders_status) */
SELECT * FROM orders WHERE status = 'pending';   -- requires pg_hint_plan loaded
```

The critical mental distinction: **Family 1 forbids, Family 2 informs, Family 3 fences.** A principal engineer reaches for Family 2 first (fix the map), Family 3 when a rewrite is cleaner than a schema change, and Family 1 *only* inside a diagnostic session to prove *why* the planner chose what it chose.

---

## 5. Why Optimizer Intervention Mastery Matters in Production

1. **The "why won't it use my index?" incident.** The single most common production SQL escalation is a query that has a perfectly good index which the planner refuses to use, falling back to a Seq Scan that takes 40 seconds. A junior's instinct — reach for a hint — *doesn't exist in PostgreSQL*. The engineer who understands interventions knows to check the row estimate first (`EXPLAIN` estimated vs actual), because 90% of the time the planner seq-scans because it *thinks* the predicate matches half the table when it really matches 0.1%. The fix is `ALTER COLUMN ... SET STATISTICS` + `ANALYZE`, not a hint.

2. **Session flags leak into production and cause outages.** `SET enable_seqscan = off` in a connection-pooled application is genuinely dangerous: connection poolers (PgBouncer in session mode, or `pg`'s `Pool`) reuse physical connections, so a `SET` executed on one logical request *persists* onto the next request that borrows that connection. A debugging flag left on can silently degrade every query that reuses that connection. Knowing this prevents a class of "works on my machine, mysteriously slow in prod" bugs.

3. **CTE materialisation changed behaviour in PostgreSQL 12.** Before v12, *every* CTE was an optimization fence (always materialised). From v12 on, CTEs are **inlined by default** when referenced once and side-effect-free. Teams that upgraded and relied on the old fence behaviour — using a CTE to force a good plan — silently got different, sometimes worse, plans. Mastery means knowing when to write `AS MATERIALIZED` to restore the fence and when to write `AS NOT MATERIALIZED` to force inlining.

4. **Top-N-per-group without LATERAL is a full-table sort.** "Get the 3 most recent orders per customer" written with window functions sorts the *entire* orders table; written with `LATERAL` + an index it does one small index scan per customer. On a 100M-row table that's the difference between 90 seconds and 90 milliseconds. This is a pure query-rewrite intervention with no schema change.

5. **`random_page_cost` is wrong by default on SSDs.** The default `random_page_cost = 4.0` encodes the seek penalty of spinning disks. On SSDs and cloud block storage, random reads are nearly as cheap as sequential ones, so the *correct* value is closer to `1.1`. A too-high `random_page_cost` systematically biases the planner *away* from index scans across your entire database — a single global-tuning intervention can fix a whole class of "won't use the index" problems at once.

---

## 6. Deep Technical Content

### 6.1 Why PostgreSQL Has No Hints — The Design Philosophy

This is a genuine interview topic, not trivia. The PostgreSQL core team has repeatedly and deliberately rejected first-class query hints. The stated reasoning:

- **Hints rot.** A hint like "use index X" is correct for today's data distribution. Six months later the table has grown, the distribution has shifted, and the hint now forces a bad plan — but the hint is buried in application code that nobody revisits. The cost-based planner, by contrast, *re-decides every time it plans*.
- **Hints hide bugs.** If a query needs a hint, something is usually wrong with the statistics or the schema. A hint papers over the real problem (stale stats, missing index, bad cost constant) so it never gets fixed.
- **Hints are a support burden.** Once hints exist, users apply them everywhere, and every "PostgreSQL is slow" report becomes "well, you hinted it into a corner."

The counter-argument — and it's a legitimate one — is that sometimes you *know* more than the planner (e.g., a correlation the statistics can't capture), and in those cases the lack of an escape hatch is genuinely painful. This is exactly why the third-party `pg_hint_plan` extension exists and is widely used at scale (notably by NTT, its author). But it is *not* in core, and it never will be. A principal engineer states this trade-off explicitly in interviews rather than complaining about it.

### 6.2 The `enable_*` Family — How They Actually Work

There are roughly twenty `enable_*` GUCs. The important ones:

| Flag | Default | Discourages | Common diagnostic use |
|------|---------|-------------|----------------------|
| `enable_seqscan` | on | Seq Scan | Prove an index *could* be used |
| `enable_indexscan` | on | Index Scan | Force Seq/Bitmap to compare cost |
| `enable_indexonlyscan` | on | Index-Only Scan | Test visibility-map impact |
| `enable_bitmapscan` | on | Bitmap Heap/Index Scan | Isolate bitmap behaviour |
| `enable_hashjoin` | on | Hash Join | Force Merge/Nested Loop |
| `enable_mergejoin` | on | Merge Join | Force Hash/Nested Loop |
| `enable_nestloop` | on | Nested Loop | Force Hash/Merge |
| `enable_material` | on | Materialize | Test re-scan cost |
| `enable_hashagg` | on | HashAggregate | Force GroupAggregate (sorted) |
| `enable_sort` | on | explicit Sort | Force index-provided ordering |
| `enable_partitionwise_join` | off | — | *Enable* per-partition joins |

The crucial mechanical detail: **these do not remove the plan type. They add `disable_cost` (a very large constant) to it.** This has two consequences:

1. If there is *no other way* to run the query, the "disabled" node is still used — you'll see it in the plan, and modern PostgreSQL (v14+) marks it with `Disabled: true` in `EXPLAIN`. Turning off `enable_seqscan` on a table with no indexes still produces a Seq Scan.
2. Because it's an additive penalty, not a hard ban, on very large plans the disable cost can occasionally be "outweighed" in bizarre ways. Treat them as strong discouragement, not guarantees.

They are `SET`-table at session, transaction (`SET LOCAL`), role, and database level. **Never** set them globally in `postgresql.conf` for a production system.

### 6.3 The Right Way to Use the Flags — Diagnosis, Not Cure

The legitimate workflow:

```sql
-- Step 1: the query is slow, EXPLAIN shows a Seq Scan you didn't expect
EXPLAIN ANALYZE SELECT * FROM orders WHERE status = 'pending';
-- Seq Scan ... rows=2,000,000 ... actual rows=1,203

-- Step 2: prove the planner CAN use the index, and see what it would cost
SET LOCAL enable_seqscan = off;   -- SET LOCAL = auto-reverts at transaction end
EXPLAIN ANALYZE SELECT * FROM orders WHERE status = 'pending';
-- Bitmap Index Scan ... actual rows=1,203 ... 40x faster

-- Step 3: The index IS better but the planner didn't pick it.
--         That means its COST ESTIMATE was wrong. The estimate said 2,000,000
--         rows; reality is 1,203. Fix the STATISTICS, not the plan:
ALTER TABLE orders ALTER COLUMN status SET STATISTICS 1000;
ANALYZE orders;

-- Step 4: RESET the flag and confirm the planner now chooses the index ON ITS OWN
RESET enable_seqscan;
EXPLAIN ANALYZE SELECT * FROM orders WHERE status = 'pending';
-- Bitmap Index Scan ... chosen without any flag. Fixed at the root cause.
```

The flag was the *stethoscope*, not the *medicine*. The medicine was the statistics change. Using `SET LOCAL` inside an explicit transaction (`BEGIN; ... COMMIT/ROLLBACK`) guarantees the flag can't leak out of your diagnostic session.

### 6.4 Statistics Targets — Sharpening the Planner's Vision

The `default_statistics_target` (default 100) controls how many entries `ANALYZE` puts in the most-common-values list and how many buckets go in the histogram, *per column*. For a column with a skewed distribution — like `status`, where 99% of rows are `'completed'` and the interesting ones are `'pending'`/`'failed'` — the default 100 may not capture the rare values in the MCV list, so the planner assumes they're uniformly distributed and overestimates their row count.

```sql
-- Global default (affects all future ANALYZE on all columns)
SET default_statistics_target = 250;   -- session; or set in postgresql.conf

-- Per-column override (the surgical tool — preferred)
ALTER TABLE orders ALTER COLUMN status SET STATISTICS 1000;
ALTER TABLE orders ALTER COLUMN customer_id SET STATISTICS 500;
ANALYZE orders;   -- REQUIRED: the target change is inert until ANALYZE re-runs

-- Inspect what the planner now knows
SELECT attname, n_distinct, array_length(most_common_vals, 1) AS mcv_count,
       array_length(histogram_bounds, 1) AS histogram_buckets
FROM pg_stats
WHERE tablename = 'orders' AND attname IN ('status', 'customer_id');
```

Trade-offs: a higher target means `ANALYZE` reads more sample rows (sample size ≈ `300 × target`) and takes longer, `pg_statistic` grows, and *planning time itself* increases slightly because the planner scans longer MCV lists. The maximum is 10000. Reserve high targets for columns that are (a) used in `WHERE`/`JOIN` and (b) skewed. Never blanket-raise the global to 10000.

### 6.5 Extended Statistics — When Columns Are Correlated

The planner assumes columns are **statistically independent** by default. So for `WHERE city = 'Seattle' AND state = 'WA'`, it multiplies the two selectivities — even though `state` is functionally determined by `city`, so the combined selectivity should equal the `city` selectivity alone. This underestimates rows badly and picks wrong plans.

```sql
CREATE STATISTICS orders_customer_status (dependencies, ndistinct, mcv)
  ON customer_id, status FROM orders;
ANALYZE orders;
--  dependencies → functional-dependency degree between the columns
--  ndistinct    → true distinct count of the COMBINATION
--  mcv          → most-common combined value pairs
```

This is a statistics intervention with no query change and no index — pure planner education. It's the correct fix for the classic "the estimate is off by 100x on a two-column filter" problem.

### 6.6 Partial Indexes — Making a Predicate Cheap

A **partial index** indexes only rows matching a `WHERE` predicate. It's smaller (faster to scan, cheaper to maintain) and — critically — the planner will use it *only* when it can prove the query predicate implies the index predicate.

```sql
-- 50M orders, but only ~5,000 are 'pending' at any time (a work queue)
CREATE INDEX idx_orders_pending ON orders (created_at)
  WHERE status = 'pending';

-- USES the partial index (query predicate implies index predicate)
SELECT * FROM orders WHERE status = 'pending' ORDER BY created_at LIMIT 100;

-- Does NOT use it — planner can't prove status='pending'
SELECT * FROM orders WHERE status = $1;   -- $1 unknown at plan time
```

The intervention value: instead of a 50M-entry full index on `status`, you have a 5,000-entry index that is almost always in cache. This *changes the cost the planner computes* for the pending-queue query so dramatically that it will always be chosen. Partial indexes are one of PostgreSQL's most powerful and underused features — a way to make a specific hot query fast without a hint.

### 6.7 Expression Indexes — Giving the Planner Statistics on Computed Values

If you filter on `lower(email)`, a plain index on `email` is useless — the planner has no access path and no statistics for the *expression*. An expression index provides both.

```sql
CREATE INDEX idx_users_lower_email ON users (lower(email));
ANALYZE users;   -- also collects statistics ON the expression

-- Now this uses the index AND gets a real row estimate:
SELECT * FROM users WHERE lower(email) = 'ada@example.com';
```

The query must reference the expression *exactly* as indexed. This is an intervention because it creates a statistics + access path where none existed — the planner literally could not estimate `lower(email)` selectivity before, and now it can.

### 6.8 The CTE Fence — v12's Behavioural Change

Pre-PostgreSQL 12: every `WITH` CTE was materialised (executed once, result stored, then referenced) — an **unconditional optimization fence**. This was famous as "the CTE trick" for forcing a plan. PostgreSQL 12 changed the default: a CTE that is (a) referenced exactly once, (b) side-effect-free (no `INSERT`/`UPDATE`/`DELETE`), and (c) not recursive, is now **inlined** into the parent query — meaning the planner can push predicates through it and optimise across the boundary.

```sql
-- v12+ default: this CTE is INLINED — predicate pushdown happens,
-- the WHERE created_at filter can reach into the CTE's scan
WITH recent_orders AS (
  SELECT * FROM orders WHERE status = 'shipped'
)
SELECT * FROM recent_orders WHERE created_at > now() - interval '7 days';

-- Force the OLD fence behaviour (materialise once, no pushdown):
WITH recent_orders AS MATERIALIZED (
  SELECT * FROM orders WHERE status = 'shipped'
)
SELECT * FROM recent_orders WHERE created_at > now() - interval '7 days';

-- Force inlining even when referenced multiple times:
WITH recent_orders AS NOT MATERIALIZED (
  SELECT * FROM orders WHERE status = 'shipped'
)
SELECT ... FROM recent_orders a JOIN recent_orders b ...;
```

When is the fence (`MATERIALIZED`) the *right* intervention?
- When the CTE is **expensive and referenced multiple times** — materialise once instead of recomputing per reference.
- When the CTE contains a **volatile function** you want evaluated exactly once.
- When inlining causes the planner to make a **bad choice** (e.g., pushing a predicate that defeats an index) and you want to pin the plan.

When is inlining (`NOT MATERIALIZED`) right? When a legacy CTE was only ever used for readability and the fence is now *preventing* a good predicate pushdown.

### 6.9 The `OFFSET 0` and `LIMIT` Fences

`OFFSET 0` is a legendary PostgreSQL trick. Because `OFFSET` *could* change results, the planner is forbidden from flattening a subquery that has an `OFFSET` into its parent — so `OFFSET 0`, which changes nothing semantically, acts as a pure optimization fence.

```sql
-- Problem: the planner flattens the subquery and applies expensive_fn()
-- to rows BEFORE the cheap tenant_id filter, evaluating it millions of times
SELECT * FROM (
  SELECT *, expensive_fn(payload) AS score
  FROM events
  WHERE tenant_id = $1
  OFFSET 0                       -- fence: force tenant_id filter FIRST,
) e                              -- so expensive_fn runs only on the survivors
WHERE score > 0.9;
```

It's a hack — brittle, non-obvious, and a reviewer will ask "why is there an OFFSET 0 here?" Prefer a `MATERIALIZED` CTE for the same effect with clearer intent. But you must *recognise* `OFFSET 0` in existing code and know exactly what it does.

### 6.10 LATERAL — The Per-Row Rewrite

`LATERAL` lets a subquery in the `FROM` clause reference columns from tables listed before it, and it is re-evaluated *for each outer row*. This is the canonical, index-friendly way to do **top-N-per-group**.

```sql
-- Top 3 most recent orders per active user — index-driven, no giant sort
SELECT u.id, u.name, o.id AS order_id, o.total_amount, o.created_at
FROM users u
CROSS JOIN LATERAL (
  SELECT id, total_amount, created_at
  FROM orders
  WHERE customer_id = u.id      -- references the OUTER row — that's LATERAL
  ORDER BY created_at DESC
  LIMIT 3
) o
WHERE u.status = 'active';
-- With an index on orders(customer_id, created_at DESC), each lateral
-- subquery is a 3-row index scan. Contrast: a ROW_NUMBER() window function
-- sorts the ENTIRE orders table first.
```

Use `LEFT JOIN LATERAL (...) ON true` instead of `CROSS JOIN LATERAL` if you want to keep outer rows that have no matching inner rows (users with zero orders). This is a pure query-rewrite intervention: same result set as a window function, radically different and usually far better plan.

### 6.11 The `pg_hint_plan` Extension — Real Hints, If You Must

`pg_hint_plan` is a third-party extension (not in core) that parses hints from a leading block comment and injects them into the planner. It gives you the Oracle-style directives PostgreSQL core refuses to.

```sql
-- After: CREATE EXTENSION pg_hint_plan; and loading it into the session
/*+ SeqScan(orders) */
SELECT * FROM orders WHERE status = 'pending';

/*+ IndexScan(orders idx_orders_status) */
SELECT * FROM orders WHERE status = 'pending';

/*+ HashJoin(o u) Leading((o u)) */
SELECT * FROM orders o JOIN users u ON u.id = o.customer_id;

/*+ Rows(o u *10) */               -- correct a cardinality estimate directly
SELECT * FROM orders o JOIN users u ON u.id = o.customer_id;
```

Hint scopes include scan-method hints (`SeqScan`, `IndexScan`, `BitmapScan`, `NoSeqScan`), join-method hints (`HashJoin`, `NestLoop`, `MergeJoin`), join-order hints (`Leading`), and — uniquely powerful — `Rows()` to *correct an estimate* rather than force a method. The `Rows()` hint is philosophically the "best" hint because it fixes the planner's input rather than overriding its output.

**When is `pg_hint_plan` justified?** When you have a genuinely un-fixable estimate (a correlation no statistics can capture), a critical query that must have a stable plan, and you accept the maintenance burden. It requires installing the extension (not available on every managed platform — check RDS/Cloud SQL support), loading it via `shared_preload_libraries` or `session_preload_libraries`, and it's disabled by default. It is a last resort, and stating that in an interview signals maturity.

### 6.12 Global Cost Constants — The Whole-Database Intervention

Some interventions aren't per-query at all — they retune the cost model globally.

```sql
-- SSD / cloud storage: random reads are cheap; lower the penalty
--   so the planner stops avoiding index scans database-wide
ALTER SYSTEM SET random_page_cost = 1.1;   -- default 4.0 (spinning-disk era)

-- Tell the planner how much of the DB fits in RAM (OS cache + shared_buffers)
ALTER SYSTEM SET effective_cache_size = '24GB';   -- ~50-75% of server RAM

-- Bigger sorts/hashes stay in memory instead of spilling → Hash Join & Sort
--   become cheaper, planner picks them more readily. Per-operation, per-connection!
SET work_mem = '256MB';                    -- session or per-transaction

SELECT pg_reload_conf();                   -- apply ALTER SYSTEM changes
```

`random_page_cost` is the single highest-leverage global intervention on modern hardware — leaving it at 4.0 on an SSD systematically biases against every index in the database. `work_mem` is per-sort/per-hash *and* per-connection, so setting it high globally on a 500-connection server can OOM the box; set it per-session for heavy analytical queries instead.

---

## 7. EXPLAIN — Interventions in the Plan

### 7.1 Seeing a Disabled Node

When you set an `enable_*` flag off but the planner has no alternative, PostgreSQL v14+ shows it explicitly:

```sql
SET enable_seqscan = off;
EXPLAIN SELECT * FROM audit_logs;   -- no index that covers a full scan
```

```
Seq Scan on audit_logs  (cost=10000000000.00..10000000180.00 rows=8000 width=120)
  Disabled: true
```

**Reading it**: the cost begins at `1e10` — that's the `disable_cost` penalty added to the node. `Disabled: true` (v16+ formatting) confirms the node was penalised but used anyway because no alternative exists. This is your proof that turning off the flag did *not* magically produce an index — there simply isn't one.

### 7.2 Before/After: The Statistics Fix

The core diagnostic pattern. Before raising the statistics target:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE status = 'pending';
```

```
Seq Scan on orders  (cost=0.00..185000.00 rows=1000000 width=140)
                    (actual time=0.03..842.11 rows=1203 loops=1)
  Filter: (status = 'pending')
  Rows Removed by Filter: 4998797
  Buffers: shared hit=1240 read=83760
Planning Time: 0.15 ms
Execution Time: 842.55 ms
```

**Reading it**: the planner *estimated* `rows=1000000` but the *actual* was `rows=1203` — an 830x overestimate. That wrong estimate is *why* it chose a Seq Scan: it believed the index would touch a fifth of the table. `Rows Removed by Filter: 4998797` and `read=83760` (heap pages read from disk) confirm the full-table scan.

After `ALTER TABLE orders ALTER COLUMN status SET STATISTICS 1000; ANALYZE orders;`:

```
Bitmap Heap Scan on orders  (cost=28.55..4820.10 rows=1150 width=140)
                            (actual time=0.42..2.10 rows=1203 loops=1)
  Recheck Cond: (status = 'pending')
  Heap Blocks: exact=1180
  Buffers: shared hit=1195
  ->  Bitmap Index Scan on idx_orders_status  (cost=0.00..28.26 rows=1150 width=0)
                                              (actual time=0.29..0.29 rows=1203 loops=1)
        Index Cond: (status = 'pending')
Planning Time: 0.20 ms
Execution Time: 2.44 ms
```

**Reading it**: estimate `rows=1150` now matches actual `rows=1203` — the MCV list captured the rare value. With a correct estimate, the index path's cost (`4820`) beats the Seq Scan (`185000`), so the planner chooses it *without any flag*. Execution dropped from 842ms to 2.4ms. **The plan changed because the estimate changed** — that's the entire thesis of this topic.

### 7.3 The CTE Fence in the Plan

An inlined CTE (v12+ default) shows no CTE node — the subquery scan is fused into the parent:

```
Bitmap Heap Scan on orders  (cost=... rows=... )
  Recheck Cond: ((status = 'shipped') AND (created_at > ...))   ← predicates fused
```

A `MATERIALIZED` CTE shows a distinct `CTE Scan` reading from a materialised result:

```
CTE Scan on recent_orders  (cost=... rows=50000 ...)
  Filter: (created_at > ...)                    ← filter applied AFTER materialisation
  CTE recent_orders
    ->  Seq Scan on orders  (cost=... rows=50000 ...)
          Filter: (status = 'shipped')          ← the created_at filter did NOT push in
```

**Reading it**: in the materialised plan, the `Seq Scan` inside the CTE only knows `status = 'shipped'` — the `created_at` filter is stuck *outside*, applied to every materialised row. The fence blocked predicate pushdown. That's sometimes what you want (stable plan) and sometimes a performance bug (missed index).

### 7.4 LATERAL in the Plan

```
Nested Loop  (cost=0.43..8450.00 rows=300 width=48)
             (actual time=0.05..12.40 rows=287 loops=1)
  ->  Seq Scan on users u  (actual rows=100 loops=1)
        Filter: (status = 'active')
  ->  Limit  (actual rows=3 loops=100)                    ← re-runs per outer row
        ->  Index Scan using idx_orders_cust_created on orders
              (actual rows=3 loops=100)
              Index Cond: (customer_id = u.id)             ← the LATERAL correlation
```

**Reading it**: `loops=100` on the inner `Limit` means the lateral subquery ran once per active user, each time an index scan returning exactly 3 rows. No sort of the full orders table appears anywhere. Compare a `ROW_NUMBER()` version, which would show a `WindowAgg` over a full `Sort` of every order — orders of magnitude more work.

### 7.5 What to Look For

| Symptom in EXPLAIN | What it tells you | Intervention |
|--------------------|-------------------|--------------|
| Estimated rows ≫ actual rows | Stale/coarse statistics | `SET STATISTICS` + `ANALYZE` |
| `cost=10000000000..` + `Disabled: true` | An `enable_*` flag is off | `RESET` the flag |
| Big estimate error on 2-column `AND` | Column correlation | `CREATE STATISTICS` |
| Seq Scan despite a matching index | Wrong row estimate OR high `random_page_cost` | Fix stats, or lower `random_page_cost` |
| `CTE Scan` + filter not pushed in | CTE fence blocking pushdown | `AS NOT MATERIALIZED` |
| `WindowAgg` over full `Sort` for top-N | Window-function top-N-per-group | Rewrite with `LATERAL` |
| Hash Join `Batches > 1` | `work_mem` too small, spilling | Raise `work_mem` for the session |

---

## 8. Query Examples

### Example 1 — Basic: Diagnosing With a Session Flag

```sql
-- The query seq-scans and you suspect an index would help.
-- Use SET LOCAL inside a transaction so the flag CANNOT leak.
BEGIN;
  SET LOCAL enable_seqscan = off;      -- diagnostic only, auto-reverts at COMMIT
  EXPLAIN (ANALYZE, BUFFERS)
  SELECT id, total_amount, created_at
  FROM orders
  WHERE status = 'refunded';           -- rare status
COMMIT;                                 -- flag reverts here; nothing leaks
-- If the forced index plan is much faster, the real fix is statistics,
-- not leaving this flag on.
```

### Example 2 — Intermediate: Partial + Expression Index Together

```sql
-- Case-insensitive lookup of ACTIVE users only.
-- A partial expression index: smallest possible, exactly targets the hot query.
CREATE INDEX idx_users_active_lower_email
  ON users (lower(email))
  WHERE status = 'active';

ANALYZE users;

-- Uses the index: predicate implies status='active' AND references lower(email)
SELECT id, name, email
FROM users
WHERE lower(email) = 'ada@example.com'
  AND status = 'active';
-- The index has entries ONLY for active users, keyed by lower(email).
-- On 10M users with 2M active, this is a ~2M-entry index vs a 10M-entry one,
-- and it gives the planner real selectivity statistics for lower(email).
```

### Example 3 — Production Grade: Top-N-Per-Group Rewrite

```sql
-- CONTEXT:
--   users:  5,000,000 rows
--   orders: 120,000,000 rows, index idx_orders_cust_created (customer_id, created_at DESC)
-- GOAL: the 3 most recent orders for each of the ~40,000 users active this week.
-- PERF EXPECTATION: ~40,000 tiny index scans of 3 rows each; sub-second.
--   A ROW_NUMBER() approach would sort all 120M orders — minutes.

SELECT u.id AS user_id, u.name,
       o.id AS order_id, o.total_amount, o.created_at
FROM users u
CROSS JOIN LATERAL (
  SELECT id, total_amount, created_at
  FROM orders
  WHERE customer_id = u.id
  ORDER BY created_at DESC
  LIMIT 3
) o
WHERE u.last_active_at > now() - interval '7 days'
ORDER BY u.id, o.created_at DESC;
```

```
-- EXPLAIN (ANALYZE) shape:
Nested Loop  (cost=0.57..412300.00 rows=120000 width=56)
             (actual time=0.06..310.42 rows=118442 loops=1)
  ->  Index Scan using idx_users_last_active on users u
        (actual rows=40000 loops=1)
        Index Cond: (last_active_at > (now() - '7 days'::interval))
  ->  Limit  (actual rows=3 loops=40000)                       ← 40k small scans
        ->  Index Scan using idx_orders_cust_created on orders
              (actual rows=3 loops=40000)
              Index Cond: (customer_id = u.id)
Planning Time: 0.30 ms
Execution Time: 361.88 ms
-- 120M-row table, sub-second, because we never sorted it — we index-probed it
-- 40,000 times for 3 rows each. This is the intervention: a pure rewrite.
```

---

## 9. Wrong → Right Patterns

### Wrong 1: Reaching for a Hint That Doesn't Exist

```sql
-- WRONG: copied from an Oracle/MySQL codebase — this is NOT valid core PostgreSQL
SELECT /*+ INDEX(orders idx_orders_status) */ *
FROM orders WHERE status = 'pending';
-- EXACT RESULT: the comment is IGNORED (it's just a comment to core PostgreSQL).
-- The planner still seq-scans. Developers waste hours thinking the hint "isn't working"
-- when in fact core PostgreSQL never parses hint comments at all.
```

**Why it's wrong at the execution level**: PostgreSQL's parser treats `/*+ ... */` as an ordinary comment and discards it before planning. There is no code path that reads directive hints unless the `pg_hint_plan` extension is installed *and* loaded into the session. The plan is chosen purely from cost.

```sql
-- RIGHT: fix the estimate so the planner chooses the index on its own
ALTER TABLE orders ALTER COLUMN status SET STATISTICS 1000;
ANALYZE orders;
SELECT * FROM orders WHERE status = 'pending';   -- now uses the index by cost
-- OR, if you genuinely need directive hints, install pg_hint_plan first:
--   CREATE EXTENSION pg_hint_plan;  (and load it) — only then does the comment work.
```

### Wrong 2: Leaving a Session Flag On in a Pooled Connection

```sql
-- WRONG: setting a plain SET in application code that uses a connection pool
await pool.query("SET enable_seqscan = off");   -- "to make sure it uses the index"
await pool.query("SELECT * FROM orders WHERE status = 'pending'");
-- EXACT PROBLEM: the pool reuses this physical connection for UNRELATED later queries.
-- Every subsequent query borrowing this connection now has seqscan disabled —
-- including small-table queries where a Seq Scan was optimal. Random queries across
-- the app mysteriously slow down, and nothing in THEIR code explains why.
```

**Why it's wrong at the execution level**: `SET` without `LOCAL` is *session-scoped*, and a pooled physical connection is a long-lived session shared across many logical requests. The flag persists until the connection is reset or the pooler recycles it. This is a genuine production outage pattern.

```sql
-- RIGHT: never disable seqscan in app code. If you MUST scope a planner GUC,
-- do it per-transaction with SET LOCAL so it cannot outlive the request:
await pool.query("BEGIN");
await pool.query("SET LOCAL work_mem = '256MB'");   -- reverts at COMMIT
await pool.query("SELECT ... /* heavy analytical query */");
await pool.query("COMMIT");
-- Better still: fix statistics/indexes so no per-query GUC is needed at all.
```

### Wrong 3: Assuming a CTE Still Fences (Post-v12 Regression)

```sql
-- WRONG: written for PostgreSQL 11, relied on the CTE materialising to force a plan
WITH candidates AS (
  SELECT * FROM orders WHERE customer_id = $1     -- assumed: computed once, fenced
)
SELECT * FROM candidates WHERE total_amount > 500;
-- EXACT PROBLEM: on v12+ this CTE is INLINED. The planner fuses both predicates and
-- may now pick a completely different (and occasionally worse) plan than it did on v11.
-- Teams that upgraded saw plans silently change with no query edit.
```

**Why it's wrong at the execution level**: from v12, a single-reference, side-effect-free CTE is inlined, so predicate pushdown and subquery flattening happen across the old boundary. The optimization fence the code depended on is gone.

```sql
-- RIGHT: state your intent explicitly rather than relying on version defaults
WITH candidates AS MATERIALIZED (        -- restore the fence deliberately
  SELECT * FROM orders WHERE customer_id = $1
)
SELECT * FROM candidates WHERE total_amount > 500;
-- If instead you WANT the inlining, write AS NOT MATERIALIZED and delete the
-- assumption from your mental model. Either way: be explicit, don't rely on the default.
```

### Wrong 4: Top-N-Per-Group That Sorts the Whole Table

```sql
-- WRONG: window function to get 3 latest orders per customer on a 120M-row table
SELECT * FROM (
  SELECT o.*,
         ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY created_at DESC) AS rn
  FROM orders o
) ranked
WHERE rn <= 3;
-- EXACT PROBLEM: the WindowAgg must sort/partition ALL 120M rows before the rn<=3
-- filter can run. EXPLAIN shows a full Sort of the entire table — minutes, huge temp
-- files spilling to disk, even though you only want 3 rows per customer.
```

**Why it's wrong at the execution level**: a window function's `WHERE rn <= 3` filter is applied *after* the window is fully computed, and the window computation requires ordering every row within every partition — there is no early cutoff. The whole table is sorted.

```sql
-- RIGHT: LATERAL turns it into one tiny index scan per customer
SELECT u.id, o.*
FROM users u
CROSS JOIN LATERAL (
  SELECT * FROM orders WHERE customer_id = u.id
  ORDER BY created_at DESC LIMIT 3
) o;
-- With idx_orders(customer_id, created_at DESC): 3 rows per user, no global sort.
```

### Wrong 5: Global `work_mem` Set High to "Speed Up Sorts"

```sql
-- WRONG: in postgresql.conf on a 500-connection server
-- work_mem = 1GB
-- INTENT: "sorts and hashes should never spill to disk"
-- EXACT PROBLEM: work_mem is per-sort/per-hash-node PER CONNECTION. A single complex
-- query can use several work_mem allocations at once. 500 connections × multiple
-- allocations × 1GB can demand hundreds of GB — the OOM killer terminates the postmaster.
```

**Why it's wrong at the execution level**: `work_mem` is not a global pool; it's a per-operation limit. Total memory can be `work_mem × (concurrent sort/hash nodes) × (active connections)`. Setting it high globally is a latent OOM.

```sql
-- RIGHT: keep the global small; raise it per-session only for known-heavy queries
-- postgresql.conf: work_mem = 16MB   (safe global)
BEGIN;
  SET LOCAL work_mem = '512MB';   -- just this transaction, just this heavy report
  SELECT ... /* big analytical aggregation */;
COMMIT;
```

---

## 10. Performance Profile

### 10.1 Cost of Each Intervention

| Intervention | Planning cost | Runtime cost | Maintenance cost |
|--------------|--------------|--------------|------------------|
| `enable_*` flag | negligible | none (changes plan only) | dangerous if leaked |
| `SET STATISTICS` higher | slightly higher planning (longer MCV scan) | none | bigger `pg_statistic`, slower `ANALYZE` |
| Extended statistics | slightly higher planning | none | recomputed each `ANALYZE` |
| Partial index | none | faster hot query | tiny — indexes few rows |
| Expression index | none | faster + real estimates | full index maintenance on write |
| CTE `MATERIALIZED` | none | can be slower (no pushdown) or faster (no recompute) | none |
| `LATERAL` rewrite | none | usually far faster for top-N | none |
| `pg_hint_plan` | slight parse overhead | fixes plan | high — extension + brittle hints |
| `random_page_cost` global | none | shifts plan choices DB-wide | one-time tuning |

### 10.2 Statistics Target — Scaling Behaviour

`ANALYZE` samples approximately `300 × statistics_target` rows. At the default 100 that's ~30,000 rows; at 1000 it's ~300,000; at the max 10,000 it's ~3,000,000 sampled rows.

| Rows in table | `SET STATISTICS 100` ANALYZE | `SET STATISTICS 1000` ANALYZE | Planning-time impact |
|---------------|------------------------------|-------------------------------|----------------------|
| 1M | ~200 ms | ~1.5 s | +micro-seconds |
| 10M | ~400 ms | ~2.5 s | +tens of micro-seconds |
| 100M | ~1 s | ~5 s | measurable if many columns raised |

The runtime *query* benefit is a step change (right plan vs wrong plan), so the amortised trade is almost always worth it for skewed `WHERE`/`JOIN` columns. The cost is longer `ANALYZE` and marginally longer planning — both usually irrelevant next to picking the correct plan.

### 10.3 Partial Index Scaling — The Standout Win

| Total rows | Rows matching partial predicate | Full index size | Partial index size | Scan/cache benefit |
|------------|--------------------------------|-----------------|--------------------|--------------------|
| 1M | 5,000 (0.5%) | ~21 MB | ~120 KB | fits entirely in cache |
| 10M | 50,000 (0.5%) | ~214 MB | ~1.2 MB | fits in cache |
| 100M | 500,000 (0.5%) | ~2.1 GB | ~12 MB | fits in cache vs likely-evicted |

For a hot "work queue" query (`WHERE status = 'pending'`), the partial index is often 100–1000x smaller than a full index, so it stays resident in shared buffers and the query is essentially always index-cache-hot. This is the highest-leverage schema intervention in the topic.

### 10.4 LATERAL vs Window Function at Scale

| Orders table size | `ROW_NUMBER()` window (full sort) | `LATERAL` + index (per-group) |
|-------------------|-----------------------------------|-------------------------------|
| 1M | ~1.2 s (sorts 1M rows) | ~15 ms |
| 10M | ~14 s + temp spill | ~40 ms |
| 100M | minutes, large temp files | ~350 ms (40k probes) |

The window function's cost scales with the *whole table*; LATERAL's cost scales with the *number of groups × N*. When groups ≪ rows (few users, many orders each), LATERAL wins by orders of magnitude. When N (the per-group limit) approaches the group size, the gap narrows.

### 10.5 The `random_page_cost` Lever

On SSD/cloud storage, lowering `random_page_cost` from 4.0 toward 1.1 makes every index scan look cheaper relative to sequential scans. Effect is database-wide and immediate (next plan). It does not change any data — only which plans win. Combine with a realistic `effective_cache_size` (50–75% of RAM) so the planner knows most index pages are cached; together these two constants correct the systemic anti-index bias inherited from the spinning-disk defaults.

---

## 11. Node.js Integration

### 11.1 Per-Transaction GUC With `SET LOCAL` (the safe pattern)

```javascript
import pg from 'pg';
const { Pool } = pg;
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

// Raise work_mem for ONE heavy report without leaking into the pooled connection.
async function heavyMonthlyReport(month) {
  const client = await pool.connect();     // take a dedicated connection
  try {
    await client.query('BEGIN');
    await client.query("SET LOCAL work_mem = '256MB'");  // reverts at COMMIT/ROLLBACK
    const { rows } = await client.query(
      `SELECT category_id, SUM(total_amount) AS revenue
       FROM orders
       WHERE created_at >= $1 AND created_at < ($1::date + INTERVAL '1 month')
       GROUP BY category_id
       ORDER BY revenue DESC`,
      [month]
    );
    await client.query('COMMIT');
    return rows;
  } catch (err) {
    await client.query('ROLLBACK');        // SET LOCAL reverts here too
    throw err;
  } finally {
    client.release();                      // safe to return to pool — no leaked GUC
  }
}
```

The rule: **any planner GUC you set from Node must be `SET LOCAL` inside an explicit transaction on a checked-out client.** A bare `pool.query("SET ...")` sets it on whatever physical connection the pool hands you, and that connection goes back into rotation with the flag still on.

### 11.2 The LATERAL Top-N Query

```javascript
async function recentOrdersPerActiveUser(days = 7, limitPerUser = 3) {
  const { rows } = await pool.query(
    `SELECT u.id AS user_id, u.name,
            o.id AS order_id, o.total_amount, o.created_at
     FROM users u
     CROSS JOIN LATERAL (
       SELECT id, total_amount, created_at
       FROM orders
       WHERE customer_id = u.id
       ORDER BY created_at DESC
       LIMIT $2
     ) o
     WHERE u.last_active_at > NOW() - ($1 || ' days')::INTERVAL
     ORDER BY u.id, o.created_at DESC`,
    [days, limitPerUser]
  );
  return rows;   // parameterised: $1 = days, $2 = per-user limit
}
```

### 11.3 Creating Interventions From a Migration

```javascript
// Interventions like partial/expression indexes belong in MIGRATIONS, not runtime code.
async function applyOptimizerMigration() {
  // Use CONCURRENTLY so index creation doesn't lock writes on a live table.
  await pool.query(
    `CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_orders_pending
       ON orders (created_at) WHERE status = 'pending'`
  );
  await pool.query(
    `ALTER TABLE orders ALTER COLUMN status SET STATISTICS 1000`
  );
  await pool.query(`ANALYZE orders`);   // rebuild stats with the new target
}
// NOTE: CREATE INDEX CONCURRENTLY cannot run inside a transaction block,
// so do NOT wrap it in BEGIN/COMMIT. Many migration tools need a "no transaction" flag.
```

**Do ORMs handle this?** ORMs express *queries*, not *planner interventions*. None of them have a "raise statistics target" or "disable seqscan" API. Partial and expression indexes are increasingly expressible in migration DSLs (Prisma, Drizzle, Knex all support partial/expression indexes in recent versions), but statistics targets, extended statistics, and any `enable_*`/`work_mem` tuning are always raw SQL. `LATERAL` and `OFFSET 0` fences require the ORM's raw-SQL escape hatch in nearly every case.

---

## 12. ORM Comparison

The blunt truth for this topic: **no ORM has a planner-hint or planner-flag API**. Postgres itself has no hints, so there is nothing for an ORM to wrap. What the popular ORMs *can* do is (a) create the physical interventions — partial/expression indexes, statistics targets — inside their migration layer, and (b) express the query rewrites (`LATERAL`, `OFFSET 0`) that steer the planner. Everything else drops to raw SQL. This section evaluates each ORM against those two axes.

### Prisma

**Can Prisma set optimizer flags?** — No. There is no `SET enable_nestloop` API and none is planned. Session-level GUCs and `pg_hint_plan` comments require `$executeRaw` / `$queryRaw`.

**Can Prisma create the physical interventions?** — Partially. Prisma's schema supports plain and composite indexes and (since recent versions) some expression/functional index syntax via `@@index`, but **partial indexes (`WHERE`) and per-column statistics targets are not expressible in the Prisma schema**. The standard pattern is to drop into a raw migration.

```typescript
import { PrismaClient } from '@prisma/client';
const prisma = new PrismaClient();

// --- Intervention 1: session flags must be raw, and must run on the SAME
//     connection as the query. Wrap both in one interactive transaction so
//     they share a connection from the pool.
const rows = await prisma.$transaction(async (tx) => {
  await tx.$executeRawUnsafe(`SET LOCAL enable_seqscan = off`);
  await tx.$executeRawUnsafe(`SET LOCAL work_mem = '256MB'`);
  return tx.$queryRaw`
    SELECT o.id, o.user_id, o.total_amount
    FROM orders o
    WHERE o.status = 'pending'
    ORDER BY o.created_at DESC
    LIMIT 100
  `;
});
// SET LOCAL auto-resets at transaction end — no leakage back into the pool.

// --- Intervention 2: a query rewrite Prisma CAN'T model — LATERAL top-N-per-group.
//     Prisma's nested reads issue separate queries (or a JSON aggregation); to
//     force the LATERAL plan you write it out.
const latestPerUser = await prisma.$queryRaw`
  SELECT u.id, u.email, recent.id AS order_id, recent.created_at
  FROM users u
  CROSS JOIN LATERAL (
    SELECT o.id, o.created_at
    FROM orders o
    WHERE o.user_id = u.id
    ORDER BY o.created_at DESC
    LIMIT 3
  ) recent
  WHERE u.last_active_at > NOW() - INTERVAL '30 days'
`;
```

Physical interventions belong in a Prisma raw migration file:

```sql
-- prisma/migrations/xxxx_optimizer_interventions/migration.sql
CREATE INDEX CONCURRENTLY idx_orders_pending
  ON orders (created_at) WHERE status = 'pending';       -- partial index
CREATE INDEX idx_users_lower_email
  ON users (lower(email));                                -- expression index
ALTER TABLE orders ALTER COLUMN status SET STATISTICS 1000;
ANALYZE orders;
```

**Where it breaks**: `CREATE INDEX CONCURRENTLY` cannot run in a transaction, and Prisma Migrate wraps each migration in one by default — you must split it into its own migration or mark it accordingly. Prisma also cannot represent the partial-index predicate in `schema.prisma`, so drift-detection will not "see" it; treat such indexes as raw-managed.

**Verdict**: Prisma models ordinary indexes but hands you nothing for partial indexes, statistics targets, or flags. Use raw migrations for physical interventions and `$queryRaw` for rewrite fences.

---

### Drizzle ORM

**Can Drizzle set optimizer flags?** — No API, but its `sql` template makes running them ergonomic, and it models **partial and expression indexes natively** in the schema — the best of the group for this topic.

```typescript
import { pgTable, bigserial, text, timestamp, index } from 'drizzle-orm/pg-core';
import { sql } from 'drizzle-orm';

export const orders = pgTable('orders', {
  id: bigserial('id', { mode: 'number' }).primaryKey(),
  userId: bigserial('user_id', { mode: 'number' }),
  status: text('status'),
  createdAt: timestamp('created_at'),
}, (t) => ({
  // Partial index — Drizzle supports the WHERE predicate directly:
  pendingIdx: index('idx_orders_pending')
    .on(t.createdAt)
    .where(sql`${t.status} = 'pending'`),
}));
```

```typescript
import { db } from './db';

// Session flags + query on one connection via a transaction:
const rows = await db.transaction(async (tx) => {
  await tx.execute(sql`SET LOCAL enable_nestloop = off`);
  return tx.execute(sql`
    SELECT o.id, o.total_amount
    FROM orders o
    JOIN order_items oi ON oi.order_id = o.id
    WHERE o.status = 'completed'
  `);
});

// CTE materialization fence — force MATERIALIZED to compute a CTE once:
const report = await db.execute(sql`
  WITH active_users AS MATERIALIZED (
    SELECT id FROM users WHERE last_active_at > NOW() - INTERVAL '7 days'
  )
  SELECT au.id, COUNT(o.id) AS orders
  FROM active_users au
  JOIN orders o ON o.user_id = au.id
  GROUP BY au.id
`);
```

**Where it breaks**: Statistics targets (`ALTER TABLE ... SET STATISTICS`) and extended statistics (`CREATE STATISTICS`) still have no schema representation — put them in a raw SQL migration. `SET LOCAL` inside `db.transaction` is the only leak-safe way to apply flags.

**Verdict**: Drizzle is the strongest ORM here — native partial/expression indexes plus a first-class `sql` tag for `LATERAL`, `OFFSET 0`, and `AS MATERIALIZED`. Only statistics tuning drops fully to raw.

---

### Sequelize

**Can Sequelize set optimizer flags?** — No. `sequelize.query()` with raw SQL is the whole story for flags, fences, and `pg_hint_plan`.

```javascript
const { Sequelize, QueryTypes } = require('sequelize');
const sequelize = new Sequelize(process.env.DATABASE_URL);

// Flags must share the connection with the query. Sequelize hands you a
// managed transaction; SET LOCAL is scoped to it:
const rows = await sequelize.transaction(async (t) => {
  await sequelize.query(`SET LOCAL enable_seqscan = off`, { transaction: t });
  return sequelize.query(
    `SELECT p.id, p.name
     FROM products p
     JOIN order_items oi ON oi.product_id = p.id
     WHERE oi.order_id = :orderId`,
    { replacements: { orderId }, type: QueryTypes.SELECT, transaction: t }
  );
});

// pg_hint_plan hint (leading block comment) — must survive to the server:
const hinted = await sequelize.query(
  `/*+ IndexScan(orders idx_orders_pending) */
   SELECT id FROM orders WHERE status = 'pending' ORDER BY created_at DESC LIMIT 50`,
  { type: QueryTypes.SELECT }
);
```

Partial / expression indexes go in a migration via `queryInterface`:

```javascript
module.exports = {
  async up(queryInterface) {
    await queryInterface.addIndex('orders', ['created_at'], {
      name: 'idx_orders_pending',
      where: { status: 'pending' },        // Sequelize supports partial-index `where`
    });
    await queryInterface.sequelize.query(
      `ALTER TABLE orders ALTER COLUMN status SET STATISTICS 1000`
    );
  },
};
```

**Where it breaks**: Sequelize's model finders (`findAll` etc.) offer no hook for flags or leading comments, so any intervention query must be raw. Expression indexes need `Sequelize.fn`/`literal` and are fiddly; statistics targets are always raw.

**Verdict**: Sequelize can build partial indexes in migrations but everything planner-facing is raw `sequelize.query()`. Wrap flags in a managed transaction with `SET LOCAL`.

---

### TypeORM

**Can TypeORM set optimizer flags?** — No API. Use `queryRunner` (which pins one connection) for flags, and `entityManager.query()` for hinted / fenced SQL.

```typescript
import { DataSource } from 'typeorm';
const dataSource = new DataSource({ /* config */ });

// A QueryRunner holds a single connection — flag + query stay together:
const runner = dataSource.createQueryRunner();
await runner.connect();
try {
  await runner.query(`SET enable_hashjoin = off`);   // no transaction => session-scoped
  const rows = await runner.query(
    `SELECT o.id FROM orders o
     JOIN payments pm ON pm.order_id = o.id
     WHERE pm.captured = true`
  );
  // ... use rows ...
} finally {
  await runner.release();   // returns the connection with its GUCs reset by the pool
}

// OFFSET 0 optimization fence in a builder — forces the subquery to plan separately:
const fenced = await dataSource
  .createQueryBuilder()
  .select('t.*')
  .from(qb => qb
    .select('o.id', 'id')
    .addSelect('o.total_amount', 'total_amount')
    .from('orders', 'o')
    .where('o.status = :s', { s: 'completed' })
    .offset(0), 't')          // OFFSET 0 = optimization fence
  .getRawMany();
```

Partial / expression indexes are declared on the entity via `@Index`:

```typescript
@Entity('orders')
@Index('idx_orders_pending', ['createdAt'], { where: `status = 'pending'` })
export class Order { /* ... */ }
```

**Where it breaks**: `@Index` supports a `where` string for partial indexes, but statistics targets and `CREATE STATISTICS` need a raw migration. Prefer `SET LOCAL` inside a transaction over a bare `SET` on a QueryRunner, since a thrown error before `release()` could otherwise return a mutated connection to the pool.

**Verdict**: TypeORM models partial/expression indexes on entities and gives you `queryRunner`/`entityManager.query()` for flags and fences. Statistics tuning is raw.

---

### Knex.js

**Can Knex set optimizer flags?** — No dedicated API, but Knex is the most SQL-transparent: `knex.raw()` plus explicit transaction connections cover every intervention cleanly.

```javascript
const knex = require('knex')({ client: 'pg', connection: process.env.DATABASE_URL });

// Flags + query on one connection using a transaction:
const rows = await knex.transaction(async (trx) => {
  await trx.raw(`SET LOCAL join_collapse_limit = 1`);   // freeze join order to written order
  return trx('orders as o')
    .join('order_items as oi', 'oi.order_id', 'o.id')
    .where('o.status', 'completed')
    .select('o.id', 'oi.product_id');
});

// LATERAL join — Knex has no builder method, so use knex.raw for the derived table:
const topOrders = await knex('users as u')
  .crossJoin(
    knex.raw(`LATERAL (
      SELECT o.id, o.created_at FROM orders o
      WHERE o.user_id = u.id ORDER BY o.created_at DESC LIMIT 3
    ) recent`)
  )
  .select('u.id', 'recent.id as order_id', 'recent.created_at');
```

Partial and expression indexes in a Knex migration:

```javascript
exports.up = async (knex) => {
  // Partial index via the predicate option:
  await knex.schema.alterTable('orders', (t) => {
    t.index(['created_at'], 'idx_orders_pending', {
      predicate: knex.whereRaw(`status = 'pending'`),
    });
  });
  // Expression index + statistics target must be raw:
  await knex.raw(`CREATE INDEX idx_users_lower_email ON users (lower(email))`);
  await knex.raw(`ALTER TABLE audit_logs ALTER COLUMN actor_id SET STATISTICS 500`);
};
```

**Where it breaks**: Nothing structural — Knex's honesty is that it never pretends to model planner behaviour, so you always reach for `knex.raw()` for expression indexes, statistics, `LATERAL`, and `AS MATERIALIZED`. `t.index(..., { predicate })` is the one native ergonomic win.

**Verdict**: Knex is ideal for this topic precisely because it gets out of the way — raw SQL for interventions is idiomatic, not an escape hatch.

---

### ORM Summary Table

| ORM | Set planner flags | Partial index | Expression index | Statistics target | LATERAL / OFFSET 0 / AS MATERIALIZED | Verdict |
|-----|-------------------|---------------|------------------|-------------------|--------------------------------------|---------|
| Prisma | Raw (`$executeRaw` in a txn) | Raw migration | Partial (`@@index`) | Raw | `$queryRaw` | Raw for most interventions |
| Drizzle | `sql` tag in a txn | Native (`.where()`) | Native | Raw | `sql` tag | Best overall support |
| Sequelize | Raw (`sequelize.query`) | `addIndex({ where })` | `literal` (fiddly) | Raw | Raw | Partial idx native, rest raw |
| TypeORM | `queryRunner.query()` | `@Index({ where })` | `@Index` expression | Raw | `queryRunner` / builder `offset(0)` | Entity-level indexes, raw stats |
| Knex | `knex.raw()` in a txn | `t.index({ predicate })` | `knex.raw()` | `knex.raw()` | `knex.raw()` | Most transparent; raw is idiomatic |

**The universal rule**: apply session flags with `SET LOCAL` inside a transaction so they never leak back into the pooled connection, and keep the flag and the query on the *same* connection. Every ORM in the table honours this through its transaction API.

---

## 13. Practice Exercises

### Exercise 1 — Easy

The planner keeps choosing a sequential scan on `orders` for the query below, even though you believe an index scan on `created_at` would be faster for this narrow date window. As a **diagnostic experiment only** (not a permanent fix), write the statements that temporarily disable sequential scans for just this one query, run it under `EXPLAIN (ANALYZE, BUFFERS)`, and guarantee the flag resets automatically afterwards so the pooled connection is unaffected.

```sql
SELECT o.id, o.user_id, o.total_amount, o.created_at
FROM orders o
WHERE o.created_at >= DATE '2026-07-01'
  AND o.created_at <  DATE '2026-07-08';

-- Write your query here
```

---

### Exercise 2 — Medium

You run a dashboard query that first finds the set of "recently active" users, then joins that set to `orders` and `payments` to total their spend. The planner is inlining the "active users" subquery and re-evaluating it against each join, blowing up the cost. Rewrite the query as a CTE and use the **materialization fence** so the active-user set is computed exactly once, then joined. Only these tables: `users`, `orders`, `payments`.

```sql
-- Naive version the planner keeps inlining:
SELECT u.id, u.email, SUM(pm.amount) AS total_paid
FROM users u
JOIN orders o    ON o.user_id = u.id
JOIN payments pm ON pm.order_id = o.id AND pm.captured = TRUE
WHERE u.last_active_at > NOW() - INTERVAL '7 days'
GROUP BY u.id, u.email;

-- Write your query here
```

---

### Exercise 3 — Hard

Two interventions in one. (a) Analysts frequently filter `audit_logs` to only the rows where `action = 'delete'`, which are ~0.5% of a 200M-row table; existing full-column indexes are enormous and rarely chosen. (b) The same table has a heavily skewed `actor_id` column whose default statistics make the planner badly mis-estimate row counts for busy actors. Write the DDL that (1) creates a **partial index** serving the `action = 'delete'` filter on `created_at`, built without locking writes, and (2) raises the **statistics target** on `actor_id` and refreshes statistics so the planner's estimates improve. Note any constraint about transaction wrapping.

```sql
-- Target query the partial index should serve:
SELECT id, actor_id, created_at
FROM audit_logs
WHERE action = 'delete'
  AND created_at >= NOW() - INTERVAL '1 day'
ORDER BY created_at DESC;

-- Write your query here
```

---

### Exercise 4 — Interview

The query below finds, for each of the 500 most-recently-active users, that user's 3 most recent completed orders. Written as a correlated subquery the planner produces a slow plan. Rewrite it using a `LATERAL` join so the planner runs a top-N index scan on `orders` once per user. Then, in a comment, name the single index that makes the LATERAL scan fast, and explain why `OFFSET 0` would be an alternative fence if `LATERAL` were unavailable. Tables: `users`, `orders`.

```sql
-- Slow correlated form:
SELECT u.id, u.email,
       (SELECT o.id FROM orders o
        WHERE o.user_id = u.id AND o.status = 'completed'
        ORDER BY o.created_at DESC LIMIT 1) AS most_recent_order
FROM users u
ORDER BY u.last_active_at DESC
LIMIT 500;

-- Write your query here
```

---

## 14. Interview Questions

### Question 1 — Why doesn't Postgres have query hints, and what do you use instead?

**The question**: "In Oracle or SQL Server I'd add a hint like `/*+ INDEX(orders idx) */` or `WITH (INDEX(...))` to force a plan. PostgreSQL doesn't support hints. How do you influence the planner when it picks a bad plan?"

**Junior answer**: "You can turn off the plan you don't want with settings like `SET enable_seqscan = off`, which makes the planner avoid sequential scans."

**Principal answer**: This is a philosophy question as much as a technical one. Core Postgres deliberately omits hints because hints encode a *decision* ("use this index") rather than the *facts* the decision should be based on. When data distribution shifts, a hardcoded hint becomes wrong and there is no feedback loop — the community view is that a hint is a bug waiting to happen. So the intended interventions all work by improving the planner's *inputs* or *choices* rather than dictating the output:

1. **Fix the statistics first** — most "wrong plan" cases are estimation errors. Raise `default_statistics_target` or a per-column target, run `ANALYZE`, or add extended statistics (`CREATE STATISTICS`) for correlated columns. This is the correct fix ~80% of the time.
2. **Give the planner a better physical option** — a partial or expression index that exactly matches the predicate often makes the desired plan the *cheapest* one honestly, so no forcing is needed.
3. **`enable_*` flags** (`enable_seqscan`, `enable_nestloop`, `enable_hashjoin`, ...) are coarse cost penalties, not true off switches — they add a huge cost constant, so the planner still uses the "disabled" node if it is the only option. They are diagnostic tools for `EXPLAIN`, scoped with `SET LOCAL`, not production controls.
4. **Query rewrites** — `OFFSET 0` and CTE `AS MATERIALIZED` are optimization fences; `LATERAL` and `join_collapse_limit = 1` steer join shape/order. These change the *query* so the plan you want becomes natural.
5. **`pg_hint_plan`** is a third-party extension that adds Oracle-style hint comments if you genuinely need forcing — but reach for it last, because it reintroduces the staleness problem hints were avoided for.

**Follow-up**: *"You set `enable_seqscan = off` and the plan still shows a Seq Scan. Why?"* — Because `enable_seqscan = off` doesn't remove the option, it just adds a large cost penalty (a disable-cost constant). If a sequential scan is the *only* way to satisfy the query — no usable index exists — the planner still chooses it despite the penalty. The flag reveals "there is no alternative," which is itself the diagnosis: you need an index, not a flag.

---

### Question 2 — Walk me through fixing a query where actual rows are 1000x the estimate.

**The question**: "`EXPLAIN ANALYZE` shows a node estimated at 5 rows but actually returning 5,000, and downstream the planner chose a Nested Loop that's now catastrophic. How do you approach this?"

**Junior answer**: "The estimate is wrong so I'd run `ANALYZE` on the table to refresh the statistics, then re-check the plan."

**Principal answer**: A 1000x misestimate almost always means the planner's model of the data is wrong, so I work from statistics outward before touching plan shape:

1. **Run `ANALYZE`** on the affected tables and re-check — stale stats after a bulk load are the cheapest possible cause.
2. **Look at *why* the estimate is off.** If it's a single skewed column (a few `actor_id`s dominate `audit_logs`), the default 100-bucket histogram is too coarse: `ALTER TABLE audit_logs ALTER COLUMN actor_id SET STATISTICS 1000; ANALYZE;` gives finer per-value stats.
3. **If it's a multi-column correlation** — e.g. `status` and `payment_method` in `orders` are correlated, and the planner multiplies their independent selectivities and gets a tiny fraction — create **extended statistics**: `CREATE STATISTICS ord_corr (dependencies, ndistinct) ON status, payment_method FROM orders; ANALYZE orders;`. This is the canonical fix for "the AND of two filters is wildly underestimated."
4. **If the misestimate is from an opaque expression** (a function call, a `LIKE`, a JSONB extraction) the planner falls back to hardcoded default selectivities. An **expression index** on that expression gives it real statistics for the computed value.
5. **Only if none of that is fixable** do I fence the plan — wrap the underestimated subquery in `OFFSET 0` or an `AS MATERIALIZED` CTE so its bad estimate can't propagate into a Nested Loop decision above it, and consider `SET LOCAL enable_nestloop = off` as a last resort for that statement.

The principle: change the planner's *knowledge*, not its *verdict*. A forced plan is right today and wrong after the next data shift; better statistics stay right.

**Follow-up**: *"When would extended statistics NOT help?"* — When the correlation is between columns of *different tables* (join-level correlation), which single-table extended statistics can't capture, or when the misestimate comes from a join selectivity rather than a scan selectivity. There, the fix is usually a query rewrite (pre-aggregate, fence with a materialized CTE) rather than more statistics.

---

### Question 3 — Design a partial index and justify it over a full index.

**The question**: "You have a 200M-row `orders` table. 0.5% of rows are `status = 'pending'`, and the hot query is a worker polling for pending orders oldest-first. Design the index and justify your choice."

**Junior answer**: "Create an index on `status` and `created_at` so the worker query can use it: `CREATE INDEX ON orders (status, created_at);`."

**Principal answer**: I'd use a **partial index** keyed on `created_at` with a `WHERE status = 'pending'` predicate:

```sql
CREATE INDEX CONCURRENTLY idx_orders_pending
  ON orders (created_at) WHERE status = 'pending';
```

Justification across four dimensions:

- **Size**: a full `(status, created_at)` index has 200M entries; the partial index has ~1M (0.5%). It's roughly 200x smaller, so it stays cached in RAM and its maintenance overhead on every insert/update is trivial for the 99.5% of rows that don't match the predicate.
- **The planner uses it automatically**: for `WHERE status = 'pending' ORDER BY created_at`, the predicate matches, so the planner reads it as a pre-filtered, pre-sorted list — a plain index scan that satisfies both the filter and the `ORDER BY` with no sort node.
- **It's self-maintaining as rows churn**: when an order flips from `pending` to `completed`, Postgres removes it from the partial index. The index naturally tracks the small working set — it's essentially a permanent, indexed "queue" view of the table.
- **Correctness caveat**: the planner only uses a partial index when it can *prove* the query predicate implies the index predicate. The query must literally include `status = 'pending'` (a parameter `status = $1` won't match at plan time unless the value is known). I'd also build it `CONCURRENTLY` so creating it doesn't take an `ACCESS EXCLUSIVE` lock on a live 200M-row table — noting `CONCURRENTLY` can't run inside a transaction block.

**Follow-up**: *"The worker actually filters `status IN ('pending','retrying')`. Does your index still get used?"* — Not directly: the planner can't prove `status IN ('pending','retrying')` implies `status = 'pending'`. Either widen the predicate to `WHERE status IN ('pending','retrying')` in the index definition, or split the worker into two queries. This is the sharp edge of partial indexes — the predicate must be a provable superset of the query filter, so index and query must be designed together.

---

## 15. Mental Model Checkpoint

Answer these from your own understanding before moving on. If any feels shaky, re-read the referenced section.

1. **Why does PostgreSQL core refuse to add Oracle-style index hints, and what does that design choice tell you about where you should intervene instead — the planner's inputs or its outputs?**

2. **You run `SET LOCAL enable_seqscan = off` and the plan *still* shows a Seq Scan. What has actually happened at the cost level, and what is this outcome telling you to go fix?**

3. **A plain (non-materialized) CTE and one written `AS MATERIALIZED` can produce very different plans for the same query. What does the materialization fence stop the planner from doing, and when is that a win versus a mistake?**

4. **A worker query filters `WHERE status = 'pending'` and your partial index is `WHERE status IN ('pending','retrying')`. Will the planner use it? Reverse the two predicates — does the answer change? What is the general rule?**

5. **The planner estimates 5 rows and 5,000 come back for an `AND` of two correlated column filters on `orders`. Why does raising `default_statistics_target` alone often fail to fix this, and what mechanism actually addresses it?**

6. **`OFFSET 0` and `LATERAL` both change how the planner treats a subquery, but they do fundamentally different jobs. Explain what each one accomplishes and give one scenario where you'd reach for each.**

7. **You applied `SET enable_nestloop = off` and it made your query fast in `psql`, but performance in the app is unchanged and other unrelated queries got slower. What went wrong with *how* the flag was applied through the connection pool, and what is the correct pattern?**

---

## 16. Quick Reference Card

```sql
-- ============================================================
-- POSTGRES HAS NO PLANNER HINTS. Intervene on inputs & choices.
-- Order of preference: statistics > indexes > rewrites > flags > pg_hint_plan
-- ============================================================

-- 1) enable_* FLAGS (coarse cost penalties, NOT true off-switches).
--    Diagnostic use only. ALWAYS scope with SET LOCAL inside a txn so the
--    pooled connection is not mutated for the next borrower.
BEGIN;
  SET LOCAL enable_seqscan  = off;   -- penalize seq scans
  SET LOCAL enable_nestloop = off;   -- penalize nested loop joins
  SET LOCAL enable_hashjoin = off;   -- penalize hash joins
  SET LOCAL work_mem = '256MB';      -- more memory => hash/sort in RAM, not disk
  EXPLAIN (ANALYZE, BUFFERS) SELECT ...;
COMMIT;   -- SET LOCAL auto-resets here. Nothing leaks into the pool.
-- If a "disabled" node still appears => it's the ONLY option: add an index.

-- 2) STATISTICS (fix the estimate — the real fix ~80% of the time)
ALTER TABLE orders ALTER COLUMN status SET STATISTICS 1000;  -- per-column granularity
ANALYZE orders;                                             -- rebuild after change
SET default_statistics_target = 500;                        -- global default (was 100)
-- Correlated columns underestimated by AND of filters => extended statistics:
CREATE STATISTICS ord_corr (dependencies, ndistinct)
  ON status, payment_method FROM orders;
ANALYZE orders;

-- 3) PARTIAL INDEX (tiny index for a hot sliver of rows)
CREATE INDEX CONCURRENTLY idx_orders_pending             -- CONCURRENTLY: no write lock
  ON orders (created_at) WHERE status = 'pending';       -- ...cannot run in a txn block
-- Used ONLY when query predicate provably implies the index predicate.

-- 4) EXPRESSION INDEX (gives the planner stats on a computed value)
CREATE INDEX idx_users_lower_email ON users (lower(email));
-- Query must use the SAME expression: WHERE lower(email) = $1

-- 5) OPTIMIZATION FENCES (stop the planner flattening/moving a subquery)
WITH active AS MATERIALIZED (                             -- compute once, reuse
  SELECT id FROM users WHERE last_active_at > NOW() - INTERVAL '7 days'
) SELECT ... FROM active JOIN orders o ON o.user_id = active.id;
SELECT * FROM ( SELECT ... FROM orders OFFSET 0 ) t;      -- OFFSET 0 = plan barrier

-- 6) STEER JOIN SHAPE / ORDER
SET LOCAL join_collapse_limit = 1;   -- freeze join order = written order
-- LATERAL: run a top-N-per-row scan once per outer row (needs matching index)
SELECT u.id, r.id
FROM users u
CROSS JOIN LATERAL (
  SELECT o.id FROM orders o
  WHERE o.user_id = u.id ORDER BY o.created_at DESC LIMIT 3
) r;
-- Index that makes it fast: orders (user_id, created_at DESC)

-- 7) LAST RESORT: pg_hint_plan extension (real Oracle-style hints)
/*+ IndexScan(orders idx_orders_pending) */
SELECT id FROM orders WHERE status = 'pending';
-- Reintroduces staleness risk. Prefer everything above it.

-- ============================================================
-- DECISION ORDER when a plan is bad:
--   1. ANALYZE / raise stats target / extended stats  (fix the estimate)
--   2. Add partial or expression index                (make good plan cheapest)
--   3. Rewrite: OFFSET 0, AS MATERIALIZED, LATERAL     (change the query shape)
--   4. SET LOCAL enable_* (diagnostic) / join_collapse (steer)
--   5. pg_hint_plan                                    (force, accept staleness)
-- ============================================================
```

---

## Connected Topics

- **Topic 03 — The Query Planner**: The cost model these interventions manipulate. You cannot reason about an `enable_*` flag or a partial index without understanding how the planner assigns and compares costs.
- **Topic 04 — Reading EXPLAIN ANALYZE**: Every intervention here is validated by reading a plan — spotting the estimate-vs-actual gap, the disabled-node cost, the Seq Scan that won't go away. This is the feedback loop that replaces hints.
- **Topic 50 — Statistics and the Planner**: The deepest and most frequent intervention. Statistics targets, `ANALYZE`, and extended statistics live here — the "fix the inputs" half of this topic in full detail.
- **Topic 49 — Query Optimisation Methodology**: The disciplined order in which to apply these tools (measure, fix stats, add index, rewrite, and only then force). This topic is one chapter of that methodology.
- **Topic 45 — Partial Indexes**: The predicate-implication rule, size trade-offs, and the "index and query designed together" principle in full.
- **Topic 46 — Expression Indexes**: Why an opaque expression gets default selectivity, and how an expression index restores real statistics for the planner.
- **Topic 28 — CTEs** and **Topic 30 — Subquery vs CTE vs JOIN**: The `AS MATERIALIZED` fence and when materialization helps versus hurts — the query-rewrite half of this topic.
- **Topic 31 — LATERAL Joins**: The full treatment of the top-N-per-group rewrite used here as a planner-steering fence.
- **Topic 48 — When Indexes Hurt**: The counterweight — interventions that *add* physical structures also add write cost and can mislead the planner. Know when not to intervene.
