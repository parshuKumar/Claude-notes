# 98 — Database seeding and fixtures

## What is this?

**Seeding** is the process of inserting a known set of data into a database so your app has something to work with before real users show up. **Fixtures** are the actual data records used for that — either fixed sample data (a "demo admin" account) or realistically generated fake data (thousands of fake users). Think of it like stocking a brand-new store with sample products before opening day — customers (and your tests) need something on the shelves to interact with, not an empty warehouse.

## Why does it matter for backend development?

A fresh database is completely empty — no users, no products, no orders. Without seed data, you cannot manually test a login form, demo a dashboard to a client, or run automated tests that check "does the orders list show 3 items?" Backend developers write seed scripts to populate local/dev databases instantly, and use factories with libraries like `@faker-js/faker` to generate hundreds of realistic-looking records (names, emails, addresses) for load testing and QA — instead of typing fake data by hand every time the database is wiped.

---

## Syntax / API

```js
// Install faker as a dev dependency (never needed in production):
// npm install --save-dev @faker-js/faker

const { faker } = require('@faker-js/faker');   // import the faker library

// ── A "factory" is a function that builds ONE fake record on demand ────────
function createUserFixture(overrides = {}) {
  return {
    id: faker.string.uuid(),                 // random UUID, works as a primary key
    fullName: faker.person.fullName(),        // realistic fake name, e.g. "Meera Rao"
    email: faker.internet.email(),            // realistic fake email address
    passwordHash: faker.internet.password(),  // stand-in for a real bcrypt hash
    createdAt: faker.date.past(),              // random date within the past year
    ...overrides,                              // caller can override any field
  };
}

// ── Building many fixtures at once ──────────────────────────────────────────
const seedUsers = Array.from({ length: 5 }, () => createUserFixture());
// Array.from({ length: 5 }, fn) calls fn() 5 times and collects the results

console.log(seedUsers[0]);
// → { id: 'a1b2...', fullName: 'Meera Rao', email: 'meera.rao23@example.com', ... }

// ── Overriding a field when a test needs a specific value ──────────────────
const adminUser = createUserFixture({ email: 'admin@example.com' });
// Every field is random EXCEPT email, which is pinned for a login test
```

---

## How it works — line by line

`require('@faker-js/faker')` pulls in the faker library and destructures the `faker` object, which exposes dozens of namespaced generators (`faker.person`, `faker.internet`, `faker.date`, `faker.commerce`, etc.) — each call returns a different random-but-realistic value every time it runs.

`createUserFixture(overrides = {})` is a **factory function**: it does not store data anywhere, it just *builds and returns* one plain JavaScript object shaped like a database row. The `overrides = {}` default parameter means calling it with no arguments still works.

Inside the returned object, `...overrides` is spread **last**, so any key the caller passes in overwrites the randomly generated one above it — this is what lets `createUserFixture({ email: 'admin@example.com' })` keep every field random except the one you pinned.

`Array.from({ length: 5 }, () => createUserFixture())` is the standard way to call a factory *N* times: the first argument is an array-like with a `length`, and the second argument is a map function invoked once per index — since the factory generates new random data every call, none of the 5 users are identical.

A **seed script** simply takes fixtures like these and inserts them into real database tables (via `pg`, `mongoose`, `knex`, or Prisma) so the database is populated the moment the app starts.

---

## Example 1 — basic

```js
// File: scripts/seed-users.js
// A standalone script: run with `node scripts/seed-users.js`

const { Pool } = require('pg');                 // PostgreSQL driver (Topic 90)
const { faker } = require('@faker-js/faker');    // fake data generator

// Connection pool reads credentials from environment variables
const dbConnection = new Pool({
  connectionString: process.env.DATABASE_URL,   // e.g. postgres://user:pass@localhost/app_dev
});

// Factory: builds one fake user row
function createUserFixture() {
  return {
    fullName: faker.person.fullName(),           // e.g. "Arjun Mehta"
    email: faker.internet.email(),               // e.g. "arjun.mehta5@example.com"
    passwordHash: 'seed-only-not-a-real-hash',   // never store plaintext, even in seeds
  };
}

async function seedUsers() {
  const usersToInsert = Array.from({ length: 10 }, createUserFixture); // build 10 fake users

  for (const user of usersToInsert) {
    // Insert one row at a time — fine for small seed sets
    await dbConnection.query(
      'INSERT INTO users (full_name, email, password_hash) VALUES ($1, $2, $3)',
      [user.fullName, user.email, user.passwordHash]   // parameterized — prevents SQL injection
    );
  }

  console.log(`Seeded ${usersToInsert.length} users.`);  // confirm how many rows were added
  await dbConnection.end();                               // close the pool so the script can exit
}

seedUsers().catch((err) => {
  console.error('Seeding failed:', err.message);   // log the failure reason
  process.exit(1);                                  // exit with a non-zero code (Topic 05)
});
```

---

## Example 2 — real world backend use case

```js
// File: prisma/seed.js  (or scripts/seed.js for a raw pg setup)
// Seeds users AND their related posts — a realistic relational seed script
// with a safety guard, a reset step, and a transaction for consistency.

const { Pool } = require('pg');
const { faker } = require('@faker-js/faker');

const dbConnection = new Pool({ connectionString: process.env.DATABASE_URL });

// ── Factories ────────────────────────────────────────────────────────────
function createUserFixture() {
  return {
    fullName: faker.person.fullName(),
    email: faker.internet.email(),
    passwordHash: 'seed-only-not-a-real-hash',
  };
}

function createPostFixture(userId) {
  return {
    userId,                                        // foreign key — links back to a real user
    title: faker.lorem.sentence(),                 // fake blog-post title
    body: faker.lorem.paragraphs(3),                // fake multi-paragraph body text
    publishedAt: faker.date.recent({ days: 30 }),   // published sometime in the last 30 days
  };
}

async function seedDatabase() {
  // Guard rail — NEVER let a seed script run against production data
  if (process.env.NODE_ENV === 'production') {
    throw new Error('Refusing to seed: NODE_ENV is production.');
  }

  const client = await dbConnection.connect();   // grab a dedicated client for a transaction

  try {
    await client.query('BEGIN');                 // start transaction (Topic 97)

    // Reset — clear old seed data so re-running this script is repeatable
    await client.query('TRUNCATE TABLE posts, users RESTART IDENTITY CASCADE');
    // TRUNCATE ... CASCADE also empties dependent tables (posts) safely

    // Insert parent rows (users) first — posts need a real userId to reference
    const seedUsers = Array.from({ length: 5 }, createUserFixture);
    const insertedUserIds = [];

    for (const user of seedUsers) {
      const result = await client.query(
        `INSERT INTO users (full_name, email, password_hash)
         VALUES ($1, $2, $3) RETURNING id`,       // RETURNING gives back the new primary key
        [user.fullName, user.email, user.passwordHash]
      );
      insertedUserIds.push(result.rows[0].id);    // remember the id for the posts below
    }

    // Insert child rows (posts) — each post belongs to a real, already-inserted user
    for (const userId of insertedUserIds) {
      const postCount = faker.number.int({ min: 1, max: 3 });  // 1–3 posts per user
      const posts = Array.from({ length: postCount }, () => createPostFixture(userId));

      for (const post of posts) {
        await client.query(
          `INSERT INTO posts (user_id, title, body, published_at)
           VALUES ($1, $2, $3, $4)`,
          [post.userId, post.title, post.body, post.publishedAt]
        );
      }
    }

    await client.query('COMMIT');                 // save everything atomically
    console.log(`Seeded ${insertedUserIds.length} users with related posts.`);
  } catch (err) {
    await client.query('ROLLBACK');               // undo everything on any failure
    console.error('Seeding failed, rolled back:', err.message);
    throw err;
  } finally {
    client.release();                              // return the client to the pool
  }
}

seedDatabase()
  .then(() => dbConnection.end())
  .catch(() => {
    dbConnection.end();
    process.exit(1);
  });

// Usage: add to package.json → "scripts": { "seed": "node prisma/seed.js" }
// Run with: npm run seed
```

---

## Common mistakes

### Mistake 1 — No environment guard, seeding runs against production

```js
// ❌ WRONG — this script has no idea what environment it is running in
async function seedDatabase() {
  await dbConnection.query('TRUNCATE TABLE users CASCADE');  // could wipe REAL production users
  // ... insert fake data ...
}
// If someone runs `npm run seed` with a production DATABASE_URL, real data is gone

// ✅ CORRECT — refuse to run unless explicitly in a safe environment
async function seedDatabase() {
  if (process.env.NODE_ENV === 'production') {
    throw new Error('Refusing to seed: NODE_ENV is production.');
  }
  await dbConnection.query('TRUNCATE TABLE users CASCADE');
  // ... insert fake data ...
}
```

### Mistake 2 — Seed script is not idempotent, re-running it duplicates data

```js
// ❌ WRONG — running this twice inserts 10 more users every time,
// and eventually hits a unique constraint error on email
async function seedUsers() {
  const users = Array.from({ length: 10 }, createUserFixture);
  for (const user of users) {
    await dbConnection.query(
      'INSERT INTO users (full_name, email) VALUES ($1, $2)',
      [user.fullName, user.email]
    );
  }
}

// ✅ CORRECT — reset the table first, so the script is safe to run any number of times
async function seedUsers() {
  await dbConnection.query('TRUNCATE TABLE users RESTART IDENTITY CASCADE');
  // RESTART IDENTITY resets auto-increment ids back to 1 as well

  const users = Array.from({ length: 10 }, createUserFixture);
  for (const user of users) {
    await dbConnection.query(
      'INSERT INTO users (full_name, email) VALUES ($1, $2)',
      [user.fullName, user.email]
    );
  }
}
```

### Mistake 3 — Inserting child rows before their parent rows exist

```js
// ❌ WRONG — posts reference userId values that don't exist yet
// PostgreSQL throws: "insert or update on table posts violates foreign key constraint"
async function seedDatabase() {
  const fakeUserId = faker.string.uuid();          // random id, NOT actually in the users table
  await dbConnection.query(
    'INSERT INTO posts (user_id, title) VALUES ($1, $2)',
    [fakeUserId, faker.lorem.sentence()]
  );
}

// ✅ CORRECT — insert the parent first, capture the REAL generated id, then insert the child
async function seedDatabase() {
  const result = await dbConnection.query(
    `INSERT INTO users (full_name, email) VALUES ($1, $2) RETURNING id`,
    [faker.person.fullName(), faker.internet.email()]
  );
  const realUserId = result.rows[0].id;             // the id PostgreSQL actually assigned

  await dbConnection.query(
    'INSERT INTO posts (user_id, title) VALUES ($1, $2)',
    [realUserId, faker.lorem.sentence()]              // guaranteed to satisfy the foreign key
  );
}
```

---

## Practice exercises

### Exercise 1 — easy

Using `@faker-js/faker`, write a factory function `createProductFixture(overrides)` that returns an object shaped like a product row:
1. `sku` — a random alphanumeric code (hint: `faker.string.alphanumeric(8)`)
2. `name` — a realistic product name (hint: `faker.commerce.productName()`)
3. `price` — a realistic price as a number (hint: `faker.commerce.price()`)
4. `inStock` — a random boolean

The function should accept an `overrides` object and merge it in, just like `createUserFixture` in the lesson. Generate an array of 10 products with the factory and log them to the console.

```js
// Write your code here
```

---

### Exercise 2 — medium

Write a seed script `scripts/seed-authors-books.js` that seeds two related tables, `authors` and `books` (`books.author_id` references `authors.id`):
1. Guard the script so it refuses to run when `process.env.NODE_ENV === 'production'`
2. Truncate both tables first (in the correct order so foreign keys don't break) so the script is safe to re-run
3. Use a factory to insert 3 fake authors, capturing each real generated `id` with `RETURNING id`
4. Use a second factory to insert 2–4 fake books per author, each referencing the correct `author_id`
5. Wrap the whole thing in a transaction (`BEGIN` / `COMMIT` / `ROLLBACK` on error)
6. Log a summary: how many authors and how many books were inserted

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a small **seed runner system** that tracks which seeds have already run, so re-deploying doesn't re-seed data that's already there:
1. A `seed_history` table (or an in-memory `Set` if you don't have a real DB handy) that stores the names of seeds that already ran
2. A `SeedRunner` class with:
   - `register(name, seedFn)` — registers a named seed function
   - `async run(name)` — runs one seed by name, but only if it is not already in `seed_history`; after it succeeds, records the name in `seed_history`
   - `async runAll()` — runs every registered seed, in the order registered, skipping ones already run
   - `async reset()` — clears `seed_history` and truncates every table the registered seeds touch, so everything can be re-seeded from scratch
3. Register at least two seeds: `'seed:users'` (uses a faker factory to insert 5 users) and `'seed:posts'` (depends on users existing, uses a factory to insert posts per user)
4. Demonstrate that calling `runAll()` twice in a row only inserts data once, and that `reset()` followed by `runAll()` re-seeds everything

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
WHAT
  Seeding   → inserting known/sample data into a database
  Fixture   → one unit of that sample data (a row/object)
  Factory   → a function that builds one fixture on demand, with random or fixed fields

INSTALL
  npm install --save-dev @faker-js/faker

COMMON FAKER GENERATORS
  faker.person.fullName()            → "Arjun Mehta"
  faker.internet.email()             → "arjun23@example.com"
  faker.internet.password()          → random fake password string
  faker.string.uuid()                → random UUID v4
  faker.string.alphanumeric(8)       → random 8-char code (e.g. SKU)
  faker.number.int({ min, max })     → random integer in range
  faker.date.past() / .recent()      → random past date
  faker.lorem.sentence() / .paragraphs(n) → fake text
  faker.commerce.productName()       → fake product name
  faker.commerce.price()             → fake price string

FACTORY PATTERN
  function createXFixture(overrides = {}) {
    return { ...randomFields, ...overrides };   // overrides always win — spread last
  }
  Array.from({ length: N }, () => createXFixture());  // build N fixtures at once

SEED SCRIPT CHECKLIST
  1. Guard against NODE_ENV === 'production'
  2. TRUNCATE (or delete) existing rows first — makes the script idempotent
  3. Insert PARENT rows before CHILD rows (respect foreign keys)
  4. Use RETURNING id to capture real generated ids for relations
  5. Wrap multi-table inserts in a transaction (BEGIN / COMMIT / ROLLBACK)
  6. Log a summary of what was inserted

GOTCHAS
  Running seed against prod           → catastrophic data loss, always guard NODE_ENV
  Re-running without reset            → unique constraint violations, duplicate rows
  Child before parent                 → foreign key constraint violation
  Hardcoded fake IDs across tables    → collide with auto-increment/real ids — use RETURNING
  Faker in production dependencies    → keep @faker-js/faker in devDependencies only

RUNNING
  package.json → "scripts": { "seed": "node scripts/seed.js" }
  npm run seed
```

---

## Connected topics

- **97 — Transactions** — seed scripts that touch multiple related tables should be wrapped in `BEGIN`/`COMMIT`/`ROLLBACK` so a failure never leaves half-seeded data.
- **90 — Connecting to PostgreSQL** — seed scripts use the same `pg` `Pool`/client and parameterized queries covered there to actually insert fixture rows.
- **87 — Integration testing with a real DB** — test suites seed a real (or test) database before each run so assertions have predictable, known data to check against.
