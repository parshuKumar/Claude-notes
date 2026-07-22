# 69 — Enum vs Union Types

## What is this?

TypeScript gives you two very different tools for expressing "this value is one of a fixed set of options":

```ts
// Option A — an enum (a TypeScript language construct that emits real JavaScript):
enum OrderStatus {
  Pending   = "pending",
  Paid      = "paid",
  Shipped   = "shipped",
  Cancelled = "cancelled",
}

// Option B — a union of string literal types (pure type-level, zero runtime output):
type OrderStatus = "pending" | "paid" | "shipped" | "cancelled";
```

Both let you write `function updateOrder(status: OrderStatus)` and get autocomplete plus a compile error on typos. But they behave completely differently: **an enum is a value that exists at runtime**, while **a union of literals is erased entirely at compile time**.

An `enum` declaration creates *two* things — a type named `OrderStatus` and a runtime object named `OrderStatus`. A union type creates *one* thing — a type. That single difference cascades into every trade-off in this document: bundle size, nominal typing surprises, reverse mappings, `const enum` footguns, and whether your code can run under Node's native type-stripping.

This doc explains numeric enums, string enums, the JavaScript they compile to, why `const enum` is discouraged, the nominal-typing behaviour that surprises everyone, and the modern `as const` object + `typeof` union pattern that most TypeScript codebases now prefer.

---

## Why does it matter?

Backend code is saturated with fixed option sets:

- `OrderStatus`, `PaymentState`, `SubscriptionTier`, `UserRole`
- HTTP methods, log levels, queue priorities, feature flag names
- Postgres `enum` columns, Prisma `enum` blocks, protobuf enums
- Event names on a message bus (`user.created`, `order.placed`)

Whichever you pick propagates through your entire domain model — DTOs, database rows, API contracts, Zod schemas, event payloads. Changing from `enum` to a union six months later means touching hundreds of call sites, because `OrderStatus.Paid` and `"paid"` are not interchangeable in every context.

There is also a hard practical constraint that has emerged recently: **enums are not erasable syntax**. Node.js's built-in TypeScript type-stripping (Node 22.6+ with `--experimental-strip-types`, on by default from Node 23.6) refuses to run enums, because stripping an enum would delete a real runtime object. TypeScript 5.8 added `erasableSyntaxOnly` to flag exactly this. If you want your `.ts` files to run directly under Node with no build step, enums are off the table.

So this is not a stylistic debate. It's a decision that determines whether your files can run unbuilt, how big your bundles are, whether tree-shaking works, and whether your `status` field can accidentally accept a value from an unrelated enum.

---

## The JavaScript way vs the TypeScript way

```js
// ── JavaScript — you hand-roll constants and hope ─────────────────────────────

// Approach 1: bare string literals scattered through the codebase.
function updateOrderStatus(orderId, status) {
  // No validation. No autocomplete. Typos are silent runtime bugs.
  return db.orders.update(orderId, { status });
}

updateOrderStatus(42, "shiped");      // typo — writes garbage to the DB, no error
updateOrderStatus(42, "SHIPPED");     // wrong case — silently wrong
updateOrderStatus(42, "cancelled ");  // trailing space — silently wrong
updateOrderStatus(42, 7);             // wrong type entirely — still no error

// Approach 2: a frozen constants object — better, but still no type safety.
const ORDER_STATUS = Object.freeze({
  PENDING:   "pending",
  PAID:      "paid",
  SHIPPED:   "shipped",
  CANCELLED: "cancelled",
});

updateOrderStatus(42, ORDER_STATUS.SHIPED);
// → undefined. No error at all. `undefined` gets written to the DB.
// JavaScript happily reads a missing property off an object and hands you undefined.

// Approach 3: runtime validation — works, but only after the code already ran.
const VALID = ["pending", "paid", "shipped", "cancelled"];
function updateOrderStatusSafe(orderId, status) {
  if (!VALID.includes(status)) {
    throw new Error(`Invalid status: ${status}`); // caught in prod, at 3am
  }
  return db.orders.update(orderId, { status });
}

// You now maintain the list in three places: the array, the constants object,
// and every JSDoc comment describing the parameter. They drift. They always drift.
```

```ts
// ── TypeScript — the set of legal values IS the type ──────────────────────────

type OrderStatus = "pending" | "paid" | "shipped" | "cancelled";

function updateOrderStatus(orderId: number, status: OrderStatus): Promise<void> {
  return db.orders.update(orderId, { status });
}

updateOrderStatus(42, "shipped");   // ✅ compiles — and your editor autocompleted it
updateOrderStatus(42, "shiped");    // ❌ Type '"shiped"' is not assignable to type 'OrderStatus'
updateOrderStatus(42, "SHIPPED");   // ❌ same
updateOrderStatus(42, 7);           // ❌ Type 'number' is not assignable to type 'OrderStatus'

// And when you add a new status, every exhaustive switch in the codebase
// becomes a compile error until you handle it:
type OrderStatus2 = "pending" | "paid" | "shipped" | "cancelled" | "refunded";

function label(status: OrderStatus2): string {
  switch (status) {
    case "pending":   return "Awaiting payment";
    case "paid":      return "Payment received";
    case "shipped":   return "In transit";
    case "cancelled": return "Cancelled";
    // ❌ Error: Type '"refunded"' is not assignable to type 'never'
    default: {
      const _exhaustive: never = status;
      throw new Error(`Unhandled status: ${_exhaustive}`);
    }
  }
}
```

The revelation: in JavaScript, the *list of valid values* lives in your head, in a comment, or in a runtime array that nobody checks. In TypeScript, the list of valid values **is the type**, checked at every call site, on every property assignment, in every object literal, for free, before the code ever runs.

The remaining question — and the actual subject of this document — is whether that type should be an `enum` or a union of literals.

---

## Syntax

```ts
// ── Numeric enum — values auto-increment from 0 ───────────────────────────────
enum LogLevel {
  Debug,   // 0
  Info,    // 1
  Warn,    // 2
  Error,   // 3
}
const lvl: LogLevel = LogLevel.Warn;      // the type and the value share a name
const n: number = LogLevel.Warn;          // 2 — numeric enum members ARE numbers
const name = LogLevel[2];                 // "Warn" — reverse mapping (numeric only)

// ── Numeric enum with explicit start / mixed values ───────────────────────────
enum HttpStatus {
  Ok        = 200,
  Created   = 201,          // explicit
  NoContent = 204,
  BadRequest = 400,
  NotFound   = 404,
}

// ── String enum — every member needs an explicit initializer ──────────────────
enum OrderStatus {
  Pending   = "pending",    // no auto-increment for strings — all values required
  Paid      = "paid",
  Shipped   = "shipped",
  Cancelled = "cancelled",
}
const s: string = OrderStatus.Paid;       // "paid"
// OrderStatus["paid"]                    // ❌ no reverse mapping for string enums

// ── const enum — inlined at call sites, emits nothing (see "Going deeper") ────
const enum Direction { Up = 1, Down = 2 }
const d = Direction.Up;                   // compiles to: const d = 1 /* Direction.Up */

// ── Union of string literals — pure type, zero runtime output ─────────────────
type UserRole = "admin" | "editor" | "viewer";
const role: UserRole = "admin";           // ✅
// const bad: UserRole = "owner";         // ❌ not assignable

// ── as const object + typeof union — the modern alternative ───────────────────
const USER_ROLE = {
  Admin:  "admin",
  Editor: "editor",
  Viewer: "viewer",
} as const;                               // `as const` freezes literal types

type UserRoleValue = typeof USER_ROLE[keyof typeof USER_ROLE];
// = "admin" | "editor" | "viewer"

const r1: UserRoleValue = USER_ROLE.Admin;  // ✅ named constant, like an enum
const r2: UserRoleValue = "admin";          // ✅ raw literal, unlike an enum
```

---

## How it works — concept by concept

### Concept 1 — What an enum actually compiles to

This is the single most clarifying thing you can learn about enums. An `enum` is not erased. It emits a real JavaScript object built by an IIFE.

```ts
// ── Source TypeScript ─────────────────────────────────────────────────────────
enum LogLevel {
  Debug,
  Info,
  Warn,
  Error,
}
```

```js
// ── Emitted JavaScript (target ES2020, TypeScript 5.x) ────────────────────────
var LogLevel;
(function (LogLevel) {
  LogLevel[LogLevel["Debug"] = 0] = "Debug";
  LogLevel[LogLevel["Info"]  = 1] = "Info";
  LogLevel[LogLevel["Warn"]  = 2] = "Warn";
  LogLevel[LogLevel["Error"] = 3] = "Error";
})(LogLevel || (LogLevel = {}));

// Unpacking `LogLevel[LogLevel["Debug"] = 0] = "Debug";`:
//   1. LogLevel["Debug"] = 0        → assigns forward mapping, evaluates to 0
//   2. LogLevel[0] = "Debug"        → assigns reverse mapping
// Result object:
//   { "0": "Debug", "1": "Info", "2": "Warn", "3": "Error",
//     Debug: 0, Info: 1, Warn: 2, Error: 3 }
```

That two-way object is why `LogLevel[2] === "Warn"` works. It's also why a numeric enum costs you **eight properties for four members** at runtime.

Now a string enum:

```ts
// ── Source ────────────────────────────────────────────────────────────────────
enum OrderStatus {
  Pending = "pending",
  Paid    = "paid",
}
```

```js
// ── Emitted JavaScript ────────────────────────────────────────────────────────
var OrderStatus;
(function (OrderStatus) {
  OrderStatus["Pending"] = "pending";   // forward mapping only
  OrderStatus["Paid"]    = "paid";
})(OrderStatus || (OrderStatus = {}));

// Result object: { Pending: "pending", Paid: "paid" }
// NO reverse mapping — because "paid" as a key could collide with a member name.
```

And the union? It compiles to nothing:

```ts
// ── Source ────────────────────────────────────────────────────────────────────
type OrderStatus = "pending" | "paid";
const status: OrderStatus = "paid";
```

```js
// ── Emitted JavaScript ────────────────────────────────────────────────────────
const status = "paid";
// The type declaration vanished. Zero bytes shipped.
```

Practical takeaway: every enum in your codebase is a live object in your bundle, in every worker process, in every Lambda cold start. Usually negligible — but it's real, and it's why bundlers struggle to tree-shake enums (the IIFE has side effects on `LogLevel`, so naive tree-shakers keep it).

---

### Concept 2 — Numeric enums and reverse mapping

Reverse mapping is genuinely useful — and genuinely dangerous.

```ts
enum QueuePriority {
  Low      = 0,
  Normal   = 1,
  High     = 2,
  Critical = 3,
}

// Forward: name → number
const p: QueuePriority = QueuePriority.High;      // 2

// Reverse: number → name (numeric enums only)
const label = QueuePriority[p];                    // "High"
console.log(`Job queued at priority ${QueuePriority[p]}`);  // "priority High"

// This is the ONE thing enums do that unions can't do for free —
// mapping a stored integer back to a human-readable name.
// Very handy when your DB column is a smallint.

// ── The danger: numeric enums accept ANY number ───────────────────────────────
function enqueue(jobId: string, priority: QueuePriority): void { /* ... */ }

enqueue("job-1", QueuePriority.High);   // ✅ obviously fine
enqueue("job-1", 2);                    // ✅ also fine — 2 is assignable to QueuePriority
enqueue("job-1", 99);                   // ❌ error in TS 5.0+ ... but read on

// Before TypeScript 5.0, `enqueue("job-1", 99)` COMPILED. Any number was
// assignable to a numeric enum type, because numeric enums support bit flags:
enum Permission {
  None   = 0,
  Read   = 1 << 0,   // 1
  Write  = 1 << 1,   // 2
  Delete = 1 << 2,   // 4
  Admin  = Read | Write | Delete,  // 7 — a computed member
}

const perms: Permission = Permission.Read | Permission.Write;  // 3
// 3 is not a declared member, yet it must be assignable — hence the historical hole.
// TS 5.0 tightened this so out-of-range LITERALS error, but the type is still
// fundamentally "a number", and a `number` variable widens right through it:

const fromDb: number = 99;
const sketchy: QueuePriority = fromDb as QueuePriority;  // no runtime check whatsoever
console.log(QueuePriority[sketchy]);                     // undefined — silent hole
```

```ts
// ── The union equivalent — no holes, but no reverse map either ────────────────
type QueuePriorityU = 0 | 1 | 2 | 3;

function enqueueU(jobId: string, priority: QueuePriorityU): void { /* ... */ }
enqueueU("job-1", 2);    // ✅
enqueueU("job-1", 99);   // ❌ Type '99' is not assignable to type '0 | 1 | 2 | 3'

// Need a label? Build the map explicitly — it's honest and tree-shakeable:
const PRIORITY_LABEL: Record<QueuePriorityU, string> = {
  0: "Low",
  1: "Normal",
  2: "High",
  3: "Critical",
};
console.log(PRIORITY_LABEL[2]);  // "High"
// Bonus: if you add priority 4 to the union, this object becomes a compile error.
```

---

### Concept 3 — String enums and nominal typing

TypeScript's type system is **structural** everywhere — except string enums, which are **nominal**. This trips up every developer exactly once, memorably.

```ts
enum OrderStatus {
  Pending = "pending",
  Paid    = "paid",
}

function setStatus(status: OrderStatus): void { /* ... */ }

setStatus(OrderStatus.Paid);   // ✅
setStatus("paid");             // ❌ Argument of type '"paid"' is not assignable
                               //    to parameter of type 'OrderStatus'.
// The VALUE is identical at runtime. The compiler rejects it anyway, because
// string enum members are nominally typed — identity, not shape, decides.
```

This has real consequences the moment data crosses a boundary:

```ts
// ── The boundary problem ──────────────────────────────────────────────────────

// JSON.parse gives you `any`/`unknown`; a DB driver gives you `string`.
// Neither can produce an OrderStatus without a cast.

interface OrderRow {
  id:     number;
  status: string;      // whatever pg / mysql2 handed back
}

function fromRow(row: OrderRow): { id: number; status: OrderStatus } {
  return {
    id:     row.id,
    // ❌ Type 'string' is not assignable to type 'OrderStatus'
    // status: row.status,

    // You are forced into a cast — which does ZERO runtime validation:
    status: row.status as OrderStatus,
  };
}

// If the DB contains "shipped" (a status you deleted from the enum last sprint),
// the cast happily produces a value that no switch case handles.
// The enum bought you nothing at the boundary — it only cost you friction inside.
```

```ts
// ── Union version — the boundary is a real, checkable narrowing ───────────────
type OrderStatusU = "pending" | "paid";

const ORDER_STATUSES = ["pending", "paid"] as const;

function isOrderStatus(value: unknown): value is OrderStatusU {
  return typeof value === "string"
      && (ORDER_STATUSES as readonly string[]).includes(value);
}

function fromRowU(row: OrderRow): { id: number; status: OrderStatusU } {
  if (!isOrderStatus(row.status)) {
    throw new Error(`Unknown order status in DB: ${row.status}`);
  }
  return { id: row.id, status: row.status };  // ✅ narrowed, genuinely validated
}
// The union pushed you toward a runtime check. The enum pushed you toward a cast.
```

Note the asymmetry: **numeric** enum members accept plain numbers (structural-ish), while **string** enum members reject plain strings (nominal). That inconsistency alone is a good reason to be wary.

---

### Concept 4 — `const enum` and why it is discouraged

`const enum` looks like the perfect answer: enum ergonomics, zero runtime cost.

```ts
// ── Source ────────────────────────────────────────────────────────────────────
const enum CacheTtl {
  Short  = 60,
  Medium = 300,
  Long   = 3600,
}

const ttl = CacheTtl.Medium;
await redis.set("user:42", payload, "EX", CacheTtl.Long);
```

```js
// ── Emitted JavaScript — the enum object is GONE, values inlined ──────────────
const ttl = 300 /* CacheTtl.Medium */;
await redis.set("user:42", payload, "EX", 3600 /* CacheTtl.Long */);
```

Beautiful. So why is it discouraged?

**Reason 1 — it breaks under `isolatedModules` / any single-file transpiler.**

Inlining requires the compiler to look *across files*: to inline `CacheTtl.Long`, the transpiler must have read the file that declares `CacheTtl`. Babel, esbuild, swc, Vite, and `ts-loader` in transpile-only mode all compile **one file at a time** with no cross-file type information. They physically cannot inline it.

```ts
// constants.ts
export const enum CacheTtl { Long = 3600 }

// service.ts — transpiled in isolation by esbuild/babel
import { CacheTtl } from "./constants";
await redis.set(key, val, "EX", CacheTtl.Long);
// The transpiler emits a property access on an object that was never emitted.
// → Runtime: TypeError: Cannot read properties of undefined (reading 'Long')
```

With `"isolatedModules": true` in tsconfig, `tsc` refuses to let you export a `const enum` at all:

```
error TS1205: Re-exporting a type when 'isolatedModules' is enabled requires using 'export type'.
error TS2748: Cannot access ambient const enums when 'isolatedModules' is enabled.
```

Since TypeScript 5.0, `isolatedModules` is implied by `verbatimModuleSyntax`, and virtually every modern toolchain (Vite, Next.js, tsx, Bun, Deno) turns it on. `const enum` is effectively incompatible with the modern build ecosystem.

**Reason 2 — it is a hazard in published libraries.**

If your `.d.ts` exposes a `const enum`, consumers inline your *current* values into their build. Ship a patch release that renumbers a member and their already-built bundles are silently wrong — no version check catches it. The TypeScript team's own guidance is to avoid `const enum` in library `.d.ts` files entirely, and TypeScript itself stopped using them internally.

**Reason 3 — `preserveConstEnums` makes it pointless.**

```jsonc
// tsconfig.json
{ "compilerOptions": { "preserveConstEnums": true } }
// Now `const enum` emits the full IIFE object anyway — you get the runtime cost
// AND the isolatedModules incompatibility. Worst of both.
```

The modern replacement is embarrassingly simple:

```ts
// ── Just use plain const values ───────────────────────────────────────────────
export const CACHE_TTL = {
  Short:  60,
  Medium: 300,
  Long:   3600,
} as const;

export type CacheTtlValue = typeof CACHE_TTL[keyof typeof CACHE_TTL];  // 60 | 300 | 3600

// Works with every transpiler, tree-shakes, no cross-file magic, no cast at boundaries.
```

---

### Concept 5 — `as const` object + `typeof` union: the modern pattern

This pattern gives you everything an enum gives you (named constants, a type, autocomplete, iteration) with none of the costs.

```ts
// ── Step 1 — the object, frozen at the type level with `as const` ─────────────
export const USER_ROLE = {
  Admin:     "admin",
  Editor:    "editor",
  Viewer:    "viewer",
  Suspended: "suspended",
} as const;
// Without `as const`, each property would widen to `string`.
// With it, the type is:
//   { readonly Admin: "admin"; readonly Editor: "editor"; ... }

// ── Step 2 — derive the union from the object's values ────────────────────────
export type UserRole = typeof USER_ROLE[keyof typeof USER_ROLE];
// Decomposed:
//   typeof USER_ROLE              → { readonly Admin: "admin"; ... }
//   keyof typeof USER_ROLE        → "Admin" | "Editor" | "Viewer" | "Suspended"
//   typeof USER_ROLE[<those>]     → "admin" | "editor" | "viewer" | "suspended"

// ── Step 3 — you now have both a value namespace AND a type ───────────────────
function assignRole(userId: number, role: UserRole): void { /* ... */ }

assignRole(1, USER_ROLE.Admin);   // ✅ named constant — enum ergonomics
assignRole(1, "admin");           // ✅ raw literal ALSO works — better than enum
assignRole(1, "owner");           // ❌ Type '"owner"' is not assignable to 'UserRole'

// ── Step 4 — iteration and runtime validation come free ───────────────────────
const ALL_ROLES = Object.values(USER_ROLE);          // readonly ["admin", ...]
                                                     // typed as UserRole[]

function isUserRole(value: unknown): value is UserRole {
  return typeof value === "string"
      && (Object.values(USER_ROLE) as string[]).includes(value);
}

// ── Step 5 — a reverse map, if you want one, is explicit ─────────────────────
const ROLE_LABEL: Record<UserRole, string> = {
  admin:     "Administrator",
  editor:    "Content Editor",
  viewer:    "Read-only Viewer",
  suspended: "Suspended Account",
};
// If you add a role to USER_ROLE and forget it here → compile error.
// An enum's reverse mapping can never give you that guarantee.
```

Compare to the enum version of the exact same thing:

```ts
enum UserRoleEnum {
  Admin     = "admin",
  Editor    = "editor",
  Viewer    = "viewer",
  Suspended = "suspended",
}

// Iteration: works, but returns string[] — you lose the narrow type.
const all: string[] = Object.values(UserRoleEnum);

// Raw literals: rejected.
// assignRoleE(1, "admin");   ❌

// Boundary: requires an unchecked cast.
// const role = req.body.role as UserRoleEnum;

// Runtime cost: an IIFE + object per enum.
// Erasable: no.
```

---

### Concept 6 — `erasableSyntaxOnly` and Node type-stripping

TypeScript's original design included several constructs that emit runtime code — "non-erasable syntax":

- `enum` (emits an IIFE + object)
- `namespace` / `module` with a body (emits an IIFE)
- parameter properties (`constructor(private readonly userId: number)`)
- `import x = require(...)` / `export =`

Everything else in TypeScript is **erasable**: delete the type annotations and you have valid JavaScript. That property is what makes "type stripping" possible — no compilation, just deletion.

```ts
// ── Erasable — Node can run this file directly ────────────────────────────────
type UserRole = "admin" | "editor" | "viewer";

interface UserRecord {
  userId: number;
  role:   UserRole;
}

function promote(user: UserRecord): UserRecord {
  return { ...user, role: "admin" };
}
// Strip the annotations → plain JS. Node just replaces types with whitespace.
```

```ts
// ── NON-erasable — Node refuses ───────────────────────────────────────────────
enum UserRoleEnum {
  Admin = "admin",
}
// $ node --experimental-strip-types server.ts
// SyntaxError: TypeScript enum is not supported in strip-only mode

class UserService {
  // Parameter property — also non-erasable:
  constructor(private readonly repo: UserRepository) {}
}
// SyntaxError: TypeScript parameter property is not supported in strip-only mode
```

TypeScript 5.8 added a compiler flag that surfaces this at build time instead of runtime:

```jsonc
// tsconfig.json
{
  "compilerOptions": {
    "erasableSyntaxOnly": true,   // error on any non-erasable construct
    "verbatimModuleSyntax": true, // usually paired with it
    "isolatedModules": true
  }
}
```

```ts
// With erasableSyntaxOnly: true —
enum OrderStatus { Pending = "pending" }
// ❌ error TS1294: This syntax is not allowed when 'erasableSyntaxOnly' is enabled.

class OrderService {
  constructor(private readonly db: Database) {}
  //          ^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  // ❌ error TS1294: This syntax is not allowed when 'erasableSyntaxOnly' is enabled.
}

// ✅ The erasable rewrite:
const ORDER_STATUS = { Pending: "pending" } as const;
type OrderStatus = typeof ORDER_STATUS[keyof typeof ORDER_STATUS];

class OrderServiceOk {
  private readonly db: Database;
  constructor(db: Database) {
    this.db = db;
  }
}
```

Why this matters practically: `node server.ts` with no build step is now viable for scripts, migrations, seed files, CLI tools, and increasingly whole services. Deno and Bun already work this way. Choosing unions over enums keeps that door open; choosing enums closes it permanently for that file and everything that imports it.

---

### Concept 7 — Exhaustiveness, narrowing, and switch behaviour

Both enums and unions support exhaustive switches — but the union version gives better errors and works with plain data.

```ts
type PaymentState = "initiated" | "authorized" | "captured" | "refunded" | "failed";

function describePayment(state: PaymentState): string {
  switch (state) {
    case "initiated":  return "Payment started";
    case "authorized": return "Funds held";
    case "captured":   return "Funds captured";
    case "refunded":   return "Refunded to customer";
    case "failed":     return "Payment failed";
    default: {
      // If a new member is added to PaymentState, `state` is no longer `never`
      // here and this line becomes a compile error — exactly what you want.
      const _exhaustive: never = state;
      throw new Error(`Unhandled payment state: ${_exhaustive}`);
    }
  }
}

// ── The enum version works identically, but the values are less portable ──────
enum PaymentStateEnum {
  Initiated  = "initiated",
  Authorized = "authorized",
  Captured   = "captured",
}

function describeEnum(state: PaymentStateEnum): string {
  switch (state) {
    case PaymentStateEnum.Initiated:  return "Payment started";
    case PaymentStateEnum.Authorized: return "Funds held";
    case PaymentStateEnum.Captured:   return "Funds captured";
    default: {
      const _exhaustive: never = state;
      throw new Error(`Unhandled: ${_exhaustive}`);
    }
  }
}
// Note: you must import PaymentStateEnum to write a case clause.
// With a union, `case "initiated"` needs no import at all — the literal is the value.
```

Unions also compose with discriminated unions, `Extract`, `Exclude`, and template literal types in ways enums cannot:

```ts
type PaymentState2 = "initiated" | "authorized" | "captured" | "refunded" | "failed";

// Slice the union:
type TerminalState = Extract<PaymentState2, "captured" | "refunded" | "failed">;
type ActiveState   = Exclude<PaymentState2, TerminalState>;  // "initiated" | "authorized"

// Build derived types from it:
type PaymentEventName = `payment.${PaymentState2}`;
// "payment.initiated" | "payment.authorized" | "payment.captured" | ...

// Map over it:
type PaymentHandlers = {
  [S in PaymentState2]: (paymentId: string) => Promise<void>;
};
// { initiated: (...) => ...; authorized: (...) => ...; ... }

// With an enum you can do the mapped-type part (`[S in PaymentStateEnum]`),
// but template literal expansion over enum members does not produce clean
// string literal keys, and Extract/Exclude on enum members reads far worse.
```

---

## Example 1 — basic

```ts
// ── A log level system, built three ways, so you can see the difference ───────

// ═════ Version A — numeric enum ══════════════════════════════════════════════

enum LogLevelEnum {
  Debug = 10,
  Info  = 20,
  Warn  = 30,
  Error = 40,
}

function shouldLogEnum(messageLevel: LogLevelEnum, minLevel: LogLevelEnum): boolean {
  return messageLevel >= minLevel;              // numeric comparison — this is the win
}

function formatEnum(level: LogLevelEnum, message: string): string {
  // Reverse mapping turns 30 back into "Warn":
  return `[${LogLevelEnum[level].toUpperCase()}] ${message}`;
}

console.log(formatEnum(LogLevelEnum.Warn, "disk usage at 91%"));
// "[WARN] disk usage at 91%"
console.log(shouldLogEnum(LogLevelEnum.Debug, LogLevelEnum.Info));  // false

// ═════ Version B — string union ══════════════════════════════════════════════

type LogLevel = "debug" | "info" | "warn" | "error";

// Ordering must be explicit — but it's typed, and adding a level breaks it loudly:
const LOG_LEVEL_WEIGHT: Record<LogLevel, number> = {
  debug: 10,
  info:  20,
  warn:  30,
  error: 40,
};

function shouldLog(messageLevel: LogLevel, minLevel: LogLevel): boolean {
  return LOG_LEVEL_WEIGHT[messageLevel] >= LOG_LEVEL_WEIGHT[minLevel];
}

function format(level: LogLevel, message: string): string {
  return `[${level.toUpperCase()}] ${message}`;   // no reverse map needed — it IS the name
}

console.log(format("warn", "disk usage at 91%"));       // "[WARN] disk usage at 91%"
console.log(shouldLog("debug", "info"));                // false

// ═════ Version C — as const object + typeof union (recommended) ══════════════

export const LOG_LEVEL = {
  Debug: "debug",
  Info:  "info",
  Warn:  "warn",
  Error: "error",
} as const;

export type LogLevelValue = typeof LOG_LEVEL[keyof typeof LOG_LEVEL];
// "debug" | "info" | "warn" | "error"

const WEIGHT: Record<LogLevelValue, number> = {
  debug: 10,
  info:  20,
  warn:  30,
  error: 40,
};

export function isLogLevel(value: unknown): value is LogLevelValue {
  return typeof value === "string"
      && (Object.values(LOG_LEVEL) as string[]).includes(value);
}

export class Logger {
  private readonly minWeight: number;

  constructor(minLevel: LogLevelValue = LOG_LEVEL.Info) {
    this.minWeight = WEIGHT[minLevel];
  }

  // Reading config from the environment — the boundary is properly validated:
  static fromEnv(): Logger {
    const raw = process.env.LOG_LEVEL;
    return new Logger(isLogLevel(raw) ? raw : LOG_LEVEL.Info);
  }

  private write(level: LogLevelValue, message: string, meta?: Record<string, unknown>): void {
    if (WEIGHT[level] < this.minWeight) return;
    const line = JSON.stringify({
      level,
      message,
      timestamp: new Date().toISOString(),
      ...meta,
    });
    process.stdout.write(line + "\n");
  }

  debug(message: string, meta?: Record<string, unknown>): void { this.write(LOG_LEVEL.Debug, message, meta); }
  info (message: string, meta?: Record<string, unknown>): void { this.write(LOG_LEVEL.Info,  message, meta); }
  warn (message: string, meta?: Record<string, unknown>): void { this.write(LOG_LEVEL.Warn,  message, meta); }
  error(message: string, meta?: Record<string, unknown>): void { this.write(LOG_LEVEL.Error, message, meta); }
}

// Usage — both styles accepted, environment input safely narrowed:
const logger = Logger.fromEnv();
logger.warn("disk usage at 91%", { host: "api-03", usagePct: 91 });
logger.debug("cache miss", { key: "user:42" });   // suppressed unless LOG_LEVEL=debug
```

---

## Example 2 — real world backend use case

```ts
// ── Order + subscription domain: enum-free, boundary-safe, fully exhaustive ───
// This is the shape most production TypeScript backends converge on.

// ═════ 1. Domain constants and derived unions ════════════════════════════════

export const ORDER_STATUS = {
  Draft:     "draft",
  Pending:   "pending_payment",
  Paid:      "paid",
  Shipped:   "shipped",
  Delivered: "delivered",
  Cancelled: "cancelled",
  Refunded:  "refunded",
} as const;

export type OrderStatus = typeof ORDER_STATUS[keyof typeof ORDER_STATUS];

export const PAYMENT_PROVIDER = {
  Stripe:  "stripe",
  Adyen:   "adyen",
  PayPal:  "paypal",
} as const;

export type PaymentProvider = typeof PAYMENT_PROVIDER[keyof typeof PAYMENT_PROVIDER];

export const USER_ROLE = {
  Admin:    "admin",
  Support:  "support",
  Customer: "customer",
} as const;

export type UserRole = typeof USER_ROLE[keyof typeof USER_ROLE];

// ═════ 2. Derived subsets — impossible to express this cleanly with enums ════

export type TerminalOrderStatus = Extract<
  OrderStatus,
  "delivered" | "cancelled" | "refunded"
>;

export type ActiveOrderStatus = Exclude<OrderStatus, TerminalOrderStatus>;
// "draft" | "pending_payment" | "paid" | "shipped"

// Event names generated from the status union via template literal types:
export type OrderEventName = `order.${OrderStatus}`;
// "order.draft" | "order.pending_payment" | "order.paid" | ...

// ═════ 3. Boundary validation — HTTP body, DB row, queue message ═════════════

const ORDER_STATUS_VALUES = Object.values(ORDER_STATUS) as readonly string[];

export function isOrderStatus(value: unknown): value is OrderStatus {
  return typeof value === "string" && ORDER_STATUS_VALUES.includes(value);
}

export function parseOrderStatus(value: unknown, field = "status"): OrderStatus {
  if (!isOrderStatus(value)) {
    throw new ValidationError(
      `Invalid ${field}: ${JSON.stringify(value)}. ` +
      `Expected one of: ${ORDER_STATUS_VALUES.join(", ")}`,
    );
  }
  return value;
}

// A generic factory so you don't repeat this per constant object:
export function makeEnumGuard<const T extends Record<string, string>>(
  obj: T,
): (value: unknown) => value is T[keyof T] {
  const allowed = new Set<string>(Object.values(obj));
  return (value: unknown): value is T[keyof T] =>
    typeof value === "string" && allowed.has(value);
}

export const isPaymentProvider = makeEnumGuard(PAYMENT_PROVIDER);
export const isUserRole        = makeEnumGuard(USER_ROLE);

// ═════ 4. Zod schema derived from the same constant — one source of truth ════
// (z.enum accepts a readonly tuple of literals; Object.values gives us that.)

import { z } from "zod";

export const OrderStatusSchema = z.enum(
  Object.values(ORDER_STATUS) as [OrderStatus, ...OrderStatus[]],
);

export const UpdateOrderSchema = z.object({
  orderId:  z.number().int().positive(),
  status:   OrderStatusSchema,
  note:     z.string().max(500).optional(),
});

export type UpdateOrderRequest = z.infer<typeof UpdateOrderSchema>;
// { orderId: number; status: OrderStatus; note?: string }
// Note: z.infer produces the UNION, not an enum — the types line up exactly.
// With a TS enum you'd need z.nativeEnum() and the inferred type would be the
// nominal enum, forcing casts anywhere raw strings appear.

// ═════ 5. Exhaustive business logic ══════════════════════════════════════════

function assertNever(value: never, context: string): never {
  throw new Error(`${context}: unhandled value ${JSON.stringify(value)}`);
}

export function isTerminal(status: OrderStatus): status is TerminalOrderStatus {
  return status === ORDER_STATUS.Delivered
      || status === ORDER_STATUS.Cancelled
      || status === ORDER_STATUS.Refunded;
}

export function getCustomerFacingLabel(status: OrderStatus): string {
  switch (status) {
    case "draft":           return "Not yet submitted";
    case "pending_payment": return "Awaiting payment";
    case "paid":            return "Payment confirmed";
    case "shipped":         return "On its way";
    case "delivered":       return "Delivered";
    case "cancelled":       return "Cancelled";
    case "refunded":        return "Refunded";
    // Add a status to ORDER_STATUS → this becomes a compile error:
    default: return assertNever(status, "getCustomerFacingLabel");
  }
}

// A transition table typed by the union — every status must be listed:
const ALLOWED_TRANSITIONS: Record<OrderStatus, readonly OrderStatus[]> = {
  draft:           ["pending_payment", "cancelled"],
  pending_payment: ["paid", "cancelled"],
  paid:            ["shipped", "refunded", "cancelled"],
  shipped:         ["delivered", "refunded"],
  delivered:       ["refunded"],
  cancelled:       [],
  refunded:        [],
};

export function canTransition(from: OrderStatus, to: OrderStatus): boolean {
  return ALLOWED_TRANSITIONS[from].includes(to);
}

// ═════ 6. Service layer ══════════════════════════════════════════════════════

interface OrderRow {
  id:         number;
  user_id:    number;
  status:     string;          // raw from the driver — deliberately untyped
  provider:   string | null;
  total_cents: number;
}

export interface Order {
  orderId:    number;
  userId:     number;
  status:     OrderStatus;
  provider:   PaymentProvider | null;
  totalCents: number;
}

export class OrderService {
  private readonly db: Database;
  private readonly events: EventBus;
  private readonly logger: Logger;

  constructor(db: Database, events: EventBus, logger: Logger) {
    this.db = db;
    this.events = events;
    this.logger = logger;
  }

  // The single place where `string` becomes `OrderStatus`:
  private static fromRow(row: OrderRow): Order {
    return {
      orderId:    row.id,
      userId:     row.user_id,
      status:     parseOrderStatus(row.status, "orders.status"),
      provider:   isPaymentProvider(row.provider) ? row.provider : null,
      totalCents: row.total_cents,
    };
  }

  async findById(orderId: number): Promise<Order | null> {
    const row = await this.db.queryOne<OrderRow>(
      "SELECT * FROM orders WHERE id = $1",
      [orderId],
    );
    return row ? OrderService.fromRow(row) : null;
  }

  async updateStatus(
    orderId: number,
    nextStatus: OrderStatus,
    actorRole: UserRole,
  ): Promise<Order> {
    const order = await this.findById(orderId);
    if (!order) throw new NotFoundError(`Order ${orderId}`);

    if (isTerminal(order.status)) {
      throw new ConflictError(
        `Order ${orderId} is in terminal state "${order.status}" ` +
        `(${getCustomerFacingLabel(order.status)})`,
      );
    }

    if (!canTransition(order.status, nextStatus)) {
      throw new ConflictError(
        `Cannot move order ${orderId} from "${order.status}" to "${nextStatus}"`,
      );
    }

    // Refunds are staff-only — the union narrows the role check cleanly:
    if (nextStatus === "refunded" && actorRole === USER_ROLE.Customer) {
      throw new ForbiddenError("Customers cannot issue refunds");
    }

    await this.db.query(
      "UPDATE orders SET status = $1, updated_at = now() WHERE id = $2",
      [nextStatus, orderId],   // ✅ a plain string — no enum unwrapping needed
    );

    // Template literal type gives a typed event name for free:
    const eventName: OrderEventName = `order.${nextStatus}`;
    await this.events.publish(eventName, { orderId, userId: order.userId });

    this.logger.info("Order status changed", {
      orderId,
      from: order.status,
      to:   nextStatus,
      actorRole,
    });

    return { ...order, status: nextStatus };
  }
}

// ═════ 7. HTTP handler — request body validated into the union ═══════════════

export async function handleUpdateOrder(
  requestBody: unknown,
  authToken: AuthToken,
  service: OrderService,
): Promise<ApiResponse<Order>> {
  const parsed = UpdateOrderSchema.safeParse(requestBody);
  if (!parsed.success) {
    return { ok: false, statusCode: 422, error: parsed.error.message };
  }

  // parsed.data.status is already `OrderStatus` — no cast anywhere in this file.
  const order = await service.updateStatus(
    parsed.data.orderId,
    parsed.data.status,
    authToken.role,
  );

  return { ok: true, statusCode: 200, data: order };
}

// ── Supporting declarations used above ───────────────────────────────────────
declare class ValidationError extends Error {}
declare class NotFoundError   extends Error {}
declare class ConflictError   extends Error {}
declare class ForbiddenError  extends Error {}
declare interface Database { query(sql: string, params: unknown[]): Promise<void>;
                             queryOne<T>(sql: string, params: unknown[]): Promise<T | null>; }
declare interface EventBus { publish(name: string, payload: unknown): Promise<void>; }
declare interface Logger   { info(msg: string, meta?: Record<string, unknown>): void; }
declare interface AuthToken { userId: number; role: UserRole; }
declare type ApiResponse<T> =
  | { ok: true;  statusCode: number; data: T }
  | { ok: false; statusCode: number; error: string };
```

---

## Going deeper

### 1. Enums declare a type *and* a value — union types do not

```ts
enum Region { UsEast = "us-east-1", EuWest = "eu-west-1" }

// `Region` lives in BOTH namespaces:
const r: Region = Region.UsEast;   // type position + value position
const obj = Region;                // ✅ it's a real object you can pass around
console.log(Object.keys(obj));     // ["UsEast", "EuWest"]

// A union type lives ONLY in the type namespace:
type RegionU = "us-east-1" | "eu-west-1";
// const x = RegionU;              // ❌ 'RegionU' only refers to a type, not a value

// This is exactly why the `as const` object pattern pairs a VALUE (the object)
// with a TYPE (the derived union) — it reconstructs the dual nature deliberately,
// out of two plain, erasable pieces.
```

### 2. Enum declaration merging and `declare enum`

```ts
// Enums can merge across declarations, like interfaces — but only if all but
// the first are const-initialized or the members don't collide:
enum FeatureFlag { NewCheckout = "new_checkout" }
enum FeatureFlag { DarkMode    = "dark_mode" }   // merged

const f: FeatureFlag = FeatureFlag.DarkMode;     // ✅ both members exist

// Numeric enum merging requires explicit initializers after the first block:
enum Codes { A = 1 }
enum Codes { B = 2 }        // ✅ explicit — required
// enum Codes { C }         // ❌ error: enum member must have initializer

// `declare enum` describes an enum that exists at runtime elsewhere
// (typical in .d.ts for a JS library). No code emitted for the declaration:
declare enum ExternalStatus { Ok = 0, Fail = 1 }
```

### 3. Computed vs constant enum members

```ts
enum Mixed {
  A = 1,                          // constant
  B = A * 2,                      // constant (compiler can evaluate it)
  C = "abc".length,               // COMPUTED — evaluated at runtime
  // D,                           // ❌ error: member must have initializer,
                                  //    because it follows a computed member
  D = 4,                          // ✅ explicit is fine
}

// Constant members participate in narrowing and can be used in type positions:
type OnlyA = Mixed.A;             // ✅

// Computed members cannot be used as literal types in some positions and
// prevent `const enum` inlining entirely:
// const enum Bad { X = "abc".length }  // ❌ error: const enum member initializers
//                                      //    must be constant expressions
```

### 4. Numeric enums make terrible object keys and JSON payloads

```ts
enum Priority { Low, Normal, High }

const job = { id: "job-1", priority: Priority.High };
JSON.stringify(job);                // '{"id":"job-1","priority":2}'
// The wire format is `2`. A consumer in another language, or a human reading
// the logs, has no idea what 2 means. Renumbering the enum silently changes
// the meaning of every historical record in your database and event log.

// String enums fix the wire format:
enum PriorityStr { Low = "low", Normal = "normal", High = "high" }
JSON.stringify({ priority: PriorityStr.High });   // '{"priority":"high"}'

// Unions give the same wire format with zero runtime:
type PriorityU = "low" | "normal" | "high";
JSON.stringify({ priority: "high" as PriorityU }); // '{"priority":"high"}'

// Rule: never persist numeric enum values. Ever. If your DB column is an int,
// map it explicitly through a Record so the mapping is versioned in code.
```

### 5. Tree-shaking: enums usually survive, `as const` objects usually don't

```ts
// enums.ts
export enum HttpMethod { Get = "GET", Post = "POST", Put = "PUT", Delete = "DELETE" }

// The emitted IIFE mutates the outer `HttpMethod` binding — many bundlers treat
// that as a side effect and keep the whole thing even if you use one member.
// Rollup and esbuild have special-cased TS enums, but the handling is fragile
// and breaks when the enum is re-exported through barrel files.

// constants.ts
export const HTTP_METHOD = { Get: "GET", Post: "POST" } as const;
// A plain object literal with no side effects. Bundlers can drop unused
// properties via property mangling, and drop the whole binding if unreferenced.
// More importantly, if you only need the TYPE, you import nothing at runtime:

import type { HttpMethodValue } from "./constants";   // erased completely
```

### 6. `keyof typeof` on an enum vs on a const object

```ts
enum Tier { Free = "free", Pro = "pro", Enterprise = "enterprise" }

type TierKeys   = keyof typeof Tier;    // "Free" | "Pro" | "Enterprise"  (the NAMES)
type TierValues = `${Tier}`;            // "free" | "pro" | "enterprise"  (the VALUES)
// `${Tier}` — a template literal trick that converts a string enum to its
// value union. Extremely useful for accepting raw strings at an API boundary
// while keeping the enum internally:

function setTier(tier: `${Tier}`): void { /* accepts "pro" AND Tier.Pro */ }
setTier("pro");        // ✅ now allowed
setTier(Tier.Pro);     // ✅ also allowed

// The const-object version needs no trick at all:
const TIER = { Free: "free", Pro: "pro", Enterprise: "enterprise" } as const;
type TierKeys2   = keyof typeof TIER;                 // "Free" | "Pro" | "Enterprise"
type TierValues2 = typeof TIER[keyof typeof TIER];    // "free" | "pro" | "enterprise"
```

### 7. Enums cross module boundaries badly in monorepos

```ts
// packages/shared/src/status.ts
export enum OrderStatus { Paid = "paid" }

// If two packages end up with two COPIES of shared (different versions hoisted
// differently by the package manager), you get two distinct OrderStatus objects.
// They are structurally identical but nominally different:
//
//   import { OrderStatus } from "@acme/shared";        // v1.2.0 copy
//   import { updateOrder } from "@acme/orders";        // bundles v1.1.0 copy
//   updateOrder(OrderStatus.Paid);
//   // ❌ Argument of type 'OrderStatus' is not assignable to
//   //    parameter of type 'OrderStatus'. Two different declarations.
//
// This is one of the most confusing errors in TypeScript, and unions are
// completely immune to it: "paid" is "paid" in every copy of every package.
```

### 8. Performance is a non-issue; DX is the real cost

Property access on an enum object is a monomorphic inline-cached lookup — V8 handles it at full speed. Nobody's API is slow because of enums. The real costs are:

- **Import friction**: every `case` clause and every test fixture needs an import.
- **Cast pressure**: every boundary (`JSON.parse`, DB row, `process.env`, queue message) needs `as SomeEnum`, and casts are where bugs hide.
- **Tooling friction**: `erasableSyntaxOnly`, `isolatedModules`, Node type-stripping, Babel, Bun, Deno.
- **Test noise**: `expect(order.status).toBe(OrderStatus.Paid)` reads worse than `toBe("paid")`, and snapshot diffs of numeric enums are unreadable.

### 9. When an enum is genuinely the right call

- **Bit flags.** `Permission.Read | Permission.Write` is idiomatic with numeric enums and awkward with unions.
- **A large ordered scale where reverse mapping is core**, e.g. mapping a stored `smallint` severity back to a name in a hot logging path.
- **Generated code**, where the generator emits enums and you don't control it: protobuf (`ts-proto`), some GraphQL codegen presets, older Prisma output. Fighting the generator is not worth it.
- **Interop with a JS library that exports a real enum object** — you want the same object identity.

Everywhere else in a modern Node backend: use a union, usually derived from an `as const` object.

---

## Common mistakes

### Mistake 1 — Casting external data straight into an enum

```ts
// ❌ WRONG — `as` performs zero runtime validation
enum OrderStatus { Pending = "pending", Paid = "paid" }

app.post("/orders/:id/status", (req, res) => {
  const status = req.body.status as OrderStatus;   // could be "banana"
  // Every downstream switch now has an unhandled case that falls through
  // to `default`, or worse, silently does nothing.
  orderService.update(Number(req.params.id), status);
});
```

```ts
// ✅ RIGHT — validate at the boundary, narrow into the union
const ORDER_STATUS = { Pending: "pending", Paid: "paid" } as const;
type OrderStatus = typeof ORDER_STATUS[keyof typeof ORDER_STATUS];

function isOrderStatus(value: unknown): value is OrderStatus {
  return typeof value === "string"
      && (Object.values(ORDER_STATUS) as string[]).includes(value);
}

app.post("/orders/:id/status", (req, res) => {
  const raw = (req.body as { status?: unknown }).status;
  if (!isOrderStatus(raw)) {
    res.status(422).json({ error: `Invalid status: ${String(raw)}` });
    return;
  }
  orderService.update(Number(req.params.id), raw);   // raw: OrderStatus, verified
});
```

### Mistake 2 — Persisting numeric enum values to a database

```ts
// ❌ WRONG — the integer meaning is defined only by source-code ordering
enum SubscriptionTier { Free, Basic, Pro }        // 0, 1, 2

await db.query(
  "INSERT INTO subscriptions (user_id, tier) VALUES ($1, $2)",
  [userId, SubscriptionTier.Pro],                 // writes 2
);

// Six months later someone alphabetises the enum:
enum SubscriptionTierV2 { Basic, Free, Pro }      // Basic=0, Free=1, Pro=2
// ...or inserts a tier in the middle:
enum SubscriptionTierV3 { Free, Trial, Basic, Pro }  // Pro is now 3
// Every existing row silently changes meaning. No migration ran. No test failed.
```

```ts
// ✅ RIGHT — persist strings; they are self-describing and stable
const SUBSCRIPTION_TIER = {
  Free:  "free",
  Basic: "basic",
  Pro:   "pro",
} as const;
type SubscriptionTier = typeof SUBSCRIPTION_TIER[keyof typeof SUBSCRIPTION_TIER];

await db.query(
  "INSERT INTO subscriptions (user_id, tier) VALUES ($1, $2)",
  [userId, SUBSCRIPTION_TIER.Pro],                // writes "pro" — unambiguous forever
);
// Reordering the object literal changes nothing. Renaming a KEY changes nothing.
// Only changing a VALUE is a breaking change — and that's visible in the diff.
```

### Mistake 3 — Using `const enum` in a library or with a modern bundler

```ts
// ❌ WRONG — breaks under esbuild/babel/swc/Vite, and in published packages
// packages/shared/src/limits.ts
export const enum RateLimit {
  PerMinute = 60,
  PerHour   = 1000,
}

// packages/api/src/middleware.ts  (transpiled file-by-file)
import { RateLimit } from "@acme/shared";
limiter.configure({ max: RateLimit.PerMinute });
// esbuild emits:  limiter.configure({ max: RateLimit.PerMinute })
// but nothing named RateLimit exists at runtime →
// TypeError: Cannot read properties of undefined (reading 'PerMinute')
```

```ts
// ✅ RIGHT — plain const object, works with every toolchain
export const RATE_LIMIT = {
  PerMinute: 60,
  PerHour:   1000,
} as const;

export type RateLimitValue = typeof RATE_LIMIT[keyof typeof RATE_LIMIT];  // 60 | 1000

import { RATE_LIMIT } from "@acme/shared";
limiter.configure({ max: RATE_LIMIT.PerMinute });   // a real import, a real value
```

### Mistake 4 — Assuming `Object.values(SomeEnum)` gives you just the values

```ts
// ❌ WRONG — numeric enums include the reverse-mapped keys
enum Priority { Low, Normal, High }

const all = Object.values(Priority);
console.log(all);
// ["Low", "Normal", "High", 0, 1, 2]   ← six entries, mixed types!

for (const p of Object.values(Priority)) {
  registerPriority(p as Priority);   // registers 6 things, half of them strings
}
```

```ts
// ✅ RIGHT (if you must keep the numeric enum) — filter the reverse map out
enum Priority2 { Low, Normal, High }

const numericValues = Object.values(Priority2)
  .filter((v): v is Priority2 => typeof v === "number");
console.log(numericValues);   // [0, 1, 2]

// ✅ BETTER — a const object has no reverse map to filter
const PRIORITY = { Low: "low", Normal: "normal", High: "high" } as const;
const values = Object.values(PRIORITY);   // ["low", "normal", "high"] — correct, typed
```

### Mistake 5 — Comparing an enum member to a raw string

```ts
// ❌ WRONG — the compiler rejects the comparison outright
enum UserRole { Admin = "admin", Viewer = "viewer" }

function canDelete(role: UserRole): boolean {
  return role === "admin";
  // ❌ This comparison appears to be unintentional because the types
  //    'UserRole' and '"admin"' have no overlap.
}
// Developers "fix" this with `role === ("admin" as UserRole)` — which works
// but disables the very check that made them use an enum in the first place.
```

```ts
// ✅ RIGHT (with an enum) — compare against the member
function canDeleteEnum(role: UserRole): boolean {
  return role === UserRole.Admin;
}

// ✅ RIGHT (with a union) — the comparison you naturally wanted just works
const USER_ROLE = { Admin: "admin", Viewer: "viewer" } as const;
type UserRoleU = typeof USER_ROLE[keyof typeof USER_ROLE];

function canDeleteU(role: UserRoleU): boolean {
  return role === "admin";              // ✅ and "admn" would be a compile error
}
```

---

## Practice exercises

### Exercise 1 — easy

Model an HTTP status classification system without using `enum` anywhere.

Requirements:

1. Define `HTTP_METHOD` as an `as const` object with `Get`, `Post`, `Put`, `Patch`, `Delete` mapping to `"GET"`, `"POST"`, `"PUT"`, `"PATCH"`, `"DELETE"`.
2. Derive `type HttpMethod` from it using `typeof` + `keyof typeof`.
3. Write `isHttpMethod(value: unknown): value is HttpMethod` — a real runtime guard.
4. Define `type StatusCategory = "success" | "client_error" | "server_error" | "redirect" | "informational"`.
5. Write `classifyStatus(statusCode: number): StatusCategory` handling 1xx–5xx, throwing on anything outside 100–599.
6. Write `isSafeMethod(method: HttpMethod): boolean` — true only for `GET`.
7. Write `describeMethod(method: HttpMethod): string` as an exhaustive `switch` with an `assertNever` default, so adding `Options` to `HTTP_METHOD` becomes a compile error.

```ts
// Write your code here
```

### Exercise 2 — medium

You have inherited a service that uses enums throughout. Migrate it to unions **without breaking any existing call site that passes enum members**, then remove the enum.

Starting point (rewrite it yourself — do not copy mechanically):

```ts
enum NotificationChannel {
  Email = "email",
  Sms   = "sms",
  Push  = "push",
  Slack = "slack",
}

enum DeliveryStatus {
  Queued    = 0,
  Sent      = 1,
  Delivered = 2,
  Bounced   = 3,
  Failed    = 4,
}
```

Write, from scratch:

1. `NOTIFICATION_CHANNEL` and `DELIVERY_STATUS` as `as const` objects — but `DELIVERY_STATUS` must map to **strings** (`"queued"`, `"sent"`, …), not numbers, since the numbers were being persisted.
2. The derived union types `NotificationChannel` and `DeliveryStatus`.
3. `DELIVERY_STATUS_CODE: Record<DeliveryStatus, number>` preserving the old integers `0–4`, plus `deliveryStatusFromCode(code: number): DeliveryStatus` that throws on unknown codes — this is your migration bridge for existing DB rows.
4. `parseNotificationChannel(value: unknown): NotificationChannel` throwing a descriptive error listing valid values.
5. A `NotificationPayload` discriminated union keyed on `channel`, where each channel carries different fields: email needs `to`/`subject`/`body`, sms needs `phoneNumber`/`message`, push needs `deviceToken`/`title`/`body`, slack needs `webhookUrl`/`text`.
6. `renderPreview(payload: NotificationPayload): string` — exhaustive switch over `payload.channel` with `assertNever`.
7. `isRetryable(status: DeliveryStatus): boolean` — true for `"bounced"` and `"failed"` only, written so adding a new status forces you to revisit it.
8. A `NotificationService` class with `send(payload, channel)`, `recordDelivery(notificationId, status)` and `retryFailed(since: Date)`. No `enum` keyword, no `as` casts except inside the guard functions.

```ts
// Write your code here
```

### Exercise 3 — hard

Build a **type-safe enum-replacement toolkit** that a whole codebase could adopt, then use it to model a permission system.

Part A — the toolkit:

```ts
// Implement these, all fully typed, no `enum` keyword, no `any`:
//
// createConstEnum<const T extends Record<string, string | number>>(def: T)
//   returns an object with:
//     .values     — readonly array of T[keyof T], correctly typed
//     .keys       — readonly array of keyof T
//     .is(v: unknown): v is T[keyof T]      — runtime guard
//     .parse(v: unknown): T[keyof T]        — throws a descriptive error
//     .keyOf(v: T[keyof T]): keyof T        — the reverse map enums give you free
//     .labels<L extends Record<T[keyof T], string>>(map: L): (v: T[keyof T]) => string
//         — exhaustive label map; missing a member must be a compile error
//   plus the def's own members accessible as properties (Toolkit.Admin === "admin")
//
// Type-level helpers:
//   type ValueOf<T>          — T[keyof T]
//   type EnumLike            — Record<string, string | number>
//   type Prefixed<P extends string, T extends string> — `${P}${T}` template union
```

Part B — use it to model permissions:

```ts
// - PERMISSION: "user.read" | "user.write" | "user.delete" | "order.read"
//               | "order.write" | "order.refund" | "admin.impersonate"
// - RESOURCE derived from PERMISSION via a template-literal split:
//     type Resource = "user" | "order" | "admin"      (derived, NOT hand-written)
//     type Action   = "read" | "write" | "delete" | "refund" | "impersonate"
//   Hint: use an infer-based conditional type on the `${Resource}.${Action}` shape.
// - ROLE: "owner" | "admin" | "support" | "customer"
// - ROLE_PERMISSIONS: Record<Role, readonly Permission[]> — exhaustive
// - hasPermission(role: Role, permission: Permission): boolean
// - requirePermission(authToken, permission): asserts the caller has it, else throws
// - permissionsForResource<R extends Resource>(resource: R): Extract<Permission, `${R}.${string}`>[]
//     — the RETURN TYPE must be narrowed to that resource's permissions only
//
// Part C — prove it works:
// - Show a compile error when a Role is added to ROLE but missing from ROLE_PERMISSIONS.
// - Show a compile error when .labels() omits a member.
// - Show that permissionsForResource("order") is typed as
//     ("order.read" | "order.write" | "order.refund")[]
//   and NOT Permission[].
// - Show that a raw string "user.read" is accepted where Permission is expected,
//   but "user.destroy" is a compile error.
```

```ts
// Write your code here
```

---

## Quick reference cheat sheet

```ts
// ── Numeric enum ─────────────────────────────────────────────────────────────
enum Level { Debug, Info, Warn }        // 0, 1, 2
Level.Info;                             // 1
Level[1];                               // "Info" — reverse mapping
Object.values(Level);                   // ["Debug","Info","Warn",0,1,2]  ⚠️ both

// ── String enum ──────────────────────────────────────────────────────────────
enum Status { Paid = "paid" }           // every member needs an initializer
Status.Paid;                            // "paid"
// Status["paid"];                      // ❌ no reverse mapping
// const s: Status = "paid";            // ❌ nominal — raw literal rejected
type StatusValues = `${Status}`;        // "paid" — escape hatch to accept literals

// ── const enum (avoid) ───────────────────────────────────────────────────────
const enum Ttl { Long = 3600 }          // inlined; breaks isolatedModules/esbuild

// ── Literal union ────────────────────────────────────────────────────────────
type Role = "admin" | "viewer";         // zero runtime output
const r: Role = "admin";                // ✅ raw literals work

// ── as const object + typeof union (recommended) ─────────────────────────────
const ROLE = { Admin: "admin", Viewer: "viewer" } as const;
type RoleValue = typeof ROLE[keyof typeof ROLE];   // "admin" | "viewer"
type RoleKey   = keyof typeof ROLE;                // "Admin" | "Viewer"
Object.values(ROLE);                               // ["admin","viewer"] — clean

// ── Runtime guard ────────────────────────────────────────────────────────────
const isRole = (v: unknown): v is RoleValue =>
  typeof v === "string" && (Object.values(ROLE) as string[]).includes(v);

// ── Exhaustiveness ───────────────────────────────────────────────────────────
function assertNever(v: never): never { throw new Error(`Unhandled: ${String(v)}`); }

// ── Derived types (unions only) ──────────────────────────────────────────────
type Terminal = Extract<RoleValue, "admin">;
type Active   = Exclude<RoleValue, "admin">;
type EventName = `role.${RoleValue}`;              // "role.admin" | "role.viewer"
type Handlers  = { [K in RoleValue]: () => void };

// ── tsconfig flags that matter ───────────────────────────────────────────────
// "isolatedModules": true       → const enum export becomes an error
// "verbatimModuleSyntax": true  → implies isolatedModules
// "erasableSyntaxOnly": true    → enum + parameter properties become errors
// "preserveConstEnums": true    → const enum emits the object anyway (pointless)
```

### Decision table

| You need… | Enum | Literal union | `as const` + `typeof` |
|---|---|---|---|
| Zero runtime output | ❌ emits IIFE + object | ✅ fully erased | ⚠️ one small frozen object |
| Runs under `node --experimental-strip-types` | ❌ SyntaxError | ✅ | ✅ |
| Passes `erasableSyntaxOnly: true` | ❌ TS1294 | ✅ | ✅ |
| Works with esbuild / swc / Babel / Vite | ⚠️ yes, but `const enum` breaks | ✅ | ✅ |
| Accepts raw string literals at call sites | ❌ nominal (string enums) | ✅ | ✅ |
| Named constants (`X.Admin`) | ✅ | ❌ | ✅ |
| Reverse mapping (`X[2] → "High"`) | ✅ numeric only | ❌ build a `Record` | ❌ build a `Record` |
| Iterate all members | ⚠️ numeric includes reverse keys | ❌ need a separate array | ✅ `Object.values` |
| `Extract` / `Exclude` / template literal types | ⚠️ clumsy | ✅ | ✅ |
| Immune to duplicate-package nominal clashes | ❌ | ✅ | ✅ |
| Bit flags (`Read \| Write`) | ✅ idiomatic | ⚠️ manual | ⚠️ manual |
| Safe to persist to a DB | ⚠️ strings only, never numeric | ✅ | ✅ |
| Zod / codegen interop | ⚠️ `z.nativeEnum`, casts | ✅ `z.enum` | ✅ `z.enum` |
| **Default recommendation** | only for bit flags / generated code | small local sets | **everything else** |

---

## Connected topics

- **09 — Union types** — the foundation; literal unions are just unions of literal types, and every enum alternative here is built on them.
- **42 — Discriminated unions** — once your status field is a literal union, tagging object types by it gives you per-state field guarantees and exhaustive switches.
- **58 — Literal types and `as const`** — the exact widening rules that make `as const` produce `"admin"` instead of `string`, which the whole modern pattern depends on.
- **70 — Namespaces** — the other classic non-erasable TypeScript construct, with the same `isolatedModules` / `erasableSyntaxOnly` story as `enum`.
- **32 — Utility types** — `Extract`, `Exclude`, and `Record` are what turn a literal union into subsets, transition tables, and exhaustive label maps.
