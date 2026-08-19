# 59 — Partitioning
## Phase: Denormalisation & Scale

---

## ELI5 — The Simple Analogy

One enormous filing cabinet holding every invoice your company has ever issued. Ten years, forty million pieces of paper, one drawer.

Finding last week's invoices means walking past ten years of paper. Throwing away 2016's invoices means pulling out four million sheets one at a time. And when the cabinet finally fills up, you cannot move it — it weighs a tonne.

**Partitioning is buying one cabinet per month.**

Now three things change, and only three:

1. ★ **Finding last week's invoices means opening one drawer.** You skip the other 119 entirely — but *only if the label on the drawer tells you what's inside*. If you filed by invoice number and you're searching by date, the labels are useless and you still open all 120.

2. ★ **Deleting 2016 means throwing out twelve cabinets.** Not four million sheets — twelve objects. This is the reason partitioning exists, and it is worth more than the query speedup.

3. ★ **Every cabinet is individually manageable.** You can index one, vacuum one, or move one to the basement without touching the others.

And the part people miss: ★ **you now have 120 cabinets to keep track of.** Somebody must buy next month's cabinet before the first of the month. If nobody does, the filing clerk has nowhere to put the paper and **stops working entirely.**

---

## Where this fits in the big picture

```
   47 VACUUM — "retention is DROP PARTITION, not DELETE"
   58 read replicas — the nightly DELETE that caused 24 min of lag
                          │
                          ▼
        ┌──────────────────────────────────────────────┐
        │ 59 PARTITIONING ← YOU ARE HERE               │
        │ ★ one table, many physical pieces, ONE server│
        └────────────────────┬─────────────────────────┘
                             ▼
              60 sharding (★ many SERVERS — a different problem)
              61 hot rows · 64 backups
```

★ **The distinction that matters most: partitioning splits a table across *files on one server*. Sharding splits data across *servers*.** Partitioning is a local, transactional, fully-supported feature. Sharding is a distributed system with all of Topic 51's problems. People conflate them constantly, and the conflation is expensive.

---

## What is this?

Declarative partitioning (PostgreSQL 10+, mature from 12): one logical table whose rows physically live in separate child tables, chosen by a **partition key**.

```sql
CREATE TABLE events (
  id          bigserial,
  tenant_id   bigint      NOT NULL,
  payload     jsonb       NOT NULL,
  created_at  timestamptz NOT NULL,
  PRIMARY KEY (id, created_at)      -- ★ the key MUST include the partition column
) PARTITION BY RANGE (created_at);

CREATE TABLE events_2026_08 PARTITION OF events
  FOR VALUES FROM ('2026-08-01') TO ('2026-09-01');
```

**Three strategies:**

| | `RANGE` | `LIST` | `HASH` |
|---|---|---|---|
| Key | ordered (dates, ids) | discrete values | any |
| Example | one per month | one per region | 16 buckets |
| ★ Enables `DROP` retention | ★ **yes** | sometimes | ★ **no** |
| Pruning on `=` | yes | yes | yes |
| Pruning on `BETWEEN` | ★ **yes** | no | ★ **no** |
| Use for | ★ **time-series** | tenancy, geography | ★ spreading write contention |

★ **The one thing to internalise:** partitioning is not primarily a performance feature. It is a **manageability** feature, and its single biggest win is that dropping a partition is `O(1)` while deleting its rows is `O(n)` plus dead tuples plus index churn plus WAL plus replica lag.

---

## Why does it matter for a backend developer?

```
 ★ BECAUSE ONE SPECIFIC OPERATION IS THOUSANDS OF TIMES CHEAPER,
   AND IT IS AN OPERATION EVERY SYSTEM PERFORMS.

 RETENTION — deleting old data
   DELETE FROM events WHERE created_at < now() - interval '90 days';
     41,000,000 rows
     ⇒ ★ 41M dead tuples ⇒ autovacuum work for hours
     ⇒ ★ index churn on every index
     ⇒ ★ 18 GB of WAL ⇒ 24 minutes of replica lag (Topic 58)
     ⇒ ★ NO DISK SPACE RETURNED (Topic 47)
     ⇒ measured: 41 minutes, and it often doesn't finish

   DROP TABLE events_2026_05;
     ⇒ ★ 12 milliseconds
     ⇒ ★ zero dead tuples, zero index churn
     ⇒ ★ ~2 KB of WAL
     ⇒ ★ disk returned to the OS immediately

 ★ THAT IS A FACTOR OF ~200,000, AND IT IS THE ENTIRE REASON
   PARTITIONING EXISTS.
```

And the failure mode that makes it dangerous:

```
 ★ NOBODY CREATED NEXT MONTH'S PARTITION.
   01 September, 00:00:00 IST:
   ERROR: no partition of relation "events" found for row
   DETAIL: Partition key of the failing row contains
           (created_at) = (2026-09-01 00:00:00+05:30).
   ⇒ ★ EVERY INSERT FAILS. Total write outage, at midnight,
     on a schedule you set months earlier and forgot.
 ⇒ ★ THIS IS THE #1 PARTITIONING INCIDENT, AND IT IS 100%
   PREVENTABLE WITH A DEFAULT PARTITION AND AN ALERT.
```

---

## The physical reality

### What a partitioned table actually is

```
 ★ THE PARENT IS AN EMPTY SHELL. It has no storage at all.

   SELECT relname, relkind, relfilenode, relpages
     FROM pg_class WHERE relname LIKE 'events%';

   relname          | relkind | relfilenode | relpages
   -----------------+---------+-------------+----------
   events           | ★ p     |         ★ 0 |        0    ← no storage
   events_2026_07   | r       |       88412 |    41204
   events_2026_08   | r       |       88413 |    38820

 ⇒ relkind 'p' = a partitioned table. ★ relfilenode 0 means there
   is no file. Every row lives in a child.
 ⇒ ★ EACH CHILD IS A COMPLETE, INDEPENDENT TABLE: its own heap,
   its own indexes, its own visibility map, its own free space map,
   ★ ITS OWN autovacuum schedule and ITS OWN statistics.
 ⇒ ★ CONSEQUENCE: autovacuum works per partition, so a 400 GB
   table becomes 40 × 10 GB tables that vacuum independently and
   in parallel. This is a real and underappreciated benefit.
```

### Partition pruning — where the speed comes from, and when it fails

```
 ★ PRUNING = THE PLANNER (or executor) PROVING A PARTITION CANNOT
   CONTAIN MATCHING ROWS, AND SKIPPING IT ENTIRELY.

 ★ IT ONLY WORKS IF THE QUERY CONSTRAINS THE PARTITION KEY.

 ✓ PRUNES
   WHERE created_at >= '2026-08-01' AND created_at < '2026-09-01'
   ⇒ Seq Scan on events_2026_08 only. 119 partitions never opened.

 ✗ DOES NOT PRUNE — the four failure modes:

 ① ★ THE KEY IS ABSENT FROM THE PREDICATE
    WHERE tenant_id = 42
    ⇒ ★ scans ALL 120 partitions. SLOWER than an unpartitioned
      table, because it's 120 index scans instead of 1.

 ② ★ THE KEY IS WRAPPED IN A NON-IMMUTABLE FUNCTION
    WHERE date_trunc('month', created_at) = '2026-08-01'
    ⇒ ★ NO PRUNING. The planner cannot invert the function.
    ⇒ ★ FIX: always express it as a RANGE on the raw column.

 ③ ★ THE VALUE IS A PARAMETER AND THE PLAN IS GENERIC
    WHERE created_at >= $1
    ⇒ at plan time $1 is unknown ⇒ no PLANNER pruning.
    ⇒ ★ BUT: PostgreSQL 11+ does RUN-TIME PRUNING —
      look for "Subplans Removed: 118" in EXPLAIN ANALYZE.
    ⇒ ★ run-time pruning happens per execution and is nearly as
      good; but it does NOT help the planner's cost estimates.

 ④ ★ A JOIN ON A NON-KEY COLUMN
    JOIN tenants t ON t.id = e.tenant_id
    ⇒ the planner can't prune from a join condition unless
      partitionwise join is enabled AND both sides are
      partitioned identically.

 ⇒ ★ SEE IT:
   EXPLAIN (ANALYZE) …
   -> Append (actual rows=…)
        ★ Subplans Removed: 118      ← run-time pruning worked
        -> Seq Scan on events_2026_08
```

### The constraints partitioning imposes — the ones that bite

```
 ★ ① EVERY UNIQUE CONSTRAINT MUST INCLUDE THE PARTITION KEY.
    PRIMARY KEY (id)                    ⇒ ★ ERROR
    PRIMARY KEY (id, created_at)        ⇒ ✓
    ⇒ ★ WHY: a global unique index across partitions would require
      checking every partition on every insert. PostgreSQL has no
      global indexes.
    ⇒ ★ CONSEQUENCE: `id` alone is NO LONGER UNIQUE across the
      table. Two rows in different partitions can share an id
      unless your id generator guarantees otherwise.
      ⇒ use bigserial/Snowflake/UUIDv7 (Topic 22) — globally unique
        by construction — and accept that the DB won't enforce it.

 ★ ② FOREIGN KEYS *TO* A PARTITIONED TABLE
    supported since PG 12, but ★ the referenced columns must be a
    unique constraint, which must include the partition key.
    ⇒ so children must store the partition key too.
    ⇒ ★ IN PRACTICE: many teams drop the FK to a partitioned table
      and enforce it in the application. Be deliberate about it.

 ★ ③ INDEXES ARE PER-PARTITION
    CREATE INDEX ON events (tenant_id);
    ⇒ creates an index on the PARENT, which cascades to every
      partition and to every FUTURE partition.
    ⇒ ★ BUT: `CREATE INDEX` on the parent takes ACCESS EXCLUSIVE
      and ★ CANNOT be `CONCURRENTLY`.
    ⇒ ★ THE SAFE PATTERN:
      ① CREATE INDEX … ON ONLY events (tenant_id);   -- invalid, no lock
      ② CREATE INDEX CONCURRENTLY on each partition
      ③ ALTER INDEX events_tenant_idx ATTACH PARTITION …
      ⇒ the parent index becomes valid once all children attach.

 ★ ④ max_locks_per_transaction
    a query touching N partitions needs N lock entries.
    ⇒ ★ 5,000 partitions × 200 connections blows shared memory:
      ERROR: out of shared memory
      HINT: You might need to increase max_locks_per_transaction.
    ⇒ ★ AND THE PLANNER SLOWS DOWN: planning time grows roughly
      linearly with partition count.
      MEASURED: 12 partitions 0.4 ms · 120 → 2.1 ms ·
                ★ 1,200 → 41 ms · ★ 10,000 → 880 ms
    ⇒ ★ KEEP PARTITION COUNT UNDER ~1,000. Prefer fewer, larger.
```

### `DROP` vs `DETACH` vs `DELETE` — the retention decision

```
 ★ DROP TABLE events_2026_05;
   ⇒ 12 ms · ~2 KB WAL · disk returned immediately
   ⇒ ★ takes ACCESS EXCLUSIVE on the PARENT briefly
     ⇒ ★ USE lock_timeout (Topic 45) or it can queue behind a
       long query and block everything.

 ★ ALTER TABLE events DETACH PARTITION events_2026_05;
   ⇒ the partition becomes a standalone table; data preserved
   ⇒ ★ CONCURRENTLY variant (PG 14+) avoids ACCESS EXCLUSIVE:
     ALTER TABLE events DETACH PARTITION events_2026_05 CONCURRENTLY;
   ⇒ ★ THE RIGHT PATTERN FOR ARCHIVING: detach → copy to cold
     storage → drop.

 ✗ DELETE FROM events WHERE created_at < …;
   ⇒ ★ everything partitioning exists to avoid.

 ★ AND THE ATTACH SIDE HAS A TRAP:
   ALTER TABLE events ATTACH PARTITION events_2026_09
     FOR VALUES FROM … TO …;
   ⇒ ★ PostgreSQL SCANS THE WHOLE TABLE to verify every row
     satisfies the bound — holding ACCESS EXCLUSIVE.
   ⇒ ★ FIX: add a matching CHECK constraint FIRST, and the scan
     is skipped:
     ALTER TABLE events_2026_09 ADD CONSTRAINT c CHECK
       (created_at >= '2026-09-01' AND created_at < '2026-10-01');
     ALTER TABLE events ATTACH PARTITION events_2026_09 FOR VALUES …;
     ⇒ ★ instant instead of a full scan.
```

---

## How it works — step by step

### Choosing the partition key

```
 ★ THE KEY IS DECIDED BY THE OPERATION YOU MOST NEED TO BE CHEAP.
   Not by what you query most.

 ① ★ DO YOU HAVE A RETENTION POLICY?
    "delete data older than N days"
    ⇒ ★ RANGE ON THE TIME COLUMN. This is the dominant reason to
      partition and it decides the key immediately.

 ② ★ IS THERE A HARD TENANCY BOUNDARY?
    every query filters by tenant_id; tenants are deleted whole
    ⇒ LIST or HASH on tenant_id
    ⇒ ★ CAUTION: tenant sizes are almost always power-law
      distributed. LIST partitioning by tenant gives you one
      500 GB partition and 4,000 empty ones.

 ③ ★ IS WRITE CONTENTION THE PROBLEM?
    ⇒ HASH partitioning spreads inserts across partitions,
      reducing index-page contention on the rightmost leaf.
    ⇒ ★ but it makes range queries scan everything.

 ④ ★ IS THE TABLE JUST BIG?
    ⇒ ★ THAT IS NOT A REASON ON ITS OWN. A well-indexed 500 GB
      table performs fine. Partitioning it without a retention or
      manageability motive adds planning overhead and constraints
      for nothing.

 ⇒ ★ AND THE KEY MUST APPEAR IN YOUR HOT QUERIES' WHERE CLAUSES,
   or pruning never happens and you have made things worse.
```

### Sizing partitions

```
 ★ TARGET: 10–100 GB PER PARTITION, AND UNDER ~1,000 PARTITIONS.

 TOO MANY (10,000 monthly partitions over 800 years — or hourly
 partitions kept for 3 years):
   ⇒ ★ planning time 880 ms per query
   ⇒ ★ max_locks_per_transaction exhaustion
   ⇒ ★ autovacuum worker starvation (3 workers, 10,000 tables)
   ⇒ ★ pg_dump and backups slow to a crawl

 TOO FEW (4 partitions of 200 GB each):
   ⇒ pruning saves little
   ⇒ DROP granularity is coarse — you can only expire in
     200 GB chunks

 ⇒ ★ THE ARITHMETIC:
   retention 90 days, 500 GB total
   ⇒ daily:   90 partitions × 5.5 GB   ✓ good
   ⇒ weekly:  13 partitions × 38 GB    ✓ also fine
   ⇒ hourly:  ★ 2,160 partitions × 230 MB  ✗ too many
 ⇒ ★ CHOOSE THE COARSEST GRANULARITY THAT MEETS YOUR RETENTION
   PRECISION.
```

### Automating partition creation — the thing that must never fail

```sql
-- ★ ① A DEFAULT PARTITION IS YOUR SEATBELT.
--    It catches rows that fit nowhere, turning a write outage
--    into an alertable anomaly.
CREATE TABLE events_default PARTITION OF events DEFAULT;

-- ★ CAUTION: while a DEFAULT partition holds rows, ATTACHing a new
--   partition must SCAN the default to check for conflicting rows.
--   ⇒ keep the default EMPTY. Alert on any row in it.
SELECT count(*) FROM events_default;    -- ★ alert if > 0
```

```sql
-- ★ ② CREATE PARTITIONS AHEAD, IDEMPOTENTLY
CREATE OR REPLACE FUNCTION ensure_event_partitions(months_ahead int DEFAULT 3)
RETURNS void AS $$
DECLARE
  m date;
  part text;
BEGIN
  FOR i IN 0..months_ahead LOOP
    m := date_trunc('month', current_date)::date + (i || ' months')::interval;
    part := 'events_' || to_char(m, 'YYYY_MM');
    IF NOT EXISTS (SELECT 1 FROM pg_class WHERE relname = part) THEN
      EXECUTE format(
        'CREATE TABLE %I PARTITION OF events FOR VALUES FROM (%L) TO (%L)',
        part, m, (m + interval '1 month')::date);
      EXECUTE format('ALTER TABLE %I SET (autovacuum_vacuum_scale_factor=0.02)', part);
      RAISE NOTICE 'created partition %', part;
    END IF;
  END LOOP;
END $$ LANGUAGE plpgsql;

SELECT ensure_event_partitions(3);   -- ★ run daily from cron
```

```sql
-- ★ ③ THE ALERT THAT PREVENTS THE MIDNIGHT OUTAGE
SELECT count(*) AS future_partitions
  FROM pg_class c
  JOIN pg_inherits i ON i.inhrelid = c.oid
 WHERE i.inhparent = 'events'::regclass
   AND c.relname > 'events_' || to_char(current_date, 'YYYY_MM');
-- ★ alert if < 2. You want at least two months of runway.
```

```sql
-- ★ ④ RETENTION, WITH A LOCK TIMEOUT
DO $$
DECLARE p text;
BEGIN
  SET LOCAL lock_timeout = '5s';              -- ★ never queue (Topic 45)
  FOR p IN
    SELECT c.relname FROM pg_class c
      JOIN pg_inherits i ON i.inhrelid = c.oid
     WHERE i.inhparent = 'events'::regclass
       AND c.relname < 'events_' || to_char(current_date - interval '90 days',
                                            'YYYY_MM')
  LOOP
    EXECUTE format('DROP TABLE %I', p);
    RAISE NOTICE 'dropped %', p;
  END LOOP;
END $$;
```

---

## Concept breakdown

```
WHAT IT IS
└── one logical table, rows physically in child tables by a KEY
     ★ parent has relkind='p' and ★ NO STORAGE
     ★ each child is a full independent table: own heap, indexes,
       statistics, ★ own autovacuum schedule

★ THE REAL REASON: RETENTION
   DELETE 41M rows: ★ 41 min · 18 GB WAL · dead tuples · no space back
   DROP PARTITION:  ★ 12 ms  · 2 KB WAL  · space returned
   ⇒ ★ ~200,000×. This alone justifies it.

THREE STRATEGIES
├── ★ RANGE  ordered keys · ★ enables DROP retention · time-series
├── LIST     discrete values · tenancy, geography
└── HASH     spreads write contention · ★ no range pruning, no DROP

★ PRUNING — and its four failure modes
   ✓ WHERE created_at >= 'x' AND < 'y'
   ✗ ① the key is absent          ⇒ scans ALL partitions, ★ SLOWER
   ✗ ② wrapped in a function       date_trunc(...) = ... ⇒ no pruning
   ✗ ③ a parameter + generic plan  ⇒ ★ run-time pruning saves it
                                     ("Subplans Removed: 118")
   ✗ ④ a join on a non-key column

★ THE CONSTRAINTS THAT BITE
├── ★ every UNIQUE must INCLUDE the partition key
│    ⇒ `id` alone is no longer unique — rely on your id generator
├── FKs to a partitioned table need the key in the child too
├── ★ CREATE INDEX on the parent = ACCESS EXCLUSIVE, no CONCURRENTLY
│    ⇒ ON ONLY + per-partition CONCURRENTLY + ATTACH
└── ★ max_locks_per_transaction + planning time grow with count
     12→0.4ms · 120→2.1ms · 1,200→41ms · ★ 10,000→880ms

★ SIZING: 10–100 GB per partition, ★ under ~1,000 partitions

★ ATTACH SCANS THE TABLE unless a matching CHECK exists first
★ DETACH CONCURRENTLY (PG14+) avoids ACCESS EXCLUSIVE
★ DROP takes ACCESS EXCLUSIVE on the parent ⇒ use lock_timeout

★ THE #1 INCIDENT: no partition for tomorrow's rows
   ERROR: no partition of relation "events" found for row
   ⇒ ★ TOTAL WRITE OUTAGE AT MIDNIGHT
   ⇒ PREVENT: a DEFAULT partition (kept empty, alerted) +
     idempotent creation 3 months ahead + a runway alert
```

---

## Diagrams

**Diagram 1 — big picture: what pruning does, and when it doesn't**

```
                   events (relkind='p', ★ NO STORAGE)
   ┌──────────┬──────────┬──────────┬─────┬──────────┬──────────┐
   │ 2026_01  │ 2026_02  │  …       │ …   │ 2026_08  │ default  │
   │  5.5 GB  │  5.8 GB  │          │     │  6.1 GB  │  ★ empty │
   └──────────┴──────────┴──────────┴─────┴──────────┴──────────┘

 ✓ PRUNES — the key is constrained by a range
   WHERE created_at >= '2026-08-01' AND created_at < '2026-09-01'
   ┌──────────┬──────────┬─────┬──────────┬──────────┐
   │ ✗ skip   │ ✗ skip   │ ✗   │ ★ SCAN   │ ✗ skip   │
   └──────────┴──────────┴─────┴──────────┴──────────┘
   ⇒ 6.1 GB read instead of 500 GB. ★ 82× less I/O.
   ⇒ EXPLAIN: "Subplans Removed: 118"

 ✗ DOES NOT PRUNE — the key is absent
   WHERE tenant_id = 42
   ┌──────────┬──────────┬─────┬──────────┬──────────┐
   │ ★ SCAN   │ ★ SCAN   │ ★   │ ★ SCAN   │ ★ SCAN   │
   └──────────┴──────────┴─────┴──────────┴──────────┘
   ⇒ ★ 120 index scans instead of 1.
   ⇒ ★ SLOWER THAN NOT PARTITIONING AT ALL.
   ⇒ ★ THIS IS WHY THE KEY MUST MATCH YOUR HOT QUERIES.

 ✗ DOES NOT PRUNE — the key is wrapped in a function
   WHERE date_trunc('month', created_at) = '2026-08-01'
   ⇒ ★ the planner cannot invert date_trunc. All 120 scanned.
   ⇒ ★ FIX: rewrite as a raw range on created_at.
```

**Diagram 2 — data flow: `DELETE` retention vs `DROP` retention**

```
 ✗ DELETE FROM events WHERE created_at < now() - interval '90 days';
 ┌────────────────────────────────────────────────────────────────┐
 │ ① find 41,000,000 rows       ← index scan or, worse, seq scan  │
 │ ② set t_xmax on each          ⇒ ★ 41M tuple rewrites           │
 │ ③ update ★ every index         ⇒ 6 indexes × 41M entries        │
 │ ④ write ★ 18 GB of WAL                                          │
 │ ⑤ ⇒ ★ 24 min of replica lag (replay is serial — Topic 58)      │
 │ ⑥ ⇒ ★ 41M dead tuples ⇒ hours of autovacuum                    │
 │ ⑦ ⇒ ★ ZERO DISK SPACE RETURNED (Topic 47)                      │
 │ ⑧ ⇒ ★ locks held throughout; often doesn't finish              │
 │                                                                 │
 │ MEASURED: ★ 41 minutes                                          │
 └────────────────────────────────────────────────────────────────┘

 ✓ DROP TABLE events_2026_05;
 ┌────────────────────────────────────────────────────────────────┐
 │ ① brief ACCESS EXCLUSIVE on the parent (★ with lock_timeout)   │
 │ ② unlink the heap file and its indexes                         │
 │ ③ ~2 KB of WAL                                                  │
 │                                                                 │
 │ MEASURED: ★ 12 milliseconds                                     │
 │ ★ zero dead tuples · zero index churn · ★ disk returned         │
 └────────────────────────────────────────────────────────────────┘
                          ★ ~200,000×
```

**Diagram 3 — before/after: the partition-count mistake**

```
 ✗ HOURLY PARTITIONS, 90-DAY RETENTION
 ┌───────────────────────────────────────────────────────────────┐
 │ 90 days × 24 hours = ★ 2,160 partitions                       │
 │ each ~230 MB                                                   │
 │                                                                │
 │ ★ planning time per query        41 ms → ★ 184 ms              │
 │   (a 0.4 ms query now spends 184 ms being PLANNED)            │
 │ ★ max_locks_per_transaction      exhausted at 200 connections  │
 │   ERROR: out of shared memory                                  │
 │ ★ autovacuum                     3 workers, 2,160 tables       │
 │                                  ⇒ each visited every ~9 hours │
 │ ★ pg_dump                        4 min → ★ 51 min              │
 │ ★ pg_class rows                  2,160 × (1 + 6 indexes)       │
 │                                  = ★ 15,120 relations          │
 └───────────────────────────────────────────────────────────────┘

 ✓ DAILY PARTITIONS, SAME RETENTION
 ┌───────────────────────────────────────────────────────────────┐
 │ 90 partitions, each ~5.5 GB                                    │
 │                                                                │
 │ ★ planning time                  ★ 1.8 ms                      │
 │ ★ locks                          fine at any connection count  │
 │ ★ autovacuum                     each table visited often      │
 │ ★ pg_dump                        ★ 4.2 min                     │
 │ ★ retention granularity          1 day — ★ still precise       │
 │                                    enough for a 90-day policy  │
 └───────────────────────────────────────────────────────────────┘
   ★ THE RULE: choose the COARSEST granularity that meets your
     retention precision. Finer buys nothing and costs a lot.
```

---

## Example 1 — basic

```sql
CREATE TABLE events (
  id         bigserial,
  tenant_id  bigint      NOT NULL,
  event_type text        NOT NULL,
  payload    jsonb       NOT NULL,
  created_at timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (id, created_at)          -- ★ must include the key
) PARTITION BY RANGE (created_at);
```

**Prove the key must be in every unique constraint.**
```sql
CREATE TABLE bad (id bigserial PRIMARY KEY, created_at timestamptz NOT NULL)
  PARTITION BY RANGE (created_at);
```
```
ERROR:  unique constraint on partitioned table must include all
        partitioning columns
DETAIL:  PRIMARY KEY constraint on table "bad" lacks column
         "created_at" which is part of the partition key.
```

**Create partitions and the default.**
```sql
CREATE TABLE events_2026_06 PARTITION OF events
  FOR VALUES FROM ('2026-06-01') TO ('2026-07-01');
CREATE TABLE events_2026_07 PARTITION OF events
  FOR VALUES FROM ('2026-07-01') TO ('2026-08-01');
CREATE TABLE events_2026_08 PARTITION OF events
  FOR VALUES FROM ('2026-08-01') TO ('2026-09-01');
CREATE TABLE events_default PARTITION OF events DEFAULT;   -- ★ the seatbelt

INSERT INTO events (tenant_id, event_type, payload, created_at)
SELECT (random()*1000)::bigint, 'click', '{}'::jsonb,
       '2026-06-01'::timestamptz + (random()*90)::int * interval '1 day'
  FROM generate_series(1, 3000000);
VACUUM ANALYZE events;
```

**Prove the parent has no storage.**
```sql
SELECT c.relname, c.relkind, c.relfilenode,
       pg_size_pretty(pg_relation_size(c.oid)) AS size
  FROM pg_class c WHERE c.relname LIKE 'events%' ORDER BY 1;
```
```
    relname     | relkind | relfilenode |  size
----------------+---------+-------------+---------
 events         | ★ p     |         ★ 0 | 0 bytes
 events_2026_06 | r       |       88412 | 224 MB
 events_2026_07 | r       |       88418 | 231 MB
 events_2026_08 | r       |       88424 | 228 MB
 events_default | r       |       88430 | ★ 0 bytes
```

**Prove pruning works.**
```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT count(*) FROM events
 WHERE created_at >= '2026-08-01' AND created_at < '2026-09-01';
```
```
 Aggregate  (actual time=88.2..88.2 rows=1)
   Buffers: shared hit=29,204
   ->  Seq Scan on ★ events_2026_08 events  (actual rows=1,004,882)
 Execution Time: ★ 88.4 ms
   ★ ONE partition scanned. The other three never opened.
```

**And the four ways it fails.**
```sql
-- ① the key is absent
EXPLAIN (ANALYZE) SELECT count(*) FROM events WHERE tenant_id = 42;
```
```
 Aggregate
   ->  Append  (actual rows=3,004)
         ->  Seq Scan on events_2026_06 events_1
         ->  Seq Scan on events_2026_07 events_2
         ->  ★ Seq Scan on events_2026_08 events_3
         ->  Seq Scan on events_default  events_4
 Execution Time: ★ 412.8 ms      — all four scanned
```
```sql
-- ② wrapped in a function
EXPLAIN (ANALYZE) SELECT count(*) FROM events
 WHERE date_trunc('month', created_at) = '2026-08-01';
```
```
   ->  Append
         ->  Seq Scan on events_2026_06 …
         ->  Seq Scan on events_2026_07 …
         ->  Seq Scan on events_2026_08 …
 Execution Time: ★ 388.4 ms      — ★ NO PRUNING
```
```sql
-- ★ the fix — a raw range
EXPLAIN (ANALYZE) SELECT count(*) FROM events
 WHERE created_at >= '2026-08-01' AND created_at < '2026-09-01';
--  ⇒ one partition, 88 ms
```
```sql
-- ③ a parameter — run-time pruning saves it
PREPARE q(timestamptz, timestamptz) AS
  SELECT count(*) FROM events WHERE created_at >= $1 AND created_at < $2;
EXPLAIN (ANALYZE) EXECUTE q('2026-08-01','2026-09-01');
```
```
 Aggregate
   ->  Append  (actual rows=1,004,882)
         ★ Subplans Removed: 3
         ->  Seq Scan on events_2026_08 events_1
 Execution Time: 89.1 ms
   ★ "Subplans Removed: 3" — run-time pruning. Nearly as good.
```

**Prove `DROP` beats `DELETE`.**
```sql
\timing on
DELETE FROM events WHERE created_at < '2026-07-01';
```
```
 DELETE 1004118
 Time: ★ 8,842.4 ms
```
```sql
SELECT pg_size_pretty(pg_relation_size('events_2026_06')) AS size,
       n_dead_tup FROM pg_stat_user_tables WHERE relname='events_2026_06';
```
```
  size  | n_dead_tup
--------+------------
 224 MB | ★ 1004118      — ★ full size, one million dead tuples
```
```sql
-- vs the partition-native way
DROP TABLE events_2026_07;
```
```
 DROP TABLE
 Time: ★ 11.2 ms        — ★ 790× faster, and the disk is returned
```

**Measure the WAL difference.**
```sql
SELECT pg_current_wal_lsn() AS a \gset
DELETE FROM events WHERE created_at >= '2026-08-01' AND created_at < '2026-08-10';
SELECT pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), :'a')) AS delete_wal;
```
```
 delete_wal
------------
 ★ 84 MB
```
```sql
SELECT pg_current_wal_lsn() AS b \gset
DROP TABLE events_2026_06;
SELECT pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), :'b')) AS drop_wal;
```
```
 drop_wal
----------
 ★ 1,842 bytes      — 47,000×
```

**Reproduce the midnight outage.**
```sql
INSERT INTO events (tenant_id, event_type, payload, created_at)
VALUES (1, 'click', '{}', '2027-01-15');
```
```
 INSERT 0 1        ★ it went to events_default — the seatbelt worked
```
```sql
DROP TABLE events_default;
INSERT INTO events (tenant_id, event_type, payload, created_at)
VALUES (1, 'click', '{}', '2027-01-15');
```
```
ERROR:  no partition of relation "events" found for row
DETAIL:  Partition key of the failing row contains (created_at) =
         (2027-01-15 00:00:00+05:30).
   ★ WITHOUT A DEFAULT PARTITION, THIS IS A TOTAL WRITE OUTAGE.
```

**Prove `ATTACH` scans unless a `CHECK` exists.**
```sql
CREATE TABLE events_2026_09 (LIKE events INCLUDING ALL);
INSERT INTO events_2026_09 SELECT * FROM events LIMIT 1000000;
-- (with created_at values inside the September range)

\timing on
ALTER TABLE events ATTACH PARTITION events_2026_09
  FOR VALUES FROM ('2026-09-01') TO ('2026-10-01');
-- Time: ★ 1,884.2 ms      — a full scan to verify
```
```sql
-- ★ with a matching CHECK first
ALTER TABLE events DETACH PARTITION events_2026_09;
ALTER TABLE events_2026_09 ADD CONSTRAINT c_range
  CHECK (created_at >= '2026-09-01' AND created_at < '2026-10-01');
ALTER TABLE events ATTACH PARTITION events_2026_09
  FOR VALUES FROM ('2026-09-01') TO ('2026-10-01');
-- Time: ★ 4.1 ms      — 460×, the scan was skipped
```

**Measure planning time vs partition count.**
```sql
-- create N partitions and time a simple query
EXPLAIN (ANALYZE) SELECT count(*) FROM events WHERE created_at >= now() - interval '1 day';
```
```
 partitions |  planning time
------------+----------------
         12 |     ★ 0.412 ms
        120 |     ★ 2.104 ms
       1200 |    ★ 41.882 ms
      10000 |   ★ 884.221 ms       — ★ the query itself takes 0.4 ms
```

**Create an index safely.**
```sql
-- ✗ this takes ACCESS EXCLUSIVE on every partition
CREATE INDEX idx_events_tenant ON events (tenant_id);

-- ✓ the safe pattern
CREATE INDEX idx_events_tenant ON ONLY events (tenant_id);   -- invalid, no data
CREATE INDEX CONCURRENTLY idx_events_tenant_2026_08
  ON events_2026_08 (tenant_id);
ALTER INDEX idx_events_tenant ATTACH PARTITION idx_events_tenant_2026_08;
-- … repeat per partition; the parent index becomes valid when all attach
SELECT indexrelid::regclass, indisvalid FROM pg_index
 WHERE indrelid = 'events'::regclass;
```

---

## Example 2 — production scenario

**The situation.** A payments platform's `payment_attempts` table — the same one from Topic 47's wraparound incident, now being redesigned.

```
 CURRENT STATE
   rows                    ★ 4.1 billion
   size                    ★ 412 GB (11 GB of live data after the repack)
   retention policy        90 days, ★ enforced by a nightly DELETE
   the nightly DELETE      ★ started at 02:00, ran until 06:40, often
                             did not finish
   replica lag at 02:00    ★ 24 minutes
   autovacuum              ★ permanently behind
   pg_dump                 ★ 6.2 hours
   the wraparound incident ★ 1h 27m outage (Topic 47)
```

**Step 1 — establish why partitioning, not something else.**

```sql
-- what does the workload actually look like?
SELECT calls, mean_exec_time::numeric(10,2) AS ms, substring(query,1,70) AS q
  FROM pg_stat_statements
 WHERE query ILIKE '%payment_attempts%' ORDER BY calls DESC LIMIT 5;
```
```
  calls   |    ms   |                              q
----------+---------+--------------------------------------------------------
 88402118 |    0.08 | SELECT … FROM payment_attempts WHERE id = $1
 41204882 |    0.12 | INSERT INTO payment_attempts (…) VALUES (…)
  8842119 |   14.20 | SELECT … WHERE created_at >= $1 AND created_at < $2
    41204 | ★ 8,842 | SELECT … WHERE merchant_id = $1 AND created_at >= $2
      365 | ★ 14M   | DELETE FROM payment_attempts WHERE created_at < $1
```

```
 ★ THE DECISION IS MADE BY ROW 5, NOT ROWS 1–4.
   The point-lookups are fine. The retention DELETE is a
   4-hour-40-minute nightly catastrophe that causes 24 minutes of
   replica lag and keeps autovacuum permanently behind.
 ⇒ ★ RANGE PARTITION ON created_at. The key is decided by the
   operation that must become cheap, not by the query that runs
   most.
```

**Step 2 — size it.**

```sql
SELECT date_trunc('day', created_at)::date AS day,
       count(*), pg_size_pretty(sum(pg_column_size(t.*))) AS approx
  FROM payment_attempts t
 WHERE created_at > now() - interval '7 days'
 GROUP BY 1 ORDER BY 1 DESC;
```
```
    day     |  count   | approx
------------+----------+---------
 2026-08-18 | 41204882 | ★ 4.8 GB
 2026-08-17 | 38820114 | 4.5 GB
```
```
 ★ SIZING:
   90 days × 4.8 GB = 432 GB total
   daily:   ★ 90 partitions × 4.8 GB   ✓ in the 10–100 GB… no, under.
   weekly:  ★ 13 partitions × 33 GB    ✓ ideal size, but retention
                                          granularity is 7 days
   ⇒ ★ THE POLICY SAYS "90 DAYS". Weekly means data lives 90–96
     days. ★ FOR A PAYMENTS SYSTEM WITH A REGULATORY RETENTION
     LIMIT, THAT MATTERS.
   ⇒ ★ CHOOSE DAILY. 90 partitions is comfortably under 1,000,
     4.8 GB each is manageable, and retention is exact.
```

**Step 3 — the migration, without downtime.**

```sql
-- ★ ① the new partitioned table, alongside the old one
CREATE TABLE payment_attempts_p (
  id              bigint      NOT NULL,
  merchant_id     bigint      NOT NULL,
  amount_minor    bigint      NOT NULL,
  status          text        NOT NULL,
  idempotency_key text        NOT NULL,
  created_at      timestamptz NOT NULL,
  PRIMARY KEY (id, created_at)         -- ★ key must include created_at
) PARTITION BY RANGE (created_at);

-- ★ id alone is no longer enforced unique. It is a bigserial from a
--   single sequence, so it IS globally unique — but the database
--   will not check that. Document it.
COMMENT ON COLUMN payment_attempts_p.id IS
  'Globally unique by construction (single bigserial sequence).
   ★ NOT enforced across partitions — PostgreSQL has no global
   unique indexes on partitioned tables.';

-- ★ and the idempotency key, which MUST be unique (Topic 52)
--   ⇒ it must include created_at, which weakens the guarantee.
--   ⇒ THE FIX: keep a separate, unpartitioned key table.
CREATE TABLE payment_idempotency (
  idempotency_key text PRIMARY KEY,
  attempt_id      bigint      NOT NULL,
  created_at      timestamptz NOT NULL DEFAULT now()
);
-- ★ small, unpartitioned, genuinely globally unique.
--   This is a common and important pattern: partition the bulk,
--   keep the uniqueness constraint in a small side table.
```

```sql
-- ② the partition manager, run daily
CREATE OR REPLACE FUNCTION ensure_payment_partitions(days_ahead int DEFAULT 14)
RETURNS int AS $$
DECLARE d date; part text; created int := 0;
BEGIN
  FOR i IN 0..days_ahead LOOP
    d := current_date + i;
    part := 'payment_attempts_p_' || to_char(d, 'YYYY_MM_DD');
    IF NOT EXISTS (SELECT 1 FROM pg_class WHERE relname = part) THEN
      EXECUTE format(
        'CREATE TABLE %I PARTITION OF payment_attempts_p
           FOR VALUES FROM (%L) TO (%L)', part, d, d + 1);
      EXECUTE format('ALTER TABLE %I SET (
         autovacuum_vacuum_scale_factor = 0.02,
         autovacuum_vacuum_cost_delay   = 0)', part);
      EXECUTE format('CREATE INDEX CONCURRENTLY %I ON %I (merchant_id, created_at)',
                     part || '_merchant_idx', part);
      created := created + 1;
    END IF;
  END LOOP;
  RETURN created;
END $$ LANGUAGE plpgsql;

SELECT ensure_payment_partitions(14);   -- ★ 14 days of runway
```

```sql
-- ★ ③ the DEFAULT partition — the seatbelt
CREATE TABLE payment_attempts_p_default PARTITION OF payment_attempts_p DEFAULT;
-- ★ and the alert: any row here means the manager failed
SELECT count(*) FROM payment_attempts_p_default;   -- ★ alert if > 0
```

```js
// ④ dual-write during the cutover (the expand phase — Topic 28)
async function recordAttempt(tx, attempt) {
  await tx.query(`INSERT INTO payment_attempts (…) VALUES (…)`, [...]);
  await tx.query(`INSERT INTO payment_attempts_p (…) VALUES (…)`, [...]);
  // ★ same transaction, so they cannot diverge
}
```

```sql
-- ⑤ backfill the last 90 days, one day at a time, ★ off-peak
DO $$
DECLARE d date := current_date - 90;
BEGIN
  WHILE d <= current_date LOOP
    EXECUTE format($f$
      INSERT INTO payment_attempts_p
      SELECT * FROM payment_attempts
       WHERE created_at >= %L AND created_at < %L
      ON CONFLICT DO NOTHING $f$, d, d + 1);
    RAISE NOTICE 'backfilled %', d;
    d := d + 1;
    COMMIT;                       -- ★ one transaction per day
    PERFORM pg_sleep(2);          -- ★ let replicas catch up
  END LOOP;
END $$;
```

```sql
-- ⑥ verify, then cut over
SELECT
  (SELECT count(*) FROM payment_attempts WHERE created_at >= current_date - 90) AS old,
  (SELECT count(*) FROM payment_attempts_p) AS new;
-- ⇒ must match

BEGIN;
SET LOCAL lock_timeout = '5s';                 -- ★ never queue
ALTER TABLE payment_attempts   RENAME TO payment_attempts_old;
ALTER TABLE payment_attempts_p RENAME TO payment_attempts;
COMMIT;
-- later, after a soak period:
DROP TABLE payment_attempts_old;
```

**Step 4 — retention becomes trivial.**

```sql
CREATE OR REPLACE FUNCTION drop_old_payment_partitions(keep_days int DEFAULT 90)
RETURNS int AS $$
DECLARE p record; dropped int := 0;
BEGIN
  SET LOCAL lock_timeout = '5s';               -- ★ Topic 45
  FOR p IN
    SELECT c.relname
      FROM pg_class c JOIN pg_inherits i ON i.inhrelid = c.oid
     WHERE i.inhparent = 'payment_attempts'::regclass
       AND c.relname ~ '^payment_attempts_p_\d{4}_\d{2}_\d{2}$'
       AND to_date(right(c.relname, 10), 'YYYY_MM_DD') < current_date - keep_days
  LOOP
    -- ★ archive first: detach, copy to cold storage, then drop
    EXECUTE format('ALTER TABLE payment_attempts DETACH PARTITION %I CONCURRENTLY',
                   p.relname);
    PERFORM archive_to_s3(p.relname);
    EXECUTE format('DROP TABLE %I', p.relname);
    dropped := dropped + 1;
  END LOOP;
  RETURN dropped;
END $$ LANGUAGE plpgsql;
```
```
 ★ MEASURED: 14 ms per partition. The 4h40m nightly DELETE is gone.
```

**Step 5 — the alerts.**

```sql
-- ★ ① RUNWAY — the alert that prevents the midnight outage
SELECT count(*) AS days_of_runway
  FROM pg_class c JOIN pg_inherits i ON i.inhrelid = c.oid
 WHERE i.inhparent = 'payment_attempts'::regclass
   AND c.relname ~ '^payment_attempts_p_\d{4}_\d{2}_\d{2}$'
   AND to_date(right(c.relname, 10), 'YYYY_MM_DD') > current_date;
-- ★ alert if < 5

-- ★ ② anything in the default partition
SELECT count(*) FROM payment_attempts_p_default;   -- ★ alert if > 0

-- ★ ③ partition count (planning-time creep)
SELECT count(*) FROM pg_inherits WHERE inhparent = 'payment_attempts'::regclass;
-- ★ alert if > 200

-- ★ ④ per-partition bloat, since autovacuum now works per child
SELECT c.relname, s.n_dead_tup,
       pg_size_pretty(pg_relation_size(c.oid)) AS size
  FROM pg_class c JOIN pg_inherits i ON i.inhrelid = c.oid
  LEFT JOIN pg_stat_user_tables s ON s.relid = c.oid
 WHERE i.inhparent = 'payment_attempts'::regclass
 ORDER BY s.n_dead_tup DESC NULLS LAST LIMIT 5;
```

**Step 6 — the queries that needed rewriting.**

```sql
-- ✗ this no longer prunes
SELECT * FROM payment_attempts WHERE merchant_id = 8842 ORDER BY created_at DESC LIMIT 50;
-- ⇒ ★ scans all 90 partitions

-- ✓ every hot query must constrain created_at
SELECT * FROM payment_attempts
 WHERE merchant_id = 8842
   AND created_at >= now() - interval '30 days'      -- ★ prunes to 30
 ORDER BY created_at DESC LIMIT 50;
```
```
 Limit  (actual time=0.884..1.204 rows=50)
   ->  Merge Append  (actual rows=50)
         ★ Subplans Removed: 60
         ->  Index Scan Backward using payment_attempts_p_2026_08_18_merchant_idx
         ->  … 29 more
 Execution Time: ★ 1.284 ms       (was 8,842 ms)
```
```
 ★ AND THE PRODUCT DECISION THAT MADE IT POSSIBLE:
   the merchant dashboard's "all time" filter was changed to
   default to 30 days, with older ranges routed to the analytics
   replica.
 ⇒ ★ PARTITIONING FORCED A GOOD PRODUCT DECISION. An unbounded
   "all time" query over 4.1 billion rows was never reasonable.
```

**Step 7 — results.**

| | Before | After |
|---|---|---|
| Retention operation | ★ 4h 40m `DELETE`, often unfinished | **14 ms** per partition |
| Replica lag at 02:00 | 24 min | **< 1 s** |
| WAL from retention | 18 GB/night | **~2 KB/night** |
| Table size | 412 GB | **432 GB across 90 partitions**, stable |
| Disk reclaimed | never | **immediately, per partition** |
| `pg_dump` | 6.2 h | **48 min** (parallel per partition) |
| Autovacuum | permanently behind | ★ per-partition, current |
| Merchant dashboard p99 | 8,842 ms | **1.3 ms** |
| Wraparound risk | ★ the Topic 47 outage | ★ each partition freezes independently |

```
 ★ FIVE LESSONS:
 ① ★ THE PARTITION KEY WAS DECIDED BY THE RETENTION OPERATION,
   not by the most frequent query. 365 nightly DELETEs justified
   the design; 88 million point-lookups didn't care either way.
 ② ★ THE UNIQUENESS CONSTRAINT HAD TO MOVE. `idempotency_key`
   cannot be globally unique on a partitioned table, so it went
   into a small unpartitioned side table. ★ This pattern —
   partition the bulk, keep uniqueness in a side table — comes up
   constantly.
 ③ ★ DAILY, NOT WEEKLY, BECAUSE THE RETENTION POLICY WAS EXACT.
   Sizing is a policy question as much as a performance one.
 ④ ★ EVERY HOT QUERY HAD TO BE AUDITED for whether it constrains
   the partition key. One didn't, and fixing it required a product
   decision — which turned out to be the right one anyway.
 ⑤ ★ WRAPAROUND STOPPED BEING A RISK, because each partition
   freezes independently and old ones are dropped before they age.
   ★ Partitioning solved the Topic 47 incident structurally.
```

---

## Common mistakes

**1. No process to create future partitions.**
- *Symptom:* `ERROR: no partition of relation "events" found for row` — a total write outage at midnight.
- *Fix:* an idempotent creation function run daily with 14+ days of runway, a `DEFAULT` partition as a seatbelt, and a runway alert.

**2. A `DEFAULT` partition that accumulates rows.**
- *Symptom:* attaching a new partition takes minutes because the default must be scanned for conflicting rows.
- *Fix:* keep it empty; alert on any row in it. It's an alarm, not a destination.

**3. Partitioning on a key your hot queries don't filter by.**
- *Symptom:* every query scans every partition — **slower** than before.
- *Fix:* the key must appear in the `WHERE` clause of your hot queries. Audit them before choosing.

**4. Wrapping the partition key in a function.**
- *Symptom:* `date_trunc('month', created_at) = …` scans everything.
- *Engine-level why:* the planner cannot invert an arbitrary function to derive bounds.
- *Fix:* always express it as a raw range on the column.

**5. Too many partitions.**
- *Symptom:* planning time in the hundreds of milliseconds, `out of shared memory`, autovacuum starvation, slow dumps.
- *Fix:* the coarsest granularity that meets your retention precision. Target under ~1,000.

**6. `CREATE INDEX` on the parent in production.**
- *Symptom:* an `ACCESS EXCLUSIVE` lock across every partition; `CONCURRENTLY` isn't allowed there.
- *Fix:* `ON ONLY` the parent, `CONCURRENTLY` per partition, then `ATTACH PARTITION`.

**7. `ATTACH` without a matching `CHECK`.**
- *Symptom:* attaching a large table scans it entirely under `ACCESS EXCLUSIVE`.
- *Fix:* add the equivalent `CHECK` constraint first; the scan is skipped.

**8. `DROP PARTITION` without `lock_timeout`.**
- *Symptom:* the `ACCESS EXCLUSIVE` request queues behind a long query and blocks everything on the parent (Topic 45).
- *Fix:* `SET LOCAL lock_timeout` and retry.

**9. Assuming the primary key is still globally unique.**
- *Symptom:* duplicate ids across partitions.
- *Fix:* every unique constraint must include the partition key. Use a globally-unique id generator, and put genuine uniqueness requirements in an unpartitioned side table.

**10. Partitioning "because the table is big".**
- *Symptom:* constraints and planning overhead with no benefit.
- *Fix:* partition for retention or manageability. Size alone is not a reason.

**11. Forgetting per-partition autovacuum settings.**
- *Symptom:* new partitions inherit the defaults and fall behind.
- *Fix:* set them in the creation function, so every future partition gets them.

**12. Not auditing existing queries before migrating.**
- *Symptom:* a dashboard that was 8 s becomes 40 s because it doesn't constrain the key.
- *Fix:* audit `pg_stat_statements` for every query touching the table, before the cutover.

---

## Hands-on proof

**PROVE IT #1–#10 — Example 1** (the unique-constraint error, the parent's `relfilenode 0`, pruning to one partition, all four pruning failures, run-time pruning's `Subplans Removed`, `DROP` at 790× `DELETE`, the WAL difference at 47,000×, the midnight outage with and without a `DEFAULT`, `ATTACH` with and without a `CHECK` at 460×, and planning time vs partition count).

**PROVE IT #11 — per-partition autovacuum runs in parallel.**
```sql
UPDATE events SET payload = payload WHERE created_at >= '2026-08-01';
SELECT relid::regclass, phase, heap_blks_scanned, heap_blks_total
  FROM pg_stat_progress_vacuum;
```
```
        relid        |     phase     | heap_blks_scanned | heap_blks_total
---------------------+---------------+-------------------+-----------------
 events_2026_08      | scanning heap |             12204 |           29204
 events_2026_07      | scanning heap |              8842 |           28820
   ★ two partitions vacuumed simultaneously by different workers.
     An unpartitioned table gets ONE worker.
```

**PROVE IT #12 — partitionwise join and aggregate.**
```sql
SET enable_partitionwise_join = on;
SET enable_partitionwise_aggregate = on;
EXPLAIN (ANALYZE)
SELECT e.tenant_id, count(*) FROM events e
 WHERE created_at >= '2026-08-01' GROUP BY e.tenant_id;
```
```
 ★ Append
   ->  HashAggregate  (on events_2026_08)
 ⇒ ★ the aggregate is pushed INTO each partition rather than
   computed over the union. Off by default — measure before enabling,
   as it increases planning time.
```

**PROVE IT #13 — `max_locks_per_transaction` exhaustion.**
```sql
-- with 5,000 partitions and max_locks_per_transaction = 64
SELECT count(*) FROM events;      -- touches all partitions
```
```
ERROR:  out of shared memory
HINT:  You might need to increase max_locks_per_transaction.
   ★ each partition needs a lock entry. 5,000 × 200 connections
     exceeds the shared lock table.
```

**PROVE IT #14 — `DETACH CONCURRENTLY` doesn't block reads.**
```sql
-- session 1
BEGIN; SELECT count(*) FROM events; -- holds ACCESS SHARE, stays open
-- session 2
ALTER TABLE events DETACH PARTITION events_2026_06;              -- ⏸ blocks
ALTER TABLE events DETACH PARTITION events_2026_06 CONCURRENTLY; -- ★ proceeds
```

---

## The design decision framework

```
★★★ PARTITION FOR RETENTION AND MANAGEABILITY.
    PERFORMANCE IS A SIDE EFFECT. ★★★

 ① IS THERE A REASON?
    ✓ ★ a RETENTION POLICY ("delete older than N days")
        ⇒ this alone justifies it: 200,000× cheaper
    ✓ ★ MANAGEABILITY — per-partition vacuum, freeze, backup,
        independent index builds
    ✓ ★ a hard tenancy boundary where tenants are deleted whole
    ✗ ★ "the table is big" — NOT a reason. A well-indexed 500 GB
        table is fine.
    ✗ ★ "we might need to shard later" — different problem (60)

 ② CHOOSE THE KEY FROM THE OPERATION THAT MUST BE CHEAP
    retention          ⇒ ★ RANGE on the time column
    tenancy            ⇒ LIST/HASH on tenant_id
                         ★ beware power-law tenant sizes
    write contention   ⇒ HASH (★ but no range pruning, no DROP)
    ⇒ ★ AND THE KEY MUST APPEAR IN YOUR HOT QUERIES' WHERE
      CLAUSES. Audit pg_stat_statements BEFORE deciding.

 ③ SIZE IT
    ★ target 10–100 GB per partition, ★ under ~1,000 partitions
    ⇒ choose the COARSEST granularity meeting your retention
      PRECISION (a regulatory "90 days" may force daily over weekly)
    ⇒ planning time: 120→2ms · 1,200→41ms · ★ 10,000→880ms

 ④ HANDLE THE CONSTRAINTS DELIBERATELY
    ★ every UNIQUE must include the partition key
      ⇒ genuine global uniqueness (idempotency keys!) goes in a
        SMALL UNPARTITIONED SIDE TABLE
    ★ FKs to the partitioned table need the key in the child
    ★ indexes: ON ONLY + per-partition CONCURRENTLY + ATTACH
    ★ per-partition autovacuum settings, in the creation function

 ⑤ ★ AUTOMATE CREATION — THIS IS THE #1 INCIDENT
    ✓ idempotent function, run daily, ★ 14+ days of runway
    ✓ ★ a DEFAULT partition as a seatbelt — kept EMPTY
    ✓ ★ ALERT: runway < 5 days
    ✓ ★ ALERT: any row in the default partition

 ⑥ RETENTION: DETACH → ARCHIVE → DROP
    ✓ ★ SET lock_timeout before any DROP/DETACH (Topic 45)
    ✓ ★ DETACH CONCURRENTLY (PG14+) to avoid ACCESS EXCLUSIVE
    ✓ ★ ATTACH: add a matching CHECK first, or it scans the table

 ⑦ MIGRATING AN EXISTING TABLE
    ① create the partitioned table alongside
    ② ★ dual-write in the same transaction (Topic 28's expand)
    ③ backfill ★ one partition per transaction, with pauses
    ④ verify counts
    ⑤ rename under lock_timeout
    ⑥ soak, then drop the old table

 ⑧ ★ AUDIT EVERY HOT QUERY FOR PRUNING
    EXPLAIN and look for "Subplans Removed: N".
    ⇒ if a query doesn't constrain the key, it now scans EVERY
      partition and is SLOWER than before.
    ⇒ ★ this sometimes forces a product decision (bounding an
      "all time" filter) — which is usually correct anyway.

 ⑨ THE FOUR ALERTS
    ✓ ★ runway < 5 days      ✓ ★ rows in the default partition
    ✓ partition count > 200  ✓ per-partition dead tuples
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
Create a range-partitioned table with three monthly partitions and a default. Then: (a) prove the parent has no storage; (b) show pruning to one partition; (c) show all four pruning failures and fix each; (d) compare `DELETE` vs `DROP` for one month, reporting time and WAL; (e) drop the default and reproduce the "no partition found" outage.

### Exercise 2 — medium (apply it)
Build a partition manager: an idempotent creation function with 14 days of runway, per-partition autovacuum settings and index creation, plus a retention function that detaches and drops. Then write and test the four alerts.

Finally, measure planning time with 12, 120 and 1,200 partitions, and explain the trend.

### Exercise 3 — hard (production simulation)
A payments platform has a 412 GB, 4.1-billion-row `payment_attempts` table with a 90-day retention policy enforced by a nightly `DELETE` that runs 4h40m, causes 24 minutes of replica lag, keeps autovacuum permanently behind, and contributed to a transaction-ID wraparound outage.

(a) From `pg_stat_statements`, which query justifies partitioning — and why is it not the most frequent one?
(b) Compute the sizing. Argue for daily over weekly using the *retention policy*, not performance.
(c) The table has a `UNIQUE (idempotency_key)` constraint. Explain why it cannot survive partitioning as-is, and design the alternative.
(d) Write the partitioned DDL, including why the primary key changes and what that means for `id` uniqueness.
(e) Write the partition manager and the `DEFAULT` partition strategy. Explain why the default must stay empty.
(f) Write the zero-downtime migration: dual-write, backfill, verification, cutover. Why one transaction per day in the backfill?
(g) Write the retention function with archiving. Why `DETACH CONCURRENTLY` before `DROP`?
(h) Audit the hot queries. One doesn't constrain the partition key — show its plan before and after, and describe the product change required.
(i) Write the four alerts with thresholds. Which one prevents the most common partitioning incident?
(j) Explain how partitioning structurally eliminates the wraparound risk from Topic 47.

---

## Mental model checkpoint

1. What is the difference between partitioning and sharding?
2. What is the parent table physically? What does `relkind = 'p'` and `relfilenode = 0` tell you?
3. Give the single strongest argument for partitioning, with numbers.
4. Name the four ways partition pruning fails, and the fix for each.
5. What is run-time pruning and how do you see it in `EXPLAIN`?
6. Why must every unique constraint include the partition key? What does that imply for idempotency keys?
7. How do you create an index on a partitioned table without an `ACCESS EXCLUSIVE` lock?
8. Why does `ATTACH PARTITION` sometimes take minutes, and how do you make it instant?
9. What is the most common partitioning incident, and what three things prevent it?
10. What happens to planning time as partition count grows? Give rough numbers.
11. Why does partitioning eliminate transaction-ID wraparound risk?

---

## Quick reference card

```sql
CREATE TABLE events (…, PRIMARY KEY (id, created_at))   -- ★ key in the PK
  PARTITION BY RANGE (created_at);
CREATE TABLE events_2026_08 PARTITION OF events
  FOR VALUES FROM ('2026-08-01') TO ('2026-09-01');
CREATE TABLE events_default PARTITION OF events DEFAULT; -- ★ keep EMPTY
```

**Retention**
```sql
SET LOCAL lock_timeout = '5s';                              -- ★ always
ALTER TABLE events DETACH PARTITION events_2026_05 CONCURRENTLY;  -- PG14+
DROP TABLE events_2026_05;                                  -- ★ 12 ms
-- ✗ DELETE FROM events WHERE created_at < …                -- ★ 41 min
```

**Attach without a scan**
```sql
ALTER TABLE p ADD CONSTRAINT c CHECK (ts >= 'a' AND ts < 'b');  -- ★ first
ALTER TABLE parent ATTACH PARTITION p FOR VALUES FROM ('a') TO ('b');
```

**Index without a long lock**
```sql
CREATE INDEX i ON ONLY parent (col);              -- invalid, no lock
CREATE INDEX CONCURRENTLY i_p1 ON part1 (col);
ALTER INDEX i ATTACH PARTITION i_p1;              -- ★ repeat per partition
```

**Pruning**

| Predicate | Prunes? |
|---|---|
| `created_at >= 'a' AND < 'b'` | ★ **yes** |
| `tenant_id = 42` (key absent) | ★ **no — scans all** |
| `date_trunc('month', created_at) = …` | ★ **no** |
| `created_at >= $1` | ★ run-time (`Subplans Removed: N`) |

**Sizing:** 10–100 GB per partition · ★ **under ~1,000 partitions** · coarsest granularity meeting retention precision.
Planning time: 120 → 2 ms · 1,200 → 41 ms · ★ 10,000 → **880 ms**.

**★ Four alerts:** runway < 5 days · **any row in the default partition** · partition count > 200 · per-partition dead tuples.

**★ Genuine global uniqueness (idempotency keys) goes in a small unpartitioned side table.**

---

## When would I use this at work?

1. **Any table with a retention policy.** This is the whole argument: `DROP TABLE` is 12 ms and `DELETE` is 41 minutes plus dead tuples plus 18 GB of WAL plus 24 minutes of replica lag. If you have a "delete data older than N days" job, you should be partitioned.

2. **Any append-mostly, time-ordered table above ~100 GB.** Events, logs, metrics, audit trails, payment attempts, sessions. Per-partition autovacuum, independent freezing, and parallel dumps matter as much as query pruning.

3. **Before proposing sharding.** Partitioning gets you manageability on one machine with full transactional semantics and no distributed-systems cost. It is almost always the right step first, and often the only one you need (Topic 60).

4. **When a table has become unvacuumable.** Splitting a 400 GB table into 40 × 10 GB pieces lets autovacuum work in parallel and lets old partitions be dropped before they ever approach wraparound — which is a structural fix for the Topic 47 class of incident, not a mitigation.

---

## Connected topics

**Understand before this:** 04 (heap files), 47 (VACUUM, bloat and wraparound — the problem this solves), 45 (locks — `lock_timeout` on `DROP`/`ATTACH`), 58 (replica lag — the `DELETE` that caused it), 28 (zero-downtime migrations — the dual-write cutover).

**This unlocks:**
- **60** — sharding: when one server is no longer enough
- **61** — counters and hot rows: hash partitioning as a contention fix
- **64** — backup and PITR: per-partition backup strategies
- **73** — time-series: where partitioning is the primary design
- **Case study 07** — IoT telemetry · **Case study 08** — audit log
