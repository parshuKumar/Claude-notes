# 37 — Normalisation: The Full Worked Example
## Phase: Normalisation

---

## ELI5 — The Simple Analogy

Someone hands you a spreadsheet.

It has forty columns. Every row is one line of one order. The customer's name is on it. So is their phone number, their city, the product name, the product's category, the category's manager, the warehouse, the warehouse's address, the courier, the courier's rate card, and three columns called `tag1`, `tag2`, `tag3`.

It works. People have been running the business on it for four years. And it is wrong in eleven distinct ways, each of which you now have a name for.

This topic is the **whole procedure, start to finish, on one table** — no new theory, just the six forms applied in order, with every dependency written down, every decomposition verified, and every judgement call made explicitly.

By the end you will have done it once. That is the point.

---

## Where this fits in the big picture

```
   29 anomalies · 30 FDs · 31 1NF · 32 2NF · 33 3NF
   34 BCNF · 35 4NF · 36 5NF
                          │
                          ▼
        ┌──────────────────────────────────────────┐
        │ 37 THE WORKED EXAMPLE    ← YOU ARE HERE  │
        │ all of it, on one messy table, in order  │
        └────────────────────┬─────────────────────┘
                             ▼
                    38 when to stop  ← PHASE 4 ENDS
```

Everything here is applied theory. **No new concepts.** If a step confuses you, the topic that explains it is named inline.

---

## What is this?

One realistic, messy table taken through the complete procedure:

```
 ① write down the FDs and MVDs, from the DOMAIN
 ② find the candidate keys
 ③ 1NF  — atomicity
 ④ 2NF  — no partial dependency
 ⑤ 3NF  — no transitive dependency
 ⑥ BCNF — every determinant is a key
 ⑦ 4NF  — no independent multi-valued facts
 ⑧ 5NF  — check, and stop
 ⑨ VERIFY every decomposition is lossless
 ⑩ decide which "violations" are deliberate snapshots
 ⑪ write the final DDL with every constraint the analysis earned
```

---

## Why does it matter for a backend developer?

Because the individual forms are easy in isolation and hard in combination. The real skills only appear when you do the whole thing:

```
 ① ORDER MATTERS. Fixing 1NF changes the candidate keys, which
    changes what counts as a 2NF violation. Do them out of order and
    you will find violations that aren't there and miss ones that are.

 ② RE-CHECK AFTER EVERY STEP. A decomposition can create a new
    violation in one of the pieces. Chains are common.

 ③ ★ THE JUDGEMENT CALLS ARE THE JOB.
    Three of the eleven "violations" in this table are NOT violations —
    they are snapshots, temporal facts, or entities. Removing them
    would destroy information. Recognising which is which is the
    entire difference between normalising and vandalising.

 ④ THE OUTPUT IS CONSTRAINTS, NOT TABLES.
    You will finish with 11 tables — and 34 enforceable business rules
    that the original schema could not express at all.
```

---

## The physical reality

### The starting table

```sql
CREATE TABLE order_master (
  -- identity
  order_id        bigint      NOT NULL,
  line_no         smallint    NOT NULL,
  order_date      timestamp   NOT NULL,          -- ⚠ no time zone (T26)
  -- customer
  customer_id     bigint      NOT NULL,
  customer_name   text        NOT NULL,
  customer_email  text        NOT NULL,
  customer_phones text        NOT NULL,          -- ⚠ '98765-43210,91234-56789'
  customer_city   text        NOT NULL,
  customer_state  text        NOT NULL,
  customer_pin    text        NOT NULL,
  -- product
  product_id      bigint      NOT NULL,
  product_sku     text        NOT NULL,
  product_name    text        NOT NULL,
  product_price   float8      NOT NULL,          -- ⚠ float money (T26)
  category_code   text        NOT NULL,
  category_name   text        NOT NULL,
  category_manager_id bigint  NOT NULL,
  category_manager_name text  NOT NULL,
  -- the line
  quantity        int         NOT NULL,
  unit_price      float8      NOT NULL,
  line_total      float8      NOT NULL,
  -- fulfilment
  warehouse_code  text        NOT NULL,
  warehouse_city  text        NOT NULL,
  warehouse_mgr   text        NOT NULL,
  courier_code    text        NOT NULL,
  courier_name    text        NOT NULL,
  courier_rate_per_kg float8  NOT NULL,
  -- tags
  tag1 text, tag2 text, tag3 text,               -- ⚠ repeating group (T31)
  PRIMARY KEY (order_id, line_no)
);
```

**40 columns. 8 million rows. 620 GB.** Let's take it apart.

---

## How it works — step by step

### STEP ① — Write down the dependencies, from the domain

```
 ★ THESE COME FROM THE BUSINESS, NOT FROM THE DATA (Topic 30).
   Each one was confirmed with the operations team.

 FUNCTIONAL DEPENDENCIES
  f1  order_id → order_date, customer_id
  f2  customer_id → customer_name, customer_email, customer_city,
                    customer_state, customer_pin
  f3  customer_pin → customer_city, customer_state     ★ note this one
  f4  product_id → product_sku, product_name, product_price, category_code
  f5  product_sku → product_id                          ★ and this one
  f6  category_code → category_name, category_manager_id
  f7  category_manager_id → category_manager_name
  f8  (order_id, line_no) → product_id, quantity, unit_price, line_total
  f9  order_id → warehouse_code, courier_code
  f10 warehouse_code → warehouse_city, warehouse_mgr
  f11 courier_code → courier_name, courier_rate_per_kg
  f12 (quantity, unit_price) → line_total               ★ derived

 MULTI-VALUED DEPENDENCIES
  m1  customer_id ↠ phone         (a customer has several phones)
  m2  order_id ↠ tag              (an order has several tags)
  ⚠ are m1 and m2 in the SAME table? No — phones belong to the
    customer, tags to the order. They are not independent lists of the
    same key, so this is not the classic 4NF shape. But both are 1NF
    violations. (Topic 31.)
```

### STEP ② — Find the candidate keys

```
 Attributes appearing ONLY on the left: line_no
   ⇒ must be in every candidate key
 Attributes appearing ONLY on the right: everything except
   order_id, customer_id, customer_pin, product_id, product_sku,
   category_code, category_manager_id, warehouse_code, courier_code,
   quantity, unit_price

 L = {line_no}.  L⁺ = {line_no}. Not a superkey.

 Try {order_id, line_no}:
   + f1  → order_date, customer_id
   + f2  → customer_name, email, city, state, pin
   + f8  → product_id, quantity, unit_price, line_total
   + f4  → product_sku, product_name, product_price, category_code
   + f6  → category_name, category_manager_id
   + f7  → category_manager_name
   + f9  → warehouse_code, courier_code
   + f10 → warehouse_city, warehouse_mgr
   + f11 → courier_name, courier_rate_per_kg
   = ALL   ⇒ ★ SUPERKEY, and minimal ⇒ CANDIDATE KEY

 ⇒ ONE candidate key: {order_id, line_no}
 ⇒ PRIME: order_id, line_no
 ⇒ NON-PRIME: everything else (36 attributes)
```

### STEP ③ — 1NF (Topic 31)

```
 VIOLATION A: customer_phones = '98765-43210,91234-56789'
   → multi-valued attribute in a cell
 VIOLATION B: tag1, tag2, tag3
   → repeating group

 ⚠ AND CHECK: is customer_pin/city/state a composite violation?
   We DO query by city and state ⇒ they must stay separate columns.
   They already are ⇒ ✓ no violation.

 FIX — each multi-valued attribute becomes its own table:
   customer_phones(customer_id, phone, kind, is_primary)
   order_tags(order_id, tag)

 ★ RE-CHECK THE CANDIDATE KEY: unchanged. Good — 1NF fixes here
   removed attributes rather than adding determinants.
```

### STEP ④ — 2NF (Topic 32)

```
 The key is composite: {order_id, line_no}. So partial dependencies
 are possible. Test each proper subset.

 {line_no}⁺ = {line_no}                      → determines nothing ✓
 {order_id}⁺ = order_date, customer_id, customer_name, email, city,
               state, pin, warehouse_code, warehouse_city,
               warehouse_mgr, courier_code, courier_name,
               courier_rate_per_kg
   ⇒ ★ ALL NON-PRIME. A massive PARTIAL DEPENDENCY.

 FIX — split along the determinant:
   orders(order_id PK, order_date, customer_id, warehouse_code, courier_code,
          + everything order_id determines transitively — handled in 3NF)
   order_lines(order_id, line_no, product_id, quantity, unit_price, line_total)

 ⇒ RESULT AFTER 2NF:
   orders(order_id, order_date, customer_id, customer_name, customer_email,
          customer_city, customer_state, customer_pin,
          warehouse_code, warehouse_city, warehouse_mgr,
          courier_code, courier_name, courier_rate_per_kg)
   order_lines(order_id, line_no, product_id, product_sku, product_name,
               product_price, category_code, category_name,
               category_manager_id, category_manager_name,
               quantity, unit_price, line_total)
   customer_phones(customer_id, phone, …)
   order_tags(order_id, tag)

 ★ LOSSLESS CHECK (Topic 30):
   orders ∩ order_lines = {order_id}, a KEY of orders  ✓

 ★ RE-CHECK: both new tables now have SINGLE-attribute candidate keys
   ({order_id} and {order_id, line_no} respectively — the second is
   still composite). No further partial dependencies. 2NF holds.
```

### STEP ⑤ — 3NF (Topic 33)

```
 Now the transitive dependencies. For each FD, is the determinant a
 superkey of ITS table, or is the RHS prime?

 IN `orders` (key = {order_id}):
   f2  customer_id → name, email, city, state, pin
       customer_id not a superkey, RHS non-prime ⇒ ★ VIOLATION
   f10 warehouse_code → warehouse_city, warehouse_mgr
       ⇒ ★ VIOLATION
   f11 courier_code → courier_name, courier_rate_per_kg
       ⇒ ★ VIOLATION

 IN `order_lines` (key = {order_id, line_no}):
   f4  product_id → sku, name, price, category_code   ⇒ ★ VIOLATION
   f6  category_code → category_name, manager_id      ⇒ ★ VIOLATION (2 steps)
   f7  category_manager_id → manager_name             ⇒ ★ VIOLATION (3 steps)

 ★ NOTE THE CHAINS:
   order_lines → product_id → category_code → manager_id → manager_name
   That is a FOUR-LINK chain. You must re-check after each split.

 FIX — one table per determinant:
   customers(customer_id PK, name, email, city, state, pin)
   warehouses(warehouse_code PK, city, manager)
   couriers(courier_code PK, name, rate_per_kg)
   products(product_id PK, sku, name, price, category_code)
   categories(category_code PK, name, manager_id)
   managers(manager_id PK, name)

 ⇒ orders(order_id PK, order_date, customer_id FK, warehouse_code FK,
          courier_code FK)
   order_lines(order_id, line_no, product_id FK, quantity, unit_price,
               line_total)

 ★ RE-CHECK EACH NEW TABLE:
   customers: any FD whose determinant isn't {customer_id}?
     ⇒ f3: customer_pin → city, state.  ★ STILL A VIOLATION.
     ⇒ decompose again:
         pincodes(pin PK, city, state)
         customers(customer_id PK, name, email, pin FK)
   ⇒ ★ THIS IS WHY YOU RE-CHECK. f3 was invisible until customers
     became its own table.
```

### STEP ⑥ — BCNF (Topic 34)

```
 Now the stricter test: is EVERY determinant a superkey?

 products(product_id PK, sku, name, price, category_code)
   f4  product_id → …        superkey ✓
   f5  product_sku → product_id
       product_sku is NOT the declared key…
       BUT {product_sku}⁺ = {product_sku, product_id, name, price,
                             category_code} = ALL
       ⇒ ★ product_sku IS A SUPERKEY. It is a second CANDIDATE KEY.
       ⇒ ✓ BCNF holds — and the analysis has just told you that
         `sku` needs a UNIQUE constraint, which nobody had declared.

 pincodes(pin PK, city, state)
   f3  pin → city, state     superkey ✓  BCNF holds

 ⚠ CHECK THE OVERLAP CONDITION (Topic 34): a BCNF violation requires
   two OVERLAPPING candidate keys. products has {product_id} and
   {product_sku} — they do NOT overlap. ⇒ no BCNF violation possible.

 ⇒ ★ ALL TABLES ARE IN BCNF.
   And the analysis produced a missing UNIQUE constraint as a by-product.
```

### STEP ⑦ — 4NF (Topic 35)

```
 Are there two INDEPENDENT multi-valued facts in one table?

 customer_phones(customer_id, phone, kind, is_primary)
   one list. ⇒ ✓ 4NF (a table with one multi-valued fact cannot violate it)

 order_tags(order_id, tag)
   two columns ⇒ ★ can never violate 4NF (Topic 35)

 ⇒ ✓ NO 4NF VIOLATIONS.

 ⚠ BUT ASK ANYWAY: is there anywhere two independent lists could hide?
   Consider: does a warehouse serve multiple couriers, AND multiple
   regions, independently?
     SELECT warehouse_code, count(DISTINCT courier_code) …
   If yes, and they're stored together, that WOULD be a 4NF violation.
   ⇒ In this schema, warehouse↔courier is per-ORDER, not a capability
     list. ✓ No violation.
```

### STEP ⑧ — 5NF (Topic 36)

```
 Any three-way relationship tables?
   order_lines(order_id, line_no, product_id, …) — has extra columns
     ⇒ ★ the combination carries facts (quantity, price). It is an
       ENTITY. Already in 5NF. (Topic 36's anti-tell.)

 ⇒ ✓ NO 5NF VIOLATIONS. Stop.
```

### STEP ⑨ — Verify losslessness at every step

```sql
-- After the 2NF split
WITH rejoined AS (
  SELECT o.order_id, l.line_no, o.order_date, o.customer_id, l.product_id,
         l.quantity, l.unit_price
  FROM orders o JOIN order_lines l USING (order_id))
SELECT (SELECT count(*) FROM order_master_1nf) AS original,
       (SELECT count(*) FROM rejoined) AS rejoined,
       (SELECT count(*) FROM (SELECT order_id,line_no,order_date,customer_id,
                                     product_id,quantity,unit_price
                              FROM rejoined
                              EXCEPT
                              SELECT order_id,line_no,order_date,customer_id,
                                     product_id,quantity,unit_price
                              FROM order_master_1nf) x) AS spurious;
-- ★ spurious MUST be 0 (Topic 36)
```

### STEP ⑩ — The judgement calls

★ **This is the most important step, and it is not mechanical.**

```
 THREE "VIOLATIONS" THAT ARE NOT VIOLATIONS:

 ① order_lines.unit_price  vs  products.price
    LOOKS LIKE: a transitive dependency (product_id → price).
    THE TEST (Topics 29, 33): "if the product's price changes, SHOULD
      this line's price change?"
    ANSWER: ★ NO. The invoice is immutable. This is the price CHARGED.
    ⇒ ★ NOT a violation. It is a SNAPSHOT. Keep it.
    ⇒ AND RENAME IT so nobody re-normalises it later:
        unit_price → unit_price_charged_minor
    ⇒ AND: `products` should have NO price column at all here — prices
      are temporal, so they belong in product_prices with a validity
      range (Topic 26).

 ② order_lines.line_total
    LOOKS LIKE: f12, a derived value — pure redundancy.
    THE TEST: is it computable from other columns of the SAME ROW?
    ANSWER: YES ⇒ it should not be STORED as an ordinary column.
    ⇒ ★ NOT removed — made GENERATED, so it CANNOT disagree (Topic 24):
        line_total_minor bigint GENERATED ALWAYS AS
          (quantity * unit_price_charged_minor) STORED

 ③ customer_pin → city, state (f3)
    We DID decompose this into a `pincodes` table. But check the domain:
    "if a pincode's city is corrected, should the customer's city change?"
    ANSWER: ★ YES for the CURRENT address; NO for a historical one.
    ⇒ the customer's CURRENT address references pincodes ✓
    ⇒ but an ORDER's DELIVERY address must be a SNAPSHOT — you must be
      able to reprint an invoice with the address it was sent to.
    ⇒ ★ SO: add order_addresses, snapshotted at order time.
      This is a NEW TABLE the normalisation revealed, not one it removed.

 ★ ALL THREE ARE FACTS ABOUT THE ROW, NOT COPIES OF ANOTHER TABLE'S
   CURRENT VALUE. The test is always the same:
     "if the source changes, SHOULD this change?"
       yes → a copy. Normalise it away.
       no  → a snapshot. Keep it, and name it so.
```

---

## Concept breakdown

```
THE PROCEDURE
│
├── ① FDs and MVDs from the DOMAIN (not the data)
├── ② candidate keys via closure
├── ③ 1NF   atomicity — do this FIRST; it changes the keys
├── ④ 2NF   partial dependencies (composite keys only)
├── ⑤ 3NF   transitive dependencies ★ RE-CHECK AFTER EACH SPLIT
├── ⑥ BCNF  every determinant a superkey (needs overlapping keys)
├── ⑦ 4NF   two independent lists in one table
├── ⑧ 5NF   three-way join dependency — usually stop before here
├── ⑨ verify LOSSLESSNESS after every step
├── ⑩ ★ the judgement calls: snapshots, derived values, temporal facts
└── ⑪ write the DDL, with every constraint the analysis earned

★ WHY ORDER MATTERS
  1NF changes the attribute set ⇒ changes the candidate keys
  ⇒ changes what counts as a partial dependency
  Doing 2NF before 1NF finds violations that don't exist.

★ WHY YOU RE-CHECK
  f3 (pin → city, state) was INVISIBLE until `customers` became its
  own table. Chains hide inside chains.

★ THE THREE OUTPUTS
  ① tables — the obvious one
  ② ★ CONSTRAINTS — the valuable one (34 rules the original couldn't express)
  ③ ★ DISCOVERIES — a missing UNIQUE on sku; a needed address snapshot
```

---

## Diagrams

**Diagram 1 — big picture: 1 table becomes 11**

```
 order_master (40 columns, 8M rows, 620 GB)
        │
        ├─ 1NF ──▶ customer_phones · order_tags
        │
        ├─ 2NF ──▶ orders │ order_lines
        │
        ├─ 3NF ──▶ customers · warehouses · couriers
        │          products · categories · managers
        │             └─ re-check ──▶ pincodes
        │
        ├─ BCNF ─▶ (no change — but found a missing UNIQUE on sku)
        ├─ 4NF ──▶ (no change)
        ├─ 5NF ──▶ (no change)
        │
        └─ ⑩ ──▶ product_prices (temporal) · order_addresses (snapshot)

 FINAL: 13 tables
 ┌──────────────────────────────────────────────────────────────────┐
 │ pincodes ◀── customers ◀── orders ──▶ warehouses                 │
 │                              │  │  ▲                             │
 │                              │  │  └── couriers                  │
 │                              │  └────▶ order_addresses (snapshot)│
 │                              ▼                                   │
 │                        order_lines ──▶ products ──▶ categories   │
 │                              │              │           │        │
 │                              │              ▼           ▼        │
 │                              │     product_prices    managers    │
 │                              │      (temporal)                   │
 │                        order_tags                                │
 │                        customer_phones ──▶ customers             │
 └──────────────────────────────────────────────────────────────────┘
```

**Diagram 2 — data flow: the four-link chain**

```
  order_lines
       │  product_id
       ▼
   products
       │  category_code
       ▼
  categories
       │  category_manager_id
       ▼
   managers
       │  manager_name
       ▼
     (leaf)

 ★ IN THE ORIGINAL TABLE, ALL FOUR LEVELS WERE INLINE.
   `category_manager_name` was stored once per ORDER LINE —
   8,000,000 times, for 40 managers.
   ⇒ repetition factor: 200,000×

 ⇒ AND YOU ONLY FIND THE LAST LINK BY RE-CHECKING:
   after splitting products out, `categories` is a new table, and
   only THEN is category_code → manager_id visible as a violation
   of that table's 3NF.
```

**Diagram 3 — before/after: what the analysis earned**

```
 BEFORE — one table, 40 columns
 ┌──────────────────────────────────────────────────────────────────┐
 │ ENFORCEABLE CONSTRAINTS: 1  (the primary key)                    │
 │                                                                  │
 │ ✗ "a customer has one email"                                     │
 │ ✗ "a SKU is unique"                                              │
 │ ✗ "a pincode has one city"                                       │
 │ ✗ "a category has one manager"                                   │
 │ ✗ "quantity > 0"          (expressible, but repeated 8M times)   │
 │ ✗ "line_total = qty × price"                                     │
 │ ✗ "a product must exist"                                         │
 │ ✗ … 27 more                                                      │
 └──────────────────────────────────────────────────────────────────┘

 AFTER — 13 tables
 ┌──────────────────────────────────────────────────────────────────┐
 │ ENFORCEABLE CONSTRAINTS: 34                                      │
 │   7 primary keys on natural keys                                 │
 │   6 surrogate primary keys                                       │
 │  11 foreign keys                                                 │
 │   5 unique constraints (incl. the SKU one nobody had declared)   │
 │   4 check constraints                                            │
 │   1 generated column (line_total cannot disagree)                │
 │                                                                  │
 │ STORAGE: 620 GB → 180 GB                                         │
 └──────────────────────────────────────────────────────────────────┘
        ↑ ★ THE CONSTRAINTS ARE THE POINT. The storage is a side effect.
```

---

## Example 1 — basic

The complete run, in SQL you can execute.

**Step 1 — build the mess.**

```sql
CREATE TABLE order_master (
  order_id bigint NOT NULL, line_no smallint NOT NULL,
  order_date timestamp NOT NULL,
  customer_id bigint NOT NULL, customer_name text NOT NULL,
  customer_email text NOT NULL, customer_phones text NOT NULL,
  customer_city text NOT NULL, customer_state text NOT NULL,
  customer_pin text NOT NULL,
  product_id bigint NOT NULL, product_sku text NOT NULL,
  product_name text NOT NULL, product_price float8 NOT NULL,
  category_code text NOT NULL, category_name text NOT NULL,
  category_manager_id bigint NOT NULL, category_manager_name text NOT NULL,
  quantity int NOT NULL, unit_price float8 NOT NULL, line_total float8 NOT NULL,
  warehouse_code text NOT NULL, warehouse_city text NOT NULL,
  warehouse_mgr text NOT NULL,
  courier_code text NOT NULL, courier_name text NOT NULL,
  courier_rate_per_kg float8 NOT NULL,
  tag1 text, tag2 text, tag3 text,
  PRIMARY KEY (order_id, line_no)
);

INSERT INTO order_master VALUES
 (1,1,'2026-03-01 10:00',7,'Arjun','arjun@shop.in','98765-43210,91234-56789',
  'Bengaluru','Karnataka','560001',
  101,'SKU-RICE-5','Basmati Rice 5kg',549.00,'GROC','Groceries',
  501,'Meera Rao',2,549.00,1098.00,
  'WH-BLR','Bengaluru','Ravi K','BLUEDART','Blue Dart',45.00,
  'priority','gift',NULL),
 (1,2,'2026-03-01 10:00',7,'Arjun','arjun@shop.in','98765-43210,91234-56789',
  'Bengaluru','Karnataka','560001',
  102,'SKU-GHEE-1','Ghee 1L',720.00,'GROC','Groceries',
  501,'Meera Rao',1,720.00,720.00,
  'WH-BLR','Bengaluru','Ravi K','BLUEDART','Blue Dart',45.00,
  'priority','gift',NULL),
 (2,1,'2026-03-02 11:30',8,'Meera','meera@shop.in','91111-22222',
  'Mumbai','Maharashtra','400001',
  101,'SKU-RICE-5','Basmati Rice 5kg',549.00,'GROC','Groceries',
  501,'Meera Rao',3,549.00,1647.00,
  'WH-BOM','Mumbai','Sunil P','DELHIVERY','Delhivery',38.00,
  'bulk',NULL,NULL);
```

**Step 2 — verify the FDs against the data (Topic 30).**

```sql
CREATE OR REPLACE FUNCTION fd(tbl text, lhs text, rhs text)
RETURNS TABLE(dependency text, groups bigint, violations bigint) AS $$
BEGIN
  RETURN QUERY EXECUTE format($f$
    SELECT %L, count(*)::bigint, count(*) FILTER (WHERE n>1)::bigint
    FROM (SELECT %s, count(DISTINCT (%s)) n FROM %I GROUP BY %s) t
  $f$, lhs||' -> '||rhs, lhs, rhs, tbl, lhs);
END $$ LANGUAGE plpgsql;

SELECT * FROM fd('order_master','customer_id','customer_email')
UNION ALL SELECT * FROM fd('order_master','customer_pin','customer_city')
UNION ALL SELECT * FROM fd('order_master','product_id','product_name')
UNION ALL SELECT * FROM fd('order_master','category_code','category_manager_id')
UNION ALL SELECT * FROM fd('order_master','warehouse_code','warehouse_city')
UNION ALL SELECT * FROM fd('order_master','product_sku','product_id');
```
```
              dependency               | groups | violations
---------------------------------------+--------+------------
 customer_id -> customer_email         |      2 |          0
 customer_pin -> customer_city         |      2 |          0
 product_id -> product_name            |      2 |          0
 category_code -> category_manager_id  |      1 |          0
 warehouse_code -> warehouse_city      |      2 |          0
 product_sku -> product_id             |      2 |          0
```
**All consistent** — which is what you'd expect in clean sample data. On production data you would find violations, and Topic 30 tells you how to interpret the rate.

**Step 3 — 1NF.**

```sql
CREATE TABLE customer_phones (
  customer_id bigint NOT NULL,
  phone text NOT NULL,
  kind text NOT NULL DEFAULT 'mobile',
  is_primary boolean NOT NULL DEFAULT false,
  PRIMARY KEY (customer_id, phone)
);
INSERT INTO customer_phones (customer_id, phone, is_primary)
SELECT DISTINCT customer_id, trim(p), row_number() OVER (PARTITION BY customer_id) = 1
FROM order_master, unnest(string_to_array(customer_phones, ',')) p;

CREATE TABLE order_tags (order_id bigint NOT NULL, tag text NOT NULL,
                         PRIMARY KEY (order_id, tag));
INSERT INTO order_tags
SELECT DISTINCT order_id, t FROM order_master,
     unnest(ARRAY[tag1,tag2,tag3]) t WHERE t IS NOT NULL;

SELECT * FROM customer_phones ORDER BY 1,2;
```
```
 customer_id |    phone    |  kind  | is_primary
-------------+-------------+--------+------------
           7 | 91234-56789 | mobile | f
           7 | 98765-43210 | mobile | t
           8 | 91111-22222 | mobile | t
```

**Step 4 — 2NF: split on the partial dependency.**

```sql
CREATE TABLE orders_2nf AS
SELECT DISTINCT order_id, order_date, customer_id, customer_name,
       customer_email, customer_city, customer_state, customer_pin,
       warehouse_code, warehouse_city, warehouse_mgr,
       courier_code, courier_name, courier_rate_per_kg
FROM order_master;

CREATE TABLE order_lines_2nf AS
SELECT order_id, line_no, product_id, product_sku, product_name, product_price,
       category_code, category_name, category_manager_id, category_manager_name,
       quantity, unit_price, line_total
FROM order_master;

-- ★ LOSSLESS CHECK
SELECT (SELECT count(*) FROM order_master) AS original,
       (SELECT count(*) FROM orders_2nf o JOIN order_lines_2nf l USING (order_id))
         AS rejoined;
```
```
 original | rejoined
----------+----------
        3 |        3      ✓ lossless
```

**Step 5 — 3NF: one table per determinant, then re-check.**

```sql
CREATE TABLE pincodes AS
  SELECT DISTINCT customer_pin AS pin, customer_city AS city,
                  customer_state AS state FROM orders_2nf;
CREATE TABLE customers AS
  SELECT DISTINCT customer_id AS id, customer_name AS name,
                  customer_email AS email, customer_pin AS pin FROM orders_2nf;
CREATE TABLE warehouses AS
  SELECT DISTINCT warehouse_code AS code, warehouse_city AS city,
                  warehouse_mgr AS manager FROM orders_2nf;
CREATE TABLE couriers AS
  SELECT DISTINCT courier_code AS code, courier_name AS name,
                  courier_rate_per_kg AS rate_per_kg FROM orders_2nf;
CREATE TABLE managers AS
  SELECT DISTINCT category_manager_id AS id, category_manager_name AS name
  FROM order_lines_2nf;
CREATE TABLE categories AS
  SELECT DISTINCT category_code AS code, category_name AS name,
                  category_manager_id AS manager_id FROM order_lines_2nf;
CREATE TABLE products AS
  SELECT DISTINCT product_id AS id, product_sku AS sku, product_name AS name,
                  category_code FROM order_lines_2nf;

CREATE TABLE orders AS
  SELECT order_id AS id, order_date, customer_id, warehouse_code, courier_code
  FROM orders_2nf;
CREATE TABLE order_lines AS
  SELECT order_id, line_no, product_id, quantity, unit_price, line_total
  FROM order_lines_2nf;
```

```sql
-- ★ RE-CHECK: does `customers` have an internal violation?
SELECT * FROM fd('customers','pin','id');       -- pin does NOT determine id ✓
-- but pin → city, state was already extracted into `pincodes` ✓
-- ★ THIS is the step people skip. f3 only became visible here.
```

**Step 6 — BCNF: the discovery.**

```sql
-- product_sku is a second candidate key
SELECT count(*) AS rows, count(DISTINCT sku) AS distinct_skus FROM products;
```
```
 rows | distinct_skus
------+---------------
    2 |             2      ★ sku is unique ⇒ a candidate key
```
```sql
ALTER TABLE products ADD CONSTRAINT uq_products_sku UNIQUE (sku);
-- ★ A CONSTRAINT THE ORIGINAL SCHEMA NEVER HAD, discovered by the analysis.
```

**Step 7 — the judgement calls.**

```sql
-- ① unit_price is a SNAPSHOT — rename it, and remove price from products
ALTER TABLE order_lines RENAME COLUMN unit_price TO unit_price_charged;
-- (products already has no price column — we never carried it forward,
--  because prices are temporal)
CREATE TABLE product_prices (
  product_id bigint NOT NULL,
  price_minor bigint NOT NULL CHECK (price_minor >= 0),
  valid tstzrange NOT NULL
);

-- ② line_total is DERIVED — make it GENERATED so it cannot disagree
ALTER TABLE order_lines DROP COLUMN line_total;
ALTER TABLE order_lines ADD COLUMN line_total_minor bigint
  GENERATED ALWAYS AS (quantity * unit_price_charged_minor) STORED;

-- ③ the delivery address must be a SNAPSHOT
CREATE TABLE order_addresses (
  order_id bigint PRIMARY KEY,
  line1 text NOT NULL, city text NOT NULL, state text NOT NULL,
  pin char(6) NOT NULL          -- ★ copied at order time, never updated
);
```

**Step 8 — the final schema, with everything the analysis earned.**

```sql
CREATE EXTENSION IF NOT EXISTS citext;

CREATE TABLE pincodes (
  pin   char(6) PRIMARY KEY CHECK (pin ~ '^[1-9][0-9]{5}$'),
  city  text NOT NULL,
  state text NOT NULL
);

CREATE TABLE customers (
  id    bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  name  text   NOT NULL,
  email citext NOT NULL,
  pin   char(6) NOT NULL REFERENCES pincodes(pin) ON DELETE RESTRICT,
  CONSTRAINT uq_customers_email UNIQUE (email)
);

CREATE TABLE customer_phones (
  customer_id bigint  NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
  phone       text    NOT NULL,
  kind        text    NOT NULL CHECK (kind IN ('mobile','home','work')),
  is_primary  boolean NOT NULL DEFAULT false,
  PRIMARY KEY (customer_id, phone)
);
CREATE UNIQUE INDEX uq_primary_phone ON customer_phones (customer_id) WHERE is_primary;

CREATE TABLE managers (
  id   bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  name text NOT NULL
);

CREATE TABLE categories (
  code       text   PRIMARY KEY,
  name       text   NOT NULL,
  manager_id bigint NOT NULL REFERENCES managers(id) ON DELETE RESTRICT
);

CREATE TABLE products (
  id       bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  sku      text NOT NULL,
  name     text NOT NULL,
  category_code text NOT NULL REFERENCES categories(code) ON DELETE RESTRICT,
  CONSTRAINT uq_products_sku UNIQUE (sku)          -- ★ discovered in BCNF
);

CREATE TABLE product_prices (                       -- ★ temporal (T26)
  product_id  bigint    NOT NULL REFERENCES products(id) ON DELETE CASCADE,
  price_minor bigint    NOT NULL CHECK (price_minor >= 0),
  valid       tstzrange NOT NULL,
  EXCLUDE USING gist (product_id WITH =, valid WITH &&)
);

CREATE TABLE warehouses (
  code text PRIMARY KEY, city text NOT NULL, manager text NOT NULL
);
CREATE TABLE couriers (
  code text PRIMARY KEY, name text NOT NULL,
  rate_per_kg_minor bigint NOT NULL CHECK (rate_per_kg_minor >= 0)
);

CREATE TABLE orders (
  id             bigint      GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  order_date     timestamptz NOT NULL DEFAULT now(),     -- ★ T26
  customer_id    bigint      NOT NULL REFERENCES customers(id) ON DELETE RESTRICT,
  warehouse_code text        NOT NULL REFERENCES warehouses(code) ON DELETE RESTRICT,
  courier_code   text        NOT NULL REFERENCES couriers(code) ON DELETE RESTRICT
);
CREATE INDEX ON orders (customer_id, order_date DESC);

CREATE TABLE order_addresses (                      -- ★ the snapshot
  order_id bigint PRIMARY KEY REFERENCES orders(id) ON DELETE CASCADE,
  line1 text NOT NULL, city text NOT NULL, state text NOT NULL,
  pin char(6) NOT NULL
);

CREATE TABLE order_lines (
  order_id  bigint   NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
  line_no   smallint NOT NULL,
  product_id bigint  NOT NULL REFERENCES products(id) ON DELETE RESTRICT,
  quantity  int      NOT NULL CHECK (quantity > 0),
  unit_price_charged_minor bigint NOT NULL CHECK (unit_price_charged_minor >= 0),
  line_total_minor bigint GENERATED ALWAYS AS      -- ★ cannot disagree
    (quantity * unit_price_charged_minor) STORED,
  PRIMARY KEY (order_id, line_no),
  CONSTRAINT uq_line_product UNIQUE (order_id, product_id)
);
CREATE INDEX ON order_lines (product_id);

CREATE TABLE order_tags (
  order_id bigint NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
  tag      text   NOT NULL,
  PRIMARY KEY (order_id, tag)
);
CREATE INDEX ON order_tags (tag);
```

---

## Example 2 — production scenario

**Applying the procedure to the real 620 GB table.**

**Step 1 — measure before you touch anything.**

```sql
SELECT count(*) AS rows,
       pg_size_pretty(pg_total_relation_size('order_master')) AS size,
       count(DISTINCT order_id) AS orders,
       count(DISTINCT customer_id) AS customers,
       count(DISTINCT product_id) AS products,
       count(DISTINCT category_code) AS categories,
       count(DISTINCT category_manager_id) AS managers
FROM order_master;
```
```
   rows   |  size  |  orders  | customers | products | categories | managers
----------+--------+----------+-----------+----------+------------+----------
 8012004  | 620 GB |  2004112 |    204882 |    41208 |         88 |       40
```
```
 REPETITION FACTORS:
   customer_name  8.0M / 204,882 = 39×
   product_name   8.0M /  41,208 = 194×
   category_name  8.0M /      88 = 91,045×
   manager_name   8.0M /      40 = 200,300×     ★
```

**Step 2 — find the corruption the redundancy permitted (Topic 30).**

```sql
SELECT * FROM fd('order_master','customer_id','customer_email')
UNION ALL SELECT * FROM fd('order_master','customer_id','customer_name')
UNION ALL SELECT * FROM fd('order_master','product_id','product_name')
UNION ALL SELECT * FROM fd('order_master','product_sku','product_id')
UNION ALL SELECT * FROM fd('order_master','category_code','category_manager_id')
UNION ALL SELECT * FROM fd('order_master','customer_pin','customer_city')
UNION ALL SELECT * FROM fd('order_master','warehouse_code','warehouse_mgr')
UNION ALL SELECT * FROM fd('order_master','courier_code','courier_rate_per_kg');
```
```
                dependency                 | groups | violations
-------------------------------------------+--------+------------
 customer_id -> customer_email             | 204882 |       8412
 customer_id -> customer_name              | 204882 |       2104
 product_id -> product_name                |  41208 |       8104
 product_sku -> product_id                 |  41194 |         14   ⚠⚠
 category_code -> category_manager_id      |     88 |         12
 customer_pin -> customer_city             |  18402 |        882
 warehouse_code -> warehouse_mgr           |     14 |         14   ⚠⚠⚠
 courier_code -> courier_rate_per_kg       |      8 |          8   ⚠⚠⚠
```

**Step 3 — interpret each, using the rate (Topic 30's rule).**

| Finding | Rate | Interpretation | Action |
|---|---|---|---|
| `customer_id → email` | 4% | corruption (email changes) | take the most recent |
| `customer_id → name` | 1% | corruption (typos) | most recent |
| `product_id → name` | 20% | corruption (renames) | most recent |
| **`product_sku → product_id`** | 0.03% | ⚠ **14 SKUs map to two product_ids** — a data-entry error creating duplicate products | **manual review** |
| `category_code → manager_id` | 14% | ⚠ ambiguous — managers change | check temporal pattern |
| `pin → city` | 5% | corruption (typos) | canonicalise against a postal reference |
| **`warehouse_code → manager`** | **100%** | ★ **the FD is FALSE** — managers change over time | **temporal, not a violation** |
| **`courier_code → rate`** | **100%** | ★ **the FD is FALSE** — rates are renegotiated | **temporal + a snapshot** |

```sql
-- confirm the two 100% cases are temporal
SELECT warehouse_code, warehouse_mgr, min(order_date), max(order_date), count(*)
FROM order_master GROUP BY 1,2 ORDER BY 1,3 LIMIT 6;
```
```
 warehouse_code | warehouse_mgr | min        | max        | count
----------------+---------------+------------+------------+--------
 WH-BLR         | Ravi K        | 2022-01-04 | 2024-06-30 | 412008
 WH-BLR         | Sunil P       | 2024-07-01 | 2026-08-11 | 388104
```
**Clean, non-overlapping date ranges ⇒ temporal.** The warehouse manager is not a violation to remove; it's a fact that needs a validity range.

```sql
-- and the courier rate is a SNAPSHOT — the rate CHARGED on that order
SELECT courier_code, courier_rate_per_kg, min(order_date), max(order_date)
FROM order_master GROUP BY 1,2 ORDER BY 1,3 LIMIT 4;
-- also clean ranges ⇒ the rate on the order is what was charged.
-- ⇒ ★ KEEP it on `orders`, renamed. And ALSO create courier_rates
--   with a validity range for the current rate card.
```

**Step 4 — the three judgement calls, decided.**

```
 ① unit_price → a SNAPSHOT. Keep, rename to unit_price_charged_minor.
    products gets NO price column; product_prices gets a tstzrange.
 ② courier_rate_per_kg → a SNAPSHOT on the order (what was charged),
    PLUS courier_rates(courier_code, rate, valid) for the current card.
 ③ warehouse_mgr → TEMPORAL, not a per-order snapshot (nobody needs
    "who managed the warehouse when this order shipped").
    ⇒ warehouse_managers(warehouse_code, manager, valid) with an
      EXCLUDE constraint. NOT copied onto orders.

 ★ NOTE THE DIFFERENCE BETWEEN ② AND ③:
   the courier RATE affects what the customer paid ⇒ snapshot it.
   the warehouse MANAGER does not ⇒ temporal reference only.
   ⇒ the test is "does this value affect a number on the invoice?"
```

**Step 5 — the migration order (Topic 28).**

```
 ① CREATE the new tables (empty). Instant.
 ② BACKFILL reference tables from DISTINCT — small, fast:
      managers, categories, couriers, warehouses, pincodes  (< 20k rows)
 ③ RESOLVE the conflicts. THIS IS THE SCHEDULE:
      • 8,412 email conflicts    → most recent. Mechanical.
      • 8,104 product renames    → most recent. Mechanical.
      • 882 pincode conflicts    → against India Post reference. Semi.
      • ★ 14 duplicate SKUs      → MANUAL. Two product_ids share a SKU;
        merging them changes order history. Needs merchandising.
      • ★ 12 category-manager    → check temporal; likely a handover.
 ④ BACKFILL customers, products (batched, 10k rows, throttled).
 ⑤ BACKFILL orders, order_addresses, order_lines, order_tags,
    customer_phones (batched — 8M and 2M rows).
 ⑥ VERIFY LOSSLESSNESS on a sample (Topic 36):
      rebuild the original view by joining, EXCEPT against order_master.
      ★ spurious and lost must both be 0.
 ⑦ DUAL-WRITE, cut reads over, stop writing the old table, drop it.
```

```sql
-- ⑥ the verification that gates the cutover
WITH rebuilt AS (
  SELECT o.id AS order_id, l.line_no, o.order_date, o.customer_id,
         c.name AS customer_name, c.email AS customer_email,
         p.id AS product_id, p.sku AS product_sku, p.name AS product_name,
         cat.code AS category_code, cat.name AS category_name,
         m.id AS category_manager_id, m.name AS category_manager_name,
         l.quantity, l.unit_price_charged_minor
  FROM orders o
  JOIN order_lines l ON l.order_id = o.id
  JOIN customers c ON c.id = o.customer_id
  JOIN products p ON p.id = l.product_id
  JOIN categories cat ON cat.code = p.category_code
  JOIN managers m ON m.id = cat.manager_id
)
SELECT (SELECT count(*) FROM order_master) AS original,
       (SELECT count(*) FROM rebuilt) AS rebuilt,
       (SELECT count(*) FROM (
          SELECT order_id,line_no,quantity FROM rebuilt
          EXCEPT SELECT order_id,line_no,quantity FROM order_master) x) AS spurious,
       (SELECT count(*) FROM (
          SELECT order_id,line_no,quantity FROM order_master
          EXCEPT SELECT order_id,line_no,quantity FROM rebuilt) x) AS lost;
```
```
 original | rebuilt | spurious | lost
----------+---------+----------+------
  8012004 | 8012004 |        0 |    0      ✓ CUTOVER APPROVED
```

**Step 6 — results.**

| | Before | After |
|---|---|---|
| Tables | 1 | 15 |
| Storage | 620 GB | **178 GB** |
| Enforceable constraints | 1 | **34** |
| `customer_id → email` conflicts | 8,412 | **0, impossible** |
| Duplicate SKUs | 14 | **0, `UNIQUE` enforced** |
| Adding a product | needs an order | **`INSERT INTO products`** |
| Reprinting a 2023 invoice | wrong prices | **exact — snapshots kept** |
| Manager rename | 200,300 rows | **1 row** |

---

## Common mistakes

**1. Doing the forms out of order.**
- *Symptom:* finding partial dependencies that vanish after you fix 1NF.
- *Fix:* 1NF first, always. It changes the attribute set and therefore the keys.

**2. Not re-checking after each decomposition.**
- *Symptom:* a violation hiding one level down (`pin → city` inside `customers`).
- *Fix:* after every split, run the check again on each new table.

**3. Normalising away a snapshot.**
- *Symptom:* historical invoices change when a price or rate changes.
- *Fix:* the test — "if the source changes, *should* this?" — plus the temporal check on the violation rate.

**4. Treating a 100% violation rate as corruption.**
- *Symptom:* deleting `courier_rate_per_kg` because "every courier has multiple rates."
- *Fix:* a 100% rate with clean date ranges means the FD is **false** and the value is temporal. Check the dates.

**5. Assuming the decomposition is lossless because it looks right.**
- *Symptom:* the rebuilt view has more rows than the original.
- *Fix:* run the `EXCEPT` test in both directions before cutover. Both must be 0.

**6. Underestimating the conflict resolution.**
- *Symptom:* a plan that says "backfill with `SELECT DISTINCT`" and slips by two months.
- *Fix:* every FD violation is a conflict someone must resolve. 8,412 emails are mechanical; **14 duplicate SKUs need merchandising**, and that's the critical path.

**7. Stopping at tables instead of constraints.**
- *Symptom:* 15 tidy tables and still no `UNIQUE` on `sku`.
- *Fix:* the analysis *produces* constraints. Write them all down as you go — the SKU one only appeared during the BCNF step.

---

## Hands-on proof

**PROVE IT #1 — the FD checker.** (Example 1, step 2.)
**PROVE IT #2 — 1NF explosion.** (Example 1, step 3.)
**PROVE IT #3 — the 2NF lossless check.** (Example 1, step 4.)
**PROVE IT #4 — the hidden violation found by re-checking.** (Example 1, step 5.)
**PROVE IT #5 — the BCNF discovery.** (Example 1, step 6.)

**PROVE IT #6 — the full rebuild verification.** (Example 2, step 5 ⑥.)

**PROVE IT #7 — the repetition audit, on any wide table.**
```sql
SELECT 'customer_name' AS col, count(*) AS rows,
       count(DISTINCT customer_id) AS entities,
       round(count(*)::numeric/count(DISTINCT customer_id)) AS repetition
FROM order_master
UNION ALL SELECT 'category_manager_name', count(*),
       count(DISTINCT category_manager_id),
       round(count(*)::numeric/count(DISTINCT category_manager_id))
FROM order_master
ORDER BY 4 DESC;
```
```
         col           |  rows   | entities | repetition
-----------------------+---------+----------+------------
 category_manager_name | 8012004 |       40 |     200300
 customer_name         | 8012004 |   204882 |         39
```
**Repetition above ~100 on a descriptive column is a missing table.**

---

## The design decision framework

```
THE ELEVEN-STEP PROCEDURE — run it in this order, every time:

 ① WRITE THE FDs AND MVDs, FROM THE DOMAIN. Get them confirmed.
 ② FIND THE CANDIDATE KEYS by closure. Declare any natural key as UNIQUE.
 ③ 1NF FIRST — atomicity. It changes the attribute set and the keys.
 ④ 2NF — only if a candidate key is composite.
 ⑤ 3NF — the one that finds the most. ★ RE-CHECK EACH NEW TABLE.
 ⑥ BCNF — only possible with overlapping candidate keys.
 ⑦ 4NF — two independent lists in one table.
 ⑧ 5NF — check the anti-tell, then stop.
 ⑨ VERIFY LOSSLESSNESS after every step: EXCEPT both ways, 0 and 0.
 ⑩ ★ THE JUDGEMENT CALLS — for every column you're about to remove:
      "if the source changes, SHOULD this change?"
        yes → a copy. Remove it.
        no  → a SNAPSHOT. Keep it, and RENAME it to say so.
      And check the violation rate:
        low + no pattern     → corruption. Fix data, then decompose.
        100% + clean ranges  → the FD is FALSE. It's temporal.
 ⑪ WRITE THE DDL WITH EVERY CONSTRAINT THE ANALYSIS EARNED.

★ THE THREE OUTPUTS, in order of value:
  ③ CONSTRAINTS — 34 rules the original could not express
  ② DISCOVERIES — a missing UNIQUE, a needed snapshot table
  ① TABLES — the visible but least interesting result

THE SIGNAL TO LOOK FOR — before you start:
      SELECT count(*) AS rows, count(DISTINCT <id_col>) AS entities,
             round(count(*)::numeric/count(DISTINCT <id_col>)) AS repetition
      FROM <table>;
  Run it for every `*_id` column. Repetition > 100 on a descriptive
  column means a missing table. Then run the FD checker to see whether
  the redundancy has already corrupted itself — it usually has.

⚠ AND BUDGET THE CONFLICT RESOLUTION.
  The decomposition is an afternoon. Resolving 14 duplicate SKUs with
  merchandising is the schedule.
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
Build `order_master` with the sample data. Run the FD checker for all twelve FDs. Then take the table through 1NF and 2NF only, and verify losslessness after each step with the `EXCEPT` test in both directions.

### Exercise 2 — medium (apply it)
Continue from Exercise 1 through 3NF, BCNF, 4NF and 5NF.
(a) Show the four-link chain and where each link becomes visible.
(b) Identify the constraint discovered during the BCNF step and explain how the analysis revealed it.
(c) For each of the three judgement calls, argue both sides before deciding.
(d) Produce the final DDL and count the enforceable constraints.
(e) Verify the complete rebuild is lossless.

### Exercise 3 — hard (production simulation)
You inherit the 620 GB, 8M-row `order_master` table. The audit finds the eight FD violations from Example 2 step 2.

(a) Classify each of the eight by rate and temporal pattern, and state the action for each.
(b) Two have a 100% violation rate. Explain why that means the FD is *false* rather than the data corrupt, and give the query that confirms it.
(c) The two 100% cases need *different* treatments — one a snapshot, one a temporal reference. Explain the test that distinguishes them.
(d) 14 SKUs map to two `product_id`s each. Explain why this cannot be resolved mechanically and what merging them would do to order history.
(e) Produce the full 15-table schema with every constraint.
(f) Write the losslessness verification that gates the cutover.
(g) Give the migration plan with Topic 28's techniques, identifying the critical path.
(h) List every business rule that becomes enforceable, mapping each to the normalisation step that revealed it.

---

## Mental model checkpoint

1. Why must 1NF be done first? What breaks if you do 2NF first?
2. Why must you re-check each table after a decomposition? Give the example from this topic.
3. Give the test that distinguishes a copy from a snapshot.
4. An FD shows a 100% violation rate. What does that mean, and what query confirms it?
5. Two columns both have a 100% violation rate — one becomes a snapshot, one a temporal reference. What distinguishes them?
6. What are the three outputs of a normalisation exercise, in order of value?
7. Why is the decomposition the easy part of the migration?

---

## Quick reference card

**The eleven steps**

| # | Step | Topic |
|---|---|---|
| ① | FDs and MVDs from the domain | 30 |
| ② | candidate keys via closure | 30 |
| ③ | **1NF first** — atomicity | 31 |
| ④ | 2NF — composite keys only | 32 |
| ⑤ | 3NF — **re-check each new table** | 33 |
| ⑥ | BCNF — overlapping keys only | 34 |
| ⑦ | 4NF — two independent lists | 35 |
| ⑧ | 5NF — check the anti-tell, stop | 36 |
| ⑨ | verify losslessness (both ways, 0 and 0) | 36 |
| ⑩ | **the judgement calls** | 26, 29 |
| ⑪ | DDL with every constraint earned | 24 |

**The snapshot test:** *if the source changes, should this change?* No → keep it, rename it.

**The violation-rate rule**

| Rate | Meaning |
|---|---|
| low, no pattern | corruption — fix, then decompose |
| 100%, clean date ranges | the FD is **false** — temporal |

**The repetition detector**
```sql
SELECT count(*) AS rows, count(DISTINCT <id>) AS entities,
       round(count(*)::numeric/count(DISTINCT <id>)) AS repetition FROM t;
-- > 100 on a descriptive column → a missing table
```

**The losslessness gate** — both must be 0:
```sql
SELECT count(*) FROM (SELECT … FROM rebuilt EXCEPT SELECT … FROM original) x;
SELECT count(*) FROM (SELECT … FROM original EXCEPT SELECT … FROM rebuilt) x;
```

---

## When would I use this at work?

1. **Inheriting a wide legacy table.** The repetition audit plus the FD checker gives you a complete, defensible list of defects in about an hour — and tells you which are corruption and which are temporal facts.

2. **Planning a decomposition migration.** The eleven steps produce the table list, the constraint list, *and* the conflict list — and the conflict list is what determines the schedule, not the DDL.

3. **Reviewing someone else's normalisation.** Two questions catch most errors: "did you re-check each table after splitting?" and "which of the removed columns were snapshots?"

---

## Connected topics

**Understand before this:** 29–36 (every step here applies one of them), 26 (time, money, snapshots — three of the judgement calls), 28 (the migration techniques).

**This unlocks:**
- **38** — when to stop, and how far down this procedure to go in practice
- **53–55** — deliberate denormalisation, which starts from the output of this exercise
- **Case studies 03, 04** — the snapshot-vs-reference distinction in full production designs
