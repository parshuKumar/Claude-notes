# 66 — Database Transactions and Isolation Levels
## Category: HLD Components

---

## What is this?

A **transaction** is a group of database operations the database treats as a **single indivisible unit** — either every operation succeeds, or none do. There is no "half-done."

Think of an **ATM withdrawal**. The machine counts out ₹5,000, your balance drops by ₹5,000, and the event is logged. If the power dies after the cash comes out but before the balance drops, the bank just gave you free money. A transaction is the mechanism that says: *these things happen together, or they don't happen at all.*

**Isolation levels** are the second half of the story: they control how much two transactions running *at the same time* can see of each other's half-finished work. Turn isolation up and you get correctness; turn it down and you get speed. That dial is one of the most consequential — and least understood — settings in your entire backend.

---

## Why does it matter?

Because without transactions, **concurrency silently corrupts your data** — quietly, in production, under load, in ways that never appear in your tests.

- **Money vanishes.** A transfer debits account A, the process crashes, the credit to B never happens. ₹5,000 gone from the universe.
- **Overselling.** Two customers buy the last concert ticket. Both read `remaining = 1`, both write `remaining = 0`. You sold one ticket twice.
- **Invariants break.** Your system guarantees "at least one doctor is always on call." Two doctors go off call at the same instant, and the rule quietly dies.

**Interview angle:** "What's the difference between Repeatable Read and Serializable?" and "How do you stop two users booking the same seat?" are among the most common backend follow-ups. The expected answers: isolation levels, `SELECT ... FOR UPDATE`, optimistic version columns, retry-on-deadlock.

**Real-work angle:** Every read-modify-write path in your Node service — decrementing inventory, incrementing a counter, transferring credits, assigning a slot — is a correctness bug waiting to happen unless you consciously choose an isolation level or locking strategy. Most engineers never choose. They inherit the default (Postgres: Read Committed) and ship a race condition.

---

## The core idea — explained simply

### The Shared Whiteboard Analogy

A small accounting office. One whiteboard on the wall holds everyone's account balances. Several clerks work at once — walking up, reading numbers, erasing them, writing new ones.

**Clerk A** does a transfer: move ₹5,000 from Alice to Bob.
1. Read Alice: ₹10,000 → 2. Erase, write ₹5,000 → 3. Read Bob: ₹2,000 → 4. Erase, write ₹7,000

Four things can go wrong:

**The clerk faints between step 2 and step 3.** Alice lost ₹5,000; Bob never got it. Money destroyed. **Atomicity** fixes this: the office keeps a *notebook of every erasure* ("Alice was 10,000 before I changed it"). If a clerk collapses mid-way, the supervisor reads the notebook backwards and restores the board. That notebook is the **undo log**.

**A power surge wipes the (electronic) board.** Everything gone. **Durability** fixes this: before *any* change touches the board, the clerk writes the intended change into a bound paper ledger and hands it to the safe. If the board is wiped, you re-apply the ledger. That ledger is the **write-ahead log (WAL)**.

**Clerk B walks up mid-transfer**, reads Alice's ₹5,000 *and* Bob's still-old ₹2,000, and computes the office total as ₹7,000 instead of ₹12,000. He saw a half-finished transfer. **Isolation** fixes this.

**A clerk writes -₹300** on a board whose rules say balances can never go negative. **Consistency** fixes this.

| Whiteboard problem | ACID property | The mechanism |
|---|---|---|
| Clerk faints mid-way | **A**tomicity | Undo log / rollback segment |
| The board's rules are violated | **C**onsistency | Constraints, foreign keys, your app's invariants |
| Another clerk sees half-finished work | **I**solation | Locks / MVCC snapshots |
| The board is wiped by a power cut | **D**urability | Write-ahead log + fsync |

Everything below is detail on top of that picture.

---

## Key concepts inside this topic

### 1. ACID — property by property, with the HOW

Anyone can recite ACID. Few can explain *how the database actually delivers each letter.* That's the interview differentiator.

#### A — Atomicity: all or nothing

**Promise:** a transaction's writes are all applied or none are. No partial state is ever visible — not even after a crash.

**Mechanism: the undo log** (Postgres: old row versions; MySQL/InnoDB: the rollback segment; Oracle: undo tablespace). Before modifying a row, the database records its *previous* value:

```
Txn 42 wants: UPDATE accounts SET balance = 5000 WHERE id = 'alice'

Step 1 — write the undo entry: [txn 42 | accounts | id=alice | old balance = 10000]
Step 2 — modify the data page: alice.balance = 5000
```

On `COMMIT`, the undo entry is discarded — nobody needs it. On `ROLLBACK` (or a kill, or a power cut), recovery walks the undo log backwards and puts 10000 back. That is *literally all* a rollback is: replaying the undo log in reverse. Atomicity isn't magic; it's bookkeeping.

```sql
BEGIN;
  UPDATE accounts SET balance = balance - 5000 WHERE id = 'alice';
  UPDATE accounts SET balance = balance + 5000 WHERE id = 'bob';
COMMIT;   -- both, or (on failure) neither
```

Unplug the server between the two `UPDATE`s: on restart the database sees txn 42 in the log with **no COMMIT record**, so it undoes it. Alice gets her ₹5,000 back.

#### C — Consistency: the invariants hold before and after

**Promise:** a transaction takes the database from one *valid* state to another. "Valid" comes from two places — **database-enforced rules** (`NOT NULL`, `UNIQUE`, `CHECK`, `FOREIGN KEY`) and **application invariants** ("the sum of all balances is constant," "≥1 doctor is on call," "an order's total equals the sum of its line items").

```sql
CREATE TABLE accounts (
  id       TEXT PRIMARY KEY,
  balance  BIGINT NOT NULL CHECK (balance >= 0)   -- a database-enforced rule
);
```

Try to overdraw and the transaction is rejected and rolled back — the invariant survives.

**The honest truth about C:** it's the odd one out. A, I and D are properties the *database* guarantees. C is mostly a property **you** guarantee, and the database just hands you tools (constraints) to help. If your code debits an account twice by accident, that's atomic, isolated, durable — and completely wrong.

#### I — Isolation: concurrent transactions don't corrupt each other

**Promise:** concurrent transactions produce the same result as if they had run **one after another**. That's the ideal — full serializability. In practice databases offer **weaker levels**, because true serializability is expensive. Weaker levels let specific anomalies through in exchange for throughput. Knowing *exactly which anomaly each level allows* is the heart of this topic (sections 2 and 3).

#### D — Durability: once committed, it survives a crash

**Promise:** when `COMMIT` returns, the data is safe. Pull the power cord a nanosecond later; it's still there on reboot.

**Mechanism: Write-Ahead Logging (WAL).** The naive approach would be "on commit, write the modified data pages to disk." That's terrible: data pages are scattered all over the disk (**random I/O** — slow, even on SSDs), one transaction may touch 8 pages in 8 places, and if you crash halfway through you have a **torn, half-updated database** with no way to tell what's valid.

WAL flips the order:

> **The WAL rule:** write the log record to disk and `fsync()` it **BEFORE** you write the corresponding data page to disk.

The sequence on `COMMIT`:

```
1. Append to the WAL buffer (RAM):
     [LSN 1001 | txn 42 | page 7 | alice.balance: 10000 -> 5000]
     [LSN 1002 | txn 42 | page 9 | bob.balance:    2000 -> 7000]
     [LSN 1003 | txn 42 | COMMIT]
2. fsync() the WAL to physical disk.  <-- THE DURABILITY POINT. Blocks until the
                                          disk confirms the bytes are truly persisted,
                                          not merely sitting in an OS or disk cache.
3. NOW return "COMMIT OK" to the client.
4. Data pages 7 and 9 are STILL DIRTY in memory. They're flushed lazily, later,
   in bulk, at a checkpoint.
```

Why this is both **safe** and **fast**:

- **Safe:** the WAL is on disk before you're told "committed." Die at step 4 and recovery reads the WAL from the last checkpoint and **replays (redoes)** LSN 1001–1002, reconstructing the lost pages. Find a transaction with no COMMIT record and it **undoes** it.
- **Fast:** the WAL is **sequential** — one append-only file, one contiguous write. Sequential writes are 10–100× faster than random ones. And one fsync can commit many transactions at once (**group commit**).

| Operation | Typical latency |
|---|---|
| Sequential WAL append + fsync (NVMe SSD) | ~0.1–1 ms |
| Random write of 8 scattered data pages | ~1–8 ms (SSD), 40–80 ms (spinning disk) |
| fsync on a cloud network disk (EBS gp3) | ~1–2 ms |

That fsync is why "how many transactions per second can Postgres do?" is often really "how fast is your disk's fsync?" A single-threaded commit loop against a 1 ms fsync tops out around **1,000 commits/sec**; group commit and batching get you past that.

**The dangerous knob:** `synchronous_commit = off` skips waiting for the fsync. Commits get dramatically faster and **durability breaks** — you can lose the last ~200 ms of "committed" transactions on a crash. Fine for analytics ingest. Not fine for payments.

---

### 2. The read anomalies — the heart of this topic

These are the specific ways concurrent transactions corrupt each other. Learn all five by their interleaving table. Time flows **downward**; the two columns are two transactions running at the same time.

#### Anomaly 1: Dirty Read

T1 reads a row T2 has modified but **not yet committed**. T2 then rolls back — so T1 read a value that *never existed*. Start: `alice.balance = 10000`.

| Time | T1 (reporting job) | T2 (transfer) | What happens |
|---|---|---|---|
| 1 | | `BEGIN` | |
| 2 | | `UPDATE accounts SET balance=5000 WHERE id='alice'` | uncommitted! |
| 3 | `BEGIN` | | |
| 4 | `SELECT balance ... id='alice'` → **5000** | | **DIRTY READ** |
| 5 | | `ROLLBACK` | alice is back to 10000 |
| 6 | reports "Alice has ₹5,000" | | **A lie. That value never existed.** |

**Damage:** your report, your email, your Kafka event are all built on a phantom value.
**Prevented by:** Read Committed and above. (Postgres never allows dirty reads at *any* level.)

#### Anomaly 2: Non-Repeatable Read

T1 reads the **same row twice** and gets **different values**, because T2 committed in between. Start: `product.price = 100`.

| Time | T1 (checkout) | T2 (admin price change) | What happens |
|---|---|---|---|
| 1 | `BEGIN` | | |
| 2 | `SELECT price ... id=7` → **100** | | quotes ₹100 to the user |
| 3 | | `BEGIN` | |
| 4 | | `UPDATE products SET price=150 WHERE id=7` | |
| 5 | | `COMMIT` | |
| 6 | `SELECT price ... id=7` → **150** | | **NON-REPEATABLE READ** |
| 7 | charges the card ₹150 | | **You quoted 100 and charged 150.** |

**Damage:** any validate-then-use logic that reads a row twice is broken.
**Prevented by:** Repeatable Read and above.

#### Anomaly 3: Phantom Read

T1 runs the same **range query** twice and gets **different rows** — new ones appeared, because T2 inserted and committed. (Non-repeatable read = a **row's value** changed. Phantom = the **set of matching rows** changed.) Start: 3 bookings for room 5, capacity 4.

| Time | T1 (booking check) | T2 (another booker) | What happens |
|---|---|---|---|
| 1 | `BEGIN` | | |
| 2 | `SELECT COUNT(*) FROM bookings WHERE room=5 AND day='2026-08-01'` → **3** | | "1 slot left" |
| 3 | | `BEGIN` | |
| 4 | | `INSERT INTO bookings (room, day) VALUES (5,'2026-08-01')` | |
| 5 | | `COMMIT` | now there are 4 |
| 6 | `SELECT COUNT(*) ...` → **4** | | **PHANTOM READ** — a row appeared |
| 7 | `INSERT` its own booking anyway | | **5 bookings in a 4-capacity room** |

**Damage:** every check-then-insert path (uniqueness, capacity, quota) is unsafe.
**Prevented by:** Serializable — and in practice by Postgres's snapshot-based Repeatable Read too (section 3).

#### Anomaly 4: Lost Update

Two transactions **read-modify-write** the same row. The second write overwrites the first, and the first **silently disappears**. This is the most common data race in real applications. Start: `inventory.stock = 10`.

| Time | T1 (order #1) | T2 (order #2) | Value in DB |
|---|---|---|---|
| 1 | `BEGIN` | | 10 |
| 2 | `SELECT stock ... id=7` → **10** | | 10 |
| 3 | | `BEGIN` | 10 |
| 4 | | `SELECT stock ... id=7` → **10** | 10 |
| 5 | (in Node) `newStock = 10 - 1 = 9` | | 10 |
| 6 | | (in Node) `newStock = 10 - 1 = 9` | 10 |
| 7 | `UPDATE inventory SET stock=9 WHERE id=7` | | 9 |
| 8 | `COMMIT` | | 9 |
| 9 | | `UPDATE inventory SET stock=9 WHERE id=7` | 9 |
| 10 | | `COMMIT` | **9** |

Two units sold. Stock dropped by **one**. You just oversold, and it will never reconcile.

**Note carefully:** this happens at **Read Committed — the Postgres default.** It is not an exotic edge case. If your Node code does `SELECT` → compute in JS → `UPDATE`, you have this bug right now.
**Prevented by:** `SELECT ... FOR UPDATE`, an atomic `UPDATE ... SET stock = stock - 1`, a version column, or Serializable. Code in section 5.

#### Anomaly 5: Write Skew (the subtle one)

Two transactions read an **overlapping set of rows**, each decides based on what it read, and each writes to a **different** row. Neither write conflicts with the other — but the **combined result violates an invariant** that each transaction individually preserved. This is the anomaly snapshot isolation does **not** prevent, and the one senior interviews ask about.

**The on-call doctor example.** Hospital rule: **at least one doctor must be on call at all times.** Alice and Bob are both on call. Both feel sick and both hit "go off call" at the same instant. Both requests run:

```sql
BEGIN;
  SELECT COUNT(*) FROM doctors WHERE on_call = true AND shift_id = 1234;
  -- if count >= 2, it's safe for me to leave
  UPDATE doctors SET on_call = false WHERE name = <me> AND shift_id = 1234;
COMMIT;
```

| Time | T1 (Alice leaves) | T2 (Bob leaves) | Invariant |
|---|---|---|---|
| 1 | `BEGIN` | | Alice ✓, Bob ✓ |
| 2 | | `BEGIN` | Alice ✓, Bob ✓ |
| 3 | `SELECT COUNT(*) WHERE on_call` → **2** | | both see 2 |
| 4 | | `SELECT COUNT(*) WHERE on_call` → **2** | both see 2 |
| 5 | "2 ≥ 2, safe to leave" | | |
| 6 | | "2 ≥ 2, safe to leave" | |
| 7 | `UPDATE doctors SET on_call=false WHERE name='Alice'` | | |
| 8 | | `UPDATE doctors SET on_call=false WHERE name='Bob'` | |
| 9 | `COMMIT` | | |
| 10 | | `COMMIT` | **ZERO DOCTORS ON CALL** |

Why nothing caught it: **T1 wrote the Alice row, T2 wrote the Bob row.** Different rows. No write-write conflict. No lock contention. Both saw a perfectly consistent snapshot. Both were individually correct. The *combination* is catastrophic.

Same shape, different clothes:
- **Meeting-room double booking:** both check "is the room free at 3 PM?", both see yes, both insert.
- **Overdraft across two accounts:** the rule is "checking + savings ≥ 0." Two withdrawals each check the combined balance, each sees enough, each withdraws from a *different* account. Combined goes negative.
- **Username uniqueness in app code:** both check "is `parshu` taken?", both see no, both insert.

**Prevented by:** `SERIALIZABLE` (Postgres SSI detects it and aborts one), or by *forcing* a write-write conflict yourself — `SELECT ... FOR UPDATE` on every row you read, or a `UNIQUE` constraint when the invariant happens to be expressible as one.

---

### 3. The four isolation levels

| Level | Dirty read | Non-repeatable read | Phantom read | Lost update | Write skew | Cost |
|---|---|---|---|---|---|---|
| **Read Uncommitted** | ✗ possible | ✗ possible | ✗ possible | ✗ possible | ✗ possible | cheapest |
| **Read Committed** | ✓ prevented | ✗ possible | ✗ possible | ✗ possible | ✗ possible | low |
| **Repeatable Read** | ✓ prevented | ✓ prevented | ✗ possible* | ✗ possible* | ✗ possible | medium |
| **Serializable** | ✓ prevented | ✓ prevented | ✓ prevented | ✓ prevented | ✓ prevented | highest |

`*` = the SQL standard permits it, but real implementations differ — see below.

- **Read Uncommitted:** you can see other transactions' uncommitted writes. Almost never useful. **Postgres doesn't implement it** — asking for it silently gives you Read Committed.
- **Read Committed:** you only see committed data, but each *statement* gets a **fresh snapshot**, so two identical `SELECT`s can differ.
- **Repeatable Read:** the whole *transaction* gets **one snapshot**, taken at the start. Every read sees the same frozen world.
- **Serializable:** the result is equivalent to *some* serial (one-at-a-time) execution. All five anomalies gone.

#### The real-world defaults you must memorize

| Database | Default | Notes |
|---|---|---|
| **PostgreSQL** | **Read Committed** | Never allows dirty reads at any level |
| **MySQL (InnoDB)** | **Repeatable Read** | Gap locks block most phantoms too |
| **Oracle** | Read Committed | Its "Serializable" is really snapshot isolation |
| **SQL Server** | Read Committed | Lock-based by default |
| **CockroachDB** | **Serializable** | Serializable is the *only* level |

#### The subtlety interviewers probe

**Postgres's "Repeatable Read" is really Snapshot Isolation (SI)** — which is *stronger* than the standard's Repeatable Read in one way and still *weaker* than Serializable in another.

- **Stronger:** because the whole transaction reads one consistent snapshot, a repeated range query returns the same rows. So **Postgres's Repeatable Read prevents phantom reads**, even though the standard permits them. It also prevents **lost updates** — Postgres aborts the second writer with `could not serialize access due to concurrent update`.
- **Still weaker: snapshot isolation does NOT prevent write skew.** The doctor example runs happily under Postgres Repeatable Read and breaks your invariant. SI only detects *write-write* conflicts on the same row; write skew is a *read-write* conflict on **different** rows.

To catch write skew you must go to `SERIALIZABLE`, which enables **SSI (Serializable Snapshot Isolation)**: Postgres tracks read/write dependencies between transactions and aborts one when it spots a dangerous cycle. It's optimistic — no extra locking — but you **must** retry on SQLSTATE `40001`.

```sql
BEGIN ISOLATION LEVEL SERIALIZABLE;
  SELECT COUNT(*) FROM doctors WHERE on_call = true AND shift_id = 1234;
  UPDATE doctors SET on_call = false WHERE name = 'alice' AND shift_id = 1234;
COMMIT;   -- may fail with 40001; you retry the WHOLE transaction
```

**Performance cost, roughly:** Read Committed ≈ 1.0× throughput; Repeatable Read ≈ 0.95–1.0× (snapshots are nearly free under MVCC); Serializable ≈ 0.7–0.95× at low contention, and it can collapse much further under high contention. **The cost of Serializable is not lock waiting — it's retries.**

---

### 4. MVCC — why readers and writers don't block each other

**MVCC = Multi-Version Concurrency Control.** In one sentence:

> Instead of overwriting a row, the database writes a **new version** of it and keeps the old one, so readers keep reading the old version while a writer creates the new one.

Every physical row in Postgres carries two hidden columns: `xmin` (the transaction that **created** this version) and `xmax` (the transaction that **superseded** it; 0 if still live). An `UPDATE` is not an in-place edit — it's *mark the old version dead (`xmax = me`), insert a brand-new version (`xmin = me`)*.

```
Row "alice" after txn 100 created it and txn 205 updated the balance:

  ┌────────────────────────────────────────────────┐
  │ v1:  xmin=100  xmax=205   balance = 10000      │ ← dead to new txns, still
  └────────────────────────────────────────────────┘   visible to older snapshots
  ┌────────────────────────────────────────────────┐
  │ v2:  xmin=205  xmax=0     balance =  5000      │ ← the live version
  └────────────────────────────────────────────────┘
```

When a transaction starts it takes a **snapshot** ("I am txn 300; txns 250, 260, 270 were still in flight when I began"). For every row version it meets, it applies a visibility rule: *show me the version whose creator committed before my snapshot and whose deleter had not.*

**The two consequences that make Postgres feel fast under load:**

1. **Readers never block writers.** A long analytics `SELECT` scanning a million rows takes **no locks** on them. A writer can update those rows concurrently — it just creates new versions the reader can't see.
2. **Writers never block readers.** The writer never had to take the old version away.

Writers **do** still block *other writers* on the same row — that's a genuine conflict, implemented with row locks. Contrast an old-school lock-based database, where a reader takes a shared lock and a writer needs an exclusive one, so a big report **blocks all writes**. Every engineer who ever ran a report on a production OLTP database and took the site down has felt that. MVCC is the fix.

**The cost of MVCC: dead tuples and VACUUM.** Old versions don't vanish on their own. Once no live snapshot can see v1 of alice, it's a **dead tuple** — garbage occupying disk pages. Postgres's background **autovacuum** reclaims it.

What goes wrong in production:

- **Table bloat.** A heavily updated table (job queue, counter, sessions) churns out dead tuples. If autovacuum can't keep up, the table's physical size balloons to 10× the live data. Every sequential scan now reads 10× the pages, and your queries slow down for no visible reason.
- **Blocked vacuum.** Vacuum **cannot** remove a dead tuple that *any* open transaction might still need to see. One long-running transaction pins the horizon and **stops vacuum across the whole database.** See section 7.

```sql
-- Your production diagnostic. Learn it.
SELECT relname, n_live_tup, n_dead_tup,
       ROUND(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 1) AS dead_pct,
       last_autovacuum
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC LIMIT 10;
```

`dead_pct` above ~20% on a hot table with a stale `last_autovacuum` means you have a bloat problem.

---

### 5. Pessimistic vs optimistic locking (with working code)

Both solve the **lost update** problem. They make opposite bets.

| | Pessimistic | Optimistic |
|---|---|---|
| The bet | "A conflict is likely — I'll lock first." | "A conflict is unlikely — I'll check at the end." |
| Mechanism | `SELECT ... FOR UPDATE` | `version` column + compare-and-swap |
| Others' experience | They **wait** | They **fail and retry** |
| Cost | Lock waits, deadlock risk | Wasted work on retry |

#### Pessimistic: `SELECT ... FOR UPDATE`

You take an **exclusive row lock at read time**. Any other transaction that tries to `SELECT ... FOR UPDATE` or `UPDATE` that row **blocks** until you commit or roll back.

```javascript
// pessimistic.js — inventory decrement with an exclusive row lock (node-postgres).
import pg from 'pg';
const pool = new pg.Pool({ connectionString: process.env.DATABASE_URL });

/**
 * We grab the row lock BEFORE we compute, so no other transaction can even
 * read-for-update this row until we're done. The lost-update race from
 * section 2 becomes physically impossible.
 */
async function reserveStockPessimistic(productId, qty) {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');

    // FOR UPDATE = "lock this row exclusively until my transaction ends."
    // A concurrent call for the same productId BLOCKS on this line.
    const { rows } = await client.query(
      `SELECT stock FROM inventory WHERE id = $1 FOR UPDATE`, [productId]
    );
    if (rows.length === 0) throw new Error('PRODUCT_NOT_FOUND');

    const stock = rows[0].stock;        // nobody else can change it now
    if (stock < qty) throw new Error('OUT_OF_STOCK');

    await client.query(
      `UPDATE inventory SET stock = $1 WHERE id = $2`, [stock - qty, productId]
    );

    await client.query('COMMIT');       // the lock is released HERE
    return { ok: true, remaining: stock - qty };
  } catch (err) {
    await client.query('ROLLBACK');     // ...or here
    throw err;
  } finally {
    client.release();
  }
}
```

Two variants worth knowing:

```sql
-- Don't wait; error immediately if someone else holds the lock.
SELECT stock FROM inventory WHERE id = $1 FOR UPDATE NOWAIT;

-- Skip locked rows — the standard job-queue trick so workers don't queue behind
-- each other; each worker grabs a DIFFERENT unclaimed job.
SELECT id FROM jobs WHERE status = 'pending'
ORDER BY created_at LIMIT 1 FOR UPDATE SKIP LOCKED;
```

#### Optimistic: a `version` column and compare-and-swap

No locks. Read the row **and its version**, think, then write with a condition: *"apply this only if the version is still what I read."* If someone beat you, the `UPDATE` matches **zero rows** — that's your signal to retry.

```sql
ALTER TABLE inventory ADD COLUMN version INTEGER NOT NULL DEFAULT 0;
```

```javascript
// optimistic.js — inventory decrement via compare-and-swap on a version column.
import pg from 'pg';
const pool = new pg.Pool({ connectionString: process.env.DATABASE_URL });
const sleep = (ms) => new Promise((r) => setTimeout(r, ms));
const MAX_ATTEMPTS = 5;

async function reserveStockOptimistic(productId, qty) {
  for (let attempt = 1; attempt <= MAX_ATTEMPTS; attempt++) {
    const client = await pool.connect();
    try {
      await client.query('BEGIN');

      // No lock. Read the current state AND its version stamp.
      const { rows } = await client.query(
        `SELECT stock, version FROM inventory WHERE id = $1`, [productId]
      );
      if (rows.length === 0) throw new Error('PRODUCT_NOT_FOUND');

      const { stock, version } = rows[0];
      if (stock < qty) throw new Error('OUT_OF_STOCK');

      // THE COMPARE-AND-SWAP. `WHERE version = $3` is the whole trick: if another
      // transaction committed since our SELECT, it bumped the version, this WHERE
      // matches nothing, and rowCount === 0.
      const res = await client.query(
        `UPDATE inventory SET stock = $1, version = version + 1
          WHERE id = $2 AND version = $3`,
        [stock - qty, productId, version]
      );

      if (res.rowCount === 0) {
        // Somebody beat us; our read is stale. Throw it all away and retry with a
        // FRESH read — never reuse the stale `stock` value.
        await client.query('ROLLBACK');
        client.release();
        await sleep(2 ** attempt * 10 + Math.random() * 20); // backoff + jitter
        continue;
      }

      await client.query('COMMIT');
      return { ok: true, remaining: stock - qty, attempts: attempt };
    } catch (err) {
      await client.query('ROLLBACK');
      client.release();
      throw err;
    }
  }
  throw new Error('TOO_MUCH_CONTENTION');
}
```

#### The third option: don't read-modify-write at all

Often the problem evaporates if you let the **database** do the arithmetic atomically:

```sql
-- No lock, no version, no retry. Correct at ANY isolation level.
-- The WHERE clause is the safety check; rowCount === 0 means "out of stock".
UPDATE inventory SET stock = stock - $1 WHERE id = $2 AND stock >= $1;
```

This is safe because a single `UPDATE` holds a row lock for its own duration and evaluates `stock - $1` against the **current** value — not a stale one you read a network round-trip ago. **Reach for this first.** Use pessimistic/optimistic locking only when the decision logic won't fit in one statement.

#### When to use which

| Situation | Use |
|---|---|
| High contention (everyone on one hot row — flash sale, one concert seat) | **Pessimistic.** Optimistic degenerates into a retry storm where nobody progresses. |
| Low contention (editing your own profile, one order) | **Optimistic.** Locks would be pure overhead. |
| Short transaction (a few ms, no network calls inside) | **Pessimistic** is safe — the lock is held briefly. |
| Long transaction, or a human "thinking" (an edit form open 5 minutes) | **Optimistic**, always. Never hold a DB lock across a human. |
| Stateless HTTP across services, no open transaction | **Optimistic** — the version is your only option (send it as an ETag). |
| The whole operation is one arithmetic update | **Neither** — atomic `UPDATE ... SET x = x - 1 WHERE x >= 1`. |

**Rule of thumb:** *pessimistic for high contention and short transactions; optimistic for low contention and long or stateless ones.*

---

### 6. Deadlocks inside the database

A **deadlock** is a cycle of waiting: T1 holds a lock T2 wants, and T2 holds a lock T1 wants. Neither can proceed. Ever. Recall from [52 — Concurrency Fundamentals] that this is the same deadlock you know from threads and mutexes — the database version just uses **row locks**.

Two transfers running in opposite directions:

| Time | T1 (Alice → Bob) | T2 (Bob → Alice) |
|---|---|---|
| 1 | `BEGIN` | `BEGIN` |
| 2 | `UPDATE accounts ... WHERE id='alice'` → **locks alice** | |
| 3 | | `UPDATE accounts ... WHERE id='bob'` → **locks bob** |
| 4 | `UPDATE accounts ... WHERE id='bob'` → **WAITS** (T2 holds bob) | |
| 5 | | `UPDATE accounts ... WHERE id='alice'` → **WAITS** (T1 holds alice) |
| 6 | ⟳ cycle — deadlock | ⟳ cycle — deadlock |

**How the database handles it:** it doesn't hang. A background **deadlock detector** builds a *wait-for graph* (who waits on whom) and looks for a cycle. Postgres runs it after `deadlock_timeout` (default **1 second**) of waiting. On finding a cycle it picks a **victim** and kills it:

```
ERROR:  deadlock detected
DETAIL: Process 1234 waits for ShareLock on transaction 5678; blocked by process 5678.
        Process 5678 waits for ShareLock on transaction 1234; blocked by process 1234.
        (SQLSTATE 40P01)
```

The victim is **rolled back entirely**; the survivor proceeds.

**What YOUR code must do: catch the error and RETRY.** A deadlock is not a bug in your data — it's a transient scheduling collision. The correct response is almost always "try again."

```javascript
// withTxRetry.js — wrap ANY transaction that might deadlock or hit a serialization failure.
const RETRYABLE = new Set([
  '40P01', // deadlock_detected
  '40001', // serialization_failure (Serializable / Repeatable Read conflicts)
]);

/**
 * `fn` MUST be safe to re-run: it re-reads everything it needs. Never carry
 * state from a failed attempt into the next one.
 */
export async function withTxRetry(pool, fn, { maxAttempts = 5, isolation = 'READ COMMITTED' } = {}) {
  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    const client = await pool.connect();
    try {
      await client.query(`BEGIN ISOLATION LEVEL ${isolation}`);
      const result = await fn(client);
      await client.query('COMMIT');
      return result;
    } catch (err) {
      await client.query('ROLLBACK').catch(() => {});
      if (!RETRYABLE.has(err.code) || attempt === maxAttempts) throw err;

      // Jitter is NOT optional: without it both victims retry at the same instant
      // and deadlock again immediately.
      await new Promise((r) => setTimeout(r, Math.min(2 ** attempt * 10, 500) + Math.random() * 50));
    } finally {
      client.release();
    }
  }
}

// Usage:
await withTxRetry(pool, async (client) => {
  await client.query(`UPDATE accounts SET balance = balance - 5000 WHERE id='alice'`);
  await client.query(`UPDATE accounts SET balance = balance + 5000 WHERE id='bob'`);
});
```

**How to make deadlocks rare in the first place:**

1. **Acquire locks in a consistent global order.** Sort the two account IDs and lock the lexicographically smaller one first. Both transactions then lock `alice` before `bob`, and the cycle can never form. This one trick eliminates most application deadlocks.
2. **Keep transactions short.** Shorter lock hold = smaller window for a cycle.
3. **Touch fewer rows.** An `UPDATE ... WHERE status='x'` across 10,000 rows takes 10,000 locks and is a deadlock magnet. Batch it.
4. **Don't do a bare `SELECT` then `UPDATE`** in Repeatable Read — the lock upgrade is a classic cycle source. Use `SELECT ... FOR UPDATE` from the start.

---

### 7. Long-running transactions: the silent production killer

An open transaction that sits idle is one of the most damaging things that can happen to your database — and it's shockingly easy to cause from Node:

```javascript
// THE BUG. Do not do this.
await client.query('BEGIN');
const order = await client.query('SELECT * FROM orders WHERE id=$1 FOR UPDATE', [id]);

// A network call to Stripe. 200ms on a good day. 30 SECONDS on a bad one. Possibly
// hangs until your HTTP timeout. THE TRANSACTION IS OPEN THE ENTIRE TIME.
const charge = await stripe.charges.create({ amount: order.total });

await client.query('UPDATE orders SET status=$1 WHERE id=$2', ['paid', id]);
await client.query('COMMIT');
```

Three disasters flow from this:

1. **It holds row locks for the whole network call.** Every other request touching that order queues behind Stripe's latency. Under load your connection pool exhausts and the entire API goes down — because of *one slow third party*.
2. **It blocks VACUUM globally.** Vacuum can't reclaim any dead tuple newer than the **oldest running transaction's snapshot**. One idle-in-transaction connection open for four hours means **four hours of dead tuples pile up across every table in the database.** Tables bloat, indexes bloat, unrelated queries slow down — and the cause is invisible unless you know to look.
3. **It burns a pooled connection** for its entire lifetime. See [65 — Database Connection Pooling].

**The fix:** move the external call **outside** the transaction. Transaction 1 marks intent, you commit, you call Stripe, transaction 2 records the result. That's the **Saga / outbox** shape — see [77 — Distributed Transactions].

```javascript
// GOOD: two short transactions with the slow work in between.
await withTxRetry(pool, (c) =>
  c.query(`UPDATE orders SET status='charging' WHERE id=$1 AND status='pending'`, [id]));

const charge = await stripe.charges.create({ amount, idempotencyKey: `order-${id}` });

await withTxRetry(pool, (c) =>
  c.query(`UPDATE orders SET status='paid', charge_id=$2 WHERE id=$1`, [id, charge.id]));
```

**Your production guardrails:**

```sql
ALTER SYSTEM SET idle_in_transaction_session_timeout = '30s';  -- kill idle-in-txn sessions
ALTER SYSTEM SET statement_timeout = '30s';                    -- kill runaway statements

-- Find the offenders RIGHT NOW:
SELECT pid, state, now() - xact_start AS txn_age, left(query, 60) AS query
FROM pg_stat_activity
WHERE state IN ('idle in transaction', 'active')
  AND xact_start < now() - interval '1 minute'
ORDER BY xact_start;
```

---

## Visual / Diagram description

### Diagram 1: The Write-Ahead Log commit path

```
   Client                Postgres backend                   Disk
     │                          │                             │
     │  BEGIN                   │                             │
     ├─────────────────────────▶│                             │
     │  UPDATE alice -5000      │  ┌───────────────────┐      │
     ├─────────────────────────▶├─▶│ Shared Buffers    │      │
     │  UPDATE bob   +5000      │  │ (data pages, RAM) │      │
     ├─────────────────────────▶├─▶│ page7  DIRTY ●    │      │
     │                          │  │ page9  DIRTY ●    │      │
     │                          │  └─────────┬─────────┘      │
     │                          │            │                │
     │                          │  ┌─────────▼─────────┐      │
     │                          │  │  WAL buffer (RAM) │      │
     │                          │  │ LSN1001 page7 old→new    │
     │                          │  │ LSN1002 page9 old→new    │
     │  COMMIT                  │  │ LSN1003 COMMIT    │      │
     ├─────────────────────────▶│  └─────────┬─────────┘      │
     │                          │            │  ① fsync()     │
     │                          │            ├───────────────▶│ ┌──────────────┐
     │                          │            │                │ │  pg_wal/     │
     │                          │            │◀───── OK ──────┤ │ (SEQUENTIAL) │
     │  ◀───── "COMMIT OK" ─────┤ ② DURABLE NOW              │ └──────────────┘
     │                          │                             │
     │                          │  ③ later, at a CHECKPOINT:  │
     │                          ├────── flush dirty pages ───▶│ ┌──────────────┐
     │                          │        (RANDOM, in bulk)    │ │ base/ (heap) │
     │                          │                             │ └──────────────┘
```

**What it shows.** `COMMIT` does **not** wait for the data pages — those are still dirty in RAM at step ②. It waits only for the **WAL fsync** at step ①. That single sequential write is the durability boundary: everything before it survives a crash; everything after it is an optimization. Die between ② and ③ and recovery reads `pg_wal/`, finds LSN 1003 (the COMMIT record), and **replays** 1001–1002 to rebuild pages 7 and 9. Find WAL records with **no** COMMIT record and it **undoes** them — that's atomicity.

### Diagram 2: MVCC — one row, three versions, three snapshots

```
                       TIME ──────────────────────────────────────▶

   ROW VERSIONS on disk (the "heap"):
   ┌──────────────────────┐┌──────────────────────┐┌──────────────────────┐
   │ v1  xmin=100         ││ v2  xmin=205         ││ v3  xmin=310         │
   │     xmax=205         ││     xmax=310         ││     xmax=0  (LIVE)   │
   │     balance = 10000  ││     balance =  5000  ││     balance =  7500  │
   └──────────────────────┘└──────────────────────┘└──────────────────────┘
              ▲                       ▲                        ▲
              │                       │                        │
      ┌───────┴───────┐       ┌───────┴───────┐        ┌───────┴───────┐
      │  Reader A     │       │  Reader B     │        │  Reader C     │
      │  snapshot@150 │       │  snapshot@280 │        │  snapshot@400 │
      │  sees 10000   │       │  sees 5000    │        │  sees 7500    │
      └───────┬───────┘       └───────────────┘        └───────────────┘
              │
              └──▶ Reader A is a 4-hour analytics query. It PINS v1 and v2:
                   VACUUM cannot reclaim them, because A might still read them.
                   ──▶ BLOAT across the whole database.

   Nobody blocks anybody. A, B and C all read concurrently while writers
   205 and 310 create new versions alongside them.
```

**What it shows.** Three readers, three snapshots, three different answers — every one **correct for its own consistent point in time**. No reader took a lock; no writer waited. That is the whole magic of MVCC. And the caption on Reader A is the whole cost: a long-lived snapshot **pins old versions alive**, which is exactly why one forgotten `BEGIN` bloats your entire database.

---

## Real world examples

### 1. PostgreSQL — MVCC + SSI, and the VACUUM tax

Postgres implements MVCC exactly as diagrammed: every `UPDATE` writes a new tuple and marks the old one dead. It defaults to **Read Committed**, offers snapshot isolation under the name **Repeatable Read**, and implements true **Serializable Snapshot Isolation** — an optimistic algorithm that tracks read/write dependencies between transactions and aborts one with SQLSTATE `40001` when it detects a structure that could produce write skew. The price of this design is **autovacuum**: Postgres shops at scale end up tuning `autovacuum_vacuum_cost_limit` and per-table vacuum thresholds, and most have had at least one outage traced to table bloat caused by an idle-in-transaction connection.

### 2. MySQL / InnoDB — Repeatable Read plus gap locks

InnoDB also uses MVCC (via undo logs in the rollback segment) but defaults to **Repeatable Read** — a stronger default than Postgres. To block phantoms it adds **gap locks** and **next-key locks**: when you `SELECT ... FOR UPDATE` over a range, InnoDB locks not just the matching rows but the *gaps between index entries*, so nobody can insert a phantom into your range. That makes InnoDB's Repeatable Read close to serializable for range queries — at the cost of much more lock contention and, famously, more deadlocks. If you've seen `Deadlock found when trying to get lock; try restarting transaction` in a Rails or Django log, you've met a gap lock.

### 3. Booking systems (Ticketmaster-class seat reservation)

Selling a specific seat is the canonical high-contention problem: thousands of people want row 12, seat 4, in the same millisecond. These systems converge on **pessimistic locking with a very short critical section plus a reservation TTL**: `SELECT ... FOR UPDATE` on the seat row (often `FOR UPDATE SKIP LOCKED` so different users get handed different seats instead of queuing), flip the status to `HELD` with an `expires_at` ten minutes out, `COMMIT` immediately — and then run the slow payment flow entirely **outside** the transaction, with a sweeper job releasing expired holds. Optimistic locking is the wrong tool here: with 5,000 people CAS-ing one row, nearly every attempt fails and retries, and the retry storm is worse than the lock queue.

---

## Trade-offs

| Isolation level | You gain | You give up |
|---|---|---|
| **Read Committed** | Best throughput; no serialization failures to handle | Non-repeatable reads, phantoms, and **lost updates** — you must handle races yourself |
| **Repeatable Read** (snapshot) | A stable view for the whole transaction; phantoms and lost updates gone in Postgres | **Write skew still possible**; you must retry on `40001` |
| **Serializable** | All five anomalies gone; you can reason as if transactions ran one at a time | Throughput under contention; **mandatory retry logic**; abort storms on hot rows |

| Locking strategy | Pros | Cons |
|---|---|---|
| **Pessimistic (`FOR UPDATE`)** | Guaranteed progress; no wasted work; simple mental model | Blocks others; deadlock risk; disastrous if the transaction is long |
| **Optimistic (`version` CAS)** | No locks; works across stateless HTTP; scales beautifully at low contention | Wasted work on conflict; retry storms under high contention; needs a schema column |
| **Atomic `UPDATE ... WHERE`** | Simplest, fastest, correct at any isolation level | Only works when the decision fits in one SQL statement |

| Durability setting | Pros | Cons |
|---|---|---|
| `synchronous_commit = on` (default) | Committed means committed; survives power loss | Every commit pays an fsync (~0.1–2 ms) |
| `synchronous_commit = off` | 10×+ higher commit throughput | You can lose the last ~200 ms of "committed" transactions on a crash |

**The sweet spot:** Keep the **Read Committed** default. Make every read-modify-write path safe *explicitly* — first try to express it as one atomic `UPDATE ... WHERE`; if you can't, use `SELECT ... FOR UPDATE` for hot rows and an optimistic `version` column for everything else. Escalate a *specific* transaction to `SERIALIZABLE` only when you have a real multi-row invariant that write skew can break (the doctor case) — and wrap it in retry logic. Above all: **keep transactions short, and never make a network call inside one.**

---

## Common interview questions on this topic

### Q1: "Explain ACID. For Atomicity and Durability, tell me *how* the database actually does it."
**Hint:** Atomicity comes from the **undo log** — before modifying a row the DB records the old value; a rollback (or crash recovery) replays it backwards. Durability comes from **write-ahead logging** — the log record is written and `fsync`ed to disk *before* the data pages and *before* COMMIT returns to the client, so a crash can replay the log and reconstruct the pages. The key insight: the WAL is a **sequential** write (fast) while data pages are **random** writes (slow), which is why WAL is both safer *and* faster than flushing pages on commit. Consistency is mostly *your* job (constraints + invariants); Isolation is locks or MVCC.

### Q2: "What's the difference between a non-repeatable read and a phantom read?"
**Hint:** Non-repeatable read = **the same row's value changed** between two reads in one transaction (someone `UPDATE`d it). Phantom read = **the set of rows matching a query changed** (someone `INSERT`ed or `DELETE`d). The distinction matters because a row-level lock stops the first but not the second — you can't lock a row that doesn't exist yet. Blocking phantoms needs range/gap locks (InnoDB) or snapshots/predicate locks (Postgres).

### Q3: "Two users click 'buy the last ticket' at the same time. What goes wrong, and give me three fixes."
**Hint:** Name the anomaly: **lost update**. Draw the interleaving — both `SELECT stock` → both see 1 → both compute 0 → both `UPDATE stock = 0` → two tickets sold. Fixes in order of preference: (1) **atomic update** — `UPDATE inventory SET stock = stock - 1 WHERE id=? AND stock >= 1`, check `rowCount`; (2) **pessimistic** — `SELECT ... FOR UPDATE` so the second request blocks; (3) **optimistic** — a `version` column with `UPDATE ... WHERE version = ?`, retry when 0 rows. For *this* case (high contention on one row) pessimistic beats optimistic — optimistic would cause a retry storm.

### Q4: "Postgres's Repeatable Read prevents phantom reads. So is it the same as Serializable?"
**Hint:** No — this is the snapshot-isolation subtlety. Postgres's "Repeatable Read" is **snapshot isolation**: one consistent snapshot for the whole transaction, so repeated range queries return the same rows (no phantoms), and lost updates are caught as write-write conflicts. But SI **cannot detect write skew**, because the two transactions write *different* rows and never conflict. Give the doctor example: both read "2 doctors on call," both decide it's safe, each updates their own row, zero doctors remain. Only `SERIALIZABLE` (Postgres SSI, which tracks read-write dependencies) catches it — by **aborting** one transaction, so you must retry on SQLSTATE `40001`.

### Q5: "Your service is throwing 'deadlock detected' in production. What do you do?"
**Hint:** Two answers, both needed. **Immediately:** catch SQLSTATE `40P01` and **retry** the whole transaction with exponential backoff *and jitter* (without jitter both victims retry in lockstep and deadlock again). A deadlock is transient, not a data bug. **Structurally:** eliminate the cycle — acquire locks in a **consistent global order** (e.g. always lock account IDs sorted ascending, so two opposing transfers can't form a cycle), shorten transactions, touch fewer rows per statement.

### Q6: "Why is a long-running transaction dangerous, even a read-only one?"
**Hint:** Under MVCC, VACUUM can't reclaim any dead tuple an open snapshot might still need. One transaction open for hours **pins the vacuum horizon for the entire database** — dead tuples pile up, tables and indexes bloat, unrelated queries slow down. Write transactions additionally hold row locks the whole time and burn a pooled connection. The classic bug is calling a payment API inside an open transaction. The fix: `idle_in_transaction_session_timeout`, `statement_timeout`, and moving external calls out of the transaction.

---

## Practice exercise

### Build and Break a Concert Ticket Booking Endpoint

**Setup (5 min).** In Postgres:

```sql
CREATE TABLE seats (
  id         INT PRIMARY KEY,
  status     TEXT NOT NULL DEFAULT 'free',   -- 'free' | 'booked'
  booked_by  TEXT,
  version    INT  NOT NULL DEFAULT 0
);
INSERT INTO seats (id) VALUES (1);   -- exactly ONE seat. That's the point.
```

**Part A — reproduce the bug (10 min).** Write `bookSeatNaive(userId)` in Node with `pg`: `BEGIN`, `SELECT status FROM seats WHERE id=1`, check `status === 'free'` **in JavaScript**, then `UPDATE seats SET status='booked', booked_by=$1 WHERE id=1`, `COMMIT`. Now fire 50 concurrent calls:

```javascript
const results = await Promise.allSettled(
  Array.from({ length: 50 }, (_, i) => bookSeatNaive('user' + i))
);
console.log('successes:', results.filter(r => r.status === 'fulfilled').length);
```

**You must see more than one success.** Write the number down. You have just reproduced a lost update in your own terminal.

**Part B — fix it three ways (20 min).**
1. `bookSeatPessimistic` — add `FOR UPDATE` to the `SELECT`.
2. `bookSeatOptimistic` — add `AND version = $2` to the `UPDATE`, bump the version, and retry (max 5 attempts, exponential backoff + jitter) when `rowCount === 0`.
3. `bookSeatAtomic` — one statement: `UPDATE seats SET status='booked', booked_by=$1 WHERE id=1 AND status='free'`; treat `rowCount === 0` as "already taken."

Run the same 50-concurrent test against each. **All three must report exactly 1 success and 49 failures.**

**Part C — measure and reason (10 min).** Time all four variants, then answer in writing:
- Which fix was **fastest** under 50-way contention on a single row, and why?
- How many retries did the optimistic version burn in total? What does that tell you about using optimistic locking for a flash sale?
- Now seed **500 seats** and have each of 50 users book a *random* seat. Re-run. Does your answer about the best strategy **flip**? Explain why.

**Deliverable:** one file, four functions, a table of results (successes / total time / total retries) for each strategy at both contention levels, and a three-sentence conclusion.

---

## Quick reference cheat sheet

- **Transaction** = a group of operations that all succeed or all fail. `BEGIN ... COMMIT` / `ROLLBACK`.
- **Atomicity** is delivered by the **undo log** — the DB records old values and replays them backwards on rollback or crash recovery.
- **Durability** is delivered by the **WAL** — write the log record to disk and **fsync it BEFORE** the data pages, and before COMMIT returns. Sequential log write = fast; data pages flushed lazily at a checkpoint.
- **Consistency** is mostly *your* job — constraints (`CHECK`, `UNIQUE`, `FK`) plus application invariants. The DB won't invent your business rules.
- **The five anomalies:** dirty read (see uncommitted data), non-repeatable read (same row, different value), phantom read (same query, new rows), lost update (two read-modify-writes, one vanishes), write skew (both read a valid state, both write *different* rows, invariant breaks).
- **Isolation levels:** Read Uncommitted ⊂ Read Committed ⊂ Repeatable Read ⊂ Serializable — each prevents strictly more anomalies at strictly more cost.
- **Defaults — know these cold:** **Postgres = Read Committed. MySQL InnoDB = Repeatable Read.**
- **Postgres's "Repeatable Read" is snapshot isolation** — it prevents phantoms and lost updates, but **NOT write skew**. Only `SERIALIZABLE` (SSI) catches write skew, and it does so by aborting — so retry on `40001`.
- **Lost update happens at the default isolation level.** Any `SELECT` → compute in JS → `UPDATE` is a race unless you protect it.
- **MVCC** = keep multiple row versions, so **readers never block writers and writers never block readers**. The cost is **dead tuples** and **VACUUM**.
- **Pessimistic** = `SELECT ... FOR UPDATE` (others wait). Use for **high contention, short transactions**.
- **Optimistic** = `version` column + `UPDATE ... WHERE version = ?`; `rowCount === 0` means you lost — **retry with a fresh read**. Use for **low contention** and stateless flows.
- **Best of all:** if it fits in one statement, use an atomic `UPDATE t SET x = x - 1 WHERE id = ? AND x >= 1` — correct at any isolation level.
- **Deadlocks are normal.** The DB detects the cycle and kills a victim (`40P01`). **Your code must catch and RETRY** with backoff + jitter. Prevent them by locking rows in a consistent global order.
- **Never make a network call inside an open transaction.** It holds locks, burns a connection, and **blocks VACUUM database-wide**. Set `idle_in_transaction_session_timeout` and `statement_timeout`.

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [65 — Database Connection Pooling](./65-database-connection-pooling.md) — a transaction pins one pooled connection for its entire life, which is why long transactions exhaust the pool |
| **Next** | [77 — Distributed Transactions](./77-distributed-transactions.md) — what happens to ACID when the data lives in two databases and you can't just `COMMIT` |
| **Related** | [08 — Consistency Models](./08-consistency-models.md) — isolation is single-node consistency; this is the distributed generalization of the same idea |
| **Related** | [52 — Concurrency Fundamentals](./52-concurrency-fundamentals.md) — race conditions, locks and deadlocks in your own process; the DB versions are the same beasts with row locks |
| **Related** | [84 — Distributed Locking](./84-distributed-locking.md) — when `SELECT ... FOR UPDATE` isn't enough because the resource lives outside the database |
| **Related** | [62 — Database Indexing](./62-database-indexing.md) — which rows get locked depends on which index the query uses; a missing index can turn one row lock into a scan's worth of locks |
