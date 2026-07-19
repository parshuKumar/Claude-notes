# Topic 57 — Transaction Patterns in Node.js
### SQL Mastery Curriculum — Phase 8: Transactions and Concurrency in SQL

---

## 1. ELI5 — The Analogy That Makes It Click

Imagine a busy restaurant kitchen with one shared set of tools: knives, cutting boards, the stove. When you get an order to make a dish, you don't just grab whatever is lying around and hope nobody else touches it mid-cook. You **check out** a station from the head chef — "station 3 is yours" — and for the whole time you're cooking that dish, station 3 belongs to you alone.

Now the important part: you announce **"I'm starting"** (BEGIN). You chop, sauté, plate. If everything goes right, you announce **"done, send it"** (COMMIT) and the dish leaves the kitchen as one complete meal. But if halfway through you drop the pan on the floor, you don't send half a meal — you announce **"scrap it"** (ROLLBACK), throw everything away, and the customer never sees the mess. Either the whole dish goes out, or none of it does. There is no "the sauce made it but the protein didn't."

And here's the rule everyone forgets under pressure: **no matter what happens — success, dropped pan, kitchen fire — you must return station 3 to the head chef when you're done.** If you walk away still "holding" the station, the next cook can never use it. Do that a few times and the whole kitchen grinds to a halt with nobody able to cook anything, even though the stations are just sitting there, claimed by cooks who left.

That is exactly transaction management in Node.js with a connection pool:

- **Checking out a station** = `pool.connect()` → you get a dedicated `client`.
- **"I'm starting"** = `BEGIN`.
- **"Send it"** = `COMMIT`. **"Scrap it"** = `ROLLBACK`.
- **Returning the station no matter what** = `client.release()` in a `finally` block.

The entire discipline of this topic is: hold one client for the life of the transaction, commit or roll back as a unit, and **always** give the client back — even when everything is on fire. Forget the last part and you leak connections until the pool is exhausted and your API stops responding, which is one of the most common production outages in Node.js + Postgres systems.

---

## 2. Connection to SQL Internals

A transaction in PostgreSQL is not a Node.js concept — it is a server-side concept that Node merely *drives* by sending SQL commands over a single connection. Understanding what the server does underneath is what separates a developer who "knows to write BEGIN/COMMIT" from one who can reason about why a transaction leaked, deadlocked, or silently lost writes.

**Every transaction lives on exactly one backend process.** PostgreSQL is process-per-connection: when you `pool.connect()`, you are bound to one `postgres` backend process on the server. `BEGIN`, every statement, and `COMMIT`/`ROLLBACK` must travel down that *same* physical connection. This is the entire reason you cannot run a multi-statement transaction with `pool.query()` calls — each `pool.query()` may grab a *different* client from the pool, so your `BEGIN` lands on connection A and your `INSERT` lands on connection B, which knows nothing about the transaction.

**MVCC (Multi-Version Concurrency Control).** When your transaction issues `BEGIN`, PostgreSQL assigns it a snapshot. Every row in Postgres carries hidden system columns `xmin` (the transaction ID that created this row version) and `xmax` (the transaction ID that deleted/superseded it). Your snapshot decides which row versions you can "see." An `UPDATE` never overwrites in place — it writes a **new tuple** and marks the old one's `xmax`. This is why `ROLLBACK` is cheap: the new tuples simply never become visible, and `VACUUM` reclaims them later. Remember from Topic 51 (MVCC) that this is why readers never block writers and writers never block readers.

**The WAL (Write-Ahead Log).** `COMMIT` is the moment durability happens. Before your `COMMIT` returns success, PostgreSQL flushes the transaction's WAL records to disk (subject to `synchronous_commit`). The "D" in ACID is literally an `fsync` of the WAL. A `ROLLBACK` writes an abort record (or nothing meaningful) and the dirty tuples are just abandoned. This is why a transaction that stays open for a long time is expensive: it holds locks, pins its snapshot (blocking `VACUUM` from cleaning up dead tuples system-wide), and keeps a slot in the `pg_xact` (commit log).

**Transaction IDs (XIDs) and the danger of long transactions.** Each transaction that writes gets a 32-bit XID. A single connection holding a transaction open for hours prevents `VACUUM` from advancing the "xid horizon," which in extreme cases risks **transaction ID wraparound**. A leaked client stuck `IN TRANSACTION` is not just a pool problem — it is a server-health problem.

**Locks and the lock table.** `SELECT ... FOR UPDATE`, `UPDATE`, and `DELETE` acquire row-level locks recorded partly in the tuple itself and partly in the shared lock table. Two transactions grabbing the same rows in opposite orders is exactly the deadlock scenario (error `40P01`) the retry section addresses. Serializable isolation adds predicate locks (SIReadLock) and can abort a transaction with `40001` when it detects a serialization anomaly.

So when you write `BEGIN; ... COMMIT;` from Node, you are: pinning a snapshot, writing new tuple versions, accumulating locks, and finally forcing a WAL flush. The Node-side patterns in this topic exist to make sure that server-side lifecycle always terminates cleanly and on the right connection.

---

## 3. Logical Execution Order Context

Transactions do not slot into the single-query logical pipeline (FROM → JOIN → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → window → ORDER BY → LIMIT) the way a clause does. Instead, a transaction is a **wrapper around a sequence of complete statements**, each of which internally runs that full pipeline. The ordering that matters here is the *transaction-level* order:

```
pool.connect()          ← check out ONE client from the pool
  │
  ├── BEGIN              ← snapshot acquired (or deferred to first statement)
  │     └── [ SET TRANSACTION ISOLATION LEVEL ... ]   ← must come right after BEGIN
  │
  ├── statement 1        ← full FROM→...→LIMIT pipeline runs here
  ├── statement 2        ← sees its own uncommitted changes from statement 1
  ├── [ SAVEPOINT sp1 ]  ← optional nested checkpoint
  ├── statement 3
  ├── [ ROLLBACK TO sp1 ]← undo statement 3 only, transaction still alive
  │
  ├── COMMIT  or  ROLLBACK   ← the atomic decision point
  │
  └── client.release()   ← ALWAYS, in finally
```

Three ordering rules that trip people up:

1. **`SET TRANSACTION ISOLATION LEVEL` must be the first statement after `BEGIN`**, before any query touches data. Issue it too late and you get `ERROR: SET TRANSACTION ISOLATION LEVEL must be called before any query`. (You can also bundle it: `BEGIN ISOLATION LEVEL SERIALIZABLE;`.)

2. **A statement sees the effects of earlier statements in the same transaction.** Unlike separate autocommit queries, `INSERT` then `SELECT` inside one transaction: the `SELECT` sees the just-inserted row. This intra-transaction visibility is the whole point of Read Committed's statement-level snapshots (Topic 52).

3. **After any error inside a transaction, the transaction enters the *aborted* state.** Every subsequent statement fails with `current transaction is aborted, commands ignored until end of transaction block` (`25P02`) until you `ROLLBACK` (or `ROLLBACK TO SAVEPOINT`). This is why your `catch` block must issue `ROLLBACK`, and why swallowing an error and trying to continue does not work.

The Node driver does not reorder any of this — it is a faithful pipe. The correctness burden is entirely on you to emit these commands in order on one client and to terminate the block.

---

## 4. What Is a Transaction Pattern in Node.js?

A **transaction pattern** in Node.js is the disciplined code structure that (a) checks out a single dedicated `client` from the connection pool, (b) drives an explicit `BEGIN … COMMIT`/`ROLLBACK` lifecycle on that one client, and (c) guarantees the client is released back to the pool regardless of success or failure. It is the bridge between Postgres's server-side transaction semantics and JavaScript's asynchronous, exception-driven control flow.

The canonical shape using `node-postgres` (`pg`):

```javascript
const client = await pool.connect();   // ── check out ONE client (a dedicated connection)
try {                                  // ── guard the whole lifecycle
  await client.query('BEGIN');         // ── open the transaction on THIS client
  //                    │
  //                    └── every subsequent client.query() runs on the same backend
  await client.query(                  // ── work statement 1 (parameterized)
    'UPDATE accounts SET balance = balance - $1 WHERE id = $2',
    [amount, fromId]
  );
  await client.query(                  // ── work statement 2
    'UPDATE accounts SET balance = balance + $1 WHERE id = $2',
    [amount, toId]
  );
  await client.query('COMMIT');        // ── atomically make ALL work durable (WAL fsync)
} catch (err) {                        // ── ANY failure above lands here
  await client.query('ROLLBACK');      // ── discard ALL work; leave no partial state
  throw err;                           // ── re-throw so the caller knows it failed
} finally {                            // ── runs on success AND on failure
  client.release();                    // ── return the client to the pool — NON-NEGOTIABLE
}
```

Annotated breakdown of each critical piece:

```
await pool.connect()
   │  └── returns a PoolClient bound to one physical connection.
   │      Distinct from pool.query(), which grabs+returns a client per call.
   │
await client.query('BEGIN')
   │  └── sends the literal SQL command BEGIN. Not a driver method — just SQL.
   │      Alternatives: 'BEGIN ISOLATION LEVEL SERIALIZABLE', 'START TRANSACTION'.
   │
client.query(text, params)
   │  └── text uses $1,$2 placeholders; params is an array. NEVER string-concatenate.
   │      Runs on the same backend, inside the open transaction.
   │
await client.query('COMMIT')
   │  └── the durability point. Returns only after WAL flush (default). Success here
   │      means every statement in the block is now permanent.
   │
await client.query('ROLLBACK')
   │  └── discards every uncommitted change. Also the ONLY way to clear an aborted
   │      transaction so the connection is reusable.
   │
client.release()
   │  └── returns the client to the pool. Optionally client.release(true) DESTROYS
   │      the connection instead of reusing it — used when the connection may be
   │      in an unknown state.
   └──
```

`release()` is not `end()`. `client.release()` returns the connection to the pool for reuse. `pool.end()` (or `client.release(true)`) actually closes connections. Confusing these is a frequent bug: calling `client.end()` on a pooled client throws or corrupts the pool.

---

## 5. Why Transaction Pattern Mastery Matters in Production

1. **Connection leaks take down the entire service.** The single most common production incident with `pg`: a code path that checks out a client, throws before `release()`, and never returns it. The pool has a fixed size (default 10). Leak 10 clients and *every* subsequent `pool.connect()` hangs until `connectionTimeoutMillis` fires, then rejects. Your health checks fail, your API returns 503s, and the database itself looks perfectly healthy — because the bottleneck is your Node pool, not Postgres. The `finally { client.release() }` discipline is the fix, and it must be airtight.

2. **Partial writes corrupt business state.** Without a transaction, a two-step "debit A, credit B" can debit A and then crash before crediting B — money vanishes. Or an order is created but its `order_items` insert fails, leaving a headless order. Transactions make these all-or-nothing. Getting the pattern wrong (e.g., running the two statements via `pool.query()` on different connections) silently reintroduces the partial-write bug even though "there's a transaction in the code."

3. **The wrong-connection bug is invisible in testing.** `pool.query('BEGIN')` then `pool.query('INSERT ...')` *appears* to work under light load because the pool often hands back the same idle client. Under concurrency, the statements scatter across connections and the transaction silently does nothing. This class of bug passes tests and fails in production.

4. **Serialization failures and deadlocks are normal, not exceptional.** Under `SERIALIZABLE` isolation or with concurrent updates to overlapping rows, Postgres *will* abort transactions with `40001` (serialization failure) or `40P01` (deadlock). These are not bugs — they are the database protecting correctness and telling you to retry. Applications that don't implement retry logic surface these as random 500s to users. Applications that do retry (with backoff) turn them into invisible, self-healing hiccups.

5. **Long-open transactions degrade the whole database.** A transaction left open (because of a slow external API call inside the transaction, or a leaked client) holds its snapshot, blocks `VACUUM` from reclaiming dead tuples cluster-wide, holds locks other transactions wait on, and bloats tables. One leaked transaction can slowly starve a healthy database. Mastery means keeping transactions short and never doing network I/O (HTTP calls, queue publishes) *inside* the transaction boundary.

6. **The Unit of Work pattern is how real applications stay consistent.** A single API request often mutates several tables (create order, decrement inventory, write audit log). Doing each in its own transaction means a crash between them leaves inconsistency. A Unit of Work collects all those mutations into one transaction so the request is atomic end-to-end. Getting this abstraction right — passing the same `client` through every repository call — is a core backend design skill.

---

## 6. Deep Technical Content

### 6.1 Pool vs Client vs PoolClient — the three objects

`node-postgres` gives you three distinct things, and transactions only work with the right one:

```javascript
import pg from 'pg';
const { Pool, Client } = pg;

const pool = new Pool({ /* ... */ });    // manages N connections, hands them out
const client = new Client({ /* ... */ });// ONE standalone connection you manage by hand
const poolClient = await pool.connect(); // a client CHECKED OUT from the pool
```

- **`pool.query(text, params)`** — a convenience method. It internally does `connect()`, runs one query, and `release()`s automatically. **Great for single autocommit statements. Useless for multi-statement transactions** because each call may use a different connection.
- **`pool.connect()`** returns a **PoolClient** — a real, dedicated connection you hold until you `release()` it. **This is the only correct object for a transaction.**
- **`new Client()`** — a single connection with no pooling. Valid for scripts, migrations, and `LISTEN/NOTIFY` long-lived connections, but you manage `connect()`/`end()` yourself. In a web server you almost always want the pool.

The mental rule: **one transaction = one checked-out PoolClient, held for the transaction's entire life.**

### 6.2 Why `pool.query('BEGIN')` is a silent bug

```javascript
// ❌ BROKEN — do not do this
await pool.query('BEGIN');
await pool.query('UPDATE accounts SET balance = balance - 100 WHERE id = 1');
await pool.query('UPDATE accounts SET balance = balance + 100 WHERE id = 2');
await pool.query('COMMIT');
```

Each `pool.query()` independently checks out a client, runs, and releases. There is no guarantee the four calls hit the same backend. In the worst case:

- `BEGIN` runs on connection A, then A is released **still in a transaction** (pg will actually roll it back on release, but the point stands).
- The two `UPDATE`s run on connections B and C **in autocommit mode** — they commit immediately and independently. There is no atomicity: a crash between them leaves a partial transfer.
- `COMMIT` runs on connection D and commits *nothing* meaningful.

`pg` mitigates the worst case by rolling back any open transaction when a client is released, but the atomicity you wanted is gone. Always hold one PoolClient.

### 6.3 The `finally` release discipline in depth

The release must be unconditional. Consider the failure modes:

```javascript
const client = await pool.connect();
try {
  await client.query('BEGIN');
  await doWork(client);
  await client.query('COMMIT');
} catch (err) {
  await client.query('ROLLBACK');   // ← what if THIS throws?
  throw err;
} finally {
  client.release();                 // ← still runs. Good.
}
```

Edge cases the `finally` must survive:

- **`ROLLBACK` itself throws** (e.g., the connection died). The `finally` still runs `release()`. But note: releasing a broken connection back into the pool is dangerous — prefer `client.release(err)` / `client.release(true)` to destroy it (see 6.4).
- **`COMMIT` throws.** This is subtle: if `COMMIT` throws, did it commit? Usually the connection failed *before* the server confirmed — so the transaction likely did NOT commit, but you cannot be 100% certain without checking. The `catch` will run `ROLLBACK`, which on an already-dead connection is a no-op, and `finally` releases.
- **`release()` called twice.** Calling `client.release()` a second time throws `Release called on client which has already been released to the pool`. Guard against double-release by only releasing in `finally` and never elsewhere.

### 6.4 Releasing a poisoned connection

If a connection errored in a way that leaves it in an unknown state, don't hand it back to the pool for the next request to inherit. `pg` lets you signal this:

```javascript
} catch (err) {
  try { await client.query('ROLLBACK'); } catch { /* connection may be dead */ }
  throw err;
} finally {
  // Passing a truthy value removes the client from the pool and closes it.
  client.release(err instanceof Error ? err : undefined);
}
```

`client.release(true)` or `client.release(someError)` **destroys** the physical connection instead of returning it to the pool. The pool will lazily open a fresh one when needed. Use this when the connection may be corrupted (protocol error, terminated backend). For ordinary application errors where the transaction rolled back cleanly, a plain `client.release()` is correct and cheaper.

### 6.5 The reusable `withTransaction` helper

Repeating the connect/BEGIN/try/catch/finally boilerplate everywhere is error-prone — one forgotten `release()` leaks. Factor it into a single helper that is impossible to misuse:

```javascript
/**
 * Runs `work` inside a transaction. Passes the checked-out client to the
 * callback. Commits if the callback resolves, rolls back if it throws.
 * ALWAYS releases the client.
 */
async function withTransaction(pool, work) {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');
    const result = await work(client);   // caller does its queries via this client
    await client.query('COMMIT');
    return result;
  } catch (err) {
    try {
      await client.query('ROLLBACK');
    } catch (rollbackErr) {
      // Log but prefer to surface the ORIGINAL error.
      rollbackErr.cause = err;
    }
    throw err;
  } finally {
    client.release();
  }
}

// Usage — the callback receives the transactional client:
const orderId = await withTransaction(pool, async (client) => {
  const { rows } = await client.query(
    'INSERT INTO orders (customer_id, status) VALUES ($1, $2) RETURNING id',
    [customerId, 'pending']
  );
  const orderId = rows[0].id;
  for (const item of items) {
    await client.query(
      'INSERT INTO order_items (order_id, product_id, quantity) VALUES ($1, $2, $3)',
      [orderId, item.productId, item.quantity]
    );
  }
  return orderId;
});
```

This helper is the workhorse. Everything else in this topic — isolation levels, retries, the Unit of Work — builds on it. Key properties: the caller **cannot** forget `release()`, **cannot** forget `ROLLBACK`, and the transaction boundary is exactly the callback's lifetime.

### 6.6 Configuring isolation level per transaction

Isolation level is a property of a transaction, set right after `BEGIN`:

```javascript
async function withSerializableTransaction(pool, work) {
  const client = await pool.connect();
  try {
    await client.query('BEGIN ISOLATION LEVEL SERIALIZABLE');
    const result = await work(client);
    await client.query('COMMIT');
    return result;
  } catch (err) {
    try { await client.query('ROLLBACK'); } catch {}
    throw err;
  } finally {
    client.release();
  }
}
```

Valid levels (Topic 52): `READ COMMITTED` (Postgres default), `REPEATABLE READ`, `SERIALIZABLE`. `READ UNCOMMITTED` is accepted but behaves as `READ COMMITTED` (Postgres never shows uncommitted data). You can also set read-only or deferrable:

```sql
BEGIN ISOLATION LEVEL SERIALIZABLE READ ONLY DEFERRABLE;
```

`DEFERRABLE` (only meaningful for `SERIALIZABLE READ ONLY`) lets a read-only transaction wait for a "safe" snapshot so it never has to be retried with `40001` — ideal for long reporting queries.

### 6.7 Savepoints — nested/partial rollback

A savepoint is a named checkpoint inside a transaction. Rolling back to it undoes work *after* the savepoint while keeping the transaction alive and keeping everything before it.

```javascript
await client.query('BEGIN');
await client.query('INSERT INTO orders (id, status) VALUES ($1, $2)', [oid, 'pending']);

await client.query('SAVEPOINT before_items');
try {
  await client.query('INSERT INTO order_items (order_id, product_id) VALUES ($1, $2)', [oid, pid]);
} catch (err) {
  // Undo ONLY the failed item insert; the order row survives.
  await client.query('ROLLBACK TO SAVEPOINT before_items');
  // optionally: RELEASE SAVEPOINT before_items; then take a different path
}
await client.query('COMMIT');
```

Critical use case: recovering from a caught error inside a transaction. Recall from §3 that after *any* error the whole transaction is aborted and all statements fail with `25P02`. `ROLLBACK TO SAVEPOINT` is the **only** way to catch an error mid-transaction and continue — it clears the aborted state back to the savepoint. This is exactly how ORMs implement "nested transactions": there is no true nesting in Postgres; nested transactions are savepoints under the hood.

```sql
SAVEPOINT sp_name;             -- create checkpoint
ROLLBACK TO SAVEPOINT sp_name; -- undo to checkpoint, transaction still alive
RELEASE SAVEPOINT sp_name;     -- discard the checkpoint (keeps its work)
```

### 6.8 The Unit of Work pattern

A **Unit of Work** (UoW) tracks all the database changes for one business operation and commits them as a single transaction. In practice, it means: **one transaction owns one `client`, and every repository/DAO involved in the operation uses that same `client`.** The anti-pattern is repositories that each grab their own `pool.query()` — then there is no shared transaction and no atomicity.

```javascript
// Repositories take a "db executor" (either the pool for autocommit,
// or a transactional client). They do NOT know about transactions.
class OrderRepository {
  constructor(db) { this.db = db; }              // db = pool OR client
  async create(customerId) {
    const { rows } = await this.db.query(
      'INSERT INTO orders (customer_id, status) VALUES ($1, $2) RETURNING id',
      [customerId, 'pending']
    );
    return rows[0].id;
  }
  async addItem(orderId, productId, qty) {
    await this.db.query(
      'INSERT INTO order_items (order_id, product_id, quantity) VALUES ($1,$2,$3)',
      [orderId, productId, qty]
    );
  }
}

class InventoryRepository {
  constructor(db) { this.db = db; }
  async decrement(productId, qty) {
    const { rowCount } = await this.db.query(
      'UPDATE products SET stock = stock - $1 WHERE id = $2 AND stock >= $1',
      [qty, productId]
    );
    if (rowCount === 0) throw new Error(`Insufficient stock for product ${productId}`);
  }
}

// The service composes repositories inside ONE transaction (the Unit of Work).
async function placeOrder(pool, customerId, items) {
  return withTransaction(pool, async (client) => {
    // All repos share the SAME transactional client — this is the Unit of Work.
    const orders = new OrderRepository(client);
    const inventory = new InventoryRepository(client);

    const orderId = await orders.create(customerId);
    for (const item of items) {
      await inventory.decrement(item.productId, item.quantity); // throws → whole thing rolls back
      await orders.addItem(orderId, item.productId, item.quantity);
    }
    await client.query(
      'INSERT INTO audit_logs (entity, entity_id, action) VALUES ($1,$2,$3)',
      ['order', orderId, 'created']
    );
    return orderId;
  });
}
```

The elegance: because every repository received the same `client`, the entire `placeOrder` — order creation, N inventory decrements, N item inserts, and the audit log — commits atomically or rolls back entirely. If `inventory.decrement` throws on item 3, the order row, the first two decrements, and the first two items all vanish. The database is never left with a half-placed order.

The key design constraint: **repositories must accept an injected executor (`pool` or `client`), never reach for a global `pool` themselves.** That injection is what lets the same repository code run both in autocommit (pass the pool) and inside a Unit of Work (pass the transactional client).

### 6.9 Retrying serialization failures (40001) and deadlocks (40P01)

Under `REPEATABLE READ` and `SERIALIZABLE`, or under concurrent conflicting updates, Postgres aborts a transaction and returns a SQLSTATE the application is **expected** to retry:

- **`40001` — `serialization_failure`**: the transaction could not be serialized against a concurrent one. The correct response is to roll back and re-run the *entire* transaction from the top (a new snapshot may succeed).
- **`40P01` — `deadlock_detected`**: two transactions waited on each other's locks; Postgres killed one. Retry the victim.

Both are **transient** and safe to retry *only if the whole transaction is re-executed from BEGIN* — you cannot retry a single statement, because the snapshot and all prior work are gone. Retry with exponential backoff and jitter to avoid a thundering herd re-colliding:

```javascript
const RETRYABLE = new Set(['40001', '40P01']);

async function withRetryableTransaction(pool, work, { maxRetries = 5 } = {}) {
  for (let attempt = 0; ; attempt++) {
    const client = await pool.connect();
    try {
      await client.query('BEGIN ISOLATION LEVEL SERIALIZABLE');
      const result = await work(client);
      await client.query('COMMIT');
      return result;                                   // success → return
    } catch (err) {
      try { await client.query('ROLLBACK'); } catch {}
      // Only retry the specific transient conflict codes, and only up to the cap.
      if (RETRYABLE.has(err.code) && attempt < maxRetries) {
        const backoff = Math.min(1000, 2 ** attempt * 25);   // 25,50,100,200,400,800ms cap 1s
        const jitter = Math.random() * backoff;              // full jitter
        await new Promise((r) => setTimeout(r, backoff / 2 + jitter / 2));
        continue;                                            // re-run the WHOLE transaction
      }
      throw err;                                             // non-retryable or out of retries
    } finally {
      client.release();                                      // released every attempt
    }
  }
}
```

Design points that matter:

- **Retry the whole transaction, not a statement.** A fresh `client`, a fresh `BEGIN`, a fresh snapshot. The loop checks out and releases a client *per attempt*.
- **Only retry `40001`/`40P01`.** Retrying a unique-violation (`23505`) or check-violation loops forever — those are deterministic and will fail again.
- **Cap retries.** Without a ceiling, a persistently contended row causes unbounded retries.
- **Backoff with jitter.** If ten transactions collide, retrying them all after exactly 25ms just re-collides. Jitter spreads them out. "Full jitter" (random between 0 and the cap) is the standard AWS-recommended approach.
- **The callback `work` must be idempotent-on-retry.** Because `work` can run multiple times, it must not have side effects outside the transaction (no sending emails, no HTTP calls) — those would fire once per attempt. Keep external effects *after* the successful commit.

### 6.10 Never do external I/O inside a transaction

A transaction holds locks and pins a snapshot. Doing slow, unpredictable work inside it — an HTTP call to a payment gateway, publishing to Kafka, sending an email — extends the transaction's lifetime to the mercy of that external system. A 30-second gateway timeout becomes a 30-second-open transaction blocking `VACUUM` and holding row locks.

```javascript
// ❌ WRONG — external call inside the transaction
await withTransaction(pool, async (client) => {
  await client.query('UPDATE orders SET status = $1 WHERE id = $2', ['paid', orderId]);
  await paymentGateway.charge(card, amount);   // ← 5s network call with the transaction OPEN
});

// ✅ RIGHT — charge first (or use an outbox), keep the transaction tiny
const charge = await paymentGateway.charge(card, amount);   // outside any transaction
await withTransaction(pool, async (client) => {
  await client.query(
    'UPDATE orders SET status = $1, charge_id = $2 WHERE id = $3',
    ['paid', charge.id, orderId]
  );
});
```

For "do X in the DB and then reliably tell an external system," use the **transactional outbox pattern**: write the intent to an `outbox` table inside the transaction, and a separate worker reads the outbox and does the external call after commit. This keeps transactions short while preserving reliability.

### 6.11 Statement and idle-in-transaction timeouts

Defense-in-depth against leaked or runaway transactions is set at the server/session level:

```javascript
const pool = new Pool({
  // Kill any single statement that runs longer than 5s.
  statement_timeout: 5000,
  // Kill a transaction that sits idle (open but not executing) longer than 10s —
  // this is the safety net for a leaked client stuck IN TRANSACTION.
  idle_in_transaction_session_timeout: 10000,
});
```

`idle_in_transaction_session_timeout` is the server-side backstop for the exact leak this topic warns about: if your code opens a transaction and then hangs (or leaks the client), Postgres will terminate the backend after the timeout, rolling back and freeing the locks. It is a safety net, **not** a substitute for the `finally { release() }` discipline.

### 6.12 `RETURNING`, `rowCount`, and conditional logic inside transactions

Inside a transaction you often need to branch on what a write did. Two tools:

- **`RETURNING`** turns a write into a read, atomically: `INSERT ... RETURNING id` gives you the new id without a second round trip and without a race.
- **`rowCount`** tells you how many rows a statement affected — essential for optimistic checks like the stock decrement in §6.8 (`WHERE stock >= $1` + `rowCount === 0` means "insufficient stock"), which does the check-and-act atomically instead of a racy `SELECT` then `UPDATE`.

```javascript
const { rowCount } = await client.query(
  'UPDATE accounts SET balance = balance - $1 WHERE id = $2 AND balance >= $1',
  [amount, fromId]
);
if (rowCount === 0) throw new Error('Insufficient funds');   // triggers ROLLBACK in the helper
```

This pattern — mutate with a guard in the `WHERE`, then check `rowCount` — is how you avoid the classic read-then-write race without needing `SELECT ... FOR UPDATE`, all inside the transaction.

---

## 7. EXPLAIN — Transaction Behavior in the Plan

`EXPLAIN` analyzes a *single statement*, not a transaction block — there is no "EXPLAIN BEGIN." But the transactional context profoundly changes what a statement's plan does at runtime, and you can observe transaction-level effects through Postgres's system views and through `EXPLAIN ANALYZE` on the locking statements inside a transaction.

### 7.1 `SELECT ... FOR UPDATE` inside a transaction

The workhorse locking read. Its plan shows a `LockRows` node:

```sql
BEGIN;
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, balance FROM accounts WHERE id = 42 FOR UPDATE;
```

```
LockRows  (cost=0.29..8.31 rows=1 width=22)
          (actual time=0.045..0.047 rows=1 loops=1)
  ->  Index Scan using accounts_pkey on accounts
        (cost=0.29..8.30 rows=1 width=22)
        (actual time=0.028..0.029 rows=1 loops=1)
        Index Cond: (id = 42)
  Buffers: shared hit=4
Planning Time: 0.11 ms
Execution Time: 0.071 ms
COMMIT;
```

**Reading it:**
- `LockRows` — the node that acquires the row-level `FOR UPDATE` lock on each returned tuple. This lock is held **until the transaction ends** (COMMIT/ROLLBACK). Any concurrent transaction that tries to `UPDATE`, `DELETE`, or `FOR UPDATE` the same row **blocks** here until you commit.
- `Index Scan using accounts_pkey` — locking by primary key is fast and locks exactly one row. Locking via a `Seq Scan` (no index on the filter) would lock every row it scans — a common accidental over-locking bug.
- `Buffers: shared hit=4` — the index + heap pages touched.

The plan looks like an ordinary index scan plus a lock node. The *cost* the plan does not show is the **contention**: how long other transactions wait on this lock. That you observe in `pg_locks`, not `EXPLAIN`.

### 7.2 Observing lock waits (the real transaction cost)

To see what `EXPLAIN` cannot — one transaction blocked on another's lock:

```sql
SELECT
  blocked.pid          AS blocked_pid,
  blocked.query        AS blocked_query,
  blocking.pid         AS blocking_pid,
  blocking.query       AS blocking_query,
  blocking.state       AS blocking_state    -- often 'idle in transaction' → a leak!
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocking
  ON blocking.pid = ANY(pg_blocking_pids(blocked.pid))
WHERE blocked.wait_event_type = 'Lock';
```

If `blocking_state` is `idle in transaction`, you have found a leaked or stalled transaction (§6.11) holding a lock while doing nothing — precisely the Node-side leak this topic is about.

### 7.3 A serialization failure has no plan — it's a runtime abort

Under `SERIALIZABLE`, the conflict is detected at `COMMIT` (or mid-transaction), not at plan time. There is nothing in `EXPLAIN` to see; the transaction simply raises:

```
ERROR:  could not serialize access due to read/write dependencies among transactions
DETAIL:  Reason code: Canceled on identification as a pivot, during commit attempt.
HINT:  The transaction might succeed if retried.
SQLSTATE: 40001
```

The `HINT` literally tells you to retry — which is what §6.9's `withRetryableTransaction` does. The predicate locks (`SIReadLock`) that drove this decision are visible in `pg_locks` with `mode = 'SIReadLock'` while the transaction runs, but they carry no cost in the per-statement plan.

### 7.4 What to look for

| Symptom | Where you see it | Meaning / Fix |
|---------|------------------|---------------|
| `LockRows` over `Seq Scan` | `EXPLAIN` | `FOR UPDATE` locking every scanned row — add an index on the filter column |
| `wait_event_type = 'Lock'` | `pg_stat_activity` | A transaction is blocked waiting on a lock held by another |
| `state = 'idle in transaction'` | `pg_stat_activity` | Leaked/stalled transaction holding locks — check Node `finally` release |
| `mode = 'SIReadLock'` | `pg_locks` | Serializable predicate locks; expect possible `40001` |
| Repeated `40001`/`40P01` in logs | app logs / `pg_stat_database` | Contention hotspot; ensure retry logic + consider lock-ordering |

---

## 8. Query Examples

### Example 1 — Basic: Atomic two-row transfer

```javascript
// Move `amount` from one account to another, atomically.
// Either both balances change or neither does.
async function transfer(pool, fromId, toId, amount) {
  const client = await pool.connect();          // one dedicated connection
  try {
    await client.query('BEGIN');                // open transaction
    await client.query(
      'UPDATE accounts SET balance = balance - $1 WHERE id = $2',
      [amount, fromId]                           // debit
    );
    await client.query(
      'UPDATE accounts SET balance = balance + $1 WHERE id = $2',
      [amount, toId]                             // credit
    );
    await client.query('COMMIT');               // make both durable together
  } catch (err) {
    await client.query('ROLLBACK');             // undo any partial change
    throw err;
  } finally {
    client.release();                           // ALWAYS return the client
  }
}
```

### Example 2 — Intermediate: Guarded transfer with `rowCount` and the helper

```javascript
// Uses the withTransaction helper (§6.5) and an atomic balance guard.
// The WHERE ... AND balance >= $1 makes the check-and-debit a single atomic step.
async function safeTransfer(pool, fromId, toId, amount) {
  return withTransaction(pool, async (client) => {
    const debit = await client.query(
      `UPDATE accounts
         SET balance = balance - $1
       WHERE id = $2
         AND balance >= $1`,                    // guard: only if funds suffice
      [amount, fromId]
    );
    if (debit.rowCount === 0) {
      // No row updated → insufficient funds (or account missing).
      // Throwing here triggers ROLLBACK inside the helper.
      throw new Error('INSUFFICIENT_FUNDS');
    }
    await client.query(
      'UPDATE accounts SET balance = balance + $1 WHERE id = $2',
      [amount, toId]
    );
    await client.query(
      'INSERT INTO audit_logs (entity, entity_id, action, detail) VALUES ($1,$2,$3,$4)',
      ['account', fromId, 'transfer', JSON.stringify({ toId, amount })]
    );
  });
}
```

### Example 3 — Production-grade: Order placement with retry, isolation, and Unit of Work

**Context:** `orders` (~5M rows), `order_items` (~40M rows), `products` (~200K rows) with `products.stock` a hot, contended column during flash sales. Indexes: `products_pkey` on `products(id)`, `orders_pkey`, `order_items_order_id_idx`. Under concurrency, overlapping stock decrements produce `40001`/deadlocks. **Perf expectation:** the transaction itself touches a handful of rows by primary key and commits in low single-digit milliseconds; retries add bounded latency only under contention.

```javascript
const RETRYABLE = new Set(['40001', '40P01']);

async function withRetryableTransaction(pool, work, { maxRetries = 5, isolation = 'READ COMMITTED' } = {}) {
  for (let attempt = 0; ; attempt++) {
    const client = await pool.connect();
    try {
      await client.query(`BEGIN ISOLATION LEVEL ${isolation}`);
      const result = await work(client);
      await client.query('COMMIT');
      return result;
    } catch (err) {
      try { await client.query('ROLLBACK'); } catch {}
      if (RETRYABLE.has(err.code) && attempt < maxRetries) {
        const cap = Math.min(1000, 2 ** attempt * 25);
        await new Promise((r) => setTimeout(r, cap / 2 + Math.random() * (cap / 2)));
        continue;
      }
      throw err;
    } finally {
      client.release();
    }
  }
}

async function placeOrder(pool, customerId, items) {
  // NOTE: side-effect-free work only — safe to run multiple times on retry.
  return withRetryableTransaction(pool, async (client) => {
    const { rows } = await client.query(
      'INSERT INTO orders (customer_id, status, created_at) VALUES ($1,$2,NOW()) RETURNING id',
      [customerId, 'pending']
    );
    const orderId = rows[0].id;

    let total = 0;
    for (const { productId, quantity } of items) {
      // Atomic decrement-with-guard; returns the price so we can total it.
      const dec = await client.query(
        `UPDATE products
            SET stock = stock - $1
          WHERE id = $2 AND stock >= $1
          RETURNING price`,
        [quantity, productId]
      );
      if (dec.rowCount === 0) throw new Error(`OUT_OF_STOCK:${productId}`);
      const price = dec.rows[0].price;
      total += price * quantity;

      await client.query(
        'INSERT INTO order_items (order_id, product_id, quantity, unit_price) VALUES ($1,$2,$3,$4)',
        [orderId, productId, quantity, price]
      );
    }

    await client.query('UPDATE orders SET status = $1, total_amount = $2 WHERE id = $3',
      ['confirmed', total, orderId]);
    return { orderId, total };
  }, { isolation: 'READ COMMITTED', maxRetries: 5 });
}
```

**EXPLAIN of the hot statement** (the stock decrement, the source of contention):

```sql
EXPLAIN (ANALYZE, BUFFERS)
UPDATE products SET stock = stock - 2 WHERE id = 8123 AND stock >= 2 RETURNING price;
```

```
Update on products  (cost=0.42..8.44 rows=1 width=6)
                    (actual time=0.089..0.091 rows=1 loops=1)
  ->  Index Scan using products_pkey on products
        (cost=0.42..8.44 rows=1 width=6)
        (actual time=0.041..0.043 rows=1 loops=1)
        Index Cond: (id = 8123)
        Filter: (stock >= 2)
  Buffers: shared hit=5
Planning Time: 0.14 ms
Execution Time: 0.140 ms
```

Reading it: primary-key `Index Scan` locates exactly one row; `Filter: (stock >= 2)` enforces the guard; the `Update` node writes the new tuple and takes the row lock held until commit. Sub-millisecond per statement — the latency budget under load is dominated by lock waits and retries, not by the plan, which is why the retry/backoff wrapper (not query tuning) is the right lever here.

---

## 9. Wrong → Right Patterns

### Wrong 1: Multi-statement transaction via `pool.query()`

```javascript
// ❌ WRONG — statements scatter across different pooled connections
await pool.query('BEGIN');
await pool.query('UPDATE accounts SET balance = balance - 100 WHERE id = 1');
await pool.query('UPDATE accounts SET balance = balance + 100 WHERE id = 2');
await pool.query('COMMIT');
```

**What actually happens:** each `pool.query()` checks out an arbitrary client, runs, and releases it. `BEGIN` opens a transaction on connection A which is then returned to the pool (pg auto-rolls-back the open transaction on release). The two `UPDATE`s land on connections B and C **in autocommit mode** — they commit independently. There is **no atomicity**: a crash between them permanently debits account 1 without crediting account 2. Under light test load the pool often hands back the same idle client so it *appears* to work — this is why the bug survives to production.

```javascript
// ✅ RIGHT — one checked-out client for the whole transaction
const client = await pool.connect();
try {
  await client.query('BEGIN');
  await client.query('UPDATE accounts SET balance = balance - 100 WHERE id = 1');
  await client.query('UPDATE accounts SET balance = balance + 100 WHERE id = 2');
  await client.query('COMMIT');
} catch (err) {
  await client.query('ROLLBACK');
  throw err;
} finally {
  client.release();
}
```

### Wrong 2: Releasing before commit / leaking on error

```javascript
// ❌ WRONG — release() inside try, no finally; error path leaks the client
const client = await pool.connect();
await client.query('BEGIN');
await client.query('UPDATE accounts SET balance = balance - 100 WHERE id = 1');
await doRiskyThing();                 // ← throws here...
await client.query('COMMIT');
client.release();                     // ← never reached; client is LEAKED, forever IN TRANSACTION
```

**What actually happens:** when `doRiskyThing()` throws, execution jumps out of the function. `COMMIT` and `release()` are skipped. The client is never returned — it sits in the pool's "checked out" set holding an open transaction (and any locks) until `idle_in_transaction_session_timeout` (if configured) reaps it. Repeat this on a hot code path and the pool exhausts: every new request hangs at `pool.connect()`, the service returns 503s, and Postgres looks idle.

```javascript
// ✅ RIGHT — release in finally, rollback in catch
const client = await pool.connect();
try {
  await client.query('BEGIN');
  await client.query('UPDATE accounts SET balance = balance - 100 WHERE id = 1');
  await doRiskyThing();
  await client.query('COMMIT');
} catch (err) {
  await client.query('ROLLBACK');
  throw err;
} finally {
  client.release();                   // runs on success AND failure
}
```

### Wrong 3: Continuing after an error without a savepoint

```javascript
// ❌ WRONG — swallow the error and keep querying the same transaction
await client.query('BEGIN');
try {
  await client.query('INSERT INTO users (email) VALUES ($1)', [email]); // unique violation 23505
} catch (e) {
  // "handled" — but the transaction is now ABORTED
}
await client.query('INSERT INTO audit_logs (action) VALUES ($1)', ['signup']);
// ↑ ERROR: current transaction is aborted, commands ignored until end of block (25P02)
await client.query('COMMIT');         // also fails; effectively a ROLLBACK
```

**What actually happens:** after the unique-violation, the whole transaction enters the aborted state. Every subsequent statement fails with `25P02` until a `ROLLBACK`. The audit insert never runs and the `COMMIT` is rejected. Catching the error in JS does **not** clear the aborted state in Postgres.

```javascript
// ✅ RIGHT — wrap the fallible statement in a savepoint
await client.query('BEGIN');
await client.query('SAVEPOINT sp_user');
try {
  await client.query('INSERT INTO users (email) VALUES ($1)', [email]);
} catch (e) {
  if (e.code !== '23505') throw e;             // only tolerate the duplicate
  await client.query('ROLLBACK TO SAVEPOINT sp_user');  // clears aborted state
  // ... take an alternative path (e.g., fetch the existing user)
}
await client.query('INSERT INTO audit_logs (action) VALUES ($1)', ['signup']);  // now works
await client.query('COMMIT');
```

### Wrong 4: No retry on serialization/deadlock failures

```javascript
// ❌ WRONG — SERIALIZABLE with no retry; 40001 surfaces as a random 500
async function increment(pool, counterId) {
  return withSerializableTransaction(pool, async (client) => {
    const { rows } = await client.query('SELECT val FROM counters WHERE id=$1', [counterId]);
    await client.query('UPDATE counters SET val=$1 WHERE id=$2', [rows[0].val + 1, counterId]);
  });
  // Under concurrency this THROWS 40001 to the caller ~randomly. Users see errors.
}
```

**What actually happens:** two concurrent transactions read the same counter snapshot; at commit Postgres detects the read/write dependency cycle and aborts one with `40001`. Without retry, that abort propagates as an unhandled error and the increment is simply lost from the user's perspective (they got a 500).

```javascript
// ✅ RIGHT — retry the whole transaction with backoff
async function increment(pool, counterId) {
  return withRetryableTransaction(pool, async (client) => {
    const { rows } = await client.query('SELECT val FROM counters WHERE id=$1', [counterId]);
    await client.query('UPDATE counters SET val=$1 WHERE id=$2', [rows[0].val + 1, counterId]);
  }, { isolation: 'SERIALIZABLE', maxRetries: 5 });
}
// Even simpler when possible: avoid the read-modify-write entirely:
//   UPDATE counters SET val = val + 1 WHERE id = $1;   -- atomic, no 40001 needed
```

### Wrong 5: External I/O inside the transaction

```javascript
// ❌ WRONG — holds the transaction open across a slow network call
await withTransaction(pool, async (client) => {
  await client.query('UPDATE orders SET status=$1 WHERE id=$2', ['paid', orderId]);
  await sendReceiptEmail(customerEmail);   // ← 2–30s; transaction + row lock held the whole time
});
```

**What actually happens:** the row lock on the order and the transaction's snapshot are held for the entire duration of the email call. Under load these pile up, `VACUUM` stalls, and lock waits cascade. If the email service hangs, the transaction stays open indefinitely.

```javascript
// ✅ RIGHT — commit fast, do external effects after (or via an outbox)
await withTransaction(pool, async (client) => {
  await client.query('UPDATE orders SET status=$1 WHERE id=$2', ['paid', orderId]);
  await client.query(
    'INSERT INTO outbox (topic, payload) VALUES ($1,$2)',
    ['order.paid', JSON.stringify({ orderId, customerEmail })]
  );
});
// A separate worker reads `outbox` and calls sendReceiptEmail() AFTER commit — reliably, once.
```

---

## 10. Performance Profile

### Cost anatomy of a transaction

| Component | Cost | Notes |
|-----------|------|-------|
| `pool.connect()` (warm pool) | microseconds | Reuses an idle connection; no TCP/TLS/auth handshake |
| `pool.connect()` (cold / pool empty) | 1–20ms+ | New TCP + TLS + Postgres auth + backend fork |
| `BEGIN` | ~microseconds | Cheap; snapshot often deferred to first real statement |
| Each statement | plan + execute | Same as autocommit; transaction adds no per-statement overhead |
| Row locks (`FOR UPDATE`, `UPDATE`) | in-tuple + lock table | Cheap to take; expensive if others *wait* on them |
| `COMMIT` | 1 WAL fsync | The durability cost; `synchronous_commit=off` trades durability for latency |
| `ROLLBACK` | ~free | Abandoned tuples cleaned by VACUUM later |

The dominant, non-obvious costs are **connection acquisition** (keep the pool warm and sized right) and **lock contention** (short transactions, consistent lock ordering).

### Scaling behavior

- **1M rows, low contention:** transactions touching a handful of rows by PK commit in low single-digit ms. Throughput is bounded by `COMMIT` fsync rate and pool size, not row count. Batching many independent writes into one transaction amortizes the fsync — 1 transaction of 1,000 inserts is dramatically faster than 1,000 single-insert transactions (one WAL flush vs a thousand).
- **10M rows, moderate contention:** hot rows (a shared counter, popular product stock) become the bottleneck. `SELECT ... FOR UPDATE` serializes access to those rows; throughput on a single hot row is capped at `1 / (transaction hold time)`. Keep the critical section minimal.
- **100M rows, high contention / SERIALIZABLE:** `40001` and `40P01` rates climb with concurrency. Retry storms can amplify load — jittered backoff is essential. At this scale, redesign to reduce contention (atomic `col = col + 1` instead of read-modify-write; sharded counters; advisory locks from Topic 56 for coarse serialization; queue-based single-writer for the hottest rows) rather than relying on retries.

### Pool sizing

```javascript
const pool = new Pool({
  max: 20,                              // upper bound on concurrent connections
  min: 2,                               // keep a few warm to avoid cold-connect latency
  idleTimeoutMillis: 30000,             // close idle connections after 30s
  connectionTimeoutMillis: 5000,        // fail fast if no client available in 5s
  statement_timeout: 5000,
  idle_in_transaction_session_timeout: 10000,
});
```

**Sizing rule:** `max` is bounded by Postgres `max_connections` divided across all app instances (plus headroom for admin/replication). More is not better — each connection is a backend process with memory (`work_mem` per sort/hash). A common formula is `connections ≈ ((core_count * 2) + effective_spindle_count)` *per database node*, split across app instances. If you need thousands of app-side connections, put **PgBouncer** in front (transaction-pooling mode) rather than raising `max` — but note PgBouncer transaction mode is incompatible with session-level features (prepared statements across calls, `SET`, advisory locks held across statements).

### Optimization techniques specific to transactions

- **Keep transactions short.** Every millisecond a transaction is open is a millisecond of held locks and a pinned snapshot. Do computation and external I/O outside the boundary.
- **Order locks consistently.** Deadlocks (`40P01`) arise from transactions acquiring the same rows in different orders. Always lock rows in a deterministic order (e.g., `ORDER BY id`) to prevent cycles.
- **Prefer atomic single statements** over read-modify-write when possible: `UPDATE ... SET n = n + 1` needs no lock-and-reread and never throws `40001`.
- **Batch writes** to amortize `COMMIT` fsync, but not so large that the transaction holds locks too long — there is a sweet spot (often 500–5,000 rows).
- **Use `synchronous_commit = off`** (per-session) for high-throughput, loss-tolerant writes (e.g., analytics events) — commits return before the WAL fsync, trading a few hundred ms of durability for large throughput gains. Never for money.

---

## 11. Node.js Integration

`node-postgres` (`pg`) is the foundation. Everything above is `pg`; this section consolidates the integration details and result handling.

### 11.1 The complete, production-shaped transaction module

```javascript
// db.js
import pg from 'pg';
const { Pool } = pg;

export const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  max: 20,
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 5000,
  statement_timeout: 5000,
  idle_in_transaction_session_timeout: 10000,
});

// Surface pool-level errors on IDLE clients (e.g., server killed the backend).
pool.on('error', (err) => {
  console.error('Unexpected idle client error', err);
});

const RETRYABLE = new Set(['40001', '40P01']);

export async function withTransaction(work, { isolation = 'READ COMMITTED', maxRetries = 5 } = {}) {
  for (let attempt = 0; ; attempt++) {
    const client = await pool.connect();
    try {
      await client.query(`BEGIN ISOLATION LEVEL ${isolation}`);
      const result = await work(client);
      await client.query('COMMIT');
      return result;
    } catch (err) {
      try { await client.query('ROLLBACK'); } catch { /* connection may be dead */ }
      if (RETRYABLE.has(err.code) && attempt < maxRetries) {
        const cap = Math.min(1000, 2 ** attempt * 25);
        await new Promise((r) => setTimeout(r, cap / 2 + Math.random() * (cap / 2)));
        continue;
      }
      throw err;
    } finally {
      client.release();
    }
  }
}
```

### 11.2 Using it — `$1` parameter binding and result handling

```javascript
import { pool, withTransaction } from './db.js';

async function createUserWithProfile(email, displayName) {
  return withTransaction(async (client) => {
    // RETURNING turns the INSERT into a read of the generated id — no extra round trip.
    const { rows, rowCount } = await client.query(
      'INSERT INTO users (email) VALUES ($1) RETURNING id, created_at',
      [email]                                    // params array; $1 is safely bound (no SQLi)
    );
    const userId = rows[0].id;                   // result.rows is an array of row objects
    await client.query(
      'INSERT INTO user_profiles (user_id, display_name) VALUES ($1, $2)',
      [userId, displayName]
    );
    return { userId, createdAt: rows[0].created_at, rowCount };
  });
}
```

Key `pg` result-handling facts:

- **`result.rows`** — array of plain objects keyed by column/alias name. Empty array (`[]`) when no rows, never `null`.
- **`result.rowCount`** — number of rows affected (for `INSERT/UPDATE/DELETE`) or returned (for `SELECT`). The primary signal for conditional logic (`rowCount === 0` → guard failed).
- **`$1, $2, …`** — positional placeholders. `pg` sends parameters separately from SQL (the extended query protocol), so they are never string-interpolated — this is your SQL-injection defense. **Never** build SQL with template literals from user input.
- **Type parsing:** `pg` maps Postgres types to JS: `int4`→`number`, `int8`(bigint)→**`string`** (to avoid precision loss beyond 2^53), `numeric`→`string`, `timestamptz`→`Date`, `bool`→`boolean`, `json/jsonb`→parsed object, `uuid`→`string`. The bigint-as-string behavior surprises people — configure `pg.types.setTypeParser` if you need numbers.

### 11.3 Error codes you branch on

`pg` errors carry a `.code` (the SQLSTATE) — branch on it, never on message strings:

```javascript
try {
  await createUserWithProfile(email, name);
} catch (err) {
  switch (err.code) {
    case '23505': return { error: 'EMAIL_TAKEN' };        // unique_violation
    case '23503': return { error: 'FK_MISSING' };         // foreign_key_violation
    case '23514': return { error: 'CHECK_FAILED' };       // check_violation
    case '40001': // serialization_failure — should have been retried already
    case '40P01': // deadlock_detected
      return { error: 'CONFLICT_RETRY_EXHAUSTED' };
    case '25P02': return { error: 'TXN_ABORTED' };         // in_failed_sql_transaction
    default: throw err;
  }
}
```

### 11.4 Do ORMs handle transactions?

Yes — every major ORM wraps this connect/BEGIN/COMMIT/ROLLBACK/release lifecycle. Most also implement retry helpers or expose isolation levels, and they implement "nested transactions" as savepoints exactly as described in §6.7. But **all of them ultimately do what this section shows**: hold one connection, drive the transaction on it, release in a finally. When you need per-transaction control (custom retry policy, savepoint choreography, advisory locks alongside), the raw `pg` client is always reachable underneath. The next section compares them.

---

## 12. ORM Comparison

This is the deep section for ORMs — transactions are where ORMs differ most, because they must translate the "one client, held for the whole block" requirement into their own abstractions.

### Prisma

**Can Prisma do transactions?** — Yes, two forms: the **sequential array** API and the **interactive** API.

```typescript
import { PrismaClient, Prisma } from '@prisma/client';
const prisma = new PrismaClient();

// (a) Sequential array — all-or-nothing, but you can't branch between statements.
await prisma.$transaction([
  prisma.account.update({ where: { id: 1 }, data: { balance: { decrement: 100 } } }),
  prisma.account.update({ where: { id: 2 }, data: { balance: { increment: 100 } } }),
]);

// (b) Interactive transaction — a callback receives a transactional `tx` client.
// This is the Prisma equivalent of withTransaction: every tx.* call uses ONE connection.
await prisma.$transaction(async (tx) => {
  const debit = await tx.account.updateMany({
    where: { id: 1, balance: { gte: 100 } },
    data: { balance: { decrement: 100 } },
  });
  if (debit.count === 0) throw new Error('INSUFFICIENT_FUNDS');  // throw → auto ROLLBACK
  await tx.account.update({ where: { id: 2 }, data: { balance: { increment: 100 } } });
}, {
  isolationLevel: Prisma.TransactionIsolationLevel.Serializable,
  timeout: 5000,       // max time the interactive txn may run before auto-rollback
  maxWait: 2000,       // max time to wait for a connection from the pool
});
```

**Retry:** Prisma does **not** auto-retry `40001`/`40P01`. You wrap `$transaction` in your own retry loop, branching on `err.code === 'P2034'` (Prisma's "transaction failed due to a write conflict or deadlock, retry") or the underlying SQLSTATE.

**Where it breaks:** the interactive transaction's callback must not do slow external work (Prisma enforces `timeout`). The `tx` object is a scoped Prisma client — you cannot use the global `prisma` inside (that would use a different connection and escape the transaction). Raw SQL inside: `tx.$queryRaw` / `tx.$executeRaw`.

**Verdict:** Interactive transactions are the right tool and closely mirror `withTransaction`. Add your own retry wrapper for serialization failures.

### Drizzle ORM

**Can Drizzle do transactions?** — Yes, via `db.transaction(async (tx) => ...)`, and it supports nested transactions (savepoints).

```typescript
import { db } from './db';
import { accounts, auditLogs } from './schema';
import { eq, sql, and, gte } from 'drizzle-orm';

await db.transaction(async (tx) => {
  const debited = await tx
    .update(accounts)
    .set({ balance: sql`${accounts.balance} - 100` })
    .where(and(eq(accounts.id, 1), gte(accounts.balance, 100)))
    .returning({ id: accounts.id });
  if (debited.length === 0) throw new Error('INSUFFICIENT_FUNDS');  // throw → ROLLBACK

  await tx.update(accounts).set({ balance: sql`${accounts.balance} + 100` }).where(eq(accounts.id, 2));

  // Nested transaction = SAVEPOINT under the hood.
  await tx.transaction(async (tx2) => {
    await tx2.insert(auditLogs).values({ entity: 'account', action: 'transfer' });
  });
});

// Isolation level (pg driver):
await db.transaction(async (tx) => { /* ... */ }, {
  isolationLevel: 'serializable',
  accessMode: 'read write',
  deferrable: false,
});
```

**Retry:** not built in — wrap `db.transaction` in your own loop on `err.code`.

**Where it breaks:** nothing major; Drizzle is a thin, honest layer over `pg`. The `tx` object must be used for all in-transaction queries (using the outer `db` escapes the transaction). Non-trivial `sql` expressions for atomic updates are idiomatic.

**Verdict:** Excellent and transparent. Closest to hand-written `pg`. Add your own retry wrapper.

### Sequelize

**Can Sequelize do transactions?** — Yes, **managed** (auto commit/rollback) and **unmanaged** (manual).

```javascript
const { sequelize, Account, AuditLog } = require('./models');
const { Transaction } = require('sequelize');

// Managed — commits if the callback resolves, rolls back if it throws. Preferred.
await sequelize.transaction(
  { isolationLevel: Transaction.ISOLATION_LEVELS.SERIALIZABLE },
  async (t) => {
    // EVERY query must be passed { transaction: t } or it runs OUTSIDE the transaction!
    const [count] = await Account.update(
      { balance: sequelize.literal('balance - 100') },
      { where: { id: 1, balance: { [Op.gte]: 100 } }, transaction: t }
    );
    if (count === 0) throw new Error('INSUFFICIENT_FUNDS');
    await Account.update(
      { balance: sequelize.literal('balance + 100') },
      { where: { id: 2 }, transaction: t }
    );
    await AuditLog.create({ entity: 'account', action: 'transfer' }, { transaction: t });
  }
);
```

**Retry:** Sequelize has built-in retry via the `retry` option on the connection/transaction for matching error patterns, but it is limited; most teams write their own loop.

**Where it breaks:** the classic Sequelize bug — **forgetting `{ transaction: t }` on a query silently runs it outside the transaction** on a different connection, breaking atomicity with no error. (CLS/`AsyncLocalStorage` "namespaces" can auto-inject `t`, but must be explicitly configured.) This is the single most common Sequelize transaction mistake.

**Verdict:** Managed transactions are solid, but the mandatory-and-forgettable `{ transaction: t }` threading is a footgun. Enable CLS namespaces to mitigate.

### TypeORM

**Can TypeORM do transactions?** — Yes: `dataSource.transaction()`, `QueryRunner` (manual), or the `@Transaction` decorator (deprecated).

```typescript
import { DataSource } from 'typeorm';

// (a) Callback form — an EntityManager scoped to the transaction is provided.
await dataSource.transaction('SERIALIZABLE', async (manager) => {
  const res = await manager.decrement(Account, { id: 1, balance: MoreThanOrEqual(100) }, 'balance', 100);
  if (res.affected === 0) throw new Error('INSUFFICIENT_FUNDS');
  await manager.increment(Account, { id: 2 }, 'balance', 100);
  await manager.insert(AuditLog, { entity: 'account', action: 'transfer' });
});

// (b) QueryRunner — full manual control, mirrors raw pg most closely.
const qr = dataSource.createQueryRunner();
await qr.connect();
await qr.startTransaction('SERIALIZABLE');   // BEGIN
try {
  await qr.manager.update(Account, 1, { balance: () => 'balance - 100' });
  await qr.manager.update(Account, 2, { balance: () => 'balance + 100' });
  await qr.commitTransaction();              // COMMIT
} catch (err) {
  await qr.rollbackTransaction();            // ROLLBACK
  throw err;
} finally {
  await qr.release();                        // release the QueryRunner's connection — required!
}
```

**Retry:** none built in; wrap the callback form in your own loop.

**Where it breaks:** the `QueryRunner` form is powerful but you must `release()` it in a `finally` — the exact same leak risk as raw `pg`. In the callback form, using a repository from the global `dataSource` instead of the provided `manager` escapes the transaction.

**Verdict:** Callback form for most cases (safer, no manual release); `QueryRunner` when you need savepoints/advisory locks. Same release discipline as raw `pg` applies to `QueryRunner`.

### Knex.js

**Can Knex do transactions?** — Yes, callback and promise forms; supports savepoints via nested `trx.transaction`.

```javascript
const knex = require('knex')({ client: 'pg', connection: process.env.DATABASE_URL });

// Callback form — auto commit on resolve, auto rollback on throw.
await knex.transaction(async (trx) => {
  const count = await trx('accounts')
    .where({ id: 1 })
    .andWhere('balance', '>=', 100)
    .update({ balance: trx.raw('balance - 100') });
  if (count === 0) throw new Error('INSUFFICIENT_FUNDS');

  await trx('accounts').where({ id: 2 }).update({ balance: trx.raw('balance + 100') });
  await trx('audit_logs').insert({ entity: 'account', action: 'transfer' });
}, { isolationLevel: 'serializable' });

// You can also promote an existing query to a transaction with .transacting(trx):
await knex('audit_logs').insert({ action: 'x' }).transacting(trx);
```

**Retry:** not built in — wrap in your own loop on `err.code`.

**Where it breaks:** every query must go through `trx` (or `.transacting(trx)`); a plain `knex(...)` call inside the callback runs outside the transaction — same footgun family as Sequelize. Knex's `trx` is a real checked-out connection, and the callback form releases it for you.

**Verdict:** The most SQL-transparent option; `trx` behaves like a `pg` client. Add your own retry wrapper.

### ORM Summary Table

| ORM | Transaction API | Auto commit/rollback | Isolation level | Nested = savepoint | Built-in retry | Main footgun |
|-----|-----------------|----------------------|-----------------|--------------------|----------------|--------------|
| Prisma | `$transaction(cb)` (interactive) | Yes | `isolationLevel` opt | Yes | No (`P2034` yourself) | Callback timeout; must use `tx` |
| Drizzle | `db.transaction(cb)` | Yes | `isolationLevel` opt | Yes | No | Must use `tx`, not `db` |
| Sequelize | `sequelize.transaction(cb)` | Yes (managed) | `isolationLevel` opt | Yes | Limited | Forgetting `{ transaction: t }` |
| TypeORM | `dataSource.transaction(cb)` / `QueryRunner` | Yes / manual | 1st arg / `startTransaction` | Yes | No | `QueryRunner.release()` leak |
| Knex | `knex.transaction(cb)` | Yes | `isolationLevel` opt | Yes | No | Must use `trx` |
| Raw `pg` | manual `BEGIN`/`COMMIT` | No (you write it) | `BEGIN ISOLATION LEVEL` | `SAVEPOINT` | No | `finally` release |

**Cross-cutting truth:** none of the mainstream ORMs auto-retry serialization failures/deadlocks — you always own the retry loop. And all of them share the same core rule as raw `pg`: **every query in the transaction must go through the transaction-scoped object** (`tx`/`t`/`trx`/`manager`), or it silently escapes onto another connection.

---

## 13. Practice Exercises

### Exercise 1 — Easy

Using the `withTransaction` helper from §6.5 and tables `accounts(id, balance)` and `audit_logs(id, entity, entity_id, action, detail, created_at)`, write a Node.js function `deposit(pool, accountId, amount)` that:
- Increases the account's balance by `amount` in a transaction.
- Writes an audit log row recording the deposit.
- Returns the new balance (use `RETURNING`).

```javascript
async function deposit(pool, accountId, amount) {
  // Write your implementation here
}
```

### Exercise 2 — Medium (combines Topic 56 advisory locks + transactions)

You are generating human-friendly sequential invoice numbers per customer (`invoice_counters(customer_id, next_number)`), and you must never hand out a duplicate under concurrency. Using a transaction, write `nextInvoiceNumber(pool, customerId)` that:
- Acquires a **transaction-level advisory lock** keyed on `customerId` (recall Topic 56: `pg_advisory_xact_lock`) so concurrent callers for the same customer serialize.
- Reads and increments `next_number` atomically, returning the number to use.
- Releases the advisory lock automatically at commit (which flavor of advisory lock does that for you?).

```sql
-- invoice_counters(customer_id BIGINT PRIMARY KEY, next_number BIGINT NOT NULL)
```
```javascript
async function nextInvoiceNumber(pool, customerId) {
  // Write your query/implementation here
}
```

### Exercise 3 — Hard (production simulation; the naive answer is wrong)

A flash sale drops stock on a single hot product. Two hundred concurrent requests call `purchase(pool, productId, userId, qty)`. The naive implementation reads stock, checks it in JS, then updates — and oversells (sells more than exists) under concurrency. Write a **correct** implementation that:
- Never oversells, even at 200 concurrent calls.
- Uses `SERIALIZABLE` (or an atomic guarded `UPDATE`) and retries `40001`/`40P01` with jittered backoff.
- Inserts an `order_items` row and decrements `products.stock` atomically.
- Returns `{ ok: true, remaining }` on success or `{ ok: false, reason: 'OUT_OF_STOCK' }` when sold out.
- Explain in a comment why the naive `SELECT` then `UPDATE` oversells and how your version prevents it.

```javascript
async function purchase(pool, productId, userId, qty) {
  // Write your implementation here
}
```

### Exercise 4 — Interview Simulation

Whiteboard, then implement. "Design a `transferFunds(pool, fromId, toId, amount)` that is safe under high concurrency across a large `accounts` table. Requirements: atomic debit+credit, reject overdrafts, be deadlock-free even when two users transfer to each other simultaneously (A→B and B→A at the same time), retry transient conflicts, and never leak a connection. Walk through your isolation-level choice, your lock-ordering strategy, and your retry policy."

```javascript
async function transferFunds(pool, fromId, toId, amount) {
  // Write your query here
  // Hint: to be deadlock-free, lock the two rows in a DETERMINISTIC order
  //       (e.g., always the smaller id first), regardless of transfer direction.
}
```

---

## 14. Interview Questions

### Junior Level

**Q: Why do you need a `withTransaction` helper instead of just calling `BEGIN` and `COMMIT` yourself?**

A: The manual approach leaks connections. If any query between `BEGIN` and `COMMIT` throws, the `COMMIT` line never runs, the `ROLLBACK` never runs, and — worse — `client.release()` never runs. That connection is now checked out of the pool forever, holding an open transaction that keeps locks and pins the oldest transaction ID. A `withTransaction(pool, fn)` helper wraps the work in `try/catch/finally`: it acquires a client, runs `BEGIN`, calls your callback, `COMMIT`s on success, `ROLLBACK`s in the `catch`, and — critically — `client.release()`s in the `finally` so the connection always goes back to the pool no matter what.

```javascript
async function withTransaction(pool, fn) {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');
    const result = await fn(client);
    await client.query('COMMIT');
    return result;
  } catch (err) {
    await client.query('ROLLBACK');
    throw err;
  } finally {
    client.release(); // always runs — no leaked connection
  }
}
```

**Follow-up:** *What is the single most common bug people write inside the callback?* — Using `pool.query(...)` instead of `client.query(...)`. `pool.query` grabs a *different* connection from the pool, so that statement runs outside your transaction — it commits independently and is invisible to the `ROLLBACK`. Every statement in the unit of work must go through the same `client`.

---

**Q: When you insert an order and its `order_items` in one request, why should both be in a single transaction?**

A: They form one atomic business fact: "an order exists with its line items." If the `orders` INSERT succeeds but the process crashes before the `order_items` INSERTs, you get a phantom order with zero items — a broken invariant that downstream code (totals, fulfilment) will trip over. Wrapping both in a transaction means either the order and all its items are visible together, or nothing is.

```javascript
await withTransaction(pool, async (client) => {
  const { rows } = await client.query(
    'INSERT INTO orders (user_id, status) VALUES ($1, $2) RETURNING id',
    [userId, 'pending']
  );
  const orderId = rows[0].id;
  for (const item of items) {
    await client.query(
      'INSERT INTO order_items (order_id, product_id, quantity, unit_price) VALUES ($1,$2,$3,$4)',
      [orderId, item.productId, item.qty, item.price]
    );
  }
});
```

**Follow-up:** *How would you make this faster than a per-item loop?* — Batch the items into a single multi-row `INSERT ... VALUES ($1,$2,...),($5,$6,...)` or use `unnest($1::int[], $2::int[], ...)`. One round trip instead of N, still inside the same transaction.

---

### Principal Level

**Q: Walk me through the Unit of Work pattern and why it matters at scale for a Node service.**

A: A Unit of Work treats a set of reads and writes as one bounded, consistent chunk of work with a single lifecycle: it opens once, all repository operations enlist on the *same* connection/transaction, and it commits or rolls back once at the boundary (typically the request or job handler). The value is threefold. (1) **Consistency** — every write in the request sees the same snapshot and commits atomically, so you never persist half a change. (2) **Connection discipline** — one connection is checked out for the request, not one-per-repository-call, which prevents pool exhaustion under load. (3) **Testability and composition** — a service method receives a `client`/UoW and never knows whether it's the top-level transaction or nested inside a larger one, so business logic composes without each method opening its own transaction.

The implementation trap is *ambient* vs *explicit* context. The clean version threads the `client` explicitly through repository function signatures (`createOrder(client, ...)`). The ergonomic-but-dangerous version stores the client in `AsyncLocalStorage` so repositories fetch it implicitly — convenient, but if any code path escapes the async context (a stray `setImmediate`, a detached promise) it silently falls back to `pool.query` and writes outside the transaction. At scale I keep the boundary explicit and audit that no repository ever imports the pool directly.

**Follow-up:** *How do you handle nested `withTransaction` calls when a service method that opens its own transaction is called from inside another one?* — Postgres doesn't support true nested transactions, so an inner `BEGIN` inside an open transaction is a no-op warning. You make the helper savepoint-aware: if it detects it's already inside a transaction (via the passed-in client or a depth counter in the context), it issues `SAVEPOINT sp_n` on entry, `RELEASE SAVEPOINT` on success, and `ROLLBACK TO SAVEPOINT` on failure — giving partial rollback semantics without breaking the outer transaction.

---

**Q: Under `SERIALIZABLE` isolation, Postgres will abort transactions with `40001`. How do you build a retry layer, and what are the correctness constraints on the callback?**

A: `SERIALIZABLE` uses optimistic concurrency (SSI). When two concurrent transactions would produce a result inconsistent with any serial order, Postgres aborts one with SQLSTATE `40001` (`serialization_failure`); deadlocks surface as `40P01` (`deadlock_detected`). These are *transient* — the correct response is to roll back and retry the *entire* transaction from scratch, because the aborted transaction saw a snapshot that's now invalid. My retry wrapper loops N times (say 5), and on catching `40001`/`40P01` sleeps with exponential backoff plus jitter (to avoid retry stampedes where the same pair keeps colliding in lockstep) and re-runs the callback with a fresh `BEGIN`. Any other error is rethrown immediately — retrying a unique-violation or a check-constraint failure just wastes work and can mask bugs.

The hard constraint: **the callback must be idempotent and side-effect-free outside the database.** Because the whole thing may run 2–5 times, the callback must not send an email, charge a card, or push to a queue mid-flight — those go *after* the transaction commits, or through a transactional outbox (`audit_logs`/an outbox table written inside the transaction, dispatched after commit). It also must not mutate objects passed in from the caller in a way that isn't safe to repeat (e.g., don't pop items off an array you're iterating).

```javascript
function isRetryable(err) {
  return err.code === '40001' || err.code === '40P01';
}

async function withRetryableTransaction(pool, fn, { retries = 5 } = {}) {
  for (let attempt = 0; ; attempt++) {
    try {
      return await withTransaction(pool, async (client) => {
        await client.query("SET TRANSACTION ISOLATION LEVEL SERIALIZABLE");
        return fn(client);
      });
    } catch (err) {
      if (isRetryable(err) && attempt < retries) {
        const backoff = Math.min(2 ** attempt * 20, 500);
        const jitter = Math.random() * backoff;
        await new Promise((r) => setTimeout(r, backoff + jitter));
        continue; // fresh attempt — new snapshot
      }
      throw err; // non-retryable, or out of attempts
    }
  }
}
```

**Follow-up:** *How do you keep a runaway retry loop from amplifying an outage?* — Cap total attempts, cap total elapsed time (a deadline, not just a count), and emit a metric on every retry so a rising `40001` rate is visible. Combine with a circuit breaker: if serialization failures spike, shed load rather than hammering the database with retries that will also fail.

---

**Q: A payment flow decrements `products.stock`, inserts a `payments` row, and updates `orders.status`. It occasionally oversells and occasionally double-charges. Diagnose the likely causes and the fix.**

A: Two distinct bugs. **Overselling** comes from a read-then-write race: `SELECT stock` in JS, check `>= qty`, then `UPDATE stock = stock - qty`. Between the read and the write, another transaction does the same — both see stock=1, both decrement, stock goes negative. Fix: make the decrement an atomic guarded write — `UPDATE products SET stock = stock - $1 WHERE id = $2 AND stock >= $1` and check `rowCount`; if zero, it's `OUT_OF_STOCK`. This pushes the check-and-set into the database under a row lock. Alternatively run the whole flow under `SERIALIZABLE` with retries.

**Double-charging** is an idempotency failure, usually from client retries or the retry layer itself re-running a non-idempotent side effect. Fix: an idempotency key. The client (or the order) supplies a unique key; the `payments` insert carries a `UNIQUE` constraint on that key. On retry the second insert hits `23505` (unique violation) and you treat it as "already processed" rather than charging again. Crucially, the external charge call must sit behind the same idempotency guarantee — never inside the retryable transaction body, because a `40001` retry would fire the charge twice.

**Follow-up:** *Why not just wrap everything in `SERIALIZABLE` and call it done?* — `SERIALIZABLE` protects the *database* state from anomalies, but it does nothing about *external* side effects. A retried transaction that charges a card in its body will charge twice even though the DB stays consistent. Isolation solves the in-database race; idempotency keys and a post-commit dispatch (transactional outbox) solve the external double-effect. You need both.

---

## 15. Mental Model Checkpoint

Answer these from memory. If any one is fuzzy, re-read the section it maps to before moving on.

1. **In `withTransaction`, which line *must* live in the `finally` block and why does putting it in the `try` (or after `COMMIT`) silently break under load?** What symptom does the mistake produce in the connection pool?

2. **Your callback runs `pool.query('INSERT INTO order_items ...')` instead of `client.query(...)`. The outer transaction rolls back. What happens to that inserted row, and why?**

3. **A `40001` (`serialization_failure`) bubbles up. Explain why retrying a *single statement* is wrong and you must replay the *entire* transaction callback.** What property must the callback have for that replay to be safe?

4. **Between exponential backoff *with* jitter and *without* jitter for two transactions that keep colliding — which one can livelock in lockstep, and why does jitter break the cycle?**

5. **A service method that opens its own `withTransaction` is called from inside another `withTransaction` on the same request. What does the inner `BEGIN` actually do in Postgres, and what mechanism gives you real partial-rollback semantics instead?**

6. **Your payment flow is wrapped in `SERIALIZABLE` with retries, yet customers are charged twice. Isolation is working correctly — where is the double-charge actually coming from, and what fixes it?**

7. **Distinguish `40001` from `40P01`. Are both retryable? Is `23505` (unique violation) retryable? Why does the distinction between "transient" and "deterministic" errors decide your retry predicate?**

---

## 16. Quick Reference Card

```javascript
// ── withTransaction: the non-negotiable skeleton ─────────────────
async function withTransaction(pool, fn) {
  const client = await pool.connect();      // 1 connection for the whole unit
  try {
    await client.query('BEGIN');
    const result = await fn(client);         // pass client, NEVER use pool inside
    await client.query('COMMIT');
    return result;
  } catch (err) {
    await client.query('ROLLBACK');
    throw err;
  } finally {
    client.release();                        // ALWAYS — even on throw
  }
}

// ── Retry wrapper for SERIALIZABLE / deadlocks ───────────────────
const RETRYABLE = new Set(['40001', '40P01']); // serialization_failure, deadlock_detected
async function withRetryableTransaction(pool, fn, { retries = 5 } = {}) {
  for (let attempt = 0; ; attempt++) {
    try {
      return await withTransaction(pool, async (client) => {
        await client.query('SET TRANSACTION ISOLATION LEVEL SERIALIZABLE');
        return fn(client);                   // callback MUST be idempotent
      });
    } catch (err) {
      if (!RETRYABLE.has(err.code) || attempt >= retries) throw err;
      const base = Math.min(2 ** attempt * 20, 500);
      await new Promise(r => setTimeout(r, base + Math.random() * base)); // jitter!
    }
  }
}
```

| SQLSTATE | Name | Retry? | Meaning |
|----------|------|--------|---------|
| `40001`  | serialization_failure | Yes | SSI aborted this txn; replay whole callback |
| `40P01`  | deadlock_detected     | Yes | Deadlock; Postgres killed this txn as victim |
| `23505`  | unique_violation      | No  | Deterministic; retry re-fails. Use for idempotency keys |
| `23503`  | foreign_key_violation | No  | Deterministic constraint failure |
| `23514`  | check_violation       | No  | Deterministic constraint failure |

**Atomic guarded write (prevents oversell without SELECT-then-UPDATE):**
```sql
UPDATE products SET stock = stock - $1
 WHERE id = $2 AND stock >= $1;   -- check rowCount === 0 → OUT_OF_STOCK
```

**Deadlock avoidance:** lock rows in a deterministic order (e.g. smaller `id` first) in every transaction, regardless of business direction.

**Rules of thumb:**
- One `client` per unit of work; repositories receive it, never import the pool.
- Nested `withTransaction` → make it savepoint-aware (`SAVEPOINT` / `ROLLBACK TO SAVEPOINT`).
- No external side effects (email, charge, queue) inside a retryable body — dispatch after COMMIT or via a transactional outbox (`audit_logs`/outbox table).
- Retry predicate = transient errors only; cap attempts *and* wall-clock deadline; emit a metric per retry.

---

## Connected Topics

- **Phase 8 — ACID and Isolation Levels**: The theory `withTransaction` operationalizes; `READ COMMITTED` vs `REPEATABLE READ` vs `SERIALIZABLE` decides which anomalies you must code around and whether `40001` can even occur.
- **Phase 8 — SELECT FOR UPDATE and Pessimistic Locking**: The alternative to optimistic SSI retries — take row locks up front (`FOR UPDATE`), the basis for the deterministic lock-ordering that keeps `transferFunds` deadlock-free.
- **Phase 8 — Deadlocks: Detection and Prevention**: Where `40P01` comes from, how Postgres picks a victim, and why consistent lock ordering matters — the sibling of the serialization retry story.
- **Connection Pooling (pg / node-postgres)**: Why `client.release()` in `finally` is load-bearing; pool exhaustion is the failure mode of a leaked transactional connection.
- **Idempotency Keys and the Transactional Outbox**: How to make retries and client re-sends safe for `payments` and other external side effects that isolation alone cannot protect.
- **ORM Transactions (Sequelize `sequelize.transaction`, Prisma `$transaction`)**: How these frameworks implement the same Unit of Work — managed vs unmanaged transactions, and their built-in retry/`isolationLevel` options.
- **Optimistic Concurrency with Version Columns**: A lighter-weight alternative to `SERIALIZABLE` — a `version` column plus `WHERE version = $expected` for last-writer-wins conflict detection without full-transaction retries.
