# 35 — Fourth Normal Form (4NF)
## Phase: Normalisation

---

## ELI5 — The Simple Analogy

A restaurant records two independent things about each chef:

- The **cuisines** they can cook: Arjun does Italian and Thai.
- The **shifts** they can work: Arjun does mornings and evenings.

Someone puts both on one form, one line per combination:

```
 Arjun | Italian | morning
 Arjun | Italian | evening
 Arjun | Thai    | morning
 Arjun | Thai    | evening
```

**Four rows to say two things.** And the multiplication is the tell: 2 cuisines × 2 shifts = 4 rows. Add Mexican and you must add *two* rows, not one — or the table becomes inconsistent, implying Arjun can only cook Mexican in the morning, which was never true.

The two facts have **nothing to do with each other**. Arjun's cuisines don't depend on his shifts. Putting them in one table forces the database to store every combination of two unrelated lists.

That's a **multi-valued dependency**, and 4NF removes it.

---

## Where this fits in the big picture

```
   31 1NF · 32 2NF · 33 3NF · 34 BCNF
        (all about FUNCTIONAL dependencies: X determines ONE value)
                          │
                          ▼
        ┌──────────────────────────────────────────┐
        │ 35 FOURTH NORMAL FORM    ← YOU ARE HERE  │
        │ MULTI-VALUED dependencies:               │
        │ X determines a SET of values             │
        └────────────────────┬─────────────────────┘
                             ▼
                    36 5NF (join dependencies —
                       the final generalisation)
```

**The leap:** 1NF–BCNF all concern *functional* dependencies — one determinant, one value. 4NF concerns **multi-valued** dependencies — one determinant, one *set* of values, independent of everything else in the row.

---

## What is this?

A **multi-valued dependency** `X ↠ Y` (read "X multi-determines Y") holds when, for a given X, the set of Y values is **independent** of the remaining attributes Z.

Formally: if `(x, y₁, z₁)` and `(x, y₂, z₂)` both exist, then `(x, y₁, z₂)` and `(x, y₂, z₁)` must also exist. **The table is forced to contain every combination.**

A relation is in **4NF** when it is in BCNF and every non-trivial MVD `X ↠ Y` has **X as a superkey**.

The practical form: **a table must not contain two or more independent multi-valued facts about the same key.**

---

## Why does it matter for a backend developer?

Because the symptom is unmistakable once you know it — **combinatorial row growth** — and the consequences are real:

```
 chef_capabilities(chef, cuisine, shift)

 ① ROW COUNT IS A PRODUCT, NOT A SUM
    5 cuisines × 4 shifts = 20 rows per chef.
    Should be 5 + 4 = 9.
    At 2,000 chefs: 40,000 rows instead of 18,000.
    ⇒ and it gets worse quadratically as either list grows.

 ② EVERY INSERT IS N ROWS
    "Arjun learns Mexican" = one INSERT per shift he works.
    Forget one and the table says he cooks Mexican only in the morning.
    ⇒ ★ AN INCONSISTENT STATE IS A LEGAL STATE.

 ③ EVERY DELETE IS N ROWS
    "Arjun stops working evenings" = delete one row per cuisine.

 ④ ★ THE CONSTRAINT IS INEXPRESSIBLE
    "the cuisine set and the shift set are independent" cannot be
    stated as a CHECK, UNIQUE, or FK. Nothing prevents the table
    from encoding a dependency that does not exist.
```

And a fifth, subtler one: **the data now asserts something false.** A row `(Arjun, Mexican, morning)` with no `(Arjun, Mexican, evening)` *reads* as "Arjun cooks Mexican only in the morning" — a claim the business never made.

---

## The physical reality

### The combinatorial explosion, measured

```
 chef_capabilities(chef, cuisine, shift)   PK = all three columns

 ONE CHEF with 5 cuisines and 4 shifts:
 ┌───────┬──────────┬──────────┐
 │ Arjun │ Italian  │ morning  │
 │ Arjun │ Italian  │ afternoon│
 │ Arjun │ Italian  │ evening  │
 │ Arjun │ Italian  │ night    │
 │ Arjun │ Thai     │ morning  │      ★ 5 × 4 = 20 ROWS
 │ …     │ …        │ …        │        to express 5 + 4 = 9 facts
 │ Arjun │ Mexican  │ night    │
 └───────┴──────────┴──────────┘

 AT SCALE — 2,000 chefs, avg 5 cuisines, 4 shifts:
   violating: 2,000 × 20 = 40,000 rows × ~40 B = 1.6 MB
   4NF:       2,000 × 5 = 10,000 rows  (chef_cuisines)
            + 2,000 × 4 =  8,000 rows  (chef_shifts)
              = 18,000 rows × ~28 B    = 0.5 MB
   ⇒ 2.2× fewer rows, 3.2× less storage

 ⚠ AND THE GROWTH IS MULTIPLICATIVE:
   a chef who learns a 6th cuisine adds 4 rows, not 1.
   a chef who adds a 5th shift adds 6 rows, not 1.
   ⇒ the table grows as O(m × n), not O(m + n).
```

### Where it becomes serious

```
 THE SHAPE APPEARS IN REAL SCHEMAS AS:

  product_availability(product_id, size, colour)
    ⇒ 8 sizes × 12 colours = 96 rows per product
      for 8 + 12 = 20 facts
    ⚠ AND IT IS OFTEN NOT AN MVD — see "the crucial test" below

  employee_qualifications(emp_id, skill, language)
  doctor_availability(doctor_id, clinic, day_of_week)
  course_resources(course_id, textbook, lecturer)
  vehicle_compatibility(part_id, make, model_year)

 ★ AT 8M PRODUCTS × 96 COMBINATIONS = 768 MILLION ROWS
   for what should be 160 million. That is a 4.8× table, and it
   compounds every time either list grows.
```

---

## How it works — step by step

### The formal definition, and the test that matters

```
 X ↠ Y  holds in R(X, Y, Z)  if:
   whenever (x, y₁, z₁) and (x, y₂, z₂) are both in R,
   then (x, y₁, z₂) and (x, y₂, z₁) are ALSO in R.

 ⇒ i.e. for each x, the table contains the full CROSS PRODUCT of
   Y-values and Z-values.

 ★ NOTE: X ↠ Y always implies X ↠ Z. MVDs come in PAIRS.
   (That is why the condition is symmetric.)

 ★ AND: every FD is an MVD. If X → Y then X ↠ Y trivially.
   ⇒ 4NF implies BCNF.

 TRIVIAL MVDs (ignore them):
   X ↠ Y when Y ⊆ X, or when X ∪ Y = all attributes.
   ⇒ a two-column table can never violate 4NF.

 ★★★ THE CRUCIAL TEST — INDEPENDENCE ★★★
   An MVD exists ONLY IF the two sets are genuinely INDEPENDENT.

   ASK: "If I add a value to list A, must I add it paired with EVERY
         value currently in list B?"
     YES → ★ independent → it's an MVD → 4NF violation → decompose
     NO  → the combination carries information → it is NOT an MVD →
           the table is CORRECT and must NOT be decomposed

   ⇒ THIS IS THE ENTIRE DECISION, and it is a domain question, not a
     data question. Getting it wrong in either direction is expensive.
```

### The decomposition

```
 IF X ↠ Y (and therefore X ↠ Z) and X is not a superkey:
   R1 = (X, Y)
   R2 = (X, Z)

 ★ ALWAYS LOSSLESS: R1 ∩ R2 = X, and X ↠ Y is exactly the condition
   under which the join reconstructs R with no spurious rows.
   (This is Fagin's theorem: R decomposes losslessly into (X,Y) and
    (X,Z) if and only if X ↠ Y holds.)

 APPLIED:
   chef_capabilities(chef, cuisine, shift)
     chef ↠ cuisine,  chef ↠ shift
     chef is not a superkey (the key is all three columns)
   ⇒ chef_cuisines(chef, cuisine)
     chef_shifts(chef, shift)
   ⇒ 20 rows become 9.
```

### The false-MVD trap — when decomposing destroys information

```
 ✗ WHEN THE COMBINATION MATTERS, THERE IS NO MVD.

 product_variants(product_id, size, colour, sku, stock_qty)
   A T-shirt in Medium/Red is a DIFFERENT physical item from
   Medium/Blue. The combination has its own SKU and its own stock.
   ⇒ NOT independent. NOT an MVD. NOT a 4NF violation.
   ⇒ ★ DECOMPOSING WOULD DESTROY INFORMATION: you could no longer
     say which combinations actually exist, or how many of each.

 ✓ VS THE TRUE MVD:
 product_options(product_id, available_size, available_colour)
   IF every colour is genuinely offered in every size, and there is
   no per-combination fact, THEN it is an MVD.
   ⇒ but this is RARE in retail, because stock is per-combination.

 ★ THE DIAGNOSTIC:
   Is the table's row count EXACTLY the product of the two list
   lengths, for every key, always?
     YES, by rule → an MVD
     NO, or "usually but not always" → NOT an MVD. The gaps carry
                                        information. Leave it alone.
```

---

## Concept breakdown

```
MULTI-VALUED DEPENDENCY  X ↠ Y
│  └── for a given X, the set of Y values is INDEPENDENT of the
│      remaining attributes Z. The table must contain the full
│      cross product.
│
├── MVDs COME IN PAIRS: X ↠ Y implies X ↠ Z
├── EVERY FD IS AN MVD  ⇒ 4NF implies BCNF
├── TRIVIAL when Y ⊆ X or X ∪ Y = R
│   ⇒ a TWO-COLUMN table can never violate 4NF
└── ★ requires THREE OR MORE columns and two INDEPENDENT lists

FOURTH NORMAL FORM
└── BCNF + every non-trivial MVD has a superkey determinant.
    Practical form: no table holds two independent multi-valued
    facts about the same key.

THE SYMPTOM
└── ★ ROW COUNT IS A PRODUCT, NOT A SUM.
    m cuisines × n shifts = m×n rows for m+n facts.
    Adding one value to either list adds N rows.

THE DECOMPOSITION
└── R1 = (X, Y), R2 = (X, Z). Always lossless (Fagin's theorem).

★★★ THE TEST — and it is a DOMAIN question, not a data question:
    "If I add a value to list A, must it pair with EVERY value in B?"
      YES → independent → MVD → decompose
      NO  → the combination carries information → NOT an MVD →
            LEAVE IT ALONE
    ⇒ decomposing a false MVD DESTROYS INFORMATION.

HOW COMMON IS IT?
  Rarer than 3NF violations, more common than BCNF ones. The shape —
  two independent lists sharing a key — does occur, especially in
  capability, availability, and compatibility tables.
```

---

## Diagrams

**Diagram 1 — big picture: the cross product**

```
 chef_capabilities(chef, cuisine, shift)

              CUISINES                    SHIFTS
              ┌─────────┐                 ┌──────────┐
              │ Italian │                 │ morning  │
              │ Thai    │                 │ afternoon│
     Arjun ───│ Mexican │        ×        │ evening  │
              └─────────┘                 │ night    │
                  (3)                     └──────────┘
                                              (4)
                          ▼
                   ★ 3 × 4 = 12 ROWS
                     to express 3 + 4 = 7 facts

 ⇒ DECOMPOSE ALONG THE INDEPENDENCE:
   ┌──────────────────────┐        ┌──────────────────────┐
   │ chef_cuisines        │        │ chef_shifts          │
   │ chef │ cuisine       │        │ chef │ shift         │
   ├──────┼───────────────┤        ├──────┼───────────────┤
   │ Arjun│ Italian       │        │ Arjun│ morning       │
   │ Arjun│ Thai          │        │ Arjun│ afternoon     │
   │ Arjun│ Mexican       │        │ Arjun│ evening       │
   └──────┴───────────────┘        │ Arjun│ night         │
            (3 rows)               └──────┴───────────────┘
                                            (4 rows)
                          = 7 rows total
```

**Diagram 2 — data flow: adding one fact**

```
  VIOLATING 4NF                        4NF
  ──────────────────────────           ──────────────────────────
  "Arjun learns Mexican"               "Arjun learns Mexican"
          │                                     │
  must insert ONE ROW PER SHIFT          INSERT INTO chef_cuisines
          │                               VALUES ('Arjun','Mexican')
  ┌───────┴──────────────┐                      │
  │ (Arjun,Mexican,morning)   │                 ▼
  │ (Arjun,Mexican,afternoon) │          ┌─────────────┐
  │ (Arjun,Mexican,evening)   │          │ ONE ROW     │
  │ (Arjun,Mexican,night)     │          │             │
  │                           │          │ ★ partial   │
  │ ★ MISS ONE and the table  │          │  insertion  │
  │   says he cooks Mexican   │          │  is not     │
  │   only in the morning —   │          │  POSSIBLE   │
  │   a claim nobody made.    │          └─────────────┘
  │   And it is a LEGAL state.│
  └───────────────────────────┘
```

**Diagram 3 — before/after: the true MVD vs the false one**

```
 ✓ TRUE MVD — decompose
 chef_capabilities(chef, cuisine, shift)
 ┌──────────────────────────────────────────────────────────────┐
 │ Is (Italian, morning) a THING?  No — it's just a combination  │
 │ of two independent facts. It has no SKU, no stock, no id.     │
 │ Adding a cuisine REQUIRES adding it for every shift.          │
 │ ⇒ MVD. Decompose to (chef,cuisine) + (chef,shift).            │
 └──────────────────────────────────────────────────────────────┘

 ✗ FALSE MVD — do NOT decompose
 product_variants(product_id, size, colour, sku, stock_qty)
 ┌──────────────────────────────────────────────────────────────┐
 │ Is (Medium, Red) a THING?  ★ YES — it has its own SKU, its    │
 │ own stock level, its own barcode. It can be discontinued      │
 │ independently. Medium/Red can exist while Medium/Blue doesn't.│
 │ ⇒ NOT an MVD. The combination IS the entity.                  │
 │ ⇒ DECOMPOSING WOULD DESTROY: which variants exist, and how    │
 │   many of each are in stock.                                  │
 └──────────────────────────────────────────────────────────────┘

 ★ THE QUESTION THAT SEPARATES THEM:
   "Does the COMBINATION have any fact of its own?"
     no  → MVD → decompose
     yes → an entity → leave it, and give it a primary key
```

---

## Example 1 — basic

**Step 1 — build the violating table.**

```sql
CREATE TABLE chef_capabilities (
  chef    text NOT NULL,
  cuisine text NOT NULL,
  shift   text NOT NULL,
  PRIMARY KEY (chef, cuisine, shift)
);

INSERT INTO chef_capabilities VALUES
 ('Arjun','Italian','morning'),('Arjun','Italian','evening'),
 ('Arjun','Thai','morning'),   ('Arjun','Thai','evening'),
 ('Meera','French','afternoon'),('Meera','French','night'),
 ('Meera','Japanese','afternoon'),('Meera','Japanese','night');
```

**Step 2 — see the cross product.**

```sql
SELECT chef,
       count(DISTINCT cuisine) AS cuisines,
       count(DISTINCT shift)   AS shifts,
       count(*)                AS rows,
       count(DISTINCT cuisine) * count(DISTINCT shift) AS product,
       count(DISTINCT cuisine) + count(DISTINCT shift) AS facts
FROM chef_capabilities GROUP BY 1;
```
```
 chef  | cuisines | shifts | rows | product | facts
-------+----------+--------+------+---------+-------
 Arjun |        2 |      2 |    4 |       4 |     4
 Meera |        2 |      2 |    4 |       4 |     4
```
**`rows = product` exactly.** That equality, holding for every key, is the signature of an MVD.

**Step 3 — the anomalies.**

```sql
-- ① INSERT: Arjun learns Mexican — must add one row per shift
INSERT INTO chef_capabilities VALUES ('Arjun','Mexican','morning');
-- forgot the evening row:
SELECT chef, cuisine, count(DISTINCT shift) AS shifts_for_cuisine
FROM chef_capabilities WHERE chef='Arjun' GROUP BY 1,2;
```
```
 chef  | cuisine | shifts_for_cuisine
-------+---------+--------------------
 Arjun | Italian |                  2
 Arjun | Mexican |                  1     ★ INCONSISTENT
 Arjun | Thai    |                  2
```
**The table now asserts that Arjun cooks Mexican only in the morning** — which was never true. And it is a perfectly legal state; no constraint can prevent it.

```sql
-- ② DELETE: Arjun stops working evenings — one row per cuisine
DELETE FROM chef_capabilities WHERE chef='Arjun' AND shift='evening';
-- 2 rows. Miss one and the same inconsistency appears.

-- ③ the constraint you want is inexpressible
ALTER TABLE chef_capabilities ADD CONSTRAINT ck_full_cross CHECK (
  (SELECT count(*) FROM chef_capabilities c2 WHERE c2.chef = chef)
  = (SELECT count(DISTINCT cuisine) * count(DISTINCT shift)
       FROM chef_capabilities c3 WHERE c3.chef = chef));
-- ERROR: cannot use subquery in check constraint            (Topic 24)
```

**Step 4 — decompose to 4NF.**

```sql
DROP TABLE chef_capabilities;
CREATE TABLE chef_cuisines (
  chef    text NOT NULL,
  cuisine text NOT NULL,
  PRIMARY KEY (chef, cuisine)
);
CREATE TABLE chef_shifts (
  chef  text NOT NULL,
  shift text NOT NULL,
  PRIMARY KEY (chef, shift)
);

INSERT INTO chef_cuisines VALUES ('Arjun','Italian'),('Arjun','Thai'),
                                 ('Meera','French'),('Meera','Japanese');
INSERT INTO chef_shifts VALUES ('Arjun','morning'),('Arjun','evening'),
                               ('Meera','afternoon'),('Meera','night');
```

```sql
-- ① Arjun learns Mexican: ONE row, and no inconsistency is possible
INSERT INTO chef_cuisines VALUES ('Arjun','Mexican');

-- ② Arjun stops evenings: ONE row
DELETE FROM chef_shifts WHERE chef='Arjun' AND shift='evening';

-- and the original view is still available
SELECT c.chef, c.cuisine, s.shift
FROM chef_cuisines c JOIN chef_shifts s ON s.chef=c.chef
ORDER BY 1,2,3;
```
```
 chef  | cuisine | shift
-------+---------+---------
 Arjun | Italian | morning
 Arjun | Mexican | morning
 Arjun | Thai    | morning
 Meera | French  | afternoon
 …
```
**The cross product is reconstructed by the join — losslessly, and always consistently.**

**Step 5 — prove losslessness.**

```sql
-- rebuild the original and compare
CREATE TEMP TABLE reconstructed AS
SELECT c.chef, c.cuisine, s.shift FROM chef_cuisines c JOIN chef_shifts s ON s.chef=c.chef;

CREATE TEMP TABLE original AS
SELECT * FROM (VALUES
 ('Arjun','Italian','morning'),('Arjun','Thai','morning'),
 ('Arjun','Mexican','morning'),
 ('Meera','French','afternoon'),('Meera','French','night'),
 ('Meera','Japanese','afternoon'),('Meera','Japanese','night')
) v(chef,cuisine,shift);

SELECT (SELECT count(*) FROM reconstructed) AS reconstructed,
       (SELECT count(*) FROM original) AS original,
       (SELECT count(*) FROM (SELECT * FROM reconstructed
                              EXCEPT SELECT * FROM original) x) AS spurious,
       (SELECT count(*) FROM (SELECT * FROM original
                              EXCEPT SELECT * FROM reconstructed) x) AS lost;
```
```
 reconstructed | original | spurious | lost
---------------+----------+----------+------
             7 |        7 |        0 |    0     ★ lossless
```

**Step 6 — the false MVD, and why decomposing would be wrong.**

```sql
CREATE TABLE product_variants (
  product_id bigint NOT NULL,
  size text NOT NULL,
  colour text NOT NULL,
  sku text NOT NULL UNIQUE,          -- ★ the combination has its own identity
  stock_qty int NOT NULL,            -- ★ and its own fact
  PRIMARY KEY (product_id, size, colour)
);
INSERT INTO product_variants VALUES
 (88,'M','Red','TS-88-M-RED',12),
 (88,'M','Blue','TS-88-M-BLU',0),
 (88,'L','Red','TS-88-L-RED',5);
 -- ★ note: (L, Blue) does NOT exist. The cross product is INCOMPLETE.

SELECT product_id, count(DISTINCT size) AS sizes, count(DISTINCT colour) AS colours,
       count(*) AS rows, count(DISTINCT size)*count(DISTINCT colour) AS product
FROM product_variants GROUP BY 1;
```
```
 product_id | sizes | colours | rows | product
------------+-------+---------+------+---------
         88 |     2 |       2 |    3 |       4     ★ rows ≠ product
```
**`rows < product` — and the gap carries information: (L, Blue) is not made.** Decomposing into `(product, size)` and `(product, colour)` would invent it, and lose both the SKU and the stock level.

**Step 7 — measure the explosion at scale.**

```sql
CREATE TABLE cap_bad AS
SELECT 'chef'||c AS chef, cu AS cuisine, sh AS shift
FROM generate_series(1,2000) c,
     unnest(ARRAY['Italian','Thai','French','Japanese','Mexican']) cu,
     unnest(ARRAY['morning','afternoon','evening','night']) sh;

CREATE TABLE cap_cuisines AS SELECT DISTINCT chef, cuisine FROM cap_bad;
CREATE TABLE cap_shifts   AS SELECT DISTINCT chef, shift   FROM cap_bad;

SELECT 'violating' AS design, (SELECT count(*) FROM cap_bad) AS rows,
       pg_size_pretty(pg_relation_size('cap_bad')) AS size
UNION ALL
SELECT '4NF', (SELECT count(*) FROM cap_cuisines)+(SELECT count(*) FROM cap_shifts),
       pg_size_pretty(pg_relation_size('cap_cuisines')+pg_relation_size('cap_shifts'));
```
```
   design   | rows  |  size
------------+-------+--------
 violating  | 40000 | 2208 kB
 4NF        | 18000 |  968 kB      ← 2.2× fewer rows, 2.3× less storage
```

Now add a sixth cuisine to everyone:
```sql
-- violating: 4 rows per chef
INSERT INTO cap_bad SELECT 'chef'||c,'Korean',sh
FROM generate_series(1,2000) c, unnest(ARRAY['morning','afternoon','evening','night']) sh;
-- 8,000 rows

-- 4NF: 1 row per chef
INSERT INTO cap_cuisines SELECT 'chef'||c,'Korean' FROM generate_series(1,2000) c;
-- 2,000 rows
```
**8,000 vs 2,000 for the same fact.** The multiplication is the cost.

---

## Example 2 — production scenario

**The situation.** A logistics platform. The `driver_capabilities` table has 84 million rows and is growing faster than the driver count.

```sql
CREATE TABLE driver_capabilities (
  driver_id      bigint NOT NULL,
  vehicle_type   text   NOT NULL,   -- 'van','truck','bike','refrigerated'
  region_code    text   NOT NULL,   -- 'BLR','BOM','DEL',…
  certification  text   NOT NULL,   -- 'hazmat','fragile','cold_chain',…
  PRIMARY KEY (driver_id, vehicle_type, region_code, certification)
);
```

**Three independent lists in one table.** 84 million rows for 40,000 drivers.

**Step 1 — confirm the MVDs with the cross-product test.**

```sql
SELECT driver_id,
       count(DISTINCT vehicle_type)  AS v,
       count(DISTINCT region_code)   AS r,
       count(DISTINCT certification) AS c,
       count(*) AS actual_rows,
       count(DISTINCT vehicle_type)*count(DISTINCT region_code)
         *count(DISTINCT certification) AS cross_product
FROM driver_capabilities GROUP BY 1
ORDER BY actual_rows DESC LIMIT 5;
```
```
 driver_id | v | r  | c | actual_rows | cross_product
-----------+---+----+---+-------------+---------------
      4471 | 4 | 12 | 5 |         240 |           240     ★ exact
      8812 | 3 | 14 | 6 |         252 |           252     ★ exact
      1204 | 2 |  8 | 4 |          64 |            64     ★ exact
```

```sql
-- how many drivers show an EXACT cross product?
SELECT count(*) FILTER (WHERE actual = xprod) AS exact_cross_product,
       count(*) AS total_drivers
FROM (SELECT driver_id, count(*) AS actual,
             count(DISTINCT vehicle_type)*count(DISTINCT region_code)
               *count(DISTINCT certification) AS xprod
      FROM driver_capabilities GROUP BY 1) t;
```
```
 exact_cross_product | total_drivers
---------------------+---------------
               40000 |         40000     ★ ALL of them. Three MVDs confirmed.
```

**★ But the data cannot decide this alone.** Confirm with the domain:

> *"If a driver gets a new certification, is it valid for every vehicle type and every region they already cover?"*
> **"Yes — certifications are personal, not per-vehicle or per-region."**

Three independent lists. Three MVDs. Confirmed.

**Step 2 — the anomalies in the business.**

```
 ① INSERT: a driver adds one region.
    ⇒ 4 vehicle types × 5 certifications = 20 rows.
    The onboarding tool does this in a loop. If it fails halfway,
    the driver is available in the new region for only some vehicles.
    ⇒ ★ 412 drivers currently have inconsistent rows from partial
      inserts. The dispatcher sees them as unavailable for jobs they
      can actually do.

 ② DELETE: a certification expires.
    ⇒ 4 × 12 = 48 rows to delete.
    ⇒ ★ 88 drivers have an EXPIRED hazmat certification still present
      for some vehicle/region combinations. They are being dispatched
      to hazmat jobs they are no longer certified for.
      ⇒ THIS IS A SAFETY AND LIABILITY PROBLEM.

 ③ GROWTH: adding a 15th region for all drivers
    ⇒ 40,000 × 4 × 5 = 800,000 rows for one business change.
```

```sql
-- quantify ②
SELECT count(DISTINCT driver_id) AS drivers_with_partial_certs
FROM (SELECT driver_id, certification,
             count(*) AS rows_for_cert,
             (SELECT count(DISTINCT vehicle_type) FROM driver_capabilities d2
               WHERE d2.driver_id = d.driver_id)
             * (SELECT count(DISTINCT region_code) FROM driver_capabilities d3
                 WHERE d3.driver_id = d.driver_id) AS expected
      FROM driver_capabilities d GROUP BY 1,2) t
WHERE rows_for_cert <> expected;
```
```
 drivers_with_partial_certs
----------------------------
                        500     ★ 412 under-available + 88 over-certified
```

**Step 3 — the 4NF decomposition.**

```sql
CREATE TABLE driver_vehicle_types (
  driver_id bigint NOT NULL REFERENCES drivers(id) ON DELETE CASCADE,
  vehicle_type text NOT NULL REFERENCES vehicle_types(code),
  PRIMARY KEY (driver_id, vehicle_type)
);
CREATE TABLE driver_regions (
  driver_id bigint NOT NULL REFERENCES drivers(id) ON DELETE CASCADE,
  region_code text NOT NULL REFERENCES regions(code),
  PRIMARY KEY (driver_id, region_code)
);
CREATE TABLE driver_certifications (
  driver_id bigint NOT NULL REFERENCES drivers(id) ON DELETE CASCADE,
  certification text NOT NULL REFERENCES certifications(code),
  -- ★ AND NOW each certification can carry its own facts —
  --   impossible in the combinatorial table, because the fact would
  --   have had to be repeated 48 times
  issued_at  date NOT NULL,
  expires_at date NOT NULL,
  issuing_body text NOT NULL,
  PRIMARY KEY (driver_id, certification),
  CHECK (expires_at > issued_at)
);
CREATE INDEX ON driver_certifications (expires_at) WHERE expires_at > current_date;
```

**★ The expiry columns are the real win.** In the combinatorial table, `expires_at` would have been a partial dependency (`(driver_id, certification) → expires_at`) repeated 48 times — and it would have drifted, exactly like every other repeated fact. Now expiry is one row, and the partial index makes "whose certification expires this month?" a single index scan.

**Step 4 — the dispatch query, rewritten.**

```sql
-- BEFORE: one row per capability combination
SELECT DISTINCT driver_id FROM driver_capabilities
WHERE vehicle_type='refrigerated' AND region_code='BLR' AND certification='cold_chain';

-- AFTER: three joins, each on a small indexed table
SELECT d.id FROM drivers d
JOIN driver_vehicle_types v ON v.driver_id=d.id AND v.vehicle_type='refrigerated'
JOIN driver_regions r       ON r.driver_id=d.id AND r.region_code='BLR'
JOIN driver_certifications c ON c.driver_id=d.id AND c.certification='cold_chain'
                             AND c.expires_at > current_date;   -- ★ now checkable
```

```sql
EXPLAIN (ANALYZE,BUFFERS) <the BEFORE query>;
-- Bitmap Heap Scan on driver_capabilities … Execution Time: 412 ms

EXPLAIN (ANALYZE,BUFFERS) <the AFTER query>;
-- Nested Loop … Execution Time: 8.4 ms
```
**412 ms → 8.4 ms**, because each join reads a table of 200k rows instead of one of 84M.

**Step 5 — results.**

| | Before | After |
|---|---|---|
| Rows | 84,000,000 | **760,000** (110×) |
| Storage | 6.2 GB | **58 MB** |
| Adding a region for all drivers | 800,000 rows | **40,000 rows** |
| Drivers with partial capability data | 500 | **0, impossible** |
| Expired certifications still active | 88 drivers | **0 — enforced by `expires_at`** |
| Certification expiry tracking | impossible | one column |
| Dispatch query | 412 ms | **8.4 ms** |

**The 88 over-certified drivers is the finding that mattered.** A 4NF violation made it impossible to store an expiry date once, so it was stored 48 times, so it drifted, so drivers were dispatched to hazmat jobs without valid certification.

---

## Common mistakes

**1. Decomposing a false MVD.**
- *Symptom:* you can no longer say which combinations exist, or store per-combination facts.
- *Engine-level why:* the combination was an entity with its own identity (a SKU, a stock level), not a coincidence of two lists.
- *Fix:* the test — "does the combination have any fact of its own?" If yes, it's not an MVD.

**2. Assuming a cross product in the data proves an MVD.**
- *Symptom:* decomposing a table that happens to be complete today.
- *Engine-level why:* like FDs (Topic 30), data can *disprove* an MVD but never *prove* one. A complete cross product might be an accident.
- *Fix:* ask the domain: "must a new value pair with every existing value in the other list?"

**3. Assuming an incomplete cross product disproves an MVD.**
- *Symptom:* leaving a genuine 4NF violation because a few rows are missing.
- *Engine-level why:* the missing rows may be the *anomaly*, not evidence against the MVD.
- *Fix:* check whether the gaps are meaningful or accidental. In Example 2, all 500 gaps were partial-update bugs.

**4. Missing that a two-column table can never violate 4NF.**
- *Symptom:* wasted analysis on junction tables.
- *Fix:* 4NF violations need three or more columns and two independent lists.

**5. Not noticing the multiplicative growth.**
- *Symptom:* a table growing faster than the entity it describes.
- *Fix:* compare row count to entity count over time. Growing as O(m×n) rather than O(m+n) is the signature.

**6. Forgetting that decomposition enables per-item facts.**
- *Symptom:* an expiry date repeated 48 times and drifting.
- *Fix:* after decomposing, ask what facts each list item can now carry. This is often the biggest win, and it's easy to miss.

---

## Hands-on proof

**PROVE IT #1 — the cross-product signature.** (Example 1, step 2.)
**PROVE IT #2 — the partial-insert anomaly.** (Example 1, step 3.)
**PROVE IT #3 — decomposition, and the join that reconstructs it.** (Example 1, step 4.)
**PROVE IT #4 — losslessness, verified.** (Example 1, step 5.)
**PROVE IT #5 — the false MVD.** (Example 1, step 6.)

**PROVE IT #6 — detect MVD candidates across a schema.**
```sql
-- for any 3+ column table with an all-column PK, compare rows to the
-- cross product per key
CREATE OR REPLACE FUNCTION mvd_check(tbl text, key_col text, a text, b text)
RETURNS TABLE(keys bigint, exact_cross bigint, pct numeric) AS $$
BEGIN
  RETURN QUERY EXECUTE format($f$
    SELECT count(*)::bigint,
           count(*) FILTER (WHERE actual = xprod)::bigint,
           round(100.0*count(*) FILTER (WHERE actual = xprod)/count(*),1)
    FROM (SELECT %I, count(*) AS actual,
                 count(DISTINCT %I)*count(DISTINCT %I) AS xprod
          FROM %I GROUP BY 1) t
  $f$, key_col, a, b, tbl);
END $$ LANGUAGE plpgsql;

SELECT * FROM mvd_check('chef_capabilities','chef','cuisine','shift');
```
```
 keys | exact_cross |  pct
------+-------------+-------
    2 |           2 | 100.0     ★ 100% ⇒ investigate as an MVD
```
```
 INTERPRETING pct:
   100%      → a strong MVD candidate. CONFIRM WITH THE DOMAIN.
   90–99%    → probably an MVD with partial-update anomalies (the gaps
               are bugs — exactly Example 2's case)
   < 50%     → NOT an MVD. The combinations carry information.
```

**PROVE IT #7 — the multiplicative growth.**
```sql
-- add one value to one list and count the rows required
SELECT count(DISTINCT driver_id) * count(DISTINCT vehicle_type)
       * count(DISTINCT certification) AS rows_needed_for_one_new_region
FROM driver_capabilities;
```

---

## The design decision framework

```
THE CHECK — for every table with 3+ columns where the PK is all of them:

 ① Does it hold TWO OR MORE multi-valued facts about the same key?
    (two lists that both hang off one entity)
    NO → 4NF holds. Stop.

 ② Run the cross-product test:
      SELECT key, count(*) AS actual,
             count(DISTINCT a)*count(DISTINCT b) AS xprod
      FROM t GROUP BY 1;
    • actual = xprod for ~all keys → an MVD candidate
    • actual < xprod for most keys → probably NOT an MVD

 ③ ★★★ THE DECIDING QUESTION — ask the DOMAIN, not the data:
      "If I add a value to list A, must it apply to EVERY value
       currently in list B?"
        YES → independent → MVD → decompose
        NO  → the combination carries information → NOT an MVD →
              ★ LEAVE IT ALONE

 ④ CONFIRM with the second question:
      "Does the COMBINATION have any fact of its own?"
        (a SKU, a stock level, a price, an id, a status)
        yes → it is an ENTITY. Not an MVD. Give it a proper key.
        no  → an MVD.

 ⑤ DECOMPOSE: R1 = (X, Y), R2 = (X, Z). Always lossless.

 ⑥ ★ THEN ASK WHAT EACH LIST CAN NOW CARRY.
    This is usually the biggest win: a certification can have an
    expiry date; a region can have a start date; a vehicle type can
    have a licence class. None of these were possible when the fact
    would have had to be repeated m×n times.

THE SIGNAL TO LOOK FOR:
  A table growing MULTIPLICATIVELY rather than additively.
      SELECT count(*) AS rows, count(DISTINCT key_col) AS entities,
             round(count(*)::numeric/count(DISTINCT key_col)) AS rows_per_entity
      FROM t;
  • rows_per_entity in the hundreds, for an entity with a handful of
    real attributes → ★ investigate for an MVD
  • and if rows_per_entity is growing over time while the entity count
    is flat → almost certainly one

⚠ THE ASYMMETRY OF THE MISTAKE:
  Failing to decompose a true MVD costs you redundancy and anomalies —
  recoverable.
  Decomposing a FALSE MVD DESTROYS INFORMATION — which combinations
  existed, and any per-combination fact. That is not recoverable.
  ⇒ ★ WHEN IN DOUBT, DO NOT DECOMPOSE. Ask the domain first.
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
Build `chef_capabilities`. Run the cross-product test and show `rows = product` for every chef. Produce the partial-insert anomaly. Decompose to 4NF, and verify losslessness by rebuilding the original with a join and comparing with `EXCEPT` in both directions.

### Exercise 2 — medium (apply it)
For each table, decide whether an MVD exists. Justify with **both** the cross-product test and the domain question, and state what decomposing would cost if you were wrong:
(a) `student_activities(student_id, sport, club)` · (b) `product_variants(product_id, size, colour, sku, stock)` · (c) `employee_skills(emp_id, language, framework)` · (d) `flight_crew(flight_id, pilot_id, cabin_crew_id)` · (e) `restaurant_hours(restaurant_id, day_of_week, cuisine_served)`

### Exercise 3 — hard (production simulation)
A `driver_capabilities` table has 84M rows for 40,000 drivers, with columns `(driver_id, vehicle_type, region_code, certification)` and an all-column PK. An audit finds 412 drivers with partial capability data and **88 drivers with expired certifications still active**.

(a) Write the query proving all three lists are independent, and explain why the data alone cannot confirm it.
(b) Write the domain questions you would ask, and say what answer would *disprove* each MVD.
(c) Quantify the anomalies with SQL, distinguishing under-availability from over-certification.
(d) Explain how a 4NF violation caused the certification-expiry safety problem specifically.
(e) Design the 4NF decomposition, including the facts each list can now carry.
(f) Rewrite the dispatch query and estimate the improvement.
(g) One of the three lists might *not* be independent — a driver might be certified for hazmat only in certain vehicle types. Explain how you would find out, and how the design changes if so.
(h) Give the migration plan (Topic 28), including how you resolve the 500 inconsistent drivers — which is not a mechanical deduplication.

---

## Mental model checkpoint

1. Define a multi-valued dependency. Why do MVDs always come in pairs?
2. What is the visual signature of a 4NF violation in the data?
3. Why can a two-column table never violate 4NF?
4. Give the two questions that distinguish a true MVD from a false one.
5. Why is decomposing a *false* MVD worse than failing to decompose a true one?
6. Every FD is an MVD. What does that imply about the relationship between 4NF and BCNF?
7. After decomposing, what is the win that people usually forget to look for?

---

## Quick reference card

**MVD `X ↠ Y`:** for a given X, the set of Y values is independent of the remaining attributes. The table must contain the full cross product.

**4NF:** BCNF + every non-trivial MVD has a superkey determinant.
**Practical form:** no table holds two independent multi-valued facts about the same key.

| | |
|---|---|
| Signature | `rows = count(DISTINCT a) × count(DISTINCT b)` per key |
| Requires | 3+ columns, two independent lists |
| Two-column table | can never violate 4NF |
| Every FD is an MVD | ⇒ 4NF implies BCNF |
| Decomposition | `R1 = (X,Y)`, `R2 = (X,Z)` — always lossless (Fagin) |

**★ The two tests**

1. *"If I add a value to list A, must it pair with every value in B?"* — yes → MVD.
2. *"Does the combination have a fact of its own?"* — yes → an entity, **not** an MVD.

**Interpreting the cross-product percentage**

| % of keys with exact cross product | Meaning |
|---|---|
| 100% | strong MVD candidate — confirm with the domain |
| 90–99% | an MVD with partial-update anomalies (the gaps are bugs) |
| < 50% | not an MVD — combinations carry information |

**⚠ Asymmetry:** failing to decompose a true MVD costs redundancy (recoverable). Decomposing a false MVD **destroys information** (not recoverable). **When in doubt, don't.**

---

## When would I use this at work?

1. **A table growing faster than the entity it describes.** 84 million rows for 40,000 drivers is a multiplicative signature, and one `GROUP BY` confirms it.

2. **Capability, availability, or compatibility tables.** These are where independent lists genuinely collide — driver skills, product options, part compatibility. The cross-product test plus one domain question settles it.

3. **When a per-item fact has nowhere to live.** "Where do I put the certification's expiry date?" has no good answer in a combinatorial table — and that question is often what reveals the 4NF violation in the first place.

---

## Connected topics

**Understand before this:** 30 (dependencies, lossless decomposition), 34 (BCNF — 4NF's precondition), 31 (1NF and multi-valued attributes, the simpler cousin).

**This unlocks:**
- **36** — 5NF and join dependencies: the final generalisation
- **37** — the worked example
- **38** — when to stop, including why 4NF is worth checking but 5NF usually isn't
- **20** — ER modelling, where two independent 1:N relationships should have been two tables from the start
