# 43 — Concurrency Anomalies
## Phase: Transactions & Concurrency

---

## ELI5 — The Simple Analogy

Two people editing the same shared shopping list, at the same time, on paper.

**You read a line Meera is still writing** and she then crosses it out. You bought something she'd changed her mind about. — *dirty read*

**You read "milk" at the top, look away, look back, and it says "oat milk."** She edited it while you were reading. Your two glances disagree. — *non-repeatable read*

**You count six items, turn the page to check the total, and there are now seven.** She added one. Nothing you'd read changed — a *new* thing appeared. — *phantom read*

**You both cross off "eggs" and write "12 eggs."** Two people did the work; the list records it once. One of you wasted a trip. — *lost update*

**The rule is "at least one of us must stay home."** You both check: "she's home, so I can go out." She checks: "he's home, so I can go out." Both leave. Both checks were true when made; the outcome violates the rule anyway. — ***write skew***

That last one is the interesting one. Nobody read stale data, nobody overwrote anybody, and the invariant broke regardless. It is the anomaly that survives every isolation level except the strongest — and it is the one that causes real production bugs.

---

## Where this fits in the big picture

```
   39 transactions · 40 ACID (I = isolation)
                          │
                          ▼
        ┌──────────────────────────────────────────┐
        │ 43 CONCURRENCY ANOMALIES ← YOU ARE HERE  │
        │ the catalogue of what goes wrong         │
        └────────────────────┬─────────────────────┘
                             ▼
              44 isolation levels (which anomaly each prevents)
              45 locks · 46 MVCC (the mechanisms)
              50 SSI (the only thing that stops write skew)
```

Topic 40 said isolation is "the only letter with a dial." **This topic is the list of things the dial protects you from** — so that Topic 44's table means something.

---

## What is this?

Six named failure modes that occur when transactions run concurrently. Each has a precise definition, a two-transaction timeline that produces it, and a specific mechanism that prevents it.

| Anomaly | One-line definition |
|---|---|
| **Dirty read** | you read data another transaction has not committed |
| **Non-repeatable read** | you read the same row twice and get different values |
| **Phantom read** | you run the same query twice and get different *rows* |
| **Lost update** | two transactions read-modify-write; one update vanishes |
| **Read skew** | you read two related rows and see an inconsistent combination |
| **Write skew** | two transactions each read, then write, breaking an invariant neither could see |

The first three are in the SQL standard. **The last three are not** — and they are the ones that cause production incidents.

---

## Why does it matter for a backend developer?

Because the SQL standard's list is incomplete, and PostgreSQL's default level permits three of the six:

```
 ★ WHAT READ COMMITTED (the default) ALLOWS:
   ✗ non-repeatable read
   ✗ phantom read
   ✗ ★ LOST UPDATE
   ✗ ★ READ SKEW
   ✗ ★ WRITE SKEW

 ⇒ AND THE CODE THAT HITS THEM LOOKS OBVIOUSLY CORRECT:

   const row = await db.query('SELECT balance FROM accounts WHERE id=$1');
   if (row.balance >= amount) {
     await db.query('UPDATE accounts SET balance = $1 WHERE id=$2',
                    [row.balance - amount, id]);
   }

   ⇒ this is a LOST UPDATE, and at low traffic you will never see it.
     At 500 requests/second you will see it hundreds of times a day.
```

And the deeper reason: **you cannot choose an isolation level until you can name the anomaly you're afraid of.** "Use SERIALIZABLE to be safe" is not engineering — it costs retries and throughput. Naming the anomaly tells you the cheapest sufficient fix, which is often not an isolation level at all.

---

## The physical reality

### Why each anomaly is possible — the mechanism

```
 EVERY ANOMALY COMES FROM ONE OF TWO GAPS:

 ① THE SNAPSHOT GAP (read anomalies)
    Your transaction has a snapshot (Topic 46). Under READ COMMITTED
    a NEW snapshot is taken per STATEMENT. Between two statements,
    the world moves.
      ⇒ non-repeatable read, phantom read, read skew

 ② ★ THE READ-THEN-WRITE GAP (write anomalies)
    You SELECT, decide, then UPDATE. Between the SELECT and the UPDATE,
    another transaction can act on the same data.
      ⇒ lost update, write skew

 ⇒ THE FIXES MAP DIRECTLY:
   ① is fixed by holding ONE snapshot for the whole transaction
      (REPEATABLE READ — Topic 44)
   ② is fixed by:
      • removing the gap  →  one atomic statement          ★ cheapest
      • locking the gap   →  SELECT … FOR UPDATE           (Topic 45)
      • detecting the gap →  SERIALIZABLE / SSI            (Topic 50)
```

### What a lost update looks like in the tuple

```
 accounts row, id=1, balance=100000

 t1  txn A: SELECT balance → 100000        (reads tuple v1)
 t2  txn B: SELECT balance → 100000        (reads tuple v1, same snapshot)
 t3  txn A: UPDATE … SET balance = 90000
            ⇒ v1.t_xmax = A, new tuple v2 written with balance=90000
 t4  txn A: COMMIT
 t5  txn B: UPDATE … SET balance = 80000
            ⇒ B wants to update v1. But v1 is now dead (xmax=A, committed).
            ⇒ ★ UNDER READ COMMITTED, B RE-READS the latest version
              (an "EPQ" — EvalPlanQual re-check) and updates v2.
            ⇒ v2.t_xmax = B, new tuple v3 written with balance=80000
              ★ USING THE VALUE B COMPUTED FROM v1 (100000 - 20000)
 t6  txn B: COMMIT

 RESULT: balance = 80000.
 EXPECTED: 100000 - 10000 - 20000 = 70000.
 ⇒ ★ ₹100 LOST. A's update was applied and then overwritten by a
   calculation based on the pre-A value.

 ★ NOTE WHAT DID *NOT* HAPPEN: B did not fail. It did not block for
   long. It re-read and proceeded. READ COMMITTED's re-check makes
   the UPDATE apply to the newest row — but the VALUE B is writing
   was computed from the old one.
```

### Write skew — why it is different from everything else

```
 THE SETUP: at least one doctor must be on call.
 on_call(doctor_id, is_on_call)  — Alice: true, Bob: true

 t1  txn A: SELECT count(*) FROM on_call WHERE is_on_call  → 2
 t2  txn B: SELECT count(*) FROM on_call WHERE is_on_call  → 2
 t3  txn A: "2 >= 2, safe to go off call"
            UPDATE on_call SET is_on_call=false WHERE doctor_id='alice'
 t4  txn B: "2 >= 2, safe to go off call"
            UPDATE on_call SET is_on_call=false WHERE doctor_id='bob'
 t5  A COMMIT.  t6  B COMMIT.

 RESULT: ZERO doctors on call.

 ★ WHY NOTHING PREVENTS IT:
   • no dirty read      — both read committed data
   • no lost update     — they updated DIFFERENT ROWS
   • no write conflict  — no row lock is contended
   • REPEATABLE READ does NOT help — each transaction's snapshot was
     internally consistent, and neither wrote a row the other read.
   ⇒ ★ THEY READ A SET AND WROTE OUTSIDE IT. The conflict is between
     A's WRITE and B's READ PREDICATE — which no row-level mechanism
     can see.

 ⇒ ONLY TWO THINGS STOP IT:
   ① SERIALIZABLE (SSI) — tracks read PREDICATES and aborts one
      (Topic 50)
   ② materialising the conflict — a constraint or a lock on something
      both transactions must touch
```

---

## How it works — step by step

### ① Dirty read

```
 DEFINITION: reading data written by a transaction that has not
 committed (and may never commit).

 t1  txn A: BEGIN; UPDATE accounts SET balance = 0 WHERE id=1;
 t2  txn B: SELECT balance FROM accounts WHERE id=1;   → 0   ⚠
 t3  txn A: ROLLBACK;
 ⇒ B acted on a value that never existed.

 ★ POSTGRESQL: IMPOSSIBLE AT ANY ISOLATION LEVEL.
   MVCC means B's snapshot cannot see A's uncommitted tuple —
   there is no code path that would show it.
   READ UNCOMMITTED is accepted as syntax and behaves as READ COMMITTED.
 ⇒ ⚠ THIS IS NOT UNIVERSAL. Some engines genuinely permit it.
```

### ② Non-repeatable read

```
 DEFINITION: reading the SAME ROW twice in one transaction and
 getting different values.

 t1  txn A: BEGIN; SELECT balance FROM accounts WHERE id=1;  → 100000
 t2  txn B: UPDATE accounts SET balance=50000 WHERE id=1; COMMIT;
 t3  txn A: SELECT balance FROM accounts WHERE id=1;  → 50000   ⚠
 ⇒ A's two reads disagree. A report summing the same rows twice
   produces two different totals.

 ★ WHY: under READ COMMITTED a new snapshot is taken PER STATEMENT.
 ⇒ PREVENTED BY: REPEATABLE READ (one snapshot for the transaction).
```

### ③ Phantom read

```
 DEFINITION: re-running the same QUERY and getting different ROWS —
 not different values, NEW rows matching the predicate.

 t1  txn A: BEGIN; SELECT count(*) FROM orders WHERE total > 1000; → 6
 t2  txn B: INSERT INTO orders (total) VALUES (5000); COMMIT;
 t3  txn A: SELECT count(*) FROM orders WHERE total > 1000; → 7   ⚠

 ★ THE DISTINCTION FROM ②: non-repeatable read = an EXISTING row
   changed. Phantom = the SET of matching rows changed.

 ★ POSTGRESQL: REPEATABLE READ ALREADY PREVENTS PHANTOMS.
   The SQL standard only requires SERIALIZABLE to do so, but
   PostgreSQL's REPEATABLE READ is SNAPSHOT ISOLATION, which is
   stronger — one snapshot means new rows are invisible.
   ⇒ ★ do not carry over the standard's table from other engines.
```

### ④ Lost update

```
 DEFINITION: two transactions read the same value, each computes a new
 value from it, and one write is silently overwritten.

 (the tuple-level trace is in "physical reality" above)

 ★ THE APPLICATION SHAPE THAT CAUSES IT:
     SELECT x → compute in application code → UPDATE with the result
   ⇒ ANY read-modify-write across two statements.

 ★ PREVENTED BY, cheapest first:
   ① ONE ATOMIC STATEMENT — the fix that costs nothing
        UPDATE accounts SET balance = balance - $1
         WHERE id=$2 AND balance >= $1;
        ⇒ the read and the write are the same operation. No gap.
   ② SELECT … FOR UPDATE — take the row lock during the read
   ③ REPEATABLE READ — the second transaction gets
        ERROR: could not serialize access due to concurrent update
        ⇒ ★ it does not block; it ABORTS. You must retry.
   ④ an optimistic version column (Topic 49)
```

### ⑤ Read skew (inconsistent read)

```
 DEFINITION: reading two RELATED rows at different times and seeing a
 combination that was never simultaneously true.

 accounts: id=1 balance=100000, id=2 balance=100000.  Total: 200000.

 t1  txn A: SELECT balance FROM accounts WHERE id=1;  → 100000
 t2  txn B: BEGIN;
            UPDATE accounts SET balance=50000  WHERE id=1;
            UPDATE accounts SET balance=150000 WHERE id=2;
            COMMIT;
 t3  txn A: SELECT balance FROM accounts WHERE id=2;  → 150000
 ⇒ A computes a total of 250000. ★ THAT TOTAL NEVER EXISTED.

 ★ WHY IT MATTERS: every "does the ledger balance?" report run at
   READ COMMITTED can produce a number that was never true. Backups
   taken with an inconsistent read are worse still.
 ⇒ PREVENTED BY: REPEATABLE READ. One snapshot ⇒ one consistent
   view of the whole database.
```

### ⑥ Write skew — the one that survives snapshot isolation

```
 DEFINITION: two transactions read overlapping data, then write
 DISJOINT rows, and the combined result violates an invariant that
 each transaction individually preserved.

 (the on-call trace is in "physical reality" above)

 ★ THE SHAPE, GENERALISED:
     BEGIN;
       SELECT … WHERE <predicate>;      -- read a SET
       -- decide based on an aggregate or a count over that set
       INSERT/UPDATE … ;                 -- write something that CHANGES
                                         --   which rows match the predicate
     COMMIT;
   ⇒ if two transactions do this concurrently, each one's write
     invalidates the other's read — but neither read the other's row.

 ★ REAL EXAMPLES:
   • "at least one doctor on call"
   • "no double-booking" (both check the slot is free, both book)
   • "total allocated must not exceed budget" (both check, both spend)
   • "at most N active subscriptions" (both count, both insert)
   • "a meeting room must not be double-booked" (case study 02)

 ⇒ PREVENTED BY, in order of preference:
   ① ★ MATERIALISE THE CONFLICT AS A CONSTRAINT
        make both transactions touch the SAME object:
          UNIQUE, EXCLUDE, or a counter row they both update.
        ⇒ the conflict becomes a write-write conflict, which every
          isolation level handles. (Topic 24, case study 02.)
   ② SELECT … FOR UPDATE on a shared row (a "gate" row)
   ③ SERIALIZABLE — SSI detects it and aborts one (Topic 50)
   ✗ REPEATABLE READ DOES NOT HELP. This is the key fact.
```

---

## Concept breakdown

```
THE TWO GAPS THAT CAUSE EVERYTHING
├── ① THE SNAPSHOT GAP  (a new snapshot per statement)
│      → non-repeatable read · phantom read · read skew
│      ⇒ fixed by ONE snapshot per transaction (REPEATABLE READ)
└── ② ★ THE READ-THEN-WRITE GAP (SELECT, decide, UPDATE)
       → lost update · write skew
       ⇒ fixed by removing the gap, locking it, or detecting it

THE SIX ANOMALIES
├── DIRTY READ            read uncommitted data
│                         ★ impossible in PostgreSQL, any level
├── NON-REPEATABLE READ   same row, two values
├── PHANTOM READ          same query, different rows
│                         ★ PostgreSQL's REPEATABLE READ already
│                           prevents this (stronger than the standard)
├── LOST UPDATE           two read-modify-writes, one vanishes
│                         ★ NOT in the SQL standard
├── READ SKEW             two related rows, a combination that never existed
│                         ★ NOT in the SQL standard
└── ★ WRITE SKEW          disjoint writes break a shared invariant
                          ★ NOT in the standard; survives snapshot
                            isolation; only SSI or a materialised
                            conflict prevents it

★ WHAT READ COMMITTED (the default) ALLOWS
  non-repeatable · phantom · lost update · read skew · write skew
  ⇒ five of six.

THE FIX HIERARCHY — cheapest first
├── ① one atomic statement          (lost update; costs nothing)
├── ② a constraint that materialises the conflict
│      UNIQUE / EXCLUDE             (write skew; costs nothing at runtime)
├── ③ SELECT … FOR UPDATE           (both; costs contention)
├── ④ REPEATABLE READ               (read anomalies + lost update;
│                                    costs retries)
└── ⑤ SERIALIZABLE                  (everything; costs more retries)

★ THE RULE: NAME THE ANOMALY FIRST. Then pick the cheapest fix that
  covers it. "Use SERIALIZABLE to be safe" skips the analysis and
  buys retries you may not need.
```

---

## Diagrams

**Diagram 1 — big picture: the six anomalies by gap**

```
                        WHERE IS THE GAP?
                              │
        ┌─────────────────────┴──────────────────────┐
        ▼                                            ▼
  ① SNAPSHOT GAP                            ② READ-THEN-WRITE GAP
  (the world moves between                  (another txn acts between
   your statements)                          your SELECT and UPDATE)
        │                                            │
   ┌────┼─────────────┐                    ┌─────────┴─────────┐
   ▼    ▼             ▼                    ▼                   ▼
 non-  phantom     read skew          lost update        ★ WRITE SKEW
 repeat- read      (two rows,         (same row,         (DIFFERENT rows,
 able              inconsistent       one write          shared invariant
 read              combination)       overwritten)        broken)
   │      │             │                    │                   │
   └──────┴─────────────┘                    │                   │
        FIXED BY:                            │                   │
   one snapshot per txn                      │                   │
   (REPEATABLE READ)                         │                   │
                                    FIXED BY:               FIXED BY:
                                    an atomic statement,    a materialised
                                    FOR UPDATE, or          conflict, or
                                    REPEATABLE READ         ★ SERIALIZABLE
                                                            ONLY
```

**Diagram 2 — data flow: lost update vs write skew**

```
  LOST UPDATE — same row                WRITE SKEW — different rows
  ────────────────────────────          ──────────────────────────────
   A: SELECT bal=100                     A: SELECT count(on_call)=2
   B: SELECT bal=100                     B: SELECT count(on_call)=2
   A: UPDATE bal=90  ─┐                  A: UPDATE alice=false ─┐
   A: COMMIT          │                  A: COMMIT              │
   B: UPDATE bal=80 ──┤ SAME ROW         B: UPDATE bob=false ───┤ DIFFERENT
   B: COMMIT          │                  B: COMMIT              │  ROWS
                      ▼                                         ▼
   ★ B's write LANDS ON TOP of A's.      ★ NEITHER write conflicts.
     A row-level mechanism CAN see         No row-level mechanism can
     this conflict.                        see it — the conflict is
                                           between A's WRITE and B's
                                           READ PREDICATE.
   ⇒ FOR UPDATE fixes it.                ⇒ FOR UPDATE on those rows
   ⇒ REPEATABLE READ aborts B.             does NOT fix it.
                                         ⇒ REPEATABLE READ does NOT
                                           fix it.
                                         ⇒ ★ only SSI, or forcing both
                                           to touch a shared object.
```

**Diagram 3 — before/after: materialising a write-skew conflict**

```
 ✗ WRITE SKEW IS POSSIBLE
 ┌──────────────────────────────────────────────────────────────────┐
 │ on_call(doctor_id, is_on_call)                                   │
 │                                                                   │
 │ A: SELECT count(*) … WHERE is_on_call  → 2                       │
 │ B: SELECT count(*) … WHERE is_on_call  → 2                       │
 │ A: UPDATE … WHERE doctor_id='alice'    ─┐ disjoint rows          │
 │ B: UPDATE … WHERE doctor_id='bob'      ─┘ no conflict detected   │
 │ ⇒ ★ ZERO DOCTORS ON CALL                                         │
 └──────────────────────────────────────────────────────────────────┘

 ✓ THE CONFLICT MATERIALISED — both must touch the SAME row
 ┌──────────────────────────────────────────────────────────────────┐
 │ shift_coverage(shift_id PK, on_call_count int                    │
 │                CHECK (on_call_count >= 1))                       │
 │                                                                   │
 │ A: UPDATE shift_coverage SET on_call_count = on_call_count - 1   │
 │      WHERE shift_id = 1;          ← takes a ROW LOCK             │
 │ B: UPDATE shift_coverage SET on_call_count = on_call_count - 1   │
 │      WHERE shift_id = 1;          ← ⏸ BLOCKS on A's lock         │
 │ A: COMMIT   (count now 1)                                        │
 │ B: proceeds → count would be 0 → ★ CHECK CONSTRAINT VIOLATION    │
 │ ⇒ B is rejected. The invariant holds, at READ COMMITTED.         │
 └──────────────────────────────────────────────────────────────────┘
        ↑ ★ no isolation level change, no retries, no SSI overhead.
          The write skew became a write-write conflict.
```

---

## Example 1 — basic

Every anomaly, reproduced. Open two `psql` sessions.

```sql
CREATE TABLE accounts (
  id bigint PRIMARY KEY,
  balance_minor bigint NOT NULL
);
INSERT INTO accounts VALUES (1,100000),(2,100000);
```

**① Dirty read — impossible in PostgreSQL.**

```sql
-- session 1                          -- session 2
BEGIN;
UPDATE accounts SET balance_minor=0
  WHERE id=1;
-- do NOT commit
                                      BEGIN ISOLATION LEVEL READ UNCOMMITTED;
                                      SELECT balance_minor FROM accounts WHERE id=1;
```
```
 balance_minor
---------------
        100000        ★ the OLD value. The uncommitted write is invisible.
```
```sql
-- session 1
ROLLBACK;
```
**Even at `READ UNCOMMITTED`, PostgreSQL shows committed data.** MVCC has no code path to do otherwise.

**② Non-repeatable read — at the default level.**

```sql
-- session 1                          -- session 2
BEGIN;  -- READ COMMITTED
SELECT balance_minor FROM accounts
  WHERE id=1;              -- 100000
                                      UPDATE accounts SET balance_minor=50000
                                        WHERE id=1;   -- autocommit
SELECT balance_minor FROM accounts
  WHERE id=1;              -- 50000   ★ CHANGED
COMMIT;
```
```sql
-- now at REPEATABLE READ
UPDATE accounts SET balance_minor=100000 WHERE id=1;

-- session 1                          -- session 2
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT balance_minor FROM accounts
  WHERE id=1;              -- 100000
                                      UPDATE accounts SET balance_minor=50000
                                        WHERE id=1;
SELECT balance_minor FROM accounts
  WHERE id=1;              -- 100000  ★ STABLE
COMMIT;
```

**③ Phantom read — and why REPEATABLE READ already stops it.**

```sql
CREATE TABLE orders (id bigserial PRIMARY KEY, total_minor bigint NOT NULL);
INSERT INTO orders (total_minor) SELECT g*1000 FROM generate_series(1,6) g;

-- session 1                          -- session 2
BEGIN;  -- READ COMMITTED
SELECT count(*) FROM orders
  WHERE total_minor > 3000;  -- 3
                                      INSERT INTO orders (total_minor) VALUES (9000);
SELECT count(*) FROM orders
  WHERE total_minor > 3000;  -- 4     ★ A PHANTOM
COMMIT;

-- REPEATABLE READ
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT count(*) FROM orders WHERE total_minor > 3000;  -- 4
                                      INSERT INTO orders (total_minor) VALUES (9999);
SELECT count(*) FROM orders WHERE total_minor > 3000;  -- 4  ★ no phantom
COMMIT;
```
**PostgreSQL's REPEATABLE READ is snapshot isolation** — stronger than the SQL standard, which permits phantoms at this level.

**④ Lost update — the one that costs money.**

```sql
UPDATE accounts SET balance_minor=100000 WHERE id=1;

-- session 1                          -- session 2
BEGIN;
SELECT balance_minor FROM accounts
  WHERE id=1;              -- 100000
                                      BEGIN;
                                      SELECT balance_minor FROM accounts
                                        WHERE id=1;      -- 100000
UPDATE accounts
  SET balance_minor = 100000 - 10000
  WHERE id=1;
COMMIT;
                                      UPDATE accounts
                                        SET balance_minor = 100000 - 20000
                                        WHERE id=1;
                                      COMMIT;
SELECT balance_minor FROM accounts WHERE id=1;
```
```
 balance_minor
---------------
         80000        ★ expected 70000. ₹100 lost.
```

**The three fixes, cheapest first:**

```sql
-- ① ATOMIC STATEMENT — costs nothing
UPDATE accounts SET balance_minor = balance_minor - 10000
 WHERE id=1 AND balance_minor >= 10000;
-- ★ the read and the write are one operation. No gap exists.

-- ② FOR UPDATE — costs contention
BEGIN;
SELECT balance_minor FROM accounts WHERE id=1 FOR UPDATE;  -- ★ blocks others
UPDATE accounts SET balance_minor = … WHERE id=1;
COMMIT;

-- ③ REPEATABLE READ — costs a retry
-- session 1                          -- session 2
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT balance_minor FROM accounts
  WHERE id=1;              -- 100000
                                      BEGIN ISOLATION LEVEL REPEATABLE READ;
                                      SELECT balance_minor FROM accounts WHERE id=1;
UPDATE accounts SET balance_minor=90000
  WHERE id=1;
COMMIT;
                                      UPDATE accounts SET balance_minor=80000
                                        WHERE id=1;
-- ERROR: could not serialize access due to concurrent update
--   ★ it ABORTS rather than blocking. The application must retry.
```

**⑤ Read skew — the total that never existed.**

```sql
UPDATE accounts SET balance_minor=100000;   -- both back to 100000

-- session 1                          -- session 2
BEGIN;  -- READ COMMITTED
SELECT balance_minor FROM accounts
  WHERE id=1;              -- 100000
                                      BEGIN;
                                      UPDATE accounts SET balance_minor=50000  WHERE id=1;
                                      UPDATE accounts SET balance_minor=150000 WHERE id=2;
                                      COMMIT;
SELECT balance_minor FROM accounts
  WHERE id=2;              -- 150000
COMMIT;
-- ★ session 1 computed a total of 250,000. The real total was
--   always 200,000. That number never existed at any instant.
```
```sql
-- REPEATABLE READ gives a consistent view
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT sum(balance_minor) FROM accounts;   -- always internally consistent
COMMIT;
```

**⑥ Write skew — and proof that REPEATABLE READ does not help.**

```sql
CREATE TABLE on_call (
  doctor_id text PRIMARY KEY,
  is_on_call boolean NOT NULL
);
INSERT INTO on_call VALUES ('alice',true),('bob',true);
-- INVARIANT: at least one doctor on call.

-- session 1                          -- session 2
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT count(*) FROM on_call
  WHERE is_on_call;        -- 2
                                      BEGIN ISOLATION LEVEL REPEATABLE READ;
                                      SELECT count(*) FROM on_call
                                        WHERE is_on_call;   -- 2
UPDATE on_call SET is_on_call=false
  WHERE doctor_id='alice';
COMMIT;
                                      UPDATE on_call SET is_on_call=false
                                        WHERE doctor_id='bob';
                                      COMMIT;              -- ★ SUCCEEDS

SELECT count(*) FROM on_call WHERE is_on_call;
```
```
 count
-------
     0        ★ THE INVARIANT IS BROKEN, at REPEATABLE READ.
```

```sql
-- SERIALIZABLE catches it
UPDATE on_call SET is_on_call=true;

-- session 1                          -- session 2
BEGIN ISOLATION LEVEL SERIALIZABLE;
SELECT count(*) FROM on_call
  WHERE is_on_call;        -- 2
                                      BEGIN ISOLATION LEVEL SERIALIZABLE;
                                      SELECT count(*) FROM on_call
                                        WHERE is_on_call;   -- 2
UPDATE on_call SET is_on_call=false
  WHERE doctor_id='alice';
COMMIT;                    -- ✓
                                      UPDATE on_call SET is_on_call=false
                                        WHERE doctor_id='bob';
                                      COMMIT;
-- ERROR: could not serialize access due to read/write dependencies
--        among transactions
-- HINT: The transaction might succeed if retried.
--   ★ SSI detected the read/write dependency cycle. (Topic 50.)
```

**And the fix that needs no isolation change at all:**

```sql
UPDATE on_call SET is_on_call=true;
CREATE TABLE shift_coverage (
  shift_id int PRIMARY KEY,
  on_call_count int NOT NULL CHECK (on_call_count >= 1)
);
INSERT INTO shift_coverage VALUES (1, 2);

-- session 1                          -- session 2  (both READ COMMITTED)
BEGIN;
UPDATE shift_coverage
  SET on_call_count = on_call_count-1
  WHERE shift_id=1;                   -- takes the row lock
UPDATE on_call SET is_on_call=false
  WHERE doctor_id='alice';
                                      BEGIN;
                                      UPDATE shift_coverage
                                        SET on_call_count = on_call_count-1
                                        WHERE shift_id=1;   -- ⏸ BLOCKS
COMMIT;
                                      -- unblocks, count would be 0:
-- ERROR: new row for relation "shift_coverage" violates check
--        constraint "shift_coverage_on_call_count_check"
--   ★ REJECTED, at READ COMMITTED, with no retries.
```

---

## Example 2 — production scenario

**The situation.** A hotel-booking platform. Three bug reports over one quarter that nobody connects to each other.

```
 ① "Two guests assigned the same room on the same night."   ~4/month
 ② "The nightly revenue report differs from the sum of bookings."
 ③ "Loyalty points went negative for 41 accounts."
```

**Step 1 — classify each as a named anomaly.**

```js
// ① the booking handler
const conflicts = await db.query(
  `SELECT 1 FROM bookings
    WHERE room_id=$1 AND daterange(check_in, check_out) && daterange($2,$3)`,
  [roomId, checkIn, checkOut]);
if (conflicts.rowCount === 0) {
  await db.query('INSERT INTO bookings (room_id, check_in, check_out) VALUES ($1,$2,$3)',
                 [roomId, checkIn, checkOut]);
}
```
```
 ★ WRITE SKEW. Two transactions each read a predicate ("no overlapping
   booking"), each find it true, each INSERT a row the other did not
   read. Disjoint writes, shared invariant, broken.
   ⇒ REPEATABLE READ would NOT fix this.
```

```js
// ② the revenue report
const rows = await db.query('SELECT sum(amount_minor) FROM bookings WHERE …');
const refunds = await db.query('SELECT sum(amount_minor) FROM refunds WHERE …');
const revenue = rows[0].sum - refunds[0].sum;
```
```
 ★ READ SKEW. Two statements, two snapshots (READ COMMITTED). A
   booking+refund pair committed between them is counted once.
   ⇒ the number was never true at any instant.
```

```js
// ③ the loyalty deduction
const acct = await db.query('SELECT points FROM loyalty WHERE user_id=$1', [uid]);
if (acct.rows[0].points >= cost) {
  await db.query('UPDATE loyalty SET points=$1 WHERE user_id=$2',
                 [acct.rows[0].points - cost, uid]);
}
```
```
 ★ LOST UPDATE. Classic read-modify-write across two statements.
```

**Step 2 — quantify each.**

```sql
-- ① overlapping bookings that exist right now
SELECT count(*) FROM bookings a JOIN bookings b
  ON a.room_id=b.room_id AND a.id<b.id
 AND daterange(a.check_in,a.check_out) && daterange(b.check_in,b.check_out)
WHERE a.status<>'cancelled' AND b.status<>'cancelled';
```
```
 count
-------
    47        ★ over 11 months
```
```sql
-- ③ negative balances
SELECT count(*) FROM loyalty WHERE points < 0;
```
```
 count
-------
    41
```

**Step 3 — pick the cheapest sufficient fix for each. They are all different.**

```sql
-- ①  WRITE SKEW → ★ MATERIALISE THE CONFLICT AS A CONSTRAINT
--     No isolation change, no retries, no application logic.
CREATE EXTENSION IF NOT EXISTS btree_gist;
ALTER TABLE bookings ADD CONSTRAINT no_double_booking
  EXCLUDE USING gist (room_id WITH =, daterange(check_in, check_out) WITH &&)
  WHERE (status <> 'cancelled');
-- ⇒ the two transactions now conflict on the SAME index entry.
--   The second gets 23P01 immediately. (Topics 16, 24, case study 02.)
```

```js
// ②  READ SKEW → one snapshot for the whole report
async function revenueReport(from, to) {
  const client = await pool.connect();
  try {
    await client.query('BEGIN ISOLATION LEVEL REPEATABLE READ');
    // ★ or, better, one statement — no snapshot question at all:
    const { rows } = await client.query(`
      SELECT
        (SELECT coalesce(sum(amount_minor),0) FROM bookings
          WHERE created_at >= $1 AND created_at < $2) -
        (SELECT coalesce(sum(amount_minor),0) FROM refunds
          WHERE created_at >= $1 AND created_at < $2) AS revenue_minor`,
      [from, to]);
    await client.query('COMMIT');
    return rows[0];
  } finally { client.release(); }
}
```

```sql
-- ③  LOST UPDATE → ★ ONE ATOMIC STATEMENT. Costs nothing.
UPDATE loyalty
   SET points = points - $1
 WHERE user_id = $2 AND points >= $1
RETURNING points;
-- rowCount === 0 ⇒ insufficient points. No race window.

-- and make the invariant structural
ALTER TABLE loyalty ADD CONSTRAINT ck_points_nonneg CHECK (points >= 0) NOT VALID;
-- (clean the 41 rows first — they are a business decision, case study 03)
```

**Step 4 — what they deliberately did *not* do.**

```
 ★ THE TEMPTING WRONG ANSWER: "set SERIALIZABLE globally."

 MEASURED on the booking workload:
   READ COMMITTED + EXCLUDE constraint : 8,412 bookings/s, 0.4% clean 409s
   SERIALIZABLE everywhere             : 1,204 bookings/s, 18% serialization
                                         failures needing retry
 ⇒ 7× less throughput, and the application now needs a retry loop on
   every write path.

 ⇒ ★ NAMING THE ANOMALY GAVE A CHEAPER, STRONGER FIX:
   an EXCLUDE constraint is enforced at READ COMMITTED, needs no
   retries, and cannot be bypassed by a code path someone forgets.
```

**Step 5 — where SERIALIZABLE *was* the right answer.**

One requirement had no constraint-shaped fix:

> *"A hotel's total confirmed bookings for a night must not exceed its room count, across all channels."*

```
 ⇒ this reads an AGGREGATE over a set and writes a new row into that
   set. There is no single object both transactions touch, and no
   EXCLUDE can express "count of matching rows <= N".
 ⇒ options:
   ① a per-hotel-per-night counter row that both must UPDATE
      ✓ works at READ COMMITTED
      ✗ ★ a hot row: every booking for a popular hotel serialises
        on it (case study 01's problem)
   ② SERIALIZABLE for this one path
      ✓ no hot row
      ✗ retries under contention
 ⇒ THEY CHOSE ② — but only for this handler:
```

```js
async function bookWithCapacityCheck(hotelId, night, roomType) {
  for (let attempt = 0; attempt < 3; attempt++) {
    try {
      return await withTransaction(async (tx) => {
        await tx.query('SET TRANSACTION ISOLATION LEVEL SERIALIZABLE');
        const { rows } = await tx.query(
          `SELECT count(*) AS booked FROM bookings
            WHERE hotel_id=$1 AND $2 BETWEEN check_in AND check_out - 1
              AND room_type=$3 AND status<>'cancelled'`,
          [hotelId, night, roomType]);
        const cap = await tx.query(
          'SELECT room_count FROM hotel_capacity WHERE hotel_id=$1 AND room_type=$2',
          [hotelId, roomType]);
        if (Number(rows[0].booked) >= cap.rows[0].room_count)
          throw new AppError('SOLD_OUT');
        return tx.query('INSERT INTO bookings (…) VALUES (…) RETURNING id', […]);
      });
    } catch (e) {
      if (e.code === '40001' && attempt < 2) {     // ★ serialization_failure
        await sleep(20 * Math.pow(2, attempt) + Math.random() * 20);
        continue;                                  // retry with backoff
      }
      throw e;
    }
  }
}
```

**Step 6 — results.**

| Bug | Anomaly | Fix | Isolation level | Retries |
|---|---|---|---|---|
| Double-booked room | **write skew** | `EXCLUDE` constraint | READ COMMITTED | none |
| Revenue mismatch | **read skew** | one statement | READ COMMITTED | none |
| Negative points | **lost update** | atomic `UPDATE` + `CHECK` | READ COMMITTED | none |
| Over-capacity | **write skew (aggregate)** | SERIALIZABLE + retry | SERIALIZABLE | ~2% |

```
 OUTCOME:
   double-bookings   47 → 0 (structurally impossible)
   revenue mismatch  fixed
   negative points   41 → 0 (impossible)
   throughput        unchanged on 3 of 4 paths
   ⇒ ★ ONE of four bugs needed an isolation-level change. The other
     three had cheaper, stronger fixes — found by naming the anomaly.
```

---

## Common mistakes

**1. Not knowing the default permits lost updates.**
- *Symptom:* balances, counters and inventory drift under load, with code that looks correct.
- *Engine-level why:* READ COMMITTED re-reads the row on conflict, so the `UPDATE` lands — but the *value* was computed from the pre-conflict read.
- *Fix:* one atomic statement. `SET x = x - $1 WHERE … AND x >= $1`.

**2. Assuming REPEATABLE READ prevents write skew.**
- *Symptom:* an invariant breaks despite raising the isolation level.
- *Engine-level why:* snapshot isolation prevents anomalies involving rows you *read*. Write skew's conflict is between one transaction's write and another's read *predicate*.
- *Fix:* materialise the conflict (`EXCLUDE`, `UNIQUE`, a shared counter row) or use SERIALIZABLE.

**3. Reaching for SERIALIZABLE without naming the anomaly.**
- *Symptom:* 7× throughput loss and retry loops everywhere.
- *Fix:* name it first. Most real cases have a constraint-shaped fix that is cheaper *and* stronger.

**4. Carrying over the SQL standard's table from another engine.**
- *Symptom:* believing PostgreSQL's REPEATABLE READ permits phantoms.
- *Engine-level why:* PostgreSQL implements snapshot isolation, which is stronger than the standard requires at that level.
- *Fix:* use PostgreSQL's own table (Topic 44).

**5. Multi-statement reports at READ COMMITTED.**
- *Symptom:* a total that doesn't reconcile, intermittently.
- *Engine-level why:* a new snapshot per statement means two queries see two different worlds.
- *Fix:* one statement, or `REPEATABLE READ` for the whole report.

**6. Handling `40001` as a generic error.**
- *Symptom:* transient serialization failures surfaced to users as 500s.
- *Engine-level why:* `40001` is *expected* at REPEATABLE READ and SERIALIZABLE — it means "retry me."
- *Fix:* a retry loop with exponential backoff and jitter, bounded attempts.

**7. Solving write skew with a hot counter row without measuring.**
- *Symptom:* correctness fixed, throughput destroyed on popular entities.
- *Fix:* it is a valid fix — but it serialises every writer on one row (case study 01). Compare against SERIALIZABLE for that path.

---

## Hands-on proof

**PROVE IT #1–#6 — every anomaly.** (Example 1, all six.)

**PROVE IT #7 — see the serialization failure codes.**
```sql
-- REPEATABLE READ lost-update conflict
-- ERROR: could not serialize access due to concurrent update   → SQLSTATE 40001
-- SERIALIZABLE read/write dependency
-- ERROR: could not serialize access due to read/write dependencies … → 40001
-- deadlock
-- ERROR: deadlock detected                                     → 40P01
-- ⇒ ★ 40001 and 40P01 are both RETRYABLE. Everything else is not.
```

**PROVE IT #8 — measure the cost of each fix.**
```bash
cat > /tmp/lost_update_atomic.sql <<'EOF'
\set id random(1, 1000)
UPDATE accounts SET balance_minor = balance_minor - 1
 WHERE id = :id AND balance_minor >= 1;
EOF
cat > /tmp/lost_update_serializable.sql <<'EOF'
\set id random(1, 1000)
BEGIN ISOLATION LEVEL SERIALIZABLE;
SELECT balance_minor FROM accounts WHERE id = :id;
UPDATE accounts SET balance_minor = balance_minor - 1 WHERE id = :id;
COMMIT;
EOF
pgbench -f /tmp/lost_update_atomic.sql       -c 50 -j 4 -T 30 shop
pgbench -f /tmp/lost_update_serializable.sql -c 50 -j 4 -T 30 shop
```
```
 atomic       : tps = 18,412   failed = 0
 serializable : tps =  2,104   failed = 4,882 (retryable 40001)
```

**PROVE IT #9 — SSI's conflict tracking, visible.**
```sql
SELECT count(*) AS predicate_locks FROM pg_locks WHERE mode = 'SIReadLock';
-- ★ non-zero only when SERIALIZABLE transactions are active.
--   These are the predicate locks SSI uses to detect write skew (Topic 50).
```

---

## The design decision framework

```
★★★ NAME THE ANOMALY BEFORE CHOOSING A FIX. ★★★

 ① WHAT IS THE SHAPE OF YOUR CODE?

    SELECT x → compute → UPDATE the SAME row
      ⇒ ★ LOST UPDATE
      ⇒ FIX: ONE ATOMIC STATEMENT.  Costs nothing.
             UPDATE t SET x = x - $1 WHERE id=$2 AND x >= $1;

    SELECT a set / an aggregate → INSERT or UPDATE a DIFFERENT row
    that changes which rows match
      ⇒ ★ WRITE SKEW
      ⇒ FIX, in order:
         ① materialise the conflict: EXCLUDE / UNIQUE / a shared
            counter row  ← cheapest AND strongest; works at READ
            COMMITTED, no retries, unbypassable
         ② SERIALIZABLE + a retry loop  ← when no constraint fits
            (typically "count over a set must not exceed N")
         ✗ REPEATABLE READ does NOT help

    Two or more SELECTs whose results must agree
      ⇒ ★ READ SKEW / NON-REPEATABLE READ / PHANTOM
      ⇒ FIX: one statement, or REPEATABLE READ for the whole
             transaction

 ② PICK THE CHEAPEST SUFFICIENT MECHANISM
    atomic statement      free, no retries, no contention
    UNIQUE / EXCLUDE      free at runtime, no retries, unbypassable
    SELECT … FOR UPDATE   contention on the locked rows
    REPEATABLE READ       retries on 40001; one snapshot per txn
    SERIALIZABLE          more retries; the only general answer to
                          aggregate-based write skew
    ⇒ ★ 3 of 4 real cases have a fix above the isolation-level line.

 ③ IF YOU CHOOSE AN ISOLATION LEVEL, YOU OWE A RETRY LOOP
    catch SQLSTATE 40001 (serialization_failure) and 40P01 (deadlock)
    exponential backoff + jitter, bounded attempts, and a metric
    ⇒ an unhandled 40001 is a 500 the user did not need to see.

 ④ SET IT PER TRANSACTION, NOT GLOBALLY
    BEGIN ISOLATION LEVEL SERIALIZABLE;   -- for the one handler that
                                          --   needs it
    ⇒ a global change costs every path for the benefit of one.

THE SIGNAL TO LOOK FOR — grep your codebase:
  • a SELECT followed by an UPDATE of the same row, in the same handler
    ⇒ lost update. Rewrite as one statement.
  • a SELECT COUNT / SUM / EXISTS followed by an INSERT or UPDATE
    ⇒ ★ write skew. Look for a constraint that materialises it.
  • two SELECTs whose results are combined arithmetically
    ⇒ read skew. One statement, or REPEATABLE READ.
  • any catch block that treats 40001 as fatal
    ⇒ a retryable error surfaced as an outage.
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
Using two `psql` sessions, reproduce all five anomalies PostgreSQL permits at READ COMMITTED. For each, state which gap (snapshot or read-then-write) causes it. Then show that REPEATABLE READ fixes three of them and not the other two.

### Exercise 2 — medium (apply it)
For each, name the anomaly and give the cheapest sufficient fix:
(a) `SELECT stock; if (stock > 0) UPDATE stock = stock - 1` · (b) a report summing two tables in two statements · (c) "at most 3 admins per organisation," implemented as `SELECT count(*)` then `INSERT` · (d) `SELECT version; UPDATE … WHERE id=$1` · (e) "a seat may not be double-booked," implemented as `SELECT` then `INSERT` · (f) two SELECTs of the same row in one transaction that must agree

For (c) and (e), show why REPEATABLE READ is insufficient and give a fix that works at READ COMMITTED.

### Exercise 3 — hard (production simulation)
A booking platform reports three bugs: 47 double-booked rooms over 11 months, a nightly revenue report that doesn't reconcile, and 41 accounts with negative loyalty points. A fourth requirement — "confirmed bookings must not exceed room count" — has no obvious constraint.

(a) Classify each of the four as a named anomaly, with the two-transaction timeline.
(b) Write the detection query for each.
(c) For each, give the cheapest sufficient fix and explain why the more expensive options are unnecessary.
(d) One of the four genuinely needs SERIALIZABLE. Identify it and explain precisely why no constraint can express it.
(e) For that one, compare SERIALIZABLE against a per-hotel-per-night counter row. Give the trade-off in throughput and contention terms, referencing case study 01.
(f) Measure the throughput cost of setting SERIALIZABLE globally versus per-handler.
(g) Write the retry helper: which SQLSTATEs, what backoff, what bound, what metric.
(h) Write the code-review checklist that catches all four shapes in a pull request.

---

## Mental model checkpoint

1. Name the two gaps that cause all six anomalies, and which anomalies come from each.
2. Which anomalies does PostgreSQL's READ COMMITTED permit? Which three are not in the SQL standard?
3. Why can PostgreSQL never produce a dirty read, at any isolation level?
4. Explain the difference between a non-repeatable read and a phantom read.
5. Explain why write skew survives REPEATABLE READ. What is the conflict actually between?
6. Give three ways to prevent write skew, in order of cost. Which works at READ COMMITTED?
7. What is SQLSTATE 40001, and what must the application do about it?

---

## Quick reference card

| Anomaly | Shape | Prevented by |
|---|---|---|
| Dirty read | read uncommitted | ★ impossible in PostgreSQL |
| Non-repeatable read | same row, two values | REPEATABLE READ |
| Phantom read | same query, new rows | ★ REPEATABLE READ (PG is stronger than the standard) |
| **Lost update** | read-modify-write, same row | ★ **atomic statement** · `FOR UPDATE` · REPEATABLE READ |
| **Read skew** | two rows, impossible combination | REPEATABLE READ · one statement |
| **Write skew** | disjoint writes, shared invariant | ★ **`EXCLUDE`/`UNIQUE`/counter row** · SERIALIZABLE. **Not** REPEATABLE READ |

**READ COMMITTED (the default) permits five of the six.**

**The fix hierarchy — cheapest first**

```sql
-- ① atomic statement (lost update) — free
UPDATE t SET x = x - $1 WHERE id = $2 AND x >= $1;

-- ② materialise the conflict (write skew) — free at runtime
EXCLUDE USING gist (room_id WITH =, during WITH &&) WHERE (status <> 'cancelled')

-- ③ FOR UPDATE — costs contention
-- ④ REPEATABLE READ — costs retries (40001)
-- ⑤ SERIALIZABLE — costs more retries; the only answer to aggregate write skew
```

**Retryable SQLSTATEs:** `40001` (serialization failure) · `40P01` (deadlock). Everything else is not.

**The rule:** name the anomaly, then pick the cheapest sufficient fix. Set the level **per transaction**, never globally.

---

## When would I use this at work?

1. **Any bug report of the form "sometimes the number is wrong."** Matching the symptom to one of six named anomalies turns an unreproducible ghost into a specific timeline and a specific fix.

2. **Code review.** `SELECT count(*)` followed by an `INSERT` is write skew on sight; `SELECT x` followed by `UPDATE x` is a lost update. Both are recognisable in seconds and both have free fixes.

3. **When someone proposes SERIALIZABLE globally.** Naming the anomaly usually reveals a constraint-shaped fix that is cheaper, needs no retries, and cannot be bypassed by a code path someone forgets.

---

## Connected topics

**Understand before this:** 39 (transactions), 40 (ACID's I), 24 (`UNIQUE`, `EXCLUDE` — the materialised-conflict fixes).

**This unlocks:**
- **44** — isolation levels: the dial, and exactly what each setting buys
- **45** — locks: `FOR UPDATE` and how write-write conflicts are detected
- **46** — MVCC: snapshots, and why readers never block writers
- **49** — optimistic vs pessimistic concurrency in application code
- **50** — SSI: how SERIALIZABLE detects write skew without locking everything
- **Case study 02** — the `EXCLUDE` constraint as a production write-skew fix
