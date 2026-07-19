# Topic 64 — JSON and JSONB in PostgreSQL
### SQL Mastery Curriculum — Phase 9: Advanced SQL Patterns

---

## 1. ELI5 — The Analogy That Makes It Click

Imagine you receive a hand-written letter in the mail. Someone wrote you a shopping list on a single sheet of paper:

```
milk, eggs (dozen), bread — whole wheat, notes: "get organic if under $6"
```

That letter is **JSON**. It is the *exact text* your friend wrote — every space, every comma, the order they chose, even the duplicate word if they accidentally wrote "milk" twice. If you want to answer "how many eggs?", you have to re-read the whole letter from the top, parse the words, and figure it out. Every single time. The paper is stored word-for-word, faithfully, but it is dumb paper.

Now imagine instead you handed that letter to a diligent assistant. The assistant reads it once, throws away the original paper, and files everything into a neat set of labeled index cards: one card says `milk`, one says `eggs → 12`, one says `bread → whole wheat`. If your friend wrote "milk" twice, the assistant keeps one card. The cards are sorted so the assistant can instantly flip to "eggs" without re-reading everything. That filing cabinet is **JSONB**.

The difference is the entire topic:

- **JSON** stores the raw text exactly as received. Fast to write (just save the paper), slow to query (re-parse every time), preserves whitespace, key order, and duplicate keys.
- **JSONB** parses the text once at write time into a decomposed binary structure. Slightly slower to write (the assistant has to do filing work), dramatically faster to query, loses whitespace and key order, de-duplicates keys, and — crucially — can be **indexed**.

Almost every time you reach for a document column in Postgres, you want the assistant with the filing cabinet. You want `jsonb`. The plain `json` type exists mostly for the rare case where you must round-trip the exact bytes you were given.

---

## 2. Connection to SQL Internals

Understanding what happens beneath `json` vs `jsonb` explains every behavioural and performance difference in this topic.

**`json` — stored as `text` with a validation check.** When you insert into a `json` column, Postgres validates that the string is syntactically valid JSON, then stores the *bytes verbatim* in the heap tuple (subject to TOAST compression for large values). There is no parse tree on disk. Every query operator (`->`, `->>`, etc.) must **re-parse the entire document from the start** to find the requested element. This is O(document size) per access, every time.

**`jsonb` — stored as a decomposed binary format.** On insert, Postgres parses the document into an internal tree of `JsonbValue` nodes and serializes it in a binary layout (the on-disk format has a version header byte, then a tree of `JEntry` headers plus values). Object keys are **sorted by (length, bytewise)** and **de-duplicated** (last value wins). Scalar values are stored in native form. Because keys are sorted, key lookup inside an object uses a **binary search**, not a linear scan — O(log k) for k keys. Arrays preserve order; objects do not.

**TOAST.** Both types participate in TOAST (The Oversized-Attribute Storage Technique). A `jsonb` value larger than ~2KB is compressed and/or moved to an out-of-line TOAST table. This matters: if you `->>'small_field'` out of a large TOASTed `jsonb`, Postgres must still **de-TOAST (decompress) the whole value** to read one field — there is no partial column read. This is the single biggest reason not to stuff megabyte documents into `jsonb` and then query one key at high frequency.

**GIN indexes (Topic on indexing).** The reason `jsonb` can be indexed and `json` cannot: `jsonb`'s decomposed structure lets a **GIN (Generalized Inverted Index)** enumerate the keys and values as index-able tokens. A GIN index maps each token (e.g. the key `"status"` or the pair `status=shipped`) to a posting list of heap TIDs. Containment queries (`@>`) then become a posting-list intersection instead of a table scan. `json` has no such structure to enumerate, so it cannot be GIN-indexed (except via a functional expression index on an extracted scalar).

**MVCC and updates.** `jsonb` is not updated in place. `jsonb_set()` produces a **new** document, and updating the column writes a **new heap tuple** (MVCC — Topic on MVCC), leaving the old row version as dead until VACUUM. There is no "patch a field cheaply" — every field mutation rewrites the whole document and its TOAST chunks. Fat documents with hot fields cause heavy write amplification and bloat.

**Planner and statistics.** The planner has weak statistics inside a `jsonb` column. It does not build per-key histograms, so selectivity estimates for `@>` and `->>` predicates are crude (often a hardcoded fraction). This frequently produces bad row estimates and poor join/scan choices — a recurring theme in the EXPLAIN section.

---

## 3. Logical Execution Order Context

JSON operators are **scalar expressions**, not clauses. They do not have their own phase in the logical execution order — they are evaluated wherever the expression that contains them is evaluated:

```
FROM        ← the jsonb column is read from the heap tuple here (and de-TOASTed)
WHERE       ← jsonb predicates (data @> '...', data->>'k' = '...') filter rows here
GROUP BY    ← can GROUP BY (data->>'category')
HAVING
SELECT      ← extraction/projection (data->>'name', jsonb_build_object(...)) happens here
DISTINCT
ORDER BY    ← can ORDER BY (data->>'priority')::int
LIMIT
```

Two consequences matter in practice:

1. **A `@>` predicate in `WHERE` is where a GIN index can be used.** If your containment filter lives in `WHERE`, the planner may satisfy it with a GIN index scan before de-TOASTing whole rows. If the same logic is buried inside a `SELECT` CASE expression, no index helps — it runs per surviving row.

2. **Set-returning functions change cardinality in `FROM`.** `jsonb_array_elements()` and `jsonb_each()` are set-returning functions. When used in `FROM` (as a `LATERAL` or implicit lateral join), they run in the FROM/JOIN phase and **multiply rows** — one input row with a 5-element array becomes 5 rows, exactly like the one-to-many fan-out from INNER JOIN (Topic 11). When the same function is used in the `SELECT` list, it also expands rows but the mental model is muddier — prefer the explicit `CROSS JOIN LATERAL` form so the fan-out is visible where it happens.

Because the `jsonb` column is fully read in the FROM phase, filtering on a JSON field never avoids the de-TOAST cost of the rows it inspects — only an index that answers the predicate *without* touching the heap value can do that.

---

## 4. What Is JSON / JSONB?

`json` and `jsonb` are Postgres column types that store an entire JSON document — objects, arrays, strings, numbers, booleans, null — in a single column. `json` preserves the exact input text; `jsonb` stores a parsed, decomposed binary form that is de-duplicated, key-sorted, and indexable. In almost all cases you want `jsonb`.

```sql
CREATE TABLE events (
  id          bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  payload     jsonb NOT NULL,          -- decomposed binary; indexable, fast to query
  raw_body    json,                    -- verbatim text; use only if exact bytes matter
  received_at timestamptz DEFAULT now()
);
```

### Annotated operator syntax

```sql
SELECT
  payload -> 'user'                 -- ->  : get object field / array elem AS jsonb
  --      │      │                          returns jsonb (can be chained: -> -> ->)
  --      │      └── key name (text) or, with int, array index
  --      └── field access, keeps json type

  payload ->> 'user'                -- ->> : get object field / array elem AS text
  --      │                                 returns text (NOT jsonb) — end of a chain
  --      └── same access, but unwraps to text; a JSON string loses its quotes

  payload #> '{user,address,city}'  -- #>  : get element at a PATH AS jsonb
  --      │      │                          path is a text[] of keys/indexes
  --      │      └── '{a,b,2}' means payload['a']['b'][2]
  --      └── deep access without chaining ->

  payload #>> '{user,address,city}' -- #>> : get element at a PATH AS text
  --                                         same as #> but final result is text

  payload @> '{"status":"paid"}'    -- @>  : does LEFT contain RIGHT?  (boolean)
  --      │                                 the workhorse for GIN-indexed filtering
  --      └── containment: every key/value on the right exists on the left

  payload <@ '{"a":1,"b":2}'        -- <@  : is LEFT contained BY RIGHT? (boolean)
  --                                         reverse of @>

  payload ? 'status'                -- ?   : does LEFT have this top-level KEY? (bool)
  --      │                                 for objects: key exists; for arrays: string elem exists
  --      └── single text key

  payload ?| array['a','b']         -- ?|  : does LEFT have ANY of these keys? (bool)
  payload ?& array['a','b']         -- ?&  : does LEFT have ALL of these keys? (bool)

  payload @? '$.items[*].price'     -- @?  : does JSONPath match anything? (bool)
  payload @@ '$.total > 100'        -- @@  : does JSONPath predicate return true? (bool)
;
```

Key rules to memorize:

- **`->` returns `jsonb`; `->>` returns `text`.** Chain with `->` and finish with `->>` (or `#>>`).
- **A JSON string value keeps its quotes under `->` and loses them under `->>`.** `('{"a":"x"}'::jsonb -> 'a')` is `"x"` (jsonb string); `->> 'a'` is `x` (text).
- **Negative array indexes work**: `arr -> -1` is the last element (jsonb only).
- **Missing keys return SQL `NULL`, not an error**: `payload -> 'nope'` is `NULL`. This differs from JSON `null`, which is `'null'::jsonb`.
- **`@>`, `?`, `?|`, `?&` are the operators a `jsonb_ops` GIN index accelerates.** `->>` equality is *not* directly GIN-accelerated (it needs an expression index instead).

---

## 5. Why JSON/JSONB Mastery Matters in Production

1. **The wrong type choice costs you at every read.** Choosing `json` when you meant `jsonb` means every `->>`, every filter, every extraction re-parses the whole document. On a hot endpoint reading a field out of a 10KB document millions of times a day, that is measurable CPU you will never get back without a migration. The reverse — choosing `jsonb` when you needed byte-exact round-tripping (signature verification of a webhook body, for example) — silently corrupts your verification because key order and whitespace changed.

2. **Missing GIN indexes turn every filter into a Seq Scan.** A `WHERE payload @> '{"status":"failed"}'` on a 50M-row table with no GIN index is a full sequential scan that de-TOASTs every row. Teams routinely ship this, watch a dashboard query take 40 seconds, and never realize one `CREATE INDEX ... USING gin` makes it 20ms.

3. **Write amplification from fat documents.** Because `jsonb_set` rewrites the whole document and re-TOASTs it, a table where you update one counter inside a 200KB document on every request produces enormous WAL (Topic on WAL) and table bloat. The fix is almost always schema, not tuning.

4. **The schema-smell trap.** JSONB is a genuine feature, but it is also where under-designed schemas go to hide. Storing `first_name`, `email`, and `status` inside a `jsonb` blob — fields that are always present, always the same shape, and frequently filtered — throws away type checking, foreign keys, NOT NULL, cheap B-tree indexes, and honest statistics. Recognizing when JSONB is the right tool vs when it is an excuse to skip data modeling is a senior judgment call interviewers probe directly.

5. **Query correctness with fan-out.** `jsonb_array_elements` multiplies rows. Aggregating after it without accounting for the fan-out produces the same class of double-counting bug as a one-to-many JOIN (Topic 11) — but harder to spot because there is no visible second table.

---

## 6. Deep Technical Content

### 6.1 JSON vs JSONB — the exhaustive comparison

| Property | `json` | `jsonb` |
|---|---|---|
| On-disk form | verbatim text | decomposed binary |
| Insert cost | lower (validate only) | higher (parse + serialize) |
| Query cost | re-parse every access | binary search, no re-parse |
| Whitespace preserved | yes | no (normalized) |
| Key order preserved | yes | no (sorted by length then bytes) |
| Duplicate keys | kept | de-duplicated (last wins) |
| Indexable (GIN) | no | yes |
| Supports `@>`, `?`, JSONPath containment ops | no | yes |
| Equality comparison (`=`) | not defined | defined (structural) |
| Number precision | preserved as written (`1.0` stays `1.0`) | numeric-normalized (`1.0` → `1`... see 6.9) |

The one-line rule: **use `jsonb` unless you must reproduce the exact original bytes.**

Demonstration of normalization — this is `jsonb` de-duplicating and re-ordering keys:

```sql
SELECT '{"b":1, "a":2, "a":3}'::json  AS as_json,
       '{"b":1, "a":2, "a":3}'::jsonb AS as_jsonb;
```
```
   as_json          |    as_jsonb
--------------------+------------------
 {"b":1, "a":2, "a":3} | {"a": 3, "b": 1}
```
`json` kept everything verbatim including the duplicate `a`. `jsonb` dropped the whitespace, sorted `a` before `b`, and kept only the last `a` (3).

### 6.2 The extraction operators in full

```sql
WITH doc AS (SELECT '{
  "id": 7,
  "name": "Ada",
  "tags": ["admin","beta"],
  "address": {"city":"Paris","zip":"75001"}
}'::jsonb AS j)
SELECT
  j -> 'name'              AS arrow_jsonb,   -- "Ada"     (jsonb string, quoted)
  j ->> 'name'             AS arrow_text,    -- Ada       (text, unquoted)
  j -> 'tags' -> 0         AS elem_jsonb,    -- "admin"   (jsonb)
  j -> 'tags' ->> 0        AS elem_text,     -- admin     (text)
  j -> 'tags' -> -1        AS last_elem,     -- "beta"    (negative index)
  j #> '{address,city}'    AS path_jsonb,    -- "Paris"   (jsonb)
  j #>> '{address,city}'   AS path_text,     -- Paris     (text)
  j -> 'missing'           AS missing,       -- NULL      (no error)
  j ->> 'id'               AS id_text,       -- 7         (number → text)
  (j ->> 'id')::int        AS id_int         -- 7         (cast to int)
FROM doc;
```

Chaining rule of thumb: use `->` for every step except the last; use `->>` (or `#>>`) only at the very end when you want a scalar text value. Doing `->>` in the middle breaks the chain because `text` has no further JSON operators.

### 6.3 The containment operators: `@>` and `<@`

`@>` ("contains") asks: *is every key/value pair on the right present on the left?* It is the single most important operator for indexed filtering.

```sql
-- object contains object: all right-side pairs must exist on the left
'{"a":1,"b":2,"c":3}'::jsonb @> '{"a":1}'::jsonb          -- true
'{"a":1,"b":2}'::jsonb       @> '{"a":1,"b":2}'::jsonb    -- true
'{"a":1}'::jsonb             @> '{"a":1,"b":2}'::jsonb    -- false (b missing)
'{"a":1}'::jsonb             @> '{"a":2}'::jsonb          -- false (value differs)

-- array contains array: every right element must appear on the left (order-free)
'[1,2,3]'::jsonb   @> '[3,1]'::jsonb                      -- true
'[1,2,3]'::jsonb   @> '[1,4]'::jsonb                      -- false

-- object membership inside an array of objects
'[{"x":1},{"y":2}]'::jsonb @> '[{"x":1}]'::jsonb          -- true
```

Critical gotchas with `@>`:

- **Containment is nested and structural**, not a substring match. `@> '{"a":1}'` matches only when key `a` has value `1` at that level.
- **Scalars: `@>` requires exact structural match** for top-level scalars. `'"foo"'::jsonb @> '"foo"'::jsonb` is true; `'"foobar"'::jsonb @> '"foo"'::jsonb` is false (it is *not* `LIKE`).
- **An empty object is contained by everything**: `anything @> '{}'::jsonb` is true. This is a classic accidental-match-everything bug when the right side is built dynamically and comes out empty.
- **`@>` does not descend into arrays by key path.** To match "an array element where price > 100" you need JSONPath (6.7), not `@>`.

`<@` is just `@>` with the arguments flipped: `a <@ b` ≡ `b @> a`.

### 6.4 The key-existence operators: `?`, `?|`, `?&`

```sql
'{"a":1,"b":2}'::jsonb ? 'a'                  -- true  (top-level key exists)
'{"a":1,"b":2}'::jsonb ? 'x'                  -- false
'["a","b","c"]'::jsonb ? 'b'                  -- true  (array: string element exists)
'{"a":1}'::jsonb ?| array['x','a']            -- true  (ANY key exists)
'{"a":1,"b":2}'::jsonb ?& array['a','b']      -- true  (ALL keys exist)
'{"a":1}'::jsonb ?& array['a','b']            -- false (b missing)
```

Gotchas:

- **`?` checks *keys* for objects and *string elements* for arrays.** It does **not** check values. `'{"a":1}'::jsonb ? '1'` is false — `1` is a value, not a key.
- **`?` only checks the top level.** It does not descend. `'{"a":{"b":1}}'::jsonb ? 'b'` is false.
- **In a `WHERE` clause, `?` collides with the driver's parameter placeholder in some client libraries** (JDBC, and historically some Node setups). In `pg` for Node the placeholder is `$1`, so `?` is safe there, but be aware when copying SQL between ecosystems — you may need the function form `jsonb_exists(payload, 'key')`.

### 6.5 Building JSON: `jsonb_build_object`, `jsonb_build_array`, `to_jsonb`, `jsonb_agg`

```sql
-- build an object from alternating key, value, key, value ...
SELECT jsonb_build_object('id', u.id, 'name', u.name, 'active', u.is_active)
FROM users u WHERE u.id = 1;
-- {"id": 1, "name": "Ada", "active": true}

-- build an array
SELECT jsonb_build_array(1, 'two', true, null);   -- [1, "two", true, null]

-- convert a whole row / value to jsonb
SELECT to_jsonb(u) FROM users u WHERE u.id = 1;   -- {"id":1,"name":"Ada",...}

-- aggregate many rows into one array (the JSON analog of array_agg)
SELECT jsonb_agg(jsonb_build_object('id', o.id, 'total', o.total_amount))
FROM orders o WHERE o.customer_id = 42;
-- [{"id":1,"total":10}, {"id":2,"total":25}, ...]

-- aggregate key/value pairs into one object
SELECT jsonb_object_agg(k, v) FROM (VALUES ('a',1),('b',2)) t(k,v);
-- {"a": 1, "b": 2}
```

Notes: `jsonb_build_object` takes an **even** number of arguments (pairs); an odd count is an error. `to_jsonb` respects column names, so aliasing columns in a subquery controls the output keys. `jsonb_agg` and `jsonb_object_agg` are aggregates — they collapse groups just like `SUM`/`COUNT` and pair naturally with `GROUP BY` to build nested documents in one query (see Example 3).

### 6.6 Deconstructing and mutating JSON

```sql
-- jsonb_each: expand an object into (key text, value jsonb) rows
SELECT key, value FROM jsonb_each('{"a":1,"b":2}'::jsonb);
--  key | value
--  ----+------
--  a   | 1
--  b   | 2

-- jsonb_each_text: same but value is text
SELECT key, value FROM jsonb_each_text('{"a":1,"b":2}'::jsonb);

-- jsonb_array_elements: expand an array into one row per element (jsonb)
SELECT value FROM jsonb_array_elements('[10,20,30]'::jsonb);   -- 3 rows: 10,20,30
-- jsonb_array_elements_text: text variant
-- jsonb_object_keys: just the keys of an object, one per row

-- jsonb_set: return a NEW document with a path replaced/added
SELECT jsonb_set('{"a":{"b":1}}'::jsonb, '{a,b}', '99');       -- {"a":{"b":99}}
SELECT jsonb_set('{"a":1}'::jsonb, '{c}', '5', true);          -- {"a":1,"c":5}  (create_missing=true)
SELECT jsonb_set('{"a":1}'::jsonb, '{c}', '5', false);         -- {"a":1}        (create_missing=false → no-op)

-- jsonb_insert: insert without overwriting existing (has a before/after flag)
SELECT jsonb_insert('[1,2,3]'::jsonb, '{1}', '99');            -- [1,99,2,3]

-- remove: - key, - array index, #- path
SELECT '{"a":1,"b":2}'::jsonb - 'a';                           -- {"b":2}
SELECT '[1,2,3]'::jsonb - 0;                                   -- [2,3]
SELECT '{"a":{"b":1,"c":2}}'::jsonb #- '{a,b}';                -- {"a":{"c":2}}

-- merge / shallow concat with ||
SELECT '{"a":1,"b":2}'::jsonb || '{"b":9,"c":3}'::jsonb;       -- {"a":1,"b":9,"c":3}
```

Critical mutation gotchas:

- **`jsonb_set` on a NULL path element returns NULL for the whole document.** `jsonb_set(doc, '{a,b}', v)` where `doc->'a'` does not exist and `create_missing` cannot build the intermediate object returns the document unchanged (or NULL if `doc` itself is NULL). Always guard: `jsonb_set(COALESCE(doc,'{}'::jsonb), ...)`.
- **`||` is a *shallow* merge.** `'{"a":{"x":1}}' || '{"a":{"y":2}}'` gives `{"a":{"y":2}}` — the nested object is *replaced*, not deep-merged. Postgres has no built-in deep merge; you write a recursive function or use `jsonb_set` per path.
- **`jsonb_set` and `||` return new documents.** Persisting a change is `UPDATE t SET doc = jsonb_set(doc, ...)` — a full-row rewrite (MVCC, 6.10).

### 6.7 JSONPath — `@?`, `@@`, `jsonb_path_query`

JSONPath (SQL/JSON path language, Postgres 12+) is a mini query language *inside* a `jsonb` value. It expresses filters that `@>` cannot — comparisons, wildcards, array-element predicates.

```sql
-- @? : does the path find at least one match? (boolean, good for WHERE + GIN)
SELECT '{"items":[{"price":50},{"price":150}]}'::jsonb @? '$.items[*] ? (@.price > 100)';
-- true  (there exists an item with price > 100)

-- @@ : does the path PREDICATE evaluate to true? (boolean)
SELECT '{"total": 250}'::jsonb @@ '$.total > 100';            -- true

-- jsonb_path_query: return every match as a set (rows)
SELECT jsonb_path_query('{"items":[{"price":50},{"price":150}]}'::jsonb,
                        '$.items[*] ? (@.price > 100)');
-- {"price": 150}

-- jsonb_path_query_first: just the first match
-- jsonb_path_query_array: collect matches into one jsonb array
SELECT jsonb_path_query_array('{"a":[1,2,3,4]}'::jsonb, '$.a[*] ? (@ > 2)');  -- [3, 4]

-- variables and the strict/lax modes
SELECT jsonb_path_query('{"a":[1,2,3]}'::jsonb, '$.a[*] ? (@ > $min)',
                        '{"min": 1}');                        -- 2, then 3
```

Key JSONPath facts:

- **`$` is the root, `@` is the current element**, `.key` navigates, `[*]` is any array element, `[0]` is index, `? (...)` is a filter predicate.
- **`lax` (default) vs `strict`**: in `lax` mode, path errors (missing keys, indexing a non-array) are swallowed and simply produce no match; in `strict` mode they raise. Prefix with `strict ` to force errors: `'strict $.a.b'`.
- **`@?` and `@@` differ subtly.** `@?` takes a path *expression* and asks "any match?". `@@` takes a path *predicate* (a boolean JSONPath) and asks "is it true?". `doc @? '$ ? (@.total > 100)'` and `doc @@ '$.total > 100'` express the same idea; `@?`/`@@` are the two that a `jsonb_path_ops` GIN index can accelerate.
- **JSONPath is the tool for value comparisons and array-element conditions** — precisely the queries `@>` cannot express.

### 6.8 GIN indexing — `jsonb_ops` vs `jsonb_path_ops`

Only `jsonb` can be GIN-indexed. There are two operator classes:

```sql
-- default: jsonb_ops — indexes every KEY and every VALUE as separate tokens
CREATE INDEX idx_payload_gin ON events USING gin (payload);
-- equivalently: USING gin (payload jsonb_ops)

-- jsonb_path_ops — indexes hashed key→value PATHS only
CREATE INDEX idx_payload_pathops ON events USING gin (payload jsonb_path_ops);
```

| | `jsonb_ops` (default) | `jsonb_path_ops` |
|---|---|---|
| Tokens indexed | every key and every value separately | a hash of each full path-to-value |
| Index size | larger | ~2–3× smaller |
| Supports `@>` | yes | yes |
| Supports `?`, `?|`, `?&` | **yes** | **no** |
| Supports `@?`, `@@` (JSONPath) | yes | yes |
| Typical `@>` speed | good | often faster (smaller, fewer false positives) |

Choose `jsonb_path_ops` when your workload is dominated by `@>` containment and JSONPath and you never need the key-existence operators (`?`). It is smaller and usually faster for `@>`. Choose the default `jsonb_ops` when you need `?`/`?|`/`?&`, or when you filter on keys existing regardless of value.

**Expression (functional) indexes for scalar equality.** GIN accelerates `@>`/`?`/JSONPath, but **not** `payload->>'status' = 'shipped'`. For a hot scalar-equality or range filter, index the extracted expression with a plain B-tree:

```sql
CREATE INDEX idx_status ON events ((payload ->> 'status'));
CREATE INDEX idx_priority ON events (((payload ->> 'priority')::int));
-- now: WHERE payload->>'status' = 'shipped'  can use idx_status (B-tree)
```

This is often the *better* choice than GIN when you filter on one well-known field: a B-tree on `(payload->>'status')` is smaller, supports ordering and ranges, and gives the planner real statistics on that expression (after `ANALYZE`).

### 6.9 Type and precision gotchas

```sql
-- numbers are numeric-normalized in jsonb
SELECT '{"x": 1.0}'::jsonb;              -- {"x": 1.0}  (trailing zero kept in modern PG's numeric...)
SELECT '{"x": 1e3}'::jsonb;              -- {"x": 1000}
SELECT '{"x": 0.1}'::jsonb = '{"x": 0.10}'::jsonb;   -- true (numeric equality)

-- JSON null vs SQL NULL — a constant source of bugs
SELECT '{"a": null}'::jsonb -> 'a';      -- 'null'::jsonb   (a JSON null value)
SELECT '{"a": null}'::jsonb ->> 'a';     -- NULL (SQL null — text of json null is null)
SELECT '{"b": 1}'::jsonb -> 'a';         -- NULL (SQL null — key absent)
SELECT ('{"a": null}'::jsonb -> 'a') IS NULL;      -- false! it's json null, not sql null
SELECT ('{"a": null}'::jsonb ->> 'a') IS NULL;     -- true

-- distinguishing "key absent" from "key present with json null"
SELECT '{"a": null}'::jsonb ? 'a';       -- true  (key exists)
SELECT '{"b": 1}'::jsonb ? 'a';          -- false (key absent)
```

The `->` vs `->>` NULL asymmetry is the most common JSON bug: `WHERE payload->>'field' IS NOT NULL` treats JSON-null the same as absent (both become SQL NULL under `->>`), while `WHERE payload ? 'field'` distinguishes presence. Know which semantic you want.

### 6.10 The update / write-amplification problem

```sql
-- every one of these rewrites the ENTIRE document and its TOAST chunks
UPDATE events SET payload = jsonb_set(payload, '{views}', to_jsonb((payload->>'views')::int + 1))
WHERE id = 1;
```

Because `jsonb` is immutable-on-disk, incrementing a counter inside a 100KB document:
- produces a new ~100KB document,
- writes a new heap tuple (old one becomes dead → bloat until VACUUM),
- rewrites the TOAST chunks,
- writes the full-page/row image to WAL.

Doing this per request is a classic self-inflicted performance incident. If a field is hot and mutated frequently, it should be a **real column** (a `bigint views`), not a JSON key. JSONB is for data you write-once/read-many, or mutate rarely.

### 6.11 When JSONB is right vs when it is a schema smell

**Good fits for JSONB:**
- Genuinely schemaless / variable-shape data: third-party webhook payloads, event envelopes, feature flags, per-tenant custom fields, sparse attributes that differ per row.
- Store-and-return blobs you rarely filter on: an API response cached verbatim.
- A denormalized read-model / audit snapshot where the whole document is read together.

**Schema smells (model it relationally instead):**
- Fields that are **always present and always the same shape** (`email`, `status`, `created_at`) — these want real typed columns with NOT NULL, defaults, and B-tree indexes.
- Fields you **frequently filter, join, or aggregate on** — the planner's poor JSON statistics produce bad plans; a real column gives histograms and accurate estimates.
- Fields with **referential meaning** (a `user_id` that should be a foreign key) — JSONB cannot enforce FKs.
- **Hot, frequently-mutated** fields — write amplification (6.10).
- Using JSONB to store a **list of child rows** that you query individually — that is a child table with a foreign key, not an array.

The senior heuristic: *If you know the shape and you query the field, make it a column. Use JSONB for the parts that are genuinely unknown, variable, or read as an opaque whole.* A hybrid table — typed columns for the known/queried fields plus one `jsonb` column for the variable remainder — is usually the right production design.

---

## 7. EXPLAIN — JSONB Filters in the Plan

### 7.1 Seq Scan — the un-indexed `@>` filter

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, payload->>'status'
FROM events
WHERE payload @> '{"status":"failed"}';
```
```
Seq Scan on events  (cost=0.00..48210.00 rows=5000 width=40)
                    (actual time=0.4..612.8 rows=4871 loops=1)
  Filter: (payload @> '{"status": "failed"}'::jsonb)
  Rows Removed by Filter: 995129
  Buffers: shared hit=1024 read=41186
Planning Time: 0.2 ms
Execution Time: 618.4 ms
```
Reading it: no index, so every one of 1M rows is read and de-TOASTed, `@>` runs per row, 995K discarded. `read=41186` is heap pages pulled from disk. `rows=5000` estimate is the planner's blind guess (poor JSON statistics) — actual is 4871, close here but often wildly off.

### 7.2 Bitmap Index Scan — GIN on `@>`

```sql
CREATE INDEX idx_events_gin ON events USING gin (payload jsonb_path_ops);

EXPLAIN (ANALYZE, BUFFERS)
SELECT id, payload->>'status'
FROM events
WHERE payload @> '{"status":"failed"}';
```
```
Bitmap Heap Scan on events  (cost=44.00..1820.00 rows=5000 width=40)
                            (actual time=1.1..8.9 rows=4871 loops=1)
  Recheck Cond: (payload @> '{"status": "failed"}'::jsonb)
  Heap Blocks: exact=3901
  Buffers: shared hit=112 read=3812
  ->  Bitmap Index Scan on idx_events_gin  (cost=0.00..42.75 rows=5000 width=0)
                                           (actual time=0.6..0.6 rows=4871 loops=1)
        Index Cond: (payload @> '{"status": "failed"}'::jsonb)
Buffers: shared hit=6 read=48
Planning Time: 0.3 ms
Execution Time: 10.2 ms
```
Reading it: the **Bitmap Index Scan** consults the GIN index (`Index Cond`), producing a bitmap of candidate heap TIDs in ~0.6ms reading only 54 index pages. The **Bitmap Heap Scan** then fetches those heap pages and **rechecks** the condition (`Recheck Cond`) — GIN can produce false positives, so a recheck against the actual `jsonb` is always required. 618ms → 10ms. GIN scans are almost always *bitmap* scans, not plain index scans, because they return many unordered TIDs.

### 7.3 B-tree expression index — scalar equality

```sql
CREATE INDEX idx_events_status ON events ((payload->>'status'));

EXPLAIN (ANALYZE)
SELECT id FROM events WHERE payload->>'status' = 'failed';
```
```
Index Scan using idx_events_status on events
    (cost=0.42..185.00 rows=4900 width=8) (actual time=0.03..2.1 rows=4871 loops=1)
  Index Cond: ((payload ->> 'status'::text) = 'failed'::text)
Planning Time: 0.2 ms
Execution Time: 3.0 ms
```
Reading it: a plain **Index Scan** (not bitmap, no recheck) on the expression index — the fastest of the three because it is a compact B-tree over one scalar and needs no de-TOAST to evaluate the predicate (the value is in the index). This is why an expression index beats GIN for single-field equality. Note the estimate improves once you `ANALYZE` — Postgres collects statistics on the indexed expression.

### 7.4 The set-returning fan-out in the plan

```sql
EXPLAIN (ANALYZE)
SELECT e.id, elem->>'sku'
FROM events e, jsonb_array_elements(e.payload->'items') elem
WHERE e.id = 42;
```
```
Nested Loop  (cost=0.42..12.50 rows=100 width=40) (actual rows=6 loops=1)
  ->  Index Scan using events_pkey on events e  (actual rows=1 loops=1)
        Index Cond: (id = 42)
  ->  Function Scan on jsonb_array_elements elem  (actual rows=6 loops=1)
Planning Time: 0.2 ms
Execution Time: 0.1 ms
```
Reading it: the `Function Scan` on `jsonb_array_elements` is the fan-out — one input row (id=42) with a 6-element `items` array becomes 6 output rows via a Nested Loop (implicit LATERAL). The `rows=100` estimate is the planner's fixed default for set-returning functions — it has no idea how many elements the array holds, another source of bad estimates.

---

## 8. Query Examples

### Example 1 — Basic: extract fields and filter with containment

```sql
-- Pull id, customer name, and status text out of an event envelope,
-- keeping only paid events. @> is index-friendly; ->> extracts scalars.
SELECT
  e.id,
  e.payload ->> 'event_type'          AS event_type,
  e.payload #>> '{customer,name}'     AS customer_name,   -- deep path to text
  (e.payload ->> 'amount')::numeric   AS amount
FROM events e
WHERE e.payload @> '{"status":"paid"}'                    -- GIN-accelerated filter
ORDER BY (e.payload ->> 'amount')::numeric DESC
LIMIT 50;
```

### Example 2 — Intermediate: fan-out an array and aggregate

```sql
-- Each event has payload->'items' = [{"sku":..,"qty":..,"price":..}, ...].
-- Expand items to rows, then aggregate revenue per SKU across all paid events.
-- CROSS JOIN LATERAL makes the one-to-many fan-out explicit (Topic 11 mental model).
SELECT
  item ->> 'sku'                                        AS sku,
  SUM((item->>'qty')::int)                              AS units,
  SUM((item->>'qty')::int * (item->>'price')::numeric)  AS revenue
FROM events e
CROSS JOIN LATERAL jsonb_array_elements(e.payload -> 'items') AS item
WHERE e.payload @> '{"status":"paid"}'
GROUP BY item ->> 'sku'
ORDER BY revenue DESC;
```

### Example 3 — Production Grade: build a nested document, index-backed

```sql
-- Context: events table, 40M rows, ~4KB avg payload (TOASTed).
-- Index available: CREATE INDEX idx_events_gin ON events USING gin (payload jsonb_path_ops);
--                  CREATE INDEX idx_events_created ON events (received_at);
-- Goal: for failed payment events in the last 24h, return one row per customer
--       with a nested JSON summary of their failures. Perf target: < 50ms.
--
-- Strategy: GIN index answers @> '{"event_type":"payment_failed"}', the received_at
-- B-tree bounds the time window; we aggregate into a jsonb document with jsonb_agg.

WITH recent_failures AS (
  SELECT
    e.payload #>> '{customer,id}'          AS customer_id,
    e.payload ->> 'error_code'             AS error_code,
    (e.payload ->> 'amount')::numeric      AS amount,
    e.received_at
  FROM events e
  WHERE e.payload @> '{"event_type":"payment_failed"}'   -- GIN
    AND e.received_at >= now() - interval '24 hours'      -- B-tree range
)
SELECT
  rf.customer_id,
  COUNT(*)                                 AS failure_count,
  SUM(rf.amount)                           AS total_failed_amount,
  jsonb_agg(
    jsonb_build_object(
      'error_code', rf.error_code,
      'amount',     rf.amount,
      'at',         rf.received_at
    ) ORDER BY rf.received_at DESC
  )                                        AS failures      -- nested doc per customer
FROM recent_failures rf
GROUP BY rf.customer_id
HAVING COUNT(*) >= 3                         -- customers with 3+ failures (fraud signal)
ORDER BY failure_count DESC
LIMIT 100;
```
```
-- EXPLAIN (ANALYZE) shape:
Limit  (actual time=31.2..31.4 rows=100 loops=1)
  ->  Sort  (actual time=31.2..31.3 rows=214 loops=1)
        Sort Key: (count(*)) DESC
        ->  HashAggregate  (actual time=29.8..30.6 rows=214 loops=1)
              Filter: (count(*) >= 3)
              ->  Bitmap Heap Scan on events e  (actual time=2.1..22.4 rows=9832 loops=1)
                    Recheck Cond: (payload @> '{"event_type": "payment_failed"}'::jsonb)
                    Filter: (received_at >= now() - '24:00:00'::interval)
                    ->  Bitmap Index Scan on idx_events_gin  (actual time=1.6..1.6 rows=41003 loops=1)
                          Index Cond: (payload @> '{"event_type": "payment_failed"}'::jsonb)
Execution Time: 42.9 ms
```
The GIN index narrows 40M rows to ~41K candidates, the recheck + time filter cut to ~9.8K, aggregation builds the nested `failures` document per customer. Meets the 50ms target. Note the GIN scan returns 41K but only 9.8K survive the time filter — if failures were common, adding `received_at` into the filtering strategy (e.g. a partial index or BRIN on `received_at`) would tighten it further.

---

## 9. Wrong → Right Patterns

### Wrong 1: Using `json` for a column you constantly query

```sql
-- WRONG: column typed as json — every read re-parses the whole document
CREATE TABLE events (id bigint, payload json);
SELECT id FROM events WHERE payload->>'status' = 'failed';
-- Every row's text is re-parsed from byte 0 to find 'status'. No index is possible
-- (GIN needs jsonb). On 10M rows this is a slow Seq Scan with per-row parse cost,
-- and there is no CREATE INDEX ... USING gin available at all.

-- RIGHT: use jsonb + the appropriate index
CREATE TABLE events (id bigint, payload jsonb);
CREATE INDEX ON events USING gin (payload jsonb_path_ops);      -- for @> filters
CREATE INDEX ON events ((payload->>'status'));                  -- for scalar equality
SELECT id FROM events WHERE payload->>'status' = 'failed';      -- uses B-tree expr index
```
Why it's wrong at the execution level: `json` stores text with no parse tree, so every operator re-parses O(document size) and the type is fundamentally un-GIN-indexable. `jsonb` parses once at write time and exposes a structure the planner can index.

### Wrong 2: Expecting a GIN index to accelerate `->>` equality

```sql
-- WRONG: GIN index present, but the filter uses ->> equality
CREATE INDEX idx_gin ON events USING gin (payload);
SELECT * FROM events WHERE payload->>'status' = 'failed';
-- EXPLAIN shows a Seq Scan — the jsonb_ops GIN index does NOT accelerate ->> = .
-- Developers see "I have a GIN index" and assume all JSON filters are fast. They aren't.

-- RIGHT (option A): rewrite the predicate as containment so GIN applies
SELECT * FROM events WHERE payload @> '{"status":"failed"}';    -- uses GIN

-- RIGHT (option B): add a B-tree expression index for the scalar path
CREATE INDEX idx_status ON events ((payload->>'status'));
SELECT * FROM events WHERE payload->>'status' = 'failed';       -- uses idx_status
```
Why it's wrong: GIN indexes the containment/existence operators (`@>`, `?`, JSONPath), not arbitrary functional expressions. `->>` extraction equality needs either a containment rewrite or its own expression index.

### Wrong 3: Double-counting after array fan-out

```sql
-- WRONG: fan out items, then SUM a top-level field that repeats per item
SELECT
  e.id,
  (e.payload->>'shipping_fee')::numeric AS shipping,   -- one value per event...
  SUM((item->>'qty')::int)              AS units
FROM events e
CROSS JOIN LATERAL jsonb_array_elements(e.payload->'items') item
GROUP BY e.id, (e.payload->>'shipping_fee')::numeric;
-- If you later SUM(shipping) across the fanned-out set, each event's shipping_fee
-- is counted once PER ITEM — a 5-item order counts shipping 5×. Same fan-out bug
-- as COUNT after a one-to-many JOIN in Topic 11.

-- RIGHT: aggregate the array in a subquery, keep event-level fields un-multiplied
SELECT
  e.id,
  (e.payload->>'shipping_fee')::numeric AS shipping,
  items.units
FROM events e
CROSS JOIN LATERAL (
  SELECT SUM((item->>'qty')::int) AS units
  FROM jsonb_array_elements(e.payload->'items') item
) items;
-- Now each event contributes exactly one row; shipping is never multiplied.
```

### Wrong 4: `jsonb_set` silently doing nothing (or nulling the row)

```sql
-- WRONG: set a nested field that doesn't exist yet, default create_missing behavior misunderstood
UPDATE events
SET payload = jsonb_set(payload, '{meta,reviewed}', 'true')
WHERE id = 1;
-- If payload has no 'meta' object, the path {meta,reviewed} cannot be built:
-- jsonb_set returns the document UNCHANGED (the intermediate 'meta' is missing).
-- Worse: if payload itself is SQL NULL, jsonb_set returns NULL and you wipe the column.

-- RIGHT: guard NULL, and ensure the parent object exists
UPDATE events
SET payload = jsonb_set(
      COALESCE(payload, '{}'::jsonb) || '{"meta":{}}'::jsonb,  -- ensure meta exists (shallow)
      '{meta,reviewed}', 'true', true                          -- create_missing = true
    )
WHERE id = 1;
-- COALESCE guards NULL; ensuring 'meta' exists lets the nested path resolve.
```

### Wrong 5: `@> '{}'` matching every row

```sql
-- WRONG: filter object built dynamically in app code, sometimes empty
SELECT * FROM events WHERE payload @> $1;   -- $1 = '{}' when no filter chosen
-- '{}'::jsonb is contained by EVERYTHING → returns the entire table.
-- A "no filter selected" case silently becomes "select all", often with no LIMIT.

-- RIGHT: skip the predicate entirely when the filter object is empty
-- (application side), or guard in SQL:
SELECT * FROM events
WHERE ($1 = '{}'::jsonb) OR (payload @> $1);   -- explicit, but prefer app-side skip
-- Best practice: build the WHERE clause conditionally so an empty filter
-- does not emit a @> predicate at all.
```

---

## 10. Performance Profile

### Storage and read cost

| Aspect | `json` | `jsonb` |
|---|---|---|
| Write CPU | low (validate) | higher (parse + binary serialize) |
| Read CPU per field access | O(size), re-parse | O(log keys), binary search |
| On-disk size | text, compressible | binary; often slightly larger before TOAST compression |
| Indexable | no (except functional expr on extracted scalar) | yes (GIN, B-tree expr) |

### Scaling the un-indexed `@>` filter

| Rows | No index (Seq Scan) | `jsonb_path_ops` GIN | B-tree expr index (scalar =) |
|---|---|---|---|
| 1M | ~600ms | ~10ms | ~3ms |
| 10M | ~6s | ~30–60ms | ~5ms |
| 100M | ~60s+ | ~150–400ms | ~10–20ms |

GIN scan cost grows with the number of matching TIDs (posting-list length) and heap-fetch/recheck cost, not with total table size — so a *selective* `@>` stays fast even at 100M rows. A B-tree expression index on a single hot field is the fastest and smallest when the query is always scalar equality/range on that field.

### `work_mem`, `maintenance_work_mem`, and GIN

- **Building** a GIN index is memory-hungry; raise `maintenance_work_mem` before `CREATE INDEX` on large tables to avoid slow, spill-heavy builds.
- **GIN insert cost** at write time is nontrivial — every key/value token must be inserted into the inverted index. High-write tables pay for GIN on every INSERT/UPDATE. The `fastupdate` GIN option batches these into a pending list (flushed by autovacuum or at threshold), trading read latency for write throughput. For write-heavy JSONB, consider `jsonb_path_ops` (fewer tokens → cheaper inserts) or index only the fields you actually filter via expression indexes.

### TOAST and the large-document trap

- A `jsonb` value over ~2KB is compressed and may move out-of-line to a TOAST table. Reading *any* field de-TOASTs (decompresses) the *whole* value — there is no partial read.
- Consequence: extracting one small field from a 500KB document at high QPS is dominated by decompression cost, not JSON traversal. If you frequently read one field from a large document, **promote that field to its own column** (or a generated column, below).

### Generated columns — index and read a hot field cheaply

```sql
ALTER TABLE events
  ADD COLUMN status text GENERATED ALWAYS AS (payload->>'status') STORED;
CREATE INDEX ON events (status);
-- 'status' is materialized as a real, cheap-to-read, index-able column,
-- kept in sync automatically. Reads of status avoid de-TOASTing the payload.
```

### Optimization checklist specific to JSONB

1. Use `jsonb`, never `json`, for anything queried.
2. `@>`/JSONPath filters → `jsonb_path_ops` GIN (smaller, faster) unless you need `?`.
3. Single hot scalar field with equality/range → B-tree expression or generated column, not GIN.
4. Hot mutated field → real column, never a JSON key (write amplification).
5. Large documents with a frequently-read field → generated STORED column to avoid de-TOAST.
6. Watch estimates: JSON predicate row estimates are unreliable; verify plans with `EXPLAIN ANALYZE`, and consider `CREATE STATISTICS` or expression-index statistics for better selectivity.

---

## 11. Node.js Integration

### 11.1 Reading JSONB — `pg` parses it to a JS object automatically

```javascript
import pg from 'pg';
const { Pool } = pg;
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

async function getPaidEvents(limit = 50) {
  const { rows } = await pool.query(
    `SELECT
       id,
       payload ->> 'event_type'        AS event_type,
       payload #>> '{customer,name}'   AS customer_name,
       payload                          AS payload      -- whole jsonb comes back as a JS object
     FROM events
     WHERE payload @> $1::jsonb                          -- containment, GIN-accelerated
     ORDER BY (payload ->> 'amount')::numeric DESC
     LIMIT $2`,
    [{ status: 'paid' }, limit]                          // pass a JS object; see 11.2
  );
  // rows[i].payload is already a parsed object (node-pg parses jsonb/json OIDs)
  return rows;
}
```

### 11.2 Writing JSONB — bind a JS object, cast the placeholder

```javascript
async function insertEvent(payload) {
  // node-pg does NOT auto-serialize an object to jsonb for a bare $1.
  // Two safe patterns:
  //   (a) JSON.stringify and cast:  $1::jsonb  with  JSON.stringify(payload)
  //   (b) pass the object and cast: $1::jsonb  — pg serializes objects for json/jsonb params
  // The explicit, portable form is (a):
  const { rows } = await pool.query(
    `INSERT INTO events (payload) VALUES ($1::jsonb) RETURNING id`,
    [JSON.stringify(payload)]
  );
  return rows[0].id;
}
```

### 11.3 The `@>` / `?` parameter and the placeholder collision

```javascript
// The jsonb ? operator (key existence) can collide with query-parameter parsing
// in libraries that use '?' placeholders. node-pg uses $1, so '?' is safe here —
// but prefer the function form for portability and clarity:
async function eventsHavingKey(key) {
  const { rows } = await pool.query(
    `SELECT id FROM events WHERE jsonb_exists(payload, $1)`,  // == payload ? $1
    [key]
  );
  return rows;
}
```

### 11.4 Updating a field — and why you often should not

```javascript
// Mutating one field rewrites the whole document (write amplification).
// If 'reviewed' is hot, make it a real column instead. If it is genuinely rare:
async function markReviewed(id) {
  await pool.query(
    `UPDATE events
     SET payload = jsonb_set(COALESCE(payload,'{}'::jsonb), '{reviewed}', 'true', true)
     WHERE id = $1`,
    [id]
  );
}
```

### 11.5 Aggregating array elements safely

```javascript
async function revenueBySku() {
  const { rows } = await pool.query(
    `SELECT item ->> 'sku'                                       AS sku,
            SUM((item->>'qty')::int * (item->>'price')::numeric) AS revenue
     FROM events e
     CROSS JOIN LATERAL jsonb_array_elements(e.payload -> 'items') AS item
     WHERE e.payload @> '{"status":"paid"}'
     GROUP BY item ->> 'sku'
     ORDER BY revenue DESC`
  );
  return rows;   // revenue comes back as string (numeric); parse if you need a JS number
}
```

Note on types: `pg` returns `numeric`/`bigint` as **strings** to avoid float precision loss. `(payload->>'amount')::numeric` sums are strings in JS — wrap with `Number()` or a decimal library deliberately. `jsonb` columns and JSON scalars, by contrast, arrive as parsed JS values.

---

## 12. ORM Comparison

### Prisma

**Can Prisma do JSONB?** — Yes. Map a field to `Json` and Prisma uses `jsonb` on Postgres by default. It supports a set of JSON filters (`path`, `equals`, `string_contains`, `array_contains`, and `@>`-style filters via `path`).

```typescript
// schema.prisma:  payload Json
const events = await prisma.event.findMany({
  where: {
    payload: {
      path: ['status'],          // navigate to payload->>'status'
      equals: 'failed',
    },
  },
  take: 50,
});

// containment-style filter
const paid = await prisma.event.findMany({
  where: { payload: { path: ['status'], equals: 'paid' } },
});

// deep read + aggregation → raw SQL
const bySku = await prisma.$queryRaw`
  SELECT item->>'sku' AS sku,
         SUM((item->>'qty')::int * (item->>'price')::numeric) AS revenue
  FROM events e
  CROSS JOIN LATERAL jsonb_array_elements(e.payload->'items') item
  WHERE e.payload @> '{"status":"paid"}'
  GROUP BY item->>'sku'
  ORDER BY revenue DESC`;
```

**Where it breaks**: Prisma's JSON filters cannot express `jsonb_array_elements` fan-out, JSONPath predicates (`@@`, `@?`), `jsonb_set` mutations of nested paths, or `jsonb_agg` construction. It also cannot declare a GIN index in the schema for older versions cleanly (use raw migration SQL). **Verdict**: fine for simple path-equality reads; use `$queryRaw` for containment aggregation, JSONPath, and any building/mutation.

---

### Drizzle ORM

**Can Drizzle do JSONB?** — Yes, with a typed `jsonb` column and the `sql` template for operators.

```typescript
import { jsonb, bigint, pgTable } from 'drizzle-orm/pg-core';
import { sql } from 'drizzle-orm';

export const events = pgTable('events', {
  id: bigint('id', { mode: 'number' }).primaryKey(),
  payload: jsonb('payload').$type<{ status: string; items: any[] }>(),
});

// containment filter via sql template (GIN-friendly)
const paid = await db.select().from(events)
  .where(sql`${events.payload} @> '{"status":"paid"}'::jsonb`);

// scalar extraction filter
const failed = await db.select().from(events)
  .where(sql`${events.payload} ->> 'status' = 'failed'`);

// fan-out aggregation
const bySku = await db.execute(sql`
  SELECT item->>'sku' AS sku,
         SUM((item->>'qty')::int * (item->>'price')::numeric) AS revenue
  FROM events e
  CROSS JOIN LATERAL jsonb_array_elements(e.payload->'items') item
  WHERE e.payload @> '{"status":"paid"}'::jsonb
  GROUP BY item->>'sku'`);
```

**Where it breaks**: operators (`@>`, `->>`, `#>>`, JSONPath) all go through the `sql` tag — there is no typed operator DSL, so you lose compile-time checking on the JSON path itself. The `$type<>()` annotation types the column value but not the operator results. **Verdict**: excellent — typed column, transparent SQL for operators, no impedance mismatch.

---

### Sequelize

**Can Sequelize do JSONB?** — Yes, `DataTypes.JSONB`, with nested-path filtering via object where-clauses.

```javascript
const Event = sequelize.define('Event', {
  payload: DataTypes.JSONB,
});

// nested path equality: payload.status = 'failed'
const failed = await Event.findAll({
  where: { 'payload.status': 'failed' },        // generates payload->>'status' = 'failed'
});

// containment / JSONPath / fan-out → raw
const bySku = await sequelize.query(
  `SELECT item->>'sku' AS sku,
          SUM((item->>'qty')::int * (item->>'price')::numeric) AS revenue
   FROM events e
   CROSS JOIN LATERAL jsonb_array_elements(e.payload->'items') item
   WHERE e.payload @> '{"status":"paid"}'::jsonb
   GROUP BY item->>'sku'`,
  { type: QueryTypes.SELECT }
);
```

**Where it breaks**: the `'payload.status'` dotted syntax generates `->>` extraction (no GIN acceleration) and is easy to typo silently. No support for `@>`, `?`, JSONPath, or `jsonb_set` through the query API. **Verdict**: usable for simple path reads; drop to `sequelize.query()` for anything indexed or structural.

---

### TypeORM

**Can TypeORM do JSONB?** — Yes, `@Column('jsonb')`, with raw operators in `QueryBuilder`.

```typescript
@Column('jsonb', { nullable: true })
payload: Record<string, any>;

// containment filter
const paid = await repo.createQueryBuilder('e')
  .where(`e.payload @> :f::jsonb`, { f: JSON.stringify({ status: 'paid' }) })
  .getMany();

// scalar extraction
const failed = await repo.createQueryBuilder('e')
  .where(`e.payload ->> 'status' = :s`, { s: 'failed' })
  .getMany();
```

**Where it breaks**: all JSON operators are raw strings in `.where()` — no type safety, and the GIN index must be declared with `@Index()` using a raw expression or a manual migration. No first-class JSONPath or `jsonb_set` helpers. **Verdict**: works via raw predicates; the ORM adds little over hand-written SQL for JSON.

---

### Knex.js

**Can Knex do JSONB?** — Yes, with dedicated helpers (`jsonExtract`, `whereJsonPath`, `whereJsonObject`) plus `knex.raw` for operators.

```javascript
// scalar path extraction
const failed = await knex('events')
  .whereRaw(`payload ->> 'status' = ?`, ['failed'])
  .select('id');

// containment
const paid = await knex('events')
  .whereRaw(`payload @> ?::jsonb`, [JSON.stringify({ status: 'paid' })]);

// fan-out aggregation
const bySku = await knex.raw(`
  SELECT item->>'sku' AS sku,
         SUM((item->>'qty')::int * (item->>'price')::numeric) AS revenue
  FROM events e
  CROSS JOIN LATERAL jsonb_array_elements(e.payload->'items') item
  WHERE e.payload @> '{"status":"paid"}'::jsonb
  GROUP BY item->>'sku'`);
```

**Where it breaks**: `whereJsonPath` and friends cover common cases but map to `->>`/JSONPath, not containment; for `@>`, `?`, `jsonb_set`, and JSONPath predicates you use `whereRaw`/`knex.raw`. **Verdict**: the most transparent — helpers for the easy paths, raw for the rest, no hidden behavior.

---

### ORM Summary Table

| ORM | JSONB column | Path read | `@>` containment | JSONPath / fan-out / build | Verdict |
|-----|-------------|-----------|------------------|----------------------------|---------|
| Prisma | `Json` (jsonb) | `path`+`equals` | limited | raw SQL | Good for simple reads |
| Drizzle | `jsonb()` typed | `sql` tag | `sql` tag | `sql` tag / `execute` | Best transparency + types |
| Sequelize | `JSONB` | `'col.path'` | raw | raw | Path reads only |
| TypeORM | `@Column('jsonb')` | raw `.where` | raw `.where` | raw | Raw predicates |
| Knex | `jsonb` | `whereJsonPath`/raw | `whereRaw` | `knex.raw` | Transparent, raw for structural |

Across all five: **path-equality reads are supported; containment, JSONPath, fan-out aggregation, and mutation almost always require raw SQL.** JSONB is a place where the raw-SQL escape hatch is the norm, not the exception.

---

## 13. Practice Exercises

### Exercise 1 — Basic

Given `events(id bigint, payload jsonb, received_at timestamptz)` where each payload looks like `{"event_type":"login","user":{"id":42,"name":"Ada"},"ip":"1.2.3.4"}`:

Write a query returning `id`, the `event_type`, the nested `user.name`, and `user.id` (as an integer), for every event whose `event_type` is `"login"`. Use a containment filter so a GIN index could accelerate it. Order by `received_at` descending.

Then: which index would make the filter fast, and why `payload->>'event_type' = 'login'` would *not* use a plain `jsonb_ops` GIN index?

```sql
-- Write your query here
```

---

### Exercise 2 — Medium (combines fan-out + aggregation, Topic 11)

Each `events` payload for a completed order has `{"status":"paid","items":[{"sku":"A","qty":2,"price":9.99}, ...],"shipping_fee":5.00}`.

Write a query returning, per `sku`, across all paid events: `units_sold`, `gross_revenue` (sum of `qty*price`), and `order_count` (number of distinct events that contain that sku). Ensure the per-event `shipping_fee` is **not** multiplied by the item fan-out. Order by `gross_revenue` descending.

```sql
-- Write your query here
```

---

### Exercise 3 — Hard (production simulation; the naive answer is wrong)

A `sessions(id, user_id, data jsonb)` table stores `data` like `{"flags":{"beta":true,"dark_mode":false},"cart":[{"sku":"X","qty":1}]}`. Ops report that this query is slow and returns wrong counts:

```sql
SELECT u.id, COUNT(*) AS beta_users
FROM users u
JOIN sessions s ON s.user_id = u.id
WHERE s.data->>'flags' = '{"beta":true,"dark_mode":false}'
GROUP BY u.id;
```

1. Identify why the `WHERE` is both wrong (semantically) and un-indexable as written.
2. Rewrite it to correctly find users who have at least one session with `flags.beta = true`, using a predicate a GIN index can accelerate.
3. Make it return each such user once (no fan-out duplicates) with their count of beta sessions.
4. State the index you would create and its operator class, and whether a `jsonb_path_ops` index suffices.

```sql
-- Write your query here
```

---

### Exercise 4 — Interview Simulation

You are handed a `products(id, attributes jsonb)` table. Product attributes are genuinely heterogeneous (a laptop has `ram_gb`, a shirt has `size` and `color`). The team wants to: (a) filter products by arbitrary attribute equality quickly, (b) find all products that have a given attribute key at all, and (c) run a report of "products where price > 100 stored inside attributes."

1. Which single index (or indexes) supports (a) and (b), and what is the operator-class tradeoff?
2. Write the query for (c) using JSONPath, and explain why `@>` cannot express it.
3. The lead proposes moving `price` out of `attributes` into a real `numeric` column. Argue for or against, referencing statistics, indexing, and write amplification.

```sql
-- Write your query here
```

---

## 14. Interview Questions

### Q1 — What is the difference between `json` and `jsonb`, and which should you default to?

**Junior answer**: `jsonb` is binary and `json` is text. Use `jsonb` because it's faster.

**Principal answer**: `json` stores the document as verbatim text with only a validation check — it preserves whitespace, key order, and duplicate keys, but every access re-parses the whole document (O(size)) and it cannot be GIN-indexed. `jsonb` parses once at write time into a decomposed binary form: keys are sorted and de-duplicated, scalars stored natively, key lookup is a binary search, and — critically — it supports GIN indexes and the containment/existence/JSONPath operators. The tradeoff is slightly higher write cost and loss of exact textual fidelity. Default to `jsonb` for anything you query; reach for `json` only when you must reproduce the exact original bytes — e.g. verifying a webhook signature computed over the raw body, where reordered keys would break the HMAC.

**Follow-up the interviewer asks**: *"You said jsonb de-duplicates keys — walk me through what `'{"a":1,"a":2}'::jsonb` produces and why that matters for an idempotency key stored in JSON."* (Answer: `{"a": 2}` — last wins; if two systems disagree on which duplicate is authoritative, jsonb silently picks the last, so never rely on duplicate keys.)

---

### Q2 — A `WHERE payload @> '{"status":"failed"}'` on 50M rows takes 40 seconds. How do you fix it, and how do you fix `WHERE payload->>'status' = 'failed'`?

**Junior answer**: Add an index on the payload column.

**Principal answer**: These need *different* indexes because different operators are involved. For `@>`, create a GIN index — `CREATE INDEX ... USING gin (payload jsonb_path_ops)`. `jsonb_path_ops` hashes each path-to-value into one token, so it is 2–3× smaller than the default `jsonb_ops` and usually faster for containment, with fewer false positives at recheck. The plan becomes a Bitmap Index Scan feeding a Bitmap Heap Scan with a Recheck (GIN can yield false positives, so a recheck against the real jsonb is mandatory). For `payload->>'status' = 'failed'`, a GIN index does **not** help — GIN accelerates `@>`/`?`/JSONPath, not functional extraction equality. Either rewrite the predicate as `@> '{"status":"failed"}'` to hit the GIN index, or create a B-tree expression index `((payload->>'status'))`, which is smaller, supports ranges and ordering, and gives the planner real statistics on that expression after ANALYZE. If `status` is a hot, always-present field, the best answer is often to promote it to a generated `STORED` column and index that.

**Follow-up**: *"Your GIN index is on a write-heavy table and inserts got slower. Why, and what's the lever?"* (GIN must insert every token per row; the `fastupdate` pending-list batches inserts, and `jsonb_path_ops` produces fewer tokens — or index only the specific fields via expression indexes.)

---

### Q3 — When is storing data in JSONB a good decision, and when is it a schema smell?

**Junior answer**: JSONB is flexible, so it's good when you might add fields later without migrations.

**Principal answer**: JSONB is right for genuinely variable-shape data — third-party webhook payloads, event envelopes, per-tenant custom fields, sparse attributes that differ per row, or a blob read as an opaque whole. It is a schema smell when it holds fields that are always present and always the same shape, fields you frequently filter/join/aggregate on, fields with referential meaning (a `user_id` that should be a foreign key), or hot frequently-mutated fields. The costs you eat by hiding structured data in JSONB: no NOT NULL / type checking / foreign keys, no cheap B-tree indexes with honest statistics (the planner's JSON selectivity estimates are crude and produce bad plans), and severe write amplification on mutation because `jsonb_set` rewrites the entire document and re-TOASTs it, generating a new MVCC tuple and full WAL each time. The senior heuristic: if you know the shape and you query the field, make it a column; use JSONB only for the parts that are genuinely unknown, variable, or read as a whole — a hybrid table (typed columns + one jsonb column for the remainder) is usually the production-correct design.

**Follow-up**: *"You have a 200KB JSONB document and you increment a view counter inside it on every request. What happens at scale?"* (Each update rewrites ~200KB, re-TOASTs, writes a dead tuple + full WAL image → massive write amplification and bloat; the counter must be a real `bigint` column.)

---

### Q4 — Explain the difference between `->` and `->>`, and the JSON-null vs SQL-null trap.

**Junior answer**: `->` gives you JSON and `->>` gives you text.

**Principal answer**: `->` returns `jsonb` (keeps the type, so you can chain further JSON operators); `->>` returns `text` (unwraps a JSON string, losing its quotes, and ends the chain). The trap is null handling: `'{"a":null}'::jsonb -> 'a'` returns the *jsonb value* `'null'` (JSON null, which is NOT SQL NULL — `IS NULL` is false), while `'{"a":null}'::jsonb ->> 'a'` returns SQL NULL, indistinguishable from an *absent* key which also yields SQL NULL under `->`/`->>`. So `WHERE payload->>'field' IS NOT NULL` cannot tell "key present with JSON null" apart from "key absent" — both are SQL NULL. To distinguish presence, use the `?` operator: `payload ? 'field'` is true only when the key exists, regardless of its value. Knowing which semantic your query needs is the difference between a correct and a subtly-wrong filter.

**Follow-up**: *"Write a predicate for 'the key exists and its value is not JSON null'."* (`payload ? 'field' AND payload->'field' <> 'null'::jsonb`.)

---

## 15. Mental Model Checkpoint

1. You insert `'{"z":1,"a":2,"a":3}'` into a `json` column and a `jsonb` column. What comes back out of each on `SELECT`, and why do they differ in three distinct ways?

2. You have a GIN index on `payload`. Which of these use it: `payload @> '{"k":1}'`, `payload ? 'k'`, `payload->>'k' = '1'`, `payload @@ '$.k == 1'`? What changes if the index is `jsonb_path_ops` instead of `jsonb_ops`?

3. An event's `payload->'items'` is a 4-element array. You write `SELECT e.id, SUM((e.payload->>'total')::numeric) FROM events e CROSS JOIN LATERAL jsonb_array_elements(e.payload->'items') GROUP BY e.id`. Why is the SUM wrong, and what is it wrong by?

4. Why does reading one 20-byte field out of a 400KB TOASTed `jsonb` value cost far more than the 20 bytes suggests? What single schema change fixes it without denormalizing everything?

5. `jsonb_set(payload, '{meta,verified}', 'true')` runs on 1,000 rows and appears to do nothing for most of them. Give two independent reasons this can happen.

6. You filter with `payload @> $1` where `$1` is built by your application. In production it occasionally returns the entire table. What value of `$1` causes this, and why is it mathematically correct behavior?

7. When would you deliberately choose a B-tree expression index `((payload->>'status'))` over a GIN index, and when would you choose neither and use a generated column?

---

## 16. Quick Reference Card

```sql
-- TYPE CHOICE
jsonb   -- default: parsed binary, indexable, de-duplicated, key-sorted
json    -- only when you must preserve exact bytes/whitespace/key order/dupes

-- EXTRACTION            returns
a -> 'k'      -- jsonb   field/elem as jsonb (chainable)
a ->> 'k'     -- text    field/elem as text  (unwraps strings; ends chain)
a -> 2        -- jsonb   array index (negatives ok: -1 = last)
a #> '{x,y}'  -- jsonb   deep path
a #>> '{x,y}' -- text    deep path as text
-- missing key -> SQL NULL;  {"k":null}->'k' -> jsonb 'null' (NOT sql null)

-- CONTAINMENT / EXISTENCE (GIN-indexable)
a @> b        -- boolean  a contains b  (workhorse filter)   note: a @> '{}' is TRUE
a <@ b        -- boolean  a contained by b
a ? 'k'       -- boolean  top-level key (obj) / string elem (arr) exists  (keys, not values!)
a ?| array[]  -- boolean  ANY key exists
a ?& array[]  -- boolean  ALL keys exist

-- JSONPath (GIN-indexable via @?/@@)
a @? '$.x[*] ? (@.p > 100)'   -- boolean: any match?
a @@ '$.total > 100'          -- boolean: predicate true?
jsonb_path_query(a, '$...')   -- SET of matches
jsonb_path_query_array(...)   -- matches as one array

-- BUILD
jsonb_build_object('k',v,...) -- object (even # of args)
jsonb_build_array(v, ...)     -- array
to_jsonb(row_or_val)          -- convert
jsonb_agg(expr [ORDER BY ..]) -- aggregate rows -> array
jsonb_object_agg(k, v)        -- aggregate -> object

-- DECONSTRUCT (set-returning → FAN-OUT rows, like a one-to-many JOIN)
jsonb_each(a) / jsonb_each_text(a)          -- (key, value) rows
jsonb_array_elements(a) / _text(a)          -- one row per element
jsonb_object_keys(a)                        -- keys as rows

-- MUTATE (returns NEW doc → full-row rewrite when persisted)
jsonb_set(a, '{path}', newval [, create_missing])
jsonb_insert(a, '{path}', val [, insert_after])
a || b            -- SHALLOW merge (nested objects replaced, not deep-merged)
a - 'k'  |  a - 2  |  a #- '{path}'          -- remove key / index / path

-- INDEXES
CREATE INDEX ON t USING gin (col);                  -- jsonb_ops: keys+values; supports ? @> @@ @?
CREATE INDEX ON t USING gin (col jsonb_path_ops);   -- smaller/faster; @> @@ @? only, NO ?
CREATE INDEX ON t ((col->>'field'));                -- B-tree: for ->> = and ranges
ALTER TABLE t ADD COLUMN f text GENERATED ALWAYS AS (col->>'field') STORED;  -- hot field

-- PERF RULES OF THUMB
-- @> / JSONPath filter  -> GIN (jsonb_path_ops unless you need ?)
-- scalar = / range      -> B-tree expression index or generated column
-- hot mutated field     -> real column (jsonb_set rewrites whole doc → write amplification)
-- large doc, hot field  -> generated STORED column (avoids de-TOAST of whole value)
-- GIN scan => Bitmap Index Scan + Bitmap Heap Scan + Recheck (false positives possible)

-- INTERVIEW ONE-LINERS
-- "jsonb parses once; json re-parses every access."
-- "GIN accelerates @> / ? / JSONPath, never ->> equality."
-- "@> '{}' matches everything — guard dynamic filters."
-- "->> collapses JSON-null and absent-key both to SQL NULL; use ? to tell them apart."
-- "If you know the shape and query it, it's a column, not a JSON key."
```

---

## Connected Topics

- **Topic 11 — INNER JOIN in Depth**: `jsonb_array_elements`/`jsonb_each` fan-out multiplies rows exactly like a one-to-many JOIN; the double-counting and pre-aggregation lessons transfer directly.
- **Topic 63 — Date and Time Mastery** (previous): extracting and casting `payload->>'ts'` to `timestamptz`, and time-window filtering alongside JSONB containment predicates.
- **Topic 65 — Array Operations** (next): SQL arrays vs JSON arrays — when to use a native `text[]`/`int[]` column with GIN vs a JSON array, and the `@>`/`<@` operators they share.
- **Internals — GIN Indexes**: the inverted-index structure, posting lists, `fastupdate` pending list, and bitmap scans that make `@>` fast.
- **Internals — TOAST**: why large `jsonb` values are compressed/out-of-line and why any field read de-TOASTs the whole value.
- **Internals — MVCC & WAL**: why `jsonb_set` rewrites the whole document as a new tuple and the resulting write amplification and bloat.
- **Topic — Expression & Generated Columns**: promoting a hot JSON field to an indexed generated column.
- **Topic — Query Planner & Statistics**: why JSON predicate selectivity estimates are unreliable and how expression-index statistics improve them.
