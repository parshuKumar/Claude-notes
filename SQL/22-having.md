# Topic 22 — HAVING

### SQL Mastery Curriculum — Phase 4: Aggregation and Grouping

---

## 1. ELI5 — The Analogy That Makes It Click

Imagine you run a bookshop and you keep a giant pile of every sale slip from the year. Each slip says: which customer, which book, how much they paid.

Your accountant asks two very different questions, and they sound almost the same but aren't:

**Question A:** "Only look at sales slips over $20." — You go through the pile *slip by slip*, and you throw away any individual slip under $20 before you do anything else. You're filtering **individual slips**.

**Question B:** "Only show me customers whose *total* spending for the year was over $500." — Now you can't decide by looking at a single slip. A slip for $12 might belong to a customer who spent $9,000 across the year. To answer this, you first have to **gather all the slips for each customer into a pile**, add up each pile, and *then* decide which piles to keep or throw away. You're filtering **whole piles**.

`WHERE` is Question A: it filters **individual rows** before they are grouped.

`HAVING` is Question B: it filters **whole groups** after they have been gathered and summed.

That's the entire difference, and almost every HAVING mistake in the world comes from confusing these two questions. If your condition can be answered by looking at one row alone, it belongs in `WHERE`. If your condition needs a whole group's total, count, or average, it belongs in `HAVING`.

The word "HAVING" is old SQL English for "keep only the groups *having* this property."

---

## 2. Connection to SQL Internals

`HAVING` is not a separate scanning or joining operation — it is a **filter node that sits directly on top of the aggregation node** in the plan tree. To understand HAVING you have to understand what the engine has already built by the time HAVING runs.

By the time `HAVING` executes, PostgreSQL has already produced grouped rows via one of two physical strategies:

1. **HashAggregate** — the engine builds an in-memory hash table keyed by the `GROUP BY` columns. As it scans input rows, it looks up the group's bucket and updates running aggregate state (a `sum`, a `count`, a sort-free running `max`, etc.). This uses `O(number of distinct groups)` memory in `work_mem`. When the hash table would exceed `work_mem`, PostgreSQL 13+ spills partitions to disk (`Disk Usage` in EXPLAIN) rather than blowing up.

2. **GroupAggregate** — the engine first **sorts** the input on the `GROUP BY` keys (or reads it pre-sorted from a B-tree index), then walks the sorted stream, emitting one aggregated row each time the group key changes. This uses `O(1)` aggregate memory but pays for the sort.

The aggregate **state machine** is the internal concept that matters: each aggregate function is a `(initial_state, transition_function, final_function)` triple registered in `pg_aggregate`. `sum` starts at NULL/0, adds each value, and returns the accumulator. `HAVING` is evaluated **against the finalized aggregate values** for each group — after the final function has run.

The critical internal fact: **`HAVING` is a post-aggregation predicate**. It cannot reduce how many rows go *into* the aggregate; it only decides which finished groups survive. This is why moving a row-level predicate from `HAVING` into `WHERE` is one of the highest-leverage query optimizations that exists — `WHERE` shrinks the input to the (expensive) aggregation, while `HAVING` only trims the output.

There is no special index for HAVING. An index can accelerate the `WHERE` filter feeding the aggregate, and can supply pre-sorted input to a GroupAggregate, but no B-tree can be "on" a `SUM(...) > 500` condition because that value does not exist until the group is fully consumed. (A materialized view or a partial index on a precomputed column is the only way to "index an aggregate," covered in later topics.)

---

## 3. Logical Execution Order Context

The SQL logical processing order — the order the language is *defined* to behave as if it runs, regardless of physical plan — is:

```
FROM        → assemble source rows (tables, subqueries)
JOIN        → combine per ON conditions
WHERE       → filter INDIVIDUAL rows        ← runs BEFORE grouping
GROUP BY    → collapse rows into groups
HAVING      → filter GROUPS                 ← runs AFTER grouping   ← THIS TOPIC
SELECT      → compute select-list expressions, aggregates, aliases
DISTINCT    → de-duplicate
window      → window functions (OVER)
ORDER BY    → sort
LIMIT       → truncate
```

Three consequences flow directly from HAVING's position in this pipeline, and every one of them is an interview trap:

**(1) HAVING runs after GROUP BY, so it can reference aggregates.**
`HAVING SUM(amount) > 500` is legal because by the time HAVING runs, the groups and their sums exist. `WHERE SUM(amount) > 500` is a hard **error** — at WHERE time, no group and no sum exists yet.

**(2) HAVING runs before SELECT, so it generally cannot use SELECT-list aliases.**
In standard SQL, the alias `total` in `SELECT SUM(amount) AS total ... HAVING total > 500` does not exist yet when HAVING runs. PostgreSQL is stricter than MySQL here: `HAVING total > 500` raises `column "total" does not exist`. You must repeat the aggregate: `HAVING SUM(amount) > 500`. (PostgreSQL *does* allow the alias in `GROUP BY` and `ORDER BY` as an extension, but **not** in `HAVING` or `WHERE`.)

**(3) HAVING runs before LIMIT and ORDER BY**, so filtering groups happens before you sort or truncate them — you can never "HAVING away" a row that ORDER BY already sorted; the sort sees only surviving groups.

Remember from Topic 20 (GROUP BY Fundamentals): after grouping, the only things you may reference in SELECT/HAVING/ORDER BY are (a) the grouping columns themselves and (b) aggregate expressions. HAVING obeys exactly this rule — a non-grouped, non-aggregated column in HAVING is an error, just as it is in SELECT.

---

## 4. What Is HAVING?

`HAVING` is a clause that filters **grouped rows** produced by `GROUP BY` (or by an implicit whole-table grouping when aggregates appear with no `GROUP BY`), keeping only the groups for which its condition evaluates to TRUE. It is to groups what `WHERE` is to individual rows.

```sql
SELECT   customer_id,
         COUNT(*)        AS order_count,
         SUM(amount)     AS total_spend
FROM     orders                              -- ← FROM: source rows
WHERE    status = 'completed'                -- ← WHERE: filter rows BEFORE grouping
GROUP BY customer_id                         -- ← collapse into one row per customer
HAVING   SUM(amount) > 500                   -- ← HAVING: keep only groups summing > 500
     AND COUNT(*) >= 3                        -- ← multiple group conditions, AND/OR allowed
ORDER BY total_spend DESC;                    -- ← ORDER BY: sort surviving groups
```

Annotated syntax breakdown — every element:

```
HAVING <group_predicate>
  │
  ├── HAVING ............ keyword; introduces a post-aggregation filter
  │
  ├── <group_predicate>  a boolean expression that must be constant per group:
  │        │
  │        ├── SUM(amount) > 500 ......... aggregate compared to a constant  ✅
  │        ├── COUNT(*) >= 3 ............. aggregate over the whole group     ✅
  │        ├── AVG(amount) > MIN(amount)*2  aggregate vs aggregate            ✅
  │        ├── customer_id > 100 ......... a GROUP BY column (constant/group) ✅ but belongs in WHERE
  │        ├── status = 'completed' ...... NON-grouped column                 ❌ ERROR
  │        └── total > 500 ............... a SELECT alias                      ❌ ERROR in PostgreSQL
  │
  ├── AND / OR / NOT ..... combine multiple group predicates
  │
  └── evaluated ONCE PER GROUP, against finalized aggregate values;
      group is emitted only when the predicate is TRUE
      (UNKNOWN/NULL is treated as not-TRUE → group dropped)
```

The single most important rule: **the HAVING predicate must be constant within a group.** That means every column reference must be either a `GROUP BY` column or wrapped in an aggregate. Anything that could vary row-to-row inside a group is illegal, because HAVING has no individual rows left to look at — only groups.

---

## 5. Why HAVING Mastery Matters in Production

**1. The WHERE-vs-HAVING performance cliff.** The most common HAVING mistake that survives code review isn't a wrong result — it's a *slow correct result*. Writing `HAVING status = 'completed'` instead of `WHERE status = 'completed'` produces the right answer but forces the engine to aggregate every row (including the ones you'll throw away) before filtering. On a 100M-row table this can be the difference between 200ms and 40 seconds. The engine sometimes rescues you by pushing the predicate down, but it cannot always prove the push is safe, so you must not rely on it.

**2. Silent wrong counts from filtering in the wrong place.** `HAVING COUNT(*) > 5` after a row filter means something completely different from the same query without the filter. If you put a condition that *should* narrow the population into HAVING, the counts and sums are computed over the wrong population and the numbers are quietly wrong — the query runs, returns rows, and lies.

**3. The alias trap breaks portability.** Code written against MySQL (which allows `HAVING total > 500`) breaks the moment it hits PostgreSQL. Teams migrating databases hit a wall of `column does not exist` errors that only surface at runtime, not at deploy time.

**4. HAVING without GROUP BY is a real and useful pattern** — a single guard on a whole-table aggregate (`HAVING COUNT(*) > 0`, `HAVING SUM(balance) <> 0`) — and misunderstanding it leads to either missing rows or an empty result where you expected one.

**5. Threshold reports are the backbone of analytics.** "Customers who spent over $X," "products ordered more than N times," "categories with margin below Y%," "sessions with more than K events" — these are all HAVING queries, and they are exactly the queries product managers ask for daily. Getting the WHERE/HAVING split right is the difference between a dashboard that loads and one that times out.

---

## 6. Deep Technical Content

### 6.1 The Core Decision Rule: WHERE vs HAVING

There is a single, mechanical rule that resolves 95% of WHERE-vs-HAVING decisions:

> **If the condition can be evaluated by looking at one row in isolation, it goes in `WHERE`. If it requires an aggregate over a group, it goes in `HAVING`.**

| Condition | Needs a group? | Clause |
|-----------|---------------|--------|
| `status = 'completed'` | No — one row tells you | `WHERE` |
| `created_at >= '2025-01-01'` | No | `WHERE` |
| `amount > 20` | No | `WHERE` |
| `SUM(amount) > 500` | Yes — need the whole group | `HAVING` |
| `COUNT(*) >= 3` | Yes | `HAVING` |
| `AVG(amount) > 100` | Yes | `HAVING` |
| `MAX(created_at) < NOW() - INTERVAL '90 days'` | Yes | `HAVING` |

The performance corollary: **push every row-level filter down into WHERE.** WHERE shrinks the input to the aggregation; HAVING only trims the output. Filtering early means fewer rows to sort, fewer hash buckets, less memory, and a smaller chance of spilling to disk.

```sql
-- SLOW: aggregate everything, then discard cancelled orders' groups... 
--       except this is also WRONG — see §6.2
SELECT customer_id, SUM(amount)
FROM orders
GROUP BY customer_id
HAVING status = 'completed';   -- ❌ ERROR anyway: status not grouped

-- FAST and CORRECT: filter rows first, aggregate the survivors
SELECT customer_id, SUM(amount)
FROM orders
WHERE status = 'completed'      -- ✅ shrink input before aggregating
GROUP BY customer_id
HAVING SUM(amount) > 500;       -- ✅ genuine group filter stays in HAVING
```

### 6.2 Why a Row Filter in HAVING Is Both Slow AND Often Wrong

It is tempting to think "WHERE vs HAVING is only about performance for row filters." It is not — for a row filter that references a non-grouped column, HAVING is a syntax error in the first place (`status` isn't a grouping column). But there's a subtler wrong-result case: putting a filter that references a **grouping column** in HAVING is legal but changes nothing about correctness and only hurts performance:

```sql
-- Legal but pointless in HAVING — customer_id IS a grouping column
SELECT customer_id, SUM(amount)
FROM orders
GROUP BY customer_id
HAVING customer_id > 1000;   -- works, but should be WHERE customer_id > 1000
```

Here the planner *usually* pushes `customer_id > 1000` down to before the aggregation because it can prove the predicate references only the grouping key (so it's constant per group and safe to apply early). But "usually" is not "always" — write it in WHERE and remove all doubt.

The genuinely dangerous case is when a row filter would change *which rows feed the aggregate*. Consider counting completed orders per customer:

```sql
-- WRONG population: SUM includes cancelled orders, then filters groups
SELECT customer_id, COUNT(*) AS n
FROM orders
GROUP BY customer_id
HAVING COUNT(*) > 5;
-- This counts ALL orders (including cancelled), then keeps customers with >5 total.

-- RIGHT population: count only completed, then threshold
SELECT customer_id, COUNT(*) AS n
FROM orders
WHERE status = 'completed'
GROUP BY customer_id
HAVING COUNT(*) > 5;
-- Counts only completed orders per customer, keeps those with >5 completed.
```

Both queries run. Both return rows. They answer different questions. The bug is invisible until someone reconciles the numbers.

### 6.3 HAVING With Complex Aggregates

HAVING is not limited to `agg > constant`. The predicate can be any boolean expression that is constant per group, including:

**Aggregate vs aggregate:**
```sql
-- Customers whose largest single order is more than half their total spend
-- (i.e., concentrated in one big purchase)
SELECT customer_id, MAX(amount) AS biggest, SUM(amount) AS total
FROM orders
WHERE status = 'completed'
GROUP BY customer_id
HAVING MAX(amount) > 0.5 * SUM(amount);
```

**Arithmetic on aggregates:**
```sql
-- Categories with gross margin below 20%
SELECT p.category_id,
       SUM(oi.quantity * oi.unit_price)              AS revenue,
       SUM(oi.quantity * p.cost_price)               AS cost
FROM order_items oi
JOIN products p ON p.id = oi.product_id
GROUP BY p.category_id
HAVING (SUM(oi.quantity * oi.unit_price) - SUM(oi.quantity * p.cost_price))
       / NULLIF(SUM(oi.quantity * oi.unit_price), 0) < 0.20;
```
Note the `NULLIF(..., 0)` guard — a group with zero revenue would otherwise divide by zero *inside HAVING*, raising a runtime error that aborts the whole query, not just that group.

**COUNT(DISTINCT ...) in HAVING:**
```sql
-- Products ordered by more than 50 distinct customers
SELECT oi.product_id
FROM order_items oi
JOIN orders o ON o.id = oi.order_id
WHERE o.status = 'completed'
GROUP BY oi.product_id
HAVING COUNT(DISTINCT o.customer_id) > 50;
```

**FILTER inside HAVING** (previews Topic 23):
```sql
-- Customers with more than 3 completed orders AND at least 1 refunded order
SELECT customer_id
FROM orders
GROUP BY customer_id
HAVING COUNT(*) FILTER (WHERE status = 'completed') > 3
   AND COUNT(*) FILTER (WHERE status = 'refunded')  >= 1;
```
This is the elegant modern way to express "groups meeting multiple aggregate conditions on different row subsets" without self-joins or multiple subqueries. Topic 23 covers `FILTER` fully.

**Boolean aggregates:**
```sql
-- Departments where EVERY employee earns above 50k (no one below)
SELECT department_id
FROM employees
GROUP BY department_id
HAVING bool_and(salary > 50000);       -- TRUE only if all rows satisfy it

-- Departments where AT LEAST ONE employee earns above 200k
SELECT department_id
FROM employees
GROUP BY department_id
HAVING bool_or(salary > 200000);
```

### 6.4 HAVING Without GROUP BY

When you use an aggregate with no `GROUP BY`, the entire table (post-WHERE) is treated as **one single group**. HAVING then filters that one group — it either survives (emitting one row) or is dropped (emitting zero rows).

```sql
-- Emit one row IF the whole table has more than 1M completed orders, else nothing
SELECT COUNT(*) AS total_completed
FROM orders
WHERE status = 'completed'
HAVING COUNT(*) > 1000000;
```

This is a **binary gate**: the query returns exactly one row or exactly zero rows. It's genuinely useful for guard conditions:

```sql
-- Return the aggregate ONLY if the data is non-empty and balanced
SELECT SUM(debit) AS total_debit, SUM(credit) AS total_credit
FROM audit_logs
WHERE log_date = CURRENT_DATE
HAVING SUM(debit) <> SUM(credit);   -- returns a row ONLY if the ledger is unbalanced
-- Empty result = ledger balanced = good. One row = anomaly to investigate.
```

The subtle edge case: **an aggregate over an empty set.** `COUNT(*)` over zero rows returns `0` (one row with the value 0), NOT no rows. So:

```sql
-- Table is empty (or WHERE matches nothing)
SELECT COUNT(*) FROM orders WHERE status = 'nonexistent';
-- → returns ONE row: 0

SELECT COUNT(*) FROM orders WHERE status = 'nonexistent' HAVING COUNT(*) > 0;
-- → returns ZERO rows (the single group's count is 0, fails HAVING)

SELECT SUM(amount) FROM orders WHERE status = 'nonexistent';
-- → returns ONE row: NULL (SUM of empty set is NULL, not 0!)
```

This distinction — `COUNT` of empty is `0`, `SUM`/`AVG`/`MAX`/`MIN` of empty is `NULL` — matters because a HAVING predicate on a NULL aggregate evaluates to UNKNOWN, which drops the group. `HAVING SUM(amount) > 0` over an empty set is `NULL > 0` = UNKNOWN → zero rows.

### 6.5 HAVING and NULL — Three-Valued Logic

HAVING, like WHERE, keeps a group only when the predicate is **TRUE**. UNKNOWN (from NULL) and FALSE both drop the group. This is the same three-valued logic from Topic 07 (NULL in Depth), applied to aggregate values.

```sql
-- AVG ignores NULLs; if ALL values in a group are NULL, AVG is NULL
SELECT customer_id, AVG(rating) AS avg_rating
FROM reviews
GROUP BY customer_id
HAVING AVG(rating) > 4.0;
-- A customer who left only NULL ratings has AVG(rating) = NULL
-- → NULL > 4.0 is UNKNOWN → that group is DROPPED (not an error, silently gone)
```

Key aggregate-NULL behaviours that surface in HAVING:
- `COUNT(*)` counts all rows including those with NULLs — never NULL, minimum 0.
- `COUNT(column)` counts only non-NULL values of that column.
- `SUM`, `AVG`, `MAX`, `MIN` **ignore NULL inputs**, and return NULL if *every* input is NULL (or the group is empty).
- `HAVING <aggregate> IS NOT NULL` is a legitimate group filter: "keep groups that had at least one non-NULL value."

```sql
-- Keep only groups that actually have a computable average
SELECT product_id, AVG(rating) AS avg_rating
FROM reviews
GROUP BY product_id
HAVING AVG(rating) IS NOT NULL          -- drop all-NULL-rating products
   AND AVG(rating) >= 3.5;
```

### 6.6 HAVING With Multiple Conditions and Precedence

Multiple predicates combine with `AND`, `OR`, `NOT`, and follow standard boolean precedence (`NOT` > `AND` > `OR`). Parenthesize when mixing AND/OR:

```sql
SELECT customer_id, COUNT(*) AS n, SUM(amount) AS total, AVG(amount) AS avg_val
FROM orders
WHERE status = 'completed'
GROUP BY customer_id
HAVING (COUNT(*) >= 10 AND AVG(amount) > 50)   -- loyal high-value
    OR SUM(amount) > 10000;                     -- OR any whale
```

Without the parentheses, `A AND B OR C` binds as `(A AND B) OR C`, which happens to be what we want here — but relying on implicit precedence in a group filter that a non-author reads six months later is how bugs are born. Parenthesize.

### 6.7 HAVING vs a Filtering Subquery / CTE

Any HAVING can be rewritten as a subquery or CTE that computes the aggregate and filters in an outer WHERE. They are usually equivalent, and the planner often produces the same plan:

```sql
-- HAVING form
SELECT customer_id, SUM(amount) AS total
FROM orders
WHERE status = 'completed'
GROUP BY customer_id
HAVING SUM(amount) > 500;

-- CTE / subquery form (equivalent)
SELECT customer_id, total
FROM (
  SELECT customer_id, SUM(amount) AS total
  FROM orders
  WHERE status = 'completed'
  GROUP BY customer_id
) g
WHERE total > 500;    -- now 'total' IS an alias in scope — WHERE on a derived column
```

When to prefer the subquery form:
- You want to reference the aggregate by alias (avoiding the PostgreSQL no-alias-in-HAVING restriction and avoiding repeating a long expression).
- You need to filter on the aggregate **and then** join the surviving groups to another table.
- You want the aggregate value in the output *and* to filter on a transformed version of it.

When to prefer HAVING:
- It's a straightforward "aggregate compared to threshold" — HAVING is more readable and idiomatic.
- No downstream join or reshaping is needed.

### 6.8 HAVING With Window Functions — A Trap

You **cannot** put a window function in HAVING, and you cannot use HAVING to filter window-function output. Window functions (Phase 6) are computed *after* HAVING in the logical order — they run at the `window` step, which comes after GROUP BY/HAVING. So this is an error:

```sql
-- ❌ ERROR: window functions are not allowed in HAVING
SELECT customer_id, SUM(amount) AS total
FROM orders
GROUP BY customer_id
HAVING RANK() OVER (ORDER BY SUM(amount)) <= 10;   -- illegal here
```

To filter on a window function's result, you must wrap the query and filter in an outer WHERE:

```sql
SELECT * FROM (
  SELECT customer_id,
         SUM(amount) AS total,
         RANK() OVER (ORDER BY SUM(amount) DESC) AS rnk
  FROM orders
  GROUP BY customer_id
) ranked
WHERE rnk <= 10;
```

This distinction — HAVING for aggregate filters, outer-WHERE for window filters — is a frequent senior-level interview probe.

### 6.9 HAVING and GROUPING SETS / ROLLUP / CUBE

When you group with `ROLLUP`, `CUBE`, or `GROUPING SETS`, HAVING filters the combined result including the super-aggregate (subtotal/grand-total) rows. Use the `GROUPING()` function to distinguish real groups from subtotal rows in the HAVING predicate:

```sql
SELECT category_id, product_id, SUM(revenue) AS rev
FROM sales
GROUP BY ROLLUP (category_id, product_id)
HAVING SUM(revenue) > 1000            -- applies to detail AND subtotal rows
   OR GROUPING(product_id) = 1;        -- always keep the category-subtotal rows
```

Here `GROUPING(product_id) = 1` is TRUE for the rows where `product_id` was rolled up (the subtotals), letting you keep subtotals even if an individual product's revenue is below the threshold.

### 6.10 HAVING Does Not Reduce Aggregation Work

A mental-model correction worth stating explicitly: adding a HAVING clause never makes the aggregation itself cheaper. Every group is still fully built and every aggregate fully computed before HAVING sees it. HAVING only reduces the number of rows *emitted* to the next step (ORDER BY, the client, an enclosing query). If the aggregation is your bottleneck, HAVING cannot help — only a tighter WHERE, a better index, pre-aggregation, or a materialized view can.

---

## 7. EXPLAIN — HAVING in the Plan

HAVING appears in EXPLAIN as a `Filter:` line **attached to the aggregate node** (HashAggregate or GroupAggregate), distinct from a `Filter:` on a scan node (which is a WHERE predicate). Learning to spot which filter is which is the core skill.

### 7.1 HashAggregate with a HAVING filter

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT customer_id, SUM(amount) AS total, COUNT(*) AS n
FROM orders
WHERE status = 'completed'
GROUP BY customer_id
HAVING SUM(amount) > 500;
```

```
HashAggregate  (cost=2450.00..2612.50 rows=1083 width=44)
               (actual time=41.2..44.8 rows=920 loops=1)
  Group Key: customer_id
  Filter: (sum(amount) > '500'::numeric)          ← THIS is the HAVING clause
  Rows Removed by Filter: 4180                     ← groups that failed HAVING
  Batches: 1  Memory Usage: 913kB
  ->  Seq Scan on orders  (cost=0.00..1950.00 rows=100000 width=12)
                          (actual time=0.02..18.4 rows=100000 loops=1)
        Filter: (status = 'completed')             ← THIS is the WHERE clause
        Rows Removed by Filter: 50000              ← rows dropped before grouping
Buffers: shared hit=850
Planning Time: 0.28 ms
Execution Time: 45.3 ms
```

**Reading it:**
- The **`Filter` on the `Seq Scan`** (`status = 'completed'`) is the **WHERE** clause — applied per row, before grouping. It removed 50,000 rows so only 100,000 flow up into the aggregate.
- The **`Filter` on the `HashAggregate`** (`sum(amount) > 500`) is the **HAVING** clause — applied per group, after aggregation. `Rows Removed by Filter: 4180` means 4,180 groups were built, aggregated, then discarded.
- `Group Key: customer_id` confirms the grouping column.
- `Memory Usage: 913kB`, `Batches: 1` — the hash table fit in `work_mem`; no disk spill.
- Note the cost: the aggregate still processed all 100K post-WHERE rows and built ~5,100 groups (`rows=1083` estimate vs `920` actual after HAVING). HAVING trimmed the *output*, not the work.

### 7.2 GroupAggregate (sorted input) with HAVING

```sql
EXPLAIN (ANALYZE)
SELECT department_id, AVG(salary) AS avg_sal
FROM employees
GROUP BY department_id
HAVING AVG(salary) > 80000
ORDER BY department_id;
```

```
GroupAggregate  (cost=0.42..3200.00 rows=40 width=40)
                (actual time=0.15..12.9 rows=17 loops=1)
  Group Key: department_id
  Filter: (avg(salary) > '80000'::numeric)        ← HAVING on the GroupAggregate
  Rows Removed by Filter: 33
  ->  Index Scan using employees_dept_idx on employees
        (cost=0.42..2800.00 rows=50000 width=12)
        (actual time=0.03..6.1 rows=50000 loops=1)
Planning Time: 0.19 ms
Execution Time: 13.1 ms
```

**Reading it:**
- `GroupAggregate` was chosen because the `employees_dept_idx` supplies rows already sorted by `department_id` — no separate Sort node, and the output is naturally ordered (satisfying the `ORDER BY department_id` for free).
- The HAVING `Filter` sits on the GroupAggregate: 50 departments were aggregated, 33 removed by HAVING, 17 survive.

### 7.3 The predicate-pushdown case — a "HAVING" that becomes a WHERE

Watch what happens when you put a grouping-column filter in HAVING and the planner rescues you:

```sql
EXPLAIN
SELECT customer_id, SUM(amount)
FROM orders
GROUP BY customer_id
HAVING customer_id > 1000;      -- a grouping-column filter, mis-placed in HAVING
```

```
HashAggregate  (cost=1980.00..2100.00 rows=900 width=36)
  Group Key: customer_id
  ->  Seq Scan on orders  (cost=0.00..1950.00 rows=90000 width=12)
        Filter: (customer_id > 1000)    ← planner PUSHED it down to the scan!
```

There is **no** `Filter` on the HashAggregate — the planner proved `customer_id > 1000` depends only on the grouping key, so it moved the predicate to the scan (a `WHERE`), shrinking the input from 100K to 90K rows before grouping. This is the optimization you should not rely on: it worked here, but for anything the planner can't prove safe, the filter stays post-aggregation. Write it in WHERE yourself.

### 7.4 HashAggregate spilling to disk (HAVING can't save you)

```
HashAggregate  (cost=... rows=5000000 ...)
               (actual time=8100..21400 rows=4200000 loops=1)
  Group Key: session_id
  Filter: (count(*) > 100)
  Rows Removed by Filter: 4995800
  Batches: 33  Memory Usage: 262144kB  Disk Usage: 1855032kB   ← SPILLED
  ->  Seq Scan on sessions ...
```

**Reading it:** `Batches: 33` and `Disk Usage: 1.8GB` mean the hash table of 5M groups exceeded `work_mem` and spilled. HAVING removed 4,995,800 of 5,000,000 groups — but every one of those groups was still built, hashed, and spilled *before* HAVING trimmed them. The lesson: HAVING selectivity does not reduce aggregation memory. Fix with a tighter WHERE, a higher `work_mem`, or pre-aggregation — not by tuning HAVING.

---

## 8. Query Examples

### Example 1 — Basic: Customers Who Spent Over a Threshold

```sql
-- The canonical HAVING query: filter groups by their total
SELECT
  customer_id,
  COUNT(*)       AS order_count,      -- orders per customer (post-WHERE)
  SUM(amount)    AS total_spend       -- total spend per customer
FROM orders
WHERE status = 'completed'            -- row filter: only completed orders count
GROUP BY customer_id                  -- one group per customer
HAVING SUM(amount) > 500              -- group filter: keep big spenders only
ORDER BY total_spend DESC;
```

### Example 2 — Intermediate: Multi-Condition Group Filter With Complex Aggregates

```sql
-- Find "concentrated" customers: many orders, but revenue dominated by one purchase
SELECT
  customer_id,
  COUNT(*)                              AS order_count,
  SUM(amount)                           AS total_spend,
  MAX(amount)                           AS biggest_order,
  ROUND(MAX(amount) / SUM(amount), 3)   AS concentration_ratio
FROM orders
WHERE status = 'completed'
  AND created_at >= CURRENT_DATE - INTERVAL '1 year'
GROUP BY customer_id
HAVING COUNT(*) >= 5                    -- at least 5 orders
   AND MAX(amount) > 0.5 * SUM(amount)  -- one order is >50% of all spend
ORDER BY concentration_ratio DESC
LIMIT 100;
```

### Example 3 — Production Grade: Category Margin Report With Threshold

```sql
-- Category profitability report for finance dashboard.
-- Table sizes: order_items ~20M rows, orders ~5M rows, products ~50K, categories ~200.
-- Indexes assumed:
--   orders(status, created_at)          -- for the WHERE filter
--   order_items(order_id)               -- FK join
--   order_items(product_id)             -- FK join
--   products(id) PK, products(category_id)
-- Perf expectation: ~1.5–3s cold, sub-second warm; HashAggregate over ~200 category groups
--   (tiny), so the cost is entirely in the join + WHERE scan, not the HAVING.
WITH line_items AS (
  SELECT
    p.category_id,
    oi.order_id,
    oi.quantity,
    oi.unit_price,
    p.cost_price
  FROM order_items oi
  JOIN orders   o ON o.id = oi.order_id
  JOIN products p ON p.id = oi.product_id
  WHERE o.status = 'completed'                                  -- row filter, pushed to scan
    AND o.created_at >= DATE_TRUNC('quarter', CURRENT_DATE)     -- current quarter only
)
SELECT
  c.name                                                        AS category,
  COUNT(DISTINCT li.order_id)                                   AS orders,
  SUM(li.quantity)                                              AS units_sold,
  SUM(li.quantity * li.unit_price)                              AS gross_revenue,
  SUM(li.quantity * li.cost_price)                              AS total_cost,
  ROUND(
    (SUM(li.quantity * li.unit_price) - SUM(li.quantity * li.cost_price))
    / NULLIF(SUM(li.quantity * li.unit_price), 0) * 100
  , 2)                                                          AS gross_margin_pct
FROM line_items li
JOIN categories c ON c.id = li.category_id
GROUP BY c.id, c.name
HAVING COUNT(DISTINCT li.order_id) >= 25                        -- meaningful volume only
   AND (SUM(li.quantity * li.unit_price) - SUM(li.quantity * li.cost_price))
       / NULLIF(SUM(li.quantity * li.unit_price), 0) < 0.20     -- flag low-margin categories
ORDER BY gross_margin_pct ASC;                                  -- worst margins first
```

EXPLAIN sketch for Example 3:

```
Sort  (cost=... rows=12 ...)  (actual rows=8 loops=1)
  Sort Key: (round(...))
  ->  HashAggregate  (cost=... rows=180 ...)  (actual rows=8 loops=1)
        Group Key: c.id, c.name
        Filter: ((count(DISTINCT li.order_id) >= 25) AND (... < 0.20))  ← HAVING
        Rows Removed by Filter: 172
        ->  Hash Join  (cost=... )  Hash Cond: (li.category_id = c.id)
              ->  Hash Join  Hash Cond: (oi.product_id = p.id)
                    ->  Hash Join  Hash Cond: (oi.order_id = o.id)
                          ->  Seq Scan on order_items oi ...
                          ->  Hash
                                ->  Index Scan on orders o
                                      Index Cond: (created_at >= ...)
                                      Filter: (status = 'completed')   ← WHERE
                    ->  Hash -> Seq Scan on products p
              ->  Hash -> Seq Scan on categories c
```

The HAVING filter removes 172 of 180 category groups, but note it sits *above* the entire join+scan pipeline — all the real work happens below it. HAVING here is cheap because there are only ~200 groups; the cost is the 20M-row `order_items` scan and joins.

---

## 9. Wrong → Right Patterns

### Wrong 1: Row filter placed in HAVING (error or slow)

```sql
-- WRONG: status is not a grouping column and not aggregated
SELECT customer_id, SUM(amount)
FROM orders
GROUP BY customer_id
HAVING status = 'completed';
-- ERROR:  column "status" must appear in the GROUP BY clause
--         or be used in an aggregate function
```

Why it's wrong at the execution level: at HAVING time there are no individual rows left — only groups. `status` varies row-to-row within a customer's group, so it isn't constant per group; the engine has nothing to evaluate.

```sql
-- RIGHT: row filter belongs in WHERE, before grouping
SELECT customer_id, SUM(amount)
FROM orders
WHERE status = 'completed'
GROUP BY customer_id;
```

### Wrong 2: Aggregate condition placed in WHERE

```sql
-- WRONG: SUM() cannot exist at WHERE time — groups don't exist yet
SELECT customer_id, SUM(amount)
FROM orders
WHERE SUM(amount) > 500
GROUP BY customer_id;
-- ERROR:  aggregate functions are not allowed in WHERE
```

Why it's wrong: WHERE runs in the row-filtering phase, strictly before GROUP BY. No group, therefore no `SUM`, exists. The aggregate is undefined at that point in the pipeline.

```sql
-- RIGHT: aggregate condition belongs in HAVING, after grouping
SELECT customer_id, SUM(amount)
FROM orders
GROUP BY customer_id
HAVING SUM(amount) > 500;
```

### Wrong 3: Using a SELECT alias in HAVING (works in MySQL, fails in PostgreSQL)

```sql
-- WRONG in PostgreSQL: 'total' is a SELECT alias; SELECT runs AFTER HAVING
SELECT customer_id, SUM(amount) AS total
FROM orders
GROUP BY customer_id
HAVING total > 500;
-- ERROR:  column "total" does not exist
```

Why it's wrong: logical order is `... GROUP BY → HAVING → SELECT`. The alias `total` is created in the SELECT step, which hasn't run when HAVING is evaluated. PostgreSQL enforces this; MySQL leniently allows it, which is a portability landmine.

```sql
-- RIGHT: repeat the aggregate expression in HAVING...
SELECT customer_id, SUM(amount) AS total
FROM orders
GROUP BY customer_id
HAVING SUM(amount) > 500;

-- ...or use a subquery/CTE so the alias is in scope for an outer WHERE
SELECT customer_id, total
FROM (
  SELECT customer_id, SUM(amount) AS total
  FROM orders
  GROUP BY customer_id
) g
WHERE total > 500;
```

### Wrong 4: Filtering the wrong population by mis-splitting WHERE/HAVING

```sql
-- WRONG: counts ALL orders, then keeps customers with >5 — but the question was
--        "customers with more than 5 COMPLETED orders"
SELECT customer_id, COUNT(*) AS completed_orders
FROM orders
GROUP BY customer_id
HAVING COUNT(*) > 5;
-- Silently counts cancelled/refunded/pending orders too. Numbers are wrong,
-- no error is raised, the query "works."
```

Why it's wrong: the row filter that defines the population (`status = 'completed'`) was omitted. HAVING can only threshold the count; it cannot retroactively decide which rows should have been counted.

```sql
-- RIGHT: define the population in WHERE, threshold the count in HAVING
SELECT customer_id, COUNT(*) AS completed_orders
FROM orders
WHERE status = 'completed'
GROUP BY customer_id
HAVING COUNT(*) > 5;

-- ALTERNATIVE (mixed populations in one query): use FILTER instead of WHERE
SELECT customer_id,
       COUNT(*) FILTER (WHERE status = 'completed') AS completed_orders
FROM orders
GROUP BY customer_id
HAVING COUNT(*) FILTER (WHERE status = 'completed') > 5;
```

### Wrong 5: Division by zero inside HAVING

```sql
-- WRONG: a group with zero revenue makes the ratio divide by zero → whole query aborts
SELECT category_id,
       SUM(revenue) AS rev,
       SUM(cost)    AS cost
FROM sales
GROUP BY category_id
HAVING (SUM(revenue) - SUM(cost)) / SUM(revenue) < 0.2;
-- ERROR:  division by zero   (raised for the first zero-revenue group; query fails)
```

Why it's wrong: HAVING evaluates the arithmetic per group; a single group with `SUM(revenue)=0` triggers a runtime division-by-zero that aborts the entire statement — you don't just lose that group, you lose all results.

```sql
-- RIGHT: guard the denominator with NULLIF; NULL result → group dropped, no crash
SELECT category_id,
       SUM(revenue) AS rev,
       SUM(cost)    AS cost
FROM sales
GROUP BY category_id
HAVING (SUM(revenue) - SUM(cost)) / NULLIF(SUM(revenue), 0) < 0.2;
```

### Wrong 6: Expecting a row back from HAVING over an empty set

```sql
-- WRONG expectation: "this always returns a count"
SELECT COUNT(*) AS n
FROM orders
WHERE status = 'archived'          -- suppose no rows match
HAVING COUNT(*) > 0;
-- Returns ZERO rows — the single implicit group has count 0, fails HAVING > 0.
-- Calling code that does rows[0].n will crash with "cannot read property of undefined".
```

```sql
-- RIGHT: if you need a guaranteed single row (0 when empty), drop HAVING
SELECT COUNT(*) AS n
FROM orders
WHERE status = 'archived';         -- always returns one row: 0
-- Apply the >0 decision in application code, or use it as a deliberate gate.
```

---

## 10. Performance Profile

### 10.1 The Fundamental Cost Model

HAVING itself is nearly free — it's a boolean test per group. The cost of a HAVING query lives almost entirely in the steps *below* it: the scan, the WHERE filter, the join, and the aggregation. HAVING's only performance lever is **how many rows it stops from flowing upward** to ORDER BY / LIMIT / the client — which matters only when the number of surviving groups is large and there's expensive downstream work.

### 10.2 Scaling With Table Size

| Rows scanned | Distinct groups | Aggregate strategy | HAVING impact |
|--------------|-----------------|--------------------|---------------|
| 1M | 10K | HashAggregate, fits work_mem | Negligible — trims output only |
| 10M | 500K | HashAggregate, may spill | HAVING can't prevent spill; tighten WHERE |
| 100M | 50K | HashAggregate over few groups | Cost is the 100M scan, not HAVING |
| 100M | 20M | HashAggregate spills hard | Consider pre-aggregation / materialized view |

The pattern: **HAVING scales with the number of groups (cheap), the query scales with the number of input rows (expensive).** A HAVING over 200 category groups is free even if those 200 groups summarize 100M line items — the 100M-row scan dominates.

### 10.3 The WHERE-Pushdown Optimization (the big win)

The single highest-leverage HAVING-adjacent optimization is moving row filters out of HAVING into WHERE. Concrete numbers on a 100M-row `orders` table where 5% are 'completed':

```sql
-- Version A: filter in WHERE — aggregate 5M rows
WHERE status = 'completed' GROUP BY customer_id HAVING SUM(amount) > 500
-- Aggregation input: 5,000,000 rows → ~200ms

-- Version B (only possible if the filter references a grouping column): 
-- filter in HAVING — aggregate 100M rows, then drop groups
GROUP BY customer_id HAVING ... AND customer_id-based-filter
-- Aggregation input: 100,000,000 rows → ~40s if not pushed down
```

A 20× input reduction from WHERE translates almost linearly into aggregation time and memory. Always push.

### 10.4 Index Interactions

- **Index on the WHERE column(s)** — accelerates the scan feeding the aggregate. This is where indexing pays off for HAVING queries. `CREATE INDEX ON orders(status, created_at)` lets the planner Index-Scan the 5% completed rows instead of Seq-Scanning 100M.
- **Index on the GROUP BY column(s)** — enables a `GroupAggregate` fed by an ordered Index Scan, avoiding the hash table and its memory/spill risk entirely. Good when groups are numerous. `CREATE INDEX ON orders(customer_id)`.
- **No index helps the HAVING predicate itself** — you cannot index `SUM(amount) > 500` because the sum doesn't exist until aggregation completes. The only way to "index an aggregate" is a materialized view with the aggregate precomputed, then index that.

### 10.5 work_mem and HashAggregate Spilling

HashAggregate holds one entry per distinct group in `work_mem`. Many distinct groups (high-cardinality GROUP BY) → large hash table → spill to disk (`Batches > 1`, `Disk Usage` in EXPLAIN), which is 5–10× slower. HAVING does **not** reduce this because groups are built before HAVING runs. Mitigations:
- Raise `work_mem` for the session (`SET work_mem = '256MB'`) if the group count is bounded.
- Reduce group cardinality with a tighter WHERE.
- Switch to GroupAggregate via an index on the GROUP BY key (`O(1)` aggregate memory).
- Pre-aggregate in a rollup table / materialized view refreshed on a schedule.

### 10.6 When to Pre-Aggregate Instead

If the same threshold report runs constantly over a huge fact table, don't recompute the aggregate each time. Maintain a summary table (or `MATERIALIZED VIEW`) of `customer_id, total_spend, order_count`, refresh it incrementally or on a schedule, and run the HAVING-style filter against the small summary. The 100M-row scan happens once per refresh instead of once per query.

---

## 11. Node.js Integration

### 11.1 Basic HAVING query with pg

```javascript
import pg from 'pg';
const { Pool } = pg;
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

// Customers who spent more than a (parameterized) threshold
async function getBigSpenders(minSpend, minOrders = 1) {
  const { rows } = await pool.query(
    `SELECT
       customer_id,
       COUNT(*)     AS order_count,
       SUM(amount)  AS total_spend
     FROM orders
     WHERE status = 'completed'
     GROUP BY customer_id
     HAVING SUM(amount) > $1          -- threshold is a bound parameter, never string-concatenated
        AND COUNT(*)   >= $2
     ORDER BY total_spend DESC`,
    [minSpend, minOrders]
  );
  return rows;
}
```

Note: `$1` and `$2` bind into the HAVING clause exactly like any other placeholder — HAVING has no special binding rules. Never interpolate thresholds into the SQL string; parameterize to avoid SQL injection.

### 11.2 The empty-result gotcha with HAVING gates

```javascript
// HAVING can return ZERO rows — always handle rows[0] being undefined.
async function ledgerImbalance(day) {
  const { rows } = await pool.query(
    `SELECT SUM(debit) AS debit, SUM(credit) AS credit
     FROM audit_logs
     WHERE log_date = $1
     HAVING SUM(debit) <> SUM(credit)`,   // returns a row ONLY if unbalanced
    [day]
  );
  if (rows.length === 0) {
    return { balanced: true };            // no row = balanced (or no data)
  }
  return { balanced: false, ...rows[0] }; // a row means an anomaly
}
```

### 11.3 Numeric type handling — SUM/COUNT come back as strings

```javascript
// pg returns bigint (COUNT) and numeric (SUM) as JS STRINGS to avoid precision loss.
async function getBigSpendersTyped(minSpend) {
  const { rows } = await pool.query(
    `SELECT customer_id,
            COUNT(*)::int                 AS order_count,   -- cast small counts to int
            SUM(amount)                   AS total_spend
     FROM orders
     WHERE status = 'completed'
     GROUP BY customer_id
     HAVING SUM(amount) > $1
     ORDER BY total_spend DESC`,
    [minSpend]
  );
  return rows.map(r => ({
    customerId: r.customer_id,
    orderCount: r.order_count,             // already int via ::int cast
    totalSpend: Number(r.total_spend),     // numeric arrives as string → parse
  }));
}
```
`COUNT(*)` is `bigint` and `SUM(numeric)` is `numeric`; node-postgres returns both as strings by default so you don't silently lose precision on large values. Cast in SQL (`::int`, `::float8`) or parse in JS deliberately.

### 11.4 Streaming large HAVING result sets

```javascript
// If HAVING still leaves many groups, stream instead of buffering all rows.
import QueryStream from 'pg-query-stream';

async function streamActiveCustomers(minOrders, onRow) {
  const client = await pool.connect();
  try {
    const query = new QueryStream(
      `SELECT customer_id, COUNT(*) AS n
       FROM orders
       WHERE status = 'completed'
       GROUP BY customer_id
       HAVING COUNT(*) >= $1`,
      [minOrders]
    );
    const stream = client.query(query);
    for await (const row of stream) {
      await onRow(row);                    // backpressure-friendly per-group processing
    }
  } finally {
    client.release();
  }
}
```

**Do ORMs handle HAVING?** Query-builder ORMs (Knex, Drizzle, TypeORM QueryBuilder) expose an explicit `.having()`. Model-level ORMs (Prisma, Sequelize) support it in limited "group + having" query modes but push you to raw SQL for anything beyond `aggregate > constant`. Details in §12.

---

## 12. ORM Comparison

### Prisma

**Can Prisma do HAVING?** — Yes, via `groupBy` + `having`, but only for aggregate conditions on grouped fields, and the syntax is verbose.

```typescript
import { PrismaClient } from '@prisma/client';
const prisma = new PrismaClient();

// Customers with SUM(amount) > 500 among completed orders
const bigSpenders = await prisma.order.groupBy({
  by: ['customerId'],
  where: { status: 'completed' },          // → WHERE (row filter), correct place
  _sum: { amount: true },
  _count: { _all: true },
  having: {
    amount: { _sum: { gt: 500 } },         // → HAVING SUM(amount) > 500
  },
  orderBy: { _sum: { amount: 'desc' } },
});

// Complex aggregate-vs-aggregate HAVING (MAX > 0.5*SUM) — NOT expressible; use raw SQL
const concentrated = await prisma.$queryRaw`
  SELECT customer_id, MAX(amount) AS biggest, SUM(amount) AS total
  FROM orders
  WHERE status = 'completed'
  GROUP BY customer_id
  HAVING MAX(amount) > 0.5 * SUM(amount)
`;
```

**Where it breaks:** Prisma's `having` only supports `aggregate(field) <op> constant`. Aggregate-vs-aggregate, arithmetic on aggregates, `COUNT(DISTINCT ...)`, and `FILTER` are all impossible — drop to `$queryRaw`.

**Verdict:** Fine for simple threshold reports; raw SQL for anything real.

---

### Drizzle ORM

**Can Drizzle do HAVING?** — Yes, cleanly, with `.having()` taking a `sql`/expression condition.

```typescript
import { db } from './db';
import { orders } from './schema';
import { eq, sql, gt, and } from 'drizzle-orm';

const bigSpenders = await db
  .select({
    customerId: orders.customerId,
    orderCount: sql<number>`count(*)`,
    totalSpend: sql<number>`sum(${orders.amount})`,
  })
  .from(orders)
  .where(eq(orders.status, 'completed'))            // WHERE
  .groupBy(orders.customerId)
  .having(sql`sum(${orders.amount}) > 500 and count(*) >= 3`);  // HAVING, full expression
```

**Where it breaks:** Nothing significant — the `sql` template gives you the full HAVING expression including aggregate-vs-aggregate, FILTER, and arithmetic. You lose some type-checking inside the raw `sql` fragment.

**Verdict:** Best-in-class. HAVING reads almost exactly like SQL.

---

### Sequelize

**Can Sequelize do HAVING?** — Yes, via the `having` option with `sequelize.fn` / `sequelize.literal`.

```javascript
const { Sequelize, Op } = require('sequelize');
const { Order } = require('./models');

const bigSpenders = await Order.findAll({
  attributes: [
    'customerId',
    [Sequelize.fn('COUNT', Sequelize.col('id')), 'orderCount'],
    [Sequelize.fn('SUM', Sequelize.col('amount')), 'totalSpend'],
  ],
  where: { status: 'completed' },                    // WHERE
  group: ['customerId'],
  having: Sequelize.where(
    Sequelize.fn('SUM', Sequelize.col('amount')),
    { [Op.gt]: 500 }                                 // HAVING SUM(amount) > 500
  ),
  order: [[Sequelize.literal('"totalSpend"'), 'DESC']],
});

// Multi-condition / complex HAVING — easiest via literal
const complex = await Order.findAll({
  attributes: ['customerId', [Sequelize.fn('SUM', Sequelize.col('amount')), 'total']],
  where: { status: 'completed' },
  group: ['customerId'],
  having: Sequelize.literal('SUM(amount) > 500 AND COUNT(*) >= 3'),
});
```

**Where it breaks:** The `Sequelize.where/fn/col` construction is clumsy for anything beyond one condition; most teams fall back to `Sequelize.literal('...')`, which is just raw SQL with extra ceremony.

**Verdict:** Works, but verbose. Reach for `literal` (or `sequelize.query`) for real HAVING logic.

---

### TypeORM

**Can TypeORM do HAVING?** — Yes, via QueryBuilder `.having()` / `.andHaving()`.

```typescript
import { AppDataSource } from './data-source';
import { Order } from './entities/Order';

const bigSpenders = await AppDataSource
  .getRepository(Order)
  .createQueryBuilder('o')
  .select('o.customerId', 'customerId')
  .addSelect('COUNT(*)', 'orderCount')
  .addSelect('SUM(o.amount)', 'totalSpend')
  .where('o.status = :status', { status: 'completed' })   // WHERE
  .groupBy('o.customerId')
  .having('SUM(o.amount) > :min', { min: 500 })            // HAVING, parameterized
  .andHaving('COUNT(*) >= :n', { n: 3 })
  .orderBy('SUM(o.amount)', 'DESC')
  .getRawMany();
```

**Where it breaks:** `.having()` takes a raw SQL string (parameterized) — full expressive power, but no type-checking of the expression, and you must use `getRawMany()` since aggregates aren't entity columns.

**Verdict:** Clean and parameterizable. Preferred for HAVING in a TypeORM codebase.

---

### Knex.js

**Can Knex do HAVING?** — Yes, with `.having()`, `.havingRaw()`, `.andHaving()`, `.orHaving()`.

```javascript
const knex = require('knex')({ client: 'pg', connection: process.env.DATABASE_URL });

const bigSpenders = await knex('orders')
  .select('customer_id')
  .count('* as order_count')
  .sum('amount as total_spend')
  .where('status', 'completed')                     // WHERE
  .groupBy('customer_id')
  .having(knex.raw('SUM(amount)'), '>', 500)        // HAVING SUM(amount) > 500
  .andHaving(knex.raw('COUNT(*)'), '>=', 3)
  .orderBy('total_spend', 'desc');

// Complex expression → havingRaw
const concentrated = await knex('orders')
  .select('customer_id')
  .max('amount as biggest')
  .sum('amount as total')
  .where('status', 'completed')
  .groupBy('customer_id')
  .havingRaw('MAX(amount) > 0.5 * SUM(amount)');     // full expression, parameters supported
```

**Where it breaks:** The structured `.having(column, op, value)` form is awkward for aggregate-vs-aggregate; use `.havingRaw()` for those. `havingRaw` accepts bindings (`havingRaw('SUM(amount) > ?', [500])`) to stay injection-safe.

**Verdict:** Most SQL-transparent. `.having()` for simple, `.havingRaw()` for complex.

---

### ORM Summary Table

| ORM | HAVING method | Aggregate vs constant | Aggregate vs aggregate / complex | Escape hatch |
|-----|--------------|----------------------|----------------------------------|--------------|
| Prisma | `groupBy({ having })` | ✅ (verbose object) | ❌ | `$queryRaw` |
| Drizzle | `.having(sql\`...\`)` | ✅ | ✅ (via `sql`) | `sql` template |
| Sequelize | `having:` + `fn`/`literal` | ✅ (clumsy) | ✅ via `literal` | `sequelize.query()` |
| TypeORM | `.having()` / `.andHaving()` | ✅ | ✅ (raw string) | `dataSource.query()` |
| Knex | `.having()` / `.havingRaw()` | ✅ | ✅ (`havingRaw`) | `knex.raw()` |

---

## 13. Practice Exercises

### Exercise 1 — Basic

Given:
- `orders(id, customer_id, amount, status, created_at)`

Write a query that returns, for each customer, their `customer_id`, number of completed orders, and total completed spend — but only for customers whose total completed spend exceeds $1,000. Order by total spend descending.

Then answer: if you accidentally moved `status = 'completed'` from WHERE into HAVING, what error (or wrong result) would you get, and why?

```sql
-- Write your query here
```

---

### Exercise 2 — Medium (combines Topic 20 GROUP BY and Topic 11 INNER JOIN)

Given:
- `orders(id, customer_id, status, created_at)`
- `order_items(id, order_id, product_id, quantity, unit_price)`
- `products(id, name, category_id)`

Write a query that returns each product's `name`, the number of **distinct completed orders** it appeared in, and total units sold — but only for products that appear in **more than 20 distinct completed orders** AND sold **more than 100 total units**, in the last 6 months. Order by units sold descending.

Watch out for the fan-out from Topic 11: which count needs `DISTINCT`?

```sql
-- Write your query here
```

---

### Exercise 3 — Hard (production simulation; the naive answer is wrong)

Given `orders(id, customer_id, amount, status, created_at)`, a product manager asks:

> "Give me customers who have placed **more than 5 completed orders** and whose **refund rate is above 20%**, where refund rate = refunded orders ÷ all orders."

A teammate writes:

```sql
SELECT customer_id, COUNT(*) AS n
FROM orders
WHERE status IN ('completed', 'refunded')
GROUP BY customer_id
HAVING COUNT(*) > 5
   AND COUNT(*) FILTER (WHERE status = 'refunded') / COUNT(*) > 0.2;
```

Identify every bug (there are at least three: the population is wrong for "more than 5 *completed*", integer division truncates the ratio, and the denominator "all orders" is being restricted by the WHERE). Then write the correct query.

```sql
-- Write your query here
```

---

### Exercise 4 — Interview Simulation

Given `sessions(id, user_id, event_count, started_at, duration_seconds)`:

The interviewer says: "Return users who had more than 10 sessions this month, whose **average** session duration is over 5 minutes, but exclude any user whose **single longest** session exceeded 4 hours (likely a stuck/bot session inflating the average). Also, only count sessions with `event_count > 0` as real sessions."

Write the query. Be explicit about what goes in WHERE vs HAVING and why, and handle the units (duration in seconds).

```sql
-- Write your query here
```

---

## 14. Interview Questions

### Q1 — What is the difference between WHERE and HAVING?

**Junior answer:** "WHERE filters rows and HAVING filters groups. You use HAVING with GROUP BY."

**Principal answer:** "They filter at different stages of the logical pipeline. WHERE runs *before* GROUP BY — it filters individual rows, so it can only reference columns available per-row, never aggregates. HAVING runs *after* GROUP BY — it filters whole groups, so it can (and typically does) reference aggregates like `SUM` or `COUNT`, and it may only reference grouping columns or aggregate expressions, nothing that varies within a group. The mechanical rule: if a condition can be decided by looking at one row, put it in WHERE; if it needs an aggregate over the group, put it in HAVING. Critically, this isn't just semantics — WHERE shrinks the input to the aggregation while HAVING only trims its output, so pushing row filters into WHERE is a major performance optimization. The planner sometimes pushes a HAVING predicate down to a WHERE when it can prove the predicate depends only on the grouping key, but it can't always prove that, so you shouldn't rely on it."

**Interviewer follow-up:** "Can HAVING exist without GROUP BY? What does it do then?"
*(Answer: Yes — the whole table becomes one implicit group; HAVING acts as a binary gate that emits one row or zero rows. And note `COUNT(*)` over an empty set is 0, but `SUM` over empty is NULL, which changes whether the gate passes.)*

---

### Q2 — Why can't I use a column alias defined in SELECT inside my HAVING clause in PostgreSQL?

**Junior answer:** "It just doesn't work in Postgres, you have to repeat the whole `SUM(...)`."

**Principal answer:** "Because of logical processing order: `GROUP BY → HAVING → SELECT`. SELECT-list aliases are created in the SELECT step, which runs *after* HAVING, so the alias name simply doesn't exist yet when HAVING is evaluated — Postgres raises `column does not exist`. This is standard-SQL-compliant; MySQL relaxes it and lets you use the alias, which is a portability trap when migrating from MySQL to Postgres. The fixes are to repeat the aggregate expression in HAVING, or wrap the aggregation in a subquery/CTE so the alias becomes a real derived column you can filter in an outer WHERE. Note Postgres *does* allow the alias in ORDER BY and GROUP BY as an extension — but not in HAVING or WHERE, precisely because those run before SELECT."

**Interviewer follow-up:** "So how would you filter on a value that's a transformation of an aggregate, like `ROUND(SUM(x)/COUNT(x), 2) = 4.00`?"
*(Answer: Either repeat the whole expression in HAVING, or — cleaner — compute it in a subquery/CTE, alias it, and filter the alias in an outer WHERE.)*

---

### Q3 — This report is slow. It groups a 100M-row table and has `HAVING status = 'active' AND SUM(amount) > 1000`. What's wrong?

**Junior answer:** "Add an index on `status` and `amount`."

**Principal answer:** "First, `HAVING status = 'active'` is a red flag — `status` is a per-row column, not an aggregate, so unless it's a grouping column this is actually a syntax error; if it *is* somehow valid it's a row filter stuck in the wrong clause. The fix is to move `status = 'active'` into WHERE. That's not cosmetic: in HAVING (or post-aggregation) the engine aggregates all 100M rows and only then discards groups, whereas in WHERE it filters rows *before* aggregating, shrinking the hash table dramatically — often 10–20× less work and avoiding a disk spill. Then `SUM(amount) > 1000` correctly stays in HAVING because it genuinely needs the group total. For the index: put it on the WHERE columns (`status`, and any date range) so the scan feeding the aggregate is cheap. You cannot index the `SUM > 1000` predicate — that value doesn't exist until aggregation finishes; the only way to 'index an aggregate' is a materialized view. I'd confirm all this by reading EXPLAIN ANALYZE: a `Filter` on the scan node is WHERE; a `Filter` on the HashAggregate is HAVING, and `Rows Removed by Filter` plus `Batches`/`Disk Usage` tell me whether it's spilling."

**Interviewer follow-up:** "The GROUP BY has 20 million distinct keys and EXPLAIN shows `Batches: 40, Disk Usage: 2GB`. HAVING removes 99% of groups. Does that help the spill?"
*(Answer: No. HAVING runs after the hash table is fully built and spilled; its selectivity can't reduce aggregation memory. Fixes: tighter WHERE to reduce input, raise work_mem, switch to GroupAggregate via an index on the group key, or pre-aggregate into a rollup/materialized view.)*

---

### Q4 — How would you find the top 10 customers by spend using HAVING?

**Junior answer:** "`HAVING RANK() OVER (ORDER BY SUM(amount)) <= 10`."

**Principal answer:** "You can't — window functions like RANK aren't allowed in HAVING because they're evaluated at the `window` step, which is *after* GROUP BY/HAVING in logical order. HAVING is for aggregate predicates only. 'Top 10' isn't an aggregate threshold; it's a ranking, so it belongs to a different mechanism. The idiomatic way is `GROUP BY customer_id ... ORDER BY SUM(amount) DESC LIMIT 10` — no HAVING needed. If I need the rank as a value or ties handled a specific way, I compute the aggregate and `RANK()` in a subquery/CTE, then filter `WHERE rnk <= 10` in the outer query. HAVING would only enter if 'top' were actually a threshold, like 'customers who spent more than $10,000,' which is `HAVING SUM(amount) > 10000`."

**Interviewer follow-up:** "What if I want customers in the top 10 *and* who have at least 5 orders?"
*(Answer: The 'at least 5 orders' is a genuine aggregate threshold → HAVING COUNT(*) >= 5 in the inner grouped query; the 'top 10' ranking → RANK/ROW_NUMBER in the same inner query, filtered in the outer WHERE. Two different mechanisms, composed.)*

---

## 15. Mental Model Checkpoint

1. You have a query `SELECT dept_id, AVG(salary) FROM employees GROUP BY dept_id`. You want only departments where the average salary exceeds 90,000 **and** only counting full-time employees. Which condition goes in WHERE, which in HAVING, and why does the placement of the full-time filter change the computed average?

2. A `SUM(amount)` over a group whose every `amount` is NULL returns what? What does `COUNT(*)` return for that same group? How does each behave when compared in a HAVING clause like `> 0`?

3. Explain why `HAVING total > 500` (where `total` is a SELECT alias) fails in PostgreSQL but works in MySQL. What does this tell you about each engine's adherence to the logical processing order?

4. You write `... GROUP BY customer_id HAVING customer_id > 1000`. This runs fine and the planner shows the predicate as a Filter on the Seq Scan, not on the HashAggregate. What did the planner do, and why should you still write it in WHERE yourself?

5. A HAVING query over a 100M-row table with a high-cardinality GROUP BY is spilling to disk (`Batches > 1`). Your HAVING removes 98% of groups. Will making the HAVING more selective reduce the disk spill? Why or why not, and what actually would?

6. When is it correct and useful to use HAVING with no GROUP BY at all? Give a concrete production example and describe exactly how many rows the query can return.

7. You need "products ordered by more than 50 distinct customers, where at least one of those orders was refunded." Sketch how you'd express both conditions in a single HAVING clause. What feature lets you count over a row subset inside HAVING?

---

## 16. Quick Reference Card

```sql
-- ── THE ONE RULE ──────────────────────────────────────────────
-- Condition decidable from ONE row?  → WHERE   (runs before GROUP BY)
-- Condition needs an AGGREGATE?       → HAVING  (runs after GROUP BY)

-- ── CANONICAL SHAPE ───────────────────────────────────────────
SELECT   group_col, SUM(x) AS total, COUNT(*) AS n
FROM     t
WHERE    row_filter          -- shrink input BEFORE aggregating (perf!)
GROUP BY group_col
HAVING   SUM(x) > 500         -- filter GROUPS by aggregate
     AND COUNT(*) >= 3        -- AND/OR/NOT allowed
ORDER BY total DESC
LIMIT    100;

-- ── LOGICAL ORDER ─────────────────────────────────────────────
-- FROM → JOIN → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → window → ORDER BY → LIMIT

-- ── LEGAL / ILLEGAL IN HAVING ─────────────────────────────────
HAVING SUM(x) > 500                    -- ✅ aggregate vs constant
HAVING MAX(x) > 0.5 * SUM(x)           -- ✅ aggregate vs aggregate
HAVING COUNT(DISTINCT c) > 50          -- ✅ distinct count
HAVING COUNT(*) FILTER (WHERE ...) > 3 -- ✅ filtered aggregate (Topic 23)
HAVING group_col > 100                 -- ✅ legal, but belongs in WHERE
HAVING status = 'active'               -- ❌ non-grouped column → ERROR
HAVING total > 500                     -- ❌ SELECT alias → ERROR in PostgreSQL
HAVING RANK() OVER (...) <= 10         -- ❌ window function → ERROR

-- ── HAVING WITHOUT GROUP BY (binary gate) ─────────────────────
SELECT COUNT(*) FROM t WHERE ... HAVING COUNT(*) > 0;  -- 1 row or 0 rows
-- COUNT of empty set = 0 (a row); SUM/AVG/MAX/MIN of empty = NULL

-- ── NULL / EMPTY EDGE CASES ───────────────────────────────────
-- HAVING <agg> > k  where <agg> is NULL  → UNKNOWN → group DROPPED
-- Guard division: HAVING (a) / NULLIF(b, 0) < x   -- avoid divide-by-zero abort

-- ── EXPLAIN SIGNALS ───────────────────────────────────────────
-- Filter on Seq/Index Scan node   → WHERE predicate
-- Filter on HashAggregate/GroupAggregate → HAVING predicate
-- Rows Removed by Filter (on agg) → groups dropped by HAVING
-- Batches > 1 / Disk Usage        → hash table spilled (HAVING can't fix; tighten WHERE)

-- ── PERF RULES OF THUMB ───────────────────────────────────────
-- 1. Push every row-level filter into WHERE, not HAVING.
-- 2. HAVING scales with #groups (cheap); the query scales with #input rows.
-- 3. No index accelerates a HAVING predicate; index the WHERE/GROUP BY columns.
-- 4. Spilling? Tighten WHERE, raise work_mem, index the group key, or pre-aggregate.

-- ── INTERVIEW ONE-LINERS ──────────────────────────────────────
-- "WHERE filters rows before grouping; HAVING filters groups after."
-- "Row filter in HAVING = right answer computed the slow way, or an error."
-- "You can't index an aggregate — only a materialized view precomputes it."
-- "Window functions can't go in HAVING; wrap and filter in an outer WHERE."
```

---

## Connected Topics

- **Topic 20 — GROUP BY Fundamentals**: HAVING is meaningless without the grouping it filters; the "grouping-column-or-aggregate-only" rule is shared by both SELECT and HAVING.
- **Topic 21 — Aggregate Functions**: HAVING predicates are built almost entirely from `SUM`, `COUNT`, `AVG`, `MIN`, `MAX`, `COUNT(DISTINCT)`, `bool_and/bool_or` — their NULL and empty-set behaviour drives HAVING's edge cases.
- **Topic 23 — FILTER Clause**: `COUNT(*) FILTER (WHERE ...)` inside HAVING is the clean way to threshold multiple row-subset aggregates in one group filter — the natural next step from this topic.
- **Topic 11 — INNER JOIN in Depth**: Fan-out from one-to-many joins inflates counts *before* aggregation, so HAVING thresholds like `COUNT(*)` need `DISTINCT` awareness.
- **Topic 07 — NULL in Depth**: Three-valued logic governs whether a NULL aggregate value keeps or drops a group in HAVING (UNKNOWN ≠ TRUE → dropped).
- **Internals — HashAggregate vs GroupAggregate**: HAVING is a Filter node atop the aggregate; understanding the hash table, `work_mem`, and spill behaviour explains why HAVING can't reduce aggregation cost.
- **Later — Window Functions (Phase 6)**: The mechanism you must use *instead of* HAVING to filter ranked/windowed results, because windows run after HAVING in logical order.
- **Later — Materialized Views**: The only real way to "index an aggregate" and make repeated HAVING-threshold reports fast at scale.
