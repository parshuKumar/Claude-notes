# Topic 39 — Frame Clauses in Depth
### SQL Mastery Curriculum — Phase 6: Window Functions — Complete Mastery

---

## 1. ELI5 — The Analogy That Makes It Click

Imagine you're a runner in a marathon, and at every step you glance at a small handheld whiteboard to compute your "running average pace." The question that decides what number you write down is: **which other runners do I look at right now?**

- Maybe you look at **everyone who started before you and where you are now** — from the very first runner up to your own current position. That's a *cumulative* window.
- Maybe you only look at **the three runners immediately ahead and the three immediately behind you** — a small sliding band that travels with you. That's a *sliding* window.
- Maybe you look at **everyone in the whole race**, front to back, no matter where you personally are. That's the *whole partition*.

A **frame clause** is the rule that tells SQL, *for each row being processed*, exactly which surrounding rows count as "in view." The window function (SUM, AVG, FIRST_VALUE, …) then does its math over only those visible rows.

Here's the twist that catches everyone: the frame is **recomputed for every single row**. It is not one fixed set. As the "current row" marches down the partition, the frame slides, grows, or stays put depending on the rule you wrote. A running total grows one row at a time; a 7-day moving average is a band that slides; a "percent of partition total" needs the whole partition visible at every row.

And the part that *really* surprises people: **if you write an `ORDER BY` inside the window but no explicit frame, SQL silently gives you a frame anyway** — and it's the growing "from the start up to here" kind, with a subtle tie-handling rule (`RANGE`) that can make your running total jump by more than one row at a time. Most "my running total is wrong" bugs are exactly this default biting.

The frame clause is how you take back explicit control over "which rows do I look at right now."

---

## 2. Connection to SQL Internals

A window function in PostgreSQL is executed by a **WindowAgg** plan node. Understanding the frame clause means understanding what WindowAgg physically does with the rows handed to it.

**The input must be sorted.** Before WindowAgg runs, its input is already ordered by `PARTITION BY` keys followed by `ORDER BY` keys (either via a `Sort` node or an ordered index scan). WindowAgg then walks this single sorted stream top to bottom. It does **not** re-sort per row — the sort happens once.

**The frame is a moving pair of pointers into that sorted stream.** For each current row, WindowAgg maintains two cursors:
- a **frame start** pointer
- a **frame end** pointer

The window function's aggregate is computed over the rows between those two pointers. The whole art of frame execution is advancing those pointers cheaply as the current row moves.

**Three physical strategies the engine uses depending on frame type:**

1. **Incremental aggregation (the fast path).** For a frame like `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` (a running total), the frame only *grows* by one row each step. PostgreSQL keeps a running aggregate state and just *adds* the new row's contribution — O(1) per row, O(N) total. This is why running totals are cheap.

2. **Inverse-transition (the removable-aggregate path).** For a sliding frame like `ROWS BETWEEN 6 PRECEDING AND CURRENT ROW`, rows both *enter* (at the front) and *leave* (at the back) each step. For aggregates that support an **inverse transition function** (SUM, COUNT, AVG over non-float types), PostgreSQL *subtracts* the departing row and *adds* the arriving row — still O(1) per row. `SUM(numeric)` has an inverse; `MAX`/`MIN` do **not** (you can't "un-max"), so those force a rescan.

3. **Full frame rescan (the slow path).** When no incremental trick applies (e.g., `MAX()` over a sliding frame, or a `RANGE`/`GROUPS` frame where the boundary can jump), WindowAgg re-walks the frame rows for each current row. Worst case this is O(N²) within a partition.

**Memory: the tuplestore.** WindowAgg buffers rows of the current partition in a **tuplestore** (in `work_mem`, spilling to a temp file on disk when it overflows). It must buffer because a frame can reference rows *behind* the current row (e.g., `UNBOUNDED FOLLOWING` needs to peek ahead; `LAG`-style access needs to look back). The frame type dictates how much must be buffered at once.

**MVCC and visibility** still apply underneath: the rows fed into WindowAgg are the ones visible to your transaction snapshot, already filtered by `WHERE` and grouped by `GROUP BY`. The frame clause operates purely on that post-filter, post-sort logical stream — it never touches the heap or B-tree directly.

So: **frame clause = the rule governing two pointers into a sorted, buffered tuplestore, plus whether the aggregate can be maintained incrementally or must be rescanned.** Everything about frame performance flows from those two facts.

---

## 3. Logical Execution Order Context

Window functions — and therefore frame clauses — run at a very specific point in the logical pipeline:

```
FROM / JOIN          ← tables assembled
WHERE                ← row filtering (BEFORE windows exist)
GROUP BY             ← grouping
HAVING               ← group filtering
SELECT (windows)     ← ★ WINDOW FUNCTIONS + FRAMES EVALUATED HERE ★
DISTINCT
ORDER BY             ← final output ordering (separate from window ORDER BY)
LIMIT / OFFSET
```

Critical consequences of this position:

**1. Window functions see rows *after* WHERE, HAVING, and GROUP BY.** You cannot filter on a window function's result in `WHERE` — the window hasn't been computed yet. This fails:

```sql
-- ERROR: window functions are not allowed in WHERE
SELECT id, SUM(amount) OVER (ORDER BY id) AS running
FROM payments
WHERE SUM(amount) OVER (ORDER BY id) > 1000;  -- illegal
```

The fix is to wrap the window in a subquery/CTE and filter in an outer `WHERE` (covered in Topic 38 and revisited in section 6).

**2. The window's own `ORDER BY` is independent of the final `ORDER BY`.** The `ORDER BY` *inside* `OVER (...)` only sequences rows for the frame — it decides "which row is PRECEDING vs FOLLOWING." The `ORDER BY` at the end of the statement decides how the final result is displayed. They can differ, and confusing them produces subtly wrong frames.

**3. Frames operate on the partition, which is defined at window-evaluation time.** `PARTITION BY` slices the post-HAVING rowset; the frame then applies *within* each partition and never crosses a partition boundary. `UNBOUNDED PRECEDING` means "start of *this* partition," not start of the table.

**4. All window functions in one SELECT run over the same input snapshot.** Multiple `OVER` clauses with different frames all read the same post-HAVING rowset; the planner may compute several in one WindowAgg if they share a window definition, or stack WindowAgg nodes if they differ.

Because the frame is evaluated at the SELECT stage, it is one of the *last* logical operations — which is exactly why it can reference the fully-assembled, filtered, sorted rowset, and why it cannot be referenced by earlier clauses.

---

## 4. What Is a Frame Clause?

A **frame clause** (the SQL standard calls it the *window frame*) defines, for each row processed by a window function, the subset of partition rows — the *frame* — over which the function computes. It is the third component of a window definition, after `PARTITION BY` and `ORDER BY`.

```sql
SELECT
  col,
  SUM(amount) OVER (
    PARTITION BY customer_id      -- ① slice rows into partitions
    ORDER BY created_at           -- ② order rows within each partition
    ROWS BETWEEN 6 PRECEDING      -- ③ ┐
              AND CURRENT ROW     --   ┘ the FRAME CLAUSE
  ) AS rolling_7
FROM payments;
```

Annotated syntax breakdown — every keyword explained:

```
<frame_clause> ::=
  { ROWS | RANGE | GROUPS }              │ ── FRAME MODE: how boundaries are measured
                                         │      ROWS   → count physical rows
                                         │      RANGE  → value-range around ORDER BY key
                                         │      GROUPS → count peer-groups (tie clusters)
                                         │
  BETWEEN <frame_start>                  │ ── lower boundary of the frame
      AND <frame_end>                    │ ── upper boundary of the frame
                                         │      (single-bound form = "BETWEEN x AND CURRENT ROW")
                                         │
  [ EXCLUDE <exclusion> ]                │ ── optionally remove rows from the computed frame

<frame_start> / <frame_end> ::=
    UNBOUNDED PRECEDING                  │ ── from the very first row of the partition
  | N PRECEDING                          │ ── N units before the current row
  |                                      │      (ROWS: N rows; RANGE: value − N; GROUPS: N groups)
  | CURRENT ROW                          │ ── the current row itself
  |                                      │      (RANGE/GROUPS: the current row's whole PEER GROUP)
  | N FOLLOWING                          │ ── N units after the current row
  | UNBOUNDED FOLLOWING                  │ ── through the very last row of the partition

<exclusion> ::=
    EXCLUDE CURRENT ROW                  │ ── omit the current row, keep its peers
  | EXCLUDE GROUP                        │ ── omit the current row AND all its peers
  | EXCLUDE TIES                         │ ── omit peers of the current row, KEEP current row
  | EXCLUDE NO OTHERS                    │ ── the default: exclude nothing (explicit no-op)
```

Two hard rules on boundary ordering:
- The frame **start** must not be after the frame **end**. `ROWS BETWEEN 2 FOLLOWING AND 2 PRECEDING` is an error.
- A start of `UNBOUNDED FOLLOWING` or an end of `UNBOUNDED PRECEDING` is illegal (a frame can't start at the end or end at the start).

**The single most important sentence in this topic:** if you write `ORDER BY` in the window but **omit** the frame clause entirely, PostgreSQL applies the default `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`. If you write **no** `ORDER BY` at all, the default is `RANGE BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING` (the whole partition). Section 6 dissects why the first default surprises people.

---

## 5. Why Frame Clause Mastery Matters in Production

1. **Silent wrong numbers, not errors.** A misunderstood frame doesn't throw — it returns a plausible-looking result that is subtly wrong. A running total that jumps by 300 on rows sharing a timestamp (because the default `RANGE` lumps ties together), a "7-day average" that's actually a "7-*row* average" because rows aren't one-per-day — these ship to dashboards and finance reports undetected. Frame bugs are the most expensive kind: they corrupt data quietly.

2. **The `RANGE` vs `ROWS` default is a landmine.** The default frame is `RANGE`, and virtually everyone *means* `ROWS`. On unique ORDER BY keys they behave identically, so the bug lies dormant in dev and detonates in production the first time two rows tie on the ordering column. Knowing to write `ROWS` explicitly is a senior reflex.

3. **Performance cliffs.** `SUM() OVER (ORDER BY ...)` with an incremental frame is O(N). The same query with `MAX()` over a sliding frame, or a `RANGE` frame that can't maintain state, silently becomes O(N²) — fine on 10K rows, a timeout on 10M. Recognizing which frames the engine can maintain incrementally is the difference between a 40ms query and a 40s query.

4. **Rolling metrics are everywhere.** Moving averages, cumulative revenue, running balances, "last 30 days" trailing windows, percent-of-total, session gap detection — the entire vocabulary of analytics and time-series backends is frame clauses. Getting them right is table stakes for any reporting or metrics system.

5. **Correctness of financial and audit computations.** Running account balances (`audit_logs`, `payments`) must include exactly the right rows — off-by-one at the frame boundary is a real money bug. `CURRENT ROW` semantics under `RANGE` (include the whole peer group) versus `ROWS` (just this row) directly change a reported balance.

6. **Interview signal.** "What's the default frame and why does it surprise people?" and "When is `RANGE` different from `ROWS`?" are standard senior-level discriminators. A candidate who explains peer groups, the inverse-transition optimization, and `EXCLUDE TIES` is demonstrably above the pack.

---

## 6. Deep Technical Content

This is the core of the topic. We go frame mode by frame mode, boundary by boundary, then the defaults, then `EXCLUDE`, then the gotchas.

### 6.1 The Three Frame Modes: ROWS, RANGE, GROUPS

All three answer "which rows are in the frame?" but they *measure distance differently*.

Consider this partition, ordered by `day`, with **ties** on `day`:

```
row#  day  amount
 1     1     10
 2     2     20
 3     2     30     ← tie with row 2 on `day`
 4     2     40     ← tie with row 2,3 on `day`
 5     3     50
```

Rows 2, 3, 4 form a **peer group**: they are *equal* under the window's `ORDER BY day`. Peer groups are the pivot on which ROWS/RANGE/GROUPS differ.

**ROWS — count physical rows.** Boundaries are literal row offsets. `ROWS BETWEEN 1 PRECEDING AND CURRENT ROW` on row 3 = rows {2, 3}. Ties are irrelevant; ROWS doesn't know or care that rows 2–4 are peers. Each row is one unit.

**RANGE — measure by ORDER BY *value*.** Boundaries are ranges of the ordering key's value. `CURRENT ROW` under RANGE does **not** mean "this one row" — it means "all rows whose ordering value equals the current row's value," i.e. **the entire peer group**. So `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` on row 2 includes rows {1, 2, 3, 4} — because rows 3 and 4 have the *same* `day` value as row 2, they are "≤ current value" and swept in. On row 3 the frame is *also* {1,2,3,4}. On row 4, still {1,2,3,4}. This is precisely why a running SUM under RANGE reports the **same total** for all of rows 2, 3, 4 — the whole peer group's total — instead of incrementing one row at a time.

**GROUPS — count peer groups.** Boundaries are counts of *peer groups*, treating each tie-cluster as one unit. `GROUPS BETWEEN 1 PRECEDING AND CURRENT ROW` on row 3 = the current peer group (rows 2,3,4) plus one preceding group (row 1) = rows {1,2,3,4}. GROUPS is the middle ground: it respects ties like RANGE (whole groups move together) but counts *groups* as integer offsets like ROWS.

Side-by-side, frame of each row under `... BETWEEN 1 PRECEDING AND CURRENT ROW`:

| current row | ROWS | RANGE | GROUPS |
|---|---|---|---|
| row 1 (day 1) | {1} | {1} | {1} |
| row 2 (day 2) | {1,2} | {1,2,3,4} | {1,2,3,4} |
| row 3 (day 2) | {2,3} | {1,2,3,4} | {1,2,3,4} |
| row 4 (day 2) | {3,4} | {1,2,3,4} | {1,2,3,4} |
| row 5 (day 3) | {4,5} | {2,3,4,5} | {2,3,4,5} |

Note RANGE and GROUPS agree here *only because* the day values happen to be consecutive integers (1,2,3). They diverge the moment values have gaps — see 6.7.

**Requirements and restrictions:**
- `RANGE` with a numeric offset (`RANGE BETWEEN 5 PRECEDING ...`) requires **exactly one** `ORDER BY` column, and that column must be a type that supports `+`/`-` with the offset type (numeric, date/time with an interval offset). `RANGE UNBOUNDED`/`CURRENT ROW` (no offset) works with any number of ORDER BY columns.
- `GROUPS` requires an `ORDER BY` in the window (it's meaningless without a defined peer grouping).
- `ROWS` has no such restriction and works with or without `ORDER BY`.

### 6.2 The Boundary Keywords in Physical Terms

Each boundary keyword, described as "where does the pointer land":

**`UNBOUNDED PRECEDING`** — the frame edge is pinned to the **first row of the partition**. Never moves. Used for cumulative/running computations.

**`N PRECEDING`** — N units *before* the current row, where "unit" depends on the mode:
- ROWS: N physical rows earlier.
- RANGE: rows whose ordering value ≥ (current value − N). With `ORDER BY t`, `RANGE BETWEEN INTERVAL '7 days' PRECEDING AND CURRENT ROW` = "all rows within the last 7 days by timestamp."
- GROUPS: N peer-groups earlier.

**`CURRENT ROW`** — mode-dependent, and this is the classic trap:
- ROWS: literally the current row, a single row.
- RANGE: the current row's **entire peer group** (all rows equal on the ORDER BY key).
- GROUPS: the current row's entire peer group (same as RANGE for CURRENT ROW).

**`N FOLLOWING`** — N units *after* the current row (mirror of `N PRECEDING`).

**`UNBOUNDED FOLLOWING`** — pinned to the **last row of the partition**. Never moves. Used for "from here to the end" and reverse-cumulative computations.

Legal combinations form the frame `BETWEEN start AND end`, with start ≤ end in stream position. Common shapes:

```sql
-- Running total (cumulative from partition start)
ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW

-- Full partition (every row sees every row) — needed for percent-of-total
ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING

-- Trailing 7-row moving window (this row + 6 before)
ROWS BETWEEN 6 PRECEDING AND CURRENT ROW

-- Centered 5-row window (2 before, self, 2 after)
ROWS BETWEEN 2 PRECEDING AND 2 FOLLOWING

-- Reverse cumulative (this row through the end)
ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING

-- Future-only window (next 3 rows, not including self)
ROWS BETWEEN 1 FOLLOWING AND 3 FOLLOWING

-- Trailing 30 days by actual timestamp (RANGE, not ROWS!)
RANGE BETWEEN INTERVAL '30 days' PRECEDING AND CURRENT ROW
```

### 6.3 The Default Frame — Why It Surprises People

**The rule:**
- Window has an `ORDER BY`, no explicit frame → default is **`RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`**.
- Window has **no** `ORDER BY` → default is **`RANGE BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`** (whole partition, every row sees all).

The second case is intuitive: no ordering means "the whole partition," so `AVG(x) OVER (PARTITION BY g)` gives each row the partition average. Fine.

The **first** case is the trap. Developers write:

```sql
SELECT day, amount,
       SUM(amount) OVER (ORDER BY day) AS running_total
FROM daily_sales;
```

…expecting a row-by-row running total. What they *actually* wrote — because the frame defaulted — is:

```sql
SUM(amount) OVER (ORDER BY day
                  RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
```

On data where `day` is **unique**, this is identical to a `ROWS` running total. Everyone's dev data has unique days. It works. It ships.

Then production has two sales rows with the *same* `day`:

```
day  amount   what user EXPECTED (ROWS)   what they GOT (default RANGE)
 1    100          100                          100
 2     50          150                          250   ← both day-2 rows
 2    100          250                          250   ← show 250, not 150 then 250
 3     30          280                          280
```

Under the default `RANGE`, `CURRENT ROW` = the whole peer group of `day = 2`, so both day-2 rows report the **cumulative total through the end of the day-2 group** (250). The running total appears to "skip" — it never shows 150. To a finance user this looks like the running balance jumped by 150 in one step and duplicated a value.

**Why the standard chose this default:** `RANGE ... CURRENT ROW` is *deterministic under ties*. A `ROWS` running total would give rows 2 and 3 different values (150, 250) even though they're indistinguishable under the `ORDER BY` — the result would depend on the physical/arbitrary order of tied rows, which is non-deterministic. The standard preferred a tie-stable answer. That reasoning is defensible for `AVG`/statistical framing but violates the intuitive "one row at a time" mental model of a running total.

**The fix is one word:** write `ROWS` explicitly whenever you want per-row cumulative behavior.

```sql
SUM(amount) OVER (ORDER BY day ROWS UNBOUNDED PRECEDING)   -- per-row running total
```

`ROWS UNBOUNDED PRECEDING` is shorthand for `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`.

### 6.4 Single-Boundary Shorthand

When you write a frame with only a *start*, the end defaults to `CURRENT ROW`:

```sql
ROWS UNBOUNDED PRECEDING      ≡  ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
ROWS 3 PRECEDING             ≡  ROWS BETWEEN 3 PRECEDING AND CURRENT ROW
RANGE CURRENT ROW            ≡  RANGE BETWEEN CURRENT ROW AND CURRENT ROW
```

You cannot write a "start only" that is `FOLLOWING`-based this way in a natural sense — `ROWS 3 FOLLOWING` would mean `BETWEEN 3 FOLLOWING AND CURRENT ROW`, which is start-after-end and therefore illegal. Use the explicit `BETWEEN` form for forward frames.

### 6.5 EXCLUDE — Removing Rows From a Computed Frame

After the frame is computed by mode + boundaries, `EXCLUDE` optionally removes specific rows. It's the least-known part of the frame clause and rarely needed — but occasionally exactly the tool.

Given the current row and its peer group, the four options:

- **`EXCLUDE NO OTHERS`** — the default; removes nothing. Explicit no-op.
- **`EXCLUDE CURRENT ROW`** — removes just the current row from the frame; keeps its peers.
- **`EXCLUDE GROUP`** — removes the current row *and* all of its peers.
- **`EXCLUDE TIES`** — removes the current row's peers but **keeps the current row itself**.

Worked example. Partition ordered by `day`, current row = row 3 (day 2), peer group = {2,3,4}, frame `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` = base frame {1,2,3,4}:

| EXCLUDE option | resulting frame |
|---|---|
| NO OTHERS (default) | {1,2,3,4} |
| CURRENT ROW | {1,2,4} |
| GROUP | {1} |
| TIES | {1,3} (drops peers 2,4; keeps self 3) |

**A real use case for `EXCLUDE CURRENT ROW`:** "average of all *other* rows in the window" — a leave-one-out average, useful for anomaly detection ("how does this row compare to its neighbors excluding itself").

```sql
-- Each product's price vs. the average price of the OTHER products in its category
SELECT p.id, p.name, p.price,
       AVG(p.price) OVER (
         PARTITION BY p.category_id
         ORDER BY p.id
         ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
         EXCLUDE CURRENT ROW
       ) AS avg_of_others
FROM products p;
```

**`EXCLUDE TIES`** is useful when you want the current row counted once but don't want its duplicates double-counting. In practice `EXCLUDE` is rare; know it exists and what each does, because interviewers probe whether you know the frame has this fourth clause at all.

### 6.6 Frame Behavior of Different Window Function Categories

The frame clause **only affects functions that aggregate over the frame**. Not every window function respects it:

**Respect the frame (aggregate/offset-into-frame functions):**
- Aggregates as window functions: `SUM`, `AVG`, `COUNT`, `MIN`, `MAX`, `array_agg`, `string_agg`, etc. — computed over exactly the frame rows.
- `FIRST_VALUE(expr)` — first row *of the frame*.
- `LAST_VALUE(expr)` — last row *of the frame*. **This is the #1 `LAST_VALUE` gotcha:** with the default frame (`... AND CURRENT ROW`), the frame's last row *is the current row*, so `LAST_VALUE` returns the current row's value, not the partition's last value. To get the true partition-final value you must widen the frame to `... AND UNBOUNDED FOLLOWING`.
- `NTH_VALUE(expr, n)` — the nth row of the frame.

**Ignore the frame entirely (they operate on the whole partition):**
- `ROW_NUMBER`, `RANK`, `DENSE_RANK`, `PERCENT_RANK`, `CUME_DIST`, `NTILE` — ranking functions use the full partition and `ORDER BY`; a frame clause on them is accepted syntactically but has no effect.
- `LAG(expr, n)`, `LEAD(expr, n)` — these address rows by fixed offset within the *partition*, not the frame. A frame clause does not change what `LAG` returns.

This split matters: putting a frame on `ROW_NUMBER()` won't error but won't do anything — a sign of a misunderstanding. Frames are for the aggregate-and-value family.

### 6.7 RANGE with Value Offsets and Gaps

`RANGE N PRECEDING/FOLLOWING` measures by the *value* of the single ORDER BY column, which makes it the correct tool for **time-based** trailing windows regardless of how many rows fall in the interval.

```sql
-- "Sum of the last 7 calendar days of revenue for each row's date"
-- Correct even if some days have 0 rows or many rows.
SELECT created_at, amount,
       SUM(amount) OVER (
         ORDER BY created_at
         RANGE BETWEEN INTERVAL '7 days' PRECEDING AND CURRENT ROW
       ) AS trailing_7d
FROM payments;
```

Contrast with GROUPS and ROWS on gapped data. Suppose `day` values are `1, 2, 5, 6` (gaps at 3,4):

- `RANGE BETWEEN 2 PRECEDING AND CURRENT ROW` on the `day=6` row = rows with day in [4,6] = {5,6} (day 4 doesn't exist, but the *value range* still spans it).
- `GROUPS BETWEEN 2 PRECEDING AND CURRENT ROW` on the `day=6` row = current group + 2 preceding groups = days {2,5,6} — it counts *groups*, ignoring the numeric gap.
- `ROWS BETWEEN 2 PRECEDING AND CURRENT ROW` on the last row = last 3 physical rows = days {2,5,6}.

They give different answers precisely because the data has gaps. Choose the mode that matches your intent:
- "within N units of the key's value" → **RANGE**
- "within N distinct key-values back" → **GROUPS**
- "within N physical rows back" → **ROWS**

**NULL ordering and RANGE:** with `ORDER BY col` and NULLs present, NULLs sort together (NULLS LAST by default for ASC). Under RANGE, a NULL current row's "value ± offset" is undefined; PostgreSQL treats NULL as its own peer group at the end/start. In practice, filter NULL ordering keys before framing to avoid surprises.

### 6.8 Empty Frames and NULL Results

A frame can legitimately be **empty** for some rows. Example: `ROWS BETWEEN 3 PRECEDING AND 1 PRECEDING` on the very first row of a partition — there are no preceding rows, so the frame is empty.

What aggregates return over an empty frame:
- `SUM` → **NULL** (not 0! — this trips people up constantly).
- `COUNT` → **0** (COUNT is the exception; it returns 0 over empty).
- `AVG`, `MIN`, `MAX` → **NULL**.
- `FIRST_VALUE`, `LAST_VALUE`, `NTH_VALUE` → **NULL**.

So a "sum of the previous 3 rows excluding current" will be NULL for the first row and partial for rows 2–3. Wrap in `COALESCE(..., 0)` if you need zero:

```sql
COALESCE(SUM(amount) OVER (ORDER BY id ROWS BETWEEN 3 PRECEDING AND 1 PRECEDING), 0)
```

### 6.9 Filtering on Window Results (Frame + Outer Query)

Because windows run after WHERE (section 3), any filter on a framed result needs a wrapping layer:

```sql
-- Keep only rows where the rolling 7-row average exceeds a threshold
SELECT * FROM (
  SELECT day, amount,
         AVG(amount) OVER (ORDER BY day ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS ma7
  FROM daily_sales
) s
WHERE ma7 > 100;
```

The inner query computes the frame; the outer `WHERE` filters. This is unavoidable and correct — the frame must be materialized before it can be compared.

### 6.10 Named Windows and Frame Reuse

When several functions share a window (including its frame), define it once with `WINDOW`:

```sql
SELECT day, amount,
       SUM(amount) OVER w  AS running_sum,
       AVG(amount) OVER w  AS running_avg,
       COUNT(*)    OVER w  AS running_cnt
FROM daily_sales
WINDOW w AS (ORDER BY day ROWS UNBOUNDED PRECEDING);
```

All three share one WindowAgg node and one frame definition — cleaner and often faster than repeating the `OVER (...)` three times (the planner can also collapse identical inline windows, but named windows make intent explicit). You can even inherit and extend: `WINDOW w2 AS (w RANGE ...)` reuses `w`'s partition/order and overrides the frame.

---

## 7. EXPLAIN — WindowAgg and Frames in the Plan

The frame clause itself does not appear as separate text in most EXPLAIN output, but its *cost consequences* do — through the presence of `WindowAgg`, preceding `Sort` nodes, and the actual-time growth pattern.

### 7.1 Incremental frame (running total) — the fast path

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT customer_id, created_at, amount,
       SUM(amount) OVER (PARTITION BY customer_id ORDER BY created_at
                         ROWS UNBOUNDED PRECEDING) AS running
FROM payments;
```

```
WindowAgg  (cost=125000.34..145000.34 rows=1000000 width=48)
           (actual time=812.4..1420.7 rows=1000000 loops=1)
  ->  Sort  (cost=125000.34..127500.34 rows=1000000 width=40)
            (actual time=812.3..980.1 rows=1000000 loops=1)
        Sort Key: customer_id, created_at
        Sort Method: external merge  Disk: 51200kB
        ->  Seq Scan on payments  (cost=0.00..18334.00 rows=1000000 width=40)
                                   (actual time=0.02..210.5 rows=1000000 loops=1)
Buffers: shared hit=8334, temp read=6400 written=6400
Planning Time: 0.4 ms
Execution Time: 1495.2 ms
```

**Reading it:**
- `WindowAgg` is the frame executor. Its input is the `Sort` below it — the mandatory ordering by `PARTITION BY` then `ORDER BY` keys.
- `Sort Method: external merge Disk: 51200kB` — the sort spilled to disk because 1M rows exceeded `work_mem`. This is often the dominant cost, not the window math itself.
- The `ROWS UNBOUNDED PRECEDING` frame is maintained incrementally, so WindowAgg's own overhead (actual time from 812→1420 = ~600ms for 1M rows) is roughly linear — O(N).
- `temp read/written` confirms the disk spill during the sort.

### 7.2 Avoiding the sort with an ordered index

```sql
CREATE INDEX ON payments (customer_id, created_at);

EXPLAIN (ANALYZE, BUFFERS)
SELECT customer_id, created_at, amount,
       SUM(amount) OVER (PARTITION BY customer_id ORDER BY created_at
                         ROWS UNBOUNDED PRECEDING) AS running
FROM payments;
```

```
WindowAgg  (cost=0.43..92000.43 rows=1000000 width=48)
           (actual time=0.05..905.3 rows=1000000 loops=1)
  ->  Index Scan using payments_customer_id_created_at_idx on payments
        (cost=0.43..67000.43 rows=1000000 width=40)
        (actual time=0.03..410.2 rows=1000000 loops=1)
Buffers: shared hit=1000200
Planning Time: 0.5 ms
Execution Time: 975.1 ms
```

**Reading it:**
- No `Sort` node — the composite index already supplies rows in `customer_id, created_at` order, so WindowAgg reads them directly.
- Execution dropped from ~1495ms to ~975ms: eliminating the external sort is the single biggest win for framed queries on large data.
- The index must match the window's `PARTITION BY` + `ORDER BY` exactly (leading columns) to be usable this way.

### 7.3 The O(N²) trap — MAX over a sliding frame

```sql
EXPLAIN (ANALYZE)
SELECT id, amount,
       MAX(amount) OVER (ORDER BY id ROWS BETWEEN 100 PRECEDING AND CURRENT ROW) AS local_max
FROM payments;
```

```
WindowAgg  (cost=125000.34..430000.34 rows=1000000 width=16)
           (actual time=815.1..24300.8 rows=1000000 loops=1)
  ->  Sort  (cost=125000.34..127500.34 rows=1000000 width=12)
            (actual time=815.0..990.4 rows=1000000 loops=1)
        Sort Key: id
        ->  Seq Scan on payments (actual time=0.02..205.1 rows=1000000 loops=1)
Execution Time: 24455.9 ms
```

**Reading it:**
- Same shape as 7.1, but `MAX` has **no inverse transition function** — you can't remove the departing row from a running max. Each of the 1M rows re-scans up to 101 frame rows.
- WindowAgg actual time exploded to ~23.5s (vs ~600ms for the incremental `SUM`). The frame math, not the sort, now dominates.
- Fix options: use `SUM`/`COUNT`/`AVG` (which have inverses) where the metric allows; reduce the frame width; or precompute via a different approach. This is the frame-choice performance cliff made visible.

### 7.4 What to look for in framed-query plans

| Signal in EXPLAIN | Meaning | Action |
|---|---|---|
| `Sort` above the scan, below `WindowAgg` | Ordering the window input | Add composite index on PARTITION BY + ORDER BY cols |
| `Sort Method: external merge Disk:` | Sort spilled — work_mem too small | Raise `work_mem` or add ordered index |
| WindowAgg actual-time growing super-linearly | Non-incremental frame (MAX/MIN sliding, RANGE rescans) | Switch aggregate or frame if possible |
| Two stacked `WindowAgg` nodes | Two different window definitions | Consider a shared `WINDOW` clause / same frame |
| `Index Scan` directly under `WindowAgg`, no Sort | Ideal — ordered input for free | Nothing; this is the goal |

---

## 8. Query Examples

### Example 1 — Basic: Explicit ROWS Running Total

```sql
-- Per-customer running total of payment amounts, ordered by time.
-- ROWS (not the default RANGE) guarantees one-row-at-a-time accumulation,
-- correct even if two payments share the exact same created_at.
SELECT
  p.customer_id,
  p.created_at,
  p.amount,
  SUM(p.amount) OVER (
    PARTITION BY p.customer_id             -- restart the total per customer
    ORDER BY p.created_at, p.id            -- id as tiebreaker → fully deterministic order
    ROWS UNBOUNDED PRECEDING               -- = BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
  ) AS running_total
FROM payments p
ORDER BY p.customer_id, p.created_at, p.id;
```

Adding `p.id` to the window `ORDER BY` breaks ties so that, even under `ROWS`, the accumulation order is deterministic — no two runs disagree.

### Example 2 — Intermediate: 7-Row Moving Average and a True Trailing 7-Day Sum

```sql
-- Two different "last 7" windows on the same data, side by side:
--   ma7_rows  → 7 PHYSICAL rows (this + 6 prior), regardless of dates
--   sum7_days → true 7 CALENDAR days by value (RANGE with an interval offset)
SELECT
  d.day,
  d.amount,

  AVG(d.amount) OVER (
    ORDER BY d.day
    ROWS BETWEEN 6 PRECEDING AND CURRENT ROW     -- exactly 7 rows
  ) AS ma7_rows,

  SUM(d.amount) OVER (
    ORDER BY d.day
    RANGE BETWEEN INTERVAL '6 days' PRECEDING     -- value-based: [day-6, day]
              AND CURRENT ROW
  ) AS sum7_days
FROM daily_sales d
ORDER BY d.day;
```

If `daily_sales` has exactly one row per day with no gaps, the two windows cover the same rows. The moment days are missing or duplicated, they diverge — `RANGE` follows the calendar, `ROWS` follows the row count.

### Example 3 — Production Grade: Rolling 30-Day Revenue and Share-of-Period per Category

```sql
-- Report: for each category and each day it had sales, compute
--   • rolling 30-day revenue (true calendar window, RANGE by date)
--   • that day's share of the category's ENTIRE-period revenue (full-partition frame)
--   • rank of the day within the category by daily revenue
--
-- Table sizes:
--   order_items : 50M rows
--   orders      : 12M rows (indexed on (status, created_at))
--   products    : 200K rows (PK id, indexed category_id)
-- Assumed index for the window: a materialized/CTE result is small per category,
--   so the WindowAgg sorts an already-reduced set (one row per category-day).
-- Perf expectation: heavy work is the join+GROUP BY (hash agg on 50M items);
--   the framed WindowAggs run on the aggregated daily rows (tens of thousands),
--   so the window step is sub-100ms. Total dominated by the scan/agg, ~seconds.

WITH daily AS (
  SELECT
    p.category_id,
    o.created_at::date              AS day,
    SUM(oi.quantity * oi.unit_price) AS revenue
  FROM order_items oi
  JOIN orders   o ON o.id = oi.order_id
  JOIN products p ON p.id = oi.product_id
  WHERE o.status = 'completed'
    AND o.created_at >= DATE_TRUNC('year', CURRENT_DATE) - INTERVAL '1 year'
  GROUP BY p.category_id, o.created_at::date
)
SELECT
  category_id,
  day,
  revenue,

  -- true trailing 30 calendar days, per category
  SUM(revenue) OVER (
    PARTITION BY category_id
    ORDER BY day
    RANGE BETWEEN INTERVAL '29 days' PRECEDING AND CURRENT ROW
  ) AS rolling_30d_revenue,

  -- share of the category's whole-period revenue (full-partition frame → no ORDER BY needed)
  ROUND(
    100.0 * revenue / SUM(revenue) OVER (PARTITION BY category_id)
  , 2) AS pct_of_category_total,

  -- rank ignores the frame (ranking function) — uses the full partition + ORDER BY
  RANK() OVER (
    PARTITION BY category_id
    ORDER BY revenue DESC
  ) AS revenue_rank_in_category
FROM daily
ORDER BY category_id, day;
```

```
-- Representative EXPLAIN (ANALYZE) shape:
Sort  (actual time=4210.5..4215.9 rows=42000 loops=1)
  Sort Key: daily.category_id, daily.day
  ->  WindowAgg  (actual time=4120.1..4180.3 rows=42000 loops=1)      -- RANK window
        ->  WindowAgg  (actual time=4080.7..4110.2 rows=42000 loops=1) -- pct full-partition
              ->  WindowAgg (actual time=4020.4..4060.9 rows=42000)    -- rolling_30d RANGE
                    ->  Sort (actual time=4019.8..4025.1 rows=42000)
                          Sort Key: daily.category_id, daily.day
                          ->  Subquery Scan on daily (actual time=...)
                                ->  HashAggregate (actual time=380..3990 rows=42000)
                                      ->  Hash Join ... 50M order_items ...
Execution Time: 4260.0 ms
```

The stacked `WindowAgg` nodes (one per distinct window definition) all read the small aggregated `daily` set — a few tens of thousands of rows — so the framed windows cost tens of milliseconds. The multi-second total is the 50M-row join and hash aggregate feeding them, not the frames.

---

## 9. Wrong → Right Patterns

### Wrong 1: Relying on the default frame for a running total

```sql
-- WRONG: no explicit frame → defaults to RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
SELECT day, amount,
       SUM(amount) OVER (ORDER BY day) AS running_total
FROM daily_sales;

-- Data with a tie on `day`:
--   day=1 amount=100  → running_total 100
--   day=2 amount=50   → running_total 250   ← both day-2 rows show 250
--   day=2 amount=100  → running_total 250   ← never shows the intermediate 150
--   day=3 amount=30   → running_total 280
```

**Why it's wrong at the execution level:** under the default `RANGE`, `CURRENT ROW` expands to the current row's *entire peer group* (all rows equal on `day`). Both day-2 rows have the same ordering value, so each one's frame is "everything through the end of the day-2 group," yielding the same cumulative total (250) for both. The per-row 150 never appears. On unique keys this is invisible; on ties it corrupts the running balance.

```sql
-- RIGHT: ROWS makes CURRENT ROW mean exactly one row → true per-row accumulation
SELECT day, amount,
       SUM(amount) OVER (ORDER BY day, id ROWS UNBOUNDED PRECEDING) AS running_total
FROM daily_sales;
-- day=2 rows now show 150 then 250 (order made deterministic by the id tiebreaker)
```

### Wrong 2: LAST_VALUE returning the current row instead of the partition's last

```sql
-- WRONG: expecting the LAST value in each partition
SELECT customer_id, created_at, amount,
       LAST_VALUE(amount) OVER (PARTITION BY customer_id ORDER BY created_at) AS last_amount
FROM payments;
-- BUG: last_amount always equals the CURRENT row's amount, not the final one
```

**Why it's wrong at the execution level:** the default frame is `RANGE ... AND CURRENT ROW`. `LAST_VALUE` returns the last row *of the frame*, and the frame ends at the current row — so the "last value" is the current row itself, changing on every row. The frame simply doesn't extend far enough to see the partition's end.

```sql
-- RIGHT: widen the frame end to UNBOUNDED FOLLOWING so the frame spans the whole partition
SELECT customer_id, created_at, amount,
       LAST_VALUE(amount) OVER (
         PARTITION BY customer_id ORDER BY created_at
         ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
       ) AS last_amount
FROM payments;
-- Now last_amount is the final payment amount for each customer, on every row
```

(`FIRST_VALUE` does *not* suffer this because the default frame already starts at `UNBOUNDED PRECEDING` — the first frame row is the partition's first row.)

### Wrong 3: "7-day moving average" that is actually a "7-row moving average"

```sql
-- WRONG: intending a calendar-based trailing 7 days
SELECT day, amount,
       AVG(amount) OVER (ORDER BY day ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS ma7
FROM daily_sales;
-- BUG: if some days are missing (weekends, outages), 6 PRECEDING rows may span
-- 10+ calendar days; if a day has multiple rows, 6 rows may span < 1 day.
```

**Why it's wrong at the execution level:** `ROWS` counts physical rows with no regard for the `day` value. On gapped or duplicated dates, "6 rows back" is not "6 days back." The window silently covers the wrong time span.

```sql
-- RIGHT: RANGE measures by the actual date value → exactly the last 7 calendar days
SELECT day, amount,
       AVG(amount) OVER (
         ORDER BY day
         RANGE BETWEEN INTERVAL '6 days' PRECEDING AND CURRENT ROW
       ) AS ma7
FROM daily_sales;
-- Covers [day-6, day] by value, no matter how many rows fall in that span
```

### Wrong 4: Forgetting an empty frame returns NULL, not 0

```sql
-- WRONG: "sum of the previous 3 rows" used directly in arithmetic
SELECT id, amount,
       amount - SUM(amount) OVER (ORDER BY id
                                  ROWS BETWEEN 3 PRECEDING AND 1 PRECEDING) AS delta
FROM payments;
-- BUG: on the first row the preceding frame is EMPTY → SUM returns NULL →
--      amount - NULL = NULL. The delta column is NULL for the first rows.
```

**Why it's wrong at the execution level:** for the first partition row, `3 PRECEDING AND 1 PRECEDING` selects zero rows. `SUM` over an empty frame is NULL (only `COUNT` returns 0). NULL then poisons the subtraction.

```sql
-- RIGHT: coalesce the empty-frame NULL to 0
SELECT id, amount,
       amount - COALESCE(
         SUM(amount) OVER (ORDER BY id ROWS BETWEEN 3 PRECEDING AND 1 PRECEDING)
       , 0) AS delta
FROM payments;
```

### Wrong 5: Putting a frame on a ranking function and expecting an effect

```sql
-- WRONG: trying to "limit" ROW_NUMBER with a frame
SELECT id,
       ROW_NUMBER() OVER (ORDER BY id ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS rn
FROM payments;
-- The frame is accepted but IGNORED — rn is a normal 1,2,3,... over the whole partition
```

**Why it's wrong at the execution level:** `ROW_NUMBER`, `RANK`, `DENSE_RANK`, `NTILE`, `LAG`, `LEAD` do not consult the frame — they operate on the full partition and its `ORDER BY`. Writing a frame signals a misunderstanding; it does nothing.

```sql
-- RIGHT: if you want a count over a sliding window, use a frame-respecting aggregate
SELECT id,
       COUNT(*) OVER (ORDER BY id ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS rows_in_window
FROM payments;
-- COUNT respects the frame → returns 1,2,3,3,3,... (capped at 3 once the window fills)
```

---

## 10. Performance Profile

### 10.1 The two performance axes

Framed-window cost is governed by two independent questions:

1. **Does the input need a Sort, or can an index supply the order?** The `Sort` that feeds `WindowAgg` is frequently the dominant cost on large data (and spills to disk past `work_mem`). A composite index on `(partition_cols..., order_cols...)` removes it entirely.

2. **Can the frame be maintained incrementally, or must it rescan?** This determines whether WindowAgg itself is O(N) or O(N·W) / O(N²).

### 10.2 Incremental vs rescan by aggregate

| Aggregate | Has inverse-transition? | Sliding frame cost |
|---|---|---|
| `SUM`, `COUNT`, `AVG` (integer/numeric) | Yes | O(N) — add entering, subtract leaving |
| `SUM`/`AVG` (float4/float8) | No (float non-associativity) | O(N·W) rescan |
| `MIN`, `MAX` | No | O(N·W) rescan |
| `FIRST_VALUE` | N/A (edge lookup) | O(N) |
| `LAST_VALUE`, `NTH_VALUE` | N/A (edge lookup) | O(N) |
| `string_agg`, `array_agg` | No | O(N·W) rescan |

Here W = frame width. For `UNBOUNDED PRECEDING AND CURRENT ROW`, the frame only grows, so even non-invertible aggregates like `MAX` are O(N) (state never needs a row removed). The O(N²) danger is specific to **sliding** frames (both edges move) with **non-invertible** aggregates.

### 10.3 Scaling table (single partition, ROWS frame)

| Rows | `SUM` running total (incremental) | `MAX` sliding 100 (rescan) | Dominant cost |
|---|---|---|---|
| 10K | ~5 ms | ~15 ms | window math trivial |
| 100K | ~40 ms | ~180 ms | sort begins to matter |
| 1M | ~1.0 s (mostly sort) | ~24 s | sort (SUM) / rescan (MAX) |
| 10M | ~12 s (sort spills) | minutes → avoid | sort spill / rescan blowup |
| 100M | needs index + partitioning | infeasible as-is | I/O + sort; redesign |

### 10.4 Optimization techniques specific to frames

- **Add the ordered index.** `CREATE INDEX ON t (partition_col, order_col)` matching the window lets WindowAgg read pre-sorted rows — kills the Sort node and its disk spill. Biggest single win.
- **Raise `work_mem` for the session** when a Sort must happen anyway (`SET work_mem = '256MB'`) — keeps the sort in memory, avoiding `external merge Disk:`.
- **Prefer invertible aggregates for sliding windows.** If you need a sliding extremum and it's a hot path, consider precomputing or approximating; `MAX`/`MIN` over wide sliding frames are the classic O(N²) trap.
- **Narrow the frame.** A `6 PRECEDING` window rescans at most 7 rows; a `1000 PRECEDING` window rescans up to 1001. Frame width directly multiplies rescan cost.
- **Aggregate before framing.** As in Example 3, reduce to one row per group *first* (GROUP BY), then run the window on the small result — turns a 50M-row window into a 40K-row window.
- **Share windows.** Multiple functions over the same `WINDOW w` reuse one WindowAgg + one sort instead of stacking nodes.
- **Beware float SUM/AVG losing the incremental path.** If precision allows, `numeric` keeps the O(N) inverse-transition path that `float8` gives up.

---

## 11. Node.js Integration

### 11.1 Running total with an explicit ROWS frame

```javascript
import pg from 'pg';
const { Pool } = pg;
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

async function customerRunningBalance(customerId) {
  const { rows } = await pool.query(
    `SELECT
       created_at,
       amount,
       SUM(amount) OVER (
         ORDER BY created_at, id
         ROWS UNBOUNDED PRECEDING            -- explicit ROWS: true per-row running total
       ) AS running_balance
     FROM payments
     WHERE customer_id = $1
     ORDER BY created_at, id`,
    [customerId]
  );
  return rows;
}
```

The `$1` bind carries the customer id; the frame is static SQL. Note the explicit `ROWS` — do not rely on the default `RANGE`, which would collapse same-timestamp payments into one cumulative value.

### 11.2 True calendar trailing window (RANGE + interval)

```javascript
// Rolling N-day revenue. The interval is parameterized as an integer number of days
// and composed into an INTERVAL using make_interval — safe, no string concatenation.
async function rollingRevenue(customerId, days = 30) {
  const { rows } = await pool.query(
    `SELECT
       created_at::date AS day,
       amount,
       SUM(amount) OVER (
         ORDER BY created_at
         RANGE BETWEEN make_interval(days => $2) PRECEDING AND CURRENT ROW
       ) AS trailing_revenue
     FROM payments
     WHERE customer_id = $1
     ORDER BY created_at`,
    [customerId, days]
  );
  return rows;
}
```

`make_interval(days => $2)` builds the offset from a bound integer — never interpolate an interval via string building (injection risk and type ambiguity).

### 11.3 Handling empty-frame NULLs in application code

```javascript
async function priorThreeSum(customerId) {
  const { rows } = await pool.query(
    `SELECT
       id,
       amount,
       COALESCE(
         SUM(amount) OVER (ORDER BY id ROWS BETWEEN 3 PRECEDING AND 1 PRECEDING)
       , 0) AS prior3_sum      -- COALESCE: empty frame (first rows) yields 0, not NULL
     FROM payments
     WHERE customer_id = $1
     ORDER BY id`,
    [customerId]
  );
  // prior3_sum arrives as a string for numeric types in node-postgres; parse if needed
  return rows.map(r => ({ ...r, prior3_sum: Number(r.prior3_sum) }));
}
```

Remember that `pg` returns `numeric`/`bigint` as JavaScript strings to avoid precision loss — `Number()` or a custom type parser is required if you need numbers. This applies to any framed `SUM`/`AVG` result.

**Do ORMs handle frame clauses?** Largely no. The frame clause is one of the least-supported corners of query builders — most expose window functions only through raw-SQL fragments. Section 12 covers exactly where each falls back to raw SQL.

---

## 12. ORM Comparison

The frame clause is where ORMs are weakest. Window functions themselves have partial support; the frame sub-clause (`ROWS/RANGE/GROUPS ... BETWEEN ... EXCLUDE ...`) is almost universally raw-SQL territory.

### Prisma

**Can Prisma express a frame clause?** — No, not through its type-safe API. Prisma has no window-function builder at all; anything with `OVER (...)` requires `$queryRaw`.

```typescript
import { PrismaClient, Prisma } from '@prisma/client';
const prisma = new PrismaClient();

const rows = await prisma.$queryRaw<Array<{ day: Date; amount: string; running: string }>>(
  Prisma.sql`
    SELECT day, amount,
           SUM(amount) OVER (ORDER BY day, id ROWS UNBOUNDED PRECEDING) AS running
    FROM daily_sales
    WHERE customer_id = ${customerId}
    ORDER BY day, id
  `
);
```

**Where it breaks:** everything window-related. There is no `frame` option, no `.over()`, no partition/order builder. `Prisma.sql` with `${}` interpolation gives safe parameter binding, which is the only Prisma-native benefit here.

**Verdict:** Prisma does not model frames. Use `$queryRaw` with `Prisma.sql` for every framed query; treat Prisma purely as the connection/parameter layer.

---

### Drizzle ORM

**Can Drizzle express a frame clause?** — Partially. Drizzle has a window-function helper API, but frame boundaries typically drop to the `sql` template tag.

```typescript
import { db } from './db';
import { dailySales } from './schema';
import { sql } from 'drizzle-orm';

const rows = await db
  .select({
    day: dailySales.day,
    amount: dailySales.amount,
    // frame written via the sql tag — Drizzle has no first-class ROWS/RANGE builder
    running: sql<string>`
      SUM(${dailySales.amount}) OVER (
        ORDER BY ${dailySales.day}, ${dailySales.id}
        ROWS UNBOUNDED PRECEDING
      )`.as('running'),
  })
  .from(dailySales);
```

**Where it breaks:** the `ROWS/RANGE/GROUPS ... BETWEEN` text is a raw `sql` fragment. Drizzle interpolates column references type-safely, but the frame grammar itself is unmodeled — you write it as SQL text.

**Verdict:** Best-in-class *columns-in-raw-fragment* ergonomics: you keep type-safe column references while writing the frame as SQL. Use the `sql` tag for the `OVER (...)` body.

---

### Sequelize

**Can Sequelize express a frame clause?** — No native support. Use `sequelize.literal()` inside attributes, or a full raw query.

```javascript
const { QueryTypes } = require('sequelize');

// Option A: literal() embedded in a findAll attribute
const rows = await DailySales.findAll({
  attributes: [
    'day', 'amount',
    [sequelize.literal(
      `SUM(amount) OVER (ORDER BY day, id ROWS UNBOUNDED PRECEDING)`
    ), 'running'],
  ],
  order: [['day', 'ASC'], ['id', 'ASC']],
});

// Option B: full raw query (clearer for anything non-trivial)
const raw = await sequelize.query(
  `SELECT day, amount,
          SUM(amount) OVER (ORDER BY day, id ROWS UNBOUNDED PRECEDING) AS running
   FROM daily_sales WHERE customer_id = :cid
   ORDER BY day, id`,
  { replacements: { cid: customerId }, type: QueryTypes.SELECT }
);
```

**Where it breaks:** `literal()` is an unparameterized string — never interpolate user input into it. For bound parameters you must switch to `sequelize.query()` with `replacements`. No frame modeling whatsoever.

**Verdict:** Treat framed windows as raw SQL. Prefer `sequelize.query()` with `replacements` so parameters are bound safely; reserve `literal()` for constant frame text only.

---

### TypeORM

**Can TypeORM express a frame clause?** — Only through `addSelect` with a raw string, or `query()`.

```typescript
const rows = await dataSource
  .getRepository(DailySales)
  .createQueryBuilder('d')
  .select(['d.day', 'd.amount'])
  .addSelect(
    `SUM(d.amount) OVER (ORDER BY d.day, d.id ROWS UNBOUNDED PRECEDING)`,
    'running'
  )
  .where('d.customer_id = :cid', { cid: customerId })
  .orderBy('d.day', 'ASC')
  .addOrderBy('d.id', 'ASC')
  .getRawMany();
```

**Where it breaks:** the window/frame expression is an unchecked raw string in `addSelect` — no type safety on columns, and it lands in `getRawMany()` output (not mapped onto the entity). Parameters in the `WHERE` are still bound safely; the frame itself is literal text.

**Verdict:** Usable via `addSelect` + `getRawMany`, but the frame is raw text. For complex framed analytics, `dataSource.query()` is cleaner.

---

### Knex.js

**Can Knex express a frame clause?** — No dedicated builder, but Knex is the most transparent: `knex.raw()` in a select expresses any frame cleanly, with bound parameters.

```javascript
const rows = await knex('daily_sales')
  .select('day', 'amount')
  .select(
    knex.raw(
      `SUM(amount) OVER (ORDER BY day, id ROWS UNBOUNDED PRECEDING) AS running`
    )
  )
  .where('customer_id', customerId)
  .orderBy([{ column: 'day' }, { column: 'id' }]);

// Parameterized RANGE interval offset:
const trailing = await knex('payments')
  .select('created_at', 'amount')
  .select(
    knex.raw(
      `SUM(amount) OVER (
         ORDER BY created_at
         RANGE BETWEEN make_interval(days => ?) PRECEDING AND CURRENT ROW
       ) AS trailing`,
      [days]
    )
  )
  .where('customer_id', customerId);
```

**Where it breaks:** nothing structural — you write the frame as SQL text, but `knex.raw(sql, [bindings])` gives proper `?` parameter binding even inside the `OVER (...)`. This is the cleanest of the five.

**Verdict:** Best practical choice for framed windows in a query-builder world. `knex.raw()` with positional bindings expresses ROWS/RANGE/GROUPS and even parameterized interval offsets safely.

---

### ORM Summary Table

| ORM | Frame support | How you write it | Param binding inside frame | Verdict |
|---|---|---|---|---|
| Prisma | None | `$queryRaw` + `Prisma.sql` | Yes (`${}`) | Raw only; Prisma is just the driver |
| Drizzle | Partial | `sql` tag in `.select()` | Yes (typed cols) | Best typed column refs in raw frame |
| Sequelize | None | `literal()` / `query()` | Only via `query()` replacements | Raw; avoid `literal()` for user input |
| TypeORM | None | `addSelect` raw + `getRawMany` | WHERE only | Usable but untyped frame text |
| Knex | None (but clean) | `knex.raw()` in select | Yes (`?` bindings) | Most transparent; recommended |

**Overall:** no mainstream Node ORM models the frame clause as structured API. The frame is always SQL text. The differentiator is how safely each lets you bind parameters (like a RANGE interval) and how cleanly it interleaves column references. Knex and Drizzle come out ahead; Prisma/Sequelize/TypeORM push you fully to raw SQL.

---

## 13. Practice Exercises

### Exercise 1 — Basic

Given `payments(id, customer_id, amount, created_at)`:

Write a query that returns, for customer 42, each payment's `created_at`, `amount`, and a **per-row running total** of `amount` ordered by `created_at`. Ensure the running total increments one payment at a time even if two payments share the exact same `created_at`.

Then answer in a comment: what would change in the output if you omitted the frame clause entirely and relied on the default?

```sql
-- Write your query here
```

---

### Exercise 2 — Medium (combines Topic 38 aggregate windows + this topic)

Given `daily_sales(day DATE, amount NUMERIC)` with **at most one row per day but gaps on weekends**:

Write a single query returning, for each day:
- `amount`
- `ma7_rows` — the 7-*row* trailing average
- `ma7_days` — the true 7-*calendar-day* trailing average (must be correct despite weekend gaps)
- `cumulative` — cumulative revenue from the first day through the current day (per-row)

Explain in a comment why `ma7_rows` and `ma7_days` differ on the Monday after a weekend.

```sql
-- Write your query here
```

---

### Exercise 3 — Hard (production simulation; the naive answer is wrong)

Given `payments(id, customer_id, amount, created_at)` where a customer can have **multiple payments at the same `created_at` timestamp** (batch imports):

A teammate wrote this to get each customer's running balance and their final balance on every row:

```sql
SELECT customer_id, created_at, amount,
       SUM(amount)        OVER (PARTITION BY customer_id ORDER BY created_at) AS running,
       LAST_VALUE(amount) OVER (PARTITION BY customer_id ORDER BY created_at) AS final_amount
FROM payments;
```

1. Identify both frame-related bugs (one in `running`, one in `final_amount`).
2. Explain at the execution level why each is wrong given the same-timestamp rows.
3. Write the corrected query so that `running` increments per row and `final_amount` is the customer's last payment amount on every row.

```sql
-- Write your query here
```

---

### Exercise 4 — Interview Simulation

Given `payments(id, customer_id, amount, created_at)`:

Produce, for each payment, the **average of the surrounding payments excluding the payment itself** — specifically, the 2 payments before and the 2 payments after (a centered 5-row window minus the current row), per customer, ordered by `created_at, id`. Rows near a partition edge should average whatever neighbors exist.

State which frame clause and which `EXCLUDE` option you need, and what value the column takes for a customer who has only one payment.

```sql
-- Write your query here
```

---

## 14. Interview Questions

### Q1 — What is the default frame, and why does it surprise people?

**The question:** "If I write `SUM(x) OVER (ORDER BY d)` with no frame clause, what frame do I get, and what's the practical consequence?"

**Junior answer:** "You get a running total — it sums from the start up to the current row."

**Principal answer:** "You get `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` — note `RANGE`, not `ROWS`. Under `RANGE`, `CURRENT ROW` doesn't mean the single current row; it means the current row's entire *peer group* — all rows with the same `d` value. So on unique `d` it behaves like a per-row running total and looks correct. But the moment two rows tie on `d`, both tied rows report the cumulative total *through the end of that peer group* — identical values — instead of incrementing one at a time. The running total appears to skip an intermediate value. The standard chose this default because it's deterministic under ties (a `ROWS` result would depend on the arbitrary physical order of tied rows), but it violates the intuitive per-row mental model. The fix is to write `ROWS UNBOUNDED PRECEDING` explicitly and add a tiebreaker to the `ORDER BY`. If there's no `ORDER BY` at all, the default is instead the whole partition, `RANGE BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`."

**Interviewer follow-up:** "Show me a two-row dataset where the default and `ROWS` disagree." (Two rows with equal ordering value; default RANGE gives both the sum-of-both, ROWS gives first then first+second.)

---

### Q2 — Explain ROWS vs RANGE vs GROUPS.

**The question:** "Three frame modes exist. When would you reach for each?"

**Junior answer:** "ROWS counts rows, RANGE is a range of values. I usually just use ROWS."

**Principal answer:** "They differ in how they measure the frame boundary. `ROWS` counts physical rows — `2 PRECEDING` is literally two rows back, ties irrelevant; use it for fixed-row windows and per-row running totals. `RANGE` measures by the *value* of the single ORDER BY column — `INTERVAL '7 days' PRECEDING` means all rows whose timestamp is within 7 days, regardless of how many rows that is; use it for calendar/value-based trailing windows and when you want tie-stable `CURRENT ROW` (whole peer group). `GROUPS` counts *peer groups* — `2 PRECEDING` means two distinct tie-clusters back; use it when you want 'N distinct ordering values back' irrespective of gaps or duplicate counts. The key discriminator is behavior under ties and gaps: on unique, gapless integer keys all three coincide, which is exactly why the differences hide until real data. `RANGE` with an offset requires exactly one ORDER BY column of a type that supports arithmetic with the offset; `GROUPS` requires an ORDER BY; `ROWS` requires neither."

**Interviewer follow-up:** "Data has `day` values 1, 2, 5, 6. For `... 2 PRECEDING AND CURRENT ROW` on the `day=6` row, which rows does each mode include?" (RANGE: values in [4,6] → {5,6}; GROUPS: 2 groups back → {2,5,6}; ROWS: 3 physical rows → depends on order, {2,5,6} if those are the last three.)

---

### Q3 — Why does LAST_VALUE often return the wrong thing?

**The question:** "A developer uses `LAST_VALUE(x) OVER (PARTITION BY g ORDER BY t)` to get the last value per group and it returns the current row's value instead. Why?"

**Principal answer:** "`LAST_VALUE` returns the last row *of the frame*, and the default frame ends at `CURRENT ROW`. So the frame never extends past the current row — its 'last' row is always the current row itself, and the column just mirrors the current value. The frame is too narrow. The fix is to set the frame end to `UNBOUNDED FOLLOWING`: `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`, so the frame spans the whole partition and its last row is the partition's actual last row. `FIRST_VALUE` doesn't have this problem because the default frame already starts at `UNBOUNDED PRECEDING`. It's a pure frame-boundary issue, not a bug in the function."

**Interviewer follow-up:** "Could you get the same result without widening the frame?" (Yes — e.g., `FIRST_VALUE(x) OVER (PARTITION BY g ORDER BY t DESC)`, reversing the order so the last becomes first; or `NTH_VALUE` with the full frame.)

---

### Q4 — When does a window function become O(N²)?

**The question:** "You have a `SUM` sliding window that's fast and a `MAX` sliding window over the same frame that's 40× slower. Why?"

**Principal answer:** "Sliding frames move both edges each step — rows enter at the front and leave at the back. `SUM`, `COUNT`, and `AVG` (over exact types) have an *inverse transition function*: PostgreSQL maintains a running state, adds the entering row and subtracts the leaving row, O(1) per step, O(N) total. `MAX` and `MIN` have no inverse — you can't 'un-max' a value that's leaving without knowing the next-largest — so WindowAgg must rescan every frame row for each current row: O(N·W), degrading toward O(N²) as the frame widens. Float `SUM`/`AVG` also lose the inverse path because floating-point addition isn't associative, so PostgreSQL won't risk the subtraction. Note that `UNBOUNDED PRECEDING AND CURRENT ROW` frames only *grow*, never shed rows, so even `MAX` stays O(N) there — the O(N²) trap is specific to sliding frames with non-invertible aggregates. Mitigations: narrow the frame, switch to an invertible aggregate, use `numeric` instead of `float`, or precompute."

**Interviewer follow-up:** "How would you spot this in EXPLAIN ANALYZE?" (WindowAgg actual-time growing super-linearly with row count, disproportionate to the Sort cost; compare against the same query with SUM.)

---

## 15. Mental Model Checkpoint

1. You write `SUM(amount) OVER (ORDER BY created_at)` with no frame clause. Two rows have identical `created_at`. What running-total value does each of those two rows show, and why is it not the per-row incremental value?

2. Under `RANGE`, what does `CURRENT ROW` physically include when the current row is part of a 3-row tie? How does that differ from `ROWS`?

3. `LAST_VALUE(x) OVER (PARTITION BY g ORDER BY t)` returns the current row's value on every row. What single change to the frame fixes it, and why doesn't `FIRST_VALUE` need the same fix?

4. Data has `day` values 10, 20, 40, 50. For a frame `2 PRECEDING AND CURRENT ROW` on the `day=50` row, name the rows included under ROWS, RANGE, and GROUPS. Why do they differ?

5. A sliding `MAX()` window is far slower than a sliding `SUM()` window over the identical frame. What property of the aggregate causes this, and which frame boundary would make even `MAX` fast again?

6. Over an empty frame (e.g. `3 PRECEDING AND 1 PRECEDING` on the first partition row), what does `SUM` return? What does `COUNT` return? Why are they different, and how do you defend downstream arithmetic?

7. You put `ROWS BETWEEN 2 PRECEDING AND CURRENT ROW` on a `RANK()` call and it has no effect. Which window functions ignore the frame, and which respect it?

---

## 16. Quick Reference Card

```sql
-- ── FRAME SYNTAX ───────────────────────────────────────────────
OVER (
  PARTITION BY ...          -- optional: slice into partitions
  ORDER BY ...              -- orders rows; defines PRECEDING/FOLLOWING & peer groups
  { ROWS | RANGE | GROUPS } BETWEEN <start> AND <end>
  [ EXCLUDE { CURRENT ROW | GROUP | TIES | NO OTHERS } ]
)
-- <start>/<end>: UNBOUNDED PRECEDING | N PRECEDING | CURRENT ROW
--              | N FOLLOWING | UNBOUNDED FOLLOWING

-- ── DEFAULT FRAME (memorize) ───────────────────────────────────
-- ORDER BY present, no frame  →  RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
-- no ORDER BY                  →  RANGE BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
-- The default is RANGE, not ROWS. On ties they DIFFER. Write ROWS for per-row totals.

-- ── COMMON FRAMES ──────────────────────────────────────────────
ROWS UNBOUNDED PRECEDING                          -- per-row running total
ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING  -- whole partition (LAST_VALUE, %-of-total)
ROWS BETWEEN 6 PRECEDING AND CURRENT ROW          -- trailing 7 ROWS
ROWS BETWEEN 2 PRECEDING AND 2 FOLLOWING          -- centered 5 rows
ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING  -- reverse cumulative
RANGE BETWEEN INTERVAL '30 days' PRECEDING AND CURRENT ROW  -- trailing 30 CALENDAR days

-- ── MODE CHEAT SHEET ───────────────────────────────────────────
-- ROWS   → count physical rows; ties ignored; CURRENT ROW = 1 row
-- RANGE  → by ORDER BY VALUE; CURRENT ROW = whole peer group; offset needs 1 ORDER BY col
-- GROUPS → count peer groups; CURRENT ROW = whole peer group; needs ORDER BY

-- ── EXCLUDE ────────────────────────────────────────────────────
-- NO OTHERS (default) | CURRENT ROW (drop self, keep peers)
-- GROUP (drop self + peers) | TIES (drop peers, keep self)

-- ── GOTCHAS ────────────────────────────────────────────────────
-- LAST_VALUE with default frame  → returns CURRENT row (widen to UNBOUNDED FOLLOWING)
-- SUM over empty frame           → NULL (COUNT → 0); wrap in COALESCE(...,0)
-- RANK/ROW_NUMBER/LAG/LEAD       → ignore the frame entirely
-- MAX/MIN over sliding frame     → O(N²) (no inverse-transition); SUM/COUNT/AVG → O(N)
-- float SUM/AVG sliding          → loses O(N) inverse path; numeric keeps it
```

**Perf rules of thumb:**
- Add `CREATE INDEX ON t (partition_cols, order_cols)` to remove the Sort feeding WindowAgg.
- Sliding-frame `MAX`/`MIN` is the O(N²) trap; growing frames (`UNBOUNDED PRECEDING AND CURRENT ROW`) stay O(N) even for MAX.
- Aggregate (GROUP BY) *before* windowing to shrink the frame input.

**Interview one-liners:**
- "The default frame is `RANGE ... CURRENT ROW`, and `CURRENT ROW` under RANGE means the whole peer group — that's why running totals surprise people on ties. Write `ROWS`."
- "`ROWS` counts rows, `RANGE` measures by value, `GROUPS` counts peer groups; they only diverge on ties and gaps."
- "`LAST_VALUE` looks wrong because the frame ends at CURRENT ROW — widen to UNBOUNDED FOLLOWING."
- "`SUM` slides in O(N) via inverse-transition; `MAX` can't un-max, so sliding MAX is O(N²)."

---

## Connected Topics

- **Topic 38 — Aggregate Window Functions** (previous): `SUM`/`AVG`/`COUNT` as window functions — the frame clause is what makes them cumulative, sliding, or whole-partition. This topic is the mechanics *underneath* those aggregates.
- **Topic 40 — Window Function Performance** (next): the WindowAgg node, sort elimination via indexes, `work_mem` spills, and the inverse-transition optimization introduced here get their full performance treatment.
- **Topic 36 — Window Functions Introduction**: `OVER`, `PARTITION BY`, and window `ORDER BY` — the two components that precede the frame in a window definition.
- **Topic 37 — Ranking Functions**: `ROW_NUMBER`, `RANK`, `DENSE_RANK`, `NTILE` — the functions that *ignore* the frame, and why (they operate on the full partition).
- **Topic 07 — NULL in Depth**: empty-frame `SUM` returns NULL (not 0); NULL ordering keys form their own peer group — the three-valued-logic and NULL-ordering rules at work in frames.
- **Topic 20 — GROUP BY Fundamentals**: pre-aggregating before a window shrinks the frame input; GROUP BY collapses rows, windows retain them — the two are complementary, not interchangeable.
- **Internals — Sort and WindowAgg**: the mandatory sort feeding WindowAgg, the tuplestore buffering, and external-merge disk spills — the physical substrate every frame clause runs on.

