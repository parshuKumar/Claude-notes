# 70 — Document Modelling
## Phase: Beyond Relational

---

## ELI5 — The Simple Analogy

Two ways to file a customer's order paperwork.

**The filing-cabinet way (relational):** the order form goes in the Orders drawer, the line items go in the Items drawer, the shipping address goes in the Addresses drawer. Nothing is duplicated. To see the whole order you walk to three drawers.

**The folder way (document):** one physical folder per order, containing the form, the items, the address, everything — stapled together. One trip. You pull the folder and you have it all.

★ **The folder is faster to fetch and worse at everything else.** Ask *"which orders contain item SKU-902?"* and you must open every folder. Ask *"change the tax rate on all pending orders"* and you rewrite every folder. Ask *"does this address actually exist in our address book?"* and nothing checks.

And the trap that catches people: ★ **the folder was chosen because "we fetch the whole order anyway."** Then a requirement arrives that needs to look *across* folders — and it always does, eventually. The folder shape isn't wrong; **it's a bet on which questions you'll ask, and that bet is hard to reverse.**

---

## Where this fits in the big picture

```
   20–28 relational modelling · 55 embedding as a denormalisation
   29–38 normalisation — one fact, one place
                          │
                          ▼
        ┌──────────────────────────────────────────────┐
        │ 70 DOCUMENT MODELLING ← YOU ARE HERE         │
        │ ★ when nesting is the PRIMARY shape,         │
        │   not a denormalisation                      │
        └────────────────────┬─────────────────────────┘
                             ▼
              71 key-value · 72 wide-column · 75 SQL vs NoSQL
```

★ **Topic 55 treated embedding as a denormalisation — a deliberate copy with an obligation.** This topic treats it as a **primary storage model**, and asks the question that actually matters: *when is nesting the right shape, and when have you just made joins impossible?*

---

## What is this?

Storing data as **self-contained, nested documents** (JSON/BSON) rather than as rows in related tables.

```json
{
  "_id": "ord_8842119",
  "customer": { "id": 42, "name": "Meera Sharma", "tier": "gold" },
  "items": [
    { "sku": "KUR-001", "qty": 2, "unit_price_minor": 49900 },
    { "sku": "DUP-014", "qty": 1, "unit_price_minor": 29900 }
  ],
  "shipping_address": { "line1": "…", "city": "Bengaluru", "pincode": "560001" },
  "status": "paid",
  "created_at": "2026-08-25T09:14:22+05:30"
}
```

★ **And the fact that reframes the whole topic: you do not need MongoDB to do this.** PostgreSQL's `jsonb` is a full document store with indexing, operators and constraints — **plus** transactions, foreign keys and joins for the parts that need them.

```
 ★ THE REAL CHOICE IS NOT "PostgreSQL OR MongoDB".
   IT IS: ★ WHICH PARTS OF MY DATA ARE DOCUMENT-SHAPED,
          AND WHICH ARE RELATIONAL-SHAPED?
   ⇒ ★ AND THE ANSWER IS ALMOST ALWAYS "SOME OF EACH".
```

**The three modelling decisions, which are the entire discipline:**

| Decision | Question |
|---|---|
| ★ **Embed or reference?** | does this child ever exist or change independently? |
| ★ **What is the aggregate boundary?** | what must be updated atomically together? |
| ★ **What is the document key?** | what do 95% of reads look up by? |

---

## Why does it matter for a backend developer?

```
 ★ BECAUSE THE MOST COMMON FAILURE IS NOT "WE CHOSE THE WRONG
   DATABASE" — IT IS "WE CHOSE THE WRONG DOCUMENT BOUNDARY".

 ★ THE THREE SYMPTOMS, AND THEY ARE ALL THE SAME BUG:
 ① ★ THE UNBOUNDED ARRAY
    a `comments` array inside a `post` document.
    ⇒ 200,000 comments ⇒ ★ a 40 MB document ⇒ every read fetches
      40 MB to show 20 comments ⇒ ★ and MongoDB's hard 16 MB
      document limit eventually rejects the write outright.
 ② ★ THE WRITE-AMPLIFIED DOCUMENT
    a counter inside a large document ⇒ ★ every increment rewrites
    the whole thing (Topic 46: the whole row is copied anyway).
 ③ ★ THE QUERY YOU CANNOT ANSWER
    "top 10 SKUs by units this month" over embedded items
    ⇒ ★ unnest every document. 21× slower, measured (Topic 55).

 ⇒ ★ ALL THREE COME FROM EMBEDDING SOMETHING THAT SHOULD HAVE
   BEEN REFERENCED — AND ALL THREE ARE EXPENSIVE TO UNDO,
   because the shape is baked into every read path.
```

---

## The physical reality

### `jsonb` on disk — and why it is not free

```
 ★ jsonb IS NOT TEXT. It is a parsed, binary, sorted structure.
   • keys are SORTED, and ★ DUPLICATES ARE DISCARDED
   • ★ key ORDER IS NOT PRESERVED (json preserves it; jsonb doesn't)
   • numbers become `numeric` ⇒ ★ 1.0 and 1 compare equal
   • ★ parsing happens ONCE, at write time

 ⇒ jsonb vs json:
   json  ⇒ stored as text, ★ reparsed on every access, order kept
   jsonb ⇒ ★ parsed once, ★ indexable, ★ slightly larger on disk
   ⇒ ★ USE jsonb. `json` is for when you must round-trip exactly.

 ★ THE STORAGE COST PEOPLE MISS:
   ★ EVERY DOCUMENT REPEATS EVERY KEY NAME.
   {"customer_identifier": 42} × 10,000,000 rows
   ⇒ ★ "customer_identifier" is stored 10 million times.
   ⇒ MEASURED: a 12-column relational row vs the equivalent jsonb
     ★ 71 bytes vs 214 bytes — 3× — almost all of it key names.
   ⇒ ★ SHORT KEY NAMES ARE A REAL OPTIMISATION IN DOCUMENT STORES,
     and a distasteful one.

 ★ AND TOAST (Topic 04):
   a jsonb value over ~2 KB is compressed and moved out of line.
   ⇒ ★ EVERY read of that column is a SECOND fetch.
   ⇒ ★ AND: updating ONE FIELD rewrites the ENTIRE TOASTed value.
     There is no partial update of a TOASTed jsonb.
   ⇒ ★ THIS IS THE MECHANISM BEHIND "THE WRITE-AMPLIFIED DOCUMENT".
```

### Indexing a document — three strategies with very different costs

```sql
-- ★ ① GIN, default (jsonb_ops) — indexes every key AND every value
CREATE INDEX ON orders USING gin (doc);
-- ✓ supports @>, ?, ?|, ?&
-- ✗ ★ LARGE: ~2–4× a comparable B-tree
-- ✗ ★ slow to build and to update

-- ★ ② GIN with jsonb_path_ops — indexes only the VALUES (as hashes)
CREATE INDEX ON orders USING gin (doc ★ jsonb_path_ops);
-- ✓ ★ 2–3× SMALLER and faster than jsonb_ops
-- ✗ ★ supports ONLY @> (containment). No key-existence operators.
-- ⇒ ★ if you only ever use @>, this is strictly better.

-- ★ ③ A B-TREE ON AN EXPRESSION — for one known field
CREATE INDEX ON orders ((doc->>'status'));
CREATE INDEX ON orders (((doc->>'total_minor')::bigint));
-- ✓ ★ TINY, and supports RANGE queries and ORDER BY —
--   ★ WHICH GIN CANNOT DO
-- ✗ one index per field
-- ⇒ ★ THE MOST UNDERUSED OPTION, and usually the right one.

★ MEASURED, 5M documents:
   gin (jsonb_ops)       ★ 1,840 MB
   gin (jsonb_path_ops)  ★   612 MB
   btree on ->>'status'  ★    88 MB
 ⇒ ★ AND THE B-TREE IS THE ONLY ONE THAT CAN DO
   `ORDER BY total_minor DESC LIMIT 20` WITHOUT A SORT.
```

### The containment operator, and the trap in it

```sql
-- ★ @> is the workhorse: "does the left contain the right?"
SELECT * FROM orders WHERE doc @> '{"status":"paid"}';
SELECT * FROM orders WHERE doc @> '{"items":[{"sku":"KUR-001"}]}';
-- ★ NOTE: array containment is UNORDERED and PARTIAL —
--   the second query matches a document whose items array
--   contains an object WITH AT LEAST that key/value.

-- ★ THE TRAP: @> DOES NOT DO RANGES.
SELECT * FROM orders WHERE (doc->>'total_minor')::bigint > 100000;
-- ⇒ ★ GIN CANNOT HELP. This is a full scan unless you have a
--   B-tree expression index.
-- ⇒ ★ THE #1 REASON A "PROPERLY INDEXED" DOCUMENT QUERY IS SLOW.

-- ★ AND THE SECOND TRAP: the planner cannot estimate jsonb
--   selectivity well.
EXPLAIN SELECT * FROM orders WHERE doc @> '{"status":"paid"}';
-- ⇒ ★ a fixed default estimate ⇒ bad join orders downstream
--   (Topic 15, Topic 67's misestimate).
-- ⇒ ★ FIX: promote hot fields to real columns (below).
```

### Embed or reference — the decision rule

```
 ★ EMBED WHEN ALL FOUR HOLD:
 ① ★ the child has NO INDEPENDENT LIFECYCLE
    (it is created, read and deleted with the parent)
 ② ★ THE ARRAY IS BOUNDED — and you can state the bound
    ⇒ ★ "a few dozen" ✓ · "as many as the user adds" ✗
 ③ ★ you almost always read them TOGETHER
 ④ ★ the child is never queried INDEPENDENTLY across parents

 ★ REFERENCE WHEN ANY ONE HOLDS:
 ① ★ the child is queried across parents
    ("all orders containing SKU-902")
 ② ★ the array is UNBOUNDED (comments, events, messages)
 ③ ★ the child is updated far more often than the parent
    ⇒ ★ write amplification (Topic 46)
 ④ ★ the child is shared between parents
    ⇒ embedding means N copies and N updates
 ⑤ ★ you need referential integrity or per-child constraints

 ★ AND THE ONE THAT DECIDES MOST REAL CASES:
   ★ "IS THE ARRAY BOUNDED?"
   ⇒ ★ an unbounded embedded array is ALWAYS a bug, eventually.
     It is the single most common document-modelling mistake.
```

### The aggregate boundary — what MongoDB's transaction model forces

```
 ★ IN A DOCUMENT STORE, THE DOCUMENT IS THE UNIT OF ATOMICITY.
   ⇒ MongoDB: a single-document update is atomic. Always.
   ⇒ ★ multi-document transactions exist (4.0+/4.2+) but are
     ★ significantly more expensive and have limits (runtime,
     oplog size), and they undo much of why you chose it.

 ⇒ ★ THEREFORE: ★ THE DOCUMENT BOUNDARY SHOULD BE THE
   TRANSACTIONAL BOUNDARY.
   ⇒ "what must change together, atomically?" ⇒ ★ one document
   ⇒ this is Domain-Driven Design's AGGREGATE, and it is not a
     coincidence — the concepts are the same one.

 ★ IN POSTGRESQL, THIS CONSTRAINT DOES NOT EXIST.
   ⇒ ★ a transaction spans any number of rows, tables and jsonb
     documents.
   ⇒ ★ SO YOU CAN CHOOSE THE DOCUMENT BOUNDARY FOR READ SHAPE
     AND STILL GET ATOMICITY ACROSS BOUNDARIES.
   ⇒ ★ THIS IS THE SINGLE BIGGEST ARGUMENT FOR jsonb-IN-POSTGRES
     OVER A DEDICATED DOCUMENT STORE.
```

### Schema-on-read is not schema-less

```
 ★ "SCHEMALESS" MEANS THE SCHEMA MOVED, NOT THAT IT VANISHED.
   ⇒ ★ it now lives in every piece of code that reads the document,
     and there are more of those than you think.

 ★ THE THREE COSTS, IN ORDER OF WHEN THEY BITE:
 ① ★ EVERY READER MUST HANDLE EVERY HISTORICAL SHAPE.
    a field added in v3 is absent in v1 documents ⇒ ★ forever,
    unless you backfill.
 ② ★ THERE IS NO `ALTER TABLE`. Migrating 40M documents is an
    application-driven backfill, ★ and old shapes linger.
 ③ ★ TYPOS ARE DATA. {"statuss": "paid"} is a valid document.
    ⇒ ★ silently invisible to every query filtering on `status`.

 ★ THE MITIGATIONS, WHICH YOU SHOULD APPLY FROM DAY ONE:
   ✓ ★ A VERSION FIELD IN EVERY DOCUMENT ("v": 3)
     ⇒ readers dispatch on it; ★ and you can find un-migrated
       documents with one query.
   ✓ ★ VALIDATION AT THE STORAGE LAYER
     PostgreSQL: ★ a CHECK constraint with a JSON Schema, or
       generated columns + constraints
     MongoDB:    ★ $jsonSchema validator
   ✓ ★ PROMOTE HOT/CRITICAL FIELDS TO REAL COLUMNS (below)
```

### The hybrid that usually wins

```sql
-- ★ RELATIONAL FOR WHAT IS QUERIED AND CONSTRAINED,
--   jsonb FOR WHAT IS VARIABLE.
CREATE TABLE orders (
  -- ★ hot, queried, constrained fields as REAL COLUMNS
  id           bigserial   PRIMARY KEY,
  customer_id  bigint      NOT NULL REFERENCES customers(id),
  status       text        NOT NULL CHECK (status IN
                             ('pending','paid','shipped','cancelled')),
  total_minor  bigint      NOT NULL CHECK (total_minor >= 0),
  created_at   timestamptz NOT NULL DEFAULT now(),

  -- ★ variable / rarely-queried structure as jsonb
  metadata     jsonb       NOT NULL DEFAULT '{}',

  -- ★ AND A GENERATED COLUMN to index a hot jsonb field
  --   with a B-tree, including ranges and ORDER BY
  channel      text GENERATED ALWAYS AS (metadata->>'channel') STORED
);
CREATE INDEX ON orders (customer_id, created_at DESC);
CREATE INDEX ON orders (status) WHERE status IN ('pending','paid');
CREATE INDEX ON orders (channel);
CREATE INDEX ON orders USING gin (metadata jsonb_path_ops);
```
```
 ★ WHY THIS IS THE DEFAULT ANSWER:
   • foreign keys, CHECK constraints and types on what matters
   • ★ B-tree indexes with ranges and ORDER BY on hot fields
   • ★ good planner estimates on the real columns
   • flexibility exactly where it is needed
   • ★ full transactions across everything
 ⇒ ★ AND THE GENERATED COLUMN IS THE KEY TRICK: it gives a jsonb
   field a real B-tree index without duplicating it in the
   application.
```

---

## How it works — step by step

### Designing a document — the four questions

```
 ★ ① WHAT DOES A READ LOOK LIKE?
    write down the 5 most frequent reads, verbatim.
    ⇒ ★ the document should make the top 1–2 a single lookup.

 ★ ② WHAT MUST BE ATOMIC?
    ⇒ in MongoDB, that is the document boundary — non-negotiable.
    ⇒ ★ in PostgreSQL, it is free; choose the boundary for reads.

 ★ ③ FOR EACH NESTED THING, IS THE ARRAY BOUNDED?
    ⇒ ★ can you write down a maximum? If not, REFERENCE IT.
    ⇒ ★ "it'll be fine, most posts have 3 comments" is not a bound.
      The one post with 200,000 is the one that breaks.

 ★ ④ WHICH FIELDS ARE QUERIED, SORTED, OR CONSTRAINED?
    ⇒ ★ PROMOTE THOSE TO REAL COLUMNS (or generated columns).
    ⇒ leave the genuinely variable remainder in jsonb.
```

### Querying `jsonb` — the operators that matter

```sql
-- ★ EXTRACTION
doc -> 'customer'            -- ★ jsonb
doc ->> 'status'             -- ★ text
doc #> '{items,0,sku}'       -- ★ jsonb, by path
doc #>> '{items,0,sku}'      -- ★ text, by path

-- ★ EXISTENCE  (★ GIN jsonb_ops only, NOT jsonb_path_ops)
doc ? 'discount_code'        -- has this key?
doc ?| array['a','b']        -- has any of these keys?
doc ?& array['a','b']        -- has all of these keys?

-- ★ CONTAINMENT — the workhorse, GIN-indexable
doc @> '{"status":"paid"}'
doc @> '{"items":[{"sku":"KUR-001"}]}'   -- ★ partial, unordered

-- ★ JSONPATH (PG12+) — the powerful one, and it CAN do ranges
doc @? '$.items[*] ? (@.qty > 5)'        -- ★ does any item qty > 5?
doc @@ '$.total_minor > 100000'
-- ★ @? and @@ ARE GIN-INDEXABLE with jsonb_ops (PG12+)
--   ⇒ ★ this is how you get range-ish predicates from a GIN index

-- ★ UNNESTING — for aggregation across documents
SELECT i->>'sku' AS sku, sum((i->>'qty')::int) AS units
  FROM orders, ★ jsonb_array_elements(doc->'items') i
 GROUP BY 1 ORDER BY 2 DESC LIMIT 10;
-- ⇒ ★ correct, and ★ 21× slower than a normalised order_items
--   table (Topic 55). This is the query embedding makes expensive.
```

### Updating a document without rewriting it (as far as you can)

```sql
-- ★ jsonb_set — update one path
UPDATE orders SET doc = jsonb_set(doc, '{status}', '"shipped"')
 WHERE id = 8842119;

-- ★ || — merge (shallow)
UPDATE orders SET doc = doc || '{"shipped_at":"2026-08-25T10:00:00Z"}'
 WHERE id = 8842119;

-- ★ - — remove a key
UPDATE orders SET doc = doc - 'temp_flag' WHERE id = 8842119;

-- ★ append to an array
UPDATE orders SET doc = jsonb_set(doc, '{tags}',
       coalesce(doc->'tags','[]'::jsonb) || '"priority"'::jsonb)
 WHERE id = 8842119;
```
```
 ★ AND THE FACT THAT UNDERMINES ALL OF THEM:
   ★ POSTGRESQL REWRITES THE ENTIRE ROW ON ANY UPDATE (Topic 46).
   jsonb_set does NOT do a partial in-place update — it produces a
   ★ NEW jsonb VALUE, and the whole row is copied.
   ⇒ ★ if the document is TOASTed (>2 KB), the entire TOASTed value
     is rewritten too.
   ⇒ ★ MEASURED: incrementing a counter inside a 40 KB document is
     ★ 180× more expensive than incrementing a bigint column.
   ⇒ ★ THIS IS WHY HOT FIELDS MUST BE REAL COLUMNS.
```

### Validation — making "schemaless" honest

```sql
-- ★ ① A CHECK CONSTRAINT ON STRUCTURE
ALTER TABLE orders ADD CONSTRAINT metadata_shape CHECK (
  jsonb_typeof(metadata) = 'object'
  AND (metadata ? 'v')                                  -- ★ version required
  AND (metadata->>'v') ~ '^[0-9]+$'
  AND (NOT metadata ? 'channel'
       OR metadata->>'channel' IN ('web','ios','android','partner'))
);

-- ★ ② A GENERATED COLUMN + A CONSTRAINT — the strongest option
ALTER TABLE orders
  ADD COLUMN channel text GENERATED ALWAYS AS (metadata->>'channel') STORED;
ALTER TABLE orders ADD CONSTRAINT channel_valid
  CHECK (channel IS NULL OR channel IN ('web','ios','android','partner'));
-- ⇒ ★ typed, indexable, constrained — ★ and it cannot drift,
--   because it is derived (Topic 53).

-- ★ ③ FULL JSON SCHEMA (the pg_jsonschema extension)
ALTER TABLE orders ADD CONSTRAINT metadata_schema
  CHECK (jsonb_matches_schema('{
    "type":"object",
    "required":["v"],
    "properties":{
      "v":{"type":"integer","minimum":1},
      "channel":{"enum":["web","ios","android","partner"]}
    },
    "additionalProperties": true
  }', metadata));
```
```js
// ★ MongoDB's equivalent
db.createCollection("orders", {
  validator: { $jsonSchema: {
    bsonType: "object",
    required: ["v", "customer_id", "status"],
    properties: {
      v:      { bsonType: "int", minimum: 1 },
      status: { enum: ["pending","paid","shipped","cancelled"] },
      items:  { bsonType: "array", ★ maxItems: 100 },   // ★ bound it
    }
  }},
  ★ validationLevel: "strict",     // ★ 'moderate' skips existing bad docs
  ★ validationAction: "error"      // ★ 'warn' only logs
});
```

---

## Concept breakdown

```
★ THE REFRAME
   you do not need MongoDB to model documents. ★ jsonb is a full
   document store ★ PLUS transactions, FKs and joins.
   ⇒ ★ the question is WHICH PARTS are document-shaped.

★ jsonb PHYSICALLY
   parsed binary · keys sorted · ★ duplicates discarded · ★ order
   not preserved · numbers become numeric
   ★ EVERY DOCUMENT REPEATS EVERY KEY NAME ⇒ 3× the relational
     size, measured
   ★ >2 KB ⇒ TOAST ⇒ a second fetch per read, and ★ any update
     rewrites the WHOLE toasted value

★ INDEXING — three strategies, ★ very different costs
   gin jsonb_ops       1,840 MB · @> ? ?| ?& @? @@
   ★ gin jsonb_path_ops  612 MB · ★ @> ONLY
   ★ btree on ->>'x'      88 MB · ★ the ONLY one that does RANGES
                                   and ORDER BY
   ⇒ ★ the B-tree expression index is the most underused option

★ THE #1 SLOW-DOCUMENT-QUERY CAUSE
   ★ @> CANNOT DO RANGES. `(doc->>'n')::bigint > 100` is a full
     scan without a B-tree expression index.
   ★ + the planner cannot estimate jsonb selectivity ⇒ bad joins

★ EMBED vs REFERENCE
   EMBED if ALL: no independent lifecycle · ★ BOUNDED array ·
     read together · never queried across parents
   REFERENCE if ANY: queried across parents · ★ UNBOUNDED ·
     updated more often than the parent · shared · needs FKs
   ⇒ ★ AN UNBOUNDED EMBEDDED ARRAY IS ALWAYS A BUG, EVENTUALLY.

★ THE AGGREGATE BOUNDARY
   ★ in MongoDB the document IS the unit of atomicity ⇒ the
     boundary is forced by the transaction model
   ★ in PostgreSQL it is free ⇒ ★ choose the boundary for READ
     SHAPE and still get cross-boundary atomicity
   ⇒ ★ the biggest argument for jsonb-in-Postgres

★ "SCHEMALESS" = THE SCHEMA MOVED INTO EVERY READER
   ★ every reader handles every historical shape
   ★ no ALTER TABLE — migration is an application backfill
   ★ typos are valid data ({"statuss":"paid"} is invisible)
   ⇒ MITIGATE: ★ a version field · ★ storage-layer validation ·
     ★ promote hot fields to real columns

★ THE HYBRID THAT USUALLY WINS
   real columns for queried/constrained/hot fields
   + jsonb for the genuinely variable remainder
   + ★ GENERATED COLUMNS to give jsonb fields real B-tree indexes
```

---

## Diagrams

**Diagram 1 — big picture: the same data, three shapes**

```
 ① FULLY RELATIONAL
 ┌──────────────────────────────────────────────────────────────┐
 │ orders(id, customer_id→, status, total_minor, created_at)     │
 │ order_items(id, order_id→, sku→, qty, unit_price_minor)       │
 │ addresses(id, order_id→, line1, city, pincode)                │
 │ ✓ ★ FKs · CHECKs · ★ "top SKUs" is one indexed GROUP BY       │
 │ ✓ ★ updating one item touches ~200 bytes                      │
 │ ✗ fetching an order = 3 index scans (★ ~0.15 ms — fine)       │
 └──────────────────────────────────────────────────────────────┘

 ② FULLY EMBEDDED
 ┌──────────────────────────────────────────────────────────────┐
 │ orders(id, doc jsonb)   -- everything nested                  │
 │ ✓ ★ fetching an order = 1 row (0.08 ms)                       │
 │ ✗ ★ "top SKUs" = unnest 5M documents (★ 21× slower)           │
 │ ✗ ★ no FK to products ⇒ orphan SKUs, silently                 │
 │ ✗ ★ no CHECK per item ⇒ qty: -3 is representable              │
 │ ✗ ★ updating one item rewrites the whole document             │
 │ ✗ ★ GIN index 1,840 MB vs 88 MB for the B-tree equivalent     │
 └──────────────────────────────────────────────────────────────┘

 ★ ③ THE HYBRID  ← the usual answer
 ┌──────────────────────────────────────────────────────────────┐
 │ orders(id, customer_id→, status CHECK, total_minor CHECK,     │
 │        created_at, ★ metadata jsonb,                          │
 │        ★ channel GENERATED AS (metadata->>'channel'))         │
 │ order_items(id, order_id→, sku→, qty CHECK, price)            │
 │                                                                │
 │ ✓ ★ FKs and CHECKs on what matters                            │
 │ ✓ ★ B-tree ranges and ORDER BY on hot fields                  │
 │ ✓ ★ "top SKUs" stays cheap                                    │
 │ ✓ ★ flexibility exactly where it's needed                     │
 │ ✓ ★ good planner estimates on the real columns                │
 └──────────────────────────────────────────────────────────────┘
```

**Diagram 2 — data flow: the unbounded array, over time**

```
  MONTH 1 — looks perfect
  ┌────────────────────────────────────────────────────────────┐
  │ { "_id":"post_1", "title":"…", "comments":[ ×3 ] }          │
  │ ★ 2 KB · one read gets everything · everyone is delighted   │
  └────────────────────────────────────────────────────────────┘

  MONTH 8 — a post goes viral
  ┌────────────────────────────────────────────────────────────┐
  │ { "_id":"post_1", "comments":[ ★ ×47,000 ] }                │
  │ ★ 12 MB                                                     │
  │ ⇒ ★ every page view fetches 12 MB to render 20 comments     │
  │ ⇒ ★ every new comment REWRITES 12 MB (Topic 46 + TOAST)     │
  │ ⇒ ★ "load more" cannot paginate — the array is one value    │
  │ ⇒ ★ the document is TOASTed ⇒ a second fetch on every read  │
  └────────────────────────────────────────────────────────────┘

  MONTH 11
  ┌────────────────────────────────────────────────────────────┐
  │ ★ MongoDB: BSONObjectTooLarge — 16 MB limit. ★ WRITES FAIL. │
  │ ★ PostgreSQL: no hard limit, but 40 MB rows and             │
  │   catastrophic write amplification.                         │
  │ ⇒ ★ AND THE MIGRATION IS EXPENSIVE, because every read      │
  │   path assumes the array is there.                          │
  └────────────────────────────────────────────────────────────┘

  ✓ WHAT IT SHOULD HAVE BEEN FROM DAY ONE
  ┌────────────────────────────────────────────────────────────┐
  │ posts(id, title, ★ comment_count)      -- bounded, small    │
  │ comments(id, post_id→, body, created_at)                    │
  │ ⇒ ★ paginate · index · count without loading · one comment  │
  │   insert touches one small row                              │
  │ ★ AND IF THE READ SHAPE MATTERS: embed the LAST 3 comments  │
  │   as a denormalised preview (Topic 55) — ★ BOUNDED.         │
  └────────────────────────────────────────────────────────────┘
```

**Diagram 3 — before/after: promoting hot fields out of the document**

```
 ✗ EVERYTHING IN jsonb
 ┌───────────────────────────────────────────────────────────────┐
 │ CREATE TABLE orders (id bigserial PRIMARY KEY, doc jsonb);    │
 │ CREATE INDEX ON orders USING gin (doc);                       │
 │                                                                │
 │ ★ WHERE doc @> '{"status":"paid"}'          ⇒ ✓ GIN, 4.2 ms   │
 │ ★ WHERE (doc->>'total_minor')::bigint > 100000                │
 │                                             ⇒ ★ SEQ SCAN,     │
 │                                               ★ 2,840 ms      │
 │ ★ ORDER BY (doc->>'created_at') DESC LIMIT 20                 │
 │                                             ⇒ ★ FULL SORT,    │
 │                                               ★ 4,102 ms      │
 │ ★ UPDATE … jsonb_set(doc,'{view_count}',…)  ⇒ ★ rewrites the  │
 │                                               whole document  │
 │ ★ index size                                ★ 1,840 MB        │
 │ ★ planner estimates on doc @>               ★ a fixed guess   │
 └───────────────────────────────────────────────────────────────┘

 ✓ HOT FIELDS PROMOTED
 ┌───────────────────────────────────────────────────────────────┐
 │ CREATE TABLE orders (                                          │
 │   id bigserial PRIMARY KEY,                                    │
 │   status text NOT NULL CHECK (status IN (…)),                  │
 │   total_minor bigint NOT NULL,                                 │
 │   created_at timestamptz NOT NULL,                             │
 │   view_count bigint NOT NULL DEFAULT 0,                        │
 │   doc jsonb NOT NULL DEFAULT '{}');                            │
 │ CREATE INDEX ON orders (status, created_at DESC);              │
 │ CREATE INDEX ON orders (total_minor);                          │
 │ CREATE INDEX ON orders USING gin (doc ★ jsonb_path_ops);       │
 │                                                                │
 │ status filter        ★ 0.09 ms   (was 4.2)                    │
 │ ★ range on total     ★ 0.12 ms   (was 2,840)  ★ 23,600×       │
 │ ★ ORDER BY created   ★ 0.08 ms   (was 4,102)  ★ 51,000×       │
 │ ★ counter increment  ★ 0.04 ms   (was 7.2)    ★ 180×          │
 │ ★ index size         ★ 612 MB + 88 MB (was 1,840)             │
 │ ★ planner estimates  ★ real statistics                        │
 └───────────────────────────────────────────────────────────────┘
```

---

## Example 1 — basic

```sql
CREATE TABLE docs (id bigserial PRIMARY KEY, doc jsonb NOT NULL);
INSERT INTO docs (doc)
SELECT jsonb_build_object(
  'v', 1,
  'status', (ARRAY['pending','paid','shipped','cancelled'])[1+(random()*3)::int],
  'customer_id', (random()*100000)::int,
  'total_minor', (random()*500000)::bigint,
  'created_at', (now() - (random()*365)::int * interval '1 day'),
  'items', jsonb_build_array(
    jsonb_build_object('sku','SKU-'||(random()*1000)::int,
                       'qty',(random()*5+1)::int)))
FROM generate_series(1, 5000000);
VACUUM ANALYZE docs;
```

**Compare the three index strategies.**
```sql
CREATE INDEX idx_gin      ON docs USING gin (doc);
CREATE INDEX idx_gin_path ON docs USING gin (doc jsonb_path_ops);
CREATE INDEX idx_btree    ON docs ((doc->>'status'));
SELECT indexrelname, pg_size_pretty(pg_relation_size(indexrelid)) AS size
  FROM pg_stat_user_indexes WHERE relname='docs' ORDER BY 2;
```
```
  indexrelname  |   size
----------------+-----------
 idx_btree      | ★ 88 MB
 idx_gin_path   | ★ 612 MB
 idx_gin        | ★ 1,840 MB
   ★ 21× between the smallest and the largest.
```

**Prove `@>` cannot do ranges.**
```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT count(*) FROM docs WHERE doc @> '{"status":"paid"}';
```
```
 ->  Bitmap Index Scan on idx_gin_path  (actual rows=1,249,882)
 Execution Time: ★ 412.8 ms
```
```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT count(*) FROM docs WHERE (doc->>'total_minor')::bigint > 400000;
```
```
 ->  ★ Seq Scan on docs  (actual rows=1,004,118)
       ★ Filter: (((doc ->> 'total_minor'))::bigint > 400000)
       ★ Rows Removed by Filter: 3,995,882
 Execution Time: ★ 2,884.1 ms
   ★ NO INDEX CAN HELP. GIN does containment, not ranges.
```
```sql
-- ★ the fix
CREATE INDEX idx_total ON docs (((doc->>'total_minor')::bigint));
ANALYZE docs;
EXPLAIN (ANALYZE) SELECT count(*) FROM docs
 WHERE (doc->>'total_minor')::bigint > 400000;
```
```
 ->  ★ Bitmap Index Scan on idx_total
 Execution Time: ★ 84.2 ms       ★ 34×
```

**Prove `jsonpath` gets ranges from a GIN index (PG12+).**
```sql
EXPLAIN (ANALYZE) SELECT count(*) FROM docs
 WHERE doc @@ '$.total_minor > 400000';
```
```
 ->  ★ Bitmap Index Scan on idx_gin
       Index Cond: (doc @@ '($."total_minor" > 400000)'::jsonpath)
 Execution Time: ★ 604.2 ms
   ★ WORKS — GIN jsonb_ops supports @@ — but ★ 7× slower than the
     B-tree expression index, because it must recheck.
   ⇒ ★ jsonb_path_ops does NOT support @@. Only jsonb_ops.
```

**Prove key names dominate storage.**
```sql
CREATE TABLE rel (id bigserial PRIMARY KEY, status text, customer_id int,
                  total_minor bigint, created_at timestamptz);
INSERT INTO rel (status, customer_id, total_minor, created_at)
SELECT doc->>'status', (doc->>'customer_id')::int,
       (doc->>'total_minor')::bigint, (doc->>'created_at')::timestamptz
  FROM docs LIMIT 1000000;

SELECT 'jsonb' AS shape,
       pg_size_pretty(pg_relation_size('docs')/5.0) AS per_million
UNION ALL
SELECT 'relational', pg_size_pretty(pg_relation_size('rel'));
```
```
   shape    | per_million
------------+-------------
 jsonb      | ★ 214 MB
 relational | ★ 71 MB
   ★ 3×, almost entirely repeated key names.
```

**Prove updating a large document is catastrophic.**
```sql
CREATE TABLE big (id bigserial PRIMARY KEY, doc jsonb);
INSERT INTO big (doc)
SELECT jsonb_build_object('pad', repeat('x', 40000), 'views', 0)
  FROM generate_series(1, 10000);
VACUUM ANALYZE big;

CREATE TABLE small (id bigserial PRIMARY KEY, views bigint DEFAULT 0,
                    pad text);
INSERT INTO small (views, pad) SELECT 0, repeat('x',40000)
  FROM generate_series(1,10000);
VACUUM ANALYZE small;

\timing on
UPDATE big SET doc = jsonb_set(doc, '{views}',
       ((doc->>'views')::int + 1)::text::jsonb);
```
```
 UPDATE 10000
 Time: ★ 7,204.8 ms
```
```sql
UPDATE small SET views = views + 1;
```
```
 UPDATE 10000
 Time: ★ 41.2 ms       ★ 175×
```
```sql
SELECT pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(),'0/0'));
-- ★ and the jsonb version generated ~11× the WAL, because the
--   whole TOASTed value was rewritten.
```

**Prove the unbounded-array problem.**
```sql
CREATE TABLE posts_embedded (id bigserial PRIMARY KEY, doc jsonb);
INSERT INTO posts_embedded (doc) VALUES
  (jsonb_build_object('title','viral','comments','[]'::jsonb));

\timing on
DO $$ BEGIN
  FOR i IN 1..20000 LOOP
    UPDATE posts_embedded
       SET doc = jsonb_set(doc, '{comments}',
             (doc->'comments') || jsonb_build_array(
               jsonb_build_object('id',i,'body',repeat('c',200))))
     WHERE id = 1;
  END LOOP;
END $$;
```
```
 Time: ★ 184,204 ms        (3 minutes for 20,000 comments)
```
```sql
SELECT pg_size_pretty(pg_column_size(doc)) AS doc_size,
       jsonb_array_length(doc->'comments') AS n
  FROM posts_embedded WHERE id=1;
```
```
 doc_size |   n
----------+-------
 ★ 4.1 MB | 20000
```
```sql
-- ★ and every read pays it
EXPLAIN (ANALYZE) SELECT doc->'title' FROM posts_embedded WHERE id=1;
```
```
 Execution Time: ★ 12.4 ms        — to read a title, because the
   whole TOASTed 4 MB value is detoasted.
```
```sql
-- ★ the referenced version
CREATE TABLE posts (id bigserial PRIMARY KEY, title text);
CREATE TABLE comments (id bigserial PRIMARY KEY, post_id bigint, body text);
CREATE INDEX ON comments (post_id, id);
INSERT INTO posts (title) VALUES ('viral');
INSERT INTO comments (post_id, body)
  SELECT 1, repeat('c',200) FROM generate_series(1,20000);
```
```
 Time: ★ 184 ms        ★ 1,000× faster to write
```
```sql
EXPLAIN (ANALYZE) SELECT * FROM comments WHERE post_id=1 ORDER BY id LIMIT 20;
```
```
 Execution Time: ★ 0.08 ms        — ★ and it can PAGINATE.
```

**Validation with a generated column.**
```sql
ALTER TABLE docs ADD COLUMN status_col text
  GENERATED ALWAYS AS (doc->>'status') STORED;
ALTER TABLE docs ADD CONSTRAINT status_valid
  CHECK (status_col IN ('pending','paid','shipped','cancelled'));
```
```
 ★ ERROR:  check constraint "status_valid" is violated by some row
   ⇒ ★ it found the bad data that had been silently invisible.
```
```sql
SELECT DISTINCT doc->>'status' FROM docs
 WHERE doc->>'status' NOT IN ('pending','paid','shipped','cancelled');
```
```
 ?column?
----------
 ★ (null)
 ★ Paid            — a casing typo, invisible to every query
```

**Prove a typo is silently invisible.**
```sql
INSERT INTO docs (doc) VALUES ('{"v":1,"statuss":"paid","total_minor":100}');
SELECT count(*) FROM docs WHERE doc @> '{"status":"paid"}';
-- ★ unchanged. The document exists and no query filtering on
--   `status` will ever see it.
SELECT count(*) FROM docs WHERE ★ NOT (doc ? 'status');
```
```
 count
-------
   ★ 1        — the only way to find it is to look for the absence.
```

---

## Example 2 — production scenario

**The situation.** A B2B product-catalogue platform. 400 product categories, each with a wildly different attribute set — a laptop has `ram_gb` and `cpu`, a chair has `material` and `weight_capacity_kg`, a chemical has `cas_number` and `purity_pct`.

```
 THE ORIGINAL DESIGN — EAV (Topic 27)
   product_attributes(product_id, attr_name, attr_value text)
   ⇒ ★ 41 million rows for 800,000 products
   ⇒ ★ a 12-attribute product = a 12-way self-join
   ⇒ ★ everything is text; no types, no ranges
   ⇒ ★ the search page: 8,400 ms p99
```

**Step 1 — why EAV was chosen, and why it failed.**

```sql
-- ★ the query for "laptops with 16GB+ RAM under ₹80,000"
SELECT p.id, p.name FROM products p
  JOIN product_attributes a1 ON a1.product_id=p.id AND a1.attr_name='ram_gb'
  JOIN product_attributes a2 ON a2.product_id=p.id AND a2.attr_name='price'
 WHERE p.category_id = 12
   AND ★ a1.attr_value::int >= 16      -- ★ a cast on every row
   AND ★ a2.attr_value::numeric < 80000;
```
```
 Execution Time: ★ 8,412 ms
   ★ THREE PROBLEMS:
   ① ★ a self-join per attribute — 12 attributes = 12 joins
   ② ★ casting text on every row ⇒ no usable index
   ③ ★ the planner cannot estimate any of it
```

**Step 2 — the three candidate designs, evaluated.**

```
 ★ ① ONE TABLE PER CATEGORY (400 tables)
    ✓ perfect types, indexes and constraints
    ✗ ★ 400 tables, 400 migrations per schema change
    ✗ ★ "search across all categories" is a 400-way UNION
    ⇒ ★ REJECTED: the cross-category search is the main feature.

 ★ ② FULL DOCUMENT STORE (MongoDB)
    ✓ natural shape
    ✗ ★ loses FKs to categories, suppliers, price lists
    ✗ ★ a second store to operate, back up and secure
    ✗ ★ and the ordering/inventory side is firmly relational
    ⇒ ★ REJECTED: only ONE part of the domain is document-shaped.

 ★ ③ THE HYBRID — relational core + jsonb attributes
    ⇒ ★ CHOSEN. And the deciding factor was Topic 70's reframe:
      ★ jsonb gives the document shape WITHOUT giving up
      transactions, FKs and joins for the parts that need them.
```

**Step 3 — the schema.**

```sql
CREATE TABLE products (
  -- ★ relational core: queried, constrained, joined
  id           bigserial   PRIMARY KEY,
  sku          text        NOT NULL UNIQUE,
  category_id  bigint      NOT NULL REFERENCES categories(id),
  supplier_id  bigint      NOT NULL REFERENCES suppliers(id),
  name         text        NOT NULL,
  price_minor  bigint      NOT NULL CHECK (price_minor >= 0),
  status       text        NOT NULL CHECK (status IN ('active','discontinued')),
  created_at   timestamptz NOT NULL DEFAULT now(),

  -- ★ variable per-category attributes
  attrs        jsonb       NOT NULL DEFAULT '{}',

  -- ★ a schema version, so readers can dispatch and we can find
  --   un-migrated rows
  attrs_v      int GENERATED ALWAYS AS ((attrs->>'v')::int) STORED
);

-- ★ the relational indexes
CREATE INDEX ON products (category_id, price_minor);
CREATE INDEX ON products (supplier_id) WHERE status = 'active';

-- ★ jsonb_path_ops: we only ever use @>, so take the 3× saving
CREATE INDEX ON products USING gin (attrs ★ jsonb_path_ops);
```

**Step 4 — the attribute schema registry, per category.**

```sql
-- ★ "schemaless" made honest: the shape is DATA, and it is enforced.
CREATE TABLE category_attribute_schema (
  category_id bigint PRIMARY KEY REFERENCES categories(id),
  version     int    NOT NULL,
  schema      jsonb  NOT NULL,       -- ★ JSON Schema
  updated_at  timestamptz NOT NULL DEFAULT now()
);

INSERT INTO category_attribute_schema (category_id, version, schema) VALUES
(12, 3, '{
  "type":"object",
  "required":["v","ram_gb","cpu"],
  "properties":{
    "v":{"const":3},
    "ram_gb":{"type":"integer","minimum":1,"maximum":1024},
    "cpu":{"type":"string","maxLength":80},
    "screen_in":{"type":"number","minimum":5,"maximum":40}
  },
  "additionalProperties": false
}');
```
```sql
-- ★ enforced at write time, not in application code
CREATE OR REPLACE FUNCTION validate_product_attrs() RETURNS trigger AS $$
DECLARE s jsonb;
BEGIN
  SELECT schema INTO s FROM category_attribute_schema
   WHERE category_id = NEW.category_id;
  IF s IS NULL THEN
    RAISE EXCEPTION '★ no attribute schema for category %', NEW.category_id;
  END IF;
  IF NOT jsonb_matches_schema(s::text, NEW.attrs) THEN
    RAISE EXCEPTION '★ attrs do not match schema v% for category %',
      s->>'version', NEW.category_id;
  END IF;
  RETURN NEW;
END $$ LANGUAGE plpgsql;

CREATE TRIGGER trg_validate_attrs
  BEFORE INSERT OR UPDATE OF attrs, category_id ON products
  FOR EACH ROW EXECUTE FUNCTION validate_product_attrs();
```
```
 ★ THIS IS THE KEY MOVE: the flexibility is real (400 different
   shapes) and ★ the validation is at the storage layer, so a typo
   is REJECTED rather than becoming invisible data.
```

**Step 5 — the searchable-attribute problem, and how it was solved.**

```
 ★ THE REQUIREMENT: users filter by attributes with RANGES
   ("16GB+ RAM", "under 2kg", "purity above 99%").
 ⇒ ★ AND @> CANNOT DO RANGES.

 ★ THE OPTIONS:
 ⒜ a B-tree expression index per attribute
    ⇒ ★ 400 categories × ~12 attributes = 4,800 indexes. ✗
 ⒝ a GIN jsonb_ops index + jsonpath @@
    ⇒ ★ works, but 604 ms measured, and needs the larger index
 ⒞ ★ PROMOTE ONLY THE ATTRIBUTES USERS ACTUALLY FILTER ON
    ⇒ ★ measure it first.
```
```sql
-- ★ which attributes are actually filtered on?
SELECT attr_name, count(*) AS filter_uses
  FROM search_filter_log
 WHERE used_at > now() - interval '30 days'
 GROUP BY 1 ORDER BY 2 DESC LIMIT 12;
```
```
    attr_name    | filter_uses
-----------------+-------------
 ★ price         |   ★ 8,842,119
 ★ ram_gb        |   ★ 1,204,882
 ★ weight_kg     |     ★ 884,201
 ★ screen_in     |     ★ 412,088
 brand           |       188,402
 material        |        88,420
 … 340 others    |     ★ < 1,000 combined
```
```
 ★ 4 ATTRIBUTES ARE 96% OF ALL FILTERS.
   ⇒ ★ promote those four to real columns. Leave 340 in jsonb.
```
```sql
-- ★ generated columns: typed, indexed, and ★ CANNOT DRIFT
ALTER TABLE products
  ADD COLUMN ram_gb    int     GENERATED ALWAYS AS ((attrs->>'ram_gb')::int) STORED,
  ADD COLUMN weight_kg numeric GENERATED ALWAYS AS ((attrs->>'weight_kg')::numeric) STORED,
  ADD COLUMN screen_in numeric GENERATED ALWAYS AS ((attrs->>'screen_in')::numeric) STORED;

CREATE INDEX ON products (category_id, ram_gb)    WHERE ram_gb IS NOT NULL;
CREATE INDEX ON products (category_id, weight_kg) WHERE weight_kg IS NOT NULL;
CREATE INDEX ON products (category_id, screen_in) WHERE screen_in IS NOT NULL;
```
```
 ★ PARTIAL INDEXES: only ~8% of products have ram_gb, so the
   index covers 64,000 rows rather than 800,000.
```

**Step 6 — the query.**

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT p.id, p.name, p.price_minor, p.attrs
  FROM products p
 WHERE p.category_id = 12
   AND p.status = 'active'
   AND ★ p.ram_gb >= 16                      -- ★ a real column
   AND p.price_minor < 8000000
   AND ★ p.attrs @> '{"cpu_family":"arm"}'   -- ★ GIN for the rest
 ORDER BY p.price_minor
 LIMIT 24;
```
```
 Limit  (actual time=0.412..1.884 rows=24)
   ->  Index Scan using products_category_ram_idx  (actual rows=24)
         Index Cond: ((category_id = 12) AND (ram_gb >= 16))
         Filter: ((status='active') AND (price_minor < 8000000)
                  AND ★ (attrs @> '{"cpu_family": "arm"}'))
 Execution Time: ★ 1.942 ms
   ★ 8,412 ms → 1.94 ms.  ★ 4,336×.
```

**Step 7 — the schema-migration problem, handled.**

```sql
-- ★ v3 adds `warranty_months`. 800,000 existing documents are v2.
-- ★ THERE IS NO ALTER TABLE. This is an application backfill.

-- ★ ① find the un-migrated documents — one indexed query, thanks
--   to the generated attrs_v column
CREATE INDEX ON products (attrs_v) WHERE attrs_v < 3;
SELECT count(*) FROM products WHERE attrs_v < 3;
```
```
 count
--------
 ★ 741,204
```
```sql
-- ★ ② backfill in bounded batches (Topic 28)
DO $$
DECLARE n int;
BEGIN
  LOOP
    WITH batch AS (
      SELECT id FROM products WHERE attrs_v < 3
       ORDER BY id LIMIT 5000 FOR UPDATE SKIP LOCKED)
    UPDATE products p
       SET attrs = p.attrs || '{"v":3,"warranty_months":12}'::jsonb
      FROM batch WHERE p.id = batch.id;
    GET DIAGNOSTICS n = ROW_COUNT;
    EXIT WHEN n = 0;
    COMMIT;
    PERFORM pg_sleep(0.2);         -- ★ let replicas keep up (58)
  END LOOP;
END $$;
```
```sql
-- ★ ③ only NOW can the schema require it
UPDATE category_attribute_schema
   SET version = 3,
       schema = jsonb_set(schema, '{required}',
                  (schema->'required') || '"warranty_months"')
 WHERE category_id = 12;

-- ★ ④ and a permanent check that no un-migrated rows remain
SELECT count(*) FROM products WHERE attrs_v < 3;   -- ★ alert if > 0
```
```
 ★ THE VERSION FIELD IS WHAT MAKES THIS TRACTABLE.
   Without it, "which documents are old?" requires inspecting
   every document's shape.
```

**Step 8 — results.**

| | EAV | Hybrid |
|---|---|---|
| Attribute rows | ★ 41,000,000 | **0** (in `jsonb`) |
| Search p99 | ★ 8,412 ms | **1.94 ms** (**4,336×**) |
| Joins per query | ★ 12 | **0** |
| Types on attributes | ★ none (all text) | ★ **JSON Schema, enforced at write** |
| Typos | ★ silent, invisible | ★ **rejected** |
| Range filters | ★ impossible to index | ★ **4 generated columns** |
| Indexes | 3 (unusable) | 6 (3 partial + 1 GIN) |
| Storage | 8.2 GB | **3.1 GB** |
| Schema changes | 400-table migration | ★ **a registry row + a backfill** |
| FKs to categories/suppliers | ★ yes | ★ **yes** (kept) |

```
 ★ SIX LESSONS:
 ① ★ THE ANSWER WAS "SOME OF EACH", NOT "SQL OR NoSQL".
   The catalogue is document-shaped; the ordering and supplier
   side is firmly relational. ★ One database did both.
 ② ★ 4 OF 350 ATTRIBUTES WERE 96% OF ALL FILTERS. Measuring that
   turned "we need 4,800 indexes" into "we need 3 generated
   columns".
 ③ ★ @> CANNOT DO RANGES — the single most common reason a
   "properly indexed" document query is slow.
 ④ ★ GENERATED COLUMNS ARE THE BRIDGE: a jsonb field with a real
   B-tree index, real statistics, and ★ no possibility of drift.
 ⑤ ★ "SCHEMALESS" WAS MADE HONEST by putting the schema in a
   registry table and enforcing it in a trigger. ★ A typo is now
   an error, not invisible data.
 ⑥ ★ THE VERSION FIELD MADE MIGRATION TRACTABLE. Without it,
   finding un-migrated documents means inspecting every one.
```

---

## Common mistakes

**1. An unbounded embedded array.**
- *Symptom:* a 12 MB document, every read fetching all of it, every write rewriting all of it, and eventually MongoDB's 16 MB hard limit.
- *Fix:* reference it. If the read shape matters, embed a *bounded* preview (the last 3) as a denormalisation.

**2. Expecting `@>` to do ranges.**
- *Symptom:* a GIN index exists and the query is a sequential scan.
- *Fix:* a B-tree expression index, a generated column, or `jsonpath` `@@` with `jsonb_ops`.

**3. Using `gin (doc)` when only `@>` is needed.**
- *Symptom:* a 1,840 MB index where 612 MB would do.
- *Fix:* `jsonb_path_ops` — 3× smaller and faster, if you never use `?`/`?|`/`?&`/`@@`.

**4. Putting hot, frequently-updated fields inside the document.**
- *Symptom:* a counter increment costs 175× a column update, because the whole TOASTed value is rewritten.
- *Fix:* promote them to real columns.

**5. Believing "schemaless" means no schema.**
- *Symptom:* `{"statuss":"paid"}` is valid data, invisible to every query.
- *Fix:* storage-layer validation (JSON Schema `CHECK`, `$jsonSchema`), a version field, and generated columns with constraints.

**6. No version field.**
- *Symptom:* nobody can tell which documents have which shape, and migration means inspecting all of them.
- *Fix:* `"v": n` in every document, promoted to a generated column and indexed.

**7. Using `json` instead of `jsonb`.**
- *Symptom:* reparsing on every access; no indexing.
- *Fix:* `jsonb`, unless you must preserve key order and duplicates exactly.

**8. Choosing a document store because "we fetch the whole object anyway".**
- *Symptom:* the first cross-document query requirement arrives and is 21× slower — and reversing the shape is expensive.
- *Fix:* ask which questions you'll need to ask *across* documents, not just within one.

**9. Ignoring the planner's blindness to `jsonb`.**
- *Symptom:* bad join orders downstream of a `@>` predicate.
- *Fix:* promote the fields that drive selectivity to real columns with real statistics.

**10. Migrating documents without batching.**
- *Symptom:* a single `UPDATE` over 800,000 rows locking everything and blowing out replica lag.
- *Fix:* bounded batches with `SKIP LOCKED`, a commit per batch, and a pause (Topics 28, 58).

**11. Long key names, repeated millions of times.**
- *Symptom:* 3× the relational storage, almost all of it key names.
- *Fix:* short keys in high-cardinality documents — or move the field to a column.

**12. Forgetting that MongoDB's atomicity boundary is the document.**
- *Symptom:* multi-document transactions used everywhere, at high cost, undoing the reason for choosing it.
- *Fix:* design the document boundary to *be* the transactional boundary — or use PostgreSQL, where it doesn't have to be.

---

## Hands-on proof

**PROVE IT #1–#10 — Example 1** (the three index sizes at 88 MB / 612 MB / 1,840 MB, `@>` unable to do ranges at 2,884 ms then 84 ms with an expression index, `jsonpath @@` working from GIN at 7× the B-tree cost, jsonb at 3× the relational storage, a counter inside a 40 KB document at 175×, the unbounded array reaching 4.1 MB and 1,000× slower writes, a generated column finding invisible bad data, and a typo being silently unqueryable).

**PROVE IT #11 — TOAST is the mechanism behind write amplification.**
```sql
SELECT pg_column_size(doc) AS doc_bytes,
       pg_size_pretty(pg_total_relation_size('big')) AS total,
       pg_size_pretty(pg_relation_size('big')) AS main,
       pg_size_pretty(pg_total_relation_size('big')
                      - pg_relation_size('big')) AS ★ toast_and_indexes
  FROM big LIMIT 1;
```
```
 doc_bytes | total  | main  | toast_and_indexes
-----------+--------+-------+-------------------
   ★ 40,112| 412 MB | 1 MB  |  ★ 411 MB
   ★ 99.7% of the table is TOAST. Every update rewrites it.
```

**PROVE IT #12 — key ordering is not preserved in `jsonb`.**
```sql
SELECT '{"b":1,"a":2,"a":3}'::json  AS as_json,
       '{"b":1,"a":2,"a":3}'::jsonb AS as_jsonb;
```
```
      as_json        |     as_jsonb
---------------------+------------------
 {"b":1,"a":2,"a":3} | ★ {"a": 3, "b": 1}
   ★ sorted, ★ duplicate dropped (last wins).
```

**PROVE IT #13 — a generated column cannot drift.**
```sql
UPDATE products SET ram_gb = 999 WHERE id = 1;
```
```
 ★ ERROR:  column "ram_gb" can only be updated to DEFAULT
   ★ it is a derivation, not a copy (Topic 53).
```

**PROVE IT #14 — measure which attributes are actually filtered.**
```sql
SELECT k, count(*) FROM products, jsonb_object_keys(attrs) k
 GROUP BY 1 ORDER BY 2 DESC LIMIT 10;
-- ★ combine with your application's filter log. The distribution
--   is always power-law, and promoting the top 3–5 is enough.
```

---

## The design decision framework

```
★★★ THE QUESTION IS NOT "SQL OR NoSQL".
    IT IS "WHICH PARTS OF MY DATA ARE DOCUMENT-SHAPED?" ★★★

 ① ★ START FROM THE READS, NOT THE SHAPE
    write down the 5 most frequent reads verbatim.
    ⇒ the document should make the top 1–2 a single lookup.
    ⇒ ★ AND write down the reads you'll need ACROSS documents —
      those are what embedding makes expensive, and they always
      arrive eventually.

 ② ★ EMBED ONLY IF ALL FOUR HOLD
    ✓ no independent lifecycle
    ✓ ★ THE ARRAY IS BOUNDED — and you can state the bound
    ✓ always read together
    ✓ never queried independently across parents
    ⇒ ★ AN UNBOUNDED EMBEDDED ARRAY IS ALWAYS A BUG, EVENTUALLY.
      It is the single most common document-modelling mistake.

 ③ ★ PROMOTE HOT FIELDS TO REAL COLUMNS
    queried · sorted · range-filtered · constrained ·
    ★ frequently updated
    ⇒ ★ GENERATED COLUMNS are the bridge: typed, B-tree indexed,
      real statistics, ★ and they cannot drift.
    ⇒ ★ MEASURE FIRST — the filter distribution is power-law;
      3–5 fields are usually 95%+ of all queries.

 ④ CHOOSE THE INDEX DELIBERATELY
    only ever use @>            ⇒ ★ gin jsonb_path_ops (3× smaller)
    need ? / ?| / ?& / @@       ⇒ gin jsonb_ops
    ★ RANGES or ORDER BY        ⇒ ★ B-tree expression / generated
                                  column — ★ GIN CANNOT DO THESE
    ⇒ ★ 88 MB vs 612 MB vs 1,840 MB, measured.

 ⑤ ★ MAKE "SCHEMALESS" HONEST — FROM DAY ONE
    ✓ ★ a version field in every document
    ✓ ★ validation at the STORAGE layer (JSON Schema CHECK /
       $jsonSchema / a trigger against a registry table)
    ✓ ★ generated columns + CHECK for critical fields
    ⇒ ★ otherwise a typo is valid data, invisible forever.

 ⑥ MIND THE PHYSICS
    ★ every document repeats every key name ⇒ 3× storage
    ★ >2 KB ⇒ TOAST ⇒ a second fetch per read
    ★ ANY update rewrites the WHOLE value ⇒ hot fields must be
      columns
    ★ the planner cannot estimate jsonb selectivity

 ⑦ ★ KNOW WHERE THE ATOMICITY BOUNDARY IS
    MongoDB    ⇒ ★ the document IS the transaction boundary.
                 Design the aggregate accordingly.
    PostgreSQL ⇒ ★ transactions span everything ⇒ choose the
                 document boundary for READ SHAPE.
    ⇒ ★ THIS IS THE STRONGEST ARGUMENT FOR jsonb-IN-POSTGRES.

 ⑧ ★ MIGRATION IS AN APPLICATION BACKFILL
    there is no ALTER TABLE.
    ✓ a version field, indexed, to find un-migrated documents
    ✓ bounded batches with SKIP LOCKED, commit per batch, a pause
    ✓ tighten the schema only AFTER the backfill completes
    ✓ ★ an alert on remaining un-migrated documents
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
Create a 1M-row `jsonb` table. Build all three index types and compare their sizes. Then show: (a) `@>` using the GIN index; (b) a range predicate falling back to a sequential scan; (c) the same range using a B-tree expression index. Report all three timings.

### Exercise 2 — medium (apply it)
Build an embedded-comments document and append 20,000 comments one at a time, measuring total time and final document size. Build the referenced equivalent and do the same. Report the ratio for writes, for reading the title, and for paginating comments.

Then add a generated column with a `CHECK` constraint to an existing `jsonb` table and show that it finds pre-existing invalid data.

### Exercise 3 — hard (production simulation)
A B2B catalogue stores 800,000 products across 400 categories using EAV — 41 million attribute rows, a 12-way self-join per query, and an 8,412 ms search p99.

(a) Explain the three separate reasons the EAV query is slow.
(b) Evaluate one-table-per-category, a full document store, and a hybrid. Give the deciding factor for rejecting the first two.
(c) Design the hybrid schema. Justify which fields are relational and which are `jsonb`.
(d) Users filter by attribute *ranges*, and `@>` cannot do ranges. Give three options and the measurement that chooses between them.
(e) The filter distribution shows 4 of 350 attributes are 96% of usage. Design the promotion using generated columns, and explain why partial indexes matter here.
(f) Make "schemaless" honest: design the schema registry, the validation trigger, and explain what it prevents.
(g) A v3 schema adds a required field to 741,204 existing v2 documents. Write the migration, including batching, and explain the ordering of the steps.
(h) Explain the role of the version field and why the migration would be intractable without it.
(i) The final design keeps foreign keys to `categories` and `suppliers`. Explain why that mattered enough to rule out a dedicated document store.

---

## Mental model checkpoint

1. What does `jsonb` do to keys, duplicates and numbers that `json` does not?
2. Why is `jsonb` roughly 3× relational storage for the same data?
3. Name the three index strategies with their relative sizes. Which can do ranges?
4. Why can't `@>` do range queries, and what are the three workarounds?
5. State the four conditions for embedding and the five for referencing.
6. Why is an unbounded embedded array always eventually a bug? Name three failure modes.
7. Why does updating one field of a large document cost so much?
8. What does "schemaless" actually mean, and what three costs does it carry?
9. What is a generated column here, and why can it not drift?
10. Where is the atomicity boundary in MongoDB, and why does that constrain the document design?
11. Why is that constraint absent in PostgreSQL, and what does that let you do?
12. Why is a version field essential?

---

## Quick reference card

```sql
-- ★ operators
doc->'k'  doc->>'k'  doc#>'{a,b}'  doc#>>'{a,b}'
doc @> '{"status":"paid"}'          -- ★ containment, GIN-indexable
doc ? 'key'   doc ?| ARRAY[...]     -- ★ existence (jsonb_ops only)
doc @@ '$.n > 100'                  -- ★ jsonpath, ranges, jsonb_ops only
jsonb_array_elements(doc->'items')  -- ★ unnest for aggregation
```

| Index | Size (5M) | Supports |
|---|---|---|
| `gin (doc)` | ★ 1,840 MB | `@>` `?` `?|` `?&` `@?` `@@` |
| ★ `gin (doc jsonb_path_ops)` | ★ **612 MB** | ★ `@>` only |
| ★ `btree ((doc->>'x'))` | ★ **88 MB** | ★ **ranges, `ORDER BY`** |

**★ Embed only if:** no independent lifecycle · ★ **bounded array** · read together · never queried across parents.
**★ Reference if any of:** queried across parents · ★ **unbounded** · updated more than the parent · shared · needs FKs.

**★ The hybrid**
```sql
CREATE TABLE t (
  id bigserial PRIMARY KEY,
  fk bigint REFERENCES other(id),         -- ★ relational core
  status text CHECK (status IN (…)),
  attrs jsonb NOT NULL DEFAULT '{}',      -- ★ variable remainder
  hot int GENERATED ALWAYS AS ((attrs->>'hot')::int) STORED  -- ★ the bridge
);
CREATE INDEX ON t (hot) WHERE hot IS NOT NULL;
CREATE INDEX ON t USING gin (attrs jsonb_path_ops);
```

**Physics:** every key name repeated ⇒ ★ 3× storage · >2 KB ⇒ ★ TOAST ⇒ second fetch · ★ **any update rewrites the whole value** · the planner can't estimate `jsonb`.

**★ Make schemaless honest:** a version field · storage-layer validation · generated columns + `CHECK`.
**★ Migration is an application backfill** — batch it, commit per batch, tighten the schema last.

---

## When would I use this at work?

1. **Any domain with genuinely variable structure** — product attributes, form responses, webhook payloads, event data, per-tenant custom fields. `jsonb` is the right answer and EAV (Topic 27) almost never is.

2. **Before adding MongoDB to a PostgreSQL stack.** Ask which *part* of the data is document-shaped. Usually it's one table's worth, and `jsonb` gives you the shape while keeping transactions, foreign keys and joins for everything else — plus one database to operate, back up and secure.

3. **Reviewing any embedded array.** One question: *can you state a maximum?* If not, it will eventually become a multi-megabyte document that is expensive to read, catastrophic to write, and impossible to paginate.

4. **When a "properly indexed" document query is slow.** It's almost always a range predicate against a GIN index. A generated column with a B-tree index fixes it, and measuring which fields are actually filtered usually shows you need three, not three hundred.

---

## Connected topics

**Understand before this:** 04 (TOAST — the mechanism behind write amplification), 16 (GIN/GiST), 27 (EAV — what this replaces), 55 (embedding as a denormalisation), 53 (generated columns cannot drift).

**This unlocks:**
- **71** — key-value design: the same access-pattern-first thinking, taken further
- **72** — wide-column: where the partition key replaces the document key
- **75** — SQL vs NoSQL: the decision framework in full
- **76** — polyglot persistence: running both without lying about consistency
