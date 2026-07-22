# 60 — Error handling in TypeScript

## What is this?

**Error handling in TypeScript** is the set of rules, types, and patterns you use to work with `throw`, `try`/`catch`, `Error` subclasses, and rejected promises *while keeping the type system honest*.

The single most important fact — and the one that surprises every JS developer on day one:

```ts
try {
  await chargeCard(userId, amountCents);
} catch (err) {
  // err is `unknown` — NOT `Error`
  err.message; // ❌ Error: 'err' is of type 'unknown'
}
```

TypeScript types the catch variable as `unknown` (under `useUnknownInCatchVariables`, which `strict` turns on). It does this because **JavaScript lets you throw literally anything**:

```js
throw new Error("boom");      // an Error
throw "boom";                 // a string
throw { code: 500 };          // a plain object
throw null;                   // null
throw undefined;              // undefined (yes, really)
```

So the compiler refuses to guess. Everything in this document flows from that one decision: since `catch` gives you `unknown`, you must **narrow** before you can use it — and once you accept that, you end up building typed error hierarchies, type guards, and typed error middleware, which is exactly what a production backend needs anyway.

---

## Why does it matter?

Backend code is mostly error handling. A single HTTP request touches a dozen failure modes:

- The request body is malformed → `400` with per-field validation messages
- The `authToken` is missing, expired, or forged → `401`
- The user is authenticated but lacks the role → `403`
- The `userId` in the path doesn't exist → `404`
- The email is already registered → `409`
- Postgres times out, Redis is down, Stripe returns a `502` → `503` and a page to on-call
- A `JSON.parse` blows up somewhere deep in a library → `500`, and you need the stack

If every one of those is a bare `new Error("something went wrong")`, your error handler is a pile of string matching (`if (err.message.includes("not found"))`), your status codes are guesses, and your logs are useless. Typed errors let you:

1. **Attach structured data** to a failure (`statusCode`, `code`, `fields`, `retryable`, `requestId`).
2. **Narrow safely** — `if (err instanceof ValidationError)` gives you `err.fields` with full autocomplete.
3. **Centralise the mapping** from error type → HTTP response in one exhaustively-checked place.
4. **Distinguish operational errors** (expected: bad input, missing row) from **programmer errors** (a bug: `undefined is not a function`) — the first you answer with a 4xx, the second you log loudly and return a 500.

---

## The JavaScript way vs the TypeScript way

### The JavaScript way — everything is a guess

```js
// JavaScript — the catch variable is whatever it is. You hope it's an Error.
async function handleCreateUser(req, res) {
  try {
    const user = await userService.create(req.body);
    res.status(201).json(user);
  } catch (err) {
    // Guess #1: does err have .message? Usually. Not always.
    console.error(err.message);          // TypeError if someone threw a string

    // Guess #2: string-matching to decide the status code. Fragile forever.
    if (err.message.includes("duplicate")) {
      return res.status(409).json({ error: "Email already registered" });
    }
    if (err.message.includes("invalid")) {
      return res.status(400).json({ error: err.message });
    }

    // Guess #3: maybe the ORM attached a code? Maybe it's a number? A string?
    if (err.code === "23505") {          // Postgres unique_violation, if you remember it
      return res.status(409).json({ error: "Conflict" });
    }

    // Guess #4: err.fields might be there. Might be undefined. Might be a string.
    if (err.fields) {
      return res.status(422).json({ fields: err.fields });
    }

    res.status(500).json({ error: "Internal server error" });
  }
}

// And when a library does `throw "TIMEOUT"` (real libraries do this),
// `err.message` is undefined and `err.message.includes` throws INSIDE your
// catch block — turning a handled 503 into an unhandled crash.
```

The pain: **nothing about the failure is in the type system.** The function signature `async function create(dto)` says nothing about the fact that it can throw three different shapes. You find out in production.

### The TypeScript way — errors are types you narrow

```ts
// ── One base class carrying the fields every operational error needs ─────────
class AppError extends Error {
  constructor(
    message: string,
    readonly statusCode: number,
    readonly code: string,
    readonly isOperational: boolean = true,
  ) {
    super(message);
    this.name = new.target.name;                  // "ValidationError", not "Error"
    Object.setPrototypeOf(this, new.target.prototype); // ES5 target fix (see Going deeper)
    Error.captureStackTrace?.(this, new.target);  // drop the constructor from the stack
  }
}

// ── Specific errors with their OWN typed payloads ────────────────────────────
class ValidationError extends AppError {
  constructor(readonly fields: Record<string, string>) {
    super("Request validation failed", 422, "VALIDATION_FAILED");
  }
}

class NotFoundError extends AppError {
  constructor(readonly resource: string, readonly resourceId: string | number) {
    super(`${resource} ${resourceId} not found`, 404, "NOT_FOUND");
  }
}

class ConflictError extends AppError {
  constructor(message: string, readonly conflictingField: string) {
    super(message, 409, "CONFLICT");
  }
}

// ── The handler: narrow, don't guess ─────────────────────────────────────────
async function handleCreateUser(req: Request, res: Response): Promise<void> {
  try {
    const user = await userService.create(req.body as CreateUserDto);
    res.status(201).json({ data: user });
  } catch (err: unknown) {
    if (err instanceof ValidationError) {
      // err.fields is Record<string, string> — guaranteed, autocompleted
      res.status(err.statusCode).json({ code: err.code, fields: err.fields });
      return;
    }
    if (err instanceof AppError) {
      // Covers NotFoundError, ConflictError, and every future subclass
      res.status(err.statusCode).json({ code: err.code, error: err.message });
      return;
    }
    // Anything else is a bug or a rogue throw — log the raw value, hide it from the client
    logger.error("Unhandled error in createUser", { err: normalizeError(err) });
    res.status(500).json({ code: "INTERNAL", error: "Internal server error" });
  }
}
```

The revelation: `err.fields` is not a hope, it is a *fact*, and the compiler proved it. And the `unknown` you resented at the start is what forced you to write the `else` branch that catches the rogue `throw "TIMEOUT"` instead of crashing on it.

---

## Syntax

```ts
// ── The catch variable is `unknown` under strict / useUnknownInCatchVariables ─
try {
  riskyOperation();
} catch (err) {           // err: unknown
  // err.message;         // ❌ compile error — must narrow first
}

// ── You may annotate it explicitly — but ONLY as `unknown` or `any` ──────────
try { riskyOperation(); } catch (err: unknown) { }  // ✅ allowed (and preferred)
try { riskyOperation(); } catch (err: any)     { }  // ✅ allowed (opts out of safety)
// try { } catch (err: Error) { }                   // ❌ error: catch clause variable
                                                    //    type must be 'any' or 'unknown'

// ── Narrowing with instanceof ────────────────────────────────────────────────
try { riskyOperation(); } catch (err) {
  if (err instanceof Error)      err.message;    // err: Error
  if (err instanceof TypeError)  err.message;    // err: TypeError
  if (err instanceof AppError)   err.statusCode; // err: AppError
}

// ── Narrowing with typeof for primitives ────────────────────────────────────
try { riskyOperation(); } catch (err) {
  if (typeof err === "string") console.error(err.toUpperCase()); // err: string
}

// ── A user-defined type guard for non-class error shapes ────────────────────
function isNodeError(err: unknown): err is NodeJS.ErrnoException {
  return err instanceof Error && "code" in err;   // ENOENT, ECONNREFUSED, ...
}

// ── Custom error class — the four lines that matter ─────────────────────────
class AppError extends Error {
  constructor(message: string, readonly statusCode: number) {
    super(message);                                     // 1. set message
    this.name = "AppError";                             // 2. fix the name
    Object.setPrototypeOf(this, AppError.prototype);     // 3. fix instanceof on ES5
    Error.captureStackTrace?.(this, AppError);          // 4. clean stack (V8 only)
  }
}

// ── `cause` — chain the original error (ES2022 lib required) ────────────────
try {
  await db.query(sql);
} catch (err) {
  throw new AppError("Failed to load user", 503, { cause: err });
}

// ── never as the return type of a function that always throws ───────────────
function fail(message: string): never {
  throw new AppError(message, 500);
}

// ── Rejected promises land in the same catch ────────────────────────────────
async function load(userId: number): Promise<User> {
  try {
    return await userRepo.findById(userId);  // note: `await` inside try, not just `return`
  } catch (err) {                            // err: unknown
    throw new NotFoundError("User", userId);
  }
}
```

---

## How it works — concept by concept

### Concept 1 — Why `catch (err)` is `unknown`, and the flag that controls it

The compiler option is `useUnknownInCatchVariables`, enabled automatically by `"strict": true` since TypeScript 4.4.

```jsonc
// tsconfig.json
{
  "compilerOptions": {
    "strict": true,                        // turns useUnknownInCatchVariables on
    // "useUnknownInCatchVariables": false // ← opt out (do NOT do this)
  }
}
```

With it **off**, `err` is `any`, and this compiles and then explodes at runtime:

```ts
// useUnknownInCatchVariables: false
try {
  JSON.parse(requestBody);
} catch (err) {          // err: any
  logger.error(err.response.data.message);  // ✅ compiles, 💥 crashes — no .response
}
```

With it **on**, the compiler forces the question "what did you actually catch?":

```ts
// useUnknownInCatchVariables: true (the default under strict)
try {
  JSON.parse(requestBody);
} catch (err) {          // err: unknown
  // logger.error(err.response.data.message);  // ❌ won't compile — good
  logger.error("Failed to parse request body", { err: normalizeError(err) });
}
```

Why can't TypeScript just say `Error`? Because `throw` accepts any expression, and real code in the wild throws non-Errors constantly:

```ts
// All legal JavaScript — all end up in your catch block:
throw "connection lost";                  // string  (some older libraries)
throw 404;                                // number
throw { code: "RATE_LIMITED", retryAfter: 30 };  // plain object (many SDKs)
throw Symbol("nope");                     // symbol
throw null;                               // null — and `null instanceof Error` is false

// Promise rejections too:
Promise.reject("not an Error");           // rejects with a string
```

`unknown` is the only sound type for a value produced by a mechanism with no constraints.

The same rule applies to `.catch()` callbacks and to the promise rejection channel — though there the parameter is typed `any` for historical reasons:

```ts
somePromise.catch((err) => {
  // err is `any` here (the lib.d.ts signature says any) — treat it as unknown yourself:
  handle(err as unknown);
});
```

### Concept 2 — Narrowing an `unknown` error, from cheapest to most thorough

There is a ladder of techniques. Use the highest rung that works.

**Rung 1 — `instanceof` for your own classes (best):**

```ts
if (err instanceof ValidationError) {
  err.fields;      // ✅ Record<string, string>
  err.statusCode;  // ✅ 422 (inherited from AppError)
}
```

**Rung 2 — `instanceof Error` for anything Error-shaped:**

```ts
if (err instanceof Error) {
  err.message;  // ✅ string
  err.stack;    // ✅ string | undefined
  err.name;     // ✅ string
  err.cause;    // ✅ unknown  (requires "lib": ["ES2022"] or later)
}
```

**Rung 3 — structural type guards for objects you don't own** (third-party SDKs, cross-realm errors, `worker_threads` boundaries where `instanceof` fails):

```ts
// Shape check — works even when instanceof does not
function isErrorLike(err: unknown): err is { message: string; name: string } {
  return (
    typeof err === "object" &&
    err !== null &&
    "message" in err &&
    typeof (err as { message: unknown }).message === "string"
  );
}

// Node system errors: ENOENT, ECONNREFUSED, ETIMEDOUT, EADDRINUSE ...
function isErrnoException(err: unknown): err is NodeJS.ErrnoException {
  return err instanceof Error && typeof (err as NodeJS.ErrnoException).code === "string";
}

// A Postgres driver error (pg): has a 5-character SQLSTATE in `code`
interface PostgresError extends Error {
  code: string;         // "23505" = unique_violation, "23503" = foreign_key_violation
  detail?: string;
  constraint?: string;
  table?: string;
}
function isPostgresError(err: unknown): err is PostgresError {
  return err instanceof Error && /^[0-9A-Z]{5}$/.test((err as PostgresError).code ?? "");
}
```

**Rung 4 — a normalizer that always produces an `Error`**, so downstream logging code never has to think about it again:

```ts
/** Turn any thrown value into a real Error. Never throws, never returns undefined. */
function normalizeError(thrown: unknown): Error {
  if (thrown instanceof Error) return thrown;

  if (typeof thrown === "string") return new Error(thrown);

  if (isErrorLike(thrown)) {
    const normalized = new Error(thrown.message);
    normalized.name = thrown.name;
    return normalized;
  }

  // Last resort: serialize whatever it was, keeping the original as `cause`
  let serialized: string;
  try {
    serialized = JSON.stringify(thrown);
  } catch {
    serialized = String(thrown);          // handles circular refs, symbols, bigint
  }
  return new Error(`Non-Error value thrown: ${serialized}`, { cause: thrown });
}
```

Now every catch block in the codebase can start with one line:

```ts
} catch (thrown) {
  const err = normalizeError(thrown);  // err: Error — narrowing done once, centrally
  logger.error(err.message, { stack: err.stack });
}
```

### Concept 3 — Custom error classes with typed fields

The whole point of a custom error is the *data* it carries, not the message.

```ts
// ❌ Data smuggled inside a string — unparseable, untypeable
throw new Error(`User ${userId} not found`);

// ✅ Data as typed fields — the handler reads them structurally
throw new NotFoundError("User", userId);
```

A well-formed error class:

```ts
class RateLimitError extends AppError {
  constructor(
    readonly limit: number,          // 100
    readonly windowSeconds: number,  // 60
    readonly retryAfterSeconds: number,
    readonly clientId: string,
  ) {
    super(
      `Rate limit exceeded: ${limit} requests per ${windowSeconds}s`,
      429,
      "RATE_LIMITED",
    );
  }

  /** Errors can own their own serialization — keeps the handler dumb. */
  toResponseBody(): { code: string; error: string; retryAfter: number } {
    return { code: this.code, error: this.message, retryAfter: this.retryAfterSeconds };
  }
}

// The handler needs zero knowledge of rate limiting:
if (err instanceof RateLimitError) {
  res.setHeader("Retry-After", String(err.retryAfterSeconds));
  res.status(err.statusCode).json(err.toResponseBody());
}
```

Note `readonly` on the constructor parameters — these are **parameter properties** (see `36 — Parameter properties`), so `readonly limit: number` declares the field, assigns it, and freezes it in one token.

### Concept 4 — The `extends Error` + `Object.setPrototypeOf` gotcha

This is the single most-hit trap in TypeScript error handling.

```ts
class NotFoundError extends Error {
  constructor(message: string) {
    super(message);
  }
}

const err = new NotFoundError("User not found");
console.log(err instanceof NotFoundError);  // true when target >= ES2015
                                            // FALSE when target is ES5  😱
```

**Why.** When `target` is `"ES5"` (or `"ES3"`), TypeScript downlevels `class` to a function plus prototype wiring. But `Error` is a *built-in* whose constructor, when called as `Error.call(this, message)`, **ignores `this` and returns a brand-new plain Error object**. That returned object becomes the result of `new NotFoundError(...)`, and its prototype is `Error.prototype`, not `NotFoundError.prototype`. So:

- `err instanceof NotFoundError` → `false`
- Your own methods (`err.toResponseBody()`) → `undefined`, `TypeError` on call
- Your own fields (`err.statusCode`) → `undefined`

The same applies to subclassing `Array`, `Map`, and `Promise` under ES5.

**The fix** — restore the prototype explicitly as the last thing the constructor does:

```ts
class NotFoundError extends Error {
  constructor(readonly resource: string, readonly resourceId: string | number) {
    super(`${resource} ${resourceId} not found`);

    // ✅ Re-point the prototype chain at THIS class.
    Object.setPrototypeOf(this, NotFoundError.prototype);

    this.name = "NotFoundError";
  }
}
```

**With a hierarchy, hardcoding the class breaks subclasses.** `Object.setPrototypeOf(this, AppError.prototype)` in the base would flatten every subclass down to `AppError`. Use `new.target.prototype`, which is the *actually constructed* class:

```ts
class AppError extends Error {
  constructor(message: string, readonly statusCode: number, readonly code: string) {
    super(message);

    // new.target is NotFoundError when you do `new NotFoundError(...)`,
    // AppError when you do `new AppError(...)`. One line fixes the whole tree.
    Object.setPrototypeOf(this, new.target.prototype);

    this.name = new.target.name;   // "NotFoundError" — great in logs and JSON
  }
}
```

**`Error.captureStackTrace` — the second half of the fix.** V8 (Node, Chrome) exposes `Error.captureStackTrace(targetObject, constructorOpt)`. Passing the constructor as the second argument **omits every frame at or above it**, so the stack starts at the line that did `throw new NotFoundError(...)` rather than inside your error class plumbing:

```ts
class AppError extends Error {
  constructor(message: string, readonly statusCode: number, readonly code: string) {
    super(message);
    Object.setPrototypeOf(this, new.target.prototype);
    this.name = new.target.name;

    // Optional chaining: captureStackTrace is V8-only (absent in Safari/Bun-WebKit).
    // The `?.` makes this safe everywhere without a feature-detect branch.
    Error.captureStackTrace?.(this, new.target);
  }
}
```

Without it, the top of every stack trace is `at new AppError (errors.ts:12)` — noise that repeats on every error in your logs.

**When can you skip `setPrototypeOf`?** Only when *all* of these hold:

- `"target"` in tsconfig is `"ES2015"` or later (in 2026, `"ES2022"` is a sane default for Node 18+)
- You are not down-compiling with Babel's loose class transform
- Nothing in your build chain re-downlevels the output

Since it costs one line and is a no-op on modern targets, **always include it in the base class**. It is insurance against a future `target` change made by someone else.

### Concept 5 — Error hierarchies

A hierarchy gives you two narrowing granularities from one `instanceof` check: broad (`AppError` → "this is an expected failure, use its statusCode") and narrow (`ValidationError` → "this one has `fields`").

```ts
// ── Root of the hierarchy ────────────────────────────────────────────────────
abstract class AppError extends Error {
  abstract readonly statusCode: number;
  abstract readonly code: string;

  /** true = expected failure (bad input, missing row). false = a bug. */
  readonly isOperational: boolean = true;

  constructor(message: string, options?: { cause?: unknown }) {
    super(message, options);
    Object.setPrototypeOf(this, new.target.prototype);
    this.name = new.target.name;
    Error.captureStackTrace?.(this, new.target);
  }

  /** Shape sent to the client. Subclasses widen it. */
  toJSON(): Record<string, unknown> {
    return { name: this.name, code: this.code, message: this.message };
  }
}

// ── 4xx family ───────────────────────────────────────────────────────────────
class ValidationError extends AppError {
  readonly statusCode = 422;
  readonly code = "VALIDATION_FAILED";

  constructor(readonly fields: Readonly<Record<string, string>>) {
    super("Request validation failed");
  }

  override toJSON(): Record<string, unknown> {
    return { ...super.toJSON(), fields: this.fields };
  }
}

class NotFoundError extends AppError {
  readonly statusCode = 404;
  readonly code = "NOT_FOUND";

  constructor(readonly resource: string, readonly resourceId: string | number) {
    super(`${resource} '${resourceId}' not found`);
  }
}

class UnauthorizedError extends AppError {
  readonly statusCode = 401;
  readonly code = "UNAUTHORIZED";

  constructor(readonly reason: "missing_token" | "expired_token" | "invalid_signature") {
    super(`Authentication failed: ${reason}`);
  }
}

class ForbiddenError extends AppError {
  readonly statusCode = 403;
  readonly code = "FORBIDDEN";

  constructor(readonly requiredRole: string, readonly actualRole: string) {
    super(`Requires role '${requiredRole}', caller has '${actualRole}'`);
  }
}

class ConflictError extends AppError {
  readonly statusCode = 409;
  readonly code = "CONFLICT";

  constructor(message: string, readonly conflictingField: string) {
    super(message);
  }
}

// ── 5xx family ───────────────────────────────────────────────────────────────
class DatabaseError extends AppError {
  readonly statusCode = 503;
  readonly code = "DATABASE_UNAVAILABLE";
  readonly retryable = true;

  constructor(readonly operation: string, options?: { cause?: unknown }) {
    super(`Database operation '${operation}' failed`, options);
  }
}

class ExternalServiceError extends AppError {
  readonly statusCode = 502;
  readonly code = "UPSTREAM_FAILED";

  constructor(
    readonly service: "stripe" | "sendgrid" | "s3" | "twilio",
    readonly upstreamStatus: number | null,
    options?: { cause?: unknown },
  ) {
    super(`Upstream service '${service}' failed`, options);
  }
}

// ── One narrow, five outcomes ────────────────────────────────────────────────
function toHttpResponse(err: unknown): { status: number; body: Record<string, unknown> } {
  if (err instanceof AppError) {
    return { status: err.statusCode, body: err.toJSON() };  // covers every subclass
  }
  return {
    status: 500,
    body: { name: "InternalError", code: "INTERNAL", message: "Internal server error" },
  };
}
```

Note the `abstract readonly statusCode: number` in the base: every subclass is *forced* by the compiler to declare a status code. Forget it and the build fails (see `37 — Abstract classes`).

### Concept 6 — `cause`: chaining errors without losing the original

ES2022 added a second argument to the `Error` constructor. TypeScript types it once you set `"lib": ["ES2022"]` (or `"target": "ES2022"`, which implies it).

```jsonc
// tsconfig.json
{ "compilerOptions": { "target": "ES2022", "lib": ["ES2022"] } }
```

```ts
async function loadUserProfile(userId: number): Promise<UserProfile> {
  try {
    return await profileRepo.findByUserId(userId);
  } catch (err) {
    // Wrap with domain meaning, but KEEP the driver-level error underneath
    throw new DatabaseError("profileRepo.findByUserId", { cause: err });
  }
}
```

`err.cause` is typed `unknown` — deliberately, for the same reason `catch` is. Walk the chain with a helper:

```ts
/** Flatten an error and all its causes into an array, oldest last. */
function unwrapCauses(err: unknown, maxDepth = 10): Error[] {
  const chain: Error[] = [];
  let current: unknown = err;

  for (let depth = 0; depth < maxDepth && current !== undefined && current !== null; depth++) {
    const normalized = normalizeError(current);
    chain.push(normalized);
    current = normalized.cause;      // unknown — loop normalizes it next pass
  }
  return chain;
}

/** Find the root cause matching a specific class, anywhere in the chain. */
function findCause<T extends Error>(
  err: unknown,
  ctor: new (...args: never[]) => T,
): T | undefined {
  for (const link of unwrapCauses(err)) {
    if (link instanceof ctor) return link;
  }
  return undefined;
}

// Usage: was this 503 really a connection-pool exhaustion?
const pgError = findCause(err, PoolExhaustedError);
if (pgError) metrics.increment("db.pool_exhausted");
```

For logging, serialize the whole chain — otherwise the interesting error is the one you dropped:

```ts
function serializeError(err: unknown): Record<string, unknown> {
  return {
    chain: unwrapCauses(err).map((link) => ({
      name:    link.name,
      message: link.message,
      stack:   link.stack?.split("\n").slice(0, 5).join("\n"),  // trim noise
      ...(link instanceof AppError ? { code: link.code, statusCode: link.statusCode } : {}),
    })),
  };
}
```

Node's `util.inspect` and `console.error` already print `[cause]:` chains natively, which makes local debugging pleasant with zero extra code.

### Concept 7 — Typing async error paths

**The signature lies, and that is fine — but know it.** A `Promise<User>` return type says nothing about rejection. TypeScript has no checked exceptions and no `Promise<T, E>`. Your options:

```ts
// (a) Document it and throw — the idiomatic default
/** @throws {NotFoundError} when no user has this id */
/** @throws {DatabaseError} when the query fails */
async function getUserById(userId: number): Promise<User> { /* ... */ }

// (b) Return the failure in the type — see 61 — Result type pattern
async function getUserById(userId: number): Promise<Result<User, NotFoundError | DatabaseError>> { }
```

**`return await` vs `return` inside `try`.** This bites everyone:

```ts
// ❌ The promise is returned BEFORE it settles — the catch never runs
async function loadUser(userId: number): Promise<User> {
  try {
    return userRepo.findById(userId);        // no await
  } catch (err) {
    throw new NotFoundError("User", userId); // 🚫 unreachable for async rejections
  }
}

// ✅ await inside the try, so the rejection becomes a throw here
async function loadUser(userId: number): Promise<User> {
  try {
    return await userRepo.findById(userId);  // ← the await matters
  } catch (err) {
    throw new NotFoundError("User", userId);
  }
}
```

The ESLint rule `@typescript-eslint/return-await` with the `"in-try-catch"` option enforces exactly this.

**Parallel work: `Promise.all` vs `Promise.allSettled`.** `Promise.all` rejects with the *first* error and abandons the rest — the other rejections become unhandled:

```ts
// ❌ If both reject, the second rejection is unhandled
const [user, orders] = await Promise.all([
  userRepo.findById(userId),
  orderRepo.findByUserId(userId),
]);

// ✅ allSettled gives you a discriminated union per task — nothing is lost
const settled = await Promise.allSettled([
  userRepo.findById(userId),
  orderRepo.findByUserId(userId),
]);

// settled[0]: PromiseSettledResult<User>
//   = { status: "fulfilled"; value: User } | { status: "rejected"; reason: any }
const [userResult, ordersResult] = settled;

if (userResult.status === "rejected") {
  throw new DatabaseError("userRepo.findById", { cause: userResult.reason });
}
const user: User = userResult.value;                 // ✅ narrowed
const orders: Order[] = ordersResult.status === "fulfilled" ? ordersResult.value : [];
```

`PromiseSettledResult<T>` is itself a discriminated union on `status` — see `42 — Discriminated unions`.

**Unhandled rejections and uncaught exceptions.** Since Node 15 an unhandled rejection **terminates the process** by default. Register typed last-resort handlers at boot:

```ts
// src/bootstrap/process-handlers.ts
export function installProcessHandlers(logger: Logger, server: HttpServer): void {
  // A promise rejected with no .catch() anywhere.
  process.on("unhandledRejection", (reason: unknown, promise: Promise<unknown>) => {
    logger.fatal("Unhandled promise rejection", { err: serializeError(reason), promise });
    shutdown(1);
  });

  // A synchronous throw that escaped every try/catch.
  process.on("uncaughtException", (err: Error, origin: NodeJS.UncaughtExceptionOrigin) => {
    logger.fatal("Uncaught exception", { err: serializeError(err), origin });
    shutdown(1);
  });

  function shutdown(exitCode: number): void {
    // Stop accepting new connections, drain in-flight requests, then die.
    // The process is in an UNKNOWN state — do not try to keep serving.
    server.close(() => process.exit(exitCode));
    setTimeout(() => process.exit(exitCode), 10_000).unref();  // hard deadline
  }
}
```

Note the parameter types: `reason` is `unknown` (correct — anything can reject a promise) while `err` in `uncaughtException` is typed `Error` by `@types/node`, which is a small lie (`throw "x"` at top level also lands there). Run both through `serializeError`.

### Concept 8 — Operational errors vs programmer errors

The most valuable distinction in production error handling:

| | Operational error | Programmer error |
|---|---|---|
| Example | `NotFoundError`, `ValidationError`, `ETIMEDOUT` | `TypeError: x.map is not a function` |
| Cause | The world (bad input, missing row, upstream down) | A bug in your code |
| Response | 4xx/5xx with a specific `code` | 500, generic message |
| Log level | `warn` / `info` | `error` / `fatal` + page on-call |
| Message to client | Safe to show | **Never** show — leaks internals |
| Recoverable | Yes, retry or fix the input | No, redeploy |

```ts
function isOperational(err: unknown): boolean {
  return err instanceof AppError && err.isOperational;
}

// In the handler:
if (isOperational(err)) {
  logger.warn("Operational error", { code: (err as AppError).code, path: req.path });
} else {
  logger.error("BUG — unexpected error", { err: serializeError(err), path: req.path });
  metrics.increment("errors.unexpected");   // this one should alert
}
```

Leaking a programmer error's `message` to the client is a genuine security issue — stack traces and SQL fragments end up in browser dev tools. Gate it on the environment:

```ts
const isProduction = process.env.NODE_ENV === "production";
const clientMessage = isOperational(err) || !isProduction
  ? normalizeError(err).message
  : "Internal server error";
```

---

## Example 1 — basic

```ts
// A small, complete typed-error module you could drop into any Node service.

// ── errors.ts ────────────────────────────────────────────────────────────────

/** Base class for every error this application throws on purpose. */
export abstract class AppError extends Error {
  abstract readonly statusCode: number;
  abstract readonly code: string;
  readonly isOperational = true;

  constructor(message: string, options?: { cause?: unknown }) {
    super(message, options);
    Object.setPrototypeOf(this, new.target.prototype); // ES5 instanceof fix
    this.name = new.target.name;                       // "NotFoundError", not "Error"
    Error.captureStackTrace?.(this, new.target);       // hide constructor frames
  }
}

export class NotFoundError extends AppError {
  readonly statusCode = 404;
  readonly code = "NOT_FOUND";
  constructor(readonly resource: string, readonly resourceId: string | number) {
    super(`${resource} '${resourceId}' not found`);
  }
}

export class ValidationError extends AppError {
  readonly statusCode = 422;
  readonly code = "VALIDATION_FAILED";
  constructor(readonly fields: Readonly<Record<string, string>>) {
    super("Request validation failed");
  }
}

export class UnauthorizedError extends AppError {
  readonly statusCode = 401;
  readonly code = "UNAUTHORIZED";
  constructor(readonly reason: "missing_token" | "expired_token" | "invalid_signature") {
    super(`Authentication failed: ${reason}`);
  }
}

// ── guards.ts ────────────────────────────────────────────────────────────────

/** True for anything shaped like an Error, even across realms. */
export function isErrorLike(value: unknown): value is { name: string; message: string } {
  return (
    typeof value === "object" &&
    value !== null &&
    "message" in value &&
    typeof (value as { message: unknown }).message === "string"
  );
}

/** Coerce any thrown value into a real Error. Never throws. */
export function normalizeError(thrown: unknown): Error {
  if (thrown instanceof Error) return thrown;
  if (typeof thrown === "string") return new Error(thrown);
  if (isErrorLike(thrown)) {
    const err = new Error(thrown.message);
    err.name = thrown.name;
    return err;
  }
  let text: string;
  try { text = JSON.stringify(thrown) ?? String(thrown); }
  catch { text = String(thrown); }
  return new Error(`Non-Error thrown: ${text}`, { cause: thrown });
}

// ── usage.ts ─────────────────────────────────────────────────────────────────

interface User {
  userId: number;
  email:  string;
  role:   "admin" | "member" | "guest";
}

const usersById = new Map<number, User>([
  [1, { userId: 1, email: "ada@example.com", role: "admin" }],
]);

function parseUserId(raw: string): number {
  const userId = Number.parseInt(raw, 10);
  if (!Number.isInteger(userId) || userId <= 0) {
    throw new ValidationError({ userId: "must be a positive integer" });
  }
  return userId;
}

function getUser(rawUserId: string, authToken: string | undefined): User {
  if (!authToken) throw new UnauthorizedError("missing_token");

  const userId = parseUserId(rawUserId);
  const user = usersById.get(userId);
  if (!user) throw new NotFoundError("User", userId);

  return user;
}

// ── The single place that turns errors into responses ────────────────────────

interface ApiResponse {
  status: number;
  body:   Record<string, unknown>;
}

function handleGetUser(rawUserId: string, authToken: string | undefined): ApiResponse {
  try {
    const user = getUser(rawUserId, authToken);
    return { status: 200, body: { data: user } };
  } catch (thrown: unknown) {
    // Most specific first — ValidationError has a field the others don't.
    if (thrown instanceof ValidationError) {
      return {
        status: thrown.statusCode,                     // 422
        body: { code: thrown.code, fields: thrown.fields },
      };
    }
    // Broad: every other AppError subclass, present and future.
    if (thrown instanceof AppError) {
      return { status: thrown.statusCode, body: { code: thrown.code, error: thrown.message } };
    }
    // Not ours — a bug or a rogue throw. Log the truth, tell the client nothing.
    console.error("Unexpected error", normalizeError(thrown));
    return { status: 500, body: { code: "INTERNAL", error: "Internal server error" } };
  }
}

console.log(handleGetUser("1", "Bearer abc"));
// { status: 200, body: { data: { userId: 1, ... } } }

console.log(handleGetUser("1", undefined));
// { status: 401, body: { code: "UNAUTHORIZED", error: "Authentication failed: missing_token" } }

console.log(handleGetUser("abc", "Bearer abc"));
// { status: 422, body: { code: "VALIDATION_FAILED", fields: { userId: "must be ..." } } }

console.log(handleGetUser("999", "Bearer abc"));
// { status: 404, body: { code: "NOT_FOUND", error: "User '999' not found" } }
```

---

## Example 2 — real world backend use case

```ts
// A production Express error pipeline: typed hierarchy, driver-error translation,
// async route wrapper, and a single exhaustive error-handling middleware.

import type { Request, Response, NextFunction, RequestHandler, ErrorRequestHandler } from "express";

// ═══════════════════════════════════════════════════════════════════════════
// 1. The error hierarchy
// ═══════════════════════════════════════════════════════════════════════════

export abstract class AppError extends Error {
  abstract readonly statusCode: number;
  abstract readonly code: string;
  /** false = a bug in our code; true = an expected failure mode. */
  readonly isOperational: boolean = true;

  constructor(message: string, options?: { cause?: unknown }) {
    super(message, options);
    Object.setPrototypeOf(this, new.target.prototype);
    this.name = new.target.name;
    Error.captureStackTrace?.(this, new.target);
  }

  /** Body sent to the client. Subclasses add their own fields. */
  toResponseBody(): Record<string, unknown> {
    return { code: this.code, message: this.message };
  }
}

export class ValidationError extends AppError {
  readonly statusCode = 422;
  readonly code = "VALIDATION_FAILED";
  constructor(readonly fields: Readonly<Record<string, string>>) {
    super("Request validation failed");
  }
  override toResponseBody(): Record<string, unknown> {
    return { ...super.toResponseBody(), fields: this.fields };
  }
}

export class NotFoundError extends AppError {
  readonly statusCode = 404;
  readonly code = "NOT_FOUND";
  constructor(readonly resource: string, readonly resourceId: string | number) {
    super(`${resource} '${resourceId}' not found`);
  }
}

export class UnauthorizedError extends AppError {
  readonly statusCode = 401;
  readonly code = "UNAUTHORIZED";
  constructor(readonly reason: "missing_token" | "expired_token" | "invalid_signature") {
    super(`Authentication failed: ${reason}`);
  }
}

export class ForbiddenError extends AppError {
  readonly statusCode = 403;
  readonly code = "FORBIDDEN";
  constructor(readonly requiredRole: string, readonly actualRole: string) {
    super(`Requires role '${requiredRole}'`);
  }
}

export class ConflictError extends AppError {
  readonly statusCode = 409;
  readonly code = "CONFLICT";
  constructor(message: string, readonly conflictingField: string) {
    super(message);
  }
  override toResponseBody(): Record<string, unknown> {
    return { ...super.toResponseBody(), field: this.conflictingField };
  }
}

export class RateLimitError extends AppError {
  readonly statusCode = 429;
  readonly code = "RATE_LIMITED";
  constructor(readonly retryAfterSeconds: number, readonly clientId: string) {
    super(`Rate limit exceeded; retry in ${retryAfterSeconds}s`);
  }
}

export class DatabaseError extends AppError {
  readonly statusCode = 503;
  readonly code = "DATABASE_UNAVAILABLE";
  readonly retryable = true;
  constructor(readonly operation: string, options?: { cause?: unknown }) {
    super(`Database operation '${operation}' failed`, options);
  }
}

export class ExternalServiceError extends AppError {
  readonly statusCode = 502;
  readonly code = "UPSTREAM_FAILED";
  constructor(
    readonly service: "stripe" | "sendgrid" | "s3",
    readonly upstreamStatus: number | null,
    options?: { cause?: unknown },
  ) {
    super(`Upstream '${service}' failed`, options);
  }
}

// ═══════════════════════════════════════════════════════════════════════════
// 2. Guards and normalizers for errors we do NOT own
// ═══════════════════════════════════════════════════════════════════════════

interface PostgresError extends Error {
  code:        string;      // SQLSTATE, e.g. "23505"
  detail?:     string;
  constraint?: string;
  table?:      string;
}

function isPostgresError(err: unknown): err is PostgresError {
  return err instanceof Error && /^[0-9A-Z]{5}$/.test((err as PostgresError).code ?? "");
}

function isErrnoException(err: unknown): err is NodeJS.ErrnoException {
  return err instanceof Error && typeof (err as NodeJS.ErrnoException).code === "string";
}

/** Translate a raw driver error into our domain hierarchy at the repository edge. */
function translateDatabaseError(operation: string, err: unknown): AppError {
  if (isPostgresError(err)) {
    switch (err.code) {
      case "23505":  // unique_violation
        return new ConflictError(
          `Duplicate value violates '${err.constraint ?? "unique constraint"}'`,
          err.constraint ?? "unknown",
        );
      case "23503":  // foreign_key_violation
        return new ValidationError({
          [err.constraint ?? "reference"]: "referenced record does not exist",
        });
      case "23502":  // not_null_violation
        return new ValidationError({ [err.column ?? "field"]: "must not be null" });
      case "57014":  // query_canceled (statement_timeout)
        return new DatabaseError(`${operation} (timed out)`, { cause: err });
      default:
        return new DatabaseError(operation, { cause: err });
    }
  }

  if (isErrnoException(err) && (err.code === "ECONNREFUSED" || err.code === "ETIMEDOUT")) {
    return new DatabaseError(`${operation} (${err.code})`, { cause: err });
  }

  return new DatabaseError(operation, { cause: err });
}

// pg exposes `column` on not-null violations; widen the interface for it:
declare module "pg" {
  interface DatabaseError { column?: string }
}

// ═══════════════════════════════════════════════════════════════════════════
// 3. Repository — the boundary where foreign errors become AppErrors
// ═══════════════════════════════════════════════════════════════════════════

interface User {
  userId:    number;
  email:     string;
  role:      "admin" | "member" | "guest";
  createdAt: Date;
}

interface CreateUserDto {
  email:    string;
  password: string;
  role:     "admin" | "member" | "guest";
}

class UserRepository {
  constructor(private readonly pool: { query<T>(sql: string, params: unknown[]): Promise<{ rows: T[] }> }) {}

  async findById(userId: number): Promise<User | null> {
    try {
      const { rows } = await this.pool.query<User>(
        "SELECT user_id AS \"userId\", email, role, created_at AS \"createdAt\" FROM users WHERE user_id = $1",
        [userId],
      );
      return rows[0] ?? null;
    } catch (err: unknown) {
      // Never let a raw pg error escape the repository.
      throw translateDatabaseError("UserRepository.findById", err);
    }
  }

  async create(dto: CreateUserDto, passwordHash: string): Promise<User> {
    try {
      const { rows } = await this.pool.query<User>(
        "INSERT INTO users (email, password_hash, role) VALUES ($1, $2, $3) RETURNING user_id AS \"userId\", email, role, created_at AS \"createdAt\"",
        [dto.email, passwordHash, dto.role],
      );
      // A RETURNING insert always yields a row; if it doesn't, that's a bug.
      const created = rows[0];
      if (!created) throw new DatabaseError("UserRepository.create (no row returned)");
      return created;
    } catch (err: unknown) {
      if (err instanceof AppError) throw err;             // already translated
      throw translateDatabaseError("UserRepository.create", err);
    }
  }
}

// ═══════════════════════════════════════════════════════════════════════════
// 4. Service — throws domain errors only
// ═══════════════════════════════════════════════════════════════════════════

class UserService {
  constructor(
    private readonly userRepo: UserRepository,
    private readonly hasher:   { hash(plain: string): Promise<string> },
  ) {}

  async getById(userId: number): Promise<User> {
    const user = await this.userRepo.findById(userId);
    if (!user) throw new NotFoundError("User", userId);
    return user;
  }

  async create(requestBody: unknown): Promise<User> {
    const dto = this.validateCreateUserDto(requestBody);   // throws ValidationError
    const passwordHash = await this.hasher.hash(dto.password);
    return this.userRepo.create(dto, passwordHash);        // may throw ConflictError
  }

  /** Hand-rolled validation that collects ALL field errors, not just the first. */
  private validateCreateUserDto(requestBody: unknown): CreateUserDto {
    const fields: Record<string, string> = {};

    if (typeof requestBody !== "object" || requestBody === null) {
      throw new ValidationError({ body: "must be a JSON object" });
    }
    const body = requestBody as Record<string, unknown>;

    if (typeof body.email !== "string" || !body.email.includes("@")) {
      fields.email = "must be a valid email address";
    }
    if (typeof body.password !== "string" || body.password.length < 12) {
      fields.password = "must be at least 12 characters";
    }
    if (body.role !== "admin" && body.role !== "member" && body.role !== "guest") {
      fields.role = "must be one of: admin, member, guest";
    }
    if (Object.keys(fields).length > 0) throw new ValidationError(fields);

    return {
      email:    body.email as string,
      password: body.password as string,
      role:     body.role as CreateUserDto["role"],
    };
  }
}

// ═══════════════════════════════════════════════════════════════════════════
// 5. Async route wrapper — forwards rejections to Express's error pipeline
// ═══════════════════════════════════════════════════════════════════════════

/**
 * Express 4 does NOT catch rejected promises from async handlers — the request
 * hangs forever. This wrapper attaches .catch(next) so every rejection reaches
 * the error middleware. (Express 5 does this natively; the wrapper is harmless.)
 */
function asyncRoute(
  handler: (req: Request, res: Response, next: NextFunction) => Promise<void>,
): RequestHandler {
  return (req, res, next) => {
    handler(req, res, next).catch(next);
  };
}

// ═══════════════════════════════════════════════════════════════════════════
// 6. Routes — no try/catch, they just throw
// ═══════════════════════════════════════════════════════════════════════════

declare const userService: UserService;
declare const logger: {
  warn(msg: string, meta?: Record<string, unknown>): void;
  error(msg: string, meta?: Record<string, unknown>): void;
};

const getUserRoute: RequestHandler = asyncRoute(async (req, res) => {
  const userId = Number.parseInt(req.params.userId ?? "", 10);
  if (!Number.isInteger(userId) || userId <= 0) {
    throw new ValidationError({ userId: "must be a positive integer" });
  }
  const user = await userService.getById(userId);   // throws NotFoundError / DatabaseError
  res.status(200).json({ data: user });
});

const createUserRoute: RequestHandler = asyncRoute(async (req, res) => {
  const user = await userService.create(req.body);  // throws ValidationError / ConflictError
  res.status(201).json({ data: user });
});

// ═══════════════════════════════════════════════════════════════════════════
// 7. The typed error middleware — the ONE place errors become responses
// ═══════════════════════════════════════════════════════════════════════════

interface ErrorResponseBody {
  code:       string;
  message:    string;
  requestId:  string;
  fields?:    Readonly<Record<string, string>>;
  [extra: string]: unknown;
}

const isProduction = process.env.NODE_ENV === "production";

/**
 * Express identifies error middleware by ARITY — it must declare exactly four
 * parameters. If you drop `_next`, Express treats it as a normal handler and
 * your errors silently bypass it. Hence the eslint-disable on the unused param.
 */
const errorMiddleware: ErrorRequestHandler = (
  thrown: unknown,
  req: Request,
  res: Response,
  // eslint-disable-next-line @typescript-eslint/no-unused-vars
  _next: NextFunction,
): void => {
  const requestId = (req.headers["x-request-id"] as string | undefined) ?? "unknown";

  // If headers already went out (e.g. a stream failed mid-write), we cannot
  // send a body — delegate to Express's default handler by destroying the socket.
  if (res.headersSent) {
    logger.error("Error after headers sent", { requestId, err: serializeError(thrown) });
    res.destroy();
    return;
  }

  // ── Our errors: structured, safe to surface ────────────────────────────────
  if (thrown instanceof AppError) {
    const body: ErrorResponseBody = {
      requestId,
      ...thrown.toResponseBody(),
    } as ErrorResponseBody;

    if (thrown instanceof RateLimitError) {
      res.setHeader("Retry-After", String(thrown.retryAfterSeconds));
    }

    // 4xx = the caller's problem (warn). 5xx = ours (error).
    const log = thrown.statusCode >= 500 ? logger.error : logger.warn;
    log(`${thrown.name}: ${thrown.message}`, {
      requestId,
      code:       thrown.code,
      statusCode: thrown.statusCode,
      method:     req.method,
      path:       req.path,
      // Include the cause chain for 5xx only — 4xx causes are rarely interesting.
      ...(thrown.statusCode >= 500 ? { err: serializeError(thrown) } : {}),
    });

    res.status(thrown.statusCode).json(body);
    return;
  }

  // ── Body-parser's own error, which is not ours but is still a client fault ─
  if (
    thrown instanceof SyntaxError &&
    "status" in thrown &&
    (thrown as SyntaxError & { status: number }).status === 400
  ) {
    res.status(400).json({ code: "MALFORMED_JSON", message: "Request body is not valid JSON", requestId });
    return;
  }

  // ── Everything else is a bug ──────────────────────────────────────────────
  const normalized = normalizeError(thrown);
  logger.error("Unhandled error", {
    requestId,
    method: req.method,
    path:   req.path,
    err:    serializeError(normalized),
  });

  res.status(500).json({
    code:      "INTERNAL",
    // Never leak a bug's message in production — it can contain SQL or paths.
    message:   isProduction ? "Internal server error" : normalized.message,
    requestId,
    ...(isProduction ? {} : { stack: normalized.stack }),
  });
};

// ── Shared helpers used above ───────────────────────────────────────────────

function normalizeError(thrown: unknown): Error {
  if (thrown instanceof Error) return thrown;
  if (typeof thrown === "string") return new Error(thrown);
  let text: string;
  try { text = JSON.stringify(thrown) ?? String(thrown); }
  catch { text = String(thrown); }
  return new Error(`Non-Error thrown: ${text}`, { cause: thrown });
}

function serializeError(thrown: unknown, maxDepth = 5): Record<string, unknown>[] {
  const chain: Record<string, unknown>[] = [];
  let current: unknown = thrown;

  for (let depth = 0; depth < maxDepth && current != null; depth++) {
    const link = normalizeError(current);
    chain.push({
      name:    link.name,
      message: link.message,
      stack:   link.stack?.split("\n").slice(0, 6).join("\n"),
      ...(link instanceof AppError ? { code: link.code, statusCode: link.statusCode } : {}),
    });
    current = link.cause;
  }
  return chain;
}

// ═══════════════════════════════════════════════════════════════════════════
// 8. Wiring — order matters: routes first, 404, then the error middleware LAST
// ═══════════════════════════════════════════════════════════════════════════

declare const app: {
  get(path: string, handler: RequestHandler): void;
  post(path: string, handler: RequestHandler): void;
  use(handler: RequestHandler | ErrorRequestHandler): void;
};

app.get("/users/:userId", getUserRoute);
app.post("/users", createUserRoute);

// Nothing matched → a NotFoundError like any other
app.use((req: Request, _res: Response, next: NextFunction) => {
  next(new NotFoundError("Route", `${req.method} ${req.path}`));
});

app.use(errorMiddleware);   // must be registered after every route
```

---

## Going deeper

### `useUnknownInCatchVariables` is not retroactive to `.catch()`

The flag only affects `catch` *clauses*. The `Promise.prototype.catch` callback parameter is typed `any` in `lib.es5.d.ts`:

```ts
// lib.es5.d.ts (simplified)
catch<TResult = never>(
  onrejected?: ((reason: any) => TResult | PromiseLike<TResult>) | null
): Promise<T | TResult>;
```

So this compiles and crashes:

```ts
// ❌ `err` is any — the compiler will not save you here
userRepo.findById(userId).catch((err) => {
  logger.error(err.response.data.message);   // 💥 at runtime
});

// ✅ Annotate it yourself
userRepo.findById(userId).catch((err: unknown) => {
  logger.error("findById failed", { err: serializeError(err) });
});
```

The `@typescript-eslint/use-unknown-in-catch-callback-variable` rule enforces this automatically.

### `instanceof` fails across realms and duplicate module instances

`instanceof` compares prototype *identity*. Two situations break it even on modern targets:

1. **Cross-realm** — a value from a `vm` context, a `worker_threads` message, or (in older Node) a different `require` cache entry has a *different* `Error.prototype`.
2. **Duplicate package copies** — if `AppError` is defined in `@acme/errors` and npm installs two versions (one hoisted, one nested), `err instanceof AppError` is `false` for errors thrown by the other copy. This is brutal in monorepos.

Defence: add a branded structural check alongside `instanceof`:

```ts
const APP_ERROR_BRAND = Symbol.for("@acme/errors.AppError");  // Symbol.for is cross-realm

abstract class AppError extends Error {
  readonly [APP_ERROR_BRAND] = true as const;
  abstract readonly statusCode: number;
  abstract readonly code: string;
  // ...
}

/** Works even when instanceof does not. */
function isAppError(err: unknown): err is AppError {
  return (
    typeof err === "object" &&
    err !== null &&
    APP_ERROR_BRAND in err &&
    (err as Record<symbol, unknown>)[APP_ERROR_BRAND] === true
  );
}
```

`Symbol.for` uses the global symbol registry, which is shared across module copies within a realm — so the brand matches even when the classes don't.

### `Error.captureStackTrace` and `stackTraceLimit` are V8-only and cost real time

```ts
// V8-only API. Absent in JavaScriptCore (Safari, Bun's WebKit paths) and
// SpiderMonkey. The types live in @types/node, so it compiles in Node but
// will be `undefined` at runtime elsewhere — always use ?. :
Error.captureStackTrace?.(this, new.target);
```

Stack capture is **not free**. V8 captures a structured stack eagerly at construction and formats it lazily on first `.stack` access. Constructing hundreds of thousands of errors per second (a hot validation loop, a parser that throws on every candidate) is measurable:

```ts
// In an extremely hot path, you can disable capture for a specific error class:
class FastControlFlowError extends Error {
  constructor(message: string) {
    const previousLimit = Error.stackTraceLimit;
    Error.stackTraceLimit = 0;      // skip capturing frames entirely
    super(message);
    Error.stackTraceLimit = previousLimit;
    Object.setPrototypeOf(this, FastControlFlowError.prototype);
  }
}
```

Better still: **don't use exceptions for control flow in hot paths** — return a `Result` (see `61 — Result type pattern`). Exceptions are for exceptional situations partly *because* they are expensive.

Conversely, in production you often want *more* frames, not fewer:

```ts
// Default is 10 frames — often not enough to see past framework plumbing.
Error.stackTraceLimit = 30;
```

### `Error` fields do not survive `JSON.stringify`

```ts
const err = new NotFoundError("User", 42);
JSON.stringify(err);
// → "{"resource":"User","resourceId":42}"
//   message, name, and stack are MISSING — they are non-enumerable own/inherited props
```

`message` is a non-enumerable own property; `name` lives on the prototype; `stack` is non-enumerable. So the most important parts vanish silently in any logger that does `JSON.stringify(meta)`. Two fixes:

```ts
// (a) Give the class an explicit toJSON — JSON.stringify calls it automatically
abstract class AppError extends Error {
  toJSON(): Record<string, unknown> {
    return { name: this.name, code: this.code, message: this.message, stack: this.stack };
  }
}

// (b) Or serialize explicitly at the log site (works for foreign errors too)
logger.error("failed", { err: serializeError(thrown) });
```

Pino, Winston, and Bunyan all have error serializers for this reason — make sure yours is registered (`pino({ serializers: { err: pino.stdSerializers.err } })`).

### `finally` swallows errors if it returns or throws

```ts
// ❌ The `return` in finally DISCARDS the in-flight exception entirely
function readConfig(): string {
  try {
    throw new Error("disk failure");
  } finally {
    return "default-config";   // 😱 the error vanishes; caller sees a normal return
  }
}
// TypeScript does not error on this by default. ESLint's no-unsafe-finally does.
```

The same applies to `break`, `continue`, and a `throw` inside `finally` — each replaces the pending exception. Keep `finally` blocks to cleanup only:

```ts
// ✅ Cleanup, nothing else
async function withTransaction<T>(fn: (tx: Tx) => Promise<T>): Promise<T> {
  const tx = await pool.begin();
  try {
    const result = await fn(tx);
    await tx.commit();
    return result;
  } catch (err) {
    await tx.rollback().catch(() => { /* rollback failure must not mask the real error */ });
    throw err;                         // ← rethrow the original
  } finally {
    tx.release();                      // no return, no throw
  }
}
```

Note the `.catch(() => {})` on the rollback: if `rollback()` itself rejects inside a `catch` block, that rejection *replaces* the error you were handling, and you lose the real cause.

### `never` as a return type makes control flow analysis smarter

A function annotated `: never` tells the compiler that code after the call is unreachable:

```ts
function raise(err: AppError): never {
  throw err;
}

function requireUser(user: User | null, userId: number): User {
  if (!user) raise(new NotFoundError("User", userId));
  return user;   // ✅ compiles — the compiler knows raise() never returns, so user is User
}
```

**Caveat:** this narrowing only applies when the function is declared with an *explicit* `never` return type **and** referenced as a `const`/`function` declaration — not through a mutable variable or an object property without an explicit type annotation:

```ts
const raise = (err: AppError): never => { throw err; };  // ✅ works (const + explicit never)

let raiseLet = (err: AppError): never => { throw err; };
function f(user: User | null): User {
  if (!user) raiseLet(new NotFoundError("User", 1));
  return user;   // ❌ Error: 'user' is possibly 'null' — let bindings don't get this
}
```

### Assertion functions for validation

`asserts` signatures narrow the *argument* for the rest of the enclosing scope:

```ts
function assertIsUser(value: unknown, userId: number): asserts value is User {
  if (typeof value !== "object" || value === null || typeof (value as User).email !== "string") {
    throw new ValidationError({ user: "malformed user record" });
  }
}

function process(record: unknown, userId: number): string {
  assertIsUser(record, userId);
  return record.email;   // ✅ record: User for the rest of the function
}
```

Two hard rules the compiler enforces:

1. The function must have an **explicit type annotation** at the call site's binding — `const assertIsUser = (...): asserts v is User => ...` fails unless the variable itself is annotated.
2. Assertion functions cannot be used on values whose declaration the compiler can't track (e.g. a property access like `obj.field`).

### `strict` vs the error path: `noImplicitAny` inside catch is unrelated

A common confusion: turning off `useUnknownInCatchVariables` gives `any`, and `noImplicitAny` does **not** flag it — implicit-`any` diagnostics do not apply to catch clause variables. So a codebase can be "fully strict" in the linting sense while every catch block is `any`. Check the flag explicitly:

```bash
npx tsc --showConfig | grep useUnknownInCatchVariables
```

### Errors in TypeScript's own `--isolatedModules` / `verbatimModuleSyntax`

When re-exporting error classes from a barrel file, remember that classes are **values**, not types:

```ts
// ❌ Under verbatimModuleSyntax this erases the class at runtime — instanceof breaks
export type { AppError, NotFoundError } from "./errors";

// ✅ Classes must be value exports
export { AppError, NotFoundError } from "./errors";
```

This bites when a linter auto-fixes "prefer type-only imports" on a class used only in `catch (e) { if (e instanceof X) }` — the fixer sees a type position and rewrites it, silently breaking every `instanceof` at runtime.

---

## Common mistakes

### Mistake 1 — Treating the catch variable as an `Error`

```ts
// ❌ Casting away the unknown — reintroduces exactly the bug the flag prevents
try {
  await stripeClient.charges.create(payload);
} catch (err) {
  const error = err as Error;
  logger.error(error.message);        // 💥 if Stripe rejected with a plain object
  if (error.message.includes("card_declined")) { /* string matching, again */ }
}

// ❌ Equally bad: silencing it at the annotation
try { /* ... */ } catch (err: any) {
  res.status(err.statusCode).json({ error: err.message });  // statusCode may be undefined
}                                                           // → res.status(undefined) throws

// ✅ Narrow, then handle; normalize whatever is left
try {
  await stripeClient.charges.create(payload);
} catch (err: unknown) {
  if (err instanceof ExternalServiceError) {
    res.status(err.statusCode).json({ code: err.code, service: err.service });
    return;
  }
  if (isStripeCardError(err)) {                 // structural guard for the SDK's shape
    res.status(402).json({ code: "CARD_DECLINED", declineCode: err.decline_code });
    return;
  }
  logger.error("Charge failed", { err: serializeError(err) });
  res.status(502).json({ code: "UPSTREAM_FAILED" });
}

function isStripeCardError(err: unknown): err is { type: "StripeCardError"; decline_code: string } {
  return (
    typeof err === "object" && err !== null &&
    (err as { type?: unknown }).type === "StripeCardError"
  );
}
```

### Mistake 2 — Subclassing `Error` without `setPrototypeOf` while targeting ES5

```ts
// ❌ Compiles fine. Breaks silently when tsconfig target is "ES5".
class PaymentError extends Error {
  constructor(message: string, public readonly paymentId: string) {
    super(message);
  }
  isRetryable(): boolean { return true; }
}

try {
  throw new PaymentError("Card declined", "pi_123");
} catch (err) {
  if (err instanceof PaymentError) {   // ❌ false under ES5 — branch never runs
    console.log(err.paymentId);        // never reached
  }
  // Falls through to the generic 500 handler. In production. Silently.
}

// ✅ Restore the prototype and clean the stack — in the BASE class, using new.target
class AppError extends Error {
  constructor(message: string, readonly statusCode: number, readonly code: string) {
    super(message);
    Object.setPrototypeOf(this, new.target.prototype);  // ← the fix
    this.name = new.target.name;
    Error.captureStackTrace?.(this, new.target);
  }
}

class PaymentError extends AppError {
  constructor(message: string, readonly paymentId: string) {
    super(message, 402, "PAYMENT_FAILED");              // base handles the prototype
  }
  isRetryable(): boolean { return true; }
}

try {
  throw new PaymentError("Card declined", "pi_123");
} catch (err) {
  if (err instanceof PaymentError) {   // ✅ true on every target
    console.log(err.paymentId, err.isRetryable());
  }
}
```

### Mistake 3 — Losing the original error when wrapping

```ts
// ❌ The pg error, its SQLSTATE, and its stack are gone forever
try {
  await pool.query(sql, params);
} catch (err) {
  throw new DatabaseError("query failed");   // no cause → undebuggable in production
}

// ❌ Only marginally better — the message is stringified, the stack and code are still lost
try {
  await pool.query(sql, params);
} catch (err) {
  throw new DatabaseError(`query failed: ${String(err)}`);
}

// ✅ Chain it — the original stays reachable via .cause
try {
  await pool.query(sql, params);
} catch (err: unknown) {
  throw new DatabaseError("UserRepository.findById", { cause: err });
}

// And read the chain when logging:
const chain = serializeError(caughtError);
// [ { name: "DatabaseError", code: "DATABASE_UNAVAILABLE", ... },
//   { name: "error", message: "connection terminated unexpectedly", ... } ]
```

### Mistake 4 — `return` without `await` inside a `try`

```ts
// ❌ The catch is dead code for async rejections — the promise escapes the try block
async function chargeUser(userId: number, amountCents: number): Promise<Charge> {
  try {
    return paymentGateway.charge(userId, amountCents);   // no await
  } catch (err) {
    throw new ExternalServiceError("stripe", null, { cause: err });  // 🚫 never runs
  }
}
// The caller gets the raw Stripe rejection, not your typed ExternalServiceError.

// ✅ Await inside the try so rejections become throws in this frame
async function chargeUser(userId: number, amountCents: number): Promise<Charge> {
  try {
    return await paymentGateway.charge(userId, amountCents);
  } catch (err: unknown) {
    throw new ExternalServiceError("stripe", null, { cause: err });
  }
}
```

### Mistake 5 — Catching, logging, and rethrowing at every layer

```ts
// ❌ The same failure logged four times, each with a less useful stack
async function repoFind(userId: number) {
  try { return await db.find(userId); }
  catch (err) { logger.error("repo failed", { err }); throw err; }
}
async function serviceGet(userId: number) {
  try { return await repoFind(userId); }
  catch (err) { logger.error("service failed", { err }); throw err; }
}
// → 4 log lines, 4 alerts, one actual problem. Log volume explodes; signal drowns.

// ✅ Translate at boundaries, log ONCE at the top
async function repoFind(userId: number): Promise<User | null> {
  try { return await db.find(userId); }
  catch (err: unknown) { throw translateDatabaseError("repoFind", err); }  // translate, don't log
}
async function serviceGet(userId: number): Promise<User> {
  const user = await repoFind(userId);      // no try/catch — nothing to add here
  if (!user) throw new NotFoundError("User", userId);
  return user;
}
// The error middleware logs it exactly once, with the full cause chain.
```

### Mistake 6 — Registering the Express error middleware with three parameters

```ts
// ❌ Express identifies error handlers by function ARITY (length === 4).
//    With three parameters this is a NORMAL middleware — errors bypass it entirely
//    and Express's default HTML error page is sent instead.
app.use((err: unknown, req: Request, res: Response) => {
  res.status(500).json({ error: "Internal" });
});

// ✅ Four parameters, even though `next` is unused
const errorMiddleware: ErrorRequestHandler = (err, req, res, _next) => {
  res.status(500).json({ error: "Internal" });
};
app.use(errorMiddleware);
// Typing it as ErrorRequestHandler also stops TS from letting you drop the 4th param.
```

---

## Practice exercises

### Exercise 1 — easy

Build a minimal typed error module for a file-processing CLI.

Requirements:

1. A base class `AppError extends Error` with `readonly code: string` and `readonly exitCode: number`, correctly handling `Object.setPrototypeOf`, `this.name`, and `Error.captureStackTrace`.
2. Three subclasses:
   - `FileNotFoundError` — carries `readonly path: string`, exit code `2`
   - `PermissionDeniedError` — carries `readonly path: string` and `readonly requiredMode: "read" | "write"`, exit code `77`
   - `ParseError` — carries `readonly path: string`, `readonly line: number`, `readonly column: number`, exit code `65`
3. A type guard `isAppError(value: unknown): value is AppError`.
4. A function `describeFailure(thrown: unknown): { exitCode: number; message: string }` that narrows with `instanceof`, returns each subclass's own detailed message (including its typed fields), and falls back to `{ exitCode: 1, message: "Unexpected error" }` for anything else — including a thrown string and a thrown `null`.
5. Prove point 4 works by calling `describeFailure` with each of: a `ParseError`, a plain `Error`, the string `"boom"`, and `null`.

```ts
// Write your code here
```

### Exercise 2 — medium

Build a typed retry wrapper for an unreliable HTTP client.

Requirements:

1. An error hierarchy: `HttpError extends AppError` with `readonly status: number`, plus `ClientHttpError` (4xx, never retry) and `ServerHttpError` (5xx, retry) and `NetworkError` (no response at all — retry).
2. A guard `isRetryable(err: unknown): boolean` returning `true` only for `ServerHttpError`, `NetworkError`, and Node `ErrnoException`s whose `code` is `"ECONNRESET"`, `"ETIMEDOUT"`, or `"ENOTFOUND"`.
3. `async function withRetry<T>(operation: () => Promise<T>, options: { maxAttempts: number; baseDelayMs: number; onRetry?: (attempt: number, err: unknown) => void }): Promise<T>` which:
   - Retries only when `isRetryable(err)` is true
   - Uses exponential backoff with jitter (`baseDelayMs * 2 ** (attempt - 1)` plus up to 20% random)
   - Throws immediately on non-retryable errors
   - After exhausting attempts, throws a `RetryExhaustedError` carrying `readonly attempts: number` and the **last** error as `cause`
   - Uses `unknown` for every caught value, never `any`
4. A `fetchJson<T>(url: string, authToken: string): Promise<T>` that translates a `Response` into the right error subclass (4xx → `ClientHttpError`, 5xx → `ServerHttpError`, a thrown `TypeError` from `fetch` → `NetworkError` with the original as `cause`), and is wrapped in `withRetry`.
5. A `unwrapCauses(err: unknown): Error[]` helper, and a demonstration that after a `RetryExhaustedError` you can still reach the original `NetworkError` through the cause chain.

```ts
// Write your code here
```

### Exercise 3 — hard

Build a complete, framework-agnostic error-handling layer for a REST API and prove its correctness with type-level tests.

Requirements:

1. **Hierarchy** — an `abstract class AppError` with `abstract readonly statusCode`, `abstract readonly code`, `readonly isOperational`, a `toResponseBody()` method, and a `Symbol.for`-based brand so `isAppError` works across duplicate package copies. At least seven subclasses covering 400, 401, 403, 404, 409, 422, 429, 502, 503.

2. **A discriminated union view of the hierarchy** — define
   `type ErrorCode = AppError["code"]` and a function
   `errorCodeToStatus(code: ErrorCode): number` implemented as an **exhaustive switch** with an `assertNever` default, so adding a new error class without updating the map is a compile error. (Hint: you will need each subclass's `code` to be a literal type, not `string` — figure out what declaration makes that happen.)

3. **Driver translation** — `translateUnknownError(context: string, thrown: unknown): AppError` handling: Postgres SQLSTATEs (`23505`, `23503`, `23502`, `57014`, `40001`), Node `ErrnoException`s (`ECONNREFUSED`, `ETIMEDOUT`, `ENOTFOUND`), a Zod-style `{ issues: Array<{ path: (string|number)[]; message: string }> }` shape → `ValidationError`, JSON `SyntaxError`, and anything else → a non-operational `InternalError` with the original as `cause`.

4. **A transport-agnostic handler** — `function toErrorResponse(thrown: unknown, requestId: string, isProduction: boolean): { status: number; headers: Record<string, string>; body: Record<string, unknown> }` that: sets `Retry-After` for rate-limit errors, includes `fields` only for validation errors, includes `stack` only when `!isProduction`, and never leaks a non-operational error's message in production.

5. **Cause-chain utilities** — `unwrapCauses`, `findCause<T extends Error>(err, ctor)`, and `serializeErrorChain(err, maxDepth)` that is safe against circular `cause` references (an error whose cause is itself).

6. **Process-level safety net** — `installProcessHandlers(logger, onShutdown)` wiring `unhandledRejection` and `uncaughtException` with correctly-typed parameters, a 10-second hard-exit deadline using an `unref`'d timer, and idempotency (a second fatal error while shutting down must not restart the shutdown).

7. **Type-level tests** — using `// @ts-expect-error` comments, prove that:
   - `catch (err) { err.message }` does not compile
   - `catch (err: Error)` does not compile
   - a subclass that forgets `statusCode` does not compile
   - `errorCodeToStatus` fails to compile when a new subclass adds a code that the switch does not handle

8. Write a `main()` that exercises the whole pipeline against at least six different thrown values (including a raw string, `null`, a nested three-level cause chain, and a fake Postgres unique-violation object) and prints the resulting responses.

```ts
// Write your code here
```

---

## Quick reference cheat sheet

```ts
// ── The catch variable ──────────────────────────────────────────────────────
try {} catch (err) {}            // err: unknown  (under strict)
try {} catch (err: unknown) {}   // ✅ explicit and allowed
try {} catch (err: any) {}       // ✅ allowed, unsafe
try {} catch (err: Error) {}     // ❌ compile error

// ── Narrowing ladder ────────────────────────────────────────────────────────
if (err instanceof ValidationError) err.fields;     // your class + its data
if (err instanceof Error)           err.message;    // any Error
if (typeof err === "string")        err.length;     // primitive throw
if (isErrorLike(err))               err.message;    // cross-realm / SDK shapes
const e = normalizeError(err);                      // always an Error

// ── Correct custom error base ───────────────────────────────────────────────
abstract class AppError extends Error {
  abstract readonly statusCode: number;
  abstract readonly code: string;
  readonly isOperational = true;
  constructor(message: string, options?: { cause?: unknown }) {
    super(message, options);
    Object.setPrototypeOf(this, new.target.prototype); // ES5 instanceof fix
    this.name = new.target.name;                       // real class name
    Error.captureStackTrace?.(this, new.target);       // trim stack (V8)
  }
  toJSON() { return { name: this.name, code: this.code, message: this.message }; }
}

// ── cause chaining (needs lib >= ES2022) ────────────────────────────────────
throw new DatabaseError("findById", { cause: originalErr });
err.cause          // unknown — normalize before use

// ── async ───────────────────────────────────────────────────────────────────
try { return await p; } catch {}         // ✅ await INSIDE try
try { return p; }        catch {}        // ❌ catch never fires
Promise.allSettled([...])                // per-task { status: "fulfilled"|"rejected" }
p.catch((err: unknown) => {})            // annotate — the lib types it `any`

// ── process safety net ──────────────────────────────────────────────────────
process.on("unhandledRejection", (reason: unknown) => {});
process.on("uncaughtException",  (err: Error, origin) => {});

// ── never / asserts ─────────────────────────────────────────────────────────
function raise(e: AppError): never { throw e; }
function assertIsUser(v: unknown): asserts v is User { /* throw if not */ }
```

| Topic | Rule |
|---|---|
| `catch (err)` type | `unknown` under `strict` (`useUnknownInCatchVariables`) |
| Allowed catch annotations | only `unknown` or `any` — never a specific type |
| `.catch(cb)` param | typed `any` by lib — annotate `unknown` yourself |
| `extends Error` on ES5 | needs `Object.setPrototypeOf(this, new.target.prototype)` |
| Clean stack traces | `Error.captureStackTrace?.(this, new.target)` — V8 only |
| Error name in logs | set `this.name = new.target.name` in the base constructor |
| `JSON.stringify(err)` | drops `message`/`name`/`stack` — add `toJSON()` |
| Wrapping errors | always pass `{ cause: original }` |
| `return` in `try` | must be `return await` or the catch is dead |
| `finally` | never `return`/`throw` inside it — it swallows the exception |
| Express error middleware | must declare **4** parameters or it is ignored |
| Where to log | once, at the top boundary — not at every layer |
| Client-facing message | operational errors only; never a bug's message in production |
| `instanceof` across realms | unreliable — add a `Symbol.for` brand + structural guard |

---

## Connected topics

- **41 — Type guards** — `err is AppError` predicates and `asserts` signatures are how you turn `unknown` catch values into usable types.
- **42 — Discriminated unions** — `PromiseSettledResult<T>` and error-code unions narrow the same way; `assertNever` gives you exhaustive error mapping.
- **37 — Abstract classes** — `abstract readonly statusCode` forces every error subclass to declare its HTTP status at compile time.
- **61 — Result type pattern** — the alternative to throwing: put the failure in the return type so the compiler forces the caller to handle it.
- **07 — Special types** — why `unknown` (not `any`) is the correct type for a value produced by an unconstrained mechanism like `throw`.
