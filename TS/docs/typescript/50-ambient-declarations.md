# 50 — Ambient declarations

## What is this?

An **ambient declaration** is a statement that tells the TypeScript compiler *"a thing with this name and this type already exists at runtime — I am not creating it, I am describing it."*

The keyword is `declare`.

```ts
// src/types/runtime-globals.d.ts

// There is a global variable called __BUILD_SHA__ at runtime.
// I did not create it. My bundler injected it. Here is its type.
declare const __BUILD_SHA__: string;

// There is a global function called reportRequestMetric.
// The APM agent attached it to globalThis before my code ran.
declare function reportRequestMetric(routeName: string, durationMs: number): void;

// There is a class called LegacyAuditWriter that a <script>-style
// vendor bundle put on the global object.
declare class LegacyAuditWriter {
  constructor(endpoint: string);
  write(userId: number, action: string): void;
}
```

None of those three lines emit a single byte of JavaScript. They are pure compile-time assertions. After writing them, `__BUILD_SHA__.slice(0, 7)` type-checks anywhere in your project, and `__BUILD_SHA__.toFixed(2)` is an error — even though the compiler has never seen the code that actually defines `__BUILD_SHA__`.

Three properties define an ambient declaration:

1. **It has no implementation.** No function body, no initializer, no field assignment. The compiler will reject one if you try.
2. **It emits nothing.** `declare` is erased completely. If the runtime value does not actually exist, you get `ReferenceError` at runtime with a perfectly clean build.
3. **It is a claim, not a proof.** `declare` is you signing a contract on behalf of the runtime. TypeScript has no way to verify it and will not try.

"Ambient" is the word TypeScript uses for **the global, script-level type environment** — the space where `process`, `Buffer`, `setTimeout`, `console`, `JSON` and `Math` live. Ambient declarations are how you add things to that space, and how every one of those built-ins got there in the first place. `Buffer` is not magic; it is a `declare class Buffer` inside `@types/node`.

The previous doc (`49 — Declaration files`) was about the **file format** — what a `.d.ts` is and how it ships. This doc is about the **keyword and the scoping rules** — what `declare` does, where the declarations land, and the single most confusing rule in the whole system: whether your file is a *script* or a *module*.

## Why does it matter?

On a Node backend, ambient declarations are not an exotic corner. You will need them in the first month of any real TypeScript service:

- **`process.env` is a lie by default.** `@types/node` types it as `Record<string, string | undefined>`. Every env var is `string | undefined`, every typo compiles. `declare namespace NodeJS { interface ProcessEnv { DATABASE_URL: string } }` turns your environment contract into a checked type — one small ambient block that pays off in every file that touches config.

- **Things get attached to `globalThis` at runtime and TypeScript cannot see them.** A DataDog/OpenTelemetry agent, a `--require`d instrumentation hook, a test setup file that sets `globalThis.testDbPool`, a serverless platform that injects `__REQUEST_CONTEXT__`. Without `declare global`, every access is `TS2304: Cannot find name`.

- **Untyped packages.** `declare module "some-untyped-pkg"` is the ambient mechanism that makes an untyped npm package importable and typed. This is *the* everyday use of `declare`.

- **Non-code imports.** `import schema from "./user.graphql"`, `import template from "./welcome-email.hbs?raw"`, `import config from "./limits.json"` — the compiler has no idea what those are. Wildcard ambient module declarations (`declare module "*.graphql"`) teach it.

- **Global augmentation of existing types.** Adding `req.userId` to Express's `Request`, adding a `toJSONSafe()` method your polyfill installed on `Error.prototype`, extending `NodeJS.Global`. All of it runs through ambient declarations.

And a hard-won practical reason: **ambient declarations are the #1 source of "it works on my machine but the CI build says `Cannot find name`."** The rules governing *when* an ambient declaration is visible — script vs module, `include` globs, `typeRoots` — are subtle, and getting them wrong produces errors that look completely unrelated to the file you edited. Understanding this doc is what makes those errors five-second fixes instead of afternoon-long ones.

---

## The JavaScript way vs the TypeScript way

There is no JavaScript equivalent of `declare` — JavaScript has no concept of "describe something that exists elsewhere." So the contrast is between **the untyped/defensive way** and the **ambient-declaration way**.

### The JavaScript way — globals as a fog of `undefined`

Here is a real Node service. An APM agent is loaded via `node --require ./instrument.js server.js`. It attaches a tracer to the global object. A build step injects the git SHA. And config comes from `process.env`.

```js
// The JavaScript way — src/routes/orders.js

// Where does globalThis.__tracer come from? The instrument.js require hook.
// Where is it documented? A Confluence page from 2023.
// What are its methods? Nobody remembers.

async function handleCreateOrder(req, res) {
  const span = globalThis.__tracer.startSpan("orders.create");   // hope it's loaded

  const userId = req.body.userId;

  // process.env — everything is a string or undefined, forever
  const maxOrderCents = process.env.MAX_ORDER_CENTS;             // "50000" or undefined
  if (req.body.totalCents > maxOrderCents) {                     // 💥 string comparison!
    return res.status(400).json({ error: "too large" });         //    "9000" > "50000" === true
  }

  const dbUrl = process.env.DATBASE_URL;                         // typo → undefined
  const pool  = await getPool(dbUrl);                            // 💥 connects to nothing

  // Build metadata injected by esbuild --define
  res.setHeader("x-build", __BUILD_SHA__);                       // ReferenceError in dev,
                                                                 // where the define isn't applied

  span.end();
  globalThis.__tracer.recordMetric("orders.created");             // 💥 method is `record`, not
                                                                 //    `recordMetric` — crash
}
```

Count the failure modes:

- `"9000" > "50000"` is `true` in JavaScript — lexicographic string comparison. Your order-size limit silently inverts. This is a real bug class, not a hypothetical.
- `DATBASE_URL` is a typo. `process.env.DATBASE_URL` is `undefined`. `getPool(undefined)` either throws deep in the driver or, worse, falls back to a default local connection.
- `__BUILD_SHA__` exists in the production bundle and not in `ts-node` dev. `ReferenceError` in exactly the environment you test in least.
- `__tracer.recordMetric` — wrong method name. Crashes on the success path of your highest-traffic route.

The defensive JavaScript answer is a wall of runtime checks:

```js
// The "careful" JavaScript way — noise, and still not safe
const MAX_ORDER_CENTS = Number(process.env.MAX_ORDER_CENTS);
if (!Number.isFinite(MAX_ORDER_CENTS)) throw new Error("MAX_ORDER_CENTS missing/invalid");

const DATABASE_URL = process.env.DATABASE_URL;
if (!DATABASE_URL) throw new Error("DATABASE_URL missing");

const tracer = globalThis.__tracer;
if (!tracer || typeof tracer.startSpan !== "function") throw new Error("tracer not installed");
// ...and you still have to remember it's `record` and not `recordMetric`.
```

You write that once per global, per service, and you *still* get no autocomplete, no rename support, and no protection against `tracer.recordMetric`.

### The TypeScript way — declare the world once

```ts
// ── src/types/ambient.d.ts — written once, ~30 lines ─────────────────────────

// 1. The build-time injected constant. Now typed everywhere.
declare const __BUILD_SHA__: string;

// 2. The APM tracer that instrument.js attaches to globalThis.
interface RequestTracer {
  startSpan(name: string, attributes?: Record<string, string>): TraceSpan;
  record(metricName: string, value?: number): void;
}
interface TraceSpan {
  setAttribute(key: string, value: string | number | boolean): void;
  recordException(error: Error): void;
  end(): void;
}

// 3. Add them to the global scope, and type process.env properly.
declare global {
  // eslint-disable-next-line no-var
  var __tracer: RequestTracer;

  namespace NodeJS {
    interface ProcessEnv {
      NODE_ENV:        "development" | "test" | "production";
      DATABASE_URL:    string;
      MAX_ORDER_CENTS: string;      // still a string — but a GUARANTEED one
      SENTRY_DSN?:     string;      // genuinely optional
    }
  }
}

export {};   // ← makes this file a module so `declare global` is legal (see Concept 4)
```

```ts
// ── src/routes/orders.ts — the same handler, now checked ─────────────────────
import type { Request, Response } from "express";

interface CreateOrderBody {
  userId:     number;
  totalCents: number;
  sku:        string;
}

export async function handleCreateOrder(req: Request, res: Response): Promise<void> {
  const span = globalThis.__tracer.startSpan("orders.create");   // ✅ typed, autocompleted

  const requestBody = req.body as CreateOrderBody;

  // ✅ process.env.MAX_ORDER_CENTS is `string`, not `string | undefined`.
  //    Number() is required by YOU, not by a defensive check.
  const maxOrderCents = Number(process.env.MAX_ORDER_CENTS);
  if (requestBody.totalCents > maxOrderCents) {                   // ✅ number vs number
    res.status(400).json({ error: "Order exceeds limit" });
    return;
  }

  // ❌ const dbUrl = process.env.DATBASE_URL;
  //    TS2339: Property 'DATBASE_URL' does not exist on type 'ProcessEnv'.
  //            Did you mean 'DATABASE_URL'?
  const dbUrl = process.env.DATABASE_URL;                         // ✅ string, no `?? throw`

  res.setHeader("x-build", __BUILD_SHA__);                        // ✅ string

  // ❌ globalThis.__tracer.recordMetric("orders.created");
  //    TS2551: Property 'recordMetric' does not exist on type 'RequestTracer'.
  //            Did you mean 'record'?
  globalThis.__tracer.record("orders.created");                   // ✅

  span.end();
  void dbUrl;
}
```

Thirty lines of `declare` eliminated four production incident classes across every file in the service — permanently, for every teammate, with autocomplete and rename support included. That is the trade ambient declarations offer: **one unverifiable file buys verification at a thousand call sites.**

The catch — and it is the reason this doc is long — is that `declare` is a promise the compiler cannot check. If `instrument.js` stops attaching `__tracer`, TypeScript will keep telling you it exists. Ambient declarations move the risk from "a thousand scattered typos" to "one file that must be kept honest." That is a very good trade, but it is a trade.

---

## Syntax

```ts
// ═══════════════════════════════════════════════════════════════════════════════
// 1. AMBIENT VALUES — `declare` on variables, functions, classes, enums
// ═══════════════════════════════════════════════════════════════════════════════

declare const   API_BASE_URL: string;          // const → not reassignable, not widened
declare let     currentRequestId: string;      // let   → reassignable
declare var     legacyGlobalCounter: number;   // var   → also lands on globalThis (see Concept 5)

declare function verifyAuthToken(token: string): boolean;      // no body allowed
declare function readConfig(path: string): Record<string, unknown>;

declare class AuditWriter {                    // no method bodies, no field initializers
  constructor(endpoint: string, batchSize?: number);
  readonly pending: number;
  write(userId: number, action: string): void;
  flush(): Promise<number>;
  static fromEnv(): AuditWriter;
}

declare enum LogLevel { Debug, Info, Warn, Error }   // ambient enum — no JS emitted
declare namespace Billing {                          // ambient namespace
  interface Invoice { invoiceId: string; amountCents: number }
  function issue(customerId: string): Invoice;
}

// ═══════════════════════════════════════════════════════════════════════════════
// 2. TYPES NEVER NEED `declare` — they have no runtime existence to declare
// ═══════════════════════════════════════════════════════════════════════════════

interface ApiResponse<TBody> { statusCode: number; body: TBody }   // ✅ no `declare`
type AuthRole = "admin" | "member" | "service";                    // ✅ no `declare`

// ═══════════════════════════════════════════════════════════════════════════════
// 3. SCRIPT FILE vs MODULE FILE — the single most important distinction
// ═══════════════════════════════════════════════════════════════════════════════

// ── SCRIPT: no top-level import/export → everything is GLOBAL ────────────────
// file: src/types/globals.d.ts
declare const __BUILD_SHA__: string;      // visible in EVERY file, no import needed

// ── MODULE: has a top-level import or export → everything is LOCAL ───────────
// file: src/types/client.d.ts
import type { Readable } from "node:stream";        // ← this import makes it a module
declare const __BUILD_SHA__: string;                // ← now PRIVATE to this file!

// ── MODULE, but reaching back into global scope ──────────────────────────────
declare global {
  const __BUILD_SHA__: string;            // ✅ global again
}
export {};                                // ← the marker that forces module-hood

// ═══════════════════════════════════════════════════════════════════════════════
// 4. AMBIENT MODULES — typing packages by their import specifier
// ═══════════════════════════════════════════════════════════════════════════════

declare module "@acme/rate-limiter" {          // specifier must match the import EXACTLY
  export interface LimiterOptions { windowMs: number; maxRequests: number }
  export function createLimiter(options: LimiterOptions): { check(key: string): boolean };
}

declare module "@acme/totally-untyped";        // no body → the whole module is `any`

declare module "some-cjs-pkg" {                // CommonJS `module.exports = fn`
  function main(options?: object): void;
  export = main;
}

// ═══════════════════════════════════════════════════════════════════════════════
// 5. WILDCARD MODULES — non-code imports
// ═══════════════════════════════════════════════════════════════════════════════

declare module "*.graphql" {
  const source: string;
  export default source;
}

declare module "*.sql" {
  const query: string;
  export default query;
}

declare module "*?raw" {                       // bundler query suffixes
  const contents: string;
  export default contents;
}

// ═══════════════════════════════════════════════════════════════════════════════
// 6. NodeJS NAMESPACE — the canonical process.env pattern
// ═══════════════════════════════════════════════════════════════════════════════

declare namespace NodeJS {                     // in a SCRIPT file: no `declare global` needed
  interface ProcessEnv {
    DATABASE_URL: string;
    PORT:         string;
  }
}

// ═══════════════════════════════════════════════════════════════════════════════
// 7. tsconfig — making sure tsc actually READS your ambient files
// ═══════════════════════════════════════════════════════════════════════════════
// {
//   "compilerOptions": { "strict": true, "types": ["node"] },
//   "include": ["src/**/*.ts"],        // matches src/**/*.d.ts too
//   "files":   ["types/globals.d.ts"]  // belt-and-braces explicit inclusion
// }
//
// Verify:  npx tsc --noEmit --listFiles | grep globals.d.ts
```

---

## How it works — concept by concept

### Concept 1 — What `declare` actually means to the compiler

`declare` puts a declaration into **ambient context**. In ambient context the compiler applies two rules:

1. **No implementation may be provided.** The declaration is a signature only.
2. **Nothing is emitted.** The declaration disappears entirely from the JavaScript output.

```ts
// ── src/types/build-info.d.ts ────────────────────────────────────────────────

// ✅ Legal — signature only
declare const __BUILD_SHA__: string;
declare const __BUILD_TIME_MS__: number;
declare function reportRequestMetric(routeName: string, durationMs: number): void;

// ❌ Illegal — an initializer is an implementation
declare const __BUILD_SHA__: string = "abc123";
//                                  ^ TS1039: Initializers are not allowed in ambient contexts.

// ❌ Illegal — a function body is an implementation
declare function reportRequestMetric(routeName: string, durationMs: number): void {
  console.log(routeName, durationMs);
}
// ^ TS1183: An implementation cannot be declared in ambient contexts.

// ❌ Illegal — a class field initializer
declare class MetricsBuffer {
  size: number = 0;
  //           ^ TS1039
  flush(): void { }
  //           ^ TS1183
}
```

The **erasure** property is what makes `declare` dangerous and useful in equal measure. Consider what actually reaches Node:

```ts
// ── src/config.ts ────────────────────────────────────────────────────────────
declare const DEFAULT_PAGE_SIZE: number;      // I claim this exists globally

export function buildPageQuery(cursor?: string): { limit: number; cursor?: string } {
  return { limit: DEFAULT_PAGE_SIZE, cursor };
}
```

```js
// ── dist/config.js — what tsc actually emits ─────────────────────────────────
"use strict";
Object.defineProperty(exports, "__esModule", { value: true });
exports.buildPageQuery = buildPageQuery;
function buildPageQuery(cursor) {
    return { limit: DEFAULT_PAGE_SIZE, cursor };   // ← no definition anywhere
}
// 💥 ReferenceError: DEFAULT_PAGE_SIZE is not defined
```

The `declare` line vanished. Nothing defines `DEFAULT_PAGE_SIZE`. The build was green.

**The rule to internalise:** `declare` is only ever correct when *something else* — a `--require` hook, a bundler `--define`, a vendor `<script>`, a native binding, an npm package — genuinely puts that value into scope before your code runs. It is a description of reality, never a creation of it.

Inside a `.d.ts` file, everything is *implicitly* in ambient context, so `declare` is often optional there:

```ts
// ── src/types/metrics.d.ts — these two lines mean exactly the same thing ─────
declare function recordMetric(name: string, value: number): void;
function recordMetric(name: string, value: number): void;   // implicitly ambient in a .d.ts
```

Write `declare` explicitly anyway. It is the convention, it survives a copy-paste into a `.ts` file (where it is mandatory), and it makes the intent legible to the next reader.

### Concept 2 — Ambient variables, functions, classes, enums, and namespaces

Each declaration form has behaviour worth knowing.

**Ambient variables — `const` vs `let` vs `var`**

```ts
// ── src/types/globals.d.ts (a SCRIPT file — no top-level import/export) ──────

declare const   BUILD_CHANNEL: "stable" | "canary";   // literal union preserved
declare let     activeRequestId: string;
declare var     legacyRequestCount: number;
```

```ts
// ── Consuming code ───────────────────────────────────────────────────────────
if (BUILD_CHANNEL === "canary") { /* ✅ narrowed */ }
if (BUILD_CHANNEL === "beta")   { }
//  ❌ TS2367: This comparison appears unintentional because the types
//     '"stable" | "canary"' and '"beta"' have no overlap.

BUILD_CHANNEL = "stable";          // ❌ TS2588: Cannot assign to 'BUILD_CHANNEL'
                                   //    because it is a constant.
activeRequestId = "req_92f1";      // ✅ `let` is assignable

globalThis.legacyRequestCount++;   // ✅ only `var` (and `declare global { var }`)
globalThis.activeRequestId;        // ❌ TS2339: Property 'activeRequestId' does not
                                   //    exist on type 'typeof globalThis'.
```

That last pair is the practical difference and it trips everyone: **only `var` declarations become properties of `globalThis`** in TypeScript's model, mirroring real JavaScript semantics where `var` at script top level creates a global object property and `let`/`const` create a lexical binding that does not. If you want `globalThis.__tracer` to type-check, you must write `var __tracer: RequestTracer`.

**Ambient functions — overloads work as expected**

```ts
// ── src/types/native-bindings.d.ts ───────────────────────────────────────────
// A native addon loaded by process._linkedBinding, exposed as a global.

declare function hashPassword(plaintext: string): Promise<string>;
declare function hashPassword(plaintext: string, rounds: number): Promise<string>;
declare function hashPassword(plaintext: Buffer, rounds: number): Promise<Buffer>;

// ✅ All three overloads are usable; no implementation signature is required
//    (and none is permitted) in ambient context.
```

Note the contrast with a normal `.ts` file: there you must supply an implementation signature after the overload list. In ambient context you must **not** — there is no implementation to sign for.

**Ambient classes — instance type and constructor type in one**

```ts
// ── src/types/vendor-sdk.d.ts ────────────────────────────────────────────────
declare class PaymentGateway {
  constructor(apiKey: string, options?: { timeoutMs?: number; baseUrl?: string });

  readonly apiKey: string;
  charge(customerId: string, amountCents: number, currency: string): Promise<string>;
  refund(paymentId: string): Promise<void>;

  static fromEnvironment(): PaymentGateway;
  static readonly SUPPORTED_CURRENCIES: readonly string[];

  protected buildHeaders(): Record<string, string>;   // ✅ modifiers allowed
  private connection: unknown;                        // ✅ affects nominal compatibility
}
```

```ts
// A `declare class` gives you BOTH a type and a value:
const gateway: PaymentGateway = PaymentGateway.fromEnvironment();  // type position ✅
const another = new PaymentGateway(process.env.STRIPE_KEY!);       // value position ✅

// And because of the `private` member, it is nominally typed —
// you cannot substitute a plain object:
const fake: PaymentGateway = { apiKey: "x", charge: async () => "p_1", refund: async () => {} };
// ❌ TS2739: Type '{ ... }' is missing the following properties from type
//    'PaymentGateway': connection, buildHeaders
```

**Ambient enums — the `declare enum` trap**

```ts
// ── src/types/vendor-enums.d.ts ──────────────────────────────────────────────
declare enum PaymentStatus {
  Pending   = "pending",
  Succeeded = "succeeded",
  Failed    = "failed",
}
```

```ts
// ✅ Type position works fine:
function isTerminal(status: PaymentStatus): boolean {
  return status !== PaymentStatus.Pending;   // ← ⚠️ this is a VALUE access
}
```

`PaymentStatus.Pending` compiles to `PaymentStatus.Pending` in the emitted JS — a real property lookup on a real object that must exist at runtime. If nothing defines a global `PaymentStatus`, you get `ReferenceError`. `declare enum` is only correct when a vendor genuinely exposes that enum object at runtime.

`declare const enum` is different — it is fully inlined at compile time:

```ts
declare const enum RetryStrategy { None = 0, Linear = 1, Exponential = 2 }

const strategy = RetryStrategy.Exponential;
// emits: const strategy = 2;   ← no runtime object needed at all
```

But `declare const enum` is banned under `isolatedModules: true` (which every esbuild/swc/ts-node-swc setup requires), because single-file transpilers cannot see the declaration to inline it. On a modern backend, **prefer a union of string literals over any ambient enum**:

```ts
// ✅ The safe modern equivalent — zero runtime footprint, no isolatedModules problem:
type PaymentStatus = "pending" | "succeeded" | "failed";
```

**Ambient namespaces — grouping and reopening**

```ts
// ── src/types/legacy-globals.d.ts ────────────────────────────────────────────
declare namespace AcmePlatform {
  interface TenantContext {
    tenantId:   string;
    region:     "us-east" | "eu-west" | "ap-south";
    featureFlags: Readonly<Record<string, boolean>>;
  }

  function currentTenant(): TenantContext;
  const SDK_VERSION: string;

  namespace Internal {              // nesting is allowed
    function flushTelemetry(): Promise<void>;
  }
}
```

```ts
const tenant: AcmePlatform.TenantContext = AcmePlatform.currentTenant();  // ✅
await AcmePlatform.Internal.flushTelemetry();                             // ✅
```

Namespaces are largely legacy for organising **your own** code (ES modules replaced them — see `48 — Modules in TypeScript`). But they remain essential for two things: describing SDKs that really do expose nested global objects, and **reopening namespaces that `@types` packages declare** — which is exactly what `declare namespace NodeJS { interface ProcessEnv }` does. That is Concept 7.

### Concept 3 — Script files vs module files: the rule that breaks everything

This is the concept that costs the most debugging time in the entire ambient-declaration space. Read it twice.

TypeScript classifies every file as either a **script** or a **module**, using one rule:

> **If a file contains a top-level `import` or `export`, it is a module. Otherwise it is a script.**

That single bit changes where every declaration in the file lands.

| | **Script file** (no top-level import/export) | **Module file** (has one) |
|---|---|---|
| Declarations go into | the **global** scope | the **module's own** scope |
| Visible from other files | everywhere, no import | only via `import` |
| `declare module "pkg"` means | **ambient module** — types for package `"pkg"` | **module augmentation** — merges into an already-imported module |
| `declare global { }` | ❌ TS1038 — illegal, already global | ✅ required to reach the global scope |

Here is the failure sequence that happens in real life:

```ts
// ── STEP 1 — src/types/globals.d.ts, day one. A SCRIPT. Works. ───────────────
declare const __BUILD_SHA__: string;

declare namespace NodeJS {
  interface ProcessEnv {
    DATABASE_URL: string;
    REDIS_URL:    string;
  }
}

declare module "@acme/legacy-cache" {
  export function get(key: string): string | null;
  export function set(key: string, value: string, ttlSeconds?: number): void;
}
```

Everything above is global. `process.env.DATABASE_URL` is `string` everywhere. `import { get } from "@acme/legacy-cache"` resolves. Perfect.

```ts
// ── STEP 2 — six weeks later, someone needs a type from another file ─────────
import type { Redis } from "ioredis";        // ⚠️⚠️⚠️ FILE IS NOW A MODULE

declare const __BUILD_SHA__: string;         // now private to this file

declare namespace NodeJS {                   // now a LOCAL namespace named NodeJS
  interface ProcessEnv {                     // merges with NOTHING
    DATABASE_URL: string;
    REDIS_URL:    string;
  }
}

declare module "@acme/legacy-cache" {        // now AUGMENTATION of a module that
  export function get(key: string): string | null;   // was never imported here
  export function set(key: string, value: string, ttlSeconds?: number): void;
}

declare const redisClient: Redis;
```

One `import type` line, and the whole file silently stops working globally:

```ts
// ── STEP 3 — the errors, in files nobody touched ─────────────────────────────
console.log(__BUILD_SHA__);
// ❌ TS2304: Cannot find name '__BUILD_SHA__'.

const dbUrl: string = process.env.DATABASE_URL;
// ❌ TS2322: Type 'string | undefined' is not assignable to type 'string'.
//    (back to the @types/node default — your ProcessEnv merge did nothing)

import { get } from "@acme/legacy-cache";
// ❌ TS2664: Invalid module name in augmentation, module '@acme/legacy-cache'
//    cannot be found.   ← or TS7016, depending on setup
```

Three unrelated-looking errors, and `git blame` points at a file whose only change was adding an import. The `TS2664` message is the giveaway: "**Invalid module name in augmentation**" means the compiler treated your `declare module` as an *augmentation* (module file) rather than an *ambient module declaration* (script file).

**The two correct fixes:**

```ts
// ✅ FIX A — keep it a script. Use inline import types instead of top-level imports.
//    `import("...")` in TYPE POSITION does not make the file a module.

declare const __BUILD_SHA__: string;
declare const redisClient: import("ioredis").Redis;    // ← inline, file stays a script

declare namespace NodeJS {
  interface ProcessEnv { DATABASE_URL: string; REDIS_URL: string }
}

declare module "@acme/legacy-cache" {
  export function get(key: string): string | null;
}
```

```ts
// ✅ FIX B — accept module-hood, wrap globals in `declare global`.
import type { Redis } from "ioredis";

declare global {
  const __BUILD_SHA__: string;
  var redisClient: Redis;               // `var` → also available as globalThis.redisClient

  namespace NodeJS {
    interface ProcessEnv { DATABASE_URL: string; REDIS_URL: string }
  }
}

// ⚠️ `declare module "@acme/legacy-cache"` for an UNTYPED package must NOT go here —
//    in a module file it becomes augmentation. Ambient module declarations for
//    untyped packages belong in a separate SCRIPT file.

export {};
```

**Why `export {}` matters.** It is an export statement that exports nothing. Its *entire* purpose is to flip the script/module bit:

```ts
// This file has no imports and no other exports — it would be a SCRIPT.
// `declare global` inside a script is illegal:
declare global {
  var jobQueue: unknown;
}
// ❌ TS1038: A 'declare' modifier cannot be used in an already ambient context.
//    (or: "Augmentations for the global scope can only be directly nested in
//     external modules or ambient module declarations.")

export {};   // ← add this one line and the error disappears
```

So the rule of thumb: **`export {}` and `declare global` travel together.** If you see one without the other in a `.d.ts`, look closely.

**A practical file-layout convention that avoids the whole problem:**

```text
src/types/
  globals.d.ts          ← SCRIPT. Ambient globals + `declare namespace NodeJS`.
                          Never add a top-level import here. Use import("...") inline.
  ambient-modules.d.ts  ← SCRIPT. `declare module "untyped-pkg"` + wildcard modules.
                          Never add a top-level import here either.
  augmentations.d.ts    ← MODULE. `import "express"` + `declare module "express"`
                          + `declare global`. Ends with `export {}`.
```

Put a comment at the top of each script file saying **"DO NOT add top-level imports — this file must remain a script."** It will save someone six weeks from now.

### Concept 4 — `declare global`

`declare global` is a **global augmentation block**. It says: "from inside this module, add the following declarations to the global scope."

```ts
// ── src/types/global-augmentations.d.ts ──────────────────────────────────────
import type { Pool } from "pg";
import type { Logger } from "pino";

declare global {
  // ── Global variables ──────────────────────────────────────────────────────
  // Use `var` if you want globalThis.X to type-check. `const`/`let` give you a
  // bare identifier only.
  var dbPool: Pool;                       // globalThis.dbPool ✅  and  dbPool ✅
  var appLogger: Logger;
  const __BUILD_SHA__: string;            // __BUILD_SHA__ ✅  but globalThis.__BUILD_SHA__ ❌

  // ── Global functions ──────────────────────────────────────────────────────
  function reportRequestMetric(routeName: string, durationMs: number): void;

  // ── Global interfaces / types ─────────────────────────────────────────────
  interface RequestContext {
    requestId: string;
    userId:    number | null;
    authToken: string | null;
    startedAt: number;
  }

  // ── Reopening built-in interfaces ─────────────────────────────────────────
  interface Error {
    // Your error-serialisation polyfill installs this on Error.prototype
    toJSONSafe?(): { name: string; message: string; stack?: string };
  }

  interface Array<T> {
    // A prototype extension a legacy vendor bundle installed. Typing it does not
    // make it exist — this is a description of something that already happened.
    lastOrNull(): T | null;
  }

  // ── Nested namespaces (this is how ProcessEnv typing works) ───────────────
  namespace NodeJS {
    interface ProcessEnv {
      DATABASE_URL: string;
      LOG_LEVEL:    "debug" | "info" | "warn" | "error";
    }
    interface Global {
      requestContext?: RequestContext;
    }
  }
}

export {};   // ← required: makes the file a module so `declare global` is legal
```

Four rules govern `declare global`:

1. **It is only legal inside a module** (or inside an ambient module declaration). In a script file you are already global, so it is an error — TS1038 or TS2669.
2. **The `declare` modifier is implicit inside the block.** Write `var dbPool: Pool`, not `declare var dbPool: Pool` (the latter is TS1038: "A 'declare' modifier cannot be used in an already ambient context").
3. **Interfaces inside it merge; type aliases inside it do not.** `interface Error { ... }` reopens the built-in. `type Error = ...` is TS2300 "Duplicate identifier".
4. **The file must be included in the compilation.** A `declare global` in a file `tsc` never reads has zero effect and produces no warning.

**Where `declare global` earns its keep on a backend** — the classic case is Express:

```ts
// ── src/types/express.d.ts ───────────────────────────────────────────────────
import "express";                          // ← makes it a module AND loads the base types

declare global {
  namespace Express {
    interface Request {
      userId?:    number;
      authToken?: string;
      requestId:  string;
    }
  }
}
```

```ts
// ── src/middleware/authenticate.ts ───────────────────────────────────────────
import type { Request, Response, NextFunction } from "express";
import { randomUUID } from "node:crypto";

export function authenticate(req: Request, res: Response, next: NextFunction): void {
  req.requestId = randomUUID();                        // ✅ typed
  const header = req.headers.authorization;
  if (!header?.startsWith("Bearer ")) {
    res.status(401).json({ error: "Missing bearer token" });
    return;
  }
  req.authToken = header.slice("Bearer ".length);      // ✅ typed
  req.userId    = decodeUserId(req.authToken);         // ✅ typed
  next();
}

declare function decodeUserId(authToken: string): number;
```

`@types/express` declares `namespace Express { interface Request }` in the *global* namespace, which is why you augment it through `declare global` rather than `declare module "express"`. Full treatment in `53 — Extending Express Request` and `51 — Module augmentation`.

**A safety note on `var` in `declare global`.** ESLint's `no-var` rule will fight you. The `var` is semantically required — `let`/`const` do not add a `globalThis` property in TypeScript's model. Configure the exception rather than changing the keyword:

```jsonc
// eslint.config.js — allow `var` inside .d.ts files
{
  "files": ["**/*.d.ts"],
  "rules": { "no-var": "off" }
}
```

### Concept 5 — Typing globals injected at runtime

This is the concept with the most real-world backend surface area. Runtime-injected globals arrive by four distinct mechanisms, and each has a matching declaration pattern.

**Mechanism A — bundler `--define` / `DefinePlugin` (compile-time constant substitution)**

```jsonc
// package.json
{
  "scripts": {
    "build": "esbuild src/server.ts --bundle --platform=node --outfile=dist/server.js --define:__BUILD_SHA__='\"'$(git rev-parse HEAD)'\"' --define:__BUILD_TIME_MS__=$(date +%s000)"
  }
}
```

```ts
// ── src/types/build-constants.d.ts (SCRIPT — no imports) ─────────────────────
/** Git commit SHA, substituted textually by esbuild at build time. */
declare const __BUILD_SHA__: string;
/** Unix epoch millis at build time. */
declare const __BUILD_TIME_MS__: number;
```

```ts
// ── src/routes/health.ts ─────────────────────────────────────────────────────
import type { Request, Response } from "express";

interface HealthResponse {
  status:      "ok";
  buildSha:    string;
  builtAt:     string;
  uptimeSecs:  number;
}

export function handleHealth(_req: Request, res: Response<HealthResponse>): void {
  res.json({
    status:     "ok",
    buildSha:   __BUILD_SHA__.slice(0, 7),                       // ✅ string methods
    builtAt:    new Date(__BUILD_TIME_MS__).toISOString(),       // ✅ number
    uptimeSecs: Math.floor(process.uptime()),
  });
}
```

⚠️ **The dev-environment trap.** `ts-node`/`tsx`/`jest` do not apply esbuild's `--define`. Under those runners `__BUILD_SHA__` is genuinely undefined and you get a `ReferenceError`. The fix is a guarded accessor:

```ts
// ── src/build-info.ts — a real .ts file that handles both worlds ─────────────
export function buildSha(): string {
  // `typeof x === "undefined"` is the ONE safe way to test an undeclared global —
  // it does not throw ReferenceError.
  return typeof __BUILD_SHA__ === "undefined" ? "dev-local" : __BUILD_SHA__;
}
```

**Mechanism B — a `--require` preload hook attaching to `globalThis`**

```js
// ── instrument.js — loaded via: node --require ./instrument.js dist/server.js
const { createTracer } = require("@acme/apm-agent");
globalThis.__tracer = createTracer({ service: process.env.SERVICE_NAME });
globalThis.__startedAtMs = Date.now();
```

```ts
// ── src/types/instrumentation.d.ts ───────────────────────────────────────────
interface TraceSpan {
  setAttribute(key: string, value: string | number | boolean): void;
  recordException(error: Error): void;
  end(): void;
}

interface RequestTracer {
  startSpan(name: string, attributes?: Record<string, string>): TraceSpan;
  record(metricName: string, value?: number): void;
  activeTraceId(): string | null;
}

declare global {
  /**
   * Installed by instrument.js via `node --require`.
   * Undefined if the preload was skipped — see `hasTracer()` in src/tracing.ts.
   */
  var __tracer: RequestTracer;
  var __startedAtMs: number;
}

export {};
```

Because it is `var`, both access styles type-check:

```ts
globalThis.__tracer.record("orders.created");   // ✅
__tracer.record("orders.created");              // ✅ bare identifier also works
```

The honest-typing question: should `__tracer` be `RequestTracer` or `RequestTracer | undefined`? Declaring it non-optional is a lie whenever the preload can be skipped (dev, tests, a misconfigured Dockerfile). Declaring it optional is truthful but forces `?.` at every call site. The professional pattern is **declare it optional and provide a checked accessor**:

```ts
// ── src/types/instrumentation.d.ts ───────────────────────────────────────────
declare global {
  var __tracer: RequestTracer | undefined;    // honest
}
export {};
```

```ts
// ── src/tracing.ts — one place handles the uncertainty ───────────────────────
const NOOP_SPAN: TraceSpan = {
  setAttribute() {}, recordException() {}, end() {},
};

const NOOP_TRACER: RequestTracer = {
  startSpan: () => NOOP_SPAN,
  record:    () => {},
  activeTraceId: () => null,
};

/** Always returns a usable tracer — real one in prod, no-op in dev/test. */
export function tracer(): RequestTracer {
  return globalThis.__tracer ?? NOOP_TRACER;
}
```

```ts
// Call sites are clean AND safe:
import { tracer } from "../tracing.js";
const span = tracer().startSpan("orders.create");
```

**Mechanism C — test setup files sharing state**

```ts
// ── test/setup.ts ────────────────────────────────────────────────────────────
import { newDb } from "pg-mem";
import type { Pool } from "pg";

beforeAll(async () => {
  globalThis.testDbPool = newDb().adapters.createPg().Pool as unknown as Pool;
  globalThis.testAuthToken = "test_" + "a".repeat(32);
});

afterAll(async () => {
  await globalThis.testDbPool.end();
});
```

```ts
// ── test/types/test-globals.d.ts ─────────────────────────────────────────────
import type { Pool } from "pg";

declare global {
  var testDbPool: Pool;
  var testAuthToken: string;
}

export {};
```

Keep this file inside the test-only tsconfig's `include` so production code cannot accidentally reference `testDbPool`:

```jsonc
// tsconfig.json — app code, does NOT include test/
{ "include": ["src/**/*.ts"], "exclude": ["src/**/*.test.ts"] }

// tsconfig.test.json — test code only
{
  "extends": "./tsconfig.json",
  "compilerOptions": { "types": ["node", "jest"] },
  "include": ["src/**/*.ts", "test/**/*.ts"]
}
```

**Mechanism D — platform-injected context (serverless, edge runtimes, workers)**

```ts
// ── src/types/platform.d.ts ──────────────────────────────────────────────────
interface PlatformRequestContext {
  readonly awsRequestId:      string;
  readonly functionVersion:   string;
  readonly remainingTimeMs:   () => number;
  readonly invokedFunctionArn: string;
}

declare global {
  /** Injected by the runtime shim before the handler module is evaluated. */
  var __REQUEST_CONTEXT__: PlatformRequestContext | undefined;
}

export {};
```

```ts
// ── src/handlers/process-order.ts ────────────────────────────────────────────
export async function handler(event: { orderId: string }): Promise<{ statusCode: number }> {
  const ctx = globalThis.__REQUEST_CONTEXT__;

  // ✅ `| undefined` forces the check — which is correct, because local
  //    invocations via `node --eval` genuinely don't have it.
  const budgetMs = ctx?.remainingTimeMs() ?? 30_000;

  if (budgetMs < 2_000) {
    return { statusCode: 503 };   // not enough time; let the platform retry
  }
  await processOrder(event.orderId, budgetMs - 1_000);
  return { statusCode: 200 };
}

declare function processOrder(orderId: string, budgetMs: number): Promise<void>;
```

### Concept 6 — Ambient modules: `declare module "untyped-pkg"`

An **ambient module declaration** attaches a type shape to a bare import specifier. It is how you type a package you do not own and cannot modify.

```ts
// ── src/types/ambient-modules.d.ts (SCRIPT — no top-level imports!) ──────────

declare module "@acme/legacy-cache" {
  export interface CacheOptions {
    ttlSeconds?: number;
    namespace?:  string;
  }

  export function get(key: string): Promise<string | null>;
  export function set(key: string, value: string, options?: CacheOptions): Promise<void>;
  export function del(key: string): Promise<boolean>;
  export const DEFAULT_TTL_SECONDS: number;
}
```

```ts
// ── src/services/session-service.ts ──────────────────────────────────────────
import { get, set, DEFAULT_TTL_SECONDS } from "@acme/legacy-cache";

interface SessionRecord {
  userId:    number;
  authToken: string;
  expiresAt: string;
}

export async function loadSession(authToken: string): Promise<SessionRecord | null> {
  const raw = await get(`session:${authToken}`);   // ✅ Promise<string | null>
  if (raw === null) return null;                   // ✅ null branch forced by the type
  return JSON.parse(raw) as SessionRecord;
}

export async function saveSession(session: SessionRecord): Promise<void> {
  await set(`session:${session.authToken}`, JSON.stringify(session), {
    ttlSeconds: DEFAULT_TTL_SECONDS,
    // ❌ namesapce: "auth",
    //    TS2561: Did you mean to write 'namespace'?
  });
}
```

**The specifier must match the import exactly — it is a literal string match, not a path resolution.**

```ts
declare module "@acme/legacy-cache" { /* ... */ }

import { get } from "@acme/legacy-cache";          // ✅ matches
import { get } from "@acme/legacy-cache/redis";    // ❌ different specifier, unmatched
import { get } from "./legacy-cache";              // ❌ relative — never matches ambient
```

Subpaths need their own declarations (or a wildcard):

```ts
declare module "@acme/legacy-cache" { export function get(key: string): Promise<string | null> }
declare module "@acme/legacy-cache/redis" { export function connect(url: string): Promise<void> }
declare module "@acme/legacy-cache/*";   // ← wildcard fallback: any other subpath is `any`
```

**The shorthand form — `any` for the whole module:**

```ts
declare module "@acme/really-gnarly-sdk";
// Every import from it is `any`. No error, no safety.
```

Use this only as a deliberate, grep-able, temporary escape hatch — and *never* reach for `noImplicitAny: false` instead, which disables the check globally:

```ts
// ✅ Honest, scoped, and it leaves a paper trail:
// TODO(PLAT-4127): replace with real types once we fork the SDK. — @parshuram 2026-03
declare module "@acme/really-gnarly-sdk";
```

**Modelling CommonJS export styles.** Match what the JavaScript actually does:

```ts
// JS: module.exports = { encode, decode }   → named exports
declare module "@acme/cursor-codec" {
  export function encode(record: { id: number }, sortField: string): string;
  export function decode(cursor: string): { id: number; v: unknown } | null;
}

// JS: module.exports = function createLimiter(opts) {...}   → callable default
declare module "@acme/rate-limiter" {
  interface LimiterOptions { windowMs: number; maxRequests: number }
  interface RateLimiter { check(key: string): boolean; reset(key: string): void }
  function createLimiter(options: LimiterOptions): RateLimiter;
  export = createLimiter;                 // mirrors `module.exports = fn`
}

// JS: module.exports = fn; module.exports.VERSION = "2.1.0";  → callable + properties
declare module "@acme/audit-logger" {
  interface AuditOptions { batchSize?: number }
  class AuditLogger {
    constructor(url: string, options?: AuditOptions);
    log(actorId: number, action: string): void;
    flush(): Promise<number>;
  }
  function createLogger(url: string, options?: AuditOptions): AuditLogger;
  namespace createLogger {                // declaration merge: function + namespace
    export { AuditLogger, AuditOptions };
    export const VERSION: string;
  }
  export = createLogger;
}
```

**Ambient module vs module augmentation — the same syntax, two meanings.** This is the trap from Concept 3, restated because it matters:

```ts
// ── In a SCRIPT file (no top-level import/export) ────────────────────────────
declare module "@acme/legacy-cache" {   // ← DEFINES types for an untyped package
  export function get(key: string): Promise<string | null>;
}

// ── In a MODULE file (has a top-level import/export) ─────────────────────────
import "@acme/legacy-cache";            // ← must already have types
declare module "@acme/legacy-cache" {   // ← AUGMENTS them (adds/merges members)
  export function getWithMetadata(key: string): Promise<{ value: string; ttl: number }>;
}
export {};
```

If you write the second form for a package that has *no* types, you get:

```text
TS2664: Invalid module name in augmentation, module '@acme/legacy-cache' cannot be found.
```

That error message is your signal: **"my file became a module and I did not mean it to."** Look for a stray top-level `import` or `export`.

### Concept 7 — Wildcard modules for non-code imports

A wildcard ambient module declaration uses `*` in the specifier to match a family of import paths. This is how you type imports of things that are not JavaScript.

```ts
// ── src/types/asset-modules.d.ts (SCRIPT) ────────────────────────────────────

/** GraphQL SDL loaded as a raw string by the bundler/loader. */
declare module "*.graphql" {
  const source: string;
  export default source;
}

/** Raw .sql files imported as query strings. */
declare module "*.sql" {
  const query: string;
  export default query;
}

/** Handlebars/Mustache email templates. */
declare module "*.hbs" {
  const template: string;
  export default template;
}

/** Vite/esbuild `?raw` query suffix. */
declare module "*?raw" {
  const contents: string;
  export default contents;
}

/** Protobuf descriptors compiled to a JSON-ish shape by a loader. */
declare module "*.proto" {
  const descriptor: { nested: Record<string, unknown> };
  export default descriptor;
}
```

```ts
// ── src/graphql/schema.ts ────────────────────────────────────────────────────
import userTypeDefs  from "./schemas/user.graphql";      // ✅ string
import orderTypeDefs from "./schemas/order.graphql";     // ✅ string
import { buildSchema } from "graphql";

export const schema = buildSchema([userTypeDefs, orderTypeDefs].join("\n"));

// ❌ userTypeDefs.definitions
//    TS2339: Property 'definitions' does not exist on type 'string'.
```

```ts
// ── src/repositories/order-repository.ts ─────────────────────────────────────
import findOrdersByUser from "./queries/find-orders-by-user.sql";
import type { Pool } from "pg";

interface OrderRow {
  order_id:    number;
  user_id:     number;
  total_cents: number;
  created_at:  Date;
}

export async function listOrdersForUser(pool: Pool, userId: number): Promise<OrderRow[]> {
  const result = await pool.query<OrderRow>(findOrdersByUser, [userId]);  // ✅ string
  return result.rows;
}
```

**Four rules for wildcards:**

1. **At most one `*` per specifier.** `declare module "*.graphql"` ✅. `declare module "*/*.graphql"` ❌ (TS2436).
2. **The most specific match wins.** `"@acme/sdk/client"` beats `"@acme/sdk/*"` beats `"*"`.
3. **The declaration alone does not make it work at runtime.** You still need a loader (esbuild plugin, `--loader` flag, webpack rule, `node --experimental-*`). The declaration only satisfies the type checker. Getting this backwards produces a clean build and `SyntaxError: Unexpected token` at startup.
4. **JSON is different.** `.json` imports are handled by `resolveJsonModule: true`, which gives you the *actual literal type of the file's contents* — far better than a wildcard.

```jsonc
// ✅ tsconfig.json — real types for JSON, no ambient declaration needed
{ "compilerOptions": { "resolveJsonModule": true, "esModuleInterop": true } }
```

```ts
// ── src/config/limits.json ───────────────────────────────────────────────────
// { "maxOrderCents": 5000000, "maxItemsPerOrder": 50, "allowedRegions": ["us", "eu"] }

import limits from "../config/limits.json";

limits.maxOrderCents;        // ✅ number — inferred from the actual file contents
limits.allowedRegions[0];    // ✅ string
limits.maxOrdercents;        // ❌ TS2551: Did you mean 'maxOrderCents'?
```

If you write `declare module "*.json" { const value: any; export default value }` you **destroy** that inference and turn every JSON import into `any`. Never do it.

**Making a wildcard less blunt.** `declare module "*.graphql"` types every GraphQL import identically. If you want per-file precision, declare the exact specifier — it wins over the wildcard:

```ts
declare module "*.graphql" {
  const source: string;
  export default source;
}

// A specific file that a codegen step turns into a typed document node:
declare module "./schemas/user.graphql" {
  import type { DocumentNode } from "graphql";   // inline-import style keeps it clean
  const document: DocumentNode;
  export default document;
}
```

### Concept 8 — `declare namespace NodeJS { interface ProcessEnv }`

The highest-value ambient declaration on a Node backend, and the one with the most subtlety.

**The default state.** `@types/node` declares:

```ts
// node_modules/@types/node/process.d.ts (simplified)
declare namespace NodeJS {
  interface ProcessEnv {
    [key: string]: string | undefined;      // ← the index signature
  }
  interface Process {
    env: ProcessEnv;
    // ...
  }
}
declare var process: NodeJS.Process;
```

Consequences:

```ts
process.env.DATABASE_URL;     // string | undefined
process.env.DATBASE_URL;      // string | undefined  ← typo compiles fine
process.env.ANY_NONSENSE;     // string | undefined  ← index signature accepts everything
```

**The augmentation.** Because `ProcessEnv` is an `interface` inside a global `namespace`, you can reopen and merge into it:

```ts
// ── src/types/env.d.ts (SCRIPT — no top-level imports) ───────────────────────
declare namespace NodeJS {
  interface ProcessEnv {
    // ── Required in every environment ──────────────────────────────────────
    NODE_ENV:     "development" | "test" | "production";
    PORT:         string;
    DATABASE_URL: string;
    REDIS_URL:    string;
    JWT_SECRET:   string;

    // ── Optional / environment-specific ────────────────────────────────────
    SENTRY_DSN?:      string;
    LOG_LEVEL?:       "debug" | "info" | "warn" | "error";
    MAX_ORDER_CENTS?: string;
  }
}
```

```ts
// ── src/config.ts ────────────────────────────────────────────────────────────
export interface AppConfig {
  readonly nodeEnv:        "development" | "test" | "production";
  readonly port:           number;
  readonly databaseUrl:    string;
  readonly redisUrl:       string;
  readonly jwtSecret:      string;
  readonly sentryDsn:      string | null;
  readonly logLevel:       "debug" | "info" | "warn" | "error";
  readonly maxOrderCents:  number;
}

export function loadConfig(): AppConfig {
  return {
    nodeEnv:       process.env.NODE_ENV,                       // ✅ the literal union
    port:          Number(process.env.PORT),                   // ✅ string, no `?? "3000"`
    databaseUrl:   process.env.DATABASE_URL,                   // ✅ string
    redisUrl:      process.env.REDIS_URL,                      // ✅ string
    jwtSecret:     process.env.JWT_SECRET,                     // ✅ string
    sentryDsn:     process.env.SENTRY_DSN ?? null,             // ✅ optional → must handle
    logLevel:      process.env.LOG_LEVEL ?? "info",            // ✅ union narrowed
    maxOrderCents: Number(process.env.MAX_ORDER_CENTS ?? "5000000"),
  };
}

// ❌ Errors you now get for free:
// process.env.DATBASE_URL
//   TS2339: Property 'DATBASE_URL' does not exist on type 'ProcessEnv'.
//           Did you mean 'DATABASE_URL'?
// const env: "staging" = process.env.NODE_ENV;
//   TS2322: Type '"development" | "test" | "production"' is not assignable to '"staging"'.
```

Wait — how does `TS2339` fire if `ProcessEnv` has an index signature `[key: string]: string | undefined`? Because **the index signature only applies when a specific property is not declared, and TypeScript reports unknown property access on interfaces with index signatures via dot notation only when the index signature is absent**. In practice: with `@types/node`'s index signature still present, `process.env.DATBASE_URL` resolves through the index signature and gives `string | undefined` — **no error**.

That is the crucial catch, and it means the naive augmentation gives you *narrowing* (declared vars become `string` not `string | undefined`) but **not typo detection**. To get both, you must remove the index signature — which you cannot do by merging. You do it with a separate typed accessor:

```ts
// ── src/types/env.d.ts ───────────────────────────────────────────────────────
interface StrictProcessEnv {
  NODE_ENV:     "development" | "test" | "production";
  PORT:         string;
  DATABASE_URL: string;
  REDIS_URL:    string;
  JWT_SECRET:   string;
  SENTRY_DSN?:  string;
}

declare namespace NodeJS {
  // Merge for the narrowing benefit...
  interface ProcessEnv extends StrictProcessEnv {}
}
```

```ts
// ── src/env.ts — the typo-proof accessor ─────────────────────────────────────
/**
 * Use this instead of `process.env` everywhere. It has NO index signature,
 * so unknown keys are a compile error.
 */
export const env: Readonly<StrictProcessEnv> = process.env;

// ❌ env.DATBASE_URL
//    TS2339: Property 'DATBASE_URL' does not exist on type 'Readonly<StrictProcessEnv>'.
//            Did you mean 'DATABASE_URL'?
```

Then ban raw `process.env` with a lint rule:

```jsonc
// eslint.config.js
{
  "rules": {
    "no-restricted-properties": ["error", {
      "object":   "process",
      "property": "env",
      "message":  "Import { env } from '@/env' instead — it is typo-checked."
    }]
  }
}
```

**The honesty problem, stated plainly.** Declaring `DATABASE_URL: string` does not make the variable exist. If the container starts without it, `process.env.DATABASE_URL` is `undefined` at runtime while TypeScript insists it is `string`, and you get `TypeError: Cannot read properties of undefined` deep inside the pg driver — or worse, a connection string of literally `"undefined"`.

The complete professional pattern pairs the ambient declaration with **runtime validation at startup**:

```ts
// ── src/env.ts ───────────────────────────────────────────────────────────────
import { z } from "zod";

const EnvSchema = z.object({
  NODE_ENV:     z.enum(["development", "test", "production"]),
  PORT:         z.string().regex(/^\d+$/),
  DATABASE_URL: z.string().url(),
  REDIS_URL:    z.string().url(),
  JWT_SECRET:   z.string().min(32),
  SENTRY_DSN:   z.string().url().optional(),
});

/** Parsed ONCE at module load. Crashes the process immediately if invalid. */
export const env = EnvSchema.parse(process.env);
export type Env = z.infer<typeof EnvSchema>;
// env.DATABASE_URL is `string` — and it is TRUE, because we checked.
```

With that in place the ambient `ProcessEnv` augmentation becomes an editor-ergonomics feature (autocomplete on `process.env.` inside the schema file and in shell-adjacent code), while `env` is the checked source of truth. Full treatment in `55 — Typing environment variables`.

### Concept 9 — Getting `tsc` to include your ambient files

An ambient declaration that `tsc` never reads has **exactly zero effect and produces no warning**. This is the most common reason "my `declare global` isn't working."

**Rule 1 — `include` globs match `.d.ts`.** `"include": ["src/**/*.ts"]` matches `src/types/globals.d.ts`, because `.d.ts` ends in `.ts`. If your declarations live outside `src/`, they are not included:

```jsonc
// ❌ types/globals.d.ts is invisible
{ "include": ["src/**/*.ts"] }

// ✅ Option A — move the file under src/
//    src/types/globals.d.ts

// ✅ Option B — widen the glob
{ "include": ["src/**/*.ts", "types/**/*.d.ts"] }

// ✅ Option C — force it with `files` (survives any future include change)
{
  "files":   ["types/globals.d.ts"],
  "include": ["src/**/*.ts"]
}
```

**Rule 2 — `files` and `include` are unioned.** Listing something in `files` adds it; it does not replace `include`.

**Rule 3 — `exclude` cannot rescue a file that `include` never matched,** and `exclude` does not apply to `files`.

**Rule 4 — `typeRoots` is a different mechanism and usually not what you want.** `typeRoots` controls where the compiler looks for *auto-included type packages* — directories shaped like `@types/<name>/index.d.ts`. A loose `src/types/globals.d.ts` is not a type package. Adding `"typeRoots": ["./src/types"]` does **not** include it, and worse, it *replaces* the default `["./node_modules/@types"]`, breaking `@types/node`:

```jsonc
// ❌ A very common self-inflicted wound
{
  "compilerOptions": {
    "typeRoots": ["./src/types"]   // ← @types/node is now NOT auto-included
  }
}
// → TS2580: Cannot find name 'process'. Do you need to install type definitions for node?
```

```jsonc
// ✅ If you use typeRoots at all, always keep node_modules/@types:
{ "compilerOptions": { "typeRoots": ["./node_modules/@types", "./src/types"] } }
```

If you genuinely want a directory to work as a type root, structure it like a package:

```text
src/types/
  acme-platform/
    index.d.ts        ← declare namespace AcmePlatform { ... }
```

```jsonc
{
  "compilerOptions": {
    "typeRoots": ["./node_modules/@types", "./src/types"],
    "types":     ["node", "acme-platform"]      // ← name matches the DIRECTORY
  }
}
```

**Rule 5 — `"types": [...]` restricts auto-inclusion, not your own files.** Setting `"types": ["node"]` stops `@types/jest` globals leaking into app code. It has no effect on `src/types/globals.d.ts`, which comes in via `include`.

**Rule 6 — triple-slash references are the last resort.** They work but create hidden coupling:

```ts
// ── src/server.ts ────────────────────────────────────────────────────────────
/// <reference path="./types/globals.d.ts" />
/// <reference types="node" />
```

Prefer `include`/`files`. Use `/// <reference types="..." />` only inside a published `.d.ts` that genuinely depends on another types package.

**Rule 7 — verify, do not assume.** This is the command to memorise:

```bash
# Is my declaration file actually being read?
npx tsc --noEmit --listFiles | grep globals.d.ts
# no output → tsc is not reading it. Fix include/files.

# What is the FULL set of files in the program?
npx tsc --noEmit --listFiles | wc -l

# Why is some file included? (walks the reference chain)
npx tsc --noEmit --explainFiles | grep -A 3 globals.d.ts

# Which tsconfig is actually in effect, fully resolved?
npx tsc --showConfig
```

**Rule 8 — editors and `tsc` can disagree.** VS Code uses the nearest `tsconfig.json` to the open file; your build may use `tsconfig.build.json`. A `declare global` that works in the editor and fails in CI almost always means the CI config's `include` does not cover the file. Always reproduce with `npx tsc --noEmit -p tsconfig.build.json` before debugging anything else.

---

## Example 1 — basic

```ts
// ══════════════════════════════════════════════════════════════════════════════
// Typing everything a small Node service gets from outside TypeScript's view:
//   - a build-time constant injected by esbuild
//   - a global attached by a --require preload
//   - a strongly typed process.env
//   - an untyped internal npm package
//   - raw .sql file imports
// ══════════════════════════════════════════════════════════════════════════════

// ─────────────────────────────────────────────────────────────────────────────
// FILE: src/types/globals.d.ts
// ⚠️  THIS FILE MUST REMAIN A SCRIPT — do NOT add top-level import/export.
//     Use inline `import("pkg").Type` if you need a type from a module.
// ─────────────────────────────────────────────────────────────────────────────

/** Git SHA, substituted by `esbuild --define:__BUILD_SHA__=...` at build time. */
declare const __BUILD_SHA__: string;

/** Build timestamp in epoch millis, also from --define. */
declare const __BUILD_TIME_MS__: number;

/** Shape of the health-probe hook installed by ./preload.js. */
interface HealthProbe {
  markReady(): void;
  markUnhealthy(reason: string): void;
  isReady(): boolean;
}

/**
 * `var` (not `const`) so that BOTH `__healthProbe` and
 * `globalThis.__healthProbe` type-check.
 * Optional because the preload is skipped under `tsx watch` in dev.
 */
declare var __healthProbe: HealthProbe | undefined;

/** Strongly typed environment contract for this service. */
declare namespace NodeJS {
  interface ProcessEnv {
    NODE_ENV:     "development" | "test" | "production";
    PORT:         string;
    DATABASE_URL: string;
    LOG_LEVEL?:   "debug" | "info" | "warn" | "error";
  }
}

// ─────────────────────────────────────────────────────────────────────────────
// FILE: src/types/ambient-modules.d.ts
// ⚠️  ALSO A SCRIPT. `declare module "x"` here DEFINES types for untyped
//     packages. If this file becomes a module these turn into augmentations
//     and you get TS2664.
// ─────────────────────────────────────────────────────────────────────────────

/**
 * @acme/request-id — plain JS, no types.
 * node_modules/@acme/request-id/index.js:
 *   exports.generate = () => "req_" + crypto.randomBytes(8).toString("hex");
 *   exports.isValid  = (id) => /^req_[0-9a-f]{16}$/.test(id);
 *   exports.PREFIX   = "req_";
 */
declare module "@acme/request-id" {
  /** Generate a new opaque request identifier. */
  export function generate(): string;
  /** True if `candidate` is a well-formed request id. */
  export function isValid(candidate: string): boolean;
  /** The literal prefix every id carries. */
  export const PREFIX: string;
}

/** Raw .sql files, loaded as strings by the esbuild `text` loader. */
declare module "*.sql" {
  const query: string;
  export default query;
}

// ─────────────────────────────────────────────────────────────────────────────
// FILE: src/server.ts — every declaration above put to work
// ─────────────────────────────────────────────────────────────────────────────
import { generate, isValid } from "@acme/request-id";
import findActiveUsers from "./queries/find-active-users.sql";

interface ActiveUserRow {
  user_id:       number;
  email:         string;
  last_seen_at:  Date;
}

interface ApiResponse<TBody> {
  statusCode: number;
  requestId:  string;
  body:       TBody;
}

// ✅ __BUILD_SHA__ is `string` — slice() available, no ReferenceError-guard noise
//    in prod. (Dev safety handled by the typeof check below.)
const BUILD_LABEL =
  typeof __BUILD_SHA__ === "undefined" ? "dev-local" : __BUILD_SHA__.slice(0, 7);

// ✅ process.env.DATABASE_URL is `string`, not `string | undefined`
const DATABASE_URL: string = process.env.DATABASE_URL;

// ✅ LOG_LEVEL is optional → the `??` is required by the type, not by superstition
const LOG_LEVEL = process.env.LOG_LEVEL ?? "info";

export async function handleListActiveUsers(
  incomingRequestId: string | undefined,
): Promise<ApiResponse<ActiveUserRow[]>> {
  // ✅ isValid takes a string; the `?? generate()` satisfies it
  const requestId =
    incomingRequestId && isValid(incomingRequestId) ? incomingRequestId : generate();

  // ✅ findActiveUsers is `string` thanks to the "*.sql" wildcard module
  const rows = await runQuery<ActiveUserRow>(findActiveUsers, [30]);

  // ✅ `| undefined` in the declaration forces the optional-chain — honest typing
  globalThis.__healthProbe?.markReady();

  return { statusCode: 200, requestId, body: rows };
}

declare function runQuery<TRow>(sql: string, params: unknown[]): Promise<TRow[]>;

// ─────────────────────────────────────────────────────────────────────────────
// ❌ Everything the ambient declarations now catch
// ─────────────────────────────────────────────────────────────────────────────
// __BUILD_SHA__.toFixed(2);
//   TS2339: Property 'toFixed' does not exist on type 'string'.
//
// __BUILD_SHA__ = "override";
//   TS2588: Cannot assign to '__BUILD_SHA__' because it is a constant.
//
// process.env.DATBASE_URL;
//   → `string | undefined` (index signature still present — see Concept 8).
//     Use the src/env.ts accessor pattern for real typo detection.
//
// const port: number = process.env.PORT;
//   TS2322: Type 'string' is not assignable to type 'number'.
//
// process.env.NODE_ENV === "staging";
//   TS2367: This comparison appears unintentional; the types
//           '"development" | "test" | "production"' and '"staging"' have no overlap.
//
// globalThis.__healthProbe.markReady();
//   TS18048: '__healthProbe' is possibly 'undefined'.
//
// import { genrate } from "@acme/request-id";
//   TS2305: Module '"@acme/request-id"' has no exported member 'genrate'.
//
// isValid(42);
//   TS2345: Argument of type 'number' is not assignable to parameter of type 'string'.
//
// findActiveUsers.query;
//   TS2339: Property 'query' does not exist on type 'string'.

// ─────────────────────────────────────────────────────────────────────────────
// FILE: tsconfig.json — make sure tsc reads all of the above
// ─────────────────────────────────────────────────────────────────────────────
// {
//   "compilerOptions": {
//     "strict":        true,
//     "target":        "ES2022",
//     "module":        "NodeNext",
//     "lib":           ["ES2022"],       // no DOM — this is a server
//     "types":         ["node"],
//     "skipLibCheck":  true,
//     "outDir":        "./dist"
//   },
//   "include": ["src/**/*.ts"]           // ← matches src/types/*.d.ts
// }
//
// Verify:
//   npx tsc --noEmit --listFiles | grep -E "globals|ambient-modules"
```

---

## Example 2 — real world backend use case

```ts
// ══════════════════════════════════════════════════════════════════════════════
// A complete ambient-declaration layer for an Express order service.
//
// Runtime facts TypeScript cannot see on its own:
//   1. `node --require ./preload.js` installs globalThis.__tracer and
//      globalThis.__featureFlags before any app module loads.
//   2. esbuild --define injects __BUILD_SHA__ / __BUILD_TIME_MS__.
//   3. An internal package @acme/idempotency ships plain JS with no types.
//   4. SQL and GraphQL files are imported as raw strings by a loader.
//   5. Express's Request must carry userId / authToken / requestId.
//   6. process.env is the deployment contract for this service.
//
// File layout (the convention that prevents script/module accidents):
//   src/types/globals.d.ts          — SCRIPT: bare globals + NodeJS namespace
//   src/types/ambient-modules.d.ts  — SCRIPT: untyped pkgs + wildcard modules
//   src/types/augmentations.d.ts    — MODULE: declare global + Express, ends `export {}`
// ══════════════════════════════════════════════════════════════════════════════


// ═════════════════════════════════════════════════════════════════════════════
// FILE 1 — src/types/globals.d.ts
// ⚠️ SCRIPT FILE. Adding a top-level `import` or `export` here silently breaks
//    every declaration below. Use `import("pkg").Type` inline instead.
// ═════════════════════════════════════════════════════════════════════════════

// ── Build-time constants (esbuild --define) ─────────────────────────────────

/** Full git SHA of the build. Substituted textually; undefined under tsx/jest. */
declare const __BUILD_SHA__: string;
/** Build time in epoch millis. Substituted textually. */
declare const __BUILD_TIME_MS__: number;

// ── The deployment environment contract ─────────────────────────────────────

declare namespace NodeJS {
  interface ProcessEnv {
    // Required in every environment — deploy fails without them
    NODE_ENV:          "development" | "test" | "production";
    SERVICE_NAME:      string;
    PORT:              string;
    DATABASE_URL:      string;
    REDIS_URL:         string;
    JWT_SECRET:        string;
    INVENTORY_API_URL: string;

    // Optional — features degrade gracefully without them
    SENTRY_DSN?:       string;
    OTEL_ENDPOINT?:    string;
    LOG_LEVEL?:        "debug" | "info" | "warn" | "error";
    MAX_ORDER_CENTS?:  string;
  }
}


// ═════════════════════════════════════════════════════════════════════════════
// FILE 2 — src/types/ambient-modules.d.ts
// ⚠️ ALSO A SCRIPT FILE. `declare module "x"` here DEFINES types for packages
//    that have none. In a module file these become augmentations → TS2664.
// ═════════════════════════════════════════════════════════════════════════════

/**
 * @acme/idempotency — internal CJS package, no types.
 *
 * node_modules/@acme/idempotency/index.js:
 *   class IdempotencyStore {
 *     constructor(redisUrl, opts) { ... }
 *     async claim(key, ttlSeconds) { ... }   // → { claimed: bool, existingResult: any|null }
 *     async complete(key, result) { ... }    // → void
 *     async release(key) { ... }             // → void
 *   }
 *   function createStore(redisUrl, opts) { return new IdempotencyStore(redisUrl, opts); }
 *   module.exports = createStore;
 *   module.exports.IdempotencyStore = IdempotencyStore;
 *   module.exports.DEFAULT_TTL_SECONDS = 86400;
 */
declare module "@acme/idempotency" {
  export interface StoreOptions {
    /** Key namespace prefix. @default "idem:" */
    keyPrefix?: string;
    /** Redis command timeout. @default 2000 */
    timeoutMs?: number;
  }

  /** Result of attempting to claim an idempotency key. */
  export interface ClaimResult<TResult = unknown> {
    /** true → you own the key and must call complete(); false → replay. */
    claimed: boolean;
    /** Present only when `claimed` is false and the original call finished. */
    existingResult: TResult | null;
  }

  export class IdempotencyStore {
    constructor(redisUrl: string, options?: StoreOptions);
    claim<TResult = unknown>(key: string, ttlSeconds?: number): Promise<ClaimResult<TResult>>;
    complete<TResult>(key: string, result: TResult): Promise<void>;
    release(key: string): Promise<void>;
  }

  function createStore(redisUrl: string, options?: StoreOptions): IdempotencyStore;

  // Declaration merge: the callable default PLUS its attached properties,
  // mirroring `module.exports = createStore; module.exports.X = ...`
  namespace createStore {
    export { IdempotencyStore, StoreOptions, ClaimResult };
    export const DEFAULT_TTL_SECONDS: number;
  }

  export = createStore;
}

// ── Non-code imports ────────────────────────────────────────────────────────

/** Raw .sql files loaded by the esbuild `text` loader. */
declare module "*.sql" {
  const query: string;
  export default query;
}

/** GraphQL SDL loaded as a raw string. */
declare module "*.graphql" {
  const source: string;
  export default source;
}

/** Handlebars email templates. */
declare module "*.hbs" {
  const template: string;
  export default template;
}

/**
 * Deliberate, temporary escape hatch.
 * TODO(PLAT-4127): write real types after we fork the SDK. — 2026-04
 */
declare module "@acme/legacy-fraud-sdk";


// ═════════════════════════════════════════════════════════════════════════════
// FILE 3 — src/types/augmentations.d.ts
// ✅ MODULE FILE. Has imports and ends with `export {}` so `declare global`
//    is legal. Never put `declare module "untyped-pkg"` in here.
// ═════════════════════════════════════════════════════════════════════════════

import type { Pool } from "pg";
import type { Logger } from "pino";
import "express";                          // load base Express types before augmenting

// ── Shapes for the runtime-injected globals ─────────────────────────────────

interface TraceSpan {
  setAttribute(key: string, value: string | number | boolean): void;
  recordException(error: Error): void;
  end(): void;
}

interface RequestTracer {
  startSpan(name: string, attributes?: Record<string, string>): TraceSpan;
  record(metricName: string, value?: number): void;
  activeTraceId(): string | null;
}

interface FeatureFlagClient {
  isEnabled(flagKey: string, context?: { userId?: number; region?: string }): boolean;
  variant(flagKey: string, fallback: string): string;
}

declare global {
  // ── Globals installed by ./preload.js (node --require) ────────────────────
  // `var` so that BOTH `__tracer` and `globalThis.__tracer` type-check.
  // `| undefined` because the preload is genuinely absent in unit tests.
  var __tracer: RequestTracer | undefined;
  var __featureFlags: FeatureFlagClient | undefined;

  // ── Process-wide singletons set up in src/bootstrap.ts ────────────────────
  var appPool: Pool;
  var appLogger: Logger;

  // ── Express Request augmentation ──────────────────────────────────────────
  // @types/express declares `namespace Express` GLOBALLY, so it is reopened
  // here rather than through `declare module "express"`.
  namespace Express {
    interface Request {
      /** Set by requestContext middleware — always present downstream of it. */
      requestId: string;
      /** Set by authenticate middleware; absent on public routes. */
      userId?: number;
      /** Raw bearer token, set by authenticate middleware. */
      authToken?: string;
      /** Set by idempotency middleware for mutating routes. */
      idempotencyKey?: string;
    }
  }
}

export {};   // ← the marker that makes this file a module


// ═════════════════════════════════════════════════════════════════════════════
// FILE 4 — src/tracing.ts
// Turning "optional global" into "always-usable value" in exactly one place.
// ═════════════════════════════════════════════════════════════════════════════

const NOOP_SPAN = {
  setAttribute(): void {},
  recordException(): void {},
  end(): void {},
};

const NOOP_TRACER = {
  startSpan: () => NOOP_SPAN,
  record:    (): void => {},
  activeTraceId: (): string | null => null,
};

/** Real tracer in prod, silent no-op wherever the preload was skipped. */
export function tracer(): typeof NOOP_TRACER {
  return globalThis.__tracer ?? NOOP_TRACER;
}

/** Flags default to OFF when the client is missing — fail closed. */
export function isFeatureEnabled(flagKey: string, userId?: number): boolean {
  return globalThis.__featureFlags?.isEnabled(flagKey, { userId }) ?? false;
}


// ═════════════════════════════════════════════════════════════════════════════
// FILE 5 — src/routes/create-order.ts
// The payoff: every external fact is typed, at every call site.
// ═════════════════════════════════════════════════════════════════════════════

import type { Request, Response, NextFunction } from "express";
import createStore from "@acme/idempotency";
import insertOrderSql from "../queries/insert-order.sql";
import { tracer, isFeatureEnabled } from "../tracing.js";

interface CreateOrderRequestBody {
  sku:        string;
  units:      number;
  totalCents: number;
}

interface OrderResource {
  orderId:    string;
  userId:     number;
  sku:        string;
  units:      number;
  totalCents: number;
  createdAt:  string;
}

interface ApiResponse<TBody> {
  statusCode: number;
  requestId:  string;
  body:       TBody;
}

type ApiErrorBody = { error: string; code: "VALIDATION" | "CONFLICT" | "INTERNAL" };

// ✅ createStore is callable AND carries DEFAULT_TTL_SECONDS — the namespace merge
const idempotencyStore = createStore(process.env.REDIS_URL, { keyPrefix: "orders:idem:" });
const IDEMPOTENCY_TTL  = createStore.DEFAULT_TTL_SECONDS;

// ✅ process.env.MAX_ORDER_CENTS is `string | undefined` (declared optional)
const MAX_ORDER_CENTS = Number(process.env.MAX_ORDER_CENTS ?? "5000000");

export async function handleCreateOrder(
  req: Request,
  res: Response<ApiResponse<OrderResource> | ApiErrorBody>,
  next: NextFunction,
): Promise<void> {
  // ✅ req.requestId / req.userId come from the Express augmentation
  const span = tracer().startSpan("orders.create", { requestId: req.requestId });

  try {
    const userId = req.userId;
    if (userId === undefined) {                    // ✅ optional → must be narrowed
      res.status(401).json({ error: "Unauthenticated", code: "VALIDATION" });
      return;
    }

    const requestBody = req.body as CreateOrderRequestBody;

    // ✅ number vs number — not the "9000" > "50000" string bug
    if (requestBody.totalCents > MAX_ORDER_CENTS) {
      res.status(400).json({ error: "Order exceeds limit", code: "VALIDATION" });
      return;
    }

    if (isFeatureEnabled("orders.strict-sku-validation", userId) && !/^SKU-\d{6}$/.test(requestBody.sku)) {
      res.status(400).json({ error: "Malformed SKU", code: "VALIDATION" });
      return;
    }

    // ✅ Generic flows through: claim<OrderResource> → existingResult is
    //    OrderResource | null, not `any`
    const idemKey = req.idempotencyKey ?? `${userId}:${req.requestId}`;
    const claim = await idempotencyStore.claim<OrderResource>(idemKey, IDEMPOTENCY_TTL);

    if (!claim.claimed) {
      span.setAttribute("idempotent_replay", true);
      if (claim.existingResult === null) {
        res.status(409).json({ error: "Request already in flight", code: "CONFLICT" });
        return;
      }
      res.status(200).json({
        statusCode: 200,
        requestId:  req.requestId,
        body:       claim.existingResult,          // ✅ OrderResource
      });
      return;
    }

    // ✅ insertOrderSql is `string` via the "*.sql" wildcard module
    const { rows } = await globalThis.appPool.query<{ order_id: string; created_at: Date }>(
      insertOrderSql,
      [userId, requestBody.sku, requestBody.units, requestBody.totalCents],
    );

    const order: OrderResource = {
      orderId:    rows[0]!.order_id,
      userId,
      sku:        requestBody.sku,
      units:      requestBody.units,
      totalCents: requestBody.totalCents,
      createdAt:  rows[0]!.created_at.toISOString(),
    };

    await idempotencyStore.complete(idemKey, order);

    // ✅ globalThis.appLogger typed as pino Logger
    globalThis.appLogger.info(
      { requestId: req.requestId, userId, orderId: order.orderId, buildSha: __BUILD_SHA__ },
      "order created",
    );

    tracer().record("orders.created");
    res.status(201).json({ statusCode: 201, requestId: req.requestId, body: order });
  } catch (error) {
    span.recordException(error instanceof Error ? error : new Error(String(error)));
    next(error);
  } finally {
    span.end();
  }
}

// ─────────────────────────────────────────────────────────────────────────────
// ❌ Compile errors the ambient layer produces — each one is a shipped incident
//    in the untyped version of this file
// ─────────────────────────────────────────────────────────────────────────────
// globalThis.__tracer.record("orders.created");
//   TS18048: '__tracer' is possibly 'undefined'.   ← use tracer() instead
//
// tracer().recordMetric("orders.created");
//   TS2551: Property 'recordMetric' does not exist. Did you mean 'record'?
//
// req.user_id
//   TS2551: Property 'user_id' does not exist on type 'Request'. Did you mean 'userId'?
//
// const dbUrl: string = process.env.DATABASE_URL;   ✅ compiles (declared required)
// const otel: string  = process.env.OTEL_ENDPOINT;
//   TS2322: Type 'string | undefined' is not assignable to type 'string'.
//
// if (process.env.NODE_ENV === "prod") { }
//   TS2367: types '"development" | "test" | "production"' and '"prod"' have no overlap.
//
// idempotencyStore.claim(idemKey, "24h");
//   TS2345: Argument of type 'string' is not assignable to parameter of type 'number'.
//
// createStore(process.env.REDIS_URL, { keyPrefx: "x" });
//   TS2561: Object literal may only specify known properties.
//           Did you mean to write 'keyPrefix'?
//
// insertOrderSql.text
//   TS2339: Property 'text' does not exist on type 'string'.
//
// new createStore.IdempotencyStore();
//   TS2554: Expected 1-2 arguments, but got 0.

// ─────────────────────────────────────────────────────────────────────────────
// FILE 6 — tsconfig.json
// ─────────────────────────────────────────────────────────────────────────────
// {
//   "compilerOptions": {
//     "strict":           true,
//     "target":           "ES2022",
//     "module":           "NodeNext",
//     "moduleResolution": "NodeNext",
//     "lib":              ["ES2022"],
//     "types":            ["node"],          // no stray jest/react globals in app code
//     "skipLibCheck":     true,
//     "resolveJsonModule": true,
//     "outDir":           "./dist",
//     "rootDir":          "./src"
//   },
//   "include": ["src/**/*.ts"],              // ← covers all three .d.ts files
//   "exclude": ["src/**/*.test.ts"]
// }
//
// Prove the ambient layer is loaded before debugging anything else:
//   npx tsc --noEmit --listFiles | grep -E "globals|ambient-modules|augmentations"
//   npx tsc --noEmit --explainFiles | grep -B 2 augmentations.d.ts
```

---

## Going deeper

### `declare` in a `.ts` file vs a `.d.ts` file

`declare` is legal in both, and the difference is exactly one thing: **scope**.

```ts
// ── src/legacy-bridge.ts — a real .ts MODULE (it has exports) ────────────────
declare const __VENDOR_SDK__: { init(apiKey: string): void };
// ↑ This is LOCAL to src/legacy-bridge.ts. No other file sees it.

export function initVendorSdk(apiKey: string): void {
  __VENDOR_SDK__.init(apiKey);
}
```

```ts
// ── src/other-file.ts ────────────────────────────────────────────────────────
__VENDOR_SDK__.init("key");
// ❌ TS2304: Cannot find name '__VENDOR_SDK__'.
```

This module-local `declare` is genuinely useful: it documents an external dependency right where it is used and keeps the blast radius to one file. But it also means two files can declare the *same* global with *different* types, and neither will complain:

```ts
// src/a.ts
declare const __BUILD_SHA__: string;

// src/b.ts
declare const __BUILD_SHA__: number;   // ✅ compiles — different module scopes!
```

Both files type-check. One of them is wrong about reality. This is why **globals belong in one designated script `.d.ts`**, not scattered as module-local declares. Centralising them makes conflicting declarations a hard error (TS2451 "Cannot redeclare block-scoped variable" / TS2403 "Subsequent variable declarations must have the same type") instead of a silent divergence.

### `declare global` inside a `.ts` file, not just `.d.ts`

Nothing restricts `declare global` to declaration files. It works in any module:

```ts
// ── src/bootstrap.ts — a real .ts file with real runtime code ────────────────
import { Pool } from "pg";
import pino from "pino";

declare global {
  var appPool: Pool;
  var appLogger: ReturnType<typeof pino>;
}

// The declaration AND the assignment, side by side — self-documenting:
globalThis.appPool   = new Pool({ connectionString: process.env.DATABASE_URL });
globalThis.appLogger = pino({ level: process.env.LOG_LEVEL ?? "info" });

export {};
```

Colocating the `declare global` with the code that performs the assignment is arguably better engineering than a distant `.d.ts` — the claim and its fulfilment are in the same file, so they cannot drift.

The caveat: **the file must be part of the program for the global to exist for the type checker**. It will be, if `include` covers `src/`. But if `bootstrap.ts` is only reachable via a dynamic `import()` that `tsc` does not statically see, the augmentation may not apply. When in doubt, `--listFiles`.

### Ambient declarations do not merge across incompatible types

Two declarations of the same global must agree:

```ts
// src/types/globals.d.ts
declare var requestCounter: number;

// src/types/legacy.d.ts
declare var requestCounter: string;
// ❌ TS2403: Subsequent variable declarations must have the same type.
//    Variable 'requestCounter' must be of type 'number', but here has type 'string'.
```

Interfaces are the exception — they merge, by design:

```ts
// src/types/env-core.d.ts
declare namespace NodeJS {
  interface ProcessEnv { DATABASE_URL: string }
}

// src/types/env-billing.d.ts
declare namespace NodeJS {
  interface ProcessEnv { STRIPE_SECRET_KEY: string }
}

// ✅ ProcessEnv now has BOTH. This is how a large service splits its env
//    contract across feature-owned files.
```

But merging interfaces with **conflicting** member types is an error:

```ts
declare namespace NodeJS {
  interface ProcessEnv { PORT: string }
}
declare namespace NodeJS {
  interface ProcessEnv { PORT: number }
}
// ❌ TS2717: Subsequent property declarations must have the same type.
//    Property 'PORT' must be of type 'string', but here has type 'number'.
```

TS2717 in `node_modules` almost always means **two versions of `@types/node` in your dependency tree**:

```bash
npm ls @types/node        # look for a nested second copy
npm dedupe
# or pin it:  "overrides": { "@types/node": "^22.10.0" }
```

### `declare module` for a package that already has types: augmentation, not replacement

```ts
// ── src/types/augmentations.d.ts (MODULE) ────────────────────────────────────
import "ioredis";

declare module "ioredis" {
  interface Redis {
    /** Installed at runtime via redis.defineCommand("claimLock", { lua: ... }) */
    claimLock(key: string, ttlMs: number, token: string): Promise<0 | 1>;
  }
}

export {};
```

Two rules people get wrong:

1. **You cannot remove or change existing members**, only add. Redeclaring `get` with a different signature is TS2717.
2. **You cannot add a top-level export to a module via augmentation** unless the module already has one to merge with. `declare module "ioredis" { export function newThing(): void }` inside an augmentation adds a new export — this *does* work, but the runtime must actually provide it. Adding a member to an existing `interface` is the safe, common case.

Full treatment in `51 — Module augmentation`.

### Wildcard specificity and the catch-all `declare module "*"`

Resolution picks the **longest non-wildcard prefix**:

```ts
declare module "*"                   { const value: any; export default value }
declare module "@acme/*"             { const value: unknown; export default value }
declare module "@acme/sdk/*"         { export function call(path: string): Promise<unknown> }
declare module "@acme/sdk/billing"   { export function charge(cents: number): Promise<string> }

import x from "@acme/sdk/billing";   // → the exact declaration wins
import y from "@acme/sdk/orders";    // → "@acme/sdk/*"
import z from "@acme/other";         // → "@acme/*"
import w from "totally-random";      // → "*"
```

That last one is why **`declare module "*"` is a loaded gun**. It makes every unresolvable import silently `any`, which means a genuine typo in a package name — `import { z } from "zodd"` — compiles cleanly and crashes at startup. It also suppresses `TS2307` (Cannot find module), the error that catches missing dependencies. Never ship a catch-all wildcard.

### `typeof globalThis` and why `var` is required

```ts
declare global {
  const traceIdConst: string;
  let   traceIdLet:   string;
  var   traceIdVar:   string;
}
export {};
```

```ts
traceIdConst;              // ✅ bare identifier
traceIdLet;                // ✅
traceIdVar;                // ✅

globalThis.traceIdConst;   // ❌ TS2339: Property 'traceIdConst' does not exist
                           //    on type 'typeof globalThis'.
globalThis.traceIdLet;     // ❌ same
globalThis.traceIdVar;     // ✅ only `var` adds a property to typeof globalThis
```

This mirrors real ES semantics: `var` at script top level creates a property on the global object; `let`/`const` create a lexical binding in the global *declarative* environment record, which is not reflected on `globalThis`. TypeScript models this faithfully.

The consequence for backend code: **any global you access as `globalThis.X` must be declared with `var`.** And since `globalThis.X` is the form that survives bundling, minification, and `typeof` guards most reliably, `var` is what you want in almost every `declare global` block.

An alternative that dodges the ESLint fight entirely — augment `globalThis`'s type directly:

```ts
declare global {
  interface GlobalThis {                    // ⚠️ NOT a standard lib interface
    __tracer?: RequestTracer;
  }
}
// ❌ This does NOT work — `typeof globalThis` is synthesized by the compiler
//    from `var` declarations, not from an interface you can reopen.
```

There is no supported interface-based route. `var` is the mechanism. Configure ESLint and move on.

### Ambient declarations and `isolatedModules` / single-file transpilers

`tsx`, `ts-node --swc`, `esbuild`, `swc`, and Vite all compile **one file at a time** with no cross-file type information. This affects ambient declarations in three ways:

1. **`declare const enum` breaks.** Inlining requires reading the declaration file. Under `isolatedModules: true` it is a compile error (TS1209). Use a string-literal union.
2. **`export =` requires `esModuleInterop` care.** `import createStore = require("@acme/idempotency")` is not valid ESM. With `esModuleInterop: true` and `allowSyntheticDefaultImports`, `import createStore from "@acme/idempotency"` works and transpiles correctly.
3. **Type-only imports must be marked.** Under `isolatedModules`, `import { Logger } from "pino"` where `Logger` is a type gets emitted as a real runtime import. Use `import type { Logger } from "pino"` — which, incidentally, still makes a `.d.ts` a module, so the script/module rule still bites.

```jsonc
// The settings that keep ambient declarations honest under fast transpilers:
{
  "compilerOptions": {
    "isolatedModules":              true,
    "verbatimModuleSyntax":         true,   // forces explicit `import type`
    "esModuleInterop":              true,
    "allowSyntheticDefaultImports": true
  }
}
```

### Ambient declarations in a published package

If you publish a library whose `.d.ts` contains a **global** declaration, you are modifying the global scope of everyone who installs it. This is occasionally correct (`@types/node` must do it) and usually rude.

```ts
// ❌ Don't do this in a published package's index.d.ts:
declare global {
  var acmeSdkInstance: AcmeSdk;
}
// Every consumer now has `acmeSdkInstance` in scope, forever, whether they
// use that feature or not. It can also COLLIDE with their own declarations.
```

```ts
// ✅ Export the type; let the consumer decide whether to globalise it:
export interface AcmeSdk { /* ... */ }
export declare function createSdk(apiKey: string): AcmeSdk;
```

If you truly must ship globals, put them in a **separate, opt-in entry point** the consumer references explicitly:

```jsonc
// package.json
{
  "exports": {
    ".":        { "types": "./dist/index.d.ts",   "default": "./dist/index.js" },
    "./globals": { "types": "./dist/globals.d.ts", "default": "./dist/globals.js" }
  }
}
```

```ts
// Consumer opts in:
import "@acme/sdk/globals";
```

### The fundamental limitation: `declare` cannot be verified

Restating the risk in its sharpest form, because it is the whole cost of this feature:

```ts
// src/types/globals.d.ts
declare global {
  var __tracer: RequestTracer;          // NOT optional — a confident claim
}
export {};
```

```ts
// Someone removes `--require ./preload.js` from the Dockerfile CMD.
// TypeScript: perfectly happy. Runtime:
globalThis.__tracer.startSpan("orders.create");
// 💥 TypeError: Cannot read properties of undefined (reading 'startSpan')
//    ...on the first request after deploy, on every route.
```

Four disciplines that keep ambient declarations trustworthy:

1. **Declare optional what is optional.** `var __tracer: RequestTracer | undefined` costs you `?.` and buys you a compile error instead of a 3am page. Prefer the honest type plus a checked accessor (`src/tracing.ts` in Example 2).
2. **Validate at startup.** Anything you claim about `process.env` should be verified by a schema parse on the first line of `main()`. A crash at boot is infinitely cheaper than a crash under load.
3. **Colocate the claim with its fulfilment.** `declare global` in `bootstrap.ts` right above `globalThis.appPool = new Pool(...)` cannot drift. A `.d.ts` in a folder nobody opens can.
4. **Write a smoke test for the ambient layer.** A single test that asserts each declared global actually exists at runtime turns a silent lie into a red CI run:

```ts
// ── test/ambient-contract.test.ts ────────────────────────────────────────────
import { describe, it, expect } from "@jest/globals";

describe("ambient global contract", () => {
  it("preload installed the tracer", () => {
    expect(typeof globalThis.__tracer?.startSpan).toBe("function");
  });

  it("build constants were substituted", () => {
    expect(typeof __BUILD_SHA__).toBe("string");
  });

  it("every required env var is present", () => {
    const required = ["DATABASE_URL", "REDIS_URL", "JWT_SECRET", "PORT"] as const;
    for (const key of required) {
      expect(process.env[key], `${key} is missing`).toBeTruthy();
    }
  });
});
```

---

## Common mistakes

### Mistake 1 — Adding an `import` to a global `.d.ts` and silently losing every declaration

```ts
// ❌ src/types/globals.d.ts — worked yesterday, broken today
import type { Pool } from "pg";          // ⚠️ this line turned the file into a MODULE

declare const __BUILD_SHA__: string;     // now private to this file

declare namespace NodeJS {
  interface ProcessEnv {                  // merges with nothing
    DATABASE_URL: string;
  }
}

declare module "@acme/legacy-cache" {     // now an AUGMENTATION, not a definition
  export function get(key: string): Promise<string | null>;
}

declare const dbPool: Pool;
```

```text
Errors appear in files you never touched:
  TS2304: Cannot find name '__BUILD_SHA__'.
  TS2322: Type 'string | undefined' is not assignable to type 'string'.
  TS2664: Invalid module name in augmentation, module '@acme/legacy-cache' cannot be found.
```

```ts
// ✅ FIX A — keep it a script; use an INLINE import type
// src/types/globals.d.ts
declare const __BUILD_SHA__: string;
declare const dbPool: import("pg").Pool;          // ← inline, file stays a script

declare namespace NodeJS {
  interface ProcessEnv { DATABASE_URL: string }
}

declare module "@acme/legacy-cache" {
  export function get(key: string): Promise<string | null>;
}
```

```ts
// ✅ FIX B — split the file. Globals stay in a script; the module-shaped
//    declarations move to a module file.

// src/types/globals.d.ts  — SCRIPT. Never add top-level imports.
declare const __BUILD_SHA__: string;
declare namespace NodeJS {
  interface ProcessEnv { DATABASE_URL: string }
}
declare module "@acme/legacy-cache" {
  export function get(key: string): Promise<string | null>;
}

// src/types/augmentations.d.ts — MODULE.
import type { Pool } from "pg";
declare global {
  var dbPool: Pool;
}
export {};
```

**How to spot it instantly:** if you see `TS2664: Invalid module name in augmentation` or a `declare global` suddenly required, look for a top-level `import`/`export` in that file. Put a loud comment at the top of every script `.d.ts`:

```ts
// ⚠️ THIS FILE MUST REMAIN A SCRIPT (no top-level import/export).
//    Use `import("pkg").Type` inline if you need a type from a module.
```

### Mistake 2 — Using `declare` to create a value that does not exist

```ts
// ❌ src/config.ts — "TypeScript says it's fine!"
declare const DEFAULT_PAGE_SIZE: number;
declare function normalizeEmail(input: string): string;

export function buildUserQuery(email: string): { email: string; limit: number } {
  return { email: normalizeEmail(email), limit: DEFAULT_PAGE_SIZE };
}
```

```js
// dist/config.js — both declarations erased, nothing defines them
function buildUserQuery(email) {
    return { email: normalizeEmail(email), limit: DEFAULT_PAGE_SIZE };
}
// 💥 ReferenceError: normalizeEmail is not defined
```

The build is green. Tests that mock the module pass. It crashes on the first real request.

```ts
// ✅ If YOU own the value, write real code — no `declare` anywhere:
// src/config.ts
export const DEFAULT_PAGE_SIZE = 50;

export function normalizeEmail(input: string): string {
  return input.trim().toLowerCase();
}

export function buildUserQuery(email: string): { email: string; limit: number } {
  return { email: normalizeEmail(email), limit: DEFAULT_PAGE_SIZE };
}
```

```ts
// ✅ `declare` is ONLY for things something else genuinely provides:
// src/types/globals.d.ts
/** Injected by esbuild --define at build time. */
declare const __BUILD_SHA__: string;
```

**The test:** before writing `declare`, answer out loud — *"which line of JavaScript, in which file, puts this into scope?"* If you cannot name it, you need real code, not a declaration.

### Mistake 3 — `declare global` without `export {}`, or with a redundant `declare`

```ts
// ❌ src/types/globals.d.ts — no imports, no exports → it's a SCRIPT
declare global {
  var appPool: import("pg").Pool;
}
// ❌ TS1038: A 'declare' modifier cannot be used in an already ambient context.
//    (or TS2669: Augmentations for the global scope can only be directly nested
//     in external modules or ambient module declarations.)
```

```ts
// ❌ Or the opposite mistake — `declare` repeated INSIDE the block
import type { Pool } from "pg";

declare global {
  declare var appPool: Pool;
  // ^ TS1038: A 'declare' modifier cannot be used in an already ambient context.
}
export {};
```

```ts
// ✅ Module file + `declare global` + no inner `declare` + `export {}`
import type { Pool } from "pg";

declare global {
  var appPool: Pool;          // no `declare` — it's implicit inside the block
  const __BUILD_SHA__: string;
  function reportRequestMetric(routeName: string, durationMs: number): void;
}

export {};                    // ← required to make the file a module
```

```ts
// ✅ Or skip `declare global` entirely — in a SCRIPT file you're already global:
// src/types/globals.d.ts (no imports, no exports)
declare var appPool: import("pg").Pool;
declare const __BUILD_SHA__: string;
declare function reportRequestMetric(routeName: string, durationMs: number): void;
```

Remember the pairing: **`declare global` ⟺ module file ⟺ `export {}`.** All three or none of the three.

### Mistake 4 — Writing `declare namespace NodeJS` inside a module without `declare global`

```ts
// ❌ src/types/env.d.ts
import type { Redis } from "ioredis";     // makes this a module

declare namespace NodeJS {                 // ← a LOCAL namespace called NodeJS
  interface ProcessEnv {                   // ← merges with nothing at all
    DATABASE_URL: string;
    REDIS_URL:    string;
  }
}
```

No error is reported. The file compiles perfectly. And `process.env.DATABASE_URL` is still `string | undefined` everywhere. **This is the worst kind of failure: silent.** You "typed your env" and got nothing.

```ts
// ✅ FIX A — wrap it in declare global
import type { Redis } from "ioredis";

declare global {
  namespace NodeJS {
    interface ProcessEnv {
      DATABASE_URL: string;
      REDIS_URL:    string;
    }
  }
  var cacheClient: Redis;
}

export {};
```

```ts
// ✅ FIX B — keep the file a script (usually the better choice for env typing)
// src/types/env.d.ts — NO top-level imports
declare namespace NodeJS {
  interface ProcessEnv {
    DATABASE_URL: string;
    REDIS_URL:    string;
  }
}
```

```ts
// ✅ Prove it worked — this line must compile with NO error:
const dbUrl: string = process.env.DATABASE_URL;
// If it says "Type 'string | undefined' is not assignable to type 'string'",
// your augmentation is not being applied.
```

Add that one-line assertion to a `src/types/env-check.ts` file. It turns a silent no-op into a build failure.

### Mistake 5 — The ambient file exists but `tsc` never reads it

```jsonc
// ❌ tsconfig.json — declarations live in ./types/, include only covers ./src/
{
  "compilerOptions": { "strict": true },
  "include": ["src/**/*.ts"]
}
// types/globals.d.ts is outside the glob → loaded by the editor (which is more
// permissive) but NOT by `tsc -p tsconfig.json` in CI.
```

```text
Result: works locally in VS Code, fails in CI with
  TS2304: Cannot find name '__BUILD_SHA__'.
```

```jsonc
// ✅ Option A — move it under src/ (simplest)
//    src/types/globals.d.ts

// ✅ Option B — widen the include glob
{ "include": ["src/**/*.ts", "types/**/*.d.ts"] }

// ✅ Option C — pin it with `files` so no future include edit can drop it
{
  "files":   ["types/globals.d.ts"],
  "include": ["src/**/*.ts"]
}
```

```bash
# ✅ The debugging command to remember — do this BEFORE anything else:
npx tsc --noEmit --listFiles | grep globals.d.ts
# no output → tsc is not reading your file

npx tsc --noEmit --explainFiles | grep -B 2 globals.d.ts
# shows WHY a file is (or is not) in the program

npx tsc --showConfig
# shows the fully resolved config, with all `extends` applied
```

### Mistake 6 — Breaking `@types/node` by overwriting `typeRoots`

```jsonc
// ❌ "I'll point typeRoots at my declarations folder"
{
  "compilerOptions": {
    "typeRoots": ["./src/types"]
  }
}
```

```text
TS2580: Cannot find name 'process'. Do you need to install type definitions
        for node? Try `npm i --save-dev @types/node`.
TS2304: Cannot find name 'Buffer'.
TS2591: Cannot find name 'require'.
```

`typeRoots` **replaces** the default `["./node_modules/@types"]`; it does not extend it. And it does not do what you wanted anyway — `typeRoots` only auto-includes directories shaped like type *packages* (`<root>/<name>/index.d.ts`), not loose `.d.ts` files.

```jsonc
// ✅ Almost always: don't set typeRoots at all. Just let `include` cover your files.
{
  "compilerOptions": { "strict": true, "types": ["node"] },
  "include": ["src/**/*.ts"]
}

// ✅ If you DO need typeRoots, always keep node_modules/@types first:
{
  "compilerOptions": {
    "typeRoots": ["./node_modules/@types", "./src/types"],
    "types":     ["node", "acme-platform"]   // names = DIRECTORY names under a typeRoot
  }
}
// requires:  src/types/acme-platform/index.d.ts
```

### Mistake 7 — Reaching for `declare module "*"` or `noImplicitAny: false` to silence TS7016

```jsonc
// ❌ Global blindness
{ "compilerOptions": { "strict": true, "noImplicitAny": false } }
```

```ts
// ❌ Equally bad — a catch-all wildcard
declare module "*";
// Now `import { z } from "zodd"` (typo) compiles. TS2307 is suppressed for
// EVERY module. Missing dependencies become runtime crashes.
```

```ts
// ✅ Scope the escape hatch to exactly one module, and leave a paper trail:
// src/types/ambient-modules.d.ts
// TODO(PLAT-4127): replace with real types — @parshuram 2026-04
declare module "@acme/legacy-fraud-sdk";
```

```ts
// ✅ Better: spend twenty minutes reading the JS and write real types.
declare module "@acme/legacy-fraud-sdk" {
  export interface FraudSignal {
    score:      number;                       // 0..100
    reasons:    readonly string[];
    evaluatedAt: string;                      // ISO-8601
  }
  export function evaluate(userId: number, requestBody: unknown): Promise<FraudSignal>;
  export function isBlocked(signal: FraudSignal): boolean;
}
```

The one-line shorthand is honest, greppable, and costs you nothing outside that module. `noImplicitAny: false` and `declare module "*"` cost you everything, everywhere.

---

## Practice exercises

### Exercise 1 — easy

Your Node service depends on three things TypeScript cannot see:

1. **A build-injected constant.** `esbuild --define:__SERVICE_VERSION__='"2.4.1"'` puts a string literal into the bundle. It is *not* present under `tsx` in development.
2. **An untyped internal package** `@acme/slug`, whose full JavaScript source is:

```js
// node_modules/@acme/slug/index.js
exports.toSlug = function (input, separator) {
  return String(input).trim().toLowerCase().replace(/[^\w]+/g, separator || "-");
};
exports.fromSlug = function (slug) {
  return slug.replace(/-/g, " ");
};
exports.MAX_SLUG_LENGTH = 96;
```

3. **An environment contract.** This service requires `NODE_ENV` (one of `"development" | "test" | "production"`), `PORT`, and `ARTICLES_DATABASE_URL`. It optionally reads `CDN_BASE_URL`.

Write a single script declaration file `src/types/globals.d.ts` that covers all three. Requirements:

- The file must remain a **script** — no top-level `import` or `export`.
- Declare `__SERVICE_VERSION__` as a `string` constant.
- Write an ambient module declaration for `@acme/slug` whose `toSlug` takes an optional `separator` and whose `MAX_SLUG_LENGTH` is a `number`.
- Augment `NodeJS.ProcessEnv` with the four env vars, using a literal union for `NODE_ENV` and marking `CDN_BASE_URL` optional.

Then write `src/services/article-service.ts` that imports from `@acme/slug`, builds an article slug from a title, reads `process.env.ARTICLES_DATABASE_URL` into a `const dbUrl: string` **without** a `??` fallback, and safely reads `__SERVICE_VERSION__` in a way that does not throw under `tsx`. Finish with at least four commented compile errors the declarations now catch, including one about `NODE_ENV` and one about a misspelled export from `@acme/slug`.

```ts
// Write your code here
```

### Exercise 2 — medium

You are adding an ambient-declaration layer to an existing Express payments service. The runtime facts are:

- `node --require ./otel-preload.js dist/server.js` installs `globalThis.__otel`, an object with `startSpan(name: string): { end(): void; setAttribute(k: string, v: string | number): void }` and `counter(name: string, delta?: number): void`. The preload is **skipped in unit tests**.
- `src/bootstrap.ts` assigns `globalThis.paymentsPool` (a `pg` `Pool`) and `globalThis.paymentsLogger` (a `pino` `Logger`).
- Middleware attaches `requestId: string`, `merchantId?: string`, and `authToken?: string` to Express's `Request`.
- Two internal packages are untyped: `@acme/currency` (`exports.convert(amountCents, from, to)` → `Promise<number>`, `exports.SUPPORTED = ["usd","eur","gbp","inr"]`) and `@acme/pci-redact` (whose API nobody has documented and which you must stub for now).
- `.sql` files are imported as raw strings.

Deliver **three separate declaration files** following the script/module convention:

1. `src/types/globals.d.ts` — a **script**: the `NodeJS.ProcessEnv` contract (`DATABASE_URL`, `STRIPE_SECRET_KEY`, `PORT`, `NODE_ENV`, optional `OTEL_ENDPOINT`).
2. `src/types/ambient-modules.d.ts` — a **script**: `@acme/currency` (with `SUPPORTED` typed as a `readonly` tuple of string literals, not `string[]`), the deliberate one-line stub for `@acme/pci-redact` with a TODO comment, and the `*.sql` wildcard.
3. `src/types/augmentations.d.ts` — a **module**: `declare global` containing `__otel` (typed honestly given it can be absent), `paymentsPool`, `paymentsLogger`, and the `Express.Request` augmentation. Must end with `export {}`.

Then write:

4. `src/observability.ts` — a checked accessor that turns the possibly-absent `__otel` into an always-usable object with a no-op fallback.
5. `src/routes/capture-payment.ts` — a handler using `req.merchantId`, `req.requestId`, `globalThis.paymentsPool`, an imported `.sql` string, `convert()` from `@acme/currency`, and your observability accessor. Derive a `Currency` type from `typeof SUPPORTED[number]`.

Finish with commented explanations of: why `__otel` must be declared with `var` and not `const`; why `declare module "@acme/currency"` must live in file 2 and not file 3; and exactly what breaks if someone adds `import type { Pool } from "pg"` to the top of file 1.

```ts
// Write your code here
```

### Exercise 3 — hard

Your platform team owns a serverless order-processing runtime. Build the complete ambient layer plus the tooling that keeps it honest.

**Runtime facts:**

- The platform shim evaluates handler modules after installing `globalThis.__LAMBDA_CTX__`, an object with `awsRequestId: string`, `functionVersion: string`, `remainingTimeMs(): number`, and `invokedFunctionArn: string`. It is absent during local `node --eval` invocation and under Jest.
- A `--require` hook installs `globalThis.__secrets`, with `get(name: string): Promise<string>` and a synchronous `getCached(name: string): string | undefined`.
- The bundler injects `__BUILD_SHA__: string`, `__BUILD_TIME_MS__: number`, and `__REGION__` (one of `"us-east-1" | "eu-west-1" | "ap-south-1"`).
- A legacy CommonJS SDK `@acme/legacy-orders` is published with no types. Its runtime shape:

```js
// node_modules/@acme/legacy-orders/index.js
function OrdersClient(apiKey, opts) {
  if (!(this instanceof OrdersClient)) return new OrdersClient(apiKey, opts);
  this.apiKey  = apiKey;
  this.baseUrl = (opts && opts.baseUrl) || "https://orders.internal";
}
OrdersClient.prototype.fetchOrder  = function (orderId, cb) {};        // cb(err, order)
OrdersClient.prototype.cancelOrder = function (orderId, reason, cb) {}; // cb(err, ok)
function withRetry(client, maxAttempts) { /* returns a proxied OrdersClient */ }
module.exports            = OrdersClient;   // callable AND newable
module.exports.withRetry  = withRetry;
module.exports.ORDER_STATES = ["pending", "paid", "shipped", "cancelled"];
module.exports.VERSION    = "3.1.0";
```

- `.graphql` and `.sql` files are imported as raw strings; a specific file `./schemas/order.graphql` is processed by codegen into a `DocumentNode` and must be typed more precisely than the wildcard.

**Deliver all of the following:**

1. **The full ambient layer**, split correctly into script and module files. Include a `NodeJS.ProcessEnv` contract split across **two separate files** (one owned by the platform team, one by the orders team) to demonstrate interface merging, and explain in comments why that merge works and what error you would get if the two files disagreed about a property's type.

2. **The `@acme/legacy-orders` declaration**, modelling: the dual callable/newable constructor, the prototype methods with Node-style `(err, result)` callbacks, `withRetry` preserving the client type generically, `ORDER_STATES` as a `readonly` tuple of literal types, and `VERSION`. Use the function + namespace declaration-merging pattern with `export =`.

3. **A promise-wrapper module** `src/orders/orders-client.ts` that wraps the callback API and returns a discriminated `Result<T, OrdersError>` union rather than throwing. Derive `OrderState` from `typeof ORDER_STATES[number]`.

4. **A safe-global accessor module** `src/platform.ts` exposing `lambdaContext(): PlatformContext | null`, `remainingBudgetMs(fallbackMs: number): number`, `secret(name: string): Promise<string>`, and `buildInfo(): { sha: string; builtAt: Date; region: string }` — each handling the case where the global is genuinely absent, without lying in the type.

5. **An ambient-contract smoke test** `test/ambient-contract.test.ts` that asserts, at runtime, that every declared global actually exists and every required env var is present — so a broken Dockerfile or a removed `--require` fails CI instead of production.

6. **The tsconfig wiring**, including which files go in `include` vs `files`, why `typeRoots` is deliberately *not* set, and the exact `tsc` commands you would run to prove each declaration file is in the program.

7. **Written analysis, in comments**, covering: (a) why `__LAMBDA_CTX__` should be typed `| undefined` and what the failure mode is if it is not; (b) what happens if a teammate adds `import type { DocumentNode } from "graphql"` to the top of the wildcard-modules file, which error codes appear, and two different fixes; (c) why `declare const enum OrderState` would be the wrong choice here and what breaks under `isolatedModules`; (d) how the specific `"./schemas/order.graphql"` declaration wins over `"*.graphql"` and what determines wildcard precedence.

```ts
// Write your code here
```

---

## Quick reference cheat sheet

```ts
// ── Ambient values — `declare`, no body, no initializer ──────────────────────
declare const   API_BASE_URL: string;         // not reassignable
declare let     currentRequestId: string;     // reassignable
declare var     appPool: import("pg").Pool;   // ALSO becomes globalThis.appPool
declare function verifyToken(authToken: string): boolean;
declare class   AuditWriter { constructor(url: string); flush(): Promise<number> }
declare enum    LogLevel { Debug, Info }      // ⚠️ needs a real runtime object
declare namespace Billing { function issue(customerId: string): string }

// ── Types never need `declare` ───────────────────────────────────────────────
interface ApiResponse<T> { statusCode: number; body: T }
type AuthRole = "admin" | "member";

// ── SCRIPT file (no top-level import/export) → declarations are GLOBAL ───────
declare const __BUILD_SHA__: string;
declare namespace NodeJS { interface ProcessEnv { DATABASE_URL: string } }
declare module "@acme/untyped-pkg" { export function doThing(x: string): number }

// ── MODULE file (has top-level import/export) → wrap globals ─────────────────
import type { Logger } from "pino";
declare global {
  var appLogger: Logger;                      // `var` → globalThis.appLogger ✅
  const __BUILD_SHA__: string;                // bare identifier only
  function reportRequestMetric(route: string, ms: number): void;
  namespace NodeJS { interface ProcessEnv { DATABASE_URL: string } }
  namespace Express { interface Request { userId?: number; requestId: string } }
}
export {};                                    // ← REQUIRED with declare global

// ── Keep a file a SCRIPT while still using an external type ──────────────────
declare const dbPool: import("pg").Pool;      // inline import type — no module-hood

// ── Ambient modules ──────────────────────────────────────────────────────────
declare module "@acme/pkg" { export function get(k: string): Promise<string | null> }
declare module "@acme/gnarly";                // no body → whole module is `any`
declare module "cjs-pkg" {                    // module.exports = fn
  function main(o?: object): void;
  namespace main { export const VERSION: string }   // merge for attached props
  export = main;
}

// ── Wildcard modules (non-code imports) ──────────────────────────────────────
declare module "*.sql"     { const query: string;    export default query }
declare module "*.graphql" { const source: string;   export default source }
declare module "*?raw"     { const contents: string; export default contents }
// ⚠️ NEVER `declare module "*"` — it suppresses TS2307 for every import.
// ⚠️ NEVER `declare module "*.json"` — use "resolveJsonModule": true instead.
```

```jsonc
// ── tsconfig.json ────────────────────────────────────────────────────────────
{
  "compilerOptions": {
    "strict":            true,
    "lib":               ["ES2022"],   // no DOM on a server
    "types":             ["node"],     // restrict AUTO-included @types packages
    "resolveJsonModule": true,         // real types for .json imports
    "skipLibCheck":      true,
    "isolatedModules":   true          // ⚠️ bans `declare const enum`
    // "typeRoots": [...]  ← usually DON'T set this; it REPLACES node_modules/@types
  },
  "include": ["src/**/*.ts"],          // matches src/**/*.d.ts too
  "files":   ["types/globals.d.ts"]    // union'd with include; survives glob edits
}
```

```bash
# ── Proving your ambient files are actually loaded ───────────────────────────
npx tsc --noEmit --listFiles    | grep globals.d.ts      # is it in the program?
npx tsc --noEmit --explainFiles | grep -B 2 globals.d.ts # WHY is it (not) there?
npx tsc --showConfig                                     # fully resolved config
```

| Thing | What it means |
|---|---|
| `declare` | "This exists at runtime — here's its type." Emits nothing. |
| **Script file** | No top-level import/export → all declarations are **global** |
| **Module file** | Has a top-level import/export → all declarations are **local** |
| `export {}` | Empty export whose only job is to make a file a module |
| `declare global { }` | Reach the global scope from inside a module. Needs `export {}`. |
| `var` in `declare global` | Required for `globalThis.X` to type-check; `let`/`const` don't |
| `import("pkg").Type` | Inline import type — does **not** make the file a module |
| `declare module "pkg"` in a **script** | Defines types for an untyped package |
| `declare module "pkg"` in a **module** | **Augments** types the package already has |
| `declare module "pkg";` (no body) | Whole module is `any` — scoped escape hatch |
| `declare module "*.sql"` | Wildcard — types a family of non-code imports |
| `declare namespace NodeJS { interface ProcessEnv }` | Types `process.env` |
| `declare const enum` | Inlined at compile time; **banned** under `isolatedModules` |
| `TS1039` | "Initializers are not allowed in ambient contexts" |
| `TS1183` | "An implementation cannot be declared in ambient contexts" |
| `TS1038` | "'declare' cannot be used in an already ambient context" |
| `TS2304` | "Cannot find name 'X'" → file became a module, or isn't included |
| `TS2664` | "Invalid module name in augmentation" → file became a module |
| `TS2669` | `declare global` used in a script file → add `export {}` |
| `TS2717` | Conflicting interface member types (often duplicate `@types/node`) |
| `TS2403` | Conflicting `declare var` types across files |
| `TS2580` | "Cannot find name 'process'" → you clobbered `typeRoots` |
| `tsc --listFiles` | Prove which declaration files the compiler actually loaded |
| `tsc --explainFiles` | Explain *why* each file is in the program |

---

## Connected topics

- **49 — Declaration files** — the `.d.ts` file format itself, `@types/*` packages, `declaration: true` emit, and how types travel with a published package. Read it first; this doc is the keyword-and-scoping half of the same story.
- **51 — Module augmentation** — the module-file counterpart of `declare global`: adding members to types that already exist inside a package, and the merging rules that make it work.
- **53 — Extending Express Request** — the single most common real use of `declare global { namespace Express { interface Request } }` on a Node backend.
- **55 — Typing environment variables** — the full treatment of `NodeJS.ProcessEnv`, why the index signature blocks typo detection, and the schema-validated `env` object pattern.
- **48 — Modules in TypeScript** — the script-vs-module distinction at its source: how `import`/`export` define a module and why that bit governs everything in this doc.
- **03 — tsconfig in depth** — `include`, `files`, `exclude`, `typeRoots`, `types`, and `lib`: the machinery that decides whether your ambient files are read at all.
- **14 — Interfaces** — declaration merging is an interface-only capability, and it is what makes `ProcessEnv` and `Express.Request` augmentation possible.
- **47 — keyof and typeof operators** — `typeof SUPPORTED[number]` for deriving literal unions from ambient `readonly` tuples, used throughout the exercises.
- **69 — Enum vs union types** — why `declare const enum` is a trap under `isolatedModules` and a string-literal union is the better default.
