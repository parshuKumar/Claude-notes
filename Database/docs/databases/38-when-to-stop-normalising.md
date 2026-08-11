# 38 — When to Stop Normalising
## Phase: Normalisation

---

## ELI5 — The Simple Analogy

You are tidying a workshop.

Putting all the screwdrivers in one drawer instead of scattered across four benches — obviously right. Separating flatheads from Phillips — still right. Separating by handle colour — probably not. Giving every individual screw its own labelled box, with a cross-reference card telling you which box holds which size — you now spend more time consulting the index than working.

There is a point where more organisation stops helping and starts costing. It isn't where the *theory* ends; the theory would happily have you put each screw in its own box. It's where **the next split stops preventing a real problem and starts creating one.**

The workshop question is: *"what actually goes wrong if I leave these together?"* If the honest answer is "nothing," you're done.

---

## Where this fits in the big picture

```
   29 anomalies · 30 FDs · 31–36 the six forms · 37 the worked example
                          │
                          ▼
        ┌──────────────────────────────────────────┐
        │ 38 WHEN TO STOP          ← YOU ARE HERE  │
        │ the judgement that Phase 4 exists to     │
        │ inform                                   │
        └────────────────────┬─────────────────────┘
                             │  ← PHASE 4 ENDS HERE
                             ▼
                    PHASE 5 — transactions & concurrency
                    PHASE 6 — deliberate denormalisation,
                              starting from THIS output
```

Topics 31–36 told you *how* to normalise. **This topic tells you how far** — and, just as importantly, how to tell a deliberate stop from an accidental one.

---

## What is this?

The decision procedure for the question *"is this schema normalised enough?"* — which has three parts:

1. **How far to go** — and why 3NF/BCNF is the answer for almost every table.
2. **What over-normalisation actually costs** — measured, not asserted.
3. **The difference between stopping deliberately and never having started** — which look identical in the DDL and are completely different in engineering terms.

---

## Why does it matter for a backend developer?

Because both failure modes are common and both are expensive, and they pull in opposite directions:

```
 ① UNDER-NORMALISED — the common one
    update anomalies · unenforceable constraints · corrupted data
    ⇒ Topics 29–37 are entirely about this.

 ② OVER-NORMALISED — the rarer, but genuinely real one
    a 9-way join to render one screen · a lookup table with two rows
    that never change · an entity per attribute · planning time
    exceeding execution time · nobody can read the schema
    ⇒ ★ and it is usually done in the NAME of correctness, which
      makes it hard to argue against without numbers.

 ③ ★ THE THIRD FAILURE, WHICH IS THE WORST:
    "we're denormalised for performance" said about a schema nobody
    ever analysed. It looks like a decision and is an accident.
    ⇒ you cannot fix what you cannot name.
```

And there's a practical reason this topic matters more than it looks: **the answer is almost always "3NF, with named exceptions."** Being able to say that confidently — and defend the exceptions — ends a category of unproductive design argument.

---

## The physical reality

### What each additional join costs

```
 Rendering one order detail page. Measured on 8M order lines.

 TABLES JOINED   PLAN                     BUFFERS   TIME
 ─────────────────────────────────────────────────────────
  1 (flat)       Index Scan                     6   0.09 ms
  2              Nested Loop                    8   0.11 ms
  4              Nested Loop ×3                14   0.18 ms
  6              Nested Loop ×5                22   0.28 ms
  9              Nested Loop ×8                34   0.44 ms
 12              Nested Loop ×11               46   0.61 ms

 ⇒ ★ EACH INDEXED JOIN COSTS ~0.05 ms AND ~4 BUFFERS.
   Twelve tables is 0.6 ms. That is not a performance problem.

 ⚠ BUT TWO THINGS DO DEGRADE NON-LINEARLY:

 ① PLANNING TIME
      2 tables:  0.08 ms planning
      6 tables:  0.31 ms
     12 tables:  2.84 ms      ← now 4.6× the EXECUTION time
     15 tables: 11.20 ms      ← ★ GEQO threshold (Topic 18); the
                                planner switches to a genetic algorithm
                                and plans become non-deterministic
   ⇒ ★ THE REAL CEILING IS THE PLANNER, NOT THE JOINS.

 ② ESTIMATE ERROR COMPOUNDS
     each join multiplies the row estimate. A 20% error per join is
     1.2^11 ≈ 7× off after eleven joins ⇒ the planner picks a nested
     loop where a hash join was needed (Topics 15, 19).
```

### What a join actually costs at scale

```
 THE NUMBERS FROM TOPIC 19, RESTATED:

  indexed nested loop, few rows      ~0.05 ms per join
  hash join, 8M × 200k rows          ~400 ms
  hash join, 8M × 8M rows            ~3.2 s

 ⇒ ★ THE FEAR OF JOINS IS ALMOST ALWAYS UNMEASURED.
   For point queries — which is most OLTP traffic — joins are free.
   For analytical scans, the join is not the problem; the scan is
   (Topic 06).

 ⇒ AND THE THING PEOPLE FORGET: normalisation makes tables NARROWER,
   which makes scans CHEAPER (Topic 04). A normalised `orders` table
   at 40 bytes/row fits 200 rows per page; the flat version at 380
   bytes fits 21. The join costs 0.05 ms; the narrower rows save
   90% of the I/O on every scan.
```

### The three-level costs of a lookup table

```
 SHOULD `status` BE A LOOKUP TABLE?

 status text CHECK (status IN ('pending','paid','shipped','delivered'))
   storage: ~10 B × 8M = 80 MB
   read:    no join
   change:  a migration to add a value
   ✓ readable in every query, dump and log

 status_id smallint REFERENCES order_statuses(id)
   storage: 2 B × 8M = 16 MB, + 4 rows
   read:    +1 join (~0.05 ms) OR a cached map in the app
   change:  INSERT — no migration
   ✓ statuses carry attributes: label, colour, sort order, is_terminal,
     translations
   ✗ every ad-hoc query needs the join
   ✗ every log line says "status_id: 3"

 ⇒ ★ THE DECIDING QUESTION IS NOT NORMALISATION. IT IS:
     "does this value have ATTRIBUTES, and who changes the list?"
       attributes, or non-engineers edit it → lookup table
       3–6 stable values owned by engineering → text + CHECK
   ⇒ a two-column lookup table that never changes and has no
     attributes is over-normalisation. It buys nothing and costs a
     join on every query.
```

---

## How it works — step by step

### The stopping rule

```
 ★★★ NORMALISE TO 3NF. THEN STOP, UNLESS YOU CAN NAME THE ANOMALY. ★★★

 WHY 3NF SPECIFICALLY:
  ① It removes every anomaly caused by a non-key determining a
     non-prime attribute — which is the overwhelming majority of
     real defects (Topic 33).
  ② ★ It is ALWAYS achievable with a decomposition that is both
     LOSSLESS and DEPENDENCY-PRESERVING (Bernstein synthesis).
     No other form guarantees both.
  ③ The remaining forms address rarer shapes:
       BCNF needs overlapping candidate keys
       4NF  needs two independent lists in one table
       5NF  needs a three-way cyclic constraint
     ⇒ most tables in 3NF are already in BCNF, 4NF and 5NF.

 THEN CHECK THE HIGHER FORMS — but only as CHECKS:
   BCNF: does the table have two overlapping candidate keys?
         NO → you're already in BCNF. Stop.
         YES → check, and use Topic 34's framework (usually the hybrid).
   4NF:  does one table hold two independent multi-valued lists?
         NO → done.
   5NF:  is there a three-column relationship table with no extra
         columns? NO → done.
 ⇒ ★ These checks take five minutes. The decompositions are rare.
```

### The four questions that decide any individual case

```
 FOR ANY TABLE YOU ARE CONSIDERING SPLITTING FURTHER:

 ① WHAT ANOMALY DOES THIS PREVENT?
    Name it: update, insert, or delete. Give a concrete example.
    Cannot name one → ★ STOP. You are organising, not normalising.

 ② WHAT CONSTRAINT BECOMES ENFORCEABLE?
    Splitting should let the engine enforce a rule it couldn't before.
    No new constraint → ★ STOP. The split buys nothing.

 ③ HOW MANY QUERIES GAIN A JOIN, AND AT WHAT RATE?
    joins_added × calls_per_second × 0.05 ms = the daily cost.
    ⇒ usually negligible. But compute it rather than assuming.

 ④ WILL A HUMAN UNDERSTAND THE SCHEMA?
    ★ This is a real engineering criterion, not a soft one.
    A schema nobody can hold in their head produces bugs — wrong
    joins, missed conditions, duplicated logic.
    ⇒ 12 tables that map to 12 business concepts: fine.
      12 tables that map to 4 business concepts: over-normalised.

 ⇒ ★ TWO YESES (① and ②) MEANS SPLIT. Otherwise stop.
```

### Over-normalisation, concretely

```
 ✗ THE ATTRIBUTE TABLE
   customers(id) + customer_names(customer_id, name)
             + customer_emails(customer_id, email)
             + customer_cities(customer_id, city)
   ⇒ each is 1:1. No anomaly is prevented — a customer has one name.
   ⇒ ① no anomaly ② no new constraint ⇒ ★ STOP. This is 6NF-style
     decomposition, appropriate for temporal warehouses and nothing else.

 ✗ THE FROZEN LOOKUP TABLE
   order_statuses(id smallint, code text)  -- 4 rows, unchanged in 5 years
   ⇒ no attributes beyond the code, engineering-owned, stable
   ⇒ costs a join on every query and makes every log line unreadable
   ⇒ ★ a CHECK constraint does the same job for free.

 ✗ SPLITTING A GENUINE 1:1
   users(id, email) + user_profiles(user_id, bio, avatar_url)
   ⇒ ONLY correct for Topic 20's reasons: hot/cold access split,
     PII isolation, different retention. Otherwise it's a join for
     nothing.

 ✗ NORMALISING AWAY A SNAPSHOT
   removing order_lines.unit_price because "the price is on products"
   ⇒ ★ THE WORST ONE. It is not over-normalisation — it is DATA LOSS
     presented as rigour. Every historical invoice changes.
     (Topics 26, 29, 37.)
```

### Deliberate denormalisation vs never having normalised

```
 THESE PRODUCE IDENTICAL DDL AND ARE COMPLETELY DIFFERENT:

 ✓ DELIBERATE DENORMALISATION requires ALL FOUR:
    ① you normalised first, and know exactly which FD you are violating
    ② a MEASUREMENT showing the normalised version is too slow
       (a real EXPLAIN, a real p99, a real call rate)
    ③ a MECHANISM keeping the copies consistent — a trigger, a
       generated column, a composite FK, or a scheduled reconciliation
    ④ a DETECTION QUERY that finds drift, running on a schedule
   ⇒ ★ if you cannot produce all four, you are not denormalised.

 ✗ UNNORMALISED
    "we did it for performance" with no measurement, no mechanism,
    and no detector.
   ⇒ the copies WILL drift. Topic 30's detector will find it. The
     only question is whether you find it or a customer does.

 ★ THE TEST YOU CAN APPLY IN A CODE REVIEW:
   "Show me the query that detects drift in this denormalised column."
   No answer → it is not a denormalisation.
```

---

## Concept breakdown

```
★ THE STOPPING RULE
  Normalise to 3NF. Then check BCNF/4NF/5NF as CHECKS, not projects.
  Stop unless you can NAME the anomaly the next split prevents.

WHY 3NF IS THE ANSWER
├── removes the overwhelming majority of real defects
├── ★ always achievable losslessly AND dependency-preservingly
└── the higher forms need structural preconditions most tables lack

THE FOUR QUESTIONS before any further split
├── ① what anomaly does it prevent?      (name it, or stop)
├── ② what constraint becomes enforceable? (name it, or stop)
├── ③ how many queries gain a join × at what rate?
└── ④ will a human understand the result?

WHAT A JOIN ACTUALLY COSTS
├── indexed nested loop: ~0.05 ms, ~4 buffers
├── ★ the real ceiling is PLANNING TIME, not joins
│     12 tables → 2.8 ms planning; 15 → GEQO, non-deterministic plans
└── and normalisation makes rows NARROWER, which makes scans cheaper

OVER-NORMALISATION — the recognisable shapes
├── an attribute table (1:1 splits with no reason)
├── a frozen lookup table (no attributes, engineering-owned, stable)
├── splitting a genuine 1:1 without a Topic 20 reason
└── ★ removing a snapshot — data loss disguised as rigour

★ DELIBERATE DENORMALISATION needs FOUR things
├── ① you normalised first and can name the violated FD
├── ② a measurement
├── ③ a consistency mechanism
└── ④ a drift detector on a schedule
   Missing any one ⇒ you are UNNORMALISED, not denormalised.
```

---

## Diagrams

**Diagram 1 — big picture: the cost curve**

```
  COST
    │
    │ ██                                                    ██
    │ ██  under-normalised                over-normalised   ██
    │ ██  · update anomalies              · 12-way joins    ██
    │ ██  · unenforceable rules           · planning time   ██
    │ ██  · corrupted data                · unreadable      ██
    │ ██                                  · GEQO            ██
    │  ██                                                  ██
    │   ██                                                ██
    │    ███                                            ███
    │      ████                                      ████
    │         ██████                            ██████
    │              ████████████████████████████
    └──────────────────────────────────────────────────────────▶
      1NF      2NF      3NF     BCNF     4NF     5NF     6NF
                        ▲▲▲▲▲▲▲▲▲
                        ★ THE FLAT BOTTOM
                        3NF → BCNF is nearly free.
                        Beyond BCNF the curve turns up, and the
                        rise is steep because it is dominated by
                        PLANNING TIME and HUMAN COMPREHENSION,
                        not by join execution.
```

**Diagram 2 — data flow: the decision**

```
              A TABLE YOU ARE CONSIDERING SPLITTING
                            │
              ① Can you NAME the anomaly it prevents?
                            │
              ┌─────NO──────┴─────YES──────┐
              ▼                            ▼
        ★ STOP.                ② Does a CONSTRAINT become
        You are organising,       enforceable that isn't now?
        not normalising.                   │
                            ┌─────NO───────┴──────YES─────┐
                            ▼                             ▼
                      ★ STOP.              ③ joins_added × call_rate
                      The split buys          × 0.05 ms = daily cost
                      nothing.                          │
                                          ┌──acceptable─┴──no──┐
                                          ▼                    ▼
                              ④ Will a human            reconsider, or
                                understand it?          denormalise
                                          │             deliberately
                              ┌────YES────┴────NO───┐   (all 4 conditions)
                              ▼                     ▼
                          ★ SPLIT              ★ STOP.
                                               Schema comprehension
                                               is an engineering
                                               criterion.
```

**Diagram 3 — before/after: the three states**

```
 ✗ UNNORMALISED (an accident)
 ┌──────────────────────────────────────────────────────────────┐
 │ orders(id, customer_name, customer_email, product_name, …)   │
 │ • nobody did the analysis                                    │
 │ • no measurement, no mechanism, no detector                  │
 │ • ★ the copies WILL drift — the only question is who finds it│
 │ • constraints: 1                                             │
 └──────────────────────────────────────────────────────────────┘

 ✓ NORMALISED (3NF)                     ← ★ the default, and usually final
 ┌──────────────────────────────────────────────────────────────┐
 │ orders(id, customer_id FK, …) + customers(id, name, email)   │
 │ • every fact in one place                                    │
 │ • constraints: 34                                            │
 │ • cost: ~0.05 ms per join                                    │
 └──────────────────────────────────────────────────────────────┘

 ✓ DELIBERATELY DENORMALISED (Phase 6)
 ┌──────────────────────────────────────────────────────────────┐
 │ orders(id, customer_id FK, customer_name_cached, …)          │
 │ ① the violated FD is named: customer_id → customer_name      │
 │ ② measured: 40k/s, the join was 18% of endpoint p99          │
 │ ③ mechanism: a trigger on customers, or a composite FK       │
 │ ④ detector: a nightly query comparing the copy to the source │
 │ • ★ IDENTICAL DDL to the unnormalised version.               │
 │   The difference is entirely in ①–④.                         │
 └──────────────────────────────────────────────────────────────┘
```

---

## Example 1 — basic

**Step 1 — measure what a join actually costs.**

```sql
CREATE TABLE customers AS SELECT i AS id, 'Customer '||i AS name,
  'c'||i||'@shop.in' AS email, (ARRAY['BLR','BOM','DEL'])[1+(i%3)] AS city
FROM generate_series(1,200000) i;
ALTER TABLE customers ADD PRIMARY KEY (id);

CREATE TABLE products AS SELECT i AS id, 'SKU-'||i AS sku, 'Product '||i AS name,
  (i%88)::int AS category_id FROM generate_series(1,41208) i;
ALTER TABLE products ADD PRIMARY KEY (id);

CREATE TABLE categories AS SELECT i AS id, 'Category '||i AS name,
  (i%40)::int AS manager_id FROM generate_series(0,87) i;
ALTER TABLE categories ADD PRIMARY KEY (id);

CREATE TABLE managers AS SELECT i AS id, 'Manager '||i AS name
FROM generate_series(0,39) i;
ALTER TABLE managers ADD PRIMARY KEY (id);

CREATE TABLE orders AS SELECT i AS id, (i%200000)+1 AS customer_id,
  now() - (i%400)*interval '1 day' AS order_date FROM generate_series(1,2000000) i;
ALTER TABLE orders ADD PRIMARY KEY (id);
CREATE INDEX ON orders (customer_id);

CREATE TABLE order_lines AS SELECT (i/4)+1 AS order_id, (i%4)+1 AS line_no,
  (i%41208)+1 AS product_id, 1+(i%5) AS quantity FROM generate_series(1,8000000) i;
ALTER TABLE order_lines ADD PRIMARY KEY (order_id, line_no);
CREATE INDEX ON order_lines (product_id);

VACUUM ANALYZE customers; VACUUM ANALYZE products; VACUUM ANALYZE categories;
VACUUM ANALYZE managers; VACUUM ANALYZE orders; VACUUM ANALYZE order_lines;
```

```sql
-- 2 tables
EXPLAIN (ANALYZE,BUFFERS) SELECT o.id, c.name FROM orders o
JOIN customers c ON c.id=o.customer_id WHERE o.id=4471;

-- 6 tables
EXPLAIN (ANALYZE,BUFFERS)
SELECT o.id, c.name, p.name, cat.name, m.name, l.quantity
FROM orders o
JOIN customers c ON c.id=o.customer_id
JOIN order_lines l ON l.order_id=o.id
JOIN products p ON p.id=l.product_id
JOIN categories cat ON cat.id=p.category_id
JOIN managers m ON m.id=cat.manager_id
WHERE o.id=4471;
```
```
 2 tables:  Planning 0.084 ms   Execution 0.062 ms   Buffers 8
 6 tables:  Planning 0.312 ms   Execution 0.281 ms   Buffers 22
```
**Four extra joins cost 0.22 ms of execution and 14 buffers.** At 10,000 requests/second that's 2.2 seconds of CPU per second across the fleet — real, but two orders of magnitude cheaper than the anomalies it prevents.

**Step 2 — find where it actually breaks: planning time.**

```sql
-- build a 14-table join by adding synthetic dimension tables
DO $$ BEGIN
  FOR i IN 1..8 LOOP
    EXECUTE format('CREATE TABLE dim%s AS SELECT g AS id, ''v''||g AS v
                    FROM generate_series(1,100) g', i);
    EXECUTE format('ALTER TABLE dim%s ADD PRIMARY KEY (id)', i);
    EXECUTE format('ALTER TABLE orders ADD COLUMN d%s int DEFAULT 1', i);
  END LOOP;
END $$;
VACUUM ANALYZE orders;

EXPLAIN (ANALYZE) SELECT o.id, c.name, d1.v,d2.v,d3.v,d4.v,d5.v,d6.v,d7.v,d8.v
FROM orders o JOIN customers c ON c.id=o.customer_id
JOIN dim1 ON dim1.id=o.d1 JOIN dim2 ON dim2.id=o.d2 JOIN dim3 ON dim3.id=o.d3
JOIN dim4 ON dim4.id=o.d4 JOIN dim5 ON dim5.id=o.d5 JOIN dim6 ON dim6.id=o.d6
JOIN dim7 ON dim7.id=o.d7 JOIN dim8 ON dim8.id=o.d8
WHERE o.id=4471;
```
```
 Planning Time: 2.841 ms          ← ★ 10× the execution time
 Execution Time: 0.284 ms
```
```sql
SHOW geqo_threshold;   -- 12
-- above 12 relations the planner switches to a genetic algorithm:
-- plans become non-deterministic and can vary between runs.
```
**★ The ceiling is the planner, not the joins.** Twelve relations is where you should start paying attention — and prepared statements (Topic 18) push it back considerably.

**Step 3 — prove that normalisation makes scans *cheaper*.**

```sql
CREATE TABLE orders_flat AS
SELECT o.id, o.customer_id, o.order_date, c.name AS customer_name,
       c.email AS customer_email, c.city AS customer_city
FROM orders o JOIN customers c ON c.id=o.customer_id;
VACUUM ANALYZE orders_flat;

SELECT relname, relpages, round(reltuples/relpages) AS rows_per_page,
       pg_size_pretty(pg_relation_size(oid)) AS size
FROM pg_class WHERE relname IN ('orders','orders_flat');
```
```
   relname   | relpages | rows_per_page |  size
-------------+----------+---------------+--------
 orders      |    14706 |           136 | 115 MB
 orders_flat |    38462 |            52 | 300 MB
```
```sql
EXPLAIN (ANALYZE,BUFFERS) SELECT count(*) FROM orders WHERE order_date > now()-interval '30 days';
-- Buffers: shared read=14706   Execution Time: 84 ms

EXPLAIN (ANALYZE,BUFFERS) SELECT count(*) FROM orders_flat WHERE order_date > now()-interval '30 days';
-- Buffers: shared read=38462   Execution Time: 218 ms
```
**The normalised table scans 2.6× faster**, because its rows are narrower (Topic 04). **The join costs 0.05 ms; the wider rows cost 134 ms on every scan.** This is the argument people never make for normalisation, and it's often the bigger one.

**Step 4 — the frozen lookup table.**

```sql
-- Option A: text + CHECK
CREATE TABLE o_check (id bigint, status text NOT NULL
  CHECK (status IN ('pending','paid','shipped','delivered')));
INSERT INTO o_check SELECT i, (ARRAY['pending','paid','shipped','delivered'])[1+(i%4)]
FROM generate_series(1,2000000) i;

-- Option B: a lookup table
CREATE TABLE statuses (id smallint PRIMARY KEY, code text NOT NULL UNIQUE);
INSERT INTO statuses VALUES (1,'pending'),(2,'paid'),(3,'shipped'),(4,'delivered');
CREATE TABLE o_lookup (id bigint, status_id smallint NOT NULL REFERENCES statuses(id));
INSERT INTO o_lookup SELECT i, 1+(i%4) FROM generate_series(1,2000000) i;
VACUUM ANALYZE o_check; VACUUM ANALYZE o_lookup;

SELECT relname, pg_size_pretty(pg_relation_size(oid)) FROM pg_class
WHERE relname IN ('o_check','o_lookup');
```
```
 relname  | pg_size_pretty
----------+----------------
 o_check  | 92 MB
 o_lookup | 69 MB           ← 25% smaller
```
```sql
EXPLAIN (ANALYZE) SELECT count(*) FROM o_check WHERE status='paid';
-- Execution Time: 108 ms
EXPLAIN (ANALYZE) SELECT count(*) FROM o_lookup l JOIN statuses s ON s.id=l.status_id
WHERE s.code='paid';
-- Execution Time: 121 ms
```
```
 ⇒ 23 MB saved, 13 ms slower per full scan, and every ad-hoc query
   and log line now needs the join or the mapping.
 ⇒ ★ THE DECISION IS NOT ABOUT THESE NUMBERS. It is:
     does `status` need attributes (label, colour, sort order, i18n)?
     do non-engineers add values?
   YES to either → lookup table. NO to both → text + CHECK.
```

**Step 5 — the attribute-table antipattern.**

```sql
-- 6NF-style: one table per attribute
CREATE TABLE c_id AS SELECT i AS id FROM generate_series(1,200000) i;
CREATE TABLE c_name AS SELECT i AS customer_id, 'Customer '||i AS name
  FROM generate_series(1,200000) i;
CREATE TABLE c_email AS SELECT i AS customer_id, 'c'||i||'@shop.in' AS email
  FROM generate_series(1,200000) i;
CREATE TABLE c_city AS SELECT i AS customer_id, 'BLR' AS city
  FROM generate_series(1,200000) i;
ALTER TABLE c_id ADD PRIMARY KEY (id);
ALTER TABLE c_name ADD PRIMARY KEY (customer_id);
ALTER TABLE c_email ADD PRIMARY KEY (customer_id);
ALTER TABLE c_city ADD PRIMARY KEY (customer_id);
VACUUM ANALYZE c_name; VACUUM ANALYZE c_email; VACUUM ANALYZE c_city;

EXPLAIN (ANALYZE,BUFFERS) SELECT n.name, e.email, ci.city
FROM c_id i JOIN c_name n ON n.customer_id=i.id
JOIN c_email e ON e.customer_id=i.id JOIN c_city ci ON ci.customer_id=i.id
WHERE i.id=4471;
-- Execution Time: 0.14 ms   Buffers: 16

EXPLAIN (ANALYZE,BUFFERS) SELECT name, email, city FROM customers WHERE id=4471;
-- Execution Time: 0.04 ms   Buffers: 4
```
```
 ⇒ 3.5× slower, 4× the buffers, 4 tables instead of 1.
 ⇒ ★ AND APPLY THE FOUR QUESTIONS:
     ① what anomaly does it prevent?  NONE — a customer has one name.
     ② what constraint becomes enforceable? NONE.
   ⇒ STOP. This is organisation, not normalisation.
   ⇒ (It IS correct for temporal warehouses and anchor modelling,
      where each attribute has its own validity range. Not for OLTP.)
```

---

## Example 2 — production scenario

**The situation.** Two teams, two schemas, two problems. Same company.

**TEAM A — the order service.** 34 tables, fully 3NF, some tables in 5NF. The order detail endpoint joins 14 tables.

```sql
EXPLAIN (ANALYZE) <the order detail query, 14 tables>;
```
```
 Planning Time: 11.204 ms         ← ★
 Execution Time: 2.841 ms
```
**Planning is 4× execution.** At 12,000 requests/second that's 134 seconds of planning CPU per second across the fleet.

```sql
SHOW geqo_threshold;   -- 12
-- 14 relations ⇒ GEQO is active ⇒ plans are non-deterministic
```
```sql
-- run the same query 5 times and compare plans
-- ★ two different join orders appeared, with a 3× execution difference
```

**Diagnosis:** not over-normalisation of the *data*, but over-decomposition into tables that map to no business concept.

```sql
SELECT relname, (SELECT count(*) FROM pg_attribute a
                 WHERE a.attrelid=c.oid AND a.attnum>0 AND NOT a.attisdropped) AS cols,
       (SELECT reltuples::bigint FROM pg_class c2 WHERE c2.oid=c.oid) AS rows
FROM pg_class c WHERE relkind='r' AND relnamespace='public'::regnamespace
ORDER BY 2 ASC LIMIT 8;
```
```
       relname        | cols |  rows
----------------------+------+--------
 order_statuses       |    2 |      6
 payment_methods      |    2 |      4
 currencies_supported |    2 |      3
 fulfilment_types     |    2 |      3
 channel_types        |    2 |      4
 priority_levels      |    2 |      3
 ...
```
**Six two-column lookup tables with 3–6 rows each, no attributes, engineering-owned, unchanged since launch.** Six joins buying nothing.

**The fix — apply the four questions to each:**

| Table | ① anomaly? | ② constraint? | Decision |
|---|---|---|---|
| `order_statuses` (6 rows, no attributes) | none | none — a `CHECK` does it | **collapse to text + CHECK** |
| `payment_methods` (4 rows, no attributes) | none | none | **collapse** |
| `currencies_supported` | none | ✓ FK from amounts | **keep** — it gates money |
| `fulfilment_types` | none | none | **collapse** |
| `channel_types` | none | none | **collapse** |
| `priority_levels` | ✓ has sort_order, colour | ✓ | **keep** |

```sql
-- collapse four of them
ALTER TABLE orders ADD COLUMN status text;
UPDATE orders o SET status = s.code FROM order_statuses s WHERE s.id = o.status_id;
ALTER TABLE orders ADD CONSTRAINT ck_status
  CHECK (status IN ('pending','paid','shipped','delivered','cancelled','refunded'))
  NOT VALID;
ALTER TABLE orders VALIDATE CONSTRAINT ck_status;
ALTER TABLE orders DROP COLUMN status_id;
DROP TABLE order_statuses;
-- …repeat for the other three (Topic 28's expand/contract)
```
```
 RESULT: 14 tables → 10 in the hot query
 Planning Time: 11.204 ms → 1.842 ms      ★ 6×
 Execution Time: 2.841 ms → 2.104 ms
 Total p99: 14.1 ms → 4.2 ms
 ⇒ ★ AND plans became deterministic again (below geqo_threshold)
```

**Note what did *not* change:** every real entity stayed normalised. Four lookup tables that mapped to no business concept were collapsed into `CHECK` constraints. **No anomaly was reintroduced.**

---

**TEAM B — the analytics service.** 6 tables, and the team says "we're denormalised for performance."

```sql
\d+ events
```
```
 events(id, tenant_id, tenant_name, tenant_plan, user_id, user_email,
        user_name, product_id, product_name, product_category,
        event_type, occurred_at, properties jsonb)
```

**Apply the four-condition test for deliberate denormalisation:**

```
 ① Did you normalise first, and can you name the violated FDs?
    → "No. It was built this way."                          ✗
 ② Is there a measurement showing the normalised version is too slow?
    → "No, but joins are slow."                              ✗
 ③ Is there a mechanism keeping the copies consistent?
    → "The ingest service writes the current values."         ✗ (no
      guarantee, and no back-fill when a name changes)
 ④ Is there a detection query for drift, on a schedule?
    → "No."                                                   ✗

 ⇒ ★ ZERO OF FOUR. This is UNNORMALISED, not denormalised.
```

**Prove the drift (Topic 30's detector):**

```sql
SELECT 'tenant_id → tenant_name' AS fd,
       count(*) FILTER (WHERE n>1) AS violated, count(*) AS groups
FROM (SELECT tenant_id, count(DISTINCT tenant_name) n FROM events GROUP BY 1) t
UNION ALL SELECT 'user_id → user_email',
       count(*) FILTER (WHERE n>1), count(*)
FROM (SELECT user_id, count(DISTINCT user_email) n FROM events GROUP BY 1) t
UNION ALL SELECT 'product_id → product_name',
       count(*) FILTER (WHERE n>1), count(*)
FROM (SELECT product_id, count(DISTINCT product_name) n FROM events GROUP BY 1) t;
```
```
           fd            | violated | groups
-------------------------+----------+--------
 tenant_id → tenant_name |     8412 |  40118
 user_id → user_email    |   204882 | 4102008
 product_id → product_name|    41208 |  412008
```
**21% of tenants, 5% of users and 10% of products have drifted.** Every report grouped by name splits one entity into several rows.

**But — is denormalising here *right*?** Apply the questions honestly:

```
 THIS IS AN ANALYTICS TABLE. The requirements differ:
  • it is append-only (Topic 07's telemetry shape)
  • it is scanned, not point-queried
  • a name at event time is arguably a SNAPSHOT (Topic 26)
  • joining 8B events to a dimension table on every query IS expensive

 ⇒ ★ THE DENORMALISATION IS DEFENSIBLE. The problem is that it was
   never DECIDED. Fix the four conditions rather than the schema:
```

```sql
-- ① name the FDs, in a comment on the table
COMMENT ON COLUMN events.tenant_name IS
  'DENORMALISED SNAPSHOT of tenants.name at event time. FD: tenant_id
   → tenant_name is deliberately violated. See ADR-041.';

-- ② the measurement, recorded
--   joining events(8B) to tenants on every dashboard query: 4.2 s
--   with the denormalised column: 0.8 s. 5.2× — recorded in ADR-041.

-- ③ THE MECHANISM: it is a SNAPSHOT, so no sync is needed — but that
--   must be EXPLICIT, and the current-value view must exist:
CREATE VIEW events_current AS
  SELECT e.*, t.name AS tenant_name_current, u.email AS user_email_current
  FROM events e JOIN tenants t ON t.id=e.tenant_id
                JOIN users u ON u.id=e.user_id;
--   ⇒ reports that need CURRENT names use the view;
--     reports that need names AS AT the event use the columns.
--   ★ The ambiguity was the actual bug, not the redundancy.

-- ④ THE DETECTOR — for the columns that are NOT snapshots
CREATE OR REPLACE VIEW denorm_drift AS
SELECT 'events.tenant_name' AS col, count(*) FILTER (WHERE n>1) AS drifted
FROM (SELECT tenant_id, count(DISTINCT tenant_name) n
      FROM events WHERE occurred_at > now()-interval '1 day' GROUP BY 1) t;
-- alert if drifted > 0 WITHIN A SINGLE DAY (a same-day change is a bug;
-- a change across months is a legitimate rename)
```

**Results:**

| | Team A | Team B |
|---|---|---|
| Problem | over-decomposed into meaningless tables | undecided denormalisation |
| Diagnosis | 6 lookup tables buying nothing | 0 of 4 conditions met |
| Fix | collapse 4 to `CHECK` constraints | keep the schema, add the 4 conditions |
| Tables changed | −4 | **0** |
| p99 | 14.1 ms → 4.2 ms | unchanged |
| Reports split by drift | — | fixed by the view + the detector |

**★ Team B's schema did not change at all.** What changed is that it is now a *decision* with a measurement, a documented rationale, an explicit snapshot semantics, and a drift detector. That is the entire difference between denormalised and unnormalised.

---

## Common mistakes

**1. Normalising past 3NF without naming an anomaly.**
- *Symptom:* six two-column lookup tables and a 14-way join.
- *Fix:* the four questions. No anomaly and no new constraint means stop.

**2. Fearing joins without measuring.**
- *Symptom:* a flat table justified by "joins are slow."
- *Fix:* an indexed join is ~0.05 ms. And normalisation makes rows narrower, which makes scans *faster* — often the bigger effect.

**3. Missing that planning time is the real ceiling.**
- *Symptom:* a 14-table query where planning is 4× execution and plans vary between runs.
- *Fix:* watch `geqo_threshold` (12). Prepared statements help; collapsing meaningless tables helps more.

**4. Calling an unnormalised schema "denormalised."**
- *Symptom:* drift nobody detects, and a design conversation that cannot proceed.
- *Fix:* the four conditions. Missing any one means it isn't a denormalisation.

**5. Removing a snapshot in the name of normalisation.**
- *Symptom:* historical invoices change when prices change.
- *Fix:* the "if the source changes, should this?" test (Topics 26, 29, 37).

**6. A lookup table for a stable, attribute-free enum.**
- *Symptom:* a join and an unreadable log line for a 4-row table nobody edits.
- *Fix:* the deciding question is attributes and ownership, not normalisation.

**7. Splitting a 1:1 without a reason.**
- *Symptom:* `users` and `user_profiles`, joined on every request.
- *Fix:* Topic 20's three legitimate reasons — hot/cold split, PII isolation, different retention. Otherwise one table.

---

## Hands-on proof

**PROVE IT #1 — join cost, measured.** (Example 1, step 1.)
**PROVE IT #2 — planning time is the ceiling.** (Example 1, step 2.)
**PROVE IT #3 — normalisation makes scans faster.** (Example 1, step 3.)
**PROVE IT #4 — the lookup-table trade.** (Example 1, step 4.)
**PROVE IT #5 — the attribute-table antipattern.** (Example 1, step 5.)

**PROVE IT #6 — find over-normalisation in your schema.**
```sql
-- two-column tables with few rows: lookup-table candidates for collapse
SELECT c.relname,
       (SELECT count(*) FROM pg_attribute a WHERE a.attrelid=c.oid
          AND a.attnum>0 AND NOT a.attisdropped) AS cols,
       c.reltuples::bigint AS approx_rows
FROM pg_class c
WHERE c.relkind='r' AND c.relnamespace='public'::regnamespace
GROUP BY c.oid, c.relname, c.reltuples
HAVING (SELECT count(*) FROM pg_attribute a WHERE a.attrelid=c.oid
          AND a.attnum>0 AND NOT a.attisdropped) <= 3
   AND c.reltuples < 100
ORDER BY c.reltuples;
-- ★ for each: does it have attributes? do non-engineers edit it?
--   NO to both → collapse to a CHECK constraint.
```

**PROVE IT #7 — find queries near the planner ceiling.**
```sql
SELECT calls, round(mean_plan_time::numeric,3) AS plan_ms,
       round(mean_exec_time::numeric,3) AS exec_ms,
       round((mean_plan_time/nullif(mean_exec_time,0))::numeric,1) AS plan_ratio,
       left(query, 70)
FROM pg_stat_statements
WHERE mean_plan_time > mean_exec_time AND calls > 1000
ORDER BY mean_plan_time * calls DESC LIMIT 10;
-- ★ plan_ratio > 1 means you are paying more to plan than to execute.
--   Prepared statements, or fewer relations.
```

**PROVE IT #8 — the denormalisation audit.**
```sql
-- for every column that looks like a copy, is there a detector?
SELECT 'events.tenant_name' AS col,
       count(*) FILTER (WHERE n>1) AS drifted_entities
FROM (SELECT tenant_id, count(DISTINCT tenant_name) n FROM events GROUP BY 1) t;
-- ★ if this query does not exist and run on a schedule, the column is
--   not denormalised — it is unnormalised.
```

---

## The design decision framework

```
★★★ THE RULE: NORMALISE TO 3NF. STOP UNLESS YOU CAN NAME THE ANOMALY. ★★★

 THEN RUN THE HIGHER FORMS AS FIVE-MINUTE CHECKS, NOT PROJECTS:
   BCNF: overlapping candidate keys?     no → done
   4NF:  two independent lists in one table? no → done
   5NF:  a 3-column relationship table with no extra columns? no → done

 FOR ANY FURTHER SPLIT, THE FOUR QUESTIONS:
   ① What anomaly does it prevent?        (name it, or stop)
   ② What constraint becomes enforceable? (name it, or stop)
   ③ joins_added × call_rate × 0.05 ms = ? (usually negligible)
   ④ Will a human understand the schema?  (a real criterion)
   ⇒ ① and ② both YES → split. Otherwise stop.

 THE CEILINGS TO WATCH:
   ~12 relations in one query → geqo_threshold; planning time and
                                non-deterministic plans (Topic 18)
   plan_time > exec_time      → prepared statements, or fewer tables
   a 2-column table with < 100 rows, no attributes, engineering-owned
                              → collapse to a CHECK

 ★ DELIBERATE DENORMALISATION REQUIRES ALL FOUR:
   ① you normalised first and can NAME the violated FD
   ② a MEASUREMENT (EXPLAIN, p99, call rate) showing the need
   ③ a MECHANISM: trigger · generated column · composite FK ·
      or explicit snapshot semantics with a current-value view
   ④ a DETECTOR query for drift, running on a schedule with an alert
   ⇒ ★ missing any one → you are UNNORMALISED, not denormalised.
   ⇒ THE CODE-REVIEW TEST: "show me the drift detector."

 ★ NEVER "NORMALISE AWAY" A SNAPSHOT.
   Test: "if the source changes, SHOULD this row's value change?"
     no → it is a snapshot. Keep it, rename it to say so.
   Removing one is data loss wearing the costume of rigour.

THE SIGNAL TO LOOK FOR:
 UNDER-normalised:
   SELECT <id>, count(DISTINCT <descriptive_col>) FROM t
   GROUP BY 1 HAVING count(DISTINCT <descriptive_col>) > 1;
   ⇒ any row → an update anomaly has ALREADY happened.

 OVER-normalised:
   • two-column tables with < 100 rows and no attributes
   • plan_time > exec_time in pg_stat_statements
   • a query joining > 12 relations
   • a table whose name maps to no business concept
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
Build the 6-table normalised schema from Example 1. Measure planning and execution time for a 2-table, 4-table and 6-table join of the same logical query. Then build the flat equivalent and measure both a point query and a full scan on each. Explain why the normalised version loses on one and wins on the other.

### Exercise 2 — medium (apply it)
For each, decide keep-as-lookup-table or collapse-to-CHECK, and justify with the four questions:
(a) `countries(code, name)` — 195 rows, referenced by addresses · (b) `order_statuses(id, code)` — 5 rows, engineering-owned, no attributes · (c) `subscription_plans(id, code, name, price, features)` — 4 rows, edited by product · (d) `http_methods(id, method)` — 7 rows · (e) `currencies(code, name, exponent)` — 8 rows, referenced by every amount

### Exercise 3 — hard (production simulation)
Two teams, two problems.

**Team A:** 34 tables, fully 3NF+, a 14-table hot query with planning at 11.2 ms vs execution at 2.8 ms, and non-deterministic plans.
**Team B:** 6 tables, "denormalised for performance," with 21% of tenants, 5% of users and 10% of products showing drift.

(a) For Team A, write the query that finds collapse candidates, and apply the four questions to each of the six lookup tables.
(b) Explain why planning time rather than join execution is Team A's problem, and name the specific threshold.
(c) Give Team A's migration using Topic 28's techniques, and predict the p99 improvement.
(d) For Team B, apply the four-condition test and show that zero conditions are met.
(e) Write the drift detector and interpret its output.
(f) **Team B's schema turns out to be defensible.** Explain why, and describe what must change instead of the tables.
(g) Team B's `tenant_name` is ambiguous — is it a snapshot or a stale copy? Design the resolution so that both "name at event time" and "current name" are answerable.
(h) Write the code-review checklist that would have prevented both teams' problems.

---

## Mental model checkpoint

1. State the stopping rule in one sentence. Why 3NF specifically, and not BCNF?
2. Give the four questions you ask before any further decomposition. Which two must both be yes?
3. What does an indexed join actually cost? What is the real ceiling in a many-table query?
4. Give two arguments *for* normalisation that are about performance rather than correctness.
5. Name the four conditions that distinguish deliberate denormalisation from an unnormalised schema.
6. When should a fixed set of values be a lookup table, and when a `CHECK` constraint?
7. What is the one thing you must never "normalise away," and what's the test?

---

## Quick reference card

**★ The rule:** normalise to **3NF**; run BCNF/4NF/5NF as five-minute *checks*; stop unless you can **name the anomaly**.

**The four questions before any further split**

| # | Question | If no |
|---|---|---|
| ① | What anomaly does it prevent? | **stop** |
| ② | What constraint becomes enforceable? | **stop** |
| ③ | joins × call rate × 0.05 ms? | usually fine |
| ④ | Will a human understand it? | reconsider |

**Costs**

| | |
|---|---|
| Indexed nested-loop join | ~0.05 ms, ~4 buffers |
| **12 relations** | `geqo_threshold` — planning time, non-deterministic plans |
| Narrower rows from normalising | often saves more on scans than joins cost |
| 2-column lookup, < 100 rows, no attributes | collapse to `CHECK` |

**★ Deliberate denormalisation needs all four**

1. You normalised first and can **name the violated FD**
2. A **measurement**
3. A **mechanism** (trigger / generated column / composite FK / explicit snapshot semantics)
4. A **drift detector** on a schedule

**Missing any one ⇒ unnormalised, not denormalised.**
**Code-review test:** *"show me the drift detector."*

**Never normalise away a snapshot.** Test: *if the source changes, should this change?* No → keep it.

---

## When would I use this at work?

1. **A design review that has become an argument.** "Normalise to 3NF, then name the anomaly" ends it, because it converts a taste debate into a specific, answerable question.

2. **A slow endpoint joining a dozen tables.** Checking `plan_time` vs `exec_time` and counting relations against `geqo_threshold` finds the real problem — which is usually four meaningless lookup tables, not the data model.

3. **Any PR touching a denormalised column.** Asking for the drift detector takes ten seconds and reliably separates a considered decision from an accident that will corrupt data.

---

## Connected topics

**Understand before this:** 29–37 (the whole of Phase 4), 19 (what joins actually cost), 18 (planning time and GEQO).

**This unlocks — Phase 6, which starts here:**
- **53** — what denormalisation is, formally
- **54** — when to denormalise: the measured signals
- **55** — the denormalisation patterns, each with its consistency mechanism
- **56** — materialised views, the mechanism that often removes the need
- **Case studies 03, 04, 05** — snapshots, temporal facts, and measured denormalisation in production
