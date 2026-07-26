# 90 — Connecting to PostgreSQL

## What is this?

`pg` (also called `node-postgres`) is the standard Node.js driver for talking to a PostgreSQL database. It lets your Node process open a network connection to Postgres, send SQL queries, and get rows back as JavaScript objects. Think of Postgres as a filing cabinet in another building — `pg` is the courier service that carries your requests ("give me user #42") over there and brings the answer back, and a **connection pool** is a small fleet of couriers kept on standby so you never have to hire a brand-new one for every single trip.

---

## Why does it matter for backend development?

Almost every backend service needs to persist data — users, orders, sessions, products — and PostgreSQL is one of the most widely used relational databases in production systems. A backend developer uses `pg` to run SQL queries from route handlers, services, or repository layers: fetching a user by `userId`, inserting a new order, updating a row inside a transaction. Doing this correctly — with a pool instead of one-off connections, with parameterized queries instead of string concatenation, and with proper error handling — is the difference between an app that survives real traffic and one that crashes or gets SQL-injected on day one. Every Express/Fastify API that talks to Postgres is built on top of exactly what this topic covers.

---

## Syntax / API

```js
// Install first: npm install pg
const { Pool } = require('pg');   // Pool manages a set of reusable connections

// Create a pool — this does NOT connect immediately, it connects lazily on first query
const dbPool = new Pool({
  host: process.env.DB_HOST || 'localhost',     // where Postgres is running
  port: process.env.DB_PORT || 5432,            // default Postgres port
  user: process.env.DB_USER || 'postgres',      // login role
  password: process.env.DB_PASSWORD,            // NEVER hardcode this — use env vars
  database: process.env.DB_NAME || 'app_db',    // which database to use
  max: 10,                                       // max simultaneous connections in the pool
  idleTimeoutMillis: 30000,                      // close idle clients after 30s
  connectionTimeoutMillis: 5000,                 // fail fast if a connection can't be made
});

// ── Running a query (recommended: pool.query, no manual connect/release needed) ──
async function findUserById(userId) {
  // $1 is a placeholder — pg substitutes it safely (prevents SQL injection)
  const result = await dbPool.query('SELECT * FROM users WHERE id = $1', [userId]);
  return result.rows[0];   // rows is always an array; [0] is the first match or undefined
}

// ── Listening for unexpected pool-level errors (idle client crashes, network drop) ──
dbPool.on('error', (err) => {
  console.error('Unexpected error on idle Postgres client:', err);
  // Do NOT crash the whole process here — log it and let the pool recover/reconnect
});

module.exports = dbPool;   // export the pool so the whole app shares one instance
```

---

## How it works — line by line

- `require('pg')` loads the driver you installed with npm; `Pool` is the class you use almost always instead of a single raw `Client`.
- `new Pool({...})` creates a manager that will open real TCP connections to Postgres **only when needed**, up to the `max` you set. It does not open 10 connections immediately — it opens them on demand and reuses them.
- The connection settings (`host`, `port`, `user`, `password`, `database`) are exactly what you'd type into `psql` to connect manually — the driver just does it over the network for you.
- `max` caps how many connections can be open to Postgres at once from this app instance — Postgres itself has a limit on total connections, so this protects the database from being overwhelmed.
- `pool.query(sql, values)` grabs a free connection from the pool, runs your SQL with the values safely substituted in place of `$1`, `$2`, etc., and automatically gives the connection back to the pool when done — you never have to manually "release" it for simple one-off queries.
- The placeholders (`$1`, `$2`, ...) exist so that user-provided values are never pasted directly into the SQL string — Postgres treats them strictly as data, not as executable SQL, which is what stops SQL injection.
- `result.rows` is always an array of plain JavaScript objects, one per matching database row, with column names as keys.
- The `dbPool.on('error', ...)` listener catches problems that happen on connections sitting idle in the pool (for example, the network to the database dropping) — without this listener, such an error would crash your entire Node process.

---

## Example 1 — basic

```js
// File: src/db/pool.js
// Sets up one shared connection pool for the whole app.

const { Pool } = require('pg');   // import the Pool class from the pg driver

// Create the pool once — this instance is reused everywhere in the app
const dbPool = new Pool({
  host: 'localhost',        // Postgres server address
  port: 5432,                // Postgres default port
  user: 'postgres',          // database login user
  password: 'devpassword',   // password for local dev only — real apps use env vars
  database: 'shop_db',       // the database name to connect to
  max: 10,                    // never open more than 10 connections at once
});

// Log a clear message the first time the pool actually connects
dbPool.on('connect', () => {
  console.log('New client connected to Postgres'); // fires once per new physical connection
});

// Catch idle-client errors so they don't crash the process
dbPool.on('error', (err) => {
  console.error('Postgres pool error:', err.message); // log and move on
});

async function runDemoQuery() {
  // Simple query with no user input — safe to write directly
  const result = await dbPool.query('SELECT NOW() AS current_time');
  console.log('DB time:', result.rows[0].current_time); // print the server's current timestamp
}

runDemoQuery(); // call the async function (top-level call, fine for a demo script)

module.exports = dbPool; // share this single pool instance across the app
```

---

## Example 2 — real world backend use case

```js
// File: src/repositories/orderRepository.js
// A repository-style module an Express route would call to fetch/create orders.

const dbPool = require('../db/pool'); // shared pool created once at app startup

// Fetch a single order, joined with the user who placed it
async function getOrderById(orderId) {
  const sql = `
    SELECT orders.id, orders.total_cents, orders.status, users.email
    FROM orders
    JOIN users ON users.id = orders.user_id
    WHERE orders.id = $1
  `;                                        // $1 keeps orderId out of the raw SQL string

  try {
    const result = await dbPool.query(sql, [orderId]); // run the query safely
    return result.rows[0] || null;                      // null if no matching order
  } catch (err) {
    console.error(`Failed to fetch order ${orderId}:`, err.message); // log context
    throw new Error('Could not retrieve order'); // rethrow a clean error for the caller
  }
}

// Insert a new order and return the created row
async function createOrder({ userId, totalCents }) {
  const sql = `
    INSERT INTO orders (user_id, total_cents, status, created_at)
    VALUES ($1, $2, 'pending', NOW())
    RETURNING id, user_id, total_cents, status
  `;                                        // RETURNING sends the new row straight back

  try {
    const result = await dbPool.query(sql, [userId, totalCents]); // parameterized insert
    return result.rows[0]; // the newly created order
  } catch (err) {
    if (err.code === '23503') {
      // Postgres foreign key violation code — userId doesn't exist in users table
      throw new Error(`User ${userId} does not exist`);
    }
    console.error('Failed to create order:', err.message); // log unexpected errors
    throw new Error('Could not create order'); // hide raw DB error from the API caller
  }
}

module.exports = { getOrderById, createOrder }; // exported for use by controllers/routes
```

---

## Common mistakes

### Mistake 1 — Building SQL with string concatenation instead of parameters

```js
// ❌ WRONG — user input goes straight into the SQL string, opening the door to SQL injection
async function findUserByEmail(userEmail) {
  const sql = `SELECT * FROM users WHERE email = '${userEmail}'`; // DANGEROUS
  const result = await dbPool.query(sql);
  return result.rows[0];
}
// If userEmail is: ' OR '1'='1  → returns every row in the table

// ✅ CORRECT — always use placeholders, let pg escape the value for you
async function findUserByEmail(userEmail) {
  const sql = 'SELECT * FROM users WHERE email = $1'; // $1 is safely substituted
  const result = await dbPool.query(sql, [userEmail]); // value passed separately
  return result.rows[0];
}
```

### Mistake 2 — Creating a new Pool (or Client) on every request

```js
// ❌ WRONG — a fresh pool per request exhausts Postgres's connection limit fast
app.get('/api/users/:userId', async (req, res) => {
  const dbPool = new Pool({ /* ...config... */ }); // new pool EVERY request — very wasteful
  const result = await dbPool.query('SELECT * FROM users WHERE id = $1', [req.params.userId]);
  res.json(result.rows[0]);
});

// ✅ CORRECT — create ONE pool at app startup, import and reuse it everywhere
const dbPool = require('../db/pool'); // single shared pool, created once

app.get('/api/users/:userId', async (req, res) => {
  const result = await dbPool.query('SELECT * FROM users WHERE id = $1', [req.params.userId]);
  res.json(result.rows[0]); // reuses a connection from the shared pool
});
```

### Mistake 3 — Not handling query errors, letting them crash the request (or process)

```js
// ❌ WRONG — no try/catch; a bad query or a dropped connection crashes the whole handler
app.get('/api/orders/:orderId', async (req, res) => {
  const result = await dbPool.query(
    'SELECT * FROM orders WHERE id = $1',
    [req.params.orderId]
  ); // if this throws, the request hangs or the process emits an unhandled rejection
  res.json(result.rows[0]);
});

// ✅ CORRECT — wrap the query, respond with a proper error status, and log the failure
app.get('/api/orders/:orderId', async (req, res) => {
  try {
    const result = await dbPool.query(
      'SELECT * FROM orders WHERE id = $1',
      [req.params.orderId]
    );
    if (!result.rows[0]) {
      return res.status(404).json({ error: 'Order not found' }); // no row = 404, not a crash
    }
    res.json(result.rows[0]);
  } catch (err) {
    console.error('Query failed:', err.message); // log for debugging
    res.status(500).json({ error: 'Internal server error' }); // never leak raw DB errors to clients
  }
});
```

---

## Practice exercises

### Exercise 1 — easy

Set up a Postgres connection pool in a new file `src/db/pool.js` using `pg`. Configure it with `host`, `port`, `user`, `password`, and `database` values read from `process.env` (with sensible local defaults). Add an `error` listener that logs any pool-level error without crashing the process. Then write a small script that calls `dbPool.query('SELECT version()')` and logs the Postgres version string that comes back.

```js
// Write your code here
```

---

### Exercise 2 — medium

Write a module `src/repositories/productRepository.js` with two functions:
1. `getProductById(productId)` — runs a parameterized `SELECT` query for a single product by id and returns the row, or `null` if not found. Wrap the query in a try/catch and rethrow a clean, generic error on failure (do not leak the raw Postgres error).
2. `listProductsByCategory(category, minPrice)` — runs a parameterized `SELECT` query returning all products in a given `category` with `price >= minPrice`, ordered by `price` ascending. Return the array of rows (empty array if none match).

Both functions must use `$1`, `$2` placeholders — never string concatenation.

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a small `UserService` class backed by a shared `pg` Pool that:
1. Takes the shared `dbPool` instance in its constructor
2. Has an async `createUser({ email, passwordHash })` method that inserts a new row into a `users` table and returns the created row using `RETURNING *`
3. Catches a Postgres unique-violation error (error code `'23505'`, thrown when the email already exists) and rethrows a clear `Error('Email already registered')` instead of the raw driver error
4. Has an async `getUserByEmail(email)` method returning the row or `null`
5. Has an async `updateLastLogin(userId)` method that updates a `last_login_at` column to `NOW()` for the given user and returns the updated row (`null` if the user doesn't exist)
6. Every method must use parameterized queries and handle query errors without letting them crash the caller — log the error and rethrow a clean, descriptive `Error`

Test it by creating a user, trying to create the same user again (should throw the friendly duplicate-email error), fetching it by email, and updating its last login.

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
INSTALL
  npm install pg

CONNECTING
  const { Pool } = require('pg');
  const dbPool = new Pool({ host, port, user, password, database, max });
  → connects lazily, on first query — not immediately

RUNNING QUERIES
  await dbPool.query(sql, values)         → preferred, auto acquire + release
  result.rows                             → array of row objects
  result.rows[0]                          → first row, or undefined if none
  result.rowCount                         → number of rows affected/returned

PLACEHOLDERS (always use these — never string concatenation)
  'SELECT * FROM users WHERE id = $1'     with values: [userId]
  'INSERT INTO orders (...) VALUES ($1,$2)' with values: [userId, totalCents]
  ... RETURNING *                          → get inserted/updated row back immediately

POOL CONFIG KEYS
  max                     → max simultaneous connections (tune to Postgres's own limit)
  idleTimeoutMillis       → close idle connections after N ms
  connectionTimeoutMillis → fail fast instead of hanging forever

ERROR HANDLING
  dbPool.on('error', cb)   → catches idle-client / network errors, prevents process crash
  try/catch around every await dbPool.query(...) in route handlers and services
  err.code === '23505'     → unique constraint violation
  err.code === '23503'     → foreign key violation
  Never leak raw err.message to API clients — log it, return a generic message

NEVER DO
  Template-literal user input into SQL strings  → SQL injection risk
  new Pool() per request                         → exhausts Postgres connection limit
  Hardcoded password in source                   → use process.env + dotenv
  Letting a query error crash the whole process  → always try/catch + pool.on('error')
```

---

## Connected topics

- **93 — Connection pooling** — goes deeper into why pools exist, pool size tuning, and pool exhaustion, expanding directly on the `Pool` setup shown here
- **76 — SQL injection prevention** — the parameterized-query pattern (`$1`, `$2`) used throughout this doc is the core defense covered in that topic
- **97 — Transactions** — building on a single pooled client to run `BEGIN`/`COMMIT`/`ROLLBACK` around multiple related queries
