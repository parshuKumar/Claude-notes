# 40 — ACID in Depth
## Phase: Transactions & Concurrency

---

## ELI5 — The Simple Analogy

Four different promises a bank makes about your transfer, and four different people who keep them.

**Atomicity** — *"both halves happen, or neither."* The clerk tears up the slip if anything goes wrong. Kept by the person who can undo the paperwork.

**Consistency** — *"the books still balance afterwards."* The auditor's rule: total money in the bank is the same before and after a transfer. Kept by the **rules you wrote down** — nobody enforces a rule that was never written.

**Isolation** — *"nobody sees the middle."* While the ₹500 is in flight, no other clerk sees an account with money missing. Kept by the person managing who may look at what, and when.

**Durability** — *"once I say it's done, it's done."* Even if the building burns down, the record survives. Kept by whoever writes to the fireproof ledger *before* telling you it's done.

Four promises, four mechanisms, four ways to break. And the one people misunderstand is **Consistency** — because the database only guarantees the rules you actually declared.

---

## Where this fits in the big picture

```
   39 what a transaction is (the boundary)
                          │
                          ▼
        ┌──────────────────────────────────────────┐
        │ 40 ACID IN DEPTH        ← YOU ARE HERE   │
        │ what a transaction GUARANTEES,           │
        │ and which component keeps each promise   │
        └────────────────────┬─────────────────────┘
                             ▼
     A → 41 WAL, 42 recovery      (atomicity + durability)
     C → 24 constraints           (you declare it; the engine keeps it)
     I → 43–46, 50                (anomalies, isolation, locks, MVCC)
     D → 41 WAL, 62 replication
```

Topic 39 defined the *unit*. **This topic defines the four properties that unit has** — and maps each to the machinery that implements it, so the rest of Phase 5 has a place to hang.

---

## What is this?

Four properties a transaction system guarantees:

| | Property | Guarantee | Kept by |
|---|---|---|---|
| **A** | Atomicity | all or nothing | WAL + `pg_xact` |
| **C** | Consistency | valid state → valid state | **your constraints** |
| **I** | Isolation | concurrent transactions don't interfere | MVCC + locks |
| **D** | Durability | committed survives a crash | WAL fsync |

The acronym is famous and slightly misleading: **A, I and D are properties of the database. C is a property of your schema.** The database's contribution to C is only "I will enforce whatever you declared."

---

## Why does it matter for a backend developer?

Because each letter has a specific, common way of being weakened — often by a configuration change someone made for performance:

```
 ① ATOMICITY is broken by the boundary, not the engine
    Two statements, two implicit transactions ⇒ no atomicity.
    An external call inside ⇒ the DB half rolls back, the charge doesn't.
    ⇒ ★ the engine's atomicity is perfect. Yours is where it fails.

 ② CONSISTENCY is only as strong as your DDL
    "The database guarantees consistency" is false as usually meant.
    It guarantees YOUR declared constraints. A rule you enforce in
    application code is not a database guarantee, and it has a race
    window (Topics 24, 30).

 ③ ISOLATION is weaker than you think by default
    PostgreSQL defaults to READ COMMITTED, which permits non-repeatable
    reads, phantoms, lost updates and write skew (Topics 43, 44).
    ⇒ ★ most developers assume SERIALIZABLE semantics and get none.

 ④ DURABILITY IS A SETTING, AND PEOPLE TURN IT OFF
    synchronous_commit = off  →  up to 3× faster commits, and a
    window of committed-then-lost transactions on crash.
    fsync = off               →  faster still, and corruption on crash.
    ⇒ ★ both appear in "performance tuning" blog posts without the
      warning. Know what you've traded.
```

---

## The physical reality

### Which component keeps which promise

```
 ┌─────────────────────────────────────────────────────────────────┐
 │ A — ATOMICITY                                                   │
 │   WAL: every change is logged BEFORE the page is written        │
 │   pg_xact: 2 bits say committed / aborted / in-progress         │
 │   ⇒ ROLLBACK = set the bit to ABORTED. Nothing is undone;       │
 │     the tuples simply become invisible. (Topic 39.)             │
 │   ⇒ CRASH = replay WAL to the last checkpoint, then mark any    │
 │     transaction without a commit record as ABORTED. (Topic 42.) │
 ├─────────────────────────────────────────────────────────────────┤
 │ C — CONSISTENCY                                                 │
 │   ★ NOT A DATABASE MECHANISM. It is the set of rules YOU wrote: │
 │     NOT NULL · CHECK · UNIQUE · FOREIGN KEY · EXCLUDE (T24)     │
 │   The engine's promise: "a transaction that would violate any   │
 │   declared constraint is aborted."                              │
 │   ⇒ A rule you did not declare is not guaranteed. Full stop.    │
 ├─────────────────────────────────────────────────────────────────┤
 │ I — ISOLATION                                                   │
 │   MVCC: every row version carries xmin/xmax; every transaction  │
 │         carries a snapshot (Topic 46)                           │
 │   Locks: row locks for write-write conflicts (Topic 45)         │
 │   SSI:   predicate tracking for SERIALIZABLE (Topic 50)         │
 │   ⇒ THE ONLY LETTER WITH A DIAL. Four levels (Topic 44).        │
 ├─────────────────────────────────────────────────────────────────┤
 │ D — DURABILITY                                                  │
 │   WAL fsync at COMMIT. The commit returns only after the OS     │
 │   confirms the bytes are on stable storage.                     │
 │   ⇒ ★ ALSO A DIAL: synchronous_commit has five settings.        │
 └─────────────────────────────────────────────────────────────────┘
```

### The durability dial, in full

```
 synchronous_commit — what must be flushed before COMMIT returns

  off             ★ commit returns IMMEDIATELY. WAL is flushed by the
                    walwriter within ~3× wal_writer_delay (600 ms).
                    ⇒ DATA LOSS WINDOW: up to ~600 ms of COMMITTED
                      transactions on a crash.
                    ⇒ ✓ NO CORRUPTION — the database is consistent,
                      it has simply forgotten recent commits.
                    ⇒ 2–3× faster commits.

  local           flush to THIS server's disk. (= `on` with no replicas)

  on   (default)  flush to this server's disk, fsync confirmed.
                  ⇒ zero loss on a single-server crash.

  remote_write    + at least one synchronous replica has RECEIVED and
                    written it to its OS (not necessarily fsynced)

  remote_apply    + at least one synchronous replica has APPLIED it,
                    so a read on that replica will see it
                  ⇒ ★ the strongest, and the slowest — a full network
                    round trip per commit.

 ⚠ AND THE ONE THAT IS NOT A DIAL:
   fsync = off      ⇒ ★ NEVER. This does not lose recent commits —
                      it permits ARBITRARY CORRUPTION on crash,
                      because pages may be written out of order.
                      Use it only for a throwaway test database you
                      are willing to recreate.
   full_page_writes = off
                    ⇒ ★ risks TORN PAGES unless your storage
                      guarantees atomic 8 KB writes. Almost none do.
```

### The cost of durability, measured

```
 A single-row INSERT, 8,000 commits:

  synchronous_commit = on          1,204 tps    fsync per commit
  synchronous_commit = off         3,880 tps    ★ 3.2×, 600 ms at risk
  synchronous_commit = remote_apply  412 tps    network round trip

 ★ AND THE THING THAT MAKES `on` AFFORDABLE: GROUP COMMIT.
   Under concurrency, PostgreSQL batches many transactions' WAL into
   one fsync (commit_delay / commit_siblings).
     1 concurrent client:    1,204 tps  (1 fsync per commit)
   200 concurrent clients:  18,400 tps  (≈ 90 commits per fsync)
   ⇒ the per-commit fsync cost AMORTISES. Durability is cheap under
     load and expensive when idle — the opposite of most intuitions.
```

---

## How it works — step by step

### A — Atomicity

```
 THE MECHANISM, in three parts:

 ① WRITE-AHEAD RULE: no data page may be written to disk before the
    WAL record describing the change is durable. (Topic 41.)
    ⇒ so a crash can never leave a page changed with no log of why.

 ② pg_xact: the single source of truth for "did this commit?"
    A tuple with xmin = 91012 is visible ONLY IF pg_xact[91012] says
    COMMITTED. Until then it exists on the page and is invisible.

 ③ RECOVERY: replay WAL forward from the last checkpoint. Any
    transaction with no commit record is implicitly aborted — no undo
    is needed, because its tuples were never visible.

 ★ WHY POSTGRESQL NEEDS NO UNDO LOG:
   Oracle and InnoDB write the NEW value in place and the OLD value to
   a rollback segment; abort means copying the old values back.
   PostgreSQL writes a NEW TUPLE and leaves the old one; abort means
   flipping two bits.
   ⇒ TRADE: O(1) rollback and no undo-log contention, paid for with
     VACUUM (Topic 47). This single design choice explains most of
     PostgreSQL's character.

 ⚠ THE LIMIT OF ATOMICITY:
   It covers DATABASE work only. A payment gateway charge, an email,
   an S3 upload — none of these are rolled back. (Topics 39, 52.)
```

### C — Consistency: the letter people misread

```
 THE TEXTBOOK DEFINITION:
   "a transaction takes the database from one VALID state to another
    VALID state."

 ★ VALID ACCORDING TO WHAT? Only your declared constraints.

 EXAMPLE — the classic transfer:
   BEGIN;
     UPDATE accounts SET balance = balance - 500 WHERE id=1;
     UPDATE accounts SET balance = balance + 500 WHERE id=2;
   COMMIT;

   THE INVARIANT: sum(balance) is unchanged.
   ⇒ ★ THE DATABASE DOES NOT KNOW THIS. There is no constraint
     expressing it, and none can be — a CHECK cannot see other rows
     (Topic 24).
   ⇒ If a bug writes `- 500` and `+ 50`, the transaction is ATOMIC,
     ISOLATED and DURABLE — and the books do not balance.
     ⇒ ★ ACID DID NOT PROTECT YOU. The missing constraint did.

 WHAT THE ENGINE ACTUALLY GUARANTEES:
   ✓ NOT NULL, CHECK, UNIQUE, PRIMARY KEY, FOREIGN KEY, EXCLUDE
   ✓ type and domain constraints
   ✓ that a violation ABORTS the transaction rather than being
     partially applied
   ✗ any rule you did not declare
   ✗ any rule that spans rows and cannot be a constraint

 ⇒ ★ THE PRACTICAL CONSEQUENCE — Phase 4 was about C:
   normalisation exists so that facts live where the engine CAN
   constrain them (Topic 30's unifying sentence).
   ⇒ "we validate in the application" means "C is not guaranteed."

 ⇒ FOR CROSS-ROW INVARIANTS the options are (Topic 24):
   ① restructure so a UNIQUE or EXCLUDE expresses it   ★ best
   ② a trigger WITH an explicit lock
   ③ SERIALIZABLE isolation (Topic 50)
   ④ application logic + a scheduled reconciliation (case study 03)
```

### I — Isolation: the only letter with a dial

```
 THE FOUR LEVELS (Topic 44 in full):

                    dirty   non-repeatable  phantom   lost    write
                    read    read            read      update  skew
  READ UNCOMMITTED   —¹      possible       possible  poss.   poss.
  READ COMMITTED ★   no      possible       possible  poss.   poss.
  REPEATABLE READ    no      no             no²       no³     poss.
  SERIALIZABLE       no      no             no        no      no

  ¹ PostgreSQL has no true READ UNCOMMITTED; it behaves as READ COMMITTED
  ² PostgreSQL's REPEATABLE READ (snapshot isolation) also prevents
    phantoms — stronger than the SQL standard requires
  ³ prevented by aborting with a serialization failure, not by blocking

 ★ THE DEFAULT IS READ COMMITTED, and it permits lost updates and
   write skew. Most developers assume otherwise.

 THE MECHANISM:
   MVCC gives each transaction a SNAPSHOT (Topic 46). Readers never
   block writers; writers never block readers. Write-WRITE conflicts
   still need row locks (Topic 45).
   SERIALIZABLE adds predicate tracking (SSI) and aborts transactions
   that would produce a non-serialisable outcome (Topic 50).

 ⇒ ★ ISOLATION IS THE ONLY LETTER YOU ROUTINELY TRADE.
   And unlike durability, weakening it does not lose data — it
   permits INCORRECT data. That is worse.
```

### D — Durability

```
 THE COMMIT SEQUENCE (Topic 39, step 5):
   a) build the commit WAL record
   b) ★ fsync() the WAL file up to that LSN
   c) mark pg_xact COMMITTED
   d) remove the xid from the proc array
   e) release locks
   ⇒ the client is told "committed" only after (b).

 ★ NOTE WHAT IS *NOT* DURABLE AT COMMIT:
   the heap pages. They are still dirty in the buffer pool and will be
   written by the checkpointer minutes later (Topic 07).
   ⇒ durability is a property of the LOG, not of the table file.

 THE FAILURE MODES, and what each survives:
   process crash        → survived by WAL replay             ✓ always
   OS crash / power cut → survived IF fsync actually reached
                          stable storage
   ★ disk lies about fsync → NOT survived. Consumer SSDs with
     volatile write caches, and some virtualised storage, acknowledge
     fsync before the data is durable.
     ⇒ TEST IT: `pg_test_fsync` measures your storage's real fsync
       behaviour. A result of 100,000+ fsync/s means the cache is
       lying to you.
   whole-machine loss   → needs replication (Topic 62) or archiving
                          (Topic 64). Durability on one node is not
                          durability of your business.
```

---

## Concept breakdown

```
A — ATOMICITY   all or nothing
├── WAL (write-ahead rule) + pg_xact (2 bits)
├── ROLLBACK = one bit. No undo log. O(1). (Topic 39.)
└── ★ covers DATABASE work only — not gateways, email, or queues

C — CONSISTENCY   valid state → valid state
├── ★ NOT A DATABASE MECHANISM. It is your DDL.
├── the engine promises: a declared-constraint violation ABORTS
├── ⇒ Phase 4 (normalisation) exists so facts live where they CAN
│     be constrained
└── cross-row invariants: EXCLUDE/UNIQUE > trigger+lock >
    SERIALIZABLE > app + reconciliation

I — ISOLATION   concurrent transactions don't interfere
├── MVCC snapshots + row locks + (for SERIALIZABLE) SSI
├── ★ THE ONLY LETTER WITH A DIAL — four levels
├── default READ COMMITTED permits lost updates and write skew
└── ★ weakening it does not lose data; it permits WRONG data

D — DURABILITY   committed survives a crash
├── WAL fsync at COMMIT. The LOG is durable; the heap is not yet.
├── ★ ALSO A DIAL: synchronous_commit off/local/on/remote_write/remote_apply
├── GROUP COMMIT amortises the fsync — durability is cheap under load
└── ⚠ fsync=off is NOT a dial. It permits corruption, not just loss.

★ THE HONEST SUMMARY
  A and D are engine properties you can weaken by configuration.
  I is a dial you choose per transaction.
  C is YOUR schema. The database only enforces what you declared.
```

---

## Diagrams

**Diagram 1 — big picture: four promises, four mechanisms**

```
 ┌──────────────┬────────────────────────┬───────────────────────────┐
 │ PROPERTY     │ MECHANISM              │ HOW IT FAILS              │
 ├──────────────┼────────────────────────┼───────────────────────────┤
 │ ATOMICITY    │ WAL + pg_xact          │ your BOUNDARY is wrong,   │
 │              │ (2 bits per txn)       │ or an external call is    │
 │              │                        │ inside it                 │
 ├──────────────┼────────────────────────┼───────────────────────────┤
 │ CONSISTENCY  │ ★ YOUR CONSTRAINTS     │ you didn't declare the    │
 │              │ NOT NULL · CHECK ·     │ rule, or it can't be a    │
 │              │ UNIQUE · FK · EXCLUDE  │ constraint and you didn't │
 │              │                        │ build another mechanism   │
 ├──────────────┼────────────────────────┼───────────────────────────┤
 │ ISOLATION    │ MVCC snapshots +       │ the DEFAULT LEVEL is      │
 │              │ row locks + SSI        │ weaker than you assumed   │
 ├──────────────┼────────────────────────┼───────────────────────────┤
 │ DURABILITY   │ WAL fsync at COMMIT    │ synchronous_commit = off, │
 │              │                        │ or storage that lies      │
 │              │                        │ about fsync               │
 └──────────────┴────────────────────────┴───────────────────────────┘
```

**Diagram 2 — data flow: where each letter is enforced**

```
   BEGIN
     │
     ├── statement ──▶ [ C ] constraints checked per statement
     │                     NOT NULL, CHECK, UNIQUE, FK, EXCLUDE
     │                     ⇒ violation ⇒ ABORT the whole transaction
     │
     ├── read ──────▶ [ I ] snapshot decides what is visible (MVCC)
     │
     ├── write ─────▶ [ I ] row lock taken (t_xmax in the tuple)
     │                [ A ] WAL record → WAL buffer (RAM)
     │                      heap page modified in the buffer pool
     │
   COMMIT
     │
     ├──────────────▶ [ C ] DEFERRED constraints checked here
     │                      (DEFERRABLE FK, UNIQUE, EXCLUDE — T24)
     │
     ├──────────────▶ [ D ] ★ fsync the WAL   ← THE DURABILITY POINT
     │
     ├──────────────▶ [ A ] pg_xact = COMMITTED
     │
     └──────────────▶ [ I ] xid removed from the proc array
                            ⇒ now visible to new snapshots

   (later) ─────────▶ checkpointer writes dirty heap pages
                      ⇒ ★ the TABLE becomes durable long after
                        the transaction did
```

**Diagram 3 — before/after: the durability dial**

```
 synchronous_commit = on   (default)
 ┌──────────────────────────────────────────────────────────────────┐
 │ COMMIT ──▶ WAL fsync ──▶ "committed" ──▶ client                  │
 │              │                                                    │
 │              └── on disk. A power cut here loses NOTHING.        │
 │ 1,204 tps single client · 18,400 tps at 200 clients (group commit)│
 └──────────────────────────────────────────────────────────────────┘

 synchronous_commit = off
 ┌──────────────────────────────────────────────────────────────────┐
 │ COMMIT ──▶ "committed" ──▶ client                                 │
 │              │                                                    │
 │              └── WAL still in RAM. walwriter flushes within       │
 │                  ~600 ms.                                         │
 │ ★ A POWER CUT HERE LOSES UP TO 600 ms OF COMMITTED TRANSACTIONS. │
 │ ✓ NO CORRUPTION — the database is consistent, just older.        │
 │ 3,880 tps single client · 3.2× faster                            │
 └──────────────────────────────────────────────────────────────────┘

 fsync = off
 ┌──────────────────────────────────────────────────────────────────┐
 │ ★ NOT A DIAL. Pages may reach disk out of order.                 │
 │ A crash can leave the database CORRUPT and unrecoverable.        │
 │ ⇒ throwaway test databases only.                                 │
 └──────────────────────────────────────────────────────────────────┘
```

---

## Example 1 — basic

**Step 1 — Atomicity: all or nothing.**

```sql
CREATE TABLE accounts (
  id bigint PRIMARY KEY,
  balance_minor bigint NOT NULL CHECK (balance_minor >= 0)
);
INSERT INTO accounts VALUES (1,100000),(2,50000);

BEGIN;
  UPDATE accounts SET balance_minor = balance_minor - 150000 WHERE id=1;
  -- ERROR: violates check constraint "accounts_balance_minor_check"
  UPDATE accounts SET balance_minor = balance_minor + 150000 WHERE id=2;
  -- ERROR: current transaction is aborted
COMMIT;
SELECT * FROM accounts;
```
```
 id | balance_minor
----+---------------
  1 |        100000
  2 |         50000      ★ neither change applied
```

**Step 2 — Consistency: the engine enforces what you declared, and nothing more.**

```sql
-- the invariant: total money is conserved
SELECT sum(balance_minor) AS total FROM accounts;   -- 150000

-- a buggy transfer — atomic, isolated, durable, and WRONG
BEGIN;
  UPDATE accounts SET balance_minor = balance_minor - 50000 WHERE id=1;
  UPDATE accounts SET balance_minor = balance_minor +  5000 WHERE id=2;  -- ★ typo
COMMIT;

SELECT sum(balance_minor) AS total FROM accounts;
```
```
  total
--------
 105000        ★ ₹450 vanished. ACID did not stop it.
```
```sql
-- can you declare the invariant?
ALTER TABLE accounts ADD CONSTRAINT ck_total
  CHECK ((SELECT sum(balance_minor) FROM accounts) = 150000);
-- ERROR: cannot use subquery in check constraint     ★ (Topic 24)
```
**The rule is not expressible as a constraint.** The correct answer is a double-entry ledger where the invariant *becomes* expressible (case study 03):

```sql
CREATE TABLE ledger_entries (
  txn_id bigint NOT NULL,
  account_id bigint NOT NULL,
  amount_minor bigint NOT NULL CHECK (amount_minor <> 0)
);
-- ★ now "money is conserved" is  sum(amount_minor) = 0  per txn_id,
--   checkable in one query — and enforceable in the same transaction.
```

**Step 3 — Isolation: the default is weaker than you think.**

```sql
-- session 1                          -- session 2
SHOW transaction_isolation;           -- read committed
BEGIN;
SELECT balance_minor FROM accounts
  WHERE id=1;                -- 100000
                                      BEGIN;
                                      UPDATE accounts
                                        SET balance_minor = 999
                                        WHERE id=1;
                                      COMMIT;
SELECT balance_minor FROM accounts
  WHERE id=1;                -- 999   ★ CHANGED MID-TRANSACTION
COMMIT;
```
**A non-repeatable read**, permitted at READ COMMITTED. Two reads of the same row in one transaction returned different values.

```sql
-- REPEATABLE READ fixes this
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT balance_minor FROM accounts WHERE id=1;   -- 999
-- (another session updates it and commits)
SELECT balance_minor FROM accounts WHERE id=1;   -- 999 ★ stable
COMMIT;
```

**Step 4 — Isolation: the lost update, which READ COMMITTED permits.**

```sql
UPDATE accounts SET balance_minor = 100000 WHERE id=1;

-- session 1                          -- session 2
BEGIN;
SELECT balance_minor FROM accounts
  WHERE id=1;                -- 100000
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
         80000        ★ TWO withdrawals totalling ₹300, but only ₹200
                        was deducted. The first is LOST.
```
```sql
-- the fix that costs nothing: make the write atomic
UPDATE accounts SET balance_minor = balance_minor - 10000 WHERE id=1;
-- ★ read-modify-write in ONE statement. No lost update possible.
```

**Step 5 — Durability: measure the dial.**

```sql
\timing on
SHOW synchronous_commit;   -- on

CREATE TABLE d_test (id bigserial PRIMARY KEY, v text);

-- one commit per row, synchronous_commit = on
DO $$ BEGIN FOR i IN 1..2000 LOOP
  INSERT INTO d_test (v) VALUES ('x'); COMMIT;
END LOOP; END $$;
-- Time: 1,662 ms  ⇒ ~1,204 tps

SET synchronous_commit = off;
DO $$ BEGIN FOR i IN 1..2000 LOOP
  INSERT INTO d_test (v) VALUES ('x'); COMMIT;
END LOOP; END $$;
-- Time: 515 ms    ⇒ ~3,880 tps    ★ 3.2×
RESET synchronous_commit;
```

```bash
# ★ and check whether your storage is telling the truth about fsync
pg_test_fsync
```
```
fsync                              1,204.882 ops/sec     830.000 usecs/op
```
```
 ⇒ ~1,200 fsync/s is plausible for real durable storage.
 ⚠ IF YOU SEE 100,000+ ops/sec, the device is acknowledging fsync
   before the data is durable. Your D is a fiction.
```

**Step 6 — group commit makes durability affordable.**

```bash
pgbench -i -s 10 shop
pgbench -c 1   -j 1 -T 20 -N shop      # single client
pgbench -c 200 -j 8 -T 20 -N shop      # 200 clients
```
```
  -c 1   :  tps = 1,188   (1 fsync per commit)
  -c 200 :  tps = 18,412  (≈ 90 commits amortised per fsync)
```
**Durability costs 0.83 ms when idle and 0.05 ms per transaction under load.** The fsync is shared.

---

## Example 2 — production scenario

**The situation.** A fintech platform. A previous engineer "tuned for performance." Three months later, three problems surface in one week.

**Step 1 — audit the durability settings.**

```sql
SELECT name, setting, source, sourcefile, sourceline
FROM pg_settings
WHERE name IN ('synchronous_commit','fsync','full_page_writes',
               'wal_level','commit_delay','wal_sync_method')
ORDER BY name;
```
```
       name         | setting  |  source
--------------------+----------+------------
 commit_delay       | 0        | default
 fsync              | off      | ★ configuration file
 full_page_writes   | off      | ★ configuration file
 synchronous_commit | off      | ★ configuration file
 wal_level          | replica  | configuration file
 wal_sync_method    | fdatasync| default
```

**All three durability guarantees disabled**, on a system holding money.

```
 WHAT EACH ACTUALLY COSTS HERE:
  synchronous_commit = off
    ⇒ up to ~600 ms of COMMITTED transactions lost on a crash.
    ⇒ at 4,000 txn/s that is ~2,400 transactions. Consistent, but gone.
    ⇒ ★ and the CUSTOMER WAS TOLD THE PAYMENT SUCCEEDED.

  full_page_writes = off
    ⇒ TORN PAGES. A crash during an 8 KB write can leave half-old,
      half-new bytes, and WAL replay cannot repair a page it assumed
      was intact.
    ⇒ ★ CORRUPTION, not loss.

  fsync = off
    ⇒ pages may reach disk in any order. Recovery is not guaranteed
      to produce a valid database at all.
    ⇒ ★ UNRECOVERABLE CORRUPTION.
```

**Step 2 — the incident that exposed it.**

```
 A node was terminated by the cloud provider (a host failure) during
 a normal Tuesday afternoon.

 ON RESTART:
   ✓ the database started (it was lucky)
   ✗ 2,104 payments that returned 200 OK to the client did not exist
   ✗ 14 index pages failed checksum validation
   ✗ one heap page had a corrupt line pointer array

 RECONCILIATION AGAINST THE GATEWAY:
   SELECT count(*) FROM gateway_settlement_report g
   LEFT JOIN payments p ON p.gateway_ref = g.ref
   WHERE p.id IS NULL;
   ⇒ 2,104 charges taken, no record. ₹41,20,000 unaccounted.
```

**Step 3 — restore the guarantees, and measure the real cost.**

```sql
ALTER SYSTEM SET fsync = on;
ALTER SYSTEM SET full_page_writes = on;
ALTER SYSTEM SET synchronous_commit = on;
SELECT pg_reload_conf();   -- fsync and full_page_writes need a restart
```

```bash
pgbench -c 200 -j 8 -T 60 -N shop
```
```
 BEFORE (all off): tps = 22,104
 AFTER  (all on) : tps = 18,412       ★ −17%
```
**17%.** Not 3×, because at 200 concurrent clients **group commit amortises the fsync** across ~90 transactions. The "performance tuning" bought 17% and cost ₹41 lakh and a corrupt database.

**Step 4 — the Consistency finding, which was worse.**

```sql
-- what constraints exist on the money tables?
SELECT conrelid::regclass AS tbl, contype, count(*)
FROM pg_constraint
WHERE conrelid IN ('accounts'::regclass,'payments'::regclass,'transfers'::regclass)
GROUP BY 1,2 ORDER BY 1,2;
```
```
    tbl     | contype | count
------------+---------+-------
 accounts   | p       |     1
 payments   | p       |     1
 transfers  | p       |     1
```
**Three primary keys. Nothing else.** No `NOT NULL`, no `CHECK`, no `FOREIGN KEY`, no `UNIQUE`.

```sql
-- so: has the data already violated the rules nobody declared?
SELECT count(*) FILTER (WHERE balance_minor < 0)             AS negative_balances,
       count(*) FILTER (WHERE balance_minor IS NULL)         AS null_balances
FROM accounts;

SELECT count(*) AS payments_with_no_account
FROM payments p LEFT JOIN accounts a ON a.id = p.account_id WHERE a.id IS NULL;

SELECT count(*) AS duplicate_gateway_refs FROM (
  SELECT gateway_ref FROM payments GROUP BY 1 HAVING count(*) > 1) x;
```
```
 negative_balances | null_balances
-------------------+---------------
              412  |            88

 payments_with_no_account
--------------------------
                     1204

 duplicate_gateway_refs
------------------------
                    340       ★ 340 double-charges
```

```
 ★ THE POINT: A, I and D were configuration choices. C was never
   attempted at all. And C is the letter that had already silently
   failed 2,044 times before the crash ever happened.
```

**Step 5 — restore C, using Phase 3 and 4.**

```sql
-- clean first (each one is a business decision — case study 03)
INSERT INTO reconciliation_queue (reason, payload)
SELECT 'NEGATIVE_BALANCE', to_jsonb(a) FROM accounts a WHERE balance_minor < 0;
INSERT INTO reconciliation_queue (reason, payload)
SELECT 'ORPHAN_PAYMENT', to_jsonb(p) FROM payments p
LEFT JOIN accounts a ON a.id=p.account_id WHERE a.id IS NULL;

-- then constrain (Topic 28's NOT VALID pattern)
ALTER TABLE accounts ALTER COLUMN balance_minor SET NOT NULL;
ALTER TABLE accounts ADD CONSTRAINT ck_balance_nonneg
  CHECK (balance_minor >= 0 OR allows_overdraft) NOT VALID;

CREATE INDEX CONCURRENTLY idx_payments_account ON payments (account_id);
ALTER TABLE payments ADD CONSTRAINT fk_payments_account
  FOREIGN KEY (account_id) REFERENCES accounts(id) ON DELETE RESTRICT NOT VALID;
ALTER TABLE payments VALIDATE CONSTRAINT fk_payments_account;

CREATE UNIQUE INDEX CONCURRENTLY uq_payments_gateway_ref
  ON payments (gateway_ref) WHERE gateway_ref IS NOT NULL;
-- ★ this one alone makes the 340 double-charges impossible to repeat
```

**Step 6 — choose the isolation level deliberately.**

```sql
-- the balance transfer, at READ COMMITTED, has a lost-update window
-- ⇒ either make the write atomic:
UPDATE accounts SET balance_minor = balance_minor - $1
 WHERE id = $2 AND balance_minor >= $1;
-- ⇒ or raise the level for this transaction only:
BEGIN ISOLATION LEVEL REPEATABLE READ;
  …
COMMIT;   -- and handle serialization_failure with a retry (Topic 50)
```

**Step 7 — results.**

| | Before | After |
|---|---|---|
| `fsync` | off | **on** |
| `full_page_writes` | off | **on** |
| `synchronous_commit` | off | **on** |
| Throughput at 200 clients | 22,104 tps | **18,412 tps** (−17%) |
| Data-loss window | ~600 ms | **0** |
| Corruption risk on crash | **arbitrary** | none |
| Constraints on money tables | 3 (PKs) | **19** |
| Negative balances | 412 | **0, impossible** |
| Orphan payments | 1,204 | **0, impossible** |
| Duplicate charges | 340 | **0, impossible** |

**The 17% was the cheapest part of the fix.** The expensive part was C — which no configuration setting controls.

---

## Common mistakes

**1. Believing "the database guarantees consistency."**
- *Symptom:* invariant violations in a fully ACID database.
- *Engine-level why:* C means "your declared constraints are enforced." A rule you did not declare is not guaranteed.
- *Fix:* declare it. If it can't be a constraint, restructure so it can (Phase 4), or build another mechanism and a reconciliation job.

**2. `synchronous_commit = off` without knowing the window.**
- *Symptom:* committed transactions missing after a crash; customers charged with no record.
- *Engine-level why:* commit returns before the WAL is on disk; the walwriter flushes within ~600 ms.
- *Fix:* only for data you can regenerate. Never for money. And measure the real cost first — group commit often makes it ~17%, not 3×.

**3. `fsync = off` or `full_page_writes = off`.**
- *Symptom:* corruption, not just loss.
- *Engine-level why:* pages may reach disk out of order; a torn 8 KB write cannot be repaired by replay.
- *Fix:* never, outside a throwaway test database.

**4. Assuming SERIALIZABLE semantics at READ COMMITTED.**
- *Symptom:* lost updates and write skew in code that "looks obviously correct."
- *Fix:* know the default. Make read-modify-write atomic in one statement, or raise the level and handle retries (Topics 44, 50).

**5. Trusting fsync without testing the hardware.**
- *Symptom:* corruption after a power cut despite `fsync = on`.
- *Engine-level why:* consumer SSDs and some virtualised storage acknowledge fsync from a volatile cache.
- *Fix:* `pg_test_fsync`. 100,000+ ops/sec means the device is lying.

**6. Thinking durability on one node is enough.**
- *Symptom:* the instance is lost and so is the data.
- *Fix:* D on one node survives a crash, not a disk or a datacentre. Replication (Topic 62) and archiving (Topic 64) are separate concerns.

**7. Expecting atomicity to cover external effects.**
- *Symptom:* "charged but no order."
- *Fix:* A covers database work only. Reservations plus an outbox (Topics 39, 52).

---

## Hands-on proof

**PROVE IT #1 — atomicity.** (Example 1, step 1.)
**PROVE IT #2 — C is only your DDL.** (Example 1, step 2.)
**PROVE IT #3 — a non-repeatable read at the default level.** (Example 1, step 3.)
**PROVE IT #4 — a lost update at READ COMMITTED.** (Example 1, step 4.)
**PROVE IT #5 — the durability dial, timed.** (Example 1, step 5.)
**PROVE IT #6 — group commit amortises the fsync.** (Example 1, step 6.)

**PROVE IT #7 — audit your own durability settings.**
```sql
SELECT name, setting, source,
       CASE
         WHEN name='fsync' AND setting='off' THEN '★ CORRUPTION RISK'
         WHEN name='full_page_writes' AND setting='off' THEN '★ TORN PAGES'
         WHEN name='synchronous_commit' AND setting='off' THEN '⚠ DATA LOSS WINDOW'
         ELSE 'ok' END AS verdict
FROM pg_settings
WHERE name IN ('fsync','full_page_writes','synchronous_commit',
               'wal_level','wal_sync_method');
```

**PROVE IT #8 — audit your Consistency (the letter people skip).**
```sql
-- tables with nothing but a primary key
SELECT c.relname,
       count(con.oid) FILTER (WHERE con.contype='p') AS pks,
       count(con.oid) FILTER (WHERE con.contype='f') AS fks,
       count(con.oid) FILTER (WHERE con.contype='c') AS checks,
       count(con.oid) FILTER (WHERE con.contype='u') AS uniques
FROM pg_class c
LEFT JOIN pg_constraint con ON con.conrelid = c.oid
WHERE c.relkind='r' AND c.relnamespace='public'::regnamespace
GROUP BY c.oid, c.relname
HAVING count(con.oid) FILTER (WHERE con.contype <> 'p') = 0
ORDER BY c.relname;
-- ★ every table listed is trusting application code entirely.
--   For each, write down the invariants and ask which the engine could keep.
```

**PROVE IT #9 — does your storage tell the truth?**
```bash
pg_test_fsync -f /var/lib/postgresql/data/testfile
# > 100,000 ops/sec ⇒ ★ the write cache is volatile and lying.
#   Your durability is not real.
```

---

## The design decision framework

```
★ ATOMICITY — the engine's is perfect; yours is the boundary.
  ✓ one transaction per set of writes that must succeed together
  ✗ never an external call inside (Topic 39)
  ⇒ for external effects: reserve → call → confirm, plus an outbox (T52)

★ CONSISTENCY — this is YOUR job, and it is the letter that fails.
  FOR EVERY BUSINESS INVARIANT, ASK: "can the engine enforce this?"
    single-row rule        → CHECK
    presence               → NOT NULL
    uniqueness             → UNIQUE (partial if it applies to a subset)
    referential            → FOREIGN KEY (+ an index on the child!)
    no-two-rows-overlap    → EXCLUDE                        ★ (T24)
    derived value          → GENERATED ALWAYS AS … STORED
    cross-row / cross-table→ ① restructure so it CAN be a constraint
                             ② a trigger WITH an explicit lock
                             ③ SERIALIZABLE (T50)
                             ④ app logic + ★ a reconciliation job
  ⇒ ★ if you choose ④, the reconciliation job is NOT optional.

★ ISOLATION — choose per transaction, deliberately.
  READ COMMITTED (default) ✓ most reads; single-statement writes
  REPEATABLE READ          ✓ multi-statement reads that must agree;
                             read-modify-write over several statements
  SERIALIZABLE             ✓ invariants spanning rows that no
                             constraint can express (T50)
  ⇒ ★ AND the cheapest fix is usually neither: make the
    read-modify-write ONE atomic statement.
      UPDATE t SET n = n - 1 WHERE id=$1 AND n > 0

★ DURABILITY — a dial, with a window you must be able to state.
  synchronous_commit = on            ★ default. Money, orders, anything
                                       a customer was told succeeded.
                     = off           ✓ regenerable data ONLY: analytics
                                       events, caches, bulk loads.
                                       ⇒ ★ state the window: ~600 ms.
                     = remote_apply  ✓ when a read-your-writes guarantee
                                       on a replica is required
  fsync = off · full_page_writes = off
                     ⇒ ★ NEVER outside a disposable test database.
  ⇒ MEASURE BEFORE TRADING: group commit means the real cost at
    production concurrency is often ~15–20%, not 3×.
  ⇒ AND TEST THE HARDWARE: pg_test_fsync.

THE SIGNAL TO LOOK FOR:
  ① SELECT name, setting FROM pg_settings
     WHERE name IN ('fsync','full_page_writes','synchronous_commit');
     any 'off' → know exactly what was traded, and for what measured gain
  ② the constraint audit (PROVE IT #8)
     any table with only a PK → C is unenforced there
  ③ SHOW transaction_isolation;  and ask: does the code assume more?
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
Demonstrate each letter: (a) atomicity via a failed second statement, (b) that a "money conserved" invariant is *not* enforced and cannot be expressed as a `CHECK`, (c) a non-repeatable read at READ COMMITTED and its absence at REPEATABLE READ, (d) the throughput difference between `synchronous_commit` on and off. For (b), explain what schema change makes the invariant expressible.

### Exercise 2 — medium (apply it)
For each business rule, state which ACID letter it belongs to and the strongest available mechanism:
(a) "an order total equals the sum of its lines" · (b) "a user's email is unique" · (c) "two bookings for one room must not overlap" · (d) "a transfer moves money atomically" · (e) "the sum of all ledger entries is zero" · (f) "a committed payment survives a power cut" · (g) "two concurrent withdrawals cannot overdraw an account"

For any that cannot be a constraint, give the fallback and the reconciliation query.

### Exercise 3 — hard (production simulation)
A fintech platform runs with `fsync=off`, `full_page_writes=off`, `synchronous_commit=off`, and money tables that have primary keys and nothing else. A host failure produces: 2,104 payments that returned 200 OK but do not exist, 14 index pages failing checksum, and one corrupt heap page. A subsequent audit finds 412 negative balances, 1,204 orphan payments and 340 duplicate gateway refs.

(a) For each of the three settings, state exactly what it risks — distinguishing *loss* from *corruption*.
(b) Explain why the 2,104 missing payments are a durability failure while the 340 duplicates are a consistency failure.
(c) Measure the real throughput cost of restoring all three, and explain why it is ~17% rather than ~3× at production concurrency.
(d) Write the audit query that finds every table trusting application code for consistency.
(e) Design the constraints for the money tables. Identify which invariant still cannot be a constraint, and give the mechanism plus the reconciliation query.
(f) The balance transfer is written as `SELECT` then `UPDATE` at READ COMMITTED. Show the lost-update interleaving, and give two fixes — one that costs nothing and one that changes the isolation level.
(g) `pg_test_fsync` reports 180,000 ops/sec. What does that mean, and what do you do?
(h) Write the deployment plan: which changes need a restart, which are online, and in what order.

---

## Mental model checkpoint

1. Name the four properties and the mechanism that keeps each.
2. Why is C different in kind from A, I and D? What does the engine actually promise?
3. Why does PostgreSQL need no undo log? What does it pay instead?
4. At what exact moment does a transaction become durable, and what is *not* durable at that moment?
5. Name the five `synchronous_commit` settings and the guarantee each adds.
6. Why is `fsync = off` categorically different from `synchronous_commit = off`?
7. Why is durability cheap under high concurrency and expensive when idle?

---

## Quick reference card

| Letter | Guarantee | Mechanism | Fails when |
|---|---|---|---|
| **A** | all or nothing | WAL + `pg_xact` | your boundary is wrong |
| **C** | valid → valid | ★ **your constraints** | you didn't declare the rule |
| **I** | no interference | MVCC + locks + SSI | the default level is weaker than assumed |
| **D** | survives a crash | WAL fsync | `synchronous_commit = off`, or lying storage |

**`synchronous_commit`**

| Setting | Guarantee | Cost |
|---|---|---|
| `off` | ~600 ms loss window, **no corruption** | 2–3× faster (idle) |
| `local` | this server's disk | — |
| `on` ★ | this server's disk, fsync confirmed | default |
| `remote_write` | a replica has received it | + network |
| `remote_apply` | a replica has applied it | + network + apply |

**Never:** `fsync = off` · `full_page_writes = off` — these risk **corruption**, not loss.

**The C checklist** — for every invariant: `CHECK` · `NOT NULL` · `UNIQUE` · `FOREIGN KEY` · `EXCLUDE` · `GENERATED`. If none fits: restructure → trigger+lock → `SERIALIZABLE` → app + **reconciliation job**.

**The cheapest isolation fix:** make read-modify-write one statement.
```sql
UPDATE t SET n = n - 1 WHERE id = $1 AND n > 0;
```

**Test your hardware:** `pg_test_fsync`. Over 100,000 ops/sec means the cache is lying.

---

## When would I use this at work?

1. **Auditing an inherited database.** Two queries — the durability settings and the constraint audit — tell you within a minute which ACID guarantees you actually have. It is common to find that C was never attempted.

2. **When someone proposes `synchronous_commit = off`.** You can state the exact window (~600 ms), measure the real gain at production concurrency (often ~17%, not 3×), and ask which specific data is regenerable.

3. **Explaining a data-integrity incident.** Being able to say "this was a C failure, not an ACID failure — the engine did exactly what we told it to" moves the conversation from blame to the missing constraint.

---

## Connected topics

**Understand before this:** 39 (the transaction boundary), 24 (constraints — the whole of C), 07 (the buffer pool, and why the heap isn't durable at commit).

**This unlocks:**
- **41–42** — WAL and recovery: A and D implemented
- **43–44** — the concurrency anomalies and the isolation dial
- **45–46** — locks and MVCC: I implemented
- **50** — SERIALIZABLE, for the invariants no constraint can express
- **62, 64** — replication and PITR: durability beyond one node
