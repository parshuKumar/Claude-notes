# 70 — Namespaces

## What is this?

A **namespace** is TypeScript's own module system — invented in 2012, years before `import`/`export` were standardised in JavaScript, and kept alive ever since purely for backwards compatibility and for a handful of declaration-file jobs that ES modules genuinely cannot do.

```ts
// A namespace groups related declarations under a single named object:
namespace TokenUtils {
  const SECRET = process.env.JWT_SECRET ?? "dev-secret";   // private — not exported

  export interface DecodedToken {
    userId: number;
    role: string;
    expiresAt: number;
  }

  export function decode(authToken: string): DecodedToken {
    // ... verify signature with SECRET, parse payload ...
    return { userId: 1, role: "admin", expiresAt: Date.now() + 3600_000 };
  }

  export function isExpired(token: DecodedToken): boolean {
    return token.expiresAt < Date.now();
  }
}

// Consumers reach in through the namespace name — no import statement:
const decoded: TokenUtils.DecodedToken = TokenUtils.decode("eyJhbGci...");
if (TokenUtils.isExpired(decoded)) {
  throw new Error("Token expired");
}
```

Two things are worth noticing immediately.

First, `TokenUtils` — exactly like an `enum` — creates **both a type-namespace entry and a runtime value**. `TokenUtils.DecodedToken` is a type; `TokenUtils.decode` is a real function living on a real object. That dual nature is the source of every namespace superpower and every namespace problem.

Second, there is no `import`. The namespace is globally visible to every file in the compilation, with nothing linking them together except a shared global scope. That was the whole point in 2012 (script tags, `<script src>` concatenation, no bundlers) and is the whole problem in 2026.

This document covers what `namespace` compiles to, nesting and merging, the declaration-merging tricks that make namespaces still relevant, why ES modules replaced them, the four places you will legitimately meet a namespace in a modern Node backend, how to read and migrate legacy namespace code — and why you should never start a new file with one.

---

## Why does it matter?

You will not *write* namespaces in new application code. You will absolutely *read* them, and you will occasionally *have* to write one. Concretely:

- **`declare global { namespace Express { interface Request { … } } }`** — the single most common way to attach `req.user` / `req.requestId` to Express. If you have ever typed an auth middleware, you have written a namespace.
- **`NodeJS.ProcessEnv`, `NodeJS.Timeout`, `NodeJS.ErrnoException`, `NodeJS.ReadableStream`** — the `@types/node` package puts almost everything global inside a `NodeJS` namespace. Typing `process.env` correctly means augmenting it.
- **`.d.ts` files for untyped JavaScript libraries** — the older the library, the more likely its types are a namespace. `@types/express`, `@types/jsonwebtoken`, `@types/multer`, `@types/passport` all use them.
- **Legacy TypeScript codebases** — anything started before ~2016, and plenty started after, is organised as `namespace App.Services.Orders { … }` across dozens of files glued with `/// <reference path="…" />`. You will be asked to migrate one.
- **`erasableSyntaxOnly` / Node type-stripping** — namespaces with bodies are non-erasable, exactly like enums. If your project wants `node server.ts` with no build step, an inherited namespace is a hard blocker you must remove.

Getting this wrong has real costs: an augmentation placed in the wrong file silently does nothing (the most common TypeScript support question on the internet), a namespace inside a module accidentally becomes a nested object nobody can import, and a namespace in application code quietly disables tree-shaking for the whole file.

---

## The JavaScript way vs the TypeScript way

Before ES modules, JavaScript solved "don't pollute the global scope" with the **module pattern** — an IIFE that returns a public object.

```js
// ── JavaScript circa 2012 — the revealing module pattern ─────────────────────

var TokenUtils = (function () {
  // Private state: invisible outside this function.
  var SECRET = "dev-secret";

  function sign(payload) {
    return "header." + btoa(JSON.stringify(payload)) + "." + SECRET;
  }

  function decode(authToken) {
    var parts = authToken.split(".");
    return JSON.parse(atob(parts[1]));
  }

  // Only what you return is public:
  return {
    sign: sign,
    decode: decode,
  };
})();

TokenUtils.decode("a.b.c");     // ✅ public
// TokenUtils.SECRET;           // undefined — private, by construction

// ── Extending it later, in another <script> ──────────────────────────────────
var TokenUtils = TokenUtils || {};
(function (ns) {
  ns.isExpired = function (token) {
    return token.exp * 1000 < Date.now();
  };
})(TokenUtils);
// The `ns || (ns = {})` dance is how you MERGE across files. Order matters:
// if this script loads first, `TokenUtils.decode` doesn't exist yet.

// ── What this pattern cannot do ──────────────────────────────────────────────
// 1. No types. `token` is whatever showed up. `TokenUtils.decdoe(...)` → undefined.
// 2. No dependency declaration. Load order is managed by hand in index.html.
// 3. No tooling. Nothing can tell you TokenUtils is unused, or safely rename it.
// 4. Nested namespaces (App.Services.Orders) mean four lines of || {} boilerplate.
```

```ts
// ── The TypeScript way (2012–2015): `namespace` is that pattern, typed ───────

namespace TokenUtils {
  const SECRET: string = "dev-secret";     // no `export` → private, exactly like above

  export interface TokenPayload {
    userId: number;
    role: "admin" | "support" | "customer";
    exp: number;                            // unix seconds
  }

  export function sign(payload: TokenPayload): string {
    return `header.${btoa(JSON.stringify(payload))}.${SECRET}`;
  }

  export function decode(authToken: string): TokenPayload {
    const [, body] = authToken.split(".");
    return JSON.parse(atob(body)) as TokenPayload;
  }
}

// Merging across files needs no boilerplate — the compiler emits the `|| {}`:
namespace TokenUtils {
  export function isExpired(token: TokenPayload): boolean {
    return token.exp * 1000 < Date.now();
  }
}

const payload = TokenUtils.decode("a.b.c");   // payload: TokenUtils.TokenPayload
TokenUtils.isExpired(payload);                // ✅ typed, autocompleted
// TokenUtils.decdoe("a.b.c");                // ❌ Property 'decdoe' does not exist
// TokenUtils.SECRET;                         // ❌ Property 'SECRET' does not exist
```

```ts
// ── The modern way (2015–today): ES modules ──────────────────────────────────
// tokenUtils.ts — the FILE is the boundary. No wrapper, no global name.

const SECRET: string = process.env.JWT_SECRET ?? "dev-secret";   // file-private

export interface TokenPayload {
  userId: number;
  role: "admin" | "support" | "customer";
  exp: number;
}

export function sign(payload: TokenPayload): string { /* ... */ return ""; }
export function decode(authToken: string): TokenPayload { /* ... */ return {} as TokenPayload; }
export function isExpired(token: TokenPayload): boolean { return token.exp * 1000 < Date.now(); }
```

```ts
// consumer.ts — dependencies are explicit, statically analysable, tree-shakeable
import { decode, isExpired, type TokenPayload } from "./tokenUtils.js";

const payload: TokenPayload = decode("a.b.c");
if (isExpired(payload)) throw new Error("Token expired");
```

The revelation: **a namespace is the 2012 module pattern with types bolted on; an ES module is the same idea blessed by the language, with a real dependency graph.** Everything a namespace does for code organisation, a module file does better. The only things left are the type-level tricks in the second half of this document.

---

## Syntax

```ts
// ── Basic namespace ──────────────────────────────────────────────────────────
namespace Validation {
  export const MAX_BODY_BYTES = 1_048_576;      // exported → visible outside
  const INTERNAL_TAG = "validation";            // not exported → private

  export interface ValidationIssue {
    field: string;
    message: string;
  }

  export function fail(field: string, message: string): ValidationIssue {
    return { field, message: `[${INTERNAL_TAG}] ${message}` };
  }
}

Validation.MAX_BODY_BYTES;                      // ✅ 1048576
const issue: Validation.ValidationIssue = Validation.fail("email", "required");
// Validation.INTERNAL_TAG;                     // ❌ not exported

// ── Nested namespaces ────────────────────────────────────────────────────────
namespace Api {
  export namespace V1 {
    export interface UserDto { userId: number; email: string }
  }
  export namespace V2 {
    export interface UserDto { userId: string; email: string; createdAt: string }
  }
}

const legacy: Api.V1.UserDto = { userId: 42, email: "a@b.com" };
const modern: Api.V2.UserDto = { userId: "usr_42", email: "a@b.com", createdAt: "2026-01-01" };

// ── Dotted shorthand — identical to the nested form above ────────────────────
namespace Api.V3 {
  export interface UserDto { userId: string; email: string; tier: "free" | "pro" }
}
const next: Api.V3.UserDto = { userId: "usr_42", email: "a@b.com", tier: "pro" };
// NOTE: with dotted shorthand, every intermediate namespace is implicitly exported.

// ── Merging: two blocks with the same name become one ────────────────────────
namespace Metrics { export function counter(name: string): void {} }
namespace Metrics { export function histogram(name: string): void {} }
Metrics.counter("http_requests_total");
Metrics.histogram("http_request_duration_seconds");

// ── Alias import (namespace-local, erasable) ─────────────────────────────────
import UserV2 = Api.V2;                          // alias for a namespace
import UserDtoV2 = Api.V2.UserDto;               // alias for a type inside it
const aliased: UserDtoV2 = { userId: "u1", email: "x@y.com", createdAt: "" };

// ── Ambient namespace — describes something that exists elsewhere ────────────
declare namespace LegacyAnalytics {
  interface TrackEvent { name: string; userId: number; props?: Record<string, unknown> }
  function track(event: TrackEvent): void;
  const version: string;
}
// No `export` keyword needed inside `declare namespace` — everything is exported.
// Emits ZERO JavaScript. This is the .d.ts form.

// ── Global augmentation from inside a module ─────────────────────────────────
export {};                                       // makes this file a module
declare global {
  namespace Express {
    interface Request {
      userId?: number;
      requestId: string;
    }
  }
  namespace NodeJS {
    interface ProcessEnv {
      DATABASE_URL: string;
      JWT_SECRET: string;
    }
  }
}

// ── Legacy multi-file glue (pre-modules) ─────────────────────────────────────
/// <reference path="./validation.ts" />
/// <reference types="node" />
```

---

## How it works — concept by concept

### Concept 1 — What a namespace actually compiles to

This is the foundation. A namespace with a body is **not erased** — it emits an IIFE and a real object, exactly like an enum.

```ts
// ── Source TypeScript ────────────────────────────────────────────────────────
namespace TokenUtils {
  const SECRET = "dev-secret";

  export interface TokenPayload {
    userId: number;
    exp: number;
  }

  export function decode(authToken: string): TokenPayload {
    return JSON.parse(atob(authToken.split(".")[1])) as TokenPayload;
  }

  export const ALGORITHM = "HS256";
}
```

```js
// ── Emitted JavaScript (target ES2020) ───────────────────────────────────────
var TokenUtils;
(function (TokenUtils) {
  const SECRET = "dev-secret";                     // stays inside the closure — private

  function decode(authToken) {
    return JSON.parse(atob(authToken.split(".")[1]));
  }
  TokenUtils.decode = decode;                      // `export` → attach to the object

  TokenUtils.ALGORITHM = "HS256";                  // `export const` → attach too
})(TokenUtils || (TokenUtils = {}));

// The `TokenPayload` interface emitted NOTHING — types are always erased.
// The IIFE parameter shadowing is what makes merging work:
//   `TokenUtils || (TokenUtils = {})` reuses the object if it already exists.
```

Three consequences fall straight out of that emitted code:

```ts
// 1. `export` inside a namespace means "attach to the namespace object".
//    It has nothing to do with ES module exports.
namespace Config {
  const poolSize = 10;              // → local const, invisible outside
  export const timeoutMs = 5_000;   // → Config.timeoutMs = 5000
}

// 2. The namespace object is a real runtime value you can pass around:
const ns = TokenUtils;
console.log(Object.keys(ns));       // ["decode", "ALGORITHM"] — no SECRET, no interface

// 3. Because the IIFE mutates an outer binding, bundlers see a side effect
//    and generally keep the whole namespace even if you use one member.
//    Namespaces do not tree-shake.
```

A namespace that contains **only types** is different — it emits nothing at all:

```ts
// ── Source ───────────────────────────────────────────────────────────────────
namespace ApiTypes {
  export interface UserDto { userId: number; email: string }
  export type ApiResponse<T> = { ok: true; data: T } | { ok: false; error: string };
}

const response: ApiTypes.ApiResponse<ApiTypes.UserDto> = {
  ok: true,
  data: { userId: 1, email: "a@b.com" },
};
```

```js
// ── Emitted JavaScript ───────────────────────────────────────────────────────
const response = { ok: true, data: { userId: 1, email: "a@b.com" } };
// The namespace vanished completely. TypeScript elides namespaces whose members
// are all types — there is nothing to attach to an object at runtime.
```

That elision is exactly why "type-only grouping" survives as one of the few legitimate uses. But it is fragile: add a single `export const` and the IIFE reappears.

---

### Concept 2 — `export` inside a namespace controls visibility, not modules

Newcomers read `export` and think "importable from another file". Inside a namespace it means something completely different.

```ts
namespace RateLimiter {
  // ── Private implementation details ─────────────────────────────────────────
  const buckets = new Map<string, { count: number; resetAt: number }>();

  function keyFor(userId: number, route: string): string {
    return `${userId}:${route}`;
  }

  function nowMs(): number {
    return Date.now();
  }

  // ── Public surface ─────────────────────────────────────────────────────────
  export interface LimitResult {
    allowed: boolean;
    remaining: number;
    resetAt: number;
  }

  export const WINDOW_MS = 60_000;
  export const MAX_REQUESTS = 100;

  export function check(userId: number, route: string): LimitResult {
    const key = keyFor(userId, route);          // ✅ private helper reachable inside
    const bucket = buckets.get(key);
    const now = nowMs();

    if (!bucket || bucket.resetAt <= now) {
      buckets.set(key, { count: 1, resetAt: now + WINDOW_MS });
      return { allowed: true, remaining: MAX_REQUESTS - 1, resetAt: now + WINDOW_MS };
    }

    bucket.count += 1;
    return {
      allowed: bucket.count <= MAX_REQUESTS,
      remaining: Math.max(0, MAX_REQUESTS - bucket.count),
      resetAt: bucket.resetAt,
    };
  }
}

// Outside the namespace:
RateLimiter.check(42, "POST /orders");   // ✅
RateLimiter.WINDOW_MS;                   // ✅ 60000
// RateLimiter.buckets;                  // ❌ Property 'buckets' does not exist on
//                                       //    type 'typeof RateLimiter'.
// RateLimiter.keyFor(42, "x");          // ❌ same
```

The encapsulation is real and enforced at both the type level and the runtime level (the private members live in the IIFE closure and are never attached to the object). It is genuinely the one thing namespaces do that plain object literals cannot.

But a module file gives you the exact same encapsulation for free:

```ts
// rateLimiter.ts — everything not exported is file-private, no wrapper needed
const buckets = new Map<string, { count: number; resetAt: number }>();
function keyFor(userId: number, route: string): string { return `${userId}:${route}`; }

export interface LimitResult { allowed: boolean; remaining: number; resetAt: number }
export const WINDOW_MS = 60_000;
export function check(userId: number, route: string): LimitResult { /* ... */ }
```

Same privacy, zero runtime wrapper, tree-shakeable, statically importable. This is the core argument of this entire document in miniature.

---

### Concept 3 — Nesting, dotted shorthand, and how deep nesting compiles

Namespaces nest arbitrarily, which is how large pre-module codebases simulated folders.

```ts
namespace Acme {
  export namespace Billing {
    export namespace Invoices {
      export interface Invoice {
        invoiceId: string;
        userId: number;
        totalCents: number;
        status: "draft" | "issued" | "paid" | "void";
      }

      export function isPayable(invoice: Invoice): boolean {
        return invoice.status === "issued";
      }
    }

    export namespace Payments {
      // Sibling namespaces refer to each other by their path from a common root:
      export function settle(invoice: Acme.Billing.Invoices.Invoice): void {
        if (!Acme.Billing.Invoices.isPayable(invoice)) {
          throw new Error(`Invoice ${invoice.invoiceId} is not payable`);
        }
      }

      // Inside the same enclosing namespace you can shorten the path:
      export function settleShort(invoice: Invoices.Invoice): void {
        if (!Invoices.isPayable(invoice)) throw new Error("not payable");
      }
    }
  }
}

const invoice: Acme.Billing.Invoices.Invoice = {
  invoiceId: "inv_001",
  userId: 42,
  totalCents: 9_900,
  status: "issued",
};
Acme.Billing.Payments.settle(invoice);
```

The dotted shorthand collapses the nesting visually but produces identical output:

```ts
// Equivalent to three nested `export namespace` blocks:
namespace Acme.Billing.Refunds {
  export interface Refund {
    refundId: string;
    invoiceId: string;
    amountCents: number;
  }
  export function isFullRefund(refund: Refund, invoiceTotal: number): boolean {
    return refund.amountCents === invoiceTotal;
  }
}
```

```js
// ── Emitted JavaScript for the dotted form ───────────────────────────────────
var Acme;
(function (Acme) {
  let Billing;
  (function (Billing) {
    let Refunds;
    (function (Refunds) {
      function isFullRefund(refund, invoiceTotal) {
        return refund.amountCents === invoiceTotal;
      }
      Refunds.isFullRefund = isFullRefund;
    })(Refunds = Billing.Refunds || (Billing.Refunds = {}));
  })(Billing = Acme.Billing || (Acme.Billing = {}));
})(Acme || (Acme = {}));

// Three nested IIFEs. Every property access `Acme.Billing.Refunds.isFullRefund`
// is three real object lookups. Deep namespace trees were a known perf annoyance
// in hot loops, which is why legacy code is full of local aliases:
//   var Refunds = Acme.Billing.Refunds;
```

TypeScript's own alias import is the typed version of that manual caching:

```ts
import Refunds = Acme.Billing.Refunds;      // alias — NOT an ES import
const r: Refunds.Refund = { refundId: "rf_1", invoiceId: "inv_1", amountCents: 500 };
Refunds.isFullRefund(r, 500);

// Emitted: `const Refunds = Acme.Billing.Refunds;`
// If the alias is only ever used in type positions, it is erased entirely.
```

---

### Concept 4 — Namespace merging across declarations and files

Two namespace blocks with the same name in the same scope **merge**. This is not a convenience — it is the mechanism that made multi-file namespace codebases possible at all.

```ts
// ── validation/strings.ts ────────────────────────────────────────────────────
namespace Validation {
  export function isEmail(value: string): boolean {
    return /^[^@\s]+@[^@\s]+\.[^@\s]+$/.test(value);
  }
}

// ── validation/numbers.ts ────────────────────────────────────────────────────
namespace Validation {
  export function isPositiveInt(value: number): boolean {
    return Number.isInteger(value) && value > 0;
  }
}

// ── validation/index.ts ──────────────────────────────────────────────────────
namespace Validation {
  // Members from OTHER blocks are visible here — the merged namespace is one scope:
  export function isUserId(value: unknown): value is number {
    return typeof value === "number" && isPositiveInt(value);
  }
}

Validation.isEmail("a@b.com");        // ✅ from strings.ts
Validation.isPositiveInt(42);         // ✅ from numbers.ts
Validation.isUserId(42);              // ✅ from index.ts — all one object
```

The rules of merging, precisely:

```ts
// RULE 1 — Only EXPORTED members merge into the shared surface.
namespace Settings {
  const secret = "a";                    // private to THIS block only
  export const publicA = "a";
}
namespace Settings {
  // console.log(secret);                // ❌ Cannot find name 'secret'.
  //                                     //    Non-exported members are block-local.
  export const publicB = "b";            // ✅
}
Settings.publicA;                        // ✅
Settings.publicB;                        // ✅

// RULE 2 — Same-named exports collide, unless they are mergeable themselves.
namespace Conflict {
  export const version = "1.0.0";
}
// namespace Conflict {
//   export const version = "2.0.0";     // ❌ Duplicate identifier 'version'.
// }

// Interfaces, being mergeable, are the exception:
namespace Contracts {
  export interface ApiResponse { statusCode: number }
}
namespace Contracts {
  export interface ApiResponse { requestId: string }   // ✅ interface merging
}
const res: Contracts.ApiResponse = { statusCode: 200, requestId: "req_1" };  // both fields

// RULE 3 — Nested namespaces merge recursively.
namespace Acme.Billing { export const currency = "USD" }
namespace Acme.Shipping { export const carrier = "ups" }
Acme.Billing.currency;                   // ✅
Acme.Shipping.carrier;                   // ✅ both live on the same Acme object

// RULE 4 — Merging is order-independent for TYPES, order-dependent for VALUES
//          at runtime, because the IIFEs execute in emit order.
namespace Boot {
  export const startedAt = Date.now();
}
namespace Boot {
  // At runtime this block runs AFTER the one above (same file, source order).
  export const uptimeAtLoad = Date.now() - startedAt;   // ✅ fine here
}
// Across files, emit order is decided by your build (file order / import graph).
// This was a classic source of `undefined` bugs in legacy codebases.
```

Critically, **merging only works in the global scope or within the same enclosing namespace**. Once a file becomes an ES module (it has a top-level `import` or `export`), its namespaces are module-scoped and merge only with namespaces in that same file.

---

### Concept 5 — Declaration merging: namespace + function, class, and enum

This is the feature that keeps namespaces alive. A namespace can merge with a *function*, a *class*, or an *enum* declared with the same name — letting you attach static properties that the type system knows about.

**Namespace + function** — the "function with properties" pattern:

```ts
// The function itself:
function createRequestId(prefix: string = "req"): string {
  createRequestId.counter += 1;
  return `${prefix}_${Date.now().toString(36)}_${createRequestId.counter}`;
}

// The namespace merges INTO the function, adding typed static members:
namespace createRequestId {
  export let counter = 0;
  export const DEFAULT_PREFIX = "req";

  export function reset(): void {
    counter = 0;
  }
}

createRequestId("order");        // "order_lx3k9_1"
createRequestId.counter;         // ✅ 1 — typed as number
createRequestId.reset();         // ✅ typed
createRequestId.DEFAULT_PREFIX;  // ✅ "req"

// ORDER MATTERS: the function declaration must come BEFORE the namespace,
// otherwise: "error TS2434: A namespace declaration cannot be located prior to
// a class or function with which it is merged."
```

```js
// ── Emitted JavaScript ───────────────────────────────────────────────────────
function createRequestId(prefix = "req") {
  createRequestId.counter += 1;
  return `${prefix}_${Date.now().toString(36)}_${createRequestId.counter}`;
}
(function (createRequestId) {
  createRequestId.counter = 0;
  createRequestId.DEFAULT_PREFIX = "req";
  function reset() { createRequestId.counter = 0; }
  createRequestId.reset = reset;
})(createRequestId || (createRequestId = {}));
// The IIFE bolts properties onto the function object. Exactly what jQuery,
// lodash and Express do in plain JS — now with types.
```

**Namespace + class** — static members plus nested types:

```ts
class ApiError extends Error {
  constructor(
    message: string,
    public readonly statusCode: number,
    public readonly code: ApiError.ErrorCode,
  ) {
    super(message);
    this.name = "ApiError";
  }
}

namespace ApiError {
  // A nested TYPE, referenced as ApiError.ErrorCode — impossible without merging:
  export type ErrorCode =
    | "VALIDATION_FAILED"
    | "UNAUTHORIZED"
    | "NOT_FOUND"
    | "CONFLICT"
    | "INTERNAL";

  export interface Serialized {
    error: { code: ErrorCode; message: string; statusCode: number };
  }

  // Named constructors as statics:
  export function notFound(resource: string): ApiError {
    return new ApiError(`${resource} not found`, 404, "NOT_FOUND");
  }

  export function unauthorized(reason = "Missing or invalid token"): ApiError {
    return new ApiError(reason, 401, "UNAUTHORIZED");
  }

  export function serialize(error: ApiError): Serialized {
    return { error: { code: error.code, message: error.message, statusCode: error.statusCode } };
  }
}

const err = ApiError.notFound(`Order 42`);
const code: ApiError.ErrorCode = err.code;              // the nested type
const body: ApiError.Serialized = ApiError.serialize(err);

// Note the double life of the name `ApiError`:
//   - as a TYPE:  `const e: ApiError` → an instance
//   - as a VALUE: `ApiError.notFound` → a static
//   - as a NAMESPACE: `ApiError.ErrorCode` → a nested type
// Three meanings, one identifier. Powerful, and confusing.
```

**Namespace + enum** — attaching helpers to an enum:

```ts
enum OrderStatus {
  Pending = "pending",
  Paid    = "paid",
  Shipped = "shipped",
}

namespace OrderStatus {
  const TERMINAL: readonly OrderStatus[] = [OrderStatus.Shipped];

  export function isTerminal(status: OrderStatus): boolean {
    return TERMINAL.includes(status);
  }

  export function label(status: OrderStatus): string {
    switch (status) {
      case OrderStatus.Pending: return "Awaiting payment";
      case OrderStatus.Paid:    return "Payment received";
      case OrderStatus.Shipped: return "In transit";
    }
  }
}

OrderStatus.Paid;                      // "paid" — an enum member
OrderStatus.isTerminal(OrderStatus.Paid);   // false — a namespace member
OrderStatus.label(OrderStatus.Shipped);     // "In transit"

// This is a genuinely neat trick — and also a double dose of non-erasable syntax.
// See doc 69 for why you probably shouldn't have the enum in the first place.
```

**What does NOT merge:**

```ts
// A namespace cannot merge with a `const`, `let`, or `var`:
const settings = { retries: 3 };
// namespace settings { export const timeout = 5000 }
// ❌ error TS2567: Enum declarations can only merge with namespace or other enum
//    declarations. / Duplicate identifier 'settings'.

// A namespace CAN merge with an interface or type alias, because they occupy
// different declaration spaces (type vs value):
interface Pagination { limit: number; offset: number }
namespace Pagination {
  export const DEFAULT_LIMIT = 50;
  export function normalise(input: Partial<Pagination>): Pagination {
    return { limit: input.limit ?? DEFAULT_LIMIT, offset: input.offset ?? 0 };
  }
}
const page: Pagination = Pagination.normalise({ offset: 100 });   // type + value
```

---

### Concept 6 — Ambient namespaces in `.d.ts` files

`declare namespace` describes a shape that already exists at runtime. It emits nothing and is the single most common namespace form you will meet.

```ts
// ── types/legacy-analytics.d.ts ──────────────────────────────────────────────
// Describing an old vendor script loaded via <script> that sets window.Analytics.

declare namespace LegacyAnalytics {
  // Inside `declare namespace`, `export` is implied — every member is public.

  interface TrackOptions {
    userId?: number;
    sessionId: string;
    timestampMs?: number;
  }

  interface AnalyticsEvent {
    name: string;
    properties: Record<string, string | number | boolean>;
  }

  type Environment = "development" | "staging" | "production";

  const version: string;
  const environment: Environment;

  function init(apiKey: string, environment: Environment): void;
  function track(event: AnalyticsEvent, options: TrackOptions): void;
  function identify(userId: number, traits: Record<string, unknown>): void;
  function flush(): Promise<void>;

  // Nested ambient namespace:
  namespace Errors {
    interface AnalyticsError extends Error {
      readonly retryable: boolean;
    }
    function isRetryable(error: unknown): error is AnalyticsError;
  }
}

declare const Analytics: typeof LegacyAnalytics;   // the actual global binding
```

```ts
// ── usage in your app, fully typed, no runtime import ────────────────────────
Analytics.init(process.env.ANALYTICS_KEY!, "production");

const event: LegacyAnalytics.AnalyticsEvent = {
  name: "order.placed",
  properties: { orderId: "ord_42", totalCents: 9900, currency: "USD" },
};

Analytics.track(event, { userId: 42, sessionId: "sess_abc" });

try {
  await Analytics.flush();
} catch (error) {
  if (LegacyAnalytics.Errors.isRetryable(error)) {
    // error: LegacyAnalytics.Errors.AnalyticsError
    console.warn("Analytics flush failed, will retry", error.retryable);
  }
}
```

Why a namespace here rather than a module? Because the runtime thing genuinely *is* a global object with a dotted structure. There is nothing to `import`. A namespace is the honest description of that reality.

`@types/node` is built on exactly this:

```ts
// ── Roughly what @types/node declares (simplified) ───────────────────────────
declare namespace NodeJS {
  interface ProcessEnv {
    [key: string]: string | undefined;
    NODE_ENV?: string;
  }

  interface Timeout {
    ref(): this;
    unref(): this;
    hasRef(): boolean;
    refresh(): this;
  }

  interface ErrnoException extends Error {
    errno?: number;
    code?: string;
    path?: string;
    syscall?: string;
  }

  interface ReadableStream { /* ... */ }
  interface WritableStream { /* ... */ }
}

// Which is why these types are spelled with a dot and need no import:
const timer: NodeJS.Timeout = setTimeout(() => {}, 1_000);
const env: NodeJS.ProcessEnv = process.env;

function onFsError(error: NodeJS.ErrnoException): void {
  if (error.code === "ENOENT") {
    console.warn(`Missing file: ${error.path}`);
  }
}
```

---

### Concept 7 — `declare global { namespace … }`: the augmentation you will actually write

Once a file has a top-level `import` or `export`, it is a module and its declarations are local. To touch the global namespace from inside a module you need `declare global`.

```ts
// ── src/types/express.d.ts ───────────────────────────────────────────────────

// This import makes the file a MODULE — required so `declare global` is legal,
// and required so we can reference imported types inside the augmentation.
import type { UserRole } from "../domain/roles.js";

declare global {
  namespace Express {
    // Express's own types declare `interface Request {}` inside `namespace Express`.
    // Our block MERGES with theirs — we are adding fields to the real Request.
    interface Request {
      requestId: string;
      startedAtMs: number;
      authToken?: {
        userId: number;
        role: UserRole;
        expiresAt: number;
      };
    }

    interface Response {
      /** Set by our response-timing middleware. */
      durationMs?: number;
    }

    // Passport-style: augmenting Express.User propagates to req.user
    interface User {
      userId: number;
      email: string;
      role: UserRole;
    }
  }

  namespace NodeJS {
    interface ProcessEnv {
      NODE_ENV: "development" | "test" | "production";
      PORT: string;
      DATABASE_URL: string;
      REDIS_URL: string;
      JWT_SECRET: string;
      LOG_LEVEL?: "debug" | "info" | "warn" | "error";
    }
  }
}

// A `.d.ts` with `import` at the top needs no explicit `export {}` because the
// import already made it a module. In a `.ts` file with no imports you MUST add
// `export {}` or `declare global` is an error:
//   error TS1046: Top-level declarations in .d.ts files must start with either
//   a 'declare' or 'export' modifier.
export {};
```

```ts
// ── src/middleware/auth.ts — the augmentation is now globally visible ────────
import type { Request, Response, NextFunction } from "express";
import { randomUUID } from "node:crypto";

export function requestContext(req: Request, res: Response, next: NextFunction): void {
  req.requestId = randomUUID();          // ✅ typed — thanks to the augmentation
  req.startedAtMs = Date.now();          // ✅
  res.setHeader("x-request-id", req.requestId);
  next();
}

export function requireAuth(req: Request, res: Response, next: NextFunction): void {
  const header = req.headers.authorization;
  if (!header?.startsWith("Bearer ")) {
    res.status(401).json({ error: "Missing bearer token" });
    return;
  }

  const decoded = verifyToken(header.slice(7), process.env.JWT_SECRET);
  //                                            ^^^^^^^^^^^^^^^^^^^^^
  // ✅ typed as `string`, not `string | undefined`, because of the
  //    NodeJS.ProcessEnv augmentation above. No `!` needed.

  req.authToken = decoded;               // ✅ typed
  next();
}

declare function verifyToken(token: string, secret: string): {
  userId: number;
  role: "admin" | "support" | "customer";
  expiresAt: number;
};
```

Two failure modes bite everyone exactly once:

```ts
// ❌ FAILURE 1 — the file is NOT a module, so `declare global` is illegal.
// src/types/express.ts with no import and no export:
// declare global { namespace Express { interface Request { userId: number } } }
// error TS2669: Augmentations for the global scope can only be directly nested
//               in external modules or ambient module declarations.
// FIX: add `export {};` at the top or bottom.

// ❌ FAILURE 2 — the file is never included in the compilation, so nothing happens.
// tsconfig.json must reach it:
// {
//   "include": ["src/**/*"],              // must cover src/types/*.d.ts
//   "compilerOptions": { "typeRoots": ["./node_modules/@types", "./src/types"] }
// }
// Symptom: `req.userId` is an error, the augmentation "does nothing", and you
// spend two hours on it. Check `tsc --listFiles | grep express.d.ts` first.
```

---

### Concept 8 — Namespaces vs ES modules: what actually replaced them

TypeScript's original docs called namespaces "internal modules" and ES modules "external modules" — the terminology alone tells you they were competing solutions to the same problem. ES modules won decisively.

```ts
// ── The namespace world ──────────────────────────────────────────────────────
// orderService.ts
namespace App.Services {
  export class OrderService {
    findById(orderId: number): Promise<App.Domain.Order | null> {
      // Depends on App.Domain, App.Db, App.Logging — none of it declared anywhere.
      return App.Db.queryOne<App.Domain.Order>("SELECT * FROM orders WHERE id = $1", [orderId]);
    }
  }
}

// How does the compiler know to load domain.ts before orderService.ts?
// It doesn't. You tell it, by hand:
/// <reference path="../domain/order.ts" />
/// <reference path="../db/client.ts" />

// And you tell the RUNTIME by hand too — either by `"outFile": "bundle.js"` with
// files listed in tsconfig order, or by ordering <script> tags in index.html.
```

```ts
// ── The module world ─────────────────────────────────────────────────────────
// orderService.ts
import { queryOne } from "../db/client.js";
import type { Order } from "../domain/order.js";
import type { Logger } from "../logging/logger.js";

export class OrderService {
  constructor(private readonly logger: Logger) {}

  findById(orderId: number): Promise<Order | null> {
    return queryOne<Order>("SELECT * FROM orders WHERE id = $1", [orderId]);
  }
}
// Dependencies are DECLARED. The compiler, the bundler, the test runner, the
// IDE, and Node itself all read the same graph. Load order is derived, not managed.
```

Point by point:

| Concern | Namespace | ES module |
|---|---|---|
| Dependency declaration | `/// <reference>` comments, by hand | `import` — machine-readable |
| Load order | Manual (`outFile` order, script tags) | Derived from the import graph |
| Encapsulation boundary | The `namespace` block | The file |
| Runtime output | IIFE + object per namespace | Nothing extra |
| Tree-shaking | Effectively none | Per-export |
| Lazy loading | Impossible | `await import("./heavy.js")` |
| Name collisions | Global — one `Validation` for the whole app | Per-file — rename on import |
| Circular deps | Silent `undefined` at runtime | Diagnosable, tooling warns |
| Works with esbuild/swc/Babel | ⚠️ per-file only, no cross-file merge | ✅ |
| `erasableSyntaxOnly` / node type-stripping | ❌ non-erasable | ✅ |
| Publishable to npm | ❌ pollutes consumers' globals | ✅ |

The killer argument is the last two rows. Just like `enum`, a namespace **with a body** is non-erasable syntax:

```ts
// ── With "erasableSyntaxOnly": true in tsconfig ──────────────────────────────
namespace Metrics {
  export function increment(name: string): void {}
}
// ❌ error TS1294: This syntax is not allowed when 'erasableSyntaxOnly' is enabled.

// ── Under Node's type stripping ──────────────────────────────────────────────
// $ node server.ts
// SyntaxError: TypeScript namespace declaration is not supported in strip-only mode

// ✅ These forms ARE allowed, because they emit nothing:
declare namespace ExternalSdk {              // ambient — fine
  function connect(url: string): void;
}

declare global {                             // global augmentation — fine
  namespace Express {
    interface Request { requestId: string }
  }
}

namespace TypesOnly {                        // ⚠️ emits nothing, but still flagged
  export interface UserDto { userId: number }
}
// erasableSyntaxOnly flags ANY non-declare namespace, even type-only ones,
// because whether it emits depends on its contents — too subtle for a syntax rule.
// In a .d.ts file it is fine; in a .ts file, prefer exported types.
```

And with `isolatedModules` (implied by `verbatimModuleSyntax`, and on by default in Vite, Next.js, tsx, Bun and Deno), cross-file namespace merging simply cannot work — each file is transpiled alone, so a second `namespace Validation { … }` block in another file emits its own independent `var Validation` in its own module scope.

---

### Concept 9 — Where namespaces still legitimately appear in 2026

Four places. Learn to recognise them; do not invent a fifth.

**1. Global augmentation of third-party globals** — already covered, and by far the most common.

```ts
// Express, Passport, Jest, Cypress, Fastify (via module augmentation), NodeJS.*
declare global {
  namespace Express { interface Request { tenantId: string } }
  namespace NodeJS  { interface ProcessEnv { STRIPE_SECRET_KEY: string } }
  namespace jest    { interface Matchers<R> { toBeValidUserId(): R } }
}
export {};
```

**2. `.d.ts` files describing a global (non-module) JavaScript runtime object.**

```ts
// A vendor script that sets window.PaymentWidget with a dotted API:
declare namespace PaymentWidget {
  interface MountOptions { containerId: string; publicKey: string; currency: "USD" | "EUR" }
  interface PaymentResult { paymentId: string; status: "succeeded" | "failed" }
  function mount(options: MountOptions): void;
  function onComplete(handler: (result: PaymentResult) => void): void;
}
```

**3. Grouping types under a single exported name in a library's public API.**

```ts
// A pattern popularised by libraries like Stripe's SDK and AWS SDK v2:
export declare namespace StripeWebhook {
  type EventType =
    | "payment_intent.succeeded"
    | "payment_intent.payment_failed"
    | "customer.subscription.deleted";

  interface Event<T = unknown> {
    id: string;
    type: EventType;
    created: number;
    data: { object: T };
  }

  interface PaymentIntent {
    id: string;
    amount: number;
    currency: string;
    status: "succeeded" | "requires_payment_method" | "canceled";
  }
}

// Consumers get a tidy dotted API with a single import:
// import type { StripeWebhook } from "@acme/billing";
// function handle(event: StripeWebhook.Event<StripeWebhook.PaymentIntent>) { … }
//
// ⚠️ This is a taste call. The alternative — exporting the types flat and letting
// consumers write `import * as StripeWebhook from "…"` — is equally good and
// fully erasable. Prefer it unless you specifically want the grouping baked in.
```

**4. Merging statics/nested types onto a class or function you export.**

```ts
export class Money {
  private constructor(public readonly cents: number, public readonly currency: Money.Currency) {}

  static fromCents(cents: number, currency: Money.Currency): Money {
    return new Money(cents, currency);
  }

  add(other: Money): Money {
    if (other.currency !== this.currency) {
      throw new Error(`Currency mismatch: ${this.currency} vs ${other.currency}`);
    }
    return new Money(this.cents + other.cents, this.currency);
  }
}

export namespace Money {
  export type Currency = "USD" | "EUR" | "GBP";
  export const ZERO_USD = Money.fromCents(0, "USD");
}
// `Money.Currency` as a type and `Money.fromCents` as a value, from one import.
// ⚠️ Non-erasable (the namespace has a runtime member). If you only need the
// TYPE nested, put `export namespace Money { export type Currency = … }` in a
// .d.ts, or just export `MoneyCurrency` flat.
```

Everywhere else: **use a module file.**

---

### Concept 10 — Reading and migrating legacy namespace code

Here is a realistic legacy file and its module-based rewrite, step by step.

```ts
// ═════ BEFORE — src/services/orders.ts (legacy, no imports anywhere) ═════════
/// <reference path="../domain/order.ts" />
/// <reference path="../db/client.ts" />
/// <reference path="../logging/logger.ts" />

namespace App.Services {
  import Order = App.Domain.Order;            // alias to shorten the path
  import Db = App.Infra.Db;

  const RETRY_LIMIT = 3;                      // private to this block

  export interface OrderFilters {
    userId?: number;
    status?: Order.Status;
    createdAfter?: Date;
  }

  export class OrderService {
    constructor(private readonly logger: App.Infra.Logger) {}

    async findById(orderId: number): Promise<Order.Order | null> {
      return Db.queryOne<Order.Order>("SELECT * FROM orders WHERE id = $1", [orderId]);
    }

    async list(filters: OrderFilters): Promise<Order.Order[]> {
      const clauses: string[] = [];
      const params: unknown[] = [];
      if (filters.userId !== undefined) {
        params.push(filters.userId);
        clauses.push(`user_id = $${params.length}`);
      }
      if (filters.status !== undefined) {
        params.push(filters.status);
        clauses.push(`status = $${params.length}`);
      }
      const where = clauses.length ? `WHERE ${clauses.join(" AND ")}` : "";
      return Db.query<Order.Order>(`SELECT * FROM orders ${where}`, params);
    }

    private async withRetry<T>(operation: () => Promise<T>): Promise<T> {
      let lastError: unknown;
      for (let attempt = 1; attempt <= RETRY_LIMIT; attempt++) {
        try {
          return await operation();
        } catch (error) {
          lastError = error;
          this.logger.warn(`Attempt ${attempt}/${RETRY_LIMIT} failed`);
        }
      }
      throw lastError;
    }
  }
}
```

The migration, in the order you should actually do it:

```ts
// ── STEP 1 — Delete the /// <reference> lines. Add real imports. ─────────────
// ── STEP 2 — Delete the `namespace App.Services {` wrapper and its brace. ────
// ── STEP 3 — Dedent everything one level. ───────────────────────────────────
// ── STEP 4 — `export` on the members that were exported; drop it on private
//             ones (file scope already makes them private). ──────────────────
// ── STEP 5 — Replace `import X = A.B.C` aliases with ES imports. ────────────
// ── STEP 6 — Fix references: `App.Domain.Order` → the imported name. ────────

// ═════ AFTER — src/services/orderService.ts ══════════════════════════════════
import { queryOne, query } from "../db/client.js";
import type { Order, OrderStatus } from "../domain/order.js";
import type { Logger } from "../logging/logger.js";

const RETRY_LIMIT = 3;                        // file-private, no `export`

export interface OrderFilters {
  userId?: number;
  status?: OrderStatus;
  createdAfter?: Date;
}

export class OrderService {
  constructor(private readonly logger: Logger) {}

  async findById(orderId: number): Promise<Order | null> {
    return queryOne<Order>("SELECT * FROM orders WHERE id = $1", [orderId]);
  }

  async list(filters: OrderFilters): Promise<Order[]> {
    const clauses: string[] = [];
    const params: unknown[] = [];
    if (filters.userId !== undefined) {
      params.push(filters.userId);
      clauses.push(`user_id = $${params.length}`);
    }
    if (filters.status !== undefined) {
      params.push(filters.status);
      clauses.push(`status = $${params.length}`);
    }
    const where = clauses.length ? `WHERE ${clauses.join(" AND ")}` : "";
    return query<Order>(`SELECT * FROM orders ${where}`, params);
  }

  private async withRetry<T>(operation: () => Promise<T>): Promise<T> {
    let lastError: unknown;
    for (let attempt = 1; attempt <= RETRY_LIMIT; attempt++) {
      try {
        return await operation();
      } catch (error) {
        lastError = error;
        this.logger.warn(`Attempt ${attempt}/${RETRY_LIMIT} failed`);
      }
    }
    throw lastError;
  }
}
```

Practical migration tactics for a large codebase:

```ts
// TACTIC 1 — Migrate leaves first (files nothing depends on), then work upward.

// TACTIC 2 — Keep the old dotted API alive during the transition with a
//            compatibility barrel, so call sites can change gradually:
// src/compat/App.ts
import * as OrderServiceModule from "../services/orderService.js";
import * as UserServiceModule from "../services/userService.js";

export const App = {
  Services: {
    ...OrderServiceModule,
    ...UserServiceModule,
  },
} as const;
// Existing `App.Services.OrderService` call sites keep working while you convert
// them file by file. Delete this barrel when the last one is gone.

// TACTIC 3 — For type-only namespaces, `import * as` is the drop-in replacement:
// BEFORE:  const dto: Api.V2.UserDto = …
// AFTER:   import type * as ApiV2 from "./api/v2/types.js";
//          const dto: ApiV2.UserDto = …
// Identical ergonomics, zero runtime, fully erasable.

// TACTIC 4 — Turn on the compiler flags that make regression impossible:
// tsconfig.json
// {
//   "compilerOptions": {
//     "module": "nodenext",
//     "isolatedModules": true,
//     "verbatimModuleSyntax": true,
//     "erasableSyntaxOnly": true     // ← now a new namespace is a build error
//   }
// }

// TACTIC 5 — ESLint enforcement:
// "@typescript-eslint/no-namespace": ["error", {
//    "allowDeclarations": true,        // permits `declare namespace` in .d.ts
//    "allowDefinitionFiles": true      // permits namespaces in .d.ts files
// }]
```

---

## Example 1 — basic

```ts
// ── The same small utility library, written three ways ───────────────────────

// ═════ Version A — namespace (the 2013 way) ══════════════════════════════════

namespace HttpUtils {
  // Private — lives in the IIFE closure, never attached to the object.
  const STATUS_TEXT: Record<number, string> = {
    200: "OK",
    201: "Created",
    400: "Bad Request",
    401: "Unauthorized",
    404: "Not Found",
    409: "Conflict",
    422: "Unprocessable Entity",
    500: "Internal Server Error",
  };

  export type StatusCategory =
    | "informational"
    | "success"
    | "redirect"
    | "client_error"
    | "server_error";

  export interface ApiResponse<T> {
    statusCode: number;
    statusText: string;
    body: T;
    requestId: string;
  }

  export function statusText(statusCode: number): string {
    return STATUS_TEXT[statusCode] ?? "Unknown";
  }

  export function categorise(statusCode: number): StatusCategory {
    if (statusCode < 100 || statusCode > 599) {
      throw new RangeError(`Invalid HTTP status code: ${statusCode}`);
    }
    if (statusCode < 200) return "informational";
    if (statusCode < 300) return "success";
    if (statusCode < 400) return "redirect";
    if (statusCode < 500) return "client_error";
    return "server_error";
  }

  export function ok<T>(body: T, requestId: string): ApiResponse<T> {
    return { statusCode: 200, statusText: statusText(200), body, requestId };
  }

  export function isRetryable(statusCode: number): boolean {
    return categorise(statusCode) === "server_error" || statusCode === 429;
  }
}

// Usage — no import, global reach, dotted access:
const response: HttpUtils.ApiResponse<{ userId: number }> = HttpUtils.ok(
  { userId: 42 },
  "req_abc123",
);
console.log(HttpUtils.categorise(response.statusCode));   // "success"
console.log(HttpUtils.isRetryable(503));                  // true
// console.log(HttpUtils.STATUS_TEXT);                    // ❌ private

// ═════ Version B — the same thing as an ES module (the right way) ════════════
// File: src/http/httpUtils.ts

const STATUS_TEXT: Record<number, string> = {
  200: "OK", 201: "Created", 400: "Bad Request", 401: "Unauthorized",
  404: "Not Found", 409: "Conflict", 422: "Unprocessable Entity",
  500: "Internal Server Error",
};

export type StatusCategory =
  | "informational" | "success" | "redirect" | "client_error" | "server_error";

export interface ApiResponse<T> {
  statusCode: number;
  statusText: string;
  body: T;
  requestId: string;
}

export function statusText(statusCode: number): string {
  return STATUS_TEXT[statusCode] ?? "Unknown";
}

export function categorise(statusCode: number): StatusCategory {
  if (statusCode < 100 || statusCode > 599) {
    throw new RangeError(`Invalid HTTP status code: ${statusCode}`);
  }
  if (statusCode < 200) return "informational";
  if (statusCode < 300) return "success";
  if (statusCode < 400) return "redirect";
  if (statusCode < 500) return "client_error";
  return "server_error";
}

export function ok<T>(body: T, requestId: string): ApiResponse<T> {
  return { statusCode: 200, statusText: statusText(200), body, requestId };
}

export function isRetryable(statusCode: number): boolean {
  return categorise(statusCode) === "server_error" || statusCode === 429;
}

// File: src/routes/users.ts
// import { ok, categorise, isRetryable, type ApiResponse } from "../http/httpUtils.js";
//
// Or, if you want the dotted feel back:
// import * as HttpUtils from "../http/httpUtils.js";
// HttpUtils.categorise(200);
//
// The `import * as` form gives you namespace ergonomics with module semantics.
// This is the answer to "but I like the HttpUtils.x style".

// ═════ Version C — a legitimate namespace: augmenting a global ═══════════════
// File: src/types/global.d.ts

declare global {
  namespace NodeJS {
    interface ProcessEnv {
      NODE_ENV: "development" | "test" | "production";
      PORT: string;
      DATABASE_URL: string;
    }
  }
}
export {};

// Now, anywhere in the app:
// const port = Number(process.env.PORT);            // PORT is `string`, not `string | undefined`
// if (process.env.NODE_ENV === "prodution") { }     // ❌ typo caught at compile time
```

---

## Example 2 — real world backend use case

```ts
// ═══════════════════════════════════════════════════════════════════════════
// A production Express + Node service showing every place a namespace still
// belongs — and every place it does not.
// ═══════════════════════════════════════════════════════════════════════════

// ════════════════════════════════════════════════════════════════════════════
// FILE 1 — src/domain/roles.ts   (a plain module — NO namespace)
// ════════════════════════════════════════════════════════════════════════════

export const USER_ROLE = {
  Owner:    "owner",
  Admin:    "admin",
  Support:  "support",
  Customer: "customer",
} as const;

export type UserRole = typeof USER_ROLE[keyof typeof USER_ROLE];

export function isUserRole(value: unknown): value is UserRole {
  return typeof value === "string"
      && (Object.values(USER_ROLE) as string[]).includes(value);
}

// ════════════════════════════════════════════════════════════════════════════
// FILE 2 — src/types/express.d.ts   (namespace: GLOBAL AUGMENTATION ✅)
// ════════════════════════════════════════════════════════════════════════════

import type { UserRole } from "../domain/roles.js";

declare global {
  // `Express` is a namespace that @types/express declares in the global scope.
  // Our block merges into it. Express's own `Request` type extends
  // `Express.Request`, so anything we add here lands on `req`.
  namespace Express {
    interface Request {
      /** Set by the requestContext middleware — always present downstream. */
      requestId: string;
      /** High-resolution start time, for the response-timing middleware. */
      startedAtNs: bigint;
      /** Populated by requireAuth; absent on public routes. */
      authToken?: AuthToken;
      /** Populated by the tenant resolver. */
      tenantId?: string;
    }

    interface Response {
      /** Set by responseTiming after the response is flushed. */
      durationMs?: number;
    }

    /** Passport writes `req.user`; typing it here types it everywhere. */
    interface User {
      userId: number;
      email: string;
      role: UserRole;
    }
  }

  // `NodeJS` is @types/node's global namespace. Augmenting ProcessEnv turns
  // every `process.env.X` access from `string | undefined` into `string`.
  namespace NodeJS {
    interface ProcessEnv {
      NODE_ENV: "development" | "test" | "production";
      PORT: string;
      DATABASE_URL: string;
      REDIS_URL: string;
      JWT_SECRET: string;
      STRIPE_WEBHOOK_SECRET: string;
      LOG_LEVEL?: "debug" | "info" | "warn" | "error";
    }
  }
}

interface AuthToken {
  userId: number;
  role: UserRole;
  issuedAtMs: number;
  expiresAtMs: number;
}

export {};   // ← makes this a module so `declare global` is legal

// ════════════════════════════════════════════════════════════════════════════
// FILE 3 — src/types/legacy-fraud-sdk.d.ts
//          (namespace: DESCRIBING A GLOBAL VENDOR SCRIPT ✅)
// ════════════════════════════════════════════════════════════════════════════

// An old fraud-scoring SDK loaded via a <script> tag on our checkout page and
// via a preload script in our Node worker. It sets a global `FraudSdk` object.
// There is no npm package, no module — a namespace is the honest description.

declare namespace FraudSdk {
  type RiskLevel = "low" | "medium" | "high" | "blocked";

  interface ScoreRequest {
    userId: number;
    orderId: string;
    amountCents: number;
    currency: "USD" | "EUR" | "GBP";
    ipAddress: string;
    deviceFingerprint?: string;
  }

  interface ScoreResult {
    score: number;                 // 0–100
    level: RiskLevel;
    reasons: readonly string[];
    evaluatedAtMs: number;
  }

  interface Config {
    apiKey: string;
    timeoutMs?: number;
    environment: "sandbox" | "live";
  }

  const version: string;

  function configure(config: Config): void;
  function score(request: ScoreRequest): Promise<ScoreResult>;
  function shutdown(): Promise<void>;

  namespace Errors {
    interface FraudSdkError extends Error {
      readonly code: "TIMEOUT" | "RATE_LIMITED" | "INVALID_KEY" | "UNKNOWN";
      readonly retryable: boolean;
    }
    function isFraudSdkError(error: unknown): error is FraudSdkError;
  }
}

declare const FraudSdk: typeof FraudSdk;

// ════════════════════════════════════════════════════════════════════════════
// FILE 4 — src/errors/apiError.ts
//          (namespace: MERGED ONTO A CLASS for nested types + statics ⚠️)
// ════════════════════════════════════════════════════════════════════════════

export class ApiError extends Error {
  readonly statusCode: number;
  readonly code: ApiError.Code;
  readonly details?: readonly ApiError.FieldIssue[];

  constructor(
    message: string,
    statusCode: number,
    code: ApiError.Code,
    details?: readonly ApiError.FieldIssue[],
  ) {
    super(message);
    this.name = "ApiError";
    this.statusCode = statusCode;
    this.code = code;
    this.details = details;
    Error.captureStackTrace?.(this, ApiError);
  }

  toJSON(): ApiError.Serialized {
    return {
      error: {
        code: this.code,
        message: this.message,
        statusCode: this.statusCode,
        ...(this.details ? { details: this.details } : {}),
      },
    };
  }
}

export namespace ApiError {
  // Nested TYPES — the reason this merge exists at all:
  export type Code =
    | "VALIDATION_FAILED"
    | "UNAUTHORIZED"
    | "FORBIDDEN"
    | "NOT_FOUND"
    | "CONFLICT"
    | "RATE_LIMITED"
    | "INTERNAL";

  export interface FieldIssue {
    field: string;
    message: string;
  }

  export interface Serialized {
    error: {
      code: Code;
      message: string;
      statusCode: number;
      details?: readonly FieldIssue[];
    };
  }

  // Named constructors — VALUES, which is what makes this non-erasable:
  export function validation(issues: readonly FieldIssue[]): ApiError {
    return new ApiError("Request validation failed", 422, "VALIDATION_FAILED", issues);
  }

  export function unauthorized(reason = "Missing or invalid token"): ApiError {
    return new ApiError(reason, 401, "UNAUTHORIZED");
  }

  export function forbidden(action: string): ApiError {
    return new ApiError(`Not permitted: ${action}`, 403, "FORBIDDEN");
  }

  export function notFound(resource: string, id: string | number): ApiError {
    return new ApiError(`${resource} ${id} not found`, 404, "NOT_FOUND");
  }

  export function conflict(message: string): ApiError {
    return new ApiError(message, 409, "CONFLICT");
  }

  export function is(error: unknown): error is ApiError {
    return error instanceof ApiError;
  }
}

// ⚠️ HONEST NOTE: this file will NOT compile under `erasableSyntaxOnly: true`,
// because the namespace contains runtime members. The erasable rewrite is:
//
//   export type ApiErrorCode = "VALIDATION_FAILED" | …;
//   export interface FieldIssue { … }
//   export class ApiError extends Error {
//     static validation(issues: readonly FieldIssue[]): ApiError { … }
//     static unauthorized(reason?: string): ApiError { … }
//   }
//
// Flat names, static methods, zero namespace. Slightly less pretty at call
// sites, fully erasable, and what a new codebase should do.

// ════════════════════════════════════════════════════════════════════════════
// FILE 5 — src/middleware/context.ts   (plain module, consuming the augmentation)
// ════════════════════════════════════════════════════════════════════════════

import type { Request, Response, NextFunction } from "express";
import { randomUUID } from "node:crypto";
import { ApiError } from "../errors/apiError.js";
import { isUserRole, type UserRole } from "../domain/roles.js";

export function requestContext(req: Request, res: Response, next: NextFunction): void {
  req.requestId = randomUUID();          // ✅ typed by our Express.Request augmentation
  req.startedAtNs = process.hrtime.bigint();
  res.setHeader("x-request-id", req.requestId);

  res.on("finish", () => {
    const elapsedNs = process.hrtime.bigint() - req.startedAtNs;
    res.durationMs = Number(elapsedNs) / 1_000_000;   // ✅ typed on Express.Response
  });

  next();
}

interface DecodedJwt {
  sub: string;
  role: string;
  iat: number;
  exp: number;
}

declare function verifyJwt(token: string, secret: string): DecodedJwt;

export function requireAuth(req: Request, _res: Response, next: NextFunction): void {
  const header = req.headers.authorization;
  if (!header?.startsWith("Bearer ")) {
    next(ApiError.unauthorized());
    return;
  }

  let decoded: DecodedJwt;
  try {
    // process.env.JWT_SECRET is `string` (not `string | undefined`) thanks to
    // the NodeJS.ProcessEnv augmentation in FILE 2. No `!`, no fallback.
    decoded = verifyJwt(header.slice(7), process.env.JWT_SECRET);
  } catch {
    next(ApiError.unauthorized("Token signature invalid or expired"));
    return;
  }

  if (!isUserRole(decoded.role)) {
    next(ApiError.unauthorized(`Unknown role in token: ${decoded.role}`));
    return;
  }

  req.authToken = {
    userId: Number(decoded.sub),
    role: decoded.role,
    issuedAtMs: decoded.iat * 1_000,
    expiresAtMs: decoded.exp * 1_000,
  };

  next();
}

export function requireRole(...allowed: readonly UserRole[]) {
  return (req: Request, _res: Response, next: NextFunction): void => {
    if (!req.authToken) {
      next(ApiError.unauthorized());
      return;
    }
    if (!allowed.includes(req.authToken.role)) {
      next(ApiError.forbidden(`requires one of: ${allowed.join(", ")}`));
      return;
    }
    next();
  };
}

// ════════════════════════════════════════════════════════════════════════════
// FILE 6 — src/services/checkoutService.ts
//          (plain module, using the ambient FraudSdk namespace)
// ════════════════════════════════════════════════════════════════════════════

import { ApiError } from "../errors/apiError.js";

export interface CheckoutRequest {
  userId: number;
  orderId: string;
  amountCents: number;
  currency: "USD" | "EUR" | "GBP";
  ipAddress: string;
}

export interface CheckoutDecision {
  approved: boolean;
  riskLevel: FraudSdk.RiskLevel;      // ← ambient namespace type, no import needed
  riskScore: number;
  reasons: readonly string[];
}

export class CheckoutService {
  private readonly maxScore: number;

  constructor(maxScore = 70) {
    this.maxScore = maxScore;
    FraudSdk.configure({
      apiKey: process.env.STRIPE_WEBHOOK_SECRET,   // ✅ typed as string
      environment: process.env.NODE_ENV === "production" ? "live" : "sandbox",
      timeoutMs: 2_000,
    });
  }

  async evaluate(request: CheckoutRequest): Promise<CheckoutDecision> {
    let result: FraudSdk.ScoreResult;

    try {
      result = await FraudSdk.score({
        userId: request.userId,
        orderId: request.orderId,
        amountCents: request.amountCents,
        currency: request.currency,
        ipAddress: request.ipAddress,
      });
    } catch (error) {
      // Nested ambient namespace — FraudSdk.Errors.isFraudSdkError narrows:
      if (FraudSdk.Errors.isFraudSdkError(error) && error.retryable) {
        // Fail open on transient fraud-service errors, but flag it.
        return {
          approved: true,
          riskLevel: "medium",
          riskScore: 50,
          reasons: [`fraud_service_${error.code.toLowerCase()}`],
        };
      }
      throw ApiError.conflict("Fraud evaluation failed");
    }

    return {
      approved: result.level !== "blocked" && result.score <= this.maxScore,
      riskLevel: result.level,
      riskScore: result.score,
      reasons: result.reasons,
    };
  }
}

// ════════════════════════════════════════════════════════════════════════════
// FILE 7 — src/routes/checkout.ts   (plain module, everything wired together)
// ════════════════════════════════════════════════════════════════════════════

import { Router, type Request, type Response, type NextFunction } from "express";
import { requireAuth, requireRole } from "../middleware/context.js";
import { CheckoutService } from "../services/checkoutService.js";
import { ApiError } from "../errors/apiError.js";
import { USER_ROLE } from "../domain/roles.js";

const checkoutService = new CheckoutService();
export const checkoutRouter = Router();

interface CheckoutRequestBody {
  orderId?: unknown;
  amountCents?: unknown;
  currency?: unknown;
}

function parseCheckoutBody(requestBody: unknown): {
  orderId: string;
  amountCents: number;
  currency: "USD" | "EUR" | "GBP";
} {
  const body = (requestBody ?? {}) as CheckoutRequestBody;
  const issues: ApiError.FieldIssue[] = [];   // ← nested type from the merged namespace

  if (typeof body.orderId !== "string" || body.orderId.length === 0) {
    issues.push({ field: "orderId", message: "must be a non-empty string" });
  }
  if (typeof body.amountCents !== "number" || !Number.isInteger(body.amountCents) || body.amountCents <= 0) {
    issues.push({ field: "amountCents", message: "must be a positive integer" });
  }
  if (body.currency !== "USD" && body.currency !== "EUR" && body.currency !== "GBP") {
    issues.push({ field: "currency", message: "must be USD, EUR or GBP" });
  }

  if (issues.length > 0) throw ApiError.validation(issues);

  return {
    orderId: body.orderId as string,
    amountCents: body.amountCents as number,
    currency: body.currency as "USD" | "EUR" | "GBP",
  };
}

checkoutRouter.post(
  "/checkout",
  requireAuth,
  requireRole(USER_ROLE.Customer, USER_ROLE.Admin),
  async (req: Request, res: Response, next: NextFunction) => {
    try {
      const { orderId, amountCents, currency } = parseCheckoutBody(req.body);

      const decision = await checkoutService.evaluate({
        userId: req.authToken!.userId,      // ✅ typed by the Express augmentation
        orderId,
        amountCents,
        currency,
        ipAddress: req.ip ?? "0.0.0.0",
      });

      if (!decision.approved) {
        throw ApiError.forbidden(
          `checkout blocked (risk ${decision.riskLevel}, score ${decision.riskScore})`,
        );
      }

      res.status(200).json({
        ok: true,
        requestId: req.requestId,           // ✅ typed by the Express augmentation
        data: decision,
      });
    } catch (error) {
      next(error);
    }
  },
);

// ════════════════════════════════════════════════════════════════════════════
// FILE 8 — src/middleware/errorHandler.ts
// ════════════════════════════════════════════════════════════════════════════

import type { Request, Response, NextFunction } from "express";
import { ApiError as AppError } from "../errors/apiError.js";

export function errorHandler(
  error: unknown,
  req: Request,
  res: Response,
  _next: NextFunction,
): void {
  if (AppError.is(error)) {
    res.status(error.statusCode).json({ ...error.toJSON(), requestId: req.requestId });
    return;
  }

  // NodeJS.ErrnoException — another ambient namespace type from @types/node:
  const errno = error as NodeJS.ErrnoException;
  if (errno.code === "ECONNREFUSED" || errno.code === "ETIMEDOUT") {
    res.status(503).json({
      error: { code: "INTERNAL", message: "Upstream unavailable", statusCode: 503 },
      requestId: req.requestId,
    });
    return;
  }

  res.status(500).json({
    error: { code: "INTERNAL", message: "Internal server error", statusCode: 500 },
    requestId: req.requestId,
  });
}

// ═══════════════════════════════════════════════════════════════════════════
// SCORECARD for this service:
//   Namespaces used:  3
//     - declare global { namespace Express }    → unavoidable, correct
//     - declare global { namespace NodeJS }     → unavoidable, correct
//     - declare namespace FraudSdk              → correct (real global object)
//     - export namespace ApiError               → optional; flat statics are better
//   Namespaces NOT used:  every service, route, middleware, and domain file.
// ═══════════════════════════════════════════════════════════════════════════
```

---

## Going deeper

### 1. `namespace` vs the legacy `module` keyword

```ts
// TypeScript originally used `module` for what is now `namespace`:
module LegacyStyle {                 // ⚠️ deprecated spelling
  export const version = "1.0.0";
}

// It was renamed in TypeScript 1.5 to stop the collision with ES modules:
namespace ModernStyle {              // ✅ identical semantics
  export const version = "1.0.0";
}

// `module` with a STRING literal name means something different — an ambient
// module declaration, which is still current and NOT deprecated:
declare module "legacy-image-resizer" {
  export interface ResizeOptions { width: number; height: number; quality?: number }
  export function resize(input: Buffer, options: ResizeOptions): Promise<Buffer>;
}

// And in a .d.ts, `declare module "x"` is how you type an untyped npm package.
// See doc 50 — Ambient declarations.
```

### 2. `export =` and `import x = require(...)`

```ts
// The CommonJS interop pair — also non-erasable, also namespace-adjacent.

// legacyLogger.d.ts
declare namespace LegacyLogger {
  interface LogFields { requestId?: string; userId?: number }
  function info(message: string, fields?: LogFields): void;
  function error(message: string, fields?: LogFields): void;
}
export = LegacyLogger;              // "this module's export IS this namespace"

// consumer.ts
import LegacyLogger = require("./legacyLogger");   // the matching import form
LegacyLogger.info("boot complete", { requestId: "req_1" });

// Under "esModuleInterop": true you can usually write:
import LegacyLogger from "./legacyLogger";
// ...but `export =` remains the accurate description of a CommonJS module whose
// `module.exports` is a function or object rather than a namespace of exports.

// ⚠️ `import x = require()` is flagged by erasableSyntaxOnly. In a .d.ts it's fine.
```

### 3. Namespaces inside modules: legal, and almost always a mistake

```ts
// orderTypes.ts — this file has an `export`, so it IS a module.
export namespace OrderTypes {
  export interface Order { orderId: string; totalCents: number }
  export type Status = "draft" | "paid";
}

// Consumers must now double-dot:
// import { OrderTypes } from "./orderTypes.js";
// const o: OrderTypes.Order = …;
//
// Or worse, if they use a namespace import:
// import * as Types from "./orderTypes.js";
// const o: Types.OrderTypes.Order = …;      ← "the double namespace" smell

// ✅ Just export flat. The file already IS the namespace.
export interface Order { orderId: string; totalCents: number }
export type OrderStatus = "draft" | "paid";
// Consumers choose their own grouping:
//   import * as OrderTypes from "./orderTypes.js";   → OrderTypes.Order
//   import type { Order } from "./orderTypes.js";    → Order
// Both work. Zero runtime. This is strictly better.
```

### 4. Cross-file merging silently dies under `isolatedModules`

```ts
// ── With tsc and NO modules in these files (global scripts) ──────────────────
// metrics/counters.ts
namespace Metrics { export function counter(name: string): void {} }
// metrics/histograms.ts
namespace Metrics { export function histogram(name: string): void {} }
// ✅ tsc merges them: one global `Metrics` object with both functions.

// ── The moment either file gains an import/export ───────────────────────────
// metrics/counters.ts
import { config } from "../config.js";                 // ← now a module
namespace Metrics { export function counter(name: string): void {} }
export { Metrics };
// ❌ The two Metrics no longer merge. They are two separate module-local
//    namespaces that happen to share a name. Nothing errors — the second file's
//    functions are just silently missing from the first file's object.

// ── Under esbuild / swc / babel (isolatedModules) ───────────────────────────
// Cross-file merging is IMPOSSIBLE in principle: each file is transpiled with
// no knowledge of the others, so each emits its own `var Metrics; (function… )`.
// In a bundle they may or may not merge depending on scope hoisting. Do not rely on it.
```

### 5. Namespaces do not tree-shake, and cannot be lazy-loaded

```ts
// ── Namespace: all-or-nothing ────────────────────────────────────────────────
namespace ReportGenerators {
  export function csv(rows: readonly Record<string, unknown>[]): string { /* 50 lines */ return ""; }
  export function pdf(rows: readonly Record<string, unknown>[]): Buffer { /* 900 lines + a heavy dep */ return Buffer.alloc(0); }
  export function xlsx(rows: readonly Record<string, unknown>[]): Buffer { /* 600 lines */ return Buffer.alloc(0); }
}
ReportGenerators.csv([]);   // you used 1 of 3 — you ship all 3, plus the pdf dep

// ── Modules: pay for what you use ────────────────────────────────────────────
// reports/csv.ts, reports/pdf.ts, reports/xlsx.ts
import { toCsv } from "./reports/csv.js";        // only csv.ts is bundled
toCsv([]);

// ── And lazy loading is a one-liner ──────────────────────────────────────────
async function exportReport(format: "csv" | "pdf" | "xlsx", rows: readonly Record<string, unknown>[]) {
  switch (format) {
    case "csv":  return (await import("./reports/csv.js")).toCsv(rows);
    case "pdf":  return (await import("./reports/pdf.js")).toPdf(rows);   // loaded on demand
    case "xlsx": return (await import("./reports/xlsx.js")).toXlsx(rows);
  }
}
// There is no namespace equivalent. A namespace is defined at parse time,
// in one scope, forever.
```

### 6. Namespaces are global — the collision problem

```ts
// ── Two libraries, both from 2014, both helpful ──────────────────────────────
// node_modules/@types/lib-a/index.d.ts
declare namespace Utils {
  function formatDate(date: Date): string;
}
// node_modules/@types/lib-b/index.d.ts
declare namespace Utils {
  function formatDate(date: Date, locale: string): string;
  //       ^ ❌ error TS2393: Duplicate function implementation
  //         (or worse: they silently merge into an overload set that matches neither)
}

// You cannot rename either one. You cannot scope them. Your only options are
// `skipLibCheck: true` (hiding the problem) or dropping a library.

// With modules, the same names cost you nothing:
import { formatDate as formatDateA } from "lib-a";
import { formatDate as formatDateB } from "lib-b";
// Every consumer picks their own local names. Collisions are structurally impossible.
```

### 7. Type-only namespaces: the least-bad remaining use in `.ts` files

```ts
// If you truly want dotted type grouping in application code, this emits nothing:
namespace Webhooks {
  export type Provider = "stripe" | "github" | "slack";

  export interface Envelope<TPayload> {
    provider: Provider;
    eventId: string;
    receivedAtMs: number;
    signature: string;
    payload: TPayload;
  }

  export interface StripePayload  { type: string; data: { object: unknown } }
  export interface GithubPayload  { action: string; repository: { full_name: string } }
  export interface SlackPayload   { type: string; team_id: string }

  export type AnyEnvelope =
    | Envelope<StripePayload>
    | Envelope<GithubPayload>
    | Envelope<SlackPayload>;
}

function handle(envelope: Webhooks.AnyEnvelope): void { /* ... */ }

// ✅ Zero JS emitted (all members are types).
// ❌ Still flagged by erasableSyntaxOnly and by @typescript-eslint/no-namespace.
// ✅ The equivalent with strictly better tooling support:
//    // webhooks.ts — export the types flat
//    export type Provider = "stripe" | "github" | "slack";
//    export interface Envelope<TPayload> { … }
//    // consumer.ts
//    import type * as Webhooks from "./webhooks.js";
//    function handle(envelope: Webhooks.AnyEnvelope): void { … }
//    Identical call-site ergonomics, real module semantics.
```

### 8. How `declare global { namespace Express }` actually reaches `req`

```ts
// Understanding WHY this works stops it feeling like magic.
//
// 1. @types/express-serve-static-core declares (roughly):
//
//      declare global {
//        namespace Express {
//          interface Request {}          // ← deliberately EMPTY extension point
//          interface Response {}
//          interface Application {}
//          interface User {}
//        }
//      }
//
//      interface Request<P, ResBody, ReqBody, ReqQuery>
//        extends http.IncomingMessage, Express.Request { … }
//               //                     ^^^^^^^^^^^^^^^ inherits our additions
//
// 2. Interfaces MERGE (doc 18, doc 51). Our `interface Request { requestId: string }`
//    inside `namespace Express` merges into their empty one.
//
// 3. Express's real `Request` extends `Express.Request`, so `req.requestId` resolves.
//
// This is why the augmentation MUST target `Express.Request` and not
// `express.Request`, and why it must be a namespace — because that is the shape
// the library authors chose as their extension point. You are not choosing a
// namespace; you are matching one.

// The equivalent for a modern library that uses module augmentation instead:
declare module "fastify" {
  interface FastifyRequest {
    requestId: string;
    authToken?: { userId: number; role: string };
  }
}
// No namespace at all — Fastify's extension point is the module itself.
// See doc 51 — Module augmentation.
```

### 9. `namespace` + generics: not supported

```ts
// A namespace cannot take type parameters:
// namespace Repository<TEntity> { … }
// ❌ error TS1098: Type parameter list cannot be empty. / unexpected token

// Only the members can be generic:
namespace Repository {
  export interface Crud<TEntity, TId = number> {
    findById(id: TId): Promise<TEntity | null>;
    insert(entity: Omit<TEntity, "id">): Promise<TEntity>;
  }
  export function create<TEntity, TId = number>(table: string): Crud<TEntity, TId> {
    throw new Error("not implemented");
  }
}

// Modules have the same limitation (a file can't be generic), so this is not an
// argument either way — but it does kill the "namespace as a generic module" idea
// that people sometimes reach for.
```

### 10. Debugging "my augmentation does nothing"

```ts
// A checklist, in the order that fixes the most cases:
//
// 1. Is the file a module?
//    `declare global` requires a top-level `import` or `export {}`.
//
// 2. Is the file in the program?
//    $ npx tsc --listFiles | grep "types/express"
//    If it is absent, fix `include` / `files` / `typeRoots` in tsconfig.json.
//
// 3. Are you augmenting the right namespace?
//    `Express.Request`, not `express.Request`, not `Request`.
//
// 4. Did you write `interface` and not `type`?
//    Only interfaces merge. `type Request = { requestId: string }` inside the
//    namespace is a NEW type that shadows nothing and errors on duplicate.
//
// 5. Is `skipLibCheck` hiding a real conflict?
//    Temporarily set it to false and read the errors.
//
// 6. Are there two copies of @types/express in node_modules?
//    $ npm ls @types/express
//    Two copies = two distinct `Express` namespaces; your augmentation lands on
//    one and the runtime types come from the other. Dedupe or add an override.
//
// 7. Is your editor using a different TypeScript version than your build?
//    VS Code: "TypeScript: Select TypeScript Version" → "Use Workspace Version".
```

---

## Common mistakes

### Mistake 1 — Using a namespace to organise application code inside modules

```ts
// ❌ WRONG — src/services/userService.ts
export namespace UserService {
  const CACHE_TTL_SECONDS = 300;

  export interface UserProfile {
    userId: number;
    email: string;
    displayName: string;
  }

  export async function findById(userId: number): Promise<UserProfile | null> {
    return null;
  }

  export async function updateEmail(userId: number, email: string): Promise<void> {}
}

// Consumers are now stuck with a double indirection:
//   import { UserService } from "./services/userService.js";
//   const user: UserService.UserProfile = await UserService.findById(1);
//
// Costs you actually pay:
//   • An IIFE + object at runtime, in every process.
//   • No tree-shaking — importing `findById` drags `updateEmail` along.
//   • Non-erasable: fails `erasableSyntaxOnly`, fails `node --experimental-strip-types`.
//   • The file was already a namespace. You built a second one inside it.
```

```ts
// ✅ RIGHT — src/services/userService.ts
const CACHE_TTL_SECONDS = 300;              // file-private already

export interface UserProfile {
  userId: number;
  email: string;
  displayName: string;
}

export async function findById(userId: number): Promise<UserProfile | null> {
  return null;
}

export async function updateEmail(userId: number, email: string): Promise<void> {}

// Consumers pick the ergonomics they want:
//   import { findById, type UserProfile } from "./services/userService.js";
//   import * as UserService from "./services/userService.js";   // dotted, if you like it
//
// Erasable, tree-shakeable, lazy-loadable, no runtime wrapper.
```

### Mistake 2 — `declare global` in a file that is not a module

```ts
// ❌ WRONG — src/types/express.d.ts with no import and no export
declare global {
  namespace Express {
    interface Request {
      userId: number;
    }
  }
}
// error TS2669: Augmentations for the global scope can only be directly nested
//               in external modules or ambient module declarations.
//
// Or, in a .d.ts with no `declare global` wrapper at all and no exports, the
// namespace becomes a NEW global `Express` that shadows @types/express — the
// augmentation appears to compile but `req.userId` is still an error, because
// Express's Request extends THEIR Express.Request, not yours.
```

```ts
// ✅ RIGHT — make the file a module, then augment the global scope
import type { UserRole } from "../domain/roles.js";   // ← this makes it a module

declare global {
  namespace Express {
    interface Request {
      userId?: number;
      role?: UserRole;
    }
  }
}

export {};   // belt and braces — required if there is no import
```

### Mistake 3 — Expecting cross-file namespace merging in a module-based project

```ts
// ❌ WRONG
// src/metrics/counters.ts
import { registry } from "../telemetry/registry.js";     // ← file is a module

export namespace Metrics {
  export function counter(name: string, value = 1): void {
    registry.increment(name, value);
  }
}

// src/metrics/histograms.ts
import { registry } from "../telemetry/registry.js";

export namespace Metrics {
  export function histogram(name: string, valueMs: number): void {
    registry.observe(name, valueMs);
  }
}

// src/routes/orders.ts
import { Metrics } from "../metrics/counters.js";
Metrics.counter("orders_created_total");        // ✅ works
// Metrics.histogram("order_duration_ms", 12);  // ❌ Property 'histogram' does not exist
//
// The two namespaces NEVER merged — they are module-local. You now have two
// unrelated objects called `Metrics` and no compiler error telling you so.
```

```ts
// ✅ RIGHT — one module per concern, one barrel if you want a combined surface
// src/metrics/counters.ts
import { registry } from "../telemetry/registry.js";
export function counter(name: string, value = 1): void { registry.increment(name, value); }

// src/metrics/histograms.ts
import { registry } from "../telemetry/registry.js";
export function histogram(name: string, valueMs: number): void { registry.observe(name, valueMs); }

// src/metrics/index.ts — the explicit, statically-checked combined surface
export * from "./counters.js";
export * from "./histograms.js";

// src/routes/orders.ts
import * as Metrics from "../metrics/index.js";
Metrics.counter("orders_created_total");
Metrics.histogram("order_duration_ms", 12);     // ✅ both exist, verified by tsc
```

### Mistake 4 — Putting the namespace before the function or class it merges with

```ts
// ❌ WRONG — namespace declared first
namespace createCorrelationId {
  export let counter = 0;
}

function createCorrelationId(prefix = "corr"): string {
  createCorrelationId.counter += 1;
  return `${prefix}_${createCorrelationId.counter}`;
}
// error TS2434: A namespace declaration cannot be located prior to a class or
//               function with which it is merged.
//
// The reason is runtime, not aesthetic: the emitted IIFE does
// `(function (createCorrelationId) { … })(createCorrelationId || (createCorrelationId = {}))`
// which would create a plain object, and the later `function` declaration would
// then be hoisted over it — destroying the properties you just attached.
```

```ts
// ✅ RIGHT — the function (or class) first, the namespace second
function createCorrelationId(prefix = "corr"): string {
  createCorrelationId.counter += 1;
  return `${prefix}_${Date.now().toString(36)}_${createCorrelationId.counter}`;
}

namespace createCorrelationId {
  export let counter = 0;
  export const DEFAULT_PREFIX = "corr";
  export function reset(): void { counter = 0; }
}

createCorrelationId("order");     // "order_lx3k9_1"
createCorrelationId.reset();      // ✅

// ✅ EVEN BETTER for new code — an object with a call signature, fully erasable:
interface CorrelationIdFactory {
  (prefix?: string): string;
  counter: number;
  reset(): void;
}

export const correlationId: CorrelationIdFactory = Object.assign(
  (prefix = "corr"): string => {
    correlationId.counter += 1;
    return `${prefix}_${Date.now().toString(36)}_${correlationId.counter}`;
  },
  {
    counter: 0,
    reset(): void { correlationId.counter = 0; },
  },
);
```

### Mistake 5 — Using `type` instead of `interface` inside an augmentation

```ts
// ❌ WRONG — type aliases do not merge
declare global {
  namespace Express {
    type Request = {
      requestId: string;
      userId: number;
    };
    // error TS2300: Duplicate identifier 'Request'.
    // (Or, if the name happens not to clash, you have created an unrelated type
    //  that Express's real Request does not extend — silently useless.)
  }
}
export {};
```

```ts
// ✅ RIGHT — interfaces merge; that is the whole mechanism
declare global {
  namespace Express {
    interface Request {
      requestId: string;
      userId?: number;
    }
  }
}
export {};

// If you need a type alias, declare it OUTSIDE the interface and reference it:
type AuthContext = { userId: number; role: "admin" | "customer"; expiresAtMs: number };

declare global {
  namespace Express {
    interface Request {
      authContext?: AuthContext;    // ✅ alias used as a member type — fine
    }
  }
}
```

### Mistake 6 — Assuming a namespace gives you runtime privacy for secrets

```ts
// ❌ WRONG — "it's not exported, so it's safe"
namespace Secrets {
  const JWT_SECRET = process.env.JWT_SECRET ?? "dev-secret";
  export function sign(payload: object): string {
    return `${btoa(JSON.stringify(payload))}.${JWT_SECRET}`;
  }
}
// The closure genuinely hides it from `Secrets.JWT_SECRET` — but the STRING is
// still in the emitted bundle, readable by anyone with the file, visible in a
// heap dump, and printed by any error that stringifies the closure's scope in
// some debuggers. Namespace privacy is an API-surface tool, not a security tool.
```

```ts
// ✅ RIGHT — secrets come from the environment at call time, never bundled
export function sign(payload: object, secret: string): string {
  return `${btoa(JSON.stringify(payload))}.${secret}`;
}

// src/config.ts — read once, validated, never inlined into a bundle
function requireEnv(name: keyof NodeJS.ProcessEnv): string {
  const value = process.env[name];
  if (typeof value !== "string" || value.length === 0) {
    throw new Error(`Missing required environment variable: ${String(name)}`);
  }
  return value;
}

export const config = {
  jwtSecret: requireEnv("JWT_SECRET"),
  databaseUrl: requireEnv("DATABASE_URL"),
} as const;
```

---

## Practice exercises

### Exercise 1 — easy

Write a `.d.ts` file that types a legacy global JavaScript SDK, then use it.

Requirements:

1. Declare an ambient namespace `AuditSdk` (no `export` keywords needed inside it) containing:
   - a type `AuditAction = "create" | "update" | "delete" | "login" | "logout"`
   - an interface `AuditEntry` with `entryId: string`, `userId: number`, `action: AuditAction`, `resource: string`, `timestampMs: number`, and an optional `metadata: Record<string, string | number>`
   - an interface `AuditConfig` with `endpoint: string`, `apiKey: string`, and optional `batchSize: number`
   - a `const version: string`
   - functions `configure(config: AuditConfig): void`, `record(entry: Omit<AuditEntry, "entryId" | "timestampMs">): void`, and `flush(): Promise<number>`
   - a nested namespace `Query` with `interface QueryFilters { userId?: number; action?: AuditAction; sinceMs?: number }` and `function search(filters: QueryFilters): Promise<readonly AuditEntry[]>`
2. Declare the actual global binding so `AuditSdk.record(...)` type-checks as a value.
3. In a separate module, write `recordOrderDeletion(userId: number, orderId: string): void` that calls `AuditSdk.record` with `action: "delete"` and `resource: \`order:${orderId}\``.
4. Write `recentDeletionsByUser(userId: number): Promise<readonly AuditSdk.AuditEntry[]>` using `AuditSdk.Query.search` with a 24-hour window.
5. Add a `declare global { namespace NodeJS { interface ProcessEnv { … } } }` block typing `AUDIT_ENDPOINT` and `AUDIT_API_KEY` as required strings, and use them in a `configure` call with no `!` and no `??` fallback.

```ts
// Write your code here
```

### Exercise 2 — medium

Migrate this legacy namespace-based service to ES modules, from scratch. Do not copy mechanically — rewrite it, and improve it where the module form allows.

Starting point:

```ts
/// <reference path="../domain/types.ts" />
/// <reference path="../infra/cache.ts" />

namespace App.Notifications {
  import Cache = App.Infra.Cache;

  const MAX_BATCH = 50;
  const DEDUPE_WINDOW_MS = 60_000;

  export type Channel = "email" | "sms" | "push";

  export interface Notification {
    notificationId: string;
    userId: number;
    channel: Channel;
    subject: string;
    body: string;
    scheduledAtMs: number;
  }

  export interface DeliveryReceipt {
    notificationId: string;
    deliveredAtMs: number;
    providerMessageId: string;
  }

  function dedupeKey(notification: Notification): string {
    return `${notification.userId}:${notification.channel}:${notification.subject}`;
  }

  export class NotificationDispatcher {
    constructor(private readonly logger: App.Infra.Logger) {}

    async dispatch(notifications: Notification[]): Promise<DeliveryReceipt[]> {
      const batch = notifications.slice(0, MAX_BATCH);
      const receipts: DeliveryReceipt[] = [];
      for (const notification of batch) {
        const key = dedupeKey(notification);
        if (await Cache.exists(key)) {
          this.logger.warn(`Deduped ${notification.notificationId}`);
          continue;
        }
        await Cache.set(key, "1", DEDUPE_WINDOW_MS);
        receipts.push(await this.send(notification));
      }
      return receipts;
    }

    private async send(notification: Notification): Promise<DeliveryReceipt> {
      return {
        notificationId: notification.notificationId,
        deliveredAtMs: Date.now(),
        providerMessageId: `msg_${Math.random().toString(36).slice(2)}`,
      };
    }
  }
}

namespace App.Notifications {
  export function isUrgent(notification: Notification): boolean {
    return notification.channel === "sms" || notification.subject.startsWith("URGENT");
  }
}
```

Write, from scratch:

1. `src/notifications/types.ts` — the `Channel`, `Notification`, and `DeliveryReceipt` types, exported flat. Derive `Channel` from a `NOTIFICATION_CHANNEL` `as const` object rather than hand-writing the union.
2. `src/notifications/dedupe.ts` — the `dedupeKey` helper plus the `DEDUPE_WINDOW_MS` constant, both properly scoped (only `dedupeKey` exported).
3. `src/notifications/dispatcher.ts` — the `NotificationDispatcher` class with real `import` statements for the cache, the logger, and the types. Constructor takes both dependencies explicitly.
4. `src/notifications/predicates.ts` — `isUrgent`, plus a new `isScheduledForFuture(notification, nowMs?)`.
5. `src/notifications/index.ts` — a barrel re-exporting the public surface, using `export type` for the types so the file stays erasable.
6. A consumer file that imports the barrel with `import * as Notifications from "./notifications/index.js"` and shows that the old dotted call sites still read naturally.
7. Add a comment above each file explaining which piece of the original namespace it replaced and what the module form bought you (privacy, tree-shaking, or erasability).

Constraint: no `namespace` keyword anywhere in your answer, no `/// <reference>`, no `any`.

```ts
// Write your code here
```

### Exercise 3 — hard

Build the complete type layer for a multi-tenant Express API, using namespaces **only** where they are genuinely required, and prove the boundary.

Part A — global augmentations (namespaces required):

```ts
// - src/types/global.d.ts
//   * declare global { namespace Express { … } } adding to Request:
//       requestId: string
//       receivedAtNs: bigint
//       tenant?: TenantContext          (interface defined in the same file)
//       authToken?: AuthToken
//     and to Response:
//       durationMs?: number
//   * declare global { namespace NodeJS { interface ProcessEnv { … } } } typing:
//       NODE_ENV: "development" | "test" | "production"
//       PORT: string
//       DATABASE_URL: string
//       REDIS_URL: string
//       JWT_SECRET: string
//       TENANT_HEADER: string
//   * The file must import UserRole from a domain module and still compile —
//     i.e. get the module/global interaction right.
```

Part B — an ambient namespace for a global vendor SDK (namespace required):

```ts
// - src/types/billing-sdk.d.ts
//   declare namespace BillingSdk with:
//     type SubscriptionTier = "free" | "starter" | "growth" | "enterprise"
//     interface Subscription { subscriptionId, tenantId, tier, seats, renewsAtMs, status }
//     interface UsageRecord { tenantId, metric, quantity, recordedAtMs }
//     function getSubscription(tenantId: string): Promise<Subscription>
//     function reportUsage(record: UsageRecord): Promise<void>
//     namespace Limits {
//       interface TierLimits { maxSeats: number; maxRequestsPerMinute: number; maxStorageGb: number }
//       function forTier(tier: SubscriptionTier): TierLimits
//     }
//   plus the global const binding.
```

Part C — everything else must be a plain module, with NO namespace:

```ts
// - src/domain/roles.ts        — USER_ROLE as const + UserRole union + guard
// - src/domain/tenant.ts       — TenantContext interface, parseTenantId guard
// - src/errors/apiError.ts     — ApiError class with STATIC methods and FLAT
//                                exported types (ApiErrorCode, FieldIssue,
//                                SerializedApiError). Deliberately NOT a merged
//                                namespace — write the erasable version.
// - src/middleware/tenant.ts   — resolveTenant middleware reading the header named
//                                by process.env.TENANT_HEADER, looking up the
//                                subscription via BillingSdk, and attaching
//                                req.tenant. 404s on unknown tenants.
// - src/middleware/quota.ts    — enforceQuota middleware using BillingSdk.Limits.forTier
//                                to reject over-limit requests with 429.
// - src/routes/usage.ts        — POST /usage that validates the request body,
//                                calls BillingSdk.reportUsage, and returns an
//                                ApiResponse<{ accepted: true }>.
```

Part D — prove the boundary:

```ts
// 1. Show the exact compile error you get if src/types/global.d.ts has neither
//    an import nor `export {}`. Quote the error code.
// 2. Show that removing src/types/global.d.ts from tsconfig `include` makes
//    `req.requestId` an error, and write the tsconfig snippet that fixes it.
// 3. Write a tsconfig.json with "erasableSyntaxOnly": true and confirm every
//    file in Part C compiles under it while Parts A and B are still legal.
//    Explain in a comment WHY the declare-namespace forms are exempt.
// 4. Take the ApiError-with-merged-namespace version from Example 2 of this doc
//    and show the mechanical rewrite to the flat/static version, listing every
//    call site that changes.
// 5. Add an ESLint config snippet with @typescript-eslint/no-namespace configured
//    so Parts A and B pass and any namespace in Part C fails.
```

```ts
// Write your code here
```

---

## Quick reference cheat sheet

```ts
// ── Declaring ────────────────────────────────────────────────────────────────
namespace Utils {                       // emits an IIFE + object (non-erasable)
  const privateHelper = () => {};       // no `export` → invisible outside
  export const PUBLIC_CONST = 1;        // `export` → attached to the object
  export interface Dto { id: number }   // types are always erased
}

declare namespace Sdk {                 // ambient — emits NOTHING
  interface Options { key: string }     // `export` is implicit inside `declare`
  function init(options: Options): void;
}

namespace Deep.Nested.Path {}           // dotted shorthand == nested blocks
namespace A { export namespace B {} }   // explicit nesting

// ── Merging ──────────────────────────────────────────────────────────────────
namespace M { export const a = 1 }
namespace M { export const b = 2 }      // ✅ merges — M.a and M.b
// Only in the SAME scope. Module files do not merge across files.

function fn() {}                        // function FIRST
namespace fn { export const version = "1" }   // then namespace — statics

class C {}                              // class FIRST
namespace C { export type Id = string } // then namespace — nested types + statics

interface I { x: number }
namespace I { export const DEFAULT: I = { x: 0 } }   // interface + namespace ✅

enum E { A = "a" }
namespace E { export function isA(e: E) { return e === E.A } }   // enum + namespace ✅

// const x = 1; namespace x {}          // ❌ cannot merge with a variable

// ── Aliases ──────────────────────────────────────────────────────────────────
import Short = Deep.Nested.Path;        // namespace alias (erased if type-only)
import Legacy = require("./legacy");    // CommonJS import (non-erasable)
export = Sdk;                           // CommonJS whole-module export

// ── Global augmentation (the one you will write) ─────────────────────────────
declare global {
  namespace Express {
    interface Request { requestId: string; userId?: number }
    interface Response { durationMs?: number }
    interface User { userId: number; role: string }
  }
  namespace NodeJS {
    interface ProcessEnv { DATABASE_URL: string; JWT_SECRET: string }
  }
}
export {};                              // REQUIRED if the file has no import

// ── @types/node namespace types you use daily ────────────────────────────────
const t: NodeJS.Timeout = setTimeout(() => {}, 1_000);
const env: NodeJS.ProcessEnv = process.env;
function onError(e: NodeJS.ErrnoException) { e.code; e.errno; e.path; }
type Sig = NodeJS.Signals;              // "SIGINT" | "SIGTERM" | …

// ── Module replacements ──────────────────────────────────────────────────────
// namespace grouping        →  a file + `import * as Group from "./group.js"`
// namespace privacy         →  don't export it (file scope)
// namespace merging         →  a barrel: `export * from "./a.js"`
// namespace nested types    →  export flat + `import type * as X`
// namespace statics on class→  `static` methods
// namespace statics on fn   →  `Object.assign(fn, { … })` + a call-signature interface

// ── tsconfig flags ───────────────────────────────────────────────────────────
// "erasableSyntaxOnly": true   → any non-`declare` namespace is TS1294
// "isolatedModules": true      → cross-file namespace merging is impossible
// "verbatimModuleSyntax": true → implies isolatedModules
// "outFile": "bundle.js"       → the legacy namespace concatenation mode

// ── ESLint ───────────────────────────────────────────────────────────────────
// "@typescript-eslint/no-namespace": ["error", {
//   "allowDeclarations": true, "allowDefinitionFiles": true
// }]
// "@typescript-eslint/prefer-namespace-keyword": "error"   // `module X` → `namespace X`
```

### Decision table

| You want to… | Namespace | ES module | Verdict |
|---|---|---|---|
| Group functions in app code | ⚠️ works, non-erasable, no tree-shaking | ✅ one file per concern | **module** |
| Hide implementation details | ✅ closure privacy | ✅ file privacy | **module** |
| Add `req.userId` to Express | ✅ the only way | ❌ | **namespace** |
| Type `process.env` | ✅ `NodeJS.ProcessEnv` | ❌ | **namespace** |
| Type a global `<script>` SDK | ✅ `declare namespace` | ❌ nothing to import | **namespace** |
| Type an npm package | ❌ | ✅ `declare module "pkg"` | **module** |
| Nest types under a class name | ✅ class + namespace merge | ⚠️ flat names or a nested `type` | **taste; prefer flat** |
| Add statics to a function | ✅ function + namespace merge | ✅ `Object.assign` + interface | **module** |
| Combine several files' exports | ⚠️ merging, order-dependent, fragile | ✅ barrel `export *` | **module** |
| Lazy-load a heavy dependency | ❌ impossible | ✅ `await import()` | **module** |
| Run under `node --strip-types` | ❌ unless `declare` | ✅ | **module** |
| Publish to npm | ❌ pollutes global scope | ✅ | **module** |
| Avoid name collisions | ❌ global, unrenameable | ✅ rename on import | **module** |
| **Default for new code** | never | always | **module** |

### One-line rule

> Write a namespace only when the thing you are describing is genuinely global and you did not create it. Everywhere else, the file is your namespace.

---

## Connected topics

- **48 — Modules in TypeScript** — the replacement for everything in this document; `import`/`export`, the module resolution graph, barrels, and `import * as` as the ergonomic substitute for dotted namespace access.
- **49 — Declaration files** — where `declare namespace` legitimately lives; how `.d.ts` files are found, and why a namespace in one emits nothing.
- **50 — Ambient declarations** — `declare global`, `declare module "pkg"`, and `declare const`; the exact mechanics that make `declare namespace NodeJS` work.
- **51 — Module augmentation** — the modern alternative to namespace augmentation for libraries that expose module-scoped types (Fastify, Vitest, Vue), and how it differs from `declare global`.
- **53 — Extending Express Request** — the full treatment of `declare global { namespace Express { interface Request } }`, including the two-copies-of-@types problem and how the augmentation reaches `req`.
- **55 — Typing environment variables** — `NodeJS.ProcessEnv` augmentation in practice, plus why a validated `config` object usually beats it.
- **18 — Extending interfaces** — interface declaration merging, the mechanism that every namespace augmentation depends on.
- **69 — Enum vs union types** — the other classic non-erasable TypeScript construct, with the identical `isolatedModules` / `erasableSyntaxOnly` / type-stripping story.
- **47 — `keyof` and `typeof` operators** — `typeof SomeNamespace` gives you the shape of the emitted object, which is how you type a namespace as a value.
- **33 — Classes in TypeScript** — static members are the erasable replacement for the class + namespace merge pattern.
