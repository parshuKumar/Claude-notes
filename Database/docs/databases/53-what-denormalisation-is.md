# 53 — What Denormalisation Is
## Phase: Denormalisation & Scale

---

## ELI5 — The Simple Analogy

You run a restaurant. The menu prices live in one place — a laminated master sheet in the office. **One price, one location.** Change it there and it is changed everywhere, instantly and correctly. That is normalisation.

But every time a waiter takes an order, they walk to the office to check the price. Twelve waiters, three hundred orders a night, one office. **The office is the bottleneck** — not because looking up a price is hard, but because everybody has to go there.

So you print the price onto every waiter's notepad. Now nobody walks anywhere. Orders fly. That is denormalisation.

And now you own a **new obligation**: when a price changes, **you must find and update twelve notepads.** Miss one and a waiter charges last month's price. The master sheet and the notepad disagree, and nothing in the building tells you.

★ **That is the entire trade, and it never changes shape:**

> **You buy read speed with a permanent, ongoing obligation to keep copies in sync.**

The obligation is not a one-time cost. It is a tax you pay on **every write, forever, in every code path anyone ever adds.** The question is never "is denormalisation good?" It is "**is this specific read fast enough to be worth owning that obligation?**"

---

## Where this fits in the big picture

```
   Phase 4 (29–38): normalisation — ★ one fact, one place
   Phase 2 (10–19): indexes — make reads fast WITHOUT copying
   Phase 5 (39–52): transactions — how to keep things consistent
                          │
                          ▼
        ┌──────────────────────────────────────────────┐
        │ 53 WHAT DENORMALISATION IS ← YOU ARE HERE    │
        │ ★ the definition, the cost, the taxonomy     │
        └────────────────────┬─────────────────────────┘
                             ▼
              54 when to denormalise (★ the decision)
              55 the patterns · 56 materialised views
              57 caching · 61 counters and hot rows
```

Phase 4 spent ten topics proving that redundancy causes anomalies. **This phase is about deliberately reintroducing redundancy — with your eyes open, and with a mechanism to keep it honest.** The difference between a normalised schema you denormalised on purpose and a badly-designed one is entirely in whether you can name the obligation you took on.

---

## What is this?

**Denormalisation is deliberately storing a fact in more than one place, to make a read cheaper.**

Three things that are *not* denormalisation, and get called it constantly:

```
 ✗ AN INDEX IS NOT DENORMALISATION.
   An index is a redundant COPY of data — but ★ the database
   maintains it atomically, in the same transaction, with no
   possibility of drift. You take on NO obligation.
   ⇒ ★ ALWAYS EXHAUST INDEXES FIRST. An index that makes a query
     100× faster costs you nothing but write throughput and disk.
     A denormalised column costs you correctness risk forever.

 ✗ A CACHE IS NOT (QUITE) DENORMALISATION.
   A cache is a copy with a ★ TTL and an accepted staleness window,
   living outside the database. Denormalisation is a copy inside
   the database that is supposed to be CORRECT. (Topic 57.)

 ✗ A BAD SCHEMA IS NOT DENORMALISATION.
   Storing `customer_name` in `orders` because you never learned
   about foreign keys is a mistake. Storing it because you measured
   the join and it cost you 40 ms at p99 on your highest-traffic
   endpoint is a decision. ★ SAME COLUMN. DIFFERENT ENGINEERING.
   ⇒ THE DIFFERENCE IS WHETHER YOU CAN STATE THE OBLIGATION AND
     SHOW THE MECHANISM THAT DISCHARGES IT.
```

**The four kinds** (detailed in Topic 55):

| Kind | Example | Obligation |
|---|---|---|
| **Copied column** | `orders.customer_name` | update on every parent change |
| **Precomputed aggregate** | `products.review_count` | update on every child insert/delete |
| **Materialised path/hierarchy** | `categories.path` | rebuild the subtree on any move |
| **Repeating group / embedded** | `orders.items_json` | ★ lose the ability to query children |

---

## Why does it matter for a backend developer?

```
 ★ BECAUSE THE COST IS INVISIBLE AT THE MOMENT YOU PAY IT AND
   ENORMOUS LATER.

 THE DAY YOU ADD IT:
   • one column
   • one trigger, or three lines in one service
   • a query goes from 120 ms → 4 ms
   • ★ everyone is delighted

 EIGHTEEN MONTHS LATER:
   • four services write to that table
   • two of them were written by people who left
   • a bulk-import script updates the parent directly
   • an admin tool uses raw SQL
   • ★ 0.4% of rows disagree with the source of truth
   • ★ NOBODY KNOWS, because nothing checks
   • a customer notices before you do

 ⇒ ★ THE COST OF DENORMALISATION IS NOT THE WRITE AMPLIFICATION.
   IT IS THE ORGANISATIONAL OBLIGATION TO KEEP N CODE PATHS
   CORRECT FOREVER — AND ORGANISATIONS ARE BAD AT THAT.
 ⇒ WHICH IS WHY THE ENFORCEMENT MECHANISM MATTERS MORE THAN THE
   COLUMN: a trigger cannot be forgotten by a new service. Three
   lines in one service can.
```

---

## The physical reality

### What a join actually costs — measure before you assume

```
 ★ MOST "THE JOIN IS SLOW" CLAIMS ARE FALSE. Measure it.

 SELECT o.id, o.total_minor, c.name
   FROM orders o JOIN customers c ON c.id = o.customer_id
  WHERE o.id = 8842119;

 THE PLAN:
   Nested Loop
     -> Index Scan using orders_pkey on orders    (1 row)
          Buffers: shared hit=4
     -> Index Scan using customers_pkey on customers (1 row)
          Buffers: shared hit=3
   ★ TOTAL: 7 buffer hits. ~0.03 ms. The join costs THREE PAGE
     READS FROM MEMORY.

 ⇒ ★ A SINGLE-ROW JOIN ON AN INDEXED FOREIGN KEY IS ESSENTIALLY
   FREE. Denormalising to avoid it is pure loss.

 WHERE JOINS ACTUALLY COST:
 ① ★ N+1 — 200 rows, each triggering a separate round trip
    ⇒ ★ THE FIX IS ONE QUERY, NOT DENORMALISATION (Topic 66)
 ② ★ FAN-OUT — joining 5 tables where each multiplies rows
 ③ ★ THE JOIN CROSSES A SERVICE BOUNDARY — you literally cannot
    join. ⇒ this is the strongest genuine case.
 ④ ★ A LARGE SORT/AGGREGATE OVER THE JOIN RESULT
    ⇒ "top 20 products by review count" over 40M reviews
 ⑤ ★ THE JOINED TABLE IS HUGE AND COLD — random I/O per row
 ⑥ pagination/sorting by a column in the joined table
    ⇒ ★ forces a full join before LIMIT can apply
```

### The write amplification, physically

```
 ONE UPDATE TO A NORMALISED FACT:

   UPDATE customers SET name = 'Meera Sharma' WHERE id = 42;
   ⇒ 1 tuple rewritten (Topic 46)
   ⇒ 1–2 index entries if `name` is indexed
   ⇒ ~200 bytes of WAL
   ⇒ 0.3 ms

 THE SAME UPDATE, DENORMALISED INTO orders:

   UPDATE customers SET name = 'Meera Sharma' WHERE id = 42;
   UPDATE orders SET customer_name = 'Meera Sharma' WHERE customer_id = 42;
   ⇒ ★ customer 42 has 8,402 orders
   ⇒ 8,402 tuples rewritten — ★ EACH A FULL ROW COPY (Topic 46)
   ⇒ every index on `orders` updated 8,402 times ★ unless HOT applies
   ⇒ ~14 MB of WAL
   ⇒ 4,100 ms
   ⇒ ★ AND 8,402 rows are now LOCKED for the duration

 ⇒ ★ THE ASYMMETRY THAT DECIDES EVERYTHING:
   the read got 30× faster ONCE PER READ.
   the write got 13,000× slower ONCE PER WRITE.
   ⇒ ★ THE RATIO OF READS TO WRITES IS THE WHOLE CALCULATION.
     10,000 reads : 1 write ⇒ obviously worth it.
     10 reads : 1 write     ⇒ obviously not.
     ⇒ MEASURE IT. Don't guess.
```

### The three failure modes of a denormalised copy

```
 ★ ① DRIFT — the copy is wrong and nothing notices
    a code path updates the source and forgets the copy.
    ⇒ ★ SILENT. No error, no exception, no log line.
    ⇒ detected only by a reconciliation query nobody wrote.

 ★ ② HISTORICAL AMBIGUITY — is this drift, or a SNAPSHOT?
    `orders.customer_name` = 'Meera Singh' but customers.name is
    now 'Meera Sharma'.
    ⇒ IS THAT A BUG, OR IS IT CORRECT?
    ⇒ ★ IF THE ORDER SHOULD RECORD THE NAME AT THE TIME OF THE
      ORDER, IT IS NOT DENORMALISATION AT ALL. It is a legitimate
      SNAPSHOT, and it is 3NF-correct (Topic 38's "snapshot vs
      copy" test).
    ⇒ ★ AND THIS IS THE MOST IMPORTANT DISTINCTION IN THE WHOLE
      TOPIC, because it determines whether you need a sync
      mechanism at all.
      • a SNAPSHOT is frozen deliberately    ⇒ ★ NO obligation
      • a COPY must track the source          ⇒ ★ obligation forever
      ⇒ THE TEST: "if the source changes, must this change too?"
        ⇒ NO  ⇒ snapshot. Name the column so it's obvious:
                 `customer_name_at_order`, `unit_price_charged`
        ⇒ YES ⇒ a real denormalisation. Now you owe a mechanism.

 ★ ③ TRANSACTIONAL SKEW
    the source and copy are updated in DIFFERENT transactions
    ⇒ a window where a reader sees one and not the other
    ⇒ ★ FIX: same transaction, always. A trigger guarantees it;
      application code does not.
```

### Where the copy can be kept in sync — and what each guarantees

```
 ① ★ THE SAME TRANSACTION, IN APPLICATION CODE
    ✓ atomic
    ✗ ★ every code path must remember. Bulk imports, admin tools,
      psql, a new service — all bypass it.
    ⇒ ACCEPTABLE ONLY IF EXACTLY ONE SERVICE WRITES THE TABLE, AND
      YOU CAN PROVE IT.

 ② ★ A TRIGGER
    ✓ atomic, and ★ CANNOT BE BYPASSED — not by a bulk load, not
      by psql, not by a service written next year
    ✗ invisible in application code; surprises people debugging
    ✗ fires per row — a 1M-row bulk update runs it 1M times
    ⇒ ★ THE DEFAULT CHOICE FOR AN IN-DATABASE COPY.

 ③ ★ A GENERATED COLUMN — when the value is a pure function of
    the same row
    ALTER TABLE orders ADD COLUMN total_minor bigint
      GENERATED ALWAYS AS (qty * unit_price_minor) STORED;
    ✓ ★ CANNOT DRIFT. It is not a copy; it is a derivation.
    ✗ only same-row expressions; must be IMMUTABLE
    ⇒ ★ IF THIS FITS, IT IS STRICTLY THE BEST OPTION.

 ④ ASYNC — a job, CDC, or a materialised view
    ✓ no write-path cost
    ✗ ★ a staleness window you must bound and monitor
    ⇒ fine for aggregates and reports; ★ not for anything a user
      immediately reads back after writing.

 ⑤ ★ NOTHING — a snapshot
    ✓ zero obligation
    ⇒ ★ ONLY IF THE VALUE IS GENUINELY MEANT TO BE FROZEN. Name
      it accordingly.
```

---

## How it works — step by step

### The decision, made properly

```
 STEP 1 — MEASURE THE ACTUAL READ COST
   EXPLAIN (ANALYZE, BUFFERS) the real query, with real data
   volumes, at the real concurrency.
   ⇒ ★ if the join is 0.03 ms, stop. You have no problem.

 STEP 2 — TRY EVERY CHEAPER FIX FIRST
   ① a better index (composite, covering, partial)  ← Topics 14–17
   ② ★ fix the N+1 — one query instead of 200      ← Topic 66
   ③ a covering index enabling an index-only scan   ← Topic 12
   ④ raise work_mem so the hash join fits in memory
   ⑤ ★ a materialised view, if it's an aggregate    ← Topic 56
   ⇒ ★ EACH OF THESE COSTS YOU NOTHING IN CORRECTNESS.

 STEP 3 — MEASURE THE READ:WRITE RATIO
   SELECT calls FROM pg_stat_statements WHERE query LIKE '%...%';
   ⇒ ★ reads per write < 100 ⇒ think very hard
   ⇒ reads per write > 10,000 ⇒ probably justified

 STEP 4 — ASK THE SNAPSHOT QUESTION
   "if the source changes, must this change too?"
   ⇒ NO ⇒ ★ IT'S A SNAPSHOT. Not denormalisation. No obligation.
           Rename the column so nobody mistakes it.

 STEP 5 — CHOOSE THE ENFORCEMENT MECHANISM BEFORE THE COLUMN
   ★ if you cannot name the mechanism, you are not ready to add
     the column.
   generated column > trigger > same-transaction app code > async

 STEP 6 — ★ WRITE THE RECONCILIATION QUERY IN THE SAME PR
   the query that PROVES the copy matches the source.
   ⇒ ★ IF YOU CANNOT WRITE IT, YOU CANNOT DETECT DRIFT, AND YOU
     WILL HAVE DRIFT.
   ⇒ schedule it. Alert on non-zero.
```

### The reconciliation query — the non-negotiable artefact

```sql
-- ★ FOR EVERY DENORMALISED COLUMN, THIS QUERY MUST EXIST,
--   BE SCHEDULED, AND ALERT.

-- copied column
SELECT count(*) AS drifted
  FROM orders o JOIN customers c ON c.id = o.customer_id
 WHERE o.customer_name IS DISTINCT FROM c.name;   -- ★ IS DISTINCT FROM,
                                                  --   not <> (NULLs)

-- precomputed aggregate
SELECT count(*) AS drifted FROM (
  SELECT p.id, p.review_count AS stored,
         (SELECT count(*) FROM reviews r WHERE r.product_id = p.id) AS actual
    FROM products p) x
 WHERE stored IS DISTINCT FROM actual;

-- ★ and for large tables, sample instead of scanning:
SELECT count(*) FROM (
  SELECT p.id, p.review_count,
         (SELECT count(*) FROM reviews r WHERE r.product_id=p.id) AS actual
    FROM products TABLESAMPLE SYSTEM (1) p) x
 WHERE review_count IS DISTINCT FROM actual;
```

---

## Concept breakdown

```
DEFINITION
└── deliberately storing one fact in more than one place, to make
     a read cheaper
     ★ THE PRICE: a permanent obligation to keep copies in sync,
       paid on EVERY write, by EVERY code path, FOREVER

WHAT IT IS NOT
├── ★ an INDEX — redundant but maintained ATOMICALLY by the engine
│    ⇒ ★ ZERO obligation ⇒ ALWAYS EXHAUST INDEXES FIRST
├── a CACHE — outside the DB, with an accepted TTL (Topic 57)
└── a BAD SCHEMA — same column, but no measurement and no mechanism

★ THE SNAPSHOT vs COPY TEST — ask this FIRST
   "if the source changes, must this change too?"
   NO  ⇒ ★ a SNAPSHOT. 3NF-correct. No obligation. Name it
          `customer_name_at_order`, `unit_price_charged`.
   YES ⇒ a real denormalisation. You now owe a mechanism.
   ⇒ ★ MOST "denormalisation" IN E-COMMERCE IS ACTUALLY SNAPSHOTS,
     and needs no sync at all.

THE FOUR KINDS (Topic 55)
├── copied column          orders.customer_name
├── precomputed aggregate  products.review_count
├── materialised path      categories.path
└── embedded/repeating     orders.items_json  ★ loses queryability

THE ARITHMETIC THAT DECIDES
   read cost saved × read frequency
   ★ vs  write cost added × write frequency × fan-out
   ⇒ one UPDATE of a customer name → 8,402 order rows rewritten
   ⇒ ★ MEASURE THE READ:WRITE RATIO. <100 ⇒ almost never worth it.

WHERE JOINS ACTUALLY COST (the genuine cases)
├── ★ N+1 → ★ FIX THE N+1, don't denormalise (Topic 66)
├── large aggregates over big tables → ★ materialised view first
├── ★ the join crosses a SERVICE BOUNDARY → the strongest case
├── sorting/paginating by a joined column
└── fan-out across 5+ tables

THE THREE FAILURE MODES
├── ★ DRIFT — silent, undetectable without a reconciler
├── ★ HISTORICAL AMBIGUITY — is it stale, or is it a snapshot?
└── TRANSACTIONAL SKEW — source and copy in different transactions

ENFORCEMENT, BEST FIRST
├── ★ GENERATED COLUMN — cannot drift; a derivation, not a copy
├── ★ TRIGGER — atomic, ★ unbypassable by any future code path
├── same-transaction app code — ★ only if ONE service writes
└── async (job/CDC/matview) — ★ a staleness window to bound & monitor

★ THE NON-NEGOTIABLE ARTEFACT
   A RECONCILIATION QUERY, IN THE SAME PULL REQUEST, SCHEDULED,
   ALERTING.
   ⇒ if you cannot write it, you cannot detect drift, and you will
     have drift.
```

---

## Diagrams

**Diagram 1 — big picture: the trade, both directions**

```
              NORMALISED                    DENORMALISED
 ┌──────────────────────────┐      ┌──────────────────────────────┐
 │ customers                │      │ customers                    │
 │  id │ name               │      │  id │ name                   │
 │  42 │ Meera Sharma       │      │  42 │ Meera Sharma           │
 │                          │      │                              │
 │ orders                   │      │ orders                       │
 │  id  │ customer_id       │      │  id  │ customer_id │ customer│
 │ 8801 │ 42                │      │ 8801 │ 42          │ _name   │
 │ 8802 │ 42                │      │      │             │ Meera S │
 │  …   │ …  (8,402 rows)   │      │ 8802 │ 42          │ Meera S │
 └──────────────────────────┘      │  …   │ …           │ … ×8402 │
                                   └──────────────────────────────┘
 READ  "orders with names"          READ
   JOIN on indexed FK                 no join
   ★ 7 buffers, 0.03 ms               ★ 4 buffers, 0.02 ms
   at 200 rows: 1.4 ms                at 200 rows: ★ 0.4 ms  (3.5×)

 WRITE "rename customer 42"          WRITE
   1 tuple, 200 B WAL, 0.3 ms          ★ 8,403 tuples
                                       ★ 14 MB WAL
                                       ★ 4,100 ms
                                       ★ 8,402 rows locked
                                       ★ 13,000× slower

 ⇒ ★ THE QUESTION IS NEVER "WHICH IS FASTER". IT IS:
   how many reads per write, and what is the p99 you actually need?
```

**Diagram 2 — data flow: how drift happens**

```
  MONTH 0 — one service, everything correct
  ┌──────────────────────────────────────────────────────────────┐
  │  order-service                                                │
  │    BEGIN;                                                     │
  │      UPDATE customers SET name=$1 WHERE id=$2;                │
  │      UPDATE orders SET customer_name=$1 WHERE customer_id=$2; │
  │    COMMIT;                              ★ correct             │
  └──────────────────────────────────────────────────────────────┘

  MONTH 7 — a second writer appears
  ┌──────────────────────────────────────────────────────────────┐
  │  admin-service (new team, didn't know)                        │
  │    UPDATE customers SET name=$1 WHERE id=$2;                  │
  │                                         ★ ← DRIFT STARTS HERE │
  │                                         ★ no error, no log    │
  └──────────────────────────────────────────────────────────────┘

  MONTH 11 — a bulk import
  ┌──────────────────────────────────────────────────────────────┐
  │  COPY customers FROM 'crm_export.csv';                        │
  │  UPDATE customers SET name = … FROM staging …;                │
  │                                         ★ 41,000 rows drift   │
  └──────────────────────────────────────────────────────────────┘

  MONTH 14 — someone fixes a typo in psql at 2am
  ┌──────────────────────────────────────────────────────────────┐
  │  UPDATE customers SET name='Meera Sharma' WHERE id=42;        │
  └──────────────────────────────────────────────────────────────┘

  MONTH 18 — a customer emails: "your invoice has my old name"
  ┌──────────────────────────────────────────────────────────────┐
  │  SELECT count(*) FROM orders o JOIN customers c              │
  │    ON c.id=o.customer_id                                      │
  │   WHERE o.customer_name IS DISTINCT FROM c.name;              │
  │  ⇒ ★ 184,220 rows                                             │
  │  ⇒ ★ AND NOBODY CAN SAY WHICH ARE BUGS AND WHICH ARE          │
  │    LEGITIMATE HISTORICAL SNAPSHOTS.                           │
  └──────────────────────────────────────────────────────────────┘
        ★ THE TRIGGER WOULD HAVE SURVIVED ALL FOUR.
        ★ THE RECONCILER WOULD HAVE CAUGHT IT IN MONTH 7.
```

**Diagram 3 — before/after: the right fix was not denormalisation**

```
 THE COMPLAINT: "GET /orders is 340 ms at p99. The join is slow."

 ✗ THE PROPOSED FIX — denormalise
 ┌───────────────────────────────────────────────────────────────┐
 │ ALTER TABLE orders ADD COLUMN customer_name text;             │
 │ ALTER TABLE orders ADD COLUMN customer_email text;            │
 │ ALTER TABLE orders ADD COLUMN customer_tier text;             │
 │ + a trigger, + backfill, + a reconciler                       │
 │                                                                │
 │ ★ RESULT: 340 ms → 310 ms.    (9% better)                     │
 │ ★ AND: 3 columns of permanent obligation                      │
 └───────────────────────────────────────────────────────────────┘

 ✓ WHAT EXPLAIN ACTUALLY SAID
 ┌───────────────────────────────────────────────────────────────┐
 │ Limit  (actual time=338.2..338.4 rows=50)                     │
 │   ->  Sort  (actual time=338.2..338.3 rows=50 loops=1)        │
 │         Sort Key: o.created_at DESC                           │
 │         ★ Sort Method: external merge  Disk: 84,112kB         │
 │         ->  Hash Join  (actual time=2.1..291.4 rows=412,880)  │
 │               ->  Seq Scan on orders  (rows=412,880)          │
 │                     ★ Filter: (customer_id = 42)              │
 │                     ★ Rows Removed by Filter: 39,587,120      │
 │               ->  Hash  (rows=1)                              │
 │                                                                │
 │ ★ THE JOIN COST 2.1 ms. THE SEQ SCAN AND THE DISK SORT COST   │
 │   336 ms.                                                     │
 └───────────────────────────────────────────────────────────────┘

 ✓ THE ACTUAL FIX — one index
 ┌───────────────────────────────────────────────────────────────┐
 │ CREATE INDEX CONCURRENTLY idx_orders_cust_created             │
 │   ON orders (customer_id, created_at DESC);                   │
 │                                                                │
 │ Limit  (actual time=0.08..0.34 rows=50)                       │
 │   ->  Nested Loop  (actual time=0.08..0.31 rows=50)           │
 │         ->  ★ Index Scan using idx_orders_cust_created        │
 │               (actual time=0.04..0.09 rows=50)                │
 │         ->  Index Scan using customers_pkey (loops=50)        │
 │                                                                │
 │ ★ 340 ms → 0.4 ms.   850×.   ZERO obligation.                 │
 └───────────────────────────────────────────────────────────────┘
      ★ THE JOIN WAS NEVER THE PROBLEM. It almost never is.
```

---

## Example 1 — basic

```sql
CREATE TABLE customers (
  id bigint PRIMARY KEY,
  name text NOT NULL,
  email text NOT NULL,
  tier text NOT NULL DEFAULT 'standard'
);
CREATE TABLE orders (
  id bigserial PRIMARY KEY,
  customer_id bigint NOT NULL REFERENCES customers(id),
  total_minor bigint NOT NULL,
  created_at timestamptz NOT NULL DEFAULT now()
);
INSERT INTO customers SELECT g, 'Customer '||g, 'c'||g||'@example.com', 'standard'
  FROM generate_series(1,100000) g;
INSERT INTO orders (customer_id, total_minor, created_at)
SELECT (random()*99999+1)::bigint, (random()*500000)::bigint,
       now() - (random()*365)::int * interval '1 day'
  FROM generate_series(1,4000000);
CREATE INDEX idx_orders_customer ON orders (customer_id);
VACUUM ANALYZE customers, orders;
```

**Measure the join before assuming anything.**
```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT o.id, o.total_minor, c.name
  FROM orders o JOIN customers c ON c.id = o.customer_id
 WHERE o.id = 2000000;
```
```
 Nested Loop  (actual time=0.031..0.033 rows=1 loops=1)
   Buffers: shared hit=8
   ->  Index Scan using orders_pkey on orders  (actual time=0.019..0.020 rows=1)
         Buffers: shared hit=4
   ->  Index Scan using customers_pkey on customers  (actual time=0.008..0.008 rows=1)
         Buffers: shared hit=4
 Execution Time: ★ 0.058 ms
```
```
 ★ THE JOIN ADDED 4 BUFFER HITS AND ~0.01 ms.
   Denormalising to avoid this would be pure loss.
```

**Now the case where it does cost — 200 rows.**
```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT o.id, o.total_minor, c.name
  FROM orders o JOIN customers c ON c.id = o.customer_id
 ORDER BY o.created_at DESC LIMIT 200;
```
```
 Limit  (actual time=1284.2..1284.4 rows=200)
   ->  Sort  (actual time=1284.2..1284.3 rows=200)
         ★ Sort Method: external merge  Disk: 412,880kB
         ->  Hash Join  (actual time=88.1..1041.2 rows=4,000,000)
               ->  Seq Scan on orders  (rows=4,000,000)
               ->  Hash  (rows=100,000)
 Execution Time: ★ 1,284.8 ms
```

**The index fix — no denormalisation needed.**
```sql
CREATE INDEX CONCURRENTLY idx_orders_created ON orders (created_at DESC);
EXPLAIN (ANALYZE, BUFFERS)
SELECT o.id, o.total_minor, c.name
  FROM orders o JOIN customers c ON c.id = o.customer_id
 ORDER BY o.created_at DESC LIMIT 200;
```
```
 Limit  (actual time=0.062..1.104 rows=200)
   ->  Nested Loop  (actual time=0.060..1.081 rows=200)
         ->  ★ Index Scan using idx_orders_created on orders (rows=200)
         ->  Index Scan using customers_pkey on customers (loops=200)
 Execution Time: ★ 1.182 ms
```
```
 ★ 1,284 ms → 1.18 ms. 1,087×. One index. Zero obligation.
```

**Now denormalise, and measure both sides.**
```sql
ALTER TABLE orders ADD COLUMN customer_name text;
UPDATE orders o SET customer_name = c.name FROM customers c WHERE c.id = o.customer_id;
VACUUM ANALYZE orders;

EXPLAIN (ANALYZE, BUFFERS)
SELECT id, total_minor, customer_name
  FROM orders ORDER BY created_at DESC LIMIT 200;
```
```
 Limit  (actual time=0.041..0.612 rows=200)
   ->  Index Scan using idx_orders_created on orders (rows=200)
 Execution Time: ★ 0.674 ms      — 1.18 → 0.67 ms.  1.8×.
```

**And the write cost.**
```sql
SELECT count(*) FROM orders WHERE customer_id = 42;
```
```
 count
-------
    41
```
```sql
\timing on
UPDATE customers SET name = 'Meera Sharma' WHERE id = 42;
-- Time: 0.412 ms
UPDATE orders SET customer_name = 'Meera Sharma' WHERE customer_id = 42;
-- Time: ★ 8.204 ms      — 20× for 41 rows.

-- and for a customer with many orders
INSERT INTO orders (customer_id, total_minor)
SELECT 43, 1000 FROM generate_series(1,50000);
UPDATE orders SET customer_name = 'Big Customer' WHERE customer_id = 43;
-- Time: ★ 1,142.882 ms
```

**Measure the WAL both ways.**
```sql
SELECT pg_current_wal_lsn() AS a \gset
UPDATE customers SET name = 'X' WHERE id = 43;
SELECT pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), :'a')) AS normalised_wal;

SELECT pg_current_wal_lsn() AS b \gset
UPDATE orders SET customer_name = 'X' WHERE customer_id = 43;
SELECT pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), :'b')) AS denorm_wal;
```
```
 normalised_wal
----------------
 232 bytes
 denorm_wal
------------
 ★ 12 MB          — 54,000× more WAL for the same logical change
```

**Enforce it with a trigger — the mechanism that can't be bypassed.**
```sql
CREATE OR REPLACE FUNCTION sync_order_customer_name() RETURNS trigger AS $$
BEGIN
  IF NEW.name IS DISTINCT FROM OLD.name THEN
    UPDATE orders SET customer_name = NEW.name WHERE customer_id = NEW.id;
  END IF;
  RETURN NEW;
END $$ LANGUAGE plpgsql;

CREATE TRIGGER trg_sync_customer_name
  AFTER UPDATE OF name ON customers        -- ★ OF name: only fires when
  FOR EACH ROW EXECUTE FUNCTION sync_order_customer_name();
                                            --   that column changes

-- and for new orders
CREATE OR REPLACE FUNCTION fill_order_customer_name() RETURNS trigger AS $$
BEGIN
  IF NEW.customer_name IS NULL THEN
    SELECT name INTO NEW.customer_name FROM customers WHERE id = NEW.customer_id;
  END IF;
  RETURN NEW;
END $$ LANGUAGE plpgsql;

CREATE TRIGGER trg_fill_customer_name
  BEFORE INSERT ON orders
  FOR EACH ROW EXECUTE FUNCTION fill_order_customer_name();
```

**Prove the trigger survives what application code wouldn't.**
```sql
-- a "rogue" update, exactly like a bulk import or an admin tool
UPDATE customers SET name = 'Renamed Directly' WHERE id = 42;
SELECT DISTINCT customer_name FROM orders WHERE customer_id = 42;
```
```
   customer_name
-------------------
 Renamed Directly       ★ the trigger caught it. No app code involved.
```

**★ The reconciliation query — write it, schedule it, alert on it.**
```sql
SELECT count(*) AS drifted
  FROM orders o JOIN customers c ON c.id = o.customer_id
 WHERE o.customer_name IS DISTINCT FROM c.name;
```
```
 drifted
---------
       0        ★ alert if > 0
```
```sql
-- simulate drift by disabling the trigger, as a bulk load might
ALTER TABLE customers DISABLE TRIGGER trg_sync_customer_name;
UPDATE customers SET name = 'Drifted' WHERE id BETWEEN 100 AND 200;
ALTER TABLE customers ENABLE TRIGGER trg_sync_customer_name;

SELECT count(*) AS drifted
  FROM orders o JOIN customers c ON c.id = o.customer_id
 WHERE o.customer_name IS DISTINCT FROM c.name;
```
```
 drifted
---------
   ★ 4,082        ⇒ the alert fires. This is the whole point.
```
```sql
-- repair
UPDATE orders o SET customer_name = c.name FROM customers c
 WHERE c.id = o.customer_id AND o.customer_name IS DISTINCT FROM c.name;
```

**★ The generated column — when it fits, it cannot drift at all.**
```sql
CREATE TABLE order_items (
  id bigserial PRIMARY KEY,
  order_id bigint NOT NULL,
  qty int NOT NULL CHECK (qty > 0),
  unit_price_minor bigint NOT NULL,
  line_total_minor bigint
    GENERATED ALWAYS AS (qty * unit_price_minor) STORED   -- ★ a derivation
);
INSERT INTO order_items (order_id, qty, unit_price_minor) VALUES (1, 3, 49900);
SELECT * FROM order_items;
```
```
 id | order_id | qty | unit_price_minor | line_total_minor
----+----------+-----+------------------+------------------
  1 |        1 |   3 |            49900 |         ★ 149700
```
```sql
UPDATE order_items SET line_total_minor = 1 WHERE id = 1;
```
```
ERROR:  column "line_total_minor" can only be updated to DEFAULT
   ★ IT IS STRUCTURALLY IMPOSSIBLE TO MAKE IT WRONG.
```

---

## Example 2 — production scenario

**The situation.** A marketplace. The product listing page shows, for each product: name, price, seller name, seller rating, review count and average rating. It is the highest-traffic endpoint in the system.

```
 GET /api/products?category=…&page=1
   p50   180 ms
   p99   ★ 2,400 ms
   rps   4,200
   ★ 38% of all database time in the cluster
```

The team's proposal: *"denormalise everything onto `products` — seller name, seller rating, review count, avg rating. One table, one index scan, done."*

**Step 1 — look at what the query actually does.**

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT p.id, p.name, p.price_minor,
       s.name AS seller_name, s.rating AS seller_rating,
       count(r.id) AS review_count,
       avg(r.stars)::numeric(2,1) AS avg_rating
  FROM products p
  JOIN sellers s ON s.id = p.seller_id
  LEFT JOIN reviews r ON r.product_id = p.id
 WHERE p.category_id = 88 AND p.status = 'active'
 GROUP BY p.id, p.name, p.price_minor, s.name, s.rating
 ORDER BY p.created_at DESC
 LIMIT 24;
```
```
 Limit  (actual time=2388.1..2388.4 rows=24 loops=1)
   Buffers: shared hit=41,204 read=182,004
   ->  Sort  (actual time=2388.1..2388.2 rows=24)
         Sort Key: p.created_at DESC
         ★ Sort Method: external merge  Disk: 18,442kB
         ->  GroupAggregate  (actual time=88.2..2311.4 rows=41,204)
               ->  Nested Loop Left Join  (actual rows=8,842,119)
                     ->  Nested Loop  (actual rows=41,204)
                           ->  ★ Seq Scan on products
                                 Filter: (category_id=88 AND status='active')
                                 ★ Rows Removed by Filter: 2,158,796
                           ->  Index Scan using sellers_pkey (loops=41,204)
                     ->  ★ Index Scan using idx_reviews_product
                           (loops=41,204, actual rows=214)
 Planning Time: 0.412 ms
 Execution Time: ★ 2,388.9 ms
```

**Step 2 — attribute the cost. It is three separate problems.**

```
 ★ BREAK IT DOWN — do NOT treat "the query is slow" as one problem:

 ① Seq Scan on products, 2.15M rows removed by filter
    ⇒ ★ A MISSING INDEX. Nothing to do with joins.  ~180 ms

 ② the sellers join: Index Scan, 41,204 loops
    ⇒ ★ 41,204 × ~0.002 ms ≈ 82 ms
    ⇒ and this only happens because we're processing 41,204
      products instead of 24.

 ③ ★ THE REVIEWS AGGREGATE: 8.8 MILLION review rows scanned to
    produce 24 rows of output.
    ⇒ ★ ~2,100 ms. 88% OF THE TOTAL.

 ④ the external-merge sort: 18 MB to disk    ~90 ms

 ⇒ ★ THE SELLER JOIN — THE THING THEY WANTED TO DENORMALISE —
   IS 3% OF THE COST.
   THE AGGREGATE IS 88%.
```

**Step 3 — fix ① and ④ with an index. Free.**

```sql
CREATE INDEX CONCURRENTLY idx_products_cat_created
  ON products (category_id, created_at DESC)
  WHERE status = 'active';
```
```
 Limit  (actual time=0.412..1,984.2 rows=24)
   ->  Nested Loop Left Join
         ->  Nested Loop
               ->  ★ Index Scan using idx_products_cat_created (rows=24)
               ->  Index Scan using sellers_pkey (loops=24)
         ->  Index Scan using idx_reviews_product (loops=24, rows=214)
   ->  GroupAggregate
 Execution Time: ★ 1,984.6 ms
```
```
 ★ 2,388 → 1,984 ms. The seq scan and the sort are gone.
   ★ AND NOTE: the seller join now runs 24 times instead of 41,204.
     It costs 0.05 ms. Denormalising it would save NOTHING.
```

**Step 4 — the aggregate is the real problem. Denormalise *that*.**

```
 ★ THE SNAPSHOT-vs-COPY TEST:
   "if a new review is added, must review_count change?"
   ⇒ ★ YES. This is a genuine denormalisation. We owe a mechanism.

 ★ CHEAPER OPTIONS FIRST:
   • an index on reviews(product_id, stars)?
     ⇒ still 214 rows aggregated per product. Helps, doesn't fix.
   • a materialised view? (Topic 56)
     ⇒ ★ REFRESH cost on 8.8M reviews, and a staleness window.
       Review counts are expected to update immediately when a
       user posts. ⇒ NO.
   • ★ maintained aggregate columns, updated by trigger
     ⇒ THIS IS THE RIGHT ANSWER. The read is O(1) and the write
       cost is bounded: ONE review insert touches ONE product row.
```

```sql
ALTER TABLE products
  ADD COLUMN review_count   integer NOT NULL DEFAULT 0,
  ADD COLUMN review_sum     bigint  NOT NULL DEFAULT 0,
  ADD COLUMN avg_rating     numeric(2,1)
    GENERATED ALWAYS AS (
      CASE WHEN review_count = 0 THEN NULL
           ELSE round(review_sum::numeric / review_count, 1) END
    ) STORED;
-- ★ STORE SUM AND COUNT, DERIVE THE AVERAGE.
--   Storing avg directly would make it impossible to update
--   incrementally on a DELETE, and averages of averages are wrong.
--   ⇒ the average is a GENERATED column ⇒ it cannot drift.
```

```sql
CREATE OR REPLACE FUNCTION sync_product_review_stats() RETURNS trigger AS $$
BEGIN
  IF TG_OP = 'INSERT' THEN
    UPDATE products
       SET review_count = review_count + 1,
           review_sum   = review_sum   + NEW.stars
     WHERE id = NEW.product_id;
  ELSIF TG_OP = 'DELETE' THEN
    UPDATE products
       SET review_count = review_count - 1,
           review_sum   = review_sum   - OLD.stars
     WHERE id = OLD.product_id;
  ELSIF TG_OP = 'UPDATE' THEN
    IF NEW.product_id IS DISTINCT FROM OLD.product_id THEN
      UPDATE products SET review_count=review_count-1, review_sum=review_sum-OLD.stars
       WHERE id = OLD.product_id;
      UPDATE products SET review_count=review_count+1, review_sum=review_sum+NEW.stars
       WHERE id = NEW.product_id;
    ELSIF NEW.stars IS DISTINCT FROM OLD.stars THEN
      UPDATE products SET review_sum = review_sum - OLD.stars + NEW.stars
       WHERE id = NEW.product_id;
    END IF;
  END IF;
  RETURN NULL;
END $$ LANGUAGE plpgsql;

CREATE TRIGGER trg_review_stats
  AFTER INSERT OR UPDATE OR DELETE ON reviews
  FOR EACH ROW EXECUTE FUNCTION sync_product_review_stats();
```

```sql
-- backfill
UPDATE products p SET
  review_count = coalesce(x.n, 0),
  review_sum   = coalesce(x.s, 0)
FROM (SELECT product_id, count(*) AS n, sum(stars) AS s
        FROM reviews GROUP BY product_id) x
WHERE x.product_id = p.id;

-- ★ and make the invariant structural
ALTER TABLE products
  ADD CONSTRAINT ck_review_count_nonneg CHECK (review_count >= 0),
  ADD CONSTRAINT ck_review_sum_valid
    CHECK (review_sum >= 0 AND review_sum <= review_count * 5);
```

**Step 5 — the new query.**

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT p.id, p.name, p.price_minor,
       s.name AS seller_name, s.rating AS seller_rating,
       p.review_count, p.avg_rating
  FROM products p JOIN sellers s ON s.id = p.seller_id
 WHERE p.category_id = 88 AND p.status = 'active'
 ORDER BY p.created_at DESC LIMIT 24;
```
```
 Limit  (actual time=0.048..0.284 rows=24 loops=1)
   Buffers: ★ shared hit=104
   ->  Nested Loop  (actual time=0.046..0.271 rows=24)
         ->  Index Scan using idx_products_cat_created on products (rows=24)
         ->  Index Scan using sellers_pkey on sellers (loops=24)
 Execution Time: ★ 0.318 ms
```
```
 ★ 2,388 ms → 0.318 ms.  7,509×.
 ★ AND THE SELLER JOIN IS STILL THERE. It costs 0.03 ms.
   The team's original proposal would have added two more
   denormalised columns and two more obligations to save that.
```

**Step 6 — the write cost, measured honestly.**

```sql
\timing on
INSERT INTO reviews (product_id, user_id, stars, body)
VALUES (88412, 41209, 5, 'Great');
-- Time: 0.884 ms   (was 0.412 ms — ★ 2.1×, one extra row update)

-- and the contention question:
SELECT product_id, count(*) AS reviews_per_day
  FROM reviews WHERE created_at > now() - interval '1 day'
 GROUP BY 1 ORDER BY 2 DESC LIMIT 3;
```
```
 product_id | reviews_per_day
------------+-----------------
      88412 |            ★ 41
      12004 |              28
```
```
 ★ 41 reviews/day on the busiest product = 0.0005 writes/sec.
   The trigger's row lock on products(88412) is held for <1 ms.
   ⇒ ★ NO HOT-ROW PROBLEM. (Contrast case study 01, where a
     counter takes 4,000 writes/sec and this design would collapse
     — Topic 61.)
 ⇒ ★ ALWAYS CHECK THE WRITE DISTRIBUTION BEFORE ADDING A
   TRIGGER-MAINTAINED COUNTER.
```

**Step 7 — the reconciler, in the same pull request.**

```sql
-- scheduled hourly; alerts on non-zero
WITH actual AS (
  SELECT product_id, count(*) AS n, sum(stars) AS s
    FROM reviews GROUP BY product_id
)
SELECT count(*) AS drifted
  FROM products p LEFT JOIN actual a ON a.product_id = p.id
 WHERE p.review_count IS DISTINCT FROM coalesce(a.n, 0)
    OR p.review_sum   IS DISTINCT FROM coalesce(a.s, 0);
```
```
 drifted
---------
       0
```
```sql
-- ★ on a 40M-row reviews table this is expensive. Sample instead,
--   and run the full version nightly.
SELECT count(*) FROM (
  SELECT p.id, p.review_count,
         (SELECT count(*) FROM reviews r WHERE r.product_id = p.id) AS actual
    FROM products TABLESAMPLE SYSTEM (1) p) x
 WHERE review_count IS DISTINCT FROM actual;

-- and the self-repair
UPDATE products p SET review_count = coalesce(a.n,0), review_sum = coalesce(a.s,0)
  FROM (SELECT product_id, count(*) n, sum(stars) s FROM reviews GROUP BY 1) a
 WHERE a.product_id = p.id
   AND (p.review_count IS DISTINCT FROM a.n OR p.review_sum IS DISTINCT FROM a.s);
```

**Step 8 — results.**

| | Before | After |
|---|---|---|
| p50 | 180 ms | 3 ms |
| p99 | 2,400 ms | **11 ms** (**218×**) |
| Buffers per request | 223,208 | **104** |
| Share of cluster DB time | 38% | **1.2%** |
| Review insert | 0.412 ms | 0.884 ms (2.1×) |
| Denormalised columns added | *(proposed: 4)* | **2** (+1 generated) |
| Reconcilers owed | *(proposed: 4)* | **1** |

```
 ★ THE THREE LESSONS:
 ① ★ "THE QUERY IS SLOW" IS NEVER ONE PROBLEM. EXPLAIN attributed
   88% to the aggregate, 8% to a missing index, 3% to the join
   they wanted to remove.
 ② ★ THE FIRST FIX WAS AN INDEX (free). The second was
   denormalisation (an obligation). ★ In that order, always.
 ③ ★ THEY DENORMALISED 2 COLUMNS INSTEAD OF 4. The seller name and
   rating join costs 0.03 ms and always will, because the index
   reduced the loop count from 41,204 to 24.
   ⇒ ★ FIXING THE ROW COUNT OFTEN MAKES THE JOIN FREE, AND THEN
     THERE IS NOTHING TO DENORMALISE.
```

---

## Common mistakes

**1. Denormalising before measuring the join.**
- *Symptom:* an obligation taken on for a 9% improvement.
- *Fix:* `EXPLAIN (ANALYZE, BUFFERS)` first. A single-row join on an indexed FK is ~0.01 ms.

**2. Denormalising to fix an N+1.**
- *Symptom:* 200 round trips become 200 faster round trips.
- *Fix:* one query. The problem is the round trips, not the join (Topic 66).

**3. Confusing a snapshot with a copy.**
- *Symptom:* a "drift" report full of rows that are actually correct history — or a genuine copy that nobody syncs because someone called it a snapshot.
- *Fix:* ask "if the source changes, must this change too?" and *name the column accordingly*: `unit_price_charged` vs `product_price`.

**4. Syncing in application code when more than one service writes.**
- *Symptom:* drift starting the month a second team touches the table.
- *Fix:* a trigger. It cannot be bypassed by a bulk load, an admin tool, or a service written next year.

**5. Shipping without a reconciliation query.**
- *Symptom:* drift discovered by a customer 18 months later, with no way to tell bugs from history.
- *Fix:* the reconciler goes in the same PR, scheduled, alerting.

**6. Storing an average instead of sum and count.**
- *Symptom:* impossible to update incrementally on delete; averages of averages.
- *Fix:* store `sum` and `count`; make the average a `GENERATED` column.

**7. Not checking the write distribution before a trigger-maintained counter.**
- *Symptom:* correctness fine, throughput destroyed — every writer serialises on one row.
- *Fix:* measure writes/sec on the hottest key. Above a few hundred, shard the counter (Topic 61).

**8. Using a trigger where a generated column would do.**
- *Symptom:* an obligation and a reconciler for something that is a pure function of the same row.
- *Fix:* `GENERATED ALWAYS AS (…) STORED`. It cannot drift.

**9. Denormalising and keeping the join.**
- *Symptom:* the column exists, but half the queries still join.
- *Fix:* if you took on the obligation, use it everywhere — or drop it.

**10. Forgetting `DELETE` and `UPDATE` in the trigger.**
- *Symptom:* counters that only ever go up.
- *Fix:* handle all three operations, including a changed foreign key.

---

## Hands-on proof

**PROVE IT #1–#9 — Example 1** (the join costing 4 buffers, the index beating denormalisation 1,087×, the read gain from denormalising measured honestly at 1.8×, the write cost at 20× and 1,142 ms, the WAL at 54,000×, a trigger surviving a rogue update, drift created and detected, and a generated column that cannot be made wrong).

**PROVE IT #10 — the read:write ratio, from real data.**
```sql
SELECT
  sum(calls) FILTER (WHERE query ILIKE '%FROM orders%' AND query ILIKE '%SELECT%')
    AS reads,
  sum(calls) FILTER (WHERE query ILIKE '%UPDATE customers%')
    AS writes
FROM pg_stat_statements;
```
```
  reads   | writes
----------+--------
 41882119 |   4120
   ★ 10,166 reads per write. This ratio justifies denormalisation.
     A ratio of 40 would not.
```

**PROVE IT #11 — trigger overhead on a bulk operation.**
```sql
\timing on
ALTER TABLE reviews DISABLE TRIGGER trg_review_stats;
INSERT INTO reviews (product_id, user_id, stars)
  SELECT (random()*100000)::bigint, 1, 5 FROM generate_series(1,100000);
-- Time: 1,204 ms
ALTER TABLE reviews ENABLE TRIGGER trg_review_stats;

INSERT INTO reviews (product_id, user_id, stars)
  SELECT (random()*100000)::bigint, 1, 5 FROM generate_series(1,100000);
-- Time: ★ 18,882 ms      — 15.7× for a bulk load
```
```
 ★ A ROW-LEVEL TRIGGER FIRES ONCE PER ROW. For bulk loads:
   disable → load → recompute → enable, inside one transaction.
```

**PROVE IT #12 — the generated column refuses to be wrong.**
```sql
UPDATE order_items SET line_total_minor = 1 WHERE id = 1;
-- ERROR: column "line_total_minor" can only be updated to DEFAULT
```

---

## The design decision framework

```
★★★ DENORMALISATION IS A LOAN. THE INTEREST IS PAID FOREVER,
    BY EVERY FUTURE ENGINEER. ★★★

 ① MEASURE THE READ FIRST
    EXPLAIN (ANALYZE, BUFFERS) with real volumes and real
    concurrency.
    ⇒ ★ ATTRIBUTE THE COST. "The query is slow" is never one
      problem. Find out what percentage the join actually is.
    ⇒ a single-row join on an indexed FK is ~0.01 ms. Stop.

 ② EXHAUST THE FREE FIXES — ★ ALL OF THEM
    ① a composite/partial/covering index      (Topics 14–17)
    ② ★ fix the N+1 — one query, not 200      (Topic 66)
    ③ an index-only scan                       (Topic 12)
    ④ raise work_mem so the sort/hash fits
    ⑤ a materialised view for aggregates       (Topic 56)
    ⇒ ★ THESE COST NOTHING IN CORRECTNESS.
    ⇒ ★ AND OFTEN THE INDEX MAKES THE JOIN FREE by reducing the
      row count from 41,204 to 24.

 ③ ASK THE SNAPSHOT QUESTION
    "if the source changes, must this change too?"
    NO  ⇒ ★ IT'S A SNAPSHOT. Not denormalisation. No obligation.
          ★ NAME IT SO: unit_price_charged, customer_name_at_order
    YES ⇒ continue.

 ④ MEASURE THE READ:WRITE RATIO AND THE FAN-OUT
    reads per write < 100        ⇒ ★ almost never worth it
    reads per write > 10,000     ⇒ probably justified
    fan-out per write (rows touched) — ★ one customer rename
    touching 8,402 orders is a different decision from touching 3.

 ⑤ ★ CHOOSE THE MECHANISM BEFORE YOU ADD THE COLUMN
    ① GENERATED column   — ★ cannot drift. If it fits, done.
    ② TRIGGER            — ★ atomic, unbypassable. The default.
    ③ same-txn app code  — ★ only if EXACTLY ONE service writes,
                            and you can prove it
    ④ async job / CDC    — ★ only with a bounded, monitored
                            staleness window
    ⇒ ★ IF YOU CANNOT NAME THE MECHANISM, YOU ARE NOT READY.

 ⑥ ★ THE RECONCILER SHIPS IN THE SAME PULL REQUEST
    • the query that proves copy = source (use IS DISTINCT FROM)
    • sampled for large tables, full nightly
    • scheduled, alerting on non-zero
    • ★ AND a self-repair statement
    ⇒ ★ IF YOU CANNOT WRITE THE RECONCILER, YOU CANNOT DETECT
      DRIFT, AND YOU WILL HAVE DRIFT.

 ⑦ CHECK THE WRITE DISTRIBUTION BEFORE A MAINTAINED COUNTER
    writes/sec on the HOTTEST key, not the average.
    ⇒ under ~100/sec: fine.
    ⇒ above: ★ you have created a hot row (Topic 61, case study 01).

 ⑧ ADD CONSTRAINTS SO THE COPY CANNOT BE ABSURD
    CHECK (review_count >= 0)
    CHECK (review_sum <= review_count * 5)
    ⇒ ★ catches a class of drift at write time rather than in a
      nightly report.

 ⑨ WRITE DOWN WHY — in a comment on the column
    COMMENT ON COLUMN products.review_count IS
      'Denormalised from reviews. Maintained by trg_review_stats.
       Reconciler: ops/reconcile_review_stats.sql (hourly).
       Added 2026-08 — the aggregate was 88% of a 2.4s p99.';
    ⇒ ★ the next engineer must be able to find the obligation.
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
Create `customers` and `orders` with 100,000 and 4,000,000 rows. (a) `EXPLAIN (ANALYZE, BUFFERS)` a single-row join and report the extra buffers the join costs. (b) Add `orders.customer_name`, backfill it, and measure the read improvement. (c) Measure the time and WAL for renaming a customer with 41 orders and one with 50,000. (d) State the read:write ratio at which this becomes worthwhile.

### Exercise 2 — medium (apply it)
For each, decide: index, snapshot, denormalisation, or materialised view — and justify:
(a) `orders.customer_email` for sending receipts · (b) `order_items.unit_price` at the time of purchase · (c) `products.review_count` on a listing page · (d) `users.last_login_at` · (e) `invoices.customer_billing_address` · (f) `categories.product_count` in a nav menu

For the ones you call denormalisation, write the trigger, the reconciler, and the constraint.

### Exercise 3 — hard (production simulation)
A marketplace listing endpoint runs at p99 2,400 ms and 4,200 rps, consuming 38% of all cluster database time. It joins `products`, `sellers` and aggregates `reviews`. The team proposes denormalising seller name, seller rating, review count and average rating onto `products`.

(a) From the `EXPLAIN` output, attribute the 2,388 ms across the seq scan, the seller join, the reviews aggregate, and the sort. What percentage is the seller join?
(b) Which part is fixable with an index alone? Write it and predict the new plan.
(c) After the index, the seller join runs 24 times instead of 41,204. What does that do to the case for denormalising it?
(d) For the reviews aggregate, apply the snapshot-vs-copy test and state the obligation precisely.
(e) Explain why storing `avg_rating` directly is wrong, and design the sum/count/generated alternative.
(f) Write the complete trigger, handling `INSERT`, `UPDATE` (including a changed `product_id`) and `DELETE`.
(g) Write the backfill, the constraints, the reconciler (sampled and full), and the self-repair.
(h) Measure the write cost and the write distribution on the hottest product. At what reviews/sec would this design fail, and what would you do then?
(i) The team wanted 4 denormalised columns. Justify shipping 2.
(j) Write the `COMMENT ON COLUMN` that lets the next engineer find the obligation.

---

## Mental model checkpoint

1. Define denormalisation in one sentence, naming both what you gain and what you owe.
2. Why is an index not denormalisation? What does that imply about the order you try things?
3. State the snapshot-vs-copy test. Give an example of each from an e-commerce schema.
4. Name the three failure modes of a denormalised copy. Which is silent?
5. Rank the four enforcement mechanisms and say when each is appropriate.
6. Why is a generated column strictly better than a trigger when it fits?
7. Why store sum and count rather than an average?
8. What is the one artefact that must ship in the same pull request as the column, and why?
9. A customer rename touches 8,402 order rows. Give the write cost in tuples, WAL and milliseconds, and explain why.
10. When does a trigger-maintained counter become a hot row, and what do you do about it?

---

## Quick reference card

**The trade:** read speed **now**, in exchange for a **permanent obligation** to keep copies in sync — paid by every write, in every code path, forever.

**The order to try things**
```
① measure (EXPLAIN ANALYZE, BUFFERS) and ★ ATTRIBUTE the cost
② ★ indexes · fix N+1 · index-only scan · work_mem · matview
③ ★ the snapshot test: "if the source changes, must this?"
④ read:write ratio + fan-out
⑤ ★ choose the mechanism BEFORE the column
⑥ ★ ship the reconciler in the same PR
```

**Mechanisms, best first**

| Mechanism | Atomic | Bypassable | Use when |
|---|---|---|---|
| ★ `GENERATED ALWAYS AS … STORED` | ✓ | ★ **never** | pure function of the same row |
| ★ trigger | ✓ | ★ **no** | the default for in-DB copies |
| same-txn app code | ✓ | ★ **yes** | exactly one writing service |
| async job / CDC | ✗ | — | aggregates, bounded staleness |

**The reconciler — non-negotiable**
```sql
SELECT count(*) FROM orders o JOIN customers c ON c.id=o.customer_id
 WHERE o.customer_name IS DISTINCT FROM c.name;   -- ★ alert if > 0
-- large tables: TABLESAMPLE SYSTEM (1) hourly, full nightly
-- ★ plus a self-repair UPDATE
```

**Constrain it**
```sql
CHECK (review_count >= 0)
CHECK (review_sum <= review_count * 5)
```

**Rules:** an index costs nothing in correctness — exhaust indexes first · a single-row join on an indexed FK is ~0.01 ms · a snapshot is not a copy — **name it** · store sum and count, generate the average · check the hottest-key write rate before a maintained counter · `COMMENT ON COLUMN` so the obligation is findable.

---

## When would I use this at work?

1. **Every time someone says "the join is slow."** It usually isn't. `EXPLAIN (ANALYZE, BUFFERS)` and attributing cost across the plan turns a vague complaint into three separate problems with three different fixes — usually one index and, at most, one denormalisation.

2. **Designing an order/invoice/payment schema.** Nearly every "denormalised" column in that domain is actually a *snapshot* — the price charged, the address shipped to, the tax rate applied. Recognising that means no trigger, no reconciler, no obligation — and a correctly named column that nobody later "fixes."

3. **Reviewing a PR that adds a copied column.** Two questions settle it: *what mechanism keeps it in sync, and where is the reconciliation query?* If either answer is missing, the PR is incomplete regardless of how good the benchmark looks.

4. **Inheriting a schema with unexplained duplicate columns.** Run the reconciler you write yourself. A non-zero result tells you whether you have drift or history — and that distinction determines whether you're looking at a bug or at a design decision nobody documented.

---

## Connected topics

**Understand before this:** 29–38 (normalisation — especially 38's snapshot-vs-copy test), 10–19 (indexes — the free alternative), 46 (MVCC — why updating 8,402 rows costs what it does), 24 (constraints).

**This unlocks:**
- **54** — *when* to denormalise: the decision framework in full, with measured thresholds
- **55** — the concrete patterns and their trade-offs
- **56** — materialised views: async denormalisation the database maintains
- **57** — caching: denormalisation outside the database
- **61** — counters and hot rows: when a maintained aggregate becomes a bottleneck
- **66** — N+1: the problem most often misdiagnosed as "the join is slow"
