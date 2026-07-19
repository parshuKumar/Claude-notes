# Topic 55 — Deadlocks in Practice
### SQL Mastery Curriculum — Phase 8: Transactions and Concurrency in SQL

---

## 1. ELI5 — The Analogy That Makes It Click

Two people walk into a shared kitchen to make a sandwich. There is exactly one knife and one cutting board.

- **Alice** grabs the **knife** first. She now needs the **cutting board** to finish.
- **Bob** grabs the **cutting board** first. He now needs the **knife** to finish.

Alice is standing there holding the knife, waiting for Bob to put down the cutting board. Bob is standing there holding the cutting board, waiting for Alice to put down the knife. Neither will let go until they finish, and neither can finish. They will stand there forever.

That is a **deadlock**: a cycle of "I'm waiting for what you hold, and you're waiting for what I hold." No amount of patience resolves it, because the thing each is waiting for will never be released.

In a kitchen, a manager eventually notices, walks over, and taps one person on the shoulder: "You — put your tool down and start over." That person's half-made sandwich is scrapped, the other person finishes, and then the tapped person tries again — this time, hopefully, grabbing the tools in the same order everyone else uses.

PostgreSQL is that manager. It runs a background check, notices the waiting cycle, picks one transaction as the **victim**, cancels it with an error, and lets the others proceed. The victim's application is expected to catch the error and **retry**. And the way you *prevent* the manager ever having to intervene is the same as in the kitchen: everyone agrees to grab the knife before the cutting board — a **consistent lock ordering**.

The word "deadlock" means exactly this: a set of transactions frozen in a waiting cycle that can only be broken by force.

---

## 2. Connection to SQL Internals

A deadlock is a property of the **lock manager**, one of PostgreSQL's core subsystems. To understand deadlocks you must understand what the engine is actually doing under the hood.

**Row locks live in two places.** When a transaction locks a row (via `UPDATE`, `DELETE`, `SELECT ... FOR UPDATE` from Topic 54, or a foreign-key check), PostgreSQL does **not** keep a giant in-memory table of every locked row — that would not scale to millions of rows. Instead:

1. The lock is written **into the row itself**, on the heap page. The tuple header has an `xmax` field and infomask bits; a row locked by transaction 5000 records `xmax = 5000` plus flags indicating whether it is a full lock or a shared lock. This is why row locks cost essentially zero extra memory — they ride along in the tuple that MVCC already maintains (Topic 50).
2. To **wait** on a locked row, a transaction takes a lock not on the row but on the **transaction ID** holding it. Every transaction implicitly holds an exclusive lock on its own virtual and real XID. A waiter does `LOCK TABLE`-style bookkeeping in the in-memory **lock table** (`pg_locks`) requesting a `ShareLock` on the holder's XID. When the holder commits or aborts, it releases that XID lock, and the waiter wakes up.

**The wait-for graph.** Because waiting is modeled as "transaction A waits for a lock held by transaction B," the lock manager can construct a directed **wait-for graph**: an edge `A → B` means "A is blocked waiting for B." A deadlock is precisely a **cycle** in this graph: `A → B → A`, or a longer loop `A → B → C → A`. Detecting a deadlock is detecting a cycle in a directed graph — a classic depth-first-search problem.

**The lock manager's partitions.** The in-memory lock table is split into 16 partitions (`NUM_LOCK_PARTITIONS`) guarded by lightweight locks (LWLocks) to reduce contention. Heavyweight locks (row/table locks, the kind that can deadlock) live here. Deadlock detection must briefly examine multiple partitions to trace the graph.

**Why detection is deferred, not instant.** Checking for a cycle on *every* lock wait would be expensive, and the vast majority of lock waits are short and resolve on their own. So PostgreSQL waits `deadlock_timeout` (default **1 second**) after a transaction *starts* waiting before running the cycle-detection algorithm. If the wait clears naturally within that second, no detection ever runs. Only a wait that persists past the timeout triggers `DeadLockCheck()`.

**MVCC does not save you here.** MVCC (Topic 50) means readers never block writers and writers never block readers — plain `SELECT` never participates in a deadlock. But the moment you take **explicit write locks** (`UPDATE`, `FOR UPDATE`, `FOR NO KEY UPDATE`), you are in the heavyweight lock manager's territory, and cycles become possible. Deadlocks are a write-path phenomenon.

---

## 3. Logical Execution Order Context

Deadlocks are not a clause in the `FROM → WHERE → GROUP BY → SELECT → ORDER BY → LIMIT` pipeline. They live at a level **above** any single statement — the **transaction** and **statement scheduling** level. But the pipeline still matters, because *when* within statement execution a lock is acquired determines *when* a wait (and thus a deadlock) can occur.

```
BEGIN                              ← transaction starts; XID may be assigned lazily
  Statement 1 (e.g. UPDATE ...)
    FROM / scan                    ← rows located
    WHERE                          ← rows filtered
    ROW LOCK ACQUISITION           ← ★ locks taken here, as qualifying rows are found
    modify heap tuple
  Statement 2 (e.g. UPDATE ...)
    ... lock acquisition again ...  ← ★ a second set of locks; ordering across
                                       statements is what creates deadlock potential
COMMIT / ROLLBACK                  ← ★ ALL locks released here, atomically
```

Three consequences of this ordering:

1. **Locks are acquired mid-statement, as rows are matched** — not all at once at `BEGIN`. An `UPDATE ... WHERE status = 'x'` locks each qualifying row *in the physical order the scan visits them* (heap order for a Seq Scan, index order for an Index Scan). This physical visitation order is a hidden source of deadlocks between two bulk `UPDATE`s: see Section 6.

2. **Locks are held until the transaction ends** — PostgreSQL uses strict two-phase locking for row locks: once acquired, a row lock is never released until `COMMIT` or `ROLLBACK`. There is no "release this row early." This is why holding a transaction open across multiple statements accumulates locks and widens the deadlock window.

3. **`ORDER BY` in the statement does not guarantee lock order** unless it drives the actual access path. `UPDATE ... WHERE id IN (...)` does not lock in the `IN`-list order; it locks in scan order. To force lock *acquisition* order you generally need `SELECT ... FOR UPDATE ... ORDER BY id` (Topic 54) as a separate locking step before the writes — this is the primary prevention technique, covered in Section 6.

---

## 4. What Is a Deadlock?

A **deadlock** is a state in which two or more transactions each hold a lock the other needs, forming a cycle in the wait-for graph such that none can proceed. PostgreSQL cannot resolve it by waiting, so it detects the cycle and aborts one transaction (the **victim**) with `SQLSTATE 40P01` (`deadlock_detected`), releasing that transaction's locks so the survivors continue.

There is no special SQL keyword to "do a deadlock" — it is an emergent runtime condition. But there is configuration and there are error signatures. Here is the anatomy, annotated:

```
ERROR:  deadlock detected
   │
   └── the top-level error; SQLSTATE = 40P01, class 40 = "Transaction Rollback"
DETAIL:  Process 18471 waits for ShareLock on transaction 6912; blocked by process 18472.
   │              │                   │                  │                    │
   │              │                   │                  │                    └── the other backend PID
   │              │                   │                  └── the XID being waited on (the holder's transaction)
   │              │                   └── lock mode requested (ShareLock on an XID = "wait for that txn to end")
   │              └── the OS/backend PID of the waiting process
   │
   Process 18472 waits for ShareLock on transaction 6911; blocked by process 18471.
   │
   └── the reciprocal edge — this line + the one above form the CYCLE (18471→18472→18471)
HINT:  See server log for query details.
   │
   └── the full SQL of each participant is written to the server log, not the client
CONTEXT:  while updating tuple (0,14) in relation "orders"
   │                            │  │                  │
   │                            │  │                  └── the table whose row was contended
   │                            │  └── tuple offset within the page (item pointer 14)
   │                            └── the heap block number (page 0)
   └── which physical row the victim was trying to lock when the cycle was found
```

Key configuration knobs (annotated):

```sql
SHOW deadlock_timeout;
   │
   └── how long a transaction waits on a lock BEFORE the deadlock checker runs.
       Default '1s'. NOT a limit on how long you wait for a lock in general —
       only the delay before cycle-detection kicks in. Lower = faster detection
       but more CPU spent checking; higher = deadlocks linger longer before resolution.

SHOW log_lock_waits;
   │
   └── boolean, default off. When ON, any wait exceeding deadlock_timeout is logged
       (even if it is NOT a deadlock — just a long wait). Essential for diagnosing
       lock contention in production. Turn this ON.

SHOW lock_timeout;
   │
   └── caps how long a single statement waits for ANY lock before erroring with 55P03
       (lock_not_available). A blunt instrument that also aborts non-deadlock waits,
       but useful as a safety net so a wedged lock can't hang a request forever.

SHOW statement_timeout;
   │
   └── caps total statement runtime. Indirectly bounds lock waits too. Set per-role
       or per-transaction for user-facing queries.
```

---

## 5. Why Deadlock Mastery Matters in Production

1. **They appear only under concurrency, never in testing.** A deadlock requires two transactions to interleave in a specific way at the same instant. Single-threaded local tests and low-traffic staging environments almost never trigger them. They surface for the first time in production, at peak load, as sporadic `deadlock detected` errors — and if you have no retry logic, they surface to the **end user** as failed requests, failed payments, failed order placements.

2. **The cost is a fully rolled-back transaction.** The victim doesn't lose one statement — it loses *everything* since `BEGIN`. If a transaction debited an account, wrote an audit log, and updated inventory, and it becomes the deadlock victim on the fourth statement, all three prior writes are undone. Without retry, that unit of work simply never happened, silently, from the user's perspective (they got a 500).

3. **Deadlocks scale super-linearly with traffic.** The probability that two transactions collide in a cycle rises roughly with the square of the concurrent transaction count touching the same rows. A system fine at 100 req/s can enter deadlock storms at 400 req/s — a 4× traffic increase producing a ~16× deadlock rate. This is why they feel like they appear "suddenly" during growth or a traffic spike.

4. **They are almost always a code-ordering bug, not a database bug.** The overwhelming majority of production deadlocks come from two code paths that lock the same set of rows **in different orders** — e.g., a "transfer" function that locks `from_account` then `to_account`, invoked concurrently as `transfer(A,B)` and `transfer(B,A)`. The fix is a code convention (consistent lock ordering), and understanding this saves you from blaming the database or randomly adding indexes.

5. **The victim-selection is not something you control directly**, so you must design for *any* participant being chosen. PostgreSQL picks the victim by a heuristic (roughly, the transaction that would be cheapest to abort / is later in the cycle), not by priority. Your critical payment transaction can be the victim just as easily as a low-priority background job. Correct retry logic is therefore mandatory on every write path that can contend.

6. **Silent data-integrity risk when mishandled.** A team that "handles" deadlocks by catching the error and *ignoring* it (not retrying) produces the worst outcome: no error surfaced to the user, but the work silently dropped. Understanding that `40P01` means "safe to retry the whole transaction" — and that retrying is the *correct* response — is the difference between a resilient system and a silently lossy one.

---

## 6. Deep Technical Content

### 6.1 The Canonical Two-Transaction Deadlock

The textbook deadlock: two rows, two transactions, opposite lock order. Use two `psql` sessions.

```sql
-- Setup
CREATE TABLE accounts (
  id      INT PRIMARY KEY,
  balance NUMERIC NOT NULL
);
INSERT INTO accounts VALUES (1, 1000), (2, 1000);
```

```
Session A (txn 1)                     Session B (txn 2)
------------------------------------  ------------------------------------
BEGIN;                                BEGIN;

UPDATE accounts SET balance =
  balance - 100 WHERE id = 1;
-- locks row id=1  (holds lock on 1)
                                      UPDATE accounts SET balance =
                                        balance - 100 WHERE id = 2;
                                      -- locks row id=2  (holds lock on 2)

UPDATE accounts SET balance =
  balance + 100 WHERE id = 2;
-- wants row id=2, held by B → WAITS
                                      UPDATE accounts SET balance =
                                        balance + 100 WHERE id = 1;
                                      -- wants row id=1, held by A → WAITS
                                      -- ★ CYCLE: A→B→A
```

At this instant both sessions are blocked. `deadlock_timeout` (1s) after A began waiting, the checker runs, finds the cycle A→B→A, and picks a victim. One session receives:

```
ERROR:  deadlock detected
DETAIL:  Process 18471 waits for ShareLock on transaction 6912; blocked by process 18472.
         Process 18472 waits for ShareLock on transaction 6911; blocked by process 18471.
HINT:  See server log for query details.
CONTEXT:  while updating tuple (0,1) in relation "accounts"
```

The victim's transaction is now aborted (all its locks released). The **survivor's** blocked `UPDATE` immediately succeeds and it can `COMMIT`. The victim must `ROLLBACK` (it's already in a failed state) and retry from `BEGIN`.

**The root cause in one sentence:** A locked rows in order (1, 2); B locked them in order (2, 1). Opposite orders → cycle possible.

### 6.2 How PostgreSQL Detects the Cycle — `DeadLockCheck`

The algorithm, step by step:

1. Transaction A requests a lock and cannot get it immediately → it enters the wait queue for that lock and **sleeps**, arming a timer for `deadlock_timeout`.
2. If A is woken by the lock being granted before the timer fires, nothing happens — no detection, no overhead. (This is the common case: most waits are short.)
3. If the timer fires while A is still waiting, A runs `DeadLockCheck()` **on its own backend**. It acquires all lock-partition LWLocks, then performs a depth-first search of the wait-for graph starting from itself.
4. It builds edges: "A waits for B" (hard edge — B holds a conflicting lock) and also considers **soft edges** (ordering among waiters in the same queue that could be rearranged to avoid a deadlock).
5. If the DFS finds a cycle involving A, a deadlock exists. Before aborting, PostgreSQL tries to resolve it by **reordering soft edges** (rearranging wait queues) if that breaks the cycle without aborting anyone. If no reordering helps, it selects a victim.
6. The **victim is the transaction running `DeadLockCheck` itself** — i.e., the one whose timer fired and found the cycle. So the victim is essentially "whichever waiter's `deadlock_timeout` expired first and detected the loop." This is why victim selection feels non-deterministic: it depends on wait timing.
7. The victim's backend raises `ERROR: deadlock detected` (40P01), which aborts its transaction and releases its locks; the survivors' waits then resolve.

The DFS is bounded by `max_locks_per_transaction` and the number of active backends; it is fast (microseconds) relative to the 1-second wait that preceded it.

### 6.3 Deadlocks From Bulk UPDATEs in Different Scan Orders

You do not need two explicit rows in opposite order in your code. Two bulk `UPDATE`s that touch **overlapping sets of rows in different physical orders** deadlock just as readily.

```sql
-- Session A
UPDATE products SET stock = stock - 1 WHERE category_id = 5;
-- Session B
UPDATE products SET stock = stock + 1 WHERE supplier_id = 9;
```

If category 5 and supplier 9 overlap on some rows, and the two statements visit those rows in different orders (because A scans via a category index and B via a supplier index, or via Seq Scan from different positions), A can lock row X then wait for row Y while B locked Y then waits for X. Same cycle, no explicit per-row code.

**Mitigation:** force a deterministic lock order. If both statements first do `SELECT id FROM products WHERE ... ORDER BY id FOR UPDATE`, they acquire locks in ascending `id` order and cannot form a cycle (proof in 6.6). Or accept the deadlock and retry.

### 6.4 Deadlocks From Foreign Keys

Foreign keys take locks you did not write. Inserting or updating a child row that references a parent takes a `FOR KEY SHARE` lock on the **parent** row (to prevent the parent from being deleted or its key changed mid-transaction). Two transactions inserting children of *different* parents, then updating each other's parents, can deadlock through FK locks alone.

```
Session A: INSERT INTO orders(customer_id) VALUES (1);
           -- takes FOR KEY SHARE on customers row id=1
Session B: INSERT INTO orders(customer_id) VALUES (2);
           -- takes FOR KEY SHARE on customers row id=2
Session A: UPDATE customers SET name=... WHERE id=2;  -- wants lock on 2 → waits
Session B: UPDATE customers SET name=... WHERE id=1;  -- wants lock on 1 → waits → CYCLE
```

Since PostgreSQL 9.3, FK checks use `FOR KEY SHARE` (a weaker lock) precisely to reduce this class of deadlock — key-share locks are compatible with each other, so two inserts referencing the *same* parent no longer block. But cross-parent update cycles like the above are still possible.

### 6.5 Deadlocks From Index/Unique Constraints and INSERT Conflicts

Two transactions inserting the **same unique key** value can deadlock: each inserts a speculative index tuple, then when the second tries to insert a conflicting key it waits for the first transaction to commit or abort (to know whether the constraint is violated). If they then both wait on each other's pending inserts across two different unique keys, that is a cycle. `INSERT ... ON CONFLICT` reduces but does not fully eliminate this under high contention on multiple constraints.

### 6.6 The Consistent Lock-Ordering Theorem (Why Ordering Prevents Deadlock)

**Claim:** If every transaction acquires locks on a set of resources in the same total order (e.g., always ascending by primary key), no deadlock is possible.

**Proof sketch:** A deadlock requires a cycle T1 → T2 → ... → Tn → T1 in the wait-for graph. An edge Ti → Tj means Ti is waiting for a lock held by Tj. If Ti is waiting for resource R (held by Tj), then Ti already holds some resource with a *lower* order number than R (because Ti acquires in ascending order and has not yet reached R). So along any edge, the "highest lock held by the waiter" strictly increases as you follow held→wanted. Following a cycle back to the start would require the order number to be strictly greater than itself — a contradiction. Therefore no cycle can exist. ∎

**Practical form:** pick a canonical ordering key (usually the primary key) and always lock in ascending order.

```sql
-- Transfer money between two accounts, deadlock-free
BEGIN;
-- Lock BOTH rows up front, in a deterministic order (ascending id),
-- regardless of which is "from" and which is "to".
SELECT id, balance
FROM accounts
WHERE id IN (:from_id, :to_id)
ORDER BY id            -- ★ the canonical order — every caller uses ascending id
FOR UPDATE;

UPDATE accounts SET balance = balance - :amt WHERE id = :from_id;
UPDATE accounts SET balance = balance + :amt WHERE id = :to_id;
COMMIT;
```

Now `transfer(A,B)` and `transfer(B,A)` both lock `min(A,B)` first, then `max(A,B)`. The second to arrive simply waits for the first to finish — a clean wait, never a cycle.

### 6.7 Reducing the Deadlock Window

Even with imperfect ordering, you can shrink the odds:

- **Keep transactions short.** The longer locks are held, the wider the window for another transaction to build a conflicting cycle. Do not do HTTP calls, file I/O, or user-think-time inside an open transaction.
- **Acquire all needed locks early and in order**, then do the work. Lock-then-compute, not compute-then-lock-then-compute-more.
- **Touch rows in a consistent order** even in bulk `UPDATE`/`DELETE` — add `ORDER BY` via a locking `SELECT` first when the set is contended.
- **Lower `lock_timeout`** on user-facing paths so a wedged wait fails fast and retries, rather than hanging.
- **Use `SELECT ... FOR UPDATE SKIP LOCKED`** (Topic 54) for queue/worker patterns so workers grab *different* rows and never contend at all.

### 6.8 What Deadlock Detection Does NOT Do

- It does **not** prevent deadlocks — it only resolves them after they occur, ~1s later.
- It does **not** choose the victim by business priority. You cannot mark a transaction "please don't kill me."
- It does **not** fire for simple long waits — a transaction blocked for 10 minutes on a lock that is genuinely still held (no cycle) is *not* a deadlock and will wait indefinitely (unless `lock_timeout`/`statement_timeout` intervene). `log_lock_waits` logs these, but the checker leaves them alone because there is no cycle.
- It does **not** involve `SELECT` (non-locking reads). MVCC snapshots mean plain reads never wait on row locks and never deadlock.

### 6.9 Lock Modes and Compatibility (Why Some Pairs Deadlock and Others Don't)

Row-level lock modes, weakest to strongest:

| Mode | Taken by | Conflicts with |
|------|----------|----------------|
| `FOR KEY SHARE` | FK child insert/update | `FOR UPDATE` (on key cols) |
| `FOR SHARE` | `SELECT ... FOR SHARE` | `FOR NO KEY UPDATE`, `FOR UPDATE`, writes |
| `FOR NO KEY UPDATE` | `UPDATE` of non-key cols | `FOR SHARE`+, writes |
| `FOR UPDATE` | `SELECT ... FOR UPDATE`, `DELETE`, key `UPDATE` | everything except nothing (strongest) |

Two `FOR KEY SHARE` locks on the same row are **compatible** — no wait, no deadlock. Two `FOR UPDATE` on the same row conflict — the second waits. Deadlocks require at least one conflicting pair on each edge of the cycle; understanding which modes conflict tells you which workloads can and cannot deadlock.

---

## 7. EXPLAIN — Deadlocks in the Plan (and Why EXPLAIN Can't Show Them)

Deadlocks are a **runtime concurrency** phenomenon, not a query-plan property. `EXPLAIN` shows a single statement's plan in isolation; it has no notion of a *second* concurrent transaction, so it can never "show" a deadlock. What it *can* show is **where and in what order a statement acquires row locks**, which is the raw material for reasoning about deadlock risk.

### 7.1 Seeing the Lock Step in a Plan

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, balance
FROM accounts
WHERE id IN (1, 2)
ORDER BY id
FOR UPDATE;
```

```
LockRows  (cost=0.29..12.62 rows=2 width=42)
          (actual time=0.031..0.040 rows=2 loops=1)
  ->  Sort  (cost=0.29..12.60 rows=2 width=42)
            (actual time=0.024..0.028 rows=2 loops=1)
        Sort Key: id
        Sort Method: quicksort  Memory: 25kB
        ->  Index Scan using accounts_pkey on accounts
              (cost=0.15..12.59 rows=2 width=42)
              (actual time=0.010..0.016 rows=2 loops=1)
              Index Cond: (id = ANY ('{1,2}'::integer[]))
Buffers: shared hit=6
Planning Time: 0.12 ms
Execution Time: 0.061 ms
```

**Reading it for deadlock risk:**
- `LockRows` — the node that actually acquires the `FOR UPDATE` row locks. Its child feeds it rows **in the order the child produces them**, and `LockRows` locks them **in that order**.
- `Sort ... Sort Key: id` **before** `LockRows` — this is the crucial guarantee. Because a `Sort` sits under `LockRows`, rows are locked in ascending `id` order. This is exactly the consistent-ordering discipline from 6.6, made visible in the plan. Every caller that runs this query locks in the same order → no cycle.
- Contrast: without `ORDER BY id`, the plan would be `LockRows → Index Scan` locking in index-scan order, which for an `IN (1,2)` list is *usually* ascending but is **not guaranteed** across different predicates. The explicit `Sort` makes the order contractual.

### 7.2 A Plan That Locks in a Dangerous Order

```sql
EXPLAIN (ANALYZE)
UPDATE products SET stock = stock - 1
WHERE category_id = 5;
```

```
Update on products  (cost=0.00..1834.00 rows=0 width=0)
                    (actual time=25.3..25.3 rows=0 loops=1)
  ->  Seq Scan on products  (cost=0.00..1834.00 rows=980 width=10)
                            (actual time=0.02..12.1 rows=980 loops=1)
        Filter: (category_id = 5)
        Rows Removed by Filter: 49020
```

**Reading it for deadlock risk:**
- `Seq Scan` → rows are located, and thus **locked**, in **physical heap order** — the order tuples happen to sit on disk pages, which correlates with nothing predictable.
- A *different* statement (`UPDATE ... WHERE supplier_id = 9`) that also Seq Scans will lock its overlapping rows in the *same* heap order — so ironically two Seq Scans on the same table often lock in a **consistent** order (heap order) and are less likely to deadlock than a Seq Scan vs an Index Scan, which visit rows in different orders.
- The takeaway EXPLAIN gives you: **which access path** (Seq Scan = heap order, Index Scan = index order) each contending statement uses. Two statements using *different* access paths over overlapping rows are the classic bulk-update deadlock, and the plan is where you spot the mismatch.

### 7.3 Instrumentation That Does Show Deadlocks

Since EXPLAIN can't, use these instead:

```sql
-- Cluster-wide deadlock counter (per database), since last stats reset
SELECT datname, deadlocks, xact_commit, xact_rollback
FROM pg_stat_database
WHERE datname = current_database();
```
```
 datname  | deadlocks | xact_commit | xact_rollback
----------+-----------+-------------+---------------
 shop_prod|       147 |    88213440 |         21094
```

```sql
-- Live view of who is blocked by whom (the wait-for graph, right now)
SELECT
  blocked.pid            AS blocked_pid,
  blocked.query          AS blocked_query,
  blocking.pid           AS blocking_pid,
  blocking.query         AS blocking_query
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocking
  ON blocking.pid = ANY(pg_blocking_pids(blocked.pid))
WHERE blocked.wait_event_type = 'Lock';
```

`deadlocks` climbing in `pg_stat_database` is your top-level "are we deadlocking?" signal. `pg_blocking_pids()` + `pg_stat_activity` is your "who is currently in a wait cycle?" live probe. Neither is EXPLAIN — deadlocks live in the stats and log subsystems, not the planner.

---

## 8. Query Examples

### Example 1 — Basic: Reproduce a Deadlock Deterministically

```sql
-- Run in two psql sessions, interleaved as commented.
-- Purpose: see 40P01 with your own eyes.

-- ── Session A ──
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;   -- locks row 1
-- (pause here, switch to B)

-- ── Session B ──
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 2;   -- locks row 2
UPDATE accounts SET balance = balance + 100 WHERE id = 1;   -- waits on A's row 1

-- ── Session A ──
UPDATE accounts SET balance = balance + 100 WHERE id = 2;   -- waits on B's row 2
-- ~1s later: one session gets "ERROR: deadlock detected"
```

### Example 2 — Intermediate: The Fix via Consistent Ordering

```sql
-- The SAME logical work (swap 100 between accounts 1 and 2),
-- but both callers lock in ascending id order first → cannot deadlock.

-- ── Both sessions run this identical, order-safe transaction ──
BEGIN;

-- Step 1: acquire ALL row locks up front, in canonical ascending-id order.
-- Whichever session arrives second simply waits for the first to COMMIT,
-- then proceeds — a clean wait, never a cycle.
SELECT id
FROM accounts
WHERE id IN (1, 2)
ORDER BY id                 -- deterministic lock order
FOR UPDATE;

-- Step 2: now do the arithmetic; the order of these UPDATEs no longer
-- matters, because the locks are already held.
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;

COMMIT;
```

### Example 3 — Production Grade: Idempotent, Order-Safe, Retryable Money Transfer

```sql
-- Scenario: a payments service moves funds between wallets.
--   Table: wallets(id BIGINT PK, user_id BIGINT, balance NUMERIC(20,4), version INT)
--          ~5M rows, PK index on id, index on user_id.
--   Table: transfers(id UUID PK, from_wallet BIGINT, to_wallet BIGINT,
--                    amount NUMERIC, idempotency_key TEXT UNIQUE, created_at)
--   Concurrency: hundreds of concurrent transfers/sec; the SAME pair of
--     popular wallets (e.g. a marketplace escrow wallet) is frequently
--     contended from both directions.
--   Perf expectation: each transfer < 3ms server-side; deadlocks driven to
--     ~0 by ordering; any residual deadlock retried by the app (Section 11).

BEGIN;

-- (a) Idempotency guard: if this key was already processed, do nothing.
--     ON CONFLICT DO NOTHING makes a duplicate request a no-op insert.
INSERT INTO transfers (id, from_wallet, to_wallet, amount, idempotency_key)
VALUES (gen_random_uuid(), $1, $2, $3, $4)
ON CONFLICT (idempotency_key) DO NOTHING;

-- (b) Lock BOTH wallets in canonical ascending-id order — the deadlock-
--     prevention core. LEAST/GREATEST guarantee the order regardless of
--     which wallet is the sender.
SELECT id, balance
FROM wallets
WHERE id IN (LEAST($1, $2), GREATEST($1, $2))
ORDER BY id
FOR UPDATE;

-- (c) Debit the sender; the CHECK enforces sufficient funds atomically.
UPDATE wallets
SET balance = balance - $3, version = version + 1
WHERE id = $1 AND balance >= $3;
--   If 0 rows updated → insufficient funds → app rolls back (not a deadlock).

-- (d) Credit the receiver.
UPDATE wallets
SET balance = balance + $3, version = version + 1
WHERE id = $2;

COMMIT;
```

```sql
-- EXPLAIN of the critical locking step (b):
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, balance FROM wallets
WHERE id IN (LEAST(1001, 2002), GREATEST(1001, 2002))
ORDER BY id FOR UPDATE;
```
```
LockRows  (cost=0.43..16.90 rows=2 width=30)
          (actual time=0.028..0.035 rows=2 loops=1)
  ->  Sort  (cost=0.43..16.88 rows=2 width=30)
            (actual time=0.021..0.024 rows=2 loops=1)
        Sort Key: id
        Sort Method: quicksort  Memory: 25kB
        ->  Index Scan using wallets_pkey on wallets
              (cost=0.43..16.87 rows=2 width=30)
              (actual time=0.009..0.014 rows=2 loops=1)
              Index Cond: (id = ANY ('{1001,2002}'::bigint[]))
Buffers: shared hit=8
Planning Time: 0.15 ms
Execution Time: 0.058 ms
```

The `Sort Key: id` under `LockRows` is the visible proof that both directions of transfer lock in the same order — the plan itself certifies the deadlock-prevention property.

---

## 9. Wrong → Right Patterns

### Wrong 1: Locking Two Rows in "Business Order" Instead of Canonical Order

```sql
-- WRONG: lock the SENDER first, then the RECEIVER — i.e. lock in the order
-- the business logic names them. transfer(A→B) locks A then B;
-- transfer(B→A) locks B then A. Opposite orders → deadlock cycle.
BEGIN;
SELECT * FROM accounts WHERE id = :from_id FOR UPDATE;  -- locks :from_id
SELECT * FROM accounts WHERE id = :to_id   FOR UPDATE;  -- then :to_id
UPDATE accounts SET balance = balance - :amt WHERE id = :from_id;
UPDATE accounts SET balance = balance + :amt WHERE id = :to_id;
COMMIT;
```

**The exact failure:** two concurrent transfers in opposite directions between the same pair. `transfer(1→2)` holds row 1, wants row 2. `transfer(2→1)` holds row 2, wants row 1. After `deadlock_timeout`, one gets `ERROR: deadlock detected` (40P01) and its whole transaction rolls back. **Why at the execution level:** each `SELECT ... FOR UPDATE` is a separate lock-acquisition step; the acquisition *order* is dictated by `:from_id`/`:to_id`, which differ between the two callers, so the wait-for graph forms a cycle.

```sql
-- RIGHT: lock BOTH rows in a single statement, in canonical ascending-id order,
-- decoupled from who is sender/receiver.
BEGIN;
SELECT id FROM accounts
WHERE id IN (:from_id, :to_id)
ORDER BY id                    -- canonical order → no cycle possible
FOR UPDATE;
UPDATE accounts SET balance = balance - :amt WHERE id = :from_id;
UPDATE accounts SET balance = balance + :amt WHERE id = :to_id;
COMMIT;
```

### Wrong 2: Catching the Deadlock Error and Swallowing It

```javascript
// WRONG: treat 40P01 as "oh well" and move on — the transaction's work
// is silently lost. The user thinks it succeeded; the money never moved.
try {
  await runTransfer(pool, from, to, amount);
} catch (err) {
  if (err.code === '40P01') {
    logger.warn('deadlock, skipping');   // ★ BUG: the transfer never happened
    return { ok: true };                 // ★ lies to the caller
  }
  throw err;
}
```

**The exact failure:** the victim transaction was fully rolled back — no debit, no credit, no audit row. Swallowing the error reports success while the unit of work vanished. This is worse than crashing, because it is silent and un-alerted.

```javascript
// RIGHT: 40P01 means "safe to retry the WHOLE transaction". Retry with backoff.
async function runTransferWithRetry(pool, from, to, amount, maxAttempts = 5) {
  for (let attempt = 1; ; attempt++) {
    try {
      return await runTransfer(pool, from, to, amount);   // full BEGIN..COMMIT
    } catch (err) {
      if (err.code === '40P01' && attempt < maxAttempts) {
        await sleep(2 ** attempt * 10 + Math.random() * 20); // jittered backoff
        continue;                                            // retry from scratch
      }
      throw err;   // give up after N attempts, or rethrow non-deadlock errors
    }
  }
}
```

### Wrong 3: Retrying a Single Statement Instead of the Whole Transaction

```javascript
// WRONG: catch the deadlock on ONE statement and re-run just that statement
// on the SAME connection/transaction. But a deadlock ABORTS the whole
// transaction — the connection is now in the "aborted" state (25P02).
const client = await pool.connect();
await client.query('BEGIN');
await client.query('UPDATE accounts SET balance = balance - 100 WHERE id = 1');
try {
  await client.query('UPDATE accounts SET balance = balance + 100 WHERE id = 2');
} catch (err) {
  if (err.code === '40P01') {
    // ★ BUG: re-issuing on the same aborted txn:
    await client.query('UPDATE accounts SET balance = balance + 100 WHERE id = 2');
    // → ERROR: current transaction is aborted, commands ignored until
    //   end of transaction block  (SQLSTATE 25P02)
  }
}
```

**The exact failure:** after `40P01`, PostgreSQL puts the transaction into an aborted state; every subsequent command returns `25P02` until you `ROLLBACK`. The first `UPDATE`'s effect is already gone too. You cannot "resume" — you must roll back and replay from `BEGIN`.

```javascript
// RIGHT: on 40P01, ROLLBACK, release, and replay the entire transaction.
async function transferOnce(pool, from, to, amount) {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');
    await client.query(
      'SELECT id FROM accounts WHERE id IN ($1,$2) ORDER BY id FOR UPDATE',
      [from, to]
    );
    await client.query('UPDATE accounts SET balance = balance - $1 WHERE id = $2', [amount, from]);
    await client.query('UPDATE accounts SET balance = balance + $1 WHERE id = $2', [amount, to]);
    await client.query('COMMIT');
  } catch (err) {
    await client.query('ROLLBACK');   // clear the aborted state
    throw err;                        // let the retry wrapper replay BEGIN..COMMIT
  } finally {
    client.release();
  }
}
```

### Wrong 4: Assuming `ORDER BY` in an `UPDATE`'s Subquery Guarantees Lock Order

```sql
-- WRONG: developers assume this locks rows in id order because of ORDER BY.
-- PostgreSQL's UPDATE has no ORDER BY clause, and an ORDER BY inside an
-- IN-subquery does NOT control lock-acquisition order.
UPDATE accounts SET balance = balance + 10
WHERE id IN (SELECT id FROM accounts WHERE user_id = 7 ORDER BY id);
--   The ORDER BY is discarded for a bare IN; rows are locked in scan order.
```

**The exact failure:** the `ORDER BY` in the subquery is not a locking contract — the outer `UPDATE` visits rows in whatever access-path order the planner picks (heap or index). Two such statements over overlapping sets can still deadlock. **Why:** lock order is a property of the *physical access path* of the locking node, not of an `ORDER BY` that isn't driving that path.

```sql
-- RIGHT: force lock order with an explicit locking SELECT first, then update.
BEGIN;
SELECT id FROM accounts
WHERE user_id = 7
ORDER BY id           -- this SELECT's LockRows honors the sort → ordered locks
FOR UPDATE;
UPDATE accounts SET balance = balance + 10 WHERE user_id = 7;
COMMIT;
```

### Wrong 5: Holding a Transaction Open Across an External Call

```javascript
// WRONG: lock rows, then make a slow network call while holding the locks.
// The lock-hold window balloons from microseconds to hundreds of ms,
// massively widening the deadlock window (and blocking other writers).
await client.query('BEGIN');
await client.query('SELECT * FROM orders WHERE id = $1 FOR UPDATE', [orderId]);
const shipping = await fetch('https://carrier.example/rate', {...}); // ★ 300ms held lock
await client.query('UPDATE orders SET shipping_cost = $1 WHERE id = $2',
                   [await shipping.json(), orderId]);
await client.query('COMMIT');
```

**The exact failure:** the row lock on the order is held for the full duration of the HTTP round-trip. Every other transaction needing that order queues behind it; the probability of forming a cycle with some other contended lock rises with the hold time. Under load this produces both deadlocks and general lock pile-ups.

```javascript
// RIGHT: do all slow/external work OUTSIDE the transaction; open the txn only
// to read-lock, mutate, and commit — keep it in the millisecond range.
const rate = await fetch('https://carrier.example/rate', {...}); // no lock held
const cost = (await rate.json()).amount;
await client.query('BEGIN');
await client.query('SELECT * FROM orders WHERE id = $1 FOR UPDATE', [orderId]);
await client.query('UPDATE orders SET shipping_cost = $1 WHERE id = $2', [cost, orderId]);
await client.query('COMMIT');   // total lock-hold: sub-millisecond
```

---

## 10. Performance Profile

### 10.1 The Direct Cost of a Deadlock

| Component | Cost |
|-----------|------|
| Wait before detection | `deadlock_timeout` (default **1000ms**) — the victim was blocked this whole time |
| Detection (`DeadLockCheck` DFS) | microseconds — negligible |
| Rollback of victim | proportional to work done — undo of dirtied buffers, released locks |
| Application retry | one or more full transaction replays + backoff sleep |
| **Total user-visible latency of a deadlocked+retried request** | **~1s (the timeout) + retry time** — often the single worst-tail latency in a write-heavy system |

The 1-second `deadlock_timeout` wait is the dominant cost. A deadlocked request that succeeds on retry still took ~1s+ end-to-end. This is why deadlocks show up as **p99/p999 latency spikes**, not average-latency regressions.

### 10.2 Deadlock Rate vs Concurrency

For N transactions concurrently contending the same small set of rows with random lock orders, the deadlock probability per transaction scales roughly with N (and total deadlocks with N²). Concretely, on a hot pair of rows:

| Concurrent writers on hot rows | Approx deadlock rate (random order) | With consistent ordering |
|-------------------------------|-------------------------------------|--------------------------|
| 2 | rare (needs precise interleave) | 0 |
| 10 | occasional (seconds between) | 0 |
| 100 | frequent (many per second) | 0 |
| 500 | deadlock storm; retries pile up | 0 |

Consistent ordering collapses the entire column to **0** — this is why ordering, not tuning, is the real fix. Tuning `deadlock_timeout` only changes *how fast* you detect the storm, not whether it happens.

### 10.3 Scaling at 1M / 10M / 100M Rows

Row *count* barely affects deadlock frequency — deadlocks depend on **contention on specific rows**, not table size. What changes with scale:

- **1M rows:** if writes are spread uniformly, hot-row contention is low; deadlocks negligible. Danger is a few "celebrity" rows (a global counter, an escrow wallet, a popular product's stock).
- **10M rows:** bulk `UPDATE`/`DELETE` jobs now touch large row sets; two such jobs with different access paths (Seq vs Index scan) become a real deadlock source. Batch them with consistent `ORDER BY id` and small chunk sizes.
- **100M rows:** long-running bulk operations hold thousands of row locks for extended periods. Combine consistent ordering + chunking (e.g. `WHERE id BETWEEN x AND x+10000 ORDER BY id FOR UPDATE`) + short transactions so each chunk commits and releases locks promptly. Also watch `max_locks_per_transaction` — a huge single transaction can exhaust the shared lock table.

### 10.4 Tuning Knobs and Their Trade-offs

| Knob | Effect | When to change |
|------|--------|----------------|
| `deadlock_timeout` | delay before detection runs | Lower (e.g. 500ms) if deadlocks are common and the 1s wait dominates latency — but detection CPU rises. Rarely raise it. |
| `log_lock_waits = on` | logs waits > `deadlock_timeout` | **Always on** in production — free visibility into contention. |
| `lock_timeout` | statement aborts after waiting this long for any lock | Set on user-facing writes (e.g. 2s) so a wedged lock fails fast and retries. |
| `max_locks_per_transaction` | size of shared lock table | Raise if bulk jobs hit "out of shared memory / max_locks" errors. |
| `default_transaction_isolation` | higher isolation → more `40001` serialization failures (a sibling of deadlocks) | Use `SERIALIZABLE` only where you need it; it adds retryable errors of its own. |

### 10.5 Deadlock vs Serialization Failure (a Performance-Relevant Distinction)

Two different retryable errors, often conflated:

- **`40P01` deadlock_detected** — a lock-cycle at any isolation level. Caused by lock ordering.
- **`40001` serialization_failure** — only at `REPEATABLE READ`/`SERIALIZABLE`; the engine detected that committing would violate serializability. Caused by isolation, not lock order.

Both are safe to **retry the whole transaction**. Your retry wrapper should treat `40P01` **and** `40001` identically (retry with backoff). A system that retries only one and not the other will still leak failures under load.

---

## 11. Node.js Integration

### 11.1 The Core Retry Wrapper (the thing every write path needs)

```javascript
import pg from 'pg';
const { Pool } = pg;
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

// SQLSTATEs that mean "your transaction was aborted for a concurrency reason;
// it is SAFE to replay the entire transaction from BEGIN."
const RETRYABLE = new Set([
  '40P01', // deadlock_detected
  '40001', // serialization_failure (REPEATABLE READ / SERIALIZABLE)
]);

const sleep = (ms) => new Promise((r) => setTimeout(r, ms));

/**
 * Runs `work(client)` inside a transaction, retrying the WHOLE transaction
 * on deadlock/serialization failure with exponential backoff + full jitter.
 * `work` must be a pure function of its inputs — it may run more than once.
 */
export async function withTxnRetry(work, { maxAttempts = 5, baseMs = 10 } = {}) {
  for (let attempt = 1; ; attempt++) {
    const client = await pool.connect();
    try {
      await client.query('BEGIN');
      const result = await work(client);   // caller issues its statements here
      await client.query('COMMIT');
      return result;
    } catch (err) {
      // Roll back to clear the aborted-transaction state (25P02) on this conn.
      try { await client.query('ROLLBACK'); } catch { /* conn may be dead */ }

      if (RETRYABLE.has(err.code) && attempt < maxAttempts) {
        // Full-jitter backoff: random in [0, base * 2^attempt].
        const cap = baseMs * 2 ** attempt;
        await sleep(Math.random() * cap);
        continue;                          // replay the entire transaction
      }
      throw err;                           // non-retryable, or out of attempts
    } finally {
      client.release();
    }
  }
}
```

### 11.2 Using It — the Order-Safe Transfer

```javascript
async function transfer(fromId, toId, amount) {
  return withTxnRetry(async (client) => {
    // Lock BOTH rows in canonical ascending-id order (deadlock prevention).
    await client.query(
      `SELECT id FROM accounts
       WHERE id IN ($1, $2)
       ORDER BY id
       FOR UPDATE`,
      [fromId, toId]
    );

    // Debit sender only if funds suffice; capture affected row count.
    const debit = await client.query(
      `UPDATE accounts SET balance = balance - $1
       WHERE id = $2 AND balance >= $1`,
      [amount, fromId]
    );
    if (debit.rowCount === 0) {
      // Not a deadlock — a business rule. Throw a NON-retryable error so the
      // wrapper does not pointlessly replay it.
      const e = new Error('insufficient funds');
      e.code = 'INSUFFICIENT_FUNDS';
      throw e;
    }

    await client.query(
      `UPDATE accounts SET balance = balance + $1 WHERE id = $2`,
      [amount, toId]
    );
    return { fromId, toId, amount };
  });
}
```

### 11.3 Observing Deadlocks From the App Side

```javascript
// A thin instrumentation layer: count retries and surface deadlock metrics.
export async function withTxnRetryInstrumented(work, opts, metrics) {
  let attempts = 0;
  try {
    return await withTxnRetry(async (client) => {
      attempts++;
      return work(client);
    }, opts);
  } finally {
    if (attempts > 1) {
      metrics.increment('db.txn.retries', attempts - 1);
      metrics.increment('db.txn.deadlock_recovered');
    }
  }
}
```

### 11.4 Important pg-specific notes

- **`err.code` is the SQLSTATE**, exposed directly by `node-postgres` on the thrown error (e.g. `'40P01'`). Do **not** string-match on `err.message` — the text is localizable and version-dependent; the code is stable.
- **You must `ROLLBACK` on the erroring client before reuse.** After `40P01`, the connection is in an aborted state; the pool will return a poisoned connection if you `release()` without rolling back. The wrapper above always rolls back first.
- **`work` may execute multiple times**, so it must be **idempotent or side-effect-free outside the DB** — do not send an email or call an external API inside `work`; do that after `withTxnRetry` resolves.
- **A fresh connection per attempt** (as above) is the safest pattern — it guarantees clean transaction state even if `ROLLBACK` itself failed on a broken connection.
- **Cap attempts and add jitter.** Un-jittered retries of two colliding transactions can re-collide in lockstep forever; full jitter de-synchronizes them.

---

## 12. ORM Comparison

The question for each ORM: **does it provide (a) a way to lock rows in a controlled order, and (b) automatic transaction retry on `40P01`?** Almost none provide (b) out of the box — you supply the retry loop. All provide (a) via row-locking clauses (Topic 54) or raw SQL.

### Prisma

**Can it?** Locking order: only via `$queryRaw` (`FOR UPDATE` with `ORDER BY`) — Prisma's fluent API has no `FOR UPDATE`. Retry: not built in.

```typescript
import { PrismaClient, Prisma } from '@prisma/client';
const prisma = new PrismaClient();

async function withPrismaRetry<T>(fn: () => Promise<T>, max = 5): Promise<T> {
  for (let attempt = 1; ; attempt++) {
    try {
      return await fn();
    } catch (e) {
      // Prisma surfaces deadlocks as P2034 (write conflict / deadlock) OR
      // as a raw PostgresError with code 40P01 when using $queryRaw.
      const isDeadlock =
        (e instanceof Prisma.PrismaClientKnownRequestError && e.code === 'P2034') ||
        (e as any)?.code === '40P01';
      if (isDeadlock && attempt < max) {
        await new Promise((r) => setTimeout(r, Math.random() * 10 * 2 ** attempt));
        continue;
      }
      throw e;
    }
  }
}

async function transfer(fromId: bigint, toId: bigint, amount: number) {
  return withPrismaRetry(() =>
    prisma.$transaction(async (tx) => {
      // Ordered lock must be raw — Prisma has no FOR UPDATE builder.
      await tx.$queryRaw`
        SELECT id FROM accounts
        WHERE id IN (${fromId}, ${toId})
        ORDER BY id
        FOR UPDATE`;
      await tx.$executeRaw`UPDATE accounts SET balance = balance - ${amount} WHERE id = ${fromId}`;
      await tx.$executeRaw`UPDATE accounts SET balance = balance + ${amount} WHERE id = ${toId}`;
    })
  );
}
```

**Where it breaks:** no `FOR UPDATE` in the type-safe API; you drop to raw SQL for ordered locking. Prisma reports deadlocks inconsistently (`P2034` for its own detection vs `40P01` from raw). **Verdict:** usable, but you own both the ordering (raw) and the retry loop.

### Drizzle ORM

**Can it?** Locking order: yes — `.for('update')` plus `.orderBy()`. Retry: not built in.

```typescript
import { db } from './db';
import { accounts } from './schema';
import { inArray, asc, sql } from 'drizzle-orm';

async function transfer(fromId: number, toId: number, amount: number) {
  return withTxnRetry(() =>                       // your own retry wrapper
    db.transaction(async (tx) => {
      await tx
        .select({ id: accounts.id })
        .from(accounts)
        .where(inArray(accounts.id, [fromId, toId]))
        .orderBy(asc(accounts.id))                // canonical lock order
        .for('update');                           // FOR UPDATE
      await tx.update(accounts)
        .set({ balance: sql`${accounts.balance} - ${amount}` })
        .where(sql`${accounts.id} = ${fromId} AND ${accounts.balance} >= ${amount}`);
      await tx.update(accounts)
        .set({ balance: sql`${accounts.balance} + ${amount}` })
        .where(sql`${accounts.id} = ${toId}`);
    })
  );
}
```

**Where it breaks:** nothing for ordering — `.for('update')` + `.orderBy(asc(...))` maps cleanly to `LockRows` over a sorted input. You still write the retry loop (catch `err.code === '40P01'`). **Verdict:** best fit — type-safe ordered locking; bring your own retry.

### Sequelize

**Can it?** Locking order: yes — `{ lock: t.LOCK.UPDATE, order: [...] }` in a transaction. Retry: partial — Sequelize has `retry` config but it targets connection-level errors, not per-transaction deadlock replay reliably; write your own.

```javascript
const { Transaction } = require('sequelize');

async function transfer(fromId, toId, amount) {
  return withTxnRetry(() =>
    sequelize.transaction(async (t) => {
      await Account.findAll({
        where: { id: [fromId, toId] },
        order: [['id', 'ASC']],           // canonical order
        lock: t.LOCK.UPDATE,              // FOR UPDATE
        transaction: t,
      });
      await Account.decrement('balance', { by: amount, where: { id: fromId }, transaction: t });
      await Account.increment('balance', { by: amount, where: { id: toId }, transaction: t });
    })
  );
  // In the catch of withTxnRetry, deadlocks arrive as err.parent.code === '40P01'
  // (Sequelize wraps the pg error; the original SQLSTATE is on err.parent/err.original).
}
```

**Where it breaks:** the SQLSTATE is nested on `err.parent.code` / `err.original.code`, not `err.code` — a common reason retry logic silently never fires. **Verdict:** capable of ordered locking; be careful to unwrap the pg error code.

### TypeORM

**Can it?** Locking order: yes — `.setLock('pessimistic_write')` + `.orderBy()` in QueryBuilder. Retry: not built in.

```typescript
async function transfer(fromId: number, toId: number, amount: number) {
  return withTxnRetry(() =>
    dataSource.transaction(async (em) => {
      await em.createQueryBuilder(Account, 'a')
        .setLock('pessimistic_write')             // FOR UPDATE
        .where('a.id IN (:...ids)', { ids: [fromId, toId] })
        .orderBy('a.id', 'ASC')                   // canonical order
        .getMany();
      await em.decrement(Account, { id: fromId }, 'balance', amount);
      await em.increment(Account, { id: toId }, 'balance', amount);
    })
  );
  // Deadlock code arrives as err.driverError.code === '40P01'.
}
```

**Where it breaks:** as with Sequelize, the pg SQLSTATE is nested (`err.driverError.code`). `pessimistic_write` + `orderBy` is correct for ordered locking. **Verdict:** solid; unwrap `driverError` for the code.

### Knex.js

**Can it?** Locking order: yes — `.forUpdate()` + `.orderBy()`. Retry: not built in.

```javascript
async function transfer(fromId, toId, amount) {
  return withTxnRetry(() =>
    knex.transaction(async (trx) => {
      await trx('accounts')
        .whereIn('id', [fromId, toId])
        .orderBy('id', 'asc')            // canonical order
        .forUpdate();                    // FOR UPDATE
      await trx('accounts').where('id', fromId).andWhere('balance', '>=', amount)
        .decrement('balance', amount);
      await trx('accounts').where('id', toId).increment('balance', amount);
    })
  );
  // Deadlock arrives as err.code === '40P01' (Knex/pg passes it through directly).
}
```

**Where it breaks:** almost nothing — Knex passes the pg error through with `err.code` intact, and `.forUpdate().orderBy()` is transparent SQL. **Verdict:** most transparent; `err.code === '40P01'` works directly.

### ORM Summary Table

| ORM | Ordered `FOR UPDATE` | Retry built in? | Where the SQLSTATE lives | Verdict |
|-----|----------------------|-----------------|--------------------------|---------|
| Prisma | Raw SQL only | No | `P2034` or raw `40P01` | Raw for locking; own retry |
| Drizzle | `.for('update')` + `.orderBy(asc)` | No | `err.code` | Best typed fit; own retry |
| Sequelize | `lock: LOCK.UPDATE` + `order` | Partial/unreliable | `err.parent.code` | Unwrap parent error |
| TypeORM | `setLock('pessimistic_write')` + `orderBy` | No | `err.driverError.code` | Unwrap driverError |
| Knex | `.forUpdate()` + `.orderBy()` | No | `err.code` | Most transparent |

**Universal truth:** no mainstream Node ORM prevents deadlocks for you, and retry is your responsibility on every ORM. The two things you always own: **(1) consistent lock ordering** (via ordered `FOR UPDATE`) and **(2) a transaction-level retry loop** keyed on `40P01`/`40001`.

---

## 13. Practice Exercises

### Exercise 1 — Easy

You have:

```sql
CREATE TABLE seats (
  id          INT PRIMARY KEY,
  event_id    INT NOT NULL,
  status      TEXT NOT NULL DEFAULT 'free',  -- 'free' | 'held' | 'sold'
  held_by     INT
);
```

Two concurrent booking requests each try to hold seats 12 and 17. Request X runs `UPDATE seats SET status='held' WHERE id=12` then `... WHERE id=17`. Request Y runs the same two updates but in the order 17 then 12.

1. Explain, step by step, how these two requests can deadlock.
2. Rewrite the booking transaction so the two requests **cannot** deadlock, using a single locking statement. Assume both requests want the same two seat ids.

```sql
-- Write your query here
```

---

### Exercise 2 — Medium (combines Topic 54 SELECT FOR UPDATE)

Using `seats` from Exercise 1 plus:

```sql
CREATE TABLE holds (
  id        SERIAL PRIMARY KEY,
  seat_id   INT NOT NULL REFERENCES seats(id),
  user_id   INT NOT NULL,
  expires_at TIMESTAMPTZ NOT NULL
);
```

Write a transaction that, given a **list of seat ids** for a single booking, does the following without any deadlock risk between concurrent bookings that share seats:

1. Locks all requested seats **in ascending id order** with `FOR UPDATE`.
2. Fails the whole booking (raise/return) if **any** requested seat is not currently `'free'`.
3. Marks all of them `'held'` and inserts a `holds` row for each with a 10-minute expiry.

Explain why locking in ascending id order is what makes concurrent overlapping bookings safe, and why doing the "is it free?" check must come **after** the lock, not before.

```sql
-- Write your query here
```

---

### Exercise 3 — Hard (production simulation; the naive answer is wrong)

A background job reconciles inventory nightly:

```sql
-- Job A (runs at 02:00): apply returns, adding stock back
UPDATE products SET stock = stock + r.qty
FROM nightly_returns r
WHERE r.product_id = products.id;

-- Job B (runs at 02:00, different worker): apply shrinkage, removing stock
UPDATE products SET stock = stock - s.qty
FROM nightly_shrinkage s
WHERE s.product_id = products.id;
```

Both jobs frequently touch overlapping product rows and started deadlocking after the catalog grew to 10M products. The naive fix "just add a retry loop" makes the jobs *eventually* finish but they now take 40 minutes with constant deadlock churn.

1. Explain the physical reason these two bulk `UPDATE`s deadlock (think about access paths and lock-acquisition order).
2. Propose a rewrite that eliminates the deadlocks **structurally** (not just retries), processing in bounded chunks. Both jobs must converge to the same lock order.
3. What index and what chunking key make your rewrite efficient on 10M rows, and why does chunking also help `max_locks_per_transaction`?

```sql
-- Write your query here
```

---

### Exercise 4 — Interview Simulation

You are on call. Alerts show `pg_stat_database.deadlocks` for `shop_prod` jumped from ~5/hour to ~4,000/hour starting 14 minutes ago, correlating with a deploy. Checkout success rate dropped 6%.

1. What are the first three things you query/inspect, and in what order?
2. The server log shows repeated deadlock reports where both statements are `UPDATE inventory SET reserved = reserved + $1 WHERE sku_id = $2`, but with different `sku_id` values in the `CONTEXT` lines. What does that tell you about the new code path?
3. Give an immediate mitigation (minutes) and a proper fix (the deploy-worthy change).

```sql
-- Write your query here (diagnostic queries + the fix)
```

---

## 14. Interview Questions

### Q1 — What exactly is a deadlock in PostgreSQL, and how does the engine get out of one?

**Junior answer:** "It's when two queries lock each other and the database freezes, so PostgreSQL kills one of them."

**Principal answer:** A deadlock is a cycle in the lock manager's wait-for graph: transaction A holds a lock B wants, and B holds a lock A wants (possibly through a longer chain A→B→C→A). Row locks are stored in the tuple header (`xmax`), and waiting is modeled as taking a `ShareLock` on the holder's transaction id, so the engine can represent "who waits for whom" as a directed graph. PostgreSQL does not check for cycles on every wait — that would be wasteful since most waits are short. Instead, when a transaction blocks, it arms a timer for `deadlock_timeout` (default 1s). If still waiting when the timer fires, that backend runs `DeadLockCheck`, a DFS over the wait-for graph. If it finds a cycle it can't resolve by reordering wait queues (soft edges), it aborts the transaction that ran the check — the victim — with SQLSTATE `40P01`, releasing its locks so the survivors proceed. The victim's whole transaction is rolled back; the application must retry it.

**Interviewer follow-up:** "So who gets chosen as the victim, and can I influence it?" — Effectively the transaction whose `deadlock_timeout` expired and ran the check; there is no priority mechanism, so you must assume any participant (including your critical one) can be the victim, which is why retry is mandatory on all contending write paths.

---

### Q2 — Two services both transfer money between the same two accounts and you're seeing deadlocks. Walk me through the cause and the fix.

**Junior answer:** "Add a try/catch and retry the query when it fails."

**Principal answer:** The cause is almost certainly opposite lock ordering: `transfer(A→B)` locks A then B, while `transfer(B→A)` locks B then A. Concurrently, the first holds A and waits for B, the second holds B and waits for A — a cycle. Retrying alone treats the symptom: under load the two keep re-colliding and you burn ~1s per deadlock (the timeout) plus retries, spiking p99 latency. The structural fix is **consistent lock ordering**: always lock the involved rows in a canonical order — e.g. ascending primary key — regardless of business direction. Concretely, before any writes, run `SELECT id FROM accounts WHERE id IN (from, to) ORDER BY id FOR UPDATE`. Now both directions lock `min(id)` first, so the second arrival cleanly waits instead of forming a cycle. There's a proof: if every transaction acquires locks in ascending order, a cycle would require a lock id greater than itself, which is impossible. I'd keep a bounded, jittered retry loop as a safety net for residual cases (and for `40001` serialization failures), but ordering is what actually eliminates the storm.

**Interviewer follow-up:** "Why not just set a higher isolation level or use a bigger lock?" — Higher isolation adds `40001` retryable errors, not fewer; a coarser lock (e.g. table lock) serializes all transfers and destroys throughput. Ordered row locks give correctness with maximal concurrency.

---

### Q3 — After a deadlock, your app catches the error. What must the application actually do, and what are the common mistakes?

**Junior answer:** "Log it and return an error to the user, or just retry the failed statement."

**Principal answer:** On `40P01` the entire transaction is aborted — the connection is now in the aborted state (`25P02`), so you cannot re-run just the failed statement; any further command errors until you `ROLLBACK`. The correct response is: `ROLLBACK` (or discard the connection), then **replay the whole transaction from `BEGIN`**, ideally on a fresh connection, with exponential backoff plus **full jitter** to avoid re-colliding in lockstep, and a bounded attempt count. Three common mistakes: (1) swallowing the error and reporting success — silently loses the unit of work; (2) retrying a single statement on the same aborted transaction — yields `25P02`; (3) retrying without jitter — two colliding transactions re-synchronize and deadlock again. Also, the transaction body must be safe to run more than once — no external side effects (emails, API calls) inside it; those go after commit. And retry `40001` with the same machinery, since it's the isolation-level sibling.

**Interviewer follow-up:** "How do you detect the deadlock code in Node?" — Check `err.code === '40P01'` from node-postgres; but ORMs often nest it (`err.parent.code` in Sequelize, `err.driverError.code` in TypeORM, `P2034`-or-raw in Prisma), and matching `err.message` text is wrong because it's localizable.

---

### Q4 — Can a plain SELECT deadlock? Can a bulk UPDATE with no explicit FOR UPDATE deadlock?

**Junior answer:** "SELECT can deadlock if the table is busy. Bulk UPDATE probably not, it's one statement."

**Principal answer:** A plain, non-locking `SELECT` cannot deadlock — MVCC gives it a snapshot; it never waits on row locks, so it can't be part of a wait cycle. (A `SELECT ... FOR UPDATE`/`FOR SHARE` *can*, because it takes real locks.) Conversely, a single bulk `UPDATE` absolutely can deadlock against another statement, even though it's "one statement," because it acquires row locks **incrementally as it scans**, in the physical order of its access path (heap order for Seq Scan, index order for Index Scan). Two bulk UPDATEs over overlapping rows using different access paths lock those rows in different orders and form a cycle. That's why bulk maintenance jobs need consistent ordering (an ordered locking `SELECT` first, or matching `ORDER BY`/chunking) even when there's no explicit `FOR UPDATE` in the original code.

**Interviewer follow-up:** "How would you confirm the access-path theory?" — `EXPLAIN` each UPDATE: if one shows `Seq Scan` and the other `Index Scan` over the same table, they visit rows in different orders — the smoking gun. Fix by forcing both onto the same order.

---

### Q5 — What's the difference between `deadlock_timeout`, `lock_timeout`, and `statement_timeout`, and how do you use them together?

**Junior answer:** "They're all timeouts for slow queries."

**Principal answer:** They operate at different layers. `deadlock_timeout` (default 1s) is *not* a cap on anything — it's the delay before the deadlock **detector** runs after a transaction starts waiting; lowering it makes detection faster (at more CPU), it doesn't prevent deadlocks. `lock_timeout` caps how long a *single statement* will wait for *any* lock before aborting with `55P03` — a blunt safety net so a wedged lock can't hang a request; it fires on ordinary long waits too, not just deadlocks. `statement_timeout` caps *total* statement runtime regardless of cause. In production I'd set `log_lock_waits = on` (free visibility), a modest `lock_timeout` (e.g. 2s) and `statement_timeout` on user-facing write paths so requests fail fast and retry rather than pile up, and I'd generally leave `deadlock_timeout` at 1s unless deadlocks are frequent enough that the 1s detection wait itself dominates latency — then I might lower it, accepting the extra detection cost.

**Interviewer follow-up:** "If I set `lock_timeout` very low, do I still need deadlock detection?" — A low `lock_timeout` would break most deadlocks by timing out one waiter before the 1s detector runs, but it also aborts many *legitimate* waits, hurting throughput; detection is more surgical because it only acts on actual cycles.

---

## 15. Mental Model Checkpoint

1. **Two application threads each run a transaction that updates two rows in `orders`. Thread 1 updates order 100 then order 200; thread 2 updates order 200 then order 100. Under what timing do they deadlock, and under what timing do they *not*? Why can't you fix this by making the transactions faster?**

2. **A colleague says "we added a `lock_timeout` of 500ms, so we don't need to handle deadlocks anymore." Is the deadlock *error* (`40P01`) still possible? What does `lock_timeout` actually change about which transaction aborts, and what new failure code (`55P03`) now appears?**

3. **You have two write paths: one updates `orders` then `payments`, the other updates `payments` then `orders` (both for the same order). Neither uses explicit `FOR UPDATE`. Explain precisely why this deadlocks even though nobody wrote a `LOCK` statement, and give the single ordering rule that eliminates it.**

4. **A single bulk `UPDATE orders SET status='shipped' WHERE ...` deadlocks against another single bulk `UPDATE orders SET priority=...`. There is no explicit transaction ordering in the code — it's one statement each. How can two *single statements* form a lock cycle? What does `EXPLAIN` on each need to show for your theory to hold?**

5. **Your deadlock victim retry logic re-runs the failed transaction immediately, with no delay. Two colliding transactions retry in perfect lockstep and deadlock again, repeatedly. What is this pattern called, and why does exponential backoff *with jitter* fix it where plain backoff does not?**

6. **A foreign key from `order_items.order_id` to `orders.id` is involved in a deadlock, yet the developer swears they only ran plain `INSERT`s into `order_items` and never touched `orders`. What lock does the FK check take on the parent `orders` row, and how can two inserts referencing overlapping parents deadlock?**

7. **You inherit a system with frequent deadlocks between an `audit_logs` writer and the main `orders` writer. You cannot change the lock *order* because the two code paths genuinely need the rows in opposite orders. What are two strategies (other than reordering) that break the cycle — think about lock scope, transaction boundaries, and whether the audit write even needs to be in the same transaction?**

---

## 16. Quick Reference Card

```sql
-- WHAT A DEADLOCK IS
-- A cycle of waiters: T1 holds lock A, wants B; T2 holds B, wants A.
-- Postgres detects the cycle and aborts one transaction (the "victim"):
--   ERROR:  deadlock detected            SQLSTATE 40P01
--   (victim's whole transaction is rolled back)

-- THE #1 PREVENTION RULE: consistent lock ordering
-- Always acquire rows/tables in the SAME order in every code path.
-- Order by a stable key (e.g. ascending id):
BEGIN;
  UPDATE orders   SET status = 'paid' WHERE id = 100;   -- lower id first
  UPDATE payments SET state  = 'done' WHERE order_id = 100;
COMMIT;

-- Force ordered row locking before doing the work:
SELECT id FROM orders
WHERE id IN (100, 200)
ORDER BY id                 -- deterministic acquisition order
FOR UPDATE;

-- HIDDEN LOCK SOURCES (deadlock without any explicit LOCK)
--  * UPDATE / DELETE                -> row locks, acquired as the scan proceeds
--  * SELECT ... FOR UPDATE / SHARE  -> explicit row locks
--  * Foreign-key inserts/updates    -> FOR KEY SHARE lock on the PARENT row
--  * Bulk UPDATE via different       -> two statements lock rows in different
--    access paths (Seq vs Index)       physical orders => cycle

-- RELEVANT TIMEOUTS (different layers — not interchangeable)
SET deadlock_timeout  = '1s';    -- delay before the detector RUNS (not a cap)
SET lock_timeout      = '2s';    -- max wait for ONE lock  -> aborts 55P03
SET statement_timeout = '5s';    -- max total statement runtime (any cause)
SET log_lock_waits    = on;      -- log any wait longer than deadlock_timeout

-- DIAGNOSE / OBSERVE
SHOW deadlock_timeout;
SELECT * FROM pg_locks WHERE NOT granted;          -- who is waiting
SELECT * FROM pg_stat_activity WHERE wait_event_type = 'Lock';
-- Turn on log_lock_waits + check server log for the deadlock CONTEXT block,
-- which prints both statements and both PIDs involved in the cycle.

-- RETRY THE VICTIM (application side) — pseudo-Node
--   if (err.code === '40P01') { ROLLBACK; retry whole txn with backoff+jitter }
--   * Retry the ENTIRE transaction from BEGIN, not the single statement
--     (the connection is in aborted state 25P02 after 40P01)
--   * Exponential backoff + FULL JITTER (avoid lockstep re-collision)
--   * Bounded attempts; keep side effects (emails, API calls) OUTSIDE the txn
--   * Retry 40001 (serialization_failure) with the same machinery
```

| Code | Meaning | Who aborts | Typical fix |
|------|---------|-----------|-------------|
| `40P01` | deadlock detected | detector's chosen victim | consistent lock order + retry |
| `55P03` | lock_not_available | statement that hit `lock_timeout` | shorter txns / fail fast + retry |
| `25P02` | in_failed_sql_transaction | any stmt after a prior error | `ROLLBACK`, replay whole txn |
| `40001` | serialization_failure | SERIALIZABLE/REPEATABLE READ conflict | retry whole txn (same as 40P01) |

---

## Connected Topics

- **Topic 50 — Transactions and ACID**: A deadlock is resolved by rolling back one transaction; atomicity is what makes that safe — the victim's partial work vanishes cleanly, leaving no torn state to clean up.
- **Topic 51 — Isolation Levels**: Higher isolation (`REPEATABLE READ`, `SERIALIZABLE`) trades deadlock-style aborts (`40P01`) for serialization aborts (`40001`); both demand the same whole-transaction retry loop, so the machinery you build here is reused directly.
- **Topic 52 — Locking and Lock Modes**: The full lock-mode compatibility matrix — which locks conflict with which — is the raw material from which deadlock cycles are built; you cannot reason about a cycle without it.
- **Topic 53 — Row-Level vs Table-Level Locks**: Deadlocks most often form on row locks acquired incrementally during a scan; understanding row-lock granularity explains why bulk `UPDATE`s with different access paths collide.
- **Topic 54 — SELECT FOR UPDATE**: The primary tool for *imposing* a deterministic lock order (`ORDER BY ... FOR UPDATE`) — the single most effective structural fix for the deadlocks described here.
- **Topic 56 — Optimistic vs Pessimistic Locking**: Optimistic concurrency (version columns, compare-and-swap) sidesteps lock cycles entirely by never holding locks across the read-modify-write gap — the main alternative to the pessimistic ordering discipline in this topic.
- **Topic 18 — The N+1 Query Problem**: The retry-with-backoff pattern for deadlock victims lives in the same application data layer where N+1 fixes and connection pooling decisions are made; both are properties of how the app drives the database, not the SQL itself.
