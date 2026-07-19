# Topic 70 — Table Partitioning
### SQL Mastery Curriculum — Phase 10: PostgreSQL Specific Power Features

---

## 1. ELI5 — The Analogy That Makes It Click

Imagine you run a library with 100 million books, all crammed into one gigantic room. When someone asks for "every book published in March 2024," a librarian has to walk the entire room, glancing at the publication date on every single spine, because the books are in no useful physical order for that question. Even with a card catalog (an index), the librarian still has to run around the whole enormous room fetching books scattered everywhere.

Now imagine you rebuild the library so there is **one room per month**. The March 2024 room. The April 2024 room. When the "March 2024" request comes in, the librarian walks straight past 99 rooms — doesn't even open their doors — and only searches inside the one March room. The other 99 rooms might as well not exist for this query.

That is **table partitioning**. You take one logical table (`orders`) and physically split it into many smaller child tables (`orders_2024_03`, `orders_2024_04`, …) based on a rule — a **partition key**. To your application it still looks like one table. But PostgreSQL, at plan time, figures out which rooms it can skip entirely. That skipping is called **partition pruning**, and it is the whole point.

There's a second magic trick. When the library decides March 2021 data is old and should be moved to cold storage, you don't run a slow `DELETE` that scans 100 million rows. You just **wheel the entire March 2021 room out of the building** in one instant — `DETACH PARTITION`. And when April 2025 arrives, you wheel in a fresh empty room — `ATTACH PARTITION`. Rooms in, rooms out, in constant time, no matter how full they are.

Partitioning is *not* about making one query on one row faster — a B-tree index already does that. It is about (1) skipping enormous swaths of data the query provably can't need, and (2) turning "delete/archive a whole time range" from an O(rows) bloat-generating nightmare into an O(1) metadata operation.

---

## 2. Connection to SQL Internals

A partitioned table in PostgreSQL is not a single heap. The **parent** is a purely logical relation — it has `relkind = 'p'` (partitioned table) in `pg_class` and **stores zero rows and occupies zero heap pages of its own**. Every actual tuple lives in a **leaf partition**, which is an ordinary heap (`relkind = 'r'`) with its own set of 8KB pages, its own visibility map, its own free space map, and its own indexes.

Name the internal machinery explicitly:

- **`pg_class` / `pg_inherits` / `pg_partitioned_table`**: The parent-child relationships are recorded in `pg_inherits`; the partition strategy (RANGE/LIST/HASH) and partition-key expression live in `pg_partitioned_table`. Each partition's bound (`FOR VALUES FROM … TO …`) is stored as `relpartbound` in the child's `pg_class` row.
- **The planner's partition pruning**: Before execution, the planner examines the query's `WHERE` clause against each partition's bound. If a partition's range provably cannot satisfy the predicate, its scan node is **removed from the plan entirely** (`relation_excluded_by_constraints` / the newer `partprune` machinery). This is *not* index skipping — the partition's pages are never touched, its index is never opened.
- **B-tree indexes are per-partition**: A partitioned table cannot have a single physical index spanning all children. When you `CREATE INDEX` on the parent, PostgreSQL creates a **partitioned index** (a template) and one real B-tree per leaf. Each leaf's B-tree is independent, smaller, and shallower — a 100M-row global index of depth 5 becomes twelve 8M-row indexes of depth 4.
- **Tuple routing on INSERT**: When you `INSERT` into the parent, the executor evaluates the partition key on each incoming row and routes it to the correct leaf heap. This is a hash/binary-search over partition bounds, done in the `ExecFindPartition` path — cheap, but not free, and it's why bulk `COPY` into a heavily-partitioned table has measurable overhead.
- **MVCC and VACUUM are per-partition**: Each leaf has its own visibility map and dead-tuple accounting. Autovacuum treats each partition as a separate table — a huge win, because vacuuming a 10M-row partition is fast and its bloat is isolated, whereas one 1B-row monolith can accumulate bloat that vacuum struggles to keep up with.
- **The partition key must be part of every UNIQUE/PRIMARY KEY**: Because each leaf's unique B-tree is local, PostgreSQL cannot enforce global uniqueness across partitions without including the partition key in the constraint. This is a structural consequence of per-partition indexes, not an arbitrary rule.

The mental model: the parent is a *view-like router*; the leaves are *real tables*; the planner's job is to prove which leaves are irrelevant and delete them from the plan before a single page is read.

---

## 3. Logical Execution Order Context

Partitioning interacts with query processing at two distinct times, and confusing them causes most partitioning disappointment.

```
FROM partitioned_table
  ├── PLAN-TIME pruning   ← planner reads WHERE, compares to relpartbound,
  │                         removes provably-empty partitions from the plan
JOIN ...
WHERE partition_key = ... ← the predicate that enables pruning MUST reference
  │                         the partition key (or an expression the planner
  │                         can map to it)
  ├── EXECUTION-TIME pruning ← for params not known at plan time (generic plans,
  │                            $1 parameters, nested-loop join keys), partitions
  │                            are pruned when the actual value arrives
GROUP BY ...
HAVING ...
SELECT ...
ORDER BY ...   ← Append/MergeAppend combines surviving partitions here
LIMIT ...
```

Two critical ordering facts:

1. **Pruning happens before scanning, and it depends entirely on the `WHERE` clause referencing the partition key.** If you partition `orders` by `created_at` but filter only on `customer_id`, the planner *cannot* prune — it must scan every partition. Partitioning bought you nothing for that query (and cost you Append overhead). The partition key must appear in the predicates of the queries you care about.

2. **Plan-time vs execution-time pruning are different mechanisms.** Plan-time pruning works when the partition-key value is a constant literal at planning (`WHERE created_at >= '2024-03-01'`). Execution-time pruning handles values not known until runtime: `$1` bind parameters under a generic plan, and the inner side of a nested-loop join where each outer row supplies a different partition-key value. You verify plan-time pruning by counting scan nodes in `EXPLAIN`; you verify execution-time pruning by reading `(never executed)` markers and `Subplans Removed: N` in `EXPLAIN ANALYZE`.

The combination of `FROM` producing an `Append` (or `MergeAppend` when ordered) node over surviving partitions is what lets everything downstream — `GROUP BY`, aggregates, `ORDER BY` — treat the pile of leaves as one stream.

---

## 4. What Is Table Partitioning?

Table partitioning is the declarative splitting of one logical table into multiple physical child tables (partitions) according to a **partition key** and a **partition strategy** (RANGE, LIST, or HASH), where the parent holds no data and the planner can skip whole partitions that provably cannot match a query's predicates.

```sql
CREATE TABLE orders (
    id           BIGINT       GENERATED ALWAYS AS IDENTITY,
    customer_id  BIGINT       NOT NULL,
    status       TEXT         NOT NULL,
    total_amount NUMERIC(12,2) NOT NULL,
    created_at   TIMESTAMPTZ  NOT NULL,
    PRIMARY KEY (id, created_at)          -- ← partition key MUST be in the PK
) PARTITION BY RANGE (created_at);
--   │            │      │
--   │            │      └── the partition KEY: the column/expression whose value
--   │            │          decides which partition a row lands in
--   │            └── the STRATEGY: RANGE | LIST | HASH
--   └── declares this table as a partitioned PARENT (relkind='p'); it will hold
--       no rows of its own — every row lives in a leaf partition

CREATE TABLE orders_2024_q1 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2024-04-01');
--   │                 │                   │
--   │                 │                   └── UPPER bound, EXCLUSIVE (< '2024-04-01')
--   │                 └── LOWER bound, INCLUSIVE (>= '2024-01-01')
--   └── this child is a real heap (relkind='r') owning the Q1 2024 range

CREATE TABLE orders_2024_q2 PARTITION OF orders
    FOR VALUES FROM ('2024-04-01') TO ('2024-07-01');
```

The three strategies:

```sql
-- RANGE: contiguous value ranges. Ideal for time-series (created_at), sequences.
PARTITION BY RANGE (created_at)
  FOR VALUES FROM ('2024-01-01') TO ('2024-04-01');   -- [lower, upper)

-- LIST: explicit discrete values. Ideal for low-cardinality categoricals (region, status).
PARTITION BY LIST (region)
  FOR VALUES IN ('us-east', 'us-west');

-- HASH: modulus/remainder spread. Ideal for evening out load with no natural range/list.
PARTITION BY HASH (customer_id)
  FOR VALUES WITH (MODULUS 8, REMAINDER 0);           -- one per remainder 0..7
```

Supporting syntax you must know:

```sql
-- DEFAULT partition: catches rows matching no other partition (RANGE/LIST only)
CREATE TABLE orders_default PARTITION OF orders DEFAULT;

-- Sub-partitioning: a partition can itself be partitioned
CREATE TABLE orders_2024_q1 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2024-04-01')
    PARTITION BY LIST (region);

-- ATTACH: bring an existing standalone table in as a partition
ALTER TABLE orders ATTACH PARTITION orders_2024_q3
    FOR VALUES FROM ('2024-07-01') TO ('2024-10-01');

-- DETACH: remove a partition, turning it back into a standalone table (O(1))
ALTER TABLE orders DETACH PARTITION orders_2024_q1;
ALTER TABLE orders DETACH PARTITION orders_2024_q1 CONCURRENTLY;  -- no ACCESS EXCLUSIVE
```

---

## 5. Why Table Partitioning Mastery Matters in Production

1. **Archival/retention becomes O(1) instead of O(rows).** The single biggest reason to partition time-series data. `DELETE FROM orders WHERE created_at < '2021-01-01'` on a 500M-row table scans and marks tens of millions of dead tuples, generates enormous WAL, bloats the heap, and hammers autovacuum for hours. `ALTER TABLE orders DETACH PARTITION orders_2020` plus `DROP TABLE orders_2020` frees the same data instantly with a metadata change. This alone justifies partitioning for any table with a time-based retention policy.

2. **Partition pruning cuts I/O by orders of magnitude on range queries.** A dashboard querying "last 7 days" against a table partitioned by day touches 7 partitions out of 730 — the planner never opens the other 723. Without partitioning, even with an index, the query planner must at least walk an index spanning all 730 days' worth of entries and the buffer pool churns cold historical index pages.

3. **Autovacuum and bloat stay under control.** Each partition vacuums independently and quickly. A monolithic append-heavy table can outrun autovacuum, leading to transaction-ID-wraparound risk and unbounded bloat. Partitioned, only the hot recent partition needs frequent vacuuming; cold partitions are effectively frozen and left alone.

4. **Smaller, shallower indexes.** Twelve monthly B-trees are each ~1/12 the size and often one level shallower than one giant index. Index maintenance on INSERT touches only the current partition's index, keeping the hot working set in cache.

5. **The failure modes are expensive and subtle.** Partitioning the wrong way is *worse* than not partitioning: queries that don't filter on the partition key now pay Append overhead across hundreds of partitions; the mandatory partition-key-in-PK rule can force a composite primary key that breaks foreign keys pointing at the table; a missing partition with no DEFAULT causes INSERTs to fail with a hard error; too many partitions blow up planning time. Mastery is knowing when partitioning helps versus when it silently taxes every query.

---

## 6. Deep Technical Content

### 6.1 RANGE Partitioning — Semantics and Boundaries

RANGE partitions cover a half-open interval `[lower, upper)` — **lower bound inclusive, upper bound exclusive**. This is deliberate: adjacent partitions share a boundary value with no gap and no overlap.

```sql
CREATE TABLE orders_2024_01 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');   -- Jan: >= Jan 1, < Feb 1
CREATE TABLE orders_2024_02 PARTITION OF orders
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');   -- Feb: >= Feb 1, < Mar 1
```

A row with `created_at = '2024-02-01 00:00:00'` lands in `orders_2024_02`, **not** January — the upper bound is exclusive. Getting this backwards (using overlapping inclusive ranges) is a common mental error; PostgreSQL will reject overlapping RANGE partitions at `CREATE`/`ATTACH` time with `partition ... would overlap partition ...`.

Special boundary values `MINVALUE` and `MAXVALUE` represent unbounded ends:

```sql
CREATE TABLE orders_ancient PARTITION OF orders
    FOR VALUES FROM (MINVALUE) TO ('2020-01-01');    -- everything before 2020
CREATE TABLE orders_future PARTITION OF orders
    FOR VALUES FROM ('2030-01-01') TO (MAXVALUE);    -- everything from 2030 on
```

Multi-column RANGE keys compare **lexicographically** (like a tuple/row comparison), which is subtle:

```sql
PARTITION BY RANGE (year, month)
-- FOR VALUES FROM (2024, 1) TO (2024, 4) means:
--   (year, month) >= (2024, 1) AND (year, month) < (2024, 4)
-- NOT "year=2024 AND month between 1 and 3" in the naive sense —
-- a row (2024, 12) is >= (2024,1) and < (2024,4)? No: (2024,12) >= (2024,4) → excluded. OK here.
-- But (2023, 99) >= (2024,1)? No. Lexicographic ordering, first column dominates.
```

### 6.2 LIST Partitioning

LIST assigns explicit discrete values to partitions. Best for low-cardinality categorical keys.

```sql
CREATE TABLE events (
    id BIGINT GENERATED ALWAYS AS IDENTITY,
    region TEXT NOT NULL,
    payload JSONB,
    PRIMARY KEY (id, region)
) PARTITION BY LIST (region);

CREATE TABLE events_us PARTITION OF events FOR VALUES IN ('us-east', 'us-west');
CREATE TABLE events_eu PARTITION OF events FOR VALUES IN ('eu-central', 'eu-west');
CREATE TABLE events_apac PARTITION OF events FOR VALUES IN ('ap-south', 'ap-northeast');
CREATE TABLE events_other PARTITION OF events DEFAULT;   -- anything unlisted
```

A single value can belong to exactly one LIST partition; PostgreSQL rejects duplicates across partitions. `NULL` in the partition key requires either a partition that explicitly lists NULL (`FOR VALUES IN (NULL)`) or a DEFAULT partition, otherwise the INSERT fails.

### 6.3 HASH Partitioning

HASH spreads rows evenly using `hash(key) mod modulus = remainder`. There is no meaningful pruning for range queries (hashing destroys ordering), but it prunes **equality** predicates and balances write load and per-partition size when no natural range/list key exists.

```sql
CREATE TABLE sessions (
    id UUID DEFAULT gen_random_uuid(),
    user_id BIGINT NOT NULL,
    data JSONB,
    PRIMARY KEY (id, user_id)
) PARTITION BY HASH (user_id);

CREATE TABLE sessions_0 PARTITION OF sessions FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE sessions_1 PARTITION OF sessions FOR VALUES WITH (MODULUS 4, REMAINDER 1);
CREATE TABLE sessions_2 PARTITION OF sessions FOR VALUES WITH (MODULUS 4, REMAINDER 2);
CREATE TABLE sessions_3 PARTITION OF sessions FOR VALUES WITH (MODULUS 4, REMAINDER 3);
```

Key facts about HASH:
- `WHERE user_id = 42` prunes to exactly one partition (the hash is computed at plan/exec time).
- `WHERE user_id BETWEEN 1 AND 100` prunes **nothing** — hashing scatters those values across all partitions. HASH is useless for range scans.
- You cannot have a DEFAULT partition with HASH, and the modulus set must be complete (every remainder covered) or INSERTs to uncovered remainders fail. You can grow a hash set by increasing modulus, but it requires detaching and redistributing — HASH is not friendly to online resizing.

### 6.4 Partition Pruning — Plan Time vs Execution Time

**Plan-time pruning** fires when partition-key predicates are constant-folded at planning:

```sql
-- Constant literal: planner prunes to the single February partition at plan time
EXPLAIN SELECT * FROM orders WHERE created_at >= '2024-02-10' AND created_at < '2024-02-20';
-- Plan contains ONLY orders_2024_02 as a scan node; other partitions absent.
```

**Execution-time pruning** fires when the value isn't known until runtime:

```sql
-- Generic plan with a bind parameter: the specific partition is chosen at execute time
PREPARE q(timestamptz) AS SELECT * FROM orders WHERE created_at >= $1;
EXPLAIN (ANALYZE) EXECUTE q('2024-06-01');
-- Plan shows all partitions, but ANALYZE shows "(never executed)" on pruned ones
-- and possibly "Subplans Removed: N"
```

Execution-time pruning also drives **nested-loop join pruning**: for each outer row, the inner partitioned table is pruned to the matching partition using that row's key. This is what makes joining a small driving table to a large partitioned table efficient.

The controlling GUC is `enable_partition_pruning` (default `on`). The related `constraint_exclusion` GUC governs the *older* inheritance-based exclusion and is **not** what drives declarative-partition pruning (a frequent misconception) — leave it at `partition` and rely on `enable_partition_pruning`.

Pruning is defeated by:
- Wrapping the partition key in a non-immutable or non-pruneable function: `WHERE date(created_at) = '2024-02-10'` may not prune if the planner can't map `date(created_at)` back to the raw `created_at` bounds. Prefer `WHERE created_at >= '2024-02-10' AND created_at < '2024-02-11'`.
- Filtering on a column that isn't the partition key at all.
- `WHERE created_at = some_volatile_expr()` where the value can't be resolved.

### 6.5 Partition-Wise Joins

When **both** tables are partitioned on the join key with **matching partition bounds**, PostgreSQL can join partition-by-partition rather than materializing both whole tables — a **partition-wise join**. This turns one huge join into N small independent joins, each fitting in cache/work_mem.

```sql
SET enable_partitionwise_join = on;   -- OFF by default! must be enabled

-- Both partitioned by RANGE(created_at) with identical monthly bounds:
SELECT o.id, oi.product_id, oi.quantity
FROM orders o
JOIN order_items oi ON oi.order_id = o.id AND oi.created_at = o.created_at
WHERE o.created_at >= '2024-01-01' AND o.created_at < '2024-02-01';
-- Planner joins orders_2024_01 ⋈ order_items_2024_01 as an isolated pair.
```

Requirements and caveats:
- `enable_partitionwise_join` is **off by default** because planning cost grows with partition count; you must turn it on.
- Partition bounds must match exactly (same strategy, same boundaries). Mismatched bounds fall back to a normal join over the Append.
- The join clause must include the partition key(s).

### 6.6 Partition-Wise Aggregation

Analogously, `enable_partitionwise_aggregate` (also **off by default**) lets `GROUP BY` on the partition key be computed per-partition and then combined, avoiding one giant hash-aggregate:

```sql
SET enable_partitionwise_aggregate = on;

SELECT date_trunc('month', created_at) AS mth, COUNT(*), SUM(total_amount)
FROM orders
GROUP BY 1;
-- Each partition aggregates its own rows; results are appended. Great for parallelism.
```

This is especially powerful combined with parallel query — each partition's partial aggregate can run in a separate worker.

### 6.7 The ATTACH / DETACH Pattern for Time-Series Rotation

The canonical operational workflow for rolling time-series retention:

```sql
-- 1. Pre-create NEXT month's partition ahead of time (never let INSERTs hit a gap)
CREATE TABLE orders_2025_02 PARTITION OF orders
    FOR VALUES FROM ('2025-02-01') TO ('2025-03-01');

-- 2. To retire the oldest month: detach (O(1) metadata op), then archive/drop
ALTER TABLE orders DETACH PARTITION orders_2024_01 CONCURRENTLY;
-- orders_2024_01 is now a normal standalone table. Copy it to cold storage, then:
DROP TABLE orders_2024_01;
```

**Why DETACH beats DELETE:** `DELETE` marks every row dead (MVCC), writes a WAL record per row, leaves the pages bloated until VACUUM reclaims them, and can take minutes-to-hours on large ranges. `DETACH` flips a catalog pointer. The data is gone from the logical table instantly with negligible WAL.

**The ATTACH validation trap:** When you `ATTACH` a pre-existing populated table, PostgreSQL must verify every row satisfies the new partition's bound. By default this takes an `ACCESS EXCLUSIVE` lock and scans the whole table. To avoid the scan, add a matching `CHECK` constraint *before* attaching — PostgreSQL uses it to prove the bound is satisfied and skips the scan:

```sql
-- Prepare the table so ATTACH is instant (no validation scan):
ALTER TABLE orders_2025_03 ADD CONSTRAINT ck_range
    CHECK (created_at >= '2025-03-01' AND created_at < '2025-04-01');
-- Now ATTACH proves the bound from the CHECK and skips the full scan:
ALTER TABLE orders ATTACH PARTITION orders_2025_03
    FOR VALUES FROM ('2025-03-01') TO ('2025-04-01');
-- Afterwards the redundant CHECK can be dropped.
ALTER TABLE orders_2025_03 DROP CONSTRAINT ck_range;
```

`DETACH ... CONCURRENTLY` (PG 14+) avoids the `ACCESS EXCLUSIVE` lock on the parent, letting reads/writes continue against other partitions during detach — essential for zero-downtime rotation.

### 6.8 Indexes on Partitioned Tables

Creating an index on the parent creates a **partitioned index** (a catalog template, `relkind='I'`) plus one real B-tree per leaf:

```sql
CREATE INDEX idx_orders_customer ON orders (customer_id);
-- Automatically creates idx on every existing partition AND every future one.
```

Key mechanics:
- **`CREATE INDEX ... CONCURRENTLY` is NOT supported on the partitioned parent directly.** To build without long locks, create the index on each partition `CONCURRENTLY` individually, then create an `ONLY` index on the parent and `ATTACH` each child index:
  ```sql
  CREATE INDEX CONCURRENTLY idx_orders_2024_01_cust ON orders_2024_01 (customer_id);
  -- repeat per partition, then:
  CREATE INDEX idx_orders_customer ON ONLY orders (customer_id);   -- invalid until all attached
  ALTER INDEX idx_orders_customer ATTACH PARTITION idx_orders_2024_01_cust;
  ```
  Once every child index is attached, the parent index automatically becomes valid.
- Each leaf B-tree is smaller and shallower — better cache behavior, faster maintenance.
- A partition can have **extra** local indexes the parent doesn't define (e.g., a specialized index only on the hot partition).

### 6.9 Constraints, the Partition-Key-in-PK Rule, and Foreign Keys

**The rule:** any `PRIMARY KEY` or `UNIQUE` constraint on a partitioned table **must include all partition-key columns**. Because each leaf's unique index is local, PostgreSQL can only guarantee global uniqueness if uniqueness within a partition plus partition-disjointness together imply global uniqueness — which requires the partition key to be part of the constraint.

```sql
-- WRONG: id alone as PK on a table partitioned by created_at
CREATE TABLE orders (
    id BIGINT PRIMARY KEY,               -- ERROR: unique constraint on partitioned table
    created_at TIMESTAMPTZ NOT NULL      --   must include partition key "created_at"
) PARTITION BY RANGE (created_at);

-- RIGHT: composite PK including the partition key
CREATE TABLE orders (
    id BIGINT GENERATED ALWAYS AS IDENTITY,
    created_at TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (id, created_at)         -- id no longer globally unique BY ITSELF
) PARTITION BY RANGE (created_at);
```

Consequences you must design around:
- `id` is now only unique *together with* `created_at`. If application code assumes `id` is a globally unique lookup key, it still works for point lookups but the *database* only enforces uniqueness on the pair. Two rows with the same `id` but different `created_at` are legal — a real hazard if `id` comes from a sequence but you don't trust it.
- **Foreign keys referencing a partitioned table** must reference a unique constraint, i.e. the full `(id, created_at)` — so child tables must carry `created_at` too. This often makes FKs *into* partitioned tables awkward. (FKs *from* a partitioned table to a normal table work fine and are fully supported since PG 12.)
- `CHECK` constraints and `NOT NULL` on the parent propagate to all partitions.

### 6.10 Tuple Routing, DEFAULT Partitions, and INSERT Failures

On INSERT/COPY the executor routes each row to its leaf. If no partition matches and there is **no DEFAULT partition**, the statement fails:

```
ERROR: no partition of relation "orders" found for row
DETAIL: Partition key of the failing row contains (created_at) = (2025-12-25 ...).
```

This is a production outage waiting to happen: forget to pre-create next month's partition and every INSERT starts erroring at midnight on the 1st. Defenses:
- **Automate partition creation** ahead of time (cron/pg_partman/scheduled job creating N months ahead).
- **A DEFAULT partition** catches strays — but beware: attaching a *new* partition whose range overlaps rows currently sitting in DEFAULT forces a scan of the DEFAULT partition to prove none conflict, taking a strong lock. Keep the DEFAULT empty in steady state.

### 6.11 When Partitioning Helps vs Hurts

**Helps when:**
- The table is large (rule of thumb: north of tens of GB / hundreds of millions of rows).
- There is a natural partition key that appears in the `WHERE` of the important queries (usually time).
- You have a time-based retention/archival policy (the DETACH win).
- Write and vacuum load concentrates on recent data.

**Hurts when:**
- Queries don't filter on the partition key → no pruning, pure Append overhead.
- Too many partitions (thousands): planning time balloons, execution-time pruning helps but plan-time pruning and lock acquisition on all partitions cost real milliseconds. Keep partitions in the low hundreds where possible.
- The table is small — partitioning a 1M-row table is almost always net-negative overhead.
- You need `id`-only uniqueness or simple inbound foreign keys — the partition-key-in-PK rule fights you.
- Cross-partition queries dominate — e.g., `WHERE customer_id = ?` on a table partitioned by `created_at` scans every partition (though a `customer_id` index on each still helps within each leaf).

---

## 7. EXPLAIN — Partition Pruning in the Plan

### Plan-time pruning: only surviving partitions appear

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, customer_id, total_amount
FROM orders
WHERE created_at >= '2024-02-01' AND created_at < '2024-03-01';
```

Assume `orders` has 24 monthly partitions (2023-01 … 2024-12).

```
Seq Scan on orders_2024_02 orders  (cost=0.00..2310.00 rows=41000 width=28)
                                    (actual time=0.02..9.8 rows=40213 loops=1)
  Filter: (created_at >= '2024-02-01' AND created_at < '2024-03-01')
  Buffers: shared hit=1210
Planning Time: 0.6 ms
Execution Time: 12.1 ms
```

**Reading it:**
- Only `orders_2024_02` appears — the other 23 partitions were **pruned at plan time** and are entirely absent from the plan. No Append node is even needed because exactly one partition survived.
- `Buffers: shared hit=1210` reflects only the February partition's pages. The other 23 partitions' pages were never touched.
- `Planning Time: 0.6 ms` — pruning cost. With hundreds of partitions this grows; watch it.

### Multiple surviving partitions: Append node

```sql
EXPLAIN (ANALYZE)
SELECT count(*) FROM orders
WHERE created_at >= '2024-11-15' AND created_at < '2025-01-10';
```

```
Aggregate  (cost=6820.00..6820.01 rows=1 width=8) (actual time=41.2..41.2 rows=1 loops=1)
  ->  Append  (cost=0.00..6210.00 rows=122000 width=0) (actual time=0.03..33.1 rows=121540 loops=1)
        ->  Seq Scan on orders_2024_11 orders_1  (actual time=0.02..8.9 rows=19980 loops=1)
              Filter: (created_at >= '2024-11-15' ...)
        ->  Seq Scan on orders_2024_12 orders_2  (actual time=0.02..14.0 rows=61200 loops=1)
        ->  Seq Scan on orders_2025_01 orders_3  (actual time=0.01..6.1 rows=40360 loops=1)
              Filter: (created_at < '2025-01-10')
Planning Time: 0.9 ms
Execution Time: 41.5 ms
```

**Reading it:**
- Three partitions survived (Nov, Dec, Jan); an `Append` stitches them. Only the boundary partitions carry a `Filter` (the middle December partition is fully inside the range, so no per-row filter needed — a nice pruning bonus).
- The other 21 partitions are pruned and absent.

### Execution-time pruning: `(never executed)` and `Subplans Removed`

```sql
PREPARE dashboard(timestamptz, timestamptz) AS
  SELECT count(*) FROM orders WHERE created_at >= $1 AND created_at < $2;
EXPLAIN (ANALYZE) EXECUTE dashboard('2024-06-01', '2024-07-01');
```

```
Aggregate  (actual time=7.1..7.1 rows=1 loops=1)
  ->  Append  (actual time=0.03..5.2 rows=39800 loops=1)
        Subplans Removed: 23
        ->  Seq Scan on orders_2024_06 orders_1  (actual time=0.02..3.1 rows=39800 loops=1)
              Filter: ((created_at >= $1) AND (created_at < $2))
Planning Time: 0.4 ms
Execution Time: 7.3 ms
```

**Reading it:**
- `Subplans Removed: 23` — the generic plan contained all 24 partitions, but at **execution time** 23 were pruned once `$1`/`$2` were bound. This is execution-time pruning; you only see it under `ANALYZE`.
- If instead you see every partition scanned with most showing `(never executed)`, pruning is working but via a slightly different path (per-partition subplan skipped).

### The anti-pattern: no pruning (wrong partition key for the query)

```sql
EXPLAIN (ANALYZE)
SELECT * FROM orders WHERE customer_id = 8842;   -- filtering on NON-partition-key
```

```
Append  (cost=... rows=96 ...) (actual time=0.4..38.9 rows=91 loops=1)
  ->  Index Scan using idx_orders_2023_01_cust on orders_2023_01  (actual rows=3 loops=1)
  ->  Index Scan using idx_orders_2023_02_cust on orders_2023_02  (actual rows=5 loops=1)
  ...  (ALL 24 partitions scanned)
  ->  Index Scan using idx_orders_2024_12_cust on orders_2024_12  (actual rows=4 loops=1)
Planning Time: 3.8 ms         ← note: 24-partition planning overhead
Execution Time: 39.4 ms
```

**Reading it:** No pruning possible — `customer_id` isn't the partition key. Every partition's index is probed. The per-partition indexes still make each probe cheap, but you pay Append + 24× planning overhead. This query would be *faster on an unpartitioned table with one `customer_id` index*. This is the classic "partitioning hurt me" plan.

---

## 8. Query Examples

### Example 1 — Basic: Create a Range-Partitioned Table and Query with Pruning

```sql
-- Create the partitioned parent (time-series orders)
CREATE TABLE orders (
    id           BIGINT        GENERATED ALWAYS AS IDENTITY,
    customer_id  BIGINT        NOT NULL,
    status       TEXT          NOT NULL DEFAULT 'pending',
    total_amount NUMERIC(12,2) NOT NULL,
    created_at   TIMESTAMPTZ   NOT NULL,
    PRIMARY KEY (id, created_at)          -- partition key must be in PK
) PARTITION BY RANGE (created_at);

-- Create monthly partitions
CREATE TABLE orders_2025_01 PARTITION OF orders
    FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');
CREATE TABLE orders_2025_02 PARTITION OF orders
    FOR VALUES FROM ('2025-02-01') TO ('2025-03-01');
CREATE TABLE orders_default PARTITION OF orders DEFAULT;   -- safety net

-- A pruning-friendly query: filters on the partition key
SELECT id, customer_id, total_amount
FROM orders
WHERE created_at >= '2025-02-01'
  AND created_at <  '2025-02-08'          -- one week: hits ONLY orders_2025_02
ORDER BY created_at DESC;
```

### Example 2 — Intermediate: LIST Partitioning + Per-Partition Index + Pruned Aggregate

```sql
-- Audit logs partitioned by log level for hot/cold separation
CREATE TABLE audit_logs (
    id         BIGINT       GENERATED ALWAYS AS IDENTITY,
    level      TEXT         NOT NULL,
    actor_id   BIGINT,
    action     TEXT         NOT NULL,
    logged_at  TIMESTAMPTZ  NOT NULL DEFAULT now(),
    PRIMARY KEY (id, level)
) PARTITION BY LIST (level);

CREATE TABLE audit_logs_critical PARTITION OF audit_logs FOR VALUES IN ('error', 'fatal');
CREATE TABLE audit_logs_normal   PARTITION OF audit_logs FOR VALUES IN ('info', 'warn');
CREATE TABLE audit_logs_debug    PARTITION OF audit_logs FOR VALUES IN ('debug', 'trace');

-- Parent index → cascades a B-tree to every partition (and future ones)
CREATE INDEX idx_audit_logged_at ON audit_logs (logged_at);

-- Pruned aggregate: only the 'error'/'fatal' partition is scanned
SELECT date_trunc('hour', logged_at) AS hr, count(*) AS errors
FROM audit_logs
WHERE level IN ('error', 'fatal')             -- prunes to audit_logs_critical only
  AND logged_at >= now() - INTERVAL '24 hours'
GROUP BY 1
ORDER BY 1;
```

### Example 3 — Production Grade: Partition-Wise Join + Execution-Time Pruning

```sql
-- SCENARIO:
--   orders:      ~400M rows, RANGE(created_at) monthly, 36 partitions (~11M rows each)
--   order_items: ~1.8B rows, RANGE(created_at) monthly, 36 partitions, SAME bounds
--   Both indexed on (order_id, created_at). enable_partitionwise_join = on.
--   PERF EXPECTATION: with matching bounds + partition key in the join, the planner
--   joins month-by-month; a single-month report touches 1 orders partition + 1
--   order_items partition, ~11M ⋈ ~50M within one month, not 400M ⋈ 1.8B.

SET enable_partitionwise_join = on;

SELECT
    date_trunc('day', o.created_at) AS day,
    count(DISTINCT o.id)            AS orders,
    sum(oi.quantity * oi.unit_price) AS revenue
FROM orders o
JOIN order_items oi
  ON oi.order_id   = o.id
 AND oi.created_at = o.created_at            -- carry partition key into join → partition-wise
WHERE o.created_at >= '2025-06-01'
  AND o.created_at <  '2025-07-01'           -- prune both sides to the June partitions
  AND o.status = 'completed'
GROUP BY 1
ORDER BY 1;
```

```
-- EXPLAIN (ANALYZE) sketch:
GroupAggregate  (actual time=812..980 rows=30 loops=1)
  ->  Sort (actual time=810..840 rows=1240000 loops=1)
        ->  Append  (actual time=0.1..690 rows=1240000 loops=1)
              Subplans Removed: 35            ← execution-time pruning to June only
              ->  Hash Join  (actual time=120..640 rows=1240000 loops=1)
                    Hash Cond: (oi.order_id = o.id AND oi.created_at = o.created_at)
                    ->  Seq Scan on order_items_2025_06 oi  (actual rows=49800000 ...)
                    ->  Hash  (actual rows=1120000 ...)
                          ->  Seq Scan on orders_2025_06 o
                                Filter: (status = 'completed')
Planning Time: 4.1 ms
Execution Time: 992 ms
```

**Why it's fast:** pruning restricts both tables to the single June partition (35 of 36 subplans removed), and the partition-wise join means the hash table is built from *one month* of orders (~1.1M rows), not the full 400M. Without partitioning this is a 400M ⋈ 1.8B join that would spill work_mem to disk and run for many minutes.

---

## 9. Wrong → Right Patterns

### Wrong 1: Partition key not in the PRIMARY KEY

```sql
-- WRONG: id alone as PK on a table you're partitioning by created_at
CREATE TABLE orders (
    id BIGINT PRIMARY KEY,
    customer_id BIGINT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL
) PARTITION BY RANGE (created_at);
-- EXACT ERROR:
-- ERROR:  unique constraint on partitioned table must include all
--         partitioning columns
-- DETAIL: PRIMARY KEY constraint on table "orders" lacks column "created_at"
--         which is part of the partition key.
```

**Why it's wrong at the execution level:** each leaf partition has its *own local* unique B-tree. PostgreSQL cannot enforce a global uniqueness guarantee on `id` alone without cross-partition coordination it does not do. It can only guarantee uniqueness if (uniqueness within each partition) + (partitions are value-disjoint on the key) together imply global uniqueness — which mathematically requires the partition key to be part of the constraint.

```sql
-- RIGHT: include the partition key in the PK
CREATE TABLE orders (
    id BIGINT GENERATED ALWAYS AS IDENTITY,
    customer_id BIGINT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (created_at);
-- Design note: id is now unique only WITH created_at. If you need id-only lookups,
-- keep created_at in your lookup path, or reconsider whether to partition at all.
```

### Wrong 2: Filtering on an expression the planner can't map to the partition bound

```sql
-- WRONG: wrapping the partition key defeats pruning
SELECT * FROM orders
WHERE date(created_at) = '2025-02-10';
-- date(created_at) is not the raw partition-key expression. Depending on the
-- function's mutability and the planner version, this may scan EVERY partition
-- (no pruning) even though it's logically one day. EXPLAIN shows all partitions.
```

**Why it's wrong at the execution level:** pruning compares partition **bounds** (raw `created_at` boundaries) against predicates on the raw partition key. `date(created_at) = X` is a predicate on a *derived* value; the planner won't rewrite it into `created_at >= X AND created_at < X+1` for pruning purposes, so it cannot exclude partitions.

```sql
-- RIGHT: express the filter as a half-open range on the raw partition key
SELECT * FROM orders
WHERE created_at >= '2025-02-10'
  AND created_at <  '2025-02-11';
-- Now the planner prunes to exactly the partition(s) covering Feb 10.
```

### Wrong 3: Forgetting to pre-create the next partition (no DEFAULT)

```sql
-- WRONG: partitions exist only through 2025-02, no DEFAULT, and it's now March 1
INSERT INTO orders (customer_id, total_amount, created_at)
VALUES (42, 99.00, '2025-03-01 00:00:01');
-- EXACT ERROR:
-- ERROR:  no partition of relation "orders" found for row
-- DETAIL: Partition key of the failing row contains (created_at) =
--         (2025-03-01 00:00:01).
-- → every INSERT fails at midnight; production write outage.
```

**Why it's wrong at the execution level:** tuple routing (`ExecFindPartition`) finds no leaf whose bound contains the row and, with no DEFAULT partition, has nowhere to put it, so the whole statement aborts.

```sql
-- RIGHT: automate ahead-of-time creation AND keep a DEFAULT as a backstop
CREATE TABLE orders_2025_03 PARTITION OF orders
    FOR VALUES FROM ('2025-03-01') TO ('2025-04-01');
-- plus a permanent safety net:
CREATE TABLE orders_default PARTITION OF orders DEFAULT;
-- Best practice: a scheduled job (pg_partman or cron) creates N months ahead.
```

### Wrong 4: Expecting HASH partitioning to prune a range query

```sql
-- WRONG: sessions partitioned by HASH(user_id); expecting a range scan to prune
SELECT * FROM sessions WHERE user_id BETWEEN 1000 AND 2000;
-- Scans ALL hash partitions. Hashing scatters 1000..2000 across every partition,
-- so no partition can be excluded. You lost pruning AND pay Append overhead.
```

**Why it's wrong at the execution level:** HASH pruning only works for **equality** on the key, because only then is the hash (and therefore the target partition) computable. A range predicate maps to no single hash bucket, so every partition survives.

```sql
-- RIGHT (option A): if you need range pruning, use RANGE partitioning on that key
--   PARTITION BY RANGE (created_at)   -- and query created_at ranges
-- RIGHT (option B): if HASH is correct for your workload, only expect equality pruning
SELECT * FROM sessions WHERE user_id = 1500;   -- prunes to exactly one partition
```

### Wrong 5: Attaching a large populated table without a pre-existing CHECK

```sql
-- WRONG: attach a 50M-row table; PostgreSQL scans all 50M rows under ACCESS EXCLUSIVE
ALTER TABLE orders ATTACH PARTITION orders_2025_03
    FOR VALUES FROM ('2025-03-01') TO ('2025-04-01');
-- Locks the parent, full-scans orders_2025_03 to validate the bound → long stall.
```

**Why it's wrong at the execution level:** ATTACH must prove every existing row satisfies the new bound. Without evidence, it performs a validating `Seq Scan` while holding `ACCESS EXCLUSIVE` on the parent — blocking all access to the whole partitioned table for the duration.

```sql
-- RIGHT: add a matching CHECK first so ATTACH proves the bound without scanning
ALTER TABLE orders_2025_03
    ADD CONSTRAINT ck_2025_03
    CHECK (created_at >= '2025-03-01' AND created_at < '2025-04-01');
ALTER TABLE orders ATTACH PARTITION orders_2025_03
    FOR VALUES FROM ('2025-03-01') TO ('2025-04-01');   -- instant; skips the scan
ALTER TABLE orders_2025_03 DROP CONSTRAINT ck_2025_03;  -- redundant now, clean up
```

---

## 10. Performance Profile

### Scaling Behavior

| Table size | Unpartitioned | Partitioned (by time), query filters partition key |
|-----------|---------------|-----------------------------------------------------|
| 1M rows | Fine; index handles it | Net negative — partition overhead > benefit |
| 10M rows | Usually fine | Marginal; only if strong retention need |
| 100M rows | Index bloat, slow vacuum, slow archival DELETE | Clear win: pruning + fast per-partition vacuum |
| 1B+ rows | Vacuum can't keep up; wraparound risk | Essential: only hot partition needs frequent vacuum |

### Memory and CPU

- **Pruning saves I/O, not just CPU.** A pruned partition's heap and index pages are never read into the buffer pool — the biggest single saving. A "last 7 days" query on a 2-year daily-partitioned table reads ~1% of the data.
- **Append/MergeAppend overhead is per-surviving-partition.** Each surviving partition adds a scan node. Hundreds of surviving partitions add real per-node executor overhead; keep the number of partitions a query touches small.
- **Planning time grows with total partition count.** The planner locks and considers partitions during planning. Thousands of partitions can push planning time into tens of milliseconds even when execution prunes to one — a hidden tax on high-QPS OLTP. Practical ceiling: prefer low hundreds of partitions, not thousands.
- **Partition-wise join/aggregate trade planning cost for execution parallelism.** Off by default precisely because they enlarge the plan search space. Turn them on for large analytic joins where the payoff dominates.

### Index Interactions

- Per-partition B-trees are smaller and often one level shallower → fewer page reads per lookup and a smaller hot index working set.
- INSERT maintains only the current partition's indexes → the write hot path keeps a small index in cache; historical indexes stay cold and untouched.
- A non-partition-key lookup (`WHERE customer_id = ?`) probes **every** partition's index. This is N index scans; each is cheap but the total can exceed a single global index on an unpartitioned table. If such lookups are frequent, either don't partition, or ensure the query also constrains the partition key.

### Optimization Techniques Specific to Partitioning

1. **Match your partition key to your dominant query's WHERE clause.** This is the whole game. No pruning = no benefit.
2. **Right-size the partition.** Aim for partitions in the low single-digit GB / ~1–10M rows each. Too-fine (daily for years) → thousands of partitions and planning bloat; too-coarse → weak pruning and slow vacuum per partition.
3. **Use `DETACH CONCURRENTLY` + `DROP`** for retention instead of `DELETE`. Constant-time, no bloat, negligible WAL.
4. **Pre-create partitions and keep a DEFAULT** empty as a backstop.
5. **Enable `enable_partitionwise_join`/`_aggregate`** for large analytic workloads on matching-bound partitioned tables.
6. **BRIN indexes pair beautifully with time-range partitions** — within a partition, rows are naturally clustered by time, so a tiny BRIN index on `created_at` gives near-index-scan performance at a fraction of the size.

---

## 11. Node.js Integration

### 11.1 Pruning-friendly range query with `pg`

```javascript
import pg from 'pg';
const { Pool } = pg;
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

// Always pass a half-open range on the PARTITION KEY so the planner can prune.
async function getOrdersInRange(fromISO, toISO) {
  const { rows } = await pool.query(
    `SELECT id, customer_id, total_amount, created_at
       FROM orders
      WHERE created_at >= $1
        AND created_at <  $2          -- half-open range → clean partition pruning
      ORDER BY created_at DESC`,
    [fromISO, toISO]
  );
  return rows;
}
// getOrdersInRange('2025-02-01', '2025-02-08')  → touches only orders_2025_02
```

### 11.2 Ahead-of-time partition creation (scheduled maintenance job)

```javascript
// Run daily/weekly. Creates the NEXT month's partition if it doesn't exist.
// Identifiers cannot be parameterized ($1) — build them safely from validated dates.
async function ensureNextMonthPartition() {
  const now = new Date();
  const start = new Date(Date.UTC(now.getUTCFullYear(), now.getUTCMonth() + 1, 1));
  const end   = new Date(Date.UTC(now.getUTCFullYear(), now.getUTCMonth() + 2, 1));
  const startStr = start.toISOString().slice(0, 10);
  const endStr   = end.toISOString().slice(0, 10);
  const suffix   = `${start.getUTCFullYear()}_${String(start.getUTCMonth() + 1).padStart(2, '0')}`;
  const partName = `orders_${suffix}`;          // e.g. orders_2025_03

  // partName/startStr/endStr are derived from Date math, not user input → safe to interpolate.
  await pool.query(
    `CREATE TABLE IF NOT EXISTS ${partName}
       PARTITION OF orders
       FOR VALUES FROM ('${startStr}') TO ('${endStr}')`
  );
  return partName;
}
```

### 11.3 Retention rotation: DETACH CONCURRENTLY then DROP

```javascript
// DETACH CONCURRENTLY cannot run inside a transaction block — use autocommit.
async function retirePartition(partitionName) {
  const client = await pool.connect();
  try {
    // Step 1: O(1) metadata detach, no ACCESS EXCLUSIVE on the parent.
    await client.query(`ALTER TABLE orders DETACH PARTITION ${partitionName} CONCURRENTLY`);
    // (optional) Step 2: archive partitionName to cold storage here...
    // Step 3: free the space.
    await client.query(`DROP TABLE ${partitionName}`);
  } finally {
    client.release();
  }
}
// vs DELETE FROM orders WHERE created_at < ... → scans/bloats; this is instant.
```

### 11.4 Verifying pruning from the app (guardrail in tests)

```javascript
// Assert in CI that a hot query still prunes — catches regressions where someone
// adds a function wrapper around the partition key and silently kills pruning.
async function assertPrunes(sql, params, maxScannedPartitions = 2) {
  const { rows } = await pool.query(`EXPLAIN (FORMAT JSON) ${sql}`, params);
  const plan = JSON.stringify(rows[0]['QUERY PLAN']);
  const scanned = (plan.match(/"Relation Name":"orders_/g) || []).length;
  if (scanned > maxScannedPartitions) {
    throw new Error(`Pruning regression: ${scanned} partitions scanned (max ${maxScannedPartitions})`);
  }
}
```

**Do ORMs handle partitioning?** Mostly transparently for **reads and writes** — because the parent looks like an ordinary table, `SELECT`/`INSERT`/`UPDATE` through any ORM just work and pruning happens server-side as long as your ORM-generated `WHERE` includes the partition key. What ORMs do **not** manage is the DDL lifecycle: creating/attaching/detaching partitions, the partition-key-in-PK requirement, and retention rotation are all outside the ORM and must be done in raw SQL / migrations / a scheduler.

---

## 12. ORM Comparison

### Prisma

**Can Prisma do partitioning?** — Not natively. Prisma's schema language has **no** concept of `PARTITION BY`, partition bounds, or ATTACH/DETACH. You partition via raw SQL migrations and Prisma treats the parent as a normal model.

```prisma
// schema.prisma — Prisma sees only a normal model; note the composite PK it needs
model Order {
  id          BigInt   @default(autoincrement())
  customerId  BigInt
  status      String   @default("pending")
  totalAmount Decimal  @db.Decimal(12, 2)
  createdAt   DateTime @db.Timestamptz
  @@id([id, createdAt])          // composite PK to satisfy partition-key-in-PK
  @@map("orders")
}
```

```typescript
// The partitioning DDL lives in a raw migration Prisma cannot generate:
await prisma.$executeRawUnsafe(`
  CREATE TABLE orders_2025_02 PARTITION OF orders
    FOR VALUES FROM ('2025-02-01') TO ('2025-03-01')
`);

// Reads/writes are transparent and DO prune, as long as createdAt is in the filter:
const rows = await prisma.order.findMany({
  where: { createdAt: { gte: new Date('2025-02-01'), lt: new Date('2025-02-08') } },
});
```

**Where it breaks:** `prisma migrate` wants the parent to be an ordinary table and can get confused by partitions it didn't create (drift detection). The composite `@@id` propagates into every relation — a `Payment` referencing `Order` now needs both `orderId` and `createdAt`. `prisma db pull` won't reconstruct partition definitions.

**Verdict:** Use Prisma for the app-level reads/writes; manage all partition DDL in hand-written SQL migrations marked as such so Prisma leaves them alone.

---

### Drizzle ORM

**Can Drizzle do partitioning?** — Partially. Drizzle can *declare* a partitioned parent and composite PKs in its schema, but partition children and ATTACH/DETACH are raw SQL.

```typescript
import { pgTable, bigint, text, numeric, timestamp, primaryKey } from 'drizzle-orm/pg-core';
import { sql } from 'drizzle-orm';

export const orders = pgTable('orders', {
  id: bigint('id', { mode: 'bigint' }).generatedAlwaysAsIdentity(),
  customerId: bigint('customer_id', { mode: 'bigint' }).notNull(),
  status: text('status').notNull().default('pending'),
  totalAmount: numeric('total_amount', { precision: 12, scale: 2 }).notNull(),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull(),
}, (t) => ({
  pk: primaryKey({ columns: [t.id, t.createdAt] }),   // partition key in PK
}));
// PARTITION BY clause + child partitions via raw migration:
//   sql`... PARTITION BY RANGE (created_at)`  (append to CREATE TABLE)
//   sql`CREATE TABLE orders_2025_02 PARTITION OF orders FOR VALUES ...`

// Query prunes normally when createdAt is constrained:
await db.select().from(orders)
  .where(sql`${orders.createdAt} >= '2025-02-01' AND ${orders.createdAt} < '2025-02-08'`);
```

**Where it breaks:** the `PARTITION BY` clause on the CREATE TABLE isn't a first-class builder option — you supply it via a custom/raw migration. Partition rotation is entirely your own SQL.

**Verdict:** Drizzle's explicit-SQL philosophy fits partitioning well — declare the parent + composite PK in schema, do partition lifecycle in raw `sql` migrations.

---

### Sequelize

**Can Sequelize do partitioning?** — No native support. Define the model normally with a composite PK and run partition DDL through `sequelize.query()` or migrations.

```javascript
const Order = sequelize.define('Order', {
  id:          { type: DataTypes.BIGINT, autoIncrement: true, primaryKey: true },
  createdAt:   { type: DataTypes.DATE,   primaryKey: true },   // part of composite PK
  customerId:  { type: DataTypes.BIGINT, allowNull: false },
  totalAmount: { type: DataTypes.DECIMAL(12, 2), allowNull: false },
}, { tableName: 'orders', timestamps: false });

// Partition DDL — raw:
await sequelize.query(`
  CREATE TABLE orders_2025_02 PARTITION OF orders
    FOR VALUES FROM ('2025-02-01') TO ('2025-03-01')
`);
```

**Where it breaks:** Sequelize's `sync()` will try to `CREATE TABLE orders (...)` without `PARTITION BY` and cannot express it — never use `sync()` for partitioned tables; use migrations. The composite PK confuses `belongsTo`/`hasMany` associations that assume a single-column key.

**Verdict:** Model + queries via Sequelize; every bit of partition DDL via raw queries in migrations. Avoid `sync()`.

---

### TypeORM

**Can TypeORM do partitioning?** — No dedicated API, but entities support composite primary keys, and you drive partition DDL through `queryRunner.query()` in migrations.

```typescript
@Entity('orders')
export class Order {
  @PrimaryColumn({ type: 'bigint' })
  id: string;

  @PrimaryColumn({ type: 'timestamptz' })   // composite PK includes partition key
  createdAt: Date;

  @Column({ type: 'bigint' }) customerId: string;
  @Column({ type: 'numeric', precision: 12, scale: 2 }) totalAmount: string;
}
```

```typescript
// In a migration:
export class CreateOrders implements MigrationInterface {
  async up(q: QueryRunner) {
    await q.query(`
      CREATE TABLE orders (
        id BIGINT GENERATED ALWAYS AS IDENTITY,
        customer_id BIGINT NOT NULL,
        total_amount NUMERIC(12,2) NOT NULL,
        created_at TIMESTAMPTZ NOT NULL,
        PRIMARY KEY (id, created_at)
      ) PARTITION BY RANGE (created_at)`);
    await q.query(`CREATE TABLE orders_2025_02 PARTITION OF orders
      FOR VALUES FROM ('2025-02-01') TO ('2025-03-01')`);
  }
  async down(q: QueryRunner) { await q.query(`DROP TABLE orders`); }
}
```

**Where it breaks:** `synchronize: true` cannot produce `PARTITION BY` and will fight you — disable it for partitioned tables. `@Generated`/`@PrimaryGeneratedColumn` on a single column collides with the required composite PK.

**Verdict:** Use TypeORM migrations with raw `query()` for the partitioned DDL; keep `synchronize` off.

---

### Knex.js

**Can Knex do partitioning?** — Only through `knex.raw()`. The schema builder has no partition primitives, but as the most SQL-transparent tool it's the least friction.

```javascript
// Create partitioned parent + a child — raw, but clean:
await knex.raw(`
  CREATE TABLE orders (
    id BIGINT GENERATED ALWAYS AS IDENTITY,
    customer_id BIGINT NOT NULL,
    total_amount NUMERIC(12,2) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (id, created_at)
  ) PARTITION BY RANGE (created_at)
`);
await knex.raw(`
  CREATE TABLE orders_2025_02 PARTITION OF orders
    FOR VALUES FROM ('2025-02-01') TO ('2025-03-01')
`);

// Normal query-builder reads prune fine when created_at is constrained:
const rows = await knex('orders')
  .where('created_at', '>=', '2025-02-01')
  .andWhere('created_at', '<', '2025-02-08')
  .select('id', 'customer_id', 'total_amount');

// Rotation:
await knex.raw(`ALTER TABLE orders DETACH PARTITION orders_2024_02 CONCURRENTLY`);
await knex.schema.dropTable('orders_2024_02');
```

**Where it breaks:** everything partition-specific is raw; `knex.schema.createTable` can't emit `PARTITION BY`. But the query builder's normal `.where()` output prunes without any special handling.

**Verdict:** Knex is the smoothest fit — normal builder for queries, `knex.raw()` for the (unavoidable) DDL.

---

### ORM Summary Table

| ORM | Declare partitioned parent | Partition DDL / rotation | Composite PK support | Verdict |
|-----|---------------------------|--------------------------|----------------------|---------|
| Prisma | No `PARTITION BY` | Raw `$executeRaw` migrations | `@@id([...])` | App layer only; DDL in SQL |
| Drizzle | PK yes, `PARTITION BY` via raw | Raw `sql` migrations | `primaryKey({columns})` | Good fit, explicit SQL |
| Sequelize | No; avoid `sync()` | `sequelize.query()` | Multi `primaryKey:true` | Migrations only |
| TypeORM | No; disable `synchronize` | `queryRunner.query()` | `@PrimaryColumn` ×2 | Migrations only |
| Knex | `knex.raw()` | `knex.raw()` + builder drop | raw | Most transparent |

**Universal truth:** every ORM handles partitioned-table *queries and DML* transparently (pruning is a server-side planner concern), and **none** manages the partition *lifecycle*. Own the DDL in raw SQL migrations plus a scheduled job for rotation.

---

## 13. Practice Exercises

### Exercise 1 — Basic

You have a `sessions` table that will hold ~200M rows and you want to partition it by `created_at` monthly.

1. Write the `CREATE TABLE sessions (...) PARTITION BY RANGE (created_at)` statement. It needs `id`, `user_id BIGINT`, `data JSONB`, `created_at TIMESTAMPTZ`. Choose a correct primary key.
2. Create the January and February 2025 partitions.
3. Write a query that returns all of a user's sessions in the first week of February and prunes to a single partition.

```sql
-- Write your query here
```

---

### Exercise 2 — Medium

Given `orders(id, customer_id, status, total_amount, created_at)` partitioned by `RANGE(created_at)` monthly, and `order_items(id, order_id, created_at, product_id, quantity, unit_price)` partitioned by `RANGE(created_at)` with **identical** monthly bounds.

Write a query (combining pruning from Topic 08 WHERE-filtering and JOIN mechanics from Topic 11) that returns, for **March 2025 completed orders only**, per `product_id`: distinct order count, units sold, and revenue. Ensure it (a) prunes both tables to March and (b) is eligible for a partition-wise join. State which `SET` you'd issue and why.

```sql
-- Write your query here
```

---

### Exercise 3 — Hard (production simulation, naive answer is wrong)

Your `orders` table is partitioned by `RANGE(created_at)` monthly. A teammate wrote this "last 30 days revenue by customer" query and it scans **every** partition:

```sql
SELECT customer_id, SUM(total_amount)
FROM orders
WHERE created_at::date >= CURRENT_DATE - 30
GROUP BY customer_id;
```

1. Explain, at the planner level, *why* it scans every partition.
2. Rewrite it so it prunes to only the ~2 relevant partitions.
3. The report also must include customers with **zero** orders in the window (showing 0). Does that change your pruning story? Write that version.

```sql
-- Write your query here
```

---

### Exercise 4 — Interview Simulation

A 900M-row `audit_logs` table (single unpartitioned heap) has a nightly `DELETE FROM audit_logs WHERE logged_at < now() - interval '90 days'` that now takes 4 hours, bloats the table, and starves autovacuum. Design a partitioning scheme and a rotation process that turns the nightly purge into a sub-second operation. Specify: partition strategy and key, partition granularity, the PK, how new partitions get created, how old data is removed, and the one query pattern that would make your scheme a mistake.

```sql
-- Write your design (DDL + rotation SQL) here
```

---

## 14. Interview Questions

### Q1 — What actually makes a partitioned query faster? Is it the index?

**The question:** "You partitioned a 200M-row table by month and a dashboard query got 20× faster. Explain precisely what changed."

**Junior answer:** "Partitioning makes tables smaller so queries are faster because there's less data."

**Principal answer:** "It's **partition pruning**, and it happens at plan time (or execution time for bind params). The planner compares the query's `WHERE created_at >= X AND < Y` against each partition's stored bound (`relpartbound`) and *removes* partitions that provably can't match from the plan entirely — their heap and index pages are never read into the buffer pool. That's the win: I/O elimination, not merely smaller tables. It only works because the predicate references the **partition key**. If the dashboard filtered on a non-key column, pruning couldn't fire and I'd actually be slower than unpartitioned due to Append overhead across every partition. Secondary wins: each leaf's B-tree is smaller and shallower, and autovacuum handles each partition independently so bloat stays local to the hot partition."

**Interviewer follow-up:** "How would you *prove* pruning is happening, and distinguish plan-time from execution-time pruning?"
*(Expected: plan-time → pruned partitions absent from `EXPLAIN` output entirely; execution-time → `EXPLAIN ANALYZE` shows `Subplans Removed: N` or `(never executed)` for prepared statements / generic plans with bind params.)*

---

### Q2 — Why must the partition key be in the primary key, and what does it cost you?

**The question:** "Why can't I have `id BIGINT PRIMARY KEY` on a table partitioned by `created_at`?"

**Junior answer:** "PostgreSQL just requires it; you add `created_at` to the PK and move on."

**Principal answer:** "Each partition has its own **local** unique B-tree — there is no global cross-partition unique index. PostgreSQL can only guarantee a value is globally unique if uniqueness-within-each-partition plus the partitions being value-disjoint on the key together *imply* global uniqueness, and that's only true when the partition key is part of the constraint. So the PK must include `created_at`. The cost is real: `id` is now unique only *together with* `created_at`, so two rows with the same `id` and different `created_at` are legal — a hazard if `id` is treated as a global handle. And **foreign keys pointing at this table** must reference the full `(id, created_at)`, so referencing rows must also carry `created_at`. That often makes inbound FKs so awkward that teams drop DB-level referential integrity — which is itself a decision to make consciously."

**Interviewer follow-up:** "Given that, would you still partition an OLTP table that lots of other tables FK into?"
*(Expected: weigh the retention/pruning benefit against losing clean inbound FKs; often the answer is to partition the append-heavy log/event tables, not the core entity tables.)*

---

### Q3 — DELETE vs DETACH for retention.

**The question:** "Nightly `DELETE FROM events WHERE created_at < now() - '90 days'` takes hours. Fix it."

**Junior answer:** "Add an index on `created_at` and batch the deletes in chunks."

**Principal answer:** "Batching helps the lock story but not the fundamental problem: `DELETE` is O(rows) — it marks every tuple dead (MVCC), writes a WAL record per row, and leaves the pages bloated until VACUUM reclaims them, which on a big append-only table struggles to keep up. The right fix is to **partition by time** (say monthly) and rotate: `ALTER TABLE events DETACH PARTITION events_2024_10 CONCURRENTLY` followed by `DROP TABLE events_2024_10`. DETACH is a catalog metadata change — O(1), negligible WAL, no bloat — regardless of how many rows the partition holds. `CONCURRENTLY` avoids taking `ACCESS EXCLUSIVE` on the parent so the rest of the table stays online. Retention goes from hours to milliseconds."

**Interviewer follow-up:** "What breaks if you forget to pre-create next month's partition, and how do you defend against it?"
*(Expected: INSERTs hit `no partition of relation found for row` and error out → write outage; defend with an ahead-of-time creation job (pg_partman/cron creating N months ahead) plus a DEFAULT partition backstop, kept empty in steady state.)*

---

### Q4 — Partition-wise joins.

**The question:** "You join two large tables both partitioned on the same key with the same bounds, but `EXPLAIN` shows one giant hash join over the full Append, not a per-partition join. Why?"

**Junior answer:** "Maybe the bounds don't match."

**Principal answer:** "First thing to check is `enable_partitionwise_join` — it's **off by default** because it enlarges the planner's search space and costs planning time proportional to partition count. Turn it on (`SET enable_partitionwise_join = on`). Then the *requirements* must hold: both tables partitioned with identical strategy and bounds, and the **partition key(s) present in the join clause** — so I have to join `ON a.pkcol = b.pkcol` (e.g. carry `created_at` into the join `AND oi.created_at = o.created_at`), not just `ON oi.order_id = o.id`. When those hold, the planner joins partition-by-partition — N small independent joins each fitting in work_mem — instead of one massive join that spills to disk. Same reasoning applies to `enable_partitionwise_aggregate` for `GROUP BY` on the partition key."

**Interviewer follow-up:** "Why are these off by default if they're so good?"
*(Expected: planning cost — the combinatorial join/aggregate search grows with partition count; on tables with many partitions or high-QPS simple queries the extra planning time isn't worth it, so PostgreSQL makes you opt in.)*

---

## 15. Mental Model Checkpoint

1. You partition `orders` by `RANGE(created_at)` monthly, then run `SELECT * FROM orders WHERE customer_id = 42`. How many partitions does the planner scan, and is this query faster or slower than on an equivalent unpartitioned table? Why?

2. A row arrives with `created_at = '2025-02-01 00:00:00'` and you have partitions `[2025-01-01, 2025-02-01)` and `[2025-02-01, 2025-03-01)`. Which partition receives it, and what property of RANGE bounds determines this?

3. Explain the difference between what `EXPLAIN` shows for plan-time pruning versus what `EXPLAIN ANALYZE` shows for execution-time pruning. Which one produces `Subplans Removed: N`?

4. You HASH-partition `sessions` by `user_id` into 8 partitions. Which of these prune, and to how many partitions each: `WHERE user_id = 500`, `WHERE user_id BETWEEN 1 AND 1000`, `WHERE user_id IN (5, 6, 7)`?

5. Why does attaching a large *populated* table as a partition potentially stall the whole parent table, and what one preparatory step makes ATTACH instant?

6. You need a foreign key from `payments` to `orders`, but `orders` is partitioned by `created_at` with PK `(id, created_at)`. What must `payments` now store, and what does this imply about whether partitioning `orders` was a good idea?

7. At what point does adding *more* partitions start to hurt rather than help, and which two times (plan vs execution) pay that cost differently?

---

## 16. Quick Reference Card

```sql
-- ── CREATE A PARTITIONED TABLE ───────────────────────────────────────────
CREATE TABLE orders (
    id BIGINT GENERATED ALWAYS AS IDENTITY,
    created_at TIMESTAMPTZ NOT NULL,
    ...,
    PRIMARY KEY (id, created_at)          -- partition key MUST be in every UNIQUE/PK
) PARTITION BY RANGE (created_at);        -- RANGE | LIST | HASH

-- ── ADD PARTITIONS ───────────────────────────────────────────────────────
-- RANGE: [lower inclusive, upper EXCLUSIVE)
CREATE TABLE orders_2025_02 PARTITION OF orders
    FOR VALUES FROM ('2025-02-01') TO ('2025-03-01');
-- LIST
CREATE TABLE ev_us PARTITION OF ev FOR VALUES IN ('us-east','us-west');
-- HASH
CREATE TABLE s0 PARTITION OF s FOR VALUES WITH (MODULUS 4, REMAINDER 0);
-- DEFAULT (RANGE/LIST only) — catches unmatched rows
CREATE TABLE orders_default PARTITION OF orders DEFAULT;
-- Unbounded ends
FOR VALUES FROM (MINVALUE) TO ('2020-01-01');
FOR VALUES FROM ('2030-01-01') TO (MAXVALUE);

-- ── ROTATION (time-series retention) ─────────────────────────────────────
ALTER TABLE orders DETACH PARTITION orders_2024_01 CONCURRENTLY;  -- O(1), no ACCESS EXCL
DROP TABLE orders_2024_01;                                        -- frees space instantly
-- ATTACH without a validation scan: add a matching CHECK first
ALTER TABLE t ADD CONSTRAINT ck CHECK (created_at >= 'lo' AND created_at < 'hi');
ALTER TABLE orders ATTACH PARTITION t FOR VALUES FROM ('lo') TO ('hi');

-- ── INDEXES ──────────────────────────────────────────────────────────────
CREATE INDEX idx ON orders (customer_id);   -- cascades to all + future partitions
-- CONCURRENTLY on parent NOT allowed → build per-partition CONCURRENTLY, then:
CREATE INDEX idx ON ONLY orders (customer_id);
ALTER INDEX idx ATTACH PARTITION child_idx;  -- parent index valid once all attached

-- ── PRUNING KNOBS ────────────────────────────────────────────────────────
SET enable_partition_pruning = on;          -- default on; drives declarative pruning
SET enable_partitionwise_join = on;          -- OFF by default; needs matching bounds + key in join
SET enable_partitionwise_aggregate = on;     -- OFF by default; GROUP BY on partition key
-- constraint_exclusion is for OLD inheritance, NOT declarative partitions.
```

**Rules of thumb**
- Partition key MUST appear in the query's `WHERE` or you get **zero** pruning + Append tax.
- RANGE bounds are `[lower, upper)` — upper is exclusive.
- HASH prunes **equality only**, never ranges.
- Partition key must be in every PK/UNIQUE constraint → inbound FKs must carry it too.
- `DETACH`/`DROP` for retention, never `DELETE` on the whole range.
- Keep partition count in the low hundreds; thousands inflate planning time.
- Partition when: big (100M+), time-based retention, queries filter the key. Don't when: small, queries don't filter the key, you need id-only uniqueness/simple FKs.

**Interview one-liners**
- "The win is pruning — I/O elimination at plan time — not smaller tables per se."
- "No filter on the partition key = no pruning = slower than unpartitioned."
- "Local per-partition unique indexes are *why* the partition key must be in the PK."
- "DETACH is O(1) metadata; DELETE is O(rows) + WAL + bloat."
- "Partition-wise join/aggregate are off by default — enable them and put the key in the join."

---

## Connected Topics

- **Topic 69 — pg_stat_statements**: How you *find* the slow full-table query that motivates partitioning, and how you verify per-query improvement after partitioning by comparing mean/total exec time.
- **Topic 71 — Schema Design Patterns**: Where partitioning fits among sharding, denormalization, and time-series design; choosing the partition key is a schema-design decision.
- **Topic 08 — WHERE Filtering**: Pruning is entirely driven by the `WHERE` predicate on the partition key — the mechanics of predicate form (half-open ranges vs wrapped expressions) decide whether pruning fires.
- **Topic 11 — INNER JOIN in Depth**: The join algorithms (Hash/Merge/Nested Loop) that partition-wise joins parallelize per partition; matching-bound joins turn one big Hash Join into N small ones.
- **Topic 20 — GROUP BY Fundamentals**: Partition-wise aggregation computes `GROUP BY` per partition — the same collapse mechanics, distributed across leaves.
- **Internals — MVCC & VACUUM**: Per-partition visibility maps and autovacuum are the reason partitioning tames bloat and wraparound risk on huge append-heavy tables.
- **Internals — B-tree Indexes**: Local per-partition B-trees (smaller, shallower) explain both the index-maintenance win and the partition-key-in-PK constraint.
- **Internals — The Query Planner**: Plan-time vs execution-time pruning, `Append`/`MergeAppend`, and `Subplans Removed` are all planner/executor behaviors you read directly in `EXPLAIN`.
