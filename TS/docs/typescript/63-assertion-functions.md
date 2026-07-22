# 63 — Assertion functions

## What is this?

An **assertion function** is a function whose return type is written with the `asserts` keyword. It does not return a useful value — it either *throws* or it *returns normally*, and the fact that it returned normally teaches the compiler something new about your variables from that line onward.

```ts
// A normal boolean-returning type guard — you must USE the boolean in a branch:
function isUserRow(value: unknown): value is UserRow {
  return typeof value === "object" && value !== null && "email" in value;
}

// An assertion function — it narrows by SURVIVING, not by returning true:
function assertIsUserRow(value: unknown): asserts value is UserRow {
  if (typeof value !== "object" || value === null || !("email" in value)) {
    throw new TypeError("Expected a UserRow");
  }
}

declare const requestBody: unknown;

assertIsUserRow(requestBody);
requestBody.email;   // ✅ requestBody is UserRow from this statement to the end of the scope
```

There are exactly two shapes:

```ts
// 1. asserts <param> is <Type>   — narrows a specific parameter to a type
function assertIsString(value: unknown): asserts value is string { /* ... */ }

// 2. asserts <param>             — narrows a specific parameter to "truthy"
function assertTruthy(condition: unknown): asserts condition { /* ... */ }
```

Both were added in TypeScript 3.7, alongside optional chaining and nullish coalescing, specifically so that Node's `assert` module and hand-written `invariant()` helpers could actually participate in control-flow analysis instead of being invisible to it.

The mental model: a type predicate (`value is T`) is a **question** the caller must ask inside an `if`. An assertion (`asserts value is T`) is a **statement** the caller makes, after which the compiler simply believes it.

## Why does it matter?

Backend code has a hard boundary running through the middle of it. On one side, values come from the outside world: HTTP bodies, query strings, headers, `JSON.parse`, message queues, Redis, third-party webhooks, `process.env`. Those values are genuinely `unknown`. On the other side sits your domain logic, where `userId` is a `number` and `ApiResponse` has the shape it claims.

Something has to sit on that boundary and convert `unknown` into `CreateOrderRequest`. You have three options:

1. **`as CreateOrderRequest`** — a lie. Zero runtime check, zero runtime cost, and the crash happens three layers deeper with a stack trace that points at the wrong file.
2. **A type predicate + `if`** — honest, but forces every call site into a branch, and the "else" branch is almost always the same `throw` repeated everywhere.
3. **An assertion function** — one runtime check, one throw site, and the call site is a single statement that narrows the rest of the function.

```ts
// (1) Cast — compiles, no check, explodes later somewhere unrelated:
const order = requestBody as CreateOrderRequest;
await chargeCard(order.paymentToken);   // 💥 "Cannot read properties of undefined"

// (2) Predicate — correct, but this if/throw pair gets copy-pasted 40 times:
if (!isCreateOrderRequest(requestBody)) {
  throw new ApiError(400, "VALIDATION_FAILED", "Invalid order payload");
}
await chargeCard(requestBody.paymentToken);

// (3) Assertion — the check and the throw live in ONE place, forever:
assertIsCreateOrderRequest(requestBody);
await chargeCard(requestBody.paymentToken);
```

Beyond validation, assertion functions are how you write **invariants** that the compiler respects:

- `invariant(user !== null, "user must be loaded")` — narrows `user` to `User`
- `assertNever(status)` — makes a missing `switch` case a *compile* error the day someone adds a new order status
- `assert(typeof authToken === "string")` — Node's built-in, correctly typed
- `assertAuthenticated(req)` — turns `req` into `AuthenticatedRequest` for the rest of a handler

The alternative to all of these is `!` and `as`, both of which erase to nothing and give you a false sense of safety (see `62 — Strict null checks`). An assertion function has the same *ergonomics* as `!` and an actual runtime check behind it. That trade is almost always worth making.

---

## The JavaScript way vs the TypeScript way

```js
// ── JavaScript — validation and usage drift apart ─────────────────────────────

function assertValidOrder(body) {
  if (!body || typeof body !== "object") throw new Error("body must be an object");
  if (typeof body.userId !== "number")   throw new Error("userId must be a number");
  if (!Array.isArray(body.items))        throw new Error("items must be an array");
  if (body.items.length === 0)           throw new Error("items must not be empty");
}

async function createOrder(requestBody) {
  assertValidOrder(requestBody);

  // Everything below is a guess. Your editor offers no completions. Nothing
  // stops you from typing a field the validator never checked:
  const total = requestBody.items.reduce(
    (sum, item) => sum + item.unitPrice * item.quantity,   // never validated!
    0,
  );

  // Six months later someone renames `userId` to `customerId` in the validator
  // and forgets this line. It fails silently — `undefined` flows into the DB:
  return db.insert("orders", { user_id: requestBody.userId, total_cents: total });
}
```

Three separate failures, all of them normal in a JS codebase:

- **No feedback loop.** The validator says `userId` is a number; the consumer accesses `items[0].unitPrice`, which was never validated. Nothing connects them.
- **No refactoring safety.** Rename a field in one place, the other place keeps compiling and starts writing `undefined` rows.
- **No completions.** Your editor knows nothing about `requestBody` after the check, so you type field names from memory. Typos become production bugs.

The validator did real work at runtime, and *none* of that work was available to you while writing the next line.

```ts
// ── TypeScript — the runtime check and the static type are the same thing ─────

interface OrderItemInput {
  readonly sku:            string;
  readonly quantity:       number;
  readonly unitPriceCents: number;
}

interface CreateOrderRequest {
  readonly userId:       number;
  readonly items:        readonly OrderItemInput[];
  readonly couponCode:   string | null;
  readonly paymentToken: string;
}

class ValidationError extends Error {
  constructor(readonly field: string, message: string) {
    super(`${field}: ${message}`);
    this.name = "ValidationError";
  }
}

// ONE function. It throws at runtime AND it narrows at compile time.
function assertCreateOrderRequest(
  value: unknown,
): asserts value is CreateOrderRequest {
  if (typeof value !== "object" || value === null) {
    throw new ValidationError("body", "must be a JSON object");
  }
  const body = value as Record<string, unknown>;

  if (typeof body.userId !== "number" || !Number.isInteger(body.userId)) {
    throw new ValidationError("userId", "must be an integer");
  }
  if (typeof body.paymentToken !== "string" || body.paymentToken.length === 0) {
    throw new ValidationError("paymentToken", "must be a non-empty string");
  }
  if (body.couponCode !== null && typeof body.couponCode !== "string") {
    throw new ValidationError("couponCode", "must be a string or null");
  }
  if (!Array.isArray(body.items) || body.items.length === 0) {
    throw new ValidationError("items", "must be a non-empty array");
  }
  for (const [index, raw] of body.items.entries()) {
    if (typeof raw !== "object" || raw === null) {
      throw new ValidationError(`items[${index}]`, "must be an object");
    }
    const item = raw as Record<string, unknown>;
    if (typeof item.sku !== "string" || item.sku.length === 0) {
      throw new ValidationError(`items[${index}].sku`, "must be a non-empty string");
    }
    if (typeof item.quantity !== "number" || item.quantity < 1) {
      throw new ValidationError(`items[${index}].quantity`, "must be >= 1");
    }
    if (typeof item.unitPriceCents !== "number" || item.unitPriceCents < 0) {
      throw new ValidationError(`items[${index}].unitPriceCents`, "must be >= 0");
    }
  }
}

async function createOrder(requestBody: unknown): Promise<{ orderId: number }> {
  assertCreateOrderRequest(requestBody);
  // ↑ one statement. Below this line `requestBody` is CreateOrderRequest.

  const totalCents = requestBody.items.reduce(
    //                            ^? readonly OrderItemInput[]
    (sum, item) => sum + item.unitPriceCents * item.quantity,   // ✅ completions, checked
    0,
  );

  // Rename `userId` in the interface → this line becomes a compile error.
  // The validator and the consumer can no longer drift apart:
  const orderId = await db.insertOrder({
    userId:      requestBody.userId,
    totalCents,
    couponCode:  requestBody.couponCode,
  });

  return { orderId };
}
```

The satisfying part: `assertCreateOrderRequest` is the *only* place that knows how to reject a bad order body, and the `asserts` clause makes the compiler enforce that everything downstream matches the shape it validated. One function, two jobs, permanently in sync.

---

## Syntax

```ts
// ── Form 1: asserts <param> is <Type> ───────────────────────────────────────
// "if this returns, <param> has type <Type>"
function assertIsString(value: unknown): asserts value is string {
  if (typeof value !== "string") throw new TypeError("Expected string");
}

// ── Form 2: asserts <param> (bare) ──────────────────────────────────────────
// "if this returns, <param> is truthy" — narrows by truthiness, not by type
function invariant(condition: unknown, message: string): asserts condition {
  if (!condition) throw new Error(`Invariant failed: ${message}`);
}

// ── The asserted parameter MUST be a parameter name, not an expression ──────
function bad(value: unknown, other: number): asserts other is string {} // legal but useless
// function bad2(): asserts someGlobal is string {}   // ❌ must reference a parameter

// ── Rest/optional parameters cannot be asserted ─────────────────────────────
// function bad3(...values: unknown[]): asserts values is string[] {}   // ❌

// ── The asserted parameter may be any position, and may use `this` ──────────
function assertInitialized(this: Connection): asserts this is OpenConnection {
  if (this.socket === null) throw new Error("Connection not open");
}

// ── Combining with generics ─────────────────────────────────────────────────
function assertNonNull<T>(
  value: T | null | undefined,
  message: string,
): asserts value is T {
  if (value === null || value === undefined) throw new Error(message);
}

// ── Assertion methods on classes and interfaces ─────────────────────────────
interface Validator {
  assertValid(value: unknown): asserts value is ValidatedPayload;
}

class RequestValidator implements Validator {
  assertValid(value: unknown): asserts value is ValidatedPayload {
    if (!this.check(value)) throw new ValidationError("body", "invalid");
  }
  private check(value: unknown): boolean { return typeof value === "object" && value !== null; }
}

// ── ⚠️ THE BIG RULE: the CALLING reference needs an explicit type ────────────
// This works — a function declaration:
function assertOk(value: unknown): asserts value is string { /* ... */ }
declare const authToken: unknown;
assertOk(authToken);
authToken.length;                       // ✅ narrowed

// This does NOT work — an inferred const:
const assertOk2 = (value: unknown): asserts value is string => { /* ... */ };
// ❌ Assertions require every name in the call target to be declared with an
//    explicit type annotation.

// This DOES work — the same arrow with an explicit annotation:
type StringAsserter = (value: unknown) => asserts value is string;
const assertOk3: StringAsserter = (value) => {
  if (typeof value !== "string") throw new TypeError("Expected string");
};
assertOk3(authToken);
authToken.length;                       // ✅ narrowed

// ── Types used above ────────────────────────────────────────────────────────
interface ValidatedPayload { readonly kind: string }
interface Connection { socket: unknown | null }
interface OpenConnection extends Connection { socket: NodeJS.Socket }
```

---

## How it works — concept by concept

### Concept 1 — `asserts value is T` vs `value is T`

They solve the same problem — moving a runtime fact into the type system — but they hand the result to you in opposite ways.

```ts
interface AdminUser {
  readonly userId: number;
  readonly role:   "admin";
  readonly scopes: readonly string[];
}

interface RegularUser {
  readonly userId: number;
  readonly role:   "member";
}

type AppUser = AdminUser | RegularUser;

// ── Type predicate: returns a boolean you must branch on ───────────────────
function isAdmin(user: AppUser): user is AdminUser {
  return user.role === "admin";
}

// ── Assertion: returns nothing; narrows by not throwing ────────────────────
function assertIsAdmin(user: AppUser): asserts user is AdminUser {
  if (user.role !== "admin") {
    throw new ApiError(403, "FORBIDDEN", "Admin role required");
  }
}
```

Compare the call sites:

```ts
function listAllUsersWithPredicate(currentUser: AppUser): readonly AppUser[] {
  if (!isAdmin(currentUser)) {
    throw new ApiError(403, "FORBIDDEN", "Admin role required");
  }
  // currentUser: AdminUser here — narrowing came from the `if`
  return userRepo.findAll(currentUser.scopes);
}

function listAllUsersWithAssertion(currentUser: AppUser): readonly AppUser[] {
  assertIsAdmin(currentUser);
  // currentUser: AdminUser here — narrowing came from the call ITSELF
  return userRepo.findAll(currentUser.scopes);
}
```

The differences that actually matter:

| | `user is AdminUser` | `asserts user is AdminUser` |
|---|---|---|
| Returns | `boolean` | `void` (never usefully) |
| Narrows | inside the `if`/`else` branches | for the rest of the enclosing flow |
| Failure path | caller decides | baked into the function (it throws) |
| Usable in `.filter()` | ✅ yes | ❌ no |
| Usable in a ternary / `&&` | ✅ yes | ❌ no |
| Requires explicit annotation on the reference | no | **yes** |
| Right when | both branches are meaningful | the failure is always an error |

The rule of thumb: **if every call site would write the same `if (!guard) throw`, use an assertion.** If different call sites do different things on failure — return a 404 here, fall back to a default there, filter it out somewhere else — use a predicate.

You can also build one from the other, which is what most codebases end up doing:

```ts
// Write the predicate once (it composes: filter, ternary, boolean logic)...
function isAdmin(user: AppUser): user is AdminUser {
  return user.role === "admin";
}

// ...then derive the assertion from it (it reads better at 40 call sites):
function assertIsAdmin(user: AppUser): asserts user is AdminUser {
  if (!isAdmin(user)) throw new ApiError(403, "FORBIDDEN", "Admin role required");
}

const admins = allUsers.filter(isAdmin);   // ✅ AdminUser[] — only the predicate can do this
assertIsAdmin(currentUser);                // ✅ narrows a single value cleanly
```

### Concept 2 — The bare `asserts condition` form

The second form takes no `is` clause. It says: *if this function returns, the named parameter was truthy.* TypeScript then applies exactly the same narrowing it would apply inside `if (condition) { ... }`.

```ts
function invariant(condition: unknown, message: string): asserts condition {
  if (!condition) {
    throw new Error(`Invariant violation: ${message}`);
  }
}
```

That single tiny helper does a surprising amount of work, because the narrowing follows from the *expression you pass*:

```ts
declare const session: { user: User | null; expiresAt: Date | null };

invariant(session.user !== null, "session must have a user");
session.user.email;
//      ^? User   ← the `!== null` comparison was narrowed by the assertion

declare const authToken: string | undefined;
invariant(authToken !== undefined, "authToken required");
authToken.slice(7);
//        ^? string

declare const payload: unknown;
invariant(typeof payload === "object" && payload !== null, "payload must be an object");
// payload: object   ← typeof narrowing works too

declare const orderIds: readonly number[];
const firstId = orderIds[0];   // number | undefined under noUncheckedIndexedAccess
invariant(firstId !== undefined, "orderIds must not be empty");
firstId.toFixed(0);
//      ^? number

declare const user: AppUser;
invariant(user.role === "admin", "must be admin");
// ⚠️ this does NOT narrow `user` to AdminUser — see Concept 6
```

The mechanism is worth internalizing: `asserts condition` does **not** know anything about types. It knows the argument expression was truthy, and TypeScript replays its normal control-flow narrowing on that expression. Anything you could write inside `if (...)` and get narrowing from, you can pass to an `asserts condition` helper.

The bare form is also what lets you write a *generic-free*, *shape-free* helper that works on every type in the codebase. `invariant` is usually the single most-used assertion function in a mature TypeScript service, because it replaces every `!` with a real check:

```ts
// Instead of the lying non-null assertion:
const user = (await userRepo.findById(userId))!;

// ...the invariant that actually checks and produces a real error message:
const maybeUser = await userRepo.findById(userId);
invariant(maybeUser !== null, `user ${userId} not found`);
maybeUser.email;   // ✅ User — with a runtime guarantee behind it
```

### Concept 3 — Why assertion calls need an explicit type annotation

This is the rule that trips up everyone, and the error message is genuinely confusing the first time:

```ts
const assertIsString = (value: unknown): asserts value is string => {
  if (typeof value !== "string") throw new TypeError("Expected string");
};

declare const requestId: unknown;
assertIsString(requestId);
// ❌ Assertions require every name in the call target to be declared with an
//    explicit type annotation. ts(2775)
```

The function itself is fine. The **call** is rejected. Why?

Narrowing is done by TypeScript's control-flow analyser, which runs *before* and *interleaved with* type inference. When it reaches a call expression, it must decide "does this call narrow anything?" — and to answer that it must know the callee's signature. If the callee is `const assertIsString = ...`, its type comes from inferring the initializer, which may itself depend on narrowing elsewhere. That is a circularity the compiler refuses to enter.

So the rule is mechanical: **every identifier in the call target must have an explicit, declared type**, resolvable without inference.

```ts
// ✅ 1. Function declaration — hoisted, signature is explicit by construction:
function assertIsString(value: unknown): asserts value is string {
  if (typeof value !== "string") throw new TypeError("Expected string");
}
assertIsString(requestId);            // ✅

// ❌ 2. Inferred const arrow:
const assertA = (v: unknown): asserts v is string => { /* ... */ };
assertA(requestId);                   // ❌ ts(2775)

// ✅ 3. Annotated const arrow — the annotation breaks the circularity:
type StringAssertion = (value: unknown) => asserts value is string;
const assertB: StringAssertion = (v) => {
  if (typeof v !== "string") throw new TypeError("Expected string");
};
assertB(requestId);                   // ✅

// ❌ 4. Inferred const holding a function EXPRESSION (same problem):
const assertC = function (v: unknown): asserts v is string { /* ... */ };
assertC(requestId);                   // ❌ ts(2775)

// ✅ 5. Class method — methods have declared signatures:
class Guards {
  assertIsString(value: unknown): asserts value is string {
    if (typeof value !== "string") throw new TypeError("Expected string");
  }
}
const guards = new Guards();
guards.assertIsString(requestId);     // ✅ — `guards` is inferred, but its TYPE is a class type
```

The subtlety in case 5: "every name in the call target" means every name in the *path* — `guards.assertIsString` requires `guards` to have a known type, which it does (`Guards`). But this bites when you build the object literally:

```ts
// ❌ Object literal with inferred type — the METHOD is fine, the object is inferred:
const validators = {
  assertIsString(value: unknown): asserts value is string {
    if (typeof value !== "string") throw new TypeError("Expected string");
  },
};
validators.assertIsString(requestId);
// ❌ ts(2775) — `validators` has no explicit type annotation

// ✅ Annotate the object:
interface Validators {
  assertIsString(value: unknown): asserts value is string;
}
const validators2: Validators = {
  assertIsString(value) {
    if (typeof value !== "string") throw new TypeError("Expected string");
  },
};
validators2.assertIsString(requestId);   // ✅
```

Practical consequences for a real codebase:

- Write assertion helpers as **`function` declarations** in a shared `assertions.ts`. It is the path of least friction.
- If you must have a `const` (e.g. it is produced by a factory), give it an explicit function-type annotation.
- Importing works fine either way — an imported `function` declaration keeps its declared signature across module boundaries.
- Destructuring an assertion off an object **loses** the annotation unless you re-annotate:

```ts
import * as assertions from "./assertions";
assertions.assertIsString(requestId);      // ✅ namespace member, declared type

const { assertIsString: assertLocal } = assertions;
assertLocal(requestId);                    // ❌ ts(2775) — `assertLocal` is inferred

const assertLocal2: (v: unknown) => asserts v is string = assertions.assertIsString;
assertLocal2(requestId);                   // ✅ explicitly annotated
```

### Concept 4 — Control-flow effects: narrowing persists after the call

This is the whole point, and it behaves differently from a type predicate in ways that matter.

```ts
declare function assertIsDefined<T>(
  value: T | null | undefined,
  label: string,
): asserts value is T;

async function shipOrder(orderId: number): Promise<void> {
  const order = await orderRepo.findById(orderId);   // Order | null

  assertIsDefined(order, `order ${orderId}`);
  //                 ↑ narrowing takes effect on the NEXT statement

  order.items.length;         // ✅ Order
  await inventory.reserve(order.items);
  await carrier.book(order.shippingAddress);
  await orderRepo.markShipped(order.id);
  //               ^^^^^^^^ still Order — narrowing lasts to the end of the function
}
```

The narrowing is a **control-flow fact**, so it obeys all the normal control-flow rules:

```ts
declare const maybeToken: string | undefined;

function branchy(flag: boolean): void {
  if (flag) {
    invariant(maybeToken !== undefined, "token required in flag branch");
    maybeToken.length;      // ✅ string — inside the branch
  }
  // maybeToken;            // string | undefined — the branch merged back
}

function loopy(orderIds: readonly number[]): void {
  for (const id of orderIds) {
    const order = orderCache.get(id);         // Order | undefined
    invariant(order !== undefined, `order ${id} not cached`);
    order.total;                              // ✅ Order — per iteration
  }
  // Each iteration re-runs the assertion; no leakage between iterations.
}

function reassigned(): void {
  let currentOrder: Order | null = null;
  assertIsDefined(currentOrder, "order");   // (would throw at runtime — illustrative)
  currentOrder.id;                          // ✅ Order
  currentOrder = null;                      // reassignment resets the narrowing
  // currentOrder.id;                       // ❌ 'currentOrder' is possibly 'null'
}
```

Two things assertion functions do that predicates cannot:

```ts
// 1. Narrow at the TOP LEVEL of a module — no branch required:
const DATABASE_URL: string | undefined = process.env.DATABASE_URL;
invariant(DATABASE_URL !== undefined, "DATABASE_URL is required");
export const dbUrl: string = DATABASE_URL;   // ✅ module-level narrowing

// 2. Narrow SEVERAL values in a row without nesting:
declare const host: string | undefined;
declare const port: string | undefined;
declare const user: string | undefined;

invariant(host !== undefined, "DB_HOST required");
invariant(port !== undefined, "DB_PORT required");
invariant(user !== undefined, "DB_USER required");
const dsn = `postgres://${user}@${host}:${port}/app`;   // ✅ all three are string

// vs the predicate version, which pyramids:
// if (host !== undefined) { if (port !== undefined) { if (user !== undefined) { ... } } }
```

And one thing they do *not* do, which surprises people: an assertion function does **not** narrow the value inside its own body.

```ts
function assertIsUserRow(value: unknown): asserts value is UserRow {
  if (typeof value !== "object" || value === null) throw new TypeError("not an object");
  // value: object — narrowed by the `if` above, NOT by the asserts clause.
  // The asserts clause is a promise to the CALLER, not a fact inside the body.
  // value.email;   // ❌ Property 'email' does not exist on type 'object'
}
```

The `asserts` clause is a *post-condition you are promising*, not a *pre-condition you get to assume*.

### Concept 5 — The safety contract: TypeScript does not check your work

Here is the uncomfortable truth. TypeScript performs **no verification** that the body of an assertion function actually implements the assertion. An empty body compiles:

```ts
// ✅ Compiles. Checks nothing. Asserts everything. This is legal TypeScript:
function assertIsCreateOrderRequest(value: unknown): asserts value is CreateOrderRequest {
  // (nothing)
}

const garbage: unknown = "not an order at all";
assertIsCreateOrderRequest(garbage);
garbage.items.length;   // ✅ compiles. 💥 TypeError at runtime.
```

The compiler's only requirement is that the function **can** return `void` and that the asserted name is a parameter. Whether the body throws on bad input is entirely on you.

This puts assertion functions in the same trust category as `as`, `!`, and type predicates: they are **unchecked assertions of programmer knowledge**. The difference is one of *concentration*, not of *kind*:

```ts
// `as` at 60 call sites = 60 unverified claims scattered through the codebase:
const order = requestBody as CreateOrderRequest;     // ← one lie, here
const user  = row as UserRow;                        // ← another lie, there

// An assertion function = ONE unverified claim, in one file, that you review
// once, test heavily, and then trust at 60 call sites:
assertCreateOrderRequest(requestBody);               // ← trust delegated to the helper
```

That is the actual value proposition: assertion functions **centralize** the place where you lie to the compiler, and give you a natural place to make the lie true (an exhaustive runtime check, unit tests, or a schema library).

The rules that keep the contract honest:

1. **The body must throw on every input that does not match `T`.** Not return false, not log — throw. A `return` on a bad value silently corrupts every downstream type.
2. **Never write an assertion function without a runtime check.** An empty-bodied assertion is strictly worse than `as` because it looks safe.
3. **Assertion functions are the perfect unit-test target.** They are pure, they take `unknown`, and the failure mode is a thrown error. Test the reject cases, not just the accept case.
4. **Prefer generating them.** Schema libraries (zod, valibot, typebox, ajv+`json-schema-to-ts`) produce assertions whose runtime check and static type are derived from *one* source, which removes the possibility of drift. See Concept 9.
5. **Assertions must be side-effect-free apart from throwing.** A "validator" that also mutates or normalizes its input is a landmine, because narrowing tells the reader nothing changed.

### Concept 6 — What `asserts condition` cannot narrow

`asserts condition` replays control-flow narrowing on the argument expression. Control-flow narrowing has known limits, and the assertion inherits every one of them.

```ts
type OrderStatus = "pending" | "paid" | "shipped" | "cancelled";
interface Order { readonly id: number; readonly status: OrderStatus }

declare const order: Order;

// ❌ Discriminant narrowing does NOT survive the call:
invariant(order.status === "shipped", "order must be shipped");
// order.status;      // still OrderStatus — NOT narrowed to "shipped"
```

Why? `asserts condition` narrows the *parameter named `condition`*, which received the value `order.status === "shipped"` — a `boolean`. TypeScript then asks "what does knowing this boolean expression is truthy tell me?" and, for a **reference** (`order.status`) that was passed by value into another function, it can only apply narrowing to references it can track through the call. For direct comparisons of a property to a literal, it *does* work in TypeScript 4.4+ thanks to **aliased conditions and discriminants**, but only under specific conditions:

```ts
// ✅ TS 4.4+ — the condition is stored in a `const` first, then asserted:
const isShipped = order.status === "shipped";
invariant(isShipped, "order must be shipped");
// order.status;      // ⚠️ still NOT narrowed — aliased-condition narrowing works
//                    //    for `if (isShipped)`, but assertion calls narrow the
//                    //    PARAMETER, and the alias link is not re-established here.
```

The reliable rule: **`asserts condition` narrows the exact reference you compare, when that reference is a `const` or a readonly-ish property path, and the comparison is directly the argument.** In practice these work:

```ts
declare const authToken: string | undefined;
declare const currentUser: User | null;
declare const rawPayload: unknown;

invariant(authToken !== undefined, "token");        // ✅ authToken: string
invariant(currentUser !== null, "user");            // ✅ currentUser: User
invariant(typeof rawPayload === "string", "raw");   // ✅ rawPayload: string
invariant(Array.isArray(rawPayload), "raw");        // ✅ rawPayload: any[]
invariant(rawPayload instanceof Error, "raw");      // ✅ rawPayload: Error
```

And these do **not** reliably narrow the underlying object:

```ts
invariant(order.status === "shipped", "shipped");     // ❌ order not narrowed to a variant
invariant(user.role === "admin", "admin");            // ❌ user not narrowed to AdminUser
invariant(items.length > 0, "non-empty");             // ❌ items not narrowed to non-empty
invariant(a !== null && b !== null, "both");          // ⚠️ narrows neither reliably
```

When you need a union member, reach for the `is` form, which narrows the **object**, not the boolean:

```ts
function assertOrderShipped(order: Order): asserts order is Order & { status: "shipped" } {
  if (order.status !== "shipped") {
    throw new ApiError(409, "INVALID_STATE", `Order ${order.id} is ${order.status}`);
  }
}

assertOrderShipped(order);
order.status;   // ✅ "shipped"
```

The general lesson: **`asserts condition` is for nullability and `typeof` facts; `asserts x is T` is for shapes and union members.**

### Concept 7 — `assertNever` and exhaustiveness

`never` is the type with no values. If a variable's type has narrowed to `never`, every possible case has been handled. That makes `never` a compile-time exhaustiveness checker, and an assertion-flavoured helper turns it into a runtime one too.

The classic version is not technically an assertion function — it is a `never`-returning function:

```ts
// ── Version A: the never-returning helper (most common) ─────────────────────
function assertNever(value: never, context: string): never {
  throw new Error(`Unhandled ${context}: ${JSON.stringify(value)}`);
}
```

```ts
type OrderStatus = "pending" | "paid" | "shipped" | "cancelled";

function statusLabel(status: OrderStatus): string {
  switch (status) {
    case "pending":   return "Awaiting payment";
    case "paid":      return "Payment received";
    case "shipped":   return "In transit";
    case "cancelled": return "Cancelled";
    default:
      // `status` is `never` here — all four cases handled.
      return assertNever(status, "OrderStatus");
  }
}
```

Now add a fifth status to the union:

```ts
type OrderStatus = "pending" | "paid" | "shipped" | "cancelled" | "refunded";

// statusLabel now fails to compile:
// ❌ Argument of type '"refunded"' is not assignable to parameter of type 'never'.
```

That error appears in **every** switch in the codebase that forgot the new case. This is the single highest-value pattern in TypeScript for evolving domain models, and it costs four lines.

Why `: never` and not `: void`? Because a `never` return type also tells control-flow analysis that the function does not return, which makes the `default` branch satisfy the function's `string` return type without an extra `throw`:

```ts
function statusLabelB(status: OrderStatus): string {
  if (status === "pending") return "Awaiting payment";
  if (status === "paid")    return "Payment received";
  assertNever(status, "OrderStatus");
  // ↑ no `return` needed — the compiler knows control cannot reach past a
  //   `never`-returning call, so "not all code paths return a value" is satisfied.
}
```

⚠️ The `never`-returning-function trick has the **same annotation requirement** as `asserts`:

```ts
const assertNeverConst = (value: never): never => { throw new Error("unreachable"); };
function f(status: OrderStatus): string {
  switch (status) {
    case "pending": return "a";
    default: assertNeverConst(status);
    // ❌ A function returning 'never' cannot have a reachable end point / and
    //    control-flow effects require an explicit type annotation. ts(2775)
  }
}
```

There is also a genuine `asserts`-based variant, useful when you want exhaustiveness *without* being in return position:

```ts
// ── Version B: the true assertion form ──────────────────────────────────────
function assertUnreachable(value: never, context: string): asserts value is never {
  throw new Error(`Unhandled ${context}: ${String(value)}`);
}
```

Exhaustiveness over discriminated unions is where it earns its keep:

```ts
type WebhookEvent =
  | { readonly type: "payment.succeeded"; readonly chargeId: string; readonly amountCents: number }
  | { readonly type: "payment.failed";    readonly chargeId: string; readonly reason: string }
  | { readonly type: "refund.created";    readonly refundId: string; readonly amountCents: number }
  | { readonly type: "subscription.cancelled"; readonly subscriptionId: string };

async function handleWebhook(event: WebhookEvent): Promise<void> {
  switch (event.type) {
    case "payment.succeeded":
      await ledger.credit(event.chargeId, event.amountCents);
      return;
    case "payment.failed":
      await notifications.paymentFailed(event.chargeId, event.reason);
      return;
    case "refund.created":
      await ledger.debit(event.refundId, event.amountCents);
      return;
    case "subscription.cancelled":
      await subscriptions.deactivate(event.subscriptionId);
      return;
    default:
      // Compile-time: a new event type breaks the build.
      // Runtime: an unexpected payload from the provider throws loudly instead
      // of silently doing nothing — which is exactly what you want on a webhook.
      assertNever(event, "WebhookEvent");
  }
}
```

The runtime half matters more than people expect. Webhook providers add event types without asking. Without `assertNever`, the `default` branch falls through and your handler returns `200 OK` for an event it did not process — the worst possible outcome, because the provider never retries.

### Concept 8 — Node's `assert` module and how it is typed

Node ships `assert`, and since TypeScript 3.7 `@types/node` declares it with `asserts` clauses. It works out of the box:

```ts
import assert from "node:assert";
// or: import { strict as assert } from "node:assert";

declare const authToken: string | undefined;

assert(authToken !== undefined, "authToken must be present");
authToken.slice(7);      // ✅ string — narrowed by node:assert
```

The relevant declarations in `@types/node` look essentially like this:

```ts
declare function assert(value: unknown, message?: string | Error): asserts value;

declare namespace assert {
  function ok(value: unknown, message?: string | Error): asserts value;
  function strictEqual<T>(actual: unknown, expected: T, message?: string | Error): asserts actual is T;
  function notStrictEqual(actual: unknown, expected: unknown, message?: string | Error): void;
  function fail(message?: string | Error): never;
}
```

Note the mix: `assert` and `assert.ok` use the bare `asserts value` form; `strictEqual` uses `asserts actual is T`; `fail` returns `never`.

```ts
import assert from "node:assert";

declare const status: unknown;

assert.strictEqual(status, "shipped");
status;   // ✅ "shipped" — asserts actual is T, with T inferred from the literal

declare const order: Order | null;
assert.ok(order, "order must exist");
order.id; // ✅ Order
```

Three real caveats before you reach for it in production code:

**1. `assert` can be disabled.** Node has no `NODE_ENV`-based stripping for `node:assert` itself, but bundlers and some runtimes do transform it, and `--no-force-async-hooks-checks`-style flags plus custom builds have historically messed with it. More importantly, many teams treat `assert` as a *development* tool. If it can be stripped, your narrowing becomes a lie in production. Prefer your own `invariant` for anything on a request path.

**2. The thrown error is an `AssertionError`, not your domain error.** It maps to a 500, not a 400, unless you translate it:

```ts
// ❌ A malformed request body becomes an unhandled AssertionError → 500:
assert(typeof requestBody === "object", "body must be an object");

// ✅ Your own assertion throws the error your error middleware understands:
function assertRequestBody(value: unknown): asserts value is Record<string, unknown> {
  if (typeof value !== "object" || value === null || Array.isArray(value)) {
    throw new ApiError(400, "BAD_REQUEST", "Body must be a JSON object");
  }
}
```

**3. `import assert from "assert"` vs `import { strict as assert }`.** The default export uses legacy loose comparison in `deepEqual`/`equal`; the `strict` variant uses `===` semantics everywhere. Always use `node:assert/strict` (or `import { strict as assert }`) — but the `asserts` typing is identical either way.

```ts
import assert from "node:assert/strict";   // ✅ strict semantics, same narrowing
```

For most backends the right answer is: use `node:assert` in **tests**, and a hand-rolled `invariant` (which throws an error your logging and HTTP layers understand) in **production code**.

### Concept 9 — Assertion functions and schema libraries

Hand-written assertion bodies drift from their interfaces. The interface says `couponCode: string | null`; six months later someone adds `giftCardCode` to the interface and forgets the validator. The compiler cannot catch this — the `asserts` clause is unchecked.

Schema libraries fix this by making the type a *derivation* of the runtime check:

```ts
import { z } from "zod";

// ONE source of truth. The runtime validator and the static type cannot diverge.
const createOrderSchema = z.object({
  userId:       z.number().int().positive(),
  paymentToken: z.string().min(1),
  couponCode:   z.string().nullable(),
  items: z
    .array(
      z.object({
        sku:            z.string().min(1),
        quantity:       z.number().int().min(1),
        unitPriceCents: z.number().int().min(0),
      }),
    )
    .min(1),
});

type CreateOrderRequest = z.infer<typeof createOrderSchema>;

// Wrap it in an assertion so call sites stay one line and errors stay yours:
function assertCreateOrderRequest(value: unknown): asserts value is CreateOrderRequest {
  const result = createOrderSchema.safeParse(value);
  if (!result.success) {
    const first = result.error.issues[0];
    throw new ApiError(
      400,
      "VALIDATION_FAILED",
      first === undefined
        ? "Invalid request body"
        : `${first.path.join(".")}: ${first.message}`,
    );
  }
}
```

A generic factory makes this reusable across every endpoint — note the explicit return-type annotation on the produced function, which is required for the caller to get narrowing:

```ts
import type { ZodType } from "zod";

// The FACTORY returns a value; assign it to an explicitly annotated const:
function makeAssertion<T>(
  schema: ZodType<T>,
  label: string,
): (value: unknown) => asserts value is T {
  return (value): asserts value is T => {
    const result = schema.safeParse(value);
    if (!result.success) {
      throw new ApiError(400, "VALIDATION_FAILED", `${label}: ${result.error.message}`);
    }
  };
}

// ✅ explicit annotation on the const — otherwise ts(2775) at the call site:
const assertCreateOrder: (value: unknown) => asserts value is CreateOrderRequest =
  makeAssertion(createOrderSchema, "createOrder");

declare const requestBody: unknown;
assertCreateOrder(requestBody);
requestBody.items;   // ✅ narrowed
```

⚠️ One catch with the factory: an arrow function whose declared return type is an `asserts` clause must have that clause written explicitly on the arrow too (`(value): asserts value is T => {...}`), because a bare `=> {}` infers `void` and `void` is not assignable to an `asserts` signature in a return position.

Trade-off summary:

| | Hand-written assertion | Schema-derived assertion |
|---|---|---|
| Type/runtime drift possible | **yes** | no |
| Dependency weight | zero | 8–60 kB |
| Error messages | exactly what you write | library format, needs mapping |
| Perf on hot paths | fastest | ~2–10× slower per parse |
| Best for | small internal shapes, invariants | public API request bodies |

---

## Example 1 — basic

```ts
// A small assertions module of the kind every Node service ends up writing,
// plus the call sites that show what each one buys you.

// ── The helpers — ALL function declarations, so callers get narrowing ────────

/** Throws unless `condition` is truthy. Narrows the asserted expression. */
export function invariant(condition: unknown, message: string): asserts condition {
  if (!condition) {
    throw new Error(`Invariant violation: ${message}`);
  }
}

/** Throws unless `value` is neither null nor undefined. Returns nothing; narrows. */
export function assertDefined<T>(
  value: T | null | undefined,
  label: string,
): asserts value is T {
  if (value === null || value === undefined) {
    throw new Error(`Expected ${label} to be defined, got ${String(value)}`);
  }
}

/** Throws unless `value` is a string. */
export function assertString(value: unknown, label: string): asserts value is string {
  if (typeof value !== "string") {
    throw new TypeError(`Expected ${label} to be a string, got ${typeof value}`);
  }
}

/** Throws unless `value` is a finite number (rejects NaN and Infinity). */
export function assertFiniteNumber(
  value: unknown,
  label: string,
): asserts value is number {
  if (typeof value !== "number" || !Number.isFinite(value)) {
    throw new TypeError(`Expected ${label} to be a finite number, got ${String(value)}`);
  }
}

/** Throws unless `value` is a plain object (not null, not an array). */
export function assertPlainObject(
  value: unknown,
  label: string,
): asserts value is Record<string, unknown> {
  if (typeof value !== "object" || value === null || Array.isArray(value)) {
    throw new TypeError(`Expected ${label} to be an object`);
  }
}

/** Exhaustiveness guard — makes an unhandled union member a compile error. */
export function assertNever(value: never, context: string): never {
  throw new Error(`Unhandled ${context}: ${JSON.stringify(value)}`);
}

// ── Using them ──────────────────────────────────────────────────────────────

interface User {
  readonly userId:      number;
  readonly email:       string;
  readonly displayName: string | null;
}

declare const usersById: Map<number, User>;

// 1. assertDefined replaces `!` with a real check and a real error message:
function getUserEmail(userId: number): string {
  const user = usersById.get(userId);
  //    ^? User | undefined
  assertDefined(user, `user ${userId}`);
  return user.email;
  //     ^? User — narrowed, with a runtime guarantee
}

// 2. invariant narrows env vars at module scope, no `if` block needed:
const rawPort: string | undefined = process.env.PORT;
invariant(rawPort !== undefined, "PORT environment variable is required");
const port: number = Number.parseInt(rawPort, 10);
//                                   ^? string
invariant(Number.isInteger(port) && port > 0, `PORT must be a positive integer, got ${rawPort}`);

// 3. assertString/assertFiniteNumber peel apart an unknown payload safely:
function parseHeartbeat(payload: unknown): { serviceName: string; uptimeSeconds: number } {
  assertPlainObject(payload, "heartbeat payload");
  // payload: Record<string, unknown>

  const serviceName = payload.serviceName;
  //    ^? unknown
  assertString(serviceName, "heartbeat.serviceName");
  // serviceName: string

  const uptimeSeconds = payload.uptimeSeconds;
  assertFiniteNumber(uptimeSeconds, "heartbeat.uptimeSeconds");
  // uptimeSeconds: number

  return { serviceName, uptimeSeconds };
}

// 4. assertNever turns a new enum member into a build failure:
type NotificationChannel = "email" | "sms" | "push";

function channelCostCents(channel: NotificationChannel): number {
  switch (channel) {
    case "email": return 0;
    case "sms":   return 4;
    case "push":  return 0;
    default:      return assertNever(channel, "NotificationChannel");
  }
}

// 5. Narrowing persists — no nesting, no re-checking:
function buildGreeting(userId: number): string {
  const user = usersById.get(userId);
  assertDefined(user, `user ${userId}`);

  const name = user.displayName ?? user.email;   // ✅ user is User
  invariant(name.length > 0, `user ${userId} has an empty name`);

  return `Hello, ${name}!`;                      // ✅ still User, still narrowed
}

// 6. ⚠️ The inferred-const trap, shown once so you recognise the error:
const assertPositive = (value: number): asserts value is number => {
  if (value <= 0) throw new RangeError("must be positive");
};
// assertPositive(port);
// ❌ Assertions require every name in the call target to be declared with an
//    explicit type annotation. ts(2775)

// ✅ The fix — annotate the const with a function type:
const assertPositiveOk: (value: number) => asserts value is number = (value) => {
  if (value <= 0) throw new RangeError("must be positive");
};
assertPositiveOk(port);   // ✅ compiles
```

---

## Example 2 — real world backend use case

```ts
// An Express-style order service where assertion functions sit on every trust
// boundary: request bodies, auth context, domain state transitions, DB rows,
// and the exhaustive webhook handler.

import type { Request, Response, NextFunction, RequestHandler } from "express";

// ═══════════════════════════════════════════════════════════════════════════
// 1. Error types — assertions throw THESE, so the error middleware maps them
//    to correct HTTP status codes instead of a blanket 500.
// ═══════════════════════════════════════════════════════════════════════════

class ApiError extends Error {
  constructor(
    readonly statusCode: number,
    readonly code: string,
    message: string,
    readonly details?: Readonly<Record<string, unknown>>,
  ) {
    super(message);
    this.name = "ApiError";
  }

  static badRequest(message: string, details?: Record<string, unknown>): ApiError {
    return new ApiError(400, "BAD_REQUEST", message, details);
  }
  static unauthorized(message = "Authentication required"): ApiError {
    return new ApiError(401, "UNAUTHORIZED", message);
  }
  static forbidden(message = "Insufficient permissions"): ApiError {
    return new ApiError(403, "FORBIDDEN", message);
  }
  static notFound(what: string): ApiError {
    return new ApiError(404, "NOT_FOUND", `${what} not found`);
  }
  static conflict(message: string): ApiError {
    return new ApiError(409, "CONFLICT", message);
  }
}

/**
 * A programmer-error invariant. Distinct from ApiError on purpose: these are
 * bugs, never user input, and they should page someone.
 */
class InvariantError extends Error {
  constructor(message: string) {
    super(`Invariant violation: ${message}`);
    this.name = "InvariantError";
  }
}

function invariant(condition: unknown, message: string): asserts condition {
  if (!condition) throw new InvariantError(message);
}

function assertDefined<T>(value: T | null | undefined, label: string): asserts value is T {
  if (value === null || value === undefined) {
    throw new InvariantError(`${label} is ${String(value)}`);
  }
}

function assertNever(value: never, context: string): never {
  throw new InvariantError(`Unhandled ${context}: ${JSON.stringify(value)}`);
}

// ═══════════════════════════════════════════════════════════════════════════
// 2. Domain types
// ═══════════════════════════════════════════════════════════════════════════

type OrderStatus = "draft" | "pending_payment" | "paid" | "shipped" | "cancelled" | "refunded";

interface OrderItem {
  readonly sku:            string;
  readonly quantity:       number;
  readonly unitPriceCents: number;
}

interface Order {
  readonly orderId:    number;
  readonly userId:     number;
  readonly status:     OrderStatus;
  readonly items:      readonly OrderItem[];
  readonly totalCents: number;
  readonly shippedAt:  Date | null;
  readonly couponCode: string | null;
}

/** A branded sub-type: an Order proven to be in `paid` state. */
type PaidOrder = Order & { readonly status: "paid" };

/** A branded sub-type: an Order proven to be cancellable. */
type CancellableOrder = Order & { readonly status: "draft" | "pending_payment" | "paid" };

interface AuthContext {
  readonly userId: number;
  readonly scopes: readonly string[];
  readonly role:   "member" | "support" | "admin";
}

interface AuthenticatedRequest extends Request {
  /** Optional because it is absent BEFORE the auth middleware runs. */
  auth?: AuthContext;
}

/** The same request, proven to have passed authentication. */
interface VerifiedRequest extends AuthenticatedRequest {
  readonly auth: AuthContext;
}

// ═══════════════════════════════════════════════════════════════════════════
// 3. Request-body validation assertions
//    One function per endpoint. It throws ApiError (→ 400), and it narrows.
// ═══════════════════════════════════════════════════════════════════════════

interface CreateOrderBody {
  readonly items:      readonly OrderItem[];
  readonly couponCode: string | null;
}

function assertRecord(
  value: unknown,
  field: string,
): asserts value is Record<string, unknown> {
  if (typeof value !== "object" || value === null || Array.isArray(value)) {
    throw ApiError.badRequest(`${field} must be a JSON object`, { field });
  }
}

function assertPositiveInt(
  value: unknown,
  field: string,
): asserts value is number {
  if (typeof value !== "number" || !Number.isInteger(value) || value < 1) {
    throw ApiError.badRequest(`${field} must be a positive integer`, { field, received: value });
  }
}

function assertNonEmptyString(
  value: unknown,
  field: string,
): asserts value is string {
  if (typeof value !== "string" || value.trim().length === 0) {
    throw ApiError.badRequest(`${field} must be a non-empty string`, { field });
  }
}

function assertOrderItem(value: unknown, field: string): asserts value is OrderItem {
  assertRecord(value, field);
  assertNonEmptyString(value.sku, `${field}.sku`);
  assertPositiveInt(value.quantity, `${field}.quantity`);
  if (
    typeof value.unitPriceCents !== "number" ||
    !Number.isInteger(value.unitPriceCents) ||
    value.unitPriceCents < 0
  ) {
    throw ApiError.badRequest(`${field}.unitPriceCents must be a non-negative integer`);
  }
}

function assertCreateOrderBody(value: unknown): asserts value is CreateOrderBody {
  assertRecord(value, "body");

  const items = value.items;
  if (!Array.isArray(items)) {
    throw ApiError.badRequest("body.items must be an array", { field: "items" });
  }
  if (items.length === 0) {
    throw ApiError.badRequest("body.items must not be empty", { field: "items" });
  }
  if (items.length > 100) {
    throw ApiError.badRequest("body.items may contain at most 100 entries", { field: "items" });
  }
  for (const [index, item] of items.entries()) {
    assertOrderItem(item, `body.items[${index}]`);
  }

  const couponCode = value.couponCode;
  if (couponCode !== null && couponCode !== undefined && typeof couponCode !== "string") {
    throw ApiError.badRequest("body.couponCode must be a string or null", {
      field: "couponCode",
    });
  }
}

// ═══════════════════════════════════════════════════════════════════════════
// 4. Auth assertions — these narrow the REQUEST itself
// ═══════════════════════════════════════════════════════════════════════════

function assertAuthenticated(req: AuthenticatedRequest): asserts req is VerifiedRequest {
  if (req.auth === undefined) {
    // Reachable only if a route was wired without the auth middleware — a bug,
    // but we still return a correct 401 rather than crashing with a 500.
    throw ApiError.unauthorized("Authentication middleware did not run");
  }
}

function assertHasScope(auth: AuthContext, scope: string): asserts auth is AuthContext {
  if (!auth.scopes.includes(scope)) {
    throw ApiError.forbidden(`Missing required scope: ${scope}`);
  }
}

function assertIsAdmin(
  auth: AuthContext,
): asserts auth is AuthContext & { readonly role: "admin" } {
  if (auth.role !== "admin") {
    throw ApiError.forbidden("Admin role required");
  }
}

// ═══════════════════════════════════════════════════════════════════════════
// 5. Domain-state assertions — illegal transitions become impossible
// ═══════════════════════════════════════════════════════════════════════════

function assertOrderPaid(order: Order): asserts order is PaidOrder {
  if (order.status !== "paid") {
    throw ApiError.conflict(
      `Order ${order.orderId} must be paid to ship, but is '${order.status}'`,
    );
  }
}

function assertOrderCancellable(order: Order): asserts order is CancellableOrder {
  if (order.status === "shipped" || order.status === "cancelled" || order.status === "refunded") {
    throw ApiError.conflict(
      `Order ${order.orderId} cannot be cancelled from status '${order.status}'`,
    );
  }
}

function assertOwnedBy(order: Order, auth: AuthContext): asserts order is Order {
  if (order.userId !== auth.userId && auth.role === "member") {
    // 404 rather than 403 on purpose — do not leak the existence of the row.
    throw ApiError.notFound("Order");
  }
}

// ═══════════════════════════════════════════════════════════════════════════
// 6. Repositories
// ═══════════════════════════════════════════════════════════════════════════

interface DbClient {
  query<T>(sql: string, params: readonly unknown[]): Promise<readonly T[]>;
}

class OrderRepository {
  constructor(private readonly db: DbClient) {}

  async findById(orderId: number): Promise<Order | null> {
    const rows = await this.db.query<Order>(
      "SELECT * FROM orders WHERE order_id = $1",
      [orderId],
    );
    return rows[0] ?? null;
  }

  /**
   * The "must exist" variant. Note this returns Order rather than being an
   * assertion function — an assertion cannot narrow an awaited expression,
   * only an already-bound reference. See "Going deeper".
   */
  async getById(orderId: number): Promise<Order> {
    const order = await this.findById(orderId);
    if (order === null) throw ApiError.notFound(`Order ${orderId}`);
    return order;
  }

  async insert(
    userId: number,
    items: readonly OrderItem[],
    totalCents: number,
    couponCode: string | null,
  ): Promise<Order> {
    const rows = await this.db.query<Order>(
      `INSERT INTO orders (user_id, status, items, total_cents, coupon_code)
       VALUES ($1, 'pending_payment', $2, $3, $4) RETURNING *`,
      [userId, JSON.stringify(items), totalCents, couponCode],
    );
    const inserted = rows[0];
    // A RETURNING clause on a successful INSERT always produces exactly one row.
    // If it did not, the database is broken — that is an invariant, not a 4xx.
    assertDefined(inserted, "INSERT ... RETURNING produced no row");
    return inserted;
  }

  async updateStatus(orderId: number, status: OrderStatus): Promise<void> {
    await this.db.query("UPDATE orders SET status = $1 WHERE order_id = $2", [status, orderId]);
  }
}

// ═══════════════════════════════════════════════════════════════════════════
// 7. Service layer — assertions compose into a readable pipeline
// ═══════════════════════════════════════════════════════════════════════════

interface CarrierClient {
  book(orderId: number, items: readonly OrderItem[]): Promise<{ trackingId: string }>;
}
interface PaymentClient {
  refund(orderId: number, amountCents: number): Promise<void>;
}
interface CouponRepository {
  findByCode(code: string): Promise<{ code: string; percentOff: number } | null>;
}

class OrderService {
  constructor(
    private readonly orders:  OrderRepository,
    private readonly coupons: CouponRepository,
    private readonly carrier: CarrierClient,
    private readonly payments: PaymentClient,
  ) {}

  async create(auth: AuthContext, body: CreateOrderBody): Promise<Order> {
    const subtotalCents = body.items.reduce(
      (sum, item) => sum + item.unitPriceCents * item.quantity,
      0,
    );
    invariant(subtotalCents >= 0, "subtotal computed negative");

    let totalCents = subtotalCents;

    if (body.couponCode !== null) {
      const coupon = await this.coupons.findByCode(body.couponCode);
      if (coupon === null) {
        throw ApiError.badRequest(`Unknown coupon code: ${body.couponCode}`);
      }
      // Bounds we control, so an invariant (500) is correct — a coupon row with
      // percentOff = 300 is a data bug, not a user error:
      invariant(
        coupon.percentOff >= 0 && coupon.percentOff <= 100,
        `coupon ${coupon.code} has invalid percentOff ${coupon.percentOff}`,
      );
      totalCents = Math.round(subtotalCents * (1 - coupon.percentOff / 100));
    }

    return this.orders.insert(auth.userId, body.items, totalCents, body.couponCode);
  }

  async ship(auth: AuthContext, orderId: number): Promise<{ trackingId: string }> {
    assertHasScope(auth, "orders:write");

    const order = await this.orders.getById(orderId);
    assertOwnedBy(order, auth);
    assertOrderPaid(order);
    // order: PaidOrder — the compiler now knows the state machine is satisfied.

    const shipment = await this.carrier.book(order.orderId, order.items);
    await this.orders.updateStatus(order.orderId, "shipped");
    return shipment;
  }

  async cancel(auth: AuthContext, orderId: number): Promise<void> {
    assertHasScope(auth, "orders:write");

    const order = await this.orders.getById(orderId);
    assertOwnedBy(order, auth);
    assertOrderCancellable(order);
    // order: CancellableOrder — status is "draft" | "pending_payment" | "paid"

    // Because the type narrowed, this switch is exhaustive over THREE cases,
    // not six. Adding a new cancellable status breaks the build here:
    switch (order.status) {
      case "draft":
      case "pending_payment":
        await this.orders.updateStatus(order.orderId, "cancelled");
        return;
      case "paid":
        await this.payments.refund(order.orderId, order.totalCents);
        await this.orders.updateStatus(order.orderId, "refunded");
        return;
      default:
        assertNever(order.status, "CancellableOrder status");
    }
  }
}

// ═══════════════════════════════════════════════════════════════════════════
// 8. HTTP handlers — three lines of assertions, then pure domain code
// ═══════════════════════════════════════════════════════════════════════════

interface ApiResponse<T> {
  readonly data:      T;
  readonly requestId: string;
}

function assertRouteParam(
  params: Request["params"],
  name: string,
): asserts params is Record<string, string> {
  if (typeof params[name] !== "string") {
    throw ApiError.badRequest(`Missing route parameter :${name}`);
  }
}

function parseIntParam(raw: string, name: string): number {
  const parsed = Number.parseInt(raw, 10);
  if (!Number.isInteger(parsed) || parsed < 1) {
    throw ApiError.badRequest(`Route parameter :${name} must be a positive integer`);
  }
  return parsed;
}

function makeCreateOrderHandler(orderService: OrderService): RequestHandler {
  return async function (req: AuthenticatedRequest, res: Response, next: NextFunction) {
    try {
      assertAuthenticated(req);            // req: VerifiedRequest
      const requestBody: unknown = req.body;
      assertCreateOrderBody(requestBody);  // requestBody: CreateOrderBody

      const order = await orderService.create(req.auth, requestBody);
      //                                      ^^^^^^^^ AuthContext, not undefined

      const response: ApiResponse<Order> = {
        data:      order,
        requestId: typeof req.headers["x-request-id"] === "string"
          ? req.headers["x-request-id"]
          : "unknown",
      };
      res.status(201).json(response);
    } catch (err) {
      next(err);
    }
  } as RequestHandler;
}

function makeShipOrderHandler(orderService: OrderService): RequestHandler {
  return async function (req: AuthenticatedRequest, res: Response, next: NextFunction) {
    try {
      assertAuthenticated(req);
      assertRouteParam(req.params, "orderId");
      const orderId = parseIntParam(req.params.orderId, "orderId");

      const shipment = await orderService.ship(req.auth, orderId);
      const response: ApiResponse<{ trackingId: string }> = {
        data:      shipment,
        requestId: String(req.headers["x-request-id"] ?? "unknown"),
      };
      res.status(200).json(response);
    } catch (err) {
      next(err);
    }
  } as RequestHandler;
}

// ═══════════════════════════════════════════════════════════════════════════
// 9. Error middleware — this is what makes assertion-thrown errors correct
// ═══════════════════════════════════════════════════════════════════════════

function errorHandler(
  err: unknown,
  _req: Request,
  res: Response,
  _next: NextFunction,
): void {
  if (err instanceof ApiError) {
    // Expected: a client sent something wrong, or a state transition is illegal.
    res.status(err.statusCode).json({ error: err.message, code: err.code, details: err.details });
    return;
  }
  if (err instanceof InvariantError) {
    // Unexpected: OUR bug. Log loudly, return an opaque 500.
    console.error({ msg: "invariant violated", err });
    res.status(500).json({ error: "Internal server error", code: "INTERNAL" });
    return;
  }
  console.error({ msg: "unhandled error", err });
  res.status(500).json({ error: "Internal server error", code: "INTERNAL" });
}

// ═══════════════════════════════════════════════════════════════════════════
// 10. Webhooks — assertNever makes an unknown provider event fail loudly
// ═══════════════════════════════════════════════════════════════════════════

type PaymentWebhook =
  | { readonly type: "charge.succeeded"; readonly orderId: number; readonly amountCents: number }
  | { readonly type: "charge.failed";    readonly orderId: number; readonly reason: string }
  | { readonly type: "refund.settled";   readonly orderId: number; readonly amountCents: number };

function assertPaymentWebhook(value: unknown): asserts value is PaymentWebhook {
  assertRecord(value, "webhook");
  const type = value.type;
  if (type !== "charge.succeeded" && type !== "charge.failed" && type !== "refund.settled") {
    throw ApiError.badRequest(`Unsupported webhook type: ${String(type)}`);
  }
  assertPositiveInt(value.orderId, "webhook.orderId");
  if (type === "charge.failed") {
    assertNonEmptyString(value.reason, "webhook.reason");
  } else {
    assertPositiveInt(value.amountCents, "webhook.amountCents");
  }
}

async function handlePaymentWebhook(rawBody: unknown, orders: OrderRepository): Promise<void> {
  assertPaymentWebhook(rawBody);
  // rawBody: PaymentWebhook

  switch (rawBody.type) {
    case "charge.succeeded":
      await orders.updateStatus(rawBody.orderId, "paid");
      return;
    case "charge.failed":
      console.warn({ msg: "charge failed", orderId: rawBody.orderId, reason: rawBody.reason });
      await orders.updateStatus(rawBody.orderId, "cancelled");
      return;
    case "refund.settled":
      await orders.updateStatus(rawBody.orderId, "refunded");
      return;
    default:
      // Provider adds an event type → compile error here, not a silent 200 OK.
      assertNever(rawBody, "PaymentWebhook");
  }
}
```

---

## Going deeper

### An assertion cannot narrow an expression — only a bound reference

This is the most common structural surprise:

```ts
declare function assertDefined<T>(value: T | null | undefined, label: string): asserts value is T;
declare function findOrder(orderId: number): Promise<Order | null>;

// ❌ Narrowing has nowhere to land — there is no reference to narrow:
assertDefined(await findOrder(orderId), "order");
// The call compiles, but you have no variable holding the narrowed value.

// ❌ Same problem with a property path through a function call:
assertDefined(getSession().user, "user");
// getSession() may return a different object next call; TS narrows nothing useful.

// ✅ Bind first, assert second:
const order = await findOrder(orderId);
assertDefined(order, `order ${orderId}`);
order.items;   // ✅ Order
```

This is why repository "get or throw" helpers are usually written to **return** the narrowed value rather than assert it:

```ts
// The assertion form forces two statements at every call site:
const order = await this.findById(orderId);
assertDefined(order, "order");

// The returning form is one statement — prefer it for async lookups:
async function getById(orderId: number): Promise<Order> {
  const order = await this.findById(orderId);
  if (order === null) throw ApiError.notFound(`Order ${orderId}`);
  return order;
}
```

Rule of thumb: **assert when you already have a reference; return when you are producing one.**

### Assertions do not survive `await` on a mutable property — and TS does not warn

Assertion narrowing has exactly the same unsoundness as `if`-based narrowing (see `62 — Strict null checks`):

```ts
interface Session { user: User | null }
declare const session: Session;
declare function refreshSession(s: Session): Promise<void>;   // may set s.user = null

invariant(session.user !== null, "session must have a user");
await refreshSession(session);
session.user.email;      // ✅ compiles. 💥 possibly at runtime.
//      ^? User          ← TS does NOT re-check after the await
```

The fix is the same too — copy into a `const` before asserting:

```ts
const { user } = session;
invariant(user !== null, "session must have a user");
await refreshSession(session);
user.email;              // ✅ genuinely safe — `user` is a frozen local
```

### Assertion narrowing is lost inside closures over `let`

```ts
let currentOrder: Order | null = await orderRepo.findById(orderId);
assertDefined(currentOrder, "order");
currentOrder.orderId;             // ✅ Order

setTimeout(() => {
  currentOrder.orderId;           // ❌ 'currentOrder' is possibly 'null'
}, 1_000);
```

The compiler cannot know when the callback runs, and `currentOrder` is a `let`. Use `const`:

```ts
const currentOrder = await orderRepo.findById(orderId);
assertDefined(currentOrder, "order");
setTimeout(() => {
  currentOrder.orderId;           // ✅ narrowing survives into the closure
}, 1_000);
```

### `asserts this is T` — assertion methods

The asserted "parameter" may be `this`, which lets a class narrow *itself*:

```ts
class DbConnection {
  private socket: NodeJS.Socket | null = null;

  async open(): Promise<void> {
    this.socket = await createSocket();
  }

  assertOpen(): asserts this is DbConnection & { socket: NodeJS.Socket } {
    if (this.socket === null) {
      throw new InvariantError("DbConnection used before open()");
    }
  }

  send(payload: Buffer): void {
    this.assertOpen();
    this.socket.write(payload);   // ✅ socket: NodeJS.Socket
  }
}

declare function createSocket(): Promise<NodeJS.Socket>;
```

⚠️ Two constraints:

- `asserts this is T` requires the method to be called on a reference whose type is known — `this.assertOpen()` is fine, `const f = conn.assertOpen; f()` is not.
- The narrowed `this` type is discarded when any method that could mutate `socket` is called, in the same way as ordinary narrowing... except it is not. TypeScript will happily keep believing it. The unsoundness above applies here too.

### An assertion function's return type must be `void`-compatible

You cannot return a value *and* assert. This is a hard rule:

```ts
// ❌ Not expressible — you must pick one:
// function assertAndReturn(value: unknown): asserts value is string, string {}

// ✅ Either assert (returns void)...
function assertString(value: unknown): asserts value is string {
  if (typeof value !== "string") throw new TypeError("expected string");
}

// ✅ ...or parse (returns the value):
function parseString(value: unknown, label: string): string {
  if (typeof value !== "string") throw new TypeError(`${label} must be a string`);
  return value;
}

// The parse form composes better in expressions:
const config = {
  host: parseString(process.env.DB_HOST, "DB_HOST"),
  user: parseString(process.env.DB_USER, "DB_USER"),
};

// The assert form composes better when you already have a variable:
const rawHost: unknown = process.env.DB_HOST;
assertString(rawHost);
```

Mature codebases carry both flavours. Rough guidance: **`assertX` for existing references and invariants; `parseX` for building new values.**

### `asserts value is never` and the unreachability effect

`asserts value is never` has a special consequence: after the call, the compiler believes the value is `never`, which usually means the code after it is unreachable.

```ts
function assertUnreachable(value: never): asserts value is never {
  throw new InvariantError(`Unreachable value: ${String(value)}`);
}

function label(status: OrderStatus): string {
  if (status === "draft")           return "Draft";
  if (status === "pending_payment") return "Awaiting payment";
  if (status === "paid")            return "Paid";
  if (status === "shipped")         return "Shipped";
  if (status === "cancelled")       return "Cancelled";
  if (status === "refunded")        return "Refunded";
  assertUnreachable(status);
  // ❌ Function lacks ending return statement and return type does not include 'undefined'.
  //    `asserts value is never` does NOT tell the compiler control stops here!
}
```

Compare the `never`-**returning** version, which does:

```ts
function assertNever(value: never): never {
  throw new InvariantError(`Unreachable value: ${String(value)}`);
}

function label2(status: OrderStatus): string {
  if (status === "draft") return "Draft";
  // ... all cases ...
  assertNever(status);   // ✅ control provably stops — no ending return required
}
```

This is the reason `assertNever` is conventionally written with `: never` and not `: asserts value is never`. **`: never` affects reachability; `asserts` only affects narrowing.**

### Assertion functions and `strictNullChecks` are coupled

`asserts value is T` where `T` excludes `null | undefined` only *means* anything under `strictNullChecks`. With the flag off, `null` and `undefined` are members of every type, so the narrowing is a no-op:

```ts
// strictNullChecks: false
declare const order: Order | null;
assertDefined(order, "order");
order.items;    // "narrowed" — but `order` could equally have been null all along
```

Every assertion helper in this document assumes `"strict": true`. Without it, assertion functions are decoration.

### Assertion narrowing beats `!` even where both compile

Consider the classic `Map.has` / `Map.get` gap:

```ts
const ordersById = new Map<number, Order>();

// The `!` version — compiles, no check, silently wrong if the map mutated between:
if (ordersById.has(orderId)) {
  const order = ordersById.get(orderId)!;
  order.items;
}

// The assertion version — one extra call, a real check, a real error:
const order = ordersById.get(orderId);
assertDefined(order, `order ${orderId} in cache`);
order.items;
```

The emitted JavaScript for the first is `ordersById.get(orderId)`. For the second it is a `get`, a comparison, and a possible `throw`. That is the entire runtime cost of the difference, and it converts a `TypeError: Cannot read properties of undefined (reading 'items')` twelve frames later into `InvariantError: order 4821 in cache is undefined` at the exact site.

### Cost, inlining, and hot paths

Assertion functions are ordinary function calls. They emit fully:

```ts
assertDefined(order, "order");
```

```js
// emitted output — nothing is erased:
assertDefined(order, "order");
```

For 99% of backend code this is free — V8 inlines small monomorphic throw-guards trivially. Two places where it is not free:

```ts
// ❌ Per-element assertion in a hot loop over 500k rows:
for (const row of rows) {
  assertUserRow(row);            // full shape validation, 500k times
  totals.set(row.userId, row.balanceCents);
}

// ✅ Assert the shape ONCE at the boundary, then trust the type:
assertUserRowArray(rows);        // one pass, or validate at ingestion
for (const row of rows) {
  totals.set(row.userId, row.balanceCents);
}
```

Also note that a template-literal message argument is evaluated **before** the call, even when the assertion passes:

```ts
// ❌ builds the string on every call, even the 99.99% that succeed:
invariant(order !== null, `order ${orderId} not found for user ${auth.userId} at ${Date.now()}`);

// ✅ lazy message — only built on failure:
function invariantLazy(condition: unknown, message: () => string): asserts condition {
  if (!condition) throw new InvariantError(message());
}
invariantLazy(order !== null, () => `order ${orderId} not found for user ${auth.userId}`);
```

Only bother with the lazy form on genuinely hot paths; the eager form reads better everywhere else.

### Assertion functions cannot be used as callbacks

Because narrowing requires a *call expression* the compiler can see, an assertion function passed as a value loses its assertion power entirely:

```ts
const bodies: unknown[] = [/* ... */];

// ❌ Does not narrow — `forEach` sees a `(value: unknown) => void`:
bodies.forEach(assertCreateOrderBody);
// bodies is still unknown[]

// ❌ Assertion functions cannot filter — they return void, not boolean:
// bodies.filter(assertCreateOrderBody);   // filters nothing meaningfully

// ✅ Use a type predicate for collection operations:
function isCreateOrderBody(value: unknown): value is CreateOrderBody {
  try {
    assertCreateOrderBody(value);
    return true;
  } catch {
    return false;
  }
}
const valid = bodies.filter(isCreateOrderBody);
//    ^? CreateOrderBody[]

// ✅ Or assert the whole array shape in one function:
function assertCreateOrderBodies(value: unknown): asserts value is CreateOrderBody[] {
  if (!Array.isArray(value)) throw ApiError.badRequest("expected an array");
  for (const [i, item] of value.entries()) {
    assertRecord(item, `body[${i}]`);
    assertCreateOrderBody(item);
  }
}
```

(The `try/catch`-wrapping predicate is convenient but throws away the error message and is slow on the reject path. For anything hot, write the predicate as the primitive and derive the assertion from it — Concept 1.)

### `asserts` in `.d.ts` files and declaration emit

An `asserts` clause is part of the public signature and is preserved in `.d.ts` output:

```ts
// assertions.d.ts — generated by tsc
export declare function invariant(condition: unknown, message: string): asserts condition;
export declare function assertDefined<T>(value: T | null | undefined, label: string): asserts value is T;
```

Consumers of your package get narrowing for free — provided their reference to the function is explicitly typed, which an `import` of a `function` declaration always is.

⚠️ One trap: if an assertion function is exported from a module that is **re-exported through a barrel with `export *`**, narrowing still works. But if the barrel re-assigns it into an object literal without a type annotation, narrowing breaks for every consumer:

```ts
// ❌ barrel.ts — silently destroys narrowing downstream:
import { invariant, assertDefined } from "./assertions";
export const guards = { invariant, assertDefined };   // inferred object type

// ✅ barrel.ts — keep the declarations exported directly:
export { invariant, assertDefined } from "./assertions";
```

---

## Common mistakes

### Mistake 1 — Writing an assertion function that does not actually check

```ts
// ❌ Wrong — the classic "I'll add the validation later" that never happens.
//    This is a cast in disguise, and it looks safer than `as`, which is worse.
function assertCreateOrderBody(value: unknown): asserts value is CreateOrderBody {
  // TODO: validate
}

const requestBody: unknown = JSON.parse(rawBody);
assertCreateOrderBody(requestBody);
const total = requestBody.items.reduce((s, i) => s + i.unitPriceCents, 0);
// 💥 TypeError: Cannot read properties of undefined (reading 'reduce')
// ...and the stack trace points at your pricing code, not at the parser.

// ✅ Right — every field the interface promises is checked, and the failure is
//    an error your HTTP layer understands.
function assertCreateOrderBody(value: unknown): asserts value is CreateOrderBody {
  if (typeof value !== "object" || value === null || Array.isArray(value)) {
    throw ApiError.badRequest("body must be a JSON object");
  }
  const body = value as Record<string, unknown>;

  if (!Array.isArray(body.items) || body.items.length === 0) {
    throw ApiError.badRequest("body.items must be a non-empty array");
  }
  for (const [index, raw] of body.items.entries()) {
    if (typeof raw !== "object" || raw === null) {
      throw ApiError.badRequest(`body.items[${index}] must be an object`);
    }
    const item = raw as Record<string, unknown>;
    if (typeof item.sku !== "string" || item.sku.length === 0) {
      throw ApiError.badRequest(`body.items[${index}].sku must be a non-empty string`);
    }
    if (typeof item.quantity !== "number" || !Number.isInteger(item.quantity) || item.quantity < 1) {
      throw ApiError.badRequest(`body.items[${index}].quantity must be a positive integer`);
    }
    if (typeof item.unitPriceCents !== "number" || item.unitPriceCents < 0) {
      throw ApiError.badRequest(`body.items[${index}].unitPriceCents must be >= 0`);
    }
  }

  if (body.couponCode !== null && body.couponCode !== undefined && typeof body.couponCode !== "string") {
    throw ApiError.badRequest("body.couponCode must be a string or null");
  }
}
```

The discipline: **an assertion function is the one place in your codebase where a missing runtime check is invisible.** Review them like security code, and unit-test the reject cases.

### Mistake 2 — Returning instead of throwing

```ts
// ❌ Wrong — returns on the failure path. The compiler still narrows!
function assertAuthToken(value: unknown): asserts value is string {
  if (typeof value !== "string") {
    console.warn("authToken is not a string");
    return;                        // 💀 the caller now believes it IS a string
  }
}

declare const authToken: unknown;
assertAuthToken(authToken);
authToken.slice(7);                // ✅ compiles. 💥 when authToken is undefined.

// ❌ Also wrong — returning a boolean. `asserts` ignores the return VALUE
//    entirely; only throwing vs returning matters.
function assertAuthToken2(value: unknown): asserts value is string {
  return typeof value === "string";   // ❌ Type 'boolean' is not assignable to 'void'
}

// ✅ Right — the only non-throwing exit is the success path:
function assertAuthToken3(value: unknown): asserts value is string {
  if (typeof value !== "string" || value.length === 0) {
    throw ApiError.unauthorized("Missing or malformed bearer token");
  }
}
```

Every code path through an assertion function must either **throw** or represent a genuinely valid value. If you find yourself wanting to log-and-continue, you wanted a type predicate.

### Mistake 3 — Using an inferred arrow const and fighting ts(2775)

```ts
// ❌ Wrong — this is the #1 confusion with assertion functions:
export const assertDefined = <T>(value: T | null | undefined, label: string): asserts value is T => {
  if (value === null || value === undefined) throw new Error(`${label} is missing`);
};

const order = await orderRepo.findById(orderId);
assertDefined(order, "order");
// ❌ Assertions require every name in the call target to be declared with an
//    explicit type annotation. ts(2775)

// ❌ Also wrong — "fixing" it by giving up and casting:
const safeOrder = order as Order;

// ✅ Right (best) — use a function declaration:
export function assertDefined<T>(value: T | null | undefined, label: string): asserts value is T {
  if (value === null || value === undefined) throw new Error(`${label} is missing`);
}

// ✅ Right (when a const is unavoidable) — annotate the const explicitly.
//    Note that a GENERIC assertion cannot be written as an annotated const in a
//    fully generic way; you must monomorphise it or use a declaration.
const assertOrderDefined: (value: Order | null | undefined, label: string) => asserts value is Order =
  (value, label) => {
    if (value === null || value === undefined) throw new Error(`${label} is missing`);
  };
assertOrderDefined(order, "order");   // ✅ narrows
```

The practical rule: **assertion helpers live in a module as `function` declarations.** Every workaround is more code than just writing `function`.

### Mistake 4 — Expecting `asserts condition` to narrow a discriminated union

```ts
type AppUser = AdminUser | RegularUser;
declare const currentUser: AppUser;

// ❌ Wrong — the bare form narrows the CONDITION, not the object:
invariant(currentUser.role === "admin", "must be admin");
// currentUser.scopes;
// ❌ Property 'scopes' does not exist on type 'AppUser'.

// ❌ Also wrong — a cast to "fix" it, throwing away the check you just did:
const admin = currentUser as AdminUser;

// ✅ Right — use the `is` form, which narrows the object itself:
function assertIsAdmin(user: AppUser): asserts user is AdminUser {
  if (user.role !== "admin") throw ApiError.forbidden("Admin role required");
}
assertIsAdmin(currentUser);
currentUser.scopes;   // ✅ readonly string[]

// ✅ Also right — plain control flow, when it is a one-off:
if (currentUser.role !== "admin") throw ApiError.forbidden("Admin role required");
currentUser.scopes;   // ✅ discriminant narrowing works normally in an `if`
```

The bare `asserts condition` form is for `!== null`, `!== undefined`, `typeof`, `Array.isArray`, and `instanceof`. For union members, use `asserts x is T`.

### Mistake 5 — Throwing an error type the HTTP layer maps to 500

```ts
// ❌ Wrong — a malformed client request produces a 500 and pages the on-call.
//    The client sees "Internal server error" and cannot fix their integration.
function assertCreateOrderBody(value: unknown): asserts value is CreateOrderBody {
  if (typeof value !== "object" || value === null) {
    throw new Error("body must be an object");     // → generic Error → 500
  }
}

// ❌ Also wrong — Node's assert throws AssertionError, also a 500:
import assert from "node:assert";
function assertCreateOrderBody2(value: unknown): asserts value is CreateOrderBody {
  assert(typeof value === "object" && value !== null, "body must be an object");
}

// ✅ Right — user-input assertions throw a 4xx-mapped error; INTERNAL invariants
//    throw a distinct type that maps to 500 and gets alerted on:
function assertCreateOrderBody3(value: unknown): asserts value is CreateOrderBody {
  if (typeof value !== "object" || value === null || Array.isArray(value)) {
    throw ApiError.badRequest("body must be a JSON object");     // → 400
  }
  // ...
}

function invariant(condition: unknown, message: string): asserts condition {
  if (!condition) throw new InvariantError(message);              // → 500 + alert
}
```

The distinction matters operationally: **`ApiError` means "the client did something wrong"; `InvariantError` means "we did something wrong."** If both throw the same class, your 500 rate becomes meaningless as a health signal.

### Mistake 6 — Asserting a value that outlives the assertion

```ts
interface RequestContext { user: User | null; requestId: string }
declare const ctx: RequestContext;
declare function reloadUser(c: RequestContext): Promise<void>;   // may null out c.user

// ❌ Wrong — the narrowing survives the await in the type system, not in reality:
invariant(ctx.user !== null, "user required");
await reloadUser(ctx);
await auditLog.write(ctx.user.userId, "order.created");
//                       ^^^^ TS says User. 💥 if reloadUser cleared it.

// ❌ Also wrong — re-asserting after every call is noise and still misses cases:
invariant(ctx.user !== null, "user required");
await reloadUser(ctx);
invariant(ctx.user !== null, "user required");     // works, but unsustainable
await someOtherThing(ctx);
invariant(ctx.user !== null, "user required");     // ...and again

// ✅ Right — snapshot into a const, then assert. The const cannot be mutated
//    by anyone, so the narrowing is genuinely valid for its whole lifetime:
const { user } = ctx;
invariant(user !== null, "user required");
await reloadUser(ctx);
await auditLog.write(user.userId, "order.created");   // ✅ safe for real
```

### Mistake 7 — Using `assertNever` without a runtime throw in production

```ts
type OrderStatus = "draft" | "paid" | "shipped";

// ❌ Wrong — "compile-time only" exhaustiveness. When the runtime value is
//    outside the union (a new DB enum value, a stale cached row, a webhook),
//    this silently returns undefined and callers get garbage:
function statusLabel(status: OrderStatus): string {
  switch (status) {
    case "draft":   return "Draft";
    case "paid":    return "Paid";
    case "shipped": return "Shipped";
  }
  // no default — TS is satisfied, runtime is not
}

// ❌ Also wrong — a default that swallows the unknown case:
function statusLabel2(status: OrderStatus): string {
  switch (status) {
    case "draft":   return "Draft";
    case "paid":    return "Paid";
    case "shipped": return "Shipped";
    default:        return "Unknown";   // hides a data bug forever
  }
}

// ✅ Right — compile-time exhaustiveness AND a loud runtime failure:
function statusLabel3(status: OrderStatus): string {
  switch (status) {
    case "draft":   return "Draft";
    case "paid":    return "Paid";
    case "shipped": return "Shipped";
    default:        return assertNever(status, "OrderStatus");
  }
}
```

Remember that `OrderStatus` is a *compile-time* claim about a value that came from a database or an HTTP body. The runtime throw is what catches the day the claim stops being true.

---

## Practice exercises

### Exercise 1 — easy

Build a small `assertions.ts` module for a Node service and use it.

Define, as **function declarations** (so call sites narrow):

1. `invariant(condition: unknown, message: string): asserts condition`
2. `assertDefined<T>(value: T | null | undefined, label: string): asserts value is T`
3. `assertNonEmptyString(value: unknown, label: string): asserts value is string` — rejects non-strings and strings that are empty after trimming
4. `assertPositiveInt(value: unknown, label: string): asserts value is number` — rejects non-numbers, non-integers, `NaN`, and values `< 1`
5. `assertNever(value: never, context: string): never`

Then write these consumers, each of which must compile with `"strict": true` and contain **zero** `!` and **zero** `as`:

- `loadDatabaseUrl(env: NodeJS.ProcessEnv): string` — reads `DATABASE_URL`, uses `assertNonEmptyString`, returns it
- `getUserDisplayName(usersById: Map<number, { userId: number; email: string; displayName: string | null }>, userId: number): string` — uses `assertDefined`, falls back from `displayName` to `email`
- `parsePageSize(raw: unknown): number` — uses `assertPositiveInt` and clamps the result to `[1, 100]`
- `channelLabel(channel: "email" | "sms" | "push" | "webhook"): string` — a `switch` ending in `assertNever`

Finally, write the deliberately-broken version of each helper as a commented-out block and note in a comment what specifically goes wrong at runtime — one that returns instead of throwing, and one written as an inferred `const` arrow (name the exact error code you get).

```ts
// Write your code here
```

### Exercise 2 — medium

Build the validation layer for a `POST /api/v1/payments` endpoint using assertion functions only — no schema library.

Types to define:

```ts
// type CardPayment   = { method: "card";   cardToken: string; cvcProvided: boolean }
// type BankPayment   = { method: "bank";   iban: string; accountHolder: string }
// type WalletPayment = { method: "wallet"; walletId: string; provider: "apple" | "google" }
// type PaymentMethodInput = CardPayment | BankPayment | WalletPayment
//
// interface CreatePaymentRequest {
//   readonly idempotencyKey: string;      // uuid v4 format
//   readonly orderId:        number;      // positive integer
//   readonly amountCents:    number;      // positive integer, <= 100_000_00
//   readonly currency:       "USD" | "EUR" | "GBP";
//   readonly payment:        PaymentMethodInput;
//   readonly metadata:       Readonly<Record<string, string>> | null;
// }
```

Requirements:

1. Write `assertCreatePaymentRequest(value: unknown): asserts value is CreatePaymentRequest`, composed from smaller assertion functions (`assertRecord`, `assertUuid`, `assertCurrency`, `assertPaymentMethodInput`, ...). Each must throw an `ApiError(400, "VALIDATION_FAILED", ...)` whose message names the exact failing field path (e.g. `"payment.iban must be 15–34 characters"`).
2. `assertPaymentMethodInput` must narrow the discriminated union correctly — validating `cardToken` only for `"card"`, `iban`/`accountHolder` only for `"bank"`, and so on — and must end with an `assertNever` on the `method` discriminant so an added method breaks the build.
3. Write `assertMetadata(value: unknown, field: string): asserts value is Readonly<Record<string, string>> | null` — `null` is allowed, otherwise it must be a plain object whose every value is a string, with at most 20 keys and no key longer than 40 characters.
4. Write a handler `handleCreatePayment(requestBody: unknown, auth: { userId: number; scopes: readonly string[] }): Promise<ApiResponse<{ paymentId: string }>>` that: asserts the scope `payments:write`, asserts the body, then calls a declared `paymentGateway.charge(...)`. The handler body below the assertions must have full type information with no casts.
5. Write a `describeMethod(payment: PaymentMethodInput): string` that switches on `method` and ends in `assertNever`. Then add a fourth method `{ method: "crypto"; chainId: number; address: string }` to the union in a comment and list every location that now fails to compile.
6. Write at least eight plain-function test cases (no test framework) asserting that specific bad payloads throw with the expected `field` path — including: missing `idempotencyKey`, non-integer `amountCents`, `amountCents` of `0`, unknown `currency`, a `card` payment carrying `iban`, `metadata` with a numeric value, `metadata` with 21 keys, and a valid payload that must **not** throw.

```ts
// Write your code here
```

### Exercise 3 — hard

Design and implement a complete assertion toolkit for a service, then reason about its limits.

**Part A — the toolkit.** In one module, implement:

- `invariant(condition: unknown, message: string): asserts condition` throwing `InvariantError`
- `invariantLazy(condition: unknown, message: () => string): asserts condition`
- `assertDefined<T>(value: T | null | undefined, label: string): asserts value is T`
- `assertNever(value: never, context: string): never`
- `assertUnreachable(value: never, context: string): asserts value is never`
- A `Guards` class with an `assertOpen(): asserts this is Guards & { connection: DbConnection }` method
- A `makeAssertion<T>(predicate: (value: unknown) => value is T, label: string)` factory that returns an assertion function, and demonstrate the **exact annotation** a caller must write on the receiving `const` to get narrowing

For each helper, write a one-line comment stating precisely what a caller may assume after the call returns.

**Part B — predicate/assertion duality.** For a `UserRow` type with `userId: number`, `email: string`, `role: "member" | "support" | "admin"`, `deletedAt: Date | null`:

- Write `isUserRow(value: unknown): value is UserRow` as the primitive
- Derive `assertUserRow(value: unknown): asserts value is UserRow` from it
- Show one call site where only the predicate works (`.filter()` over `unknown[]`), one where only the assertion reads well (a single value at the top of a handler), and one where both work but the assertion is shorter
- Explain in comments why deriving the assertion from the predicate — rather than the reverse — is the right dependency direction

**Part C — the state machine.** Model an order lifecycle with `type OrderStatus = "draft" | "pending_payment" | "paid" | "packed" | "shipped" | "delivered" | "cancelled" | "refunded"` and write assertion functions that narrow `Order` to state-specific sub-types (`PaidOrder`, `PackedOrder`, `ShippedOrder`) so that `pack(order)`, `ship(order)`, and `deliver(order)` can only accept an order that is provably in the right state. Then write `advance(order: Order): Promise<Order>` with a `switch` over all eight statuses ending in `assertNever`. Add a ninth status `"returned"` in a comment and list every compile error it produces.

**Part D — the unsound cases.** Write four short, runnable examples that each demonstrate a place where assertion narrowing is *wrong* at runtime despite compiling:

1. Narrowing a mutable object property, then `await`ing something that clears it
2. Narrowing a `let` and reassigning it inside a callback
3. An assertion function whose body has a bug and returns on an invalid value
4. `asserts this is T` on a class whose method mutates the asserted field afterwards

For each, write the corrected version and state in a comment what structural change (not what extra check) made it safe.

**Part E — cost and boundaries.** Write `ingestUserRows(rows: unknown): readonly UserRow[]` two ways: one asserting every element inside the consuming loop, one asserting the whole array once at the boundary. Add comments quantifying which does more work for a 250 000-row import, and state the rule you would put in a code-review checklist about where assertion functions belong in a request lifecycle.

```ts
// Write your code here
```

---

## Quick reference cheat sheet

```ts
// ── The two forms ───────────────────────────────────────────────────────────
function assertIsT(v: unknown): asserts v is SomeType { if (bad) throw new Error(); }
function invariant(c: unknown, msg: string): asserts c { if (!c) throw new Error(msg); }

// ── Call sites narrow for the REST of the flow ──────────────────────────────
assertIsT(requestBody);      // requestBody: SomeType from here on
invariant(user !== null, "user required");   // user: User from here on

// ── The annotation rule (ts2775) ────────────────────────────────────────────
function f(v: unknown): asserts v is string {}          // ✅ declaration — works
const g = (v: unknown): asserts v is string => {};      // ❌ ts(2775) at CALL site
const h: (v: unknown) => asserts v is string = (v) => {}; // ✅ annotated const
class C { m(v: unknown): asserts v is string {} }       // ✅ method
const obj = { m(v: unknown): asserts v is string {} };  // ❌ ts(2775) — obj inferred
const obj2: { m(v: unknown): asserts v is string } = { m(v) {} };  // ✅

// ── Predicate vs assertion ──────────────────────────────────────────────────
function isAdmin(u: AppUser): u is AdminUser { return u.role === "admin"; }
function assertIsAdmin(u: AppUser): asserts u is AdminUser {
  if (!isAdmin(u)) throw ApiError.forbidden("admin required");
}
users.filter(isAdmin);       // ✅ predicate only — assertions cannot filter
assertIsAdmin(currentUser);  // ✅ assertion only — no branch needed

// ── Exhaustiveness ──────────────────────────────────────────────────────────
function assertNever(value: never, context: string): never {
  throw new Error(`Unhandled ${context}: ${JSON.stringify(value)}`);
}
switch (status) {
  case "paid":    return "Paid";
  case "shipped": return "Shipped";
  default:        return assertNever(status, "OrderStatus");  // ← breaks on new members
}
// `: never` stops control flow (satisfies "all paths return").
// `: asserts v is never` does NOT — it only narrows.

// ── Node's assert ───────────────────────────────────────────────────────────
import assert from "node:assert/strict";
assert(authToken !== undefined, "token required");   // asserts condition
assert.ok(order, "order required");                  // asserts value
assert.strictEqual(status, "paid");                  // asserts actual is T
// ⚠️ throws AssertionError → maps to 500. Use your own ApiError on request paths.

// ── this-assertions ─────────────────────────────────────────────────────────
class Conn {
  socket: Socket | null = null;
  assertOpen(): asserts this is Conn & { socket: Socket } {
    if (this.socket === null) throw new Error("not open");
  }
}

// ── Safe usage patterns ─────────────────────────────────────────────────────
const { user } = ctx; invariant(user !== null, "u");   // snapshot before asserting
const order = await repo.findById(id); assertDefined(order, "order");  // bind, then assert
// assertDefined(await repo.findById(id), "order");    // ❌ nothing to narrow
```

| Construct | Returns | Narrows | Needs explicit annotation on ref | Stops control flow | Runtime check |
|---|---|---|---|---|---|
| `v is T` (predicate) | `boolean` | inside `if`/`filter` | no | no | **yes (your body)** |
| `asserts v is T` | `void` | rest of the flow | **yes** | no | **yes (your body)** |
| `asserts v` | `void` | rest of the flow | **yes** | no | **yes (your body)** |
| `asserts this is T` | `void` | rest of the flow | **yes** | no | **yes (your body)** |
| `: never` return | `never` | n/a | **yes** | **yes** | **yes (your body)** |
| `v as T` | value | expression only | no | no | **no — emits nothing** |
| `v!` | value | expression only | no | no | **no — emits nothing** |

| Question | Answer |
|---|---|
| Does TS verify the body implements the assertion? | **No.** An empty body compiles. |
| Can an assertion return a value too? | No — the return type must be `void`-compatible. |
| Can it narrow `await someCall()` directly? | No — bind to a `const` first. |
| Can it be a `.filter()` / `.forEach()` callback? | No — use a type predicate. |
| Does narrowing survive an `await`? | In the type system yes; at runtime only if you snapshotted to a `const`. |
| Does it survive into a closure? | Only over `const`, never over `let`. |
| Does it emit runtime code? | Yes, fully — unlike `!` and `as`. |
| Which error should it throw? | `ApiError` (4xx) for input, `InvariantError` (5xx) for bugs. |

---

## Connected topics

- **40 — Type narrowing** — the control-flow analysis that assertion calls hook into; every limit of `if`-narrowing applies to `asserts` too.
- **41 — Type guards** — `value is T` predicates: when to write one instead of an assertion, and how to derive one from the other.
- **62 — Strict null checks** — assertion functions are the honest replacement for `!`; without `strictNullChecks` they narrow nothing.
- **09 — Union types** — discriminated unions are what `asserts x is T` and `assertNever` exist to make safe.
- **07 — Special types** — `never`, `unknown`, and `void`: why `assertNever` returns `never` and why validators take `unknown` rather than `any`.
- **26 — Type assertions (`as`)** — the unchecked cousin; assertion functions concentrate the same trust into one testable place.
- **44 — Discriminated unions** — the pattern that makes exhaustiveness checking worth the four lines of `assertNever`.
- **03 — tsconfig in depth** — `strict`, and why assertion helpers belong in a module compiled with it.
