# 50 — Serializable Snapshot Isolation (SSI)
## Phase: Transactions & Concurrency

---

## ELI5 — The Simple Analogy

A librarian who lets everyone read freely, then **checks at the exit** whether the visits could have happened one at a time.

Everyone gets a photocopy of the catalogue when they arrive (that's the snapshot — Topic 46). They wander, read shelves, and change some cards. Nobody waits for anybody.

But the librarian keeps a small notebook: **"Meera looked at shelf 12. Arjun looked at shelf 12 too. Arjun then changed a card on shelf 12."**

At the exit, the librarian asks one question: *"Could I write down an order — Meera then Arjun, or Arjun then Meera — that produces exactly what happened?"*

Usually yes, and everyone leaves. But sometimes the notebook shows a shape where **no ordering works**: Meera read something Arjun later changed, *and* Arjun read something Meera later changed. Whichever order you claim, one of them acted on information that the ordering says didn't exist yet. That's a contradiction, and the librarian sends one person back out to start over.

Two things make this remarkable:

1. **Nobody ever waited.** Not once. The checking is bookkeeping, not blocking.
2. ★ **The librarian doesn't need to catch every suspicious pattern — only one specific shape.** That's the theorem SSI rests on, and it's why the overhead is small enough to be practical.

---

## Where this fits in the big picture

```
   44 isolation levels — "SERIALIZABLE = snapshot isolation + read tracking"
   43 write skew       — the anomaly that survives everything else
   49 optimistic concurrency — the same idea, done by hand
                          │
                          ▼
        ┌──────────────────────────────────────────────┐
        │ 50 SSI ← YOU ARE HERE                        │
        │ what "read tracking" actually is             │
        └────────────────────┬─────────────────────────┘
                             ▼
              51 distributed transactions
              68 CAP and consistency models
```

Topic 44 gave you the *what*: SERIALIZABLE prevents write skew and costs retries. **This topic is the *how* and the *why it costs what it costs*** — which is what you need in order to tune it or decide against it.

---

## What is this?

**Serializable Snapshot Isolation** is PostgreSQL's implementation of true serializability (since 9.1). It gives the strongest correctness guarantee in the SQL standard:

> **Any set of concurrent SERIALIZABLE transactions that all commit produces a result identical to running them one at a time, in some order.**

Not "almost." Not "for the anomalies we listed." **Any anomaly, including ones nobody has named.**

The remarkable part is the method. Classic serializability (two-phase locking) achieves this by **blocking**: readers take shared locks, writers take exclusive ones, everybody waits. SSI achieves it by **never blocking anyone and detecting afterwards**:

| | Two-phase locking (S2PL) | ★ SSI |
|---|---|---|
| Readers block writers | yes | ★ **no** |
| Writers block readers | yes | ★ **no** |
| Read-only transactions | take locks, cause blocking | ★ nearly free |
| Failure mode | blocking, deadlocks | ★ aborts (`40001`), retries |
| Overhead when uncontended | lock acquisition on every read | ★ predicate-lock bookkeeping |
| Wasted work on conflict | none | ★ the whole transaction |

**It is optimistic concurrency control (Topic 49), applied to entire read predicates rather than single rows** — and that's exactly why it catches write skew, which no per-row scheme can.

---

## Why does it matter for a backend developer?

```
 ★ THREE PRACTICAL REASONS.

 ① IT IS THE ONLY GENERAL ANSWER TO WRITE SKEW.
    "the count over this set must not exceed N"
    "the sum must not exceed the credit limit"
    "at least one must remain"
    ⇒ no UNIQUE, no EXCLUDE, no CHECK can express these, because
      the constraint is over a SET the transaction is adding to.
    ⇒ your options are: a hot counter row (case study 01's problem)
      or SERIALIZABLE.

 ② KNOWING THE COST MODEL IS THE DIFFERENCE BETWEEN 2% AND 40001
    STORMS.
    ★ Most "SERIALIZABLE is too slow" conclusions are actually
      PREDICATE LOCK ESCALATION producing FALSE POSITIVES —
      a tuning problem, not a design one (Topic 44's incident).

 ③ IT LETS YOU WRITE THE OBVIOUS CODE.
    Under SERIALIZABLE you may write:
      SELECT count(*) …; if (count < N) INSERT …;
    and it is CORRECT. Under any other level it is a bug.
    ⇒ for genuinely complex invariants, that is worth a lot more
      than the retry rate costs.
```

---

## The physical reality

### The theorem SSI is built on

```
 ★ SSI DOES NOT DETECT NON-SERIALIZABLE EXECUTIONS DIRECTLY.
   It detects ONE STRUCTURAL PATTERN that must be present in any
   of them.

 THE DEPENDENCY GRAPH. Between two concurrent transactions:
   wr-dependency  T1 writes, T2 reads it       (write→read)
   ww-dependency  T1 writes, T2 overwrites     (write→write)
   ★ rw-ANTIDEPENDENCY  T1 READS, T2 then WRITES what T1 read
                        ⇒ T1 must come BEFORE T2 in any serial
                          order, because T1 didn't see T2's write

 ★ THE THEOREM (Cahill, Röhm & Fekete, 2008):
   In snapshot isolation, EVERY non-serializable execution contains
   a cycle in the dependency graph, and in that cycle there exist
   TWO CONSECUTIVE rw-ANTIDEPENDENCY EDGES:

        Tin  ──rw──►  Tpivot  ──rw──►  Tout

   ⇒ SO YOU DO NOT NEED TO BUILD OR SEARCH THE WHOLE GRAPH.
     You only need to notice when one transaction has BOTH an
     incoming and an outgoing rw-edge. That transaction is the
     ★ PIVOT.

 ⇒ ★ AND THAT IS THE WHOLE ALGORITHM: track rw-edges; when a
   transaction acquires both directions, abort somebody.

 ★ THE PRICE OF THE SIMPLIFICATION: this condition is NECESSARY
   but NOT SUFFICIENT. Some executions with a pivot ARE actually
   serializable.
 ⇒ ★ SSI THEREFORE PRODUCES FALSE POSITIVES BY DESIGN. It never
   allows an anomaly (no false negatives), but it sometimes aborts
   a transaction that would have been fine.
 ⇒ THIS IS A DELIBERATE TRADE: cheap detection, occasional
   unnecessary retries. Not a bug.
```

### `SIReadLock` — the thing that is not a lock

```
 ★ SIReadLock IS A READ MARKER. IT BLOCKS NOTHING. EVER.
   The name is unfortunate. Nothing ever waits on one.

 WHAT IT RECORDS: "transaction T read this tuple / page / relation."
 WHY: so that when someone later WRITES that tuple, PostgreSQL can
      look up who read it and create the rw-edge.

 GRANULARITY LADDER — and this is where the cost lives:
   tuple    most precise, most entries
   page     coarser
   relation ★ "T read the whole table"

 ESCALATION HAPPENS AUTOMATICALLY UNDER MEMORY PRESSURE:
   max_pred_locks_per_page        (2)   tuples→page above this
   max_pred_locks_per_relation    (-2)  pages→relation above this
                                        (-2 ⇒ max_pred_locks_per_
                                         transaction / 2)
   max_pred_locks_per_transaction (64)  ★ the overall budget

 ⇒ ★ THE FAILURE MODE THAT MATTERS:
   escalation to RELATION level means "this transaction read the
   entire table." Now ANY write to that table by anyone creates an
   rw-edge with it.
   ⇒ FALSE POSITIVES EXPLODE.
   ⇒ THIS IS WHY A READ-ONLY REPORTING ENDPOINT CAN PRODUCE 98% OF
     YOUR 40001s (Topic 44's incident). It scanned 2,000 tuples
     with a budget of 64.

 SEE IT:
   SELECT locktype, relation::regclass, page, tuple, count(*)
     FROM pg_locks WHERE mode='SIReadLock'
    GROUP BY 1,2,3,4 ORDER BY 5 DESC;
   ★ locktype='relation' with mode='SIReadLock' ⇒ escalation
     happened ⇒ expect false positives ⇒ raise the budget.
```

### The memory that outlives the transaction

```
 ★ SSI MUST REMEMBER COMMITTED TRANSACTIONS.

 Why: T1 commits. Later, T2 writes something T1 read. The rw-edge
 T1→T2 still matters, because T1 might be part of a cycle with a
 transaction that hasn't finished yet.
 ⇒ SSI keeps information about committed serializable transactions
   until no active transaction could still form a cycle with them.

 ⇒ CONSEQUENCE ①: ★ A SINGLE LONG-RUNNING SERIALIZABLE TRANSACTION
   PREVENTS CLEANUP FOR EVERYONE. All the committed-transaction
   state piles up behind it.
 ⇒ CONSEQUENCE ②: if that memory is exhausted, PostgreSQL SUMMARISES
   old transactions into a coarse "everything before this point"
   marker — which is safe but ★ generates more false positives.
 ⇒ CONSEQUENCE ③: you can see the pressure:
     SELECT count(*) FROM pg_locks WHERE mode='SIReadLock';
   growing steadily with no long transaction visible ⇒ look for a
   long SERIALIZABLE one specifically.

 ⇒ ★ THE OPERATIONAL RULE: SERIALIZABLE TRANSACTIONS MUST BE SHORT.
   Shorter than at any other isolation level, because the cost is
   borne by the whole cluster, not just by you.
```

### Why `READ ONLY DEFERRABLE` is free

```
 ★ A READ-ONLY TRANSACTION CANNOT BE THE PIVOT'S SUCCESSOR.
   It never writes, so it can never have an OUTGOING rw-edge.
   ⇒ it can only ever be Tin, never Tpivot.
   ⇒ SSI's tracking for it is much cheaper, and it can often be
     skipped entirely.

 AND `DEFERRABLE` GOES FURTHER:
 BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE READ ONLY DEFERRABLE;
   ⇒ the transaction WAITS at the start until it can obtain a
     "safe snapshot" — one where no concurrent transaction could
     possibly create a conflict with it.
   ⇒ ★ ONCE IT STARTS, IT HAS A MATHEMATICAL GUARANTEE OF NEVER
     GETTING 40001, AND CONTRIBUTES ZERO SSI OVERHEAD.
 ⇒ THE ONLY COST: it may wait to begin.
 ⇒ ★ FOR ANY LONG REPORT, THIS IS STRICTLY THE BEST OPTION IN
   POSTGRESQL. Perfectly consistent, zero failure risk, zero cost
   to other transactions. Almost nobody uses it.
```

### The two abort messages, and what each means

```
 ① ERROR: could not serialize access due to read/write dependencies
           among transactions
    DETAIL: Reason code: Canceled on identification as a pivot,
            during write.
    ⇒ ★ SSI. A dangerous structure was found. This is the write-skew
      catch. Retryable — and the retry usually succeeds because the
      other transaction has now committed.

 ② ERROR: could not serialize access due to concurrent update
    ⇒ ★ NOT SSI. This is the ordinary snapshot-isolation
      first-updater-wins rule, identical at REPEATABLE READ
      (Topic 44). It means you tried to update a row that changed
      after your snapshot.

 ⇒ BOTH ARE SQLSTATE 40001. BOTH ARE RETRYABLE.
 ⇒ ★ BUT THEY MEAN DIFFERENT THINGS: ② says "row contention,"
   ① says "predicate conflict." Log the DETAIL, not just the
   SQLSTATE — it tells you whether to look at row design or at
   query breadth.

 THE REASON CODES YOU'LL SEE IN ①:
   "Canceled on identification as a pivot, during write."
   "Canceled on identification as a pivot, during commit attempt."
   "Canceled on conflict out to pivot <xid>, during read."
   ★ "during commit attempt" is the expensive one — the transaction
     did ALL its work and was killed at the end.
```

---

## How it works — step by step

### The doctors example, traced through SSI

```
 on_call: alice=true, bob=true.  INVARIANT: at least one on call.

 ┌─ T1 ─────────────────────────┬─ T2 ─────────────────────────────┐
 │ BEGIN ISOLATION LEVEL        │                                  │
 │   SERIALIZABLE;              │                                  │
 │ SELECT count(*) FROM on_call │                                  │
 │  WHERE is_on_call;  → 2      │                                  │
 │ ★ SIReadLock recorded on     │                                  │
 │   the alice tuple AND the    │                                  │
 │   bob tuple                  │                                  │
 │                              │ BEGIN … SERIALIZABLE;            │
 │                              │ SELECT count(*) … → 2            │
 │                              │ ★ SIReadLock on BOTH tuples too  │
 │ UPDATE on_call               │                                  │
 │  SET is_on_call=false        │                                  │
 │  WHERE doctor_id='alice';    │                                  │
 │ ★ writing a tuple T2 READ    │                                  │
 │   ⇒ rw-EDGE  T2 ──rw──► T1   │                                  │
 │   (T2 must come before T1)   │                                  │
 │                              │ UPDATE on_call                   │
 │                              │  SET is_on_call=false            │
 │                              │  WHERE doctor_id='bob';          │
 │                              │ ★ writing a tuple T1 READ        │
 │                              │   ⇒ rw-EDGE  T1 ──rw──► T2       │
 │                              │                                  │
 │  ★ NOW: T2──rw──►T1──rw──►T2 │                                  │
 │    BOTH T1 AND T2 HAVE AN    │                                  │
 │    INCOMING AND AN OUTGOING  │                                  │
 │    rw-EDGE. A PIVOT EXISTS.  │                                  │
 │                              │                                  │
 │ COMMIT;  ✓                   │                                  │
 │ ⇒ ★ the FIRST committer      │                                  │
 │   succeeds                   │ COMMIT;                          │
 │                              │ ★ ERROR: could not serialize     │
 │                              │   access due to read/write       │
 │                              │   dependencies among             │
 │                              │   transactions                   │
 │                              │ DETAIL: Reason code: Canceled    │
 │                              │   on identification as a pivot,  │
 │                              │   during commit attempt.         │
 └──────────────────────────────┴──────────────────────────────────┘

 ★ NOTE THE TIMING: T2 did all its work and was killed AT COMMIT.
   SSI cannot know earlier — the dangerous structure only became
   complete when T2 issued its UPDATE, and even then one of them
   might have rolled back.
 ⇒ THIS IS WHY THE RETRY MUST RE-RUN THE WHOLE TRANSACTION, and
   why long SERIALIZABLE transactions are expensive to lose.
```

### What SSI costs, measured

```
 THE OVERHEAD HAS THREE PARTS:

 ① PREDICATE LOCK ACQUISITION — on every tuple/page you READ
    ⇒ scales with ROWS READ, not rows written
    ⇒ ★ a wide SELECT is expensive even if it writes nothing
 ② CONFLICT CHECKING — on every WRITE, look up who read this tuple
    ⇒ scales with writes × readers of that data
 ③ COMMIT-TIME VALIDATION — check for the dangerous structure
    ⇒ cheap, but it is where aborts surface

 MEASURED, single-row-update workload, 50 clients:
   READ COMMITTED       18,412 tps   0 aborts
   REPEATABLE READ       6,880 tps   2,214 aborts / 30s
   SERIALIZABLE          2,104 tps   4,882 aborts / 30s

 MEASURED, the same workload spread over 100,000 rows:
   READ COMMITTED       19,204 tps   0 aborts
   SERIALIZABLE       ★ 16,880 tps   ★ 41 aborts / 30s
   ⇒ ★ 12% OVERHEAD, NOT 88%.

 ⇒ ★ THE HEADLINE NUMBER PEOPLE QUOTE FOR "SERIALIZABLE IS SLOW"
   IS MEASURED ON A HOT ROW, WHERE EVERY ISOLATION LEVEL IS SLOW.
   On a normally-distributed workload the overhead is modest.
```

---

## Concept breakdown

```
WHAT SSI GUARANTEES
└── ★ any set of committed SERIALIZABLE transactions is equivalent
     to SOME serial execution — for EVERY anomaly, named or not

HOW — WITHOUT BLOCKING ANYONE
├── snapshot isolation, unchanged (Topic 46)
├── ★ PLUS: track what each transaction READ (SIReadLock markers)
├── ★ PLUS: on every write, create rw-antidependency edges
└── ★ PLUS: abort when a transaction has BOTH an incoming and an
     outgoing rw-edge — the PIVOT

★ THE THEOREM
   every non-serializable snapshot-isolation execution contains
   two CONSECUTIVE rw-edges ⇒ detect only that, not the full graph
   ⇒ ★ necessary but not sufficient ⇒ FALSE POSITIVES BY DESIGN
   ⇒ never a false NEGATIVE — no anomaly ever slips through

SIReadLock — ★ NOT A LOCK. NOTHING EVER WAITS ON ONE.
├── granularity: tuple → page → relation
├── ★ ESCALATES under memory pressure
└── ★ relation-level = "read the whole table" = false-positive storm
     ⇒ raise max_pred_locks_per_transaction (default 64 is LOW)

MEMORY THAT OUTLIVES THE TRANSACTION
└── ★ committed-transaction state is kept until no cycle is possible
     ⇒ ONE long SERIALIZABLE transaction blocks cleanup for EVERYONE
     ⇒ SERIALIZABLE transactions must be SHORT

★ READ ONLY DEFERRABLE — the free lunch
   a read-only txn can never be a pivot's successor
   DEFERRABLE waits for a provably safe snapshot
   ⇒ ★ zero 40001 risk, zero overhead to others, perfect consistency
   ⇒ the best option for any long report. Almost nobody uses it.

THE TWO 40001 MESSAGES — same SQLSTATE, different meaning
├── "read/write dependencies among transactions" ⇒ ★ SSI, predicate
└── "concurrent update"                          ⇒ row contention
     ⇒ ★ LOG THE DETAIL, not just the code

THE COST, HONESTLY
├── on a hot row:        88% slower  ← ★ but so is everything
├── ★ spread workload:   ~12% slower
└── most "too slow" verdicts are escalation false positives
```

---

## Diagrams

**Diagram 1 — big picture: the dangerous structure**

```
       THE ONLY PATTERN SSI LOOKS FOR

              rw                    rw
    ┌─────┐  ────►  ┌────────┐  ────►  ┌──────┐
    │ Tin │         │ Tpivot │         │ Tout │
    └─────┘         └────────┘         └──────┘
       │                 ▲                 │
       │                 │                 │
       │        ★ HAS BOTH AN INCOMING     │
       │          AND AN OUTGOING          │
       │          rw-EDGE                  │
       │                                   │
    reads data                        writes data
    Tpivot later                      Tpivot read
    writes

 ★ rw-EDGE (antidependency): T1 READS something, T2 later WRITES it.
   ⇒ T1 must precede T2 in any serial order, because T1 did not
     see T2's write.

 ★ WHY TWO CONSECUTIVE rw-EDGES:
   the theorem says every non-serializable execution under snapshot
   isolation contains a cycle, and every such cycle contains this
   shape. So detecting the shape catches every anomaly.
 ★ AND WHY FALSE POSITIVES: the shape can also appear in
   executions that ARE serializable. SSI aborts those too —
   deliberately, because checking properly would cost more than
   the occasional retry.

 ⇒ IN THE 2-TRANSACTION CASE, Tin AND Tout ARE THE SAME
   TRANSACTION, and the cycle is just T1⇄T2. That's write skew.
```

**Diagram 2 — data flow: predicate-lock escalation and false positives**

```
 ✓ TUPLE-LEVEL TRACKING — precise, conflicts are REAL
 ┌──────────────────────────────────────────────────────────────┐
 │ SELECT * FROM invoices WHERE org_id = 42;   -- 30 rows       │
 │                                                               │
 │  SIReadLocks:  tuple(0,4) tuple(0,7) tuple(1,2) … × 30       │
 │                                                               │
 │  another txn: INSERT INTO invoices (org_id) VALUES (99);     │
 │    ⇒ does tuple X have a reader?  NO.                        │
 │    ⇒ ★ NO EDGE. No conflict. Both commit.                    │
 └──────────────────────────────────────────────────────────────┘

 ✗ ESCALATED TO RELATION — every write now conflicts
 ┌──────────────────────────────────────────────────────────────┐
 │ SELECT * FROM invoices ORDER BY issued_at DESC LIMIT 50      │
 │   OFFSET 4000;                        -- ★ scans 4,050 rows  │
 │                                                               │
 │  budget: max_pred_locks_per_transaction = 64                 │
 │  4,050 tuples ≫ 64                                           │
 │    ⇒ tuples ──► pages ──► ★ RELATION                         │
 │  SIReadLocks:  relation(invoices)                            │
 │                = "this transaction read the ENTIRE table"    │
 │                                                               │
 │  another txn: INSERT INTO invoices (org_id) VALUES (99);     │
 │    ⇒ does the relation have a reader?  ★ YES.                │
 │    ⇒ ★ rw-EDGE CREATED — for a row the reader never saw      │
 │    ⇒ pivot forms ⇒ ★ 40001 ON A READ-ONLY ENDPOINT           │
 │                                                               │
 │  MEASURED: 184,220 false-positive 40001/hour on GET /invoices│
 └──────────────────────────────────────────────────────────────┘
        ★ THE FIX IS A SETTING, NOT A REDESIGN:
          ALTER SYSTEM SET max_pred_locks_per_transaction = 512;
          (and narrow the query — fewer rows read, fewer edges)
```

**Diagram 3 — before/after: where SERIALIZABLE belongs**

```
 ✗ BEFORE — SERIALIZABLE globally
 ┌───────────────────────────────────────────────────────────────┐
 │ default_transaction_isolation = 'serializable'                │
 │                                                                │
 │ GET  /invoices      SSI tracking, ★ escalation, 184k 40001/hr │
 │ GET  /customers/:id SSI tracking, ★ 31k 40001/hr              │
 │ POST /invoices      SSI tracking  ✓ genuinely needs it        │
 │ GET  /health        SSI tracking  ✗ pure waste                │
 │                                                                │
 │ 1,140 rps · p99 2,400 ms · 4.1% 5xx                           │
 └───────────────────────────────────────────────────────────────┘

 ✓ AFTER — per transaction, with the right variant each time
 ┌───────────────────────────────────────────────────────────────┐
 │ default_transaction_isolation = 'read committed'              │
 │                                                                │
 │ GET  /invoices      ★ REPEATABLE READ READ ONLY               │
 │                       consistent pagination, ★ zero SSI cost  │
 │ GET  /reports/daily ★ SERIALIZABLE READ ONLY DEFERRABLE       │
 │                       perfect consistency, ★ zero 40001 risk  │
 │ POST /invoices      ★ SERIALIZABLE + retry loop               │
 │                       (credit limit = aggregate over a set —  │
 │                        no constraint can express it)          │
 │ POST /cart/add      READ COMMITTED + atomic UPDATE            │
 │ GET  /health        READ COMMITTED                            │
 │                                                                │
 │ 8,050 rps · p99 190 ms · 1,840 40001/hr (★ 99.2% fewer)       │
 │ max_pred_locks_per_transaction = 512                          │
 └───────────────────────────────────────────────────────────────┘
```

---

## Example 1 — basic

```sql
CREATE TABLE on_call (
  doctor_id text PRIMARY KEY,
  is_on_call boolean NOT NULL
);
INSERT INTO on_call VALUES ('alice',true),('bob',true);
```

**Reproduce write skew at REPEATABLE READ, then watch SSI catch it.**
```sql
-- session 1                          -- session 2
BEGIN ISOLATION LEVEL SERIALIZABLE;
SELECT count(*) FROM on_call
  WHERE is_on_call;        -- 2
                                      BEGIN ISOLATION LEVEL SERIALIZABLE;
                                      SELECT count(*) FROM on_call
                                        WHERE is_on_call;   -- 2
UPDATE on_call SET is_on_call=false
  WHERE doctor_id='alice';
                                      UPDATE on_call SET is_on_call=false
                                        WHERE doctor_id='bob';
COMMIT;                    -- ✓ first committer wins
                                      COMMIT;
```
```
ERROR:  could not serialize access due to read/write dependencies among transactions
DETAIL:  Reason code: Canceled on identification as a pivot, during commit attempt.
HINT:  The transaction might succeed if retried.
```
```sql
SELECT count(*) FROM on_call WHERE is_on_call;
```
```
 count
-------
     1        ★ the invariant held.
```

**See the predicate locks that made it possible.**
```sql
-- with a SERIALIZABLE transaction open in another session:
SELECT locktype, relation::regclass AS rel, page, tuple, pid, mode
  FROM pg_locks WHERE mode='SIReadLock' ORDER BY pid;
```
```
 locktype | rel     | page | tuple |  pid  |    mode
----------+---------+------+-------+-------+------------
 tuple    | on_call |    0 |     1 | 41202 | SIReadLock
 tuple    | on_call |    0 |     2 | 41202 | SIReadLock
 tuple    | on_call |    0 |     1 | 41288 | SIReadLock
 tuple    | on_call |    0 |     2 | 41288 | SIReadLock
   ★ tuple-level. Both transactions read both rows. Precise.
   ★ nothing is waiting on any of these — they are markers.
```

**Force escalation and see the granularity change.**
```sql
CREATE TABLE invoices (
  id bigserial PRIMARY KEY, org_id bigint NOT NULL, total_minor bigint NOT NULL);
INSERT INTO invoices (org_id, total_minor)
SELECT (random()*100)::bigint, (random()*100000)::bigint
  FROM generate_series(1,200000);
VACUUM ANALYZE invoices;

SHOW max_pred_locks_per_transaction;   -- 64

-- session 1
BEGIN ISOLATION LEVEL SERIALIZABLE;
SELECT count(*) FROM invoices;         -- reads all 200,000 rows

-- session 2
SELECT locktype, relation::regclass, count(*)
  FROM pg_locks WHERE mode='SIReadLock' GROUP BY 1,2;
```
```
 locktype | relation | count
----------+----------+-------
 relation | invoices |     1        ★ ESCALATED to relation level
```
```sql
-- session 2 — now ANY write to invoices conflicts with session 1
BEGIN ISOLATION LEVEL SERIALIZABLE;
INSERT INTO invoices (org_id, total_minor) VALUES (999, 100);
COMMIT;
-- session 1
UPDATE invoices SET total_minor = total_minor + 1 WHERE id = 1;
COMMIT;
```
```
ERROR:  could not serialize access due to read/write dependencies among transactions
   ★ a conflict with a row session 1 could not possibly care about.
     A FALSE POSITIVE caused by escalation.
```

**Prove `READ ONLY DEFERRABLE` never fails.**
```sql
-- run this while heavy concurrent writes are happening
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE READ ONLY DEFERRABLE;
SELECT count(*), sum(total_minor) FROM invoices;
COMMIT;
-- ★ it may pause briefly at BEGIN, then runs with a mathematical
--   guarantee of no 40001, and adds no SSI cost for other
--   transactions.
```

**Distinguish the two `40001` messages.**
```sql
-- ① SSI predicate conflict
-- ERROR: could not serialize access due to read/write dependencies
--        among transactions
-- ② ordinary first-updater-wins (also at REPEATABLE READ)
BEGIN ISOLATION LEVEL SERIALIZABLE;
UPDATE invoices SET total_minor = 1 WHERE id = 1;
-- (another session updates and commits id=1 first)
-- ERROR: could not serialize access due to concurrent update
--   ★ same SQLSTATE 40001, different DETAIL, different cause.
```

**Measure the real overhead — spread, not hot.**
```bash
cat > /tmp/spread.sql <<'EOF'
\set id random(1, 200000)
BEGIN;
SELECT total_minor FROM invoices WHERE id = :id;
UPDATE invoices SET total_minor = total_minor + 1 WHERE id = :id;
COMMIT;
EOF
for lvl in "read committed" "serializable"; do
  echo "== $lvl"
  PGOPTIONS="-c default_transaction_isolation='$lvl'" \
    pgbench -f /tmp/spread.sql -c 50 -j 4 -T 30 shop 2>&1 | grep -E 'tps|failed'
done
```
```
 == read committed
 tps = 19,204.8   failed = 0
 == serializable
 tps = 16,880.2   failed = 41       ★ 12% overhead, 0.0004% aborts
```
```bash
# now the same thing on ONE row
sed 's/random(1, 200000)/random(1,1)/' /tmp/spread.sql > /tmp/hot.sql
for lvl in "read committed" "serializable"; do
  echo "== $lvl"
  PGOPTIONS="-c default_transaction_isolation='$lvl'" \
    pgbench -f /tmp/hot.sql -c 50 -j 4 -T 30 shop 2>&1 | grep -E 'tps|failed'
done
```
```
 == read committed
 tps = 18,412.4   failed = 0
 == serializable
 tps =  2,104.7   failed = 4,882    ★ 88% overhead
 ⇒ ★ SAME ISOLATION LEVEL. THE DIFFERENCE IS ENTIRELY CONTENTION.
```

---

## Example 2 — production scenario

**The situation.** A subscription billing platform. Each organisation has a plan with a seat limit. The rule:

> *An organisation may not have more active seats than its plan allows.*

Seats are added by three independent paths: the admin UI, a SCIM directory sync, and a self-serve invite link. After a growth push:

```
 organisations over their seat limit    ★ 412 of 8,904 (4.6%)
 worst offender                         plan limit 25, actual 61
 revenue impact                          ~₹18 lakh/year of unbilled seats
 the check in code                      present, correct-looking, in all 3 paths
```

**Step 1 — why the obvious fixes don't apply.**

```js
// all three paths do this, and it looks right
const { rows } = await db.query(
  `SELECT count(*) AS used FROM seats WHERE org_id=$1 AND status='active'`, [orgId]);
const { rows: [plan] } = await db.query(
  'SELECT seat_limit FROM organisations WHERE id=$1', [orgId]);
if (Number(rows[0].used) >= plan.seat_limit) throw new AppError('SEAT_LIMIT_REACHED');
await db.query(`INSERT INTO seats (org_id, user_id, status)
                VALUES ($1,$2,'active')`, [orgId, userId]);
```

```
 ★ TEXTBOOK WRITE SKEW (Topic 43): read an AGGREGATE over a set,
   then INSERT into that set.

 THE USUAL FIXES, AND WHY EACH FAILS HERE:
   ✗ UNIQUE          — nothing is duplicated. Every seat is distinct.
   ✗ EXCLUDE         — expresses "no two rows overlap", not
                       "no more than N rows exist"
   ✗ CHECK           — a row-level constraint cannot see the other
                       rows in the table
   ✗ REPEATABLE READ — ★ the three transactions insert DIFFERENT
                       rows; snapshot isolation sees no conflict
                       (Topic 43 proves this)
   ✗ a trigger with a count — ★ the trigger runs inside the same
                       transaction and sees the same snapshot.
                       It is the identical bug, moved.

 ⇒ ★ THE ONLY TWO REAL OPTIONS:
   ① a per-org counter row both paths must UPDATE (materialises
      the conflict — Topic 43)
   ② SERIALIZABLE
```

**Step 2 — evaluate the counter-row option first.**

```sql
ALTER TABLE organisations ADD COLUMN active_seats integer NOT NULL DEFAULT 0;
ALTER TABLE organisations
  ADD CONSTRAINT ck_seats_within_limit CHECK (active_seats <= seat_limit);

-- every path must now do:
BEGIN;
UPDATE organisations SET active_seats = active_seats + 1 WHERE id=$1;
-- ★ CHECK fires here if it would exceed
INSERT INTO seats (org_id, user_id, status) VALUES ($1,$2,'active');
COMMIT;
```

```
 ★ THIS WORKS, AT READ COMMITTED, WITH NO RETRIES. Evaluate it
   honestly:

 ✓ correct, enforced by the database, unbypassable
 ✓ no isolation change, no retry loop
 ✓ the count is also now a cheap read (no aggregate)
 ✗ ★ every seat change for an org serialises on ONE ROW
 ✗ ★ the counter can DRIFT from reality if any path forgets it
   ⇒ needs a reconciliation job
 ✗ deletes/suspensions must decrement, and getting that wrong
   silently under-counts forever

 MEASURED on the real distribution:
   largest org: 4,100 seats, ~40 seat changes/day
   p99 org:     ~2 seat changes/day
 ⇒ ★ CONTENTION ON THE COUNTER ROW IS EFFECTIVELY ZERO.
   The hot-row objection does not apply to THIS workload.

 ⇒ SO WHY NOT JUST DO THIS?
```

**Step 3 — the requirement that breaks it.**

```
 The product team adds: SCIM sync provisions seats IN BULK, up to
 2,000 at a time, and must be ATOMIC — either the whole batch fits
 within the limit or none of it applies.

 AND: seats can be 'active', 'suspended', or 'pending_invite', and
 only 'active' counts. Suspension/reactivation happens constantly
 from a background job.

 ⇒ ★ THE COUNTER NOW HAS FOUR TRANSITIONS TO MAINTAIN
   (insert-active, insert-pending, pending→active, active→suspended,
    suspended→active, delete) ACROSS THREE SERVICES.
 ⇒ EVERY ONE IS A PLACE TO GET IT WRONG, AND A WRONG COUNTER IS
   SILENT — it fails open (allows too many) or closed (blocks a
   paying customer), and nobody notices for weeks.

 ⇒ ★ THIS IS THE HONEST CASE FOR SERIALIZABLE:
   not "it's easier", but "the invariant is genuinely a predicate
   over a set with a non-trivial definition, and materialising it
   into a counter creates more failure modes than it removes."
```

**Step 4 — implement it, correctly.**

```js
const RETRYABLE = new Set(['40001', '40P01']);

async function withSerializableRetry(fn, { attempts = 4, op } = {}) {
  for (let i = 0; i < attempts; i++) {
    const client = await pool.connect();
    try {
      await client.query('BEGIN ISOLATION LEVEL SERIALIZABLE');
      const result = await fn(client);
      await client.query('COMMIT');
      if (i > 0) metrics.increment('ssi.retry_succeeded', { op, attempt: i });
      return result;
    } catch (e) {
      await client.query('ROLLBACK').catch(() => {});
      if (!RETRYABLE.has(e.code) || i === attempts - 1) {
        if (RETRYABLE.has(e.code)) metrics.increment('ssi.retry_exhausted', { op });
        throw e;
      }
      // ★ log the DETAIL — it distinguishes a predicate conflict
      //   from ordinary row contention
      metrics.increment('ssi.conflict', { op, detail: classify(e.detail) });
      await sleep(Math.random() * (20 * 2 ** i));   // ★ full jitter
    } finally {
      client.release();
    }
  }
}

async function grantSeats(orgId, userIds) {
  return withSerializableRetry(async (tx) => {
    // ★ narrow the read. Every tuple read becomes an SIReadLock,
    //   and every SIReadLock is a chance for a false positive.
    //   count(*) with a partial index reads far fewer pages than
    //   SELECT * would.
    const { rows: [{ used }] } = await tx.query(
      `SELECT count(*)::int AS used FROM seats
        WHERE org_id = $1 AND status = 'active'`, [orgId]);
    const { rows: [org] } = await tx.query(
      'SELECT seat_limit FROM organisations WHERE id = $1', [orgId]);
    if (!org) throw new AppError('ORG_NOT_FOUND');

    if (used + userIds.length > org.seat_limit)
      throw new AppError('SEAT_LIMIT_REACHED', {
        limit: org.seat_limit, used, requested: userIds.length });

    const { rows } = await tx.query(
      `INSERT INTO seats (org_id, user_id, status)
       SELECT $1, unnest($2::bigint[]), 'active'
       ON CONFLICT (org_id, user_id) DO NOTHING
       RETURNING id`, [orgId, userIds]);
    return { granted: rows.length };
  }, { op: 'grantSeats' });
}
```

```sql
-- ★ the partial index that makes the count cheap AND the predicate
--   locks narrow — both matter for SSI
CREATE INDEX CONCURRENTLY idx_seats_org_active
  ON seats (org_id) WHERE status = 'active';
```

**Step 5 — tune the predicate-lock budget.**

```sql
-- the largest org has 4,100 active seats. With max_pred_locks_per_
-- transaction = 64, reading them escalates straight to relation
-- level ⇒ every seat change anywhere in the system conflicts.
SHOW max_pred_locks_per_transaction;   -- 64

ALTER SYSTEM SET max_pred_locks_per_transaction = 1024;  -- ★ needs restart
ALTER SYSTEM SET max_pred_locks_per_page = 8;
```

```sql
-- verify after restart, under load
SELECT locktype, count(*) FROM pg_locks WHERE mode='SIReadLock' GROUP BY 1;
```
```
 locktype | count
----------+-------
 tuple    | 88204
 page     |  1204
   ★ NO 'relation' rows ⇒ no escalation ⇒ conflicts are real
```

```
 ★ MEASURED IMPACT OF THIS SETTING ALONE:
   max_pred_locks_per_transaction = 64    → 8.2% of grants got 40001
   max_pred_locks_per_transaction = 1024  → ★ 0.4%
   ⇒ 95% of the "SERIALIZABLE is too slow" problem was ONE SETTING.
```

**Step 6 — the reports get the free variant.**

```js
async function seatUsageReport(month) {
  const client = await pool.connect();
  try {
    // ★ perfectly consistent, mathematically cannot get 40001,
    //   contributes ZERO SSI overhead to the write paths
    await client.query(
      'BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE READ ONLY DEFERRABLE');
    const { rows } = await client.query(`
      SELECT o.id, o.name, o.seat_limit,
             count(*) FILTER (WHERE s.status='active') AS active_seats
        FROM organisations o LEFT JOIN seats s ON s.org_id = o.id
       GROUP BY o.id, o.name, o.seat_limit
      HAVING count(*) FILTER (WHERE s.status='active') > o.seat_limit`);
    await client.query('COMMIT');
    return rows;
  } finally { client.release(); }
}
```

**Step 7 — clean up the existing 412 violations.**

```sql
-- ★ SERIALIZABLE prevents NEW violations. It does not fix old ones.
--   This is a business decision, not a technical one.
SELECT o.id, o.name, o.seat_limit,
       count(*) FILTER (WHERE s.status='active') AS active
  FROM organisations o JOIN seats s ON s.org_id=o.id
 GROUP BY o.id, o.name, o.seat_limit
HAVING count(*) FILTER (WHERE s.status='active') > o.seat_limit
 ORDER BY active - o.seat_limit DESC;
-- ⇒ 412 orgs. Sales contacted each; most upgraded. None were
--   auto-suspended.
```

**Step 8 — results, 30 days.**

| | Before | After |
|---|---|---|
| Orgs over seat limit | 412 (4.6%) | **0 new** (412 legacy resolved by sales) |
| `grantSeats` p99 | 42 ms | 58 ms (+38%) |
| `grantSeats` throughput | 2,100/min | 1,980/min (−6%) |
| 40001 rate on `grantSeats` | n/a | **0.4%**, all retried successfully |
| 40001 on other endpoints | n/a | **0** (per-transaction, not global) |
| Report consistency | occasionally wrong | guaranteed, zero failure risk |
| Unbilled seat revenue | ₹18 lakh/yr leaking | **0** |

```
 ★ THE THREE LESSONS:
 ① SERIALIZABLE was the right answer here — but only after ruling
   out UNIQUE, EXCLUDE, CHECK, triggers and a counter row, each
   for a SPECIFIC reason. "It's easier" is not one of them.
 ② ★ 95% of the performance objection was max_pred_locks_per_
   transaction = 64. Escalation, not SSI, was the cost.
 ③ Setting it PER TRANSACTION meant a 6% cost on one endpoint
   instead of an 8.7× cost on all of them (Topic 44).
```

---

## Common mistakes

**1. Setting SERIALIZABLE globally.**
- *Symptom:* every endpoint pays; read-only endpoints produce most of the aborts.
- *Fix:* `BEGIN ISOLATION LEVEL SERIALIZABLE` on the specific handler.

**2. Leaving `max_pred_locks_per_transaction` at 64.**
- *Symptom:* high `40001` rates that don't correspond to real conflicts; `pg_locks` shows `locktype='relation'` with `mode='SIReadLock'`.
- *Engine-level why:* escalation to relation level means "read the whole table," so every write to it creates an rw-edge.
- *Fix:* raise it (512–2048 for workloads that read hundreds of rows), and narrow the queries.

**3. Long SERIALIZABLE transactions.**
- *Symptom:* rising `SIReadLock` counts, growing false-positive rate cluster-wide.
- *Engine-level why:* committed-transaction state cannot be cleaned up while any older serializable transaction is active; exhaustion forces coarse summarisation.
- *Fix:* keep them short. Use `READ ONLY DEFERRABLE` for anything long.

**4. Not using `READ ONLY` / `DEFERRABLE` for reports.**
- *Symptom:* reports fail with `40001` and add overhead for everyone.
- *Fix:* `SERIALIZABLE READ ONLY DEFERRABLE` — consistent, zero abort risk, zero cost to others.

**5. Reading more than you need.**
- *Symptom:* `SELECT *` where `count(*)` would do; conflicts far exceed the logical contention.
- *Engine-level why:* every tuple read becomes a predicate lock and a chance for an rw-edge.
- *Fix:* narrow columns, narrow predicates, and index so fewer pages are touched.

**6. Treating `40001` as fatal.**
- *Symptom:* users see 500s for a condition the server labelled retryable.
- *Fix:* a bounded retry loop with full jitter, and a metric.

**7. Not logging the `DETAIL`.**
- *Symptom:* you know you're getting `40001` but not whether it's a predicate conflict or row contention — which have different fixes.
- *Fix:* log the `DETAIL` and the reason code; classify them separately.

**8. Retrying a non-idempotent transaction.**
- *Symptom:* duplicate side effects on retry.
- *Fix:* the retried unit must be safe to re-run (Topic 52).

**9. Reaching for SERIALIZABLE before ruling out the cheaper options.**
- *Symptom:* retries and overhead to enforce something a `UNIQUE` constraint would have enforced for free.
- *Fix:* work through `UNIQUE` → `EXCLUDE` → `CHECK` → atomic statement → counter row → SERIALIZABLE, and be able to say why each was rejected.

**10. Believing "SERIALIZABLE is 8× slower."**
- *Symptom:* it's dismissed without measurement.
- *Engine-level why:* that figure comes from single-hot-row benchmarks, where every isolation level performs badly. On a spread workload the overhead is ~12%.
- *Fix:* measure on your distribution.

---

## Hands-on proof

**PROVE IT #1–#6 — Example 1** (write skew caught, tuple-level `SIReadLock`s, forced escalation producing a false positive, `READ ONLY DEFERRABLE` never failing, the two `40001` messages, spread vs hot overhead).

**PROVE IT #7 — see a read-only transaction contribute nothing.**
```sql
-- session 1
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE READ ONLY;
SELECT count(*) FROM on_call WHERE is_on_call;
-- session 2
SELECT count(*) FROM pg_locks WHERE mode='SIReadLock';
-- ★ far fewer entries than a read-write transaction doing the
--   same query, because a read-only txn can never be a pivot's
--   successor.
```

**PROVE IT #8 — the escalation threshold, measured.**
```bash
for n in 64 256 1024; do
  echo "== max_pred_locks_per_transaction = $n"
  psql -c "ALTER SYSTEM SET max_pred_locks_per_transaction = $n" >/dev/null
  pg_ctl restart -D "$PGDATA" -m fast >/dev/null 2>&1; sleep 3
  PGOPTIONS="-c default_transaction_isolation='serializable'" \
    pgbench -f /tmp/seatgrant.sql -c 50 -j 4 -T 30 shop 2>&1 | grep -E 'tps|failed'
  psql -tc "SELECT locktype, count(*) FROM pg_locks WHERE mode='SIReadLock' GROUP BY 1"
done
```
```
 == 64
 tps = 1,204.8   failed = 4,102     locktype: relation
 == 256
 tps = 6,880.1   failed =   882     locktype: page
 == 1024
 tps = 14,204.2  failed =    58     ★ locktype: tuple
```

**PROVE IT #9 — the retry actually succeeds.**
```sql
-- after a 40001, immediately re-run the same transaction.
-- ★ it succeeds, because the conflicting transaction has now
--   committed and your new snapshot includes it.
-- This is why bounded retries with small backoff work so well:
-- the conflict window has already closed.
```

**PROVE IT #10 — no false negatives, ever.**
```bash
# hammer the doctors invariant at 100 clients for 5 minutes
cat > /tmp/oncall.sql <<'EOF'
BEGIN ISOLATION LEVEL SERIALIZABLE;
SELECT count(*) FROM on_call WHERE is_on_call;
UPDATE on_call SET is_on_call = false
 WHERE doctor_id = (SELECT doctor_id FROM on_call WHERE is_on_call LIMIT 1);
COMMIT;
EOF
pgbench -f /tmp/oncall.sql -c 100 -j 8 -T 300 shop
psql -c "SELECT count(*) FROM on_call WHERE is_on_call"
```
```
 count
-------
     1        ★ never zero. Not once, across millions of attempts.
              SSI has no false negatives.
```

---

## The design decision framework

```
★★★ SERIALIZABLE IS THE LAST RESORT — AND SOMETIMES THE RIGHT ONE. ★★★

 ① WORK THROUGH THE CHEAPER OPTIONS AND BE ABLE TO SAY WHY EACH
    WAS REJECTED
    UNIQUE            — is something duplicated?
    EXCLUDE           — do two rows overlap?
    CHECK             — is it a per-row rule?
    atomic UPDATE     — is the whole decision expressible in SQL?
    ★ a counter row   — can the invariant be materialised into one
                        object every writer must touch?
    ⇒ if any fits: use it. READ COMMITTED, no retries, unbypassable.

 ② SERIALIZABLE IS RIGHT WHEN THE INVARIANT IS A PREDICATE OVER A
    SET THAT THE TRANSACTION ALSO WRITES INTO
      "count over a set ≤ N"      "sum over a set ≤ limit"
      "at least one must remain"  "no overlap across a join"
    ⇒ ★ AND when materialising it into a counter would create more
      failure modes than it removes (many state transitions, many
      writing services, silent drift).

 ③ ALWAYS PER TRANSACTION
    BEGIN ISOLATION LEVEL SERIALIZABLE;
    ✗ never default_transaction_isolation
    ⇒ a 6% cost on one endpoint beats an 8.7× cost on all of them.

 ④ THE FOUR TUNING LEVERS, IN ORDER OF IMPACT
    ① ★ max_pred_locks_per_transaction — default 64 is far too low
       for anything reading >100 rows. 512–2048.
       ⇒ check pg_locks for locktype='relation' + SIReadLock.
         Its presence means false positives are happening NOW.
    ② ★ NARROW THE READS — every tuple read is a predicate lock.
       count(*) over a partial index beats SELECT * enormously.
    ③ ★ KEEP TRANSACTIONS SHORT — long ones block SSI cleanup for
       the whole cluster.
    ④ mark read-only transactions READ ONLY; use DEFERRABLE for
       long reports.

 ⑤ WHAT YOU OWE
    ✓ a retry loop: 40001 + 40P01, bounded, ★ full jitter
    ✓ ★ an idempotent transaction body (Topic 52)
    ✓ ★ log the DETAIL, not just the SQLSTATE
    ✓ metrics: conflict rate, retry-success rate, retry-exhausted rate
    ⇒ a rising conflict rate is a DESIGN signal, not noise

 ⑥ THE FREE WIN NOBODY USES
    ★ BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE
        READ ONLY DEFERRABLE;
    for every long report, export, or reconciliation.
    ⇒ perfect consistency · zero 40001 risk · zero cost to others
    ⇒ the only price is possibly waiting to start.

 ⑦ BEFORE CONCLUDING "SERIALIZABLE IS TOO SLOW"
    ★ measure on YOUR key distribution, not on one hot row.
    ★ check for relation-level SIReadLocks first.
    ⇒ 12% overhead and 95% of the objection being one GUC is the
      common outcome.
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
Reproduce the doctors write skew at REPEATABLE READ (succeeds, invariant broken) and at SERIALIZABLE (one transaction aborts). Then: (a) capture the `SIReadLock` entries during the SERIALIZABLE run and explain what each represents; (b) read the abort `DETAIL` and explain the reason code; (c) show that `READ ONLY DEFERRABLE` never aborts under the same load.

### Exercise 2 — medium (apply it)
Force predicate-lock escalation by running a SERIALIZABLE transaction that scans 200,000 rows with `max_pred_locks_per_transaction = 64`. Show `pg_locks` moving from `tuple` → `page` → `relation`. Then demonstrate a false-positive `40001` caused by a write to a row the reader could not care about.

Repeat at 256 and 1024 and report tps, failure count, and lock granularity for each. Explain the shape of the curve.

### Exercise 3 — hard (production simulation)
A SaaS platform enforces "an organisation may not exceed its plan's seat limit." 412 of 8,904 orgs are over. Three services grant seats. The check exists in all three and looks correct.

(a) Name the anomaly and give the exact three-transaction timeline that produces it.
(b) Explain, individually, why `UNIQUE`, `EXCLUDE`, `CHECK`, a trigger, and `REPEATABLE READ` each fail to fix it.
(c) Design the counter-row solution. Give the schema, the constraint, and the required transitions. Measure the contention on the real distribution (largest org: 4,100 seats, ~40 changes/day) and say whether the hot-row objection applies.
(d) The requirements grow: bulk SCIM provisioning of up to 2,000 seats atomically, plus three seat statuses with constant transitions. Argue *specifically* why this makes the counter row worse than SERIALIZABLE — not "harder," but which failure modes it introduces.
(e) Implement `grantSeats` under SERIALIZABLE with the retry helper. Justify the partial index and explain how it reduces *conflicts*, not just query time.
(f) The largest org has 4,100 active seats. Compute what happens at `max_pred_locks_per_transaction = 64` and explain the resulting `40001` rate.
(g) Measure tps and failure rate at 64, 256 and 1024, and report the lock granularity at each.
(h) Write the report endpoint using the free variant, and explain the guarantee it gets.
(i) SERIALIZABLE prevents new violations but not the 412 existing ones. Say why, and what the remediation is.
(j) Write the four metrics you'd expose and the threshold at which each means "the design is wrong, not the tuning."

---

## Mental model checkpoint

1. State SSI's guarantee in one sentence. How is it different from "prevents the anomalies in the SQL standard"?
2. What is an rw-antidependency? Why does it force an ordering?
3. State the theorem SSI relies on. What is a pivot?
4. Why does SSI produce false positives but never false negatives?
5. What is an `SIReadLock`? What does it block?
6. Explain predicate-lock escalation and the exact mechanism by which it causes spurious `40001`s on a read-only endpoint.
7. Why must SSI remember committed transactions, and what does one long SERIALIZABLE transaction cost the cluster?
8. Why is `SERIALIZABLE READ ONLY DEFERRABLE` guaranteed never to abort?
9. Distinguish the two `40001` messages and what each tells you to look at.
10. Why is "SERIALIZABLE is 8× slower" usually a misleading benchmark?

---

## Quick reference card

**The guarantee:** any set of committed SERIALIZABLE transactions ≡ some serial execution. **Every** anomaly, named or not.

**The mechanism:** snapshot isolation + track reads (`SIReadLock`) + create rw-edges on writes + abort when one transaction has **both** an incoming and an outgoing rw-edge (**the pivot**).

```
   Tin ──rw──► Tpivot ──rw──► Tout      ★ the only pattern SSI looks for
```

**Use it**
```sql
BEGIN ISOLATION LEVEL SERIALIZABLE;                                    -- writes
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE READ ONLY;              -- cheaper
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE READ ONLY DEFERRABLE;   -- ★ free
```

**Diagnose**
```sql
SELECT locktype, relation::regclass, count(*) FROM pg_locks
 WHERE mode='SIReadLock' GROUP BY 1,2 ORDER BY 3 DESC;
-- ★ locktype='relation' ⇒ ESCALATION ⇒ false-positive 40001s NOW
SHOW max_pred_locks_per_transaction;   -- ★ 64 is far too low
```

**Tune, in order of impact**
1. ★ `max_pred_locks_per_transaction` → 512–2048
2. ★ narrow the reads (`count(*)` + partial index, not `SELECT *`)
3. ★ short transactions
4. `READ ONLY` / `DEFERRABLE` on reads

**The two `40001`s**

| `DETAIL` | Meaning | Look at |
|---|---|---|
| *read/write dependencies among transactions* | ★ SSI predicate conflict | query breadth, `max_pred_locks_*` |
| *concurrent update* | row contention (also at REPEATABLE READ) | row design, hot rows |

**Cost:** ~12% on a spread workload · ~88% on one hot row · **most "too slow" verdicts are escalation.**

**Order of resort:** `UNIQUE` → `EXCLUDE` → `CHECK` → atomic `UPDATE` → counter row → **SERIALIZABLE**.

---

## When would I use this at work?

1. **Any invariant of the form "no more than N" or "the sum must not exceed."** These cannot be expressed as constraints, and `REPEATABLE READ` does not help. SERIALIZABLE and a materialised counter are the only two real options — and knowing SSI's cost model is what lets you choose between them on evidence.

2. **When someone reports high `40001` rates and concludes SERIALIZABLE doesn't work.** Check `pg_locks` for relation-level `SIReadLock`s first. In practice this is the cause far more often than genuine contention, and it's one GUC.

3. **Every long-running report.** `SERIALIZABLE READ ONLY DEFERRABLE` gives a mathematically consistent result with zero abort risk and zero cost to other transactions. It's strictly better than what most teams do, and it's one line.

4. **Explaining to a team why "we'll just be careful in the application" doesn't work.** Write skew is invisible in code review — every path looks correct in isolation. The engine can enforce it; three services agreeing to remember something cannot.

---

## Connected topics

**Understand before this:** 43 (write skew — the anomaly SSI exists for), 44 (isolation levels), 46 (MVCC — the snapshot SSI builds on), 49 (optimistic concurrency — SSI is the same idea over predicates), 24 (constraints — the cheaper options to rule out first).

**This unlocks:**
- **51** — distributed transactions: what serializability costs across nodes
- **52** — idempotency: what makes the retry safe
- **68** — CAP and consistency models: where serializability sits among them
- **Case study 15** — multi-tenant SaaS, where per-tenant limits are exactly this problem
