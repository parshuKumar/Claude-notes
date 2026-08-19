# 51 — Distributed Transactions and Two-Phase Commit
## Phase: Transactions & Concurrency

---

## ELI5 — The Simple Analogy

Three friends agreeing to meet. You want a guarantee that **either all three show up, or none does** — nobody wastes a trip.

So you appoint a coordinator. She calls each person: *"Can you commit to Saturday? Don't answer yes unless you're certain — once you say yes, you cannot change your mind."* Each person clears their calendar, blocks the slot, and says **yes**. That's **phase one: prepare.**

She then calls back: *"It's on."* That's **phase two: commit.**

Now the failure that makes this hard. Suppose everyone has said yes — they've all blocked Saturday and promised not to move — and then **the coordinator's phone dies**.

Nobody can move. They can't cancel (they promised). They can't confirm (nobody knows if the others said yes). ★ **They must sit there with Saturday blocked, indefinitely, until she comes back.** There is no timeout that is safe, because timing out and cancelling might contradict a decision she already made and told someone else.

That is the **blocking problem**, and it is not an implementation flaw. It is a proven property: **no protocol can guarantee atomic commit across nodes and also guarantee that everyone makes progress when the coordinator can fail.** Every real system either accepts the blocking, or gives up atomicity and does something else.

---

## Where this fits in the big picture

```
   39 transactions · 40 ACID · 44 isolation · 47 VACUUM (2PC pins xmin!)
                          │
                          ▼
        ┌──────────────────────────────────────────────┐
        │ 51 DISTRIBUTED TXNS & 2PC ← YOU ARE HERE     │
        │ ★ and why you almost certainly shouldn't     │
        └────────────────────┬─────────────────────────┘
                             ▼
              52 idempotency & the transactional outbox
                 ★ what you use INSTEAD
              68 CAP · 76 CDC
```

Topic 47's production incident was caused by a prepared transaction left behind for 107 days. **This topic explains what that object was, why it existed, and why the right fix was `max_prepared_transactions = 0`.**

---

## What is this?

A **distributed transaction** spans more than one independently-failing resource — two databases, a database and a message broker, two microservices — and must be atomic across all of them.

**Two-phase commit (2PC)** is the classic protocol:

```
 PHASE 1 — PREPARE (voting)
   coordinator → each participant: "prepare to commit"
   each participant:
     • does all the work
     • ★ writes everything it needs to WAL and FLUSHES
     • ★ holds ALL locks
     • ★ enters a state where it CANNOT unilaterally abort
     • replies YES or NO
   ⇒ a YES is an irrevocable promise.

 PHASE 2 — COMMIT / ABORT (decision)
   all YES  ⇒ coordinator logs COMMIT, tells everyone to commit
   any NO   ⇒ coordinator logs ABORT, tells everyone to abort
   ⇒ participants obey. They have no choice.
```

In PostgreSQL:

```sql
BEGIN;
INSERT INTO ledger_entries …;
PREPARE TRANSACTION 'txn-8842119';   -- ★ phase 1. Locks held, xmin pinned.
-- …the session may now disconnect entirely. The transaction survives.
COMMIT PREPARED 'txn-8842119';       -- phase 2
-- or
ROLLBACK PREPARED 'txn-8842119';
```

**The thing to internalise:** `PREPARE TRANSACTION` creates an object that **outlives your connection, outlives a server restart, holds every lock, and pins `xmin` forever**, until someone explicitly resolves it. It is the only construct in PostgreSQL that can do unbounded damage while being completely invisible on a connections dashboard.

---

## Why does it matter for a backend developer?

```
 ★ THREE REASONS, AND THE THIRD IS THE REAL ONE.

 ① YOU WILL BE ASKED TO DO THIS.
    "Order service writes to its DB, payment service writes to its
     DB, they must both succeed or both fail."
    ⇒ 2PC is the textbook answer and it is usually the wrong one.
      You need to be able to say WHY, specifically, and offer the
      alternative.

 ② IF SOMEONE ENABLED max_prepared_transactions, YOU HAVE A
    LATENT OUTAGE.
    ★ one abandoned prepared transaction blocks VACUUM CLUSTER-WIDE,
      forever, with no connection, no query, no CPU (Topic 47).
    ⇒ the payments incident: 107 days, 412 GB table, database
      refused all writes.

 ③ ★ THE ALTERNATIVE IS ALMOST ALWAYS BETTER, NOT JUST CHEAPER.
    Idempotency + a transactional outbox + eventual consistency
    gives you a system that keeps working when a participant is
    down, instead of one that blocks. (Topic 52.)
    ⇒ 2PC turns N independent services into ONE system whose
      availability is the PRODUCT of theirs.
```

### The availability arithmetic

```
 ★ THIS SINGLE CALCULATION ENDS MOST 2PC DISCUSSIONS.

 Each service: 99.9% available.
 Independent operation:  99.9%
 2PC across 2 services:  0.999² = 99.8%
 2PC across 3 services:  0.999³ = ★ 99.7%
 2PC across 5 services:  0.999⁵ = ★ 99.5%   (43.8 hours/year down)

 ⇒ 2PC does not merely ADD latency. It MULTIPLIES failure
   probabilities, because the transaction cannot complete unless
   EVERY participant is up for the WHOLE protocol.
 ⇒ ★ AND IT'S WORSE THAN THE ARITHMETIC: a coordinator failure
   between phases leaves participants BLOCKED — holding locks —
   which degrades them for everyone else too.
 ⇒ ⇒ ★ AVAILABILITY DOESN'T JUST MULTIPLY. FAILURES CASCADE.
```

---

## The physical reality

### What `PREPARE TRANSACTION` actually creates

```
 ON `PREPARE TRANSACTION 'gid'`:

 ① all the transaction's work is already in WAL
 ② ★ a 2PC state file is written:  $PGDATA/pg_twophase/<xid>
    (and, since PG 9.6, kept in shared memory until a checkpoint
     moves it to disk)
    it contains: the gid, the XID, the subtransaction list, the
    LOCKS HELD, and the invalidation messages
 ③ ★ the WAL record XLOG_XACT_PREPARE is written and FLUSHED
    ⇒ this is a real fsync. A prepared transaction is DURABLE
      before phase 2 begins.
 ④ ★ the backend DISASSOCIATES from the transaction:
    • the session can disconnect
    • the transaction still exists
    • ★ it survives a server restart
    • ★ it survives a failover to a replica (it's in the WAL)
 ⑤ ★ ALL LOCKS REMAIN HELD, owned by nobody in particular
 ⑥ ★ THE XID REMAINS IN THE "RUNNING" SET
    ⇒ oldest-xmin cannot advance past it
    ⇒ VACUUM cannot reclaim ANY tuple newer than it
    ⇒ ANYWHERE IN THE CLUSTER (Topic 47)

 SEE IT:
   SELECT gid, prepared, owner, database, transaction
     FROM pg_prepared_xacts;
   SELECT locktype, relation::regclass, mode
     FROM pg_locks WHERE virtualtransaction LIKE '-1/%';
   ★ virtualtransaction = '-1/...' means "held by a prepared
     transaction with no backend."
```

### The blocking problem, precisely

```
 ★ WHY 2PC CANNOT BE MADE NON-BLOCKING.

 Participant P has voted YES. It is waiting for phase 2.
 The coordinator becomes unreachable.

 CAN P ABORT?
   NO. The coordinator may have already logged COMMIT and told
   participant Q to commit. If P aborts, atomicity is violated.

 CAN P COMMIT?
   NO. The coordinator may have logged ABORT (because R voted NO).
   If P commits, atomicity is violated.

 CAN P ASK THE OTHER PARTICIPANTS?
   ★ SOMETIMES — this is "cooperative termination". If any
   participant knows the decision, it can share it. But if ALL
   surviving participants are in the prepared state and none knows,
   ★ THEY ALL BLOCK. The information simply does not exist anywhere
   except the failed coordinator's log.

 ⇒ ★ THE THEOREM (Skeen & Stonebraker, 1983):
   THERE IS NO NON-BLOCKING ATOMIC COMMIT PROTOCOL FOR ASYNCHRONOUS
   SYSTEMS WITH CRASH FAILURES.
   ⇒ 3PC reduces the window but assumes synchronous timing —
     an assumption real networks violate.
   ⇒ Paxos/Raft-based commit (Spanner, CockroachDB) replaces the
     single coordinator with a REPLICATED one, so the coordinator
     itself doesn't fail. ★ That is the actual modern answer —
     and it requires consensus infrastructure, not just 2PC.

 ⇒ AND WHILE P BLOCKS, IT HOLDS EVERY LOCK IT TOOK. So a
   coordinator crash degrades not just that transaction, but every
   other transaction that touches the same rows.
```

### Why PostgreSQL ships with `max_prepared_transactions = 0`

```
 THE DEFAULT IS 0. PREPARE TRANSACTION IS DISABLED OUT OF THE BOX.
 ★ THIS IS A DELIBERATE SAFETY DECISION, AND THE DOCUMENTATION
   SAYS SO EXPLICITLY.

 THE FOUR COSTS OF AN UNRESOLVED PREPARED TRANSACTION:
 ① ★ VACUUM STOPS CLUSTER-WIDE (its xmin pins everything)
 ② ★ WRAPAROUND MARCHES ON (freezing needs an advancing xmin)
    ⇒ eventually the database refuses all writes (Topic 47)
 ③ ★ LOCKS HELD FOREVER — any conflicting query blocks indefinitely
 ④ ★ INVISIBLE. No connection. No query. No CPU. Not in
    pg_stat_activity. It appears in exactly one place:
    pg_prepared_xacts.

 ⇒ ★ THE OPERATIONAL RULE:
   IF YOU ENABLE max_prepared_transactions, YOU MUST HAVE:
     ① a transaction manager that RELIABLY resolves prepared
        transactions after ITS OWN crash (a durable coordinator log)
     ② an alert on ANY prepared transaction older than 5 minutes
     ③ a documented manual-resolution runbook
   IF YOU DO NOT HAVE ALL THREE ⇒ ★ SET IT TO 0.
```

### The latency cost, measured

```
 A LOCAL COMMIT:  1 fsync
 A 2PC COMMIT:    ★ 2 fsyncs PER PARTICIPANT + 2 network round trips
                    + the coordinator's own 2 log flushes

   local commit                    0.4 ms
   2PC, 2 participants, same DC    ★ 4.2 ms      (10×)
   2PC, 2 participants, cross-AZ   ★ 11.8 ms     (29×)
   2PC, 3 participants, cross-region ★ 148 ms    (370×)

 ⇒ AND LOCKS ARE HELD FOR THE ENTIRE DURATION, INCLUDING THE
   NETWORK TIME. ⇒ contention = hold_time × arrival_rate (Topic 49)
 ⇒ ★ SO 2PC DOESN'T JUST MAKE ONE TRANSACTION SLOW. IT MULTIPLIES
   CONTENTION FOR EVERY TRANSACTION TOUCHING THE SAME ROWS.
```

---

## How it works — step by step

### The protocol, with every failure point marked

```
  COORDINATOR                 PARTICIPANT A          PARTICIPANT B
       │                            │                      │
   ┌───┴────────────────────────────┴──────────────────────┴───┐
   │ PHASE 1 — PREPARE                                          │
   └───┬────────────────────────────┬──────────────────────┬───┘
       │──── "prepare" ────────────►│                      │
       │──── "prepare" ─────────────┼─────────────────────►│
       │                            │ do work              │ do work
       │                            │ ★ WAL + FSYNC        │ ★ WAL + FSYNC
       │                            │ ★ HOLD ALL LOCKS     │ ★ HOLD ALL LOCKS
       │                            │ ★ CANNOT ABORT NOW   │ ★ CANNOT ABORT
       │◄─── "yes" ─────────────────│                      │
       │◄─── "yes" ─────────────────┼──────────────────────│
       │                            │                      │
       │ ★ ⚠ FAILURE POINT 1: coordinator dies HERE        │
       │   ⇒ A and B block FOREVER holding locks           │
       │   ⇒ ★ neither can safely decide                   │
       │                            │                      │
   ┌───┴────────────────────────────┴──────────────────────┴───┐
   │ PHASE 2 — COMMIT                                           │
   └───┬────────────────────────────┬──────────────────────┬───┘
       │ ★ LOG "COMMIT" DURABLY ← the point of no return    │
       │──── "commit" ─────────────►│                      │
       │                            │ commit, release locks│
       │                            │                      │
       │ ★ ⚠ FAILURE POINT 2: coordinator dies here        │
       │   ⇒ A committed, B still prepared                 │
       │   ⇒ ★ B blocks until the coordinator RECOVERS its │
       │     log and re-sends. The DECISION exists, so this │
       │     is recoverable — but only if the coordinator's │
       │     log survived.                                  │
       │──── "commit" ──────────────┼─────────────────────►│
       │                            │                      │ commit
       │◄─── ack ───────────────────│◄─────────────────────│

 ★ THE ASYMMETRY THAT MATTERS:
   • FAILURE POINT 2 is RECOVERABLE — the decision was logged.
   • ★ FAILURE POINT 1 IS NOT — no decision exists anywhere.
     Only a human, or a coordinator that comes back with its log
     intact, can resolve it.
```

### Manual resolution — the runbook you will need

```sql
-- ① FIND THEM
SELECT gid, prepared, now()-prepared AS age, owner, database,
       transaction AS xid
  FROM pg_prepared_xacts ORDER BY prepared;

-- ② SEE WHAT IT HOLDS
SELECT l.locktype, l.relation::regclass, l.mode, l.granted
  FROM pg_locks l
 WHERE l.virtualtransaction = '-1/' || (
   SELECT transaction FROM pg_prepared_xacts WHERE gid = 'txn-8842119');

-- ③ ★ SEE THE DAMAGE
SELECT relname, n_dead_tup, last_autovacuum FROM pg_stat_user_tables
 ORDER BY n_dead_tup DESC LIMIT 5;
SELECT datname, age(datfrozenxid) FROM pg_database ORDER BY 2 DESC;

-- ④ ★ DECIDE. THIS IS THE HARD PART, AND IT IS NOT A DATABASE
--    QUESTION. You must determine what the coordinator decided —
--    from ITS log, from the other participants, or from business
--    state ("did the payment actually go through?").
--    ⇒ GUESSING BREAKS ATOMICITY, WHICH IS THE ONLY THING 2PC
--      WAS BOUGHT FOR.

COMMIT PREPARED   'txn-8842119';
-- or
ROLLBACK PREPARED 'txn-8842119';

-- ⑤ CONFIRM
SELECT count(*) FROM pg_prepared_xacts;
VACUUM (VERBOSE) the_bloated_table;   -- ★ now it can reclaim
```

### When 2PC *is* the right answer

```
 ★ IT IS NOT NEVER. THE LEGITIMATE CASES:

 ① TWO DATABASES YOU FULLY CONTROL, IN ONE DATACENTRE, WITH A
    REAL TRANSACTION MANAGER (JTA/XA, Atomikos, Narayana) that has
    a durable log and automatic recovery.
    ⇒ common in banking cores and ERP systems. It works. It is
      operated by people who know it.

 ② ★ POSTGRES_FDW ACROSS SHARDS, where a single logical write must
    hit two shards and the application genuinely cannot tolerate a
    window of inconsistency.
    ⇒ Citus does this; it manages the prepared transactions for you
      and has its own recovery daemon. ★ You are not hand-rolling it.

 ③ MIGRATIONS: dual-writing to an old and a new store during a
    cutover, for a bounded period, with an operator watching.

 ⇒ ★ WHAT THEY ALL HAVE IN COMMON:
   • a transaction manager with a DURABLE LOG and AUTOMATIC RECOVERY
   • all participants in ONE failure domain
   • an operator who knows the runbook
   • ★ a bounded, monitored lifetime
 ⇒ IF ANY OF THOSE IS MISSING, USE TOPIC 52's PATTERNS INSTEAD.
```

---

## Concept breakdown

```
THE PROBLEM
└── atomicity across independently-failing resources

2PC — THE CLASSIC ANSWER
├── phase 1 PREPARE: do the work, fsync, ★ hold locks, vote,
│    ★ lose the right to abort
└── phase 2 COMMIT/ABORT: coordinator decides, participants obey

★ WHAT `PREPARE TRANSACTION` CREATES IN POSTGRESQL
├── survives disconnection AND server restart AND failover
├── ★ holds every lock, owned by no backend
├── ★ pins xmin ⇒ VACUUM stops CLUSTER-WIDE ⇒ wraparound
└── ★ INVISIBLE except in pg_prepared_xacts

★ THE BLOCKING PROBLEM — a theorem, not a bug
   a participant that voted YES cannot abort (someone may have
   committed) and cannot commit (someone may have aborted)
   ⇒ no non-blocking atomic commit protocol exists for
     asynchronous systems with crash failures
   ⇒ 3PC assumes synchronous timing (real networks violate it)
   ⇒ ★ the real fix is a REPLICATED coordinator (Paxos/Raft) —
     Spanner, CockroachDB. Consensus, not 2PC.

★ THE AVAILABILITY ARITHMETIC — the argument that ends discussions
   3 services at 99.9% ⇒ 2PC gives 99.7%
   ⇒ and coordinator failure leaves locks held ⇒ failures CASCADE

★ THE LATENCY — 2 fsyncs per participant + 2 round trips
   local 0.4ms → same-DC 4.2ms → cross-region ★ 148ms
   ⇒ and locks are held for ALL of it

★ WHY THE DEFAULT IS max_prepared_transactions = 0
   an abandoned prepared transaction is the single most damaging
   invisible object in PostgreSQL

★ WHAT TO DO INSTEAD — the actual answer (Topic 52)
├── ① ONE database, ONE transaction — reconsider the boundaries
├── ② ★ TRANSACTIONAL OUTBOX — local commit + async delivery
├── ③ SAGA — a sequence of local transactions + compensations
└── ④ IDEMPOTENCY KEYS — make retries safe, then just retry
     ⇒ these trade ATOMICITY for EVENTUAL CONSISTENCY,
       and gain AVAILABILITY and INDEPENDENT FAILURE

WHEN 2PC IS ACTUALLY RIGHT
   a real transaction manager with a durable log + automatic
   recovery, one failure domain, a monitored bounded lifetime
   ⇒ Citus/XA yes; hand-rolled between two microservices no
```

---

## Diagrams

**Diagram 1 — big picture: the two failure points**

```
       PHASE 1                          PHASE 2
  ┌──────────────────┐            ┌──────────────────┐
  │ prepare / vote   │            │ commit / abort   │
  └────────┬─────────┘            └────────┬─────────┘
           │                               │
   ┌───────┴────────┐             ┌────────┴────────┐
   │ participants:  │             │ participants:   │
   │ • work done    │             │ • obey          │
   │ • fsynced      │             │ • release locks │
   │ • ★ LOCKS HELD │             │                 │
   │ • ★ CANNOT     │             │                 │
   │   ABORT        │             │                 │
   └───────┬────────┘             └─────────────────┘
           │
     ★ ⚠ FAILURE HERE = THE BLOCKING PROBLEM
       ┌────────────────────────────────────────────────┐
       │ no decision exists ANYWHERE                    │
       │ • participants cannot abort (someone may have  │
       │   committed)                                   │
       │ • cannot commit (someone may have aborted)     │
       │ • ★ locks held indefinitely                    │
       │ • ★ xmin pinned ⇒ VACUUM dead cluster-wide     │
       │ ⇒ RESOLVABLE ONLY BY THE COORDINATOR'S LOG     │
       │   OR A HUMAN                                   │
       └────────────────────────────────────────────────┘

     ⚠ FAILURE IN PHASE 2 IS DIFFERENT — the decision WAS logged.
       Participants block, but recovery is mechanical.
```

**Diagram 2 — data flow: 2PC vs the transactional outbox**

```
 ✗ 2PC — atomic, blocking, availability multiplies
 ┌──────────────────────────────────────────────────────────────────┐
 │  order-svc ──prepare──► orders DB    ⏸ locks held               │
 │      │      ──prepare──► payments DB ⏸ locks held               │
 │      │      ──prepare──► Kafka       ⏸ (if it even supports XA) │
 │      │                                                            │
 │      └──commit──► all three                                       │
 │                                                                   │
 │  ★ availability = 0.999³ = 99.7%                                 │
 │  ★ latency = 2 fsync × 3 + 2 RTT = ~12–150 ms                    │
 │  ★ ANY participant down ⇒ NOTHING WORKS                          │
 │  ★ coordinator down mid-protocol ⇒ locks held indefinitely       │
 └──────────────────────────────────────────────────────────────────┘

 ✓ TRANSACTIONAL OUTBOX — eventually consistent, non-blocking
 ┌──────────────────────────────────────────────────────────────────┐
 │  order-svc:                                                       │
 │    BEGIN;                        ← ★ ONE local transaction        │
 │      INSERT INTO orders …;                                        │
 │      INSERT INTO outbox (event_type, payload, idempotency_key)…;  │
 │    COMMIT;                       ← ★ ONE fsync. 0.4 ms. Done.     │
 │                                                                   │
 │  relay (separate process, or logical decoding — Topic 76):        │
 │    SELECT … FROM outbox WHERE published_at IS NULL                │
 │     ORDER BY id FOR UPDATE SKIP LOCKED LIMIT 100;                 │
 │    → publish to Kafka → UPDATE outbox SET published_at = now();   │
 │                                                                   │
 │  payment-svc: consumes, ★ deduplicates on idempotency_key,        │
 │               commits ITS OWN local transaction                   │
 │                                                                   │
 │  ★ availability = each service's own                             │
 │  ★ latency = 0.4 ms on the write path                            │
 │  ★ payments down ⇒ orders KEEP WORKING; events queue up          │
 │  ★ no locks held across services, ever                           │
 │  ✗ TRADE: a window where the order exists and the payment        │
 │    hasn't been processed. ⇒ ★ THE BUSINESS MUST TOLERATE THIS.   │
 └──────────────────────────────────────────────────────────────────┘
```

**Diagram 3 — before/after: the 107-day prepared transaction**

```
 ✗ BEFORE — max_prepared_transactions = 8, "for safety"
 ┌────────────────────────────────────────────────────────────────┐
 │ 2 May 04:11  reconciliation job:                                │
 │                BEGIN; … ; PREPARE TRANSACTION 'recon_...';      │
 │                ★ then the job process was OOM-killed            │
 │                ★ no coordinator, no recovery, no alert          │
 │                                                                 │
 │ 2 May → 17 Aug (107 days):                                      │
 │   pg_stat_activity        ★ shows NOTHING                       │
 │   CPU / connections       ★ normal                              │
 │   autovacuum              ★ ran 1,841 times, reclaimed ZERO     │
 │   payment_attempts        12 GB → ★ 412 GB                      │
 │   age(datfrozenxid)       → ★ 2,145,482,106                     │
 │                                                                 │
 │ 17 Aug 02:14  ERROR: database is not accepting commands to      │
 │               avoid wraparound data loss                        │
 │               ★ 1h 27m total outage                             │
 └────────────────────────────────────────────────────────────────┘

 ✓ AFTER
 ┌────────────────────────────────────────────────────────────────┐
 │ ALTER SYSTEM SET max_prepared_transactions = 0;   ★ the fix     │
 │                                                                 │
 │ the reconciliation job rewritten as:                            │
 │   • ONE local transaction (it only ever touched one database)   │
 │   • an idempotency key with a UNIQUE constraint                 │
 │   • re-runnable from scratch on failure                         │
 │                                                                 │
 │ ★ AND THE FINDING THAT MATTERS MOST:                           │
 │   the job NEVER NEEDED 2PC. It wrote to one database. Someone   │
 │   used PREPARE TRANSACTION because it sounded safer than        │
 │   COMMIT.                                                       │
 │                                                                 │
 │ + alert: any row in pg_prepared_xacts older than 5 minutes      │
 └────────────────────────────────────────────────────────────────┘
```

---

## Example 1 — basic

```sql
-- 2PC is OFF by default. Turn it on only to learn.
SHOW max_prepared_transactions;   -- 0
ALTER SYSTEM SET max_prepared_transactions = 10;
-- ★ requires a restart
```

```sql
CREATE TABLE accounts (id bigint PRIMARY KEY, balance_minor bigint NOT NULL);
INSERT INTO accounts VALUES (1,100000),(2,100000);
```

**Prepare a transaction and watch it outlive the session.**
```sql
BEGIN;
UPDATE accounts SET balance_minor = balance_minor - 5000 WHERE id = 1;
PREPARE TRANSACTION 'transfer-001';
-- ★ the session is now free. The transaction is not gone.
\q
```
```bash
psql -d shop
```
```sql
SELECT gid, prepared, owner, database, transaction FROM pg_prepared_xacts;
```
```
     gid      |         prepared          | owner  | database | transaction
--------------+---------------------------+--------+----------+-------------
 transfer-001 | 2026-08-18 09:14:22+05:30 | backend| shop     |     8842119
   ★ the session that created it is gone. This object is not.
```
```sql
SELECT balance_minor FROM accounts WHERE id=1;
```
```
 balance_minor
---------------
        100000        ★ not committed — invisible to everyone
```

**See the locks it holds, owned by no backend.**
```sql
SELECT locktype, relation::regclass AS rel, mode, granted, virtualtransaction
  FROM pg_locks WHERE virtualtransaction LIKE '-1/%';
```
```
 locktype |   rel    |       mode       | granted | virtualtransaction
----------+----------+------------------+---------+--------------------
 relation | accounts | RowExclusiveLock | t       | -1/8842119
 transactionid |     | ExclusiveLock    | t       | -1/8842119
   ★ '-1/' means "held by a prepared transaction, no backend".
```

**Prove it blocks conflicting work.**
```sql
UPDATE accounts SET balance_minor = 1 WHERE id = 1;
-- ⏸ HANGS. Forever. There is no session to cancel.
```
```sql
-- from another session
SELECT pid, pg_blocking_pids(pid), wait_event_type, wait_event
  FROM pg_stat_activity WHERE cardinality(pg_blocking_pids(pid)) > 0;
```
```
  pid  | pg_blocking_pids | wait_event_type | wait_event
-------+------------------+-----------------+--------------
 41288 | {}               | Lock            | transactionid
   ★ blocked_by is EMPTY — there is no blocking PID, because
     there is no blocking process. This is the diagnostic signature.
```

**Prove it survives a restart.**
```bash
pg_ctl restart -D "$PGDATA" -m fast
psql -d shop -c "SELECT gid, prepared FROM pg_prepared_xacts;"
```
```
     gid      |         prepared
--------------+---------------------------
 transfer-001 | 2026-08-18 09:14:22+05:30
   ★ still there. It is in the WAL and in pg_twophase/.
```
```bash
ls -la "$PGDATA/pg_twophase/"
```
```
 -rw------- 1 postgres postgres  216 Aug 18 09:14 000000000086E5D7
   ★ the on-disk state file. Named by XID in hex.
```

**★ Prove it stops VACUUM cluster-wide.**
```sql
CREATE TABLE unrelated (id bigserial PRIMARY KEY, n int);
INSERT INTO unrelated (n) SELECT g FROM generate_series(1,200000) g;
UPDATE unrelated SET n = n + 1;
VACUUM (VERBOSE) unrelated;
```
```
INFO:  table "unrelated": found 0 removable, 400000 nonremovable row versions
DETAIL:  ★ 200000 dead row versions cannot be removed yet, oldest xmin: 8842119
   ★ 8842119 IS THE PREPARED TRANSACTION'S XID.
   ★ AND `unrelated` HAS NOTHING TO DO WITH `accounts`.
     One prepared transaction froze cleanup for the entire cluster.
```

**Resolve it.**
```sql
ROLLBACK PREPARED 'transfer-001';
SELECT count(*) FROM pg_prepared_xacts;   -- 0
VACUUM (VERBOSE) unrelated;
```
```
INFO:  table "unrelated": found 200000 removable, 200000 nonremovable row versions
DETAIL:  0 dead row versions cannot be removed yet
   ★ same command. The only thing that changed.
```

**A complete, successful 2PC across two databases.**
```sql
-- database `orders`
\c orders
BEGIN;
INSERT INTO orders (id, customer_id, total_minor) VALUES (9001, 42, 250000);
PREPARE TRANSACTION 'order-9001';

-- database `payments`
\c payments
BEGIN;
INSERT INTO payments (order_id, amount_minor, status)
  VALUES (9001, 250000, 'authorised');
PREPARE TRANSACTION 'order-9001';

-- ★ THE COORDINATOR'S JOB: it must now DURABLY LOG "commit"
--   BEFORE issuing either COMMIT PREPARED. If it doesn't, and it
--   crashes, nobody can ever resolve these.
\c orders
COMMIT PREPARED 'order-9001';
\c payments
COMMIT PREPARED 'order-9001';
```

**Simulate the coordinator dying between phases.**
```sql
\c orders
BEGIN;
INSERT INTO orders (id, customer_id, total_minor) VALUES (9002, 42, 100000);
PREPARE TRANSACTION 'order-9002';
\c payments
BEGIN;
INSERT INTO payments (order_id, amount_minor, status)
  VALUES (9002, 100000, 'authorised');
PREPARE TRANSACTION 'order-9002';
-- ★ the coordinator process is now killed. No decision was logged.
```
```sql
-- both databases:
SELECT gid, now()-prepared AS age FROM pg_prepared_xacts;
```
```
     gid    |      age
------------+---------------
 order-9002 | 00:04:18.229
   ★ AND THEY WILL STAY THIS WAY FOREVER.
     No timeout. No cleanup. VACUUM dead in both databases.
     ⇒ a HUMAN must decide, and must determine what the coordinator
       would have decided. Guessing breaks atomicity.
```

**Turn it back off.**
```sql
ALTER SYSTEM SET max_prepared_transactions = 0;
-- restart. ★ Any unresolved prepared transaction must be resolved
--   FIRST or the server will refuse to start.
```

---

## Example 2 — production scenario

**The situation.** An e-commerce platform splits its monolith into `order-service` and `inventory-service`, each with its own PostgreSQL database. The requirement is stated as:

> *"Placing an order and reserving stock must be atomic. We cannot have an order with no reservation, or a reservation with no order."*

The team implements 2PC with a home-grown coordinator.

```
 AFTER 6 WEEKS IN PRODUCTION
   checkout p99            140 ms → ★ 890 ms
   checkout availability   99.95% → ★ 99.61%   (34 hours/year)
   incidents caused by
     prepared transactions ★ 4 (two requiring manual resolution
                             at 3am)
   inventory-service
     deploys                ★ now cause checkout errors — a rolling
                             restart drops the coordinator mid-protocol
   both databases' bloat    ★ growing; autovacuum intermittently
                             reclaiming nothing
```

**Step 1 — quantify what 2PC actually bought.**

```sql
-- how often did the "atomicity" guarantee actually matter?
-- i.e. how many transactions had a participant vote NO?
SELECT decision, count(*) FROM coordinator_log
 WHERE decided_at > now() - interval '30 days' GROUP BY 1;
```
```
 decision |  count
----------+---------
 commit   | 8842119
 abort    |    ★ 41
```
```
 ★ 41 ABORTS IN 8.8 MILLION TRANSACTIONS — 0.0005%.
   And of those 41, 38 were "insufficient stock", which
   inventory-service could have reported with a plain HTTP 409
   BEFORE anything was written anywhere.
 ⇒ ★ THE PROTOCOL WAS PAYING ITS FULL COST 8.8 MILLION TIMES TO
   HANDLE 3 GENUINE CROSS-SERVICE FAILURES.
```

**Step 2 — quantify what it cost.**

```sql
-- prepared transactions lingering, sampled every minute for 30 days
SELECT date_trunc('day', sampled_at) AS day,
       max(max_age_seconds) AS worst_lingering_seconds,
       sum(CASE WHEN max_age_seconds > 300 THEN 1 ELSE 0 END) AS minutes_over_5min
  FROM prepared_xact_samples GROUP BY 1 ORDER BY 1 DESC LIMIT 5;
```
```
    day     | worst_lingering_seconds | minutes_over_5min
------------+-------------------------+-------------------
 2026-08-17 |                  ★ 4102 |              ★ 68
 2026-08-16 |                      12 |                 0
 2026-08-15 |                  ★ 8804 |             ★ 147
```
```sql
-- and the vacuum damage on those days
SELECT relname, n_dead_tup, last_autovacuum FROM pg_stat_user_tables
 WHERE relname IN ('orders','reservations') ORDER BY n_dead_tup DESC;
```
```
   relname    | n_dead_tup |    last_autovacuum
--------------+------------+------------------------
 reservations |  ★ 4188204 | 2026-08-15 09:04:11+05:30
 orders       |    1204882 | 2026-08-17 22:14:02+05:30
```

**Step 3 — challenge the requirement.**

```
 ★ ASK: "WHAT ACTUALLY HAPPENS IF THEY ARE INCONSISTENT FOR
   200 MILLISECONDS?"

 CASE A — order exists, reservation doesn't (yet)
   ⇒ for ~200 ms, an order is placed and stock isn't reserved.
   ⇒ ★ ANOTHER CUSTOMER MIGHT BUY THE LAST UNIT.
   ⇒ CONSEQUENCE: one order gets cancelled and refunded.
   ⇒ FREQUENCY: only matters when stock is near zero for a
     contended SKU.
   ⇒ ★ MEASURED: the existing oversell rate from OTHER causes
     (returns, damaged stock, supplier shortfalls) was ALREADY
     0.4% of orders. The business had a refund process. It ran
     ~1,200 times a month regardless.

 CASE B — reservation exists, order doesn't
   ⇒ stock is held for an order that failed.
   ⇒ CONSEQUENCE: stock is unavailable until a sweeper releases it.
   ⇒ ★ FIX: reservations already expire after 15 minutes. This is
     a solved problem in the existing design.

 ⇒ ★ THE CONCLUSION: the "atomicity" requirement was never a
   business requirement. It was an ENGINEERING ASSUMPTION,
   inherited from the monolith where a single transaction was free.
 ⇒ THE REAL REQUIREMENT: "an order must eventually have a
   reservation or be cancelled, and stock must never be held
   indefinitely for a non-existent order."
   ⇒ ★ THAT IS AN EVENTUAL-CONSISTENCY REQUIREMENT, AND IT IS
     MUCH CHEAPER TO SATISFY.
```

**Step 4 — the outbox rewrite.**

```sql
-- in the orders database
CREATE TABLE outbox (
  id             bigserial PRIMARY KEY,
  aggregate_type text        NOT NULL,
  aggregate_id   bigint      NOT NULL,
  event_type     text        NOT NULL,
  payload        jsonb       NOT NULL,
  idempotency_key text       NOT NULL UNIQUE,   -- ★ the consumer's dedupe key
  created_at     timestamptz NOT NULL DEFAULT now(),
  published_at   timestamptz
);
-- ★ a partial index: the relay only ever queries unpublished rows,
--   and the index stays tiny even as the table grows
CREATE INDEX idx_outbox_unpublished ON outbox (id) WHERE published_at IS NULL;
```

```js
// ★ ONE local transaction. One fsync. No coordinator.
async function placeOrder(customerId, items, requestId) {
  return withTransaction(async (tx) => {
    const { rows: [order] } = await tx.query(
      `INSERT INTO orders (customer_id, total_minor, status)
       VALUES ($1, $2, 'pending_reservation')
       RETURNING id`, [customerId, total(items)]);

    // ★ the event is written in the SAME transaction as the order.
    //   If the order commits, the event exists. If it rolls back,
    //   the event doesn't. THAT is the atomicity that was actually
    //   needed — and it is local.
    await tx.query(
      `INSERT INTO outbox (aggregate_type, aggregate_id, event_type,
                           payload, idempotency_key)
       VALUES ('order', $1, 'OrderPlaced', $2, $3)`,
      [order.id,
       JSON.stringify({ order_id: order.id, customer_id: customerId, items }),
       `order-placed:${order.id}`]);

    return order;
  });
}
```

```js
// the relay — a separate process. ★ SKIP LOCKED means N relays
// run concurrently with zero contention (Topic 45).
async function relayLoop() {
  for (;;) {
    const published = await withTransaction(async (tx) => {
      const { rows } = await tx.query(
        `SELECT id, event_type, payload, idempotency_key
           FROM outbox
          WHERE published_at IS NULL
          ORDER BY id
          LIMIT 100
            FOR UPDATE SKIP LOCKED`);
      if (!rows.length) return 0;

      // ★ at-least-once. The consumer MUST deduplicate.
      await broker.publishBatch(rows.map(r => ({
        key: r.idempotency_key, type: r.event_type, value: r.payload })));

      await tx.query(
        'UPDATE outbox SET published_at = now() WHERE id = ANY($1)',
        [rows.map(r => r.id)]);
      return rows.length;
    });
    if (published === 0) await sleep(200);
  }
}
```

```js
// inventory-service consumes — its own local transaction
async function onOrderPlaced(event) {
  await withTransaction(async (tx) => {
    // ★ dedupe FIRST. This is what makes at-least-once delivery safe.
    const { rowCount } = await tx.query(
      `INSERT INTO processed_events (idempotency_key)
       VALUES ($1) ON CONFLICT DO NOTHING`, [event.key]);
    if (rowCount === 0) return;              // already handled

    for (const item of [...event.items].sort((a,b) => a.sku_id - b.sku_id)) {
      const { rowCount: ok } = await tx.query(
        `UPDATE inventory SET on_hand = on_hand - $1
          WHERE sku_id = $2 AND on_hand >= $1`, [item.qty, item.sku_id]);
      if (ok === 0) {
        // ★ the compensating path — a saga step (Topic 52)
        await tx.query(
          `INSERT INTO outbox (aggregate_type, aggregate_id, event_type,
                               payload, idempotency_key)
           VALUES ('order', $1, 'ReservationFailed', $2, $3)`,
          [event.order_id, JSON.stringify({ reason: 'INSUFFICIENT_STOCK', item }),
           `reservation-failed:${event.order_id}`]);
        return;
      }
    }
    await tx.query(
      `INSERT INTO reservations (order_id, expires_at)
       VALUES ($1, now() + interval '15 minutes')
       ON CONFLICT (order_id) DO NOTHING`, [event.order_id]);
    await tx.query(
      `INSERT INTO outbox (…) VALUES ('order', $1, 'StockReserved', …)`, […]);
  });
}
```

**Step 5 — make the eventual consistency observable, not hoped-for.**

```sql
-- ★ THE MOST IMPORTANT PART OF ANY EVENTUALLY-CONSISTENT DESIGN:
--   an alert on the LAG, and a reconciler for what falls through.

-- ① relay lag
SELECT count(*) AS unpublished,
       extract(epoch from now() - min(created_at)) AS oldest_seconds
  FROM outbox WHERE published_at IS NULL;
-- ★ alert: oldest_seconds > 60

-- ② orders stuck without a resolution
SELECT count(*) FROM orders
 WHERE status = 'pending_reservation'
   AND created_at < now() - interval '5 minutes';
-- ★ alert: > 0. This is the "eventually" not happening.

-- ③ the reconciler — runs every minute, fixes what fell through
UPDATE orders SET status = 'cancelled', cancel_reason = 'reservation_timeout'
 WHERE status = 'pending_reservation'
   AND created_at < now() - interval '10 minutes'
RETURNING id;
-- ⇒ ★ THIS IS THE PIECE PEOPLE FORGET. Eventual consistency
--   without a reconciler is just inconsistency with extra steps.
```

**Step 6 — turn 2PC off, permanently.**

```sql
-- resolve every lingering prepared transaction first
SELECT gid, now()-prepared AS age FROM pg_prepared_xacts;
-- (decide each from the coordinator log, then COMMIT/ROLLBACK PREPARED)

ALTER SYSTEM SET max_prepared_transactions = 0;
-- ★ restart. The server REFUSES to start if any remain unresolved —
--   which is correct: it will not silently discard them.

-- and the alert that stays forever, on both databases
SELECT count(*) FROM pg_prepared_xacts WHERE prepared < now() - interval '5 min';
```

**Step 7 — results, 8 weeks.**

| | 2PC | Outbox + saga |
|---|---|---|
| Checkout p99 | 890 ms | 118 ms (**7.5×**) |
| Checkout availability | 99.61% | 99.97% |
| Writes per checkout | 2 prepares + 2 commits, 2 DBs | ★ **1 local commit** |
| `inventory-service` down | ★ checkout **fails** | ★ checkout **works**, events queue |
| `inventory-service` deploy | causes checkout errors | invisible |
| Prepared-transaction incidents | 4 in 6 weeks | **0** (feature disabled) |
| Vacuum health | intermittently frozen | normal |
| Inconsistency window | 0 | ★ p99 **340 ms**, p999 1.8 s |
| Orders needing reconciliation | n/a | ★ 12 in 8 weeks, all auto-resolved |

```
 ★ THE HONEST TRADE, STATED PLAINLY:
   THEY GAVE UP a 0-millisecond consistency window.
   THEY GAINED  7.5× latency, independent service availability,
                deployability, and the elimination of an entire
                class of 3am incident.
   THEY PAID    a reconciler, a dedupe table, and the discipline
                of monitoring lag.

 ★ AND THE FINDING THAT SHOULD COME FIRST NEXT TIME:
   0.0005% of transactions ever aborted, and 93% of THOSE were
   "insufficient stock" — which is a normal HTTP 409, not a
   distributed-atomicity problem.
   ⇒ MEASURE HOW OFTEN THE GUARANTEE IS EXERCISED BEFORE PAYING
     FOR IT.
```

---

## Common mistakes

**1. Enabling `max_prepared_transactions` without a recovering coordinator.**
- *Symptom:* an abandoned prepared transaction blocks VACUUM cluster-wide, invisibly, until wraparound.
- *Engine-level why:* the prepared XID stays in the running set; oldest-xmin cannot advance past it.
- *Fix:* set it to 0 unless you have a transaction manager with a durable log and automatic recovery, plus an alert and a runbook.

**2. Using `PREPARE TRANSACTION` for a single-database operation.**
- *Symptom:* the 107-day incident — someone used it because it "sounded safer" than `COMMIT`.
- *Fix:* one database means one transaction. `COMMIT`.

**3. Assuming a timeout can safely resolve a prepared transaction.**
- *Symptom:* an automatic rollback that contradicts a commit another participant already performed.
- *Engine-level why:* the blocking problem — a prepared participant genuinely cannot know the decision.
- *Fix:* resolution must come from the coordinator's durable log or a human who determines what it decided.

**4. Not alerting on `pg_prepared_xacts`.**
- *Symptom:* the object is invisible on every normal dashboard.
- *Fix:* alert on any row older than 5 minutes. It is a one-line query.

**5. Reaching for 2PC before measuring how often atomicity is exercised.**
- *Symptom:* paying full protocol cost 8.8 million times for 3 genuine cross-service failures.
- *Fix:* measure the abort rate first. Most "aborts" are business rejections that could be checked before writing anything.

**6. Holding a prepared transaction across a slow operation.**
- *Symptom:* locks and `xmin` pinned for the duration of a network call.
- *Fix:* if you must use 2PC, the prepare→commit window must be milliseconds.

**7. Treating eventual consistency as "no consistency."**
- *Symptom:* an outbox with no lag alert and no reconciler; inconsistencies accumulate silently.
- *Fix:* eventual consistency requires a *bound* and a *reconciler*. Measure the lag; sweep what falls through.

**8. Building a saga without compensations.**
- *Symptom:* a partial sequence with no way to unwind.
- *Fix:* every forward step needs a defined compensating action, and compensations must be idempotent (Topic 52).

**9. Believing 3PC solves it.**
- *Symptom:* a more complex protocol with the same fundamental exposure.
- *Engine-level why:* 3PC assumes synchronous timing bounds that real networks violate. Under a network partition it can produce inconsistency.
- *Fix:* if you truly need non-blocking atomic commit, you need a *replicated* coordinator (Raft/Paxos) — i.e. Spanner or CockroachDB, not a protocol tweak.

**10. Forgetting the consumer must deduplicate.**
- *Symptom:* an outbox delivers at-least-once, the consumer applies twice, stock double-decrements.
- *Fix:* a `processed_events` table with a `UNIQUE` idempotency key, checked first (Topic 52).

---

## Hands-on proof

**PROVE IT #1–#7 — Example 1** (a prepared transaction outliving its session, surviving a restart, its on-disk state file, locks with `virtualtransaction = '-1/…'`, a blocked query whose `pg_blocking_pids` is *empty*, VACUUM frozen on an unrelated table, and resolution restoring it).

**PROVE IT #8 — the diagnostic signature.**
```sql
-- an ordinary lock wait
SELECT pid, pg_blocking_pids(pid) FROM pg_stat_activity
 WHERE cardinality(pg_blocking_pids(pid)) > 0;
--  41288 | {41202}     ← there is a blocker

-- a prepared-transaction wait
--  41288 | {}          ← ★ EMPTY. wait_event = 'transactionid'.
-- ⇒ ★ "blocked, with no blocking PID" means: check pg_prepared_xacts.
SELECT pid, wait_event_type, wait_event FROM pg_stat_activity
 WHERE wait_event = 'transactionid';
SELECT gid, transaction FROM pg_prepared_xacts;
```

**PROVE IT #9 — measure the latency cost.**
```bash
cat > /tmp/local.sql <<'EOF'
BEGIN;
UPDATE accounts SET balance_minor = balance_minor - 1 WHERE id = 1;
COMMIT;
EOF
cat > /tmp/twopc.sql <<'EOF'
BEGIN;
UPDATE accounts SET balance_minor = balance_minor - 1 WHERE id = 1;
PREPARE TRANSACTION 'bench-:client_id';
COMMIT PREPARED 'bench-:client_id';
EOF
pgbench -f /tmp/local.sql -c 20 -j 4 -T 30 shop | grep -E 'tps|latency'
pgbench -f /tmp/twopc.sql -c 20 -j 4 -T 30 shop | grep -E 'tps|latency'
```
```
 local : tps = 14,204.8   latency average = 1.41 ms
 2PC   : tps =  4,102.1   latency average = ★ 4.88 ms   (3.5× — and
         this is ONE database with NO network. Add a second
         participant and a real coordinator and it is 10–30×.)
```

**PROVE IT #10 — the server refuses to start with unresolved prepared transactions.**
```bash
psql -c "BEGIN; UPDATE accounts SET balance_minor=1 WHERE id=1; PREPARE TRANSACTION 'stuck';"
psql -c "ALTER SYSTEM SET max_prepared_transactions = 0"
pg_ctl restart -D "$PGDATA" -m fast
```
```
FATAL:  maximum number of prepared transactions reached
HINT:  Increase max_prepared_transactions (currently 0).
   ★ it will not silently discard them. Resolve first, then disable.
```

**PROVE IT #11 — the outbox, end to end.**
```sql
BEGIN;
INSERT INTO orders (customer_id, total_minor, status)
  VALUES (42, 250000, 'pending_reservation') RETURNING id \gset
INSERT INTO outbox (aggregate_type, aggregate_id, event_type, payload, idempotency_key)
  VALUES ('order', :id, 'OrderPlaced', '{"order_id":' || :id || '}', 'order-placed:' || :id);
COMMIT;
-- ★ ONE fsync. Both rows, or neither. No coordinator involved.

SELECT id, event_type, published_at FROM outbox WHERE published_at IS NULL;
```

---

## The design decision framework

```
★★★ ASSUME YOU DO NOT NEED 2PC. MAKE THE CASE, DON'T ASSUME IT. ★★★

 ① CAN THIS BE ONE TRANSACTION IN ONE DATABASE?
    ⇒ ★ THE MOST COMMON RIGHT ANSWER, AND THE MOST OFTEN SKIPPED.
    "atomicity across services" is often a sign the SERVICE
    BOUNDARY is wrong, not that you need a distributed protocol.
    ⇒ if two things must be atomic, they may belong together.

 ② MEASURE HOW OFTEN ATOMICITY IS ACTUALLY EXERCISED
    what fraction of these operations would ABORT?
    ⇒ 0.0005% with 93% of those being business rejections
      ⇒ ★ you are paying a protocol tax on every request to handle
        a case a pre-check would catch.
    ⇒ CHECK PRECONDITIONS FIRST (HTTP 409), THEN WRITE.

 ③ ASK: "WHAT BREAKS IF THESE ARE INCONSISTENT FOR 500 ms?"
    ⇒ ★ ASK THE BUSINESS, NOT THE ENGINEERS.
    if the answer is "a small number of orders get cancelled and
    refunded, which already happens for other reasons"
      ⇒ ★ YOU DO NOT NEED ATOMICITY. You need EVENTUAL consistency
        with a BOUND and a RECONCILER.

 ④ THE ALTERNATIVES, IN ORDER (Topic 52)
    ① one database, one transaction        ★ always try first
    ② ★ TRANSACTIONAL OUTBOX — local commit + async publish
       ⇒ atomicity between YOUR state and YOUR event. Local.
    ③ SAGA — local transactions + compensating actions
    ④ IDEMPOTENCY KEYS — make retries safe, then just retry
    ⇒ these trade a consistency WINDOW for AVAILABILITY and
      INDEPENDENT FAILURE.

 ⑤ IF YOU GENUINELY NEED 2PC, YOU MUST HAVE ALL FOUR
    ✓ a transaction manager with a DURABLE LOG and AUTOMATIC
      RECOVERY (XA/JTA, or Citus which manages it for you)
    ✓ all participants in ONE failure domain
    ✓ ★ an alert on any prepared transaction older than 5 minutes
    ✓ a written manual-resolution runbook, and someone who has
      practised it
    ⇒ ★ MISSING ANY ONE ⇒ max_prepared_transactions = 0

 ⑥ IF YOU RUN 2PC
    • prepare→commit window in MILLISECONDS, never seconds
    • monitor pg_prepared_xacts continuously
    • ★ never resolve by guessing — determine the coordinator's
      decision, or atomicity was never real
    • remember the availability multiplies: 0.999ⁿ

 ⑦ EVENTUAL CONSISTENCY IS NOT "NO CONSISTENCY"
    ★ it requires THREE things, and teams routinely ship only one:
      ① at-least-once delivery      (the outbox)
      ② ★ consumer-side dedupe      (idempotency key + UNIQUE)
      ③ ★ a RECONCILER + a LAG ALERT
    ⇒ without ③ you have built inconsistency with extra steps.
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
Enable `max_prepared_transactions`, create a prepared transaction, then: (a) disconnect and show it survives; (b) restart the server and show it survives; (c) find its on-disk state file; (d) show the locks it holds and explain `virtualtransaction = '-1/…'`; (e) run a conflicting `UPDATE` and show `pg_blocking_pids` returns *empty*; (f) show it blocks VACUUM on an unrelated table; (g) resolve it and show VACUUM working.

### Exercise 2 — medium (apply it)
Set up two databases and run a complete 2PC by hand. Then simulate a coordinator crash between phases and explain, precisely, why neither participant can safely decide. Write the manual runbook: how you'd determine what the coordinator would have decided, in order of preference, and what you'd do if none of those sources survived.

Then measure: local commit vs 2PC throughput and latency with `pgbench`.

### Exercise 3 — hard (production simulation)
A team split a monolith into `order-service` and `inventory-service` and used hand-rolled 2PC to keep order placement and stock reservation atomic. After 6 weeks: p99 140 → 890 ms, availability 99.95% → 99.61%, four prepared-transaction incidents, and inventory deploys now break checkout.

(a) Compute the theoretical availability of 2PC across 2 and 3 services at 99.9% each. Explain why the observed number is *worse* than the arithmetic predicts.
(b) Write the query that measures how often the atomicity guarantee was actually exercised. The answer is 41 aborts in 8.8M transactions, 38 of them "insufficient stock." What does that tell you?
(c) For each direction of inconsistency (order without reservation; reservation without order), state the concrete business consequence and how long a window would be tolerable. Which one is already a solved problem in the existing design?
(d) Design the outbox: schema, the partial index and why it matters, the write-path transaction, and the relay using `SKIP LOCKED`.
(e) Explain precisely what atomicity the outbox *does* guarantee, and why that is the atomicity that was actually needed.
(f) Write the consumer, including the dedupe step. Explain why at-least-once delivery is safe *only* with it.
(g) Write the reconciler and the two lag alerts. Explain why a design without these is "inconsistency with extra steps."
(h) Give the sequence for turning `max_prepared_transactions` off safely, including what happens if you skip a step.
(i) State the trade honestly: what was given up, what was gained, what new operational burden was accepted.
(j) Under what circumstances would you have *kept* 2PC? Name all four preconditions.

---

## Mental model checkpoint

1. Describe both phases of 2PC and say what a participant loses the moment it votes yes.
2. What does `PREPARE TRANSACTION` create in PostgreSQL? Name four things it outlives or holds.
3. State the blocking problem. Why can a prepared participant neither commit nor abort?
4. Why is there no non-blocking atomic commit protocol? What do Spanner and CockroachDB do instead?
5. Compute 2PC availability across three 99.9% services. Why is reality worse than that number?
6. Why is `max_prepared_transactions = 0` the default, and what four things does an abandoned prepared transaction cost?
7. What is the diagnostic signature of a query blocked by a prepared transaction?
8. Explain what atomicity the transactional outbox provides, and what it does *not*.
9. Name the three things an eventually-consistent design needs. Which one do teams most often omit?
10. Name the four preconditions under which 2PC is a defensible choice.

---

## Quick reference card

**The protocol**
```
 phase 1  PREPARE : work + fsync + ★ hold locks + vote (★ cannot abort after)
 phase 2  COMMIT  : coordinator logs the decision durably, then tells everyone
 ★ coordinator dies between phases ⇒ participants BLOCK indefinitely
```

**PostgreSQL**
```sql
BEGIN; …; PREPARE TRANSACTION 'gid';   -- survives disconnect, restart, failover
COMMIT PREPARED 'gid';  |  ROLLBACK PREPARED 'gid';
SHOW max_prepared_transactions;        -- ★ 0 by default, and that is correct
```

**Diagnose**
```sql
SELECT gid, prepared, now()-prepared AS age, database FROM pg_prepared_xacts;
SELECT locktype, relation::regclass, mode FROM pg_locks
 WHERE virtualtransaction LIKE '-1/%';          -- ★ held by no backend
-- ★ blocked query with pg_blocking_pids() = {} ⇒ check pg_prepared_xacts
```

**The costs of one abandoned prepared transaction**
1. ★ VACUUM stops **cluster-wide** 2. wraparound advances → write refusal 3. locks held forever 4. ★ invisible everywhere except `pg_prepared_xacts`

**Availability:** `0.999ⁿ` — 3 services ⇒ 99.7%. **Latency:** local 0.4 ms → same-DC 4.2 ms → cross-region **148 ms**.

**Instead (Topic 52):** one DB/one transaction → **transactional outbox** → saga + compensations → idempotency keys.

**Eventual consistency needs three things:** at-least-once delivery · ★ consumer dedupe · ★ **a reconciler + a lag alert**.

**Alert, forever, on every database:**
```sql
SELECT count(*) FROM pg_prepared_xacts WHERE prepared < now() - interval '5 minutes';
```

---

## When would I use this at work?

1. **When someone proposes distributed transactions between microservices.** The availability arithmetic (`0.999ⁿ`) plus "how often does the abort path actually fire?" usually settles it in five minutes — and points at the outbox instead.

2. **When you inherit a system with `max_prepared_transactions > 0`.** Check `pg_prepared_xacts` immediately, add the alert, and find out whether a recovering transaction manager exists. If not, you have a latent multi-hour outage with no timer on it.

3. **Debugging "this query is blocked but nothing is blocking it."** Empty `pg_blocking_pids` with `wait_event = 'transactionid'` is the signature. It's a five-second diagnosis if you know it and an afternoon if you don't.

4. **Designing service boundaries.** "These two writes must be atomic" is strong evidence they belong in the same service and the same database. 2PC is often the cost of a boundary drawn in the wrong place — and moving the boundary is cheaper than the protocol.

---

## Connected topics

**Understand before this:** 39 (transactions), 40 (ACID — atomicity and durability), 41/42 (WAL and recovery — how prepare is made durable), 45 (locks — what a prepared transaction holds), 47 (VACUUM — the xmin pin, and the 107-day incident).

**This unlocks:**
- **52** — idempotency and transactional messaging: the outbox and saga patterns in full
- **68** — CAP and consistency models: where atomicity sits among the guarantees
- **76** — polyglot persistence and CDC: logical decoding as an outbox without a table
- **62** — replication: how prepared transactions interact with failover
