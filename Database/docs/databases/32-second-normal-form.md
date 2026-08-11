# 32 — Second Normal Form (2NF)
## Phase: Normalisation

---

## ELI5 — The Simple Analogy

A school keeps one sheet per **(student, subject)** pair. On each sheet it writes:

```
 Student: Arjun    Subject: Physics
 ─────────────────────────────────
 Marks:            78              ← depends on BOTH student and subject ✓
 Student's class:  10-B            ← depends only on the STUDENT
 Subject's teacher: Mrs Iyer       ← depends only on the SUBJECT
```

Arjun takes six subjects, so **"10-B" is written on six sheets.** Physics is taken by forty students, so **"Mrs Iyer" is written on forty sheets.**

Move Arjun to 10-C and you must find six sheets. Miss one and the school's records say Arjun is in two classes at once. Mrs Iyer goes on leave and you have forty sheets to correct.

And a new student who hasn't picked subjects yet has **no sheet at all**, so their class cannot be recorded.

The fix is obvious once you see it: **facts about the student go on a student sheet; facts about the subject go on a subject sheet; only the marks stay on the (student, subject) sheet.**

That's 2NF. It only ever applies when the key is made of *more than one part*.

---

## Where this fits in the big picture

```
   30 functional dependencies (the language)
   31 1NF (atomic values — the precondition)
                          │
                          ▼
        ┌──────────────────────────────────────────┐
        │ 32 SECOND NORMAL FORM    ← YOU ARE HERE  │
        │ no PARTIAL dependency on a composite key │
        └────────────────────┬─────────────────────┘
                             ▼
                    33 3NF (no transitive dependency)
                    34 BCNF (every determinant is a key)
```

2NF is the **first of the three "misplaced fact" forms**. Each removes one precise way a fact can sit in the wrong table:

| Form | The fact depends on… |
|---|---|
| **2NF** | **part of the key** |
| 3NF | a non-key attribute |
| BCNF | a non-key that determines part of a key |

---

## What is this?

A relation is in **second normal form** when:

1. It is in 1NF, **and**
2. **No non-prime attribute is partially dependent on any candidate key.**

A **partial dependency** is `X → A` where X is a *proper subset* of a candidate key and A is non-prime (not part of any candidate key).

**Consequence:** if every candidate key is a single attribute, 2NF is automatic. **2NF is only ever an issue for composite keys.**

---

## Why does it matter for a backend developer?

Because composite keys are everywhere — junction tables, weak entities, `(order_id, product_id)`, `(user_id, date)` — and the partial dependency creeps in the moment someone adds a "convenient" column.

```
 order_lines(order_id, product_id, quantity, order_date, product_name)
             └──── PK ─────────┘

 order_date   depends on order_id alone     ← PARTIAL
 product_name depends on product_id alone   ← PARTIAL
 quantity     depends on BOTH               ← ✓ full

 THE COST:
 ① order_date is repeated once per LINE, not once per order.
    A 40-line order stores the same date 40 times.
    Changing it = 40 rows, and an inconsistent state is LEGAL.
 ② product_name is repeated once per SALE. 8 million times for
    40,000 products.
 ③ You cannot record an order with no lines, or a product never sold.
 ④ ★ You cannot declare UNIQUE, NOT NULL or an FK on either — the
    fact isn't in a table where its determinant is a key (Topic 30).
```

And the reason it is so common: **a partial dependency always looks like a helpful denormalisation.** "We put `product_name` on the line so we don't have to join." Sometimes that *is* right (a snapshot — see the framework), and knowing the difference is the skill.

---

## The physical reality

### The dependency diagram

```
 order_lines(order_id, product_id, quantity, order_date, product_name, product_price)
             └──────── candidate key ────────┘

              order_id ──────────────────────▶ order_date
                 │                              ↑ PARTIAL (left half only)
                 │
              product_id ───────────────────▶ product_name, product_price
                 │                              ↑ PARTIAL (right half only)
                 │
   (order_id, product_id) ──────────────────▶ quantity
                                                ↑ FULL — needs both ✓

 ★ THREE DIFFERENT DETERMINANTS. ONE TABLE.
   Only the third one's determinant is the key.
```

### What the repetition costs

```
 8,000,000 order lines · 2,000,000 orders · 40,000 products

 order_date    (8 B) × 8,000,000 = 64 MB   ← should be 2,000,000 × 8 = 16 MB
 product_name (24 B) × 8,000,000 = 192 MB  ← should be   40,000 × 24 = 1 MB
 product_price (8 B) × 8,000,000 = 64 MB   ← should be   40,000 × 8 = 0.3 MB
                                   ───────
                                   320 MB, of which ~303 MB is repetition

 THE WRITE COST — the part that matters:
   "a product is renamed"
     violating:  UPDATE 200 rows on average (8M/40k). Every one is a
                 tuple rewrite plus index entries (Topic 46).
                 ★ AND a partial UPDATE leaves two names for one product.
     2NF:        UPDATE 1 row. Inconsistency is impossible.

 ⇒ AND THE CONSTRAINT COST (the real one):
     violating:  cannot declare UNIQUE(product_name), cannot FK to a
                 product list, cannot NOT NULL a product with no sales.
     2NF:        all three are one line of DDL.
```

---

## How it works — step by step

### Detecting a partial dependency

```
 THE PROCEDURE:

 ① Find the candidate key(s). (Topic 30's closure algorithm.)
 ② If every candidate key is a SINGLE attribute → ★ already in 2NF. Stop.
 ③ For each composite candidate key K and each proper subset S ⊂ K:
      compute S⁺
      if S⁺ contains any NON-PRIME attribute → ★ PARTIAL DEPENDENCY.

 WORKED EXAMPLE
   R(order_id, product_id, quantity, order_date, product_name)
   F = { (order_id, product_id) → quantity
       , order_id → order_date
       , product_id → product_name }

   ① candidate key: (order_id, product_id)   — the only one
   ② composite ⇒ continue
   ③ proper subsets: {order_id}, {product_id}
      {order_id}⁺   = {order_id, order_date}
                      order_date is NON-PRIME  ⇒ ★ PARTIAL
      {product_id}⁺ = {product_id, product_name}
                      product_name is NON-PRIME ⇒ ★ PARTIAL
   ⇒ NOT in 2NF. Two violations.
```

### The decomposition

```
 FOR EACH partial dependency S → A:
   ① create a new relation R_S = S ∪ {everything S determines}
   ② remove those non-prime attributes from the original
   ③ S becomes the primary key of R_S
   ④ S remains in the original as a FOREIGN KEY

 APPLIED:
   {order_id} → order_date
     ⇒ orders(order_id PK, order_date)
   {product_id} → product_name
     ⇒ products(product_id PK, product_name)
   remaining:
     ⇒ order_lines(order_id, product_id, quantity)  PK (order_id, product_id)

 ★ LOSSLESS CHECK (Topic 30):
   order_lines ∩ orders = {order_id}, which is a KEY of orders  ✓
   order_lines ∩ products = {product_id}, a KEY of products     ✓
   ⇒ the decomposition is lossless.

 ★ DEPENDENCY PRESERVATION:
   order_id → order_date        → enforced by orders' PK        ✓
   product_id → product_name    → enforced by products' PK      ✓
   (order_id,product_id) → qty  → enforced by order_lines' PK   ✓
   ⇒ all three preserved. ★ 2NF decomposition ALWAYS preserves
     dependencies, because you never split a determinant.
```

### Why 2NF is automatic with a single-attribute key

```
 If the candidate key is {id}, its only proper subset is {} (the empty set).
 {}⁺ = {} for any non-trivial F.
 ⇒ there is no proper subset that determines anything.
 ⇒ ★ NO PARTIAL DEPENDENCY IS POSSIBLE. 2NF holds automatically.

 ⇒ THE PRACTICAL CONSEQUENCE:
   A surrogate `id` primary key makes 2NF trivially true — but it does
   NOT make the table correct. The partial dependency is still there,
   hiding behind the surrogate:

   order_lines(id PK, order_id, product_id, quantity, order_date, product_name)
   ⇒ candidate keys are {id} AND {order_id, product_id}
     (the second only if you declared it UNIQUE — and you should have)
   ⇒ against {order_id, product_id}, the partial dependencies are STILL
     VIOLATIONS.

 ★★★ THIS IS THE MOST IMPORTANT PRACTICAL POINT IN THIS TOPIC:
   A surrogate key can HIDE a 2NF violation from the formal check while
   leaving every anomaly in place. You must normalise against the
   NATURAL candidate key, not the surrogate. (Topic 30's point about
   finding the real candidate keys.)
```

---

## Concept breakdown

```
SECOND NORMAL FORM
│  └── 1NF + no non-prime attribute partially depends on a candidate key.
│
├── PARTIAL DEPENDENCY   S → A where S ⊊ K (a proper subset of a
│                        candidate key) and A is non-prime
├── FULL DEPENDENCY      the whole key is needed
└── ★ ONLY POSSIBLE with a COMPOSITE candidate key.

THE THREE-STEP CHECK
 ① find the candidate keys
 ② single-attribute keys only? → 2NF automatic, stop
 ③ for each proper subset S of each composite key: does S⁺ contain a
    non-prime attribute? → violation

THE DECOMPOSITION
 for each partial dependency S → A:
   new table (S, everything S determines), S as PK
   remove those attributes from the original, keep S as an FK
 ⇒ always lossless (S is a key of the new table)
 ⇒ always dependency-preserving (no determinant is split)

★ THE SURROGATE-KEY TRAP
  Adding `id bigserial PRIMARY KEY` makes {id} a candidate key, so the
  formal 2NF check passes — while every anomaly remains. Normalise
  against the NATURAL key. Declare it UNIQUE so it is visible.

WHAT 2NF DOES NOT CATCH
 └── a non-key determining a non-key (order_id → customer_id → city).
     That is transitive, and 3NF's job (Topic 33).
```

---

## Diagrams

**Diagram 1 — big picture: three determinants, one table**

```
 order_lines
 ┌──────────┬────────────┬──────────┬────────────┬──────────────┐
 │ order_id │ product_id │ quantity │ order_date │ product_name │
 └────┬─────┴──────┬─────┴─────┬────┴──────┬─────┴───────┬──────┘
      │            │           │           │             │
      └────────────┴───────────┘           │             │
              the KEY determines           │             │
              quantity ✓ FULL              │             │
                                           │             │
      └────────────────────────────────────┘             │
              order_id ALONE determines order_date       │
              ★ PARTIAL — the left half of the key       │
                                                          │
                   └──────────────────────────────────────┘
                           product_id ALONE determines product_name
                           ★ PARTIAL — the right half of the key

 ⇒ DECOMPOSE ALONG THE DETERMINANTS:
   ┌──────────────────┐  ┌────────────────────┐  ┌──────────────────┐
   │ orders           │  │ order_lines        │  │ products         │
   │ order_id  PK     │◀─┤ order_id    FK     │  │ product_id PK    │
   │ order_date       │  │ product_id  FK     ├─▶│ product_name     │
   └──────────────────┘  │ quantity           │  └──────────────────┘
                         │ PK (order_id,      │
                         │     product_id)    │
                         └────────────────────┘
```

**Diagram 2 — data flow: renaming a product**

```
  VIOLATING 2NF                        2NF
  ──────────────────────────           ──────────────────────────
  "rename product 88"                  "rename product 88"
          │                                     │
  UPDATE order_lines                     UPDATE products
   SET product_name = 'New'               SET name = 'New'
   WHERE product_id = 88                  WHERE id = 88
          │                                     │
  ┌───────┴────────────┐                        ▼
  │ ~200 rows          │                 ┌─────────────┐
  │ 200 tuple rewrites │                 │ 1 row       │
  │ 200×N index entries│                 │ 1 rewrite   │
  │                    │                 │             │
  │ ★ AND a partial    │                 │ ★ TWO NAMES │
  │  UPDATE leaves TWO │                 │  FOR ONE    │
  │  names for one     │                 │  PRODUCT IS │
  │  product — a LEGAL │                 │  PHYSICALLY │
  │  state of the table│                 │  IMPOSSIBLE │
  └────────────────────┘                 └─────────────┘
```

**Diagram 3 — before/after: the surrogate-key trap**

```
 ✗ THE TRAP
 order_lines(id PK, order_id, product_id, quantity, order_date, product_name)
 ┌──────────────────────────────────────────────────────────────────┐
 │ Candidate keys:  {id}  ← the surrogate                           │
 │ Formal 2NF check against {id}: single attribute ⇒ PASSES         │
 │                                                                  │
 │ ★ BUT: order_date is still repeated per line                     │
 │        product_name is still repeated per sale                   │
 │        renaming a product still touches 200 rows                 │
 │        inconsistent states are still LEGAL                       │
 │                                                                  │
 │ ⇒ THE FORM PASSED. THE TABLE IS STILL WRONG.                     │
 └──────────────────────────────────────────────────────────────────┘

 ✓ THE FIX: find and DECLARE the natural key first
 ALTER TABLE order_lines ADD CONSTRAINT uq_line UNIQUE (order_id, product_id);
 ┌──────────────────────────────────────────────────────────────────┐
 │ Candidate keys:  {id}  AND  {order_id, product_id}               │
 │ 2NF check against the COMPOSITE key:                             │
 │   {order_id}⁺   ∋ order_date   (non-prime)  ⇒ ★ VIOLATION       │
 │   {product_id}⁺ ∋ product_name (non-prime)  ⇒ ★ VIOLATION       │
 │ ⇒ now the check finds what was always there.                     │
 └──────────────────────────────────────────────────────────────────┘
```

---

## Example 1 — basic

**Step 1 — build the violating table and reproduce the anomalies.**

```sql
CREATE TABLE order_lines_bad (
  order_id      bigint NOT NULL,
  product_id    bigint NOT NULL,
  quantity      int    NOT NULL,
  order_date    date   NOT NULL,     -- ★ depends on order_id alone
  customer_id   bigint NOT NULL,     -- ★ depends on order_id alone
  product_name  text   NOT NULL,     -- ★ depends on product_id alone
  product_price bigint NOT NULL,     -- ★ depends on product_id alone
  PRIMARY KEY (order_id, product_id)
);

INSERT INTO order_lines_bad VALUES
 (1,101,2,'2026-03-01',7,'Basmati Rice',54900),
 (1,102,1,'2026-03-01',7,'Ghee',        72000),
 (1,103,3,'2026-03-01',7,'Jaggery',     18000),
 (2,101,1,'2026-03-02',8,'Basmati Rice',54900),
 (3,101,5,'2026-03-03',9,'Basmati Rice',54900);
```

```sql
-- ★ the repetition, made visible
SELECT order_id, count(*) AS lines, count(DISTINCT order_date) AS dates
FROM order_lines_bad GROUP BY 1;
```
```
 order_id | lines | dates
----------+-------+-------
        1 |     3 |     1      ← the same date stored 3 times
        2 |     1 |     1
        3 |     1 |     1
```

**The update anomaly:**
```sql
-- rename a product — it appears on 3 lines
UPDATE order_lines_bad SET product_name='Basmati Rice 5kg'
WHERE product_id=101 AND order_id=1;          -- a partial update

SELECT product_id, count(DISTINCT product_name) AS variants,
       array_agg(DISTINCT product_name) AS names
FROM order_lines_bad GROUP BY 1 HAVING count(DISTINCT product_name) > 1;
```
```
 product_id | variants |              names
------------+----------+---------------------------------
        101 |        2 | {Basmati Rice,Basmati Rice 5kg}
```
**Two names for one product — a legal state of this table.** And no constraint can prevent it (Topic 30's Case C).

**The insert anomaly:**
```sql
INSERT INTO order_lines_bad (product_id, product_name, product_price)
VALUES (104,'Turmeric',22000);
-- ERROR: null value in column "order_id" violates not-null constraint
--   ★ a new product cannot exist until it is sold
```

**The delete anomaly:**
```sql
DELETE FROM order_lines_bad WHERE order_id IN (2,3);
SELECT DISTINCT product_id, product_name FROM order_lines_bad WHERE product_id=101;
-- still there via order 1 — but delete order 1's line too:
DELETE FROM order_lines_bad WHERE order_id=1 AND product_id=101;
SELECT count(*) FROM order_lines_bad WHERE product_id=101;
```
```
 count
-------
     0        ★ product 101's name and price no longer exist anywhere
```

**Step 2 — verify formally.**

```
 F = { (order_id, product_id) → quantity
     , order_id → order_date, customer_id
     , product_id → product_name, product_price }

 Candidate key: {order_id, product_id}
 Prime: order_id, product_id
 Non-prime: quantity, order_date, customer_id, product_name, product_price

 {order_id}⁺   = {order_id, order_date, customer_id}
                  ⇒ contains NON-PRIME → ★ PARTIAL
 {product_id}⁺ = {product_id, product_name, product_price}
                  ⇒ contains NON-PRIME → ★ PARTIAL
 ⇒ NOT 2NF.
```

**Step 3 — decompose.**

```sql
CREATE TABLE orders (
  id bigint PRIMARY KEY,
  order_date date NOT NULL,
  customer_id bigint NOT NULL
);
CREATE TABLE products (
  id bigint PRIMARY KEY,
  name text NOT NULL,
  price_minor bigint NOT NULL CHECK (price_minor >= 0),
  CONSTRAINT uq_products_name UNIQUE (name)          -- ★ NOW POSSIBLE
);
CREATE TABLE order_lines (
  order_id bigint NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
  product_id bigint NOT NULL REFERENCES products(id) ON DELETE RESTRICT,
  quantity int NOT NULL CHECK (quantity > 0),
  PRIMARY KEY (order_id, product_id)
);
```

**Step 4 — all three anomalies are now impossible.**

```sql
INSERT INTO orders VALUES (1,'2026-03-01',7),(2,'2026-03-02',8),(3,'2026-03-03',9);
INSERT INTO products VALUES (101,'Basmati Rice',54900),(102,'Ghee',72000),
                            (103,'Jaggery',18000);
INSERT INTO order_lines VALUES (1,101,2),(1,102,1),(1,103,3),(2,101,1),(3,101,5);

-- ① UPDATE: one row, and two names is now physically impossible
UPDATE products SET name='Basmati Rice 5kg' WHERE id=101;
SELECT count(DISTINCT name) FROM products WHERE id=101;   -- always 1

-- ② INSERT: a product with no sales — fine
INSERT INTO products VALUES (104,'Turmeric',22000);       -- ✓

-- ③ DELETE: remove every line for product 101; the product survives
DELETE FROM order_lines WHERE product_id=101;
SELECT name, price_minor FROM products WHERE id=101;
```
```
       name       | price_minor
------------------+-------------
 Basmati Rice 5kg |       54900     ★ still there
```

**Step 5 — the surrogate-key trap, demonstrated.**

```sql
CREATE TABLE ol_surrogate (
  id bigserial PRIMARY KEY,                 -- ★ the surrogate
  order_id bigint NOT NULL,
  product_id bigint NOT NULL,
  quantity int NOT NULL,
  order_date date NOT NULL,
  product_name text NOT NULL
);
INSERT INTO ol_surrogate (order_id,product_id,quantity,order_date,product_name)
VALUES (1,101,2,'2026-03-01','Basmati Rice'),
       (1,102,1,'2026-03-01','Ghee'),
       (2,101,1,'2026-03-02','Basmati Rice');

-- FORMAL CHECK against {id}: single attribute ⇒ 2NF holds. ✓
-- REALITY:
UPDATE ol_surrogate SET product_name='Renamed' WHERE id=1;
SELECT product_id, array_agg(DISTINCT product_name) FROM ol_surrogate
GROUP BY 1 HAVING count(DISTINCT product_name) > 1;
```
```
 product_id |      array_agg
------------+----------------------------
        101 | {Basmati Rice,Renamed}      ★ THE ANOMALY IS STILL THERE
```
```sql
-- ★ and worse: the surrogate also hid a MISSING constraint
INSERT INTO ol_surrogate (order_id,product_id,quantity,order_date,product_name)
VALUES (1,101,99,'2026-03-01','Basmati Rice');   -- ✓ ACCEPTED — a duplicate line!
```
**The surrogate key made 2NF pass *and* allowed duplicate order lines.** Both problems are fixed by finding and declaring the natural key:
```sql
ALTER TABLE ol_surrogate ADD CONSTRAINT uq_line UNIQUE (order_id, product_id);
-- now the 2NF check against the composite key finds both violations
```

**Step 6 — measure it at scale.**

```sql
CREATE TABLE ol_bad AS
SELECT (i/4)::bigint AS order_id, (i%40000)::bigint AS product_id,
       1+(i%5) AS quantity, current_date - (i%400) AS order_date,
       'Product '||(i%40000) AS product_name, ((i%40000)*13)::bigint AS product_price
FROM generate_series(1,8000000) i;

CREATE TABLE n_orders AS SELECT DISTINCT order_id AS id, order_date FROM ol_bad;
CREATE TABLE n_products AS SELECT DISTINCT product_id AS id, product_name AS name,
       product_price AS price FROM ol_bad;
CREATE TABLE n_lines AS SELECT order_id, product_id, quantity FROM ol_bad;

SELECT 'violating' AS design, pg_size_pretty(pg_relation_size('ol_bad')) AS size
UNION ALL SELECT '2NF', pg_size_pretty(pg_relation_size('n_orders')
   + pg_relation_size('n_products') + pg_relation_size('n_lines'));
```
```
   design   |  size
------------+---------
 violating  | 727 MB
 2NF        | 388 MB      ← 47% smaller
```

---

## Example 2 — production scenario

**The situation.** An LMS (learning management system), 5 years old. The `enrolments` table has 240 million rows and is the busiest in the schema.

```sql
CREATE TABLE enrolments (
  student_id     bigint NOT NULL,
  course_code    text   NOT NULL,
  semester       text   NOT NULL,
  -- the actual enrolment fact
  grade          text,
  enrolled_at    timestamptz NOT NULL,
  -- ★ facts about the student
  student_name   text NOT NULL,
  student_email  text NOT NULL,
  student_programme text NOT NULL,
  student_year   smallint NOT NULL,
  -- ★ facts about the course
  course_title   text NOT NULL,
  course_credits smallint NOT NULL,
  course_dept    text NOT NULL,
  -- ★ facts about the (course, semester) offering
  instructor_id  bigint NOT NULL,
  instructor_name text NOT NULL,
  room           text NOT NULL,
  PRIMARY KEY (student_id, course_code, semester)
);
```

**Step 1 — classify every column by its determinant.**

```
 Candidate key: (student_id, course_code, semester)

 grade, enrolled_at        → the FULL key                      ✓
 student_*                 → {student_id}                      ★ PARTIAL
 course_title/credits/dept → {course_code}                     ★ PARTIAL
 instructor_*, room        → {course_code, semester}           ★ PARTIAL
                              (a proper subset of the 3-part key)

 ⇒ THREE distinct partial dependencies, on three different subsets.
```

**Step 2 — measure the repetition and find the corruption.**

```sql
SELECT count(*) AS rows,
       count(DISTINCT student_id) AS students,
       count(DISTINCT course_code) AS courses,
       count(DISTINCT (course_code, semester)) AS offerings
FROM enrolments;
```
```
   rows    | students | courses | offerings
-----------+----------+---------+-----------
 240102884 |  1204008 |    4102 |     41208
```
```
 student_name  repeated 240M/1.2M = ~200× per student
 course_title  repeated 240M/4.1k = ~58,500× per course
 instructor_name repeated 240M/41k = ~5,800× per offering
```

```sql
-- and the inevitable corruption (Topic 30's detector)
SELECT 'student_id → student_email' AS fd,
       count(*) FILTER (WHERE n>1) AS violated_groups
FROM (SELECT student_id, count(DISTINCT student_email) n FROM enrolments GROUP BY 1) t
UNION ALL
SELECT 'course_code → course_title',
       count(*) FILTER (WHERE n>1)
FROM (SELECT course_code, count(DISTINCT course_title) n FROM enrolments GROUP BY 1) t
UNION ALL
SELECT 'course_code → course_credits',
       count(*) FILTER (WHERE n>1)
FROM (SELECT course_code, count(DISTINCT course_credits) n FROM enrolments GROUP BY 1) t
UNION ALL
SELECT '(course,sem) → instructor_id',
       count(*) FILTER (WHERE n>1)
FROM (SELECT course_code, semester, count(DISTINCT instructor_id) n
      FROM enrolments GROUP BY 1,2) t;
```
```
              fd              | violated_groups
------------------------------+-----------------
 student_id → student_email   |            8412   ⚠ email changes
 course_code → course_title   |             412   ⚠ course renames
 course_code → course_credits |              88   ⚠⚠ CREDITS DIFFER
 (course,sem) → instructor_id |              14   ⚠ instructor swaps
```

**The `course_credits` one is the serious finding:**

```sql
SELECT course_code, array_agg(DISTINCT course_credits) AS credit_variants,
       count(*) AS affected_enrolments
FROM enrolments GROUP BY 1 HAVING count(DISTINCT course_credits) > 1
ORDER BY 3 DESC LIMIT 3;
```
```
 course_code | credit_variants | affected_enrolments
-------------+-----------------+---------------------
 CS301       | {3,4}           |              418204
 MA201       | {3,4}           |              288104
 PH102       | {2,3}           |              204882
```

**88 courses have two different credit values.** Credits determine graduation eligibility. **Some students' degree progress has been computed with the wrong number** — and the database cannot say which value is correct, because there is no single row that *is* the course.

**Step 3 — the insert and delete anomalies, in the business.**

```
 INSERT: a new course cannot be created until a student enrols.
   ⇒ the registry's actual workaround: enrol a dummy student
     (student_id = 0, name = 'PLACEHOLDER') in every new course.
     There are 4,102 such rows, and every enrolment report has
     `AND student_id > 0`. For five years.

 DELETE: purging enrolments older than 7 years (a policy requirement)
   would delete the only record of 340 discontinued courses, including
   their titles and credits — which are needed to interpret old
   transcripts.
   ⇒ the purge has been deferred for three years. The table is 240M
     rows when it should be ~90M.
```

**Step 4 — the 2NF decomposition.**

```sql
CREATE TABLE students (
  id bigint PRIMARY KEY,
  name text NOT NULL,
  email citext NOT NULL,
  programme text NOT NULL,
  year smallint NOT NULL CHECK (year BETWEEN 1 AND 8),
  CONSTRAINT uq_students_email UNIQUE (email)         -- ★ now enforceable
);

CREATE TABLE courses (
  code text PRIMARY KEY,
  title text NOT NULL,
  credits smallint NOT NULL CHECK (credits BETWEEN 1 AND 12),  -- ★ ONE value
  dept text NOT NULL
);

CREATE TABLE instructors (
  id bigint PRIMARY KEY,
  name text NOT NULL
);

CREATE TABLE course_offerings (
  course_code text NOT NULL REFERENCES courses(code) ON DELETE RESTRICT,
  semester text NOT NULL,
  instructor_id bigint NOT NULL REFERENCES instructors(id) ON DELETE RESTRICT,
  room text NOT NULL,
  PRIMARY KEY (course_code, semester)                 -- ★ ONE instructor
);

CREATE TABLE enrolments (
  student_id bigint NOT NULL REFERENCES students(id) ON DELETE RESTRICT,
  course_code text NOT NULL,
  semester text NOT NULL,
  grade text NULL CHECK (grade IN ('A','B','C','D','F','W','I')),
  enrolled_at timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (student_id, course_code, semester),
  FOREIGN KEY (course_code, semester)
    REFERENCES course_offerings(course_code, semester) ON DELETE RESTRICT
);
CREATE INDEX ON enrolments (course_code, semester);
```

⚠ **One deliberate exception, and it matters here:**

```sql
-- Credits can legitimately change between semesters (a curriculum revision).
-- The credits a student EARNED are a fact about the enrolment, not about
-- the course today. (Topics 26, 29 — snapshot vs reference.)
ALTER TABLE enrolments ADD COLUMN credits_earned smallint NULL;
-- ★ This is NOT the 2NF violation returning. `courses.credits` is
--   "what this course is worth now"; `enrolments.credits_earned` is
--   "what this student earned then". Two different facts.
--   ⇒ and it is what makes the 88-course discrepancy resolvable:
--     for each affected enrolment, the historically correct value can
--     be recorded once, deliberately, with the registry's sign-off.
```

**Step 5 — the migration, with the hard part named.**

```sql
-- ① COURSES: 412 title variants, 88 credit conflicts.
--    Titles: pick the most recent — mechanical.
INSERT INTO courses (code, title, credits, dept)
SELECT DISTINCT ON (course_code) course_code, course_title, course_credits, course_dept
FROM enrolments ORDER BY course_code, enrolled_at DESC;

--    ★ CREDITS: NOT mechanical. 88 courses, 900k affected enrolments,
--      and the answer affects graduation. This goes to the registry.
CREATE TABLE credit_conflicts AS
SELECT course_code, course_credits, count(*) AS enrolments,
       min(enrolled_at) AS first_seen, max(enrolled_at) AS last_seen
FROM enrolments WHERE course_code IN (
  SELECT course_code FROM enrolments GROUP BY 1 HAVING count(DISTINCT course_credits)>1)
GROUP BY 1,2 ORDER BY 1, 4;
-- ⇒ the pattern is almost always "changed on date X" — which confirms
--   it is TEMPORAL, and the fix is credits_earned per enrolment.

-- ② STUDENTS: 8,412 email conflicts — take the most recent. Mechanical.
INSERT INTO students (id, name, email, programme, year)
SELECT DISTINCT ON (student_id) student_id, student_name, student_email,
       student_programme, student_year
FROM enrolments WHERE student_id > 0        -- ★ exclude the PLACEHOLDER rows
ORDER BY student_id, enrolled_at DESC;

-- ③ THE 4,102 PLACEHOLDER ROWS: delete them, and remove
--    `AND student_id > 0` from 80 reports.
```

**Step 6 — results.**

| | Before | After |
|---|---|---|
| Storage | 118 GB | **41 GB** |
| `course_code → credits` variants | 88 courses conflicted | **0, enforced** |
| Enrolments with wrong credits | ~900,000 unknown | **resolved, per-enrolment** |
| Creating a new course | a dummy enrolment | **`INSERT INTO courses`** |
| 7-year purge | blocked 3 years | **safe** |
| Reports with `student_id > 0` | 80 | **0** |
| Enforceable constraints | 1 | **17** |

**The graduation-eligibility bug is the headline.** A 2NF violation put a fact that determines a legal outcome in a table where the engine could not enforce it.

---

## Common mistakes

**1. Assuming a surrogate key means 2NF.**
- *Symptom:* the formal check passes; every anomaly remains.
- *Engine-level why:* `{id}` is a single-attribute candidate key, so no proper subset can determine anything. The check is vacuously true.
- *Fix:* find the **natural** candidate key, declare it `UNIQUE`, and check against that.

**2. Not declaring the natural key at all.**
- *Symptom:* duplicate order lines, duplicate enrolments.
- *Fix:* the same `UNIQUE` constraint. It fixes the 2NF check *and* a real bug.

**3. Removing a column that is actually a snapshot.**
- *Symptom:* historical records change when a reference value changes.
- *Fix:* the test from Topic 29 — "if the source changes, *should* this change too?" No → it's a snapshot, keep it and rename it to say so (`credits_earned`, not `credits`).

**4. Decomposing without checking losslessness.**
- *Symptom:* joining the pieces produces rows that never existed.
- *Fix:* the shared attribute must be a key of the new table. For 2NF it always is (S is the PK of R_S), so this is automatic — but verify it anyway.

**5. Treating 2NF as the finish line.**
- *Symptom:* `orders(id, customer_id, customer_city)` — 2NF holds (single-attribute key) but there's a transitive dependency.
- *Fix:* 2NF is one of three checks. Continue to 3NF (Topic 33).

**6. Underestimating the data-cleaning cost.**
- *Symptom:* a migration plan that assumes decomposition is `INSERT … SELECT DISTINCT`.
- *Fix:* every FD violation you found is a conflict someone must resolve. 412 title variants are mechanical; **88 credit conflicts need the registry**, and that is the schedule.

---

## Hands-on proof

**PROVE IT #1 — the three anomalies.** (Example 1, step 1.)
**PROVE IT #2 — all three become impossible after decomposition.** (Example 1, step 4.)
**PROVE IT #3 — the surrogate-key trap.** (Example 1, step 5.)
**PROVE IT #4 — the storage cost.** (Example 1, step 6.)

**PROVE IT #5 — detect partial dependencies in any table.**
```sql
-- For a table with composite PK (a, b) and non-prime column c,
-- test whether c depends on `a` alone:
SELECT a, count(DISTINCT c) AS distinct_c
FROM t GROUP BY a HAVING count(DISTINCT c) = 1;
-- ★ if EVERY group has exactly 1, then a → c holds, and since `a` is
--   a proper subset of the key, that is a PARTIAL DEPENDENCY.

-- applied:
SELECT count(*) FILTER (WHERE n = 1) AS consistent,
       count(*) FILTER (WHERE n > 1) AS violated,
       count(*) AS total_groups
FROM (SELECT order_id, count(DISTINCT order_date) n
      FROM order_lines_bad GROUP BY 1) t;
-- consistent = total_groups ⇒ order_id → order_date holds ⇒ PARTIAL
```

**PROVE IT #6 — find composite-key tables in your schema.**
```sql
SELECT c.relname,
       string_agg(a.attname, ', ' ORDER BY x.ord) AS pk_columns,
       (SELECT count(*) FROM pg_attribute a2
         WHERE a2.attrelid=c.oid AND a2.attnum>0 AND NOT a2.attisdropped) AS total_cols
FROM pg_constraint con
JOIN pg_class c ON c.oid = con.conrelid
CROSS JOIN LATERAL unnest(con.conkey) WITH ORDINALITY x(attnum, ord)
JOIN pg_attribute a ON a.attrelid=c.oid AND a.attnum=x.attnum
WHERE con.contype='p' AND cardinality(con.conkey) > 1
GROUP BY c.oid, c.relname
HAVING (SELECT count(*) FROM pg_attribute a2
         WHERE a2.attrelid=c.oid AND a2.attnum>0 AND NOT a2.attisdropped) > 4
ORDER BY 3 DESC;
-- ★ a composite PK with many extra columns is where 2NF violations live
```

---

## The design decision framework

```
THE CHECK — for every table:

 ① FIND THE CANDIDATE KEYS. (Topic 30.)
    ★ NOT just the declared PK. If it's a surrogate `id`, find the
      NATURAL key too — and DECLARE IT UNIQUE.

 ② ARE ALL CANDIDATE KEYS SINGLE-ATTRIBUTE?
    YES → 2NF is automatic. Go to 3NF (Topic 33).
    NO  → continue.

 ③ FOR EACH PROPER SUBSET S OF EACH COMPOSITE KEY:
      does S determine any NON-PRIME attribute?
      Test it:  SELECT S, count(DISTINCT A) FROM t GROUP BY S;
                if every group has exactly 1 → S → A holds → ★ PARTIAL
      → decompose: new table (S, everything S determines), S as PK

 ④ BEFORE REMOVING A COLUMN, CHECK IT IS NOT A SNAPSHOT:
    "If the source value changes, SHOULD this row's value change?"
      YES → a copy. Remove it; it belongs in the new table.
      NO  → ★ a snapshot. KEEP IT, and RENAME IT so the difference is
             visible: credits_earned, unit_price_paid, name_at_purchase.
    ⇒ This is the single most common way 2NF gets misapplied.

WHEN A PARTIAL DEPENDENCY IS DELIBERATE (rare, and must be justified):
  ✓ it is a snapshot (see above) — not actually a violation
  ✓ measured evidence that the join is a bottleneck, AND a documented
    plan for keeping the copies consistent (Phase 6)
  ✗ "we didn't want to join" with no measurement — that is a violation
    with a story attached

THE SIGNAL TO LOOK FOR:
  Run PROVE IT #6 to find composite-key tables with many columns. For
  each, run the FD detector (PROVE IT #5) on each key-subset/column pair.

  • every group has exactly 1 distinct value → a partial dependency,
    confirmed. Decompose.
  • SOME groups have >1 → either the FD is false (it's a snapshot or
    temporal), or corruption has already occurred. Check the RATE:
      low  → corruption. Fix the data, then decompose.
      high → it's a snapshot or a temporal fact. Do NOT decompose;
             rename the column to reflect what it actually is.
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
Build `order_lines_bad` from Example 1. For each of the four non-prime columns, run the FD detector and state whether it depends on the full key, on `order_id` alone, or on `product_id` alone. Then produce all three anomalies with SQL, and decompose the table. Verify losslessness by joining the pieces and comparing row counts.

### Exercise 2 — medium (apply it)
Given:
```sql
CREATE TABLE shipment_items (
  shipment_id bigint, item_sku text, quantity int,
  shipment_date date, carrier_code text, carrier_name text,
  origin_hub text, item_name text, item_weight_g int, item_hs_code text,
  PRIMARY KEY (shipment_id, item_sku)
);
```
(a) Write out every FD and classify it as full or partial.
(b) Identify which proper subsets of the key are determinants.
(c) Decompose to 2NF and prove the decomposition is lossless and dependency-preserving.
(d) One column is arguably a snapshot rather than a partial dependency. Identify it and argue both sides.
(e) Show which constraints become expressible after the decomposition.

### Exercise 3 — hard (production simulation)
A 240M-row `enrolments` table has PK `(student_id, course_code, semester)` and carries student attributes, course attributes, and offering attributes inline. An audit finds: 8,412 students with conflicting emails, 412 courses with conflicting titles, **88 courses with conflicting credit values affecting 900,000 enrolments**, 14 offerings with conflicting instructors, 4,102 `PLACEHOLDER` rows excluded by 80 reports, and a 7-year purge blocked for three years.

(a) Classify every column by its determinant, showing three distinct partial dependencies.
(b) Write the detection queries that produced the audit numbers.
(c) The credit conflict affects graduation eligibility. Explain why the database cannot resolve it, and what that says about the cost of a 2NF violation.
(d) Distinguish which conflicts are mechanical to resolve and which need a human decision. Justify each.
(e) Design the 2NF decomposition. Identify the one column that must be added as a *snapshot* and explain why it is not the violation returning.
(f) Explain how the delete anomaly blocked a policy requirement for three years.
(g) Give the migration plan using Topic 28's techniques, including how you handle the 900,000 affected enrolments.
(h) List every constraint that becomes enforceable, and say which business rule each encodes.

---

## Mental model checkpoint

1. Define a partial dependency precisely. What two conditions must hold?
2. Why is 2NF automatic when every candidate key is a single attribute?
3. Explain the surrogate-key trap. Why does the formal check pass while the table stays wrong?
4. Give the three-step procedure for detecting a partial dependency.
5. Why is a 2NF decomposition always lossless *and* always dependency-preserving?
6. `order_lines.unit_price` looks like a partial dependency on `product_id`. When is it, and when isn't it? What's the test?
7. A partial-dependency check shows 88 of 4,102 groups violated. What are the two possible interpretations, and how do you choose?

---

## Quick reference card

**2NF:** 1NF + no non-prime attribute partially depends on a candidate key.

| Term | Meaning |
|---|---|
| Partial dependency | `S → A`, S ⊊ candidate key, A non-prime |
| Full dependency | the whole key is needed |
| Prime attribute | in some candidate key |

**Only applies to composite keys.** Single-attribute key ⇒ 2NF automatic.

**The check**
```
① find candidate keys (including the NATURAL one, not just the surrogate)
② single-attribute? → 2NF automatic
③ for each proper subset S: does S⁺ contain a non-prime attribute?
```

**The detector**
```sql
SELECT <subset>, count(DISTINCT <column>) FROM t GROUP BY <subset>;
-- every group = 1  →  the FD holds  →  partial dependency
```

**The decomposition:** for each `S → A`, create `(S, everything S determines)` with S as PK; keep S in the original as an FK. **Always lossless, always dependency-preserving.**

**★ The two traps**
1. A surrogate `id` makes the check pass vacuously — normalise against the natural key.
2. A snapshot (`unit_price_paid`, `credits_earned`) is *not* a partial dependency — test with "if the source changes, should this?"

---

## When would I use this at work?

1. **Any junction table with more than three columns.** `(order_id, product_id, quantity, order_date, product_name)` — the extra columns are almost always partial dependencies, and the detector confirms it in one query.

2. **When a table has a surrogate PK and no unique constraint on the natural key.** That combination hides a 2NF violation *and* permits duplicate business rows. One `UNIQUE` constraint fixes both.

3. **When a reference value has drifted.** "88 courses have two credit values" is a partial dependency that has already corrupted itself — and the fact that the database *cannot say which is right* is the clearest possible argument for decomposing.

---

## Connected topics

**Understand before this:** 30 (closure, candidate keys, prime attributes), 31 (1NF, the precondition), 22 (surrogate vs natural keys — the trap here).

**This unlocks:**
- **33** — 3NF: the transitive dependency, which 2NF does not catch
- **34** — BCNF: the stricter form, and where 3NF is deliberately preferred
- **37** — the full worked example, applying 1NF → 2NF → 3NF → BCNF in order
- **38** — when to stop, and when a "violation" is a deliberate snapshot
