# 93 — Connection pooling

## What is this?

A connection pool is a small collection of already-open database connections that your app keeps ready and hands out to queries as they arrive, instead of opening a brand new connection every time. Think of it like a taxi rank outside a busy office building — instead of calling a new taxi from across town for every single passenger (slow, expensive), a handful of taxis wait right there, pick up a passenger, drop them off, and come straight back to the rank for the next one. In Node.js backends, `pg-pool` (built into the `pg` package) is the standard tool that manages this rank of database connections for you.

---

## Why does it matter for backend development?

Opening a TCP connection to PostgreSQL involves a network round trip, TLS negotiation (if enabled), authentication, and the database spinning up a dedicated backend process — this can take tens of milliseconds. If your API opens and closes a fresh connection for every incoming HTTP request, you pay that cost on every single request and your database quickly runs out of capacity, since PostgreSQL has a hard limit on how many connections it can hold open at once (often 100 by default). A pool solves both problems: connections are reused across requests (fast), and the pool caps how many connections exist at once (safe for the database). Every production Node.js backend that talks to Postgres, MySQL, or similar databases uses pooling — it is not optional at any real traffic volume.

---

## Syntax / API

```js
// pg-pool ships inside the 'pg' package — no separate install needed
const { Pool } = require('pg');

// Create ONE pool for the whole app lifetime — never create a pool per request
const dbPool = new Pool({
  host: process.env.DB_HOST,          // database server address
  port: process.env.DB_PORT || 5432,  // Postgres default port
  user: process.env.DB_USER,          // login user
  password: process.env.DB_PASSWORD,  // login password
  database: process.env.DB_NAME,      // which database to connect to

  max: 20,                  // maximum connections the pool will ever hold open at once
  min: 2,                   // minimum idle connections kept alive (not all pools support this)
  idleTimeoutMillis: 30000, // close an idle connection after 30s of no use
  connectionTimeoutMillis: 5000, // fail fast if no connection is free within 5s
});

// Simple query — pool.query() borrows a connection, runs the query, returns it automatically
async function getUserById(userId) {
  const result = await dbPool.query('SELECT * FROM users WHERE id = $1', [userId]);
  return result.rows[0]; // first (and only) matching row
}

// Manual checkout — needed for transactions (multiple queries on the SAME connection)
async function transferCredits(fromUserId, toUserId, amount) {
  const dbConnection = await dbPool.connect(); // borrow one connection from the pool
  try {
    await dbConnection.query('BEGIN'); // start transaction on THIS connection
    await dbConnection.query('UPDATE accounts SET credits = credits - $1 WHERE user_id = $2', [amount, fromUserId]);
    await dbConnection.query('UPDATE accounts SET credits = credits + $1 WHERE user_id = $2', [amount, toUserId]);
    await dbConnection.query('COMMIT'); // save both changes together
  } catch (err) {
    await dbConnection.query('ROLLBACK'); // undo everything on failure
    throw err;
  } finally {
    dbConnection.release(); // ALWAYS return the connection to the pool — even on error
  }
}

// Graceful shutdown — closes every connection in the pool cleanly
async function closeDbPool() {
  await dbPool.end();
}
```

---

## How it works — line by line

`new Pool({...})` does not connect to the database immediately — it just creates a manager object holding your connection settings and rules. Real connections are opened lazily, the first time something asks the pool for one.

`max: 20` tells the pool the hard ceiling — it will never have more than 20 real connections open to Postgres at the same time, no matter how many requests arrive.

`idleTimeoutMillis` tells the pool to close a connection that has been sitting unused for that long, so you're not paying to keep connections alive that nobody needs — Postgres also has a resource cost per idle connection.

`connectionTimeoutMillis` is the pool's patience limit — if every connection is busy and none frees up within that window, the pool gives up and throws an error rather than making the request wait forever.

`dbPool.query(sql, params)` is the simple path: the pool silently borrows one free connection (or opens a new one if under `max` and none are free), runs your query on it, gets the result back, and immediately returns the connection to the pool — you never see the connection object itself.

`dbPool.connect()` is the manual path: it hands you the actual connection object and does **not** return it automatically. You must call `dbConnection.release()` yourself when you're done — this is required for transactions, because `BEGIN`, your queries, and `COMMIT`/`ROLLBACK` must all run on the exact same physical connection.

The `try / finally` around manual connections is not optional — if you forget `release()`, that connection is permanently checked out and lost to the pool forever, even after an error, which is the single most common pooling bug.

`dbPool.end()` closes every open connection and stops the pool from accepting new checkouts — call this once, during graceful shutdown, never mid-request.

---

## Example 1 — basic

```js
// File: src/db/pool.js
const { Pool } = require('pg');

// One pool, shared across the whole app — created once at module load time
const dbPool = new Pool({
  host: 'localhost',
  port: 5432,
  user: 'app_user',
  password: 'app_password',
  database: 'shop_db',
  max: 10,                        // small pool is fine for a single-instance dev app
  idleTimeoutMillis: 30000,       // free idle connections after 30s
  connectionTimeoutMillis: 5000, // give up waiting after 5s
});

// Log every time the pool opens a brand new physical connection — useful while learning
dbPool.on('connect', () => {
  console.log('[pool] new client connected to Postgres');
});

// Log unexpected errors on idle clients (e.g. network drop) so they don't crash the app
dbPool.on('error', (err) => {
  console.error('[pool] unexpected error on idle client', err);
});

async function listProducts() {
  // pool.query() handles checkout + query + return-to-pool automatically
  const result = await dbPool.query('SELECT id, name, price FROM products ORDER BY name');
  return result.rows; // array of product rows
}

async function main() {
  const products = await listProducts();
  console.log('Products:', products);

  await dbPool.end(); // close all connections cleanly before the script exits
}

main();
```

---

## Example 2 — real world backend use case

```js
// File: src/db/pool.js
// Central pool module — imported everywhere else in the app, created exactly once.
const { Pool } = require('pg');

const dbPool = new Pool({
  host: process.env.DB_HOST,
  port: Number(process.env.DB_PORT) || 5432,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  database: process.env.DB_NAME,
  max: Number(process.env.DB_POOL_MAX) || 20,   // tunable per environment via env var
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 5000,
});

module.exports = { dbPool };


// File: src/services/orderService.js
// Transaction-based checkout flow — must use a single connection for all 3 statements.
const { dbPool } = require('../db/pool');

async function placeOrder(userId, cartItems) {
  const dbConnection = await dbPool.connect(); // check out ONE dedicated connection

  try {
    await dbConnection.query('BEGIN'); // start the transaction

    // 1. Insert the order row and get its generated id back
    const orderResult = await dbConnection.query(
      'INSERT INTO orders (user_id, status) VALUES ($1, $2) RETURNING id',
      [userId, 'pending']
    );
    const orderId = orderResult.rows[0].id;

    // 2. Insert each cart item as an order line, decrementing stock as we go
    for (const item of cartItems) {
      await dbConnection.query(
        'INSERT INTO order_items (order_id, product_id, quantity) VALUES ($1, $2, $3)',
        [orderId, item.productId, item.quantity]
      );

      const stockResult = await dbConnection.query(
        'UPDATE products SET stock = stock - $1 WHERE id = $2 AND stock >= $1 RETURNING id',
        [item.quantity, item.productId]
      );

      // If no row was updated, stock was insufficient — abort the whole order
      if (stockResult.rowCount === 0) {
        throw new Error(`Insufficient stock for product ${item.productId}`);
      }
    }

    await dbConnection.query('COMMIT'); // all inserts + stock updates succeed together
    return orderId;
  } catch (err) {
    await dbConnection.query('ROLLBACK'); // undo the whole order on any failure
    throw err;
  } finally {
    dbConnection.release(); // return the connection to the pool no matter what happened
  }
}

module.exports = { placeOrder };


// File: src/routes/orders.js
// Express route — every request borrows briefly from the SAME shared pool.
const express = require('express');
const router = express.Router();
const { placeOrder } = require('../services/orderService');

router.post('/orders', async (req, res, next) => {
  try {
    const requestBody = req.body; // { userId, cartItems }
    const orderId = await placeOrder(requestBody.userId, requestBody.cartItems);
    res.status(201).json({ orderId });
  } catch (err) {
    next(err); // hand off to centralized error handler (Topic 62)
  }
});

module.exports = router;
```

---

## Common mistakes

### Mistake 1 — Creating a new pool (or connection) per request

```js
// ❌ WRONG — a fresh Pool per request opens fresh connections every time,
// which is exactly what pooling is supposed to prevent
app.get('/users/:userId', async (req, res) => {
  const dbPool = new Pool({ /* ...config... */ }); // new pool EVERY request!
  const result = await dbPool.query('SELECT * FROM users WHERE id = $1', [req.params.userId]);
  res.json(result.rows[0]);
});

// ✅ CORRECT — one pool created at app startup, imported and reused everywhere
const { dbPool } = require('../db/pool'); // module created once, cached by require()

app.get('/users/:userId', async (req, res) => {
  const result = await dbPool.query('SELECT * FROM users WHERE id = $1', [req.params.userId]);
  res.json(result.rows[0]);
});
```

### Mistake 2 — Forgetting to release a manually checked-out connection

```js
// ❌ WRONG — dbConnection is never released, so this connection is leaked
// on EVERY call; after `max` calls, the pool is exhausted and every future
// query hangs until connectionTimeoutMillis fires
async function getUserProfile(userId) {
  const dbConnection = await dbPool.connect();
  const result = await dbConnection.query('SELECT * FROM profiles WHERE user_id = $1', [userId]);
  return result.rows[0]; // returns early — release() never runs!
}

// ✅ CORRECT — try/finally guarantees release() runs even on early return or error
async function getUserProfile(userId) {
  const dbConnection = await dbPool.connect();
  try {
    const result = await dbConnection.query('SELECT * FROM profiles WHERE user_id = $1', [userId]);
    return result.rows[0];
  } finally {
    dbConnection.release(); // always runs, whatever happens above
  }
}
```

### Mistake 3 — Setting `max` far higher than the database can actually handle

```js
// ❌ WRONG — 5 app instances × max: 100 = up to 500 simultaneous connections,
// but Postgres's default max_connections is only 100 — the DB rejects the excess
const dbPool = new Pool({
  // ...config...
  max: 100, // way too high per instance when you run multiple app instances
});

// ✅ CORRECT — size the pool around (Postgres max_connections / number of app instances),
// leaving headroom for admin tools and other services
const dbPool = new Pool({
  // ...config...
  max: 15, // e.g. 100 total / 5 app instances = 20, minus headroom = 15
  connectionTimeoutMillis: 5000, // fail fast instead of piling up requests when the pool is full
});
```

---

## Practice exercises

### Exercise 1 — easy

Create a `pg` pool connected to any local Postgres database (or use a mock/in-memory stand-in if Postgres isn't available). Configure it with `max: 5`, `idleTimeoutMillis: 10000`, and `connectionTimeoutMillis: 3000`. Write a function `getServerTime()` that runs `SELECT NOW()` through `pool.query()` and logs the result. Call it, then call `pool.end()` to shut the pool down cleanly.

```js
// Write your code here
```

---

### Exercise 2 — medium

Write a function `updateUserEmail(userId, newEmail)` that uses `pool.connect()` to manually check out a connection, runs an `UPDATE users SET email = $1 WHERE id = $2` query, and correctly releases the connection in a `finally` block whether the query succeeds or throws. Then write a small test harness that calls this function 30 times in a row with a pool configured at `max: 3`, and logs how long the batch takes — to observe queries queueing up when there are more requests than available connections.

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a `PoolMonitor` class that wraps a `pg` `Pool` instance and tracks pool health in real time:
1. Constructor takes an existing `Pool` instance.
2. Exposes a `getStats()` method returning `{ totalCount, idleCount, waitingCount }` (these are real properties/behaviors already tracked by `pg-pool` — look them up).
3. Exposes a `logStatsEvery(intervalMs)` method that logs stats on a timer (use `setInterval`, covered in Topic 30).
4. Exposes a `stopLogging()` method that clears the timer.
5. Simulate exhaustion: fire off more concurrent `pool.query()` calls than `max` allows (e.g. `max: 2` with 10 concurrent slow queries like `SELECT pg_sleep(1)`), and use your monitor to observe `waitingCount` rise above zero while queries queue for a free connection.

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
WHY POOLS EXIST
  Opening a DB connection is slow (TCP + auth + backend process spin-up)
  Postgres has a hard cap on simultaneous connections (default ~100)
  A pool reuses a small set of open connections instead of opening one per request

CREATING A POOL
  const { Pool } = require('pg');
  const dbPool = new Pool({ host, port, user, password, database, max, idleTimeoutMillis, connectionTimeoutMillis });
  → create ONCE per app process, never per request

KEY CONFIG OPTIONS (pg-pool)
  max                      → max simultaneous connections (hard ceiling)
  min                      → min idle connections kept warm (driver-dependent)
  idleTimeoutMillis        → close idle connections after this long
  connectionTimeoutMillis  → give up waiting for a free connection after this long

TWO WAYS TO QUERY
  pool.query(sql, params)          → auto checkout + query + auto release (simple queries)
  const c = await pool.connect();  → manual checkout — REQUIRED for transactions
    c.query('BEGIN'); ... c.query('COMMIT'/'ROLLBACK');
    c.release();                    → MUST call this in a finally block

POOL EXHAUSTION
  Happens when all `max` connections are busy and more requests arrive
  New requests QUEUE, waiting up to connectionTimeoutMillis
  If timeout expires → error thrown, request fails
  Symptoms: slow endpoints under load, timeout errors spiking together

SIZING max
  Rule of thumb: (Postgres max_connections) / (number of app instances) − headroom
  Too low  → requests queue and time out under load
  Too high → too many instances × pool combine to exceed DB's max_connections

NEVER DO
  new Pool() inside a route handler or per-request function
  await pool.connect() without a matching try/finally release()
  Set max higher than the DB can actually support across all app instances

SHUTDOWN
  await dbPool.end();   → closes all connections, call once during graceful shutdown
```

---

## Connected topics

- **90 — Connecting to PostgreSQL** — introduces the `pg` driver and pool basics; this topic goes deeper into tuning and failure modes of that same pool.
- **97 — Transactions** — `BEGIN`/`COMMIT`/`ROLLBACK` require a single manually checked-out connection from the pool, exactly as shown in Example 2.
- **64 — Graceful shutdown** — `dbPool.end()` must be called during shutdown so the process doesn't exit with connections still open or in-flight queries abandoned.
