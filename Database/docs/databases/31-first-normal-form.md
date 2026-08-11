# 31 — First Normal Form (1NF)
## Phase: Normalisation

---

## ELI5 — The Simple Analogy

A form at a government office with a box labelled **"Phone numbers."**

Someone writes: `98765-43210, 91234-56789, 90000-11111`.

The form is now unusable for its purpose. The clerk cannot sort by phone number. She cannot search for a specific one without reading every form and squinting. She cannot say "this number belongs to exactly one person," because it's buried in a sentence. And when the citizen wants to remove the middle number, she has to rewrite the whole box and hope she doesn't fumble a comma.

The fix is not clever. It is: **one box, one value.** If a person can have three phone numbers, the form needs three lines — or better, a separate sheet of phone numbers with the person's reference on each.

That's 1NF. It is the least mathematical of the normal forms and the one most often violated in real code, because the violation always looks convenient at the moment you write it.

---

## Where this fits in the big picture

```
   29 the anomalies · 30 functional dependencies
                          │
                          ▼
        ┌──────────────────────────────────────────┐
        │ 31 FIRST NORMAL FORM     ← YOU ARE HERE  │
        │ one value per cell; no repeating groups  │
        └────────────────────┬─────────────────────┘
                             ▼
                    32 2NF · 33 3NF · 34 BCNF · 35 4NF · 36 5NF
                    (each ASSUMES 1NF — they are all about
                     which FDs may share a table, which only
                     makes sense once cells hold single values)
```

**1NF is the precondition for everything else.** The FD machinery of Topic 30 assumes a value is a value. If a cell holds a list, "X → Y" is meaningless.

---

## What is this?

A relation is in **first normal form** when:

1. Every attribute value is **atomic** — a single value from its domain, not a collection.
2. There are **no repeating groups** — no `phone1, phone2, phone3`.
3. Every row is **uniquely identifiable** (there is a key).
4. There is no significance to **row or column order**.

In practice, points 1 and 2 are what you check. Points 3 and 4 are automatic in any real SQL table.

---

## Why does it matter for a backend developer?

Because a 1NF violation destroys four capabilities at once, and every one of them is something you will need:

```
 tags text = 'ethnic,sale,bestseller'

 ① NO INDEX CAN SERVE IT
    WHERE tags LIKE '%sale%'  → leading wildcard → no B-tree (Topic 16)
    ⇒ full table scan, every time, forever.

 ② FALSE MATCHES
    '%sale%' also matches 'flash-sale-2024', 'wholesale', 'resale'.
    ⇒ silently wrong results.

 ③ NO REFERENTIAL INTEGRITY
    You cannot FK a substring to a `tags` table. Typos are permanent.

 ④ UPDATES ARE STRING SURGERY
    Removing 'sale' from 'ethnic,sale,bestseller' means split, filter,
    rejoin — in application code, non-atomically, per row.
```

And the repeating-group form has its own distinct failure:

```
 phone1, phone2, phone3
 ⑤ A FOURTH PHONE NUMBER NEEDS A MIGRATION.
 ⑥ "find everyone with this number" needs 3 OR clauses, and 4 tomorrow.
 ⑦ Most rows have NULLs in phone2/phone3, and NULL means "no more"
   rather than "unknown" — two different things, one representation.
```

---

## The physical reality

### What each form costs

```
 ✗ DELIMITED STRING
   products(id, tags text)          'ethnic,sale,bestseller'  ~24 B
   ┌──────────────────────────────────────────────────────────────┐
   │ ✗ no index possible for containment                          │
   │ ✗ WHERE tags LIKE '%sale%' → Seq Scan, always                 │
   │ ✗ false matches on substrings                                 │
   │ ✗ no FK, no CHECK on individual values                        │
   └──────────────────────────────────────────────────────────────┘

 ✗ REPEATING GROUP
   customers(id, phone1, phone2, phone3)      24 B + 3 NULL bits
   ┌──────────────────────────────────────────────────────────────┐
   │ ✗ a 4th value needs DDL                                       │
   │ ✗ "which customer has 98765-43210?" needs 3 OR clauses and    │
   │   3 separate indexes (or one useless composite)               │
   │ ✗ ordering is meaningless but implied                         │
   │ ✗ mostly NULL — and NULL is overloaded                        │
   └──────────────────────────────────────────────────────────────┘

 ~ ARRAY  (see the nuance below)
   products(id, tags text[])        24 B header + elements
   ┌──────────────────────────────────────────────────────────────┐
   │ ✓ GIN-indexable for containment: tags @> ARRAY['sale']        │
   │ ✓ atomic AS A VALUE if you only ever read/write the whole set │
   │ ✗ no FK to a tags table                                       │
   │ ✗ no per-element CHECK                                        │
   │ ✗ updating one element rewrites the whole row (Topic 05)      │
   └──────────────────────────────────────────────────────────────┘

 ✓ JUNCTION TABLE
   product_tags(product_id, tag_id)  PK (product_id, tag_id)
   ┌──────────────────────────────────────────────────────────────┐
   │ ✓ B-tree index both directions                                │
   │ ✓ FK to tags(id) — no typos, ever                             │
   │ ✓ add/remove one tag = one row                                │
   │ ✓ tags can have attributes (colour, sort order, i18n)         │
   │ ✗ a join to read                                              │
   └──────────────────────────────────────────────────────────────┘
```

### The measured difference

```
 8,000,000 products, average 3 tags each

 CSV column       'ethnic,sale,bestseller'
   storage: 8M × 24 B = 192 MB
   WHERE tags LIKE '%sale%'  → Seq Scan, 190,000 pages, 3,900 ms

 text[] + GIN
   storage: 8M × 48 B = 384 MB + 180 MB index
   WHERE tags @> ARRAY['sale']  → Bitmap scan, 88 ms

 junction table (24M rows)
   storage: 24M × 28 B = 672 MB + 2 indexes
   WHERE t.code='sale'  → Index scan, 12 ms
   ⇒ 325× faster than CSV, and typos are impossible
```

---

## How it works — step by step

### The three violation shapes

```
 ① MULTI-VALUE IN ONE CELL
    tags = 'ethnic,sale,bestseller'
    skills = 'java;python;sql'
    ⇒ the cell holds a SET, not a value.

 ② REPEATING GROUP ACROSS COLUMNS
    phone1, phone2, phone3
    line_item_1_sku, line_item_1_qty, line_item_2_sku, line_item_2_qty
    ⇒ the SAME attribute, numbered. The number is data pretending
      to be schema.

 ③ COMPOSITE VALUE YOU QUERY THE PARTS OF
    full_address = 'Flat 4, MG Road, Bengaluru, 560001'
    ⇒ atomic only if you NEVER query by city or pin.
      The moment you write WHERE address LIKE '%Bengaluru%',
      it is a 1NF violation.
```

### The fix, mechanically

```
 ① and ② → the multi-valued attribute becomes ITS OWN TABLE.
   This is exactly Topic 20's Q5/multi-valued rule and Topic 21's Rule 7.

   customers(id, phone1, phone2, phone3)
        ↓
   customers(id, name, …)
   customer_phones(customer_id, phone, kind, is_primary,
                   PRIMARY KEY (customer_id, phone))

 ③ → decompose ONLY the parts you query.
   full_address → line1, line2, city, pin
   ⇒ if you never query by city, leaving it composite is fine.
     1NF is about ATOMICITY WITH RESPECT TO YOUR QUERIES.
```

### ★ The modern nuance: arrays and JSONB

This is where a purist reading of 1NF meets 2026 reality, and the honest answer is more interesting than either extreme.

```
 THE STRICT VIEW (Codd, 1970):
   a value must be atomic — indivisible in the domain.
   ⇒ text[] and jsonb both violate 1NF.

 THE PRACTICAL VIEW:
   "atomic" means "the database never needs to look INSIDE it to
    answer a query, and you never need to constrain the parts."
   ⇒ a value is atomic RELATIVE TO HOW YOU USE IT.

 ★ THE TEST THAT ACTUALLY DECIDES IT — four questions:
   ① Do you ever query INDIVIDUAL elements?
   ② Do you ever UPDATE individual elements?
   ③ Do the elements need REFERENTIAL INTEGRITY (an FK)?
   ④ Do the elements have ATTRIBUTES of their own?
   ⇒ ANY yes → it is a separate table. No exceptions.
   ⇒ ALL no  → an array or JSONB is legitimately atomic. Keep it.

 EXAMPLES, decided by the test:
   tags text[] you filter with @>              → ① yes → junction table
                                                  (unless the tags are
                                                   opaque and unmanaged)
   embedding vector(1536) for similarity        → all no → ✓ ATOMIC
   coordinates point                            → all no → ✓ ATOMIC
   raw webhook payload jsonb, stored for audit  → all no → ✓ ATOMIC
   order line items jsonb                       → ①②③④ all yes → ✗ TABLE
   permissions text[] checked with @>           → borderline: ① yes but
                                                  ③ ④ no. An array + GIN
                                                  is defensible IF the
                                                  permission list is
                                                  code-managed.

 ⇒ ★ 1NF's SPIRIT is "the database can see and constrain your data."
   An array of opaque, code-managed, always-whole values does not
   violate that spirit. An array you query, join, and constrain does.
```

---

## Concept breakdown

```
FIRST NORMAL FORM
│  └── ① atomic values  ② no repeating groups  ③ a key  ④ no order
│      significance. In practice: ① and ②.
│
├── ATOMIC — relative to your queries.
│     "does the database need to look inside it?"
│     NO  → atomic. YES → not atomic.
│
├── MULTI-VALUED ATTRIBUTE → its own table. Always. (T20 Q5, T21 Rule 7)
│
└── REPEATING GROUP → the numbering is DATA masquerading as SCHEMA.
      phone1..3 · item_1_sku..item_5_sku · jan_total..dec_total

THE FOUR THINGS A 1NF VIOLATION DESTROYS
├── indexability      no B-tree can serve a substring or an OR-chain
├── correctness       '%sale%' matches 'wholesale'
├── integrity         no FK, no CHECK on individual values
└── updatability      string surgery, or a migration for a 4th value

THE FOUR-QUESTION TEST for arrays/JSONB
├── ① query individual elements?
├── ② update individual elements?
├── ③ need referential integrity?
└── ④ elements have their own attributes?
    ANY yes → separate table.  ALL no → atomic, keep it.

WHAT 1NF IS NOT ABOUT
├── ✗ it does not forbid composite types where you never query the parts
├── ✗ it does not forbid arrays used as opaque whole values
└── ✗ it does not say "one column per fact" — that's 2NF/3NF
```

---

## Diagrams

**Diagram 1 — big picture: the three violation shapes**

```
 ① MULTI-VALUE IN A CELL
    ┌────┬──────────────────────────┐
    │ id │ tags                     │
    ├────┼──────────────────────────┤
    │ 88 │ ethnic,sale,bestseller   │  ← a SET pretending to be a value
    └────┴──────────────────────────┘

 ② REPEATING GROUP
    ┌────┬────────┬────────┬────────┐
    │ id │ phone1 │ phone2 │ phone3 │  ← the NUMBER is data
    ├────┼────────┼────────┼────────┤
    │  7 │ 98765… │ 91234… │ NULL   │
    └────┴────────┴────────┴────────┘

 ③ COMPOSITE YOU QUERY THE PARTS OF
    ┌────┬─────────────────────────────────────┐
    │ id │ address                             │
    ├────┼─────────────────────────────────────┤
    │  7 │ Flat 4, MG Road, Bengaluru, 560001  │
    └────┴─────────────────────────────────────┘
          ↑ atomic UNTIL you write WHERE address LIKE '%Bengaluru%'

 ✓ ALL THREE FIX THE SAME WAY: the repeating thing becomes rows.
    ┌────┬────────┐        ┌────────────┬────────┬────────┐
    │ id │ name   │        │ customer_id│ phone  │ kind   │
    ├────┼────────┤        ├────────────┼────────┼────────┤
    │  7 │ Arjun  │        │          7 │ 98765… │ mobile │
    └────┴────────┘        │          7 │ 91234… │ work   │
                           └────────────┴────────┴────────┘
```

**Diagram 2 — data flow: finding a value**

```
  CSV COLUMN                          JUNCTION TABLE
  ──────────────────────────          ──────────────────────────────
   WHERE tags LIKE '%sale%'            WHERE t.code = 'sale'
        │                                    │
   ✗ leading wildcard                   ✓ index on tags(code)
   ✗ no index usable                         │
        │                                    ▼
        ▼                              index scan → tag_id
   SEQ SCAN 190,000 pages                    │
        │                                    ▼
   read EVERY row                      index scan product_tags(tag_id)
        │                                    │
   substring match                           ▼
        │                              12 matching product_ids
   ⚠ also matches 'wholesale',              │
     'flash-sale-2024', 'resale'            ▼
        │                              index scan products
        ▼
   3,900 ms, WRONG RESULTS             12 ms, CORRECT RESULTS
```

**Diagram 3 — before/after: the repeating group**

```
 BEFORE
 customers(id, name, phone1, phone2, phone3)
 ┌──────────────────────────────────────────────────────────────────┐
 │ "find the customer with 98765-43210"                             │
 │   WHERE phone1='98765-43210' OR phone2='98765-43210'             │
 │      OR phone3='98765-43210'                                     │
 │   ⇒ 3 index probes, or one BitmapOr, or a Seq Scan               │
 │                                                                  │
 │ "a customer needs a 4th number"     → ★ ALTER TABLE. A migration.│
 │ "which number is primary?"          → by convention: phone1.     │
 │                                        Unenforceable.            │
 │ "no two customers share a number"   → ★ impossible to express    │
 └──────────────────────────────────────────────────────────────────┘

 AFTER
 customers(id, name)
 customer_phones(customer_id, phone, kind, is_primary, PK(customer_id,phone))
 ┌──────────────────────────────────────────────────────────────────┐
 │ "find the customer with 98765-43210"                             │
 │   WHERE phone='98765-43210'   ⇒ ONE index probe                  │
 │ "a 4th number"                → INSERT. No DDL.                  │
 │ "which is primary?"           → CREATE UNIQUE INDEX …            │
 │                                   ON customer_phones (customer_id)│
 │                                   WHERE is_primary;   ★ enforced │
 │ "no two customers share one"  → UNIQUE (phone)        ★ enforced │
 └──────────────────────────────────────────────────────────────────┘
```

---

## Example 1 — basic

**Step 1 — build all four designs.**

```sql
-- ✗ CSV
CREATE TABLE p_csv (id bigint PRIMARY KEY, name text, tags text);
-- ✗ REPEATING GROUP
CREATE TABLE p_rep (id bigint PRIMARY KEY, name text,
                    tag1 text, tag2 text, tag3 text);
-- ~ ARRAY
CREATE TABLE p_arr (id bigint PRIMARY KEY, name text, tags text[]);
-- ✓ JUNCTION
CREATE TABLE tags (id smallint PRIMARY KEY, code text NOT NULL UNIQUE);
CREATE TABLE p_norm (id bigint PRIMARY KEY, name text);
CREATE TABLE p_tags (product_id bigint NOT NULL REFERENCES p_norm(id) ON DELETE CASCADE,
                     tag_id smallint NOT NULL REFERENCES tags(id) ON DELETE RESTRICT,
                     PRIMARY KEY (product_id, tag_id));

INSERT INTO tags VALUES (1,'ethnic'),(2,'casual'),(3,'formal'),
                        (4,'sale'),(5,'new'),(6,'bestseller');

-- 2M products, 3 tags each
INSERT INTO p_csv SELECT i,'Product '||i,
  (ARRAY['ethnic','casual','formal'])[1+(i%3)]||','||
  (ARRAY['sale','new','bestseller'])[1+(i%3)]
FROM generate_series(1,2000000) i;
INSERT INTO p_rep SELECT i,'Product '||i,
  (ARRAY['ethnic','casual','formal'])[1+(i%3)],
  (ARRAY['sale','new','bestseller'])[1+(i%3)], NULL
FROM generate_series(1,2000000) i;
INSERT INTO p_arr SELECT i,'Product '||i,
  ARRAY[(ARRAY['ethnic','casual','formal'])[1+(i%3)],
        (ARRAY['sale','new','bestseller'])[1+(i%3)]]
FROM generate_series(1,2000000) i;
INSERT INTO p_norm SELECT i,'Product '||i FROM generate_series(1,2000000) i;
INSERT INTO p_tags SELECT i, 1+(i%3) FROM generate_series(1,2000000) i;
INSERT INTO p_tags SELECT i, 4+(i%3) FROM generate_series(1,2000000) i;

CREATE INDEX ON p_arr USING gin (tags);
CREATE INDEX ON p_tags (tag_id, product_id);
VACUUM ANALYZE p_csv; VACUUM ANALYZE p_arr; VACUUM ANALYZE p_tags;
```

**Step 2 — "find products tagged 'sale'".**

```sql
EXPLAIN (ANALYZE,BUFFERS) SELECT count(*) FROM p_csv WHERE tags LIKE '%sale%';
```
```
Seq Scan on p_csv  (actual time=0.4..1204.8 rows=666667 loops=1)
  Filter: (tags ~~ '%sale%'::text)
  Buffers: shared hit=2104 read=16204
Execution Time: 1206.1 ms
```
```sql
EXPLAIN (ANALYZE,BUFFERS) SELECT count(*) FROM p_arr WHERE tags @> ARRAY['sale'];
-- Bitmap Heap Scan … Execution Time: 84.2 ms

EXPLAIN (ANALYZE,BUFFERS) SELECT count(*) FROM p_tags pt
JOIN tags t ON t.id=pt.tag_id WHERE t.code='sale';
-- Index Only Scan … Execution Time: 11.4 ms
```
**1,206 ms → 84 ms → 11 ms.**

**Step 3 — the false-match bug, which is worse than the slowness.**

```sql
INSERT INTO p_csv VALUES (9000001,'Wholesale Pack','wholesale,bulk');
INSERT INTO p_csv VALUES (9000002,'Flash Sale Item','flash-sale-2024,new');

SELECT id, name, tags FROM p_csv WHERE tags LIKE '%sale%' AND id > 9000000;
```
```
    id     |      name       |      tags
-----------+-----------------+------------------
   9000001 | Wholesale Pack  | wholesale,bulk
   9000002 | Flash Sale Item | flash-sale-2024,new
```
**Neither is tagged 'sale'.** Both match. Every "products on sale" report has been wrong, silently, for the life of the system.

```sql
-- the "fix" people write:
SELECT count(*) FROM p_csv
WHERE ','||tags||',' LIKE '%,sale,%';       -- ✓ correct… and still a Seq Scan
```

**Step 4 — the repeating group's failures.**

```sql
-- "which product has tag 'sale'?"
EXPLAIN (ANALYZE) SELECT count(*) FROM p_rep
WHERE tag1='sale' OR tag2='sale' OR tag3='sale';
-- needs 3 indexes, or a BitmapOr, or a Seq Scan.

-- a 4th tag:
UPDATE p_rep SET tag4='premium' WHERE id=1;
-- ERROR: column "tag4" of relation "p_rep" does not exist
--   ★ data requires a MIGRATION

-- "no product may have the same tag twice"
--   ★ inexpressible. tag1='sale' AND tag2='sale' is a legal row.
INSERT INTO p_rep VALUES (9000003,'Dup','sale','sale','sale');   -- ✓ accepted
```

**Step 5 — what the junction table makes possible.**

```sql
-- referential integrity: typos are impossible
INSERT INTO p_tags VALUES (1, 99);
-- ERROR: violates foreign key constraint "p_tags_tag_id_fkey"

-- no duplicate tag per product: the PK handles it
INSERT INTO p_tags VALUES (1, 1);
-- ERROR: duplicate key value violates unique constraint "p_tags_pkey"

-- tags can have attributes now
ALTER TABLE tags ADD COLUMN display_name text, ADD COLUMN colour char(7),
                 ADD COLUMN sort_order smallint;

-- and the reverse query is equally fast
EXPLAIN (ANALYZE) SELECT t.code FROM p_tags pt JOIN tags t ON t.id=pt.tag_id
WHERE pt.product_id = 4471;
-- Index Scan, 0.04 ms
```

**Step 6 — the four-question test, applied.**

```sql
-- CASE A: an embedding vector for similarity search
CREATE TABLE docs (id bigint PRIMARY KEY, body text, embedding real[]);
--  ① query individual elements?  NO — you compute a distance over the whole
--  ② update individual elements? NO — recomputed wholesale
--  ③ referential integrity?      NO
--  ④ elements have attributes?   NO
--  ⇒ ★ ATOMIC. An array is correct here. (A vector type is better still.)

-- CASE B: order line items as JSONB
CREATE TABLE bad_orders (id bigint PRIMARY KEY, items jsonb);
--  ① query individual items?     YES ("how many of SKU 88 did we sell?")
--  ② update individual items?    YES (refund one line)
--  ③ referential integrity?      YES (product_id must exist)
--  ④ attributes of their own?    YES (qty, price, status)
--  ⇒ ★ FOUR YESES. This is a table. (Topics 20–21.)
```

---

## Example 2 — production scenario

**The situation.** A recruitment platform, 4 years old. `candidates` has 12 million rows.

```sql
CREATE TABLE candidates (
  id bigserial PRIMARY KEY,
  name text NOT NULL,
  email text NOT NULL,
  phone1 text, phone2 text,
  skills text,                    -- 'java,python,sql,aws'
  languages text,                 -- 'english:fluent;hindi:native'
  education text,                 -- 'BTech CSE, IIT-M, 2019 | MTech, IISc, 2021'
  preferred_locations text,       -- 'Bengaluru,Hyderabad,Remote'
  salary_expectation text         -- '25-35 LPA'
);
```

**Six 1NF violations in one table.** Each one is blocking a feature.

**Step 1 — what the business cannot do.**

| Requirement | Blocked by |
|---|---|
| "Candidates who know Python **and** AWS" | `skills` is a CSV — `LIKE '%python%'` also matches `python3`, and AND needs two scans |
| "Rank by years of experience per skill" | skills have no attributes |
| "Candidates fluent in Hindi" | `languages` is a doubly-delimited string |
| "Candidates from IIT" | `education` is free text with `|` and `,` |
| "Open to Bengaluru" | `preferred_locations` CSV; `'Bengaluru'` vs `'bengaluru'` vs `'Bangalore'` |
| "Salary between 30 and 40 LPA" | `salary_expectation` is text — no range query at all |

**Step 2 — quantify the damage.**

```sql
-- how many distinct skill strings exist? (typos and casing)
SELECT count(DISTINCT trim(lower(s))) AS distinct_skills
FROM candidates, unnest(string_to_array(skills, ',')) s;
```
```
 distinct_skills
-----------------
           41208        ★ for what should be ~800 real skills
```
```sql
SELECT trim(lower(s)) AS skill, count(*) FROM candidates,
       unnest(string_to_array(skills,',')) s
WHERE trim(lower(s)) LIKE '%python%' GROUP BY 1 ORDER BY 2 DESC LIMIT 8;
```
```
     skill      | count
----------------+--------
 python         | 412008
 python3        |  18204
 python 3       |  12402
 phyton         |   8104     ← typo
 python(django) |   4102
 Python         |   2104     ← casing survived a partial lower()
 python2        |   1204
 py             |    882
```
**41,208 "distinct skills" for ~800 real ones.** Every skill-based search misses candidates. The recruiters' workaround is to search four spellings manually.

```sql
-- and the search cost
EXPLAIN (ANALYZE) SELECT count(*) FROM candidates
WHERE skills LIKE '%python%' AND skills LIKE '%aws%';
```
```
Seq Scan on candidates (actual time=0.4..8412.1 rows=41204 loops=1)
Execution Time: 8413.8 ms      ★ 8.4 seconds per search, 200 searches/min
```

**Step 3 — the normalised design.**

```sql
CREATE TABLE skills (
  id smallint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  code text NOT NULL UNIQUE,          -- canonical: 'python'
  display_name text NOT NULL,
  category text NOT NULL
);
CREATE TABLE skill_aliases (          -- ★ handles the 41,208 variants
  alias text PRIMARY KEY,             -- 'python3','phyton','py'
  skill_id smallint NOT NULL REFERENCES skills(id)
);

CREATE TABLE candidate_skills (
  candidate_id bigint NOT NULL REFERENCES candidates(id) ON DELETE CASCADE,
  skill_id smallint NOT NULL REFERENCES skills(id) ON DELETE RESTRICT,
  years_experience smallint NULL CHECK (years_experience BETWEEN 0 AND 60),
  proficiency smallint NULL CHECK (proficiency BETWEEN 1 AND 5),
  PRIMARY KEY (candidate_id, skill_id)      -- ★ no duplicate skills
);
CREATE INDEX ON candidate_skills (skill_id, candidate_id);   -- reverse direction

CREATE TABLE languages (id smallint PRIMARY KEY, code char(2) UNIQUE, name text);
CREATE TABLE candidate_languages (
  candidate_id bigint NOT NULL REFERENCES candidates(id) ON DELETE CASCADE,
  language_id smallint NOT NULL REFERENCES languages(id),
  proficiency text NOT NULL CHECK (proficiency IN ('basic','conversational','fluent','native')),
  PRIMARY KEY (candidate_id, language_id)
);

CREATE TABLE candidate_education (
  candidate_id bigint NOT NULL REFERENCES candidates(id) ON DELETE CASCADE,
  seq smallint NOT NULL,
  degree text NOT NULL, field text NOT NULL,
  institution_id bigint NOT NULL REFERENCES institutions(id),
  graduated_year smallint NOT NULL CHECK (graduated_year BETWEEN 1950 AND 2100),
  PRIMARY KEY (candidate_id, seq)
);

CREATE TABLE candidate_phones (
  candidate_id bigint NOT NULL REFERENCES candidates(id) ON DELETE CASCADE,
  phone text NOT NULL,
  kind text NOT NULL CHECK (kind IN ('mobile','home','work')),
  is_primary boolean NOT NULL DEFAULT false,
  PRIMARY KEY (candidate_id, phone)
);
CREATE UNIQUE INDEX ON candidate_phones (candidate_id) WHERE is_primary;  -- ★ T13

CREATE TABLE candidate_locations (
  candidate_id bigint NOT NULL REFERENCES candidates(id) ON DELETE CASCADE,
  location_id bigint NOT NULL REFERENCES locations(id),
  PRIMARY KEY (candidate_id, location_id)
);

ALTER TABLE candidates
  ADD COLUMN salary_min_lpa numeric(5,1) NULL,     -- ★ now a RANGE query works
  ADD COLUMN salary_max_lpa numeric(5,1) NULL,
  ADD CONSTRAINT ck_salary CHECK (salary_max_lpa IS NULL OR salary_min_lpa IS NULL
                                  OR salary_max_lpa >= salary_min_lpa);
```

**Step 4 — the migration, with the hard part named honestly.**

```sql
-- ① extract and canonicalise the skills — a HUMAN reviews the alias map
CREATE TEMP TABLE raw_skills AS
SELECT DISTINCT trim(lower(s)) AS raw FROM candidates,
       unnest(string_to_array(skills,',')) s
WHERE trim(s) <> '';
-- 41,208 rows. A data analyst + fuzzy matching (pg_trgm similarity)
-- reduces this to ~800 canonical skills + an alias map.
SELECT a.raw, b.raw, similarity(a.raw, b.raw) AS sim
FROM raw_skills a, raw_skills b
WHERE a.raw < b.raw AND similarity(a.raw, b.raw) > 0.6
ORDER BY sim DESC LIMIT 50;
-- ★ THIS STEP CANNOT BE AUTOMATED FULLY. 'python2' and 'python3' are
--   0.9 similar and are DIFFERENT skills. Human judgement required.

-- ② explode into the junction table
INSERT INTO candidate_skills (candidate_id, skill_id)
SELECT DISTINCT c.id, COALESCE(al.skill_id, sk.id)
FROM candidates c, unnest(string_to_array(c.skills,',')) s
LEFT JOIN skill_aliases al ON al.alias = trim(lower(s))
LEFT JOIN skills sk ON sk.code = trim(lower(s))
WHERE COALESCE(al.skill_id, sk.id) IS NOT NULL
ON CONFLICT DO NOTHING;

-- ③ salary: parse '25-35 LPA'
UPDATE candidates SET
  salary_min_lpa = (regexp_match(salary_expectation, '(\d+(?:\.\d+)?)'))[1]::numeric,
  salary_max_lpa = (regexp_match(salary_expectation, '-\s*(\d+(?:\.\d+)?)'))[1]::numeric
WHERE salary_expectation ~ '\d';
-- ⚠ 8% of rows don't match any pattern ('negotiable', 'as per market').
--   Those become NULL and are flagged for manual review. DOCUMENT IT.
```

**Step 5 — results.**

```sql
EXPLAIN (ANALYZE) SELECT count(DISTINCT cs1.candidate_id)
FROM candidate_skills cs1
JOIN skills s1 ON s1.id=cs1.skill_id AND s1.code='python'
JOIN candidate_skills cs2 ON cs2.candidate_id=cs1.candidate_id
JOIN skills s2 ON s2.id=cs2.skill_id AND s2.code='aws';
-- Execution Time: 42 ms       ★ was 8,414 ms
```

| | Before | After |
|---|---|---|
| "Python AND AWS" search | 8,414 ms | **42 ms** |
| Distinct skill values | 41,208 | **812, canonical** |
| Candidates missed by a search | ~15% | **0** |
| "Rank by years per skill" | impossible | one column |
| "Fluent in Hindi" | impossible | one join |
| "Salary 30–40 LPA" | impossible | a range query |
| Enforceable constraints | 2 | **19** |

**Five product features that were blocked became one query each.** That is the real return on 1NF — not performance, capability.

---

## Common mistakes

**1. A delimited string because "it's just a list."**
- *Symptom:* `LIKE '%x%'` searches that are slow *and* wrong.
- *Fix:* a junction table. Or an array + GIN if the four-question test says all no.

**2. `phone1, phone2, phone3`.**
- *Symptom:* a migration when someone needs a fourth, and OR-chains everywhere.
- *Fix:* the multi-valued attribute is its own table (Topic 20's Q5).

**3. Assuming arrays and JSONB always violate 1NF.**
- *Symptom:* decomposing an embedding vector into 1,536 rows.
- *Fix:* the four-question test. An opaque, always-whole, code-managed value is atomic.

**4. Assuming arrays and JSONB never violate 1NF.**
- *Symptom:* `items jsonb` on `orders`, and no way to answer "how many of SKU 88 did we sell?"
- *Fix:* the same four questions. Four yeses means a table.

**5. Decomposing a composite you never query the parts of.**
- *Symptom:* `address_line1..5` when you only ever print the whole thing.
- *Fix:* atomicity is relative to your queries. If you never query by city, leave it.

**6. Underestimating the canonicalisation work.**
- *Symptom:* a migration plan that says "split the CSV" and takes three months.
- *Fix:* the *splitting* is trivial; the *canonicalisation* (41,208 → 812) needs fuzzy matching plus human review, and that is the actual project.

---

## Hands-on proof

**PROVE IT #1 — CSV vs array vs junction, timed.** (Example 1, step 2.)
**PROVE IT #2 — the false-match bug.** (Example 1, step 3.)
**PROVE IT #3 — the repeating group's three failures.** (Example 1, step 4.)
**PROVE IT #4 — what the junction table makes possible.** (Example 1, step 5.)

**PROVE IT #5 — find 1NF violations in your own schema.**
```sql
-- ① repeating groups: numbered column names
SELECT c.relname, string_agg(a.attname, ', ' ORDER BY a.attnum) AS numbered_cols
FROM pg_class c JOIN pg_attribute a ON a.attrelid=c.oid
WHERE c.relkind='r' AND c.relnamespace='public'::regnamespace
  AND a.attnum>0 AND NOT a.attisdropped AND a.attname ~ '\d$'
GROUP BY 1 HAVING count(*) > 1;

-- ② likely delimited strings
SELECT c.relname, a.attname FROM pg_class c JOIN pg_attribute a ON a.attrelid=c.oid
JOIN pg_type t ON t.oid=a.atttypid
WHERE c.relkind='r' AND t.typname='text'
  AND a.attname ~ '(tags|skills|categories|ids|list|codes|emails|phones)$';

-- ③ confirm: does the column actually contain delimiters?
SELECT count(*) FILTER (WHERE tags LIKE '%,%') AS comma_separated,
       count(*) AS total FROM p_csv;
```

**PROVE IT #6 — the four-question test, in SQL.**
```sql
-- ① do you query individual elements? Check pg_stat_statements:
SELECT calls, left(query,80) FROM pg_stat_statements
WHERE query ~* '(tags\s+LIKE|tags\s+@>|jsonb_array_elements|string_to_array)'
ORDER BY calls DESC;
-- ★ any hits → you query individual elements → it is not atomic
```

---

## The design decision framework

```
FOR EVERY COLUMN, ASK: DOES IT HOLD MORE THAN ONE VALUE?
  NO  → 1NF is satisfied for this column. Move on.
  YES → run the four-question test:

    ① Do you query INDIVIDUAL elements?
    ② Do you update INDIVIDUAL elements?
    ③ Do elements need REFERENTIAL INTEGRITY?
    ④ Do elements have ATTRIBUTES of their own?

    ANY YES  → ★ SEPARATE TABLE. No exceptions.
    ALL NO   → an array or JSONB is legitimately atomic. Keep it.

FOR NUMBERED COLUMNS (phone1, item_2_sku, jan_total):
  → ★ ALWAYS a separate table. The number is data, not schema.
    There is no version of this that is correct.

FOR COMPOSITE VALUES (address, full_name):
  Do you ever query or constrain the PARTS?
    NO  → leave it composite. 1NF is about YOUR queries.
    YES → decompose ONLY the parts you query.

CHOOSING BETWEEN ARRAY AND JUNCTION TABLE (when ① is the only yes):
  array + GIN when:
    ✓ elements are opaque labels with no attributes
    ✓ the vocabulary is managed in code, not by users
    ✓ you only need containment (@>), never a join
    ✓ the set is small and always read whole
  junction table when:
    ✓ elements are entities (have a name, a category, an id)
    ✓ users can create them (⇒ typos ⇒ you need canonicalisation)
    ✓ you need FK integrity
    ✓ the relationship has attributes (years_experience, added_at)
    ✓ you query in BOTH directions

THE SIGNAL TO LOOK FOR:
  Run PROVE IT #5. Then for each hit:
    SELECT count(*) FILTER (WHERE col LIKE '%,%') FROM t;
  • any delimiters present → a 1NF violation, confirmed
  • any numbered column family → a 1NF violation, no test needed
  • then check pg_stat_statements for LIKE '%…%' on that column —
    every such query is both slow and possibly WRONG.

  ⚠ AND BUDGET FOR CANONICALISATION.
    Splitting the string is an afternoon. Reducing 41,208 variants to
    812 canonical values needs fuzzy matching AND human review, and it
    is the real cost of the migration.
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
Build the four `products` designs from Example 1 with 500k rows. Measure the time for "products tagged 'sale'" in each. Then insert a product tagged `'wholesale'` and show that the CSV query returns it incorrectly. Finally, write the CSV query that *is* correct and explain why it's still a sequential scan.

### Exercise 2 — medium (apply it)
For each column, apply the four-question test and justify your answer:
(a) `roles text[]` on `users`, checked with `@> ARRAY['admin']` · (b) `embedding real[]` used only for cosine similarity · (c) `line_items jsonb` on `orders` · (d) `coordinates point` on `stores` · (e) `attachment_urls text[]`, rendered as a list, never queried individually · (f) `permissions text[]`, where permissions are created by customers.

For any that fail, design the replacement.

### Exercise 3 — hard (production simulation)
A `candidates` table with 12M rows has six 1NF violations: `phone1/phone2`, `skills` (CSV, 41,208 distinct variants for ~800 real skills), `languages` (`'english:fluent;hindi:native'`), `education` (free text with two delimiters), `preferred_locations` (CSV), and `salary_expectation` (`'25-35 LPA'`).

(a) For each, name the specific business question it makes impossible.
(b) Write the queries that quantify the damage (distinct variants, search time, candidates missed).
(c) Design the normalised schema, including how you handle the 41,208 skill variants.
(d) The canonicalisation cannot be fully automated. Explain why, give a concrete pair of strings that a similarity function would wrongly merge, and describe the human-in-the-loop process.
(e) 8% of `salary_expectation` values don't match any pattern. What do you do with them, and what do you tell the business?
(f) `education` has two nesting levels. Show the extraction and explain why it needs a `seq` column.
(g) Give the migration order using Topic 28's techniques, and the verification queries for each step.
(h) After migration, list the business capabilities that became possible, with one query each.

---

## Mental model checkpoint

1. State the four conditions of 1NF. Which two do you actually check in practice?
2. Name the three shapes a 1NF violation takes.
3. Give four things a delimited-string column destroys. Which is the *dangerous* one?
4. Why is `phone1, phone2, phone3` a violation even though each cell holds one value?
5. State the four-question test for arrays and JSONB. Give one example that passes and one that fails.
6. When is a composite value like `address` acceptable, and when is it not?
7. Why is the string-splitting the *easy* part of a 1NF migration?

---

## Quick reference card

**1NF:** atomic values · no repeating groups · a key · no order significance

**The three violations**

| Shape | Example | Fix |
|---|---|---|
| Multi-value in a cell | `'ethnic,sale'` | junction table (or array, if the test allows) |
| Repeating group | `phone1, phone2, phone3` | **always** a separate table |
| Composite you query the parts of | `'…, Bengaluru, 560001'` | decompose the queried parts only |

**The four-question test** (for arrays/JSONB)

1. Query individual elements? 2. Update individual elements? 3. Need referential integrity? 4. Elements have attributes?

**Any yes → separate table. All no → atomic.**

**Costs, measured (2M products, 3 tags)**

| Design | "tagged 'sale'" | Correct? | FK? |
|---|---|---|---|
| CSV | 1,206 ms | ✗ false matches | ✗ |
| `text[]` + GIN | 84 ms | ✓ | ✗ |
| Junction table | 11 ms | ✓ | ✓ |

**Array vs junction:** array for opaque, code-managed, whole-set values. Junction for entities, user-created values, FK integrity, attributes, or bidirectional queries.

---

## When would I use this at work?

1. **Any schema review with a plural column name.** `tags`, `skills`, `categories`, `emails` — check the type. If it's `text`, it's almost certainly a CSV, and it's almost certainly making a search both slow and wrong.

2. **When a product requirement is "impossible."** "We can't filter by two skills at once" is usually a 1NF violation, not a hard problem. Naming it turns a feature refusal into a migration.

3. **Deciding on a JSONB column in a design doc.** The four-question test settles it in thirty seconds, and prevents both mistakes — the purist one (decomposing an embedding) and the lazy one (`line_items jsonb`).

---

## Connected topics

**Understand before this:** 20 (multi-valued attributes, Q5), 21 (Rule 7), 16 (GIN, and why `LIKE '%x%'` can't use a B-tree).

**This unlocks:**
- **32–36** — every later normal form assumes 1NF
- **25** — arrays and JSONB, and what they cost physically
- **27** — the CSV column as a named antipattern
- **35** — 4NF, which is what happens when you have *two* independent multi-valued attributes
