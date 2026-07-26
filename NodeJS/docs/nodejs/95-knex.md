# 95 — Query builders — Knex

## What is this?

Knex is a "query builder" library for Node.js — it lets you write database queries as chained JavaScript method calls (`.select()`, `.where()`, `.join()`) instead of hand-typing raw SQL strings. It supports PostgreSQL, MySQL, SQLite, and others through the same API, and it converts your JS chain into the correct SQL dialect underneath. Think of it as a translator that sits between your code and the database — you speak fluent JavaScript, Knex speaks fluent SQL back to whichever database you connected.

## Why does it matter for backend development?

Writing raw SQL as strings is error-prone — it's easy to forget an escape, mistype a column name, or accidentally build a query vulnerable to SQL injection. Knex gives you parameterized queries by default (safe from injection), auto-escapes identifiers, and lets you compose queries dynamically (adding a `.where()` only if a filter was provided, for example) which is painful with raw string concatenation. It also gives you a single, consistent API for transactions and batch inserts across different databases, so switching from SQLite in dev to PostgreSQL in production doesn't require rewriting your queries. Most Node.js backends that aren't using a full ORM (like Prisma) reach for Knex specifically for this middle ground — more control than an ORM, more safety and ergonomics than raw SQL.

---

## Syntax / API

```js
// Install: npm install knex pg   (pg driver needed for PostgreSQL)
const knex = require('knex');

// Create a single shared connection instance — reused across the whole app
const dbConnection = knex({
  client: 'pg',                          // which database dialect to speak (pg, mysql2, sqlite3...)
  connection: {
    host: process.env.DB_HOST,           // database server address
    user: process.env.DB_USER,           // database username
    password: process.env.DB_PASSWORD,   // database password
    database: process.env.DB_NAME,       // database name
  },
  pool: { min: 2, max: 10 },              // connection pool size (Topic 93)
});

// ── SELECT with WHERE and JOIN ──────────────────────────────────────────────
dbConnection('users')                            // FROM users
  .select('users.id', 'users.email', 'orders.total') // SELECT these columns
  .join('orders', 'users.id', 'orders.user_id')  // INNER JOIN orders ON users.id = orders.user_id
  .where('users.is_active', true)                // WHERE users.is_active = true
  .then((rows) => console.log(rows));             // rows is a plain JS array of objects

// ── INSERT a single row ─────────────────────────────────────────────────────
dbConnection('users')
  .insert({ email: 'user@example.com', password_hash: 'abc123' })
  .returning('id')                                 // Postgres: get back the new row's id
  .then((inserted) => console.log(inserted));

// ── BATCH INSERT (many rows in one round trip) ──────────────────────────────
dbConnection.batchInsert('users', [
  { email: 'a@example.com', password_hash: 'hash1' },
  { email: 'b@example.com', password_hash: 'hash2' },
], 100)                                            // 100 = chunk size per INSERT statement
  .then(() => console.log('all rows inserted'));

// ── TRANSACTION (multiple queries, all-or-nothing) ──────────────────────────
dbConnection.transaction(async (trx) => {
  // trx behaves exactly like dbConnection, but scoped to this one transaction
  const [order] = await trx('orders')
    .insert({ user_id: 1, total: 250 })
    .returning('id');

  await trx('order_items').insert([
    { order_id: order.id, product_id: 10, qty: 2 },
    { order_id: order.id, product_id: 11, qty: 1 },
  ]);
  // if nothing throws, Knex commits automatically when the callback resolves
});
```

---

## How it works — line by line

`knex({ client, connection, pool })` creates one shared object — call it once at app startup, not per-request. It opens (or reuses) a pool of real database connections behind the scenes and gives you back a function you call like `dbConnection('users')` to start building a query against that table.

Each method you chain — `.select()`, `.where()`, `.join()` — doesn't touch the database yet. It just adds a piece to an internal "query description" object. Nothing actually runs until you `await` the chain or call `.then()` on it — at that exact moment Knex compiles everything you chained into one SQL string, sends it to the database using a safe parameterized query (your values are never glued into the SQL text directly, which is what prevents SQL injection), and returns the rows as plain JavaScript objects/arrays.

`.insert()` builds an `INSERT INTO ... VALUES (...)` statement from the object you pass; `.returning('id')` (PostgreSQL/SQLite feature) asks the database to hand back the auto-generated id in the same round trip instead of a second query. `dbConnection.batchInsert(table, rows, chunkSize)` is a helper that splits a large array of rows into smaller `INSERT` batches (controlled by `chunkSize`) so you don't send one gigantic query that could hit database limits or lock the table for too long.

`dbConnection.transaction(async (trx) => { ... })` opens a real database transaction (`BEGIN`). Inside the callback you use `trx` exactly like `dbConnection`, but every query runs inside that one transaction. If the callback function finishes without throwing, Knex automatically runs `COMMIT`, permanently saving all the changes. If anything inside throws an error, Knex automatically runs `ROLLBACK`, undoing everything in that block — so you never end up with half-finished multi-step writes (like an order created but its items missing).

---

## Example 1 — basic

```js
// File: src/db/connection.js
const knex = require('knex');

// One shared connection object for the whole app
const dbConnection = knex({
  client: 'sqlite3',                       // simplest dialect for local examples
  connection: { filename: './dev.sqlite3' }, // file-based DB, no server needed
  useNullAsDefault: true,                    // required by sqlite3 for missing columns
});

module.exports = dbConnection;
```

```js
// File: src/scripts/basicQuery.js
const dbConnection = require('../db/connection');

async function run() {
  // Create the table first (normally done via a migration — Topic 94)
  await dbConnection.schema.createTable('users', (table) => {
    table.increments('id');               // auto-incrementing primary key
    table.string('email').notNullable();  // required text column
    table.boolean('is_active').defaultTo(true); // defaults to true
  });

  // INSERT one row and get its new id back
  const [userId] = await dbConnection('users')
    .insert({ email: 'ada@example.com' })
    .returning('id');
  console.log('Inserted user id:', userId);

  // SELECT with a WHERE filter
  const activeUsers = await dbConnection('users')
    .select('id', 'email')          // only fetch these two columns
    .where('is_active', true);      // WHERE is_active = true
  console.log('Active users:', activeUsers);

  // Always close the connection pool when a standalone script is done
  await dbConnection.destroy();
}

run().catch((err) => console.error('Script failed:', err));
```

---

## Example 2 — real world backend use case

```js
// File: src/repositories/orderRepository.js
// A typical repository layer (Topic 117) that wraps Knex for order creation.

const dbConnection = require('../db/connection');

/**
 * Creates an order and its line items atomically, then returns the full order
 * with the customer's email joined in — a pattern used constantly in checkout flows.
 */
async function createOrderWithItems(userId, cartItems) {
  // transaction() guarantees the order row AND all its items succeed together,
  // or neither is saved — critical for financial/inventory correctness
  return dbConnection.transaction(async (trx) => {
    // Calculate the total server-side — never trust a total sent by the client
    const total = cartItems.reduce((sum, item) => sum + item.price * item.qty, 0);

    // Insert the parent order row, get its generated id back
    const [order] = await trx('orders')
      .insert({ user_id: userId, total, status: 'pending' })
      .returning(['id', 'created_at']);

    // Build all line-item rows in memory first, then insert them in one go
    const orderItemRows = cartItems.map((item) => ({
      order_id: order.id,
      product_id: item.productId,
      quantity: item.qty,
      unit_price: item.price,
    }));

    // batchInsert avoids one huge INSERT if the cart has many distinct items
    await trx.batchInsert('order_items', orderItemRows, 50);

    // Deduct stock for each product — still inside the same transaction
    for (const item of cartItems) {
      await trx('products')
        .where('id', item.productId)
        .decrement('stock_quantity', item.qty); // atomic SQL decrement
    }

    return { orderId: order.id, total, createdAt: order.created_at };
  });
}

/**
 * Fetches an order joined with the user's email and all of its line items.
 */
async function getOrderWithDetails(orderId) {
  // JOIN users to attach the buyer's email to the order row
  const order = await dbConnection('orders')
    .select('orders.id', 'orders.total', 'orders.status', 'users.email as customerEmail')
    .join('users', 'users.id', 'orders.user_id')
    .where('orders.id', orderId)
    .first();                                     // .first() returns one row, not an array

  if (!order) return null;                         // order not found — let caller 404

  // Separate query for the child rows (line items) — kept simple and readable
  const items = await dbConnection('order_items')
    .select('product_id', 'quantity', 'unit_price')
    .where('order_id', orderId);

  return { ...order, items };
}

module.exports = { createOrderWithItems, getOrderWithDetails };
```

---

## Common mistakes

### Mistake 1 — Building queries with raw string interpolation instead of Knex's builder

```js
// ❌ WRONG — defeats the entire purpose of using Knex; wide open to SQL injection
const emailInput = requestBody.email;                // untrusted user input
const rows = await dbConnection.raw(
  `SELECT * FROM users WHERE email = '${emailInput}'`  // string glued directly into SQL
);
// If emailInput is  ' OR '1'='1  the WHERE clause becomes always-true

// ✅ CORRECT — let Knex parameterize the value safely
const rows = await dbConnection('users')
  .select('*')
  .where('email', emailInput);   // Knex sends emailInput as a bound parameter, not raw text
```

### Mistake 2 — Doing multi-step writes without a transaction

```js
// ❌ WRONG — if the second insert fails, the first one is already permanently saved,
// leaving an order with no items ("orphaned" data)
const [order] = await dbConnection('orders').insert({ user_id: userId, total: 500 }).returning('id');
await dbConnection('order_items').insert({ order_id: order.id, product_id: 10, qty: 1 });
// A crash between these two lines leaves inconsistent data forever

// ✅ CORRECT — wrap related writes in a transaction so they succeed or fail together
await dbConnection.transaction(async (trx) => {
  const [order] = await trx('orders').insert({ user_id: userId, total: 500 }).returning('id');
  await trx('order_items').insert({ order_id: order.id, product_id: 10, qty: 1 });
  // both committed together, or both rolled back on any error
});
```

### Mistake 3 — Inserting rows one at a time in a loop instead of batching

```js
// ❌ WRONG — sends one INSERT round trip per row; extremely slow for large arrays
// (500 products = 500 separate network round trips to the database)
for (const productRow of productList) {
  await dbConnection('products').insert(productRow);
}

// ✅ CORRECT — batchInsert groups rows into a handful of INSERT statements
await dbConnection.batchInsert('products', productList, 100);
// 500 rows with chunk size 100 → only 5 INSERT statements total, far faster
```

---

## Practice exercises

### Exercise 1 — easy

Set up a Knex instance connected to a local SQLite file (`client: 'sqlite3'`, `useNullAsDefault: true`). Create a `products` table with columns `id` (increments), `name` (string), `price` (decimal), and `stock_quantity` (integer, default 0). Insert three products using `.insert()`, then write a `.select().where()` query that returns only products with `stock_quantity` greater than 0. Log the results and destroy the connection when done.

```js
// Write your code here
```

---

### Exercise 2 — medium

Write a function `searchOrders(filters)` where `filters` is an object that may contain any combination of `userId`, `status`, and `minTotal`. The function should build a Knex query against an `orders` table, joined with `users` to include `users.email`, applying a `.where()` clause for each filter key that is actually present (skip filters that are `undefined`). Use `.orderBy('orders.created_at', 'desc')` and return the results. Test it by calling `searchOrders({ status: 'pending' })` and `searchOrders({ userId: 3, minTotal: 100 })` and confirming the generated SQL only includes the filters you passed (hint: use `.toSQL()` or `.toString()` on the query before awaiting it to inspect it).

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a function `transferInventory(fromProductId, toProductId, quantity)` that moves stock between two products atomically using a single Knex transaction: it must decrement `stock_quantity` on `fromProductId` by `quantity`, throw an error (which should trigger an automatic rollback) if the resulting stock would go negative, then increment `stock_quantity` on `toProductId` by the same `quantity`. Also write an `importProducts(productArray)` function that batch-inserts an array of up to thousands of product objects in chunks of 200 using `batchInsert`, and returns how many rows were inserted. Write a small test script that seeds two products, calls `transferInventory` successfully, then calls it again with a quantity larger than the remaining stock and confirms the transaction rolled back (stock values unchanged).

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
SETUP
  const knex = require('knex');
  const dbConnection = knex({ client: 'pg', connection: {...}, pool: {...} });
  → create ONE instance at startup, reuse everywhere, don't recreate per-request

BASIC QUERIES
  dbConnection('table').select('col1', 'col2')      → SELECT col1, col2 FROM table
  .where('col', value)                              → WHERE col = value
  .where('col', '>', value)                         → WHERE col > value
  .join('other', 'table.id', 'other.table_id')      → INNER JOIN
  .leftJoin(...)                                     → LEFT JOIN
  .orderBy('col', 'desc')                           → ORDER BY col DESC
  .first()                                           → returns one row (or undefined), not an array
  .count('* as total')                               → COUNT aggregate

WRITES
  .insert({...})                                     → INSERT one row
  .insert({...}).returning('id')                     → INSERT and get generated id back (PG/SQLite)
  dbConnection.batchInsert(table, rowsArray, chunkSize) → INSERT many rows efficiently, in chunks
  .update({...}).where(...)                          → UPDATE ... SET ... WHERE ...
  .del().where(...)                                  → DELETE FROM ... WHERE ...
  .increment('col', n) / .decrement('col', n)        → atomic SQL increment/decrement

TRANSACTIONS
  await dbConnection.transaction(async (trx) => {
    await trx('table').insert({...});                → use trx instead of dbConnection inside
  });
  → auto COMMIT if callback resolves, auto ROLLBACK if it throws
  → never mix trx and dbConnection for related writes in the same transaction

SAFETY
  ALWAYS pass values through .where()/.insert() args   → parameterized, injection-safe
  NEVER build queries with string interpolation        → SQL injection risk
  .toSQL() / .toString()                                → inspect the generated SQL without running it

CLEANUP
  await dbConnection.destroy()                          → closes the pool (scripts/tests only,
                                                            not in a long-running server)
```

---

## Connected topics

- **93 — Connection pooling** — Knex manages its connection pool internally via the `pool` config option; understanding pooling explains Knex's `min`/`max` settings.
- **94 — Database migrations** — Knex's own migration system (`knex migrate:make`, `knex migrate:latest`) is the standard way to create the tables these queries run against.
- **97 — Transactions** — this doc's `dbConnection.transaction()` is Knex's wrapper around raw `BEGIN`/`COMMIT`/`ROLLBACK`, covered in full detail there.
