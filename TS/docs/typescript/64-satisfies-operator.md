# 64 — Satisfies operator

## What is this?

`satisfies` is an operator (TypeScript **4.9+**) that says to the compiler:

> *"Check this expression against this type. If it does not match, error. But do **not** change the type you inferred for it."*

That last clause is the whole point. Every other tool you have — a type annotation, a type assertion — **replaces** the inferred type with the one you wrote. `satisfies` **validates against** it and then throws the constraint away, leaving you with the precise, literal, narrow type the compiler worked out on its own.

```ts
// ── A type annotation VALIDATES but WIDENS ──────────────────────────────────
const routeStatus: Record<string, number> = {
  createUser: 201,
  getUser:    200,
};
routeStatus.createUser;   // number      ← lost the literal 201
routeStatus.deleteUsr;    // number      ← typo compiles! Record<string, …> allows any key

// ── A type assertion WIDENS *and* SKIPS THE CHECK ───────────────────────────
const routeStatus2 = {
  createUser: 201,
  getUsr:     200,        // typo — nobody will ever tell you
} as Record<"createUser" | "getUser", number>;
routeStatus2.createUser;  // number      ← still lost the literal

// ── `satisfies` VALIDATES and KEEPS the inferred type ───────────────────────
const routeStatus3 = {
  createUser: 201,
  getUser:    200,
} satisfies Record<string, number>;
routeStatus3.createUser;  // 201         ← literal survived
routeStatus3.deleteUsr;   // ❌ Property 'deleteUsr' does not exist
```

The mental model: `satisfies` is a **type-level assertion that leaves no trace**. It is a compile-time unit test on a value. It emits absolutely nothing at runtime — `expr satisfies T` compiles to `expr`, exactly like `as T` does.

Its natural companions:

- `as const` — freeze an object literal into its narrowest possible type
- `as const satisfies T` — the power combination: narrowest possible type, *and* verified against a contract
- `satisfies Record<UnionKey, V>` — exhaustiveness checking over a union of keys
- `satisfies` on function expressions, arrays, tuples, and nested config trees

---

## Why does it matter?

Backend code is full of **registries**: objects whose *keys* and *values* are both meaningful and both worth typing precisely.

- A route table mapping `"POST /users"` → handler
- An error-code registry mapping `"USER_NOT_FOUND"` → `{ status: 404, message: string }`
- A permissions matrix mapping `"admin" | "editor" | "viewer"` → allowed actions
- A queue registry mapping job name → payload schema
- A feature-flag defaults object
- Environment presets: `development` | `staging` | `production` → config

For every one of these you want **two things at once**:

1. **Validation** — "every value in here must be a `RouteConfig`", "every role in the union must have an entry", "no typo'd keys".
2. **Precision** — "`ERROR_CODES.USER_NOT_FOUND.status` is `404`, not `number`"; "`Object.keys(ROUTES)` gives me a union of the real route names"; "`CONFIG.production.region` is `\"us-east-1\"` so I can feed it to a function that only accepts real regions".

Before `satisfies`, those two goals were **mutually exclusive**. Annotating gave you (1) and destroyed (2). Omitting the annotation gave you (2) and destroyed (1). People invented an entire genre of generic helper functions (`function defineConfig<T extends Record<string, RouteConfig>>(cfg: T): T { return cfg }`) purely to work around the gap.

`satisfies` closes the gap in one keyword. That is why it shipped, and that is why it appears in nearly every modern TypeScript config file, ORM schema, and router definition written since 2022.

---

## The JavaScript way vs the TypeScript way

```js
// ── JavaScript — a config object is just an object. Nothing checks it. ────────

const ERROR_CODES = {
  USER_NOT_FOUND:   { status: 404, message: "User not found" },
  INVALID_TOKEN:    { status: 401, message: "Invalid or expired token" },
  RATE_LIMITED:     { status: 249, message: "Too many requests" },  // typo: 249 not 429
  VALIDATION_ERROR: { status: 422 },                                // forgot `message`
  // FORBIDDEN is missing entirely — nobody noticed
};

function sendError(res, code) {
  const entry = ERROR_CODES[code];
  // If `code` was misspelled, entry is undefined:
  res.status(entry.status).json({ error: entry.message, code });
}

sendError(res, "USER_NOTFOUND");   // 💥 Cannot read properties of undefined
sendError(res, "RATE_LIMITED");    // ships a 249 status to clients forever
sendError(res, "VALIDATION_ERROR");// {"error": undefined}
```

Three separate categories of bug, none of them detectable until production:

- **Typo'd key at the call site** → runtime crash.
- **Wrong-shaped value** → `undefined` leaking into the response body.
- **Missing entry** → an error code your handler throws that the registry has never heard of.

The JS mitigation is a runtime validation loop at boot (`for (const [k, v] of Object.entries(ERROR_CODES)) assert(...)`) — which catches shape errors but *still* cannot catch a typo at a call site, and *still* cannot tell you a key is missing, because JS has no idea what the complete set of keys should be.

Now the first TypeScript attempt — the obvious one, with an annotation:

```ts
// ── TypeScript, attempt 1 — annotation. Validates, but destroys precision. ────

type ErrorCode =
  | "USER_NOT_FOUND"
  | "INVALID_TOKEN"
  | "RATE_LIMITED"
  | "VALIDATION_ERROR"
  | "FORBIDDEN";

interface ErrorDefinition {
  status:  number;
  message: string;
  retryable?: boolean;
}

const ERROR_CODES: Record<ErrorCode, ErrorDefinition> = {
  USER_NOT_FOUND:   { status: 404, message: "User not found" },
  INVALID_TOKEN:    { status: 401, message: "Invalid or expired token" },
  RATE_LIMITED:     { status: 429, message: "Too many requests", retryable: true },
  VALIDATION_ERROR: { status: 422, message: "Request validation failed" },
  FORBIDDEN:        { status: 403, message: "Forbidden" },
};
// ✅ WINS: missing FORBIDDEN → compile error. Missing `message` → compile error.
//          Typo'd key → compile error.

// ❌ LOSSES — every single value is now widened to the annotation:
ERROR_CODES.RATE_LIMITED.status;
//          ^? number    ← not 429. You cannot narrow on it, switch on it exhaustively,
//                          or feed it to a function expecting `429 | 503`.

ERROR_CODES.USER_NOT_FOUND.retryable;
//          ^? boolean | undefined   ← even though we KNOW it was never set

// And the killer: you cannot derive anything from the object anymore.
type Codes = keyof typeof ERROR_CODES;
//   ^? ErrorCode   ← ok here, but only because we already had ErrorCode.
//      For an object with no pre-existing union, `Record<string, X>` gives you
//      `keyof typeof obj = string` and you get NOTHING.
```

```ts
// ── TypeScript, attempt 2 — no annotation. Precise, but unvalidated. ──────────

const ERROR_CODES = {
  USER_NOT_FOUND:   { status: 404, message: "User not found" },
  INVALID_TOKEN:    { status: 401, message: "Invalid or expired token" },
  RATE_LIMITED:     { status: 429, message: "Too many requests", retryable: true },
  VALIDATION_ERROR: { status: 422 },     // ❌ missing `message` — NOBODY CHECKS
  // FORBIDDEN missing entirely          // ❌ NOBODY CHECKS
};

ERROR_CODES.RATE_LIMITED.status;   // number  ← still widened! (object literals widen
                                   //   number/string properties unless `as const`)
```

```ts
// ── TypeScript, attempt 3 — `as const satisfies T`. Both. At once. ───────────

const ERROR_CODES = {
  USER_NOT_FOUND:   { status: 404, message: "User not found" },
  INVALID_TOKEN:    { status: 401, message: "Invalid or expired token" },
  RATE_LIMITED:     { status: 429, message: "Too many requests", retryable: true },
  VALIDATION_ERROR: { status: 422, message: "Request validation failed" },
  FORBIDDEN:        { status: 403, message: "Forbidden" },
} as const satisfies Record<ErrorCode, ErrorDefinition>;

// ✅ VALIDATION — every one of these is now a compile error:
//    - omitting FORBIDDEN            → "Property 'FORBIDDEN' is missing"
//    - `USER_NOTFOUND` (typo)        → "Object literal may only specify known properties"
//    - `{ status: 422 }`             → "Property 'message' is missing"
//    - `{ status: "404" }`           → "Type 'string' is not assignable to type 'number'"

// ✅ PRECISION — every literal survives:
ERROR_CODES.RATE_LIMITED.status;      // 429
ERROR_CODES.RATE_LIMITED.retryable;   // true
ERROR_CODES.USER_NOT_FOUND.message;   // "User not found"

// ✅ DERIVATION — the object is now a source of types, not just data:
type KnownErrorCode  = keyof typeof ERROR_CODES;
//   ^? "USER_NOT_FOUND" | "INVALID_TOKEN" | "RATE_LIMITED" | "VALIDATION_ERROR" | "FORBIDDEN"
type KnownHttpStatus = (typeof ERROR_CODES)[KnownErrorCode]["status"];
//   ^? 404 | 401 | 429 | 422 | 403

function sendError(res: Response, code: KnownErrorCode): void {
  const entry = ERROR_CODES[code];       // never undefined — key set is exact
  res.status(entry.status).json({ error: entry.message, code });
}

sendError(res, "USER_NOTFOUND");   // ❌ compile error, not a 3am page
```

The satisfying part: you wrote the data **once**, in plain object-literal form, with no ceremony — and got a validated contract, exact literal types, *and* a derived union of keys out of it. The object is now the single source of truth and the types follow it, instead of you maintaining a type and a value in parallel and hoping they stay in sync.

---

## Syntax

```ts
// ── Basic form: <expression> satisfies <type> ───────────────────────────────
const port = 8080 satisfies number;
//    ^? 8080          ← literal preserved (it's a const with a literal initializer)

const timeoutMs = 5_000 satisfies number;

// ── On object literals — the main use case ──────────────────────────────────
const dbConfig = {
  host: "db.internal",
  port: 5432,
  ssl:  true,
} satisfies { host: string; port: number; ssl: boolean };

dbConfig.host;   // string   ← ⚠️ widened, because plain object literals widen
dbConfig.port;   // number

// ── With `as const` — no widening at all ────────────────────────────────────
const dbConfig2 = {
  host: "db.internal",
  port: 5432,
  ssl:  true,
} as const satisfies { host: string; port: number; ssl: boolean };

dbConfig2.host;  // "db.internal"
dbConfig2.port;  // 5432
// dbConfig2.port = 5433;   // ❌ readonly

// ── On arrays and tuples ────────────────────────────────────────────────────
const allowedOrigins = [
  "https://app.example.com",
  "https://admin.example.com",
] as const satisfies readonly string[];
//    ^? readonly ["https://app.example.com", "https://admin.example.com"]

// ── On function expressions ─────────────────────────────────────────────────
type RequestHandler = (userId: number) => Promise<string>;

const getUserEmail = (async (userId) => {
  //                        ^? userId is contextually typed as number
  return `user-${userId}@example.com`;
}) satisfies RequestHandler;
//    ^? (userId: number) => Promise<string>

// ── Exhaustiveness over a union of keys ─────────────────────────────────────
type UserRole = "admin" | "editor" | "viewer";

const ROLE_RANK = {
  admin:  3,
  editor: 2,
  viewer: 1,
} as const satisfies Record<UserRole, number>;
// Adding "auditor" to UserRole later → ❌ compile error here. That is the point.

// ── Chained / nested ────────────────────────────────────────────────────────
const retryPolicy = {
  maxAttempts: 3,
  backoff: { kind: "exponential", baseMs: 200 } satisfies BackoffConfig,
} satisfies { maxAttempts: number; backoff: BackoffConfig };

// ── satisfies vs the two alternatives, side by side ─────────────────────────
const a: Record<string, number> = { requestsPerMinute: 60 };  // annotation
const b = { requestsPerMinute: 60 } as Record<string, number>; // assertion
const c = { requestsPerMinute: 60 } satisfies Record<string, number>; // satisfies

a.requestsPerMinute;  // number, and a.anythingElse also compiles
b.requestsPerMinute;  // number, and b.anythingElse also compiles
c.requestsPerMinute;  // number (widened prop), but c.anythingElse is an ERROR

// ── Declarations used above ─────────────────────────────────────────────────
interface BackoffConfig { kind: "fixed" | "exponential"; baseMs: number }
```

---

## How it works — concept by concept

### Concept 1 — The three-way comparison: `: Type` vs `as Type` vs `satisfies Type`

This is the single most important table in the document. Three operations, three different jobs:

| | Checks the value? | Resulting type | Excess keys? | Missing keys? |
|---|---|---|---|---|
| `const x: T = expr` | ✅ yes | **`T`** (widened) | ❌ error (fresh literal) | ❌ error |
| `const x = expr as T` | ⚠️ only "overlap" | **`T`** (widened) | ✅ silently allowed | ✅ silently allowed |
| `const x = expr satisfies T` | ✅ yes | **`typeof expr`** (inferred) | ❌ error | ❌ error |

Worked through concretely:

```ts
interface RateLimitRule {
  windowSeconds: number;
  maxRequests:   number;
  burst?:        number;
}

// ── 1. ANNOTATION — the variable's type BECOMES RateLimitRule ───────────────
const loginLimit: RateLimitRule = {
  windowSeconds: 60,
  maxRequests:   5,
  burst:         2,
};
loginLimit.windowSeconds;  // number ← the literal 60 is gone
loginLimit.burst;          // number | undefined ← even though we set it!
// loginLimit.rpm;         // ❌ Property 'rpm' does not exist on type 'RateLimitRule'

// ── 2. ASSERTION — you OVERRIDE the compiler. It barely checks. ─────────────
const signupLimit = {
  windowSeconds: 60,
  // maxRequests forgotten entirely
} as RateLimitRule;
// ✅ COMPILES. `as` only requires the types to "sufficiently overlap", and a
//    subset of the properties overlaps fine. You have lied, and the compiler
//    took your word for it.
signupLimit.maxRequests;   // number ← at runtime this is `undefined`. 💥

// ── 3. SATISFIES — checked like an annotation, typed like no annotation ────
const apiLimit = {
  windowSeconds: 60,
  maxRequests:   1000,
  burst:         50,
} satisfies RateLimitRule;

apiLimit.windowSeconds;    // number (widened by literal inference rules)
apiLimit.burst;            // number ← NOT `number | undefined`. We know it's set.
// apiLimit.rpm;           // ❌ Property 'rpm' does not exist

// ── The `burst` difference is the everyday payoff ──────────────────────────
function describe(limit: RateLimitRule): string {
  return limit.burst === undefined ? "no burst" : `burst ${limit.burst}`;
}
// With the ANNOTATION, you must check `burst` even when you literally just wrote it:
if (loginLimit.burst !== undefined) { /* forced */ }
// With SATISFIES, you don't:
apiLimit.burst.toFixed(0);   // ✅ number, no check needed
```

The one-line summary to remember:

- **`: T`** — "this variable *is* a T. Forget what I wrote."
- **`as T`** — "shut up, I know better." (unsound)
- **`satisfies T`** — "verify what I wrote is a valid T, then forget T."

### Concept 2 — What `satisfies` does to inference (and what it does not)

`satisfies` **does not** turn on `as const`. It does not freeze anything, and it does not prevent literal widening. What it does do is participate in inference as a *contextual type*, which sometimes preserves literals that would otherwise widen — and sometimes does not.

```ts
// ── No contextual pressure → normal widening rules apply ────────────────────
const a = { retries: 3 } satisfies { retries: number };
a.retries;   // number   ← widened, exactly as it would be with no `satisfies`

// ── Contextual type is a literal union → the literal is PRESERVED ───────────
type LogLevel = "debug" | "info" | "warn" | "error";

const b = { level: "warn" } satisfies { level: LogLevel };
b.level;     // "warn"   ← ✅ NOT widened to string, because the target property
             //              type is a union of literals, which acts as a
             //              contextual type and pins the literal.

// ── Compare: annotation gives you the whole union, not the one you wrote ────
const c: { level: LogLevel } = { level: "warn" };
c.level;     // LogLevel  ← "debug" | "info" | "warn" | "error"

// ── Numbers behave identically ─────────────────────────────────────────────
type HttpSuccess = 200 | 201 | 204;
const d = { status: 201 } satisfies { status: HttpSuccess };
d.status;    // 201      ← pinned

const e = { status: 201 } satisfies { status: number };
e.status;    // number   ← widened, because `number` gives no literal pressure
```

So the practical rule:

- Target property type is a **literal union** (`"a" | "b"`, `200 | 404`, `true`) → literal survives.
- Target property type is a **wide primitive** (`string`, `number`, `boolean`) → the literal widens.
- Want literals **regardless** of the target type → add `as const`.

```ts
const f = { retries: 3, level: "warn" } as const satisfies {
  retries: number;
  level:   LogLevel;
};
f.retries;   // 3
f.level;     // "warn"
// and both are `readonly`
```

This is why the idiom you see everywhere is `as const satisfies T` and not bare `satisfies T`: `as const` supplies the precision, `satisfies` supplies the validation, and neither does the other's job.

### Concept 3 — Excess property checking, and why it is stronger than you expect

Excess property checking (EPC) applies to **fresh object literals** assigned to a known target type. `satisfies` triggers it, exactly like an annotation does.

```ts
interface QueueConfig {
  name:          string;
  concurrency:   number;
  visibilityMs?: number;
}

// ── Direct literal → EPC fires ─────────────────────────────────────────────
const emailQueue = {
  name:        "email-outbox",
  concurrency: 4,
  conccurency: 8,          // ❌ Object literal may only specify known properties.
                           //    Did you mean to write 'concurrency'?
} satisfies QueueConfig;

// This is the typo-catcher. Without `satisfies`, that duplicate-ish key would
// just become part of the inferred type and silently do nothing.

// ── ⚠️ EPC does NOT fire through a variable ────────────────────────────────
const raw = { name: "email-outbox", concurrency: 4, conccurency: 8 };
const q2 = raw satisfies QueueConfig;   // ✅ COMPILES — `raw` is not a fresh literal
// The structural check still passes (raw has all required props of the right
// type), and EPC is deliberately skipped for non-fresh types.

// ── ✅ So always apply `satisfies` at the literal, never one hop later ─────
const q3 = { name: "email-outbox", concurrency: 4 } satisfies QueueConfig;
```

EPC also propagates into nested literals:

```ts
interface ServiceDefinition {
  name:    string;
  health:  { path: string; intervalSeconds: number };
  scaling: { min: number; max: number };
}

const ordersService = {
  name: "orders-api",
  health: {
    path:           "/healthz",
    intervalSecond: 10,       // ❌ caught — nested literals get EPC too
  },
  scaling: { min: 2, max: 20 },
} satisfies ServiceDefinition;
```

And it fires on values inside an index-signature target, which is the workhorse case:

```ts
const ROUTES = {
  "GET /users":      { handler: "listUsers",  auth: true },
  "POST /users":     { handler: "createUser", auth: false },
  "GET /users/:id":  { handler: "getUser",    auht: true },  // ❌ 'auht' caught
} satisfies Record<string, { handler: string; auth: boolean }>;
```

### Concept 4 — Missing keys, and `satisfies Record<Union, V>` as an exhaustiveness check

A `Record<SomeUnion, V>` target requires **every member of the union** to be a key. Combine that with `satisfies` and you get exhaustiveness checking on a data object — the same guarantee `never` gives you on a `switch`, but for a lookup table.

```ts
type SubscriptionPlan = "free" | "starter" | "pro" | "enterprise";

interface PlanLimits {
  maxProjects:       number;
  maxSeats:          number;
  apiRequestsPerDay: number;
  supportSla:        "none" | "business-hours" | "24x7";
}

const PLAN_LIMITS = {
  free:       { maxProjects: 1,   maxSeats: 1,   apiRequestsPerDay: 1_000,    supportSla: "none" },
  starter:    { maxProjects: 5,   maxSeats: 5,   apiRequestsPerDay: 50_000,   supportSla: "business-hours" },
  pro:        { maxProjects: 50,  maxSeats: 25,  apiRequestsPerDay: 500_000,  supportSla: "business-hours" },
  enterprise: { maxProjects: 999, maxSeats: 999, apiRequestsPerDay: 10_000_000, supportSla: "24x7" },
} as const satisfies Record<SubscriptionPlan, PlanLimits>;

// Tomorrow, someone adds a plan:
//   type SubscriptionPlan = "free" | "starter" | "pro" | "enterprise" | "team";
// → ❌ Property 'team' is missing in type '{ free: ...; }' but required in type
//      'Record<SubscriptionPlan, PlanLimits>'.
//
// The compiler now walks you to EVERY table that needs the new plan. That is
// the whole reason to write `Record<Union, V>` rather than `Record<string, V>`.
```

And crucially, the values are still exact:

```ts
PLAN_LIMITS.pro.maxSeats;       // 25
PLAN_LIMITS.free.supportSla;    // "none"

// Which means you can derive types from the data:
type PlanName  = keyof typeof PLAN_LIMITS;        // "free"|"starter"|"pro"|"enterprise"
type SlaTier   = (typeof PLAN_LIMITS)[PlanName]["supportSla"];  // "none"|"business-hours"|"24x7"
type SeatCount = (typeof PLAN_LIMITS)[PlanName]["maxSeats"];    // 1 | 5 | 25 | 999
```

Contrast with what happens if you use a plain annotation:

```ts
const PLAN_LIMITS_ANNOTATED: Record<SubscriptionPlan, PlanLimits> = { /* same data */ } as Record<SubscriptionPlan, PlanLimits>;
type SlaTierBad = (typeof PLAN_LIMITS_ANNOTATED)[SubscriptionPlan]["supportSla"];
//   ^? "none" | "business-hours" | "24x7"  ← this one happens to survive because
//      PlanLimits declares it as a literal union. But:
type SeatCountBad = (typeof PLAN_LIMITS_ANNOTATED)[SubscriptionPlan]["maxSeats"];
//   ^? number   ← gone. You can no longer prove `maxSeats` is one of four values.
```

### Concept 5 — Extra keys *are* allowed when the target has an index signature

There is one asymmetry that trips people up. With `Record<string, V>` the key set is open, so `satisfies` cannot catch a typo'd **key** — only a typo'd **property inside a value**.

```ts
const HANDLERS = {
  createUser: (body: unknown) => "created",
  updateUser: (body: unknown) => "updated",
} satisfies Record<string, (body: unknown) => string>;

HANDLERS.deleteUser;   // ❌ caught — but only because the INFERRED type has no
                       //    such key. `satisfies` kept the inferred type, and
                       //    the inferred type is exact. This is the win.
```

Read that carefully: the check that catches `HANDLERS.deleteUser` is **not** the `satisfies` check — it is the fact that `satisfies` *preserved the exact inferred type*, so property access is checked against `{ createUser: ...; updateUser: ... }` rather than against `Record<string, ...>`. With an annotation, `HANDLERS.deleteUser` would have compiled and returned `undefined` at runtime.

So `satisfies Record<string, V>` gives you:

- ✅ value-shape validation
- ✅ exact key set **at use sites** (via the preserved inferred type)
- ❌ *no* completeness guarantee (there is no finite key set to be complete against)

whereas `satisfies Record<Union, V>` gives you all three.

### Concept 6 — `satisfies` on functions: contextual parameter typing without losing the return type

Applying `satisfies` to a function expression contextually types its parameters (so you can drop the annotations) while preserving the **specific** return type.

```ts
type JobHandler = (payload: unknown) => Promise<void>;

// ── Annotation: parameters typed, return type flattened to the annotation ───
const sendWelcomeEmail: JobHandler = async (payload) => {
  //                                        ^? unknown — contextually typed ✅
  console.log(payload);
};
sendWelcomeEmail;   // JobHandler

// ── satisfies: parameters STILL contextually typed, return type preserved ───
type Reducer = (total: number, lineTotalCents: number) => number;

const sumLineItems = ((total, lineTotalCents) => {
  //                   ^? number, ^? number — contextually typed from Reducer ✅
  return total + lineTotalCents;
}) satisfies Reducer;
//    ^? (total: number, lineTotalCents: number) => number
```

Where it really pays off is when the implementation returns something *narrower* than the contract:

```ts
type StatusResolver = (userId: number) => "active" | "suspended" | "deleted";

// ── Annotation: the return type is the full union, forever ─────────────────
const alwaysActive: StatusResolver = (userId) => "active";
const s1 = alwaysActive(1);   // "active" | "suspended" | "deleted"

// ── satisfies: validated against StatusResolver, but the narrow return kept ─
const alwaysActive2 = ((userId: number) => "active" as const) satisfies StatusResolver;
const s2 = alwaysActive2(1);  // "active"   ← you can now prove it never suspends
```

Same trick on a map of handlers — every handler is validated, and every handler keeps its own precise signature:

```ts
interface CreateUserBody { email: string; password: string }
interface DeleteUserBody { userId: number }

const commandHandlers = {
  createUser: (body: CreateUserBody) => ({ userId: 1, email: body.email }),
  deleteUser: (body: DeleteUserBody) => ({ deleted: true }),
} satisfies Record<string, (body: never) => unknown>;

// Each handler kept its EXACT parameter and return type:
type CreateResult = ReturnType<typeof commandHandlers.createUser>;
//   ^? { userId: number; email: string }
type DeleteParam  = Parameters<typeof commandHandlers.deleteUser>[0];
//   ^? DeleteUserBody

// (`(body: never) => unknown` is the standard "any function" upper bound here:
//  parameters are contravariant, so `never` accepts any parameter type.)
```

### Concept 7 — `as const satisfies T`: order matters, and why

The two operators compose left to right: `as const` applies first, producing the deeply-readonly literal type; `satisfies` then validates *that* type against the target.

```ts
const CACHE_TTLS = {
  userProfile:  300,
  sessionToken: 900,
  featureFlags: 60,
} as const satisfies Record<string, number>;
//  ^^^^^^^^ 1. freeze          ^^^^^^^^^ 2. validate the frozen thing
```

The reverse order does not work:

```ts
// const bad = { a: 1 } satisfies Record<string, number> as const;
// ❌ 'as const' can only be applied to references to enum members, or string,
//    number, boolean, array, or object literals.
// (The result of `satisfies` is an expression, not a literal, so `as const`
//  has nothing to freeze.)
```

The important interaction: **`as const` makes everything `readonly`, so your target type must accept `readonly`**.

```ts
interface CorsConfig {
  origins: string[];       // ← mutable array
  credentials: boolean;
}

const cors = {
  origins: ["https://app.example.com"],
  credentials: true,
} as const satisfies CorsConfig;
// ❌ The type 'readonly ["https://app.example.com"]' is 'readonly' and cannot be
//    assigned to the mutable type 'string[]'.

// ✅ Fix the TARGET to accept readonly — this is almost always the right change:
interface CorsConfigRo {
  origins: readonly string[];
  credentials: boolean;
}
const cors2 = {
  origins: ["https://app.example.com", "https://admin.example.com"],
  credentials: true,
} as const satisfies CorsConfigRo;

cors2.origins;   // readonly ["https://app.example.com", "https://admin.example.com"]
type Origin = (typeof cors2.origins)[number];
//   ^? "https://app.example.com" | "https://admin.example.com"
```

That last line is the payoff — a union of the actual allowed origins, derived from the config, usable in a function signature so no other code can pass an origin that isn't in the list.

### Concept 8 — Deriving types *from* the value (the reason all of this matters)

`satisfies` turns a config object into a **type source**. This inverts the usual dependency: instead of writing a type and then a value that conforms to it, you write the value and derive the type.

```ts
const QUEUE_REGISTRY = {
  "email.welcome":       { concurrency: 5,  maxAttempts: 3, deadLetter: true  },
  "email.password-reset":{ concurrency: 10, maxAttempts: 5, deadLetter: true  },
  "billing.invoice":     { concurrency: 2,  maxAttempts: 8, deadLetter: true  },
  "analytics.pageview":  { concurrency: 50, maxAttempts: 1, deadLetter: false },
} as const satisfies Record<string, QueueOptions>;

interface QueueOptions {
  concurrency:  number;
  maxAttempts:  number;
  deadLetter:   boolean;
}

// ── Derivations, all free ──────────────────────────────────────────────────
type QueueName = keyof typeof QUEUE_REGISTRY;
//   ^? "email.welcome" | "email.password-reset" | "billing.invoice" | "analytics.pageview"

type QueueOf<N extends QueueName> = (typeof QUEUE_REGISTRY)[N];
type WelcomeOpts = QueueOf<"email.welcome">;
//   ^? { readonly concurrency: 5; readonly maxAttempts: 3; readonly deadLetter: true }

// ── Now the API is typo-proof, with autocomplete, and needs no runtime guard ─
function enqueue(queue: QueueName, payload: unknown): void {
  const opts = QUEUE_REGISTRY[queue];   // never undefined
  console.log(`enqueue on ${queue} with concurrency ${opts.concurrency}`, payload);
}

enqueue("email.welcome", { userId: 42 });
// enqueue("email.wellcome", {});   // ❌ Argument of type '"email.wellcome"' is not
//                                  //    assignable to parameter of type 'QueueName'.

// ── Filtering by a literal value — only possible because literals survived ──
type DeadLetterQueues = {
  [K in QueueName]: (typeof QUEUE_REGISTRY)[K]["deadLetter"] extends true ? K : never
}[QueueName];
//   ^? "email.welcome" | "email.password-reset" | "billing.invoice"
// With an annotation, `deadLetter` would be `boolean` and this would be `never`.
```

### Concept 9 — Where the error is reported, and how to read it

`satisfies` errors point at the **expression**, and the message names the target type. They read differently from annotation errors in a way worth internalising.

```ts
interface WebhookEndpoint {
  url:     string;
  events:  readonly string[];
  secret:  string;
  retries: number;
}

const stripeWebhook = {
  url:    "https://api.example.com/webhooks/stripe",
  events: ["invoice.paid", "invoice.payment_failed"],
  secret: "whsec_abc",
  // retries missing
} satisfies WebhookEndpoint;
// ❌ Type '{ url: string; events: string[]; secret: string; }' does not satisfy
//    the expected type 'WebhookEndpoint'.
//    Property 'retries' is missing in type '{ url: string; ... }' but required
//    in type 'WebhookEndpoint'.
//
// Note the wording: "does not SATISFY the expected type" — that phrase is unique
// to `satisfies` and is how you spot these errors in a wall of tsc output.
```

For big registries the error lands on the whole object, which can be hard to scan. A useful trick during debugging is to temporarily `satisfies` each entry individually to localise the failure:

```ts
const WEBHOOKS = {
  stripe: {
    url: "https://api.example.com/webhooks/stripe",
    events: ["invoice.paid"],
    secret: "whsec_abc",
    retries: 3,
  } satisfies WebhookEndpoint,          // ← per-entry check, precise error location
  github: {
    url: "https://api.example.com/webhooks/github",
    events: ["push", "pull_request"],
    secret: "ghs_xyz",
    retries: 5,
  } satisfies WebhookEndpoint,
} satisfies Record<string, WebhookEndpoint>;
// ⚠️ but note: per-entry `satisfies` here means the entries widen individually.
//    Use it to FIND the error, then remove it and keep only the outer one
//    (with `as const` if you want literals).
```

### Concept 10 — Runtime output: exactly zero

```ts
// TypeScript in:
const FEATURE_FLAGS = {
  newCheckout: true,
  betaSearch:  false,
} as const satisfies Record<string, boolean>;

export function isEnabled(flag: keyof typeof FEATURE_FLAGS): boolean {
  return FEATURE_FLAGS[flag];
}
```

```js
// JavaScript out (target ES2020):
const FEATURE_FLAGS = {
  newCheckout: true,
  betaSearch: false,
};
export function isEnabled(flag) {
  return FEATURE_FLAGS[flag];
}
```

`satisfies`, `as const`, and the type annotations all vanish. There is **no runtime validation**. If the object is built from `JSON.parse(configFile)` or `process.env`, `satisfies` proves nothing about it — you need a real validator (Zod, Valibot, hand-written guards) for that. `satisfies` is only ever a statement about **literals you wrote in source**.

Because `as const` also does not emit `Object.freeze`, the object is still mutable at runtime:

```ts
const flags = { newCheckout: true } as const satisfies Record<string, boolean>;
// flags.newCheckout = false;             // ❌ compile error
(flags as { newCheckout: boolean }).newCheckout = false;   // ✅ runtime mutation happens
// For genuine immutability you still need: Object.freeze(flags)
```

---

## Example 1 — basic

```ts
// A small HTTP error registry, built the way `satisfies` makes it worth building.

// ── The contract every entry must meet ──────────────────────────────────────
interface ErrorDefinition {
  readonly status:    number;
  readonly message:   string;
  readonly retryable: boolean;
}

// ── The closed set of codes this service can emit ───────────────────────────
type ApiErrorCode =
  | "USER_NOT_FOUND"
  | "EMAIL_TAKEN"
  | "INVALID_CREDENTIALS"
  | "TOKEN_EXPIRED"
  | "RATE_LIMITED"
  | "INTERNAL";

// ── The registry: `as const` for precision, `satisfies` for completeness ────
const API_ERRORS = {
  USER_NOT_FOUND:      { status: 404, message: "User not found",              retryable: false },
  EMAIL_TAKEN:         { status: 409, message: "Email already registered",    retryable: false },
  INVALID_CREDENTIALS: { status: 401, message: "Invalid email or password",   retryable: false },
  TOKEN_EXPIRED:       { status: 401, message: "Token expired",               retryable: false },
  RATE_LIMITED:        { status: 429, message: "Too many requests",           retryable: true  },
  INTERNAL:            { status: 500, message: "Internal server error",       retryable: true  },
} as const satisfies Record<ApiErrorCode, ErrorDefinition>;
//
// Try any of these and the build fails:
//   - delete the INTERNAL line          → "Property 'INTERNAL' is missing"
//   - rename EMAIL_TAKEN → EMAILTAKEN   → missing + excess property errors
//   - drop `retryable` from RATE_LIMITED→ "Property 'retryable' is missing"
//   - write `status: "429"`             → "Type 'string' is not assignable to 'number'"

// ── Everything below is derived — no second source of truth to drift ────────

type ErrorCode = keyof typeof API_ERRORS;
//   ^? same union as ApiErrorCode, but now *provably* in sync with the data

type ErrorStatus = (typeof API_ERRORS)[ErrorCode]["status"];
//   ^? 404 | 409 | 401 | 429 | 500     ← exact, because of `as const`

// A union of only the codes worth retrying — computed from the data:
type RetryableCode = {
  [K in ErrorCode]: (typeof API_ERRORS)[K]["retryable"] extends true ? K : never
}[ErrorCode];
//   ^? "RATE_LIMITED" | "INTERNAL"

// ── Using it ────────────────────────────────────────────────────────────────

class ApiError extends Error {
  readonly code:      ErrorCode;
  readonly status:    number;
  readonly retryable: boolean;

  constructor(code: ErrorCode, detail?: string) {
    const def = API_ERRORS[code];          // never undefined — the key set is exact
    super(detail === undefined ? def.message : `${def.message}: ${detail}`);
    this.code      = code;
    this.status    = def.status;
    this.retryable = def.retryable;
  }
}

function toHttpResponse(err: ApiError): { status: number; body: { code: string; message: string } } {
  return { status: err.status, body: { code: err.code, message: err.message } };
}

// ✅ autocomplete lists exactly six codes; a typo is a compile error:
throw new ApiError("USER_NOT_FOUND");
// throw new ApiError("USER_NOTFOUND");   // ❌ not assignable to 'ErrorCode'

// ── A function that only accepts retryable codes — impossible without `as const`
function scheduleRetry(code: RetryableCode, attempt: number): number {
  return Math.min(2 ** attempt * 250, 30_000);
}
scheduleRetry("RATE_LIMITED", 2);         // ✅
// scheduleRetry("USER_NOT_FOUND", 1);    // ❌ not assignable to 'RetryableCode'

// ── Compare with the annotation version, for contrast ──────────────────────
const API_ERRORS_ANNOTATED: Record<ApiErrorCode, ErrorDefinition> = API_ERRORS;
API_ERRORS_ANNOTATED.RATE_LIMITED.status;   // number   ← 429 is gone
API_ERRORS.RATE_LIMITED.status;             // 429      ← preserved
// Note the assignment above COMPILES: the `satisfies` object is a valid
// Record<ApiErrorCode, ErrorDefinition>. `satisfies` proved it. You can always
// hand the precise object to code that only wants the wide type.
```

---

## Example 2 — real world backend use case

```ts
// A full service-configuration and routing layer for a Node/Express backend,
// where `satisfies` is the mechanism that keeps five separate registries
// mutually consistent and typo-proof.

import type { Request, Response, NextFunction } from "express";

// ═══════════════════════════════════════════════════════════════════════════
// 1. PERMISSIONS MATRIX — exhaustive over two unions
// ═══════════════════════════════════════════════════════════════════════════

type UserRole   = "owner" | "admin" | "member" | "readonly";
type Permission = "users:read" | "users:write" | "billing:read" | "billing:write" | "audit:read";

// Every role must appear; every entry must be a list of REAL permissions.
const ROLE_PERMISSIONS = {
  owner:    ["users:read", "users:write", "billing:read", "billing:write", "audit:read"],
  admin:    ["users:read", "users:write", "billing:read", "audit:read"],
  member:   ["users:read"],
  readonly: ["users:read", "billing:read"],
} as const satisfies Record<UserRole, readonly Permission[]>;
//
// - Add a role to `UserRole` → this object fails to compile until you handle it.
// - Typo a permission ("user:read") → "not assignable to type 'Permission'".
// - `as const` keeps the exact tuples, so we can derive per-role unions:

type PermissionsOf<R extends UserRole> = (typeof ROLE_PERMISSIONS)[R][number];
type OwnerPerms  = PermissionsOf<"owner">;   // all five, as a union
type MemberPerms = PermissionsOf<"member">;  // "users:read"

// A compile-time check that no role can write billing except owner:
type BillingWriters = {
  [R in UserRole]: "billing:write" extends PermissionsOf<R> ? R : never
}[UserRole];
//   ^? "owner"   ← if someone grants admin billing:write, this type changes and
//                   a downstream `const _check: "owner" = ...` assertion breaks.

function hasPermission(role: UserRole, permission: Permission): boolean {
  // `readonly Permission[]` — the includes() call is fully typed:
  const granted: readonly Permission[] = ROLE_PERMISSIONS[role];
  return granted.includes(permission);
}

// ═══════════════════════════════════════════════════════════════════════════
// 2. ROUTE TABLE — key set derived from the data, values validated
// ═══════════════════════════════════════════════════════════════════════════

type HttpMethod = "GET" | "POST" | "PATCH" | "DELETE";

interface RouteDefinition {
  readonly method:      HttpMethod;
  readonly path:        string;
  readonly permission:  Permission | null;   // null = public route
  readonly rateLimit:   { readonly windowSeconds: number; readonly max: number };
  readonly idempotent:  boolean;
}

const ROUTES = {
  listUsers: {
    method: "GET", path: "/api/users", permission: "users:read",
    rateLimit: { windowSeconds: 60, max: 120 }, idempotent: true,
  },
  createUser: {
    method: "POST", path: "/api/users", permission: "users:write",
    rateLimit: { windowSeconds: 60, max: 20 }, idempotent: false,
  },
  getUser: {
    method: "GET", path: "/api/users/:userId", permission: "users:read",
    rateLimit: { windowSeconds: 60, max: 240 }, idempotent: true,
  },
  deleteUser: {
    method: "DELETE", path: "/api/users/:userId", permission: "users:write",
    rateLimit: { windowSeconds: 60, max: 10 }, idempotent: true,
  },
  health: {
    method: "GET", path: "/healthz", permission: null,
    rateLimit: { windowSeconds: 1, max: 100 }, idempotent: true,
  },
} as const satisfies Record<string, RouteDefinition>;
//
// - "GTE" instead of "GET"      → ❌ not assignable to 'HttpMethod'
// - `permission: "user:read"`   → ❌ not assignable to 'Permission | null'
// - `rateLimt: {...}` (typo)    → ❌ excess property + missing 'rateLimit'
// - forgetting `idempotent`     → ❌ Property 'idempotent' is missing

type RouteName = keyof typeof ROUTES;
//   ^? "listUsers" | "createUser" | "getUser" | "deleteUser" | "health"

// Public routes, computed from the data — no second list to forget to update:
type PublicRoute = {
  [K in RouteName]: (typeof ROUTES)[K]["permission"] extends null ? K : never
}[RouteName];
//   ^? "health"

// Every path, as a literal union — usable for typed URL building:
type RoutePath = (typeof ROUTES)[RouteName]["path"];
//   ^? "/api/users" | "/api/users/:userId" | "/healthz"

// ═══════════════════════════════════════════════════════════════════════════
// 3. HANDLER REGISTRY — must cover every route exactly once
// ═══════════════════════════════════════════════════════════════════════════

interface AuthenticatedRequest extends Request {
  auth?: { userId: number; role: UserRole };
}

type Handler = (req: AuthenticatedRequest, res: Response, next: NextFunction) => Promise<void>;

declare const userService: {
  list(page: number): Promise<{ userId: number; email: string }[]>;
  create(body: unknown): Promise<{ userId: number }>;
  get(userId: number): Promise<{ userId: number; email: string } | null>;
  remove(userId: number): Promise<void>;
};

// `Record<RouteName, Handler>` — a missing handler is a compile error, and an
// extra handler for a route that no longer exists is ALSO a compile error.
const HANDLERS = {
  listUsers: async (req, res) => {
    const users = await userService.list(1);
    res.status(200).json({ data: users });
  },
  createUser: async (req, res) => {
    const created = await userService.create(req.body);
    res.status(201).json({ data: created });
  },
  getUser: async (req, res) => {
    const raw = req.params.userId;
    const userId = raw === undefined ? Number.NaN : Number.parseInt(raw, 10);
    if (Number.isNaN(userId)) {
      res.status(400).json({ error: "Invalid userId" });
      return;
    }
    const user = await userService.get(userId);
    if (user === null) {
      res.status(404).json({ error: "User not found" });
      return;
    }
    res.status(200).json({ data: user });
  },
  deleteUser: async (req, res) => {
    const raw = req.params.userId;
    const userId = raw === undefined ? Number.NaN : Number.parseInt(raw, 10);
    if (Number.isNaN(userId)) {
      res.status(400).json({ error: "Invalid userId" });
      return;
    }
    await userService.remove(userId);
    res.status(204).send();
  },
  health: async (req, res) => {
    res.status(200).json({ status: "ok" });
  },
} satisfies Record<RouteName, Handler>;
//  ↑ NOTE: no `as const` here. Freezing function expressions gains nothing, and
//    `satisfies` already contextually types `req`/`res`/`next` for us — look,
//    none of the parameters above carry annotations.

// ═══════════════════════════════════════════════════════════════════════════
// 4. WIRING — one loop, fully typed, nothing can drift
// ═══════════════════════════════════════════════════════════════════════════

interface MinimalApp {
  get(path: string, ...h: Handler[]): void;
  post(path: string, ...h: Handler[]): void;
  patch(path: string, ...h: Handler[]): void;
  delete(path: string, ...h: Handler[]): void;
}

function requirePermission(permission: Permission | null): Handler {
  return async (req, res, next) => {
    if (permission === null) { next(); return; }
    const auth = req.auth;
    if (auth === undefined) {
      res.status(401).json({ error: "Unauthenticated" });
      return;
    }
    if (!hasPermission(auth.role, permission)) {
      res.status(403).json({ error: `Missing permission: ${permission}` });
      return;
    }
    next();
  };
}

function registerRoutes(app: MinimalApp): void {
  // Object.keys is `string[]`, so cast the iteration source once, deliberately:
  const routeNames = Object.keys(ROUTES) as RouteName[];

  for (const name of routeNames) {
    const route   = ROUTES[name];      // fully typed, never undefined
    const handler = HANDLERS[name];    // guaranteed to exist by the Record<RouteName, …>
    const guard   = requirePermission(route.permission);

    switch (route.method) {
      case "GET":    app.get(route.path, guard, handler); break;
      case "POST":   app.post(route.path, guard, handler); break;
      case "PATCH":  app.patch(route.path, guard, handler); break;
      case "DELETE": app.delete(route.path, guard, handler); break;
      default: {
        // Exhaustiveness: `route.method` is the literal union from the data,
        // so this branch is `never`. Add a method to HttpMethod → error here.
        const exhaustive: never = route.method;
        throw new Error(`Unhandled method: ${String(exhaustive)}`);
      }
    }
  }
}

// ═══════════════════════════════════════════════════════════════════════════
// 5. ENVIRONMENT PRESETS — literal values that feed typed APIs
// ═══════════════════════════════════════════════════════════════════════════

type Environment = "development" | "staging" | "production";
type AwsRegion   = "us-east-1" | "us-west-2" | "eu-west-1";

interface EnvironmentPreset {
  readonly region:            AwsRegion;
  readonly logLevel:          "debug" | "info" | "warn" | "error";
  readonly dbPoolSize:        number;
  readonly enableMetrics:     boolean;
  readonly corsOrigins:       readonly string[];
  readonly sessionTtlSeconds: number;
}

const ENVIRONMENTS = {
  development: {
    region: "us-east-1", logLevel: "debug", dbPoolSize: 2, enableMetrics: false,
    corsOrigins: ["http://localhost:3000", "http://localhost:5173"],
    sessionTtlSeconds: 86_400,
  },
  staging: {
    region: "us-east-1", logLevel: "info", dbPoolSize: 10, enableMetrics: true,
    corsOrigins: ["https://staging.example.com"],
    sessionTtlSeconds: 3_600,
  },
  production: {
    region: "us-west-2", logLevel: "warn", dbPoolSize: 50, enableMetrics: true,
    corsOrigins: ["https://app.example.com", "https://admin.example.com"],
    sessionTtlSeconds: 1_800,
  },
} as const satisfies Record<Environment, EnvironmentPreset>;

// Derived unions that no human maintains:
type ConfiguredRegion = (typeof ENVIRONMENTS)[Environment]["region"];
//   ^? "us-east-1" | "us-west-2"     ← eu-west-1 is unused; the type proves it
type AllowedOrigin = (typeof ENVIRONMENTS)[Environment]["corsOrigins"][number];
//   ^? "http://localhost:3000" | "http://localhost:5173" | "https://staging.example.com"
//      | "https://app.example.com" | "https://admin.example.com"

// A function that can only ever be called with an origin the config declares:
function isAllowedOrigin(origin: string): origin is AllowedOrigin {
  const all: readonly string[] = [
    ...ENVIRONMENTS.development.corsOrigins,
    ...ENVIRONMENTS.staging.corsOrigins,
    ...ENVIRONMENTS.production.corsOrigins,
  ];
  return all.includes(origin);
}

// Production's pool size is the literal 50, so it can be checked at compile time:
type ProdPoolSize = (typeof ENVIRONMENTS)["production"]["dbPoolSize"];
//   ^? 50
const _poolSizeIsSane: ProdPoolSize extends 50 ? true : never = true;
// ↑ a "type test": if someone changes production pool size, this line errors.
//   Cheap, zero-runtime regression guard on a value that matters.

function currentEnvironment(raw: string | undefined): Environment {
  // ⚠️ RUNTIME data — `satisfies` proves nothing here. A real check is required:
  const known = ["development", "staging", "production"] as const satisfies readonly Environment[];
  const match = known.find(e => e === raw);
  return match ?? "development";
}

export function loadPreset(env: NodeJS.ProcessEnv): EnvironmentPreset {
  return ENVIRONMENTS[currentEnvironment(env.NODE_ENV)];
}

// ═══════════════════════════════════════════════════════════════════════════
// 6. WHY THIS ALL HOLDS TOGETHER
// ═══════════════════════════════════════════════════════════════════════════
//
// - Add a permission to `Permission`     → nothing breaks (permissions are opt-in)
// - Add a role to `UserRole`             → ROLE_PERMISSIONS fails to compile ✅
// - Add an environment to `Environment`  → ENVIRONMENTS fails to compile ✅
// - Add a route to ROUTES                → HANDLERS fails to compile ✅
// - Remove a route from ROUTES           → HANDLERS fails (excess property) ✅
// - Typo any permission / method / role  → fails at the literal ✅
// - Change production's pool size        → the type test fails ✅
//
// None of that is possible with `: Record<...>` annotations, because the
// annotation would erase the literal types that every derivation depends on.
```

---

## Going deeper

### `satisfies` does not narrow — it only checks

A common mistaken expectation: that `x satisfies T` behaves like a type guard and narrows `x`. It does not. If the expression is already wide, `satisfies` leaves it wide.

```ts
declare const requestBody: unknown;

const body = requestBody satisfies { email: string };
// ❌ Type 'unknown' does not satisfy the expected type '{ email: string; }'.
// (`unknown` is not assignable to anything, so this is an error, not a narrowing.)

declare const maybeUser: { email: string } | null;
const u = maybeUser satisfies { email: string } | null;
u.email;   // ❌ 'u' is possibly 'null' — satisfies did NOT remove the null
```

`satisfies` answers "is this assignable?", not "make this be". For narrowing, use a type guard (`41 — Type guards`) or control-flow narrowing (`40 — Type narrowing`).

### The assignability direction: `expr` must be assignable to `T`, not the reverse

```ts
interface HasEmail { email: string }

// ✅ Extra properties in the TYPE of a non-fresh expression are fine:
const account = { email: "ops@example.com", userId: 7, createdAt: new Date() };
const checked = account satisfies HasEmail;   // ✅ account is assignable to HasEmail
checked.userId;   // number   ← the extra property survives in the result type

// ❌ But a FRESH literal triggers excess property checking:
const inline = { email: "ops@example.com", userId: 7 } satisfies HasEmail;
// ❌ Object literal may only specify known properties, and 'userId' does not
//    exist in type 'HasEmail'.
```

This asymmetry surprises everyone once. The rule is **freshness**: an object literal written directly at the `satisfies` site is "fresh" and gets EPC; a literal that has passed through a variable has lost its freshness and only gets structural assignability. If you want the extra-properties behaviour with an inline literal, widen the target:

```ts
const inline2 = { email: "ops@example.com", userId: 7 } satisfies HasEmail & Record<string, unknown>;
// ✅ compiles — the index signature absorbs the extra key
inline2.userId;   // number
```

### `satisfies` with generics: it does not infer type arguments for you the way you might hope

```ts
interface Paginated<T> { items: readonly T[]; total: number; cursor: string | null }

// ✅ Explicit type argument — checked, and items keeps its exact tuple type:
const page = {
  items: [{ userId: 1 }, { userId: 2 }],
  total: 2,
  cursor: null,
} satisfies Paginated<{ userId: number }>;
page.items;   // { userId: number }[]

// ⚠️ You cannot leave the argument to be inferred from the value:
// const page2 = { items: [...], total: 2, cursor: null } satisfies Paginated<infer T>;
// ❌ 'infer' declarations are only permitted in the 'extends' clause of a
//    conditional type.

// If you need inference from the value, you still need a generic helper —
// this is the one job `satisfies` did NOT replace:
function definePage<T>(page: Paginated<T>): Paginated<T> { return page; }
const page3 = definePage({ items: [{ userId: 1 }], total: 1, cursor: null });
//    ^? Paginated<{ userId: number }>
// ...but note definePage WIDENS to Paginated<T>, exactly like an annotation.
// To get both inference AND precision you need the constrained-identity trick:
function definePageExact<T extends Paginated<unknown>>(page: T): T { return page; }
const page4 = definePageExact({ items: [{ userId: 1 }], total: 1, cursor: null });
//    ^? { items: { userId: number }[]; total: number; cursor: null }   ← precise
```

`satisfies` replaced the *non-generic* identity-function idiom entirely. The generic one (`definePageExact`) still has a niche: when the target type itself depends on the value.

### `as const satisfies` and the `readonly` mismatch, in both directions

```ts
// ── Direction 1: `as const` produces readonly, target wants mutable → error ──
const a = { tags: ["billing", "urgent"] } as const satisfies { tags: string[] };
// ❌ readonly ["billing","urgent"] is not assignable to string[]

// ── Direction 2: target is readonly, `as const` omitted → fine ──────────────
const b = { tags: ["billing", "urgent"] } satisfies { tags: readonly string[] };
b.tags;   // string[]   ← still MUTABLE at the type level; satisfies did not
          //               make it readonly. `satisfies` never changes the type.
b.tags.push("new");   // ✅ compiles

// ── Only `as const` confers readonly ────────────────────────────────────────
const c = { tags: ["billing", "urgent"] } as const satisfies { tags: readonly string[] };
c.tags;   // readonly ["billing", "urgent"]
// c.tags.push("new");   // ❌ Property 'push' does not exist on readonly type
```

The mental model that avoids all confusion: **`satisfies` is a filter on the right-hand side, not a transformer.** It can only reject; it can never modify.

### Deep `as const` vs shallow `Readonly<T>`

`as const` is deep. `Readonly<T>` is one level. This matters when writing the target type:

```ts
interface WorkerConfig {
  name:    string;
  retries: { max: number; backoffMs: number };
}

const indexer = {
  name: "search-indexer",
  retries: { max: 5, backoffMs: 1_000 },
} as const satisfies WorkerConfig;
// ✅ compiles — a deeply-readonly object IS assignable to a fully-mutable
//    interface, because readonly-ness of PROPERTIES is not checked in
//    assignability (only readonly ARRAYS/TUPLES are). Surprising but true:

const mutable: WorkerConfig = indexer;   // ✅ compiles!
mutable.retries.max = 99;                // ✅ and mutates the "const" object
// This is a known unsoundness in TypeScript's treatment of `readonly` properties.
// `readonly` on properties is advisory; `readonly` on arrays is enforced.
```

That is why the array case errors and the object case does not. If you want the object case to error too, model the target with `readonly` explicitly and it still will not — there is no way to force it. Use arrays/tuples if you need the guarantee, or lean on `Object.freeze` at runtime.

### `satisfies` on `enum`-like patterns beats `enum`

```ts
// ── The old way — a TS enum ────────────────────────────────────────────────
enum OrderStatusEnum { Pending = "pending", Shipped = "shipped", Cancelled = "cancelled" }
// Emits a runtime object, has nominal-ish typing quirks, does not work with
// `isolatedModules` + `const enum`, and `OrderStatusEnum.Pending` is not
// assignable from the plain string "pending" in some positions.

// ── The satisfies way — plain data, exact types, zero surprises ────────────
const ORDER_STATUS = {
  pending:   { label: "Pending",   terminal: false, nextStates: ["shipped", "cancelled"] },
  shipped:   { label: "Shipped",   terminal: true,  nextStates: [] },
  cancelled: { label: "Cancelled", terminal: true,  nextStates: [] },
} as const satisfies Record<string, { label: string; terminal: boolean; nextStates: readonly string[] }>;

type OrderStatus = keyof typeof ORDER_STATUS;   // "pending" | "shipped" | "cancelled"

// And a state machine transition check, derived from the data:
type NextStatesOf<S extends OrderStatus> = (typeof ORDER_STATUS)[S]["nextStates"][number];
type AfterPending = NextStatesOf<"pending">;    // "shipped" | "cancelled"
type AfterShipped = NextStatesOf<"shipped">;    // never  ← terminal, provably

function transition<S extends OrderStatus>(from: S, to: NextStatesOf<S>): OrderStatus {
  return to as OrderStatus;
}
transition("pending", "shipped");     // ✅
// transition("shipped", "pending");  // ❌ Argument of type '"pending"' is not
//                                    //    assignable to parameter of type 'never'.
```

An enum could never express "the legal next states of `shipped` is the empty set" as a type.

### Errors are reported against the *whole* expression, which hurts on huge objects

For a 200-entry registry, a single missing property produces one error whose message includes the inferred type of the entire object — often thousands of characters, truncated with `...`. Mitigations:

1. Split the registry into per-domain files, each with its own `satisfies`.
2. Use a helper type that produces a *targeted* error:

```ts
// Instead of `satisfies Record<RouteName, Handler>` on a 200-key object,
// validate key coverage separately from value shape:

type MissingKeys<Actual, Required extends string> = Exclude<Required, keyof Actual>;

const handlers = { /* ...200 entries... */ };
type _Missing = MissingKeys<typeof handlers, RouteName>;
const _coverageCheck: _Missing extends never ? true : ["Missing handlers:", _Missing] = true;
// ↑ the error message now literally NAMES the missing route in the tuple type.
```

3. `tsc --noErrorTruncation` when you genuinely need to read the full inferred type.

### `satisfies` in `.d.ts` files and at the type level

`satisfies` is an **expression** operator. It cannot appear in a type position, and it cannot appear in an ambient declaration:

```ts
// type X = { a: 1 } satisfies { a: number };   // ❌ syntax error
// declare const y: { a: 1 } satisfies { a: number };   // ❌ syntax error
```

The type-level equivalent — "assert that type A is assignable to type B" — is a conditional-type assertion:

```ts
type Assert<A extends B, B> = A;
type _CheckErrors = Assert<typeof API_ERRORS, Record<ApiErrorCode, ErrorDefinition>>;
// ❌ if the constraint fails

// Or the more readable "expect extends" helper:
type Expect<T extends true> = T;
type Extends<A, B> = A extends B ? true : false;
type _Check = Expect<Extends<typeof API_ERRORS, Record<ApiErrorCode, ErrorDefinition>>>;
```

Use these when the value lives in another module and you want a check without importing it into an expression position.

### Version and tooling notes

- **Requires TypeScript 4.9+** (released Nov 2022). On 4.8 and below it is a syntax error, not a "type error" — the parser rejects it.
- Babel supports it from `@babel/plugin-transform-typescript` 7.20+; esbuild from 0.15.13; swc from 1.3.x. All simply erase it.
- Prettier formats it from 2.8+. Older Prettier will fail to parse the file.
- ESLint needs `@typescript-eslint/parser` 5.42+.
- If your editor shows a red squiggle under `satisfies` in a project that builds fine, your editor is using a bundled TypeScript older than the workspace one — switch to "Use Workspace Version".

### The `satisfies` + `Object.entries` interaction

`Object.entries` and `Object.keys` are typed loosely by design (they return `string` keys because of prototype/inherited-key soundness concerns). `satisfies` does not fix that; you still need a cast or a helper:

```ts
const RETRY_BUDGETS = {
  "payments.charge":  { attempts: 5, timeoutMs: 10_000 },
  "email.send":       { attempts: 3, timeoutMs: 5_000 },
  "search.index":     { attempts: 1, timeoutMs: 2_000 },
} as const satisfies Record<string, { attempts: number; timeoutMs: number }>;

type OperationName = keyof typeof RETRY_BUDGETS;

// ❌ loses the literal keys:
for (const [op, budget] of Object.entries(RETRY_BUDGETS)) {
  op;   // string
}

// ✅ a typed helper, written once:
function typedEntries<T extends object>(obj: T): [keyof T, T[keyof T]][] {
  return Object.entries(obj) as [keyof T, T[keyof T]][];
}
for (const [op, budget] of typedEntries(RETRY_BUDGETS)) {
  op;       // OperationName
  budget;   // the union of all three budget shapes
}
```

The cast inside `typedEntries` is genuinely unavoidable and genuinely safe for object literals you own. Isolating it in one helper is the right trade.

---

## Common mistakes

### Mistake 1 — Using `satisfies` where an annotation was correct

`satisfies` is for values whose *precise inferred type you want to keep*. For a variable that is going to be reassigned, or whose only job is to be passed to something expecting the wide type, an annotation is simpler and communicates intent better.

```ts
// ❌ Wrong — `satisfies` on a mutable variable does not constrain reassignment:
let currentConfig = { retries: 3, timeoutMs: 5_000 } satisfies RetryConfig;
currentConfig = { retries: 1, timeoutMs: 100 };            // fine
currentConfig = { retries: 1 } as RetryConfig;             // 😬 also "fine"
// Worse — the inferred type is `{ retries: number; timeoutMs: number }`, so:
currentConfig = { retries: 1, timeoutMs: 100, extra: 9 };  // ❌ error you did NOT want
// ...but a genuinely-different-but-valid RetryConfig might also be rejected,
// because the variable's type came from the FIRST literal, not from RetryConfig.

// ✅ Right — an annotation, because the variable holds "any RetryConfig over time":
let currentConfig: RetryConfig = { retries: 3, timeoutMs: 5_000 };
currentConfig = { retries: 1, timeoutMs: 100 };            // ✅
// currentConfig = { retries: 1 };                          // ❌ correctly rejected

interface RetryConfig { retries: number; timeoutMs: number }
```

**Rule of thumb:** `const` registry / literal you will read from → `satisfies`. `let`, function parameters, class fields, or anything that holds a *family* of values over time → annotation.

### Mistake 2 — Reaching for `as` when you meant `satisfies`

```ts
interface WebhookConfig {
  url:     string;
  secret:  string;
  events:  string[];
  timeout: number;
}

// ❌ Wrong — `as` skips the check entirely. Two bugs shipped:
const slackWebhook = {
  url:    "https://hooks.slack.com/services/T000/B000/XXX",
  secret: "shhh",
  events: ["deploy.started", "deploy.finished"],
  // `timeout` forgotten
} as WebhookConfig;

slackWebhook.timeout.toFixed(0);   // compiles. 💥 "Cannot read properties of undefined"

// ❌ Also wrong — `as` on a typo'd key silently accepts it:
const pagerWebhook = {
  url: "https://events.pagerduty.com/v2/enqueue",
  secrets: "abc",     // note the S — `as` does not care
  events: ["incident.trigger"],
  timeout: 3_000,
} as unknown as WebhookConfig;    // the double-cast people write to "make it work"

// ✅ Right — `satisfies` catches both at compile time:
const slackWebhookFixed = {
  url:     "https://hooks.slack.com/services/T000/B000/XXX",
  secret:  "shhh",
  events:  ["deploy.started", "deploy.finished"],
  timeout: 3_000,
} satisfies WebhookConfig;
// ❌ if `timeout` is missing:  "Property 'timeout' is missing"
// ❌ if `secrets` is written:  "Object literal may only specify known properties"
```

If you find yourself writing `as SomeInterface` on an object literal you just typed out, the answer is essentially always `satisfies` instead. `as` on a literal is a code smell — you are asserting something you could have proved.

### Mistake 3 — Forgetting `as const`, then wondering where the literals went

```ts
type Region = "us-east-1" | "us-west-2" | "eu-west-1";

interface BucketConfig { region: Region; bucket: string; versioned: boolean }

// ❌ Wrong — no `as const`. Literals widen where the target is a wide primitive:
const uploads = {
  region:    "us-east-1",
  bucket:    "example-uploads",
  versioned: true,
} satisfies BucketConfig;

uploads.region;     // "us-east-1"  ← survived! (target type is a literal union)
uploads.bucket;     // string       ← widened  (target type is `string`)
uploads.versioned;  // boolean      ← widened  (target type is `boolean`)

// So this fails:
type BucketName = typeof uploads.bucket;   // string — useless for deriving anything
declare function assertBucket(b: "example-uploads"): void;
// assertBucket(uploads.bucket);            // ❌ string not assignable to "example-uploads"

// ✅ Right — `as const` first, then `satisfies`:
const uploadsExact = {
  region:    "us-east-1",
  bucket:    "example-uploads",
  versioned: true,
} as const satisfies BucketConfig;

uploadsExact.region;     // "us-east-1"
uploadsExact.bucket;     // "example-uploads"
uploadsExact.versioned;  // true
assertBucket(uploadsExact.bucket);   // ✅
```

The trap: it *sometimes* works without `as const` (whenever the target property type is a literal union), so people learn the wrong lesson from one working example and are baffled the next time.

### Mistake 4 — Using `Record<string, V>` when you meant `Record<Union, V>`

```ts
type NotificationChannel = "email" | "sms" | "push" | "webhook";

interface ChannelConfig { enabled: boolean; rateLimitPerHour: number }

// ❌ Wrong — `string` keys means missing channels are never detected:
const CHANNELS = {
  email: { enabled: true,  rateLimitPerHour: 100 },
  sms:   { enabled: true,  rateLimitPerHour: 10 },
  // push and webhook missing — compiles happily
} as const satisfies Record<string, ChannelConfig>;

function send(channel: NotificationChannel): void {
  // @ts-expect-error — CHANNELS has no 'push' key, but the code needs one
  const cfg = CHANNELS[channel];
  // With Record<string, …> as the target and no `keyof typeof` discipline, this
  // is the exact place the bug surfaces — and only when you happen to index it.
}

// ✅ Right — the union as the key type makes completeness a compile error:
const CHANNELS_COMPLETE = {
  email:   { enabled: true,  rateLimitPerHour: 100 },
  sms:     { enabled: true,  rateLimitPerHour: 10 },
  push:    { enabled: true,  rateLimitPerHour: 500 },
  webhook: { enabled: false, rateLimitPerHour: 1_000 },
} as const satisfies Record<NotificationChannel, ChannelConfig>;

function sendFixed(channel: NotificationChannel): void {
  const cfg = CHANNELS_COMPLETE[channel];   // ✅ always defined
  if (!cfg.enabled) return;
}
```

`Record<string, V>` validates *values*. `Record<Union, V>` validates values **and** completeness. If a finite set of keys exists, name it.

### Mistake 5 — Applying `satisfies` to a variable instead of the literal

```ts
interface CacheEntry { ttlSeconds: number; staleWhileRevalidate: boolean }

// ❌ Wrong — the typo survives, because EPC only fires on fresh literals:
const rawCaches = {
  userProfile: { ttlSeconds: 300, staleWhileRevalidate: true },
  featureFlag: { ttlSeconds: 60,  staleWhileRevalidat: false },   // typo!
};
const CACHES = rawCaches satisfies Record<string, CacheEntry>;
// ❌ Actually this DOES error here — the value is missing a required property.
// But make the property optional and the typo becomes invisible:

interface CacheEntryOptional { ttlSeconds: number; staleWhileRevalidate?: boolean }
const rawCaches2 = {
  userProfile: { ttlSeconds: 300, staleWhileRevalidat: true },    // typo!
};
const CACHES2 = rawCaches2 satisfies Record<string, CacheEntryOptional>;
// ✅ COMPILES. The typo'd key is just an extra property on a non-fresh object.

// ✅ Right — attach `satisfies` directly to the literal, so EPC fires:
const CACHES3 = {
  userProfile: { ttlSeconds: 300, staleWhileRevalidat: true },
} satisfies Record<string, CacheEntryOptional>;
// ❌ Object literal may only specify known properties, and
//    'staleWhileRevalidat' does not exist in type 'CacheEntryOptional'.
```

### Mistake 6 — Expecting `satisfies` to validate runtime data

```ts
// ❌ Wrong — this is a lie dressed up as a check:
const rawConfig = JSON.parse(await readFile("config.json", "utf8")) satisfies ServiceConfig;
// `JSON.parse` returns `any`, and `any satisfies T` always passes. Zero
// validation happened. The file could be `{}` or `null` or an array.

// ❌ Also wrong — env vars are strings, and `satisfies` cannot check their contents:
const env = process.env satisfies { DATABASE_URL: string };
// ❌ actually errors (ProcessEnv values are `string | undefined`), but people
//    then "fix" it with `as` and lose everything.

// ✅ Right — `satisfies` for literals you wrote; a real validator for I/O:
function parseServiceConfig(raw: unknown): ServiceConfig {
  if (typeof raw !== "object" || raw === null) throw new Error("config must be an object");
  const obj = raw as Record<string, unknown>;

  const port = obj.port;
  if (typeof port !== "number" || !Number.isInteger(port) || port <= 0) {
    throw new Error("config.port must be a positive integer");
  }
  const databaseUrl = obj.databaseUrl;
  if (typeof databaseUrl !== "string" || databaseUrl.length === 0) {
    throw new Error("config.databaseUrl must be a non-empty string");
  }
  return { port, databaseUrl };
}

interface ServiceConfig { port: number; databaseUrl: string }
declare function readFile(path: string, enc: string): Promise<string>;
```

The rule: **`satisfies` is a check on your source code, never on your data.**

### Mistake 7 — Using `satisfies` on function *declarations*

```ts
// ❌ Not valid — `satisfies` is an expression operator:
// function handleRequest(req: Request): Response { ... } satisfies Handler;

// ❌ Also not what you want — this checks the RESULT of calling it:
// const x = handleRequest(req) satisfies Response;   // checks the return VALUE

// ✅ Right — a function expression, parenthesised:
type WebhookVerifier = (rawBody: string, signature: string) => boolean;

const verifyStripeSignature = ((rawBody, signature) => {
  //                            ^? string, ^? string — contextually typed
  return signature.length > 0 && rawBody.length > 0;
}) satisfies WebhookVerifier;

// ✅ Or, for a declaration, a separate type-level assertion:
function verifyGithubSignature(rawBody: string, signature: string): boolean {
  return signature.startsWith("sha256=") && rawBody.length > 0;
}
const _verifierCheck = verifyGithubSignature satisfies WebhookVerifier;
// ↑ zero runtime cost (it is erased), and it fails the build if the signature drifts.
```

---

## Practice exercises

### Exercise 1 — easy

Build a typed HTTP status registry for a small API service.

Define:

- `type HttpStatusName = "ok" | "created" | "noContent" | "badRequest" | "unauthorized" | "notFound" | "conflict" | "tooManyRequests" | "serverError"`
- `interface StatusDefinition { code: number; retryable: boolean; category: "success" | "client" | "server" }`

Then:

1. Write a `HTTP_STATUS` object using `as const satisfies Record<HttpStatusName, StatusDefinition>` with a correct entry for every name.
2. Derive `type StatusCode = ...` — the union of the actual numeric codes (`200 | 201 | 204 | 400 | ...`). Prove it works by declaring `const check: StatusCode = 429;` and confirming `const bad: StatusCode = 418;` errors.
3. Derive `type RetryableStatus = ...` — the union of only the status *names* whose `retryable` is `true`.
4. Write `function statusCodeFor(name: HttpStatusName): number` that indexes the registry with no `!` and no `??`.
5. Add a second object `HTTP_STATUS_WIDE: Record<HttpStatusName, StatusDefinition>` holding the same data via a plain annotation, and write three comment lines explaining exactly which of the derivations in steps 2 and 3 stop working with it, and why.

Constraint: no `as` casts anywhere except the single `as const`.

```ts
// Write your code here
```

### Exercise 2 — medium

Build a job-queue registry and dispatcher for a Node backend, where the payload type of each job is derived from the registry itself.

Requirements:

- `interface JobDefinition { queue: string; maxAttempts: number; backoff: "fixed" | "exponential"; timeoutMs: number; priority: 1 | 2 | 3 }`
- A `JOBS` registry with at least six jobs across three queues: `"welcomeEmail"`, `"passwordReset"`, `"generateInvoice"`, `"chargeSubscription"`, `"reindexSearch"`, `"exportUserData"` — validated with `as const satisfies Record<string, JobDefinition>`.
- Derive `type JobName`, `type QueueName` (the union of distinct `queue` values), and `type HighPriorityJob` (jobs whose `priority` is `1`).
- A **separate** `JOB_PAYLOADS` type map: `interface JobPayloads { welcomeEmail: { userId: number; locale: string }; ... }` — one entry per job. Enforce that `JobPayloads` covers exactly `JobName` using a type-level assertion (not `satisfies`, since it is a type not a value). Adding a job to `JOBS` without a payload entry must be a compile error.
- A `HANDLERS` object typed with `satisfies { [K in JobName]: (payload: JobPayloads[K]) => Promise<void> }` — note this is a mapped type, not a `Record`, so each handler gets its own payload type contextually with no annotations at the handler.
- `function enqueue<N extends JobName>(name: N, payload: JobPayloads[N]): void` that reads `JOBS[name]` for retry settings and logs them. `enqueue("welcomeEmail", { userId: 1 })` must error (missing `locale`); `enqueue("welcomeEmial", ...)` must error (unknown job).
- `function jobsInQueue<Q extends QueueName>(queue: Q): JobName[]` implemented with a typed `Object.entries` helper you write yourself, with the cast isolated to that one helper.

Finally, write five commented "regression scenarios" describing what breaks (and where) if someone: adds a job without a handler, adds a job without a payload type, renames a queue, changes a priority from 1 to 2, or removes `timeoutMs` from one entry.

```ts
// Write your code here
```

### Exercise 3 — hard

Build a fully type-derived API surface for a multi-tenant SaaS backend, driven from a single `satisfies`-validated specification object.

**Part A — the spec.** Define `interface EndpointSpec { method: "GET" | "POST" | "PATCH" | "DELETE"; path: string; scopes: readonly Scope[]; requestShape: "none" | "body" | "query"; successStatus: 200 | 201 | 204; idempotent: boolean }` with `type Scope = "org:read" | "org:write" | "member:read" | "member:write" | "billing:read" | "billing:write"`. Write an `API_SPEC` object of at least eight endpoints using `as const satisfies Record<string, EndpointSpec>`.

**Part B — derivations.** From `API_SPEC` alone, derive:

- `EndpointName`
- `MutatingEndpoint` (method is not `"GET"`)
- `PublicEndpoint` (empty `scopes` tuple)
- `ScopesRequiredBy<E extends EndpointName>` — the union of scopes for one endpoint
- `EndpointsRequiring<S extends Scope>` — the union of endpoint names that require a given scope (this one needs a distributed conditional over a mapped type)
- `PathOf<E extends EndpointName>` and a `type PathParams<P extends string>` that extracts `:param` segments from a path literal into an object type (`"/orgs/:orgId/members/:memberId"` → `{ orgId: string; memberId: string }`)

**Part C — the typed client.** Write `function callApi<E extends EndpointName>(name: E, params: PathParams<PathOf<E>>, ...rest): Promise<unknown>` where the `rest` parameter is required only when `requestShape` is `"body"` or `"query"`, and forbidden when it is `"none"` — use a conditional tuple type for the rest parameter. Demonstrate three call sites: one valid, one that fails for a missing path param, one that fails for passing a body to a `"none"` endpoint.

**Part D — the server side.** Write a `HANDLERS` object validated against a mapped type over `EndpointName`, where each handler receives `{ params: PathParams<PathOf<E>>; scopes: readonly Scope[] }` and must return a value whose HTTP status matches `successStatus` (model this with a discriminated return type: `{ status: 201; body: unknown }` etc.). A handler returning status `200` for an endpoint declared `201` must be a compile error.

**Part E — the audit.** Write three zero-runtime type assertions that fail the build if:
1. any endpoint with `idempotent: true` uses method `"POST"`,
2. any endpoint requiring `"billing:write"` is also a `PublicEndpoint`,
3. the set of `HANDLERS` keys is not exactly `EndpointName` (neither missing nor extra).

**Part F — write-up.** In comments, explain for each of Parts A–E which piece would break if `API_SPEC` used `: Record<string, EndpointSpec>` instead of `as const satisfies`, and rank the six derivations in Part B by how badly they degrade.

```ts
// Write your code here
```

---

## Quick reference cheat sheet

```ts
// ── The three tools ─────────────────────────────────────────────────────────
const a: T = expr;          // check + REPLACE type with T (widens)
const b = expr as T;        // NO real check + REPLACE type with T (unsound)
const c = expr satisfies T; // check + KEEP inferred type of expr

// ── The idiom you will write 95% of the time ────────────────────────────────
const REGISTRY = {
  keyOne: { /* ... */ },
  keyTwo: { /* ... */ },
} as const satisfies Record<SomeUnion, SomeShape>;
//  ^^^^^^^^ precision      ^^^^^^^^^ validation + completeness

// ── What each part buys you ─────────────────────────────────────────────────
// as const                    → literal types, deep readonly, exact tuples
// satisfies Record<string, V> → value shapes checked; exact keys at use sites
// satisfies Record<Union, V>  → the above PLUS "every union member is present"

// ── Deriving types from the value ──────────────────────────────────────────
type Name  = keyof typeof REGISTRY;                    // union of keys
type Value = (typeof REGISTRY)[Name];                  // union of values
type Field = (typeof REGISTRY)[Name]["someField"];     // union of one field
type Elems = (typeof REGISTRY)[Name]["someArray"][number];  // union of array items

// Filter keys by a literal value:
type Selected = { [K in Name]: (typeof REGISTRY)[K]["flag"] extends true ? K : never }[Name];

// ── Functions ───────────────────────────────────────────────────────────────
const handler = ((userId) => userId * 2) satisfies (userId: number) => number;
//                ^? contextually typed, no annotation needed
const _check = existingFunction satisfies SomeSignature;   // erased assertion

// ── Type-level equivalent (no value available) ─────────────────────────────
type Expect<T extends true> = T;
type Extends<A, B> = A extends B ? true : false;
type _Ok = Expect<Extends<typeof REGISTRY, Record<SomeUnion, SomeShape>>>;

// ── Gotchas ─────────────────────────────────────────────────────────────────
expr satisfies T;                 // does NOT narrow; does NOT add readonly
{ a: 1 } as const satisfies { a: number };   // ✅ order: as const FIRST
// { a: 1 } satisfies { a: number } as const; // ❌ syntax error
const v = { x: 1, y: 2 };
v satisfies { x: number };        // ✅ no excess-property error (not fresh)
{ x: 1, y: 2 } satisfies { x: number };  // ❌ excess property 'y' (fresh literal)
anyValue satisfies AnyType;       // ✅ always passes — `any` defeats it
// as const + readonly array vs mutable array target → assignability error
```

| Situation | Use |
|---|---|
| `const` lookup table / registry you read from | `as const satisfies Record<K, V>` |
| Finite key set that must be complete | `satisfies Record<Union, V>` |
| Open key set, values must conform | `satisfies Record<string, V>` |
| Variable that will be reassigned (`let`) | `: T` annotation |
| Function parameter / class field / return type | `: T` annotation |
| Passing a literal straight into a call | nothing — contextual typing already checks it |
| Verifying an existing function matches a signature | `const _c = fn satisfies Sig;` |
| Data from `JSON.parse` / `process.env` / network | a **runtime validator**, never `satisfies` |
| You need the compiler to stop complaining | fix the type; `as` is a last resort |
| Type-level check with no value | `type Expect<T extends true> = T` |

| Behaviour | `: T` | `as T` | `satisfies T` |
|---|---|---|---|
| Rejects missing required properties | ✅ | ❌ | ✅ |
| Rejects excess properties (fresh literal) | ✅ | ❌ | ✅ |
| Rejects wrong property types | ✅ | ⚠️ partial | ✅ |
| Resulting type | `T` | `T` | inferred |
| Keeps literal values | ❌ | ❌ | ✅ (with `as const`) |
| Optional props stay optional in result | ✅ | ✅ | ❌ (become required if set) |
| `keyof typeof x` is useful | only if `T` is exact | only if `T` is exact | ✅ always |
| Emits runtime code | no | no | no |
| Minimum TS version | 1.0 | 1.0 | **4.9** |

---

## Connected topics

- **11 — Literal types** — where `201`, `"us-east-1"` and `true` as *types* come from; `satisfies` exists to stop you losing them.
- **12 — const assertions** — `as const` in full: deep readonly, tuple inference, and why it pairs with `satisfies` rather than competing with it.
- **13 — Type assertions** — `as`, `as unknown as`, and the exact sense in which assertions are unsound; the contrast that makes `satisfies` click.
- **09 — Union types** — `Record<Union, V>` completeness checking is just union exhaustiveness applied to keys.
- **26 — keyof and typeof operators** — `keyof typeof REGISTRY` is the mechanism that turns a `satisfies`-validated object into a type source.
- **28 — Indexed access types** — `(typeof REGISTRY)[Name]["status"]` and the `[number]` array-element trick.
- **30 — Mapped types** — `{ [K in JobName]: (p: JobPayloads[K]) => Promise<void> }` as a `satisfies` target for per-key contextual typing.
- **31 — Conditional types** — filtering keys by a literal value (`extends true ? K : never`) to derive sub-unions from a registry.
- **40 — Type narrowing** — why `satisfies` is *not* narrowing, and what to use when you actually need it.
- **62 — Strict null checks** — `satisfies Record<Union, V>` removes the `| undefined` worry from lookups by proving the key set is exact.
