# 55 — Denormalisation Patterns
## Phase: Denormalisation & Scale

---

## ELI5 — The Simple Analogy

A library with a card catalogue. Four different shortcuts people take, each with a different failure mode.

**Copy the author's name onto every book card.** Fast to read a shelf. But when an author changes their name, you must find every card. — *the copied column*

**Write "42 books" on the shelf label.** No counting needed. But every time a book is added or removed, someone must update the label — and if two people add books at once, they can both read "42" and both write "43." — *the maintained aggregate*

**Write the full path on each card:** *Science → Physics → Quantum.* Instant to display, instant to search "everything under Science." But move the Physics section and you must rewrite every card beneath it. — *the materialised path*

**Staple the table of contents onto the book cover.** Everything about the book in one place, one lookup. But now you cannot ask "which books contain a chapter about entropy?" without opening every cover. — *the embedded document*

★ **Each buys a different read, and each has a completely different cost.** The mistake is treating "denormalisation" as one technique. It is four, and choosing the wrong one for your access pattern costs you more than not denormalising at all.

---

## Where this fits in the big picture

```
   53 what it is (the obligation) · 54 when (the five gates)
                          │
                          ▼
        ┌──────────────────────────────────────────────┐
        │ 55 THE PATTERNS ← YOU ARE HERE               │
        │ ★ four shapes, four failure modes            │
        └────────────────────┬─────────────────────────┘
                             ▼
              56 materialised views (async, engine-maintained)
              57 caching · 59 partitioning · 61 hot rows
```

You have decided **yes**. This topic is **which shape**, and how to build each one correctly.

---

## What is this?

Four patterns, distinguished by **what is being copied** and therefore **what invalidates it**:

| # | Pattern | Copies | Invalidated by | Cost |
|---|---|---|---|---|
| **1** | **Copied column** | one attribute of a parent | a parent update | ★ **fan-out** |
| **2** | **Maintained aggregate** | a `count`/`sum` over children | any child change | ★ **contention** |
| **3** | **Materialised path/tree** | ancestry of a node | a subtree move | ★ **subtree rewrite** |
| **4** | **Embedded document** | a whole child collection | any child change | ★ **loss of queryability** |

Plus two that people *call* denormalisation and aren't:

```
 ✗ A GENERATED COLUMN — a derivation of the SAME row.
   ★ Cannot drift. Not a copy. Use it whenever it fits.
 ✗ A COVERING INDEX (INCLUDE) — ★ denormalisation the engine
   maintains atomically. Zero obligation. (Topic 12.)
   ⇒ ★ TRY BOTH OF THESE BEFORE ANY PATTERN BELOW.
```

---

## Why does it matter for a backend developer?

```
 ★ BECAUSE THE FOUR PATTERNS FAIL IN COMPLETELY DIFFERENT WAYS,
   AND THE FIX FOR ONE MAKES ANOTHER WORSE.

 ① COPIED COLUMN fails on FAN-OUT
    one parent rename → 8,402 child rows rewritten
    ⇒ ★ the fix is: don't. Use an index, or accept the join.

 ② MAINTAINED AGGREGATE fails on CONTENTION
    one popular parent → every child write serialises on one row
    ⇒ ★ the fix is SHARDING THE COUNTER, which makes reads harder

 ③ MATERIALISED PATH fails on RESTRUCTURING
    move one node → rewrite every descendant
    ⇒ ★ the fix is choosing a different tree encoding entirely

 ④ EMBEDDED DOCUMENT fails on QUERYABILITY
    you can no longer ask questions about the children
    ⇒ ★ and this failure is DISCOVERED LATE, when a new
      requirement arrives that the shape cannot answer

 ⇒ ★ CHOOSING THE PATTERN IS THE DECISION. Not "should we
   denormalise" — that was Topic 54. This is "which shape survives
   the access patterns we will actually have."
```

---

## The physical reality

### Pattern 1 — the copied column

```
 SHAPE:   orders.customer_name  ← copied from customers.name

 ★ THE PHYSICS: a parent update triggers N child updates, where N
   is the fan-out. Each child update is a FULL ROW REWRITE
   (Topic 46), plus every index on the child table.

   1 parent row  →  N child tuples  ×  (1 + index_count) pages

 MEASURED (orders, 4 indexes, customer with 8,402 orders):
   normalised   1 tuple,    232 B WAL,     0.4 ms
   denormalised 8,403 tuples, 14 MB WAL, ★ 4,100 ms

 ★ WHEN IT IS ACTUALLY RIGHT — three cases only:
   ① ★ THE PARENT IS IMMUTABLE (a country code, a currency,
      a plan name that never changes)
      ⇒ fan-out × 0 changes = 0 cost. ★ FREE.
   ② ★ THE JOIN CROSSES A SERVICE BOUNDARY (you cannot JOIN)
   ③ ★ IT IS ACTUALLY A SNAPSHOT (Topic 54's gate 3)

 ★ THE ANTI-PATTERN: copying a MUTABLE attribute of a parent with
   HIGH fan-out, "to avoid a join." This is the single most common
   bad denormalisation in production systems.
```

### Pattern 2 — the maintained aggregate

```
 SHAPE:   products.review_count / review_sum ← count/sum of reviews

 ★ THE PHYSICS: every child INSERT/UPDATE/DELETE takes a row lock
   on ONE parent row for the rest of the transaction.
   ⇒ throughput ceiling ≈ 1 / (lock hold time)
   ⇒ ★ measured: ~800 writes/sec practical on one row

 ★ THE FOUR RULES THAT MAKE IT CORRECT:
   ① STORE SUM AND COUNT, DERIVE THE AVERAGE
      ⇒ an average cannot be updated incrementally on DELETE, and
        averages of averages are wrong.
      ⇒ ★ avg as a GENERATED column ⇒ it cannot drift.
   ② HANDLE ALL FOUR TRANSITIONS
      INSERT · DELETE · UPDATE of the value · ★ UPDATE of the FK
      ⇒ ★ the changed-parent case is the one people forget.
   ③ ★ PUT IT IN A NARROW SIDE TABLE IF THE PARENT IS WIDE OR
      HEAVILY INDEXED
      parent with 7 indexes: 31 GB/hr WAL
      narrow side table, fillfactor 70: ★ 5.2 GB/hr (Topic 54)
   ④ ★ CONSTRAIN IT
      CHECK (count >= 0), CHECK (sum BETWEEN 0 AND count*max)
      ⇒ catches drift at write time, not in a nightly report.

 ★ WHEN IT FAILS: a hot parent.
   4,000 writes/sec on one product ⇒ every writer queues.
   ⇒ ★ SHARD IT (Topic 61) or go ASYNC (Topic 56).
```

### Pattern 3 — the materialised path and its alternatives

```
 ★ FOUR WAYS TO STORE A TREE. THE CHOICE IS ENTIRELY ABOUT WHICH
   OPERATION MUST BE FAST.

 ① ADJACENCY LIST — parent_id
    ✓ ★ normalised, no obligation, moves are O(1)
    ✓ PostgreSQL's RECURSIVE CTE handles it well
    ✗ "all descendants" is a recursive query
    MEASURED (depth 6, 1M nodes): subtree query 41 ms

 ② ★ MATERIALISED PATH — path text/array: 'Science.Physics.Quantum'
    ✓ ★ subtree query is ONE index range scan: path LIKE 'Science.%'
    ✓ ancestors are free (split the string)
    ✓ ★ ltree gives GiST indexing and proper operators
    ✗ ★ MOVING A NODE REWRITES EVERY DESCENDANT
    MEASURED: subtree query 0.4 ms (★ 102×), move-with-8k-descendants
              ★ 340 ms
    ⇒ ★ THE RIGHT CHOICE WHEN TREES ARE READ CONSTANTLY AND
      RESTRUCTURED RARELY — which is most category trees, org
      charts and file hierarchies.

 ③ NESTED SETS — lft/rgt integers
    ✓ subtree query is one range scan
    ✗ ★ ANY INSERT REWRITES HALF THE TABLE (all lft/rgt > the point)
    ⇒ ★ almost never correct in a system with concurrent writes.
      It is a read-only-tree structure.

 ④ CLOSURE TABLE — a row per (ancestor, descendant) pair
    ✓ ★ every query is a simple join; both directions fast
    ✓ moves touch only the affected pairs
    ✗ ★ O(depth) rows per node — a depth-8 tree of 1M nodes is
      ~8M closure rows
    ⇒ ★ the right choice for deep trees with frequent moves.

 ⇒ ★ THE DECISION:
   reads only, shallow            ⇒ adjacency list + RECURSIVE CTE
   read-heavy, rare moves         ⇒ ★ materialised path (ltree)
   frequent moves, deep           ⇒ ★ closure table
   never                          ⇒ nested sets
```

### Pattern 4 — the embedded document

```
 SHAPE:   orders.items jsonb  ← the whole line-item collection

 ★ THE PHYSICS:
   • the whole document is ONE value ⇒ ★ any change rewrites ALL
     of it (Topic 46: the whole row is copied anyway)
   • over ~2 KB it TOASTs ⇒ ★ a separate fetch per row (Topic 04)
   • a GIN index makes containment queries work, but ★ costs
     3–10× the size of a B-tree and slows writes substantially

 ★ WHAT YOU LOSE — and this is the real cost:
   ✗ foreign keys to products     ⇒ ★ orphan SKUs, silently
   ✗ CHECK constraints per item   ⇒ ★ qty = -3 is representable
   ✗ "which orders contain SKU X" ⇒ needs GIN, still slower
   ✗ "top 10 products by units"   ⇒ ★ requires unnesting everything
   ✗ per-item updates             ⇒ rewrite the whole document
   ⇒ ★ AND THE LOSS IS DISCOVERED LATE, when a new requirement
     arrives that the shape cannot answer.

 ★ WHEN IT IS RIGHT — three genuine cases:
   ① ★ THE CHILDREN ARE NEVER QUERIED INDEPENDENTLY
      an audit record's "before/after" blob; a webhook payload
   ② ★ THE SHAPE IS GENUINELY VARIABLE
      product attributes across 400 categories
      ⇒ ★ the alternative is EAV, which is worse (Topic 27)
   ③ ★ IT IS AN IMMUTABLE SNAPSHOT
      the order as it was confirmed, for legal/audit purposes
      ⇒ ★ stored ALONGSIDE normalised tables, not instead of them

 ★ THE HYBRID THAT USUALLY WINS:
   normalised order_items (queryable, constrained, FK'd)
   + orders.items_snapshot jsonb (immutable, for reprinting the
     original confirmation)
   ⇒ ★ you get both, and the jsonb has no obligation because it
     is a snapshot.
```

---

## How it works — step by step

### Choosing the pattern from the access pattern

```
 ★ START FROM THE READ YOU NEED, NOT FROM THE SHAPE YOU LIKE.

 "show a parent attribute alongside each child row"
   ⇒ ★ FIRST: an index that reduces the row count. Then the join
     is free (Topic 54).
   ⇒ if it truly crosses a service boundary ⇒ COPIED COLUMN
   ⇒ if the parent is immutable ⇒ COPIED COLUMN (free)
   ⇒ otherwise ⇒ ★ DON'T. Fan-out will kill you.

 "show a count/sum of children on a list of parents"
   ⇒ ★ MAINTAINED AGGREGATE — sum+count, generated average,
     narrow side table if the parent is wide
   ⇒ check the hottest-parent write rate first (Topic 61)

 "show all descendants / all ancestors of a node"
   ⇒ reads ≫ moves      ⇒ ★ MATERIALISED PATH (ltree)
   ⇒ moves are frequent ⇒ ★ CLOSURE TABLE
   ⇒ shallow and rare   ⇒ adjacency list + RECURSIVE CTE

 "fetch a parent and all its children in one round trip"
   ⇒ ★ NOT A DENORMALISATION PROBLEM. Use ONE query with a
     LATERAL join or json_agg. (Topic 66.)
   ⇒ embed ONLY if the children are never queried independently

 "aggregate over a large table for a dashboard"
   ⇒ ★ MATERIALISED VIEW (Topic 56), not any of these.
```

### The four transitions a maintained aggregate must handle

```sql
CREATE OR REPLACE FUNCTION sync_counts() RETURNS trigger AS $$
BEGIN
  IF TG_OP = 'INSERT' THEN
    -- ① new child
    UPDATE parents SET n = n + 1, s = s + NEW.value WHERE id = NEW.parent_id;

  ELSIF TG_OP = 'DELETE' THEN
    -- ② removed child
    UPDATE parents SET n = n - 1, s = s - OLD.value WHERE id = OLD.parent_id;

  ELSIF TG_OP = 'UPDATE' THEN
    IF NEW.parent_id IS DISTINCT FROM OLD.parent_id THEN
      -- ★ ③ THE ONE PEOPLE FORGET: the child moved to another parent
      UPDATE parents SET n = n - 1, s = s - OLD.value WHERE id = OLD.parent_id;
      UPDATE parents SET n = n + 1, s = s + NEW.value WHERE id = NEW.parent_id;
    ELSIF NEW.value IS DISTINCT FROM OLD.value THEN
      -- ④ the value changed in place
      UPDATE parents SET s = s - OLD.value + NEW.value WHERE id = NEW.parent_id;
    END IF;
  END IF;
  RETURN NULL;                    -- ★ AFTER trigger: return value ignored
END $$ LANGUAGE plpgsql;

CREATE TRIGGER trg_sync_counts
  AFTER INSERT OR UPDATE OR DELETE ON children
  FOR EACH ROW EXECUTE FUNCTION sync_counts();
```

```
 ★ AND THE LOCK-ORDER TRAP IN CASE ③:
   updating two parent rows in the order (OLD, NEW) is
   NON-DETERMINISTIC across transactions ⇒ ★ DEADLOCK (Topic 48).
 ⇒ ★ FIX: always touch them in ascending id order.
```

```sql
    IF NEW.parent_id IS DISTINCT FROM OLD.parent_id THEN
      -- ★ deterministic order prevents the deadlock
      PERFORM 1 FROM parents WHERE id IN (OLD.parent_id, NEW.parent_id)
        ORDER BY id FOR NO KEY UPDATE;
      UPDATE parents SET n=n-1, s=s-OLD.value WHERE id=OLD.parent_id;
      UPDATE parents SET n=n+1, s=s+NEW.value WHERE id=NEW.parent_id;
    END IF;
```

---

## Concept breakdown

```
★ FOUR PATTERNS, FOUR FAILURE MODES
├── ① COPIED COLUMN        → fails on ★ FAN-OUT
│     right only when: ★ the parent is IMMUTABLE · the join crosses
│     a SERVICE BOUNDARY · it's a SNAPSHOT
├── ② MAINTAINED AGGREGATE → fails on ★ CONTENTION
│     ★ sum+count, generated avg · all FOUR transitions ·
│     ★ narrow side table if the parent is wide · CHECK constraints
├── ③ MATERIALISED PATH    → fails on ★ RESTRUCTURING
│     ★ one of FOUR tree encodings; pick by read/move ratio
└── ④ EMBEDDED DOCUMENT    → fails on ★ QUERYABILITY
      ★ and the loss is discovered LATE

★ NOT DENORMALISATION — TRY THESE FIRST
├── GENERATED column   — a same-row derivation. ★ Cannot drift.
└── COVERING INDEX     — ★ the engine maintains it atomically

★ THE TREE DECISION
   adjacency list   moves O(1), subtree = RECURSIVE CTE   41 ms
   ★ materialised path  subtree = 1 range scan  0.4 ms; ★ move = 340 ms
   nested sets      ★ any insert rewrites half the table ⇒ no
   ★ closure table  both directions fast; ★ O(depth) rows per node
   ⇒ read-heavy + rare moves ⇒ path.  Frequent moves ⇒ closure.

★ THE EMBEDDED-DOCUMENT LOSSES
   ✗ FKs  ✗ per-item CHECKs  ✗ "which orders contain SKU X"
   ✗ "top products by units"  ✗ per-item updates
   ⇒ ★ THE HYBRID USUALLY WINS: normalised children FOR QUERIES
     + an immutable jsonb snapshot FOR REPRINTING

★ THE FOUR TRANSITIONS AN AGGREGATE TRIGGER MUST HANDLE
   INSERT · DELETE · value UPDATE · ★ FK UPDATE (the forgotten one)
   ⇒ ★ and the FK-update case needs ASCENDING-ID LOCK ORDER or it
     deadlocks (Topic 48)
```

---

## Diagrams

**Diagram 1 — big picture: what each pattern buys and what it costs**

```
 ① COPIED COLUMN                    ② MAINTAINED AGGREGATE
 ┌──────────────────────────┐       ┌──────────────────────────┐
 │ customers(id, name)      │       │ products(id, review_count│
 │        │                 │       │          , review_sum)   │
 │        │ copy            │       │        ▲                 │
 │        ▼                 │       │        │ +1 per insert   │
 │ orders(…, customer_name) │       │ reviews(product_id, …)   │
 │                          │       │                          │
 │ READ  ★ no join          │       │ READ  ★ O(1), no scan    │
 │ WRITE ★ 1 → N rows       │       │ WRITE ★ 1 row lock,      │
 │       FAN-OUT KILLS IT   │       │       CONTENTION KILLS IT│
 └──────────────────────────┘       └──────────────────────────┘

 ③ MATERIALISED PATH                ④ EMBEDDED DOCUMENT
 ┌──────────────────────────┐       ┌──────────────────────────┐
 │ categories(id, path)     │       │ orders(id, items jsonb)  │
 │  1  'electronics'        │       │  8801  [{sku, qty, …},   │
 │  2  'electronics.phones' │       │         {sku, qty, …}]   │
 │  3  'electronics.phones. │       │                          │
 │      android'            │       │ READ  ★ 1 row, 1 fetch   │
 │                          │       │ WRITE rewrite ALL of it  │
 │ READ  ★ path <@ 'elec'   │       │ ★ LOSE: FKs, CHECKs,     │
 │       1 index scan       │       │   "which orders have     │
 │ WRITE ★ move → rewrite   │       │   SKU X", per-item edits │
 │       EVERY DESCENDANT   │       │   QUERYABILITY KILLS IT  │
 └──────────────────────────┘       └──────────────────────────┘
```

**Diagram 2 — data flow: the four tree encodings on one operation**

```
 THE TREE:              electronics
                        ├── phones
                        │   ├── android      ← MOVE THIS
                        │   └── ios              under `mobile`
                        └── laptops
                            └── mobile

 QUERY: "all descendants of electronics"   MOVE: android → mobile
 ─────────────────────────────────────     ──────────────────────────
 ① ADJACENCY LIST (parent_id)
    WITH RECURSIVE t AS (…)                UPDATE SET parent_id=…
    ★ 41 ms, depth-6, 1M nodes             ★ 1 row. 0.3 ms.

 ② ★ MATERIALISED PATH (ltree)
    WHERE path <@ 'electronics'            UPDATE path = 'electronics.
    ★ 0.4 ms — ONE GiST range scan           laptops.mobile' || subpath(…)
    ★ 102× FASTER                           WHERE path <@ 'electronics.
                                             phones.android'
                                           ★ 340 ms for 8k descendants

 ③ NESTED SETS (lft, rgt)
    WHERE lft BETWEEN 1 AND 20             ★ RECOMPUTE lft/rgt FOR
    0.3 ms                                   ~HALF THE TABLE
                                           ★ 8,400 ms. Unusable
                                             concurrently.

 ④ ★ CLOSURE TABLE (ancestor, descendant, depth)
    JOIN closure c ON c.ancestor=1         DELETE old pairs,
    ★ 0.6 ms — a plain join                INSERT new pairs
                                           ★ 12 ms. Both directions
                                             fast, moves cheap.
                                           ✗ 8M rows for 1M nodes

 ⇒ ★ THERE IS NO BEST. THERE IS ONLY "WHICH OPERATION MUST BE
   FAST", AND THE READ:MOVE RATIO DECIDES.
```

**Diagram 3 — before/after: the hybrid that beats embedding**

```
 ✗ FULLY EMBEDDED — one jsonb column
 ┌───────────────────────────────────────────────────────────────┐
 │ orders(id, customer_id, items jsonb)                          │
 │   items = [{"sku":"KUR-001","qty":2,"price_minor":49900}, …]  │
 │                                                                │
 │ ✓ "show me order 8801"          ★ 1 row, 0.1 ms               │
 │ ✗ "which orders contain KUR-001"                              │
 │     ⇒ GIN index, 0.8 ms — but ★ the GIN index is 4.2 GB       │
 │       vs 380 MB for a B-tree on order_items(sku)              │
 │ ✗ "top 10 SKUs by units this month"                           │
 │     ⇒ ★ jsonb_array_elements over 8.8M orders = 41 s          │
 │ ✗ FK to products                ★ IMPOSSIBLE — orphan SKUs    │
 │ ✗ CHECK (qty > 0)               ★ IMPOSSIBLE — qty:-3 is legal│
 │ ✗ "change qty on one line"      ★ rewrites the whole document │
 └───────────────────────────────────────────────────────────────┘

 ✓ HYBRID — normalised for queries, jsonb for the snapshot
 ┌───────────────────────────────────────────────────────────────┐
 │ order_items(id, order_id FK, sku FK, qty CHECK(qty>0),        │
 │             unit_price_charged_minor,                          │
 │             line_total_minor GENERATED)                        │
 │   ⇒ ★ FKs enforced · CHECKs enforced · fully queryable        │
 │   ⇒ "top 10 SKUs": ★ 0.4 s with an index                      │
 │                                                                │
 │ orders.confirmation_snapshot jsonb                             │
 │   ⇒ ★ the order EXACTLY as confirmed: prices, taxes, address,  │
 │     terms text, at that moment                                 │
 │   ⇒ ★ IT IS A SNAPSHOT (Topic 54 gate 3) ⇒ NO OBLIGATION,     │
 │     no trigger, no reconciler                                  │
 │   ⇒ used ONLY for reprinting the original confirmation         │
 │                                                                │
 │ ★ YOU GET BOTH, AND THE JSONB COSTS NOTHING TO MAINTAIN.      │
 └───────────────────────────────────────────────────────────────┘
```

---

## Example 1 — basic

**Pattern 1 — the copied column, and when it's free.**

```sql
CREATE TABLE countries (code char(2) PRIMARY KEY, name text NOT NULL);
INSERT INTO countries VALUES ('IN','India'),('US','United States');

CREATE TABLE addresses (
  id bigserial PRIMARY KEY,
  country_code char(2) NOT NULL REFERENCES countries(code),
  country_name text NOT NULL,          -- ★ copied
  city text NOT NULL
);
COMMENT ON COLUMN addresses.country_name IS
  'Copied from countries.name. ★ SAFE: country names change roughly
   never (fan-out × ~0 changes = ~0 cost). No trigger; a one-off
   migration handles the rare rename. Reconciler: ops/reconcile_country.sql
   (weekly).';
```
```sql
-- the reconciler is still cheap to run
SELECT count(*) FROM addresses a JOIN countries c ON c.code = a.country_code
 WHERE a.country_name IS DISTINCT FROM c.name;
```
```
 count
-------
     0
```

**And the version that is not free.**
```sql
CREATE TABLE customers (id bigint PRIMARY KEY, name text NOT NULL);
CREATE TABLE orders (
  id bigserial PRIMARY KEY,
  customer_id bigint NOT NULL REFERENCES customers(id),
  customer_name text,                  -- ★ the dangerous one
  total_minor bigint NOT NULL
);
INSERT INTO customers SELECT g,'C'||g FROM generate_series(1,10000) g;
INSERT INTO orders (customer_id, customer_name, total_minor)
SELECT 42, 'C42', 1000 FROM generate_series(1,8402);
INSERT INTO orders (customer_id, customer_name, total_minor)
SELECT (random()*9999+1)::bigint, 'x', 1000 FROM generate_series(1,500000);
CREATE INDEX ON orders (customer_id);
CREATE INDEX ON orders (total_minor);
CREATE INDEX ON orders (customer_name);
VACUUM ANALYZE orders;

\timing on
SELECT pg_current_wal_lsn() AS a \gset
UPDATE customers SET name = 'Meera Sharma' WHERE id = 42;
UPDATE orders SET customer_name = 'Meera Sharma' WHERE customer_id = 42;
SELECT pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), :'a')) AS wal;
```
```
 Time: 0.412 ms
 Time: ★ 3,884.221 ms
    wal
-----------
 ★ 18 MB          — for renaming one customer
```

**Pattern 2 — the maintained aggregate, done properly.**

```sql
CREATE TABLE products (
  id bigserial PRIMARY KEY,
  name text NOT NULL,
  review_count integer NOT NULL DEFAULT 0,
  review_sum   bigint  NOT NULL DEFAULT 0,
  avg_rating numeric(2,1)
    GENERATED ALWAYS AS (                      -- ★ cannot drift
      CASE WHEN review_count = 0 THEN NULL
           ELSE round(review_sum::numeric / review_count, 1) END) STORED,
  CONSTRAINT ck_rc CHECK (review_count >= 0),
  CONSTRAINT ck_rs CHECK (review_sum BETWEEN 0 AND review_count * 5)
);
CREATE TABLE reviews (
  id bigserial PRIMARY KEY,
  product_id bigint NOT NULL REFERENCES products(id),
  stars smallint NOT NULL CHECK (stars BETWEEN 1 AND 5)
);
```
```sql
CREATE OR REPLACE FUNCTION sync_review_stats() RETURNS trigger AS $$
BEGIN
  IF TG_OP = 'INSERT' THEN
    UPDATE products SET review_count = review_count + 1,
                        review_sum   = review_sum + NEW.stars
     WHERE id = NEW.product_id;
  ELSIF TG_OP = 'DELETE' THEN
    UPDATE products SET review_count = review_count - 1,
                        review_sum   = review_sum - OLD.stars
     WHERE id = OLD.product_id;
  ELSIF NEW.product_id IS DISTINCT FROM OLD.product_id THEN
    -- ★ the forgotten transition, with ★ deterministic lock order
    PERFORM 1 FROM products WHERE id IN (OLD.product_id, NEW.product_id)
      ORDER BY id FOR NO KEY UPDATE;
    UPDATE products SET review_count=review_count-1, review_sum=review_sum-OLD.stars
     WHERE id = OLD.product_id;
    UPDATE products SET review_count=review_count+1, review_sum=review_sum+NEW.stars
     WHERE id = NEW.product_id;
  ELSIF NEW.stars IS DISTINCT FROM OLD.stars THEN
    UPDATE products SET review_sum = review_sum - OLD.stars + NEW.stars
     WHERE id = NEW.product_id;
  END IF;
  RETURN NULL;
END $$ LANGUAGE plpgsql;

CREATE TRIGGER trg_review_stats
  AFTER INSERT OR UPDATE OR DELETE ON reviews
  FOR EACH ROW EXECUTE FUNCTION sync_review_stats();
```
```sql
INSERT INTO products (name) VALUES ('Cotton Kurta'), ('Silk Saree');
INSERT INTO reviews (product_id, stars) VALUES (1,5),(1,4),(1,3);
SELECT id, name, review_count, review_sum, avg_rating FROM products;
```
```
 id |     name     | review_count | review_sum | avg_rating
----+--------------+--------------+------------+------------
  1 | Cotton Kurta |            3 |         12 |      ★ 4.0
  2 | Silk Saree   |            0 |          0 |
```
```sql
-- all four transitions
UPDATE reviews SET stars = 1 WHERE id = 1;              -- value change
SELECT review_count, review_sum, avg_rating FROM products WHERE id=1;
```
```
 review_count | review_sum | avg_rating
--------------+------------+------------
            3 |          8 |      ★ 2.7
```
```sql
UPDATE reviews SET product_id = 2 WHERE id = 1;         -- ★ FK change
SELECT id, review_count, review_sum FROM products ORDER BY id;
```
```
 id | review_count | review_sum
----+--------------+------------
  1 |            2 |          7
  2 |          ★ 1 |        ★ 1
```
```sql
DELETE FROM reviews WHERE id = 2;                       -- delete
SELECT id, review_count, review_sum FROM products ORDER BY id;
```
```
 id | review_count | review_sum
----+--------------+------------
  1 |            1 |          3
  2 |            1 |          1
```

**Prove the constraint catches drift at write time.**
```sql
UPDATE products SET review_sum = 999 WHERE id = 1;
```
```
ERROR:  new row for relation "products" violates check constraint "ck_rs"
DETAIL:  Failing row contains (1, Cotton Kurta, 1, 999, …).
   ★ an absurd value is rejected immediately, not found in a report.
```

**Pattern 3 — the four tree encodings, compared.**

```sql
CREATE EXTENSION IF NOT EXISTS ltree;

-- ① adjacency list
CREATE TABLE cat_adj (id bigserial PRIMARY KEY, parent_id bigint
  REFERENCES cat_adj(id), name text NOT NULL);

-- ② ★ materialised path
CREATE TABLE cat_path (id bigserial PRIMARY KEY, path ltree NOT NULL,
  name text NOT NULL);
CREATE INDEX idx_cat_path_gist ON cat_path USING gist (path);
CREATE INDEX idx_cat_path_btree ON cat_path USING btree (path);

-- ④ ★ closure table
CREATE TABLE cat_node (id bigserial PRIMARY KEY, name text NOT NULL);
CREATE TABLE cat_closure (
  ancestor_id bigint NOT NULL REFERENCES cat_node(id),
  descendant_id bigint NOT NULL REFERENCES cat_node(id),
  depth int NOT NULL,
  PRIMARY KEY (ancestor_id, descendant_id)
);
CREATE INDEX ON cat_closure (descendant_id, depth);
```

```sql
INSERT INTO cat_path (path, name) VALUES
  ('electronics','Electronics'),
  ('electronics.phones','Phones'),
  ('electronics.phones.android','Android'),
  ('electronics.phones.ios','iOS'),
  ('electronics.laptops','Laptops');

-- ★ subtree: one index scan
SELECT id, name, path FROM cat_path WHERE path <@ 'electronics.phones';
```
```
 id |  name   |            path
----+---------+-----------------------------
  2 | Phones  | electronics.phones
  3 | Android | electronics.phones.android
  4 | iOS     | electronics.phones.ios
```
```sql
-- ★ ancestors: also free
SELECT name FROM cat_path WHERE path @> 'electronics.phones.android'
 ORDER BY nlevel(path);
```
```
    name
-------------
 Electronics
 Phones
 Android
```
```sql
-- ★ the move — this is the cost
BEGIN;
UPDATE cat_path
   SET path = 'electronics.laptops' || subpath(path, nlevel('electronics.phones')-1)
 WHERE path <@ 'electronics.phones';
SELECT path FROM cat_path ORDER BY path;
ROLLBACK;
```
```
              path
---------------------------------
 electronics
 electronics.laptops
 electronics.laptops.phones
 electronics.laptops.phones.android
 electronics.laptops.phones.ios
   ★ correct — but EVERY DESCENDANT ROW WAS REWRITTEN.
```

**Compare against the recursive CTE.**
```sql
WITH RECURSIVE sub AS (
  SELECT id, parent_id, name, 1 AS depth FROM cat_adj WHERE id = 2
  UNION ALL
  SELECT c.id, c.parent_id, c.name, s.depth+1
    FROM cat_adj c JOIN sub s ON c.parent_id = s.id
)
SELECT * FROM sub;
-- ★ correct, no obligation, but O(depth) round trips through
--   the recursion at 1M nodes: 41 ms vs 0.4 ms for ltree.
```

**Pattern 4 — the hybrid.**

```sql
CREATE TABLE order_items (
  id bigserial PRIMARY KEY,
  order_id bigint NOT NULL,
  sku text NOT NULL,
  qty int NOT NULL CHECK (qty > 0),                 -- ★ enforceable
  unit_price_charged_minor bigint NOT NULL,         -- ★ a snapshot
  line_total_minor bigint
    GENERATED ALWAYS AS (qty * unit_price_charged_minor) STORED
);
CREATE INDEX ON order_items (sku);
CREATE INDEX ON order_items (order_id);

ALTER TABLE orders ADD COLUMN confirmation_snapshot jsonb;
COMMENT ON COLUMN orders.confirmation_snapshot IS
  'Immutable record of the order exactly as confirmed (prices, tax
   rates, address, terms text). ★ A SNAPSHOT, not a copy — no
   trigger, no reconciler. Used only for reprinting the original
   confirmation. Queries go to order_items.';
```
```sql
-- the query the embedded version cannot answer cheaply
SELECT sku, sum(qty) AS units
  FROM order_items
 WHERE order_id IN (SELECT id FROM orders WHERE created_at > now()-interval '30 days')
 GROUP BY sku ORDER BY units DESC LIMIT 10;
-- ★ an index scan and a group. The jsonb version needs
--   jsonb_array_elements over every row.
```

---

## Example 2 — production scenario

**The situation.** A B2B procurement platform. The category tree drives navigation, permissions and reporting. 1.2 million categories, depth up to 9. Two complaints arrive the same week:

```
 ① "Category navigation takes 4 seconds."
      GET /api/categories/:id/tree     p99 ★ 4,180 ms
      called on every page load        ★ 2.1M times/day

 ② "Reorganising the taxonomy locks the whole system for 20 minutes."
      the quarterly restructure moves ~40,000 nodes
      ★ during which all category reads block
```

The current design: **nested sets** (`lft`, `rgt`), chosen three years ago because a blog post said it was fastest for reads.

**Step 1 — confirm the read cost.**

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, name, lft, rgt FROM categories
 WHERE lft > 88412 AND rgt < 104882
 ORDER BY lft;
```
```
 Index Scan using idx_categories_lft on categories  (actual time=0.041..2.884 rows=8,204)
   Index Cond: (lft > 88412)
   ★ Filter: (rgt < 104882)
   ★ Rows Removed by Filter: 411,208
 Execution Time: ★ 184.2 ms
```
```
 ★ THE READ ISN'T EVEN FAST. A range on lft can use the index; the
   rgt predicate cannot, so 411k rows are fetched and discarded.
 ⇒ nested sets' famous "fast subtree read" requires a composite
   index and still degrades with depth.
```

**Step 2 — the real problem is the write.**

```sql
-- moving one node with 8,204 descendants
EXPLAIN (ANALYZE)
UPDATE categories SET lft = lft + 16408, rgt = rgt + 16408
 WHERE lft >= 88412 AND rgt <= 104882;
```
```
 Update on categories  (actual time=8412.2..8412.2 rows=0)
   ->  Seq Scan on categories  (actual rows=8,204)
 Execution Time: 8,412.9 ms
```
```sql
-- but that's only the subtree. Every node to the RIGHT must shift:
EXPLAIN (ANALYZE)
UPDATE categories SET lft = lft - 16408 WHERE lft > 104882;
```
```
 Update on categories  (actual rows=★ 604,882)
 Execution Time: ★ 41,204.8 ms
```
```
 ★ MOVING ONE NODE REWROTE 613,086 ROWS — HALF THE TABLE.
 ★ AND IT HELD ROW LOCKS ON ALL OF THEM FOR 41 SECONDS.
 ⇒ 40,000 moves in a restructure ⇒ the 20-minute lockup.
 ⇒ ★ THIS IS NESTED SETS' DEFINING FLAW, and it is not tunable.
```

**Step 3 — choose the replacement from the actual read:write ratio.**

```sql
-- reads
SELECT calls FROM pg_stat_statements WHERE query ILIKE '%lft >%';
```
```
  calls
---------
 2,104,882 per day
```
```sql
-- moves
SELECT count(*) FROM category_audit
 WHERE action = 'move' AND changed_at > now() - interval '90 days';
```
```
 count
-------
 ★ 41,204 in 90 days = 458/day
```
```
 ★ READ:MOVE = 4,596 : 1

 ⇒ ★ MATERIALISED PATH (ltree). Reads are 4,600× more frequent
   than moves, so paying 340 ms per move to make reads 0.4 ms is
   overwhelmingly correct.
 ⇒ ★ A CLOSURE TABLE would also work and makes moves cheaper —
   but at depth 9 and 1.2M nodes it is ~9M closure rows, and the
   move cost of ltree (340 ms) is already acceptable at 458/day.
 ⇒ ★ THE DECIDING FACTOR WAS THE RATIO, NOT THE THEORY.
```

**Step 4 — migrate.**

```sql
CREATE EXTENSION IF NOT EXISTS ltree;
ALTER TABLE categories ADD COLUMN path ltree;

-- build the paths from the existing parent_id (kept alongside nested sets)
WITH RECURSIVE build AS (
  SELECT id, parent_id,
         (regexp_replace(lower(slug), '[^a-z0-9]+', '_', 'g'))::ltree AS path
    FROM categories WHERE parent_id IS NULL
  UNION ALL
  SELECT c.id, c.parent_id,
         b.path || (regexp_replace(lower(c.slug), '[^a-z0-9]+', '_', 'g'))::ltree
    FROM categories c JOIN build b ON c.parent_id = b.id
)
UPDATE categories c SET path = b.path FROM build b WHERE b.id = c.id;

ALTER TABLE categories ALTER COLUMN path SET NOT NULL;
CREATE INDEX CONCURRENTLY idx_categories_path_gist ON categories USING gist (path);
CREATE INDEX CONCURRENTLY idx_categories_path_btree ON categories USING btree (path);
-- ★ BOTH indexes: GiST for <@ / @> operators, B-tree for equality,
--   ordering and prefix ranges. They serve different queries.
```

```sql
-- ★ enforce that path and parent_id cannot disagree
CREATE OR REPLACE FUNCTION check_path_matches_parent() RETURNS trigger AS $$
DECLARE parent_path ltree;
BEGIN
  IF NEW.parent_id IS NULL THEN
    IF nlevel(NEW.path) <> 1 THEN
      RAISE EXCEPTION 'root node must have a single-level path';
    END IF;
  ELSE
    SELECT path INTO parent_path FROM categories WHERE id = NEW.parent_id;
    IF NOT (NEW.path ~ (parent_path::text || '.*{1}')::lquery) THEN
      RAISE EXCEPTION 'path % is not a direct child of %', NEW.path, parent_path;
    END IF;
  END IF;
  RETURN NEW;
END $$ LANGUAGE plpgsql;

CREATE TRIGGER trg_check_path
  BEFORE INSERT OR UPDATE OF path, parent_id ON categories
  FOR EACH ROW EXECUTE FUNCTION check_path_matches_parent();
-- ★ this is the reconciler, enforced at WRITE time. The two
--   representations cannot drift.
```

**Step 5 — the move operation.**

```js
async function moveCategory(nodeId, newParentId) {
  return withTransaction(async (tx) => {
    const { rows: [node]   } = await tx.query(
      'SELECT id, path FROM categories WHERE id=$1 FOR UPDATE', [nodeId]);
    const { rows: [parent] } = await tx.query(
      'SELECT id, path FROM categories WHERE id=$1 FOR UPDATE', [newParentId]);
    if (!node || !parent) throw new AppError('NOT_FOUND');

    // ★ guard against moving a node under its own descendant
    if (parent.path.startsWith(node.path + '.') || parent.path === node.path)
      throw new AppError('CYCLE');

    const oldDepth = node.path.split('.').length;

    // ★ ONE statement rewrites the whole subtree
    const { rowCount } = await tx.query(
      `UPDATE categories
          SET path = $1::ltree || subpath(path, $2),
              parent_id = CASE WHEN id = $3 THEN $4 ELSE parent_id END
        WHERE path <@ $5::ltree`,
      [parent.path, oldDepth - 1, nodeId, newParentId, node.path]);

    metrics.histogram('category.move.descendants', rowCount);
    return { moved: rowCount };
  });
}
```

**Step 6 — measure.**

```sql
-- subtree read
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, name, path FROM categories
 WHERE path <@ 'electronics.phones' ORDER BY path;
```
```
 Sort  (actual time=1.884..2.104 rows=8,204)
   ->  Bitmap Heap Scan on categories  (actual time=0.412..1.204 rows=8,204)
         Recheck Cond: (path <@ 'electronics.phones'::ltree)
         ->  ★ Bitmap Index Scan on idx_categories_path_gist
               (actual time=0.388..0.388 rows=8,204)
 Execution Time: ★ 2.412 ms
```
```
 ★ 184 ms → 2.4 ms.  76×.
```
```sql
-- the move
\timing on
SELECT moveCategory(88412, 41209);
-- Time: ★ 342.884 ms       (was 41,204 ms)
```
```
 ★ 41,204 ms → 343 ms.  120×.
 ★ AND it locks 8,204 rows instead of 613,086.
```

**Step 7 — the second pattern, discovered during the work.**

```
 While migrating, the team noticed the nav also shows a product
 count per category, computed live:

   SELECT c.id, c.name,
          (SELECT count(*) FROM products p
            WHERE p.category_path <@ c.path) AS product_count
     FROM categories c WHERE c.path <@ $1;

 ⇒ ★ 8,204 categories × a subtree count over 4.2M products
 ⇒ MEASURED: 2,840 ms
```

```
 ★ APPLY TOPIC 54'S GATES:
   GATE 1 ✓ 2,840 ms, 2.1M reads/day
   GATE 2 — an index on products(category_path) using gist? ⇒ helps
            the individual count, still 8,204 of them. ⇒ ✗
          — a materialised view? ⇒ ★ counts must reflect a product
            being published immediately. ⇒ ✗ (Topic 56)
   GATE 3 ✓ a real COPY
   GATE 4   reads 2.1M/day × 2,840 ms  = ★ 1,657 hours/day
            writes: product category changes ~18,000/day
            ★ FAN-OUT: changing one product's category must update
              the count for ★ EVERY ANCESTOR — up to 9 rows
            cost 0.4 ms × 18,000 × 9 = 65 seconds/day
            ⇒ ★ RATIO 91,700 : 1
            ★ CONTENTION: the root category is an ancestor of
              EVERY product ⇒ ★ 18,000 writes/day = 0.2/sec ✓
              (but note: at 800/sec the ROOT would be a hot row)
   GATE 5 ✓ trigger + reconciler + constraint + comment
 ⇒ ★ SHIP IT — as a MAINTAINED AGGREGATE, one per category,
   updated for every ancestor.
```

```sql
ALTER TABLE categories
  ADD COLUMN product_count integer NOT NULL DEFAULT 0
    CONSTRAINT ck_pc CHECK (product_count >= 0);

CREATE OR REPLACE FUNCTION sync_category_counts() RETURNS trigger AS $$
BEGIN
  IF TG_OP = 'INSERT' THEN
    UPDATE categories SET product_count = product_count + 1
     WHERE path @> NEW.category_path;              -- ★ every ancestor
  ELSIF TG_OP = 'DELETE' THEN
    UPDATE categories SET product_count = product_count - 1
     WHERE path @> OLD.category_path;
  ELSIF NEW.category_path IS DISTINCT FROM OLD.category_path THEN
    -- ★ the FK-change transition, in deterministic order
    UPDATE categories SET product_count = product_count - 1
     WHERE path @> OLD.category_path AND NOT (path @> NEW.category_path);
    UPDATE categories SET product_count = product_count + 1
     WHERE path @> NEW.category_path AND NOT (path @> OLD.category_path);
    -- ★ shared ancestors are untouched — no net change, no lock churn
  END IF;
  RETURN NULL;
END $$ LANGUAGE plpgsql;

CREATE TRIGGER trg_category_counts
  AFTER INSERT OR UPDATE OF category_path OR DELETE ON products
  FOR EACH ROW EXECUTE FUNCTION sync_category_counts();
```

```sql
-- ★ the reconciler
WITH actual AS (
  SELECT c.id, count(p.id) AS n
    FROM categories c LEFT JOIN products p ON p.category_path <@ c.path
   GROUP BY c.id)
SELECT count(*) AS drifted FROM categories c JOIN actual a ON a.id = c.id
 WHERE c.product_count IS DISTINCT FROM a.n;

COMMENT ON COLUMN categories.product_count IS
  'Denormalised count of products in this category AND all descendants.
   Maintained by trg_category_counts on products (updates every ancestor
   via path @> ). Reconciler: ops/reconcile_category_counts.sql (nightly
   + repair). Added 2026-08 — the live subtree count was 2,840ms on a
   2.1M/day endpoint. ★ NOTE: the root category is an ancestor of every
   product; at >800 product writes/sec it becomes a hot row and must be
   sharded (see Topic 61). Currently 0.2/sec. Owner: catalogue-team.';
```

**Step 8 — results.**

| | Nested sets | ltree + maintained count |
|---|---|---|
| Subtree read | 184 ms | 2.4 ms (**76×**) |
| Nav endpoint p99 | 4,180 ms | **8 ms** (**522×**) |
| Move one node | 41,204 ms | 343 ms (**120×**) |
| Rows locked per move | 613,086 | **8,204** |
| Quarterly restructure | 20 min lockup | **3.9 min**, no lockup |
| Product count query | 2,840 ms | **0.1 ms** (a column read) |
| Patterns used | — | ★ materialised path **+** maintained aggregate |
| Reconcilers owed | — | 1 (the path/parent trigger enforces the other) |

```
 ★ FOUR LESSONS:
 ① ★ NESTED SETS' FAMOUS "FAST READS" WEREN'T EVEN FAST. The rgt
   predicate can't use the lft index. The blog post was wrong on
   both axes.
 ② ★ THE ENCODING WAS CHOSEN FROM THEORY, NOT FROM THE
   READ:MOVE RATIO. Measuring it (4,596:1) picked ltree in one step.
 ③ ★ TWO DIFFERENT PATTERNS WERE NEEDED FOR ONE PAGE — a
   materialised path for the tree, a maintained aggregate for the
   counts. Treating "denormalisation" as one technique would have
   produced one wrong answer.
 ④ ★ THE PATH/PARENT_ID TRIGGER MAKES DRIFT IMPOSSIBLE rather
   than detectable. Where you can enforce at write time, do —
   a reconciler is the fallback, not the goal.
```

---

## Common mistakes

**1. Copying a mutable parent attribute with high fan-out.**
- *Symptom:* renaming one customer takes 3.9 seconds and 18 MB of WAL.
- *Fix:* an index instead. Copy only immutable attributes, cross-service values, or genuine snapshots.

**2. Storing an average instead of sum and count.**
- *Symptom:* impossible to update on `DELETE`; averages of averages.
- *Fix:* store both; make the average a `GENERATED` column.

**3. Forgetting the FK-change transition in an aggregate trigger.**
- *Symptom:* counts drift only when a child moves between parents — rare, so it's found months later.
- *Fix:* handle all four transitions, and lock the two parents in ascending id order to avoid deadlocks.

**4. Choosing nested sets.**
- *Symptom:* one insert or move rewrites half the table, holding locks throughout.
- *Fix:* materialised path (read-heavy) or closure table (move-heavy). Nested sets are a read-only structure.

**5. Choosing a tree encoding from a blog post rather than the read:move ratio.**
- *Symptom:* an encoding optimised for the operation you rarely perform.
- *Fix:* measure reads/day and moves/day. The ratio picks the encoding.

**6. Only creating a GiST index on an `ltree` column.**
- *Symptom:* `<@` is fast but equality, ordering and prefix ranges are not.
- *Fix:* both GiST and B-tree — they serve different operators.

**7. Embedding children that will need querying.**
- *Symptom:* "which orders contain SKU X" needs a 4 GB GIN index; "top SKUs by units" takes 41 seconds.
- *Fix:* the hybrid — normalised children for queries, an immutable jsonb snapshot for reprinting.

**8. Losing constraints by embedding.**
- *Symptom:* `qty: -3` and SKUs that don't exist, discovered in a report.
- *Fix:* normalised children carry FKs and `CHECK`s; jsonb cannot.

**9. Not enforcing consistency between two representations.**
- *Symptom:* `path` and `parent_id` disagree after a bulk import.
- *Fix:* a `BEFORE` trigger that rejects the write. Enforcement beats reconciliation where it's possible.

**10. Putting a maintained aggregate on a wide, heavily-indexed parent.**
- *Symptom:* 6× the expected WAL from non-HOT updates.
- *Fix:* a narrow side table with `fillfactor 70` (Topics 46, 54).

**11. Not checking which parent is hottest for a subtree aggregate.**
- *Symptom:* the root category is an ancestor of everything and becomes a hot row.
- *Fix:* measure it, document the threshold, and plan the sharding before you hit it (Topic 61).

---

## Hands-on proof

**PROVE IT #1–#8 — Example 1** (the free copied column and the expensive one measured at 3,884 ms / 18 MB, all four aggregate transitions, the `CHECK` rejecting an absurd value, `ltree` subtree and ancestor queries, the subtree rewrite on a move, the recursive CTE comparison, and the hybrid's queryability).

**PROVE IT #9 — nested sets rewriting half the table.**
```sql
CREATE TABLE ns (id bigserial PRIMARY KEY, lft int NOT NULL, rgt int NOT NULL);
INSERT INTO ns (lft, rgt) SELECT g, g+1 FROM generate_series(1, 1000000, 2) g;
CREATE INDEX ON ns (lft);
VACUUM ANALYZE ns;

\timing on
UPDATE ns SET lft = lft + 100, rgt = rgt + 100 WHERE lft > 500000;
```
```
 UPDATE 250000
 Time: ★ 18,412.8 ms      — for one conceptual "insert"
```

**PROVE IT #10 — ltree vs recursive CTE at scale.**
```sql
-- 1M nodes, depth 6
EXPLAIN (ANALYZE) SELECT count(*) FROM cat_path WHERE path <@ 'root.a.b';
--  Execution Time: ★ 0.412 ms

EXPLAIN (ANALYZE)
WITH RECURSIVE s AS (
  SELECT id FROM cat_adj WHERE id = 1204
  UNION ALL SELECT c.id FROM cat_adj c JOIN s ON c.parent_id = s.id)
SELECT count(*) FROM s;
--  Execution Time: ★ 41.204 ms      — 100×
```

**PROVE IT #11 — the GIN index cost of embedding.**
```sql
CREATE TABLE orders_json (id bigserial PRIMARY KEY, items jsonb NOT NULL);
INSERT INTO orders_json (items)
SELECT jsonb_build_array(
         jsonb_build_object('sku','K'||(random()*1000)::int,'qty',(random()*5+1)::int))
  FROM generate_series(1,1000000);
CREATE INDEX idx_oj_gin ON orders_json USING gin (items);

CREATE TABLE order_items_norm (id bigserial PRIMARY KEY, order_id bigint,
                               sku text NOT NULL, qty int NOT NULL);
INSERT INTO order_items_norm (order_id, sku, qty)
SELECT g, 'K'||(random()*1000)::int, (random()*5+1)::int FROM generate_series(1,1000000) g;
CREATE INDEX idx_oin_sku ON order_items_norm (sku);

SELECT 'gin' AS kind, pg_size_pretty(pg_relation_size('idx_oj_gin')) AS size
UNION ALL
SELECT 'btree', pg_size_pretty(pg_relation_size('idx_oin_sku'));
```
```
 kind  |  size
-------+---------
 gin   | ★ 412 MB
 btree | ★ 28 MB      — 14.7×
```
```sql
-- and the aggregate the embedded form cannot do cheaply
EXPLAIN (ANALYZE)
SELECT e->>'sku' AS sku, sum((e->>'qty')::int) AS units
  FROM orders_json, jsonb_array_elements(items) e
 GROUP BY 1 ORDER BY 2 DESC LIMIT 10;
--  Execution Time: ★ 8,842 ms

EXPLAIN (ANALYZE)
SELECT sku, sum(qty) AS units FROM order_items_norm GROUP BY 1 ORDER BY 2 DESC LIMIT 10;
--  Execution Time: ★ 412 ms      — 21×
```

**PROVE IT #12 — the deadlock the FK-change transition causes without ordering.**
```sql
-- session 1: UPDATE reviews SET product_id = 2 WHERE id = 1;  (1 → 2)
-- session 2: UPDATE reviews SET product_id = 1 WHERE id = 9;  (2 → 1)
-- ★ without the ORDER BY id FOR NO KEY UPDATE, these deadlock.
-- ERROR: deadlock detected
```

---

## The design decision framework

```
★★★ THE PATTERN IS DECIDED BY THE ACCESS PATTERN, NOT BY TASTE. ★★★

 ① BEFORE ANY PATTERN — TRY THE TWO THAT COST NOTHING
    ★ GENERATED column — if it's a function of the same row
    ★ COVERING INDEX (INCLUDE) — the engine maintains it atomically
    ⇒ both give the read benefit with ZERO obligation. (Topic 12.)

 ② "PARENT ATTRIBUTE ON EACH CHILD ROW"  ⇒ COPIED COLUMN
    ✓ only when: ★ the parent is IMMUTABLE (country, currency)
                 ★ the join crosses a SERVICE BOUNDARY
                 ★ it's a SNAPSHOT (then it's not denormalisation)
    ✗ otherwise ⇒ ★ FAN-OUT will kill you. Use an index.
    ★ MEASURE: p99 rows-per-parent-change. >100 ⇒ don't.

 ③ "COUNT/SUM OF CHILDREN"  ⇒ MAINTAINED AGGREGATE
    ★ store SUM and COUNT; the average is a GENERATED column
    ★ handle ALL FOUR transitions (INSERT/DELETE/value/★ FK change)
    ★ lock two parents in ASCENDING ID ORDER on an FK change
    ★ narrow side table if the parent is wide or heavily indexed
    ★ CHECK constraints so absurd values are rejected at write time
    ★ MEASURE: writes/sec on the HOTTEST parent.
      >800/sec ⇒ shard it (Topic 61) or go async (Topic 56).

 ④ "SUBTREE / ANCESTORS"  ⇒ PICK BY READ:MOVE RATIO
    ratio > ~100 : 1   ⇒ ★ MATERIALISED PATH (ltree)
                          + BOTH a GiST and a B-tree index
                          + ★ a BEFORE trigger enforcing
                            path ↔ parent_id consistency
    frequent moves     ⇒ ★ CLOSURE TABLE (O(depth) rows per node)
    shallow, rare      ⇒ adjacency list + RECURSIVE CTE
    ★ never            ⇒ nested sets (one insert rewrites half
                          the table and holds locks throughout)

 ⑤ "THE WHOLE CHILD COLLECTION"  ⇒ EMBED ONLY IF ALL THREE HOLD
    ★ ① the children are NEVER queried independently
    ★ ② no per-item constraints or FKs are needed
    ★ ③ the shape is variable, or the value is an immutable snapshot
    ⇒ ★ OTHERWISE USE THE HYBRID:
      normalised children (queryable, constrained, FK'd)
      + an immutable jsonb snapshot (no obligation — it's history)
    ★ REMEMBER: a GIN index is ~15× a B-tree, and "top N by units"
      goes from 0.4 s to 8.8 s.

 ⑥ TWO PATTERNS ON ONE PAGE IS NORMAL
    the category example needed a materialised path AND a
    maintained aggregate. ★ Evaluate each read independently.

 ⑦ ★ PREFER ENFORCEMENT OVER RECONCILIATION
    a BEFORE trigger that REJECTS an inconsistent write beats a
    nightly job that DETECTS one.
    ⇒ reconciliation is the fallback for what you cannot enforce.
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
Build a maintained aggregate with `sum`, `count` and a `GENERATED` average. Exercise all four transitions and show the values after each. Then prove the `CHECK` constraint rejects a manually corrupted value.

### Exercise 2 — medium (apply it)
Build the same 1,000-node tree four ways: adjacency list, materialised path (ltree), nested sets, closure table. For each, measure: (a) "all descendants of node X"; (b) "all ancestors of node Y"; (c) "move node X under node Z", reporting rows touched and time.

Produce a table of the twelve numbers and state which encoding you'd choose at read:move ratios of 10:1, 1,000:1 and 1:10.

### Exercise 3 — hard (production simulation)
A procurement platform stores 1.2M categories (depth 9) as nested sets. Navigation p99 is 4,180 ms across 2.1M requests/day; the quarterly restructure moves 40,000 nodes and locks the system for 20 minutes.

(a) Show why the nested-sets subtree read is *not* fast, using `EXPLAIN`. Which predicate can't use the index, and why?
(b) Measure the move: how many rows does moving one node with 8,204 descendants actually rewrite, and why is it not just the subtree?
(c) Compute the read:move ratio from `pg_stat_statements` and an audit table. Which encoding does it select, and why not the other candidate?
(d) Write the migration: the recursive path build, both indexes, and why two are needed.
(e) Write the `BEFORE` trigger enforcing `path ↔ parent_id` consistency. Explain why this is better than a reconciler.
(f) Write `moveCategory`, including the cycle guard and the single subtree-rewrite statement.
(g) A second problem surfaces: a live subtree product count taking 2,840 ms. Walk Topic 54's five gates for it with numbers.
(h) Write the trigger that updates *every ancestor's* count, including the category-change transition. Explain the `AND NOT (path @> …)` clauses.
(i) The root category is an ancestor of every product. At what write rate does it become a hot row? Document the threshold and the plan.
(j) Write the `COMMENT ON COLUMN` capturing the why, the mechanism, the reconciler and the hot-row caveat.

---

## Mental model checkpoint

1. Name the four patterns and the failure mode of each.
2. Under what three conditions is a copied column actually safe?
3. Why store sum and count rather than an average? What should the average be?
4. Name the four transitions an aggregate trigger must handle. Which is forgotten, and what second bug does it cause?
5. Name the four tree encodings, and the one operation each makes fast.
6. Why are nested sets almost never correct in a concurrent system?
7. Why does an `ltree` column want both a GiST and a B-tree index?
8. Name five things you lose by embedding children as `jsonb`.
9. Describe the hybrid, and explain why the `jsonb` half carries no obligation.
10. Why is enforcement (a `BEFORE` trigger) preferable to reconciliation, and when can't you use it?

---

## Quick reference card

| Pattern | Buys | Costs | Use when |
|---|---|---|---|
| **Copied column** | no join | ★ **fan-out** | parent immutable · crosses a service boundary · it's a snapshot |
| **Maintained aggregate** | O(1) count | ★ **contention** | reads ≫ writes, hottest parent < 800 w/s |
| **Materialised path** | subtree in 1 scan | ★ **subtree rewrite** | read:move > ~100:1 |
| **Embedded doc** | 1 fetch | ★ **queryability** | children never queried alone |

**Try first, zero obligation:** `GENERATED ALWAYS AS (…) STORED` · a covering index with `INCLUDE`.

**Maintained aggregate — the checklist**
```sql
review_count int, review_sum bigint,
avg numeric GENERATED ALWAYS AS (…) STORED,       -- ★ cannot drift
CHECK (review_count >= 0),
CHECK (review_sum BETWEEN 0 AND review_count*5)
-- trigger handles: INSERT · DELETE · value UPDATE · ★ FK UPDATE
-- ★ FK UPDATE: lock both parents ORDER BY id FOR NO KEY UPDATE
-- ★ narrow side table + fillfactor 70 if the parent is wide
```

**Trees**

| Encoding | Subtree | Move | Verdict |
|---|---|---|---|
| adjacency list | 41 ms | 0.3 ms | shallow, rare reads |
| ★ **ltree path** | **0.4 ms** | 340 ms | ★ read-heavy (most cases) |
| nested sets | 0.3 ms | ★ **8,400 ms** | ★ **never** |
| ★ closure table | 0.6 ms | 12 ms | frequent moves, deep |

```sql
CREATE INDEX … USING gist (path);    -- ★ for <@ and @>
CREATE INDEX … USING btree (path);   -- ★ for = , ORDER BY, prefix
WHERE path <@ 'electronics.phones'   -- descendants
WHERE path @> 'a.b.c'                -- ancestors
```

**Embedding costs:** GIN index ~15× a B-tree · "top N by units" 21× slower · no FKs · no per-item `CHECK`s.
**★ The hybrid:** normalised children **for queries** + immutable `jsonb` **for reprinting** — and the jsonb is a snapshot, so it costs nothing.

**★ Prefer enforcement (a `BEFORE` trigger that rejects) over reconciliation (a job that detects).**

---

## When would I use this at work?

1. **Designing any category, org-chart, folder or comment-thread hierarchy.** The encoding choice is made once and is expensive to change. Measuring read:move takes five minutes and picks it correctly; picking from a blog post produced a 20-minute quarterly lockup in the example above.

2. **Reviewing a PR that adds `jsonb` for a child collection.** One question: *"will anyone ever need to query across these?"* If yes — and it usually becomes yes — the hybrid gives you both for almost no extra cost, and the `jsonb` half needs no maintenance because it's a snapshot.

3. **Any counter or aggregate column.** The four-transition checklist catches the FK-change bug that otherwise surfaces months later as unexplained drift, and the ascending-lock-order rule prevents the deadlock that comes with it.

4. **When someone says "we denormalised and it's still slow."** Usually the wrong *pattern* was chosen — a copied column where fan-out dominates, or an embedded document where the new requirement needs querying. Naming the four patterns makes the mismatch obvious.

---

## Connected topics

**Understand before this:** 53 (the obligation), 54 (the five gates — you're past them), 46 (HOT updates — why a narrow side table wins), 48 (deadlocks — the FK-change lock order), 16 (GiST/GIN — the index types these patterns need), 27 (antipatterns — EAV, which embedding sometimes rightly replaces).

**This unlocks:**
- **56** — materialised views: the async alternative to a maintained aggregate
- **57** — caching: the same patterns, outside the database
- **59** — partitioning: how these patterns interact with partitioned tables
- **61** — counters and hot rows: what to do when the aggregate's contention check fails
- **70** — document modelling: when embedding is the *primary* design, not a denormalisation
