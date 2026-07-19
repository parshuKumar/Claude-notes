# Topic 65 — Array Operations
### SQL Mastery Curriculum — Phase 9: Advanced SQL Patterns

---

## 1. ELI5 — The Analogy That Makes It Click

Imagine every user in your app has a lunchbox. Most database columns are like a single compartment — one sandwich, one number, one date. But a PostgreSQL **array column** is a lunchbox with a row of numbered compartments inside a single lid. Compartment 1 holds "sci-fi", compartment 2 holds "thriller", compartment 3 holds "comedy" — all of it belongs to *one* user, and it all travels together under one lid (one row, one column).

Now the interesting part. Because everything is inside one lid, you can ask questions about the *whole lunchbox at once*:

- "Does this lunchbox **contain** a comedy compartment?" → the `@>` operator.
- "Is everything in this lunchbox **inside** the approved-snacks list?" → the `<@` operator.
- "Do these two lunchboxes **share** any snack at all?" → the `&&` operator.
- "Tape these two lunchboxes together into one longer one." → the `||` operator.

There is a catch that trips up everyone. A lunchbox with three compartments is still **one lunch**. If you want to count how many people like comedy, you can't just count lunchboxes — you have to first tip every lunchbox out onto the table so each snack becomes its own line item. That "tipping out" is `unnest()`, and gathering scattered snacks back into a lunchbox is `array_agg()`.

The deepest lesson, though, is one of judgement, not syntax: a lunchbox is wonderful when the snacks *only ever matter to that one kid* and *never need their own rules*. The moment snacks need prices, expiry dates, supplier records, or you find yourself constantly tipping lunchboxes out to answer questions — you no longer want lunchboxes. You want a proper cafeteria menu: a separate table with one row per snack. Knowing which world you're in is the whole game. Arrays that should have been rows are the single most common self-inflicted wound in PostgreSQL schema design.

---

## 2. Connection to SQL Internals

An array is not a special table hiding inside a column. It is a single **value** — one contiguous binary blob — stored inline in the row's heap tuple, exactly like a `text` or `numeric` value. This one fact explains almost everything about array performance and array pain.

At the storage layer:

1. **TOAST.** A heap tuple must fit within an 8KB page. An array of a few hundred integers is small and lives inline. An array of ten thousand elements, or an array of long text values, exceeds the ~2KB inline threshold and gets pushed out to the **TOAST** table (The Oversized-Attribute Storage Technique), compressed and sliced into chunks. Every read of that column then costs an extra TOAST fetch and decompression. A big array is a big value, and big values are slow to read, slow to update, and expensive in the buffer pool.

2. **MVCC rewrites the whole value.** PostgreSQL has no in-place array mutation on disk. Appending one element with `arr || 5` does not patch byte 41 of the tuple — it writes an entirely new tuple version (MVCC — see Topic 46) containing the full new array, and marks the old tuple dead for `VACUUM` to reclaim. Updating one element of a 5,000-element array rewrites all 5,000 elements plus TOAST churn. This is why "just append to the array" is a scaling trap that a junction table never has.

3. **GIN indexes make containment fast.** A plain B-tree on an array column indexes the array *as a whole opaque value* — useful only for whole-array equality, useless for "contains element X". To index *membership*, PostgreSQL uses a **GIN** index (Generalized Inverted Index). GIN inverts the relationship: instead of "row → array of elements", it stores "element → posting list of rows containing it", exactly like a search-engine inverted index. That is what makes `tags @> ARRAY['sql']` an index scan instead of a full sequential scan over every row.

4. **The planner and `ANY`.** `col = ANY($1)` where `$1` is an array is how PostgreSQL expresses an IN-list against a bound parameter. The planner treats `= ANY(array)` almost identically to `IN (list)`: with a B-tree on `col` it can produce an index scan (often a `BitmapOr` of per-value probes); without one it is a filter over a scan. Understanding this equivalence is what lets a Node.js developer pass one `int[]` parameter instead of building `IN ($1,$2,$3,...)` by hand.

5. **`unnest()` is a set-returning function.** `unnest()` runs in the executor as a function that emits N rows from one input row — it materialises the fan-out that a normalised table would have had physically. The array was the compressed form; `unnest()` decompresses it back into relational rows so that `GROUP BY`, `JOIN`, and aggregates can operate.

Hold these five facts. Every gotcha below is a direct consequence of "an array is one value in one tuple."

---

## 3. Logical Execution Order Context

```
FROM / JOIN            ← unnest() in FROM runs here, as a set-returning function
  (LATERAL unnest)     ← array is expanded into rows before WHERE sees them
WHERE                  ← @>, <@, &&, = ANY(array) filter rows here
GROUP BY               ← array_agg() collects grouped rows into an array here
HAVING
SELECT                 ← array_agg() / array constructors in the target list run here
DISTINCT
ORDER BY               ← array_agg(... ORDER BY ...) internal ordering resolved here
LIMIT
```

Array operations appear in three distinct phases, and confusing them is a frequent bug source:

- **`unnest()` in the `FROM` clause** (usually via `LATERAL`) runs during the `FROM`/`JOIN` phase, *before* `WHERE`. This means a `WHERE` predicate can filter the already-expanded rows. If you `unnest` tags and then `WHERE tag = 'sql'`, the expansion happens first and the filter runs on the exploded rows.

- **Containment operators (`@>`, `<@`, `&&`) in `WHERE`** run in the normal `WHERE` phase, operating on the array as a whole value — no expansion. `WHERE tags @> ARRAY['sql']` never explodes the row; it tests the intact array. This is the fast path and the one a GIN index accelerates.

- **`array_agg()` in `SELECT`/`GROUP BY`** runs late, during aggregation — it is the inverse of `unnest`, collapsing many grouped rows back into one array per group. `array_agg(x ORDER BY y)` accepts an inner `ORDER BY` that is resolved during aggregation, independent of the query's final `ORDER BY`.

The mental rule: **`@>`/`<@`/`&&`/`= ANY` keep the row intact and filter (WHERE phase); `unnest` explodes rows early (FROM phase); `array_agg` collects rows late (aggregation phase).** Remember from Topic 20 that `GROUP BY` runs after `WHERE` and before `SELECT` — `array_agg` lives in that same aggregation window.

---

## 4. What Is an Array Column?

A PostgreSQL array is an ordered, 1-indexed, homogeneous collection of values of a single element type, stored as one value in one column of one row. Any base type, enum, or composite type can have an array form, written by appending `[]` to the type name. Arrays can be multi-dimensional, though single-dimension is by far the most common in practice.

```sql
CREATE TABLE products (
  id          bigint PRIMARY KEY,
  name        text NOT NULL,
  tags        text[]        DEFAULT '{}',      -- 1-D array of text, default empty (NOT null)
  price_tiers numeric(10,2)[],                 -- array of fixed-precision numerics
  dimensions  integer[]                        -- e.g. {width, height, depth}
);
--            │            │                   │
--            │            │                   └── no DEFAULT → column is NULL until set
--            │            └── numeric(10,2)[] : element type carries its own typmod
--            └── text[] : shorthand for the internal type _text; [] means "array of"
```

### Array Literals and Constructors

There are two syntaxes to build an array, and they are **not** interchangeable in every context.

```sql
-- 1. ARRAY constructor  (SQL expression — evaluates its elements)
SELECT ARRAY[1, 2, 3];                     -- {1,2,3}
SELECT ARRAY['sql', 'db', lower('PG')];    -- {sql,db,pg}  ← expressions allowed
SELECT ARRAY[]::text[];                    -- empty array — cast REQUIRED (no type to infer)

-- 2. Array literal  (a text constant, parsed as an array — NO expression evaluation)
SELECT '{1,2,3}'::int[];                    -- {1,2,3}
SELECT '{sql,db,pg}'::text[];               -- {sql,db,pg}
SELECT '{"has,comma","has space"}'::text[]; -- elements with commas/spaces need double quotes
SELECT '{}'::int[];                         -- empty array literal
SELECT '{NULL,2,3}'::int[];                 -- NULL element (unquoted NULL keyword)
SELECT '{"NULL"}'::text[];                  -- the literal STRING 'NULL', not a null element
```

Key distinctions annotated:

```sql
ARRAY[a, b, c]        -- constructor: a,b,c are EVALUATED as SQL expressions
--    │
--    └── use when elements come from columns, functions, or params

'{a,b,c}'::type[]     -- literal: parsed as raw text, curly-brace format
--    │        │
--    │        └── cast is what tells the parser the element type
--    └── use for constants; watch quoting for commas, spaces, NULL, empty string
```

### Element Type and Dimensionality

```sql
SELECT array_ndims('{1,2,3}'::int[]);          -- 1  (number of dimensions)
SELECT array_ndims('{{1,2},{3,4}}'::int[]);    -- 2
SELECT array_length('{1,2,3}'::int[], 1);      -- 3  (length along dimension 1)
SELECT array_length('{}'::int[], 1);           -- NULL, not 0  ← classic gotcha
SELECT cardinality('{1,2,3}'::int[]);          -- 3  (total elements across all dims)
SELECT cardinality('{}'::int[]);               -- 0  ← use cardinality for a real count
```

`cardinality()` returns 0 for an empty array; `array_length(arr, 1)` returns **NULL** for an empty array. This asymmetry is the source of countless off-by-NULL bugs — prefer `cardinality()` when you want a length that behaves numerically.

---

## 5. Why Array Mastery Matters in Production

1. **The IN-list-from-a-parameter problem.** Every Node.js developer eventually needs "give me all rows whose id is in this JavaScript list." Building `IN ($1,$2,$3,…$n)` by string-concatenating placeholders is fragile, breaks the prepared-statement cache (a different N is a different query text), and invites injection if done sloppily. `WHERE id = ANY($1::bigint[])` passes the entire list as **one** parameter, one stable query text, fully parameterised. Not knowing this pattern leads to real production incidents.

2. **Silent NULL and empty-array wrong answers.** `WHERE tag = ANY(tags)` behaves differently for an empty array (matches nothing) versus a NULL array (returns NULL → row excluded). `array_length(tags,1)` returning NULL instead of 0 breaks pagination math and dashboard counts. These are not crashes — they are silently wrong numbers that ship to production and corrupt reports.

3. **The unindexed containment scan.** `WHERE tags @> ARRAY['premium']` without a GIN index is a sequential scan that de-TOASTs and re-parses the array of every row. At 100 rows it's invisible; at 10M rows it's a 20-second query that pages the on-call engineer. Knowing that containment needs GIN — and that `= ANY` needs B-tree on the *element*, not GIN — is the difference between 2ms and 20s.

4. **The MVCC append trap.** A column like `event_ids bigint[]` that grows by one element per request looks elegant and is a scaling time bomb: every append rewrites the entire array tuple, TOAST included, and generates dead tuples that bloat the table and starve autovacuum. Teams discover this at 3am when a "small" table is 40GB of bloat. Recognising that an ever-growing array should have been a child table is production-critical judgement.

5. **Fan-out and aggregation correctness.** `unnest()` multiplies rows; forgetting that when you also join produces silent row-multiplication bugs (Topic 11's fan-out, now with arrays). Conversely, `array_agg` without an explicit `ORDER BY` produces arrays in an undefined element order that changes between runs — breaking any downstream code that assumes `tags[1]` is stable.

---

## 6. Deep Technical Content

### 6.1 The Containment Operators — `@>`, `<@`, `&&`

These three operators treat arrays as **sets** (ignoring order and duplicates for the purpose of the test) and are the workhorses of array querying.

```sql
-- @>  "left contains right"  : does the left array include ALL elements of the right?
SELECT ARRAY[1,2,3,4] @> ARRAY[2,4];      -- true  (both 2 and 4 present)
SELECT ARRAY[1,2,3]   @> ARRAY[2,5];      -- false (5 not present)
SELECT ARRAY[1,2,3]   @> ARRAY[]::int[];  -- true  (everything contains the empty set)

-- <@  "left is contained by right" : is EVERY left element present in the right?
SELECT ARRAY[2,4]     <@ ARRAY[1,2,3,4];  -- true
SELECT ARRAY[]::int[] <@ ARRAY[1,2,3];    -- true  (empty set is subset of anything)

-- &&  "overlap"  : do the arrays share AT LEAST ONE element?
SELECT ARRAY[1,2,3] && ARRAY[3,4,5];      -- true  (share 3)
SELECT ARRAY[1,2]   && ARRAY[3,4];        -- false (disjoint)
SELECT ARRAY[1,2]   && ARRAY[]::int[];    -- false (nothing overlaps the empty set)
```

Semantic subtleties that bite in production:

- **Order and duplicates are ignored** by all three. `ARRAY[1,1,2] @> ARRAY[2,1]` is true; multiplicity is not checked. These test set membership, not multiset equality.
- **The empty set is the universal subset.** `X @> '{}'` is always true and `'{}' <@ X` is always true. If you build `@>` conditions dynamically and the right side can be empty, every row matches — a common "why is my filter returning everything?" bug.
- **NULL arrays poison the result.** `NULL @> ARRAY[1]` is NULL (not false), so the row is excluded from a `WHERE`. Guard with `COALESCE(tags,'{}')` or a `NOT NULL DEFAULT '{}'` column.
- **NULL *elements* are not found by containment.** `ARRAY[1,NULL,3] @> ARRAY[NULL::int]` is NULL, not true — you cannot test for a NULL element with `@>`. Use `array_position(arr, NULL)` is also unreliable; use `EXISTS (SELECT 1 FROM unnest(arr) e WHERE e IS NULL)`.

### 6.2 Concatenation and Element Manipulation — `||`

The `||` operator concatenates arrays, or prepends/appends a single element.

```sql
SELECT ARRAY[1,2] || ARRAY[3,4];   -- {1,2,3,4}   (array || array)
SELECT ARRAY[1,2] || 3;            -- {1,2,3}     (array || element → append)
SELECT 0 || ARRAY[1,2];            -- {0,1,2}     (element || array → prepend)
SELECT ARRAY[1,2] || NULL;         -- {1,2}       (NULL element is a no-op here... careful)
SELECT NULL::int[] || ARRAY[1,2];  -- {1,2}       (NULL array treated as empty in ||)
```

Related mutation helpers (all return a *new* array — nothing mutates in place):

```sql
SELECT array_append(ARRAY[1,2], 3);        -- {1,2,3}
SELECT array_prepend(0, ARRAY[1,2]);       -- {0,1,2}
SELECT array_cat(ARRAY[1,2], ARRAY[3,4]);  -- {1,2,3,4}
SELECT array_remove(ARRAY[1,2,3,2], 2);    -- {1,3}      (removes ALL matching elements)
SELECT array_replace(ARRAY[1,2,3], 2, 9);  -- {1,9,3}    (replaces ALL matching)
SELECT array_position(ARRAY['a','b','c'], 'b');  -- 2    (first index, 1-based; NULL if absent)
SELECT array_positions(ARRAY[1,2,1,2], 2);       -- {2,4}(all indexes)
```

To remove duplicates or sort, round-trip through `unnest`:

```sql
-- Deduplicate an array (order NOT preserved)
SELECT ARRAY(SELECT DISTINCT unnest(ARRAY[3,1,2,1,3]));   -- {1,2,3} (some order)

-- Sort an array
SELECT ARRAY(SELECT unnest(ARRAY[3,1,2]) ORDER BY 1);     -- {1,2,3}

-- Deduplicate AND sort deterministically
SELECT ARRAY(SELECT DISTINCT unnest(ARRAY[3,1,2,1]) ORDER BY 1);  -- {1,2,3}
```

### 6.3 `ANY` and `ALL` — The IN-List Pattern

`ANY` and `ALL` compare a scalar against every element of an array using a chosen operator. This is the single most important pattern for Node.js developers.

```sql
-- = ANY : scalar equals AT LEAST ONE element  → this is exactly "IN (list)"
SELECT 3 = ANY(ARRAY[1,2,3]);     -- true
SELECT 9 = ANY(ARRAY[1,2,3]);     -- false
WHERE id = ANY(ARRAY[10,20,30]);  -- equivalent to  WHERE id IN (10,20,30)

-- Other operators work too:
SELECT 5 > ALL(ARRAY[1,2,3]);     -- true  (5 greater than every element)
SELECT 5 > ANY(ARRAY[1,2,9]);     -- true  (greater than at least one)
SELECT 'a%' = ANY(ARRAY['ab','cd']);        -- false (= is exact, not LIKE)
SELECT EXISTS (SELECT 1 FROM unnest(ARRAY['ab','cd']) t WHERE t LIKE 'a%'); -- pattern match

-- <> ALL : scalar differs from EVERY element  → this is "NOT IN (list)"
SELECT 5 <> ALL(ARRAY[1,2,3]);    -- true  (5 is not in the list)
```

Critical NULL and empty-set semantics — memorise these, they cause the worst bugs:

```sql
SELECT 3 = ANY('{}'::int[]);        -- false  (nothing to match → false)
SELECT 3 <> ALL('{}'::int[]);       -- TRUE   (vacuously true — no element to contradict)
SELECT 3 = ANY(NULL::int[]);        -- NULL   (unknown → row excluded in WHERE)
SELECT 3 = ANY(ARRAY[1,NULL,3]);    -- true   (found in a non-null element)
SELECT 9 = ANY(ARRAY[1,NULL,3]);    -- NULL   (not found, and a NULL present → UNKNOWN)
SELECT 9 <> ALL(ARRAY[1,NULL,3]);   -- NULL   (the NULL contaminates NOT IN, exactly like SQL NOT IN)
```

That last pair is the array version of the infamous `NOT IN (subquery-with-NULL)` trap from Topic 27: a single NULL element turns `<> ALL` into NULL for non-matching scalars, silently dropping rows. When the array may contain NULLs and you need NOT-IN semantics, strip them first: `x <> ALL(array_remove(arr, NULL))`.

### 6.4 `unnest()` — Exploding Arrays into Rows

`unnest()` is a set-returning function that turns one array into one row per element.

```sql
SELECT unnest(ARRAY['a','b','c']);   -- 3 rows: 'a', 'b', 'c'

-- WITH ORDINALITY : also get the 1-based position (order is preserved!)
SELECT * FROM unnest(ARRAY['x','y','z']) WITH ORDINALITY AS t(val, pos);
--  val | pos
--  ----+----
--  x   |  1
--  y   |  2
--  z   |  3

-- Unnest multiple arrays in parallel (zips them; shorter is padded with NULL)
SELECT * FROM unnest(ARRAY['a','b'], ARRAY[1,2,3]) AS t(letter, num);
--  letter | num
--  -------+----
--  a      |  1
--  b      |  2
--  (null) |  3   ← parallel unnest pads the shorter array with NULLs
```

The dominant production pattern is `LATERAL unnest` to expand a column per-row:

```sql
-- One row per (product, tag) pair — the array is flattened relationally
SELECT p.id, p.name, t.tag
FROM products p
CROSS JOIN LATERAL unnest(p.tags) AS t(tag);
-- A product with 3 tags produces 3 rows. Products with empty tags[] produce
-- ZERO rows (CROSS JOIN LATERAL drops them) — use LEFT JOIN LATERAL to keep them:

SELECT p.id, p.name, t.tag
FROM products p
LEFT JOIN LATERAL unnest(p.tags) AS t(tag) ON TRUE;
-- Products with empty/NULL tags now appear once with tag = NULL.
```

The empty-array-drops-the-row behaviour of `CROSS JOIN LATERAL unnest` mirrors INNER JOIN's row-dropping from Topic 11: no elements means no matching rows means the parent row vanishes. Reach for `LEFT JOIN LATERAL ... ON TRUE` whenever you must preserve rows with empty arrays.

### 6.5 `array_agg()` — Collecting Rows into Arrays

`array_agg()` is the inverse aggregate: it gathers a column's values across a group into one array.

```sql
-- Collect all tag rows back into one array per product
SELECT product_id, array_agg(tag) AS tags
FROM product_tags
GROUP BY product_id;

-- ORDER the elements deterministically (ESSENTIAL for reproducible arrays)
SELECT product_id, array_agg(tag ORDER BY tag) AS tags
FROM product_tags
GROUP BY product_id;

-- DISTINCT inside the aggregate to dedupe
SELECT order_id, array_agg(DISTINCT product_id ORDER BY product_id) AS product_ids
FROM order_items
GROUP BY order_id;

-- FILTER clause to conditionally include elements
SELECT customer_id,
       array_agg(id) FILTER (WHERE status = 'completed') AS completed_order_ids,
       array_agg(id) FILTER (WHERE status = 'refunded')  AS refunded_order_ids
FROM orders
GROUP BY customer_id;
```

Two non-obvious behaviours:

- **`array_agg` skips nothing by default** — it includes NULL inputs as NULL elements. `array_agg(x)` over rows where `x` is sometimes NULL yields an array with NULL elements. Use `array_agg(x) FILTER (WHERE x IS NOT NULL)` to exclude them.
- **Without `ORDER BY`, element order is undefined.** It happens to follow input order in simple sequential scans but is *not guaranteed* and changes under parallel plans or different scan orders. Always add `ORDER BY` inside `array_agg` if downstream code indexes into the array.

### 6.6 Subscripting and Slicing

Arrays are 1-indexed. Out-of-bounds access returns NULL, never an error.

```sql
SELECT (ARRAY['a','b','c'])[1];        -- 'a'   (1-based!)
SELECT (ARRAY['a','b','c'])[0];        -- NULL  (no zero index; NOT an error)
SELECT (ARRAY['a','b','c'])[9];        -- NULL  (out of bounds → NULL)
SELECT (ARRAY['a','b','c'])[2:3];      -- {b,c} (slice — inclusive range)
SELECT (ARRAY['a','b','c'])[:2];       -- {a,b} (slice from start)
SELECT (ARRAY['a','b','c'])[2:];       -- {b,c} (slice to end)

-- On a column:
SELECT id, dimensions[1] AS width, dimensions[2] AS height FROM products;

-- Assignment via subscript in UPDATE (rewrites the whole array value — MVCC!)
UPDATE products SET tags[1] = 'featured' WHERE id = 5;
-- Assigning past the end auto-extends, filling gaps with NULL:
UPDATE products SET tags[5] = 'x' WHERE id = 5;  -- indexes 2,3,4 become NULL if unset
```

Because a scalar subscript on an out-of-bounds index yields NULL rather than erroring, `arr[i]` is safe to use without a length guard — but a NULL result may propagate silently. Distinguish "element is genuinely NULL" from "index out of range" using `cardinality(arr) >= i`.

### 6.7 Multi-Dimensional Arrays and Their Sharp Edges

PostgreSQL arrays can be multi-dimensional, but every "row" must have the same length — they are true rectangular matrices, not jagged arrays.

```sql
SELECT '{{1,2,3},{4,5,6}}'::int[];              -- 2x3 matrix
SELECT ('{{1,2,3},{4,5,6}}'::int[])[2][3];      -- 6  (row 2, col 3)
SELECT array_dims('{{1,2},{3,4}}'::int[]);      -- [1:2][1:2]
-- Jagged arrays are ILLEGAL:
SELECT '{{1,2},{3,4,5}}'::int[];  -- ERROR: multidimensional arrays must have
                                  -- array expressions with matching dimensions
```

Multi-dimensional arrays are rare in application schemas and awkward to query (you cannot GIN-index them for element containment the same way). Prefer normalisation or `jsonb` (Topic 64) for genuinely nested or ragged structures.

### 6.8 GIN Indexes on Arrays

A GIN index makes `@>`, `<@`, `&&`, and `= ANY` (element membership) fast by inverting the array into an element→rows posting-list structure.

```sql
-- Default GIN opclass for arrays supports @>, <@, &&, and = ANY
CREATE INDEX idx_products_tags_gin ON products USING GIN (tags);

-- Now this is an index scan instead of a seq scan:
SELECT * FROM products WHERE tags @> ARRAY['premium'];
SELECT * FROM products WHERE tags && ARRAY['sale','clearance'];  -- overlap
```

What GIN does and does not accelerate:

- **Accelerates:** `tags @> ARRAY[...]` (contains), `tags && ARRAY[...]` (overlap), `tags <@ ARRAY[...]` (contained-by, but see caveat), and `'x' = ANY(tags)` (membership of one element).
- **`<@` caveat:** contained-by can be checked by GIN but is often *not* selective — asking "is this row's array a subset of {...}" frequently matches many rows (especially short arrays and the empty array), so the planner may still choose a seq scan.
- **Does NOT accelerate:** `array_length`, subscript access `tags[1] = 'x'`, ordering, or `=` whole-array equality (that wants a B-tree).
- **`gin__int_ops` (from the `intarray` extension)** offers a more compact, faster GIN variant specifically for `int[]` with no NULLs — worth it for large integer-array workloads.

GIN trade-offs: GIN indexes are **slower to update** than B-trees (each element is a separate index entry) and use the `fastupdate` pending-list mechanism to batch inserts. For write-heavy array columns, GIN maintenance cost is real — another nudge toward a junction table when writes dominate.

### 6.9 The `= ANY($1)` Parameter Pattern vs `@>`

Two different questions that beginners conflate:

```sql
-- Q1: "is my row's scalar column one of these values?"  → = ANY on a SCALAR column
--     needs a B-tree on the scalar column
WHERE id = ANY($1::bigint[])        -- IN-list; $1 is the array of candidate ids

-- Q2: "does my row's ARRAY column contain any/all of these values?" → @> / && on ARRAY column
--     needs a GIN index on the array column
WHERE tags @> $1::text[]            -- row's tags contain ALL of $1
WHERE tags && $1::text[]            -- row's tags overlap $1 (any)
```

`= ANY($1)` scans a scalar column and matches against the parameter array (B-tree territory). `@>`/`&&` scan an array column and match against the parameter array (GIN territory). Passing the array to the wrong side, or expecting GIN to help `id = ANY(...)` on a scalar column, is a frequent misdiagnosis.

### 6.10 When Arrays Solve Real Problems vs When to Normalise

The single most important non-syntactic skill. Use an array column when **all** of the following hold:

- The elements are simple scalars with no attributes of their own (no price, no timestamp, no status per element).
- You almost always read/write the whole set together, atomically, as part of the parent row.
- The set is small and bounded (dozens, not thousands) and does not grow unboundedly over time.
- You rarely need to join, aggregate, or filter *across* rows by individual element in ways GIN can't serve.
- Elements are never referenced by foreign keys and have no referential integrity requirements.

Good array fits: a handful of `tags`, a fixed `dimensions` triple, a small set of `permission_flags`, a `preferred_locales` list.

Normalise into a child/junction table when **any** of these appear:

- Elements need their own columns (price, added_at, added_by) → you're modelling a relationship, use a row.
- The set grows unboundedly (event ids, audit entries) → MVCC rewrite cost and TOAST bloat will hurt.
- You need foreign keys, per-element constraints, or referential integrity.
- You frequently aggregate/join by element (top-N tags across all products, counts per element) — a junction table with a B-tree is simpler and faster than perpetual `unnest`.
- Multiple writers append concurrently — array append has no per-element locking; a child table row does.

The tell: **if you find yourself `unnest`-ing the array in most queries, it wanted to be rows.** Arrays are a denormalisation for read-mostly, whole-value, attribute-free small sets. The moment any of those adjectives fails, reach for the junction table — this is Topic 11's relational model doing what it does best.

---

## 7. EXPLAIN — Array Operators in the Plan

### GIN Index Scan for `@>` (containment)

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, name FROM products WHERE tags @> ARRAY['premium','sale'];
```

```
Bitmap Heap Scan on products  (cost=28.05..420.13 rows=95 width=32)
                              (actual time=0.21..0.74 rows=88 loops=1)
  Recheck Cond: (tags @> '{premium,sale}'::text[])
  Heap Blocks: exact=71
  ->  Bitmap Index Scan on idx_products_tags_gin  (cost=0.00..28.03 rows=95 width=0)
                              (actual time=0.18..0.18 rows=88 loops=1)
        Index Cond: (tags @> '{premium,sale}'::text[])
  Buffers: shared hit=79
Planning Time: 0.14 ms
Execution Time: 0.81 ms
```

**Reading it:**
- `Bitmap Index Scan on idx_products_tags_gin` — GIN answered the containment test, producing a bitmap of candidate heap pages. GIN scans almost always surface as Bitmap (not plain Index) scans.
- `Recheck Cond` — GIN is *lossy* at the page level; PostgreSQL re-tests `@>` on each heap tuple to discard false positives from the bitmap. Normal and expected.
- `Heap Blocks: exact=71` — 71 heap pages visited. Buffers `shared hit=79` all cache hits — fast.

### Seq Scan Fallback — no GIN index

```sql
-- Same query, but the GIN index has been dropped
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, name FROM products WHERE tags @> ARRAY['premium','sale'];
```

```
Seq Scan on products  (cost=0.00..24500.00 rows=95 width=32)
                      (actual time=0.03..842.11 rows=88 loops=1)
  Filter: (tags @> '{premium,sale}'::text[])
  Rows Removed by Filter: 999912
  Buffers: shared hit=1240 read=18760
Planning Time: 0.10 ms
Execution Time: 843.02 ms
```

**Reading it:**
- `Seq Scan` + `Filter: (tags @> ...)` — every one of 1,000,000 rows is read, de-TOASTed, and containment-tested.
- `Rows Removed by Filter: 999912` — 99.99% of work discarded. This is the exact symptom that a GIN index cures (843ms → 0.8ms above, a 1000x difference).
- `Buffers: read=18760` — 18,760 pages read from disk. The array de-TOAST amplifies I/O.

### `= ANY($1)` on a scalar column — B-tree Bitmap

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, total_amount FROM orders WHERE id = ANY('{101,205,377,890,1204}'::bigint[]);
```

```
Bitmap Heap Scan on orders  (cost=21.50..42.30 rows=5 width=16)
                            (actual time=0.03..0.05 rows=5 loops=1)
  Recheck Cond: (id = ANY('{101,205,377,890,1204}'::bigint[]))
  Heap Blocks: exact=5
  ->  Bitmap Index Scan on orders_pkey  (cost=0.00..21.50 rows=5 width=0)
                            (actual time=0.02..0.02 rows=5 loops=1)
        Index Cond: (id = ANY('{101,205,377,890,1204}'::bigint[]))
  Buffers: shared hit=17
Planning Time: 0.09 ms
Execution Time: 0.07 ms
```

**Reading it:**
- The B-tree PK handles `= ANY(array)` directly as an `Index Cond` — the planner probes the index once per array element and ORs the results into a bitmap. This is precisely the `IN (list)` plan, driven by one bound parameter.
- No GIN needed here — the array is the *parameter*, not the indexed column. The scalar `id` column's B-tree does the work.

### `unnest` with `LATERAL` — Set-Returning Function in the plan

```sql
EXPLAIN (ANALYZE)
SELECT p.id, t.tag
FROM products p
CROSS JOIN LATERAL unnest(p.tags) AS t(tag)
WHERE p.category_id = 7;
```

```
Nested Loop  (cost=0.29..3150.00 rows=5000 width=40)
             (actual time=0.04..6.12 rows=4820 loops=1)
  ->  Index Scan using idx_products_category on products p
        (cost=0.29..410.00 rows=1000 width=64) (actual rows=1000 loops=1)
        Index Cond: (category_id = 7)
  ->  Function Scan on unnest t  (cost=0.00..2.50 rows=5 width=32)
        (actual time=0.001..0.002 rows=5 loops=1000)
Planning Time: 0.12 ms
Execution Time: 6.55 ms
```

**Reading it:**
- `Function Scan on unnest t` nested inside a `Nested Loop` — for each of the 1,000 filtered products, `unnest` fires once (`loops=1000`) and emits ~5 tag rows.
- Output `rows=4820` ≈ 1000 products × ~4.8 tags — the fan-out is visible in the row count. This is the relational explosion of the array made physical.

---

## 8. Query Examples

### Example 1 — Basic: Tag Containment and Membership

```sql
-- Products tagged BOTH 'premium' AND 'wireless' (containment — all present)
SELECT id, name, tags
FROM products
WHERE tags @> ARRAY['premium','wireless'];

-- Products tagged with AT LEAST ONE of these (overlap)
SELECT id, name, tags
FROM products
WHERE tags && ARRAY['sale','clearance','discount'];

-- Products whose tag list INCLUDES 'featured' (single-element membership)
SELECT id, name
FROM products
WHERE 'featured' = ANY(tags);          -- equivalent to tags @> ARRAY['featured']
```

### Example 2 — Intermediate: Explode, Aggregate, Re-collect

```sql
-- Top 10 most-used tags across all products, with product counts.
-- unnest explodes each product's tags; GROUP BY counts occurrences.
SELECT t.tag, COUNT(*) AS product_count
FROM products p
CROSS JOIN LATERAL unnest(p.tags) AS t(tag)
GROUP BY t.tag
ORDER BY product_count DESC, t.tag
LIMIT 10;

-- Inverse: rebuild a deduplicated, sorted tag array per category
SELECT p.category_id,
       array_agg(DISTINCT t.tag ORDER BY t.tag) AS category_tags
FROM products p
CROSS JOIN LATERAL unnest(p.tags) AS t(tag)
GROUP BY p.category_id;
```

### Example 3 — Production Grade: Bulk Lookup + Containment Report

```sql
-- Scenario: an admin dashboard fetches a specific set of products by id
-- (passed as one array parameter from Node.js) and, for each, flags whether
-- it carries any of the currently-active promotion tags.
--
-- Table sizes:   products ~2,000,000 rows, avg 6 tags each
-- Indexes:       products_pkey (B-tree on id), idx_products_tags_gin (GIN on tags)
-- Expectation:   sub-millisecond — PK bitmap for the id set, GIN unused here because
--                the row set is already tiny (planner tests @> as a cheap filter).

SELECT
  p.id,
  p.name,
  p.tags,
  p.tags && ARRAY['flash_sale','doorbuster','bogo'] AS has_active_promo,
  cardinality(p.tags)                               AS tag_count
FROM products p
WHERE p.id = ANY($1::bigint[])          -- $1 = [1001, 2050, 3399, ...] from the app
ORDER BY p.id;
```

```
EXPLAIN (ANALYZE, BUFFERS) for $1 with 50 ids:

Sort  (cost=182.40..182.53 rows=50 width=80) (actual time=0.19..0.20 rows=50 loops=1)
  Sort Key: id
  Sort Method: quicksort  Memory: 32kB
  ->  Bitmap Heap Scan on products p  (cost=101.2..181.0 rows=50 width=80)
                                      (actual time=0.06..0.14 rows=50 loops=1)
        Recheck Cond: (id = ANY($1::bigint[]))
        Heap Blocks: exact=50
        ->  Bitmap Index Scan on products_pkey
              (actual time=0.04..0.04 rows=50 loops=1)
              Index Cond: (id = ANY($1::bigint[]))
  Buffers: shared hit=204
Planning Time: 0.16 ms
Execution Time: 0.24 ms
```

The `&&` overlap test runs as a cheap per-row filter on the 50 already-fetched rows — no GIN scan needed because the PK bitmap already reduced the candidate set to 50. This is the planner correctly declining an index it doesn't need.

---

## 9. Wrong → Right Patterns

### Wrong 1: Building an IN-list by concatenating placeholders

```sql
-- WRONG (in the app): dynamically built SQL, one placeholder per id
--   const sql = `SELECT * FROM orders WHERE id IN (${ids.map((_,i)=>'$'+(i+1)).join(',')})`;
--   → query text CHANGES with every list length: "IN ($1)", "IN ($1,$2)", ...
SELECT * FROM orders WHERE id IN ($1, $2, $3, $4, $5);
-- Problems at the execution level:
--   1. Each distinct N is a NEW query text → prepared-statement cache thrashes,
--      planner re-plans for every list size.
--   2. Empty list produces "IN ()" → SYNTAX ERROR.
--   3. Hand-built string joins invite injection if any element isn't parameterised.

-- RIGHT: one array parameter, one stable query text, any length incl. empty
SELECT * FROM orders WHERE id = ANY($1::bigint[]);
-- $1 = '{1,2,3,4,5}' (pg sends a JS array as a single param). Empty array → 0 rows,
-- no syntax error. One cached plan regardless of list length.
```

### Wrong 2: `<> ALL` over an array that contains NULL

```sql
-- WRONG: "give me orders whose status is none of these"
SELECT * FROM orders WHERE status <> ALL($1);   -- $1 = '{shipped,delivered,NULL}'
-- BUG: the NULL element makes  status <> ALL(...)  evaluate to NULL for EVERY row
-- whose status isn't matched (UNKNOWN <> gives UNKNOWN) → those rows are DROPPED.
-- Result: mysteriously fewer rows, exactly the NOT IN (NULL) trap from Topic 27.

-- RIGHT: strip NULLs before the NOT-IN test
SELECT * FROM orders WHERE status <> ALL(array_remove($1, NULL));
-- Now non-matching rows survive; the NULL no longer contaminates the ALL.
```

### Wrong 3: Empty right-hand side of `@>` matches everything

```sql
-- WRONG: filter built dynamically; when the user selects no tags, $1 = '{}'
SELECT * FROM products WHERE tags @> $1::text[];   -- $1 = '{}'
-- BUG: X @> '{}' is ALWAYS TRUE → returns the ENTIRE table, not "no filter, no rows"
-- or "everything" depending on intent. Silent, and usually the opposite of intended.

-- RIGHT: guard the empty case in the query or the app
SELECT * FROM products
WHERE cardinality($1::text[]) = 0 OR tags @> $1::text[];   -- explicit intent
-- ...or short-circuit in Node.js and skip the WHERE clause entirely when the list is empty.
```

### Wrong 4: `COUNT`/`array_length` confusion on empty arrays

```sql
-- WRONG: paginate/branch on tag count using array_length
SELECT id FROM products WHERE array_length(tags, 1) > 0;
-- BUG: array_length('{}', 1) is NULL, and NULL > 0 is NULL (not false) — the row is
-- excluded, which happens to be "right" here, but the same NULL breaks arithmetic:
--   array_length(tags,1) + 1  → NULL for empty arrays, corrupting computed columns.

-- RIGHT: use cardinality(), which returns 0 for empty arrays
SELECT id FROM products WHERE cardinality(tags) > 0;
SELECT id, cardinality(tags) + 1 AS slots_needed FROM products;  -- never NULL for '{}'
```

### Wrong 5: Aggregating array elements without exploding first

```sql
-- WRONG: trying to count tag occurrences by treating the array as comparable
SELECT tags, COUNT(*) FROM products GROUP BY tags;
-- BUG: this groups by the WHOLE array value — {a,b} and {b,a} are different groups,
-- {a} and {a,b} are different groups. You get per-distinct-array counts, NOT per-tag.

-- RIGHT: explode with unnest, then group by the element
SELECT t.tag, COUNT(*) AS occurrences
FROM products p
CROSS JOIN LATERAL unnest(p.tags) AS t(tag)
GROUP BY t.tag
ORDER BY occurrences DESC;
```

---

## 10. Performance Profile

### Storage and Memory

- An array lives inline in the heap tuple until it exceeds ~2KB, then TOASTs (compressed, out-of-line). Small arrays (a few dozen scalars) are cheap; large arrays pay de-TOAST + decompress cost on every read of that column.
- **`SELECT *` on a wide array column is expensive** even when you don't use the array — the tuple still carries or references it. Project only the columns you need.
- `work_mem` governs `array_agg` materialisation and any `Sort` used to order elements; a large `array_agg` over millions of rows can spill.

### Scaling `@>` / `&&` Containment

| Rows | No index (Seq Scan + de-TOAST) | GIN index (Bitmap Index Scan) |
|------|-------------------------------|-------------------------------|
| 10K | ~8 ms | < 1 ms |
| 1M | ~800 ms | ~1–3 ms |
| 10M | ~8 s | ~3–10 ms (posting-list size dependent) |
| 100M | ~80 s (table-scan bound) | ~10–40 ms (+ heap recheck cost) |

GIN turns a linear table scan into a logarithmic posting-list lookup + bitmap heap fetch. The heap recheck (lossy GIN) means very low-selectivity predicates (matching most rows) can still be seq-scan-preferred by the planner.

### Scaling `= ANY($1)` (IN-list) on a scalar column

Driven by the B-tree on the scalar column, cost is roughly `O(K · log N)` for K array elements against N rows — one index probe per element, OR-ed into a bitmap. A 1,000-element `= ANY` against a 100M-row table is ~1,000 index descents: fast (single-digit ms) if the column is indexed, catastrophic (full seq scan with a giant filter) if not. Very large K (tens of thousands) is better served by joining against `unnest($1)` as a VALUES-like set so the planner can hash-join.

### The Write Cost — MVCC and GIN Maintenance

- **Every array update rewrites the entire value.** Appending to an N-element array is O(N) in bytes written plus a dead tuple. A column appended-to on every request grows unbounded write amplification and table bloat; autovacuum struggles to keep up. At scale this is the dominant reason to normalise.
- **GIN writes are heavier than B-tree writes.** Each array element is a separate index key; inserting a 20-element array touches up to 20 posting lists. The `fastupdate` pending list batches this, but a bulk load into a GIN-indexed array column is markedly slower — consider dropping and rebuilding the GIN index around bulk loads.

### Optimisation Techniques Specific to Arrays

1. **GIN for membership/containment; B-tree on the element via a junction table for heavy aggregation.** If most queries `unnest` and `GROUP BY` the element, a junction table with a B-tree beats GIN.
2. **`gin__int_ops`** (intarray extension) for large NULL-free `int[]` — smaller, faster index.
3. **Keep arrays small and bounded.** Model growth as rows, not elements.
4. **`array_remove(arr, NULL)`** before `<> ALL` to avoid the NULL-contamination correctness trap.
5. **`cardinality()` over `array_length(arr,1)`** everywhere a numeric length is needed.

---

## 11. Node.js Integration

### 11.1 The IN-list pattern with `pg` — pass a JS array as one parameter

```javascript
import pg from 'pg';
const { Pool } = pg;
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

// node-postgres maps a JS array to a PostgreSQL array parameter automatically.
async function getOrdersByIds(ids) {
  if (ids.length === 0) return [];            // short-circuit: avoid an all-rows/empty surprise
  const { rows } = await pool.query(
    `SELECT id, customer_id, total_amount, status
       FROM orders
      WHERE id = ANY($1::bigint[])            -- ONE param, stable query text, any length
      ORDER BY id`,
    [ids]                                     // $1 = [101, 205, 377] → '{101,205,377}'
  );
  return rows;
}
```

### 11.2 Containment / overlap with an array parameter

```javascript
// Products carrying ANY of the active promo tags (overlap → &&)
async function productsWithAnyTag(tagList) {
  const { rows } = await pool.query(
    `SELECT id, name, tags
       FROM products
      WHERE tags && $1::text[]                -- GIN-accelerated overlap
      ORDER BY id
      LIMIT 200`,
    [tagList]                                 // $1 = ['flash_sale','bogo']
  );
  return rows;
}

// Products carrying ALL of the required tags (containment → @>)
async function productsWithAllTags(tagList) {
  const { rows } = await pool.query(
    `SELECT id, name FROM products WHERE tags @> $1::text[] ORDER BY id`,
    [tagList]
  );
  return rows;
}
```

### 11.3 Reading arrays back — pg parses them into JS arrays

```javascript
// A text[] / int[] column comes back as a native JS array — no manual parsing.
const { rows } = await pool.query(`SELECT id, tags FROM products WHERE id = $1`, [42]);
console.log(rows[0].tags);        // → ['premium', 'wireless']  (real JS array)
console.log(Array.isArray(rows[0].tags));  // true

// NULL array column → null in JS; empty array '{}' → []  (distinct — check both!)
// numeric[] comes back as strings by default (pg preserves precision):
//   price_tiers numeric[]  →  ['19.99', '29.99']  — parse with parseFloat if needed.
```

### 11.4 Exploding an array parameter into a joinable set

```javascript
// For very large id lists, join against unnest so the planner can hash-join
// instead of doing K index probes.
async function bulkFetch(ids) {
  const { rows } = await pool.query(
    `SELECT o.id, o.total_amount
       FROM unnest($1::bigint[]) AS wanted(id)      -- turn the param into a set
       JOIN orders o ON o.id = wanted.id`,          -- hash/merge join on id
    [ids]
  );
  return rows;
}
```

**Do ORMs handle this?** The raw driver (`pg`) handles array params and array results natively and correctly — this is the reliable escape hatch. Most ORMs support array *columns* to some degree (see Section 12), but `@>`, `&&`, `= ANY` operators and `unnest`/`array_agg` usually require the ORM's raw-SQL facility or a `sql` template tag. When in doubt, drop to `pool.query` with `$1::type[]`.

---

## 12. ORM Comparison

### Prisma

**Can Prisma do array columns?** — Yes for scalar-list columns (`String[]`, `Int[]`) on PostgreSQL, with a dedicated set of list filters.

```typescript
import { PrismaClient } from '@prisma/client';
const prisma = new PrismaClient();

// Schema:  tags String[]   (maps to text[])
// has ANY of  → hasSome  (this is &&, overlap)
const overlap = await prisma.product.findMany({
  where: { tags: { hasSome: ['flash_sale', 'bogo'] } },
});

// has ALL of  → hasEvery  (this is @>, containment)
const contains = await prisma.product.findMany({
  where: { tags: { hasEvery: ['premium', 'wireless'] } },
});

// single membership → has  ('featured' = ANY(tags))
const member = await prisma.product.findMany({ where: { tags: { has: 'featured' } } });

// IN-list on a scalar column → the normal `in` filter (compiles to = ANY under the hood)
const byIds = await prisma.order.findMany({ where: { id: { in: [101, 205, 377] } } });

// unnest / array_agg / cross-element aggregation → raw SQL
const topTags = await prisma.$queryRaw`
  SELECT t.tag, COUNT(*) AS n
  FROM products p CROSS JOIN LATERAL unnest(p.tags) AS t(tag)
  GROUP BY t.tag ORDER BY n DESC LIMIT 10`;
```

**Where it breaks**: Prisma covers `has`/`hasSome`/`hasEvery`/`isEmpty` for list columns and `in`/`notIn` for scalars — but there is no way to express `unnest`, `array_agg`, subscripting, or per-element aggregation without `$queryRaw`. GIN index creation must be done in migration SQL. **Verdict**: excellent for the four common list predicates; raw SQL for anything that explodes or collects arrays.

---

### Drizzle ORM

**Can Drizzle do arrays?** — Yes, with typed array columns and `arrayContains`/`arrayContained`/`arrayOverlaps` helpers.

```typescript
import { db } from './db';
import { products, orders } from './schema';
import { sql, inArray, arrayContains, arrayOverlaps, arrayContained } from 'drizzle-orm';

// tags: text('tags').array()
await db.select().from(products).where(arrayContains(products.tags, ['premium','wireless'])); // @>
await db.select().from(products).where(arrayOverlaps(products.tags, ['sale','bogo']));         // &&
await db.select().from(products).where(arrayContained(products.tags, ['a','b','c']));          // <@

// IN-list on a scalar column
await db.select().from(orders).where(inArray(orders.id, [101, 205, 377]));   // = ANY

// unnest / array_agg via the sql template tag
const topTags = await db.execute(sql`
  SELECT t.tag, COUNT(*) AS n
  FROM ${products} p CROSS JOIN LATERAL unnest(p.tags) AS t(tag)
  GROUP BY t.tag ORDER BY n DESC LIMIT 10`);
```

**Where it breaks**: the three containment helpers cover `@>`/`<@`/`&&` cleanly and `inArray` covers `= ANY`; `unnest`, `array_agg`, slicing, and `WITH ORDINALITY` need the `sql` tag. **Verdict**: best-typed array support of the mainstream ORMs — dedicated operators plus a clean SQL escape hatch.

---

### Sequelize

**Can Sequelize do arrays?** — Yes, via `DataTypes.ARRAY(...)` and the `Op.contains` / `Op.contained` / `Op.overlap` operators.

```javascript
const { DataTypes, Op } = require('sequelize');
// tags: { type: DataTypes.ARRAY(DataTypes.TEXT) }

await Product.findAll({ where: { tags: { [Op.contains]:  ['premium','wireless'] } } }); // @>
await Product.findAll({ where: { tags: { [Op.overlap]:   ['sale','bogo'] } } });        // &&
await Product.findAll({ where: { tags: { [Op.contained]: ['a','b','c'] } } });          // <@

// IN-list on a scalar column
await Order.findAll({ where: { id: { [Op.in]: [101, 205, 377] } } });                   // = ANY

// unnest / array_agg → raw
const [rows] = await sequelize.query(
  `SELECT t.tag, COUNT(*) n FROM products p
   CROSS JOIN LATERAL unnest(p.tags) t(tag) GROUP BY t.tag ORDER BY n DESC LIMIT 10`);
```

**Where it breaks**: `Op.contains`/`Op.overlap`/`Op.contained` map directly to `@>`/`&&`/`<@`, and `ARRAY` columns round-trip as JS arrays. But `unnest`, `array_agg`, and element subscripting need `sequelize.query`. Migrations must add the GIN index by raw SQL. **Verdict**: solid operator coverage; raw query for explode/collect.

---

### TypeORM

**Can TypeORM do arrays?** — Yes, via `{ type: 'text', array: true }` (or `'int', array: true`). Operator support is thinner — most array predicates are written as raw `where` strings.

```typescript
@Column({ type: 'text', array: true, default: () => "'{}'" })
tags: string[];

// Containment / overlap: raw fragment in QueryBuilder (no dedicated helper)
await repo.createQueryBuilder('p')
  .where('p.tags @> :req', { req: ['premium','wireless'] })   // @>
  .getMany();

await repo.createQueryBuilder('p')
  .where('p.tags && :any', { any: ['sale','bogo'] })          // &&
  .getMany();

// IN-list: In() helper (compiles to IN / = ANY)
import { In } from 'typeorm';
await orderRepo.find({ where: { id: In([101, 205, 377]) } });

// unnest / array_agg → dataSource.query(...)
```

**Where it breaks**: TypeORM stores/reads arrays fine and `In()` covers the IN-list, but it has no first-class `@>`/`&&`/`<@` operator — you hand-write the fragment and bind the array parameter. Watch parameter expansion: a bound array in a raw fragment must be the array itself, matched to `@>`/`&&`. **Verdict**: workable but the least ergonomic; expect raw fragments for containment and raw queries for explode/collect.

---

### Knex.js

**Can Knex do arrays?** — Yes, with array column types in migrations and `.whereRaw()` / `.whereIn()` for predicates.

```javascript
// migration: table.specificType('tags', 'text[]')
await knex('products').whereRaw('tags @> ?', [['premium','wireless']]);  // @>
await knex('products').whereRaw('tags && ?', [['sale','bogo']]);         // &&

// IN-list on a scalar column
await knex('orders').whereIn('id', [101, 205, 377]);                     // → in (...) 
// or explicitly as an array param:
await knex('orders').whereRaw('id = ANY(?)', [[101, 205, 377]]);         // = ANY($1)

// unnest / array_agg
const topTags = await knex.raw(`
  SELECT t.tag, COUNT(*) n FROM products p
  CROSS JOIN LATERAL unnest(p.tags) t(tag) GROUP BY t.tag ORDER BY n DESC LIMIT 10`);

// GIN index in a migration:
await knex.raw('CREATE INDEX idx_products_tags_gin ON products USING GIN (tags)');
```

**Where it breaks**: Knex has no native array operators — everything containment-related goes through `whereRaw` with a bound array. `whereIn` builds a real `IN (...)` list (not `= ANY`), which reintroduces the variable-length-query-text concern; prefer `whereRaw('id = ANY(?)', [ids])` for a stable plan. **Verdict**: transparent and predictable; you write the SQL, Knex binds the array param.

---

### ORM Summary Table

| ORM | `@>` contains | `&&` overlap | `= ANY` IN-list | `unnest`/`array_agg` | Verdict |
|-----|--------------|-------------|-----------------|----------------------|---------|
| Prisma | `hasEvery` | `hasSome` | `in` | Raw SQL | Great for list filters; raw for explode |
| Drizzle | `arrayContains` | `arrayOverlaps` | `inArray` | `sql` tag | Best typed support |
| Sequelize | `Op.contains` | `Op.overlap` | `Op.in` | Raw query | Solid operator coverage |
| TypeORM | raw fragment | raw fragment | `In()` | Raw query | Workable, least ergonomic |
| Knex | `whereRaw('@>')` | `whereRaw('&&')` | `whereRaw('= ANY')` | `knex.raw` | Transparent; you write SQL |

---

## 13. Practice Exercises

### Exercise 1 — Easy

Given `products(id, name, tags text[])`:

1. Find every product tagged with **both** `'organic'` and `'gluten_free'`.
2. Find every product tagged with **at least one** of `'sale'`, `'clearance'`.
3. Return each product's id and its **number of tags**, treating an empty array as 0 (not NULL).

```sql
-- Write your query here
```

---

### Exercise 2 — Medium (combines Topic 11 JOIN + Topic 20 GROUP BY)

Given `products(id, name, category_id, tags text[])` and `categories(id, name)`:

Produce, per category name, the **top 3 tags** by product count, as a single ordered array per category (element 1 = most common tag). Include the per-tag counts as a parallel array. Only consider categories that have at least 5 products. Hint: `unnest` to explode, aggregate per (category, tag), rank with a window function (Topic 40s), then `array_agg` the top 3.

```sql
-- Write your query here
```

---

### Exercise 3 — Hard (production simulation; the naive answer is wrong)

An endpoint receives a list of `status` values to **exclude** from a report, passed as one `text[]` parameter `$1`. A developer wrote:

```sql
SELECT * FROM orders WHERE status <> ALL($1);
```

QA reports that when `$1` happens to include a NULL (a bug upstream occasionally injects one), the endpoint returns **far fewer rows than expected**, sometimes zero.

1. Explain precisely why a single NULL element collapses the result.
2. Rewrite the query so it is correct regardless of NULLs in `$1`.
3. Also handle the case where `$1` is empty (should return **all** orders) and where `$1` is SQL NULL (decide and document the intended behaviour).

```sql
-- Write your query here
```

---

### Exercise 4 — Interview Simulation

You have `user_roles(user_id bigint, roles text[])`. The product team wants: "list every distinct role in use, with how many users hold it, and for each role the array of the 5 most-recently-active user ids holding it." There is a `users(id, last_active_at)` table. Design the query. Then answer: at 50M users, is `roles text[]` the right model, or should roles be a junction table? Justify with the write pattern and the query pattern.

```sql
-- Write your query here
```

---

## 14. Interview Questions

### Q1 — "How do you pass a JavaScript array of ids into a parameterised query, and why not build an IN-list?"

**Junior answer**: "I'd build the string `IN ($1, $2, $3)` with a placeholder per element and spread the array into the params."

**Principal answer**: "Pass the whole list as a single array parameter and use `WHERE id = ANY($1::bigint[])`. The `= ANY(array)` form is planner-equivalent to `IN (list)` — with a B-tree on `id` it produces the same bitmap-of-index-probes plan — but it has three decisive advantages. First, the query text is *constant* regardless of list length, so the prepared-statement cache keeps one plan instead of thrashing a new plan per N. Second, an empty array yields zero rows cleanly, whereas a hand-built `IN ()` is a syntax error you must special-case. Third, there's exactly one bound parameter, so there's no string interpolation surface for injection. For very large lists — tens of thousands — I'd instead `JOIN unnest($1)` so the planner can hash-join rather than do K individual index descents."

**Follow-up**: *"When would `= ANY($1)` be slow?"* — When `id` has no index (it degrades to a seq scan with a big filter), or when K is so large that per-element index probes dominate and a hash join over `unnest` would be cheaper.

---

### Q2 — "What's the difference between `@>`, `= ANY`, and where each needs an index?"

**Junior answer**: "They both check if something is in the array."

**Principal answer**: "They operate on opposite sides. `col = ANY($1)` scans a **scalar** column and asks 'is this row's scalar one of the parameter array's elements?' — that's the IN-list, accelerated by a **B-tree on the scalar column**. `arr @> $1` scans an **array** column and asks 'does this row's array contain all of the parameter's elements?' — that's containment, accelerated by a **GIN index on the array column**. `&&` is the overlap (any-of) sibling of `@>`, also GIN-served. The classic misdiagnosis is expecting GIN to help `id = ANY(...)` on a scalar id — GIN indexes the array's *elements*, so it only helps when the array is the *column*, not when it's the parameter."

**Follow-up**: *"Why does the GIN plan show a Recheck Cond?"* — GIN is lossy at the page/bitmap level, so PostgreSQL re-evaluates the operator on each heap tuple to drop false positives; the recheck is expected and cheap relative to the scan it avoids.

---

### Q3 — "You have an `int[]` column that grows by one element on every user action. What's wrong and what do you do?"

**Junior answer**: "Nothing — appending to an array with `arr || x` is simple and keeps everything in one row."

**Principal answer**: "It's a scaling time bomb because of MVCC. PostgreSQL has no in-place array mutation — every append rewrites the *entire* array value into a new tuple version and marks the old one dead. So the append is O(N) in bytes written and produces a dead tuple each time; once the array exceeds ~2KB it TOASTs, adding compression and out-of-line churn on every write. Under load this is massive write amplification plus table bloat that autovacuum can't keep up with, and concurrent appenders serialise on the row. The right model is a child table: one row per event with a foreign key to the parent and a B-tree on that key. Inserts become O(1) append-only, there's per-row locking for concurrent writers, and cross-element queries (counts, top-N, time filters) use a normal index instead of perpetual `unnest`. Arrays are for small, bounded, read-mostly, attribute-free sets — an unboundedly growing log fails every one of those tests."

**Follow-up**: *"When is the array actually fine?"* — Small bounded sets read and written as a whole: a handful of tags, a fixed dimensions triple, a short permissions list — where you rarely `unnest` and never need per-element constraints or foreign keys.

---

## 15. Mental Model Checkpoint

1. **`SELECT 9 <> ALL(ARRAY[1, NULL, 3])` — what does it return, and why is that dangerous in a `WHERE` clause meant to express NOT IN?**

2. **A dynamically built filter does `WHERE tags @> $1` and the user selects no tags, so `$1 = '{}'`. What rows come back, and why is that almost never the intended behaviour?**

3. **You `CROSS JOIN LATERAL unnest(p.tags)` and notice products with no tags have vanished from the result. Explain the mechanism and the one-word fix to keep them.**

4. **`array_length(arr, 1)` and `cardinality(arr)` disagree for one specific input. Which input, what does each return, and which should you default to?**

5. **Why does a GIN index accelerate `tags @> ARRAY['x']` but not `tags[1] = 'x'`? What is GIN actually storing?**

6. **You run the same `array_agg(tag)` query twice and `result.tags[1]` differs between runs even though the data didn't change. What's missing and why is element order otherwise undefined?**

7. **Given a 50M-row table, when would you deliberately choose `JOIN unnest($1::bigint[])` over `WHERE id = ANY($1::bigint[])` for the same id list?**

---

## 16. Quick Reference Card

```sql
-- LITERALS & CONSTRUCTORS
ARRAY[1,2,3]                 -- constructor: elements are EVALUATED expressions
'{1,2,3}'::int[]             -- literal: raw text, needs cast; quote commas/spaces/NULL
ARRAY[]::text[]              -- empty array (cast REQUIRED)

-- SET OPERATORS (order & duplicates ignored; GIN-indexable)
tags @>  ARRAY['a','b']      -- contains ALL         (X @> '{}' is ALWAYS true!)
tags <@  ARRAY['a','b','c']  -- is contained by (subset)
tags &&  ARRAY['a','b']      -- overlaps (shares ANY)
arr1 || arr2                 -- concatenate; arr || elem appends; elem || arr prepends

-- IN-LIST PATTERN (scalar column vs parameter array — needs B-tree on the column)
WHERE id = ANY($1::bigint[])   -- = IN (list); stable query text; empty → 0 rows
WHERE x <> ALL($1)             -- = NOT IN; STRIP NULLS FIRST: <> ALL(array_remove($1,NULL))

-- ANY/ALL empty & NULL semantics
3 = ANY('{}')   -> false      3 <> ALL('{}')  -> TRUE (vacuous)
3 = ANY(NULL)   -> NULL        9 <> ALL('{1,NULL}') -> NULL (NULL contaminates NOT IN)

-- EXPLODE / COLLECT
SELECT unnest(arr) WITH ORDINALITY;                 -- rows + 1-based position
FROM t CROSS JOIN LATERAL unnest(t.arr) x(v)        -- empty arr DROPS the row...
FROM t LEFT  JOIN LATERAL unnest(t.arr) x(v) ON TRUE -- ...LEFT keeps it (v = NULL)
array_agg(v ORDER BY v)                             -- ALWAYS order for reproducibility
array_agg(v) FILTER (WHERE v IS NOT NULL)           -- exclude NULL elements
array_agg(DISTINCT v ORDER BY v)                    -- dedupe + sort

-- LENGTH (mind the empty-array trap)
cardinality(arr)        -- 0 for '{}'   <- prefer this
array_length(arr, 1)    -- NULL for '{}' <- breaks arithmetic

-- SUBSCRIPT / SLICE (1-indexed; out-of-bounds -> NULL, never error)
arr[1]      arr[2:3]     arr[:2]     arr[2:]

-- INDEXING
CREATE INDEX ... USING GIN (tags);          -- @>, <@, &&, element = ANY(col)
CREATE INDEX ... USING GIN (int_col gin__int_ops);  -- compact int[] (intarray ext)

-- PERF RULES OF THUMB
-- @>/&& without GIN on 1M+ rows = seq scan + de-TOAST -> seconds. Add GIN.
-- Every array UPDATE rewrites the WHOLE value (MVCC) -> don't grow arrays unboundedly.
-- If you unnest in MOST queries, it should have been a junction table.

-- INTERVIEW ONE-LINERS
-- "= ANY(array) is IN(list) with one stable param — no cache thrash, no empty-list error."
-- "@> needs GIN on the array column; = ANY needs B-tree on the scalar column — opposite sides."
-- "Arrays are one value in one tuple: TOAST, whole-value MVCC rewrite, GIN inverted index."
-- "Strip NULLs before <> ALL — one NULL element collapses NOT-IN to zero rows."
```

---

## Connected Topics

- **Topic 46 — MVCC and Tuple Versioning**: Why every array update rewrites the entire value and generates dead tuples — the core reason growing arrays bloat.
- **Topic 64 — JSON and JSONB**: The sibling denormalisation. Use `jsonb` for heterogeneous/nested/ragged structures; arrays for homogeneous flat sets of one scalar type. `jsonb` also GIN-indexes with `@>`.
- **Topic 66 — Full-Text Search**: The other flagship GIN use case (`tsvector`); the same inverted-index machinery that powers `@>` powers `@@`.
- **Topic 11 — INNER JOIN in Depth**: `CROSS JOIN LATERAL unnest` drops empty-array rows exactly as INNER JOIN drops unmatched rows; the junction-table alternative to arrays lives here.
- **Topic 20 — GROUP BY Fundamentals**: `array_agg` is a grouping aggregate; `unnest` + `GROUP BY` is the canonical per-element counting pattern.
- **Topic 27 — EXISTS and NOT IN**: The `<> ALL(array-with-NULL)` trap is the array form of the `NOT IN (subquery-with-NULL)` trap.
- **Topic 07 — NULL in Depth**: Three-valued logic explains every `= ANY`/`<> ALL`/`@>` NULL result in this topic.
- **Topic 40s — Window Functions**: Ranking elements before `array_agg`-ing the top-N per group (Exercise 2).
- **GIN Indexes (Indexing phase)**: The inverted-index structure shared by array containment, `jsonb`, and full-text search.
