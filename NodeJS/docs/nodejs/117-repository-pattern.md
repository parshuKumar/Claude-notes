# 117 — Repository pattern in Node

## What is this?

The Repository pattern puts one layer of code between your business logic and your actual database, so the rest of your app never talks to the database directly — it only talks to the repository. Think of it like a librarian: you ask the librarian for "the book about dinosaurs," and you never need to know whether the book is on shelf 4B, in a back room, or in another branch entirely. The librarian (the repository) knows the storage details; you just ask for what you want. Your `userService` or `orderController` should feel the same way about a `UserRepository` — it asks for a user by ID and gets one back, with zero knowledge of SQL, MongoDB, or file storage underneath.

## Why does it matter for backend development?

Without a repository, SQL queries (or `mongoose.find()` calls) end up scattered across controllers, services, and route handlers. That means every business-logic file is welded to one specific database — switching from PostgreSQL to MongoDB, or even changing a table's column names, forces you to hunt through the whole codebase. A repository isolates that change to one file. It also makes testing dramatically easier: in unit tests you swap the real database-backed repository for a fake in-memory one, so your business logic tests run in milliseconds with no real database connection required. Every backend built with any seriousness — from a small Express API to a large microservice — uses this pattern to keep the "what the app does" code separate from the "where the data lives" code.

---

## Syntax / API

```js
// File: src/repositories/userRepository.js
// A repository is just an object (or class) exposing methods your app needs —
// findById, findByEmail, create, update, delete — with the database details hidden inside.

const dbConnection = require('../db/connection');   // the real database client (e.g. pg Pool)

class UserRepository {
  // Find one user by their primary key
  async findById(userId) {
    const result = await dbConnection.query(
      'SELECT id, email, name FROM users WHERE id = $1',
      [userId]
    );
    return result.rows[0] || null;   // return null instead of undefined for a clean "not found"
  }

  // Find one user by email — used during login
  async findByEmail(email) {
    const result = await dbConnection.query(
      'SELECT id, email, name, password_hash FROM users WHERE email = $1',
      [email]
    );
    return result.rows[0] || null;
  }

  // Insert a new user row and return the created record
  async create(userData) {
    const result = await dbConnection.query(
      'INSERT INTO users (email, name, password_hash) VALUES ($1, $2, $3) RETURNING id, email, name',
      [userData.email, userData.name, userData.passwordHash]
    );
    return result.rows[0];
  }
}

module.exports = new UserRepository();   // export a single shared instance (a singleton)
```

---

## How it works — line by line

- `class UserRepository { ... }` groups every user-related database operation under one name, instead of spreading `SELECT` statements across the codebase.
- Each method (`findById`, `findByEmail`, `create`) has a name that describes **what** the caller wants, not **how** it's stored — the caller never sees the word "SQL" or "table."
- Inside each method, `dbConnection.query(...)` is the only place that actually knows this app uses PostgreSQL with a `users` table and specific column names.
- `result.rows[0] || null` normalizes the database driver's specific response shape (`.rows` is a `pg`-specific detail) into a plain, predictable value — `null` or a plain object — that the rest of the app can rely on regardless of which database library is underneath.
- `module.exports = new UserRepository()` exports one ready-to-use object, so every file that `require`s this repository shares the same instance and the same underlying connection.
- The business logic that calls this repository (a service, a controller) writes `userRepository.findById(userId)` and never has to change even if the SQL query, the table name, or the entire database engine changes later — only this one file needs updating.

---

## Example 1 — basic

```js
// File: src/repositories/inMemoryUserRepository.js
// The simplest possible repository — no real database at all, just an array.
// This is what you swap in during tests, or use while prototyping before a DB exists.

class InMemoryUserRepository {
  constructor() {
    this.users = [];       // plain array acts as our "table"
    this.nextId = 1;       // fake auto-increment counter
  }

  // Return a shallow copy so callers can't mutate our internal array by accident
  async findById(userId) {
    const user = this.users.find((u) => u.id === userId);
    return user ? { ...user } : null;
  }

  // Create a new user with a generated id, mimicking a real DB's behavior
  async create(userData) {
    const newUser = { id: this.nextId++, ...userData };
    this.users.push(newUser);
    return { ...newUser };
  }

  // List everyone — useful for debugging/tests only, not usually exposed in production
  async findAll() {
    return this.users.map((u) => ({ ...u }));
  }
}

module.exports = InMemoryUserRepository;

// ── Using it ─────────────────────────────────────────────────────────────
async function demo() {
  const repo = new InMemoryUserRepository();               // instantiate the fake repository

  const created = await repo.create({ email: 'ada@example.com', name: 'Ada' }); // add a user
  console.log('Created:', created);                        // → { id: 1, email: 'ada@example.com', name: 'Ada' }

  const found = await repo.findById(created.id);            // look it up by the generated id
  console.log('Found by id:', found);                       // → same object, safely copied

  const missing = await repo.findById(999);                 // id that doesn't exist
  console.log('Missing user:', missing);                    // → null, never a crash
}

demo();
```

---

## Example 2 — real world backend use case

```js
// File: src/repositories/orderRepository.js
// A realistic repository for an e-commerce backend, wrapping a PostgreSQL pool.
// Notice: it exposes an INTERFACE (method names) the service layer relies on,
// so this file could later be rewritten for MongoDB without touching orderService.js.

const dbConnection = require('../db/connection');   // pg.Pool instance, created once at startup

class OrderRepository {
  // Fetch a single order along with its line items in one round trip
  async findById(orderId) {
    const orderResult = await dbConnection.query(
      'SELECT id, user_id, status, total_amount, created_at FROM orders WHERE id = $1',
      [orderId]
    );
    const order = orderResult.rows[0];
    if (!order) return null;                        // no such order — let the caller decide what to do

    const itemsResult = await dbConnection.query(
      'SELECT product_id, quantity, unit_price FROM order_items WHERE order_id = $1',
      [orderId]
    );
    return { ...order, items: itemsResult.rows };    // attach line items to the order object
  }

  // Fetch every order belonging to one user, most recent first
  async findByUserId(userId) {
    const result = await dbConnection.query(
      'SELECT id, status, total_amount, created_at FROM orders WHERE user_id = $1 ORDER BY created_at DESC',
      [userId]
    );
    return result.rows;
  }

  // Insert a new order inside a transaction so the order + items are atomic
  async create(userId, items) {
    const client = await dbConnection.connect();     // grab a dedicated client for the transaction
    try {
      await client.query('BEGIN');                   // start the transaction

      const totalAmount = items.reduce((sum, item) => sum + item.quantity * item.unitPrice, 0);

      const orderResult = await client.query(
        'INSERT INTO orders (user_id, status, total_amount) VALUES ($1, $2, $3) RETURNING id',
        [userId, 'pending', totalAmount]
      );
      const orderId = orderResult.rows[0].id;

      for (const item of items) {                    // insert each line item tied to the new order
        await client.query(
          'INSERT INTO order_items (order_id, product_id, quantity, unit_price) VALUES ($1, $2, $3, $4)',
          [orderId, item.productId, item.quantity, item.unitPrice]
        );
      }

      await client.query('COMMIT');                  // all inserts succeeded — save permanently
      return this.findById(orderId);                  // return the full order with items attached
    } catch (error) {
      await client.query('ROLLBACK');                 // undo everything if any insert failed
      throw error;                                     // let the service layer decide how to respond
    } finally {
      client.release();                                // always return the connection to the pool
    }
  }

  // Update just the status column — used by payment/shipping webhooks
  async updateStatus(orderId, newStatus) {
    const result = await dbConnection.query(
      'UPDATE orders SET status = $1 WHERE id = $2 RETURNING id, status',
      [newStatus, orderId]
    );
    return result.rows[0] || null;
  }
}

module.exports = new OrderRepository();

// ── File: src/services/orderService.js ──────────────────────────────────
// The service layer NEVER writes SQL — it only calls repository methods by name.
const orderRepository = require('../repositories/orderRepository');

async function placeOrder(userId, requestBody) {
  if (!requestBody.items || requestBody.items.length === 0) {
    const error = new Error('Order must contain at least one item');
    error.statusCode = 400;
    throw error;
  }
  const order = await orderRepository.create(userId, requestBody.items);   // delegate to repository
  return order;
}

module.exports = { placeOrder };
```

---

## Common mistakes

### Mistake 1 — Writing raw queries directly inside controllers/services

```js
// ❌ WRONG — the controller knows SQL and table structure; changing the DB means
// hunting through every route file in the whole project
app.get('/users/:userId', async (req, res) => {
  const result = await dbConnection.query('SELECT * FROM users WHERE id = $1', [req.params.userId]);
  res.json(result.rows[0]);
});

// ✅ CORRECT — controller asks the repository, stays ignorant of SQL entirely
const userRepository = require('../repositories/userRepository');

app.get('/users/:userId', async (req, res) => {
  const user = await userRepository.findById(req.params.userId);   // no SQL here at all
  if (!user) return res.status(404).json({ error: 'User not found' });
  res.json(user);
});
```

### Mistake 2 — Leaking the database driver's shape out of the repository

```js
// ❌ WRONG — returns pg's raw result object; callers now depend on `.rows`,
// which breaks the moment you switch database libraries
async function findById(userId) {
  return dbConnection.query('SELECT * FROM users WHERE id = $1', [userId]);
  // caller has to know to write: (await userRepository.findById(id)).rows[0]
}

// ✅ CORRECT — repository unwraps the driver-specific shape before returning
async function findById(userId) {
  const result = await dbConnection.query('SELECT * FROM users WHERE id = $1', [userId]);
  return result.rows[0] || null;   // caller just gets a plain object or null
}
```

### Mistake 3 — Making the repository depend on a concrete class instead of an injected connection

```js
// ❌ WRONG — repository imports and creates its own database client internally,
// so tests can never substitute a fake one without monkey-patching require()
const { Pool } = require('pg');

class UserRepository {
  constructor() {
    this.pool = new Pool({ connectionString: process.env.DATABASE_URL });   // hardcoded, untestable
  }
  async findById(userId) {
    const result = await this.pool.query('SELECT * FROM users WHERE id = $1', [userId]);
    return result.rows[0] || null;
  }
}

// ✅ CORRECT — connection is passed in (dependency injection, see Topic 116),
// so tests can pass a fake pool with a fake .query() method
class UserRepository {
  constructor(dbConnection) {
    this.dbConnection = dbConnection;   // injected, swappable, mockable
  }
  async findById(userId) {
    const result = await this.dbConnection.query('SELECT * FROM users WHERE id = $1', [userId]);
    return result.rows[0] || null;
  }
}

module.exports = UserRepository;
// Usage: new UserRepository(realPool)  in production
//        new UserRepository(fakePool)  in tests
```

---

## Practice exercises

### Exercise 1 — easy

Create an `InMemoryProductRepository` class (no real database) that manages products in a plain array. It must have:
1. A constructor that initializes an empty array and a `nextId` counter starting at 1
2. A `create(productData)` method that stores a new product with an auto-generated `id` and returns it
3. A `findById(productId)` method that returns the matching product or `null` if not found
4. A `findAll()` method that returns every stored product as an array

Test it by creating two products, finding one by id, and calling `findAll()` to confirm both are present.

```js
// Write your code here
```

---

### Exercise 2 — medium

Build a `UserRepository` class that depends on an injected `dbConnection` (you can fake the connection with a simple object whose `query` method returns hardcoded data — no real database needed for this exercise). It must have:
1. A constructor accepting `dbConnection` and storing it
2. A `findByEmail(email)` method that calls `dbConnection.query(sql, params)` and returns `result.rows[0] || null`
3. A `create(userData)` method that inserts a user and returns the created row
4. A `updateName(userId, newName)` method that updates the `name` column and returns the updated row

Then write a fake `dbConnection` object with a `query(sql, params)` method that logs the SQL and params it received and returns a fake `{ rows: [...] }` result, so you can run and verify all three repository methods without any real database.

```js
// Write your code here
```

---

### Exercise 3 — hard

Design a repository layer for a `Comment` feature that can run against either an in-memory store or a fake SQL-like store, decided by a factory function — proving the rest of the app never needs to know which one is active. Build:
1. A shared contract (in comments or a base class) describing the methods every comment repository must implement: `create(postId, authorId, text)`, `findByPostId(postId)`, `deleteById(commentId)`
2. An `InMemoryCommentRepository` implementing that contract using an array
3. A `SqlCommentRepository` implementing the same contract, using an injected fake `dbConnection` (simulate `.query()` returning objects shaped like real SQL results)
4. A factory function `createCommentRepository(type, dbConnection)` that returns the in-memory version when `type === 'memory'` and the SQL version when `type === 'sql'`
5. A small `commentService` object (or set of functions) that receives a repository instance through its constructor/parameters and calls only the shared method names — write one test-like script that runs the SAME service logic against both repository implementations and logs that both produce the expected `findByPostId` results, proving the service code never changed between the two.

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
WHAT IT IS
  A repository is a class/object that hides ALL database access behind
  plain method names: findById, findByEmail, create, update, delete.
  Business logic calls the repository — never writes SQL/queries itself.

WHY USE IT
  - Swap databases (Postgres → Mongo) by rewriting ONE file, not the whole app
  - Unit test business logic with a fake in-memory repository — no real DB needed
  - Keep controllers/services thin and focused on "what", not "how"

SHAPE OF A REPOSITORY METHOD
  async findById(id)      → returns one record or null
  async findByX(value)    → returns one record or null (lookup by any field)
  async findAll(filters)  → returns an array (never throws for "no results")
  async create(data)      → returns the created record
  async update(id, data)  → returns the updated record or null
  async delete(id)        → returns true/false or the deleted record

KEY RULES
  - Never let the database driver's raw response leak out (e.g. pg's `.rows`)
    → always unwrap it inside the repository, return plain objects/arrays
  - Inject the connection (constructor param), don't hardcode it
    → makes the repository swappable and testable
  - One repository per aggregate/entity (UserRepository, OrderRepository)
    → not one giant repository for the whole database

TESTING WITH MOCKS
  - InMemoryRepository = a fake implementation using a plain array
  - Fake dbConnection = an object with a `.query()` method you control in tests
  - Service/business logic tests run against the fake — fast, no network, no DB setup

COMMON PATTERNS
  module.exports = new UserRepository(dbConnection);   // shared singleton instance
  class UserRepository { constructor(dbConnection) {...} }  // DI-friendly version
  const repo = type === 'test' ? new InMemoryRepo() : new SqlRepo(pool);  // factory swap

NEVER DO
  Writing SQL inside a controller or service file
  Returning driver-specific shapes (result.rows) out of the repository
  Hardcoding `new Pool(...)` inside the repository class itself
```

---

## Connected topics

- **116 — Dependency injection in Node** — repositories are almost always constructed with an injected `dbConnection`, which is exactly what DI provides in a structured way.
- **118 — Clean architecture in Node** — the repository is the "infrastructure" layer in clean architecture, sitting behind the domain and use-case layers via the dependency rule.
- **85 — Mocking and stubbing** — swapping a real repository for an in-memory or mocked one during tests is the core technique this topic teaches, applied directly to the repository pattern.
