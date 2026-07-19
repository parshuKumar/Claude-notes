# Topic 77 — Canonical Hard SQL Problems

### SQL Mastery Curriculum — Phase 11: SQL for Backend Developer Interviews

---

## 1. ELI5 — The Analogy That Makes It Click

Imagine you're a master locksmith who has just finished a five-year apprenticeship. Someone hands you a ring with ten locks on it and says: "These are the ten locks that show up in every real building. Open all ten and you can call yourself a locksmith."

That's what this topic is. Over the previous 76 topics you learned every individual skill — window functions, CTEs, joins, indexes, NULLs, `DISTINCT ON`, `LATERAL`, recursion. But interviewers and production systems don't ask you "explain `ROW_NUMBER()`." They hand you a *lock*: "Give me the top 3 products per category." "Find users with a 7-day login streak." "Split these raw pageviews into sessions." Each lock has a *classic key* — a pattern that senior engineers recognise on sight and reach for without hesitation.

There are exactly ten locks that show up again and again:

1. **Top-N per group** — "best 3 of each kind"
2. **Running totals & moving averages** — "cumulative sum, rolling 7-day average"
3. **Gaps and islands** — "find the missing invoice numbers / the unbroken runs"
4. **Consecutive-day streaks** — "longest login streak" (a special case of islands)
5. **Hierarchical data** — "the whole org chart under this manager"
6. **Sessionisation** — "group events into sessions with a 30-min timeout"
7. **Deduplication** — "keep one row per natural key, drop the rest"
8. **Median & percentiles** — "the p50 and p95 order value"
9. **Pivot without crosstab** — "months as columns, one row per product"
10. **Self-referencing queries** — "employees who earn more than their manager"

A junior stares at each of these and reinvents a broken wheel. A senior sees the lock, names the pattern out loud, and turns the key. This topic gives you all ten keys, filed and oiled. Learn to *recognise the lock* — that recognition is 80% of the interview.

---

## 2. Connection to SQL Internals

Every one of these ten problems is, underneath, one of a small set of physical operations the engine already knows how to do. Recognising the internal machinery tells you *why* the classic pattern is the fast one.

- **Top-N per group, dedup, streaks, islands** all lower to a **WindowAgg** node sitting on top of a **Sort** (or an index that provides the sort order for free). The window function `ROW_NUMBER()`/`RANK()`/`LAG()` is computed by scanning a partition-ordered stream once. The cost is dominated by the sort: `O(N log N)` unless a B-tree index already delivers rows in `PARTITION BY, ORDER BY` order, in which case it collapses to `O(N)` with `Incremental Sort` or no sort at all.

- **Running totals & moving averages** are the same **WindowAgg**, but with a moving **frame** (`ROWS`/`RANGE`/`GROUPS BETWEEN ...`). Postgres maintains a running aggregate state as the frame slides. `ROWS` frames are cheap (pointer arithmetic on the sorted buffer); `RANGE` frames with peer groups need extra lookahead; `RANGE` with `EXCLUDE` or a moving lower bound may recompute the aggregate per row — an `O(N·W)` trap.

- **Hierarchical data** uses a **Recursive Union** node driving a **worktable** (a temp tuplestore). Each iteration reads the previous iteration's output from the worktable, joins it to the base table (ideally via an index on `parent_id`), and appends new rows. This is a fixpoint computation — it loops until an iteration produces zero rows. Closure tables trade this recursion at read-time for extra rows and write-time maintenance.

- **Sessionisation** is `LAG()` (WindowAgg) to detect gaps, then a **running SUM over a boolean flag** (another WindowAgg) to assign session IDs — two window passes over the same sort. The engine can share the sort between them.

- **Median / `percentile_cont`** is an **ordered-set aggregate**. Internally it buffers the whole group into a **tuplesort**, then interpolates at the requested fraction. It is materialising and memory-hungry: the entire partition lands in `work_mem` (or spills to a temp file on disk).

- **Pivot** is a **HashAggregate** with conditional aggregates (`SUM(CASE WHEN ...)`), or Postgres's `crosstab()` from the `tablefunc` extension. The `CASE` approach is a single grouped scan; no special node beyond the aggregate.

The unifying internal fact: **eight of the ten reduce to a Sort followed by a WindowAgg.** If you internalise "these are all window-function problems riding on a sort, and the sort is the cost," you can predict the plan and the index that kills the sort before you even run `EXPLAIN`.

---

## 3. Logical Execution Order Context

These patterns live at the *late* end of the logical execution pipeline, which is exactly why they trip people up:

```
FROM / JOIN          ← assemble the raw rows (recursion happens here for CTEs)
WHERE                ← row filters BEFORE any window sees the data
GROUP BY             ← collapse into groups (median/pivot aggregates run here)
HAVING               ← filter groups
SELECT               ← window functions computed HERE (ROW_NUMBER, LAG, SUM OVER)
DISTINCT / DISTINCT ON
ORDER BY
LIMIT / OFFSET
```

Two consequences dominate every hard problem in this topic:

1. **Window functions run in the SELECT phase — after WHERE, after GROUP BY.** You therefore *cannot* filter on a window result in the same query's `WHERE` clause. `WHERE ROW_NUMBER() OVER (...) = 1` is a syntax error. Every "top-N per group" and "dedup" solution must wrap the window in a subquery/CTE and filter in the *outer* query's `WHERE`. This single ordering fact is the reason the classic pattern is always two-level.

2. **`WHERE` filters before the window sees the data, so filtering changes the window's universe.** If you `WHERE status = 'completed'` before a running total, the running total is over completed rows only. If you wanted the running total over *all* rows but to *display* only completed ones, you must compute the window first (inner query) and filter in the outer query. The place you put a predicate relative to the window boundary is a correctness decision, not a style choice.

Recursive CTEs are a special case: the recursion is fully evaluated in the `FROM` phase before the outer `WHERE`/`SELECT` ever runs, so the recursive term's own `WHERE` is the only place to prune branches early (prune inside the recursion, not outside — pruning outside still walks the whole tree).

---

## 4. What Is a Canonical Hard Problem?

A canonical hard problem is a query shape that (a) recurs across essentially every business domain, (b) has a *known-correct classic pattern* that a senior applies by reflex, and (c) has at least one seductive naive solution that is either wrong or catastrophically slow. Mastery means recognising the shape, naming the pattern, and reproducing the key from memory.

Here is the annotated skeleton of the single most important key — **top-N per group** — with every clause explained. The other nine are variations on this same anatomy.

```sql
SELECT product_id, category_id, revenue          -- ⑦ final projection, window already resolved
FROM (
  SELECT
    p.product_id,                                --   the row's identity
    p.category_id,                               -- │ the PARTITION key (one ranking per category)
    p.revenue,                                   -- │ the metric we rank by
    ROW_NUMBER() OVER (                          -- ① the ranking window function
      PARTITION BY p.category_id                 -- │ └── restart the counter for each category
      ORDER BY p.revenue DESC                    -- │ └── 1 = highest revenue within the category
    ) AS rn                                       -- │     alias so the outer query can filter it
  FROM products p                                -- ② source rows
  WHERE p.active                                 -- ③ pre-window filter: shrinks the ranked set
) ranked                                          -- ④ subquery: the window is now a plain column
WHERE ranked.rn <= 3                              -- ⑤ THE filter — impossible in the inner WHERE
ORDER BY ranked.category_id, ranked.rn;           -- ⑥ present grouped, best-first
```

```
①  ROW_NUMBER()  → strict 1,2,3… no ties share a number (arbitrary tiebreak)
   RANK()        → ties share a number, then a GAP (1,1,3) — use for "top-N including ties"
   DENSE_RANK()  → ties share, NO gap (1,1,2) — use for "top-N distinct values"
⑤  rn <= N       → the N in top-N. ROW_NUMBER caps the count exactly;
                   RANK/DENSE_RANK may return MORE than N rows when ties exist at the boundary
```

The three choices — which ranking function, what `ORDER BY` inside `OVER`, and `<= N` versus `= 1` — are the entire decision surface. Every hard problem in this topic is "pick the right window function, pick the right frame, wrap and filter."

---

## 5. Why Mastering These Matters in Production

These aren't academic puzzles — they are the queries that back real product features and real 3 a.m. incidents:

1. **They are the features.** "Top 3 products per category" is a homepage widget. "Rolling 7-day active users" is the North Star metric on every dashboard. "Login streak" is a gamification feature shipped by Duolingo, Snapchat, GitHub. "Sessionise pageviews" is how every analytics product bills. If you can't write these, you can't build the product.

2. **The naive version is a correctness bug that ships silently.** A `GROUP BY` "top-N" that returns the wrong row, a median computed as `AVG`, a dedup that keeps a random row instead of the newest — these produce *plausible-looking* wrong numbers. They pass code review, pass the demo, and corrupt a quarterly report three months later. The cost is trust, not a stack trace.

3. **The naive version is an `O(N²)` performance bomb.** The classic "greatest-N-per-group with a correlated subquery" (`WHERE revenue = (SELECT MAX(revenue) ...)`) runs a subquery per row. It's instant on 1,000 test rows and takes 40 minutes on 10M production rows. The window-function key is `O(N log N)`. This is the single most common "it worked in staging" outage in analytics code.

4. **Interviewers screen on exactly these.** Senior and staff SQL interviews are not "write a JOIN." They are "here's an events table, give me sessions" or "here's a logins table, longest streak." The interviewer is checking whether you *recognise the lock*. Fumbling gaps-and-islands is a near-automatic no-hire for a data-heavy backend role.

5. **They compose.** Real dashboards stack them: sessionise → dedup → top-N session per user → running total of session count. If each sub-pattern isn't automatic, the composite is impossible. This capstone exists so the whole stack becomes reflex.

---

## 6. Deep Technical Content — The Ten Canonical Problems

This is the heart of the capstone. Each problem gets: the shape, the classic pattern, the key variations, and the traps.

### 6.1 Top-N Per Group (Greatest-N-Per-Group)

**Shape:** "For each group, return the N rows with the highest/lowest value of some metric."

**Classic pattern — window function (the default):**

```sql
-- Top 3 highest-value orders per customer
SELECT customer_id, order_id, total_amount
FROM (
  SELECT customer_id, id AS order_id, total_amount,
         ROW_NUMBER() OVER (PARTITION BY customer_id
                            ORDER BY total_amount DESC, id) AS rn
  FROM orders
) t
WHERE rn <= 3;
```

**Which ranking function?**

| Function | Ties at the boundary | Use when |
|----------|---------------------|----------|
| `ROW_NUMBER()` | Exactly N rows, arbitrary tiebreak | "Give me at most N rows, I don't care which of the tied ones" |
| `RANK()` | May return >N (all ties at rank N included), gaps after | "Top N *including everyone tied for Nth place*" |
| `DENSE_RANK()` | Top N *distinct values* of the metric | "The 3 highest distinct prices and everything at them" |

**Always add a deterministic tiebreaker** to the `ORDER BY` inside `OVER` (e.g. `, id`). Without it, `ROW_NUMBER()` picks arbitrarily among ties and results are non-reproducible across runs — a classic flaky-test cause.

**The `N = 1` special case — `DISTINCT ON` (Postgres-native, often faster):**

```sql
-- The single most recent order per customer
SELECT DISTINCT ON (customer_id)
       customer_id, id AS order_id, created_at, total_amount
FROM orders
ORDER BY customer_id, created_at DESC;   -- ORDER BY must LEAD with the DISTINCT ON key
```

`DISTINCT ON` keeps the first row per leading-column group in the `ORDER BY`. It's terse and lets the planner use an index on `(customer_id, created_at DESC)` for a near-free scan. Limitation: only `N = 1`. For `N > 1` you need `ROW_NUMBER()` or a `LATERAL` join.

**The `LATERAL` pattern (great when N is small and an index exists):**

```sql
-- Top 3 orders per customer, using an index-backed correlated top-N
SELECT c.id, o.order_id, o.total_amount
FROM customers c
CROSS JOIN LATERAL (
  SELECT id AS order_id, total_amount
  FROM orders
  WHERE orders.customer_id = c.id
  ORDER BY total_amount DESC
  LIMIT 3                               -- index on (customer_id, total_amount DESC) → 3 index reads/customer
) o;
```

With a matching index this is the *fastest* option when the number of groups (customers) is small relative to total rows, because it never sorts the whole table — it does `groups × N` index lookups. When groups are many and dense, the single-sort `ROW_NUMBER()` wins.

**Trap:** doing top-N with `GROUP BY customer_id HAVING total_amount = MAX(...)` — you cannot, because the other columns (`order_id`, `created_at`) aren't functionally dependent on the group key; you'd have to self-join back, which is the slow correlated-subquery anti-pattern.

---

### 6.2 Running Totals & Moving Averages

**Shape:** "Cumulative sum to date" / "rolling 7-day average" / "each row's share of a partition total."

**Classic pattern — window aggregate with an explicit frame:**

```sql
SELECT
  order_date,
  daily_revenue,
  -- Running total: default frame is RANGE UNBOUNDED PRECEDING → CURRENT ROW
  SUM(daily_revenue) OVER (ORDER BY order_date) AS cumulative_revenue,
  -- 7-day moving average: last 7 ROWS including current
  AVG(daily_revenue) OVER (ORDER BY order_date
                           ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS ma7,
  -- Share of grand total: whole-partition frame
  ROUND(daily_revenue
        / SUM(daily_revenue) OVER () * 100, 2) AS pct_of_total
FROM daily_sales
ORDER BY order_date;
```

**`ROWS` vs `RANGE` — the single most important frame distinction:**

- `ROWS BETWEEN n PRECEDING AND CURRENT ROW` counts *physical rows*. "Last 7 rows."
- `RANGE BETWEEN ...` counts *logical peers by ORDER BY value*. With ties, **all peer rows collapse into one frame position** — a running total with the default `RANGE` frame gives every tied row the *same* cumulative value (the total through the end of the tie group), which surprises people expecting per-row accumulation.

```sql
-- Two orders on the same date:
-- ROWS  frame → each gets its own incremental running total
-- RANGE frame → both get the SAME running total (sum through that whole date)
SUM(amt) OVER (ORDER BY d ROWS  UNBOUNDED PRECEDING)   -- per-row
SUM(amt) OVER (ORDER BY d RANGE UNBOUNDED PRECEDING)   -- per-date (peers merged)
```

**Rule of thumb: use `ROWS` for running totals unless you specifically want peer-grouping semantics.** The default frame when you write only `ORDER BY` (no frame clause) is `RANGE UNBOUNDED PRECEDING AND CURRENT ROW` — the peer-merging one — so *always spell out `ROWS`* for a true running total.

**Genuine time-window moving averages** ("average over the last 7 *calendar* days, gaps allowed") need `RANGE` with an interval:

```sql
AVG(daily_revenue) OVER (
  ORDER BY order_date
  RANGE BETWEEN INTERVAL '6 days' PRECEDING AND CURRENT ROW
) AS ma7_calendar   -- correct even when some days have no rows... but only if each date appears once
```

If a date can be missing entirely, generate a dense date spine first (`generate_series`) and `LEFT JOIN` the data, so the frame counts real calendar days.

**Trap:** a moving average with `ROWS 6 PRECEDING` at the start of the partition averages over *fewer than 7 rows* (there aren't 7 yet). If you need `NULL` until the window is full, guard with `CASE WHEN COUNT(*) OVER (... same frame) = 7 THEN avg END`.

---

### 6.3 Gaps and Islands

**Shape:** In an ordered sequence, find the **islands** (maximal runs of consecutive values) or the **gaps** (the missing stretches between them). Examples: consecutive invoice numbers, contiguous booking dates, uninterrupted "online" status intervals.

**The classic key — the "difference of two sequences" trick:**

If rows are consecutive, then `value − ROW_NUMBER()` is *constant* across the run. Subtract a dense counter from the value; every island shares one grp value.

```sql
-- Find islands of consecutive integer IDs
WITH marked AS (
  SELECT id,
         id - ROW_NUMBER() OVER (ORDER BY id) AS grp   -- constant within a run
  FROM seat_reservations
)
SELECT MIN(id) AS island_start,
       MAX(id) AS island_end,
       COUNT(*) AS length
FROM marked
GROUP BY grp
ORDER BY island_start;
```

Why it works: as long as `id` increases by exactly 1 each step and `ROW_NUMBER()` also increases by exactly 1, their difference is invariant. The moment a gap appears, `id` jumps but `ROW_NUMBER()` doesn't — the difference shifts to a new constant, starting a new island.

**For dates** (islands of consecutive *days*), subtract a row-number *of days* instead of an integer:

```sql
WITH marked AS (
  SELECT activity_date,
         activity_date - (ROW_NUMBER() OVER (ORDER BY activity_date))::int AS grp
  FROM (SELECT DISTINCT activity_date FROM user_activity) d
)
SELECT MIN(activity_date) AS streak_start,
       MAX(activity_date) AS streak_end,
       COUNT(*)           AS streak_days
FROM marked
GROUP BY grp;
```

Subtracting an integer `ROW_NUMBER()` from a `date` shifts each date back by its ordinal; consecutive dates all land on the same anchor date → same `grp`. **Deduplicate the dates first** (`DISTINCT`) or two activities on one day break the arithmetic.

**Finding gaps (the complement) with `LEAD`:**

```sql
-- Missing invoice-number ranges
SELECT invoice_no + 1                       AS gap_start,
       next_no - 1                          AS gap_end,
       next_no - invoice_no - 1             AS missing_count
FROM (
  SELECT invoice_no,
         LEAD(invoice_no) OVER (ORDER BY invoice_no) AS next_no
  FROM invoices
) t
WHERE next_no - invoice_no > 1;             -- a jump > 1 means a gap
```

**The general "islands with a threshold" variant** (runs where the gap between consecutive rows is ≤ some tolerance) is the same machinery as sessionisation — see 6.6.

---

### 6.4 Consecutive-Day Streaks

**Shape:** The gamification classic — "current login streak," "longest streak ever," "users with a ≥ N-day streak." This is a *specialisation* of gaps-and-islands (6.3) on calendar dates, but it deserves its own treatment because interviewers ask it by name and there are streak-specific twists.

**Longest streak per user — the islands key applied per partition:**

```sql
WITH daily AS (   -- one row per (user, day); collapse multiple logins/day
  SELECT DISTINCT user_id, login_date
  FROM logins
),
grouped AS (
  SELECT user_id, login_date,
         login_date
           - (ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY login_date))::int
         AS grp
  FROM daily
)
SELECT user_id,
       MIN(login_date) AS streak_start,
       MAX(login_date) AS streak_end,
       COUNT(*)        AS streak_length
FROM grouped
GROUP BY user_id, grp
ORDER BY user_id, streak_length DESC;
```

Add `PARTITION BY user_id` so each user gets an independent counter — that's the only change from the single-sequence island query.

**Longest streak (one number per user):**

```sql
SELECT user_id, MAX(streak_length) AS longest_streak
FROM ( /* the grouped query above, aggregated per (user,grp) */ ) s
GROUP BY user_id;
```

**Current streak (streak that includes today):** filter the islands to the one whose `MAX(login_date)` is today (or yesterday, depending on whether "today not yet logged in" still counts):

```sql
... HAVING MAX(login_date) = CURRENT_DATE      -- streak is live today
```

**Streak-specific traps:**
- **Timezones.** "A day" is a business decision. Cast timestamps to the *user's* local date (`(ts AT TIME ZONE user_tz)::date`), not UTC, or a user logging in at 11 p.m. and 1 a.m. local looks like one day in UTC and two in local — or vice versa.
- **Duplicate logins per day** must be collapsed (`DISTINCT`) or the `date − row_number` arithmetic breaks.
- **"Streak" defined with a grace period** (miss one day, keep the streak) is no longer pure consecutive-day islands — it becomes the threshold-island / sessionisation pattern with a 2-day tolerance.

---

### 6.5 Hierarchical Data — Three Models

**Shape:** Trees. Org charts, category trees, comment threads, bill-of-materials. Three storage models, each with a canonical query.

#### 6.5.1 Adjacency List (each row stores its `parent_id`)

The default, normalised model. Reads require **recursion**.

```sql
-- All descendants of employee 42 (the subtree under them), with depth
WITH RECURSIVE subtree AS (
  -- Anchor: the starting node
  SELECT id, manager_id, name, 1 AS depth,
         ARRAY[id] AS path
  FROM employees
  WHERE id = 42
  UNION ALL
  -- Recursive term: children of already-found nodes
  SELECT e.id, e.manager_id, e.name, s.depth + 1,
         s.path || e.id
  FROM employees e
  JOIN subtree s ON e.manager_id = s.id
  WHERE NOT e.id = ANY(s.path)          -- cycle guard: never revisit a node
)
SELECT * FROM subtree ORDER BY path;
```

- **Anchor / recursive term / `UNION ALL`** are mandatory structure. `UNION ALL` (not `UNION`) unless you need dedup — `UNION` adds a per-iteration distinct that's usually wasteful.
- **`path` array** gives you the full ancestry *and* a natural sort order (`ORDER BY path` = pre-order DFS) *and* a **cycle guard** (`NOT id = ANY(path)`) — essential for real data where a bad edge can create a loop that otherwise loops forever.
- **Index on `manager_id`** (the recursive join column) is the difference between fast and quadratic.
- **Ancestors instead of descendants:** flip the join to `JOIN subtree s ON e.id = s.manager_id` and anchor on the leaf.

#### 6.5.2 Nested Sets (each row stores `lft`/`rgt` bounds)

Encode the tree as a pre-order traversal: every node gets a `lft` and `rgt` number; a node's descendants are exactly the rows with `lft` between the parent's `lft` and `rgt`. **Reads become a plain range query — no recursion.**

```sql
-- All descendants of a node: no recursion, one indexed range scan
SELECT child.*
FROM categories parent
JOIN categories child
  ON child.lft > parent.lft AND child.rgt < parent.rgt
WHERE parent.id = 42;

-- Depth of every node in one scan
SELECT n.id, n.name, COUNT(p.id) - 1 AS depth
FROM categories n
JOIN categories p ON n.lft BETWEEN p.lft AND p.rgt
GROUP BY n.id, n.name;
```

**Trade-off:** reads are trivially fast and index-friendly, but **any insert/move rewrites `lft`/`rgt` for up to half the table** (every node to the "right" shifts). Great for read-heavy, rarely-mutated trees (product taxonomies); terrible for chat threads.

#### 6.5.3 Closure Table (a separate table of every ancestor→descendant pair)

Materialise transitive closure: for every node, store a row `(ancestor, descendant, depth)` for *each* ancestor including itself.

```sql
-- Schema: category_paths(ancestor_id, descendant_id, depth)
-- All descendants of 42: a single indexed lookup, no recursion
SELECT c.*
FROM category_paths cp
JOIN categories c ON c.id = cp.descendant_id
WHERE cp.ancestor_id = 42 AND cp.depth > 0;

-- Insert a new leaf under parent P: copy P's ancestor set + self-row
INSERT INTO category_paths (ancestor_id, descendant_id, depth)
SELECT ancestor_id, :new_id, depth + 1 FROM category_paths WHERE descendant_id = :parent
UNION ALL SELECT :new_id, :new_id, 0;
```

**Trade-off:** reads *and* subtree queries are O(rows-returned) index scans with no recursion; writes cost O(depth) extra rows and need transactional maintenance (usually via triggers). This is the model most large systems converge on for deep, frequently-queried trees.

**Model choice summary:**

| Model | Read subtree | Insert/move | Storage | Best for |
|-------|-------------|-------------|---------|----------|
| Adjacency list | Recursive CTE | O(1) | Minimal | Default; write-heavy trees |
| Nested sets | Range scan (fast) | O(N) rewrite | Two ints/row | Read-heavy, static trees |
| Closure table | Index scan (fast) | O(depth) rows | O(N·depth) | Deep, read-heavy, mutable |

---

### 6.6 Sessionisation

**Shape:** A stream of timestamped events per user. Group them into **sessions**: a session breaks when the gap to the previous event exceeds a timeout (e.g. 30 minutes). Assign each event a session number; report session boundaries.

**The classic two-window key: `LAG` to find breaks, running `SUM` to number sessions.**

```sql
WITH gaps AS (
  SELECT user_id, event_time,
         -- Is this event the START of a new session?
         CASE
           WHEN event_time - LAG(event_time) OVER (PARTITION BY user_id ORDER BY event_time)
                > INTERVAL '30 minutes'
             OR LAG(event_time) OVER (PARTITION BY user_id ORDER BY event_time) IS NULL
           THEN 1 ELSE 0
         END AS is_new_session
  FROM events
),
sessioned AS (
  SELECT user_id, event_time,
         -- Cumulative count of "new session" flags = a session ID, dense per user
         SUM(is_new_session) OVER (PARTITION BY user_id ORDER BY event_time
                                   ROWS UNBOUNDED PRECEDING) AS session_num
  FROM gaps
)
SELECT user_id, session_num,
       MIN(event_time)                       AS session_start,
       MAX(event_time)                       AS session_end,
       MAX(event_time) - MIN(event_time)     AS duration,
       COUNT(*)                              AS event_count
FROM sessioned
GROUP BY user_id, session_num
ORDER BY user_id, session_num;
```

Step by step:
1. **`LAG`** fetches the previous event's time within the user's ordered stream.
2. The **`CASE`** flags a row as a session-starter when the gap exceeds the timeout, or when there's no previous row (first event = new session).
3. The **running `SUM` of the flag** turns those 0/1 markers into a monotonically increasing session number — every event inherits the session of the most recent "start." This "running sum of a boolean" is the same trick as islands (6.3), just triggered by a gap threshold instead of a strict `+1`.
4. **Aggregate** per `(user, session_num)` for the session report.

**Variations:**
- **Fixed-length sessions** (e.g. "a new session every 24h regardless of activity") use `date_trunc`/`floor(epoch/window)` instead of the gap logic.
- **Session on inactivity *or* a boundary event** (login/logout markers): `OR event_type = 'session_start'` inside the `CASE`.
- **Cross-day sessions:** the gap logic handles midnight naturally — no special case, unlike naive `GROUP BY date`.

---

### 6.7 Deduplication With Window Functions

**Shape:** A table has duplicate rows by some *natural key* (same `email`, same `(user_id, day)`), and you want exactly one per key — typically the "best" one (newest, highest-priority) — either to *query* clean data or to *delete* the extras.

**The classic key — `ROW_NUMBER()` over the natural key, keep `rn = 1`:**

```sql
-- Keep the most recent row per email; drop older duplicates (query-time)
SELECT *
FROM (
  SELECT *,
         ROW_NUMBER() OVER (PARTITION BY email          -- the natural key
                            ORDER BY updated_at DESC,    -- "best" = newest
                                     id DESC)            -- deterministic tiebreak
         AS rn
  FROM users
) t
WHERE rn = 1;
```

The `ORDER BY` inside `OVER` *defines what "keep" means* — newest, highest score, non-null-first (`ORDER BY (col IS NULL), updated_at DESC`). Always add a unique tiebreak (`id`) so "the winner" is deterministic.

**Deleting duplicates in place (the destructive version):**

```sql
DELETE FROM users u
USING (
  SELECT id,
         ROW_NUMBER() OVER (PARTITION BY email ORDER BY updated_at DESC, id DESC) AS rn
  FROM users
) d
WHERE u.id = d.id AND d.rn > 1;   -- delete everything that ISN'T the winner
```

**`DISTINCT ON` — the Postgres shorthand for the same thing:**

```sql
SELECT DISTINCT ON (email) *
FROM users
ORDER BY email, updated_at DESC, id DESC;   -- leading col = dedup key; rest = "best"
```

**Traps:**
- **`SELECT DISTINCT` is not dedup by key** — it dedups whole rows. If any non-key column differs (a `last_seen` timestamp), the "duplicates" survive. `DISTINCT` on the full row almost never solves a natural-key dedup.
- **`GROUP BY email` loses the other columns** — you can only keep aggregates, not "the whole newest row." That's exactly why window functions win here.
- **Before adding a `UNIQUE` constraint**, dedup with the `DELETE ... rn > 1` pattern, then add the constraint in the same transaction so no new dupes sneak in.

---

### 6.8 Median and Percentiles

**Shape:** "The median (p50) order value," "the p95 latency," "the interquartile range." The *mean* (`AVG`) is not the median and lies in the presence of skew — every senior knows to reach for percentiles on skewed distributions (revenue, latency, session length).

**The built-in key — ordered-set aggregates:**

```sql
SELECT
  category_id,
  PERCENTILE_CONT(0.5)  WITHIN GROUP (ORDER BY total_amount) AS median_interpolated,
  PERCENTILE_DISC(0.5)  WITHIN GROUP (ORDER BY total_amount) AS median_actual_value,
  PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY total_amount) AS p95,
  PERCENTILE_CONT(ARRAY[0.25,0.5,0.75])
    WITHIN GROUP (ORDER BY total_amount) AS quartiles   -- returns an array
FROM orders
GROUP BY category_id;
```

**`PERCENTILE_CONT` vs `PERCENTILE_DISC` — know the difference cold:**
- **`PERCENTILE_CONT`** *interpolates* between the two nearest values. Median of `{10, 20}` = **15**. Returns a value that may not exist in the data. This is the statistical median.
- **`PERCENTILE_DISC`** returns an *actual data value* — the first whose cumulative distribution ≥ the fraction. Median of `{10, 20}` = **10** (or 20 depending on rule). Use when the answer must be a real observed value (e.g. "the median *customer*").

**The manual median (the interview favourite — "do it without `PERCENTILE_CONT`"):**

```sql
-- Median via ROW_NUMBER from both ends: the middle 1 (odd) or 2 (even) rows, averaged
WITH ordered AS (
  SELECT total_amount,
         ROW_NUMBER() OVER (ORDER BY total_amount) AS rn_asc,
         COUNT(*)     OVER ()                       AS n
  FROM orders
)
SELECT AVG(total_amount) AS median
FROM ordered
WHERE rn_asc IN ( (n + 1) / 2, (n + 2) / 2 );
-- odd n:  both expressions equal the single middle row → AVG of one value
-- even n: they pick the two middle rows → AVG of the two → interpolated median
```

The `(n+1)/2` and `(n+2)/2` integer-division trick selects one middle row when `n` is odd and the two middle rows when `n` is even, so a single `AVG` handles both parities. Interviewers love this because it tests whether you understand median's definition, not just the function name.

**Per-group manual median** uses `PARTITION BY` in the `ROW_NUMBER()`/`COUNT()` and `GROUP BY` in the outer query.

**Performance note:** `PERCENTILE_CONT` **materialises and sorts the entire group in memory** (ordered-set aggregates buffer everything). On huge groups it spills to disk and is slow. For approximate percentiles at scale, reach for the `tdigest` or `hll`-style extensions, or pre-bucket. This is a real staff-level talking point.

---

### 6.9 Pivot Without Crosstab

**Shape:** Turn rows into columns — "months across the top, one row per product, revenue in the cells." Postgres has no `PIVOT` keyword (unlike SQL Server/Oracle).

**The classic key — conditional aggregation (`SUM(CASE WHEN ...)`):**

```sql
-- Monthly revenue per product, months as columns
SELECT
  p.name AS product,
  SUM(CASE WHEN date_trunc('month', o.created_at) = '2026-01-01' THEN oi.quantity*oi.unit_price END) AS jan,
  SUM(CASE WHEN date_trunc('month', o.created_at) = '2026-02-01' THEN oi.quantity*oi.unit_price END) AS feb,
  SUM(CASE WHEN date_trunc('month', o.created_at) = '2026-03-01' THEN oi.quantity*oi.unit_price END) AS mar,
  SUM(oi.quantity*oi.unit_price) AS total          -- row total for free
FROM order_items oi
JOIN orders o   ON o.id = oi.order_id
JOIN products p ON p.id = oi.product_id
WHERE o.created_at >= '2026-01-01' AND o.created_at < '2026-04-01'
GROUP BY p.name;
```

Each `CASE` acts as a filtered aggregate — rows not matching the condition contribute `NULL`, which `SUM` ignores. One grouped scan produces all columns. **`FILTER` is the cleaner Postgres syntax** for the same thing:

```sql
SUM(oi.quantity*oi.unit_price) FILTER (WHERE date_trunc('month', o.created_at) = '2026-01-01') AS jan
```

**The `crosstab()` extension** (`tablefunc`) does true dynamic pivoting but requires declaring the output column list up front and is fiddly — most teams prefer `FILTER`/`CASE` for its transparency and index-friendliness.

**The fundamental limitation: SQL columns are fixed at parse time.** You cannot pivot an *unknown* set of months into columns in pure SQL — the column list must be literal. Truly dynamic pivots require: (a) a first query to fetch the distinct values, (b) building the SQL string in the application, (c) executing it. This is a legitimate "you can't do it in one static query" answer in an interview — knowing the *why* (columns bind at parse time) is the senior signal.

**Unpivot (columns → rows)** is the reverse, via `LATERAL (VALUES ...)` or `UNNEST`:

```sql
SELECT product, month, revenue
FROM monthly_pivot
CROSS JOIN LATERAL (VALUES ('jan', jan), ('feb', feb), ('mar', mar))
           AS u(month, revenue);
```

---

### 6.10 Self-Referencing Queries

**Shape:** Compare a row to *another row in the same table* — "employees who earn more than their manager," "days where revenue beat the prior day," "products priced above their category average." The table joins to itself.

**Self-join key (row-to-related-row):**

```sql
-- Employees earning more than their manager
SELECT e.name AS employee, e.salary, m.name AS manager, m.salary AS manager_salary
FROM employees e
JOIN employees m ON m.id = e.manager_id      -- second alias = "the manager row"
WHERE e.salary > m.salary;
-- NOTE: INNER JOIN drops top-level employees (manager_id IS NULL) — intentional here
```

The whole trick is **two aliases of one table** (`e` = employee, `m` = manager) treated as if they were different tables.

**Window-function alternative (row-to-neighbour), usually faster:** when "the other row" is the previous/next in some order, `LAG`/`LEAD` beats a self-join because it's one sorted pass instead of an N×N join:

```sql
-- Days where revenue beat the previous day
SELECT day, revenue, prev_revenue
FROM (
  SELECT day, revenue,
         LAG(revenue) OVER (ORDER BY day) AS prev_revenue
  FROM daily_sales
) t
WHERE revenue > prev_revenue;
```

**Row-to-group comparison** ("above category average") — a window partition, no join at all:

```sql
SELECT name, category_id, price, avg_price
FROM (
  SELECT name, category_id, price,
         AVG(price) OVER (PARTITION BY category_id) AS avg_price
  FROM products
) t
WHERE price > avg_price;
```

**Choosing between them:**
- "This row vs one specific related row (parent, manager)" → **self-join**.
- "This row vs the previous/next in an order" → **`LAG`/`LEAD`**.
- "This row vs its group's aggregate" → **window aggregate with `PARTITION BY`** (no self-join, no double scan).

The window forms avoid the self-join's `O(N²)` risk and are the senior default whenever the "other row" is positional or aggregate rather than a specific FK-linked row.

---

## 7. EXPLAIN — Reading the Plans of the Canonical Patterns

### 7.1 Top-N per group (`ROW_NUMBER`) — WindowAgg over Sort

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM (
  SELECT customer_id, id, total_amount,
         ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY total_amount DESC) rn
  FROM orders
) t WHERE rn <= 3;
```

```
Subquery Scan on t  (cost=... rows=333 width=24) (actual time=42.1..118.7 rows=2891 loops=1)
  Filter: (t.rn <= 3)
  Rows Removed by Filter: 97109
  ->  WindowAgg  (cost=13850..16350 rows=100000 width=24) (actual time=42.1..108.2 rows=100000 loops=1)
        ->  Sort  (cost=13850..14100 rows=100000 width=16) (actual time=42.0..55.9 rows=100000 loops=1)
              Sort Key: orders.customer_id, orders.total_amount DESC
              Sort Method: quicksort  Memory: 9216kB
              ->  Seq Scan on orders  (cost=0..1730 rows=100000 width=16) (actual rows=100000 loops=1)
Buffers: shared hit=730
Execution Time: 121.4 ms
```

**Reading it:**
- `Sort` (on `customer_id, total_amount DESC`) is the cost centre — 55ms of the 121ms. `WindowAgg` runs the ranking in one pass over the sorted stream.
- `Rows Removed by Filter: 97109` — the `WindowAgg` computed `ROW_NUMBER()` for *all* 100k rows, then threw away 97k. The window can't be filtered early; this waste is inherent.
- **Optimisation:** an index on `(customer_id, total_amount DESC)` removes the `Sort` node entirely — the plan becomes `Index Scan → WindowAgg`, dropping the 55ms sort.

### 7.2 Recursive CTE — the WorkTable loop

```
CTE Scan on subtree  (actual time=0.03..2.11 rows=214 loops=1)
  CTE subtree
    ->  Recursive Union  (actual time=0.02..1.98 rows=214 loops=1)
          ->  Index Scan using employees_pkey on employees  (actual rows=1 loops=1)  -- anchor
          ->  Nested Loop  (actual rows=27 loops=8)                                  -- recursive term
                ->  WorkTable Scan on subtree  (actual rows=8 loops=8)
                ->  Index Scan using employees_manager_id_idx on employees e
                      Index Cond: (manager_id = subtree.id)
```

**Reading it:** `Recursive Union` = fixpoint loop. `loops=8` on the recursive term = tree depth of ~8 levels. The `Index Scan using employees_manager_id_idx` is the payoff of indexing the recursive join column — without it that inner node is a `Seq Scan` per iteration, turning the recursion quadratic.

### 7.3 Median — the materialising ordered-set aggregate

```
GroupAggregate  (actual time=88.4..402.1 rows=12 loops=1)
  Group Key: category_id
  ->  Sort  (actual time=52.1..120.7 rows=100000 loops=1)
        Sort Key: category_id
        Sort Method: external merge  Disk: 24560kB      -- ⚠ spilled: work_mem too small
  ...
```

**Reading it:** `PERCENTILE_CONT` forces a full sort *and* buffers each group. `Sort Method: external merge  Disk: 24560kB` is the red flag — the sort spilled to disk. Bumping `work_mem` keeps it in RAM (`Sort Method: quicksort`) and can cut runtime by 3–5×.

### 7.4 Sessionisation — two WindowAggs sharing one Sort

```
WindowAgg (running SUM)  (actual rows=500000 loops=1)
  ->  WindowAgg (LAG)  (actual rows=500000 loops=1)
        ->  Sort  Sort Key: user_id, event_time   -- ONE sort feeds BOTH windows
              ->  Seq Scan on events
```

**Reading it:** both window functions share `PARTITION BY user_id ORDER BY event_time`, so the planner sorts *once* and stacks the two `WindowAgg` nodes. An index on `(user_id, event_time)` eliminates the sort — the single biggest win for sessionisation at scale.

---

## 8. Query Examples — Each Canonical Problem End to End

### Example 1 — Basic: Top-N per group + running total combined

```sql
-- The 3 highest-spending customers per region, with each region's cumulative spend share
WITH ranked AS (
  SELECT
    u.region,
    u.id AS user_id,
    u.name,
    SUM(o.total_amount) AS spend,
    ROW_NUMBER() OVER (PARTITION BY u.region ORDER BY SUM(o.total_amount) DESC) AS rn
  FROM users u
  JOIN orders o ON o.customer_id = u.id AND o.status = 'completed'
  GROUP BY u.region, u.id, u.name
)
SELECT region, name, spend,
       ROUND(spend / SUM(spend) OVER (PARTITION BY region) * 100, 1) AS pct_of_region
FROM ranked
WHERE rn <= 3
ORDER BY region, spend DESC;
```

### Example 2 — Intermediate: Consecutive-day streaks with current-streak flag

```sql
-- Every login streak per user, flagging the one that is still live today
WITH daily AS (
  SELECT DISTINCT user_id, (login_at AT TIME ZONE 'UTC')::date AS d
  FROM sessions
),
islands AS (
  SELECT user_id, d,
         d - (ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY d))::int AS grp
  FROM daily
),
streaks AS (
  SELECT user_id,
         MIN(d) AS streak_start,
         MAX(d) AS streak_end,
         COUNT(*) AS length,
         (MAX(d) >= CURRENT_DATE - 1) AS is_current   -- live if it reaches yesterday/today
  FROM islands
  GROUP BY user_id, grp
)
SELECT user_id, streak_start, streak_end, length, is_current
FROM streaks
WHERE length >= 3                          -- only streaks worth showing
ORDER BY user_id, streak_end DESC;
```

### Example 3 — Production Grade: Sessionisation with revenue attribution

**Context:** `sessions` here is the raw event stream — `events(user_id, event_time, event_type, revenue)`, 200M rows, growing 5M/day. Index available: `events_user_time_idx ON events(user_id, event_time)`. Goal: attribute revenue to marketing sessions (30-min inactivity timeout), report per session. Perf expectation: with the composite index the sort is skipped; runs in a few seconds for a single user's history, minutes for a full backfill batched by `user_id` range.

```sql
-- Sessionise a user event stream and attribute revenue per session
WITH flagged AS (
  SELECT
    user_id, event_time, event_type, revenue,
    CASE
      WHEN event_time - LAG(event_time)
             OVER (PARTITION BY user_id ORDER BY event_time) > INTERVAL '30 minutes'
        OR LAG(event_time) OVER (PARTITION BY user_id ORDER BY event_time) IS NULL
      THEN 1 ELSE 0
    END AS new_session
  FROM events
  WHERE event_time >= NOW() - INTERVAL '90 days'          -- prune BEFORE the window
),
numbered AS (
  SELECT *,
         SUM(new_session) OVER (PARTITION BY user_id ORDER BY event_time
                                ROWS UNBOUNDED PRECEDING) AS session_num
  FROM flagged
)
SELECT
  user_id,
  session_num,
  MIN(event_time)                                   AS started_at,
  MAX(event_time)                                   AS ended_at,
  MAX(event_time) - MIN(event_time)                 AS duration,
  COUNT(*)                                          AS event_count,
  COUNT(*) FILTER (WHERE event_type = 'page_view')  AS page_views,
  COALESCE(SUM(revenue), 0)                         AS session_revenue,
  (COUNT(*) FILTER (WHERE event_type = 'purchase') > 0) AS converted
FROM numbered
GROUP BY user_id, session_num
HAVING COUNT(*) > 1                                   -- drop single-event bounces
ORDER BY user_id, session_num;
```

```sql
EXPLAIN (ANALYZE, BUFFERS) -- shape
```
```
GroupAggregate  (actual time=210..3480 rows=4.2M loops=1)
  Group Key: numbered.user_id, numbered.session_num
  ->  WindowAgg  (running SUM new_session)            -- session numbering
      ->  WindowAgg  (LAG, gap detection)             -- reuses the sort below
          ->  Index Scan using events_user_time_idx on events   -- NO Sort node ✓
                Index Cond: (event_time >= now() - '90 days')
Buffers: shared hit=1.1M read=94k
Execution Time: 3510 ms
```

The index supplies `(user_id, event_time)` order, so **both** window functions run without a sort node — the whole 30M-row window pipeline is one indexed scan plus two streaming WindowAggs.

---

## 9. Wrong → Right Patterns

### Wrong 1: Top-N per group via correlated subquery (the O(N²) bomb)

```sql
-- WRONG: a MAX subquery re-scans orders for every row
SELECT o.*
FROM orders o
WHERE o.total_amount = (
  SELECT MAX(o2.total_amount) FROM orders o2 WHERE o2.customer_id = o.customer_id
);
```
Executes the inner aggregate once per outer row — O(N²). On 1M orders it can run for many minutes. It also only gets N=1 and returns *ties* (two orders at the same max both qualify), which may or may not be intended.

```sql
-- RIGHT: one sorted pass with a window function
SELECT customer_id, id, total_amount
FROM (
  SELECT *, ROW_NUMBER() OVER (PARTITION BY customer_id
                               ORDER BY total_amount DESC, id) rn
  FROM orders
) t WHERE rn = 1;
```

### Wrong 2: Median computed as AVG

```sql
-- WRONG: this is the MEAN, not the median. On skewed data it is off by a lot.
SELECT category_id, AVG(total_amount) AS "median?" FROM orders GROUP BY category_id;
```
For revenue/latency (right-skewed), the mean is dragged up by a few whales and is *not* the typical value. Reporting it as "median" is a data-integrity bug.

```sql
-- RIGHT: the actual median
SELECT category_id,
       PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY total_amount) AS median
FROM orders GROUP BY category_id;
```

### Wrong 3: Running total with the default (RANGE) frame

```sql
-- WRONG: default frame is RANGE — tied dates share ONE cumulative value
SELECT order_date, amount,
       SUM(amount) OVER (ORDER BY order_date) AS running_total   -- RANGE peers merged!
FROM orders;
-- Two orders on 2026-03-01 both show the SAME running_total (sum through that whole day)
```

```sql
-- RIGHT: ROWS frame accumulates per row
SELECT order_date, amount,
       SUM(amount) OVER (ORDER BY order_date, id
                         ROWS UNBOUNDED PRECEDING) AS running_total
FROM orders;
```

### Wrong 4: Dedup with SELECT DISTINCT

```sql
-- WRONG: DISTINCT dedups WHOLE rows. A differing last_login keeps both "duplicates".
SELECT DISTINCT email, name, last_login FROM users;   -- duplicates by email survive
```

```sql
-- RIGHT: one row per email, the newest, all columns intact
SELECT DISTINCT ON (email) *
FROM users
ORDER BY email, last_login DESC NULLS LAST, id;
```

### Wrong 5: Streak query without collapsing same-day duplicates

```sql
-- WRONG: two logins on the same day break the date - row_number arithmetic
SELECT user_id, login_date,
       login_date - (ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY login_date))::int AS grp
FROM logins;   -- duplicate dates make row_number outpace the date → false gaps/merges
```

```sql
-- RIGHT: dedup to one row per (user, day) FIRST
WITH daily AS (SELECT DISTINCT user_id, login_date FROM logins)
SELECT user_id, login_date,
       login_date - (ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY login_date))::int AS grp
FROM daily;
```

### Wrong 6: Recursive CTE with no cycle guard

```sql
-- WRONG: a single bad edge (A→B→A) loops forever / errors on a huge worktable
WITH RECURSIVE tree AS (
  SELECT id, parent_id FROM categories WHERE id = 1
  UNION ALL
  SELECT c.id, c.parent_id FROM categories c JOIN tree t ON c.parent_id = t.id
)
SELECT * FROM tree;
```

```sql
-- RIGHT: carry a path array and refuse to revisit a node
WITH RECURSIVE tree AS (
  SELECT id, parent_id, ARRAY[id] AS path FROM categories WHERE id = 1
  UNION ALL
  SELECT c.id, c.parent_id, t.path || c.id
  FROM categories c JOIN tree t ON c.parent_id = t.id
  WHERE NOT c.id = ANY(t.path)          -- cycle guard
)
SELECT * FROM tree;
```

---

## 10. Performance Profile

| Pattern | Core op | 1M rows | 10M rows | 100M rows | Index that saves it |
|---------|---------|---------|----------|-----------|---------------------|
| Top-N per group | Sort + WindowAgg | ~0.3s | ~4s | ~50s (spills) | `(partition_key, metric DESC)` → no sort |
| Running total / MA | Sort + WindowAgg (frame) | ~0.3s | ~4s | ~50s | `(order_key)` → no sort |
| Gaps & islands | Sort + WindowAgg | ~0.3s | ~4s | ~50s | index on the ordered col |
| Streaks | DISTINCT + islands | +dedup cost | | | `(user_id, date)` |
| Hierarchy (adjacency) | Recursive Union | depth-bound | depth-bound | depth-bound | `(parent_id)` — critical |
| Hierarchy (nested set) | Range scan | ~ms | ~ms | ~ms | `(lft, rgt)` |
| Sessionisation | 2× WindowAgg, 1 sort | ~0.4s | ~5s | minutes (batch it) | `(user_id, event_time)` |
| Dedup | WindowAgg or DISTINCT ON | ~0.3s | ~4s | ~50s | `(natural_key, tiebreak)` |
| Median / percentile | **Full sort, buffers group** | ~0.5s | ~8s + spill | disk-bound | limited help; raise `work_mem` |
| Pivot | HashAggregate | ~0.3s | ~3s | ~40s | index on GROUP BY key |

**Cross-cutting optimisation rules:**
1. **The sort is the cost.** Seven of ten patterns are `Sort → WindowAgg`. A composite index matching `PARTITION BY … ORDER BY …` eliminates the `Sort` node — routinely a 2–5× win. Check for a `Sort` node in `EXPLAIN`; if present and an index could supply that order, add it.
2. **`work_mem` governs spills.** Window sorts and median's group buffer spill to disk (`Sort Method: external merge`) when they exceed `work_mem`. For analytics sessions, `SET work_mem = '256MB'` per query/session can turn minutes into seconds. Raise it locally, not globally (it's per-sort, per-connection — a global bump can OOM the server under concurrency).
3. **Recursion lives or dies on the join index.** A recursive CTE without an index on the recursive join column is quadratic. `(parent_id)` / `(manager_id)` is non-negotiable.
4. **Median doesn't scale by index.** `PERCENTILE_CONT` must sort and buffer the whole group; no index removes that. At 100M+ rows use approximate-percentile extensions (`tdigest`) or pre-aggregated buckets.
5. **Batch the giant backfills.** Sessionising 200M events in one query risks OOM. Batch by `user_id` range or day; each batch is an independent, index-scan-driven window pass.
6. **Materialise repeated hard queries.** A streak/session/top-N feeding a dashboard that reloads every minute belongs in a materialised view refreshed on a schedule, not recomputed per request.

---

## 11. Node.js Integration

```javascript
import pg from 'pg';
const { Pool } = pg;
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

// 11.1 Top-N per group — parameterised N and metric window
async function topOrdersPerCustomer(limitPerCustomer = 3) {
  const { rows } = await pool.query(
    `SELECT customer_id, order_id, total_amount
     FROM (
       SELECT customer_id, id AS order_id, total_amount,
              ROW_NUMBER() OVER (PARTITION BY customer_id
                                 ORDER BY total_amount DESC, id) AS rn
       FROM orders
     ) t
     WHERE rn <= $1
     ORDER BY customer_id, total_amount DESC`,
    [limitPerCustomer]
  );
  return rows;
}

// 11.2 Sessionisation for one user — raise work_mem for the window sort within the txn
async function sessionise(userId, timeoutMinutes = 30) {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');
    await client.query(`SET LOCAL work_mem = '128MB'`);        // scoped to this txn only
    const { rows } = await client.query(
      `WITH flagged AS (
         SELECT event_time, event_type, revenue,
                CASE WHEN event_time - LAG(event_time) OVER w > ($2 || ' minutes')::interval
                       OR LAG(event_time) OVER w IS NULL
                     THEN 1 ELSE 0 END AS new_session
         FROM events WHERE user_id = $1
         WINDOW w AS (ORDER BY event_time)
       ),
       numbered AS (
         SELECT *, SUM(new_session) OVER (ORDER BY event_time ROWS UNBOUNDED PRECEDING) AS s
         FROM flagged
       )
       SELECT s AS session_num, MIN(event_time) started_at, MAX(event_time) ended_at,
              COUNT(*) events, COALESCE(SUM(revenue),0) revenue
       FROM numbered GROUP BY s ORDER BY s`,
      [userId, timeoutMinutes]
    );
    await client.query('COMMIT');
    return rows;
  } catch (e) { await client.query('ROLLBACK'); throw e; }
  finally { client.release(); }
}

// 11.3 Recursive hierarchy — descendants of a node
async function orgSubtree(rootId) {
  const { rows } = await pool.query(
    `WITH RECURSIVE subtree AS (
       SELECT id, manager_id, name, 1 AS depth, ARRAY[id] AS path
       FROM employees WHERE id = $1
       UNION ALL
       SELECT e.id, e.manager_id, e.name, s.depth + 1, s.path || e.id
       FROM employees e JOIN subtree s ON e.manager_id = s.id
       WHERE NOT e.id = ANY(s.path)
     )
     SELECT id, name, depth FROM subtree ORDER BY path`,
    [rootId]
  );
  return rows;   // rows[].depth lets the UI indent the tree
}

// 11.4 Median + p95 — one round trip
async function latencyPercentiles(service) {
  const { rows } = await pool.query(
    `SELECT PERCENTILE_CONT(0.5)  WITHIN GROUP (ORDER BY latency_ms) AS p50,
            PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY latency_ms) AS p95,
            PERCENTILE_CONT(0.99) WITHIN GROUP (ORDER BY latency_ms) AS p99
     FROM request_logs WHERE service = $1 AND created_at >= NOW() - INTERVAL '1 hour'`,
    [service]
  );
  return rows[0];
}
```

**Notes:** Always parameterise (`$1`) — never string-interpolate `N`, user IDs, or intervals (SQL injection + no plan caching). Use `SET LOCAL` inside a transaction to scope `work_mem` bumps so they don't leak to other pooled queries. The `WINDOW w AS (...)` named-window clause de-duplicates a window spec referenced by multiple functions — cleaner and it makes the shared-sort intent explicit.

---

## 12. ORM Comparison

Every pattern here leans on window functions, recursive CTEs, or ordered-set aggregates — the exact SQL surface ORMs handle *worst*. The honest verdict for all five: **use the raw-SQL escape hatch.**

### Prisma
**Can it?** No first-class window functions, recursive CTEs, or `PERCENTILE_CONT`. `groupBy` handles simple aggregates only.
```typescript
// The only realistic path: $queryRaw with the classic pattern
const topN = await prisma.$queryRaw`
  SELECT customer_id, order_id, total_amount FROM (
    SELECT customer_id, id AS order_id, total_amount,
           ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY total_amount DESC) rn
    FROM orders
  ) t WHERE rn <= 3`;
```
**Where it breaks:** everything in this topic. **Verdict:** `$queryRaw` for all ten patterns; use Prisma types via `$queryRaw<Row[]>`.

### Drizzle
**Can it?** Partially — Drizzle has typed `sql` helpers and a `.$with()`/window API that can express `ROW_NUMBER() OVER (...)` and CTEs.
```typescript
const rn = sql<number>`row_number() over (partition by ${orders.customerId}
                       order by ${orders.totalAmount} desc)`.as('rn');
const sub = db.select({ id: orders.id, customerId: orders.customerId, rn }).from(orders).as('sub');
const top = await db.select().from(sub).where(sql`${sub.rn} <= 3`);
```
**Where it breaks:** recursive CTEs and ordered-set aggregates still need `sql` raw fragments. **Verdict:** best of the five for windows; drop to `sql` for recursion/percentiles.

### Sequelize
**Can it?** Window functions only via `sequelize.literal()` / `sequelize.fn` hacks; no recursive CTE support.
**Verdict:** use `sequelize.query(sql, { type: QueryTypes.SELECT })` for every pattern here.

### TypeORM
**Can it?** QueryBuilder has `.addSelect('ROW_NUMBER() OVER (...)','rn')` as raw strings and supports subquery wrapping, but no typed window API and no recursive-CTE builder.
```typescript
const rows = await ds.createQueryBuilder()
  .select('t.*').from(qb => qb
     .select('o.*').addSelect('ROW_NUMBER() OVER (PARTITION BY o.customer_id ORDER BY o.total_amount DESC)','rn')
     .from('orders','o'), 't')
  .where('t.rn <= :n', { n: 3 }).getRawMany();
```
**Verdict:** works for windows via raw strings (no type safety); `ds.query()` for recursion and percentiles.

### Knex
**Can it?** Knex has `.rowNumber()`, `.rank()`, `.partitionBy()`, `.over()` and `.withRecursive()` — the most complete window/CTE builder of the five.
```javascript
const top = await knex.select('*').from(
  knex('orders').select('*')
    .rowNumber('rn', { column: 'total_amount', order: 'desc' }, 'customer_id')
).where('rn', '<=', 3);
// Recursive:
knex.withRecursive('subtree', qb => { /* anchor */ }, /* recursive */ );
```
**Verdict:** best window/recursion coverage; `knex.raw()` for `PERCENTILE_CONT` and `DISTINCT ON`.

### Summary

| Pattern | Prisma | Drizzle | Sequelize | TypeORM | Knex |
|---------|--------|---------|-----------|---------|------|
| Window functions | Raw | ✅ `sql` | Raw | Raw string | ✅ builder |
| Recursive CTE | Raw | Raw `sql` | Raw | Raw | ✅ `.withRecursive` |
| `PERCENTILE_CONT` | Raw | Raw | Raw | Raw | Raw |
| `DISTINCT ON` | Raw | ✅ | Raw | Raw | Raw |
| **Verdict** | `$queryRaw` | Strong | `.query()` | QueryBuilder+raw | Strongest |

**Overall:** for a curriculum built around these ten patterns, treat raw SQL as the primary interface and the ORM as connection/typing plumbing. Knex and Drizzle reduce the raw surface the most; Prisma and Sequelize essentially always fall back to raw.

---

## 13. Practice Exercises

### Exercise 1 — Easy (top-N per group)
Given `products(id, name, category_id, price)`, return the 2 most expensive products in each category, with columns `category_id, name, price`. Break ties deterministically by `id`. Order the output by category, price descending.
```sql
-- Write your query here
```

### Exercise 2 — Medium (combines dedup + running total, Topics 6.7 & 6.2)
Given `payments(id, user_id, amount, paid_at)` where a retry bug inserted some exact duplicate `(user_id, amount, paid_at)` rows: (a) dedup to one row per `(user_id, amount, paid_at)` keeping the lowest `id`; (b) from the deduped set, produce each user's running cumulative total ordered by `paid_at`, with columns `user_id, paid_at, amount, running_total`. Use a true per-row running total (mind the frame).
```sql
-- Write your query here
```

### Exercise 3 — Hard (production simulation; naive answer is wrong)
Given `user_activity(user_id, activity_date)` with possible multiple rows per day and duplicate dates, compute each user's **longest login streak** and whether it is **current** (reaches `CURRENT_DATE` or `CURRENT_DATE - 1`). The naive `MAX(activity_date) - MIN(activity_date)` answer is wrong (it ignores gaps) and forgetting to dedup dates is also wrong. Return `user_id, longest_streak, is_current`, only for users whose longest streak ≥ 5.
```sql
-- Write your query here
```

### Exercise 4 — Interview simulation
"Here is an `events(user_id, event_time, event_type)` table, ~50M rows. Split each user's events into sessions with a 30-minute inactivity timeout, then return, per user: total sessions, average events per session, and the timestamp of the session with the most events. What single index makes this fast, and why does the plan avoid a sort?"
```sql
-- Write your query here
```

---

## 14. Interview Questions

### Q1 — "Give me the top 3 products by sales in each category."

**Junior answer:** `GROUP BY category_id ORDER BY sales DESC LIMIT 3` — which returns 3 rows *total*, not 3 per category. Or a correlated `WHERE sales >= (SELECT ... ORDER BY ... LIMIT 3 OFFSET 2)` that's slow and wrong on ties.

**Principal answer:** "Top-N per group — window function. `ROW_NUMBER() OVER (PARTITION BY category_id ORDER BY sales DESC)` in a subquery, filter `rn <= 3` in the outer query, because window functions run after `WHERE` so I can't filter the rank inline. I'd add `, id` as a deterministic tiebreak. If they want ties included I'd switch to `RANK()`; if N=1 I'd use `DISTINCT ON (category_id)` with a matching index for a sort-free scan. And a composite index on `(category_id, sales DESC)` removes the sort node from the plan."

**Follow-up:** "What if a product is tied for 3rd?" → "`ROW_NUMBER` picks one arbitrarily (or per my tiebreak); `RANK` returns all tied for 3rd, possibly >3 rows. The choice is a product decision — I'd confirm which the requirement wants."

### Q2 — "Compute the median order value per category. Then do it without `PERCENTILE_CONT`."

**Junior answer:** `AVG(total_amount)` — that's the mean, not the median, and it's badly wrong on skewed revenue data.

**Principal answer:** "`PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY total_amount)` per category — that interpolates, giving the true statistical median. `PERCENTILE_DISC` if I need an actual observed value. Without the function: `ROW_NUMBER()` ascending plus `COUNT(*) OVER ()` as `n`, then average the rows at positions `(n+1)/2` and `(n+2)/2` — integer division picks one middle row for odd n and two for even n, so a single `AVG` handles both. Caveat: `PERCENTILE_CONT` buffers and sorts the whole group in `work_mem`, so on huge groups it spills to disk — at scale I'd use a `tdigest` approximate percentile."

**Follow-up:** "Difference between `PERCENTILE_CONT` and `PERCENTILE_DISC`?" → "CONT interpolates between neighbours (median of {10,20} = 15, may not exist in data); DISC returns a real data value (10 or 20)."

### Q3 — "Find each user's longest consecutive-day login streak."

**Junior answer:** `MAX(login_date) - MIN(login_date)` — ignores gaps entirely; a user who logged in on Jan 1 and Jan 30 shows a 29-day streak.

**Principal answer:** "Gaps-and-islands. First `DISTINCT` to one row per (user, day) — duplicate same-day logins break the arithmetic. Then `login_date - (ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY login_date))::int` gives a constant `grp` per consecutive run, because a real date and a dense counter both step by 1 inside a run and diverge at a gap. Group by `(user_id, grp)`, `COUNT(*)` is each streak's length, `MAX` of that per user is the longest. Timezone matters — I'd cast to the user's local date, not UTC, so late-night logins land on the right day."

**Follow-up:** "Now make it the *current* streak." → "Keep only the island whose `MAX(date)` is today or yesterday; filter with `HAVING MAX(login_date) >= CURRENT_DATE - 1`."

### Q4 — "Split these pageview events into sessions with a 30-minute timeout."

**Junior answer:** `GROUP BY user_id, date_trunc('hour', event_time)` — arbitrary fixed buckets that split an active session at the top of the hour and merge two distinct visits in the same hour.

**Principal answer:** "Sessionisation — two window passes. `LAG(event_time)` to get the previous event per user; flag a row as a new-session start when the gap exceeds 30 minutes or there's no previous row. Then a running `SUM` of that 0/1 flag (`ROWS UNBOUNDED PRECEDING`) assigns a dense session number — same running-sum-over-a-boolean trick as gaps-and-islands. Group by `(user_id, session_num)` for the session report. Both windows share `PARTITION BY user_id ORDER BY event_time`, so an index on `(user_id, event_time)` lets the planner sort once — or skip the sort entirely — and stack both `WindowAgg` nodes on the index scan."

**Follow-up:** "Sessions can also start on an explicit `login` event." → "Add `OR event_type = 'login'` inside the `CASE` — the running-sum machinery is unchanged."

---

## 15. Mental Model Checkpoint

1. Eight of these ten problems compile down to the same two physical operators. Which two, and what is the single index shape that most often eliminates the expensive one?

2. You wrote `WHERE ROW_NUMBER() OVER (PARTITION BY x ORDER BY y) = 1` and got a syntax error. Explain *from the logical execution order* why window functions can never be filtered in the same query level's `WHERE`, and what structure fixes it.

3. A running-total query gives two rows dated the same day the *identical* cumulative value instead of two increasing values. Which frame semantics caused it, and what one-word change fixes it?

4. In the gaps-and-islands date trick, why does `date - ROW_NUMBER()` stay constant across a consecutive run but jump at a gap? Walk through three consecutive dates and one gap.

5. You must return the *whole newest row* per email, not just aggregates. Explain why `GROUP BY email` cannot do this but `ROW_NUMBER()`/`DISTINCT ON` can.

6. When is `PERCENTILE_DISC(0.5)` the *correct* choice over `PERCENTILE_CONT(0.5)`? Give a concrete business example where returning an interpolated non-existent value would be wrong.

7. For a deep, frequently-read, occasionally-mutated category tree, argue for closure table over both adjacency list and nested sets — name the specific read cost, the specific write cost, and the storage cost of each.

---

## 16. Quick Reference Card

```sql
-- ══ TOP-N PER GROUP ══ (window; N=1 → DISTINCT ON; small groups+index → LATERAL)
SELECT * FROM (SELECT *, ROW_NUMBER() OVER (PARTITION BY g ORDER BY m DESC, id) rn
               FROM t) x WHERE rn <= N;
SELECT DISTINCT ON (g) * FROM t ORDER BY g, m DESC;          -- N=1
-- ROW_NUMBER=exactly N | RANK=ties incl (>N, gaps) | DENSE_RANK=top-N distinct values

-- ══ RUNNING TOTAL / MOVING AVG ══ (ALWAYS spell ROWS for per-row running total)
SUM(x)  OVER (ORDER BY d ROWS UNBOUNDED PRECEDING)              -- running total
AVG(x)  OVER (ORDER BY d ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)  -- 7-row MA
-- default frame = RANGE (peers merged) ← the running-total gotcha

-- ══ GAPS & ISLANDS / STREAKS ══ (value − dense counter = constant per run)
val   - ROW_NUMBER() OVER (ORDER BY val)                    AS grp   -- integer islands
date  - (ROW_NUMBER() OVER (PARTITION BY u ORDER BY date))::int AS grp -- day streaks
--   ↑ DISTINCT the dates first!  GROUP BY grp → MIN/MAX/COUNT = each run
LEAD(n) OVER (ORDER BY n) - n > 1                                    -- gap detection

-- ══ HIERARCHY ══
WITH RECURSIVE t AS (SELECT ..., ARRAY[id] path FROM x WHERE id=:root
  UNION ALL SELECT ..., path||c.id FROM x c JOIN t ON c.parent_id=t.id
  WHERE NOT c.id=ANY(t.path)) SELECT * FROM t ORDER BY path;   -- adjacency + cycle guard
-- nested set:  child.lft > parent.lft AND child.rgt < parent.rgt   (no recursion)
-- closure:     WHERE ancestor_id = :root AND depth > 0             (no recursion)

-- ══ SESSIONISATION ══ (LAG gap flag → running SUM = session id)
CASE WHEN t - LAG(t) OVER w > INTERVAL '30 min' OR LAG(t) OVER w IS NULL THEN 1 ELSE 0 END
SUM(flag) OVER (PARTITION BY u ORDER BY t ROWS UNBOUNDED PRECEDING) AS session_num

-- ══ DEDUP ══ (ROW_NUMBER over natural key, keep rn=1;  NOT plain DISTINCT)
... ROW_NUMBER() OVER (PARTITION BY natural_key ORDER BY updated_at DESC, id) ... WHERE rn=1
DELETE FROM t USING (...rn...) d WHERE t.id=d.id AND d.rn>1;

-- ══ MEDIAN / PERCENTILE ══
PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY x)   -- interpolated (statistical median)
PERCENTILE_DISC(0.5) WITHIN GROUP (ORDER BY x)   -- an actual observed value
-- manual: AVG(x) WHERE rn_asc IN ((n+1)/2,(n+2)/2)   [n = COUNT(*) OVER ()]

-- ══ PIVOT ══ (rows→cols; columns are fixed at parse time)
SUM(v) FILTER (WHERE month='2026-01') AS jan, ...        -- or SUM(CASE WHEN ...)

-- ══ SELF-REFERENCE ══
JOIN t2 self ON self.id = t.parent_id             -- row vs specific related row
LAG(x) OVER (ORDER BY d)                           -- row vs previous row
AVG(x) OVER (PARTITION BY g)                        -- row vs group aggregate
```

**Perf rules:** the Sort is the cost → index on `(PARTITION BY…, ORDER BY…)` kills it • median can't be indexed away (raise `work_mem`, or `tdigest` at scale) • recursion needs an index on the join column • batch giant backfills • materialise hot dashboard queries.

**Interview one-liners:** "Top-N per group = `ROW_NUMBER` + outer filter." • "Streak = gaps-and-islands: value minus a dense counter is constant per run." • "Session = `LAG` gap flag, running `SUM` for the id." • "Median isn't `AVG` — it's `PERCENTILE_CONT`." • "Pivot columns bind at parse time — dynamic pivots need app-built SQL." • "Dedup keeps the whole row — window function, not `DISTINCT`."

---

## Connected Topics

**Internals this builds on:**
- **Sort & WindowAgg** — the physical operators under eight of these ten patterns.
- **Recursive Union / worktable** — the engine machinery of hierarchical queries.
- **Ordered-set aggregates & tuplesort** — how `PERCENTILE_CONT` materialises and spills.
- **B-tree ordering** — the index property that eliminates the sort feeding every window.

**Prior SQL topics (the keys, taught individually):**
- **Window Functions (`ROW_NUMBER`, `RANK`, `LAG`/`LEAD`, frames)** — the engine of patterns 1–4, 6–7, 10.
- **Recursive CTEs** — the engine of hierarchical data.
- **`DISTINCT ON`** — the Postgres shorthand for top-1 and dedup.
- **`LATERAL` joins** — the index-driven alternative for top-N with few groups.
- **`GROUP BY` & aggregates, `FILTER`** — pivot and every per-group rollup.
- **Indexing for sort avoidance** — the perf lever throughout.
- **Topic 76 — System Design SQL Questions** — the preceding topic; these patterns are the building blocks of the analytics systems designed there.

**This is the final topic in the curriculum.** Everything from Phase 1 (relational foundations) through Phase 11 (interview readiness) converges here: recognise the lock, name the pattern, turn the key.
