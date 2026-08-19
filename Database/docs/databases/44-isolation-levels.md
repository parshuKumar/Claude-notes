# 44 — Transaction Isolation Levels
## Phase: Transactions & Concurrency

---

## ELI5 — The Simple Analogy

A library with one shared card catalogue that people are constantly updating.

**READ COMMITTED** — every time you walk to the catalogue you see whatever it says *right now*. Fast, no waiting. But if you check twice, you may get two different answers, because someone edited it in between.

**REPEATABLE READ** — the moment you begin, the librarian hands you a **photocopy** of the entire catalogue. For the rest of your visit, you read the photocopy. It never changes, and it's internally consistent. But if you try to *write* a card that someone else has already changed since your photocopy was made, the librarian stops you: *"your copy is stale — start over."*

**SERIALIZABLE** — you still get the photocopy, but now the librarian also **writes down which shelves you looked at**. At the end, if your changes and someone else's changes together produce a result that no single ordering of visits could have produced, one of you is sent away to start over.

The photocopy is a snapshot. The list of shelves you looked at is a predicate. And **that list is the entire difference between REPEATABLE READ and SERIALIZABLE** — it is what lets the librarian catch write skew.

---

## Where this fits in the big picture

```
   40 ACID — "I is the only letter with a dial"
   43 the six anomalies — what the dial protects you from
                          │
                          ▼
        ┌──────────────────────────────────────────┐
        │ 44 ISOLATION LEVELS ← YOU ARE HERE       │
        │ the dial itself: 3 real settings         │
        └────────────────────┬─────────────────────┘
                             ▼
     45 locks · 46 MVCC — HOW the levels are implemented
     50 SSI              — HOW SERIALIZABLE detects write skew
```

---

## What is this?

A per-transaction setting that decides **which concurrency anomalies the engine will prevent for you**, and therefore how much correctness you must implement yourself.

The SQL standard defines four levels. **PostgreSQL implements three**, because its MVCC design makes the weakest one impossible:

| You ask for | PostgreSQL gives you |
|---|---|
| `READ UNCOMMITTED` | READ COMMITTED (accepted as syntax, silently upgraded) |
| `READ COMMITTED` | READ COMMITTED — **the default** |
| `REPEATABLE READ` | **snapshot isolation** — stronger than the standard requires |
| `SERIALIZABLE` | **SSI** — true serializability, no phantoms, no write skew |

The levels are defined by the standard in terms of *what they forbid*, not *how they work*. That's why two engines can both claim "REPEATABLE READ" and behave differently — the standard is a floor, not a specification.

---

## Why does it matter for a backend developer?

Because your default is READ COMMITTED, and READ COMMITTED permits **five of the six anomalies**:

```
 ★ WHAT YOU GET BY DEFAULT, AND WHAT IT COSTS YOU

   READ COMMITTED prevents: dirty reads. That is the entire list.
   It permits:  non-repeatable read · phantom · lost update ·
                read skew · WRITE SKEW

 ⇒ EVERY OTHER GUARANTEE IS YOURS TO IMPLEMENT:
     atomic statements, FOR UPDATE, constraints, or a higher level.
```

And because **raising the level is not free and not always sufficient**:

```
 ✗ "Set SERIALIZABLE and stop worrying"  — measured, one workload:
     READ COMMITTED : 18,412 tps, 0 retries
     SERIALIZABLE   :  2,104 tps, 4,882 serialization failures/30s
 ⇒ 8.7× throughput loss, plus a retry loop on every write path.

 ✗ "REPEATABLE READ is a safe middle ground"
 ⇒ it does NOT prevent write skew (Topic 43). The middle ground
   protects reads, not invariants.
```

The engineering skill is **choosing the level per transaction**, knowing exactly what each buys and what it costs.

---

## The physical reality

### The whole difference, in one sentence per level

```
 ★ WHEN IS THE SNAPSHOT TAKEN?  That is the whole mechanism.

 READ COMMITTED
   ⇒ a NEW snapshot at the start of EVERY STATEMENT.
     The world moves between your statements.
     Cost: essentially zero. Snapshots are cheap (Topic 46).

 REPEATABLE READ
   ⇒ ONE snapshot, taken at the first statement, held to COMMIT.
     Your whole transaction sees one instant in time.
     Cost: xmin is held back ⇒ VACUUM cannot remove tuples newer
           than your snapshot ⇒ ★ long transactions cause BLOAT
           (Topic 47). Plus 40001 aborts on write conflicts.

 SERIALIZABLE
   ⇒ the SAME snapshot as REPEATABLE READ, PLUS the engine records
     WHICH DATA YOU READ (SIReadLocks — predicate locks) and looks
     for dangerous read/write dependency patterns among concurrent
     transactions.
     Cost: predicate-lock memory + tracking + more 40001 aborts.
 ⇒ ★ SERIALIZABLE = SNAPSHOT ISOLATION + CONFLICT DETECTION.
   Nothing about the snapshot changes. Only the bookkeeping.
```

### What READ COMMITTED actually does on a write conflict

This is the subtlest behaviour in PostgreSQL and it explains lost updates:

```
 txn A: UPDATE accounts SET balance = 90000 WHERE id=1;   (uncommitted)
 txn B: UPDATE accounts SET balance = 80000 WHERE id=1;

 B's snapshot says the row is version v1. But v1 has t_xmax = A.
 ⇒ B BLOCKS on A's row lock.
 ⇒ A commits.
 ⇒ ★ B DOES NOT ABORT. It performs an "EvalPlanQual" re-check:
      • fetch the NEWEST committed version (v2)
      • RE-EVALUATE THE WHERE CLAUSE against v2
      • if it still matches, apply the update to v2
 ⇒ this is why the atomic form is safe:
      UPDATE accounts SET balance = balance - 10000 WHERE id=1;
      ⇒ "balance" is re-read from v2 during the re-check. Correct.
   and why the application form is not:
      UPDATE accounts SET balance = 90000 WHERE id=1;
      ⇒ 90000 was computed from v1. Applied to v2. ★ LOST UPDATE.
```

```
 AT REPEATABLE READ, THE SAME SITUATION ENDS DIFFERENTLY:
 ⇒ B blocks, A commits, and B sees that the row changed after B's
   snapshot. There is no honest way to proceed.
 ⇒ ERROR: could not serialize access due to concurrent update
   (SQLSTATE 40001)
 ⇒ ★ this is the "first updater wins" rule. It converts a silent
   lost update into a loud, retryable error.
```

### What SSI adds — the memory cost

```
 SERIALIZABLE tracks reads as SIReadLocks in shared memory:

   SELECT count(*) FROM pg_locks WHERE mode='SIReadLock';

 These are NOT locks — they block nothing. They are read markers.
 They are granular: tuple → page → relation, and they ESCALATE
 upward under memory pressure.

 ⇒ ★ ESCALATION CAUSES FALSE POSITIVES. If per-tuple tracking
   exhausts max_pred_locks_per_transaction, PostgreSQL coarsens to
   page-level, then relation-level. A relation-level SIReadLock means
   "this transaction read the whole table" — so ANY write to that
   table looks like a conflict, and you get 40001s that a finer
   granularity would not have produced.

 ⇒ THE TUNING KNOBS:
     max_pred_locks_per_transaction  (default 64)
     max_pred_locks_per_relation     (default -2 ⇒ per_txn / -2)
     max_pred_locks_per_page         (default 2)
 ⇒ if you see high 40001 rates at SERIALIZABLE on large scans,
   raise max_pred_locks_per_transaction before blaming SSI.
```

---

## How it works — step by step

### The table that actually matters

```
 ★ POSTGRESQL'S REAL BEHAVIOUR — not the SQL standard's table

 ┌──────────────────┬──────┬──────┬────────┬──────┬──────┬────────┐
 │                  │dirty │ non- │phantom │ lost │ read │ WRITE  │
 │                  │ read │repeat│        │update│ skew │ SKEW   │
 ├──────────────────┼──────┼──────┼────────┼──────┼──────┼────────┤
 │ READ UNCOMMITTED │  ✓   │  ✗   │   ✗    │  ✗   │  ✗   │   ✗    │
 │  (= READ COMM.)  │      │      │        │      │      │        │
 │ READ COMMITTED   │  ✓   │  ✗   │   ✗    │  ✗   │  ✗   │   ✗    │
 │ REPEATABLE READ  │  ✓   │  ✓   │  ★ ✓   │ ★ ✓* │  ✓   │  ★ ✗   │
 │ SERIALIZABLE     │  ✓   │  ✓   │   ✓    │  ✓   │  ✓   │   ✓    │
 └──────────────────┴──────┴──────┴────────┴──────┴──────┴────────┘
   ✓ = prevented   ✗ = still possible
   * prevented by ABORTING with 40001, not by blocking. You must retry.

 ★ TWO CELLS WHERE POSTGRESQL DIFFERS FROM THE SQL STANDARD:
   ① phantom at REPEATABLE READ — the standard PERMITS it.
      PostgreSQL prevents it, because REPEATABLE READ is snapshot
      isolation (one snapshot ⇒ new rows are invisible).
   ② dirty read at READ UNCOMMITTED — the standard PERMITS it.
      PostgreSQL cannot produce one; MVCC has no such code path.

 ★ THE ONE CELL THAT CAUSES PRODUCTION BUGS:
   WRITE SKEW at REPEATABLE READ. People raise the level expecting
   safety and get an invariant violation anyway. (Topic 43.)
```

### How to set it

```sql
-- ① per transaction, at BEGIN — ★ the form you should use
BEGIN ISOLATION LEVEL REPEATABLE READ;
  …
COMMIT;

-- ② per transaction, after BEGIN but before the first query
BEGIN;
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
  …
COMMIT;
-- ⇒ ERROR if any query has already run: the snapshot is already taken.

-- ③ per session — affects every subsequent transaction on this connection
SET SESSION CHARACTERISTICS AS TRANSACTION ISOLATION LEVEL REPEATABLE READ;
-- ⚠ DANGEROUS WITH A CONNECTION POOL. The setting persists on the
--   pooled connection and leaks into unrelated requests. (Topic 65.)

-- ④ globally
ALTER SYSTEM SET default_transaction_isolation = 'repeatable read';
-- ⚠ every transaction pays. Almost never right.

-- read it back
SHOW transaction_isolation;
SELECT current_setting('transaction_isolation');
```

### Read-only and deferrable — the free upgrades

```sql
-- ★ READ ONLY at SERIALIZABLE costs almost nothing.
--   A read-only transaction cannot create a dangerous structure alone,
--   so SSI's bookkeeping is much cheaper.
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE READ ONLY;
  SELECT …;   -- a report that is guaranteed consistent
COMMIT;

-- ★ DEFERRABLE: for long read-only reports, this is the best option
--   in PostgreSQL.
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE READ ONLY DEFERRABLE;
  SELECT …;
COMMIT;
-- ⇒ the transaction WAITS until it can acquire a snapshot that is
--   guaranteed conflict-free, then runs with ★ ZERO chance of 40001
--   and ★ zero SSI overhead for the rest of the system.
-- ⇒ THE TRADE: it may wait at the start. For a nightly report, free.
```

---

## Concept breakdown

```
THE THREE REAL LEVELS
├── READ COMMITTED  ★ the default
│    snapshot: NEW PER STATEMENT
│    prevents: dirty reads only
│    on write conflict: BLOCKS, then re-checks and proceeds (EPQ)
│    cost: ~zero
│    ⇒ correct when every write is an atomic statement or protected
│      by a constraint/lock
├── REPEATABLE READ  (= snapshot isolation)
│    snapshot: ONE, at the first statement, held to COMMIT
│    prevents: + non-repeatable, phantom, read skew, lost update
│    ★ does NOT prevent: WRITE SKEW
│    on write conflict: ABORT with 40001 ("first updater wins")
│    cost: retries; ★ holds xmin back ⇒ VACUUM blocked ⇒ bloat
│    ⇒ correct for multi-statement reports and read-modify-write
└── SERIALIZABLE  (= snapshot isolation + SSI)
     snapshot: same as REPEATABLE READ
     ★ PLUS: records what you READ (SIReadLocks) and detects
       dangerous read/write dependency cycles
     prevents: everything, including write skew
     cost: predicate-lock memory, tracking, more 40001s
     ⇒ correct when an invariant spans rows you did not write

★ SERIALIZABLE = REPEATABLE READ + READ TRACKING. That's the delta.

WHAT POSTGRESQL DOES DIFFERENTLY FROM THE STANDARD
├── READ UNCOMMITTED is silently READ COMMITTED (MVCC can't do dirty)
├── REPEATABLE READ PREVENTS PHANTOMS (standard permits them)
└── ⇒ never reuse another engine's isolation table

THE COSTS, NAMED
├── READ COMMITTED : you implement correctness yourself
├── REPEATABLE READ: retries + ★ long txns block VACUUM (bloat)
└── SERIALIZABLE   : more retries + predicate-lock memory
                     ★ escalation ⇒ FALSE-POSITIVE 40001s

WHAT YOU OWE THE MOMENT YOU LEAVE READ COMMITTED
└── ★ A RETRY LOOP. 40001 and 40P01, backoff + jitter, bounded,
      metered. An unhandled 40001 is a 500 the user didn't need.

THE FREE UPGRADE
└── SERIALIZABLE READ ONLY DEFERRABLE
     ⇒ consistent reports, zero 40001 risk, zero SSI overhead,
       at the cost of possibly waiting to start.
```

---

## Diagrams

**Diagram 1 — big picture: when the snapshot is taken**

```
 READ COMMITTED — a new snapshot per statement
 ────────────────────────────────────────────────────────────────
  BEGIN     SELECT       SELECT       UPDATE      COMMIT
    │         │            │            │           │
    │         ▼            ▼            ▼           │
    │       [snap1]      [snap2]      [snap3]       │
    │         └─ world ────┴─ moves ───┴─ between ──┘
    ⇒ each statement sees the latest committed data.
      Cheap. Inconsistent across statements.

 REPEATABLE READ — one snapshot, held
 ────────────────────────────────────────────────────────────────
  BEGIN     SELECT       SELECT       UPDATE      COMMIT
    │         │            │            │           │
    │       [snap]─────────┴────────────┴───────────┘
    │         ★ ONE view of one instant, for the whole transaction
    ⇒ consistent reads. Writes to rows changed since the snapshot
      ABORT with 40001.
    ⇒ ★ xmin is pinned for the whole duration → VACUUM blocked

 SERIALIZABLE — one snapshot + read tracking
 ────────────────────────────────────────────────────────────────
  BEGIN     SELECT       SELECT       UPDATE      COMMIT
    │         │            │            │           │
    │       [snap]─────────┴────────────┴───────────┤
    │         │            │                        │
    │      SIReadLock  SIReadLock                   ▼
    │      recorded    recorded             ★ CHECK: did my writes
    │                                         and someone else's
    │                                         reads form a dangerous
    │                                         cycle?  → 40001 if so
    ⇒ same snapshot. The ONLY addition is the bookkeeping.
```

**Diagram 2 — data flow: the same write conflict at each level**

```
  A: UPDATE accounts SET balance=90000 WHERE id=1;   (uncommitted)
  B: UPDATE accounts SET balance=80000 WHERE id=1;

 ┌─ READ COMMITTED ────────────────────────────────────────────────┐
 │ B blocks on A's row lock                                        │
 │ A commits                                                       │
 │ B performs EvalPlanQual: re-fetch newest version, re-check WHERE │
 │ B applies its update to the NEW version                         │
 │ ⇒ SUCCEEDS. ★ If B's value came from a prior SELECT, that value │
 │   is now stale ⇒ LOST UPDATE, silently.                         │
 └─────────────────────────────────────────────────────────────────┘

 ┌─ REPEATABLE READ ───────────────────────────────────────────────┐
 │ B blocks on A's row lock                                        │
 │ A commits                                                       │
 │ B sees the row changed after B's snapshot                       │
 │ ⇒ ★ ERROR 40001: could not serialize access due to concurrent   │
 │   update.  B must retry.                                        │
 │ ⇒ the lost update became a LOUD, RETRYABLE failure.             │
 └─────────────────────────────────────────────────────────────────┘

 ┌─ SERIALIZABLE ──────────────────────────────────────────────────┐
 │ identical to REPEATABLE READ for this case                      │
 │ ★ PLUS: even if A and B had written DIFFERENT rows, SSI would   │
 │   check whether their reads and writes formed a dangerous cycle │
 │   ⇒ ERROR 40001: … due to read/write dependencies among         │
 │     transactions.  ← this is the write-skew catch.              │
 └─────────────────────────────────────────────────────────────────┘
```

**Diagram 3 — before/after: choosing per-transaction instead of globally**

```
 ✗ GLOBAL SERIALIZABLE
 ┌───────────────────────────────────────────────────────────────┐
 │ default_transaction_isolation = 'serializable'                │
 │                                                                │
 │  GET  /products      → SSI tracking, snapshot pinned  ✗ waste │
 │  GET  /orders/:id    → SSI tracking                   ✗ waste │
 │  POST /cart/add      → SSI tracking, retry loop needed ✗      │
 │  POST /checkout      → SSI tracking                   ✓ needed│
 │  GET  /health        → SSI tracking                   ✗ waste │
 │                                                                │
 │ MEASURED: 18,412 tps → 2,104 tps    ★ 8.7× loss               │
 │           4,882 serialization failures per 30s                │
 │ ⇒ every path pays for the one path that needed it.            │
 └───────────────────────────────────────────────────────────────┘

 ✓ PER-TRANSACTION
 ┌───────────────────────────────────────────────────────────────┐
 │ default_transaction_isolation = 'read committed'   (default)  │
 │                                                                │
 │  GET  /products      → READ COMMITTED                 ✓       │
 │  GET  /orders/:id    → READ COMMITTED                 ✓       │
 │  POST /cart/add      → READ COMMITTED + atomic UPDATE ✓       │
 │  POST /checkout      → ★ SERIALIZABLE + retry loop    ✓       │
 │  GET  /reports/daily → ★ SERIALIZABLE READ ONLY DEFERRABLE ✓  │
 │  GET  /health        → READ COMMITTED                 ✓       │
 │                                                                │
 │ MEASURED: 17,900 tps    ★ 2.8% loss                           │
 │           ~2% retries, confined to /checkout                  │
 └───────────────────────────────────────────────────────────────┘
```

---

## Example 1 — basic

```sql
CREATE TABLE accounts (id bigint PRIMARY KEY, balance_minor bigint NOT NULL);
INSERT INTO accounts VALUES (1,100000),(2,100000);
```

**See the default.**
```sql
SHOW default_transaction_isolation;
```
```
 default_transaction_isolation
-------------------------------
 read committed
```

**Prove READ UNCOMMITTED is silently READ COMMITTED.**
```sql
BEGIN ISOLATION LEVEL READ UNCOMMITTED;
SHOW transaction_isolation;
```
```
 transaction_isolation
-----------------------
 read committed        ★ the request was upgraded
```
```sql
ROLLBACK;
```

**Prove the snapshot timing difference.**
```sql
-- session 1                          -- session 2
BEGIN;  -- READ COMMITTED
SELECT balance_minor FROM accounts WHERE id=1;   -- 100000
                                      UPDATE accounts SET balance_minor=1
                                        WHERE id=1;
SELECT balance_minor FROM accounts WHERE id=1;   -- 1        ★ moved
COMMIT;

UPDATE accounts SET balance_minor=100000 WHERE id=1;

BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT balance_minor FROM accounts WHERE id=1;   -- 100000
                                      UPDATE accounts SET balance_minor=1
                                        WHERE id=1;
SELECT balance_minor FROM accounts WHERE id=1;   -- 100000   ★ frozen
COMMIT;
SELECT balance_minor FROM accounts WHERE id=1;   -- 1  (after commit)
```

**Prove REPEATABLE READ turns a lost update into an error.**
```sql
UPDATE accounts SET balance_minor=100000 WHERE id=1;

-- session 1                          -- session 2
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT balance_minor FROM accounts WHERE id=1;
                                      BEGIN ISOLATION LEVEL REPEATABLE READ;
                                      SELECT balance_minor FROM accounts WHERE id=1;
UPDATE accounts SET balance_minor=90000 WHERE id=1;
COMMIT;
                                      UPDATE accounts SET balance_minor=80000
                                        WHERE id=1;
```
```
ERROR:  could not serialize access due to concurrent update
```
```sql
-- session 2: check the code
                                      ROLLBACK;
-- ★ SQLSTATE 40001. Retryable. Not a bug — a signal.
```

**Prove REPEATABLE READ does NOT prevent write skew.**
```sql
CREATE TABLE on_call (doctor_id text PRIMARY KEY, is_on_call boolean NOT NULL);
INSERT INTO on_call VALUES ('alice',true),('bob',true);

-- session 1                          -- session 2
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT count(*) FROM on_call WHERE is_on_call;   -- 2
                                      BEGIN ISOLATION LEVEL REPEATABLE READ;
                                      SELECT count(*) FROM on_call WHERE is_on_call; -- 2
UPDATE on_call SET is_on_call=false WHERE doctor_id='alice';
COMMIT;
                                      UPDATE on_call SET is_on_call=false
                                        WHERE doctor_id='bob';
                                      COMMIT;    -- ★ SUCCEEDS
SELECT count(*) FROM on_call WHERE is_on_call;
```
```
 count
-------
     0        ★ the invariant is broken AT REPEATABLE READ
```

**And that SERIALIZABLE does.**
```sql
UPDATE on_call SET is_on_call=true;
-- same script with ISOLATION LEVEL SERIALIZABLE
```
```
ERROR:  could not serialize access due to read/write dependencies among transactions
DETAIL:  Reason code: Canceled on identification as a pivot, during commit attempt.
HINT:  The transaction might succeed if retried.
```

**See the predicate locks SSI uses.**
```sql
-- with a SERIALIZABLE transaction open elsewhere
SELECT locktype, relation::regclass, page, tuple, mode
  FROM pg_locks WHERE mode = 'SIReadLock';
```
```
 locktype | relation |  page  | tuple |    mode
----------+----------+--------+-------+------------
 tuple    | on_call  |      0 |     1 | SIReadLock
 tuple    | on_call  |      0 |     2 | SIReadLock
   ★ these block nothing. They are read markers used to detect cycles.
```

**Use the deferrable read-only form for a report.**
```sql
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE READ ONLY DEFERRABLE;
SELECT sum(balance_minor) FROM accounts;
COMMIT;
-- ★ guaranteed consistent, guaranteed never to get 40001.
```

---

## Example 2 — production scenario

**The situation.** A B2B invoicing platform, 40 million invoices. After a duplicate-invoice-number incident, an engineer set `default_transaction_isolation = 'serializable'` globally. It fixed the duplicates. Two weeks later:

```
 SYMPTOMS AFTER THE GLOBAL CHANGE
   p99 API latency        180 ms → 2,400 ms
   error rate             0.02%  → 4.1%   (all HTTP 500)
   throughput             8,200 rps → 1,140 rps
   the "duplicate invoice number" bug                     ✓ gone
   ★ and: table bloat on invoices grew 12 GB → 61 GB in 14 days
```

**Step 1 — find where the errors are.**

```sql
SELECT
  substring(query from 1 for 60) AS q,
  calls, mean_exec_time::numeric(10,2) AS mean_ms
FROM pg_stat_statements
ORDER BY calls DESC LIMIT 5;

-- and the failures, from the application log aggregation:
```
```
 SQLSTATE  count/hour  handler
 --------  ----------  --------------------------------
 40001        184,220  GET  /api/invoices           ← ★ READ-ONLY
 40001         31,400  GET  /api/customers/:id
 40001          2,910  POST /api/invoices           ← the one that needed it
 40P01            140  POST /api/invoices
```

```
 ★ 98.4% OF THE SERIALIZATION FAILURES WERE ON READ-ONLY ENDPOINTS.
   Those endpoints did not need SERIALIZABLE at all — and read-only
   transactions at SERIALIZABLE should rarely conflict.
```

**Step 2 — why were read-only queries failing?**

```sql
SELECT count(*), mode FROM pg_locks WHERE mode='SIReadLock' GROUP BY mode;
```
```
  count  |    mode
---------+------------
   61402 | SIReadLock
```
```sql
SHOW max_pred_locks_per_transaction;   -- 64
SHOW max_pred_locks_per_relation;      -- -2
SHOW max_pred_locks_per_page;          -- 2
```

```
 ★ THE MECHANISM:
   GET /api/invoices runs a paginated scan touching ~2,000 tuples.
   With max_pred_locks_per_transaction = 64, per-tuple tracking is
   exhausted immediately.
   ⇒ SSI ESCALATES: tuple → page → ★ RELATION.
   ⇒ a relation-level SIReadLock means "this transaction read the
     ENTIRE invoices table."
   ⇒ ANY concurrent INSERT into invoices now looks like a conflict.
   ⇒ ★ 184,220 FALSE-POSITIVE 40001s PER HOUR on a read-only endpoint.
```

**Step 3 — and the bloat?**

```sql
SELECT pid, state, xact_start, now()-xact_start AS age, query
  FROM pg_stat_activity
 WHERE xact_start IS NOT NULL
 ORDER BY xact_start LIMIT 5;
```
```
  pid  | state  |         age         | query
-------+--------+---------------------+---------------------------
 41209 | idle in transaction | 00:41:12 | SELECT … FROM invoices …
 41377 | idle in transaction | 00:28:04 | SELECT … FROM invoices …
```
```sql
SELECT relname, n_dead_tup, last_autovacuum
  FROM pg_stat_user_tables WHERE relname='invoices';
```
```
 relname  | n_dead_tup | last_autovacuum
----------+------------+-----------------
 invoices |  184000000 | 2026-07-24 03:11:02+05:30   ← ★ 14 days ago
```

```
 ★ THE SECOND MECHANISM:
   At SERIALIZABLE (and REPEATABLE READ), the snapshot is taken at
   the FIRST STATEMENT and held to COMMIT. The transaction's xmin
   is pinned for the whole duration.
   ⇒ VACUUM cannot remove any tuple newer than the oldest pinned xmin.
   ⇒ report endpoints holding 40-minute transactions pinned xmin
     for 40 minutes at a time, continuously.
   ⇒ ★ AUTOVACUUM RAN AND REMOVED NOTHING FOR 14 DAYS.
     12 GB → 61 GB. (Topic 47.)

 ⇒ AT READ COMMITTED this does not happen the same way: each
   statement takes a fresh snapshot, so xmin advances between
   statements even inside a long transaction.
```

**Step 4 — the fix. Revert the global setting; apply it where it belongs.**

```sql
ALTER SYSTEM SET default_transaction_isolation = 'read committed';
SELECT pg_reload_conf();
```

```js
// ① the read-only report endpoints — the free upgrade
async function listInvoices(orgId, page) {
  return withTransaction(async (tx) => {
    await tx.query(
      'SET TRANSACTION ISOLATION LEVEL REPEATABLE READ READ ONLY');
    // ★ consistent pagination across statements, no SSI tracking,
    //   no predicate locks, no 40001.
    return tx.query(
      `SELECT id, invoice_number, total_minor, issued_at
         FROM invoices
        WHERE org_id=$1 AND status <> 'draft'
        ORDER BY issued_at DESC, id DESC
        LIMIT 50 OFFSET $2`, [orgId, page * 50]);
  });
}

// ② the nightly reconciliation report — deferrable
async function reconciliationReport(day) {
  return withTransaction(async (tx) => {
    await tx.query(
      'SET TRANSACTION ISOLATION LEVEL SERIALIZABLE READ ONLY DEFERRABLE');
    // ★ guaranteed consistent, guaranteed no 40001, zero SSI cost
    //   to other transactions. May wait a moment to start.
    return tx.query(`SELECT … FROM invoices JOIN payments … WHERE …`, [day]);
  });
}
```

```sql
-- ③ ★ the actual duplicate-invoice-number bug: not an isolation problem.
--    It was a write skew, and the correct fix is a constraint.
ALTER TABLE invoices
  ADD CONSTRAINT uq_invoice_number_per_org UNIQUE (org_id, invoice_number);
-- ⇒ enforced at READ COMMITTED, no retries, and ★ unbypassable by
--   any code path anyone forgets. (Topics 24, 43.)
```

```js
// ④ the one path that genuinely needs SERIALIZABLE:
//    "an org's total outstanding must not exceed its credit limit"
//    — an aggregate over a set, written into that set. No constraint
//    can express it. (Topic 43's fourth case.)
async function issueInvoice(orgId, lines) {
  return retryOnSerializationFailure(async () =>
    withTransaction(async (tx) => {
      await tx.query('SET TRANSACTION ISOLATION LEVEL SERIALIZABLE');
      const { rows: [agg] } = await tx.query(
        `SELECT coalesce(sum(total_minor),0) AS outstanding_minor
           FROM invoices WHERE org_id=$1 AND status='unpaid'`, [orgId]);
      const { rows: [org] } = await tx.query(
        'SELECT credit_limit_minor FROM organisations WHERE id=$1', [orgId]);
      const total = lines.reduce((s, l) => s + l.amount_minor, 0);
      if (Number(agg.outstanding_minor) + total > Number(org.credit_limit_minor))
        throw new AppError('CREDIT_LIMIT_EXCEEDED');
      return tx.query('INSERT INTO invoices (…) VALUES (…) RETURNING id', […]);
    }));
}
```

```js
// ⑤ the retry helper — owed the moment you leave READ COMMITTED
const RETRYABLE = new Set(['40001', '40P01']);   // serialization, deadlock

async function retryOnSerializationFailure(fn, { attempts = 4 } = {}) {
  for (let i = 0; i < attempts; i++) {
    try {
      return await fn();
    } catch (e) {
      if (!RETRYABLE.has(e.code) || i === attempts - 1) throw e;
      metrics.increment('db.serialization_retry', { sqlstate: e.code });
      // exponential backoff + full jitter — without jitter, retries
      // collide and the same pair conflicts again
      const base = 10 * 2 ** i;                  // 10, 20, 40, 80 ms
      await sleep(Math.random() * base);
    }
  }
}
```

**Step 5 — raise the predicate-lock budget for the one SERIALIZABLE path.**

```sql
-- issueInvoice reads a per-org aggregate — a few hundred tuples.
-- 64 is too low; escalation to relation level would make every
-- concurrent invoice conflict.
ALTER SYSTEM SET max_pred_locks_per_transaction = 512;   -- needs restart
-- ⇒ keeps tracking at tuple/page granularity ⇒ conflicts are real,
--   not artefacts of escalation.
```

**Step 6 — results.**

| Metric | Global SERIALIZABLE | Per-transaction | Change |
|---|---|---|---|
| Throughput | 1,140 rps | 8,050 rps | **7.1×** |
| p99 latency | 2,400 ms | 190 ms | **12.6×** |
| 40001/hour | 218,530 | 1,840 | **99.2% fewer** |
| Error rate (5xx) | 4.1% | 0.01% | retries absorb them |
| `invoices` bloat | 61 GB, growing | 13 GB, stable | autovacuum works again |
| Duplicate invoice numbers | 0 | **0** | now a `UNIQUE` constraint |

```
 ★ THE LESSON, PRECISELY:
   The global change fixed the bug — but the bug was a write skew
   whose correct fix was a UNIQUE constraint costing nothing.
   The isolation level was the expensive answer to a question a
   constraint answered for free, and it dragged in two second-order
   costs nobody predicted:
     ① SSI predicate-lock ESCALATION ⇒ false-positive 40001s on
        read-only endpoints (98.4% of all failures)
     ② PINNED xmin ⇒ autovacuum removed nothing for 14 days ⇒ 5× bloat
   ⇒ ★ ONE of five endpoints genuinely needed SERIALIZABLE.
```

---

## Common mistakes

**1. Setting the level globally.**
- *Symptom:* every endpoint slows, retries appear everywhere, and unrelated tables bloat.
- *Engine-level why:* every transaction pays SSI tracking and snapshot pinning for the benefit of the one path that needed it.
- *Fix:* `BEGIN ISOLATION LEVEL …` per transaction. Leave the default alone.

**2. Assuming REPEATABLE READ prevents write skew.**
- *Symptom:* the invariant still breaks after raising the level.
- *Engine-level why:* snapshot isolation protects rows you *read*; write skew's conflict is between one transaction's write and another's read *predicate*.
- *Fix:* a constraint that materialises the conflict, or SERIALIZABLE (Topic 43).

**3. Leaving the level set on a pooled connection.**
- *Symptom:* random unrelated requests get 40001, or read stale-feeling data.
- *Engine-level why:* `SET SESSION CHARACTERISTICS` persists on the physical connection; a pool hands it to the next request.
- *Fix:* only ever set it per transaction. `SET TRANSACTION …` is scoped to the transaction and resets automatically (Topic 65).

**4. Treating 40001 as a bug.**
- *Symptom:* 500s spike after raising the level; someone reverts it and concludes the level "doesn't work."
- *Engine-level why:* at REPEATABLE READ and SERIALIZABLE, 40001 is the *designed* mechanism. It means "retry me."
- *Fix:* a bounded retry loop with jitter, and a metric on the retry rate.

**5. Retrying without jitter.**
- *Symptom:* retry storms — the same two transactions conflict again on the same schedule.
- *Fix:* full jitter (`Math.random() * base`), not fixed backoff.

**6. Long transactions at REPEATABLE READ / SERIALIZABLE.**
- *Symptom:* `n_dead_tup` climbs into the hundreds of millions while autovacuum runs constantly and removes nothing.
- *Engine-level why:* one snapshot held to COMMIT pins the transaction's xmin; VACUUM cannot remove tuples newer than the oldest pinned xmin.
- *Fix:* keep these transactions short; use `SERIALIZABLE READ ONLY DEFERRABLE` for long reports; alert on `max(now() - xact_start)` (Topic 47).

**7. Not raising `max_pred_locks_per_transaction` for large SERIALIZABLE scans.**
- *Symptom:* very high 40001 rates that don't correspond to real conflicts.
- *Engine-level why:* predicate-lock escalation to relation level makes every write to that table look like a conflict.
- *Fix:* raise it (default 64 is low for anything scanning more than a few hundred tuples), or narrow the query.

**8. Raising the level when a constraint would do.**
- *Symptom:* an 8× throughput loss to enforce uniqueness.
- *Fix:* `UNIQUE` / `EXCLUDE` are enforced at READ COMMITTED, need no retries, and cannot be bypassed. Reach for them first (Topics 24, 43).

**9. Assuming another engine's table applies.**
- *Symptom:* over-engineering against phantoms at REPEATABLE READ, or expecting dirty reads to be possible.
- *Fix:* PostgreSQL's REPEATABLE READ is snapshot isolation; it prevents phantoms. `READ UNCOMMITTED` is READ COMMITTED.

---

## Hands-on proof

**PROVE IT #1 — READ UNCOMMITTED is a lie.**
```sql
BEGIN ISOLATION LEVEL READ UNCOMMITTED; SHOW transaction_isolation; ROLLBACK;
-- read committed
```

**PROVE IT #2 — snapshot timing.** (Example 1.)

**PROVE IT #3 — 40001 at REPEATABLE READ.** (Example 1.)

**PROVE IT #4 — write skew survives REPEATABLE READ, dies at SERIALIZABLE.** (Example 1.)

**PROVE IT #5 — the throughput cost of each level.**
```bash
cat > /tmp/iso.sql <<'EOF'
\set id random(1, 1000)
BEGIN;
SELECT balance_minor FROM accounts WHERE id = :id;
UPDATE accounts SET balance_minor = balance_minor - 1 WHERE id = :id;
COMMIT;
EOF
for lvl in "read committed" "repeatable read" "serializable"; do
  echo "== $lvl"
  PGOPTIONS="-c default_transaction_isolation='$lvl'" \
    pgbench -f /tmp/iso.sql -c 50 -j 4 -T 30 shop 2>&1 | grep -E 'tps|failed'
done
```
```
 == read committed
 tps = 18,412.4    failed = 0
 == repeatable read
 tps =  6,880.1    failed = 2,214   (40001)
 == serializable
 tps =  2,104.7    failed = 4,882   (40001)
```

**PROVE IT #6 — see the predicate locks.**
```sql
SELECT locktype, relation::regclass, mode, count(*)
  FROM pg_locks WHERE mode='SIReadLock'
 GROUP BY 1,2,3;
-- ★ locktype 'relation' means escalation happened — expect false positives.
```

**PROVE IT #7 — prove a long REPEATABLE READ transaction blocks VACUUM.**
```sql
-- session 1
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT 1 FROM accounts LIMIT 1;   -- take the snapshot

-- session 2
UPDATE accounts SET balance_minor = balance_minor + 1;   -- create dead tuples
VACUUM (VERBOSE) accounts;
```
```
INFO:  vacuuming "public.accounts"
INFO:  table "accounts": found 0 removable, 2000 nonremovable row versions
DETAIL:  1000 dead row versions cannot be removed yet, oldest xmin: 8842119
   ★ "cannot be removed yet" — session 1's pinned xmin is the reason.
```
```sql
-- session 1
COMMIT;
-- session 2
VACUUM (VERBOSE) accounts;   -- now they are removable
```

**PROVE IT #8 — the deferrable form never fails.**
```sql
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE READ ONLY DEFERRABLE;
SELECT sum(balance_minor) FROM accounts;
COMMIT;
-- ★ run this under concurrent write load. It will never raise 40001.
```

---

## The design decision framework

```
★★★ CHOOSE PER TRANSACTION. THE DEFAULT STAYS READ COMMITTED. ★★★

 ① CAN A CONSTRAINT OR AN ATOMIC STATEMENT SOLVE IT?
    ⇒ ★ TRY THIS FIRST, ALWAYS.
      UNIQUE / EXCLUDE / CHECK       — free at runtime, no retries,
                                       unbypassable
      UPDATE t SET x = x - $1 WHERE … AND x >= $1
                                     — free, no gap to exploit
    ⇒ if yes: STAY AT READ COMMITTED. You are done.
    ⇒ 3 of 4 real cases end here (Topic 43).

 ② WHAT DOES THE TRANSACTION ACTUALLY NEED?

    Single-statement writes, or writes protected by ①
      ⇒ READ COMMITTED

    Multiple SELECTs whose results must agree with each other
    (reports, pagination, exports, "sum A minus sum B")
      ⇒ REPEATABLE READ READ ONLY
      ⇒ or ★ SERIALIZABLE READ ONLY DEFERRABLE for long reports:
        consistent, zero 40001, zero SSI cost to others

    Read-modify-write across statements that you cannot make atomic
      ⇒ REPEATABLE READ (40001 ⇒ retry), or SELECT … FOR UPDATE

    ★ An INVARIANT OVER A SET that you also write into
      ("count must not exceed N", "sum must not exceed the limit",
       "at least one must remain")
      ⇒ no constraint can express it
      ⇒ ★ SERIALIZABLE + a retry loop
      ⇒ the alternative is a shared counter row, which is correct at
        READ COMMITTED but creates a HOT ROW (case study 01) —
        compare, measure, choose.

 ③ THE MOMENT YOU LEAVE READ COMMITTED, YOU OWE:
    ✓ a retry loop on 40001 and 40P01
    ✓ exponential backoff with FULL JITTER
    ✓ a bounded attempt count (3–4)
    ✓ a metric on the retry rate — a rising rate is a design signal
    ✓ ★ SHORT TRANSACTIONS. The snapshot pins xmin and blocks VACUUM.

 ④ HOW TO SET IT
    ✓ BEGIN ISOLATION LEVEL …           per transaction
    ✓ SET TRANSACTION …                 inside BEGIN, before the
                                        first query
    ✗ SET SESSION CHARACTERISTICS …     leaks across a pool
    ✗ default_transaction_isolation     every path pays for one

 ⑤ IF YOU SEE HIGH 40001 AT SERIALIZABLE
    ① check pg_locks for locktype='relation' AND mode='SIReadLock'
       ⇒ escalation ⇒ FALSE POSITIVES
       ⇒ raise max_pred_locks_per_transaction (default 64 is low)
    ② narrow the query — fewer tuples read, fewer conflicts
    ③ mark read-only transactions READ ONLY — they conflict far less
    ④ reconsider ①: is there a constraint that would do this for free?

THE ONE-LINE SUMMARY
  READ COMMITTED  + a constraint      → most write paths
  REPEATABLE READ + READ ONLY         → consistent multi-statement reads
  SERIALIZABLE READ ONLY DEFERRABLE   → long reports, free consistency
  SERIALIZABLE    + retry loop        → set-invariants only
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
Fill in PostgreSQL's real anomaly table for all four levels and six anomalies. Then mark the two cells where PostgreSQL is stronger than the SQL standard, and the one cell that causes production bugs. Verify three of your answers with two `psql` sessions.

### Exercise 2 — medium (apply it)
Choose the isolation level (and any other mechanism) for each, with justification:
(a) `POST /orders` — insert an order and its lines · (b) `GET /reports/monthly` — a 90-second query joining six tables · (c) decrement inventory by 1 · (d) "an organisation may have at most 5 active API keys" · (e) paginated listing where page 2 must not show a row already seen on page 1 · (f) a health check

For at least two, argue that the correct answer is READ COMMITTED plus something else.

### Exercise 3 — hard (production simulation)
An engineer set `default_transaction_isolation = 'serializable'` to fix duplicate invoice numbers. It worked. Two weeks later: throughput 8,200 → 1,140 rps, p99 180 → 2,400 ms, 4.1% 5xx, and the `invoices` table grew from 12 GB to 61 GB.

(a) Explain both second-order effects at the engine level: why read-only endpoints produced 98.4% of the 40001s, and why autovacuum stopped reclaiming space.
(b) Write the queries that diagnose each (`pg_locks`, `pg_stat_activity`, `pg_stat_user_tables`, `VACUUM VERBOSE`).
(c) The original duplicate-invoice bug was a write skew. Give the fix that costs nothing at runtime and explain why it is *stronger* than SERIALIZABLE, not merely cheaper.
(d) One endpoint genuinely needs SERIALIZABLE. Identify its shape and explain why no constraint can express it.
(e) Assign a level to each of the five endpoints, with justification, and say which get `READ ONLY` and which get `DEFERRABLE`.
(f) Write the retry helper: SQLSTATEs, backoff, jitter, bound, metric. Explain why fixed backoff causes retry storms.
(g) Compute the correct `max_pred_locks_per_transaction` for the SERIALIZABLE path and explain what escalation would cost if you left it at 64.
(h) Write the alert that would have caught this in week one, before the bloat.

---

## Mental model checkpoint

1. Name the three real levels in PostgreSQL and say, in one sentence each, when the snapshot is taken.
2. What is the *only* difference between REPEATABLE READ and SERIALIZABLE?
3. Which two cells of PostgreSQL's anomaly table differ from the SQL standard, and why?
4. At READ COMMITTED, what does a transaction do when it hits an uncommitted row lock? Why does that produce lost updates but not corrupt atomic `UPDATE`s?
5. What is SQLSTATE 40001, when should you expect it, and what must the application do?
6. Why do long REPEATABLE READ transactions cause table bloat, and why is READ COMMITTED different?
7. What is `SERIALIZABLE READ ONLY DEFERRABLE` and when is it the best choice available?
8. You see 200k 40001s/hour at SERIALIZABLE on a read-only endpoint. What is the first thing you check?

---

## Quick reference card

| Level | Snapshot | Prevents | On write conflict | Cost |
|---|---|---|---|---|
| READ UNCOMMITTED | — | *(= READ COMMITTED)* | — | — |
| **READ COMMITTED** ★default | per **statement** | dirty read | blocks, re-checks, **proceeds** | ~zero |
| **REPEATABLE READ** | per **transaction** | + non-repeatable, phantom, read skew, lost update | **abort 40001** | retries; **pins xmin → bloat** |
| **SERIALIZABLE** | per transaction **+ read tracking** | **+ write skew** | abort 40001 | more retries; predicate-lock memory |

```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;                        -- per txn ✓
BEGIN; SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;          -- per txn ✓
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE READ ONLY DEFERRABLE;  -- ★ reports
SET SESSION CHARACTERISTICS AS TRANSACTION ISOLATION LEVEL …; -- ✗ leaks in a pool
ALTER SYSTEM SET default_transaction_isolation = …;           -- ✗ everyone pays
```

**Retryable:** `40001` serialization_failure · `40P01` deadlock. Backoff **with full jitter**, bounded, metered.

**Diagnostics**
```sql
SHOW transaction_isolation;
SELECT count(*), locktype FROM pg_locks WHERE mode='SIReadLock' GROUP BY 2;
-- locktype='relation' ⇒ ★ escalation ⇒ false-positive 40001s
SELECT max(now()-xact_start) FROM pg_stat_activity WHERE xact_start IS NOT NULL;
-- ⇒ the oldest snapshot; this is what blocks VACUUM
```

**The order to try things:** constraint or atomic statement → `READ ONLY` at a higher level → `FOR UPDATE` → SERIALIZABLE + retries. **Most problems stop at step one.**

---

## When would I use this at work?

1. **Choosing the level for a new write path.** Ask "what invariant am I protecting, and can a constraint hold it?" first. If yes, stay at READ COMMITTED — you get correctness with no retries and no throughput cost.

2. **Diagnosing a 40001 spike.** Distinguish real conflicts from predicate-lock escalation artefacts. One is a design problem; the other is a `max_pred_locks_per_transaction` setting.

3. **Reviewing a proposal to change the global default.** The invoicing example is the standard outcome: it fixes the one bug and introduces retries on every path plus a bloat problem that surfaces two weeks later, when nobody connects it to the change.

4. **Writing any multi-statement report.** `SERIALIZABLE READ ONLY DEFERRABLE` gives a guaranteed-consistent report with no failure mode, and it is nearly free. Most teams have never heard of it.

---

## Connected topics

**Understand before this:** 39 (transactions), 40 (ACID), 43 (the six anomalies — the vocabulary this topic depends on), 24 (constraints — the fix that beats raising the level).

**This unlocks:**
- **45** — locks: `FOR UPDATE`, row locks, and what actually blocks
- **46** — MVCC: snapshots, xmin/xmax, and why readers never block writers
- **47** — VACUUM and bloat: the cost of pinning xmin, in detail
- **48** — deadlocks: the other retryable error, `40P01`
- **49** — optimistic vs pessimistic concurrency in application code
- **50** — SSI: exactly how SERIALIZABLE detects write skew
- **65** — connection pooling: why session-level settings leak
