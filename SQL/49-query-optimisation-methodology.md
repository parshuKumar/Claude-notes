# Topic 49 — Query Optimisation Methodology
### SQL Mastery Curriculum — Phase 7: Indexes and Query Optimisation

---

## 1. ELI5 — The Analogy That Makes It Click

Imagine your car makes a horrible grinding noise every time you brake. You have two kinds of mechanics.

**The guessing mechanic** hears the word "brakes" and immediately starts replacing parts. New brake pads — still grinding. New rotors — still grinding. New calipers — still grinding. Three hundred dollars later he shrugs and says "cars are like that sometimes." He never once put the car on a lift and *looked*. He pattern-matched a symptom to a part and swapped parts until he ran out of ideas or money.

**The methodical mechanic** puts the car on a lift, spins the wheel, and watches. She sees the actual problem: a pebble wedged between the rotor and the dust shield. Thirty seconds with a screwdriver. Fixed. She didn't know the answer in advance — she *measured* her way to it. She formed a hypothesis ("something is contacting the rotor"), made one observation ("there's a pebble"), removed exactly one thing, and confirmed the noise was gone.

Query optimisation is the exact same discipline. The guessing engineer sees a slow query, mutters "probably needs an index," adds three indexes, and moves on — sometimes it helps, often it doesn't, occasionally it makes things worse (Topic 48). The methodical engineer measures: *which* query is actually slow (pg_stat_statements), *where* inside that query the time goes (EXPLAIN ANALYZE), *what* the single dominant cost is, forms one hypothesis, changes one thing, and measures again.

The entire topic is this: **stop guessing, start measuring.** Optimisation is not a bag of tricks. It is a loop — observe, hypothesise, change one variable, measure — run over and over until the query is fast enough or you've proven it can't get faster. Everything else in this document is the detail of how to run that loop like a principal engineer.

---

## 2. Connection to SQL Internals

Query optimisation methodology is not itself a SQL feature — it is the *discipline of interrogating* every SQL internal you've studied so far. To run the loop you must be fluent in what the engine exposes about itself:

1. **The cumulative statistics system (`pg_stat_statements`)** — PostgreSQL keeps a hash table, keyed by a normalised query fingerprint, of every statement's total calls, total time, mean time, rows, and shared-buffer hits/reads. This is a shared-memory structure flushed to `pg_stat_statements` view on demand. It is your *entry point*: it tells you which normalised query, aggregated across all executions, consumes the most server time.

2. **The planner (cost-based optimiser)** — For each query the planner enumerates candidate plans (join orders, join algorithms from Topic 17, scan methods, sort/aggregate strategies) and assigns each a **cost** in abstract units derived from `seq_page_cost`, `random_page_cost`, `cpu_tuple_cost`, and friends. It picks the cheapest. `EXPLAIN` shows you the plan it chose and the estimated cost; `EXPLAIN ANALYZE` additionally *runs* it and shows the real time and real row counts.

3. **Statistics in `pg_statistic` / `pg_stats`** — The planner's cost estimates are only as good as its table statistics: n_distinct, most-common-values, histograms, correlation. When these are stale or missing, estimates diverge from reality — the actual-vs-estimated row gap you learn to read in this topic. This is the subject of the next topic (Topic 50).

4. **The buffer pool and `BUFFERS`** — `EXPLAIN (ANALYZE, BUFFERS)` reports `shared hit` (pages found in PostgreSQL's buffer cache), `shared read` (pages read from OS/disk), `temp read/written` (spills of sorts and hashes to disk). This is how you distinguish a CPU-bound plan from an I/O-bound one.

5. **`track_io_timing` and real I/O** — When enabled, EXPLAIN reports actual milliseconds spent in read I/O, letting you separate "slow because it touched a million buffers" from "slow because it computed a million comparisons."

The methodology is the connective tissue: it is the *procedure* that turns "this endpoint is slow" into "this specific Seq Scan on `orders` reads 4 million buffers because the planner mis-estimated selectivity by 200×, and here is the one-line fix."

---

## 3. Logical Execution Order Context

Query optimisation is meta to the logical execution order — you are optimising a query whose clauses run in the usual sequence:

```
FROM / JOIN      ← scan methods, join algorithms, join order chosen here
WHERE            ← predicate pushdown, index usage decided here
GROUP BY         ← hash vs sort aggregate chosen here
HAVING           ← post-aggregation filter
SELECT           ← projection, expression evaluation
DISTINCT         ← another sort/hash
window functions ← WindowAgg nodes
ORDER BY         ← Sort node (or index-provided order)
LIMIT            ← can abort a plan early — changes the optimal plan!
```

Why this matters for methodology:

- **The bottleneck lives at a specific stage.** A slow query is rarely "slow everywhere." It is slow at *one* node — a Seq Scan in the FROM/WHERE stage, a Sort in the ORDER BY stage, a hash spill in GROUP BY. The methodology's core skill is locating *which stage* dominates, then optimising only that stage.

- **`LIMIT` changes the optimal plan.** A plan that is cheapest for "return all rows" is often not cheapest for "return the first 10." The planner accounts for this — a `LIMIT 10` with a matching index can abort after 10 rows via an Index Scan, whereas without the index it must sort the whole set. When you optimise, you must optimise *the actual query including its LIMIT*, not a simplified version.

- **Predicate placement (from Topic 11) still applies.** For INNER JOINs, WHERE and ON are interchangeable and the planner pushes predicates down. For OUTER JOINs (Topic 12) it is not — a misplaced predicate can both change results *and* defeat an index. Part of methodology is checking that your filters are actually being applied where you think.

The read order of an EXPLAIN plan is the *reverse* of the logical order in one sense — you read the plan tree from the most-indented (deepest, runs first) nodes outward — but each node maps to one of these logical stages. Learning to map plan nodes back to clauses is half the skill.

---

## 4. What Is Query Optimisation Methodology?

Query optimisation methodology is the **systematic, measurement-driven loop** for making a slow query fast: identify the genuinely expensive query from server-wide statistics, capture its real execution plan with `EXPLAIN (ANALYZE, BUFFERS)`, locate the single dominant cost node, form one falsifiable hypothesis about the cause, apply exactly one change, and re-measure — repeating until the query meets its performance budget or is proven optimal. It is the opposite of "add an index and hope."

The methodology has a canonical toolchain. Here is the diagnostic entry point, annotated:

```sql
SELECT
  queryid,                                  -- │ stable fingerprint of the normalised query
  calls,                                    -- │ how many times it ran in the sampling window
  total_exec_time,                          -- │ ← THE headline number: total ms across all calls
  mean_exec_time,                           -- │ total_exec_time / calls — per-execution cost
  stddev_exec_time,                         -- │ variance — high stddev = unstable plan / param sensitivity
  rows,                                     -- │ total rows returned across all calls
  100.0 * shared_blks_hit                   -- │ ┐
    / NULLIF(shared_blks_hit                -- │ ├ cache hit ratio: low = I/O-bound query
      + shared_blks_read, 0) AS hit_pct,    -- │ ┘
  query                                     -- │ the normalised text ($1, $2 for literals)
FROM pg_stat_statements                     -- ← the cumulative per-query stats view
ORDER BY total_exec_time DESC               -- ← rank by TOTAL server time, not mean (see §6.1)
LIMIT 20;                                   -- ← the "top 20" you actually need to look at
```

And the core measurement command, annotated:

```sql
EXPLAIN (
  ANALYZE,        -- │ actually RUN the query; report real time + real row counts
  BUFFERS,        -- │ report shared/local/temp buffer hits, reads, writes
  VERBOSE,        -- │ show output columns and schema-qualified names per node
  SETTINGS,       -- │ show any non-default planner GUCs in effect (e.g. work_mem)
  WAL,            -- │ report WAL generation (relevant for DML you're optimising)
  FORMAT TEXT     -- │ TEXT for reading; JSON for tooling/pev2 visualisers
)
SELECT ... ;      -- ← the exact query, with representative parameter values bound
```

The loop itself, as a state machine:

```
        ┌─────────────────────────────────────────────┐
        │  1. IDENTIFY  (pg_stat_statements, top-N)    │
        └───────────────────┬─────────────────────────┘
                            ▼
        ┌─────────────────────────────────────────────┐
        │  2. REPRODUCE + MEASURE  (EXPLAIN ANALYZE)   │
        └───────────────────┬─────────────────────────┘
                            ▼
        ┌─────────────────────────────────────────────┐
        │  3. LOCATE the dominant node (self-time)     │
        └───────────────────┬─────────────────────────┘
                            ▼
        ┌─────────────────────────────────────────────┐
        │  4. HYPOTHESISE one cause (estimate gap?     │
        │     missing index? spill? bad join order?)   │
        └───────────────────┬─────────────────────────┘
                            ▼
        ┌─────────────────────────────────────────────┐
        │  5. CHANGE exactly ONE thing                 │
        └───────────────────┬─────────────────────────┘
                            ▼
        ┌─────────────────────────────────────────────┐
        │  6. RE-MEASURE. Better? keep. Worse? revert. │
        └───────────────────┬─────────────────────────┘
                            ▼
             fast enough? ──no──► back to step 3
                  │yes
                  ▼
              STOP. Document the change + the numbers.
```

The whole document elaborates each box. The defining property: **at every step you have a number, and you never change two things at once.**

---

## 5. Why Query Optimisation Methodology Mastery Matters in Production

1. **Guessing wastes days and sometimes makes things worse.** Without a method, engineers add indexes speculatively. Each index slows every INSERT/UPDATE/DELETE and consumes disk and buffer cache (Topic 48). Adding five indexes to "fix" a slow query, four of which the planner never uses, is a permanent tax on write throughput paid to fix nothing. The methodology tells you *which* one index (if any) the planner will actually use before you create it.

2. **The slowest query is usually not the one you'd guess.** Engineers optimise the query that *feels* slow (the big analytics report) while the actual server killer is a tiny query that runs 4 million times an hour. `pg_stat_statements` ranked by `total_exec_time` routinely surprises people: the top consumer is often a sub-millisecond query executed absurdly often, or an ORM-generated `SELECT ... WHERE id = $1` missing a cache. You cannot fix what you haven't measured.

3. **A 30-second query and a 30-millisecond query can have identical-looking SQL.** The difference is in the plan — one gets an Index Scan, the other a Seq Scan, because of statistics, parameter values, or data volume. Reading EXPLAIN is the only way to know which you have. Two queries that look the same in the code review can differ by 1000× in production.

4. **Actual-vs-estimated row gaps are the single highest-signal diagnostic in the entire database.** When the planner thinks a step returns 10 rows and it returns 2 million, *every downstream decision it made is wrong* — it picked a Nested Loop that's now catastrophic, it under-sized a hash table that now spills. Learning to spot and explain that gap is what separates a senior from a principal engineer in an interview and in an incident.

5. **Incident response is timed.** When the database is on fire at 3am, the difference between "I have a repeatable four-step procedure" and "I start randomly adding indexes on production" is the difference between a ten-minute mitigation and a multi-hour outage. The methodology *is* the runbook.

6. **It compounds.** Every optimisation you make methodically teaches you the shape of your data and your workload. Guess-based fixes teach you nothing. Six months of methodical optimisation makes you the person who can look at a plan and know the fix in seconds — not because you memorised tricks, but because you built a mental model by measuring.

---

## 6. Deep Technical Content

This is the heart of the topic: the full methodology, stage by stage, with the judgement calls a principal engineer makes at each one.

### 6.1 Stage 1 — Identify the Right Query (pg_stat_statements)

You must enable the extension first (it requires a `shared_preload_libraries` entry and a restart):

```sql
-- postgresql.conf: shared_preload_libraries = 'pg_stat_statements'
-- then, once, per database:
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
```

**Rank by total time, not mean time.** This is the cardinal rule and the most common beginner mistake.

- `mean_exec_time DESC` surfaces the query with the highest *per-call* cost. That might be a monthly report that runs once and takes 8 seconds — real, but it costs the server 8 seconds a month.
- `total_exec_time DESC` surfaces the query consuming the most *aggregate* server time. A 2ms query called 50 million times a day costs 100,000 seconds — 28 hours of CPU. *That* is what's saturating your database.

```sql
-- The principal's first query in any performance investigation:
SELECT
  substring(query, 1, 80) AS query_start,
  calls,
  round(total_exec_time::numeric, 1)          AS total_ms,
  round(mean_exec_time::numeric, 3)           AS mean_ms,
  round(stddev_exec_time::numeric, 3)         AS stddev_ms,
  rows,
  round(100.0 * shared_blks_hit
    / NULLIF(shared_blks_hit + shared_blks_read, 0), 1) AS hit_pct
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

**Read the columns like a diagnostician:**

- **High `total_ms`, high `calls`, low `mean_ms`** → a fast query called too often. The fix is usually *not* SQL optimisation — it's caching, batching, or killing an N+1 (Topic 18). No index will help a query that's already 1ms; you have to call it less.
- **High `total_ms`, low `calls`, high `mean_ms`** → a genuinely expensive query. This is the classic EXPLAIN-ANALYZE target.
- **High `stddev_ms`** → the query's cost is unstable. Often parameter-sensitivity: `WHERE status = $1` is fast for `status = 'archived'` (rare, indexed) and slow for `status = 'active'` (90% of the table, Seq Scan). The planner may even be flipping plans between executions. This is a signal to test EXPLAIN with *multiple representative parameter values*.
- **Low `hit_pct`** → I/O-bound. The query is reading from disk, not cache. Either the working set exceeds RAM, or the query touches far more data than it needs (missing index causing a scan).

**Reset the statistics before a measurement window** so you're looking at a clean sample, not months of accumulated noise:

```sql
SELECT pg_stat_statements_reset();
-- ... let representative traffic run for 15-60 minutes ...
-- ... then run the top-20 query above ...
```

**A caveat on normalisation:** `pg_stat_statements` groups queries by fingerprint, replacing literals with `$1`, `$2`. This means `WHERE id = 5` and `WHERE id = 9` are one row — good. But it also means `IN (1,2,3)` and `IN (1,2,3,4)` may fingerprint *differently* on older versions (each list length is distinct), scattering one logical query across many rows. Be aware when a query you know is hot doesn't appear near the top — it may be fragmented.

### 6.2 Stage 2 — Reproduce and Measure (EXPLAIN ANALYZE, BUFFERS)

Once you have a target query and representative parameter values, capture the *real* plan.

**`EXPLAIN` vs `EXPLAIN ANALYZE`:**
- `EXPLAIN` alone shows the planner's *estimated* plan and cost without running the query. Fast, safe, but only estimates.
- `EXPLAIN ANALYZE` *actually executes* the query and reports real timings and real row counts alongside the estimates. This is what you need — the estimate-vs-actual comparison is the whole game.

**Danger:** `EXPLAIN ANALYZE` runs the query. For a `SELECT` that's usually fine (though a 30-second query still takes 30 seconds). For `INSERT`/`UPDATE`/`DELETE` **it performs the write**. To analyse DML safely, wrap it:

```sql
BEGIN;
EXPLAIN (ANALYZE, BUFFERS)
UPDATE orders SET status = 'shipped' WHERE created_at < '2020-01-01';
ROLLBACK;   -- the plan is captured; the write is undone
```

**Always include `BUFFERS`.** Without it you know *how long* a node took but not *why*. `BUFFERS` tells you whether the time went into touching pages (I/O or cache pressure) or into CPU. Enable `track_io_timing` server-side to additionally get real I/O milliseconds.

**Run it more than once.** The first execution of a query may read from cold cache (disk); the second reads from warm buffer cache. A principal engineer runs EXPLAIN ANALYZE two or three times and notes both the cold and warm numbers — production behaviour depends on which state your buffers are usually in.

**Use `FORMAT JSON` for tooling.** For visual analysis, `EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)` output feeds directly into visualisers (pev2, explain.dalibo.com) that highlight the dominant node and the worst estimate gaps automatically. For the terminal and for interviews, `FORMAT TEXT`.

### 6.3 Stage 3 — Locate the Dominant Node (self-time, not total-time)

An EXPLAIN plan is a tree. Each node reports `actual time=START..END`, and this is where beginners misread it.

**The times are cumulative and per-loop.** The `END` time of a node *includes the time of all its children*. So the topmost node always shows the largest number — that's just the total. To find the *bottleneck*, you need each node's **self-time**: its own time minus its children's time.

**The times are per-loop, multiply by `loops`.** A node showing `actual time=0.5..0.8 rows=3 loops=50000` did not take 0.8ms — it ran 50,000 times. Its real contribution is roughly `0.8ms × 50000 = 40 seconds`. This is the single most common misreading of EXPLAIN output and a favourite interview trap. The inner side of a Nested Loop is where this bites.

```
Nested Loop  (actual time=0.05..41230.0 rows=48000 loops=1)   ← total 41s
  ->  Seq Scan on orders o
        (actual time=0.02..12.1 rows=50000 loops=1)            ← self ~12ms
  ->  Index Scan using ... on customers c
        (actual time=0.8..0.82 rows=1 loops=50000)             ← 0.82 × 50000 ≈ 41s  ← BOTTLENECK
```

The top line says 41 seconds, but the *cause* is the inner Index Scan invoked 50,000 times, not the cheap Seq Scan on top. The self-time / loops arithmetic points straight at it.

**The principal's node-triage checklist**, applied to every node:
1. What is this node's real cost (`self-time × loops`)?
2. What is its `rows estimated` vs `rows actual`? (the gap — §6.4)
3. Is it a scan (Seq/Index/Bitmap), a join (Nested/Hash/Merge), or a materialiser (Sort/Hash/Materialize/Aggregate)?
4. Does its `BUFFERS` line show a lot of `read` (I/O) or `temp` (spill)?

The node with the largest real cost is your target. Optimise *that node*. Everything else is a distraction until that one is fixed — a 41-second query does not get faster by shaving 2ms off the Seq Scan.

### 6.4 Stage 4 — The Actual-vs-Estimated Row Gap (the master diagnostic)

Every node prints `rows=N` (the planner's **estimate**) and `actual ... rows=M` (**reality**). The ratio `M/N` (or `N/M`) is the highest-signal number in the entire plan.

**Why it matters:** the planner chose the entire plan — join algorithms, join order, scan methods, memory sizing — based on the *estimates*. If an estimate is off by 100×, the plan built on top of it is very likely wrong:
- Estimated 5 rows, actual 500,000 → the planner picked a **Nested Loop** (great for 5 iterations, catastrophic for 500,000) and probed an index half a million times.
- Estimated 1,000,000 rows, actual 50 → the planner picked a **Hash Join** and built a giant hash table, or refused an Index Scan it should have used.
- Under-estimate feeding a hash/sort → the node was allocated too little `work_mem` and **spilled to disk** (temp buffers, `Batches > 1`).

**How to read the gap:**

```
->  Index Scan using orders_status_idx on orders o
      (cost=0.42..8.10 rows=12 width=48)            ← planner thinks 12 rows
      (actual time=0.03..842.1 rows=1904233 loops=1) ← reality: 1.9 MILLION rows
      Index Cond: (status = 'active')
```

A 12-vs-1,900,000 gap is a five-order-of-magnitude miss. The cause is almost always **statistics**: the planner's most-common-values / histogram for `status` is stale or the column has skew it can't model. The immediate remedies: `ANALYZE orders;` to refresh stats; raise the statistics target for that column (`ALTER TABLE orders ALTER COLUMN status SET STATISTICS 1000;`); or add an extended statistics object for correlated columns. This is the doorway to Topic 50.

**The rule of thumb principal engineers use:** *a plan is suspect the moment any node's estimate is off by more than ~10×.* Find the deepest node where estimates first go wrong — errors propagate upward, so the deepest wrong estimate is the root cause; the ones above it are just inheriting the mistake.

**Where estimates go wrong most often:**
- **Correlated predicates.** `WHERE city = 'Paris' AND country = 'France'` — the planner multiplies the two selectivities as if independent, massively under-estimating (every Paris *is* in France). Fix: `CREATE STATISTICS ... (dependencies)` (Topic 50).
- **Expressions the planner can't see through.** `WHERE lower(email) = $1` or `WHERE created_at::date = $1` — the planner has no statistics on the *expression*, only the raw column, so it guesses (often a hard-coded default selectivity). Fix: an expression index gives it stats to use.
- **Skewed data.** A `status` column that's 95% `'active'` and 5% everything else. If MCV stats are stale, the planner assumes uniform distribution and gets it wildly wrong for the common value.
- **Cross-table correlation after joins.** Join selectivity estimation is inherently harder; errors compound multiplicatively through a multi-join.

### 6.5 Stage 5 — Form Exactly One Hypothesis

A hypothesis is a *falsifiable* statement of cause plus a predicted fix. Not "the query is slow because it needs tuning" — that's not a hypothesis, it's a shrug. A real hypothesis:

> "The Seq Scan on `orders` (self-time 4.2s, reads 4M buffers, filter `status='active'` removing 5% of rows) is the bottleneck. There is no index on `status`. **Hypothesis:** a partial index `WHERE status='active'` will let the planner replace the Seq Scan with an Index Scan and cut buffer reads by ~20×. Predicted result: node drops from 4.2s to <200ms."

Notice the anatomy:
- **The node** (which one, with its real cost).
- **The evidence** (buffers, filter, row counts).
- **The cause** (no usable index).
- **The intervention** (specific, single).
- **The prediction** (a number you can check).

If your predicted result comes true, you've confirmed the mechanism *and* fixed the problem. If it doesn't, you've *learned something real* (the index wasn't the cause) instead of accumulating superstition. Either way you made progress — that's the point of a hypothesis over a guess.

**Common hypothesis families**, mapped to the evidence that suggests each:

| Evidence in the plan | Likely hypothesis | Candidate one-change |
|---|---|---|
| Seq Scan on big table + selective filter | Missing/unusable index | Add (partial/expression) index |
| Nested Loop, inner Index Scan, huge `loops` | Bad join, driven by under-estimate | Fix stats, or coax Hash Join |
| Estimate ≪ actual on a scan | Stale/insufficient statistics | `ANALYZE`, raise stats target |
| `Batches > 1` on Hash / external merge Sort | Spill: `work_mem` too small | Raise `work_mem` (session) |
| Index Scan but reading many heap pages | Low correlation, index not covering | Covering index (`INCLUDE`), or CLUSTER |
| Filter removing 99% *after* an Index Scan | Index on wrong/leading column | Reorder/compose the index |
| Same query, wildly varying time | Parameter-sensitive plan | Test multiple params; consider plan hints/rewrite |

### 6.6 Stage 6 — Change One Thing, Then Measure

**One variable at a time.** If you add an index *and* bump `work_mem` *and* rewrite the join in one commit, and the query gets faster, you have learned *nothing* about which change mattered — and you may be carrying two useless changes (the extra index tax, the raised memory) to production forever. Change one thing. Measure. Keep or revert. Then the next.

**Measure with the same command every time.** Re-run the identical `EXPLAIN (ANALYZE, BUFFERS)` with the same parameters. Compare: did the target node's real cost drop? Did the plan *shape* change (did the planner switch algorithms)? Did a *new* bottleneck appear elsewhere (common — you fix the scan and now the Sort dominates)?

**Test changes cheaply before committing them:**

- **Session-level GUCs** let you test a hypothesis without touching global config:
  ```sql
  SET work_mem = '256MB';   -- session only; test the spill hypothesis
  EXPLAIN (ANALYZE, BUFFERS) SELECT ...;
  RESET work_mem;           -- undo — don't leak session state
  ```
- **`enable_*` flags** let you *force* the planner off a choice to test a counterfactual — "would a Hash Join actually be faster than this Nested Loop?":
  ```sql
  SET enable_nestloop = off;   -- diagnostic ONLY, never in production
  EXPLAIN (ANALYZE, BUFFERS) SELECT ...;
  RESET enable_nestloop;
  ```
  These are *diagnostic instruments*, not fixes. If turning off nestloop makes the query 10× faster, that's evidence your *statistics* are wrong (the planner should have chosen the hash join itself) — the real fix is fixing the stats, not shipping `enable_nestloop=off`.
- **`CREATE INDEX CONCURRENTLY`** on a replica or a copy to test whether an index helps, before building it on the primary.
- **`HypoPG`** extension creates *hypothetical* indexes that exist only in the planner's imagination — you can see whether the planner *would* use an index, and what the estimated cost becomes, without paying the build cost at all. The principal's favourite for "would this index even get used?"

**Document the result.** The final artefact of an optimisation is not just the faster query — it's the record: "Query X, was 4.2s, added partial index Y, now 180ms, plan switched Seq Scan → Index Scan, verified 2026-07-19." This is what makes the *next* incident faster and what you paste into the PR.

### 6.7 Anti-Patterns — How Optimisation Goes Wrong

- **Optimising the wrong query.** Skipping Stage 1 and optimising the query that *feels* slow instead of the one `pg_stat_statements` proves is expensive.
- **Reading total-time as self-time.** Chasing the top node's big number instead of the deep node that actually owns the cost.
- **Ignoring `loops`.** Missing that the inner node's tiny per-loop time × 50,000 loops is the whole query.
- **Changing many things at once.** Learning nothing, carrying dead weight.
- **Adding indexes speculatively.** Every index taxes writes (Topic 48). Confirm the planner will use it (HypoPG / EXPLAIN) *before* creating it.
- **Optimising with unrepresentative data.** A plan on a 1,000-row dev table is meaningless; the planner picks Seq Scans for small tables regardless. Test on production-scale data.
- **Optimising with unrepresentative parameters.** `WHERE status = 'archived'` (rare) gives a different plan than `WHERE status = 'active'` (common). Test the real hot path.
- **Stopping too early or too late.** "Fast enough" is defined by a budget (the SLO). Below the budget, stop — further optimisation is gold-plating. Above it, keep going. Don't optimise a 50ms query to 40ms because you can.
- **Trusting `EXPLAIN` without `ANALYZE`.** Estimates lie; that's the entire premise. Never conclude from estimates alone.

### 6.8 The Performance Budget — Knowing When to Stop

Optimisation without a target is infinite. Before you start, know the budget:
- **Interactive/OLTP query:** typically < 100ms end-to-end, often < 10ms for a point lookup.
- **API-backing query:** whatever leaves the endpoint within its p99 SLO after network + serialisation overhead.
- **Background/analytics:** minutes may be fine if it's async and off-peak.

You stop when the query is under budget *or* when you've proven (via the plan) that it's doing the minimum work possible — reading only the rows it must, via the best available access method, with no spills. A Seq Scan that returns 90% of a table is *not* a bug; it's the correct plan, and no index will beat it. Recognising "this is already optimal" is as important as fixing the slow ones — it stops you burning days on a query that can't get faster (that's Topic 48's lesson: sometimes the index *hurts*).

---

## 7. EXPLAIN — Reading a Real Optimisation in the Plan

This section walks a single query from slow to fast, showing the plan at each step of the loop. The scenario: an orders dashboard query that got slow as the table grew to 20M rows.

### The query and its first (slow) plan

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT o.id, o.total_amount, o.created_at, c.name
FROM orders o
INNER JOIN customers c ON c.id = o.customer_id
WHERE o.status = 'active'
  AND o.created_at >= '2026-07-01'
ORDER BY o.created_at DESC
LIMIT 50;
```

```
Limit  (cost=458210.5..458216.3 rows=50 width=56)
       (actual time=3821.4..3821.9 rows=50 loops=1)
  Buffers: shared hit=1204 read=248510
  ->  Sort  (cost=458210.5..461887.2 rows=1470680 width=56)
            (actual time=3821.4..3821.6 rows=50 loops=1)
        Sort Key: o.created_at DESC
        Sort Method: top-N heapsort  Memory: 34kB
        Buffers: shared hit=1204 read=248510
        ->  Hash Join  (cost=3502.0..409221.0 rows=1470680 width=56)
                       (actual time=48.2..3402.1 rows=1512340 loops=1)
              Hash Cond: (o.customer_id = c.id)
              Buffers: shared hit=1204 read=248510
              ->  Seq Scan on orders o
                    (cost=0.00..401210.0 rows=1470680 width=44)
                    (actual time=0.3..2705.4 rows=1512340 loops=1)
                    Filter: ((status = 'active')
                       AND (created_at >= '2026-07-01'))
                    Rows Removed by Filter: 18487660
                    Buffers: shared read=248000
              ->  Hash  (cost=2002.0..2002.0 rows=100000 width=20)
                        (actual time=47.1..47.1 rows=100000 loops=1)
                    Buckets: 131072  Batches: 1  Memory Usage: 6144kB
                    ->  Seq Scan on customers c
                          (actual time=0.01..18.2 rows=100000 loops=1)
                          Buffers: shared hit=1204 read=510
Planning Time: 0.4 ms
Execution Time: 3822.3 ms
```

**Line-by-line reading (Stage 3 + 4):**

- **`Execution Time: 3822.3 ms`** — 3.8 seconds. Budget is 100ms. We have work to do.
- **Top node `Limit`** shows 3821ms — but that's total time, not where the time *went*. Read downward.
- **`Sort`** — top-N heapsort, only 34kB, fast (no spill). Not the problem, despite sitting near the top. Note it processed the *whole* join result to find the top 50.
- **`Hash Join`** — self-time ≈ 3402 − 2705 (child) ≈ 700ms building/probing. Estimated 1,470,680 rows, actual 1,512,340. Estimate is *good* here (~3% off). The join isn't the root cause.
- **`Seq Scan on orders`** — **here it is.** 2705ms of self-time, `Buffers: shared read=248000` (248,000 pages read from disk — that's the I/O cost). Crucially: **`Rows Removed by Filter: 18,487,660`** — it read all 20M rows and *threw away 18.5M of them*. The filter `status='active' AND created_at >= '2026-07-01'` is selective (returns 1.5M of 20M ≈ 7.5%) but there's no index to exploit it, so the engine scans the entire heap.

**The estimate gap check (§6.4):** estimate 1,470,680 vs actual 1,512,340 on the Seq Scan — only 3% off. So this is *not* a statistics problem. The planner correctly knows the filter returns ~1.5M rows; it simply has no better access path than a full scan. This distinction matters: the fix is an *index*, not `ANALYZE`.

### Hypothesis and the one change

> The Seq Scan on `orders` reads all 20M rows (248K buffers, 2.7s) and discards 18.5M via a filter that's ~7.5% selective. **Hypothesis:** a composite index on `(status, created_at DESC)` lets the planner do an Index Scan that reads only matching rows *and* provides the ORDER BY order, letting `LIMIT 50` abort after 50 rows. Predicted: Seq Scan + Sort both vanish; execution drops to single-digit ms.

```sql
CREATE INDEX CONCURRENTLY idx_orders_status_created
  ON orders (status, created_at DESC);
```

Note the index column order is deliberate: equality column (`status`) first, then the range/order column (`created_at`) — this is what lets a single index serve both the filter and the `ORDER BY ... LIMIT`.

### The re-measured (fast) plan

```
Limit  (cost=0.56..42.1 rows=50 width=56)
       (actual time=0.09..1.24 rows=50 loops=1)
  Buffers: shared hit=212
  ->  Nested Loop  (cost=0.56..122340.0 rows=1512340 width=56)
                   (actual time=0.08..1.19 rows=50 loops=1)
        Buffers: shared hit=212
        ->  Index Scan using idx_orders_status_created on orders o
              (cost=0.43..70120.0 rows=1512340 width=44)
              (actual time=0.05..0.42 rows=50 loops=1)
              Index Cond: ((status = 'active')
                 AND (created_at >= '2026-07-01'))
              Buffers: shared hit=54
        ->  Index Scan using customers_pkey on customers c
              (actual time=0.01..0.01 rows=1 loops=50)
              Index Cond: (id = o.customer_id)
              Buffers: shared hit=158
Planning Time: 0.5 ms
Execution Time: 1.31 ms
```

**Reading the win:**
- **`Execution Time: 1.31 ms`** — from 3822ms to 1.31ms, a **~2900× improvement**.
- The **Sort node is gone** — the index provides `created_at DESC` order directly, so `LIMIT` reads exactly 50 rows and stops.
- The **Seq Scan is gone** — replaced by an **Index Scan** with `Index Cond` doing the filtering *in the index*, reading only 54 buffers instead of 248,000.
- The planner **switched Hash Join → Nested Loop** — correct now, because it only needs to look up 50 customers (`loops=50`), not join 1.5M rows. This is the planner adapting to the LIMIT once the index makes the ordered path cheap.
- `Rows Removed by Filter` is absent — no wasted row reads.

**The lesson embodied:** one index, chosen by reading exactly where the time went (the Seq Scan's 18.5M discarded rows) and confirming it wasn't a stats problem (tight estimate), turned a 3.8-second query into a 1.3ms query — and cascaded into the planner making three *other* good decisions on its own (drop the sort, switch the join, abort at the limit).

---

## 8. Query Examples

### Example 1 — Basic: The Two-Command Triage

The minimum viable optimisation session: find the worst query, look at its plan.

```sql
-- STEP 1: which query costs the server the most total time?
SELECT
  substring(query, 1, 60) AS q,
  calls,
  round(mean_exec_time::numeric, 2) AS mean_ms,
  round(total_exec_time::numeric, 0) AS total_ms
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 5;

--            q                          | calls  | mean_ms | total_ms
-- ---------------------------------------+--------+---------+----------
--  SELECT * FROM orders WHERE user_id=$1 | 4210355|   2.10  | 8841745   ← total killer
--  SELECT ... big monthly report ...     |     31 | 8210.0  |  254510
--  UPDATE sessions SET last_seen=$1 ...  |1204331 |   0.90  | 1083898

-- STEP 2: capture the real plan of the top offender with representative params
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE user_id = 91823;
```

The insight this example teaches: the "big monthly report" *feels* like the problem at 8.2 seconds each, but the point-lookup at 2.1ms × 4.2M calls costs the server **35× more total time**. Stage 1 saves you from optimising the wrong thing.

### Example 2 — Intermediate: Confirming a Statistics Problem Before Acting

Distinguishing a *missing index* problem from a *bad statistics* problem — because the fix is completely different.

```sql
-- The query is slow. Get the plan.
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders
WHERE status = 'active' AND region = 'EU'
ORDER BY created_at DESC LIMIT 20;
```

```
Limit (actual time=1240.5..1240.6 rows=20 loops=1)
  ->  Sort (actual time=1240.5..1240.5 rows=20 loops=1)
        Sort Key: created_at DESC
        ->  Bitmap Heap Scan on orders
              (cost=... rows=95 width=...)              ← estimate: 95 rows
              (actual time=210.1..1180.2 rows=842000 loops=1)  ← actual: 842,000
              Recheck Cond: (status = 'active')
              Filter: (region = 'EU')
              Rows Removed by Filter: 0
              ->  Bitmap Index Scan on idx_orders_status
                    (actual rows=842000 loops=1)
```

**The diagnosis:** estimate 95, actual 842,000 — an **8,800× under-estimate**. There *is* an index (`idx_orders_status`), and it's being used. This is **not** a missing-index problem. The planner thinks `status='active' AND region='EU'` matches 95 rows because it multiplies the two selectivities independently — but active EU orders are highly correlated. The under-estimate is why it chose to sort 842K rows instead of using an ordered index path.

```sql
-- The RIGHT fix is stats, not another index:
CREATE STATISTICS orders_status_region (dependencies)
  ON status, region FROM orders;
ANALYZE orders;
-- Now the planner estimates correctly and may pick a
-- (status, region, created_at) index path or a smarter plan.
```

The point of Example 2: reading the estimate gap told you the category of problem. Blindly adding `CREATE INDEX ON orders(region)` here would have wasted a write-tax index without addressing the mis-estimate that caused the bad plan.

### Example 3 — Production Grade: Full Loop on a Reporting Query

**Context:** A revenue-by-category endpoint backing an internal dashboard. Table sizes: `orders` 20M rows, `order_items` 85M rows, `products` 400K, `categories` 300. Index availability at start: PKs only, plus `orders(customer_id)`. Performance expectation/budget: **< 500ms** (internal dashboard, p95). It currently takes **11 seconds**.

```sql
-- The query under investigation
EXPLAIN (ANALYZE, BUFFERS)
SELECT
  cat.name AS category,
  COUNT(DISTINCT o.id)                    AS orders,
  SUM(oi.quantity * oi.unit_price)        AS revenue
FROM orders o
INNER JOIN order_items oi ON oi.order_id = o.id
INNER JOIN products p     ON p.id = oi.product_id
INNER JOIN categories cat ON cat.id = p.category_id
WHERE o.status = 'completed'
  AND o.created_at >= DATE_TRUNC('month', CURRENT_DATE) - INTERVAL '3 months'
GROUP BY cat.name
ORDER BY revenue DESC;
```

**First plan (abridged to the dominant nodes):**

```
Sort  (actual time=11021.5..11021.6 rows=300 loops=1)
  Sort Key: (sum(oi.quantity * oi.unit_price)) DESC
  ->  HashAggregate  (actual time=10980.1..11020.0 rows=300 loops=1)
        Group Key: cat.name
        Batches: 5  Memory: 4096kB  Disk: 51200kB          ← SPILL to disk
        ->  Hash Join  (actual time=210.1..9820.4 rows=41200000 loops=1)
              Hash Cond: (oi.order_id = o.id)
              ->  Seq Scan on order_items oi
                    (actual time=0.1..3900.2 rows=85000000 loops=1)  ← reads ALL 85M
                    Buffers: shared read=1120000
              ->  Hash (actual time=205.0..205.0 rows=920000 loops=1)
                    ->  Seq Scan on orders o
                          (actual time=0.2..180.1 rows=920000 loops=1)
                          Filter: (status='completed' AND created_at >= ...)
                          Rows Removed by Filter: 19080000
Execution Time: 11023.0 ms
```

**Diagnosis (Stages 3-4):**
1. **`Seq Scan on order_items`** reads all 85M rows (1.12M buffers). It's the biggest I/O cost. But note — there's no filter on `order_items`; it *needs* every item whose order matches. The real waste is that it processes items for *all* orders before the join filters them down.
2. **`Seq Scan on orders`** discards 19M of 20M rows — the date+status filter is selective (~4.6%) but unindexed.
3. **`HashAggregate` spills** — `Disk: 51200kB`, `Batches: 5` — `work_mem` too small for the 41M-row join result.
4. Estimates are roughly accurate (not shown but tight) — so this is an **access-path + memory** problem, not a stats problem.

**The loop — one change at a time:**

```sql
-- CHANGE 1: index the orders filter so we don't scan 20M rows.
CREATE INDEX CONCURRENTLY idx_orders_status_created
  ON orders (status, created_at)
  WHERE status = 'completed';        -- partial: only the rows we ever query
-- Re-measure: orders Seq Scan → Index Scan. 11.0s → 6.2s.

-- CHANGE 2: index order_items on the join key so the large table
-- can be probed by index instead of fully scanned once orders is small.
CREATE INDEX CONCURRENTLY idx_order_items_order_id
  ON order_items (order_id) INCLUDE (product_id, quantity, unit_price);
-- INCLUDE makes it a covering index — the join + aggregate read
-- everything from the index, skipping heap fetches.
-- Re-measure: order_items Seq Scan → Index Scan, Nested Loop from
-- the ~920K filtered orders. 6.2s → 900ms.

-- CHANGE 3: the aggregate still spills. Raise work_mem for this query.
SET work_mem = '128MB';   -- session/transaction scope for the report
-- Re-measure: HashAggregate Batches: 1 (no disk). 900ms → 410ms.  ✓ under budget
```

**Final plan (abridged):**

```
Sort (actual time=408.1..408.2 rows=300 loops=1)
  ->  HashAggregate (actual time=380.0..405.1 rows=300 loops=1)
        Group Key: cat.name
        Batches: 1  Memory: 61440kB                       ← no spill
        ->  Nested Loop (actual time=0.4..320.5 rows=4100000 loops=1)
              ->  Nested Loop ...
                    ->  Index Scan using idx_orders_status_created on orders o
                          (actual time=0.05..85.1 rows=920000 loops=1)
                    ->  Index Only Scan using idx_order_items_order_id on order_items oi
                          Index Cond: (order_id = o.id)
                          Heap Fetches: 0                   ← covering index working
Execution Time: 410.3 ms
```

**Result:** 11,023ms → 410ms (**~27× faster**), now under the 500ms budget. Three changes, each measured independently: two indexes (each confirmed used before keeping) and one memory bump scoped to the query. The write-tax of the two indexes is justified because the plan *proves* they're used; the `work_mem` change is scoped to the reporting transaction, not global. Documented and shipped.

---

## 9. Wrong → Right Patterns

The methodology's failures are process failures — the query you optimise, the number you read, the count of things you change. Each pair below shows the wrong process, then the right one, with what actually happens at the execution level.

### Wrong 1: Ranking by mean time and optimising the "slow" query

```sql
-- WRONG: find the "slowest" query and pour effort into it
SELECT query, mean_exec_time
FROM pg_stat_statements
ORDER BY mean_exec_time DESC       -- ranks by PER-CALL cost
LIMIT 5;
-- Surfaces the monthly analytics report at 8s/call. You spend a day
-- indexing it. Server-wide load barely moves.
```

**Why it's wrong at the execution level:** `mean_exec_time DESC` weights a query that runs once a month the same as one that runs 4 million times an hour. The monthly report costs the server 8 seconds *per month*. Meanwhile a 2ms point-lookup called 4.2M times/day burns 8,400 seconds/day of CPU — 1000× the aggregate load — and it never appears near the top of a mean-ranked list. You optimised the query whose total contribution to server load is a rounding error.

```sql
-- RIGHT: rank by TOTAL server time consumed
SELECT query, calls, total_exec_time, mean_exec_time
FROM pg_stat_statements
ORDER BY total_exec_time DESC      -- ranks by AGGREGATE cost = calls × mean
LIMIT 5;
-- Surfaces the 2ms × 4.2M-calls point-lookup. The fix isn't an index
-- (it's already 2ms) — it's caching or killing an N+1. That moves the needle.
```

`total_exec_time = calls × mean_exec_time` is the number that maps to actual database load. Optimising the top of *this* list is the only ranking that reduces server-wide time. (§6.1)

### Wrong 2: Reading the top node's time as the bottleneck

```sql
-- The plan (abridged):
-- Nested Loop  (actual time=0.05..41230.0 rows=48000 loops=1)   ← 41 seconds
--   ->  Seq Scan on orders o   (actual time=0.02..12.1 rows=50000 loops=1)
--   ->  Index Scan on customers c (actual time=0.8..0.82 rows=1 loops=50000)

-- WRONG: "The Nested Loop takes 41s and the Seq Scan feeds it 50000 rows —
--         the Seq Scan is the problem, let me index orders."
CREATE INDEX ON orders (customer_id);   -- indexes the CHEAP node
-- Re-measure: still 41 seconds. Nothing changed.
```

**Why it's wrong at the execution level:** the Nested Loop's 41,230ms is *cumulative* — it includes every child's time. The Seq Scan's self-time is ~12ms; it is not the bottleneck. The inner Index Scan shows `actual time=0.8..0.82 ... loops=50000`. That 0.82ms is **per loop**. Its real contribution is `0.82ms × 50,000 = 41 seconds` — the entire query. You indexed a node that costs 12ms and left the node that costs 41s untouched.

```sql
-- RIGHT: compute self-time × loops for every node, act on the largest.
--   Seq Scan:    ~12ms × 1 loop      = 12ms      (ignore)
--   Index Scan:  0.82ms × 50000 loops = ~41s     ← THE bottleneck
-- The inner Index Scan runs 50000 times because the Nested Loop drives it
-- once per outer row. The fix targets THAT: make the join not iterate 50000
-- times (coax a Hash Join by fixing the estimate that made nestloop look cheap),
-- or ensure the inner probe is a single-buffer index hit, not a heap fetch.
```

The arithmetic `self-time × loops` is the whole skill of Stage 3. The top number tells you the total; it never tells you the cause. (§6.3)

### Wrong 3: Adding an index without checking the estimate gap

```sql
-- Plan shows a slow scan. Estimate says 95 rows, actual is 842,000.
-- WRONG: "It's scanning a lot — add an index on the filtered column."
CREATE INDEX ON orders (region);
-- Re-measure: the planner STILL mis-estimates and STILL picks the bad plan,
-- because the problem was never the access path.
```

**Why it's wrong at the execution level:** an 8,800× estimate gap (95 vs 842,000) means the planner's *model of the data* is broken — it multiplied `sel(status='active') × sel(region='EU')` as if independent, but the columns are correlated. Every downstream choice (sort 842K rows instead of an ordered path, or which index to even consider) was made on the false belief that 95 rows come out. A new single-column index doesn't correct the correlation estimate, so the planner keeps choosing wrong. You paid a permanent write-tax and fixed nothing.

```sql
-- RIGHT: read the gap FIRST, let it classify the problem.
-- Gap ≈ 1×  → access-path problem → maybe an index helps.
-- Gap ≫ 10× → statistics problem → fix the estimate, THEN re-plan.
CREATE STATISTICS orders_status_region (dependencies) ON status, region FROM orders;
ANALYZE orders;
-- Now the planner estimates ~842K correctly and chooses a sane plan on its own.
```

The estimate-vs-actual ratio is the master diagnostic precisely because it tells you *which category* of fix applies before you spend a write-tax finding out. (§6.4, Example 2)

### Wrong 4: Changing several things in one commit

```sql
-- WRONG: the query is slow, so throw the whole toolbox at it at once.
CREATE INDEX ON orders (status, created_at);
CREATE INDEX ON order_items (order_id);
SET work_mem = '256MB';
ALTER TABLE orders ALTER COLUMN status SET STATISTICS 1000;
-- Re-measure: 8s → 300ms. Ship it! ...but which change did the work?
```

**Why it's wrong at the execution level:** the query got faster, so you learned *nothing about the mechanism*. Maybe only the first index mattered and the other three are dead weight — two unused indexes taxing every INSERT/UPDATE/DELETE forever (Topic 48), a global-ish `work_mem` bump risking OOM under concurrency, and a stats-target change that recomputes on every ANALYZE. You are now carrying three permanent costs to production, none of which you can prove earned its place.

```sql
-- RIGHT: one change, re-measure, keep or revert, then the next.
CREATE INDEX CONCURRENTLY idx_orders_status_created ON orders (status, created_at);
EXPLAIN (ANALYZE, BUFFERS) SELECT ...;   -- 8s → 900ms? keep. Seq Scan → Index Scan confirmed.
-- Only if a bottleneck REMAINS, address the next one:
SET work_mem = '256MB';
EXPLAIN (ANALYZE, BUFFERS) SELECT ...;   -- 900ms → 300ms? the spill was real; scope it. Else RESET.
```

One variable at a time is not pedantry — it is the only way each surviving change is *proven* to pull its weight, so production carries exactly the cost that buys the speed and no more. (§6.6, §6.7)

---

## 10. Performance Profile

The methodology itself has costs and characteristics worth knowing — the diagnostic tools are not free, and each stage has a signal-to-effort profile.

| Tool / Action | Cost to Run | What It Tells You | When to Reach For It |
|---|---|---|---|
| `pg_stat_statements` query | Near-zero (reads shared memory) | Which query dominates aggregate load | Always first — Stage 1 |
| `EXPLAIN` (no ANALYZE) | Planning time only (~ms) | The *estimated* plan and cost | Quick "what would it do?"; DML preview |
| `EXPLAIN (ANALYZE)` | **Runs the query fully** | Real time + real rows per node | The core measurement — Stage 2 |
| `EXPLAIN (ANALYZE, BUFFERS)` | Same as ANALYZE + trivial overhead | I/O vs CPU split, spills | Always pair with ANALYZE |
| `track_io_timing = on` | Small per-query overhead | Real milliseconds in I/O | When distinguishing I/O-bound from CPU-bound |
| `HypoPG` hypothetical index | Near-zero (no build) | Whether planner *would* use an index | Before paying a real `CREATE INDEX` |
| `CREATE INDEX CONCURRENTLY` | Minutes on a large table + write-tax forever | Whether a real index helps | After HypoPG/estimate say it's worth it |

### Cost characteristics that shape the loop

| Concern | Detail |
|---|---|
| `EXPLAIN ANALYZE` on a 30s query | Takes 30 seconds — measurement is as slow as the query. Use `LIMIT` or a smaller representative window when iterating. |
| `EXPLAIN ANALYZE` on DML | Performs the write. Always wrap `INSERT/UPDATE/DELETE` in `BEGIN; ... ROLLBACK;` |
| Per-node timing overhead | `ANALYZE` adds a `gettimeofday()` call per row on some platforms — a plan can look ~10-30% slower under ANALYZE than in production. Compare *relative* node costs, not absolute wall-clock, when timing overhead is high. |
| Cold vs warm cache | First run reads from disk (`shared read`), second from buffer cache (`shared hit`). Run 2-3× and note both — production may be either state. |
| `pg_stat_statements` memory | Bounded by `pg_stat_statements.max` (default 5000 entries). Beyond that, least-used entries are evicted — a rare hot query can fall off. |

### The signal-to-effort ranking of the stages

| Stage | Effort | Signal | Notes |
|---|---|---|---|
| 1. Identify (pg_stat_statements) | Seconds | Highest | Prevents optimising the wrong query entirely |
| 4. Estimate-vs-actual gap | Seconds (read the plan) | Highest | Classifies the problem: stats vs access-path |
| 3. Locate dominant node | Minutes (arithmetic) | High | self-time × loops points to the one node |
| 2. Measure (EXPLAIN ANALYZE) | Query runtime | High | The ground truth all hypotheses check against |
| 5-6. Hypothesise + one change | Varies | Compounds | Each measured change teaches the data shape |

The two highest-signal, lowest-effort moves — ranking by total time and reading the estimate gap — are exactly the two beginners skip. That is why the methodology front-loads them.

---

## 11. Node.js Integration

The methodology is not a one-off ritual — it belongs in code. Wrap `EXPLAIN` in helpers so you can run the loop programmatically, capture plans over time, and gate deploys on regressions.

### 11.1 Capturing a plan as JSON

```javascript
import pg from 'pg';
const { Pool } = pg;
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

// Stage 2 — measure. Returns the machine-readable plan for a candidate query.
async function explainAnalyze(sql, params = []) {
  // FORMAT JSON gives structured nodes you can walk, diff, and assert on.
  const { rows } = await pool.query(
    `EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) ${sql}`,
    params
  );
  return rows[0]['QUERY PLAN'][0];   // { Plan: {...}, "Planning Time":.., "Execution Time":.. }
}

// Example: measure a candidate before shipping it
const plan = await explainAnalyze(
  `SELECT o.id, o.total_amount, u.email
     FROM orders o
     INNER JOIN users u ON u.id = o.user_id
    WHERE o.status = $1
      AND o.created_at >= NOW() - ($2 || ' days')::INTERVAL`,
  ['completed', 30]
);
console.log('exec ms:', plan['Execution Time']);
```

### 11.2 Stage 1 — find the worst offenders from pg_stat_statements

```javascript
// Identify: never optimise a query you picked by gut feel.
async function topQueriesByTotalTime(limit = 10) {
  const { rows } = await pool.query(
    `SELECT query,
            calls,
            round(total_exec_time::numeric, 1)          AS total_ms,
            round(mean_exec_time::numeric, 2)           AS mean_ms,
            round((100 * total_exec_time /
                   sum(total_exec_time) OVER ())::numeric, 1) AS pct_total
       FROM pg_stat_statements
      ORDER BY total_exec_time DESC
      LIMIT $1`,
    [limit]
  );
  return rows;   // pct_total tells you where the wall-clock actually goes
}
```

### 11.3 Stage 4 — flag estimate-vs-actual gaps automatically

```javascript
// Walk the JSON plan tree and surface nodes where the planner guessed wrong.
function findEstimationGaps(node, threshold = 10, out = []) {
  const est = node['Plan Rows'];
  const act = node['Actual Rows'];
  const loops = node['Actual Loops'] ?? 1;
  const actualTotal = act * loops;
  if (est > 0 && actualTotal > 0) {
    const ratio = Math.max(actualTotal / est, est / actualTotal);
    if (ratio >= threshold) {
      out.push({
        node: node['Node Type'],
        relation: node['Relation Name'],
        estimated: est,
        actual: actualTotal,
        ratio: Math.round(ratio),
      });
    }
  }
  for (const child of node['Plans'] ?? []) findEstimationGaps(child, threshold, out);
  return out;
}

const gaps = findEstimationGaps(plan.Plan);
// gaps → e.g. [{ node:'Seq Scan', relation:'order_items', estimated:120, actual:840000, ratio:7000 }]
// Ratio ≫ 1 ⇒ run ANALYZE / raise statistics target / rethink the predicate.
```

### 11.4 Regression gate for CI

```javascript
// Fail the build if a known-hot query's plan degrades (e.g. index scan → seq scan).
async function assertUsesIndex(sql, params, expectedIndex) {
  const plan = await explainAnalyze(sql, params);
  const flat = JSON.stringify(plan);
  if (!flat.includes(expectedIndex)) {
    throw new Error(
      `Plan regression: expected index ${expectedIndex} not used. ` +
      `Exec time ${plan['Execution Time']}ms.`
    );
  }
}
```

**$1 parameterisation note**: always pass values through `$1, $2, …`, never string-concatenate. Beyond injection safety, parameterised queries share one entry in `pg_stat_statements` (constants are normalised to `$n`), so Stage 1 aggregates all executions of the same query shape instead of scattering them across thousands of rows.

**ORM note**: most ORMs expose a raw escape hatch (`prisma.$queryRaw`, `sequelize.query`, `knex.raw`, `dataSource.query`) — run `EXPLAIN (ANALYZE, BUFFERS)` through it against the ORM-generated SQL, because the SQL the ORM emits is frequently not the SQL you imagine. See Section 12.

---

## 12. ORM Comparison

The optimisation methodology is ORM-agnostic in principle, but each ORM hides the plan behind a different wall. The universal move: capture the real SQL the ORM emits, then run `EXPLAIN (ANALYZE, BUFFERS)` on that exact string.

### Prisma

```typescript
import { PrismaClient } from '@prisma/client';
const prisma = new PrismaClient({ log: [{ emit: 'event', level: 'query' }] });

// Capture the emitted SQL + params so you can EXPLAIN it yourself.
prisma.$on('query', (e) => {
  console.log('SQL   :', e.query);
  console.log('params:', e.params);
  console.log('ms    :', e.duration);   // Prisma's own timing ≈ Stage 2 signal
});

// EXPLAIN through the raw escape hatch on the ORM-generated query
const plan = await prisma.$queryRawUnsafe(
  `EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)
   SELECT "id","total_amount" FROM "orders" WHERE "status" = $1`,
  'completed'
);
```

**Where it breaks**: Prisma's default relation loading can split one logical read into several round trips (or use correlated subqueries), so `mean_exec_time` in `pg_stat_statements` is spread across multiple statements — Stage 1 ranking looks deceptively cheap per statement. Check `calls` too.

**Verdict**: Excellent query logging; use `$queryRaw` to run EXPLAIN on the generated SQL.

---

### Drizzle ORM

```typescript
import { sql } from 'drizzle-orm';
import { db } from './db';

// Drizzle exposes .toSQL() — read the exact string without executing.
const q = db.select().from(orders).where(eq(orders.status, 'completed'));
const { sql: text, params } = q.toSQL();
console.log(text, params);

// Run EXPLAIN ANALYZE via the sql template tag
const plan = await db.execute(
  sql`EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) ${sql.raw(text)}`
);
```

**Where it breaks**: Nothing structural — Drizzle is thin. The risk is that `.toSQL()` is trivial to skip, so people never look at the plan.

**Verdict**: Best transparency. `.toSQL()` gives you the exact string for Stage 2 with zero friction.

---

### Sequelize

```javascript
const { QueryTypes } = require('sequelize');

// benchmark:true + a logging hook surfaces per-query timing (Stage 1 candidate signal)
const sequelize = new Sequelize(url, {
  benchmark: true,
  logging: (sqlText, ms) => console.log(`${ms}ms`, sqlText),
});

// EXPLAIN through raw query on the generated SQL
const plan = await sequelize.query(
  `EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)
   SELECT o.id FROM orders o INNER JOIN users u ON u.id = o.user_id
   WHERE o.status = :status`,
  { replacements: { status: 'completed' }, type: QueryTypes.SELECT }
);
```

**Where it breaks**: Eager loading (`include`) generates large multi-JOIN statements that fan out rows; a query that looks fine per-row in code produces a plan with a huge `Actual Rows` at the top node. Always EXPLAIN the `include`-heavy queries.

**Verdict**: `benchmark:true` for Stage 1; `sequelize.query()` for Stage 2 EXPLAIN.

---

### TypeORM

```typescript
// DataSource with logging captures the SQL; getQueryAndParameters() gives it without executing.
const qb = dataSource.getRepository(Order)
  .createQueryBuilder('o')
  .innerJoin('o.user', 'u')
  .where('o.status = :status', { status: 'completed' });

const [text, params] = qb.getQueryAndParameters();

const plan = await dataSource.query(
  `EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) ${text}`, params);
```

**Where it breaks**: `maxExecutionTime` option warns on slow queries but does not give you a plan. `getQueryAndParameters()` is the key method — it exposes the exact SQL for EXPLAIN.

**Verdict**: `getQueryAndParameters()` + `dataSource.query()` gives full methodology access.

---

### Knex.js

```javascript
const knex = require('knex')({ client: 'pg', connection: url });

// .toSQL().toNative() gives the exact string + $1-style bindings
const built = knex('orders').where('status', 'completed').select('id').toSQL().toNative();
console.log(built.sql, built.bindings);

const plan = await knex.raw(
  `EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) ${built.sql}`, built.bindings);
```

**Where it breaks**: Nothing — Knex is a query builder, not an abstraction. `.toSQL().toNative()` is the most direct path to the exact string of any ORM here.

**Verdict**: Most SQL-transparent. `.toSQL().toNative()` → `knex.raw('EXPLAIN ...')` is the cleanest loop.

---

### ORM Summary Table

| ORM | Get exact SQL without executing | Per-query timing (Stage 1) | Run EXPLAIN ANALYZE (Stage 2) | Transparency |
|-----|--------------------------------|----------------------------|-------------------------------|--------------|
| Prisma | Query event log | `$on('query')` duration | `$queryRawUnsafe` | Good |
| Drizzle | `.toSQL()` | wrap + time | `db.execute(sql\`EXPLAIN...\`)` | Best |
| Sequelize | `logging` hook (executes) | `benchmark: true` | `sequelize.query()` | Medium |
| TypeORM | `getQueryAndParameters()` | `maxExecutionTime` warn | `dataSource.query()` | Good |
| Knex | `.toSQL().toNative()` | wrap + time | `knex.raw('EXPLAIN ...')` | Best |

The single rule across all five: **EXPLAIN the SQL the ORM actually emits, not the SQL you think it emits.**

---

## 13. Practice Exercises

### Exercise 1 — Basic

You suspect a reporting query is slow but you are guessing which one. Using `pg_stat_statements`, write a query that lists the top 5 statements by **total** execution time, showing `calls`, `mean_exec_time`, and each statement's percentage of all execution time.

Then explain, in one sentence, why ordering by `total_exec_time` finds different queries than ordering by `mean_exec_time`.

```sql
-- Write your query here
```

---

### Exercise 2 — Intermediate

Given this query and its plan fragment:

```sql
SELECT u.id, u.email, COUNT(o.id) AS order_count
FROM users u
INNER JOIN orders o ON o.user_id = u.id
WHERE o.status = 'completed'
  AND o.created_at >= NOW() - INTERVAL '90 days'
GROUP BY u.id, u.email;
```

```
Seq Scan on orders o  (cost=0.00..48210 rows=210 width=24)
                      (actual time=0.03..812.4 rows=486293 loops=1)
  Filter: (status = 'completed' AND created_at >= ...)
  Rows Removed by Filter: 1408100
```

1. Identify the bottleneck node.
2. Name the estimate-vs-actual gap (give the ratio, roughly).
3. State one hypothesis for the gap and the single change you would test first.
4. Write the `EXPLAIN` command you would run to confirm your fix, including BUFFERS.

```sql
-- Write your query here
```

---

### Exercise 3 — Hard

A query joins `order_items` (80M rows) to `products` and `categories`, then aggregates revenue per category for `payments` that succeeded in the last month. It runs in 42 seconds. The plan shows a `Hash Join` whose `Hash Batches` is 16 and a top-level `Sort` that spills (`Sort Method: external merge  Disk: 512000kB`).

Write out the full methodology you would follow:
1. The exact statement to capture the baseline plan.
2. Which two numbers in the plan you read first, and what each one would tell you.
3. Two independent hypotheses (one about memory, one about access path), and the single change you would test for each — **one change at a time**.
4. How you would measure whether each change helped, controlling for cold vs warm cache.

```sql
-- Write your query here
```

---

### Exercise 4 — Interview / Production Grade

Production alert: p99 latency on the `sessions`/`users` login-audit query tripled overnight. Nothing was deployed. Using only the methodology (identify → EXPLAIN(ANALYZE,BUFFERS) → bottleneck → hypothesis → test → measure), write the sequence of SQL commands and the reasoning you would run, in order, to (a) confirm which query regressed, (b) prove whether the plan changed, and (c) determine whether stale statistics (a large estimate-vs-actual gap that was not there before) are the cause. End with the one-line command that would test the "stale stats" hypothesis.

```sql
-- Write your query here
```

---

## 14. Interview Questions

### Junior Level

**Q: What is the difference between `EXPLAIN` and `EXPLAIN ANALYZE`?**

A: `EXPLAIN` shows the planner's chosen plan and its *estimated* costs and row counts without running the query. `EXPLAIN ANALYZE` actually **executes** the query and reports the *real* execution time and *actual* row counts per node. You need ANALYZE to see the truth; plain EXPLAIN only shows the guess. Because ANALYZE runs the query, on a DML statement (`INSERT/UPDATE/DELETE`) you must wrap it in `BEGIN; ... ROLLBACK;` so you do not perform the write, and on a 30-second query the EXPLAIN itself takes 30 seconds.

- *Follow-up: "Why also add BUFFERS?"* — BUFFERS shows how many blocks were read from the cache (`shared hit`) versus disk (`shared read`), which tells you whether the node is I/O-bound or CPU-bound — the same wall-clock time means very different fixes in each case.

---

**Q: A query is slow. Where do you start?**

A: Not by editing the query. First **identify** whether this query even matters — query `pg_stat_statements` ordered by `total_exec_time` to confirm it is actually a top consumer of database time. Optimising a query that runs twice a day wastes effort no matter how slow it is. Only once it's confirmed as a real cost do I run `EXPLAIN (ANALYZE, BUFFERS)` to measure it.

- *Follow-up: "What's the difference between total and mean time here?"* — A query with high *mean* time is individually slow; a query with high *total* time dominates the server's workload (it may be fast but called millions of times). You usually want to fix high-total-time queries first, because that is where the wall-clock actually goes.

---

### Mid Level

**Q: You see a node estimated to return 100 rows but it actually returned 900,000. What does that gap mean and what do you do?**

A: A large estimate-vs-actual gap means the planner is working from bad information, so every decision downstream of that node (join algorithm, join order, whether to use an index) is likely wrong — it may have chosen a Nested Loop expecting 100 iterations and got 900,000. The gap is the single most diagnostic signal in a plan. First hypothesis: **stale statistics** — run `ANALYZE <table>` and re-plan. If the gap persists, the cause is often a correlated or complex predicate the default statistics can't capture; raise the column's statistics target (`ALTER TABLE ... ALTER COLUMN ... SET STATISTICS 1000`) or add extended statistics (`CREATE STATISTICS`) for correlated columns, then `ANALYZE` again.

- *Follow-up: "How would you confirm the fix actually worked?"* — Re-run `EXPLAIN (ANALYZE, BUFFERS)` and check that (a) the estimate now tracks the actual, and (b) execution time dropped. Estimate alignment without a time drop means I fixed the stats but the real bottleneck was elsewhere.

---

### Principal Level

**Q: Design a repeatable optimisation methodology for a team, and explain why the order of steps matters.**

A: The pipeline is: **identify → measure → locate bottleneck → read estimate-vs-actual gap → hypothesise → test one change → re-measure**, then loop. The order matters because each step protects against a specific failure of intuition:

1. **Identify first** (`pg_stat_statements`) so nobody spends a day speeding up a query that contributes 0.1% of database time.
2. **Measure with `EXPLAIN (ANALYZE, BUFFERS)`** to get ground truth, not the estimated plan — the estimate is often exactly what's misleading you.
3. **Locate the dominant node** by self-time × loops, so effort goes to the one node that owns the runtime, not a cheap node that merely looks scary.
4. **Read the estimate-vs-actual gap** to *classify* the problem: a large gap means a statistics/planner problem (fix stats), a small gap with a slow node means an access-path/resource problem (fix indexes or `work_mem`). This classification decides which family of fix to reach for.
5. **One hypothesis, one change, re-measure.** Changing two things at once means you learn nothing — you can't attribute the delta.

The two highest-signal, lowest-effort steps (ranking by total time and reading the estimate gap) are the two beginners skip, so the methodology deliberately front-loads them.

- *Follow-up: "How do you keep this from regressing after you've left?"* — Snapshot the plan of each hot query in CI (Section 11.4). Assert the expected access path (e.g. the index name appears in the plan) and gate deploys on it, so a schema change or a stats drift that flips the plan back to a Seq Scan fails the build instead of paging someone at 2am.

---

## 15. Mental Model Checkpoint

1. You run `EXPLAIN ANALYZE` on a `DELETE` to see why it's slow. What have you just done to your data, and how should you have run it instead?

2. A plan node reads `rows=120 (actual rows=95 loops=8000)`. How many rows did this node actually produce in total, and is 120 a good estimate?

3. Two queries: query A has `mean_exec_time = 2000ms, calls = 5`; query B has `mean_exec_time = 4ms, calls = 40,000,000`. Which do you optimise first and why?

4. You cut a query from 900ms to 300ms on your second run without changing anything. What most likely happened, and why should you distrust that number?

5. Why does reading the estimate-vs-actual gap tell you *which kind* of fix to reach for (stats vs index vs memory), before you've formed any hypothesis?

6. You form two plausible hypotheses and apply both, then the query gets faster. What is the problem with what you just learned?

7. Under what circumstance is a node with `loops=50000` in a Nested Loop perfectly fine, and under what circumstance is it a disaster?

---

## 16. Quick Reference Card

```sql
-- ── THE LOOP ────────────────────────────────────────────────
-- identify → EXPLAIN(ANALYZE,BUFFERS) → bottleneck → hypothesis → test → measure

-- 1. IDENTIFY: what actually costs the server time?
SELECT query, calls,
       round(total_exec_time::numeric,1) AS total_ms,
       round(mean_exec_time::numeric,2)  AS mean_ms
FROM pg_stat_statements
ORDER BY total_exec_time DESC          -- total, not mean, finds the real cost
LIMIT 20;

-- 2. MEASURE: the ground truth (always ANALYZE + BUFFERS)
EXPLAIN (ANALYZE, BUFFERS) SELECT ...;
-- DML: wrap so you don't perform the write
BEGIN; EXPLAIN (ANALYZE, BUFFERS) DELETE FROM ...; ROLLBACK;

-- 3. LOCATE the dominant node: self-time × loops (not total subtree cost)

-- 4. READ the estimate-vs-actual gap  → classifies the problem:
--    est ≈ actual, node slow  → access path / resource (index, work_mem)
--    est ≪ actual (big gap)   → statistics / planner problem

-- ── SIGNALS TO WATCH ────────────────────────────────────────
-- rows estimate ≪ actual        → stale stats:  ANALYZE <table>;
-- Seq Scan + "Rows Removed by Filter" huge → missing/unused index
-- Nested Loop + Seq Scan inner  → missing index on inner join column
-- Hash Batches > 1              → work_mem too small (hash spills to disk)
-- Sort Method: external merge   → work_mem too small (sort spills to disk)
-- shared read ≫ shared hit      → I/O-bound (cold cache or too much data)

-- ── HYPOTHESIS TOOLS (cheap → expensive) ────────────────────
ANALYZE orders;                                    -- refresh stats (seconds)
ALTER TABLE orders ALTER COLUMN status SET STATISTICS 1000;  -- finer histogram
CREATE STATISTICS ord_corr (dependencies)
  ON status, created_at FROM orders;               -- correlated columns
SET LOCAL work_mem = '256MB';                      -- test a memory hypothesis
-- HypoPG: does the planner WOULD use this index? (no build cost)
SELECT * FROM hypopg_create_index('CREATE INDEX ON orders(user_id, status)');
CREATE INDEX CONCURRENTLY ON orders(user_id, status);  -- only after it's proven

-- ── DISCIPLINE ──────────────────────────────────────────────
-- Change ONE thing, re-measure. Run 2-3× (note cold vs warm cache).
-- Compare RELATIVE node cost, not absolute wall-clock (ANALYZE adds overhead).
```

---

## Connected Topics

- **Topic 17 — JOIN Performance Deep Dive**: Nested Loop vs Hash vs Merge — the estimate-vs-actual gap is what makes the planner pick the wrong one; this is the most common thing the methodology diagnoses.
- **Topic 44 — Indexing Strategy**: The "hypothesis" step almost always ends in an index decision; HypoPG lets you test one before paying to build it.
- **Topic 45 — EXPLAIN and EXPLAIN ANALYZE**: The full grammar of reading a plan — node types, cost model, self-time vs total time — that Stage 2 and 3 depend on.
- **Topic 46 — Statistics and the Query Planner**: Why the estimate-vs-actual gap appears, and how `ANALYZE`, `SET STATISTICS`, and `CREATE STATISTICS` close it.
- **Topic 47 — work_mem, Spills, and Sorts**: The resource-side hypotheses — Hash Batches > 1 and external-merge sorts — and how to test a `work_mem` change safely.
- **Topic 48 — pg_stat_statements**: Stage 1 in depth — normalisation of `$n` parameters, `total` vs `mean` time, and why parameterised queries aggregate correctly.
- **Topic 07 — NULL in Depth**: A frequent hidden cause of estimate gaps — NULL fractions and null-excluding predicates the planner models poorly.
- **Topic 20 — GROUP BY Fundamentals**: Aggregation after a fan-out join is a classic source of the top-node row explosion the methodology hunts for.
