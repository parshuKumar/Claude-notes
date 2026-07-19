# Topic 61 — Pivot and Crosstab
### SQL Mastery Curriculum — Phase 9: Advanced SQL Patterns

---

## 1. ELI5 — The Analogy That Makes It Click

Imagine you run a small coffee shop and you keep a notebook. Every time you sell something, you write one line:

```
Monday    | espresso    | 3
Monday    | latte       | 5
Tuesday   | espresso    | 2
Tuesday   | latte       | 4
Wednesday | espresso    | 6
```

This is the **long format** — one fact per line, stacked vertically. It's how databases love to store things: skinny, tall, easy to append to. Every new sale is just one more line at the bottom.

But when your accountant asks "show me the week at a glance," she doesn't want a scroll. She wants a **grid**:

```
day        | espresso | latte
-----------+----------+------
Monday     |    3     |   5
Tuesday    |    2     |   4
Wednesday  |    6     |   0
```

Notice what happened. The *values* that used to live in the `product` column — "espresso", "latte" — became **column headers**. The rows collapsed: three lines for Monday-and-Tuesday-and-Wednesday became one line per day. This rotation — taking the distinct values of one column and spreading them out sideways into new columns — is called a **pivot** (or a **crosstab**, short for cross-tabulation).

Here's the catch that trips everyone up, and it's the single most important thing in this entire topic: **to draw that grid, you must know the column names in advance.** Before the accountant sees a single number, someone has to decide "the columns are `espresso` and `latte`." If tomorrow you start selling `cappuccino`, the grid does not magically grow a column — you have to redraw it. A notebook can grow a new *row* effortlessly. Growing a new *column* is a structural change.

That asymmetry is the soul of pivoting in SQL. Rows are cheap and dynamic. Columns are fixed and must be named up front. Everything hard about pivots comes from fighting that one rule.

The reverse operation — taking that wide grid and melting it back down into skinny long lines — is called an **unpivot**. You'd do it when someone hands you a spreadsheet shaped like the accountant's grid and you need to load it into a normalized table.

---

## 2. Connection to SQL Internals

Pivoting is not a native relational operation. The relational algebra — select, project, join, union, aggregate — has no "rotate" operator. So when you pivot in SQL, you are *emulating* rotation using operators the engine already has. Understanding which internal machinery you're borrowing tells you exactly what it costs.

**The aggregate + conditional trick.** The portable pivot pattern (`CASE ... END` inside `SUM`/`COUNT`, or the `FILTER` clause) rides entirely on the **GROUP BY aggregation machinery** you learned in Topic 20. The engine builds one aggregate state per group (either via a **HashAggregate** — a hash table keyed by the GROUP BY columns, held in `work_mem` — or a **GroupAggregate** over a sorted stream). For each incoming row, it updates *every* aggregate for that group's bucket. A pivot with 12 output columns is just 12 aggregate transition functions firing per row, each guarded by a `CASE`. There is no special pivot node in the plan — you will see a plain `HashAggregate` or `GroupAggregate`, exactly as if you'd written a normal grouped query.

**Why the column list must be static.** PostgreSQL's planner resolves the output row descriptor — the number of columns, their names, their types — at **parse/plan time**, before a single heap page is read. This is the `TupleDesc`: a fixed struct nailed down during planning. The executor then streams tuples that all conform to that one descriptor. There is no mechanism for the executor to *discover* it needs a 13th column halfway through the scan because it just saw a new `product` value. This is not a limitation someone forgot to fix; it is foundational to how a compiled query plan streams rows. **This single fact is why "dynamic pivot" is impossible in pure SQL** and why every workaround (dynamic SQL, JSON aggregation, application-side reshaping) exists.

**The `crosstab()` function and its RECORD problem.** The `tablefunc` extension provides `crosstab()`, a C-level set-returning function that does the row-collapsing internally (it reads an ordered input and emits one output row per group, filling category slots positionally). But because it's a set-returning function returning `RECORD`, PostgreSQL still cannot know its shape at plan time — so **you must hand-write the column definition list** (`AS ct(row_name text, cat1 int, cat2 int, ...)`). You've moved the "columns must be static" tax from the `CASE` expressions into the `AS` clause, but you have not escaped it.

**Unpivot and the `LATERAL` node.** Unpivoting one wide row into N tall rows is a **set-returning function in the target list** or a **`LATERAL` join** (Topic 33) against a `VALUES`/`unnest` list. Internally this is a `ProjectSet` node or a nested-loop `LATERAL` join: for each input row, the SRF emits several output rows. `UNION ALL` unpivot is N sequential scans of the same table stitched by an `Append` node — more I/O, but dead simple.

**Memory.** A `HashAggregate` pivot holds one row of aggregate state per group. If the number of *groups* (output rows) exceeds `work_mem`, PostgreSQL 13+ spills the hash table to disk (batched, like Hash Join). The number of *pivot columns* affects the width of each state row, not the number of buckets. So a pivot with millions of output rows is the memory risk — not a pivot with many columns over few rows.

---

## 3. Logical Execution Order Context

Pivoting via the aggregate pattern lives squarely inside the clauses you already know, and its position in the pipeline explains several gotchas:

```
FROM          ← source rows, in long format
JOIN          ← bring in the label table (e.g. product names)
WHERE         ← filter BEFORE aggregation (cheap, reduces rows fanned into groups)
GROUP BY      ← collapse to one row per pivot ROW-key (e.g. per day)
HAVING        ← filter the pivoted groups (e.g. only days with > 0 total)
SELECT        ← the CASE/FILTER aggregates evaluate HERE, producing the columns
DISTINCT
ORDER BY      ← sort the finished grid
LIMIT
```

Key consequences of this ordering:

- **The `CASE`/`FILTER` aggregates are `SELECT`-list expressions evaluated during the GROUP BY step.** They cannot reference each other, and they cannot appear in `WHERE` (which runs first). To filter *on* a pivoted column, you must use `HAVING`, or wrap the pivot in a subquery/CTE and filter the outer query.

- **`WHERE` filters the long rows before they're bucketed.** Filtering `WHERE product IN ('espresso','latte')` is far cheaper than pivoting everything and discarding columns, because it shrinks the input to the aggregate.

- **The set of output columns is decided while writing the query, not during execution.** By the time `FROM` runs, the column list is already frozen. This is the execution-order restatement of the "no dynamic pivot" law from Section 2.

- **For `crosstab()`**, the ordering is different: `crosstab()` sits in the `FROM` clause as a table source. Its *input* query (a string) does its own `FROM/WHERE/ORDER BY`, and critically **must be ordered by `row_name, category`** so the C function can walk groups in order. The outer query then treats `crosstab(...)` as an ordinary relation you can `WHERE`, `JOIN`, and `ORDER BY`.

- **Unpivot with `LATERAL`/`unnest`** happens in the `FROM`/`JOIN` phase — it multiplies rows *before* `WHERE` sees them, so a `WHERE` on the unpivoted value filters the already-melted rows.

---

## 4. What Is Pivot / Crosstab?

**Pivot** (cross-tabulation) is the transformation that turns the distinct *values* of one column into *columns of the result*, collapsing multiple input rows into one output row per remaining key, with an aggregate filling each cell. **Unpivot** is the inverse: it turns a set of columns back into (key, column-name, value) rows. Because SQL's result shape is fixed at plan time, a pivot's target columns must be enumerated explicitly in the query text.

### The portable pattern — CASE inside an aggregate

```sql
SELECT
  d.day                                              AS day,
  SUM(CASE WHEN s.product = 'espresso' THEN s.qty END) AS espresso,
  SUM(CASE WHEN s.product = 'latte'    THEN s.qty END) AS latte,
  SUM(CASE WHEN s.product = 'mocha'    THEN s.qty END) AS mocha
FROM sales s
JOIN days d ON d.id = s.day_id
GROUP BY d.day;
```

Annotated:

```
SELECT
  d.day                                                 AS day,
  │    └── the ROW key — one output row per distinct value of this
  │        (must appear in GROUP BY)
  SUM(CASE WHEN s.product = 'espresso' THEN s.qty END)  AS espresso
  │   │    │                            │          │       └── the new COLUMN name (a literal identifier,
  │   │    │                            │          │           chosen by YOU at write time — this is why
  │   │    │                            │          │           the pivot cannot be dynamic)
  │   │    │                            │          └── ELSE is omitted → defaults to NULL, so non-matching
  │   │    │                            │              rows contribute NULL, which SUM ignores
  │   │    │                            └── the value to place in the cell when this row belongs here
  │   │    └── the predicate that selects WHICH rows belong in THIS column
  │   └── the aggregate that reduces all matching rows in the group to one cell value
  │       (SUM, COUNT, MAX, AVG, bool_or, string_agg — any aggregate works)
  └── ...one such expression PER output column. N columns = N hand-written expressions.
FROM sales s
JOIN days d ON d.id = s.day_id
GROUP BY d.day;
   │        └── everything NOT being spread into columns; defines the output rows
   └── GROUP BY is mandatory — the pivot IS an aggregation
```

### The `FILTER` variant (SQL-standard, PostgreSQL 9.4+)

```sql
SELECT
  d.day,
  SUM(s.qty) FILTER (WHERE s.product = 'espresso') AS espresso,
  SUM(s.qty) FILTER (WHERE s.product = 'latte')    AS latte,
  SUM(s.qty) FILTER (WHERE s.product = 'mocha')    AS mocha
FROM sales s
JOIN days d ON d.id = s.day_id
GROUP BY d.day;
```

`FILTER (WHERE ...)` is a cleaner, standard way to write the same thing:

```
SUM(s.qty) FILTER (WHERE s.product = 'espresso') AS espresso
│          │             └── only rows satisfying this predicate feed the aggregate
│          └── the FILTER clause — attaches a per-aggregate row filter
└── the value being aggregated is now clean (s.qty), not a CASE expression
```

`FILTER` and `CASE` compile to nearly identical plans. `FILTER` reads better and lets you keep the aggregated expression separate from the selection predicate. Use it when your PostgreSQL version supports it (all supported versions do).

### The `crosstab()` function (tablefunc extension)

```sql
CREATE EXTENSION IF NOT EXISTS tablefunc;

SELECT * FROM crosstab(
  $$ SELECT day, product, qty FROM sales_by_day ORDER BY 1, 2 $$,   -- source query
  $$ SELECT unnest(ARRAY['espresso','latte','mocha']) $$            -- category query
) AS ct(day text, espresso int, latte int, mocha int);             -- column definition
```

Annotated:

```
crosstab(
  $$ SELECT day, product, qty FROM sales_by_day ORDER BY 1, 2 $$,
  │        │    │        │                       └── MUST order by row_name, then category
  │        │    │        └── column 3+ = the VALUE(s) placed in cells
  │        │    └── column 2 = the CATEGORY (becomes a column, matched against category query)
  │        └── column 1 = the ROW name (the grouping key)
  │
  $$ SELECT unnest(ARRAY['espresso','latte','mocha']) $$
  │  └── the ordered list of categories → determines column ORDER and which columns exist
  │
) AS ct(day text, espresso int, latte int, mocha int)
        │         └── you STILL hand-write every column name and type here
        └── the column definition list is mandatory — crosstab returns RECORD
```

`INNER` join semantics apply within `crosstab`: a `day` that has no `espresso` row simply gets `NULL` in that cell. `crosstab` fills cells positionally by matching the category value to the category-query list.

---

## 5. Why Pivot Mastery Matters in Production

1. **Reporting and dashboards are pivots.** Almost every "matrix" report a stakeholder asks for — revenue by month across regions, signups by plan across weeks, error counts by service across days — is a pivot. If you can't pivot fluently, you push the reshaping into the application layer, shipping tall data over the wire and reassembling it in JavaScript. That's more bytes, more latency, and reshape logic scattered across the codebase.

2. **The "dynamic columns" trap wastes days.** A junior engineer promises the product manager "a column per product category, automatically." They discover mid-sprint that SQL cannot do this without dynamic SQL, then either hack together fragile string-building or over-engineer a code generator. Knowing the fixed-column law up front lets you make the right call immediately: enumerate the columns, generate the SQL from a known list, or return JSON and pivot in the app.

3. **Fan-out and double-counting bugs hide in pivots.** A pivot is a `GROUP BY`. All the fan-out hazards from Topic 11 and Topic 20 apply: join a one-to-many table before pivoting and your cells silently sum duplicated rows. The bug is invisible because the grid *looks* plausible — every cell has a number, just the wrong number.

4. **`crosstab()` misuse produces misaligned grids.** `crosstab(text)` (the one-argument form) fills columns *left to right in the order categories appear*, not by matching category names. If different rows have different missing categories, values land in the wrong columns — a silent, dangerous data-corruption-shaped bug. Knowing to use the two-argument `crosstab(source, categories)` form prevents this entire class of error.

5. **Unpivot is how you fix bad schemas.** Wide tables (`jan_sales, feb_sales, ..., dec_sales`) are an anti-pattern that blocks indexing, filtering, and time-series queries. Unpivoting them into (month, sales) rows is the standard remediation. Doing it with `LATERAL`/`unnest` in one pass is far cheaper than twelve `UNION ALL` scans.

6. **`work_mem` and spill behavior.** A pivot over 50M long rows producing 2M output rows builds a 2M-bucket hash aggregate. If that exceeds `work_mem`, it spills to disk and slows sharply. Knowing that the *row count*, not the column count, drives memory lets you size `work_mem` or pre-filter correctly.

---

## 6. Deep Technical Content

### 6.1 The three cell-fill aggregates and when each is right

The aggregate wrapping your `CASE`/`FILTER` determines what a cell means and how it behaves on the empty and multi-row cases.

```sql
-- SUM: additive measures (revenue, quantity). Empty cell → NULL (SUM ignores NULLs).
SUM(qty) FILTER (WHERE product = 'espresso')

-- COUNT: occurrence counts. Empty cell → 0 (COUNT never returns NULL).
COUNT(*) FILTER (WHERE product = 'espresso')

-- MAX/MIN: "pick the one value" when you KNOW there's at most one row per cell.
MAX(price) FILTER (WHERE product = 'espresso')

-- bool_or / bool_and: presence/flag matrices.
bool_or(true) FILTER (WHERE product = 'espresso')  -- did this product sell that day?

-- string_agg: collect multiple values into one cell.
string_agg(note, '; ') FILTER (WHERE product = 'espresso')
```

**The empty-cell asymmetry is a frequent bug.** `SUM` and `MAX` return `NULL` for a cell with no matching rows; `COUNT` returns `0`. If you want zeros instead of NULLs in a `SUM` pivot, wrap it:

```sql
COALESCE(SUM(qty) FILTER (WHERE product = 'espresso'), 0) AS espresso
```

Report consumers almost always want `0`, not `NULL`, in an empty revenue cell. Forgetting the `COALESCE` produces grids with holes.

### 6.2 CASE vs FILTER — equivalence and the one real difference

```sql
-- These two produce identical results and near-identical plans:
SUM(CASE WHEN product = 'espresso' THEN qty END)      -- CASE form
SUM(qty) FILTER (WHERE product = 'espresso')          -- FILTER form
```

Both fire the `SUM` transition function only for espresso rows (the `CASE` feeds `NULL` for others, which `SUM` skips; `FILTER` skips them before they reach the aggregate). The difference shows up with `COUNT`:

```sql
COUNT(CASE WHEN product = 'espresso' THEN 1 END)  -- counts non-NULL → espresso rows. OK.
COUNT(CASE WHEN product = 'espresso' THEN qty END) -- counts rows where qty IS NOT NULL too!
COUNT(*) FILTER (WHERE product = 'espresso')       -- unambiguous: espresso rows, period.
```

With `COUNT`, the `CASE` form counts non-NULL results of the `CASE` expression, so if `qty` can be NULL you undercount. `COUNT(*) FILTER` counts *rows*, which is almost always what you mean. This is the strongest practical reason to prefer `FILTER`.

Also, `FILTER` cannot be combined with `DISTINCT` inside the same aggregate in older versions carelessly — `COUNT(DISTINCT x) FILTER (WHERE ...)` *is* supported in modern PostgreSQL; the `CASE` equivalent `COUNT(DISTINCT CASE WHEN ... THEN x END)` also works but counts distinct non-NULL CASE results.

### 6.3 Two-dimensional pivots (row key AND sub-row key)

You can pivot with a compound row key — e.g., revenue by (region, month-name) across product columns:

```sql
SELECT
  region,
  to_char(created_at, 'YYYY-MM')                              AS month,
  COALESCE(SUM(amount) FILTER (WHERE category = 'hardware'), 0) AS hardware,
  COALESCE(SUM(amount) FILTER (WHERE category = 'software'), 0) AS software,
  COALESCE(SUM(amount) FILTER (WHERE category = 'services'), 0) AS services
FROM orders o
JOIN products p ON p.id = o.product_id
GROUP BY region, to_char(created_at, 'YYYY-MM')
ORDER BY region, month;
```

Everything in `GROUP BY` that isn't spread into columns forms the row identity. This is the most common real-world shape: a couple of dimensions down the left, one dimension across the top, a measure in the cells.

### 6.4 Pivoting multiple measures at once

Sometimes each cell needs more than one number (revenue *and* units). You simply write more columns — the naming convention `<measure>_<category>` keeps it readable:

```sql
SELECT
  region,
  SUM(amount) FILTER (WHERE category = 'hardware') AS hardware_rev,
  SUM(units)  FILTER (WHERE category = 'hardware') AS hardware_units,
  SUM(amount) FILTER (WHERE category = 'software') AS software_rev,
  SUM(units)  FILTER (WHERE category = 'software') AS software_units
FROM sales
GROUP BY region;
```

The column count multiplies: `categories × measures`. Three categories × two measures = six hand-written columns. This is where the enumeration burden becomes real and where code-generating the SQL from a known list starts to pay off.

### 6.5 The fundamental limitation — no dynamic pivot in pure SQL

This deserves its own careful treatment because it's the #1 interview trap and the #1 production surprise.

**The rule:** a single SQL statement's output columns (count, names, types) are fixed when the statement is planned, before any data is read. Therefore a statement cannot grow columns based on the data it encounters. If a new `product` value appears, no pre-written pivot query will show it.

**The three legitimate escapes:**

**(a) Enumerate — accept a fixed, known set of columns.** If the categories are stable (months of the year, a controlled enum of plan tiers, days of the week), just write them out. This is the correct answer far more often than people expect. Maintenance is trivial: twelve months don't change.

**(b) Dynamic SQL — build the query string, then execute it.** Query the distinct categories first, construct the `SELECT ... CASE ...` text, and run it. Inside PostgreSQL this is a PL/pgSQL function using `format()` and `EXECUTE`; it must return a fixed type or `refcursor` or `SETOF record` (which forces the *caller* to supply a column list — so you've only moved the problem). This is two round trips or a function, and it demands rigorous quoting to avoid SQL injection.

```sql
-- Sketch of a dynamic-pivot function (PL/pgSQL)
CREATE OR REPLACE FUNCTION pivot_sales()
RETURNS SETOF record LANGUAGE plpgsql AS $$
DECLARE
  cols text;
  q    text;
BEGIN
  SELECT string_agg(
           format('SUM(qty) FILTER (WHERE product = %L) AS %I',
                  product, product),
           ', ' ORDER BY product)
    INTO cols
  FROM (SELECT DISTINCT product FROM sales) s;

  q := format('SELECT day, %s FROM sales GROUP BY day ORDER BY day', cols);
  RETURN QUERY EXECUTE q;   -- caller must still say AS t(day text, ...)
END; $$;
```

Note the two safety tools: `%L` (quote as a **literal**, injection-safe) for the value inside the predicate, and `%I` (quote as an **identifier**, injection-safe) for the generated column name. Never build these with plain string concatenation.

**(c) Return a document — pivot outside the fixed-column world.** Aggregate into `JSON`/`JSONB` (or `hstore`, or an array) where keys are the category values. The result is *one* column of type `jsonb` whose shape can vary per row — the column list stays fixed (it's always "one jsonb column") while the *contents* are dynamic. The application then reads the object.

```sql
SELECT
  day,
  jsonb_object_agg(product, qty) AS by_product   -- {"espresso": 3, "latte": 5, ...}
FROM sales
GROUP BY day;
```

This is often the *best* production answer for truly dynamic categories: the database does the aggregation, the wire carries a compact object, and the front end renders whatever keys exist. You trade rigid columns for a flexible document — which is frequently exactly what a dashboard wants.

### 6.6 crosstab() in depth — the two forms and the alignment trap

`tablefunc` offers two overloads. The distinction is safety-critical.

**One-argument `crosstab(text)`** — fills columns *positionally by appearance order*:

```sql
SELECT * FROM crosstab(
  $$ SELECT day, product, qty FROM sales ORDER BY 1, 2 $$
) AS ct(day text, c1 int, c2 int, c3 int);
```

Danger: it takes, per row, the 1st category it sees → column 1, the 2nd → column 2, etc. If row `Monday` has (espresso, latte, mocha) but row `Tuesday` has only (latte, mocha), then Tuesday's `latte` lands in `c1` and `mocha` in `c2` — **misaligned against Monday**. The one-argument form is only safe when every row has exactly the same, complete set of categories in the same order. In practice that's rarely guaranteed. **Avoid it.**

**Two-argument `crosstab(source, categories)`** — fills columns *by matching the category value* against the category list, inserting `NULL` for missing categories:

```sql
SELECT * FROM crosstab(
  $$ SELECT day, product, qty FROM sales ORDER BY 1, 2 $$,
  $$ SELECT DISTINCT product FROM sales ORDER BY 1 $$
) AS ct(day text, espresso int, latte int, mocha int);
```

Now `Tuesday`'s missing `espresso` correctly becomes `NULL` in the `espresso` column. **Always use the two-argument form** unless you can prove completeness.

**Hard requirements for `crosstab`:**
- The source query **must** be `ORDER BY row_name, category`. Out-of-order input silently produces wrong groupings — it does not error.
- Row 1 = row name, row 2 = category, row 3 = value (extra source columns beyond 3 are ignored in the standard form; there's a variant with extra row-header columns).
- The `AS ct(...)` list must have exactly `1 + N` columns (row name + N categories) with correct types, or you get a runtime error.
- The category query in the two-arg form should return the categories in the same order you list them in `AS ct(...)`.

**When to use `crosstab` vs `CASE`/`FILTER`:** `crosstab` reads the source **once** and is written in C, so for very wide pivots (dozens of categories) over large inputs it can beat the `CASE` pattern, which evaluates N predicates per row. But it requires the extension, has strict input ordering, and the column definition is just as static. For most pivots, `FILTER` is simpler, portable, and fast enough. Reach for `crosstab` when the category count is high and the extension is available.

### 6.7 Unpivot — turning columns back into rows

Given a wide table:

```
region  | q1   | q2   | q3   | q4
--------+------+------+------+-----
West    | 100  | 120  | 90   | 140
East    | 80   | 95   | 110  | 130
```

**Method A — `UNION ALL` (portable, verbose, N scans):**

```sql
SELECT region, 'q1' AS quarter, q1 AS amount FROM regional_sales
UNION ALL
SELECT region, 'q2', q2 FROM regional_sales
UNION ALL
SELECT region, 'q3', q3 FROM regional_sales
UNION ALL
SELECT region, 'q4', q4 FROM regional_sales;
```

Simple and works everywhere, but scans `regional_sales` four times (an `Append` over four `Seq Scan`s). For a big table that's 4× the I/O.

**Method B — `LATERAL` + `VALUES` (one scan, PostgreSQL-idiomatic):**

```sql
SELECT r.region, v.quarter, v.amount
FROM regional_sales r
CROSS JOIN LATERAL (VALUES
  ('q1', r.q1),
  ('q2', r.q2),
  ('q3', r.q3),
  ('q4', r.q4)
) AS v(quarter, amount);
```

The `LATERAL` subquery references `r`'s columns and emits four rows per input row in a **single scan**. This is the preferred unpivot in PostgreSQL — one pass, readable, and it keeps the column-to-label mapping explicit.

**Method C — `unnest` of parallel arrays (compact):**

```sql
SELECT r.region, u.quarter, u.amount
FROM regional_sales r
CROSS JOIN LATERAL unnest(
  ARRAY['q1','q2','q3','q4'],
  ARRAY[r.q1, r.q2, r.q3, r.q4]
) AS u(quarter, amount);
```

`unnest` with multiple arrays zips them in parallel (position 1 with position 1, etc.). Also one scan. Handy when the labels and values line up cleanly.

**NULL handling in unpivot:** all three methods produce a row even when the source cell is NULL. If a NULL cell means "no data, drop it," add `WHERE amount IS NOT NULL` to the outer query. That's usually what you want when melting a sparse wide table.

### 6.8 The fan-out trap inside a pivot

Because a pivot is a `GROUP BY`, joining a one-to-many table before pivoting multiplies the rows that feed each cell:

```sql
-- WRONG: orders joined to order_items fans out; SUM(o.amount) double-counts
SELECT
  o.region,
  SUM(o.amount) FILTER (WHERE p.category = 'hardware') AS hardware
FROM orders o
JOIN order_items oi ON oi.order_id = o.id     -- fan-out: one order → many items
JOIN products p     ON p.id = oi.product_id
GROUP BY o.region;
-- o.amount is repeated once per item → the hardware cell is inflated
```

The cell looks like a number; it's just the wrong number. Fix by aggregating at the correct grain — pivot the item-level measure, not the order-level one, or pre-aggregate orders before joining. This is the Topic 11 fan-out bug wearing a pivot costume.

### 6.9 GROUPING SETS / ROLLUP — the adjacent tool

Pivot spreads a dimension *across*. `GROUPING SETS`, `ROLLUP`, and `CUBE` (Topic 22) add *subtotal rows down*. They're complementary and often combined: pivot the columns, then `ROLLUP` the row key to get per-row totals and a grand total.

```sql
SELECT
  COALESCE(region, 'ALL') AS region,
  SUM(amount) FILTER (WHERE category = 'hardware') AS hardware,
  SUM(amount) FILTER (WHERE category = 'software') AS software,
  SUM(amount)                                       AS total
FROM sales
GROUP BY ROLLUP(region)   -- adds a grand-total row where region IS NULL
ORDER BY region NULLS LAST;
```

A common polished report is "pivot across categories, `ROLLUP` down regions, plus a `total` column" — combining both operations in one query.

### 6.10 Type consistency across pivot columns

Every cell in a given column shares one type (fixed at plan time). If you pivot a `text` measure into some columns and a numeric into others, that's fine — different columns can have different types. But within one column, the aggregate must yield a single type. Mixing `SUM(int)` and `SUM(numeric)` conditionally in the same column expression forces a common type (numeric), which is usually fine but worth knowing when a column comes back `numeric` instead of `integer`.

---

## 7. EXPLAIN — Pivot Patterns in the Plan

### 7.1 The FILTER/CASE pivot — it's just a HashAggregate

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT
  region,
  SUM(amount) FILTER (WHERE category = 'hardware') AS hardware,
  SUM(amount) FILTER (WHERE category = 'software') AS software,
  SUM(amount) FILTER (WHERE category = 'services') AS services
FROM sales
GROUP BY region;
```

```
HashAggregate  (cost=20834.00..20834.06 rows=5 width=100)
               (actual time=182.4..182.5 rows=5 loops=1)
  Group Key: region
  Batches: 1  Memory Usage: 24kB
  ->  Seq Scan on sales  (cost=0.00..15834.00 rows=1000000 width=40)
                          (actual time=0.02..61.3 rows=1000000 loops=1)
  Buffers: shared hit=5834
Planning Time: 0.14 ms
Execution Time: 182.6 ms
```

**Reading it:**
- **`HashAggregate`** — there is no "pivot node." The three `FILTER` aggregates are three transition functions maintained per group. The pivot is entirely inside this one node.
- **`Group Key: region`** — the row key. Only 5 groups → tiny hash table.
- **`Batches: 1  Memory Usage: 24kB`** — the aggregate state fit in `work_mem`, no spill. Note the memory is driven by the **5 groups**, not by the three columns.
- **`Seq Scan ... rows=1000000`** — every long row is read once and fanned into the aggregates. This single scan feeds all three pivot columns simultaneously — a key efficiency of the `FILTER` pattern: **one pass, N columns.**
- `width=100` on the aggregate — wider output row because it carries three sums plus the key.

### 7.2 What a spilling pivot looks like (many output rows)

```sql
EXPLAIN (ANALYZE)
SELECT
  user_id,
  COUNT(*) FILTER (WHERE event = 'click') AS clicks,
  COUNT(*) FILTER (WHERE event = 'view')  AS views
FROM events            -- 50M rows, 8M distinct users
GROUP BY user_id;
```

```
HashAggregate  (cost=1150000..1400000 rows=8000000 width=24)
               (actual time=9500..14200 rows=8000000 loops=1)
  Group Key: user_id
  Planned Partitions: 32  Batches: 33  Memory Usage: 262144kB  Disk Usage: 1835008kB
  ->  Seq Scan on events  (cost=0.00..800000 rows=50000000 width=16)
                           (actual rows=50000000 loops=1)
Execution Time: 14850 ms
```

**Reading it:**
- **`Batches: 33` and `Disk Usage: 1835008kB`** — the 8M-group hash table blew past `work_mem` and **spilled to disk** (PostgreSQL 13+ hash-aggregate spill). This is the real memory risk of pivots: **many output rows, not many columns.**
- Fix options: raise `work_mem` for this query, pre-filter to fewer users, or pre-aggregate. The two pivot columns are irrelevant to the spill; the 8M `user_id` groups cause it.

### 7.3 crosstab() in the plan — a Function Scan

```sql
EXPLAIN (ANALYZE)
SELECT * FROM crosstab(
  $$ SELECT day, product, qty FROM sales ORDER BY 1,2 $$,
  $$ SELECT DISTINCT product FROM sales ORDER BY 1 $$
) AS ct(day text, espresso int, latte int, mocha int);
```

```
Function Scan on crosstab ct  (cost=0.25..10.25 rows=1000 width=44)
                              (actual time=210.7..210.9 rows=365 loops=1)
Planning Time: 0.05 ms
Execution Time: 211.4 ms
```

**Reading it:**
- **`Function Scan on crosstab`** — to the planner, `crosstab` is an opaque set-returning function. It sees `rows=1000` (a hard-coded SRF default estimate, **not** based on real statistics) and `width=44` from your `AS` list.
- All the real work (running the source query, ordering, grouping, filling cells) happens *inside* the C function and is invisible to `EXPLAIN`. You cannot see the source query's own plan here — run `EXPLAIN` on the inner query separately to tune it.
- The flat `rows=1000` estimate is a planner blind spot: if you `JOIN` this crosstab to another table, the planner may misestimate and pick a poor join. Materialize into a temp table with real stats if that bites.

### 7.4 LATERAL unpivot — ProjectSet / nested loop

```sql
EXPLAIN (ANALYZE)
SELECT r.region, v.quarter, v.amount
FROM regional_sales r
CROSS JOIN LATERAL (VALUES ('q1', r.q1), ('q2', r.q2),
                           ('q3', r.q3), ('q4', r.q4)) AS v(quarter, amount);
```

```
Nested Loop  (cost=0.00..35.00 rows=4000 width=40)
             (actual time=0.01..0.9 rows=4000 loops=1)
  ->  Seq Scan on regional_sales r  (cost=0.00..15.00 rows=1000 width=20)
                                     (actual rows=1000 loops=1)
  ->  Values Scan on "*VALUES*"  (cost=0.00..0.05 rows=4 width=36)
                                  (actual rows=4 loops=1000)
Execution Time: 1.2 ms
```

**Reading it:**
- **`Nested Loop`** with the base table as outer (1000 rows) and a **`Values Scan`** as the lateral inner, `loops=1000` — four melted rows per input row, computed in one scan of `regional_sales`.
- Contrast the `UNION ALL` unpivot, which would show an **`Append`** over four separate `Seq Scan on regional_sales` — 4× the base-table I/O for the same 4000 output rows.

---

## 8. Query Examples

### Example 1 — Basic: Monthly signups pivoted by plan

```sql
-- Long → wide: one row per month, one column per subscription plan.
-- Categories (plans) are a small, stable, known set → just enumerate them.
SELECT
  to_char(u.created_at, 'YYYY-MM')                                  AS signup_month,
  COUNT(*) FILTER (WHERE u.plan = 'free')                            AS free,
  COUNT(*) FILTER (WHERE u.plan = 'pro')                             AS pro,
  COUNT(*) FILTER (WHERE u.plan = 'enterprise')                      AS enterprise,
  COUNT(*)                                                           AS total
FROM users u
WHERE u.created_at >= DATE_TRUNC('year', CURRENT_DATE)
GROUP BY to_char(u.created_at, 'YYYY-MM')
ORDER BY signup_month;
-- COUNT returns 0 (not NULL) for empty cells — no COALESCE needed here.
```

### Example 2 — Intermediate: Revenue matrix, categories across, months down, with total

```sql
-- Two-dimensional pivot: month down the side, product category across the top,
-- revenue in the cells, plus a per-month total column.
-- COALESCE turns empty SUM cells (NULL) into 0 for a clean grid.
SELECT
  to_char(o.created_at, 'YYYY-MM')                                          AS month,
  COALESCE(SUM(oi.quantity * oi.unit_price)
           FILTER (WHERE c.name = 'Hardware'), 0)                           AS hardware,
  COALESCE(SUM(oi.quantity * oi.unit_price)
           FILTER (WHERE c.name = 'Software'), 0)                           AS software,
  COALESCE(SUM(oi.quantity * oi.unit_price)
           FILTER (WHERE c.name = 'Services'), 0)                           AS services,
  SUM(oi.quantity * oi.unit_price)                                          AS total
FROM orders o
JOIN order_items oi ON oi.order_id = o.id       -- item grain: measure lives here, no fan-out bug
JOIN products p     ON p.id = oi.product_id
JOIN categories c   ON c.id = p.category_id
WHERE o.status = 'completed'
  AND o.created_at >= CURRENT_DATE - INTERVAL '12 months'
GROUP BY to_char(o.created_at, 'YYYY-MM')
ORDER BY month;
-- The measure (quantity * unit_price) is at the order_items grain, so joining
-- order_items does NOT double-count — the SUM is over genuine line items.
```

### Example 3 — Production Grade: Cohort retention matrix with crosstab()

```sql
-- Retention grid: rows = signup cohort (month), columns = months-since-signup (0..5),
-- cell = % of the cohort still active that month.
-- Table sizes: users ~2M rows, sessions ~400M rows (partitioned by month).
-- Indexes assumed: sessions(user_id, created_at), users(created_at).
-- Perf expectation: heavy aggregation over sessions; runs as a scheduled job,
-- materialized into a reporting table — not a live request-path query.
CREATE EXTENSION IF NOT EXISTS tablefunc;

WITH cohort AS (
  SELECT id AS user_id, DATE_TRUNC('month', created_at) AS cohort_month
  FROM users
  WHERE created_at >= CURRENT_DATE - INTERVAL '6 months'
),
activity AS (
  -- one row per (cohort_month, months_since, user) that was active
  SELECT
    c.cohort_month,
    (EXTRACT(YEAR  FROM age(DATE_TRUNC('month', s.created_at), c.cohort_month)) * 12
   + EXTRACT(MONTH FROM age(DATE_TRUNC('month', s.created_at), c.cohort_month)))::int
                                                            AS months_since,
    c.user_id
  FROM cohort c
  JOIN sessions s ON s.user_id = c.user_id
  WHERE s.created_at >= c.cohort_month
  GROUP BY c.cohort_month, 2, c.user_id
),
retention AS (
  SELECT
    to_char(cohort_month, 'YYYY-MM')                        AS cohort,
    months_since,
    COUNT(DISTINCT user_id)                                 AS active_users
  FROM activity
  WHERE months_since BETWEEN 0 AND 5
  GROUP BY 1, 2
)
SELECT * FROM crosstab(
  $$ SELECT cohort, months_since, active_users
     FROM retention ORDER BY 1, 2 $$,                        -- MUST be ordered by cohort, months_since
  $$ SELECT generate_series(0, 5) $$                         -- fixed category list: months 0..5
) AS ct(
  cohort text,
  m0 int, m1 int, m2 int, m3 int, m4 int, m5 int            -- 1 row-name + 6 category columns
)
ORDER BY cohort;
```

```sql
-- EXPLAIN (ANALYZE) of the crosstab call (outer plan only — inner query is opaque):
```
```
Function Scan on crosstab ct  (cost=0.25..10.25 rows=1000 width=56)
                              (actual time=4120.5..4120.8 rows=6 loops=1)
Planning Time: 0.06 ms
Execution Time: 4121.0 ms
```

The 4.1s cost lives inside the source query (the `activity`/`retention` CTEs scanning 400M sessions), invisible to this outer `EXPLAIN` — which is exactly why this runs as a nightly materialization, not on the request path. To tune it, `EXPLAIN` the `retention` query on its own.

---

## 9. Wrong → Right Patterns

### Wrong 1: The one-argument crosstab() alignment corruption

```sql
-- WRONG: one-argument crosstab fills columns by APPEARANCE ORDER, not by name
SELECT * FROM crosstab(
  $$ SELECT day, product, qty FROM sales ORDER BY 1, 2 $$
) AS ct(day text, espresso int, latte int, mocha int);
```

**The exact wrong result:** suppose Monday sold all three products but Tuesday sold only `latte` and `mocha` (no espresso). The one-arg form fills each row's columns left-to-right from whatever categories that row has:

```
day       | espresso | latte | mocha
----------+----------+-------+------
Monday    |    3     |   5   |   2     ← correct
Tuesday   |    4     |   2   |  NULL   ← WRONG: latte(4) landed in "espresso",
                                          mocha(2) landed in "latte", espresso is NULL
```

**Why it's wrong at the execution level:** the one-argument C function has no category list to match against. It emits the k-th category it encounters into the k-th column. Because Tuesday's *first* category is `latte`, `latte` goes to column 1 (`espresso`). The grid is silently transposed for any row missing an early category. No error is raised — the numbers are just in the wrong cells.

```sql
-- RIGHT: two-argument form matches by category value, NULLs the gaps
SELECT * FROM crosstab(
  $$ SELECT day, product, qty FROM sales ORDER BY 1, 2 $$,
  $$ SELECT DISTINCT product FROM sales ORDER BY 1 $$   -- category list to match against
) AS ct(day text, espresso int, latte int, mocha int);
-- Now Tuesday.espresso correctly = NULL, latte = 4, mocha = 2.
```

### Wrong 2: Fan-out double-counting the pivot cells

```sql
-- WRONG: SUM(o.total_amount) at ORDER grain, but joined to order_items (item grain)
SELECT
  o.region,
  SUM(o.total_amount) FILTER (WHERE c.name = 'Hardware') AS hardware_rev
FROM orders o
JOIN order_items oi ON oi.order_id = o.id       -- fan-out: 1 order → many items
JOIN products p     ON p.id = oi.product_id
JOIN categories c   ON c.id = p.category_id
GROUP BY o.region;
```

**The exact wrong result:** an order of `$100` with 4 hardware line items contributes `$100` **four times** → `$400` in the hardware cell. Revenue is inflated by the average items-per-order factor. The grid looks totally normal — every cell has a plausible dollar amount — which is what makes it dangerous.

**Why it's wrong at the execution level:** the join expands each order row into one row per line item *before* the `GROUP BY` aggregates. `o.total_amount` is repeated on each expanded row, so `SUM` adds it once per item. The pivot didn't cause the bug; the fan-out did — but the pivot hid it behind a clean-looking matrix.

```sql
-- RIGHT: aggregate the measure at its true grain (line items)
SELECT
  o.region,
  SUM(oi.quantity * oi.unit_price)
    FILTER (WHERE c.name = 'Hardware') AS hardware_rev
FROM orders o
JOIN order_items oi ON oi.order_id = o.id
JOIN products p     ON p.id = oi.product_id
JOIN categories c   ON c.id = p.category_id
GROUP BY o.region;
-- The measure now lives at the item grain, so each line item is summed exactly once.
```

### Wrong 3: NULL cells where the report needs zeros

```sql
-- WRONG: empty SUM cells come back NULL, breaking downstream math and display
SELECT
  region,
  SUM(amount) FILTER (WHERE category = 'hardware') AS hardware,
  SUM(amount) FILTER (WHERE category = 'software') AS software
FROM sales
GROUP BY region;
-- A region that sold no software shows software = NULL, not 0.
```

**The exact wrong result:** `hardware + software` for that region is `NULL` (NULL propagates through arithmetic), a totals row using `SUM(software)` skips it, and a CSV export shows an empty cell that a spreadsheet may misread. The "grid with holes" problem.

**Why it's wrong at the execution level:** `SUM` over zero matching rows returns `NULL` by definition (not `0`) — the aggregate never fired its transition function for that cell, so it stays at its initial `NULL` state.

```sql
-- RIGHT: COALESCE every SUM/MAX/MIN pivot cell to 0 (or the appropriate default)
SELECT
  region,
  COALESCE(SUM(amount) FILTER (WHERE category = 'hardware'), 0) AS hardware,
  COALESCE(SUM(amount) FILTER (WHERE category = 'software'), 0) AS software
FROM sales
GROUP BY region;
-- COUNT-based pivots don't need this (COUNT returns 0), but SUM/MAX/MIN/AVG do.
```

### Wrong 4: Expecting the pivot to auto-add columns for new categories

```sql
-- WRONG assumption: "this query will grow a column when we add a new plan"
SELECT
  to_char(created_at, 'YYYY-MM') AS month,
  COUNT(*) FILTER (WHERE plan = 'free') AS free,
  COUNT(*) FILTER (WHERE plan = 'pro')  AS pro
FROM users
GROUP BY 1;
-- Business adds an 'enterprise' plan. Those users are simply ABSENT from the grid —
-- silently uncounted. No error. The report under-reports total signups forever.
```

**Why it's wrong at the execution level:** the output `TupleDesc` (columns `month, free, pro`) was fixed at plan time. Enterprise rows match neither `FILTER`, so they contribute to no cell and vanish from the pivot (though they'd still appear in an unfiltered `COUNT(*)`). SQL cannot invent an `enterprise` column at runtime.

```sql
-- RIGHT (option A): keep a catch-all + a grand total so nothing is silently lost
SELECT
  to_char(created_at, 'YYYY-MM') AS month,
  COUNT(*) FILTER (WHERE plan = 'free')                     AS free,
  COUNT(*) FILTER (WHERE plan = 'pro')                      AS pro,
  COUNT(*) FILTER (WHERE plan NOT IN ('free','pro'))        AS other,  -- catches new plans
  COUNT(*)                                                  AS total
FROM users
GROUP BY 1;

-- RIGHT (option B): return a document so the shape follows the data
SELECT
  to_char(created_at, 'YYYY-MM')          AS month,
  jsonb_object_agg(plan, cnt)             AS by_plan     -- {"free":10,"pro":4,"enterprise":2}
FROM (
  SELECT created_at, plan, COUNT(*) AS cnt
  FROM users GROUP BY created_at, plan
) s
GROUP BY 1;
-- The app renders whatever keys exist; new plans appear automatically.
```

### Wrong 5: Unpivot with UNION ALL scanning the table N times

```sql
-- WRONG (not incorrect, but wasteful): 4 full scans of a large wide table
SELECT region, 'q1' AS quarter, q1 AS amount FROM regional_sales
UNION ALL SELECT region, 'q2', q2 FROM regional_sales
UNION ALL SELECT region, 'q3', q3 FROM regional_sales
UNION ALL SELECT region, 'q4', q4 FROM regional_sales;
-- EXPLAIN shows Append over FOUR Seq Scans on regional_sales → 4x the I/O.
```

**Why it's wasteful at the execution level:** each `UNION ALL` branch is an independent scan. For a 10M-row table that's 40M rows read to produce 40M output rows — but from four passes instead of one. On a wide table with a large row width, that's four times the buffer traffic.

```sql
-- RIGHT: one scan via LATERAL VALUES (Nested Loop over a 4-row Values Scan)
SELECT r.region, v.quarter, v.amount
FROM regional_sales r
CROSS JOIN LATERAL (VALUES
  ('q1', r.q1), ('q2', r.q2), ('q3', r.q3), ('q4', r.q4)
) AS v(quarter, amount)
WHERE v.amount IS NOT NULL;   -- optional: drop empty cells while melting
-- Single Seq Scan on regional_sales, four melted rows per input row.
```

---

## 10. Performance Profile

### Cost model of the FILTER/CASE pivot

A `FILTER` pivot is **one aggregation over one scan**, regardless of how many pivot columns you add. Cost breaks down as:

- **Scan cost:** read every long-format input row once (Seq Scan or Index Scan). This dominates for large inputs.
- **Per-row aggregate cost:** for each row, evaluate N predicates (one per pivot column) and fire the matching aggregate's transition function. N columns = N predicate evaluations per row. This is CPU, usually small next to I/O.
- **Grouping cost:** build a HashAggregate keyed by the row key — memory proportional to the **number of output rows (groups)**, not the number of columns.

| Scale | Input rows | Output rows (groups) | Typical behavior |
|-------|-----------|---------------------|------------------|
| Small | 1M | 12 (months) | HashAggregate in `work_mem`, single scan, ~100–300ms. Column count irrelevant. |
| Medium | 10M | 10K (e.g. per-product) | Still fits `work_mem` (10K small rows). ~1–3s, scan-bound. |
| Large | 50M | 8M (per-user) | Hash table spills to disk (`Batches > 1`). 10–20s. **The output-row count, not columns, causes the spill.** |
| Very wide | 5M | 100 | 50 pivot columns × 5M rows = 250M predicate evals, but only 100 groups. CPU-heavier per row; consider `crosstab` (single C-level pass). |

### The key scaling insight

**Pivot memory scales with output rows, not output columns.** Ten pivot columns over 100 groups is trivial. Two pivot columns over 8M groups spills to disk. When a pivot is slow or memory-hungry, the fix is almost always to **reduce the number of groups** (pre-filter with `WHERE`, or coarsen the row key), or to raise `work_mem`.

### Index interactions

- **Pre-aggregation filter:** an index supporting the `WHERE` (e.g. `sales(created_at)` for a date-range pivot) lets the scan skip irrelevant rows before they fan into groups — the single biggest win.
- **GroupAggregate via index:** if an index provides the rows already sorted by the row key, the planner can use a **GroupAggregate** (streaming, O(1) extra memory) instead of a **HashAggregate**, avoiding spill entirely for high-cardinality row keys. `CREATE INDEX ON events(user_id)` can turn the spilling 8M-group pivot above into a streaming sort-based aggregate. Verify with `EXPLAIN` (`GroupAggregate` + `Index Scan` vs `HashAggregate` + `Seq Scan`).
- **No index helps the pivot columns themselves** — the `FILTER` predicates are evaluated per row during aggregation, not via index lookup.

### crosstab() vs FILTER performance

- `crosstab` reads the (already ordered) source **once** in C, doing the grouping and cell-filling without evaluating N SQL predicates per row. For **many categories** (dozens) over large inputs, this CPU saving can matter.
- But `crosstab` **requires the source pre-sorted** by (row_name, category), so it pays a Sort (or needs a matching index). The `FILTER` pattern uses an unordered HashAggregate — often cheaper overall for modest category counts.
- Rule of thumb: `FILTER` for ≤ ~15 categories or when the extension isn't available; consider `crosstab` for wide, high-category pivots where the source is cheaply sortable.

### Unpivot performance

- **`LATERAL`/`unnest`:** one scan, N output rows per input row. Cost ≈ scan + N× projection. Preferred.
- **`UNION ALL`:** N scans. Cost ≈ N × scan. For a wide 10M-row table this is meaningfully worse on I/O.
- Add `WHERE value IS NOT NULL` to drop empty cells cheaply during the melt rather than in a later pass.

### Optimization checklist specific to pivots

1. **Filter before you pivot.** `WHERE` shrinks the input; every dropped row is one fewer to fan into aggregates.
2. **Watch group cardinality.** Millions of output rows = memory pressure. Raise `work_mem` for the session or coarsen the row key.
3. **Prefer an index that pre-sorts the row key** for high-cardinality pivots → GroupAggregate, no spill.
4. **Materialize expensive report pivots** (cohort/retention over hundreds of millions of rows) into a table on a schedule; don't run them on the request path.
5. **Return JSONB for dynamic categories** instead of round-tripping to build dynamic SQL — fewer moving parts, no injection surface.

---

## 11. Node.js Integration

### 11.1 Fixed-column pivot with `pool.query`

```javascript
import pg from 'pg';
const { Pool } = pg;
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

// Known, stable categories → enumerate them. Params bind the date range only.
async function signupsByPlan(fromDate) {
  const { rows } = await pool.query(
    `SELECT
       to_char(created_at, 'YYYY-MM')                    AS month,
       COUNT(*) FILTER (WHERE plan = 'free')             AS free,
       COUNT(*) FILTER (WHERE plan = 'pro')              AS pro,
       COUNT(*) FILTER (WHERE plan = 'enterprise')       AS enterprise,
       COUNT(*)                                          AS total
     FROM users
     WHERE created_at >= $1
     GROUP BY 1
     ORDER BY 1`,
    [fromDate]
  );
  return rows;   // [{ month: '2026-01', free: '120', pro: '30', ... }]
  // NOTE: COUNT/SUM come back as strings (pg maps bigint/numeric to string to avoid
  // precision loss). Convert with Number() only when you know values fit in a JS number.
}
```

### 11.2 Safe dynamic pivot — enumerate categories, then build SQL with an allowlist

```javascript
// When categories are NOT known ahead of time but you still want real columns.
// CRITICAL: never interpolate a raw category into SQL. Validate against the actual
// distinct values from the DB, and quote identifiers safely.

async function dynamicPivotByProduct() {
  // 1) Discover the categories (parameterized, safe)
  const { rows: cats } = await pool.query(
    `SELECT DISTINCT product FROM sales ORDER BY product`
  );
  const products = cats.map((r) => r.product);

  // 2) Build the SELECT list. Values go into FILTER as bound params is not possible
  //    inside identifiers, so we use pg-format style quoting for BOTH the literal
  //    (predicate value) and the identifier (column alias).
  //    Here we hand-quote defensively; in real code use the `pg-format` package.
  const quoteLiteral = (s) => `'${String(s).replace(/'/g, "''")}'`;
  const quoteIdent   = (s) => `"${String(s).replace(/"/g, '""')}"`;

  const cols = products
    .map(
      (p) =>
        `COALESCE(SUM(qty) FILTER (WHERE product = ${quoteLiteral(p)}), 0) AS ${quoteIdent(p)}`
    )
    .join(',\n       ');

  const sql = `
    SELECT day,
       ${cols}
    FROM sales
    GROUP BY day
    ORDER BY day`;

  const { rows } = await pool.query(sql);
  return { columns: ['day', ...products], rows };
}
// The doubled-quote escaping is the minimum bar; prefer the `pg-format` library's
// %L (literal) and %I (identifier) which mirror PostgreSQL's format() exactly.
```

### 11.3 The usually-better answer: aggregate to JSONB, pivot in JS

```javascript
// For truly dynamic categories, let Postgres return one JSONB column and
// reshape in the application. No dynamic SQL, no injection surface, fixed column list.
async function signupsByPlanDynamic(fromDate) {
  const { rows } = await pool.query(
    `SELECT
       to_char(created_at, 'YYYY-MM') AS month,
       jsonb_object_agg(plan, cnt)    AS by_plan
     FROM (
       SELECT created_at, plan, COUNT(*)::int AS cnt
       FROM users
       WHERE created_at >= $1
       GROUP BY created_at, plan
     ) s
     GROUP BY 1
     ORDER BY 1`,
    [fromDate]
  );
  // rows: [{ month: '2026-01', by_plan: { free: 120, pro: 30, enterprise: 5 } }]
  // Collect the union of all keys to render a stable table in the UI:
  const allPlans = [...new Set(rows.flatMap((r) => Object.keys(r.by_plan)))].sort();
  return { plans: allPlans, rows };
}
```

### 11.4 Consuming a crosstab result

```javascript
// crosstab returns fixed columns you declared in AS ct(...); consume like any query.
async function retentionMatrix() {
  const { rows } = await pool.query(
    `SELECT * FROM crosstab(
       $ct$ SELECT cohort, months_since, active_users
            FROM retention ORDER BY 1, 2 $ct$,
       $ct$ SELECT generate_series(0,5) $ct$
     ) AS ct(cohort text, m0 int, m1 int, m2 int, m3 int, m4 int, m5 int)
     ORDER BY cohort`
  );
  return rows;   // one object per cohort with keys m0..m5
}
// Use dollar-quote tags ($ct$...$ct$) for the inner query strings so they don't
// collide with JS template literals or with $1-style placeholders.
```

**Do ORMs handle pivots?** No ORM has a first-class pivot API. The `FILTER`/`CASE` expressions live in the `SELECT` list as raw fragments in every ORM, and `crosstab` always needs raw SQL. Treat pivots as a raw-SQL concern in all Node ORMs.

---

## 12. ORM Comparison

Pivoting is a place where every ORM leans on its raw-SQL escape hatch. None model "spread values into columns" declaratively — the column set is fixed at query-build time, which conflicts with ORMs' row-oriented result mapping.

### Prisma

**Can Prisma pivot?** — Not with the fluent API. No `findMany` shape expresses conditional aggregates as columns. You use `$queryRaw`.

```typescript
import { PrismaClient, Prisma } from '@prisma/client';
const prisma = new PrismaClient();

// Fixed-category pivot via $queryRaw (tagged template = parameterized, injection-safe)
const rows = await prisma.$queryRaw<Array<{
  month: string; free: number; pro: number; enterprise: number;
}>>`
  SELECT
    to_char(created_at, 'YYYY-MM')              AS month,
    COUNT(*) FILTER (WHERE plan = 'free')        AS free,
    COUNT(*) FILTER (WHERE plan = 'pro')         AS pro,
    COUNT(*) FILTER (WHERE plan = 'enterprise')  AS enterprise
  FROM users
  WHERE created_at >= ${fromDate}
  GROUP BY 1
  ORDER BY 1
`;

// Dynamic column names cannot go through ${} (that only parameterizes VALUES, not
// identifiers). For dynamic pivots use Prisma.raw() with a validated allowlist, or
// aggregate to JSON:
const dyn = await prisma.$queryRaw<Array<{ month: string; by_plan: Record<string, number> }>>`
  SELECT to_char(created_at,'YYYY-MM') AS month, jsonb_object_agg(plan, cnt) AS by_plan
  FROM (SELECT created_at, plan, COUNT(*)::int AS cnt FROM users GROUP BY created_at, plan) s
  GROUP BY 1`;
```

**Where it breaks:** `Prisma.$queryRaw`'s `${}` interpolation binds **values only** — it cannot inject a dynamic column name or a `FILTER` predicate identifier. Dynamic pivots require `Prisma.raw()` (unsafe unless you allowlist) or the JSON approach. Also, `COUNT`/`SUM` return `BigInt`/`Decimal`; cast with `::int`/`::float` in SQL or handle the types in TS.

**Verdict:** Use `$queryRaw` for fixed pivots; JSON-aggregate for dynamic ones. Never build pivot columns by string-concatenating user input.

---

### Drizzle ORM

**Can Drizzle pivot?** — Yes, using the `sql` template for the `FILTER`/`CASE` expressions inside `.select()`.

```typescript
import { db } from './db';
import { users } from './schema';
import { sql, gte } from 'drizzle-orm';

const rows = await db
  .select({
    month: sql<string>`to_char(${users.createdAt}, 'YYYY-MM')`,
    free:  sql<number>`count(*) filter (where ${users.plan} = 'free')`,
    pro:   sql<number>`count(*) filter (where ${users.plan} = 'pro')`,
    enterprise: sql<number>`count(*) filter (where ${users.plan} = 'enterprise')`,
  })
  .from(users)
  .where(gte(users.createdAt, fromDate))
  .groupBy(sql`to_char(${users.createdAt}, 'YYYY-MM')`)
  .orderBy(sql`1`);
```

**Where it breaks:** column *keys* in the `.select({...})` object are static TypeScript identifiers — you cannot generate them from data without dropping to a fully raw `db.execute(sql.raw(...))`. Drizzle interpolates column *references* safely but treats string literals in the `sql` tag as raw text, so validate any dynamic category.

**Verdict:** Best typed experience for fixed pivots — the `sql` tag reads close to the SQL. Dynamic pivots still need `sql.raw` with an allowlist or JSON aggregation.

---

### Sequelize

**Can Sequelize pivot?** — Only via `sequelize.literal()` for each conditional aggregate, which is verbose, or via a raw query. Raw is cleaner.

```javascript
const { QueryTypes } = require('sequelize');

// Recommended: raw query with replacements (named binds are for VALUES, not identifiers)
const rows = await sequelize.query(
  `SELECT
     to_char(created_at, 'YYYY-MM')              AS month,
     COUNT(*) FILTER (WHERE plan = 'free')        AS free,
     COUNT(*) FILTER (WHERE plan = 'pro')         AS pro,
     COUNT(*) FILTER (WHERE plan = 'enterprise')  AS enterprise
   FROM users
   WHERE created_at >= :fromDate
   GROUP BY 1
   ORDER BY 1`,
  { replacements: { fromDate }, type: QueryTypes.SELECT }
);

// findAll form (awkward): each column is a literal in attributes
const model = await User.findAll({
  attributes: [
    [sequelize.fn('to_char', sequelize.col('created_at'), 'YYYY-MM'), 'month'],
    [sequelize.literal(`COUNT(*) FILTER (WHERE plan = 'free')`), 'free'],
    [sequelize.literal(`COUNT(*) FILTER (WHERE plan = 'pro')`),  'pro'],
  ],
  group: [sequelize.literal(`to_char(created_at, 'YYYY-MM')`)],
  raw: true,
});
```

**Where it breaks:** `sequelize.literal()` content is **raw, unescaped SQL** — interpolating a category value there is an injection hole. `replacements` bind values but not identifiers/column names. Dynamic pivots must validate categories against a DB-fetched allowlist.

**Verdict:** Use `sequelize.query()` with `replacements` for pivots. The `literal()` route works but is noisy and easy to make unsafe.

---

### TypeORM

**Can TypeORM pivot?** — Yes, via `QueryBuilder.addSelect()` with raw expressions, consumed through `getRawMany()`.

```typescript
const rows = await dataSource
  .createQueryBuilder()
  .select(`to_char(created_at, 'YYYY-MM')`, 'month')
  .addSelect(`COUNT(*) FILTER (WHERE plan = 'free')`, 'free')
  .addSelect(`COUNT(*) FILTER (WHERE plan = 'pro')`, 'pro')
  .addSelect(`COUNT(*) FILTER (WHERE plan = 'enterprise')`, 'enterprise')
  .from('users', 'u')
  .where('created_at >= :fromDate', { fromDate })
  .groupBy('1')
  .orderBy('1')
  .getRawMany();   // must use getRawMany — pivot columns aren't entity fields
```

**Where it breaks:** the pivot columns aren't entity properties, so `getMany()` won't map them — you must use `getRawMany()` and lose entity typing. The raw expression strings in `addSelect` have no type safety and no escaping; parameters (`:fromDate`) bind values only. Dynamic columns require building the `addSelect` calls in a loop over a validated category list.

**Verdict:** QueryBuilder + `getRawMany()` is workable for fixed pivots. Accept that results are plain objects, not entities.

---

### Knex.js

**Can Knex pivot?** — Yes, most transparently. `knex.raw()` for each conditional aggregate; building columns in a loop is natural.

```javascript
// Fixed pivot
const rows = await knex('users')
  .select(knex.raw(`to_char(created_at, 'YYYY-MM') AS month`))
  .select(knex.raw(`COUNT(*) FILTER (WHERE plan = 'free') AS free`))
  .select(knex.raw(`COUNT(*) FILTER (WHERE plan = 'pro') AS pro`))
  .where('created_at', '>=', fromDate)
  .groupByRaw('1')
  .orderByRaw('1');

// Dynamic pivot — Knex makes the loop clean, but YOU must guarantee safety.
// Use bindings (?) for the literal value; validate the identifier for the alias.
async function dynamicPivot() {
  const cats = (await knex('sales').distinct('product').orderBy('product'))
    .map((r) => r.product);

  const qb = knex('sales').select('day').groupBy('day').orderBy('day');
  for (const p of cats) {
    if (!/^[\w -]+$/.test(p)) throw new Error(`unsafe category: ${p}`); // allowlist the alias
    // ?? = identifier binding (safe), ? = value binding (safe)
    qb.select(knex.raw('COALESCE(SUM(qty) FILTER (WHERE product = ?), 0) AS ??', [p, p]));
  }
  return qb;
}
```

**Where it breaks:** `knex.raw` with `?` (value) and `??` (identifier) bindings is the safest builder-level pivot in Node — but the identifier binding still needs a sane category, so keep the allowlist check. Non-trivial nesting (crosstab) drops to fully raw strings.

**Verdict:** Knex is the best fit for **dynamic** pivots in Node — the `?`/`??` bindings plus loop-built `select` calls give safe, readable dynamic column generation.

---

### ORM Summary Table

| ORM | Pivot method | Dynamic columns | Injection-safe path | Verdict |
|-----|-------------|-----------------|--------------------|---------|
| Prisma | `$queryRaw` (FILTER/CASE) | `Prisma.raw()` + allowlist, or JSON | `${}` binds values only | Raw for fixed; JSON for dynamic |
| Drizzle | `sql` template in `.select()` | `sql.raw` + allowlist | Column refs interpolated safely | Best typed fixed pivot |
| Sequelize | `query()` / `literal()` | Validate + literal | `replacements` (values only) | Use raw `query()`; literal is unsafe if careless |
| TypeORM | `addSelect` + `getRawMany` | Loop `addSelect`, validate | `:param` binds values only | Workable; loses entity mapping |
| Knex | `knex.raw` `?`/`??` bindings | Loop + `??` identifier binding | `?` value, `??` identifier | Best for dynamic pivots |

Across all five: **no fluent pivot API, values-only parameter binding, and dynamic column names always require a validated allowlist or JSON aggregation.** The JSONB-aggregate approach sidesteps every ORM's limitation at once.

---

## 13. Practice Exercises

### Exercise 1 — Basic

Given `orders(id, customer_id, status, created_at, total_amount)`, write a pivot that returns one row per `status` (`pending`, `paid`, `shipped`, `cancelled`) is **not** what we want — instead return **one row per month** (`YYYY-MM`) with one column per status holding the **count of orders** in that status, plus a `total` column. Empty cells should read `0`, not `NULL`. Restrict to the last 6 months.

```sql
-- Write your query here
```

---

### Exercise 2 — Medium (combines Topics 11, 20, and this topic)

Given `orders(id, customer_id, status, created_at)`, `order_items(id, order_id, product_id, quantity, unit_price)`, `products(id, name, category_id)`, `categories(id, name)`:

Produce a matrix with **one row per customer** and **one revenue column per category** (`Hardware`, `Software`, `Services`), where revenue = `SUM(quantity * unit_price)` over that customer's **completed** orders. Add a `grand_total` column. Beware the fan-out: make sure joining `order_items` does not double-count. Zeros, not NULLs, in empty cells. Only include customers whose `grand_total` exceeds 1000.

```sql
-- Write your query here
```

---

### Exercise 3 — Hard (production simulation; the naive answer is wrong)

You have `events(id, user_id, event_type, occurred_at)` with ~80M rows. Product wants a daily grid for the last 30 days: **one row per day**, columns for `event_type IN ('login','purchase','refund')`, cells = distinct-user counts, plus a `dau` (distinct active users, any event) column.

A teammate wrote:

```sql
SELECT
  occurred_at::date AS day,
  COUNT(DISTINCT CASE WHEN event_type = 'login' THEN user_id END)    AS logins,
  COUNT(DISTINCT CASE WHEN event_type = 'purchase' THEN user_id END) AS purchases,
  COUNT(DISTINCT user_id)                                            AS dau
FROM events
GROUP BY occurred_at::date;
```

1. Name two correctness/robustness problems (hint: the missing `refund` column silently drops nothing but the missing `WHERE` scans all 80M rows; and consider what happens to the plan's memory with 30 days × millions of users of `DISTINCT` state).
2. Rewrite it correctly and efficiently: restrict to 30 days, include `refund`, and make the distinct-user cells correct.
3. What index would you add, and would it enable a GroupAggregate to avoid a hash-aggregate spill?

```sql
-- Write your query here
```

---

### Exercise 4 — Interview Simulation

"Our categories are user-defined and change weekly. Build me a report with one column per category showing this month's revenue, and it must automatically include new categories without a code change."

1. Explain in one sentence why a single static SQL statement cannot satisfy the "automatically include new categories" requirement.
2. Give the JSONB-aggregate query that solves it without dynamic SQL.
3. Sketch the alternative dynamic-SQL PL/pgSQL approach and state its two safety requirements.

```sql
-- Write your query here
```

---

## 14. Interview Questions

### Junior Level

**Q: What is a pivot, and how do you write one in standard SQL when there is no `PIVOT` keyword?**

A: A pivot turns rows into columns — it takes a categorical column (e.g., `payment_method`) whose distinct values become the *headers* of new columns, and aggregates a measure (e.g., `SUM(amount)`) into each cell. PostgreSQL has no `PIVOT` keyword (unlike SQL Server / Oracle), so the standard technique is **conditional aggregation**: `GROUP BY` the row key, then one aggregate-over-CASE expression per output column.

```sql
SELECT
  o.user_id,
  SUM(CASE WHEN p.method = 'card'   THEN p.amount ELSE 0 END) AS card,
  SUM(CASE WHEN p.method = 'paypal' THEN p.amount ELSE 0 END) AS paypal
FROM orders o
JOIN payments p ON p.order_id = o.id
GROUP BY o.user_id;
```

Each `CASE` selects the rows belonging to one column; the aggregate collapses them into a single cell. The number of output columns is fixed at the moment you write the query.

**Follow-up: Why `ELSE 0` and not just letting it be NULL?**

It depends on intent. `ELSE 0` makes empty cells show as `0` (good for "revenue by method" where absence means zero). Omitting `ELSE` leaves them NULL, which `SUM`/`COUNT` treat correctly and which distinguishes "no data" from "genuinely zero" — but then downstream arithmetic must handle NULL. For a `COUNT` pivot, `COUNT(CASE WHEN ... THEN 1 END)` (no ELSE) is idiomatic because `COUNT` ignores NULLs.

---

**Q: What's the difference between the `CASE`-based pivot and the `FILTER` clause?**

A: They produce identical results; `FILTER` is just cleaner, standard SQL syntax for the same conditional aggregation.

```sql
-- These two are equivalent:
SUM(CASE WHEN status = 'paid' THEN amount ELSE 0 END) AS paid
SUM(amount) FILTER (WHERE status = 'paid')          AS paid
```

`FILTER` reads better, avoids the `ELSE 0`/NULL confusion (it simply excludes non-matching rows from the aggregate), and works with any aggregate. The one thing to remember: `SUM(...) FILTER (WHERE ...)` returns NULL, not 0, when no rows match — wrap in `COALESCE(..., 0)` if you need zeros.

**Follow-up: Does `FILTER` perform better than `CASE`?**

No meaningful difference — the planner treats them the same way; both scan the table once and evaluate the predicate per row per aggregate. Choose `FILTER` for readability, not speed.

---

**Q: What does "unpivot" mean and when would you need it?**

A: Unpivot is the inverse — turning columns back into rows. If you have a wide `employees` table with columns `q1_sales, q2_sales, q3_sales, q4_sales`, unpivoting produces one row per (employee, quarter) pair. You need it to normalize a wide/denormalized layout so you can `GROUP BY`, filter, or join on the quarter dimension. In PostgreSQL the common technique is a `LATERAL` join over a `VALUES` list (or `unnest`):

```sql
SELECT e.id, v.quarter, v.sales
FROM employees e
CROSS JOIN LATERAL (VALUES
  ('q1', e.q1_sales), ('q2', e.q2_sales),
  ('q3', e.q3_sales), ('q4', e.q4_sales)
) AS v(quarter, sales);
```

---

### Principal Level

**Q: A team wants a report with "one column per category," and categories are user-editable and grow weekly. They wrote a static conditional-aggregation query and now file a ticket every time a category is added. Walk me through why this is fundamentally hard and what your options are.**

A: The core constraint: **SQL's result shape (the column list) must be known at plan time, before any data is read.** A pivot's columns come *from data* (the distinct category values), so a single static statement can never adapt its column set to rows it hasn't scanned yet. This isn't a PostgreSQL limitation — it's inherent to the relational model. `crosstab()` from the `tablefunc` extension doesn't escape it either: you must still declare the output column list in the `AS (...)` clause.

Options, in order of how I'd usually recommend them:

1. **Pivot in the application layer.** Return the data *tall* (`category, revenue` — one row per category) and let the app/BI tool (Metabase, a React table, pandas `pivot_table`) spread it into columns. This is almost always the right answer: the presentation layer already knows how to render a dynamic column set, and the SQL stays trivial and cacheable. The "report must be SQL" assumption is usually the real bug.

2. **Return a single JSONB column.** Aggregate into a document whose keys are the categories: `jsonb_object_agg(c.name, c.revenue)`. One static query, keys grow automatically, and the consumer reads by key. Loses tabular typing but fully dynamic with zero DDL.

3. **Dynamic SQL in PL/pgSQL.** A function that queries the distinct categories, builds the pivot statement as a string, and `EXECUTE`s it. This genuinely produces dynamic columns, but the price is steep: the return type varies per call (so callers need `RETURNS SETOF record` and a column definition list, or a refcursor), and it's a **SQL-injection surface** — every interpolated identifier must go through `quote_ident()` / `format(%I)`, never string concatenation.

I'd push back hard on option 3 unless there's a concrete reason the pivot must happen inside the database.

**Follow-up: Suppose it must be server-side and typed. What's the safety checklist for the dynamic-SQL function?**

Two non-negotiables: (1) **identifier quoting** — every category value spliced into a column name goes through `format('%I', name)` so a category literally named `"; DROP TABLE ...` can't inject; (2) **a stable, declared return contract** — either a fixed `RETURNS TABLE(...)` when the domain of categories is bounded, or force callers to supply the column definition list, because a function whose column *set* changes between calls breaks prepared statements, views built on top of it, and most ORMs.

---

**Q: You have a 200M-row `audit_logs` table and need a daily pivot: for each day, counts of distinct users per `action` type, over the last 90 days. The naive `COUNT(DISTINCT user_id) FILTER (...)` per action is spilling `work_mem` and taking minutes. How do you make this fast?**

A: Several layers:

1. **Filter before you pivot.** The predicate `created_at >= now() - interval '90 days'` must be pushed down to an index (`audit_logs (created_at)` or better `(created_at, action, user_id)`) so you scan ~90 days, not 200M rows. A pivot over an unfiltered table is the most common cause of these blowups.

2. **Understand the memory cost.** `COUNT(DISTINCT user_id)` per group must hold a distinct-set in memory for every (day × action) group simultaneously during a HashAggregate — with millions of users that's what spills. Two mitigations: (a) a covering index that lets the planner do a `GroupAggregate` over sorted input (bounded memory instead of a giant hash), or (b) if exactness isn't required, `APPROX_COUNT_DISTINCT`-style HyperLogLog via the `postgresql-hll` extension, which uses kilobytes per group.

3. **Pre-aggregate incrementally.** A 90-day daily report shouldn't rescan history every run. Maintain a rollup table `daily_action_users(day, action, distinct_users)` (or an HLL sketch per day that's mergeable), populated by a nightly job for the closed day. The live query then pivots a tiny summary table — milliseconds. This is the answer that actually scales.

**Follow-up: The rollup for distinct users can't just be SUMmed across days if you later want "distinct users over the whole 90 days." Why, and what fixes it?**

Because distinct-counts aren't additive — a user active on 30 days is counted 30 times if you sum daily distincts. Only the *per-day* number is correct from a per-day rollup. To get a mergeable multi-day distinct count you must store something composable: an HLL sketch per day (`hll_union` across days, then `hll_cardinality`), or a roaring-bitmap of user IDs. That's the whole reason HLL exists — additivity of a distinct measure across your pivot's row dimension.

---

## 15. Mental Model Checkpoint

1. **Why must the number of columns in a pivot be fixed before the query runs, and what does that imply about "dynamic" pivots?**

2. **`SUM(amount) FILTER (WHERE status='paid')` returns NULL for a group with no paid orders, but `SUM(CASE WHEN status='paid' THEN amount ELSE 0 END)` returns 0. Explain the difference and when each is what you want.**

3. **You pivot `payments` by `method` into 5 columns. A new method `'crypto'` starts appearing in the data. What does your existing query show for those rows, and why is that dangerous silently?**

4. **`crosstab()` from `tablefunc` is described as "the built-in pivot." Yet you still have to type out the column names in an `AS (...)` clause. So what problem does `crosstab()` actually solve that plain conditional aggregation doesn't?**

5. **To unpivot a wide `employees(q1_sales..q4_sales)` table, you use `CROSS JOIN LATERAL (VALUES ...)`. How many output rows does one employee produce, and what happened to the four column *names*?**

6. **A conditional-aggregation pivot and a `crosstab()` call over the same data can disagree on the result. Under what data condition does that happen, and which one is usually "right"?**

7. **Why is dynamic-SQL pivot generation a SQL-injection risk even when the raw category values come from your own `categories` table and never from user input directly?**

---

## 16. Quick Reference Card

```sql
-- ============ PIVOT: rows -> columns (conditional aggregation) ============
-- No PIVOT keyword in PostgreSQL. Column set is FIXED at write time.

-- CASE form (ELSE 0 => empty cells show 0):
SELECT o.user_id,
  SUM(CASE WHEN p.method='card'   THEN p.amount ELSE 0 END) AS card,
  SUM(CASE WHEN p.method='paypal' THEN p.amount ELSE 0 END) AS paypal
FROM orders o JOIN payments p ON p.order_id = o.id
GROUP BY o.user_id;

-- FILTER form (standard, cleaner; empty cells => NULL, wrap in COALESCE):
SELECT o.user_id,
  COALESCE(SUM(p.amount) FILTER (WHERE p.method='card'),   0) AS card,
  COALESCE(SUM(p.amount) FILTER (WHERE p.method='paypal'), 0) AS paypal
FROM orders o JOIN payments p ON p.order_id = o.id
GROUP BY o.user_id;

-- COUNT pivot: no ELSE, COUNT ignores NULL
COUNT(*) FILTER (WHERE status='shipped') AS shipped_count

-- ============ crosstab() : tablefunc extension ============
CREATE EXTENSION IF NOT EXISTS tablefunc;
SELECT * FROM crosstab(
  $$ SELECT user_id, method, SUM(amount)
     FROM payments GROUP BY 1,2 ORDER BY 1,2 $$    -- MUST be ordered by row,cat
) AS ct(user_id int, card numeric, paypal numeric);  -- columns STILL declared
-- crosstab buys: cleaner N-column output from tall input; NOT dynamic columns.

-- ============ UNPIVOT: columns -> rows ============
SELECT e.id, v.quarter, v.sales
FROM employees e
CROSS JOIN LATERAL (VALUES
  ('q1', e.q1_sales), ('q2', e.q2_sales),
  ('q3', e.q3_sales), ('q4', e.q4_sales)
) AS v(quarter, sales);          -- 4 rows per employee; names become data

-- ============ DYNAMIC columns (the real limitation) ============
-- A static statement CANNOT grow its column list from data.
-- Option A (preferred): return TALL, pivot in app/BI layer.
-- Option B: one JSONB column, keys grow automatically:
SELECT jsonb_object_agg(c.name, c.revenue) FROM (...) c;
-- Option C: PL/pgSQL + EXECUTE format('... %I ...', category)  -- quote_ident!
--           varies return type; injection surface; last resort.

-- ============ PERFORMANCE ============
-- Always FILTER the source to a date range BEFORE pivoting (index the predicate).
-- COUNT(DISTINCT x) per cell holds a set per group -> work_mem spills.
--   fix: covering index for GroupAggregate, or HLL for approx + mergeability.
-- Pre-aggregate to a rollup table for recurring reports; distinct counts are
--   NOT additive across the row dimension -> store HLL sketches to merge.
```

---

## Connected Topics

- **Topic 20 — GROUP BY Fundamentals**: Every pivot is a `GROUP BY` on the row key with conditional aggregates in the SELECT — pivoting is impossible to understand without it.
- **Topic 22 — Aggregate Functions**: `SUM`/`COUNT`/`AVG` are the cell values; the `FILTER` clause that powers clean pivots is an aggregate feature.
- **Topic 07 — NULL in Depth**: The `ELSE 0` vs NULL cell decision, and why `COUNT` ignores NULL, both hinge on three-valued logic.
- **Topic 11 — INNER JOIN**: Pivots almost always join fact to dimension (`payments`→`orders`, `order_items`→`products`) before aggregating.
- **Topic 33 — JSONB Operations**: `jsonb_object_agg` is the escape hatch for genuinely dynamic pivot columns without dynamic SQL.
- **Topic 44 — LATERAL Joins**: The `CROSS JOIN LATERAL (VALUES ...)` pattern is the idiomatic unpivot in PostgreSQL.
- **Topic 51 — Dynamic SQL and PL/pgSQL**: The only way to get data-driven column *sets* server-side — and its `quote_ident`/injection safety requirements.
- **Topic 58 — Materialized Views and Rollups**: Pre-aggregating a pivot into a summary table is how these reports scale past millions of rows.
- **Topic 60 — Approximate Aggregation (HLL)**: Makes distinct-user pivot cells cheap and, crucially, mergeable across the row dimension.
