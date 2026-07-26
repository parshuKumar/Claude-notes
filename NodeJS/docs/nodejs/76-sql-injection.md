# 76 — SQL injection prevention

## What is this?

SQL injection is an attack where a malicious user sneaks raw SQL syntax into an input field so the database executes commands the developer never intended. SQL injection **prevention** is the set of coding practices — mainly parameterized queries, prepared statements, and ORM query builders — that stop user input from ever being treated as executable SQL. Think of it like a bank teller who only accepts the *amount* you write on a withdrawal slip and never lets you scribble extra instructions like "and also empty the vault" — the slip's format is fixed, and your input can only ever fill in the blank.

## Why does it matter for backend development?

Every backend that touches a SQL database (PostgreSQL, MySQL, SQLite) builds queries using data that came from a client — a login form, a search box, a URL parameter. If that data is glued directly into a SQL string, an attacker can type `' OR '1'='1` into a password field and log in as any user, or type `'; DROP TABLE users; --` and destroy your data. SQL injection has been in the OWASP Top 10 for over a decade because it is trivially exploitable and catastrophically damaging — full data theft, authentication bypass, or total data loss. Every backend developer who writes a single raw SQL query must know how to write it safely, and every developer using an ORM must know which ORM methods silently reopen this hole.

---

## Syntax / API

```js
// The 'pg' driver (PostgreSQL) — used here as the reference client
const { Pool } = require('pg');

const dbConnection = new Pool({
  connectionString: process.env.DATABASE_URL, // never hardcode credentials
});

// ── DANGEROUS — string concatenation builds SQL from raw user input ────────
async function findUserUnsafe(userId) {
  // userId could be: "1 OR 1=1" — returns EVERY user in the table
  const query = `SELECT * FROM users WHERE id = ${userId}`;
  return dbConnection.query(query);
}

// ── SAFE — parameterized query using placeholders ($1, $2, ...) ────────────
async function findUserSafe(userId) {
  // $1 is a PLACEHOLDER — the driver sends the SQL text and the value separately
  const query = 'SELECT * FROM users WHERE id = $1';
  // values array maps to placeholders in order — the driver escapes/types them
  return dbConnection.query(query, [userId]);
}

// ── SAFE — prepared statement (named, reusable, pre-compiled by the DB) ────
async function findUserPrepared(userId) {
  const preparedQuery = {
    name: 'find-user-by-id',        // name lets Postgres cache the query plan
    text: 'SELECT * FROM users WHERE id = $1',
    values: [userId],
  };
  return dbConnection.query(preparedQuery);
}

// ── SAFE — ORM query builder (Knex) never lets values touch raw SQL ────────
// const knexDb = require('knex')({ client: 'pg', connection: process.env.DATABASE_URL });
// knexDb('users').where({ id: userId }).select('*');  // value bound safely
```

---

## How it works — line by line

- `const query = \`SELECT * FROM users WHERE id = ${userId}\`` builds one final string before the database ever sees it. If `userId` contains SQL keywords, the database cannot tell them apart from your intended query — they just become part of the command.
- `dbConnection.query('SELECT * FROM users WHERE id = $1', [userId])` sends **two separate things** to the database: the fixed query text with a placeholder, and a values array. The database driver transmits these on separate channels of the wire protocol.
- Because the placeholder and the value travel separately, the database engine parses the SQL structure **first**, before it ever looks at what `userId` contains. Whatever is in `userId` is only ever treated as a literal value to compare — it can never change the shape of the query, no matter what characters it contains.
- A named prepared statement (`name: 'find-user-by-id'`) tells Postgres to compile the query plan once and reuse it on every call with different values — this is both safer and faster for queries you run repeatedly.
- The ORM query builder (`knexDb('users').where({ id: userId })`) generates the same parameterized SQL under the hood — you write JavaScript objects, and the library is responsible for turning them into placeholders and values, so you never touch a raw SQL string at all.

---

## Example 1 — basic

```js
// File: src/db/pool.js
const { Pool } = require('pg');

// Create one shared connection pool for the whole app (see Topic 93)
const dbConnection = new Pool({
  connectionString: process.env.DATABASE_URL,
});

// A safe login lookup — the classic SQL injection target
async function findUserByEmail(email) {
  // Placeholder $1 — email is passed as data, never concatenated into the string
  const query = 'SELECT id, email, password_hash FROM users WHERE email = $1';
  const result = await dbConnection.query(query, [email]);
  return result.rows[0] || null; // rows[0] is the matched user, or undefined
}

// Try it with a malicious-looking input — it is treated as a LITERAL string
findUserByEmail("attacker' OR '1'='1").then((user) => {
  console.log(user);
  // → null — the whole string is compared literally against the email column,
  //   it never becomes part of the SQL logic
});

module.exports = { dbConnection, findUserByEmail };
```

---

## Example 2 — real world backend use case

```js
// File: src/repositories/orderRepository.js
// A repository function for an e-commerce API — search orders by status and date range.
// Demonstrates multiple parameters, dynamic filters, and safe pagination.

const { dbConnection } = require('../db/pool');

async function searchOrders({ userId, status, fromDate, toDate, page = 1, pageSize = 20 }) {
  // Build the WHERE clause and values array together, index by index
  const conditions = ['user_id = $1']; // always filter by the requesting user
  const values = [userId];

  if (status) {
    values.push(status);
    conditions.push(`status = $${values.length}`); // placeholder index matches array position
  }

  if (fromDate) {
    values.push(fromDate);
    conditions.push(`created_at >= $${values.length}`);
  }

  if (toDate) {
    values.push(toDate);
    conditions.push(`created_at <= $${values.length}`);
  }

  const offset = (page - 1) * pageSize; // compute pagination offset in JS, not SQL string math
  values.push(pageSize);
  const limitIndex = values.length;      // capture placeholder index BEFORE pushing offset
  values.push(offset);
  const offsetIndex = values.length;

  const query = `
    SELECT id, status, total_amount, created_at
    FROM orders
    WHERE ${conditions.join(' AND ')}
    ORDER BY created_at DESC
    LIMIT $${limitIndex} OFFSET $${offsetIndex}
  `;

  // Every value the client controls — status, fromDate, toDate, pageSize — travels
  // through the values array, never through string interpolation
  const result = await dbConnection.query(query, values);
  return result.rows;
}

module.exports = { searchOrders };

// Usage in an Express route:
// router.get('/orders', requireAuth, async (req, res) => {
//   const orders = await searchOrders({ userId: req.user.id, ...req.query });
//   res.json(orders);
// });
```

---

## Common mistakes

### Mistake 1 — String-concatenating or template-literal-building SQL

```js
// ❌ WRONG — template literal embeds raw user input directly into SQL text
async function loginUnsafe(email, password) {
  const query = `SELECT * FROM users WHERE email = '${email}' AND password = '${password}'`;
  // Input: email = "' OR '1'='1' -- " logs in as the FIRST user in the table, no password needed
  return dbConnection.query(query);
}

// ✅ CORRECT — placeholders keep SQL structure and data completely separate
async function loginSafe(email, password) {
  const query = 'SELECT * FROM users WHERE email = $1 AND password_hash = $2';
  return dbConnection.query(query, [email, password]); // driver escapes/types values internally
}
```

### Mistake 2 — Trusting ORM "raw" escape hatches with interpolated input

```js
// ❌ WRONG — Sequelize/Knex/Prisma all have a `.raw()` or `$queryRawUnsafe` escape hatch
// that bypasses all built-in protection the moment you interpolate a variable into it
const knexDb = require('knex')({ client: 'pg' });
async function searchProductsUnsafe(searchTerm) {
  return knexDb.raw(`SELECT * FROM products WHERE name LIKE '%${searchTerm}%'`);
  // searchTerm = "%'; DROP TABLE products; --" destroys the table
}

// ✅ CORRECT — pass bindings as a second argument, never interpolate into the raw string
async function searchProductsSafe(searchTerm) {
  return knexDb.raw('SELECT * FROM products WHERE name LIKE ?', [`%${searchTerm}%`]);
  // ? is a placeholder bound safely by the driver, exactly like $1 in pg
}

// ✅ EVEN BETTER — avoid .raw() entirely, use the query builder API
async function searchProductsBuilder(searchTerm) {
  return knexDb('products').where('name', 'like', `%${searchTerm}%`);
}
```

### Mistake 3 — Interpolating identifiers (table/column names) that came from user input

```js
// ❌ WRONG — placeholders only protect VALUES, not table or column names,
// so "safe-looking" parameterized code is still vulnerable if sortColumn is user-controlled
async function listUsersUnsafe(sortColumn) {
  // sortColumn = "id; DROP TABLE users; --" — placeholders CANNOT protect identifiers
  const query = `SELECT * FROM users ORDER BY ${sortColumn}`;
  return dbConnection.query(query);
}

// ✅ CORRECT — validate identifiers against a strict allowlist, never pass them as placeholders
const ALLOWED_SORT_COLUMNS = ['id', 'email', 'created_at']; // whitelist of real column names

async function listUsersSafe(sortColumn) {
  const column = ALLOWED_SORT_COLUMNS.includes(sortColumn) ? sortColumn : 'id'; // reject unknown input
  const query = `SELECT * FROM users ORDER BY ${column}`; // column is now provably safe
  return dbConnection.query(query);
}
```

---

## Practice exercises

### Exercise 1 — easy

Write a function `findProductById(dbConnection, productId)` that:
1. Queries a `products` table for a single row matching an `id` column
2. Uses a parameterized query (`$1` placeholder) — never string concatenation or template literals
3. Returns the matched row, or `null` if no product was found

Manually test it by calling the function with a normal numeric id, and then with the malicious string `"1 OR 1=1"` — confirm the second call returns `null` instead of leaking every row.

```js
// Write your code here
```

---

### Exercise 2 — medium

Write a function `updateUserProfile(dbConnection, userId, updates)` where `updates` is an object that may contain any of `{ name, bio, avatarUrl }` (only the keys present should be updated — this is a partial update, like a PATCH endpoint). Requirements:
1. Dynamically build the `SET` clause based on which keys are present in `updates`
2. Every value (including `userId` for the `WHERE` clause) must go through the parameterized values array — no interpolation of values into the SQL string
3. If `updates` is empty, throw an error instead of running a no-op query
4. Return the updated row

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a small `QueryFilterBuilder` class for a `GET /api/orders` search endpoint that accepts arbitrary query-string filters from the client (e.g. `?status=shipped&minTotal=50&sort=total_amount&dir=desc`). Requirements:
1. Constructor takes a base table name and an allowlist of filterable column names (e.g. `['status', 'minTotal', 'maxTotal']`) and a separate allowlist of sortable columns (e.g. `['created_at', 'total_amount']`)
2. A method `addFilter(column, operator, value)` that only accepts columns present in the filter allowlist (silently ignore or throw on anything else) and always appends the value to an internal values array with the correct placeholder index — never interpolates the value
3. A method `setSort(column, direction)` that only accepts columns from the sort allowlist and only accepts `'asc'`/`'desc'` for direction (reject or default to `'asc'` on anything else) — sort column and direction must be validated against an allowlist since placeholders cannot protect identifiers or keywords
4. A method `build()` that returns `{ text, values }` ready to pass straight into `dbConnection.query(text, values)`
5. Demonstrate an attempted attack: call `addFilter` or `setSort` with a malicious string (e.g. `"created_at; DROP TABLE orders; --"`) and show it gets rejected instead of reaching the database

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
THE RULE
  User input is DATA, never SQL TEXT. Never build a query string by
  concatenating or interpolating a variable that came from the client.

SAFE PATTERNS
  pg (Postgres)   : dbConnection.query('... WHERE id = $1', [userId])
  mysql2          : dbConnection.query('... WHERE id = ?', [userId])
  Knex            : knexDb('users').where({ id: userId })
  Prisma          : prisma.user.findUnique({ where: { id: userId } })
  Sequelize       : User.findOne({ where: { id: userId } })

UNSAFE PATTERNS (never do these)
  `SELECT * FROM users WHERE id = ${userId}`        → template literal injection
  "SELECT * FROM users WHERE id = " + userId        → string concat injection
  knexDb.raw(`... ${searchTerm} ...`)                → raw() with interpolation
  prisma.$queryRawUnsafe(`... ${input} ...`)         → unsafe raw method + interpolation

WHAT PLACEHOLDERS DO / DON'T PROTECT
  Protect   : values — strings, numbers, dates compared or inserted
  DO NOT protect : identifiers — table names, column names, ORDER BY direction,
                   SQL keywords (ASC/DESC) — these need an ALLOWLIST, not a placeholder

DEFENSE LAYERS (use more than one)
  1. Parameterized queries / prepared statements — always, no exceptions
  2. ORM / query builder — adds a safety net + readability
  3. Allowlist validation for identifiers and dynamic column/sort names
  4. Least-privilege DB user — app's DB account should not have DROP/ALTER rights
  5. Input validation (Topic 75) — reject malformed input before it reaches the query

QUICK SELF-CHECK
  If you can find a backtick, +, or template literal ${} touching a SQL
  string with a variable inside it → stop, refactor to a placeholder.
```

---

## Connected topics

- **75 — Input validation and sanitization** — validating and rejecting malformed input before it ever reaches a query is a complementary layer of defense, not a replacement for parameterized queries
- **90 — Connecting to PostgreSQL** — the `pg` driver and connection pool used throughout this doc's examples; covers pool setup and query execution in depth
- **96 — ORM — Prisma** — shows how a full ORM builds parameterized queries automatically and where its `$queryRawUnsafe` escape hatch can reintroduce this exact vulnerability
