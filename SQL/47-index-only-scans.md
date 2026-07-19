# Topic 47 — Index-Only Scans
### SQL Mastery Curriculum — Phase 7: Indexes and Query Optimisation

---

## 1. ELI5 — The Analogy That Makes It Click

Imagine a library with two things:

- **The shelves** — where the actual books physically sit, thousands of them, in no particular order (books get re-shelved wherever there's space when returned).
- **The card catalogue** — a set of small index cards, one per book, sorted alphabetically by title. Each card lists the title, the author, and the exact shelf location.

You want to answer a question: *"What are the titles and authors of every book whose title starts with 'S'?"*

**The slow way (Index Scan + Heap Fetch):** You flip to the 'S' section of the card catalogue, and for each card you find, you walk over to the shelf it points to, pull the physical book, read the title and author off the cover, and walk back. Hundreds of trips across the library floor. The card told you where to go, but you still visited every book.

**The fast way (Index-Only Scan):** You realise the card *already contains* the title and author. You never need to touch the shelves at all. You just read the cards in order and copy off the title and author. Zero trips to the shelves. The entire answer lives in the catalogue.

That is an **index-only scan**: PostgreSQL answers your query *entirely from the index*, never touching the actual table rows (the "heap"). It works only when every column you need is already sitting in the index card.

But there's a catch that the library analogy hides, and it's the single most important thing in this whole topic. In PostgreSQL, the card catalogue doesn't record whether a book has been *checked out and lost* (deleted) or *replaced by a newer edition* (updated). That information — "is this row still valid for you to see?" — lives only on the shelf (in the heap). So even in the fast path, PostgreSQL sometimes has to walk to the shelf just to check *"is this card still pointing at a real, visible book?"*

To avoid those trips, PostgreSQL keeps a separate little cheat-sheet — the **visibility map** — a bitmap that says "every book on shelf 47 is definitely still there and visible to everyone." If the card points at a shelf that's marked all-visible on the cheat-sheet, PostgreSQL trusts it and skips the trip. If not, it walks over anyway (a "heap fetch"). And the thing that keeps that cheat-sheet up to date is `VACUUM`. That is why, uniquely among all index topics, index-only scans depend on VACUUM running. Hold onto that — sections 2 and 6 are built around it.

---

## 2. Connection to SQL Internals

An index-only scan is a physical-plan node (`Index Only Scan`) that the planner chooses when it can prove the query's every referenced column is retrievable from a single index — and when it estimates the visibility-map coverage is high enough that skipping the heap actually pays off. Several internal subsystems collide here:

1. **The B-tree (or other index) structure.** PostgreSQL's default index is a B-tree storing `(indexed_key…, heap_tid)` in each leaf entry. An index-only scan walks the leaf level in key order, reading the indexed columns straight out of the leaf tuples. No separate structure is needed — the data is *in* the index.

2. **The heap and the TID.** The table itself is the "heap" — an unordered pile of 8KB pages. Each index entry carries a `heap_tid` (a `(page, offset)` pointer) to the row's physical location. A normal Index Scan *follows* that pointer to fetch the full row. An index-only scan tries *not* to.

3. **MVCC and visibility.** This is the crux. PostgreSQL is a multi-version store (Topic on MVCC): a single logical row can have multiple physical versions (tuples), each stamped with `xmin`/`xmax` transaction IDs. Whether a given tuple is visible to *your* snapshot is not recorded in the index — only in the heap tuple's header. So the index alone cannot tell you "is this row version visible to me right now?"

4. **The visibility map (VM).** To make index-only scans possible, PostgreSQL maintains a per-table **visibility map**: two bits per heap page. The relevant one is the **all-visible bit** — set when *every* tuple on that page is visible to *every* current and future transaction (no dead tuples, no in-flight changes). If the VM says a page is all-visible, the index-only scan trusts the index entry without visiting the heap. If the bit is clear, it must do a **heap fetch** to check visibility. The VM lives in a separate fork file (`<relfilenode>_vm`), is tiny (1 bit of the pair we care about per 8KB page → ~32KB of VM per 1GB of table), and is usually fully cached.

5. **VACUUM.** The all-visible bit is *set* by `VACUUM` (and autovacuum) after it confirms a page has no dead tuples and no recently-changed tuples. Any `INSERT`/`UPDATE`/`DELETE` to a page *clears* the bit. So a table that is written to constantly but rarely vacuumed will have poor VM coverage → many heap fetches → index-only scans degrade toward regular index scans. **This is the one index feature where VACUUM directly controls query performance.**

6. **The planner cost model.** The planner estimates the fraction of the table's pages that are all-visible (from `pg_class.relallvisible / pg_class.relpages`) and prices the expected heap fetches into the cost. If it estimates too few pages are all-visible, it may not even choose the index-only scan.

7. **INCLUDE columns (covering indexes).** Since PostgreSQL 11, `CREATE INDEX … INCLUDE (cols)` stores extra non-key columns in the B-tree leaf as payload — present for retrieval but not part of the ordered key. This lets you make an index "cover" a query (satisfy an index-only scan) without bloating the searchable key or forcing uniqueness across the extra columns.

The mental model: **an index-only scan is an index scan that is *allowed to skip the heap*, and the visibility map is the thing that grants or denies that permission, page by page, kept current by VACUUM.**

---

## 3. Logical Execution Order Context

Index-only scans are a *physical access-path* decision, not a logical clause. In the logical order —

```
FROM
JOIN
WHERE
GROUP BY
HAVING
SELECT
DISTINCT
window functions
ORDER BY
LIMIT
```

— the index-only scan is how the engine *physically realises* the `FROM` + `WHERE` (and often supplies `ORDER BY` order for free). But the reason it belongs in a query-optimisation phase is that it is uniquely sensitive to **which columns the entire query references**, across *every* clause:

- Columns in `SELECT`
- Columns in `WHERE`
- Columns in `ORDER BY`
- Columns in `GROUP BY`
- Columns in `HAVING`
- Columns in join conditions (for that table)

**Every** column touched anywhere in the query *for that table* must be present in the index for an index-only scan to be possible. Add a single column to your `SELECT` list that isn't in the index and the plan collapses back to a regular Index Scan with a heap fetch on every row. This is why `SELECT *` almost never gets an index-only scan, and why analysts obsess over "covering" exactly the columns a hot query needs.

A subtle ordering consequence: because the B-tree leaf is already sorted on the key columns, an index-only scan can satisfy an `ORDER BY` on the leading index columns *without a Sort node*, and can feed a `LIMIT` that stops early — reading only the first N leaf entries. So an index-only scan often eliminates *both* the heap access *and* a downstream sort. That combined win (no heap, no sort, early stop) is what makes covering indexes so devastatingly effective for top-N and pagination queries — more in Topic 48 and the windowing/pagination topics.

---

## 4. What Is an Index-Only Scan?

An **index-only scan** is a query execution plan in which PostgreSQL retrieves *all* the data a query needs for a table directly from an index, skipping access to the table heap for any heap page marked all-visible in the visibility map. It is possible only when every column the query references for that table is stored in the index (as a key column or an `INCLUDE` payload column).

```sql
-- Create an index that covers a specific query
CREATE INDEX idx_orders_cust_created
  ON orders (customer_id, created_at);
--            │           │
--            │           └── second key column: also stored in the leaf,
--            │               so a query selecting created_at is covered
--            └── leading key column: used for lookups AND stored in the leaf

-- This query can use an INDEX ONLY SCAN:
SELECT customer_id, created_at          -- ← both columns are in the index
FROM orders
WHERE customer_id = 42                    -- ← filter on leading key column
ORDER BY created_at;                      -- ← order supplied by the index for free
```

### Annotated syntax — the covering-index toolkit

```sql
CREATE INDEX idx_name
  ON table_name
     (key_col_a, key_col_b)         -- KEY columns
--    │          │
--    │          └── part of the ordered B-tree key: searchable, sortable,
--    │              AND retrievable in an index-only scan
--    └── leading key column: drives WHERE lookups; leftmost-prefix rule applies
  INCLUDE (payload_col_c, payload_col_d);  -- NON-KEY payload columns (PG 11+)
--         │             │
--         │             └── stored in the leaf tuple ONLY as retrievable payload
--         └── NOT part of the search key: cannot be used to seek or order,
--             but IS available to satisfy an index-only scan
```

```sql
-- Reading an index-only scan in EXPLAIN:
EXPLAIN (ANALYZE, BUFFERS)
SELECT customer_id, created_at FROM orders WHERE customer_id = 42;
```

```
Index Only Scan using idx_orders_cust_created on orders   -- ← the plan node
  Index Cond: (customer_id = 42)          -- ← seek condition on the key
  Heap Fetches: 0                          -- ← THE line that proves it worked
```

- `Index Only Scan` — the node type. Contrast with plain `Index Scan` (always visits heap) and `Bitmap Index Scan` (never index-only).
- `Index Cond` — the searchable predicate satisfied by the B-tree key.
- **`Heap Fetches: 0`** — the single most important diagnostic in this entire topic. It counts how many rows had to fall back to a heap visit because their page was *not* marked all-visible. `0` means the visibility map fully covered the scan — pure index-only. A high number means VACUUM is behind and you're paying heap I/O anyway.

### Precise semantics

An index-only scan returns exactly the same rows and values as the equivalent index scan. It differs *only* in physical access: for each candidate index entry, it consults the visibility map for that entry's heap page.

- If the page's all-visible bit is **set** → return the values straight from the index entry. No heap access.
- If the bit is **clear** → perform a **heap fetch**: read the heap tuple, check MVCC visibility against the snapshot, and (if visible) return the values. Counted in `Heap Fetches`.

The correctness is never in doubt — a cleared bit just means "I'm not sure, go check." The *performance* is what varies, and it varies with VACUUM freshness.

---

## 5. Why Index-Only-Scan Mastery Matters in Production

1. **It is the single biggest lever for read-heavy hot paths.** Turning a query from Index Scan (one random heap I/O per matched row) into Index Only Scan (zero heap I/O) can cut latency by 5–50× on large tables, because it eliminates the random heap page reads that dominate cost. For pagination, dashboards, count-ish aggregates, and existence checks, it's often the difference between 200ms and 3ms.

2. **It is the one query pattern where VACUUM tuning is a *query performance* concern, not just a bloat concern.** Engineers who don't understand the VM dependency ship a covering index, see it fly in staging, then watch it degrade in production under write load — because autovacuum can't keep the visibility map current on a hot table. Diagnosing that requires knowing that `Heap Fetches` climbing is a VACUUM problem, not an index problem.

3. **`SELECT *` silently defeats it.** A huge fraction of "why isn't my covering index being used?" incidents are simply someone selecting a column (or `*`) that isn't in the index. Knowing that *every* referenced column must be covered — across SELECT/WHERE/ORDER BY — prevents wasted index builds.

4. **INCLUDE columns let you cover without penalty.** Before understanding `INCLUDE`, engineers cram extra columns into the *key* to cover a query — bloating the key, slowing inserts, and sometimes breaking a `UNIQUE` constraint's intent. `INCLUDE` is the correct tool, and knowing when to use key vs INCLUDE is a senior-level design skill.

5. **It changes index-design economics.** Covering indexes are larger (they carry payload) and slow writes slightly. Deciding *which* hot queries deserve a covering index — and which should not, because the write cost or index size outweighs the read win — is exactly the trade-off judgement that separates a principal engineer from someone who "adds an index and hopes."

6. **Wrong diagnosis wastes days.** Without this knowledge, a team seeing a slow index scan will try bumping `work_mem`, rewriting the query, or adding *more* indexes — when the real fix is `VACUUM`, an `INCLUDE` clause, or dropping one column from the SELECT list.

---

## 6. Deep Technical Content

### 6.1 The Visibility Map — Structure and Lifecycle

Every table (heap relation) has an auxiliary **visibility map** fork, stored in a file named `<relfilenode>_vm`. It holds **two bits per heap page**:

- **all-visible bit** — set when every tuple on the page is visible to all transactions (no dead tuples, no unresolved recent changes). This is the bit index-only scans consult.
- **all-frozen bit** — set when every tuple is frozen (relevant to freezing/wraparound; not directly used by index-only scans).

Lifecycle of the all-visible bit for a page:

```
INSERT/UPDATE/DELETE on the page  ──►  bit CLEARED (page now "dirty" for visibility)
        │
        ▼
VACUUM scans the page, finds no dead tuples and all tuples old enough
        │
        ▼
bit SET  ──►  index-only scans over rows on this page skip the heap
```

Key properties:

- The VM is **tiny**: ~2 bits per 8KB page ≈ 1 byte per 32KB of table, so roughly **32KB of VM per 1GB of heap**. It is almost always fully resident in shared buffers. Consulting it is nearly free.
- The bit is **conservative**: it is only ever set when the page is provably all-visible, and cleared on *any* modification. There are no false "all-visible" claims — so index-only scans are always correct.
- Because *any* write to a page clears the bit, a page that receives even one `UPDATE` between vacuums forces heap fetches for *all* index-only scans touching *any* row on that page until the next VACUUM re-sets it.

You can inspect VM coverage:

```sql
-- Requires the pg_visibility extension
CREATE EXTENSION IF NOT EXISTS pg_visibility;

-- Fraction of pages marked all-visible for a table
SELECT
  relname,
  relpages,
  relallvisible,
  ROUND(100.0 * relallvisible / NULLIF(relpages, 0), 1) AS pct_all_visible
FROM pg_class
WHERE relname = 'orders';
```

`relallvisible` is the planner's *cached* estimate of all-visible pages (updated by VACUUM/ANALYZE). The planner uses `relallvisible / relpages` as its estimate of the heap-fetch-avoidance fraction.

### 6.2 Why VACUUM Is Load-Bearing (Not Optional) Here

For every other index feature, VACUUM matters only for bloat and wraparound. For index-only scans, **VACUUM directly determines query latency**:

- A freshly loaded, then vacuumed, read-mostly table → VM ~100% all-visible → `Heap Fetches: 0` → full index-only benefit.
- A write-heavy table where autovacuum falls behind → VM coverage drops → `Heap Fetches` climbs → the "index-only" scan pays a random heap I/O for a growing fraction of rows, degrading toward a plain index scan.

Consequences and tuning:

1. **After a bulk load**, the VM is empty (all bits clear) until the first VACUUM. Run `VACUUM (or VACUUM ANALYZE) tablename;` explicitly after large loads if you want index-only scans to work immediately, rather than waiting for autovacuum.

2. **HOT updates help but don't fully save you.** A Heap-Only Tuple (HOT) update — where the changed columns aren't indexed and the new tuple fits on the same page — avoids new index entries, but it *still clears the all-visible bit* on that page until the next vacuum. So even HOT-friendly workloads need vacuuming for index-only performance.

3. **Autovacuum tuning for hot tables** — to keep VM coverage high on a frequently-updated table that serves index-only scans, make autovacuum more aggressive for that table:

```sql
ALTER TABLE orders SET (
  autovacuum_vacuum_scale_factor = 0.02,   -- vacuum after 2% of rows change (default 0.2)
  autovacuum_vacuum_cost_delay   = 2       -- shorter throttling delay
);
```

4. **Read-only / append-only tables are ideal.** Time-series or audit tables that are inserted once and never updated/deleted reach ~100% all-visible after vacuum and stay there — perfect index-only-scan candidates.

### 6.3 The `Heap Fetches` Metric — Reading It Correctly

`Heap Fetches: N` in `EXPLAIN (ANALYZE)` counts how many index entries had to fall back to a heap visit because their page's all-visible bit was clear.

Interpretation guide:

| `Heap Fetches` | Meaning | Action |
|---------------|---------|--------|
| `0` | Perfect: entire scan served from index + VM | None — this is the goal |
| Small, stable | A few recently-modified pages | Usually fine; maybe tune autovacuum slightly |
| High, ≈ rows returned | VM coverage is poor; effectively a normal index scan | VACUUM the table; check autovacuum lag |
| Growing over time | Autovacuum falling behind write load | Make autovacuum more aggressive for this table |

Crucial nuance: a high `Heap Fetches` count means the index-only scan **degraded to roughly an index scan** — it's not *broken*, just not delivering the win. The plan node still says `Index Only Scan`; only the `Heap Fetches` line reveals the truth. **Never evaluate an index-only scan by the node name alone — always read `Heap Fetches`.**

Also note: `Heap Fetches` accumulates across `loops`. If the node runs in a nested loop with `loops=1000`, the reported figure is the total across all iterations.

### 6.4 Which Columns Must Be Covered

For an index-only scan, the index must supply **every column referenced for that table anywhere in the query**:

```sql
CREATE INDEX idx_orders_cust_created_status
  ON orders (customer_id, created_at) INCLUDE (status);

-- COVERED → Index Only Scan possible:
SELECT customer_id, created_at, status         -- all three in the index
FROM orders
WHERE customer_id = 42                          -- key column
ORDER BY created_at;                            -- key column

-- NOT COVERED → falls back to Index Scan + heap fetch:
SELECT customer_id, created_at, status, total_amount  -- total_amount not in index
FROM orders
WHERE customer_id = 42;
```

Column requirements by clause — all count:

- **SELECT list** — obvious.
- **WHERE** — even a column used only for filtering must be in the index; otherwise the heap is needed to evaluate the filter. (A key column used in `Index Cond` is covered; a non-indexed column in a `Filter` forces heap access.)
- **ORDER BY** — must be on covered (ideally leading key) columns.
- **GROUP BY / HAVING / DISTINCT** — every referenced column must be present.
- **Join conditions** — the join key(s) for this table must be covered.

A column used *only* as an equality on the leading key (`WHERE customer_id = 42`) is covered because it's a key column. A column used in a *residual filter* that isn't in the index (`WHERE ... AND total_amount > 100` with `total_amount` not indexed) forces a heap fetch to evaluate the filter — defeating the index-only scan.

### 6.5 KEY Columns vs INCLUDE Columns — The Design Decision

`INCLUDE` (PostgreSQL 11+) stores extra columns in the B-tree leaf as **non-key payload**. They are retrievable (so they help cover a query) but are *not* part of the searchable, ordered key.

```sql
-- Option A: everything as key columns
CREATE INDEX ON orders (customer_id, created_at, status, total_amount);

-- Option B: search/sort columns as key, the rest as INCLUDE payload
CREATE INDEX ON orders (customer_id, created_at) INCLUDE (status, total_amount);
```

When to use **key** columns:
- You need to **search** on the column (`WHERE col = …`, range scans).
- You need to **sort** by the column (`ORDER BY`), exploiting index order.
- The column participates in the **leftmost-prefix** the planner seeks on.

When to use **INCLUDE**:
- You only need the column's **value retrieved**, never searched or sorted on.
- You want to keep the **key narrow** (smaller key = smaller internal pages, faster descents, less write amplification).
- The column has a type that **can't be a B-tree key** but can be payload (e.g. some non-indexable types) — INCLUDE allows carrying it along.
- You are covering a `UNIQUE` index without widening the uniqueness constraint:

```sql
-- Uniqueness is on email ALONE; full_name is just carried for covering
CREATE UNIQUE INDEX idx_users_email
  ON users (email) INCLUDE (full_name, created_at);
-- A lookup by email can now return full_name and created_at index-only,
-- while uniqueness is still enforced on email only (not on the triple).
```

That last case is the killer feature: with pre-11 PostgreSQL you'd have to either add `full_name` to the key (changing the uniqueness semantics — now `(email, full_name)` must be unique, which is wrong) or forgo covering. `INCLUDE` cleanly separates "what's unique/searchable" from "what's retrievable."

Costs of INCLUDE columns:
- They **enlarge the leaf tuples**, so the index is bigger and slightly slower to write and to scan.
- They are stored **only in leaf pages**, not internal pages, so they don't slow tree descent (unlike widening the key).
- Updating an INCLUDE column still requires an index update if the row moves — same as any index maintenance.

### 6.6 Designing a Covering Index Step by Step

Given a hot query, the recipe:

1. **List every column the query references for the target table** (SELECT, WHERE, ORDER BY, GROUP BY, HAVING, joins).
2. **Put search/sort columns in the key**, ordered to match the leftmost-prefix needed by WHERE equality first, then range/ORDER BY column(s).
3. **Put retrieve-only columns in INCLUDE.**
4. **Verify with `EXPLAIN (ANALYZE, BUFFERS)`** that you get `Index Only Scan` with `Heap Fetches` low.
5. **VACUUM the table** (or confirm autovacuum keeps VM coverage high) so the win is real, not just theoretical.

```sql
-- Hot query:
--   SELECT customer_id, status, total_amount
--   FROM orders
--   WHERE customer_id = $1 AND created_at >= $2
--   ORDER BY created_at DESC;

-- Referenced columns: customer_id (WHERE eq), created_at (WHERE range + ORDER BY),
--                      status (SELECT), total_amount (SELECT)

CREATE INDEX idx_orders_cover
  ON orders (customer_id, created_at DESC)      -- equality col, then range/sort col
  INCLUDE (status, total_amount);               -- retrieve-only payload

-- customer_id = $1 seeks; created_at DESC gives range + ordering for free;
-- status/total_amount are read from the leaf → Index Only Scan, no sort, no heap.
```

### 6.7 Index-Only Scans and Aggregates — The `COUNT` Case

A frequent win: `COUNT(*)` and `COUNT(col)` over a filtered range can be served index-only.

```sql
-- With idx_orders_cust_created on (customer_id, created_at):
SELECT COUNT(*)
FROM orders
WHERE customer_id = 42 AND created_at >= '2026-01-01';
```

PostgreSQL can do an `Index Only Scan` over the matching key range and count entries without touching the heap (given good VM coverage). This is why a well-covered index makes range counts dramatically faster than a heap-based count. Note: `COUNT(*)` over an *entire* unfiltered table still has to scan every index entry (it's not free), but it can do so index-only, which is cheaper than a heap seq scan only when the index is much narrower than the table — often it is not, so full-table `COUNT(*)` is not automatically improved. The win is on *filtered* counts and counts over a narrow covering index.

### 6.8 When Index-Only Scans Are NOT Chosen (Even With a Covering Index)

The planner may *decline* an index-only scan even when one is possible:

1. **Poor estimated VM coverage.** If `relallvisible / relpages` is low (stale stats or a write-heavy table), the planner prices in many heap fetches and may prefer a different plan. Fix: VACUUM/ANALYZE.
2. **A referenced column isn't covered.** Any uncovered column → not eligible. Fix: add it (key or INCLUDE).
3. **Expression/function on a column not stored.** If the query needs `lower(email)` but the index stores `email` (not the expression), it's not covered unless you built an *expression index* (Topic 46) storing `lower(email)`. An index-only scan can be served by an expression index if the query references exactly the stored expression.
4. **The index type doesn't support index-only scans for the operator.** B-tree and (with limits) GiST/SP-GiST support them; some access methods or operator classes don't return the original value (e.g. certain GIN cases) and can't serve index-only scans. B-tree is the common, fully-supported case.
5. **Wrong cost estimate / small table.** On a tiny table a seq scan is cheaper; the planner correctly skips the index-only path.
6. **The column is `TOAST`ed and only the key is in the index** — large values stored out-of-line are only reachable via the heap; but if the column is in the index (key or INCLUDE) its value is duplicated there and is available. (INCLUDE of a large value enlarges the index.)

### 6.9 Interaction With Expression Indexes (Topic 46)

An expression index like `CREATE INDEX ON users (lower(email))` stores the *computed* value. A query filtering and selecting exactly that expression can be served index-only:

```sql
CREATE INDEX idx_users_lower_email ON users (lower(email));

-- Index-only if the query needs ONLY lower(email):
SELECT lower(email) FROM users WHERE lower(email) = 'a@b.com';   -- covered

-- NOT index-only — needs the raw email column (not stored as such):
SELECT email FROM users WHERE lower(email) = 'a@b.com';          -- heap needed for email
```

You can combine: `CREATE INDEX ON users (lower(email)) INCLUDE (email, id)` stores both the expression and the raw columns, enabling an index-only scan that filters on `lower(email)` and returns `email, id`.

### 6.10 Partial Indexes + Covering = Very Cheap Hot Paths

Combine a partial index (WHERE clause on the index) with covering columns to make a small, hot index that serves index-only scans over just the relevant subset:

```sql
-- Only "pending" orders are hot; cover the columns the queue query needs
CREATE INDEX idx_orders_pending_cover
  ON orders (created_at)
  INCLUDE (customer_id, total_amount)
  WHERE status = 'pending';

-- Serves this index-only, over a tiny fraction of the table:
SELECT customer_id, created_at, total_amount
FROM orders
WHERE status = 'pending'
ORDER BY created_at;
```

The partial predicate keeps the index tiny (only pending rows), the INCLUDE covers the SELECT, and because pending rows churn but the index is small, autovacuum can keep its pages' VM current more easily.

---

## 7. EXPLAIN — Index-Only Scan in the Plan

### 7.1 The ideal case — Heap Fetches: 0

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT customer_id, created_at
FROM orders
WHERE customer_id = 42
ORDER BY created_at;
```

```
Index Only Scan using idx_orders_cust_created on orders
    (cost=0.43..8.61 rows=42 width=12)
    (actual time=0.018..0.041 rows=39 loops=1)
  Index Cond: (customer_id = 42)
  Heap Fetches: 0
  Buffers: shared hit=5
Planning Time: 0.12 ms
Execution Time: 0.061 ms
```

**Reading it:**
- `Index Only Scan using idx_orders_cust_created` — the covering index is used; the node type confirms the heap-skip path is active.
- `Index Cond: (customer_id = 42)` — the seek on the leading key column.
- **`Heap Fetches: 0`** — no heap visits at all; the visibility map fully covered the scan. This is the success signal.
- `Buffers: shared hit=5` — only 5 buffer hits, all index (+VM) pages. No random heap page reads. Compare to the same query as a plain index scan, which would add one heap buffer access per matched row.
- No `Sort` node — the B-tree supplied `ORDER BY created_at` order for free (created_at is the second key column, and customer_id is fixed by the equality).

### 7.2 The degraded case — VACUUM behind, Heap Fetches high

Same query, but the table has had heavy recent writes and autovacuum is behind:

```
Index Only Scan using idx_orders_cust_created on orders
    (cost=0.43..12.40 rows=42 width=12)
    (actual time=0.030..0.512 rows=39 loops=1)
  Index Cond: (customer_id = 42)
  Heap Fetches: 37
  Buffers: shared hit=5 read=34
Planning Time: 0.14 ms
Execution Time: 0.71 ms
```

**Reading it:**
- Same node type (`Index Only Scan`) — the plan *looks* identical.
- **`Heap Fetches: 37`** — of 39 rows, 37 fell back to a heap visit because their pages weren't all-visible. This is effectively an index scan now.
- `Buffers: ... read=34` — 34 heap pages had to be read from disk. This is the cost the VM was supposed to eliminate.
- Execution time is ~12× the ideal case. **The fix is `VACUUM orders;`**, not a query rewrite. After vacuuming, `Heap Fetches` drops back toward 0.

### 7.3 Why an index-only scan was NOT chosen

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT customer_id, created_at, total_amount   -- total_amount not in the index
FROM orders
WHERE customer_id = 42;
```

```
Index Scan using idx_orders_cust_created on orders
    (cost=0.43..48.20 rows=42 width=20)
    (actual time=0.021..0.180 rows=39 loops=1)
  Index Cond: (customer_id = 42)
  Buffers: shared hit=44
Planning Time: 0.11 ms
Execution Time: 0.21 ms
```

**Reading it:**
- Plain `Index Scan`, **no `Heap Fetches` line at all** — a plain index scan *always* visits the heap, so the metric is meaningless and omitted. Its presence/absence alone tells you which node you have.
- `Buffers: shared hit=44` — up from 5: the extra ~39 buffer touches are the per-row heap visits to fetch `total_amount`.
- Fix: add `total_amount` to the index as an `INCLUDE` column, and the plan becomes an `Index Only Scan`.

### 7.4 Index-only aggregate

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT COUNT(*)
FROM orders
WHERE customer_id = 42 AND created_at >= '2026-01-01';
```

```
Aggregate  (cost=8.72..8.73 rows=1 width=8) (actual time=0.055..0.056 rows=1 loops=1)
  ->  Index Only Scan using idx_orders_cust_created on orders
        (cost=0.43..8.64 rows=31 width=0)
        (actual time=0.020..0.048 rows=31 loops=1)
        Index Cond: ((customer_id = 42) AND (created_at >= '2026-01-01'::date))
        Heap Fetches: 0
        Buffers: shared hit=4
Execution Time: 0.079 ms
```

**Reading it:**
- `width=0` on the scan — `COUNT(*)` needs no column values, only the count of matching index entries.
- `Heap Fetches: 0` — the count is computed purely from index entries over the all-visible range.
- The `Aggregate` node sits above, tallying the rows the index-only scan streams up.

### 7.5 What to look for

| Symptom in EXPLAIN | Meaning | Fix |
|--------------------|---------|-----|
| `Index Only Scan` + `Heap Fetches: 0` | Ideal — pure index-only | None |
| `Index Only Scan` + high `Heap Fetches` | VM coverage poor (VACUUM behind) | `VACUUM` / tune autovacuum |
| `Index Scan` (no Heap Fetches line) when you expected index-only | A referenced column isn't covered | Add column via `INCLUDE` (or key) |
| `Seq Scan` when you expected index-only | Small table, or planner cost/stat issue | Check size; `ANALYZE`; verify index |
| `Sort` node above an index-only scan | ORDER BY not on leading key columns | Reorder index keys to match ORDER BY |

---

## 8. Query Examples

### Example 1 — Basic: a two-column covering index

```sql
-- Index covers exactly the two columns this lookup needs.
CREATE INDEX idx_sessions_user_created
  ON sessions (user_id, created_at);

-- Query references ONLY user_id and created_at → Index Only Scan.
SELECT
  user_id,
  created_at
FROM sessions
WHERE user_id = 1001            -- seek on leading key column
ORDER BY created_at DESC        -- ordering supplied by the index (second key col)
LIMIT 20;                       -- early stop: reads only the first 20 leaf entries
-- Expect: Index Only Scan, Heap Fetches: 0 (on a vacuumed table), no Sort node.
```

### Example 2 — Intermediate: adding INCLUDE to cover a SELECT column

```sql
-- We search/sort on (customer_id, created_at) but also need to RETURN status
-- and total_amount. Those go in INCLUDE — retrieved but not searched/sorted.
CREATE INDEX idx_orders_cover
  ON orders (customer_id, created_at DESC)
  INCLUDE (status, total_amount);

SELECT
  customer_id,
  created_at,
  status,            -- from INCLUDE payload
  total_amount       -- from INCLUDE payload
FROM orders
WHERE customer_id = 42
  AND created_at >= CURRENT_DATE - INTERVAL '30 days'
ORDER BY created_at DESC;
-- All four columns come from the index leaf → Index Only Scan, no heap, no sort.
-- Contrast: without INCLUDE (status, total_amount), this is a plain Index Scan
-- with one heap fetch per row.
```

### Example 3 — Production Grade: covering index for a high-traffic pagination endpoint

**Scenario:** An "activity feed" endpoint returns a user's most recent orders, 25 per page, showing status and amount. The `orders` table has **60 million rows**; a given user has up to a few thousand orders. This endpoint is called ~2,000×/sec. Without a covering index it does a random heap read per returned row; the goal is `Heap Fetches: 0` and sub-millisecond latency.

```sql
-- Covering + partial: only non-cancelled orders are shown in the feed.
CREATE INDEX idx_orders_feed
  ON orders (customer_id, created_at DESC)
  INCLUDE (status, total_amount, currency)
  WHERE status <> 'cancelled';

-- Keep the visibility map current so index-only scans truly skip the heap:
ALTER TABLE orders SET (
  autovacuum_vacuum_scale_factor = 0.02,
  autovacuum_vacuum_cost_delay   = 2
);

-- The endpoint query (keyset pagination — see pagination topic):
SELECT
  customer_id,
  created_at,
  status,
  total_amount,
  currency
FROM orders
WHERE customer_id = $1
  AND status <> 'cancelled'
  AND (created_at, id) < ($2, $3)   -- keyset cursor from previous page
ORDER BY created_at DESC, id DESC
LIMIT 25;
```

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT customer_id, created_at, status, total_amount, currency
FROM orders
WHERE customer_id = 77 AND status <> 'cancelled'
ORDER BY created_at DESC
LIMIT 25;
```

```
Limit  (cost=0.56..3.14 rows=25 width=32) (actual time=0.020..0.038 rows=25 loops=1)
  ->  Index Only Scan using idx_orders_feed on orders
        (cost=0.56..214.90 rows=2083 width=32)
        (actual time=0.019..0.033 rows=25 loops=1)
        Index Cond: (customer_id = 77)
        Heap Fetches: 0
        Buffers: shared hit=6
Planning Time: 0.16 ms
Execution Time: 0.052 ms
```

**Why it performs:**
- `LIMIT 25` + index order → the scan stops after 25 leaf entries (`actual rows=25`, not 2083). No sort of the user's full history.
- `Heap Fetches: 0` → zero random heap reads, the whole point at 2,000 req/s.
- `Buffers: shared hit=6` → six cached index/VM pages per request. This is what sub-millisecond at scale looks like.
- The partial predicate (`status <> 'cancelled'`) keeps the index small and matches the query's filter, and the `INCLUDE` columns cover the SELECT list.
- The `ALTER TABLE … autovacuum` settings are not decoration: at this write rate, default autovacuum would let VM coverage decay and `Heap Fetches` would creep up, silently reintroducing heap I/O. Aggressive autovacuum keeps `Heap Fetches` at 0.

---

## 9. Wrong → Right Patterns

### Wrong 1: `SELECT *` (or one stray column) silently defeats the covering index

```sql
-- WRONG: the covering index has (customer_id, created_at) INCLUDE (status),
-- but SELECT * pulls total_amount, currency, shipping_address, … — none covered.
SELECT *
FROM orders
WHERE customer_id = 42
ORDER BY created_at DESC
LIMIT 25;
-- RESULT: plan is a plain Index Scan (NOT Index Only Scan). One random heap
-- fetch per returned row. At scale this is the difference between 3ms and 200ms.
-- The index build was wasted for THIS query — the planner can't skip the heap
-- because * references uncovered columns.
```

**Why it's wrong at the execution level:** an index-only scan requires *every* referenced column to live in the index. `*` expands to all table columns; the moment one isn't in the index leaf, the engine must visit the heap for each row to fetch it, and the plan degrades to `Index Scan`.

```sql
-- RIGHT: select only the covered columns (and/or extend the index to cover the rest).
SELECT customer_id, created_at, status          -- all covered by the index
FROM orders
WHERE customer_id = 42
ORDER BY created_at DESC
LIMIT 25;
-- Now: Index Only Scan, Heap Fetches: 0.
-- If you genuinely need total_amount too, add it: ... INCLUDE (status, total_amount).
```

### Wrong 2: Building a covering index but never checking `Heap Fetches` under load

```sql
-- WRONG assumption: "the plan says Index Only Scan, so we're done."
EXPLAIN (ANALYZE)
SELECT customer_id, created_at FROM orders WHERE customer_id = 42;
-- Index Only Scan ...
--   Heap Fetches: 4120        ← IGNORED by the developer
-- The node name is 'Index Only Scan', but 4120 heap fetches means it's paying
-- random heap I/O for nearly every row. This table is written to constantly and
-- autovacuum is far behind, so the visibility map is mostly clear.
```

**Why it's wrong at the execution level:** the plan-node *name* does not tell you whether the heap was actually skipped. Only `Heap Fetches` does. A high count means the visibility map didn't cover the pages, so the "index-only" scan behaved like a regular index scan — the entire performance premise was false.

```sql
-- RIGHT: measure Heap Fetches, and if high, fix the VACUUM/VM situation.
VACUUM (ANALYZE) orders;   -- rebuilds VM coverage, updates relallvisible

ALTER TABLE orders SET (
  autovacuum_vacuum_scale_factor = 0.02   -- keep VM current under write load
);

-- Re-check:
EXPLAIN (ANALYZE)
SELECT customer_id, created_at FROM orders WHERE customer_id = 42;
-- Index Only Scan ...
--   Heap Fetches: 0          ← now the win is real
```

### Wrong 3: Widening the KEY to cover, when INCLUDE was the right tool

```sql
-- WRONG: you want a UNIQUE index on email that also returns full_name index-only,
-- so you add full_name to the key:
CREATE UNIQUE INDEX idx_users_email ON users (email, full_name);
-- BUG: uniqueness is now enforced on the PAIR (email, full_name), not email alone.
-- Two rows with the same email but different full_name are now both allowed —
-- the uniqueness guarantee you actually wanted is broken.
```

**Why it's wrong:** key columns define the sort order *and* (for a unique index) the uniqueness constraint. Adding `full_name` to the key changed the semantic meaning of the constraint. It also enlarged internal pages, slowing tree descent.

```sql
-- RIGHT: keep email as the sole key; carry full_name as INCLUDE payload.
CREATE UNIQUE INDEX idx_users_email ON users (email) INCLUDE (full_name);
-- Uniqueness is on email ALONE (correct). full_name is retrievable for an
-- index-only scan of `SELECT email, full_name FROM users WHERE email = $1`.
```

### Wrong 4: Expecting index-only after selecting a column the expression index doesn't store

```sql
CREATE INDEX idx_users_lower_email ON users (lower(email));

-- WRONG: the index stores lower(email), not the raw email column.
SELECT email                                   -- raw email is NOT in the index
FROM users
WHERE lower(email) = 'a@b.com';
-- RESULT: Index Scan (heap fetch per row) to retrieve raw email. Not index-only.
```

**Why it's wrong:** the expression index stores the *computed* value `lower(email)`, not the original `email`. Selecting `email` forces a heap visit to read the stored column.

```sql
-- RIGHT (option A): select the stored expression, if that satisfies the caller.
SELECT lower(email) FROM users WHERE lower(email) = 'a@b.com';   -- index-only

-- RIGHT (option B): store the raw column too, via INCLUDE.
CREATE INDEX idx_users_lower_email ON users (lower(email)) INCLUDE (email, id);
SELECT email, id FROM users WHERE lower(email) = 'a@b.com';      -- now index-only
```

---

## 10. Performance Profile

### 10.1 Where the win comes from

A plain index scan does, per matched row: one B-tree descent (amortised), then **one heap page access** to fetch the row — and on a large table that heap page is very often a *random*, uncached read. Random heap I/O is the dominant cost. An index-only scan eliminates that heap access entirely (given VM coverage), leaving only the sequential-ish walk of index leaf pages plus near-free VM bit checks.

### 10.2 Scaling behaviour

| Table size | Plain Index Scan (100 matched rows) | Index Only Scan, VM ~100% | Index Only Scan, VM poor |
|-----------|-------------------------------------|---------------------------|--------------------------|
| 1M rows | ~100 heap page reads (many cached) | 0 heap reads | up to ~100 heap reads |
| 10M rows | ~100 heap reads, more misses | 0 heap reads | ~100 heap reads |
| 100M rows | ~100 heap reads, mostly cold random I/O — dominant cost | 0 heap reads — index+VM pages, mostly cached | ~100 heap reads — degrades to index-scan cost |

The key insight: **as the table grows, the heap becomes colder and more random, so the *relative* win of skipping it grows.** On a 100M-row table where the heap doesn't fit in cache, index-only scans can be 10–50× faster than the equivalent index scan — but *only* if the visibility map is current. If VM coverage is poor, the index-only scan collapses to the same cost as an index scan.

### 10.3 Memory and CPU

- **Memory:** index-only scans need no extra `work_mem` (no hash table, no sort buffer when order comes from the index). The VM fork is tiny (~32KB/GB) and effectively always resident.
- **CPU:** slightly higher per index entry if INCLUDE payload is wide (more bytes to copy from the leaf), but this is trivial next to the eliminated random I/O.
- **Buffer cache:** far kinder — you touch index + VM pages (small, hot) instead of scattered heap pages, improving cache hit ratios for the *whole* system, not just this query.

### 10.4 Write-side cost (the trade-off)

Covering indexes are not free:
- **Larger index** — INCLUDE payload and extra key columns enlarge every leaf tuple, so the index consumes more disk and cache.
- **Slower writes** — every `INSERT`, and any `UPDATE` that changes an indexed *or included* column, must maintain the wider index. More/larger indexes = more WAL, more write amplification.
- **HOT-update inhibition** — if you `INCLUDE` a column that gets updated frequently, updates to it can no longer be HOT (heap-only) with respect to that index, increasing index churn and *reducing* VM coverage — a double loss. **Do not `INCLUDE` hot, frequently-updated columns.**

### 10.5 Optimisation techniques specific to index-only scans

1. **Keep VM coverage high** — aggressive per-table autovacuum on hot tables; explicit `VACUUM` after bulk loads.
2. **Cover exactly what the hot query needs** — no more (write cost), no less (defeats index-only). Audit the query's real column set.
3. **Prefer partial indexes** to shrink the covering index to the hot subset, which also makes VM maintenance cheaper.
4. **Order key columns** as `equality columns → range/sort column` so the scan both seeks tightly and returns rows in the needed order (no Sort node).
5. **Watch `Heap Fetches` in production**, not just staging — the VM behaves differently under real write load.
6. **Reindex if the covering index bloats** (`REINDEX CONCURRENTLY`) — a bloated index has more pages to walk, eroding the win.

---

## 11. Node.js Integration

### 11.1 Basic index-only-served query with `pg`

```javascript
import pg from 'pg';
const { Pool } = pg;
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

// Selects only covered columns → served by an Index Only Scan.
async function getRecentOrderHeaders(customerId, limit = 25) {
  const { rows } = await pool.query(
    `SELECT customer_id, created_at, status, total_amount
     FROM orders
     WHERE customer_id = $1
       AND status <> 'cancelled'
     ORDER BY created_at DESC
     LIMIT $2`,
    [customerId, limit]
  );
  return rows;
}
// Index: (customer_id, created_at DESC) INCLUDE (status, total_amount)
//        WHERE status <> 'cancelled'.
// Keep the SELECT list == covered columns. Adding a non-covered column here
// silently turns this back into a heap-fetching Index Scan.
```

### 11.2 Verifying index-only behaviour from Node (a health check)

```javascript
// Programmatically assert Heap Fetches stays low — catch VACUUM regressions in CI/monitoring.
async function checkHeapFetches(customerId) {
  const { rows } = await pool.query(
    `EXPLAIN (ANALYZE, FORMAT JSON)
     SELECT customer_id, created_at, status, total_amount
     FROM orders
     WHERE customer_id = $1
     ORDER BY created_at DESC
     LIMIT 25`,
    [customerId]
  );
  const plan = rows[0]['QUERY PLAN'][0].Plan;
  // Walk to the Index Only Scan node (here it's under a Limit node):
  const scan = plan.Plans?.[0] ?? plan;
  return {
    nodeType: scan['Node Type'],          // expect 'Index Only Scan'
    heapFetches: scan['Heap Fetches'],    // expect 0 (or small)
  };
}
// If heapFetches climbs in production, alert: the table needs VACUUM /
// autovacuum tuning, NOT a query rewrite.
```

### 11.3 Explicit maintenance after a bulk load

```javascript
// After a large import, the visibility map is empty until VACUUM runs.
// Vacuum explicitly so index-only scans work immediately instead of waiting for autovacuum.
async function bulkLoadThenEnableIndexOnly(client, rowsToInsert) {
  await client.query('BEGIN');
  // ... COPY / batched INSERT of rowsToInsert ...
  await client.query('COMMIT');

  // VACUUM cannot run inside a transaction block — run it standalone.
  await client.query('VACUUM (ANALYZE) orders');
  // Now relallvisible is populated and index-only scans hit Heap Fetches: 0.
}
```

**Do ORMs handle this?** No ORM decides index-only scans — that's the planner's job. What the ORM *controls* is the **SELECT list**, which is exactly what makes or breaks index-only eligibility. The danger: many ORMs default to selecting *all* columns of an entity (equivalent to `SELECT *`), which defeats covering indexes. To get index-only scans through an ORM you must explicitly project only the covered columns (see Section 12). The index itself — including `INCLUDE` — is defined in a migration, and most ORMs need raw SQL for `INCLUDE`.

---

## 12. ORM Comparison

The recurring theme: an ORM cannot *force* an index-only scan (the planner does), but it can *enable* or *sabotage* one by controlling the projected column list and by whether it can express `INCLUDE`.

### Prisma

**Can it?** Partially. Prisma's `select` lets you project a subset of columns (enabling index-only if that subset is covered), but Prisma **cannot define `INCLUDE` indexes** in its schema — you need a raw migration.

```typescript
import { PrismaClient } from '@prisma/client';
const prisma = new PrismaClient();

// Project only covered columns → gives the planner a shot at an Index Only Scan.
const rows = await prisma.order.findMany({
  where: { customerId: 42, status: { not: 'cancelled' } },
  select: { customerId: true, createdAt: true, status: true, totalAmount: true },
  orderBy: { createdAt: 'desc' },
  take: 25,
});
```

```prisma
// schema.prisma — Prisma supports composite @@index but NOT INCLUDE columns.
// @@index([customerId, createdAt])   // key-only; can't express INCLUDE here
```

```sql
-- Define the covering index via a raw migration (prisma migrate + custom SQL):
CREATE INDEX idx_orders_cover
  ON orders (customer_id, created_at DESC)
  INCLUDE (status, total_amount);
```

**Where it breaks:** default `findMany` without `select` fetches all scalar fields (`SELECT *` semantics) → defeats index-only. And Prisma's schema DSL can't express `INCLUDE`. **Verdict:** usable if you always specify `select` and add the `INCLUDE` index by raw SQL migration.

### Drizzle ORM

**Can it?** Yes — best of the group. Explicit column selection is idiomatic, and Drizzle can define `INCLUDE` indexes in the schema.

```typescript
import { db } from './db';
import { orders } from './schema';
import { and, eq, ne, desc } from 'drizzle-orm';

const rows = await db
  .select({
    customerId: orders.customerId,
    createdAt: orders.createdAt,
    status: orders.status,
    totalAmount: orders.totalAmount,
  })                                     // explicit projection = covered columns
  .from(orders)
  .where(and(eq(orders.customerId, 42), ne(orders.status, 'cancelled')))
  .orderBy(desc(orders.createdAt))
  .limit(25);
```

```typescript
// schema.ts — Drizzle supports .include() on index definitions:
index('idx_orders_cover')
  .on(orders.customerId, desc(orders.createdAt))
  .include(orders.status, orders.totalAmount);
```

**Where it breaks:** essentially nothing for this use case — you must still remember not to over-select. **Verdict:** best index-only support; explicit projection is the default idiom and `INCLUDE` is first-class.

### Sequelize

**Can it?** Yes for projection (`attributes`), no for `INCLUDE` in schema.

```javascript
const rows = await Order.findAll({
  where: { customerId: 42, status: { [Op.ne]: 'cancelled' } },
  attributes: ['customerId', 'createdAt', 'status', 'totalAmount'], // covered subset
  order: [['createdAt', 'DESC']],
  limit: 25,
});
```

**Where it breaks:** default `findAll` selects every attribute (`SELECT *`) → no index-only. `INCLUDE` indexes must be created in a raw migration (`queryInterface.sequelize.query('CREATE INDEX ... INCLUDE ...')`). Note the naming clash: Sequelize's `include` option means *eager-loading joins*, unrelated to SQL `INCLUDE`. **Verdict:** always set `attributes`; add covering indexes by raw migration.

### TypeORM

**Can it?** Yes for projection (`select` in QueryBuilder or find options); `INCLUDE` needs raw SQL.

```typescript
const rows = await dataSource
  .getRepository(Order)
  .createQueryBuilder('o')
  .select(['o.customerId', 'o.createdAt', 'o.status', 'o.totalAmount']) // covered
  .where('o.customerId = :cid', { cid: 42 })
  .andWhere("o.status <> 'cancelled'")
  .orderBy('o.createdAt', 'DESC')
  .limit(25)
  .getMany();
```

**Where it breaks:** `getMany()` without an explicit `select` hydrates full entities (`SELECT *`). The `@Index` decorator can't express `INCLUDE`; use a raw migration. **Verdict:** use QueryBuilder with explicit `select`; define the `INCLUDE` index in a migration `query(...)`.

### Knex.js

**Can it?** Yes — the most transparent. You write the column list; `INCLUDE` goes in `schema.raw`.

```javascript
const rows = await knex('orders')
  .select('customer_id', 'created_at', 'status', 'total_amount')  // covered subset
  .where({ customer_id: 42 })
  .andWhereNot('status', 'cancelled')
  .orderBy('created_at', 'desc')
  .limit(25);

// Covering index via raw DDL in a migration:
await knex.schema.raw(`
  CREATE INDEX idx_orders_cover
    ON orders (customer_id, created_at DESC)
    INCLUDE (status, total_amount)
`);
```

**Where it breaks:** `select('*')` or omitting `.select()` (defaults to `*`) defeats index-only. Knex's builder has no first-class `INCLUDE`, but `schema.raw` is clean. **Verdict:** most SQL-transparent; you control exactly the projection and DDL.

### ORM Summary Table

| ORM | Explicit projection | `INCLUDE` in schema | Default select | Verdict |
|-----|--------------------|--------------------|----------------|---------|
| Prisma | `select: {...}` | No — raw migration | All fields (defeats) | OK; always `select`, raw INCLUDE |
| Drizzle | `.select({...})` | Yes — `.include()` | Explicit | Best support |
| Sequelize | `attributes: [...]` | No — raw migration | All attrs (defeats) | Always set `attributes` |
| TypeORM | `.select([...])` | No — raw migration | Full entity (defeats) | QueryBuilder + `select` |
| Knex | `.select(cols)` | No — `schema.raw` | `*` (defeats) | Most transparent |

**The universal rule:** whichever ORM, the failure mode is the same — the default "give me the whole entity" projection acts like `SELECT *` and defeats the covering index. You must explicitly project only the covered columns, and (for all but Drizzle) define `INCLUDE` via raw SQL.

---

## 13. Practice Exercises

### Exercise 1 — Easy

You have:
- `sessions(id, user_id, created_at, ip_address, user_agent)`

A hot endpoint runs:

```sql
SELECT user_id, created_at
FROM sessions
WHERE user_id = $1
ORDER BY created_at DESC
LIMIT 50;
```

1. Write the `CREATE INDEX` that lets this run as an `Index Only Scan` with no Sort node.
2. What single line in `EXPLAIN (ANALYZE)` confirms the heap was fully skipped?

```sql
-- Write your index here
```

### Exercise 2 — Medium (combines prior topics)

You have:
- `orders(id, customer_id, created_at, status, total_amount, currency, shipping_address)`

An analytics query (uses the range/ORDER-BY reasoning from Topic 46 expression indexes and the covering ideas here):

```sql
SELECT customer_id, created_at, status, total_amount
FROM orders
WHERE customer_id = $1
  AND created_at >= $2
ORDER BY created_at DESC;
```

1. Design a covering index using the correct split of KEY vs INCLUDE columns. Justify why each column is where it is.
2. `total_amount` is updated occasionally when an order is amended. What is the downside of putting it in `INCLUDE`, and would you still do it here?
3. Write a second index that would serve `SELECT COUNT(*) FROM orders WHERE customer_id = $1 AND created_at >= $2` index-only, and explain whether your first index already handles it.

```sql
-- Write your indexes and reasoning here
```

### Exercise 3 — Hard (production simulation, naive answer is wrong)

An engineer ships this covering index and endpoint on a 40M-row, write-heavy `orders` table:

```sql
CREATE INDEX idx_orders_cover
  ON orders (customer_id, created_at DESC)
  INCLUDE (status, total_amount);

SELECT customer_id, created_at, status, total_amount
FROM orders WHERE customer_id = $1 ORDER BY created_at DESC LIMIT 25;
```

In staging (freshly loaded, then vacuumed) it shows `Index Only Scan, Heap Fetches: 0`, sub-millisecond. In production the same query averages 40ms and `EXPLAIN (ANALYZE)` shows `Index Only Scan` but `Heap Fetches: 24` (out of 25 rows).

1. Explain precisely why staging and production differ, at the visibility-map level.
2. The naive fix is "rewrite the query" or "add work_mem." Why are both wrong?
3. Give the correct fix(es), with the exact SQL, and explain how each restores `Heap Fetches` toward 0.
4. `total_amount` turns out to be updated on ~15% of orders shortly after creation. How does that interact with your fix, and would you reconsider the index design?

```sql
-- Write your diagnosis and fix here
```

### Exercise 4 — Interview Simulation

> "We have a `users(email UNIQUE, full_name, created_at, last_login_at, ...)` table with 20M rows. The login flow does `SELECT id, full_name FROM users WHERE email = $1`. It's slow under load. Design the index so this is an index-only scan, keep email uniqueness intact, and explain what could still make it slow in production and how you'd detect it."

Write the index and a short spoken answer covering: KEY vs INCLUDE, why not to widen the key, the visibility-map/VACUUM caveat, and the exact `EXPLAIN` line you'd monitor.

```sql
-- Write your index and answer here
```

---

## 14. Interview Questions

### Question 1

**Q: What is an index-only scan, and what has to be true for PostgreSQL to use one?**

**Junior answer:** "It's when the query uses an index instead of scanning the table. It's faster because indexes are sorted."

**Principal answer:** "An index-only scan is an index scan that is *permitted to skip the heap* — it returns the query's data straight from the index leaf without following the TID to the table row. Two things must hold. First, *coverage*: every column the query references for that table — across SELECT, WHERE, ORDER BY, GROUP BY, joins — must be present in the index, either as a key column or an `INCLUDE` payload column. Second, *visibility*: because the index doesn't store MVCC visibility information, PostgreSQL must confirm each row is visible. It does that via the visibility map — a per-page bitmap where an all-visible page means 'every tuple here is visible to everyone.' For index entries on all-visible pages, it trusts the index and skips the heap; for pages whose bit is clear, it does a *heap fetch* to check. So an index-only scan is only fully effective when the visibility map is well-maintained, which is VACUUM's job."

**Follow-up:** *"You add a covering index and the plan says `Index Only Scan`, but it's not faster. What do you check first?"* → The `Heap Fetches` line. A high count means the visibility map isn't covering the pages, so it's paying heap I/O despite the node name — a VACUUM/autovacuum problem, not an index or query problem.

### Question 2

**Q: Why does VACUUM affect the performance of index-only scans, when for every other index it only matters for bloat?**

**Junior answer:** "VACUUM cleans up dead rows so the table is smaller and queries are faster."

**Principal answer:** "Index-only scans depend on the visibility map's all-visible bits. That bit is *set* by VACUUM after it verifies a page has no dead tuples and nothing recently changed, and it's *cleared* by any INSERT/UPDATE/DELETE touching the page. So the VM is only as current as your last VACUUM of those pages. On a write-heavy table where autovacuum falls behind, VM coverage decays, `Heap Fetches` climbs, and the index-only scan degrades toward a plain index scan — one random heap read per row again. Uniquely, this makes VACUUM a *query-latency* concern here, not just a space concern. The mitigations are per-table aggressive autovacuum settings, explicit VACUUM after bulk loads, and — as a design choice — favouring append-only/read-mostly tables and partial indexes so the hot page set stays small and easy to keep all-visible."

**Follow-up:** *"You bulk-load 50M rows and immediately query with a covering index — index-only or not?"* → Not index-only in effect: right after a load the VM is empty (all bits clear) until the first VACUUM, so every row is a heap fetch. Run `VACUUM (ANALYZE)` after the load to populate the VM and `relallvisible`.

### Question 3

**Q: When would you put a column in `INCLUDE` versus in the index key? Give a case where the difference is semantically important, not just performance.**

**Junior answer:** "INCLUDE is just for adding more columns to the index so it covers more queries."

**Principal answer:** "Key columns are searchable, sortable, and — for a unique index — define the uniqueness constraint and the tree's internal-page structure. INCLUDE columns are non-key payload stored only in the leaf: retrievable for covering, but not searchable, not sortable, and not part of uniqueness. Use key columns for anything in WHERE seeks or ORDER BY; use INCLUDE for retrieve-only columns you want available for an index-only scan without paying to widen the key. The semantically critical case is a unique index: `CREATE UNIQUE INDEX ON users(email) INCLUDE (full_name)` keeps uniqueness on *email alone* while still letting `SELECT email, full_name WHERE email = $1` run index-only. If you instead put `full_name` in the key — `UNIQUE(email, full_name)` — you've silently changed the constraint to allow duplicate emails with different names. That's a correctness bug, not a performance nuance. INCLUDE also keeps internal pages narrow, so tree descent stays fast, and lets you carry columns of types that can't be B-tree keys."

**Follow-up:** *"Any column you'd refuse to put in INCLUDE?"* → A frequently-updated column. Including it means updates to it must maintain the index (and can inhibit HOT updates), increasing churn and clearing visibility-map bits more often — which hurts the very index-only scans the covering index was built for.

---

## 15. Mental Model Checkpoint

1. A query runs as `Index Only Scan` but `Heap Fetches` equals the number of rows returned. Is the index broken? What is actually happening, and what one command likely fixes it?

2. You `SELECT customer_id, created_at, total_amount` and your index is `(customer_id, created_at)`. Why does the plan show `Index Scan` rather than `Index Only Scan`, and what are two different ways to make it index-only?

3. Explain, in terms of MVCC and the visibility map, *why* an index alone cannot decide whether a row is visible to your transaction — even though the index entry clearly exists.

4. You bulk-load a 100M-row read-only table and immediately benchmark a covering-index query. It's slower than expected with high `Heap Fetches`. Nothing is being written. Why, and what did you forget to do?

5. Why can putting a hot, frequently-updated column into an index's `INCLUDE` list *reduce* the effectiveness of index-only scans on that same index? Trace the chain of cause and effect.

6. For a `UNIQUE(email)` index, contrast the constraint semantics of `INCLUDE (full_name)` versus adding `full_name` as a second key column. Why is this a correctness issue and not just performance?

7. A full-table `SELECT COUNT(*)` over a 100M-row table can use an index-only scan, yet it may be no faster than a sequential scan. Explain when the index-only count actually wins and when it doesn't.

---

## 16. Quick Reference Card

```sql
-- Covering index: KEY columns (search/sort) + INCLUDE (retrieve-only payload)
CREATE INDEX idx ON t (key_a, key_b) INCLUDE (payload_c, payload_d);
--                     └ searchable/sortable      └ retrievable only (PG 11+)

-- Unique + covering WITHOUT widening the constraint:
CREATE UNIQUE INDEX ON users (email) INCLUDE (full_name);

-- Partial + covering (small, hot index):
CREATE INDEX ON orders (created_at) INCLUDE (customer_id, total_amount)
  WHERE status = 'pending';

-- Confirm index-only in EXPLAIN — READ THE HEAP FETCHES LINE:
EXPLAIN (ANALYZE, BUFFERS) SELECT ...;
--   Index Only Scan using idx on t
--     Heap Fetches: 0        ← 0 = pure index-only; high = VACUUM behind

-- Keep the visibility map current (this is what makes it fast):
VACUUM (ANALYZE) t;                       -- after bulk loads, or to refresh VM
ALTER TABLE t SET (                       -- for hot, write-heavy tables:
  autovacuum_vacuum_scale_factor = 0.02,
  autovacuum_vacuum_cost_delay   = 2
);

-- Inspect VM coverage:
SELECT relname, relpages, relallvisible,
       ROUND(100.0*relallvisible/NULLIF(relpages,0),1) AS pct_visible
FROM pg_class WHERE relname = 't';
```

**Rules of thumb**
- Index-only requires **every referenced column** (SELECT + WHERE + ORDER BY + …) to be in the index.
- `SELECT *` (or one stray column) **defeats** it → plain Index Scan + heap fetch per row.
- **`Heap Fetches: 0`** is the goal; the node name `Index Only Scan` alone proves nothing.
- High/rising `Heap Fetches` → **VACUUM problem**, not a query problem.
- After a **bulk load**, VACUUM before expecting index-only (VM starts empty).
- Put **search/sort** columns in the **key**, **retrieve-only** columns in **INCLUDE**.
- Never `INCLUDE` a **frequently-updated** column (churn + HOT inhibition + VM clearing).

**Interview one-liners**
- "An index-only scan is an index scan *allowed to skip the heap*; the visibility map grants that permission page by page, and VACUUM keeps it current."
- "The plan node name lies — only `Heap Fetches` tells you if the heap was actually skipped."
- "`INCLUDE` separates *what's unique/searchable* from *what's retrievable* — that's why unique covering indexes are correct."
- "Rising `Heap Fetches` is the one query symptom whose fix is `VACUUM`."

---

## Connected Topics

- **MVCC and Tuple Visibility** (internals): the multi-version model and `xmin`/`xmax` visibility checks that the index cannot see — the entire reason the visibility map exists.
- **VACUUM and Autovacuum** (internals): sets the all-visible bits that index-only scans depend on; the one maintenance process that is also a query-performance lever here.
- **The Visibility Map & Heap Page Structure** (internals): the `_vm` fork, all-visible/all-frozen bits, and how any write clears them.
- **Topic 44 — B-tree Index Internals**: leaf tuples storing `(key…, tid)` and INCLUDE payload; why key width affects tree descent but INCLUDE (leaf-only) does not.
- **Topic 45 — Composite & Multicolumn Indexes**: leftmost-prefix rule and key ordering (equality-then-range) that also governs covering-index key design.
- **Topic 46 — Expression Indexes** (previous): storing computed values; an expression index can serve an index-only scan when the query references exactly the stored expression, and can be combined with `INCLUDE`.
- **Topic 48 — When Indexes Hurt** (next): the write-side cost, index bloat, and HOT-update inhibition that covering/INCLUDE indexes introduce — the other side of this trade-off.
- **Partial Indexes**: shrinking a covering index to the hot subset, which also keeps visibility-map maintenance cheap.
- **Keyset Pagination**: the canonical beneficiary — covering index + `LIMIT` + index order yields no heap, no sort, early stop.


