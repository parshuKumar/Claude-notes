# Topic 36 — LAG and LEAD
### SQL Mastery Curriculum — Phase 6: Window Functions — Complete Mastery

---

## 1. ELI5 — The Analogy That Makes It Click

Imagine you're standing in a line of people at a bakery, and each person is holding a numbered ticket. You're curious about two simple things:

- **What number did the person *in front of me* have?** (That's `LAG`.)
- **What number does the person *behind me* have?** (That's `LEAD`.)

To answer, you don't leave your spot. You just turn your head — one step back, or one step forward — and read their ticket. You stay exactly where you are; you only *peek* at a neighbour.

Now here's the important part: the person at the very **front** of the line has *nobody* in front of them. If they turn around to look forward, they see empty air — nothing. And the person at the very **back** has nobody behind them; they turn back and also see nothing. That "nothing" is SQL's `NULL`. The edges of the line always have a blind spot on one side.

`LAG` and `LEAD` are exactly this: for every row, they let you glance at a neighbouring row — a fixed number of steps back (`LAG`) or forward (`LEAD`) — *without collapsing your data into a summary*. You keep all your rows; you just gain a new column that says "here's what the neighbour held."

The magic that makes "in front" and "behind" meaningful is the **line itself** — the order. Change how people are ordered (by ticket number, by arrival time, by height) and "the person in front of me" changes too. That's why every `LAG`/`LEAD` needs an `ORDER BY` inside its window: without an order, there's no "front" or "behind," just a shapeless crowd.

And if the bakery has multiple *separate* queues — one for bread, one for cakes — you'd never want the last person in the bread queue to peek at the first person in the cake queue. They're different lines. `PARTITION BY` draws those separate queues. The peek never crosses a partition boundary.

That's the whole idea. `LAG` = look back. `LEAD` = look forward. Order defines the direction. Partitions define the walls. Edges return `NULL`.

---

## 2. Connection to SQL Internals

`LAG` and `LEAD` are **window functions** (also called *analytic functions*). Unlike aggregate functions that collapse many rows into one, window functions produce one output value *per input row* while being able to see other rows in a defined "window." Understanding what the engine physically does is what separates a developer who *uses* them from one who can *reason about their cost*.

**The execution machinery — the WindowAgg node.** In PostgreSQL, every window function is executed by a plan node called `WindowAgg`. When you see `WindowAgg` in an `EXPLAIN`, that is the engine's window-function processor. It sits *above* the scan/join/filter nodes and *below* the final `ORDER BY`/`LIMIT`. Its job: consume an input stream, group it into partitions, walk each partition in the window's order, and emit the function's result for each row.

**Sorting is the hidden cost.** A window like `OVER (PARTITION BY customer_id ORDER BY created_at)` requires the input to arrive sorted by `(customer_id, created_at)`. If no index provides that order, the planner inserts a `Sort` node directly beneath the `WindowAgg`. This is the dominant cost of window queries at scale — an O(N log N) sort of the entire input, often spilling to disk (`external merge Disk`) when the data exceeds `work_mem`. If a B-tree index already delivers rows in the partition+order sequence, the planner can skip the sort entirely and stream — this is the single biggest optimization lever for `LAG`/`LEAD`.

**LAG/LEAD are cheap once sorted.** Here is the crucial internal insight: unlike `SUM() OVER (...)` with a moving frame, `LAG`/`LEAD` do **not** examine a *range* of rows. They access exactly *one* row at a fixed positional offset. The `WindowAgg` node maintains a **tuplestore** (a buffer of the current partition's rows, held in memory up to `work_mem`, then spilled to a temp file) and simply indexes into it: for `LAG(x, 3)` on the current row at position `i`, it reads position `i - 3`. There is no re-scan, no re-aggregation. The per-row cost is O(1). This is why `LAG`/`LEAD` are among the *cheapest* window functions once the sort is paid for.

**Frames don't matter for LAG/LEAD.** `SUM`, `AVG`, `FIRST_VALUE`, `LAST_VALUE` are sensitive to the window *frame* (`ROWS BETWEEN ...`). `LAG` and `LEAD` **ignore the frame entirely** — they are defined purely by positional offset from the current row, independent of `ROWS`/`RANGE`/`GROUPS` framing. You can specify a frame clause and it will be silently irrelevant to their result. (Contrast this with Topic 37's `FIRST_VALUE`/`LAST_VALUE`, which are *acutely* frame-sensitive — the famous `LAST_VALUE` trap.)

**Memory model.** The tuplestore for the current partition is the memory footprint. For `LAG`/`LEAD`, the engine technically only needs to buffer `offset + 1` rows to look backward, but PostgreSQL's `WindowAgg` buffers whatever the partition and any peer functions require. In practice, plan for the partition-sized tuplestore, bounded by `work_mem` before it spills.

So the mental model at the internals level: **`LAG`/`LEAD` = (optional sort) + a positional pointer into a per-partition buffer.** The sort is the expense; the peek is nearly free.

---

## 3. Logical Execution Order Context

Window functions occupy a specific, late slot in SQL's logical processing pipeline. Getting this order right resolves almost every "why can't I reference my window column here?" confusion.

```
FROM / JOIN          ← tables assembled, rows produced
WHERE                ← row filtering (BEFORE window functions exist)
GROUP BY             ← grouping
HAVING               ← group filtering
SELECT               ← expressions evaluated, INCLUDING window functions ← LAG/LEAD run HERE
DISTINCT
ORDER BY             ← final output ordering
LIMIT / OFFSET
```

The critical facts that follow from this ordering:

**1. Window functions run AFTER `WHERE`, `GROUP BY`, and `HAVING`.** By the time `LAG`/`LEAD` execute, the row set is already filtered and grouped. This means a `WHERE` clause **cannot** filter on the result of a window function, and a window function operates only over the rows that survived `WHERE`. If you filter out yesterday's row in `WHERE`, then `LAG` "looking back one day" will skip to the last row that *survived* — it does not see filtered-out rows.

**2. You cannot use a `LAG`/`LEAD` result in the same query's `WHERE` or `HAVING`.** This is the number-one beginner error:

```sql
-- ILLEGAL: window function in WHERE
SELECT created_at, amount, LAG(amount) OVER (ORDER BY created_at) AS prev
FROM payments
WHERE LAG(amount) OVER (ORDER BY created_at) < amount;  -- ERROR
-- ERROR: window functions are not allowed in WHERE
```

The window column does not "exist" yet when `WHERE` runs. The fix is to compute the window function in a subquery/CTE, then filter in an outer query (shown throughout Section 6).

**3. Window functions run in `SELECT`, but they can reference columns as they exist *after* `GROUP BY`.** You can layer `LAG` on top of an aggregate:

```sql
SELECT
  DATE_TRUNC('month', created_at) AS month,
  SUM(amount) AS monthly_total,
  LAG(SUM(amount)) OVER (ORDER BY DATE_TRUNC('month', created_at)) AS prev_month_total
FROM payments
GROUP BY DATE_TRUNC('month', created_at);
```

Here `SUM(amount)` is computed by `GROUP BY`, and *then* `LAG` peeks at the previous group's sum. The window runs on the grouped output. This composition — window-over-aggregate — is one of the most powerful patterns in analytics SQL.

**4. `ORDER BY` inside `OVER()` is independent of the query's final `ORDER BY`.** The `ORDER BY` in the window clause defines the sequence *for the peek*; the query's trailing `ORDER BY` defines the sequence of the *output rows*. They can differ. A common pattern: order the window by time to compute `LAG`, but present results newest-first with a final `ORDER BY created_at DESC`.

Understanding this pipeline is why the "compute in a CTE, filter outside" pattern is not a workaround — it is the *correct* structural expression of the logical model.

---

## 4. What Is LAG / LEAD?

`LAG` and `LEAD` are **offset window functions**. For each row, within its partition and following the window's `ORDER BY`, `LAG` returns a column value from a row a given number of positions *before* the current row, and `LEAD` returns a value from a row a given number of positions *after*. When no such row exists (partition edge), they return a default value (`NULL` unless you supply one).

```sql
LAG(value_expr [, offset [, default]])  OVER (
    [PARTITION BY partition_expr, ...]
    ORDER BY sort_expr [ASC|DESC] [NULLS FIRST|LAST], ...
)

LEAD(value_expr [, offset [, default]]) OVER (
    [PARTITION BY partition_expr, ...]
    ORDER BY sort_expr [ASC|DESC] [NULLS FIRST|LAST], ...
)
```

Annotated breakdown of every part:

```sql
LAG(                         -- look BACKWARD (toward earlier rows in the window order)
    o.total_amount,          -- │ value_expr: the column/expression to read from the neighbour row
    1,                       -- │ offset: how many rows back (default = 1). LAG(x,2) = two rows back
    0                        -- │ default: value to return at the partition edge instead of NULL
)                            -- └── returns ONE value per row: the neighbour's value_expr
OVER (                       -- the window definition: defines partitions + order for "neighbour"
    PARTITION BY             -- │ split rows into independent groups; peek never crosses a boundary
        o.customer_id        -- │ └── one independent sequence per customer
    ORDER BY                 -- │ REQUIRED: defines what "before"/"after" means within a partition
        o.created_at ASC     -- │ └── earliest first; row i's "previous" is row i-1 in this order
)                            -- └── window clause closes
```

The same annotation applies to `LEAD`, with the sole difference that the offset is applied in the *forward* direction:

```sql
LEAD(                        -- look FORWARD (toward later rows in the window order)
    o.created_at,            -- │ value_expr: read created_at from the NEXT row
    1,                       -- │ offset: 1 row ahead (default)
    NULL                     -- │ default: explicit NULL at the last row of the partition (redundant here)
)
OVER (
    PARTITION BY o.customer_id
    ORDER BY o.created_at
)
```

### Exact semantics

Given one customer's orders, ordered by `created_at`:

```
row | created_at | amount |  LAG(amount)  | LEAD(amount)
----+------------+--------+---------------+-------------
 1  | 2026-01-05 |  100   |   NULL  ←edge |    150
 2  | 2026-01-09 |  150   |   100         |    150
 3  | 2026-01-20 |  150   |   150         |     80
 4  | 2026-02-02 |   80   |   150         |   NULL ←edge
```

- Row 1 has no earlier row → `LAG` is `NULL` (the top edge).
- Row 4 has no later row → `LEAD` is `NULL` (the bottom edge).
- Every interior row reads its immediate neighbour.
- Note `LAG` and `LEAD` are *mirror images*: `LEAD(x)` on row *i* equals `LAG(x)` read from the perspective of row *i+1*. Anything `LAG` can express with a reversed `ORDER BY`, `LEAD` can express directly — they are duals.

### The three arguments in detail

| Argument | Required? | Default | Meaning |
|----------|-----------|---------|---------|
| `value_expr` | Yes | — | The expression whose value is fetched from the offset row. Can be any expression: a column, `amount * 1.1`, `created_at`, even another function's output. |
| `offset` | No | `1` | Non-negative integer: how many rows away. `LAG(x, 0)` returns the *current* row's value (a degenerate but legal case). Must be non-negative; a negative offset raises an error. |
| `default` | No | `NULL` | Value returned when the offset row falls outside the partition. Must be type-compatible with `value_expr`. |

`INNER JOIN` was the default join; here, `offset = 1` is the default step. Just as omitting `INNER` still means inner join, omitting the offset still means "one row."

---

## 5. Why LAG / LEAD Mastery Matters in Production

1. **Row-over-row change is a universal reporting requirement.** "Day-over-day revenue change," "month-over-month growth %," "was this reading higher than the last?" — every dashboard, every finance report, every metrics pipeline needs to compare a row to its predecessor. Without `LAG`, developers reach for self-joins that are slower, buggier, and mishandle gaps. Mastering `LAG` turns a 40-line correlated-subquery mess into three lines.

2. **Gap and session detection depend on it.** Detecting the time elapsed between consecutive events (`created_at - LAG(created_at)`) is the foundation of *sessionization* (grouping events into user sessions), *idle-time detection*, *fraud velocity checks* ("three logins within 10 seconds"), and *SLA breach detection*. This is not exotic — it is core product analytics.

3. **Detecting value changes drives change-data-capture and state machines.** "When did the order status change?", "which rows differ from the row before them?", "collapse consecutive duplicate readings" — all built on `value <> LAG(value)`. This underpins audit trails, deduplication, and state-transition analysis over `audit_logs`.

4. **The wrong alternative is catastrophically slow.** The naive alternative to `LAG` is a correlated subquery that, for each row, scans for "the previous row" — an O(N²) pattern that turns a 5-second query into an hours-long one at 10M rows. Understanding that `LAG` is O(N log N) (one sort) vs O(N²) (self-correlate) is a genuine production-incident-preventing insight.

5. **Silent NULL-at-edge bugs corrupt aggregates.** `LAG` returns `NULL` at partition starts. If you then compute `amount - LAG(amount)` and `SUM` it, or divide by it, the `NULL`s silently propagate or (worse) `NULL / 0` style errors appear. Knowing exactly *where* the `NULL`s land — and using the `default` argument or `COALESCE` deliberately — is the difference between a correct and a subtly-wrong report.

6. **Partition boundaries are a correctness trap.** Forgetting `PARTITION BY customer_id` means the *last order of customer A* peeks at the *first order of customer B* — producing a nonsensical "change" across unrelated entities. This bug passes code review, passes small-dataset tests, and only surfaces as garbage numbers in production. Understanding partitioning is a correctness requirement, not an optimization.

---

## 6. Deep Technical Content

### 6.1 The Anatomy of an Offset Function

`LAG(expr, offset, default)` answers: "Reading the current partition in `ORDER BY` sequence, what was `expr` `offset` rows ago? If that lands outside the partition, give me `default`." `LEAD` is identical but counts forward. Three moving parts fully determine the result: **the value expression**, **the offset**, and **the partition+order** that defines the sequence.

The most under-appreciated fact: **`value_expr` is evaluated on the offset row, not the current row.** `LAG(amount * quantity)` computes `amount * quantity` *using the previous row's* `amount` and `quantity`. It is not "the previous amount times the current quantity." The entire expression is teleported to the neighbour row.

### 6.2 The offset parameter — beyond 1

The offset can be any non-negative integer, letting you reach further back or forward:

```sql
SELECT
  reading_at,
  temperature,
  LAG(temperature, 1) OVER w AS prev,          -- 1 reading ago
  LAG(temperature, 3) OVER w AS three_ago,     -- 3 readings ago (e.g., 3 hours if hourly)
  LEAD(temperature, 1) OVER w AS next,         -- 1 reading ahead
  LAG(temperature, 0) OVER w AS self           -- degenerate: the current row itself
FROM sensor_readings
WINDOW w AS (PARTITION BY sensor_id ORDER BY reading_at);
```

- `LAG(x, 0)` returns the current row's value. Legal, occasionally useful in generated SQL, but usually a mistake.
- `LAG(x, 7)` on daily data gives you *week-over-week* comparison (same weekday last week).
- A **negative offset raises an error**: `LAG(x, -1)` → `ERROR: argument of ntile must be greater than zero` style error; use `LEAD(x, 1)` instead. `LAG(x, -1)` is conceptually `LEAD(x, 1)`, but SQL will not silently invert it — it errors. Compute the direction explicitly.

The `offset` may be an expression, but it must be an integer and is evaluated **once per row** — you *can* vary it per row, though this is rare and can confuse the planner:

```sql
LAG(price, days_offset) OVER (ORDER BY trade_date)  -- offset varies by row; legal but unusual
```

### 6.3 The default parameter — controlling the edge

By default, `LAG`/`LEAD` return `NULL` when the offset row is outside the partition. The third argument replaces that `NULL`:

```sql
-- Day-over-day change; treat the first day's "previous" as 0 so change = today's value
SELECT
  day,
  revenue,
  revenue - LAG(revenue, 1, 0) OVER (ORDER BY day) AS change_vs_prev
FROM daily_revenue;
```

Without the `0` default, the first row's `change_vs_prev` would be `revenue - NULL = NULL`. With it, the first day shows its full revenue as the "change" — which may or may not be what you want. **The default is a semantic decision, not a formatting convenience.** Ask: "at the edge, what is the *correct* comparison baseline?"

Two subtle rules:

1. **The default must be type-compatible with `value_expr`.** `LAG(status, 1, 'none')` works for text; `LAG(amount, 1, 0)` for numeric. `LAG(amount, 1, 'unknown')` → type error.
2. **The default is only used at the edge, never for interior NULLs.** If an *interior* row's `value_expr` is genuinely `NULL` (the column value is NULL), `LAG` returns that `NULL` — the default does **not** kick in. The default only substitutes for "no such row." This distinction matters when your data itself contains NULLs (Section 6.10).

### 6.4 PARTITION BY — the walls between sequences

Without `PARTITION BY`, the entire result set is one giant sequence, and `LAG` on the first order of customer B reads the last order of customer A. Almost always wrong.

```sql
-- WRONG: no partition — bleeds across customers
LAG(amount) OVER (ORDER BY created_at)

-- RIGHT: each customer is an independent sequence
LAG(amount) OVER (PARTITION BY customer_id ORDER BY created_at)
```

With `PARTITION BY customer_id`, `LAG` resets to `NULL` at the first row of *each* customer. The partition boundary is a hard wall: the peek never crosses it, and `default` (or `NULL`) appears at every partition's leading edge. This is exactly what you want — "previous order *for this customer*," not "previous order in the whole table."

You can partition on multiple columns: `PARTITION BY customer_id, currency` gives an independent sequence per (customer, currency) pair.

### 6.5 ORDER BY — mandatory and meaningful

`LAG`/`LEAD` are **meaningless without `ORDER BY`** in the window. PostgreSQL *permits* omitting it syntactically, but the result is then non-deterministic — "previous row" in an undefined order is whatever physical order the executor happened to produce, which can change between runs, plans, or PostgreSQL versions. **Always specify `ORDER BY`.**

The `ORDER BY` direction flips the meaning of `LAG`/`LEAD`:

```sql
LAG(amount)  OVER (ORDER BY created_at ASC)   -- previous = chronologically earlier
LAG(amount)  OVER (ORDER BY created_at DESC)  -- previous = chronologically LATER (reversed!)
```

This is the duality from Section 4: `LAG` under `ORDER BY created_at DESC` behaves like `LEAD` under `ORDER BY created_at ASC`. Choose one convention and be deliberate.

**Ties in ORDER BY.** If two rows have identical `ORDER BY` values (same `created_at`), their relative order is *undefined*, so which one `LAG` picks is non-deterministic. Break ties with a unique tiebreaker:

```sql
LAG(amount) OVER (PARTITION BY customer_id ORDER BY created_at, id)  -- id breaks the tie deterministically
```

Without the tiebreaker, two orders placed at the same timestamp may swap positions between executions, making `LAG` return different neighbours run-to-run. This is a genuine source of "flaky" report numbers.

### 6.6 Use Case: Day-over-Day / Row-over-Row Change

The canonical use. Compute the delta and percentage change versus the previous period:

```sql
SELECT
  day,
  revenue,
  LAG(revenue) OVER w AS prev_revenue,
  revenue - LAG(revenue) OVER w AS abs_change,
  ROUND(
    100.0 * (revenue - LAG(revenue) OVER w) / NULLIF(LAG(revenue) OVER w, 0),
    2
  ) AS pct_change
FROM daily_revenue
WINDOW w AS (ORDER BY day)
ORDER BY day;
```

Two production essentials baked in:
- **`NULLIF(prev, 0)`** guards against division by zero when the previous value was 0.
- The **first row** naturally shows `NULL` for `prev_revenue`, `abs_change`, and `pct_change` — correct, because there is no prior period. Add a `default` if a baseline is desired.

### 6.7 Use Case: Finding Gaps Between Events

Subtract the previous event's timestamp from the current to get inter-event duration:

```sql
SELECT
  user_id,
  created_at,
  created_at - LAG(created_at) OVER (
    PARTITION BY user_id ORDER BY created_at
  ) AS gap
FROM sessions
ORDER BY user_id, created_at;
```

`gap` is an `interval`. The first event per user has `gap = NULL` (no prior event). You can then find "large gaps" (idle periods) in an outer query:

```sql
SELECT * FROM (
  SELECT
    user_id, created_at,
    created_at - LAG(created_at) OVER (PARTITION BY user_id ORDER BY created_at) AS gap
  FROM sessions
) t
WHERE gap > INTERVAL '30 minutes';  -- must filter in outer query — window result not visible in WHERE
```

Note the mandatory subquery: you cannot put `gap > INTERVAL '30 minutes'` in the same-level `WHERE` (Section 3, rule 2).

### 6.8 Use Case: Session Detection (Sessionization)

The classic funnel: group a user's events into sessions where a gap larger than a threshold (say 30 minutes) starts a new session. This is a three-layer pattern — `LAG` to find gaps, a boolean "new session" flag, then a running `SUM` to assign session numbers:

```sql
WITH gaps AS (
  SELECT
    user_id,
    created_at,
    LAG(created_at) OVER (PARTITION BY user_id ORDER BY created_at) AS prev_at
  FROM sessions
),
flagged AS (
  SELECT
    user_id, created_at, prev_at,
    -- new session when this is the first event OR the gap exceeds 30 min
    CASE
      WHEN prev_at IS NULL THEN 1
      WHEN created_at - prev_at > INTERVAL '30 minutes' THEN 1
      ELSE 0
    END AS is_new_session
  FROM gaps
)
SELECT
  user_id,
  created_at,
  -- cumulative count of "new session" flags = the session number
  SUM(is_new_session) OVER (
    PARTITION BY user_id ORDER BY created_at
    ROWS UNBOUNDED PRECEDING
  ) AS session_number
FROM flagged
ORDER BY user_id, created_at;
```

This "gaps-and-islands" technique — `LAG` to detect boundaries, running `SUM` to number the islands — is one of the most valuable patterns in analytical SQL. Memorize its shape.

### 6.9 Use Case: Detecting Value Changes (State Transitions)

Find the rows where a value differs from the immediately preceding row — e.g., when an order's status changed in `audit_logs`:

```sql
SELECT *
FROM (
  SELECT
    order_id,
    status,
    changed_at,
    LAG(status) OVER (PARTITION BY order_id ORDER BY changed_at) AS prev_status
  FROM audit_logs
) t
WHERE prev_status IS DISTINCT FROM status  -- catches changes AND the first row (NULL -> status)
ORDER BY order_id, changed_at;
```

The critical operator here is **`IS DISTINCT FROM`**, not `<>`. Compare:

- `prev_status <> status`: when `prev_status IS NULL` (the first row), `NULL <> 'pending'` evaluates to `UNKNOWN`, so the first row is **dropped** — you lose the initial state.
- `prev_status IS DISTINCT FROM status`: `NULL IS DISTINCT FROM 'pending'` is `TRUE`, so the first row is **kept**. `IS DISTINCT FROM` treats `NULL` as a comparable value.

Choosing `<>` vs `IS DISTINCT FROM` is a genuine correctness decision. If you want to *include* the first appearance as a "change," use `IS DISTINCT FROM`. If you want *only interior transitions*, `<>` (with its NULL-drop) may be acceptable — but do it knowingly, not accidentally.

To **collapse consecutive duplicates** (deduplicate a stream of repeated readings), keep only rows where the value changed:

```sql
SELECT sensor_id, reading_at, value
FROM (
  SELECT sensor_id, reading_at, value,
         LAG(value) OVER (PARTITION BY sensor_id ORDER BY reading_at) AS prev_value
  FROM sensor_readings
) t
WHERE prev_value IS DISTINCT FROM value;  -- one row per "run" of identical values
```

### 6.10 NULL Behaviour — Two Completely Different NULLs

This is the highest-value subtlety in the entire topic. `LAG`/`LEAD` involve **two independent sources of NULL**, and conflating them causes silent bugs:

**NULL source A — the partition edge.** When the offset row doesn't exist (start/end of partition), `LAG`/`LEAD` return `NULL` (or the `default`). This is "there is no neighbour."

**NULL source B — a genuinely NULL value in the data.** If the neighbour row *exists* but its `value_expr` *is* `NULL`, `LAG` returns that `NULL`. This is "the neighbour's value happens to be NULL."

These are indistinguishable in the output — both show `NULL` — but they mean different things, and the `default` argument only handles source A:

```sql
-- amount can be NULL for pending payments
SELECT
  id, amount,
  LAG(amount, 1, -999) OVER (ORDER BY id) AS prev
FROM payments;
```

- At the first row: no neighbour → `default` used → `prev = -999`.
- At an interior row whose *previous row had `amount = NULL`*: neighbour exists, its value is `NULL` → `prev = NULL` (**default NOT applied**).

So a `-999` means "edge," but a `NULL` means "the previous amount was genuinely null." If you need to skip NULL values and find the "last non-null previous amount," `LAG` alone cannot do it — that requires the `ignore nulls` semantics (not supported by PostgreSQL's `LAG` prior to advanced techniques) or a workaround:

```sql
-- Emulate "last non-null previous value" (LAG IGNORE NULLS is not built-in to PG's lag)
SELECT
  id, amount,
  (array_agg(amount) FILTER (WHERE amount IS NOT NULL)
     OVER (ORDER BY id ROWS BETWEEN UNBOUNDED PRECEDING AND 1 PRECEDING))[
     COUNT(amount) FILTER (WHERE amount IS NOT NULL)
       OVER (ORDER BY id ROWS BETWEEN UNBOUNDED PRECEDING AND 1 PRECEDING)] AS last_non_null_prev
FROM payments;
```

That is deliberately ugly to make the point: **PostgreSQL's `LAG` does not support `IGNORE NULLS`** (unlike Oracle). If you need last-non-null, you use `FILTER` + array tricks, or a `SUM/COUNT`-based carry-forward, or upgrade patterns in Topic 37. For the common case, know that `LAG` is *NULL-transparent*: it faithfully returns whatever the neighbour holds, including `NULL`.

**NULLS FIRST / NULLS LAST in the ORDER BY** also affects *where* NULL-keyed rows sit in the sequence, which changes who is "previous." If `created_at` can be `NULL`, decide with `ORDER BY created_at NULLS LAST` where those rows land.

### 6.11 LAG and LEAD together — bracketing a row

You can use both in one query to see a row's full neighbourhood:

```sql
SELECT
  reading_at,
  value,
  LAG(value)  OVER w AS prev_value,
  LEAD(value) OVER w AS next_value,
  -- detect a local peak: higher than both neighbours
  value > LAG(value) OVER w AND value > LEAD(value) OVER w AS is_local_peak
FROM sensor_readings
WINDOW w AS (PARTITION BY sensor_id ORDER BY reading_at);
```

The `WINDOW w AS (...)` named-window clause (from Topic 33) lets multiple functions share one window definition — cleaner and, crucially, the planner sorts **once** for all functions sharing the window rather than once per function.

### 6.12 Frame clause is IGNORED by LAG/LEAD

A trap for the over-careful. You might add a frame thinking it constrains `LAG`:

```sql
LAG(amount) OVER (ORDER BY created_at ROWS BETWEEN 1 PRECEDING AND CURRENT ROW)
```

The `ROWS BETWEEN ...` here is **completely ignored**. `LAG`/`LEAD` are defined by positional offset, independent of frame. The result is identical to `LAG(amount) OVER (ORDER BY created_at)`. Do not add frames to `LAG`/`LEAD` expecting them to do anything — they won't, and it misleads the next reader. (This is the inverse of the `LAST_VALUE` trap in Topic 37, where the frame is *essential*.)

### 6.13 Composing LAG over aggregates — month-over-month

The window runs after `GROUP BY`, so you can `LAG` an aggregate:

```sql
SELECT
  DATE_TRUNC('month', o.created_at)::date AS month,
  SUM(o.total_amount) AS monthly_revenue,
  LAG(SUM(o.total_amount)) OVER (ORDER BY DATE_TRUNC('month', o.created_at)) AS prev_month,
  ROUND(
    100.0 * (SUM(o.total_amount) - LAG(SUM(o.total_amount)) OVER (ORDER BY DATE_TRUNC('month', o.created_at)))
    / NULLIF(LAG(SUM(o.total_amount)) OVER (ORDER BY DATE_TRUNC('month', o.created_at)), 0),
    1
  ) AS mom_growth_pct
FROM orders o
WHERE o.status = 'completed'
GROUP BY DATE_TRUNC('month', o.created_at)
ORDER BY month;
```

Each `SUM(o.total_amount)` is a per-month aggregate; `LAG` peeks at the previous month's aggregate. This window-over-aggregate composition is the backbone of period-over-period reporting.

### 6.14 Multiple partitions, multiple orders

Different `LAG`/`LEAD` expressions in one `SELECT` can use *different* windows — each is independent:

```sql
SELECT
  order_id,
  customer_id,
  created_at,
  amount,
  LAG(amount)  OVER (PARTITION BY customer_id ORDER BY created_at) AS prev_for_customer,
  LAG(amount)  OVER (ORDER BY created_at)                          AS prev_globally
FROM orders;
```

`prev_for_customer` resets per customer; `prev_globally` is the previous order across the whole table. Two different windows → the planner may need two sorts (unless one order subsumes the other). Watch the `EXPLAIN` for multiple `Sort` + `WindowAgg` nodes.

---

## 7. EXPLAIN — WindowAgg in the Plan

### Case A: Sort required (no supporting index)

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT
  customer_id,
  created_at,
  total_amount,
  LAG(total_amount) OVER (PARTITION BY customer_id ORDER BY created_at) AS prev
FROM orders;
```

```
WindowAgg  (cost=134247.68..154247.68 rows=1000000 width=32)
           (actual time=612.4..1123.7 rows=1000000 loops=1)
  ->  Sort  (cost=134247.68..136747.68 rows=1000000 width=24)
            (actual time=612.3..789.1 rows=1000000 loops=1)
        Sort Key: customer_id, created_at
        Sort Method: external merge  Disk: 34112kB
        ->  Seq Scan on orders  (cost=0.00..18334.00 rows=1000000 width=24)
                                (actual time=0.02..112.6 rows=1000000 loops=1)
Buffers: shared hit=128 read=8206, temp read=4264 written=4278
Planning Time: 0.21 ms
Execution Time: 1187.4 ms
```

**Reading it line by line:**
- **`WindowAgg`** — the top node; this is where `LAG` is computed. Its incremental cost over the Sort (`154247 - 136747 ≈ 17500`) reflects the near-free per-row peek: cheap once sorted.
- **`Sort`** — the expensive node. `Sort Key: customer_id, created_at` exactly matches `PARTITION BY customer_id ORDER BY created_at`. The `WindowAgg` requires this order.
- **`Sort Method: external merge  Disk: 34112kB`** — the killer. The sort exceeded `work_mem` and spilled ~34MB to disk. `temp read=4264 written=4278` confirms temp-file I/O. This is the dominant cost.
- **`Seq Scan on orders`** — full scan; no filter, so all 1M rows feed the sort.
- **Execution Time 1187ms** — dominated by the disk-spilling sort, not the `LAG`.

**Fix direction:** provide the sort order via an index, or raise `work_mem` so the sort stays in memory.

### Case B: Index provides order — no Sort node

With `CREATE INDEX idx_orders_cust_created ON orders(customer_id, created_at);`:

```
WindowAgg  (cost=0.43..71000.43 rows=1000000 width=32)
           (actual time=0.05..534.2 rows=1000000 loops=1)
  ->  Index Scan using idx_orders_cust_created on orders
        (cost=0.43..46000.43 rows=1000000 width=24)
        (actual time=0.03..245.8 rows=1000000 loops=1)
Buffers: shared hit=812 read=9333
Planning Time: 0.18 ms
Execution Time: 588.9 ms
```

**Reading it:**
- **No `Sort` node.** The `Index Scan` on `(customer_id, created_at)` delivers rows already in partition+order sequence, so the `WindowAgg` streams directly. This is the optimal shape.
- **No `temp` I/O** — no disk spill, because there's no sort to overflow.
- **Execution Time 588ms vs 1187ms** — roughly 2× faster purely by eliminating the sort. On wider rows or larger tables the gap widens dramatically.

### Case C: Window over aggregate (Sort after GROUP BY)

```sql
EXPLAIN (ANALYZE)
SELECT DATE_TRUNC('month', created_at) AS m, SUM(total_amount) AS rev,
       LAG(SUM(total_amount)) OVER (ORDER BY DATE_TRUNC('month', created_at))
FROM orders GROUP BY 1;
```

```
WindowAgg  (cost=... rows=24 width=...)
  ->  Sort  (cost=... rows=24)
        Sort Key: (date_trunc('month', created_at))
        ->  HashAggregate  (cost=... rows=24)
              Group Key: date_trunc('month', created_at)
              ->  Seq Scan on orders
```

**Reading it:** `HashAggregate` collapses 1M rows into 24 monthly buckets *first*; the `Sort` then orders just those 24 rows (trivially cheap); `WindowAgg` runs `LAG` over 24 rows. Because the window runs on the *aggregated* output, the sort is over 24 rows, not 1M — window-over-aggregate is cheap even without an index.

### What to look for

| Symptom in EXPLAIN | Meaning | Fix |
|--------------------|---------|-----|
| `Sort Method: external merge  Disk: NkB` | Sort spilled to disk | Raise `work_mem`, or add composite index matching PARTITION BY + ORDER BY |
| `Sort` node beneath `WindowAgg` | No index supplies the window order | `CREATE INDEX (partition_cols, order_cols)` to remove it |
| Multiple `WindowAgg` + `Sort` nodes | Different windows need different sorts | Share one `WINDOW w AS (...)`; align window orders where possible |
| `WindowAgg` cost ≈ child cost | Peek is cheap; cost is all in the child | Optimize the child (scan/sort), not the window |

---

## 8. Query Examples

### Example 1 — Basic: Previous order amount per customer

```sql
-- For each order, show the customer's PREVIOUS order amount (chronologically).
-- First order per customer -> prev_amount is NULL (no earlier order).
SELECT
  o.customer_id,
  o.id                 AS order_id,
  o.created_at,
  o.total_amount,
  LAG(o.total_amount) OVER (
    PARTITION BY o.customer_id     -- independent sequence per customer
    ORDER BY o.created_at, o.id    -- chronological; id breaks timestamp ties deterministically
  ) AS prev_amount
FROM orders o
ORDER BY o.customer_id, o.created_at;
```

### Example 2 — Intermediate: Day-over-day revenue change with % and edge handling

```sql
-- Daily completed revenue, absolute change, and % change vs the previous day.
WITH daily AS (
  SELECT
    DATE_TRUNC('day', created_at)::date AS day,
    SUM(total_amount)                   AS revenue
  FROM orders
  WHERE status = 'completed'
  GROUP BY DATE_TRUNC('day', created_at)
)
SELECT
  day,
  revenue,
  LAG(revenue) OVER w                                  AS prev_revenue,
  revenue - LAG(revenue) OVER w                        AS abs_change,
  ROUND(
    100.0 * (revenue - LAG(revenue) OVER w)
    / NULLIF(LAG(revenue) OVER w, 0),                  -- guard divide-by-zero
    2
  )                                                    AS pct_change
FROM daily
WINDOW w AS (ORDER BY day)                             -- shared window: one sort for all three LAGs
ORDER BY day;
```

### Example 3 — Production Grade: Sessionization with gap detection

```sql
-- CONTEXT:
--   sessions table: ~50M rows (user_id, created_at, event_type, ...)
--   Index available: idx_sessions_user_created ON sessions(user_id, created_at)
--   Goal: assign a session number per user; a gap > 30 min starts a new session.
--   Perf expectation: with the composite index, NO sort node -> streams;
--     ~50M rows scanned in index order, WindowAggs are O(1)/row.
--     Realistic runtime: a few seconds, bounded by index scan + I/O, not sorting.

WITH gapped AS (
  SELECT
    user_id,
    created_at,
    event_type,
    LAG(created_at) OVER w AS prev_at          -- previous event time for this user
  FROM sessions
  WHERE created_at >= NOW() - INTERVAL '30 days'
  WINDOW w AS (PARTITION BY user_id ORDER BY created_at)
),
flagged AS (
  SELECT
    user_id, created_at, event_type,
    CASE
      WHEN prev_at IS NULL                              THEN 1  -- first event = new session
      WHEN created_at - prev_at > INTERVAL '30 minutes' THEN 1  -- large gap = new session
      ELSE 0
    END AS is_new_session
  FROM gapped
)
SELECT
  user_id,
  created_at,
  event_type,
  SUM(is_new_session) OVER (
    PARTITION BY user_id ORDER BY created_at
    ROWS UNBOUNDED PRECEDING                             -- running count = session number
  ) AS session_number
FROM flagged
ORDER BY user_id, created_at;
```

EXPLAIN shape for Example 3 (abbreviated):

```
WindowAgg  (session_number running SUM)
  ->  WindowAgg  (LAG prev_at)
        ->  Index Scan using idx_sessions_user_created on sessions
              Index Cond: (created_at >= now() - interval '30 days')  -- if it helps; else filter
```

Both `WindowAgg` nodes share the `(user_id, created_at)` order the index already provides, so **no Sort node appears** — the whole pipeline streams. This is the difference between a query that finishes in seconds and one that spills 50M rows to disk.

---

## 9. Wrong → Right Patterns

### Wrong 1: Window function in WHERE

```sql
-- WRONG: filtering on a LAG result in the same-level WHERE
SELECT customer_id, created_at, total_amount,
       LAG(total_amount) OVER (PARTITION BY customer_id ORDER BY created_at) AS prev
FROM orders
WHERE total_amount > LAG(total_amount) OVER (PARTITION BY customer_id ORDER BY created_at);
-- EXACT ERROR: ERROR: window functions are not allowed in WHERE
--   LINE ...: WHERE total_amount > LAG(total_amount) OVER (...)
```

**Why it fails at the execution level:** `WHERE` runs in the logical pipeline *before* `SELECT`, and window functions are evaluated in the `SELECT` phase (Section 3). The `LAG` result does not exist when `WHERE` is applied.

```sql
-- RIGHT: compute in a subquery/CTE, filter in the outer query
SELECT *
FROM (
  SELECT customer_id, created_at, total_amount,
         LAG(total_amount) OVER (PARTITION BY customer_id ORDER BY created_at) AS prev
  FROM orders
) t
WHERE total_amount > prev;   -- now `prev` exists as a plain column
```

### Wrong 2: Forgetting PARTITION BY — cross-entity bleed

```sql
-- WRONG: no PARTITION BY. The first order of each customer reads the
-- LAST order of the PREVIOUS customer in created_at order.
SELECT customer_id, created_at, total_amount,
       total_amount - LAG(total_amount) OVER (ORDER BY created_at) AS change
FROM orders;
-- WRONG RESULT: at every customer boundary, `change` is a nonsense comparison
-- between two unrelated customers' orders. Numbers look plausible but are garbage.
```

**Why it's wrong:** `LAG` sees one global sequence; the peek crosses customer boundaries. There is no error — just silently incorrect data, the most dangerous kind.

```sql
-- RIGHT: partition so each customer is an isolated sequence
SELECT customer_id, created_at, total_amount,
       total_amount - LAG(total_amount) OVER (
         PARTITION BY customer_id ORDER BY created_at
       ) AS change
FROM orders;
-- Now `change` at each customer's first order is NULL (correct: no prior order).
```

### Wrong 3: Using `<>` instead of `IS DISTINCT FROM` for change detection

```sql
-- WRONG: dropping the first status because NULL <> x is UNKNOWN
SELECT * FROM (
  SELECT order_id, status, changed_at,
         LAG(status) OVER (PARTITION BY order_id ORDER BY changed_at) AS prev
  FROM audit_logs
) t
WHERE prev <> status;   -- first row per order has prev = NULL -> NULL <> status -> UNKNOWN -> dropped
-- WRONG RESULT: the initial status of every order is missing from the output.
```

**Why:** three-valued logic — `NULL <> 'pending'` is `UNKNOWN`, and `WHERE` keeps only `TRUE`, so first rows vanish.

```sql
-- RIGHT: IS DISTINCT FROM treats NULL as comparable -> keeps first appearance
SELECT * FROM (
  SELECT order_id, status, changed_at,
         LAG(status) OVER (PARTITION BY order_id ORDER BY changed_at) AS prev
  FROM audit_logs
) t
WHERE prev IS DISTINCT FROM status;  -- NULL IS DISTINCT FROM 'pending' -> TRUE -> kept
```

### Wrong 4: Assuming `default` fixes interior NULLs

```sql
-- WRONG expectation: "default = 0 means prev is never NULL"
SELECT id, amount,
       LAG(amount, 1, 0) OVER (ORDER BY id) AS prev
FROM payments;
-- WRONG RESULT: prev is 0 only at the FIRST row. If an interior row's previous
-- amount is genuinely NULL (pending payment), prev is NULL, not 0.
```

**Why:** the `default` only substitutes when *no neighbour row exists* (edge), never for a neighbour whose value is `NULL` (Section 6.10).

```sql
-- RIGHT: handle interior NULLs explicitly with COALESCE on the result
SELECT id, amount,
       COALESCE(LAG(amount, 1, 0) OVER (ORDER BY id), 0) AS prev  -- 0 at edge AND for null neighbours
FROM payments;
-- Or, if edge vs null-neighbour must be distinguished, keep them separate deliberately.
```

### Wrong 5: Missing ORDER BY tiebreaker — non-deterministic neighbour

```sql
-- WRONG: two payments with identical created_at; LAG picks one arbitrarily,
-- and the choice can change between runs/plans.
SELECT id, created_at, amount,
       LAG(amount) OVER (PARTITION BY customer_id ORDER BY created_at) AS prev
FROM payments;
-- WRONG RESULT: "flaky" prev values for tied timestamps — reports differ run to run.
```

```sql
-- RIGHT: add a unique tiebreaker to make the order total and deterministic
SELECT id, created_at, amount,
       LAG(amount) OVER (PARTITION BY customer_id ORDER BY created_at, id) AS prev
FROM payments;
```

---

## 10. Performance Profile

### The cost model in one sentence

**`LAG`/`LEAD` cost = the cost of getting rows into partition+order sequence, plus a near-free O(1)-per-row peek.** Optimize the sort/scan; the window itself is cheap.

### Scaling behaviour

| Rows | No index (sort required) | Composite index (no sort) |
|------|--------------------------|---------------------------|
| 100K | ~50–120ms; sort fits in `work_mem` | ~30–70ms; streamed |
| 1M | ~0.6–1.2s; sort likely spills to disk | ~0.4–0.6s; streamed |
| 10M | ~8–20s; heavy disk-spill sort, temp files | ~4–8s; bounded by index scan + I/O |
| 100M | minutes; massive external sort, huge temp usage | tens of seconds; streamed, I/O-bound |

The right column is not just faster — it has **predictable, linear-ish** scaling because it avoids the O(N log N) sort *and* the disk spill. The left column degrades super-linearly once the sort exceeds `work_mem`.

### Memory (work_mem) interaction

- The `Sort` beneath `WindowAgg` uses `work_mem`. When the input exceeds it, PostgreSQL switches to `external merge` sort, writing temp files (visible as `Disk: NkB` and `temp read/written` in `EXPLAIN (BUFFERS)`).
- The `WindowAgg` tuplestore (the per-partition buffer `LAG`/`LEAD` index into) is also bounded by `work_mem`; large partitions spill too.
- Raising `work_mem` (e.g., `SET work_mem = '256MB'` for the session, or per-query) can convert a disk-spilling sort into an in-memory one — often a 2–5× speedup. Do this at the session/transaction level for a heavy analytical query, not globally (it's per-sort, per-connection — a global bump can exhaust RAM under concurrency).

### The decisive optimization: composite index matching the window

```sql
-- For:  OVER (PARTITION BY customer_id ORDER BY created_at)
CREATE INDEX idx_orders_cust_created ON orders (customer_id, created_at);
```

Column order in the index must be **partition columns first, then order columns** — exactly the window's `(PARTITION BY..., ORDER BY...)` sequence. This lets an `Index Scan` (or `Index Only Scan` if all needed columns are covered) deliver rows pre-sorted, eliminating the `Sort` node entirely (Section 7, Case B). This is the single highest-leverage change for `LAG`/`LEAD` at scale.

For a covering index that avoids heap fetches:

```sql
CREATE INDEX idx_orders_cust_created_cover
  ON orders (customer_id, created_at) INCLUDE (total_amount);
-- Enables Index Only Scan when the query needs only these columns
```

### CPU profile

- The peek is a pointer index into the tuplestore — negligible CPU.
- Sorting dominates CPU when required (comparison-heavy).
- Sharing a `WINDOW w AS (...)` across multiple `LAG`/`LEAD`/other window functions means **one sort** serves all of them — always prefer a named window over repeating the same `OVER (...)`.

### When LAG beats the alternatives

Compared to a **correlated subquery** ("select the previous row for each row"), `LAG` is O(N log N) vs O(N²). At 1M rows, the correlated subquery can be 100–1000× slower or effectively never finish. Compared to a **self-join on `row_number()`**, `LAG` is simpler and typically faster (one pass vs join + sort). `LAG` is the correct tool; the alternatives exist mostly as cautionary tales.

---

## 11. Node.js Integration

### 11.1 Basic LAG query with pg

```javascript
import pg from 'pg';
const { Pool } = pg;
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

// Previous order amount per customer, with parameterized customer filter.
async function getOrdersWithPrevAmount(customerId) {
  const { rows } = await pool.query(
    `SELECT
       o.id            AS order_id,
       o.created_at,
       o.total_amount,
       LAG(o.total_amount) OVER (
         PARTITION BY o.customer_id
         ORDER BY o.created_at, o.id
       ) AS prev_amount
     FROM orders o
     WHERE o.customer_id = $1
     ORDER BY o.created_at`,
    [customerId]                 // $1 param binding — never string-interpolate
  );
  return rows;   // prev_amount is null for the customer's first order
}
```

### 11.2 Day-over-day change (filter on window result via subquery)

```javascript
// Return only days where revenue changed by more than a threshold vs the prior day.
async function getSignificantRevenueSwings(thresholdPct) {
  const { rows } = await pool.query(
    `SELECT * FROM (
       SELECT
         day,
         revenue,
         LAG(revenue) OVER (ORDER BY day) AS prev_revenue,
         ROUND(
           100.0 * (revenue - LAG(revenue) OVER (ORDER BY day))
           / NULLIF(LAG(revenue) OVER (ORDER BY day), 0), 2
         ) AS pct_change
       FROM (
         SELECT DATE_TRUNC('day', created_at)::date AS day, SUM(total_amount) AS revenue
         FROM orders WHERE status = 'completed'
         GROUP BY 1
       ) daily
     ) t
     WHERE ABS(t.pct_change) >= $1        -- window result filtered in OUTER query
     ORDER BY t.day`,
    [thresholdPct]
  );
  return rows;
}
```

### 11.3 Gap / velocity detection (fraud check)

```javascript
// Flag logins occurring within `windowSeconds` of the previous login for the same user.
async function detectRapidLogins(windowSeconds) {
  const { rows } = await pool.query(
    `SELECT user_id, created_at, gap_seconds
     FROM (
       SELECT
         user_id,
         created_at,
         EXTRACT(EPOCH FROM (
           created_at - LAG(created_at) OVER (PARTITION BY user_id ORDER BY created_at)
         )) AS gap_seconds
       FROM sessions
       WHERE event_type = 'login'
     ) t
     WHERE gap_seconds IS NOT NULL AND gap_seconds < $1
     ORDER BY user_id, created_at`,
    [windowSeconds]
  );
  return rows;   // each row = a login suspiciously close to the previous one
}
```

### 11.4 Note on numeric types

`SUM`, subtraction of `numeric`, and `EXTRACT(EPOCH ...)` return values the `pg` driver maps to **JavaScript strings** (for `numeric`/`bigint`) to avoid float precision loss. Parse deliberately:

```javascript
const prev = row.prev_amount === null ? null : Number(row.prev_amount);
// For money, prefer a decimal library over Number() to avoid float rounding.
```

Also handle `null` explicitly — `prev_amount`/`gap_seconds` are `null` at every partition edge, and `Number(null)` is `0`, which would silently corrupt calculations. Guard the null before arithmetic.

---

## 12. ORM Comparison

Window functions like `LAG`/`LEAD` are the frontier where ORMs' declarative models break down. None express them ergonomically through their standard query builders; most require raw SQL or a raw-fragment escape hatch.

### Prisma

**Can Prisma do LAG/LEAD?** — Not through its type-safe query API. Prisma has no window-function abstraction.

```typescript
import { PrismaClient } from '@prisma/client';
const prisma = new PrismaClient();

// Must use $queryRaw. Note Prisma.sql for safe interpolation.
const rows = await prisma.$queryRaw<
  { order_id: number; total_amount: string; prev_amount: string | null }[]
>`
  SELECT
    o.id AS order_id,
    o.total_amount,
    LAG(o.total_amount) OVER (
      PARTITION BY o.customer_id ORDER BY o.created_at, o.id
    ) AS prev_amount
  FROM orders o
  WHERE o.customer_id = ${customerId}
  ORDER BY o.created_at
`;
```

**Where it breaks:** No `include`/`select` path produces a window function; results are untyped unless you supply the generic. `numeric` comes back as `string`.
**Verdict:** Raw SQL only. Use `$queryRaw` with a typed generic and `${}` parameters.

### Drizzle ORM

**Can Drizzle do LAG/LEAD?** — Not as first-class builders, but its `sql` template makes raw window expressions clean and type-annotatable.

```typescript
import { sql } from 'drizzle-orm';
import { orders } from './schema';

const rows = await db
  .select({
    orderId: orders.id,
    totalAmount: orders.totalAmount,
    prevAmount: sql<string | null>`
      LAG(${orders.totalAmount}) OVER (
        PARTITION BY ${orders.customerId} ORDER BY ${orders.createdAt}, ${orders.id}
      )`.as('prev_amount'),
  })
  .from(orders)
  .orderBy(orders.createdAt);
```

**Where it breaks:** The window body is a raw `sql` string — no compile-time validation of the window clause; you type the result yourself.
**Verdict:** Best-in-class among ORMs. The `sql` tag composes column references safely and keeps it close to real SQL.

### Sequelize

**Can Sequelize do LAG/LEAD?** — Only via `sequelize.literal()` or a full raw query.

```javascript
const { QueryTypes } = require('sequelize');

const rows = await sequelize.query(
  `SELECT o.id AS order_id, o.total_amount,
          LAG(o.total_amount) OVER (
            PARTITION BY o.customer_id ORDER BY o.created_at, o.id
          ) AS prev_amount
   FROM orders o
   WHERE o.customer_id = :customerId
   ORDER BY o.created_at`,
  { replacements: { customerId }, type: QueryTypes.SELECT }
);
```

**Where it breaks:** `attributes: [[sequelize.literal('LAG(...) OVER (...)'), 'prev']]` technically works but is fragile, unreadable, and offers no safety.
**Verdict:** Use `sequelize.query()` with `replacements`. Do not fight the model.

### TypeORM

**Can TypeORM do LAG/LEAD?** — Via `addSelect` with a raw string, retrieved through `getRawMany()`.

```typescript
const rows = await dataSource
  .getRepository(Order)
  .createQueryBuilder('o')
  .select('o.id', 'order_id')
  .addSelect('o.total_amount', 'total_amount')
  .addSelect(
    `LAG(o.total_amount) OVER (PARTITION BY o.customer_id ORDER BY o.created_at, o.id)`,
    'prev_amount'
  )
  .where('o.customer_id = :customerId', { customerId })
  .orderBy('o.created_at', 'ASC')
  .getRawMany();   // window columns require getRawMany, NOT getMany
```

**Where it breaks:** `getMany()` maps to entities and *drops* the raw window column — you must use `getRawMany()`. The window string is unchecked.
**Verdict:** Workable with `addSelect` + `getRawMany()`. Remember the raw-vs-entity distinction.

### Knex.js

**Can Knex do LAG/LEAD?** — Yes, with `knex.raw()` in `.select()`. Knex is the most SQL-transparent here.

```javascript
const rows = await knex('orders as o')
  .select(
    'o.id as order_id',
    'o.total_amount',
    knex.raw(
      `LAG(o.total_amount) OVER (
         PARTITION BY o.customer_id ORDER BY o.created_at, o.id
       ) AS prev_amount`
    )
  )
  .where('o.customer_id', customerId)
  .orderBy('o.created_at');
```

**Where it breaks:** The window expression is a raw string — no builder-level validation. Otherwise seamless.
**Verdict:** Clean escape hatch. `knex.raw()` inside a normal query keeps the rest of the query structured.

### ORM Summary Table

| ORM | LAG/LEAD support | Mechanism | Result typing | Verdict |
|-----|------------------|-----------|---------------|---------|
| Prisma | None native | `$queryRaw` | Manual generic | Raw SQL only |
| Drizzle | Via `sql` tag | `sql\`LAG(...) OVER (...)\`` | You annotate | Best ORM support |
| Sequelize | None native | `sequelize.query()` / `literal()` | `QueryTypes.SELECT` | Use raw query |
| TypeORM | Via `addSelect` | `addSelect` + `getRawMany()` | Raw rows | Workable, use getRawMany |
| Knex | Via `knex.raw()` | `.select(knex.raw(...))` | Raw rows | Cleanest escape hatch |

**Overall:** `LAG`/`LEAD` are a raw-SQL topic. Drizzle's `sql` tag and Knex's `knex.raw()` integrate them most gracefully; Prisma and Sequelize require full raw queries. In all cases, the SQL is what you must understand — the ORM is just a delivery vehicle.

---

## 13. Practice Exercises

### Exercise 1 — Basic

Given:
- `payments(id, customer_id, amount, paid_at, status)`

For each *successful* payment (`status = 'success'`), return `id`, `customer_id`, `paid_at`, `amount`, and `prev_amount` — the amount of that customer's **previous** successful payment in chronological order. The first successful payment per customer should show `prev_amount = NULL`. Order the output by `customer_id`, then `paid_at`.

Then: modify the query so the first payment shows `prev_amount = 0` instead of `NULL`, using the appropriate function argument (not `COALESCE`).

```sql
-- Write your query here
```

---

### Exercise 2 — Intermediate (combines GROUP BY + window-over-aggregate)

Given:
- `orders(id, customer_id, total_amount, status, created_at)`

Produce a monthly report over completed orders with:
- `month` (truncated to month)
- `monthly_revenue` — sum of `total_amount`
- `prev_month_revenue` — previous month's revenue (via `LAG` over the aggregate)
- `mom_growth_pct` — month-over-month growth %, rounded to 1 decimal, guarding division by zero
- `next_month_revenue` — the following month's revenue (via `LEAD`)

Order by `month`. The first month should have `NULL` for `prev_month_revenue` and the last for `next_month_revenue`.

```sql
-- Write your query here
```

---

### Exercise 3 — Hard (production simulation; the naive answer is wrong)

Given:
- `sessions(id, user_id, created_at, event_type)` — ~30M rows

Sessionize the events: a user's events belong to the same session unless the gap since their previous event exceeds **20 minutes**, which starts a new session. Return `user_id`, `created_at`, `event_type`, and a `session_number` (1, 2, 3, … per user).

Then extend it to also produce, per user, `session_count` (total number of sessions) and, for each session, its `session_duration` (last event time minus first event time within the session).

Note why a single `LAG` is *not enough* — you need the gaps-and-islands pattern (LAG → new-session flag → running SUM). State what index you would create to avoid a sort at 30M rows.

```sql
-- Write your query here
```

---

### Exercise 4 — Interview simulation

Given:
- `audit_logs(id, entity_type, entity_id, field_name, old_value, new_value, changed_at)`

An order's `status` transitions are recorded here (`entity_type = 'order'`, `field_name = 'status'`). Write a query that, for each order, returns every **actual status transition** as: `entity_id` (the order id), `changed_at`, `from_status`, `to_status`, and `time_in_prev_status` (how long the order sat in the previous status before this change).

Requirements: include the *initial* status assignment (where there is no prior status) as a transition from `NULL`. Ensure a row where `new_value` equals the row-before's `new_value` (a no-op re-write) is **excluded**. Handle timestamp ties deterministically.

```sql
-- Write your query here
```

---

## 14. Interview Questions

### Junior Level

**Q: What do `LAG` and `LEAD` do, and what's the difference?**

A: Both are window functions that let each row read a value from another row in the same partition, ordered by the window's `ORDER BY`. `LAG` reads a row *before* the current one (default one position back); `LEAD` reads a row *after* (default one position forward). They don't collapse rows like aggregates do — they add a column showing the neighbour's value. At the start of a partition, `LAG` returns `NULL` (no prior row); at the end, `LEAD` returns `NULL` (no following row), unless you pass a default as the third argument.

---

**Q: Why can't I use a `LAG` result in a `WHERE` clause?**

A: Because of SQL's logical execution order. `WHERE` runs *before* the `SELECT` phase, and window functions are evaluated in the `SELECT` phase. When `WHERE` executes, the `LAG` column doesn't exist yet, so PostgreSQL raises "window functions are not allowed in WHERE." The fix is to compute the `LAG` in a subquery or CTE and filter on it in the outer query, where it's a normal column.

---

**Q: What are the three arguments to `LAG`?**

A: `LAG(value_expr, offset, default)`. `value_expr` is the expression whose value is read from the neighbour row (required). `offset` is how many rows back to look (optional, default 1). `default` is the value returned when the offset row is outside the partition — i.e., at the edge — instead of `NULL` (optional, default `NULL`). A key subtlety: `default` only applies at the partition edge; if the neighbour row exists but its value is genuinely `NULL`, `LAG` returns that `NULL`, not the default.

---

### Principal Level

**Q: You have a 50M-row events table and a query using `LAG(created_at) OVER (PARTITION BY user_id ORDER BY created_at)`. It takes 40 seconds and EXPLAIN shows a Sort with `external merge Disk: 900MB`. Walk me through the fix.**

A: The cost is entirely in sorting 50M rows to produce the `(user_id, created_at)` order the `WindowAgg` needs, and that sort is spilling ~900MB to disk because it exceeds `work_mem`. The `LAG` itself is O(1) per row — negligible. Two levers, in order of impact: (1) Create a composite B-tree index `ON events(user_id, created_at)` — matching partition columns first, then order columns. The planner can then use an Index Scan that delivers rows already in that sequence, eliminating the Sort node entirely; the query streams and becomes I/O-bound rather than sort-bound. If the query only needs a few columns, an `INCLUDE` covering index enables an Index Only Scan, avoiding heap fetches too. (2) If an index isn't feasible, raise `work_mem` for that session/query so the sort stays in memory — turning `external merge` into an in-memory `quicksort`, often 2–5× faster. The index is the durable fix; `work_mem` is the immediate mitigation. After adding the index, re-check EXPLAIN to confirm the Sort node is gone.

---

**Q: A developer computes "did the value change from the previous row" with `WHERE prev_status <> status` and complains that the first record of each entity is missing. Diagnose and fix.**

A: The first row of each partition has `prev_status = NULL` (no prior row). `NULL <> 'pending'` evaluates to `UNKNOWN` under three-valued logic, and `WHERE` keeps only rows where the predicate is `TRUE`, so every first row is silently dropped. If the intent is to treat the initial assignment as a change (from nothing to something), the fix is `WHERE prev_status IS DISTINCT FROM status` — `IS DISTINCT FROM` treats `NULL` as a comparable value, so `NULL IS DISTINCT FROM 'pending'` is `TRUE` and the first row is kept. This also correctly handles interior NULLs. If the intent is genuinely to see only interior transitions and skip the first appearance, `<>` is acceptable — but it should be a deliberate choice, not an accident of NULL semantics. This same distinction matters whenever you compare a `LAG` result, because `LAG` produces `NULL` at every partition edge by design.

---

**Q: Explain how you'd sessionize a clickstream — group each user's events into sessions where a gap over 30 minutes starts a new session — using window functions. Why isn't a single `LAG` enough?**

A: A single `LAG` gives you the gap between consecutive events (`created_at - LAG(created_at) OVER (PARTITION BY user_id ORDER BY created_at)`), but a gap is not a session number — you need to *number the sessions*, which requires accumulating boundaries. This is the gaps-and-islands pattern in three layers: (1) `LAG` to get each event's previous timestamp; (2) a boolean/0-1 flag `is_new_session` that is 1 when the previous timestamp is NULL (first event) or the gap exceeds 30 minutes, else 0; (3) a running `SUM(is_new_session) OVER (PARTITION BY user_id ORDER BY created_at ROWS UNBOUNDED PRECEDING)` — this cumulative count *is* the session number, incrementing exactly at each boundary. The elegance is that the running sum turns discrete "new session starts here" flags into a monotonic session id. For performance at scale, index `(user_id, created_at)` so both the `LAG` and the running `SUM` stream from the same sorted order with no Sort node. From there you can group by `(user_id, session_number)` to get session durations, event counts, and session totals.

---

**Q: When would you reach for a self-join instead of `LAG`, and why is `LAG` usually better?**

A: Almost never for "previous row" access — `LAG` is strictly better there. A self-join to get "the previous row" requires either a correlated subquery (`WHERE t2.created_at < t1.created_at ORDER BY ... LIMIT 1`), which is O(N²) and catastrophic at scale, or a join on precomputed `row_number()`, which needs two passes and a join. `LAG` does it in a single sort-then-stream pass, O(N log N), and expresses intent clearly. The rare case for a self-join is when "previous" is defined by a *non-positional* condition that a window `ORDER BY` can't capture — e.g., "the previous row *matching a different predicate*" that isn't simply the adjacent row in a sortable order. Even then, a window with a carefully constructed partition/order or a `FILTER`ed approach usually wins. In production, `LAG` is the default and self-joins for adjacency are a code smell.

---

## 15. Mental Model Checkpoint

1. A partition has 5 rows ordered by time. For row 1, what does `LAG(x)` return, and what does `LEAD(x)` return? For row 5? Why?

2. You write `LAG(amount) OVER (ORDER BY created_at)` but forget `PARTITION BY customer_id`. Describe the *specific* wrong value that appears at each customer's first order, and why the query produces no error.

3. `LAG(x, 1, 0)` is applied to a column `x` that contains NULLs. A given interior row's previous row has `x = NULL`. Does `LAG` return `0` or `NULL`? Explain the rule that decides this.

4. You need "the previous row where `status = 'success'`," skipping failed rows in between. Can plain `LAG(amount)` do this? If not, what has to change about the input to the window?

5. Two rows share the identical `created_at`. Without a tiebreaker, is `LAG`'s result deterministic across query runs? What single change makes it deterministic?

6. You want to filter to only rows where `amount > previous amount`. Why does putting `amount > LAG(amount) OVER (...)` in `WHERE` fail, and what is the structural fix?

7. In the sessionization pattern, after computing `is_new_session` flags, why does a *running* `SUM` (not a plain `SUM`) produce the session number? What would a plain `SUM` give instead?

8. `EXPLAIN` shows `WindowAgg` directly above an `Index Scan` with no `Sort` node. What does the absence of the `Sort` tell you about the index, and why is that the performance ideal for `LAG`/`LEAD`?

---

## 16. Quick Reference Card

```sql
-- SYNTAX
LAG(value_expr [, offset [, default]])  OVER (PARTITION BY ... ORDER BY ...)  -- look BACK
LEAD(value_expr [, offset [, default]]) OVER (PARTITION BY ... ORDER BY ...)  -- look FORWARD
--   offset  : rows away (default 1). Must be >= 0. Negative = error (use the other function).
--   default : value at partition EDGE only (default NULL). NOT used for interior NULLs.

-- EDGES: LAG NULL at partition START ; LEAD NULL at partition END (unless default given)

-- DAY-OVER-DAY CHANGE (with divide-by-zero guard)
revenue - LAG(revenue) OVER (ORDER BY day)                                   AS abs_change
ROUND(100.0*(revenue-LAG(revenue) OVER w)/NULLIF(LAG(revenue) OVER w,0),2)   AS pct_change

-- GAP BETWEEN EVENTS
created_at - LAG(created_at) OVER (PARTITION BY user_id ORDER BY created_at)  AS gap

-- DETECT VALUE CHANGE (keeps first row; use IS DISTINCT FROM, not <>)
WHERE LAG(status) OVER (...) IS DISTINCT FROM status

-- SESSIONIZATION (gaps-and-islands): LAG -> flag -> running SUM
SUM(CASE WHEN prev_at IS NULL OR created_at-prev_at > INTERVAL '30 min'
         THEN 1 ELSE 0 END)
    OVER (PARTITION BY user_id ORDER BY created_at ROWS UNBOUNDED PRECEDING)  AS session_no

-- FILTER ON WINDOW RESULT: must be in an OUTER query (not same-level WHERE)
SELECT * FROM (SELECT ..., LAG(x) OVER (...) AS prev FROM t) s WHERE s.prev < s.x;

-- RULES OF THUMB
--  * ALWAYS give ORDER BY inside OVER; add a unique tiebreaker (…, id) for tied keys.
--  * ALWAYS PARTITION BY the entity, or the peek bleeds across entities (silent bug).
--  * Frame clause (ROWS BETWEEN …) is IGNORED by LAG/LEAD — don't add it.
--  * default fixes the EDGE only; use COALESCE for interior NULLs.
--  * LAG(x) OVER (ORDER BY t DESC)  ==  LEAD(x) OVER (ORDER BY t ASC).

-- PERFORMANCE
--  * Cost = SORT to get partition+order, then O(1)/row peek. Optimize the sort.
--  * Index (PARTITION cols, ORDER cols) removes the Sort node -> streams. Biggest win.
--      CREATE INDEX ix ON orders(customer_id, created_at);   -- + INCLUDE for Index Only Scan
--  * Sort spilling: EXPLAIN shows "external merge Disk: NkB" -> raise work_mem or add index.
--  * Share one WINDOW w AS (...) so multiple functions sort ONCE.

-- INTERVIEW ONE-LINERS
--  * "LAG/LEAD peek at a neighbour by positional offset; O(1) per row, frame-agnostic."
--  * "The sort is the cost; a matching composite index eliminates it."
--  * "NULL at edges; use IS DISTINCT FROM to keep first-row transitions."
--  * "Window functions run in SELECT — after WHERE — so filter their result in an outer query."
--  * "Sessionization = LAG gap detection + running SUM of new-session flags."
```

---

## Connected Topics

- **Topic 33 — Window Functions Fundamentals & OVER()**: The `PARTITION BY`/`ORDER BY`/`WINDOW` machinery and named windows that `LAG`/`LEAD` build on.
- **Topic 34 — ROW_NUMBER, RANK, DENSE_RANK**: Ranking functions that share the same `WindowAgg` + sort mechanics; often combined with `LAG` for adjacency-on-rank.
- **Topic 35 — NTILE, PERCENT_RANK, CUME_DIST**: The previous topic — distribution/bucketing window functions; same sort cost model.
- **Topic 37 — FIRST_VALUE, LAST_VALUE, NTH_VALUE**: The next topic — the *other* value-access window functions, which (unlike `LAG`/`LEAD`) are acutely frame-sensitive (`LAST_VALUE` trap).
- **Topic 07 — NULL in Depth**: Three-valued logic behind `IS DISTINCT FROM` vs `<>` and the two-source NULL behaviour at partition edges.
- **Topic 20 — GROUP BY Fundamentals**: Window-over-aggregate composition (`LAG(SUM(...))`) depends on understanding what `GROUP BY` produces before the window runs.
- **Topic 17 — JOIN Performance Deep Dive**: Why `LAG` beats correlated-subquery and self-join alternatives for adjacency access.
- **Gaps-and-Islands (recurring pattern)**: `LAG` boundary detection + running `SUM` — reused for sessionization, deduplication, and run-length encoding across the curriculum.
