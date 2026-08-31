# 79 — Capstone III: The Principal-Engineer Review
## Phase: Capstones

---

## ELI5 — The Simple Analogy

A building inspector walking a site.

They do not read the architect's drawings from page one. They walk in with **a small number of questions they have learned always find problems**, and they ask them in an order that eliminates whole categories at a time.

*"Where's the fire exit?"* — because if there isn't one, nothing else matters.
*"What happens when it rains hard?"* — because that's the failure everyone assumes won't happen.
*"Who checked the wiring?"* — because the answer is often "nobody, we assumed."

★ **They are not smarter than the architect. They have simply seen the same six mistakes enough times to ask about them first** — and they know the difference between a mistake that costs a day and one that costs a year.

★ **And the most valuable thing they do is not finding faults. It is asking a question the architect cannot answer** — because that reveals a decision nobody made.

---

## Where this fits in the big picture

```
   77 — ★ design it · 78 — ★ scale it
   Phases 1–8 — ★ every mechanism, decision and failure mode
                          │
                          ▼
        ┌──────────────────────────────────────────────┐
        │ 79 THE REVIEW ← YOU ARE HERE                 │
        │ ★ find it before it happens                  │
        └────────────────────┬─────────────────────────┘
                             ▼
                    ★ CURRICULUM COMPLETE
```

★ **This is the last topic because it is the compression of all the others.** Every check here exists because an earlier topic showed what happens without it. **The checklist is the curriculum, folded into questions.**

---

## What is this?

A **question-ordered review** that finds design and operational problems before production does.

```
 ★ FOUR PASSES, ★ IN THIS ORDER, ★ EACH ELIMINATING A CATEGORY:

 ★ PASS 1 — THE FIVE-MINUTE SCAN  (★ irreversible mistakes)
   ⇒ ★ things that are unrecoverable or cost quarters to undo
 ★ PASS 2 — CORRECTNESS           (★ what can be silently wrong)
   ⇒ ★ invariants, isolation, idempotency, drift
 ★ PASS 3 — SCALE                 (★ what breaks at 100×)
   ⇒ ★ hot rows, unbounded growth, missing indexes, hold time
 ★ PASS 4 — OPERATIONS            (★ what happens at 3 a.m.)
   ⇒ ★ backups, failover, observability, security

 ⇒ ★ THE ORDERING IS THE METHOD. ★ A correctness bug in a system
   that cannot be restored is not the most urgent finding.
```

★ **And the framing that makes a review useful rather than annoying:** *your job is not to list everything wrong. It is to **identify the three findings that matter most**, state each as a **specific failure scenario**, and make the trade-off visible — so the author can decide.*

---

## Why does it matter for a backend developer?

```
 ★ BECAUSE THE COST OF FINDING A PROBLEM RISES BY ~10× PER STAGE.

   ★ in review          ★ 1×      — a conversation
   ★ in implementation  ★ 10×     — a rewrite of one component
   ★ in staging         ★ 100×    — a redesign
   ★ in production      ★ 1,000×  — a migration, with data
   ★ in an incident     ★ 10,000× — a migration, at 3 a.m.,
                                    ★ with customers watching

 ⇒ ★ AND FOR ONE-WAY DOORS (77) THE MULTIPLIER IS WORSE:
   ★ `float` for money found in review        ⇒ ★ one line
   ★ found in production after 4M rows        ⇒ ★ unrecoverable
     (★ the precision is already lost)

 ★ AND THE SECOND REASON, WHICH MATTERS MORE:
   ★ A GOOD REVIEW TEACHES. ★ A finding explained as a failure
   scenario — ★ "here is the exact sequence where this loses
   money" — ★ changes what the author designs next time.
   ★ A finding delivered as a rule does not.
```

---

## The physical reality

### Pass 1 — the five-minute scan

```
 ★ EIGHT CHECKS. ★ ALL ARE IRREVERSIBLE OR COST QUARTERS.
 ★ IF ANY FAILS, ★ STOP AND RAISE IT BEFORE READING FURTHER.

 ★ ① MONEY
    grep for: ★ float · real · double precision · numeric without
    a scale · `money`
    ⇒ ★ FAILURE: ★ ₹0.01 errors compounding; ★ a ledger that does
      not balance; ★ UNRECOVERABLE once written (26).
    ⇒ ★ MUST BE: ★ bigint in minor units.

 ★ ② TIME
    grep for: ★ `timestamp` without `tz` · `now()` vs
    `current_timestamp` confusion · date arithmetic in the app
    ⇒ ★ FAILURE: ★ every stored value is ambiguous the moment a
      second region or a DST boundary appears (26).
    ⇒ ★ MUST BE: ★ timestamptz, always.

 ★ ③ TENANCY
    ⇒ ★ is there a tenant/org column on every tenant-scoped table?
    ⇒ ★ is RLS enabled ★ AND FORCED?
    ⇒ ★ FAILURE: ★ retrofitting touches every table, query and
      index (69). ★ And until then, one forgotten WHERE clause is
      a breach.

 ★ ④ PRIMARY KEY TYPE
    ⇒ ★ UUIDv4 as a PK? ⇒ ★ random insert points destroy B-tree
      locality and cache hit rates (22).
    ⇒ ★ bigserial on anything that will shard? (60, 71)

 ★ ⑤ RETENTION AND PARTITIONING
    ⇒ ★ does any table grow without bound?
    ⇒ ★ if it has a retention policy, ★ is it partitioned?
    ⇒ ★ FAILURE: ★ a `DELETE`-based retention job that becomes the
      largest workload in the system (47, 59, 73).

 ★ ⑥ IDEMPOTENCY
    ⇒ ★ does every externally-triggered write path have a key,
      ★ enforced by UNIQUE?
    ⇒ ★ FAILURE: ★ duplicate charges. ★ Retrofitting means auditing
      every handler (52).

 ★ ⑦ TRANSACTION CONTENTS
    grep inside every `BEGIN…COMMIT` for: ★ `await fetch` ·
    `axios` · `httpClient` · `sleep` · `retry`
    ⇒ ★ FAILURE: ★ hold time equals a third party's p99 ⇒ ★ no
      pool size survives it (45, 65, 78).

 ★ ⑧ BACKUPS
    ⇒ ★ "when did a restore last succeed, and how long did it take?"
    ⇒ ★ FAILURE: ★ if the answer is "never" and "unknown", ★ there
      is no backup — ★ there is a hope (64).

 ⇒ ★ THESE EIGHT TAKE FIVE MINUTES AND FIND MOST OF THE
   EXPENSIVE PROBLEMS.
```

### Pass 2 — correctness

```
 ★ THE QUESTION BEHIND EVERY CHECK: ★ "what can be silently
   wrong, and how would we know?"

 ★ ① EVERY INVARIANT HAS A MECHANISM
    ⇒ ★ ask: ★ "what enforces this when a second service writes?"
    ⇒ ★ "the application ensures it" ⇒ ★ NOT A MECHANISM (69, 77)
    ⇒ ★ classify: CHECK · UNIQUE · EXCLUDE · FK · ★ SERIALIZABLE

 ★ ② READ-MODIFY-WRITE
    grep for a `SELECT` followed by an `UPDATE` of the same row
    ⇒ ★ LOST UPDATE at READ COMMITTED (43). ★ Silent.
    ⇒ ★ FIX: ★ one atomic statement · `FOR UPDATE` · a version
      column (49)

 ★ ③ COUNT-THEN-INSERT
    grep for `SELECT count(*)` followed by an `INSERT`
    ⇒ ★ WRITE SKEW (43). ★ REPEATABLE READ does NOT help.
    ⇒ ★ FIX: ★ a constraint that materialises the conflict, ★ or
      SERIALIZABLE + retry (50)

 ★ ④ RETRYABLE ERRORS HANDLED
    ⇒ ★ is `40001` (serialization) and `40P01` (deadlock) retried?
    ⇒ ★ with ★ full jitter and a bound?
    ⇒ ★ AND: ★ is the retried unit IDEMPOTENT? (52)

 ★ ⑤ LOCK ORDERING
    ⇒ ★ does any transaction touch multiple rows without a
      deterministic order?
    ⇒ ★ `WHERE id = ANY($1)` locks in ★ PLAN order, ★ not array
      order (48). ★ An `ANALYZE` can flip it.

 ★ ⑥ DENORMALISED DATA HAS A RECONCILER
    ⇒ ★ for every copied column or maintained aggregate:
      ★ what keeps it in sync, ★ and what detects drift? (53, 54)
    ⇒ ★ and is it a SNAPSHOT (no obligation) or a COPY (forever)?

 ★ ⑦ DERIVED STORES HAVE A REBUILD
    ⇒ ★ "how do you rebuild the search index, and how long does
      it take?" (76)
    ⇒ ★ "unknown" ⇒ ★ there is no rebuild.

 ★ ⑧ CONSISTENCY IS WRITTEN DOWN
    ⇒ ★ which reads may be stale, ★ by how much, ★ measured at
      the p999? (68)
    ⇒ ★ "eventually consistent" without a bound and a reconciler
      ⇒ ★ inconsistency with extra steps (51)
```

### Pass 3 — scale

```
 ★ THE QUESTION: ★ "what breaks at 100× — and which layer is it?"
   (78)

 ★ ① EVERY ACCESS PATTERN HAS AN INDEX
    ⇒ ★ THE COMPLETENESS TEST (77): ★ "point at the index for
      each of these queries."
    ⇒ ★ silence ⇒ ★ a sequential scan nobody has noticed yet.

 ★ ② EVERY INDEX HAS AN ACCESS PATTERN
    ⇒ ★ `idx_scan = 0` after a month ⇒ ★ it costs writes and
      buys nothing.
    ⇒ ★ AND: ★ is it on a column that gets UPDATED? ⇒ ★ it kills
      HOT updates (46) — ★ the single largest WAL multiplier
      found in Topic 78, twice.

 ★ ③ THE HOTTEST KEY
    ⇒ ★ "what is the busiest single row, in writes per second?"
    ⇒ ★ > ~800/sec ⇒ ★ a hot row (61).
    ⇒ ★ AND: ★ was it measured with a REALISTIC key distribution?
      ★ A uniform load test never finds it (78).

 ★ ④ UNBOUNDED GROWTH
    ⇒ ★ any array, jsonb collection, partition or table without a
      stated bound (70, 72, 73)
    ⇒ ★ "how many rows will the largest partition hold in three
      years?"

 ★ ⑤ TRANSACTION HOLD TIME
    ⇒ ★ "how long is the longest transaction, and what is in it?"
    ⇒ ★ Little's Law: ★ pool = arrival_rate × hold_time (65)

 ★ ⑥ N+1
    ⇒ ★ queries per request. ★ > 25 ⇒ investigate (66).
    ⇒ ★ AND: ★ does the count grow with page size? ⇒ ★ that is
      the definitive test.

 ★ ⑦ AGGREGATES
    ⇒ ★ any query aggregating > 10⁶ rows on a user-facing path?
    ⇒ ★ ⇒ a rollup (56), ★ and it must store ★ (sum, count),
      ★ never an average (73).

 ★ ⑧ CONNECTION MATH
    ⇒ ★ instances × pool size vs `max_connections` vs the
      pooler's `default_pool_size` ★ per (db, user) (65)
    ⇒ ★ and reserved slots: superuser, autovacuum, walsenders,
      ★ parallel workers.
```

### Pass 4 — operations

```
 ★ THE QUESTION: ★ "what happens at 3 a.m., and who is woken?"

 ★ ① RESTORE
    ⇒ ★ "when did a restore last succeed?" ★ "how long did it
      take?" ★ "what compression and download concurrency?" (64)
    ⇒ ★ the three answers are usually "never", "unknown", "gzip,
      default" — ★ which is a multi-hour RTO nobody has budgeted.

 ★ ② REPLICATION SLOTS
    ⇒ ★ `SELECT * FROM pg_replication_slots WHERE NOT active`
    ⇒ ★ THE SAME OBJECT CAUSED INCIDENTS IN TOPICS 41, 47, 62,
      76 AND 78. ★ Five times. ★ Check it every single review.
    ⇒ ★ AND: ★ is `max_slot_wal_keep_size` set?

 ★ ③ FAILOVER
    ⇒ ★ "what fences the old primary?"
    ⇒ ★ "when was failover last tested — ★ including a FROZEN
      node, not just a kill?" (63)
    ⇒ ★ "what is `maximum_lag_on_failover`, ★ and where did that
      number come from?"

 ★ ④ CLIENT TIMEOUTS
    ⇒ ★ `tcp_user_timeout`? ★ Default is ~15 minutes.
    ⇒ ★ FAILURE: ★ a 30-second database failover becomes a
      15-minute application outage (63).

 ★ ⑤ VACUUM AND WRAPAROUND
    ⇒ ★ `age(datfrozenxid)` · ★ dead-tuple ratio · ★ per-table
      autovacuum settings on high-churn tables (47)
    ⇒ ★ "is anything pinning xmin?" ★ long txn / slot / 2PC.

 ★ ⑥ OBSERVABILITY
    ⇒ ★ `pg_stat_statements`? ★ `auto_explain`? ★ `log_lock_waits`?
      ★ `track_io_timing`? (67)
    ⇒ ★ the three per-request numbers: ★ db_ms, ★ pool_wait_ms,
      ★ query_count
    ⇒ ★ AND: ★ is log volume sampled? ★ At 100× it becomes the
      bottleneck (78).

 ★ ⑦ SECURITY
    ⇒ ★ does the application role OWN its tables? ⇒ ★ an injection
      becomes `DROP TABLE` (69)
    ⇒ ★ `sslmode`? ★ `require` encrypts but does not authenticate.
    ⇒ ★ dynamic identifiers (`ORDER BY ${x}`) ⇒ ★ an allowlist?

 ★ ⑧ MIGRATIONS
    ⇒ ★ `lock_timeout` on every DDL? (45)
    ⇒ ★ `idle_in_transaction_session_timeout` set? ★ One setting
      prevents the classic FIFO-queue outage.
```

### How to deliver a finding

```
 ★ A FINDING IS NOT "THIS IS WRONG". ★ IT IS FOUR PARTS:

 ★ ① THE SPECIFIC FAILURE SCENARIO
    ✗ "you should use bigint for money"
    ✓ ★ "an order of ₹1,234.55 stored as a float and summed
      across 10,000 line items drifts by ₹0.03. ★ The ledger
      will not balance, ★ and the drift is already in the data —
      it cannot be recovered."

 ★ ② THE COST OF FIXING IT ★ NOW vs ★ LATER
    ✓ ★ "one line today; ★ unrecoverable after the first million
      rows."

 ★ ③ THE SEVERITY, ★ HONESTLY
    ★ BLOCKING   ⇒ data loss, security, or an irreversible
                   one-way door
    ★ HIGH       ⇒ a correctness bug, or a scale cliff you will
                   hit this year
    ★ MEDIUM     ⇒ an operational gap
    ★ NIT        ⇒ ★ say it is a nit, ★ and mean it

 ★ ④ ★ A QUESTION, NOT A DIRECTIVE, WHERE THE ANSWER MIGHT
    CHANGE YOUR MIND
    ✓ ★ "what is the busiest single seller, in orders per second?"
    ⇒ ★ if the answer is 0.14/sec, ★ your hot-row concern is
      wrong ★ and you have learned something.

 ⇒ ★ AND THE MOST VALUABLE FINDING IS A QUESTION THE AUTHOR
   CANNOT ANSWER — ★ because it reveals a decision nobody made.
```

---

## How it works — step by step

### The review, run as a script

```markdown
## Database review — <system>   <date>   Reviewer: <name>

★ PASS 1 — FIVE-MINUTE SCAN
  □ money is `bigint` minor units                       (26)
  □ all timestamps are `timestamptz`                    (26)
  □ tenancy column present + ★ RLS FORCED               (69)
  □ PK type appropriate (★ no UUIDv4 as PK)             (22)
  □ every unbounded table is partitioned                (59)
  □ idempotency keys on external write paths            (52)
  □ ★ no network calls inside transactions              (45,65)
  □ ★ a restore has succeeded, and it was timed         (64)

★ PASS 2 — CORRECTNESS
  □ each invariant → a named mechanism                  (24,77)
  □ no read-modify-write across statements              (43,49)
  □ no count-then-insert without a constraint           (43,50)
  □ 40001 / 40P01 retried, ★ with jitter, ★ idempotent  (44,48,52)
  □ deterministic lock ordering                         (48)
  □ every denormalisation has a reconciler              (53,54)
  □ ★ every derived store has a TIMED rebuild           (76)
  □ ★ the consistency contract is written and signed    (68)

★ PASS 3 — SCALE
  □ ★ every access pattern → a named index              (77)
  □ every index → an access pattern (★ `idx_scan`)      (78)
  □ ★ hottest key measured with a REALISTIC distribution (61,78)
  □ no unbounded arrays / partitions / tables       (70,72,73)
  □ ★ longest transaction hold time known              (65)
  □ queries-per-request < 25, ★ constant with page size (66)
  □ user-facing aggregates use rollups (★ sum+count)    (56,73)
  □ connection arithmetic closes                        (65)

★ PASS 4 — OPERATIONS
  □ ★ restore tested, timed, on a dashboard             (64)
  □ ★ NO INACTIVE REPLICATION SLOTS + guard set     (41,47,62,76)
  □ ★ failover tested, ★ including a FROZEN node        (63)
  □ ★ `tcp_user_timeout` set on clients                 (63)
  □ XID age, dead tuples, xmin holders checked          (47)
  □ pg_stat_statements + auto_explain + track_io_timing (67)
  □ ★ log volume sampled                                (78)
  □ ★ app role owns nothing; `sslmode=verify-full`      (69)
  □ ★ `lock_timeout` + `idle_in_transaction_session_timeout` (45)

★ TOP THREE FINDINGS
  1. …
  2. …
  3. …

★ QUESTIONS THE AUTHOR COULD NOT ANSWER
  - …
```

### The queries that answer half the checklist

```sql
-- ★ ① MONEY AND TIME — Pass 1
SELECT table_name, column_name, data_type FROM information_schema.columns
 WHERE table_schema = 'public'
   AND (★ data_type IN ('real','double precision','money')
     OR ★ (data_type = 'timestamp without time zone'))
 ORDER BY 1,2;
-- ★ any row is a Pass-1 finding.
```
```sql
-- ★ ② TENANCY AND RLS — Pass 1/4
SELECT c.relname, c.relrowsecurity AS rls, c.relforcerowsecurity AS forced
  FROM pg_class c JOIN pg_namespace n ON n.oid=c.relnamespace
 WHERE n.nspname='public' AND c.relkind='r'
   AND EXISTS (SELECT 1 FROM information_schema.columns
                WHERE table_name=c.relname AND column_name IN ('tenant_id','org_id'))
   AND ★ NOT (c.relrowsecurity AND c.relforcerowsecurity);
-- ★ any row: RLS is missing or not forced (69).
```
```sql
-- ★ ③ INDEXES: unused, and on updated columns — Pass 3
SELECT s.relname, s.indexrelname, s.idx_scan,
       pg_size_pretty(pg_relation_size(s.indexrelid)) AS size,
       pg_get_indexdef(s.indexrelid) AS def
  FROM pg_stat_user_indexes s
  JOIN pg_index i ON i.indexrelid = s.indexrelid
 WHERE ★ s.idx_scan < 50 AND ★ NOT i.indisprimary AND NOT i.indisunique
   AND pg_relation_size(s.indexrelid) > 50*1024*1024
 ORDER BY pg_relation_size(s.indexrelid) DESC;
```
```sql
-- ★ ④ HOT UPDATE RATIO — ★ the WAL multiplier (46, 78)
SELECT relname, n_tup_upd, n_tup_hot_upd,
       round(100.0*n_tup_hot_upd/nullif(n_tup_upd,0),1) AS hot_pct
  FROM pg_stat_user_tables WHERE n_tup_upd > 100000
 ORDER BY hot_pct NULLS LAST LIMIT 10;
-- ★ hot_pct < 50% on a hot table ⇒ ★ an index on an updated column.
```
```sql
-- ★ ⑤ THE SLOT CHECK — ★ five incidents in this curriculum
SELECT slot_name, slot_type, ★ active, ★ wal_status,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn))
         AS retained,
       age(catalog_xmin) AS catalog_xmin_age
  FROM pg_replication_slots;
SHOW ★ max_slot_wal_keep_size;
```
```sql
-- ★ ⑥ VACUUM AND WRAPAROUND — Pass 4
SELECT datname, age(datfrozenxid) AS xid_age FROM pg_database
 ORDER BY 2 DESC LIMIT 3;
SELECT relname, n_dead_tup, last_autovacuum FROM pg_stat_user_tables
 WHERE n_dead_tup > 1000000 ORDER BY n_dead_tup DESC LIMIT 5;
-- ★ and the three xmin holders:
SELECT 'txn' AS s, pid::text, (now()-xact_start)::text FROM pg_stat_activity
 WHERE backend_xmin IS NOT NULL
UNION ALL SELECT 'slot', slot_name, active::text FROM pg_replication_slots
UNION ALL SELECT '2pc', gid, (now()-prepared)::text FROM pg_prepared_xacts;
```
```sql
-- ★ ⑦ N+1 AND CAPACITY CONSUMERS — Pass 3
SELECT calls, mean_exec_time::numeric(10,4) AS ms,
       (calls*mean_exec_time/1000)::numeric(12,1) AS total_s,
       substring(query,1,60) FROM pg_stat_statements
 ORDER BY ★ calls DESC LIMIT 5;
-- ★ and by variance, for the p99:
SELECT calls, mean_exec_time::numeric(10,2) AS mean_ms,
       stddev_exec_time::numeric(10,2) AS sd,
       (stddev_exec_time/nullif(mean_exec_time,0))::numeric(6,1) AS cv,
       substring(query,1,50) FROM pg_stat_statements
 WHERE calls > 100 ORDER BY ★ stddev_exec_time DESC LIMIT 5;
```
```sql
-- ★ ⑧ SECURITY — Pass 4
SELECT rolname, rolsuper, rolcreaterole, rolbypassrls, rolcanlogin
  FROM pg_roles WHERE rolsuper OR rolcreaterole OR rolbypassrls;
SELECT * FROM information_schema.role_table_grants WHERE grantee='PUBLIC';
SELECT a.usename, s.ssl FROM pg_stat_activity a
  LEFT JOIN pg_stat_ssl s USING (pid) WHERE a.backend_type='client backend';
```
```sql
-- ★ ⑨ CONFIGURATION — Pass 4
SELECT name, setting FROM pg_settings WHERE name IN (
  'max_connections','shared_buffers','work_mem',
  ★ 'idle_in_transaction_session_timeout', ★ 'statement_timeout',
  ★ 'log_lock_waits', ★ 'track_io_timing', ★ 'max_slot_wal_keep_size',
  ★ 'wal_compression', 'data_checksums', 'password_encryption');
```

### The code greps that answer the other half

```bash
# ★ ① NETWORK CALLS INSIDE TRANSACTIONS — Pass 1  (★ highest value)
rg -U --multiline 'BEGIN[\s\S]{0,2000}?(fetch|axios|httpClient|\.post\(|sleep)[\s\S]{0,2000}?COMMIT' src/
rg -U --multiline 'withTransaction\([\s\S]{0,1500}?(await \w+Service\.|fetch\()' src/

# ★ ② READ-MODIFY-WRITE — Pass 2
rg -U --multiline 'SELECT[\s\S]{0,600}?UPDATE\s+(\w+)\s+SET' src/ | rg -v 'FOR UPDATE'

# ★ ③ COUNT-THEN-INSERT (★ write skew) — Pass 2
rg -U --multiline 'count\(\*\)[\s\S]{0,800}?INSERT INTO' src/

# ★ ④ RETRYABLE ERRORS — Pass 2
rg "'40001'|'40P01'|serialization_failure|deadlock_detected" src/
# ★ if this returns nothing and any code path uses REPEATABLE READ
#   or SERIALIZABLE ⇒ ★ unhandled retryable errors.

# ★ ⑤ IDEMPOTENCY — Pass 1
rg -i 'idempotency|idempotent' src/ | head
rg 'app\.(post|put)\(' src/ | wc -l   # ★ compare the two counts

# ★ ⑥ DYNAMIC IDENTIFIERS (★ injection) — Pass 4
rg 'ORDER BY \$\{|FROM \$\{|`\s*SELECT[\s\S]*?\$\{' src/

# ★ ⑦ SESSION STATE BEHIND A POOLER — Pass 4
rg 'SET (search_path|role|work_mem)' src/ | rg -v 'SET LOCAL'
rg 'pg_advisory_lock\(' src/            # ★ should be _xact_

# ★ ⑧ CLIENT TIMEOUTS — Pass 4
rg 'tcp_user_timeout|keepAlives|connectionTimeoutMillis' src/
# ★ absent ⇒ ★ a dead host hangs for ~15 minutes (63)

# ★ ⑨ ARRAY LOCKING ORDER — Pass 2
rg 'WHERE id = ANY\(' src/    # ★ locks in PLAN order, not array order (48)
```

---

## Concept breakdown

```
★ FOUR PASSES, ★ IN ORDER
   ① ★ FIVE-MINUTE SCAN  — ★ irreversible mistakes
   ② CORRECTNESS         — ★ what can be silently wrong
   ③ SCALE               — ★ what breaks at 100×
   ④ OPERATIONS          — ★ what happens at 3 a.m.
   ⇒ ★ a correctness bug in a system that cannot be restored is
     not the most urgent finding

★ PASS 1 — ★ EIGHT CHECKS, FIVE MINUTES, MOST OF THE VALUE
   ★ money as bigint · ★ timestamptz · ★ tenancy + FORCED RLS ·
   ★ PK type · ★ partitioning for retention · ★ idempotency keys ·
   ★ no network calls in transactions · ★ a TIMED restore

★ THE COST CURVE
   review 1× · implementation 10× · staging 100× ·
   ★ production 1,000× · ★ incident 10,000×
   ⇒ ★ and for one-way doors, ★ later can mean UNRECOVERABLE

★ THE HIGHEST-VALUE QUESTIONS, IN ORDER
   ★ ① "point at the index for this access pattern"
   ★ ② "what enforces this when a second service writes?"
   ★ ③ "how long is the transaction, and what is inside it?"
   ★ ④ "when did a restore last succeed, and how long did it take?"
   ★ ⑤ "what is the busiest single row, per second?"
   ★ ⑥ "how do you rebuild the derived store, and how long?"

★ THE OBJECT TO CHECK EVERY SINGLE TIME
   ★ pg_replication_slots WHERE NOT active
   ⇒ ★ five separate incidents in this curriculum (41,47,62,76,78)

★ HOW TO DELIVER A FINDING — ★ four parts
   ★ ① a SPECIFIC failure scenario, not a rule
   ★ ② the cost NOW vs LATER
   ★ ③ honest severity (★ blocking / high / medium / ★ nit)
   ★ ④ ★ a QUESTION where the answer might change your mind

★ ★ THE MOST VALUABLE FINDING IS A QUESTION THE AUTHOR CANNOT
  ANSWER — ★ it reveals a decision nobody made.
```

---

## Diagrams

**Diagram 1 — big picture: the four passes and what each eliminates**

```
                       ★ A DESIGN OR SYSTEM ARRIVES
                                   │
  ┌────────────────────────────────▼──────────────────────────────┐
  │ ★ PASS 1 — FIVE-MINUTE SCAN                                    │
  │   money · time · tenancy · PK type · partitioning ·            │
  │   idempotency · ★ network-in-transaction · ★ restore           │
  │                                                                │
  │   ★ ELIMINATES: ★ the irreversible and the unrecoverable       │
  │   ★ IF ANY FAILS: ★ STOP AND RAISE IT NOW.                     │
  │     ★ Everything below is less urgent.                         │
  └────────────────────────────────┬──────────────────────────────┘
                                   ▼
  ┌───────────────────────────────────────────────────────────────┐
  │ ★ PASS 2 — CORRECTNESS                                         │
  │   invariants→mechanisms · read-modify-write · write skew ·     │
  │   retryable errors · lock order · reconcilers · rebuilds ·     │
  │   the consistency contract                                     │
  │                                                                │
  │   ★ ELIMINATES: ★ what can be SILENTLY wrong                   │
  │   ★ THE TEST: ★ "how would we know if this were wrong?"        │
  └────────────────────────────────┬──────────────────────────────┘
                                   ▼
  ┌───────────────────────────────────────────────────────────────┐
  │ ★ PASS 3 — SCALE                                               │
  │   ★ every pattern→an index · every index→a pattern ·           │
  │   ★ the hottest key · unbounded growth · ★ hold time ·         │
  │   N+1 · aggregates · connection math                           │
  │                                                                │
  │   ★ ELIMINATES: ★ the cliff you hit at 10× or 100×             │
  │   ★ THE TEST: ★ Topic 78's seven layers — which one first?     │
  └────────────────────────────────┬──────────────────────────────┘
                                   ▼
  ┌───────────────────────────────────────────────────────────────┐
  │ ★ PASS 4 — OPERATIONS                                          │
  │   ★ restore · ★ SLOTS · failover · ★ client timeouts ·         │
  │   vacuum/wraparound · observability · security · migrations    │
  │                                                                │
  │   ★ ELIMINATES: ★ the 3 a.m. surprise                          │
  │   ★ THE TEST: ★ "who is woken, and what do they run?"          │
  └────────────────────────────────┬──────────────────────────────┘
                                   ▼
              ★ THREE FINDINGS + ★ THE UNANSWERED QUESTIONS
```

**Diagram 2 — data flow: the cost of finding a problem, by stage**

```
  ★ COST OF FIXING, BY WHEN IT IS FOUND

  ★ 10,000× ┤                                          ● ★ incident
            │                                        ╱   (3 a.m., with
   1,000×   ┤                                  ●   ╱     customers)
            │                              ╱         ★ production
     100×   ┤                        ●   ╱             (a data migration)
            │                    ╱         ★ staging
      10×   ┤              ●   ╱             (a redesign)
            │          ╱         ★ implementation
       1×   ┤ ★ ●  ╱               (one component)
            │  review
            └──┬─────┬─────┬─────┬─────┬────────────────
             review impl  staging prod  incident

  ★ AND FOR ONE-WAY DOORS (77), THE CURVE HAS NO TOP:
  ┌───────────────────────────────────────────────────────────────┐
  │ ★ `float` for money                                            │
  │   in review     ⇒ ★ one line                                   │
  │   after 4M rows ⇒ ★ UNRECOVERABLE — ★ the precision is         │
  │                    already gone from the stored values         │
  │                                                                │
  │ ★ tenancy added later                                          │
  │   in review     ⇒ ★ one column                                 │
  │   in production ⇒ ★ every table, query, index (69) ⇒ months   │
  │                                                                │
  │ ★ a partition key                                              │
  │   in review     ⇒ ★ a conversation                             │
  │   in production ⇒ ★ dual-write + backfill + shadow reads       │
  │                    (72, 62) ⇒ ★ quarters                       │
  └───────────────────────────────────────────────────────────────┘
   ★ THIS IS WHY PASS 1 IS FIRST AND TAKES FIVE MINUTES.
```

**Diagram 3 — before/after: two reviews of the same PR**

```
 ✗ AN UNHELPFUL REVIEW — ★ 23 comments, all equally weighted
 ┌───────────────────────────────────────────────────────────────┐
 │ • "use snake_case for this column"                             │
 │ • "should this be NOT NULL?"                                   │
 │ • "consider an index here"                                     │
 │ • "money should be bigint"                          ← ★ buried │
 │ • "typo in the comment"                                        │
 │ • "prefer `text` over `varchar(255)`"                          │
 │ • "this transaction calls the payment gateway"      ← ★ buried │
 │ • … 16 more                                                    │
 │                                                                │
 │ ★ RESULT: ★ the author fixes the typos, ★ argues about         │
 │   varchar, ★ and ships the two that matter.                    │
 │ ★ THE REVIEWER WAS RIGHT ABOUT EVERYTHING AND ACHIEVED         │
 │   NOTHING.                                                     │
 └───────────────────────────────────────────────────────────────┘

 ✓ A USEFUL REVIEW — ★ three findings, ★ ranked, ★ with scenarios
 ┌───────────────────────────────────────────────────────────────┐
 │ ★ BLOCKING — `total_amount double precision`                   │
 │   ★ SCENARIO: ★ an order of ₹1,234.55 × 10,000 line items      │
 │     drifts ₹0.03. ★ The ledger will not reconcile, ★ and the   │
 │     drift is unrecoverable once written.                       │
 │   ★ COST: ★ one line now. ★ Unrecoverable after go-live.       │
 │   ★ FIX: ★ `total_minor bigint`.                               │
 │                                                                │
 │ ★ BLOCKING — the payment-gateway call is inside the txn        │
 │   ★ SCENARIO: ★ gateway p99 is 4.1 s ⇒ ★ a 4.1-second row lock │
 │     on `stock` ⇒ ★ Little's Law: 30 orders/s × 4.1 s = ★ 123   │
 │     connections held, ★ and every buyer of that SKU blocks.    │
 │   ★ COST: ★ redraw the boundary now (2 h). ★ A production      │
 │     incident later.                                            │
 │   ★ FIX: ★ write, call outside, write again + a sweeper (52).  │
 │                                                                │
 │ ★ HIGH — no index for "orders by tracking number"              │
 │   ★ QUESTION: ★ "this is 40,000/min in the spec — ★ point at   │
 │     the index?"                                                │
 │   ★ (★ the author's answer revealed it was in a jsonb field    │
 │     with a GIN index ⇒ ★ 4.2 ms vs 0.08 ms.)                   │
 │                                                                │
 │ ★ NITS (★ non-blocking, ★ take or leave): naming, varchar,     │
 │   the typo.                                                    │
 │                                                                │
 │ ★ QUESTIONS I COULD NOT ANSWER FROM THE PR:                    │
 │   ★ • what is the busiest single seller, in orders/sec?        │
 │   ★ • how long does a restore take?                            │
 │   ★ • what fences the old primary on failover?                 │
 └───────────────────────────────────────────────────────────────┘
   ★ THE SECOND REVIEW HAS FEWER COMMENTS AND CHANGES THE
     OUTCOME. ★ And the three unanswered questions were worth
     more than all 23 comments in the first.
```

---

## Example 1 — basic

**Run the five-minute scan against a real database.**

```sql
-- ★ ① money and time
SELECT table_name, column_name, data_type FROM information_schema.columns
 WHERE table_schema='public'
   AND (data_type IN ('real','double precision','money')
     OR data_type = 'timestamp without time zone');
```
```
   table_name   |  column_name   |        data_type
----------------+----------------+--------------------------
 ★ payments     | ★ amount       | ★ double precision
 ★ orders       | ★ created_at   | ★ timestamp without time zone
 ★ TWO PASS-1 FINDINGS. ★ Both are one-way doors.
```

```sql
-- ★ ② tenancy and RLS
SELECT c.relname, c.relrowsecurity, c.relforcerowsecurity
  FROM pg_class c JOIN pg_namespace n ON n.oid=c.relnamespace
 WHERE n.nspname='public' AND c.relkind='r'
   AND EXISTS (SELECT 1 FROM information_schema.columns
                WHERE table_name=c.relname AND column_name='org_id')
   AND NOT (c.relrowsecurity AND c.relforcerowsecurity);
```
```
  relname   | relrowsecurity | relforcerowsecurity
------------+----------------+---------------------
 ★ invoices | ★ t            | ★ f
 ⇒ ★ RLS IS ENABLED BUT NOT FORCED. ★ If the app owns the table,
   ★ RLS DOES NOTHING (69). ★ This looks configured and isn't.
```

```sql
-- ★ ③ unbounded tables
SELECT relname, pg_size_pretty(pg_total_relation_size(relid)) AS size,
       n_live_tup,
       ★ (SELECT count(*) FROM pg_inherits WHERE inhparent=relid) AS partitions
  FROM pg_stat_user_tables ORDER BY pg_total_relation_size(relid) DESC LIMIT 5;
```
```
    relname     |  size   | n_live_tup | partitions
----------------+---------+------------+------------
 ★ scan_events  | ★ 1.2 TB| ★ 4102884201|        ★ 0
 ⇒ ★ 4.1 BILLION ROWS, ★ UNPARTITIONED. ★ Ask: ★ "what is the
   retention policy, and how is it enforced?"
```

```sql
-- ★ ④ the slot check — ★ every review, every time
SELECT slot_name, active, wal_status,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained
  FROM pg_replication_slots;
SHOW max_slot_wal_keep_size;
```
```
   slot_name   | active | wal_status | retained
---------------+--------+------------+----------
 ★ debezium_v1 | ★ f    | reserved   | ★ 812 GB
 ★ max_slot_wal_keep_size: ★ -1
 ⇒ ★ AN INACTIVE SLOT RETAINING 812 GB, ★ WITH NO GUARD.
   ★ This is a latent primary-database outage with no timer.
   ★ BLOCKING.
```

```bash
# ★ ⑤ network calls inside transactions — ★ the highest-value grep
rg -U --multiline 'withTransaction\([\s\S]{0,1500}?(await \w+Service\.|fetch\()' src/ -l
```
```
 ★ src/checkout/place-order.js
 ★ src/payments/capture.js
 ⇒ ★ TWO HANDLERS. ★ Ask for the p99 of each external call.
```

```bash
# ★ ⑥ idempotency coverage
echo "external POST handlers: $(rg 'app\.post\(' src/ | wc -l)"
echo "with idempotency:       $(rg -l 'idempotency' src/routes/ | wc -l)"
```
```
 ★ external POST handlers: 18
 ★ with idempotency:       ★ 3
 ⇒ ★ 15 HANDLERS THAT CANNOT SAFELY BE RETRIED (52).
```

```sql
-- ★ ⑦ the restore question — ★ there is no query for this
--    ★ ASK IT. "When did a restore last succeed, and how long
--      did it take?"
```
```
 ★ ANSWER RECEIVED: ★ "we have nightly backups to S3."
 ⇒ ★ THAT IS NOT AN ANSWER TO EITHER QUESTION.
 ⇒ ★ BLOCKING (64).
```

**The output of five minutes:**

```markdown
★ PASS 1 FINDINGS — <system>

★ BLOCKING
1. ★ `payments.amount` is `double precision`. ★ Unrecoverable
   once written (26).
2. ★ An inactive replication slot retaining 812 GB, with
   `max_slot_wal_keep_size = -1`. ★ A latent primary outage
   (41,47,62).
3. ★ No restore has been tested or timed (64).

★ HIGH
4. ★ `invoices` has RLS enabled but ★ NOT FORCED — ★ it does
   nothing if the app owns the table (69).
5. ★ `scan_events`: 4.1 billion rows, unpartitioned. ★ What is
   the retention policy? (59)
6. ★ 15 of 18 external write handlers have no idempotency
   key (52).
7. ★ `orders.created_at` is `timestamp` without a zone (26).

★ QUESTIONS
- ★ p99 of the external calls inside `place-order` and `capture`?
- ★ Busiest single seller, in orders/sec?
```

---

## Example 2 — production scenario

**The situation.** You have joined a company as a senior engineer. You are asked to review the database of the platform you will be working on — 3 years old, 40 engineers, ₹200 crore GMV.

```
 ★ YOU HAVE ONE DAY AND NO CONTEXT.
 ⇒ ★ THE CHECKLIST IS HOW YOU PRODUCE SOMETHING USEFUL ANYWAY.
```

**Hour 1 — Pass 1.**

```sql
-- ★ money and time: ★ clean. Someone got this right.
-- ★ tenancy:
SELECT count(*) FILTER (WHERE relrowsecurity AND relforcerowsecurity) AS forced,
       count(*) AS tenant_tables
  FROM pg_class c JOIN pg_namespace n ON n.oid=c.relnamespace
 WHERE n.nspname='public' AND c.relkind='r'
   AND EXISTS (SELECT 1 FROM information_schema.columns
                WHERE table_name=c.relname AND column_name='merchant_id');
```
```
 forced | tenant_tables
--------+---------------
    ★ 0 |          ★ 41
 ⇒ ★ 41 TENANT-SCOPED TABLES. ★ ZERO WITH RLS.
   ★ Isolation depends entirely on every query remembering a
   WHERE clause.
```
```bash
# ★ how many queries touch a tenant table without merchant_id?
rg -U --multiline 'FROM (orders|invoices|payouts|products)\b[\s\S]{0,400}?(WHERE|;)' src/ \
  | rg -v 'merchant_id' | wc -l
```
```
 ★ 23
 ⇒ ★ 23 QUERIES WITH NO TENANT FILTER. ★ Any one is a
   cross-merchant data exposure (69).
 ★ BLOCKING.
```
```sql
-- ★ the slot check
SELECT slot_name, active, pg_size_pretty(
  pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained,
  age(catalog_xmin) FROM pg_replication_slots;
```
```
   slot_name    | active | retained | age
----------------+--------+----------+--------
 analytics_cdc  | ★ f    | ★ 1.4 TB | ★ 812,884,201
 ⇒ ★ INACTIVE. ★ 1.4 TB retained. ★ catalog_xmin pinned for
   what looks like months.
```
```sql
SELECT datname, age(datfrozenxid) FROM pg_database ORDER BY 2 DESC LIMIT 1;
```
```
  datname  |    age
-----------+------------
 ★ platform| ★ 1,884,201,882
 ⇒ ★ 1.88 BILLION OF 2.1 BILLION.
 ⇒ ★ THE DATABASE IS ~260 MILLION TRANSACTIONS FROM REFUSING
   ALL WRITES (47).
 ★ THIS IS THE MOST URGENT FINDING OF THE DAY, ★ and it was
   found in minute 14.
```

**Hour 2 — the wraparound emergency.**

```
 ★ YOU STOP THE REVIEW. ★ This is not a finding for a document.
 ⇒ ★ ESCALATE IMMEDIATELY, ★ with the numbers and the procedure.
```
```sql
-- ★ ① what is pinning xmin?  ★ three candidates (47)
SELECT 'txn' AS src, pid::text, (now()-xact_start)::text AS age
  FROM pg_stat_activity WHERE backend_xmin IS NOT NULL
UNION ALL SELECT 'slot', slot_name, active::text FROM pg_replication_slots
UNION ALL SELECT '2pc', gid, (now()-prepared)::text FROM pg_prepared_xacts;
```
```
  src  |     pid       |  age
-------+---------------+-------
 ★ slot| analytics_cdc | ★ false
 ⇒ ★ ONE CAUSE. ★ A CDC connector decommissioned in April.
```
```sql
-- ★ ② release it, ★ then vacuum
SELECT pg_drop_replication_slot('analytics_cdc');
ALTER SYSTEM SET ★ max_slot_wal_keep_size = '256GB';
SELECT pg_reload_conf();
```
```bash
psql -c "SET vacuum_cost_delay=0; SET maintenance_work_mem='8GB';
         VACUUM (FREEZE, VERBOSE, PARALLEL 4) big_table;"
```
```
 ★ age(datfrozenxid) ★ 1,884,201,882 → ★ 41,204,882 (4 hours)
 ★ WAL retained ★ 1.4 TB → ★ 240 MB
 ★ disk ★ 91% → ★ 44%
 ⇒ ★ A MULTI-HOUR OUTAGE, ★ AVOIDED, ★ BY ONE QUERY IN A
   CHECKLIST.
```

**Hours 3–5 — Passes 2 and 3.**

```bash
# ★ correctness greps
rg -U --multiline 'SELECT[\s\S]{0,600}?UPDATE\s+\w+\s+SET' src/ \
  | rg -v 'FOR UPDATE' | wc -l
rg -U --multiline 'count\(\*\)[\s\S]{0,800}?INSERT INTO' src/ | wc -l
rg "'40001'|'40P01'" src/ | wc -l
```
```
 ★ read-modify-write: ★ 14
 ★ count-then-insert: ★ 6
 ★ retryable handled: ★ 0
 ⇒ ★ 14 POSSIBLE LOST UPDATES (43), ★ 6 POSSIBLE WRITE SKEWS (43),
   ★ AND NO RETRY HANDLING ANYWHERE.
```
```sql
-- ★ but first: ★ is anything actually running at an isolation
--   level where 40001 can occur?
SELECT setting FROM pg_settings WHERE name='default_transaction_isolation';
```
```
 ★ read committed
 ⇒ ★ 40001 is unlikely, ★ but 40P01 (deadlock) is not.
 ⇒ ★ AND the 14 read-modify-writes are lost updates ★ AT READ
   COMMITTED — ★ which is exactly where they are silent (43).
```
```sql
-- ★ scale: the completeness test needs the access patterns.
--   ★ There is no document. ★ So derive them:
SELECT calls, mean_exec_time::numeric(10,3) AS ms,
       (calls*mean_exec_time/1000/3600)::numeric(10,1) AS hours,
       substring(query,1,55) FROM pg_stat_statements
 ORDER BY calls*mean_exec_time DESC LIMIT 5;
```
```
   calls   |   ms    | hours |                  substring
-----------+---------+-------+---------------------------------------
 ★ 1884201188| ★ 0.021| ★ 11.0| SELECT * FROM merchants WHERE id = $1
   88420118 |  ★ 8.84|★ 217.1| SELECT … FROM orders o JOIN … WHERE
   41204882 |   0.412|   4.7 | UPDATE payouts SET …
 ⇒ ★ TWO FINDINGS:
   ① ★ 1.88 BILLION calls of a 0.02 ms query ⇒ ★ N+1 (66)
   ② ★ 217 HOURS/DAY on one query ⇒ ★ check its plan
```
```sql
EXPLAIN (ANALYZE, BUFFERS) /* the 8.84 ms query */;
```
```
 ->  ★ Seq Scan on orders  (actual rows=41,204)
       ★ Rows Removed by Filter: ★ 8,204,118
 ⇒ ★ LAYER 1 (78). ★ A missing index costing 217 hours/day.
```
```sql
-- ★ hot updates and unused indexes
SELECT relname, round(100.0*n_tup_hot_upd/nullif(n_tup_upd,0),1) AS hot_pct
  FROM pg_stat_user_tables WHERE n_tup_upd > 1000000 ORDER BY 2 LIMIT 3;
SELECT relname, indexrelname, idx_scan,
       pg_size_pretty(pg_relation_size(indexrelid)) AS size
  FROM pg_stat_user_indexes WHERE idx_scan = 0
   AND pg_relation_size(indexrelid) > 1024*1024*1024;
```
```
   relname  | hot_pct
------------+---------
 ★ payouts  |   ★ 0.8
 ★ orders   |   ★ 4.1

   relname |      indexrelname       | idx_scan |  size
-----------+-------------------------+----------+--------
 ★ payouts | idx_payouts_updated_at  |     ★ 0  | ★ 88 GB
 ★ orders  | idx_orders_metadata_gin |     ★ 0  | ★ 214 GB
 ⇒ ★ 302 GB OF INDEXES SCANNED ZERO TIMES, ★ ON THE EXACT
   COLUMNS THAT ARE UPDATED. ★ They are the reason hot_pct is 0.8%.
```

**Hours 6–7 — Pass 4.**

```
 ★ THE QUESTIONS, ASKED OF THE TEAM:

 ★ "When did a restore last succeed?"
   ⇒ ★ "There's a runbook."         ⇒ ★ never tested.
 ★ "How long would it take?"
   ⇒ ★ "A few hours?"                ⇒ ★ unknown. ★ 4.1 TB, gzip,
                                       default concurrency ⇒
                                       ★ estimated 19 hours (64).
 ★ "What fences the old primary on failover?"
   ⇒ ★ "Patroni handles it."         ⇒ ★ software fencing only.
                                       ★ A frozen node is
                                       unprotected (63).
 ★ "When was failover last tested?"
   ⇒ ★ "At setup, two years ago."    ⇒ ★ and only with a kill,
                                       ★ never a freeze.
 ★ "What is `tcp_user_timeout` on the clients?"
   ⇒ ★ (silence)                     ⇒ ★ default. ★ A 30-second
                                       failover is a 15-minute
                                       outage (63).
 ★ "Does the app role own its tables?"
   ⇒ ★ "It runs the migrations, so yes."
                                     ⇒ ★ an injection becomes
                                       DROP TABLE (69).
```

**Hour 8 — the report.**

```markdown
# Database review — platform   2026-08-27
Reviewer: <name>. ★ One day, no prior context.

## ★ ACTION TAKEN DURING THE REVIEW
★ Transaction-ID wraparound was 260M transactions from a total
write outage, caused by an inactive replication slot from April.
★ Escalated at minute 14; ★ resolved in 4 hours.
★ `age(datfrozenxid)` 1.88B → 41M. ★ Disk 91% → 44%.

## ★ TOP THREE FINDINGS

★ 1. BLOCKING — ★ tenant isolation depends on developer memory
★ SCENARIO: ★ 41 tenant-scoped tables, ★ zero with RLS. ★ 23
  queries have no `merchant_id` filter. ★ Any request hitting
  one of those paths returns another merchant's data. ★ There is
  no mechanism that would detect this.
★ COST: ★ enabling FORCED RLS is ~2 days, ★ plus an index audit
  (`merchant_id` must lead every index — ★ measured elsewhere at
  4,914× when it doesn't).
★ COST OF NOT: ★ a reportable data breach.

★ 2. BLOCKING — ★ no restore has ever been tested
★ SCENARIO: ★ 4.1 TB, gzip, default download concurrency ⇒
  ★ estimated 19 hours. ★ The stated RTO is 4 hours.
★ AND: ★ a slot invalidation, a bad migration, or a corrupted
  page would all require it.
★ COST: ★ 3 days to build and time an automated nightly restore.
★ ADDITIONALLY: ★ lz4 + concurrency 32 would take the estimate
  to ~2 hours (64).

★ 3. HIGH — ★ 302 GB of indexes scanned zero times, on updated
  columns
★ SCENARIO: ★ `idx_payouts_updated_at` and
  `idx_orders_metadata_gin` are on columns every UPDATE changes
  ⇒ ★ HOT update ratio is 0.8% and 4.1% ⇒ ★ every update writes
  every index ⇒ ★ WAL amplification, replica lag, and 302 GB of
  disk.
★ COST: ★ two `DROP INDEX CONCURRENTLY` statements.
★ EXPECTED: ★ HOT ratio to ~95%, ★ WAL down ~4–6× (measured
  elsewhere in similar cases).

## ★ ALSO FOUND (ranked)
4. ★ 217 hours/day of database time on one query with a missing
   index — ★ `Rows Removed by Filter: 8.2M` (78 layer 1).
5. ★ 1.88 billion calls of a 0.02 ms query ⇒ ★ an N+1 (66).
6. ★ 14 read-modify-write patterns ⇒ ★ lost updates at READ
   COMMITTED, ★ silent (43).
7. ★ 6 count-then-insert patterns ⇒ ★ write skew (43).
8. ★ No handling of `40001`/`40P01` anywhere (44, 48).
9. ★ Software-only fencing; ★ failover untested in 2 years, and
   ★ never against a frozen node (63).
10. ★ No `tcp_user_timeout` ⇒ ★ a 30 s failover is a 15 min
    outage (63).
11. ★ The application role owns its tables ⇒ ★ an injection
    becomes `DROP TABLE` (69).
12. ★ `max_slot_wal_keep_size` was unset (now 256 GB).

## ★ QUESTIONS THE TEAM COULD NOT ANSWER
★ • What is the busiest single merchant, in writes per second?
★ • What is the retention policy for `scan_events` (4.1B rows,
    unpartitioned)?
★ • Which reads are allowed to be stale, and by how much?
★ • How would you rebuild the analytics store?
⇒ ★ EACH OF THESE IS A DECISION NOBODY HAS MADE.

## ★ WHAT IS GOOD
★ • money is `bigint` minor units and time is `timestamptz`
    throughout — ★ the two most expensive one-way doors were
    opened correctly.
★ • foreign keys are present and enforced.
★ • the schema is well normalised; ★ snapshots are used
    correctly on orders.
⇒ ★ SAY THIS. ★ A review that only lists faults is not trusted,
  ★ and these three facts are why the other problems are fixable.
```

**Step 6 — what happened next.**

| Finding | Action | Result |
|---|---|---|
| Wraparound | ★ escalated in minute 14 | ★ **outage avoided** |
| Tenant isolation | RLS forced, 41 tables, index audit | 23 unfiltered queries **became harmless** |
| Restore | nightly automated + timed | ★ 19 h estimate → **1 h 52 m measured** |
| 302 GB of indexes | 2 `DROP INDEX` | ★ HOT 0.8% → **96.2%**, WAL **5.1×** less |
| Missing index | 1 `CREATE INDEX` | ★ 217 h/day → **4 h/day** |
| N+1 | batched | 1.88B → **88M** queries |
| `tcp_user_timeout` | one connection-string param | failover impact 15 min → **33 s** |

```
 ★ SEVEN LESSONS:
 ① ★ THE MOST URGENT FINDING WAS FOUND IN MINUTE 14, ★ by a
   query that takes two seconds and is on the checklist because
   ★ the same object appeared in five separate topics.
 ② ★ THE REVIEW STOPPED WHEN IT FOUND AN EMERGENCY. ★ A review
   that keeps going while the database is 260M transactions from
   a write outage is not a review; ★ it is a document.
 ③ ★ THE BIGGEST WINS WERE REMOVALS: ★ two DROP INDEX statements
   gave a 5.1× WAL reduction (78's finding, again).
 ④ ★ "IT'S IN THE RUNBOOK" AND "PATRONI HANDLES IT" ARE NOT
   ANSWERS. ★ The follow-up question — ★ "when was it last
   tested?" — is where the real finding is.
 ⑤ ★ THE UNANSWERED QUESTIONS WERE WORTH MORE THAN MOST
   FINDINGS. ★ Four decisions nobody had made.
 ⑥ ★ SAYING WHAT IS GOOD MATTERS. ★ Money and time were correct
   — ★ the two most expensive one-way doors — ★ which is exactly
   why everything else was fixable.
 ⑦ ★ THREE FINDINGS, RANKED, WITH SCENARIOS. ★ Not twelve,
   equally weighted. ★ The twelve are in the appendix.
```

---

## Common mistakes

**1. Reading the schema top to bottom.**
- *Symptom:* three hours spent on naming conventions; the wraparound emergency missed.
- *Fix:* four passes, in order. Pass 1 takes five minutes and finds the irreversible problems.

**2. Listing everything you find, equally weighted.**
- *Symptom:* 23 comments; the author fixes the typos and ships the two that matter.
- *Fix:* three findings, ranked, with scenarios. The rest goes in an appendix marked as nits.

**3. Stating rules instead of failure scenarios.**
- *Symptom:* "you should use bigint for money" — argued with, or complied with without understanding.
- *Fix:* "₹1,234.55 × 10,000 line items drifts ₹0.03; the ledger won't reconcile; it's unrecoverable once written."

**4. Not asking questions whose answers might change your mind.**
- *Symptom:* a hot-row concern raised against a row taking 0.14 writes/sec.
- *Fix:* ask for the number first. Sometimes the design is right and you learn something.

**5. Accepting "the application handles it".**
- *Symptom:* an invariant with no mechanism, in a system three services write to.
- *Fix:* "what enforces this when a second service writes?" is the follow-up.

**6. Accepting "it's in the runbook" or "the tool handles it".**
- *Symptom:* an untested restore and software-only fencing.
- *Fix:* "when was it last tested, and how long did it take?"

**7. Continuing the review after finding an emergency.**
- *Symptom:* a wraparound risk documented as finding #7 in a report delivered next week.
- *Fix:* stop, escalate, help fix it, then resume.

**8. Not checking replication slots.**
- *Symptom:* the single most recurring incident in this entire curriculum, missed.
- *Fix:* `SELECT * FROM pg_replication_slots WHERE NOT active` — every review, every time.

**9. Only looking for things to add.**
- *Symptom:* recommending indexes while 302 GB of unused ones destroy HOT updates.
- *Fix:* `idx_scan = 0` on large indexes, and the HOT ratio. Removals are often the biggest win.

**10. Not saying what is good.**
- *Symptom:* the review is dismissed as negativity, and the findings are not acted on.
- *Fix:* name the decisions that were made well, specifically. It also tells you what is safe to build on.

**11. Reviewing without the access patterns.**
- *Symptom:* you cannot run the completeness test, which is the highest-value check.
- *Fix:* ask for them. If they don't exist, derive them from `pg_stat_statements` — and note that their absence is itself a finding.

**12. Delivering severity dishonestly.**
- *Symptom:* everything marked "critical", so nothing is.
- *Fix:* blocking / high / medium / nit — and mean "nit" when you say it.

---

## Hands-on proof

**PROVE IT #1–#7 — Example 1** (the five-minute scan finding `double precision` money, `timestamp` without a zone, RLS enabled but not forced, a 4.1-billion-row unpartitioned table, an 812 GB inactive slot with no guard, network calls inside transactions, and 15 of 18 handlers without idempotency).

**PROVE IT #8 — the whole scan as one script.**
```bash
#!/usr/bin/env bash
# ★ db-review.sh — ★ run this first, every time
set -euo pipefail
psql -X -q <<'SQL'
\echo '=== ★ PASS 1: money / time ==='
SELECT table_name, column_name, data_type FROM information_schema.columns
 WHERE table_schema='public'
   AND (data_type IN ('real','double precision','money')
     OR data_type='timestamp without time zone');

\echo '=== ★ PASS 1: tenant tables without FORCED RLS ==='
SELECT c.relname FROM pg_class c JOIN pg_namespace n ON n.oid=c.relnamespace
 WHERE n.nspname='public' AND c.relkind='r'
   AND EXISTS (SELECT 1 FROM information_schema.columns
                WHERE table_name=c.relname
                  AND column_name IN ('tenant_id','org_id','merchant_id'))
   AND NOT (c.relrowsecurity AND c.relforcerowsecurity);

\echo '=== ★ PASS 4: replication slots (★ five incidents) ==='
SELECT slot_name, active, wal_status,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained,
       age(catalog_xmin) AS cat_age FROM pg_replication_slots;
SHOW max_slot_wal_keep_size;

\echo '=== ★ PASS 4: wraparound ==='
SELECT datname, age(datfrozenxid),
       (2^31)::bigint - age(datfrozenxid) AS remaining
  FROM pg_database ORDER BY 2 DESC LIMIT 3;

\echo '=== ★ PASS 3: unused large indexes ==='
SELECT relname, indexrelname, idx_scan,
       pg_size_pretty(pg_relation_size(indexrelid)) AS size
  FROM pg_stat_user_indexes
 WHERE idx_scan < 50 AND pg_relation_size(indexrelid) > 100*1024*1024
 ORDER BY pg_relation_size(indexrelid) DESC LIMIT 10;

\echo '=== ★ PASS 3: HOT update ratio ==='
SELECT relname, n_tup_upd, n_tup_hot_upd,
       round(100.0*n_tup_hot_upd/nullif(n_tup_upd,0),1) AS hot_pct
  FROM pg_stat_user_tables WHERE n_tup_upd > 100000
 ORDER BY hot_pct NULLS LAST LIMIT 5;

\echo '=== ★ PASS 3: N+1 candidates ==='
SELECT calls, mean_exec_time::numeric(10,4) AS ms, substring(query,1,55)
  FROM pg_stat_statements ORDER BY calls DESC LIMIT 5;

\echo '=== ★ PASS 3: capacity consumers ==='
SELECT (calls*mean_exec_time/1000/3600)::numeric(10,1) AS hours,
       calls, substring(query,1,55)
  FROM pg_stat_statements ORDER BY calls*mean_exec_time DESC LIMIT 5;

\echo '=== ★ PASS 4: security ==='
SELECT rolname, rolsuper, rolcreaterole, rolbypassrls FROM pg_roles
 WHERE rolsuper OR rolcreaterole OR rolbypassrls;
SELECT DISTINCT grantee, privilege_type FROM information_schema.role_table_grants
 WHERE grantee='PUBLIC';

\echo '=== ★ PASS 4: settings ==='
SELECT name, setting FROM pg_settings WHERE name IN
 ('idle_in_transaction_session_timeout','statement_timeout','log_lock_waits',
  'track_io_timing','max_slot_wal_keep_size','wal_compression',
  'data_checksums','password_encryption','max_connections');
SQL
```
```
 ★ RUN TIME: ★ ~4 seconds. ★ It answers roughly half the checklist.
```

**PROVE IT #9 — the code greps, as a script.**
```bash
#!/usr/bin/env bash
echo "★ network calls inside transactions:"
rg -U --multiline 'withTransaction\([\s\S]{0,1500}?(await \w+Service\.|fetch\(|axios)' src/ -l

echo "★ read-modify-write (lost update):"
rg -U --multiline 'SELECT[\s\S]{0,600}?UPDATE\s+\w+\s+SET' src/ -l | head

echo "★ count-then-insert (write skew):"
rg -U --multiline 'count\(\*\)[\s\S]{0,800}?INSERT INTO' src/ -l

echo "★ retryable errors handled: $(rg -c "'40001'|'40P01'" src/ | wc -l) files"
echo "★ external POST handlers:   $(rg 'app\.(post|put)\(' src/ | wc -l)"
echo "★ with idempotency:         $(rg -l 'idempotency' src/ | wc -l)"

echo "★ dynamic identifiers (injection):"
rg 'ORDER BY \$\{|FROM \$\{' src/

echo "★ SET without LOCAL (pooler leak):"
rg 'SET (search_path|role|work_mem)' src/ | rg -v 'SET LOCAL'

echo "★ session advisory locks:"
rg 'pg_advisory_lock\(' src/

echo "★ client timeouts configured:"
rg -c 'tcp_user_timeout|connectionTimeoutMillis' src/ || echo "  ★ NONE"
```

**PROVE IT #10 — the questions with no query.**
```
 ★ ASK THESE. ★ THERE IS NO SQL FOR THEM, ★ AND THEY ARE THE
   HIGHEST-VALUE PART OF THE REVIEW:

 ★ ① "When did a restore last succeed, and how long did it take?"
 ★ ② "What fences the old primary, and when was failover last
      tested against a FROZEN node?"
 ★ ③ "What is the busiest single row, in writes per second?"
 ★ ④ "Which reads may be stale, by how much, at the p999?"
 ★ ⑤ "How do you rebuild the derived store, and how long?"
 ★ ⑥ "What is the retention policy for this table, and what
      enforces it?"
 ★ ⑦ "Point at the index for this access pattern."
 ★ ⑧ "What enforces this invariant when a second service writes?"
 ★ ⑨ "How long is the longest transaction, and what is inside it?"

 ⇒ ★ AN ANSWER OF "I DON'T KNOW" IS A FINDING.
   ⇒ ★ AN ANSWER OF "IT'S IN THE RUNBOOK" IS ALSO A FINDING.
```

---

## The design decision framework

```
★★★ FOUR PASSES. THREE FINDINGS. ★ AND THE QUESTIONS THEY
    CANNOT ANSWER. ★★★

 ★ PASS 1 — FIVE MINUTES, ★ IRREVERSIBLE MISTAKES
   money as bigint · timestamptz · ★ tenancy + FORCED RLS ·
   PK type · ★ partitioning for retention · ★ idempotency keys ·
   ★ no network calls in transactions · ★ a TIMED restore
   ⇒ ★ IF ANY FAILS, RAISE IT BEFORE READING FURTHER.

 ★ PASS 2 — CORRECTNESS: ★ "how would we know if this were wrong?"
   invariants→mechanisms · read-modify-write · count-then-insert ·
   40001/40P01 + jitter + idempotency · lock order ·
   reconcilers · ★ timed rebuilds · ★ a signed consistency contract

 ★ PASS 3 — SCALE: ★ "what breaks at 100×, and which layer?" (78)
   ★ every pattern→an index · every index→a pattern ·
   ★ the hottest key (★ measured with a REALISTIC distribution) ·
   unbounded growth · ★ hold time · N+1 · rollups ·
   connection math

 ★ PASS 4 — OPERATIONS: ★ "who is woken, and what do they run?"
   ★ a timed restore · ★ INACTIVE SLOTS (★ check every time) ·
   ★ failover tested against a FROZEN node · ★ tcp_user_timeout ·
   XID age and xmin holders · observability · ★ log sampling ·
   ★ the app role owns nothing · ★ lock_timeout

 ★ DELIVER FOUR PARTS PER FINDING
   ★ ① a SPECIFIC failure scenario, ★ never a rule
   ★ ② the cost NOW vs LATER (★ and one-way doors have no top)
   ★ ③ honest severity — ★ and mean "nit" when you say it
   ★ ④ ★ a QUESTION where the answer might change your mind

 ★ AND THE FIVE RULES OF A USEFUL REVIEW
   ★ ① THREE findings, ranked. ★ The rest is an appendix.
   ★ ② ★ STOP AND ESCALATE if you find an emergency.
   ★ ③ ★ LOOK FOR THINGS TO REMOVE — ★ the biggest wins in this
        curriculum were DROP INDEX, removing a cache, and
        REDUCING connections.
   ★ ④ ★ SAY WHAT IS GOOD, specifically. ★ It builds trust and
        tells you what is safe to build on.
   ★ ⑤ ★ RECORD THE QUESTIONS THEY COULD NOT ANSWER.
        ★ Each one is a decision nobody made.
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
Write the Pass-1 script for your own database and run it. Report every finding, and for each say whether it is reversible and at what cost.

### Exercise 2 — medium (apply it)
Take an existing service you did not write. Run all four passes. Produce a report with exactly three ranked findings — each with a specific failure scenario, the cost now versus later, and an honest severity — plus an appendix and the list of questions you could not answer from the code alone.

### Exercise 3 — hard (production simulation)
You have joined a company and are given one day to review a three-year-old platform with no prior context.

(a) Write the order in which you will work, and justify why Pass 1 comes first.
(b) In minute 14 you find `age(datfrozenxid)` at 1.88 billion. What do you do, and why does the review stop?
(c) Trace the wraparound to its cause using the three-xmin-holders query. Write the resolution sequence in the correct order and explain what happens if you vacuum first.
(d) 41 tenant-scoped tables have no RLS, and 23 queries lack a tenant filter. Write the finding as a failure scenario, with the cost of fixing versus not.
(e) "We have a runbook" is the answer to the restore question. Write the two follow-up questions and estimate the RTO from the compression and concurrency settings.
(f) 302 GB of indexes are scanned zero times, on frequently-updated columns. Explain the mechanism linking those two facts and predict the effect of dropping them.
(g) One query consumes 217 hours/day. Find it, explain the plan, and give the fix.
(h) Write the three top findings and the appendix.
(i) Write the "what is good" section. Why does it matter?
(j) Write the questions the team could not answer, and explain why each is a finding rather than a gap in your understanding.

---

## Mental model checkpoint

1. Name the four passes and what each eliminates. Why is that order correct?
2. Name the eight Pass-1 checks. What do they have in common?
3. Give the cost curve for finding a problem by stage. What is special about one-way doors?
4. Give the four parts of a well-delivered finding.
5. Why is a question sometimes more valuable than a finding?
6. Which single object should you check in every review, and why?
7. What is the completeness test, and what do you do if there are no documented access patterns?
8. Name three follow-up questions that turn a non-answer into a finding.
9. Why should a review look for things to *remove*?
10. Why must you say what is good?
11. When should a review stop before finishing?
12. Why is "everything is critical" the same as "nothing is critical"?

---

## Quick reference card

**★ Pass 1 — five minutes**
```
□ money = bigint minor units          □ all timestamptz
□ ★ tenancy column + FORCED RLS       □ PK type (no UUIDv4 PK)
□ ★ retention ⇒ partitioned           □ ★ idempotency keys
□ ★ no network calls in transactions  □ ★ a TIMED restore
```

**★ Pass 2 — correctness:** invariants→mechanisms · read-modify-write · count-then-insert · `40001`/`40P01` + jitter + idempotency · lock order · reconcilers · ★ timed rebuilds · ★ signed consistency contract.

**★ Pass 3 — scale:** ★ every pattern→an index · every index→a pattern · ★ hottest key (**realistic distribution**) · unbounded growth · ★ hold time · N+1 · rollups (**sum+count**) · connection math.

**★ Pass 4 — operations:** ★ **restore tested & timed** · ★ **inactive slots** · ★ failover vs a **frozen** node · ★ `tcp_user_timeout` · XID age & xmin holders · observability · ★ log sampling · ★ app owns nothing · `lock_timeout`.

**★ The nine questions with no query**
```
when did a restore last succeed / how long?
what fences the old primary / when last tested (frozen)?
busiest single row, writes/sec?
which reads may be stale, by how much, at p999?
how do you rebuild the derived store / how long?
retention policy, and what enforces it?
★ point at the index for this pattern
★ what enforces this when a second service writes?
★ how long is the longest transaction, and what's inside it?
```

**★ Deliver:** a specific failure scenario · cost now vs later · honest severity · ★ a question that could change your mind.
**★ Three findings, ranked. Stop for emergencies. Look for removals. Say what is good. Record the unanswered questions.**

---

## When would I use this at work?

1. **The first week at any new company or on any new system.** One day and this checklist produces something genuinely useful without prior context — and in the example it found a database fourteen minutes from an emergency, using a query that takes two seconds.

2. **Every design review and every significant schema PR.** Pass 1 takes five minutes and catches the mistakes that are unrecoverable. Everything else can be fixed later; those cannot.

3. **Before any launch or scale event.** Passes 3 and 4 answer "what breaks at 100×" and "what happens at 3 a.m." — the two questions that determine whether a launch is an event or an incident.

4. **When you are the one being reviewed.** Run it on your own work first. The questions you cannot answer are the decisions you have not yet made — and finding them yourself is considerably cheaper than having someone else find them.

---

## Connected topics

**Understand before this:** everything. Each check exists because an earlier topic showed the failure. Especially 26 (money and time), 43–52 (correctness), 47/62 (slots and vacuum), 61 (hot rows), 63–65 (operations), 69 (security), 77–78 (design and scale).

---

> ### ★ PHASE 9 COMPLETE — Capstones (77–79)
>
> **77** — design it: the first five steps produce no SQL, and the one-way doors are where the thinking goes.
> **78** — scale it: seven layers, in order, each fix 10–1000× cheaper than the next, and the biggest wins were removals.
> **79** — review it: four passes, three findings, and the questions nobody can answer.
>
> ### ★★★ CURRICULUM COMPLETE — 79 TOPICS ★★★
>
> **What you can now do:** explain any database mechanism from first principles and draw it on a whiteboard · read a query plan and name the constraint · choose an isolation level from measured evidence · design a schema you can defend, with the one-way doors identified and signed · scale a system by finding one binding constraint at a time · diagnose an incident by classification rather than pattern-matching · decide honestly whether a specialised datastore is justified · and walk into an unfamiliar system and find, in an hour, the things that will hurt it.
>
> **The thread through all 79:** *measure before you decide, name the mechanism that enforces it, prefer the cheap reversible fix, and write down what you chose and why.* Almost every production example in this curriculum was a case where someone reached for an expensive tool without measuring — and the measurement would have found something cheaper **and** better.
>
> **Next:** the companion folder — `/docs/db-case-studies/` — where these are applied to twenty-three production designs, end to end.
