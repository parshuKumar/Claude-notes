# Topic 44 — Composite Index Strategy
### SQL Mastery Curriculum — Phase 7: Indexes and Query Optimisation

---

## 1. ELI5 — The Analogy That Makes It Click

Imagine a phone book. It is sorted by **last name first, then first name**. This is a composite index on `(last_name, first_name)`.

Now try to answer these questions using the book:

- **"Find everyone with last name Smith."** Easy. Flip to the S section, all the Smiths are together. The book's sort order helps you.
- **"Find everyone with last name Smith and first name John."** Even easier. Within the Smiths, the Johns are together too. Both parts of the sort order help.
- **"Find everyone with first name John."** Useless. Johns are scattered across every last name — John Adams, John Baker, John Carter, John Smith. You'd have to read the *entire* book. The sort order gives you nothing, because first name is the *second* thing the book is sorted by, not the first.

That last case is the single most important idea in this entire topic. A composite index is sorted by its columns **in order**, like a phone book sorted by last-then-first. You can use it to jump straight to a value only if you start from the **leftmost** column and don't skip any. Ask for the second column alone, and the index can't help you find a starting point — the matching rows are smeared across the whole structure.

The order of columns in a composite index is not a stylistic choice. It decides which queries fly and which queries crawl. Get the order wrong and you've built a phone book sorted by first name for a world that searches by last name.

---

## 2. Connection to SQL Internals

A composite index in PostgreSQL is a **single B-tree** whose keys are *tuples* of column values, compared **lexicographically** — exactly like sorting words alphabetically, where you compare the first letter, and only if it ties do you look at the second letter.

For an index on `(a, b, c)`, the key for a row is the triple `(a_value, b_value, c_value)`. Two keys are ordered by comparing `a` first; if `a` is equal, compare `b`; if `b` is also equal, compare `c`. The entire index is one flat sorted sequence of these tuples, stored in B-tree pages.

This lexicographic ordering is the *root cause* of every rule in this topic:

- **Leftmost-prefix rule** — because the tuples are sorted by `a` first, all rows sharing an `a` value are physically contiguous. A predicate on `a` maps to a contiguous range of the B-tree. A predicate on `b` *alone* does not, because for a fixed `b` the rows are scattered across every `a`.
- **Equality-before-range** — an equality on `a` pins you to one `a` value, inside which rows are still sorted by `b`. A range on `a` (`a > 5`) spans many `a` values, and across that span the `b` ordering is broken — so a `b` predicate after a range on `a` can no longer use the index tree to seek.
- **Covering / index-only scans** — if every column the query needs is *in the index tuple*, PostgreSQL can answer from the index pages alone and never touch the heap. This is the `INCLUDE` clause's whole reason to exist.

Other internals in play:
- **Heap pages & the visibility map** — an index-only scan still must confirm each row is visible to your MVCC snapshot (Topic 05). PostgreSQL consults the *visibility map*; only all-visible heap pages let it skip the heap fetch. A composite/covering index reduces heap access but doesn't eliminate visibility checks.
- **The planner's cost model** — it estimates index selectivity from `pg_statistic` (per-column stats plus, if created, multi-column extended statistics). Column order changes selectivity of usable prefixes, which changes the plan.
- **Buffer pool** — a narrow, well-ordered composite index is smaller and hotter in `shared_buffers` than several wide single-column indexes. Fewer pages read = fewer buffer misses.

Remember from Topic 43 (Index Scan Types): the *access method* (Index Scan, Index-Only Scan, Bitmap Index Scan) is chosen by the planner. Composite index design is about giving the planner an index whose leftmost columns match your predicates so those fast access methods become available at all.

---

## 3. Logical Execution Order Context

```
FROM
JOIN            ← composite index on the join key(s) enables Merge/Nested-Loop seeks
WHERE           ← the primary consumer: equality + range predicates map to index prefixes
GROUP BY        ← a matching index order can supply pre-grouped input (no Sort/Hash)
HAVING
SELECT          ← covering / INCLUDE columns let this read straight from the index
DISTINCT        ← index order can satisfy DISTINCT without a separate sort
window          ← PARTITION BY / ORDER BY can be fed by a matching index order
ORDER BY        ← a composite index in the same column order removes the Sort node
LIMIT           ← index order + LIMIT = "top-N" without scanning/sorting everything
```

A composite index is unusual: it participates in **almost every clause**. It is not a WHERE-only tool.

Where the ordering rules bite:
- **WHERE** decides which *leftmost prefix* of the index columns can be used to seek. Everything hinges on which columns appear with equality vs range.
- **ORDER BY** can be satisfied *for free* if the index's column order (and direction) matches the requested sort — but only for the rows the WHERE prefix leaves in index order (see 6.6).
- **GROUP BY / DISTINCT** can also be satisfied by index order, letting the planner use a streaming aggregate instead of a hash aggregate or an explicit sort.
- **LIMIT** is where matching-index-order pays off most: an `ORDER BY ... LIMIT 10` served by an index reads ~10 rows, not the whole table.

The deep lesson: WHERE and ORDER BY *compete* for the index's column order. The same index cannot always be optimal for both. Designing composite indexes is largely about resolving that competition (see 6.6 and 6.7).

---

## 4. What Is a Composite Index?

A **composite index** (also called a multi-column or concatenated index) is a single index built over **two or more columns**, whose entries are ordered lexicographically by those columns in the declared sequence. It supports fast lookups on the leftmost columns and can serve sorting, grouping, and covering for queries whose shape matches its column order.

```sql
CREATE INDEX idx_orders_cust_status_created
    ON orders (customer_id, status, created_at DESC)
    INCLUDE (total_amount);
--         │      │           │            │              │
--         │      │           │            │              └── INCLUDE: payload column stored in
--         │      │           │            │                  the leaf pages ONLY — not a key,
--         │      │           │            │                  not sorted, not seekable; exists so
--         │      │           │            │                  the query can be answered index-only.
--         │      │           │            └── 3rd key column, DESC. Combined with the two equality
--         │      │           │                columns before it, gives newest-first order per
--         │      │           │                (customer_id, status) group for free.
--         │      │           └── 2nd key column. Sorted WITHIN each customer_id value.
--         │      └── 1st (leftmost) key column. The whole index is sorted by this first.
--         │          A query MUST constrain this (or use it in ORDER BY) to seek the tree.
--         └── the table being indexed.
```

Annotated anatomy of every option:

```sql
CREATE [UNIQUE] INDEX [CONCURRENTLY] [IF NOT EXISTS] index_name
    ON table_name [USING btree]      -- btree is the default and the ONLY method that
    │                                --   supports true multi-column key ordering + prefix
    │                                --   seeks. (GIN/GiST/BRIN "multicolumn" behave very
    │                                --   differently — see 6.10.)
    ( col_a [ASC|DESC] [NULLS {FIRST|LAST}],   -- key column 1  (+ per-column sort direction)
      col_b [ASC|DESC] [NULLS {FIRST|LAST}],   -- key column 2
      col_c )                                   -- key column 3
    [INCLUDE (payload_1, payload_2)]  -- non-key columns: stored in leaves for covering,
    │                                 --   never used for ordering or seeking.
    [WHERE predicate];                -- makes it a PARTIAL index (Topic 45) — orthogonal
                                      --   to compositeness; the two combine freely.
```

Key facts baked into the definition:
- **`UNIQUE`** on a composite index enforces uniqueness of the **whole tuple**, not each column. `UNIQUE (a, b)` allows duplicate `a`s and duplicate `b`s, forbidding only duplicate `(a, b)` pairs.
- **Per-column direction** (`ASC`/`DESC`) matters for ORDER BY matching (6.6), and only relative direction matters (an index and its total reversal serve the same ORDER BYs).
- **`INCLUDE` columns are not key columns** — they cannot be used to seek or sort; they exist purely to make an index-only scan possible (6.4, 6.5).

---

## 5. Why Composite Index Mastery Matters in Production

1. **The wrong column order silently costs you the index.** A developer creates `(status, customer_id)` but every query filters by `customer_id` alone. That index is *dead weight* for those queries — it can't seek on `customer_id` because it's the second column. The planner falls back to a Seq Scan or a less-good index, and nobody notices until the table hits 50M rows and the endpoint times out.

2. **Redundant single-column indexes waste write throughput and disk.** Every `INSERT`/`UPDATE`/`DELETE` must maintain *every* index on the table. A team that creates `(a)`, `(a, b)`, and `(a, b, c)` has three indexes where one — `(a, b, c)` — often covers all three read patterns via the leftmost-prefix rule. Those extra indexes are pure write tax and buffer-pool pollution.

3. **Covering indexes turn two I/Os into one.** A hot query that reads three columns can be served entirely from the index (index-only scan) if those columns are in the index or its `INCLUDE` list. This eliminates the heap fetch — often halving latency on a high-QPS endpoint and slashing random I/O.

4. **ORDER BY ... LIMIT without a matching index is a scan-and-sort of the whole table.** The classic "latest 20 orders for this customer" query is instant with `(customer_id, created_at DESC)` and catastrophic without it — because without the index the engine must fetch *all* the customer's orders and sort them just to return 20.

5. **Equality-before-range is the rule people get wrong most.** Putting a range column before an equality column in a composite index truncates the usable prefix at the range column, so the equality column can't be used for seeking. This is the difference between an Index Cond that filters to 12 rows and an Index Cond that filters to 400,000 rows then discards most of them as a Filter.

6. **It's the highest-leverage interview and on-call skill in this phase.** "Here's a slow query and its EXPLAIN — what index would you add?" is *the* canonical senior database interview question, and the answer is almost always a correctly-ordered composite index.

---

## 6. Deep Technical Content

### 6.1 The Leftmost-Prefix Rule (the foundation)

An index on `(a, b, c)` can be used to **seek** (jump to a starting point in the B-tree and scan a contiguous range) for these predicate shapes:

| Query predicates | Usable index prefix | Seek quality |
|------------------|---------------------|--------------|
| `a = ?` | `(a)` | good |
| `a = ? AND b = ?` | `(a, b)` | better |
| `a = ? AND b = ? AND c = ?` | `(a, b, c)` | best (point) |
| `a = ? AND c = ?` | `(a)` only; `c` becomes a filter | partial |
| `b = ?` | **none** — cannot seek | index unusable for seek* |
| `b = ? AND c = ?` | **none** | index unusable for seek* |
| `a = ? AND b > ?` | `(a, b)` as `(a=, b range)` | good |

\* PostgreSQL *may* still choose a full **Index-Only Scan** (read every leaf, filter) if the index is much narrower than the heap and covers the query — but that is a full index scan, not a seek. It is not the prefix rule working; it's a scan-the-whole-index fallback. Do not rely on it.

The rule stated precisely: **an index can seek on a set of columns only if they form a contiguous leftmost prefix of the index's key columns, where every column before the last one used is constrained by equality.** The moment you skip a column or hit a non-equality, the seekable prefix ends.

Why: the B-tree is sorted by `(a, b, c)` lexicographically. Rows with a given `b` value but *different* `a` values are not adjacent — they're spread across every `a` block. There's no single contiguous range to scan for "`b = 5`" without first fixing `a`.

### 6.2 Equality-Before-Range Ordering

Given a query with a mix of equality and range predicates, the optimal composite index puts **all equality columns first, then one range column**. Any columns after the range column cannot be used for seeking.

Consider:

```sql
SELECT * FROM orders
WHERE customer_id = 42          -- equality
  AND status = 'shipped'        -- equality
  AND created_at >= '2026-01-01'; -- range
```

**Optimal index:** `(customer_id, status, created_at)`.

Trace it: fix `customer_id = 42` (seek), within that fix `status = 'shipped'` (seek narrower), within *that* the rows are sorted by `created_at`, so `created_at >= '2026-01-01'` is a contiguous range scan. Every predicate uses the index. The Index Cond consumes all three.

**Suboptimal index:** `(customer_id, created_at, status)`.

Trace it: fix `customer_id = 42` (seek), then `created_at >= '2026-01-01'` is a range — but now the index order is broken for anything after `created_at`. `status = 'shipped'` **cannot** be an index seek; it becomes a Filter applied to every row in the created_at range. If the customer has 100k orders since Jan 1 but only 200 shipped, the index returns 100k rows and filters 99,800 away.

**The rule:** equality columns are "free" to stack because each one keeps you inside a single contiguous block. A range column "spends" the ordering — everything after it is no longer usable for seeking. So: **equalities first (in any order among themselves — see 6.3), then the single most-selective range column last.**

### 6.3 Ordering Among the Equality Columns

If you have multiple equality predicates, their order *among themselves* does not change **whether** the index can be used — any permutation seeks equally well for that exact query, because equalities compose. But it matters for three other reasons:

1. **Serving more query shapes via the prefix rule.** `(customer_id, status)` also serves queries filtering by `customer_id` alone. `(status, customer_id)` also serves queries filtering by `status` alone. Put the column that is queried *alone* more often first, so one index covers more patterns.

2. **Selectivity and index-scan efficiency.** Conventional advice is "most selective column first." For pure equality-on-all-columns this barely affects a B-tree seek (you land on the same tiny range either way). It matters more when only a prefix is constrained: a highly selective leading column makes the *prefix-only* queries fast.

3. **Correlation with range/sort columns.** The leading equality column should be the one you'll also want to combine with a range or ORDER BY, so the whole thing lines up (see 6.6).

Practical heuristic: **lead with the column most often used alone or as the sole equality; keep equalities contiguous; end with the range/sort column.**

### 6.4 Covering Indexes and the Index-Only Scan

An **index-only scan** answers a query entirely from index pages, never fetching from the heap. It's possible when **every column referenced by the query** (in SELECT, WHERE, ORDER BY) is present in the index.

```sql
CREATE INDEX idx_orders_cust_created ON orders (customer_id, created_at);

-- This can be index-only: both selected columns are in the index
SELECT customer_id, created_at
FROM orders
WHERE customer_id = 42
ORDER BY created_at;
```

The plan shows `Index Only Scan`. No heap access for the data — a huge win because the heap fetch is a random I/O per row.

Caveat — **visibility**: even an index-only scan must verify each row is visible to your snapshot (MVCC, Topic 05). PostgreSQL uses the *visibility map*: if the heap page is marked all-visible, it skips the fetch; otherwise it must visit the heap ("Heap Fetches" in EXPLAIN). Recently-written tables have many not-all-visible pages, so run `VACUUM` to keep the visibility map current and keep Heap Fetches near zero.

### 6.5 The INCLUDE Clause — Non-Key Payload Columns

Sometimes you want a column available for *covering* but you don't need to seek or sort by it. Adding it as a key column makes the index larger at every B-tree level and imposes ordering overhead. The `INCLUDE` clause stores the column **only in the leaf pages, as a non-key payload** — available for index-only scans, but not part of the ordered key.

```sql
CREATE INDEX idx_orders_cust_created_inc
    ON orders (customer_id, created_at)   -- key columns: seekable, sorted
    INCLUDE (total_amount, status);       -- payload: covering only, NOT seekable/sortable
```

Now this is index-only:

```sql
SELECT customer_id, created_at, total_amount, status
FROM orders
WHERE customer_id = 42
ORDER BY created_at;
```

`total_amount` and `status` are returned straight from the leaf pages. But you **cannot** seek on them: `WHERE total_amount > 100` still needs a Filter or a different index.

**Key vs INCLUDE — when to use which:**

| Put a column in the KEY when... | Put a column in INCLUDE when... |
|---|---|
| You filter on it with `=` or a range | You only SELECT it (payload) |
| You ORDER BY / GROUP BY it | You never seek or sort by it |
| It participates in the leftmost prefix you need | Adding it to the key would bloat internal pages |
| — | It's a wide column (e.g. text) you want covered without inflating the tree |

Why not just make everything a key column? Because:
- Internal (non-leaf) B-tree pages store key columns to guide navigation; wider keys = fewer entries per page = taller tree = more page reads per seek.
- `INCLUDE` columns live only in leaves, so the navigational part of the tree stays lean.
- A `UNIQUE` index can `INCLUDE` extra columns without them participating in the uniqueness constraint — impossible if they were key columns.

### 6.6 Composite Indexes and ORDER BY (the sort-avoidance win)

A B-tree returns rows in key order. If ORDER BY matches the index's column order **and direction** (for the rows left after the WHERE prefix), the planner skips the Sort node entirely.

```sql
CREATE INDEX idx_orders_cust_created ON orders (customer_id, created_at DESC);

-- Sort-free: WHERE fixes customer_id, index already sorted by created_at DESC within it
SELECT * FROM orders
WHERE customer_id = 42
ORDER BY created_at DESC
LIMIT 20;
-- Plan: Index Scan ... no Sort node ... reads ~20 rows then stops. Instant.
```

**Direction rules.** A B-tree can be walked forwards *or backwards*, so only **relative** directions matter:
- `(customer_id ASC, created_at DESC)` serves `ORDER BY customer_id ASC, created_at DESC` (forward) **and** `ORDER BY customer_id DESC, created_at ASC` (backward).
- It does **not** serve `ORDER BY customer_id ASC, created_at ASC` — the directions don't line up consistently, so a Sort is required.

**Prefix rule for ORDER BY too.** To sort by `(a, b)` using the index, either the index leads with `(a, b)`, or `a` is fixed by an equality in WHERE and the index provides `b`:
- `WHERE a = ? ORDER BY b` → index `(a, b)` is sort-free (a is pinned, b is in order).
- `ORDER BY a, b` with no WHERE → index `(a, b)` is sort-free.
- `WHERE a > ? ORDER BY b` → index `(a, b)` is **not** sort-free — a is a range, so within the range b is not globally ordered. Needs a Sort.

This is the WHERE-vs-ORDER-BY competition from section 3: a *range* on the leading column breaks the ordering of later columns for ORDER BY purposes, just as it breaks seekability.

### 6.7 The WHERE-Range + ORDER BY Conflict

The hardest composite-index design case: an equality, a range, and an ORDER BY on a *different* column.

```sql
SELECT * FROM orders
WHERE customer_id = 42
  AND created_at >= '2026-01-01'   -- range
ORDER BY total_amount DESC          -- sort by a DIFFERENT column
LIMIT 20;
```

You cannot have both the range seek *and* the sort served by one index, because:
- To seek the range, the index needs `(customer_id, created_at)` → but then output is ordered by `created_at`, not `total_amount`. A Sort is required for `total_amount`.
- To avoid the sort, the index needs `(customer_id, total_amount)` → but then `created_at` is a Filter, not a seek. It reads all the customer's rows in total_amount order, filtering by date, stopping after 20 *that pass the filter*.

Which is better depends on selectivity:
- If the date range keeps most rows, `(customer_id, total_amount DESC)` wins — the LIMIT lets it stop early after finding 20 in-range rows in sorted order.
- If the date range is very selective (few rows survive), `(customer_id, created_at)` + a small Sort wins — few rows to sort.

There is no free lunch. Senior engineers decide this by knowing the data distribution, or provide **both** indexes and let the planner pick per parameter value.

### 6.8 Redundant and Overlapping Indexes

The leftmost-prefix rule means `(a, b, c)` already serves queries that would use `(a)` or `(a, b)`. So:

```sql
-- If you have this:
CREATE INDEX idx_abc ON orders (customer_id, status, created_at);

-- These are REDUNDANT (drop them):
CREATE INDEX idx_a  ON orders (customer_id);              -- prefix of idx_abc
CREATE INDEX idx_ab ON orders (customer_id, status);      -- prefix of idx_abc

-- This is NOT redundant (different leading column):
CREATE INDEX idx_status ON orders (status);               -- status can't seek via idx_abc
```

Caveats:
- A shorter index is *physically smaller* and slightly faster/cheaper to maintain. If `(customer_id)` alone is an extremely hot path, a dedicated narrow index can still be justified — but usually the prefix of the composite is fine.
- A `UNIQUE` prefix constraint can't be dropped even if a longer index exists, because the longer index doesn't enforce prefix uniqueness.
- Overlap detection: query `pg_indexes` / `pg_stat_user_indexes` to find unused (`idx_scan = 0`) and redundant indexes.

### 6.9 NULLs in Composite Indexes

- B-tree indexes **do** store NULLs (unlike some other databases). By default NULLs sort **last** (`NULLS LAST` for ASC, `NULLS FIRST` for DESC) — controllable per column.
- `col = NULL` never matches (NULL comparisons are UNKNOWN — Topic 07); use `col IS NULL`, which *can* use the index.
- For a `UNIQUE (a, b)` index, two rows with `(1, NULL)` and `(1, NULL)` are considered **distinct** by default (NULLs aren't equal), so both are allowed — unless you declare `UNIQUE NULLS NOT DISTINCT` (PostgreSQL 15+), which treats NULLs as equal and forbids the duplicate.
- If your ORDER BY specifies `NULLS FIRST`/`LAST` differently from the index, the index order won't match and a Sort is added — match the NULLS placement to get sort-free scans.

### 6.10 Composite Indexes in Other Access Methods (brief)

The leftmost-prefix / equality-first rules are **B-tree** rules. Other multicolumn index types behave differently:
- **GIN** multicolumn: no column order significance — each indexed key maps to matching rows independently (used for arrays/JSONB/full-text). No prefix rule.
- **GiST/SP-GiST**: geometric/range; column order affects the tree but not via lexicographic prefixes.
- **BRIN** multicolumn: stores per-block-range summaries; order is nearly irrelevant, correlation with physical order is what matters.

For 99% of "which columns and in what order" decisions, you mean a **B-tree composite index**, and this whole topic is about B-tree. (Topic 43 covered scan types; Topic 45 covers partial indexes, which combine freely with composite.)

### 6.11 Multicolumn Statistics (helping the planner)

The planner estimates the selectivity of `a = ? AND b = ?` by *multiplying* the per-column selectivities, assuming independence. When `a` and `b` are correlated (e.g. `city` and `postal_code`), this underestimates row counts and can pick a bad plan. Fix with **extended statistics**:

```sql
CREATE STATISTICS orders_cust_status (dependencies, ndistinct)
    ON customer_id, status FROM orders;
ANALYZE orders;
```

This doesn't change the index, but it makes the planner's cost estimates for the composite index far more accurate, so it chooses the right access method.

---

## 7. EXPLAIN — Composite Index Prefixes in the Plan

Table setup for all EXPLAINs below (assume `orders` has 20M rows, `customer_id` has 500k distinct values):

```sql
CREATE INDEX idx_orders_cust_status_created
    ON orders (customer_id, status, created_at);
```

### 7.1 Full-prefix seek (all three columns used)

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, total_amount FROM orders
WHERE customer_id = 42 AND status = 'shipped' AND created_at >= '2026-01-01';
```

```
Index Scan using idx_orders_cust_status_created on orders
    (cost=0.56..38.20 rows=12 width=12)
    (actual time=0.028..0.041 rows=9 loops=1)
  Index Cond: ((customer_id = 42) AND (status = 'shipped'::text)
               AND (created_at >= '2026-01-01'::timestamptz))
  Buffers: shared hit=5
Planning Time: 0.14 ms
Execution Time: 0.058 ms
```

**Reading it:**
- All three predicates are inside **`Index Cond`** — the index seeks on the entire `(customer_id, status, created_at)` prefix. No `Filter` line, meaning nothing is discarded after fetching.
- `Buffers: shared hit=5` — five 8KB pages: a couple of B-tree levels plus a leaf plus the heap fetches. Tiny.
- `rows=9` actual vs `rows=12` estimated — good stats.

### 7.2 Broken prefix — range before equality (the anti-pattern)

Now with the *wrong* index `(customer_id, created_at, status)`:

```sql
CREATE INDEX idx_bad ON orders (customer_id, created_at, status);

EXPLAIN (ANALYZE, BUFFERS)
SELECT id FROM orders
WHERE customer_id = 42 AND created_at >= '2026-01-01' AND status = 'shipped';
```

```
Index Scan using idx_bad on orders
    (cost=0.56..2140.30 rows=12 width=8)
    (actual time=0.040..6.320 rows=9 loops=1)
  Index Cond: ((customer_id = 42) AND (created_at >= '2026-01-01'::timestamptz))
  Filter: (status = 'shipped'::text)
  Rows Removed by Filter: 4821
  Buffers: shared hit=1187
Planning Time: 0.12 ms
Execution Time: 6.351 ms
```

**Reading it — this is the whole topic in one plan:**
- `Index Cond` only has `customer_id` and `created_at`. `status` dropped out of the seek because `created_at` is a **range**, which ends the usable prefix.
- `status = 'shipped'` appears as a **`Filter`**, applied *after* fetching every matching row.
- `Rows Removed by Filter: 4821` — the index returned 4,830 rows, threw away 4,821 to keep 9. Wasted work.
- `Buffers: shared hit=1187` vs 5 in 7.1 — **237× more pages read**. Execution time 6.35ms vs 0.058ms — **~100× slower**, for an identical result.

### 7.3 Index-Only Scan (covering)

```sql
CREATE INDEX idx_cover ON orders (customer_id, created_at) INCLUDE (total_amount);

EXPLAIN (ANALYZE, BUFFERS)
SELECT customer_id, created_at, total_amount FROM orders
WHERE customer_id = 42 ORDER BY created_at;
```

```
Index Only Scan using idx_cover on orders
    (cost=0.56..44.10 rows=210 width=20)
    (actual time=0.021..0.180 rows=204 loops=1)
  Index Cond: (customer_id = 42)
  Heap Fetches: 0
  Buffers: shared hit=6
Planning Time: 0.10 ms
Execution Time: 0.210 ms
```

**Reading it:**
- **`Index Only Scan`** — never touched the heap for data. `total_amount` came from the INCLUDE payload.
- **`Heap Fetches: 0`** — the visibility map said every page was all-visible, so zero heap trips. (If this were >0, `VACUUM` the table.)
- No `Sort` node despite ORDER BY — the index is already in `created_at` order within `customer_id = 42`.

### 7.4 Sort avoided vs Sort required

Sort-free (index order matches ORDER BY):

```
Limit  (actual time=0.02..0.05 rows=20 loops=1)
  ->  Index Scan using idx_orders_cust_created on orders
        Index Cond: (customer_id = 42)
      (reads 20 rows and stops — no Sort node)
```

Sort required (ORDER BY on a non-indexed-order column, per 6.7):

```
Limit  (actual time=48.1..48.1 rows=20 loops=1)
  ->  Sort  (actual time=48.1..48.1 rows=20 loops=1)
        Sort Key: total_amount DESC
        Sort Method: top-N heapsort  Memory: 27kB
        ->  Index Scan using idx_orders_cust_created on orders
              Index Cond: ((customer_id = 42) AND (created_at >= ...))
              (returns 18,400 rows to be sorted for a LIMIT 20)
```

The `Sort` node fetching 18,400 rows to return 20 is the smell that your index order doesn't match your ORDER BY.

### 7.5 What to look for

| Symptom in EXPLAIN | Meaning | Fix |
|---|---|---|
| Predicate in `Filter`, not `Index Cond` | Column past a range/gap in the index — not seekable | Reorder columns: equalities first, range last |
| `Rows Removed by Filter` large | Index returns far more than needed | Add the filtered column to the seekable prefix |
| `Index Scan` not `Index Only Scan` when all cols indexed | Missing column, or column not in INCLUDE | Add column to key/INCLUDE |
| `Heap Fetches` high on Index Only Scan | Stale visibility map | `VACUUM` the table |
| `Sort` node before a LIMIT | ORDER BY not matching index order | Reorder index / add matching index |
| Seq Scan despite a composite index | Leading column not constrained (prefix rule) | Index leading with a queried column |

---

## 8. Query Examples

### Example 1 — Basic: Equality prefix

```sql
-- Index: (customer_id, status)
-- Query fixes both columns with equality — full-prefix seek.
CREATE INDEX idx_orders_cust_status ON orders (customer_id, status);

SELECT id, total_amount, created_at
FROM orders
WHERE customer_id = 42        -- leftmost key, equality  → seek
  AND status = 'shipped';     -- second key, equality    → seek narrower
-- Plan: Index Scan, both predicates in Index Cond, no Filter.
```

### Example 2 — Intermediate: Equality + range + sort-free LIMIT

```sql
-- Index designed for "latest N shipped orders for a customer":
--   equality columns first (customer_id, status), then the sort/range column (created_at DESC).
CREATE INDEX idx_orders_cust_status_created
    ON orders (customer_id, status, created_at DESC);

SELECT id, total_amount, created_at
FROM orders
WHERE customer_id = 42          -- equality  → seek
  AND status = 'shipped'        -- equality  → seek
  AND created_at >= '2026-01-01' -- range on the LAST key → still seekable
ORDER BY created_at DESC         -- matches index direction within the fixed prefix
LIMIT 20;
-- Plan: Index Scan, all predicates in Index Cond, NO Sort node,
--       reads ~20 leaf entries then stops. Sub-millisecond even on 20M rows.
```

### Example 3 — Production Grade: Covering index for a hot dashboard endpoint

```sql
-- CONTEXT:
--   Table: orders, 45M rows, ~120 GB heap.
--   Endpoint: GET /customers/:id/recent-orders  — ~3,000 req/s at peak.
--   It returns the 25 most recent orders per customer with amount + status,
--   and NEVER filters on amount/status (those are display-only).
--
-- GOAL: serve the entire query from the index (index-only scan), zero heap fetches,
--       no sort, stop after 25 rows.
--
-- DESIGN:
--   Key columns:  (customer_id, created_at DESC)   -- seek by customer, pre-sorted newest-first
--   INCLUDE:      (total_amount, status)           -- payload for covering; never seeked/sorted
--
--   Why INCLUDE and not key? total_amount/status are display-only. As key columns they'd
--   bloat internal B-tree pages and buy nothing (we never seek/sort by them). As INCLUDE
--   they ride along in the leaves for free covering.

CREATE INDEX CONCURRENTLY idx_orders_recent_cover
    ON orders (customer_id, created_at DESC)
    INCLUDE (total_amount, status);

SELECT id, created_at, total_amount, status
FROM orders
WHERE customer_id = $1
ORDER BY created_at DESC
LIMIT 25;

-- PERF EXPECTATION: index-only scan, Heap Fetches ~0 (keep autovacuum healthy),
--   ~5-7 buffer hits, < 0.3 ms server-side. Scales flat as the table grows because
--   work is proportional to the 25 rows returned, not the 45M total.
```

EXPLAIN for Example 3:

```
Limit  (cost=0.56..8.40 rows=25 width=28)
       (actual time=0.024..0.061 rows=25 loops=1)
  ->  Index Only Scan using idx_orders_recent_cover on orders
        (cost=0.56..320.10 rows=1020 width=28)
        (actual time=0.022..0.049 rows=25 loops=1)
        Index Cond: (customer_id = 987654)
        Heap Fetches: 0
        Buffers: shared hit=6
Planning Time: 0.09 ms
Execution Time: 0.081 ms
```

- `Index Only Scan` + `Heap Fetches: 0` → the heap (120 GB) is never touched.
- `Limit` stops the scan after 25 leaf entries; the underlying node's `rows=1020` estimate is irrelevant because we stop early.
- Six buffer hits regardless of table size — this is what "scales flat" looks like.

---

## 9. Wrong → Right Patterns

### Wrong 1: Range column before equality column

```sql
-- WRONG: index puts the range column (created_at) in the middle
CREATE INDEX idx_wrong ON orders (customer_id, created_at, status);

SELECT id FROM orders
WHERE customer_id = 42
  AND created_at >= '2026-01-01'   -- range
  AND status = 'shipped';          -- equality, but it comes AFTER the range in the index
```

**What actually happens:** `Index Cond` uses only `(customer_id, created_at)`. `status` degrades to a `Filter`. On a busy customer the index returns thousands of rows and discards nearly all of them (`Rows Removed by Filter: 4821`, 237× more buffers — see EXPLAIN 7.2). The result is correct but ~100× slower.

**Why at the execution level:** the B-tree is sorted lexicographically. Once `created_at` is a range, rows are no longer grouped by `status` within the scanned range, so `status` can't seek — it can only filter post-fetch.

```sql
-- RIGHT: all equalities first, the single range column last
CREATE INDEX idx_right ON orders (customer_id, status, created_at);
-- Now Index Cond = (customer_id = 42 AND status = 'shipped' AND created_at >= ...)
-- Zero Rows Removed by Filter.
```

### Wrong 2: Leading with the wrong (non-queried) column

```sql
-- WRONG: index leads with status, but the app almost always queries by customer_id alone
CREATE INDEX idx_wrong ON orders (status, customer_id);

SELECT id, total_amount FROM orders WHERE customer_id = 42;
```

**What actually happens:** `customer_id` is the *second* column. The leftmost-prefix rule says you can't seek on it without constraining `status` first. The planner ignores this index for the query and does a **Seq Scan** of 20M rows (or picks a worse index), timing out under load.

**Why:** the index is sorted by `status` first; the `customer_id = 42` rows are scattered across every status block — no contiguous range to seek.

```sql
-- RIGHT: lead with the column that's queried alone
CREATE INDEX idx_right ON orders (customer_id, status);
-- Serves WHERE customer_id = 42 (prefix), AND WHERE customer_id = 42 AND status = 'shipped'.
```

### Wrong 3: Believing an ORDER BY is sort-free after a leading range

```sql
CREATE INDEX idx_orders_created_amount ON orders (created_at, total_amount);

-- WRONG expectation: "the index has total_amount, so this is sort-free"
SELECT id FROM orders
WHERE created_at >= '2026-01-01'   -- RANGE on the leading column
ORDER BY total_amount DESC
LIMIT 20;
```

**What actually happens:** a `Sort` node appears (`Sort Key: total_amount DESC`, fetching every in-range row). Because `created_at` is a *range*, the index is sorted by `total_amount` only *within each individual created_at value*, not globally across the range. The engine must fetch all matching rows and sort them.

**Why:** ORDER BY can use index order only when the columns before the sort column are pinned by **equality**, not a range (section 6.6/6.7).

```sql
-- RIGHT (option A): if created_at is really the filter, accept the sort but keep it small
--   by making created_at very selective — few rows to sort.
-- RIGHT (option B): if the pattern is "top by amount within a customer", use an equality lead:
CREATE INDEX idx_orders_cust_amount ON orders (customer_id, total_amount DESC);
SELECT id FROM orders
WHERE customer_id = 42
ORDER BY total_amount DESC
LIMIT 20;   -- now sort-free: customer_id pinned by equality, index in total_amount order
```

### Wrong 4: Stacking redundant prefix indexes

```sql
-- WRONG: three indexes that overlap; the last one already covers the first two
CREATE INDEX idx_a   ON orders (customer_id);
CREATE INDEX idx_ab  ON orders (customer_id, status);
CREATE INDEX idx_abc ON orders (customer_id, status, created_at);
```

**What actually happens:** every write to `orders` now maintains three B-trees instead of one. `idx_a` and `idx_ab` are pure write tax — `idx_abc` already serves `WHERE customer_id = ?` and `WHERE customer_id = ? AND status = ?` via the leftmost-prefix rule. Disk and buffer pool are wasted; `pg_stat_user_indexes` shows `idx_a`/`idx_ab` with `idx_scan = 0`.

```sql
-- RIGHT: keep only the longest; drop the prefixes
DROP INDEX idx_a;
DROP INDEX idx_ab;
-- Keep idx_abc. (Exception: keep a shorter one only if it's UNIQUE-enforcing, or an
--  extremely hot narrow path where the smaller index measurably wins.)
```

### Wrong 5: Adding a filtered column to INCLUDE instead of the key

```sql
-- WRONG: putting a column you FILTER on into INCLUDE
CREATE INDEX idx_wrong ON orders (customer_id) INCLUDE (status, created_at);

SELECT id FROM orders WHERE customer_id = 42 AND status = 'shipped';
```

**What actually happens:** `status` is in the leaves as payload, so the query *can* be index-only, but `status = 'shipped'` is a **Filter**, not an Index Cond — INCLUDE columns are not seekable. The index returns all of customer 42's rows and filters by status. On a customer with 50k orders that's 50k leaf reads to keep 200.

**Why:** INCLUDE columns are unordered payload; only key columns participate in seeking.

```sql
-- RIGHT: status is a filter predicate → it belongs in the KEY, before the range/sort column
CREATE INDEX idx_right ON orders (customer_id, status) INCLUDE (created_at, total_amount);
-- Now status seeks (Index Cond), and created_at/total_amount still cover the SELECT.
```

---

## 10. Performance Profile

### Cost dimensions

| Dimension | Effect of a well-ordered composite index |
|---|---|
| **Read latency** | Seek is O(log N) to the prefix + a contiguous range scan. Matching ORDER BY removes the Sort (O(N log N) → 0). Covering removes the heap random I/O per row. |
| **Write latency** | Every INSERT/UPDATE-of-indexed-column/DELETE maintains the index. A composite index is *one* structure — cheaper than the several single-column indexes it replaces. Wider keys + INCLUDE = larger leaves = slightly more write cost. |
| **Memory / buffer pool** | Narrow, well-chosen indexes stay resident in `shared_buffers`. Redundant indexes evict useful pages. |
| **Disk** | A composite `(a,b,c)` is smaller than three separate `(a)`,`(a,b)`,`(a,b,c)`. INCLUDE columns add leaf size only. |

### Scaling with table size

| Rows | Well-ordered composite (full prefix + covering) | Wrong order (range-before-equality) | No usable index |
|---|---|---|---|
| 1M | < 0.1 ms, ~5 buffers | ~1-3 ms, filters thousands | ~80 ms Seq Scan |
| 10M | < 0.1 ms, ~5-6 buffers | ~5-15 ms | ~800 ms Seq Scan |
| 100M | < 0.2 ms, ~6-7 buffers | ~50-150 ms | ~8 s Seq Scan (or parallel Seq Scan) |

The crucial property: a correct composite index makes latency depend on the **number of rows returned**, not the table size. The wrong order makes it depend on the **number of rows in the seekable prefix's range** (which grows with the table). No index makes it depend on the **whole table**.

### Optimization techniques specific to composite indexes

1. **Order: equalities → range/sort → INCLUDE payload.** The single most impactful rule.
2. **Cover the hot query.** Add the SELECTed-but-not-filtered columns to INCLUDE to reach index-only scans on high-QPS endpoints.
3. **Collapse prefix-redundant indexes** into the longest one to cut write cost.
4. **Match ORDER BY direction** (per-column ASC/DESC) so the Sort node disappears; remember only *relative* direction matters (forward/backward scan).
5. **Keep the visibility map fresh** (`autovacuum`/manual `VACUUM`) so index-only scans have `Heap Fetches: 0`.
6. **Create extended statistics** (`CREATE STATISTICS`) on correlated leading columns so the planner picks the index instead of misestimating and choosing a Seq Scan.
7. **Build with `CONCURRENTLY`** in production to avoid the `ACCESS EXCLUSIVE` lock a plain `CREATE INDEX` takes (it blocks writes for the whole build on a big table).
8. **Watch index bloat**: heavy updates fragment B-trees; `REINDEX CONCURRENTLY` restores density.

### The write/read trade-off, quantified

Each extra index roughly adds one B-tree insert per row insert. On a write-heavy table (say 10k inserts/s), an unnecessary composite index is ~10k extra B-tree maintenance ops/s plus WAL. That is why "one composite that serves many prefixes" beats "many overlapping single-column indexes" not just for reads but decisively for writes.

---

## 11. Node.js Integration

### 11.1 Parameterised query that hits a composite index

```javascript
import pg from 'pg';
const { Pool } = pg;
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

// Backed by: CREATE INDEX ON orders (customer_id, status, created_at DESC);
// Equality, equality, range → full-prefix seek, sort-free, LIMIT stops early.
async function recentShippedOrders(customerId, sinceIso, limit = 20) {
  const { rows } = await pool.query(
    `SELECT id, total_amount, created_at, status
       FROM orders
      WHERE customer_id = $1        -- equality  → index seek
        AND status = 'shipped'      -- equality  → index seek
        AND created_at >= $2        -- range on last key → still seekable
      ORDER BY created_at DESC       -- matches index order → no Sort
      LIMIT $3`,
    [customerId, sinceIso, limit]   // $1,$2,$3 bound safely — never string-concatenated
  );
  return rows;
}
```

The `$1/$2/$3` placeholders matter beyond SQL-injection safety: pg sends them as bind parameters, so PostgreSQL can reuse a cached generic plan for this statement shape and consistently pick the composite index.

### 11.2 Covering / index-only path for a hot endpoint

```javascript
// Backed by: CREATE INDEX ON orders (customer_id, created_at DESC)
//            INCLUDE (total_amount, status);
// SELECT list is a subset of (key cols + INCLUDE cols) → index-only scan possible.
async function customerRecentOrders(customerId, limit = 25) {
  const { rows } = await pool.query(
    `SELECT id, created_at, total_amount, status
       FROM orders
      WHERE customer_id = $1
      ORDER BY created_at DESC
      LIMIT $2`,
    [customerId, limit]
  );
  return rows;
}
// Keep the SELECT list disciplined: adding a column NOT in the index (e.g. shipping_address)
// silently downgrades this from Index Only Scan to Index Scan + per-row heap fetch.
```

### 11.3 Verifying the plan from Node (guardrail in tests)

```javascript
// A regression test that fails if the composite index stops being used.
async function assertIndexOnly(customerId) {
  const { rows } = await pool.query(
    `EXPLAIN (FORMAT JSON)
     SELECT id, created_at, total_amount, status
       FROM orders WHERE customer_id = $1
      ORDER BY created_at DESC LIMIT 25`,
    [customerId]
  );
  const plan = rows[0]['QUERY PLAN'][0].Plan;
  const nodeType = plan.Plans?.[0]?.['Node Type'] ?? plan['Node Type'];
  if (!nodeType.includes('Index Only Scan')) {
    throw new Error(`Expected Index Only Scan, planner chose: ${nodeType}`);
  }
}
```

### 11.4 A note on ORMs and index usage

No ORM "knows" about your composite indexes — indexes are chosen by the *planner* from the SQL text, not by the client. The ORM's only job is to emit SQL whose WHERE/ORDER BY column usage lines up with an index you designed. Two practical consequences:
- If the ORM tacks on an implicit `ORDER BY id` or selects extra columns you didn't ask for, it can defeat sort-free scans or index-only scans. Inspect the generated SQL.
- Composite index *creation* usually lives in a migration (raw DDL), because most ORMs' index DSLs don't express `INCLUDE`, per-column `DESC`, or partial `WHERE`.

---

## 12. ORM Comparison

The question here is twofold: **can the ORM declare a composite index** (with ordering, INCLUDE, direction), and **does the ORM emit query SQL that a composite index can serve**.

### Prisma

**Can it?** Composite indexes: yes, via `@@index([a, b])`. Sort direction per column: yes (`@@index([a, b(sort: Desc)])`). `INCLUDE`/covering: **no native support** — needs raw migration SQL. Partial: no.

```prisma
model Order {
  id          Int      @id @default(autoincrement())
  customerId  Int
  status      String
  createdAt   DateTime
  totalAmount Decimal

  // Composite index, equality columns first, sort column last (DESC)
  @@index([customerId, status, createdAt(sort: Desc)])
}
```

```typescript
// Query that uses the index (Prisma emits WHERE customer_id=.. AND status=.. ORDER BY created_at DESC)
const orders = await prisma.order.findMany({
  where: { customerId: 42, status: 'shipped', createdAt: { gte: new Date('2026-01-01') } },
  orderBy: { createdAt: 'desc' },
  take: 20,
});
```

**Where it breaks:** No `INCLUDE`. To get a covering index you write a raw SQL migration:
```sql
-- prisma/migrations/xxxx_covering/migration.sql
CREATE INDEX idx_orders_cover ON "Order" ("customerId", "createdAt" DESC)
  INCLUDE ("totalAmount", "status");
```
Also watch Prisma adding columns to SELECT you don't need (it selects all scalar fields by default), which can defeat index-only scans — use `select` to narrow.

**Verdict:** Fine for declaring plain composite indexes with direction. Drop to raw SQL for `INCLUDE`/partial. Use `select` to keep covering scans covering.

---

### Drizzle ORM

**Can it?** Composite: yes, `index().on(a, b)`. Direction: yes (`.on(t.a.asc(), t.b.desc())`). `INCLUDE`: yes via `.include()` on newer versions / raw for older. Partial: yes (`.where()`).

```typescript
import { pgTable, serial, integer, text, timestamp, numeric, index } from 'drizzle-orm/pg-core';

export const orders = pgTable('orders', {
  id: serial('id').primaryKey(),
  customerId: integer('customer_id').notNull(),
  status: text('status').notNull(),
  createdAt: timestamp('created_at').notNull(),
  totalAmount: numeric('total_amount').notNull(),
}, (t) => ({
  // equality cols first, sort col DESC last
  custStatusCreated: index('idx_orders_cust_status_created')
    .on(t.customerId, t.status, t.createdAt.desc()),
}));
```

```typescript
import { and, eq, gte, desc } from 'drizzle-orm';
const rows = await db.select().from(orders)
  .where(and(eq(orders.customerId, 42), eq(orders.status, 'shipped'),
             gte(orders.createdAt, new Date('2026-01-01'))))
  .orderBy(desc(orders.createdAt))
  .limit(20);
```

**Where it breaks:** `INCLUDE` support depends on version; if unavailable, use a raw migration. Otherwise Drizzle maps very cleanly to the SQL you intend.

**Verdict:** Best-in-class for expressing composite indexes with direction (and often INCLUDE/partial). Its explicit `.select()` also makes covering scans easy to keep covering.

---

### Sequelize

**Can it?** Composite: yes, `indexes: [{ fields: ['a', 'b'] }]`. Direction: yes (`fields: [{ name: 'created_at', order: 'DESC' }]`). `INCLUDE`: **no**. Partial: yes (`where:`).

```javascript
const Order = sequelize.define('Order', {
  customerId: DataTypes.INTEGER,
  status: DataTypes.STRING,
  createdAt: DataTypes.DATE,
  totalAmount: DataTypes.DECIMAL,
}, {
  indexes: [{
    name: 'idx_orders_cust_status_created',
    fields: ['customerId', 'status', { name: 'createdAt', order: 'DESC' }],
  }],
});
```

**Where it breaks:** No `INCLUDE` — raw migration for covering. Sequelize eager-loading/`include` (association loading) can add JOINs and extra columns that defeat a hand-tuned index; and default `SELECT *`-style attribute lists inflate the row width, blocking index-only scans. Narrow with `attributes: [...]`.

**Verdict:** Adequate for plain composite indexes with direction. Covering needs raw SQL. Audit generated SQL when using `include`.

---

### TypeORM

**Can it?** Composite: yes, `@Index(['a', 'b'])` or entity-level `@Index('name', ['a','b'])`. Direction: limited — column order yes, per-column ASC/DESC is not cleanly expressed in the decorator; use raw migration for `DESC`. `INCLUDE`: no. Partial: yes (`{ where: ... }`).

```typescript
@Entity('orders')
@Index('idx_orders_cust_status_created', ['customerId', 'status', 'createdAt'])
export class Order {
  @PrimaryGeneratedColumn() id: number;
  @Column() customerId: number;
  @Column() status: string;
  @Column() createdAt: Date;
  @Column('numeric') totalAmount: string;
}
```

**Where it breaks:** No `INCLUDE`; per-column `DESC` and covering need a raw migration (`queryRunner.query('CREATE INDEX ... INCLUDE ...')`). The QueryBuilder happily emits usable WHERE/ORDER BY SQL, so query-side is fine.

**Verdict:** Declares composite indexes fine; drop to migration SQL for direction/INCLUDE. Query side is transparent enough to line up with the index.

---

### Knex.js

**Can it?** Composite: yes, `table.index(['a', 'b'])`. Direction/`INCLUDE`: not in the builder — use `knex.raw`. Partial: via raw. Knex is a query builder, so raw DDL is idiomatic anyway.

```javascript
// migration
exports.up = async (knex) => {
  await knex.schema.alterTable('orders', (t) => {
    t.index(['customer_id', 'status', 'created_at'], 'idx_orders_cust_status_created');
  });
  // direction + INCLUDE need raw:
  await knex.raw(`
    CREATE INDEX idx_orders_recent_cover
      ON orders (customer_id, created_at DESC)
      INCLUDE (total_amount, status)
  `);
};
```

```javascript
// query that uses it
const rows = await knex('orders')
  .select('id', 'created_at', 'total_amount', 'status')
  .where({ customer_id: 42 })
  .orderBy('created_at', 'desc')
  .limit(25);
```

**Where it breaks:** The builder can't express `DESC`/`INCLUDE`/partial — but `knex.raw` is first-class and expected, so this is barely a limitation. You control the exact SELECT list, so covering scans are easy to preserve.

**Verdict:** Most SQL-transparent. Use the builder for plain composite indexes, `knex.raw` for direction/INCLUDE/partial. Full control over emitted query SQL.

---

### ORM Summary Table

| ORM | Composite | Per-col DESC | INCLUDE (covering) | Partial | Verdict |
|---|---|---|---|---|---|
| Prisma | `@@index([a,b])` | yes (`sort: Desc`) | raw SQL | no | Good for basics; raw for INCLUDE |
| Drizzle | `.on(a,b)` | yes (`.desc()`) | yes (version-dependent) | yes | Best typed support |
| Sequelize | `indexes:[{fields}]` | yes (`order:'DESC'`) | no | yes | OK; raw for INCLUDE; audit `include` |
| TypeORM | `@Index([...])` | raw for DESC | no | yes | Declares fine; raw for direction/INCLUDE |
| Knex | `.index([...])` | raw | raw | raw | Most transparent; raw is idiomatic |

Universal rule: **no ORM changes which index the planner picks** — it only controls the SQL and the DDL. For `INCLUDE`, per-column direction, and partial predicates, a **raw migration** is the reliable path across all five.

---

## 13. Practice Exercises

### Exercise 1 — Easy

You have:
- `orders(id, customer_id, status, created_at, total_amount)` — 8M rows.

The most common query is:
```sql
SELECT id, total_amount, created_at
FROM orders
WHERE customer_id = $1 AND status = $2
ORDER BY created_at DESC
LIMIT 20;
```

Design the single composite index that makes this query a full-prefix seek **with no Sort node and no heap fetches**. State the exact column order, which columns are keys vs INCLUDE, and any per-column direction.

```sql
-- Write your index here
```

---

### Exercise 2 — Medium (combines Topics 11, 20, 43)

You have:
- `orders(id, customer_id, status, created_at)`
- `order_items(id, order_id, product_id, quantity, unit_price)` — 40M rows.

This query powers a "top products for a customer this year" panel:
```sql
SELECT oi.product_id, SUM(oi.quantity) AS units
FROM orders o
JOIN order_items oi ON oi.order_id = o.id
WHERE o.customer_id = $1
  AND o.created_at >= '2026-01-01'
GROUP BY oi.product_id
ORDER BY units DESC
LIMIT 10;
```

1. Which composite index on `orders` makes the outer filter a full seek?
2. Which composite index on `order_items` supports the join *and* lets the join read be index-only for `(order_id, product_id, quantity)`?
3. Explain (referencing the fan-out from Topic 11) why the `GROUP BY` still needs an aggregate step even with perfect indexes.

```sql
-- Write your indexes and explanation here
```

---

### Exercise 3 — Hard (production simulation; the naive answer is wrong)

A dashboard endpoint runs:
```sql
SELECT id, created_at
FROM orders
WHERE customer_id = $1
  AND created_at >= $2         -- range
ORDER BY total_amount DESC     -- sort by a DIFFERENT column than the range
LIMIT 20;
```

The naive fix is "index `(customer_id, created_at, total_amount)`". Explain precisely why that index still forces a Sort node, using the equality/range/ordering rules. Then propose **two** candidate indexes, describe the exact data distribution under which each one wins, and say how you'd let the planner choose per parameter value.

```sql
-- Write your two candidate indexes and the decision criteria here
```

---

### Exercise 4 — Interview Simulation

You're shown this EXPLAIN for a query that "should be fast":
```
Index Scan using idx_orders_cust_created_status on orders
  Index Cond: ((customer_id = 42) AND (created_at >= '2026-01-01'))
  Filter: (status = 'shipped')
  Rows Removed by Filter: 18904
  Buffers: shared hit=6031
```
The index is `(customer_id, created_at, status)`.

1. Diagnose the problem from the plan in one sentence.
2. Give the corrected index.
3. Predict what `Rows Removed by Filter` and `Buffers` become after the fix, and why.

```sql
-- Write your diagnosis and corrected index here
```

---

## 14. Interview Questions

### Q1 — "Explain the leftmost-prefix rule. Why can't an index on (a, b) help a query that filters only on b?"

**Junior answer:** "Because b is the second column, so the index doesn't work for b." (Correct conclusion, no mechanism.)

**Principal answer:** A B-tree composite index is a single tree whose keys are tuples `(a, b)` compared lexicographically — sorted by `a` first, and by `b` only within a fixed `a`. So all rows sharing a `b` value but differing in `a` are physically scattered across the whole tree; there is no contiguous range to seek for "`b = 5`". You can seek only on a **leftmost prefix** where every column before the last is an equality. `WHERE b = 5` doesn't include the leading column `a`, so the tree offers no starting point — the planner either full-scans the index (if it's narrow and covering) or ignores it for a Seq Scan. This is why "which column leads" is the single biggest composite-index decision.

**Follow-up the interviewer asks:** "So would `WHERE a > 1 AND b = 5` use `(a, b)` for both predicates?" — No. `a > 1` is a range, which ends the seekable prefix; `b = 5` becomes a Filter. Only `a`'s range is the Index Cond. To seek both you'd need `a` to be an equality.

---

### Q2 — "You have a query with two equality predicates and one range. How do you order the composite index, and why does the range go last?"

**Junior answer:** "Put the most selective column first." (A rule of thumb, but it misses the equality/range structure.)

**Principal answer:** Equality columns first (both of them), then the single range column last: `(equality_1, equality_2, range)`. Each equality pins you to one value and keeps the remaining columns contiguously ordered, so they stack losslessly. A range column spans many values, and *across that span the ordering of any subsequent column is broken* — so anything after the range can't seek, it can only filter. Putting the range last means every predicate lands in the Index Cond and nothing is discarded post-fetch. Among the two equalities, order by which is queried alone more often (to serve more prefixes) and by correlation with the sort column. Put a range in the middle and you'll see it in EXPLAIN: the later equality drops to a `Filter` with a big `Rows Removed by Filter`.

**Follow-up:** "What if there are *two* range predicates?" — Only one range can be seekable (it's the terminal column); the second range becomes a Filter. Choose the more selective range for the index position, or consider two indexes / a different structure. This is a genuine limitation of B-tree composite indexes.

---

### Q3 — "What's a covering index, what is INCLUDE for, and when would you put a column in INCLUDE rather than in the key?"

**Junior answer:** "INCLUDE adds more columns to the index so you don't hit the table." (Right effect, no boundary.)

**Principal answer:** A covering index contains every column a query references, so PostgreSQL can serve it with an **index-only scan** — no heap fetch, eliminating a random I/O per row. `INCLUDE` stores columns as **non-key payload in the leaf pages only**: they're available for covering but are *not* part of the ordered key, so they can't be used to seek or sort. Put a column in the **key** if you filter (`=`/range) or sort by it — it must be in the seekable prefix. Put it in **INCLUDE** if you only SELECT it. The reason to prefer INCLUDE for payload: key columns also live in internal B-tree pages and widen them, making the tree taller and every seek more expensive; INCLUDE columns stay out of the navigation layer. Also, a UNIQUE index can INCLUDE columns without them affecting uniqueness. One caveat: index-only scans still consult the visibility map, so keep the table vacuumed or you'll see nonzero `Heap Fetches`.

**Follow-up:** "Your index-only scan shows `Heap Fetches: 40000` — what's wrong?" — The visibility map isn't current (lots of recently-modified, not-all-visible heap pages), so PostgreSQL must visit the heap to check row visibility. Run `VACUUM` (or tune autovacuum); the covering benefit only fully materializes when pages are marked all-visible.

---

### Q4 — "A query does `WHERE customer_id = ? AND created_at >= ? ORDER BY total_amount DESC LIMIT 20`. Design the index."

**Junior answer:** "`(customer_id, created_at, total_amount)` — it has all three columns." (Fails: forces a Sort.)

**Principal answer:** There's an inherent conflict: the range (`created_at`) and the sort (`total_amount`) want different index tails, and one index can't seek the range *and* provide the sort order. Two options: (A) `(customer_id, total_amount DESC)` — sort-free (customer_id pinned by equality, index in total_amount order), with `created_at` as a Filter; the LIMIT lets it stop after finding 20 in-range rows in sorted order — wins when the date filter keeps most rows. (B) `(customer_id, created_at)` — seeks the date range tightly, then a small top-N Sort on total_amount — wins when the date range is very selective (few rows to sort). I'd pick based on the data distribution, or ship both and let the planner choose per parameter value (it will, using stats). The naive `(customer_id, created_at, total_amount)` is the trap: `created_at` being a range means total_amount is ordered only within each date, not across the range, so ORDER BY total_amount still needs a Sort.

**Follow-up:** "How would you confirm which the planner uses?" — `EXPLAIN (ANALYZE, BUFFERS)` with representative parameter values; look for the presence/absence of a `Sort` node and the `Rows Removed by Filter` count. Test both a wide date range and a narrow one, since the winner flips.

---

## 15. Mental Model Checkpoint

1. An index is `(a, b, c)`. For which of these can it *seek*, and on what prefix: `WHERE a=1`, `WHERE b=2`, `WHERE a=1 AND c=3`, `WHERE a=1 AND b>2 AND c=3`? For each, name what ends up as Index Cond vs Filter.

2. You have `(customer_id, status, created_at)`. A colleague adds `(customer_id, status)` "to speed up status filters." Is the new index doing anything the old one couldn't? What is the actual effect on the system?

3. Why does a *range* predicate on a leading column break both seekability *and* ORDER BY usability of the columns after it? Give the single underlying reason.

4. You add `total_amount` to a covering index's `INCLUDE` list and then someone writes `WHERE total_amount > 100`. What does EXPLAIN show, and why isn't it a seek?

5. Two rows have key `(1, NULL)` under a `UNIQUE (a, b)` index. Are both allowed? What one-word change to the constraint flips the answer?

6. A query's SELECT list is exactly the index's columns, yet EXPLAIN shows `Index Scan` (not `Index Only Scan`) with `Heap Fetches` climbing. Nothing is wrong with the index — so what is going on, and what do you run?

7. For `WHERE a=1 ORDER BY b DESC`, index `(a ASC, b DESC)` is sort-free. Name one *other* ORDER BY that the same index serves sort-free, and explain why direction is only "relative."

---

## 16. Quick Reference Card

```sql
-- ────────────────────────────────────────────────────────────────
-- COLUMN ORDER (the whole game):  equalities → ONE range → sort col
--   put INCLUDE payload (SELECT-only cols) last, off the key.
-- ────────────────────────────────────────────────────────────────
CREATE INDEX idx ON t (eq_col_1, eq_col_2, range_or_sort_col DESC)
       INCLUDE (payload_col_1, payload_col_2);

-- LEFTMOST-PREFIX RULE: seek only on a contiguous prefix where every
-- column before the last used is an equality.
--   (a,b,c) serves:  a= ;  a=,b= ;  a=,b=,c= ;  a=,b>  ;  a=,b=,c>
--   (a,b,c) does NOT seek:  b= ;  c= ;  b=,c=   (missing leading a)

-- EQUALITY-BEFORE-RANGE: a range column ends the seekable prefix.
--   Good: (customer_id, status, created_at)   -- eq, eq, range
--   Bad:  (customer_id, created_at, status)   -- status becomes a Filter

-- COVERING / INDEX-ONLY: every referenced column in key or INCLUDE.
--   Key col     → filter/sort/seek on it
--   INCLUDE col → SELECT-only payload; NOT seekable, NOT sortable
--   Keep table VACUUMed → Heap Fetches: 0

-- ORDER BY sort-free when index order+direction match (only RELATIVE
-- direction matters — B-tree scans forward or backward):
--   (a ASC, b DESC) serves  ORDER BY a ASC, b DESC   (forward)
--                    and     ORDER BY a DESC, b ASC   (backward)
--   NOT sort-free after a RANGE on a leading column.

-- REDUNDANCY: (a,b,c) already covers (a) and (a,b) for seeks.
--   Drop prefix-only indexes; keep the longest (+ any UNIQUE prefix).

-- ────────────────  EXPLAIN TELLS ──────────────────
-- predicate in Index Cond   → seeking (good)
-- predicate in Filter       → not seekable (wrong order / INCLUDE / gap)
-- Rows Removed by Filter big → index returns too much; fix column order
-- Sort node before LIMIT     → ORDER BY not matching index order
-- Heap Fetches > 0 (IOS)     → VACUUM to refresh the visibility map

-- ────────────────  PERF RULES OF THUMB ────────────
-- Right composite: latency ∝ rows returned (flat as table grows).
-- Wrong order:     latency ∝ rows in the prefix range (grows).
-- No index:        latency ∝ whole table (Seq Scan).
-- One composite serving many prefixes  >  many single-col indexes
--   (fewer writes, less buffer pollution, less disk).

-- ────────────────  DDL HYGIENE ────────────────────
CREATE INDEX CONCURRENTLY ...   -- no ACCESS EXCLUSIVE lock in prod
CREATE STATISTICS ... ON a, b   -- fix estimates for correlated cols
REINDEX INDEX CONCURRENTLY ...  -- de-bloat after heavy updates

-- ────────────────  INTERVIEW ONE-LINERS ───────────
-- "Composite index = one B-tree of tuples compared lexicographically."
-- "Seek needs a leftmost prefix; equalities before the one range."
-- "A range ends the prefix — everything after it only filters/sorts."
-- "INCLUDE = covering payload in leaves, not seekable, not sorted."
-- "Matching index order kills the Sort; covering kills the heap fetch."
-- "One well-ordered composite beats three overlapping single-col ones."
```

---

## Connected Topics

- **Topic 43 — Index Scan Types** (previous): defines the access methods (Index Scan, Index-Only Scan, Bitmap Index Scan) whose availability composite column order determines.
- **Topic 45 — Partial Indexes** (next): `WHERE`-qualified indexes that combine freely with composite keys/INCLUDE to shrink hot indexes further.
- **Topic 05 — MVCC & Visibility** (internals): why an index-only scan still checks the visibility map and can incur `Heap Fetches`.
- **Topic 07 — NULL in Depth** (internals): three-valued logic behind `IS NULL` seeks, `NULLS FIRST/LAST` ordering, and `NULLS NOT DISTINCT` uniqueness.
- **Topic 11 — INNER JOIN in Depth**: the fan-out and join-key indexing that composite indexes on `(join_key, …)` accelerate; why GROUP BY still aggregates.
- **Topic 20 — GROUP BY Fundamentals**: how a matching index order enables streaming (sorted) aggregates instead of hash aggregates.
- **Topic 42 (Phase 7) — B-tree Structure & The Planner**: the lexicographic tuple ordering and cost model that every rule in this topic derives from.
- **`CREATE STATISTICS` / Extended Statistics**: correcting the planner's independence assumption for correlated leading columns so it actually chooses your composite index.
