# Topic 20 — GROUP BY Fundamentals
### SQL Mastery Curriculum — Phase 4: Aggregation and Grouping

---

## 1. ELI5 — The Analogy That Makes It Click

Imagine you empty a huge bag of loose coins onto a table. Hundreds of coins, all mixed up: pennies, nickels, dimes, quarters. Someone asks you: "How much money is here, broken down by coin type?"

You don't count them one by one and report each coin. Instead you instinctively do two things:

1. **You sort them into piles** — all pennies together, all nickels together, all dimes together, all quarters together. Four piles.
2. **You collapse each pile into a single answer** — "the penny pile has 84 coins worth $0.84," "the quarter pile has 31 coins worth $7.75," and so on.

That is exactly what `GROUP BY` does. The coins are your rows. The "coin type" is your grouping column. The piling step is the grouping. The "collapse each pile into one number" step is the aggregate function (`COUNT`, `SUM`, `AVG`, `MAX`, `MIN`).

The critical insight — the one that trips up every beginner — is what happens *after* you pile the coins. Once the penny pile is a pile, you can no longer point at "the third penny" and ask about its individual serial number. The pile has become a single thing. You can only ask questions *about the whole pile*: how many, total value, oldest mint year. The moment you `GROUP BY`, every individual row inside a group loses its separate identity. That is why you can only `SELECT` two kinds of things after a `GROUP BY`:

- The thing you piled *by* (the coin type — it's the same for every coin in the pile, so it survives).
- A summary *of* the pile (a count, a sum, an average — an aggregate).

Ask for anything else — "give me the serial number of the coins, grouped by type" — and the database throws its hands up: *which* serial number? There are 84 of them in the penny pile. It can't pick one. That error message ("column must appear in the GROUP BY clause or be used in an aggregate function") is the database telling you: you're pointing at an individual coin inside a pile that no longer has individuals.

That's the whole model. Pile, then collapse. Everything else in this topic is the mechanics of *how* the engine builds the piles and *why* the rules about what you can select are the way they are.

---

## 2. Connection to SQL Internals

`GROUP BY` is not free. Underneath a single line of SQL, PostgreSQL must physically partition your rows into groups and then fold each group down to one output row. There are exactly two physical strategies, and knowing which one you got is the difference between a 5ms query and a 5-second query.

### Strategy 1: HashAggregate

The engine builds an **in-memory hash table**. The key is the set of grouping columns; the value is the running aggregate state for that group (e.g., a running count and running sum). It scans the input once. For each row:

1. Hash the grouping columns to find the bucket.
2. If the group already exists in the hash table, update its running state (increment count, add to sum).
3. If not, create a new entry.

After the scan, it walks the hash table and emits one row per entry. This is **O(N)** — a single pass. It requires **O(number of distinct groups)** memory to hold the hash table. It does **not** require the input to be sorted, and it does **not** produce sorted output (the output order is the hash-table walk order, effectively undefined). This is the strategy the planner prefers when the number of distinct groups is small enough to fit the hash table in `work_mem`.

### Strategy 2: GroupAggregate (sorted aggregation)

The engine **sorts the input by the grouping columns first**, then scans the sorted stream. Because equal grouping keys are now adjacent, it accumulates aggregate state for the current group and, the moment the grouping key changes, emits the completed group and resets. This is the classic "streaming" fold.

Cost is **O(N log N)** if a sort is needed, but only **O(1)** additional memory beyond the sort (it holds state for one group at a time). Crucially, if the input is *already sorted* on the grouping columns — because it arrived via an **index scan on a B-tree index** whose leading columns match the `GROUP BY` — the sort is free, and GroupAggregate becomes O(N) with O(1) memory. It also produces output already sorted by the grouping columns, which can satisfy a downstream `ORDER BY` for free.

### The choice, and why it matters

The planner picks between them by cost:

- **Few distinct groups, no useful index** → HashAggregate (one pass, small hash table).
- **Many distinct groups** (hash table won't fit `work_mem`) → GroupAggregate, or a HashAggregate that **spills to disk** in batches (`Batches > 1` in EXPLAIN, the aggregation equivalent of a hash-join spill).
- **A B-tree index already orders the data by the grouping key** → GroupAggregate with a free sort, often the cheapest of all and the only one that comes out pre-sorted.

Internally this touches the same machinery as everything else: the **buffer pool** (pages read to feed the scan), **B-tree indexes** (to provide pre-sorted input), `work_mem` (the memory ceiling that decides hash vs. spill), and the **planner's row estimates** (a bad `n_distinct` statistic makes the planner pick HashAggregate for what turns out to be 50 million groups, then run out of memory and spill catastrophically). `GROUP BY` is where statistics quality becomes visible in wall-clock time.

---

## 3. Logical Execution Order Context

SQL's logical clause evaluation order is the backbone you must keep in your head:

```
FROM / JOIN     ← tables assembled, rows produced (with fan-out from Topic 11)
WHERE           ← row filter, BEFORE grouping
GROUP BY        ← rows collapsed into groups        ← YOU ARE HERE
HAVING          ← group filter, AFTER grouping
SELECT          ← expressions & aggregates computed
DISTINCT
ORDER BY
LIMIT
```

Several non-negotiable consequences fall directly out of `GROUP BY`'s position in this pipeline:

**1. `WHERE` runs before `GROUP BY`, so `WHERE` cannot reference aggregates.**
`WHERE COUNT(*) > 5` is a syntax error — at the time `WHERE` runs, no groups exist yet, so no count exists. To filter *on* an aggregate you must use `HAVING`, which runs after grouping. This is the single most common `GROUP BY` beginner error.

**2. `WHERE` filters raw rows; `HAVING` filters whole groups.**
`WHERE status = 'completed'` throws away individual rows before they ever enter a group. `HAVING SUM(amount) > 1000` throws away entire groups after they are formed. They are not interchangeable, and putting a row-level predicate in `HAVING` when it belongs in `WHERE` is both slower (rows are grouped only to be discarded) and, if aggregates are involved, semantically different.

**3. `SELECT` runs *after* `GROUP BY`, so aliases defined in SELECT don't exist yet in GROUP BY... except by a special exception.**
Standard SQL says `GROUP BY` can't see `SELECT` aliases because SELECT is evaluated later. PostgreSQL grants a convenience exception: you *may* `GROUP BY` a SELECT-list output alias or an ordinal position. More on this in Section 6.

**4. Because `GROUP BY` runs after `FROM`/`JOIN`, it sees the *fanned-out* row count.**
Remember from Topic 11: an INNER JOIN to a one-to-many table multiplies rows. `GROUP BY` collapses those multiplied rows — which is exactly why `COUNT(o.id)` and `COUNT(DISTINCT o.id)` diverge after a join. GROUP BY is the second half of the join-then-aggregate pattern, and the two topics are inseparable.

**5. `ORDER BY` runs after everything, so it *can* see aggregates and aliases** — `ORDER BY SUM(amount) DESC` and `ORDER BY total_revenue DESC` both work, even though the same expression would fail in `WHERE`.

Keep this pipeline diagram burned into memory. Ninety percent of "why won't this query run?" questions in aggregation are a clause referencing something that doesn't exist yet at its point in the order.

---

## 4. What Is GROUP BY?

`GROUP BY` partitions the rows produced by `FROM`/`JOIN`/`WHERE` into groups that share the same values for the listed grouping expressions, then produces exactly **one output row per group**. Every column in the `SELECT` list must either be one of the grouping expressions or be wrapped in an aggregate function, because for anything else the engine cannot decide which of the group's many row-values to return.

```sql
SELECT
  o.status,                       -- ┐ grouping column: legal in SELECT because
                                  -- │ it is identical for every row in the group
  COUNT(*)          AS order_cnt, -- ├── aggregate: collapses the whole group to one number
  SUM(o.total_amount) AS revenue, -- │
  AVG(o.total_amount) AS avg_val  -- ┘
FROM orders o                     -- ← FROM: source rows assembled first
WHERE o.created_at >= '2025-01-01'-- ← WHERE: filters raw rows BEFORE grouping
GROUP BY o.status                 -- ← the grouping key: one output row per distinct status
HAVING COUNT(*) > 10              -- ← HAVING: filters GROUPS after aggregation
ORDER BY revenue DESC;            -- ← ORDER BY: can reference the aggregate/alias
```

Annotated syntax breakdown of the `GROUP BY` clause itself:

```sql
GROUP BY
    o.status,          -- │ a plain column reference — the common case
    o.region,          -- │ multiple columns → group by the COMBINATION
                       -- │   (one group per distinct (status, region) PAIR,
                       -- │    NOT one group per status and separately per region)
    DATE_TRUNC('month', o.created_at),  -- │ an EXPRESSION — group by computed value
    EXTRACT(dow FROM o.created_at),     -- │ another expression, fully allowed
    3                  -- │ an ORDINAL — the 3rd item in the SELECT list
                       -- │   (positional shorthand; see 6.6)
    ;
```

Key structural facts, stated precisely:

- **Multiple grouping columns group by the tuple.** `GROUP BY status, region` produces one row per distinct `(status, region)` combination — not the Cartesian product of all statuses and all regions, only the combinations that actually occur in the data.
- **`GROUP BY` with no aggregate in SELECT** still works and behaves like `DISTINCT` on the grouping columns: `SELECT status FROM orders GROUP BY status` returns each distinct status once.
- **NULL is its own group.** All rows where the grouping column is NULL land in a single group together (NULLs are treated as "not distinct from each other" for grouping — unlike in `=` comparisons). More in Section 6.
- **Grouping expressions may be columns, arbitrary expressions, or ordinal positions** — but *not* aggregate functions (you cannot `GROUP BY SUM(x)`; that's what `HAVING` is for).

---

## 5. Why GROUP BY Mastery Matters in Production

1. **It is the entire foundation of analytics and reporting.** Every dashboard number — revenue by month, orders by status, active users by plan tier, error rate by endpoint — is a `GROUP BY`. If your grouping is wrong, every downstream number a stakeholder sees is wrong, and it will be wrong *silently* (the query runs, it just returns the wrong aggregation).

2. **The "must appear in GROUP BY" error is the #1 blocker for developers new to aggregation**, and understanding *why* (the pile-has-no-individuals model from Section 1) is the difference between mechanically adding columns to `GROUP BY` until the error goes away versus writing correct queries deliberately.

3. **Hash vs. sort aggregate is a real, measurable performance fork.** A `GROUP BY` over a 100M-row table that gets a HashAggregate with a good index-fed sort finishes in seconds; the same query that misjudges `n_distinct` and spills a giant hash table to disk can take minutes. Knowing how to read the plan and steer it (indexes, `work_mem`, pre-aggregation) is a core performance skill.

4. **Fan-out interaction with JOINs produces wrong-number bugs that pass code review.** As Topic 11 showed, `SUM(o.total_amount)` after joining `orders` to `order_items` double-counts the order total once per item. `GROUP BY` is where these bugs surface, and they produce plausible-looking (just inflated) numbers — the most dangerous kind of bug.

5. **The functional-dependency exception (grouping by primary key) is a real productivity and correctness feature** that lets you select many columns while grouping by one — but only when PostgreSQL can *prove* the other columns are determined by the group key. Misunderstanding it leads to either needlessly verbose `GROUP BY` clauses or confusion about why the "shortcut" works in one query and errors in another.

6. **`GROUP BY` output ordering is undefined unless you say otherwise.** Developers routinely assume a HashAggregate returns groups in some natural order, ship code that depends on it, and get flaky output when the plan flips to hash and back. Explicit `ORDER BY` is not optional in production.

---

## 6. Deep Technical Content

### 6.1 The Core Rule: Every SELECT Column Is Grouping Key or Aggregate

This is the rule the engine enforces, and it is a direct consequence of "one output row per group." Consider:

```sql
-- ERROR
SELECT o.status, o.id, COUNT(*)
FROM orders o
GROUP BY o.status;
-- ERROR:  column "o.id" must appear in the GROUP BY clause
--         or be used in an aggregate function
```

There are, say, 40,000 orders with `status = 'completed'`. They collapse into **one** output row. That row has a single `status` value (`'completed'` — the same for all 40,000, so it's well-defined). But `o.id`? There are 40,000 different ids. The engine cannot put 40,000 values in one cell, and it refuses to silently pick one. So it errors.

The rule, stated formally: **every non-aggregated expression in `SELECT`, `HAVING`, and `ORDER BY` must be functionally determined by the `GROUP BY` columns.** The simplest way to be determined is *to be* one of the grouping columns. The other ways are covered in 6.5.

Three legal ways to fix the query above:

```sql
-- (a) Aggregate the offending column — pick a representative
SELECT o.status, MAX(o.id) AS max_id, COUNT(*) FROM orders o GROUP BY o.status;

-- (b) Add it to the grouping key — now one row per (status, id) = every row
SELECT o.status, o.id, COUNT(*) FROM orders o GROUP BY o.status, o.id;

-- (c) Don't select it at all
SELECT o.status, COUNT(*) FROM orders o GROUP BY o.status;
```

### 6.2 NULL Handling in GROUP BY

`GROUP BY` treats NULLs as a single group. This is a deliberate departure from `=` semantics, where `NULL = NULL` is `UNKNOWN` (never TRUE). For grouping, PostgreSQL uses "not distinct from" semantics: all NULLs are considered equal *to each other* for the purpose of forming a group.

```sql
-- employees with NULL department_id all collapse into ONE group
SELECT department_id, COUNT(*) AS headcount
FROM employees
GROUP BY department_id;
```

```
 department_id | headcount
---------------+-----------
             1 |        12
             2 |         8
             3 |        15
        (null) |         4   ← all four NULL-department employees, one group
```

This matters because it means unassigned rows are *not* lost — they aggregate into a visible NULL bucket. Contrast with `WHERE department_id = NULL` (always empty) or an INNER JOIN on `department_id` (silently drops them, Topic 11). If you want to relabel the NULL group:

```sql
SELECT COALESCE(department_id::text, 'unassigned') AS dept, COUNT(*)
FROM employees
GROUP BY COALESCE(department_id::text, 'unassigned');
-- Note: grouping by the EXPRESSION, so it must be repeated (or use an ordinal)
```

Multi-column grouping with NULLs: a group key of `(NULL, 'US')` is distinct from `('active', 'US')` and from `(NULL, 'UK')`. NULL participates as a normal value-slot in the tuple; it's just that NULL matches NULL for grouping.

### 6.3 GROUP BY on Multiple Columns

Grouping by several columns groups by the **combination that actually appears**, not the full Cartesian product:

```sql
SELECT o.status, o.payment_method, COUNT(*) AS cnt, SUM(o.total_amount) AS revenue
FROM orders o
GROUP BY o.status, o.payment_method
ORDER BY o.status, revenue DESC;
```

If there are 4 statuses and 5 payment methods, you get *at most* 20 rows — and fewer if some combinations never occur (e.g., no `'cancelled'` order was ever paid by `'gift_card'`). The engine never invents empty combinations; only observed tuples become groups. (To get the full grid including zero-count combinations you'd need `CROSS JOIN` of the dimension tables `LEFT JOIN`ed to the facts — a different technique, Topic 12.)

Column order in `GROUP BY` does **not** change *which* groups form or the aggregate values — `GROUP BY status, region` and `GROUP BY region, status` produce identical group sets and identical numbers. It can, however, influence which index the planner can use to feed a pre-sorted GroupAggregate (a B-tree on `(status, region)` orders by status first).

### 6.4 GROUP BY on Expressions

You can group by any expression, not just bare columns. This is how time-bucketed reports are built:

```sql
-- Revenue per calendar month
SELECT
  DATE_TRUNC('month', o.created_at) AS month,
  COUNT(*)                          AS orders,
  SUM(o.total_amount)               AS revenue
FROM orders o
WHERE o.status = 'completed'
GROUP BY DATE_TRUNC('month', o.created_at)   -- group by the computed month bucket
ORDER BY month;
```

The rule: **whatever expression appears in `SELECT` as a non-aggregate must appear identically in `GROUP BY`.** PostgreSQL matches them structurally. This leads to the annoying repetition of `DATE_TRUNC('month', o.created_at)` in both places — which is exactly what the alias and ordinal shortcuts (6.6) exist to relieve.

Common expression-grouping patterns:

```sql
GROUP BY DATE_TRUNC('day', created_at)          -- daily buckets
GROUP BY EXTRACT(hour FROM created_at)          -- hour-of-day histogram (0–23)
GROUP BY EXTRACT(dow FROM created_at)           -- day-of-week (0=Sun)
GROUP BY (total_amount >= 100)                  -- boolean bucket: big vs small orders
GROUP BY CASE WHEN age < 18 THEN 'minor'
              WHEN age < 65 THEN 'adult'
              ELSE 'senior' END                 -- custom bucketing
GROUP BY LEFT(postal_code, 3)                   -- coarse geographic bucket
GROUP BY width_bucket(total_amount, 0, 1000, 10)-- histogram into 10 equal bands
```

An important performance note: grouping by an expression like `DATE_TRUNC('month', created_at)` cannot use a plain B-tree index on `created_at` to provide a *pre-sorted* GroupAggregate unless there is an **expression index** on `DATE_TRUNC('month', created_at)` itself. Without it, the planner sorts or hashes on the computed value. For hot reporting queries, an expression index (or a materialized/generated column) is the fix.

### 6.5 The PostgreSQL Functional-Dependency Exception (Grouping by Primary Key)

Standard SQL-92 required *every* non-aggregated SELECT column to be listed in `GROUP BY`. SQL-99 relaxed this: if the `GROUP BY` includes the **primary key** of a table, you may select *any* column of that table without listing it, because the primary key **functionally determines** every other column in its row. PostgreSQL implements this.

```sql
-- LEGAL in PostgreSQL, even though name, email, tier are not in GROUP BY:
SELECT
  u.id,                          -- primary key
  u.name,                        -- functionally dependent on u.id
  u.email,                       -- functionally dependent on u.id
  u.tier,                        -- functionally dependent on u.id
  COUNT(o.id)      AS order_cnt,
  SUM(o.total_amount) AS lifetime_value
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
GROUP BY u.id;                   -- ONLY the PK — Postgres infers the rest
```

Why is this safe? Because `u.id` is the primary key, each group contains rows from exactly **one** `users` row (multiplied by matching orders). Within that group, `u.name`, `u.email`, `u.tier` are all constant — there's only one possible value, so the engine can return it unambiguously. PostgreSQL *proves* this from the primary-key constraint and permits it.

**Critical constraints on the exception — it only works when Postgres can prove the dependency:**

```sql
-- WORKS: grouping by declared primary key
GROUP BY u.id                    -- id is PRIMARY KEY → all u.* columns allowed

-- FAILS: grouping by a UNIQUE but NULLable column is NOT sufficient
--   (Postgres requires PK, or at least a NOT NULL unique — historically PK only
--    is the reliably supported case; a plain UNIQUE column does not qualify)
GROUP BY u.email                 -- ERROR if you then select u.id, u.name

-- FAILS: the dependency must be within the SAME table
SELECT u.id, u.name, d.budget    -- d.budget belongs to departments, NOT determined by u.id
FROM users u JOIN departments d ON ...
GROUP BY u.id;                   -- ERROR: "d.budget must appear in GROUP BY"

-- FAILS: only columns of the PK's own table are covered
SELECT u.id, o.status            -- o.status is from orders; u.id doesn't determine it
FROM users u JOIN orders o ON o.user_id = u.id
GROUP BY u.id;                   -- ERROR on o.status
```

The exception is genuinely useful: for the extremely common "one row per entity plus some aggregates about its children" query, you write `GROUP BY u.id` instead of `GROUP BY u.id, u.name, u.email, u.tier, u.created_at, ...`. It's shorter, it's less error-prone (add a column to SELECT, no need to touch GROUP BY), and it's marginally faster to plan (fewer grouping columns, though the group set is identical since they're all determined by the PK).

Portability caveat: this is standard SQL-99, and PostgreSQL and MySQL (in `ONLY_FULL_GROUP_BY` mode) support the PK form, but not all engines infer every dependency. When in doubt, or when writing cross-database SQL, list all columns explicitly.

### 6.6 Grouping by Ordinal Position and by SELECT Alias

Two shorthands reduce the repetition of expression grouping:

**Ordinal position** — an integer in `GROUP BY` refers to the Nth item in the `SELECT` list (1-based):

```sql
SELECT DATE_TRUNC('month', o.created_at) AS month,   -- position 1
       o.status,                                      -- position 2
       COUNT(*)                          AS cnt       -- position 3
FROM orders o
GROUP BY 1, 2                -- = GROUP BY DATE_TRUNC('month', created_at), o.status
ORDER BY 1;                  -- ordinals work in ORDER BY too
```

`GROUP BY 1, 2` means "group by the 1st and 2nd select-list expressions." It's concise and common in ad-hoc analytics, but fragile: reorder the SELECT list and the grouping silently changes meaning. Prefer explicit expressions or aliases in code that will be maintained.

**SELECT alias** — PostgreSQL lets `GROUP BY` reference an output alias defined in `SELECT`, even though logically SELECT is evaluated after GROUP BY:

```sql
SELECT DATE_TRUNC('month', o.created_at) AS month,
       COUNT(*) AS cnt
FROM orders o
GROUP BY month           -- references the SELECT alias "month"
ORDER BY month;
```

This is a PostgreSQL convenience extension. Standard SQL does not allow it (Oracle historically rejected it), so it's non-portable, but it's clean and widely used in Postgres codebases. Note the precedence subtlety: if you have both a *column* named `month` in a table and a SELECT alias `month`, PostgreSQL prefers the underlying column in `GROUP BY`, which can surprise you. Ordinals avoid that ambiguity entirely.

Precedence summary for a bare name in `GROUP BY`: PostgreSQL first looks for an input column of that name; only if none exists does it fall back to a SELECT alias. `ORDER BY` uses the opposite, alias-first preference. This asymmetry is a known Postgres wart — reach for ordinals or fully-qualified expressions when it matters.

### 6.7 GROUP BY vs DISTINCT

`SELECT DISTINCT status FROM orders` and `SELECT status FROM orders GROUP BY status` return the same rows. They are two ways to ask "what are the distinct values?" The planner often produces nearly identical plans (both may HashAggregate). The meaningful differences:

- **`GROUP BY` can also compute aggregates; `DISTINCT` cannot.** If you need `COUNT(*)` per value, only `GROUP BY` works. `DISTINCT` is deduplication only.
- **`DISTINCT ON (col)`** (a PostgreSQL extension) picks one full row per distinct `col` value, ordered — a different tool for "latest row per group" (covered later in the window-function topics).
- Stylistically, use `DISTINCT` when you only want unique values and no aggregation; use `GROUP BY` the moment an aggregate enters the picture.

### 6.8 Empty-Set and Single-Group Behavior

Two edge cases with surprising answers:

**A `GROUP BY` query over zero matching rows returns zero rows.** If `WHERE` filters everything out, there are no rows to form groups, so the result is empty — *even for `COUNT`*:

```sql
SELECT status, COUNT(*) FROM orders WHERE 1 = 0 GROUP BY status;
-- returns: (no rows)  — NOT a row with count 0
```

**But an aggregate query with NO `GROUP BY` over zero rows returns ONE row.** This is the key asymmetry:

```sql
SELECT COUNT(*) FROM orders WHERE 1 = 0;
-- returns: 0   ← one row, value 0

SELECT SUM(total_amount) FROM orders WHERE 1 = 0;
-- returns: (null)  ← one row, SUM of empty set is NULL (not 0!)
```

Without `GROUP BY`, the entire result is treated as a single implicit group ("grand total"), which always produces exactly one row. `COUNT` of nothing is `0`; `SUM`, `AVG`, `MAX`, `MIN` of nothing are all `NULL`. Wrap `SUM` in `COALESCE(SUM(...), 0)` when you need a numeric zero. This asymmetry (implicit group = one row; explicit GROUP BY = zero rows) is a favorite interview trap.

### 6.9 Aggregate FILTER — Conditional Aggregation Within Groups

PostgreSQL supports the SQL-standard `FILTER (WHERE ...)` clause on aggregates, letting you compute multiple conditional aggregates in a single grouped pass — no self-joins, no subqueries:

```sql
SELECT
  DATE_TRUNC('month', created_at)                       AS month,
  COUNT(*)                                               AS total_orders,
  COUNT(*) FILTER (WHERE status = 'completed')          AS completed,
  COUNT(*) FILTER (WHERE status = 'cancelled')          AS cancelled,
  SUM(total_amount) FILTER (WHERE status = 'completed') AS completed_revenue
FROM orders
GROUP BY 1
ORDER BY 1;
```

Each `FILTER` restricts which rows *that particular aggregate* sees, all within the same group and the same single scan. Before `FILTER` existed the idiom was `SUM(CASE WHEN status='completed' THEN total_amount ELSE 0 END)` — still valid and equivalent for `SUM`, but `FILTER` is cleaner and, for `COUNT`/`AVG`, avoids the CASE-to-NULL trickery. This belongs to `GROUP BY` fundamentals because conditional aggregation is how one grouped query replaces a dashboard's worth of separate queries.

### 6.10 The Fan-Out Trap Revisited

Because `GROUP BY` runs after `JOIN`, it aggregates *fanned-out* rows. This is worth restating as a hard rule because it produces silent numeric bugs:

```sql
-- BUG: total_amount is on ORDERS but we joined to ORDER_ITEMS (many per order).
-- Each order's total_amount is repeated once per item, so SUM over-counts.
SELECT o.customer_id, SUM(o.total_amount) AS revenue   -- WRONG: inflated
FROM orders o
JOIN order_items oi ON oi.order_id = o.id
GROUP BY o.customer_id;
```

The order-level `total_amount` should be summed over *distinct orders*, but the join repeated each order row once per item. Fixes: pre-aggregate items in a subquery before joining, or sum only genuinely additive item-level columns (`SUM(oi.quantity * oi.unit_price)`), or use `SUM(DISTINCT ...)` only when the value is truly unique per group (rarely safe). The reliable pattern is pre-aggregation, shown in Section 8 Example 3. Any time you `GROUP BY` after joining to a one-to-many child, ask: "is this aggregate summing a parent column that the join duplicated?"

---

## 7. EXPLAIN — Aggregation Strategies in the Plan

### HashAggregate — few groups, one pass, unsorted output

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT status, COUNT(*), SUM(total_amount)
FROM orders
GROUP BY status;
```

```
HashAggregate  (cost=2140.00..2140.05 rows=5 width=48)
               (actual time=31.2..31.2 rows=5 loops=1)
  Group Key: status
  Batches: 1  Memory Usage: 24kB
  ->  Seq Scan on orders  (cost=0.00..1640.00 rows=100000 width=14)
                          (actual time=0.01..12.4 rows=100000 loops=1)
Buffers: shared hit=640
Planning Time: 0.14 ms
Execution Time: 31.3 ms
```

**Reading it:**
- `HashAggregate` — the hash-table strategy. `Group Key: status` names the grouping column.
- `rows=5` — the planner estimates 5 distinct statuses. If actual distinct count blew past `work_mem`, you'd see `Batches > 1` and disk spill.
- `Memory Usage: 24kB` — the entire hash table is tiny (5 groups). Comfortably under `work_mem`.
- `Seq Scan ... rows=100000` — one full pass over the table feeds the aggregate. No sort node — that's the HashAggregate advantage.
- Output is **not** sorted; if you need order, add `ORDER BY` (which would add a Sort node *above* the HashAggregate).

### GroupAggregate fed by an index — pre-sorted, no sort node

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT customer_id, COUNT(*), SUM(total_amount)
FROM orders
GROUP BY customer_id
ORDER BY customer_id;
-- with index: CREATE INDEX ON orders(customer_id);
```

```
GroupAggregate  (cost=0.42..8930.00 rows=48000 width=48)
                (actual time=0.03..96.5 rows=48213 loops=1)
  Group Key: customer_id
  ->  Index Scan using orders_customer_id_idx on orders
        (cost=0.42..7250.00 rows=100000 width=14)
        (actual time=0.02..48.1 rows=100000 loops=1)
Buffers: shared hit=520 read=180
Planning Time: 0.20 ms
Execution Time: 99.8 ms
```

**Reading it:**
- `GroupAggregate` — the sorted-stream strategy. It works because the `Index Scan` delivers rows already ordered by `customer_id`, so no separate `Sort` node is needed and no big hash table is built.
- `rows=48000` groups — far too many to want a hash table; the planner correctly chose the index-fed sorted aggregate. Memory is O(1) (state for one group at a time).
- The output arrives pre-sorted by `customer_id`, satisfying the `ORDER BY` for free — notice there is no `Sort` node above the aggregate.

### HashAggregate spilling to disk — the pathology to recognize

```sql
EXPLAIN (ANALYZE)
SELECT user_id, COUNT(*) FROM sessions GROUP BY user_id;
-- user_id has 8 million distinct values; work_mem is only 4MB
```

```
HashAggregate  (cost=..)  (actual time=0.05..14200 rows=8000000 loops=1)
  Group Key: user_id
  Batches: 33  Memory Usage: 4096kB  Disk Usage: 512000kB
  ->  Seq Scan on sessions  (actual rows=40000000 loops=1)
Execution Time: 15100 ms
```

**Reading it:**
- `Batches: 33` and `Disk Usage: 512000kB` — the hash table did NOT fit in `work_mem`, so PostgreSQL partitioned it into 33 batches and spilled to temp files. This is the aggregation equivalent of a hash-join spill and it is the classic cause of a "GROUP BY got slow."
- **Fixes:** raise `work_mem` for this query (`SET work_mem = '256MB'`); or feed a GroupAggregate via `CREATE INDEX ON sessions(user_id)` (O(1) memory, no spill); or reduce cardinality by pre-filtering.

### Signal table

| Plan symptom | Meaning | Action |
|---|---|---|
| `HashAggregate`, `Batches: 1` | Groups fit in memory, single pass | Ideal — nothing to do |
| `HashAggregate`, `Batches > 1`, `Disk Usage` | Hash table spilled to disk | Raise `work_mem` or use index-fed GroupAggregate |
| `GroupAggregate` + `Sort` node below it | Sorting to group, no usable index | Add B-tree index on grouping columns |
| `GroupAggregate` + `Index Scan` (no Sort) | Pre-sorted input, best case for many groups | Ideal for high-cardinality grouping |
| Estimated `rows` for aggregate ≪ actual | Bad `n_distinct` stat → wrong strategy | `ANALYZE`; consider `ALTER TABLE ... ALTER COLUMN ... SET (n_distinct=...)` |

---

## 8. Query Examples

### Example 1 — Basic: Count and Sum per Status

```sql
-- The canonical GROUP BY: one summary row per order status.
SELECT
  o.status,                        -- grouping column (survives collapse)
  COUNT(*)            AS order_cnt,-- rows per group
  SUM(o.total_amount) AS revenue,  -- total per group
  ROUND(AVG(o.total_amount), 2) AS avg_order,  -- mean per group
  MIN(o.created_at)   AS first_seen,
  MAX(o.created_at)   AS last_seen
FROM orders o
GROUP BY o.status                  -- one output row per distinct status
ORDER BY revenue DESC;             -- ORDER BY may reference the aggregate alias
```

### Example 2 — Intermediate: Multi-Column Grouping with HAVING and FILTER

```sql
-- Monthly revenue by status, only for months+statuses that cleared a threshold,
-- with a conditional count of high-value orders folded into the same pass.
SELECT
  DATE_TRUNC('month', o.created_at)              AS month,   -- position 1
  o.status,                                                  -- position 2
  COUNT(*)                                        AS orders,
  COUNT(*) FILTER (WHERE o.total_amount >= 500)   AS big_orders,
  SUM(o.total_amount)                             AS revenue
FROM orders o
WHERE o.created_at >= NOW() - INTERVAL '12 months'  -- WHERE: filter raw rows first
GROUP BY 1, 2                                        -- group by month + status tuple
HAVING SUM(o.total_amount) > 10000                  -- HAVING: keep only big groups
ORDER BY 1, revenue DESC;
```

### Example 3 — Production Grade: Per-Customer Aggregates Without Fan-Out

```sql
-- Customer lifetime report. Tables (sizes for context):
--   users        ~2M rows   (PK users.id)
--   orders       ~20M rows  (index on orders.user_id, orders.status)
--   order_items  ~90M rows  (index on order_items.order_id)
-- NAIVE version (users JOIN orders JOIN order_items, GROUP BY user) DOUBLE-COUNTS
-- orders.total_amount because each order fans out per item. We pre-aggregate items
-- at the order grain, then aggregate orders at the user grain.
-- Perf expectation: index-fed aggregation, ~1–2s on warm cache for the full scan;
-- sub-100ms when filtered to a single user's id.

WITH order_lines AS (          -- collapse items to one row per order (item grain → order grain)
  SELECT
    oi.order_id,
    SUM(oi.quantity)                    AS units,
    SUM(oi.quantity * oi.unit_price)    AS line_revenue
  FROM order_items oi
  GROUP BY oi.order_id
)
SELECT
  u.id,                                 -- PK → functional-dependency exception lets us
  u.name,                               -- select name/email/tier without listing them
  u.email,                             -- (Section 6.5) in GROUP BY
  u.tier,
  COUNT(o.id)                              AS orders,          -- distinct orders (no fan-out now)
  COUNT(o.id) FILTER (WHERE o.status = 'completed') AS completed_orders,
  COALESCE(SUM(o.total_amount), 0)         AS order_total,     -- safe: one row per order here
  COALESCE(SUM(ol.units), 0)               AS total_units,
  COALESCE(SUM(ol.line_revenue), 0)        AS item_revenue,
  MAX(o.created_at)                        AS last_order_at
FROM users u
LEFT JOIN orders o      ON o.user_id = u.id
LEFT JOIN order_lines ol ON ol.order_id = o.id   -- 1:1 with orders after pre-agg → no fan-out
GROUP BY u.id                             -- PK only; Postgres infers name/email/tier
ORDER BY order_total DESC
LIMIT 100;
```

```
-- EXPLAIN (ANALYZE) shape (abbreviated):
Limit  (actual time=1820..1821 rows=100 loops=1)
  ->  Sort  (actual rows=100 loops=1)   Sort Key: (COALESCE(sum(o.total_amount),0)) DESC
        ->  GroupAggregate  (actual rows=2000000 loops=1)
              Group Key: u.id
              ->  Merge Left Join
                    ->  Merge Left Join
                          ->  Index Scan using users_pkey on users u
                          ->  Index Scan using orders_user_id_idx on orders o
                    ->  Subquery Scan on order_lines  (HashAggregate on order_items)
Execution Time: 1834 ms
```

The pre-aggregation CTE (`order_lines`) is the load-bearing part: it converts the item grain to the order grain *before* the join, so the outer `SUM(o.total_amount)` sees each order exactly once.

---

## 9. Wrong → Right Patterns

### Wrong 1: Referencing a non-grouped, non-aggregated column

```sql
-- WRONG
SELECT o.customer_id, o.status, COUNT(*)
FROM orders o
GROUP BY o.customer_id;
-- ERROR: column "o.status" must appear in the GROUP BY clause
--        or be used in an aggregate function
```

**Why (execution level):** grouping by `customer_id` collapses all of a customer's orders into one row, but that customer has orders with different statuses. The engine has multiple `status` values for the one output row and cannot choose. It refuses rather than guess.

```sql
-- RIGHT (option A): add status to the grouping key — one row per (customer, status)
SELECT o.customer_id, o.status, COUNT(*)
FROM orders o
GROUP BY o.customer_id, o.status;

-- RIGHT (option B): aggregate status if you want a per-customer roll-up
SELECT o.customer_id,
       COUNT(*) FILTER (WHERE o.status='completed') AS completed,
       COUNT(*) FILTER (WHERE o.status='cancelled') AS cancelled
FROM orders o
GROUP BY o.customer_id;
```

### Wrong 2: Filtering an aggregate in WHERE

```sql
-- WRONG
SELECT customer_id, SUM(total_amount) AS revenue
FROM orders
WHERE SUM(total_amount) > 1000      -- aggregate in WHERE
GROUP BY customer_id;
-- ERROR: aggregate functions are not allowed in WHERE
```

**Why (execution level):** `WHERE` runs *before* `GROUP BY` (Section 3). At WHERE-time no groups exist, so no `SUM` exists to compare. Aggregate filters belong to `HAVING`, which runs after grouping.

```sql
-- RIGHT
SELECT customer_id, SUM(total_amount) AS revenue
FROM orders
GROUP BY customer_id
HAVING SUM(total_amount) > 1000;    -- filter groups after aggregation
```

### Wrong 3: Summing a parent column across a fanned-out join

```sql
-- WRONG: inflated revenue
SELECT o.customer_id, SUM(o.total_amount) AS revenue
FROM orders o
JOIN order_items oi ON oi.order_id = o.id   -- many items per order
GROUP BY o.customer_id;
-- Each order's total_amount is repeated once per item → over-counted
```

**Why (execution level):** the join multiplies each order row by its item count *before* `GROUP BY` sees it (Topic 11 fan-out). `SUM(o.total_amount)` then adds the same order total multiple times.

```sql
-- RIGHT: pre-aggregate items to order grain, or sum only item-level additive values
SELECT o.customer_id,
       SUM(o.total_amount) AS order_revenue     -- now o is not fanned out
FROM orders o
GROUP BY o.customer_id;                          -- (don't join items at all if you only need order totals)

-- or, if you DO need item detail, pre-aggregate:
SELECT o.customer_id, SUM(o.total_amount) AS order_revenue, SUM(li.units) AS units
FROM orders o
JOIN (SELECT order_id, SUM(quantity) AS units FROM order_items GROUP BY order_id) li
  ON li.order_id = o.id
GROUP BY o.customer_id;
```

### Wrong 4: Expecting a zero-count row from a GROUP BY over no rows

```sql
-- WRONG expectation: "I'll get one row per status with count 0"
SELECT status, COUNT(*) FROM orders WHERE created_at > NOW() GROUP BY status;
-- Returns NO ROWS (future dates match nothing) — not zero-count rows
```

**Why (execution level):** with an explicit `GROUP BY`, groups only exist if rows exist to form them. Empty input → empty groups → empty result.

```sql
-- RIGHT: to guarantee a row per status even at zero, drive from a status dimension
SELECT s.status, COUNT(o.id) AS cnt
FROM (VALUES ('completed'),('cancelled'),('pending')) AS s(status)
LEFT JOIN orders o ON o.status = s.status AND o.created_at > NOW()
GROUP BY s.status;
-- COUNT(o.id) is 0 for statuses with no matching orders (COUNT ignores NULLs)
```

### Wrong 5: Assuming GROUP BY output is ordered

```sql
-- WRONG: relying on the display order of a HashAggregate
SELECT status, COUNT(*) FROM orders GROUP BY status;
-- The row order is whatever the hash-table walk produces — undefined, and it
-- can change between releases, plans, or after work_mem changes.
```

**Why (execution level):** HashAggregate emits groups in hash-bucket order (effectively arbitrary). Only GroupAggregate happens to emit sorted, and you cannot count on which strategy you'll get.

```sql
-- RIGHT: state the order you need
SELECT status, COUNT(*) FROM orders GROUP BY status ORDER BY status;
```

---

## 10. Performance Profile

### Memory and CPU

- **HashAggregate:** memory ≈ (number of distinct groups) × (per-group state size). CPU is one hash + one state update per input row → O(N). Fast and cache-friendly *while it fits in `work_mem`*. The cliff is the spill: once the hash table exceeds `work_mem`, PostgreSQL partitions into batches and writes temp files, adding disk I/O and multiple passes.
- **GroupAggregate:** memory O(1) beyond the sort. If a sort is required, that sort is O(N log N) CPU and may itself spill to disk (`external merge`), governed by the same `work_mem`. If the input is pre-sorted by an index, no sort, no spill — O(N) and O(1) memory.

### Scaling by row count

| Rows | Distinct groups | Likely plan | Expectation |
|---|---|---|---|
| 1M | 10 (e.g. status) | HashAggregate, Batches 1 | tens of ms, trivial |
| 1M | 500K (e.g. user_id) | Index-fed GroupAggregate | ~hundreds of ms; hash would risk spill |
| 10M | 20 | HashAggregate, Batches 1 | ~0.5–2s, dominated by the scan |
| 10M | 5M | Index GroupAggregate | seconds; a hash here spills badly |
| 100M | few | Parallel HashAggregate | seconds w/ parallelism; scan-bound |
| 100M | many | pre-aggregate / rollup table | consider materialized views |

The governing variables are **number of distinct groups** (drives hash-table size) and **whether an index provides sorted input** (removes the sort/hash memory pressure entirely).

### Index interactions

- A **B-tree index on the grouping columns (in order)** lets the planner feed a `GroupAggregate` from a sorted `Index Scan` — O(1) memory, no spill, and pre-sorted output that can satisfy `ORDER BY <same cols>` for free. This is the single most effective optimization for high-cardinality grouping.
- An **expression index** (e.g. `CREATE INDEX ON orders (DATE_TRUNC('month', created_at))`) is required for pre-sorted grouping on a computed value; without it the engine must sort/hash the derived value.
- A **covering index** including the aggregated columns (`INCLUDE (total_amount)`) can enable an index-only scan, avoiding heap fetches entirely for the aggregation.
- **Partial indexes** matching a hot `WHERE` (`... WHERE status='completed'`) shrink the scanned set before grouping.

### Optimization techniques specific to GROUP BY

1. **Right-size `work_mem`** for the query. If EXPLAIN shows `Batches > 1` on a HashAggregate, `SET work_mem='256MB'` (session-scoped) often turns minutes into seconds.
2. **Add an index on grouping columns** to switch a spilling HashAggregate to a streaming GroupAggregate.
3. **Enable parallel aggregation** — PostgreSQL can split the scan across workers, each doing a `Partial HashAggregate`, then a `Finalize HashAggregate` combines them. Ensure `max_parallel_workers_per_gather > 0` and the table is large enough to qualify.
4. **Pre-aggregate to avoid fan-out** (Section 6.10 / Example 3) — smaller intermediate rows, correct numbers.
5. **Materialize expensive rollups** — for dashboards hitting 100M-row tables repeatedly, a `MATERIALIZED VIEW` or a summary table refreshed on a schedule turns an O(N) scan into an O(groups) lookup.
6. **`SET enable_hashagg = off`** as a *diagnostic* to force GroupAggregate and compare — never as a permanent production setting.

---

## 11. Node.js Integration

### 11.1 Basic grouped query with `pg`

```javascript
import pg from 'pg';
const { Pool } = pg;
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

async function revenueByStatus(sinceDays = 90) {
  const { rows } = await pool.query(
    `SELECT
       status,
       COUNT(*)            AS order_cnt,
       SUM(total_amount)   AS revenue,
       ROUND(AVG(total_amount), 2) AS avg_order
     FROM orders
     WHERE created_at >= NOW() - ($1 || ' days')::INTERVAL
     GROUP BY status
     ORDER BY revenue DESC`,
    [sinceDays]
  );
  // NOTE: SUM/COUNT come back as STRINGS in node-postgres (bigint/numeric are
  // not safely representable as JS numbers). Parse explicitly.
  return rows.map(r => ({
    status: r.status,
    orderCnt: Number(r.order_cnt),
    revenue: Number(r.revenue),
    avgOrder: Number(r.avg_order),
  }));
}
```

The bigint/numeric-as-string behavior is the #1 Node + `GROUP BY` gotcha: `COUNT(*)` returns a JS string like `"40213"`, and `SUM` of a `numeric` column returns a string to preserve precision. Always coerce (`Number(...)`, or a decimal library for money) rather than assuming numbers.

### 11.2 Multi-column grouping with a conditional aggregate

```javascript
async function monthlyStatusBreakdown(months = 12) {
  const { rows } = await pool.query(
    `SELECT
       DATE_TRUNC('month', created_at)            AS month,
       COUNT(*)                                    AS orders,
       COUNT(*) FILTER (WHERE status='completed')  AS completed,
       COUNT(*) FILTER (WHERE status='cancelled')  AS cancelled,
       COALESCE(SUM(total_amount) FILTER (WHERE status='completed'), 0) AS revenue
     FROM orders
     WHERE created_at >= NOW() - ($1 || ' months')::INTERVAL
     GROUP BY 1
     ORDER BY 1`,
    [months]
  );
  return rows;
}
```

### 11.3 Guarding the empty-set asymmetry

```javascript
async function customerLifetimeValue(customerId) {
  const { rows } = await pool.query(
    `SELECT
       COUNT(*)                       AS orders,       -- 0 for a customer with none
       COALESCE(SUM(total_amount), 0) AS lifetime_value -- NULL→0 without COALESCE
     FROM orders
     WHERE customer_id = $1 AND status = 'completed'`,  -- NO GROUP BY → always 1 row
    [customerId]
  );
  // Because there is no GROUP BY, this ALWAYS returns exactly one row, even if the
  // customer has zero orders (orders=0, lifetime_value=0 thanks to COALESCE).
  return rows[0];
}
```

### 11.4 Do ORMs handle GROUP BY?

Partially. Simple single-column groupings with one aggregate are expressible in every ORM's query builder. But `COUNT(DISTINCT)`, `FILTER (WHERE ...)`, `HAVING` on complex expressions, `DATE_TRUNC` grouping, and the fan-out-safe pre-aggregation pattern almost always push you to raw SQL (`pool.query`, `$queryRaw`, `knex.raw`). Treat aggregation as the boundary where ORMs stop helping.

---

## 12. ORM Comparison

### Prisma

**Can Prisma do GROUP BY?** — Yes, via `groupBy`, for straightforward cases.

```typescript
const byStatus = await prisma.order.groupBy({
  by: ['status'],
  where: { createdAt: { gte: new Date('2025-01-01') } },
  _count: { _all: true },
  _sum: { totalAmount: true },
  _avg: { totalAmount: true },
  having: { totalAmount: { _sum: { gt: 1000 } } },  // HAVING on an aggregate
  orderBy: { _sum: { totalAmount: 'desc' } },
});
```

**Where it breaks:** no `COUNT(DISTINCT col)`, no `FILTER (WHERE ...)`, no grouping by an *expression* like `DATE_TRUNC('month', createdAt)` (you can only group by scalar columns). Time-bucketed reports, conditional aggregation, and the functional-dependency PK shortcut are all out of reach.

```typescript
// Escape hatch for expression grouping / conditional aggregates:
const monthly = await prisma.$queryRaw`
  SELECT DATE_TRUNC('month', created_at) AS month,
         COUNT(*) FILTER (WHERE status='completed') AS completed,
         SUM(total_amount) AS revenue
  FROM orders GROUP BY 1 ORDER BY 1`;
```

**Verdict:** great for column-grouped counts/sums with HAVING; drop to `$queryRaw` for expression grouping, `COUNT(DISTINCT)`, or `FILTER`.

### Drizzle ORM

**Can Drizzle do GROUP BY?** — Yes, and it's the most SQL-faithful of the typed ORMs.

```typescript
import { db } from './db';
import { orders } from './schema';
import { sql, gte, desc } from 'drizzle-orm';

const monthly = await db
  .select({
    month: sql<string>`DATE_TRUNC('month', ${orders.createdAt})`.as('month'),
    orders: sql<number>`COUNT(*)`,
    completed: sql<number>`COUNT(*) FILTER (WHERE ${orders.status} = 'completed')`,
    revenue: sql<number>`SUM(${orders.totalAmount})`,
  })
  .from(orders)
  .where(gte(orders.createdAt, new Date('2025-01-01')))
  .groupBy(sql`DATE_TRUNC('month', ${orders.createdAt})`)   // expression grouping OK
  .having(sql`SUM(${orders.totalAmount}) > 10000`)
  .orderBy(desc(sql`SUM(${orders.totalAmount})`));
```

**Where it breaks:** essentially nowhere for expressiveness — but you drop into the `sql` template tag for expression grouping, `FILTER`, and `COUNT(DISTINCT)`, so type inference on those columns is only as good as your annotation.

**Verdict:** best typed option for real aggregation; `.groupBy(sql\`...\`)` handles expressions and `.having()` handles aggregate filters natively.

### Sequelize

**Can Sequelize do GROUP BY?** — Yes, via `group`, `attributes` with `fn`, and `having`.

```javascript
const { fn, col, literal } = require('sequelize');
const rows = await Order.findAll({
  attributes: [
    'status',
    [fn('COUNT', col('id')), 'orderCnt'],
    [fn('SUM', col('total_amount')), 'revenue'],
  ],
  where: { createdAt: { [Op.gte]: new Date('2025-01-01') } },
  group: ['status'],
  having: literal('SUM(total_amount) > 1000'),
  order: [[literal('revenue'), 'DESC']],
  raw: true,
});
```

**Where it breaks:** expression grouping and `FILTER` require `literal()` strings (no type safety, injection risk if interpolated). Sequelize also famously adds every selected column to the generated `GROUP BY`/`ORDER BY` in some association scenarios, producing unexpected groupings. `raw: true` is usually needed to get clean aggregate rows.

**Verdict:** workable for simple groupings; anything with expressions or conditional aggregates leans on `literal()` or `sequelize.query()`.

### TypeORM

**Can TypeORM do GROUP BY?** — Yes, via QueryBuilder `groupBy`/`addGroupBy`/`having` with `getRawMany()`.

```typescript
const rows = await dataSource
  .getRepository(Order)
  .createQueryBuilder('o')
  .select('o.status', 'status')
  .addSelect('COUNT(*)', 'orderCnt')
  .addSelect('SUM(o.total_amount)', 'revenue')
  .where('o.created_at >= :since', { since: new Date('2025-01-01') })
  .groupBy('o.status')
  .having('SUM(o.total_amount) > :min', { min: 1000 })
  .orderBy('revenue', 'DESC')
  .getRawMany();   // aggregates are NOT entity fields → must use getRawMany
```

**Where it breaks:** aggregates don't map to entity properties, so `getMany()` won't return them — you must use `getRawMany()` and read raw column aliases. Expression grouping is raw strings with no type checking. `FILTER` and `COUNT(DISTINCT)` are raw SQL fragments.

**Verdict:** capable via QueryBuilder + `getRawMany`; remember aggregation lives outside the entity model.

### Knex.js

**Can Knex do GROUP BY?** — Yes, the most transparent mapping to SQL.

```javascript
const rows = await knex('orders')
  .select('status')
  .count('* as order_cnt')
  .sum('total_amount as revenue')
  .where('created_at', '>=', new Date('2025-01-01'))
  .groupBy('status')
  .havingRaw('SUM(total_amount) > ?', [1000])
  .orderBy('revenue', 'desc');

// Expression grouping + FILTER via raw:
const monthly = await knex('orders')
  .select(knex.raw(`DATE_TRUNC('month', created_at) AS month`))
  .select(knex.raw(`COUNT(*) FILTER (WHERE status = 'completed') AS completed`))
  .groupByRaw(`DATE_TRUNC('month', created_at)`)
  .orderByRaw('month');
```

**Where it breaks:** expression grouping needs `groupByRaw`; `FILTER`/`COUNT(DISTINCT)` need `knex.raw`. No type safety on raw fragments, but the SQL is exactly what you wrote.

**Verdict:** thinnest layer over SQL — `groupBy`/`havingRaw`/`groupByRaw` give full control.

### ORM summary table

| ORM | GROUP BY method | Expression grouping | FILTER / COUNT(DISTINCT) | HAVING | Verdict |
|---|---|---|---|---|---|
| Prisma | `groupBy({ by })` | No (scalar cols only) | Raw SQL | `having` object | Simple cases only |
| Drizzle | `.groupBy(sql\`...\`)` | Yes via `sql` | Yes via `sql` | `.having(sql\`...\`)` | Best typed option |
| Sequelize | `group` + `fn` | `literal()` | `literal()` | `having: literal()` | Simple groupings |
| TypeORM | `.groupBy()` + `getRawMany` | Raw string | Raw string | `.having()` | Raw-mode aggregation |
| Knex | `.groupBy()` / `groupByRaw` | `groupByRaw` | `knex.raw` | `havingRaw` | Most transparent |

---

## 13. Practice Exercises

### Exercise 1 — Basic

Given `orders(id, customer_id, status, total_amount, created_at)`:

Write a query returning, for each `status`: the number of orders, the total revenue, the average order value (rounded to 2 decimals), and the most recent `created_at`. Order the result by total revenue descending. Then answer: if the `orders` table were empty, how many rows would your query return, and why?

```sql
-- Write your query here
```

### Exercise 2 — Medium (combines Topic 11 JOIN + GROUP BY)

Given `orders(id, customer_id, status, total_amount, created_at)` and `order_items(id, order_id, product_id, quantity, unit_price)`:

For each `customer_id`, return: the number of **distinct completed orders**, the total order revenue (order-level `total_amount`, counted once per order — beware fan-out), and the total units sold across those orders. Only include customers with at least 3 completed orders. Order by revenue descending.

Hint: you cannot naively `JOIN order_items` and then `SUM(o.total_amount)` — explain in a comment why, then write the correct pre-aggregated version.

```sql
-- Write your query here
```

### Exercise 3 — Hard (production simulation; naive answer is wrong)

Given `sessions(id, user_id, started_at, duration_seconds, device_type)` with 40M rows and 8M distinct `user_id` values, and `work_mem = '4MB'`:

Write a query returning, per `device_type`, the number of sessions, distinct users, and average duration for the last 30 days. Then: the same aggregation grouped by `user_id` (8M groups) is timing out with a spilling HashAggregate. Explain what EXPLAIN would show, and give two concrete fixes (one indexing, one configuration). The naive `GROUP BY user_id` without addressing the strategy is the wrong production answer.

```sql
-- Write your query here (both the device_type report and the fix for the user_id version)
```

### Exercise 4 — Interview Simulation

You're handed this query and told "revenue looks about 3x too high":

```sql
SELECT c.id, c.name, SUM(o.total_amount) AS revenue, COUNT(oi.id) AS items
FROM categories c
JOIN products p     ON p.category_id = c.id
JOIN order_items oi ON oi.product_id = p.id
JOIN orders o       ON o.id = oi.order_id
GROUP BY c.id, c.name;
```

Identify every bug (there is at least one fan-out bug and one grouping-correctness concern), explain each at the execution level, and rewrite it to correctly report, per category: distinct orders, total item revenue (`quantity * unit_price`), and distinct products sold.

```sql
-- Write your query here
```

---

## 14. Interview Questions

### Q1 — Why must every SELECT column be in GROUP BY or an aggregate?

**Junior answer:** "Because SQL requires it / otherwise you get an error."

**Principal answer:** Because `GROUP BY` produces exactly one output row per group, and a non-aggregated column that isn't part of the grouping key can have *many different values* within a single group — the engine would have to pick one arbitrarily, which is unsafe, so the SQL standard forbids it. The formal statement is that every non-aggregated SELECT expression must be **functionally determined** by the grouping columns. Being *in* the GROUP BY is the trivial way to be determined; PostgreSQL also accepts columns determined by a grouped **primary key**. Wrapping the column in an aggregate resolves it by explicitly telling the engine how to collapse the many values into one.

**Follow-up:** "So why does `GROUP BY u.id` let me select `u.name` without listing it?" → Because `id` is the primary key, so `name` is functionally dependent on it — within any group there's exactly one `name`. PostgreSQL proves this from the PK constraint (Section 6.5).

### Q2 — What's the difference between WHERE and HAVING, and could you always use one instead of the other?

**Junior answer:** "WHERE filters rows, HAVING filters groups; HAVING is for aggregates."

**Principal answer:** `WHERE` runs *before* `GROUP BY` and filters individual raw rows; it cannot reference aggregates because no groups exist yet. `HAVING` runs *after* grouping and filters whole groups, and it's the only place an aggregate predicate is legal. They are not interchangeable: a row-level predicate belongs in `WHERE` (it removes rows before they're grouped, which is cheaper and often changes the aggregate result); an aggregate predicate can only go in `HAVING`. Putting a non-aggregate predicate in `HAVING` is legal but wasteful — the row still enters a group before being discarded — and if any aggregate depends on those rows, the numbers differ. Rule of thumb: filter as early as possible; use `HAVING` only for conditions on the aggregated result.

**Follow-up:** "Given `WHERE status='x'` vs `HAVING bool_and(status='x')` — same result?" → Not necessarily; `WHERE` reshapes what enters each group and thus the other aggregates, while HAVING filters after the aggregates are already computed over all rows.

### Q3 — A GROUP BY query on 50M rows suddenly got 10x slower after data growth. Walk me through diagnosis.

**Junior answer:** "Add an index / increase memory."

**Principal answer:** First `EXPLAIN (ANALYZE, BUFFERS)`. The signature is a `HashAggregate` that now shows `Batches > 1` and `Disk Usage` — the distinct-group count grew past what fits in `work_mem`, so the hash table spilled to temp files, adding disk I/O and multiple passes. Root cause is high (and growing) group cardinality against a fixed memory budget, sometimes compounded by a stale `n_distinct` estimate that made the planner pick HashAggregate for what's now far more groups. Fixes, in order: (1) `ANALYZE` to refresh statistics; (2) create a B-tree index on the grouping columns so the planner uses an index-fed `GroupAggregate` — O(1) memory, no spill, and pre-sorted output; (3) if hashing is genuinely best, raise `work_mem` for the query session; (4) enable/verify parallel aggregation for the scan; (5) for repeated dashboard hits, precompute with a materialized view or summary table. I'd verify each change by re-reading the plan, not by guessing.

**Follow-up:** "How do you force GroupAggregate to confirm your hypothesis?" → `SET enable_hashagg = off` for the session as a diagnostic, compare timings, then implement the index rather than leaving the flag off in production.

### Q4 — What does a GROUP BY return over an empty input, and why is that sometimes surprising?

**Junior answer:** "Zero rows / it returns 0."

**Principal answer:** It depends on whether there's an explicit `GROUP BY`. With `GROUP BY status` over zero input rows you get **zero output rows** — no rows means no groups to form. But an aggregate with **no** `GROUP BY` (`SELECT COUNT(*) FROM t WHERE false`) treats the whole result as one implicit group and returns **exactly one row**: `COUNT` gives `0`, but `SUM`/`AVG`/`MAX`/`MIN` give `NULL` (the aggregate of an empty set). This asymmetry bites when code expects a row-with-zero from a grouped query, or a numeric zero from `SUM` — wrap in `COALESCE(SUM(x),0)`.

**Follow-up:** "How would you guarantee a zero-count row per status?" → Drive the query from a status dimension (`VALUES` list or a lookup table) `LEFT JOIN`ed to the facts, then `COUNT(fact.id)` which yields 0 for unmatched dimension rows.

---

## 15. Mental Model Checkpoint

1. You `GROUP BY customer_id` but also `SELECT o.status` without aggregating it. Explain, in terms of "one output row per group," exactly why this errors — and give two different correct fixes with different meanings.

2. A table has 1M rows and 4 distinct values in the grouping column. Which aggregation strategy will the planner almost certainly choose, how much memory will it use, and will the output be sorted?

3. The same table is grouped by a column with 900K distinct values, with a B-tree index on that column. Which strategy is now preferable and why, and what happens to memory usage compared to question 2?

4. `SELECT SUM(amount) FROM payments WHERE 1=0` and `SELECT SUM(amount) FROM payments WHERE 1=0 GROUP BY method` — how many rows does each return, and what value(s)? Explain the asymmetry.

5. You join `orders` to `order_items` (many per order) and write `SUM(orders.total_amount)` grouped by customer. Trace what the engine actually sums and why the number is wrong. What is the reliable fix?

6. Why can `GROUP BY u.id` (a primary key) let you select `u.name`, `u.email`, and `u.tier` unlisted, but `GROUP BY u.email` cannot let you select `u.id`? What property is PostgreSQL relying on?

7. You wrote `GROUP BY 1, 2` in an ad-hoc query. A colleague reorders the SELECT list during a refactor and the report silently changes. Explain the failure mode and what you'd use instead in maintained code.

---

## 16. Quick Reference Card

```sql
-- CORE SHAPE
SELECT grouping_cols, AGG(other_cols)
FROM ...  WHERE row_filter          -- runs BEFORE grouping
GROUP BY grouping_cols
HAVING agg_condition                -- runs AFTER grouping (aggregates only here)
ORDER BY ...;                        -- can reference aggregates & aliases

-- THE RULE: every non-aggregated SELECT column must be a grouping column
--           (or functionally determined by a grouped PRIMARY KEY)

-- NULLs form ONE group (unlike '=' which treats NULL as never-equal)
GROUP BY department_id               -- all NULL depts → single bucket

-- MULTI-COLUMN = group by the TUPLE that actually occurs (not Cartesian)
GROUP BY status, region              -- one row per observed (status, region)

-- EXPRESSION grouping (repeat expr, or use alias/ordinal)
GROUP BY DATE_TRUNC('month', created_at)
GROUP BY month                       -- Postgres: SELECT alias allowed (non-standard)
GROUP BY 1, 2                        -- ordinal positions (fragile on SELECT reorder)

-- PK functional-dependency exception (Postgres/SQL-99)
SELECT u.id, u.name, u.email, COUNT(o.id)
FROM users u LEFT JOIN orders o ON o.user_id=u.id
GROUP BY u.id;                       -- PK only; name/email inferred

-- CONDITIONAL aggregation in one pass
COUNT(*) FILTER (WHERE status='completed')
SUM(amount) FILTER (WHERE status='completed')

-- EMPTY-SET asymmetry
SELECT COUNT(*)  FROM t WHERE false;            -- 1 row, value 0
SELECT SUM(x)    FROM t WHERE false;            -- 1 row, value NULL (use COALESCE)
SELECT x, COUNT(*) FROM t WHERE false GROUP BY x;-- 0 rows

-- FAN-OUT: GROUP BY runs after JOIN → parent columns get double-counted
--   Fix: pre-aggregate the child to parent grain BEFORE joining.

-- PLAN SIGNALS
-- HashAggregate  Batches:1     → ideal (groups fit work_mem)
-- HashAggregate  Batches:>1    → spilling; raise work_mem OR index the group cols
-- GroupAggregate + Index Scan  → best for many groups (O(1) mem, pre-sorted)
-- GroupAggregate + Sort node   → add B-tree index on grouping columns

-- INTERVIEW ONE-LINERS
-- "One output row per group; that's why non-grouped columns error."
-- "WHERE filters rows before grouping; HAVING filters groups after."
-- "HashAggregate = one pass, memory ∝ #groups; GroupAggregate = sorted, O(1) mem."
-- "Empty GROUP BY = zero rows; empty implicit group = one row (SUM→NULL, COUNT→0)."
-- "GROUP BY output is UNORDERED unless you ORDER BY."
```

---

## Connected Topics

- **Topic 07 — NULL in Depth**: why NULLs form a single group under "not distinct from" semantics, unlike `=`.
- **Topic 11 — INNER JOIN in Depth**: the fan-out that `GROUP BY` collapses; join-then-aggregate is one inseparable pattern, and `COUNT(DISTINCT)` exists to undo the multiplication.
- **Topic 19 — JOIN Alternatives**: `EXISTS` and semi-joins avoid the row multiplication that forces defensive `COUNT(DISTINCT)` in grouped queries.
- **Topic 21 — Aggregate Functions in Depth (next)**: the functions that collapse each group — `COUNT` vs `COUNT(DISTINCT)`, `SUM`/`AVG` NULL handling, `STRING_AGG`, `ARRAY_AGG`, ordered-set and `FILTER` aggregates in full.
- **Topic 22 — HAVING and Post-Aggregation Filtering**: the group-level filter introduced here, covered exhaustively.
- **Internals this builds on**: HashAggregate vs. GroupAggregate, `work_mem` and hash spill, B-tree index-fed sorted input, the planner's `n_distinct` statistics, and parallel partial/finalize aggregation.
