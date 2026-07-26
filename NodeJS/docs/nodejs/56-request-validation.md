# 56 — Request validation

## What is this?

Request validation is the process of checking that incoming data — the request `body`, `params`, and `query` — actually matches the shape and rules your route expects, **before** any of it touches your business logic or database. Think of it like airport security: every passenger (request) passes through a checkpoint that checks their documents (fields, types, formats) against a fixed set of rules, and anyone who doesn't pass the checkpoint never reaches the gate (your controller code). Libraries like **Joi**, **Zod**, and **express-validator** let you declare those rules once and enforce them automatically on every request.

## Why does it matter for backend development?

Every backend API receives data from clients you do not control — browsers, mobile apps, Postman, or attackers. If you trust that data blindly, a missing field crashes your server with a `TypeError`, a wrong type corrupts your database, and an unchecked string opens the door to injection attacks. Validation is the single control point where you guarantee: required fields exist, types are correct (a `price` is really a number, not `"free"`), formats are valid (an email looks like an email), and extra/unexpected fields or malicious characters are stripped out (sanitization). Doing this at the edge of your app — before the controller — means your business logic can always assume clean, trustworthy data, which makes the rest of your codebase simpler and safer.

---

## Syntax / API

```js
// ── Option 1: Zod (schema-first, TypeScript-friendly, very popular in 2024+) ──
const { z } = require('zod');

// Define the shape the request body MUST match
const createUserSchema = z.object({
  userId:   z.string().uuid(),                       // must be a valid UUID string
  email:    z.string().email(),                       // must look like an email
  age:      z.number().int().min(13).max(120),        // whole number in a safe range
  bio:      z.string().max(280).optional(),           // optional, max length 280
});

// .parse() throws if invalid, .safeParse() returns { success, data/error }
const result = createUserSchema.safeParse(requestBody);

// ── Option 2: Joi (mature, widely used, no TS types required) ──────────────
const Joi = require('joi');

const createUserJoiSchema = Joi.object({
  userId: Joi.string().uuid().required(),             // required UUID
  email:  Joi.string().email().required(),            // required valid email
  age:    Joi.number().integer().min(13).max(120),     // optional number in range
  bio:    Joi.string().max(280),                       // optional short string
});

// .validate() returns { value, error } — value is the sanitized/coerced data
const { value, error } = createUserJoiSchema.validate(requestBody);

// ── Option 3: express-validator (middleware chain style, built for Express) ─
const { body, param, query, validationResult } = require('express-validator');

// Each check is middleware — chain them directly in the route definition
app.post(
  '/users/:userId',
  param('userId').isUUID(),                           // validate the :userId route param
  body('email').isEmail().normalizeEmail(),            // validate + sanitize email
  query('ref').optional().isAlphanumeric(),             // validate optional query string
  (req, res) => {
    const errors = validationResult(req);              // collect all validation errors
    if (!errors.isEmpty()) {
      return res.status(400).json({ errors: errors.array() }); // 400 = bad request
    }
    res.json({ message: 'Valid request' });
  }
);
```

---

## How it works — line by line

**Zod** works by building a "schema" object that describes the exact shape of valid data — field names, types, and rules like `.email()` or `.min()`. When you call `.safeParse(data)` on that schema, Zod walks through every field, checks it against the rule, and gives back an object telling you either "everything matched" (with the cleaned data) or "here is exactly what failed and why" — without ever throwing an error you have to catch.

**Joi** works almost the same way conceptually — you build a schema describing rules, then call `.validate(data)` on it. The key difference is that Joi's `.validate()` returns both an `error` (if something is wrong) and a `value` (the data after Joi has coerced types and applied defaults) — so you can trust `value` as the sanitized version of the input from that point onward.

**express-validator** takes a different approach: instead of one schema object, you write a chain of middleware functions directly on the route — one middleware per field per rule. Express runs each middleware in order, and each one silently records whether its check passed. At the end, you call `validationResult(req)` to collect every recorded failure across all those middlewares into one list, which you then turn into a single error response.

In all three approaches, the common pattern is: **validate first, reject early, only let clean data reach your controller logic.**

---

## Example 1 — basic

```js
// File: src/schemas/loginSchema.js
// Basic Zod schema for a login request body

const { z } = require('zod');

// Describe the exact shape a login request body must have
const loginSchema = z.object({
  email:    z.string().email('Email must be a valid email address'), // required, must be email format
  password: z.string().min(8, 'Password must be at least 8 characters'), // required, min length 8
});

// Function that validates a raw body and returns a clean result
function validateLogin(requestBody) {
  // safeParse never throws — it returns a success/failure object instead
  const result = loginSchema.safeParse(requestBody);

  if (!result.success) {
    // result.error.issues is an array describing every failed field
    return { ok: false, errors: result.error.issues };
  }

  // result.data is the validated, typed, clean version of the input
  return { ok: true, data: result.data };
}

// Try it with bad input
const bad = validateLogin({ email: 'not-an-email', password: '123' });
console.log(bad);
// → { ok: false, errors: [ ...two issues: bad email, password too short... ] }

// Try it with good input
const good = validateLogin({ email: 'jane@example.com', password: 'supersecret123' });
console.log(good);
// → { ok: true, data: { email: 'jane@example.com', password: 'supersecret123' } }

module.exports = { loginSchema, validateLogin };
```

---

## Example 2 — real world backend use case

```js
// File: src/middleware/validate.js
// A reusable Express middleware factory that validates req.body/params/query
// against any Zod schema — this is the pattern used in real production APIs.

const { z } = require('zod');

// source can be 'body', 'params', or 'query' — validate any part of the request
function validate(schema, source = 'body') {
  return (req, res, next) => {
    const dataToCheck = req[source];                      // pick body/params/query

    const result = schema.safeParse(dataToCheck);          // run validation

    if (!result.success) {
      // Flatten Zod's error shape into a friendly, consistent API error response
      const formattedErrors = result.error.issues.map((issue) => ({
        field:   issue.path.join('.'),                     // e.g. "email" or "address.city"
        message: issue.message,                             // human-readable reason
      }));

      return res.status(400).json({
        status: 'error',
        errors: formattedErrors,
      });
    }

    // Overwrite req[source] with the SANITIZED, type-coerced data — never trust raw input again
    req[source] = result.data;
    next();                                                 // pass control to the next handler
  };
}

module.exports = { validate };


// File: src/schemas/createOrderSchema.js
// Schemas describing every valid request shape for the "create order" endpoint

const { z } = require('zod');

const createOrderBodySchema = z.object({
  productId: z.string().uuid(),                             // must be a valid product UUID
  quantity:  z.number().int().positive().max(100),          // whole positive number, capped at 100
  couponCode: z.string().trim().toUpperCase().optional(),   // sanitized: trimmed + uppercased
});

const orderParamsSchema = z.object({
  userId: z.string().uuid(),                                 // route param must be a UUID
});

module.exports = { createOrderBodySchema, orderParamsSchema };


// File: src/routes/orderRoutes.js
// Wiring validation middleware into a real Express route

const express = require('express');
const router  = express.Router();
const { validate } = require('../middleware/validate');
const { createOrderBodySchema, orderParamsSchema } = require('../schemas/createOrderSchema');
const orderController = require('../controllers/orderController');

router.post(
  '/users/:userId/orders',
  validate(orderParamsSchema, 'params'),   // 1. validate :userId in the URL
  validate(createOrderBodySchema, 'body'), // 2. validate the JSON body
  orderController.createOrder             // 3. only reached if BOTH pass — data is now clean
);

module.exports = router;


// File: src/controllers/orderController.js
// The controller trusts req.body and req.params completely — validation already happened

function createOrder(req, res) {
  const { userId } = req.params;           // guaranteed to be a valid UUID string
  const { productId, quantity, couponCode } = req.body; // guaranteed valid types/ranges

  // ... safe to hit the database here, no defensive checks needed ...
  res.status(201).json({ userId, productId, quantity, couponCode });
}

module.exports = { createOrder };
```

---

## Common mistakes

### Mistake 1 — Validating only in the controller, after logic has already run

```js
// ❌ WRONG — the database call happens before validation, so bad data can slip through
// or cause a crash mid-operation
app.post('/orders', async (req, res) => {
  const order = await dbConnection.orders.create(req.body); // crashes if req.body.quantity is missing
  const isValid = req.body.quantity > 0;                    // checked AFTER the damage is done
  res.json(order);
});

// ✅ CORRECT — validate BEFORE any logic or database call runs
app.post('/orders', validate(createOrderBodySchema, 'body'), async (req, res) => {
  // by this point req.body is guaranteed valid — safe to use directly
  const order = await dbConnection.orders.create(req.body);
  res.json(order);
});
```

### Mistake 2 — Trusting req.body without stripping unknown/extra fields

```js
// ❌ WRONG — a client could send { "email": "a@b.com", "role": "admin" } and the extra
// "role" field silently passes through to the database, letting a user self-promote
const userSchema = Joi.object({
  email: Joi.string().email().required(),
}); // by default Joi ALLOWS unknown keys unless told otherwise

const { value } = userSchema.validate(requestBody);
await dbConnection.users.create(value); // "role: admin" could still leak in depending on ORM

// ✅ CORRECT — explicitly forbid unknown keys so extra/malicious fields are rejected
const strictUserSchema = Joi.object({
  email: Joi.string().email().required(),
}).options({ stripUnknown: true }); // or .unknown(false) to reject instead of strip

const { value: safeValue } = strictUserSchema.validate(requestBody);
await dbConnection.users.create(safeValue); // "role" is gone — never reaches the database
```

### Mistake 3 — Only validating types, not sanitizing/normalizing the actual values

```js
// ❌ WRONG — email passes "is a valid email" but is stored inconsistently
// ("Jane@Example.COM " vs "jane@example.com") causing duplicate accounts and failed logins
const emailSchema = z.object({
  email: z.string().email(), // valid format, but casing/whitespace untouched
});

const result = emailSchema.safeParse({ email: '  Jane@Example.COM  ' });
await dbConnection.users.create({ email: result.data.email }); // stored with stray casing/spaces

// ✅ CORRECT — sanitize as part of the schema so stored data is always normalized
const cleanEmailSchema = z.object({
  email: z.string().trim().toLowerCase().email(), // trims whitespace, lowercases, THEN validates
});

const cleanResult = cleanEmailSchema.safeParse({ email: '  Jane@Example.COM  ' });
await dbConnection.users.create({ email: cleanResult.data.email });
// → stored as "jane@example.com" every time, regardless of client input
```

---

## Practice exercises

### Exercise 1 — easy

Using Zod (or Joi, your choice), write a schema called `registerSchema` for a user registration request body with these rules:
1. `username` — required string, 3 to 20 characters
2. `email` — required, must be a valid email, trimmed and lowercased
3. `password` — required string, minimum 8 characters
4. `age` — optional whole number, must be at least 13 if provided

Write a function `validateRegister(requestBody)` that runs the schema and returns either `{ ok: true, data }` or `{ ok: false, errors }`. Test it with one valid payload and one invalid payload (missing `email`, `username` too short), and log both results.

```js
// Write your code here
```

---

### Exercise 2 — medium

Build a reusable Express middleware `validateRequest(schema, source)` (similar to Example 2) that:
1. Accepts any Zod schema and a `source` of `'body'`, `'params'`, or `'query'`
2. Runs `schema.safeParse()` against `req[source]`
3. On failure, responds with `400` and a JSON array of `{ field, message }` objects
4. On success, replaces `req[source]` with the sanitized/parsed data and calls `next()`

Then create two schemas and wire them into two separate routes:
- `GET /products?minPrice=&maxPrice=&inStock=` — validate `query` (numbers coerced from strings, `inStock` optional boolean)
- `PATCH /products/:productId` — validate `params.productId` as a UUID, and validate `body` allowing only `name`, `price`, `description` fields (reject any others)

Test both routes with valid and invalid requests (you can call the middleware function directly with mock `req`/`res`/`next` objects if you don't want to spin up a real server).

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a small validation framework that supports **conditional and cross-field rules**, something schema libraries alone don't always make easy. Create a function `validateSignupForm(requestBody)` that enforces:
1. `accountType` — required string, must be either `"personal"` or `"business"`
2. If `accountType === "business"`, then `companyName` becomes REQUIRED (string, min 2 chars) and `taxId` becomes REQUIRED (string matching pattern `/^[A-Z0-9]{6,12}$/`)
3. If `accountType === "personal"`, `companyName` and `taxId` must NOT be present at all (reject if they are)
4. `password` and `confirmPassword` are both required strings, and must be EQUAL to each other — return a specific error `{ field: 'confirmPassword', message: 'Passwords do not match' }` if they differ
5. Return a consistent shape: `{ ok: boolean, errors: [{ field, message }] }` collecting ALL failures at once (not just the first one found)

Test it with at least 4 payloads: valid personal account, valid business account, business account missing `taxId`, and personal account with mismatched passwords plus an unexpected `companyName`.

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
WHAT TO VALIDATE ON EVERY ROUTE
  req.body    → the JSON/form payload sent by the client
  req.params  → route segments, e.g. :userId in /users/:userId
  req.query   → the URL query string, e.g. ?page=2&limit=10

LIBRARIES
  Zod                → schema-first, .safeParse() returns {success, data/error}, no throw
  Joi                → schema-first, .validate() returns {value, error}, mature ecosystem
  express-validator  → middleware-chain style, built directly for Express routes

ZOD QUICK PATTERNS
  z.object({ field: z.string() })          → required string
  z.string().optional()                     → optional field
  z.string().email()                        → email format check
  z.string().uuid()                         → UUID format check
  z.number().int().positive()               → whole positive number
  z.string().trim().toLowerCase()           → SANITIZE before validating
  schema.safeParse(data)                    → never throws, always inspect .success

JOI QUICK PATTERNS
  Joi.object({ field: Joi.string().required() })
  Joi.string().email().required()
  Joi.number().integer().min(0).max(100)
  schema.validate(data)                     → { value, error }
  .options({ stripUnknown: true })          → drop unexpected fields
  .unknown(false)                            → REJECT unexpected fields (default is Joi.object)

EXPRESS-VALIDATOR QUICK PATTERNS
  body('email').isEmail().normalizeEmail()
  param('userId').isUUID()
  query('page').optional().isInt({ min: 1 })
  validationResult(req)                     → collect all errors after chain runs

SANITIZATION = CLEANING, VALIDATION = CHECKING
  Sanitize: trim(), toLowerCase(), escape(), normalizeEmail(), stripUnknown
  Validate: required fields present, correct type, correct format, in range

ALWAYS
  Validate BEFORE controller logic runs — reject early with 400
  Reject/strip unknown fields — never let extra client data reach the database
  Return a consistent error shape across the whole API
  Sanitize as part of the schema, not as a separate manual step

NEVER
  Trust req.body/params/query without validating first
  Assume "type looks right" means "value is safe" (still sanitize strings)
  Let a validation library's default "allow unknown keys" behavior go unchecked
```

---

## Connected topics

- **55 — Express middleware in depth** — validation is implemented as middleware; understanding the middleware chain and `next()` is required to build validators like this
- **62 — Centralized error handling** — validation errors should flow into the same consistent error response format as every other error in the app
- **75 — Input validation and sanitization** — the security-focused deep dive into why unsanitized input leads to injection and XSS, building directly on the validation habits formed here
