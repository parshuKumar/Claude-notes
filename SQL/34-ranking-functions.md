# Topic 34 — Ranking Functions
### SQL Mastery Curriculum — Phase 6: Window Functions — Complete Mastery

---

## 1. ELI5 — The Analogy That Makes It Click

Picture a school sports day, 100-metre sprint. Eight kids run. The stopwatch records their times, and now the teacher has to hand out position numbers.

Three different teachers are standing at the finish line, each holding a different rulebook.

**Teacher ROW_NUMBER** is a bureaucrat. She doesn't care about ties. She has exactly 8 numbered stickers — 1, 2, 3, 4, 5, 6, 7, 8 — and she *will* stick one on each child, no matter what. Two kids crossed the line at exactly the same time? Doesn't matter. One gets sticker 3, the other gets sticker 4. She flips a coin if she has to. Every child gets a **unique** number, and every number from 1 to 8 is used exactly once.

**Teacher RANK** is a sports purist. If two kids genuinely tied for the same time, they *both* get the same medal position. Two kids tied for 3rd? They're **both** 3rd. But then — and this is the key — she *skips* 4th place entirely. The next child down is 5th, because two people are ahead of them in the "3rd place" slot. Positions look like: 1, 2, **3, 3**, 5, 6... There's a **gap** where 4th should be.

**Teacher DENSE_RANK** is a gentler purist. Same rule for ties — two kids tied for 3rd are both 3rd. But she refuses to skip numbers. The next child is 4th, not 5th, because she's ranking *distinct times*, not *people ahead of you*. Positions look like: 1, 2, **3, 3**, 4, 5... **No gap.**

That's the whole thing. The moment there are **no ties**, all three teachers produce identical numbers. The instant a tie appears, they diverge — and the entire interview question is about knowing exactly how.

One more subtlety that ruins careless answers: what counts as a "tie"? A tie is when the values in the `ORDER BY` are *equal*. If two runners have times of 12.40 and 12.41, they are **not** tied — even by a hundredth of a second. RANK and DENSE_RANK only merge rows whose ordering columns are byte-for-byte equal.

---

## 2. Connection to SQL Internals

Ranking functions are **window functions**. Underneath, PostgreSQL implements them with a dedicated executor node: **WindowAgg** (window aggregate). Understanding what that node does is understanding ranking functions.

The mechanics:

1. **Sort first.** A window function with `ORDER BY` (and/or `PARTITION BY`) requires its input to be sorted on `(partition_cols, order_cols)`. The planner inserts a **Sort** node beneath the WindowAgg — *unless* an index already provides that ordering, in which case an **Index Scan** feeds the WindowAgg pre-sorted and the Sort disappears. This is the single biggest performance lever for ranking queries.

2. **Partition boundaries.** As WindowAgg reads the sorted stream, it watches for changes in the `PARTITION BY` columns. When they change, it resets its internal counters — a new partition begins. Ranking restarts at 1.

3. **Peer groups and the counter state.** Within a partition, WindowAgg tracks a running row counter and a "peer group" boundary. Two adjacent rows are **peers** if their `ORDER BY` values are equal. This peer detection is *the* mechanism that makes RANK and DENSE_RANK differ from ROW_NUMBER:
   - `ROW_NUMBER` increments a counter every row, ignoring peers entirely.
   - `RANK` emits the counter value *of the first row in the current peer group* (position = number of rows that came before + 1).
   - `DENSE_RANK` emits a separate counter that increments only when the peer group *changes*.

4. **No hash table, no random access.** Unlike GROUP BY aggregation (which can use a hash table), ranking window functions are fundamentally **streaming over sorted input**. This is why they scale predictably: the cost is dominated by the sort, which is `O(N log N)`, plus a single `O(N)` pass.

5. **Memory.** Pure ranking functions (ROW_NUMBER/RANK/DENSE_RANK) need only a few integers of state per partition — they do **not** buffer the whole partition in memory the way some frame-based functions do. The memory pressure comes entirely from the **Sort** (governed by `work_mem`; spills to disk as an external merge sort if it doesn't fit).

So when you write `RANK() OVER (PARTITION BY category ORDER BY price DESC)`, the engine is: sort by `(category, price DESC)`, stream through it, reset counters at each category change, and emit rank values based on peer detection. That's the entire internal story.

Remember from Topic 33 (PARTITION BY and ORDER BY) that the `OVER()` clause defines *the window* — which rows the function sees and in what order. Ranking functions are the simplest consumers of that window: they only need the ordering, never a frame.

---

## 3. Logical Execution Order Context

```
FROM
JOIN
WHERE
GROUP BY
HAVING
SELECT           ← window functions (ranking) are computed HERE
  DISTINCT
ORDER BY
LIMIT
```

Window functions are evaluated in the **SELECT phase**, *after* `FROM`, `WHERE`, `GROUP BY`, and `HAVING`, but *before* the final `DISTINCT`, `ORDER BY`, and `LIMIT`.

This ordering has three hard consequences you must internalise:

**1. You cannot reference a window function in WHERE or HAVING.** WHERE runs before the window is computed. This fails:

```sql
-- ERROR: window functions are not allowed in WHERE
SELECT name, RANK() OVER (ORDER BY salary DESC) AS r
FROM employees
WHERE RANK() OVER (ORDER BY salary DESC) <= 3;
-- ERROR:  window functions are not allowed in WHERE
```

To filter on a rank, you must wrap the query so the rank becomes an ordinary column in an outer query:

```sql
SELECT * FROM (
  SELECT name, RANK() OVER (ORDER BY salary DESC) AS r
  FROM employees
) ranked
WHERE r <= 3;
```

This "compute in a subquery/CTE, filter outside" pattern is the most common shape for ranking queries. You will write it hundreds of times.

**2. WHERE filters the input to the ranking, not the output.** Because WHERE runs first, a `WHERE status = 'active'` clause removes rows *before* ranks are assigned. The rank of the top active employee is 1 — inactive employees never entered the window and never consumed a rank number. This is usually what you want, but it means "rank among all employees, then show only active ones" requires the outer-filter pattern above, not a WHERE.

**3. ORDER BY in the query is independent of ORDER BY in the window.** The `ORDER BY` inside `OVER(...)` controls *rank assignment*. The `ORDER BY` at the end of the query controls *the display order of the output rows*. They can differ. You can rank by salary descending but display alphabetically by name. Confusing these two `ORDER BY`s is a classic beginner error.

---

## 4. What Is a Ranking Function?

A ranking function assigns an integer position to each row within its window partition, based on the window's `ORDER BY`. The three core functions — `ROW_NUMBER()`, `RANK()`, and `DENSE_RANK()` — differ only in how they treat rows whose ordering values are equal (ties / peers).

```sql
SELECT
  employee_name,
  department_id,
  salary,
  ROW_NUMBER() OVER (PARTITION BY department_id ORDER BY salary DESC) AS row_num,
  RANK()       OVER (PARTITION BY department_id ORDER BY salary DESC) AS rank_val,
  DENSE_RANK() OVER (PARTITION BY department_id ORDER BY salary DESC) AS dense_val
FROM employees;
```

Annotated syntax breakdown:

```
RANK() OVER (PARTITION BY department_id ORDER BY salary DESC)
│      │     │                          │              │
│      │     │                          │              └── DESC: highest salary gets rank 1
│      │     │                          │                  (ASC would give lowest salary rank 1)
│      │     │                          └── ORDER BY: defines rank order AND defines "ties"
│      │     │                              (rows with equal salary are peers → same rank)
│      │     └── PARTITION BY: reset ranking to 1 within each department
│      │         (omit it → one single ranking across the whole result set)
│      └── OVER: marks this as a window function; the (...) is the window definition
└── RANK: the ranking function. () is always empty for the three core rankers
          (they take no arguments — contrast NTILE(4) in Topic 35)
```

Key rules that fall directly out of the definition:

- **The `()` is always empty** for `ROW_NUMBER`, `RANK`, and `DENSE_RANK`. They take no arguments.
- **`ORDER BY` is mandatory** for a meaningful rank. Without it, the ranking is non-deterministic (all rows are technically "peers" — see 6.7).
- **`PARTITION BY` is optional.** Omit it and the whole result set is one partition.
- **Rank values start at 1**, never 0.
- **The output is always an integer** (`bigint` in PostgreSQL).

The precise per-function definitions:

| Function | Rule | Ties (peers) | Sequence when a 2-way tie occurs at position 3 |
|----------|------|--------------|-----------------------------------------------|
| `ROW_NUMBER()` | Unique sequential number per row | Broken arbitrarily; each peer gets a distinct number | 1, 2, 3, 4, 5 |
| `RANK()` | Position = (# rows strictly before) + 1 | Peers share the value; **gaps** follow | 1, 2, 3, 3, **5** |
| `DENSE_RANK()` | Position = (# distinct values before) + 1 | Peers share the value; **no gaps** | 1, 2, 3, 3, **4** |

---

## 5. Why Ranking Function Mastery Matters in Production

1. **"Top-N per group" is one of the most common analytical requirements that exists.** Top 3 products per category. Highest-paid employee per department. Most recent order per customer. The 5 slowest queries per endpoint. Every one of these is a ranking function followed by a filter. Doing it wrong (correlated subqueries, self-joins) is slower and buggier.

2. **Deduplication.** `ROW_NUMBER() OVER (PARTITION BY natural_key ORDER BY updated_at DESC)` with an outer `WHERE row_num = 1` is *the* canonical way to keep one row per logical entity — the latest version, the first occurrence, the winning record. Data pipelines run this constantly. Choosing RANK instead of ROW_NUMBER here silently returns duplicates when timestamps tie.

3. **The tie-handling bug is silent and data-dependent.** If you use `ROW_NUMBER` where you needed `RANK` (or vice versa), the query runs fine, returns plausible-looking numbers, and produces *wrong results only when a tie happens to occur in the data*. It passes on your test dataset (no ties), then breaks in production when two rows share a value. This is a classic "works on my machine" data bug that no type checker catches.

4. **Pagination correctness.** ROW_NUMBER-based pagination and keyset pagination both depend on a **deterministic total order**. If your `ORDER BY` isn't unique, rows can shift between pages — the same row appears on page 1 and page 2, or gets skipped entirely — because the sort order among ties is unstable. Adding a tiebreaker column is the fix.

5. **Cost.** A ranking function needs one sort of the input. A naive "top per group" written as a correlated subquery re-scans the inner table once per outer row — `O(N × M)`. The ranking version is `O(N log N)`. On a 10M-row table that's the difference between 200 ms and a query that never finishes.

---

## 6. Deep Technical Content

### 6.1 The Core Difference — Worked Numerically

This is the section to memorise. Consider a single department's salaries, sorted descending:

```
salary   ROW_NUMBER   RANK   DENSE_RANK
------   ----------   ----   ----------
200000        1          1        1
150000        2          2        2
150000        3          2        2      ← tie with the row above (peers)
150000        4          2        2      ← three-way tie, all salary=150000
120000        5          5        3      ← RANK jumps to 5 (skipped 3,4); DENSE goes to 3
 90000        6          6        4
 90000        7          6        4      ← another tie
 80000        8          8        5      ← RANK jumps to 8 (skipped 7); DENSE goes to 5
```

Read every column carefully:

- **ROW_NUMBER**: 1 through 8, always unique, always contiguous. The three rows with salary 150000 get 2, 3, 4 — the split between them is *arbitrary* (see 6.3).
- **RANK**: the three 150000 rows are all rank **2**. The next value (120000) is rank **5**, because *four* rows precede it (200000, and the three 150000s), so its position is 4+1 = 5. Ranks 3 and 4 are **skipped**. Then 90000 rows are rank 6 (5 precede). Then 80000 is rank 8 (7 precede) — rank 7 skipped.
- **DENSE_RANK**: the three 150000 rows are rank **2**. The next *distinct value* (120000) is rank **3**. No skips. It counts *distinct salary values seen so far*, plus one.

Mnemonic:
- **RANK = "how many people finished ahead of me, plus one."** Ties share; gaps appear.
- **DENSE_RANK = "how many distinct scores are better than or equal to mine."** Ties share; no gaps.
- **ROW_NUMBER = "just number the rows."** No sharing, no gaps, but arbitrary among ties.

An algebraic identity worth knowing: **when there are no ties in the `ORDER BY` values, `ROW_NUMBER = RANK = DENSE_RANK` for every row.** They only diverge in the presence of peers. Interviewers love asking "when are they the same?" — the answer is "when the ordering columns are unique across the partition."

### 6.2 What Exactly Counts as a Tie?

A tie (peer relationship) exists between two rows when **all** of the window's `ORDER BY` expressions evaluate to equal values for those rows. Consequences:

- `ORDER BY salary` — rows tie when salaries are equal.
- `ORDER BY salary, hire_date` — rows tie only when *both* salary and hire_date are equal. Adding columns to the window's ORDER BY *reduces* ties. This is how you break ties (see 6.6).
- Floating point: 12.40 and 12.400000001 are **not** equal, so not tied. Beware of computed/rounded ordering columns.
- Text ordering respects collation. `'a'` and `'A'` may or may not be peers depending on the collation's case sensitivity.

### 6.3 ROW_NUMBER's Non-Determinism Among Peers

`ROW_NUMBER` guarantees unique numbers, but when rows are peers, *which* peer gets the lower number is **undefined** unless you make the ordering total. The engine assigns numbers in whatever physical order the sort produced, which can change between executions, after a `VACUUM`, after a plan change, or across PostgreSQL versions.

```sql
-- Two employees, both salary 150000. Who is row_num 2 and who is 3?
-- UNDEFINED — could be either, could change between runs.
SELECT name, salary,
       ROW_NUMBER() OVER (ORDER BY salary DESC) AS rn
FROM employees;
```

This is the root of the "row jumps between pages" pagination bug. **Fix: always add a unique tiebreaker** so the order is total:

```sql
ROW_NUMBER() OVER (ORDER BY salary DESC, id ASC)  -- id makes the order deterministic
```

`id` (a primary key) is unique, so no two rows are ever peers — the ROW_NUMBER assignment becomes fully reproducible. This is covered again in 6.6 and is a non-negotiable production habit.

### 6.4 Top-N-Per-Group — The Canonical Pattern

The single most important application. "Top 3 highest-paid employees per department":

```sql
SELECT department_id, name, salary, rnk
FROM (
  SELECT
    department_id,
    name,
    salary,
    DENSE_RANK() OVER (PARTITION BY department_id ORDER BY salary DESC) AS rnk
  FROM employees
) ranked
WHERE rnk <= 3
ORDER BY department_id, rnk;
```

The choice of ranking function here is a *business decision*, and it is the crux of the interview trap:

- **`ROW_NUMBER() <= 3`** → *exactly 3 rows per department*, even if the 3rd and 4th employee earn the same salary (one is arbitrarily excluded). Use when you need a hard cap of N rows.
- **`RANK() <= 3`** → the top 3 *positions*; if two people tie for 1st, you get them both plus one 3rd-place... actually: tie for 1st gives ranks 1,1,3 — so you'd return 3 rows but "3rd place" (rank 3) and skip nothing at the cutoff. If three tie for 3rd (ranks 3,3,3) you return all of them → possibly more than 3 rows.
- **`DENSE_RANK() <= 3`** → the top 3 *distinct salary levels*; returns *everyone* earning one of the three highest salaries — could be many rows if there are ties.

Concrete: salaries 200k, 150k, 150k, 150k, 120k.
- `ROW_NUMBER() <= 3` → 200k, 150k, 150k (3 rows, one 150k arbitrarily dropped).
- `RANK() <= 3` → 200k (rank 1), 150k, 150k, 150k (all rank 2). 120k is rank 5, excluded. → **4 rows.**
- `DENSE_RANK() <= 3` → 200k (1), the three 150k (2), 120k (3). → **5 rows.**

There is no universally "correct" one. The interview-winning answer is: *"It depends on whether the requirement is a fixed row count (ROW_NUMBER), the top N competition positions (RANK), or the top N distinct value tiers (DENSE_RANK)."*

### 6.5 Deduplication With ROW_NUMBER

Keep only the latest row per key:

```sql
-- Keep the most recent order per customer, discard older duplicates
WITH ranked AS (
  SELECT
    o.*,
    ROW_NUMBER() OVER (
      PARTITION BY customer_id
      ORDER BY created_at DESC, id DESC   -- id DESC breaks timestamp ties
    ) AS rn
  FROM orders o
)
SELECT * FROM ranked WHERE rn = 1;
```

**Why ROW_NUMBER and not RANK here?** Because you need *exactly one* row per partition. If two orders share the same `created_at`, `RANK` gives them both rank 1 and `WHERE rank = 1` returns *both* — you get duplicates, defeating the purpose. `ROW_NUMBER` guarantees a single winner. The `id DESC` tiebreaker makes *which* winner deterministic. This is why deduplication is always ROW_NUMBER, never RANK.

### 6.6 Breaking Ties Deterministically

To make ranks reproducible and pagination-safe, extend the window's `ORDER BY` with enough columns to make the ordering **total** (no two rows ever tie). The final tiebreaker should be a unique column — typically the primary key.

```sql
-- Non-deterministic: many employees can share a salary
RANK() OVER (ORDER BY salary DESC)

-- Deterministic total order (salary, then hire_date, then unique id)
ROW_NUMBER() OVER (ORDER BY salary DESC, hire_date ASC, id ASC)
```

Note a subtlety: adding tiebreakers to `RANK`/`DENSE_RANK` changes their *behaviour*, not just their determinism. Once `id` makes every row unique, there are no peers left, so `RANK` and `DENSE_RANK` collapse into `ROW_NUMBER` — every value becomes 1,2,3,... with no shared ranks and no gaps. If you genuinely want tie *sharing* (two people are "both 2nd"), you must **not** add a unique tiebreaker to the window ORDER BY. Tiebreakers are for ROW_NUMBER-style total ordering; they defeat the entire point of RANK.

The resolution: if you want *shared ranks in the value* but *deterministic display order*, rank without the tiebreaker but add the tiebreaker to the outer query's ORDER BY:

```sql
SELECT name, salary,
       RANK() OVER (ORDER BY salary DESC) AS rnk   -- ties share rank
FROM employees
ORDER BY rnk, id;                                   -- deterministic display among peers
```

### 6.7 Ranking Without ORDER BY (Degenerate Case)

If you omit `ORDER BY` inside `OVER()`, every row in the partition is a peer of every other row (no ordering means nothing is "before" anything).

```sql
SELECT name,
       ROW_NUMBER() OVER () AS rn,   -- 1,2,3,... in arbitrary physical order
       RANK()       OVER () AS rk,   -- ALL rows get rank 1 (all are peers)
       DENSE_RANK() OVER () AS dr    -- ALL rows get 1
FROM employees;
```

- `ROW_NUMBER() OVER ()` still numbers 1..N (arbitrary but unique) — occasionally used to attach a meaningless sequence.
- `RANK() OVER ()` and `DENSE_RANK() OVER ()` give **every row the value 1**, because with no ordering, all rows tie. This is almost never useful and is usually a bug — someone forgot the ORDER BY.

### 6.8 NULL Handling in the Ordering Column

Ranking functions rank NULLs like any other value, using the window ORDER BY's NULL placement rules (Topic 33). By default in PostgreSQL, `ORDER BY col ASC` puts NULLs **last**, and `ORDER BY col DESC` puts NULLs **first**. All NULLs are considered equal *to each other* for peer purposes (unlike NULL = NULL in WHERE, which is UNKNOWN — window ordering treats NULLs as tied).

```sql
SELECT name, commission,
       RANK() OVER (ORDER BY commission DESC NULLS LAST) AS r
FROM employees;
-- All rows with commission = NULL are peers → share the same rank, placed last
```

Control it explicitly with `NULLS FIRST` / `NULLS LAST` in the window ORDER BY. If your ranking of "top salespeople by commission" unexpectedly puts the NULL-commission rows at rank 1, it's because `DESC` defaults to `NULLS FIRST`. Add `NULLS LAST`.

### 6.9 PARTITION BY Resets Everything

Each partition is ranked independently, counters reset to 1 at every partition boundary (Topic 33). A row's rank is *relative to its partition only*, never global.

```sql
-- Rank within department AND globally, side by side
SELECT
  name, department_id, salary,
  RANK() OVER (PARTITION BY department_id ORDER BY salary DESC) AS dept_rank,
  RANK() OVER (ORDER BY salary DESC)                           AS company_rank
FROM employees;
-- dept_rank restarts at 1 in each department; company_rank is one global ranking
```

You can have multiple window functions with *different* windows in the same SELECT — the engine may need multiple sorts (or reuse a sort if windows are compatible; see EXPLAIN section).

### 6.10 Combining Ranking Functions — Detecting Ties Programmatically

A neat trick: `RANK() = ROW_NUMBER()` for a row exactly when that row is *not* tied with any earlier peer. And `COUNT(*) OVER (PARTITION BY ... ORDER BY same_cols)` differences reveal peer group sizes. More directly, you can detect whether ties exist:

```sql
SELECT name, salary,
       RANK()       OVER (ORDER BY salary DESC) AS rk,
       DENSE_RANK() OVER (ORDER BY salary DESC) AS dr,
       ROW_NUMBER() OVER (ORDER BY salary DESC) AS rn,
       -- if rk <> rn, this row is part of a tie group (not the first member)
       (RANK() OVER (ORDER BY salary DESC) <> ROW_NUMBER() OVER (ORDER BY salary DESC)) AS is_tied
FROM employees;
```

The gap between the maximum `RANK` and maximum `DENSE_RANK` tells you how many tie-induced skips occurred overall.

### 6.11 Ranking Over Aggregated Data

Window functions run *after* GROUP BY, so you can rank aggregates. This is extremely common — "rank categories by total revenue":

```sql
SELECT
  category_id,
  SUM(revenue) AS total_revenue,
  RANK() OVER (ORDER BY SUM(revenue) DESC) AS revenue_rank
FROM sales
GROUP BY category_id;
```

Note `SUM(revenue)` appears *both* as a grouped aggregate and *inside* the window's ORDER BY. This is legal and correct: GROUP BY collapses rows to one per category, then the window function ranks those grouped rows. You do not (and cannot) put the SUM in a nested aggregate — the window sees the already-aggregated output.

### 6.12 Ranking is Not Filterable Inline — The Subquery Wrapper (recap)

Reiterating the execution-order rule from Section 3 because it dominates real code: you *cannot* say `WHERE RANK() ... <= 3` or `HAVING RANK() ... <= 3`. You must materialise the rank in a subquery/CTE and filter in the outer query. Every top-N query has this two-level shape. There is no way around it — it is a direct consequence of window functions being computed in the SELECT phase, after WHERE and HAVING.

---

## 7. EXPLAIN — Ranking Functions in the Plan

The signature of a ranking function in a plan is the **WindowAgg** node, almost always sitting directly on top of a **Sort** node (or an Index Scan if the sort is free).

### Case 1: WindowAgg over a Sort (no useful index)

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT department_id, name, salary,
       RANK() OVER (PARTITION BY department_id ORDER BY salary DESC) AS r
FROM employees;
```

```
WindowAgg  (cost=9857.32..11357.32 rows=100000 width=48)
           (actual time=142.1..268.4 rows=100000 loops=1)
  ->  Sort  (cost=9857.32..10107.32 rows=100000 width=40)
            (actual time=142.0..178.6 rows=100000 loops=1)
        Sort Key: department_id, salary DESC
        Sort Method: external merge  Disk: 5416kB
        ->  Seq Scan on employees  (cost=0.00..1637.00 rows=100000 width=40)
                                    (actual time=0.02..22.1 rows=100000 loops=1)
Buffers: shared hit=637, temp read=677 written=679
Planning Time: 0.2 ms
Execution Time: 288.9 ms
```

**Reading it:**
- `Seq Scan on employees` — reads all 100K rows (no WHERE, so full scan).
- `Sort Key: department_id, salary DESC` — the sort the WindowAgg requires; note it matches `PARTITION BY department_id ORDER BY salary DESC`. Partition columns sort first, then the order columns.
- `Sort Method: external merge  Disk: 5416kB` — **the sort spilled to disk** because 100K rows exceeded `work_mem`. This is the bottleneck (`temp read/written` buffers confirm it). Raising `work_mem` to hold the sort in memory would flip this to `Sort Method: quicksort  Memory: NkB` and cut execution time substantially.
- `WindowAgg` — the single streaming pass that assigns ranks. Its own cost above the Sort is small (`11357 - 10107`); nearly all the time is the sort.

### Case 2: WindowAgg over an Index Scan (sort eliminated)

With a matching index, the sort vanishes:

```sql
CREATE INDEX idx_emp_dept_salary ON employees (department_id, salary DESC);

EXPLAIN (ANALYZE, BUFFERS)
SELECT department_id, name, salary,
       RANK() OVER (PARTITION BY department_id ORDER BY salary DESC) AS r
FROM employees;
```

```
WindowAgg  (cost=0.42..6789.42 rows=100000 width=48)
           (actual time=0.05..96.3 rows=100000 loops=1)
  ->  Index Scan using idx_emp_dept_salary on employees
        (cost=0.42..5289.42 rows=100000 width=40)
        (actual time=0.03..48.1 rows=100000 loops=1)
Buffers: shared hit=100234
Planning Time: 0.3 ms
Execution Time: 112.4 ms
```

**Reading it:**
- **No Sort node.** The `Index Scan using idx_emp_dept_salary` delivers rows already ordered by `(department_id, salary DESC)` — exactly what the WindowAgg needs. `Sort Method` and `temp` buffers are gone.
- Execution dropped from ~289 ms to ~112 ms, and it's now stable regardless of `work_mem`.
- Trade-off: the Index Scan touches many buffers (`shared hit=100234`) because it visits the heap for every row (the index doesn't cover `name`/`salary` for output). For a pure ordered scan this is still typically a win over sort-spill. An index-only scan (covering index) would reduce buffer traffic further.

### Case 3: Top-N with the outer filter

```sql
EXPLAIN (ANALYZE)
SELECT * FROM (
  SELECT department_id, name, salary,
         ROW_NUMBER() OVER (PARTITION BY department_id ORDER BY salary DESC) AS rn
  FROM employees
) t
WHERE rn <= 3;
```

```
Subquery Scan on t  (cost=0.42..7539.42 rows=33333 width=48)
                    (actual time=0.06..104.2 rows=142 loops=1)
  Filter: (t.rn <= 3)
  Rows Removed by Filter: 99858
  ->  WindowAgg  (cost=0.42..6789.42 rows=100000 width=48)
                 (actual time=0.05..98.9 rows=100000 loops=1)
        ->  Index Scan using idx_emp_dept_salary on employees ...
```

**Reading it:**
- The `Filter: (t.rn <= 3)` runs in the outer `Subquery Scan`, *after* WindowAgg computed all 100K ranks. `Rows Removed by Filter: 99858` — note that **all 100K rows were ranked before filtering**. Window functions cannot short-circuit; the filter is a post-pass. This is the inherent cost of top-N-per-group via ranking. (For extreme top-1 cases, `DISTINCT ON` or a lateral join can sometimes beat it — see Performance section.)

### What to Look For

| Symptom | Meaning | Fix |
|---------|---------|-----|
| `Sort Method: external merge Disk: …` under WindowAgg | Sort spilled to disk | Raise `work_mem`, or add index matching `(PARTITION BY, ORDER BY)` |
| WindowAgg with no Sort child | Index provides ordering — optimal | Nothing; this is the goal |
| Multiple WindowAgg nodes stacked | Several windows with incompatible orderings → multiple sorts | Align window ORDER BYs where possible |
| `Rows Removed by Filter` ≈ total rows | Top-N ranked everything then discarded most | Expected; consider `DISTINCT ON` for top-1 |

---

## 8. Query Examples

### Example 1 — Basic: Rank Employees by Salary

```sql
-- Show all three rankings side by side so the tie behaviour is visible
SELECT
  name,
  salary,
  ROW_NUMBER() OVER (ORDER BY salary DESC) AS row_number,   -- unique 1..N
  RANK()       OVER (ORDER BY salary DESC) AS rank,          -- ties share, gaps after
  DENSE_RANK() OVER (ORDER BY salary DESC) AS dense_rank     -- ties share, no gaps
FROM employees
ORDER BY salary DESC;
```

### Example 2 — Intermediate: Top 3 Products Per Category

```sql
-- Highest-selling products in each category.
-- DENSE_RANK: include ALL products tied at each of the top 3 sales levels.
SELECT category_id, product_name, units_sold, sales_rank
FROM (
  SELECT
    p.category_id,
    p.name AS product_name,
    SUM(oi.quantity) AS units_sold,
    DENSE_RANK() OVER (
      PARTITION BY p.category_id
      ORDER BY SUM(oi.quantity) DESC
    ) AS sales_rank
  FROM products p
  INNER JOIN order_items oi ON oi.product_id = p.id
  INNER JOIN orders o       ON o.id = oi.order_id
  WHERE o.status = 'completed'
  GROUP BY p.category_id, p.name        -- aggregate first, window ranks the groups
) ranked
WHERE sales_rank <= 3                    -- filter must be in the outer query
ORDER BY category_id, sales_rank, units_sold DESC;
```

### Example 3 — Production Grade: Deduplicate the Latest Order Per Customer

**Context:** `orders` table, 20M rows, ~2M distinct customers. A data-sync process occasionally inserts duplicate order rows for the same `(customer_id, external_ref)`; we need the single freshest row per customer for a dashboard. Index available: `idx_orders_cust_created ON orders (customer_id, created_at DESC, id DESC)`. Expectation: with the covering-ish index feeding the WindowAgg, the sort is eliminated and this runs in a few hundred ms despite 20M rows; the dominant cost is the index scan itself.

```sql
-- Keep exactly ONE row per customer: the most recent order.
-- ROW_NUMBER (not RANK) guarantees a single winner even when created_at ties.
WITH ranked_orders AS (
  SELECT
    o.id,
    o.customer_id,
    o.total_amount,
    o.status,
    o.created_at,
    ROW_NUMBER() OVER (
      PARTITION BY o.customer_id
      ORDER BY o.created_at DESC, o.id DESC   -- id DESC = deterministic tiebreak
    ) AS rn
  FROM orders o
  WHERE o.created_at >= NOW() - INTERVAL '2 years'
)
SELECT id, customer_id, total_amount, status, created_at
FROM ranked_orders
WHERE rn = 1;
```

```sql
EXPLAIN (ANALYZE, BUFFERS)  -- abbreviated
```
```
Subquery Scan on ranked_orders  (actual time=0.07..612.3 rows=1980342 loops=1)
  Filter: (ranked_orders.rn = 1)
  Rows Removed by Filter: 4820551
  ->  WindowAgg  (actual time=0.06..540.9 rows=6800893 loops=1)
        ->  Index Scan using idx_orders_cust_created on orders o
              Index Cond: (created_at >= (now() - '2 years'::interval))
              (actual time=0.04..300.2 rows=6800893 loops=1)
Buffers: shared hit=402118 read=51002
Execution Time: 690.1 ms
```

The Index Scan feeds rows pre-sorted by `(customer_id, created_at DESC, id DESC)`, so there is **no Sort node** — the WindowAgg just streams. All 6.8M in-window rows are ranked; the outer filter keeps the ~2M `rn = 1` rows. Without the index, a sort of 6.8M rows would spill heavily to disk and this could take 5–10× longer.

---

## 9. Wrong → Right Patterns

### Wrong 1: Using RANK for deduplication (silent duplicates)

```sql
-- WRONG: intends "one latest order per customer"
WITH r AS (
  SELECT o.*, RANK() OVER (PARTITION BY customer_id ORDER BY created_at DESC) AS rk
  FROM orders o
)
SELECT * FROM r WHERE rk = 1;
-- BUG: if two orders share the same created_at, BOTH get rk = 1.
-- WHERE rk = 1 returns BOTH → duplicates survive. Dedup fails silently
-- exactly when timestamps collide (common with batch inserts).
```

```sql
-- RIGHT: ROW_NUMBER guarantees a single winner per partition
WITH r AS (
  SELECT o.*,
    ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY created_at DESC, id DESC) AS rn
  FROM orders o
)
SELECT * FROM r WHERE rn = 1;
-- ROW_NUMBER never shares; id DESC makes the winner deterministic.
```

### Wrong 2: Filtering the rank in WHERE

```sql
-- WRONG: window functions are computed AFTER WHERE
SELECT name, salary, RANK() OVER (ORDER BY salary DESC) AS r
FROM employees
WHERE RANK() OVER (ORDER BY salary DESC) <= 10;
-- ERROR:  window functions are not allowed in WHERE
-- LINE ...: WHERE RANK() OVER (ORDER BY salary DESC) <= 10
```

```sql
-- RIGHT: compute in a subquery, filter outside
SELECT name, salary, r
FROM (
  SELECT name, salary, RANK() OVER (ORDER BY salary DESC) AS r
  FROM employees
) t
WHERE r <= 10
ORDER BY r;
```

### Wrong 3: ROW_NUMBER where the requirement is "top N levels" (dropping tied rows)

```sql
-- WRONG: "show employees earning one of the top 3 salaries in the company"
SELECT name, salary
FROM (
  SELECT name, salary, ROW_NUMBER() OVER (ORDER BY salary DESC) AS rn
  FROM employees
) t
WHERE rn <= 3;
-- BUG: if salaries are 500k, 500k, 500k, 400k, ... ROW_NUMBER gives 1,2,3,4...
-- so it returns three of the four 500k earners and drops the rest — arbitrarily
-- excluding people who earn the SAME top salary. Business-wrong.
```

```sql
-- RIGHT: DENSE_RANK returns everyone at each of the top 3 distinct salary levels
SELECT name, salary
FROM (
  SELECT name, salary, DENSE_RANK() OVER (ORDER BY salary DESC) AS dr
  FROM employees
) t
WHERE dr <= 3
ORDER BY salary DESC;
-- All 500k earners appear (dr=1), then all at the 2nd-highest, then the 3rd.
```

### Wrong 4: Non-deterministic pagination via ROW_NUMBER on a non-unique order

```sql
-- WRONG: page through orders 20 at a time, ordered by amount only
SELECT * FROM (
  SELECT o.*, ROW_NUMBER() OVER (ORDER BY total_amount DESC) AS rn
  FROM orders o
) t
WHERE rn BETWEEN 21 AND 40;
-- BUG: many orders share total_amount. The order AMONG equal amounts is
-- undefined and can change between the page-1 and page-2 queries → the same
-- row appears on two pages, or a row is skipped entirely.
```

```sql
-- RIGHT: make the ordering total with a unique tiebreaker
SELECT * FROM (
  SELECT o.*, ROW_NUMBER() OVER (ORDER BY total_amount DESC, id ASC) AS rn
  FROM orders o
) t
WHERE rn BETWEEN 21 AND 40
ORDER BY rn;
-- id ASC guarantees a stable, reproducible total order across all pages.
```

### Wrong 5: Expecting RANK/DENSE_RANK to differ when the order is unique

```sql
-- WRONG expectation: developer adds id to break ties, then wonders why
-- RANK() shows no shared ranks
SELECT name, salary,
       RANK() OVER (ORDER BY salary DESC, id ASC) AS r  -- always 1,2,3,4...
FROM employees;
-- "Why does RANK never repeat?" Because (salary, id) is UNIQUE → no peers exist
-- → RANK degenerates to ROW_NUMBER. This isn't a bug, but it defeats the purpose
-- of RANK if you wanted tied employees to share a rank.
```

```sql
-- RIGHT: to keep shared ranks, DON'T put a unique column in the window ORDER BY.
-- Break display ties in the OUTER order by instead.
SELECT name, salary,
       RANK() OVER (ORDER BY salary DESC) AS r  -- equal salaries share a rank
FROM employees
ORDER BY r, id;                                  -- deterministic display order
```

---

## 10. Performance Profile

### The Cost Model

A ranking query's cost is essentially:

```
cost ≈ input_scan + sort(input) + O(N) window pass
```

The window pass itself is cheap and linear — ranking maintains only a handful of counters. **The sort dominates**, so all optimisation focuses on making the sort cheap or eliminating it.

### Memory

- **WindowAgg state:** trivial — a few `bigint` counters per active partition. Ranking functions do **not** buffer partitions (unlike frame-dependent functions such as some uses of `LAG`/aggregate windows). Memory is not driven by partition size.
- **Sort:** driven by `work_mem`. If the sorted set fits, `Sort Method: quicksort Memory: NkB` — fast. If not, `Sort Method: external merge Disk: NkB` — the set spills to temp files and is merge-sorted, several times slower. This is the number-one performance issue for ranking.

### Scaling

| Rows | No index (sort in memory) | No index (sort spills) | Index provides order (no sort) |
|------|--------------------------|------------------------|-------------------------------|
| 1M | ~0.3–0.8 s | ~1–3 s | ~0.1–0.4 s |
| 10M | (rarely fits) | ~10–40 s | ~1–4 s |
| 100M | — | minutes; heavy temp I/O | ~15–60 s (index scan bound) |

The message: at 10M+ rows, an index matching `(PARTITION BY cols, ORDER BY cols)` is the difference between seconds and minutes, because it removes the sort entirely (as shown in EXPLAIN Case 2).

### Index Interaction — The Key Optimisation

Create an index whose column order and direction match the window:

```sql
-- For: RANK() OVER (PARTITION BY department_id ORDER BY salary DESC)
CREATE INDEX idx_emp_dept_sal ON employees (department_id, salary DESC);
```

Rules:
- **Partition columns first, then order columns**, in that sequence.
- **Match sort direction** (`DESC` in the index if the window uses `DESC`) — though PostgreSQL can scan a B-tree backwards, matching direction avoids ambiguity when there are multiple order keys with mixed directions.
- Include the tiebreaker column if you use one in the window ORDER BY.
- Consider a **covering index** (`INCLUDE (...)`) so the WindowAgg can use an index-only scan and skip heap visits — reduces buffer traffic for wide result rows.

### Top-1-Per-Group: When to Skip Ranking Entirely

For the special case of **exactly one row per group** (top-1), PostgreSQL's `DISTINCT ON` is often faster and cleaner than `ROW_NUMBER() ... WHERE rn = 1`, because it can stop at the first row of each group rather than ranking every row:

```sql
-- Latest order per customer, no window function
SELECT DISTINCT ON (customer_id) *
FROM orders
ORDER BY customer_id, created_at DESC, id DESC;
```

`DISTINCT ON` still needs the sort, but avoids materialising a rank column and the outer filter pass. For top-1 it's idiomatic PostgreSQL. For top-N (N > 1), ranking functions are the right tool — `DISTINCT ON` can't do top-3. (A lateral join with `LIMIT N` per group can beat ranking for small N on a huge table when a good index exists, but it's more complex.)

### CPU

Ranking is CPU-light per row (integer increments and equality checks on the order key to detect peers). The equality check on peers has a cost proportional to the width of the ORDER BY columns — ranking on a long text column costs more per comparison than on an integer. Sorting cost likewise scales with key width. Prefer narrow, indexed ordering keys.

---

## 11. Node.js Integration

### 11.1 Top-N per group with pg

```javascript
import pg from 'pg';
const { Pool } = pg;
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

// Top N highest-paid employees per department.
// $1 is the cutoff — parameterised, never string-interpolated.
async function topEarnersPerDepartment(topN = 3) {
  const { rows } = await pool.query(
    `SELECT department_id, name, salary, dept_rank
     FROM (
       SELECT
         department_id,
         name,
         salary,
         DENSE_RANK() OVER (
           PARTITION BY department_id
           ORDER BY salary DESC
         ) AS dept_rank
       FROM employees
     ) ranked
     WHERE dept_rank <= $1
     ORDER BY department_id, dept_rank, name`,
    [topN]
  );
  return rows;
}
```

### 11.2 Deduplication — latest row per key

```javascript
// Return the single most recent order per customer (dedup).
// ROW_NUMBER + id tiebreak = exactly one deterministic winner each.
async function latestOrderPerCustomer() {
  const { rows } = await pool.query(
    `WITH ranked AS (
       SELECT
         o.*,
         ROW_NUMBER() OVER (
           PARTITION BY o.customer_id
           ORDER BY o.created_at DESC, o.id DESC
         ) AS rn
       FROM orders o
     )
     SELECT * FROM ranked WHERE rn = 1`
  );
  return rows;
}
```

### 11.3 Keyset-style ranked pagination

```javascript
// Page through a global ranking. rn is deterministic because the window
// ORDER BY (total_amount DESC, id ASC) is a total order.
async function rankedOrdersPage(page = 1, pageSize = 20) {
  const startRank = (page - 1) * pageSize + 1;
  const endRank = page * pageSize;
  const { rows } = await pool.query(
    `SELECT id, customer_id, total_amount, rn
     FROM (
       SELECT id, customer_id, total_amount,
              ROW_NUMBER() OVER (ORDER BY total_amount DESC, id ASC) AS rn
       FROM orders
       WHERE status = 'completed'
     ) t
     WHERE rn BETWEEN $1 AND $2
     ORDER BY rn`,
    [startRank, endRank]
  );
  return rows;
}
```

### 11.4 A note on the driver

`pg` returns the rank column as a JavaScript **string**, not a number, because PostgreSQL ranking functions return `bigint` and node-postgres maps `bigint` to string to avoid precision loss beyond `Number.MAX_SAFE_INTEGER`. Cast in JS (`Number(row.rn)`) if you need arithmetic, or `rank_val::int` in SQL when you know the values are small:

```javascript
const rank = Number(rows[0].dept_rank);  // "3" -> 3
```

No ORM is required for any of this — ranking functions are plain SQL. The driver's only job is parameter binding (`$1`, `$2`) and returning rows.

---

## 12. ORM Comparison

Ranking / window functions are the area where ORMs are weakest. None of the major Node.js ORMs models them as first-class query-builder methods with full type safety; most require raw SQL fragments or a full raw query.

### Prisma

**Can Prisma do ranking functions?** — Not through its typed query API. Prisma has no `ROW_NUMBER`/`RANK`/`DENSE_RANK` builder. You must drop to `$queryRaw`.

```typescript
import { PrismaClient, Prisma } from '@prisma/client';
const prisma = new PrismaClient();

const topEarners = await prisma.$queryRaw<Array<{
  department_id: number; name: string; salary: number; dept_rank: number;
}>>(Prisma.sql`
  SELECT department_id, name, salary, dept_rank FROM (
    SELECT department_id, name, salary,
      DENSE_RANK() OVER (PARTITION BY department_id ORDER BY salary DESC) AS dept_rank
    FROM employees
  ) t
  WHERE dept_rank <= ${3}
  ORDER BY department_id, dept_rank
`);
```

**Where it breaks:** no type inference on the window columns; `dept_rank` comes back as `bigint`/string and you type it manually. **Verdict:** raw SQL only; Prisma treats ranking as out of scope.

### Drizzle ORM

**Can Drizzle do ranking functions?** — Yes, and it's the best of the group. Drizzle exposes window helpers.

```typescript
import { db } from './db';
import { employees } from './schema';
import { sql, desc } from 'drizzle-orm';

const ranked = db.$with('ranked').as(
  db.select({
    departmentId: employees.departmentId,
    name: employees.name,
    salary: employees.salary,
    deptRank: sql<number>`dense_rank() OVER (
      PARTITION BY ${employees.departmentId} ORDER BY ${employees.salary} DESC
    )`.as('dept_rank'),
  }).from(employees)
);

const rows = await db.with(ranked)
  .select().from(ranked)
  .where(sql`${ranked.deptRank} <= 3`);
```

Drizzle also has typed `rank()`, `denseRank()`, `rowNumber()` functions in newer versions usable inside `.select()` with an `.over()` builder. **Where it breaks:** the `.over()` builder covers common cases; exotic frame clauses still fall back to `sql`. **Verdict:** best window support; near-SQL clarity with typing.

### Sequelize

**Can Sequelize do ranking functions?** — Only via `sequelize.literal()` fragments or a full raw query. No first-class window API.

```javascript
const { QueryTypes } = require('sequelize');
const rows = await sequelize.query(
  `SELECT * FROM (
     SELECT department_id, name, salary,
       ROW_NUMBER() OVER (PARTITION BY department_id ORDER BY salary DESC) AS rn
     FROM employees
   ) t WHERE rn <= :n`,
  { replacements: { n: 3 }, type: QueryTypes.SELECT }
);
```

Embedding `literal('ROW_NUMBER() OVER (...)')` in `attributes` is possible but ugly and unsafe (no escaping of the fragment). **Verdict:** use `sequelize.query()`.

### TypeORM

**Can TypeORM do ranking functions?** — Via `addSelect()` with a raw window expression in the QueryBuilder, then `getRawMany()`.

```typescript
const rows = await dataSource.getRepository(Employee)
  .createQueryBuilder('e')
  .select('e.department_id', 'department_id')
  .addSelect('e.name', 'name')
  .addSelect('e.salary', 'salary')
  .addSelect(
    'DENSE_RANK() OVER (PARTITION BY e.department_id ORDER BY e.salary DESC)',
    'dept_rank'
  )
  .getRawMany();
// Filtering rank requires wrapping this as a subquery — TypeORM makes that awkward.
```

**Where it breaks:** you cannot filter on the window column in the same builder (`WHERE` runs first); you must nest via a subquery builder, which is verbose. **Verdict:** works for producing the rank; filtering top-N is clumsy — often cleaner as `dataSource.query()`.

### Knex.js

**Can Knex do ranking functions?** — Yes, cleanly, via `knex.raw()` in `select`, and it composes well with a wrapping query for the filter.

```javascript
const ranked = knex('employees')
  .select('department_id', 'name', 'salary')
  .select(knex.raw(
    'DENSE_RANK() OVER (PARTITION BY department_id ORDER BY salary DESC) AS dept_rank'
  ));

const rows = await knex.select('*').from(ranked.as('t'))
  .where('dept_rank', '<=', 3)
  .orderBy(['department_id', 'dept_rank']);
```

**Where it breaks:** the window expression itself is a raw string (no column typing). But the composition (`ranked.as('t')` then filter) is natural. **Verdict:** most transparent; the subquery-wrap pattern reads clearly.

### Summary Table

| ORM | Window API? | Ranking approach | Filter top-N | Verdict |
|-----|-------------|------------------|--------------|---------|
| Prisma | No | `$queryRaw` only | in raw SQL | Raw SQL |
| Drizzle | Yes (`.over()`, `sql`) | typed helpers or `sql` | `.with()` + `.where()` | Best support |
| Sequelize | No | `sequelize.query()` | in raw SQL | Raw query |
| TypeORM | Partial | `addSelect` raw + `getRawMany` | nested builder (awkward) | Works, clumsy filter |
| Knex | Partial | `knex.raw()` in select | wrap + `.where()` | Most transparent |

The consistent lesson: window/ranking functions live at the edge of ORM capability. Know the raw SQL — you will write it directly in every ORM.

---

## 13. Practice Exercises

### Exercise 1 — Basic

Given `employees(id, name, department_id, salary)`:

Write one query that returns each employee's `name`, `salary`, and three columns — `rownum`, `rnk`, `dense` — showing `ROW_NUMBER`, `RANK`, and `DENSE_RANK` over all employees ordered by salary descending. Then, in a comment, state what the three columns show for a set of salaries `100, 90, 90, 80`.

```sql
-- Write your query here
```

### Exercise 2 — Medium (combines Topic 20 GROUP BY + Topic 33 windows)

Given `orders(id, customer_id, total_amount, status, created_at)` and `order_items(id, order_id, product_id, quantity, unit_price)`:

For completed orders in the last 12 months, rank **customers** by their total spend within each **signup month** (assume `customers(id, name, created_at)` where `created_at` is signup time). Return `signup_month`, `customer_name`, `total_spend`, and `spend_rank_in_month`. Only include the top 5 spenders per signup month. Use the ranking function that returns *exactly* 5 rows per month even under ties, and justify the choice in a comment.

```sql
-- Write your query here
```

### Exercise 3 — Hard (production simulation; naive answer is wrong)

Given `sessions(id, user_id, device, started_at, ended_at)`, a data-quality process has inserted duplicate session rows: the same `(user_id, started_at)` sometimes appears 2–3 times with different `id`s and slightly different `ended_at`. 

Write a query that returns exactly one row per `(user_id, started_at)` — the one with the **latest** `ended_at`, breaking any remaining ties by the highest `id`. The naive answer using `RANK()` returns duplicates when `ended_at` also ties; avoid that. Then explain in a comment why `GROUP BY user_id, started_at` with `MAX(ended_at)` does *not* solve this correctly.

```sql
-- Write your query here
```

### Exercise 4 — Interview Simulation

You are given `products(id, name, category_id, price)`. The interviewer says: *"Return, for each category, the products with the top 3 highest prices. If several products tie for a price, they should all be shown, and a tie for a top price should not cause a lower-priced product to be excluded from a lower slot."*

Decide which ranking function satisfies this exact wording, write the query, and be ready to explain what would change if the requirement were instead "at most 3 products per category, period."

```sql
-- Write your query here
```

---

## 14. Interview Questions

### Q1: What is the difference between ROW_NUMBER, RANK, and DENSE_RANK?

**Junior answer:** "ROW_NUMBER numbers the rows, RANK ranks them, and DENSE_RANK is like RANK but denser." (Correct vibe, no precision — fails to explain ties, which is the entire point.)

**Principal answer:** "All three assign an integer per row within a window partition, ordered by the window's ORDER BY. They differ *only* in how they handle rows whose ordering values are equal — peers. ROW_NUMBER ignores ties: every row gets a unique, contiguous number 1..N, and the split among peers is arbitrary unless the order is made total with a unique tiebreaker. RANK gives peers the same value, then skips the next values so the following rank equals the count of rows before it plus one — producing gaps: 1,2,2,4. DENSE_RANK gives peers the same value but never skips — 1,2,2,3. A crucial identity: when the ORDER BY is unique across the partition, all three produce identical output; they diverge only when ties exist. So the function choice is really a decision about tie semantics: fixed row count (ROW_NUMBER), competition positions with gaps (RANK), or distinct value tiers with no gaps (DENSE_RANK)."

**Follow-up:** *"You have salaries 500, 500, 400, 300. Give me all three functions' output."* → ROW_NUMBER: 1,2,3,4. RANK: 1,1,3,4. DENSE_RANK: 1,1,2,3.

### Q2: Write "top 3 salaries per department." When would you use ROW_NUMBER vs DENSE_RANK, and how do the row counts differ?

**Junior answer:** "Use ROW_NUMBER with a subquery and WHERE rn <= 3." (Works, but misses that the choice changes results under ties.)

**Principal answer:** "The structure is a subquery/CTE computing the ranking, filtered in an outer query — you can't filter a window function in WHERE because it's evaluated after WHERE. The function choice depends on the requirement: `ROW_NUMBER() <= 3` returns *exactly* 3 rows per department, arbitrarily dropping one of two employees tied at the 3rd salary — good when you need a hard cap. `DENSE_RANK() <= 3` returns everyone in the top 3 *distinct* salary levels — could be many rows if there are ties, and never arbitrarily excludes an equal-salary peer. `RANK() <= 3` returns the top-3 competition positions and can also exceed 3 rows on ties but skips levels after a tie. If the spec says 'the three highest pay grades, include all ties,' it's DENSE_RANK. If it says 'at most three rows,' it's ROW_NUMBER with a deterministic tiebreaker."

**Follow-up:** *"Your dashboard must show at most 3 rows per department but the choice among equal-salary employees keeps flipping between deploys. Why, and fix it."* → ROW_NUMBER among peers is non-deterministic; add a unique tiebreaker to the window ORDER BY, e.g. `ORDER BY salary DESC, id ASC`.

### Q3: You deduplicate with `RANK() OVER (PARTITION BY key ORDER BY updated_at DESC)` and `WHERE rank = 1`, but you still get duplicates. Why?

**Junior answer:** "Maybe the partition key is wrong." (Possible, but not the systematic cause.)

**Principal answer:** "RANK gives *all* peers the same rank. When two rows share the same `updated_at`, both are rank 1, and `WHERE rank = 1` keeps both — so duplicates survive precisely when the ordering column ties, which is exactly the collision case dedup is meant to handle. The fix is ROW_NUMBER, which guarantees a single row per partition gets value 1, plus a unique tiebreaker (`ORDER BY updated_at DESC, id DESC`) so *which* row wins is deterministic and reproducible. RANK is never correct for deduplication; ROW_NUMBER always is."

**Follow-up:** *"Is `DISTINCT ON` a better fit here?"* → For top-1 per key, yes — `SELECT DISTINCT ON (key) * ... ORDER BY key, updated_at DESC, id DESC` is idiomatic PostgreSQL, avoids the rank column and outer filter, and can stop at the first row per group. For top-N (N>1) you still need ROW_NUMBER.

### Q4: Why can't you put `WHERE RANK() OVER (...) <= 10` in a query, and what does that tell you about execution order?

**Junior answer:** "SQL doesn't allow functions in WHERE." (Wrong — plenty of functions work in WHERE.)

**Principal answer:** "Window functions, including ranking, are computed in the SELECT phase, which runs *after* FROM, WHERE, GROUP BY, and HAVING. At the time WHERE is evaluated, the rank simply doesn't exist yet — so referencing it is a logical impossibility, and PostgreSQL raises 'window functions are not allowed in WHERE.' The consequence is the universal two-level pattern: compute the rank in an inner subquery/CTE where it becomes an ordinary column, then filter it in an outer query. It also explains why WHERE filters the *input* to the ranking (rows removed before ranking never consume a rank number), which is often exactly what you want but occasionally surprising."

**Follow-up:** *"Does the same restriction apply to GROUP BY and HAVING?"* → Yes for referencing the window value (both run before SELECT). You also cannot nest a window function inside an aggregate. You *can* rank aggregates, because the window runs after GROUP BY: `RANK() OVER (ORDER BY SUM(x) DESC)`.

---

## 15. Mental Model Checkpoint

1. A partition has values (sorted desc): 90, 90, 90, 80, 70, 70, 60. Write out ROW_NUMBER, RANK, and DENSE_RANK for all seven rows. Where exactly do the gaps appear in RANK, and why is the value after the first tie group `4`?

2. You run `SELECT name, ROW_NUMBER() OVER (ORDER BY salary DESC) FROM employees` twice and two employees with identical salaries swap positions between runs. Nothing in the data changed. What guarantee did you fail to establish, and what single change fixes it?

3. Under what precise condition do ROW_NUMBER, RANK, and DENSE_RANK all return identical values for every row? State it in terms of the ORDER BY columns.

4. You need "the latest row per user." Explain why RANK is a bug and ROW_NUMBER is correct, in terms of what happens when two rows share the ordering timestamp.

5. A query has `RANK() OVER (PARTITION BY dept ORDER BY salary DESC)` and EXPLAIN shows a Sort node with `external merge Disk: 40MB` beneath the WindowAgg. Describe two independent ways to make the sort disappear or fit in memory, and which one is generally preferable at 50M rows.

6. "Top 3 per category" with `DENSE_RANK() <= 3` sometimes returns 20 rows for a category and 3 for another. Is this a bug? Explain what determines the row count per category.

7. Your window ORDER BY is `commission DESC` and the salespeople with `NULL` commission unexpectedly appear at rank 1. Why, and how do you push them to the bottom of the ranking?

---

## 16. Quick Reference Card

```sql
-- ============ THE THREE RANKING FUNCTIONS ============
ROW_NUMBER() OVER (PARTITION BY p ORDER BY c)  -- unique 1..N; arbitrary among ties
RANK()       OVER (PARTITION BY p ORDER BY c)  -- ties share; GAPS   (1,2,2,4)
DENSE_RANK() OVER (PARTITION BY p ORDER BY c)  -- ties share; NO gaps (1,2,2,3)
-- () is ALWAYS empty for all three. ORDER BY defines rank order AND ties.

-- ============ TIE BEHAVIOUR (memorise) ============
-- values 90,90,80:  ROW_NUMBER 1,2,3 | RANK 1,1,3 | DENSE_RANK 1,1,2
-- No ties in ORDER BY  ->  all three are IDENTICAL.

-- ============ TOP-N PER GROUP (canonical shape) ============
SELECT * FROM (
  SELECT ..., DENSE_RANK() OVER (PARTITION BY g ORDER BY x DESC) AS r
  FROM t
) s
WHERE r <= 3;               -- MUST filter in outer query; not in WHERE/HAVING
-- ROW_NUMBER <= N : exactly N rows/group (drops tied Nth arbitrarily)
-- RANK       <= N : top N positions; may exceed N on ties; gaps after ties
-- DENSE_RANK <= N : top N distinct value tiers; may return many rows

-- ============ DEDUPLICATION (always ROW_NUMBER) ============
WITH r AS (
  SELECT *, ROW_NUMBER() OVER (
    PARTITION BY natural_key ORDER BY updated_at DESC, id DESC  -- id = tiebreak
  ) AS rn FROM t
) SELECT * FROM r WHERE rn = 1;
-- RANK here returns duplicates when updated_at ties. Never use RANK for dedup.

-- ============ DETERMINISM ============
-- Always add a UNIQUE final tiebreaker for ROW_NUMBER / pagination:
ROW_NUMBER() OVER (ORDER BY amount DESC, id ASC)
-- Adding a unique col to RANK's ORDER BY collapses it to ROW_NUMBER (no shared ranks).

-- ============ NULLS ============
RANK() OVER (ORDER BY commission DESC NULLS LAST)  -- else DESC puts NULLs FIRST

-- ============ TOP-1 SHORTCUT ============
SELECT DISTINCT ON (customer_id) *
FROM orders ORDER BY customer_id, created_at DESC, id DESC;  -- faster than rn=1

-- ============ PERFORMANCE ============
-- Cost = sort(input) + O(N) pass. Sort dominates.
-- Index (PARTITION cols, ORDER cols [+dir]) removes the Sort node entirely.
CREATE INDEX ON employees (department_id, salary DESC);
-- EXPLAIN red flag: "Sort Method: external merge Disk" under WindowAgg -> spill.
--   Fix: raise work_mem OR add the matching index.
-- Green flag: WindowAgg with an Index Scan child and NO Sort node.
```

**Interview one-liners:**
- "They differ only on ties; with a unique ORDER BY they are identical."
- "RANK skips (gaps), DENSE_RANK doesn't (dense), ROW_NUMBER never ties."
- "Dedup is always ROW_NUMBER — RANK keeps duplicates when the order column ties."
- "You can't filter a window function in WHERE; wrap it and filter outside."
- "An index on (partition cols, order cols) deletes the sort — the whole cost."

---

## Connected Topics

- **Topic 33 — PARTITION BY and ORDER BY**: The window definition that ranking functions consume; how partitions reset counters and how ORDER BY defines both rank order and ties. Direct prerequisite.
- **Topic 35 — NTILE, PERCENT_RANK, CUME_DIST**: The distribution/bucketing ranking functions — NTILE(n) splits a partition into n buckets, PERCENT_RANK and CUME_DIST give relative standing in [0,1]. They build on the same WindowAgg + peer-group machinery, extended to bucketing and relative rank. Next topic.
- **Topic 20 — GROUP BY Fundamentals**: Window functions run after GROUP BY, enabling `RANK() OVER (ORDER BY SUM(x))` to rank aggregates.
- **Topic 03 — Logical Execution Order**: Why window functions (SELECT phase) can't be filtered in WHERE/HAVING — the root cause of the subquery-wrap pattern.
- **Topic 07 — NULL in Depth**: NULL handling in the ordering column, NULLS FIRST/LAST, and why all NULLs are peers in window ordering.
- **Internals — Sort & work_mem**: The Sort node under WindowAgg, external merge spills, and how a matching index eliminates the sort.
- **Topic 17 — JOIN Performance / DISTINCT ON**: The `DISTINCT ON` alternative for top-1-per-group that can outperform `ROW_NUMBER ... WHERE rn = 1`.
