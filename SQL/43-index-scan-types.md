# Topic 43 — Index Scan Types
### SQL Mastery Curriculum — Phase 7: Indexes and Query Optimisation

---

## 1. ELI5 — The Analogy That Makes It Click

Imagine a giant textbook — 900 pages, no chapters, just one continuous flow of paragraphs. Someone asks you five different kinds of questions, and for each one you have to *decide how to look things up*.

**Question 1: "Read me every sentence that mentions the word 'photosynthesis'."**
You know from experience that "photosynthesis" appears on nearly every page. There's no point flipping to the index and bouncing back and forth 800 times — you'd wear out your thumbs. So you just start at page 1 and read the whole book cover to cover, calling out the matches as you find them. That's a **Sequential Scan**: when you're going to touch most of the book anyway, reading it straight through in order is fastest, because turning pages in order is smooth and you never lose your place.

**Question 2: "Find the one paragraph about the philosopher 'Anaximander'."**
That word appears exactly once. Flipping through 900 pages to find one word would be insane. So you go to the alphabetical index at the back, find "Anaximander → page 412," flip straight to 412, read that paragraph, done. That's an **Index Scan**: the index tells you *exactly where* the answer lives, and you jump directly to that page. One lookup, one page turn.

**Question 3: "How many pages mention 'Anaximander'? Just the page numbers, don't read the paragraphs."**
Here's the trick — the index *already contains the page numbers*. You don't even need to open the book to page 412. You read the answer straight off the index entry itself and never touch the actual pages. That's an **Index Only Scan**: everything you need is already in the index, so you skip the book entirely.

**Question 4: "Read me every paragraph about any Greek philosopher — there are about 40 of them scattered around."**
Forty isn't "almost the whole book" (so a sequential read is wasteful), but it's also not "just one" (so jumping around one-at-a-time means 40 chaotic back-and-forth trips, and you might visit page 88 three separate times). Instead you do something clever: first you go through the index and write down *all 40 page numbers on a scratch pad*. Then you **sort that list** — 12, 33, 88, 91, 140, … — and now you walk through the book *in page order, once*, stopping at each page on your list. You never backtrack, and you visit each page exactly once. That two-step "collect the page numbers first, then read them in order" method is a **Bitmap Index Scan** (build the list of pages) followed by a **Bitmap Heap Scan** (visit them in order).

**Question 5: The librarian's judgement.**
The crucial part isn't any single method — it's *how you decided which one to use*. You estimated **how many matches there would be** before choosing. Almost-all → read straight through. Exactly one → jump. A scattered few dozen → collect-sort-visit. That estimate is called **selectivity**, and the whole art of this topic is understanding how PostgreSQL's planner makes exactly this judgement call, and what to do when it judges wrong.

That's the entire topic. Five ways to answer a lookup, and one decision — *how many rows will match?* — that drives which one gets picked.

---

## 2. Connection to SQL Internals

Every scan type is a physical strategy for solving the same abstract problem: *produce the set of rows that satisfy a predicate.* To understand why PostgreSQL has five different strategies, you have to understand the internal structures they operate on.

**The heap.** PostgreSQL stores table rows in the **heap** — an unordered collection of 8KB **pages** (also called blocks). A row's physical address is its **TID** (tuple identifier): a `(block_number, item_offset)` pair, e.g. `(42, 7)` means "block 42, the 7th item slot." The heap has no inherent order — row 1,000,000 might sit physically before row 5. Nothing about the heap helps you find a specific value; it's just a pile.

**The B-tree index.** A B-tree (Topic 42) is a separate, sorted structure. Its leaf pages hold `(indexed_value → TID)` entries in sorted order by value. Given a value, a B-tree lookup is O(log N) tree-descent to the leaf, then a sequential walk along leaves for a range. Critically, **the index stores the value and the TID — it does not store the rest of the row.** To get the other columns, you must follow the TID back into the heap. That follow-the-pointer step is the **heap fetch**, and it is the single most important cost in this entire topic.

**Random vs sequential I/O.** Reading heap pages *in physical order* (block 0, 1, 2, 3…) is **sequential I/O** — the disk head streams, the OS prefetches the next blocks, and throughput is high. Jumping to block 42, then block 7, then block 900 is **random I/O** — each jump is an independent seek. On spinning disks random I/O was ~100× slower than sequential; on SSDs the gap is smaller (often 5–20×) but still real, and it still matters because random single-page reads defeat OS readahead and pollute the buffer cache. The planner models this with two cost constants: `seq_page_cost` (default `1.0`) and `random_page_cost` (default `4.0`). That 4:1 ratio is the number that decides most of what this topic is about.

**The buffer pool.** Pages live in `shared_buffers` once read. A "hit" is free-ish; a "miss" reads from the OS cache or disk. `EXPLAIN (BUFFERS)` reports `shared hit` vs `shared read`, and this is how you *empirically* tell whether a scan touched many pages or few.

**The visibility map.** Because of MVCC (Topic on MVCC), a row's visibility depends on the transaction snapshot — an index entry alone can't tell you whether the row it points to is visible *to you*. Normally that forces a heap fetch just to check `xmin`/`xmax`. But PostgreSQL maintains a **visibility map (VM)**: one bit per heap page saying "every tuple on this page is visible to everyone." When that bit is set (kept current by `VACUUM`), an Index Only Scan can skip the heap fetch entirely. This is *why* Index Only Scans exist and *why* they depend on `VACUUM`.

**The bitmap.** For the scattered-few-dozen case, PostgreSQL builds an in-memory **bitmap** of matching heap pages, then reads those pages in physical order. If the bitmap gets too big for `work_mem`, it degrades to page-granularity ("lossy") and rechecks the predicate per row. This is the machinery behind Bitmap Index/Heap Scan.

Five scan types, three storage structures (heap, B-tree, visibility map), one in-memory bitmap, and one cost ratio. That's the whole machine.

---

## 3. Logical Execution Order Context

```
FROM table                    ← scan strategy is chosen HERE
  ↓                             (Seq / Index / Index Only / Bitmap)
WHERE predicate               ← the predicate the scan uses to filter
  ↓
GROUP BY / HAVING
  ↓
SELECT columns                ← which columns are needed decides Index Only eligibility
  ↓
ORDER BY                      ← an ordered index scan can SATISFY this for free
  ↓
LIMIT                         ← LIMIT changes the cost math dramatically
```

Scan-type selection happens at the very first physical step — reading rows out of a table in the `FROM` clause. But it is not decided in isolation; the planner looks *forward* through the whole query to make the choice:

- **The `WHERE` predicate** determines *whether an index is usable at all* and *how many rows match*. `WHERE id = 5` (one row) and `WHERE status = 'active'` (half the table) get radically different scans even against the same table.

- **The `SELECT` list matters for Index Only Scans.** An Index Only Scan is only possible if *every column the query needs* — from SELECT, WHERE, ORDER BY, everything — is present in the index. If you `SELECT *`, the index almost never covers it, and you fall back to a plain Index Scan with heap fetches. This is why Index Only Scan eligibility is a whole-query property, not a per-clause one.

- **`ORDER BY` can be satisfied by the scan for free.** A B-tree is sorted. If your `ORDER BY` matches the index order, an Index Scan emits rows *already sorted* and the planner drops the `Sort` node entirely. A Bitmap Heap Scan, by contrast, emits rows in *physical heap order*, not index order — so it **cannot** satisfy `ORDER BY` for free. This is a key tradeoff: bitmap scans are great for scattered matches but destroy index ordering.

- **`LIMIT` reshapes the cost model.** A plain Index Scan can *stream* rows and stop early once `LIMIT` is satisfied. A Bitmap scan must build the entire bitmap before returning a single row (it's a blocking, two-phase operation). So `LIMIT 10` often flips the planner from Bitmap to a plain ordered Index Scan, because the Index Scan can bail out after 10 rows while the bitmap would pay to enumerate all matches first.

The mental model: the scan is chosen first physically, but the planner reasons about it *last logically* — it needs to know what WHERE filters, what SELECT needs, what ORDER BY wants, and what LIMIT caps before it can price each candidate scan.

---

## 4. What Is an Index Scan Type?

An **index scan type** (more precisely, a *scan method* or *access path*) is the physical algorithm PostgreSQL uses to read the rows of a single table that satisfy a query. PostgreSQL's planner considers several candidate access paths for each table, assigns each an estimated cost, and picks the cheapest. The five relevant to this topic are Sequential Scan, Index Scan, Index Only Scan, Bitmap Index Scan, and Bitmap Heap Scan (the last two always appear as a pair).

```sql
-- You never WRITE the scan type. You write a query;
-- the planner CHOOSES the scan. You influence it via
-- indexes, statistics, and cost settings.

SELECT columns              -- ┐ which columns are needed → Index Only eligibility
FROM   table                -- ┤ the relation to scan
WHERE  predicate            -- ┘ the filter → drives selectivity → drives scan choice

-- To SEE the chosen scan:
EXPLAIN (ANALYZE, BUFFERS)
SELECT ... FROM ... WHERE ...;
--       │        │
--       │        └── BUFFERS: shows shared hit/read — how many pages touched
--       └── ANALYZE: actually RUNS it, shows actual rows + timing
```

### The Five Scan Types Annotated

```
Seq Scan on orders                       -- ① SEQUENTIAL SCAN
  Filter: (status = 'shipped')           --    reads EVERY heap page in physical order,
  Rows Removed by Filter: 940000         --    applies Filter to each row. No index used.
                                         --    Chosen when: matches are a large fraction
                                         --    of the table, or no usable index exists.

Index Scan using orders_pkey on orders   -- ② INDEX SCAN
  Index Cond: (id = 42)                  --    descends B-tree, gets TID(s), fetches the
                                         --    heap page(s) for each. Random I/O per row.
                                         --    Chosen when: few matching rows (high
                                         --    selectivity), especially with ORDER BY/LIMIT.

Index Only Scan using ... on orders      -- ③ INDEX ONLY SCAN
  Index Cond: (customer_id = 42)         --    like Index Scan but reads value straight
  Heap Fetches: 0                        --    from the index — NO heap fetch, IF the
                                         --    visibility map bit is set AND the index
                                         --    covers every needed column. Heap Fetches:0
                                         --    is the marker of success.

Bitmap Heap Scan on orders               -- ④ BITMAP HEAP SCAN (the second phase)
  Recheck Cond: (status = 'shipped')     --    visits heap pages IN PHYSICAL ORDER using
  ->  Bitmap Index Scan on orders_status --    ⑤ BITMAP INDEX SCAN (the first phase)
        Index Cond: (status = 'shipped') --       scans index, sets a bit for each
                                         --       matching heap page, builds a bitmap.
                                         --    Chosen when: a moderate, scattered set of
                                         --    rows — too many for a plain Index Scan's
                                         --    random I/O, too few for a full Seq Scan.
```

### The Precise Definitions

- **Sequential Scan**: read all heap pages in physical order, apply the filter to every tuple, emit matches. No index involved. Cost ≈ (pages × `seq_page_cost`) + (rows × `cpu_tuple_cost`).

- **Index Scan**: descend the B-tree to find matching entries; for each entry, follow the TID into the heap to fetch the full row. Emits rows in *index order*. Each heap fetch is potentially a random page read.

- **Index Only Scan**: same B-tree descent, but if the index contains all needed columns and the row's heap page is marked all-visible, skip the heap fetch and read the values from the index leaf. Emits in index order. Falls back to a heap fetch per row where the VM bit is unset.

- **Bitmap Index Scan**: scan one or more indexes, producing an in-memory bitmap of which heap pages (and, if not lossy, which tuples) contain matches. Does *not* read the heap. Its output feeds a Bitmap Heap Scan.

- **Bitmap Heap Scan**: consume the bitmap, read the flagged heap pages *in physical (ascending block) order*, and — because the bitmap may be page-granular — re-apply (`Recheck Cond`) the predicate to each tuple. Emits rows in *heap order*, not index order.

---

## 5. Why Index Scan Mastery Matters in Production

1. **The difference between 0.2ms and 20,000ms is a scan-type choice.** A query that does an Index Scan on a 100M-row table to return 5 rows finishes in under a millisecond. The *same query* forced into a Seq Scan reads all 100M rows — tens of seconds, saturating disk I/O and evicting everyone else's hot pages from the buffer cache. Knowing which scan you got, and why, is the difference between a healthy database and a 3 a.m. page.

2. **The planner flips scan types silently as data grows.** A query that used an Index Scan when the table had 10,000 rows may switch to a Seq Scan at 10M rows if statistics say the predicate now matches 30% of rows — or worse, keep an Index Scan when it *should* Seq Scan because statistics are stale. Latency regressions with no code change are almost always a scan-type flip. You cannot debug them without reading `EXPLAIN`.

3. **Index Only Scans depend on `VACUUM`, and that dependency is invisible until it bites.** An Index Only Scan on a heavily-updated table can show `Heap Fetches: 900000` — meaning it degraded to a regular Index Scan because the visibility map is stale. The query "has the right index" and still runs slowly. Understanding the VM/VACUUM link is the only way to diagnose this.

4. **`random_page_cost` is wrong by default on SSDs, and it silently biases every plan.** The default `4.0` assumes spinning disks. On SSD/NVMe, random reads are much cheaper relative to sequential, so the true ratio is closer to `1.1–1.5`. Leaving it at `4.0` makes the planner *under-use* index scans across your entire workload — a fleet-wide performance tax that no single query rewrite fixes.

5. **Bitmap scans are the planner's answer to "OR" and multi-index queries — and knowing when they help vs hurt is a senior skill.** A `WHERE a = 1 OR b = 2` can combine two indexes via `BitmapOr`. But bitmap scans destroy index ordering (breaking free `ORDER BY`) and have fixed setup cost. Recognising when a bitmap is the right tool — and when to force a plain Index Scan for a `LIMIT`ed, ordered query — is exactly the kind of judgement that separates a principal engineer from someone who "just adds an index and hopes."

6. **Selectivity reasoning is the foundation of all indexing decisions.** Every "should I add an index?" question reduces to "how selective is this predicate, and which scan would the planner then choose?" If you can't predict the scan, you can't predict whether the index will be used — and unused indexes are pure write-amplification overhead (Topic 42).

---

## 6. Deep Technical Content

### 6.1 Sequential Scan — The Baseline

A Seq Scan reads the table's heap pages from block 0 to the last block, in order. For each tuple, it checks visibility (MVCC) and applies the `Filter` predicate. It is the *only* scan that works with no index, and it is often the *fastest* scan when the predicate is unselective.

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE status = 'shipped';
--                          └── matches ~40% of a 1M-row table
```

```
Seq Scan on orders  (cost=0.00..20834.00 rows=400000 width=68)
                    (actual time=0.02..142.7 rows=398211 loops=1)
  Filter: (status = 'shipped')
  Rows Removed by Filter: 601789
  Buffers: shared hit=8334
Planning Time: 0.10 ms
Execution Time: 168.3 ms
```

Why a Seq Scan wins here: 40% of rows match. An Index Scan would do ~400,000 random heap fetches; the Seq Scan does 8,334 *sequential* page reads. Even though the Seq Scan reads *more* pages (all of them), it reads them sequentially and with OS readahead, so it's dramatically cheaper than 400K random seeks. The `Rows Removed by Filter` line confirms it examined all 1M rows.

**Key properties of Seq Scan:**
- Cost is roughly constant regardless of how many rows match — it always reads the whole heap.
- It benefits from `synchronize_seqscans` (multiple concurrent Seq Scans can piggyback on each other's page reads) and from parallel workers (`Parallel Seq Scan`).
- `Rows Removed by Filter` tells you how much work was wasted — a huge number here with a tiny `rows` output is the classic "you need an index" signal.

### 6.2 Index Scan — Direct, Ordered, Random

An Index Scan descends the B-tree using the `Index Cond`, walks the matching leaf entries, and for *each* entry follows the TID into the heap to fetch the full tuple. The output is in **index order**.

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE id = 42;
```

```
Index Scan using orders_pkey on orders  (cost=0.42..8.44 rows=1 width=68)
                                        (actual time=0.03..0.03 rows=1 loops=1)
  Index Cond: (id = 42)
  Buffers: shared hit=4
Planning Time: 0.08 ms
Execution Time: 0.05 ms
```

Four buffer hits: ~3 to descend the B-tree (root → internal → leaf) plus 1 heap page. This is the ideal high-selectivity case.

**The `Index Cond` vs `Filter` distinction** — this is critical and constantly misread:

```sql
EXPLAIN ANALYZE
SELECT * FROM orders
WHERE customer_id = 42        -- indexed
  AND total_amount > 100;     -- NOT part of this index
```

```
Index Scan using orders_customer_id_idx on orders  (cost=0.42..48.6 rows=3 width=68)
  Index Cond: (customer_id = 42)          ← used to NAVIGATE the B-tree (efficient)
  Filter: (total_amount > 100)            ← applied AFTER heap fetch (throws rows away)
  Rows Removed by Filter: 7
```

`Index Cond` uses the index to *seek*. `Filter` is applied *after* the row is fetched from the heap — it doesn't reduce heap fetches, it only reduces output. If you see a `Filter` removing many rows on an Index Scan, that column belongs in the index (as a composite key — Topic 44).

**Index Scan emits sorted output.** Because the B-tree is ordered, `ORDER BY customer_id` after an Index Scan on `customer_id` needs no Sort node:

```sql
EXPLAIN ANALYZE
SELECT * FROM orders WHERE customer_id BETWEEN 1 AND 100 ORDER BY customer_id;
-- → Index Scan, NO Sort node — the index already returns rows in customer_id order
```

This is a superpower of Index Scan that Bitmap Heap Scan lacks.

### 6.3 Index Only Scan — Skipping the Heap

An Index Only Scan is an Index Scan that *avoids the heap fetch* by reading column values directly from the index leaf. Two conditions must both hold:

1. **Coverage**: every column the query references (SELECT, WHERE, ORDER BY) must be in the index — either as a key column or an `INCLUDE` payload column (Topic 42).
2. **Visibility**: the heap page holding the row must be marked all-visible in the visibility map. If not, PostgreSQL *still* fetches the heap tuple to check MVCC visibility — that's a `Heap Fetch`.

```sql
-- Covering index: key + payload
CREATE INDEX orders_cust_covering
  ON orders (customer_id) INCLUDE (total_amount, status);

EXPLAIN (ANALYZE, BUFFERS)
SELECT customer_id, total_amount, status   -- all three columns are in the index
FROM orders
WHERE customer_id = 42;
```

```
Index Only Scan using orders_cust_covering on orders  (cost=0.42..4.6 rows=3 width=20)
                                                      (actual time=0.02..0.02 rows=3 loops=1)
  Index Cond: (customer_id = 42)
  Heap Fetches: 0                    ← THE marker: zero heap fetches, fully index-served
  Buffers: shared hit=3
Execution Time: 0.03 ms
```

`Heap Fetches: 0` means the VM bit was set for every matching page — pure index reads. Now watch what happens right after a bulk update, before `VACUUM`:

```
Index Only Scan using orders_cust_covering on orders  (cost=0.42..4.6 rows=3 width=20)
  Index Cond: (customer_id = 42)
  Heap Fetches: 3                    ← VM bits cleared by the UPDATE — degraded to heap fetches
  Buffers: shared hit=6              ← double the buffers: index pages + heap pages
```

Same plan node, but `Heap Fetches: 3` means it's doing the heap work anyway — effectively a regular Index Scan. The fix is `VACUUM orders` (or letting autovacuum catch up), which rebuilds the VM. **An Index Only Scan is only "only" if VACUUM keeps the visibility map current.** On write-heavy tables, tune autovacuum to run more aggressively on the hot tables.

**Why not just `SELECT *`?** Because `*` pulls columns not in the index, coverage fails, and you drop to a plain Index Scan. Index Only Scans reward *narrow* SELECT lists.

### 6.4 Bitmap Index Scan + Bitmap Heap Scan — The Scattered Middle

When a predicate matches too many rows for an efficient plain Index Scan (too much random I/O) but too few to justify a full Seq Scan, PostgreSQL splits the work into two phases:

1. **Bitmap Index Scan** scans the index and builds an in-memory bitmap — one bit per heap page (and, if memory allows, per tuple) that contains a match. This phase reads *only the index*, never the heap.

2. **Bitmap Heap Scan** walks that bitmap in *ascending page order*, reading each flagged heap page once, sequentially. Because it visits pages in physical order, it converts what would have been thousands of random single-page reads into one ordered sweep of just the needed pages.

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE status = 'refunded';   -- ~3% of 1M rows, scattered
```

```
Bitmap Heap Scan on orders  (cost=337.10..12750.00 rows=30000 width=68)
                            (actual time=4.1..38.2 rows=29880 loops=1)
  Recheck Cond: (status = 'refunded')
  Heap Blocks: exact=7912                 ← pages visited, in physical order
  Buffers: shared hit=7940
  ->  Bitmap Index Scan on orders_status_idx  (cost=0.00..329.60 rows=30000 width=0)
                                              (actual time=3.0..3.0 rows=29880 loops=1)
        Index Cond: (status = 'refunded')
        Buffers: shared hit=28            ← index-only phase: just 28 index pages
Planning Time: 0.12 ms
Execution Time: 41.0 ms
```

Reading it:
- The **inner** `Bitmap Index Scan` touched only 28 index pages to enumerate ~30K matches and build the bitmap.
- The **outer** `Bitmap Heap Scan` then read 7,912 heap pages *in order* (`Heap Blocks: exact`).
- A plain Index Scan here would have done ~30,000 *random* heap fetches. The bitmap collapses those into ~7,912 *ordered* reads (many matches share a page, and they're read sequentially). That's why the planner chose it.

**`exact` vs `lossy` heap blocks.** If the bitmap of matching *tuples* fits in `work_mem`, it stays *exact* (per-tuple precision) and the `Recheck Cond` is essentially free. If there are too many matches, the bitmap degrades to *lossy* — it only tracks which *pages* have matches, not which rows. Then the Bitmap Heap Scan must re-check the predicate on every tuple of those pages:

```
  Heap Blocks: exact=1024 lossy=41200    ← lossy: work_mem was too small
  Recheck Cond: (status = 'refunded')    ← now actually re-filtering every row on lossy pages
```

Seeing `lossy=` with a big number means raising `work_mem` would let the bitmap stay exact and cut the recheck cost.

**BitmapOr and BitmapAnd — combining indexes.** The killer feature of bitmap scans: multiple indexes can be combined *before* touching the heap.

```sql
EXPLAIN ANALYZE
SELECT * FROM orders
WHERE status = 'refunded' OR customer_id = 42;
```

```
Bitmap Heap Scan on orders
  Recheck Cond: ((status = 'refunded') OR (customer_id = 42))
  ->  BitmapOr
        ->  Bitmap Index Scan on orders_status_idx
              Index Cond: (status = 'refunded')
        ->  Bitmap Index Scan on orders_customer_id_idx
              Index Cond: (customer_id = 42)
```

Two separate indexes, each producing a bitmap, unioned by `BitmapOr` into one, then a single ordered heap sweep. A plain Index Scan **cannot** do this — it uses one index. `BitmapAnd` does the same for `AND` when two single-column indexes exist and no composite index covers both. (A composite index — Topic 44 — is usually better than relying on `BitmapAnd`, but the bitmap path means you don't *always* need one.)

### 6.5 The Selectivity Spectrum — The Governing Idea

Selectivity = fraction of rows a predicate matches. It is the single variable that drives scan choice. Roughly, as selectivity moves from "one row" to "the whole table":

```
matches   0.001%        0.1%          1–5%              5–25%            >25%
scan      Index Scan → Index Scan → Bitmap Heap Scan → Bitmap/Seq → Seq Scan
          (or Index    (or Index      (scattered rows,   (planner       (index
           Only Scan)   Only Scan)     ordered sweep)     compares)      pointless)
```

The crossover points are not fixed percentages — they depend on:
- **Table clustering/correlation** (`pg_stats.correlation`): if matching rows are physically clustered (high correlation, e.g. an index-ordered table), even 20% might use an Index Scan because the "random" fetches are actually near-sequential. If they're scattered (low correlation), the Index→Bitmap crossover comes much earlier.
- **`random_page_cost` / `seq_page_cost`**: the 4:1 default pushes the crossover toward Seq Scan; lowering `random_page_cost` pushes it toward Index Scan.
- **Row width and `LIMIT`**: a `LIMIT` favours a streaming Index Scan; wide rows favour Index Only / covering indexes.

The planner estimates selectivity from `pg_statistic` — the histogram, the most-common-values (MCV) list, and `n_distinct`, all populated by `ANALYZE`. **Stale statistics are the #1 cause of a wrong scan choice.**

### 6.6 How the Planner Prices Each Scan

Simplified cost formulas (the real ones are in `costsize.c`):

```
Seq Scan          = relpages × seq_page_cost + reltuples × cpu_tuple_cost
Index Scan        = index_descent_cost
                  + (matching_rows × random_page_cost × correlation_factor)   ← heap fetches
                  + matching_rows × cpu_tuple_cost
Index Only Scan   = index_descent_cost
                  + (fraction_not_all_visible × matching_rows × random_page_cost)  ← only non-VM pages
                  + matching_rows × cpu_tuple_cost
Bitmap Heap Scan  = bitmap_build_cost
                  + (heap_pages_touched × cost_between_seq_and_random)   ← ordered, so cheaper per page
                  + matching_rows × cpu_tuple_cost (+ recheck if lossy)
```

The pivotal term is the heap-fetch cost. Index Scan pays `random_page_cost` per *row* (adjusted by correlation). Bitmap Heap Scan pays a blended cost per *page* (fewer pages, read more sequentially). Index Only Scan pays the heap cost only for the *fraction of rows on not-all-visible pages*. This is exactly why the planner's ordering of preference shifts with selectivity, correlation, VM freshness, and the cost constants.

### 6.7 Correlation — The Hidden Variable

`pg_stats.correlation` measures how well the physical row order matches the indexed column's order, from `-1` to `+1`.

```sql
SELECT attname, correlation
FROM pg_stats
WHERE tablename = 'orders' AND attname IN ('id', 'created_at', 'customer_id');
```

```
  attname    | correlation
-------------+-------------
 id          |    0.999998   ← rows inserted in id order → nearly perfect
 created_at  |    0.981     ← append-only by time → highly correlated
 customer_id |    0.014     ← customers interleaved → essentially random
```

A range scan on a *highly correlated* column (`created_at`) fetches heap pages nearly sequentially even via an Index Scan — so the planner will happily use an Index Scan for a fairly wide range. The same-width range on `customer_id` (correlation ~0) means every heap fetch is a random jump — so the planner switches to a Bitmap Heap Scan (or Seq Scan) much earlier. `CLUSTER orders USING orders_created_at_idx` physically reorders the heap to raise correlation, widening the range over which Index Scans stay cheap. This is why two columns with identical selectivity can get different scan types.

### 6.8 How to Encourage the Scan You Want

You do not command a scan type directly (PostgreSQL has no query hints in core), but you have precise levers:

**To encourage an Index Scan / Index Only Scan:**
- Add the right index; for Index Only, make it *covering* (`INCLUDE` the payload columns) and keep the table well-vacuumed.
- Narrow the SELECT list so the index covers it.
- Lower `random_page_cost` (session or global) to reflect SSD reality: `SET random_page_cost = 1.1;`
- Ensure statistics are fresh: `ANALYZE table;`
- Raise `effective_cache_size` so the planner believes pages are cached (makes index paths cheaper).

**To encourage a Bitmap scan:**
- Ensure the predicate is moderately selective and the column has low correlation (bitmap shines on scattered matches).
- Raise `work_mem` so the bitmap stays *exact* rather than lossy.

**To discourage a bad Seq Scan (diagnostically only):**
- `SET enable_seqscan = off;` — **a diagnostic tool, not a production setting.** It doesn't truly disable Seq Scan; it adds a huge cost penalty so the planner avoids it if any alternative exists. Use it to *confirm* an index scan would be faster, then fix the real cause (statistics, `random_page_cost`, a missing index). Never leave it off in production.
- Similarly `enable_indexscan`, `enable_bitmapscan`, `enable_indexonlyscan` toggle the others — all diagnostic.

**The right permanent fixes** are almost always: (a) create/adjust the index, (b) run `ANALYZE`/tune autovacuum, (c) set `random_page_cost` and `effective_cache_size` to match your hardware. Query-level flag-flipping is for investigation, not for shipping.

### 6.9 Multicolumn Predicates and Index Usability

An index on `(a, b)` can drive an Index Scan for `WHERE a = ? AND b = ?`, `WHERE a = ?`, or `WHERE a = ? AND b > ?` — but *not* for `WHERE b = ?` alone (the leading column is missing; it can't seek). For a bare `b` predicate the planner may still use the index via a full **Index Scan/Index Only Scan that reads all leaf entries** (a "full index scan," cheaper than the heap if the index is small) or, more often, a Seq Scan. This is the boundary between this topic and Topic 44 (Composite Index Strategy), where leading-column rules are covered exhaustively.

### 6.10 Parallelism and Scan Types

Each scan type has a parallel-aware variant the planner can pick when the table is large and `max_parallel_workers_per_gather > 0`:
- **Parallel Seq Scan** — workers divide the heap blocks among themselves.
- **Parallel Index Scan / Parallel Index Only Scan** — workers divide B-tree leaf pages.
- **Parallel Bitmap Heap Scan** — one worker builds the bitmap; all workers share it and divide the heap pages.

Parallelism reduces wall-clock time for large scans but doesn't change the fundamental selectivity logic — it just throws more CPU at whichever scan was chosen. You'll see a `Gather` node above the parallel scan in EXPLAIN.

---

## 7. EXPLAIN — Scan Types Side by Side in the Plan

The single best way to internalise this topic is to run the *same table* through five predicates of increasing selectivity and watch the scan flip. Assume `orders` has 5,000,000 rows across ~42,000 heap pages, with these indexes:

```sql
-- pk on id, plus:
CREATE INDEX orders_customer_id_idx ON orders (customer_id);
CREATE INDEX orders_status_idx      ON orders (status);
CREATE INDEX orders_cust_covering   ON orders (customer_id) INCLUDE (total_amount, status);
```

### 7.1 Highest selectivity → Index Scan

```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM orders WHERE id = 998877;
```

```
Index Scan using orders_pkey on orders  (cost=0.43..8.45 rows=1 width=72)
                                        (actual time=0.028..0.029 rows=1 loops=1)
  Index Cond: (id = 998877)
  Buffers: shared hit=4
Planning Time: 0.09 ms
Execution Time: 0.05 ms
```

One row, four buffers (3 index + 1 heap). Nothing beats this.

### 7.2 High selectivity, covering → Index Only Scan

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT customer_id, total_amount, status FROM orders WHERE customer_id = 4242;
```

```
Index Only Scan using orders_cust_covering on orders  (cost=0.43..14.30 rows=6 width=20)
                                                      (actual time=0.031..0.034 rows=6 loops=1)
  Index Cond: (customer_id = 4242)
  Heap Fetches: 0
  Buffers: shared hit=4
Planning Time: 0.11 ms
Execution Time: 0.05 ms
```

`Heap Fetches: 0` — the covering index served all three columns and the pages were all-visible. Six rows in 0.05ms with zero heap access.

### 7.3 Same predicate, but SELECT * → falls back to Index Scan

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE customer_id = 4242;
```

```
Index Scan using orders_customer_id_idx on orders  (cost=0.43..38.60 rows=6 width=72)
                                                   (actual time=0.030..0.052 rows=6 loops=1)
  Index Cond: (customer_id = 4242)
  Buffers: shared hit=9
Planning Time: 0.10 ms
Execution Time: 0.07 ms
```

`SELECT *` pulls columns the index doesn't hold → coverage fails → plain Index Scan with heap fetches (9 buffers instead of 4). Same rows, more work, purely because of the SELECT list.

### 7.4 Moderate, scattered selectivity → Bitmap Heap Scan

```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM orders WHERE status = 'refunded';
--                                                       ~2.5% of rows, low correlation
```

```
Bitmap Heap Scan on orders  (cost=1387.4..58210.0 rows=125000 width=72)
                            (actual time=12.4..96.7 rows=124338 loops=1)
  Recheck Cond: (status = 'refunded')
  Heap Blocks: exact=39114
  Buffers: shared hit=39270
  ->  Bitmap Index Scan on orders_status_idx  (cost=0.00..1356.1 rows=125000 width=0)
                                              (actual time=11.9..11.9 rows=124338 loops=1)
        Index Cond: (status = 'refunded')
        Buffers: shared hit=156
Planning Time: 0.13 ms
Execution Time: 101.2 ms
```

125K scattered matches. The Bitmap Index Scan built the bitmap from 156 index pages; the Bitmap Heap Scan swept ~39K heap pages *in order*. A plain Index Scan here would have meant 125K random fetches — far worse. Note `Heap Blocks: exact` (no `lossy`) → `work_mem` was sufficient, so `Recheck Cond` was cheap.

### 7.5 Low selectivity → Sequential Scan

```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM orders WHERE status = 'completed';
--                                                       ~55% of rows
```

```
Seq Scan on orders  (cost=0.00..104334.0 rows=2750000 width=72)
                    (actual time=0.01..612.4 rows=2748901 loops=1)
  Filter: (status = 'completed')
  Rows Removed by Filter: 2251099
  Buffers: shared hit=42000
Planning Time: 0.08 ms
Execution Time: 745.0 ms
```

At 55% selectivity the index is pointless — any index path would touch nearly every heap page anyway, but with random I/O and index overhead on top. The planner reads all 42,000 pages sequentially. `Rows Removed by Filter` (~2.25M) confirms it filtered the whole table.

### 7.6 Watching a scan flip with LIMIT

```sql
EXPLAIN ANALYZE
SELECT * FROM orders WHERE status = 'refunded' ORDER BY id LIMIT 10;
```

```
Limit  (cost=0.43..44.1 rows=10 width=72) (actual time=0.05..0.14 rows=10 loops=1)
  ->  Index Scan using orders_pkey on orders  (cost=0.43..546000 rows=125000 width=72)
        Filter: (status = 'refunded')
        Rows Removed by Filter: 38
```

Even though `status = 'refunded'` alone would use a Bitmap scan, `ORDER BY id LIMIT 10` flips the plan to an ordered Index Scan on the PK that *streams* rows in id order and stops after 10 matches. The bitmap can't do this — it would enumerate all 125K matches before returning row one. This is the LIMIT-reshapes-cost effect from Section 3 made concrete.

### 7.7 EXPLAIN Signal Table

| What you see | What it means | Typical action |
|---|---|---|
| `Seq Scan` + huge `Rows Removed by Filter`, tiny output | Missing/unused index on a selective predicate | Add index; check `random_page_cost`; `ANALYZE` |
| `Index Scan` with a `Filter` removing many rows | Predicate column not in the index | Make a composite/covering index (Topic 44) |
| `Index Only Scan` with `Heap Fetches` > 0 (large) | Stale visibility map | `VACUUM` the table; tune autovacuum |
| `Bitmap Heap Scan` with `lossy=` large | `work_mem` too small for the bitmap | Raise `work_mem` |
| `Bitmap` chosen but you `LIMIT`/`ORDER BY` | Bitmap can't stream/sort — may be suboptimal | Consider index matching the ORDER BY |
| Estimated `rows` far from `actual rows` | Stale statistics → wrong scan chosen | `ANALYZE`; raise statistics target |
| `Seq Scan` where you expected an index | Predicate unselective, or planner thinks so | Verify true selectivity; check stats |

---

## 8. Query Examples

### Example 1 — Basic: Forcing the Diagnosis of a Seq Scan

```sql
-- A common "why is this slow?" investigation.
-- Step 1: see what you actually get.
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, customer_id, total_amount
FROM orders
WHERE customer_id = 55123;          -- expected: few rows, should be an Index Scan

-- If you instead see a Seq Scan with 5M Rows Removed by Filter, either:
--   (a) there is no index on customer_id, or
--   (b) statistics are stale and the planner mis-estimated selectivity.
-- Step 2: confirm an index would help (DIAGNOSTIC ONLY):
SET enable_seqscan = off;
EXPLAIN ANALYZE
SELECT id, customer_id, total_amount FROM orders WHERE customer_id = 55123;
RESET enable_seqscan;               -- ALWAYS reset — never ship with this off
-- Step 3: the real fix is the index + ANALYZE, not the flag.
```

### Example 2 — Intermediate: Building a Covering Index for an Index Only Scan

```sql
-- Hot endpoint: a customer's recent order summary, called thousands of times/min.
-- We want ZERO heap fetches. The query needs customer_id (filter),
-- created_at (order), and total_amount + status (output).

CREATE INDEX orders_cust_recent_covering
  ON orders (customer_id, created_at DESC)      -- key: seek + sort
  INCLUDE (total_amount, status);               -- payload: served without heap

EXPLAIN (ANALYZE, BUFFERS)
SELECT created_at, total_amount, status
FROM orders
WHERE customer_id = 4242
ORDER BY created_at DESC
LIMIT 20;
-- Expected: Index Only Scan, Heap Fetches: 0, no Sort node
-- (the DESC key order satisfies ORDER BY, INCLUDE covers the SELECT).
-- Requires the table to be well-vacuumed so the VM bits stay set.
```

### Example 3 — Production Grade: The Selectivity Crossover, Measured

```sql
-- Scenario: audit_logs, 80,000,000 rows, ~640,000 heap pages (8KB).
-- Column `action` has ~30 distinct values, wildly skewed:
--   'view'   → 71% of rows   (unselective)
--   'login'  →  8%
--   'delete' → 0.3%          (selective, scattered — low correlation)
-- Index available: CREATE INDEX audit_action_idx ON audit_logs(action);
-- correlation(action) ≈ 0.02 (values interleaved across the heap).
-- Perf expectation: 'delete' → Bitmap Heap Scan (~0.3% scattered);
--                   'view'   → Seq Scan (71% — index pointless).

-- Selective value → Bitmap:
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, user_id, action, created_at
FROM audit_logs
WHERE action = 'delete'
  AND created_at >= NOW() - INTERVAL '7 days';
```

```
Bitmap Heap Scan on audit_logs  (cost=... rows=42000 width=48)
                                (actual time=18.3..233.6 rows=41880 loops=1)
  Recheck Cond: (action = 'delete')
  Filter: (created_at >= (now() - '7 days'::interval))
  Rows Removed by Filter: 198000
  Heap Blocks: exact=61044
  Buffers: shared hit=2110 read=59090
  ->  Bitmap Index Scan on audit_action_idx  (cost=... rows=240000 width=0)
        Index Cond: (action = 'delete')
        Buffers: shared hit=658
Planning Time: 0.20 ms
Execution Time: 240.1 ms
```

```sql
-- Lesson from the plan: the Bitmap Index Scan found 240K 'delete' rows, but the
-- created_at Filter (NOT in the index) threw away 198K of them AFTER heap fetch.
-- The 61K heap-page sweep is dominated by that post-filter waste.
-- FIX: a composite index so BOTH predicates seek in the index (Topic 44):
CREATE INDEX audit_action_time_idx ON audit_logs (action, created_at);
-- Now: Index Cond: ((action='delete') AND (created_at >= ...)) — no wasteful Filter,
-- far fewer heap pages, and possibly an Index Scan instead of Bitmap because the
-- combined predicate is now highly selective. Re-run EXPLAIN to confirm.
```

---

## 9. Wrong → Right Patterns

### Wrong 1: `SELECT *` killing an Index Only Scan

```sql
-- WRONG: you built a covering index but SELECT * defeats it
CREATE INDEX orders_cust_covering ON orders (customer_id) INCLUDE (total_amount, status);

SELECT * FROM orders WHERE customer_id = 4242;
-- Plan: Index Scan (NOT Index Only) — * pulls columns outside the index,
-- so every matching row triggers a heap fetch. The INCLUDE payload is wasted.
```

```sql
-- RIGHT: select only the covered columns
SELECT customer_id, total_amount, status FROM orders WHERE customer_id = 4242;
-- Plan: Index Only Scan, Heap Fetches: 0. The whole point of the covering index
-- is realised only when the SELECT list is a subset of the index columns.
```

**Why at the execution level:** Index Only Scan eligibility is a *coverage* test over the entire column set the query touches. `*` expands to all columns; the index holds four; coverage fails; the planner must fetch the heap tuple for the rest — which is a plain Index Scan.

### Wrong 2: Expecting an Index Only Scan on a write-heavy table without VACUUM

```sql
-- WRONG assumption: "I have a covering index, so this is always heap-free"
SELECT customer_id, total_amount, status FROM orders WHERE customer_id = 4242;
-- On a table that just took 500K UPDATEs, the plan is still Index Only Scan, BUT:
--   Heap Fetches: 6      ← every match hit the heap anyway
--   Buffers: shared hit=10
-- The UPDATEs cleared the visibility-map bits; the VM says "not all-visible,"
-- so MVCC forces a heap visit to check tuple visibility. It's an Index Scan in disguise.
```

```sql
-- RIGHT: keep the visibility map current
VACUUM orders;   -- or, better, tune autovacuum for this hot table:
ALTER TABLE orders SET (autovacuum_vacuum_scale_factor = 0.02,
                        autovacuum_vacuum_cost_delay   = 0);
-- After VACUUM: Heap Fetches: 0 again. The Index Only Scan is only "only"
-- when VACUUM has marked the touched pages all-visible.
```

**Why at the execution level:** the index entry cannot prove a tuple is visible to your snapshot (MVCC — `xmin`/`xmax` live in the heap tuple). The visibility map is the shortcut; `VACUUM` maintains it; without it, the heap fetch returns.

### Wrong 3: Trusting a Seq Scan chosen from stale statistics

```sql
-- WRONG: after a 100× data load, the planner still thinks the table is tiny
-- and picks a Seq Scan (or a wrong scan) because pg_statistic is stale.
SELECT * FROM orders WHERE customer_id = 4242;
-- Plan: Seq Scan, actual rows=6, Rows Removed by Filter=4999994 — reads all 5M rows
-- to return 6. The planner estimated the table at 50K rows (pre-load stats).
```

```sql
-- RIGHT: refresh statistics so estimates match reality
ANALYZE orders;
-- Now the planner knows the table is 5M rows and customer_id=4242 matches ~6,
-- and it chooses an Index Scan. Stale stats are the #1 cause of wrong scan types;
-- ensure autovacuum's ANALYZE runs, or ANALYZE after bulk loads.
```

**Why at the execution level:** the cost model multiplies estimated row counts by per-page/per-row costs. If `reltuples`/histograms are stale, *every* cost estimate is wrong, and the cheapest-plan choice is made on fiction.

### Wrong 4: Leaving `random_page_cost = 4` on SSD, so index scans lose

```sql
-- WRONG: default cost constants on NVMe storage
SHOW random_page_cost;   -- 4  (assumes 1990s spinning disk)
-- A query matching 3% of a big table gets a Seq Scan because the planner
-- overprices the random heap fetches of the index/bitmap path 4:1 vs sequential.
SELECT * FROM orders WHERE status = 'refunded';   -- → Seq Scan (too pessimistic)
```

```sql
-- RIGHT: set cost constants to match the hardware
SET random_page_cost = 1.1;          -- session; or set in postgresql.conf
SET effective_cache_size = '24GB';   -- tell the planner most pages are cached
-- Now the 3% predicate correctly gets a Bitmap Heap Scan. This is a FLEET-WIDE
-- fix: one setting corrects the index-vs-seq crossover for every query at once.
```

**Why at the execution level:** `random_page_cost` is literally the price the cost model assigns to each random heap fetch. Set 4× too high, and every index/bitmap path is systematically overpriced relative to Seq Scan, biasing the planner away from indexes across the whole workload.

### Wrong 5: A `LIMIT` query stuck on a Bitmap scan because ORDER BY doesn't match any index

```sql
-- WRONG: want the 10 newest refunds, but ORDER BY column isn't indexed usefully
SELECT * FROM orders WHERE status = 'refunded' ORDER BY created_at DESC LIMIT 10;
-- Plan: Bitmap Heap Scan (125K rows) → Sort → Limit
-- It enumerates ALL 125K refunds, sorts them, THEN takes 10. Slow for a "top 10".
```

```sql
-- RIGHT: give it an index whose order matches the ORDER BY and whose leading
-- key(s) match the filter, so it can stream + stop early.
CREATE INDEX orders_status_created_idx ON orders (status, created_at DESC);
SELECT * FROM orders WHERE status = 'refunded' ORDER BY created_at DESC LIMIT 10;
-- Plan: Index Scan using orders_status_created_idx → Limit
-- It walks the index in (status='refunded', created_at DESC) order and stops after 10.
-- No bitmap, no full enumeration, no sort. Bitmap scans cannot stream ordered output.
```

**Why at the execution level:** a Bitmap Heap Scan emits rows in *heap* order, so any `ORDER BY` forces a blocking `Sort` over the entire matched set before `LIMIT` can apply. A matching B-tree Index Scan emits rows already ordered, letting `LIMIT` terminate the scan after 10 rows.

---

## 10. Performance Profile

### 10.1 Cost and Behaviour by Scan Type

| Scan | Heap I/O pattern | Memory | Emits sorted? | Streams (LIMIT-friendly)? | Combines indexes? |
|---|---|---|---|---|---|
| Seq Scan | All pages, sequential | O(1) | No | Yes (but reads all) | No |
| Index Scan | 1 random fetch per row | O(1) | Yes (index order) | Yes | No (one index) |
| Index Only Scan | 0 fetches (if VM set) | O(1) | Yes (index order) | Yes | No |
| Bitmap Index Scan | None (index only) | O(matches) bitmap | N/A | No (blocking) | Yes (BitmapAnd/Or) |
| Bitmap Heap Scan | Matched pages, sequential | Shares bitmap | No (heap order) | No (blocking) | Via bitmaps |

### 10.2 Scaling at 1M / 10M / 100M Rows

Assume a predicate matching **0.01%** of rows (highly selective) vs **5%** (scattered) vs **50%** (unselective):

| Rows | 0.01% match | 5% match | 50% match |
|---|---|---|---|
| 1M | Index Scan, ~0.1ms | Bitmap, ~30ms | Seq Scan, ~120ms |
| 10M | Index Scan, ~0.15ms | Bitmap, ~300ms | Seq Scan (parallel), ~700ms |
| 100M | Index Scan, ~0.2ms | Bitmap, ~3s (I/O bound) | Parallel Seq Scan, ~5–8s |

Key scaling insight: **a highly-selective Index Scan is nearly O(log N)** — it barely slows as the table grows from 1M to 100M, because it still touches only a handful of pages. The Bitmap and Seq paths scale roughly *linearly with the matched-page count*, which grows with the table. This is why the value of a selective index *increases* with table size — the bigger the table, the more an Index Scan saves relative to a Seq Scan.

### 10.3 The Index Only Scan Payoff at Scale

On a 100M-row table, a hot lookup returning 20 rows:
- **Plain Index Scan**: ~3 index pages + up to 20 random heap pages = 23 page accesses, several of them possibly cache misses (random). ~0.3–2ms depending on cache.
- **Index Only Scan (VM fresh)**: ~3 index pages, `Heap Fetches: 0`. ~0.05ms, fully cache-friendly.

At thousands of QPS the difference is enormous: the Index Only Scan touches ~7× fewer pages and generates *zero* random heap I/O, dramatically reducing buffer-cache churn. This is why covering indexes are a top-tier optimisation for read-hot endpoints — but only if autovacuum keeps the VM current (see 10.5).

### 10.4 work_mem and Bitmap Lossiness

A Bitmap Index Scan needs memory proportional to the number of matches to stay *exact*. Rough rule: exact tracking needs a few bits per matching tuple; when that exceeds `work_mem`, it degrades to per-page (lossy) tracking and the Bitmap Heap Scan re-checks the predicate on every tuple of every flagged page.

| Matches | work_mem = 4MB (default) | work_mem = 64MB |
|---|---|---|
| 100K | Exact | Exact |
| 5M | Lossy (recheck cost) | Exact |
| 50M | Very lossy (heavy recheck) | Partly lossy |

Symptom in EXPLAIN: `Heap Blocks: exact=X lossy=Y` with large `Y`. Raising `work_mem` (session-scoped for the query, not globally — it's per-operation memory) restores exactness and cuts CPU spent on rechecks.

### 10.5 Autovacuum Tuning for Index Only Scans

The visibility map is maintained by `VACUUM`. On append-mostly tables it stays fresh cheaply. On update/delete-heavy tables it decays fast, silently downgrading Index Only Scans to Index Scans (`Heap Fetches` climbs). Levers:

```sql
-- Make autovacuum vacuum this table more often (keeps VM fresh):
ALTER TABLE orders SET (
  autovacuum_vacuum_scale_factor  = 0.02,   -- vacuum after 2% of rows change (default 20%)
  autovacuum_vacuum_insert_scale_factor = 0.02,
  autovacuum_vacuum_cost_delay    = 0       -- don't throttle on this hot table
);
-- Monitor decay:
SELECT relname, n_dead_tup, last_autovacuum,
       round(100.0 * (SELECT count(*) FILTER (WHERE all_visible)
                      FROM ... ) , 1) AS pct_all_visible  -- (illustrative)
FROM pg_stat_user_tables WHERE relname = 'orders';
```

### 10.6 CLUSTER and Correlation

For range scans on a low-correlation column, `CLUSTER table USING some_index` physically reorders the heap to match the index, raising `pg_stats.correlation` toward 1. This widens the selectivity range over which a plain Index Scan (rather than Bitmap/Seq) stays cheap, because heap fetches become near-sequential. Caveat: `CLUSTER` takes an `ACCESS EXCLUSIVE` lock and is a one-time reorder — new rows aren't kept clustered. Use it for read-heavy tables that are reloaded/maintained in batches, or use `pg_repack` for online reclustering.

### 10.7 Optimisation Checklist Specific to Scan Types

1. Match `random_page_cost` / `effective_cache_size` to hardware (biggest single fix).
2. Add covering indexes (`INCLUDE`) for read-hot narrow queries → Index Only Scan.
3. Keep the VM fresh via autovacuum on write-heavy tables.
4. Add composite indexes so predicates seek in the `Index Cond` instead of a post-heap `Filter` (Topic 44).
5. Raise `work_mem` for queries whose bitmaps go lossy.
6. `CLUSTER` low-correlation columns used in range scans.
7. Keep statistics fresh; raise `default_statistics_target` for skewed columns.

---

## 11. Node.js Integration

### 11.1 Reading the Plan Programmatically to Assert Scan Type

```javascript
import pg from 'pg';
const { Pool } = pg;
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

// Run EXPLAIN in JSON form so you can assert on the chosen scan in tests/monitoring.
async function getPlan(sql, params = []) {
  const { rows } = await pool.query(
    `EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) ${sql}`,
    params
  );
  return rows[0]['QUERY PLAN'][0].Plan;   // the root plan node
}

// Example: assert a hot endpoint query stays on an Index Only Scan with 0 heap fetches.
async function assertIndexOnly() {
  const plan = await getPlan(
    `SELECT customer_id, total_amount, status FROM orders WHERE customer_id = $1`,
    [4242]
  );
  if (plan['Node Type'] !== 'Index Only Scan') {
    throw new Error(`Regression: expected Index Only Scan, got ${plan['Node Type']}`);
  }
  if (plan['Heap Fetches'] > 0) {
    console.warn(`VM stale: ${plan['Heap Fetches']} heap fetches — VACUUM needed`);
  }
}
```

### 11.2 Parameterised Query That Stays on the Index

```javascript
// $1 binding keeps the value out of the SQL text (no injection) and lets the
// planner reuse a generic plan. The predicate is selective → Index Scan.
async function getOrder(orderId) {
  const { rows } = await pool.query(
    `SELECT id, customer_id, total_amount, status, created_at
       FROM orders
      WHERE id = $1`,
    [orderId]
  );
  return rows[0] ?? null;
}

// Range + LIMIT that should stream from a matching index (status, created_at DESC).
async function recentByStatus(status, limit = 20) {
  const { rows } = await pool.query(
    `SELECT id, customer_id, total_amount, created_at
       FROM orders
      WHERE status = $1
      ORDER BY created_at DESC
      LIMIT $2`,
    [status, limit]
  );
  return rows;   // Index Scan streams, stops after $2 rows — no full bitmap enumeration
}
```

### 11.3 The Generic-Plan Trap With Prepared Statements

```javascript
// pg reuses a prepared statement across calls. After 5 executions, PostgreSQL may
// switch from a custom plan (re-planned per parameter) to a GENERIC plan (planned once,
// parameter-agnostic). For skewed columns this can lock in the WRONG scan type:
//   - value 'view'   (71% of rows) wants a Seq Scan
//   - value 'delete' (0.3%)        wants a Bitmap/Index Scan
// A single generic plan can't be right for both.

// Mitigations:
//   1. Keep predicates on non-skewed columns where possible.
//   2. For heavily skewed columns, force custom plans:
async function forceCustomPlan(client) {
  await client.query(`SET plan_cache_mode = force_custom_plan`);  // re-plan every time
}
//   3. Or inline the skewed value (carefully, still parameterising other inputs).
// Verify with EXPLAIN using a representative value, not just the first one you tested.
```

### 11.4 Tuning Cost Constants at the Session Level

```javascript
// If your app runs analytical scans on SSD and the planner over-prefers Seq Scans,
// you can nudge cost constants per connection (e.g. a read-replica pool).
async function configureReadPool(client) {
  await client.query(`SET random_page_cost = 1.1`);      // SSD-realistic
  await client.query(`SET effective_cache_size = '24GB'`);
  await client.query(`SET work_mem = '64MB'`);           // keep bitmaps exact
}
// Better: set these in postgresql.conf / per-role so every session inherits them.
// Note: the pg driver does NOT manage any of this — scan choice is 100% the server's job.
```

**Does the driver do any of this?** No. `pg` is a thin protocol client — it sends SQL and returns rows. Scan-type selection, cost estimation, and the visibility map are entirely server-side. The only Node-side levers are *what SQL you send* (narrow SELECT, matching ORDER BY), *how you parameterise* (custom vs generic plans), and *session settings* you issue via `SET`.

---

## 12. ORM Comparison

The central question for every ORM: **can it produce SQL that keeps the good scan type** — narrow SELECT lists (for Index Only Scans), predicates that hit the index (`Index Cond`, not `Filter`), and ORDER BY that matches an index? Most ORMs happily produce `SELECT *` and can't express `INCLUDE` indexes, so the escape hatch is index DDL + raw EXPLAIN.

### Prisma

**Can it?** Partly. Prisma's `select` controls the column list (crucial for Index Only Scans), but Prisma cannot create `INCLUDE`/covering indexes in its schema — you add those via raw migrations.

```typescript
// GOOD: narrow select → gives the planner a shot at an Index Only Scan
const rows = await prisma.order.findMany({
  where: { customerId: 4242 },
  select: { customerId: true, totalAmount: true, status: true },  // no SELECT *
});

// Covering index must be added by raw migration (Prisma schema can't express INCLUDE):
//   CREATE INDEX orders_cust_covering ON orders (customer_id) INCLUDE (total_amount, status);

// Inspect the actual plan:
const plan = await prisma.$queryRawUnsafe(
  `EXPLAIN (ANALYZE, FORMAT JSON)
   SELECT customer_id, total_amount, status FROM orders WHERE customer_id = 4242`
);
```

**Where it breaks:** default `findMany` without `select` returns all scalar columns → `SELECT *` semantics → no Index Only Scan. No way to express covering indexes or force scan types. **Verdict:** use `select` religiously on hot paths; manage covering indexes and EXPLAIN via raw SQL.

### Drizzle

**Can it?** Yes, closest to SQL. You pick exact columns, and you can run `EXPLAIN` inline.

```typescript
import { db } from './db';
import { orders } from './schema';
import { eq, sql } from 'drizzle-orm';

// Narrow projection → Index Only Scan eligible
const rows = await db
  .select({ customerId: orders.customerId, total: orders.totalAmount, status: orders.status })
  .from(orders)
  .where(eq(orders.customerId, 4242));

// EXPLAIN through the sql escape hatch
const plan = await db.execute(sql`
  EXPLAIN (ANALYZE, BUFFERS)
  SELECT customer_id, total_amount, status FROM orders WHERE customer_id = 4242
`);
```

**Where it breaks:** covering `INCLUDE` indexes are declared in Drizzle's index builder in recent versions, but complex partial/expression indexes still often go in raw SQL migrations. **Verdict:** best-in-class control over the SELECT list; the transparent `sql` tag makes verifying scans easy.

### Sequelize

**Can it?** Yes, but the defaults fight you. `attributes` controls columns; without it you get `SELECT *`.

```javascript
// GOOD: restrict attributes so a covering index can serve it
const rows = await Order.findAll({
  where: { customerId: 4242 },
  attributes: ['customerId', 'totalAmount', 'status'],   // avoid SELECT *
});

// Covering index: declare in a raw migration (Sequelize index API has no INCLUDE).
// EXPLAIN via raw query:
const [plan] = await sequelize.query(
  `EXPLAIN (ANALYZE) SELECT customer_id, total_amount, status FROM orders WHERE customer_id = :id`,
  { replacements: { id: 4242 } }
);
```

**Where it breaks:** eager-loading (`include`) tends to generate wide multi-table `SELECT *` with LEFT JOINs, which almost never produces Index Only Scans and often forces large scans. No `INCLUDE` index support. **Verdict:** set `attributes` explicitly; push covering indexes and analytics to raw SQL.

### TypeORM

**Can it?** Yes via QueryBuilder `.select([...])`; entity `find` defaults to all columns.

```typescript
const rows = await ds.getRepository(Order)
  .createQueryBuilder('o')
  .select(['o.customerId', 'o.totalAmount', 'o.status'])   // narrow → Index Only eligible
  .where('o.customerId = :id', { id: 4242 })
  .getRawMany();

// EXPLAIN:
const plan = await ds.query(
  `EXPLAIN (ANALYZE) SELECT customer_id, total_amount, status FROM orders WHERE customer_id = $1`,
  [4242]
);
```

**Where it breaks:** repository `find({ where })` selects all entity columns (SELECT *). `@Index` decorators can't express `INCLUDE`; covering indexes need raw migrations. **Verdict:** use QueryBuilder with explicit `.select()` for hot reads; raw SQL for covering indexes and plan inspection.

### Knex

**Can it?** Yes — most transparent. You write the column list; `.explain()`-style raw is trivial.

```javascript
const rows = await knex('orders')
  .select('customer_id', 'total_amount', 'status')   // narrow projection
  .where('customer_id', 4242);

const plan = await knex.raw(
  `EXPLAIN (ANALYZE, BUFFERS)
   SELECT customer_id, total_amount, status FROM orders WHERE customer_id = ?`,
  [4242]
);
// Covering index in a migration:
//   knex.raw('CREATE INDEX orders_cust_covering ON orders (customer_id) INCLUDE (total_amount, status)')
```

**Where it breaks:** nothing structural — Knex won't stop you writing `select('*')`, and `INCLUDE` indexes go through `knex.raw`. **Verdict:** as good as raw SQL for controlling scan-relevant SQL.

### ORM Summary Table

| ORM | Narrow SELECT? | Declares INCLUDE index? | Easy EXPLAIN? | Verdict |
|---|---|---|---|---|
| Prisma | `select` (opt-in) | No — raw migration | `$queryRaw` | Use `select`; raw for covering indexes |
| Drizzle | Yes, explicit | Recent versions / raw | `sql` tag | Best typed control |
| Sequelize | `attributes` | No — raw migration | raw query | Set `attributes`; watch eager-load `*` |
| TypeORM | QueryBuilder `.select()` | No — raw migration | `.query()` | QueryBuilder for hot reads |
| Knex | Yes, explicit | Via `knex.raw` | `knex.raw` | Most transparent |

**Universal truth:** no ORM lets you *command* a scan type — that's the planner's job. What ORMs control is the *inputs* to that decision (column list, predicates, ORDER BY) and whether the right indexes exist. Covering (`INCLUDE`) indexes almost always require a raw migration in every ORM. Always verify hot-path scans with `EXPLAIN (ANALYZE, BUFFERS)` — the ORM's generated SQL is often wider than you think.

---

## 13. Practice Exercises

### Exercise 1 — Easy

You have `products(id PK, category_id, name, price, in_stock)` with 200,000 rows and an index `CREATE INDEX products_category_idx ON products (category_id);`. Category 7 contains ~40 products.

1. Write the query to fetch `id, name, price` for category 7.
2. Predict the scan type and justify it in one sentence (consider selectivity: 40 / 200,000).
3. Write the `EXPLAIN` command you'd run to confirm.

```sql
-- Write your query here
```

### Exercise 2 — Medium (combines Topic 42 index types)

Using the same `products` table, a hot endpoint runs thousands of times per minute:

```sql
SELECT category_id, price FROM products WHERE category_id = $1 ORDER BY price ASC LIMIT 25;
```

1. Design the *single* index that makes this an **Index Only Scan** with **no Sort node** and **`Heap Fetches: 0`**. (Hint: leading key for the filter, second key for the order, and think about coverage — do you need `INCLUDE`, or do the key columns already cover it?)
2. State the two runtime conditions that must hold for `Heap Fetches` to actually be 0 at request time.
3. What breaks the Index Only Scan if a developer later changes the SELECT to `SELECT *`?

```sql
-- Write your index and reasoning here
```

### Exercise 3 — Hard (production simulation; the naive answer is wrong)

`audit_logs(id, user_id, action, created_at)` has 80,000,000 rows. `action` is skewed: `'view'` = 70%, `'delete'` = 0.2%. There is an index on `action` only. A dashboard runs:

```sql
SELECT id, user_id, created_at
FROM audit_logs
WHERE action = $1 AND created_at >= NOW() - INTERVAL '1 day'
ORDER BY created_at DESC
LIMIT 50;
```

The team reports it's "sometimes instant, sometimes 4 seconds," with no code change between calls.

1. Explain, in scan-type terms, why the *same query* varies so much depending on `$1`.
2. The query uses a prepared statement executed on every request. Explain how the generic-plan mechanism could lock in the wrong scan for one of the two values.
3. Design the index(es) and/or settings that make **both** `'view'` and `'delete'` fast, and state the scan type each value should end up with.
4. The current EXPLAIN shows a `Filter: (created_at >= ...)` removing millions of rows on a Bitmap Heap Scan. What exactly is wrong, and what makes the `Filter` become an `Index Cond` instead?

```sql
-- Write your diagnosis, indexes, and settings here
```

### Exercise 4 — Interview Simulation

An interviewer gives you this EXPLAIN and asks you to diagnose and fix it, thinking aloud:

```
Index Only Scan using orders_cust_covering on orders  (cost=0.43..14.3 rows=6 width=20)
                                                      (actual time=0.4..142.0 rows=6 loops=1)
  Index Cond: (customer_id = 4242)
  Heap Fetches: 6
  Buffers: shared hit=9 read=3
```

1. The plan is an Index Only Scan, yet it took 142ms for 6 rows. What's wrong?
2. What server-side maintenance operation fixes it, and why does that operation specifically matter here?
3. How would you prevent recurrence on a table that takes constant UPDATEs?
4. What would `Heap Fetches: 0` change in the `Buffers` line?

```sql
-- Write your answer here
```

---

## 14. Interview Questions

### Q1 — "Walk me through how PostgreSQL decides between a Sequential Scan and an Index Scan."

**Junior answer:** "If there's an index on the column in the WHERE clause, it uses the index; otherwise it does a sequential scan."

**Principal answer:** "It's a cost comparison driven by *selectivity*, not merely index existence. The planner estimates how many rows the predicate matches using `pg_statistic` — the histogram, MCV list, and `n_distinct` from the last `ANALYZE`. It then prices each candidate path: a Seq Scan costs roughly `relpages × seq_page_cost` regardless of matches; an Index Scan costs the B-tree descent plus one heap fetch per matching row at `random_page_cost` each, adjusted by `pg_stats.correlation` (clustered rows make those fetches near-sequential and cheaper). When the predicate is selective — say under ~1% — the index path is far cheaper and wins. As selectivity rises, the per-row random-fetch cost of the index path grows until, somewhere around a few percent to tens of percent depending on correlation and the cost constants, a Bitmap Heap Scan and then a Seq Scan become cheaper. So the same query on the same table flips scan types purely based on the *parameter value's* selectivity and the freshness of statistics. The classic bug is stale statistics: the planner estimates the wrong row count and picks a Seq Scan that reads 5M rows to return 6."

**Follow-up:** "Your table is on NVMe SSD and you're still getting Seq Scans for 2%-selective predicates. What's the fix?" *(Expected: lower `random_page_cost` from 4 to ~1.1 and raise `effective_cache_size`; the default 4:1 ratio models spinning disks and systematically overprices index paths.)*

### Q2 — "What is an Index Only Scan, and why does it sometimes show non-zero `Heap Fetches`?"

**Junior answer:** "It's when the query only needs columns that are in the index, so it doesn't read the table."

**Principal answer:** "Right on coverage, but the subtle part is *visibility*. Because of MVCC, an index entry alone can't prove a tuple is visible to your snapshot — the `xmin`/`xmax` that determine visibility live in the heap tuple, not the index. So even with full column coverage, PostgreSQL must confirm visibility. The shortcut is the *visibility map*: one bit per heap page meaning 'all tuples on this page are visible to everyone.' When that bit is set, the Index Only Scan reads values straight from the index leaf and skips the heap — `Heap Fetches: 0`. When it's unset — which happens whenever a page has been modified since the last `VACUUM` — the scan must fetch the heap tuple to check visibility, and that shows as `Heap Fetches: N`. So a non-zero count means the visibility map is stale, and the 'index only' scan has silently degraded to a regular Index Scan. The fix is `VACUUM`, which rebuilds the VM; on write-heavy tables you tune autovacuum to run more aggressively so the VM stays current."

**Follow-up:** "You `SELECT *` from a table with a covering index. Do you get an Index Only Scan?" *(Expected: no — `*` pulls columns outside the index, coverage fails, and it falls back to a plain Index Scan with heap fetches.)*

### Q3 — "When does the planner choose a Bitmap Heap Scan over a plain Index Scan, and what's the tradeoff?"

**Junior answer:** "When there are a lot of rows to return."

**Principal answer:** "When the matched rows are *moderately numerous and physically scattered* — too many for a plain Index Scan's per-row random heap fetches to be cheap, but too few to justify reading the whole table. A plain Index Scan does one random heap fetch per matching row in index order, so 100K scattered matches means ~100K random I/Os, possibly re-reading the same page many times. A Bitmap Index Scan instead reads only the index to build an in-memory bitmap of matching *pages*, and the Bitmap Heap Scan then reads those pages once each, *in physical order* — converting random I/O into a sequential sweep of just the needed pages. The tradeoffs: first, it's *blocking* — it must build the whole bitmap before returning any row, so it's bad for `LIMIT` (a plain ordered Index Scan can stream and stop early). Second, it emits rows in *heap order*, not index order, so it can't satisfy `ORDER BY` for free — you'd need a Sort. Third, if the bitmap exceeds `work_mem` it goes *lossy* (page-granular), forcing a per-tuple `Recheck Cond`. Its unique superpower, though, is combining indexes: `BitmapOr`/`BitmapAnd` can union or intersect bitmaps from multiple indexes before touching the heap — something no single Index Scan can do — which is why `WHERE a=1 OR b=2` often becomes a bitmap plan."

**Follow-up:** "You have a `WHERE status='x' ORDER BY created_at DESC LIMIT 10` that's using a Bitmap scan and it's slow. Why, and how do you fix it?" *(Expected: bitmap enumerates all matches then sorts before LIMIT; add an index on `(status, created_at DESC)` so an Index Scan streams in order and stops after 10.)*

### Q4 — "How does `random_page_cost` influence scan selection, and how would you set it?"

**Junior answer:** "It's the cost of reading a page; leave it at the default."

**Principal answer:** "`random_page_cost` is the modeled cost of a *random* single-page read, relative to `seq_page_cost` (1.0). It's the price tag on every heap fetch an Index Scan or Bitmap Heap Scan performs. The default is 4.0, encoding the ~4:1 random-to-sequential penalty of spinning disks. On SSD/NVMe the real penalty is much smaller — often 1.1 to 1.5 — so leaving it at 4.0 makes the planner systematically *overprice* index and bitmap paths and over-prefer Seq Scans across the entire workload. I'd set it to ~1.1 on NVMe (measuring first), and pair it with a realistic `effective_cache_size` (say 60–75% of RAM) so the planner knows most pages are cached, which further favours index paths. It's one of the highest-leverage settings because it's fleet-wide: a single change fixes the index-vs-seq crossover for every query at once, whereas rewriting queries fixes them one at a time. I'd change it globally in `postgresql.conf`, or per-role for a read replica, and re-check a sample of plans after."

**Follow-up:** "Two columns have identical selectivity for a range query, but one uses an Index Scan and the other a Bitmap. Same cost constants. Why?" *(Expected: `pg_stats.correlation` — the highly-correlated column's heap fetches are near-sequential so Index Scan stays cheap; the uncorrelated column's fetches are random, so the planner switches to a bitmap sooner. `CLUSTER` can raise correlation.)*

---

## 15. Mental Model Checkpoint

1. A predicate matches 3 rows out of 50 million. Another matches 20 million out of 50 million. Describe the scan type you expect for each, and explain why the *cost* of the index path grows with the number of matching rows while the Seq Scan cost stays roughly flat.

2. You have a covering index and a narrow SELECT, yet `EXPLAIN` shows `Index Only Scan` with `Heap Fetches: 500000`. The index is correct and the columns are covered. What single fact about the *table's recent write activity* explains this, and what operation fixes it?

3. Explain why a Bitmap Heap Scan reads heap pages in *physical order* while a plain Index Scan reads them in *index order*, and why that difference makes the Bitmap unable to satisfy an `ORDER BY` without a Sort but able to turn thousands of random reads into one sequential sweep.

4. Two columns, `created_at` and `customer_id`, have the same number of distinct values and the same selectivity for a given range query, but `created_at` gets an Index Scan and `customer_id` gets a Bitmap Heap Scan. What table-level statistic differs between them, and what physical operation could make `customer_id` behave like `created_at`?

5. Adding `ORDER BY id LIMIT 10` to a query changed its plan from a Bitmap Heap Scan to a plain Index Scan on the primary key — even though the WHERE predicate is unchanged and would prefer a bitmap on its own. Explain the planner's reasoning in terms of *streaming vs blocking* execution.

6. Why does `SET enable_seqscan = off` not actually remove the possibility of a Seq Scan, and why is it a diagnostic tool rather than a production setting? What would you do with the information it gives you?

7. You lower `random_page_cost` from 4 to 1.1 and half your slow queries speed up while nothing gets slower. Explain, in terms of the cost model, why one setting change fixed many unrelated queries at once — and what class of hardware makes the default value wrong in the first place.

---

## 16. Quick Reference Card

```sql
-- ── THE FIVE SCAN TYPES ────────────────────────────────────────────
-- Seq Scan          reads ALL heap pages sequentially, Filter per row.
--                   Wins when: predicate unselective (>~25%) or no index.
-- Index Scan        B-tree seek + 1 random heap fetch per row, INDEX ORDER.
--                   Wins when: highly selective; ORDER BY matches index; LIMIT.
-- Index Only Scan   Index Scan with NO heap fetch — needs coverage + VM bit set.
--                   Marker of success: Heap Fetches: 0.
-- Bitmap Index Scan build in-memory bitmap of matching pages (index only, no heap).
-- Bitmap Heap Scan  read flagged pages in PHYSICAL order (+ Recheck). HEAP ORDER.
--                   Wins when: moderate, scattered matches; OR across indexes.

-- ── SELECTIVITY SPECTRUM (rough) ───────────────────────────────────
-- <0.1% ...... Index / Index Only Scan
-- ~1–5% ...... Bitmap Heap Scan (scattered) — depends on correlation
-- >25% ....... Seq Scan

-- ── READING EXPLAIN ────────────────────────────────────────────────
-- Index Cond   → used the index to SEEK (good)
-- Filter       → applied AFTER heap fetch (throws rows away — wants a better index)
-- Heap Fetches → 0 = pure index only; >0 = stale visibility map, VACUUM
-- Heap Blocks: exact=X lossy=Y  → large Y means work_mem too small
-- Rows Removed by Filter (huge) + tiny output → missing/unusable index

-- ── ENCOURAGE A SCAN (levers, not commands) ────────────────────────
CREATE INDEX ... INCLUDE (payload);   -- enable Index Only Scan (covering)
VACUUM table;                          -- refresh visibility map (Index Only)
ANALYZE table;                         -- fresh stats → correct selectivity
SET random_page_cost = 1.1;            -- SSD: stop overpricing index paths
SET effective_cache_size = '24GB';     -- planner believes pages are cached
SET work_mem = '64MB';                 -- keep bitmaps exact (avoid lossy)
CLUSTER table USING idx;               -- raise correlation → wider Index Scan range
-- DIAGNOSTIC ONLY (never ship):
SET enable_seqscan = off;              -- confirm an index path would be faster
RESET enable_seqscan;

-- ── COST CONSTANTS ─────────────────────────────────────────────────
-- seq_page_cost    = 1.0   (sequential page read)
-- random_page_cost = 4.0   (random page read — set ~1.1 on SSD)
-- cpu_tuple_cost   = 0.01  (per-row CPU)

-- ── INTERVIEW ONE-LINERS ───────────────────────────────────────────
-- "Scan choice is a cost comparison driven by selectivity, not index existence."
-- "Index Only Scan needs coverage AND a fresh visibility map — VACUUM keeps it 'only'."
-- "Bitmap = random reads turned into one ordered heap sweep; blocking, so bad for LIMIT."
-- "random_page_cost=4 assumes spinning disks; on SSD it silently kills index scans."
-- "Filter in the plan = the index didn't cover that predicate; make it an Index Cond."
-- "Correlation decides how far an Index Scan stays cheap before the bitmap wins."
```

---

## Connected Topics

**Internals this builds on:**
- **B-tree structure & the heap/TID model** — the sorted index leaves and `(block, offset)` pointers that make Index Scans and heap fetches possible.
- **MVCC & the visibility map** — why Index Only Scans depend on `VACUUM`; why `Heap Fetches` appears.
- **The buffer pool & `EXPLAIN (BUFFERS)`** — `shared hit`/`read` as the empirical measure of pages touched by each scan.
- **The cost model (`costsize.c`)** — `seq_page_cost`, `random_page_cost`, `cpu_tuple_cost`, and how `pg_statistic` selectivity estimates price each path.

**Prior / next SQL topics:**
- **Topic 42 — Index Types**: the structures (B-tree, hash, GIN, GiST, BRIN, `INCLUDE` covering indexes) whose *existence* creates the access paths this topic *chooses among*. Read it first — you can't reason about scan choice without knowing which indexes are available.
- **Topic 44 — Composite Index Strategy** *(next)*: how multicolumn indexes turn wasteful `Filter` steps into precise `Index Cond` seeks, enable Index Only Scans for more queries, and satisfy `ORDER BY` — directly extending the scan-quality ideas here.
- **Topic 11 — INNER JOIN in Depth**: Nested Loop joins perform an inner Index Scan per outer row — the same scan-selection logic applies inside each loop iteration.
- **EXPLAIN & Query Planning (Phase 7)**: the broader skill of reading plans, of which scan-type identification is the foundation.
- **VACUUM & Autovacuum (Maintenance)**: the operational discipline that keeps Index Only Scans fast and statistics fresh.
