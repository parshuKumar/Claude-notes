# 96 — ORM — Prisma

## What is this?

Prisma is a next-generation ORM (Object-Relational Mapper) for Node.js and TypeScript. Instead of writing raw SQL, you describe your database tables in a single `schema.prisma` file, and Prisma generates a fully type-safe client that lets you query your database using plain JavaScript method calls like `prisma.user.findMany()`. Think of it as a translator that sits between your code and your database — you speak JavaScript, Prisma speaks SQL, and it never lets the two get out of sync.

## Why does it matter for backend development?

Every backend app eventually needs to read and write structured data, and writing raw SQL strings by hand is slow, error-prone, and offers no autocomplete or type checking. Prisma solves three real problems backend developers hit daily: it keeps your database schema and your code's data shapes in sync (via migrations), it prevents SQL injection by generating parameterized queries automatically, and it gives you autocomplete for every table, column, and relation straight from your editor. Companies like startups building REST/GraphQL APIs on PostgreSQL or MySQL reach for Prisma specifically because it turns "did I spell the column name right?" bugs into compile-time errors instead of production incidents.

---

## Syntax / API

```prisma
// File: prisma/schema.prisma — the single source of truth for your database shape

// Tells Prisma which database engine to talk to and where the connection string lives
datasource db {
  provider = "postgresql"                 // could also be "mysql", "sqlite", "mongodb"
  url      = env("DATABASE_URL")          // reads connection string from .env — never hardcode it
}

// Tells Prisma to generate a JS/TS client library based on the models below
generator client {
  provider = "prisma-client-js"           // generates node_modules/@prisma/client
}

// A "model" maps to one database table
model User {
  id        Int      @id @default(autoincrement()) // primary key, auto-incrementing integer
  email     String   @unique                        // unique constraint — no two rows share this
  name      String?                                 // "?" means this column is nullable
  createdAt DateTime @default(now())                 // defaults to current timestamp on insert
  posts     Post[]                                   // one-to-many relation — a User has many Posts
}

model Post {
  id       Int    @id @default(autoincrement()) // primary key
  title    String                               // required column
  authorId Int                                  // foreign key column, points to User.id
  author   User   @relation(fields: [authorId], references: [id]) // defines the relation itself
}
```

```bash
# ── Core CLI commands every Prisma project uses ─────────────────────────────

npx prisma init                 # scaffolds prisma/schema.prisma and a .env file
npx prisma generate              # reads schema.prisma, generates the type-safe client into node_modules
npx prisma migrate dev --name init  # creates a SQL migration file AND applies it to the dev database
npx prisma studio                # opens a visual browser-based GUI to view/edit your data
```

```js
// File: src/db/prismaClient.js
// Import the generated client and create ONE shared instance for the whole app

const { PrismaClient } = require('@prisma/client'); // the auto-generated, type-safe client

const dbConnection = new PrismaClient();             // opens (lazily) a connection pool to the DB

module.exports = dbConnection;                       // reused everywhere — never create a new one per request
```

---

## How it works — line by line

The `schema.prisma` file is not code that runs — it is a description of your database, written in Prisma's own schema language. The `datasource` block tells Prisma which database engine and connection string to use. Each `model` block describes one table: each field becomes one column, and its type (`Int`, `String`, `DateTime`, `Boolean`) becomes the column's SQL type. Attributes starting with `@` (like `@id`, `@unique`, `@default`) add constraints — a primary key, a uniqueness rule, or a default value.

When you run `npx prisma generate`, Prisma reads this schema and writes JavaScript/TypeScript code into `node_modules/@prisma/client` — a custom client shaped exactly like your models, so `prisma.user.findMany()` exists only because you have a `User` model. This is why the client feels "magical": it is regenerated every time your schema changes, so if you rename a column, your code editor immediately shows you every place that needs updating.

`npx prisma migrate dev` does two things at once: it compares your schema file to the actual database, writes a plain `.sql` file describing the difference (add column, create table, etc.), and then runs that SQL against your dev database. This file gets committed to git, so every teammate and every environment (staging, production) applies the exact same schema changes in the exact same order.

---

## Example 1 — basic

```js
// File: src/db/prismaClient.js
const { PrismaClient } = require('@prisma/client'); // load the generated client

const dbConnection = new PrismaClient();             // create the single shared client instance

module.exports = dbConnection;

// ────────────────────────────────────────────────────────────────────────────

// File: src/scripts/basicCrud.js
const dbConnection = require('../db/prismaClient');  // reuse the shared connection

async function runBasicCrud() {
  // CREATE — insert one new row into the "User" table
  const newUser = await dbConnection.user.create({
    data: {
      email: 'ada.lovelace@example.com',   // must be unique — will throw if it already exists
      name: 'Ada Lovelace',                 // optional field
    },
  });
  console.log('Created:', newUser);         // Prisma returns the full inserted row, including the new id

  // READ — find one row by a unique field
  const foundUser = await dbConnection.user.findUnique({
    where: { email: 'ada.lovelace@example.com' }, // must query by a @unique or @id field
  });
  console.log('Found:', foundUser);

  // UPDATE — change one or more fields on an existing row
  const updatedUser = await dbConnection.user.update({
    where: { id: newUser.id },              // identify which row to update
    data: { name: 'Ada Byron' },             // only the fields listed here get changed
  });
  console.log('Updated:', updatedUser);

  // DELETE — remove the row entirely
  const deletedUser = await dbConnection.user.delete({
    where: { id: newUser.id },              // identify which row to delete
  });
  console.log('Deleted:', deletedUser);     // returns the row as it was right before deletion

  await dbConnection.$disconnect();         // close the connection pool when the script is done
}

runBasicCrud().catch((err) => {
  console.error('CRUD script failed:', err); // always catch — Prisma throws on constraint violations
  process.exit(1);
});
```

---

## Example 2 — real world backend use case

```js
// File: src/services/userService.js
// A realistic service layer used by an Express controller — handles users and their posts,
// including relations, filtering, and pagination. This is the pattern used in production APIs.

const dbConnection = require('../db/prismaClient');

// Create a user AND their first post in a single request — using a nested write
async function registerUserWithFirstPost(requestBody) {
  const { email, name, postTitle } = requestBody;   // destructure incoming request data

  const newUser = await dbConnection.user.create({
    data: {
      email,                                        // shorthand — same as email: email
      name,
      posts: {
        create: [{ title: postTitle }],              // nested create — inserts into Post AND links authorId
      },
    },
    include: { posts: true },                        // tells Prisma to return the related posts too
  });

  return newUser;
}

// Fetch a paginated list of users, including a count of how many posts each has
async function listUsersPaginated(pageNumber = 1, pageSize = 10) {
  const skipCount = (pageNumber - 1) * pageSize;      // calculate offset for pagination

  const users = await dbConnection.user.findMany({
    skip: skipCount,                                  // how many rows to skip
    take: pageSize,                                   // how many rows to return
    orderBy: { createdAt: 'desc' },                    // newest users first
    include: {
      _count: { select: { posts: true } },             // adds a postsCount-like field via relation count
    },
  });

  const totalUsers = await dbConnection.user.count();  // total rows, for building pagination metadata

  return { users, totalUsers, pageNumber, pageSize };
}

// Search users by partial email match — a common "search bar" backend query
async function searchUsersByEmail(searchTerm) {
  return dbConnection.user.findMany({
    where: {
      email: { contains: searchTerm, mode: 'insensitive' }, // SQL LIKE '%term%', case-insensitive
    },
  });
}

// Update a post only if it belongs to the requesting user — ownership check pattern
async function updateOwnPost(userId, postId, newTitle) {
  const post = await dbConnection.post.findUnique({ where: { id: postId } });

  if (!post || post.authorId !== userId) {
    throw new Error('Not authorized to edit this post'); // guard clause — never trust the client
  }

  return dbConnection.post.update({
    where: { id: postId },
    data: { title: newTitle },
  });
}

module.exports = {
  registerUserWithFirstPost,
  listUsersPaginated,
  searchUsersByEmail,
  updateOwnPost,
};

// ────────────────────────────────────────────────────────────────────────────
// File: src/routes/userRoutes.js — how a controller wires into this service
//
// const express = require('express');
// const router = express.Router();
// const userService = require('../services/userService');
//
// router.post('/users', async (req, res) => {
//   try {
//     const user = await userService.registerUserWithFirstPost(req.body);
//     res.status(201).json(user);
//   } catch (err) {
//     res.status(400).json({ error: err.message });
//   }
// });
//
// module.exports = router;
```

---

## Common mistakes

### Mistake 1 — Creating a new PrismaClient per request

```js
// ❌ WRONG — creates a fresh connection pool on every single HTTP request
app.get('/users', async (req, res) => {
  const { PrismaClient } = require('@prisma/client'); // new client every call
  const dbConnection = new PrismaClient();             // opens new connections — exhausts DB limits fast
  const users = await dbConnection.user.findMany();
  res.json(users);
});

// ✅ CORRECT — create ONE client at app startup and reuse it everywhere
// File: src/db/prismaClient.js
const { PrismaClient } = require('@prisma/client');
const dbConnection = new PrismaClient();               // single shared instance
module.exports = dbConnection;

// File: src/routes/userRoutes.js
const dbConnection = require('../db/prismaClient');    // import the shared instance
app.get('/users', async (req, res) => {
  const users = await dbConnection.user.findMany();     // reuses the existing pool
  res.json(users);
});
```

### Mistake 2 — Editing schema.prisma without regenerating or migrating

```js
// ❌ WRONG — you add a field to schema.prisma but forget to run prisma generate,
// so the old client (without the new field) is still what's loaded in node_modules
// model User { id Int @id  email String  phoneNumber String }  ← added phoneNumber
//
// const user = await dbConnection.user.create({
//   data: { email: 'test@example.com', phoneNumber: '555-0100' }, // TypeError: unknown field
// });

// ✅ CORRECT — after ANY schema.prisma edit, regenerate the client AND migrate the database
// $ npx prisma migrate dev --name add_phone_number   // creates + applies the SQL migration
// $ npx prisma generate                               // regenerates the client (migrate dev does this too)

const dbConnection = require('../db/prismaClient');
const user = await dbConnection.user.create({
  data: { email: 'test@example.com', phoneNumber: '555-0100' }, // now works — client matches schema
});
```

### Mistake 3 — Forgetting relations require `include` to load

```js
// ❌ WRONG — assumes .posts is automatically attached to every user query
const user = await dbConnection.user.findUnique({ where: { id: userId } });
console.log(user.posts.length);   // TypeError: Cannot read properties of undefined
// Prisma only loads exactly the fields/relations you ask for — no hidden joins

// ✅ CORRECT — explicitly include the relation you need
const user = await dbConnection.user.findUnique({
  where: { id: userId },
  include: { posts: true },        // tells Prisma to JOIN and attach the posts array
});
console.log(user.posts.length);    // works — posts is now a real array on the returned object
```

---

## Practice exercises

### Exercise 1 — easy

Set up a fresh Prisma project against a local SQLite database (no external DB server needed):
1. Run `npx prisma init --datasource-provider sqlite`
2. Define a `Product` model in `schema.prisma` with fields: `id` (autoincrement int, primary key), `name` (String), `price` (Float), `inStock` (Boolean, default `true`)
3. Run the migration to create the table
4. Write a script that creates three products, then fetches and logs all products ordered by `price` ascending

```js
// Write your code here
```

---

### Exercise 2 — medium

Extend the schema with a `Category` model that has a one-to-many relationship with `Product` (one category has many products; each product belongs to one category). Write a function `createProductInCategory(categoryName, productName, price)` that:
1. Uses `upsert` to find a category by name or create it if it doesn't exist
2. Creates a new product linked to that category's id
3. Returns the created product with its category included

Then write `listCategoriesWithProductCounts()` that returns every category along with how many products belong to it (hint: `_count`).

```js
// Write your code here
```

---

### Exercise 3 — hard

Build an `OrderService` module backed by three related Prisma models: `Order`, `OrderItem`, and `Product` (an Order has many OrderItems, each OrderItem references one Product and a quantity). Implement:
1. `placeOrder(userId, items)` where `items` is an array of `{ productId, quantity }` — must run inside a Prisma transaction (`dbConnection.$transaction`) so that either ALL order items are created or NONE are, and must throw if any product has insufficient stock (check a `stockQuantity` field on `Product` and decrement it as part of the same transaction)
2. `getOrderWithTotal(orderId)` that fetches an order with its items and products included, and calculates the total price server-side (never trust a client-sent total)
3. `cancelOrder(orderId)` that restores each product's `stockQuantity` and marks the order as `CANCELLED`, also inside a transaction

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
CORE FILES
  prisma/schema.prisma        → describes datasource, generator, and all models
  .env                        → holds DATABASE_URL, never committed to git
  node_modules/@prisma/client → auto-generated, type-safe client (do not edit)

CLI COMMANDS
  npx prisma init                        → scaffold schema.prisma + .env
  npx prisma generate                     → regenerate client after schema changes
  npx prisma migrate dev --name <name>    → create + apply migration (dev)
  npx prisma migrate deploy               → apply pending migrations (production)
  npx prisma studio                       → visual GUI for browsing/editing data
  npx prisma db push                      → sync schema to DB WITHOUT a migration file (prototyping only)

SCHEMA FIELD ATTRIBUTES
  @id                  → marks primary key
  @default(autoincrement()) → auto-incrementing integer
  @default(now())      → defaults to current timestamp
  @unique              → enforces uniqueness constraint
  ?                     → makes the field nullable (e.g. String?)
  []                    → marks a one-to-many relation array (e.g. Post[])

CLIENT CRUD METHODS
  model.create({ data })                  → INSERT
  model.findUnique({ where })              → SELECT by unique/id field
  model.findFirst({ where })               → SELECT first match, any field
  model.findMany({ where, skip, take })    → SELECT multiple, with pagination
  model.update({ where, data })            → UPDATE
  model.delete({ where })                  → DELETE
  model.upsert({ where, create, update })  → UPDATE if exists, else CREATE
  model.count({ where })                   → COUNT rows

RELATIONS
  include: { relationName: true }   → eager-load a related table (JOIN)
  data: { relationName: { create: [...] } } → nested write, creates related rows too

TRANSACTIONS
  dbConnection.$transaction([...])      → run multiple queries atomically (array form)
  dbConnection.$transaction(async (tx) => { ... }) → interactive transaction, use `tx` instead of client

GOTCHAS
  - ALWAYS reuse a single PrismaClient instance — never instantiate per request
  - Edit schema.prisma → must run migrate dev (or generate) or the client is stale
  - Relations are NEVER auto-loaded — must explicitly `include` them
  - `db push` skips migration history — use only for prototyping, never production
```

---

## Connected topics

- **90 — Connecting to PostgreSQL** — Prisma's `datasource` block sits on top of a real PostgreSQL connection; understanding the raw driver first makes Prisma's abstractions click
- **94 — Database migrations** — `prisma migrate dev` is Prisma's own migration system, following the same up/down migration philosophy taught in that topic
- **97 — Transactions** — `dbConnection.$transaction()` wraps ACID transaction semantics, directly applying the BEGIN/COMMIT/ROLLBACK concepts from that topic
