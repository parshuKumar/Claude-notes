# 87 — Integration testing with a real DB

## What is this?

Integration testing with a real database means running your tests against an actual database engine (Postgres, MySQL, etc.) instead of a fake in-memory stand-in, so you catch bugs that only show up when real SQL, real constraints, and real transactions are involved. The trick to doing this without leaving garbage data behind is to wrap every test in a database transaction and roll it back when the test finishes — like writing in pencil on a whiteboard instead of permanent marker, so you can wipe it clean after every test with zero trace left.

## Why does it matter for backend development?

Unit tests mock the database, which is fast but blind — a mock can't catch a missing `NOT NULL` constraint, a broken foreign key, a typo in a column name, or a query that works in SQLite but not Postgres. Backend developers write integration tests against a real (usually local or Dockerized) test database to verify that repository/model code, migrations, and queries actually work together correctly. The transaction-rollback pattern is the industry-standard way to keep these tests fast and isolated — each test runs inside `BEGIN ... ROLLBACK`, so tests never interfere with each other and the database is always clean for the next run, without needing to `TRUNCATE` tables manually.

---

## Syntax / API

```js
// Uses the 'pg' driver — same one used in Topic 90 (Connecting to PostgreSQL)
const { Pool } = require('pg');

// One pool connects to a DEDICATED test database — never the dev or prod one
const testPool = new Pool({
  connectionString: process.env.TEST_DATABASE_URL, // e.g. postgres://localhost/myapp_test
});

// Jest lifecycle hook — runs once before ALL tests in this file
beforeAll(async () => {
  // Nothing to connect explicitly — Pool connects lazily on first query
});

// Runs before EACH individual test — start a fresh transaction
let client; // holds the single connection used for this test's transaction

beforeEach(async () => {
  client = await testPool.connect();     // check out one connection from the pool
  await client.query('BEGIN');           // start a transaction — nothing is committed yet
});

// Runs after EACH individual test — undo everything, no matter what happened
afterEach(async () => {
  await client.query('ROLLBACK');        // undo every insert/update from this test
  client.release();                      // return the connection to the pool
});

// Runs once after ALL tests in this file finish
afterAll(async () => {
  await testPool.end();                  // close all pooled connections cleanly
});
```

---

## How it works — line by line

- `new Pool({ connectionString: ... })` creates a connection pool pointed at a **separate test database** — a real Postgres instance, just not the one your real users touch.
- `beforeEach` runs before every single `it()`/`test()` block. It checks out ONE dedicated connection (`client`) and issues `BEGIN`, which opens a transaction. Anything the test does now (inserts, updates, deletes) is only visible inside this transaction.
- The test itself runs using that same `client` to perform queries — it inserts a user, calls the function under test, and asserts on the result.
- `afterEach` runs after every test, whether it passed or failed. It issues `ROLLBACK`, which throws away every change made since `BEGIN` as if it never happened, then releases the connection back to the pool.
- Because every test rolls back, tests never see each other's leftover data and you never have to write manual cleanup code (`DELETE FROM users WHERE ...`).
- `afterAll` closes the whole pool once, after every test in the file has finished, so the test process can exit cleanly instead of hanging on open sockets.

---

## Example 1 — basic

```js
// File: tests/userRepository.test.js
const { Pool } = require('pg');
const { createUser, findUserByEmail } = require('../src/repositories/userRepository');

// Connect to the dedicated test database — configured in .env.test
const testPool = new Pool({ connectionString: process.env.TEST_DATABASE_URL });

let client; // transaction-scoped connection, reused for the whole test

beforeEach(async () => {
  client = await testPool.connect();   // grab a connection from the pool
  await client.query('BEGIN');         // open a transaction — a safe sandbox
});

afterEach(async () => {
  await client.query('ROLLBACK');      // discard everything this test wrote
  client.release();                    // give the connection back to the pool
});

afterAll(async () => {
  await testPool.end();                // shut down the pool after all tests finish
});

test('createUser inserts a row that findUserByEmail can find', async () => {
  const requestBody = { email: 'test.user@example.com', passwordHash: 'hashed123' };

  // Pass the SAME transaction client into the repository function under test
  const createdUser = await createUser(client, requestBody);
  expect(createdUser.id).toBeDefined();          // Postgres generated a real id

  const foundUser = await findUserByEmail(client, requestBody.email);
  expect(foundUser.email).toBe(requestBody.email); // round-tripped correctly
});
```

---

## Example 2 — real world backend use case

```js
// File: tests/orderService.integration.test.js
// Verifies that placing an order correctly deducts stock AND creates an order row —
// a real integration test that only a real DB with real constraints can catch.

const { Pool } = require('pg');
const { placeOrder } = require('../src/services/orderService');

const testPool = new Pool({ connectionString: process.env.TEST_DATABASE_URL });
let client;

beforeEach(async () => {
  client = await testPool.connect();
  await client.query('BEGIN'); // sandbox transaction for this test only

  // ── Seed test data the test depends on ──────────────────────────────────
  // Insert a product with known stock BEFORE the test runs its logic
  await client.query(
    `INSERT INTO products (id, name, price_cents, stock)
     VALUES ($1, $2, $3, $4)`,
    ['prod_1', 'Wireless Mouse', 2500, 10] // 10 units in stock to start
  );

  // Insert a user who will place the order
  await client.query(
    `INSERT INTO users (id, email) VALUES ($1, $2)`,
    ['user_42', 'buyer@example.com']
  );
});

afterEach(async () => {
  await client.query('ROLLBACK'); // wipes the seeded product + user + any order rows
  client.release();
});

afterAll(async () => {
  await testPool.end();
});

test('placeOrder deducts stock and creates an order row', async () => {
  const userId = 'user_42';
  const requestBody = { productId: 'prod_1', quantity: 3 }; // buy 3 mice

  const order = await placeOrder(client, userId, requestBody);

  // Assert the order row was created correctly
  expect(order.status).toBe('confirmed');
  expect(order.quantity).toBe(3);

  // Assert stock was actually decremented in the same DB — real integration proof
  const { rows } = await client.query(
    'SELECT stock FROM products WHERE id = $1',
    ['prod_1']
  );
  expect(rows[0].stock).toBe(7); // 10 - 3 = 7, verified against the real table
});

test('placeOrder rejects an order that exceeds available stock', async () => {
  const userId = 'user_42';
  const requestBody = { productId: 'prod_1', quantity: 999 }; // way more than 10 in stock

  // The real DB constraint / application check should reject this
  await expect(placeOrder(client, userId, requestBody)).rejects.toThrow('Insufficient stock');
});
```

---

## Common mistakes

### Mistake 1 — Running integration tests against the real dev/production database

```js
// ❌ WRONG — points at the same DB your app actually uses
const testPool = new Pool({ connectionString: process.env.DATABASE_URL });
// Every test run pollutes real data, and a bug could delete real user rows

// ✅ CORRECT — a dedicated, separate test database, never touched by real traffic
const testPool = new Pool({ connectionString: process.env.TEST_DATABASE_URL });
// e.g. postgres://localhost:5432/myapp_test — completely isolated from myapp_dev/myapp_prod
```

### Mistake 2 — Committing instead of rolling back, so tests leak state into each other

```js
// ❌ WRONG — COMMIT makes the insert permanent; the next test now sees stale data
beforeEach(async () => {
  client = await testPool.connect();
  await client.query('BEGIN');
});
afterEach(async () => {
  await client.query('COMMIT');   // permanently saves this test's data — bug!
  client.release();
});
// Second test run: "email already exists" errors from the FIRST test's leftover row

// ✅ CORRECT — ROLLBACK discards everything, every test starts from a clean slate
afterEach(async () => {
  await client.query('ROLLBACK'); // undoes all inserts/updates from this test
  client.release();
});
```

### Mistake 3 — Letting the code under test open its own connection instead of using the test's transaction client

```js
// ❌ WRONG — repository function grabs a NEW connection from the pool internally,
// so it runs OUTSIDE the test's transaction and its writes are never rolled back
async function createUser(requestBody) {
  const client = await testPool.connect();   // separate connection — separate transaction!
  const result = await client.query(
    'INSERT INTO users (email) VALUES ($1) RETURNING *',
    [requestBody.email]
  );
  client.release();
  return result.rows[0];
}
// This insert gets COMMITTED for real because it was never part of the test's BEGIN/ROLLBACK

// ✅ CORRECT — accept the client/transaction as a parameter so tests can inject their own
async function createUser(dbClient, requestBody) {
  const result = await dbClient.query(
    'INSERT INTO users (email) VALUES ($1) RETURNING *',
    [requestBody.email]
  );
  return result.rows[0];
}
// In production code, dbClient is the pool; in tests, it's the test's transaction client
```

---

## Practice exercises

### Exercise 1 — easy

Write a Jest test file for a `products` table that:
1. Opens a test database connection pool using `pg`
2. Uses `beforeEach`/`afterEach` to wrap each test in `BEGIN`/`ROLLBACK`
3. In one test, seeds a single product row with `stock: 5`, then queries it back and asserts the `stock` value is `5`
4. Closes the pool in `afterAll`

```js
// Write your code here
```

---

### Exercise 2 — medium

Write a `userRepository` module with two functions, `createUser(dbClient, requestBody)` and `deleteUser(dbClient, userId)`, both accepting an injected `dbClient` (never opening their own connection). Then write an integration test suite that:
1. Uses the transaction-rollback pattern for isolation between tests
2. Seeds an `authToken`/`sessionId` free — just seed a plain user row with `email` and `passwordHash`
3. Tests that `createUser` returns a row with a generated `id`
4. Tests that `deleteUser` removes the row, and a subsequent `SELECT` for that `userId` returns zero rows
5. Confirms in a final test that data from a previous test (e.g. a duplicate email) does NOT cause a conflict — proving rollback actually isolated the tests

```js
// Write your code here
```

---

### Exercise 3 — hard

Build an integration test suite for a `transferFunds(dbClient, fromAccountId, toAccountId, amountCents)` function that moves money between two accounts inside its own internal transaction-safe SQL (`UPDATE ... WHERE balance >= amount` style checks). Your test suite must:
1. Seed two accounts with known starting balances before each test (e.g. `acc_a: 10000` cents, `acc_b: 0` cents)
2. Test the happy path — transferring funds correctly decreases the source and increases the destination, verified by querying both rows back after the call
3. Test the failure path — attempting to transfer more than `acc_a`'s balance throws, AND assert that neither balance actually changed afterward (the failed transfer must not have partially applied)
4. Test a concurrent-looking scenario: call `transferFunds` twice in a row within the same test (two separate transfers) and assert both balances end up mathematically correct after both calls
5. Use the `BEGIN`/`ROLLBACK` pattern around the whole test so no seeded account data ever survives between tests

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
CORE IDEA
  Integration test = real DB engine, real SQL, real constraints — not a mock
  Isolation trick   = wrap every test in BEGIN ... ROLLBACK, never COMMIT

SETUP CHECKLIST
  1. Separate TEST_DATABASE_URL — never point tests at dev/prod DB
  2. Run real migrations against the test DB before the test suite starts
  3. beforeEach: client = pool.connect(); client.query('BEGIN')
  4. afterEach:  client.query('ROLLBACK'); client.release()
  5. afterAll:   pool.end()

SEEDING TEST DATA
  - Insert rows INSIDE the transaction (in beforeEach or inside the test itself)
  - Since the transaction rolls back, seeded rows never persist — no manual cleanup
  - Keep seed data minimal and explicit — only what the test actually needs

WHY ROLLBACK BEATS TRUNCATE/DELETE
  TRUNCATE/DELETE between tests → slower, resets sequences, easy to forget a table
  BEGIN/ROLLBACK per test        → instant, guaranteed complete, zero leftover state

DEPENDENCY INJECTION IS MANDATORY
  Repository/service functions must accept a `dbClient` parameter
  Production code passes the pool; tests pass the transaction client
  Never let code under test open its own separate connection

GOTCHAS
  - Nested transactions: Postgres doesn't support real nested BEGIN — use SAVEPOINT
    if the code under test also calls BEGIN internally
  - Async leaks: always release()/end() in afterEach/afterAll or Jest hangs on exit
  - Don't share ONE client across parallel tests — each test needs its own connection

TYPICAL FILE LAYOUT
  .env.test              → TEST_DATABASE_URL=postgres://localhost/myapp_test
  tests/setup.js         → shared beforeEach/afterEach transaction helpers
  tests/*.integration.test.js → one file per repository/service being tested
```

---

## Connected topics

- **97 — Transactions** — the `BEGIN`/`COMMIT`/`ROLLBACK` mechanics this topic relies on, explained in full ACID detail
- **90 — Connecting to PostgreSQL** — the `pg` driver, connection pool, and query execution patterns used to build the test setup here
- **98 — Database seeding and fixtures** — deeper patterns (factories, faker.js) for generating the test data seeded inside each transaction
