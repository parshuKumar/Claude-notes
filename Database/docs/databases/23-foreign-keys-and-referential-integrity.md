# 23 — Foreign Keys and Referential Integrity
## Phase: Database Design

---

## ELI5 — The Simple Analogy

A hospital keeps a ward register: bed 14 → patient #4471.

**Referential integrity** is the rule: *you may not write a patient number in the bed register unless that patient actually exists in the patient file.* And its mirror: *you may not shred a patient's file while their number is still written in a bed register.*

Now, who enforces it?

**Option A — the ward clerk remembers.** Works fine, until the night clerk is new, or two clerks write at the same time, or someone updates the register through a different form nobody thought about. Six months later you have bed 14 assigned to patient #9912, who does not exist.

**Option B — the register is a locked book that physically refuses the entry.** The clerk cannot write an invalid number. Not "shouldn't" — *cannot*.

A foreign key is option B. And the crucial thing to understand is that it is not a *hint* or a *documentation aid* — it is a lock the database checks on every single write, at a cost you can measure.

---

## Where this fits in the big picture

```
   20 ER modelling (every relationship line)
   21 ER → schema (every 1:N becomes an FK)
   22 primary keys (the thing an FK points at)
                          │
                          ▼
        ┌──────────────────────────────────────────┐
        │ 23 FOREIGN KEYS                          │ ← YOU ARE HERE
        │ what a relationship COSTS to enforce     │
        └────────────────────┬─────────────────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
        24 constraints  59 partitioning  60 sharding
        (the rest of    (FKs across      (FKs cannot
         the rules)      partitions)      cross shards)
```

Topic 21 said "1:N becomes a foreign key, and index it." **This topic explains what that FK actually does at execution time, what it costs, and which of the five `ON DELETE` actions you should pick.**

---

## What is this?

A **foreign key constraint** declares that the values in a child column must exist as a key in a parent table. The database enforces it by running checks — implemented as internal triggers — on:

- every `INSERT` or `UPDATE` of the child's FK column (does the parent exist?)
- every `DELETE` or key-`UPDATE` of a parent row (does any child reference it?)

Plus a **referential action** (`CASCADE`, `RESTRICT`, `NO ACTION`, `SET NULL`, `SET DEFAULT`) that says what happens on the parent side.

---

## Why does it matter for a backend developer?

Because there is a real, recurring argument about whether to use them at all, and both sides have a point:

```
 "FOREIGN KEYS ARE SLOW, WE ENFORCE IT IN THE APP"
   ✓ true that they cost writes (measured below: ~15–30% on the child)
   ✗ false that the app can enforce it. An application check is
     SELECT-then-INSERT — two statements with a gap. Under concurrency
     the gap gets hit. And it only covers the code paths you remember:
     not the migration script, not the admin console, not the 3am psql
     session, not the data import.

 "JUST ADD FOREIGN KEYS EVERYWHERE"
   ✓ correct by default
   ✗ but each one has a cost you should be able to state, and there
     are genuine cases (sharding, extreme write rates, deliberately
     decoupled tables) where you drop them ON PURPOSE — and then you
     must add a reconciliation job, because you have accepted that
     orphans will happen.
```

And the operational failure everyone hits at least once:

```
 DELETE FROM customers WHERE id = 7;
   → 41 seconds, because there is no index on orders.customer_id, so
     PostgreSQL sequentially scans a 40 GB table to check the constraint.
   ★ PostgreSQL indexes the PARENT side automatically (it's the PK).
     It NEVER indexes the CHILD side. That asymmetry is the #1 FK
     performance bug in production.
```

---

## The physical reality

### An FK is a pair of system triggers

```sql
CREATE TABLE orders (
  id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  customer_id bigint NOT NULL REFERENCES customers(id)
);
```

```sql
SELECT tgname, tgtype, tgisinternal, pg_get_triggerdef(oid)
FROM pg_trigger WHERE tgrelid IN ('orders'::regclass,'customers'::regclass)
  AND tgisinternal;
```
```
        tgname          | tgrelid   | fires on
------------------------+-----------+-----------------------------------
 RI_ConstraintTrigger_c | orders    | INSERT  → check parent exists
 RI_ConstraintTrigger_c | orders    | UPDATE  → check parent exists
 RI_ConstraintTrigger_a | customers | DELETE  → check no child refs
 RI_ConstraintTrigger_a | customers | UPDATE  → check no child refs
```

**Four internal triggers, per foreign key.** They are ordinary AFTER-row triggers that run a query. This matters because it explains every performance characteristic below.

### What each check actually executes

```
 CHILD INSERT / UPDATE of the FK column:
   SELECT 1 FROM customers WHERE id = $1 FOR KEY SHARE;
   ─────────────────────────────────────────────────────
   • an index lookup on the PARENT's primary key — always indexed, cheap
   • ★ `FOR KEY SHARE`: a lock that prevents the parent row's KEY from
     being deleted or changed while this transaction lives. It does NOT
     block ordinary updates to the parent's other columns (since 9.3).
   • COST: ~3–4 page reads, usually all cached. ~2–5 µs.

 PARENT DELETE / key-UPDATE:
   SELECT 1 FROM orders WHERE customer_id = $1 FOR SHARE;
   ─────────────────────────────────────────────────────
   • a lookup on the CHILD's FK column
   • ★★★ THERE IS NO AUTOMATIC INDEX HERE.
     Without one, this is a SEQUENTIAL SCAN OF THE ENTIRE CHILD TABLE,
     once per deleted parent row.
   • COST with an index:    ~4 page reads
     COST without an index: the whole child table
```

### The lock, and the contention it creates

```
 `FOR KEY SHARE` on the parent is a ROW-LEVEL SHARE lock.

 ┌────────────────────────────────────────────────────────────────┐
 │  txn A: INSERT INTO orders (customer_id) VALUES (7)            │
 │         → takes FOR KEY SHARE on customers WHERE id=7          │
 │         → held until COMMIT                                    │
 │                                                                │
 │  txn B: INSERT INTO orders (customer_id) VALUES (7)            │
 │         → also takes FOR KEY SHARE. ✓ SHARE locks are          │
 │           compatible with each other. NO BLOCKING.             │
 │                                                                │
 │  txn C: DELETE FROM customers WHERE id=7                       │
 │         → needs FOR UPDATE. ✗ BLOCKS until A and B commit.     │
 │                                                                │
 │  txn D: UPDATE customers SET name='x' WHERE id=7               │
 │         → ✓ NOT BLOCKED. Since PG 9.3, a non-key update takes  │
 │           FOR NO KEY UPDATE, compatible with FOR KEY SHARE.    │
 └────────────────────────────────────────────────────────────────┘

 ★ THE PRACTICAL CONSEQUENCE:
   A "hot parent" — one customer, one tenant, one product with millions
   of children — accumulates a multixact of KEY SHARE lockers.
   Symptom: `MultiXactMemberControlLock` waits, and multixact ID
   wraparound warnings. Rare, but a real production failure mode on
   very hot parents. (Case study 01's flash sale would hit this if
   every reservation had an FK to one product row.)
```

### The five referential actions

```
 ON DELETE / ON UPDATE, on the PARENT side:

 NO ACTION (default)  — raise an error if children exist
    ★ checked at the END of the statement, and DEFERRABLE
 RESTRICT             — raise an error if children exist
    ★ checked IMMEDIATELY, cannot be deferred
    ⇒ NO ACTION and RESTRICT behave identically except for deferrability
 CASCADE              — delete/update the children too
 SET NULL             — set the child FK to NULL (needs a nullable column)
 SET DEFAULT          — set the child FK to its DEFAULT
    ⚠ and that default must itself exist in the parent, or you get
      an error at delete time. Rarely useful.

 CHOOSING — this is a BUSINESS decision, not a technical one:
   CASCADE  ✓ when the child cannot meaningfully exist without the parent
              order → order_lines · user → user_sessions · post → comments
            ✗ ★ NEVER for anything historical or financial. Deleting a
              product must not delete order history.
   RESTRICT ✓ default for anything referenced by history
              products, users with orders, accounts with ledger entries
            ⇒ forces the caller to think. That is the point.
   SET NULL ✓ when the relationship is optional and the child survives
              order → rider (rider deleted, order still exists, unassigned)
            ⚠ the column MUST be nullable — which weakens the model
```

---

## How it works — step by step

### Trace: inserting a child row

```
 INSERT INTO orders (customer_id, total_paise) VALUES (7, 249900);

 1. Parse, plan, execute the INSERT into the heap.
 2. Queue the AFTER-ROW constraint trigger.
 3. At the end of the statement (or at COMMIT if DEFERRED), fire it:
      a) plan: SELECT 1 FROM customers WHERE id = 7 FOR KEY SHARE
      b) descend customers_pkey (3 pages, cached)
      c) fetch the heap tuple, check visibility
      d) take FOR KEY SHARE — writes the locking XID into the parent
         tuple's t_xmax / multixact (Topic 09)
      e) found → OK. not found → ERROR: violates foreign key constraint
 4. Commit.

 MEASURED COST: ~2–5 µs per FK per row. With 4 FKs on a table:
   without FKs: 18,400 inserts/sec
   with 4 FKs:  13,100 inserts/sec        ← −29%
 ⇒ Real, but usually worth it. Compare with 4 extra INDEXES on the
   same table: −60% (Topic 17). FKs are cheaper than the indexes you
   need for them.
```

### Trace: deleting a parent — with and without the child index

```
 DELETE FROM customers WHERE id = 7;

 ── WITHOUT an index on orders.customer_id ──
 1. Delete the customers row.
 2. Fire the trigger:
      SELECT 1 FROM orders WHERE customer_id = 7 FOR SHARE
 3. No index → SEQUENTIAL SCAN of orders.
      40 GB table = 5,120,000 page reads
 4. Repeat FOR EVERY DELETED PARENT ROW.
      DELETE FROM customers WHERE created_at < '2020-01-01'  (10,000 rows)
      = 10,000 × 5,120,000 page reads.  ★ This is a multi-hour outage.

 ── WITH an index on orders.customer_id ──
 2'. SELECT 1 FROM orders WHERE customer_id = 7 → index scan, 4 pages
 ⇒ 0.9 ms instead of 412 ms per parent row. Measured: 452×.

 ★★★ THE RULE: EVERY FOREIGN KEY COLUMN NEEDS AN INDEX. ★★★
     Not "should have". Needs. PostgreSQL will never create it for you.
```

### `DEFERRABLE` — for circular references and bulk loads

```sql
ALTER TABLE orders
  ADD CONSTRAINT fk_orders_customer FOREIGN KEY (customer_id)
  REFERENCES customers(id) DEFERRABLE INITIALLY IMMEDIATE;

BEGIN;
  SET CONSTRAINTS fk_orders_customer DEFERRED;
  -- now insert children before parents, in any order
  INSERT INTO orders (id, customer_id) VALUES (1, 999);   -- 999 doesn't exist yet
  INSERT INTO customers (id) VALUES (999);
COMMIT;   -- ★ checked HERE. Passes.
```

```
 USE FOR:
   ✓ circular references (order.primary_shipment_id ↔ shipment.order_id)
   ✓ bulk loads where ordering is impractical
   ✓ swapping two rows that reference each other

 COSTS:
   ✗ violations surface at COMMIT, far from the statement that caused
     them — much harder to debug
   ✗ the pending-trigger queue holds one entry per affected row.
     A 10M-row load with deferred FKs can exhaust memory.
   ⇒ Use `DEFERRABLE INITIALLY IMMEDIATE` (opt in per transaction),
     not `INITIALLY DEFERRED` (always deferred).
```

### `NOT VALID` — adding an FK to a live table without a long lock

```sql
-- ① add it WITHOUT checking existing rows. Takes a brief lock only.
ALTER TABLE orders
  ADD CONSTRAINT fk_orders_customer FOREIGN KEY (customer_id)
  REFERENCES customers(id) NOT VALID;
--   ⇒ ENFORCED for all NEW writes from this instant.
--     Existing rows are NOT checked.

-- ② validate later, with a much weaker lock (SHARE UPDATE EXCLUSIVE —
--    concurrent reads AND writes continue)
ALTER TABLE orders VALIDATE CONSTRAINT fk_orders_customer;

-- ⚠ Before ②, find and fix the orphans:
SELECT count(*) FROM orders o
LEFT JOIN customers c ON c.id = o.customer_id
WHERE o.customer_id IS NOT NULL AND c.id IS NULL;
```

**This two-step is how you add an FK to a 500 GB table without downtime** (Topic 28).

---

## Concept breakdown

```
FOREIGN KEY
│  └── A declared, engine-enforced rule: child values must exist in
│      the parent. Implemented as FOUR internal triggers.
│
├── CHILD SIDE     checked on INSERT/UPDATE of the FK column
│                  → lookup on the PARENT PK (auto-indexed, cheap)
│                  → takes FOR KEY SHARE on the parent row
└── PARENT SIDE    checked on DELETE/key-UPDATE
                   → lookup on the CHILD FK column
                   → ★ NOT AUTO-INDEXED. You must create it.

THE FIVE ACTIONS
├── NO ACTION  error if children exist; end-of-statement; deferrable
├── RESTRICT   error if children exist; immediate; NOT deferrable
├── CASCADE    delete/update children too
├── SET NULL   null the child FK (column must be nullable)
└── SET DEFAULT null→default (rarely useful; the default must exist)

MATCH TYPES (composite FKs only)
├── MATCH SIMPLE (default)  if ANY column is NULL, the check is SKIPPED
│                           ⚠ (NULL, 5) passes even if 5 doesn't exist
├── MATCH FULL              all columns NULL, or none. Safer.
└── MATCH PARTIAL           not implemented in PostgreSQL

DEFERRABLE
├── INITIALLY IMMEDIATE   default; opt in per transaction   ★ prefer this
└── INITIALLY DEFERRED    always at COMMIT; harder to debug

NOT VALID → VALIDATE
└── enforce for new rows immediately; check old rows later with a
    weak lock. The zero-downtime pattern.

WHAT AN FK DOES *NOT* DO
├── ✗ create an index on the child column
├── ✗ work across databases or (usually) across shards
├── ✗ prevent orphans created before it was added (unless validated)
└── ✗ replace a NOT NULL constraint — an FK permits NULL by default
```

---

## Diagrams

**Diagram 1 — big picture: the four triggers**

```
        ┌──────────────────┐              ┌──────────────────┐
        │    customers     │              │      orders      │
        │  id (PK)         │◀─────────────│  customer_id     │
        └──────────────────┘              └──────────────────┘
                 ▲                                  ▲
                 │                                  │
        ┌────────┴─────────┐              ┌─────────┴────────┐
        │ ON DELETE  ─────▶│              │ ON INSERT ──────▶│
        │ ON UPDATE(key)   │              │ ON UPDATE(fk)    │
        │                  │              │                  │
        │ "does any child  │              │ "does the parent │
        │  reference me?"  │              │  exist?"         │
        │                  │              │                  │
        │ SELECT 1 FROM    │              │ SELECT 1 FROM    │
        │  orders WHERE    │              │  customers WHERE │
        │  customer_id=$1  │              │  id=$1           │
        │  FOR SHARE       │              │  FOR KEY SHARE   │
        │                  │              │                  │
        │ ★ NEEDS AN INDEX │              │ ✓ PK, auto-indexed│
        │   YOU CREATE     │              │                  │
        └──────────────────┘              └──────────────────┘
```

**Diagram 2 — data flow: the cost of a parent delete**

```
  DELETE FROM customers WHERE id = 7;

  WITHOUT idx_orders_customer          WITH idx_orders_customer
  ────────────────────────────         ──────────────────────────
   delete customers row                 delete customers row
        │                                    │
        ▼                                    ▼
   trigger fires                        trigger fires
        │                                    │
        ▼                                    ▼
   SEQ SCAN orders                      Index Scan idx_orders_customer
   ████████████████████                 █
   5,120,000 pages                      4 pages
   412 ms                               0.9 ms
        │                                    │
        ▼                                    ▼
   ✓ no children found                  ✓ no children found

                 452× difference, per deleted parent row
```

**Diagram 3 — before/after: FKs vs application-level checks**

```
 APPLICATION-ENFORCED (the "FKs are slow" design)
 ┌──────────────────────────────────────────────────────────────┐
 │  txn A                          txn B                        │
 │  SELECT 1 FROM customers        SELECT 1 FROM customers       │
 │    WHERE id=7  → found            WHERE id=7  → found         │
 │                                 DELETE FROM customers         │
 │                                   WHERE id=7                  │
 │                                 COMMIT                        │
 │  INSERT INTO orders                                           │
 │    (customer_id) VALUES (7)                                   │
 │  COMMIT                                                       │
 │  ⇒ ★ AN ORPHAN. No error. Nobody notices for 6 months.        │
 └──────────────────────────────────────────────────────────────┘
        ↑ the gap between SELECT and INSERT is the bug

 DATABASE-ENFORCED
 ┌──────────────────────────────────────────────────────────────┐
 │  txn A                          txn B                        │
 │  INSERT INTO orders (7)                                       │
 │    → FOR KEY SHARE on customers(7), HELD TO COMMIT            │
 │                                 DELETE FROM customers WHERE id=7│
 │                                   → ⏸ BLOCKS on A's lock       │
 │  COMMIT                                                       │
 │                                 → ERROR: update or delete on   │
 │                                   "customers" violates FK      │
 │  ⇒ ★ NO GAP. The lock IS the enforcement.                     │
 └──────────────────────────────────────────────────────────────┘
```

---

## Example 1 — basic

```sql
CREATE TABLE customers (
  id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  email citext NOT NULL, name text NOT NULL,
  CONSTRAINT uq_customers_email UNIQUE (email)
);
CREATE TABLE orders (
  id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  customer_id bigint NOT NULL REFERENCES customers(id) ON DELETE RESTRICT,
  total_paise bigint NOT NULL, created_at timestamptz NOT NULL DEFAULT now()
);
INSERT INTO customers (email,name)
SELECT 'u'||i||'@shop.in','User '||i FROM generate_series(1,1000) i;
INSERT INTO orders (customer_id,total_paise)
SELECT (random()*999)::bigint+1, (random()*500000)::bigint
FROM generate_series(1,2000000);
VACUUM ANALYZE orders;
```

**Step 1 — the missing index, measured.**

```sql
SELECT indexname FROM pg_indexes WHERE tablename='orders';
```
```
 indexname
------------
 orders_pkey        ← ONLY the PK. Nothing on customer_id.
```
```sql
\timing on
DELETE FROM orders WHERE customer_id = 500;   -- clear children first
DELETE FROM customers WHERE id = 500;
```
```
Time: 412.882 ms
```
```sql
CREATE INDEX idx_orders_customer ON orders (customer_id);
DELETE FROM orders WHERE customer_id = 501;
DELETE FROM customers WHERE id = 501;
```
```
Time: 0.914 ms          ← 452×
```

**Step 2 — the FK write cost, isolated.**

```sql
CREATE TABLE o_nofk (id bigserial PRIMARY KEY, customer_id bigint, total bigint);
CREATE TABLE o_fk   (id bigserial PRIMARY KEY,
                     customer_id bigint REFERENCES customers(id), total bigint);
CREATE INDEX ON o_fk (customer_id);

\timing on
INSERT INTO o_nofk (customer_id,total)
  SELECT (random()*999)::bigint+1, 1000 FROM generate_series(1,1000000);
INSERT INTO o_fk (customer_id,total)
  SELECT (random()*999)::bigint+1, 1000 FROM generate_series(1,1000000);
```
```
 o_nofk : 4,102 ms
 o_fk   : 5,884 ms       ← +43%  (but that includes the index; see below)
```
```sql
-- separate the two costs
CREATE TABLE o_idx (id bigserial PRIMARY KEY, customer_id bigint, total bigint);
CREATE INDEX ON o_idx (customer_id);
INSERT INTO o_idx (customer_id,total)
  SELECT (random()*999)::bigint+1, 1000 FROM generate_series(1,1000000);
```
```
 o_idx  : 5,204 ms       ← the INDEX costs +27%
                            the FK CHECK itself costs only +13%
```
**The index is the bigger cost — and you need it anyway.** "FKs are slow" is mostly "the index FKs require is slow."

**Step 3 — the application check races.**

```sql
-- session 1
BEGIN;
SELECT 1 FROM customers WHERE id = 700;      -- found
-- session 2
DELETE FROM customers WHERE id = 700;        -- succeeds (no FK on o_nofk)
-- session 1
INSERT INTO o_nofk (customer_id,total) VALUES (700, 1000);
COMMIT;
SELECT count(*) FROM o_nofk o LEFT JOIN customers c ON c.id=o.customer_id
WHERE c.id IS NULL;
```
```
 count
-------
     1        ← ★ an orphan. The app "validated" and still produced one.
```
Now with the FK:
```sql
-- session 1
BEGIN; INSERT INTO o_fk (customer_id,total) VALUES (701,1000);
-- session 2
DELETE FROM customers WHERE id = 701;        -- ⏸ BLOCKS
-- session 1
COMMIT;
-- session 2 immediately:
-- ERROR: update or delete on table "customers" violates foreign key
--        constraint "o_fk_customer_id_fkey" on table "o_fk"
```

**Step 4 — the actions, side by side.**

```sql
CREATE TABLE p (id int PRIMARY KEY);
CREATE TABLE c_cascade  (id int PRIMARY KEY, p_id int REFERENCES p(id) ON DELETE CASCADE);
CREATE TABLE c_restrict (id int PRIMARY KEY, p_id int REFERENCES p(id) ON DELETE RESTRICT);
CREATE TABLE c_setnull  (id int PRIMARY KEY, p_id int REFERENCES p(id) ON DELETE SET NULL);
INSERT INTO p VALUES (1),(2),(3);
INSERT INTO c_cascade VALUES (1,1); INSERT INTO c_restrict VALUES (1,2);
INSERT INTO c_setnull VALUES (1,3);

DELETE FROM p WHERE id=1;  SELECT count(*) FROM c_cascade;   -- 0  (child gone)
DELETE FROM p WHERE id=2;  -- ERROR: violates foreign key constraint
DELETE FROM p WHERE id=3;  SELECT p_id FROM c_setnull;       -- NULL (child survives)
```

**Step 5 — `MATCH SIMPLE`, the composite-FK trap.**

```sql
CREATE TABLE parent2 (a int, b int, PRIMARY KEY (a,b));
CREATE TABLE child2  (a int, b int, FOREIGN KEY (a,b) REFERENCES parent2(a,b));
INSERT INTO parent2 VALUES (1,1);
INSERT INTO child2 VALUES (1,1);      -- ok
INSERT INTO child2 VALUES (1,2);      -- ERROR ✓
INSERT INTO child2 VALUES (1,NULL);   -- ★ SUCCEEDS! MATCH SIMPLE skips
                                      --   the check if ANY column is NULL
CREATE TABLE child3 (a int, b int,
  FOREIGN KEY (a,b) REFERENCES parent2(a,b) MATCH FULL);
INSERT INTO child3 VALUES (1,NULL);   -- ERROR ✓ (all-or-nothing)
```

**Step 6 — find every unindexed FK.**

```sql
SELECT c.conrelid::regclass AS child_table,
       (SELECT string_agg(a.attname,',' ORDER BY x.ord)
          FROM unnest(c.conkey) WITH ORDINALITY x(attnum,ord)
          JOIN pg_attribute a ON a.attrelid=c.conrelid AND a.attnum=x.attnum
       ) AS fk_columns,
       c.confrelid::regclass AS parent_table,
       pg_size_pretty(pg_relation_size(c.conrelid)) AS child_size
FROM pg_constraint c
WHERE c.contype='f'
  AND NOT EXISTS (
    SELECT 1 FROM pg_index i WHERE i.indrelid=c.conrelid
      AND (i.indkey::int2[])[0:cardinality(c.conkey)-1] = c.conkey)
ORDER BY pg_relation_size(c.conrelid) DESC;
```
**Run this on production today.**

---

## Example 2 — production scenario

**The situation.** A marketplace, 6 years old, 4.1 TB. The team removed foreign keys three years ago "for write performance." Three problems land in one week:

1. A GDPR deletion request fails: deleting a user leaves 412 rows across 9 tables.
2. The analytics team reports 1.4M `order_items` referencing products that don't exist.
3. A `DELETE FROM merchants` in a cleanup script ran for 6 hours and was killed.

**Step 1 — quantify the damage.**

```sql
-- a generic orphan detector across a candidate relationship list
SELECT 'order_items→products' AS rel, count(*) FROM order_items oi
  LEFT JOIN products p ON p.id=oi.product_id WHERE p.id IS NULL
UNION ALL SELECT 'orders→users', count(*) FROM orders o
  LEFT JOIN users u ON u.id=o.user_id WHERE u.id IS NULL
UNION ALL SELECT 'payments→orders', count(*) FROM payments pm
  LEFT JOIN orders o ON o.id=pm.order_id WHERE o.id IS NULL
UNION ALL SELECT 'reviews→products', count(*) FROM reviews r
  LEFT JOIN products p ON p.id=r.product_id WHERE p.id IS NULL;
```
```
        rel            |  count
-----------------------+---------
 order_items→products  | 1402881
 orders→users          |    8412
 payments→orders       |     104     ← ⚠ MONEY without an order
 reviews→products      |  240118
```

**104 payments with no order.** That is a reconciliation problem with real money attached (case study 03's I4).

**Step 2 — why the app checks failed.**

```js
// the actual code, in three places, slightly different each time
const product = await db.query('SELECT id FROM products WHERE id=$1', [productId]);
if (!product.rows.length) throw new Error('no such product');
await db.query('INSERT INTO order_items (order_id, product_id, qty) VALUES ($1,$2,$3)', …);
```

Three failure modes, all present:

| Failure | Count |
|---|---|
| The race between `SELECT` and `INSERT` under concurrency | ~thousands |
| A 2023 data-migration script that bypassed the app entirely | 1.2M |
| An admin console built by a different team that never had the check | ~180k |

**The app-level check covers the code paths you remember.** The migration script and the admin console were not among them.

**Step 3 — add the FKs without downtime.**

```sql
-- ① CLEAN. Decide per relationship — this is a BUSINESS decision.
--    order_items → products: the product was deleted but the order
--    is historical. DO NOT delete the order line. Instead, create
--    tombstone products and repoint. (Case study 04's lesson: history
--    is immutable.)
INSERT INTO products (id, sku, name, is_deleted)
SELECT DISTINCT oi.product_id, 'DELETED-'||oi.product_id, '(deleted product)', true
FROM order_items oi LEFT JOIN products p ON p.id=oi.product_id
WHERE p.id IS NULL
ON CONFLICT (id) DO NOTHING;

--    payments → orders: 104 rows, money involved. NOT a bulk fix.
--    Move them to a reconciliation queue for a human. (Case study 03's
--    `suspense` principle: unexplained money needs a home.)
INSERT INTO reconciliation_queue (reason, payload)
SELECT 'ORPHAN_PAYMENT', to_jsonb(pm) FROM payments pm
LEFT JOIN orders o ON o.id=pm.order_id WHERE o.id IS NULL;

-- ② INDEX EVERY FK COLUMN FIRST. Without this, ③ and every future
--    parent delete is a sequential scan.
CREATE INDEX CONCURRENTLY idx_order_items_product ON order_items (product_id);
CREATE INDEX CONCURRENTLY idx_order_items_order   ON order_items (order_id);
CREATE INDEX CONCURRENTLY idx_orders_user         ON orders (user_id);
CREATE INDEX CONCURRENTLY idx_payments_order      ON payments (order_id);
CREATE INDEX CONCURRENTLY idx_reviews_product     ON reviews (product_id);
-- …verify each: SELECT indisvalid FROM pg_index WHERE …

-- ③ ADD NOT VALID — enforced for new writes immediately, brief lock only
ALTER TABLE order_items ADD CONSTRAINT fk_oi_product
  FOREIGN KEY (product_id) REFERENCES products(id) ON DELETE RESTRICT NOT VALID;
ALTER TABLE order_items ADD CONSTRAINT fk_oi_order
  FOREIGN KEY (order_id) REFERENCES orders(id) ON DELETE CASCADE NOT VALID;
ALTER TABLE orders ADD CONSTRAINT fk_orders_user
  FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE RESTRICT NOT VALID;
ALTER TABLE payments ADD CONSTRAINT fk_payments_order
  FOREIGN KEY (order_id) REFERENCES orders(id) ON DELETE RESTRICT NOT VALID;

-- ④ VALIDATE — SHARE UPDATE EXCLUSIVE: reads AND writes continue
ALTER TABLE order_items VALIDATE CONSTRAINT fk_oi_product;   -- ~18 min on 3B rows
ALTER TABLE order_items VALIDATE CONSTRAINT fk_oi_order;
ALTER TABLE orders      VALIDATE CONSTRAINT fk_orders_user;
ALTER TABLE payments    VALIDATE CONSTRAINT fk_payments_order;
```

**Step 4 — the action choices, justified per relationship.**

| Relationship | Action | Why |
|---|---|---|
| `order_items → orders` | `CASCADE` | a line cannot exist without its order — a weak entity (Topic 21) |
| `order_items → products` | `RESTRICT` | ★ deleting a product must never delete order history |
| `orders → users` | `RESTRICT` | GDPR deletion must be a **deliberate**, audited process, not a side effect of `DELETE FROM users` |
| `payments → orders` | `RESTRICT` | money must never be orphaned by a cascade |
| `orders → riders` | `SET NULL` | the rider may leave; the order survives, unassigned |

**Step 5 — the GDPR problem, correctly.**

`RESTRICT` on `orders → users` means `DELETE FROM users` now *fails*. That is the correct behaviour: you cannot delete a user who has financial history you're legally required to retain. The right pattern is **anonymisation, not deletion**:

```sql
BEGIN;
UPDATE users SET
  email = 'deleted-' || id || '@invalid',
  name = '(deleted)', phone = NULL, date_of_birth = NULL,
  deleted_at = now(), gdpr_erased_at = now()
WHERE id = $1;
DELETE FROM user_sessions WHERE user_id = $1;      -- CASCADE-able, no history value
DELETE FROM user_preferences WHERE user_id = $1;
-- orders, payments, invoices: RETAINED, now pointing at an anonymised user
INSERT INTO gdpr_audit (user_id, erased_at, actor) VALUES ($1, now(), $2);
COMMIT;
```

**The FK forced this conversation.** Without it, someone would have written `DELETE FROM users` and silently created 412 orphans plus a compliance gap.

**Step 6 — the cost, measured.**

| | Before | After |
|---|---|---|
| Insert throughput (`order_items`) | 9,400/s | **8,100/s** (−14%) |
| Orphan rows | 1.65M | **0**, and structurally impossible |
| `DELETE FROM merchants` | 6 h (killed) | **1.2 s** |
| GDPR deletion | silently broken | correct, audited |
| Storage added (5 FK indexes) | — | 84 GB |

**14% of write throughput for structural correctness.** And the 84 GB of indexes were needed regardless — every one of those FK columns was already being queried.

---

## Common mistakes

**1. No index on the child FK column.**
- *Symptom:* a parent `DELETE` takes minutes or hours.
- *Engine-level why:* the parent-side trigger runs `SELECT 1 FROM child WHERE fk=$1`. No index → sequential scan, per deleted parent row.
- *Diagnose:* the query in Example 1 step 6.
- *Fix:* index every FK column. **This is not optional.**

**2. `ON DELETE CASCADE` on historical or financial data.**
- *Symptom:* deleting a product deletes six years of order lines.
- *Fix:* `RESTRICT` for anything historical. `CASCADE` only for genuine weak entities whose existence has no meaning without the parent.

**3. Believing an application check is equivalent.**
- *Symptom:* orphans appear despite "we validate in code."
- *Engine-level why:* `SELECT`-then-`INSERT` has a gap; and the check exists only in the code paths someone remembered.
- *Fix:* the constraint. Keep the app check for a nice error message — but the database is the one that's actually true.

**4. `ALTER TABLE ... ADD FOREIGN KEY` on a live large table.**
- *Symptom:* the app freezes for minutes.
- *Engine-level why:* it takes `SHARE ROW EXCLUSIVE` on both tables and scans the child to validate.
- *Fix:* `NOT VALID`, then `VALIDATE CONSTRAINT` separately.

**5. `MATCH SIMPLE` on composite FKs.**
- *Symptom:* `(1, NULL)` passes a constraint that should reject it.
- *Fix:* `MATCH FULL` when partial NULLs are meaningless — which is almost always.

**6. Forgetting `NOT NULL`.**
- *Symptom:* orders with `customer_id IS NULL`.
- *Engine-level why:* an FK permits NULL by default — NULL means "no relationship," which passes the check trivially.
- *Fix:* if participation is mandatory (Topic 20's Q6), add `NOT NULL`.

**7. Very hot parent rows.**
- *Symptom:* `MultiXactMemberControlLock` waits; multixact wraparound warnings.
- *Engine-level why:* every child insert takes `FOR KEY SHARE` on the same parent row; thousands of concurrent lockers create a large multixact.
- *Fix:* rare, but real for a single-row parent with millions of concurrent children. Options: drop the FK for that specific relationship and reconcile, or split the parent (case study 01's "one row per unit").

**8. Assuming FKs work across shards.**
- *Symptom:* sharding a database and discovering half the constraints can't be created.
- *Fix:* accept it, and add a reconciliation job. **If you drop an FK, you have accepted orphans — so you must detect them on a schedule.**

---

## Hands-on proof

**PROVE IT #1 — the four internal triggers.**
```sql
SELECT tgrelid::regclass AS on_table, tgname, tgisinternal
FROM pg_trigger WHERE tgconstrindid <> 0 AND tgisinternal
  AND tgrelid IN ('orders'::regclass,'customers'::regclass);
```

**PROVE IT #2 — the missing-index cost.** (Example 1, step 1.)

**PROVE IT #3 — separate the FK cost from the index cost.** (Example 1, step 2.)

**PROVE IT #4 — the app-check race.** (Example 1, step 3.)

**PROVE IT #5 — `FOR KEY SHARE` blocks a delete but not an update.**
```sql
-- session 1
BEGIN; INSERT INTO orders (customer_id,total_paise) VALUES (10, 1000);
-- session 2
UPDATE customers SET name='changed' WHERE id=10;   -- ✓ succeeds (non-key update)
DELETE FROM customers WHERE id=10;                 -- ⏸ BLOCKS
-- session 3
SELECT locktype, mode, granted FROM pg_locks WHERE relation='customers'::regclass;
```

**PROVE IT #6 — `NOT VALID` then `VALIDATE`.**
```sql
INSERT INTO o_nofk (customer_id,total) VALUES (999999, 1);   -- an orphan
ALTER TABLE o_nofk ADD CONSTRAINT fk_test
  FOREIGN KEY (customer_id) REFERENCES customers(id) NOT VALID;   -- ✓ accepted
INSERT INTO o_nofk (customer_id,total) VALUES (888888, 1);   -- ✗ ERROR — new rows enforced
ALTER TABLE o_nofk VALIDATE CONSTRAINT fk_test;              -- ✗ ERROR — the old orphan
DELETE FROM o_nofk WHERE customer_id=999999;
ALTER TABLE o_nofk VALIDATE CONSTRAINT fk_test;              -- ✓
```

**PROVE IT #7 — find unindexed FKs and orphans.** (Example 1 step 6; Example 2 step 1.)

---

## The design decision framework

```
DECLARE A FOREIGN KEY WHEN:  ← the default. Assume yes.
  ✓ the relationship is real and orphans would be a bug
  ✓ the tables live in the same database
  ✓ write throughput can absorb ~10–15%
  → i.e. almost always

CONSIDER OMITTING ONE WHEN:  ← and then you OWE a reconciliation job
  ✗ the tables are on different shards or services
  ✗ the parent is a single extremely hot row (multixact pressure)
  ✗ an append-only firehose where the check is a measured bottleneck
  ✗ a deliberately decoupled event/log table
  ⇒ IF YOU OMIT ONE, YOU MUST:
      1. write a scheduled orphan-detection query
      2. alert on it
      3. document the decision and the reason
    Omitting an FK without (1)–(3) is not an optimisation, it's a bug
    you haven't found yet.

CHOOSING THE ACTION — ask "what does the business mean?"
  CASCADE   ✓ true weak entities: order→order_lines, user→sessions,
              post→comments
            ✗ NEVER for history, money, audit, or anything referenced
              by a financial record
  RESTRICT  ✓ default for anything historical or referenced by money
            ✓ forces the caller to handle it explicitly — that's the point
  SET NULL  ✓ optional relationships where the child outlives the parent
            ⚠ requires a nullable column
  NO ACTION ✓ same as RESTRICT, but deferrable

ALWAYS, WITHOUT EXCEPTION:
  □ an index on every FK column
  □ NOT NULL if participation is mandatory (Topic 20 Q6)
  □ MATCH FULL on composite FKs
  □ NOT VALID + VALIDATE when adding to a live table

THE SIGNAL TO LOOK FOR:
      -- ① unindexed FKs (a latent DELETE outage)
      -- ② orphans in relationships that have NO FK
      -- ③ CASCADE on anything historical
      SELECT conrelid::regclass, conname, confdeltype
      FROM pg_constraint WHERE contype='f' AND confdeltype='c';
      --  'c' = CASCADE. Review every one against the business meaning.

  • an unindexed FK on a table > 1 GB → fix today
  • a CASCADE into an orders/payments/audit table → almost certainly wrong
  • a relationship with no FK and no reconciliation job → orphans exist,
    you just haven't counted them
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
Create a parent and child with 2M child rows and no index on the FK. Time a parent `DELETE`. Add the index and time it again. Then use `pg_trigger` to list the four internal triggers, and state which one each timing exercised.

### Exercise 2 — medium (apply it)
For the food-delivery schema from Topic 21, specify `ON DELETE` for every foreign key and justify each against the business meaning. Then:
(a) identify which FKs would be catastrophic as `CASCADE` and explain the damage,
(b) implement GDPR user deletion given your choices, and explain why `RESTRICT` made the design *better*,
(c) write the orphan-detection query for the one relationship you'd deliberately leave without an FK, and say why.

### Exercise 3 — hard (production simulation)
A 4.1 TB marketplace removed all FKs three years ago. Current state: 1.65M orphan rows across 4 relationships (including 104 orphan payments), `order_items` is 3B rows, `DELETE FROM merchants` runs for 6 hours, and a GDPR deletion silently leaves 412 rows behind.

(a) Write the generic orphan-detection query that works across an arbitrary list of (child, fk_column, parent) triples.
(b) For each of the four relationships, decide how to clean the existing orphans. Two of them must **not** be bulk-deleted — identify which and explain why, referencing the relevant case-study principle.
(c) Give the complete zero-downtime procedure to add all four FKs to a live 4.1 TB database, in the correct order, with the lock level of each step.
(d) `VALIDATE CONSTRAINT` on a 3B-row table takes ~18 minutes. What lock does it hold, and what is and is not blocked during it?
(e) Choose `ON DELETE` for each of the four, justify against the business meaning, and identify the one where `CASCADE` would destroy financial history.
(f) Estimate the write-throughput cost and the storage cost, separating the FK-check cost from the index cost.
(g) One relationship crosses a shard boundary and cannot have an FK. Design the reconciliation job: what it checks, how often, what it does on a mismatch, and what alert it raises.
(h) Write the CI check that prevents a future migration from adding an FK without an index.

---

## Mental model checkpoint

1. What four things does a foreign key create in the database?
2. Which side does PostgreSQL index automatically, and which does it never index? What's the consequence of forgetting?
3. What lock does a child insert take on the parent row? What does it block, and what does it deliberately not block?
4. Name the five referential actions. Which is the correct default for historical data, and why?
5. Explain why an application-level check is not equivalent to an FK. Give the two-transaction timeline *and* name a second, non-race failure mode.
6. What does `MATCH SIMPLE` do with partial NULLs in a composite FK? What should you use instead?
7. You've decided to drop an FK for performance. What three things do you now owe?

---

## Quick reference card

| | |
|---|---|
| **Child INSERT/UPDATE** | `SELECT 1 FROM parent WHERE pk=$1 FOR KEY SHARE` — auto-indexed |
| **Parent DELETE/UPDATE** | `SELECT 1 FROM child WHERE fk=$1 FOR SHARE` — ★ **you must index** |
| Cost, FK check alone | ~+13% on child inserts |
| Cost, the required index | ~+27% on child inserts |
| Missing child index | up to **450×** slower parent deletes |

**The five actions**

| Action | Behaviour | Use for |
|---|---|---|
| `NO ACTION` | error; end-of-statement; deferrable | default |
| `RESTRICT` | error; immediate | **history, money, audit** |
| `CASCADE` | delete children | true weak entities only |
| `SET NULL` | null the child FK | optional relationships |
| `SET DEFAULT` | set to default | rarely useful |

**Adding an FK to a live table**

```sql
CREATE INDEX CONCURRENTLY idx_child_fk ON child (fk_col);   -- ① first!
ALTER TABLE child ADD CONSTRAINT fk_x FOREIGN KEY (fk_col)
  REFERENCES parent(id) ON DELETE RESTRICT NOT VALID;       -- ② brief lock
ALTER TABLE child VALIDATE CONSTRAINT fk_x;                 -- ③ weak lock
```

**The non-negotiables**
1. An index on every FK column.
2. `NOT NULL` when participation is mandatory.
3. `MATCH FULL` on composite FKs.
4. `RESTRICT`, not `CASCADE`, for anything historical.
5. No FK ⇒ a scheduled orphan-detection job. Always.

---

## When would I use this at work?

1. **The "foreign keys are slow" argument.** You can separate the FK-check cost (~13%) from the index cost (~27%) and point out that the index is required regardless — turning a philosophical debate into a measured 13% for structural correctness.

2. **A `DELETE` that runs for hours.** The unindexed-FK query finds it in ten seconds, and `CREATE INDEX CONCURRENTLY` fixes a 6-hour job into 1.2 seconds.

3. **Adding integrity to a legacy schema.** The `NOT VALID` → `VALIDATE` pattern lets you start enforcing on new writes *today* while cleaning history at your own pace — instead of the all-or-nothing migration that keeps getting deferred.

---

## Connected topics

**Understand before this:** 20–21 (relationships and their translation), 22 (the key an FK points at), 14 (why the FK index must be leftmost).

**This unlocks:**
- **24** — the rest of the constraint vocabulary
- **28** — zero-downtime migrations, where `NOT VALID` lives
- **45** — locks: `FOR KEY SHARE` and multixacts in full
- **59** — partitioning and FKs across partitions
- **60** — sharding, where FKs stop working and reconciliation begins
- **69** — GDPR/PII handling, which `RESTRICT` forces you to design properly
