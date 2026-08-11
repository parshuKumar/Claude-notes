# 29 — What Normalisation Is
## Phase: Normalisation

---

## ELI5 — The Simple Analogy

A shop keeps one big notebook. Every line is a sale:

```
 Arjun | 98765-43210 | Bengaluru | Basmati Rice | ₹549 | 2 | 2026-03-01
 Arjun | 98765-43210 | Bengaluru | Ghee         | ₹720 | 1 | 2026-03-01
 Meera | 91234-56789 | Mumbai    | Basmati Rice | ₹549 | 3 | 2026-03-02
```

Three problems arrive, and they arrive in every business that keeps a notebook like this.

**Arjun changes his phone number.** It appears on 400 lines. You change 398 of them and miss two. Now the notebook says Arjun has two phone numbers, and *the notebook cannot tell you which is right.*

**A new product arrives that nobody has bought yet.** There is nowhere to write it. A line requires a customer and a sale. The product does not exist until someone buys it.

**Meera's only order is cancelled and you rub out the line.** Meera's phone number is gone too — it only ever existed on that line.

These are the **update, insert and delete anomalies**, and they are not carelessness. They are a *structural* property of putting several different kinds of fact on one line. Normalisation is the procedure for taking them out.

---

## Where this fits in the big picture

```
   PHASE 3 (20–28) — you can now design and change a schema
                          │
                          ▼
        ┌──────────────────────────────────────────┐
        │ PHASE 4 — NORMALISATION                  │
        │ 29 WHAT IT IS            ← YOU ARE HERE  │
        │    the anomalies, and why they happen    │
        └────────────────────┬─────────────────────┘
                             ▼
                    30 functional dependencies (the maths)
                    31–36 1NF → 5NF (the procedure)
                    37 a full worked example
                    38 when to stop
                             ▼
                    PHASE 6 — denormalisation
                    (breaking these rules, deliberately,
                     once you can price the risk)
```

Phase 3 taught you to *build* a schema. **Phase 4 is the theory that tells you whether the one you built is correct** — and it is the only part of this curriculum that is genuinely mathematics.

---

## What is this?

**Normalisation** is a procedure for restructuring tables so that **every fact is stored in exactly one place**.

It works by identifying *dependencies* — "given this, that is determined" — and splitting tables until no table contains a dependency that doesn't belong to it. The result is a series of **normal forms** (1NF, 2NF, 3NF, BCNF, 4NF, 5NF), each stricter than the last, each removing a specific class of anomaly.

It is not about saving space, though it usually does. It is about **making a certain category of bug impossible.**

---

## Why does it matter for a backend developer?

Because the anomalies are not theoretical — they are the most common data-corruption bugs in production, and they are *silent*:

```
 ① UPDATE ANOMALY — the same fact in many rows
    "Arjun's phone number" appears on 400 order rows. An UPDATE that
    misses some rows leaves the database holding two contradictory
    answers to one question. Nothing errors. Nobody notices.
    ⇒ THIS IS THE ONE THAT ACTUALLY HAPPENS. Constantly.

 ② INSERT ANOMALY — a fact that has nowhere to live
    You cannot record a product until someone buys it, or a customer
    until they order. So people invent placeholder rows with NULL
    everywhere, and every query grows an `AND product_id IS NOT NULL`.

 ③ DELETE ANOMALY — losing a fact you didn't mean to lose
    Deleting the last order for a customer deletes the customer.
    Deleting a cancelled booking deletes the room.
```

And a fourth consequence that people underestimate: **you cannot put a constraint on a fact that is stored in many places.** You cannot declare "a customer has one phone number" when the phone number lives on the order line. All of Topic 24's machinery — `UNIQUE`, `CHECK`, `FOREIGN KEY` — depends on each fact having exactly one home.

**Normalisation is what makes constraints possible.**

---

## The physical reality

### What redundancy costs, in bytes and in writes

```
 THE FLAT TABLE — one row per order line
 orders_flat(
   order_id, order_date,
   customer_name, customer_phone, customer_city,   ← repeated per LINE
   product_name, product_price,                    ← repeated per SALE
   quantity, line_total
 )

 8,000,000 order lines · 2,000,000 orders · 200,000 customers · 40,000 products

 STORAGE:
   customer_name  (~18 B) × 8,000,000 = 144 MB   ← but only 200,000 distinct
   customer_phone (~14 B) × 8,000,000 = 112 MB
   customer_city  (~12 B) × 8,000,000 =  96 MB
   product_name   (~24 B) × 8,000,000 = 192 MB   ← but only 40,000 distinct
   product_price  (   8 B) × 8,000,000 =  64 MB
                                        ───────
                                        608 MB of REPEATED facts

 NORMALISED:
   customers(200,000 × 44 B)  =  8.8 MB
   products (40,000 × 32 B)   =  1.3 MB
   order_lines FK columns     = 8,000,000 × 16 B = 128 MB
                                ───────
                                138 MB     ⇒ 4.4× less

 THE WRITE COST — the part that matters more:
   "Arjun changes his phone number"
     flat:       UPDATE 400 rows. 400 tuple rewrites (Topic 46),
                 400 × N index entries, 400 rows of WAL.
                 ★ AND if the UPDATE's WHERE clause is wrong, the
                   database now holds two different phone numbers for
                   one person, with no way to tell which is current.
     normalised: UPDATE 1 row. ★ It is not possible for the answer to
                 be inconsistent, because there is only one answer.
```

### The deeper point: redundancy defeats constraints

```
 IN THE FLAT TABLE, YOU CANNOT DECLARE:
   ✗ UNIQUE (customer_phone)        — it repeats legitimately
   ✗ NOT NULL on customer_city      — for a product with no orders yet
   ✗ FOREIGN KEY to a product list  — there is no product list
   ✗ CHECK (one phone per customer) — a CHECK cannot see other rows (T24)

 IN THE NORMALISED SCHEMA, ALL FOUR ARE ONE LINE OF DDL.

 ★ THIS IS THE REAL ARGUMENT FOR NORMALISATION.
   Not disk. Not elegance. It is that a fact stored once can be
   CONSTRAINED, and a fact stored many times cannot.
```

---

## How it works — step by step

### The three anomalies, precisely

```sql
CREATE TABLE orders_flat (
  order_id       bigint,
  order_date     date,
  customer_id    bigint,
  customer_name  text,
  customer_phone text,
  customer_city  text,
  product_id     bigint,
  product_name   text,
  product_price  bigint,
  quantity       int,
  PRIMARY KEY (order_id, product_id)
);
```

```
 ① UPDATE ANOMALY
    Arjun's phone changes. His phone number is on every line of every
    order he ever placed.

    UPDATE orders_flat SET customer_phone='99999-11111'
     WHERE customer_id = 7;

    ⇒ works — IF you remember the WHERE clause is on customer_id and
      not on order_id, and IF no concurrent insert adds a row with the
      old number between your SELECT and your UPDATE.
    ⇒ THE DANGER IS NOT THE STATEMENT. It is that the schema PERMITS
      an inconsistent state at all:
         SELECT DISTINCT customer_phone FROM orders_flat WHERE customer_id=7;
      returning two rows is a perfectly legal state of this table.

 ② INSERT ANOMALY
    A new product arrives; nobody has bought it.
    INSERT INTO orders_flat (product_id, product_name, product_price)
      VALUES (9001, 'Jaggery', 18000);
    ⇒ order_id is part of the PRIMARY KEY and cannot be NULL.
    ⇒ You CANNOT record the product. The fact has nowhere to live.
    ⇒ Workaround people actually use: a fake order row with
      order_id = -1 and NULLs everywhere. Then every query needs
      `AND order_id > 0`, forever.

 ③ DELETE ANOMALY
    Meera's only order is cancelled.
    DELETE FROM orders_flat WHERE order_id = 91;
    ⇒ Meera's name, phone and city are gone. They only existed on that
      line. You have deleted a customer by deleting an order.
```

### Why the anomalies happen — the one-sentence cause

```
 ★ AN ANOMALY OCCURS WHEN ONE TABLE STORES FACTS ABOUT MORE THAN ONE
   KIND OF THING.

 orders_flat stores facts about:
   • an ORDER          (order_date)          — keyed by order_id
   • a CUSTOMER        (name, phone, city)   — keyed by customer_id
   • a PRODUCT         (name, price)         — keyed by product_id
   • an ORDER LINE     (quantity)            — keyed by (order_id, product_id)

 FOUR kinds of thing. FOUR different keys. ONE table.

 ⇒ The table's primary key is (order_id, product_id). Any fact whose
   real key is NOT that pair is in the wrong place — and every such fact
   will be repeated, or absent, or lost.

 ★ THAT IS THE WHOLE OF NORMALISATION IN ONE SENTENCE:
   "every non-key fact must depend on the key, the whole key, and
    nothing but the key."
   Topics 31–34 are just the four precise ways that can fail.
```

### The normalised result

```sql
CREATE TABLE customers (
  id    bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  name  text   NOT NULL,
  phone text   NOT NULL,           -- ★ now constrainable
  city  text   NOT NULL,
  CONSTRAINT uq_customers_phone UNIQUE (phone)   -- ★ now possible
);

CREATE TABLE products (
  id          bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  name        text   NOT NULL,
  price_minor bigint NOT NULL CHECK (price_minor >= 0)
);
-- ★ a product can now exist before anyone buys it

CREATE TABLE orders (
  id          bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  customer_id bigint NOT NULL REFERENCES customers(id) ON DELETE RESTRICT,
  order_date  date   NOT NULL
);
-- ★ deleting an order cannot delete a customer

CREATE TABLE order_lines (
  order_id   bigint NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
  product_id bigint NOT NULL REFERENCES products(id) ON DELETE RESTRICT,
  quantity   int    NOT NULL CHECK (quantity > 0),
  PRIMARY KEY (order_id, product_id)
);
```

**Each table now has exactly one kind of thing, and every column depends on that table's key.** The three anomalies are structurally impossible.

⚠ **One important nuance, foreshadowing Topic 38:** `order_lines` has no `unit_price`. That is *correct normalisation* and *wrong business modelling* — the price paid is a fact about the order line, not about the product today (Topics 21, 26, case study 04). Normalisation tells you where a fact belongs *given its dependencies*; it cannot tell you which facts you need. Deciding that is design.

### The cost: joins

```sql
-- FLAT: one table
SELECT customer_name, product_name, quantity FROM orders_flat WHERE order_id=91;

-- NORMALISED: three joins
SELECT c.name, p.name, ol.quantity
FROM order_lines ol
JOIN orders o    ON o.id = ol.order_id
JOIN customers c ON c.id = o.customer_id
JOIN products p  ON p.id = ol.product_id
WHERE ol.order_id = 91;
```
```
 ★ AND THIS IS THE TRADE, STATED HONESTLY:
   normalisation moves work from WRITE time to READ time.
     flat:       1 read, N writes to change one fact, corruption possible
     normalised: N reads, 1 write to change one fact, corruption impossible

   From Topic 19 you know the read cost precisely: a hash join over 8M
   rows is ~400 ms; three indexed nested-loop joins for one order is
   ~0.3 ms. ⇒ The read cost is usually negligible and always bounded.
   The correctness benefit is unbounded.
   ⇒ NORMALISE FIRST. Denormalise later, with evidence (Phase 6).
```

---

## Concept breakdown

```
NORMALISATION
│  └── restructuring so that every fact is stored EXACTLY ONCE.
│      Achieved by splitting tables until no table holds a fact
│      whose key is not that table's key.
│
├── THE GOAL       one fact, one place
├── THE MECHANISM  identify dependencies (Topic 30), decompose
├── THE COST       more joins at read time
└── THE PAYOFF     ① no anomalies  ② ★ constraints become possible

THE THREE ANOMALIES
│
├── UPDATE  the same fact in N rows → an inconsistent state is LEGAL
│           ★ the one that actually happens in production
├── INSERT  a fact with no place to live → placeholder rows, NULL sprawl
└── DELETE  removing one fact removes another → silent data loss

THE CAUSE, IN ONE SENTENCE
└── one table storing facts about more than one kind of thing.

THE NORMAL FORMS — each removes one specific failure
├── 1NF   atomic values; no repeating groups            (Topic 31)
├── 2NF   no partial dependency on a composite key      (Topic 32)
├── 3NF   no transitive dependency (non-key → non-key)  (Topic 33)
├── BCNF  every determinant is a candidate key          (Topic 34)
├── 4NF   no multi-valued dependency                    (Topic 35)
└── 5NF   no join dependency                            (Topic 36)
   ⇒ they are CUMULATIVE: 3NF implies 2NF implies 1NF.
   ⇒ ★ 3NF/BCNF is where almost every production schema stops (Topic 38).

DENORMALISATION ≠ NOT NORMALISING
├── DENORMALISED   you normalised, then deliberately reintroduced
│                  redundancy, with a measured reason and a plan for
│                  keeping the copies consistent      (Phase 6)
└── UNNORMALISED   you never did the analysis
   ★ These look identical in the DDL and are completely different in
     engineering terms. The difference is whether you can name the
     redundancy, the reason, and the reconciliation.

LOSSLESS DECOMPOSITION — the correctness condition
└── splitting R into R1 and R2 is LOSSLESS if joining them back gives
    exactly R — no rows lost, no spurious rows invented.
    Guaranteed when the shared attributes form a key of at least one
    of the two pieces. (Topic 36 shows what happens when they don't.)
```

---

## Diagrams

**Diagram 1 — big picture: one table, four kinds of thing**

```
 orders_flat
 ┌──────────┬────────────┬──────────────────────────┬──────────────────┬──────────┐
 │ order_id │ order_date │ customer_name/phone/city │ product_name/price│ quantity │
 └────┬─────┴─────┬──────┴────────────┬─────────────┴─────────┬────────┴────┬─────┘
      │           │                   │                       │             │
      │      depends on          depends on              depends on    depends on
      │      order_id            customer_id             product_id    (order_id,
      │                                                                 product_id)
      └───────────┴───────────────────┴───────────────────────┴─────────────┘
                                      │
                        ★ FOUR DIFFERENT KEYS. ONE TABLE.
                          The PK is (order_id, product_id).
                          Everything keyed differently is misplaced —
                          and will be repeated, absent, or lost.

 ⇒ DECOMPOSE ALONG THE KEYS:
   ┌───────────┐   ┌────────┐   ┌──────────┐   ┌─────────────┐
   │ customers │   │ orders │   │ products │   │ order_lines │
   │ (cust_id) │◀──│(ord_id)│   │(prod_id) │◀──│(ord,prod)   │
   └───────────┘   └────┬───┘   └──────────┘   └──────┬──────┘
                        └──────────────────────────────┘
```

**Diagram 2 — data flow: changing one fact**

```
  FLAT                                 NORMALISED
  ──────────────────────────           ──────────────────────────
  "Arjun's phone changes"              "Arjun's phone changes"
          │                                     │
  UPDATE orders_flat                    UPDATE customers
   SET customer_phone=…                  SET phone=…
   WHERE customer_id=7                   WHERE id=7
          │                                     │
  ┌───────┴───────┐                             ▼
  │ 400 rows      │                      ┌────────────┐
  │ 400 tuple     │                      │ 1 row      │
  │  rewrites     │                      │ 1 rewrite  │
  │ 400×N index   │                      │ 0 index    │
  │  entries      │                      │  changes   │
  │ ★ AND an      │                      │            │
  │  inconsistent │                      │ ★ AN       │
  │  state is a   │                      │  INCONSISTENT
  │  LEGAL state  │                      │  STATE IS  │
  │  of the table │                      │  IMPOSSIBLE│
  └───────────────┘                      └────────────┘
```

**Diagram 3 — before/after: what becomes possible**

```
 BEFORE — flat
 ┌──────────────────────────────────────────────────────────────┐
 │ CAN YOU DECLARE…                                             │
 │   UNIQUE (customer_phone)?          ✗ it repeats legitimately│
 │   NOT NULL customer_city?           ✗ not for an unsold product│
 │   FK to a product list?             ✗ there is no product list│
 │   "one phone per customer"?         ✗ CHECK can't see other rows│
 │ ⇒ ZERO of the four business rules can be enforced.           │
 └──────────────────────────────────────────────────────────────┘

 AFTER — normalised
 ┌──────────────────────────────────────────────────────────────┐
 │   UNIQUE (phone) ON customers               ✓ one line       │
 │   NOT NULL city ON customers                ✓ one line       │
 │   FK order_lines → products                 ✓ one line       │
 │   one phone per customer                    ✓ it's a COLUMN  │
 │ ⇒ ALL FOUR, enforced by the engine, unbypassable. (Topic 24) │
 └──────────────────────────────────────────────────────────────┘
        ↑ ★ THIS is why we normalise. Not disk. Constraints.
```

---

## Example 1 — basic

**Step 1 — build the flat table and reproduce all three anomalies.**

```sql
CREATE TABLE orders_flat (
  order_id       bigint NOT NULL,
  order_date     date   NOT NULL,
  customer_id    bigint NOT NULL,
  customer_name  text   NOT NULL,
  customer_phone text   NOT NULL,
  customer_city  text   NOT NULL,
  product_id     bigint NOT NULL,
  product_name   text   NOT NULL,
  product_price  bigint NOT NULL,
  quantity       int    NOT NULL,
  PRIMARY KEY (order_id, product_id)
);

INSERT INTO orders_flat VALUES
 (1,'2026-03-01',7,'Arjun','98765-43210','Bengaluru',101,'Basmati Rice',54900,2),
 (1,'2026-03-01',7,'Arjun','98765-43210','Bengaluru',102,'Ghee',        72000,1),
 (2,'2026-03-02',7,'Arjun','98765-43210','Bengaluru',101,'Basmati Rice',54900,1),
 (3,'2026-03-02',8,'Meera','91234-56789','Mumbai',   101,'Basmati Rice',54900,3);
```

**The update anomaly — and the crucial demonstration:**

```sql
-- a partial update, exactly as a buggy WHERE clause would produce
UPDATE orders_flat SET customer_phone='99999-11111'
WHERE customer_id=7 AND order_id=1;

SELECT DISTINCT customer_id, customer_phone FROM orders_flat WHERE customer_id=7;
```
```
 customer_id | customer_phone
-------------+----------------
           7 | 99999-11111
           7 | 98765-43210        ★ TWO phone numbers for one person
```
```sql
-- can we prevent this?
ALTER TABLE orders_flat ADD CONSTRAINT uq_phone UNIQUE (customer_id, customer_phone);
-- ✓ accepted — and USELESS: it permits exactly the state above.
-- There is NO constraint expressible on this table that prevents it,
-- because a CHECK cannot see other rows (Topic 24).
```

**The insert anomaly:**
```sql
INSERT INTO orders_flat (product_id, product_name, product_price)
VALUES (103,'Jaggery',18000);
-- ERROR: null value in column "order_id" violates not-null constraint
--   ★ a new product cannot exist until someone buys it
```

**The delete anomaly:**
```sql
DELETE FROM orders_flat WHERE order_id=3;
SELECT count(*) FROM orders_flat WHERE customer_id=8;
```
```
 count
-------
     0        ★ Meera is gone. Her name, phone and city no longer exist.
```

**Step 2 — normalise, and watch all three become impossible.**

```sql
CREATE TABLE customers (
  id bigint PRIMARY KEY, name text NOT NULL,
  phone text NOT NULL UNIQUE, city text NOT NULL      -- ★ UNIQUE now possible
);
CREATE TABLE products (
  id bigint PRIMARY KEY, name text NOT NULL,
  price_minor bigint NOT NULL CHECK (price_minor >= 0)
);
CREATE TABLE orders (
  id bigint PRIMARY KEY,
  customer_id bigint NOT NULL REFERENCES customers(id) ON DELETE RESTRICT,
  order_date date NOT NULL
);
CREATE TABLE order_lines (
  order_id bigint NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
  product_id bigint NOT NULL REFERENCES products(id) ON DELETE RESTRICT,
  quantity int NOT NULL CHECK (quantity > 0),
  PRIMARY KEY (order_id, product_id)
);

INSERT INTO customers VALUES (7,'Arjun','98765-43210','Bengaluru'),
                             (8,'Meera','91234-56789','Mumbai');
INSERT INTO products VALUES (101,'Basmati Rice',54900),(102,'Ghee',72000);
INSERT INTO orders VALUES (1,7,'2026-03-01'),(2,7,'2026-03-02'),(3,8,'2026-03-02');
INSERT INTO order_lines VALUES (1,101,2),(1,102,1),(2,101,1),(3,101,3);
```

```sql
-- ① UPDATE: one row, and inconsistency is now IMPOSSIBLE
UPDATE customers SET phone='99999-11111' WHERE id=7;
SELECT count(DISTINCT phone) FROM customers WHERE id=7;   -- always 1, by construction

-- ② INSERT: a product with no orders — fine
INSERT INTO products VALUES (103,'Jaggery',18000);        -- ✓

-- ③ DELETE: an order, without losing the customer
DELETE FROM orders WHERE id=3;
SELECT name, phone FROM customers WHERE id=8;
```
```
 name  |    phone
-------+-------------
 Meera | 91234-56789        ★ still there
```

**Step 3 — measure the storage and the write cost at scale.**

```sql
-- flat, 8M lines
CREATE TABLE flat_big AS
SELECT (i/4)::bigint AS order_id, current_date AS order_date,
       (i%200000)::bigint AS customer_id,
       'Customer '||(i%200000) AS customer_name,
       '9'||lpad((i%200000)::text,9,'0') AS customer_phone,
       (ARRAY['Bengaluru','Mumbai','Delhi','Chennai'])[1+(i%4)] AS customer_city,
       (i%40000)::bigint AS product_id,
       'Product '||(i%40000) AS product_name,
       ((i%40000)*13)::bigint AS product_price,
       1+(i%5) AS quantity
FROM generate_series(1,8000000) i;

-- normalised
CREATE TABLE n_customers AS SELECT DISTINCT customer_id AS id, customer_name AS name,
  customer_phone AS phone, customer_city AS city FROM flat_big;
CREATE TABLE n_products AS SELECT DISTINCT product_id AS id, product_name AS name,
  product_price AS price FROM flat_big;
CREATE TABLE n_orders AS SELECT DISTINCT order_id AS id, customer_id, order_date FROM flat_big;
CREATE TABLE n_lines AS SELECT order_id, product_id, quantity FROM flat_big;

SELECT 'flat' AS design, pg_size_pretty(pg_relation_size('flat_big')) AS size
UNION ALL SELECT 'normalised',
  pg_size_pretty(pg_relation_size('n_customers')+pg_relation_size('n_products')
                +pg_relation_size('n_orders')+pg_relation_size('n_lines'));
```
```
   design   |  size
------------+---------
 flat       | 1104 MB
 normalised |  412 MB      ← 2.7× smaller
```

```sql
-- the write cost of changing ONE fact
\timing on
UPDATE flat_big SET customer_phone='NEW' WHERE customer_id=42;    -- 41 rows... at scale, 40
UPDATE n_customers SET phone='NEW' WHERE id=42;                   -- 1 row
```
On the real distribution (one customer with 400 lines) the flat update touches 400 tuples and every index entry on each; the normalised one touches one.

**Step 4 — the read cost, honestly.**

```sql
CREATE INDEX ON n_orders (id); CREATE INDEX ON n_lines (order_id);
ALTER TABLE n_customers ADD PRIMARY KEY (id);
ALTER TABLE n_products ADD PRIMARY KEY (id);
VACUUM ANALYZE n_customers; VACUUM ANALYZE n_products;
VACUUM ANALYZE n_orders; VACUUM ANALYZE n_lines;
CREATE INDEX ON flat_big (order_id);
VACUUM ANALYZE flat_big;

EXPLAIN (ANALYZE,BUFFERS) SELECT customer_name, product_name, quantity
FROM flat_big WHERE order_id = 4471;
-- Execution Time: 0.09 ms   Buffers: shared hit=6

EXPLAIN (ANALYZE,BUFFERS)
SELECT c.name, p.name, l.quantity FROM n_lines l
JOIN n_orders o ON o.id=l.order_id
JOIN n_customers c ON c.id=o.customer_id
JOIN n_products p ON p.id=l.product_id
WHERE l.order_id = 4471;
-- Execution Time: 0.21 ms   Buffers: shared hit=22
```
**0.09 ms vs 0.21 ms.** The joins cost 0.12 milliseconds — and bought four enforceable constraints and the impossibility of three classes of corruption.

---

## Example 2 — production scenario

**The situation.** A logistics company's `shipments` table, built by a contractor five years ago. 340 million rows, 890 GB.

```sql
CREATE TABLE shipments (
  id bigserial PRIMARY KEY,
  tracking_no text NOT NULL,
  -- sender
  sender_name text, sender_phone text, sender_email text,
  sender_address text, sender_city text, sender_pin text,
  -- recipient
  recipient_name text, recipient_phone text, recipient_email text,
  recipient_address text, recipient_city text, recipient_pin text,
  -- carrier
  carrier_code text, carrier_name text, carrier_contact text,
  carrier_sla_hours int, carrier_rate_per_kg bigint,
  -- service level
  service_code text, service_name text, service_max_weight_kg int,
  -- the shipment itself
  weight_kg numeric, declared_value bigint, created_at timestamptz,
  status text, delivered_at timestamptz
);
```

**Step 1 — quantify the redundancy.**

```sql
SELECT 'carriers' AS entity, count(*) AS rows,
       count(DISTINCT carrier_code) AS distinct_entities,
       round(count(*)::numeric/count(DISTINCT carrier_code)) AS repetition
FROM shipments
UNION ALL SELECT 'senders', count(*), count(DISTINCT sender_phone),
       round(count(*)::numeric/count(DISTINCT sender_phone)) FROM shipments
UNION ALL SELECT 'services', count(*), count(DISTINCT service_code),
       round(count(*)::numeric/count(DISTINCT service_code)) FROM shipments;
```
```
  entity  |   rows    | distinct_entities | repetition
----------+-----------+-------------------+------------
 carriers | 340102884 |                14 |   24292オ   ← ★ 14 carriers,
                                                          repeated 24M times each
 senders  | 340102884 |            412008 |        826
 services | 340102884 |                 6 |   56683814
```

**Fourteen carriers.** Their name, contact, SLA and rate are stored 340 million times.

**Step 2 — find the inconsistencies the redundancy permits.**

```sql
-- one carrier code with more than one name/SLA/rate?
SELECT carrier_code, count(DISTINCT carrier_name) AS names,
       count(DISTINCT carrier_sla_hours) AS slas,
       count(DISTINCT carrier_rate_per_kg) AS rates
FROM shipments GROUP BY 1 HAVING count(DISTINCT carrier_name) > 1
                              OR count(DISTINCT carrier_sla_hours) > 1;
```
```
 carrier_code |  names  | slas | rates
--------------+---------+------+-------
 BLUEDART     |       3 |    2 |     8
 DELHIVERY    |       2 |    3 |    11
 DTDC         |       4 |    1 |     6
```
```sql
SELECT DISTINCT carrier_name FROM shipments WHERE carrier_code='BLUEDART';
```
```
 Blue Dart
 Blue Dart Express
 BlueDart Express Ltd
```

**Three names, two SLAs and eight rates for one carrier.** Every reporting query that groups by `carrier_name` produces three rows for BlueDart. Every SLA calculation is a coin flip. **And the database has no way to say which is right, because there is no single row that is the carrier.**

```sql
-- and the senders
SELECT sender_phone, count(DISTINCT sender_name) AS names,
       count(DISTINCT sender_address) AS addresses
FROM shipments GROUP BY 1 HAVING count(DISTINCT sender_name) > 1
ORDER BY 2 DESC LIMIT 3;
```
```
  sender_phone  | names | addresses
----------------+-------+-----------
 98765-43210    |     4 |        11     ← typos and address changes,
 91234-56789    |     3 |         7        each frozen into history
```

**Step 3 — the insert and delete anomalies, in the business.**

```
 INSERT: onboarding a new carrier requires creating a fake shipment.
   ⇒ the ops team's actual workaround: a "test" shipment per carrier
     with tracking_no starting 'ZZTEST'. There are 41 of them, and
     every revenue report has `AND tracking_no NOT LIKE 'ZZTEST%'`.
     ★ 41 rows have distorted every report for five years.

 DELETE: purging shipments older than 7 years (a legal requirement)
   removes the only record of some senders — including senders with
   an open dispute. Legal have blocked the purge for two years, so
   the table is 890 GB when it should be ~380 GB.
   ★ A NORMALISATION DEFECT IS BLOCKING A COMPLIANCE REQUIREMENT.
```

**Step 4 — the normalised design.**

```sql
CREATE TABLE carriers (
  id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  code text NOT NULL UNIQUE,
  name text NOT NULL,
  contact text NOT NULL,
  sla_hours int NOT NULL CHECK (sla_hours > 0),
  rate_per_kg_minor bigint NOT NULL CHECK (rate_per_kg_minor >= 0)
);   -- ★ 14 rows, and ONE name per carrier is now a fact of the schema

CREATE TABLE service_levels (
  id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  code text NOT NULL UNIQUE, name text NOT NULL,
  max_weight_kg int NOT NULL CHECK (max_weight_kg > 0)
);   -- 6 rows

CREATE TABLE parties (              -- senders and recipients are the same shape
  id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  name text NOT NULL, phone text NOT NULL, email citext NULL,
  address text NOT NULL, city text NOT NULL, pin char(6) NOT NULL
    CHECK (pin ~ '^[1-9][0-9]{5}$'),
  CONSTRAINT uq_parties_phone_addr UNIQUE (phone, address)
);   -- ~412,000 rows

CREATE TABLE shipments (
  id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  tracking_no text NOT NULL UNIQUE,
  sender_id    bigint NOT NULL REFERENCES parties(id) ON DELETE RESTRICT,
  recipient_id bigint NOT NULL REFERENCES parties(id) ON DELETE RESTRICT,
  carrier_id   bigint NOT NULL REFERENCES carriers(id) ON DELETE RESTRICT,
  service_id   bigint NOT NULL REFERENCES service_levels(id) ON DELETE RESTRICT,
  weight_g     int    NOT NULL CHECK (weight_g > 0),      -- integer grams (T26)
  declared_value_minor bigint NOT NULL CHECK (declared_value_minor >= 0),
  created_at   timestamptz NOT NULL DEFAULT now(),
  delivered_at timestamptz NULL,
  CHECK (delivered_at IS NULL OR delivered_at >= created_at)
);
```

⚠ **One deliberate exception**, and it is important:

```sql
-- The RATE CHARGED is a fact about the SHIPMENT, not about the carrier
-- today. Carriers renegotiate rates; an invoice must not change.
ALTER TABLE shipments ADD COLUMN charged_rate_per_kg_minor bigint NOT NULL;
-- ★ This looks like redundancy. It is not: it is a DIFFERENT FACT.
--   carriers.rate_per_kg_minor = "what we charge today"
--   shipments.charged_rate…    = "what we charged for this shipment"
--   (Topics 21, 26 — snapshot vs reference; case study 04's I2.)
```

**Step 5 — the data cleaning is the hard part, and it needs humans.**

```sql
-- ① CARRIERS: 14 entities, 3 name variants each. A human decides.
CREATE TABLE carrier_name_map (raw text PRIMARY KEY, canonical_code text NOT NULL);
INSERT INTO carrier_name_map VALUES
 ('Blue Dart','BLUEDART'),('Blue Dart Express','BLUEDART'),
 ('BlueDart Express Ltd','BLUEDART'), …;
-- ★ And the SLA/rate variants: which of the 8 rates is correct?
--   ⇒ THE DATABASE CANNOT TELL YOU. This is why the redundancy is
--     expensive: recovering the truth requires going outside the system.

-- ② PARTIES: 826 rows per phone, with typos. Deduplicate by
--    (phone, normalised address), keep the MOST RECENT variant,
--    and record the mapping for audit.
CREATE TABLE party_dedup_map AS
SELECT DISTINCT ON (phone, lower(regexp_replace(address,'\s+',' ','g')))
       phone, address, name, city, pin
FROM shipments_old_senders ORDER BY 1,2, created_at DESC;

-- ③ THE 41 ZZTEST ROWS: delete them, and remove the
--    `AND tracking_no NOT LIKE 'ZZTEST%'` from 60 reports.
```

**Step 6 — results.**

| | Before | After |
|---|---|---|
| Storage | 890 GB | **310 GB** |
| Carrier names per carrier | up to 4 | **1, enforced** |
| Carrier SLA variants | up to 3 | **1, enforced** |
| Onboarding a new carrier | a fake shipment | **`INSERT INTO carriers`** |
| 7-year purge | blocked for 2 years | **`DELETE FROM shipments`, safe** |
| Reports with `NOT LIKE 'ZZTEST%'` | 60 | **0** |
| Enforceable constraints | 1 (the PK) | **14** |

**The storage saving is the least interesting number.** The important one is the last: fourteen business rules moved from "we hope the application does it" to "the engine will not permit otherwise."

---

## Common mistakes

**1. Thinking normalisation is about saving disk.**
- *Symptom:* "disk is cheap, we don't need to normalise."
- *Why it's wrong:* the storage saving is a side effect. The point is that a fact stored once can be constrained; a fact stored many times cannot. Disk is cheap; *unresolvable inconsistency* is not.

**2. Confusing denormalised with unnormalised.**
- *Symptom:* "we're denormalised for performance" about a schema nobody ever analysed.
- *Fix:* denormalisation means you normalised, then deliberately reintroduced a specific redundancy, with a measured reason and a reconciliation plan (Phase 6). If you can't name all three, you're unnormalised.

**3. Normalising a snapshot away.**
- *Symptom:* removing `unit_price` from `order_lines` because "the price is on `products`" — and every historical invoice changes when prices change.
- *Why it's subtle:* the price *paid* is functionally dependent on the order line, not on the product. It is a different fact that happens to have had the same value once. (Topic 30 makes this precise; Topic 38 revisits it.)

**4. Normalising before you understand the domain.**
- *Symptom:* a beautiful 3NF schema that cannot express a business rule.
- *Fix:* Topic 20 first. Normalisation tells you where a fact *belongs*; it cannot tell you which facts you *need*.

**5. Fearing joins without measuring them.**
- *Symptom:* a flat table "for performance" with no benchmark behind it.
- *Fix:* Topic 19's numbers. Three indexed joins for one order is 0.2 ms.

**6. Stopping the analysis at "it looks fine."**
- *Symptom:* an update anomaly discovered two years later.
- *Fix:* the detection queries in Example 2 step 2 — `count(DISTINCT x)` grouped by the thing that should determine it. Run them on any table you inherit.

---

## Hands-on proof

**PROVE IT #1 — the three anomalies.** (Example 1, step 1.)
**PROVE IT #2 — all three become impossible.** (Example 1, step 2.)
**PROVE IT #3 — storage and write cost.** (Example 1, step 3.)
**PROVE IT #4 — the read cost, measured.** (Example 1, step 4.)

**PROVE IT #5 — the detection query for update anomalies.** Run this against any wide table you inherit:
```sql
-- "does one X have more than one Y, when it should have exactly one?"
SELECT carrier_code, count(DISTINCT carrier_name) AS variants
FROM shipments GROUP BY 1 HAVING count(DISTINCT carrier_name) > 1;

-- generalised: for every pair of columns where the left should
-- determine the right
SELECT 'sender_phone → sender_name' AS dependency,
       count(*) FILTER (WHERE n > 1) AS violations
FROM (SELECT sender_phone, count(DISTINCT sender_name) n
      FROM shipments GROUP BY 1) t;
```

**PROVE IT #6 — the redundancy ratio.**
```sql
SELECT a.attname,
       count(*) AS total_rows,
       count(DISTINCT t.v) AS distinct_values
FROM shipments, LATERAL (SELECT 1) t(v)   -- (per column, run individually)
GROUP BY 1;
-- or simply:
SELECT count(*) AS rows, count(DISTINCT carrier_code) AS entities,
       round(count(*)::numeric / count(DISTINCT carrier_code)) AS repetition
FROM shipments;
-- repetition > ~100 on a text column is a strong signal of a missing table
```

**PROVE IT #7 — constraints that only become possible after normalising.**
```sql
-- on the flat table:
ALTER TABLE orders_flat ADD CONSTRAINT uq_phone UNIQUE (customer_phone);
-- ERROR: could not create unique index — duplicate key value
--   ★ the constraint you WANT is impossible to state

-- on the normalised table:
ALTER TABLE customers ADD CONSTRAINT uq_phone UNIQUE (phone);   -- ✓
```

---

## The design decision framework

```
THE DEFAULT: NORMALISE TO 3NF/BCNF FIRST. ALWAYS.
  It is the correctness baseline. You denormalise FROM it, deliberately,
  with evidence (Phase 6) — never INSTEAD of it.

FOR ANY TABLE, RUN THE THREE TESTS:

 ① THE UPDATE TEST
    "If one real-world fact changes, how many rows must I update?"
      1     → fine
      > 1   → ★ the fact is in the wrong place. Decompose.

 ② THE INSERT TEST
    "Can I record every fact I need, independently?"
      "I can't add a product until someone buys it"  → decompose
      Look for placeholder rows and `WHERE x IS NOT NULL` in queries —
      they are the fossil record of an insert anomaly.

 ③ THE DELETE TEST
    "If I delete one thing, do I lose a different thing?"
      "deleting the last order deletes the customer" → decompose

 AND THE FOURTH, WHICH IS THE REAL ONE:
 ④ THE CONSTRAINT TEST
    "Which business rules can the engine enforce on this table?"
      If a rule you care about cannot be expressed — a UNIQUE, a
      NOT NULL, an FK, an EXCLUDE — the fact it constrains is probably
      in the wrong table.

BEFORE DECOMPOSING, CHECK IT IS NOT A SNAPSHOT:
  "Is this value a COPY of another table's current value, or is it a
   FACT ABOUT THIS ROW that happened to equal it once?"
    order_line.unit_price   → a fact about the line. KEEP IT.
    order_line.product_name → a fact about the line (the invoice). KEEP IT.
    shipment.carrier_sla    → a copy of the carrier's current SLA. REMOVE IT.
  ⇒ The test: if the source changes, SHOULD this value change too?
      YES → it's a copy. Normalise it away.
      NO  → it's a snapshot. Keep it, and document why. (Topics 21, 26)

THE SIGNAL TO LOOK FOR — on any table you inherit:
      SELECT count(*) AS rows, count(DISTINCT <col>) AS entities,
             round(count(*)::numeric/count(DISTINCT <col>)) AS repetition
      FROM <table>;
  • repetition > 100 on a descriptive text column → a missing table
  Then confirm with the inconsistency detector:
      SELECT <key>, count(DISTINCT <dependent>) FROM <table>
      GROUP BY 1 HAVING count(DISTINCT <dependent>) > 1;
  • any row returned → an update anomaly has ALREADY HAPPENED
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
Build the `orders_flat` table from Example 1. Demonstrate all three anomalies with actual SQL, and for each write one sentence explaining which fact is stored in the wrong place. Then attempt to add a constraint that would prevent the update anomaly, and explain why no such constraint exists.

### Exercise 2 — medium (apply it)
Given this table, identify every fact and which key it really depends on:

```sql
CREATE TABLE enrolments (
  student_id bigint, student_name text, student_email text, student_year int,
  course_code text, course_title text, course_credits int,
  instructor_id bigint, instructor_name text, instructor_dept text,
  semester text, grade text,
  PRIMARY KEY (student_id, course_code, semester)
);
```
(a) List every column and the key it actually depends on.
(b) Produce all three anomalies with concrete examples.
(c) Decompose it, and show that the decomposition is lossless.
(d) Two columns look like redundancy but might be snapshots — identify them and argue both ways.

### Exercise 3 — hard (production simulation)
A `shipments` table has 340M rows, 890 GB, and stores sender, recipient, carrier and service-level attributes inline. Audit finds: 14 carriers with up to 4 name variants and 8 rate variants each; 412,000 senders with up to 11 address variants; 41 "ZZTEST" placeholder rows excluded by 60 reports; and a 7-year data purge blocked for two years because deleting shipments would delete party records.

(a) Write the queries that quantify the redundancy and detect the existing inconsistencies.
(b) Explain how a normalisation defect ended up blocking a legal compliance requirement.
(c) Design the normalised schema. Identify the one column that *looks* like redundancy but must be kept, and justify it.
(d) The carrier rate has 8 variants. Explain why the database cannot tell you which is correct, and what that implies about the cost of redundancy.
(e) Design the deduplication of 412,000 parties from 340M rows, including how you handle typos and address changes, and what you keep for audit.
(f) List the business rules that become enforceable after normalising, with the DDL for each.
(g) Give the migration order (using Topic 28's techniques) and estimate the storage and constraint outcomes.

---

## Mental model checkpoint

1. Name the three anomalies. Which one actually happens most in production, and why is it dangerous?
2. State in one sentence the cause of all three.
3. Why is "normalisation saves disk" the wrong headline? What's the right one?
4. What is the difference between denormalised and unnormalised?
5. Give the four tests you run on a table to decide whether it needs decomposing.
6. `order_lines.unit_price` duplicates `products.price`. Why is that *not* a normalisation defect? What's the test?
7. What does "lossless decomposition" mean, and why does it matter?

---

## Quick reference card

**The three anomalies**

| Anomaly | Symptom | Cause |
|---|---|---|
| **Update** | one fact in N rows; inconsistent states are legal | a fact keyed differently from the table |
| **Insert** | can't record a fact independently; placeholder rows | same |
| **Delete** | deleting one thing loses another | same |

**The cause, in one sentence:** one table storing facts about more than one kind of thing.

**The rule:** every non-key fact must depend on **the key, the whole key, and nothing but the key.**

**The four tests**

| Test | Question |
|---|---|
| Update | how many rows to change one fact? >1 → decompose |
| Insert | can I record every fact independently? |
| Delete | does deleting one thing lose another? |
| **Constraint** | which business rules can the engine enforce here? |

**Snapshot vs copy:** if the source changes, *should* this value change?
Yes → a copy, normalise it away. No → a snapshot, keep it and document why.

**The normal forms:** 1NF (atomic) → 2NF (no partial) → 3NF (no transitive) → BCNF (every determinant a key) → 4NF (no MVD) → 5NF (no join dependency). Cumulative. Most schemas stop at 3NF/BCNF.

**The detection query**

```sql
SELECT <key>, count(DISTINCT <dependent>) FROM <table>
GROUP BY 1 HAVING count(DISTINCT <dependent>) > 1;
-- any row → an update anomaly has already happened
```

---

## When would I use this at work?

1. **Inheriting any wide table.** The repetition-ratio query plus the inconsistency detector tell you in two minutes whether the schema has already corrupted itself — and it usually has.

2. **Pushing back on a "flat table for performance" proposal.** You can show that the read saving is ~0.1 ms and the cost is every business rule becoming unenforceable, plus three classes of silent corruption.

3. **When a compliance or reporting problem turns out to be structural.** "We can't purge old data because it would delete customers" is a delete anomaly, and naming it as one moves the conversation from a workaround to a fix.

---

## Connected topics

**Understand before this:** 20–21 (entities and their keys), 24 (constraints — what normalisation makes possible), 27 (the antipatterns this theory explains).

**This unlocks:**
- **30** — functional dependencies: the precise language for "depends on"
- **31–36** — the normal forms, each removing one specific failure
- **37** — a full worked decomposition
- **38** — when to stop, and the honest cost of over-normalising
- **53–55** — deliberate denormalisation, which only means something once you've done this
