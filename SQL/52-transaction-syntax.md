# Topic 52 — Transaction Syntax in Depth
### SQL Mastery Curriculum — Phase 8: Transactions and Concurrency in SQL

---

## 1. ELI5 — The Analogy That Makes It Click

Imagine you're moving into a new apartment and you hire a moving company. The deal is simple: **either every single box makes it into the new place, or none of them do and you get all your stuff back exactly where it was.** There is no acceptable middle state where half your furniture is in the truck, your bed is on the sidewalk, and the movers have gone home.

That "all-or-nothing" promise is a **transaction**.

Now add some structure to the move:

- **BEGIN** — the movers arrive and start loading. From this moment on, nothing is final. Everything they touch is "in progress."
- **COMMIT** — the movers finish, you sign the paperwork, and the move is now permanent. Everything they did is real and cannot be undone.
- **ROLLBACK** — halfway through, you discover the new apartment has a burst pipe. You say "stop, put everything back." The movers reverse every single action and your old apartment looks exactly as it did before they arrived. It's as if the move never happened.

Now the clever part — **SAVEPOINT**:

Suppose the movers are loading room by room. After the kitchen is packed, you say: "mark this — the kitchen is done, remember this exact moment." That mark is a **savepoint**. If something goes wrong while packing the bedroom, you don't have to abandon the entire move — you can say **ROLLBACK TO SAVEPOINT** and rewind only back to "kitchen done," keeping the kitchen work but undoing the bedroom mess. And when you're confident the kitchen truly is fine and you'll never need to rewind to it, you say **RELEASE SAVEPOINT** — you tear up that bookmark because you no longer need it.

Finally, the **aborted state**: if a mover drops a box and shatters it in a way that violates the contract (an error), the whole move freezes. Nobody can do any more work. Your only two options are "put everything back" (ROLLBACK) or, if you were smart enough to leave a bookmark, "rewind to the last bookmark" (ROLLBACK TO SAVEPOINT). You cannot just pretend nothing happened and keep loading.

That is the complete transaction control language: `BEGIN`, `COMMIT`, `ROLLBACK`, `SAVEPOINT`, `ROLLBACK TO SAVEPOINT`, `RELEASE SAVEPOINT` — plus the two states a transaction can be in (in-progress and aborted). Everything else in this topic is mechanism behind that promise.

---

## 2. Connection to SQL Internals

A transaction is not a syntactic nicety — it is the boundary within which PostgreSQL's entire concurrency and durability machinery operates. When you type `BEGIN`, several internal subsystems come alive.

**MVCC (Multi-Version Concurrency Control).** PostgreSQL never overwrites a row in place for a concurrent reader. Every transaction is assigned a **transaction ID** (`xid`) from a monotonically increasing counter. When your transaction inserts or updates a row, the new row version (a *tuple*) is stamped with `xmin = your_xid` (the transaction that created it). When you delete or update a row, the old version is stamped with `xmax = your_xid` (the transaction that expired it). A row version is visible to a given transaction only if its `xmin` is committed-and-visible and its `xmax` is either empty or not-yet-visible. `COMMIT` is what flips your `xid`'s status in the **commit log (pg_xact / clog)** to "committed," which is the single instant that makes all your tuples visible to others. `ROLLBACK` flips it to "aborted," and every tuple you wrote is now dead weight that VACUUM will later reclaim.

**WAL (Write-Ahead Log).** Durability — the "D" in ACID — comes from WAL. Before any data page modification is flushed to the heap, a WAL record describing that change is written to the log and, at `COMMIT`, flushed to disk (`fsync`). The commit is not acknowledged to your client until the WAL record for the commit is durably on disk (unless `synchronous_commit` is relaxed). This is why a `COMMIT` can be the slowest statement in your transaction: it is the fsync barrier. A `ROLLBACK`, by contrast, typically writes very little — it just marks the transaction aborted; there is no need to flush undo, because MVCC keeps old versions around rather than overwriting them.

**No undo log (unlike Oracle/MySQL-InnoDB).** This is a defining PostgreSQL design choice. Because MVCC keeps old row versions in the heap itself, a `ROLLBACK` is *cheap* — there is nothing to physically undo. The dead tuples are simply never made visible and are cleaned up asynchronously by VACUUM. The tradeoff is table bloat from aborted/updated tuples, which is why VACUUM exists.

**Snapshots.** At the start of a statement (READ COMMITTED) or the start of the transaction (REPEATABLE READ / SERIALIZABLE), PostgreSQL takes a **snapshot**: the set of transaction IDs that were committed at that instant. This snapshot is what your queries "see." Transaction syntax is the API that controls when snapshots are taken — and Topic 53 (Isolation Levels) is entirely about that timing.

**Subtransactions.** A `SAVEPOINT` opens a **subtransaction**, which gets its own `xid` (a sub-xid) tracked in the `pg_subtrans` structure. Rolling back to a savepoint marks that sub-xid (and any nested ones) aborted while keeping the parent alive. This is how "partial rollback" is physically possible.

**The transaction ID counter and wraparound.** `xid` is a 32-bit number. Every writing transaction consumes one. If they were never reused, the counter would exhaust — so PostgreSQL treats the xid space as a circle, and VACUUM "freezes" old tuples to prevent *transaction ID wraparound*, a catastrophic failure mode. Your `BEGIN`/`COMMIT` discipline directly affects how fast xids are consumed (long-running or idle-in-transaction sessions hold back the "oldest xid" horizon and stall VACUUM).

The takeaway: transaction syntax is the user-facing control surface for MVCC visibility, WAL durability, and subtransaction bookkeeping. Every keyword in this topic corresponds to a concrete internal state change.

---

## 3. Logical Execution Order Context

Transactions do not live inside the `FROM → JOIN → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → window → ORDER BY → LIMIT` pipeline — they **wrap** it. A transaction is a container *around* whole statements, not a clause within one statement.

```
BEGIN;                          ← opens the transaction block, assigns snapshot policy
   ┌───────────────────────────────────────────────────┐
   │ Statement 1: FROM→WHERE→...→LIMIT  (full pipeline) │
   │ Statement 2: INSERT ...                            │
   │ SAVEPOINT sp1;                                     │
   │ Statement 3: UPDATE ...                            │
   │ ROLLBACK TO SAVEPOINT sp1;   (undo Statement 3)    │
   │ Statement 4: DELETE ...                            │
   └───────────────────────────────────────────────────┘
COMMIT;                         ← makes everything above durable & visible atomically
```

Key ordering facts that matter for correctness:

- **The snapshot timing depends on isolation level, not on where BEGIN sits syntactically.** Under READ COMMITTED (the default), each *statement* inside the block gets a fresh snapshot at the moment it starts. Under REPEATABLE READ / SERIALIZABLE, the snapshot is frozen at the first statement that touches data after `BEGIN`. This is why `BEGIN` alone does not take a snapshot — the snapshot is deferred until the first query runs.

- **DDL is transactional in PostgreSQL.** Unlike MySQL, `CREATE TABLE`, `ALTER TABLE`, `DROP TABLE`, `CREATE INDEX` (except `CONCURRENTLY`) all participate in the transaction and roll back cleanly. This is a major operational advantage: migrations can be atomic.

- **Some statements cannot run inside a transaction block:** `VACUUM`, `CREATE INDEX CONCURRENTLY`, `CREATE DATABASE`, `ALTER SYSTEM`, and `REINDEX CONCURRENTLY`. They will error with "cannot run inside a transaction block."

- **Order of savepoint operations is a stack.** Savepoints nest LIFO. `ROLLBACK TO SAVEPOINT sp1` implicitly destroys any savepoints established *after* sp1, but keeps sp1 itself active so you can roll back to it again.

The mental model: the SQL pipeline (Topics 1–50) describes what one statement computes; transaction syntax describes when a *group* of statements becomes real, and how far back you can rewind if one of them goes wrong.

---

## 4. What Is Transaction Control Language?

A **transaction** is a sequence of one or more SQL statements executed as a single atomic, consistent, isolated, durable unit of work (ACID). **Transaction Control Language (TCL)** is the set of commands that delimit and manage these units: `BEGIN`, `COMMIT`, `ROLLBACK`, `SAVEPOINT`, `ROLLBACK TO SAVEPOINT`, and `RELEASE SAVEPOINT`. Together they let you declare "these statements succeed together or fail together," and, via savepoints, "these sub-steps can be individually rewound without discarding the whole unit."

### 4.1 The full syntax, annotated

```sql
BEGIN [ WORK | TRANSACTION ]
  [ ISOLATION LEVEL { SERIALIZABLE | REPEATABLE READ | READ COMMITTED | READ UNCOMMITTED } ]
  [ READ WRITE | READ ONLY ]
  [ [ NOT ] DEFERRABLE ];
│      │                    │            │
│      │                    │            └── DEFERRABLE: only meaningful for SERIALIZABLE READ ONLY;
│      │                    │                waits for a safe snapshot so it never aborts or blocks
│      │                    └── READ ONLY: rejects any write; lets the planner/engine optimize
│      │                        and enables DEFERRABLE mode
│      └── ISOLATION LEVEL: sets snapshot policy for THIS transaction (see Topic 53).
│          READ UNCOMMITTED is accepted but behaves as READ COMMITTED in PostgreSQL.
└── WORK / TRANSACTION: optional noise words, identical meaning. `BEGIN;` alone is idiomatic.
```

```sql
COMMIT [ WORK | TRANSACTION ] [ AND [ NO ] CHAIN ];
│                              │
│                              └── AND CHAIN: immediately starts a NEW transaction with the
│                                  same characteristics (isolation, read/write mode) as the one
│                                  just committed. AND NO CHAIN (default) just ends the block.
└── Makes every change in the block durable (WAL fsync) and visible (clog marked committed).
```

```sql
ROLLBACK [ WORK | TRANSACTION ] [ AND [ NO ] CHAIN ];
│                                │
│                                └── AND CHAIN: roll back, then immediately begin a fresh
│                                    transaction with the same characteristics.
└── Discards every change in the block. Marks the xid aborted in clog. Cheap in PostgreSQL
    (no physical undo — dead tuples reclaimed later by VACUUM).
```

```sql
SAVEPOINT savepoint_name;
│         │
│         └── An identifier. Re-using an existing name creates a NEW savepoint and hides
│             (but does not destroy) the older one of the same name until released/rolled back to.
└── Opens a subtransaction (sub-xid). Establishes a rewind point inside the current transaction.
```

```sql
ROLLBACK TO [ SAVEPOINT ] savepoint_name;
│            │
│            └── The SAVEPOINT keyword is optional noise: `ROLLBACK TO sp1;` == `ROLLBACK TO SAVEPOINT sp1;`
└── Undoes all statements executed AFTER that savepoint was established, plus destroys any
    savepoints created after it. The savepoint itself REMAINS active — you can roll back to it again.
    Also the canonical way to recover a transaction from the aborted state without discarding it.
```

```sql
RELEASE [ SAVEPOINT ] savepoint_name;
│        │
│        └── SAVEPOINT keyword optional.
└── Destroys the named savepoint (and any created after it) WITHOUT undoing any work.
    The subtransaction's effects are merged up into the parent. You simply lose the ability
    to roll back to that point. Frees the subtransaction resource.
```

### 4.2 The two aliases: `BEGIN` vs `START TRANSACTION`

```sql
START TRANSACTION [ transaction_mode [, ...] ];
-- Identical to BEGIN. This is the SQL-standard spelling; BEGIN is the PostgreSQL idiom.
```

`BEGIN` and `START TRANSACTION` are functionally identical in PostgreSQL. `START TRANSACTION` is the ANSI SQL standard form. (In some other databases — notably MySQL's stored-procedure dialect — `BEGIN` denotes a block of code, not a transaction; there `START TRANSACTION` is required. In plain PostgreSQL SQL they are interchangeable.)

### 4.3 The autocommit default

Every statement you send outside an explicit `BEGIN` runs in its **own implicit transaction that commits automatically** the instant it succeeds. This is *autocommit mode*, and it is on by default in `psql`, in the `pg` driver, and in essentially every client. A bare `UPDATE users SET ... ` with no surrounding `BEGIN` is really:

```sql
BEGIN;  UPDATE users SET ...;  COMMIT;   -- implicit, done for you
```

Autocommit is why single statements are already atomic and durable. You only need explicit TCL when you want *multiple* statements to share one atomic unit.

---

## 5. Why Transaction Syntax Mastery Matters in Production

1. **Money and correctness.** The canonical example is a bank transfer: debit one account, credit another. Without a transaction, a crash between the two updates leaves money destroyed or created. Every financial, inventory, booking, or ledger system depends on `BEGIN … COMMIT` being correct. Get the boundary wrong and you get lost writes, double-charges, or phantom inventory.

2. **The aborted-transaction trap.** Under most drivers, once any statement in a transaction errors, PostgreSQL puts the whole transaction into the **aborted state**: every subsequent statement fails with `current transaction is aborted, commands ignored until end of transaction block` (SQLSTATE `25P02`). Teams that don't understand this ship code that silently swallows the real error and then throws a confusing cascade of "transaction is aborted" errors, masking the root cause. Savepoints are the professional's tool to survive expected per-statement failures without discarding the whole unit.

3. **Idle-in-transaction disasters.** A `BEGIN` that is never committed or rolled back — because an app forgot, or an HTTP request hung mid-transaction — holds locks and pins the xid horizon. This blocks VACUUM from cleaning up dead tuples across the *entire cluster*, causing table bloat, and can block other sessions on row locks indefinitely. `idle_in_transaction_session_timeout` exists precisely because this bug is so common and so damaging.

4. **Migration safety.** Because PostgreSQL DDL is transactional, wrapping a schema migration in `BEGIN … COMMIT` means a failed migration rolls back cleanly to the prior schema — no half-applied migration. Understanding *which* statements break out of the transaction (`CREATE INDEX CONCURRENTLY`, `VACUUM`) is the difference between a safe zero-downtime migration and an outage.

5. **Connection-pool correctness.** Poolers (PgBouncer in transaction mode) hand a physical connection to a client only for the duration of a transaction. If your app leaks a `BEGIN` without a matching `COMMIT`/`ROLLBACK`, you pin a pooled backend and starve the pool. Transaction hygiene is inseparable from pool health.

6. **Performance of COMMIT.** Because COMMIT is the fsync barrier, *how often* you commit determines throughput. Committing once per row in a 1M-row load is ~1M fsyncs; batching into transactions of, say, 5,000 rows can be 50–100× faster. Knowing that COMMIT is where the cost lives is essential for bulk-load and ETL tuning.

---

## 6. Deep Technical Content

### 6.1 The three states of a transaction

At any moment a session's transaction is in exactly one of these states, visible via `psql`'s prompt suffix and programmatically via the `pg_stat_activity.state` column:

| State | `pg_stat_activity.state` | What you can do |
|-------|--------------------------|-----------------|
| **No transaction (idle / autocommit)** | `idle` | Any statement — it becomes its own implicit transaction |
| **In transaction (active block)** | `active` (running) / `idle in transaction` (waiting) | Any statement; COMMIT or ROLLBACK to end |
| **Aborted (failed block)** | `idle in transaction (aborted)` | Only ROLLBACK, ROLLBACK TO SAVEPOINT, or RELEASE — every other statement errors 25P02 |

The transition into **aborted** happens automatically the moment any statement raises an error inside an explicit transaction block. There is no way to "clear" the error and continue as if nothing happened except to roll the whole thing back — *unless* you had set a savepoint before the failing statement, in which case `ROLLBACK TO SAVEPOINT` returns the transaction to the in-transaction (non-aborted) state at that point.

```sql
BEGIN;
INSERT INTO users (id, email) VALUES (1, 'a@x.com');   -- OK
INSERT INTO users (id, email) VALUES (1, 'b@x.com');   -- ERROR: duplicate key (unique violation, 23505)
SELECT * FROM users;   -- ERROR: current transaction is aborted (25P02)
COMMIT;                -- Behaves as ROLLBACK! A COMMIT of an aborted txn rolls back and warns.
```

That last line is a crucial gotcha: **`COMMIT` on an aborted transaction does not commit — it rolls back** and emits `WARNING: there is no transaction in progress` or performs a rollback. Nothing you did is saved.

### 6.2 Autocommit in depth — the implicit transaction

Outside an explicit block, PostgreSQL wraps every statement in its own transaction. This is not the client's doing; the *server* does it. Even a `SELECT` gets an xid-less virtual transaction and a snapshot. The consequence: a single multi-row statement is atomic. `UPDATE accounts SET balance = balance * 1.05;` either updates every matching row or none — a mid-statement crash rolls the whole statement back on recovery.

Autocommit interacts with the **statement vs transaction** distinction for multi-statement strings. If you send `stmt1; stmt2; stmt3;` as a single simple-query message in autocommit mode, libpq wraps the *whole batch* in one implicit transaction unless you explicitly separate them — behavior that differs by protocol path (simple vs extended query). In the `pg` Node driver, each `pool.query()` is one statement and one implicit transaction. To group statements you must use an explicit `BEGIN`/`COMMIT` on the same client connection.

### 6.3 Savepoints and nested (sub)transactions

PostgreSQL does not support true nested `BEGIN … BEGIN` transactions. Sending `BEGIN` when already in a transaction produces `WARNING: there is already a transaction in progress` and does nothing. **Savepoints are how you get nesting.** Each `SAVEPOINT` opens a subtransaction; you can nest them arbitrarily deep.

```sql
BEGIN;                          -- top-level transaction T
  INSERT INTO orders ...;       -- part of T
  SAVEPOINT after_order;        -- subtransaction S1 opens
    INSERT INTO order_items ...;-- part of S1
    SAVEPOINT after_items;      -- subtransaction S2 opens (nested in S1)
      UPDATE inventory ...;     -- part of S2
    ROLLBACK TO after_items;    -- undo the UPDATE; S2's slot reset, S1 & T intact
  RELEASE after_order;          -- merge S1 (and the order_items insert) into T
COMMIT;                         -- everything not rolled back becomes durable
```

Semantics to internalize:

- **`ROLLBACK TO SAVEPOINT sp` keeps `sp`.** After rolling back to it, `sp` is still an open savepoint — you may execute more statements and roll back to it again. Any savepoints created *after* `sp` are destroyed.
- **`RELEASE sp` merges upward.** The subtransaction's changes are not undone; they become part of the enclosing (sub)transaction. You just lose the ability to rewind to that point.
- **Rolling back to an outer savepoint cascades.** If S1 encloses S2 and you `ROLLBACK TO` S1's savepoint, S2 is destroyed along with S1's later work.
- **Only COMMIT of the top-level transaction makes anything durable.** Releasing a savepoint does not write to disk durably by itself; it just moves work up the stack. The final `COMMIT` is still the one fsync barrier.

### 6.4 Savepoints as error recovery — the "try/catch" of SQL

The most important production use of savepoints is surviving an *expected* statement error without losing the whole transaction. Because any error aborts the transaction, you wrap the risky statement:

```sql
BEGIN;
  INSERT INTO audit_logs (event) VALUES ('batch start');
  SAVEPOINT sp_try;
  INSERT INTO users (id, email) VALUES (42, 'dup@x.com');   -- might violate unique constraint
  -- If it succeeded, we RELEASE and move on:
  RELEASE SAVEPOINT sp_try;
  -- If it FAILED, the app instead issues:
  --   ROLLBACK TO SAVEPOINT sp_try;   -- transaction is healthy again, error contained
  INSERT INTO audit_logs (event) VALUES ('batch end');
COMMIT;
```

This is exactly what PL/pgSQL's `BEGIN … EXCEPTION WHEN … END` block does under the hood: **every `EXCEPTION` block is an implicit savepoint.** That is also why exception blocks in tight loops are not free — each one is a subtransaction with sub-xid overhead.

### 6.5 The cost of subtransactions — the sub-xid gotcha

Subtransactions are not free at scale. Two well-known performance cliffs:

1. **The 64-subtransaction cache.** Each backend caches up to 64 sub-xids in shared memory (`PGPROC`). Beyond 64 open subtransactions in a single transaction, lookups spill to the `pg_subtrans` SLRU on disk, and *other* backends taking snapshots may have to consult it too. A transaction that opens thousands of savepoints (or a loop with an EXCEPTION block running millions of iterations) can trigger the infamous "subtrans SLRU contention," degrading the *whole cluster*.

2. **`SubtransSLRU` / `SubtransControlLock` contention.** Under heavy subtransaction use combined with long-running transactions, backends contend on the subtrans lookup, showing up as `LWLock:SubtransSLRU` waits in `pg_stat_activity`. This is a classic "it was fine until it wasn't" scaling surprise.

Practical rule: use savepoints deliberately for error boundaries, not as a casual loop construct that opens millions of them.

### 6.6 READ ONLY, READ WRITE, and DEFERRABLE

```sql
BEGIN READ ONLY;                       -- any INSERT/UPDATE/DELETE/DDL errors out
BEGIN ISOLATION LEVEL SERIALIZABLE READ ONLY DEFERRABLE;
```

- **`READ ONLY`** rejects writes. Useful for reporting/replica queries and as a guardrail. It does not by itself change snapshot semantics but signals intent and enables `DEFERRABLE`.
- **`DEFERRABLE`** (only meaningful for `SERIALIZABLE READ ONLY`) makes the transaction wait until it can acquire a snapshot guaranteed free of serialization anomalies, so it will **never** be rolled back with a serialization failure and never block concurrent writers. Ideal for long analytical/backup queries that must see a consistent snapshot without risking `40001` retries.

### 6.7 Transaction characteristics and `SET TRANSACTION`

You can set characteristics after `BEGIN` but only *before the first query*:

```sql
BEGIN;
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;   -- must precede any data access
SET TRANSACTION READ ONLY;
SELECT ...;
```

Attempting `SET TRANSACTION ISOLATION LEVEL` after the transaction's first statement raises `SET TRANSACTION ISOLATION LEVEL must be called before any query`. To change the default for a whole session use `SET SESSION CHARACTERISTICS AS TRANSACTION ...`.

### 6.8 COMMIT/ROLLBACK AND CHAIN

`AND CHAIN` ends the current transaction and *immediately* starts a new one inheriting the same isolation level, read/write mode, and deferrable setting — atomically, with no window of autocommit in between.

```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;
  ... work batch 1 ...
COMMIT AND CHAIN;      -- durable, and a fresh REPEATABLE READ txn is now open
  ... work batch 2 ...
COMMIT;                -- ordinary end
```

Without `AND CHAIN`, there is a brief moment in autocommit between the COMMIT and the next BEGIN, during which another session's statement could interleave and you'd lose the inherited characteristics.

### 6.9 Implicit transaction control from errors and disconnects

- **Server-detected fatal errors** (deadlock `40P01`, serialization failure `40001`, out of memory) abort the transaction; the client must ROLLBACK and, for `40001`/`40P01`, typically *retry* the whole transaction.
- **Client disconnect mid-transaction** causes the server to ROLLBACK automatically. Uncommitted work is lost — which is the correct, safe outcome.
- **`ON_ERROR_ROLLBACK`** in `psql` (a psql feature, not server behavior) transparently wraps each interactive statement in an implicit savepoint so a typo doesn't abort your whole session's transaction. Set `\set ON_ERROR_ROLLBACK interactive`.

### 6.10 Two-phase commit (PREPARE TRANSACTION)

For distributed transactions across systems, PostgreSQL exposes explicit 2PC:

```sql
BEGIN;
  ... work ...
PREPARE TRANSACTION 'txn_order_4711';   -- persists the transaction, ready to commit, releasing the session
-- later, possibly from another session, after coordinating with other resource managers:
COMMIT PREPARED 'txn_order_4711';        -- or  ROLLBACK PREPARED 'txn_order_4711';
```

Prepared transactions survive server restarts and hold their locks until resolved. **Orphaned prepared transactions are a severe hazard** — they pin the xid horizon exactly like idle-in-transaction, blocking VACUUM forever. Requires `max_prepared_transactions > 0`. Use only with a real transaction coordinator (XA); never leave them dangling.

---

## 7. EXPLAIN — Transactions in the Plan

Transactions are not planned like queries — `EXPLAIN BEGIN;` is a syntax error, and TCL statements have no query plan. What you *observe* about transactions is their effect on **locks, visibility, WAL, and timing**, using `EXPLAIN (ANALYZE, BUFFERS)` on the statements *inside* the block plus system views. Here is how to read a transaction's footprint.

### 7.1 Observing the WAL cost of COMMIT

`EXPLAIN (ANALYZE, WAL)` reports WAL generated by a statement. Wrap a write to see it:

```sql
BEGIN;
EXPLAIN (ANALYZE, BUFFERS, WAL)
UPDATE orders SET status = 'shipped' WHERE status = 'packed';
```

```
Update on orders  (cost=0.00..2310.00 rows=0 width=0)
                  (actual time=41.2..41.2 rows=0 loops=1)
  ->  Seq Scan on orders  (cost=0.00..2310.00 rows=12000 width=6)
                          (actual time=0.03..18.7 rows=12000 loops=1)
        Filter: (status = 'packed'::text)
        Rows Removed by Filter: 88000
  Buffers: shared hit=1730 read=310 dirtied=190 written=12
  WAL: records=12043 fpi=95 bytes=1892344
Planning Time: 0.14 ms
Execution Time: 42.0 ms
```

Reading it:
- `WAL: records=12043` — one WAL record per updated tuple (plus index updates). This WAL must be fsync'd at COMMIT — that fsync, not the UPDATE above, is where commit latency lives.
- `fpi=95` — *full-page images*: the first modification of a page after a checkpoint logs the entire page for crash safety. FPIs dominate WAL volume right after a checkpoint. Committing many small transactions each just after a checkpoint amplifies WAL.
- `dirtied=190` — pages made dirty; they'll be flushed by the background writer/checkpointer. The `COMMIT` itself does not flush these data pages, only the WAL.

The `EXPLAIN` output shows the *work*; the COMMIT that follows is the durability barrier that the plan does not show. To measure COMMIT latency itself, use `\timing` around the COMMIT or `pg_stat_database` / `pg_stat_wal`.

### 7.2 Observing locks a transaction holds

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 7;   -- takes a row lock
-- in another session:
SELECT locktype, relation::regclass, mode, granted
FROM pg_locks WHERE pid = <this backend pid>;
```

```
 locktype |  relation  |       mode       | granted
----------+------------+------------------+---------
 relation | accounts   | RowExclusiveLock | t
 tuple    |            | ExclusiveLock    | t
 transactionid |        | ExclusiveLock    | t     ← the xid lock, held till COMMIT/ROLLBACK
```

The `transactionid ExclusiveLock` row is the transaction's own xid lock — other transactions that need to wait on this one (e.g., to update the same row) block on *this* lock and are released the instant you COMMIT or ROLLBACK. This is the physical mechanism behind "COMMIT releases locks."

### 7.3 Observing subtransactions

`EXPLAIN` shows nothing about savepoints, but `txid_current_if_assigned()` and `pg_current_xact_id()` reveal xid assignment, and a heavily-subtransaction'd session shows up as `LWLock:SubtransSLRU` waits:

```sql
SELECT wait_event_type, wait_event, state, xact_start, now() - xact_start AS age
FROM pg_stat_activity WHERE state <> 'idle';
```

```
 wait_event_type |  wait_event   |        state        | age
-----------------+---------------+---------------------+----------
 LWLock          | SubtransSLRU  | active              | 00:04:12   ← subtrans contention + long txn
```

A non-null `xact_start` far in the past on an `idle in transaction` row is the smoking gun for the idle-in-transaction bloat problem.

### 7.4 What to look for

| Symptom | Where you see it | Likely cause |
|---------|------------------|--------------|
| High commit latency | `\timing` on COMMIT, `pg_stat_wal` | fsync barrier; consider batching or `synchronous_commit` |
| WAL volume spikes | `EXPLAIN (ANALYZE, WAL)` `fpi=` high | full-page images right after checkpoint |
| `idle in transaction` with old `xact_start` | `pg_stat_activity` | leaked BEGIN — blocks VACUUM, holds locks |
| `LWLock:SubtransSLRU` waits | `pg_stat_activity` | too many savepoints / EXCEPTION blocks per txn |
| Blocked sessions on `transactionid` | `pg_locks` `granted=f` | waiting on another txn's row lock until it commits |

---

## 8. Query Examples

### Example 1 — Basic: An atomic money transfer

```sql
-- The textbook transaction: two writes that must both happen or neither.
BEGIN;                                                     -- open the unit of work
  UPDATE accounts SET balance = balance - 100             -- debit sender
    WHERE id = 1;
  UPDATE accounts SET balance = balance + 100             -- credit receiver
    WHERE id = 2;
COMMIT;                                                    -- both become durable atomically
-- If the session crashed between the two UPDATEs, recovery rolls BOTH back:
-- no money is created or destroyed. That is atomicity.
```

### Example 2 — Intermediate: Savepoint-based partial rollback

```sql
-- Insert an order, then attempt to reserve inventory for each line item.
-- If one reservation fails (out of stock), skip that line but keep the order.
BEGIN;
  INSERT INTO orders (id, customer_id, status)
    VALUES (5001, 42, 'pending');

  SAVEPOINT line_1;
  UPDATE products SET stock = stock - 2
    WHERE id = 100 AND stock >= 2;
  -- app checks rowcount: if 0 rows updated, the item was out of stock:
  ROLLBACK TO SAVEPOINT line_1;      -- undo just this line's attempt, order still alive
  RELEASE SAVEPOINT line_1;          -- (if it had succeeded, we'd RELEASE instead)

  SAVEPOINT line_2;
  UPDATE products SET stock = stock - 1
    WHERE id = 200 AND stock >= 1;   -- succeeds
  RELEASE SAVEPOINT line_2;          -- keep it, merge into the parent transaction

  INSERT INTO order_items (order_id, product_id, quantity)
    VALUES (5001, 200, 1);
COMMIT;                              -- order + the one successful reservation persist together
```

### Example 3 — Production Grade: Idempotent, retry-safe batch settlement

**Context.** A nightly job settles `payments` for completed `orders`. Tables: `orders` (50M rows), `payments` (48M rows, index on `order_id`), `audit_logs` (append-only, 400M rows). The job must be **atomic per batch**, **idempotent** (safe to re-run after a crash), and **survive per-row constraint failures** without aborting the whole batch. Expectation: a 5,000-row batch commits in well under a second; the single COMMIT is the durability barrier.

```sql
BEGIN ISOLATION LEVEL READ COMMITTED;          -- default; each statement sees latest committed data

  -- Grab a batch of unsettled orders, locking them so concurrent workers don't double-process.
  -- SKIP LOCKED lets parallel workers each take disjoint batches (Topic on locking).
  CREATE TEMP TABLE batch ON COMMIT DROP AS      -- temp table is transaction-scoped
  SELECT id, customer_id, total_amount
  FROM orders
  WHERE status = 'completed' AND settled_at IS NULL
  ORDER BY id
  FOR UPDATE SKIP LOCKED
  LIMIT 5000;

  SAVEPOINT before_payments;                     -- error boundary for the bulk insert
  -- ON CONFLICT makes the insert idempotent: a re-run after a crash won't double-insert.
  INSERT INTO payments (order_id, amount, settled_at)
  SELECT id, total_amount, now() FROM batch
  ON CONFLICT (order_id) DO NOTHING;             -- unique(order_id) enforces idempotency
  RELEASE SAVEPOINT before_payments;

  UPDATE orders o
  SET status = 'settled', settled_at = now()
  FROM batch b
  WHERE o.id = b.id;

  INSERT INTO audit_logs (event, payload)
  SELECT 'settlement', jsonb_build_object('order_id', id, 'amount', total_amount)
  FROM batch;

COMMIT;   -- one fsync makes the whole batch durable; temp table auto-dropped
```

```sql
-- EXPLAIN of the locking SELECT that seeds the batch:
EXPLAIN (ANALYZE, BUFFERS)
SELECT id FROM orders
WHERE status = 'completed' AND settled_at IS NULL
ORDER BY id FOR UPDATE SKIP LOCKED LIMIT 5000;
```

```
Limit  (cost=0.56..842.10 rows=5000 width=14) (actual time=0.09..7.8 rows=5000 loops=1)
  ->  LockRows  (cost=0.56..71204.0 rows=422000 width=14)
                (actual time=0.09..7.4 rows=5000 loops=1)
        ->  Index Scan using orders_settle_idx on orders
              (cost=0.56..66984.0 rows=422000 width=14)
              (actual time=0.05..5.9 rows=5000 loops=1)
              Index Cond: (status = 'completed')
              Filter: (settled_at IS NULL)
  Buffers: shared hit=812 read=41
Planning Time: 0.20 ms
Execution Time: 8.3 ms
```

Reading it: the `LockRows` node applies `FOR UPDATE SKIP LOCKED`, quietly skipping any of the 422,000 candidate rows already locked by a sibling worker, and stops as soon as it has 5,000 lockable rows (`LIMIT`). A partial index `orders_settle_idx ON orders(id) WHERE settled_at IS NULL` keeps the scan tight even as the table grows to 50M rows. The COMMIT that follows is the only fsync; batching 5,000 rows into it amortizes the durability cost ~5,000×.

---

## 9. Wrong → Right Patterns

### Wrong 1: Two writes with no transaction — the non-atomic transfer

```sql
-- WRONG: each statement autocommits independently
UPDATE accounts SET balance = balance - 100 WHERE id = 1;   -- commits immediately
-- ← if the app crashes HERE, $100 has vanished; the credit never happens
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
```

**Why it's wrong at the execution level:** in autocommit mode each `UPDATE` is its own implicit `BEGIN … COMMIT`. The first debit is durable the instant it returns. A crash, exception, or connection drop before the second statement leaves the system in a permanently inconsistent state — money destroyed. There is no atomic link between the two statements.

```sql
-- RIGHT: one atomic unit
BEGIN;
  UPDATE accounts SET balance = balance - 100 WHERE id = 1;
  UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;   -- both durable together, or neither on crash
```

### Wrong 2: Ignoring the aborted state — swallowing the real error

```sql
-- WRONG: continuing to issue statements after an error inside a transaction
BEGIN;
  INSERT INTO users (id, email) VALUES (1, 'a@x.com');   -- OK
  INSERT INTO users (id, email) VALUES (1, 'dup@x.com'); -- ERROR 23505 unique_violation
  INSERT INTO audit_logs (event) VALUES ('user created');-- ERROR 25P02: txn aborted
COMMIT;                                                  -- silently ROLLS BACK everything
```

**Why it's wrong:** the second insert aborts the transaction. Every statement after it fails with `25P02 (current transaction is aborted)`, and the final `COMMIT` on an aborted transaction *rolls back* rather than commits — so even the first, successful insert is lost. Naive code that catches only the last error reports "transaction is aborted" and never surfaces the real `23505` duplicate-key cause.

```sql
-- RIGHT: guard the risky statement with a savepoint so the txn survives
BEGIN;
  INSERT INTO users (id, email) VALUES (1, 'a@x.com');
  SAVEPOINT sp;
  -- attempt; if it errors, app issues ROLLBACK TO SAVEPOINT sp instead of RELEASE
  INSERT INTO users (id, email) VALUES (1, 'dup@x.com');
  ROLLBACK TO SAVEPOINT sp;        -- error contained, transaction healthy again
  INSERT INTO audit_logs (event) VALUES ('user created');  -- now succeeds
COMMIT;                            -- the first insert + the audit log persist
```

Even better for a known conflict: `INSERT ... ON CONFLICT DO NOTHING`, which never raises and never aborts the transaction.

### Wrong 3: The leaked BEGIN (idle-in-transaction)

```sql
-- WRONG (in app code): open a transaction, do a slow external call, forget to close it
BEGIN;
  SELECT * FROM orders WHERE id = 4711 FOR UPDATE;   -- locks the row
  -- app now makes a 30-second HTTP call to a payment gateway... and maybe throws
  -- no COMMIT / ROLLBACK on the error path
```

**Why it's wrong:** the row lock is held for the entire external call. Worse, the open transaction pins the cluster-wide xid horizon, so VACUUM cannot reclaim dead tuples anywhere, causing bloat. Other sessions needing that row block on the `transactionid` lock indefinitely.

```sql
-- RIGHT: keep the transaction short; do external work OUTSIDE it
--   1. read what you need (short txn or autocommit)
--   2. do the slow HTTP call with NO transaction open
--   3. open a fresh, short transaction to write the result
BEGIN;
  UPDATE orders SET status = 'paid', paid_at = now()
  WHERE id = 4711 AND status = 'pending';   -- optimistic guard on status
COMMIT;
-- Plus a server-side seatbelt:
SET idle_in_transaction_session_timeout = '15s';  -- auto-abort leaked transactions
```

### Wrong 4: Assuming ROLLBACK TO SAVEPOINT ends the transaction

```sql
-- WRONG mental model: treating ROLLBACK TO as if it were ROLLBACK
BEGIN;
  INSERT INTO orders ... ;
  SAVEPOINT sp;
  UPDATE inventory ... ;      -- goes wrong
  ROLLBACK TO SAVEPOINT sp;   -- developer assumes the transaction is now DONE and walks away
-- ← transaction is STILL OPEN. The orders INSERT is uncommitted, locks still held,
--   connection still pinned. This becomes an idle-in-transaction leak.
```

**Why it's wrong:** `ROLLBACK TO SAVEPOINT` only rewinds *within* the transaction; it does not end it. The transaction remains open and must still be finished with `COMMIT` or `ROLLBACK`.

```sql
-- RIGHT: always finish the transaction explicitly
BEGIN;
  INSERT INTO orders ... ;
  SAVEPOINT sp;
  UPDATE inventory ... ;
  ROLLBACK TO SAVEPOINT sp;   -- undo just the update
COMMIT;                       -- <-- actually end the transaction (keeps the orders insert)
```

### Wrong 5: Per-row commits in a bulk load

```sql
-- WRONG: 1,000,000 implicit transactions = 1,000,000 fsyncs
-- (this loop in app code, each iteration autocommitting)
INSERT INTO audit_logs (event) VALUES ('row 1');   -- COMMIT (fsync)
INSERT INTO audit_logs (event) VALUES ('row 2');   -- COMMIT (fsync)
-- ... a million times. Throughput collapses to disk-fsync rate.
```

**Why it's wrong:** every autocommitted statement forces a WAL fsync. On spinning disk that's ~100–200/sec; even on SSD the syscall overhead dominates. The load takes hours.

```sql
-- RIGHT: batch into transactions so one fsync covers thousands of rows
BEGIN;
  -- ... 5,000 INSERTs (or better, a single multi-row INSERT / COPY) ...
COMMIT;   -- one fsync per 5,000 rows → 50–100× faster
-- Best of all for pure loads: COPY, which is one statement, one transaction, minimal WAL.
```

---

## 10. Performance Profile

### 10.1 Where the cost lives

A transaction's cost is dominated by **COMMIT**, not by BEGIN or the statements themselves:

| Operation | Cost | Why |
|-----------|------|-----|
| `BEGIN` | ~free | No snapshot yet, no xid yet; just a state flag |
| First write | assigns xid | The xid is allocated lazily on first write, not at BEGIN |
| Each write | WAL record buffered | Cheap; goes to WAL buffers in memory |
| `COMMIT` (write txn) | **1 fsync** | The durability barrier — flushes WAL to disk |
| `COMMIT` (read-only txn) | ~free | No xid assigned → nothing to flush, no clog entry |
| `ROLLBACK` | cheap | Marks xid aborted; no physical undo (MVCC) |
| `SAVEPOINT` | cheap-ish | Allocates a sub-xid; cheap until you exceed 64 |
| `ROLLBACK TO` / `RELEASE` | cheap | Sub-xid bookkeeping |

**Key insight:** a read-only transaction never assigns an xid and its COMMIT is essentially free. Only *writing* transactions pay the fsync. This is why wrapping read-only reporting queries in transactions costs nothing.

### 10.2 Commit throughput and batching

Because COMMIT = one fsync, commit *frequency* sets your write ceiling:

| Pattern | Fsyncs for 1M rows | Relative speed |
|---------|--------------------|----------------|
| Autocommit per row | 1,000,000 | 1× (baseline, slowest) |
| Batches of 1,000 | 1,000 | ~50–100× |
| Batches of 10,000 | 100 | ~100–200× |
| Single `COPY` | ~1 | fastest for pure load |

Diminishing returns past ~5,000–10,000 rows per transaction, and larger batches hold locks longer and generate bigger rollback segments (dead tuples on abort). The sweet spot balances fsync amortization against lock-hold time.

### 10.3 `synchronous_commit` — trading durability for speed

```sql
SET synchronous_commit = off;   -- session-level; COMMIT returns before WAL fsync completes
```

With `synchronous_commit = off`, COMMIT acknowledges *before* the WAL is durably on disk (flushed within `wal_writer_delay`, default 200ms). A crash can lose the last fraction of a second of committed transactions — but **not** corrupt the database (torn writes are still prevented). For high-volume, loss-tolerant data (analytics events, non-critical audit logs) this can multiply throughput. **Never** for money.

### 10.4 Scaling at 1M / 10M / 100M rows

- **1M-row table:** transaction mechanics are irrelevant to correctness; batch COMMITs for load speed.
- **10M-row table:** long transactions start to hurt — an open transaction holding back the xid horizon prevents VACUUM, and dead tuples accumulate. Keep transactions short.
- **100M-row table:** transaction ID consumption matters. Bulk operations that each consume an xid must be balanced against **wraparound autovacuum**. A single transaction that runs for hours on a busy cluster can delay the xid horizon enough to trigger emergency `autovacuum (to prevent wraparound)`, which is disruptive. Break huge operations into batched, separately-committed chunks.

### 10.5 Subtransaction scaling cliff

- Under 64 open savepoints per transaction: cached in shared memory, cheap.
- Over 64: spills to `pg_subtrans` SLRU; other backends taking snapshots may pay lookup cost.
- Combined with a long-running transaction on a busy cluster: `LWLock:SubtransSLRU` contention can throttle the *entire* system. Avoid loops that open millions of savepoints or PL/pgSQL EXCEPTION blocks executed millions of times.

### 10.6 Lock-hold duration

Every lock a transaction takes is held until COMMIT/ROLLBACK. Long transactions = long lock holds = more blocking and deadlock risk. The performance rule that ties it together: **keep transactions as short as correctness allows, batch writes to amortize fsync, and never do slow external I/O with a transaction open.**

---

## 11. Node.js Integration

The critical rule with `pg`: **a transaction lives on a single physical connection.** You cannot `BEGIN` on the pool and `COMMIT` on the pool — the pool may hand you a different backend. You must check out one client, run the whole transaction on it, and release it.

### 11.1 The canonical transaction helper

```javascript
import pg from 'pg';
const { Pool } = pg;
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

async function transfer(fromId, toId, amount) {
  const client = await pool.connect();          // check out ONE physical connection
  try {
    await client.query('BEGIN');
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
    await client.query('ROLLBACK');             // undo everything on any failure
    throw err;                                   // re-throw so caller sees the real error
  } finally {
    client.release();                            // ALWAYS return the connection to the pool
  }
}
```

The `finally { client.release() }` is non-negotiable: forget it and you leak a pooled connection on every error, exhausting the pool. This is the single most common `pg` transaction bug.

### 11.2 Savepoints in Node — surviving a per-row failure

```javascript
async function insertUsersTolerant(users) {
  const client = await pool.connect();
  const inserted = [];
  try {
    await client.query('BEGIN');
    for (const u of users) {
      await client.query('SAVEPOINT sp');        // error boundary per row
      try {
        await client.query(
          'INSERT INTO users (id, email) VALUES ($1, $2)',
          [u.id, u.email]
        );
        await client.query('RELEASE SAVEPOINT sp');   // keep this row
        inserted.push(u.id);
      } catch (err) {
        await client.query('ROLLBACK TO SAVEPOINT sp'); // skip just this row, txn stays healthy
        // log & continue; without the savepoint, one dup would abort the WHOLE batch
      }
    }
    await client.query('COMMIT');
  } catch (err) {
    await client.query('ROLLBACK');
    throw err;
  } finally {
    client.release();
  }
  return inserted;
}
```

Without the savepoint, the first duplicate-key error would put the transaction in the aborted state and every subsequent statement — including the COMMIT — would fail. The savepoint is Node's `try/catch` reaching into SQL.

### 11.3 Retrying serialization / deadlock failures

```javascript
// SERIALIZABLE and deadlock failures are RETRYABLE by design — wrap and retry.
async function runSerializable(fn, maxRetries = 5) {
  for (let attempt = 1; ; attempt++) {
    const client = await pool.connect();
    try {
      await client.query('BEGIN ISOLATION LEVEL SERIALIZABLE');
      const result = await fn(client);
      await client.query('COMMIT');
      return result;
    } catch (err) {
      await client.query('ROLLBACK');
      // 40001 = serialization_failure, 40P01 = deadlock_detected
      if ((err.code === '40001' || err.code === '40P01') && attempt < maxRetries) {
        await new Promise(r => setTimeout(r, 2 ** attempt * 10));  // backoff
        continue;                                                   // retry the whole txn
      }
      throw err;
    } finally {
      client.release();
    }
  }
}
```

**Do ORMs handle this?** They give you a `transaction()` wrapper that does the connect/BEGIN/COMMIT/ROLLBACK/release dance for you (next section), but **retry-on-40001 is on you** unless you add it — none retry serialization failures automatically by default. Getting connection release and retry right is exactly where hand-rolled `pg` code goes wrong, which is the main argument for using the ORM's transaction helper.

---

## 12. ORM Comparison

The question that separates the ORMs here is not "can it BEGIN/COMMIT" — they all can — but four deeper ones: **(1)** does it pin one physical connection for the whole transaction; **(2)** does it expose `SAVEPOINT` / `ROLLBACK TO` (or nested transactions built on them); **(3)** can you set the isolation level / `READ ONLY` / `DEFERRABLE`; and **(4)** does it retry `40001`/`40P01` automatically. The last answer is almost always "no."

### Prisma

**Can Prisma manage transactions?** — Yes, two ways: the array form (`$transaction([...])`) and the interactive callback form (`$transaction(async (tx) => {...})`).

```typescript
import { PrismaClient } from '@prisma/client';
const prisma = new PrismaClient();

// Interactive transaction — Prisma pins ONE connection, issues BEGIN,
// runs the callback, then COMMIT (or ROLLBACK if the callback throws).
await prisma.$transaction(async (tx) => {
  await tx.order.update({
    where: { id: 5001 },
    data: { status: 'settled', settledAt: new Date() },
  });
  await tx.payment.create({
    data: { orderId: 5001, amount: 199.0, settledAt: new Date() },
  });
  // throwing anywhere here triggers ROLLBACK automatically
}, {
  isolationLevel: 'Serializable',   // maps to BEGIN ISOLATION LEVEL SERIALIZABLE
  timeout: 5000,                    // ms the interactive txn may stay open
  maxWait: 2000,                    // ms to wait for a pooled connection
});
```

**Where it breaks**: Prisma has **no savepoint API**. There is no `tx.savepoint()` / `rollbackTo()` — you cannot survive an expected per-statement error the way raw `pg` can; any throw rolls back the *entire* interactive transaction. If you need partial rollback you drop to `$executeRawUnsafe('SAVEPOINT sp')` / `ROLLBACK TO sp`. Prisma also does **not** retry serialization failures — you must wrap `$transaction` in your own retry loop keyed on `err.code === '40001'`. The `timeout` is a Prisma-side guard, not the server's `idle_in_transaction_session_timeout`.

**Verdict**: Clean for straight-line all-or-nothing units and isolation levels. For savepoint-based error recovery or `40001` retries, you add them yourself.

---

### Drizzle ORM

**Can Drizzle manage transactions?** — Yes, and it is the one that exposes savepoints as **nested `tx.transaction()`** (compiled to `SAVEPOINT` / `ROLLBACK TO`).

```typescript
import { db } from './db';
import { orders, payments, auditLogs } from './schema';
import { eq } from 'drizzle-orm';

await db.transaction(async (tx) => {
  await tx.update(orders)
    .set({ status: 'settled' })
    .where(eq(orders.id, 5001));

  // Nested transaction == SAVEPOINT. Rolling back the inner one
  // does NOT abort the outer — this is real partial rollback.
  try {
    await tx.transaction(async (tx2) => {
      await tx2.insert(payments).values({ orderId: 5001, amount: 199.0 });
      throw new Error('reverse just this payment');   // ROLLBACK TO savepoint
    });
  } catch {
    // outer transaction is still healthy; the order update survives
  }

  await tx.insert(auditLogs).values({ event: 'settlement' });
}, {
  isolationLevel: 'serializable',
  accessMode: 'read write',       // or 'read only'
  deferrable: false,
});

// tx.rollback() inside a callback throws a special signal to force ROLLBACK
```

**Where it breaks**: The nested-transaction-as-savepoint model is exactly the right abstraction, but Drizzle still does **not** auto-retry `40001`/`40P01` — wrap `db.transaction` in a retry loop. `deferrable`/`accessMode` are only meaningful in the combinations PostgreSQL allows (`DEFERRABLE` needs `SERIALIZABLE READ ONLY`).

**Verdict**: The best transaction story of the group — real nested savepoints, typed isolation/access-mode options, SQL-transparent. Add your own retry wrapper.

---

### Sequelize

**Can Sequelize manage transactions?** — Yes, managed (`sequelize.transaction(cb)`) and unmanaged (`await sequelize.transaction()` + manual `commit()`/`rollback()`).

```javascript
const { Transaction } = require('sequelize');

// Managed: auto-COMMIT on resolve, auto-ROLLBACK on throw.
await sequelize.transaction(
  { isolationLevel: Transaction.ISOLATION_LEVELS.SERIALIZABLE },
  async (t) => {
    await Order.update(
      { status: 'settled' },
      { where: { id: 5001 }, transaction: t }   // ← MUST thread `t` into every call
    );
    await Payment.create(
      { orderId: 5001, amount: 199.0 },
      { transaction: t }
    );
  }
);

// Savepoints via nested managed transactions (CLS makes `t` implicit if enabled)
await sequelize.transaction(async (t) => {
  await Order.create({ id: 5002 }, { transaction: t });
  try {
    await sequelize.transaction({ transaction: t }, async (inner) => {
      await Payment.create({ orderId: 5002, amount: -1 }, { transaction: inner });
      throw new Error('bad payment');   // rolls back to the SAVEPOINT only
    });
  } catch { /* outer survives */ }
});
```

**Where it breaks**: The infamous footgun is **forgetting to pass `{ transaction: t }`** to a query — it then runs on a *different* pooled connection, outside your transaction, and neither commits nor rolls back with it. (Continuation-Local Storage / `cls-hooked` can make `t` implicit, but it is off by default and easy to misconfigure.) Nested `transaction` calls become savepoints automatically. No automatic `40001` retry.

**Verdict**: Powerful but ceremony-heavy; the "thread `t` everywhere" rule is where bugs live. Turn on CLS or be religious about passing the transaction.

---

### TypeORM

**Can TypeORM manage transactions?** — Yes, via `dataSource.transaction(cb)`, the `@Transaction` decorator (deprecated), or a manually managed `QueryRunner`.

```typescript
import { DataSource } from 'typeorm';

// Callback form — pins a connection, BEGIN/COMMIT/ROLLBACK handled for you.
await dataSource.transaction('SERIALIZABLE', async (manager) => {
  await manager.update(Order, { id: 5001 }, { status: 'settled' });
  await manager.insert(Payment, { orderId: 5001, amount: 199.0 });
});

// QueryRunner form — full manual control, exposes SAVEPOINT directly.
const qr = dataSource.createQueryRunner();
await qr.connect();
await qr.startTransaction('READ COMMITTED');
try {
  await qr.manager.insert(Order, { id: 5002 });
  await qr.startTransaction();          // nested → emits SAVEPOINT
  try {
    await qr.manager.insert(Payment, { orderId: 5002, amount: -1 });
    await qr.rollbackTransaction();     // ROLLBACK TO SAVEPOINT
  } catch {
    await qr.rollbackTransaction();
  }
  await qr.commitTransaction();
} catch (err) {
  await qr.rollbackTransaction();
  throw err;
} finally {
  await qr.release();                   // MUST release the QueryRunner (like client.release())
}
```

**Where it breaks**: With `QueryRunner` you must `release()` it in a `finally` — the exact same connection-leak trap as raw `pg`'s `client.release()`. Nested `startTransaction` produces savepoints, but you must match every `start` with a `commit`/`rollback`. Isolation level is a string arg; no `DEFERRABLE` sugar. No automatic retry.

**Verdict**: The callback form is safe and terse; the QueryRunner form gives real savepoint control at the cost of manual lifecycle management. Reach for QueryRunner only when you need partial rollback.

---

### Knex.js

**Can Knex manage transactions?** — Yes, and it is the closest to raw SQL, with first-class `trx.savepoint`-style nesting.

```javascript
const knex = require('knex')({ client: 'pg', connection: process.env.DATABASE_URL });

// Callback form: auto-commit on resolve, auto-rollback on throw.
await knex.transaction(async (trx) => {
  await trx('orders').where({ id: 5001 }).update({ status: 'settled' });
  await trx('payments').insert({ order_id: 5001, amount: 199.0 });

  // Nested transaction == SAVEPOINT
  await trx.transaction(async (trx2) => {
    await trx2('audit_logs').insert({ event: 'inner step' });
    // throwing here rolls back to the savepoint, not the whole txn
  }).catch(() => { /* outer continues */ });
}, { isolationLevel: 'serializable', readOnly: false });

// Explicit isolation must be set before the first query:
await knex.transaction(async (trx) => {
  await trx.raw('SET TRANSACTION ISOLATION LEVEL REPEATABLE READ');
  // ...
});
```

**Where it breaks**: Knex nested transactions map to savepoints, but a subtle trap is that a rejected inner transaction promise, if unhandled, propagates and rolls back the *outer* one too — you must `.catch()` the inner promise to contain it (shown above). No automatic `40001` retry. The `isolationLevel` option is relatively recent; older code sets it with a raw `SET TRANSACTION`.

**Verdict**: The most SQL-transparent of the ORMs — `.transaction()` and nested savepoints behave predictably and read like the underlying TCL. Add a retry loop for serializable workloads.

---

### ORM Transaction Summary Table

| ORM | Transaction API | Savepoints / nesting | Isolation level | READ ONLY / DEFERRABLE | Auto-retry 40001 | Main footgun |
|-----|-----------------|----------------------|-----------------|------------------------|------------------|--------------|
| Prisma | `$transaction([])` / callback | No native API (raw `SAVEPOINT`) | `isolationLevel` option | No | No | Whole-txn rollback only; no partial recovery |
| Drizzle | `db.transaction(cb)` | Yes — nested `tx.transaction()` | `isolationLevel` option | `accessMode` / `deferrable` | No | Retry loop is on you |
| Sequelize | managed / unmanaged | Yes — nested `transaction` | `isolationLevel` option | via raw | No | Forgetting `{ transaction: t }` |
| TypeORM | `transaction(cb)` / QueryRunner | Yes — nested `startTransaction` | string arg | via raw | No | `QueryRunner.release()` leak |
| Knex | `knex.transaction(cb)` | Yes — nested `trx.transaction()` | `isolationLevel` option | `readOnly` option | No | Unhandled inner rejection rolls back outer |

**The universal caveat**: not one of these retries serialization/deadlock failures for you. If you run `SERIALIZABLE` (or hit deadlocks under `READ COMMITTED`), you must add the retry loop yourself — see §11.3. This is the single most common production omission when moving from a raw driver to an ORM.

---

## 13. Practice Exercises

### Exercise 1 — Basic

You are moving money-like state: an order is placed, its line items are inserted, and the product stock is decremented — all or nothing.

Given:
- `orders(id, user_id, total_amount, status, created_at)`
- `order_items(id, order_id, product_id, quantity, unit_price)`
- `products(id, name, stock, price)`

Write a single transaction that:
1. Inserts a new row into `orders` for `user_id = 42` with `status = 'pending'`.
2. Inserts two rows into `order_items` referencing that new order.
3. Decrements `products.stock` by the ordered quantity for each product.
4. Commits.

Use explicit `BEGIN` / `COMMIT`. Assume `orders.id` is generated by the database and you need to reference it in steps 2–3.

```sql
-- Write your query here
```

---

### Exercise 2 — Intermediate

A batch job imports 500 audit records. Some may violate a `CHECK` constraint. You want the whole batch to succeed, but if a *single* record fails you want to skip only that record and keep the rest — not abort the entire transaction.

Given:
- `audit_logs(id, actor_id, action, entity, created_at)`

Write a transaction using `SAVEPOINT` and `ROLLBACK TO SAVEPOINT` that inserts three audit records, where the second insert is expected to fail, and demonstrate that records 1 and 3 still commit. Explain in a comment why a plain `BEGIN ... INSERT ... INSERT ... COMMIT` would lose all three once one fails.

```sql
-- Write your query here
```

---

### Exercise 3 — Intermediate

Two employees are swapping departments and you must keep each department's headcount-derived budget consistent. You want to observe MVCC snapshot behaviour.

Given:
- `employees(id, name, department_id, salary)`
- `departments(id, name, budget)`

Write a transaction at `REPEATABLE READ` that:
1. Reads the count of employees in department 10.
2. Sleeps conceptually (a comment marking where a concurrent session would insert a new employee into department 10).
3. Reads the count again.
4. In a comment, state what value the second read returns under `REPEATABLE READ` versus `READ COMMITTED`, and why.

```sql
-- Write your query here
```

---

### Exercise 4 — Hard

You are writing a payment-capture routine that must be safe under concurrency and must recover partial work.

Given:
- `orders(id, user_id, total_amount, status)`
- `payments(id, order_id, amount, status, captured_at)`

Requirements:
1. Lock the target order row (`SELECT ... FOR UPDATE`) so no other session can capture it concurrently.
2. Insert a `payments` row with `status = 'authorized'`.
3. Set a `SAVEPOINT` before attempting the (fallible) capture update that sets `payments.status = 'captured'` and `orders.status = 'paid'`.
4. If the capture step fails, `ROLLBACK TO SAVEPOINT` and instead mark the payment `status = 'failed'`, then still `COMMIT` (so the failed attempt is recorded).
5. `RELEASE` the savepoint on the success path.

Write the full transaction and annotate what state the database is in at each step. Note in a comment what happens to the whole transaction if you *forget* the savepoint and the capture update throws.

```sql
-- Write your query here
```

---

### Exercise 5 — Interview Favourite

Explain, with runnable SQL, the "aborted transaction" state.

Write a transaction that:
1. Runs a valid `INSERT` into `sessions(id, user_id, token, created_at)`.
2. Runs a statement that raises an error (e.g. divides by zero or violates a constraint).
3. Attempts another valid `INSERT` — and show, in a comment, the exact error the database returns for step 3.
4. Recovers so that step 1's work can still be committed, using a savepoint set *before* step 2.

```sql
-- Write your query here
```

---

## 14. Interview Questions

### Junior Level

**Q: What do `BEGIN`, `COMMIT`, and `ROLLBACK` do?**

A: `BEGIN` (or `START TRANSACTION`) opens a transaction — from that point every statement is part of one atomic unit and its effects are invisible to other sessions until you finish. `COMMIT` makes all the changes permanent and visible to everyone. `ROLLBACK` discards every change made since `BEGIN`, leaving the database as if the transaction never ran. Without an explicit `BEGIN`, most databases run in *autocommit* mode, where each statement is its own tiny transaction that commits immediately.

**Follow-up — "If I never type `BEGIN`, is my `UPDATE` transactional?"**: Yes — it runs inside an implicit single-statement transaction and either fully applies or fully fails. Autocommit doesn't mean "no transaction"; it means "one statement per transaction, committed automatically."

---

**Q: What is a `SAVEPOINT`?**

A: A named marker inside an open transaction. `ROLLBACK TO SAVEPOINT s` undoes everything done *after* `s` while keeping the transaction open and keeping work done *before* `s`. It's a partial rollback. `RELEASE SAVEPOINT s` discards the marker (you can no longer roll back to it) without undoing any work. Savepoints are how you skip one bad statement in a batch without losing the whole batch.

**Follow-up — "Does `ROLLBACK TO` end the transaction?"**: No. The transaction stays open; you can continue issuing statements and must still `COMMIT` (or `ROLLBACK`) at the end. Only a bare `ROLLBACK` ends it.

---

**Q: What is autocommit?**

A: A connection mode where each individual statement is automatically wrapped in its own transaction and committed the moment it succeeds. It is the default in `psql`, most drivers, and most GUI tools. To group statements atomically you must explicitly open a transaction with `BEGIN`, which suspends autocommit until you `COMMIT` or `ROLLBACK`.

**Follow-up — "How do I turn it off in psql?"**: `\set AUTOCOMMIT off`, after which every statement implicitly starts a transaction that you must commit yourself.

---

### Principal Level

**Q: A batch process runs `BEGIN`, then 50 inserts, and insert #17 throws a constraint violation. The application catches the exception and issues insert #18. What happens, and how should this have been written?**

A: In PostgreSQL, once *any* statement errors inside a transaction, the transaction enters the **aborted state** (`25P02`, "current transaction is aborted"). Every subsequent statement — including insert #18 — is rejected with "commands ignored until end of transaction block", and a final `COMMIT` behaves like a `ROLLBACK`, so *all 50 inserts are lost*, not just #17. The correct pattern is to wrap each fallible statement in a `SAVEPOINT`: set a savepoint before each insert, and on error `ROLLBACK TO SAVEPOINT` and continue. This is exactly what many drivers/ORMs do transparently. Note MySQL/InnoDB differs — a statement error there rolls back only that statement by default and leaves the transaction usable — which is a portability trap when moving code from MySQL to Postgres.

**Follow-up — "Why does Postgres abort the whole transaction instead of just the statement?"**: It's a deliberate safety stance: after an error the transaction's invariants may be inconsistent, so Postgres refuses to let you build more work on an unknown state. Savepoints give you an explicit, opt-in recovery boundary rather than silently continuing.

---

**Q: Explain how MVCC and the WAL together give you atomicity and isolation. What actually happens on `COMMIT` versus `ROLLBACK`?**

A: Under **MVCC** (Multi-Version Concurrency Control), an `UPDATE` or `DELETE` never overwrites a row in place — it writes a *new* row version and marks the old one with the transaction's XID. Each transaction sees a snapshot: a row version is visible only if the transaction that created it committed before the snapshot and the version isn't superseded by another committed version. That's why readers never block writers. The **WAL** (Write-Ahead Log) records every change as a durable log record *before* the data pages are flushed. On `COMMIT`, Postgres writes a commit record to the WAL and flushes it to disk (`fsync`) — that flush is the durable moment; the transaction's XID is now marked committed, making its row versions visible to new snapshots. On `ROLLBACK`, essentially nothing is undone eagerly: the row versions this transaction wrote simply stay marked with an aborted XID and are therefore invisible to everyone; `VACUUM` later reclaims them. This is why rollback is cheap and commit is the expensive (disk-syncing) operation — the opposite of what people intuit.

**Follow-up — "So a long-running rollback is fast, but why can an idle-in-transaction session hurt me?"**: Because MVCC can't reclaim dead row versions newer than the oldest still-running snapshot. An open (even idle) transaction pins that horizon, causing table bloat and blocking `VACUUM` — the classic "idle in transaction" production incident.

---

**Q: What is a subtransaction, and what is the hidden cost of using thousands of savepoints in one transaction?**

A: A subtransaction is the internal mechanism behind a `SAVEPOINT` (and behind PL/pgSQL `BEGIN ... EXCEPTION` blocks) — each gets its own subtransaction XID (subxid). The hidden cost: Postgres tracks per-backend subxids in a fixed-size cache (`PGPROC`, 64 entries). Once a transaction exceeds 64 live subtransactions, the cache "overflows" and every other backend must consult the `pg_subtrans` structure on disk for visibility checks — a well-documented performance cliff (the "subtransaction SLRU / suboverflow" problem). So a loop that sets a fresh savepoint per row across tens of thousands of rows can silently degrade *cluster-wide* read performance. Mitigation: release savepoints promptly, batch the fallible work, or restructure so you don't need thousands of live savepoints simultaneously.

**Follow-up — "Does `RELEASE SAVEPOINT` free the subxid slot?"**: It marks the subtransaction as no longer needed, but the assigned subxid remains part of the transaction's history; the 64-slot cache pressure is driven by the count of subtransactions the transaction has opened, so the durable fix is opening fewer of them, not just releasing them.

---

## 15. Mental Model Checkpoint

1. You run three `UPDATE`s without ever typing `BEGIN`. How many transactions occurred, and at what points did durability happen?

2. Inside a transaction you `SAVEPOINT a`, insert a row, then `ROLLBACK TO SAVEPOINT a`. Is the transaction still open? Is the inserted row gone? Is savepoint `a` still usable?

3. In PostgreSQL, statement #4 of your transaction errors. What is returned by statement #5, and what does the final `COMMIT` actually do?

4. On `COMMIT`, what specifically is flushed to disk to guarantee durability? On `ROLLBACK`, what work is undone immediately — and why is rollback usually cheaper than commit?

5. Under MVCC, an `UPDATE` "changes" a row. What physically happens to the old row version, and when is its space reclaimed?

6. Why can a single connection sitting `idle in transaction` cause table bloat across the whole database, even though it's doing nothing?

7. You set one `SAVEPOINT` per row inside a loop over 100,000 rows. Nothing errors. Why might other, unrelated sessions in the cluster suddenly slow down?

---

## 16. Quick Reference Card

```sql
-- ─── Opening / closing ────────────────────────────────────────────
BEGIN;                          -- or: START TRANSACTION;
  ...statements...
COMMIT;                         -- make permanent + durable (WAL fsync)
ROLLBACK;                       -- discard everything since BEGIN

-- ─── Isolation / access mode (set at BEGIN, before first query) ───
BEGIN ISOLATION LEVEL REPEATABLE READ;
BEGIN ISOLATION LEVEL SERIALIZABLE READ ONLY DEFERRABLE;
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;   -- alt syntax

-- ─── Savepoints (partial rollback) ────────────────────────────────
SAVEPOINT sp1;                  -- mark a point
ROLLBACK TO SAVEPOINT sp1;      -- undo work AFTER sp1; txn stays open
RELEASE SAVEPOINT sp1;          -- discard marker; keep the work
-- Re-issuing SAVEPOINT with the same name shadows the older one.

-- ─── Batch pattern: skip a bad row, keep the rest ────────────────
BEGIN;
  SAVEPOINT s;
  INSERT INTO audit_logs (...) VALUES (...);   -- may fail
  -- on error in app code:  ROLLBACK TO SAVEPOINT s;  else RELEASE s;
COMMIT;

-- ─── Autocommit (psql) ───────────────────────────────────────────
\set AUTOCOMMIT off             -- each stmt now needs explicit COMMIT
\set AUTOCOMMIT on              -- default: 1 stmt = 1 txn

-- ─── State cheatsheet ────────────────────────────────────────────
-- Autocommit ON, no BEGIN → each statement = its own committed txn
-- After an error in Postgres → ABORTED state (25P02):
--   every further stmt → "current transaction is aborted"
--   COMMIT behaves like ROLLBACK  →  recover with ROLLBACK TO SAVEPOINT
-- MySQL/InnoDB → statement error rolls back only that statement (portability trap)

-- ─── Mechanics to remember ───────────────────────────────────────
-- MVCC : UPDATE/DELETE write NEW row versions; old ones stay until VACUUM
-- WAL  : change logged BEFORE data pages flush; COMMIT = fsync the WAL
-- COMMIT is the expensive (disk-sync) op; ROLLBACK is cheap (marks XID aborted)
-- >64 live subtransactions (savepoints) per txn → subxid cache OVERFLOW → cluster-wide slowdown
-- idle-in-transaction pins the MVCC horizon → blocks VACUUM → bloat
```

---

## Connected Topics

- **Isolation Levels (READ COMMITTED / REPEATABLE READ / SERIALIZABLE)**: Set at `BEGIN`, these decide *which* snapshot your transaction sees — the direct consequence of the MVCC model described here.
- **Locking (`SELECT ... FOR UPDATE`, row/table locks, deadlocks)**: The pessimistic counterpart to MVCC; how transactions serialize writes and how deadlocks abort one transaction.
- **The ACID Properties**: Atomicity and Durability are exactly what `COMMIT`/`ROLLBACK` + WAL provide; this topic is the "how" behind the "what".
- **Write-Ahead Log (WAL) and Crash Recovery**: How the commit record makes durability real and how the database replays the WAL after a crash.
- **VACUUM and Table Bloat**: The cleanup side of MVCC — reclaiming the dead row versions that `UPDATE`/`DELETE`/`ROLLBACK` leave behind, and why long transactions block it.
- **Optimistic vs Pessimistic Concurrency & Retry Loops**: Handling `40001` serialization failures — the application-level retry pattern that raw SQL and every ORM leaves to you.
- **ORM Transaction APIs (Prisma, Drizzle, Sequelize, TypeORM, Knex)**: How each library maps to `BEGIN`/`COMMIT`/`SAVEPOINT` and where their abstractions leak (covered in §12).
- **Distributed Transactions & Two-Phase Commit (2PC)**: Extending atomic commit across multiple databases — where `PREPARE TRANSACTION` and the coordinator come in.
