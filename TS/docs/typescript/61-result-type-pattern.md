# 61 — Result type pattern

## What is this?

The **Result type pattern** is the practice of putting failure **in the return type** instead of in the `throw` channel.

Instead of this:

```ts
// The signature promises a User. It says NOTHING about the three ways it can fail.
async function getUserById(userId: number): Promise<User> {
  const row = await userRepo.findById(userId);
  if (!row) throw new NotFoundError("User", userId);   // invisible to the type system
  return row;
}
```

you write this:

```ts
// The signature now states BOTH outcomes. The compiler forces the caller to deal with it.
async function getUserById(
  userId: number,
): Promise<Result<User, NotFoundError | DatabaseError>> {
  const found = await userRepo.findById(userId);
  if (!found.ok) return found;                   // propagate the DatabaseError
  if (!found.value) return err(new NotFoundError("User", userId));
  return ok(found.value);
}
```

`Result<T, E>` is nothing exotic. It is a **discriminated union** of two object shapes (see `42 — Discriminated unions`):

```ts
type Result<T, E> =
  | { readonly ok: true;  readonly value: T }
  | { readonly ok: false; readonly error: E };
```

That is the entire idea. Everything else in this document — `ok()`, `err()`, `mapResult`, `andThen`, `combine`, `tryCatch` — is ergonomics built on top of those five lines.

The payoff is a single sentence: **a thrown error is invisible to TypeScript; a returned error is not.** TypeScript has no checked exceptions, no `throws` clause, and no `Promise<T, E>`. The only place the compiler can see a failure is the return type. Result puts it there.

---

## Why does it matter?

### 1. `throws` is not part of the type

Java has `void save() throws IOException`. TypeScript has nothing. This compiles with zero warnings:

```ts
async function chargeCard(userId: number, amountCents: number): Promise<Charge> {
  // May throw: CardDeclinedError, RateLimitError, NetworkError, ZodError, TypeError…
  // The signature says `Promise<Charge>`. That is a half-truth, and the compiler
  // will never tell you otherwise.
}

// Caller — completely unaware anything can go wrong:
const charge = await chargeCard(userId, 4999);
await orderRepo.markPaid(orderId, charge.id);       // runs only sometimes. Surprise!
```

The bug is not that it throws. The bug is that **nothing in the editor tells you it throws**, so you find out from a Sentry alert at 03:00.

### 2. `catch` gives you `unknown`, so exception handling is always a downgrade

Even when you *do* write a `try`/`catch`, the compiler hands you `unknown` (see `60 — Error handling in TypeScript`). You have to narrow your way back to the type you already knew when you threw it:

```ts
try {
  const charge = await chargeCard(userId, 4999);
} catch (thrown) {
  // thrown: unknown. All type information about WHAT can fail here was destroyed
  // by the throw. You re-derive it by hand, and there is no exhaustiveness check.
  if (thrown instanceof CardDeclinedError) { /* ... */ }
  // Did you cover every case? The compiler has no opinion.
}
```

With Result, the error is `CardDeclinedError | RateLimitError | GatewayUnavailableError` — a closed union — and a `switch` over it can be made **exhaustive** with `assertNever`. Add a new failure mode to the service and every caller that doesn't handle it fails to compile. That is the whole game.

### 3. Failure becomes documentation

```ts
// Reading this signature tells you exactly what can go wrong, without opening the body:
function activateSubscription(
  userId: number,
  planId: string,
): Promise<Result<Subscription, "user_not_found" | "plan_retired" | "already_active" | "payment_declined">>;
```

Code review, onboarding, and API design all get easier when the failure modes are in the signature.

### 4. Expected failures are not exceptional

"User not found" is not an exception. It happens thousands of times an hour on any public API. Neither is "email already registered", "coupon expired", or "insufficient balance". These are **ordinary outcomes of a business operation** and modelling them as exceptions has three costs:

- **Performance** — V8 captures a stack trace on every `new Error()`. In a hot validation loop that is measurable.
- **Control flow** — exceptions are non-local `goto`. You cannot see where they land.
- **Semantics** — mixing "the coupon expired" with "the database exploded" in one channel means your handler has to sort them back out.

### 5. It composes

A single request often chains five fallible steps. With exceptions, one `try` wraps them all and you lose track of which one failed. With Result you chain with `andThen`, and the union of possible errors is computed by the compiler.

**The honest caveat, stated up front:** Result is not a replacement for exceptions everywhere. It belongs in the **service and domain layer**. Exceptions still win at **framework boundaries** (Express middleware), for **programmer errors** (a bug — you want the process to notice), and for **truly unrecoverable** conditions. The `Going deeper` section draws the line precisely.

---

## The JavaScript way vs the TypeScript way

### The JavaScript way — throw and hope

```js
// JavaScript. Every failure mode goes into the same invisible channel.
async function registerUser(requestBody) {
  if (!requestBody.email.includes("@")) {
    throw new Error("Invalid email");            // a 422
  }

  const existing = await userRepo.findByEmail(requestBody.email);
  if (existing) {
    throw new Error("Email already registered"); // a 409
  }

  const passwordHash = await hasher.hash(requestBody.password); // may throw (bcrypt cost)
  const user = await userRepo.insert({ email: requestBody.email, passwordHash }); // may throw pg
  await mailer.sendWelcome(user.email);          // may throw (SendGrid 502)

  return user;
}

// The caller, doing its best:
async function handleRegister(req, res) {
  try {
    const user = await registerUser(req.body);
    res.status(201).json({ data: user });
  } catch (err) {
    // Which of the SIX failure modes was it? Back to string matching.
    if (err.message === "Invalid email")            return res.status(422).json({ error: err.message });
    if (err.message === "Email already registered") return res.status(409).json({ error: err.message });
    // …and if SendGrid was down, the user WAS created but we return a 500 and
    // the client retries, hitting the 409 path. Nobody noticed at review time.
    res.status(500).json({ error: "Internal server error" });
  }
}
```

Three concrete defects, all invisible to any tool:

1. Nothing lists the failure modes. You must read every line of every callee, transitively.
2. Nothing forces the caller to handle them. Deleting the whole `catch` still compiles.
3. Adding a seventh failure mode silently produces a 500 for a case that deserved a 402.

### The TypeScript way — failure in the return type

```ts
// ── The type ─────────────────────────────────────────────────────────────────
type Result<T, E> =
  | { readonly ok: true;  readonly value: T }
  | { readonly ok: false; readonly error: E };

const ok  = <T>(value: T): Result<T, never>  => ({ ok: true,  value });
const err = <E>(error: E): Result<never, E>  => ({ ok: false, error });

// ── The domain's failure modes, as a closed union of literals ────────────────
type RegisterUserError =
  | { readonly kind: "invalid_email";     readonly email: string }
  | { readonly kind: "email_taken";       readonly email: string }
  | { readonly kind: "weak_password";     readonly minLength: number }
  | { readonly kind: "database_unavailable"; readonly operation: string };

// ── The service — the signature is now the documentation ─────────────────────
async function registerUser(
  requestBody: RegisterUserRequest,
): Promise<Result<User, RegisterUserError>> {
  if (!requestBody.email.includes("@")) {
    return err({ kind: "invalid_email", email: requestBody.email });
  }
  if (requestBody.password.length < 12) {
    return err({ kind: "weak_password", minLength: 12 });
  }

  const existing = await userRepo.findByEmail(requestBody.email);
  if (!existing.ok) return err({ kind: "database_unavailable", operation: "findByEmail" });
  if (existing.value !== null) {
    return err({ kind: "email_taken", email: requestBody.email });
  }

  const passwordHash = await hasher.hash(requestBody.password);
  const inserted = await userRepo.insert({ email: requestBody.email, passwordHash });
  if (!inserted.ok) return err({ kind: "database_unavailable", operation: "insert" });

  // Welcome email is best-effort — a failure here must NOT fail registration.
  void mailer.sendWelcome(inserted.value.email).catch(() => { /* logged inside */ });

  return ok(inserted.value);
}

// ── The caller — the compiler will not let it forget a case ──────────────────
async function handleRegister(requestBody: RegisterUserRequest): Promise<ApiResponse> {
  const result = await registerUser(requestBody);

  if (result.ok) {
    return { status: 201, body: { data: result.value } };   // result.value: User
  }

  // result.error: RegisterUserError — a closed union. Exhaustive switch:
  switch (result.error.kind) {
    case "invalid_email":
      return { status: 422, body: { code: "INVALID_EMAIL", email: result.error.email } };
    case "weak_password":
      return { status: 422, body: { code: "WEAK_PASSWORD", minLength: result.error.minLength } };
    case "email_taken":
      return { status: 409, body: { code: "EMAIL_TAKEN", email: result.error.email } };
    case "database_unavailable":
      return { status: 503, body: { code: "DB_UNAVAILABLE" } };
    default:
      return assertNever(result.error);   // add a 5th kind → this line fails to compile
  }
}

function assertNever(value: never): never {
  throw new Error(`Unhandled case: ${JSON.stringify(value)}`);
}
```

The revelation: **adding a new failure mode to `RegisterUserError` breaks the build at every call site that doesn't handle it.** That is a guarantee no amount of discipline gives you with `throw`.

---

## Syntax

```ts
// ── The core type ────────────────────────────────────────────────────────────
type Ok<T>  = { readonly ok: true;  readonly value: T };
type Err<E> = { readonly ok: false; readonly error: E };
type Result<T, E = Error> = Ok<T> | Err<E>;

// ── Constructors ─────────────────────────────────────────────────────────────
const ok  = <T>(value: T): Result<T, never> => ({ ok: true,  value });
const err = <E>(error: E): Result<never, E> => ({ ok: false, error });

ok(42);                              // Result<number, never>
err(new NotFoundError("User", 7));   // Result<never, NotFoundError>

// ── The `never` trick: why the constructors are asymmetric ───────────────────
// Result<T, never> is assignable to Result<T, AnyError> because `never` is the
// bottom type — assignable to everything. So `ok(user)` fits any Result<User, E>.
const result: Result<User, NotFoundError> = ok(currentUser);  // ✅ never widens to NotFoundError

// ── Narrowing — the discriminant is `ok` ─────────────────────────────────────
if (result.ok) {
  result.value;    // ✅ User      — `error` does not exist here
} else {
  result.error;    // ✅ NotFoundError — `value` does not exist here
}

// ── Early return is the idiomatic shape ──────────────────────────────────────
function handle(result: Result<User, NotFoundError>): string {
  if (!result.ok) return `failed: ${result.error.message}`;   // narrow + bail
  return result.value.email;                                  // result: Ok<User>
}

// ── Type guards, when you want them as functions ─────────────────────────────
const isOk  = <T, E>(r: Result<T, E>): r is Ok<T>  => r.ok;
const isErr = <T, E>(r: Result<T, E>): r is Err<E> => !r.ok;

// ── Transformations ──────────────────────────────────────────────────────────
mapResult(result, (user) => user.email);            // Result<string, NotFoundError>
mapError(result, (e) => new ApiError(e));           // Result<User, ApiError>
andThen(result, (user) => loadOrders(user.userId)); // Result<Order[], NotFoundError | DbError>

// ── Unwrapping ───────────────────────────────────────────────────────────────
unwrapOr(result, GUEST_USER);                       // User — never throws
unwrapOrElse(result, (e) => fallbackFor(e));        // User — lazy default
unwrapOrThrow(result);                              // User — throws at the boundary

// ── Combining ────────────────────────────────────────────────────────────────
combine([r1, r2, r3]);          // Result<[A, B, C], E>       — first error wins
combineAll([r1, r2, r3]);       // Result<[A, B, C], E[]>     — collect ALL errors

// ── Async ────────────────────────────────────────────────────────────────────
type AsyncResult<T, E> = Promise<Result<T, E>>;

async function loadUser(userId: number): AsyncResult<User, NotFoundError> { /* … */ }

// ── Wrapping code that throws ────────────────────────────────────────────────
const parsed = tryCatch(
  () => JSON.parse(requestBody) as unknown,
  (thrown) => new MalformedJsonError({ cause: thrown }),
);                                                   // Result<unknown, MalformedJsonError>

const rows = await tryCatchAsync(
  () => pool.query("SELECT 1"),
  (thrown) => new DatabaseError("ping", { cause: thrown }),
);                                                   // AsyncResult<QueryResult, DatabaseError>

// ── Go-style tuple alternative (see Going deeper) ────────────────────────────
type Tuple<T, E> = readonly [error: null, value: T] | readonly [error: E, value: null];
const [dbError, user] = await findUserTuple(userId);
if (dbError !== null) return; // user: User after this line
```

---

## How it works — concept by concept

### Concept 1 — Why thrown errors are invisible to the type system

This is the foundation. Everything else follows.

TypeScript's function type is `(args) => ReturnType`. There is **no third slot** for "and it can also throw these things". This was a deliberate design decision, documented in the TypeScript issue tracker as [#13219](https://github.com/microsoft/TypeScript/issues/13219) — "checked exceptions" have been proposed repeatedly and declined, largely because JavaScript's `throw` accepts *any* value and exceptions propagate through callbacks, `setTimeout`, promise chains, and event emitters in ways a static checker cannot track soundly.

The concrete consequences:

```ts
// (1) The signature is a partial lie. All three of these have the SAME type.
declare function a(userId: number): Promise<User>;   // never throws
declare function b(userId: number): Promise<User>;   // throws NotFoundError
declare function c(userId: number): Promise<User>;   // throws 4 different things
// type of a === type of b === type of c === (userId: number) => Promise<User>
```

```ts
// (2) Removing error handling is not a type error.
async function handler(userId: number): Promise<ApiResponse> {
  const user = await getUserById(userId);   // may throw — no try/catch, compiles fine
  return { status: 200, body: { data: user } };
}
```

```ts
// (3) `catch` erases the type you had at throw time.
throw new NotFoundError("User", userId);    // you KNEW the type here
// …
catch (thrown) {                            // thrown: unknown — the knowledge is gone
  if (thrown instanceof NotFoundError) { /* re-derive it by hand */ }
}
```

```ts
// (4) There is no exhaustiveness. Nothing checks that your catch covers every case.
catch (thrown) {
  if (thrown instanceof NotFoundError) return notFound();
  if (thrown instanceof ValidationError) return unprocessable();
  // ConflictError? Forgot it. Compiles. Falls through to whatever comes next.
}
```

```ts
// (5) Rejections are just as invisible, and `.catch` is typed `any`.
userRepo.findById(userId).catch((thrown) => {   // thrown: any (lib.es5.d.ts)
  return thrown.response.data.message;          // compiles, crashes at runtime
});
```

**The one place the compiler is on your side is the return type.** A `Result<T, E>` is checked, narrowed, and exhaustively switchable exactly like any other union — because it *is* one. Nothing special happens; you are simply using the part of the language that works.

A useful mental model:

| | Thrown error | Returned Result |
|---|---|---|
| Appears in the signature | No | Yes |
| Compiler knows the error type | No (`unknown`) | Yes (`E`) |
| Caller forced to handle it | No | Yes (must narrow before `.value`) |
| Exhaustiveness checking | No | Yes (`assertNever`) |
| Propagates automatically | Yes (up the stack) | No (you return it explicitly) |
| Cost per failure | Stack capture | An object literal |

Note the one genuine trade-off in that table: automatic propagation. Exceptions travel up ten frames for free; Results must be threaded through each one. That verbosity is the price of visibility, and `andThen` is how you keep it manageable.

### Concept 2 — `Result<T, E>` as a discriminated union

A discriminated union needs three things (see `42 — Discriminated unions`):

1. A **common property** on every member — here, `ok`.
2. That property must be a **literal type** — here, `true` and `false`.
3. The members must otherwise be **distinguishable** — here, `value` vs `error`.

```ts
/** The success arm. */
export type Ok<T> = {
  readonly ok: true;      // ← literal `true`, NOT `boolean`
  readonly value: T;
};

/** The failure arm. */
export type Err<E> = {
  readonly ok: false;     // ← literal `false`
  readonly error: E;
};

/**
 * A computation that either produced a T or failed with an E.
 * `E = Error` is a convenience default; prefer an explicit error type.
 */
export type Result<T, E = Error> = Ok<T> | Err<E>;
```

**Why `readonly` on every field.** A Result is a value, not a mutable container. `readonly` stops `result.ok = true` from being used to "fix" a failure, and lets the compiler keep the literal types stable. See `65 — readonly and immutability patterns`.

**Why `ok: true` and not `ok: boolean`.** This is the single most common way to break the pattern:

```ts
// ❌ `ok` is widened to boolean — narrowing stops working
interface BrokenResult<T, E> {
  ok: boolean;
  value?: T;
  error?: E;
}
const r: BrokenResult<User, Error> = { ok: true, value: currentUser };
if (r.ok) {
  r.value.email;   // ❌ 'r.value' is possibly 'undefined' — the compiler learned nothing
}

// ✅ Two separate shapes, discriminated by a literal
const r2: Result<User, Error> = ok(currentUser);
if (r2.ok) {
  r2.value.email;  // ✅ User — guaranteed present
}
```

**Why not a class?** You *can* implement Result as a class with methods (`result.map(...)`, `result.unwrapOr(...)`). Chaining reads nicer. But a plain object union has three advantages that matter on a backend:

- It is **structurally typed** — a Result from one package matches a Result from another, no `instanceof`, no duplicate-copy problems (see `60 — Error handling`, "instanceof fails across realms").
- It is **JSON-serializable** — you can put a Result on a queue, in a cache, or in a worker-thread message and it survives intact.
- It has **zero prototype cost** — `{ ok: true, value }` is a plain object literal; V8 gives it a hidden class and moves on.

**Naming.** `ok`/`value`/`error` is the most common convention in the TypeScript ecosystem (it matches `neverthrow`'s `isOk()`/`value`/`error` and Rust's `Ok`/`Err`). Alternatives you will see: `success`/`data`/`error` (the fetch-wrapper convention), `_tag: "Right" | "Left"` (fp-ts / Either). Pick one and never mix them — a codebase with three Result shapes is worse than one with none.

**Deriving the payload types.** Two helper conditional types (see `44 — Conditional types`) pay for themselves immediately:

```ts
/** Extract the success type out of a Result. */
export type OkType<R>  = R extends Ok<infer T>  ? T : never;

/** Extract the error type out of a Result. */
export type ErrType<R> = R extends Err<infer E> ? E : never;

type LoadUser = Result<User, NotFoundError | DatabaseError>;
type Loaded   = OkType<LoadUser>;    // User
type Failures = ErrType<LoadUser>;   // NotFoundError | DatabaseError
```

### Concept 3 — The `ok()` / `err()` constructors and the `never` trick

```ts
export const ok  = <T>(value: T): Result<T, never> => ({ ok: true,  value });
export const err = <E>(error: E): Result<never, E> => ({ ok: false, error });
```

Look at those return types. `ok()` returns `Result<T, **never**>` and `err()` returns `Result<**never**, E>`. This asymmetry is deliberate and it is the whole reason the constructors are pleasant to use.

**Why.** `never` is TypeScript's *bottom type*: it is assignable to every other type, and no value has it. So:

```ts
// Result<User, never> is assignable to Result<User, NotFoundError>
//   because Ok<User> | Err<never>  ⊆  Ok<User> | Err<NotFoundError>
function loadUser(userId: number): Result<User, NotFoundError> {
  const found = usersById.get(userId);
  if (!found) return err(new NotFoundError("User", userId));  // Result<never, NotFoundError> ✅
  return ok(found);                                            // Result<User, never>        ✅
}
```

Both `return` statements type-check against the declared `Result<User, NotFoundError>` without a single type argument written by hand. If `ok` had been typed `<T, E>(value: T): Result<T, E>`, `E` would be unbound at every call site and infer as `unknown`, which does *not* fit `NotFoundError`.

**Contrast with the naive version:**

```ts
// ❌ E is unconstrained and uninferrable — infers `unknown`
const badOk = <T, E>(value: T): Result<T, E> => ({ ok: true, value });
const r: Result<User, NotFoundError> = badOk(currentUser);
//    ❌ Type 'Result<User, unknown>' is not assignable to 'Result<User, NotFoundError>'
```

**Two useful extras.** A `void` success is common (a delete that returns nothing), so give it a name:

```ts
/** For operations that succeed with no payload. */
export const okVoid = (): Result<void, never> => ({ ok: true, value: undefined });

// deleteSession(sessionId) → Result<void, SessionNotFound>
```

And when the error is only ever a literal string tag, skip the object entirely:

```ts
// A tiny, extremely readable style for small domains:
type CouponError = "expired" | "already_used" | "min_spend_not_met";
declare function applyCoupon(code: string, cartTotal: number): Result<Discount, CouponError>;

const applied = applyCoupon("SUMMER25", 4999);
if (!applied.ok) {
  switch (applied.error) {          // narrows string literals directly
    case "expired":            return { status: 410, body: { code: "COUPON_EXPIRED" } };
    case "already_used":       return { status: 409, body: { code: "COUPON_USED" } };
    case "min_spend_not_met":  return { status: 422, body: { code: "MIN_SPEND" } };
  }
}
```

Use string tags when the error carries no data; use tagged objects (`{ kind: "…", …fields }`) the moment it needs to carry a field like `retryAfterSeconds`.

### Concept 4 — Typed error unions per operation

This is where Result earns its keep. **Each operation declares its own closed set of failures**, and the compiler composes them for you.

```ts
// ── One tagged object per failure mode, discriminated on `kind` ───────────────

export type UserNotFound      = { readonly kind: "user_not_found";      readonly userId: number };
export type EmailTaken        = { readonly kind: "email_taken";         readonly email: string };
export type WeakPassword      = { readonly kind: "weak_password";       readonly minLength: number };
export type InsufficientFunds = { readonly kind: "insufficient_funds";  readonly balanceCents: number; readonly requiredCents: number };
export type CardDeclined      = { readonly kind: "card_declined";       readonly declineCode: string };
export type DatabaseDown      = { readonly kind: "database_down";       readonly operation: string; readonly cause: unknown };
export type GatewayTimeout    = { readonly kind: "gateway_timeout";     readonly service: "stripe" | "sendgrid"; readonly elapsedMs: number };

// ── Each operation lists ONLY what it can actually produce ────────────────────

declare function findUser(userId: number):
  Promise<Result<User, UserNotFound | DatabaseDown>>;

declare function createUser(requestBody: CreateUserRequest):
  Promise<Result<User, EmailTaken | WeakPassword | DatabaseDown>>;

declare function chargeUser(userId: number, amountCents: number):
  Promise<Result<Charge, UserNotFound | InsufficientFunds | CardDeclined | GatewayTimeout>>;
```

Note that `chargeUser` **cannot** return `EmailTaken`. That is not documentation — it is a compile-time guarantee, and the caller's `switch` only has to handle four cases, not seven.

**Unions compose automatically.** When you chain operations, the error unions add up and TypeScript computes the result:

```ts
type CheckoutError =
  | ErrType<Awaited<ReturnType<typeof findUser>>>       // UserNotFound | DatabaseDown
  | ErrType<Awaited<ReturnType<typeof chargeUser>>>;    // + InsufficientFunds | CardDeclined | GatewayTimeout
// → UserNotFound | DatabaseDown | InsufficientFunds | CardDeclined | GatewayTimeout
```

You rarely write that by hand — `andThen` (Concept 6) does it — but it is worth seeing that the union arithmetic is real.

**Mapping the union to HTTP, exhaustively.** One function, one place, checked by the compiler:

```ts
type AnyDomainError =
  | UserNotFound | EmailTaken | WeakPassword
  | InsufficientFunds | CardDeclined | DatabaseDown | GatewayTimeout;

interface ApiErrorBody {
  readonly code: string;
  readonly message: string;
  readonly details?: Readonly<Record<string, unknown>>;
}

function toApiError(domainError: AnyDomainError): { status: number; body: ApiErrorBody } {
  switch (domainError.kind) {
    case "user_not_found":
      return { status: 404, body: { code: "USER_NOT_FOUND", message: `User ${domainError.userId} does not exist` } };

    case "email_taken":
      return { status: 409, body: { code: "EMAIL_TAKEN", message: "Email already registered",
                                    details: { email: domainError.email } } };

    case "weak_password":
      return { status: 422, body: { code: "WEAK_PASSWORD", message: `Password must be at least ${domainError.minLength} characters` } };

    case "insufficient_funds":
      return { status: 402, body: { code: "INSUFFICIENT_FUNDS", message: "Not enough balance",
                                    details: { balanceCents: domainError.balanceCents,
                                               requiredCents: domainError.requiredCents } } };

    case "card_declined":
      return { status: 402, body: { code: "CARD_DECLINED", message: "Card was declined",
                                    details: { declineCode: domainError.declineCode } } };

    case "database_down":
      // Never leak `cause` to the client — it can contain SQL and connection strings.
      return { status: 503, body: { code: "SERVICE_UNAVAILABLE", message: "Please retry shortly" } };

    case "gateway_timeout":
      return { status: 504, body: { code: "GATEWAY_TIMEOUT", message: `${domainError.service} did not respond` } };

    default:
      // Add an eighth error kind → this line stops compiling. That is the point.
      return assertNever(domainError);
  }
}

function assertNever(value: never): never {
  throw new Error(`Unhandled domain error: ${JSON.stringify(value)}`);
}
```

**Grouping errors by trait.** Because the errors are plain unions you can slice them with utility types (see `32 — Utility types` and `44 — Conditional types`):

```ts
/** Every error kind that means "retry might help". */
type RetryableError = Extract<AnyDomainError, { kind: "database_down" | "gateway_timeout" }>;

function isRetryable(domainError: AnyDomainError): domainError is RetryableError {
  return domainError.kind === "database_down" || domainError.kind === "gateway_timeout";
}

/** Every error kind that is the client's fault (4xx). */
type ClientFault = Exclude<AnyDomainError, RetryableError>;
```

### Concept 5 — Narrowing: how you actually consume a Result

There are four idioms. Use the one that fits the shape of the function.

**(a) Early return — the workhorse.** Reads top-to-bottom, no nesting:

```ts
async function getUserProfile(userId: number): Promise<Result<UserProfile, ProfileError>> {
  const userResult = await findUser(userId);
  if (!userResult.ok) return userResult;      // ← propagate; type is Err<UserNotFound|DatabaseDown>
  const user = userResult.value;              // ✅ User

  const prefsResult = await findPreferences(user.userId);
  if (!prefsResult.ok) return prefsResult;
  const preferences = prefsResult.value;

  return ok({ user, preferences });
}
```

Note `return userResult;` — not `return err(userResult.error)`. Inside the `if (!userResult.ok)` branch, `userResult` is already narrowed to `Err<…>`, which is assignable to the wider return type. One fewer allocation and one fewer chance to mistype.

**(b) `if (result.ok) { … } else { … }` — when both branches do work:**

```ts
function describe(result: Result<User, UserNotFound>): string {
  if (result.ok) {
    return `Found ${result.value.email}`;        // result.value ✅, result.error ❌
  } else {
    return `Missing user ${result.error.userId}`; // result.error ✅, result.value ❌
  }
}
```

**(c) A `switch` on the error's discriminant — the exhaustive form** (shown in Concept 4).

**(d) Unwrapping with a default — when the failure genuinely does not matter:**

```ts
/** Return the value, or the supplied fallback. Never throws. */
export function unwrapOr<T, E>(result: Result<T, E>, fallback: T): T {
  return result.ok ? result.value : fallback;
}

/** Lazy version — the fallback is only computed on failure, and can see the error. */
export function unwrapOrElse<T, E>(result: Result<T, E>, onError: (error: E) => T): T {
  return result.ok ? result.value : onError(result.error);
}

const theme    = unwrapOr(await loadThemePreference(userId), "system");
const displayName = unwrapOrElse(await loadUser(userId), () => "Anonymous");
```

**And the escape hatch you should use exactly once per request** — at the transport boundary:

```ts
/**
 * Convert a Result back into an exception. ONLY call this where an exception is
 * the right mechanism: inside an Express route whose errors are caught by the
 * error middleware, or in a test.
 */
export function unwrapOrThrow<T, E>(result: Result<T, E>): T {
  if (result.ok) return result.value;
  if (result.error instanceof Error) throw result.error;
  throw new Error(`Operation failed: ${JSON.stringify(result.error)}`, { cause: result.error });
}
```

**A guard against the most dangerous mistake:** `if (result)` is always `true`, because both arms are objects.

```ts
const result = await findUser(userId);

// ❌ ALWAYS truthy — an Err object is still an object. Silent, total failure.
if (result) { /* … */ }

// ❌ Same bug, spelled differently
if (!result) return;

// ✅ Check the discriminant
if (!result.ok) return result;
```

This is the number one Result bug in real codebases, and it is invisible in review. A lint rule (`@typescript-eslint/strict-boolean-expressions`) catches it; so does never letting `Result` be `T | undefined` in the first place.

### Concept 6 — Mapping and chaining: `mapResult`, `mapError`, `andThen`

Early returns are fine, but five in a row is noise. Three combinators remove almost all of it.

**`mapResult` — transform the success, leave the error alone.**

```ts
/**
 * Apply `transform` to the value if the Result is Ok; pass an Err through unchanged.
 * The error type E is preserved exactly.
 */
export function mapResult<T, U, E>(
  result: Result<T, E>,
  transform: (value: T) => U,
): Result<U, E> {
  return result.ok ? ok(transform(result.value)) : result;
  //                                               ^^^^^^ already Err<E>; reuse the object
}

const emailResult = mapResult(await findUser(userId), (user) => user.email);
// Result<string, UserNotFound | DatabaseDown>
```

`transform` is only called on success. That is the point: you write the happy path once and the failure path zero times.

**`mapError` — transform the error, leave the value alone.** This is how you translate between layers:

```ts
export function mapError<T, E, F>(
  result: Result<T, E>,
  transform: (error: E) => F,
): Result<T, F> {
  return result.ok ? result : err(transform(result.error));
}

// The repository speaks Postgres. The service must not.
const userResult = mapError(
  await userRepo.findById(userId),
  (repoError): DatabaseDown => ({ kind: "database_down", operation: "findById", cause: repoError }),
);
```

**`andThen` — chain an operation that itself returns a Result.** This is the important one. (Rust calls it `and_then`; Haskell calls it `>>=`; `neverthrow` calls it `.andThen()`.)

```ts
/**
 * Run `next` on the value if Ok. `next` returns its OWN Result, and the error
 * types UNION together — that is where the composition happens.
 */
export function andThen<T, U, E, F>(
  result: Result<T, E>,
  next: (value: T) => Result<U, F>,
): Result<U, E | F> {
  return result.ok ? next(result.value) : result;
}
```

Watch the error type accumulate:

```ts
declare function parseUserId(raw: string): Result<number, InvalidUserId>;
declare function findUserSync(userId: number): Result<User, UserNotFound>;
declare function assertActive(user: User): Result<ActiveUser, UserSuspended>;

const activeUser = andThen(
  andThen(parseUserId(rawUserId), findUserSync),
  assertActive,
);
// Result<ActiveUser, InvalidUserId | UserNotFound | UserSuspended>
//                    ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ computed, not written
```

**Why not `mapResult` there?** Because `findUserSync` returns a `Result`, so `mapResult` would give you `Result<Result<User, UserNotFound>, InvalidUserId>` — a nested Result, which is almost never what you want. `andThen` flattens. Rule of thumb:

- The callback returns a **plain value** → `mapResult`
- The callback returns a **Result** → `andThen`

**Nesting gets ugly fast, so add a pipeline helper.** Two options:

```ts
// (a) A tiny fluent wrapper — nice ergonomics, one allocation per step.
class ResultPipeline<T, E> {
  constructor(private readonly result: Result<T, E>) {}

  map<U>(transform: (value: T) => U): ResultPipeline<U, E> {
    return new ResultPipeline(mapResult(this.result, transform));
  }

  andThen<U, F>(next: (value: T) => Result<U, F>): ResultPipeline<U, E | F> {
    return new ResultPipeline(andThen(this.result, next));
  }

  mapError<F>(transform: (error: E) => F): ResultPipeline<T, F> {
    return new ResultPipeline(mapError(this.result, transform));
  }

  /** Exit the pipeline. */
  unwrap(): Result<T, E> { return this.result; }
}

export const pipeline = <T, E>(result: Result<T, E>) => new ResultPipeline(result);

const finalResult = pipeline(parseUserId(rawUserId))
  .andThen(findUserSync)
  .andThen(assertActive)
  .map((user) => user.email)
  .unwrap();
// Result<string, InvalidUserId | UserNotFound | UserSuspended>
```

```ts
// (b) Or skip combinators entirely for long chains — early returns stay readable.
function loadActiveUserEmail(rawUserId: string): Result<string, InvalidUserId | UserNotFound | UserSuspended> {
  const idResult = parseUserId(rawUserId);
  if (!idResult.ok) return idResult;

  const userResult = findUserSync(idResult.value);
  if (!userResult.ok) return userResult;

  const activeResult = assertActive(userResult.value);
  if (!activeResult.ok) return activeResult;

  return ok(activeResult.value.email);
}
```

Both are correct. (b) debugs better (real line numbers, breakpoints land where you expect); (a) reads better when every step is a one-liner. Most production codebases end up using (b) inside service methods and (a) for small pure transformations.

**`orElse` — recover from a failure with a fallback operation:**

```ts
export function orElse<T, E, F>(
  result: Result<T, E>,
  recover: (error: E) => Result<T, F>,
): Result<T, F> {
  return result.ok ? result : recover(result.error);
}

// Try the read replica; on failure, fall back to the primary.
const rows = orElse(
  await queryReplica(sql),
  (replicaError) => {
    logger.warn("Replica read failed, falling back to primary", { replicaError });
    return queryPrimarySync(sql);
  },
);
```

### Concept 7 — Combining many Results

Two distinct semantics, and choosing the wrong one is a real bug.

**`combine` — fail fast, first error wins.** Use for *dependent* steps where continuing is pointless:

```ts
/**
 * Turn an array/tuple of Results into a Result of the array/tuple.
 * Returns the FIRST error encountered; later results are ignored.
 */
export function combine<T extends readonly Result<unknown, unknown>[]>(
  results: T,
): Result<{ -readonly [K in keyof T]: OkType<T[K]> }, ErrType<T[number]>> {
  const values: unknown[] = [];
  for (const result of results) {
    if (!result.ok) return result as Err<ErrType<T[number]>>;
    values.push(result.value);
  }
  return ok(values) as Ok<{ -readonly [K in keyof T]: OkType<T[K]> }>;
}

const combined = combine([
  parseUserId(req.params.userId),   // Result<number, InvalidUserId>
  parseLimit(req.query.limit),      // Result<number, InvalidLimit>
  parseCursor(req.query.cursor),    // Result<string | null, InvalidCursor>
] as const);
// Result<[number, number, string | null], InvalidUserId | InvalidLimit | InvalidCursor>

if (!combined.ok) return toApiError(combined.error);
const [userId, limit, cursor] = combined.value;   // ✅ fully typed tuple
```

The mapped type `{ -readonly [K in keyof T]: OkType<T[K]> }` maps a tuple of Results to a tuple of their success types, stripping `readonly` (see `43 — Mapped types`). The two `as` casts are unavoidable — the compiler cannot verify a loop that builds a tuple positionally — and they are contained inside one audited helper, which is exactly where casts belong.

**`combineAll` — collect every error.** Use for *independent* steps, above all **form validation**, where reporting only the first bad field is a terrible UX:

```ts
/** Collect ALL errors. Only returns Ok if every input is Ok. */
export function combineAll<T, E>(results: readonly Result<T, E>[]): Result<T[], E[]> {
  const values: T[] = [];
  const errors: E[] = [];

  for (const result of results) {
    if (result.ok) values.push(result.value);
    else errors.push(result.error);
  }

  return errors.length > 0 ? err(errors) : ok(values);
}

const fieldResults = [
  validateEmail(requestBody.email),
  validatePassword(requestBody.password),
  validateRole(requestBody.role),
  validateTimezone(requestBody.timezone),
];

const validated = combineAll(fieldResults);
if (!validated.ok) {
  // ✅ ALL four field errors at once — one round trip for the client
  return { status: 422, body: { code: "VALIDATION_FAILED", errors: validated.error } };
}
```

**A record-shaped variant is usually what a form actually wants:**

```ts
/** Validate an object field-by-field, returning a per-field error map. */
export function combineRecord<T extends Record<string, unknown>, E>(
  results: { [K in keyof T]: Result<T[K], E> },
): Result<T, Partial<Record<keyof T, E>>> {
  const values  = {} as T;
  const errors  = {} as Partial<Record<keyof T, E>>;
  let hasErrors = false;

  for (const key of Object.keys(results) as (keyof T)[]) {
    const result = results[key];
    if (result.ok) values[key] = result.value;
    else { errors[key] = result.error; hasErrors = true; }
  }

  return hasErrors ? err(errors) : ok(values);
}

const form = combineRecord({
  email:    validateEmail(requestBody.email),       // Result<string, string>
  password: validatePassword(requestBody.password), // Result<string, string>
  role:     validateRole(requestBody.role),         // Result<Role, string>
});

if (!form.ok) {
  return { status: 422, body: { fields: form.error } };
  // { fields: { email: "must contain @", role: "must be admin|member|guest" } }
}
const { email, password, role } = form.value;   // ✅ all present, all typed
```

**`partition` — when you want both halves.** Bulk imports need this constantly:

```ts
export function partition<T, E>(
  results: readonly Result<T, E>[],
): { readonly values: T[]; readonly errors: E[] } {
  const values: T[] = [];
  const errors: E[] = [];
  for (const result of results) {
    if (result.ok) values.push(result.value);
    else errors.push(result.error);
  }
  return { values, errors };
}

// Import 10,000 CSV rows: insert the 9,987 good ones, report the 13 bad ones.
const { values: validRows, errors: rowErrors } = partition(csvLines.map(parseCsvRow));
await userRepo.bulkInsert(validRows);
return { status: 207, body: { imported: validRows.length, failed: rowErrors } };
```

### Concept 8 — Async Result and `tryCatch`

**`AsyncResult<T, E>` is just an alias.** There is no magic:

```ts
export type AsyncResult<T, E> = Promise<Result<T, E>>;
```

Crucially, an `AsyncResult` **should never reject**. The whole contract is "failure arrives in the resolved value". A function typed `AsyncResult<User, UserNotFound>` that also rejects with a `TypeError` has broken its promise (literally), and every caller that trusted the signature now has an unhandled rejection.

That means every `await` of throwing code must be wrapped. Hence `tryCatch`.

**`tryCatch` — the bridge from the throwing world to the Result world:**

```ts
/**
 * Run a synchronous function that may throw, and capture the outcome as a Result.
 * `onThrow` maps the caught `unknown` into YOUR error type — this is where the
 * type system is repaired.
 */
export function tryCatch<T, E>(
  operation: () => T,
  onThrow: (thrown: unknown) => E,
): Result<T, E> {
  try {
    return ok(operation());
  } catch (thrown: unknown) {
    return err(onThrow(thrown));
  }
}

/** Async variant. Note the `await` INSIDE the try — see 60 for why that matters. */
export async function tryCatchAsync<T, E>(
  operation: () => Promise<T>,
  onThrow: (thrown: unknown) => E,
): AsyncResult<T, E> {
  try {
    return ok(await operation());     // ← the await must be here, not on the return
  } catch (thrown: unknown) {
    return err(onThrow(thrown));
  }
}
```

Usage — every throwing third-party call gets exactly one wrapper:

```ts
// JSON parsing
const parsed = tryCatch(
  () => JSON.parse(requestBody) as unknown,
  (thrown): MalformedJson => ({ kind: "malformed_json", detail: String(thrown) }),
);

// Database
const rows = await tryCatchAsync(
  () => pool.query<UserRow>("SELECT * FROM users WHERE user_id = $1", [userId]),
  (thrown): DatabaseDown => ({ kind: "database_down", operation: "findById", cause: thrown }),
);

// JWT verification
const claims = tryCatch(
  () => jwt.verify(authToken, secret) as TokenClaims,
  (thrown): AuthError =>
    thrown instanceof jwt.TokenExpiredError
      ? { kind: "token_expired", expiredAt: thrown.expiredAt }
      : { kind: "token_invalid" },
);
```

Note the third example: `onThrow` is where you do your `instanceof` narrowing, **once**, at the edge. After that, everything downstream sees a clean typed union.

**A default `onThrow` for the lazy case:**

```ts
/** Fallback that normalizes anything into an Error. */
export const toError = (thrown: unknown): Error =>
  thrown instanceof Error
    ? thrown
    : new Error(typeof thrown === "string" ? thrown : `Non-Error thrown: ${String(thrown)}`,
                { cause: thrown });

// Result<Config, Error> — quick and honest, if less specific
const config = tryCatch(() => JSON.parse(rawConfig) as Config, toError);
```

**Parallel async Results.** Because an `AsyncResult` never rejects, `Promise.all` is safe again — no half-abandoned rejections (contrast with `60 — Error handling`, Concept 7):

```ts
/** Run N AsyncResults in parallel; fail fast on the first error. */
export async function combineAsync<T, E>(
  operations: readonly AsyncResult<T, E>[],
): AsyncResult<T[], E> {
  const results = await Promise.all(operations);  // safe: none of them reject
  return combineAll(results).ok
    ? ok(results.map((r) => (r as Ok<T>).value))
    : err((results.find((r) => !r.ok) as Err<E>).error);
}

// Real usage — three independent loads, all failures visible
const dashboard = await Promise.all([
  loadUser(userId),          // AsyncResult<User, UserNotFound | DatabaseDown>
  loadRecentOrders(userId),  // AsyncResult<Order[], DatabaseDown>
  loadUnreadCount(userId),   // AsyncResult<number, CacheUnavailable>
] as const);

const [userResult, ordersResult, unreadResult] = dashboard;

// Each one is independently inspectable — and partial failure is expressible:
return ok({
  user:        userResult.ok ? userResult.value : null,
  orders:      ordersResult.ok ? ordersResult.value : [],
  unreadCount: unreadResult.ok ? unreadResult.value : 0,   // cache down ⇒ degrade, don't fail
});
```

That last block is the real argument for Result in a backend: **graceful degradation becomes trivial.** With exceptions, "the cache is down but the page should still render" requires a `try`/`catch` around each call and a comment explaining why it is empty.

**A timeout combinator, since every network call needs one:**

```ts
export async function withTimeout<T, E>(
  operation: AsyncResult<T, E>,
  timeoutMs: number,
  onTimeout: () => E,
): AsyncResult<T, E> {
  let timer: NodeJS.Timeout | undefined;
  const timeout = new Promise<Result<T, E>>((resolve) => {
    timer = setTimeout(() => resolve(err(onTimeout())), timeoutMs);
  });
  try {
    return await Promise.race([operation, timeout]);
  } finally {
    if (timer) clearTimeout(timer);   // cleanup only — never return/throw in finally
  }
}

const charge = await withTimeout(
  chargeUser(userId, 4999),
  5_000,
  (): GatewayTimeout => ({ kind: "gateway_timeout", service: "stripe", elapsedMs: 5_000 }),
);
```

### Concept 9 — Where Result belongs, and where exceptions still win

Result is a tool, not a religion. Applying it uniformly produces a codebase where `main()` is 400 lines of `if (!x.ok)`. The productive rule is a **layer boundary rule**.

```
┌─────────────────────────────────────────────────────────────────┐
│  HTTP / gRPC / queue-consumer layer                             │
│  → EXCEPTIONS. Framework middleware catches them. One place     │
│    turns anything into a response. unwrapOrThrow() lives here.  │
├─────────────────────────────────────────────────────────────────┤
│  Service / domain layer                                         │
│  → RESULT. Business failures are expected outcomes and belong   │
│    in the signature. Exhaustive switches, typed unions.         │
├─────────────────────────────────────────────────────────────────┤
│  Repository / gateway layer                                     │
│  → RESULT at the boundary. tryCatch() wraps the driver; a raw   │
│    pg / Stripe / fetch error never escapes upward.              │
├─────────────────────────────────────────────────────────────────┤
│  Third-party libraries (pg, stripe, jwt, zod, fs)               │
│  → THROW. Not your choice. Wrap them.                           │
└─────────────────────────────────────────────────────────────────┘
```

**Use Result when the failure is:**

| Trait | Example |
|---|---|
| **Expected** — part of the business flow | `user_not_found`, `email_taken`, `coupon_expired` |
| **Actionable by the caller** | `insufficient_funds` → show a top-up screen |
| **Enumerable** — a closed, small set | 3–8 failure kinds for the operation |
| **Frequent** | validation failures on every third request |
| **Recoverable / degradable** | cache miss, replica lag, optional enrichment |

**Keep throwing when the failure is:**

| Trait | Example |
|---|---|
| **A programmer error** | `undefined is not a function`, a broken invariant |
| **Unrecoverable** | out of memory, missing required env var at boot |
| **Truly exceptional and non-local** | you want it to blow past 10 frames to the top |
| **At a framework boundary** | Express middleware already catches; Result adds nothing |
| **In a constructor** | constructors cannot return a Result (see below) |
| **A cross-cutting concern** | auth middleware rejecting a request before routing |

**Concrete: constructors and invariants must throw.**

```ts
class Money {
  private constructor(readonly amountCents: number, readonly currency: "USD" | "EUR") {}

  /** ❌ A constructor cannot return a Result — so use a static factory instead. */
  static create(amountCents: number, currency: "USD" | "EUR"): Result<Money, InvalidAmount> {
    if (!Number.isInteger(amountCents)) {
      return err({ kind: "invalid_amount", reason: "must be an integer number of cents" });
    }
    if (amountCents < 0) {
      return err({ kind: "invalid_amount", reason: "must not be negative" });
    }
    return ok(new Money(amountCents, currency));
  }

  /** ✅ An invariant violation IS a bug — throw, do not return. */
  add(other: Money): Money {
    if (other.currency !== this.currency) {
      // If this ever runs, the caller has a bug the type system should have caught.
      throw new Error(`Cannot add ${other.currency} to ${this.currency}`);
    }
    return new Money(this.amountCents + other.amountCents, this.currency);
  }
}
```

**Concrete: the boundary adapter.** This is the single most useful piece of glue — it lets services be Result-based while routes stay conventional:

```ts
/**
 * Wrap a Result-returning service call for use inside an Express route.
 * On Ok → return the value. On Err → map to an AppError and throw, so the
 * existing error middleware handles the response exactly as before.
 */
function orThrowApiError<T, E extends AnyDomainError>(result: Result<T, E>): T {
  if (result.ok) return result.value;
  const { status, body } = toApiError(result.error);
  throw new HttpError(status, body.code, body.message, body.details);
}

// The route stays two lines long:
const getUserRoute: RequestHandler = asyncRoute(async (req, res) => {
  const user = orThrowApiError(await userService.getById(Number(req.params.userId)));
  res.status(200).json({ data: user });
});
```

You get Result's compile-time guarantees inside the service *and* the framework's error plumbing at the edge. This hybrid is what most successful production adoptions actually look like — not "Result everywhere".

**A warning about half-adoption.** The worst outcome is a codebase where a function returns `Result` *and* throws. Then callers must both narrow and `try`/`catch`, and you have doubled the work. Pick a rule per layer and enforce it:

```ts
// ❌ Both channels — the caller now has two things to remember
async function findUser(userId: number): AsyncResult<User, UserNotFound> {
  const rows = await pool.query(sql, [userId]);   // ← unwrapped: can REJECT
  return rows[0] ? ok(rows[0]) : err({ kind: "user_not_found", userId });
}

// ✅ One channel — everything that can go wrong is in E
async function findUser(userId: number): AsyncResult<User, UserNotFound | DatabaseDown> {
  const queried = await tryCatchAsync(
    () => pool.query<UserRow>(sql, [userId]),
    (thrown): DatabaseDown => ({ kind: "database_down", operation: "findUser", cause: thrown }),
  );
  if (!queried.ok) return queried;
  const row = queried.value.rows[0];
  return row ? ok(toUser(row)) : err({ kind: "user_not_found", userId });
}
```

---

## Example 1 — basic

```ts
// A complete, self-contained Result module plus a small domain that uses it.
// Everything here compiles under "strict": true.

// ═══════════════════════════════════════════════════════════════════════════
// result.ts — the whole pattern in ~40 lines
// ═══════════════════════════════════════════════════════════════════════════

export type Ok<T>  = { readonly ok: true;  readonly value: T };
export type Err<E> = { readonly ok: false; readonly error: E };
export type Result<T, E = Error> = Ok<T> | Err<E>;

export const ok  = <T>(value: T): Result<T, never> => ({ ok: true,  value });
export const err = <E>(error: E): Result<never, E> => ({ ok: false, error });

export const isOk  = <T, E>(r: Result<T, E>): r is Ok<T>  => r.ok;
export const isErr = <T, E>(r: Result<T, E>): r is Err<E> => !r.ok;

export function mapResult<T, U, E>(r: Result<T, E>, f: (value: T) => U): Result<U, E> {
  return r.ok ? ok(f(r.value)) : r;
}

export function mapError<T, E, F>(r: Result<T, E>, f: (error: E) => F): Result<T, F> {
  return r.ok ? r : err(f(r.error));
}

export function andThen<T, U, E, F>(
  r: Result<T, E>,
  f: (value: T) => Result<U, F>,
): Result<U, E | F> {
  return r.ok ? f(r.value) : r;
}

export function unwrapOr<T, E>(r: Result<T, E>, fallback: T): T {
  return r.ok ? r.value : fallback;
}

export function tryCatch<T, E>(operation: () => T, onThrow: (thrown: unknown) => E): Result<T, E> {
  try { return ok(operation()); }
  catch (thrown: unknown) { return err(onThrow(thrown)); }
}

export function assertNever(value: never): never {
  throw new Error(`Unhandled variant: ${JSON.stringify(value)}`);
}

// ═══════════════════════════════════════════════════════════════════════════
// domain.ts — a tiny user lookup with three explicit failure modes
// ═══════════════════════════════════════════════════════════════════════════

interface User {
  readonly userId: number;
  readonly email:  string;
  readonly role:   "admin" | "member" | "guest";
  readonly status: "active" | "suspended";
}

/** Every way `loadActiveUser` can fail, and nothing else. */
type LoadUserError =
  | { readonly kind: "invalid_user_id"; readonly rawValue: string }
  | { readonly kind: "user_not_found";  readonly userId: number }
  | { readonly kind: "user_suspended";  readonly userId: number };

const usersById = new Map<number, User>([
  [1, { userId: 1, email: "ada@example.com",  role: "admin",  status: "active" }],
  [2, { userId: 2, email: "grace@example.com", role: "member", status: "suspended" }],
]);

// ── Step 1: parse ────────────────────────────────────────────────────────────
function parseUserId(rawValue: string): Result<number, LoadUserError> {
  const userId = Number.parseInt(rawValue, 10);
  if (!Number.isInteger(userId) || userId <= 0) {
    return err({ kind: "invalid_user_id", rawValue });
  }
  return ok(userId);
}

// ── Step 2: look up ──────────────────────────────────────────────────────────
function findUser(userId: number): Result<User, LoadUserError> {
  const user = usersById.get(userId);
  return user ? ok(user) : err({ kind: "user_not_found", userId });
}

// ── Step 3: enforce a business rule ──────────────────────────────────────────
function requireActive(user: User): Result<User, LoadUserError> {
  return user.status === "active"
    ? ok(user)
    : err({ kind: "user_suspended", userId: user.userId });
}

// ── The pipeline, written with andThen ───────────────────────────────────────
function loadActiveUser(rawUserId: string): Result<User, LoadUserError> {
  return andThen(andThen(parseUserId(rawUserId), findUser), requireActive);
}

// ── The same pipeline, written with early returns (identical behaviour) ──────
function loadActiveUserExplicit(rawUserId: string): Result<User, LoadUserError> {
  const idResult = parseUserId(rawUserId);
  if (!idResult.ok) return idResult;

  const userResult = findUser(idResult.value);
  if (!userResult.ok) return userResult;

  return requireActive(userResult.value);
}

// ═══════════════════════════════════════════════════════════════════════════
// http.ts — one exhaustive mapping from domain error to HTTP response
// ═══════════════════════════════════════════════════════════════════════════

interface ApiResponse {
  readonly status: number;
  readonly body:   Readonly<Record<string, unknown>>;
}

function handleGetUser(rawUserId: string): ApiResponse {
  const result = loadActiveUser(rawUserId);

  if (result.ok) {
    // result.value: User — `error` is not even a property here
    return { status: 200, body: { data: result.value } };
  }

  // result.error: LoadUserError — closed union, exhaustively handled
  switch (result.error.kind) {
    case "invalid_user_id":
      return { status: 400, body: { code: "INVALID_USER_ID", rawValue: result.error.rawValue } };
    case "user_not_found":
      return { status: 404, body: { code: "USER_NOT_FOUND", userId: result.error.userId } };
    case "user_suspended":
      return { status: 403, body: { code: "USER_SUSPENDED", userId: result.error.userId } };
    default:
      // Add a fourth kind to LoadUserError and THIS LINE fails to compile.
      return assertNever(result.error);
  }
}

// ═══════════════════════════════════════════════════════════════════════════
// Run it
// ═══════════════════════════════════════════════════════════════════════════

console.log(handleGetUser("1"));
// { status: 200, body: { data: { userId: 1, email: "ada@example.com", ... } } }

console.log(handleGetUser("2"));
// { status: 403, body: { code: "USER_SUSPENDED", userId: 2 } }

console.log(handleGetUser("999"));
// { status: 404, body: { code: "USER_NOT_FOUND", userId: 999 } }

console.log(handleGetUser("not-a-number"));
// { status: 400, body: { code: "INVALID_USER_ID", rawValue: "not-a-number" } }

// ── Bonus: mapResult / unwrapOr / tryCatch in action ─────────────────────────

const emailResult = mapResult(loadActiveUser("1"), (user) => user.email);
console.log(unwrapOr(emailResult, "unknown@example.com"));   // "ada@example.com"

const suspendedEmail = mapResult(loadActiveUser("2"), (user) => user.email);
console.log(unwrapOr(suspendedEmail, "unknown@example.com")); // "unknown@example.com"

const parsedBody = tryCatch(
  () => JSON.parse('{"email":"ada@example.com"}') as unknown,
  (thrown) => ({ kind: "malformed_json" as const, detail: String(thrown) }),
);
console.log(parsedBody.ok ? parsedBody.value : parsedBody.error);
// { email: "ada@example.com" }

const badBody = tryCatch(
  () => JSON.parse("{not json") as unknown,
  (thrown) => ({ kind: "malformed_json" as const, detail: String(thrown) }),
);
console.log(badBody.ok ? badBody.value : badBody.error);
// { kind: "malformed_json", detail: "SyntaxError: Unexpected token n in JSON at position 1" }
```

---

## Example 2 — real world backend use case

```ts
// A checkout flow across four layers: repository (wraps a throwing driver),
// gateway (wraps a throwing SDK), service (pure Result composition), and an
// Express route that converts the Result back into an HTTP response.

import type { Request, Response, NextFunction, RequestHandler } from "express";

// ═══════════════════════════════════════════════════════════════════════════
// 1. The Result kernel (imported from result.ts in real code)
// ═══════════════════════════════════════════════════════════════════════════

type Ok<T>  = { readonly ok: true;  readonly value: T };
type Err<E> = { readonly ok: false; readonly error: E };
type Result<T, E> = Ok<T> | Err<E>;
type AsyncResult<T, E> = Promise<Result<T, E>>;

const ok  = <T>(value: T): Result<T, never> => ({ ok: true,  value });
const err = <E>(error: E): Result<never, E> => ({ ok: false, error });

async function tryCatchAsync<T, E>(
  operation: () => Promise<T>,
  onThrow: (thrown: unknown) => E,
): AsyncResult<T, E> {
  try { return ok(await operation()); }
  catch (thrown: unknown) { return err(onThrow(thrown)); }
}

function assertNever(value: never): never {
  throw new Error(`Unhandled variant: ${JSON.stringify(value)}`);
}

// ═══════════════════════════════════════════════════════════════════════════
// 2. Domain types
// ═══════════════════════════════════════════════════════════════════════════

interface User {
  readonly userId:      number;
  readonly email:       string;
  readonly status:      "active" | "suspended" | "deleted";
  readonly countryCode: string;
}

interface Cart {
  readonly cartId:      string;
  readonly userId:      number;
  readonly items:       readonly CartItem[];
  readonly totalCents:  number;
  readonly currency:    "USD" | "EUR";
}

interface CartItem {
  readonly sku:        string;
  readonly quantity:   number;
  readonly priceCents: number;
}

interface Order {
  readonly orderId:     string;
  readonly userId:      number;
  readonly chargeId:    string;
  readonly totalCents:  number;
  readonly placedAt:    Date;
}

interface Charge {
  readonly chargeId:    string;
  readonly amountCents: number;
}

// ═══════════════════════════════════════════════════════════════════════════
// 3. Failure modes — one tagged object per thing that can actually go wrong
// ═══════════════════════════════════════════════════════════════════════════

type InfraError =
  | { readonly kind: "database_unavailable"; readonly operation: string; readonly cause: unknown }
  | { readonly kind: "gateway_unavailable";  readonly service: "stripe"; readonly cause: unknown };

type CheckoutError =
  | InfraError
  | { readonly kind: "user_not_found";      readonly userId: number }
  | { readonly kind: "user_not_active";      readonly userId: number; readonly status: "suspended" | "deleted" }
  | { readonly kind: "cart_not_found";       readonly cartId: string }
  | { readonly kind: "cart_empty";           readonly cartId: string }
  | { readonly kind: "cart_owner_mismatch";  readonly cartId: string; readonly userId: number }
  | { readonly kind: "out_of_stock";         readonly sku: string; readonly availableQuantity: number }
  | { readonly kind: "card_declined";        readonly declineCode: string }
  | { readonly kind: "amount_mismatch";      readonly expectedCents: number; readonly actualCents: number };

// ═══════════════════════════════════════════════════════════════════════════
// 4. Repository layer — the ONLY place raw driver errors are allowed to exist
// ═══════════════════════════════════════════════════════════════════════════

interface QueryResult<Row> { readonly rows: readonly Row[] }

interface Pool {
  query<Row>(sql: string, params: readonly unknown[]): Promise<QueryResult<Row>>;
}

/** Every method returns a Result. Nothing here ever rejects. */
class UserRepository {
  constructor(private readonly pool: Pool) {}

  async findById(userId: number): AsyncResult<User | null, InfraError> {
    const queried = await tryCatchAsync(
      () => this.pool.query<User>(
        `SELECT user_id AS "userId", email, status, country_code AS "countryCode"
           FROM users WHERE user_id = $1`,
        [userId],
      ),
      (cause): InfraError => ({ kind: "database_unavailable", operation: "UserRepository.findById", cause }),
    );

    if (!queried.ok) return queried;              // propagate the InfraError unchanged
    return ok(queried.value.rows[0] ?? null);     // "no row" is NOT an error at this layer
  }
}

class CartRepository {
  constructor(private readonly pool: Pool) {}

  async findById(cartId: string): AsyncResult<Cart | null, InfraError> {
    const queried = await tryCatchAsync(
      () => this.pool.query<Cart>(
        `SELECT cart_id AS "cartId", user_id AS "userId", items,
                total_cents AS "totalCents", currency
           FROM carts WHERE cart_id = $1`,
        [cartId],
      ),
      (cause): InfraError => ({ kind: "database_unavailable", operation: "CartRepository.findById", cause }),
    );

    if (!queried.ok) return queried;
    return ok(queried.value.rows[0] ?? null);
  }

  async markCheckedOut(cartId: string): AsyncResult<void, InfraError> {
    return tryCatchAsync(
      async () => { await this.pool.query("UPDATE carts SET checked_out = true WHERE cart_id = $1", [cartId]); },
      (cause): InfraError => ({ kind: "database_unavailable", operation: "CartRepository.markCheckedOut", cause }),
    );
  }
}

class OrderRepository {
  constructor(private readonly pool: Pool) {}

  async insert(draft: Omit<Order, "orderId" | "placedAt">): AsyncResult<Order, InfraError> {
    const queried = await tryCatchAsync(
      () => this.pool.query<Order>(
        `INSERT INTO orders (user_id, charge_id, total_cents)
         VALUES ($1, $2, $3)
         RETURNING order_id AS "orderId", user_id AS "userId",
                   charge_id AS "chargeId", total_cents AS "totalCents",
                   placed_at AS "placedAt"`,
        [draft.userId, draft.chargeId, draft.totalCents],
      ),
      (cause): InfraError => ({ kind: "database_unavailable", operation: "OrderRepository.insert", cause }),
    );

    if (!queried.ok) return queried;

    const inserted = queried.value.rows[0];
    if (!inserted) {
      // A RETURNING insert with no row is a BUG, not a business failure.
      // Programmer errors still throw — see Concept 9.
      throw new Error("OrderRepository.insert: INSERT ... RETURNING produced no row");
    }
    return ok(inserted);
  }
}

// ═══════════════════════════════════════════════════════════════════════════
// 5. Gateway layer — wraps a throwing third-party SDK
// ═══════════════════════════════════════════════════════════════════════════

interface StripeSdk {
  charges: {
    create(input: { amount: number; currency: string; customer: string }): Promise<{ id: string; amount: number }>;
  };
}

/** Stripe's card errors look like this at runtime. */
function isStripeCardError(thrown: unknown): thrown is { type: "StripeCardError"; decline_code: string } {
  return (
    typeof thrown === "object" && thrown !== null &&
    (thrown as { type?: unknown }).type === "StripeCardError" &&
    typeof (thrown as { decline_code?: unknown }).decline_code === "string"
  );
}

class PaymentGateway {
  constructor(private readonly stripe: StripeSdk) {}

  /**
   * All Stripe narrowing happens HERE, once. Callers see a typed union and never
   * touch `unknown` again.
   */
  async charge(
    stripeCustomerId: string,
    amountCents: number,
    currency: "USD" | "EUR",
  ): AsyncResult<Charge, Extract<CheckoutError, { kind: "card_declined" | "gateway_unavailable" }>> {
    const charged = await tryCatchAsync(
      () => this.stripe.charges.create({
        amount:   amountCents,
        currency: currency.toLowerCase(),
        customer: stripeCustomerId,
      }),
      (thrown) =>
        isStripeCardError(thrown)
          ? ({ kind: "card_declined", declineCode: thrown.decline_code } as const)
          : ({ kind: "gateway_unavailable", service: "stripe", cause: thrown } as const),
    );

    if (!charged.ok) return charged;
    return ok({ chargeId: charged.value.id, amountCents: charged.value.amount });
  }
}

// ═══════════════════════════════════════════════════════════════════════════
// 6. Service layer — pure Result composition, zero try/catch, zero `unknown`
// ═══════════════════════════════════════════════════════════════════════════

interface InventoryService {
  reserve(items: readonly CartItem[]): AsyncResult<void, Extract<CheckoutError, { kind: "out_of_stock" }> | InfraError>;
  release(items: readonly CartItem[]): AsyncResult<void, InfraError>;
}

interface CheckoutRequest {
  readonly userId:           number;
  readonly cartId:           string;
  readonly stripeCustomerId: string;
  /** The total the client displayed — guards against a stale cart. */
  readonly expectedTotalCents: number;
}

class CheckoutService {
  constructor(
    private readonly userRepo:  UserRepository,
    private readonly cartRepo:  CartRepository,
    private readonly orderRepo: OrderRepository,
    private readonly inventory: InventoryService,
    private readonly payments:  PaymentGateway,
  ) {}

  /**
   * The entire failure surface of checkout is in this signature.
   * Nine failure kinds, all enumerable, all handled by the caller or it won't build.
   */
  async checkout(request: CheckoutRequest): AsyncResult<Order, CheckoutError> {
    // ── 1. The user must exist and be active ─────────────────────────────────
    const userLookup = await this.userRepo.findById(request.userId);
    if (!userLookup.ok) return userLookup;                    // InfraError propagates
    const user = userLookup.value;
    if (user === null) return err({ kind: "user_not_found", userId: request.userId });
    if (user.status !== "active") {
      return err({ kind: "user_not_active", userId: user.userId, status: user.status });
    }

    // ── 2. The cart must exist, belong to the user, and be non-empty ─────────
    const cartLookup = await this.cartRepo.findById(request.cartId);
    if (!cartLookup.ok) return cartLookup;
    const cart = cartLookup.value;
    if (cart === null) return err({ kind: "cart_not_found", cartId: request.cartId });
    if (cart.userId !== user.userId) {
      return err({ kind: "cart_owner_mismatch", cartId: cart.cartId, userId: user.userId });
    }
    if (cart.items.length === 0) {
      return err({ kind: "cart_empty", cartId: cart.cartId });
    }

    // ── 3. The client's displayed total must match the server's ──────────────
    if (cart.totalCents !== request.expectedTotalCents) {
      return err({
        kind:          "amount_mismatch",
        expectedCents: cart.totalCents,
        actualCents:   request.expectedTotalCents,
      });
    }

    // ── 4. Reserve stock BEFORE charging ─────────────────────────────────────
    const reserved = await this.inventory.reserve(cart.items);
    if (!reserved.ok) return reserved;                        // out_of_stock | InfraError

    // ── 5. Charge. On failure, release the reservation, then report ──────────
    const charged = await this.payments.charge(
      request.stripeCustomerId,
      cart.totalCents,
      cart.currency,
    );
    if (!charged.ok) {
      // Compensating action. Its own failure must NOT mask the payment failure —
      // log it and return the original error.
      const released = await this.inventory.release(cart.items);
      if (!released.ok) {
        logger.error("Failed to release inventory after declined charge", {
          cartId: cart.cartId,
          releaseError: released.error,
        });
      }
      return charged;                                         // card_declined | gateway_unavailable
    }

    // ── 6. Persist the order ─────────────────────────────────────────────────
    const inserted = await this.orderRepo.insert({
      userId:     user.userId,
      chargeId:   charged.value.chargeId,
      totalCents: cart.totalCents,
    });
    if (!inserted.ok) {
      // Paid but not persisted — the worst state. Emit a reconciliation event.
      logger.error("Order insert failed after successful charge — RECONCILE", {
        chargeId: charged.value.chargeId,
        userId:   user.userId,
        error:    inserted.error,
      });
      return inserted;
    }

    // ── 7. Best-effort cleanup — a failure here must not fail the checkout ───
    const markedCheckedOut = await this.cartRepo.markCheckedOut(cart.cartId);
    if (!markedCheckedOut.ok) {
      logger.warn("Cart not marked checked out", { cartId: cart.cartId, error: markedCheckedOut.error });
    }

    return ok(inserted.value);
  }
}

// ═══════════════════════════════════════════════════════════════════════════
// 7. HTTP mapping — one exhaustive switch, the compiler enforces completeness
// ═══════════════════════════════════════════════════════════════════════════

interface ApiErrorResponse {
  readonly status:  number;
  readonly headers: Readonly<Record<string, string>>;
  readonly body:    { readonly code: string; readonly message: string; readonly details?: unknown };
}

function checkoutErrorToResponse(checkoutError: CheckoutError, requestId: string): ApiErrorResponse {
  const base = { headers: { "x-request-id": requestId } } as const;

  switch (checkoutError.kind) {
    case "user_not_found":
      return { ...base, status: 404,
               body: { code: "USER_NOT_FOUND", message: "User does not exist" } };

    case "user_not_active":
      return { ...base, status: 403,
               body: { code: "USER_NOT_ACTIVE", message: `Account is ${checkoutError.status}` } };

    case "cart_not_found":
      return { ...base, status: 404,
               body: { code: "CART_NOT_FOUND", message: "Cart does not exist" } };

    case "cart_owner_mismatch":
      // Deliberately a 404, not a 403 — do not confirm that someone else's cart exists.
      return { ...base, status: 404,
               body: { code: "CART_NOT_FOUND", message: "Cart does not exist" } };

    case "cart_empty":
      return { ...base, status: 422,
               body: { code: "CART_EMPTY", message: "Cart has no items" } };

    case "amount_mismatch":
      return { ...base, status: 409,
               body: { code: "AMOUNT_MISMATCH", message: "Cart total changed; please review",
                       details: { expectedCents: checkoutError.expectedCents } } };

    case "out_of_stock":
      return { ...base, status: 409,
               body: { code: "OUT_OF_STOCK", message: `SKU ${checkoutError.sku} is unavailable`,
                       details: { sku: checkoutError.sku, availableQuantity: checkoutError.availableQuantity } } };

    case "card_declined":
      return { ...base, status: 402,
               body: { code: "CARD_DECLINED", message: "Payment was declined",
                       details: { declineCode: checkoutError.declineCode } } };

    case "gateway_unavailable":
      return { ...base, status: 502, headers: { ...base.headers, "retry-after": "30" },
               body: { code: "PAYMENT_UNAVAILABLE", message: "Payment provider is unavailable" } };

    case "database_unavailable":
      // `cause` NEVER goes to the client — it may contain SQL and credentials.
      return { ...base, status: 503, headers: { ...base.headers, "retry-after": "10" },
               body: { code: "SERVICE_UNAVAILABLE", message: "Please retry shortly" } };

    default:
      return assertNever(checkoutError);   // a tenth error kind breaks the build here
  }
}

/** Infra failures deserve an error log + a metric; business failures do not. */
function isInfraError(checkoutError: CheckoutError): checkoutError is InfraError {
  return checkoutError.kind === "database_unavailable" || checkoutError.kind === "gateway_unavailable";
}

// ═══════════════════════════════════════════════════════════════════════════
// 8. The Express route — the boundary where Result becomes an HTTP response
// ═══════════════════════════════════════════════════════════════════════════

declare const checkoutService: CheckoutService;
declare const logger: {
  warn(message: string, meta?: Record<string, unknown>): void;
  error(message: string, meta?: Record<string, unknown>): void;
};
declare const metrics: { increment(name: string, tags?: Record<string, string>): void };

function asyncRoute(
  handler: (req: Request, res: Response, next: NextFunction) => Promise<void>,
): RequestHandler {
  return (req, res, next) => { handler(req, res, next).catch(next); };
}

const checkoutRoute: RequestHandler = asyncRoute(async (req, res) => {
  const requestId = (req.headers["x-request-id"] as string | undefined) ?? "unknown";

  // Input parsing throws — it is guarded by the framework's error middleware.
  const requestBody = req.body as Partial<CheckoutRequest>;
  if (
    typeof requestBody.userId !== "number" ||
    typeof requestBody.cartId !== "string" ||
    typeof requestBody.stripeCustomerId !== "string" ||
    typeof requestBody.expectedTotalCents !== "number"
  ) {
    res.status(422).json({ code: "VALIDATION_FAILED", message: "Malformed checkout request", requestId });
    return;
  }

  const result = await checkoutService.checkout({
    userId:             requestBody.userId,
    cartId:             requestBody.cartId,
    stripeCustomerId:   requestBody.stripeCustomerId,
    expectedTotalCents: requestBody.expectedTotalCents,
  });

  if (result.ok) {
    res.status(201).json({ data: result.value, requestId });
    return;
  }

  // Log level is derived from the error KIND — no string matching anywhere.
  if (isInfraError(result.error)) {
    logger.error("Checkout infrastructure failure", { requestId, error: result.error });
    metrics.increment("checkout.infra_failure", { kind: result.error.kind });
  } else {
    logger.warn("Checkout rejected", { requestId, kind: result.error.kind });
    metrics.increment("checkout.rejected", { kind: result.error.kind });
  }

  const { status, headers, body } = checkoutErrorToResponse(result.error, requestId);
  for (const [name, value] of Object.entries(headers)) res.setHeader(name, value);
  res.status(status).json(body);
});

// ═══════════════════════════════════════════════════════════════════════════
// 9. Testing — the payoff nobody mentions until they try it
// ═══════════════════════════════════════════════════════════════════════════

// With exceptions you write: await expect(fn()).rejects.toThrow(CardDeclinedError)
// — which asserts on a class, not on data, and needs a matcher for the fields.
//
// With Result you assert on a plain value:

declare function describe(name: string, fn: () => void): void;
declare function it(name: string, fn: () => Promise<void> | void): void;
declare function expect(actual: unknown): { toEqual(expected: unknown): void; toBe(expected: unknown): void };

describe("CheckoutService", () => {
  it("returns card_declined with the decline code when Stripe rejects the card", async () => {
    const result = await checkoutService.checkout({
      userId: 1, cartId: "cart_1", stripeCustomerId: "cus_declined", expectedTotalCents: 4999,
    });

    expect(result.ok).toBe(false);
    if (!result.ok) {
      // ✅ Full autocomplete on result.error, and a structural equality assertion
      expect(result.error).toEqual({ kind: "card_declined", declineCode: "insufficient_funds" });
    }
  });

  it("returns amount_mismatch when the client total is stale", async () => {
    const result = await checkoutService.checkout({
      userId: 1, cartId: "cart_1", stripeCustomerId: "cus_ok", expectedTotalCents: 100,
    });

    expect(result.ok).toBe(false);
    if (!result.ok) {
      expect(result.error).toEqual({ kind: "amount_mismatch", expectedCents: 4999, actualCents: 100 });
    }
  });
});
```

---

## Going deeper

### Go-style tuples: the other way to return errors

Go's convention is `value, err := doThing()`. Translated to TypeScript, that is a tuple union:

```ts
/** The Go-style shape. Note the labelled tuple elements — they show in tooltips. */
type Tuple<T, E> =
  | readonly [error: null, value: T]
  | readonly [error: E,    value: null];

async function findUserTuple(userId: number): Promise<Tuple<User, UserNotFound>> {
  const user = usersById.get(userId);
  if (!user) return [{ kind: "user_not_found", userId }, null];
  return [null, user];
}

// Usage — destructuring is genuinely nice
const [findError, user] = await findUserTuple(userId);
if (findError !== null) {
  return toApiError(findError);
}
user.email;   // ✅ narrowed to User
```

**It works.** Narrowing on `findError !== null` correctly narrows `user` too, because the tuple union is discriminated by the first element's type. Node's own `util.promisify`-adjacent ecosystem and libraries like `await-to-js` popularised this shape.

**Where it is worse than an object Result:**

| | Object `Result<T, E>` | Go-style tuple |
|---|---|---|
| Discriminant | `ok: true \| false` — explicit, greppable | element type `null` vs `E` — implicit |
| Narrowing on falsy values | Safe | **Broken** — see below |
| Reads at the call site | `if (!result.ok)` | `if (findError !== null)` |
| Naming | `result.value` is self-describing | you name both, every time |
| Combinators (`map`, `andThen`) | Natural | awkward — tuples do not chain |
| Serialization | Object, self-describing keys | positional array, opaque in logs |
| Extra fields (warnings, metadata) | Just add a key | impossible without a third slot |

**The falsy-value trap** — this is the one that actually bites:

```ts
const [countError, unreadCount] = await getUnreadCount(userId);

// ❌ `unreadCount` is legitimately 0 on success — this treats success as failure
if (!unreadCount) return handleFailure();

// ❌ Same class of bug with an empty string, an empty array, or `false`
const [flagError, isEnabled] = await getFeatureFlag(userId, "new_checkout");
if (!isEnabled) { /* is this "flag is off" or "lookup failed"? */ }

// ✅ You must always compare the ERROR slot, explicitly, against null
if (countError !== null) return handleFailure();
```

An object Result cannot produce this bug, because the discriminant (`ok`) is never the payload.

**Where the tuple is genuinely better:** shallow, one-off call sites where you want zero ceremony and no import — a script, a migration, a test helper. `const [parseError, config] = tryParse(raw)` reads beautifully. For a domain layer with chaining and exhaustive mapping, the object wins.

**A hybrid you occasionally see** — an object Result with tuple-like destructuring via `Symbol.iterator`. Skip it; the cleverness costs more than it saves.

### `neverthrow` — the library you probably want instead of hand-rolling

[`neverthrow`](https://github.com/supermacro/neverthrow) is the de-facto standard Result library for TypeScript. It is small (~3 kB), dependency-free, and its API is the one this document has been reimplementing:

```ts
import { ok, err, Result, ResultAsync, okAsync, errAsync, fromPromise, fromThrowable } from "neverthrow";

// ── Same constructors ────────────────────────────────────────────────────────
const good: Result<User, UserNotFound> = ok(currentUser);
const bad:  Result<User, UserNotFound> = err({ kind: "user_not_found", userId: 7 });

// ── Fluent methods instead of standalone functions ───────────────────────────
const emailResult = good
  .map((user) => user.email)                       // Result<string, UserNotFound>
  .mapErr((error) => toApiError(error))            // Result<string, ApiError>
  .andThen((email) => validateEmail(email));       // Result<Email, ApiError | InvalidEmail>

// ── Narrowing: methods, not a discriminant property ──────────────────────────
if (good.isOk())  good.value;
if (good.isErr()) good.error;

// ── Exhaustive handling in one call ──────────────────────────────────────────
const response = good.match(
  (user)  => ({ status: 200, body: { data: user } }),
  (error) => checkoutErrorToResponse(error, requestId),
);

// ── ResultAsync — a Promise<Result> you can chain WITHOUT awaiting ───────────
// This is the feature that is genuinely painful to hand-roll.
const orderResult: ResultAsync<Order, CheckoutError> = fromPromise(
  pool.query<CartRow>(sql, [cartId]),
  (cause): CheckoutError => ({ kind: "database_unavailable", operation: "findCart", cause }),
)
  .map((queryResult) => queryResult.rows[0])
  .andThen((row) => (row ? okAsync(toCart(row)) : errAsync({ kind: "cart_not_found", cartId } as const)))
  .andThen((cart) => chargeCart(cart))             // each step returns ResultAsync
  .andThen((charge) => persistOrder(charge));

const finalResult = await orderResult;             // one await, at the end

// ── fromThrowable — turn a throwing function into a Result-returning one ─────
const safeJsonParse = fromThrowable(
  JSON.parse,
  (cause): MalformedJson => ({ kind: "malformed_json", cause }),
);
const parsed = safeJsonParse(requestBody);         // Result<unknown, MalformedJson>

// ── combine / combineWithAllErrors ───────────────────────────────────────────
import { Result as R } from "neverthrow";
R.combine([r1, r2, r3]);                 // first error wins
R.combineWithAllErrors([r1, r2, r3]);    // Result<[A,B,C], E[]>
```

**When to hand-roll and when to install:**

- **Hand-roll** when you want zero dependencies, need the Result to be a plain JSON-serializable object (queue payloads, worker messages, cache entries), or are introducing the pattern gradually into an existing codebase.
- **Install `neverthrow`** when you want `ResultAsync` chaining, `.match()`, `safeTry` (its generator-based do-notation), and the ESLint plugin `eslint-plugin-neverthrow` which flags unhandled Results — the single most valuable thing the library adds.

**The critical caveat with class-based libraries:** `neverthrow`'s `Ok`/`Err` are **classes**, so a Result does **not** survive `JSON.stringify` → `JSON.parse`, and `instanceof` can fail across duplicate package copies in a monorepo (the same hazard as custom `Error` subclasses — see `60 — Error handling in TypeScript`). If you send Results across a process boundary, serialize to your own plain shape first.

**Other options in the ecosystem:** `fp-ts` / `Effect` (`Either<E, A>`, `Effect<A, E, R>`) are far more powerful and far heavier — they bring an entire functional programming vocabulary and a real learning curve. `ts-results` is a smaller Rust-faithful alternative. `oxide.ts` mirrors Rust's API closely. For a typical Node backend, plain objects or `neverthrow` cover 95% of the need.

### The unhandled-Result problem, and how to catch it

Result's weakness is the mirror of exceptions' weakness. An unhandled exception crashes loudly; an **unhandled Result is silent**:

```ts
// ❌ The Result is discarded. If it was an Err, the failure vanishes completely.
await auditLog.record({ userId, action: "checkout" });   // returns AsyncResult<void, InfraError>

// ❌ Worse: the value is used without checking
const result = await findUser(userId);
console.log(result.value);   // ❌ compile error — good, this one IS caught
```

The `.value` access *is* caught by the compiler. Discarding the whole Result is not. Three defences:

```ts
// (1) Mark the intent explicitly, so a reader knows it was deliberate
void auditLog.record({ userId, action: "checkout" });   // fire and forget, on purpose

// (2) Handle it, even if only by logging
const recorded = await auditLog.record({ userId, action: "checkout" });
if (!recorded.ok) logger.warn("Audit log write failed", { error: recorded.error });

// (3) Lint it. This is the real fix:
//     - eslint-plugin-neverthrow's `must-use-result` rule
//     - or @typescript-eslint/no-unused-expressions + a custom rule
//     - or brand the type so ignoring it is visible in review
```

A cheap branding trick for hand-rolled Results:

```ts
declare const MUST_HANDLE: unique symbol;

type Ok<T>  = { readonly ok: true;  readonly value: T; readonly [MUST_HANDLE]?: never };
type Err<E> = { readonly ok: false; readonly error: E; readonly [MUST_HANDLE]?: never };
// The brand does not enforce anything by itself, but it makes `Result` visually
// distinct in tooltips and gives a lint rule something to key on.
```

TypeScript has no `#[must_use]`. Accept that lint rules are the enforcement mechanism.

### Result and transactions

A subtle correctness issue. With exceptions, a transaction helper rolls back on `throw`. With Result, the function **returns normally** on failure — so a naive helper commits a failed operation:

```ts
// ❌ Commits even when the callback returns an Err
async function withTransaction<T, E>(
  fn: (tx: Tx) => AsyncResult<T, E>,
): AsyncResult<T, E> {
  const tx = await pool.begin();
  const result = await fn(tx);
  await tx.commit();          // 💥 commits on Err too
  return result;
}

// ✅ Inspect the Result and roll back on Err. Also guard against a thrown bug.
async function withTransaction<T, E>(
  fn: (tx: Tx) => AsyncResult<T, E>,
): AsyncResult<T, E | InfraError> {
  const tx = await pool.begin();
  try {
    const result = await fn(tx);

    if (!result.ok) {
      await tx.rollback();     // ← the line the naive version forgot
      return result;
    }

    await tx.commit();
    return result;
  } catch (thrown: unknown) {
    // A programmer error escaped the Result channel — roll back and convert.
    await tx.rollback().catch(() => { /* rollback failure must not mask the real error */ });
    return err({ kind: "database_unavailable", operation: "withTransaction", cause: thrown });
  } finally {
    tx.release();              // cleanup only — no return, no throw
  }
}
```

The same reasoning applies to any resource-scoped helper: file handles, distributed locks, circuit breakers, spans. **If it rolls back on `throw`, it must also roll back on `Err`.**

### Result across process boundaries

Because a hand-rolled Result is a plain object, it serializes:

```ts
// Worker thread returns a Result over postMessage — it survives structured clone
parentPort?.postMessage(ok({ processedRows: 4_812 }) satisfies Result<Report, ImportError>);

// A job queue payload
await queue.add("import-users", err({ kind: "malformed_csv", line: 42 }) satisfies Result<never, ImportError>);
```

Two caveats:

1. **`cause: unknown` is not serializable.** An `InfraError` holding a raw `pg` error object will lose its `message` and `stack` through `JSON.stringify` (Error fields are non-enumerable — see `60 — Error handling`). Normalize before crossing the boundary:

   ```ts
   function serializableError(infraError: InfraError): Record<string, unknown> {
     const { cause, ...rest } = infraError;
     return { ...rest, cause: cause instanceof Error
       ? { name: cause.name, message: cause.message, stack: cause.stack }
       : String(cause) };
   }
   ```

2. **`undefined` disappears through JSON.** `ok(undefined)` round-trips to `{ ok: true }` with no `value` key. If you rely on `okVoid()` across a boundary, use `null` instead.

### Performance: Result vs throw

Measurable, and it favours Result — but not by as much as folklore suggests.

- **`new Error()`** captures a structured stack trace eagerly in V8. Cost is proportional to stack depth, roughly **1–10 µs** per construction on a deep stack. Formatting `.stack` (lazy, on first access) costs more.
- **`{ ok: false, error: {...} }`** is two object literals. V8 assigns hidden classes and allocates in the young generation: **tens of nanoseconds**.
- **`throw`/`catch` unwinding** itself is cheap in modern V8, but it **deoptimizes** the surrounding function in some cases and defeats inlining.

So a validation loop over 10,000 CSV rows where 30% fail costs ~30 ms in stack captures with exceptions and effectively nothing with Result. On a request that fails once, the difference is unmeasurable.

**Conclusion:** performance is a real reason to prefer Result in **hot paths** (parsers, validators, batch processors, per-item loops) and an irrelevant one everywhere else. Choose Result for the type safety; enjoy the speed as a side effect.

### Result does not replace input validation

A common confusion. Result is a *return shape*, not a *parser*. Zod, Valibot, and friends already return something Result-shaped:

```ts
import { z } from "zod";

const CreateUserSchema = z.object({
  email:    z.string().email(),
  password: z.string().min(12),
  role:     z.enum(["admin", "member", "guest"]),
});

/** Adapt Zod's SafeParseReturnType into OUR Result — one small function, once. */
function parseCreateUser(
  requestBody: unknown,
): Result<z.infer<typeof CreateUserSchema>, { kind: "validation_failed"; fields: Record<string, string> }> {
  const parsed = CreateUserSchema.safeParse(requestBody);
  if (parsed.success) return ok(parsed.data);

  const fields: Record<string, string> = {};
  for (const issue of parsed.error.issues) {
    fields[issue.path.join(".")] = issue.message;
  }
  return err({ kind: "validation_failed", fields });
}
```

Note that Zod's own `SafeParseReturnType` is itself a discriminated union on `success` — the same pattern, a different property name. Adapt at the edge so your codebase has exactly one Result shape.

### Migrating an existing throwing codebase

You do not rewrite. You wrap, from the bottom up:

1. **Start at one repository method.** Change `findById` to return `AsyncResult<User | null, InfraError>` using `tryCatchAsync`. Its single caller adapts with `unwrapOrThrow` — everything above is unchanged.
2. **Move up to the service that calls it.** Now the service can return a Result too, and *its* caller uses `unwrapOrThrow`.
3. **Stop at the route.** Routes keep throwing into the framework's error middleware. `orThrowApiError` is the permanent seam.
4. **Never mix within one function.** A function either returns Result or throws. Both is the failure mode.
5. **Enforce with lint.** Add `eslint-plugin-neverthrow` or a custom rule that forbids `throw` inside `src/services/**`.

The migration is boring and safe precisely because `unwrapOrThrow` and `tryCatchAsync` are perfect adapters in both directions.

---

## Common mistakes

### Mistake 1 — Using `boolean` instead of literal types for the discriminant

```ts
// ❌ `ok: boolean` plus optional fields — narrowing is impossible
interface BadResult<T, E> {
  ok: boolean;
  value?: T;
  error?: E;
}

function loadUser(userId: number): BadResult<User, string> {
  const user = usersById.get(userId);
  return user ? { ok: true, value: user } : { ok: false, error: "not found" };
}

const result = loadUser(7);
if (result.ok) {
  console.log(result.value.email);
  //                 ^^^^^ ❌ 'result.value' is possibly 'undefined'
  //                 The compiler learned NOTHING from `result.ok` being true.
}

// ❌ And the type permits nonsense states that can never be caught:
const impossible: BadResult<User, string> = { ok: true };                    // no value!
const alsoBad:    BadResult<User, string> = { ok: false, value: someUser };  // both!

// ✅ Two distinct shapes, discriminated by literal types
type Ok<T>  = { readonly ok: true;  readonly value: T };
type Err<E> = { readonly ok: false; readonly error: E };
type Result<T, E> = Ok<T> | Err<E>;

function loadUserFixed(userId: number): Result<User, "not_found"> {
  const user = usersById.get(userId);
  return user ? { ok: true, value: user } : { ok: false, error: "not_found" };
}

const fixed = loadUserFixed(7);
if (fixed.ok) {
  console.log(fixed.value.email);   // ✅ User — guaranteed
  // fixed.error;                   // ❌ Property 'error' does not exist on type 'Ok<User>'
}

// And the impossible states are genuinely impossible:
// const bad: Result<User, "not_found"> = { ok: true };                   // ❌ missing 'value'
// const bad2: Result<User, "not_found"> = { ok: false, value: someUser } // ❌ 'value' not in Err
```

**Why the broken version happens:** someone writes `{ ok: true, value: user }` as an object literal typed by an `interface` with optional fields, because it looks simpler. It compiles. It just doesn't do anything.

### Mistake 2 — Truthiness-checking the Result object

```ts
const result = await findUser(userId);

// ❌ ALWAYS true — both Ok and Err are objects. Silently ships the wrong behaviour.
if (result) {
  console.log(result.value);   // ❌ compile error on .value (saved by the compiler)
}

// ❌ ALWAYS false — the early return never fires
if (!result) return { status: 404, body: {} };

// ❌ The same bug at an await boundary — a Promise is also always truthy
if (!findUser(userId)) { /* never runs */ }

// ❌ And the version that DOES compile and IS wrong: forgetting to await
const notAwaited = findUser(userId);          // Promise<Result<...>>
if (!notAwaited.ok) { /* ❌ 'ok' does not exist on Promise — caught, luckily */ }

// ✅ Always test the discriminant, on an awaited value
const awaited = await findUser(userId);
if (!awaited.ok) {
  return checkoutErrorToResponse(awaited.error, requestId);
}
awaited.value.email;   // ✅
```

Turn on `@typescript-eslint/strict-boolean-expressions` — it rejects `if (someObject)` outright and eliminates this entire class of bug.

### Mistake 3 — Returning `Result` *and* throwing from the same function

```ts
// ❌ Two failure channels. The caller must narrow AND wrap in try/catch.
async function chargeUser(userId: number, amountCents: number): AsyncResult<Charge, CardDeclined> {
  if (amountCents <= 0) {
    throw new Error("amount must be positive");     // channel 1: throw
  }

  // The SDK can reject and nothing catches it — channel 2, invisible
  const charge = await stripe.charges.create({ amount: amountCents, currency: "usd", customer: "cus_1" });

  if (charge.status === "declined") {
    return err({ kind: "card_declined", declineCode: charge.decline_code });  // channel 3: Result
  }
  return ok({ chargeId: charge.id, amountCents: charge.amount });
}

// The caller does everything right and STILL crashes:
const result = await chargeUser(userId, 4999);   // 💥 unhandled rejection if Stripe is down
if (!result.ok) { /* … */ }

// ✅ ONE channel. Everything that can go wrong is in E.
type ChargeError =
  | { readonly kind: "invalid_amount";      readonly amountCents: number }
  | { readonly kind: "card_declined";       readonly declineCode: string }
  | { readonly kind: "gateway_unavailable"; readonly cause: unknown };

async function chargeUserFixed(userId: number, amountCents: number): AsyncResult<Charge, ChargeError> {
  if (amountCents <= 0) {
    return err({ kind: "invalid_amount", amountCents });
  }

  const charged = await tryCatchAsync(
    () => stripe.charges.create({ amount: amountCents, currency: "usd", customer: "cus_1" }),
    (cause): ChargeError => ({ kind: "gateway_unavailable", cause }),
  );
  if (!charged.ok) return charged;

  if (charged.value.status === "declined") {
    return err({ kind: "card_declined", declineCode: charged.value.decline_code });
  }
  return ok({ chargeId: charged.value.id, amountCents: charged.value.amount });
}
```

**The one legitimate exception** is a *programmer error* — an impossible state, a broken invariant — which should still throw, because the caller cannot do anything useful with it and you want the alert. `OrderRepository.insert` in Example 2 does exactly this.

### Mistake 4 — Swallowing the error type by widening `E` to `Error` or `string`

```ts
// ❌ Every operation returns Result<T, Error>. You have reinvented `catch (e)`.
declare function findUser(userId: number):   AsyncResult<User, Error>;
declare function chargeCard(userId: number): AsyncResult<Charge, Error>;

const result = await findUser(userId);
if (!result.ok) {
  // What went wrong? Back to string matching — the exact thing Result was meant to fix.
  if (result.error.message.includes("not found")) return notFound();
  if (result.error.message.includes("timeout"))   return serviceUnavailable();
  return internalError();
}

// ❌ Same problem with a bare string
declare function findUserStr(userId: number): AsyncResult<User, string>;
// error: string — no structure, no exhaustiveness, no fields.

// ✅ A closed union of tagged objects — narrowable, exhaustive, carries data
type FindUserError =
  | { readonly kind: "user_not_found";       readonly userId: number }
  | { readonly kind: "database_unavailable"; readonly cause: unknown };

declare function findUserTyped(userId: number): AsyncResult<User, FindUserError>;

const typed = await findUserTyped(userId);
if (!typed.ok) {
  switch (typed.error.kind) {
    case "user_not_found":       return { status: 404, body: { userId: typed.error.userId } };
    case "database_unavailable": return { status: 503, body: {} };
    default:                     return assertNever(typed.error);
  }
}
```

**The tell:** if your error handling contains `.message.includes(...)`, your `E` is too wide.

### Mistake 5 — Using `mapResult` where you need `andThen` (nested Results)

```ts
declare function findUser(userId: number): Result<User, UserNotFound>;
declare function loadOrders(userId: number): Result<Order[], DatabaseDown>;

// ❌ mapResult with a Result-returning callback → a nested Result
const nested = mapResult(findUser(userId), (user) => loadOrders(user.userId));
// Result<Result<Order[], DatabaseDown>, UserNotFound>
//        ^^^^^^^^^^^^^^^^^^^^^^^^^^^^ two layers deep

if (nested.ok) {
  nested.value.length;   // ❌ Property 'length' does not exist on type 'Result<Order[], DatabaseDown>'
  // You now have to unwrap twice:
  if (nested.value.ok) nested.value.value.length;   // works, but this is a smell
}

// ✅ andThen flattens and unions the error types
const flat = andThen(findUser(userId), (user) => loadOrders(user.userId));
// Result<Order[], UserNotFound | DatabaseDown>

if (flat.ok) {
  flat.value.length;   // ✅ Order[]
}

// Rule:
//   callback returns a plain value  → mapResult
//   callback returns a Result       → andThen
```

### Mistake 6 — Discarding a Result you were supposed to handle

```ts
// ❌ The Result is thrown away. If the write failed, nobody will ever know.
async function completeCheckout(orderId: string, userId: number): Promise<void> {
  auditLog.record({ orderId, userId, action: "checkout" });   // AsyncResult, not awaited
  cartRepo.markCheckedOut(cartId);                            // AsyncResult, not awaited
  await notifier.sendReceipt(userId, orderId);                // awaited, Result discarded
}

// The compiler is fine with all three lines. Two of them are also unhandled
// promises, which under Node 15+ can terminate the process on rejection.

// ✅ Handle each one, and say explicitly when a failure is acceptable
async function completeCheckoutFixed(orderId: string, userId: number, cartId: string): Promise<void> {
  const audited = await auditLog.record({ orderId, userId, action: "checkout" });
  if (!audited.ok) {
    // Compliance requirement — an audit failure is NOT acceptable
    logger.error("Audit write failed", { orderId, error: audited.error });
    metrics.increment("audit.write_failed");
  }

  const marked = await cartRepo.markCheckedOut(cartId);
  if (!marked.ok) {
    logger.warn("Cart not marked checked out", { cartId, error: marked.error });   // acceptable
  }

  const notified = await notifier.sendReceipt(userId, orderId);
  if (!notified.ok) {
    logger.warn("Receipt email failed; will be retried by the outbox", { orderId, error: notified.error });
  }
}
```

Install `eslint-plugin-neverthrow` (rule `must-use-result`) or `@typescript-eslint/no-floating-promises` — between them they catch almost every instance.

### Mistake 7 — Making `E` optional or nullable

```ts
// ❌ `error?: E` reintroduces `undefined` into the error channel
type SloppyResult<T, E> = { ok: false; error?: E } | { ok: true; value: T };

const result: SloppyResult<User, FindUserError> = { ok: false };   // ✅ compiles 😖
if (!result.ok) {
  switch (result.error.kind) {   // ❌ 'result.error' is possibly 'undefined'
    // …
  }
}

// ✅ An Err ALWAYS carries an error. If there is nothing to say, use a literal.
type Result<T, E> = { readonly ok: false; readonly error: E } | { readonly ok: true; readonly value: T };
type FeatureFlagResult = Result<boolean, "lookup_failed">;   // E is a single literal — still required
```

Same for `value?: T` on the success arm. If success carries nothing, use `Result<void, E>` and `okVoid()`.

---

## Practice exercises

### Exercise 1 — easy

Build the Result kernel from scratch and use it for a coupon-redemption domain.

Requirements:

1. Define `Ok<T>`, `Err<E>`, and `Result<T, E>` as a discriminated union with `readonly` fields and literal `ok: true` / `ok: false` discriminants. Do **not** use optional properties.
2. Write the constructors `ok<T>(value: T): Result<T, never>` and `err<E>(error: E): Result<never, E>`, and explain in a comment why the `never` in each return type is what makes them assignable to any `Result<T, E>`.
3. Write the type guards `isOk` and `isErr` as user-defined type predicates.
4. Define a closed error union for coupon redemption:
   - `{ kind: "coupon_not_found"; code: string }`
   - `{ kind: "coupon_expired"; code: string; expiredAt: Date }`
   - `{ kind: "coupon_already_used"; code: string; usedByUserId: number }`
   - `{ kind: "min_spend_not_met"; requiredCents: number; cartTotalCents: number }`
5. Implement `redeemCoupon(code: string, userId: number, cartTotalCents: number): Result<{ discountCents: number }, CouponError>` against an in-memory `Map` of coupons, returning the correct error for each case.
6. Implement `toApiResponse(result: Result<{ discountCents: number }, CouponError>): { status: number; body: Record<string, unknown> }` using an **exhaustive** `switch` on `error.kind` with an `assertNever` default (410 for expired, 409 for already used, 404 for not found, 422 for min spend).
7. Call it with at least four inputs — one success and one of each failure — and log the responses.
8. Add a commented-out line showing that `result.value` does not compile before narrowing, and one showing that `if (result)` is a bug.

```ts
// Write your code here
```

### Exercise 2 — medium

Build a Result-based configuration and startup validator for a Node service, with combinators.

Requirements:

1. Reuse (or rewrite) the Result kernel plus `mapResult`, `mapError`, `andThen`, `unwrapOr`, and `tryCatch`.
2. Implement `combineRecord` with the signature
   `combineRecord<T extends Record<string, unknown>, E>(results: { [K in keyof T]: Result<T[K], E> }): Result<T, Partial<Record<keyof T, E>>>`
   that collects **all** field errors rather than stopping at the first.
3. Define per-field parsers, each returning a `Result` with a typed error `{ field: string; reason: string }`:
   - `parsePort(raw: string | undefined)` → `number` in 1..65535
   - `parseDatabaseUrl(raw: string | undefined)` → a branded `DatabaseUrl` string that must start with `postgres://`
   - `parseLogLevel(raw: string | undefined)` → `"debug" | "info" | "warn" | "error"`
   - `parseNodeEnv(raw: string | undefined)` → `"development" | "test" | "production"`
   - `parseJwtSecret(raw: string | undefined)` → a string of at least 32 characters (and it must **not** appear in the error message)
4. Implement `loadConfig(env: NodeJS.ProcessEnv): Result<AppConfig, Partial<Record<keyof AppConfig, ConfigFieldError>>>` using `combineRecord`, so a service booted with three bad env vars reports all three at once.
5. Implement `tryCatch` and use it to wrap a `JSON.parse` of an optional `FEATURE_FLAGS` env var, mapping a thrown `SyntaxError` into a `ConfigFieldError`. A missing variable is **not** an error — default to `{}`.
6. Write `startup(env: NodeJS.ProcessEnv): never | AppConfig` which, on failure, prints every field error and calls `process.exit(1)` — demonstrating the boundary rule that a fatal startup failure is where Result stops and the process ends.
7. Prove the exhaustiveness of your field-error rendering with an `assertNever`, and demonstrate `unwrapOr` by giving `LOG_LEVEL` a default of `"info"` when the variable is absent (but still an error when it is present and invalid).
8. Run it against three env objects: fully valid, three fields invalid, and one where `FEATURE_FLAGS` is malformed JSON.

```ts
// Write your code here
```

### Exercise 3 — hard

Build a complete Result-based order-fulfilment pipeline with async combinators, compensating actions, transaction handling, and type-level tests.

Requirements:

1. **Kernel** — `Result`, `AsyncResult`, `ok`, `err`, `okVoid`, `isOk`, `isErr`, `mapResult`, `mapError`, `andThen`, `orElse`, `unwrapOr`, `unwrapOrElse`, `unwrapOrThrow`, `tryCatch`, `tryCatchAsync`, `assertNever`. Plus the conditional helpers `OkType<R>` and `ErrType<R>`.

2. **Tuple-shaped combinators** — implement
   `combine<T extends readonly Result<unknown, unknown>[]>(results: T): Result<{ -readonly [K in keyof T]: OkType<T[K]> }, ErrType<T[number]>>` (fail fast) and
   `combineAll<T, E>(results: readonly Result<T, E>[]): Result<T[], E[]>` (collect all) and
   `partition<T, E>(results: readonly Result<T, E>[]): { values: T[]; errors: E[] }`.
   Prove `combine` preserves tuple positions by destructuring a heterogeneous 3-tuple.

3. **Async combinators** — `combineAsync` (parallel, fail fast), `withTimeout<T, E>(operation, timeoutMs, onTimeout)` that clears its timer in a `finally`, and `withRetry<T, E>(operation: () => AsyncResult<T, E>, options: { maxAttempts: number; baseDelayMs: number; isRetryable: (error: E) => boolean })` that retries only retryable errors with exponential backoff and jitter, and returns the **last** error on exhaustion.

4. **Layers** — model a fulfilment flow with these failure modes, each as a tagged object:
   `order_not_found`, `order_already_fulfilled`, `out_of_stock` (with `sku` and `availableQuantity`), `warehouse_unreachable`, `label_generation_failed`, `address_invalid` (with a per-field map), `database_unavailable`, `payment_capture_declined`.
   Implement a `WarehouseGateway` and `ShippingGateway` whose underlying SDKs **throw**, wrapped with `tryCatchAsync` so nothing rejects above them.

5. **Transaction safety** — implement
   `withTransaction<T, E>(fn: (tx: Tx) => AsyncResult<T, E>): AsyncResult<T, E | DatabaseUnavailable>`
   that commits only on `Ok`, rolls back on `Err`, rolls back and converts on a **thrown** exception, and releases the connection in `finally` without returning or throwing from it.

6. **Compensating actions** — in `FulfilmentService.fulfil`, reserve stock, then generate a shipping label, then capture payment, then persist. If a later step fails, release the reservation and void the label, and make sure a failure of the *compensating* action never replaces the original error (log it instead).

7. **Exhaustive HTTP mapping** — `fulfilmentErrorToResponse(error, requestId)` with an `assertNever` default, appropriate `Retry-After` headers for the two retryable kinds, and a guarantee that no `cause` field is ever included in the client body.

8. **Go-style comparison** — reimplement `findOrder` twice: once returning `Result<Order, FindOrderError>` and once returning the tuple `readonly [error: FindOrderError, value: null] | readonly [error: null, value: Order]`. Write a short commented analysis at each call site of which reads better and, crucially, construct a concrete example where the tuple version produces a bug that the object version cannot (hint: a success value of `0`, `""`, or `false`).

9. **Type-level tests** — using `// @ts-expect-error`, prove all of:
   - `result.value` does not compile before narrowing
   - `result.error` does not compile inside the `ok` branch
   - a `Result` typed with `ok: boolean` fails to narrow (write the broken version alongside)
   - `assertNever` fails to compile when a new error kind is added to the union but not to the switch
   - `mapResult` with a Result-returning callback produces a nested `Result` whose `.value` is not the inner value

10. **A `main()`** that drives the whole pipeline through at least six scenarios — full success, out of stock, warehouse timeout retried then succeeding, warehouse timeout exhausting retries, a declined payment capture triggering both compensating actions, and a thrown programmer error inside a transaction — printing the resulting HTTP responses for each.

```ts
// Write your code here
```

---

## Quick reference cheat sheet

```ts
// ── The type ────────────────────────────────────────────────────────────────
type Ok<T>  = { readonly ok: true;  readonly value: T };
type Err<E> = { readonly ok: false; readonly error: E };
type Result<T, E = Error> = Ok<T> | Err<E>;
type AsyncResult<T, E> = Promise<Result<T, E>>;   // never rejects — failures are in E

// ── Constructors (note the `never`) ─────────────────────────────────────────
const ok  = <T>(value: T): Result<T, never> => ({ ok: true,  value });
const err = <E>(error: E): Result<never, E> => ({ ok: false, error });
const okVoid = (): Result<void, never> => ({ ok: true, value: undefined });

// ── Narrowing ───────────────────────────────────────────────────────────────
if (result.ok)  result.value;      // ✅ T   — `error` does not exist
if (!result.ok) result.error;      // ✅ E   — `value` does not exist
if (result)     { }                // ❌ ALWAYS TRUE — both arms are objects

// ── Idiomatic consumption ───────────────────────────────────────────────────
if (!result.ok) return result;                       // early return + propagate
switch (result.error.kind) { /* … */ default: assertNever(result.error); }
unwrapOr(result, fallback);                          // never throws
unwrapOrThrow(result);                               // boundary only

// ── Combinators ─────────────────────────────────────────────────────────────
mapResult(r, (v) => u)            // Result<U, E>       callback returns a VALUE
andThen(r, (v) => Result<U, F>)   // Result<U, E | F>   callback returns a RESULT
mapError(r, (e) => f)             // Result<T, F>       translate between layers
orElse(r, (e) => Result<T, F>)    // Result<T, F>       recover / fall back
combine([r1, r2, r3])             // Result<[A,B,C], E>   first error wins
combineAll([r1, r2, r3])          // Result<[A,B,C], E[]> collect all errors
combineRecord({ a: r1, b: r2 })   // per-field error map — use for form validation
partition([r1, r2, r3])           // { values: T[]; errors: E[] } — bulk imports

// ── Bridging from throwing code ─────────────────────────────────────────────
tryCatch(() => JSON.parse(s), (thrown) => ({ kind: "malformed_json" as const }));
await tryCatchAsync(() => pool.query(sql), (cause) => ({ kind: "db_down" as const, cause }));

// ── Exhaustiveness ──────────────────────────────────────────────────────────
function assertNever(value: never): never {
  throw new Error(`Unhandled variant: ${JSON.stringify(value)}`);
}

// ── Go-style tuple alternative ──────────────────────────────────────────────
type Tuple<T, E> = readonly [error: null, value: T] | readonly [error: E, value: null];
const [findError, user] = await findUserTuple(userId);
if (findError !== null) return;    // ✅ compare the ERROR slot, never `if (!user)`

// ── neverthrow ──────────────────────────────────────────────────────────────
import { ok, err, Result, ResultAsync, fromPromise, fromThrowable } from "neverthrow";
result.map(f).mapErr(g).andThen(h).match(onOk, onErr);
fromPromise(promise, (cause) => myError(cause));   // ResultAsync — chains without await
```

| Question | Answer |
|---|---|
| What is a `Result`? | A discriminated union: `{ ok: true, value: T } \| { ok: false, error: E }` |
| Why not just throw? | `throws` is not in the type — the compiler cannot see it or enforce it |
| What type is `catch (e)`? | `unknown` — all type information is destroyed by the throw |
| Discriminant type | Must be literal `true`/`false`, never `boolean` |
| Why `Result<T, never>` from `ok()` | `never` is assignable to every `E`, so `ok(v)` fits any `Result<T, E>` |
| `if (result)` | Always true — both arms are objects. Use `if (result.ok)` |
| `mapResult` vs `andThen` | Value-returning callback vs Result-returning callback (flattens) |
| Fail fast vs collect all | `combine` vs `combineAll` / `combineRecord` |
| Wrapping throwing code | `tryCatch` / `tryCatchAsync` at the lowest layer that owns the SDK |
| Should `AsyncResult` reject? | Never. If it can reject, the error type is a lie |
| Where does Result belong? | Service + domain + repository boundary |
| Where do exceptions still win? | Middleware, programmer errors, constructors, unrecoverable startup |
| One function, both channels? | Never — pick one per function, enforce per layer |
| Error type too wide? | If you write `error.message.includes(...)`, `E` should be a tagged union |
| Unhandled Result | Silent — enforce with `eslint-plugin-neverthrow` `must-use-result` |
| Transactions | Must roll back on `Err`, not only on `throw` |
| Go-style tuple risk | `if (!value)` breaks for `0` / `""` / `false` — compare the error slot |
| Library | `neverthrow` for `ResultAsync` + `.match()`; hand-roll for plain-object serializability |

---

## Connected topics

- **42 — Discriminated unions** — `Result<T, E>` *is* a discriminated union; the `ok: true \| false` discriminant and `assertNever` exhaustiveness come straight from that document.
- **60 — Error handling in TypeScript** — the other half of this story: why `catch` gives you `unknown`, typed `Error` hierarchies, `cause` chaining, and the middleware boundary where Result hands off to exceptions.
- **41 — Type guards** — `isOk` / `isErr` are user-defined type predicates; `isStripeCardError` is how `tryCatch`'s `onThrow` narrows an `unknown` at the edge.
- **26 — What are generics** and **30 — Generic constraints** — `Result<T, E>`, `mapResult<T, U, E>`, and `combine<T extends readonly Result<…>[]>` are generics with constraints doing real work.
- **44 — Conditional types** — `OkType<R>` and `ErrType<R>` use `infer` to pull the payload types back out of a `Result`.
- **43 — Mapped types** — `combine`'s `{ -readonly [K in keyof T]: OkType<T[K]> }` maps a tuple of Results to a tuple of values.
- **32 — Utility types** — `Extract` and `Exclude` slice a large error union into retryable vs client-fault groups.
- **09 — Union types** and **11 — Literal types** — the typed error union per operation is the reason the pattern pays off.
- **65 — readonly and immutability patterns** — every field of a `Result` is `readonly`; a Result is a value, not a mutable container.
- **62 — strict null checks** — the same discipline applied to `null`/`undefined`; `Result` is to failure what `strictNullChecks` is to absence.
- **67 — Repository pattern** — the repository is the layer where `tryCatchAsync` converts driver exceptions into typed Results.
- **69 — enum vs union types** — why the error tags in this document are string-literal unions rather than `enum`s.
- **58 — Typing API responses** — the `ApiResponse` shape that an exhaustive `switch` over `E` produces at the boundary.
