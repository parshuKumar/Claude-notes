# 33 — Third Normal Form (3NF)
## Phase: Normalisation

---

## ELI5 — The Simple Analogy

An office keeps one card per **employee**:

```
 Employee: Arjun Sharma   (emp_id 4471)
 ────────────────────────────────────────
 Name:            Arjun Sharma      ← about the employee ✓
 Department code: D-12              ← about the employee ✓
 Department name: Engineering       ← ★ about the DEPARTMENT
 Department head: Meera Rao         ← ★ about the DEPARTMENT
 Department floor: 3                ← ★ about the DEPARTMENT
```

Every column depends on the employee id — so 2NF is satisfied. But look at the chain: **employee → department → department name.** The department's name isn't really a fact about Arjun. It's a fact about D-12 that got copied onto Arjun's card, and onto the cards of the other 340 people in Engineering.

Rename the department and you have 341 cards to change. Appoint a new head — 341 cards. And a department with no employees yet has **no card at all**, so it cannot exist.

The fix: **give the department its own card.**

That's 3NF. Where 2NF removed facts that depend on *part of the key*, 3NF removes facts that depend on *something that isn't the key at all*.

---

## Where this fits in the big picture

```
   30 FDs · 31 1NF · 32 2NF (no partial dependency)
                          │
                          ▼
        ┌──────────────────────────────────────────┐
        │ 33 THIRD NORMAL FORM     ← YOU ARE HERE  │
        │ no TRANSITIVE dependency                 │
        │ (non-key → non-key)                      │
        └────────────────────┬─────────────────────┘
                             ▼
                    34 BCNF (every determinant is a key)
                    ★ 3NF is where almost every production
                      schema correctly stops (Topic 38)
```

| Form | The misplaced fact depends on… |
|---|---|
| 2NF | **part of the key** |
| **3NF** | **a non-key attribute** |
| BCNF | a non-key that determines part of a key |

---

## What is this?

A relation is in **third normal form** when:

1. It is in 2NF, **and**
2. **No non-prime attribute is transitively dependent on any candidate key.**

A **transitive dependency** is `K → X → A` where K is a candidate key, X is *not* a superkey, and A is non-prime.

The equivalent, more usable statement — for every non-trivial FD `X → A`:

> either **X is a superkey**, or **A is a prime attribute.**

That second escape clause is what separates 3NF from BCNF (Topic 34), and it exists for a very specific reason.

**The classic mnemonic:** every non-key attribute must depend on *the key* (1NF/identity), *the whole key* (2NF), *and nothing but the key* (3NF) — "so help me Codd."

---

## Why does it matter for a backend developer?

Because the transitive dependency is the **most common normalisation defect in real schemas**, and it hides behind a surrogate key where 2NF cannot see it:

```
 orders(id PK, customer_id, customer_name, customer_city, order_date)

 2NF? ✓ — the key is a single attribute, so no partial dependency
          is even possible (Topic 32).
 3NF? ✗ — id → customer_id → customer_name
          `customer_id` is not a superkey; `customer_name` is non-prime.

 ⇒ ★ 2NF's check passes completely and the table is still wrong.
   This is why you cannot skip 3NF: for surrogate-keyed tables —
   which is most of them — 3NF is the FIRST check that finds anything.

 THE COST:
 ① a customer's name is repeated once per ORDER (not once per customer)
 ② renaming a customer touches every order they ever placed
 ③ a customer with no orders cannot exist
 ④ deleting their last order deletes the customer
 ⑤ ★ UNIQUE(customer_name), NOT NULL city, an FK to a customer list —
    none can be declared
```

---

## The physical reality

### The chain, drawn

```
 orders(id, customer_id, customer_name, customer_city, customer_tier, order_date)

     id ────────────────▶ customer_id ──────────▶ customer_name
      │  (key → non-key)       │  (non-key → non-key)  customer_city
      │                        │                       customer_tier
      │                        └── ★ customer_id is NOT a superkey of
      │                            `orders` — one customer has many orders
      │
      └────────────────▶ order_date        ✓ genuinely about the order

 ★ THE SIGNATURE: an attribute X that is neither a key nor part of one,
   yet determines other attributes. X is a foreign entity that has
   leaked in.
```

### The repetition, measured

```
 8,000,000 orders · 200,000 customers

 customer_name (~18 B) × 8,000,000 = 144 MB  ← should be 200,000 × 18 = 3.6 MB
 customer_city (~12 B) × 8,000,000 =  96 MB  ← should be 200,000 × 12 = 2.4 MB
 customer_tier  (~8 B) × 8,000,000 =  64 MB  ← should be 200,000 × 8  = 1.6 MB
                                    ───────
                                    304 MB, of which 296 MB is repetition

 ★ AVERAGE REPETITION: 40× per customer. For a customer with 4,000
   orders, one name change is 4,000 tuple rewrites — and a partial
   update leaves two names, which is a LEGAL state of the table.
```

### Why 3NF's "or A is prime" escape clause exists

```
 THE 3NF CONDITION, restated:
   for every non-trivial X → A:  X is a superkey  OR  A is prime.

 THE BCNF CONDITION (Topic 34):
   for every non-trivial X → A:  X is a superkey.  (no escape clause)

 ★ WHY 3NF ALLOWS THE EXCEPTION:
   Because ALWAYS achieving BCNF can force you to LOSE a functional
   dependency — meaning a business rule that no single table can
   enforce, so the engine cannot check it with a constraint.

   3NF, by contrast, can ALWAYS be achieved with a decomposition that
   is BOTH lossless AND dependency-preserving. That guarantee is
   valuable enough that 3NF is often the correct final answer.
   (Topic 34 shows the exact case; Topic 38 makes the decision.)
```

---

## How it works — step by step

### Detecting a transitive dependency

```
 THE PROCEDURE:

 ① find all candidate keys (Topic 30)
 ② mark every attribute as PRIME (in some candidate key) or NON-PRIME
 ③ for every non-trivial FD  X → A  in F:
      is X a superkey?           → ✓ fine
      is A prime?                → ✓ fine (3NF's escape clause)
      NEITHER                    → ★ 3NF VIOLATION

 WORKED EXAMPLE
   orders(id, customer_id, customer_name, customer_city, order_date)
   F = { id → customer_id, order_date
       , customer_id → customer_name, customer_city }

   ① candidate key: {id}
   ② prime: id.  non-prime: customer_id, customer_name, customer_city, order_date
   ③ id → customer_id           X={id} is a superkey        ✓
      id → order_date           superkey                    ✓
      customer_id → customer_name
         X={customer_id}: is it a superkey?
           {customer_id}⁺ = {customer_id, customer_name, customer_city}
           ≠ all attributes ⇒ NOT a superkey
         A=customer_name: prime? NO
         ⇒ ★ 3NF VIOLATION
      customer_id → customer_city  → same ⇒ ★ VIOLATION
```

### The decomposition

```
 FOR EACH violating FD  X → A₁, A₂, … :
   ① create R_X = X ∪ {A₁, A₂, …}, with X as the PRIMARY KEY
   ② remove A₁, A₂, … from the original relation
   ③ X stays in the original as a FOREIGN KEY

 APPLIED:
   customer_id → customer_name, customer_city
     ⇒ customers(customer_id PK, customer_name, customer_city)
   remaining:
     ⇒ orders(id PK, customer_id FK, order_date)

 ★ LOSSLESS:  orders ∩ customers = {customer_id}, a KEY of customers ✓
 ★ DEPENDENCY-PRESERVING:
     id → customer_id, order_date       → orders' PK          ✓
     customer_id → name, city           → customers' PK       ✓
   ⇒ every FD is now enforced by a PRIMARY KEY index. Exactly the
     goal from Topic 30: move each FD into a table where its
     determinant is a key.
```

### The 3NF synthesis algorithm (Bernstein)

For a table with many FDs, decomposing one at a time can produce redundant tables. The **synthesis algorithm** produces a minimal 3NF decomposition that is guaranteed lossless *and* dependency-preserving:

```
 INPUT: R and a set of FDs F.

 ① Compute a MINIMAL COVER of F (Topic 30):
      single-attribute RHS · no redundant FD · no extraneous LHS attribute

 ② For each FD  X → A  in the minimal cover, create a relation (X ∪ A).
    ★ Combine FDs with the SAME left-hand side into one relation.

 ③ If no relation contains a candidate key of R, add one relation
    consisting of a candidate key.
    ⇒ this is what guarantees LOSSLESSNESS.

 ④ Remove any relation that is a subset of another.

 GUARANTEES: lossless join ✓ · dependency preservation ✓ · 3NF ✓
 ⚠ It does NOT guarantee BCNF. That is deliberate (Topic 34).

 WORKED EXAMPLE
   R(emp_id, emp_name, dept_code, dept_name, dept_head, project_id, hours)
   F = { emp_id → emp_name, dept_code
       , dept_code → dept_name, dept_head
       , (emp_id, project_id) → hours }

   ① minimal cover (split RHSs; none redundant; no extraneous LHS):
        emp_id → emp_name
        emp_id → dept_code
        dept_code → dept_name
        dept_code → dept_head
        (emp_id, project_id) → hours

   ② group by LHS:
        R1(emp_id, emp_name, dept_code)
        R2(dept_code, dept_name, dept_head)
        R3(emp_id, project_id, hours)

   ③ candidate key of R = {emp_id, project_id}
        contained in R3 ✓ ⇒ no extra relation needed

   ④ no relation is a subset of another ⇒ done.

   ⇒ THREE tables, lossless, dependency-preserving, in 3NF.
```

---

## Concept breakdown

```
THIRD NORMAL FORM
│  └── 2NF + no non-prime attribute is transitively dependent on a
│      candidate key.
│
├── EQUIVALENT (and more usable): for every non-trivial X → A,
│     X is a SUPERKEY  OR  A is PRIME.
│
├── TRANSITIVE DEPENDENCY   K → X → A, where X is not a superkey
│                           and A is non-prime
└── ★ THE ESCAPE CLAUSE "or A is prime" is what distinguishes 3NF
     from BCNF, and it exists so that 3NF can ALWAYS be achieved
     losslessly AND dependency-preservingly.

THE MNEMONIC
  "every non-key attribute depends on the key (1NF),
   the whole key (2NF), and nothing but the key (3NF)"

WHY 3NF MATTERS MORE THAN 2NF IN PRACTICE
  A surrogate `id` PK makes 2NF vacuously true (Topic 32's trap).
  ⇒ for most real tables, 3NF is the FIRST check that finds anything.
  ⇒ the transitive dependency is the most common defect in the wild.

THE SIGNATURE TO LOOK FOR
  A group of columns sharing a prefix — customer_*, dept_*, product_* —
  next to the id they depend on. That prefix is a foreign entity that
  leaked in.

SYNTHESIS ALGORITHM (Bernstein)
  minimal cover → one relation per LHS group → add a candidate key if
  none is contained → remove subsets.
  ⇒ guarantees lossless + dependency-preserving + 3NF.

WHAT 3NF DOES NOT CATCH
  A non-key determining a PRIME attribute. That is BCNF's job (T34),
  and it is rare.
```

---

## Diagrams

**Diagram 1 — big picture: the chain**

```
 orders
 ┌────┬─────────────┬───────────────┬───────────────┬────────────┐
 │ id │ customer_id │ customer_name │ customer_city │ order_date │
 └─┬──┴──────┬──────┴───────┬───────┴───────┬───────┴──────┬─────┘
   │         │              │               │              │
   ├─────────┘              │               │              │
   │  id → customer_id ✓    │               │              │
   │  (key → anything, fine)│               │              │
   │                        │               │              │
   ├────────────────────────┼───────────────┼──────────────┘
   │  id → order_date ✓                     │
   │                        │               │
   │         ┌──────────────┴───────────────┘
   │         │  ★ customer_id → customer_name, customer_city
   │         │    customer_id is NOT a superkey
   │         │    the RHS attributes are NOT prime
   │         │    ⇒ TRANSITIVE. 3NF VIOLATION.
   └─────────┘

 ⇒ DECOMPOSE ALONG THE CHAIN:
   ┌──────────────────┐        ┌────────────────────┐
   │ orders           │        │ customers          │
   │ id          PK   │        │ id            PK   │
   │ customer_id FK   ├───────▶│ name               │
   │ order_date       │        │ city               │
   └──────────────────┘        └────────────────────┘
```

**Diagram 2 — data flow: the 3NF check**

```
              FOR EVERY NON-TRIVIAL FD  X → A
                            │
                  Is X a SUPERKEY of R?
                            │
              ┌─────YES─────┴──────NO──────┐
              ▼                            ▼
          ✓ FINE                  Is A a PRIME attribute?
      (the key determines                  │
       everything — that's        ┌───YES──┴───NO───┐
       what a key IS)             ▼                 ▼
                            ✓ 3NF OK          ★ 3NF VIOLATION
                            ✗ BCNF violation   → decompose:
                              (Topic 34)          new table (X, A…)
                                                  X as PK
                            ↑
                     ★ the escape clause —
                       rare, and it is what
                       makes 3NF always
                       achievable
```

**Diagram 3 — before/after: what becomes enforceable**

```
 BEFORE — orders(id, customer_id, customer_name, customer_city, tier)
 ┌───────────────────────────────────────────────────────────────────┐
 │ "a customer has one name"        ✗ inexpressible                  │
 │ "city must not be null"          ✗ not for a customer with no orders│
 │ "customer_id must exist"         ✗ there is no customer table      │
 │ "tier ∈ {bronze,silver,gold}"    ~ a CHECK works, but is repeated  │
 │                                    on 8M rows and can still hold   │
 │                                    two values for one customer     │
 │                                                                    │
 │ Rename a customer → 40 rows on average, 4,000 for a big one.       │
 │ A partial update → TWO names for one customer. A LEGAL state.      │
 └───────────────────────────────────────────────────────────────────┘

 AFTER — orders(id, customer_id FK, order_date) + customers(id, name, city, tier)
 ┌───────────────────────────────────────────────────────────────────┐
 │ "a customer has one name"        ✓ it is a COLUMN of one row      │
 │ "city must not be null"          ✓ NOT NULL                       │
 │ "customer_id must exist"         ✓ FOREIGN KEY                    │
 │ "tier ∈ {…}"                     ✓ CHECK, on 200k rows, once      │
 │ "no two customers share a phone" ✓ UNIQUE                         │
 │                                                                    │
 │ Rename a customer → 1 row. Two names is PHYSICALLY IMPOSSIBLE.    │
 └───────────────────────────────────────────────────────────────────┘
        ↑ five business rules moved from "we hope" to "the engine refuses"
```

---

## Example 1 — basic

**Step 1 — the violating table.**

```sql
CREATE TABLE orders_bad (
  id            bigint PRIMARY KEY,
  customer_id   bigint NOT NULL,
  customer_name text   NOT NULL,     -- ★ depends on customer_id
  customer_city text   NOT NULL,     -- ★ depends on customer_id
  customer_tier text   NOT NULL,     -- ★ depends on customer_id
  order_date    date   NOT NULL,
  total_minor   bigint NOT NULL
);

INSERT INTO orders_bad VALUES
 (1,7,'Arjun','Bengaluru','gold','2026-03-01',249900),
 (2,7,'Arjun','Bengaluru','gold','2026-03-05',129900),
 (3,7,'Arjun','Bengaluru','gold','2026-03-09', 45000),
 (4,8,'Meera','Mumbai',   'silver','2026-03-02',88000);
```

**Step 2 — 2NF passes, 3NF fails.**

```
 Candidate key: {id}  — a single attribute
 ⇒ 2NF: no proper subset of a single-attribute key exists ⇒ AUTOMATIC ✓

 3NF check:
   id → customer_id, order_date, total_minor    superkey ✓
   customer_id → customer_name, city, tier
     {customer_id}⁺ = {customer_id, customer_name, customer_city, customer_tier}
     ≠ all attributes ⇒ NOT a superkey
     customer_name is non-prime
     ⇒ ★ 3NF VIOLATION
```

**Step 3 — the anomalies.**

```sql
-- ① UPDATE: a partial rename
UPDATE orders_bad SET customer_name='Arjun Sharma' WHERE id=1;
SELECT customer_id, count(DISTINCT customer_name) AS variants,
       array_agg(DISTINCT customer_name) AS names
FROM orders_bad GROUP BY 1 HAVING count(DISTINCT customer_name)>1;
```
```
 customer_id | variants |         names
-------------+----------+------------------------
           7 |        2 | {Arjun,Arjun Sharma}     ★ a LEGAL state
```
```sql
-- and no constraint can prevent it:
ALTER TABLE orders_bad ADD CONSTRAINT ck_name CHECK (
  (SELECT count(DISTINCT customer_name) FROM orders_bad o2
    WHERE o2.customer_id = orders_bad.customer_id) = 1);
-- ERROR: cannot use subquery in check constraint      (Topic 24)
```

```sql
-- ② INSERT: a customer with no orders
INSERT INTO orders_bad (customer_id, customer_name, customer_city, customer_tier)
VALUES (9,'Ravi','Delhi','bronze');
-- ERROR: null value in column "id" violates not-null constraint
--   ★ (and even with an id, you'd need a fake order_date and total)
```

```sql
-- ③ DELETE: the customer disappears with their last order
DELETE FROM orders_bad WHERE customer_id=8;
SELECT count(*) FROM orders_bad WHERE customer_id=8;   -- 0
--   ★ Meera's name, city and tier no longer exist anywhere
```

**Step 4 — decompose.**

```sql
CREATE TABLE customers (
  id bigint PRIMARY KEY,
  name text NOT NULL,
  city text NOT NULL,
  tier text NOT NULL CHECK (tier IN ('bronze','silver','gold'))
);
CREATE TABLE orders (
  id bigint PRIMARY KEY,
  customer_id bigint NOT NULL REFERENCES customers(id) ON DELETE RESTRICT,
  order_date date NOT NULL,
  total_minor bigint NOT NULL CHECK (total_minor >= 0)
);
CREATE INDEX ON orders (customer_id);

INSERT INTO customers VALUES (7,'Arjun','Bengaluru','gold'),
                             (8,'Meera','Mumbai','silver');
INSERT INTO orders VALUES (1,7,'2026-03-01',249900),(2,7,'2026-03-05',129900),
                          (3,7,'2026-03-09',45000),(4,8,'2026-03-02',88000);
```

```sql
-- ① UPDATE: one row; two names is now impossible
UPDATE customers SET name='Arjun Sharma' WHERE id=7;
SELECT count(DISTINCT name) FROM customers WHERE id=7;   -- always 1

-- ② INSERT: a customer with no orders
INSERT INTO customers VALUES (9,'Ravi','Delhi','bronze');   -- ✓

-- ③ DELETE: orders go, the customer stays
DELETE FROM orders WHERE customer_id=8;
SELECT name, city FROM customers WHERE id=8;
```
```
 name  |  city
-------+--------
 Meera | Mumbai        ★ preserved
```

**Step 5 — the synthesis algorithm on a bigger example.**

```sql
CREATE TABLE employee_projects (
  emp_id      bigint NOT NULL,
  emp_name    text   NOT NULL,
  dept_code   text   NOT NULL,
  dept_name   text   NOT NULL,
  dept_head   bigint NOT NULL,
  project_id  bigint NOT NULL,
  hours       int    NOT NULL,
  PRIMARY KEY (emp_id, project_id)
);
```
```
 F = { emp_id → emp_name, dept_code
     , dept_code → dept_name, dept_head
     , (emp_id, project_id) → hours }

 ① MINIMAL COVER (split, no redundancy, no extraneous LHS):
      emp_id → emp_name
      emp_id → dept_code
      dept_code → dept_name
      dept_code → dept_head
      (emp_id, project_id) → hours

 ② GROUP BY LHS:
      R1(emp_id, emp_name, dept_code)
      R2(dept_code, dept_name, dept_head)
      R3(emp_id, project_id, hours)

 ③ candidate key {emp_id, project_id} ⊆ R3  ✓  no extra relation

 ④ no subsets ⇒ FINAL
```
```sql
CREATE TABLE departments (
  code text PRIMARY KEY,
  name text NOT NULL,
  head_emp_id bigint NOT NULL
);
CREATE TABLE employees (
  id bigint PRIMARY KEY,
  name text NOT NULL,
  dept_code text NOT NULL REFERENCES departments(code) ON DELETE RESTRICT
);
ALTER TABLE departments ADD CONSTRAINT fk_head
  FOREIGN KEY (head_emp_id) REFERENCES employees(id) DEFERRABLE INITIALLY DEFERRED;
--   ★ circular reference ⇒ DEFERRABLE (Topic 23)
CREATE TABLE employee_projects (
  emp_id bigint NOT NULL REFERENCES employees(id) ON DELETE CASCADE,
  project_id bigint NOT NULL REFERENCES projects(id) ON DELETE RESTRICT,
  hours int NOT NULL CHECK (hours >= 0),
  PRIMARY KEY (emp_id, project_id)
);
```

**Step 6 — measure at scale.**

```sql
CREATE TABLE o_bad AS
SELECT i AS id, (i%200000)::bigint AS customer_id,
       'Customer '||(i%200000) AS customer_name,
       (ARRAY['Bengaluru','Mumbai','Delhi','Chennai'])[1+(i%4)] AS customer_city,
       (ARRAY['bronze','silver','gold'])[1+(i%3)] AS customer_tier,
       current_date - (i%400) AS order_date, (i*13)::bigint AS total_minor
FROM generate_series(1,8000000) i;

CREATE TABLE n_customers AS SELECT DISTINCT customer_id AS id, customer_name AS name,
  customer_city AS city, customer_tier AS tier FROM o_bad;
CREATE TABLE n_orders AS SELECT id, customer_id, order_date, total_minor FROM o_bad;

SELECT 'violating' AS design, pg_size_pretty(pg_relation_size('o_bad')) AS size
UNION ALL SELECT '3NF', pg_size_pretty(pg_relation_size('n_customers')
  + pg_relation_size('n_orders'));
```
```
   design   |  size
------------+---------
 violating  | 826 MB
 3NF        | 465 MB      ← 44% smaller
```

```sql
-- and the read cost of the join it introduces
ALTER TABLE n_customers ADD PRIMARY KEY (id);
CREATE INDEX ON n_orders (customer_id);
VACUUM ANALYZE n_customers; VACUUM ANALYZE n_orders;
CREATE INDEX ON o_bad (id);
VACUUM ANALYZE o_bad;

EXPLAIN (ANALYZE,BUFFERS) SELECT customer_name, customer_city, order_date
FROM o_bad WHERE id=4471;
-- Execution Time: 0.06 ms   Buffers: shared hit=4

EXPLAIN (ANALYZE,BUFFERS) SELECT c.name, c.city, o.order_date
FROM n_orders o JOIN n_customers c ON c.id=o.customer_id WHERE o.id=4471;
-- Execution Time: 0.11 ms   Buffers: shared hit=8
```
**0.05 ms of extra read cost**, in exchange for 361 MB, five enforceable constraints, and the impossibility of three anomaly classes.

---

## Example 2 — production scenario

**The situation.** A B2B invoicing platform, 6 years old. `invoice_lines` has 890 million rows and is 640 GB.

```sql
CREATE TABLE invoice_lines (
  id              bigserial PRIMARY KEY,
  invoice_id      bigint NOT NULL,
  line_no         smallint NOT NULL,
  -- the line itself
  quantity        int NOT NULL,
  unit_price_minor bigint NOT NULL,
  -- ★ facts about the PRODUCT
  product_id      bigint NOT NULL,
  product_sku     text NOT NULL,
  product_name    text NOT NULL,
  product_category text NOT NULL,
  product_tax_code text NOT NULL,
  -- ★ facts about the TAX CODE (a chain of two!)
  tax_rate_bp     int NOT NULL,
  tax_authority   text NOT NULL,
  -- ★ facts about the INVOICE
  invoice_number  text NOT NULL,
  invoice_date    date NOT NULL,
  customer_id     bigint NOT NULL,
  -- ★ facts about the CUSTOMER
  customer_name   text NOT NULL,
  customer_gstin  text NOT NULL,
  customer_state  text NOT NULL
);
```

**Step 1 — map the dependency chains.**

```
 Candidate keys: {id}  and  {invoice_id, line_no}   (natural — is it declared?)

 CHAIN 1:  id → invoice_id → invoice_number, invoice_date, customer_id
                                    │
                            → customer_id → customer_name, gstin, state
           ★ A TWO-STEP CHAIN. invoice → customer → customer attributes.

 CHAIN 2:  id → product_id → product_sku, name, category, tax_code
                                    │
                            → product_tax_code → tax_rate_bp, tax_authority
           ★ ANOTHER TWO-STEP CHAIN.

 ⇒ FOUR transitive dependencies, in two chains of two.
```

**Step 2 — measure the repetition and find the corruption.**

```sql
SELECT count(*) AS lines,
       count(DISTINCT invoice_id) AS invoices,
       count(DISTINCT customer_id) AS customers,
       count(DISTINCT product_id) AS products,
       count(DISTINCT product_tax_code) AS tax_codes
FROM invoice_lines;
```
```
   lines   | invoices | customers | products | tax_codes
-----------+----------+-----------+----------+-----------
 890412008 | 41204882 |     88214 |   412008 |        14
```
```
 customer_name repeated 890M/88k  = ~10,000× per customer
 product_name  repeated 890M/412k = ~2,160× per product
 tax_rate_bp   repeated 890M/14   = ~63,600,000× per tax code  ★
```

```sql
-- the FD violation detector (Topic 30)
SELECT 'customer_id → customer_gstin' AS fd,
       count(*) FILTER (WHERE n>1) AS violated
FROM (SELECT customer_id, count(DISTINCT customer_gstin) n
      FROM invoice_lines GROUP BY 1) t
UNION ALL SELECT 'product_id → product_name',
       count(*) FILTER (WHERE n>1)
FROM (SELECT product_id, count(DISTINCT product_name) n
      FROM invoice_lines GROUP BY 1) t
UNION ALL SELECT 'product_tax_code → tax_rate_bp',
       count(*) FILTER (WHERE n>1)
FROM (SELECT product_tax_code, count(DISTINCT tax_rate_bp) n
      FROM invoice_lines GROUP BY 1) t
UNION ALL SELECT 'invoice_id → customer_id',
       count(*) FILTER (WHERE n>1)
FROM (SELECT invoice_id, count(DISTINCT customer_id) n
      FROM invoice_lines GROUP BY 1) t;
```
```
               fd                | violated
---------------------------------+----------
 customer_id → customer_gstin    |      412    ⚠⚠ TAX IDENTIFIERS
 product_id → product_name       |     8104    ⚠  renames
 product_tax_code → tax_rate_bp  |       14    ⚠⚠⚠ ALL 14 tax codes
 invoice_id → customer_id        |        0    ✓
```

**The tax findings are the serious ones.**

```sql
SELECT product_tax_code, array_agg(DISTINCT tax_rate_bp ORDER BY tax_rate_bp) AS rates,
       count(*) AS lines
FROM invoice_lines GROUP BY 1 ORDER BY 3 DESC LIMIT 3;
```
```
 product_tax_code |    rates     |   lines
------------------+--------------+-----------
 GST18            | {1200,1800}  | 412008841
 GST12            | {500,1200}   | 204882104
 GST5             | {0,500}      |  88214002
```

```
 ★ TWO READINGS, and they lead to opposite actions:
   (a) THE FD IS FALSE — tax rates change over time (they did: GST
       rates were revised). The rate on a line is a SNAPSHOT of what
       was charged, and MUST NOT be normalised away. (Topics 26, 29.)
   (b) THE FD IS TRUE and the data is corrupt.

 ⇒ Check the temporal pattern:
     SELECT product_tax_code, tax_rate_bp, min(invoice_date), max(invoice_date)
     FROM invoice_lines GROUP BY 1,2 ORDER BY 1,3;
   If each rate occupies a clean date range → (a), temporal. CONFIRMED.
 ⇒ ★ tax_rate_bp on the line is CORRECT and must be KEPT.
   But `tax_authority` shows 0 violations and IS a pure transitive
   dependency — that one is removed.
```

```sql
-- the GSTIN finding is different
SELECT customer_id, array_agg(DISTINCT customer_gstin) FROM invoice_lines
GROUP BY 1 HAVING count(DISTINCT customer_gstin)>1 LIMIT 3;
```
```
 customer_id |            array_agg
-------------+----------------------------------
       88214 | {29ABCDE1234F1Z5,29ABCDE1234F1Z6}
       41022 | {27PQRST5678G1Z2,27PQRST5678G1ZZ}
```
**Two of these are typos** (a trailing character differs). **412 customers have an invalid GSTIN on some invoices** — which is a statutory filing error, because GST returns are filed per GSTIN.

**Step 3 — the insert and delete anomalies.**

```
 INSERT: a new product cannot exist until it is invoiced.
   ⇒ the actual workaround: a ₹0 "setup" invoice per new product.
     There are 412,008 of them — one per product — and every revenue
     report has `AND total_minor > 0`. For six years.

 DELETE: statutory retention is 8 years. Purging older invoices would
   delete the only record of 14,000 discontinued products and 900
   closed customers — including their GSTINs, which are needed if a
   past return is ever audited.
   ⇒ the purge is blocked. The table is 640 GB when it should be ~180 GB.
```

**Step 4 — the 3NF design.**

```sql
CREATE TABLE tax_codes (
  code text PRIMARY KEY,
  authority text NOT NULL                    -- ★ the pure transitive dep
);
CREATE TABLE tax_rates (                     -- ★ the TEMPORAL fact
  tax_code text NOT NULL REFERENCES tax_codes(code),
  rate_bp int NOT NULL CHECK (rate_bp BETWEEN 0 AND 10000),
  valid tstzrange NOT NULL,
  EXCLUDE USING gist (tax_code WITH =, valid WITH &&)   -- Topics 24, 26
);

CREATE TABLE products (
  id bigint PRIMARY KEY,
  sku text NOT NULL UNIQUE,
  name text NOT NULL,
  category text NOT NULL,
  tax_code text NOT NULL REFERENCES tax_codes(code)
);

CREATE TABLE customers (
  id bigint PRIMARY KEY,
  name text NOT NULL,
  gstin char(15) NOT NULL,
  state char(2) NOT NULL,
  CONSTRAINT ck_gstin CHECK (gstin ~ '^[0-9]{2}[A-Z]{5}[0-9]{4}[A-Z][0-9A-Z]{3}$'),
  CONSTRAINT uq_customers_gstin UNIQUE (gstin)     -- ★ NOW ENFORCEABLE
);

CREATE TABLE invoices (
  id bigint PRIMARY KEY,
  number text NOT NULL UNIQUE,
  invoice_date date NOT NULL,
  customer_id bigint NOT NULL REFERENCES customers(id) ON DELETE RESTRICT
);

CREATE TABLE invoice_lines (
  invoice_id bigint NOT NULL REFERENCES invoices(id) ON DELETE CASCADE,
  line_no smallint NOT NULL,
  product_id bigint NOT NULL REFERENCES products(id) ON DELETE RESTRICT,
  quantity int NOT NULL CHECK (quantity > 0),
  -- ★ SNAPSHOTS — facts about THIS line, not about the product today
  unit_price_minor bigint NOT NULL CHECK (unit_price_minor >= 0),
  tax_rate_bp int NOT NULL CHECK (tax_rate_bp BETWEEN 0 AND 10000),
  PRIMARY KEY (invoice_id, line_no)
);
```

⚠ **The two snapshots that stay**, and why they are not 3NF violations:

```
 unit_price_minor  — the price CHARGED. `products` has no price column
                     at all here; prices are temporal and the invoice
                     is immutable. (Case study 04's I2.)
 tax_rate_bp       — the rate APPLIED. Confirmed temporal by the date-
                     range analysis. Statutory: an invoice must show
                     the rate in force on its date, forever.

 ★ THE TEST (Topic 29): "if the source changes, SHOULD this change?"
   NO for both ⇒ they are facts about the line, not copies.
   ⇒ Not violations. And renaming them would make it clearer still:
     `unit_price_charged_minor`, `tax_rate_applied_bp`.
```

**Step 5 — results.**

| | Before | After |
|---|---|---|
| Storage | 640 GB | **178 GB** |
| `customer_id → gstin` conflicts | 412 | **0, and format-validated** |
| Invalid GSTINs on filed returns | 412 customers | **found and corrected** |
| `product_id → name` conflicts | 8,104 | **0, enforced** |
| Creating a new product | a ₹0 invoice | **`INSERT INTO products`** |
| 8-year purge | blocked | **safe** |
| Reports with `total_minor > 0` | 40 | **0** |
| Enforceable constraints | 2 | **21** |

**The GSTIN finding is the one that mattered.** A transitive dependency put a statutory identifier in a table where no `UNIQUE` or `CHECK` could validate it, and 412 customers' tax filings were affected.

---

## Common mistakes

**1. Thinking a surrogate PK means the table is normalised.**
- *Symptom:* 2NF passes trivially; nobody checks 3NF; the table has five transitive dependencies.
- *Fix:* for surrogate-keyed tables, **3NF is the first check that finds anything.** Always run it.

**2. Missing a two-step chain.**
- *Symptom:* you decompose `invoice → customer` and stop, leaving `product → tax_code → tax_rate`.
- *Fix:* after each decomposition, **re-check the new tables.** Chains can be several links long.

**3. Normalising away a snapshot.**
- *Symptom:* historical invoices change when a price or tax rate changes.
- *Fix:* the "if the source changes, should this?" test. If the FD detector shows a *high* violation rate with a clean temporal pattern, it's a snapshot, not corruption.

**4. Not distinguishing "the FD is false" from "the data is corrupt."**
- *Symptom:* bulk-updating 412 customers' GSTINs to the most common value — including 12 where the *less* common value was correct.
- *Fix:* violation *rate* plus a temporal check. Low rate + no pattern → corruption. High rate + clean date ranges → temporal.

**5. Decomposing FD-by-FD instead of using synthesis.**
- *Symptom:* six tables where three would do, some of them subsets of others.
- *Fix:* the Bernstein algorithm — minimal cover, group by LHS, add a key if needed, remove subsets.

**6. Ignoring dependency preservation.**
- *Symptom:* a decomposition that looks tidier but where a business rule now spans two tables and cannot be a constraint.
- *Fix:* 3NF synthesis *guarantees* preservation. If you decomposed by hand, verify it.

**7. Assuming 3NF means BCNF.**
- *Symptom:* a table in 3NF with a remaining anomaly, because a non-key determines a prime attribute.
- *Fix:* Topic 34. It's rare, but real.

---

## Hands-on proof

**PROVE IT #1 — 2NF passes while 3NF fails.** (Example 1, step 2.)
**PROVE IT #2 — the three anomalies.** (Example 1, step 3.)
**PROVE IT #3 — all three become impossible.** (Example 1, step 4.)
**PROVE IT #4 — storage and read cost.** (Example 1, step 6.)

**PROVE IT #5 — detect transitive dependencies.**
```sql
-- "does non-key column X determine non-key column A?"
SELECT count(*) FILTER (WHERE n=1) AS consistent,
       count(*) FILTER (WHERE n>1) AS violated,
       count(*) AS groups
FROM (SELECT customer_id, count(DISTINCT customer_name) n
      FROM orders_bad GROUP BY 1) t;
-- consistent = groups  ⇒ customer_id → customer_name holds
-- and customer_id is NOT a superkey  ⇒ ★ TRANSITIVE
```

**PROVE IT #6 — find candidate transitive dependencies by column prefix.**
```sql
-- the signature: a group of columns sharing a prefix, next to a *_id
SELECT c.relname,
       split_part(a.attname,'_',1) AS prefix,
       count(*) AS cols,
       string_agg(a.attname, ', ' ORDER BY a.attnum) AS columns
FROM pg_class c JOIN pg_attribute a ON a.attrelid=c.oid
WHERE c.relkind='r' AND c.relnamespace='public'::regnamespace
  AND a.attnum>0 AND NOT a.attisdropped AND a.attname LIKE '%\_%'
GROUP BY 1,2
HAVING count(*) >= 3
   AND EXISTS (SELECT 1 FROM pg_attribute a2 WHERE a2.attrelid=c.oid
                 AND a2.attname = split_part(a.attname,'_',1)||'_id')
ORDER BY 3 DESC;
```
```
    relname     |  prefix  | cols |                columns
----------------+----------+------+----------------------------------------
 invoice_lines  | product  |    5 | product_id, product_sku, product_name, …
 invoice_lines  | customer |    3 | customer_id, customer_name, customer_gstin
 orders_bad     | customer |    4 | customer_id, customer_name, customer_city, …
```
**★ This one query finds most 3NF violations in a real schema.**

**PROVE IT #7 — distinguish temporal from corrupt.**
```sql
SELECT product_tax_code, tax_rate_bp,
       min(invoice_date) AS first_used, max(invoice_date) AS last_used,
       count(*) AS lines
FROM invoice_lines GROUP BY 1,2 ORDER BY 1, 3;
-- clean, non-overlapping date ranges → TEMPORAL (a snapshot; keep it)
-- interleaved dates                  → CORRUPTION (fix the data)
```

---

## The design decision framework

```
THE CHECK — for every table:

 ① FIND THE CANDIDATE KEYS. Mark every attribute PRIME or NON-PRIME.
 ② FOR EVERY NON-TRIVIAL FD  X → A:
      X a superkey?  → ✓ fine
      A prime?       → ✓ 3NF ok (but check BCNF — Topic 34)
      neither        → ★ VIOLATION
 ③ DECOMPOSE: new table (X, everything X determines), X as PK;
    keep X in the original as an FK.
 ④ ★ RE-CHECK THE NEW TABLES. Chains can be several links long.

FOR A TABLE WITH MANY FDs, USE SYNTHESIS (Bernstein):
   minimal cover → one relation per LHS group → add a candidate key
   if none is contained → remove subsets.
   ⇒ guarantees lossless + dependency-preserving + 3NF.

★ BEFORE REMOVING A COLUMN, RUN THE SNAPSHOT TEST:
   "If the source value changes, SHOULD this row's value change?"
     YES → a copy. Remove it.
     NO  → ★ a SNAPSHOT. Keep it, and RENAME it to say so:
            unit_price_charged · tax_rate_applied · name_at_purchase
   AND confirm with the data:
     violation rate LOW + no temporal pattern  → corruption; fix, decompose
     violation rate HIGH + clean date ranges   → temporal; KEEP IT

WHEN TO STOP AT 3NF RATHER THAN GOING TO BCNF:
   → when BCNF would cost you dependency preservation (Topics 34, 38).
   3NF is the correct final answer for the large majority of tables.

THE SIGNAL TO LOOK FOR:
   Run PROVE IT #6 — the column-prefix detector. Any table with a
   `foo_id` plus three or more `foo_*` columns almost certainly has a
   transitive dependency.
   Then confirm with PROVE IT #5, and classify with PROVE IT #7.

   • consistent = groups            → the FD holds → decompose
   • low violation rate             → corruption → fix data, then decompose
   • high rate + temporal pattern   → a snapshot → KEEP, and rename
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
Build `orders_bad` from Example 1. Show formally that 2NF holds and 3NF does not. Produce all three anomalies. Decompose, prove losslessness by joining and comparing counts, and list every constraint that becomes expressible.

### Exercise 2 — medium (apply it)
Given `R(emp_id, emp_name, dept_code, dept_name, dept_head, project_id, hours)` with the FDs from Example 1 step 5:
(a) find all candidate keys with the closure algorithm,
(b) identify every 3NF violation and say why,
(c) run the Bernstein synthesis algorithm, showing the minimal cover and each step,
(d) verify the result is lossless and dependency-preserving,
(e) explain why decomposing FD-by-FD instead could produce a worse result.

### Exercise 3 — hard (production simulation)
An 890M-row, 640 GB `invoice_lines` table carries product, tax-code, invoice and customer attributes inline. The audit finds: 412 customers with conflicting GSTINs, 8,104 products with conflicting names, **all 14 tax codes with two distinct rates**, 412,008 ₹0 "setup" invoices excluded by 40 reports, and an 8-year statutory purge blocked because it would delete product and customer records.

(a) Map every dependency chain, showing that two of them are two steps long.
(b) Write the detection queries that produced the audit numbers.
(c) The tax-rate finding has two possible interpretations. Give both, the query that distinguishes them, and the action each implies.
(d) The GSTIN finding has statutory consequences. Explain what a transitive dependency prevented, and what it cost.
(e) Design the 3NF schema. Identify the two columns that must remain as snapshots, and justify each with the test.
(f) Explain how the delete anomaly blocked a statutory requirement, and what that says about normalisation as a compliance concern.
(g) Give the migration plan (Topic 28), including how you resolve the 412 GSTIN conflicts — which is *not* a bulk update.
(h) List every constraint that becomes enforceable and the business rule each encodes.

---

## Mental model checkpoint

1. Define a transitive dependency precisely. What three conditions must hold?
2. State the 3NF condition in the "for every FD X → A" form. What are the two escape routes?
3. Why does 3NF matter more than 2NF for a table with a surrogate primary key?
4. Why does 3NF have the "or A is prime" escape clause? What guarantee does it buy?
5. Give the four steps of the Bernstein synthesis algorithm. What three things does it guarantee?
6. `invoice_lines.tax_rate_bp` looks like a transitive dependency on `tax_code`. Give the two tests that decide whether it is.
7. After decomposing, why must you re-check the new tables?

---

## Quick reference card

**3NF:** 2NF + no non-prime attribute is transitively dependent on a candidate key.

**Equivalently:** for every non-trivial `X → A`, **X is a superkey OR A is prime.**

**The mnemonic:** depends on *the key* (1NF), *the whole key* (2NF), *and nothing but the key* (3NF).

| Form | Misplaced fact depends on |
|---|---|
| 2NF | part of the key |
| **3NF** | **a non-key attribute** |
| BCNF | a non-key that determines part of a key |

**Synthesis (Bernstein)**
```
① minimal cover
② one relation per LHS group
③ add a relation containing a candidate key if none is contained
④ remove subsets
⇒ lossless ✓  dependency-preserving ✓  3NF ✓  (BCNF ✗ — deliberately)
```

**The detector**
```sql
SELECT <non-key col>, count(DISTINCT <other non-key col>)
FROM t GROUP BY 1;
-- every group = 1 → the FD holds → transitive → decompose
```

**The prefix signature:** a table with `foo_id` plus three or more `foo_*` columns.

**★ Before removing a column:** "if the source changes, should this?" No → it's a **snapshot**. Keep it, rename it.

---

## When would I use this at work?

1. **Auditing any table with a surrogate PK.** 2NF passes vacuously, so 3NF is where you actually find things. The column-prefix query finds most violations in one pass.

2. **When a reference value has drifted.** "All 14 tax codes have two rates" is either corruption or a temporal fact, and the date-range query tells you which — with completely different actions in each case.

3. **Explaining why a compliance problem is structural.** "We can't purge old invoices because it would delete customers" and "412 GSTINs are invalid" are both transitive dependencies, and naming them turns a workaround into a fix.

---

## Connected topics

**Understand before this:** 30 (superkeys, prime attributes, closure), 32 (2NF, and the surrogate-key trap that makes 3NF the real first check).

**This unlocks:**
- **34** — BCNF: what happens when a non-key determines a *prime* attribute
- **37** — the full worked example, 1NF → BCNF in order
- **38** — when to stop, including why 3NF is usually the right stopping point
- **26** — the temporal/snapshot distinction that decides several of these cases
