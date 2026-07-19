# Topic 17 — JOIN Performance Deep Dive
### SQL Mastery Curriculum — Phase 3: JOINs — Complete Mastery

---

## 1. ELI5 — The Analogy That Makes It Click

Imagine you are matching up two stacks of paperwork: a stack of **shipping labels** and a stack of **customer address cards**. Every label has a customer number on it, and every address card has a customer number too. Your job: for each label, find the matching address card.

You have three ways to do this, and which one is fastest depends entirely on how big the stacks are and whether they are sorted.

**Way 1 — The one-at-a-time lookup (Nested Loop).** You pick up the first label, then flip through the address cards until you find the matching customer number, staple them together, and move to the next label. This is fast *only* if the address cards are in a filing cabinet sorted by customer number (an index) — then each lookup is instant. If the address cards are a shuffled heap and you have to check every single one for every single label, this is agony: 1,000 labels × 1,000 cards = a million card-checks.

**Way 2 — Build a quick directory (Hash Join).** Before you start, you go through the *smaller* stack once and pin every card to a giant pegboard, where the peg position is computed from the customer number. Now you pick up each label, compute its peg position, and instantly see which cards hang there. One pass to build the pegboard, one pass over the labels to probe it. No sorting needed. The catch: the pegboard has to fit on your wall. If the stack is so big the pegboard won't fit, you have to split the work into batches and shuffle papers to and from the floor (spill to disk) — much slower.

**Way 3 — Sort both, then zip (Merge Join).** You sort both stacks by customer number first. Now you hold one finger on each stack and walk down them together, like merging two sorted lists. Whenever the numbers match, staple. Because both are sorted, you never go backwards and never re-check. If the stacks *arrive* already sorted (because they came out of sorted filing cabinets — indexes), this is beautifully cheap. If you have to sort them first, that sorting cost might not be worth it.

The database planner is the office manager who looks at the stack sizes, checks whether sorted filing cabinets exist, estimates the effort of each of the three ways, and picks the cheapest. This topic is about understanding *how it picks*, *when it picks wrong*, and *how you force its hand*.

---

## 2. Connection to SQL Internals

A JOIN is a logical operation — "combine rows from two relations where a condition holds." But the planner must turn that logical request into a **physical plan**: a specific algorithm executed against heap pages, indexes, and memory. The three physical join algorithms are the heart of PostgreSQL's executor, and each touches a different set of internal subsystems.

**Nested Loop Join** touches the **buffer pool** and **B-tree indexes** most directly. For each outer row, it performs an index probe on the inner relation — descending the B-tree from root to leaf (typically 3–4 page reads), then fetching the matching heap tuple. The cost model multiplies the number of outer rows by the per-probe cost. If the inner side has no index, each probe becomes a full **sequential scan** of the inner heap — the classic O(N×M) disaster.

**Hash Join** builds an in-memory **hash table** in `work_mem`. It hashes the join key of the smaller (build) relation into buckets, then streams the larger (probe) relation through, hashing each row and checking the matching bucket. If the build side exceeds `work_mem`, PostgreSQL switches to a **batched hybrid hash join**: it partitions both inputs into batches by a hash of the key, spills all but the first batch to temporary files on disk, and processes them one batch at a time. This spill is visible as `Batches > 1` in EXPLAIN and is a major performance cliff.

**Merge Join** relies on **sort order**. It needs both inputs sorted on the join key. That order comes from either an **explicit Sort node** (which itself uses `work_mem`, spilling to disk with an external merge sort when it overflows) or, ideally, from an **Index Scan** that produces rows already in key order for free. The merge itself walks both streams with two cursors in O(N+M).

Underneath all three, the **planner's cost model** and **statistics** (`pg_statistic`, populated by `ANALYZE`) drive the decision. The planner estimates row counts (cardinality) and per-operation costs using tunables like `seq_page_cost`, `random_page_cost`, `cpu_tuple_cost`, and the size of `work_mem`. A bad estimate — usually a stale or missing statistic — produces a bad algorithm choice. Almost every "why is my join slow" investigation ends at one of: a missing index (forces Nested Loop with Seq Scan, or forces Hash/Merge when Nested Loop would win), a bad row estimate (wrong algorithm chosen), or `work_mem` too small (Hash spill / Sort spill).

---

## 3. Logical Execution Order Context

```
FROM tableA
JOIN tableB ON condition     ← the JOIN algorithm runs HERE, in the FROM phase
WHERE ...                     ← filters applied (often pushed INTO the join)
GROUP BY ...
HAVING ...
SELECT ...
DISTINCT
window functions
ORDER BY ...                  ← may reuse a Merge Join's sort order for free
LIMIT ...                     ← can make Nested Loop win by short-circuiting
```

Join performance is decided in the **FROM/JOIN phase**, but it is inseparable from clauses that logically come later:

- **WHERE predicates are pushed down** into the join or below it. A selective `WHERE o.status = 'new'` can shrink the outer relation *before* the join runs, changing which algorithm is cheapest. This is why you cannot reason about join algorithm choice without also looking at the WHERE clause.
- **ORDER BY interacts with Merge Join**: if the query needs output sorted on the join key, a Merge Join delivers that ordering as a byproduct — the planner may pick Merge Join precisely to avoid a later Sort node. Conversely, Hash Join destroys any input ordering.
- **LIMIT interacts with Nested Loop**: `LIMIT 10` lets a Nested Loop stop after producing 10 rows without scanning the whole outer relation. This makes Nested Loop attractive for "top N" queries even on large tables, because the plan is *incremental* — it produces rows as it goes. Hash Join must build its entire hash table before emitting the first row (it "blocks"), so a small LIMIT gives it no shortcut.
- **GROUP BY / aggregation** consumes the join output. A fan-out join (Topic 11, Topic 16) feeding an aggregate is a common reason a plan processes far more rows than the final result implies — and a reason the planner might choose a Hash Join to materialize the intermediate set efficiently.

The logical result of the join is identical across all three algorithms. Only cost, memory, and the *shape* of execution (blocking vs. incremental, sorted vs. unsorted output) differ.

---

## 4. What Is JOIN Performance (The Three Algorithms)?

"JOIN performance" is the study of the three physical algorithms PostgreSQL uses to execute any join, the conditions under which the planner selects each, and the levers you use to correct a bad choice. Every join in every plan is one of: **Nested Loop**, **Hash Join**, or **Merge Join**. They always produce the same rows; they differ radically in cost, memory footprint, and index dependency.

```sql
EXPLAIN (ANALYZE, BUFFERS, SETTINGS)
SELECT o.id, c.name
FROM orders o
JOIN customers c ON c.id = o.customer_id
WHERE o.status = 'completed';
```

The plan node that implements the join tells you which algorithm ran:

```
Hash Join                          │ ← the physical algorithm chosen (one of 3)
  Hash Cond: (o.customer_id = c.id)│ └── the equi-join predicate (equality only for Hash/Merge)
  ->  Seq Scan on orders o         │ ← the OUTER (probe) input — streamed
        Filter: (status = 'completed')│ └── pushed-down WHERE predicate
  ->  Hash                         │ ← builds the in-memory hash table
        ->  Seq Scan on customers c│ └── the BUILD (inner) input — smaller side hashed
```

### The three algorithms at a glance

```sql
-- NESTED LOOP: for each outer row, probe inner (ideally via index)
Nested Loop
  ->  <outer scan>                 -- drives the loop; runs once
  ->  Index Scan on inner          -- runs ONCE PER OUTER ROW (loops=N)
        Index Cond: (inner.key = outer.key)

-- HASH JOIN: build hash on smaller side, probe with larger
Hash Join
  Hash Cond: (probe.key = build.key)  -- EQUALITY ONLY
  ->  <probe scan>                    -- larger side, streamed once
  ->  Hash
        ->  <build scan>              -- smaller side, hashed into work_mem

-- MERGE JOIN: sort both sides, walk in lockstep
Merge Join
  Merge Cond: (left.key = right.key)  -- EQUALITY (also supports range merge)
  ->  <sorted left input>             -- Index Scan or Sort node
  ->  <sorted right input>            -- Index Scan or Sort node
```

Key structural facts to internalize:

- **Nested Loop** is the only algorithm that supports arbitrary join conditions (`<`, `>`, `BETWEEN`, function calls). Hash Join requires **equality**. Merge Join requires an **ordering operator** (equality, and it can also do range/merge conditions with sorted inputs).
- **Nested Loop** is *incremental* (produces rows immediately) and *asymmetric* (the two sides play different roles). **Hash Join** and **Merge Join** are, to varying degrees, *blocking* — Hash must build fully before probing; Merge must have both inputs sorted before merging.
- Only **Nested Loop** genuinely benefits from an index on the **inner** relation. Hash Join can run entirely on sequential scans. Merge Join benefits from indexes that *provide sort order* (so it can skip the Sort node).

---

## 5. Why JOIN Performance Mastery Matters in Production

1. **The single most common "my query got slow overnight" cause.** A table grows past a threshold, statistics drift, or an index gets dropped — and the planner flips from Nested Loop to Hash Join, or from an indexed Nested Loop to a Seq-Scan Nested Loop. Query time jumps from 5ms to 30 seconds with no code change. If you cannot read EXPLAIN and identify which algorithm ran and why, you cannot diagnose this class of incident.

2. **Missing-index Nested Loops are catastrophic, not slow.** A Nested Loop over a large outer relation with a Seq Scan on the inner side is O(N×M). At 1M outer rows and a 1M-row inner scan, that is a trillion tuple comparisons. This is the difference between a query that finishes in 20ms and one that never finishes and holds a transaction open, blocking autovacuum and bloating the table.

3. **`work_mem` is a per-operation, per-connection landmine.** Hash Joins and Sorts each consume up to `work_mem`. A single complex query with three hash joins and two sorts can allocate `5 × work_mem`. Multiply by 200 concurrent connections and you can OOM the box. Setting `work_mem` too low causes disk spills (5–10× slowdowns); too high causes memory exhaustion. Understanding which join algorithm consumes memory, and how much, is an operational necessity.

4. **The planner is only as good as its statistics.** A 1000× cardinality misestimate (common with correlated columns, or after a bulk load without `ANALYZE`) makes the planner choose the wrong algorithm confidently. Knowing how to spot the estimated-vs-actual gap in EXPLAIN ANALYZE, and how to fix it (`ANALYZE`, extended statistics, higher statistics targets), is core production skill.

5. **Forcing an algorithm is sometimes the only fast fix.** When the planner is wrong and you cannot immediately fix statistics, `enable_nestloop`, `enable_hashjoin`, `enable_mergejoin`, and `SET work_mem` per-session let you force the correct plan. Used surgically, these are legitimate production tools — but only if you understand what you are forcing and why.

---

## 6. Deep Technical Content

### 6.1 Nested Loop Join — Mechanics

The Nested Loop is the simplest algorithm and the mental model for all joins:

```
for each row R_outer in OUTER:
    for each row R_inner in INNER matching the condition:
        emit (R_outer, R_inner)
```

The critical variable is **how the inner match is found**. There are two sub-cases, and they differ by orders of magnitude:

**(a) Indexed Nested Loop (the fast case).** The inner relation has a usable index on the join key. Each outer row triggers an index probe: descend the B-tree (≈ 3–4 buffer reads on a warm cache), fetch matching heap tuples. Cost ≈ `outer_rows × (index_descent + matched_tuples)`. This is the plan you *want* when the outer relation is small (a handful to a few thousand rows).

```sql
-- Small filtered outer + PK/indexed inner → indexed Nested Loop
SELECT o.id, c.name
FROM orders o
JOIN customers c ON c.id = o.customer_id
WHERE o.id = 91823;    -- outer collapses to ~1 row
```

```
Nested Loop  (actual rows=1 loops=1)
  ->  Index Scan using orders_pkey on orders o   (actual rows=1 loops=1)
        Index Cond: (id = 91823)
  ->  Index Scan using customers_pkey on customers c  (actual rows=1 loops=1)
        Index Cond: (id = o.customer_id)
```

Note `loops=1` on the inner — because the outer produced one row. When the outer produces N rows, the inner Index Scan shows `loops=N`, and its reported `actual time` is **per-loop** (multiply by loops for total).

**(b) Non-indexed Nested Loop (the catastrophe).** No usable index on the inner join key. Each outer row triggers a full Seq Scan of the inner relation. Cost ≈ `outer_rows × inner_rows`. At 10K × 10K this is 100M comparisons (seconds); at 1M × 1M it is 10^12 (effectively never finishes).

```
Nested Loop  (actual rows=... loops=1)
  ->  Seq Scan on orders o          (actual rows=50000 loops=1)
  ->  Seq Scan on customers c       (actual rows=1000000 loops=50000)  ← 50000 full scans!
        Filter: (id = o.customer_id)
```

Seeing `Seq Scan ... loops=<big number>` on the inner side of a Nested Loop is the number-one red flag in join performance. The fix is almost always: add an index on the inner join column, or make the planner choose Hash Join.

**When the planner picks Nested Loop:**
- The outer relation is small (after filtering) — few loop iterations.
- The inner relation has an index on the join key — cheap per-probe.
- There is a `LIMIT` that lets the loop stop early.
- The join condition is **not** an equi-join (Hash/Merge can't be used at all).

### 6.2 Hash Join — Mechanics

```
BUILD phase:   scan smaller relation; for each row, compute hash(join_key),
               insert into hash table bucket in work_mem
PROBE phase:   scan larger relation; for each row, compute hash(join_key),
               look up matching bucket, emit matched pairs
```

The **build side is always the (estimated) smaller relation** — this minimizes memory. In EXPLAIN, the `Hash` node wraps the build input; the sibling scan is the probe input.

**work_mem and batching.** If the entire build-side hash table fits in `work_mem`, you get `Batches: 1` — a single in-memory pass, the fast path. If it does not fit, PostgreSQL uses a **hybrid hash join**: it partitions both inputs into `2^n` batches by hash bits, keeps batch 0 in memory, and spills batches 1..n-1 to temporary files. Each batch is then loaded and joined in turn. This is visible as `Batches: 4` (or 8, 16...) and `Memory Usage` plus temp file I/O.

```
Hash  (actual rows=2000000 loops=1)
  Buckets: 65536  Batches: 8  Memory Usage: 30720kB
                          ↑ Batches > 1 = SPILLING TO DISK
```

Batching typically costs 2–10× the in-memory time because of the extra write+read of spilled tuples. The fixes: raise `work_mem` for the session, reduce the build-side row count (more selective filter), or reduce the build-side width (select fewer columns so tuples are narrower).

**Hash Join is equality-only.** `Hash Cond` always uses `=`. You cannot hash-join on `a.x > b.y`. A hash table is a point-lookup structure; ranges have no meaning in it.

**Hash Join blocks on build.** No output row can be produced until the entire build side is hashed. This is why `LIMIT 1` does not speed up a Hash Join's build phase — the first row still requires a full build scan. (The probe side, however, is streamed and can short-circuit under LIMIT once the hash exists.)

**When the planner picks Hash Join:**
- Equi-join between two relations where at least one is large.
- No index that makes an indexed Nested Loop cheaper, or the outer is too big for Nested Loop to win.
- Enough `work_mem` (or willingness to batch) to hold the build side.

### 6.3 Merge Join — Mechanics

```
sort LEFT on join_key   (or read pre-sorted from an index)
sort RIGHT on join_key  (or read pre-sorted from an index)
cursorL, cursorR = start of each
while both have rows:
    if keyL == keyR:  emit pair(s), advance the appropriate cursor(s)
    elif keyL < keyR: advance cursorL
    else:             advance cursorR
```

Merge Join's superpower is that it needs **O(1) working memory** for the merge itself (just two cursors) — unlike Hash Join it does not hold a whole relation in memory. Its cost is dominated by getting both inputs **sorted**.

**Two sources of sort order:**

**(a) Free order from an index.** If both join keys are indexed and the planner uses Index Scans, rows arrive in key order with **no Sort node**. This is the ideal Merge Join — cheap and memory-light, excellent for very large tables where a Hash Join would spill.

```
Merge Join  (actual rows=1000000 loops=1)
  Merge Cond: (a.key = b.key)
  ->  Index Scan using a_key_idx on a    (actual rows=1000000 loops=1)   ← pre-sorted
  ->  Index Scan using b_key_idx on b    (actual rows=1000000 loops=1)   ← pre-sorted
```

**(b) Explicit Sort nodes.** If no index provides order, the planner inserts Sort nodes. Each sort is O(N log N) and consumes `work_mem` (spilling to a disk-based external merge sort if it overflows — visible as `Sort Method: external merge  Disk: 123456kB`).

```
Merge Join
  Merge Cond: (a.key = b.key)
  ->  Sort  (Sort Method: quicksort  Memory: 25kB)
        ->  Seq Scan on a
  ->  Sort  (Sort Method: external merge  Disk: 90112kB)   ← sort spilled to disk
        ->  Seq Scan on b
```

When both sides need sorting, Merge Join is usually *worse* than Hash Join. Merge Join wins when the sort is free (indexes) or when the sort output is needed anyway (an `ORDER BY` on the join key downstream).

**Merge Join and duplicates.** With duplicate keys on both sides, the merge must "rewind" the inner cursor to re-emit matches — PostgreSQL materializes runs of equal keys. Heavy duplication on both sides can degrade the neat O(N+M) toward the fan-out row count (which is inherent to the join, not a merge-specific penalty).

**When the planner picks Merge Join:**
- Both inputs are already sorted on the join key (indexes, or a prior sort).
- Both relations are large and a Hash Join would spill badly.
- The query needs the output ordered on the join key (ORDER BY reuse).

### 6.4 How the Planner Chooses — The Cost Model

The planner enumerates candidate plans and assigns each a cost in abstract units, then picks the minimum. The relevant cost parameters:

| Parameter | Default | Meaning |
|-----------|---------|---------|
| `seq_page_cost` | 1.0 | Cost of a sequential page read |
| `random_page_cost` | 4.0 | Cost of a random page read (index probes) |
| `cpu_tuple_cost` | 0.01 | Cost to process one row |
| `cpu_index_tuple_cost` | 0.005 | Cost to process one index entry |
| `cpu_operator_cost` | 0.0025 | Cost to evaluate one operator/function |
| `work_mem` | 4MB | Memory per sort/hash before spilling |
| `effective_cache_size` | 4GB | Planner's assumption about OS+PG cache (affects index vs seq choice) |

The decisive input is the **row estimate (cardinality)**. Nested Loop cost scales with `outer_rows`; if the planner thinks the outer is 5 rows but it is actually 5 million, it will pick Nested Loop and the query will melt. This is why *estimates matter more than the cost constants*: a wrong cardinality picks the wrong algorithm regardless of tuning.

```sql
-- See the estimates that drove the choice:
EXPLAIN SELECT ...   -- shows rows=<estimate>
-- Compare to reality:
EXPLAIN ANALYZE SELECT ...   -- shows rows=<estimate> ... actual rows=<truth>
```

A large gap between `rows=` (estimate) and `actual rows=` is the fingerprint of a statistics problem.

### 6.5 Reading Which Algorithm Was Chosen

The join node name is explicit in every plan:

| Plan node text | Algorithm |
|----------------|-----------|
| `Nested Loop` | Nested Loop Join |
| `Hash Join` + `Hash` child node | Hash Join |
| `Merge Join` | Merge Join |

Supporting evidence within the node:
- `Hash Cond:` / `Merge Cond:` / `Join Filter:` / `Index Cond:` tell you the predicate and how it is applied.
- `loops=N` on an inner node reveals a Nested Loop's iteration count.
- `Buckets / Batches / Memory Usage` on a `Hash` node reveal build size and spill.
- `Sort Method:` on a `Sort` node reveals in-memory vs. external (disk) sort.
- `Rows Removed by Filter:` / `Rows Removed by Join Filter:` reveal wasted work.

### 6.6 Forcing an Algorithm — The enable_* Flags

PostgreSQL exposes session-level flags that discourage (by adding a huge cost penalty, not truly disabling) each algorithm. They are invaluable for **diagnosis** ("what would the plan cost if it couldn't use a hash join?") and occasionally for **production overrides**.

```sql
-- Diagnostic: force the planner off Nested Loop to compare
SET enable_nestloop = off;
EXPLAIN ANALYZE SELECT ... ;
RESET enable_nestloop;

-- The full set:
SET enable_nestloop  = off;   -- discourage Nested Loop
SET enable_hashjoin  = off;   -- discourage Hash Join
SET enable_mergejoin = off;   -- discourage Merge Join
SET enable_seqscan   = off;   -- discourage Seq Scan (favor index paths)
SET enable_sort      = off;   -- discourage explicit Sorts
```

These are **not true toggles** — with `enable_nestloop=off`, if a Nested Loop is the *only* possible plan (e.g., a non-equi-join), the planner still uses it, just with a penalized cost. They set a hint, not a hard constraint.

**Per-session work_mem** is the other big lever, and often the cleaner fix:

```sql
SET work_mem = '256MB';       -- give this session's hashes/sorts room
-- run the heavy analytical query
RESET work_mem;
```

Never raise `work_mem` globally to a large value on an OLTP box — it is per-operation × per-connection and will OOM you. Raise it per-session for known-heavy batch/analytics queries, or use a dedicated role/pool with a higher setting.

### 6.7 Join Order vs. Join Algorithm

Two distinct decisions the planner makes, often conflated:
1. **Join order** — for 3+ tables, which pair to join first (Topic 16). Controlled/observed via `join_collapse_limit`, `from_collapse_limit`, and (as a blunt hammer) `SET join_collapse_limit = 1` to force the written order.
2. **Join algorithm** — for each pairwise join, which of the three algorithms. The subject of this topic.

They interact: a good join order keeps intermediate results small, which in turn makes a cheap Nested Loop or a non-spilling Hash Join possible. A bad order produces a huge intermediate that forces a spilling Hash Join or a Seq-Scan Nested Loop.

### 6.8 Parallel Joins

Since PostgreSQL 9.6+, joins can run in parallel across worker processes. You will see `Parallel Hash Join`, `Parallel Seq Scan`, and a `Gather` node collecting worker output:

```
Gather  (workers planned: 2)
  ->  Parallel Hash Join
        Hash Cond: (o.customer_id = c.id)
        ->  Parallel Seq Scan on orders o
        ->  Parallel Hash
              ->  Parallel Seq Scan on customers c
```

`Parallel Hash Join` (note the `Parallel Hash` build node) shares a *single* hash table across workers — a big improvement over the older per-worker hash. Parallelism is governed by `max_parallel_workers_per_gather`, `parallel_setup_cost`, `parallel_tuple_cost`, and the table's `min_parallel_table_scan_size`. Merge and Nested Loop can also be parallelized on the outer side. Parallelism helps big analytical joins; it adds startup overhead that is not worth it for small OLTP queries (the planner accounts for this).

### 6.9 Memoize — Caching Inner Results in a Nested Loop

PostgreSQL 14+ can wrap the inner side of a Nested Loop in a **Memoize** node, which caches results of inner probes keyed by the join parameter. When the outer side has many duplicate keys, repeated inner probes hit the cache instead of re-scanning:

```
Nested Loop
  ->  Seq Scan on orders o
  ->  Memoize
        Cache Key: o.customer_id
        Hits: 48213  Misses: 1787  Evictions: 0  Memory Usage: 421kB
        ->  Index Scan using customers_pkey on customers c
              Index Cond: (id = o.customer_id)
```

High `Hits` relative to `Misses` means Memoize turned an expensive repeated-probe Nested Loop into something competitive with a Hash Join, while keeping the incremental/LIMIT-friendly nature of a Nested Loop. It is bounded by `work_mem` (`hash_mem_multiplier`), and evictions signal the cache is too small for the key distribution.

### 6.10 Anti-Joins, Semi-Joins, and Their Algorithms

`EXISTS`, `NOT EXISTS`, `IN`, and `NOT IN` become **semi-joins** and **anti-joins** internally (Topic 19, 27). These use the *same three physical algorithms* but stop at the first match (semi) or emit only non-matches (anti):

```
Hash Anti Join     -- NOT EXISTS with an equi-key
Hash Semi Join     -- EXISTS / IN with an equi-key
Nested Loop Semi Join
Merge Anti Join
```

A `Hash Semi Join` never multiplies rows (unlike a plain INNER JOIN that needs DISTINCT), which is often why `EXISTS` outperforms `JOIN ... DISTINCT`. The algorithm-selection rules are the same as for inner joins.

---

## 7. EXPLAIN — Join Algorithm in the Plan

### 7.1 Indexed Nested Loop (fast, small outer)

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT o.id, o.total_amount, c.name
FROM orders o
JOIN customers c ON c.id = o.customer_id
WHERE o.customer_id = 4471;
```

```
Nested Loop  (cost=0.72..48.9 rows=12 width=44)
             (actual time=0.031..0.058 rows=9 loops=1)
  ->  Index Scan using orders_customer_id_idx on orders o
        (cost=0.43..36.2 rows=12 width=20)
        (actual time=0.019..0.033 rows=9 loops=1)
        Index Cond: (customer_id = 4471)
  ->  Index Scan using customers_pkey on customers c
        (cost=0.29..1.06 rows=1 width=32)
        (actual time=0.002..0.002 rows=1 loops=9)   ← 9 probes, one per outer row
        Index Cond: (id = 4471)
  Buffers: shared hit=31
Planning Time: 0.14 ms
Execution Time: 0.09 ms
```

Read it: outer scan finds 9 orders via `orders_customer_id_idx`. For each (`loops=9`), the inner probes `customers` by PK. All buffer hits, sub-millisecond. This is the textbook fast join.

### 7.2 Hash Join (large equi-join, fits in memory)

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT o.id, c.name
FROM orders o
JOIN customers c ON c.id = o.customer_id;
```

```
Hash Join  (cost=3145.00..64210.00 rows=2000000 width=44)
           (actual time=28.4..640.2 rows=2000000 loops=1)
  Hash Cond: (o.customer_id = c.id)
  ->  Seq Scan on orders o  (cost=0.00..41310.00 rows=2000000 width=20)
                            (actual time=0.01..120.4 rows=2000000 loops=1)
  ->  Hash  (cost=1770.00..1770.00 rows=100000 width=32)
             (actual time=28.1..28.1 rows=100000 loops=1)
        Buckets: 131072  Batches: 1  Memory Usage: 6528kB   ← Batches:1 = no spill
        ->  Seq Scan on customers c  (cost=0.00..1770.00 rows=100000 width=32)
                                     (actual time=0.01..8.9 rows=100000 loops=1)
  Buffers: shared hit=512 read=42800
Planning Time: 0.21 ms
Execution Time: 705.6 ms
```

Read it: `customers` (100K rows, smaller) is the build side inside the `Hash` node; `orders` (2M rows) is streamed as the probe. `Batches: 1` — the 6.5MB hash table fit in `work_mem`, no disk spill. This is the correct choice for an unfiltered two-large-table equi-join.

### 7.3 Hash Join spilling (work_mem too small)

```
Hash Join  (actual time=210..9800 rows=2000000 loops=1)
  Hash Cond: (o.customer_id = c.id)
  ->  Seq Scan on orders o  (actual rows=2000000 loops=1)
  ->  Hash  (actual rows=2000000 loops=1)
        Buckets: 65536  Batches: 16  Memory Usage: 4096kB   ← Batches:16 = SPILL
        ->  Seq Scan on customers_big c  (actual rows=2000000 loops=1)
  Buffers: shared hit=800 read=90000, temp read=52000 written=52000   ← temp = disk spill
Execution Time: 10240 ms
```

Read it: `Batches: 16` and `temp read/written` prove the build side (2M rows) overflowed `work_mem` and spilled to 16 disk batches. Fix: `SET work_mem = '256MB'` collapses this toward `Batches: 1` and cuts time by ~5×.

### 7.4 Merge Join with free index sort

```sql
EXPLAIN (ANALYZE)
SELECT a.id, b.id
FROM big_a a
JOIN big_b b ON b.a_id = a.id;   -- both sides indexed on the key
```

```
Merge Join  (cost=1.14..210500 rows=5000000 width=16)
            (actual time=0.05..3100 rows=5000000 loops=1)
  Merge Cond: (a.id = b.a_id)
  ->  Index Scan using big_a_pkey on big_a a
        (actual rows=2000000 loops=1)              ← pre-sorted, NO Sort node
  ->  Index Scan using big_b_a_id_idx on big_b b
        (actual rows=5000000 loops=1)              ← pre-sorted, NO Sort node
Execution Time: 3400 ms
```

Read it: no `Sort` nodes — both Index Scans deliver rows already ordered on the key. The merge walks both in lockstep with O(1) memory. This beats a Hash Join here because a 5M-row hash would spill.

### 7.5 Merge Join forced to sort (usually a warning sign)

```
Merge Join
  Merge Cond: (a.k = b.k)
  ->  Sort  (Sort Method: external merge  Disk: 71680kB)   ← spilled sort
        ->  Seq Scan on a  (actual rows=2000000)
  ->  Sort  (Sort Method: quicksort  Memory: 24kB)
        ->  Seq Scan on b  (actual rows=500)
```

Read it: the left input needed a disk-spilling sort (`external merge ... Disk`). Two Sorts on unindexed inputs usually mean a Hash Join would have been cheaper — check with `SET enable_mergejoin = off`.

### 7.6 The Nested Loop catastrophe (missing inner index)

```
Nested Loop  (actual time=0.4..142000 rows=310 loops=1)   ← 142 SECONDS
  Join Filter: (c.id = o.customer_id)
  Rows Removed by Join Filter: 49999690
  ->  Seq Scan on orders o        (actual rows=50000 loops=1)
  ->  Seq Scan on customers c      (actual rows=1000 loops=50000)   ← 50000 full scans
```

Read it: `loops=50000` on an inner `Seq Scan`, plus a huge `Rows Removed by Join Filter`. The inner `customers` has no usable index for the join, so it is fully scanned 50,000 times. Fix: index `customers(id)` (a PK would exist normally) or force Hash Join.

### 7.7 The EXPLAIN symptom → cause → fix table

| Symptom in plan | Cause | Fix |
|-----------------|-------|-----|
| `Nested Loop` + inner `Seq Scan` with big `loops` | No index on inner join key | Add index on inner join column |
| `Hash ... Batches > 1` + `temp written` | Build side exceeds `work_mem` | Raise `work_mem`; narrow/filter build side |
| `Sort ... external merge Disk:` before Merge Join | Inputs not pre-sorted; sort spills | Add index for sort order, or raise `work_mem`, or prefer Hash |
| `rows=5` est vs `actual rows=5000000` | Stale/insufficient statistics | `ANALYZE`; raise stats target; extended statistics |
| Wrong algorithm despite good stats | Cost constants off for hardware (SSD) | Tune `random_page_cost` (≈1.1 on SSD) |
| `Rows Removed by Filter` huge | Join produces then discards rows | Push predicate earlier; add composite index |
| `Memoize` with low `Hits` | Cache ineffective for key distribution | Consider Hash Join (`enable_memoize=off` to test) |

---

## 8. Query Examples

### Example 1 — Basic: See the algorithm the planner picks

```sql
-- Simplest possible diagnosis: run EXPLAIN and name the join node.
EXPLAIN
SELECT o.id, c.name
FROM orders o
JOIN customers c ON c.id = o.customer_id
WHERE o.status = 'completed';
-- Look at the top join node: "Hash Join", "Nested Loop", or "Merge Join".
-- That single line tells you which of the three algorithms will run.
```

### Example 2 — Intermediate: Compare all three algorithms for one query

```sql
-- Force each algorithm in turn to compare their real cost on YOUR data.
-- (Diagnostic technique — not something you leave in production code.)

-- Baseline: let the planner choose
EXPLAIN (ANALYZE, BUFFERS)
SELECT o.id, c.name
FROM orders o JOIN customers c ON c.id = o.customer_id;

-- Force Hash Join only
SET enable_nestloop = off; SET enable_mergejoin = off;
EXPLAIN (ANALYZE, BUFFERS)
SELECT o.id, c.name
FROM orders o JOIN customers c ON c.id = o.customer_id;
RESET enable_nestloop; RESET enable_mergejoin;

-- Force Merge Join only
SET enable_nestloop = off; SET enable_hashjoin = off;
EXPLAIN (ANALYZE, BUFFERS)
SELECT o.id, c.name
FROM orders o JOIN customers c ON c.id = o.customer_id;
RESET enable_nestloop; RESET enable_hashjoin;

-- Force Nested Loop only
SET enable_hashjoin = off; SET enable_mergejoin = off;
EXPLAIN (ANALYZE, BUFFERS)
SELECT o.id, c.name
FROM orders o JOIN customers c ON c.id = o.customer_id;
RESET enable_hashjoin; RESET enable_mergejoin;
-- Now compare Execution Time across the four runs. The planner's default
-- should be the fastest; if it isn't, you've found a stats/cost problem.
```

### Example 3 — Production Grade: Diagnose and fix a slow reporting join

```sql
-- Context:
--   order_items: 40,000,000 rows (~4 GB heap)
--   orders:      8,000,000 rows
--   products:    120,000 rows
--   work_mem:    4 MB (default) on this reporting replica
-- Symptom: this monthly revenue rollup takes 95 seconds.
-- Expectation after fix: < 8 seconds.

EXPLAIN (ANALYZE, BUFFERS, SETTINGS)
SELECT
  p.category_id,
  COUNT(DISTINCT oi.order_id)          AS orders,
  SUM(oi.quantity * oi.unit_price)     AS revenue
FROM order_items oi
JOIN orders   o ON o.id = oi.order_id
JOIN products p ON p.id = oi.product_id
WHERE o.status = 'completed'
  AND o.created_at >= DATE_TRUNC('month', NOW() - INTERVAL '1 month')
  AND o.created_at <  DATE_TRUNC('month', NOW())
GROUP BY p.category_id;
```

Diagnostic plan (before fix) — abbreviated:

```
HashAggregate  (actual time=95000..95120 rows=38 loops=1)
  ->  Hash Join  (actual time=210..92000 rows=6100000 loops=1)
        Hash Cond: (oi.product_id = p.id)
        ->  Hash Join  (actual time=180..70000 rows=6100000 loops=1)
              Hash Cond: (oi.order_id = o.id)
              ->  Seq Scan on order_items oi   (actual rows=40000000 loops=1)  ← full 40M scan
              ->  Hash  (actual rows=520000 loops=1)
                    Buckets: 65536  Batches: 8  Memory Usage: 4096kB           ← SPILL
                    ->  Seq Scan on orders o
                          Filter: (status='completed' AND created_at >= ... )
                          Rows Removed by Filter: 7480000
        ->  Hash (products) ...
  Buffers: shared read=610000, temp read=180000 written=180000                 ← disk spill
Execution Time: 95000 ms
```

Two problems: (1) `order_items` is fully scanned (40M rows) because there is no way to pre-filter it before the join; (2) the `orders` hash spills to 8 batches (`temp written`). Fixes:

```sql
-- Fix 1: index the filter+join path on orders so only completed, recent orders are read
CREATE INDEX CONCURRENTLY idx_orders_status_created_id
  ON orders (status, created_at) INCLUDE (id);

-- Fix 2: index order_items on its join key so the driving side can be an
--         indexed Nested Loop from the (now small) filtered orders set
CREATE INDEX CONCURRENTLY idx_order_items_order_id
  ON order_items (order_id) INCLUDE (product_id, quantity, unit_price);

-- Fix 3: give the session enough memory to avoid the hash spill
SET work_mem = '256MB';
```

Plan after fix — abbreviated:

```
HashAggregate  (actual time=7200..7320 rows=38 loops=1)
  ->  Nested Loop  (actual time=0.3..6400 rows=6100000 loops=1)
        ->  Hash Join  (products small build, Batches:1)
              ->  Index Scan using idx_orders_status_created_id on orders o
                    Index Cond: (status='completed' AND created_at >= ... < ...)
                    (actual rows=520000 loops=1)                     ← only recent completed orders
        ->  Index Scan using idx_order_items_order_id on order_items oi
              Index Cond: (order_id = o.id)
              (actual rows=~12 loops=520000)                         ← indexed inner probe, no 40M scan
  Buffers: shared hit=... read=...   (no temp)                       ← no spill
Execution Time: 7250 ms
```

The 40M-row Seq Scan is gone (replaced by 520K indexed probes into `order_items`), the hash no longer spills, and the query drops from 95s to ~7s. The `INCLUDE` columns make the `order_items` scan index-only-ish (covering), avoiding heap fetches.

---

## 9. Wrong → Right Patterns

### Wrong 1: "Just add an index" without checking which side is inner

```sql
-- WRONG assumption: any index on the join column fixes a slow Nested Loop.
-- Query: large outer, small inner
SELECT c.name, o.total_amount
FROM orders o                              -- 2M rows, the OUTER
JOIN customers c ON c.id = o.customer_id   -- 100K rows, the INNER
WHERE o.created_at > NOW() - INTERVAL '1 day';

-- Developer adds:  CREATE INDEX ON orders(customer_id);
-- ...but orders is the OUTER side here. The index that a Nested Loop needs is on the
-- INNER join column (customers.id) — which is already the PK. The new index does nothing
-- for this plan. Worse, if there's no PK the fix is on customers, not orders.
```

```sql
-- RIGHT: identify the inner side from EXPLAIN first, then index THAT side's join key.
-- Here the inner is customers.id (already PK, already fast). The real fix for a slow
-- query is elsewhere — probably indexing orders.created_at so the outer shrinks:
CREATE INDEX ON orders(created_at);
-- Now the outer collapses to one day's orders and the indexed inner PK probe is cheap.
```

### Wrong 2: Raising work_mem globally to fix one spilling hash join

```sql
-- WRONG: a report spills to disk, so someone sets it cluster-wide.
ALTER SYSTEM SET work_mem = '512MB';   -- applies to EVERY sort/hash in EVERY connection
-- Result: a query with 4 hashes × 200 connections can request 4 × 512MB × 200 = 400 GB.
-- The box OOMs under concurrency. The report got faster; the database fell over.
```

```sql
-- RIGHT: raise work_mem only for the session/role that runs the heavy query.
SET work_mem = '512MB';   -- this connection only
-- run the reporting query
RESET work_mem;
-- Or dedicate a reporting role/pool:  ALTER ROLE reporting SET work_mem = '512MB';
```

### Wrong 3: Trusting the plan after a bulk load without ANALYZE

```sql
-- WRONG: bulk-load 10M rows, immediately run the join.
COPY events FROM '/data/events.csv';       -- no ANALYZE afterward
SELECT ... FROM events e JOIN ref r ON r.id = e.ref_id WHERE e.type = 'x';
-- The planner still thinks events has ~0 rows (pre-load statistics). It picks a
-- Nested Loop expecting a tiny outer. Actual outer is 10M rows → 10M inner probes
-- or a Seq-Scan Nested Loop. Query hangs for minutes.
```

```sql
-- RIGHT: analyze after any bulk change so the planner has real cardinalities.
COPY events FROM '/data/events.csv';
ANALYZE events;                            -- refresh pg_statistic
SELECT ... FROM events e JOIN ref r ON r.id = e.ref_id WHERE e.type = 'x';
-- Now the planner sees 10M rows, estimates correctly, and picks Hash Join.
```

### Wrong 4: Forcing Nested Loop with an ORM-style OFFSET/LIMIT and no early-stop index

```sql
-- WRONG: "keyset-free" pagination that the planner can't short-circuit.
SELECT o.*, c.name
FROM orders o JOIN customers c ON c.id = o.customer_id
ORDER BY o.created_at DESC
OFFSET 100000 LIMIT 20;
-- The Hash Join must build over ALL rows, then Sort ALL rows, then skip 100000.
-- LIMIT gives no benefit because the Sort blocks. Slow and gets slower per page.
```

```sql
-- RIGHT: keyset pagination + an index that provides both order and early stop.
CREATE INDEX ON orders(created_at DESC, id);
SELECT o.*, c.name
FROM orders o JOIN customers c ON c.id = o.customer_id
WHERE o.created_at < $1                    -- last seen created_at from prior page
ORDER BY o.created_at DESC
LIMIT 20;
-- Now an Index Scan feeds an indexed Nested Loop; LIMIT 20 stops the loop after 20 rows.
```

### Wrong 5: Non-equi-join expecting Hash Join speed

```sql
-- WRONG: developer assumes any join can hash and is surprised by a slow Nested Loop.
SELECT e.id, w.window_name
FROM events e
JOIN time_windows w
  ON e.occurred_at >= w.start_time
 AND e.occurred_at <  w.end_time;          -- RANGE condition, no equality
-- Hash Join is IMPOSSIBLE (hashes are point lookups). The planner uses a Nested Loop
-- with a Join Filter — O(events × windows). On millions of events this is very slow.
```

```sql
-- RIGHT: give the range join an index it can actually use, or restructure.
-- Option A: a range/GiST index so each event probes a small set of windows.
CREATE INDEX ON time_windows USING gist (tsrange(start_time, end_time));
SELECT e.id, w.window_name
FROM events e
JOIN time_windows w
  ON tsrange(w.start_time, w.end_time) @> e.occurred_at;   -- GiST-accelerated containment
-- Option B: if windows are contiguous, derive the key with date_trunc and equi-join,
--           which unlocks Hash Join.
```

---

## 10. Performance Profile

### 10.1 Algorithm cost summary

| Algorithm | Time complexity | Memory | Needs index? | Supports non-equi? | Output order |
|-----------|-----------------|--------|--------------|--------------------|--------------|
| Nested Loop (indexed inner) | O(outer × log inner) | O(1) | Yes, on inner key | Yes | outer's order |
| Nested Loop (no index) | O(outer × inner) | O(1) | — | Yes | outer's order |
| Hash Join | O(outer + inner) | O(build side) | No | Equality only | unordered |
| Merge Join (pre-sorted) | O(outer + inner) | O(1) | Sort-providing index ideal | Equality/range | sorted on key |
| Merge Join (must sort) | O(N log N + M log M) | O(work_mem)/spill | — | Equality/range | sorted on key |

### 10.2 Scaling behavior at 1M / 10M / 100M rows

**Nested Loop (indexed inner), small outer (say 1K rows):**
- Inner size barely matters — each probe is O(log inner). 1M / 10M / 100M inner ≈ 20 / 23 / 27 B-tree comparisons per probe. 1K probes → all three finish in single-digit milliseconds. This is why indexed Nested Loop is the OLTP workhorse: it is nearly insensitive to inner table growth.

**Nested Loop (no index), any size:**
- Product scaling. 1M×1M = 10^12 comparisons — hours to never. Never acceptable at scale; the presence of this plan is always a bug.

**Hash Join:**
- Linear in total rows, but the *build side* must fit in `work_mem` or it spills. Build side 1M × 32B ≈ 32MB (spills at 4MB default → ~8 batches). 10M ≈ 320MB. 100M ≈ 3.2GB (spills hard even at 256MB work_mem). Rule of thumb: keep the *build* (smaller) side under `work_mem`; the probe side can be arbitrarily large and stream fine.
- With spill: 1M build at 4MB work_mem ≈ 8–16 batches, ~3–8× slower than in-memory.

**Merge Join (pre-sorted via index):**
- Linear, O(1) memory, index-order scans. Scales gracefully to 100M+ where Hash would need many GB. The cost is the index maintenance and the random I/O of the index scans (mitigated by correlation / clustering).

**Merge Join (must sort both sides):**
- Two O(N log N) sorts. At 100M rows both sorts spill to disk (`external merge`). Usually loses to Hash Join unless the sorted output is independently required.

### 10.3 Index interactions

- **Indexed Nested Loop** needs an index on the **inner** join key. A **covering index** (`INCLUDE` columns) makes the inner probe index-only, skipping heap fetches — often the difference between fast and very fast.
- **Merge Join** needs an index that **provides the sort order** of the join key on both sides, letting it skip Sort nodes. A composite index whose leading column is the join key qualifies.
- **Hash Join** needs *no* index — it is the fallback when no helpful index exists. Adding an index will *not* speed a Hash Join's core work (though it may enable a different, cheaper plan entirely).
- **Table correlation / CLUSTER**: an index scan on a well-correlated (physically clustered) column reads sequential heap pages, making both Nested Loop probes and Merge Join index scans dramatically cheaper (`random_page_cost` effectively drops).

### 10.4 Optimization techniques specific to join algorithms

1. **Shrink the build/outer side first.** Push selective WHERE predicates so the smaller input into a Hash or the outer of a Nested Loop is as small as possible. Indexing the filter column is often the highest-leverage fix.
2. **Right-size `work_mem` per query.** Eliminate `Batches > 1` on hashes and `external merge` on sorts for known-heavy queries via `SET work_mem` in that session.
3. **Add covering indexes** (`INCLUDE`) on the inner join key so Nested Loop probes are index-only.
4. **Provide sort order for Merge Join** with composite indexes when you also need `ORDER BY` on the join key.
5. **Fix statistics** with `ANALYZE`, higher `default_statistics_target` on skewed columns, and `CREATE STATISTICS` (extended stats) for correlated columns that cause cardinality misestimates.
6. **Tune `random_page_cost`** to ~1.1 on SSD/NVMe so the planner stops under-using indexes (which biases it away from Nested Loop/Merge toward Seq-Scan Hash).
7. **Enable/verify parallelism** for big analytical joins (`max_parallel_workers_per_gather`), and confirm `Parallel Hash Join` appears.
8. **Consider Memoize** (PG14+) for Nested Loops with duplicate outer keys; verify high `Hits`.

### 10.5 The decision cheat-sheet

```
Small outer (after filtering) + indexed inner            → Nested Loop
Two large tables, equi-join, no useful sort order        → Hash Join
Two large tables, equi-join, both pre-sorted by index    → Merge Join
Need output sorted on join key anyway                    → Merge Join
Non-equi (range/inequality) join                         → Nested Loop (only option)
LIMIT small + index provides order                       → Nested Loop (early stop)
```

---

## 11. Node.js Integration

### 11.1 Capturing EXPLAIN output from the app

```javascript
import pg from 'pg';
const { Pool } = pg;
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

// Programmatically fetch the machine-readable plan to log/inspect slow joins.
async function explainJoin(customerId) {
  const sql = `
    EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)
    SELECT o.id, o.total_amount, c.name
    FROM orders o
    JOIN customers c ON c.id = o.customer_id
    WHERE o.customer_id = $1`;
  const { rows } = await pool.query(sql, [customerId]);
  const plan = rows[0]['QUERY PLAN'][0].Plan;
  return {
    nodeType: plan['Node Type'],          // "Nested Loop" | "Hash Join" | "Merge Join"
    actualRows: plan['Actual Rows'],
    planRows: plan['Plan Rows'],           // estimate vs actual = stats health
    execMs: rows[0]['QUERY PLAN'][0]['Execution Time'],
  };
}
```

### 11.2 Per-session work_mem for a heavy analytics endpoint

```javascript
// Raise work_mem for ONE query using a dedicated client so the setting is scoped,
// then release it. Never ALTER SYSTEM from app code.
async function monthlyRevenueByCategory() {
  const client = await pool.connect();
  try {
    await client.query(`SET LOCAL work_mem = '256MB'`);  // SET LOCAL = this txn only
    await client.query('BEGIN');
    const { rows } = await client.query(`
      SELECT p.category_id,
             COUNT(DISTINCT oi.order_id)      AS orders,
             SUM(oi.quantity * oi.unit_price) AS revenue
      FROM order_items oi
      JOIN orders   o ON o.id = oi.order_id
      JOIN products p ON p.id = oi.product_id
      WHERE o.status = 'completed'
        AND o.created_at >= date_trunc('month', now() - interval '1 month')
        AND o.created_at <  date_trunc('month', now())
      GROUP BY p.category_id`);
    await client.query('COMMIT');
    return rows;
  } catch (e) {
    await client.query('ROLLBACK');
    throw e;
  } finally {
    client.release();   // SET LOCAL is auto-reset at txn end; pool client is clean
  }
}
```

### 11.3 Forcing an algorithm as a targeted, scoped override

```javascript
// Occasionally the planner is wrong and you need a hotfix before you can fix stats.
// Scope the override to a single transaction with SET LOCAL so it never leaks.
async function reportWithForcedHashJoin() {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');
    await client.query('SET LOCAL enable_nestloop = off');   // discourage Nested Loop
    await client.query('SET LOCAL enable_mergejoin = off');  // discourage Merge Join
    const { rows } = await client.query(`
      SELECT o.id, c.name
      FROM orders o JOIN customers c ON c.id = o.customer_id
      WHERE o.status = 'completed'`);
    await client.query('COMMIT');
    return rows;
  } finally {
    client.release();
  }
}
// NOTE: treat this as a temporary bandage. The durable fix is ANALYZE / an index /
// extended statistics so the planner chooses correctly on its own.
```

### 11.4 Detecting a spilling hash join from app-side timing + plan

```javascript
// Wrap a query: if it's slow, auto-EXPLAIN it and flag spills for the logs.
async function runAndDiagnose(sql, params) {
  const t0 = Date.now();
  const res = await pool.query(sql, params);
  const ms = Date.now() - t0;
  if (ms > 1000) {  // slow — capture the plan
    const { rows } = await pool.query(
      `EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) ${sql}`, params
    );
    const planText = JSON.stringify(rows[0]['QUERY PLAN']);
    const spilled = /"Batches":\s*[2-9]|external merge|"Temp Written Blocks":\s*[1-9]/.test(planText);
    if (spilled) {
      console.warn(`[join-perf] slow join ${ms}ms with disk spill — raise work_mem`, { sql });
    }
  }
  return res.rows;
}
```

**Do ORMs handle this?** No ORM chooses or reasons about join *algorithms* — that is 100% the PostgreSQL planner's job. ORMs generate the SQL text; the planner decides Nested Loop vs Hash vs Merge. What ORMs *can* affect is the SQL shape (which drives estimates) and, in some, whether they emit a JOIN at all vs. N separate queries (Topic 18). For algorithm-level tuning (`work_mem`, `enable_*`, EXPLAIN inspection) you use raw connections/`$queryRaw` regardless of ORM.

---

## 12. ORM Comparison

The key framing: **no ORM controls the join algorithm** — the planner does. What differs is (a) how easily you can inspect the plan, (b) how easily you can set `work_mem`/`enable_*` per query, and (c) whether the ORM even emits a single JOIN (vs N+1, Topic 18).

### Prisma

**Can Prisma influence the join algorithm?** — Only indirectly, by generating SQL; it cannot set `work_mem` or `enable_*` inline. Use `$queryRaw`/`$executeRaw` for planner-level control.

```typescript
import { PrismaClient } from '@prisma/client';
const prisma = new PrismaClient();

// Inspect the plan Prisma's query would produce (run EXPLAIN on equivalent SQL):
const plan = await prisma.$queryRawUnsafe(
  `EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)
   SELECT o.id, c.name FROM orders o JOIN customers c ON c.id = o.customer_id
   WHERE o.status = 'completed'`
);

// Scoped work_mem override for a heavy raw analytics query:
await prisma.$transaction(async (tx) => {
  await tx.$executeRawUnsafe(`SET LOCAL work_mem = '256MB'`);
  return tx.$queryRawUnsafe(`SELECT ... big join ...`);
});
```

**Where it breaks:** Prisma's relational queries (`include`) may use its JOIN strategy or split into multiple queries (`relationLoadStrategy: "join" | "query"`), which changes the plan entirely. You cannot set session GUCs on a normal Prisma call. **Verdict:** fine for CRUD; drop to `$queryRaw` + `$transaction` with `SET LOCAL` for algorithm-level tuning and EXPLAIN.

### Drizzle ORM

**Can Drizzle influence the algorithm?** — Same answer: it emits SQL; use `sql` and `db.execute` for EXPLAIN and GUCs.

```typescript
import { db } from './db';
import { sql } from 'drizzle-orm';

// EXPLAIN a Drizzle query by wrapping raw SQL
const plan = await db.execute(sql`
  EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)
  SELECT o.id, c.name FROM orders o JOIN customers c ON c.id = o.customer_id`);

// Scoped tuning within a transaction
await db.transaction(async (tx) => {
  await tx.execute(sql`SET LOCAL work_mem = '256MB'`);
  await tx.execute(sql`SET LOCAL enable_nestloop = off`);
  return tx.execute(sql`SELECT o.id, c.name FROM orders o
                        JOIN customers c ON c.id = o.customer_id`);
});
```

**Where it breaks:** nothing specific — Drizzle is thin, so raw SQL and `SET LOCAL` compose cleanly. **Verdict:** best-in-class for this because its SQL transparency makes EXPLAIN and GUC scoping trivial.

### Sequelize

**Can Sequelize influence the algorithm?** — Via `sequelize.query()` for EXPLAIN/GUCs; the model layer cannot.

```javascript
const { QueryTypes } = require('sequelize');

// EXPLAIN
const plan = await sequelize.query(
  `EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)
   SELECT o.id, c.name FROM orders o JOIN customers c ON c.id = o.customer_id`,
  { type: QueryTypes.SELECT }
);

// Scoped work_mem in a managed transaction
await sequelize.transaction(async (t) => {
  await sequelize.query(`SET LOCAL work_mem = '256MB'`, { transaction: t });
  return sequelize.query(`SELECT o.id, c.name FROM orders o
                          JOIN customers c ON c.id = o.customer_id`,
    { type: QueryTypes.SELECT, transaction: t });
});
```

**Where it breaks:** Sequelize's `include` can also generate subquery-wrapped SQL (especially with `limit` + `hasMany`), which changes the plan and can force a worse algorithm; inspect the generated SQL with `logging: console.log`. **Verdict:** usable via raw queries; watch the auto-generated subquery wrapping.

### TypeORM

**Can TypeORM influence the algorithm?** — Via `dataSource.query()`; QueryBuilder emits SQL only.

```typescript
// EXPLAIN a QueryBuilder query
const qb = dataSource.getRepository(Order).createQueryBuilder('o')
  .innerJoin('o.customer', 'c').where("o.status = :s", { s: 'completed' });
const [sqlText, params] = qb.getQueryAndParameters();
const plan = await dataSource.query(
  `EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) ${sqlText}`, params);

// Scoped tuning
await dataSource.transaction(async (mgr) => {
  await mgr.query(`SET LOCAL work_mem = '256MB'`);
  return mgr.query(`SELECT o.id, c.name FROM orders o
                    JOIN customers c ON c.id = o.customer_id`);
});
```

**Where it breaks:** `getQueryAndParameters()` is the clean way to feed generated SQL into EXPLAIN; QueryBuilder itself has no GUC support. **Verdict:** solid — QueryBuilder's `getQueryAndParameters()` makes plan inspection straightforward.

### Knex.js

**Can Knex influence the algorithm?** — Most transparent; `.toSQL()` and `knex.raw()` handle EXPLAIN and GUCs.

```javascript
// EXPLAIN a Knex query
const q = knex('orders as o')
  .join('customers as c', 'c.id', 'o.customer_id')
  .where('o.status', 'completed')
  .select('o.id', 'c.name');
const { sql, bindings } = q.toSQL().toNative();
const plan = await knex.raw(`EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) ${sql}`, bindings);

// Scoped tuning
await knex.transaction(async (trx) => {
  await trx.raw(`SET LOCAL work_mem = '256MB'`);
  return trx('orders as o')
    .join('customers as c', 'c.id', 'o.customer_id')
    .where('o.status', 'completed').select('o.id', 'c.name');
});
```

**Where it breaks:** nothing notable; `.toSQL().toNative()` gives you exactly what to EXPLAIN. **Verdict:** excellent transparency; the easiest to reason about at the plan level.

### ORM Summary Table

| ORM | Inspect plan (EXPLAIN) | Scoped `work_mem`/`enable_*` | Risk of unexpected plan shape | Verdict |
|-----|------------------------|------------------------------|-------------------------------|---------|
| Prisma | `$queryRaw EXPLAIN` | `$transaction` + `SET LOCAL` | `relationLoadStrategy`, query-splitting | CRUD good; raw for tuning |
| Drizzle | `db.execute(sql\`EXPLAIN...\`)` | `transaction` + `SET LOCAL` | Low (thin layer) | Best transparency |
| Sequelize | `sequelize.query` | `transaction` + raw | `include` subquery wrapping | OK via raw |
| TypeORM | `getQueryAndParameters()` + `query` | `transaction` + `query` | QueryBuilder subqueries | Solid |
| Knex | `.toSQL().toNative()` + `raw` | `transaction` + `raw` | Low | Easiest at plan level |

---

## 13. Practice Exercises

### Exercise 1 — Easy

Given:
- `orders(id, customer_id, status, total_amount, created_at)` — 3M rows
- `customers(id, name, email)` — 200K rows, `id` is the PK

Run `EXPLAIN` on a join of `orders` to `customers` on `customer_id = id`, once with `WHERE o.id = 12345` and once with no filter. For each, state which join algorithm the planner will choose and why. Then explain what the `loops=` value on the inner node would be in each case.

```sql
-- Write your query here
```

### Exercise 2 — Medium (combines Topic 11 fan-out + this topic)

Given `orders` (3M) and `order_items` (15M, indexed on `order_id`), write a query that returns, per customer, the count of distinct orders and total revenue for completed orders in the last 30 days. Then:
1. Predict which join algorithm connects `orders` and `order_items` when the 30-day filter reduces `orders` to ~50K rows, and why.
2. Predict what changes if you remove the date filter (all 3M orders).
3. Explain how the fan-out from `order_items` interacts with your choice of `COUNT(DISTINCT ...)` (recall Topic 11).

```sql
-- Write your query here
```

### Exercise 3 — Hard (production simulation; the naive answer is wrong)

A dashboard query joins `sessions` (80M rows) to `users` (5M rows) on `user_id`, filtered to `sessions.started_at > now() - interval '1 hour'` (~120K rows), and it runs in 45 seconds. EXPLAIN shows a `Hash Join` that builds a hash over all 5M `users` and spills to 32 batches. The naive fix is "raise work_mem." Explain why raising `work_mem` alone is the wrong primary fix here, identify the *actual* problem (which side is being hashed and why), and write the index + query changes that would make this a sub-second indexed Nested Loop. State the expected plan after your fix.

```sql
-- Write your query here
```

### Exercise 4 — Interview Simulation

You are handed this plan for a two-table join and asked to diagnose it live:

```
Nested Loop  (actual time=0.5..88000 rows=204 loops=1)
  Join Filter: (p.id = oi.product_id)
  Rows Removed by Join Filter: 5999996
  ->  Seq Scan on order_items oi   (actual rows=6000000 loops=1)
  ->  Materialize  (actual rows=... loops=6000000)
        ->  Seq Scan on products p  (actual rows=? loops=1)
```

Answer aloud: (1) Which algorithm is this and what is pathological about it? (2) What does `Materialize` tell you? (3) What single change most likely fixes it, and what will the new plan's join node be? (4) Why did the planner pick this bad plan in the first place — name the two most likely root causes.

```sql
-- Write your diagnosis and corrected query here
```

---

## 14. Interview Questions

### Junior Level

**Q: What are the three join algorithms PostgreSQL uses, in one sentence each?**

A: **Nested Loop** — for each outer row, look up matches in the inner table (fast when the inner side is indexed and the outer is small). **Hash Join** — build an in-memory hash table on the smaller table, then stream the larger table through it, matching by hash (good for large equi-joins). **Merge Join** — sort both inputs on the join key (or read them pre-sorted from indexes) and walk them in lockstep (good when inputs are already sorted or the output needs to be sorted).

**Follow-up:** *Which of the three can handle a `>` (non-equality) join condition?* → Only Nested Loop. Hash Join requires equality (hashes are point lookups); Merge Join requires an ordering/equality operator. A range or inequality join must be a Nested Loop.

---

**Q: You see `Nested Loop` with a `Seq Scan` on the inner side and `loops=50000`. What does that mean and is it good?**

A: It means the inner table is being fully sequentially scanned once for every one of the 50,000 outer rows — 50,000 full scans. That is almost always bad: it is O(outer × inner) work. The fix is to add an index on the inner table's join column so each iteration becomes a cheap index probe instead of a full scan, or to make the planner switch to a Hash Join.

**Follow-up:** *If you add the index and the planner still uses a Seq-Scan Nested Loop, what would you check?* → Whether statistics are stale (the planner may misestimate the outer as tiny), whether the index is usable (type mismatch on the join key prevents its use), and whether `enable_seqscan`/cost settings are skewing the choice.

---

### Mid Level

**Q: A Hash Join shows `Batches: 16` in EXPLAIN. What is happening and what are your options?**

A: `Batches > 1` means the build-side hash table did not fit in `work_mem`, so PostgreSQL partitioned both inputs into 16 batches, kept one in memory, and spilled the rest to temporary files on disk — a hybrid hash join. This typically costs several times the in-memory time due to the extra write+read. Options: raise `work_mem` for that session so the build side fits (targets `Batches: 1`); reduce the build side's row count with a more selective filter; reduce its width by selecting fewer/narrower columns; or accept the spill if the query is rare. Crucially, ensure the *smaller* relation is the build side — if the estimate is wrong and the large table is being hashed, fixing statistics is the real fix.

**Follow-up:** *You raise `work_mem` to 1GB and it's fast — ship it globally?* → No. `work_mem` is per-operation, per-connection. A query with several hashes/sorts times hundreds of connections can exhaust RAM and OOM the server. Scope it per session/role, not cluster-wide.

---

### Principal Level

**Q: A query joining a 50M-row table to a 20K-row reference table ran in 40ms yesterday and 30s today, with no code or schema change. Walk me through your diagnosis.**

A: The plan almost certainly flipped algorithms. Steps: (1) `EXPLAIN ANALYZE` now and compare `rows=` (estimate) to `actual rows=` at each node — a large gap points to statistics drift, which is the usual trigger. (2) Yesterday's fast plan was likely an indexed Nested Loop (small filtered outer, indexed inner PK) or a non-spilling Hash Join. Today it may have flipped to a Seq-Scan Nested Loop (inner index no longer chosen because the planner thinks the outer is tiny but it's now large) or a spilling Hash Join (build side grew past `work_mem`). (3) Check what changed underneath: did the table cross a size threshold; did autovacuum/autoanalyze fall behind (`pg_stat_user_tables.last_autoanalyze`), leaving stale stats; was an index invalidated or dropped; did data skew shift so a formerly-selective predicate now matches many rows. (4) Immediate mitigation: `ANALYZE` the tables to refresh statistics; if that fixes the estimate, the good plan returns. (5) If stats are correct but the plan is still wrong, inspect cost settings — e.g., `random_page_cost` too high for SSD biases against index/Nested-Loop plans. (6) As a temporary bandage, force the known-good algorithm with `SET LOCAL enable_hashjoin/enable_nestloop` in the query's transaction while I implement the durable fix (index, extended statistics, or a higher statistics target on the skewed column).

**Follow-up:** *`ANALYZE` didn't help and the estimate is still off by 1000×. What next?* → Likely correlated columns the planner assumes are independent (multiplying selectivities to an absurdly low estimate). Create extended statistics: `CREATE STATISTICS ... (dependencies, ndistinct) ON col_a, col_b FROM tbl; ANALYZE tbl;`. Also consider raising `default_statistics_target` on the skewed column for finer histograms.

---

**Q: When would you deliberately prefer a Merge Join over a Hash Join for two large tables?**

A: Three situations. (1) **Both inputs are already sorted on the join key** via usable indexes — the Merge Join reads them in order with no Sort node and O(1) memory, whereas a Hash Join over tens of millions of rows would need a huge hash and likely spill to many batches. (2) **`work_mem` is constrained** and the build side is too large to hash without heavy spilling — Merge Join's constant memory footprint wins because it doesn't materialize a whole relation. (3) **The query needs the result ordered on the join key anyway** (an `ORDER BY` on that key downstream) — Merge Join produces sorted output as a byproduct, so you pay for the ordering once instead of a Hash Join plus a separate final Sort. The trade-off: if neither side is pre-sorted, Merge Join must add two O(N log N) sorts that themselves spill, and then Hash Join usually wins. So the decision hinges on whether the sort order is *free* (from indexes) or *needed* (by the query).

**Follow-up:** *How do you confirm the Merge Join is getting free ordering rather than sorting?* → In EXPLAIN, the Merge Join's children should be `Index Scan` nodes with **no** `Sort` node between them and the join. If you see `Sort ... Sort Method: external merge Disk:` you're paying for the ordering, and a Hash Join is probably cheaper — verify with `SET enable_mergejoin = off`.

---

## 15. Mental Model Checkpoint

1. A query joins a 10-row filtered outer to a 100M-row inner table that has a B-tree index on the join key. Which algorithm should win, roughly how many buffer reads will it do, and why is the 100M inner size almost irrelevant?

2. You force `enable_hashjoin = off` and the plan still shows a Hash Join. How is that possible?

3. A Hash Join's `Hash` node reports `Memory Usage: 3800kB` and `Batches: 1` with the default 4MB `work_mem`. You add one more column to the SELECT and now it shows `Batches: 4`. Explain the causal chain.

4. Two tables of 20M rows each are joined on an indexed key, and the plan is a Merge Join with no Sort nodes. Someone drops one of the two indexes. Predict the new plan and its relative cost.

5. Why does a small `LIMIT` make Nested Loop attractive but give almost no benefit to the *build* phase of a Hash Join?

6. EXPLAIN ANALYZE shows a join node with `rows=3` (estimate) but `actual rows=2900000`. You have not changed the query. What is the single most likely root cause, and what one command do you run first?

7. A `Nested Loop` wraps its inner Index Scan in a `Memoize` node showing `Hits: 990000 Misses: 10000`. What is Memoize doing, why is this ratio good, and under what outer-key distribution would Memoize be useless?

---

## 16. Quick Reference Card

```sql
-- ── IDENTIFY THE ALGORITHM (top join node in EXPLAIN) ─────────────────────
Nested Loop   -- per-outer-row inner probe; only algo for non-equi joins
Hash Join     -- build hash on smaller side, probe with larger; equality only
Merge Join    -- sort both (or read pre-sorted), walk in lockstep

-- ── KEY EXPLAIN SIGNALS ───────────────────────────────────────────────────
loops=N on inner              -- Nested Loop iteration count (inner time is per-loop)
Batches: >1  + temp written   -- Hash Join spilled to disk → raise work_mem
Sort Method: external merge    -- a Sort spilled to disk → raise work_mem / add index
rows=est vs actual rows=truth  -- big gap = stale stats → ANALYZE
Rows Removed by Join Filter    -- wasted comparisons → missing index / non-equi
Memoize Hits/Misses            -- inner-probe cache effectiveness (PG14+)

-- ── WHICH ALGORITHM WHEN ──────────────────────────────────────────────────
small outer + indexed inner ................ Nested Loop
big equi-join, no sort order ............... Hash Join
big equi-join, both pre-sorted by index .... Merge Join
need output ordered on join key ............ Merge Join
non-equi (range / < > BETWEEN) ............. Nested Loop (only option)
small LIMIT + index order .................. Nested Loop (early stop)

-- ── FORCING / TUNING (diagnose first, force sparingly) ────────────────────
SET LOCAL enable_nestloop  = off;   -- discourage (penalty, not hard off)
SET LOCAL enable_hashjoin  = off;
SET LOCAL enable_mergejoin = off;
SET LOCAL work_mem = '256MB';       -- per-session; NEVER huge cluster-wide
SET LOCAL random_page_cost = 1.1;   -- SSD/NVMe: stop under-using indexes
ANALYZE tbl;                         -- refresh stats (do after bulk loads!)
CREATE STATISTICS s (dependencies) ON a, b FROM tbl;  -- correlated-column estimates

-- ── INDEXING RULES ────────────────────────────────────────────────────────
-- Nested Loop needs index on the INNER join key (covering INCLUDE = index-only)
CREATE INDEX ON inner_tbl (join_key) INCLUDE (cols_selected);
-- Merge Join wants an index that PROVIDES SORT ORDER on the join key (both sides)
-- Hash Join needs NO index (adding one may enable a different, better plan)

-- ── ONE-LINERS FOR INTERVIEWS ─────────────────────────────────────────────
-- "The algorithm is the planner's choice; my job is to make the right one cheapest."
-- "Seq Scan on the inner side of a Nested Loop with big loops = missing index."
-- "Batches > 1 means the hash spilled; work_mem too small or wrong build side."
-- "Merge Join wins when the sort is free (indexes) or needed (ORDER BY) anyway."
-- "A 1000x estimate/actual gap is a stats bug, not a query bug — ANALYZE first."
```

---

## Connected Topics

- **Topic 11 — INNER JOIN in Depth**: introduced the three algorithms conceptually; this topic is the full mechanics, spill behavior, and forcing.
- **Topic 16 — Multiple JOINs**: join *order* (which pair first) vs. join *algorithm* (this topic) — the two decisions interact through intermediate result sizes.
- **Topic 18 — The N+1 Query Problem**: what happens when an ORM avoids the JOIN entirely and issues one query per row — the anti-pattern this topic's single-JOIN plans replace.
- **Topic 19 / 27 — JOIN Alternatives / EXISTS**: semi-joins and anti-joins use these same three algorithms but stop at first match, avoiding fan-out.
- **Topic 20 — GROUP BY Fundamentals**: aggregation consumes join output; a fan-out join feeding an aggregate is why plans process far more rows than the result implies.
- **Internals this builds on**: B-tree indexes (inner probes, sort order), the buffer pool (`shared hit/read`), `work_mem` (hash build + external sort), `pg_statistic`/`ANALYZE` (cardinality → algorithm), the cost model (`random_page_cost`, `seq_page_cost`), and parallel workers (`Parallel Hash Join`, `Gather`).
```
```
```
```
```
```
```
