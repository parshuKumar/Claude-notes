# Topic 53 — Isolation Levels in SQL
### SQL Mastery Curriculum — Phase 8: Transactions and Concurrency in SQL

---

## 1. ELI5 — The Analogy That Makes It Click

Imagine a shared Google Doc that four people are editing at once — a budget spreadsheet for a shared apartment.

**Person A (READ UNCOMMITTED)** doesn't care about anything. He reads whatever pixels are on the screen right now, even mid-keystroke. If someone is halfway through typing "rent = 5000" and has only typed "rent = 5", Person A copies "5" into his notes. Then the typist hits backspace and undoes the whole line. Person A now has garbage — he read something that never officially happened. He read *uncommitted* work.

**Person B (READ COMMITTED)** is more careful. Every time he looks at a cell, he refuses to read half-typed edits — he only reads values that someone has actually saved (committed). But here's the catch: he looks at cell C4, sees "5000", looks away to do a calculation, looks back at C4 a second later, and now it says "6000" because someone saved a change in between. The same cell gave him two different answers during the same task. That's a **non-repeatable read**.

**Person C (REPEATABLE READ)** takes a photograph of the entire spreadsheet the moment she starts her task. For the rest of her task she works only from her photograph. Cell C4 will say "5000" every single time she checks, no matter what anyone else saves, because she's reading her frozen snapshot. Rows don't change under her, and new rows don't appear either. But — she might try to save a change based on her photo that conflicts with reality, and the system may reject her save at the very end.

**Person D (SERIALIZABLE)** wants an ironclad guarantee: whatever the final result is, it must be *as if* everyone took turns editing the document one at a time, in some order — never simultaneously. The system watches every read and write. If two people's overlapping work could only have produced a result that no single-file ordering could ever produce, the system throws one of them out with an error and makes them start over.

That's the whole ladder. Isolation levels are just a dial: how much of *other people's in-progress work* is your transaction allowed to see, and how strong a guarantee do you get that the concurrent chaos is equivalent to a tidy one-at-a-time sequence. Higher isolation = fewer surprises, but more work rejected and retried.

---

## 2. Connection to SQL Internals

Isolation levels are not a lock-heavy feature in PostgreSQL — they are implemented almost entirely through **MVCC (Multi-Version Concurrency Control)**. Understanding isolation *requires* understanding MVCC, because in Postgres "which version of the row do I see?" is the entire game.

Every row in a Postgres heap page carries two hidden system columns:

- **`xmin`** — the transaction ID (XID) that *inserted* (created) this row version.
- **`xmax`** — the transaction ID that *deleted* or *updated* this row version (0/null if still live).

When you `UPDATE` a row, Postgres does **not** overwrite it in place. It writes a *new* row version (a new tuple) with a fresh `xmin`, and stamps the old version's `xmax` with your transaction ID. Both versions physically coexist in the heap until `VACUUM` reclaims the dead one. This is why Postgres tables bloat under heavy update load, and why `VACUUM` exists at all.

The magic that ties isolation to MVCC is the **snapshot**. A snapshot is a small data structure captured at a point in time that answers one question for any row version: *"Was the transaction that created this version committed and visible to me, and was the transaction that deleted it not yet visible to me?"* A snapshot records:

- `xmin` of the snapshot — the lowest still-running XID (everything below is settled).
- `xmax` of the snapshot — one past the highest assigned XID (everything at/above didn't exist yet).
- `xip_list` — the list of XIDs that were **in progress** (running, not committed) at snapshot time.

**The isolation level decides *when* snapshots are taken:**

- **READ COMMITTED** takes a **fresh snapshot at the start of every statement**. That's why each new statement can see newly committed data — non-repeatable reads are possible.
- **REPEATABLE READ** and **SERIALIZABLE** take **one snapshot at the first statement** of the transaction and reuse it for the entire transaction. That's why your view of the data is frozen.

Other internals in play:

- **The WAL (Write-Ahead Log)** records commit records. A snapshot's visibility test ultimately asks the **commit log (pg_xact / CLOG)** whether a given XID committed.
- **Row locks** (`SELECT FOR UPDATE`, Topic 54) and **`xmax`-based locking** handle write-write conflicts that MVCC alone cannot resolve — two transactions cannot both write the same row version.
- **SSI (Serializable Snapshot Isolation)** — Postgres's SERIALIZABLE implementation — layers **predicate locks (SIReadLock)** on top of snapshot isolation to detect *dangerous structures* of read/write dependencies and abort a transaction before an anomaly can commit.

So: isolation level = snapshot timing policy + (for SERIALIZABLE) a dependency-tracking layer. No dirty reads are ever possible in Postgres because a snapshot never considers an uncommitted `xmin` visible — which is the deep reason Postgres's READ UNCOMMITTED behaves identically to READ COMMITTED.

---

## 3. Logical Execution Order Context

Isolation levels do not slot into the single-query `FROM → JOIN → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → window → ORDER BY → LIMIT` pipeline. They operate at a *higher* level — the **transaction** — which wraps one or more statements, each of which internally runs that full pipeline.

The relevant "execution order" for isolation is the **transaction lifecycle**:

```
BEGIN;                          ← transaction starts; XID may be assigned lazily on first write
  SET TRANSACTION ISOLATION ... ← must come BEFORE any query that touches data
  SELECT ...                    ← statement 1
    │  RC:  fresh snapshot taken HERE, at statement start
    │  RR/SER: snapshot taken HERE only if this is the first statement
  UPDATE ...                    ← statement 2
    │  RC:  ANOTHER fresh snapshot taken HERE
    │  RR/SER: reuses the snapshot from statement 1
COMMIT;                         ← RR/SER: serialization failure may be raised HERE
```

Key ordering rules that matter in practice:

1. **`SET TRANSACTION ISOLATION LEVEL` must be the first thing after `BEGIN`**, before any SELECT/INSERT/UPDATE/DELETE. Once a query has run and the transaction's snapshot is anchored, you cannot change the isolation level — Postgres raises an error.

2. **Under RC, the snapshot boundary is the *statement*, not the transaction.** Each statement in a RC transaction re-reads the committed state of the world. Within a single statement, the view is consistent (a long-running SELECT does not see rows committed mid-scan).

3. **Under RR/SER, the snapshot boundary is the *transaction*.** The first data-touching statement freezes the view for everything that follows.

4. **Serialization failures surface at different points**: under RR they typically surface at the moment of a conflicting `UPDATE`/`DELETE` (`could not serialize access due to concurrent update`); under SERIALIZABLE they can surface at `COMMIT` (`could not serialize access due to read/write dependencies`).

This is why isolation is a Phase 8 topic and not a Phase 2 one: it only makes sense once you think in terms of transactions (Topic 52) rather than individual statements.

---

## 4. What Is an Isolation Level?

An **isolation level** is the contract that defines how visible the concurrent, in-flight operations of other transactions are to yours — specifically, which classes of concurrency anomaly (dirty read, non-repeatable read, phantom, serialization anomaly) the database guarantees you will *never* observe. It is the "I" in ACID.

```sql
-- Set for a single transaction (most common form):
BEGIN;
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
│                              │
│                              └── one of: READ UNCOMMITTED
│                                          READ COMMITTED     (Postgres default)
│                                          REPEATABLE READ
│                                          SERIALIZABLE
└── applies to the CURRENT transaction only; must precede any query
  ... your statements ...
COMMIT;
```

```sql
-- Equivalent inline form on BEGIN itself:
BEGIN ISOLATION LEVEL SERIALIZABLE READ WRITE;
│     │                            │
│     │                            └── READ WRITE (default) | READ ONLY
│     └── same four levels
└── combines transaction start + isolation in one statement
```

```sql
-- Set the DEFAULT for every future transaction in this session:
SET SESSION CHARACTERISTICS AS TRANSACTION ISOLATION LEVEL SERIALIZABLE;
│   │                                                     │
│   │                                                     └── the level
│   └── SESSION scope: affects all subsequent transactions in this connection
└── does NOT affect a transaction already in progress
```

```sql
-- Cluster / database-wide default via configuration:
ALTER DATABASE mydb SET default_transaction_isolation = 'repeatable read';
-- or in postgresql.conf:
-- default_transaction_isolation = 'read committed'
```

```sql
-- Inspect the current effective level:
SHOW transaction_isolation;      -- current transaction's level
SHOW default_transaction_isolation;  -- the session/cluster default
```

### The Four Standard Levels (SQL:1992)

The ANSI/ISO SQL standard defines the four levels **by which anomalies they forbid**, not by how they're implemented:

| Level | Dirty Read | Non-Repeatable Read | Phantom Read |
|-------|-----------|---------------------|--------------|
| READ UNCOMMITTED | **Allowed** | Allowed | Allowed |
| READ COMMITTED | Prevented | Allowed | Allowed |
| REPEATABLE READ | Prevented | Prevented | **Allowed** (per standard) |
| SERIALIZABLE | Prevented | Prevented | Prevented |

Crucially, this table is the **standard's minimum requirement**. A database is free to be *stricter*. PostgreSQL is dramatically stricter than the standard demands — which is the single most important practical fact about isolation in Postgres, covered in Section 6.

---

## 5. Why Isolation-Level Mastery Matters in Production

1. **Silent data corruption under concurrency.** The classic bug: two requests read an account balance of $100, each subtracts $30, each writes $70. One $30 withdrawal vanished. This "lost update" is invisible in testing (which is single-threaded) and only appears under production load. Choosing the right isolation level — or the right explicit lock — is the difference between a correct ledger and a slowly corrupting one.

2. **The "it worked in staging" class of Heisenbugs.** Non-repeatable reads and phantoms only manifest when two transactions interleave at exactly the wrong microsecond. They are nearly impossible to reproduce on demand. Understanding the isolation model lets you reason about correctness *statically*, instead of chasing ghosts.

3. **Serialization failures crashing your API.** If you set `SERIALIZABLE` or `REPEATABLE READ` and your application code does **not** implement retry logic, a perfectly normal concurrency conflict raises a `40001` SQLSTATE and your endpoint returns a 500 to the user. Teams flip on SERIALIZABLE for safety, forget the retry loop, and turn a correctness win into an availability regression.

4. **Portability landmines.** Code that relies on Oracle's or MySQL/InnoDB's specific isolation behavior breaks subtly on Postgres, and vice versa. "REPEATABLE READ" means genuinely different things across engines (Postgres's RR forbids phantoms; the standard permits them; MySQL's RR uses next-key locks). A principal engineer must know these differences to write portable, correct data-access code.

5. **Performance vs. correctness tradeoffs at scale.** SERIALIZABLE's SSI machinery holds predicate locks and tracks read/write dependencies, adding overhead and abort rate under contention. Blindly maxing isolation on a hot table can tank throughput. Knowing exactly which anomaly you need to prevent lets you pick the *lowest* level that's still correct — often RC plus a targeted `SELECT FOR UPDATE`.

6. **Financial, inventory, and booking systems are legally/contractually wrong without this.** Overselling the last concert ticket, double-charging a card, letting an account go negative — these are the canonical failures of insufficient isolation, and they have real monetary and reputational cost.

---

## 6. Deep Technical Content

### 6.1 The Anomaly Catalogue — Precise Definitions

Isolation is defined by anomalies. You must be able to describe each one at the transaction-interleaving level.

**Dirty Read (P1)** — T1 reads a row that T2 has modified but *not yet committed*. If T2 then rolls back, T1 has read a value that never existed in the committed database.

```
T2: BEGIN; UPDATE accounts SET balance = 500 WHERE id = 1;   -- not committed
T1:                                        SELECT balance ... WHERE id = 1;  -- reads 500 (DIRTY)
T2: ROLLBACK;                              -- the 500 never existed
-- T1 acted on phantom money
```

**In PostgreSQL a dirty read is impossible at every level** — a snapshot never treats an uncommitted `xmin` as visible.

**Non-Repeatable Read (P2)** — T1 reads a row, T2 updates that row and commits, T1 reads the *same row again* and gets a different value. The same query on the same key returns two answers within one transaction.

```
T1: BEGIN; SELECT balance WHERE id=1;   -- 100
T2:        UPDATE accounts SET balance=150 WHERE id=1; COMMIT;
T1:        SELECT balance WHERE id=1;   -- 150  (NON-REPEATABLE)
```

**Phantom Read (P3)** — T1 runs a query with a search predicate (a range/`WHERE`) and gets a set of rows. T2 inserts (or updates into range) a new row matching that predicate and commits. T1 re-runs the *same query* and now a new "phantom" row appears.

```
T1: BEGIN; SELECT count(*) FROM orders WHERE total > 1000;   -- 5
T2:        INSERT INTO orders(total) VALUES (2000); COMMIT;
T1:        SELECT count(*) FROM orders WHERE total > 1000;   -- 6  (PHANTOM)
```

The difference between non-repeatable and phantom: non-repeatable is about an *existing row changing*; phantom is about the *set of rows matching a predicate changing* (rows appearing or disappearing).

**Lost Update (P4)** — Two transactions read the same row, both compute a new value from what they read, both write. The second write silently overwrites the first, and the first update is lost. This is the read-modify-write hazard.

```
T1: BEGIN; SELECT balance WHERE id=1;   -- 100
T2: BEGIN; SELECT balance WHERE id=1;   -- 100
T1:        UPDATE balance=100-30=70; COMMIT;   -- 70
T2:        UPDATE balance=100-30=70; COMMIT;   -- 70  (should be 40! one -30 LOST)
```

**Write Skew (A5B)** — The subtle one, and the reason SERIALIZABLE exists. Two transactions each read an overlapping data set, each check a constraint that currently holds, then each write to *different* rows. Individually each write is fine; together they violate an invariant that spans both rows. No single row is updated by both, so lost-update protection and row locks don't catch it.

```
-- Invariant: at least ONE doctor must remain on-call.
-- Currently Alice and Bob are both on-call.
T1 (Alice wants off): SELECT count(*) FROM doctors WHERE on_call;  -- 2, ok to leave
T2 (Bob wants off):   SELECT count(*) FROM doctors WHERE on_call;  -- 2, ok to leave
T1: UPDATE doctors SET on_call=false WHERE name='Alice'; COMMIT;
T2: UPDATE doctors SET on_call=false WHERE name='Bob';   COMMIT;
-- Result: ZERO doctors on-call. Invariant violated. Neither saw the other's write.
```

Write skew is **not** prevented by REPEATABLE READ (snapshot isolation). It is prevented by SERIALIZABLE.

### 6.2 Level 0 — READ UNCOMMITTED in PostgreSQL

The SQL standard says READ UNCOMMITTED permits dirty reads. **PostgreSQL does not implement dirty reads at all.** Requesting `READ UNCOMMITTED` is accepted syntactically but silently promoted to `READ COMMITTED` behavior.

```sql
BEGIN;
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
SHOW transaction_isolation;   -- returns 'read uncommitted' as the label...
-- ...but behaviorally it is READ COMMITTED. No dirty read is ever observable.
COMMIT;
```

Why? Because MVCC's visibility rule has no mechanism to expose an uncommitted tuple version — the snapshot machinery fundamentally excludes in-progress XIDs. There is no code path in Postgres that returns a dirty row. This is a deliberate strength: the *weakest* isolation Postgres offers is already stronger than the standard's minimum for RC.

**Interview gold:** "In PostgreSQL, how many *distinct* isolation behaviors are there?" — **Three**, not four. RU and RC are identical; RR and SERIALIZABLE are distinct. (RU/RC, RR, SER.)

### 6.3 Level 1 — READ COMMITTED (the default)

READ COMMITTED is Postgres's default (`default_transaction_isolation = 'read committed'`). Its defining behavior: **a new snapshot is taken at the start of each statement.**

Consequences:
- **No dirty reads** — you only see committed data.
- **Non-repeatable reads ARE possible** — two SELECTs in the same transaction can see different data if another transaction commits between them.
- **Phantoms ARE possible** — a range query re-run can show new rows.
- **Statement-level consistency** — a single statement sees a consistent snapshot; a 10-minute `SELECT sum(...)` does not see rows committed while it scans.

The subtle and critical part of RC is how **`UPDATE`/`DELETE`/`SELECT FOR UPDATE` handle a row that was modified by a concurrent transaction after the statement's snapshot.** This is called **EPQ (EvalPlanQual) re-checking**:

```
T1 (RC): UPDATE accounts SET balance = balance - 30 WHERE id = 1;
   -- T1's snapshot sees balance = 100
   -- but T2 concurrently committed balance = 80 for id=1
```

When T1's UPDATE reaches the row and finds that a *committed* concurrent transaction changed it, RC does **not** fail. Instead Postgres:
1. Waits for the concurrent writer to commit/abort (a row lock block).
2. Re-fetches the *latest committed* version of the row (80, not the snapshot's 100).
3. **Re-evaluates the `WHERE` clause** against that latest version. If it still matches, the UPDATE proceeds against the new version.
4. Re-computes `balance - 30` from the *new* value → 80 - 30 = 50.

This is why `UPDATE accounts SET balance = balance - 30` is safe from lost updates under RC — the arithmetic re-reads the fresh value. But `UPDATE ... SET balance = 70` (where 70 was computed in your app from a stale SELECT of 100) is **not** safe — the app's read-modify-write cycle still loses updates. RC protects in-database arithmetic, not application-level read-then-write.

A trap of EPQ re-checking: the re-evaluated `WHERE` runs against the *new* row version, so a RC `UPDATE ... WHERE status='pending'` might skip a row that a concurrent txn just moved out of 'pending' — even though your snapshot saw it as pending. This can make RC updates surprising under contention.

### 6.4 Level 2 — REPEATABLE READ (Snapshot Isolation)

REPEATABLE READ in Postgres is **true snapshot isolation**. One snapshot is taken at the first statement and used for the entire transaction.

Behavior:
- **No dirty reads, no non-repeatable reads** — the whole transaction sees one frozen view.
- **No phantoms either** — this is where Postgres exceeds the SQL standard. Because the snapshot is transaction-wide and covers *all* rows (including ones that don't exist yet from your snapshot's perspective), a re-run range query returns exactly the same rows. The standard *permits* phantoms at RR; Postgres *forbids* them.
- **Write skew IS still possible** — snapshot isolation's known gap. Two transactions on disjoint rows can each pass a check and jointly violate an invariant.
- **Lost updates are turned into errors, not silently allowed.** If a RR transaction tries to `UPDATE` a row that a concurrent transaction has *already committed a change to since the snapshot*, Postgres does **not** do EPQ re-checking (that's RC behavior). Instead it aborts:

```
ERROR:  could not serialize access due to concurrent update
SQLSTATE: 40001
```

This is the **first-committer-wins** rule. Whoever commits the write first succeeds; the loser is told to retry. This makes RR safe against lost updates *provided you retry on 40001*.

```sql
BEGIN;
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SELECT balance FROM accounts WHERE id = 1;      -- snapshot frozen here: 100
-- meanwhile another txn commits balance = 80
UPDATE accounts SET balance = balance - 30 WHERE id = 1;
-- ERROR: could not serialize access due to concurrent update
-- (the row was updated by a txn that committed after our snapshot)
COMMIT;  -- never reached
```

Compare with RC: the same sequence under RC would *succeed* via EPQ re-check (50). Under RR it *errors*. This is the fundamental behavioral fork between the two levels.

### 6.5 Level 3 — SERIALIZABLE (SSI)

SERIALIZABLE guarantees that the concurrent execution of transactions produces a result identical to *some* serial (one-at-a-time) execution of them. It prevents **every** anomaly, including write skew and read-only anomalies that snapshot isolation misses.

Postgres implements this via **SSI (Serializable Snapshot Isolation)**, introduced in 9.1. SSI starts from the same snapshot isolation as RR, then **monitors read/write dependencies between concurrent transactions** to detect "dangerous structures" — specifically a pattern of two consecutive **rw-antidependencies** (a "pivot") that indicates a possible non-serializable outcome. When detected, one transaction is aborted:

```
ERROR:  could not serialize access due to read/write dependencies among transactions
SQLSTATE: 40001
HINT:  The transaction might succeed if retried.
```

How SSI tracks reads: it takes **predicate locks** (called **SIReadLocks**), which mark *what data a transaction read* — not to block anyone (SIReadLocks never block), but to *detect* when another transaction writes into a range that a serializable transaction read. These locks can be at row, page, or relation granularity; under memory pressure they escalate coarser (page → relation), increasing false-positive aborts.

Key properties:
- SSI **never blocks reads** — no shared read locks. Readers proceed at full snapshot-isolation speed; conflicts are *detected*, not *prevented by waiting*.
- The cost is a possible **abort at any point, including COMMIT**, when a dangerous cycle is found.
- It even prevents the **read-only serialization anomaly** — a scenario where a read-only transaction observing snapshot state can expose a non-serializable outcome. (You can opt a read-only txn into `SERIALIZABLE READ ONLY DEFERRABLE` to wait for a safe snapshot and never abort.)

The write-skew doctor example from 6.1: under SERIALIZABLE, when Bob's transaction tries to commit after Alice's, SSI sees that each transaction *read* the on-call set and *wrote* into it in a way that forms a dangerous cycle, and aborts one of them with 40001. Retried serially, the second doctor's transaction now reads count=1 and correctly refuses to go off-call.

### 6.6 The PostgreSQL-Specific Truth Table

| Level (requested) | Actual behavior in PG | Dirty | Non-repeat | Phantom | Lost update | Write skew |
|-------------------|-----------------------|-------|-----------|---------|-------------|-----------|
| READ UNCOMMITTED | = READ COMMITTED | No | Yes | Yes | Yes (app-level) | Yes |
| READ COMMITTED | statement snapshots + EPQ | No | Yes | Yes | Yes (app-level)* | Yes |
| REPEATABLE READ | snapshot isolation | No | No | **No** (stricter than std) | No (errors→retry) | **Yes** |
| SERIALIZABLE | SSI | No | No | No | No | **No** |

\*RC prevents lost updates for pure in-DB arithmetic (`SET x = x - 1`) via EPQ, but not for application read-then-write cycles.

The two facts that trip up even senior engineers:
1. **PG's REPEATABLE READ prevents phantoms** (the standard doesn't require this).
2. **PG's REPEATABLE READ does NOT prevent write skew** — you need SERIALIZABLE for that.

### 6.7 How the Levels Handle a Concurrent UPDATE — Side by Side

```sql
-- Setup: accounts(id=1, balance=100)
-- Two concurrent transactions both do a read-modify-write in the app layer.

-- READ COMMITTED, in-DB arithmetic → SAFE (EPQ re-reads)
UPDATE accounts SET balance = balance - 30 WHERE id = 1;   -- always correct

-- READ COMMITTED, app-computed value → LOST UPDATE possible
--   app read 100, computes 70, both write 70

-- REPEATABLE READ, either style → second writer gets 40001, must retry
--   after retry the new snapshot sees the committed value → correct

-- SERIALIZABLE → same 40001 protection + write-skew protection
```

### 6.8 Subtransactions, SAVEPOINT, and Snapshot Timing

Under RR/SERIALIZABLE the transaction snapshot is set at the **first query that acquires a snapshot** — importantly, `BEGIN` alone does *not* take the snapshot (an XID and snapshot are acquired lazily). If you want to guarantee your snapshot reflects the state at a precise moment, run a trivial query (e.g., `SELECT 1` won't anchor data visibility meaningfully — you need a real data-touching statement) or rely on the first real statement. `SAVEPOINT`/`ROLLBACK TO SAVEPOINT` do not reset the transaction snapshot under RR/SER.

### 6.9 `READ ONLY`, `DEFERRABLE`, and Long Reporting Queries

```sql
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE READ ONLY DEFERRABLE;
-- Guarantees a serializable, consistent snapshot for a big report
-- WITHOUT any risk of a 40001 abort. Postgres may briefly WAIT at the
-- start until it can hand out a snapshot that cannot participate in a
-- dangerous structure. Ideal for nightly analytics that must be consistent.
```

`READ ONLY DEFERRABLE` is the correct tool for long consistent reports: full serializable consistency, zero abort risk, at the cost of a possible short wait for a safe snapshot.

### 6.10 Interaction with Explicit Locks

Isolation levels and explicit locks (`SELECT FOR UPDATE`, `LOCK TABLE`, advisory locks — Topic 54) are complementary. A common and correct production pattern is **READ COMMITTED + `SELECT FOR UPDATE`**: stay at the cheap default level but take an explicit row lock exactly where you need serialize-like protection for a specific read-modify-write, avoiding both lost updates and the broad overhead/abort rate of SERIALIZABLE. SSI (SERIALIZABLE) is the alternative when the invariant spans rows/predicates that a single `FOR UPDATE` cannot lock (write skew).

---

## 7. EXPLAIN — Isolation in the Plan

Isolation level does **not** appear in an `EXPLAIN` plan — it is a runtime transaction property, not a plan-shape decision. The planner produces the same node tree regardless of isolation level; what differs is the *snapshot* used at execution and the *conflict detection* at commit. This is an important conceptual point: you cannot "see" isolation in EXPLAIN the way you see a Hash Join.

What you *can* observe are the runtime artifacts of isolation via system views and error behavior. Below is a realistic session demonstrating how to *detect* what isolation is doing.

### 7.1 The plan is identical across levels

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT balance FROM accounts WHERE id = 1;
```

```
Index Scan using accounts_pkey on accounts  (cost=0.29..8.31 rows=1 width=8)
                                            (actual time=0.018..0.020 rows=1 loops=1)
  Index Cond: (id = 1)
  Buffers: shared hit=3
Planning Time: 0.09 ms
Execution Time: 0.041 ms
```

This plan is byte-for-byte the same whether the surrounding transaction is READ COMMITTED or SERIALIZABLE. The difference is invisible to the planner: under SERIALIZABLE this Index Scan *also* registers an `SIReadLock` predicate lock on the rows/pages touched — but that is a side effect of execution, not a plan node.

### 7.2 Observing predicate locks (SSI) via pg_locks

```sql
-- In a SERIALIZABLE transaction that has run a SELECT, from another session:
SELECT locktype, relation::regclass, mode, granted
FROM pg_locks
WHERE mode = 'SIReadLock';
```

```
 locktype |  relation  |    mode     | granted
----------+------------+-------------+---------
 tuple    | accounts   | SIReadLock  | t
 page     | accounts   | SIReadLock  | t
```

The `SIReadLock` rows are the predicate locks SSI uses to detect rw-conflicts. Note `granted = t` always — SIReadLocks never wait; they only record. Seeing `page`-level SIReadLocks where you expected `tuple`-level tells you lock escalation occurred (memory pressure), which raises false-positive abort rates.

### 7.3 Observing a serialization failure

```sql
-- Session A and B both REPEATABLE READ, both update id=1
-- Session A commits first; Session B's UPDATE:
UPDATE accounts SET balance = balance - 30 WHERE id = 1;
```

```
ERROR:  could not serialize access due to concurrent update
```

For SERIALIZABLE write-skew, the failure instead reads:

```
ERROR:  could not serialize access due to read/write dependencies among transactions
DETAIL:  Reason code: Canceled on identification as a pivot, during commit attempt.
HINT:  The transaction might succeed if retried.
```

The `DETAIL` reason codes ("pivot", "conflict out", "conflict in") are SSI internals telling you *why* the dangerous structure was flagged — useful when tuning abort rates.

### 7.4 Detecting long-running snapshots that block VACUUM

```sql
-- A forgotten RR/SER transaction holds an old snapshot, pinning dead tuples
SELECT pid, state, xact_start, now() - xact_start AS age,
       backend_xmin
FROM pg_stat_activity
WHERE state = 'idle in transaction'
ORDER BY xact_start;
```

```
  pid  |        state        |         xact_start         |     age      | backend_xmin
-------+---------------------+----------------------------+--------------+--------------
 20481 | idle in transaction | 2026-07-19 02:10:11.32+00  | 00:47:03     |     91827364
```

A RR/SER transaction that stays open holds its snapshot's `xmin`, preventing `VACUUM` from removing dead tuples newer than it — the leading cause of table bloat from long transactions. `backend_xmin` is the horizon it's pinning.

---

## 8. Query Examples

### Example 1 — Basic: Setting and Verifying the Level

```sql
-- Explicitly run a transaction at REPEATABLE READ and confirm the snapshot is frozen
BEGIN;
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;

SHOW transaction_isolation;          -- 'repeatable read'

SELECT balance FROM accounts WHERE id = 1;   -- e.g. 100; snapshot anchored here

-- (Imagine another session commits balance = 999 for id=1 right now)

SELECT balance FROM accounts WHERE id = 1;   -- STILL 100 — repeatable
COMMIT;

-- After COMMIT, a fresh SELECT would see 999.
```

### Example 2 — Intermediate: Safe Counter Increment Under Contention

```sql
-- A page-view counter hammered by many concurrent requests.
-- Goal: never lose an increment.

-- Option A — READ COMMITTED with in-DB arithmetic (simplest, correct):
BEGIN;  -- default READ COMMITTED
UPDATE page_stats
SET views = views + 1                 -- arithmetic re-read via EPQ on conflict
WHERE page_id = 42;
COMMIT;
-- Safe: even under 10k concurrent increments none are lost, because each
-- UPDATE re-reads the latest committed `views` when it hits a locked row.

-- Option B — REPEATABLE READ (needs retry on 40001):
BEGIN;
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
UPDATE page_stats SET views = views + 1 WHERE page_id = 42;
COMMIT;   -- may raise 40001 under contention → application must retry
-- Correct but strictly worse here: same guarantee, adds abort/retry cost.
```

The lesson: for a single-row read-modify-write, RC + in-DB arithmetic is both simpler and cheaper than RR. Reach for higher isolation only when the invariant spans multiple rows.

### Example 3 — Production Grade: Enforcing a Cross-Row Invariant (Write-Skew Safe)

```sql
-- Table: inventory(product_id PK, reserved int, capacity int)
-- Table: reservations(id, product_id, qty, created_at)
-- INVARIANT: SUM(active reservation qty) for a product must never exceed capacity.
-- This spans MANY rows (all reservations of a product), so a single FOR UPDATE
-- on one row cannot protect it. Write skew is the exact hazard. Use SERIALIZABLE.
--
-- Table sizes: reservations ~40M rows, inventory ~200K rows.
-- Index: reservations(product_id) btree; inventory PK on product_id.
-- Perf expectation: each attempt is 2 index scans + 1 insert (~1-2ms); under
-- heavy contention on a single hot product, expect a modest 40001 abort rate
-- that the retry loop absorbs. Throughput bound by the hottest product, not the table.

BEGIN;
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- Read the current committed total (registers SIReadLock predicate locks)
WITH used AS (
  SELECT COALESCE(SUM(qty), 0) AS total
  FROM reservations
  WHERE product_id = $1 AND status = 'active'
)
SELECT (SELECT capacity FROM inventory WHERE product_id = $1) - used.total AS remaining
FROM used;
-- Application checks: remaining >= requested_qty ? proceed : reject

INSERT INTO reservations(product_id, qty, status, created_at)
VALUES ($1, $2, 'active', now());

COMMIT;
-- If two requests concurrently read the same "remaining" and both insert such
-- that the sum would exceed capacity, SSI detects the rw-dependency cycle and
-- aborts one with 40001. The retry re-reads the now-smaller remaining and
-- correctly rejects. No overselling is possible.
```

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT COALESCE(SUM(qty),0) FROM reservations WHERE product_id = 12345 AND status='active';
```

```
Aggregate  (cost=412.55..412.56 rows=1 width=8) (actual time=1.83..1.83 rows=1 loops=1)
  Buffers: shared hit=118
  ->  Index Scan using reservations_product_id_idx on reservations
        (cost=0.56..409.90 rows=1060 width=4) (actual time=0.03..1.61 rows=1043 loops=1)
        Index Cond: (product_id = 12345)
        Filter: (status = 'active')
        Rows Removed by Filter: 214
        Buffers: shared hit=118
Planning Time: 0.14 ms
Execution Time: 1.88 ms
```

Reading it: the aggregate reads ~1,257 index rows for the hot product, filters to 1,043 active. Under SERIALIZABLE these touched tuples/pages get `SIReadLock` predicate locks so that any concurrent insert into this product's reservation set is detected as a conflict. Cost is dominated by the index scan, not by isolation — SSI's overhead here is the bookkeeping, not extra I/O.

---

## 9. Wrong → Right Patterns

### Wrong 1: Application-Level Read-Modify-Write Under READ COMMITTED

```sql
-- WRONG: classic lost update. Two requests interleave.
-- Request A                          Request B
BEGIN;                                BEGIN;
SELECT balance FROM accounts          SELECT balance FROM accounts
  WHERE id=1;   -- reads 100            WHERE id=1;   -- reads 100
-- app computes 100 - 30 = 70         -- app computes 100 - 20 = 80
UPDATE accounts SET balance = 70      UPDATE accounts SET balance = 80
  WHERE id=1;                           WHERE id=1;   -- blocks until A commits
COMMIT;  -- balance = 70              COMMIT;  -- balance = 80  ← A's -30 is LOST
```

**Why it's wrong at the execution level:** Under RC each transaction took a snapshot showing 100. Because the app supplied a *literal* final value (70, 80) rather than in-DB arithmetic, EPQ re-checking has nothing to recompute — it just overwrites. B's `SET balance = 80` clobbers A's committed 70. The correct final balance was 50 (100−30−20). One update vanished silently.

```sql
-- RIGHT (Option 1): let the database do the arithmetic — EPQ makes it safe
UPDATE accounts SET balance = balance - 30 WHERE id = 1;
UPDATE accounts SET balance = balance - 20 WHERE id = 1;
-- Each UPDATE re-reads the latest committed balance on conflict. Final = 50. Correct.

-- RIGHT (Option 2): lock the row explicitly for the read-modify-write
BEGIN;
SELECT balance FROM accounts WHERE id = 1 FOR UPDATE;  -- serializes on this row
-- app computes new value from the locked, current balance
UPDATE accounts SET balance = $new WHERE id = 1;
COMMIT;

-- RIGHT (Option 3): REPEATABLE READ + retry loop
-- the second committer gets 40001, retries, re-reads fresh, recomputes. Final = 50.
```

### Wrong 2: Assuming REPEATABLE READ Prevents Write Skew

```sql
-- WRONG: using RR to protect a cross-row invariant.
-- Invariant: at least one admin must remain active.
BEGIN;
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SELECT count(*) FROM users WHERE role='admin' AND active;   -- 2, safe to deactivate one
UPDATE users SET active=false WHERE id = $me;   -- deactivate myself
COMMIT;
-- Two admins run this concurrently. Each reads count=2 from its OWN snapshot.
-- Each updates a DIFFERENT row. No row is written by both → no 40001 under RR.
-- Both commit. Zero admins remain. INVARIANT VIOLATED.
```

**Why it's wrong at the execution level:** RR is snapshot isolation. Its conflict detection (`could not serialize access due to concurrent update`) only fires when two transactions write the *same* row. Here they write *different* rows, so RR sees no conflict. Snapshot isolation provably cannot prevent write skew — this is its defining theoretical gap.

```sql
-- RIGHT: SERIALIZABLE — SSI tracks the read of the admin set + the write into it
BEGIN;
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
SELECT count(*) FROM users WHERE role='admin' AND active;   -- SIReadLock on the set
UPDATE users SET active=false WHERE id = $me;
COMMIT;
-- The second committer forms a dangerous rw-dependency structure → 40001.
-- On retry it reads count=1 and its app logic refuses to deactivate. Invariant holds.

-- RIGHT (alternative without SERIALIZABLE): materialize the invariant as a lockable row
-- e.g. a single admin_count row you SELECT ... FOR UPDATE, or a CHECK via a
-- constraint trigger. Turns the predicate into a concrete lock target.
```

### Wrong 3: SERIALIZABLE Without a Retry Loop

```sql
-- WRONG: turning on SERIALIZABLE and treating 40001 as a fatal error
app.post('/reserve', async (req) => {
  await db.query('BEGIN ISOLATION LEVEL SERIALIZABLE');
  // ... reads and writes ...
  await db.query('COMMIT');   // may throw 40001 → uncaught → HTTP 500 to user
});
```

**Why it's wrong:** 40001 is not an error condition — it is a *normal, expected* signal meaning "retry me." Under SERIALIZABLE (and RR) conflicts are the mechanism by which correctness is enforced. An endpoint without retry logic converts every concurrency conflict into a user-facing 500, trading a silent-correctness problem for an availability problem.

```sql
-- RIGHT: retry the whole transaction on serialization failure (SQLSTATE 40001)
async function withSerializableRetry(fn, maxAttempts = 5) {
  for (let attempt = 1; ; attempt++) {
    const client = await pool.connect();
    try {
      await client.query('BEGIN ISOLATION LEVEL SERIALIZABLE');
      const result = await fn(client);
      await client.query('COMMIT');
      return result;
    } catch (err) {
      await client.query('ROLLBACK');
      if (err.code === '40001' && attempt < maxAttempts) {
        await sleep(2 ** attempt * 5 + Math.random() * 10);  // backoff + jitter
        continue;                                            // retry the whole txn
      }
      throw err;
    } finally {
      client.release();
    }
  }
}
```

### Wrong 4: Changing Isolation Level After a Query Has Run

```sql
-- WRONG: trying to raise isolation mid-transaction
BEGIN;
SELECT * FROM accounts WHERE id = 1;   -- snapshot already anchored
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
-- ERROR: SET TRANSACTION ISOLATION LEVEL must be called before any query
```

**Why it's wrong:** Once a data-touching statement runs, the transaction's snapshot/visibility semantics are fixed. Postgres forbids changing the level afterward.

```sql
-- RIGHT: set it first, before touching any data
BEGIN;
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;   -- first thing
SELECT * FROM accounts WHERE id = 1;
COMMIT;
-- or the inline form:
BEGIN ISOLATION LEVEL SERIALIZABLE;
```

### Wrong 5: Long-Running REPEATABLE READ Transaction Bloating Tables

```sql
-- WRONG: opening a RR txn for a long report, then doing slow app work
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT * FROM huge_table;      -- snapshot frozen
-- ... 45 minutes of app-side processing, network calls, user think-time ...
COMMIT;
-- The whole time, this txn's xmin pins the dead-tuple horizon. VACUUM cannot
-- reclaim any row version newer than this snapshot across the ENTIRE database.
-- Result: table bloat, index bloat, degraded performance for everyone.
```

**Why it's wrong at the execution level:** An open RR/SER transaction holds `backend_xmin`. `VACUUM` may not remove tuples that could still be visible to *some* running snapshot, so it stalls at this transaction's horizon. Minutes of an idle-in-transaction RR session can cause gigabytes of bloat on a busy table.

```sql
-- RIGHT: for long consistent reports, use READ ONLY DEFERRABLE and keep it short,
-- or snapshot-export, or run against a replica.
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE READ ONLY DEFERRABLE;
SELECT ... ;   -- do the read, then COMMIT promptly; do app work AFTER commit
COMMIT;
-- And: never hold a transaction open across user think-time or external I/O.
```

---

## 10. Performance Profile

### 10.1 Relative Cost of Each Level

| Level | Read overhead | Write overhead | Abort risk | Memory |
|-------|--------------|----------------|-----------|--------|
| READ COMMITTED | Snapshot per statement (cheap) | EPQ re-check on conflict (cheap) | None (waits, never aborts on RMW) | O(1) per snapshot |
| REPEATABLE READ | One snapshot per txn (cheapest reads) | First-committer-wins → 40001 possible | Moderate under write contention | O(1) per snapshot |
| SERIALIZABLE | One snapshot + **predicate-lock tracking** | rw-dependency tracking + possible abort | Highest; depends on read/write overlap | O(reads) for SIReadLocks; escalates under pressure |

Key numbers and behaviors at scale:

- **Snapshot acquisition** is cheap (microseconds) but *not free*. Under RC, every statement grabs a snapshot; a workload of millions of tiny statements pays this per statement. RR/SER pay it once per transaction — a slight read advantage for multi-statement transactions.
- **SSI predicate locks (SIReadLocks)** consume shared memory sized by `max_pred_locks_per_transaction × max_connections`. A serializable transaction that scans millions of rows can exhaust the per-transaction predicate-lock budget, triggering **granularity escalation** (tuple → page → relation). Escalation is cheaper in memory but causes **more false-positive 40001 aborts** because a whole page/table now conflicts.
- **Abort/retry amplification:** at SERIALIZABLE under high contention on a hot key, the effective throughput can *drop* as retries pile up. If 30% of transactions abort and retry, you're doing ~1.3–2× the work. This is why SERIALIZABLE shines for correctness-critical, moderate-contention workloads and struggles on a single ultra-hot row (use an explicit lock or a queue there instead).

### 10.2 Scaling by Row Count

| Rows scanned by a SERIALIZABLE txn | Predicate-lock behavior | Consequence |
|-----------------------------------|-------------------------|-------------|
| 1K–10K | Fine-grained tuple/page SIReadLocks | Precise conflict detection, low false aborts |
| ~100K–1M | Approaching per-txn predicate budget | Escalation to page/relation begins |
| 10M+ | Relation-level SIReadLock likely | Any concurrent write to the table conflicts → high abort rate |

**Rule of thumb:** SERIALIZABLE is a poor fit for transactions that *read* huge row ranges, because their predicate footprint becomes the whole table. Keep serializable transactions *narrow* (touch few rows). Big consistent reads belong in `READ ONLY DEFERRABLE` or on a replica.

### 10.3 Tuning Knobs

```sql
-- Predicate lock memory (raise if you see "out of shared memory" from SSI, or
-- to delay escalation on large-scan serializable txns):
-- max_pred_locks_per_transaction (default 64)  -- postgresql.conf, needs restart
-- max_pred_locks_per_relation, max_pred_locks_per_page  -- escalation thresholds

-- Reduce lost-update/serialization contention by keeping txns short and hot-row
-- writes serialized via advisory locks or a dedicated queue table.

-- Monitor abort rate:
SELECT datname, xact_commit, xact_rollback,
       round(100.0 * xact_rollback / NULLIF(xact_commit + xact_rollback,0), 2) AS rollback_pct
FROM pg_stat_database WHERE datname = current_database();
```

### 10.4 Index Interactions

- **RC / RR / SER read the same indexes** — isolation does not change plan shape (Section 7).
- Under **SERIALIZABLE**, an **Index Scan predicate-locks fewer rows than a Seq Scan** of the same query — the index lets SSI lock a precise range rather than the whole relation. **So a good index doesn't just speed the query; under SERIALIZABLE it directly *lowers the false-positive abort rate*** by keeping predicate locks fine-grained. This is a subtle, high-value insight: indexing is a *correctness-throughput* lever under SSI, not only a latency lever.

---

## 11. Node.js Integration

### 11.1 Setting the Level Per Transaction with `pg`

```javascript
import pg from 'pg';
const { Pool } = pg;
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

// A transaction pins ONE connection for its whole life — never interleave
// statements from the same logical txn across pooled connections.
async function transferFunds(fromId, toId, amount) {
  const client = await pool.connect();
  try {
    await client.query('BEGIN ISOLATION LEVEL REPEATABLE READ');
    // in-DB arithmetic keeps it safe; RR turns concurrent conflicts into 40001
    await client.query(
      'UPDATE accounts SET balance = balance - $1 WHERE id = $2',
      [amount, fromId]
    );
    await client.query(
      'UPDATE accounts SET balance = balance + $1 WHERE id = $2',
      [amount, toId]
    );
    await client.query('COMMIT');
  } catch (err) {
    await client.query('ROLLBACK');
    throw err;
  } finally {
    client.release();
  }
}
```

### 11.2 The Serialization-Failure Retry Wrapper (production essential)

```javascript
const SERIALIZATION_FAILURE = '40001';
const DEADLOCK_DETECTED = '40P01';   // deadlocks are also retryable

async function runSerializable(work, { maxAttempts = 5 } = {}) {
  for (let attempt = 1; ; attempt++) {
    const client = await pool.connect();
    try {
      await client.query('BEGIN ISOLATION LEVEL SERIALIZABLE');
      const out = await work(client);          // caller runs queries on `client`
      await client.query('COMMIT');
      return out;
    } catch (err) {
      await client.query('ROLLBACK').catch(() => {});
      const retryable = err.code === SERIALIZATION_FAILURE || err.code === DEADLOCK_DETECTED;
      if (retryable && attempt < maxAttempts) {
        const backoff = Math.min(2 ** attempt * 10, 500) + Math.random() * 25;
        await new Promise(r => setTimeout(r, backoff));   // exponential backoff + jitter
        continue;
      }
      throw err;   // non-retryable, or out of attempts
    } finally {
      client.release();
    }
  }
}

// Usage: overselling-safe reservation
await runSerializable(async (client) => {
  const { rows } = await client.query(
    `SELECT (SELECT capacity FROM inventory WHERE product_id = $1)
            - COALESCE(SUM(qty),0) AS remaining
       FROM reservations WHERE product_id = $1 AND status = 'active'`,
    [productId]
  );
  if (rows[0].remaining < qty) throw new Error('SOLD_OUT');   // not a 40001 → not retried
  await client.query(
    `INSERT INTO reservations(product_id, qty, status) VALUES ($1,$2,'active')`,
    [productId, qty]
  );
});
```

Two subtleties every backend engineer must internalize:

1. **The retry wraps the *entire* transaction**, re-running all reads and writes — because the whole point is to re-read fresh data under a new snapshot. Retrying only the failed statement is wrong.
2. **Distinguish business errors from serialization errors.** `throw new Error('SOLD_OUT')` must *not* be retried (it will never succeed); only `40001`/`40P01` should loop. Check `err.code`, never message strings.

### 11.3 Does the pg driver handle any of this?

The `pg` driver is a thin protocol client — it does **not** manage transactions, isolation, or retries for you. It faithfully sends `BEGIN ISOLATION LEVEL ...` and surfaces the Postgres `err.code`. All transaction scoping (pinning one `client`), level selection, and 40001 retry logic is *your* responsibility. This is a feature: no hidden magic, full control.

---

## 12. ORM Comparison

The question for isolation levels is not "can the ORM express a JOIN" but two others: **(1) can I set the isolation level for a transaction, and (2) does the ORM retry serialization failures (`40001`) for me, or must I?** Almost universally the answer to (2) is *you must* — this is the single most important thing to internalize.

### Prisma

**Can Prisma set isolation levels?** — Yes, via the `isolationLevel` option on `$transaction`.

```typescript
import { PrismaClient, Prisma } from '@prisma/client';
const prisma = new PrismaClient();

// Interactive transaction pinned to Serializable
await prisma.$transaction(
  async (tx) => {
    const seat = await tx.session.findUnique({ where: { id: seatId } });
    if (seat.status !== 'free') throw new Error('TAKEN');   // business error — do NOT retry
    await tx.session.update({ where: { id: seatId }, data: { status: 'held', userId } });
  },
  { isolationLevel: Prisma.TransactionIsolationLevel.Serializable, timeout: 5000 }
);
```

Available levels: `ReadUncommitted`, `ReadCommitted` (Prisma's default matches the DB default = RC on Postgres), `RepeatableRead`, `Serializable`.

**Where it breaks**: Prisma does **not** retry `40001`. A serialization failure surfaces as a `PrismaClientKnownRequestError` with code `P2034` ("Transaction failed due to a write conflict or a deadlock"). You must wrap `$transaction` in your own retry loop keyed on `P2034`. Forgetting this is the number-one Prisma-under-Serializable production bug.

**Verdict**: Level selection is clean. Retry is entirely on you — build a `withRetry(() => prisma.$transaction(...))` helper.

---

### Drizzle ORM

**Can Drizzle set isolation levels?** — Yes, via the transaction config (Postgres driver).

```typescript
import { db } from './db';
import { sessions } from './schema';
import { eq } from 'drizzle-orm';

await db.transaction(
  async (tx) => {
    const [seat] = await tx.select().from(sessions).where(eq(sessions.id, seatId));
    if (seat.status !== 'free') throw new Error('TAKEN');
    await tx.update(sessions).set({ status: 'held', userId }).where(eq(sessions.id, seatId));
  },
  { isolationLevel: 'serializable', accessMode: 'read write', deferrable: false }
);
```

Drizzle also exposes `accessMode: 'read only'` (enables the `READ ONLY` optimization — read-only serializable txns are cheaper for SSI) and `deferrable` (pairs with read-only serializable to wait for a safe snapshot instead of ever failing).

**Where it breaks**: Like everyone else, Drizzle does **not** retry serialization failures. The underlying driver (`postgres`/`pg`) throws with `err.code === '40001'`; you own the loop.

**Verdict**: Best level control of the bunch — it exposes `READ ONLY DEFERRABLE`, which most ORMs hide. Retry still manual.

---

### Sequelize

**Can Sequelize set isolation levels?** — Yes, via the `isolationLevel` option on `sequelize.transaction`.

```javascript
const { Transaction } = require('sequelize');

await sequelize.transaction(
  { isolationLevel: Transaction.ISOLATION_LEVELS.SERIALIZABLE },
  async (t) => {
    const seat = await Session.findByPk(seatId, { transaction: t });
    if (seat.status !== 'free') throw new Error('TAKEN');
    await seat.update({ status: 'held', userId }, { transaction: t });
  }
);
```

Levels: `READ_UNCOMMITTED`, `READ_COMMITTED`, `REPEATABLE_READ`, `SERIALIZABLE`.

**Where it breaks**: Two traps. (1) Sequelize does **not** retry `40001`. (2) A subtle footgun — you must thread `{ transaction: t }` into *every* query, or that query runs **outside** the transaction on a different connection and sees a different snapshot entirely, silently defeating your isolation level. This is the classic Sequelize isolation bug.

**Verdict**: Works, but the "forgot to pass `transaction: t`" bug quietly breaks isolation. Audit every query inside the callback.

---

### TypeORM

**Can TypeORM set isolation levels?** — Yes; `transaction()` and `QueryRunner` both accept a level.

```typescript
import { DataSource } from 'typeorm';

// Convenient form — level as first argument
await dataSource.transaction('SERIALIZABLE', async (manager) => {
  const seat = await manager.findOneBy(Session, { id: seatId });
  if (seat.status !== 'free') throw new Error('TAKEN');
  await manager.update(Session, { id: seatId }, { status: 'held', userId });
});

// Explicit QueryRunner form — needed for retry/backoff control
const runner = dataSource.createQueryRunner();
await runner.connect();
await runner.startTransaction('REPEATABLE READ');
try {
  await runner.manager.increment(Product, { id: productId }, 'stock', -1);
  await runner.commitTransaction();
} catch (e) {
  await runner.rollbackTransaction();
  throw e;                       // caller decides whether 40001 is retryable
} finally {
  await runner.release();
}
```

Accepted strings: `"READ UNCOMMITTED"`, `"READ COMMITTED"`, `"REPEATABLE READ"`, `"SERIALIZABLE"`.

**Where it breaks**: No automatic `40001` retry. The `EntityManager`-callback form is convenient but gives you no natural seam for a retry loop — you typically drop to `QueryRunner` (above) so you can catch, roll back, back off, and re-run the whole unit.

**Verdict**: Flexible level API. Use `QueryRunner` when you need retries so the loop can re-run the *entire* transaction body.

---

### Knex.js

**Can Knex set isolation levels?** — Yes, via `trx` config (`isolationLevel`), or by emitting the `SET TRANSACTION` statement.

```javascript
const knex = require('knex')({ client: 'pg', connection: process.env.DATABASE_URL });

await knex.transaction(
  async (trx) => {
    const seat = await trx('sessions').where({ id: seatId }).forUpdate().first();
    if (seat.status !== 'free') throw new Error('TAKEN');
    await trx('sessions').where({ id: seatId }).update({ status: 'held', user_id: userId });
  },
  { isolationLevel: 'serializable' }        // knex >= 0.95
);

// Older Knex / full control: set it as the first statement
await knex.transaction(async (trx) => {
  await trx.raw('SET TRANSACTION ISOLATION LEVEL SERIALIZABLE');
  // ... work ...
});
```

**Where it breaks**: `isolationLevel` in the config only landed in newer versions; on older ones you must `SET TRANSACTION ISOLATION LEVEL ...` as the first statement (it must precede any query, or Postgres rejects it). And, once more, Knex does **not** retry `40001`.

**Verdict**: Most transparent — you can always fall back to raw `SET TRANSACTION`. Retry is manual, and pair Serializable with your own backoff loop.

---

### ORM Summary Table

| ORM | Set Isolation Level | Levels Exposed | Auto-retry `40001`? | Isolation Footgun |
|-----|--------------------|----------------|---------------------|-------------------|
| Prisma | `$transaction(fn, { isolationLevel })` | RU / RC / RR / Serializable | **No** (`P2034` surfaced) | Must build own retry on `P2034` |
| Drizzle | `db.transaction(fn, { isolationLevel, accessMode, deferrable })` | RU / RC / RR / Serializable (+ READ ONLY DEFERRABLE) | **No** | None major; retry manual |
| Sequelize | `sequelize.transaction({ isolationLevel }, fn)` | RU / RC / RR / Serializable | **No** | Forgetting `{ transaction: t }` runs query outside the txn |
| TypeORM | `dataSource.transaction('LEVEL', fn)` / QueryRunner | RU / RC / RR / Serializable | **No** | Callback form has no retry seam — use QueryRunner |
| Knex | `{ isolationLevel }` or `SET TRANSACTION ...` | RU / RC / RR / Serializable | **No** | `isolationLevel` config is version-gated |

The universal lesson: **no ORM retries serialization failures for you.** If you choose `REPEATABLE READ` or `SERIALIZABLE`, you *must* wrap the transaction in a retry-with-backoff loop keyed on `40001` (and `40P01` for deadlocks), and you must re-run the *entire* transaction, not the failed statement.

---

## 13. Practice Exercises

### Exercise 1 — Easy

You want to read a user's balance and then their most recent order, and you need both reads to reflect the **same instant** — no other committed writes may appear between the two `SELECT`s. Under Postgres's default `READ COMMITTED`, each statement takes a fresh snapshot, so a concurrent commit *can* slip in between them.

Open a transaction at the isolation level that guarantees both reads see one frozen snapshot, then read from `users` and `orders`.

```sql
-- Tables: users(id, name, balance), orders(id, user_id, total_amount, created_at)
-- Write your query here
```

---

### Exercise 2 — Medium

Two admins might simultaneously run "give every employee in department 7 a 10% raise." Under `READ COMMITTED` a non-repeatable-read / lost-update pattern is possible if you read-then-write in the app. Write a transaction that performs the raise **atomically in the database** (arithmetic in the `UPDATE` itself) at `REPEATABLE READ`, so that if two run concurrently one will abort with `40001` rather than silently double- or under-applying.

State in a comment: at `REPEATABLE READ`, which specific anomaly does Postgres turn into a serialization error here, and which anomaly does it still *not* prevent?

```sql
-- Tables: employees(id, name, salary, department_id)
-- Write your query here
```

---

### Exercise 3 — Medium

Demonstrate a **phantom read** and then prevent it. Session A runs, twice within one transaction, `SELECT COUNT(*) FROM orders WHERE status = 'pending'`. Between the two counts, Session B inserts a new pending order and commits.

1. At which isolation level do A's two counts differ (the phantom appears)?
2. At which Postgres isolation level are the two counts guaranteed identical, and *why* (what does the snapshot do)?

Write the transaction body for Session A at the level that eliminates the phantom.

```sql
-- Tables: orders(id, user_id, total_amount, status, created_at)
-- Write your query here
```

---

### Exercise 4 — Hard

**Write skew.** The rule: at least one on-call employee must remain in each department. Two employees, both currently on-call in department 3, each open a transaction, read "how many others are on-call" (each sees 1 other, so each thinks it's safe to go off-call), and each sets themselves off-call. Both commit → department 3 now has **zero** on-call staff. This is write skew: two transactions read an overlapping set, write to disjoint rows, and together violate an invariant.

1. Explain why `REPEATABLE READ` (snapshot isolation) does **not** prevent this in Postgres, even though it prevents non-repeatable reads and phantoms.
2. Rewrite both transactions at `SERIALIZABLE` so that Postgres's SSI detects the read/write dependency cycle and aborts one with `40001`.

```sql
-- Tables: employees(id, name, department_id, on_call BOOLEAN)
-- Write your query here
```

---

### Exercise 5 — Interview

An `orders` write path currently runs at the default level and occasionally double-charges: the app does `SELECT status FROM orders WHERE id = $1`, checks it's `'pending'` in application code, then `UPDATE orders SET status='paid'` and `INSERT INTO payments`. Under concurrency two workers both read `'pending'` and both charge.

Without changing the isolation level, fix it with row locking. Then give the alternative fix at `SERIALIZABLE`. In a comment, state the trade-off between the two approaches (blocking vs. abort-and-retry).

```sql
-- Tables: orders(id, user_id, status, total_amount), payments(id, order_id, amount, created_at)
-- Write your query here
```

---

## 14. Interview Questions

### Q1 — Name the four standard isolation levels and the anomaly each one adds protection against.

**Junior answer:** The SQL standard defines READ UNCOMMITTED, READ COMMITTED, REPEATABLE READ, and SERIALIZABLE, from weakest to strongest. Each higher level prevents more anomalies:
- READ UNCOMMITTED allows dirty reads (seeing another transaction's uncommitted data).
- READ COMMITTED prevents dirty reads but allows non-repeatable reads (re-reading a row returns a changed value) and phantoms.
- REPEATABLE READ additionally prevents non-repeatable reads.
- SERIALIZABLE additionally prevents phantoms and guarantees the result equals *some* serial order of the transactions.

**Principal answer:** The three anomalies the *standard* enumerates (dirty read, non-repeatable read, phantom) are a historical, lock-based taxonomy and are incomplete. The standard says REPEATABLE READ *may* allow phantoms, but says nothing about **write skew** or **lost update**, which are the anomalies that actually bite MVCC databases. So the level-to-anomaly table depends heavily on the engine: Postgres's REPEATABLE READ (snapshot isolation) already prevents phantoms, yet still permits write skew — an anomaly the standard doesn't even name. The correct principal framing is: the SQL levels are *minimum guarantees*, and each engine's real behavior must be looked up, not assumed.

**Follow-up:** *"So is a database allowed to give you stronger guarantees than the level you asked for?"* — Yes. The standard specifies what a level must *at least* prevent, not what it must *allow*. Postgres READ UNCOMMITTED behaves exactly like READ COMMITTED (it never permits dirty reads at all), and its REPEATABLE READ is stronger than the standard requires.

---

### Q2 — In Postgres specifically, describe what each isolation level actually does under the hood.

**Junior answer:** Postgres has three distinct behaviors even though four levels are accepted:
- READ UNCOMMITTED is treated as READ COMMITTED — Postgres never shows uncommitted data.
- READ COMMITTED (the default) takes a **new snapshot at the start of each statement**, so successive statements can see newer committed data.
- REPEATABLE READ takes **one snapshot at the first statement** and holds it for the whole transaction — this is snapshot isolation.
- SERIALIZABLE uses snapshot isolation plus SSI (Serializable Snapshot Isolation) monitoring.

**Principal answer:** The key mechanism is MVCC snapshots. READ COMMITTED re-snapshots per statement, which is why a long-running `UPDATE ... WHERE` can see rows committed by others mid-statement and even re-evaluates the `WHERE` against the latest committed version of a row it's blocked on (the "EvalPlanQual" re-check). REPEATABLE READ pins a single snapshot; any write conflict with a transaction that committed after that snapshot raises `40001` rather than silently losing the update. SERIALIZABLE adds **predicate locking** (SIREAD locks) that track read/write dependencies between concurrent transactions; when it detects a dangerous structure — a cycle of rw-dependencies that could produce a non-serializable outcome — it aborts one transaction with `40001`. SSI is *optimistic*: it never blocks reads, it detects conflicts after the fact and rolls back.

**Follow-up:** *"What's the performance cost of SERIALIZABLE versus REPEATABLE READ in Postgres?"* — SSI adds bookkeeping (predicate locks in shared memory, which can be exhausted and escalate from row to page to relation granularity, increasing false-positive aborts) and a higher rollback rate under contention. It does **not** add read blocking. The real cost surfaces as retry churn, so SERIALIZABLE is only viable if you have a retry loop and contention is moderate. A `READ ONLY DEFERRABLE` serializable transaction is a special cheap case — it waits for a safe snapshot and then never aborts.

---

### Q3 — A financial app reads an account balance and, if sufficient, inserts a withdrawal. Under concurrency it sometimes overdraws. Walk me through the fix at each isolation level.

**Junior answer:** The problem is a read-check-write race: two transactions both read the old balance, both pass the check, both withdraw. At READ COMMITTED you fix it with row locking — `SELECT ... FOR UPDATE` on the account row so the second transaction blocks until the first commits, then re-reads the updated balance. Alternatively do the check and decrement in a single `UPDATE ... WHERE balance >= amount` and check the affected row count.

**Principal answer:** This is a **lost update / write skew** family problem, and there are three defensible fixes with different characteristics. (1) **Pessimistic:** `SELECT ... FOR UPDATE` at READ COMMITTED — simple, blocks concurrent writers, no retry needed, but serializes access to hot rows and risks deadlocks if lock ordering isn't disciplined. (2) **Atomic conditional write:** `UPDATE accounts SET balance = balance - $1 WHERE id = $2 AND balance >= $1` and treat `rowCount = 0` as "insufficient funds" — lock-free, single statement, ideal when the invariant is expressible as a predicate on one row. (3) **Optimistic / SERIALIZABLE:** run the read-check-write at SERIALIZABLE and let SSI abort losers with `40001`, wrapped in a retry loop — best when the invariant spans *multiple rows* (e.g., "sum of withdrawals ≤ balance across a ledger") where no single-row lock or predicate suffices. The choice is contention-driven: high contention on a single hot row favors (1) or (2); invariants spanning rows that can't be locked cheaply favor (3).

**Follow-up:** *"When does `FOR UPDATE` fail to protect you?"* — When the invariant depends on rows that don't yet exist (phantoms) — e.g., "no more than N active reservations for this product." `FOR UPDATE` can only lock rows it can *see* and select; it can't lock a row a competitor is about to *insert*. That's precisely the phantom/write-skew case where you need SERIALIZABLE (or an explicit `SELECT ... FOR UPDATE` on a parent/aggregate row that all inserters must also lock, or a `SERIALIZABLE`-backed predicate).

---

## 15. Mental Model Checkpoint

1. **Postgres accepts all four SQL isolation levels but only exhibits three distinct behaviors. Which two levels collapse into one, and what does that collapsed behavior actually forbid?**

2. **At READ COMMITTED, you run the same `SELECT` twice in one transaction and get different rows. Is this a bug? At which level would the two reads be guaranteed identical, and what mechanism makes that true?**

3. **REPEATABLE READ in Postgres already prevents phantoms — better than the SQL standard requires. So what anomaly can still occur at REPEATABLE READ that SERIALIZABLE fixes, and why can't a snapshot alone catch it?**

4. **Two transactions each do `SELECT` then `UPDATE` disjoint rows, and together break an invariant no single row could enforce. Name the anomaly and state exactly what SSI observes to abort one of them.**

5. **You switch a hot code path from REPEATABLE READ to SERIALIZABLE and add no other code. What new failure mode must your application now handle, and what is the SQLSTATE?**

6. **Why does `SELECT ... FOR UPDATE` protect against lost updates but *not* against phantom-based write skew? What is it fundamentally unable to lock?**

7. **A read-only reporting transaction must see a consistent snapshot but you never want it to abort. Which exact Postgres transaction mode gives you both, and what does it trade to achieve that?**

---

## 16. Quick Reference Card

```sql
-- ── Setting the level (must precede the first query) ──────────────
BEGIN ISOLATION LEVEL READ COMMITTED;     -- default: new snapshot per statement
BEGIN ISOLATION LEVEL REPEATABLE READ;    -- one snapshot for the whole txn (snapshot isolation)
BEGIN ISOLATION LEVEL SERIALIZABLE;       -- snapshot isolation + SSI conflict detection
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE READ ONLY DEFERRABLE;  -- consistent read, never aborts
-- Session default:
SET SESSION CHARACTERISTICS AS TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- ── Postgres reality (only 3 distinct behaviors) ─────────────────
-- READ UNCOMMITTED == READ COMMITTED   (Postgres never shows dirty data)
-- READ COMMITTED   -> re-snapshots each statement
-- REPEATABLE READ  -> one snapshot; write conflicts raise 40001
-- SERIALIZABLE     -> RR + predicate (SIREAD) locks; rw-dependency cycle -> 40001

-- ── Anomaly matrix (Postgres) ────────────────────────────────────
--                      Dirty  Non-repeat  Phantom  Write-skew  Lost-update
-- READ COMMITTED        no       YES        YES       YES         possible
-- REPEATABLE READ       no        no         no        YES        no (40001)
-- SERIALIZABLE          no        no         no         no        no

-- ── The retry loop is mandatory at RR / SERIALIZABLE ─────────────
-- 40001  = serialization_failure   (retryable: re-run WHOLE txn)
-- 40P01  = deadlock_detected       (retryable)
-- Business errors (e.g. 'SOLD_OUT') are NOT retryable — check err.code, not messages.

-- ── Row locking (alternative to raising the level) ───────────────
SELECT ... FOR UPDATE;   -- locks visible rows; does NOT stop phantoms/inserts
UPDATE t SET n = n - 1 WHERE id = $1 AND n >= 1;  -- atomic conditional write; check rowCount

-- Choosing:
--  single hot row, expressible predicate  -> FOR UPDATE or atomic UPDATE (RC)
--  invariant spans rows that may not exist -> SERIALIZABLE + retry loop
--  consistent read, no writes              -> SERIALIZABLE READ ONLY DEFERRABLE
```

---

## Connected Topics

- **Topic 52 — Transactions & ACID**: Isolation is the "I" in ACID; this topic sits inside the broader transaction lifecycle (BEGIN/COMMIT/ROLLBACK, atomicity, durability).
- **Topic 54 — MVCC Internals**: Snapshots, xmin/xmax, tuple visibility, and vacuum — the machinery that *implements* every isolation level described here.
- **Topic 55 — Locking (Pessimistic)**: `SELECT ... FOR UPDATE`, `FOR SHARE`, lock modes, and lock ordering — the pessimistic alternative to raising the isolation level.
- **Topic 56 — Deadlocks**: `40P01`, lock-ordering discipline, and why deadlocks share a retry loop with serialization failures.
- **Topic 57 — Optimistic Concurrency Control**: Version columns and compare-and-set — the application-level cousin of SERIALIZABLE's optimistic aborts.
- **Topic 27 — EXISTS and NOT EXISTS**: Existence checks whose correctness under concurrency depends entirely on the isolation level (the phantom-read trap).
- **Topic 11 — INNER JOIN**: Multi-row reads across joined tables are exactly the shape that write skew exploits when the invariant spans the joined set.
- **Topic 18 — The N+1 Query Problem**: Splitting one logical read into many statements at READ COMMITTED means each sees a *different* snapshot — an isolation hazard hiding inside a performance anti-pattern.

