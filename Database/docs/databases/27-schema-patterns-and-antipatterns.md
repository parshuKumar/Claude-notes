# 27 — Schema Patterns and Antipatterns
## Phase: Database Design

---

## ELI5 — The Simple Analogy

A kitchen drawer.

The first sensible idea is **one drawer per kind of thing**: cutlery here, utensils there, foil and clingfilm in the third. Everything has a place; you find things instantly.

Then someone has a "flexible" idea: **one giant drawer with a notebook.** Everything goes in the drawer, and the notebook says "item 47 is a spatula, item 48 is a battery, item 49 is a spatula." Now adding a new *kind* of thing needs no new drawer — brilliant! — but finding all the spatulas means reading the whole notebook, and nothing stops you writing "item 50 is a spatula" for a teaspoon.

That's **EAV**, and it is the most seductive antipattern in database design: it solves a real problem (varying attributes) by throwing away the one thing that makes a database a database — knowing what each thing is.

This topic is a catalogue of those trades. Six patterns that look clever and cost you, and six that look boring and save you.

---

## Where this fits in the big picture

```
   20–26 — ER modelling → schema → keys → FKs → constraints → types
                          │
                          ▼
        ┌──────────────────────────────────────────┐
        │ 27 PATTERNS AND ANTIPATTERNS             │ ← YOU ARE HERE
        │ the shapes that recur, and what they cost│
        └────────────────────┬─────────────────────┘
                             │
              ┌──────────────┴──────────────┐
              ▼                             ▼
        28 migrations              29–38 normalisation
        (fixing these safely)      (the theory that explains
                                    WHY most of these are wrong)
```

Everything in Topics 20–26 was about doing one thing correctly. **This topic is about recognising a *shape* — the recurring design that a hundred teams have tried, with a known outcome.**

---

## What is this?

A catalogue of six antipatterns and six patterns, each with:

- what it looks like
- why it's tempting (they all solve a real problem)
- what it actually costs, measured
- the correct alternative
- and — importantly — **the narrow case where the antipattern is genuinely right**

Because none of these is universally wrong. EAV is correct for a medical-records system with 40,000 possible observations. Soft deletes are correct when the law requires retention. The skill is recognising which situation you're in.

---

## Why does it matter for a backend developer?

Because these shapes are *recognisable*, which makes them the cheapest possible thing to get right:

```
 An engineer who has never seen EAV before will invent it, be pleased
 with it, and discover its costs eighteen months later when the query
 to render one product page is 40 lines and takes 4 seconds.

 An engineer who recognises the shape spends five minutes on JSONB or
 a class-table design instead, and never has that eighteen months.
```

And the antipatterns cluster around the same four root causes, which is what makes them learnable rather than memorisable:

| Root cause | Antipatterns it produces |
|---|---|
| Wanting flexibility without a migration | EAV, JSONB sprawl, generic "meta" tables |
| Wanting to keep history without designing for it | soft deletes, status columns with no history |
| Wanting one table to do two jobs | polymorphic FKs, single-table inheritance |
| Fear of joins | premature denormalisation, comma-separated columns |

---

## The physical reality

### EAV — what it costs, in pages

```
 THE PATTERN
   entity_attributes(entity_id, attribute_name, value_text)

 ONE PRODUCT WITH 12 ATTRIBUTES:
   normal table:  1 row  ×  ~120 bytes  = 1 row,  ~1 page fetch
   EAV:          12 rows × ~72 bytes    = 12 rows, 12 index lookups
                                           + 12 heap fetches

 ┌─────────────────────────────────────────────────────────────────┐
 │ EAV heap page                                                   │
 │  [24 hdr][ent 8][attr 'colour' 8+][val 'red' 4+][type 4]  ~72 B │
 │  [24 hdr][ent 8][attr 'size'   6+][val 'M'   2+][type 4]  ~68 B │
 │  ★ the ATTRIBUTE NAME is stored in EVERY ROW.                   │
 │    12 attributes × 8M products = 96M rows, of which ~35% is     │
 │    repeated attribute names.                                    │
 └─────────────────────────────────────────────────────────────────┘

 RENDERING ONE PRODUCT PAGE:
   normal:  SELECT * FROM products WHERE id=88;        → 1 page
   EAV:     SELECT * FROM entity_attributes WHERE entity_id=88;
            → 12 index entries + up to 12 heap pages, then PIVOT in
              the application, then coerce every value from text.

 FILTERING ON TWO ATTRIBUTES ("red, size M"):
   normal:  WHERE colour='red' AND size='M'      → one composite index
   EAV:     a SELF-JOIN PER ATTRIBUTE:
            FROM ea a JOIN ea b ON b.entity_id=a.entity_id
            WHERE a.attribute='colour' AND a.value='red'
              AND b.attribute='size'   AND b.value='M'
            ⇒ N attributes filtered = N-1 self-joins.
              Five filters = a 5-way self-join on a 96M-row table.
```

### Soft deletes — the cost nobody counts

```
 ALTER TABLE users ADD COLUMN deleted_at timestamptz NULL;

 WHAT YOU JUST SIGNED UP FOR:
  ① EVERY QUERY, FOREVER, needs `AND deleted_at IS NULL`.
     Forget it once and you show deleted data. There is no compiler
     that catches this.
  ② EVERY UNIQUE CONSTRAINT must become partial:
       CREATE UNIQUE INDEX … WHERE deleted_at IS NULL
     otherwise a deleted user's email is reserved forever.
  ③ EVERY FOREIGN KEY still points at the soft-deleted row.
     The database considers it present. Your app considers it gone.
     ⇒ ★ THE DIVERGENCE: the DB's referential integrity and your
       application's notion of existence no longer agree.
  ④ THE TABLE NEVER SHRINKS. Deleted rows occupy pages, buffer pool,
     and index entries forever (Topic 04).
  ⑤ Every index is larger and every scan reads deleted rows.

 MEASURED on a table 60% soft-deleted:
   rows_per_page effective:      126 → 50   (60% of what you read is dead)
   idx_users_email size:          1.8 GB (of which 1.1 GB is deleted rows)
   every query:                   +1 predicate, +1 chance to forget it
```

### Polymorphic FKs — why the database can't help you

```
 THE PATTERN
   comments(id, commentable_type text, commentable_id bigint, body text)
             -- commentable_type ∈ {'post','photo','video'}

 ★ YOU CANNOT DECLARE A FOREIGN KEY. There is no single parent table.
 ⇒ the database cannot:
     • prevent commentable_id = 99999 with no such post
     • cascade a delete
     • give the planner a join it can optimise
 ⇒ every join is a UNION or a CASE:
     SELECT c.*, COALESCE(p.title, ph.caption, v.title)
     FROM comments c
     LEFT JOIN posts p  ON c.commentable_type='post'  AND p.id=c.commentable_id
     LEFT JOIN photos ph ON c.commentable_type='photo' AND ph.id=c.commentable_id
     LEFT JOIN videos v ON c.commentable_type='video' AND v.id=c.commentable_id
   ⇒ N LEFT JOINs, N indexes probed, and the planner cannot prune —
     it does not know that a 'post' row will never match photos.

 MEASURED: 3 target types, 40M comments
   polymorphic:  4 index probes per comment, 210 ms for 50 comments
   exclusive-arc: 1 index probe, FK-enforced, 3 ms
```

---

## How it works — step by step

### ANTIPATTERN 1 — EAV (Entity-Attribute-Value)

```sql
-- THE SHAPE
CREATE TABLE entity_attributes (
  entity_id      bigint NOT NULL,
  attribute_name text   NOT NULL,
  value_text     text,
  value_number   numeric,
  value_date     timestamptz,
  PRIMARY KEY (entity_id, attribute_name)
);
```

```
 WHY IT'S TEMPTING: "products in different categories have different
 attributes; a laptop has RAM, a shirt has fabric. We can't have 400
 columns, and we don't want a migration every time merchandising adds
 an attribute."   ← A GENUINELY REAL PROBLEM.

 WHAT IT COSTS:
  ✗ NO TYPES. 'blue' can go in value_number's row. Nothing stops it.
  ✗ NO CONSTRAINTS. You cannot CHECK that ram_gb is between 4 and 128.
  ✗ NO NOT NULL. You cannot require that every laptop has a CPU.
  ✗ N-1 SELF-JOINS to filter on N attributes.
  ✗ The planner's estimates are meaningless (Topic 15): it sees one
    table with a uniform distribution, not 400 differently-shaped ones.
  ✗ Every read pivots in the application.

 THE ALTERNATIVES, in order of preference:

  ① JSONB — flexible, typed-ish, indexable, one row per entity
     CREATE TABLE products (
       id bigint … , category_id int NOT NULL,
       name text NOT NULL, price_minor bigint NOT NULL,   -- ★ shared columns
       attributes jsonb NOT NULL DEFAULT '{}'             -- ★ the varying tail
     );
     CREATE INDEX ON products USING gin (attributes jsonb_path_ops);
     SELECT … WHERE attributes @> '{"colour":"red","size":"M"}';
     ✓ ONE row, ONE index probe, no self-joins
     ✓ add an attribute with zero DDL
     ✓ per-category validation via a JSON Schema in a `categories` table,
       enforced at the write boundary
     ✗ still no DB-level type/constraint enforcement on the inner values

  ② CLASS-TABLE INHERITANCE — when categories are few and stable
     products(id, name, price_minor, category)         -- shared
     laptop_specs(product_id PK, ram_gb, cpu, screen_in)
     shirt_specs(product_id PK, size, fabric, fit)
     ✓ FULL typing, constraints, NOT NULL per category
     ✗ a new category is a migration
     → correct when categories number in the tens, not thousands

  ③ THE HYBRID — ★ what most real systems land on
     the ~8 attributes EVERY product has → real columns, constrained
     the long tail                        → jsonb + GIN
     the 3 attributes you FILTER on most  → generated columns + btree
       ALTER TABLE products ADD COLUMN colour text
         GENERATED ALWAYS AS (attributes->>'colour') STORED;
       CREATE INDEX ON products (category_id, colour);

 ★ WHEN EAV IS ACTUALLY CORRECT:
   • the attribute set is genuinely unbounded AND user-defined
     (a clinical system with 40,000 LOINC observation codes; a
      form-builder where customers define their own fields)
   • you never filter on more than one attribute at a time
   • the attribute catalogue is itself data, with its own metadata
   ⇒ and even then, model the CATALOGUE properly:
       attribute_definitions(id, code, data_type, unit, valid_range,
                             is_required, category_id)
     so the types and constraints live SOMEWHERE, even if the engine
     can't enforce them.
```

### ANTIPATTERN 2 — Soft deletes everywhere

```sql
-- THE SHAPE
ALTER TABLE users ADD COLUMN deleted_at timestamptz NULL;
-- and now every query, forever:
SELECT * FROM users WHERE deleted_at IS NULL AND …;
```

```
 WHY IT'S TEMPTING: "we need to be able to undelete", "compliance needs
 the record", "cascading deletes are scary."   ← ALL REAL.

 WHAT IT COSTS: the five items in "physical reality" above.

 THE ALTERNATIVES:

  ① A VIEW that hides the predicate — the cheapest mitigation
     ALTER TABLE users RENAME TO users_all;
     CREATE VIEW users AS SELECT * FROM users_all WHERE deleted_at IS NULL;
     ✓ you cannot forget the predicate
     ✓ updatable if simple; otherwise add INSTEAD OF triggers
     ✗ the table still never shrinks

  ② ARCHIVE TABLE — move, don't mark
     WITH moved AS (DELETE FROM users WHERE id=$1 RETURNING *)
     INSERT INTO users_archive SELECT *, now() FROM moved;
     ✓ the live table stays small and fast
     ✓ history is retained
     ✓ FKs behave correctly — the row genuinely is gone
     ✗ FKs from other tables must be handled (archive them too, or
       RESTRICT and require an explicit workflow — Topic 23)

  ③ ANONYMISE, DON'T DELETE — for GDPR/PII
     ★ often the CORRECT answer for users specifically (Topic 23):
       the row must remain for financial history, but the PII must go.

  ④ ROW-LEVEL SECURITY — enforce it in the engine
     ALTER TABLE users ENABLE ROW LEVEL SECURITY;
     CREATE POLICY p_live ON users USING (deleted_at IS NULL);
     ✓ the predicate CANNOT be forgotten, by anyone, ever
     ✓ works for psql sessions and other services too
     ✗ a superuser/owner bypasses it unless FORCE is set

 ★ WHEN SOFT DELETE IS CORRECT:
   • undelete is a genuine, used product feature (trash bin)
   • the deleted rows are a small fraction (<5%) and stay that way
   • ★ and you have applied ① or ④ so the predicate cannot be forgotten
   ⇒ Otherwise: archive.
```

### ANTIPATTERN 3 — Polymorphic foreign keys

```sql
-- THE SHAPE
CREATE TABLE comments (
  id bigint …, commentable_type text NOT NULL, commentable_id bigint NOT NULL
);
```

```
 WHY IT'S TEMPTING: "comments work the same on posts, photos and
 videos; one table is obviously right."  ← the OBSERVATION is right;
 the CONCLUSION is wrong.

 WHAT IT COSTS: no FK, no cascade, N-way LEFT JOINs, unusable estimates.

 THE ALTERNATIVES:

  ① EXCLUSIVE ARC — one nullable FK per target, with a CHECK ★ preferred
     CREATE TABLE comments (
       id bigint …, body text NOT NULL,
       post_id  bigint NULL REFERENCES posts(id)  ON DELETE CASCADE,
       photo_id bigint NULL REFERENCES photos(id) ON DELETE CASCADE,
       video_id bigint NULL REFERENCES videos(id) ON DELETE CASCADE,
       CHECK (num_nonnulls(post_id, photo_id, video_id) = 1)
     );
     ✓ REAL foreign keys, real cascades
     ✓ the planner sees a normal join
     ✓ num_nonnulls() enforces exactly one
     ✗ a new target type is a migration (usually fine — how often?)

  ② SHARED SUPERTYPE — when there are many target types
     CREATE TABLE commentables (id bigint GENERATED ALWAYS AS IDENTITY PK,
                                kind text NOT NULL);
     CREATE TABLE posts  (id bigint PRIMARY KEY REFERENCES commentables(id), …);
     CREATE TABLE photos (id bigint PRIMARY KEY REFERENCES commentables(id), …);
     CREATE TABLE comments (commentable_id bigint REFERENCES commentables(id), …);
     ✓ ONE clean FK; new types need no comments migration
     ✗ every insert touches two tables
     → this is Topic 21's Strategy C, applied to the target side

  ③ SEPARATE TABLES — when the volumes differ wildly
     post_comments, photo_comments, video_comments
     ✓ simplest, fastest, fully constrained
     ✗ "all comments by user X" is a UNION
     → correct when one type dwarfs the others

 ★ WHEN POLYMORPHIC IS ACCEPTABLE:
   • the target set is genuinely open (a generic audit/notification
     table referencing "any entity")
   • you never join to the target in a hot path
   • ⇒ and even then, add a reconciliation job that detects dangling
     references (Topic 23's discipline).
```

### ANTIPATTERN 4 — The status column with no history

```sql
-- THE SHAPE
ALTER TABLE orders ADD COLUMN status text NOT NULL DEFAULT 'pending';
UPDATE orders SET status='shipped' WHERE id=$1;
```

```
 WHAT IT COSTS:
  ✗ "when did it ship?" — unanswerable
  ✗ "how long between paid and shipped?" — unanswerable
  ✗ "which orders went backwards from shipped to pending?" — invisible
  ✗ ★ AND at scale, an indexed status column destroys HOT updates
    (Topic 17): every status change rewrites every index entry.

 THE ALTERNATIVES:

  ① TIMESTAMP PER STATE — cheap, and usually enough
     orders(placed_at, paid_at, shipped_at, delivered_at, cancelled_at)
     status text GENERATED ALWAYS AS (
       CASE WHEN cancelled_at IS NOT NULL THEN 'cancelled'
            WHEN delivered_at IS NOT NULL THEN 'delivered'
            WHEN shipped_at   IS NOT NULL THEN 'shipped'
            WHEN paid_at      IS NOT NULL THEN 'paid'
            ELSE 'pending' END) STORED;
     ✓ full history, one row, no join
     ✓ durations are subtraction
     ✓ ★ the status CANNOT disagree with the timestamps
     ✗ only works for a linear, non-repeating lifecycle

  ② A TRANSITION TABLE — for branching or repeating lifecycles
     order_status_history(order_id, seq, from_status, to_status,
                          changed_at, changed_by, reason)
     ✓ full audit, arbitrary state machines, repeated states
     ✗ current status needs a lateral/latest lookup — so keep a
       denormalised `status` on the parent, refreshed in the same txn

  ③ VALIDITY RANGES — when a state has a duration (Topic 26)
```

### ANTIPATTERN 5 — Comma-separated values in a column

```sql
-- THE SHAPE
tags text            -- 'ethnic,sale,bestseller'
```
```
 ✗ violates 1NF (Topic 31)
 ✗ `WHERE tags LIKE '%sale%'` matches 'flash-sale-2024' too
 ✗ no index can serve it (leading wildcard — Topic 16)
 ✗ no referential integrity to a tag list
 ✗ removing one tag is string surgery

 ALTERNATIVES:
   ① a junction table (product_tags) — ★ correct when tags are entities
      with their own attributes, or you need FK integrity
   ② text[] + GIN — acceptable when tags are opaque labels, always read
      whole, and you only need containment (`@>`)
   ⇒ NEVER a delimited string. Not once.
```

### ANTIPATTERN 6 — The "flexible" generic table

```sql
-- THE SHAPE
CREATE TABLE data (id bigint, type text, json_blob jsonb);
-- "everything goes here; we'll figure out the schema later"
```
```
 This is EAV with extra steps, plus polymorphism, plus no types.
 It has every cost of both and no compensating benefit.
 ⇒ There is no case where this is correct. If the shape is genuinely
   unknown, land it in an `events`/`staging` table with a documented
   payload, and MODEL IT within the sprint.
```

### THE SIX PATTERNS THAT ARE ACTUALLY GOOD

```
 ① AUDIT COLUMNS ON EVERY TABLE
    created_at timestamptz NOT NULL DEFAULT now(),
    updated_at timestamptz NOT NULL DEFAULT now(),
    created_by bigint NULL, updated_by bigint NULL
    ✓ 16–32 bytes that answer a hundred future questions
    ⚠ updated_at needs a trigger, or it lies:
      CREATE FUNCTION touch() RETURNS trigger AS $$
      BEGIN NEW.updated_at := now(); RETURN NEW; END $$ LANGUAGE plpgsql;
      CREATE TRIGGER t BEFORE UPDATE ON tbl FOR EACH ROW EXECUTE FUNCTION touch();

 ② LOOKUP TABLES FOR THINGS WITH ATTRIBUTES
    order_statuses(id, code, label, is_terminal, sort_order, colour)
    ✓ when the list has attributes or non-engineers edit it (Topic 25)

 ③ THE OUTBOX (Topic 52)
    outbox(id, topic, payload jsonb, created_at, published_at)
    ✓ the ONLY correct way to write to a DB and a queue atomically

 ④ IDEMPOTENCY KEYS (case studies 01–04)
    idempotency_key uuid NOT NULL, with a UNIQUE index
    ✓ on every externally-triggered write

 ⑤ THE SNAPSHOT-PLUS-REFERENCE (Topics 21, 26)
    order_lines(product_id,          -- reference, for analytics
                product_name,        -- snapshot, for the receipt
                unit_price_minor)    -- snapshot, for the total
    ✓ two questions, two mechanisms. Not redundancy.

 ⑥ THE HOT/COLD VERTICAL SPLIT (Topic 04)
    orders(id, user_id, total, status)          -- narrow, hot, scanned
    order_details(order_id PK, raw_payload, notes, metadata)  -- wide, cold
    ✓ 20× more rows per page on the table you actually scan
```

---

## Concept breakdown

```
THE FOUR ROOT CAUSES
│
├── ① FLEXIBILITY WITHOUT MIGRATION  → EAV · JSONB sprawl · generic tables
│      ⇒ the real fix: JSONB for the varying tail, columns for the rest
├── ② HISTORY WITHOUT DESIGN         → soft deletes · status with no history
│      ⇒ the real fix: archive tables, timestamps per state, validity ranges
├── ③ ONE TABLE, TWO JOBS            → polymorphic FKs · single-table inheritance
│      ⇒ the real fix: exclusive arc, or a shared supertype (Topic 21 Rule 8)
└── ④ FEAR OF JOINS                  → premature denormalisation · CSV columns
       ⇒ the real fix: measure first. A hash join over 8M rows is 400 ms
         (Topic 19). Most join fear is unmeasured.

THE TEST FOR EVERY "FLEXIBLE" DESIGN
  Ask: WHAT CAN THE DATABASE STILL ENFORCE?
    a normal column   → type, NOT NULL, CHECK, UNIQUE, FK
    JSONB             → the column is valid JSON. That's it.
    EAV               → nothing
    polymorphic FK    → nothing about the reference
  ⇒ Every step toward flexibility is a step away from guarantees.
    That trade is sometimes right — but make it deliberately, and
    write down what you gave up and what now enforces it instead.

THE TEST FOR SOFT DELETES
  Can you name a product feature that undeletes? If not, it's an
  archive table.

THE TEST FOR POLYMORPHISM
  How many target types, and how often does a new one appear?
    ≤ 5, rarely     → exclusive arc
    many, or open   → shared supertype
    volumes differ
      by 100×       → separate tables
```

---

## Diagrams

**Diagram 1 — big picture: the flexibility/guarantee trade**

```
  GUARANTEES the database can enforce
        ▲
        │ ████ typed columns
        │ ████ + NOT NULL, CHECK, UNIQUE, FK, EXCLUDE
        │ ███
        │ ██   generated columns from JSONB
        │ █    JSONB + a validator at the write boundary
        │ ▓    JSONB, unvalidated
        │ ░    EAV
        │ ·    polymorphic FK / generic table
        └──────────────────────────────────────────▶
            FLEXIBILITY without a migration

  ★ You cannot move right without moving down. The question is never
    "which is better" — it is "how far right do I actually need to be,
    and what will enforce the rules I just gave up?"
```

**Diagram 2 — data flow: rendering one product page**

```
  NORMAL COLUMNS                    EAV
  ──────────────────────────        ──────────────────────────────────
   SELECT * FROM products            SELECT * FROM entity_attributes
    WHERE id=88                       WHERE entity_id=88
        │                                  │
   1 index probe                      1 index probe
   1 heap page                        12 heap pages (scattered)
        │                                  │
        ▼                                  ▼
   a typed row                        12 (name, value) text pairs
                                           │
                                      PIVOT in the application
                                           │
                                      COERCE every value from text
                                           │
                                      ⚠ and hope the types are right
        │                                  │
        ▼                                  ▼
      3 ms                              41 ms

  FILTERING ON 3 ATTRIBUTES:
   normal:  one composite index scan          →  4 ms
   EAV:     a 3-way SELF-JOIN on 96M rows     →  2,800 ms
```

**Diagram 3 — before/after: soft deletes at 60%**

```
 BEFORE — hard delete / archive
 ┌──────────────────────────────────────────────────────────┐
 │ users: 400,000 live rows · 3,200 pages · idx 12 MB       │
 │ SELECT … WHERE email=$1     → 4 pages                    │
 │ every query: no extra predicate                          │
 └──────────────────────────────────────────────────────────┘

 AFTER — soft delete, 60% deleted over 3 years
 ┌──────────────────────────────────────────────────────────┐
 │ users: 1,000,000 rows (400k live) · 8,000 pages · idx 30MB│
 │ SELECT … WHERE email=$1 AND deleted_at IS NULL → 4 pages │
 │   (fine — the index still points straight at it)          │
 │ SELECT … WHERE created_at > X AND deleted_at IS NULL      │
 │   → reads 8,000 pages, discards 60% ★                     │
 │ every query: +1 predicate, +1 chance to forget it         │
 │ every UNIQUE: must be partial, or addresses are reserved  │
 │   forever                                                 │
 │ every FK: still points at a "deleted" row ★               │
 └──────────────────────────────────────────────────────────┘
        ↑ the table never shrinks, and never will
```

---

## Example 1 — basic

**Step 1 — EAV vs JSONB vs columns, measured.**

```sql
-- ① EAV
CREATE TABLE eav_products (id bigint PRIMARY KEY, name text, price_minor bigint);
CREATE TABLE eav_attrs (
  entity_id bigint NOT NULL, attribute_name text NOT NULL, value_text text,
  PRIMARY KEY (entity_id, attribute_name)
);
CREATE INDEX idx_eav_lookup ON eav_attrs (attribute_name, value_text);

INSERT INTO eav_products SELECT i,'Product '||i,(random()*500000)::bigint
FROM generate_series(1,2000000) i;
INSERT INTO eav_attrs
SELECT i, a.name, a.val FROM generate_series(1,2000000) i,
LATERAL (VALUES
  ('colour',(ARRAY['red','blue','green','black'])[1+(i%4)]),
  ('size',  (ARRAY['S','M','L','XL'])[1+(i%4)]),
  ('brand', (ARRAY['Fabindia','W','Biba'])[1+(i%3)]),
  ('fabric',(ARRAY['cotton','silk','linen'])[1+(i%3)])
) a(name,val);

-- ② JSONB
CREATE TABLE json_products (
  id bigint PRIMARY KEY, name text, price_minor bigint, attrs jsonb NOT NULL
);
INSERT INTO json_products
SELECT i,'Product '||i,(random()*500000)::bigint,
  jsonb_build_object('colour',(ARRAY['red','blue','green','black'])[1+(i%4)],
                     'size',(ARRAY['S','M','L','XL'])[1+(i%4)],
                     'brand',(ARRAY['Fabindia','W','Biba'])[1+(i%3)],
                     'fabric',(ARRAY['cotton','silk','linen'])[1+(i%3)])
FROM generate_series(1,2000000) i;
CREATE INDEX idx_json_attrs ON json_products USING gin (attrs jsonb_path_ops);

-- ③ COLUMNS
CREATE TABLE col_products (
  id bigint PRIMARY KEY, name text, price_minor bigint,
  colour text NOT NULL, size text NOT NULL, brand text NOT NULL, fabric text NOT NULL
);
INSERT INTO col_products SELECT id,name,price_minor,
  attrs->>'colour',attrs->>'size',attrs->>'brand',attrs->>'fabric' FROM json_products;
CREATE INDEX idx_col_filter ON col_products (colour, size, brand);

VACUUM ANALYZE eav_attrs; VACUUM ANALYZE json_products; VACUUM ANALYZE col_products;
```

**Storage:**
```sql
SELECT 'eav' AS design, pg_size_pretty(pg_total_relation_size('eav_products')
                                     + pg_total_relation_size('eav_attrs')) AS total
UNION ALL SELECT 'jsonb', pg_size_pretty(pg_total_relation_size('json_products'))
UNION ALL SELECT 'columns', pg_size_pretty(pg_total_relation_size('col_products'));
```
```
 design  |  total
---------+---------
 eav     | 1284 MB
 jsonb   |  684 MB
 columns |  312 MB      ← 4.1× smaller than EAV
```

**Filtering on three attributes:**
```sql
-- EAV: a 3-way self-join
EXPLAIN (ANALYZE,BUFFERS)
SELECT p.id FROM eav_products p
JOIN eav_attrs a ON a.entity_id=p.id AND a.attribute_name='colour' AND a.value_text='red'
JOIN eav_attrs b ON b.entity_id=p.id AND b.attribute_name='size'   AND b.value_text='M'
JOIN eav_attrs c ON c.entity_id=p.id AND c.attribute_name='brand'  AND c.value_text='W';
```
```
Execution Time: 2841.2 ms      Buffers: shared hit=48204 read=412008
```
```sql
EXPLAIN (ANALYZE,BUFFERS) SELECT id FROM json_products
WHERE attrs @> '{"colour":"red","size":"M","brand":"W"}';
-- Execution Time: 88.4 ms     Buffers: shared hit=8412

EXPLAIN (ANALYZE,BUFFERS) SELECT id FROM col_products
WHERE colour='red' AND size='M' AND brand='W';
-- Execution Time: 12.1 ms     Buffers: shared hit=1204
```
**2,841 ms → 88 ms → 12 ms.** EAV is 235× slower than columns for the query the schema exists to serve.

**Step 2 — the hybrid, which is what you actually ship.**

```sql
ALTER TABLE json_products
  ADD COLUMN colour text GENERATED ALWAYS AS (attrs->>'colour') STORED,
  ADD COLUMN size   text GENERATED ALWAYS AS (attrs->>'size')   STORED;
CREATE INDEX idx_hybrid ON json_products (colour, size);
VACUUM ANALYZE json_products;

EXPLAIN (ANALYZE,BUFFERS) SELECT id FROM json_products
WHERE colour='red' AND size='M' AND attrs @> '{"brand":"W"}';
-- Execution Time: 14.8 ms
```
**Column speed on the attributes you filter, JSONB flexibility on the rest.**

**Step 3 — soft deletes, and the two things people forget.**

```sql
CREATE TABLE sd_users (
  id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  email text NOT NULL, deleted_at timestamptz NULL
);
CREATE UNIQUE INDEX uq_sd_email ON sd_users (email);       -- ⚠ NOT partial

INSERT INTO sd_users (email) VALUES ('arjun@shop.in');
UPDATE sd_users SET deleted_at = now() WHERE email='arjun@shop.in';
INSERT INTO sd_users (email) VALUES ('arjun@shop.in');
-- ERROR: duplicate key value violates unique constraint "uq_sd_email"
--   ★ the deleted user's address is reserved FOREVER
```
```sql
DROP INDEX uq_sd_email;
CREATE UNIQUE INDEX uq_sd_email_live ON sd_users (email) WHERE deleted_at IS NULL;
INSERT INTO sd_users (email) VALUES ('arjun@shop.in');     -- ✓
```

And the FK divergence:
```sql
CREATE TABLE sd_orders (id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
                        user_id bigint NOT NULL REFERENCES sd_users(id));
INSERT INTO sd_orders (user_id) SELECT id FROM sd_users WHERE deleted_at IS NOT NULL;
-- ✓ SUCCEEDS. The database sees a live parent row.
-- ★ Your application believes that user is gone. The two now disagree.
```

**Step 4 — the view/RLS mitigation.**

```sql
ALTER TABLE sd_users ENABLE ROW LEVEL SECURITY;
ALTER TABLE sd_users FORCE ROW LEVEL SECURITY;   -- ★ applies to the owner too
CREATE POLICY p_live ON sd_users USING (deleted_at IS NULL);
SET ROLE app;
SELECT count(*) FROM sd_users;                    -- only live rows, always
```
**The predicate can no longer be forgotten — by anyone, including a psql session.**

**Step 5 — polymorphic vs exclusive arc.**

```sql
-- polymorphic
CREATE TABLE poly_comments (id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  target_type text NOT NULL, target_id bigint NOT NULL, body text NOT NULL);
INSERT INTO poly_comments (target_type,target_id,body) VALUES ('post', 999999, 'x');
-- ✓ SUCCEEDS. Post 999999 does not exist. Nothing can prevent this.

-- exclusive arc
CREATE TABLE posts  (id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY, title text);
CREATE TABLE photos (id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY, caption text);
CREATE TABLE arc_comments (
  id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY, body text NOT NULL,
  post_id  bigint NULL REFERENCES posts(id)  ON DELETE CASCADE,
  photo_id bigint NULL REFERENCES photos(id) ON DELETE CASCADE,
  CHECK (num_nonnulls(post_id, photo_id) = 1)
);
CREATE INDEX ON arc_comments (post_id)  WHERE post_id  IS NOT NULL;
CREATE INDEX ON arc_comments (photo_id) WHERE photo_id IS NOT NULL;

INSERT INTO arc_comments (body, post_id) VALUES ('x', 999999);
-- ERROR: violates foreign key constraint            ★ prevented
INSERT INTO arc_comments (body, post_id, photo_id) VALUES ('x', 1, 1);
-- ERROR: violates check constraint                  ★ exactly one enforced
```

**Step 6 — status with no history vs timestamps.**

```sql
CREATE TABLE orders_v1 (id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
                        status text NOT NULL DEFAULT 'pending');
UPDATE orders_v1 SET status='shipped' WHERE id=1;
-- "when did it ship?" → unanswerable. "how long from paid to shipped?" → unanswerable.

CREATE TABLE orders_v2 (
  id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  placed_at    timestamptz NOT NULL DEFAULT now(),
  paid_at      timestamptz NULL,
  shipped_at   timestamptz NULL,
  delivered_at timestamptz NULL,
  cancelled_at timestamptz NULL,
  status text GENERATED ALWAYS AS (
    CASE WHEN cancelled_at IS NOT NULL THEN 'cancelled'
         WHEN delivered_at IS NOT NULL THEN 'delivered'
         WHEN shipped_at   IS NOT NULL THEN 'shipped'
         WHEN paid_at      IS NOT NULL THEN 'paid'
         ELSE 'pending' END) STORED
);
INSERT INTO orders_v2 (paid_at, shipped_at) VALUES (now()-interval '2 days', now());
SELECT status, shipped_at - paid_at AS fulfilment_time FROM orders_v2;
```
```
 status  | fulfilment_time
---------+-----------------
 shipped | 2 days
```
**Full history, one row, no join — and the status cannot disagree with the timestamps.**

---

## Example 2 — production scenario

**The situation.** A B2B SaaS platform, 5 years old, 1.4 TB. A new team inherits it and finds four antipatterns compounding.

```sql
-- ① EAV for custom fields
CREATE TABLE custom_field_values (entity_type text, entity_id bigint,
                                  field_key text, value text);
-- 840 million rows

-- ② soft deletes on everything
-- 14 tables with deleted_at, 3 of them >50% deleted

-- ③ polymorphic activity log
CREATE TABLE activities (id bigserial, subject_type text, subject_id bigint,
                         verb text, actor_id bigint, created_at timestamptz);
-- 2.1 billion rows, 11 subject types

-- ④ a generic settings table
CREATE TABLE settings (scope text, scope_id bigint, key text, value jsonb);
```

**Step 1 — measure the damage.**

```sql
-- EAV: what fraction of the DB is repeated field names?
SELECT count(*) AS rows,
       pg_size_pretty(pg_total_relation_size('custom_field_values')) AS size,
       count(DISTINCT field_key) AS distinct_keys,
       pg_size_pretty((sum(length(field_key))+sum(length(entity_type)))::bigint) AS key_text
FROM custom_field_values;
```
```
   rows    |  size   | distinct_keys | key_text
-----------+---------+---------------+----------
 840412008 | 214 GB  |            47 | 18 GB
```
**18 GB of repeated key strings for 47 distinct values.**

```sql
-- soft deletes: how much of each table is dead?
SELECT relname,
       (SELECT count(*) FROM pg_attribute WHERE attrelid=c.oid AND attname='deleted_at') AS has_sd,
       pg_size_pretty(pg_relation_size(c.oid)) AS size
FROM pg_class c WHERE relkind='r' AND relnamespace='public'::regnamespace
ORDER BY pg_relation_size(c.oid) DESC LIMIT 5;

SELECT 'contacts' t, count(*) tot, count(*) FILTER (WHERE deleted_at IS NOT NULL) del
FROM contacts
UNION ALL SELECT 'deals', count(*), count(*) FILTER (WHERE deleted_at IS NOT NULL) FROM deals;
```
```
    t     |   tot    |   del
----------+----------+----------
 contacts | 41204882 | 28104112     ← 68% dead
 deals    | 12008412 |  7204008     ← 60% dead
```

```sql
-- polymorphic: dangling references
SELECT subject_type, count(*) AS dangling FROM activities a
WHERE (a.subject_type='deal'    AND NOT EXISTS (SELECT 1 FROM deals d WHERE d.id=a.subject_id))
   OR (a.subject_type='contact' AND NOT EXISTS (SELECT 1 FROM contacts c WHERE c.id=a.subject_id))
GROUP BY 1;
```
```
 subject_type | dangling
--------------+----------
 deal         |  1204882
 contact      |  4102008     ← 5.3M activities pointing at nothing
```

**Step 2 — prioritise by (business risk × migration cost).**

| Antipattern | Risk | Cost | Priority |
|---|---|---|---|
| Soft deletes at 68% | high — every query reads 3× the data | medium — archive tables | **1** |
| EAV, 47 keys | medium — slow filters, 214 GB | medium — 47 keys is small enough to promote | **2** |
| Polymorphic, 5.3M dangling | medium — wrong activity feeds | high — 2.1B rows, 11 types | **3** |
| Generic settings | low — small table, rarely queried | low | 4 |

**Step 3 — fix the soft deletes (archive).**

```sql
CREATE TABLE contacts_archive (LIKE contacts INCLUDING ALL);
ALTER TABLE contacts_archive ADD COLUMN archived_at timestamptz NOT NULL DEFAULT now();

-- batched, so no long transaction
DO $$ DECLARE moved int; BEGIN
  LOOP
    WITH batch AS (
      SELECT id FROM contacts WHERE deleted_at IS NOT NULL
       ORDER BY id LIMIT 10000 FOR UPDATE SKIP LOCKED
    ), gone AS (
      DELETE FROM contacts c USING batch b WHERE c.id=b.id RETURNING c.*
    )
    INSERT INTO contacts_archive SELECT *, now() FROM gone;
    GET DIAGNOSTICS moved = ROW_COUNT;
    EXIT WHEN moved = 0;
    COMMIT;
  END LOOP;
END $$;

VACUUM FULL contacts;   -- or pg_repack, to actually reclaim the space
```
```
 contacts: 41.2M rows / 38 GB  →  13.1M rows / 11 GB
 every scan of contacts: 3.1× faster, permanently
```
⚠ FKs from `deals` to `contacts` must be handled first — either archive the children too, or `RESTRICT` and require an explicit workflow (Topic 23).

**Step 4 — fix the EAV (promote the keys).**

```sql
-- 47 distinct keys is small. Find which are near-universal.
SELECT field_key, count(*) AS n,
       round(100.0*count(*)/(SELECT count(DISTINCT entity_id) FROM custom_field_values),1) AS pct_of_entities
FROM custom_field_values GROUP BY 1 ORDER BY 2 DESC LIMIT 12;
```
```
   field_key    |     n     | pct_of_entities
----------------+-----------+-----------------
 industry       | 40118004  |            97.4
 employee_count | 39204118  |            95.2
 lead_source    | 38104882  |            92.5
 …
 custom_note_7  |    41208  |             0.1     ← the genuine long tail
```
**Three keys appear on >92% of entities. Those are columns, not attributes.**

```sql
ALTER TABLE contacts
  ADD COLUMN industry text, ADD COLUMN employee_count int, ADD COLUMN lead_source text,
  ADD COLUMN custom_attrs jsonb NOT NULL DEFAULT '{}';

UPDATE contacts c SET
  industry       = (SELECT value FROM custom_field_values v
                     WHERE v.entity_type='contact' AND v.entity_id=c.id AND v.field_key='industry'),
  employee_count = (SELECT value::int FROM custom_field_values v
                     WHERE v.entity_type='contact' AND v.entity_id=c.id AND v.field_key='employee_count'),
  lead_source    = (SELECT value FROM custom_field_values v
                     WHERE v.entity_type='contact' AND v.entity_id=c.id AND v.field_key='lead_source');
-- (batched in production)

-- the long tail moves to jsonb
UPDATE contacts c SET custom_attrs = (
  SELECT jsonb_object_agg(field_key, value) FROM custom_field_values v
   WHERE v.entity_type='contact' AND v.entity_id=c.id
     AND v.field_key NOT IN ('industry','employee_count','lead_source'));

CREATE INDEX ON contacts (industry, lead_source);
CREATE INDEX ON contacts USING gin (custom_attrs jsonb_path_ops);
```
```
 custom_field_values: 214 GB → 0 (dropped)
 contacts:            11 GB → 14 GB (the promoted columns + jsonb)
 ⇒ net −211 GB
 "contacts in Retail with lead_source=webinar": 4,102 ms → 18 ms
```

**Step 5 — fix the polymorphic activity log.**

11 subject types over 2.1 billion rows makes an exclusive arc impractical (11 nullable columns). Use the shared supertype (Topic 21, Rule 8):

```sql
CREATE TABLE subjects (
  id   bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  kind text NOT NULL CHECK (kind IN ('deal','contact','company', …)),
  UNIQUE (id, kind)
);
ALTER TABLE deals    ADD COLUMN subject_id bigint UNIQUE REFERENCES subjects(id);
ALTER TABLE contacts ADD COLUMN subject_id bigint UNIQUE REFERENCES subjects(id);
-- backfill…

CREATE TABLE activities_v2 (
  id         bigint GENERATED ALWAYS AS IDENTITY,
  subject_id bigint NOT NULL REFERENCES subjects(id) ON DELETE CASCADE,  -- ★ ONE FK
  verb       text   NOT NULL,
  actor_id   bigint NOT NULL REFERENCES users(id),
  created_at timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (created_at, id)
) PARTITION BY RANGE (created_at);
```
```
 ✓ one real FK, real cascade
 ✓ the activity feed is ONE join, not an 11-way LEFT JOIN
 ✓ dangling references become impossible
 ✗ the 5.3M existing dangling rows are dropped (audited and logged first)
```

**Step 6 — results.**

| | Before | After |
|---|---|---|
| Total size | 1.4 TB | **840 GB** |
| Contact list query | 1,204 ms | **41 ms** |
| Custom-field filter | 4,102 ms | **18 ms** |
| Activity feed | 890 ms | **22 ms** |
| Dangling references | 5.3M | **0, impossible** |
| Queries needing `deleted_at IS NULL` | 340 | **0** |

---

## Common mistakes

**1. Reaching for EAV before trying JSONB.**
- *Symptom:* N-way self-joins; a 40-line query to render one page.
- *Fix:* JSONB + GIN for the tail, real columns for the head, generated columns for what you filter on. EAV only when the attribute set is genuinely user-defined and unbounded.

**2. Soft deletes without a partial unique index.**
- *Symptom:* a deleted user can never re-register with the same email.
- *Fix:* `CREATE UNIQUE INDEX … WHERE deleted_at IS NULL`.

**3. Soft deletes without a view or RLS.**
- *Symptom:* deleted rows appear in one report because someone forgot the predicate.
- *Fix:* a view over a renamed table, or `ROW LEVEL SECURITY` with `FORCE`.

**4. Polymorphic FKs "because there might be more types later."**
- *Symptom:* millions of dangling references; every join is a UNION.
- *Fix:* count the types. ≤5 and stable → exclusive arc. Many or open → shared supertype.

**5. A status column with no timestamps.**
- *Symptom:* "when did this ship?" is unanswerable; and at scale, indexing it destroys HOT updates.
- *Fix:* a timestamp per state + a generated status, or a transition table.

**6. Comma-separated values.**
- *Symptom:* `LIKE '%sale%'` matches `flash-sale-2024`.
- *Fix:* a junction table, or `text[]` + GIN. Never a delimited string.

**7. Denormalising before measuring.**
- *Symptom:* duplicated data, sync bugs, and the original query was never profiled.
- *Fix:* Topic 19's numbers. A hash join over 8M rows is 400 ms. Measure before you copy.

**8. `updated_at` without a trigger.**
- *Symptom:* the column silently lies — it only updates when the application remembers.
- *Fix:* a `BEFORE UPDATE` trigger. Otherwise don't have the column.

---

## Hands-on proof

**PROVE IT #1 — EAV vs JSONB vs columns.** (Example 1, step 1.)
**PROVE IT #2 — the hybrid with generated columns.** (Example 1, step 2.)
**PROVE IT #3 — soft delete reserves the unique value.** (Example 1, step 3.)
**PROVE IT #4 — soft delete diverges from FK integrity.** (Example 1, step 3.)
**PROVE IT #5 — RLS makes the predicate unforgettable.** (Example 1, step 4.)
**PROVE IT #6 — polymorphic FKs cannot be constrained.** (Example 1, step 5.)

**PROVE IT #7 — audit your own schema for these shapes.**
```sql
-- ① EAV-shaped tables
SELECT c.relname FROM pg_class c JOIN pg_attribute a ON a.attrelid=c.oid
WHERE c.relkind='r' AND a.attname ~ '^(attribute|field|property|meta)_?(name|key)$';

-- ② soft-deleted tables, and how dead they are
SELECT c.relname FROM pg_class c JOIN pg_attribute a ON a.attrelid=c.oid
WHERE c.relkind='r' AND a.attname IN ('deleted_at','is_deleted','deleted');

-- ③ polymorphic FKs
SELECT c.relname, a.attname FROM pg_class c JOIN pg_attribute a ON a.attrelid=c.oid
WHERE c.relkind='r' AND a.attname ~ '_(type|kind)$'
  AND EXISTS (SELECT 1 FROM pg_attribute a2 WHERE a2.attrelid=c.oid
              AND a2.attname = replace(a.attname,'_type','_id'));

-- ④ delimited-string columns
SELECT c.relname, a.attname FROM pg_class c JOIN pg_attribute a ON a.attrelid=c.oid
JOIN pg_type t ON t.oid=a.atttypid
WHERE c.relkind='r' AND t.typname='text' AND a.attname ~ '(tags|categories|ids|list)$';
```

---

## The design decision framework

```
BEFORE ADOPTING ANY "FLEXIBLE" DESIGN, ANSWER:
  1. What can the database still enforce?
  2. What will enforce the rules I just gave up?
  3. What does the most common query look like afterwards?
  4. What does adding a new field/type/state actually cost today?
  ⇒ If (4) is "a migration we run monthly anyway", the flexibility is
    not worth the guarantees. Most of the time, it isn't.

VARYING ATTRIBUTES:
  ≤ 20 attributes, stable                 → real columns
  many, some universal, a long tail       → ★ hybrid: columns + jsonb
  many, per-category, categories are few  → class-table inheritance
  genuinely unbounded and user-defined    → EAV, WITH a proper
                                             attribute-definition catalogue
  ★ ALWAYS: generated columns + btree for the 2–3 you filter on most

DELETION:
  Is there a product feature that undeletes?
    NO  → hard delete, or an archive table
    YES → soft delete, PLUS a partial unique index, PLUS a view or RLS
  Is the law requiring retention?
    → archive table, or anonymise-in-place (Topic 23)
  ⇒ and audit the ratio: >20% soft-deleted means you chose wrong.

POLYMORPHISM:
  ≤ 5 target types, changing rarely       → exclusive arc + CHECK
  many or open-ended                      → shared supertype
  volumes differ by 100×                  → separate tables
  ⇒ if you keep polymorphic, you OWE a reconciliation job (Topic 23)

STATE:
  Linear, non-repeating lifecycle         → a timestamp per state
                                             + generated status
  Branching or repeating                  → transition table
                                             + denormalised current status
  State has a duration                    → validity ranges (Topic 26)

THE SIGNAL TO LOOK FOR:
      Run the four audit queries in PROVE IT #7.
  For each hit, ask the four questions above. Then measure:
      • EAV        → time the N-way self-join for your real filter
      • soft delete→ SELECT count(*) FILTER (WHERE deleted_at IS NOT NULL)
                     / count(*).  >20% is a problem.
      • polymorphic→ count the dangling references. Any non-zero count
                     is a bug that already shipped.
      • CSV column → there is no measurement. It is simply wrong.
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
Build the three product designs (EAV, JSONB, columns) with 500k rows and 4 attributes. Measure total storage and the time to filter on 1, 2, and 3 attributes for each. Plot query time against the number of filtered attributes and explain the shape of each curve.

### Exercise 2 — medium (apply it)
You have `documents` with a `deleted_at` column, 40% deleted, 8 unique constraints, and 6 tables with FKs to it.
(a) List every problem the soft delete creates, including two that are invisible in normal operation.
(b) Design the archive migration, including how you handle each FK.
(c) Show the RLS policy that would have prevented the "forgot the predicate" class of bug, and prove it applies to the table owner.
(d) State the exact condition under which keeping the soft delete would be correct.

### Exercise 3 — hard (production simulation)
A 1.4 TB CRM has: an EAV table of 840M rows with 47 distinct keys (3 of which appear on >92% of entities), 14 soft-deleted tables (3 over 50% dead), a polymorphic `activities` table of 2.1B rows across 11 subject types with 5.3M dangling references, and a generic `settings` table.

(a) Write the audit queries that quantify each problem.
(b) Rank the four by (business risk × migration cost) and justify the ordering.
(c) For the EAV: show the analysis that identifies which keys become columns, and give the migration.
(d) For the soft deletes: give the batched archive migration, and explain how you handle FKs from other tables and why `VACUUM FULL` is needed afterwards.
(e) For the polymorphic table: explain why an exclusive arc is impractical at 11 types, design the shared-supertype alternative, and say what happens to the 5.3M dangling rows.
(f) One of the four is genuinely fine as-is. Identify it and defend leaving it.
(g) Estimate the total storage and query-latency improvement, and give the deployment order with the reasoning.

---

## Mental model checkpoint

1. Name the four root causes of these antipatterns. Which one produces EAV?
2. Why does filtering on N attributes in EAV require N−1 self-joins? What's the mechanism?
3. Name five distinct costs of a soft-delete column. Which two are invisible in normal operation?
4. Why can't a polymorphic foreign key be constrained? Name two alternatives and when each applies.
5. What is an exclusive arc, and what enforces "exactly one"?
6. Give three designs for status-with-history and say when each applies.
7. For any "flexible" design, what four questions must you answer first?

---

## Quick reference card

**Antipatterns**

| Shape | Cost | Instead |
|---|---|---|
| EAV | N−1 self-joins, no types, no constraints | JSONB + columns + generated columns |
| Soft delete everywhere | table never shrinks; predicate everywhere; FK divergence | archive table, or + partial unique + RLS |
| Polymorphic FK | no FK, no cascade, N-way joins | exclusive arc, or shared supertype |
| Status, no history | "when?" unanswerable; breaks HOT | timestamp per state + generated status |
| CSV column | no index, no integrity, false matches | junction table, or `text[]` + GIN |
| Generic `data` table | every cost, no benefit | model it |

**Patterns**

| Shape | Why |
|---|---|
| `created_at`/`updated_at` + trigger | answers a hundred future questions |
| Lookup table when the list has attributes | non-engineers can edit it |
| Outbox | the only atomic DB+queue write |
| Idempotency key + UNIQUE | on every externally-triggered write |
| Snapshot + reference | two questions, two mechanisms |
| Hot/cold vertical split | 20× more rows per page on the hot table |

**The test:** *what can the database still enforce, and what enforces the rest?*

---

## When would I use this at work?

1. **A design review where "we need flexible custom fields" comes up.** You can name EAV, show its N-way-join cost, and propose the hybrid in five minutes — saving eighteen months of pain that the team would otherwise have to live through.

2. **Inheriting a legacy schema.** The four audit queries find every instance of these shapes in about a minute, and the risk×cost ranking gives you a defensible remediation order for a planning meeting.

3. **Reviewing a migration that adds `deleted_at`.** Two questions — "what product feature undeletes?" and "where's the partial unique index?" — either justify the choice or redirect it to an archive table, before it's in 340 queries.

---

## Connected topics

**Understand before this:** 20–26 (everything these patterns are built from).

**This unlocks:**
- **28** — migrating away from these shapes safely
- **29–38** — normalisation, which is the *theory* explaining why most of these are wrong
- **53–55** — deliberate denormalisation, the disciplined version of the "fear of joins" antipattern
- **69** — RLS and PII, which appear here as soft-delete mitigations
- **Case studies 10, 15, 17** — catalogue modelling, multi-tenancy, and temporal records in full
