# 39 — What a Transaction Is
## Phase: Transactions & Concurrency

---

## ELI5 — The Simple Analogy

You are moving ₹500 from your savings account to your current account.

The bank does two things: **subtract ₹500 from savings**, then **add ₹500 to current**. Between those two moments, the ₹500 exists nowhere. It has left one account and not yet arrived in the other.

Now the power fails. Or the clerk faints. Or two people are doing this at once and the second one reads the balance in that gap.

The bank's answer is a rule: **the two steps are one act.** Either both happened or neither did. There is no moment, visible to anyone, where the money is missing. If the clerk faints halfway, the ledger is torn up and nothing happened.

That's a transaction. And notice what it is *not*: it is not a speed optimisation, not a locking mechanism, not something you use only for money. It is a boundary you draw around a group of changes so that **the outside world only ever sees the before or the after — never the middle.**

---

## Where this fits in the big picture

```
   PHASE 1–4 — storage, indexes, design, normalisation
       (all single-threaded reasoning: one writer, correct data)
                          │
                          ▼
        ┌──────────────────────────────────────────┐
        │ PHASE 5 — TRANSACTIONS & CONCURRENCY     │
        │ 39 WHAT A TRANSACTION IS ← YOU ARE HERE  │
        │    the unit of work                      │
        └────────────────────┬─────────────────────┘
                             ▼
              40 ACID · 41 WAL · 42 recovery
              43 anomalies · 44 isolation · 45 locks
              46 MVCC · 47 VACUUM · 48 deadlocks
              49 optimistic/pessimistic · 50 SSI
              51 2PC · 52 idempotency
```

Everything up to here assumed **one thing happening at a time**. Phase 5 removes that assumption, and this topic defines the unit that makes it survivable.

---

## What is this?

A **transaction** is a sequence of database operations treated as a single, indivisible unit of work. It either **commits** (all changes become permanent and visible) or **aborts/rolls back** (no change is visible, as if it never ran).

In PostgreSQL:

```sql
BEGIN;
  UPDATE accounts SET balance = balance - 50000 WHERE id = 1;
  UPDATE accounts SET balance = balance + 50000 WHERE id = 2;
COMMIT;   -- or ROLLBACK
```

And the part people miss: **every statement is already in a transaction.** A bare `UPDATE` runs inside an implicit single-statement transaction. There is no such thing as a database operation outside one.

---

## Why does it matter for a backend developer?

Because the transaction boundary is one of the few things the database *cannot* infer for you, and getting it wrong produces four distinct classes of production incident:

```
 ① TOO WIDE — the boundary includes something it shouldn't
    BEGIN; UPDATE …; await paymentGateway.charge(); COMMIT;
    ⇒ a 3-second HTTP call holds a row lock, a connection, and a
      snapshot. One slow gateway becomes a total outage. (Topic 09.)

 ② TOO NARROW — two things that must be atomic aren't
    await db.query('UPDATE accounts SET balance = balance - 500 …');
    await db.query('UPDATE accounts SET balance = balance + 500 …');
    ⇒ two implicit transactions. If the second fails, ₹500 vanished.

 ③ MISSING ENTIRELY — the ORM's default
    Many ORMs autocommit each statement. A "save the order and its
    lines" operation is N separate transactions.
    ⇒ orders with missing lines, discovered months later.

 ④ ★ LEFT OPEN — the quiet one
    A connection sits `idle in transaction` because an exception path
    forgot to ROLLBACK.
    ⇒ it holds locks, and it holds a SNAPSHOT — which blocks VACUUM
      for the ENTIRE CLUSTER (Topic 47). One forgotten transaction
      bloats every table in the database.
```

Every one of these is a boundary decision, made in application code, that no amount of schema design can fix.

---

## The physical reality

### What `BEGIN` actually does — almost nothing

```
 BEGIN;
 ⇒ PostgreSQL does NOT:
     • allocate a transaction ID
     • write anything to disk
     • take any lock
     • touch the WAL

 ⇒ It DOES:
     • set the backend's transaction state to TBLOCK_BEGIN
     • assign a VIRTUAL transaction id (backendID, localXID)
   ⇒ ★ BEGIN IS ESSENTIALLY FREE. ~1 microsecond.

 ★ A REAL TRANSACTION ID (XID) IS ASSIGNED LAZILY — on the first
   WRITE. A read-only transaction NEVER consumes one.
   ⇒ WHY THIS MATTERS: XIDs are a finite 32-bit resource (Topic 47's
     wraparound problem). Read-only transactions being free is what
     makes a read-heavy workload sustainable.

 ⚠ BUT: a read-only transaction still holds a SNAPSHOT from its first
   query. And a snapshot is what blocks VACUUM — not the XID.
   ⇒ ★ AN IDLE READ-ONLY TRANSACTION IS STILL DANGEROUS.
```

### The transaction's state, in shared memory

```
 THE PROC ARRAY — one entry per backend, in shared memory
 ┌──────────────────────────────────────────────────────────────┐
 │ pid  │ xid    │ xmin   │ vxid      │ state                   │
 ├──────┼────────┼────────┼───────────┼─────────────────────────┤
 │ 8891 │ 91012  │ 91004  │ 3/41022   │ active                  │
 │ 8892 │ 0      │ 91004  │ 4/41023   │ idle in transaction  ⚠  │
 │ 8893 │ 91014  │ 91012  │ 5/41024   │ active                  │
 └──────┴────────┴────────┴───────────┴─────────────────────────┘
    │        │        │
    │        │        └── xmin: the OLDEST xid this backend's snapshot
    │        │            can still see. ★ VACUUM cannot remove any
    │        │            tuple newer than min(xmin) ACROSS THE CLUSTER.
    │        └── xid: 0 until the first write
    └── the backend process

 ⇒ pid 8892 has NO xid (it hasn't written) but DOES have an xmin.
   ⇒ ★ it is blocking VACUUM cluster-wide while doing nothing.
```

### `pg_xact` — where commit status lives

```
 pg_xact/0000        (formerly pg_clog)
 ┌──────────────────────────────────────────────────────────────┐
 │ TWO BITS PER TRANSACTION:                                    │
 │   00 = IN PROGRESS                                           │
 │   01 = COMMITTED                                             │
 │   10 = ABORTED                                               │
 │   11 = SUB-COMMITTED (a subtransaction)                      │
 └──────────────────────────────────────────────────────────────┘
   32,768 transactions per 8 KB page. A 1 GB file covers ~4 billion.

 ★ THE ASYMMETRY THAT DEFINES COMMIT AND ROLLBACK:

   COMMIT:   write the WAL commit record, fsync it, set the bit to 01.
             ⇒ WORK PROPORTIONAL TO THE WAL, NOT TO THE ROWS CHANGED.

   ROLLBACK: set the bit to 10.  ★ THAT IS ALL.
             ⇒ The tuples written by the aborted transaction STAY ON
               DISK. They are simply never visible, because every
               reader checks pg_xact and sees "aborted".
             ⇒ ★ ROLLBACK IS O(1). It does not undo anything.
               VACUUM cleans up the debris later (Topic 47).

 ⇒ THIS IS WHY POSTGRESQL HAS NO UNDO LOG. Oracle and MySQL/InnoDB
   store old row versions in a rollback segment and must UNDO on abort.
   PostgreSQL writes new versions in the heap and marks the transaction
   dead. Different trade: cheap rollback, expensive VACUUM.
```

---

## How it works — step by step

### The complete lifecycle

```
 1. BEGIN
      state → TBLOCK_BEGIN. A virtual xid is assigned.
      NO xid, NO disk write, NO lock.                    ~1 µs

 2. FIRST STATEMENT (a read)
      A SNAPSHOT is taken from the proc array (Topic 46):
        { xmin: 91004, xmax: 91011, xip: [91007, 91009] }
      ★ THE SNAPSHOT IS THE TRANSACTION'S VIEW OF THE WORLD.
      Under READ COMMITTED a new snapshot is taken per STATEMENT.
      Under REPEATABLE READ the FIRST snapshot is kept for the whole
      transaction. (Topic 44.)

 3. FIRST WRITE
      GetNewTransactionId() → 91012. Recorded in the proc array.
      ★ This is the moment the transaction becomes "real" to others.

 4. EACH SUBSEQUENT STATEMENT
      • takes locks (Topic 45)
      • writes WAL records to the WAL buffer (RAM)
      • modifies pages in the buffer pool (RAM)
      • assigns a COMMAND ID (cid) so the transaction can see its own
        earlier changes but not its own current statement's changes

 5. COMMIT
      a) write a commit WAL record
      b) ★ fsync() the WAL up to that LSN     ← THE DURABILITY POINT
      c) set pg_xact[91012] = COMMITTED
      d) remove the xid from the proc array   ← now visible to new snapshots
      e) release all locks
      ⇒ 0.3–1.0 ms, dominated by (b)

 5'. ROLLBACK
      a) set pg_xact[91012] = ABORTED
      b) release all locks
      ⇒ ★ ~10 µs. No fsync needed — an aborted transaction that is
        lost on crash is indistinguishable from one that aborted.
```

### Savepoints — transactions inside transactions

```sql
BEGIN;
  INSERT INTO orders (customer_id) VALUES (7);
  SAVEPOINT after_order;
  INSERT INTO order_lines (order_id, product_id) VALUES (1, 999999);
  -- ERROR: violates foreign key constraint
  ROLLBACK TO SAVEPOINT after_order;      -- ★ the order survives
  INSERT INTO order_lines (order_id, product_id) VALUES (1, 101);
COMMIT;
```

```
 ★ HOW SAVEPOINTS WORK: each one starts a SUBTRANSACTION with its
   own XID. `ROLLBACK TO` marks that subtransaction aborted while the
   parent continues.

 ⚠ THE COSTS PEOPLE DON'T KNOW:
   ① EACH SAVEPOINT CONSUMES AN XID. A loop with a savepoint per
      iteration burns XIDs at the loop's rate — and XIDs are finite
      (Topic 47's wraparound).
   ② More than 64 subtransactions in one transaction OVERFLOWS the
      per-backend cache, and every visibility check for that
      transaction then hits pg_subtrans on disk.
      ⇒ ★ `SubtransSLRU` wait events, and a cluster-wide slowdown.
   ⇒ SAVEPOINTS ARE NOT FREE. Use them for genuine partial rollback,
     not as a per-row error handler.

 ★ THE CRITICAL POSTGRESQL BEHAVIOUR:
   ANY error inside a transaction aborts the WHOLE transaction.
   Every subsequent statement fails with:
     ERROR: current transaction is aborted, commands ignored until
            end of transaction block
   ⇒ unless you had a savepoint. This differs from MySQL, where a
     failed statement leaves the transaction usable.
```

### Transaction boundaries in Node.js — the shapes that work

```js
// ✗ WRONG — two implicit transactions. Not atomic.
await pool.query('UPDATE accounts SET balance = balance - $1 WHERE id=$2', [500, 1]);
await pool.query('UPDATE accounts SET balance = balance + $1 WHERE id=$2', [500, 2]);

// ✗ WRONG — a pool has many connections. BEGIN and COMMIT may land
//   on DIFFERENT ones. This is a real, common bug.
await pool.query('BEGIN');
await pool.query('UPDATE …');
await pool.query('COMMIT');

// ✓ CORRECT — one connection, checked out for the whole transaction
const client = await pool.connect();
try {
  await client.query('BEGIN');
  await client.query('UPDATE accounts SET balance = balance - $1 WHERE id=$2', [500, 1]);
  await client.query('UPDATE accounts SET balance = balance + $1 WHERE id=$2', [500, 2]);
  await client.query('COMMIT');
} catch (e) {
  await client.query('ROLLBACK');    // ★ must not throw — see below
  throw e;
} finally {
  client.release();                  // ★ ALWAYS, or the pool leaks
}

// ✓ BETTER — a helper that cannot be misused
async function withTransaction(fn) {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');
    const result = await fn(client);
    await client.query('COMMIT');
    return result;
  } catch (e) {
    // ★ if the connection is already broken, ROLLBACK throws too.
    //   Swallow that error, or you lose the original one.
    await client.query('ROLLBACK').catch(() => {});
    throw e;
  } finally {
    client.release();
  }
}
```

### What belongs inside the boundary — the rule

```
 ★★★ THE TRANSACTION BOUNDARY CONTAINS EXACTLY THE WRITES THAT MUST
     BE ATOMIC, AND NOTHING ELSE. ★★★

 INSIDE:
   ✓ the writes that must all succeed or all fail
   ✓ the reads those writes depend on (for a consistency check)
   ✓ the outbox insert that must be atomic with them (Topic 52)

 OUTSIDE — and this list is the source of most incidents:
   ✗ HTTP calls to anything you don't control
   ✗ payment gateways, email, SMS, push notifications
   ✗ S3 / object storage uploads
   ✗ queue publishes (use the outbox instead — Topic 52)
   ✗ cache writes
   ✗ any `await` on something without a timeout you set
   ✗ user interaction of any kind
   ✗ expensive pure computation (do it before BEGIN)

 ★ THE TEST: "if this step takes 30 seconds, what is held?"
   A row lock · a connection · a snapshot blocking VACUUM cluster-wide.
   ⇒ if the answer is uncomfortable, it belongs outside.
```

---

## Concept breakdown

```
TRANSACTION
│  └── an indivisible unit of work: all of it commits, or none of it.
│      ★ every statement is already in one (implicit if you don't
│        write BEGIN).
│
├── BEGIN         ~free. No xid, no disk write, no lock.
├── XID           assigned LAZILY on the first WRITE. Read-only
│                 transactions never consume one.
├── SNAPSHOT      taken on the first query. ★ THIS is what blocks
│                 VACUUM — not the xid.
├── COMMIT        WAL record + fsync + pg_xact bit. 0.3–1 ms.
│                 ★ cost is proportional to WAL, not to rows changed.
└── ROLLBACK      ★ ONE BIT IN pg_xact. O(1). Nothing is undone;
                  the dead tuples are cleaned by VACUUM later.

SAVEPOINT
├── starts a SUBTRANSACTION with its own XID
├── ⚠ consumes an XID each
├── ⚠ > 64 per transaction overflows the cache → pg_subtrans on disk
└── ★ in PostgreSQL, ANY error aborts the whole transaction unless a
     savepoint catches it

THE FOUR BOUNDARY FAILURES
├── TOO WIDE     an external call inside → locks held for seconds
├── TOO NARROW   two atomic operations in two transactions
├── MISSING      ORM autocommit, N separate transactions
└── ★ LEFT OPEN  idle in transaction → blocks VACUUM CLUSTER-WIDE

★ THE RULE
  The boundary contains exactly the writes that must be atomic.
  Nothing you don't control goes inside it.
  Test: "if this step took 30 seconds, what would be held?"
```

---

## Diagrams

**Diagram 1 — big picture: the lifecycle**

```
  BEGIN                                                    COMMIT
    │                                                         │
    │  ~1 µs        first read        first write             │ 0.3–1 ms
    ▼                   │                  │                  ▼
 ┌──────┐          ┌────────┐        ┌──────────┐      ┌────────────┐
 │ vxid │          │SNAPSHOT│        │ XID      │      │ WAL record │
 │ only │          │ taken  │        │ assigned │      │ + fsync ★  │
 │      │          │        │        │          │      │ + pg_xact  │
 │ no   │          │ ★ this │        │ ★ now    │      │ + locks    │
 │ disk │          │  blocks│        │  visible │      │   released │
 │ no   │          │  VACUUM│        │  to      │      │            │
 │ lock │          │        │        │  others  │      │            │
 └──────┘          └────────┘        └──────────┘      └────────────┘

  ROLLBACK instead:
                                                       ┌────────────┐
                                                       │ ONE BIT    │
                                                       │ pg_xact=10 │
                                                       │ ~10 µs     │
                                                       │ ★ nothing  │
                                                       │   is undone│
                                                       └────────────┘
```

**Diagram 2 — data flow: what a too-wide boundary holds**

```
  BEGIN
    │
    ├─ UPDATE products SET stock = stock-1 WHERE id=88
    │     ★ takes a ROW LOCK on product 88 (Topic 45)
    │     ★ holds a CONNECTION
    │     ★ holds a SNAPSHOT
    │
    ├─ await gateway.charge()  ─────────────── 3,000 ms ───────────┐
    │                                                              │
    │   MEANWHILE:                                                 │
    │     • every other buyer of product 88 BLOCKS                 │
    │     • one pool connection is unavailable                     │
    │     • ★ VACUUM cannot remove ANY dead tuple newer than this  │
    │       snapshot — IN ANY TABLE, IN THE WHOLE CLUSTER          │
    │                                                              │
    ├─ INSERT INTO orders … ◀──────────────────────────────────────┘
    │
  COMMIT

  ⇒ ★ A 3-SECOND EXTERNAL CALL BECAME A 3-SECOND CLUSTER-WIDE
    VACUUM STALL AND A 3-SECOND LOCK ON A HOT ROW.
```

**Diagram 3 — before/after: restructuring the boundary**

```
 ✗ ONE WIDE TRANSACTION
 ┌──────────────────────────────────────────────────────────────────┐
 │ BEGIN ─── reserve ─── [ GATEWAY: 3,000 ms ] ─── confirm ─── COMMIT│
 │       └──────────────── locks held 3,004 ms ────────────────┘     │
 │ throughput ceiling: 1000/3004 ≈ 0.33 txn/s per contended row      │
 └──────────────────────────────────────────────────────────────────┘

 ✓ THREE PHASES, TWO SHORT TRANSACTIONS
 ┌──────────────────────────────────────────────────────────────────┐
 │ BEGIN ── reserve ── COMMIT        [ GATEWAY: 3,000 ms ]           │
 │      └── 2 ms ──┘                  ★ NOTHING HELD                 │
 │                                                                   │
 │                                    BEGIN ── confirm ── COMMIT     │
 │                                         └──── 2 ms ────┘          │
 │ throughput ceiling: 1000/2 = 500 txn/s per contended row          │
 └──────────────────────────────────────────────────────────────────┘
                          ★ 1,500× more throughput, same work
```

---

## Example 1 — basic

**Step 1 — atomicity, demonstrated.**

```sql
CREATE TABLE accounts (
  id bigint PRIMARY KEY,
  balance_minor bigint NOT NULL CHECK (balance_minor >= 0)
);
INSERT INTO accounts VALUES (1, 100000), (2, 50000);

-- ✓ atomic: both or neither
BEGIN;
  UPDATE accounts SET balance_minor = balance_minor - 50000 WHERE id=1;
  UPDATE accounts SET balance_minor = balance_minor + 50000 WHERE id=2;
COMMIT;
SELECT * FROM accounts;
```
```
 id | balance_minor
----+---------------
  1 |         50000
  2 |        100000
```

```sql
-- now force the second statement to fail
BEGIN;
  UPDATE accounts SET balance_minor = balance_minor - 60000 WHERE id=1;
  UPDATE accounts SET balance_minor = balance_minor - 999999 WHERE id=2;
  -- ERROR: new row for relation "accounts" violates check constraint
COMMIT;
-- ROLLBACK  (PostgreSQL turns COMMIT into ROLLBACK after an error)
SELECT * FROM accounts;
```
```
 id | balance_minor
----+---------------
  1 |         50000        ★ UNCHANGED — the first UPDATE was undone
  2 |        100000
```

**Step 2 — the XID is assigned lazily.**

```sql
BEGIN;
SELECT txid_current_if_assigned();     -- NULL — read-only so far
SELECT count(*) FROM accounts;
SELECT txid_current_if_assigned();     -- still NULL ★
UPDATE accounts SET balance_minor = balance_minor WHERE id=1;
SELECT txid_current_if_assigned();     -- 91012 — assigned on the write
ROLLBACK;
```

**Step 3 — a read-only transaction still blocks VACUUM.**

```sql
-- session 1
BEGIN;
SELECT count(*) FROM accounts;   -- takes a snapshot
-- do nothing else. Do not commit.

-- session 2
SELECT pid, state, xact_start,
       txid_current_if_assigned() IS NULL AS no_xid,
       backend_xmin
FROM pg_stat_activity WHERE state = 'idle in transaction';
```
```
 pid  |        state        | no_xid | backend_xmin
------+---------------------+--------+--------------
 8892 | idle in transaction | t      |        91004
```
```sql
-- ★ backend_xmin is set. VACUUM cannot remove anything newer than it.
SELECT relname, n_dead_tup FROM pg_stat_user_tables ORDER BY 2 DESC LIMIT 3;
VACUUM VERBOSE accounts;
-- NOTICE: … 0 dead row versions cannot be removed yet, oldest xmin: 91004
--   ★ IN ANY TABLE. A read-only idle transaction stalls the cluster.
```

**Step 4 — rollback is O(1).**

```sql
CREATE TABLE big_rollback (id bigint, pad text);
\timing on
BEGIN;
INSERT INTO big_rollback SELECT i, repeat('x',100) FROM generate_series(1,2000000) i;
-- Time: 4,182 ms
ROLLBACK;
-- Time: 1.204 ms       ★ TWO MILLION ROWS UNDONE IN 1 MILLISECOND
```
```sql
-- and the rows are still physically there
SELECT pg_size_pretty(pg_relation_size('big_rollback'));   -- 262 MB
SELECT count(*) FROM big_rollback;                          -- 0
-- ★ 262 MB of dead tuples. VACUUM will reclaim them (Topic 47).
```

**Step 5 — savepoints, and the error behaviour.**

```sql
BEGIN;
  INSERT INTO accounts VALUES (3, 1000);
  SELECT 1/0;
  -- ERROR: division by zero
  SELECT * FROM accounts;
  -- ERROR: current transaction is aborted, commands ignored until
  --        end of transaction block           ★ THE WHOLE TXN IS DEAD
ROLLBACK;

-- with a savepoint
BEGIN;
  INSERT INTO accounts VALUES (3, 1000);
  SAVEPOINT sp1;
  SELECT 1/0;                                  -- ERROR
  ROLLBACK TO SAVEPOINT sp1;                   -- ★ recover
  SELECT count(*) FROM accounts;               -- 3 — the insert survived
COMMIT;
```

**Step 6 — the subtransaction cost.**

```sql
CREATE TABLE sub_test (id int);
\timing on
-- 1000 savepoints in one transaction
DO $$ BEGIN
  FOR i IN 1..1000 LOOP
    BEGIN
      INSERT INTO sub_test VALUES (i);
    EXCEPTION WHEN OTHERS THEN NULL;   -- ★ each BEGIN…EXCEPTION block
    END;                               --   is an implicit SAVEPOINT
  END LOOP;
END $$;
SELECT txid_current();
```
```
 ⇒ 1,000 XIDs consumed for 1,000 rows.
 ★ AND above 64 subtransactions the per-backend cache overflows; every
   visibility check for this transaction then reads pg_subtrans.
   Watch for `SubtransSLRU` in pg_stat_activity.wait_event.
```

**Step 7 — the pool bug.**

```js
// ✗ THIS IS A REAL BUG PEOPLE SHIP
await pool.query('BEGIN');                     // connection A
await pool.query('UPDATE accounts …');         // ★ maybe connection B!
await pool.query('COMMIT');                    // ★ maybe connection C!
// ⇒ connection A is left `idle in transaction` FOREVER,
//   the UPDATE autocommitted on B, and COMMIT on C did nothing.
```
```sql
-- how to spot it
SELECT pid, state, now()-state_change AS idle_for, left(query,50)
FROM pg_stat_activity WHERE state='idle in transaction'
ORDER BY idle_for DESC;
```

---

## Example 2 — production scenario

**The situation.** A checkout service. p99 is 4.2 s. The DBA reports the database is 8% CPU and mostly idle. Three separate alerts fire in the same week:

1. `orders` table bloat has grown 40% in a fortnight, and autovacuum "runs but doesn't reclaim anything."
2. Connection pool exhaustion at peak — `remaining connection slots reserved`.
3. Support: "customer charged but no order created," 14 cases.

**Step 1 — find the boundary.**

```js
// the handler, as written
async function checkout(req) {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');

    const stock = await client.query(
      'SELECT stock FROM products WHERE id=$1 FOR UPDATE', [req.productId]);
    if (stock.rows[0].stock < 1) throw new Error('OUT_OF_STOCK');

    await client.query('UPDATE products SET stock = stock-1 WHERE id=$1',
                       [req.productId]);

    // ★★★ THE PROBLEM ★★★
    const charge = await razorpay.orders.create({ amount: req.amount });  // 200–3,800 ms
    const capture = await razorpay.payments.capture(charge.id);           // 400–2,100 ms

    await client.query('INSERT INTO orders (…) VALUES (…)', [...]);
    await client.query('INSERT INTO payments (…) VALUES (…)', [...]);
    await kafka.send({ topic: 'order.created', messages: [...] });        // 20–400 ms

    await client.query('COMMIT');
  } catch (e) {
    await client.query('ROLLBACK');
    throw e;
  } finally { client.release(); }
}
```

**Two external calls and a Kafka publish, all inside the transaction.**

**Step 2 — confirm each symptom traces to the boundary.**

```sql
-- ① the bloat
SELECT max(now() - xact_start) AS longest_txn,
       count(*) FILTER (WHERE state='idle in transaction') AS idle_in_txn,
       min(backend_xmin) AS oldest_snapshot
FROM pg_stat_activity WHERE xact_start IS NOT NULL;
```
```
   longest_txn   | idle_in_txn | oldest_snapshot
-----------------+-------------+-----------------
 00:00:06.204118 |          88 |           41022
```
```sql
VACUUM VERBOSE orders;
```
```
INFO:  vacuuming "public.orders"
INFO:  1204882 dead row versions cannot be removed yet, oldest xmin: 41022
```
**★ 1.2 million dead tuples unreclaimable**, because 88 backends are sitting in a transaction waiting on Razorpay. **Every table in the cluster is affected, not just `orders`.**

```sql
-- ② the pool exhaustion
SELECT count(*) AS total,
       count(*) FILTER (WHERE state='idle in transaction') AS stuck,
       (SELECT setting::int FROM pg_settings WHERE name='max_connections') AS max_conn
FROM pg_stat_activity;
```
```
 total | stuck | max_conn
-------+-------+----------
   196 |    88 |      200
```
**88 of 200 connections held by transactions doing nothing but waiting on HTTP.**

```sql
-- ③ the charged-no-order cases
SELECT count(*) FROM payments p
LEFT JOIN orders o ON o.id = p.order_id WHERE o.id IS NULL;
```
```
 count
-------
    14
```
```
 ★ HOW: the gateway succeeded, then the INSERT failed (an FK violation
   from a concurrent product deletion, or the connection dropped after
   the 6-second wait). ROLLBACK undid the database work — but it
   CANNOT undo the charge. The money left the customer's account and
   no order exists.
 ⇒ ★ THIS IS THE FUNDAMENTAL LIMIT: a transaction can only roll back
   what the DATABASE did. Anything outside is not covered.
```

**Step 3 — the restructure: three phases, two short transactions.**

```js
const RESERVATION_TTL = '15 minutes';

async function checkout(req) {
  // ═══ PHASE 1: RESERVE — a short transaction ═══
  const reservation = await withTransaction(async (tx) => {
    await tx.query("SET LOCAL lock_timeout = '2s'");

    // atomic decrement with a guard — no SELECT…FOR UPDATE needed
    const dec = await tx.query(
      `UPDATE products SET stock = stock - 1
        WHERE id = $1 AND stock > 0 RETURNING stock`, [req.productId]);
    if (dec.rowCount === 0) throw new AppError('OUT_OF_STOCK');

    const r = await tx.query(
      `INSERT INTO stock_reservations
         (product_id, user_id, idempotency_key, expires_at, state)
       VALUES ($1,$2,$3, now() + $4::interval, 'held')
       ON CONFLICT (idempotency_key) DO NOTHING
       RETURNING id`,
      [req.productId, req.userId, req.idempotencyKey, RESERVATION_TTL]);

    if (r.rowCount === 0) {                      // ★ a retry of the same request
      const prev = await tx.query(
        'SELECT id, state FROM stock_reservations WHERE idempotency_key=$1',
        [req.idempotencyKey]);
      return { ...prev.rows[0], replayed: true };
    }
    return r.rows[0];
  });
  // ★ transaction closed. Duration: ~2 ms. Nothing held.

  // ═══ PHASE 2: THE EXTERNAL CALL — NO transaction, NO connection ═══
  let charge;
  try {
    charge = await razorpay.orders.create({
      amount: req.amount,
      receipt: `res-${reservation.id}`,          // ★ ties it to the reservation
      notes: { reservation_id: reservation.id },
    });
    await razorpay.payments.capture(charge.id);
  } catch (e) {
    // ⚠ A TIMEOUT IS NOT A FAILURE (case study 03). Do NOT release
    //   the reservation on an unknown outcome — let it expire, and
    //   let the reconciler resolve it.
    if (isDefiniteFailure(e)) await releaseReservation(reservation.id);
    throw e;
  }

  // ═══ PHASE 3: CONFIRM — a short transaction ═══
  return withTransaction(async (tx) => {
    const upd = await tx.query(
      `UPDATE stock_reservations SET state='confirmed'
        WHERE id=$1 AND state='held' AND expires_at > now()
       RETURNING product_id, user_id`, [reservation.id]);

    if (upd.rowCount === 0) {
      // the reservation expired while the gateway was slow
      throw new AppError('RESERVATION_EXPIRED', { refund: charge.id });
    }

    const order = await tx.query(
      `INSERT INTO orders (user_id, product_id, total_minor, status)
       VALUES ($1,$2,$3,'paid') RETURNING id`,
      [upd.rows[0].user_id, upd.rows[0].product_id, req.amount]);

    await tx.query(
      `INSERT INTO payments (order_id, gateway_ref, amount_minor)
       VALUES ($1,$2,$3)`, [order.rows[0].id, charge.id, req.amount]);

    // ★ THE OUTBOX — not a Kafka publish (Topic 52)
    await tx.query(
      `INSERT INTO outbox (topic, payload) VALUES ('order.created', $1)`,
      [JSON.stringify({ orderId: order.rows[0].id })]);

    return { orderId: order.rows[0].id };
  });
  // ★ transaction closed. Duration: ~3 ms.
}
```

**Step 4 — the guardrails, so it cannot recur.**

```sql
-- bound the blast radius at the database level
ALTER ROLE app SET idle_in_transaction_session_timeout = '5s';
ALTER ROLE app SET statement_timeout = '15s';
ALTER ROLE app SET lock_timeout = '3s';
```
```
 ★ idle_in_transaction_session_timeout IS THE IMPORTANT ONE.
   It kills any transaction that sits idle for 5 seconds — which means
   a future "just one quick HTTP call" gets an immediate, loud failure
   instead of silently bloating the cluster for a month.
```

```js
// and in code: a helper that makes the mistake harder
async function withTransaction(fn, { timeoutMs = 5000 } = {}) {
  const client = await pool.connect();
  const timer = setTimeout(() => {
    logger.error({ msg: 'transaction exceeded budget — check for external calls' });
  }, timeoutMs);
  try {
    await client.query('BEGIN');
    await client.query(`SET LOCAL idle_in_transaction_session_timeout = '5s'`);
    const r = await fn(client);
    await client.query('COMMIT');
    return r;
  } catch (e) {
    await client.query('ROLLBACK').catch(() => {});
    throw e;
  } finally {
    clearTimeout(timer);
    client.release();
  }
}
```

**Step 5 — results.**

| | Before | After |
|---|---|---|
| p99 checkout | 4,200 ms | **180 ms** |
| Transaction duration | 3,000–6,000 ms | **2–3 ms** |
| Connections `idle in transaction` | 88 | **0** |
| Dead tuples unreclaimable | 1,204,882 | **~0** |
| `orders` bloat growth | +40%/fortnight | **flat** |
| Charged-but-no-order | 14 | **0** — reservations + reconciler |
| Throughput per contended product | 0.33/s | **~500/s** |

**Not one query was optimised.** The entire fix was moving three operations outside the transaction boundary.

---

## Common mistakes

**1. An external call inside a transaction.**
- *Symptom:* pool exhaustion, cluster-wide bloat, lock waits, all at once.
- *Engine-level why:* the transaction holds a row lock, a connection, and a snapshot for the duration. The snapshot blocks VACUUM in **every** table.
- *Fix:* reserve → external call → confirm. Set `idle_in_transaction_session_timeout`.

**2. `pool.query('BEGIN')`.**
- *Symptom:* a connection permanently `idle in transaction`; statements autocommitting.
- *Engine-level why:* a pool hands out a different connection per query. `BEGIN`, the work, and `COMMIT` land on three different sessions.
- *Fix:* `pool.connect()`, use that client for everything, `release()` in `finally`.

**3. Forgetting `client.release()`.**
- *Symptom:* the pool slowly exhausts under error conditions.
- *Fix:* `finally`, always. Or a `withTransaction` helper that cannot be misused.

**4. `ROLLBACK` throwing and masking the real error.**
- *Symptom:* the logs show a connection error, not the constraint violation that caused it.
- *Engine-level why:* if the connection is already broken, `ROLLBACK` fails too.
- *Fix:* `await client.query('ROLLBACK').catch(() => {})`.

**5. A savepoint per loop iteration.**
- *Symptom:* XID consumption spikes; `SubtransSLRU` wait events; cluster-wide slowdown.
- *Engine-level why:* each savepoint is a subtransaction with its own XID, and >64 overflows the per-backend cache.
- *Fix:* handle errors before the loop, or batch. Savepoints are for genuine partial rollback.

**6. Expecting a failed statement to leave the transaction usable.**
- *Symptom:* `current transaction is aborted, commands ignored`.
- *Engine-level why:* PostgreSQL aborts the whole transaction on any error, unlike MySQL.
- *Fix:* a savepoint if you genuinely need to continue; otherwise restructure.

**7. Thinking `ROLLBACK` undoes external effects.**
- *Symptom:* "charged but no order."
- *Engine-level why:* a transaction covers database work only.
- *Fix:* nothing outside the database goes inside the boundary. Use reservations and an outbox (Topics 52, case study 03).

---

## Hands-on proof

**PROVE IT #1 — atomicity.** (Example 1, step 1.)
**PROVE IT #2 — the XID is lazy.** (Example 1, step 2.)
**PROVE IT #3 — a read-only idle transaction blocks VACUUM.** (Example 1, step 3.)
**PROVE IT #4 — rollback is O(1).** (Example 1, step 4.)
**PROVE IT #5 — savepoints and the aborted-transaction rule.** (Example 1, step 5.)

**PROVE IT #6 — find long or idle transactions right now.**
```sql
SELECT pid, state,
       now() - xact_start  AS txn_age,
       now() - state_change AS in_state_for,
       backend_xmin, wait_event_type, wait_event,
       left(query, 60) AS last_query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL AND state <> 'idle'
ORDER BY xact_start LIMIT 10;
```

**PROVE IT #7 — quantify the VACUUM damage from one open transaction.**
```sql
-- session 1: BEGIN; SELECT 1; (leave it open)
-- session 2:
SELECT min(backend_xmin) AS oldest_snapshot FROM pg_stat_activity;
SELECT sum(n_dead_tup) AS unreclaimable_dead_tuples FROM pg_stat_user_tables;
VACUUM VERBOSE some_busy_table;
-- ★ "N dead row versions cannot be removed yet, oldest xmin: …"
```

**PROVE IT #8 — the guardrail works.**
```sql
SET idle_in_transaction_session_timeout = '3s';
BEGIN;
SELECT 1;
-- wait 4 seconds…
SELECT 1;
-- FATAL: terminating connection due to idle-in-transaction timeout ✓
```

---

## The design decision framework

```
★★★ THE BOUNDARY CONTAINS EXACTLY THE WRITES THAT MUST BE ATOMIC. ★★★

 FOR EVERY OPERATION, ASK: "MUST THESE SUCCEED OR FAIL TOGETHER?"
   YES → one transaction
   NO  → separate transactions, and design the intermediate state

 THEN, FOR EVERY STEP INSIDE THE BOUNDARY, ASK:
   "IF THIS TOOK 30 SECONDS, WHAT WOULD BE HELD?"
     a row lock         → every other writer of that row blocks
     a connection       → one fewer for the whole application
     ★ a snapshot       → VACUUM stalls in EVERY table, cluster-wide
   ⇒ if any of those is unacceptable, the step goes OUTSIDE.

 NEVER INSIDE A TRANSACTION:
   ✗ HTTP calls · payment gateways · email/SMS/push
   ✗ object-storage uploads · queue publishes (use the outbox — T52)
   ✗ cache writes · user interaction
   ✗ any await without a timeout you control
   ✗ expensive computation (do it before BEGIN)

 THE THREE-PHASE PATTERN, for anything involving an external system:
   ① RESERVE  — a short transaction that claims the resource
   ② EXTERNAL — no transaction, no connection held
   ③ CONFIRM  — a short transaction that finalises
   + a TTL on the reservation and a sweeper for abandoned ones
   + ★ treat a TIMEOUT as UNKNOWN, not failure (case study 03)

 GUARDRAILS — set these once, at the role level:
   idle_in_transaction_session_timeout = '5s'   ★ the important one
   statement_timeout = '15s'
   lock_timeout = '3s'
   ⇒ so the next "just one quick HTTP call" fails loudly and
     immediately instead of bloating the cluster for a month.

 SAVEPOINTS:
   ✓ genuine partial rollback (one line of a multi-line import)
   ✗ a per-row error handler in a loop — each costs an XID, and
     >64 overflows the subtransaction cache

THE SIGNAL TO LOOK FOR:
      SELECT max(now()-xact_start) AS longest,
             count(*) FILTER (WHERE state='idle in transaction') AS idle
      FROM pg_stat_activity;
  • longest > 1 s on an OLTP workload → an external call is inside
  • idle > 0 for more than a moment   → a boundary bug or a missing
                                        release
  • and confirm the damage:
      VACUUM VERBOSE <any busy table>;
      "N dead row versions cannot be removed yet, oldest xmin: …"
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
Create the `accounts` table. Demonstrate: (a) an atomic two-statement transfer, (b) a failed transfer leaving both balances unchanged, (c) that the XID is assigned only on the first write, (d) that `ROLLBACK` of a 1M-row insert takes ~1 ms while the pages remain allocated. Explain (d) in terms of `pg_xact`.

### Exercise 2 — medium (apply it)
Open a read-only transaction and leave it idle. Then:
(a) show via `pg_stat_activity` that it has no XID but does have a `backend_xmin`,
(b) create dead tuples in an unrelated table and show `VACUUM VERBOSE` cannot remove them,
(c) explain precisely why a *read-only* transaction can block VACUUM,
(d) set `idle_in_transaction_session_timeout` and show the connection being terminated,
(e) write the monitoring query and alert threshold you would deploy.

### Exercise 3 — hard (production simulation)
A checkout handler wraps two payment-gateway calls (200–3,800 ms) and a Kafka publish inside one transaction. Symptoms: p99 4.2 s, 88 of 200 connections `idle in transaction`, 1.2M unreclaimable dead tuples cluster-wide, `orders` bloat +40% per fortnight, and 14 "charged but no order" support cases.

(a) Explain how a single boundary decision produces all four symptoms. Trace each to the specific resource held.
(b) Write the queries that diagnose each symptom.
(c) Explain why `ROLLBACK` could not prevent the 14 charged-no-order cases, and what the fundamental limit is.
(d) Restructure into three phases. Show the SQL and the transaction boundaries.
(e) A gateway call times out. Explain why releasing the reservation would be **wrong**, and what to do instead.
(f) Give the database-level guardrails and explain what each prevents.
(g) Compute the throughput ceiling per contended row before and after, showing your working.
(h) Write the CI check or code-review rule that prevents an `await` on a non-database call from being added inside a transaction.

---

## Mental model checkpoint

1. What does `BEGIN` actually do? What does it *not* do?
2. When is an XID assigned, and why does that matter for a read-heavy workload?
3. Why is `ROLLBACK` O(1) in PostgreSQL? What is the consequence?
4. A read-only transaction has no XID. Why is leaving it open still dangerous?
5. Name the four transaction-boundary failure modes and one symptom of each.
6. What does a savepoint cost? What happens above 64 of them?
7. Why can a transaction not roll back a payment-gateway charge, and what pattern handles that?

---

## Quick reference card

| Operation | Cost | Notes |
|---|---|---|
| `BEGIN` | ~1 µs | no XID, no disk, no lock |
| First read | — | **takes a snapshot** → blocks VACUUM |
| First write | — | assigns the XID |
| `COMMIT` | 0.3–1 ms | WAL fsync dominates |
| `ROLLBACK` | ~10 µs | **one bit in `pg_xact`**; nothing undone |
| `SAVEPOINT` | one XID each | >64 → `SubtransSLRU` on disk |

**Never inside a transaction:** HTTP calls · gateways · email/SMS · S3 · queue publishes · cache writes · user interaction · any untimed `await`.

**The three-phase pattern:** reserve (short txn) → external call (no txn) → confirm (short txn) + TTL + sweeper.

**Guardrails**
```sql
ALTER ROLE app SET idle_in_transaction_session_timeout = '5s';
ALTER ROLE app SET statement_timeout = '15s';
ALTER ROLE app SET lock_timeout = '3s';
```

**The Node.js shape**
```js
const client = await pool.connect();       // ★ not pool.query('BEGIN')
try { await client.query('BEGIN'); …; await client.query('COMMIT'); }
catch (e) { await client.query('ROLLBACK').catch(()=>{}); throw e; }
finally { client.release(); }              // ★ always
```

**The test:** *"if this step took 30 seconds, what would be held?"* — a row lock, a connection, and a **cluster-wide VACUUM stall**.

---

## When would I use this at work?

1. **Reviewing any handler that writes to the database.** Scanning for an `await` on something non-database between `BEGIN` and `COMMIT` catches the single most damaging concurrency bug there is, in ten seconds.

2. **Diagnosing unexplained bloat.** `VACUUM VERBOSE` saying "N dead rows cannot be removed yet, oldest xmin" points straight at an open transaction — often a read-only one nobody suspected.

3. **Pool exhaustion under load.** Counting `idle in transaction` sessions distinguishes "we need more connections" from "one handler holds them for 3 seconds each," which are completely different fixes.

---

## Connected topics

**Understand before this:** 09 (the life of a query — the write path this wraps), 02 (the transaction manager component).

**This unlocks:**
- **40** — ACID: what a transaction actually guarantees
- **41–42** — WAL and recovery: how durability is implemented
- **44** — isolation levels: what "the middle" means for concurrent readers
- **45–46** — locks and MVCC: the machinery snapshots depend on
- **47** — VACUUM: why an open transaction is so expensive
- **52** — idempotency and the outbox: the correct pattern for external effects
