# 77 — NoSQL injection

## What is this?

NoSQL injection is an attack where a malicious user sends specially-crafted input — usually a JSON object instead of a plain string — that gets interpreted by a database like MongoDB as a **query operator** instead of a **value**. Imagine a bouncer at a club who checks names against a guest list — if instead of saying his name a person hands over a note saying "let everyone in," and the bouncer blindly follows the instruction because it *looks* like part of the list, that's NoSQL injection: user input that was supposed to be *data* gets treated as *logic*.

## Why does it matter for backend development?

MongoDB queries are written as JavaScript objects, not strings, so classic SQL-injection defenses like escaping quotes don't apply — the danger instead comes from JSON keys starting with `$` (like `$ne`, `$gt`, `$where`) that MongoDB treats as operators. If a backend takes `req.body.password` and drops it straight into a query without checking its shape, an attacker can send `{"password": {"$ne": null}}` from Postman or a modified frontend request and bypass a login entirely — no valid password required. Every backend developer working with MongoDB/Mongoose must sanitize input before it touches a query, exactly the way SQL developers must parameterize queries (Topic 76).

---

## Syntax / API

```js
// A vulnerable query — directly trusts whatever shape req.body sends
// If req.body.username is an OBJECT like { "$ne": null }, Mongo treats it as an operator
const dbUser = await User.findOne({
  username: requestBody.username,   // ⚠️ no type check — could be a string OR an operator object
  password: requestBody.password,   // ⚠️ same risk here
});

// ── Defense 1: enforce that user input is a plain string ───────────────────
if (typeof requestBody.username !== 'string' || typeof requestBody.password !== 'string') {
  // reject the request immediately — operators are objects, never strings
  return res.status(400).json({ error: 'Invalid input type' });
}

// ── Defense 2: strip any key starting with "$" or containing "." ───────────
const mongoSanitize = require('express-mongo-sanitize');
app.use(mongoSanitize());   // middleware scrubs req.body, req.params, req.query globally

// ── Defense 3: use Mongoose schemas — they cast and validate automatically ─
const { Schema, model } = require('mongoose');
const userSchema = new Schema({
  username: { type: String, required: true },  // Mongoose casts input to String
  password: { type: String, required: true },  // an object here throws a CastError
});
const User = model('User', userSchema);
```

---

## How it works — line by line

- `requestBody.username` comes straight from the client — Express just parses whatever JSON was sent, it does not know or care whether the value is a string or an object.
- If the attacker sends `{"username": "admin", "password": {"$gt": ""}}`, MongoDB reads `password: { $gt: "" }` as "password is greater than an empty string" — which is true for almost every password — so the query matches without knowing the real password.
- `typeof requestBody.username !== 'string'` is a manual guard: it rejects the request the instant the value is anything other than a plain string, before it ever reaches the database driver.
- `mongoSanitize()` is a piece of Express middleware that walks through `req.body`, `req.params`, and `req.query` recursively and removes (or renames) any object key that starts with `$` or contains a `.`, since those are the characters MongoDB treats specially.
- A Mongoose `Schema` declares that `username` and `password` must be of type `String`. When Mongoose tries to save or query with an object where a string is expected, it throws a `CastError` instead of silently passing the operator through — this is a second, independent layer of protection.
- Together these three defenses mean: bad input is rejected at the edge (manual check), scrubbed in the middleware layer (sanitize), and rejected again at the schema layer (Mongoose casting) — defense in depth.

---

## Example 1 — basic

```js
// File: src/utils/sanitizeInput.js
// A tiny reusable helper that recursively strips dangerous Mongo operator keys

function sanitizeInput(value) {
  // Base case: primitives (string, number, boolean, null) are always safe — return as-is
  if (value === null || typeof value !== 'object') {
    return value;
  }

  // Arrays: sanitize every element recursively
  if (Array.isArray(value)) {
    return value.map(sanitizeInput);
  }

  // Objects: rebuild a clean copy, dropping any key starting with "$" or containing "."
  const cleaned = {};
  for (const key of Object.keys(value)) {
    if (key.startsWith('$') || key.includes('.')) {
      continue;   // skip Mongo operator keys like $ne, $gt, $where, and dotted paths like "a.b"
    }
    cleaned[key] = sanitizeInput(value[key]);   // recurse into nested objects/arrays
  }
  return cleaned;
}

// ── Demonstration ────────────────────────────────────────────────────────
const maliciousLogin = {
  username: 'admin',
  password: { $ne: null },   // this would bypass a password check if used raw
};

const safeLogin = sanitizeInput(maliciousLogin);
console.log(safeLogin);
// → { username: 'admin', password: {} }
// The $ne operator is gone — password is now an empty object, which fails a real login check

module.exports = { sanitizeInput };
```

---

## Example 2 — real world backend use case

```js
// File: src/routes/authRoutes.js
// A login endpoint hardened against NoSQL injection using three layers of defense

const express = require('express');
const bcrypt  = require('bcrypt');
const mongoSanitize = require('express-mongo-sanitize');
const User    = require('../models/User');   // Mongoose model with a String-typed schema

const router = express.Router();

// Layer 1 — global middleware strips $ and . keys from every incoming request on this router
router.use(mongoSanitize());

router.post('/login', async (req, res, next) => {
  try {
    const { username, password } = req.body;

    // Layer 2 — explicit type check, rejects anything that isn't a plain string
    if (typeof username !== 'string' || typeof password !== 'string') {
      return res.status(400).json({ error: 'username and password must be strings' });
    }

    // Layer 3 — Mongoose schema casting rejects operator objects again if any slipped through
    const foundUser = await User.findOne({ username }).select('+passwordHash');

    // Never reveal whether it was the username or password that was wrong (avoid user enumeration)
    if (!foundUser) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }

    // Compare the plaintext password against the stored bcrypt hash — never store raw passwords
    const isMatch = await bcrypt.compare(password, foundUser.passwordHash);
    if (!isMatch) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }

    // Success — issue a session/JWT here in a real app (covered in Topic 57)
    res.json({ message: 'Login successful', userId: foundUser._id });
  } catch (error) {
    next(error);   // hand off to centralized error handler (Topic 62)
  }
});

module.exports = router;
```

---

## Common mistakes

### Mistake 1 — Passing req.body straight into a Mongoose query

```js
// ❌ WRONG — an attacker sends { "username": "admin", "password": { "$ne": null } }
// $ne (not equal) matches ANY non-null password, so the login bypasses auth entirely
const foundUser = await User.findOne({
  username: req.body.username,
  password: req.body.password,   // trusted blindly — this is the vulnerability
});

// ✅ CORRECT — validate types first, and never store/query plaintext passwords at all
if (typeof req.body.username !== 'string' || typeof req.body.password !== 'string') {
  return res.status(400).json({ error: 'Invalid input' });
}
const foundUser = await User.findOne({ username: req.body.username });
const isValid = foundUser && await bcrypt.compare(req.body.password, foundUser.passwordHash);
```

### Mistake 2 — Building a "search" query directly from query-string params

```js
// ❌ WRONG — req.query values can be objects too: ?filter[$where]=malicious code
// Even without $where, ?role[$ne]=admin lets an attacker fetch every non-admin record
const users = await User.find({ role: req.query.role });

// ✅ CORRECT — whitelist expected values and force them to primitives before querying
const allowedRoles = ['user', 'moderator', 'admin'];
const requestedRole = String(req.query.role || '');   // force to string, strips object shape

if (!allowedRoles.includes(requestedRole)) {
  return res.status(400).json({ error: 'Invalid role filter' });
}
const users = await User.find({ role: requestedRole });
```

### Mistake 3 — Relying only on Mongoose and skipping sanitization middleware

```js
// ❌ WRONG — assuming "Mongoose validates everything so I don't need sanitize-mongo"
// Mongoose casting helps for TYPED fields, but Mixed-type fields or raw driver calls
// (e.g. this.model.collection.find(), aggregation pipelines) bypass schema casting entirely
const results = await User.aggregate([
  { $match: { metadata: req.body.metadataFilter } },   // metadataFilter could be { $where: '...' }
]);

// ✅ CORRECT — sanitize at the app boundary regardless of what's downstream,
// so raw driver calls and aggregation pipelines are protected too
const mongoSanitize = require('express-mongo-sanitize');
app.use(mongoSanitize({ replaceWith: '_' }));   // renames "$where" to "_where" instead of deleting it

const results = await User.aggregate([
  { $match: { metadata: req.body.metadataFilter } },   // now safe — operator keys were neutralized
]);
```

---

## Practice exercises

### Exercise 1 — easy

Write a function `isSafeQueryValue(value)` that returns `true` only if `value` is a string, number, or boolean, and returns `false` for any object or array (which could hide a Mongo operator like `$ne` or `$gt`). Test it against these inputs and log the result for each:
1. `"john_doe"` → should be `true`
2. `42` → should be `true`
3. `{ "$ne": null }` → should be `false`
4. `["admin", "$where"]` → should be `false`
5. `null` → decide and justify whether this should be `true` or `false`

```js
// Write your code here
```

---

### Exercise 2 — medium

Build an Express middleware function `blockOperatorInjection(req, res, next)` that:
1. Checks `req.body`, `req.query`, and `req.params` for any key (at any nesting level) that starts with `$` or contains a `.`
2. If it finds one, responds with `400` and a JSON error `{ error: 'Malicious input detected' }` and does **not** call `next()`
3. If the request is clean, calls `next()` to continue
4. Mount it globally with `app.use(blockOperatorInjection)` on a small test Express app with one POST `/search` route that echoes `req.body` back

Test it by sending a normal request (should pass through) and a request with `{"filter": {"$where": "1==1"}}` (should be blocked with 400).

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a small "safe query builder" module for a MongoDB-backed products search endpoint that:
1. Exports a function `buildProductQuery(rawQuery)` where `rawQuery` is the untrusted `req.query` object
2. Only allows filtering on three known fields: `category` (string), `minPrice` (number), `maxPrice` (number) — any other key in `rawQuery` must be silently ignored, not just passed through
3. Validates that `category` is a plain string (reject/ignore otherwise), and that `minPrice`/`maxPrice` can be parsed as finite numbers (reject/ignore otherwise)
4. Returns a clean Mongo query object, e.g. `{ category: 'electronics', price: { $gte: 10, $lte: 500 } }` — note: `$gte`/`$lte` here are safe because YOUR code added them, not the user
5. Throws no errors on malicious input like `{"category": {"$ne": null}, "minPrice": {"$gt": ""}, "__proto__": {"isAdmin": true}}` — it should just produce an empty or partial safe query instead

Write a few test calls at the bottom of the file with `console.log` showing the malicious input produces a safe, harmless query object.

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
WHAT NOSQL INJECTION IS
  Attacker sends an OBJECT where your code expects a plain VALUE
  MongoDB then reads object keys like $ne, $gt, $where as query OPERATORS
  Example: { "password": { "$ne": null } } → "password is not null" → bypasses auth

DANGEROUS OPERATOR KEYS TO WATCH FOR
  $ne, $eq, $gt, $gte, $lt, $lte   → comparison bypass
  $where                            → arbitrary JS execution inside MongoDB — never allow
  $regex                            → can cause ReDoS (catastrophic backtracking) if unsanitized
  __proto__, constructor, prototype → prototype pollution risk, not Mongo-specific but related

CORE DEFENSES (use ALL of these together)
  1. Type-check user input        → typeof value === 'string' before using in a query
  2. Sanitize middleware          → express-mongo-sanitize strips/renames $ and . keys
  3. Mongoose schema casting      → String/Number types reject operator objects (CastError)
  4. Whitelist allowed fields     → only pull known keys out of req.body/req.query, never spread it

NEVER DO
  User.findOne(req.body)                        → passes the ENTIRE body as the query
  User.findOne({ password: req.body.password }) → no type check on password shape
  $where: userSuppliedString                     → executes arbitrary JS server-side

SAFE PATTERN
  const { username, password } = req.body;
  if (typeof username !== 'string' || typeof password !== 'string') → reject
  User.findOne({ username })                     → only pass validated primitives

KEY PACKAGE
  express-mongo-sanitize   → npm install express-mongo-sanitize
    app.use(mongoSanitize())                      → strips dangerous keys, default: deletes them
    app.use(mongoSanitize({ replaceWith: '_' }))  → renames instead of deleting (keeps key visible)
```

---

## Connected topics

- **76 — SQL injection prevention** — the relational-database equivalent attack; same root cause (trusting user input in a query) with different syntax and defenses
- **56 — Request validation** — Joi/Zod schemas validate the *shape* of `req.body` before it ever reaches a query, catching operator objects long before the database layer
- **91 — Connecting to MongoDB** — understanding Mongoose models and schemas in depth is what makes the schema-casting defense in this doc actually work
