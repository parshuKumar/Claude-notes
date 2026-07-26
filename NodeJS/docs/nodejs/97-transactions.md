# 97 — Transactions

## What is this?

A transaction is a group of database operations that are treated as **one single unit** — either all of them succeed and get saved permanently, or if any one fails, all of them are undone as if none had ever happened. Think of a bank transfer: money leaves account A and arrives in account B. If the server crashes after the withdrawal but before the deposit, the money must not vanish — a transaction guarantees both steps happen together, or neither does. `BEGIN` starts the unit, `COMMIT` saves it permanently, and `ROLLBACK` undoes it completely.

## Why does it matter for backend development?

Almost every backend does multi-step database work: create an order AND decrement stock, create a user AND create their profile row, transfer money AND log the transfer. Without transactions, a crash or thrown error halfway through leaves your database in a half-done, corrupted state — stock decremented but no order row, money withdrawn but never deposited. Backend developers wrap every multi-step write in a transaction so the database is always left in a consistent state, even when something goes wrong mid-way. This is one of the most important reliability patterns in server-side code.

---

## Syntax / API

```js
// Using node-postgres (pg) — the standard PostgreSQL driver, covered in Topic 90
const { Pool } = require('pg');

// A connection pool — reused across the app, covered in Topic 93
const dbConnection = new Pool({ connectionString: process.env.DATABASE_URL });

async function transferFunds(fromAccountId, toAccountId, amount) {
  // A transaction must run on ONE dedicated client, not the shared pool —
  // otherwise each query could land on a different connection
  const client = await dbConnection.connect();

  try {
    // Start the transaction — nothing is saved yet, just staged
    await client.query('BEGIN');

    // Step 1 — withdraw from the source account
    await client.query(
      'UPDATE accounts SET balance = balance - $1 WHERE id = $2',
      [amount, fromAccountId]
    );

    // Step 2 — deposit into the destination account
    await client.query(
      'UPDATE accounts SET balance = balance + $1 WHERE id = $2',
      [amount, toAccountId]
    );

    // Both steps succeeded — make the changes permanent
    await client.query('COMMIT');
  } catch (error) {
    // ANY step failed — undo everything staged since BEGIN
    await client.query('ROLLBACK');
    throw error; // re-throw so the caller (route handler) knows it failed
  } finally {
    // Always release the client back to the pool, success or failure
    client.release();
  }
}

module.exports = { transferFunds };
```

---

## How it works — line by line

- `dbConnection.connect()` checks out **one dedicated connection** from the pool. Transactions must stay on a single connection because `BEGIN`, your queries, and `COMMIT`/`ROLLBACK` all need to run against the exact same database session — the pool cannot be trusted to hand every query to the same connection automatically.
- `client.query('BEGIN')` tells PostgreSQL: "everything from this point on is staged, not saved — wait for my signal."
- Each subsequent query runs normally and returns results as usual, but nothing is visible to other connections yet — it is invisible outside this transaction until committed.
- If every query in the `try` block succeeds, `COMMIT` tells PostgreSQL to make all the staged changes permanent and visible to everyone at once.
- If any query throws (bad SQL, a constraint violation, a network blip), execution jumps straight to the `catch` block, which sends `ROLLBACK` — this discards every change made since `BEGIN`, leaving the database exactly as it was before the transaction started.
- `finally` always runs `client.release()` — this returns the connection to the pool so other requests can reuse it. Forgetting this leaks connections and eventually exhausts the pool (Topic 93).

### The ACID guarantees a transaction gives you

```
A — Atomicity    All queries in the transaction succeed together, or none do.
C — Consistency  The database moves from one valid state to another valid state —
                  constraints (foreign keys, unique, checks) are never violated mid-way.
I — Isolation    Other connections don't see your half-finished changes while
                  your transaction is still in progress.
D — Durability   Once COMMIT succeeds, the data survives a crash — it is written
                  to disk, not just held in memory.
```

---

## Example 1 — basic

```js
// File: src/db/createUserWithProfile.js
// Creating a user AND their profile row must succeed together, or not at all.

const { Pool } = require('pg');
const dbConnection = new Pool({ connectionString: process.env.DATABASE_URL });

async function createUserWithProfile(requestBody) {
  const client = await dbConnection.connect(); // grab a dedicated connection

  try {
    await client.query('BEGIN'); // start the transaction

    // Insert the user row — RETURNING gives back the generated id
    const userResult = await client.query(
      'INSERT INTO users (email, password_hash) VALUES ($1, $2) RETURNING id',
      [requestBody.email, requestBody.passwordHash]
    );
    const userId = userResult.rows[0].id; // the new user's id

    // Insert the profile row, linked by the userId we just got back
    await client.query(
      'INSERT INTO profiles (user_id, display_name) VALUES ($1, $2)',
      [userId, requestBody.displayName]
    );

    await client.query('COMMIT'); // both inserts succeeded — save permanently
    return { userId }; // caller gets back the new user's id
  } catch (error) {
    await client.query('ROLLBACK'); // undo the user insert too, even if it succeeded
    throw error; // let the caller (e.g. an Express route) handle the error response
  } finally {
    client.release(); // always give the connection back to the pool
  }
}

module.exports = { createUserWithProfile };
```

---

## Example 2 — real world backend use case

```js
// File: src/services/orderService.js
// Placing an order: create the order row, decrement stock, and record payment —
// all three must happen together, or the customer is charged with no stock reserved.

const { Pool } = require('pg');
const dbConnection = new Pool({ connectionString: process.env.DATABASE_URL });

// Custom error so route handlers can tell "out of stock" apart from a server bug
class OutOfStockError extends Error {
  constructor(productId) {
    super(`Product ${productId} is out of stock`);
    this.name = 'OutOfStockError';
    this.statusCode = 409; // Conflict — useful for the Express error handler (Topic 62)
  }
}

async function placeOrder(userId, productId, quantity) {
  const client = await dbConnection.connect();

  try {
    await client.query('BEGIN');

    // Lock the product row FOR UPDATE — prevents two concurrent orders from
    // both reading the same stock count and over-selling the last item
    const stockResult = await client.query(
      'SELECT stock_count FROM products WHERE id = $1 FOR UPDATE',
      [productId]
    );
    const stockCount = stockResult.rows[0].stock_count;

    // Business rule check — if it fails, we throw INSIDE the transaction
    if (stockCount < quantity) {
      throw new OutOfStockError(productId); // caught below, triggers ROLLBACK
    }

    // Decrement stock now that we know enough is available
    await client.query(
      'UPDATE products SET stock_count = stock_count - $1 WHERE id = $2',
      [quantity, productId]
    );

    // Create the order row
    const orderResult = await client.query(
      `INSERT INTO orders (user_id, product_id, quantity, status)
       VALUES ($1, $2, $3, 'pending') RETURNING id`,
      [userId, productId, quantity]
    );
    const orderId = orderResult.rows[0].id;

    // Record the payment attempt linked to this order
    await client.query(
      'INSERT INTO payments (order_id, amount_cents, status) VALUES ($1, $2, $3)',
      [orderId, quantity * 1999, 'authorized'] // e.g. $19.99 per item
    );

    await client.query('COMMIT'); // stock, order, and payment all saved together
    return { orderId };
  } catch (error) {
    await client.query('ROLLBACK'); // undo stock decrement + any partial inserts
    throw error; // propagate to the Express route (e.g. res.status(err.statusCode))
  } finally {
    client.release(); // return the connection no matter what happened
  }
}

module.exports = { placeOrder, OutOfStockError };
```

### Nested transactions — SAVEPOINT

PostgreSQL has no true "transaction inside a transaction" — instead you use a **SAVEPOINT**, which lets you roll back part of a transaction without undoing everything before it.

```js
await client.query('BEGIN');                       // outer transaction starts
await client.query('INSERT INTO orders ...');       // step that must always survive

await client.query('SAVEPOINT before_discount');    // mark a rollback point
try {
  await client.query('UPDATE promotions SET uses = uses + 1 WHERE code = $1', [code]);
} catch (error) {
  // only undo the promotion update — the order insert above is untouched
  await client.query('ROLLBACK TO SAVEPOINT before_discount');
}

await client.query('COMMIT'); // the order is saved even if the promo step failed
```

---

## Common mistakes

### Mistake 1 — Running transaction queries on the pool instead of one client

```js
// ❌ WRONG — pool.query() may hand each call a DIFFERENT connection from the pool,
// so BEGIN happens on one connection while the UPDATE runs on another — no transaction
await dbConnection.query('BEGIN');
await dbConnection.query('UPDATE accounts SET balance = balance - 100 WHERE id = $1', [userId]);
await dbConnection.query('COMMIT');

// ✅ CORRECT — check out ONE client and run every step on it
const client = await dbConnection.connect();
await client.query('BEGIN');
await client.query('UPDATE accounts SET balance = balance - 100 WHERE id = $1', [userId]);
await client.query('COMMIT');
client.release();
```

### Mistake 2 — Forgetting to release the client, or releasing too early

```js
// ❌ WRONG — no finally block, so a thrown error skips client.release() forever,
// leaking a connection out of the pool on every failure
const client = await dbConnection.connect();
await client.query('BEGIN');
await client.query('UPDATE accounts SET balance = balance - 100 WHERE id = $1', [userId]);
await client.query('COMMIT');
client.release(); // never reached if the UPDATE above throws

// ✅ CORRECT — finally always runs, whether the try succeeded or the catch fired
const client = await dbConnection.connect();
try {
  await client.query('BEGIN');
  await client.query('UPDATE accounts SET balance = balance - 100 WHERE id = $1', [userId]);
  await client.query('COMMIT');
} catch (error) {
  await client.query('ROLLBACK');
  throw error;
} finally {
  client.release(); // guaranteed to run
}
```

### Mistake 3 — Swallowing the error instead of rolling back and re-throwing

```js
// ❌ WRONG — catches the error, logs it, but never sends ROLLBACK — the transaction
// stays open on that connection, and the caller thinks the order succeeded (returns undefined)
try {
  await client.query('BEGIN');
  await client.query('UPDATE products SET stock_count = stock_count - 1 WHERE id = $1', [productId]);
  await client.query('INSERT INTO orders (product_id) VALUES ($1)', [productId]);
  await client.query('COMMIT');
} catch (error) {
  console.log('Something went wrong:', error.message); // caller never finds out it failed
}

// ✅ CORRECT — always ROLLBACK on failure, and re-throw so the caller can react
try {
  await client.query('BEGIN');
  await client.query('UPDATE products SET stock_count = stock_count - 1 WHERE id = $1', [productId]);
  await client.query('INSERT INTO orders (product_id) VALUES ($1)', [productId]);
  await client.query('COMMIT');
} catch (error) {
  await client.query('ROLLBACK'); // undo the partial stock update
  throw error; // caller (route handler) can send a proper error response
} finally {
  client.release();
}
```

---

## Practice exercises

### Exercise 1 — easy

Write a function `moveTaskToList(taskId, newListId)` for a to-do app backend. It should:
1. Open a transaction on a dedicated `pg` client
2. Update the task's `list_id` column to `newListId`
3. Insert a row into an `activity_log` table recording the move (`task_id`, `new_list_id`, `moved_at`)
4. Commit if both steps succeed
5. Roll back and re-throw if either step fails, and always release the client

```js
// Write your code here
```

---

### Exercise 2 — medium

Write a function `registerUserWithWallet(requestBody)` that, inside a single transaction:
1. Inserts a new row into `users` (email, password_hash) and gets back the new `userId`
2. Inserts a matching row into `wallets` (user_id, balance) with a starting balance of `0`
3. If the email already exists (unique constraint violation), catch the specific error, roll back, and throw a custom `EmailTakenError` with a `statusCode` of `409`
4. For any other unexpected error, roll back and re-throw the original error unchanged
5. Always releases the client in a `finally` block regardless of outcome

Test it by calling it twice with the same email and confirming the second call throws `EmailTakenError`.

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a reusable `withTransaction(dbConnection, work)` helper function that:
1. Accepts a `pg` `Pool` instance and an async `work(client)` callback
2. Checks out a client, runs `BEGIN`, then calls `await work(client)` and captures its return value
3. Commits and returns the callback's return value if `work` succeeds
4. Rolls back and re-throws if `work` throws
5. Always releases the client
6. Supports **nested calls** — if `withTransaction` detects it is already inside an active transaction on the same client (hint: pass an optional existing `client` through, or track depth with a counter), it should use a `SAVEPOINT`/`RELEASE SAVEPOINT`/`ROLLBACK TO SAVEPOINT` instead of a real `BEGIN`/`COMMIT`, so nested calls don't try to commit an outer transaction early

Then rewrite `placeOrder` from Example 2 to use `withTransaction` instead of manual `BEGIN`/`COMMIT`/`ROLLBACK`.

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
CORE COMMANDS
  BEGIN                        → start a transaction, stage all following queries
  COMMIT                       → make all staged changes permanent
  ROLLBACK                     → undo everything since BEGIN
  SAVEPOINT name                → mark a rollback point WITHIN a transaction
  ROLLBACK TO SAVEPOINT name    → undo back to that point only, transaction stays open
  RELEASE SAVEPOINT name        → discard the savepoint, keep its changes

ACID
  Atomicity   → all queries succeed together, or none do
  Consistency → constraints never violated mid-transaction
  Isolation   → other connections don't see half-finished changes
  Durability  → COMMIT survives a crash, written to disk

NODE.JS PATTERN (pg driver)
  const client = await pool.connect();   // ONE dedicated connection
  try {
    await client.query('BEGIN');
    ... queries on client, not pool ...
    await client.query('COMMIT');
  } catch (err) {
    await client.query('ROLLBACK');
    throw err;
  } finally {
    client.release();                    // always, success or failure
  }

RULES
  Never mix pool.query() and client.query() inside one transaction
  Always release() in a finally block
  Always re-throw after ROLLBACK — never swallow the error silently
  Use SELECT ... FOR UPDATE to lock rows and prevent race conditions
  Use SAVEPOINT for partial rollback inside a larger transaction

GOTCHAS
  A crashed process before COMMIT   → transaction is auto-rolled-back by Postgres
  Forgetting client.release()       → pool exhaustion (Topic 93)
  Long-running transactions          → hold locks, block other queries — keep them short
```

---

## Connected topics

- **90 — Connecting to PostgreSQL** — the `pg` driver and connection basics that transactions are built on top of
- **93 — Connection pooling** — why transactions need one dedicated client instead of the shared pool, and how leaking a client exhausts the pool
- **87 — Integration testing with a real DB** — uses transactions to wrap each test and roll it back afterward, keeping the test database clean
