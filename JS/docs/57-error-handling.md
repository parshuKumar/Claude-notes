# 57 — Error Handling

## What is this?

Error handling is the practice of **anticipating, catching, and responding to failures in your code** in a controlled way instead of letting them crash your program. JavaScript provides `try/catch/finally` as the primary mechanism: you put code that might fail in a `try` block, catch any thrown errors in a `catch` block, and use `finally` for cleanup that must always run. Think of it like working in a kitchen: you try to cook the dish (`try`), if something goes wrong you handle it gracefully (`catch` — maybe you use a backup ingredient or tell the customer), and no matter what, you always clean up your workstation afterward (`finally`).

## Why does it matter?

In backend development, errors are not exceptional — they are constant. Network requests fail, databases go down, users send invalid input, third-party APIs return unexpected data, file paths don't exist. A server that crashes on the first unhandled error is useless. Proper error handling means your server stays up, gives clients meaningful error messages, logs failures for debugging, and recovers gracefully. Understanding the full error handling model — synchronous errors, async errors, custom error types, and the difference between operational vs programmer errors — is what separates junior from senior backend developers.

---

## Syntax

```js
try {
  // Code that might throw
  const data = JSON.parse(rawInput);       // throws SyntaxError if invalid JSON
  const user = findUser(data.userId);      // might throw if not found
  console.log("Success:", user.name);
} catch (err) {
  // Runs if anything in 'try' throws
  // 'err' is whatever was thrown (usually an Error object)
  console.error("Something failed:", err.message);
} finally {
  // Runs ALWAYS — whether try succeeded or catch ran
  // Used for cleanup: close connections, release locks, hide spinners
  console.log("Cleanup done");
}
```

---

## How it works — line by line

```js
try {
```
JS enters the try block. If any line throws, execution jumps immediately to `catch`. Lines after the throw in the `try` block do NOT run.

```js
} catch (err) {
```
`err` is the thrown value. Usually an `Error` object, but JavaScript lets you throw anything — a string, number, or plain object. In practice, always throw `Error` instances.

```js
} finally {
```
Runs **no matter what**: if try completed normally, if catch ran, even if there's a `return` or `throw` inside try/catch. The only exception: `process.exit()` bypasses finally.

---

## What can be thrown

```js
throw "a string";            // valid but bad practice — no stack trace
throw 42;                    // valid but useless
throw { code: 500 };         // valid but non-standard
throw new Error("message");  // CORRECT — has message, stack, name
throw new TypeError("...");  // CORRECT — specific built-in error type
throw new CustomError("..."); // CORRECT — custom class extending Error
```

**Always throw `Error` instances** — they capture the stack trace at the point of creation, which is essential for debugging. Throwing primitives loses all stack information.

---

## The Error object

```js
const err = new Error("Something went wrong");

console.log(err.message);  // "Something went wrong"
console.log(err.name);     // "Error"
console.log(err.stack);    // Full stack trace: "Error: Something went wrong\n  at ..."

// You can add custom properties:
const err2 = new Error("User not found");
err2.statusCode = 404;
err2.userId     = 123;
```

---

## Built-in error types

JavaScript has several built-in Error subclasses. Always use the most specific one:

```js
// TypeError — wrong type
new TypeError("Expected a string, got number");
// Example: accessing .length on undefined

// ReferenceError — using undeclared variable
// Example: console.log(undeclaredVar)

// SyntaxError — invalid code (usually thrown by the parser, not you)
// Example: JSON.parse("{invalid}")

// RangeError — value outside allowed range
new RangeError("Array length must be between 0 and 2^32 - 1");
// Example: new Array(-1)

// URIError — malformed URI
// Example: decodeURIComponent('%')

// EvalError — related to eval() — essentially never seen in modern code

// Using the right type helps catch blocks distinguish error kinds:
try {
  JSON.parse(rawInput);
} catch (err) {
  if (err instanceof SyntaxError) {
    console.error("Input is not valid JSON");
  } else {
    throw err;  // re-throw unexpected errors
  }
}
```

---

## Custom error classes — the professional standard

Extending `Error` lets you create semantic, catchable error types:

```js
// Base custom error
class AppError extends Error {
  constructor(message, statusCode = 500) {
    super(message);                      // sets this.message
    this.name        = this.constructor.name;  // "AppError" (not "Error")
    this.statusCode  = statusCode;
    this.isOperational = true;           // flag: expected operational error (not a bug)

    // Maintains proper stack trace (V8 only)
    if (Error.captureStackTrace) {
      Error.captureStackTrace(this, this.constructor);
    }
  }
}

// Specific error types
class NotFoundError extends AppError {
  constructor(resource, id) {
    super(`${resource} with id ${id} not found`, 404);
    this.resource = resource;
    this.id       = id;
  }
}

class ValidationError extends AppError {
  constructor(message, fields = {}) {
    super(message, 400);
    this.fields = fields;  // { email: "Invalid format", password: "Too short" }
  }
}

class UnauthorizedError extends AppError {
  constructor(message = "Authentication required") {
    super(message, 401);
  }
}

class ForbiddenError extends AppError {
  constructor(message = "Insufficient permissions") {
    super(message, 403);
  }
}

class DatabaseError extends AppError {
  constructor(message, originalError) {
    super(message, 500);
    this.isOperational = false;  // DB errors = programmer error or infrastructure issue
    this.originalError = originalError;
  }
}

// Using them:
function findUser(id) {
  const user = db.query(`SELECT * FROM users WHERE id = ${id}`);
  if (!user) throw new NotFoundError("User", id);
  return user;
}

function createUser(data) {
  const errors = {};
  if (!data.email?.includes("@")) errors.email = "Invalid email format";
  if (data.password?.length < 8)  errors.password = "Password must be at least 8 characters";
  if (Object.keys(errors).length > 0) {
    throw new ValidationError("Validation failed", errors);
  }
  // ... create user
}
```

---

## Example 1 — basic: sync error handling

```js
function divide(a, b) {
  if (typeof a !== "number" || typeof b !== "number") {
    throw new TypeError(`Expected numbers, got ${typeof a} and ${typeof b}`);
  }
  if (b === 0) {
    throw new RangeError("Cannot divide by zero");
  }
  return a / b;
}

function calculateAverage(values) {
  if (!Array.isArray(values) || values.length === 0) {
    throw new TypeError("Values must be a non-empty array");
  }
  const sum = values.reduce((acc, val) => {
    if (typeof val !== "number") throw new TypeError(`Non-numeric value: ${val}`);
    return acc + val;
  }, 0);
  return divide(sum, values.length);
}

// Catching specific error types:
try {
  const avg = calculateAverage([10, 20, "thirty", 40]);
} catch (err) {
  if (err instanceof TypeError) {
    console.error("Type error:", err.message);     // "Non-numeric value: thirty"
  } else if (err instanceof RangeError) {
    console.error("Range error:", err.message);
  } else {
    console.error("Unknown error:", err);
    throw err;  // re-throw unexpected errors
  }
}
```

---

## Example 2 — async error handling: the complete picture

```js
// Async function with comprehensive error handling
async function processUserRegistration(userData) {
  // Step 1: validate (sync — throws synchronously)
  try {
    validateRegistrationData(userData);
  } catch (err) {
    if (err instanceof ValidationError) {
      return { success: false, statusCode: 400, errors: err.fields };
    }
    throw err;  // unexpected validation failure — re-throw
  }

  // Step 2: check email uniqueness (async)
  let existingUser;
  try {
    existingUser = await db.findUserByEmail(userData.email);
  } catch (err) {
    // DB failure — wrap in our custom error type
    throw new DatabaseError("Failed to check email uniqueness", err);
  }

  if (existingUser) {
    throw new ValidationError("Email already taken", { email: "Already in use" });
  }

  // Step 3: hash password (async, should not fail but guard anyway)
  let hashedPassword;
  try {
    hashedPassword = await bcrypt.hash(userData.password, 10);
  } catch (err) {
    throw new AppError("Password processing failed", 500);
  }

  // Step 4: save to DB
  let user;
  try {
    user = await db.createUser({
      ...userData,
      password: hashedPassword,
    });
  } catch (err) {
    throw new DatabaseError("Failed to create user", err);
  }

  // Step 5: send welcome email (non-critical — log but don't fail)
  sendWelcomeEmail(user.email, user.name).catch(emailErr => {
    logger.warn("Welcome email failed:", {
      userId: user.id,
      error:  emailErr.message,
    });
  });

  return { success: true, userId: user.id };
}
```

---

## Example 3 — real world: Express.js error middleware

```js
const express = require("express");
const app     = express();

// Route — throws custom errors
app.get("/users/:id", async (req, res, next) => {
  try {
    const userId = parseInt(req.params.id, 10);
    if (isNaN(userId)) throw new ValidationError("Invalid user ID");

    const user = await UserService.findById(userId);
    res.json({ user });

  } catch (err) {
    next(err);   // pass ALL errors to error middleware
  }
});

// Centralised error handler — MUST have 4 parameters
app.use((err, req, res, next) => {
  // Log the error
  logger.error({
    message: err.message,
    stack:   err.stack,
    url:     req.url,
    method:  req.method,
    userId:  req.user?.id,
  });

  // Handle operational errors (expected, known errors)
  if (err instanceof AppError && err.isOperational) {
    return res.status(err.statusCode).json({
      error:   err.name,
      message: err.message,
      ...(err instanceof ValidationError && { fields: err.fields }),
    });
  }

  // Handle unexpected programmer errors — return generic message
  // (don't expose internal details to the client)
  return res.status(500).json({
    error:   "InternalServerError",
    message: "An unexpected error occurred",
    ...(process.env.NODE_ENV === "development" && { stack: err.stack }),
  });
});
```

---

## `finally` in depth

```js
// finally always runs — even through return, throw, break, continue

function example() {
  try {
    return "from try";     // finally STILL runs before function returns
  } finally {
    console.log("finally ran");   // prints
    // If you return here, it OVERRIDES the try's return:
    // return "from finally";    // ← this would override "from try"
  }
}

console.log(example());
// "finally ran"
// "from try"

// --- 

// throw in finally overrides throw in catch:
function dangerous() {
  try {
    throw new Error("original");
  } catch (err) {
    console.log("caught:", err.message);   // "original"
    throw err;                              // re-throwing original
  } finally {
    throw new Error("finally error");       // OVERRIDES the re-throw!
    // original error is LOST — this is almost always a bug
  }
}
dangerous().catch(e => console.log(e.message));  // "finally error"

// --- 

// Practical finally use: resource cleanup
async function withConnection(operationFn) {
  const conn = await pool.getConnection();
  try {
    const result = await operationFn(conn);
    return result;
  } catch (err) {
    await conn.rollback();   // rollback on error
    throw err;               // re-throw after rollback
  } finally {
    conn.release();          // ALWAYS release — no matter what happened
  }
}
```

---

## Error handling in async/await — every scenario

### Scenario 1: `await` a rejected Promise — throws in the current scope

```js
async function main() {
  try {
    const data = await fetchData();   // if fetchData rejects, throws here
    process(data);                     // only runs if fetchData resolved
  } catch (err) {
    console.error("fetchData failed:", err.message);
  }
}
```

### Scenario 2: Catching per-step with fallbacks

```js
async function loadWithFallback(userId) {
  const user = await fetchUser(userId).catch(() => ({
    id:   userId,
    name: "Anonymous",
    role: "guest",
  }));

  const settings = await fetchSettings(userId).catch(() => defaultSettings);

  return { user, settings };
}
```

### Scenario 3: Re-throwing with enriched context

```js
async function getUserOrders(userId) {
  try {
    const orders = await db.query("SELECT * FROM orders WHERE user_id = $1", [userId]);
    return orders.rows;
  } catch (err) {
    // Add context before re-throwing
    const enriched = new DatabaseError(
      `Failed to fetch orders for user ${userId}`,
      err
    );
    throw enriched;
  }
}
```

### Scenario 4: `error.cause` — the modern way to chain errors (ES2022)

```js
// ES2022: Error cause — preserves the original error without losing it
async function fetchUserData(userId) {
  try {
    const response = await fetch(`/api/users/${userId}`);
    return await response.json();
  } catch (originalErr) {
    throw new Error(`Failed to fetch user ${userId}`, { cause: originalErr });
  }
}

// Accessing the cause:
try {
  await fetchUserData(5);
} catch (err) {
  console.log(err.message);          // "Failed to fetch user 5"
  console.log(err.cause.message);    // "Failed to fetch" — original error preserved
  console.log(err.cause.stack);      // original stack trace
}
```

### Scenario 5: Handling `Promise.all` failures

```js
async function loadAll(ids) {
  try {
    const results = await Promise.all(ids.map(id => fetchItem(id)));
    return results;
  } catch (err) {
    // Which item failed? Not obvious — Promise.all only gives first error
    console.error("One of the fetches failed:", err.message);
    throw err;
  }
}

// Better with allSettled when you want to know which one failed:
async function loadAllRobust(ids) {
  const results = await Promise.allSettled(ids.map(id => fetchItem(id)));

  const failed = results
    .map((r, i) => r.status === "rejected" ? { id: ids[i], error: r.reason.message } : null)
    .filter(Boolean);

  if (failed.length > 0) {
    logger.warn("Some items failed to load:", failed);
  }

  return results
    .filter(r => r.status === "fulfilled")
    .map(r => r.value);
}
```

---

## Operational errors vs programmer errors

This distinction is critical for production Node.js services:

### Operational errors — expected, recoverable
- User sends invalid JSON → `SyntaxError`
- User not found in DB → `NotFoundError`
- Third-party API returns 503 → connection error
- Disk is full → `ENOSPC`
- **Correct response**: log at `warn` level, send error to client, continue serving

### Programmer errors — bugs, unrecoverable
- Calling `.map()` on `undefined`
- Accessing property of `null`
- Using wrong function signature
- Off-by-one errors
- **Correct response**: log at `error` level, crash and restart (PM2/k8s will restart the process), alert on-call

```js
// How to tell them apart in practice:
function handleError(err) {
  if (err instanceof AppError && err.isOperational) {
    // Operational — handle gracefully
    logger.warn("Operational error:", { message: err.message, code: err.statusCode });
    return;
  }

  // Programmer error — log everything, then crash
  logger.error("Programmer error — shutting down:", {
    message: err.message,
    stack:   err.stack,
  });

  // Graceful shutdown: stop accepting new requests, finish in-flight, then exit
  server.close(() => {
    process.exit(1);   // non-zero exit triggers process manager restart
  });
}

process.on("uncaughtException",  handleError);
process.on("unhandledRejection", (reason) => handleError(reason));
```

---

## Global error handlers in Node.js

```js
// Catch synchronous errors that escape all try/catch blocks
process.on("uncaughtException", (err) => {
  logger.error("Uncaught exception:", err);
  // IMPORTANT: application state may be corrupted — shut down and restart
  process.exit(1);
});

// Catch Promise rejections with no .catch() handler
process.on("unhandledRejection", (reason, promise) => {
  logger.error("Unhandled rejection at:", promise, "reason:", reason);
  // In Node.js 15+, this crashes the process automatically
  // In older Node — add your own handling here
});

// Graceful shutdown on SIGTERM (Kubernetes, Docker stop)
process.on("SIGTERM", async () => {
  logger.info("SIGTERM received — starting graceful shutdown");
  server.close(() => {
    logger.info("HTTP server closed");
    db.disconnect().then(() => {
      logger.info("DB disconnected");
      process.exit(0);
    });
  });
});
```

---

## Tricky things

### 1. `catch` without re-throw silently swallows errors

```js
// DANGEROUS — hides the error completely
async function saveData(data) {
  try {
    await db.save(data);
  } catch (err) {
    console.error(err);   // logged, but...
    // No throw — function resolves with undefined
    // Caller has no idea the save failed!
  }
}

const result = await saveData(data);
// result = undefined — was that success or failure?

// RIGHT:
async function saveData(data) {
  try {
    return await db.save(data);
  } catch (err) {
    logger.error("DB save failed:", err);
    throw new DatabaseError("Failed to persist data", err);
  }
}
```

### 2. `try/catch` does NOT catch async errors without `await`

```js
async function broken() {
  try {
    fetchData();   // missing await — Promise returned but not awaited
    // If fetchData() rejects, this try/catch DOES NOT CATCH IT
  } catch (err) {
    console.log("this never runs for fetchData errors");
  }
}

// Fix:
async function correct() {
  try {
    await fetchData();   // awaited — rejection becomes a throw
  } catch (err) {
    console.log("caught!");
  }
}
```

### 3. `finally` returning a value overrides try/catch returns

```js
function test() {
  try {
    return 1;
  } finally {
    return 2;   // overrides — function returns 2, not 1
  }
}
console.log(test());   // 2  — subtle and usually a bug

// Same with throw:
function test2() {
  try {
    throw new Error("from try");
  } finally {
    return "from finally";   // swallows the error! Function RESOLVES with "from finally"
  }
}
console.log(test2());   // "from finally"  — the error is GONE
```

**Rule:** Never `return` or `throw` from `finally` unless you have an explicit reason.

### 4. `instanceof` checks fail across module boundaries

```js
// module A defines AppError
// module B also defines AppError (different file, same code)
// Objects created by A's AppError are NOT instanceof B's AppError

// This is why you should:
// 1. Always use a single shared error module
// 2. Use duck-typing as fallback: if (err.isOperational) {...}
```

### 5. Error properties are not automatically serialised

```js
const err = new ValidationError("Failed", { email: "Invalid" });

console.log(JSON.stringify(err));
// "{}"  — Error properties are non-enumerable by default!

// Fix — custom toJSON:
class AppError extends Error {
  toJSON() {
    return {
      name:       this.name,
      message:    this.message,
      statusCode: this.statusCode,
      ...(this.fields && { fields: this.fields }),
    };
  }
}

console.log(JSON.stringify(new ValidationError("Failed", { email: "Invalid" })));
// '{"name":"ValidationError","message":"Failed","statusCode":400,"fields":{"email":"Invalid"}}'
```

### 6. Async errors in `setTimeout` are not caught by outer try/catch

```js
// WRONG — try/catch cannot reach into a different tick of the event loop
async function wrong() {
  try {
    setTimeout(() => {
      throw new Error("from timeout");  // NOT caught — different tick
    }, 100);
  } catch (err) {
    console.log("never runs");
  }
}

// RIGHT — wrap the async operation properly
function delay(ms) {
  return new Promise(resolve => setTimeout(resolve, ms));
}

async function correct() {
  try {
    await delay(100);
    throw new Error("from await");    // caught — same async flow
  } catch (err) {
    console.log("caught:", err.message);
  }
}
```

---

## Common mistakes

### Mistake 1 — Empty catch block

```js
// WRONG — error completely hidden
try {
  riskyOperation();
} catch (err) {
  // nothing — "error swallowing"
}

// RIGHT — at minimum log it; better to also re-throw or handle
try {
  riskyOperation();
} catch (err) {
  logger.error("riskyOperation failed:", err);
  throw err;   // let the caller decide what to do
}
```

### Mistake 2 — Catching `Error` when you should re-throw specific types

```js
// WRONG — catches ALL errors, including programmer bugs
try {
  processData(userInput);
} catch (err) {
  res.status(400).json({ error: "Bad request" });
  // What if err was a TypeError from your own bug? Now it looks like user error!
}

// RIGHT — check the error type
try {
  processData(userInput);
} catch (err) {
  if (err instanceof ValidationError) {
    return res.status(400).json({ error: err.message, fields: err.fields });
  }
  // Re-throw anything else — it's a programmer error, not user error
  throw err;
}
```

### Mistake 3 — Using `try/catch` for control flow

```js
// WRONG — using exceptions for normal program logic (slow, hard to read)
function findUser(id) {
  try {
    return db.mustFindUser(id);  // throws if not found
  } catch (err) {
    return null;  // using exception for "not found" flow
  }
}

// RIGHT — return null/undefined for expected "not found"; throw only for failures
function findUser(id) {
  return db.findUser(id) ?? null;  // returns null if not found — no exception
}
```

---

## Practice exercises

### Exercise 1 — easy

Create the following custom error classes, all extending a base `AppError`:
- `NotFoundError(resource, id)` — status 404
- `ValidationError(message, fields)` — status 400
- `UnauthorizedError(message)` — status 401

Write a function `getProduct(id, userRole)` that:
- Throws `ValidationError` if `id` is not a positive integer
- Throws `UnauthorizedError` if `userRole` is `"guest"`
- Throws `NotFoundError` if `id` is greater than 100 (simulate "not in DB")
- Returns a mock product object otherwise

Wrap calls to `getProduct` in a handler that responds differently for each error type.

```js
// Write your code here
```

### Exercise 2 — medium

Build a `SafeJSON` utility class with all-static methods that never throw:
- `static parse(str, fallback = null)` — parses JSON; returns `fallback` on any error
- `static stringify(value, fallback = "null")` — stringifies; returns `fallback` on error
- `static parseFile(path)` — async; reads file and parses JSON; returns `{ data, error }` tuple (never rejects)
- `static validate(str, schema)` — `schema` is an object of `{ key: typeName }` pairs; returns `{ valid: bool, errors: [] }`

Also write a `withErrorContext(label, fn)` wrapper — a higher-order function that calls `fn()` and re-throws any error with a `context` property set to `label`.

```js
// Write your code here
```

### Exercise 3 — hard

Build a production-grade `ErrorHandler` for an Express.js-style application:

1. Create a hierarchy of error classes: `AppError` → `HttpError` → `NotFoundError`, `ValidationError`, `UnauthorizedError`, `ForbiddenError`, `ConflictError`. Each has appropriate HTTP status codes.

2. Create a `DatabaseError` that wraps a cause and has `isOperational: false`.

3. Write an `errorMiddleware(err, req, res, next)` function that:
   - For operational `AppError` instances: returns structured JSON with status code
   - For `ValidationError`: includes the `fields` object in the response
   - For non-operational or unexpected errors: returns generic 500, logs full details
   - In `development` mode: always includes the stack trace
   - Formats ALL errors consistently: `{ error: { type, message, ..extras } }`

4. Write an `asyncHandler(fn)` wrapper for route handlers.

5. Write a `setupGlobalHandlers(server, db)` function that registers `uncaughtException` and `unhandledRejection` handlers, each with proper logging and graceful shutdown (close server, disconnect DB, exit).

```js
// Write your code here
```

---

## Quick reference (cheat sheet)

| Concept | Detail |
|---|---|
| `try { } catch (err) { }` | Wraps risky code; catches any thrown value |
| `finally { }` | Runs always — even through return/throw; use for cleanup |
| Always throw `Error` | Preserves stack trace; never throw strings/numbers |
| `err.message` | Human-readable description |
| `err.name` | Error type name (e.g. `"TypeError"`) |
| `err.stack` | Full stack trace string |
| `err.cause` | (ES2022) original error: `throw new Error("msg", { cause: originalErr })` |
| Custom errors | `class MyError extends Error { constructor(...) { super(...); this.name = "MyError"; } }` |
| `instanceof` check | `if (err instanceof NotFoundError)` — specific handling |
| Re-throw | `throw err` inside catch — propagates to outer handler |
| Swallowing errors | Empty catch or catch without re-throw — almost always wrong |
| Async try/catch | `try { await fn() } catch (err) {}` — catches awaited rejections |
| `.catch()` vs try/catch | Equivalent; use try/catch with async/await for readability |
| Non-awaited in try | `fn()` without `await` inside try — rejection NOT caught |
| `finally` return | Overrides try/catch return — almost always a bug |
| `finally` async | `await` is allowed inside `finally` |
| Operational vs bug | Operational = expected (404, 400) → handle; Bug = unexpected → crash+restart |
| `uncaughtException` | `process.on("uncaughtException", handler)` — last resort |
| `unhandledRejection` | `process.on("unhandledRejection", handler)` — Node 15+ crashes anyway |
| Error in `setTimeout` | Not caught by outer try/catch — different event loop tick |

---

## Connected topics

- **56 — async/await** — `try/catch` with `await` is the primary error handling pattern for all async code. These two topics are used together constantly.
- **54 — Promises** — `.catch()` on Promise chains is the alternative to try/catch; understanding both lets you choose appropriately.
- **71 — Types of JS errors** — Covers SyntaxError, TypeError, ReferenceError, RangeError in depth — the built-in error taxonomy.
- **73 — Custom error classes** — Goes deeper into the error class hierarchy, serialisation, and advanced patterns touched on here.
