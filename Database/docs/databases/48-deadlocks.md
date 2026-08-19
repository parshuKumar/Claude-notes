# 48 — Deadlocks
## Phase: Transactions & Concurrency

---

## ELI5 — The Simple Analogy

Two people cooking in a small kitchen.

Meera picks up the **knife**, then reaches for the **cutting board**.
Arjun picks up the **cutting board**, then reaches for the **knife**.

Neither will put down what they're holding until they finish. Neither can finish. **They will stand there forever.**

Nobody did anything wrong. Each acted reasonably. The problem is entirely in the **order** they picked things up — and it only appears when they happen to overlap.

The fix is not "be careful." The fix is a **house rule**: *the knife is always picked up before the cutting board.* If everyone follows it, a deadlock is not merely unlikely — it is **arithmetically impossible**, because you can never be waiting for something whose owner is waiting for you.

And the thing PostgreSQL does when it happens: after one second of nobody moving, it **taps one person on the shoulder and makes them drop everything and start over**. That's a deadlock detection, and the person who got tapped sees `40P01`.

---

## Where this fits in the big picture

```
   45 locks — what blocks what
   44 isolation levels — 40001 and the retry loop you already owe
                          │
                          ▼
        ┌──────────────────────────────────────────┐
        │ 48 DEADLOCKS ← YOU ARE HERE              │
        │ when waiting becomes a CYCLE             │
        └────────────────────┬─────────────────────┘
                             ▼
              49 optimistic vs pessimistic
              52 idempotency (retries must be safe)
```

Topic 45 established that a lock makes you wait. **This topic is what happens when the waiting graph contains a cycle** — and, more usefully, the design rules that make cycles impossible.

---

## What is this?

A **deadlock** is a cycle in the wait-for graph: transaction A waits for a lock held by B, and B waits (directly or through a chain) for a lock held by A. Neither can proceed, and neither will time out on its own.

PostgreSQL detects it and resolves it by **aborting one transaction**:

```
ERROR:  deadlock detected                                  -- SQLSTATE 40P01
DETAIL:  Process 41202 waits for ShareLock on transaction 8842119;
         blocked by process 41288.
         Process 41288 waits for ShareLock on transaction 8842118;
         blocked by process 41202.
HINT:  See server log for query details.
CONTEXT:  while updating tuple (0,2) in relation "accounts"
```

**Three facts that shape everything else:**

1. **A deadlock is not a bug in the database.** It is a correctness guarantee doing its job. The alternative is hanging forever.
2. **`40P01` is retryable** — like `40001` (Topic 44). The victim's transaction is rolled back cleanly; retrying usually succeeds.
3. ★ **Deadlocks are almost always an *ordering* problem in application code**, not a load problem. Load makes them *visible*; inconsistent ordering makes them *possible*.

---

## Why does it matter for a backend developer?

```
 ★ THE CHARACTERISTIC SIGNATURE — you will meet this exact shape:

   • the error rate is LOW but NON-ZERO (0.1%–3%)
   • it scales with CONCURRENCY, not with data size
   • it is UNREPRODUCIBLE locally (one client, no overlap)
   • the failing statement looks completely innocent
   • ★ it appears the week after a feature that added a second
     UPDATE to an existing handler

 ⇒ AND THE THREE MOST COMMON CAUSES ARE ALL DESIGN CHOICES:
   ① updating multiple rows in a non-deterministic order
      (a bulk UPDATE … WHERE id = ANY($1) with an unsorted array)
   ② ★ lock upgrades — SELECT … FOR SHARE then UPDATE
   ③ two handlers touching the same two tables in opposite orders
```

And the reason it's worth real understanding: **deadlocks are one of the very few production problems with a complete, structural fix.** Not mitigation — elimination. A consistent lock order makes the cycle arithmetically impossible.

---

## The physical reality

### How PostgreSQL detects a cycle

```
 ★ POSTGRESQL DOES NOT CHECK FOR DEADLOCKS ON EVERY LOCK REQUEST.
   That would be far too expensive.

 THE ALGORITHM:
 ① a backend requests a lock that is held ⇒ it sleeps on a semaphore
 ② ★ it sets a timer for deadlock_timeout (DEFAULT 1 SECOND)
 ③ if the lock is granted before the timer fires ⇒ nothing happens.
    ★ NO DEADLOCK CHECK IS EVER RUN. This is the common case, and
      it costs nothing.
 ④ if the timer fires, the backend runs DeadLockCheck():
      • take an exclusive lock on the whole lock manager
      • build the WAIT-FOR GRAPH from every waiting backend
      • depth-first search for a cycle
 ⑤ no cycle found ⇒ go back to sleep, re-arm the timer
    ★ AND ALSO: if log_lock_waits = on, LOG the wait. This is why
      log_lock_waits gives you free forensics — it piggybacks on
      a check that already happened.
 ⑥ cycle found ⇒ choose a victim and abort it with 40P01

 ⇒ ★ CONSEQUENCE: EVERY DEADLOCK COSTS AT LEAST deadlock_timeout
   (1 second) OF LATENCY BEFORE IT IS EVEN DETECTED.
   Two transactions in a 3ms handler will still sit frozen for a
   full second before one is released.
 ⇒ ★ THIS IS WHY LOWERING deadlock_timeout IS USUALLY WRONG:
   it makes the check run on ordinary lock waits too. On a busy
   system the lock-manager lock becomes the bottleneck. Fix the
   ordering instead.
```

### Who gets chosen as the victim

```
 ★ POSTGRESQL PICKS THE TRANSACTION THAT DETECTED THE CYCLE —
   i.e. the one whose deadlock_timeout fired.

 ⇒ THIS IS NOT "the youngest", "the one holding fewest locks", or
   "the one that has done least work". There is no cost heuristic.
 ⇒ ★ IN PRACTICE: the transaction that started waiting FIRST tends
   to be the victim, because its timer fires first.
 ⇒ SO YOU CANNOT PREDICT OR CONTROL THE VICTIM. Both sides of a
   deadlock-prone pair need a retry loop.

 (InnoDB differs: it deliberately picks the transaction that has
  modified the fewest rows, so rollback is cheapest. Another case
  where "same problem, different bill" — Topic 46.)
```

### The wait-for graph, concretely

```
 A DEADLOCK IS A CYCLE. NOTHING MORE.

  ┌───────┐   waits for lock on row 1   ┌───────┐
  │ txn A │ ─────────────────────────►  │ txn B │
  │       │                              │       │
  │ holds │  ◄───────────────────────── │ holds │
  │ row 1 │   waits for lock on row 2   │ row 2 │
  └───────┘                              └───────┘
       ★ A→B→A. A cycle of length 2.

 THREE-WAY CYCLES ARE REAL AND HARDER TO SEE:
  A holds `orders`, wants `inventory`
  B holds `inventory`, wants `payments`
  C holds `payments`, wants `orders`
  ⇒ ★ no two transactions conflict directly. The cycle only exists
    when all three overlap. This is why some deadlocks appear
    once a month and are declared "unreproducible."

 ★ THE KEY THEOREM:
   IF EVERY TRANSACTION ACQUIRES LOCKS IN A GLOBALLY CONSISTENT
   ORDER, A CYCLE IS IMPOSSIBLE.
   PROOF SKETCH: assign every lockable object a number. Every
   transaction only ever waits "upward". A cycle would require a
   transaction to wait for a LOWER number than one it already
   holds — which the rule forbids. ∎
 ⇒ this is not a heuristic. It is a proof. Which is why
   "consistent lock order" is the whole answer.
```

### The lock-upgrade deadlock — the one that is 100% reproducible

```
 ★ THIS DESERVES ITS OWN SECTION BECAUSE IT DEADLOCKS EVERY TIME,
   NOT OCCASIONALLY.

 t1  A: SELECT … WHERE id=1 FOR SHARE;   ✓ granted (shared)
 t2  B: SELECT … WHERE id=1 FOR SHARE;   ✓ granted (shared — compatible!)
 t3  A: UPDATE … WHERE id=1;
        ⇒ needs an EXCLUSIVE row lock
        ⇒ B holds a SHARE lock on it
        ⇒ ★ A WAITS FOR B
 t4  B: UPDATE … WHERE id=1;
        ⇒ needs EXCLUSIVE
        ⇒ A holds SHARE
        ⇒ ★ B WAITS FOR A
 ⇒ CYCLE. Guaranteed. Every single time these two overlap.

 ★ WHY IT'S SO COMMON: `SELECT … FOR SHARE` reads like "be safe,
   take a gentle lock." It is the single most dangerous lock mode
   in the language, because SHARE locks are compatible with each
   other and therefore both sides get in — and then both need to
   upgrade.

 ⇒ ★ THE RULE: NEVER UPGRADE A LOCK. Take the strength you will
   eventually need, at the moment of the first read.
     ✗ SELECT … FOR SHARE;      then UPDATE
     ✓ SELECT … FOR UPDATE;     then UPDATE
     ✓ SELECT … FOR NO KEY UPDATE;  then UPDATE (non-key columns)
     ✓ ★ or no SELECT at all: UPDATE … SET x = x - $1 WHERE …
```

### Foreign keys as an invisible lock source

```
 ★ A DEADLOCK BETWEEN TWO STATEMENTS THAT MENTION DIFFERENT TABLES.

 INSERT INTO order_items (order_id, …) VALUES (…);
   ⇒ the FK check takes ★ FOR KEY SHARE on orders(order_id)
     — a lock on a table the statement never names.

 SO:
 t1  A: INSERT order_items (order_id=7)  ⇒ KEY SHARE on orders(7)
 t2  B: INSERT order_items (order_id=7)  ⇒ KEY SHARE on orders(7) ✓ compatible
 t3  A: UPDATE orders SET status='paid' WHERE id=7
        ⇒ needs stronger than KEY SHARE ⇒ ★ waits for B
 t4  B: UPDATE orders SET status='paid' WHERE id=7
        ⇒ ★ waits for A
 ⇒ ★ A LOCK-UPGRADE DEADLOCK YOU DID NOT WRITE.

 ⇒ THE FIX: update the parent FIRST (a consistent order — parents
   before children), or don't update the parent at all in the same
   transaction as the child insert.
 ⇒ AND: this is why FOR KEY SHARE exists at all (Topic 45) —
   without it, this pattern would deadlock far more often.
```

---

## How it works — step by step

### Reading a deadlock log entry

```
 With log_lock_waits = on and log_min_messages default, the server
 log gives you EVERYTHING:

 ERROR:  deadlock detected
 DETAIL:  Process 41202 waits for ShareLock on transaction 8842119;
          blocked by process 41288.
          Process 41288 waits for ShareLock on transaction 8842118;
          blocked by process 41202.
          Process 41202: UPDATE accounts SET balance_minor = balance_minor - 500
                          WHERE id = 2
          Process 41288: UPDATE accounts SET balance_minor = balance_minor - 300
                          WHERE id = 1
 HINT:  See server log for query details.
 CONTEXT:  while updating tuple (0,1) in relation "accounts"
 STATEMENT:  UPDATE accounts SET balance_minor = balance_minor - 500 WHERE id = 2

 ★ HOW TO READ IT — three things, in order:
   ① THE TWO STATEMENTS. Both are shown. Almost always they are
      the SAME statement with DIFFERENT parameters.
   ② ★ THE PARAMETERS. Here: 41202 is going 1→2, 41288 is going
      2→1. THAT IS THE ENTIRE BUG, visible in one line.
   ③ CONTEXT: names the exact tuple and relation.

 ⇒ ★ SET log_lock_waits = on IN EVERY PRODUCTION DATABASE.
   Without it you get the ERROR but not the statements, and you are
   reduced to guessing.
```

### The five patterns, with their fixes

```
 ① INCONSISTENT ROW ORDER — the classic
    A: UPDATE accounts … id=1;  then id=2
    B: UPDATE accounts … id=2;  then id=1
    ★ FIX: sort. Always.
      ids.sort((a,b) => a - b)
      UPDATE … WHERE id = ANY($1)          -- ★ NOT deterministic!
      ⇒ ★ a multi-row UPDATE locks rows in the order the PLAN
        produces them, which can change with statistics.
      ⇒ to be certain, lock explicitly first:
        SELECT id FROM accounts WHERE id = ANY($1)
         ORDER BY id FOR NO KEY UPDATE;
        then UPDATE.

 ② LOCK UPGRADE — 100% reproducible
    FOR SHARE → UPDATE
    ★ FIX: take FOR UPDATE / FOR NO KEY UPDATE up front, or use a
      single atomic UPDATE.

 ③ INCONSISTENT TABLE ORDER
    handler X: UPDATE inventory; then UPDATE orders
    handler Y: UPDATE orders;    then UPDATE inventory
    ★ FIX: a written, enforced table order for the whole codebase.
      e.g. accounts → orders → order_items → inventory → ledger

 ④ FOREIGN KEY PARENT LOCKS
    INSERT child (takes KEY SHARE on parent), then UPDATE parent
    ★ FIX: update the parent first, or not in the same transaction.

 ⑤ ★ INDEX / UNIQUE CONSTRAINT DEADLOCKS
    two concurrent INSERTs of the same unique key: the second waits
    on the first's uncommitted index entry. Combine with a second
    key inserted in the opposite order ⇒ deadlock.
    Also: UPSERT (ON CONFLICT) with multi-row VALUES in different
    orders between callers.
    ★ FIX: sort the rows in a multi-row INSERT/UPSERT by the
      conflicting key.
```

---

## Concept breakdown

```
WHAT A DEADLOCK IS
└── a CYCLE in the wait-for graph. Nothing more, nothing less.

DETECTION
├── ★ NOT checked on every lock request — only after
│    deadlock_timeout (1s) of waiting
├── DeadLockCheck() builds the wait-for graph, DFS for a cycle
├── ★ every deadlock costs ≥1 SECOND of latency before detection
└── ★ log_lock_waits piggybacks on this check — free forensics

THE VICTIM
└── ★ whoever's timer fired — NOT a cost heuristic
     ⇒ you cannot predict it ⇒ BOTH sides need a retry loop
     (InnoDB picks the cheapest to roll back — different design)

★ THE THEOREM
   a globally consistent lock order makes cycles IMPOSSIBLE
   ⇒ not mitigation. ELIMINATION. With a proof.

THE FIVE PATTERNS
├── ① inconsistent ROW order        → ★ ORDER BY id, always
├── ② ★ LOCK UPGRADE (FOR SHARE→UPDATE) → 100% reproducible
│                                    → never upgrade; take it up front
├── ③ inconsistent TABLE order      → one written order, codebase-wide
├── ④ ★ FK parent locks             → a deadlock between statements
│                                      that name different tables
└── ⑤ unique/index conflicts        → sort multi-row INSERT/UPSERT

★ WHY `UPDATE … WHERE id = ANY($1)` IS NOT SAFE
   row lock order follows the PLAN, which can change with stats.
   ⇒ lock explicitly with ORDER BY … FOR NO KEY UPDATE first.

RETRY — 40P01 IS RETRYABLE, LIKE 40001
├── bounded attempts (3–4)
├── exponential backoff with ★ FULL JITTER (else they re-collide)
├── ★ the retried transaction must be IDEMPOTENT (Topic 52)
└── a metric — a rising rate is a DESIGN signal, not noise

WHAT NOT TO DO
├── ✗ lower deadlock_timeout  — makes the check run on ordinary waits
├── ✗ raise the isolation level — irrelevant; deadlocks occur at
│      every level, including READ COMMITTED
└── ✗ "just retry harder" — retries hide the ordering bug and the
       rate grows with traffic until it's an outage
```

---

## Diagrams

**Diagram 1 — big picture: the cycle, and why ordering breaks it**

```
 ✗ INCONSISTENT ORDER — a cycle is possible
 ┌───────────────────────────────────────────────────────────────┐
 │  transfer(from=1, to=2)          transfer(from=2, to=1)       │
 │                                                                │
 │  t1  A: lock row 1  ✓            t2  B: lock row 2  ✓          │
 │                                                                │
 │  t3  A: lock row 2 ⏸ ───────────────► held by B               │
 │  t4  B: lock row 1 ⏸ ◄─────────────── held by A               │
 │                                                                │
 │        ┌─────┐  waits for  ┌─────┐                            │
 │        │  A  │ ──────────► │  B  │                            │
 │        │     │ ◄────────── │     │   ★ CYCLE                  │
 │        └─────┘  waits for  └─────┘                            │
 │                                                                │
 │  t5  (1 second later) deadlock detected → one aborts, 40P01   │
 └───────────────────────────────────────────────────────────────┘

 ✓ CONSISTENT ORDER — a cycle is IMPOSSIBLE
 ┌───────────────────────────────────────────────────────────────┐
 │  BOTH handlers sort the ids ascending before locking.          │
 │                                                                │
 │  t1  A: lock row 1  ✓            t2  B: lock row 1  ⏸ waits    │
 │  t3  A: lock row 2  ✓                                          │
 │  t4  A: COMMIT                   t5  B: lock row 1  ✓          │
 │                                  t6  B: lock row 2  ✓          │
 │                                  t7  B: COMMIT                 │
 │                                                                │
 │        ┌─────┐  waits for  ┌─────┐                            │
 │        │  B  │ ──────────► │  A  │  ← A waits for nothing      │
 │        └─────┘             └─────┘                            │
 │                                                                │
 │  ★ EVERY WAIT POINTS "UPWARD" IN THE ORDERING.                │
 │    A cycle would require waiting DOWNWARD. Forbidden.          │
 │  ⇒ B simply blocks briefly, then proceeds. Zero errors.       │
 └───────────────────────────────────────────────────────────────┘
```

**Diagram 2 — data flow: the lock-upgrade deadlock**

```
        txn A                                   txn B
 ─────────────────────────────    ─────────────────────────────
 t1  SELECT … id=1 FOR SHARE
       row lock: {A: SHARE}  ✓

 t2                                SELECT … id=1 FOR SHARE
                                     row lock: {A: SHARE,
                                                B: SHARE}  ✓
                                   ★ COMPATIBLE — both get in.
                                     This is the trap.

 t3  UPDATE … id=1
       needs EXCLUSIVE
       B holds SHARE
       ⏸ ★ A WAITS FOR B

 t4                                UPDATE … id=1
                                     needs EXCLUSIVE
                                     A holds SHARE
                                     ⏸ ★ B WAITS FOR A

 t5  ═══════════ deadlock_timeout (1s) ═══════════
     DeadLockCheck() → cycle A→B→A → ERROR 40P01 for one of them

 ★ 100% REPRODUCIBLE. Not a race — a structural certainty whenever
   two transactions run this sequence concurrently.

 ✓ THE FIX — take the final strength immediately:
 t1  SELECT … id=1 FOR UPDATE        row lock: {A: EXCLUSIVE}
 t2  SELECT … id=1 FOR UPDATE        ⏸ B blocks (correctly)
 t3  A: UPDATE; COMMIT               B proceeds
 ⇒ ★ B waited. Nobody errored.
```

**Diagram 3 — before/after: a bulk update with an unsorted array**

```
 ✗ BEFORE — 3.4% of batch jobs fail with 40P01
 ┌───────────────────────────────────────────────────────────────┐
 │ // ids arrive from a Set, a Map, or an API response —          │
 │ // in arbitrary order                                          │
 │ await tx.query(                                                │
 │   'UPDATE inventory SET reserved = reserved + 1 ' +             │
 │   ' WHERE product_id = ANY($1)', [productIds]);                │
 │                                                                │
 │   worker A: [ 88, 12, 41 ]  → locks 88, then 12, then 41       │
 │   worker B: [ 41, 88, 12 ]  → locks 41, then 88 ⏸             │
 │                                                                │
 │   ★ AND WORSE: even with the same array, the lock order is     │
 │     the PLAN's row order, not the array's. A stats change      │
 │     can flip a Bitmap Heap Scan's order overnight.             │
 │                                                                │
 │ MEASURED: 3.4% of batch jobs abort with 40P01                 │
 │           mean job latency 240 ms (★ includes 1s deadlock      │
 │           waits on the failing 3.4%)                           │
 └───────────────────────────────────────────────────────────────┘

 ✓ AFTER — explicit, deterministic lock acquisition
 ┌───────────────────────────────────────────────────────────────┐
 │ const ids = [...new Set(productIds)].sort((a, b) => a - b);    │
 │                                                                │
 │ // ★ acquire ALL locks first, in a guaranteed order            │
 │ await tx.query(                                                │
 │   'SELECT product_id FROM inventory ' +                        │
 │   ' WHERE product_id = ANY($1) ORDER BY product_id ' +          │
 │   ' FOR NO KEY UPDATE', [ids]);                                │
 │                                                                │
 │ // then the update — every lock is already held                │
 │ await tx.query(                                                │
 │   'UPDATE inventory SET reserved = reserved + 1 ' +             │
 │   ' WHERE product_id = ANY($1)', [ids]);                       │
 │                                                                │
 │ MEASURED: 0.00% deadlocks over 14 days                        │
 │           mean job latency 71 ms      ★ 3.4× faster            │
 │           (no 1-second detection waits at all)                 │
 └───────────────────────────────────────────────────────────────┘
```

---

## Example 1 — basic

```sql
CREATE TABLE accounts (
  id bigint PRIMARY KEY,
  owner text NOT NULL,
  balance_minor bigint NOT NULL
);
INSERT INTO accounts VALUES (1,'Meera',100000),(2,'Arjun',100000);
```

**Produce a deadlock.**
```sql
-- session 1                          -- session 2
BEGIN;
UPDATE accounts
  SET balance_minor = balance_minor-500
  WHERE id=1;
                                      BEGIN;
                                      UPDATE accounts
                                        SET balance_minor = balance_minor-300
                                        WHERE id=2;
UPDATE accounts
  SET balance_minor = balance_minor+300
  WHERE id=2;        -- ⏸ waits for session 2
                                      UPDATE accounts
                                        SET balance_minor = balance_minor+500
                                        WHERE id=1;   -- ⏸ waits for session 1
```
After ~1 second, one session gets:
```
ERROR:  deadlock detected
DETAIL:  Process 41288 waits for ShareLock on transaction 8842118; blocked by process 41202.
         Process 41202 waits for ShareLock on transaction 8842119; blocked by process 41288.
HINT:  See server log for query details.
CONTEXT:  while updating tuple (0,1) in relation "accounts"
```
```sql
-- the victim's transaction is already aborted:
SELECT 1;
-- ERROR: current transaction is aborted, commands ignored until end of transaction block
ROLLBACK;
-- the survivor continues normally
COMMIT;
```

**Prove the 1-second cost.**
```sql
\timing on
-- reproduce the above; the victim's failing statement reports:
-- Time: 1004.882 ms       ★ deadlock_timeout, not the query
SHOW deadlock_timeout;   -- 1s
```

**Fix it with a consistent order.**
```sql
-- BOTH sessions now lock the LOWER id first
-- session 1 (1 → 2)                  -- session 2 (2 → 1, but sorted!)
BEGIN;
UPDATE accounts SET balance_minor=balance_minor-500 WHERE id=1;
                                      BEGIN;
                                      UPDATE accounts
                                        SET balance_minor=balance_minor+500
                                        WHERE id=1;   -- ⏸ BLOCKS (correct)
UPDATE accounts SET balance_minor=balance_minor+300 WHERE id=2;
COMMIT;
                                      -- unblocks
                                      UPDATE accounts
                                        SET balance_minor=balance_minor-300
                                        WHERE id=2;
                                      COMMIT;
-- ★ NO ERROR. Session 2 simply waited.
```

**The lock-upgrade deadlock — reproduce it 100% of the time.**
```sql
-- session 1                          -- session 2
BEGIN;
SELECT * FROM accounts WHERE id=1
  FOR SHARE;                          -- ✓
                                      BEGIN;
                                      SELECT * FROM accounts WHERE id=1
                                        FOR SHARE;   -- ✓ ★ compatible!
UPDATE accounts SET owner='M'
  WHERE id=1;                         -- ⏸
                                      UPDATE accounts SET owner='A'
                                        WHERE id=1;  -- ⏸
-- ★ ERROR: deadlock detected  — every single time
```
```sql
-- the fix: take FOR UPDATE up front
-- session 1                          -- session 2
BEGIN;
SELECT * FROM accounts WHERE id=1
  FOR UPDATE;                         -- ✓
                                      BEGIN;
                                      SELECT * FROM accounts WHERE id=1
                                        FOR UPDATE;   -- ⏸ blocks, correctly
UPDATE accounts SET owner='M' WHERE id=1;
COMMIT;
                                      -- proceeds. ★ No error.
```

**The foreign-key deadlock — between statements naming different tables.**
```sql
CREATE TABLE orders (id bigint PRIMARY KEY, status text NOT NULL);
CREATE TABLE order_items (
  id bigserial PRIMARY KEY,
  order_id bigint NOT NULL REFERENCES orders(id),
  sku text NOT NULL
);
INSERT INTO orders VALUES (7,'pending');

-- session 1                          -- session 2
BEGIN;
INSERT INTO order_items (order_id, sku)
  VALUES (7, 'KUR-001');
-- ★ takes FOR KEY SHARE on orders(7)
                                      BEGIN;
                                      INSERT INTO order_items (order_id, sku)
                                        VALUES (7, 'KUR-002');
                                      -- ★ also FOR KEY SHARE — compatible
UPDATE orders SET status='paid'
  WHERE id=7;                         -- ⏸ waits for session 2's KEY SHARE
                                      UPDATE orders SET status='paid'
                                        WHERE id=7;   -- ⏸ waits for session 1
-- ★ ERROR: deadlock detected
--   ...between two INSERTs into order_items and two UPDATEs of orders.
```
```sql
-- the fix: update the parent FIRST — a consistent order
BEGIN;
UPDATE orders SET status='paid' WHERE id=7;       -- parent first
INSERT INTO order_items (order_id, sku) VALUES (7,'KUR-001');
COMMIT;
-- ★ no cycle possible: everyone contends on orders(7) first.
```

**Count deadlocks cluster-wide.**
```sql
SELECT datname, deadlocks, xact_commit, xact_rollback,
       round(100.0*deadlocks/nullif(xact_commit+xact_rollback,0), 4) AS deadlock_pct
  FROM pg_stat_database WHERE datname = current_database();
```
```
 datname | deadlocks | xact_commit | xact_rollback | deadlock_pct
---------+-----------+-------------+---------------+--------------
 shop    |      4128 |   188402118 |         88204 |       0.0022
   ★ track the RATE, not the count. A rising rate is a design signal.
```

**Turn on the forensics.**
```sql
ALTER SYSTEM SET log_lock_waits = on;
ALTER SYSTEM SET deadlock_timeout = '1s';       -- ★ leave at default
ALTER SYSTEM SET log_min_duration_statement = '500ms';
SELECT pg_reload_conf();
```

---

## Example 2 — production scenario

**The situation.** A warehouse management system. A release adds "reserve stock at checkout." Within two days:

```
 checkout error rate      0.02% → ★ 2.8%
 checkout p99 latency     180 ms → ★ 1,340 ms
 all failures             SQLSTATE 40P01, "deadlock detected"
 ★ reproduces on staging  never — one client, no overlap
 ★ correlates with        concurrency, not order size or data volume
 CPU / disk / memory      unchanged
```

```
 ★ THE p99 IS THE TELL: 1,340 ms, when the handler does 4 queries
   that each take <5 ms. That extra second is deadlock_timeout.
   ⇒ p99 ≈ 1,000 ms + normal latency is the SIGNATURE of deadlock
     detection, even before you look at the error codes.
```

**Step 1 — get the statements from the log.**

```
2026-08-16 14:22:41.882 IST [41202] ERROR:  deadlock detected
2026-08-16 14:22:41.882 IST [41202] DETAIL:
  Process 41202 waits for ShareLock on transaction 8842119; blocked by process 41288.
  Process 41288 waits for ShareLock on transaction 8842118; blocked by process 41202.
  Process 41202: UPDATE inventory SET reserved = reserved + $1
                  WHERE product_id = ANY($2)
  Process 41288: UPDATE inventory SET reserved = reserved + $1
                  WHERE product_id = ANY($2)
2026-08-16 14:22:41.882 IST [41202] CONTEXT:
  while updating tuple (14,22) in relation "inventory"
2026-08-16 14:22:41.882 IST [41202] STATEMENT:
  UPDATE inventory SET reserved = reserved + $1 WHERE product_id = ANY($2)
```

```
 ★ BOTH PROCESSES ARE RUNNING THE SAME STATEMENT.
   That immediately rules out patterns ③ (table order) and ④ (FK).
   ⇒ this is a ROW-ORDER problem within one statement.
```

**Step 2 — find the handler.**

```js
// src/checkout/reserve.js
async function reserveStock(orderId, items) {
  return withTransaction(async (tx) => {
    const productIds = items.map(i => i.product_id);   // ★ order = cart order
    const qty        = items.map(i => i.quantity);

    await tx.query(
      `UPDATE inventory SET reserved = reserved + $1
        WHERE product_id = ANY($2)`, [qty, productIds]);

    await tx.query(
      `INSERT INTO reservations (order_id, product_id, quantity)
       SELECT $1, unnest($2::bigint[]), unnest($3::int[])`,
      [orderId, productIds, qty]);
  });
}
```

```
 ★ TWO CUSTOMERS BUYING THE SAME TWO PRODUCTS IN DIFFERENT CART
   ORDERS DEADLOCK EACH OTHER.
     customer A's cart: [ Kurta(88), Dupatta(12) ]
     customer B's cart: [ Dupatta(12), Kurta(88) ]
   ⇒ and it is worse than it looks, because…
```

**Step 3 — the subtlety: sorting the array is not enough.**

```sql
EXPLAIN (ANALYZE) UPDATE inventory SET reserved = reserved + 1
 WHERE product_id = ANY(ARRAY[88, 12, 41]);
```
```
 Update on inventory  (cost=… rows=0)
   ->  Bitmap Heap Scan on inventory
         Recheck Cond: (product_id = ANY ('{88,12,41}'::bigint[]))
         ★ Heap Blocks: exact=3
         ->  Bitmap Index Scan on inventory_pkey
```

```
 ★ A BITMAP HEAP SCAN RETURNS ROWS IN PHYSICAL PAGE ORDER, NOT
   ARRAY ORDER. So sorting the JavaScript array changes nothing —
   the lock order is whatever the plan produces.
 ⇒ AND WORSE: the plan can CHANGE. With 3 ids the planner may pick
   an Index Scan (array order); with 30 it picks a Bitmap Heap Scan
   (page order). A statistics refresh can silently flip the lock
   order of a statement you never touched.
 ⇒ ★ THIS IS WHY "sort the array" IS NOT THE FIX. The fix is to
   ACQUIRE THE LOCKS EXPLICITLY, IN A GUARANTEED ORDER, FIRST.
```

**Step 4 — the fix.**

```js
async function reserveStock(orderId, items) {
  // ★ ① deduplicate and sort — one canonical order, everywhere
  const byProduct = new Map();
  for (const it of items) {
    byProduct.set(it.product_id,
      (byProduct.get(it.product_id) ?? 0) + it.quantity);
  }
  const ids = [...byProduct.keys()].sort((a, b) => a - b);
  const qty = ids.map(id => byProduct.get(id));

  return retryOnConflict(() => withTransaction(async (tx) => {
    // ★ ② acquire EVERY row lock first, in a deterministic order.
    //   ORDER BY on a locking SELECT is honoured — this is the
    //   only way to guarantee lock acquisition order.
    //   FOR NO KEY UPDATE, not FOR UPDATE: we don't touch a key
    //   column, so child inserts elsewhere aren't blocked (Topic 45).
    const locked = await tx.query(
      `SELECT product_id, available - reserved AS free
         FROM inventory
        WHERE product_id = ANY($1)
        ORDER BY product_id
          FOR NO KEY UPDATE`, [ids]);

    if (locked.rowCount !== ids.length)
      throw new AppError('PRODUCT_NOT_FOUND');

    // ③ every lock is now held — the rest cannot deadlock
    const short = locked.rows.filter((r, i) => r.free < qty[i]);
    if (short.length) throw new AppError('INSUFFICIENT_STOCK', { short });

    await tx.query(
      `UPDATE inventory AS inv SET reserved = inv.reserved + v.q
         FROM unnest($1::bigint[], $2::int[]) AS v(pid, q)
        WHERE inv.product_id = v.pid`, [ids, qty]);

    await tx.query(
      `INSERT INTO reservations (order_id, product_id, quantity)
       SELECT $1, unnest($2::bigint[]), unnest($3::int[])`,
      [orderId, ids, qty]);
  }));
}
```

```js
// ④ the retry loop — 40P01 and 40001 together (Topics 44, 52)
const RETRYABLE = new Set(['40001', '40P01']);

async function retryOnConflict(fn, { attempts = 3 } = {}) {
  for (let i = 0; i < attempts; i++) {
    try {
      return await fn();
    } catch (e) {
      if (!RETRYABLE.has(e.code) || i === attempts - 1) throw e;
      metrics.increment('db.conflict_retry', { sqlstate: e.code });
      // ★ FULL JITTER. Fixed backoff makes the same pair collide
      //   again on the same schedule.
      await sleep(Math.random() * (25 * 2 ** i));
    }
  }
}
```

**Step 5 — the standing rule, written down.**

```
 ★ LOCK ORDER CONTRACT — docs/engineering/lock-order.md

 ① ROWS: always ascending primary key.
    Acquire with an explicit locking SELECT … ORDER BY pk
    FOR NO KEY UPDATE before any multi-row write.
    ★ Never rely on the order of `WHERE id = ANY($1)` — that is
      the PLAN's order and it can change.

 ② TABLES: when one transaction writes several tables, always in
    this order:
       accounts → orders → order_items → inventory
       → reservations → ledger_entries
    ★ Parents before children. Money last.

 ③ NEVER UPGRADE A LOCK.
    ✗ SELECT … FOR SHARE  then UPDATE
    ✓ SELECT … FOR UPDATE / FOR NO KEY UPDATE up front
    ✓ or a single atomic UPDATE with no prior SELECT

 ④ MULTI-ROW INSERT / UPSERT: sort by the conflicting unique key.

 ⑤ Every write path that can conflict gets retryOnConflict().
    The retried transaction MUST be idempotent (Topic 52).
```

**Step 6 — verify under load.**

```bash
cat > /tmp/reserve.sql <<'EOF'
\set a random(1, 200)
\set b random(1, 200)
BEGIN;
SELECT product_id FROM inventory WHERE product_id IN (:a, :b)
  ORDER BY product_id FOR NO KEY UPDATE;
UPDATE inventory SET reserved = reserved + 1 WHERE product_id IN (:a, :b);
COMMIT;
EOF
pgbench -f /tmp/reserve.sql -c 100 -j 8 -T 120 wms
psql -d wms -c "SELECT deadlocks FROM pg_stat_database WHERE datname='wms';"
```
```
 tps = 14,882.1  (including connections establishing)
 deadlocks
-----------
         0        ★ 100 clients, 2 minutes, 1.7M transactions, zero
```

**Step 7 — results.**

| | Before | After |
|---|---|---|
| Checkout error rate | 2.8% (`40P01`) | **0.00%** |
| Checkout p99 | 1,340 ms | 96 ms (**14×**) |
| Deadlocks / day | 41,882 | 0 |
| Retries triggered | n/a | ~0.01% (from `40001` elsewhere) |
| Throughput at 100 clients | 4,102 tps | 14,882 tps (**3.6×**) |

```
 ★ THE LESSON: the p99 of 1,340 ms was not a slow query. It was
   deadlock_timeout — one full second of two transactions frozen
   before PostgreSQL even LOOKED for a cycle. The throughput gain
   came almost entirely from removing that dead time, not from
   removing the errors.
 ⇒ AND: "sort the array" — the obvious fix — would NOT have worked,
   because a Bitmap Heap Scan locks in page order. Only an explicit
   `ORDER BY … FOR NO KEY UPDATE` guarantees acquisition order.
```

---

## Common mistakes

**1. Assuming `WHERE id = ANY($1)` locks in array order.**
- *Symptom:* deadlocks persist after sorting the array; they appear or disappear after an `ANALYZE`.
- *Engine-level why:* row lock order follows the executor's row order. A Bitmap Heap Scan returns physical page order, and the plan can change with statistics.
- *Fix:* an explicit `SELECT … ORDER BY pk FOR NO KEY UPDATE` before the write.

**2. `SELECT … FOR SHARE` followed by `UPDATE`.**
- *Symptom:* a deadlock that reproduces 100% of the time.
- *Engine-level why:* SHARE locks are mutually compatible, so both transactions acquire one; both then need to upgrade to exclusive.
- *Fix:* never upgrade. Take `FOR UPDATE`/`FOR NO KEY UPDATE` at first read, or use one atomic `UPDATE`.

**3. Lowering `deadlock_timeout` to "detect faster."**
- *Symptom:* higher CPU and lock-manager contention system-wide.
- *Engine-level why:* the check runs on *every* lock wait exceeding the timeout, not just real deadlocks. It takes an exclusive lock on the whole lock manager.
- *Fix:* leave it at 1s and fix the ordering. Deadlocks should be rare enough that detection latency doesn't matter.

**4. Retrying without fixing the order.**
- *Symptom:* the deadlock rate grows linearly with traffic until retries can't absorb it.
- *Fix:* retries are the safety net, not the fix. A rising `pg_stat_database.deadlocks` rate is a design signal.

**5. Retrying without jitter.**
- *Symptom:* the same pair collides again on the same schedule; retries make it worse.
- *Fix:* full jitter — `Math.random() * base`.

**6. Retrying a non-idempotent transaction.**
- *Symptom:* duplicate charges, double-sent emails, doubled counters.
- *Fix:* the retried unit must be safe to re-run — an idempotency key with a `UNIQUE` constraint (Topic 52).

**7. Not setting `log_lock_waits`.**
- *Symptom:* you get `ERROR: deadlock detected` with no statements, and are reduced to guessing which handler.
- *Fix:* `log_lock_waits = on` in every production database. It costs nothing — the check has already run.

**8. Missing the foreign-key case.**
- *Symptom:* a deadlock between statements that name entirely different tables.
- *Engine-level why:* an `INSERT` into a child takes `FOR KEY SHARE` on the parent row.
- *Fix:* update parents before inserting children, or don't do both in one transaction.

**9. Raising the isolation level to "fix" deadlocks.**
- *Symptom:* the deadlocks remain and `40001` errors are added on top.
- *Engine-level why:* deadlocks are a lock-ordering phenomenon; they occur identically at every isolation level.
- *Fix:* ordering.

**10. Treating `40P01` as a 500.**
- *Symptom:* users see failures for a condition the server explicitly labelled retryable.
- *Fix:* the same retry loop as `40001`.

---

## Hands-on proof

**PROVE IT #1–#6 — Example 1** (a two-row deadlock, the 1-second detection cost, the consistent-order fix, the 100%-reproducible lock upgrade, the foreign-key deadlock, `pg_stat_database.deadlocks`).

**PROVE IT #7 — a three-way cycle.**
```sql
CREATE TABLE t (id int PRIMARY KEY, n int);
INSERT INTO t SELECT g, 0 FROM generate_series(1,3) g;

-- s1: UPDATE t SET n=n+1 WHERE id=1;   then WHERE id=2;
-- s2: UPDATE t SET n=n+1 WHERE id=2;   then WHERE id=3;
-- s3: UPDATE t SET n=n+1 WHERE id=3;   then WHERE id=1;
-- ★ no two sessions conflict directly. The cycle needs all three.
-- DETAIL will show a three-process chain.
```

**PROVE IT #8 — the array order is NOT the lock order.**
```sql
CREATE TABLE inventory (product_id bigint PRIMARY KEY, reserved int NOT NULL DEFAULT 0);
INSERT INTO inventory SELECT g,0 FROM generate_series(1,100000) g;
ANALYZE inventory;

EXPLAIN (ANALYZE) UPDATE inventory SET reserved=reserved+1
 WHERE product_id = ANY(ARRAY[9000,10,5000]);
```
```
 ->  Bitmap Heap Scan on inventory
       ★ rows are returned in PAGE order: 10, 5000, 9000
         — not the array's 9000, 10, 5000
```
```sql
-- the guaranteed form:
SELECT product_id FROM inventory
 WHERE product_id = ANY(ARRAY[9000,10,5000])
 ORDER BY product_id FOR NO KEY UPDATE;
-- ★ 10, 5000, 9000 — always, regardless of plan
```

**PROVE IT #9 — measure the p99 signature.**
```bash
cat > /tmp/dl.sql <<'EOF'
\set a random(1, 50)
\set b random(1, 50)
BEGIN;
UPDATE inventory SET reserved = reserved+1 WHERE product_id = :a;
UPDATE inventory SET reserved = reserved+1 WHERE product_id = :b;
COMMIT;
EOF
pgbench -f /tmp/dl.sql -c 50 -j 4 -T 60 -r wms
psql -c "SELECT deadlocks FROM pg_stat_database WHERE datname='wms'"
```
```
 latency average = 42.8 ms
 ★ latency stddev = 214.9 ms       ← the 1s deadlock waits
 deadlocks: 1,882
```
```bash
# now with sorted acquisition
cat > /tmp/dl_sorted.sql <<'EOF'
\set a random(1, 50)
\set b random(1, 50)
BEGIN;
SELECT product_id FROM inventory WHERE product_id IN (:a,:b)
  ORDER BY product_id FOR NO KEY UPDATE;
UPDATE inventory SET reserved = reserved+1 WHERE product_id IN (:a,:b);
COMMIT;
EOF
pgbench -f /tmp/dl_sorted.sql -c 50 -j 4 -T 60 -r wms
```
```
 latency average = 3.1 ms
 ★ latency stddev = 1.8 ms
 deadlocks: 0
```

**PROVE IT #10 — see the wait-for graph before it becomes a cycle.**
```sql
SELECT pid, pg_blocking_pids(pid) AS waits_for, substring(query,1,60) AS query
  FROM pg_stat_activity WHERE cardinality(pg_blocking_pids(pid)) > 0;
-- ★ run this during the deadlock's 1-second window and you can
--   see the cycle directly: A's waits_for contains B, B's contains A.
```

---

## The design decision framework

```
★★★ DEADLOCKS ARE AN ORDERING BUG. ORDERING IS A COMPLETE FIX. ★★★

 ① ESTABLISH A GLOBAL LOCK ORDER — WRITE IT DOWN
    ROWS:   always ascending primary key
    TABLES: one documented order for the whole codebase
            (parents before children; money last)
    ⇒ ★ this is a proof, not a heuristic: every wait points
      "upward", so a cycle is arithmetically impossible.

 ② ACQUIRE LOCKS EXPLICITLY WHEN ORDER MATTERS
    ✗ UPDATE t SET … WHERE id = ANY($1)
      ⇒ locks in the PLAN's order, which can change with statistics
    ✓ SELECT id FROM t WHERE id = ANY($1)
       ORDER BY id FOR NO KEY UPDATE;      ← ★ then write
    ⇒ this is the ONLY way to guarantee acquisition order.

 ③ NEVER UPGRADE A LOCK
    ✗ FOR SHARE → UPDATE          (★ 100% reproducible deadlock)
    ✓ FOR UPDATE / FOR NO KEY UPDATE at first read
    ✓ ★ better: no locking SELECT at all —
        UPDATE t SET x = x - $1 WHERE id=$2 AND x >= $1

 ④ SHORTEN THE WINDOW
    fewer statements between the first lock and COMMIT
    ⇒ never an HTTP call, a queue publish, or user think-time
      inside a transaction (Topics 45, 52)

 ⑤ RETRY — THE SAFETY NET, NOT THE FIX
    ✓ 40P01 and 40001 together
    ✓ bounded (3–4), exponential, ★ FULL JITTER
    ✓ ★ the retried unit MUST be idempotent (Topic 52)
    ✓ a metric — a rising rate means the ordering is wrong somewhere

 ⑥ MAKE IT DIAGNOSABLE BEFORE YOU NEED IT
    log_lock_waits = on           ★ free; gives you both statements
    deadlock_timeout = '1s'       ★ leave it alone
    monitor pg_stat_database.deadlocks as a RATE

 ⑦ THE p99 TELL
    p99 ≈ 1,000 ms + normal latency, on a handler whose queries are
    all fast ⇒ ★ that second is deadlock_timeout. Look for 40P01
    before you look at query plans.

 ⑧ WHAT NOT TO DO
    ✗ lower deadlock_timeout       (moves cost onto ordinary waits)
    ✗ raise the isolation level    (irrelevant — deadlocks occur at
                                    every level)
    ✗ "just retry harder"          (hides a bug that grows with load)
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
With two `psql` sessions: (a) produce a two-row deadlock and read the `DETAIL` to identify which parameters each process used; (b) measure the failing statement's latency and explain the number; (c) fix it by sorting and show both sessions succeeding with one merely blocking.

### Exercise 2 — medium (apply it)
Reproduce and fix all five patterns:
(a) inconsistent row order · (b) the `FOR SHARE` → `UPDATE` upgrade · (c) two handlers touching two tables in opposite orders · (d) the foreign-key parent lock · (e) a multi-row `INSERT … ON CONFLICT` with rows in different orders

For each: show the deadlock, explain the cycle, apply the fix, and prove it's gone under `pgbench` load with `pg_stat_database.deadlocks`.

### Exercise 3 — hard (production simulation)
A checkout handler's error rate went 0.02% → 2.8% and p99 180 ms → 1,340 ms after a release adding stock reservation. All failures are `40P01`. It never reproduces on staging.

(a) Explain the p99 of 1,340 ms precisely. What would you conclude from that number *before* reading any error codes?
(b) From the log `DETAIL` showing both processes running the *same* statement, which of the five deadlock patterns are ruled out, and why?
(c) The handler does `UPDATE inventory … WHERE product_id = ANY($1)`. Explain why sorting the array in JavaScript does **not** fix it. Use `EXPLAIN` output in your answer.
(d) Explain how a routine `ANALYZE` could make this bug appear or disappear overnight.
(e) Write the corrected handler. Justify `FOR NO KEY UPDATE` over `FOR UPDATE`.
(f) Write the retry helper. Which SQLSTATEs, what backoff, why jitter, and what must be true of the transaction for retrying to be safe?
(g) Write the lock-order contract for the codebase — rows, tables, upgrades, multi-row upserts.
(h) Design the `pgbench` script that proves the fix at 100 concurrent clients, and state exactly which numbers you'd report.
(i) The throughput went 4,102 → 14,882 tps. Explain why most of that gain came from something other than the elimination of errors.

---

## Mental model checkpoint

1. Define a deadlock in one sentence, in terms of a graph.
2. When does PostgreSQL check for deadlocks? What does that imply about the minimum latency of every deadlock?
3. How is the victim chosen? What does that mean for your retry strategy?
4. State the theorem about consistent lock ordering and sketch why it holds.
5. Why does `SELECT … FOR SHARE` followed by `UPDATE` deadlock 100% of the time?
6. Why is sorting a `WHERE id = ANY($1)` array insufficient? What *is* sufficient?
7. How can two `INSERT`s into `order_items` deadlock with two `UPDATE`s of `orders`?
8. Why is lowering `deadlock_timeout` usually wrong?
9. What p99 signature suggests deadlocks before you've looked at error codes?

---

## Quick reference card

**The error**
```
ERROR:  deadlock detected                      -- ★ SQLSTATE 40P01, RETRYABLE
DETAIL: Process A waits for … blocked by process B.
        Process B waits for … blocked by process A.
        Process A: <statement>       ← ★ needs log_lock_waits = on
        Process B: <statement>
CONTEXT: while updating tuple (14,22) in relation "inventory"
```

**Detect & monitor**
```sql
SELECT deadlocks FROM pg_stat_database WHERE datname = current_database();
SELECT pid, pg_blocking_pids(pid), substring(query,1,60)
  FROM pg_stat_activity WHERE cardinality(pg_blocking_pids(pid)) > 0;
ALTER SYSTEM SET log_lock_waits = on;      -- ★ free forensics
SHOW deadlock_timeout;                     -- 1s — ★ leave it
```

**The five patterns → fixes**

| Pattern | Fix |
|---|---|
| inconsistent row order | ★ `SELECT … ORDER BY pk FOR NO KEY UPDATE` first |
| **lock upgrade** (`FOR SHARE`→`UPDATE`) | ★ take the final strength up front |
| inconsistent table order | one documented order, codebase-wide |
| FK parent lock | update the parent **before** inserting children |
| multi-row upsert | sort rows by the conflicting unique key |

**The guaranteed acquisition idiom**
```sql
SELECT id FROM inventory WHERE id = ANY($1)
 ORDER BY id FOR NO KEY UPDATE;   -- ★ then write. Order is guaranteed.
```

**Retry**
```js
const RETRYABLE = new Set(['40001', '40P01']);
// bounded (3–4) · exponential · ★ FULL JITTER · idempotent unit · metered
```

**The p99 tell:** ~1,000 ms + normal latency on a fast handler = `deadlock_timeout`.

**Never:** lower `deadlock_timeout` · raise the isolation level · retry without fixing the order.

---

## When would I use this at work?

1. **A low-but-nonzero error rate that scales with concurrency and won't reproduce locally.** That's the deadlock signature. The log's `DETAIL` shows both statements *and both parameter sets* — usually the whole bug in one line.

2. **Any pull request that adds a second `UPDATE` to an existing transaction.** That's the moment a lock order gets violated. A written lock-order contract turns it into a review checklist item rather than a production incident two days later.

3. **When someone proposes lowering `deadlock_timeout` or adding "just one more retry."** Both move cost around without removing the cycle. Ordering removes it permanently, with a proof.

4. **Reading a p99 that's suspiciously close to a round second.** `deadlock_timeout`, `lock_timeout` and `statement_timeout` all leave round numbers in latency histograms. Recognising them saves hours of query-plan archaeology.

---

## Connected topics

**Understand before this:** 45 (locks — modes, `FOR KEY SHARE`, the FIFO queue), 44 (isolation levels — `40001` and the retry loop), 23 (foreign keys — the invisible parent lock).

**This unlocks:**
- **49** — optimistic vs pessimistic concurrency: avoiding locks entirely
- **52** — idempotency: what makes a retried transaction safe
- **67** — performance investigation: reading round numbers out of latency histograms
- **Case study 01** — the hot row, where lock ordering and contention meet
