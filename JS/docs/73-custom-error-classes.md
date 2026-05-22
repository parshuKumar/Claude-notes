# 73 — Custom Error Classes

---

## 1. What is this?

A **custom error class** is a subclass of `Error` that you define yourself. Instead of throwing a generic `new Error("something failed")`, you throw a `new ValidationError("Email is invalid")` — an object with a specific type, extra properties, and a meaningful name that both code and logs can act on.

**Analogy — HTTP status codes vs a plain 500:**
A server that always returns `500` for every problem is useless. A server that returns `400 Bad Request`, `401 Unauthorized`, `404 Not Found`, `409 Conflict` lets the client handle each case correctly. Custom error classes give your errors the same specificity — instead of `catch (e)` guessing from a message string, it does `if (e instanceof ValidationError)` and knows exactly what to do.

---

## 2. Why does it matter?

- `catch` blocks can branch on type (`instanceof`) instead of fragile message string parsing
- Extra properties (`statusCode`, `fields`, `code`, `retryable`) carry structured metadata
- Errors log with meaningful `.name` values — `ValidationError` in a stack trace is immediately clear
- Error hierarchies let you catch at any level of specificity
- Mirrors the pattern used by every major Node.js library (Mongoose `ValidationError`, JWT `JsonWebTokenError`, Axios `AxiosError`, etc.)

---

## 3. Syntax

```js
class CustomError extends Error {
  constructor(message, options) {
    super(message, options); // passes message and { cause } to Error
    this.name = this.constructor.name; // "CustomError" — not "Error"
  }
}

// Throw it
throw new CustomError("something specific");

// Catch it
try {
  riskyOp();
} catch (e) {
  if (e instanceof CustomError) { /* handle */ }
  else throw e;
}
```

---

## 4. How it works — line by line

```js
class AppError extends Error {
  constructor(message, options) {
    super(message, options);
    // 1. super(message) — sets this.message and this.stack
    // 2. options = { cause } — sets this.cause (ES2022)

    this.name = this.constructor.name;
    // 3. Without this line, e.name would be "Error" for all subclasses
    //    this.constructor.name gives "AppError", "ValidationError", etc.
    //    dynamically — works correctly for further subclasses too
  }
}

const e = new AppError("something went wrong");
e.name;      // "AppError"
e.message;   // "something went wrong"
e.stack;     // "AppError: something went wrong\n  at ..."
e instanceof AppError; // true
e instanceof Error;    // true
```

### Why `this.name = this.constructor.name` matters

```js
// Without it:
class ValidationError extends Error {}
const e = new ValidationError("bad input");
e.name; // "Error" — misleading in logs and stack traces

// With it:
class ValidationError extends Error {
  constructor(message, options) {
    super(message, options);
    this.name = this.constructor.name;
  }
}
const e = new ValidationError("bad input");
e.name; // "ValidationError" — correct
```

---

## 5. Adding Custom Properties

```js
class HttpError extends Error {
  constructor(statusCode, message, options) {
    super(message, options);
    this.name       = this.constructor.name;
    this.statusCode = statusCode;
    this.isHttpError = true; // flag for middleware to detect
  }
}

const err = new HttpError(404, "User not found");
err.statusCode; // 404
err.message;    // "User not found"
err.name;       // "HttpError"

// In Express error middleware:
app.use((err, req, res, next) => {
  if (err instanceof HttpError) {
    return res.status(err.statusCode).json({ error: err.message });
  }
  res.status(500).json({ error: "Internal server error" });
});
```

---

## 6. A Practical Error Hierarchy

This is the pattern used in real backend applications:

```js
// ─── Base ───────────────────────────────────────────────────────────────────

class AppError extends Error {
  constructor(message, options = {}) {
    super(message, options);
    this.name = this.constructor.name;
  }
}

// ─── HTTP layer ─────────────────────────────────────────────────────────────

class HttpError extends AppError {
  constructor(statusCode, message, options = {}) {
    super(message, options);
    this.statusCode = statusCode;
  }
}

class NotFoundError extends HttpError {
  constructor(resource = "Resource", options) {
    super(404, `${resource} not found`, options);
  }
}

class UnauthorizedError extends HttpError {
  constructor(message = "Authentication required", options) {
    super(401, message, options);
  }
}

class ForbiddenError extends HttpError {
  constructor(message = "Access denied", options) {
    super(403, message, options);
  }
}

class ConflictError extends HttpError {
  constructor(message, options) {
    super(409, message, options);
  }
}

// ─── Validation ─────────────────────────────────────────────────────────────

class ValidationError extends AppError {
  constructor(message, fields = {}) {
    super(message);
    this.statusCode = 422;
    this.fields     = fields; // { fieldName: "error message" }
  }
}

// ─── Infrastructure ─────────────────────────────────────────────────────────

class DatabaseError extends AppError {
  constructor(message, options) {
    super(message, options);
    this.statusCode  = 503;
    this.retryable   = true;
  }
}

class NetworkError extends AppError {
  constructor(message, options) {
    super(message, options);
    this.statusCode = 503;
    this.retryable  = true;
  }
}

class ConfigurationError extends AppError {
  constructor(message) {
    super(message);
    this.statusCode = 500;
    this.retryable  = false;
  }
}
```

### Using the hierarchy

```js
// Throw specific errors
throw new NotFoundError("User");
throw new ValidationError("Invalid input", {
  email:    "Must be a valid email",
  password: "Minimum 8 characters"
});
throw new DatabaseError("Query timeout", { cause: originalDbError });

// Catch at any level of specificity
catch (e) {
  if (e instanceof ValidationError) {
    return res.status(422).json({ error: e.message, fields: e.fields });
  }
  if (e instanceof HttpError) {
    return res.status(e.statusCode).json({ error: e.message });
  }
  if (e instanceof AppError && e.retryable) {
    return scheduleRetry();
  }
  // Unknown — let Express error middleware handle it
  next(e);
}
```

---

## 7. Real-World Examples

### Example 1 — Full Express API error system

```js
// errors.js — shared error classes for the whole app

export class AppError extends Error {
  constructor(message, options = {}) {
    super(message, options);
    this.name = this.constructor.name;
  }
}

export class HttpError extends AppError {
  constructor(statusCode, message, options) {
    super(message, options);
    this.statusCode = statusCode;
  }
}

export class ValidationError extends AppError {
  constructor(fields) {
    const message = Object.values(fields).flat().join(", ");
    super(message);
    this.statusCode = 422;
    this.fields     = fields;
  }
}

export class NotFoundError    extends HttpError {
  constructor(r)  { super(404, `${r} not found`); }
}
export class UnauthorizedError extends HttpError {
  constructor(m = "Unauthorized") { super(401, m); }
}
export class RateLimitError   extends HttpError {
  constructor()   { super(429, "Too many requests"); }
}
```

```js
// middleware/errorHandler.js

export function errorHandler(err, req, res, next) {
  // Log everything — always
  console.error({
    name:    err.name,
    message: err.message,
    stack:   err.stack,
    cause:   err.cause,
    path:    req.path,
    method:  req.method
  });

  // ValidationError — expose field details to client
  if (err instanceof ValidationError) {
    return res.status(422).json({
      error:  err.message,
      fields: err.fields
    });
  }

  // Any HttpError subclass
  if (err instanceof HttpError) {
    return res.status(err.statusCode).json({ error: err.message });
  }

  // Unknown error — never leak internals in production
  res.status(500).json({
    error: process.env.NODE_ENV === "production"
      ? "Internal server error"
      : err.message
  });
}
```

```js
// routes/users.js

app.get("/users/:id", async (req, res, next) => {
  try {
    const user = await db.findUserById(req.params.id);
    if (!user) throw new NotFoundError("User");
    res.json(user);
  } catch (e) {
    next(e); // all errors flow to errorHandler
  }
});

app.post("/users", async (req, res, next) => {
  try {
    const errors = validateUser(req.body);
    if (Object.keys(errors).length) throw new ValidationError(errors);

    const user = await db.createUser(req.body);
    res.status(201).json(user);
  } catch (e) {
    next(e);
  }
});
```

### Example 2 — Domain errors for a payment system

```js
class PaymentError extends AppError {
  constructor(message, code, options) {
    super(message, options);
    this.code = code; // machine-readable code for clients
  }
}

class InsufficientFundsError extends PaymentError {
  constructor(available, required) {
    super(
      `Insufficient funds: available ${available}, required ${required}`,
      "INSUFFICIENT_FUNDS"
    );
    this.available = available;
    this.required  = required;
    this.statusCode = 402;
  }
}

class CardDeclinedError extends PaymentError {
  constructor(reason) {
    super(`Card declined: ${reason}`, "CARD_DECLINED");
    this.reason     = reason;
    this.statusCode = 402;
  }
}

class PaymentProviderError extends PaymentError {
  constructor(provider, message, options) {
    super(`${provider} error: ${message}`, "PROVIDER_ERROR", options);
    this.provider   = provider;
    this.statusCode = 502;
    this.retryable  = true;
  }
}

// Usage
async function processPayment(amount, card) {
  try {
    const result = await stripeClient.charge({ amount, card });
    return result;
  } catch (e) {
    if (e.type === "StripeCardError") {
      throw new CardDeclinedError(e.decline_code);
    }
    throw new PaymentProviderError("Stripe", e.message, { cause: e });
  }
}

// Route handler
try {
  await processPayment(amount, cardToken);
} catch (e) {
  if (e instanceof InsufficientFundsError) {
    return res.status(402).json({
      error: e.message,
      code: e.code,
      shortfall: e.required - e.available
    });
  }
  if (e instanceof CardDeclinedError) {
    return res.status(402).json({ error: e.message, code: e.code });
  }
  if (e instanceof PaymentProviderError && e.retryable) {
    return res.status(503).json({ error: "Payment service unavailable, please retry" });
  }
  next(e);
}
```

### Example 3 — Fetch wrapper with typed errors

```js
class NetworkError extends AppError {
  constructor(url, options) {
    super(`Network request failed: ${url}`, options);
    this.url       = url;
    this.retryable = true;
  }
}

class HttpResponseError extends AppError {
  constructor(statusCode, body, url) {
    super(`HTTP ${statusCode} from ${url}`);
    this.statusCode = statusCode;
    this.body       = body;
    this.url        = url;
    this.retryable  = statusCode >= 500;
  }
}

class ParseError extends AppError {
  constructor(url, options) {
    super(`Failed to parse response from ${url}`, options);
    this.url = url;
  }
}

async function apiFetch(url, init = {}) {
  let res;
  try {
    res = await fetch(url, init);
  } catch (e) {
    throw new NetworkError(url, { cause: e });
  }

  if (!res.ok) {
    const body = await res.json().catch(() => ({}));
    throw new HttpResponseError(res.status, body, url);
  }

  try {
    return await res.json();
  } catch (e) {
    throw new ParseError(url, { cause: e });
  }
}

// Caller gets typed errors to handle precisely
try {
  const user = await apiFetch("/api/users/42");
} catch (e) {
  if (e instanceof NetworkError)       showOfflineBanner();
  else if (e instanceof HttpResponseError && e.statusCode === 404) showNotFound();
  else if (e instanceof HttpResponseError && e.retryable)  scheduleRetry();
  else if (e instanceof ParseError)    reportBug(e);
  else throw e;
}
```

### Example 4 — Validation error with field-level detail

```js
class ValidationError extends AppError {
  constructor(errors) {
    // errors = { fieldName: ["error message", ...] }
    const count   = Object.keys(errors).length;
    const message = `Validation failed: ${count} field${count > 1 ? "s" : ""} invalid`;
    super(message);
    this.statusCode = 422;
    this.errors     = errors;
  }

  // Helper — check if a specific field has errors
  hasField(name) {
    return name in this.errors;
  }

  // Helper — get all messages as a flat array
  get allMessages() {
    return Object.values(this.errors).flat();
  }
}

// Validator that accumulates errors
function validateRegistration(data) {
  const errors = {};

  if (!data.username || data.username.length < 3)
    errors.username = ["Minimum 3 characters"];

  if (!data.email || !/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(data.email))
    errors.email = ["Must be a valid email address"];

  if (!data.password || data.password.length < 8)
    errors.password = ["Minimum 8 characters"];

  if (data.password && !/[A-Z]/.test(data.password))
    (errors.password ??= []).push("Must contain at least one uppercase letter");

  if (Object.keys(errors).length > 0) {
    throw new ValidationError(errors);
  }
}

// Usage
try {
  validateRegistration(req.body);
} catch (e) {
  if (e instanceof ValidationError) {
    res.status(422).json({
      message: e.message,
      errors:  e.errors,        // field-level detail
      count:   e.allMessages.length
    });
  }
}
```

---

## 8. Checking Error Type Across Module Boundaries

When working with multiple files or packages, `instanceof` is the cleanest approach — but only if both sides import from the same class definition.

```js
// ✅ Works — same class reference
import { ValidationError } from "./errors.js";
if (e instanceof ValidationError) { ... }

// ⚠️ Can fail — if errors.js is somehow imported twice (e.g., CJS vs ESM mix)
// Use e.name as a fallback
if (e.name === "ValidationError") { ... }

// ✅ Most robust — check both
function isValidationError(e) {
  return e instanceof ValidationError || e.name === "ValidationError";
}
```

---

## 9. Serialising Errors (for logging / API responses)

By default, `JSON.stringify(new Error("oops"))` returns `"{}"` — all Error properties are non-enumerable.

```js
// ❌ JSON.stringify loses everything
JSON.stringify(new ValidationError({ email: ["invalid"] })); // "{}"

// ✅ Add a toJSON method to your base class
class AppError extends Error {
  constructor(message, options = {}) {
    super(message, options);
    this.name = this.constructor.name;
  }

  toJSON() {
    return {
      name:    this.name,
      message: this.message,
      // include subclass properties dynamically
      ...Object.fromEntries(
        Object.entries(this).filter(([k]) => !["stack"].includes(k))
      )
    };
  }
}

// Now JSON.stringify works
JSON.stringify(new ValidationError({ email: ["invalid"] }));
// '{"name":"ValidationError","message":"...","statusCode":422,"errors":{...}}'

// For logging with pino/winston:
logger.error({ err: e.toJSON() }, "Request failed");
```

---

## 10. Tricky Things

### 1. `e.name` is "Error" unless you set it

```js
class MyError extends Error {}
new MyError("test").name; // "Error" — not "MyError"!

// Fix:
class MyError extends Error {
  constructor(m, o) { super(m, o); this.name = this.constructor.name; }
}
new MyError("test").name; // "MyError"
```

### 2. `instanceof` fails if the class is defined in two different modules

```js
// If two different bundles or module instances each define ValidationError,
// an error from one won't pass instanceof check from the other.
// Solution: use a shared package / singleton module for your error classes.
```

### 3. Don't forget to pass `options` to `super` — it carries `cause`

```js
// ❌ cause is lost
class MyError extends Error {
  constructor(message) {
    super(message); // options not passed
  }
}

// ✅
class MyError extends Error {
  constructor(message, options) {
    super(message, options); // { cause } flows through
  }
}

throw new MyError("wrapped", { cause: originalErr });
// myErr.cause === originalErr ✅
```

### 4. Stack trace points to `super()` call — not the throw site

In V8 (Node.js/Chrome), the stack trace is captured at `new Error()` time. With custom errors, the top frame might be inside your constructor. This is usually fine — the throw site appears a few frames down — but it's worth knowing when reading traces.

### 5. Custom properties must be set after `super()` — not before

```js
class MyError extends Error {
  constructor(message, code) {
    this.code = code; // ❌ ReferenceError: can't access 'this' before super()
    super(message);
  }
}

class MyError extends Error {
  constructor(message, code) {
    super(message);   // ✅
    this.code = code; // ✅
  }
}
```

---

## 11. Common Mistakes

### Mistake 1 — Not setting `this.name`

```js
// ❌ Logs as "Error: user not found" — type information lost in production logs
class NotFoundError extends Error {
  constructor(r) { super(`${r} not found`); }
}

// ✅
class NotFoundError extends Error {
  constructor(r) {
    super(`${r} not found`);
    this.name = this.constructor.name; // "NotFoundError"
  }
}
```

### Mistake 2 — Forgetting to re-throw unknown errors

```js
// ❌ Swallows all errors including unexpected bugs
catch (e) {
  if (e instanceof ValidationError) handleValidation(e);
  // everything else silently caught
}

// ✅
catch (e) {
  if (e instanceof ValidationError) handleValidation(e);
  else throw e;
}
```

### Mistake 3 — Putting business logic in constructors

```js
// ❌ Constructor does DB work — errors are hard to trace, constructors should be simple
class UserError extends Error {
  constructor(userId) {
    const user = db.getUserSync(userId); // ❌ sync DB call in constructor
    super(`User ${user.name} is invalid`);
  }
}

// ✅ Factory function or static method
class UserError extends Error {
  constructor(message, userId) {
    super(message);
    this.name   = this.constructor.name;
    this.userId = userId;
  }

  static async fromId(userId) {
    const user = await db.getUser(userId);
    return new UserError(`User ${user.name} is invalid`, userId);
  }
}
```

### Mistake 4 — Using error message strings for branching in `catch`

```js
// ❌ Fragile — message strings change, engines differ, internationalisation breaks this
catch (e) {
  if (e.message.includes("not found")) handle404();
  if (e.message.includes("unauthorized")) handle401();
}

// ✅ Use instanceof or a code property
catch (e) {
  if (e instanceof NotFoundError)    handle404();
  if (e instanceof UnauthorizedError) handle401();
}
```

### Mistake 5 — Deeply nested custom errors nobody imports

```js
// ❌ If every file defines its own local error classes, instanceof checks
// across module boundaries fail silently

// ✅ Define all shared error classes in ONE central file
// src/errors.js or src/lib/errors.js
// and import from there everywhere
```

---

## 12. Practice Exercises

### Easy — custom error with code

Create a `AppError` base class and a `NotFoundError` subclass. `NotFoundError` should:
- Accept a `resource` parameter (e.g., `"User"`, `"Post"`)
- Set `this.statusCode = 404`
- Set a message like `"User not found"`
- Set `this.name` correctly

Write a function `findUser(id)` that throws `NotFoundError` when `id > 100`. Call it and log `err.name`, `err.message`, `err.statusCode`.

---

### Medium — validation error accumulator

Build a `ValidationError` class that:
- Accepts `fields` object: `{ email: ["Invalid format"], password: ["Too short", "Needs uppercase"] }`
- Generates a `.message` automatically: `"2 fields failed validation"`
- Has a `.hasField(name)` method
- Has an `.allMessages` getter returning a flat array of all error strings
- Has a `toJSON()` method

Write a `validateSignup(data)` function that throws `ValidationError` for invalid inputs. Test with both valid and invalid data.

---

### Hard — full error hierarchy for a REST API

Design and implement an error hierarchy for a blogging API:

```
AppError
├── HttpError (statusCode)
│   ├── BadRequestError (400)
│   ├── UnauthorizedError (401)
│   ├── ForbiddenError (403)
│   ├── NotFoundError (404)
│   └── RateLimitError (429, retryAfter property)
├── ValidationError (422, fields object)
└── InfrastructureError (retryable boolean)
    ├── DatabaseError
    └── CacheError
```

Requirements:
- All classes set `this.name` correctly
- All classes pass `options` to `super` (for `cause` support)
- Write an `errorHandler(err)` function (simulating Express middleware) that returns `{ statusCode, body }` for each type
- Write 8 test assertions (plain `if` + `console.assert`) verifying the right status codes and body shapes

---

## 13. Quick Reference

```
BASE PATTERN
  class MyError extends Error {
    constructor(message, options) {
      super(message, options);   // sets .message, .stack, .cause
      this.name = this.constructor.name; // sets .name to class name
      // add custom properties here
    }
  }

PROPERTIES TO SET
  this.name           — class name (required for useful logs)
  this.statusCode     — HTTP status code (useful in Express)
  this.code           — machine-readable string ("INSUFFICIENT_FUNDS")
  this.fields         — field-level validation errors
  this.retryable      — boolean for retry logic

WITH CAUSE (ES2022)
  throw new MyError("wrapper msg", { cause: originalError });
  err.cause   → original error + its stack

INSTANCEOF CHAIN
  new NotFoundError() instanceof NotFoundError  // true
  new NotFoundError() instanceof HttpError      // true
  new NotFoundError() instanceof AppError       // true
  new NotFoundError() instanceof Error          // true

SERIALIZING
  JSON.stringify(new Error())       // "{}" — loses everything
  Add toJSON() to base class to fix

CROSS-MODULE
  instanceof works when all errors come from ONE shared module
  e.name === "ClassName" works regardless of where class was defined

HIERARCHY TIP
  AppError → HttpError → specific (404, 401, 422…)
  catch (e instanceof HttpError) catches all HTTP errors
  catch (e instanceof AppError) catches everything you own
  re-throw anything that's neither — those are bugs
```

---

## 14. Connected Topics

- **71 — Types of JS errors** — the built-in error types your custom errors sit alongside
- **72 — try/catch/finally** — the mechanics of catching, branching on type, and re-throwing
- **58 — Fetch API** — `NetworkError`, `HttpResponseError`, `ParseError` — typed fetch errors
- **52 — Async JS** — error propagation through promise chains and async/await
- **Node.js / Express** — error middleware, `next(err)`, mapping `statusCode` to HTTP responses
- **Mongoose / Sequelize** — `.ValidationError`, `.CastError` — examples of this pattern in the wild
