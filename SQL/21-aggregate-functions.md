# Topic 21 — Aggregate Functions in Depth
### SQL Mastery Curriculum — Phase 4: Aggregation and Grouping

---

## 1. ELI5 — The Analogy That Makes It Click

Imagine you run a small bakery and at the end of each day you empty the till onto the counter. You have a messy pile of receipts. Someone asks you five different questions:

- "How many receipts are there?" — you count the pieces of paper. That's `COUNT(*)`.
- "How many receipts actually had a tip written on them?" — you count only the ones where the tip line is filled in, and skip the ones where the customer left it blank. That's `COUNT(tip)`.
- "How many *different* tip amounts did people leave?" — someone left $2, someone left $2, someone left $5. That's two distinct values. That's `COUNT(DISTINCT tip)`.
- "What's the total of all the tips?" — you add them up, skipping the blank ones. That's `SUM(tip)`.
- "What was the biggest single tip?" — you find the largest number. That's `MAX(tip)`.

The crucial insight the whole day teaches you: **a blank tip line is not zero.** A customer who left the tip line blank did not tip zero dollars — they simply didn't tell you anything. So when you count "receipts with a tip," you skip them; when you total the tips, you skip them; but when you count "pieces of paper," you still count the paper.

That single distinction — **a blank (NULL) is unknown, not zero, and every aggregate function decides for itself whether to skip the blanks** — is 80% of what separates people who get aggregate bugs in production from people who don't. Everything else in this topic is a careful application of that one idea across ten different functions.

An aggregate function takes **many rows and collapses them into one value.** The pile of receipts (many rows) becomes a single number (one answer). That collapse is the entire job.

---

## 2. Connection to SQL Internals

An aggregate function is implemented in PostgreSQL as a **state machine that walks a stream of rows.** Every aggregate is defined by four internal pieces (visible in the `pg_aggregate` catalog):

1. **An initial state** (`agginitval`) — the starting accumulator. For `SUM` over `bigint` it's NULL; for `COUNT` it's `0`; for `array_agg` it's an empty internal buffer.
2. **A state transition function** (`aggtransfn`) — called once per input row. For `SUM(int)` it's `int4_sum`, which does `state = state + value`. For `COUNT(*)` it's `int8inc`, which does `state = state + 1`.
3. **A final function** (`aggfinalfn`, optional) — called once at the end to turn the internal state into the output. `AVG` keeps a running `(sum, count)` pair as state and the final function divides them.
4. **A combine function** (`aggcombinefn`, optional) — merges two partial states, which is what makes **parallel aggregation** possible.

This state-machine model explains almost every behavior in this topic:

- **Why NULL is skipped by most aggregates**: aggregates are marked `strict` on their transition function. A strict transition function is simply *not called* when the incoming value is NULL. The state passes through untouched. That's not a special rule bolted on — it falls straight out of strictness. `COUNT(*)` is the exception because it takes no column argument, so there is nothing that can be NULL.

- **Why the empty set gives NULL (but COUNT gives 0)**: if zero rows arrive, the transition function is never called, so the state stays at its initial value. For `SUM` that initial state is NULL → result NULL. For `COUNT` it's `0` → result `0`.

- **How the planner runs it**: aggregation appears in the plan as either an **`Aggregate`** node (a single group, or `HashAggregate` / `GroupAggregate` for `GROUP BY` — Topic 20) or as a **`Partial Aggregate` → Gather → Finalize Aggregate`** pipeline when parallel workers each compute a partial state and the leader combines them via the combine function.

- **DISTINCT and ORDER BY inside an aggregate force a Sort**: `COUNT(DISTINCT x)` and `array_agg(x ORDER BY y)` cannot be satisfied by a plain streaming HashAggregate. PostgreSQL must first sort (or hash-dedup) the input to that aggregate. This is why `COUNT(DISTINCT)` shows up in `EXPLAIN` with a `Sort` node feeding a `GroupAggregate`, and why it cannot be parallelized the same way a plain `COUNT` can.

- **`work_mem` governs the accumulator size**: `array_agg`, `string_agg`, `json_agg` build up an ever-growing buffer in memory as they walk the rows. That buffer lives in `work_mem`. Aggregating a billion rows into one giant array is how you OOM a worker.

Keep the state-machine picture in your head for the rest of this topic. Every "gotcha" is just the machine doing exactly what its transition function says.

---

## 3. Logical Execution Order Context

```
FROM / JOIN          ← rows are produced and combined
WHERE                ← row-level filter (runs BEFORE aggregation)
GROUP BY             ← rows partitioned into groups (Topic 20)
  ↓ AGGREGATES EVALUATED HERE — one pass per group
HAVING               ← group-level filter (runs AFTER aggregation, Topic 22)
SELECT               ← projection; aggregate results become columns
DISTINCT
ORDER BY             ← the query's final ordering (NOT the aggregate's internal ORDER BY)
LIMIT
```

Three consequences of this ordering matter enormously for aggregates:

1. **`WHERE` runs before the aggregate; `HAVING` runs after.** `WHERE amount > 0` removes rows *before* `SUM` ever sees them. `HAVING SUM(amount) > 1000` removes whole groups *after* the sum is computed. You cannot put an aggregate in `WHERE` (there's nothing to aggregate yet) and you cannot cheaply reproduce a `HAVING` in `WHERE`. Topic 22 is entirely about this line.

2. **An aggregate with no `GROUP BY` collapses the *entire* input to one row.** `SELECT COUNT(*) FROM orders` produces exactly one row even if `orders` is empty (the value is 0). This is the "grand total" mode.

3. **`ORDER BY` inside an aggregate is a different clause from the query's `ORDER BY`.** `string_agg(name, ',' ORDER BY name)` orders the *inputs being concatenated*, evaluated during the aggregation step. The query-level `ORDER BY` at the bottom orders the *output rows*. They run at different times and do different jobs. Confusing them is a common source of "why isn't my list sorted?" bugs.

Because aggregates sit *after* `WHERE` and *before* `HAVING`, the single most important predictive skill is knowing which rows have already been filtered out (by `WHERE`) before the collapse happens, and remembering that NULLs which survived `WHERE` may still be skipped by the aggregate itself.

---

## 4. What Is an Aggregate Function?

An **aggregate function** consumes a set of input rows and returns a single scalar value that summarizes them. Where a scalar function like `UPPER(name)` maps one row to one row, an aggregate maps *N* rows to *one* value per group. With no `GROUP BY`, the whole result set is one group.

```sql
SELECT
  COUNT(*)                                   AS row_count,
  COUNT(discount_code)                       AS rows_with_a_code,
  COUNT(DISTINCT customer_id)                AS unique_customers,
  SUM(total_amount)                          AS revenue,
  AVG(total_amount)                          AS avg_order_value,
  MIN(created_at)                            AS first_order,
  MAX(created_at)                            AS last_order,
  string_agg(status, ',' ORDER BY status)    AS all_statuses
FROM orders
WHERE created_at >= DATE '2026-01-01';
```

Annotated breakdown of the anatomy of an aggregate call:

```
COUNT ( DISTINCT customer_id )
│       │        │
│       │        └── the input expression — evaluated once per row,
│       │            its value fed to the transition function
│       │
│       └── set quantifier: DISTINCT (dedupe inputs first) or ALL (default, keep all)
│
└── the aggregate name — chooses which state machine runs

string_agg ( status , ',' ORDER BY status )
│            │        │     │
│            │        │     └── ORDER BY inside the aggregate: sorts the INPUT
│            │        │         values before the transition function sees them
│            │        │
│            │        └── the delimiter argument (aggregates can take >1 arg)
│            │
│            └── first argument: the value being concatenated
│
└── ordered-set-sensitive aggregate — output depends on input order
```

The full grammar of an aggregate call in PostgreSQL is:

```
aggregate_name ( [ ALL | DISTINCT ] expression [ , ... ]
                 [ ORDER BY sort_expression [ ASC | DESC ] [ , ... ] ] )
  [ FILTER ( WHERE condition ) ]
```

- **`ALL | DISTINCT`** — whether to feed every input value or only distinct ones. `ALL` is the default.
- **`ORDER BY ...`** — orders the inputs; matters for order-sensitive aggregates (`string_agg`, `array_agg`, `json_agg`) and is ignored (result-wise) for order-insensitive ones (`SUM`, `COUNT`, `MIN`).
- **`FILTER (WHERE ...)`** — a per-aggregate row filter, evaluated during aggregation. This lets two aggregates in the same `SELECT` see different subsets of rows. Covered in depth in Section 6.

There are two families:
- **General-purpose aggregates**: `COUNT`, `SUM`, `AVG`, `MIN`, `MAX`, `string_agg`, `array_agg`, `bool_and`, `bool_or`, `json_agg`, `jsonb_agg`, and more.
- **Ordered-set aggregates** (different syntax, using `WITHIN GROUP`): `percentile_cont`, `percentile_disc`, `mode`. Mentioned here, treated fully alongside window functions in later topics.

---

## 5. Why Aggregate Mastery Matters in Production

1. **The `COUNT` variants silently produce different numbers.** `COUNT(*)`, `COUNT(col)`, and `COUNT(DISTINCT col)` return three different values on the same table, and picking the wrong one produces a report that is *plausibly wrong* — no error, just an incorrect KPI on a dashboard that someone makes a business decision on. Undercounting active users or overcounting orders because of a fan-out join (Topic 11) is a classic incident.

2. **NULL handling changes financial math.** `AVG(amount)` skips NULLs and divides by the count of *non-NULL* rows. If half your rows have NULL amounts, `AVG` gives you the average of the *other* half — not the average you'd get by treating NULLs as 0. On a revenue-per-user metric, that's the difference between a 2× overstatement and the truth.

3. **Empty-set behavior breaks calling code.** `SELECT SUM(amount) FROM orders WHERE customer_id = $1` returns **one row containing NULL** when the customer has no orders — not zero rows, and not `0`. Node code that does `result.rows[0].sum + shipping` gets `NULL + number = NULL` semantics, or in JS `null + 5 = 5` (subtly wrong) — a bug that only appears for new customers.

4. **`array_agg` / `string_agg` / `json_agg` can OOM a worker.** These build an unbounded in-memory buffer. Aggregating a huge group into one JSON blob is how a well-meaning "return all a customer's orders as JSON" endpoint takes down a database under `work_mem` pressure.

5. **`COUNT(DISTINCT)` is expensive and doesn't parallelize like `COUNT(*)`.** Knowing it forces a sort/hash-dedup — and knowing the approximate alternatives — is the difference between a 200ms dashboard and a 40-second one at scale.

6. **Ordering inside aggregates is non-obvious.** Building a comma-separated list or a JSON array in a deterministic order requires `ORDER BY` *inside* the aggregate. Developers routinely add a query-level `ORDER BY` and are baffled that the concatenated list is still in random order.

Getting aggregates exactly right is the foundation of every report, every dashboard, every analytics query, and every `GROUP BY` (Topic 20) and `HAVING` (Topic 22) that builds on this topic.

---

## 6. Deep Technical Content

### 6.1 The Three `COUNT`s — Exactly Why They Differ

This is the single most-asked aggregate interview question. Here is the complete, precise story.

Consider this table:

```
orders
id | customer_id | coupon_code
---+-------------+------------
1  | 100         | 'SAVE10'
2  | 100         | NULL
3  | 101         | 'SAVE10'
4  | 101         | NULL
5  | NULL        | NULL
```

```sql
SELECT
  COUNT(*)                     AS a,  -- 5
  COUNT(coupon_code)           AS b,  -- 2
  COUNT(DISTINCT coupon_code)  AS c,  -- 1
  COUNT(customer_id)           AS d,  -- 4
  COUNT(DISTINCT customer_id)  AS e   -- 2
FROM orders;
```

- **`COUNT(*)` = 5.** `COUNT(*)` counts **rows**, full stop. It takes no argument, so there is no value that could be NULL. It is literally "how many rows survived `WHERE`." Internally its transition function is `int8inc` — "add one," called unconditionally per row.

- **`COUNT(coupon_code)` = 2.** `COUNT(expr)` counts rows where `expr IS NOT NULL`. Three rows have `NULL` coupon codes, so they are skipped. The transition function is strict — not called for NULL inputs. Result: 2 (rows 1 and 3).

- **`COUNT(DISTINCT coupon_code)` = 1.** First skip NULLs (leaving `'SAVE10'`, `'SAVE10'`), then dedupe → one distinct value. **DISTINCT also removes NULLs**, but even if it didn't, they were already gone via strictness. Result: 1.

- **`COUNT(customer_id)` = 4.** Row 5 has `NULL` customer_id → skipped. Result: 4.

- **`COUNT(DISTINCT customer_id)` = 2.** Non-NULL customer_ids are `100, 100, 101, 101` → two distinct. Result: 2.

The precise rules, memorized:

| Form | Counts | NULLs | Duplicates |
|------|--------|-------|-----------|
| `COUNT(*)` | rows | counted (there's no value to be NULL) | counted |
| `COUNT(expr)` | rows where `expr IS NOT NULL` | skipped | counted |
| `COUNT(DISTINCT expr)` | distinct non-NULL values of `expr` | skipped | collapsed to one |

**Performance corollary**: `COUNT(*)` and `COUNT(1)` are *identical* in PostgreSQL — the planner treats `COUNT(1)` as counting rows because `1` is never NULL. There is no performance difference; the old "use `COUNT(1)` it's faster" advice is a myth in Postgres. `COUNT(*)` is the idiomatic form.

`COUNT(DISTINCT expr)` is the expensive one: it must sort or hash the whole input to that aggregate to remove duplicates. See Section 10.

### 6.2 Why `COUNT(*)` Is Not Free (and the visibility problem)

A frequent misconception: "`SELECT COUNT(*) FROM big_table` should be instant, it's just a stored number." It is not. Because of **MVCC** (Topic on MVCC), PostgreSQL cannot keep a single authoritative row count: different transactions see different numbers of rows depending on which tuples are visible to their snapshot. So `COUNT(*)` must actually **visit rows to check visibility.**

- With no `WHERE`, `COUNT(*)` on a large table is typically a **Parallel Seq Scan** (or an **Index Only Scan** if there's a suitable index and the visibility map is mostly all-visible, so it can count from the index without touching the heap).
- An `Index Only Scan` for count works only when the pages are marked all-visible in the visibility map (kept fresh by `VACUUM`). Otherwise it falls back to heap fetches.
- For an approximate, instant count, `SELECT reltuples FROM pg_class WHERE relname = 'orders'` reads the planner's cached estimate — good enough for "roughly how many rows," updated by `ANALYZE`/`VACUUM`.

### 6.3 SUM — NULL Skipping and the Empty-Set NULL

`SUM(expr)` adds up all non-NULL values of `expr`.

```sql
-- amounts: 10, 20, NULL, 30
SELECT SUM(amount) FROM t;  -- 60  (NULL skipped, NOT treated as 0)
```

Two behaviors you must burn into memory:

1. **`SUM` skips NULLs.** `10 + 20 + NULL + 30` is *not* NULL — the NULL row is skipped, giving 60. (This surprises people who expect "anything + NULL = NULL." That rule applies to the `+` *operator*, not to the `SUM` *aggregate*, whose transition function is strict.)

2. **`SUM` over zero rows returns NULL, not 0.** This is the opposite of `COUNT`.

```sql
SELECT SUM(amount) FROM orders WHERE 1 = 0;  -- NULL (one row, value NULL)
SELECT COUNT(amount) FROM orders WHERE 1 = 0; -- 0
```

The standard production fix for "I want 0 for the empty set" is `COALESCE`:

```sql
SELECT COALESCE(SUM(amount), 0) AS revenue FROM orders WHERE customer_id = $1;
```

**Type/overflow notes**: `SUM(integer)` returns `bigint`; `SUM(bigint)` returns `numeric` (to avoid overflow); `SUM(numeric)` returns `numeric`; `SUM(real/double)` returns `double precision` (with floating-point rounding). Summing a large `integer` column is safe from overflow because the result widens to `bigint`. Summing `bigint` widens to `numeric`. Beware summing `money` or doing float sums where order-of-addition rounding matters.

### 6.4 AVG — The Division Trap

`AVG(expr) = SUM(non-NULL expr) / COUNT(non-NULL expr)`. The denominator is the count of **non-NULL** values, not the row count.

```sql
-- amounts: 10, 20, NULL, 30   (4 rows, 3 non-NULL)
SELECT AVG(amount) FROM t;   -- 20  =  (10+20+30) / 3   -- NOT /4
```

If you *wanted* NULLs to count as zero (i.e., divide by 4), you must say so explicitly:

```sql
SELECT AVG(COALESCE(amount, 0)) FROM t;  -- 15 = (10+20+0+30)/4
-- or equivalently
SELECT SUM(amount) / COUNT(*) FROM t;    -- careful: integer division if amount is int
```

Empty-set behavior: `AVG` over zero rows (or all-NULL) returns **NULL** (division by zero avoided — it never divides). `AVG(integer)` returns `numeric` (exact, not truncated), which is a common surprise for people expecting integer truncation — `AVG` of `1, 2` is `1.5000000000000000`, not `1`.

### 6.5 MIN / MAX — Type-Generic, NULL-Skipping, No Empty-Set Zero

`MIN` and `MAX` return the smallest/largest non-NULL value using the type's default ordering operator (`<` / `>`). They work on **any orderable type**: numbers, text (collation order), dates, timestamps, even `boolean` (FALSE < TRUE) and arrays.

```sql
SELECT MIN(created_at), MAX(created_at) FROM orders;  -- earliest, latest
SELECT MIN(name), MAX(name) FROM users;               -- alphabetically first/last
```

- **NULLs are skipped**, so `MAX(amount)` ignores NULL amounts.
- **Empty set → NULL** (like SUM/AVG, unlike COUNT). `MAX` of zero rows is NULL.
- **Text ordering follows collation**: `MAX(name)` under `en_US` collation may differ from `C` collation. Be aware when the "largest string" matters.
- **`MAX` can use an index**: `SELECT MAX(id) FROM orders` becomes an `Index Scan Backward ... Limit 1` if `id` is indexed — O(log N), not a full scan. Same for `MIN` with a forward scan. This is a key optimization: bare `MIN`/`MAX` on an indexed column is essentially free. (Adding a `GROUP BY` or a `FILTER` can defeat this.)

### 6.6 The `FILTER` Clause — Per-Aggregate Row Filtering

`FILTER (WHERE condition)` restricts which rows a *single* aggregate sees, without affecting the others. This replaces the old `SUM(CASE WHEN ... THEN x END)` idiom and is clearer and often faster.

```sql
SELECT
  COUNT(*)                                          AS total_orders,
  COUNT(*) FILTER (WHERE status = 'completed')      AS completed,
  COUNT(*) FILTER (WHERE status = 'cancelled')      AS cancelled,
  SUM(total_amount) FILTER (WHERE status = 'completed') AS completed_revenue,
  AVG(total_amount) FILTER (WHERE status = 'completed' AND total_amount > 0) AS aov
FROM orders
WHERE created_at >= DATE '2026-01-01';
```

The `WHERE` at the bottom filters rows for *all* aggregates; each `FILTER` narrows further for just that one aggregate. The equivalent pre-`FILTER` pattern:

```sql
-- old style, same result, exploits that COUNT skips NULL and SUM skips NULL:
COUNT(status = 'completed' OR NULL)                       -- hacky
COUNT(CASE WHEN status = 'completed' THEN 1 END)          -- classic
SUM(CASE WHEN status = 'completed' THEN total_amount END) -- classic
```

`FILTER` is the modern, readable form. Note that `COUNT(*) FILTER (WHERE ...)` over an all-filtered-out group still returns `0` (COUNT semantics), while `SUM(...) FILTER (...)` returns NULL — the empty-set rules apply to the *filtered* input.

### 6.7 string_agg — Concatenation with a Delimiter and Order

`string_agg(expression, delimiter)` concatenates the non-NULL values of `expression`, separated by `delimiter`.

```sql
SELECT string_agg(name, ', ') FROM products WHERE category_id = 5;
-- 'Widget, Gadget, Gizmo'
```

Critical details:

1. **NULLs are skipped** (no extra delimiter is inserted for a skipped NULL). Only non-NULL values are joined.
2. **Order is undefined unless you specify `ORDER BY` inside the aggregate.**

```sql
-- deterministic, alphabetized list:
SELECT string_agg(name, ', ' ORDER BY name) FROM products WHERE category_id = 5;
-- 'Gadget, Gizmo, Widget'
```

Without the internal `ORDER BY`, the concatenation order matches the (arbitrary) order rows happen to arrive — which can change between runs, between plan choices, and between PostgreSQL versions. **Never rely on incidental ordering.**

3. **Empty set → NULL** (not an empty string). A group with no non-NULL values yields NULL. Wrap in `COALESCE(string_agg(...), '')` if you need `''`.
4. The delimiter and value must be the same string type (`text`/`varchar`); use casts for other types: `string_agg(id::text, ',')`.
5. `string_agg(DISTINCT name, ', ' ORDER BY name)` deduplicates — but note when combining `DISTINCT` with `ORDER BY`, the `ORDER BY` expression must match the `DISTINCT` expression.

### 6.8 array_agg — Collecting Values Into an Array

`array_agg(expression)` builds a PostgreSQL array of all input values.

```sql
SELECT array_agg(product_id ORDER BY product_id) FROM order_items WHERE order_id = 42;
-- {7,7,19,33}
```

The behavior that separates `array_agg` from `string_agg`:

- **`array_agg` KEEPS NULLs.** Unlike almost every other aggregate, `array_agg` includes NULL elements in the array. `array_agg(x)` over values `1, NULL, 2` produces `{1,NULL,2}` — a three-element array with a NULL in the middle. (The transition function is *not* strict for `array_agg`.) This is a frequent surprise.
- To drop NULLs: `array_agg(x) FILTER (WHERE x IS NOT NULL)` or `array_remove(array_agg(x), NULL)`.
- **`ORDER BY` inside** sorts the elements: `array_agg(x ORDER BY x)`.
- **Empty set → NULL** (not an empty array `{}`). Use `COALESCE(array_agg(x), '{}')` for an empty array on the empty set.
- Ordering with NULLs present: NULLs sort per `NULLS FIRST/LAST` rules; specify explicitly if it matters: `array_agg(x ORDER BY x NULLS LAST)`.

`array_agg` composed with `unnest` is the workhorse for pivoting and for passing collections back to application code.

### 6.9 bool_and / bool_or — Aggregating Truth

These reduce a set of boolean values to a single boolean.

- **`bool_and(expr)`** — TRUE only if *every* non-NULL input is TRUE (logical AND across rows). Also spelled `every()` (SQL standard alias).
- **`bool_or(expr)`** — TRUE if *at least one* non-NULL input is TRUE (logical OR across rows).

```sql
-- Is every item in this order in stock?
SELECT o.id, bool_and(p.in_stock) AS fully_available
FROM orders o
JOIN order_items oi ON oi.order_id = o.id
JOIN products p     ON p.id = oi.product_id
GROUP BY o.id;

-- Has this user had ANY failed payment?
SELECT user_id, bool_or(status = 'failed') AS had_any_failure
FROM payments GROUP BY user_id;
```

NULL and empty-set behavior:

- **NULLs are skipped.** `bool_and(TRUE, TRUE, NULL)` = TRUE; `bool_or(FALSE, NULL)` = FALSE.
- **Empty set (or all-NULL) → NULL.** `bool_and` over zero rows is NULL, not TRUE. `bool_or` over zero rows is NULL, not FALSE. This matters: "are all items in stock for an order with no items?" returns NULL, so `COALESCE(bool_and(...), TRUE)` if you want the vacuous-truth convention.

Mnemonic: `bool_and` is "all", `bool_or` is "any" — with NULL as "no opinion" (skipped) and no rows as "nothing to say" (NULL).

### 6.10 json_agg / jsonb_agg (and json_object_agg)

These collect rows into a JSON array — indispensable for returning nested structures to an application in one query.

```sql
-- All line items for an order, as a JSON array of objects:
SELECT
  o.id,
  json_agg(
    json_build_object('product_id', oi.product_id, 'qty', oi.quantity)
    ORDER BY oi.product_id
  ) AS items
FROM orders o
JOIN order_items oi ON oi.order_id = o.id
GROUP BY o.id;
-- items: [{"product_id":7,"qty":2},{"product_id":19,"qty":1}]
```

Key facts:

- **`json_agg` / `jsonb_agg` KEEP NULLs** (like `array_agg`): a NULL input becomes a JSON `null` element. Filter with `FILTER (WHERE x IS NOT NULL)` to drop them.
- **`json_agg` vs `jsonb_agg`**: `json` preserves textual formatting and key order and duplicate keys; `jsonb` is a parsed binary form — faster to operate on, deduplicates/reorders object keys, strips insignificant whitespace. For building output you'll return and discard, `json_agg` is marginally cheaper to build; for anything you'll index, filter, or further manipulate, use `jsonb_agg`.
- **`ORDER BY` inside** controls element order in the array.
- **Empty set → NULL** (not `[]`). Use `COALESCE(json_agg(...), '[]'::json)`.
- **`json_object_agg(key, value)`** builds a JSON object from key/value pairs across rows: `json_object_agg(status, cnt)` → `{"completed": 12, "cancelled": 3}`. A NULL *key* raises an error; a NULL *value* becomes JSON `null`.

Danger: `json_agg` over a huge group builds a large in-memory value. See performance section — don't `json_agg` a million rows into one blob.

### 6.11 DISTINCT Inside Any Aggregate

`DISTINCT` isn't special to `COUNT`. It works in `SUM`, `AVG`, `array_agg`, `string_agg`, etc., and always means "dedupe the input values first."

```sql
SUM(DISTINCT amount)         -- sum of each distinct amount ONCE (rarely what you want!)
AVG(DISTINCT rating)         -- average of distinct rating values
array_agg(DISTINCT tag)      -- unique tags
string_agg(DISTINCT status, ',' ORDER BY status)  -- unique statuses, ordered
```

`SUM(DISTINCT amount)` is a classic trap: if three orders are each $50, `SUM(DISTINCT amount)` counts $50 *once* = $50, not $150. Almost always a bug when used on money. `DISTINCT` in an aggregate is right for `COUNT`ing uniques and building unique lists, rarely for summing.

All `DISTINCT` aggregates force a sort/hash-dedup of the input — the same cost as `COUNT(DISTINCT)`.

### 6.12 ORDER BY Inside an Aggregate — The Full Story

Only **order-sensitive** aggregates change their result based on input order: `string_agg`, `array_agg`, `json_agg`, `jsonb_agg`, `json_object_agg`. For `SUM`, `COUNT`, `MIN`, `MAX`, `bool_and`, `bool_or`, the internal `ORDER BY` is *legal but pointless* — the result is identical regardless of order (addition and min/max are commutative), and it only adds sort cost.

```sql
-- meaningful:
array_agg(event_type ORDER BY occurred_at)      -- timeline in order
string_agg(tag, ',' ORDER BY tag)               -- alphabetized list
json_agg(row_to_json(t) ORDER BY t.priority DESC)

-- pointless (result unchanged, wasted sort):
SUM(amount ORDER BY created_at)                 -- same sum, extra sort
```

You may order by an expression that is *not* in the output:

```sql
array_agg(product_name ORDER BY sales_rank)     -- names ordered by rank, rank not emitted
```

Multiple sort keys are allowed: `array_agg(x ORDER BY a DESC, b ASC NULLS LAST)`.

Do not confuse this with the query's final `ORDER BY`:

```sql
SELECT category_id, string_agg(name, ',' ORDER BY name) AS names   -- ← orders the LIST
FROM products
GROUP BY category_id
ORDER BY category_id;                                              -- ← orders the ROWS
```

### 6.13 Aggregates Reference Table — NULL and Empty-Set Behavior

| Function | Skips NULL inputs? | Empty-set result | Order-sensitive? | Notes |
|----------|-------------------|------------------|------------------|-------|
| `COUNT(*)` | n/a (counts rows) | `0` | no | counts rows incl. all-NULL rows |
| `COUNT(expr)` | yes | `0` | no | counts non-NULL values |
| `COUNT(DISTINCT expr)` | yes | `0` | no | distinct non-NULL; forces sort |
| `SUM(expr)` | yes | **NULL** | no | int→bigint, bigint→numeric |
| `AVG(expr)` | yes | **NULL** | no | divides by non-NULL count |
| `MIN` / `MAX` | yes | **NULL** | no | any orderable type; can use index |
| `string_agg` | yes | **NULL** | **yes** | delimiter arg; no trailing delim for NULLs |
| `array_agg` | **NO (keeps NULL)** | **NULL** | **yes** | NULL becomes array element |
| `json_agg` / `jsonb_agg` | **NO (keeps NULL)** | **NULL** | **yes** | NULL becomes JSON null |
| `json_object_agg` | value: no; **key NULL errors** | **NULL** | yes | dupe keys: json keeps, jsonb last-wins |
| `bool_and` (`every`) | yes | **NULL** | no | AND across rows |
| `bool_or` | yes | **NULL** | no | OR across rows |

The two rules that cover 90% of bugs:
- **Only `COUNT` returns 0 for the empty set; every other aggregate returns NULL.**
- **Only `array_agg` / `json_agg` / `jsonb_agg` keep NULL inputs; every other aggregate skips them.**

---

## 7. EXPLAIN — Aggregation in the Plan

### 7.1 Plain grand-total aggregate (single group)

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT COUNT(*), SUM(total_amount), AVG(total_amount)
FROM orders
WHERE status = 'completed';
```

```
Finalize Aggregate  (cost=21847.55..21847.56 rows=1 width=48)
                    (actual time=118.2..118.2 rows=1 loops=1)
  ->  Gather  (cost=21847.33..21847.54 rows=2 width=48)
              (actual time=117.9..120.1 rows=3 loops=1)
        Workers Planned: 2
        Workers Launched: 2
        ->  Partial Aggregate  (cost=20847.33..20847.34 rows=1 width=48)
                               (actual time=112.4..112.4 rows=1 loops=3)
              ->  Parallel Seq Scan on orders
                    (cost=0.00..19180.66 rows=333333 width=8)
                    (actual time=0.02..64.1 rows=266667 loops=3)
                    Filter: (status = 'completed'::text)
                    Rows Removed by Filter: 66666
        Buffers: shared hit=384 read=12963
Planning Time: 0.21 ms
Execution Time: 120.4 ms
```

Reading it:
- **`Parallel Seq Scan`** — 2 workers + leader each scan a slice of `orders`, applying the `status` filter.
- **`Partial Aggregate`** — each worker computes its own partial `COUNT`/`SUM`/`AVG` state (`loops=3` = 3 processes).
- **`Gather`** — collects the 3 partial states.
- **`Finalize Aggregate`** — the leader **combines** the partial states via each aggregate's combine function into the final `rows=1`. This combine step is exactly the `aggcombinefn` from Section 2. Parallel aggregation is why `SUM`/`COUNT`/`AVG` scale across cores.

### 7.2 COUNT(DISTINCT) — the sort appears

```sql
EXPLAIN (ANALYZE)
SELECT COUNT(DISTINCT customer_id) FROM orders WHERE status = 'completed';
```

```
Aggregate  (cost=24980.00..24980.01 rows=1 width=8)
           (actual time=402.7..402.7 rows=1 loops=1)
  ->  Seq Scan on orders  (cost=0.00..23180.00 rows=720000 width=8)
                          (actual time=0.02..92.5 rows=800000 loops=1)
        Filter: (status = 'completed'::text)
Planning Time: 0.15 ms
Execution Time: 402.9 ms
```

Reading it:
- Notice there is **no `Gather` / parallel worker** here. `COUNT(DISTINCT)` is **not parallelized** the same way (the distinct-set must be built globally), so this is a single-process `Aggregate`.
- The `Aggregate` node internally sorts or hashes 800K customer_ids to dedupe them — that's the ~310ms above the scan time. Compare to the plain `COUNT(*)` which parallelized to ~120ms over more rows. This is the concrete cost of `DISTINCT` inside an aggregate.

### 7.3 MIN/MAX using an index (no scan at all)

```sql
EXPLAIN (ANALYZE)
SELECT MAX(created_at) FROM orders;
```

```
Result  (cost=0.46..0.47 rows=1 width=8) (actual time=0.05..0.05 rows=1 loops=1)
  InitPlan 1 (returns $0)
    ->  Limit  (cost=0.43..0.46 rows=1 width=8) (actual time=0.04..0.04 rows=1 loops=1)
          ->  Index Only Scan Backward using orders_created_at_idx on orders
                (cost=0.43..25600.00 rows=800000 width=8)
                (actual time=0.03..0.03 rows=1 loops=1)
                Index Cond: (created_at IS NOT NULL)
Planning Time: 0.20 ms
Execution Time: 0.07 ms
```

Reading it:
- The planner rewrote `MAX(created_at)` into "read the **last** entry of the `created_at` index" — an `Index Only Scan Backward` with `Limit 1`. **0.07ms**, no table scan.
- `Index Cond: (created_at IS NOT NULL)` — the index scan skips NULLs, matching `MAX`'s NULL-skipping semantics.
- Add a `GROUP BY` or `FILTER` and this optimization usually disappears (it becomes a full scan + grouping).

### 7.4 array_agg / string_agg — memory in the plan

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT order_id, array_agg(product_id ORDER BY product_id)
FROM order_items
GROUP BY order_id;
```

```
GroupAggregate  (cost=0.43..98450.00 rows=200000 width=40)
                (actual time=0.05..980.3 rows=200000 loops=1)
  Group Key: order_id
  ->  Index Scan using order_items_order_id_idx on order_items
        (cost=0.43..71200.00 rows=1000000 width=8)
        (actual time=0.03..420.1 rows=1000000 loops=1)
Planning Time: 0.18 ms
Execution Time: 998.7 ms
```

Reading it:
- **`GroupAggregate`** (not `HashAggregate`) because the input arrives already ordered by `order_id` via the index — so it can stream groups without a hash table, *and* the `ORDER BY product_id` inside `array_agg` is satisfied cheaply because within each `order_id` group the rows are already close.
- Each group accumulates an array in memory; the aggregate buffers live in `work_mem`. If groups were huge, you'd see this spill or balloon memory. Here groups are small (a few items each), so it's fine.

---

## 8. Query Examples

### Example 1 — Basic: The full aggregate vocabulary on one table

```sql
-- Daily order summary for a single day. Demonstrates each aggregate's
-- behavior on real columns, including NULL skipping and empty-set safety.
SELECT
  COUNT(*)                                      AS orders_placed,      -- all rows
  COUNT(coupon_code)                            AS orders_with_coupon, -- non-NULL coupons
  COUNT(DISTINCT customer_id)                   AS unique_customers,   -- distinct buyers
  COALESCE(SUM(total_amount), 0)                AS gross_revenue,      -- 0 not NULL if empty
  ROUND(AVG(total_amount), 2)                   AS avg_order_value,    -- non-NULL denom
  MIN(total_amount)                             AS smallest_order,
  MAX(total_amount)                             AS largest_order,
  bool_or(status = 'fraud_flagged')             AS any_fraud_flag      -- NULL if 0 rows
FROM orders
WHERE created_at >= DATE '2026-07-17'
  AND created_at <  DATE '2026-07-18';
```

### Example 2 — Intermediate: FILTER, string_agg, and conditional aggregation

```sql
-- Per-customer order profile in one pass, using FILTER to split by status
-- and string_agg to build a readable, ordered status history.
SELECT
  c.id                                                          AS customer_id,
  c.name,
  COUNT(*)                                                      AS total_orders,
  COUNT(*) FILTER (WHERE o.status = 'completed')                AS completed_orders,
  COUNT(*) FILTER (WHERE o.status = 'cancelled')                AS cancelled_orders,
  COALESCE(SUM(o.total_amount) FILTER (WHERE o.status = 'completed'), 0)
                                                                AS completed_revenue,
  ROUND(
    AVG(o.total_amount) FILTER (WHERE o.status = 'completed'), 2
  )                                                             AS avg_completed_value,
  MAX(o.created_at)                                             AS last_order_at,
  string_agg(DISTINCT o.status, ', ' ORDER BY o.status)         AS statuses_seen
FROM customers c
JOIN orders o ON o.customer_id = c.id
WHERE o.created_at >= DATE '2026-01-01'
GROUP BY c.id, c.name
HAVING COUNT(*) FILTER (WHERE o.status = 'completed') > 0       -- Topic 22
ORDER BY completed_revenue DESC
LIMIT 100;
```

### Example 3 — Production-Grade: Order document assembly with json_agg

```sql
-- GOAL: return each recent order as a single JSON document with its line items
-- nested inside, ready to hand straight to an API response — one round trip.
--
-- TABLE SIZES: orders ~5M rows, order_items ~30M rows, products ~200K rows.
-- INDEXES: order_items(order_id) btree; orders(created_at) btree; products PK.
-- PERF EXPECTATION: with the created_at filter selecting ~2K orders and the
-- order_items(order_id) index, this is a nested-loop over a small driving set;
-- expected < 40ms. Do NOT run this without a selective WHERE — json_agg over
-- millions of grouped rows will balloon work_mem.
SELECT
  o.id                       AS order_id,
  o.customer_id,
  o.status,
  o.total_amount,
  o.created_at,
  COALESCE(
    jsonb_agg(
      jsonb_build_object(
        'product_id', oi.product_id,
        'name',       p.name,
        'quantity',   oi.quantity,
        'unit_price', oi.unit_price,
        'line_total', oi.quantity * oi.unit_price
      )
      ORDER BY oi.product_id
    ) FILTER (WHERE oi.id IS NOT NULL),   -- drop the NULL row from the LEFT JOIN
    '[]'::jsonb                            -- empty array, not NULL, for item-less orders
  )                          AS line_items,
  COUNT(oi.id)               AS item_count,
  SUM(oi.quantity * oi.unit_price) AS items_subtotal
FROM orders o
LEFT JOIN order_items oi ON oi.order_id = o.id
LEFT JOIN products p     ON p.id = oi.product_id
WHERE o.created_at >= NOW() - INTERVAL '1 day'
GROUP BY o.id, o.customer_id, o.status, o.total_amount, o.created_at
ORDER BY o.created_at DESC;
```

```
-- Representative EXPLAIN (ANALYZE) shape:
GroupAggregate  (cost=8420.11..9105.42 rows=2010 width=140)
                (actual time=6.4..33.8 rows=2010 loops=1)
  Group Key: o.id
  ->  Nested Loop Left Join  (actual time=0.09..18.2 rows=12060 loops=1)
        ->  Nested Loop Left Join  (actual time=0.07..9.1 rows=12060 loops=1)
              ->  Index Scan using orders_created_at_idx on orders o
                    Index Cond: (created_at >= (now() - '1 day'::interval))
                    (actual rows=2010 loops=1)
              ->  Index Scan using order_items_order_id_idx on order_items oi
                    Index Cond: (order_id = o.id)   (actual rows=6 loops=2010)
        ->  Index Scan using products_pkey on products p
              Index Cond: (id = oi.product_id)      (actual rows=1 loops=12060)
Execution Time: 34.6 ms
```

The `FILTER (WHERE oi.id IS NOT NULL)` is load-bearing: without it, an order with no items would produce `[null]` (a one-element array containing JSON null) because `json_agg`/`jsonb_agg` keep NULLs. With the FILTER plus `COALESCE(..., '[]')`, an item-less order correctly yields `[]`.

---

## 9. Wrong → Right Patterns

### Wrong 1 — Using `COUNT(column)` when you meant to count rows

```sql
-- WRONG: counting a nullable column to get "number of orders"
SELECT COUNT(shipped_at) AS order_count FROM orders;
-- BUG: shipped_at is NULL for every unshipped order. This counts only
-- SHIPPED orders. If 30% of orders are unshipped, you undercount by 30%
-- with no error — a plausibly-wrong number on a dashboard.
```

Why it's wrong at the execution level: `COUNT(expr)`'s transition function is strict, so it is never called for the ~30% of rows with NULL `shipped_at`. The state (the running count) simply doesn't advance for those rows.

```sql
-- RIGHT: count rows with COUNT(*); count shipped ones explicitly with FILTER
SELECT
  COUNT(*)                                 AS order_count,
  COUNT(*) FILTER (WHERE shipped_at IS NOT NULL) AS shipped_count
FROM orders;
```

### Wrong 2 — Expecting 0 (getting NULL) from SUM on the empty set

```sql
-- WRONG: assuming SUM returns 0 when a customer has no orders
SELECT SUM(total_amount) AS lifetime_value FROM orders WHERE customer_id = $1;
-- For a brand-new customer with zero orders, this returns ONE ROW whose value
-- is NULL — not zero rows, and not 0. Downstream code doing arithmetic on it
-- (lifetime_value * 1.1) gets NULL, or in JS `null * 1.1` = 0 (silently wrong),
-- and `null > 100` = false without you realizing why.
```

Why: with zero input rows the `SUM` transition function is never called; the state stays at its NULL initial value; the final function returns NULL.

```sql
-- RIGHT: coalesce to 0 at the SQL boundary so the app always gets a number
SELECT COALESCE(SUM(total_amount), 0) AS lifetime_value
FROM orders WHERE customer_id = $1;
```

### Wrong 3 — `array_agg` / `json_agg` silently keeping NULLs

```sql
-- WRONG: expecting only real tags, getting a NULL element
SELECT array_agg(tag) AS tags
FROM product_tags WHERE product_id = 7;
-- If any row has tag = NULL, the result is e.g. {'sale', NULL, 'new'} —
-- a NULL element sneaks into the array (array_agg does NOT skip NULLs),
-- and downstream `WHERE 'x' = ANY(tags)` or a JSON serializer chokes or
-- shows a spurious null.
```

Why: unlike `SUM`/`string_agg`, `array_agg`'s transition function is **not strict** — it appends every value including NULL.

```sql
-- RIGHT: filter NULLs out of the aggregate input
SELECT COALESCE(array_agg(tag) FILTER (WHERE tag IS NOT NULL), '{}') AS tags
FROM product_tags WHERE product_id = 7;
```

### Wrong 4 — Unordered `string_agg` treated as if ordered

```sql
-- WRONG: building a "chronological" event trail without an internal ORDER BY
SELECT user_id, string_agg(event_type, ' -> ') AS journey
FROM audit_logs
GROUP BY user_id
ORDER BY user_id;                       -- this orders ROWS, not the concatenation!
-- BUG: 'login -> logout -> purchase' comes out in whatever order rows were read
-- (which can differ run to run, and after a plan change like Hash vs Group
-- aggregate). The trail is not chronological.
```

Why: the query-level `ORDER BY user_id` runs at the very end and orders output rows. It has no effect on the order in which `string_agg` concatenated values *inside* each group — that was decided during aggregation, from arbitrary input order.

```sql
-- RIGHT: order the INPUT to the aggregate
SELECT user_id,
       string_agg(event_type, ' -> ' ORDER BY created_at) AS journey
FROM audit_logs
GROUP BY user_id
ORDER BY user_id;
```

### Wrong 5 — `SUM(DISTINCT)` on money

```sql
-- WRONG: trying to total revenue but deduping amounts
SELECT SUM(DISTINCT total_amount) AS revenue FROM orders WHERE status='completed';
-- BUG: three separate $49.99 orders count as $49.99 ONCE. DISTINCT collapses
-- equal amounts before summing. Revenue is massively understated.
```

Why: `DISTINCT` dedupes the input value set before the transition function runs, so identical amounts are summed a single time.

```sql
-- RIGHT: sum every row's amount
SELECT SUM(total_amount) AS revenue FROM orders WHERE status='completed';
```

### Wrong 6 — Fan-out inflating SUM after a join

```sql
-- WRONG: SUM(o.total_amount) after joining one-to-many order_items
SELECT c.id, SUM(o.total_amount) AS revenue
FROM customers c
JOIN orders o       ON o.customer_id = c.id
JOIN order_items oi ON oi.order_id = o.id      -- fans each order out by item count
GROUP BY c.id;
-- BUG: an order with 4 items is duplicated 4 times, so its total_amount is
-- added 4 times. Revenue is inflated by the average items-per-order factor.
-- (This is the Topic 11 fan-out trap meeting aggregation.)
```

```sql
-- RIGHT: sum item line totals, OR pre-aggregate orders before joining items
SELECT c.id, SUM(oi.quantity * oi.unit_price) AS revenue
FROM customers c
JOIN orders o       ON o.customer_id = c.id
JOIN order_items oi ON oi.order_id = o.id
GROUP BY c.id;
-- or, to keep order-level totals, aggregate orders in a subquery first
-- (see Topic 11 §6.8 pre-aggregation pattern).
```

---

## 10. Performance Profile

### 10.1 Cost model by aggregate class

| Aggregate class | Work per row | Memory | Parallel? | Notes |
|-----------------|-------------|--------|-----------|-------|
| `COUNT(*)`, `SUM`, `AVG`, `MIN`, `MAX`, `bool_*` | O(1) transition | O(1) state | **Yes** (partial+combine) | scans dominate; MIN/MAX may index-shortcut |
| `COUNT(DISTINCT)`, any `DISTINCT` agg | O(1) + must dedupe | O(distinct values) | **No** (single node) | sort or hash of full input |
| `array_agg`, `string_agg`, `json_agg` | O(1) append | **O(group size)** growing | Yes, but | buffer per group in work_mem — can balloon/spill |
| `array_agg(... ORDER BY ...)` | append | O(group) + sort | limited | internal sort per group |

### 10.2 Scaling at 1M / 10M / 100M rows

- **`COUNT(*)` / `SUM`** — linear in rows, but **parallelizes** across `max_parallel_workers_per_gather`. A full-table `SUM` on 100M rows is a parallel seq scan: seconds, not minutes, and scales with core count. An `Index Only Scan` (with a fresh visibility map) can count without heap access.
- **`COUNT(DISTINCT col)`** — the painful one. At 100M rows it must dedupe ~100M values in a single process (no parallel path). If the distinct set is large it sorts on disk (spills beyond `work_mem`). This is where you reach for:
  - **`SELECT COUNT(*) FROM (SELECT DISTINCT col FROM t) s`** — sometimes plans better (hash-distinct + count) and can parallelize the distinct.
  - **HyperLogLog** via the `postgresql-hll` extension or `approx_count_distinct` patterns — O(1) memory, ~2% error, for dashboards where exactness isn't required. Turns a 40s exact count into a 300ms estimate.
- **`array_agg` / `json_agg`** — memory scales with **group size**, not row count. 100M rows in tiny groups (a few per group) is fine. 100M rows in *one* group is a multi-GB in-memory value that OOMs. Always bound group size (selective `WHERE`, or `LIMIT` inside a lateral subquery per group).

### 10.3 Index interactions

- **`MIN`/`MAX` on an indexed column, no GROUP BY**: rewritten to `Index Scan ... Limit 1`, O(log N). Free. Composite index `(a, b)` lets `MAX(b) WHERE a = $1` also shortcut.
- **`COUNT(*)` with a `WHERE`** covered by an index: `Index Only Scan` counts matching entries without heap fetches (if all-visible).
- **`GROUP BY col` + aggregate** where `col` is indexed: enables `GroupAggregate` (streaming, low memory) instead of `HashAggregate` (needs a hash table sized to the number of groups). Topic 20 covers this fully.
- **`FILTER` clauses** generally *defeat* the MIN/MAX index shortcut and force a scan.

### 10.4 work_mem tuning for aggregates

- `HashAggregate` (grouping) and any `DISTINCT` aggregate sort use `work_mem`. Under-sizing causes disk spill (`Batches > 1` for hash, external merge sort for `DISTINCT`), 5–10× slower.
- Raise `work_mem` *for the session* running a heavy analytics query rather than globally (it's per-sort-node, per-connection — a global bump multiplies memory across all connections).

```sql
SET LOCAL work_mem = '256MB';   -- within a transaction, just for this heavy query
SELECT COUNT(DISTINCT customer_id), array_agg(DISTINCT region) FROM big_table;
```

---

## 11. Node.js Integration

### 11.1 Grand-total aggregate — always coalesce and cast

```javascript
import pg from 'pg';
const { Pool } = pg;
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

async function getCustomerLifetimeStats(customerId) {
  const { rows } = await pool.query(
    `SELECT
       COUNT(*)                                       AS order_count,
       COALESCE(SUM(total_amount), 0)                 AS lifetime_value,
       ROUND(AVG(total_amount), 2)                    AS avg_order_value,
       MAX(created_at)                                AS last_order_at,
       COUNT(*) FILTER (WHERE status = 'completed')   AS completed_count
     FROM orders
     WHERE customer_id = $1`,
    [customerId]
  );
  // NOTE: this ALWAYS returns exactly one row, even for a customer with zero
  // orders. order_count and completed_count will be 0; avg_order_value and
  // last_order_at will be NULL (empty-set → NULL for AVG/MAX).
  const r = rows[0];
  return {
    orderCount: Number(r.order_count),          // bigint comes back as string!
    lifetimeValue: Number(r.lifetime_value),    // numeric comes back as string!
    avgOrderValue: r.avg_order_value === null ? null : Number(r.avg_order_value),
    lastOrderAt: r.last_order_at,               // null if no orders
    completedCount: Number(r.completed_count),
  };
}
```

**Critical pg gotcha**: `COUNT(*)` returns SQL `bigint`, and `SUM`/`AVG` on numeric return SQL `numeric`. The `pg` driver maps **both to JavaScript strings** (not numbers) to avoid precision loss beyond `Number.MAX_SAFE_INTEGER`. So `r.order_count` is `"42"`, not `42`. Always `Number(...)` (or `BigInt(...)` for genuinely huge counts) at the boundary, and remember `AVG` can be `null`.

### 11.2 Returning a nested JSON document in one round trip

```javascript
async function getRecentOrdersWithItems() {
  const { rows } = await pool.query(
    `SELECT
       o.id,
       o.status,
       o.total_amount::float8 AS total_amount,   -- cast so pg returns a number
       COALESCE(
         jsonb_agg(
           jsonb_build_object('productId', oi.product_id, 'qty', oi.quantity)
           ORDER BY oi.product_id
         ) FILTER (WHERE oi.id IS NOT NULL),
         '[]'::jsonb
       ) AS items
     FROM orders o
     LEFT JOIN order_items oi ON oi.order_id = o.id
     WHERE o.created_at >= NOW() - INTERVAL '1 hour'
     GROUP BY o.id, o.status, o.total_amount
     ORDER BY o.created_at DESC`
  );
  // `items` arrives already parsed as a JS array of objects (pg parses jsonb).
  // No JSON.parse needed, no N+1 query for line items.
  return rows;
}
```

### 11.3 Guarding against unbounded aggregates

```javascript
// BAD idea in disguise: "give me every event for this user as one array"
// If the user has millions of events, json_agg builds a giant value in the DB
// AND ships a giant payload to Node. Always bound it.
async function getUserEventTrail(userId, limit = 500) {
  const { rows } = await pool.query(
    `SELECT string_agg(event_type, ' -> ' ORDER BY created_at) AS trail
     FROM (
       SELECT event_type, created_at
       FROM audit_logs
       WHERE user_id = $1
       ORDER BY created_at DESC
       LIMIT $2            -- bound the input to the aggregate
     ) recent`,
    [userId, limit]
  );
  return rows[0].trail;   // may be NULL if the user has no events — handle it
}
```

**ORM note**: most ORMs handle simple `COUNT`/`SUM`/`AVG` via a `.count()`/`.aggregate()` helper, but `COUNT(DISTINCT)`, `FILTER`, `string_agg`/`array_agg`/`json_agg`, `bool_and`/`bool_or`, and internal `ORDER BY` almost always require the raw-SQL escape hatch. See the next section.

---

## 12. ORM Comparison

### Prisma

**Can it?** Partially. `prisma.order.aggregate({ _count, _sum, _avg, _min, _max })` covers the five basic aggregates and `_count` with `distinct` at the model level. It has **no** native `string_agg`/`array_agg`/`json_agg`, no `FILTER`, no `bool_and`/`bool_or`, and `COUNT(DISTINCT col)` is limited.

```typescript
// Basic aggregates — supported:
const stats = await prisma.order.aggregate({
  where: { customerId, status: 'completed' },
  _count: true,
  _sum: { totalAmount: true },
  _avg: { totalAmount: true },
  _max: { createdAt: true },
});
// stats._sum.totalAmount is NULL (not 0) when no rows match — same empty-set rule.

// Anything richer needs raw SQL:
const rich = await prisma.$queryRaw`
  SELECT
    COUNT(*) FILTER (WHERE status = 'completed') AS completed,
    string_agg(DISTINCT status, ',' ORDER BY status) AS statuses,
    COUNT(DISTINCT customer_id) AS uniques
  FROM orders WHERE created_at >= ${since}`;
```

**Where it breaks**: `FILTER`, `DISTINCT` inside `SUM`/`AVG`, all the `*_agg` functions, ordered aggregation. **Escape hatch**: `$queryRaw`. **Verdict**: fine for basic KPI aggregates; drop to raw for anything conditional or list-building.

### Drizzle

**Can it?** Yes — best-in-class here. Drizzle exposes `count`, `sum`, `avg`, `min`, `max` helpers and lets you drop into the `sql` template for anything else, fully typed.

```typescript
import { sql, eq, and } from 'drizzle-orm';
const result = await db
  .select({
    total: sql<number>`count(*)`.mapWith(Number),
    completed: sql<number>`count(*) filter (where ${orders.status} = 'completed')`.mapWith(Number),
    revenue: sql<number>`coalesce(sum(${orders.totalAmount}), 0)`.mapWith(Number),
    uniques: sql<number>`count(distinct ${orders.customerId})`.mapWith(Number),
    statuses: sql<string>`string_agg(distinct ${orders.status}, ',' order by ${orders.status})`,
  })
  .from(orders)
  .where(gte(orders.createdAt, since));
```

**Where it breaks**: nothing fundamental — `FILTER`, `string_agg`, ordered aggregation all expressible via `sql`. `.mapWith(Number)` is needed to convert the bigint/numeric strings. **Verdict**: use Drizzle confidently; it's the closest to raw SQL with type safety.

### Sequelize

**Can it?** Basic aggregates via `sequelize.fn`. `FILTER` and `*_agg` need `sequelize.literal`.

```javascript
const { fn, col, literal } = require('sequelize');
const rows = await Order.findAll({
  attributes: [
    [fn('COUNT', col('id')), 'order_count'],
    [fn('SUM', col('total_amount')), 'revenue'],
    [literal(`COUNT(*) FILTER (WHERE status = 'completed')`), 'completed'],
    [literal(`string_agg(DISTINCT status, ',' ORDER BY status)`), 'statuses'],
  ],
  where: { createdAt: { [Op.gte]: since } },
  raw: true,
});
```

**Where it breaks**: `fn('COUNT', col('id'))` produces `COUNT(id)` (NULL-skipping!) — for row count you want `fn('COUNT', literal('*'))` or `literal('COUNT(*)')`. This is a subtle correctness trap. `FILTER`, ordered aggregates, `bool_*` all need `literal`. **Verdict**: workable but verbose; use `literal` liberally and watch the `COUNT(col)` vs `COUNT(*)` distinction.

### TypeORM

**Can it?** Via QueryBuilder `select`/`addSelect` with raw expressions.

```typescript
const row = await dataSource.createQueryBuilder()
  .select('COUNT(*)', 'order_count')
  .addSelect(`COUNT(*) FILTER (WHERE status = 'completed')`, 'completed')
  .addSelect('COALESCE(SUM(total_amount), 0)', 'revenue')
  .addSelect('COUNT(DISTINCT customer_id)', 'uniques')
  .addSelect(`string_agg(DISTINCT status, ',' ORDER BY status)`, 'statuses')
  .from('orders', 'o')
  .where('o.created_at >= :since', { since })
  .getRawOne();
```

**Where it breaks**: `getRawOne()`/`getRawMany()` return everything as strings; the `.aggregate` sugar is minimal so you write SQL expressions as strings with no type safety. **Verdict**: fully capable via raw expressions in QueryBuilder; you're basically writing SQL anyway.

### Knex

**Can it?** Yes — thin builder + `knex.raw` covers everything.

```javascript
const row = await knex('orders')
  .where('created_at', '>=', since)
  .count('* as order_count')
  .countDistinct('customer_id as uniques')
  .sum('total_amount as revenue')
  .select(
    knex.raw(`COUNT(*) FILTER (WHERE status = 'completed') AS completed`),
    knex.raw(`string_agg(DISTINCT status, ',' ORDER BY status) AS statuses`)
  )
  .first();
```

**Where it breaks**: `.count()`/`.sum()` return bigint/numeric as strings (parse them); `FILTER`, `*_agg`, `bool_*`, and internal `ORDER BY` go through `knex.raw`. **Verdict**: most transparent; `countDistinct` is a nice built-in, everything else via `raw`.

### Summary Table

| ORM | Basic 5 (`COUNT/SUM/AVG/MIN/MAX`) | `COUNT(DISTINCT)` | `FILTER` | `*_agg` / `bool_*` | Escape hatch |
|-----|-----------------------------------|-------------------|----------|--------------------|--------------|
| Prisma | `aggregate()` | limited | raw | raw | `$queryRaw` |
| Drizzle | helpers + `sql` | `sql` | `sql` | `sql` | `sql` template |
| Sequelize | `fn` | `fn('COUNT', fn('DISTINCT'…))` | `literal` | `literal` | `literal` |
| TypeORM | QueryBuilder select | raw expr | raw expr | raw expr | `getRaw*` |
| Knex | `.count/.sum/.avg/.min/.max` | `.countDistinct` | `knex.raw` | `knex.raw` | `knex.raw` |

Universal truth: **the moment you need `FILTER`, an internal `ORDER BY`, or any `*_agg`, you are writing raw SQL through the ORM.** And every ORM returns `COUNT`/`SUM` as strings — always convert at the boundary.

---

## 13. Practice Exercises

### Exercise 1 — Easy

Given `payments(id, order_id, amount, method, refunded_at)`:

Write one query that returns, for the whole table:
- the number of payment rows
- the number of payments that have been refunded (`refunded_at IS NOT NULL`)
- the number of distinct payment methods used
- the total amount collected, as `0` (not NULL) if the table is empty
- the largest single payment

```sql
-- Write your query here
```

### Exercise 2 — Medium (combines Topic 11 joins + Topic 20 GROUP BY)

Given `customers(id, name)`, `orders(id, customer_id, status, total_amount, created_at)`, and `order_items(id, order_id, product_id, quantity, unit_price)`:

For each customer, in one query, return:
- `customer_name`
- `completed_orders` — count of distinct completed orders (beware fan-out from joining `order_items`)
- `items_revenue` — sum of `quantity * unit_price` over completed orders only (use `FILTER`)
- `product_list` — a comma-separated, alphabetically ordered list of the **distinct** product IDs the customer bought in completed orders

Only include customers with at least one completed order. Order by `items_revenue` descending.

```sql
-- Write your query here
```

### Exercise 3 — Hard (production simulation — the naive answer is wrong)

You need a per-day report over `orders(id, customer_id, status, total_amount, created_at, coupon_code)`:
- `day`
- `orders` — total orders that day
- `paying_customers` — distinct customers who placed a **completed** order that day
- `coupon_usage_rate` — fraction of that day's orders that used a coupon, as a percentage rounded to 1 dp
- `revenue` — completed-order revenue, `0` if none
- `avg_completed_value` — average completed order value; must be `NULL`-safe for days with zero completed orders

A junior wrote `COUNT(coupon_code)/COUNT(*)` for the usage rate and `SUM(total_amount)` for revenue over all statuses. Explain both bugs, then write the correct query. (Hint: integer division, status filtering, and empty-set NULL all bite here.)

```sql
-- Write your query here
```

### Exercise 4 — Interview Simulation

In a single query over `orders(id, customer_id, status, total_amount, created_at)`, produce a one-row "health snapshot":
- `total_orders`
- `completed_pct` — percentage of orders that are completed, 1 dp, correct even if the table is empty (no division-by-zero, sensible result)
- `revenue`
- `median_order_value` — the median completed order value (hint: this needs an ordered-set aggregate, `percentile_cont(0.5) WITHIN GROUP (ORDER BY ...)`)
- `first_order_at`, `last_order_at`
- `any_fraud` — whether any order is `status = 'fraud'`, defaulting to `FALSE` when there are no rows at all

```sql
-- Write your query here
```

---

## 14. Interview Questions

### Q1. Explain the difference between `COUNT(*)`, `COUNT(column)`, and `COUNT(DISTINCT column)`. Why do they return different numbers?

**Junior answer**: "`COUNT(*)` counts everything, `COUNT(column)` counts that column, and `COUNT(DISTINCT column)` counts unique values."

**Principal answer**: "`COUNT(*)` counts rows — it takes no argument, so nothing can be NULL; it's literally the number of rows that survived `WHERE`. `COUNT(column)` counts rows where that column `IS NOT NULL`, because the aggregate's transition function is strict and is simply not invoked for NULL inputs — so any NULLs in the column are skipped and the count is lower. `COUNT(DISTINCT column)` additionally removes duplicates (and NULLs), returning the number of distinct non-NULL values. The performance difference is significant: `COUNT(*)` and `COUNT(column)` are single-pass and parallelizable via partial aggregates combined by the leader, while `COUNT(DISTINCT)` must build a global distinct set — a sort or hash-dedup of the entire input in a single process, which does not parallelize the same way and spills to disk when the distinct set exceeds `work_mem`. For approximate distinct counts at scale I'd reach for HyperLogLog."

**Follow-up the interviewer asks**: "Is `COUNT(1)` faster than `COUNT(*)`?" — *No. The planner treats `COUNT(1)` as a row count because `1` is never NULL; they produce identical plans and timings in PostgreSQL. `COUNT(*)` is the idiomatic form.*

### Q2. A `SUM` query over a customer's orders returns NULL instead of 0 for a customer with no orders, and it broke our billing code. Why, and how do you fix it correctly?

**Junior answer**: "SUM returned NULL because there was no data; wrap it in `COALESCE`."

**Principal answer**: "It's the empty-set rule falling out of the state-machine model. `SUM` starts with a NULL accumulator; its transition function only runs per input row; with zero rows it never runs, so the state stays NULL and the final function returns NULL. Note this returns *one row containing NULL*, not zero rows — so `rows.length` is 1 and only the value is NULL. `COUNT` is the only aggregate that returns 0 here, because its accumulator initializes to 0. The fix is `COALESCE(SUM(total_amount), 0)` at the SQL boundary. I'd also point out that `AVG` behaves the same — NULL on empty — and that `SUM` *skips* NULL rows rather than propagating them, which is a separate behavior people conflate: the empty-set NULL and the mid-set NULL-skip are two different consequences of transition-function strictness."

**Follow-up**: "What if the customer *has* orders but some `total_amount` values are NULL?" — *`SUM` skips those NULL rows (strict transition), so you get the sum of the non-NULL amounts; the empty-set `COALESCE` doesn't trigger because there's at least one row. If NULLs should count as 0, use `SUM(COALESCE(total_amount, 0))`.*

### Q3. How do you build a deterministic, alphabetized comma-separated list of a group's tags, and what's the most common bug people hit?

**Junior answer**: "Use `string_agg(tag, ',')` and add `ORDER BY tag` at the end of the query."

**Principal answer**: "`string_agg(tag, ', ' ORDER BY tag)` — the `ORDER BY` goes *inside* the aggregate call, because it orders the input values being concatenated during aggregation. The classic bug is putting `ORDER BY tag` at the query level instead; that orders the output *rows*, not the concatenation, so the list comes out in arbitrary input order — and worse, it's *incidentally* correct sometimes (small tables, one plan) and then breaks when the data grows or the planner switches from GroupAggregate to HashAggregate. `string_agg` also skips NULLs without inserting a stray delimiter, and returns NULL (not `''`) for an empty group, so I'd `COALESCE` if the app needs an empty string. If I need uniqueness too, `string_agg(DISTINCT tag, ', ' ORDER BY tag)`, remembering the ORDER BY expression must match the DISTINCT expression."

**Follow-up**: "Does `array_agg` behave the same with NULLs?" — *No — `array_agg` and `json_agg` KEEP NULLs (non-strict transition), inserting a NULL element. That's the biggest asymmetry in the aggregate family; you filter them with `FILTER (WHERE x IS NOT NULL)`.*

### Q4. Your dashboard's `COUNT(DISTINCT user_id)` over a 200M-row events table takes 45 seconds. What's happening and what are your options?

**Junior answer**: "Add an index on `user_id`."

**Principal answer**: "An index alone rarely fixes this — `COUNT(DISTINCT)` must materialize the entire distinct set of user_ids, which is a full sort or hash-dedup in a single backend with no parallel partial-aggregate path, spilling to disk once the set exceeds `work_mem`. Options, roughly in order: (1) restructure as `SELECT COUNT(*) FROM (SELECT DISTINCT user_id FROM events WHERE …) s`, which can let the planner parallelize the distinct step and hash it; (2) raise `work_mem` for the session so the dedup stays in memory; (3) if the query is a repeated dashboard read, precompute it — a rollup table or materialized view refreshed periodically; (4) if approximate is acceptable — and for a dashboard it usually is — use HyperLogLog (`postgresql-hll` or `datasketches`), which gives ~2% error in O(1) memory and turns 45s into sub-second, and is mergeable across time buckets. The key insight I'd communicate is that exact distinct-count is fundamentally not free at scale, so the real question is whether the business needs exactness."

**Follow-up**: "Why can plain `COUNT(*)` parallelize but `COUNT(DISTINCT)` can't as easily?" — *Plain `COUNT` has a trivial combine function — each worker's partial count is just summed by the leader. Distinctness isn't locally decomposable that way: a value can appear across multiple workers, so you'd have to merge full distinct sets (which is exactly what HLL sketches make cheap and mergeable).*

---

## 15. Mental Model Checkpoint

1. A table has 100 rows; column `x` is NULL in 40 of them and has only 3 unique non-NULL values among the other 60. What do `COUNT(*)`, `COUNT(x)`, and `COUNT(DISTINCT x)` each return, and why?

2. `SELECT AVG(amount) FROM t` where `amount` is `10, 20, NULL, NULL`. Is the answer 15 or 7.5? What single change would make it the other value?

3. You run `SELECT SUM(total) FROM orders WHERE customer_id = 999999` for a customer that doesn't exist. How many rows come back, and what's in them? How is this different from `COUNT`?

4. Why does `array_agg(tag)` include NULLs in the result array while `string_agg(tag, ',')` silently drops them? Trace it back to the transition function.

5. You wrote `string_agg(name, ',')` and the list is in a different order every time the query runs. Where does the ordering actually get decided, and what clause fixes it — and where exactly does that clause go?

6. Explain why `SUM(DISTINCT price)` is almost always wrong for revenue but `COUNT(DISTINCT customer_id)` is exactly right for "unique buyers." What does `DISTINCT` do to the input in each case?

7. `bool_and(in_stock)` for an order that has *zero* line items returns NULL, not TRUE. Why? What would you write to treat "no items" as "vacuously all in stock"?

---

## 16. Quick Reference Card

```sql
-- ============ COUNT VARIANTS ============
COUNT(*)                -- rows (incl. all-NULL rows); empty set -> 0
COUNT(col)              -- rows where col IS NOT NULL; empty set -> 0
COUNT(DISTINCT col)     -- distinct non-NULL values; forces sort/hash; no parallel
-- COUNT(1) == COUNT(*)  (no perf difference in Postgres)

-- ============ NUMERIC AGGREGATES ============
SUM(col)                -- skips NULL; EMPTY SET -> NULL (use COALESCE(...,0))
                        -- int->bigint, bigint->numeric (overflow-safe widening)
AVG(col)                -- SUM(non-NULL)/COUNT(non-NULL); EMPTY -> NULL; int->numeric
MIN(col) / MAX(col)     -- any orderable type; skip NULL; EMPTY -> NULL
                        -- bare MIN/MAX on indexed col -> Index Scan Limit 1 (free)

-- ============ EMPTY-SET RULE (memorize) ============
-- COUNT ....................... 0
-- EVERYTHING ELSE ............. NULL

-- ============ NULL-INPUT RULE (memorize) ============
-- array_agg / json_agg / jsonb_agg .... KEEP NULLs
-- everything else ..................... SKIP NULLs

-- ============ LIST / COLLECTION AGGREGATES ============
string_agg(col, ',' ORDER BY col)   -- skips NULL; EMPTY -> NULL; ORDER BY is INTERNAL
array_agg(col ORDER BY col)         -- KEEPS NULL elements; EMPTY -> NULL
json_agg(expr ORDER BY k)           -- KEEPS NULL as json null; EMPTY -> NULL
jsonb_agg(expr)                     -- binary json; dedupes/reorders object keys
json_object_agg(key, val)           -- {k:v,...}; NULL key = ERROR
-- empty-array/string safety:
COALESCE(array_agg(x), '{}')
COALESCE(string_agg(x, ','), '')
COALESCE(jsonb_agg(x), '[]'::jsonb)
-- drop NULL inputs:
array_agg(x) FILTER (WHERE x IS NOT NULL)

-- ============ BOOLEAN AGGREGATES ============
bool_and(cond)  -- (every) TRUE iff all non-NULL TRUE; EMPTY -> NULL
bool_or(cond)   --          TRUE iff any non-NULL TRUE; EMPTY -> NULL

-- ============ FILTER (per-aggregate row filter) ============
COUNT(*)          FILTER (WHERE status='completed')
SUM(total_amount) FILTER (WHERE status='completed')
-- modern replacement for SUM(CASE WHEN ... THEN x END)

-- ============ ORDER BY INSIDE AGGREGATE ============
-- matters for: string_agg, array_agg, json_agg (order-sensitive)
-- pointless for: SUM, COUNT, MIN, MAX, bool_* (result unchanged, wasted sort)
-- NOT the same as the query's final ORDER BY (that orders ROWS)

-- ============ PERF RULES OF THUMB ============
-- COUNT/SUM/AVG/MIN/MAX ....... parallelize (partial agg + combine)
-- COUNT(DISTINCT) / DISTINCT agg .. single-process sort/hash; may spill; HLL for approx
-- *_agg buffers grow with GROUP SIZE -> bound it or risk OOM
-- MIN/MAX on indexed col, no GROUP BY -> index shortcut, ~free
-- SET LOCAL work_mem = '256MB'  for heavy DISTINCT/hash-agg queries

-- ============ INTERVIEW ONE-LINERS ============
-- "COUNT(*) counts rows; COUNT(col) counts non-NULLs; COUNT(DISTINCT) dedupes + skips NULL."
-- "Only COUNT returns 0 on empty; everything else returns NULL — COALESCE at the boundary."
-- "Only array_agg/json_agg keep NULLs; the rest skip them."
-- "ORDER BY for a list goes INSIDE the aggregate, not at the query level."
-- "COUNT(1) is not faster than COUNT(*); that's a myth in Postgres."
-- "AVG divides by the non-NULL count, not the row count."

-- ============ pg / Node BOUNDARY ============
-- COUNT -> bigint -> JS string ; SUM/AVG(numeric) -> JS string. Always Number(...).
-- AVG/MAX/SUM can be null on empty set — handle before arithmetic.
```

---

## Connected Topics

- **Topic 20 — GROUP BY Fundamentals** (previous): aggregates without `GROUP BY` collapse to one row; with `GROUP BY` they produce one row per group. `HashAggregate` vs `GroupAggregate`, and how the grouping strategy interacts with `DISTINCT` and ordered aggregates, is the direct predecessor to everything here.
- **Topic 22 — HAVING** (next): filtering *on* aggregate results after they're computed — the group-level `WHERE`. Every `HAVING SUM(...) > x` builds on the aggregate semantics in this topic.
- **Topic 11 — INNER JOIN in Depth**: the fan-out that makes `SUM`/`COUNT` over joined one-to-many tables double-count; `COUNT(DISTINCT)` and pre-aggregation as the fixes.
- **Topic 07 — NULL in Depth**: three-valued logic is *why* strict transition functions skip NULLs and why `AVG` divides by the non-NULL count.
- **MVCC internals**: why `COUNT(*)` must check tuple visibility and cannot be a single stored number.
- **Parallel query & `work_mem`**: partial aggregate + combine function is the mechanism behind parallel `SUM`/`COUNT`; `work_mem` bounds `DISTINCT` dedup and `*_agg` buffers.
- **Window functions (later phase)**: the same aggregate names (`SUM`, `COUNT`, `array_agg`) used as `OVER (...)` window functions compute running/partitioned aggregates *without* collapsing rows — the contrast that makes both concepts click.
