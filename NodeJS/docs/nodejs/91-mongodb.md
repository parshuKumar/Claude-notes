# 91 — Connecting to MongoDB

## What is this?

MongoDB is a NoSQL database that stores data as flexible, JSON-like documents instead of rigid rows and columns. Mongoose is a Node.js library that sits on top of the raw MongoDB driver and gives you **schemas** (a blueprint for what a document should look like), **models** (classes you use to create/read/update/delete documents), and a managed **connection lifecycle**. Think of MongoDB as a warehouse full of folders, and Mongoose as the strict clerk who enforces a form template before any folder is filed away — even though the warehouse itself would happily accept any messy paperwork.

## Why does it matter for backend development?

Almost every backend app needs to persist data — users, orders, sessions, products — and MongoDB is one of the two most common choices (alongside SQL databases like PostgreSQL, covered in Topic 90). Backend developers use Mongoose daily to define what a "User" or "Order" document looks like, validate incoming data before it touches the database, and perform CRUD (Create, Read, Update, Delete) operations with a clean, promise-based API instead of writing raw MongoDB driver calls by hand. Getting the connection lifecycle right — connecting once at startup, handling disconnects, closing cleanly on shutdown — is the difference between a server that silently hangs in production and one that fails fast with a useful error.

---

## Syntax / API

```js
// Install first: npm install mongoose
const mongoose = require('mongoose');

// ── 1. CONNECT ───────────────────────────────────────────────────────────────
// connectionString points to your MongoDB instance (local or Atlas cloud)
const connectionString = process.env.MONGO_URI || 'mongodb://127.0.0.1:27017/myapp';

// connect() returns a Promise — resolves once the connection is established
mongoose.connect(connectionString)
  .then(() => console.log('MongoDB connected'))
  .catch((err) => console.error('MongoDB connection error:', err));

// ── 2. DEFINE A SCHEMA ────────────────────────────────────────────────────────
// Schema = the shape and rules for a document (like a table structure in SQL)
const userSchema = new mongoose.Schema({
  name:  { type: String, required: true, trim: true },       // must be present, no extra whitespace
  email: { type: String, required: true, unique: true },     // must be present and unique across the collection
  age:   { type: Number, min: 0 },                            // optional, but must be >= 0 if present
}, {
  timestamps: true, // auto-adds createdAt and updatedAt fields
});

// ── 3. CREATE A MODEL ─────────────────────────────────────────────────────────
// Model = a class built from the schema — this is what you use to talk to MongoDB
const User = mongoose.model('User', userSchema); // 'User' → stored in the "users" collection

// ── 4. CRUD OPERATIONS ────────────────────────────────────────────────────────
await User.create({ name: 'Aisha', email: 'aisha@example.com' }); // Create
const users = await User.find({ age: { $gte: 18 } });             // Read (query)
const oneUser = await User.findById('64f1a2b3c4d5e6f789012345');  // Read (by id)
await User.updateOne({ email: 'aisha@example.com' }, { age: 30 }); // Update
await User.deleteOne({ email: 'aisha@example.com' });              // Delete

// ── 5. DISCONNECT (on graceful shutdown) ──────────────────────────────────────
await mongoose.disconnect(); // closes the connection cleanly
```

---

## How it works — line by line

- `mongoose.connect(connectionString)` opens a connection to a running MongoDB server using the given URI. This returns a Promise, so you either `.then()`/`.catch()` it or `await` it inside an async function.
- A **schema** is not the database talking back to you — it is Mongoose's own rulebook, enforced entirely in your Node.js process before anything is sent to MongoDB. `required: true` means Mongoose refuses to save a document missing that field. `unique: true` asks MongoDB to build an index that rejects duplicate values.
- `timestamps: true` is a Mongoose convenience option — it automatically adds and maintains `createdAt` and `updatedAt` fields on every document without you writing that logic yourself.
- `mongoose.model('User', userSchema)` compiles the schema into a **model** — a JavaScript class. Calling it with a capitalized singular name (`'User'`) tells Mongoose to store documents in a pluralized, lowercased collection called `users`.
- `User.create(...)` builds a new document in memory, validates it against the schema, and inserts it into the `users` collection — all in one call.
- `User.find(...)` sends a query to MongoDB and returns an array of matching documents as Mongoose document objects (not plain JSON — they have extra methods attached).
- `User.updateOne(...)` and `User.deleteOne(...)` locate the first document matching the filter (the first argument) and apply the change — they do not load the document into memory first, making them efficient for simple updates.
- `mongoose.disconnect()` closes the underlying socket connection to MongoDB — important during graceful shutdown (Topic 64) so the process doesn't hang or leak open connections.

---

## Example 1 — basic

```js
// File: src/db.js
// Minimal script: connect, define a schema/model, do one CRUD cycle, disconnect.

const mongoose = require('mongoose');

async function run() {
  // Connect to a local MongoDB instance and a database named "practiceDb"
  await mongoose.connect('mongodb://127.0.0.1:27017/practiceDb');
  console.log('Connected to MongoDB');

  // Define a simple schema for a "Note" document
  const noteSchema = new mongoose.Schema({
    title: { type: String, required: true },  // note must have a title
    body:  { type: String, default: '' },      // body is optional, defaults to empty string
    done:  { type: Boolean, default: false },   // whether the note is completed
  });

  // Compile the schema into a usable model
  const Note = mongoose.model('Note', noteSchema);

  // CREATE — insert a new note document
  const created = await Note.create({ title: 'Buy groceries', body: 'Milk, eggs, bread' });
  console.log('Created:', created._id); // MongoDB auto-generates a unique _id

  // READ — fetch it back by id
  const found = await Note.findById(created._id);
  console.log('Found:', found.title, found.done);

  // UPDATE — mark it as done
  await Note.updateOne({ _id: created._id }, { done: true });

  // DELETE — remove it once we're done demonstrating
  await Note.deleteOne({ _id: created._id });

  // Always close the connection when a standalone script finishes
  await mongoose.disconnect();
  console.log('Disconnected');
}

run().catch((err) => console.error('Script failed:', err)); // catch any error in the async chain
```

---

## Example 2 — real world backend use case

```js
// File: src/models/user.model.js
// A realistic User model with validation rules, indexes, and instance methods —
// the pattern used in real Express + MongoDB backends.

const mongoose = require('mongoose');

const userSchema = new mongoose.Schema({
  email: {
    type: String,
    required: [true, 'Email is required'],   // custom error message
    unique: true,                             // enforced with a unique index
    lowercase: true,                          // normalize before saving
    trim: true,
  },
  passwordHash: {
    type: String,
    required: true,
    select: false,                            // excluded from queries by default (never leak by accident)
  },
  role: {
    type: String,
    enum: ['user', 'admin'],                  // only these two values are allowed
    default: 'user',
  },
}, { timestamps: true });

const User = mongoose.model('User', userSchema);
module.exports = User;
```

```js
// File: src/db.js
// Connection lifecycle module — shared across the whole app, connected once at startup.

const mongoose = require('mongoose');

async function connectDB() {
  const mongoUri = process.env.MONGO_URI; // read connection string from environment (Topic 12)

  if (!mongoUri) {
    throw new Error('MONGO_URI is not set in environment variables');
  }

  // Listen for connection events so ops issues surface in logs, not silently
  mongoose.connection.on('connected', () => console.log('[db] MongoDB connected'));
  mongoose.connection.on('error', (err) => console.error('[db] MongoDB error:', err));
  mongoose.connection.on('disconnected', () => console.warn('[db] MongoDB disconnected'));

  await mongoose.connect(mongoUri, {
    maxPoolSize: 10, // cap the number of simultaneous socket connections (Topic 93)
  });
}

module.exports = { connectDB };
```

```js
// File: src/services/user.service.js
// A service layer performing CRUD — this is what a controller/route calls into.

const User = require('../models/user.model');
const bcrypt = require('bcrypt');

// Create a new user, hashing the password before saving
async function registerUser(email, plainPassword) {
  const passwordHash = await bcrypt.hash(plainPassword, 10); // never store raw passwords
  const user = await User.create({ email, passwordHash });
  return user;
}

// Find a user by email, explicitly including the normally-hidden passwordHash
async function findUserForLogin(email) {
  return User.findOne({ email }).select('+passwordHash');
}

// Update a user's role — only admins should be able to call this in practice
async function promoteToAdmin(userId) {
  const updated = await User.findByIdAndUpdate(
    userId,
    { role: 'admin' },
    { new: true } // return the document AFTER the update, not before
  );
  if (!updated) throw new Error(`User not found: ${userId}`);
  return updated;
}

// Soft-delete pattern is common in real apps, but here's a real delete for contrast
async function deleteUser(userId) {
  const result = await User.deleteOne({ _id: userId });
  return result.deletedCount === 1; // deleteCount tells you if anything was actually removed
}

module.exports = { registerUser, findUserForLogin, promoteToAdmin, deleteUser };
```

---

## Common mistakes

### Mistake 1 — Calling mongoose.connect() on every request instead of once at startup

```js
// ❌ WRONG — reconnecting inside a route handler exhausts connections and is slow
app.get('/users', async (req, res) => {
  await mongoose.connect(process.env.MONGO_URI); // opens a new connection every request!
  const users = await User.find();
  res.json(users);
});

// ✅ CORRECT — connect once when the server starts, reuse the connection everywhere
async function startServer() {
  await mongoose.connect(process.env.MONGO_URI); // one connection, reused by the pool
  app.listen(3000, () => console.log('Server running'));
}
startServer();

// Then routes just use the model directly — Mongoose queues operations until connected
app.get('/users', async (req, res) => {
  const users = await User.find();
  res.json(users);
});
```

### Mistake 2 — No error handling on the initial connection

```js
// ❌ WRONG — if MongoDB is down or the URI is wrong, this fails silently
// and the app appears to "hang" with no clue why
mongoose.connect(process.env.MONGO_URI);
app.listen(3000);

// ✅ CORRECT — await the connection and exit clearly if it fails
async function startServer() {
  try {
    await mongoose.connect(process.env.MONGO_URI);
    console.log('MongoDB connected');
    app.listen(3000, () => console.log('Server listening on port 3000'));
  } catch (err) {
    console.error('Failed to connect to MongoDB:', err.message);
    process.exit(1); // fail fast instead of running a broken server (Topic 05)
  }
}
startServer();
```

### Mistake 3 — Trusting user input directly in a query filter (NoSQL injection risk)

```js
// ❌ WRONG — if req.body.email is an object like { "$ne": null },
// this query can match EVERY user instead of one, bypassing login logic
const user = await User.findOne({
  email: req.body.email,
  passwordHash: req.body.password,
});

// ✅ CORRECT — force fields to the expected primitive type before querying
const email = String(req.body.email);       // coerce to string, defeats operator injection
const password = String(req.body.password);
const user = await User.findOne({ email }).select('+passwordHash');
// Then compare the password separately with bcrypt.compare() — never query by raw password
// (Full defense-in-depth covered in Topic 77 — NoSQL injection)
```

---

## Practice exercises

### Exercise 1 — easy

Write a script that:
1. Connects to a local MongoDB instance (`mongodb://127.0.0.1:27017/practiceDb`)
2. Defines a `Product` schema with `name` (String, required), `price` (Number, required, minimum 0), and `inStock` (Boolean, default `true`)
3. Creates two products
4. Fetches and logs all products where `inStock` is `true`
5. Disconnects cleanly at the end

```js
// Write your code here
```

---

### Exercise 2 — medium

Build a small `taskService.js` module (CommonJS, `module.exports`) backed by a `Task` model with fields: `title` (String, required), `priority` (String, enum: `'low' | 'medium' | 'high'`, default `'medium'`), `completed` (Boolean, default `false`), and `timestamps: true`. Implement and export these functions:
1. `createTask(title, priority)` — creates and returns a new task
2. `listIncompleteTasks()` — returns all tasks where `completed` is `false`, sorted by `priority`
3. `completeTask(taskId)` — sets `completed` to `true` for a given task and returns the updated document (use `findByIdAndUpdate` with `{ new: true }`)
4. `removeTask(taskId)` — deletes a task and returns `true`/`false` depending on whether anything was deleted

Write a small script at the bottom that calls all four functions in sequence and logs the results.

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a connection-lifecycle module `db.js` used by an Express-style app that:
1. Exports a `connectDB(uri)` function that connects with Mongoose and registers listeners for the `connected`, `error`, and `disconnected` events on `mongoose.connection`, logging each with a `[db]` prefix
2. Exports a `disconnectDB()` function that gracefully closes the connection
3. Retries the initial connection up to 3 times with a 2-second delay between attempts if it fails (do not use any external retry library — write the loop yourself)
4. Throws a clear error after the 3rd failed attempt instead of hanging forever
5. Simulate graceful shutdown: register a handler on `process.on('SIGINT', ...)` that calls `disconnectDB()` and then `process.exit(0)`

Test it by intentionally pointing the URI at a wrong port first (to see the retry logs), then at the correct URI (to see it succeed).

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
CONNECTION
  mongoose.connect(uri)              → returns a Promise, await/catch it
  mongoose.disconnect()               → closes the connection cleanly
  mongoose.connection.on('connected'/'error'/'disconnected', fn) → lifecycle events
  Connect ONCE at startup — never inside a route handler

SCHEMA OPTIONS (common)
  required: true                     → field must be present
  unique: true                       → builds a unique index (not a validator!)
  default: value                     → fallback if not provided
  enum: [...]                        → restrict to a fixed set of values
  select: false                      → excluded from query results unless .select('+field')
  trim / lowercase                   → string normalization
  timestamps: true (schema option)   → auto createdAt / updatedAt

MODEL
  mongoose.model('Name', schema)     → 'Name' becomes lowercase, pluralized collection

CRUD METHODS
  Model.create(data)                 → insert one document
  Model.insertMany([data, data])     → insert many documents at once
  Model.find(filter)                 → array of matching documents
  Model.findOne(filter)              → first matching document or null
  Model.findById(id)                 → lookup by _id
  Model.updateOne(filter, changes)   → update first match, no document returned
  Model.findByIdAndUpdate(id, changes, { new: true }) → update AND return the new doc
  Model.deleteOne(filter)            → delete first match
  Model.deleteMany(filter)           → delete all matches
  Model.countDocuments(filter)       → count matches without fetching them

GOTCHAS
  unique: true builds an index but is NOT itself a validator — duplicates can slip
    through under race conditions; catch the E11000 duplicate key error too.
  Never pass raw req.body values directly into a filter — coerce types first
    to prevent NoSQL operator injection ($ne, $gt, etc.).
  findOneAndUpdate/findByIdAndUpdate return the OLD document unless
    { new: true } is passed.
  Forgetting to await mongoose.connect() means routes may run before the
    connection is ready — Mongoose buffers ops by default but don't rely on it
    in production.
```

---

## Connected topics

- **90 — Connecting to PostgreSQL** — the SQL counterpart; compare connection pooling and query style against MongoDB's document model
- **77 — NoSQL injection** — goes deep on the operator-injection risk shown in Mistake 3 and how Mongoose schemas help mitigate it
- **97 — Transactions** — MongoDB supports multi-document transactions too; useful once CRUD across multiple collections must be atomic
