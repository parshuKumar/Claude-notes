# 94 — Database migrations

## What is this?

A migration is a small, version-controlled file that describes **one change** to your database schema — creating a table, adding a column, adding an index — along with instructions for how to undo that change. Migration tools (`node-pg-migrate`, `knex`) run these files in order and keep a record in the database of which ones have already been applied. Think of migrations as **git commits for your database structure**: each one is a numbered, timestamped, reviewable diff that any teammate (or any server) can replay to reach the exact same schema.

## Why does it matter for backend development?

Every backend app's schema changes over time — you add a `status` column, create a `refresh_tokens` table, add a unique index. Without migrations, developers run ad-hoc SQL by hand, and dev/staging/production databases drift apart until nobody knows what schema is actually live. Migrations fix this: they run automatically in CI/CD, they're reviewed in pull requests like code, and — critically — they can be **rolled back** if a deploy breaks something. Any backend job that touches PostgreSQL, MySQL, or SQLite expects you to know how to write, run, and reverse migrations.

---

## Syntax / API

```js
// ── node-pg-migrate: a single migration file ────────────────────────────────
// File: migrations/1706300000000_create-users-table.js

// exports.shorthands = undefined; // optional column-type shortcuts, not used here

// up() runs when the migration is APPLIED — describes the forward change
exports.up = (pgm) => {
  // pgm.createTable(tableName, columns) — builds a CREATE TABLE statement
  pgm.createTable('users', {
    id: 'id',                                   // shorthand for SERIAL PRIMARY KEY
    email: { type: 'varchar(255)', notNull: true, unique: true }, // unique email column
    password_hash: { type: 'varchar(255)', notNull: true },       // hashed password, never plain text
    created_at: { type: 'timestamp', notNull: true, default: pgm.func('now()') }, // auto timestamp
  });
};

// down() runs when the migration is ROLLED BACK — must exactly undo up()
exports.down = (pgm) => {
  pgm.dropTable('users'); // reverse of createTable — drops the whole table
};

// ── Running node-pg-migrate from the CLI ────────────────────────────────────
// npx node-pg-migrate create create-users-table   → scaffolds a new timestamped file
// npx node-pg-migrate up                          → applies all pending migrations
// npx node-pg-migrate down                        → rolls back the most recent migration
// npx node-pg-migrate up 2                         → applies only the next 2 migrations

// ── Knex: the same idea, different API shape ────────────────────────────────
// File: migrations/20260127000000_create_users_table.js

// exports.up() — builds the table using knex's schema builder
exports.up = (knex) => {
  return knex.schema.createTable('users', (table) => {
    table.increments('id');                     // auto-incrementing primary key
    table.string('email').notNullable().unique(); // required, unique email
    table.string('password_hash').notNullable(); // required, hashed password
    table.timestamp('created_at').defaultTo(knex.fn.now()); // auto timestamp
  });
};

// exports.down() — the exact reverse of up()
exports.down = (knex) => {
  return knex.schema.dropTable('users'); // drop the table on rollback
};

// ── Knex CLI commands ────────────────────────────────────────────────────────
// npx knex migrate:make create_users_table   → scaffolds a new timestamped file
// npx knex migrate:latest                    → applies all pending migrations
// npx knex migrate:rollback                  → rolls back the last BATCH of migrations
// npx knex migrate:status                    → shows which migrations have run
```

---

## How it works — line by line

A migration tool needs to answer one question every time it runs: **"which migrations have already been applied to this specific database?"** It answers this by creating a bookkeeping table the first time it connects.

- `node-pg-migrate` creates a table called `pgmigrations` — one row per applied migration, storing its file name and the timestamp it ran.
- `knex` creates two tables: `knex_migrations` (which files have run, grouped into numbered **batches**) and `knex_migrations_lock` (prevents two processes from migrating at the same time).

When you run the "up" command, the tool:
1. Reads every migration file in your `migrations/` folder, sorted by the number/timestamp in the file name.
2. Checks the bookkeeping table to see which ones are **not yet recorded**.
3. Runs each pending file's `up()` function, in order, inside a transaction (so a failure halfway through rolls back cleanly).
4. Inserts a row into the bookkeeping table for every migration that succeeded.

When you run "down" / "rollback", it reverses the process: it looks at the most recently applied migration (or batch, in knex), calls that file's `down()` function, and deletes its bookkeeping row — so running `up` again would re-apply it.

This is why migration files are named with a sortable prefix like `1706300000000_` or `20260127000000_` — the tool relies entirely on file name order to know the sequence of changes.

---

## Example 1 — basic

```js
// File: migrations/1706400000000_create-products-table.js
// A basic node-pg-migrate migration — creates one table with a few columns.

exports.up = (pgm) => {
  // Create the "products" table with typed, constrained columns
  pgm.createTable('products', {
    id: 'id',                                              // SERIAL PRIMARY KEY shorthand
    name: { type: 'varchar(200)', notNull: true },         // product name, required
    price_cents: { type: 'integer', notNull: true },       // store money as integer cents, never float
    is_active: { type: 'boolean', notNull: true, default: true }, // soft-delete flag, defaults to active
    created_at: { type: 'timestamp', notNull: true, default: pgm.func('now()') }, // creation timestamp
  });

  // Add an index on is_active — most queries will filter WHERE is_active = true
  pgm.createIndex('products', 'is_active');
};

exports.down = (pgm) => {
  // Reversing an index drop is implicit — dropping the table removes the index too,
  // but being explicit here documents intent and works if you ever split the migration.
  pgm.dropIndex('products', 'is_active');  // undo the index first
  pgm.dropTable('products');               // then undo the table itself
};

// Run it:
//   npx node-pg-migrate up      → creates the products table
//   npx node-pg-migrate down    → drops it again, back to the previous schema state
```

---

## Example 2 — real world backend use case

```js
// File: migrations/20260127103000_add_status_and_index_to_orders.js
// Realistic knex migration: adding a column + backfilling + adding a foreign key
// to an EXISTING "orders" table that already has live production data.

exports.up = async (knex) => {
  // Step 1 — add the new column as NULLABLE first (never notNullable on a table with existing rows,
  // or every existing row violates the constraint and the migration fails)
  await knex.schema.alterTable('orders', (table) => {
    table.string('status').defaultTo('pending');   // new column, safe default for new rows
    table.integer('user_id').unsigned().references('id').inTable('users'); // FK to users table
  });

  // Step 2 — backfill existing rows so the column is meaningful for old data too
  await knex('orders')
    .whereNull('status')                 // only rows that didn't get the default (pre-existing rows)
    .update({ status: 'completed' });    // assume legacy orders were already completed

  // Step 3 — now that every row has a value, tighten the constraint
  await knex.schema.alterTable('orders', (table) => {
    table.string('status').notNullable().alter();  // enforce not-null going forward
  });

  // Step 4 — index the column since the API will filter orders by status constantly
  await knex.schema.alterTable('orders', (table) => {
    table.index('status', 'idx_orders_status');     // named index for query performance
  });
};

exports.down = async (knex) => {
  // Reverse in the OPPOSITE order the changes were applied
  await knex.schema.alterTable('orders', (table) => {
    table.dropIndex('status', 'idx_orders_status'); // remove the index first
    table.dropColumn('user_id');                     // remove the FK column
    table.dropColumn('status');                       // remove the status column last
  });
};

// package.json scripts a real backend project wires up around this:
// "scripts": {
//   "migrate:make":     "knex migrate:make",
//   "migrate:up":       "knex migrate:latest",
//   "migrate:down":     "knex migrate:rollback",
//   "migrate:status":   "knex migrate:status"
// }
//
// Typical CI/CD deploy step:
//   npm run migrate:up   ← runs automatically BEFORE the new app code starts serving traffic
```

---

## Common mistakes

### Mistake 1 — Editing an already-applied migration file

```js
// ❌ WRONG — teammate already ran this migration in staging/production.
// Editing it now means your local DB and staging/prod DBs have DIFFERENT schemas,
// because the bookkeeping table thinks it already ran the "old" version.
// File: migrations/20260110_create_users_table.js
exports.up = (knex) => {
  return knex.schema.createTable('users', (table) => {
    table.increments('id');
    table.string('email').notNullable(); // someone added .unique() here after it shipped — BAD
  });
};

// ✅ CORRECT — create a NEW migration that alters the existing table
// File: migrations/20260127_add_unique_to_users_email.js
exports.up = (knex) => {
  return knex.schema.alterTable('users', (table) => {
    table.string('email').notNullable().unique().alter(); // adds the constraint via a new step
  });
};
exports.down = (knex) => {
  return knex.schema.alterTable('users', (table) => {
    table.string('email').notNullable().alter(); // removes just the unique constraint
  });
};
```

### Mistake 2 — Writing a migration with no working down()

```js
// ❌ WRONG — down() is empty/missing, so this migration can never be rolled back safely.
// If this deploy breaks production, there is no way to undo the schema change.
exports.up = (pgm) => {
  pgm.dropColumn('orders', 'legacy_notes'); // deletes a column and its data permanently
};
exports.down = () => {
  // left empty — "we'll never need to roll this back" (famous last words)
};

// ✅ CORRECT — make destructive changes reversible, or at least fail loudly on rollback
exports.up = (pgm) => {
  pgm.renameColumn('orders', 'legacy_notes', 'legacy_notes_deprecated'); // rename, don't delete yet
};
exports.down = (pgm) => {
  pgm.renameColumn('orders', 'legacy_notes_deprecated', 'legacy_notes'); // fully reversible
};
// Only add a separate DROP COLUMN migration weeks later, once you're certain it's unused.
```

### Mistake 3 — Running manual SQL against production instead of through a migration

```js
// ❌ WRONG — a developer SSHs into prod and runs this by hand to "fix things quickly":
// ALTER TABLE users ADD COLUMN last_login_at TIMESTAMP;
// Now production's schema has a column that exists NOWHERE in the migrations folder.
// The next developer who runs migrations from scratch (new environment, CI, teammate's laptop)
// gets a database that is missing this column — silent schema drift.

// ✅ CORRECT — every schema change, no matter how small or urgent, goes through a migration file
// File: migrations/20260127120000_add_last_login_at_to_users.js
exports.up = (pgm) => {
  pgm.addColumn('users', {
    last_login_at: { type: 'timestamp', notNull: false }, // nullable — old users have no value yet
  });
};
exports.down = (pgm) => {
  pgm.dropColumn('users', 'last_login_at'); // fully reversible, tracked in version control
};
// Commit this file, open a PR, and let CI run `migrate up` against staging before merging.
```

---

## Practice exercises

### Exercise 1 — easy

Using `node-pg-migrate` syntax, write a migration file that creates a `categories` table with:
1. An auto-incrementing `id` primary key
2. A required, unique `name` column (`varchar(100)`)
3. A `created_at` timestamp column that defaults to the current time
4. A working `down()` that fully reverses the `up()`

```js
// Write your code here
```

---

### Exercise 2 — medium

Using `knex` migration syntax, write a migration that alters an existing `products` table (assume it already has `id`, `name`, `price_cents`) to:
1. Add a nullable `category_id` column that references `id` in the `categories` table
2. Add an index on `category_id` since products will frequently be filtered by category
3. Provide a `down()` that removes the index and the column, in the correct order
4. Explain in a comment why `category_id` should be added as **nullable** rather than `notNullable()` on a table that already has rows

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a **complete migration workflow** for a small blog feature, using `knex`:
1. Write one migration that creates an `authors` table (`id`, `name`, `email` unique)
2. Write a second migration (later timestamp) that creates a `posts` table with `id`, `title`, `body`, and a required `author_id` foreign key referencing `authors.id`
3. Both migrations must have correct `down()` functions — and the `down()` order across the two files must not violate the foreign key (i.e. `posts` must be droppable before `authors`, or you must handle the dependency explicitly)
4. Write a short Node script (using the `knex` library directly, not the CLI) that connects to the database, runs `knex.migrate.latest()`, logs which migrations were run, and then calls `knex.migrate.status()` to print how many migrations are pending vs completed
5. Explain in comments what would happen if you tried to roll back the `authors` migration while the `posts` migration was still applied

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
WHAT A MIGRATION IS
  A versioned file describing ONE schema change, with:
    up()   → applies the change (create table, add column, add index...)
    down() → reverses the change EXACTLY (drop table, drop column, drop index...)

BOOKKEEPING TABLES (created automatically)
  node-pg-migrate → "pgmigrations"                (one row per applied file)
  knex            → "knex_migrations" + "knex_migrations_lock" (batches + locking)

NODE-PG-MIGRATE CLI
  npx node-pg-migrate create <name>   → scaffold a new timestamped migration file
  npx node-pg-migrate up               → apply all pending migrations
  npx node-pg-migrate up N             → apply only the next N migrations
  npx node-pg-migrate down             → roll back the most recent migration
  npx node-pg-migrate down N           → roll back the last N migrations

KNEX CLI
  npx knex migrate:make <name>         → scaffold a new timestamped migration file
  npx knex migrate:latest              → apply all pending migrations
  npx knex migrate:rollback            → roll back the last BATCH of migrations
  npx knex migrate:rollback --all      → roll back every migration ever applied
  npx knex migrate:status              → show applied vs pending migrations

GOLDEN RULES
  1. NEVER edit a migration that has already run anywhere (staging/prod/teammate's machine)
     → create a NEW migration to make further changes instead.
  2. ALWAYS write a real down() — untested rollbacks are the #1 cause of failed deploys.
  3. NEVER run ad-hoc SQL directly on prod — every change goes through a migration file.
  4. Adding a NOT NULL column to a table with existing rows → add nullable first,
     backfill data, THEN alter to notNullable in a later step.
  5. Migrations run in FILE NAME ORDER — always use the tool's own timestamp generator,
     never rename or reorder files manually.
  6. Migrations run automatically in CI/CD BEFORE new app code starts serving traffic.

TYPICAL WORKFLOW
  1. Developer writes a migration locally, tests up() and down() on a local DB.
  2. Commits the migration file to git, opens a pull request.
  3. CI runs `migrate up` against a test/staging database as part of the pipeline.
  4. On deploy, `migrate up` runs against production before the new server version starts.
  5. If something breaks, `migrate:rollback` (or `down`) reverts the schema safely.
```

---

## Connected topics

- **90 — Connecting to PostgreSQL** — migrations run against the same `pg` connection pool you use for regular queries; understanding the driver first makes migration tooling make sense.
- **95 — Query builders — Knex** — the schema builder used inside `knex.schema.createTable()` migrations is the same query builder API you use for everyday CRUD queries.
- **97 — Transactions** — migration tools wrap each `up()`/`down()` run in a transaction so a failure halfway through rolls back cleanly, the same BEGIN/COMMIT/ROLLBACK mechanics covered there.
