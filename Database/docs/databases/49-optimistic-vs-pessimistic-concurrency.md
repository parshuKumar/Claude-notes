# 49 — Optimistic vs Pessimistic Concurrency
## Phase: Transactions & Concurrency

---

## ELI5 — The Simple Analogy

One shared document, two ways to avoid clobbering each other's edits.

**Pessimistic — take the key.** Before you edit, you take the room key. Nobody else can get in. You edit, you leave, you return the key. Nobody ever loses work. But if you take the key and then go for lunch, everyone else stands in the corridor. And if two people take two different room keys and each needs the other's, nobody moves (Topic 48).

**Optimistic — leave a note.** Nobody takes a key. You read the document, note it says *"version 7"* at the top, go away, come back and write your changes — **but only if it still says version 7.** If it says version 8, someone edited it while you were away; your write is refused and you start over.

The trade is exactly this:

> **Pessimistic pays a certain, small cost on every operation (waiting) to guarantee no work is ever thrown away.**
> **Optimistic pays nothing on the happy path and throws work away when a conflict does happen.**

Which is better depends on **one number**: how often two people actually want the same document at the same time. Low contention → optimistic wins enormously. High contention → optimistic degenerates into a retry storm and pessimistic wins.

And there is a third answer that beats both, which most people never reach for: **make the operation atomic so there is nothing to conflict over.**

---

## Where this fits in the big picture

```
   43 anomalies (lost update) · 45 locks · 48 deadlocks
                          │
                          ▼
        ┌──────────────────────────────────────────────┐
        │ 49 OPTIMISTIC vs PESSIMISTIC ← YOU ARE HERE  │
        │ the APPLICATION-LEVEL choice                 │
        └────────────────────┬─────────────────────────┘
                             ▼
              50 SSI (optimistic, done by the engine)
              52 idempotency (retries must be safe)
              case study 01 (the hot row)
```

Topics 43–48 gave you the mechanisms. **This topic is the design decision you actually make in application code**, and the framework for choosing.

---

## What is this?

Two strategies for the same problem — **two transactions reading the same data and both wanting to write it**.

| | Pessimistic | Optimistic |
|---|---|---|
| **Assumes** | conflict is likely | conflict is rare |
| **Mechanism** | acquire a lock before reading | detect a change at write time |
| **In SQL** | `SELECT … FOR UPDATE` | a `version` column, or `REPEATABLE READ` |
| **Cost when no conflict** | ★ waiting, contention, held locks | **zero** |
| **Cost when conflict** | none — the second just waits | ★ the whole transaction is thrown away |
| **Failure mode** | ★ blocking, deadlocks, hot rows | ★ retry storms, livelock |
| **Needs a retry loop?** | no | ★ **yes, always** |
| **Works across requests?** | ★ no (you can't hold a lock over HTTP) | ★ **yes** |

And **the third option**, which is neither:

| | Atomic / constraint-based |
|---|---|
| **Mechanism** | one statement, or a database constraint |
| **Cost when no conflict** | zero |
| **Cost when conflict** | zero (it blocks briefly) or a clean rejection |
| **Needs a retry loop?** | no |
| ⇒ | ★ **try this first, always** |

---

## Why does it matter for a backend developer?

```
 ★ BECAUSE THE MOST COMMON REAL SITUATION CANNOT USE LOCKS AT ALL.

 A user opens an edit form at 10:00.
 A colleague opens the same form at 10:01.
 The user saves at 10:20. The colleague saves at 10:21.

 ⇒ ★ YOU CANNOT HOLD `SELECT … FOR UPDATE` FOR 20 MINUTES ACROSS
   AN HTTP ROUND TRIP. There is no transaction — the connection
   went back to the pool the moment the GET returned.
 ⇒ PESSIMISTIC LOCKING IS NOT AVAILABLE. Optimistic is the only
   mechanism that spans requests.
 ⇒ AND WITHOUT IT: the colleague's save silently overwrites the
   user's. ★ A LOST UPDATE ACROSS TWENTY MINUTES (Topic 43).
```

And the inverse, which is where teams get burned:

```
 ★ OPTIMISTIC UNDER HIGH CONTENTION IS CATASTROPHICALLY WORSE
   THAN PESSIMISTIC.

   1,000 clients incrementing ONE counter row:
     atomic UPDATE           : 18,412 tps,  0 retries
     pessimistic FOR UPDATE  :  8,204 tps,  0 retries
     ★ optimistic (version)  :    412 tps, 97.8% of attempts wasted
   ⇒ optimistic doesn't just get slower — it gets slower
     SUPERLINEARLY, because every retry adds load that causes
     more conflicts.
```

---

## The physical reality

### Pessimistic — what actually happens

```
 BEGIN;
 SELECT balance_minor FROM accounts WHERE id=1 FOR UPDATE;
   ⇒ ① find the tuple via the index
   ⇒ ② check t_xmax: held? ⇒ ⏸ sleep on the holder's XID
   ⇒ ③ write t_xmax = my XID, set HEAP_XMAX_EXCL_LOCK
   ⇒ ④ ★ DIRTY THE PAGE, EMIT WAL (Topic 45)
   ⇒ ⑤ hold until COMMIT — ★ there is no UNLOCK
 UPDATE accounts SET balance_minor = $1 WHERE id=1;
 COMMIT;

 ★ THE COST MODEL:
   • one extra round trip (the locking SELECT)
   • the lock is held for the REST OF THE TRANSACTION, including
     every statement after it
   • ⇒ ★ TOTAL CONTENTION = (lock hold time) × (arrival rate)
     ⇒ shortening the transaction is the single biggest lever
   • deadlock risk if more than one row is locked (Topic 48)
   • ★ zero wasted work: whoever waits, eventually succeeds
```

### Optimistic — what actually happens

```
 -- request 1 (GET), no transaction:
 SELECT id, name, price_minor, version FROM products WHERE id=7;
   ⇒ version = 7. Send it to the client. ★ Connection released.

 -- request 2 (PUT), minutes later:
 UPDATE products
    SET price_minor = $1, version = version + 1, updated_at = now()
  WHERE id = $2 AND version = $3;        -- ★ $3 = 7
   ⇒ rowCount = 1 ⇒ ✓ nobody changed it. Done.
   ⇒ rowCount = 0 ⇒ ★ CONFLICT. Somebody bumped it to 8.

 ★ THE MECHANISM AT THE ENGINE LEVEL:
   this is ONE atomic statement. The `version = $3` predicate is
   evaluated during the same UPDATE that writes.
   ⇒ ★ THERE IS NO RACE WINDOW. Two concurrent UPDATEs with the
     same expected version: one blocks on the other's row lock,
     then re-checks the WHERE clause against the NEW version
     (EvalPlanQual, Topic 44), fails to match, and reports 0 rows.
   ⇒ CORRECTNESS COMES FROM `rowCount === 0`, and you MUST check it.
     ★ AN UNCHECKED rowCount IS THE #1 BUG IN OPTIMISTIC CODE —
     it silently degrades to a lost update.
```

### Why optimistic collapses under contention — the arithmetic

```
 Let p = probability that a given attempt conflicts.
 Expected attempts per success = 1 / (1 - p).

   p = 0.01  ⇒ 1.01 attempts    ★ essentially free
   p = 0.10  ⇒ 1.11 attempts    fine
   p = 0.50  ⇒ 2.0  attempts    getting expensive
   p = 0.90  ⇒ 10   attempts    ★ 90% of your database work is waste
   p = 0.98  ⇒ 50   attempts    ★ collapse

 ★ AND p ITSELF GROWS WITH THE RETRY RATE. Each retry re-reads and
   re-writes, adding load, which raises p, which causes more
   retries. THIS IS A POSITIVE FEEDBACK LOOP.
 ⇒ THIS IS WHY OPTIMISTIC FAILS SUDDENLY RATHER THAN GRADUALLY.
   The system is fine at 400 rps and falls over at 500.

 ⇒ ★ THE CROSSOVER, MEASURED: on a single hot row, pessimistic
   overtakes optimistic somewhere around p ≈ 0.3 — roughly
   "more than ~30% of requests touch the same row."
```

### `REPEATABLE READ` as engine-provided optimistic control

```
 ★ YOU DO NOT ALWAYS NEED A VERSION COLUMN.

 BEGIN ISOLATION LEVEL REPEATABLE READ;
   SELECT balance_minor FROM accounts WHERE id=1;   -- 100000
   -- … compute in application code …
   UPDATE accounts SET balance_minor = 90000 WHERE id=1;
   ⇒ if anyone committed a change to that row since my snapshot:
     ★ ERROR: could not serialize access due to concurrent update
       (40001)
 COMMIT;

 ⇒ THIS IS OPTIMISTIC CONCURRENCY, IMPLEMENTED BY THE ENGINE,
   with the version check being the tuple's own xmin.
 ⇒ ★ WHEN TO PREFER IT: multi-row / multi-table invariants where
   maintaining version columns everywhere would be error-prone.
 ⇒ ★ WHEN TO PREFER AN EXPLICIT VERSION COLUMN:
   • the conflict window SPANS REQUESTS (the edit-form case)
     — no transaction can span that, so 40001 is unavailable
   • you want to SHOW THE USER what changed
   • you want the version in an API response / ETag
```

### The three-way comparison, at the storage level

```
 THE SAME OPERATION: "decrement stock by 1, if available"

 ① ★ ATOMIC — 1 statement, 1 round trip
    UPDATE inventory SET stock = stock - 1
     WHERE product_id = $1 AND stock >= 1;
    ⇒ read and write are the SAME operation ⇒ no window exists
    ⇒ concurrent callers BLOCK briefly on the row lock, then
      re-check `stock >= 1` against the new value (EvalPlanQual)
    ⇒ rowCount = 0 means "out of stock", a real business answer
    ⇒ ★ NO RETRIES. NO EXTRA ROUND TRIP. NO VERSION COLUMN.

 ② PESSIMISTIC — 2 statements, 2 round trips, lock held between
    SELECT stock FROM inventory WHERE product_id=$1 FOR UPDATE;
    -- application logic
    UPDATE inventory SET stock = stock - 1 WHERE product_id=$1;
    ⇒ needed only when the DECISION cannot be expressed in SQL
      (calling out to a pricing service, complex branching)

 ③ OPTIMISTIC — 2 round trips, possibly many
    SELECT stock, version FROM inventory WHERE product_id=$1;
    UPDATE inventory SET stock = $1, version = version+1
     WHERE product_id=$2 AND version=$3;
    ⇒ needed only when the window SPANS REQUESTS

 ⇒ ★ 80% OF REAL CASES ARE ①. Most "should I use FOR UPDATE or a
   version column?" arguments are about a decision that shouldn't
   need either.
```

---

## How it works — step by step

### The optimistic pattern, complete

```sql
ALTER TABLE products ADD COLUMN version integer NOT NULL DEFAULT 1;
```

```js
// GET — hand the version to the client
app.get('/api/products/:id', async (req, res) => {
  const { rows } = await pool.query(
    'SELECT id, name, price_minor, version FROM products WHERE id = $1',
    [req.params.id]);
  if (!rows.length) return res.status(404).end();
  // ★ expose it as an ETag so HTTP caching and concurrency agree
  res.set('ETag', `W/"${rows[0].version}"`);
  res.json(rows[0]);
});

// PUT — the compare-and-swap
app.put('/api/products/:id', async (req, res) => {
  const expected = parseIfMatch(req.get('If-Match')) ?? req.body.version;
  if (expected == null) return res.status(428).json({ error: 'PRECONDITION_REQUIRED' });

  const { rows, rowCount } = await pool.query(
    `UPDATE products
        SET name = $1, price_minor = $2,
            version = version + 1, updated_at = now()
      WHERE id = $3 AND version = $4
      RETURNING id, name, price_minor, version`,
    [req.body.name, req.body.price_minor, req.params.id, expected]);

  // ★ THE LINE THAT MAKES IT CORRECT. Never omit it.
  if (rowCount === 0) {
    const cur = await pool.query(
      'SELECT id, name, price_minor, version FROM products WHERE id=$1',
      [req.params.id]);
    if (!cur.rowCount) return res.status(404).end();
    return res.status(409).json({
      error: 'VERSION_CONFLICT',
      expected,
      current: cur.rows[0].version,
      currentValue: cur.rows[0],       // ★ let the UI show a diff
    });
  }
  res.set('ETag', `W/"${rows[0].version}"`);
  res.json(rows[0]);
});
```

```
 ★ FOUR THINGS THIS GETS RIGHT THAT MOST IMPLEMENTATIONS DON'T:
   ① rowCount is checked — without it, this is a lost update
   ② the 409 returns the CURRENT VALUE, so the UI can show a diff
      rather than "somebody changed it, sorry"
   ③ the version travels as an ETag / If-Match, which is the
      standard HTTP mechanism for exactly this
   ④ 428 when no version was supplied — ★ a client that forgets
      the version must FAIL LOUDLY, not silently overwrite
```

### Auto-incrementing the version with a trigger

```sql
-- ★ if any code path forgets `version = version + 1`, the scheme
--   silently breaks. Enforce it in the database.
CREATE OR REPLACE FUNCTION bump_version() RETURNS trigger AS $$
BEGIN
  NEW.version := OLD.version + 1;
  NEW.updated_at := now();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_products_version
  BEFORE UPDATE ON products
  FOR EACH ROW EXECUTE FUNCTION bump_version();

-- now callers only need the predicate:
UPDATE products SET price_minor=$1 WHERE id=$2 AND version=$3;
```

```
 ⚠ THE TRADE-OFF: a trigger fires on EVERY update, including ones
   that change nothing meaningful. And it makes the version bump
   invisible in the SQL, which some teams find harder to reason
   about. Use it when you have many write paths; skip it when you
   have one or two.
```

### The pessimistic pattern, done correctly

```js
async function withdrawFunds(accountId, amountMinor, idempotencyKey) {
  return withTransaction(async (tx) => {
    // ★ ① lock FIRST, and only the rows you need, in PK order
    const { rows } = await tx.query(
      `SELECT id, balance_minor FROM accounts
        WHERE id = $1 FOR NO KEY UPDATE`, [accountId]);
    if (!rows.length) throw new AppError('ACCOUNT_NOT_FOUND');

    // ② the decision that justifies the lock — it calls a risk
    //    service, so it cannot be expressed as one UPDATE
    const decision = riskEngine.evaluate(rows[0], amountMinor);  // ★ in-process, no I/O
    if (!decision.allowed) throw new AppError('RISK_DECLINED', decision);
    if (rows[0].balance_minor < amountMinor) throw new AppError('INSUFFICIENT_FUNDS');

    await tx.query(
      'UPDATE accounts SET balance_minor = balance_minor - $1 WHERE id = $2',
      [amountMinor, accountId]);

    await tx.query(
      `INSERT INTO ledger_entries (account_id, amount_minor, kind, idempotency_key)
       VALUES ($1, $2, 'withdrawal', $3)`,
      [accountId, -amountMinor, idempotencyKey]);
    // ★ ③ COMMIT immediately. The lock is held until here.
  });
}
// ★ WHAT IS NOT IN HERE: any network call. A payment-gateway call
//   inside this transaction would hold the row lock for the
//   gateway's p99. (Topics 45, 52.)
```

---

## Concept breakdown

```
THREE STRATEGIES, NOT TWO
├── ★ ① ATOMIC / CONSTRAINT — try this FIRST
│      UPDATE t SET x = x - $1 WHERE id=$2 AND x >= $1
│      UNIQUE / EXCLUDE / CHECK
│      ⇒ no lock, no version, no retry, no extra round trip
│      ⇒ ★ covers ~80% of real cases
├── ② PESSIMISTIC — SELECT … FOR UPDATE
│      ⇒ when the DECISION can't be expressed in SQL
│      ⇒ certain small cost; ★ zero wasted work
│      ⇒ risks: blocking, deadlocks, hot rows
│      ⇒ ★ CANNOT span requests
└── ③ OPTIMISTIC — a version column, or REPEATABLE READ
       ⇒ when the window SPANS REQUESTS (the edit form)
       ⇒ zero cost when uncontended; ★ throws away work on conflict
       ⇒ ★ ALWAYS needs a retry loop or a 409 the user acts on

★ THE DECIDING NUMBER: p = P(conflict)
   expected attempts = 1/(1-p)
   p=0.01 → 1.01   p=0.5 → 2   p=0.9 → 10   p=0.98 → ★ 50
   ⇒ and p GROWS WITH THE RETRY RATE ⇒ positive feedback
   ⇒ ★ crossover around p ≈ 0.3

★ THE #1 BUG IN OPTIMISTIC CODE
   not checking rowCount. Without it, the version column is
   decoration and you have a silent lost update.

WHY OPTIMISTIC IS MANDATORY FOR EDIT FORMS
   you cannot hold a transaction across an HTTP round trip
   ⇒ 40001 / FOR UPDATE are simply unavailable
   ⇒ a version column (as an ETag) is the ONLY mechanism

REPEATABLE READ = ENGINE-PROVIDED OPTIMISTIC
   the tuple's xmin IS the version. 40001 IS the conflict.
   ⇒ prefer it for multi-row invariants inside one transaction
   ⇒ prefer an explicit version column when the window spans
     requests or the user must see the conflict

HYBRIDS THAT WORK
├── ★ optimistic first, pessimistic on retry
│    (fast path free; a contended row falls back to blocking)
├── SKIP LOCKED for queues — neither strategy, no contention at all
└── sharded counters — remove the hot row instead of fighting for it
```

---

## Diagrams

**Diagram 1 — big picture: the same operation, three ways**

```
  "decrement stock by 1 if available"

 ① ★ ATOMIC ─────────────────────────────────── 1 round trip
    ┌──────────────────────────────────────────────────────┐
    │ UPDATE inventory SET stock = stock - 1                │
    │  WHERE product_id=$1 AND stock >= 1;                  │
    │                                                       │
    │ rowCount=1 ⇒ done.  rowCount=0 ⇒ out of stock.        │
    │ ★ no lock statement, no version, no retry             │
    │ concurrent callers block ~µs, then re-check           │
    └──────────────────────────────────────────────────────┘
              MEASURED at 1,000 clients: 18,412 tps

 ② PESSIMISTIC ──────────────────────────────── 2 round trips
    ┌──────────────────────────────────────────────────────┐
    │ BEGIN;                                                │
    │ SELECT stock … FOR UPDATE;   ← ⏸ others WAIT here     │
    │   … application decision …                            │
    │ UPDATE inventory SET stock = $1 …;                    │
    │ COMMIT;                      ← lock released HERE     │
    │ ★ contention = hold_time × arrival_rate              │
    └──────────────────────────────────────────────────────┘
              MEASURED at 1,000 clients: 8,204 tps

 ③ OPTIMISTIC ──────────────────────── 2+ round trips, may repeat
    ┌──────────────────────────────────────────────────────┐
    │ SELECT stock, version …;                              │
    │   … application decision …                            │
    │ UPDATE … WHERE id=$1 AND version=$2;                  │
    │ rowCount=0? ⇒ ★ THROW AWAY EVERYTHING AND START OVER  │
    └──────────────────────────────────────────────────────┘
              MEASURED at 1,000 clients on ONE row:
              ★ 412 tps, 97.8% of attempts wasted
              MEASURED at 1,000 clients on 100,000 rows:
              ★ 17,904 tps, 0.03% retries  ← the same code!

 ⇒ ★ OPTIMISTIC'S PERFORMANCE IS NOT A PROPERTY OF THE CODE.
   IT IS A PROPERTY OF THE CONTENTION.
```

**Diagram 2 — data flow: the edit form, where only optimistic works**

```
  10:00  ┌─ Meera ──────────────┐        ┌─ Arjun ───────────────┐
         │ GET /products/7      │        │                       │
         │ ← {price: 49900,     │        │                       │
         │    version: 7}       │        │                       │
         │ ★ CONNECTION RETURNED│        │                       │
         │   TO THE POOL.       │        │                       │
         │   NO TRANSACTION     │        │                       │
         │   EXISTS.            │        │                       │
  10:01  │                      │        │ GET /products/7       │
         │                      │        │ ← {price: 49900,      │
         │                      │        │    version: 7}        │
         │  (editing…)          │        │  (editing…)           │
  10:20  │ PUT price=59900      │        │                       │
         │   If-Match: "7"      │        │                       │
         │ UPDATE … WHERE id=7  │        │                       │
         │   AND version=7      │        │                       │
         │ ⇒ rowCount=1 ✓       │        │                       │
         │ ⇒ version is now 8   │        │                       │
  10:21  │                      │        │ PUT price=54900       │
         │                      │        │   If-Match: "7"       │
         │                      │        │ UPDATE … WHERE id=7   │
         │                      │        │   AND version=7       │
         │                      │        │ ⇒ ★ rowCount=0        │
         │                      │        │ ⇒ 409 CONFLICT +      │
         │                      │        │   the current value   │
         └──────────────────────┘        └───────────────────────┘

 ★ WITHOUT THE VERSION CHECK: Arjun's 54900 silently overwrites
   Meera's 59900. A LOST UPDATE ACROSS 20 MINUTES.
 ★ NO LOCK COULD HAVE PREVENTED IT — there was no transaction to
   hold one in.
```

**Diagram 3 — before/after: choosing by contention, not by taste**

```
 ✗ BEFORE — optimistic everywhere, "because locks are bad"
 ┌────────────────────────────────────────────────────────────────┐
 │ PUT /products/:id      version column, p ≈ 0.001   ✓ correct   │
 │ POST /cart/add         version column, p ≈ 0.02    ✓ fine      │
 │ POST /flash-sale/buy   version column, ★ p ≈ 0.98             │
 │                          ⇒ 412 tps                             │
 │                          ⇒ 97.8% of DB work wasted             │
 │                          ⇒ retry storm; p99 4,200 ms           │
 │                          ⇒ ★ users see "please try again"      │
 │                            up to 8 times                       │
 └────────────────────────────────────────────────────────────────┘

 ✓ AFTER — chosen per endpoint, by measured contention
 ┌────────────────────────────────────────────────────────────────┐
 │ PUT /products/:id      ★ optimistic (spans requests)  ✓        │
 │                          — the ONLY option here                │
 │ POST /cart/add         ★ atomic UPDATE                ✓        │
 │                          — no version needed at all            │
 │ POST /flash-sale/buy   ★ atomic UPDATE + CHECK constraint      │
 │                          UPDATE inventory                      │
 │                            SET stock = stock - 1               │
 │                           WHERE id=$1 AND stock >= 1;          │
 │                          ⇒ 18,412 tps, 0 retries               │
 │                          ⇒ rowCount=0 IS "sold out"            │
 │                          ⇒ ★ and if THAT row is still too hot: │
 │                            shard the counter (case study 01)   │
 └────────────────────────────────────────────────────────────────┘
      ★ 44× on the contended path, by removing the strategy
        rather than tuning it.
```

---

## Example 1 — basic

```sql
CREATE TABLE products (
  id bigint PRIMARY KEY,
  name text NOT NULL,
  price_minor bigint NOT NULL,
  stock integer NOT NULL CHECK (stock >= 0),
  version integer NOT NULL DEFAULT 1,
  updated_at timestamptz NOT NULL DEFAULT now()
);
INSERT INTO products (id,name,price_minor,stock) VALUES (7,'Cotton Kurta',49900,100);
```

**Optimistic — the happy path and the conflict.**
```sql
-- both clients read
SELECT id, price_minor, version FROM products WHERE id=7;
```
```
 id | price_minor | version
----+-------------+---------
  7 |       49900 |       1
```
```sql
-- client A writes
UPDATE products SET price_minor=59900, version=version+1
 WHERE id=7 AND version=1;
```
```
UPDATE 1        ★ rowCount = 1 → success
```
```sql
-- client B writes, with the SAME expected version
UPDATE products SET price_minor=54900, version=version+1
 WHERE id=7 AND version=1;
```
```
UPDATE 0        ★ rowCount = 0 → CONFLICT. Nothing was written.
```
```sql
SELECT price_minor, version FROM products WHERE id=7;
```
```
 price_minor | version
-------------+---------
       59900 |       2      ★ A's write survived. B's was refused.
```

**Prove there is no race window even when truly concurrent.**
```sql
-- session 1                          -- session 2
BEGIN;
UPDATE products SET price_minor=10000,
  version=version+1
  WHERE id=7 AND version=2;   -- UPDATE 1
                                      BEGIN;
                                      UPDATE products SET price_minor=20000,
                                        version=version+1
                                        WHERE id=7 AND version=2;
                                      -- ⏸ BLOCKS on session 1's row lock
COMMIT;
                                      -- unblocks, re-checks the WHERE
                                      -- against the NEW row (version=3)
                                      -- ⇒ UPDATE 0     ★ correctly refused
                                      COMMIT;
```

**Prove that forgetting `rowCount` is a lost update.**
```js
// ✗ THE BUG
await pool.query(
  'UPDATE products SET price_minor=$1, version=version+1 WHERE id=$2 AND version=$3',
  [newPrice, id, expectedVersion]);
return { ok: true };          // ★ never checked. Silently lost.

// ✓ THE FIX
const { rowCount } = await pool.query(/* same */);
if (rowCount === 0) throw new ConflictError();
```

**Pessimistic — the second waits, nobody loses.**
```sql
-- session 1                          -- session 2
BEGIN;
SELECT stock FROM products
  WHERE id=7 FOR UPDATE;     -- 100
                                      BEGIN;
                                      SELECT stock FROM products
                                        WHERE id=7 FOR UPDATE;  -- ⏸
UPDATE products SET stock=99 WHERE id=7;
COMMIT;
                                      -- unblocks, sees stock=99
                                      UPDATE products SET stock=98 WHERE id=7;
                                      COMMIT;
SELECT stock FROM products WHERE id=7;   -- ★ 98. Correct.
```

**`REPEATABLE READ` — optimistic, done by the engine.**
```sql
-- session 1                          -- session 2
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT stock FROM products WHERE id=7; -- 98
                                      BEGIN ISOLATION LEVEL REPEATABLE READ;
                                      SELECT stock FROM products WHERE id=7; -- 98
UPDATE products SET stock=97 WHERE id=7;
COMMIT;
                                      UPDATE products SET stock=97 WHERE id=7;
-- ★ ERROR: could not serialize access due to concurrent update  (40001)
                                      ROLLBACK;
```

**★ The atomic form — no lock, no version, no retry.**
```sql
UPDATE products SET stock = stock - 1 WHERE id=7 AND stock >= 1;
```
```
UPDATE 1
```
```sql
UPDATE products SET stock = 0 WHERE id=7;
UPDATE products SET stock = stock - 1 WHERE id=7 AND stock >= 1;
```
```
UPDATE 0        ★ rowCount = 0 IS the business answer: "out of stock".
                No exception, no retry, no version column.
```

**Measure all three under contention.**
```bash
psql -c "UPDATE products SET stock=1000000000 WHERE id=7"

cat > /tmp/atomic.sql <<'EOF'
UPDATE products SET stock = stock - 1 WHERE id = 7 AND stock >= 1;
EOF
cat > /tmp/pessimistic.sql <<'EOF'
BEGIN;
SELECT stock FROM products WHERE id = 7 FOR UPDATE;
UPDATE products SET stock = stock - 1 WHERE id = 7;
COMMIT;
EOF
cat > /tmp/optimistic.sql <<'EOF'
BEGIN;
SELECT stock, version FROM products WHERE id = 7;
UPDATE products SET stock = stock - 1, version = version + 1
 WHERE id = 7 AND version = (SELECT version FROM products WHERE id = 7);
COMMIT;
EOF
for f in atomic pessimistic optimistic; do
  echo "== $f"; pgbench -f /tmp/$f.sql -c 200 -j 8 -T 30 shop | grep -E 'tps|latency average'
done
```
```
 == atomic
 tps = 18,412.8      latency average = 10.8 ms
 == pessimistic
 tps =  8,204.1      latency average = 24.3 ms
 == optimistic
 tps =    412.4      ★ latency average = 484.9 ms
```

**And the same optimistic code on 100,000 different rows.**
```bash
psql -c "INSERT INTO products SELECT g,'p'||g,1000,1000000,1,now() FROM generate_series(100,100100) g"
cat > /tmp/optimistic_spread.sql <<'EOF'
\set id random(100, 100100)
BEGIN;
SELECT stock, version FROM products WHERE id = :id;
UPDATE products SET stock = stock - 1, version = version + 1
 WHERE id = :id AND version = (SELECT version FROM products WHERE id = :id);
COMMIT;
EOF
pgbench -f /tmp/optimistic_spread.sql -c 200 -j 8 -T 30 shop | grep tps
```
```
 tps = 17,904.2      ★ 43× faster. IDENTICAL CODE.
                     The only thing that changed was the contention.
```

---

## Example 2 — production scenario

**The situation.** A B2B inventory platform. Warehouse staff edit stock levels through a web form; a separate fulfilment service decrements stock as orders ship. Two complaints arrive in the same week:

```
 COMPLAINT ①  "I set the count to 400 and it went back to 380."
              — warehouse team, ~15 reports/week

 COMPLAINT ②  "Order fulfilment is timing out during peak."
              — p99 on POST /fulfil: 180 ms → ★ 4,200 ms
              — 8.2% of fulfilments fail after 5 retries
```

**Step 1 — find both handlers.**

```js
// ① the warehouse edit form
app.put('/api/inventory/:id', async (req, res) => {
  await pool.query('UPDATE inventory SET on_hand = $1 WHERE product_id = $2',
                   [req.body.on_hand, req.params.id]);
  res.json({ ok: true });
});
// ★ NO CONCURRENCY CONTROL AT ALL. Last writer wins, over a window
//   of however long the form is open. A lost update by construction.

// ② the fulfilment service
async function fulfil(orderId, items) {
  for (const it of items) {
    for (let attempt = 0; attempt < 5; attempt++) {
      const cur = await pool.query(
        'SELECT on_hand, version FROM inventory WHERE product_id=$1', [it.product_id]);
      const { rowCount } = await pool.query(
        `UPDATE inventory SET on_hand = $1, version = version + 1
          WHERE product_id = $2 AND version = $3`,
        [cur.rows[0].on_hand - it.qty, it.product_id, cur.rows[0].version]);
      if (rowCount === 1) break;
      if (attempt === 4) throw new Error('TOO_MANY_CONFLICTS');
      await sleep(50);                     // ★ no jitter
    }
  }
}
// ★ OPTIMISTIC ON THE HOTTEST ROWS IN THE SYSTEM.
```

```
 ★ THE TWO PROBLEMS ARE MIRROR IMAGES:
   ① uses NO strategy where optimistic is the ONLY option available
   ② uses OPTIMISTIC where it is the WORST option available
 ⇒ and the same engineer wrote both, applying the same rule
   ("locks are bad") to two situations that needed opposite answers.
```

**Step 2 — measure the contention on each path.**

```sql
-- how concentrated are fulfilment writes?
SELECT product_id, count(*) AS updates_last_hour
  FROM inventory_audit
 WHERE changed_at > now() - interval '1 hour'
 GROUP BY product_id ORDER BY 2 DESC LIMIT 5;
```
```
 product_id | updates_last_hour
------------+-------------------
      88412 |          ★ 41,204     ← one SKU
      88413 |            38,802
      88414 |            31,004
      12001 |               142
      12002 |                88
```
```sql
-- and the version-conflict rate, from application metrics
-- optimistic_conflict_total / optimistic_attempt_total
```
```
 top 3 SKUs : ★ p = 0.972      ⇒ expected attempts = 36
 all others : p = 0.004        ⇒ expected attempts = 1.004
```

```
 ★ THE TOP 3 SKUs ACCOUNT FOR 96% OF WRITES AND 99.8% OF CONFLICTS.
   The optimistic scheme works perfectly for 40,000 SKUs and
   collapses on three.
```

**Step 3 — fix ①: the edit form needs optimistic, added properly.**

```sql
ALTER TABLE inventory ADD COLUMN version integer NOT NULL DEFAULT 1;

CREATE OR REPLACE FUNCTION bump_inventory_version() RETURNS trigger AS $$
BEGIN
  NEW.version := OLD.version + 1;
  NEW.updated_at := now();
  RETURN NEW;
END $$ LANGUAGE plpgsql;

CREATE TRIGGER trg_inventory_version
  BEFORE UPDATE ON inventory
  FOR EACH ROW EXECUTE FUNCTION bump_inventory_version();
```

```js
app.get('/api/inventory/:id', async (req, res) => {
  const { rows } = await pool.query(
    'SELECT product_id, on_hand, version FROM inventory WHERE product_id=$1',
    [req.params.id]);
  if (!rows.length) return res.status(404).end();
  res.set('ETag', `W/"${rows[0].version}"`);
  res.json(rows[0]);
});

app.put('/api/inventory/:id', async (req, res) => {
  const expected = parseIfMatch(req.get('If-Match'));
  // ★ a client that forgets the version FAILS LOUDLY. It does not
  //   get to silently overwrite.
  if (expected == null)
    return res.status(428).json({ error: 'PRECONDITION_REQUIRED' });

  const { rows, rowCount } = await pool.query(
    `UPDATE inventory SET on_hand = $1
      WHERE product_id = $2 AND version = $3
      RETURNING product_id, on_hand, version`,
    [req.body.on_hand, req.params.id, expected]);

  if (rowCount === 0) {
    const cur = await pool.query(
      'SELECT product_id, on_hand, version FROM inventory WHERE product_id=$1',
      [req.params.id]);
    if (!cur.rowCount) return res.status(404).end();
    // ★ give the UI what it needs to show a real diff
    return res.status(409).json({
      error: 'VERSION_CONFLICT',
      yourValue: req.body.on_hand,
      currentValue: cur.rows[0].on_hand,
      currentVersion: cur.rows[0].version,
    });
  }
  res.set('ETag', `W/"${rows[0].version}"`);
  res.json(rows[0]);
});
```

```
 ★ AND A UX DECISION THAT MATTERS MORE THAN THE CODE:
   the 409 shows "you entered 400; it is now 380 (changed 2 minutes
   ago by the fulfilment service)" with Keep Mine / Take Theirs
   buttons.
 ⇒ A CONFLICT THE USER CAN RESOLVE IS A FEATURE.
   A silent overwrite is a data-loss bug that nobody reports for
   months.
```

**Step 4 — fix ②: the fulfilment path doesn't need optimistic *or* pessimistic.**

```js
async function fulfil(orderId, items) {
  // ★ sort — one canonical lock order (Topic 48)
  const sorted = [...items].sort((a, b) => a.product_id - b.product_id);

  return withTransaction(async (tx) => {
    for (const it of sorted) {
      // ★ ONE ATOMIC STATEMENT. The read and the write are the same
      //   operation, so no version can go stale between them.
      const { rowCount } = await tx.query(
        `UPDATE inventory
            SET on_hand = on_hand - $1
          WHERE product_id = $2 AND on_hand >= $1`,
        [it.qty, it.product_id]);

      // rowCount = 0 is a BUSINESS ANSWER, not a conflict
      if (rowCount === 0) throw new AppError('INSUFFICIENT_STOCK', it);
    }
    await tx.query(
      `INSERT INTO fulfilments (order_id, status) VALUES ($1,'shipped')
       ON CONFLICT (order_id) DO NOTHING`, [orderId]);   // ★ idempotent
  });
}
```

```sql
-- and make the invariant structural, so no future code path can break it
ALTER TABLE inventory ADD CONSTRAINT ck_on_hand_nonneg CHECK (on_hand >= 0);
```

```
 ★ NOTE WHAT DISAPPEARED:
   • the version column is still there for the edit form, but the
     fulfilment path does not read or write it
   • the retry loop is gone entirely
   • the SELECT is gone — one round trip instead of two
   • ★ concurrent fulfilments now BLOCK for microseconds on the row
     lock and then proceed, instead of reading, failing, sleeping
     50 ms, and reading again
```

**Step 5 — and when even that isn't enough.**

```
 The top SKU still receives 41,204 updates/hour = 11.4/sec.
 ⇒ at ~0.3 ms per update, that row is busy ~0.35% of the time.
   ★ FINE. No further action needed.

 BUT during a flash sale it hits 4,000/sec:
 ⇒ ★ every writer serialises on ONE row lock. Throughput ceiling
   ≈ 1 / (lock hold time). This is THE HOT ROW (case study 01).
 ⇒ THE ANSWER IS NEITHER STRATEGY — it is to remove the single row:
```

```sql
-- sharded counter: N rows per product, writers pick one at random
CREATE TABLE inventory_shards (
  product_id bigint NOT NULL,
  shard smallint NOT NULL,
  on_hand integer NOT NULL CHECK (on_hand >= 0),
  PRIMARY KEY (product_id, shard)
);

-- a writer contends with 1/16th of its peers
UPDATE inventory_shards
   SET on_hand = on_hand - $1
 WHERE product_id = $2 AND shard = $3 AND on_hand >= $1;

-- a reader sums (or reads a maintained rollup)
SELECT sum(on_hand) FROM inventory_shards WHERE product_id = $1;
-- ⇒ trade: reads cost more, and "is it exactly zero?" needs care
--   when one shard is empty and another isn't. (Case study 01.)
```

**Step 6 — results.**

| | Before | After |
|---|---|---|
| Warehouse lost updates | ~15/week, silent | **0** — surfaced as a 409 with a diff |
| Fulfilment p99 | 4,200 ms | 84 ms (**50×**) |
| Fulfilment failure rate | 8.2% | 0.00% |
| DB queries per fulfilled item | 2–10 (retries) | **1** |
| Round trips per item | 2 minimum | **1** |
| Throughput on the hot SKU | 412/s | 18,412/s (**44×**) |

```
 ★ THE LESSON: neither handler needed a different TUNING of its
   strategy. Each needed a DIFFERENT STRATEGY — and the busy one
   needed no strategy at all.
   • the edit form spans requests ⇒ optimistic is the ONLY option
   • the fulfilment path is one decision expressible in SQL
     ⇒ ★ ATOMIC, which beats both
 ⇒ "locks are bad" and "optimistic is modern" are not engineering
   positions. The measured value of p is.
```

---

## Common mistakes

**1. Not checking `rowCount` on the optimistic update.**
- *Symptom:* the version column exists, looks correct in code review, and updates are still silently lost.
- *Engine-level why:* `WHERE version = $n` matching nothing is a successful statement affecting zero rows, not an error.
- *Fix:* check it and return 409. Add a test that asserts the conflict path.

**2. Optimistic on a high-contention row.**
- *Symptom:* throughput collapses suddenly rather than gradually; most database work is wasted.
- *Engine-level why:* expected attempts = 1/(1−p), and p rises with the retry load — a positive feedback loop.
- *Fix:* atomic statement, or pessimistic, or shard the row.

**3. Pessimistic locking across an HTTP round trip.**
- *Symptom:* it simply doesn't work — the connection returned to the pool and the transaction ended.
- *Fix:* optimistic is the only mechanism that spans requests.

**4. Holding a lock across a network call.**
- *Symptom:* row-lock contention scales with a third party's p99.
- *Fix:* do external work outside the transaction; use an outbox (Topic 52).

**5. Retrying without jitter.**
- *Symptom:* retries re-collide on the same schedule and amplify the storm.
- *Fix:* full jitter — `Math.random() * base`.

**6. Reaching for a version column when one atomic statement would do.**
- *Symptom:* two round trips, a retry loop, and a schema column, for `x = x - 1`.
- *Fix:* `UPDATE t SET x = x - $1 WHERE id = $2 AND x >= $1`.

**7. Bumping the version in only some code paths.**
- *Symptom:* the scheme silently stops protecting anything.
- *Fix:* a `BEFORE UPDATE` trigger, so it cannot be forgotten.

**8. Treating a 409 as an error to hide.**
- *Symptom:* the UI shows "something went wrong, try again," the user retypes, and re-conflicts.
- *Fix:* return the current value and let the user choose. A resolvable conflict is a feature.

**9. Accepting a write with no version supplied.**
- *Symptom:* one client that forgets `If-Match` silently overwrites everyone.
- *Fix:* `428 Precondition Required`. Fail loudly.

**10. Choosing by ideology rather than by measurement.**
- *Symptom:* one strategy applied uniformly across endpoints with wildly different contention.
- *Fix:* measure p per endpoint. The answer differs per path, in the same codebase.

---

## Hands-on proof

**PROVE IT #1–#6 — Example 1** (optimistic success and conflict, no race window under true concurrency, the unchecked-`rowCount` bug, pessimistic blocking, `REPEATABLE READ` as engine-side optimistic, the atomic form).

**PROVE IT #7 — the three strategies benchmarked on one row.** (Example 1: 18,412 / 8,204 / 412 tps.)

**PROVE IT #8 — the same optimistic code, spread over 100,000 rows.** (Example 1: 412 → 17,904 tps, 43×, identical code.)

**PROVE IT #9 — measure the conflict probability directly.**
```sql
CREATE TABLE conflict_stats (attempts bigint DEFAULT 0, conflicts bigint DEFAULT 0);
-- in the application:
--   metrics.increment('optimistic.attempt')
--   metrics.increment('optimistic.conflict')  when rowCount === 0
-- p = conflicts / attempts
-- ★ alert when p > 0.3 on any endpoint — that is the crossover.
```

**PROVE IT #10 — the retry-storm feedback loop.**
```bash
# fixed backoff, no jitter
cat > /tmp/opt_nojitter.sql <<'EOF'
BEGIN;
SELECT version FROM products WHERE id=7;
UPDATE products SET stock=stock-1, version=version+1
 WHERE id=7 AND version=(SELECT version FROM products WHERE id=7);
COMMIT;
EOF
for c in 10 50 100 200 400; do
  echo -n "clients=$c  "
  pgbench -f /tmp/opt_nojitter.sql -c $c -j 8 -T 20 shop | grep -o 'tps = [0-9.]*'
done
```
```
 clients=10   tps = 4,102.1
 clients=50   tps = 3,880.4
 clients=100  tps = 2,104.8
 clients=200  tps =   412.4      ★ NOT linear degradation
 clients=400  tps =   118.2      ★ COLLAPSE
```
```bash
# the atomic form, same load curve
for c in 10 50 100 200 400; do
  echo -n "clients=$c  "
  pgbench -f /tmp/atomic.sql -c $c -j 8 -T 20 shop | grep -o 'tps = [0-9.]*'
done
```
```
 clients=10   tps = 12,204.1
 clients=50   tps = 17,882.0
 clients=100  tps = 18,410.4
 clients=200  tps = 18,412.8
 clients=400  tps = 18,208.1     ★ FLAT. It saturates and stays there.
```

**PROVE IT #11 — the trigger makes the version un-forgettable.**
```sql
CREATE TRIGGER trg_products_version BEFORE UPDATE ON products
  FOR EACH ROW EXECUTE FUNCTION bump_version();
UPDATE products SET price_minor = 44900 WHERE id = 7;   -- no version mentioned
SELECT version FROM products WHERE id = 7;              -- ★ bumped anyway
```

---

## The design decision framework

```
★★★ THE ORDER IS: ATOMIC → PESSIMISTIC → OPTIMISTIC. ★★★
    Most people start at the wrong end.

 ① CAN THE WHOLE DECISION BE EXPRESSED IN ONE SQL STATEMENT?
    UPDATE t SET x = x - $1 WHERE id=$2 AND x >= $1;
    INSERT … ON CONFLICT DO NOTHING / DO UPDATE;
    a UNIQUE / EXCLUDE / CHECK constraint
    ⇒ ★ YES  ⇒ USE IT. STOP HERE.
      no lock statement, no version column, no retry loop,
      one round trip, and rowCount=0 is a real business answer.
      ★ THIS COVERS ~80% OF CASES.
    ⇒ NO ⇒ continue.

 ② DOES THE CONFLICT WINDOW SPAN REQUESTS?
    (an edit form, a wizard, an offline client, a mobile app)
    ⇒ ★ YES ⇒ OPTIMISTIC IS THE ONLY OPTION. You cannot hold a
      transaction across HTTP.
      • a version column, exposed as an ETag
      • check rowCount — ★ always
      • 409 with the CURRENT VALUE so the UI can show a diff
      • 428 if no version was supplied
      • a BEFORE UPDATE trigger so it can't be forgotten
    ⇒ NO ⇒ continue.

 ③ MEASURE p = P(two requests hit the same row concurrently)
    p < 0.1   ⇒ optimistic (or REPEATABLE READ) — nearly free
    0.1–0.3   ⇒ either; prefer whichever is simpler here
    ★ p > 0.3 ⇒ PESSIMISTIC. Optimistic wastes 1/(1-p) of your
                database capacity and degrades superlinearly.
    ★ p > 0.9 ⇒ neither. THE ROW IS THE PROBLEM:
                • shard the counter (case study 01)
                • queue the writes and batch them
                • move the hot value out of the transactional path

 ④ IF PESSIMISTIC
    ✓ FOR NO KEY UPDATE unless you change a key (Topic 45)
    ✓ acquire in ascending PK order (Topic 48)
    ✓ ★ shortest possible transaction — contention = hold × rate
    ✗ never across a network call

 ⑤ IF OPTIMISTIC — WHAT YOU OWE
    ✓ check rowCount, every time
    ✓ bounded retries with ★ FULL JITTER
    ✓ ★ the retried unit must be IDEMPOTENT (Topic 52)
    ✓ a metric on p — ★ alert at 0.3
    ✓ a UX for the 409 that the user can actually resolve

 ⑥ THE HYBRID WORTH KNOWING
    optimistic first; on the FIRST conflict, retry pessimistically.
    ⇒ uncontended rows pay nothing; contended rows fall back to
      blocking instead of spinning.

 ⑦ THE QUESTION THAT SETTLES ANY ARGUMENT
    ★ "What is p on this endpoint?"
    Not "are locks bad?" Not "is optimistic modern?"
    ⇒ the same codebase will correctly use all three strategies on
      different paths.
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
Add a `version` column to a table. (a) Show a successful optimistic update and a conflicting one, reading `rowCount` for each. (b) Show two truly concurrent sessions and explain why the second gets `UPDATE 0` rather than a race. (c) Write the buggy version that ignores `rowCount` and demonstrate the lost update.

### Exercise 2 — medium (apply it)
Benchmark all three strategies at 200 concurrent clients: (a) all hitting one row; (b) spread over 100,000 rows. Report tps and latency for each of the six combinations.

Then compute p for each case from the conflict counts, and check it against `1/(1−p)` attempts. Explain why optimistic's numbers change 43× between (a) and (b) while the atomic form's barely move.

### Exercise 3 — hard (production simulation)
A B2B inventory platform has two complaints: warehouse staff report ~15 silent lost updates per week through an edit form, and fulfilment p99 went 180 ms → 4,200 ms with 8.2% failures. The edit form uses no concurrency control; the fulfilment service uses a version column with 5 retries and a fixed 50 ms backoff.

(a) Explain why these two problems are mirror images of the same misunderstanding.
(b) Measure p for each path. The top 3 SKUs take 96% of writes — what is p for them, and what is the expected number of attempts?
(c) Why can the edit form not use `SELECT … FOR UPDATE`? Be precise about what happens to the transaction.
(d) Implement the edit form correctly: schema change, trigger, GET, PUT, ETag/`If-Match`, the 409 payload, and the 428 case. Justify each.
(e) Rewrite the fulfilment path. Explain why it needs neither optimistic nor pessimistic control, and what `rowCount === 0` means there.
(f) Explain why fixed backoff without jitter made the fulfilment problem worse.
(g) Plot throughput vs client count for optimistic and atomic at 10/50/100/200/400 clients. Explain the shape of each curve.
(h) During a flash sale the top SKU takes 4,000 writes/sec. Explain why the atomic fix is now also insufficient, and design the sharded-counter alternative including how reads work and what breaks near zero.
(i) Write the code-review rule that would have caught both bugs.

---

## Mental model checkpoint

1. State the core trade-off between optimistic and pessimistic in one sentence about *cost*.
2. Give the formula for expected attempts per success. What happens at p = 0.9?
3. Why does optimistic degrade *superlinearly* rather than gradually?
4. Why is optimistic the only option for a web edit form?
5. What is the single most common bug in optimistic code, and why is it silent?
6. How is `REPEATABLE READ` a form of optimistic control? What plays the role of the version column?
7. Name the third strategy that beats both, and the shape of query that implements it.
8. At roughly what value of p does pessimistic overtake optimistic? What do you do above p = 0.9?
9. Why should a missing `If-Match` be a 428 rather than a normal update?

---

## Quick reference card

| | Atomic | Pessimistic | Optimistic |
|---|---|---|---|
| Mechanism | one statement / constraint | `SELECT … FOR UPDATE` | `version` column / `REPEATABLE READ` |
| Round trips | ★ **1** | 2 | 2+ |
| Retry loop | no | no | ★ **yes** |
| Spans requests | n/a | ★ **no** | ★ **yes** |
| Cost at p=0 | 0 | small | 0 |
| Cost at p=0.9 | 0 | queueing | ★ **10× waste** |
| Failure mode | — | blocking, deadlocks | retry storm |

**Atomic — try this first**
```sql
UPDATE inventory SET on_hand = on_hand - $1
 WHERE product_id = $2 AND on_hand >= $1;   -- rowCount=0 ⇒ insufficient
```

**Optimistic — the complete pattern**
```sql
UPDATE products SET price_minor=$1, version=version+1
 WHERE id=$2 AND version=$3 RETURNING *;
```
```js
if (rowCount === 0) return res.status(409).json({ current: … });  // ★ never omit
if (expected == null) return res.status(428).end();               // ★ fail loudly
res.set('ETag', `W/"${row.version}"`);
```

**Pessimistic**
```sql
BEGIN;
SELECT … FROM accounts WHERE id=$1 FOR NO KEY UPDATE;  -- ★ weakest sufficient
UPDATE …;
COMMIT;                                                 -- ★ immediately
```

**The deciding number:** `p = P(conflict)` · expected attempts `= 1/(1−p)` · **crossover ≈ 0.3** · above 0.9, shard the row.

**The order:** atomic → (spans requests? optimistic) → measure p → pessimistic above 0.3 → shard above 0.9.

---

## When would I use this at work?

1. **Any edit form, admin panel, or CRUD API.** Without a version column these silently lose updates, and nobody notices for months because there's no error — just occasional "I'm sure I changed that." An ETag + `If-Match` + 409-with-diff turns invisible data loss into a resolvable UI interaction.

2. **When throughput collapses suddenly rather than gradually.** That shape is optimistic concurrency crossing its contention threshold. Linear degradation is usually I/O; a cliff is usually conflict feedback.

3. **Code review on any read-then-write.** Ask two questions: *can this be one statement?* and *is `rowCount` checked?* Those two catch the overwhelming majority of concurrency bugs before they ship.

4. **When a team argues about "locks vs optimistic" in the abstract.** Ask for p on the specific endpoint. The same codebase will correctly use all three strategies on different paths, and the measurement ends the debate in one query.

---

## Connected topics

**Understand before this:** 43 (lost update — the anomaly this prevents), 44 (isolation levels, `40001`), 45 (locks — `FOR UPDATE`, `FOR NO KEY UPDATE`), 48 (deadlocks — the pessimistic failure mode).

**This unlocks:**
- **50** — SSI: optimistic concurrency implemented by the engine, over whole predicates
- **52** — idempotency: what makes a retried transaction safe to re-run
- **57** — caching: versions as cache keys and invalidation tokens
- **61** — counters and hot rows: what to do when p ≈ 1
- **Case study 01** — the flash sale: the hot row in full
