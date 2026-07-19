# Topic 37 — FIRST_VALUE, LAST_VALUE, NTH_VALUE
### SQL Mastery Curriculum — Phase 6: Window Functions — Complete Mastery

---

## 1. ELI5 — The Analogy That Makes It Click

Imagine a relay race. Each runner is standing in a line, ordered by their leg number. You are a coach standing next to one runner, and you want to answer three questions about the team from where you stand:

- **Who is the first runner in this relay team?** — That's `FIRST_VALUE`.
- **Who is the last runner in this relay team?** — That's `LAST_VALUE`.
- **Who is the third runner?** — That's `NTH_VALUE(runner, 3)`.

Now here is the twist that trips up almost everyone, and it is the whole reason this topic exists.

You are standing next to a specific runner and you can only *see* the runners up to and including yourself — the ones behind you are around a bend you cannot see past yet. So if I ask you "who is the last runner?" while you are standing next to runner #2, you honestly answer "runner #2, because that's the last one I can see from here." You are not wrong — you answered "the last runner *in your field of view*," and your field of view stops at you.

That is exactly what `LAST_VALUE` does by default in SQL. The window's default "field of view" (the *frame*) stretches from the very first row up to **the current row** — not to the end of the partition. So `LAST_VALUE` looking from row 2 sees only rows 1 and 2, and returns row 2. It returns "the current row" on every single row, which is almost never what anyone wants.

To let each runner see the *entire* team all the way to the finish line, you have to explicitly widen the field of view: `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`. Now everyone can see every runner, and "who is the last runner?" gets the same correct answer no matter which runner you ask.

`FIRST_VALUE` gets away with the default because the first row is always behind you (or is you) — it is always inside the default frame. `LAST_VALUE` and `NTH_VALUE` looking toward the end do not get away with it. Remember that asymmetry and you have understood 80% of this topic.

---

## 2. Connection to SQL Internals

`FIRST_VALUE`, `LAST_VALUE`, and `NTH_VALUE` are **window functions** implemented by PostgreSQL's `WindowAgg` executor node. Understanding what that node does mechanically is what separates people who memorize the `UNBOUNDED FOLLOWING` incantation from people who know *why* it is needed.

**The WindowAgg node and the tuplestore.** When the planner sees a window function, it inserts a `WindowAgg` node above a `Sort` node (the sort orders rows by `PARTITION BY` keys, then `ORDER BY` keys). `WindowAgg` reads the sorted stream and materializes rows into a **tuplestore** — an in-memory (spilling to disk past `work_mem`) buffer that allows random access to rows within the current partition. This random-access buffer is what makes it possible to look *forward* to a row that has not been emitted yet. Aggregate functions like `SUM` can often be computed with a running state and no lookback; value functions like `LAST_VALUE` fundamentally need to peek at other physical rows, so the tuplestore is mandatory.

**Frames define the peek boundaries.** Every window function evaluates against a **frame** — a subset of the current partition's rows, defined relative to the current row. `FIRST_VALUE`, `LAST_VALUE`, and `NTH_VALUE` do not scan the whole partition; they scan the **current frame**. The frame boundaries (`frameOptions` in the plan) tell `WindowAgg` which tuplestore slots are legal to read. This is the single most important internal fact of this topic: **these three functions read the frame, not the partition.** The default frame is `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`, so by default they cannot see past the current row toward the end.

**RANGE vs ROWS and peer groups.** The default frame uses `RANGE` mode, whose `CURRENT ROW` boundary means "the last row of the current *peer group*" — all rows that tie on the `ORDER BY` key. `ROWS` mode means "this exact physical row." This distinction (covered in depth in Topic 39) changes what `LAST_VALUE` returns when there are ties: under `RANGE` it returns the last peer of the current row's tie group; under `ROWS` it returns the exact current row.

**No B-tree magic here.** Unlike joins (Topic 11) or ordinary predicates, window functions do not use indexes to *evaluate* the function. An index matters only insofar as it can supply the required sort order for free, eliminating the `Sort` node beneath `WindowAgg`. The function evaluation itself is a linear pass over the sorted tuplestore with random-access reads bounded by the frame.

---

## 3. Logical Execution Order Context

```
FROM / JOIN          ← tables assembled
WHERE                ← row filtering (window funcs NOT allowed here)
GROUP BY             ← grouping
HAVING               ← group filtering
SELECT               ← window functions evaluated HERE, in the window phase
   └── window phase  ← FIRST_VALUE / LAST_VALUE / NTH_VALUE run now
DISTINCT
ORDER BY             ← final output ordering (separate from window ORDER BY)
LIMIT / OFFSET
```

Window functions execute in the **SELECT phase**, *after* `FROM`, `WHERE`, `GROUP BY`, and `HAVING`, but *before* the final `DISTINCT`, top-level `ORDER BY`, and `LIMIT`. Several consequences follow directly and are perennial interview material:

1. **You cannot reference a window function in `WHERE` or `HAVING`.** Those clauses run before the window phase. `WHERE first_value(...) OVER (...) = x` is a syntax error. To filter on a window result you must wrap the query in a subquery or CTE and filter in the outer query — the window function computes in the inner query, the filter runs after. This is identical to the LAG/LEAD constraint from Topic 36.

2. **`WHERE` shrinks the input to the window.** Because `WHERE` runs first, any rows filtered out are simply not present when the frame is built. `FIRST_VALUE` returns the first row *that survived the WHERE clause*, not the first row in the raw table. This is a feature — filter, then rank within survivors — but it surprises people who expect "first ever."

3. **The window's own `ORDER BY` is independent of the query's `ORDER BY`.** The `ORDER BY` inside `OVER (...)` defines the sequence within the frame and therefore which row is "first," "last," or "nth." The top-level `ORDER BY` only arranges the final printed output. They can differ, and often should. If you omit the top-level `ORDER BY`, the printed row order is undefined even though the window computation was correct.

4. **`GROUP BY` collapses before windows run.** If you both `GROUP BY` and use a window function, the window sees one row per group. `FIRST_VALUE` over grouped rows means "first group," not "first raw row." Topic 38 explores aggregate-plus-window interaction.

---

## 4. What Are FIRST_VALUE, LAST_VALUE, and NTH_VALUE?

These three are **value window functions**: they return the value of a chosen expression evaluated at a specific row *within the current window frame*, without collapsing rows. Every input row produces one output row (unlike aggregates), and each output carries a value pulled from the first, last, or nth row of that row's frame.

```sql
FIRST_VALUE(expression) OVER (
  PARTITION BY partition_expr    -- │ optional: splits rows into independent groups
  ORDER BY sort_expr [ASC|DESC]  -- │ defines row sequence → who is "first"/"last"/"nth"
  frame_clause                   -- │ ROWS|RANGE|GROUPS BETWEEN ... AND ... (the peek boundary)
)

LAST_VALUE(expression) OVER (
  PARTITION BY partition_expr    -- │ optional grouping
  ORDER BY sort_expr             -- │ sequence definition
  ROWS BETWEEN UNBOUNDED PRECEDING
           AND UNBOUNDED FOLLOWING -- │ ⚠ REQUIRED to see the true last row, not current row
)

NTH_VALUE(expression, n) OVER (  -- │ n is an integer literal or expression ≥ 1 (1-based)
  PARTITION BY partition_expr
  ORDER BY sort_expr
  frame_clause
)
[FROM FIRST | FROM LAST]         -- │ count n from the frame start (default) or the frame end
[RESPECT NULLS | IGNORE NULLS]   -- │ whether NULL values of `expression` are skipped
```

Annotated breakdown of the parts that matter:

```
FIRST_VALUE(salary) OVER (
  PARTITION BY department_id   -- │ each department is an independent window; first per dept
                               -- └── omit → the whole result set is one window
  ORDER BY hire_date           -- │ "first" = earliest hire_date within the department
                               -- └── omit → no order → "first" is arbitrary/unstable
  ROWS BETWEEN UNBOUNDED PRECEDING
           AND CURRENT ROW      -- │ this is the DEFAULT frame if you write nothing
                               -- └── fine for FIRST_VALUE, WRONG for LAST_VALUE
)
```

**Precise definition.** Given the current row's frame (an ordered list of rows), `FIRST_VALUE(e)` returns `e` evaluated at the frame's first row, `LAST_VALUE(e)` at the frame's last row, and `NTH_VALUE(e, n)` at the frame's nth row (1-based), or `NULL` if the frame has fewer than `n` rows. With `IGNORE NULLS`, rows where `e IS NULL` are excluded from the counting before first/last/nth is chosen.

**Default frame (memorize this).** When you write an `ORDER BY` in the `OVER` clause but no explicit frame, PostgreSQL applies:

```
RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
```

When you write **no** `ORDER BY` at all, the default frame is the entire partition (`RANGE BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`), because with no ordering every row is a peer of every other. This is a subtle escape hatch discussed in Section 6.

---

## 5. Why Mastery Matters in Production

1. **The LAST_VALUE trap ships silently to production.** `LAST_VALUE(x) OVER (ORDER BY t)` runs without error, returns a plausible-looking column, and is *wrong on every row except the last*. It returns the current row's value, not the partition's final value. No exception, no warning — just a dashboard that shows "latest status = current row's status" and a report that quietly disagrees with reality. This is the single most common window-function bug in real codebases.

2. **"Latest value per group" is a daily requirement.** Getting the most recent status per order, the opening balance per account, the first touch and last touch in an attribution model, the entry and exit price per trading session — all of these are `FIRST_VALUE`/`LAST_VALUE` patterns. Doing them wrong corrupts financial and analytical outputs in ways that are hard to notice until an auditor does.

3. **Frame mistakes cause quadratic performance.** A carelessly framed `LAST_VALUE` with `RANGE` and a volatile order key, or an `NTH_VALUE` re-scanning a large frame per row, can turn an O(n) window into an O(n²) executor pass that melts under 10M rows. Knowing which frame you actually need keeps it linear.

4. **`IGNORE NULLS` is the clean way to do "last non-null / carry-forward."** Filling gaps (LOCF — last observation carried forward) is a real analytics need. `LAST_VALUE(x) IGNORE NULLS` with the right frame does it in one pass; the naive alternative is a correlated subquery that is slower and harder to read.

5. **These functions are *not* fully expressible in every ORM.** Prisma cannot express them at all without raw SQL, and the frame clause is where every query builder's abstraction leaks. A backend engineer who reaches for the raw escape hatch confidently, with the correct frame, avoids a class of subtle data bugs the ORM cannot catch.

---

## 6. Deep Technical Content

### 6.1 The Default Frame — The Root of Everything

Every window function with an `ORDER BY` and no explicit frame gets:

```sql
RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
```

Read literally: "from the first row of the partition, up to and including the current row (and its peers)." The frame *grows* as you move down the partition. On row 1 the frame is `{row1}`. On row 2 it is `{row1, row2}`. On the last row it is the whole partition.

Now apply each function to this growing frame:

| Row | Frame (default) | `FIRST_VALUE` sees | `LAST_VALUE` sees |
|-----|-----------------|--------------------|-------------------|
| 1 | {1} | row 1 | row 1 |
| 2 | {1,2} | row 1 | row 2 |
| 3 | {1,2,3} | row 1 | row 3 |
| 4 | {1,2,3,4} | row 1 | row 4 |

`FIRST_VALUE` returns row 1 on every row — correct and stable, because row 1 is always in the frame. `LAST_VALUE` returns the *current* row on every row — because the current row is always the last row of the growing default frame. **That is the trap.** `LAST_VALUE` under the default frame is a verbose, expensive way to write the current row's own value.

### 6.2 Fixing LAST_VALUE — Widen the Frame

To make `LAST_VALUE` return the actual last row of the partition, you must extend the frame's upper bound to the end:

```sql
LAST_VALUE(status) OVER (
  PARTITION BY order_id
  ORDER BY changed_at
  ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
)
```

Now the frame is the whole partition on every row, so `LAST_VALUE` returns the final row's `status` for all rows. Three equivalent ways to express "the whole partition" frame:

```sql
-- Explicit, most common:
ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING

-- RANGE variant (identical result when there are no ties on the order key,
-- but see 6.4 for the tie subtlety):
RANGE BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING

-- GROUPS variant (rarely needed here):
GROUPS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
```

**The clean alternative to LAST_VALUE.** Very often the elegant fix is to *not use LAST_VALUE at all*. Reverse the order and use `FIRST_VALUE`, which needs no frame gymnastics:

```sql
-- "Last status" via FIRST_VALUE on reversed order — no frame clause needed:
FIRST_VALUE(status) OVER (
  PARTITION BY order_id
  ORDER BY changed_at DESC     -- descending → the newest row is now "first"
)
```

This is idiomatic and many senior engineers prefer it precisely because it sidesteps the frame trap. The only caveat: reversing the order also reverses `NULLS FIRST/LAST` placement — be explicit with `ORDER BY changed_at DESC NULLS LAST` if NULL timestamps exist.

### 6.3 FIRST_VALUE Is Safe by Default (But Not Always)

`FIRST_VALUE` gives the correct partition-first value under the default frame, because the first row is always inside the frame. But it becomes *incorrect* if you widen or shift the frame's lower bound:

```sql
-- Correct: first row of the partition (default lower bound = UNBOUNDED PRECEDING)
FIRST_VALUE(x) OVER (PARTITION BY g ORDER BY t)

-- SUBTLE: with a sliding lower bound, "first" moves with the current row
FIRST_VALUE(x) OVER (PARTITION BY g ORDER BY t
                     ROWS BETWEEN 2 PRECEDING AND CURRENT ROW)
-- Now "first" = the row two positions before the current row (or partition start
-- if fewer than 2 exist) — a moving reference, not the partition's first row.
```

So `FIRST_VALUE` is only "safe" because the *default* lower bound happens to be `UNBOUNDED PRECEDING`. The lesson generalizes: these functions read *the frame*, and the frame is whatever you (explicitly or by default) set.

### 6.4 RANGE vs ROWS — The Tie / Peer-Group Subtlety

The default frame uses `RANGE`, whose `CURRENT ROW` bound means "through the last **peer** of the current row," where peers are rows with equal `ORDER BY` values. This changes `LAST_VALUE` behavior when ties exist even before you touch `UNBOUNDED FOLLOWING`.

Consider rows ordered by `day`, with two rows sharing `day = '2026-01-02'`:

```
row  day          amount
 1   2026-01-01     10
 2   2026-01-02     20   ← peer group
 3   2026-01-02     30   ← peer group
 4   2026-01-03     40
```

```sql
LAST_VALUE(amount) OVER (ORDER BY day)   -- default RANGE ... CURRENT ROW
```

On row 2, the frame under `RANGE` includes *all peers of row 2*, i.e. rows 1, 2, and 3 (because row 3 ties on `day`). So `LAST_VALUE` on row 2 returns **30**, not 20 — surprising if you assumed the frame stopped at the physical current row. Under `ROWS`:

```sql
LAST_VALUE(amount) OVER (ORDER BY day
                         ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
```

On row 2 the frame is exactly rows 1–2, so `LAST_VALUE` returns **20**. `ROWS` counts physical rows; `RANGE` counts value-peers. When you widen to `UNBOUNDED FOLLOWING`, `ROWS` and `RANGE` converge (both cover the whole partition) and the distinction disappears — which is another reason the `ROWS ... UNBOUNDED FOLLOWING` fix is the reliable one.

### 6.5 NTH_VALUE — Positional Access

`NTH_VALUE(expr, n)` returns `expr` at the nth row of the frame, 1-based. It is subject to the same frame rules.

```sql
-- Second-highest salary per department (careful: needs full frame)
NTH_VALUE(salary, 2) OVER (
  PARTITION BY department_id
  ORDER BY salary DESC
  ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
)
```

If you omit the full frame, `NTH_VALUE(salary, 2)` under the default growing frame returns `NULL` on row 1 (frame has 1 row, no 2nd row), then the 2nd row's value from row 2 onward — a growing-frame artifact, almost never intended. Always pair `NTH_VALUE` counting toward a fixed position with an explicit whole-partition frame unless you specifically want the growing behavior.

**`FROM FIRST` vs `FROM LAST`.** The SQL standard (and PostgreSQL) lets you count from either end of the frame:

```sql
NTH_VALUE(product_name, 2) FROM FIRST OVER (...)   -- 2nd from the start (default)
NTH_VALUE(product_name, 2) FROM LAST  OVER (...)   -- 2nd from the end
```

`FROM LAST` with the full frame is the clean way to get "the second-to-last event." Note: PostgreSQL supports `FROM FIRST`/`FROM LAST` syntax for `NTH_VALUE`. (Historically some versions were restrictive; modern PostgreSQL 11+ accepts both.)

**Under-frame NULL.** If the frame has fewer than `n` rows, `NTH_VALUE` returns `NULL`. This is *not* an error; it is the defined result. Handle it with `COALESCE` if a default is desired.

### 6.6 IGNORE NULLS / RESPECT NULLS

All three functions accept a null-treatment clause. `RESPECT NULLS` is the default: NULLs in `expr` are counted as ordinary values. `IGNORE NULLS` skips rows where `expr IS NULL` before determining first/last/nth.

```sql
-- Last NON-NULL status seen so far (last-observation-carried-forward / LOCF)
LAST_VALUE(status) IGNORE NULLS OVER (
  PARTITION BY device_id
  ORDER BY reading_at
  ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
)
```

Here the default growing frame is *intentional*: on each row, "the last non-null status up to now" is exactly LOCF gap-filling. `IGNORE NULLS` turns `LAST_VALUE` from a trap into the standard tool for forward-filling sparse time series.

```sql
-- First non-null value in the whole partition
FIRST_VALUE(config_value) IGNORE NULLS OVER (
  PARTITION BY setting_key ORDER BY effective_at
)

-- The 3rd non-null measurement per sensor
NTH_VALUE(reading, 3) IGNORE NULLS OVER (
  PARTITION BY sensor_id ORDER BY taken_at
  ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
)
```

**Important interaction:** with `IGNORE NULLS`, the frame is still built the normal way, then NULLs are filtered *within the frame* before picking. A frame containing 5 rows of which 3 are NULL behaves, for selection purposes, like a 2-row list.

### 6.7 Empty and Single-Row Partitions

- A partition with exactly **one row**: `FIRST_VALUE` = `LAST_VALUE` = that row; `NTH_VALUE(_, 1)` = that row; `NTH_VALUE(_, 2)` = `NULL`.
- A frame that is **empty** (possible with certain explicit offset frames like `ROWS BETWEEN 5 FOLLOWING AND 10 FOLLOWING` near the partition end): all three return `NULL`. The default frame is never empty because it always includes the current row.
- With `IGNORE NULLS`, a frame whose values are **all NULL** behaves as empty → returns `NULL`.

### 6.8 Interaction with DISTINCT and Duplicate Output

Because window functions run before `DISTINCT`, applying `SELECT DISTINCT` after computing `FIRST_VALUE`/`LAST_VALUE` collapses identical output rows. A common pattern for "one row per group with its first and last value" is:

```sql
SELECT DISTINCT
  order_id,
  FIRST_VALUE(status) OVER w AS first_status,
  LAST_VALUE(status)  OVER w AS last_status
FROM order_status_history
WINDOW w AS (
  PARTITION BY order_id ORDER BY changed_at
  ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
);
```

Every row in a partition gets the same `first_status`/`last_status`, and `DISTINCT` reduces them to one row per `order_id`. A `GROUP BY order_id` will *not* work directly here because the window columns are not aggregates — `DISTINCT` (or a `ROW_NUMBER()=1` filter in a subquery) is the standard collapse.

### 6.9 The `WINDOW` Clause — Reuse and Readability

When the same frame is used by several functions, define it once with a named `WINDOW` clause. This avoids copy-paste frame errors — a real source of the LAST_VALUE bug (people fix the frame on one column and forget another):

```sql
SELECT
  order_id, changed_at,
  FIRST_VALUE(status) OVER w AS first_status,
  LAST_VALUE(status)  OVER w AS last_status,
  NTH_VALUE(status,2) OVER w AS second_status
FROM order_status_history
WINDOW w AS (
  PARTITION BY order_id
  ORDER BY changed_at
  ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
);
```

### 6.10 Relationship to LAG/LEAD (Topic 36)

`LAG`/`LEAD` access a row at a *fixed offset* from the current row and ignore the frame entirely (they always see the whole partition regardless of frame clause). `FIRST_VALUE`/`LAST_VALUE`/`NTH_VALUE` access a row at a *fixed position within the frame* and are governed by the frame. This is the key distinction:

- Want "the previous row's value"? → `LAG` (offset-based, frame-independent).
- Want "the first / last / nth row's value in this window"? → these three (position-based, frame-dependent).

`NTH_VALUE(x, 1)` equals `FIRST_VALUE(x)`. `LAST_VALUE(x)` with a full frame equals `NTH_VALUE(x, 1) FROM LAST`. And `FIRST_VALUE(x) OVER (ORDER BY t DESC)` is a common substitute for `LAST_VALUE(x) OVER (ORDER BY t ...FULL FRAME)`.

---

## 7. EXPLAIN — Window Functions in the Plan

### 7.1 Baseline: LAST_VALUE with an explicit full frame

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT
  order_id,
  changed_at,
  LAST_VALUE(status) OVER (
    PARTITION BY order_id
    ORDER BY changed_at
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
  ) AS final_status
FROM order_status_history;
```

```
WindowAgg  (cost=10982.34..13482.34 rows=100000 width=52)
           (actual time=142.1..268.9 rows=100000 loops=1)
  ->  Sort  (cost=10982.34..11232.34 rows=100000 width=44)
            (actual time=142.0..168.4 rows=100000 loops=1)
        Sort Key: order_id, changed_at
        Sort Method: quicksort  Memory: 12904kB
        ->  Seq Scan on order_status_history
                 (cost=0.00..1834.00 rows=100000 width=44)
                 (actual time=0.01..14.2 rows=100000 loops=1)
Buffers: shared hit=834
Planning Time: 0.21 ms
Execution Time: 279.6 ms
```

**Reading it:**
- `Seq Scan` reads all 100K rows (no filter → full scan).
- `Sort` orders by `order_id, changed_at` — this is the sort required by `PARTITION BY order_id ORDER BY changed_at`. `Sort Method: quicksort Memory: 12904kB` means it fit in `work_mem`; if it read `external merge Disk: NkB`, `work_mem` was too small and the sort spilled.
- `WindowAgg` is the node computing `LAST_VALUE`. It scans the sorted stream and, per partition, materializes rows to answer the `UNBOUNDED FOLLOWING` lookahead.

### 7.2 Eliminating the Sort with a matching index

```sql
CREATE INDEX idx_osh_order_changed
  ON order_status_history (order_id, changed_at);

EXPLAIN (ANALYZE, BUFFERS)
SELECT order_id, changed_at,
       FIRST_VALUE(status) OVER (PARTITION BY order_id ORDER BY changed_at) AS first_status
FROM order_status_history;
```

```
WindowAgg  (cost=0.42..8912.42 rows=100000 width=52)
           (actual time=0.05..121.7 rows=100000 loops=1)
  ->  Index Scan using idx_osh_order_changed on order_status_history
           (cost=0.42..6412.42 rows=100000 width=44)
           (actual time=0.03..41.9 rows=100000 loops=1)
Buffers: shared hit=101240
Planning Time: 0.30 ms
Execution Time: 132.4 ms
```

**Reading it:**
- No `Sort` node — the composite index `(order_id, changed_at)` supplies the exact order `WindowAgg` needs, so the planner does an ordered `Index Scan` and feeds it straight into `WindowAgg`.
- Execution time dropped (279ms → 132ms) chiefly by removing the sort.
- Note `Buffers: shared hit=101240` is higher than the seq-scan version's 834 — an index scan touches more buffer pages (index + heap) than a compact sequential scan. On a cold cache this trade can reverse; on a warm cache the sort elimination usually wins. This is exactly the kind of thing you confirm with `EXPLAIN (ANALYZE, BUFFERS)` rather than assume.

### 7.3 The full-frame lookahead cost

```sql
EXPLAIN (ANALYZE)
SELECT order_id,
       LAST_VALUE(status) OVER (
         PARTITION BY order_id ORDER BY changed_at
         ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS final_status
FROM order_status_history;
```

```
WindowAgg  (cost=10982.34..14232.34 rows=100000 width=52)
           (actual time=150.3..312.7 rows=100000 loops=1)
  Storage: Memory  Maximum Storage: 2048kB
  ->  Sort ...
```

The `Storage: Memory  Maximum Storage: 2048kB` line (PostgreSQL 16+) reports the tuplestore the `WindowAgg` used for random access. A full-partition frame forces the node to buffer each partition's rows so it can reach `UNBOUNDED FOLLOWING`. If a partition is larger than `work_mem`, this becomes `Storage: Disk` and slows down sharply — the practical scaling concern for these functions.

### 7.4 Signals to watch

| Signal in plan | Meaning | Action |
|----------------|---------|--------|
| `Sort` under `WindowAgg` | No index supplies window order | Add composite index on `(partition, order)` cols |
| `Sort Method: external merge Disk` | Sort spilled | Raise `work_mem` or add index |
| `WindowAgg ... Storage: Disk` | Frame buffer spilled to disk | Raise `work_mem`; shrink frame if possible |
| Two stacked `WindowAgg` nodes | Two different window definitions | Share one `WINDOW w AS (...)` if frames match |
| `Subquery Scan` above `WindowAgg` | Filtering on window result | Expected — window can't be in WHERE |

---

## 8. Query Examples

### Example 1 — Basic: First and last status per order

```sql
-- Order status history → the very first and the very last status of each order.
-- FIRST_VALUE is safe under the default frame; LAST_VALUE needs the full frame.
SELECT DISTINCT
  order_id,
  FIRST_VALUE(status) OVER w AS first_status,   -- earliest status
  LAST_VALUE(status)  OVER w AS current_status   -- latest status (needs full frame!)
FROM order_status_history
WINDOW w AS (
  PARTITION BY order_id
  ORDER BY changed_at
  ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING  -- ⚠ the fix
)
ORDER BY order_id;
```

### Example 2 — Intermediate: Session entry/exit and second event

```sql
-- Per user session: landing page (first), exit page (last), and the 2nd page viewed.
-- Uses a named window so all three functions share the identical full frame.
SELECT
  s.session_id,
  s.user_id,
  FIRST_VALUE(pv.page_path)     OVER w AS landing_page,
  LAST_VALUE(pv.page_path)      OVER w AS exit_page,
  NTH_VALUE(pv.page_path, 2)    OVER w AS second_page,   -- NULL if session had 1 view
  NTH_VALUE(pv.page_path, 2) FROM LAST OVER w AS penultimate_page
FROM sessions s
JOIN page_views pv ON pv.session_id = s.session_id
WHERE s.started_at >= CURRENT_DATE - INTERVAL '7 days'
WINDOW w AS (
  PARTITION BY s.session_id
  ORDER BY pv.viewed_at
  ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
)
ORDER BY s.session_id;
```

### Example 3 — Production Grade: Account balance snapshot with LOCF

```sql
-- Context:
--   payments: ~40M rows, one row per transaction, some rows have NULL balance_after
--             (legacy imports left gaps). We need, per account per day snapshot:
--             the opening balance (first non-null of the day) and the closing
--             balance (last non-null of the day, carried forward over NULLs).
--   Index available: idx_payments_acct_time ON payments(account_id, occurred_at)
--                    → supplies the window order, eliminating the Sort node.
--   Perf expectation: single ordered Index Scan + one WindowAgg pass; linear in rows
--                     within each account partition. Target < 2s for one account-month.
SELECT DISTINCT
  account_id,
  DATE(occurred_at) AS day,
  FIRST_VALUE(balance_after) IGNORE NULLS OVER w AS opening_balance,
  LAST_VALUE(balance_after)  IGNORE NULLS OVER w AS closing_balance
FROM payments
WHERE account_id = $1
  AND occurred_at >= $2::timestamptz
  AND occurred_at <  $3::timestamptz
WINDOW w AS (
  PARTITION BY account_id, DATE(occurred_at)
  ORDER BY occurred_at
  ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
)
ORDER BY day;
```

```
-- EXPLAIN (ANALYZE, BUFFERS) sketch for one account-month (~9,000 rows):
Unique  (cost=0.56..742.10 rows=30 width=48) (actual time=1.9..47.3 rows=30 loops=1)
  ->  WindowAgg (cost=0.56..697.10 rows=9000 width=48) (actual time=1.9..44.1 rows=9000 loops=1)
        ->  Index Scan using idx_payments_acct_time on payments
                 Index Cond: (account_id = $1 AND occurred_at >= $2 AND occurred_at < $3)
                 (actual time=0.04..18.2 rows=9000 loops=1)
Buffers: shared hit=612
Execution Time: 48.9 ms
```

Reading: the `Index Scan` both filters (`Index Cond`) and delivers rows pre-ordered by `occurred_at`, so there is no `Sort`. `WindowAgg` does one pass; `Unique` implements the `DISTINCT`. Well under the 2s target.

---

## 9. Wrong → Right Patterns

### Wrong 1: LAST_VALUE with the default frame

```sql
-- WRONG: intends "final status of each order", gets "this row's status"
SELECT
  order_id, changed_at, status,
  LAST_VALUE(status) OVER (PARTITION BY order_id ORDER BY changed_at) AS final_status
FROM order_status_history;
```

**Exact wrong result:** `final_status` equals `status` on every row (each row's own status), because the default frame `RANGE ... CURRENT ROW` ends at the current row. On the last row of each partition it happens to be right, so casual testing on a one-row-per-order sample "passes."

**Why at the execution level:** `WindowAgg` evaluates `LAST_VALUE` over the *frame*, and the default frame's upper bound is the current row (and its peers). The last row of a growing frame is the current row.

```sql
-- RIGHT: widen the frame to the whole partition
SELECT
  order_id, changed_at, status,
  LAST_VALUE(status) OVER (
    PARTITION BY order_id ORDER BY changed_at
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
  ) AS final_status
FROM order_status_history;

-- RIGHT (alternative): FIRST_VALUE on reversed order, no frame needed
SELECT
  order_id, changed_at, status,
  FIRST_VALUE(status) OVER (
    PARTITION BY order_id ORDER BY changed_at DESC
  ) AS final_status
FROM order_status_history;
```

### Wrong 2: Filtering on a window function in WHERE

```sql
-- WRONG: window functions are not allowed in WHERE (runs before the window phase)
SELECT order_id, status
FROM order_status_history
WHERE LAST_VALUE(status) OVER (
        PARTITION BY order_id ORDER BY changed_at
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) = 'cancelled';
```

**Exact error:**
```
ERROR:  window functions are not allowed in WHERE
LINE 3: WHERE LAST_VALUE(status) OVER (...)
```

**Why:** `WHERE` executes in the row-filtering phase, strictly before the SELECT/window phase. The window result does not exist yet.

```sql
-- RIGHT: compute in a subquery/CTE, filter in the outer query
WITH labelled AS (
  SELECT order_id, status, changed_at,
         LAST_VALUE(status) OVER (
           PARTITION BY order_id ORDER BY changed_at
           ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS final_status
  FROM order_status_history
)
SELECT DISTINCT order_id
FROM labelled
WHERE final_status = 'cancelled';
```

### Wrong 3: NTH_VALUE without a full frame for a fixed position

```sql
-- WRONG: wants "the 2nd-largest sale amount per region", uses default frame
SELECT
  region, amount,
  NTH_VALUE(amount, 2) OVER (PARTITION BY region ORDER BY amount DESC) AS second_largest
FROM sales;
```

**Exact wrong result:** `second_largest` is `NULL` on the first row of each region (frame has 1 row) and then, from row 2 onward, becomes the 2nd row's amount — a value that changes down the partition instead of a single per-region answer. The growing default frame makes "the 2nd value" position-relative to the current row.

```sql
-- RIGHT: fix the frame to the whole partition so "2nd" is stable
SELECT DISTINCT
  region,
  NTH_VALUE(amount, 2) OVER (
    PARTITION BY region ORDER BY amount DESC
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
  ) AS second_largest
FROM sales;
```

### Wrong 4: Forgetting NULLs corrupt FIRST_VALUE/LAST_VALUE

```sql
-- WRONG: "opening balance" but the earliest row has a NULL balance_after
SELECT DISTINCT account_id,
  FIRST_VALUE(balance_after) OVER (
    PARTITION BY account_id ORDER BY occurred_at) AS opening_balance
FROM payments;
-- If the first row's balance_after IS NULL, opening_balance = NULL for the whole account
```

**Why:** `RESPECT NULLS` (the default) counts the NULL row as the first row, so `FIRST_VALUE` faithfully returns NULL.

```sql
-- RIGHT: skip NULLs
SELECT DISTINCT account_id,
  FIRST_VALUE(balance_after) IGNORE NULLS OVER (
    PARTITION BY account_id ORDER BY occurred_at) AS opening_balance
FROM payments;
```

### Wrong 5: Mixing up window ORDER BY and output ORDER BY

```sql
-- WRONG: expects rows printed oldest-first, but only set the window order
SELECT order_id, changed_at,
  FIRST_VALUE(status) OVER (PARTITION BY order_id ORDER BY changed_at) AS first_status
FROM order_status_history;
-- Output row order is UNDEFINED — the OVER(...ORDER BY) only sequences the frame,
-- it does NOT order the printed result. Rows may come back in any physical order.
```

```sql
-- RIGHT: add a top-level ORDER BY for output ordering
SELECT order_id, changed_at,
  FIRST_VALUE(status) OVER (PARTITION BY order_id ORDER BY changed_at) AS first_status
FROM order_status_history
ORDER BY order_id, changed_at;   -- explicit output ordering
```

---

## 10. Performance Profile

### 10.1 Cost model

The dominant costs of `FIRST_VALUE`/`LAST_VALUE`/`NTH_VALUE` are:

1. **The sort** (or index scan) to establish window order — O(n log n) if sorted, O(n) if an index supplies the order.
2. **The WindowAgg pass** — one linear scan, O(n), *plus* per-row random access into the frame buffer.
3. **The frame buffer (tuplestore)** — memory proportional to the largest partition when the frame reaches `UNBOUNDED FOLLOWING`.

For `FIRST_VALUE` with the default frame, the executor keeps only the first row's value per partition — cheap, O(1) extra state. For `LAST_VALUE`/`NTH_VALUE` with a full frame, the executor must buffer each partition to reach the far end — memory scales with partition size, and spills to disk past `work_mem`.

### 10.2 Scaling

| Rows | Partitions | Frame | Typical behavior |
|------|-----------|-------|------------------|
| 1M | many small | default (FIRST_VALUE) | ~200–500ms, sort-bound; O(1) frame state |
| 1M | many small | full (LAST_VALUE) | ~250–600ms; small per-partition buffers, in memory |
| 10M | many small | full | ~3–8s; sort dominates — add index to cut it |
| 10M | few HUGE partitions | full | Danger: partition buffer may exceed work_mem → Disk spill |
| 100M | with matching index | any | Index scan removes sort; WindowAgg is the linear floor |

### 10.3 Optimization techniques

- **Add a composite index on `(partition_cols, order_cols)`.** This is the single biggest win — it removes the `Sort` node entirely (see 7.2). For `PARTITION BY account_id ORDER BY occurred_at`, index `(account_id, occurred_at)`.
- **Prefer `FIRST_VALUE` on reversed order over `LAST_VALUE` with a full frame** when you only need the last value. `FIRST_VALUE`'s default frame keeps O(1) state and needs no partition buffering.
- **Raise `work_mem`** for the session/query when partitions are large and the plan shows `Sort Method: external merge` or `WindowAgg Storage: Disk`. Set it locally: `SET LOCAL work_mem = '256MB';` inside a transaction so it does not affect other queries.
- **Narrow the frame if semantics allow.** If you truly only need a bounded lookback, `ROWS BETWEEN 10 PRECEDING AND CURRENT ROW` buffers at most 11 rows regardless of partition size.
- **Push filters into `WHERE`** so the window operates on fewer rows. The window runs after `WHERE`, so filtering first shrinks both the sort and the frame buffer.
- **Consider `DISTINCT ON` for pure first/last-per-group** when you do not need the value on every row. `SELECT DISTINCT ON (order_id) ... ORDER BY order_id, changed_at DESC` fetches the last row per group in one pass and can be cheaper than a window when you only want one row per group (Topic covered under DISTINCT ON).

---

## 11. Node.js Integration

### 11.1 First and last status per order

```javascript
import pg from 'pg';
const { Pool } = pg;
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

async function getOrderStatusBounds(orderIds) {
  const { rows } = await pool.query(
    `SELECT DISTINCT
       order_id,
       FIRST_VALUE(status) OVER w AS first_status,
       LAST_VALUE(status)  OVER w AS current_status
     FROM order_status_history
     WHERE order_id = ANY($1::bigint[])
     WINDOW w AS (
       PARTITION BY order_id
       ORDER BY changed_at
       ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
     )
     ORDER BY order_id`,
    [orderIds]   // pass a JS array → bound as a bigint[] via $1
  );
  return rows;
}
```

### 11.2 LOCF closing balance with IGNORE NULLS and a session work_mem bump

```javascript
async function getDailyBalances(accountId, from, to) {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');
    // Local to this transaction only — safe for a heavy window over a big partition.
    await client.query("SET LOCAL work_mem = '256MB'");
    const { rows } = await client.query(
      `SELECT DISTINCT
         DATE(occurred_at) AS day,
         FIRST_VALUE(balance_after) IGNORE NULLS OVER w AS opening_balance,
         LAST_VALUE(balance_after)  IGNORE NULLS OVER w AS closing_balance
       FROM payments
       WHERE account_id = $1
         AND occurred_at >= $2::timestamptz
         AND occurred_at <  $3::timestamptz
       WINDOW w AS (
         PARTITION BY account_id, DATE(occurred_at)
         ORDER BY occurred_at
         ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
       )
       ORDER BY day`,
      [accountId, from, to]
    );
    await client.query('COMMIT');
    return rows;
  } catch (e) {
    await client.query('ROLLBACK');
    throw e;
  } finally {
    client.release();
  }
}
```

### 11.3 nth-value with a runtime position parameter

```javascript
// The nth position can be parameterized; the frame cannot (it is query structure).
async function getNthPageView(sessionId, n) {
  const { rows } = await pool.query(
    `SELECT DISTINCT
       session_id,
       NTH_VALUE(page_path, $2) OVER (
         PARTITION BY session_id
         ORDER BY viewed_at
         ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
       ) AS nth_page
     FROM page_views
     WHERE session_id = $1`,
    [sessionId, n]   // $2 = n; NULL if the session had fewer than n views
  );
  return rows[0]?.nth_page ?? null;
}
```

**ORM note:** No mainstream Node ORM models `FIRST_VALUE`/`LAST_VALUE`/`NTH_VALUE` as first-class methods with a frame clause. In every case you either write the window expression as a raw SQL fragment inside the query builder or drop to a fully raw query. Treat these as raw-SQL territory by default.

---

## 12. ORM Comparison

### Prisma

**Can Prisma do it?** No, not through its query API. Prisma has no window-function support and no frame concept. The only path is `$queryRaw`.

```typescript
import { PrismaClient, Prisma } from '@prisma/client';
const prisma = new PrismaClient();

const rows = await prisma.$queryRaw<Array<{ order_id: bigint; current_status: string }>>(
  Prisma.sql`
    SELECT DISTINCT
      order_id,
      LAST_VALUE(status) OVER (
        PARTITION BY order_id ORDER BY changed_at
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
      ) AS current_status
    FROM order_status_history
    WHERE order_id = ${orderId}
  `
);
```

**Where it breaks:** everything — there is no `include`/`select` equivalent. **Verdict:** raw SQL only. Keep the frame clause in the raw string; it is the whole point.

---

### Drizzle ORM

**Can Drizzle do it?** Partially. Drizzle has no dedicated `firstValue`/`lastValue` helpers, but its `sql` template tag composes window expressions cleanly, and you keep type inference on the surrounding query.

```typescript
import { sql } from 'drizzle-orm';
import { db } from './db';
import { orderStatusHistory } from './schema';

const rows = await db
  .select({
    orderId: orderStatusHistory.orderId,
    currentStatus: sql<string>`
      LAST_VALUE(${orderStatusHistory.status}) OVER (
        PARTITION BY ${orderStatusHistory.orderId}
        ORDER BY ${orderStatusHistory.changedAt}
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
      )`.as('current_status'),
  })
  .from(orderStatusHistory);
// Add .groupBy / a distinct wrapper as needed — Drizzle has no native DISTINCT-on-window.
```

**Where it breaks:** the frame clause and `IGNORE NULLS` must be hand-written in the `sql` tag; Drizzle validates none of it. **Verdict:** good — you write real SQL for the window while keeping typed columns and params.

---

### Sequelize

**Can Sequelize do it?** Only via `sequelize.literal()` inside attributes, or a fully raw query. There is no window-function builder.

```javascript
const { QueryTypes } = require('sequelize');

const rows = await sequelize.query(
  `SELECT DISTINCT
     order_id,
     LAST_VALUE(status) OVER (
       PARTITION BY order_id ORDER BY changed_at
       ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
     ) AS current_status
   FROM order_status_history
   WHERE order_id = :orderId`,
  { replacements: { orderId }, type: QueryTypes.SELECT }
);
```

**Where it breaks:** `literal()` window expressions are opaque strings with no validation, and mixing them with Sequelize's auto-generated `GROUP BY`/aliasing is fragile. **Verdict:** use `sequelize.query()` raw; avoid trying to shoehorn it into `findAll`.

---

### TypeORM

**Can TypeORM do it?** Via `addSelect` with a raw expression in the QueryBuilder, or `dataSource.query()`.

```typescript
const rows = await dataSource
  .getRepository(OrderStatusHistory)
  .createQueryBuilder('osh')
  .select('osh.order_id', 'order_id')
  .addSelect(
    `LAST_VALUE(osh.status) OVER (
       PARTITION BY osh.order_id ORDER BY osh.changed_at
       ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING)`,
    'current_status',
  )
  .distinct(true)
  .getRawMany();
```

**Where it breaks:** the window is a raw string (no type safety, no column-name checking), and `.distinct(true)` plus raw window columns can produce awkward SQL. **Verdict:** works through `addSelect`/`getRawMany`; treat as raw.

---

### Knex.js

**Can Knex do it?** Yes, most transparently — Knex exposes no window helper but `knex.raw()` in a `select` reads almost like SQL.

```javascript
const rows = await knex('order_status_history')
  .distinct()
  .select('order_id')
  .select(
    knex.raw(`
      LAST_VALUE(status) OVER (
        PARTITION BY order_id ORDER BY changed_at
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
      ) AS current_status`)
  )
  .where('order_id', orderId);
```

**Where it breaks:** nothing structurally — but the entire window, including the critical frame clause, lives in a raw string you must get right yourself. **Verdict:** most SQL-transparent; the frame is your responsibility.

---

### ORM Summary Table

| ORM | Native support | How to express | Frame clause | IGNORE NULLS | Verdict |
|-----|----------------|----------------|--------------|--------------|---------|
| Prisma | None | `$queryRaw` | raw string | raw string | Raw SQL only |
| Drizzle | None (composable `sql`) | `sql` tag | raw in `sql` tag | raw in `sql` tag | Best typed-raw balance |
| Sequelize | None | `literal()` / raw | raw string | raw string | Use raw `query()` |
| TypeORM | None | `addSelect` raw | raw string | raw string | Raw via `getRawMany` |
| Knex | None | `knex.raw()` | raw string | raw string | Most transparent |

**Across all five: the frame clause is never abstracted.** This is precisely why the `LAST_VALUE` trap survives into production — no ORM defaults it for you, so if the developer omits `UNBOUNDED FOLLOWING`, it is silently wrong.

---

## 13. Practice Exercises

### Exercise 1 — Basic

Given:
- `employees(id, name, salary, department_id, hire_date)`

For each employee, show `name`, `department_id`, `salary`, and two extra columns: `first_hired_name` (the name of the earliest-hired employee in that department) and `highest_paid_name` (the name of the highest-paid employee in that department). Which of your two window columns needs an explicit `UNBOUNDED FOLLOWING` frame and why?

```sql
-- Write your query here
```

---

### Exercise 2 — Medium (combines Topic 36 LAG/LEAD)

Given:
- `order_status_history(id, order_id, status, changed_at)`

For each status-change row, show `order_id`, `status`, `changed_at`, the **previous** status (`LAG`), and the **final** status of the order (`LAST_VALUE` with the correct frame). Then explain in a comment why `LAG` needs no frame clause while `LAST_VALUE` does.

```sql
-- Write your query here
```

---

### Exercise 3 — Hard (production simulation; naive answer is wrong)

Given:
- `sensor_readings(sensor_id, taken_at, temperature)` — 50M rows; `temperature` is NULL on ~15% of rows due to dropped packets.

Produce, per sensor per calendar day, the `first_reading` (first non-null temperature of the day), the `last_reading` (last non-null temperature of the day, carried forward over trailing NULLs), and the `third_reading` (the 3rd non-null temperature of the day, or NULL if fewer than three). The naive answer forgets `IGNORE NULLS` and/or uses the default frame — both are wrong. State the index you would create to avoid a sort.

```sql
-- Write your query here
```

---

### Exercise 4 — Interview Simulation

You are asked live: "Here is a query that returns each account's latest transaction amount. It runs, but finance says the numbers look like each row's own amount, not the latest. Diagnose and fix it, then make it fast on a 40M-row table."

```sql
SELECT account_id, occurred_at,
       LAST_VALUE(amount) OVER (PARTITION BY account_id ORDER BY occurred_at) AS latest_amount
FROM payments;
```

```sql
-- Write your corrected + optimized query here, and note the index you'd add
```

---

## 14. Interview Questions

### Q1 — The LAST_VALUE trap

**Question:** "Why does `LAST_VALUE(x) OVER (ORDER BY t)` return the current row's value instead of the partition's last value, and how do you fix it?"

**Junior answer:** "I think there's a bug in LAST_VALUE, so I'd use a subquery with `ORDER BY t DESC LIMIT 1` instead."

**Principal answer:** "It is not a bug — it is the default frame. When you supply `ORDER BY` but no explicit frame, PostgreSQL applies `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`. The frame grows as you descend the partition and always ends at the current row (and its ORDER BY peers), so the 'last' row of that frame is the current row. `FIRST_VALUE` escapes this because the first row is always inside the frame. Two fixes: widen the frame with `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`, or — cleaner — reverse the order and use `FIRST_VALUE(x) OVER (ORDER BY t DESC)`, which keeps the safe default frame and O(1) executor state."

**Follow-up:** "Under the default `RANGE` frame, what does `LAST_VALUE` return when several rows tie on `t`?" — *It returns the value of the last peer in the tie group, because `RANGE ... CURRENT ROW` extends through all rows equal to the current row on the ORDER BY key; switching to `ROWS` would stop at the exact physical current row.*

---

### Q2 — RANGE vs ROWS in the frame

**Question:** "You changed a `LAST_VALUE` frame from `RANGE` to `ROWS` and the numbers changed even though you didn't touch `UNBOUNDED FOLLOWING`. Explain."

**Junior answer:** "`ROWS` and `RANGE` are just two ways to write the same thing; the result shouldn't change."

**Principal answer:** "They differ at the `CURRENT ROW` boundary when there are ties on the ORDER BY key. `RANGE` treats `CURRENT ROW` as 'the last peer of the current row' — all rows with equal ordering values are in-frame together. `ROWS` treats `CURRENT ROW` as exactly the current physical row. So with duplicate order keys, `RANGE ... CURRENT ROW` includes later tied rows that `ROWS ... CURRENT ROW` excludes, changing what `LAST_VALUE` and `NTH_VALUE` return. The difference vanishes if the order key is unique, or if you extend the upper bound to `UNBOUNDED FOLLOWING` (both then cover the whole partition). That is one more reason the full-frame fix is the robust one."

**Follow-up:** "Which is faster, `ROWS` or `RANGE`?" — *`ROWS` is generally cheaper because peer detection under `RANGE` requires comparing ordering values to find peer-group boundaries; `ROWS` uses simple physical offsets. For the full `UNBOUNDED PRECEDING/FOLLOWING` frame the difference is negligible.*

---

### Q3 — NTH_VALUE and NULL handling

**Question:** "How would you get the second-most-recent non-null login timestamp per user, and what are the edge cases?"

**Junior answer:** "`NTH_VALUE(login_at, 2) OVER (PARTITION BY user_id ORDER BY login_at DESC)`."

**Principal answer:** "Close, but two things. First, the default growing frame makes 'the 2nd value' position-relative to the current row, so I need `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING` to make '2nd' a stable per-user answer. Second, if some `login_at` values are NULL and I want the 2nd *non-null*, I add `IGNORE NULLS`. Edge case: users with fewer than two non-null logins get `NULL` from `NTH_VALUE` — that is the defined result, not an error, so I'd `COALESCE` if a sentinel is needed. Full form: `NTH_VALUE(login_at, 2) IGNORE NULLS OVER (PARTITION BY user_id ORDER BY login_at DESC ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING)`, wrapped in `SELECT DISTINCT` or a `ROW_NUMBER()=1` filter to collapse to one row per user."

**Follow-up:** "Is `NTH_VALUE(x, 1)` the same as `FIRST_VALUE(x)`?" — *Yes, identical. And `NTH_VALUE(x, 1) FROM LAST` over a full frame equals `LAST_VALUE(x)` over that frame.*

---

### Q4 — Performance on huge partitions

**Question:** "A `LAST_VALUE` with a full frame is slow on a table with a few enormous partitions. Why, and what do you do?"

**Principal answer:** "The `UNBOUNDED FOLLOWING` upper bound forces `WindowAgg` to buffer each partition's rows in a tuplestore so it can reach the far end. Memory scales with the largest partition; past `work_mem` it spills to disk (`Storage: Disk` in EXPLAIN), which is where the slowdown comes from — plus the sort if no index supplies the order. Fixes, in order: add a composite index on `(partition_cols, order_cols)` to remove the sort; if I only need the last value, switch to `FIRST_VALUE` on reversed order, which needs no partition buffering and keeps O(1) state; raise `work_mem` locally for the query if buffering is unavoidable; or, if I only want one row per partition rather than a value on every row, use `DISTINCT ON` which is a single ordered pass."

**Follow-up:** "Does adding the index change correctness?" — *No. The index only supplies the required order for free; the result is identical, just without a Sort node.*

---

## 15. Mental Model Checkpoint

1. A partition has rows with values [A, B, C, D] in order. Under the *default* frame, what does `LAST_VALUE` return on the row holding `B`? What does `FIRST_VALUE` return there? Why do they differ?

2. You write `LAST_VALUE(x) OVER (ORDER BY t ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING)`. If you instead wrote `RANGE` instead of `ROWS`, would the result change? Under what condition would it *not* matter?

3. Rewrite `LAST_VALUE(status) OVER (PARTITION BY id ORDER BY t ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING)` using `FIRST_VALUE`. What must you watch out for regarding NULL ordering?

4. Under the default frame, `NTH_VALUE(x, 2)` returns NULL on the first row of every partition. Explain why in terms of frame size.

5. A time series has NULL gaps. You want last-observation-carried-forward. Which function, which null-treatment clause, and which frame — and why is the *default* frame correct here rather than the full frame?

6. Why can you not put `LAST_VALUE(...) = 'done'` in a `WHERE` clause, and what is the minimal rewrite that makes it work?

7. Two rows tie on the `ORDER BY` key. You need the *physically* current row to be the frame's last row (not the last peer). Which frame mode do you choose and what exact bound do you write?

---

## 16. Quick Reference Card

```sql
-- ── THE THREE FUNCTIONS ──────────────────────────────────────────
FIRST_VALUE(expr) OVER (PARTITION BY p ORDER BY o [frame])   -- first row of frame
LAST_VALUE(expr)  OVER (PARTITION BY p ORDER BY o  frame )   -- last row of frame
NTH_VALUE(expr,n) OVER (PARTITION BY p ORDER BY o  frame )   -- nth row (1-based)
        [FROM FIRST | FROM LAST]        -- count from frame start (default) or end
        [RESPECT NULLS | IGNORE NULLS]  -- skip NULL values of expr?

-- ── THE DEFAULT FRAME (memorize) ─────────────────────────────────
-- With ORDER BY, no explicit frame  →  RANGE UNBOUNDED PRECEDING .. CURRENT ROW
-- Consequence: FIRST_VALUE = OK,  LAST_VALUE = returns CURRENT ROW (the trap!)

-- ── THE LAST_VALUE FIX ───────────────────────────────────────────
LAST_VALUE(x) OVER (PARTITION BY p ORDER BY o
                    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING)
-- Cleaner equivalent (no frame needed, cheaper state):
FIRST_VALUE(x) OVER (PARTITION BY p ORDER BY o DESC NULLS LAST)

-- ── ROWS vs RANGE at CURRENT ROW ─────────────────────────────────
-- ROWS  = exact physical current row
-- RANGE = current row + all its ORDER-BY peers (ties)
-- They converge when the order key is unique OR bound is UNBOUNDED FOLLOWING

-- ── NULL / POSITION EDGE CASES ───────────────────────────────────
-- frame shorter than n           → NTH_VALUE = NULL (not an error)
-- IGNORE NULLS                    → NULLs skipped before first/last/nth chosen
-- single-row partition           → FIRST_VALUE = LAST_VALUE = that row
-- NTH_VALUE(x,1) == FIRST_VALUE(x); NTH_VALUE(x,1) FROM LAST == LAST_VALUE(x)

-- ── LOCF (carry forward) ─────────────────────────────────────────
LAST_VALUE(x) IGNORE NULLS OVER (PARTITION BY p ORDER BY o
                                 ROWS UNBOUNDED PRECEDING)  -- default growing frame is CORRECT here

-- ── FILTERING ON A WINDOW RESULT (WHERE forbidden) ───────────────
WITH w AS (SELECT ..., LAST_VALUE(...) OVER (...) AS lv FROM t)
SELECT * FROM w WHERE lv = 'x';

-- ── PERF RULES OF THUMB ──────────────────────────────────────────
-- Index (partition_cols, order_cols)  → removes the Sort node
-- Full frame on huge partitions       → buffers partition; watch work_mem spill
-- Prefer FIRST_VALUE(reversed) to LAST_VALUE(full frame) for "last value"
-- Use one WINDOW w AS (...) for multiple funcs → no copy-paste frame bugs
-- Only need one row/group? DISTINCT ON may beat a window

-- ── INTERVIEW ONE-LINERS ─────────────────────────────────────────
-- "LAST_VALUE returns the current row by default because the default frame
--  ends at CURRENT ROW; widen to UNBOUNDED FOLLOWING or reverse + FIRST_VALUE."
-- "FIRST_VALUE is safe because the first row is always in the default frame."
-- "RANGE includes tied peers at CURRENT ROW; ROWS is the exact physical row."
-- "NTH_VALUE returns NULL, not an error, when the frame is shorter than n."
-- "IGNORE NULLS turns LAST_VALUE into the LOCF gap-fill tool."
```

---

## Connected Topics

- **Topic 35 — Window Functions & OVER / PARTITION BY**: The `OVER`, `PARTITION BY`, and `ORDER BY` mechanics these functions build on; the foundation of the window phase.
- **Topic 36 — LAG and LEAD**: The offset-based cousins — frame-independent, fixed-distance access — versus these position-based, frame-dependent functions. Know when to reach for which.
- **Topic 38 — Aggregate Functions as Window Functions**: `SUM`/`AVG`/`COUNT` `OVER (...)` share the exact same frame machinery; the default-frame running-total behavior is the same mechanism seen from the aggregate side.
- **Topic 39 — Window Frames (ROWS / RANGE / GROUPS)**: The full treatment of frame modes and boundaries — the deep dive behind the `UNBOUNDED FOLLOWING` fix and the RANGE-vs-ROWS tie subtlety introduced here.
- **Topic 11 — INNER JOIN**: Window functions run after joins; the fan-out from a one-to-many join changes what "first/last row per partition" means.
- **Topic 07 — NULL in Depth**: `RESPECT NULLS`/`IGNORE NULLS` and the three-valued logic that decides whether a NULL row counts as first/last/nth.
- **DISTINCT ON**: The single-pass alternative when you want only one row per group rather than a value broadcast onto every row.
