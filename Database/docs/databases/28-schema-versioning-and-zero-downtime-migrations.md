# 28 — Schema Versioning and Zero-Downtime Migrations
## Phase: Database Design

---

## ELI5 — The Simple Analogy

Replacing the rails on a railway line **while the trains are still running.**

You cannot close the line. You cannot lift the old rail and lay the new one in the same instant. So you do it in stages: lay a new rail alongside, get the trains running on both, move the traffic over, then lift the old one. Each stage is individually safe, and at no point is there a gap.

Two things make it work.

**Every intermediate state must be valid.** There is never a moment where a train arrives and finds half a rail. If any single step leaves the system broken, the whole plan is wrong — because that step *will* be the one that runs when a deploy stalls.

**And the trains and the rails are different things that change at different times.** Old carriages must run on new rails, and new carriages must run on old rails — because during a rolling deploy, **both versions of your application are running against the same database at the same time.** That single fact is the entire discipline.

---

## Where this fits in the big picture

```
   20–27 — how to design a schema correctly
                          │
                          ▼
        ┌──────────────────────────────────────────┐
        │ 28 MIGRATIONS                            │ ← YOU ARE HERE
        │ how to CHANGE it without stopping        │
        └────────────────────┬─────────────────────┘
                             │  ← PHASE 3 ENDS HERE
                             ▼
                    PHASE 4 — NORMALISATION
                    (proving your design is correct —
                     and every fix is a migration)
```

Everything in Phase 3 assumed a blank page. **This topic is the only one about a schema that already has data and traffic** — which is every schema you will ever work on after week two.

---

## What is this?

**Schema versioning** is keeping the database's structure under version control, applied in order, tracked, and reproducible across every environment.

**Zero-downtime migration** is changing that structure while the application is serving traffic and while *two versions of the application code are running simultaneously*.

The governing pattern is **expand → migrate → contract**: add the new thing, move to it, remove the old thing — as separate deploys, never as one.

---

## Why does it matter for a backend developer?

Because migrations are where careful schema design meets a live system, and the failure modes are outages:

```
 ① THE LOCK THAT STOPS EVERYTHING
    ALTER TABLE orders ADD COLUMN … DEFAULT gen_random_uuid();
    → ACCESS EXCLUSIVE + a full table rewrite. On 500 GB: 40 minutes
      during which NOTHING can read or write `orders`.

 ② THE LOCK QUEUE — worse, and less obvious
    Your ALTER waits behind one long-running SELECT. Meanwhile EVERY
    subsequent query queues behind YOUR ALTER, because lock requests
    are FIFO. One 30-second report becomes a total outage.
    ★ This is the single most common migration incident, and it happens
      even with a migration that would have taken 2 milliseconds.

 ③ THE ROLLING-DEPLOY GAP
    You drop a column and deploy. For 90 seconds, half your pods are
    still running the old code that SELECTs it. Every request on those
    pods 500s.

 ④ THE IRREVERSIBLE STEP
    You drop the column, then discover the new code has a bug. There is
    no rollback — the data is gone.
```

Every one is preventable with a known technique. This topic is those techniques.

---

## The physical reality

### The lock levels, and what each blocks

```
 POSTGRESQL LOCK MODES, weakest to strongest:

 ACCESS SHARE          SELECT
 ROW SHARE             SELECT FOR UPDATE/SHARE
 ROW EXCLUSIVE         INSERT, UPDATE, DELETE
 SHARE UPDATE EXCLUSIVE  VACUUM, ANALYZE, CREATE INDEX CONCURRENTLY,
                         ALTER TABLE VALIDATE CONSTRAINT
 SHARE                 CREATE INDEX (non-concurrent)
 SHARE ROW EXCLUSIVE   CREATE TRIGGER, some ALTER TABLE forms
 EXCLUSIVE             REFRESH MATERIALIZED VIEW CONCURRENTLY
 ACCESS EXCLUSIVE      ★ most ALTER TABLE, DROP, TRUNCATE, VACUUM FULL,
                         REINDEX (non-concurrent)

 THE COMPATIBILITY THAT MATTERS:
   ACCESS EXCLUSIVE conflicts with EVERYTHING, including SELECT.
   SHARE UPDATE EXCLUSIVE conflicts only with itself and stronger —
     so reads AND writes continue. ★ This is the "safe" tier.
```

### ★ The lock queue — the mechanism that turns 2 ms into an outage

```
 t=0    a reporting query starts:  SELECT … FROM orders …   (30 s)
        → holds ACCESS SHARE on orders

 t=1    your migration runs:  ALTER TABLE orders ADD COLUMN x int;
        → requests ACCESS EXCLUSIVE
        → CONFLICTS with the report's ACCESS SHARE
        → ★ QUEUES

 t=2    a normal API query:  SELECT * FROM orders WHERE id=$1;
        → requests ACCESS SHARE
        → ACCESS SHARE is compatible with ACCESS SHARE…
        → ★ BUT POSTGRESQL'S LOCK QUEUE IS FIFO. It will not grant a
          lock that would jump ahead of a waiting stronger request.
        → QUEUES BEHIND YOUR ALTER.

 t=3..30  EVERY query on orders queues. The table is effectively down.
 t=30     the report finishes, the ALTER runs in 2 ms, the queue drains.

 ⇒ A 2-MILLISECOND MIGRATION CAUSED A 29-SECOND OUTAGE.
   And if the report had been 20 minutes, so would the outage.

 THE FIX — always, on every migration:
   SET lock_timeout = '3s';        -- give up rather than queue
   SET statement_timeout = '30s';
   and retry in a loop.
```

### What rewrites the table, and what doesn't

```
 ★ SAFE — metadata only, ACCESS EXCLUSIVE for MILLISECONDS:
   ADD COLUMN (nullable, no default)                    PG 9.x+
   ADD COLUMN … DEFAULT <constant>                      ★ PG 11+ only
   DROP COLUMN                                          (marks it dropped)
   ALTER COLUMN … DROP NOT NULL
   ALTER COLUMN … SET DEFAULT / DROP DEFAULT
   RENAME COLUMN / RENAME TABLE
   ALTER COLUMN varchar(50) → varchar(100)              PG 9.2+
   ALTER COLUMN varchar(n)  → text                      PG 9.2+
   ADD CONSTRAINT … NOT VALID                           (CHECK / FK)

 ⚠ REWRITES THE ENTIRE TABLE — ACCESS EXCLUSIVE for MINUTES OR HOURS:
   ADD COLUMN … DEFAULT <volatile>   e.g. gen_random_uuid(), now()
   ALTER COLUMN … TYPE <different>   int→bigint, float→numeric, text→uuid
   ALTER COLUMN … SET NOT NULL       (scans; see the trick below)
   ADD PRIMARY KEY                   (builds an index, non-concurrently)
   ADD CONSTRAINT … (without NOT VALID)   scans to validate
   SET TABLESPACE · CLUSTER · VACUUM FULL

 ★ THE PG11 CHANGE THAT PEOPLE MISS:
   Before PG11:  ADD COLUMN x int DEFAULT 0  → FULL REWRITE
   PG11 onwards: stored as a table-level "missing value"; existing rows
                 are unchanged and the default is materialised on read.
                 → METADATA ONLY, milliseconds.
   ⚠ BUT ONLY FOR A CONSTANT DEFAULT. `DEFAULT now()` or
     `DEFAULT gen_random_uuid()` is volatile and still rewrites.
```

### Two application versions, one database

```
 A ROLLING DEPLOY IS NOT ATOMIC.

  t=0    12 pods running v1
  t=1    2 pods v2, 10 pods v1      ★ BOTH TALKING TO THE SAME SCHEMA
  t=2    6 pods v2, 6 pods v1
  t=3    12 pods v2
  ...and if you roll back: 12 pods v1 again, against the NEW schema

 ⇒ THE TWO RULES THAT FOLLOW:
   ① THE SCHEMA MUST BE COMPATIBLE WITH v1 AND v2 SIMULTANEOUSLY.
   ② IT MUST STILL BE COMPATIBLE WITH v1 AFTER A ROLLBACK.

 ⇒ THEREFORE: every change is at least TWO deploys.
   You can add. You can start writing. But you can only REMOVE
   something after every running instance has stopped using it —
   which is a separate deploy, ideally a separate day.
```

---

## How it works — step by step

### The expand → migrate → contract pattern

```
 ═══ EXPAND (deploy 1) ═══════════════════════════════════════════
   Add the new structure. Change nothing that exists.
   ✓ old code works (it ignores the new thing)
   ✓ new code works (it can use the new thing)
   ✓ ROLLBACK IS FREE — the old code never saw it

 ═══ MIGRATE (deploy 2 + a backfill) ═════════════════════════════
   Start writing to BOTH old and new. Backfill history in batches.
   ✓ old code still reads the old thing
   ✓ new code reads the new thing
   ✓ rollback is still free

 ═══ CONTRACT (deploy 3, days later) ═════════════════════════════
   Stop using the old thing. Then, in a LATER deploy, drop it.
   ⚠ this is the only irreversible step — do it last, and only after
     you are certain no instance references the old structure.
```

### Recipe 1 — adding a column

```sql
-- ✓ SAFE (any PG version)
ALTER TABLE orders ADD COLUMN discount_minor bigint;

-- ✓ SAFE on PG 11+ — a constant default is metadata-only
ALTER TABLE orders ADD COLUMN discount_minor bigint NOT NULL DEFAULT 0;

-- ✗ REWRITES — volatile default
ALTER TABLE orders ADD COLUMN token uuid NOT NULL DEFAULT gen_random_uuid();
-- ⇒ SPLIT IT:
ALTER TABLE orders ADD COLUMN token uuid;                     -- ① instant
UPDATE orders SET token = gen_random_uuid()
 WHERE token IS NULL AND id BETWEEN $1 AND $2;                -- ② batched
ALTER TABLE orders ALTER COLUMN token SET DEFAULT gen_random_uuid();  -- ③
-- ④ then the NOT NULL, via the trick below
```

### Recipe 2 — adding `NOT NULL` without a scan

```sql
-- ✗ scans the whole table under ACCESS EXCLUSIVE
ALTER TABLE orders ALTER COLUMN token SET NOT NULL;

-- ★ THE FOUR-STEP TRICK (PG 12+)
ALTER TABLE orders ADD CONSTRAINT ck_token_nn
  CHECK (token IS NOT NULL) NOT VALID;          -- ① instant, brief lock
ALTER TABLE orders VALIDATE CONSTRAINT ck_token_nn;  -- ② SHARE UPDATE EXCLUSIVE
                                                     --    reads AND writes continue
ALTER TABLE orders ALTER COLUMN token SET NOT NULL;  -- ③ ★ NO SCAN —
                                                     --    PG uses the proven CHECK
ALTER TABLE orders DROP CONSTRAINT ck_token_nn;      -- ④ tidy up
```

### Recipe 3 — adding an index

```sql
-- ✗ SHARE lock — blocks ALL WRITES for the whole build
CREATE INDEX idx_orders_token ON orders (token);

-- ✓ does not block writes
CREATE INDEX CONCURRENTLY idx_orders_token ON orders (token);
```
```
 ⚠ CONCURRENTLY's caveats — all three matter:
  ① CANNOT run inside a transaction block. Most migration tools wrap
     each migration in one. You must opt out (Flyway: no-transaction;
     Rails: disable_ddl_transaction!; node-pg-migrate: {transaction:false}).
  ② TWO TABLE PASSES — 2–3× slower than a normal build.
  ③ ★ ON FAILURE IT LEAVES AN *INVALID* INDEX, which is still
     maintained on every write but unusable for queries — the worst of
     both worlds. ALWAYS verify:
       SELECT indexrelid::regclass FROM pg_index WHERE NOT indisvalid;
     and DROP INDEX CONCURRENTLY + retry if you find one.
  ④ It waits for all transactions older than it to finish. A long-running
     transaction stalls it indefinitely.
```

### Recipe 4 — renaming a column (the canonical expand/contract)

```
 ✗ NEVER: ALTER TABLE orders RENAME COLUMN total TO total_minor;
   The DDL is instant, but v1 pods immediately 500 on every query.

 ★ THE FIVE-DEPLOY DANCE:

 DEPLOY 1 — EXPAND
   ALTER TABLE orders ADD COLUMN total_minor bigint;
   -- app: unchanged. Reads/writes `total`.

 DEPLOY 2 — DUAL WRITE
   -- app: writes BOTH columns, reads `total`.
   -- (or a trigger, if you cannot change every writer — see below)

 BACKFILL (not a deploy — a batched job)
   UPDATE orders SET total_minor = total
    WHERE total_minor IS NULL AND id BETWEEN $1 AND $2;
   -- 10,000 rows per transaction, with a pause. NEVER one statement.

 DEPLOY 3 — SWITCH READS
   -- app: writes both, reads `total_minor`.
   -- ★ rollback is still safe: `total` is still current.

 DEPLOY 4 — STOP WRITING THE OLD
   -- app: writes and reads `total_minor` only.
   -- ⚠ from here, a rollback to deploy 2 would lose new writes to `total`.
   --   Leave a gap of days.

 DEPLOY 5 — CONTRACT
   ALTER TABLE orders DROP COLUMN total;
```

**When you cannot change every writer** (other services, legacy jobs), use a trigger for the dual write:

```sql
CREATE FUNCTION sync_total() RETURNS trigger AS $$
BEGIN
  IF NEW.total_minor IS NULL AND NEW.total IS NOT NULL THEN
    NEW.total_minor := NEW.total;
  ELSIF NEW.total IS NULL AND NEW.total_minor IS NOT NULL THEN
    NEW.total := NEW.total_minor;
  END IF;
  RETURN NEW;
END $$ LANGUAGE plpgsql;
CREATE TRIGGER t_sync_total BEFORE INSERT OR UPDATE ON orders
  FOR EACH ROW EXECUTE FUNCTION sync_total();
-- ⚠ CREATE TRIGGER takes SHARE ROW EXCLUSIVE — brief, but use lock_timeout.
```

### Recipe 5 — changing a column type

```sql
-- ✗ REWRITES the table AND every index AND every child FK column
ALTER TABLE orders ALTER COLUMN id TYPE bigint;

-- ★ EXPAND/CONTRACT with a new column, same five deploys as a rename:
ALTER TABLE orders ADD COLUMN id_new bigint;
-- dual-write, backfill in batches, switch reads, then:
--   ⚠ AND every child table's FK column must migrate too, in the same
--     dance, before you can swap the PK. This is why int4→bigint on a
--     large schema is a multi-week project, not a migration.
```

### Recipe 6 — dropping a column safely

```sql
-- The DDL itself is instant (it marks the column dropped; the bytes
-- remain until the next rewrite). The DANGER is entirely deployment order.

-- DEPLOY 1: remove all references from the application. Ship it. Wait.
-- VERIFY nothing still selects it:
SELECT calls, query FROM pg_stat_statements WHERE query ILIKE '%old_column%';
-- DEPLOY 2 (days later):
ALTER TABLE orders DROP COLUMN old_column;

-- ★ SAFER STILL, for a big or scary drop — make it invisible first:
--   Rename it out of the way for a week. If nothing breaks, drop it.
ALTER TABLE orders RENAME COLUMN old_column TO zz_deprecated_old_column;
-- (still instant, and trivially reversible)
```

### The batched backfill — the pattern for every data migration

```sql
-- ✗ NEVER: one statement over 500M rows
UPDATE orders SET total_minor = total WHERE total_minor IS NULL;
--   → one transaction, hours long, holds a snapshot (blocking vacuum
--     cluster-wide), generates 500M dead tuples, and cannot be paused.

-- ★ BATCHED, resumable, pausable, and kind to replicas:
DO $$
DECLARE
  last_id bigint := 0;
  n int;
BEGIN
  LOOP
    UPDATE orders SET total_minor = total
     WHERE id > last_id AND id <= last_id + 10000 AND total_minor IS NULL;
    GET DIAGNOSTICS n = ROW_COUNT;
    last_id := last_id + 10000;
    EXIT WHEN last_id > (SELECT max(id) FROM orders);
    COMMIT;                       -- ★ each batch is its own transaction
    PERFORM pg_sleep(0.05);       -- ★ let replicas and autovacuum breathe
  END LOOP;
END $$;
```
```
 WHY EACH ELEMENT:
   batch size 10k    short transactions; no long snapshot (Topic 46)
   COMMIT per batch  resumable if it dies; dead tuples get vacuumed as
                     you go instead of accumulating 500M of them
   pg_sleep          throttles WAL generation so replicas don't lag
                     (Topic 62) — the most-forgotten line
   id-range cursor   deterministic progress; no OFFSET, no re-scanning
```

### Migration tooling — what a versioning system must give you

```
 ① ORDERED, IMMUTABLE FILES        001_x.sql, 002_y.sql — never edited
                                    after being applied anywhere
 ② A TRACKING TABLE                schema_migrations(version, applied_at,
                                    checksum)
 ③ CHECKSUMS                       detect an edited file
 ④ A LOCK                          two pods deploying at once must not
                                    both run migration 042
 ⑤ TRANSACTION CONTROL PER FILE    so CONCURRENTLY can opt out
 ⑥ RUNS IN CI                      against a restored production dump,
                                    timed, before it ever runs live

 ★ THE RULE ABOUT DOWN MIGRATIONS:
   Write them, but do not rely on them. In production you roll FORWARD.
   A `down` that drops a column you just backfilled loses data. The real
   rollback plan is "the previous deploy's code still works against this
   schema" — which is exactly what expand/contract guarantees.
```

---

## Concept breakdown

```
THE THREE PHASES
├── EXPAND    add the new. Old and new code both work. Rollback free.
├── MIGRATE   dual-write + batched backfill. Rollback still free.
└── CONTRACT  remove the old. ★ THE ONLY IRREVERSIBLE STEP. Do it last,
              days later, after verifying nothing references it.

THE THREE THINGS THAT CAUSE MIGRATION OUTAGES
├── ① THE LOCK ITSELF        ACCESS EXCLUSIVE on a big table
├── ② ★ THE LOCK QUEUE       a 2 ms ALTER behind a 30 s SELECT blocks
│                            EVERYTHING for 30 s. FIFO, no exceptions.
└── ③ THE DEPLOY GAP         two app versions, one schema

SAFE vs REWRITING
  SAFE (ms): ADD COLUMN (constant default, PG11+) · DROP COLUMN ·
             RENAME · DROP NOT NULL · SET DEFAULT · varchar widening ·
             ADD CONSTRAINT … NOT VALID
  REWRITE:   volatile DEFAULT · TYPE change · SET NOT NULL (unless the
             CHECK trick) · ADD PRIMARY KEY · ADD CONSTRAINT (validating)

THE SAFE LOCK TIER
  SHARE UPDATE EXCLUSIVE — reads AND writes continue:
    CREATE INDEX CONCURRENTLY · VALIDATE CONSTRAINT · ANALYZE · VACUUM
  ⇒ every long operation should be in this tier. If it isn't, split it
    into a NOT VALID step plus a VALIDATE step.

EVERY MIGRATION SCRIPT STARTS WITH
    SET lock_timeout = '3s';
    SET statement_timeout = '60s';
  ⇒ so a failed migration is a failed migration, not an outage.
  ⇒ and the deploy pipeline RETRIES it.

THE BACKFILL RULES
  batched · one transaction per batch · a cursor, not OFFSET ·
  pg_sleep between batches · resumable · idempotent
```

---

## Diagrams

**Diagram 1 — big picture: expand → migrate → contract**

```
  SCHEMA         old column ████████████████████░░░░░░░░░░░░  dropped
                 new column ░░░░████████████████████████████
                            │   │        │      │         │
  APP v1         ███████████│███│        │      │         │
  APP v2         ░░░░░░░░░░░░███│████████│      │         │
  APP v3         ░░░░░░░░░░░░░░░░░░░░░░░░████████│         │
  APP v4         ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░██████████│
                            │   │        │      │         │
                       DEPLOY 1 │   DEPLOY 3    │      DEPLOY 5
                       (expand) │   (read new)  │     (contract)
                            DEPLOY 2       DEPLOY 4
                          (dual write)   (stop old write)
                                 │
                            BACKFILL (batched, between 2 and 3)

  ★ At EVERY vertical slice, the running app version and the schema
    are compatible. That is the whole property.
```

**Diagram 2 — data flow: the lock queue outage**

```
  t=0   ┌─ SELECT (report) ────────────────────────────────┐ 30 s
        │  holds ACCESS SHARE                              │
  t=1   │  ┌─ ALTER TABLE ─ ⏸ WAITING for ACCESS EXCLUSIVE │
        │  │                                               │
  t=2   │  │  ┌─ SELECT (api) ─ ⏸ QUEUED behind the ALTER  │
  t=3   │  │  ├─ SELECT (api) ─ ⏸                          │
  t=4   │  │  ├─ INSERT      ─ ⏸                           │
        │  │  ├─ …everything… ⏸                            │
  t=30  └──┤  │                                            │
           └──┤ ALTER runs (2 ms)                          │
              └─ queue drains ──────────────────────────────┘

  ★ The API queries are compatible with the report's lock. They queue
    ONLY because PostgreSQL will not let them jump ahead of a waiting
    stronger request. 29 seconds of outage from a 2 ms migration.

  WITH lock_timeout = '3s':
  t=1   ALTER requests, waits 3 s, FAILS. Nothing else ever queues.
        The deploy retries in a minute. No outage.
```

**Diagram 3 — before/after: the backfill**

```
 ✗ ONE STATEMENT
 ┌──────────────────────────────────────────────────────────────────┐
 │ UPDATE orders SET total_minor = total WHERE total_minor IS NULL; │
 │                                                                  │
 │ ████████████████████████████████████████████ 4 hours             │
 │  • one snapshot held for 4 h → NO vacuum ANYWHERE in the cluster │
 │  • 500M dead tuples created at once                              │
 │  • 180 GB of WAL in one burst → replicas lag 40 minutes          │
 │  • cannot be paused, cannot be resumed, all-or-nothing rollback  │
 └──────────────────────────────────────────────────────────────────┘

 ★ BATCHED
 ┌──────────────────────────────────────────────────────────────────┐
 │ 50,000 batches of 10,000 rows, 50 ms apart                       │
 │                                                                  │
 │ ▌▌▌▌▌▌▌▌▌▌▌▌▌▌▌▌▌▌▌▌▌▌▌▌▌▌▌▌▌▌▌▌▌▌ 6 hours (slower!)             │
 │  • longest snapshot: ~40 ms                                      │
 │  • autovacuum keeps up continuously                              │
 │  • WAL spread evenly → replica lag < 1 s                         │
 │  • pausable, resumable, observable                               │
 └──────────────────────────────────────────────────────────────────┘
        ↑ 50% LONGER IN WALL-CLOCK TIME, and unambiguously correct.
          Backfills are not optimised for speed. They are optimised
          for not being an incident.
```

---

## Example 1 — basic

**Step 1 — prove the lock queue.**

```sql
CREATE TABLE lq (id bigserial PRIMARY KEY, v text);
INSERT INTO lq (v) SELECT 'x' FROM generate_series(1,2000000);
```
```sql
-- session 1: a long read
BEGIN;
SELECT count(*) FROM lq;
SELECT pg_sleep(30);          -- simulate a slow report; keep the txn open
```
```sql
-- session 2: a trivial migration
ALTER TABLE lq ADD COLUMN w int;      -- ⏸ WAITS
```
```sql
-- session 3: an ordinary API query
SELECT count(*) FROM lq;              -- ⏸ ★ ALSO WAITS
```
```sql
-- session 4: see it
SELECT pid, mode, granted, query FROM pg_locks l
JOIN pg_stat_activity a USING (pid) WHERE relation='lq'::regclass;
```
```
  pid  |        mode         | granted | query
-------+---------------------+---------+----------------------------
 41022 | AccessShareLock     | t       | SELECT pg_sleep(30)
 41055 | AccessExclusiveLock | f       | ALTER TABLE lq ADD COLUMN w
 41088 | AccessShareLock     | f       | SELECT count(*) FROM lq     ★
```
**Session 3's `SELECT` is compatible with session 1's `SELECT`. It waits only because it will not jump the queue.**

Now with a timeout:
```sql
-- session 2, retry
SET lock_timeout = '3s';
ALTER TABLE lq ADD COLUMN w int;
-- ERROR:  canceling statement due to lock timeout
--   ★ The migration failed. Nothing else ever queued. Retry in a minute.
```

**Step 2 — safe vs rewriting `ADD COLUMN`.**

```sql
CREATE TABLE big (id bigserial PRIMARY KEY, v text);
INSERT INTO big (v) SELECT repeat('x',80) FROM generate_series(1,5000000);
VACUUM ANALYZE big;
SELECT pg_size_pretty(pg_relation_size('big'));   -- 578 MB

\timing on
ALTER TABLE big ADD COLUMN a int;                          -- 3 ms   ✓
ALTER TABLE big ADD COLUMN b int NOT NULL DEFAULT 0;       -- 4 ms   ✓ PG11+
ALTER TABLE big ADD COLUMN c uuid DEFAULT gen_random_uuid(); -- 41,204 ms ⚠
SELECT pg_size_pretty(pg_relation_size('big'));   -- 923 MB — REWRITTEN
```
**A constant default is metadata. A volatile default rewrites 5 million rows.**

**Step 3 — the `NOT NULL` trick.**

```sql
UPDATE big SET a = 1;                     -- make it non-null everywhere
\timing on
ALTER TABLE big ALTER COLUMN a SET NOT NULL;      -- 1,842 ms (full scan)
ALTER TABLE big ALTER COLUMN a DROP NOT NULL;

ALTER TABLE big ADD CONSTRAINT ck_a CHECK (a IS NOT NULL) NOT VALID;  -- 2 ms
ALTER TABLE big VALIDATE CONSTRAINT ck_a;         -- 1,204 ms, WEAK LOCK
ALTER TABLE big ALTER COLUMN a SET NOT NULL;      -- ★ 3 ms — no scan
ALTER TABLE big DROP CONSTRAINT ck_a;
```
**The same total time — but the 1.2-second step ran under `SHARE UPDATE EXCLUSIVE`, so reads and writes continued.**

**Step 4 — `CREATE INDEX` vs `CONCURRENTLY`.**

```sql
-- terminal 1
CREATE INDEX idx_big_v ON big (v);
-- terminal 2, during the build:
INSERT INTO big (v) VALUES ('test');    -- ⏸ BLOCKS until the build finishes
```
```sql
DROP INDEX idx_big_v;
-- terminal 1
CREATE INDEX CONCURRENTLY idx_big_v ON big (v);
-- terminal 2:
INSERT INTO big (v) VALUES ('test');    -- ✓ succeeds immediately
```

And the invalid-index trap:
```sql
-- simulate a failure: cancel a CONCURRENTLY build partway
SELECT indexrelid::regclass, indisvalid FROM pg_index WHERE NOT indisvalid;
```
```
    indexrelid    | indisvalid
------------------+------------
 idx_big_v        | f            ★ maintained on every write, unusable for reads
```
```sql
DROP INDEX CONCURRENTLY idx_big_v;   -- and retry
```

**Step 5 — the batched backfill.**

```sql
ALTER TABLE big ADD COLUMN v_upper text;

-- ✗ one statement
\timing on
UPDATE big SET v_upper = upper(v);          -- 28,104 ms, one transaction
-- check the damage:
SELECT n_dead_tup FROM pg_stat_user_tables WHERE relname='big';   -- 5,000,001

-- ★ batched
UPDATE big SET v_upper = NULL;
DO $$ DECLARE last_id bigint := 0; mx bigint; BEGIN
  SELECT max(id) INTO mx FROM big;
  WHILE last_id < mx LOOP
    UPDATE big SET v_upper = upper(v)
     WHERE id > last_id AND id <= last_id + 10000 AND v_upper IS NULL;
    last_id := last_id + 10000;
    COMMIT;
    PERFORM pg_sleep(0.01);
  END LOOP;
END $$;
-- longest transaction: ~35 ms. Autovacuum keeps up throughout.
```

---

## Example 2 — production scenario

**The situation.** A payments platform, 2.4 TB. `payments.amount` is `float8` and must become `bigint` minor units (Topic 26's correctness bug). The table has 890 million rows, 40,000 writes/second, six FK children, and cannot be taken offline.

**Step 1 — why the obvious approach is impossible.**

```sql
ALTER TABLE payments ALTER COLUMN amount TYPE bigint USING (amount*100)::bigint;
```
```
 • ACCESS EXCLUSIVE on payments
 • rewrites 890M rows: ~4 hours at 2.4 TB
 • rebuilds all 7 indexes
 • ★ during which payments is COMPLETELY UNAVAILABLE
 ⇒ a 4-hour total outage. Not viable.
```

**Step 2 — the plan, as five deploys plus a backfill.**

```sql
-- ═══ DEPLOY 1 — EXPAND (instant) ═══
SET lock_timeout = '3s';
ALTER TABLE payments ADD COLUMN amount_minor bigint;
ALTER TABLE payments ADD COLUMN currency char(3);
-- app: unchanged.  ROLLBACK: free.
```

```sql
-- ═══ DEPLOY 2 — DUAL WRITE ═══
-- The app writes both. But there are 4 other services and 2 cron jobs
-- writing to this table, so use a trigger as a safety net.
CREATE FUNCTION sync_amount() RETURNS trigger AS $$
BEGIN
  IF NEW.amount_minor IS NULL AND NEW.amount IS NOT NULL THEN
    NEW.amount_minor := round(NEW.amount::numeric * 100)::bigint;
    NEW.currency := COALESCE(NEW.currency, 'INR');
  ELSIF NEW.amount IS NULL AND NEW.amount_minor IS NOT NULL THEN
    NEW.amount := NEW.amount_minor / 100.0;
  END IF;
  RETURN NEW;
END $$ LANGUAGE plpgsql;

SET lock_timeout = '3s';
CREATE TRIGGER t_sync_amount BEFORE INSERT OR UPDATE ON payments
  FOR EACH ROW EXECUTE FUNCTION sync_amount();
-- ⚠ SHARE ROW EXCLUSIVE — brief, but retry on lock_timeout.
```

```sql
-- ═══ BACKFILL — batched, throttled, resumable, observable ═══
CREATE TABLE backfill_progress (
  job text PRIMARY KEY, last_id bigint NOT NULL, updated_at timestamptz NOT NULL
);
INSERT INTO backfill_progress VALUES ('payments_amount_minor', 0, now());
```
```js
// Run as a job, not a migration — it takes hours and must be pausable.
async function backfill() {
  const BATCH = 10_000;
  for (;;) {
    const { rows } = await db.query(`
      WITH cur AS (SELECT last_id FROM backfill_progress
                    WHERE job='payments_amount_minor' FOR UPDATE),
      upd AS (
        UPDATE payments p
           SET amount_minor = round(p.amount::numeric * 100)::bigint,
               currency = COALESCE(p.currency,'INR')
         WHERE p.id > (SELECT last_id FROM cur)
           AND p.id <= (SELECT last_id FROM cur) + $1
           AND p.amount_minor IS NULL
        RETURNING p.id)
      UPDATE backfill_progress
         SET last_id = (SELECT last_id FROM cur) + $1, updated_at = now()
       WHERE job='payments_amount_minor'
      RETURNING last_id, (SELECT count(*) FROM upd) AS n`, [BATCH]);

    const { last_id, n } = rows[0];
    if (last_id >= MAX_ID) break;

    // ★ THROTTLE ON REPLICA LAG — the most-forgotten part
    const lag = await db.query(
      `SELECT max(EXTRACT(epoch FROM replay_lag)) AS s FROM pg_stat_replication`);
    await sleep(lag.rows[0].s > 5 ? 5000 : 50);
  }
}
```

```sql
-- ★ VERIFY BEFORE SWITCHING READS — this is money.
SELECT count(*) AS mismatched FROM payments
WHERE amount_minor IS DISTINCT FROM round(amount::numeric*100)::bigint;
-- must be 0

SELECT count(*) AS unbackfilled FROM payments WHERE amount_minor IS NULL;
-- must be 0

-- and reconcile the TOTAL against an independent source (case study 03)
SELECT sum(amount_minor)/100.0 AS new_total, sum(amount) AS old_total,
       sum(amount_minor)/100.0 - sum(amount)::numeric AS drift
FROM payments WHERE captured_at >= '2026-01-01';
-- ⚠ drift will be non-zero, because the FLOAT values were already
--   wrong. Quantify it, document it, and get finance's sign-off on the
--   rounding rule BEFORE deploy 3.
```

```sql
-- ═══ DEPLOY 3 — SWITCH READS ═══
-- app reads amount_minor, still writes both.
-- Add the constraints now that the data is proven:
ALTER TABLE payments ADD CONSTRAINT ck_amount_minor_pos
  CHECK (amount_minor > 0) NOT VALID;
ALTER TABLE payments VALIDATE CONSTRAINT ck_amount_minor_pos;   -- weak lock
-- and the NOT NULL, via the trick:
ALTER TABLE payments ADD CONSTRAINT ck_am_nn CHECK (amount_minor IS NOT NULL) NOT VALID;
ALTER TABLE payments VALIDATE CONSTRAINT ck_am_nn;
ALTER TABLE payments ALTER COLUMN amount_minor SET NOT NULL;    -- no scan
ALTER TABLE payments DROP CONSTRAINT ck_am_nn;
```

```sql
-- ═══ DEPLOY 4 — STOP WRITING THE OLD (a week later) ═══
DROP TRIGGER t_sync_amount ON payments;
DROP FUNCTION sync_amount();
-- app writes amount_minor only.
-- ⚠ from here, rolling back to deploy 3 loses new writes to `amount`.
--   Verify with pg_stat_statements that nothing references it:
SELECT calls, left(query,80) FROM pg_stat_statements
WHERE query ~* '\mamount\M' AND query !~* 'amount_minor';
-- must be empty
```

```sql
-- ═══ DEPLOY 5 — CONTRACT (two weeks later) ═══
-- Make it invisible first, for a week:
ALTER TABLE payments RENAME COLUMN amount TO zz_deprecated_amount;
-- …one week of no incidents…
ALTER TABLE payments DROP COLUMN zz_deprecated_amount;
```

**Step 3 — the pre-flight checklist that runs in CI.**

```sql
-- Against a restored production dump, before the migration ever runs live:
--  ① time every statement
--  ② assert no statement takes ACCESS EXCLUSIVE for > 1 s
--  ③ assert no statement rewrites a table > 1 GB
--  ④ assert lock_timeout and statement_timeout are set

-- detect a rewrite: relfilenode changes
SELECT relfilenode FROM pg_class WHERE relname='payments';   -- before
-- …run the migration…
SELECT relfilenode FROM pg_class WHERE relname='payments';   -- after
-- ★ if it changed, the table was rewritten. Fail the build.
```

**Step 4 — what actually went wrong, and what caught it.**

| Incident | Caught by |
|---|---|
| The trigger's `round()` disagreed with the app's rounding for 412 rows | the `mismatched` verification query, before deploy 3 |
| The backfill caused 8 minutes of replica lag on the first run | the replica-lag throttle (added after) |
| A cron job in another repo still wrote `amount` at deploy 4 | the `pg_stat_statements` check |
| `CREATE INDEX CONCURRENTLY` failed overnight, leaving an invalid index | the daily `indisvalid` monitor |

**Step 5 — the numbers.**

| | Naive | Expand/contract |
|---|---|---|
| Downtime | **4 hours** | **0** |
| Elapsed time | 4 hours | ~3 weeks |
| Rollback possible | no | yes, until deploy 4 |
| Replica lag | 40+ min | < 5 s (throttled) |
| Verification before commit | none | 3 queries + finance sign-off |

**Three weeks instead of four hours — and that is the correct trade.**

---

## Common mistakes

**1. No `lock_timeout` on migrations.**
- *Symptom:* a trivial `ALTER` causes a full outage.
- *Engine-level why:* the lock queue is FIFO; a waiting `ACCESS EXCLUSIVE` blocks every subsequent request, even compatible ones.
- *Fix:* `SET lock_timeout = '3s'` at the top of every migration, and retry in the pipeline.

**2. `ADD COLUMN` with a volatile default.**
- *Symptom:* a 40-minute rewrite on a table you thought was safe.
- *Engine-level why:* PG11+ stores a *constant* default as metadata. A volatile expression must be evaluated per row.
- *Fix:* add the column nullable, backfill in batches, then `SET DEFAULT`.

**3. `CREATE INDEX` without `CONCURRENTLY`.**
- *Symptom:* all writes blocked for the build duration.
- *Fix:* `CONCURRENTLY`, with the transaction opt-out, and **verify `indisvalid` afterwards**.

**4. A single-statement backfill.**
- *Symptom:* hours-long transaction, replica lag, vacuum blocked cluster-wide, cannot be paused.
- *Fix:* batched, one transaction per batch, cursor-based, throttled on replica lag.

**5. Renaming or dropping in one deploy.**
- *Symptom:* 500s for the duration of the rolling deploy.
- *Fix:* expand/contract. Every removal is a separate, later deploy.

**6. Forgetting that rollback must also work.**
- *Symptom:* you roll back the code and the old version breaks against the new schema.
- *Fix:* the schema must be compatible with N−1 *and* N. That's what makes contract the last step.

**7. Not testing the migration against production-sized data.**
- *Symptom:* 40 ms in staging (10k rows), 40 minutes in production (890M rows).
- *Fix:* run every migration in CI against a restored dump, time it, and fail the build on `ACCESS EXCLUSIVE` over a threshold.

**8. Editing an already-applied migration file.**
- *Symptom:* environments diverge; a checksum mismatch, or silently different schemas.
- *Fix:* migrations are immutable once applied anywhere. Fix forward with a new file.

---

## Hands-on proof

**PROVE IT #1 — the lock queue.** (Example 1, step 1.)
**PROVE IT #2 — constant vs volatile default.** (Example 1, step 2.)
**PROVE IT #3 — the `NOT NULL` trick.** (Example 1, step 3.)
**PROVE IT #4 — `CONCURRENTLY` and the invalid-index trap.** (Example 1, step 4.)
**PROVE IT #5 — batched vs single-statement backfill.** (Example 1, step 5.)

**PROVE IT #6 — detect a table rewrite.**
```sql
SELECT relfilenode FROM pg_class WHERE relname='big';
ALTER TABLE big ADD COLUMN z int;                        -- metadata only
SELECT relfilenode FROM pg_class WHERE relname='big';    -- ★ unchanged
ALTER TABLE big ALTER COLUMN z TYPE bigint;              -- rewrite
SELECT relfilenode FROM pg_class WHERE relname='big';    -- ★ changed
```

**PROVE IT #7 — see what a statement would lock.**
```sql
BEGIN;
ALTER TABLE big ADD COLUMN q int;
SELECT locktype, relation::regclass, mode, granted
FROM pg_locks WHERE pid = pg_backend_pid() AND relation IS NOT NULL;
ROLLBACK;
```

**PROVE IT #8 — monitor for invalid indexes and long-running DDL.**
```sql
-- invalid indexes (pure cost)
SELECT indexrelid::regclass FROM pg_index WHERE NOT indisvalid;

-- anything holding ACCESS EXCLUSIVE right now
SELECT a.pid, now()-a.query_start AS held, l.relation::regclass, left(a.query,60)
FROM pg_locks l JOIN pg_stat_activity a USING (pid)
WHERE l.mode='AccessExclusiveLock' AND l.granted;

-- anything BLOCKED
SELECT a.pid, now()-a.query_start AS waited, left(a.query,60)
FROM pg_locks l JOIN pg_stat_activity a USING (pid) WHERE NOT l.granted;
```

---

## The design decision framework

```
EVERY MIGRATION SCRIPT, WITHOUT EXCEPTION, STARTS WITH:
    SET lock_timeout = '3s';
    SET statement_timeout = '60s';
  and the pipeline RETRIES on failure.

CLASSIFY THE CHANGE:
  □ ADDITIVE (new column/table/index)         → safe, one deploy
  □ MODIFYING (type, NOT NULL, constraint)    → NOT VALID + VALIDATE,
                                                 or expand/contract
  □ REMOVING (drop column/table)              → ★ expand/contract,
                                                 contract LAST, days later
  □ DATA (backfill)                           → batched job, NOT a migration

FOR EACH STATEMENT, ASK:
  ① What lock level? (ACCESS EXCLUSIVE = an outage on a big table)
  ② Does it rewrite the table? (check relfilenode in CI)
  ③ How long on PRODUCTION-SIZED data? (test against a restored dump)
  ④ Is the schema compatible with the CURRENTLY-DEPLOYED code?
  ⑤ Is it compatible with the PREVIOUS code, in case of rollback?
  ⇒ any "no" or "don't know" → split it.

THE RECIPES:
  add a column                → nullable, or a CONSTANT default (PG11+)
  add NOT NULL                → CHECK NOT VALID → VALIDATE → SET NOT NULL
  add an index                → CONCURRENTLY + verify indisvalid
  add a constraint            → NOT VALID → VALIDATE
  add a foreign key           → index the child FIRST, then NOT VALID → VALIDATE
  rename a column             → 5 deploys (expand/dual-write/read/stop/drop)
  change a type               → same 5 deploys, via a new column
  drop a column               → remove references, wait, rename to zz_, wait, drop
  backfill                    → batched job, throttled on replica lag

VERSIONING REQUIREMENTS:
  ordered · immutable once applied · checksummed · a deploy lock ·
  per-file transaction control · run in CI against a production dump

ROLLBACK PHILOSOPHY:
  ★ Roll FORWARD. `down` migrations are for local development.
    The real rollback guarantee is "the previous code still works
    against this schema" — which expand/contract gives you by design.

THE SIGNAL TO LOOK FOR — in every migration review:
  • no lock_timeout                       → reject
  • a DEFAULT that isn't a constant       → reject; split it
  • CREATE INDEX without CONCURRENTLY     → reject
  • ADD CONSTRAINT without NOT VALID      → reject
  • a DROP in the same PR as the code
    that stopped using it                 → reject; separate deploys
  • an UPDATE with no WHERE bound         → reject; batch it
  • untested against production volume    → reject
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
Reproduce the lock queue: hold a long `SELECT` in one session, run an `ALTER TABLE ADD COLUMN` in a second, then a plain `SELECT` in a third. Show via `pg_locks` that the third is blocked, and explain why it waits despite being compatible with the first. Then repeat with `lock_timeout` and show the difference.

### Exercise 2 — medium (apply it)
On a 5M-row table, demonstrate all four of: (a) a metadata-only `ADD COLUMN`, (b) a rewriting `ADD COLUMN`, (c) the four-step `NOT NULL` trick, (d) `CREATE INDEX` vs `CONCURRENTLY` blocking writes. For each, prove your claim with `relfilenode`, timing, and a concurrent write from a second session.

### Exercise 3 — hard (production simulation)
`payments.amount` is `float8` on an 890M-row, 2.4 TB table with 40,000 writes/sec, 7 indexes, 6 FK children, and 4 other services writing to it. It must become `bigint` minor units plus a `currency` column, with zero downtime.

(a) Explain precisely why `ALTER COLUMN … TYPE` is not viable, including what it does to the indexes and the FK children.
(b) Write the complete five-deploy plan, with the exact DDL for each and the lock level of every statement.
(c) Write the batched backfill, including replica-lag throttling and resumability across a restart.
(d) Write the three verification queries that must pass before switching reads. One of them will return a non-zero result **by design** — identify it and explain what to do.
(e) Two other services write to this table and you do not own their code. Design the trigger-based safety net, and explain the lock it takes.
(f) After deploy 4, how do you *prove* nothing still writes the old column? Give the query and its limitation.
(g) At deploy 5, describe the safer two-step drop and why the intermediate step is worth a week.
(h) Write the CI check that fails a migration which would take `ACCESS EXCLUSIVE` for more than 1 second on production-sized data.

---

## Mental model checkpoint

1. Explain the lock queue. Why does a compatible `SELECT` wait behind a pending `ALTER`?
2. Name four `ALTER TABLE` operations that are metadata-only and four that rewrite the table.
3. What changed in PostgreSQL 11 about `ADD COLUMN … DEFAULT`? What still rewrites?
4. Give the four-step zero-downtime `NOT NULL` procedure and explain why step 3 doesn't scan.
5. Name three caveats of `CREATE INDEX CONCURRENTLY`. Which one leaves you worse off than before?
6. Why must a schema be compatible with both the current *and* the previous application version?
7. Why is a batched backfill often *slower* in wall-clock time, and why is that the right choice?

---

## Quick reference card

**Lock levels (safe tier in bold)**

| Operation | Lock |
|---|---|
| `SELECT` | ACCESS SHARE |
| `INSERT/UPDATE/DELETE` | ROW EXCLUSIVE |
| **`CREATE INDEX CONCURRENTLY`** | **SHARE UPDATE EXCLUSIVE** |
| **`VALIDATE CONSTRAINT`** | **SHARE UPDATE EXCLUSIVE** |
| `CREATE INDEX` | SHARE (blocks writes) |
| Most `ALTER TABLE`, `DROP` | **ACCESS EXCLUSIVE** (blocks everything) |

**Safe vs rewriting**

| Safe (ms) | Rewrites |
|---|---|
| `ADD COLUMN` nullable | `ADD COLUMN` volatile default |
| `ADD COLUMN` **constant** default (PG11+) | `ALTER COLUMN … TYPE` |
| `DROP COLUMN`, `RENAME` | `SET NOT NULL` (without the trick) |
| `DROP NOT NULL`, `SET DEFAULT` | `ADD PRIMARY KEY` |
| `varchar(n)` → `varchar(n+k)` / `text` | `ADD CONSTRAINT` (validating) |
| `ADD CONSTRAINT … NOT VALID` | `VACUUM FULL`, `CLUSTER` |

**The recipes**

```sql
SET lock_timeout = '3s'; SET statement_timeout = '60s';   -- ALWAYS

-- NOT NULL, no scan
ADD CONSTRAINT ck CHECK (c IS NOT NULL) NOT VALID; VALIDATE; SET NOT NULL; DROP ck;
-- index
CREATE INDEX CONCURRENTLY …;  then verify indisvalid
-- constraint / FK
ADD CONSTRAINT … NOT VALID;  then VALIDATE CONSTRAINT …
-- rename / retype: 5 deploys — expand · dual-write · read-new · stop-old · drop
-- backfill: batched, one txn per batch, cursor, throttled on replica lag
```

**The three phases:** expand → migrate → contract. Contract is the only irreversible step; do it last, days later.

---

## When would I use this at work?

1. **Every migration PR review.** The seven-item reject list catches the outage-causing patterns in about a minute, and gives you a specific, non-negotiable reason for each.

2. **A "simple" schema change on a large table.** Knowing that `ADD COLUMN … DEFAULT now()` rewrites 890 million rows — and that the constant-default form doesn't — is the difference between a 3 ms deploy and a 40-minute outage.

3. **Planning a type change.** Being able to lay out the five-deploy plan with timings and rollback points turns "we can't fix the float money column" into a scheduled three-week project with no downtime.

---

## Connected topics

**Understand before this:** 22–27 (everything you might need to change), 23 (`NOT VALID` for FKs), 24 (`NOT VALID` for CHECK), 13 (`CREATE INDEX CONCURRENTLY`).

**This unlocks:**
- **29–38** — normalisation: every fix you identify is a migration you now know how to ship
- **45** — locks, in full
- **59** — partitioning an existing table, the hardest migration of all
- **62** — replication lag, which every backfill must respect
- **Case studies** — every "what you actually ship" section assumes these techniques
