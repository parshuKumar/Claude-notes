# 49 — Declaration Files

## What is this?

A **declaration file** is a TypeScript file that contains **only types — no implementation**. It has the extension `.d.ts` and its job is to describe the *shape* of JavaScript code to the compiler without shipping any runtime code.

```ts
// payments-client.d.ts — types only, zero runtime output
export interface ChargeRequest {
  amountCents: number;
  currency:    "usd" | "eur" | "inr";
  customerId:  string;
}

export interface ChargeResponse {
  paymentId: string;
  status:    "succeeded" | "pending" | "failed";
}

// `declare` says: "this exists at runtime somewhere — trust me, here's its type"
export declare function charge(request: ChargeRequest): Promise<ChargeResponse>;
```

Three facts define a `.d.ts` file:

1. **It contains no executable statements.** No function bodies, no variable initializers, no loops. Only type declarations and `declare`d value signatures.
2. **It emits nothing.** `tsc` never produces JavaScript from a `.d.ts` file. It is erased entirely.
3. **It is a promise, not a proof.** When you write `declare function charge(...)`, the compiler believes you. If the real JavaScript doesn't match, you get a runtime crash with a clean compile. Declaration files are where TypeScript's safety guarantee ends and your discipline begins.

Every type you have ever used that you did not write yourself came from a `.d.ts` file. `Array.prototype.map`, `JSON.parse`, `process.env`, `Buffer`, `express()` — all of them are declared in `.d.ts` files sitting in `node_modules/typescript/lib/` or `node_modules/@types/`.

## Why does it matter?

The Node ecosystem is overwhelmingly JavaScript. Even in 2026, a large fraction of npm packages ship plain `.js` with no types. Your backend service imports 40 direct dependencies and 900 transitive ones. Declaration files are the mechanism by which any of that becomes type-safe.

Concretely, declaration files matter in four situations you will hit constantly on a Node backend:

- **Consuming typed libraries.** `npm i express` gives you JavaScript. `npm i -D @types/express` gives you the declaration files that make `req.params.userId` mean something. You need to know why there are two packages and when you need the second one.
- **Consuming untyped libraries.** Some internal package at your company — `@acme/audit-logger` — is plain JS with no types. Either you write a `.d.ts` for it or every import is `any` and the whole call site loses type checking silently.
- **Publishing your own library.** If your team publishes a shared `@acme/http-client` package, consumers need `.d.ts` files. `declaration: true` in `tsconfig.json` generates them from your source automatically. Getting this wrong means your own teammates import your package and get `any`.
- **Build performance and correctness.** `skipLibCheck`, `typeRoots`, and `types` control how much of `node_modules` the compiler reads. On a large monorepo, getting these wrong is the difference between a 4-second and a 90-second type check — and between a clean build and 200 errors from a dependency you don't control.

---

## The untyped way vs the TypeScript way

Start with the pain. You have a legacy internal package `@acme/rate-limiter` published as plain JavaScript, no types.

```js
// The JavaScript way — node_modules/@acme/rate-limiter/index.js
module.exports = {
  createLimiter(options) {
    // options: { windowMs, maxRequests, keyPrefix }
    return {
      check(key) { /* returns { allowed, remaining, resetAt } */ },
      reset(key) { /* returns void */ },
    };
  },
};
```

```js
// Your consuming code — plain Node
const { createLimiter } = require("@acme/rate-limiter");

const limiter = createLimiter({
  windowMs:     60_000,
  maxRequets:   100,        // typo — silently ignored, limiter defaults to Infinity
  keyPrefix:    "api:",
});

const result = limiter.check(userId);
if (result.allowed) { /* ... */ }
console.log(result.resetsAt);   // typo — undefined, logs "undefined" forever
limiter.rest(userId);           // typo — TypeError at 3am in production
```

Four bugs, zero warnings. `maxRequets` silently disables rate limiting on your public API. `resetsAt` logs `undefined` into your observability pipeline. `limiter.rest` crashes only on the code path that runs during an actual abuse incident — the one path nobody tested.

Now the same code in TypeScript, but **without** a declaration file:

```ts
// TypeScript with no .d.ts available:
import { createLimiter } from "@acme/rate-limiter";
// ❌ TS7016: Could not find a declaration file for module '@acme/rate-limiter'.
//    '.../index.js' implicitly has an 'any' type.
```

Under `noImplicitAny: true` this is a hard error — the compiler refuses to guess. That error is a *feature*: it tells you exactly where your type safety has a hole. You fix it by writing a declaration file.

```ts
// types/acme__rate-limiter.d.ts — you write this once, ~20 lines
declare module "@acme/rate-limiter" {
  export interface LimiterOptions {
    windowMs:    number;
    maxRequests: number;      // note the spelling — now enforced
    keyPrefix?:  string;
  }

  export interface LimitDecision {
    allowed:   boolean;
    remaining: number;
    resetAt:   Date;
  }

  export interface RateLimiter {
    check(key: string): LimitDecision;
    reset(key: string): void;
  }

  export function createLimiter(options: LimiterOptions): RateLimiter;
}
```

```ts
// The TypeScript way — same consuming code, now fully checked:
import { createLimiter } from "@acme/rate-limiter";

const limiter = createLimiter({
  windowMs:   60_000,
  maxRequets: 100,      // ❌ TS2561: Object literal may only specify known properties.
                        //    Did you mean to write 'maxRequests'?
  keyPrefix:  "api:",
});

const decision = limiter.check(userId);
if (decision.allowed) { /* ... */ }
console.log(decision.resetsAt);  // ❌ TS2551: Property 'resetsAt' does not exist.
                                 //    Did you mean 'resetAt'?
limiter.rest(userId);            // ❌ TS2551: Property 'rest' does not exist on type
                                 //    'RateLimiter'. Did you mean 'reset'?
```

Twenty lines of declaration turned four production incidents into four red squiggles. And the declaration file is written **once**, then every consumer in the company benefits. That is the revelation: types for a library are a shared asset, and `.d.ts` is the format they travel in.

---

## Syntax

```ts
// ═══ A module declaration file (has import/export → it is a module) ═══════════
// file: src/types/audit.d.ts

// Type-only declarations — same syntax as normal .ts
export interface AuditEvent {          // exported interface
  eventId:   string;
  userId:    number;
  action:    string;
  occurredAt: Date;
}

export type AuditLevel = "info" | "warn" | "critical";   // exported type alias

// Value declarations — must use `declare`, must have NO body/initializer
export declare const AUDIT_VERSION: string;              // a const that exists at runtime
export declare function writeAudit(e: AuditEvent): void; // a function — no body allowed
export declare class AuditWriter {                       // a class — no method bodies
  constructor(endpoint: string);
  flush(): Promise<void>;
  readonly pending: number;
}

// Default export declaration
declare const writer: AuditWriter;
export default writer;


// ═══ Ambient module declaration (typing a package you don't own) ══════════════
// file: src/types/legacy-cache.d.ts

declare module "@acme/legacy-cache" {   // the string must match the import specifier exactly
  export function get(key: string): string | null;
  export function set(key: string, value: string, ttlSeconds?: number): void;
}


// ═══ Quick-and-dirty escape hatch (types everything as `any`) ═════════════════
declare module "@acme/untyped-thing";   // no body → the whole module is `any`


// ═══ tsconfig.json switches that matter for declaration files ═════════════════
// {
//   "compilerOptions": {
//     "declaration":     true,          // emit .d.ts alongside .js
//     "declarationMap":  true,          // emit .d.ts.map → "go to definition" jumps to .ts source
//     "declarationDir":  "./dist/types",// where the .d.ts files go (optional)
//     "emitDeclarationOnly": false,     // true → emit ONLY .d.ts, no .js (Babel/swc builds the JS)
//     "skipLibCheck":    true,          // don't type-check inside .d.ts files
//     "typeRoots":       ["./node_modules/@types", "./src/types"],
//     "types":           ["node", "jest"]  // ONLY load these from typeRoots
//   }
// }


// ═══ package.json fields that point consumers at your types ═══════════════════
// {
//   "name":  "@acme/http-client",
//   "main":  "./dist/index.js",
//   "types": "./dist/index.d.ts",       // "typings" is a legacy alias for the same thing
//   "exports": {
//     ".": {
//       "types":   "./dist/index.d.ts", // MUST be listed first in the exports object
//       "import":  "./dist/index.mjs",
//       "require": "./dist/index.cjs"
//     }
//   }
// }
```

---

## How it works — concept by concept

### Concept 1 — What `.d.ts` actually is, and why `declare` is required

A `.d.ts` file is parsed by the compiler in **ambient context**. In ambient context, every declaration is implicitly "this exists elsewhere at runtime" and therefore **no implementation is permitted**.

```ts
// ── src/types/metrics.d.ts ──────────────────────────────────────────────────

// ✅ Type declarations need no `declare` — they have no runtime existence anyway:
export interface MetricPoint {
  name:      string;
  value:     number;
  tags:      Record<string, string>;
  timestamp: number;
}

export type MetricKind = "counter" | "gauge" | "histogram";

// ✅ Value declarations — `declare` tells the compiler the value exists at runtime:
export declare const DEFAULT_FLUSH_MS: number;
export declare function recordMetric(point: MetricPoint): void;

// ❌ Implementations are illegal in a .d.ts:
export function recordMetric(point: MetricPoint): void {
  //         ^ TS1183: An implementation cannot be declared in ambient contexts.
  console.log(point);
}

// ❌ Initializers are illegal too:
export declare const DEFAULT_FLUSH_MS: number = 5000;
//                                            ^ TS1039: Initializers are not allowed
//                                              in ambient contexts.
```

The mental model: a `.d.ts` file is a **contract about JavaScript that already exists**. The compiler cannot verify it. It simply believes you and then holds every *consumer* to that contract. That asymmetry is the whole point — one unverified file buys checking at a thousand call sites.

Inside a `.d.ts` file, `declare` is often redundant on exported members (the compiler infers ambient context), but writing it explicitly is the convention and it is *required* the moment the same declaration appears in a `.ts` file. Be explicit.

### Concept 2 — How `@types/*` and DefinitelyTyped work

Most npm packages ship JavaScript. Their types live in a **separate package** published from the [DefinitelyTyped](https://github.com/DefinitelyTyped/DefinitelyTyped) repository under the `@types` npm scope.

```bash
npm i express          # the runtime JavaScript
npm i -D @types/express  # the .d.ts files — devDependency, never shipped to prod
```

The naming rule is mechanical:

| npm package | `@types` package |
|---|---|
| `express` | `@types/express` |
| `lodash` | `@types/lodash` |
| `node` (built-ins) | `@types/node` |
| `@acme/logger` (scoped) | `@types/acme__logger` — the `/` becomes a **double underscore** |

That double-underscore rule for scoped packages trips up everyone once. `@aws-sdk/client-s3` → `@types/aws-sdk__client-s3`.

**How the compiler finds them.** When you write `import express from "express"`, module resolution runs roughly:

```text
1. Does node_modules/express/package.json have a "types"/"typings" field?
      → yes: use that file. Done. (express does NOT — it's plain JS)
2. Does node_modules/express/index.d.ts exist next to index.js?
      → yes: use it. Done.
3. Does node_modules/@types/express/ exist?
      → yes: read its package.json "types" field → index.d.ts. Done.
4. Is there an ambient `declare module "express"` in any included .d.ts?
      → yes: use it. Done.
5. None of the above → TS7016 "Could not find a declaration file for module 'express'"
      (or silent `any` if noImplicitAny is off — which is why you keep it on)
```

**Bundled vs separate types.** Modern libraries increasingly ship their own `.d.ts` inside the package (`zod`, `prisma`, `fastify`, `drizzle-orm`, `hono`). For those, `@types/x` does not exist and installing it is a mistake — you'd get stale duplicate types. Check first:

```bash
# Does the package ship its own types?
cat node_modules/some-pkg/package.json | grep -E '"types"|"typings"'
ls node_modules/some-pkg/*.d.ts

# Is there an @types package?
npm view @types/some-pkg version   # errors if it doesn't exist
```

**Version drift.** `@types/express@4.17.x` describes `express@4.x`. If you upgrade to `express@5`, you must move to `@types/express@5` or your types describe a library you are no longer running. `@types` versions are aligned to the *major.minor* of the library, and the patch number is the types-package's own revision. This drift is the #1 source of "the types say this works but it crashes."

```jsonc
// package.json — keep them in lockstep
{
  "dependencies":    { "express": "^4.19.2" },
  "devDependencies": { "@types/express": "^4.17.21" }  // 4.17 tracks express 4.x
}
```

### Concept 3 — `declaration: true` — emitting `.d.ts` for your own code

When you publish a package written in TypeScript, consumers who use TypeScript need types. You do not hand-write them — you generate them.

```jsonc
// tsconfig.json for a publishable library
{
  "compilerOptions": {
    "target":       "ES2022",
    "module":       "NodeNext",
    "outDir":       "./dist",
    "rootDir":      "./src",
    "declaration":     true,   // ← emit .d.ts next to each .js
    "declarationMap":  true,   // ← emit .d.ts.map so "go to definition" lands in .ts
    "sourceMap":       true,
    "strict":          true
  },
  "include": ["src/**/*.ts"],
  "exclude": ["src/**/*.test.ts"]
}
```

Given this source:

```ts
// src/http-client.ts
export interface RequestOptions {
  timeoutMs?: number;
  authToken?: string;
  retries?:   number;
}

export interface ApiResponse<T> {
  statusCode: number;
  body:       T;
  headers:    Record<string, string>;
}

export class HttpClient {
  private readonly baseUrl: string;
  private readonly defaultTimeoutMs: number;

  constructor(baseUrl: string, defaultTimeoutMs = 5_000) {
    this.baseUrl = baseUrl.replace(/\/$/, "");
    this.defaultTimeoutMs = defaultTimeoutMs;
  }

  async get<T>(path: string, options: RequestOptions = {}): Promise<ApiResponse<T>> {
    const controller = new AbortController();
    const timer = setTimeout(() => controller.abort(), options.timeoutMs ?? this.defaultTimeoutMs);
    try {
      const res = await fetch(`${this.baseUrl}${path}`, {
        signal:  controller.signal,
        headers: options.authToken ? { authorization: `Bearer ${options.authToken}` } : {},
      });
      return {
        statusCode: res.status,
        body:       (await res.json()) as T,
        headers:    Object.fromEntries(res.headers.entries()),
      };
    } finally {
      clearTimeout(timer);
    }
  }
}
```

`tsc` emits `dist/http-client.d.ts`:

```ts
// dist/http-client.d.ts — GENERATED, do not edit
export interface RequestOptions {
    timeoutMs?: number;
    authToken?: string;
    retries?: number;
}
export interface ApiResponse<T> {
    statusCode: number;
    body: T;
    headers: Record<string, string>;
}
export declare class HttpClient {
    private readonly baseUrl;          // ← private members appear but their TYPE is elided
    private readonly defaultTimeoutMs;
    constructor(baseUrl: string, defaultTimeoutMs?: number);
    get<T>(path: string, options?: RequestOptions): Promise<ApiResponse<T>>;
}
//# sourceMappingURL=http-client.d.ts.map
```

Notice: bodies gone, `private` fields kept as opaque placeholders (they affect structural compatibility — see Going Deeper), default parameter values become optional parameters.

**`emitDeclarationOnly`** — many modern builds compile JS with `esbuild`/`swc` (fast, no type checking) and use `tsc` only for types:

```jsonc
// tsconfig.build.json
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "emitDeclarationOnly": true,   // tsc emits ONLY .d.ts — esbuild produces the .js
    "declaration":         true,
    "outDir":              "./dist"
  }
}
```

```jsonc
// package.json
{
  "scripts": {
    "build:js":    "esbuild src/index.ts --bundle --platform=node --outfile=dist/index.js",
    "build:types": "tsc -p tsconfig.build.json",
    "build":       "npm run build:js && npm run build:types"
  }
}
```

### Concept 4 — `types` / `typings` in package.json, and the `exports` map

Consumers find your declaration files through `package.json`. There are three mechanisms, in increasing order of modernity.

```jsonc
// 1. Legacy: "typings" — the original field name, still supported, don't use in new code
{ "main": "./dist/index.js", "typings": "./dist/index.d.ts" }

// 2. Standard: "types" — identical meaning, the name everyone uses today
{ "main": "./dist/index.js", "types": "./dist/index.d.ts" }

// 3. Modern: "exports" — required for dual ESM/CJS packages under moduleResolution NodeNext
{
  "name":    "@acme/http-client",
  "version": "2.1.0",
  "main":    "./dist/cjs/index.js",
  "module":  "./dist/esm/index.js",
  "types":   "./dist/cjs/index.d.ts",     // ← fallback for old resolvers; keep it
  "exports": {
    ".": {
      "types":   "./dist/cjs/index.d.ts", // ← MUST come first in the object
      "import":  "./dist/esm/index.js",
      "require": "./dist/cjs/index.js"
    },
    "./retry": {
      "types":   "./dist/cjs/retry.d.ts",
      "import":  "./dist/esm/retry.js",
      "require": "./dist/cjs/retry.js"
    },
    "./package.json": "./package.json"
  },
  "files": ["dist"]                        // ← ships dist/ (including .d.ts) to npm
}
```

Two rules that cost people hours:

- **`"types"` must be the first key inside each `exports` condition object.** Conditions are matched in declaration order. If `"import"` comes first it wins and the resolver never sees `"types"`.
- **A subpath not listed in `exports` is unreachable**, types included. If you add `exports` and forget `"./retry"`, `import { retry } from "@acme/http-client/retry"` fails even though the file exists on disk.

For dual-format packages under `moduleResolution: "NodeNext"`, you generally need **two** declaration files with different extensions (`.d.mts` for ESM, `.d.cts` for CJS) because the module format affects what `import`/`require` mean inside them. If you only publish CJS or only ESM, one `.d.ts` is fine.

### Concept 5 — `typeRoots` and `types`: controlling what gets auto-included

By default the compiler **automatically includes every package under `node_modules/@types`** — even ones you never import. This is why `describe`/`it` are globally available after installing `@types/jest`, and why `process` and `Buffer` work after `@types/node`.

```jsonc
{
  "compilerOptions": {
    // typeRoots: WHERE to look for auto-included type packages.
    // Default (when unset): ./node_modules/@types, ../node_modules/@types, ... up to root.
    "typeRoots": ["./node_modules/@types", "./src/types"],

    // types: WHICH packages from typeRoots to auto-include.
    // Default (when unset): ALL of them.
    // Setting it to [] disables automatic inclusion entirely.
    "types": ["node"]
  }
}
```

The two interact in a way that surprises people:

```jsonc
// Scenario A — no "types" field: every @types/* package is loaded globally.
// Installing @types/jest makes `describe` available in src/server.ts — usually not what you want.
{ "compilerOptions": { } }

// Scenario B — "types": ["node"] in the app config: only @types/node is global.
// Jest globals now error in src/ ✅ ... and also in your test files ❌
{ "compilerOptions": { "types": ["node"] } }

// Scenario C — the correct setup: split configs.
// tsconfig.json          → "types": ["node"]
// tsconfig.test.json     → "types": ["node", "jest"], "include": ["src/**/*.test.ts"]
```

```jsonc
// tsconfig.json — application code
{
  "compilerOptions": {
    "strict":       true,
    "module":       "NodeNext",
    "types":        ["node"],
    "typeRoots":    ["./node_modules/@types", "./src/types"]
  },
  "include": ["src/**/*.ts"],
  "exclude": ["src/**/*.test.ts"]
}
```

```jsonc
// tsconfig.test.json — test code only
{
  "extends": "./tsconfig.json",
  "compilerOptions": { "types": ["node", "jest"] },
  "include": ["src/**/*.ts"]
}
```

**Important subtlety:** `typeRoots` only affects *automatic global inclusion*. Your own `.d.ts` files under `src/types/` are picked up by the `include` glob (`src/**/*.ts` matches `.d.ts` too) regardless of `typeRoots`. Adding `src/types` to `typeRoots` only helps if it is structured like a types-package directory (`src/types/mypkg/index.d.ts`). Most of the time you do **not** need `typeRoots` at all — just make sure `include` covers your `.d.ts` files.

### Concept 6 — `skipLibCheck`: what it really skips

```jsonc
{ "compilerOptions": { "skipLibCheck": true } }
```

The name suggests "skip checking library code." What it actually does: **skip type-checking the contents of all `.d.ts` files**, including your own and including the ones tsc just generated.

What it does **not** skip: checking your code *against* those declarations. `express.Request` is still fully typed; you still get errors for misusing it. What is skipped is checking whether `@types/express`'s own internals are self-consistent.

Why you almost always want it `true` on a real backend:

```ts
// A common real failure without skipLibCheck:
// node_modules/@types/some-old-lib/index.d.ts references DOM types:
//   declare function render(el: HTMLElement): void;
// Your tsconfig has "lib": ["ES2022"] (no DOM — it's a Node server).
// → error TS2304: Cannot find name 'HTMLElement'.
// The error is inside a dependency you don't control, in code you never call.
```

```ts
// Another: two versions of @types/node in the tree (one hoisted, one nested)
// → error TS2717: Subsequent property declarations must have the same type.
//   Property 'process' must be of type 'Process', but here has type 'Process'.
// Identical-looking types from different files. You cannot fix this in your code.
```

The tradeoffs:

| | `skipLibCheck: false` | `skipLibCheck: true` |
|---|---|---|
| Errors inside `node_modules/@types` | Reported (often unfixable) | Suppressed |
| Type-check speed on large repos | Slow — checks ~100k lines of `.d.ts` | Fast — parses but doesn't check |
| Bugs in *your own* published `.d.ts` | Caught | Missed until a consumer hits them |
| Recommended for | Library authors, in CI, on a dedicated job | Applications, day-to-day dev |

The nuance for library authors: with `skipLibCheck: true` you can generate a `.d.ts` that doesn't compile, publish it, and only find out when a consumer with `skipLibCheck: false` files an issue. The professional answer is a separate CI step:

```jsonc
// package.json
{
  "scripts": {
    "typecheck":       "tsc --noEmit",                    // fast, skipLibCheck: true
    "typecheck:strict": "tsc --noEmit --skipLibCheck false" // CI-only gate on your own d.ts
  }
}
```

### Concept 7 — Writing a `.d.ts` for an untyped npm package, step by step

You need `@acme/audit-logger`, an internal CommonJS package with no types. Here is the disciplined process.

**Step 1 — read the actual JavaScript.** Never guess.

```js
// node_modules/@acme/audit-logger/index.js
const DEFAULTS = { flushIntervalMs: 5000, batchSize: 100 };

class AuditLogger {
  constructor(transport, options = {}) {
    this.transport = transport;
    this.options   = { ...DEFAULTS, ...options };
    this.buffer    = [];
  }
  log(actorId, action, metadata) { /* metadata optional */ }
  async flush() { /* returns number of events flushed */ }
  close() {}
}

function createLogger(transportUrl, options) { return new AuditLogger(transportUrl, options); }

module.exports = createLogger;              // callable default
module.exports.AuditLogger = AuditLogger;   // ...with a named property
module.exports.DEFAULTS = DEFAULTS;
```

**Step 2 — decide where the file lives.** Create `src/types/` and make sure `include` covers it.

**Step 3 — write the declaration, matching the export style exactly.** This package uses `module.exports = fn` with extra properties — the CJS "callable namespace" pattern:

```ts
// src/types/acme__audit-logger.d.ts

declare module "@acme/audit-logger" {
  /** Options accepted by the logger — all optional, defaults shown. */
  export interface AuditLoggerOptions {
    /** How often the buffer is flushed. @default 5000 */
    flushIntervalMs?: number;
    /** Max events buffered before a forced flush. @default 100 */
    batchSize?: number;
  }

  export interface AuditMetadata {
    requestId?: string;
    ip?:        string;
    userAgent?: string;
    [key: string]: unknown;   // the JS accepts arbitrary extra keys
  }

  export class AuditLogger {
    constructor(transportUrl: string, options?: AuditLoggerOptions);
    log(actorId: number, action: string, metadata?: AuditMetadata): void;
    /** Resolves with the number of events flushed. */
    flush(): Promise<number>;
    close(): void;
  }

  export const DEFAULTS: Required<AuditLoggerOptions>;

  /** The module's callable default export. */
  function createLogger(transportUrl: string, options?: AuditLoggerOptions): AuditLogger;

  export = createLogger;   // ⚠️ mirrors `module.exports = createLogger`
}
```

`export =` is the TypeScript syntax for "this module's entire export is this one thing" — the exact semantics of CommonJS `module.exports =`. Consumers import it with `import createLogger = require("@acme/audit-logger")` or, with `esModuleInterop: true`, `import createLogger from "@acme/audit-logger"`.

> Note: mixing `export =` with `export class`/`export const` in the same block is not legal. If the package really is `module.exports = fn` *plus* properties, model it as a declaration merge of a function and a namespace — shown in Going Deeper.

**Step 4 — verify against reality.** Write a throwaway script that exercises every declared signature and run it. A `.d.ts` that lies is worse than no `.d.ts`.

---

## Example 1 — basic

```ts
// ══════════════════════════════════════════════════════════════════════════════
// Typing a small untyped package end to end
// ══════════════════════════════════════════════════════════════════════════════

// The package `slugify-id` is plain JS with no types:
//   node_modules/slugify-id/index.js
//     exports.slugify   = (input, options) => "...";
//     exports.unslugify = (slug) => "...";
//     exports.MAX_LENGTH = 96;

// ── src/types/slugify-id.d.ts ────────────────────────────────────────────────
declare module "slugify-id" {
  export interface SlugifyOptions {
    /** Character used between words. @default "-" */
    separator?: string;
    /** Lowercase the result. @default true */
    lowercase?: boolean;
    /** Truncate to this many characters. @default MAX_LENGTH */
    maxLength?: number;
    /** Characters stripped before slugifying. @default /[^\w\s-]/g */
    strip?: RegExp;
  }

  /** Convert an arbitrary string into a URL-safe slug. */
  export function slugify(input: string, options?: SlugifyOptions): string;

  /** Best-effort reverse of `slugify` — separators become spaces. */
  export function unslugify(slug: string): string;

  /** Hard upper bound the library enforces on slug length. */
  export const MAX_LENGTH: number;
}

// ── src/services/article-service.ts ──────────────────────────────────────────
import { slugify, MAX_LENGTH } from "slugify-id";

interface ArticleDraft {
  title:    string;
  body:     string;
  authorId: number;
}

interface Article {
  articleId: number;
  slug:      string;
  title:     string;
  body:      string;
  authorId:  number;
  createdAt: Date;
}

function buildSlug(title: string, articleId: number): string {
  // ✅ Fully checked: options object is validated key by key
  const base = slugify(title, {
    separator: "-",
    lowercase: true,
    maxLength: MAX_LENGTH - 8,   // leave room for the id suffix
  });
  return `${base}-${articleId.toString(36)}`;
}

// ❌ Errors the declaration file now catches for you:
// slugify(title, { seperator: "-" });
//   TS2561: Object literal may only specify known properties.
//           Did you mean to write 'separator'?
// slugify(123);
//   TS2345: Argument of type 'number' is not assignable to parameter of type 'string'.
// const n: number = slugify("hello");
//   TS2322: Type 'string' is not assignable to type 'number'.
// unslugify("a", "b");
//   TS2554: Expected 1 arguments, but got 2.

// ── tsconfig.json — make sure the .d.ts is actually included ──────────────────
// {
//   "compilerOptions": { "strict": true, "noImplicitAny": true, "skipLibCheck": true },
//   "include": ["src/**/*.ts"]     // ← src/types/slugify-id.d.ts matches this glob
// }
//
// Sanity check that tsc sees your file:
//   npx tsc --listFiles | grep slugify-id
```

---

## Example 2 — real world backend use case

```ts
// ══════════════════════════════════════════════════════════════════════════════
// Publishing @acme/api-kit — a shared internal package consumed by 9 services.
// Full setup: source → declaration emit → package.json wiring → consumer view.
// ══════════════════════════════════════════════════════════════════════════════

// ─────────────────────────────────────────────────────────────────────────────
// 1. SOURCE — src/errors.ts
// ─────────────────────────────────────────────────────────────────────────────
export type ErrorCode =
  | "BAD_REQUEST"
  | "UNAUTHORIZED"
  | "FORBIDDEN"
  | "NOT_FOUND"
  | "CONFLICT"
  | "RATE_LIMITED"
  | "INTERNAL";

export interface SerializedApiError {
  code:       ErrorCode;
  message:    string;
  statusCode: number;
  requestId?: string;
  details?:   Record<string, unknown>;
}

export class ApiError extends Error {
  readonly code:       ErrorCode;
  readonly statusCode: number;
  readonly details?:   Record<string, unknown>;

  private constructor(code: ErrorCode, statusCode: number, message: string, details?: Record<string, unknown>) {
    super(message);
    this.name       = "ApiError";
    this.code       = code;
    this.statusCode = statusCode;
    this.details    = details;
  }

  static badRequest(message: string, details?: Record<string, unknown>): ApiError {
    return new ApiError("BAD_REQUEST", 400, message, details);
  }
  static unauthorized(message = "Authentication required"): ApiError {
    return new ApiError("UNAUTHORIZED", 401, message);
  }
  static notFound(resource: string): ApiError {
    return new ApiError("NOT_FOUND", 404, `${resource} not found`);
  }
  static conflict(message: string): ApiError {
    return new ApiError("CONFLICT", 409, message);
  }
  static internal(message = "Internal server error"): ApiError {
    return new ApiError("INTERNAL", 500, message);
  }

  toJSON(requestId?: string): SerializedApiError {
    return { code: this.code, message: this.message, statusCode: this.statusCode, requestId, details: this.details };
  }
}

// ─────────────────────────────────────────────────────────────────────────────
// 2. SOURCE — src/http-client.ts
// ─────────────────────────────────────────────────────────────────────────────
export interface RetryPolicy {
  maxAttempts:   number;
  baseDelayMs:   number;
  retryOnStatus: readonly number[];
}

export interface ClientConfig {
  baseUrl:      string;
  authToken?:   string;
  timeoutMs?:   number;
  retryPolicy?: Partial<RetryPolicy>;
}

export interface ApiResponse<TBody> {
  statusCode: number;
  body:       TBody;
  headers:    Readonly<Record<string, string>>;
  requestId:  string | null;
}

const DEFAULT_RETRY: RetryPolicy = {
  maxAttempts:   3,
  baseDelayMs:   200,
  retryOnStatus: [429, 502, 503, 504],
};

export class ApiClient {
  private readonly baseUrl:     string;
  private readonly authToken?:  string;
  private readonly timeoutMs:   number;
  private readonly retryPolicy: RetryPolicy;

  constructor(config: ClientConfig) {
    this.baseUrl     = config.baseUrl.replace(/\/+$/, "");
    this.authToken   = config.authToken;
    this.timeoutMs   = config.timeoutMs ?? 5_000;
    this.retryPolicy = { ...DEFAULT_RETRY, ...config.retryPolicy };
  }

  async get<TBody>(path: string): Promise<ApiResponse<TBody>> {
    return this.request<TBody>("GET", path);
  }

  async post<TBody, TRequestBody = unknown>(path: string, requestBody: TRequestBody): Promise<ApiResponse<TBody>> {
    return this.request<TBody>("POST", path, requestBody);
  }

  private async request<TBody>(method: string, path: string, requestBody?: unknown): Promise<ApiResponse<TBody>> {
    let lastError: unknown;
    for (let attempt = 1; attempt <= this.retryPolicy.maxAttempts; attempt++) {
      try {
        const res = await fetch(`${this.baseUrl}${path}`, {
          method,
          headers: {
            "content-type": "application/json",
            ...(this.authToken ? { authorization: `Bearer ${this.authToken}` } : {}),
          },
          body:   requestBody === undefined ? undefined : JSON.stringify(requestBody),
          signal: AbortSignal.timeout(this.timeoutMs),
        });

        if (this.retryPolicy.retryOnStatus.includes(res.status) && attempt < this.retryPolicy.maxAttempts) {
          await new Promise((r) => setTimeout(r, this.retryPolicy.baseDelayMs * 2 ** (attempt - 1)));
          continue;
        }

        return {
          statusCode: res.status,
          body:       (await res.json()) as TBody,
          headers:    Object.fromEntries(res.headers.entries()),
          requestId:  res.headers.get("x-request-id"),
        };
      } catch (err) {
        lastError = err;
      }
    }
    throw ApiError.internal(`Request failed after ${this.retryPolicy.maxAttempts} attempts: ${String(lastError)}`);
  }
}

// ─────────────────────────────────────────────────────────────────────────────
// 3. SOURCE — src/index.ts (the public surface)
// ─────────────────────────────────────────────────────────────────────────────
export { ApiError } from "./errors.js";
export type { ErrorCode, SerializedApiError } from "./errors.js";
export { ApiClient } from "./http-client.js";
export type { ApiResponse, ClientConfig, RetryPolicy } from "./http-client.js";

// ─────────────────────────────────────────────────────────────────────────────
// 4. BUILD CONFIG — tsconfig.build.json
// ─────────────────────────────────────────────────────────────────────────────
// {
//   "compilerOptions": {
//     "target":            "ES2022",
//     "module":            "NodeNext",
//     "moduleResolution":  "NodeNext",
//     "outDir":            "./dist",
//     "rootDir":           "./src",
//     "declaration":       true,      // ← emit .d.ts
//     "declarationMap":    true,      // ← "go to definition" jumps into src/*.ts
//     "sourceMap":         true,
//     "strict":            true,
//     "skipLibCheck":      true,
//     "isolatedDeclarations": false
//   },
//   "include": ["src/**/*.ts"],
//   "exclude": ["src/**/*.test.ts", "src/**/__fixtures__/**"]
// }

// ─────────────────────────────────────────────────────────────────────────────
// 5. EMITTED — dist/http-client.d.ts (generated by tsc, never hand-edited)
// ─────────────────────────────────────────────────────────────────────────────
// export interface RetryPolicy {
//     maxAttempts: number;
//     baseDelayMs: number;
//     retryOnStatus: readonly number[];
// }
// export interface ClientConfig {
//     baseUrl: string;
//     authToken?: string;
//     timeoutMs?: number;
//     retryPolicy?: Partial<RetryPolicy>;
// }
// export interface ApiResponse<TBody> {
//     statusCode: number;
//     body: TBody;
//     headers: Readonly<Record<string, string>>;
//     requestId: string | null;
// }
// export declare class ApiClient {
//     private readonly baseUrl;
//     private readonly authToken?;
//     private readonly timeoutMs;
//     private readonly retryPolicy;
//     constructor(config: ClientConfig);
//     get<TBody>(path: string): Promise<ApiResponse<TBody>>;
//     post<TBody, TRequestBody = unknown>(path: string, requestBody: TRequestBody): Promise<ApiResponse<TBody>>;
//     private request;                                  // ← body AND signature elided
// }
// //# sourceMappingURL=http-client.d.ts.map

// ─────────────────────────────────────────────────────────────────────────────
// 6. PACKAGE.JSON — how consumers find the types
// ─────────────────────────────────────────────────────────────────────────────
// {
//   "name":    "@acme/api-kit",
//   "version": "3.2.0",
//   "type":    "module",
//   "main":    "./dist/index.js",
//   "types":   "./dist/index.d.ts",
//   "exports": {
//     ".": {
//       "types":  "./dist/index.d.ts",     // ← FIRST key, always
//       "default": "./dist/index.js"
//     },
//     "./errors": {
//       "types":  "./dist/errors.d.ts",
//       "default": "./dist/errors.js"
//     },
//     "./package.json": "./package.json"
//   },
//   "files": ["dist"],                      // ← without this, .d.ts never ships
//   "scripts": {
//     "build":     "rm -rf dist && tsc -p tsconfig.build.json",
//     "typecheck": "tsc --noEmit",
//     "prepublishOnly": "npm run build && npm run verify:types"
//   },
//   "devDependencies": { "typescript": "^5.6.0", "@arethetypeswrong/cli": "^0.17.0" },
//   "peerDependencies": { "typescript": ">=5.0" },
//   "peerDependenciesMeta": { "typescript": { "optional": true } }
// }
//
// Verify before publishing — catches missing/unreachable/wrong-format types:
//   npx @arethetypeswrong/cli --pack .
//   npx publint

// ─────────────────────────────────────────────────────────────────────────────
// 7. CONSUMER — services/orders-api/src/clients/inventory.ts
// ─────────────────────────────────────────────────────────────────────────────
import { ApiClient, ApiError } from "@acme/api-kit";
import type { ApiResponse } from "@acme/api-kit";

interface InventoryItem {
  sku:            string;
  availableUnits: number;
  reservedUnits:  number;
  warehouseId:    string;
}

interface ReserveRequestBody {
  sku:      string;
  units:    number;
  orderId:  string;
}

const inventoryClient = new ApiClient({
  baseUrl:     process.env.INVENTORY_URL ?? "http://inventory.internal:8080",
  authToken:   process.env.INVENTORY_TOKEN,
  timeoutMs:   3_000,
  retryPolicy: { maxAttempts: 5 },      // ✅ Partial<RetryPolicy> — other fields defaulted
});

export async function reserveStock(orderId: string, sku: string, units: number): Promise<InventoryItem> {
  // ✅ TBody flows through: response.body is InventoryItem, fully autocompleted
  const response: ApiResponse<InventoryItem> = await inventoryClient.post<InventoryItem, ReserveRequestBody>(
    "/v1/reservations",
    { sku, units, orderId },
  );

  if (response.statusCode === 409) throw ApiError.conflict(`Insufficient stock for SKU ${sku}`);
  if (response.statusCode >= 400)  throw ApiError.internal(`Inventory returned ${response.statusCode}`);

  return response.body;    // ✅ InventoryItem — not any, not unknown
}

// ❌ Everything the emitted .d.ts protects the consumer from:
// new ApiClient({ baseURL: "..." });
//   TS2561: Did you mean to write 'baseUrl'?
// inventoryClient.post("/v1/reservations");
//   TS2554: Expected 2 arguments, but got 1.
// new ApiClient({ baseUrl: "x", retryPolicy: { maxAttempts: "5" } });
//   TS2322: Type 'string' is not assignable to type 'number'.
// (await inventoryClient.get<InventoryItem>("/v1/items/1")).body.availabelUnits;
//   TS2551: Did you mean 'availableUnits'?
// new ApiError("NOT_FOUND", 404, "x");
//   TS2673: Constructor of class 'ApiError' is private.
```

---

## Going deeper

### The `private` members leak into your `.d.ts` — and they matter

A `private` field is elided in the emitted declaration, but it is **still declared**. That is deliberate: TypeScript classes with private members are **nominally** typed, not structurally.

```ts
// dist/http-client.d.ts
export declare class ApiClient {
    private readonly baseUrl;   // type elided, existence preserved
    constructor(config: ClientConfig);
    get<TBody>(path: string): Promise<ApiResponse<TBody>>;
}
```

```ts
// A consumer cannot hand-roll a compatible object:
const fakeClient: ApiClient = {
  get: async () => ({ statusCode: 200, body: {}, headers: {}, requestId: null }),
};
// ❌ TS2739: Type '{ get: ... }' is missing the following properties from type
//    'ApiClient': baseUrl, authToken, timeoutMs, retryPolicy
```

Consequences you should plan for:

- Consumers **cannot mock your class** with a plain object literal. If you want mockability, export an `interface` alongside the class and type parameters as the interface.
- **Renaming a private field is a breaking change to the `.d.ts`** even though it is invisible in your public API — it changes the emitted declaration. It won't break consumers in practice (they can't reference it), but it does produce a diff and can break `tsc --build` incremental caches.

```ts
// ✅ Design for mockability — export the shape, not just the class:
export interface InventoryClient {
  get<TBody>(path: string): Promise<ApiResponse<TBody>>;
  post<TBody, TReq>(path: string, requestBody: TReq): Promise<ApiResponse<TBody>>;
}
export declare class ApiClient implements InventoryClient { /* ... */ }
// Consumers type against InventoryClient → trivially mockable in tests.
```

If you use `#private` (native ECMAScript private fields) instead, the declaration emits a single `#private;` marker with the same nominal effect.

### "The inferred type of X cannot be named without a reference to Y"

The single most confusing error when you turn on `declaration: true`:

```ts
// src/repo.ts
import { PoolClient } from "pg";   // pg is a transitive/dev dependency

// ❌ TS4023: Exported variable 'connection' has or is using name 'PoolClient'
//    from external module "pg" but cannot be named.
export const connection = getPoolClient();   // inferred type: PoolClient
```

The compiler must write a *name* for this type into the `.d.ts`, and the name it needs isn't importable from where the declaration lives. Three fixes, in order of preference:

```ts
// ✅ Fix 1 — annotate explicitly, forcing an explicit import in the emitted .d.ts:
import type { PoolClient } from "pg";
export const connection: PoolClient = getPoolClient();

// ✅ Fix 2 — re-export the type so it's part of your public surface:
export type { PoolClient } from "pg";

// ✅ Fix 3 — widen to a type you own, hiding the dependency entirely:
export interface DbConnection {
  query<T>(sql: string, params?: unknown[]): Promise<{ rows: T[] }>;
  release(): void;
}
export const connection: DbConnection = getPoolClient();
```

The general rule: **anything reachable from your public API must be importable by consumers.** If `pg` is a `devDependency` but its types appear in your `.d.ts`, consumers who don't have `pg` installed get broken types. Move it to `dependencies` or `peerDependencies`, or hide it behind your own interface.

TypeScript 5.5+ can sometimes synthesize the import itself, but relying on that is fragile. Annotate explicitly.

### `isolatedDeclarations` — the fast-declaration-emit mode

TypeScript 5.5 added `isolatedDeclarations: true`. It forbids any exported declaration whose type must be *inferred*, requiring explicit annotations on everything public.

```ts
// With isolatedDeclarations: true

// ❌ TS9007: Function must have an explicit return type annotation
//            with --isolatedDeclarations.
export function buildAuthToken(userId: number) {
  return { userId, issuedAt: Date.now(), scope: "api" as const };
}

// ✅ Explicit:
export interface AuthToken {
  userId:   number;
  issuedAt: number;
  scope:    "api";
}
export function buildAuthToken(userId: number): AuthToken {
  return { userId, issuedAt: Date.now(), scope: "api" };
}
```

Why anyone would opt into that verbosity: with fully explicit types, a `.d.ts` can be generated from a single file **without reading any other file**. That makes declaration emit parallelizable and lets non-tsc tools (esbuild, swc, oxc) generate declarations at native speed. On a 200-package monorepo this turns a multi-minute serial `tsc --build` into seconds. It is a monorepo-scale tradeoff; on a single service it is not worth it.

### `.d.ts` vs `.d.mts` vs `.d.cts`

Under `moduleResolution: "NodeNext"`, the extension of a declaration file determines the module system it describes:

| File | Describes | `import`/`require` inside it means |
|---|---|---|
| `foo.d.ts` | Format inferred from nearest `package.json` `"type"` | Depends on `"type": "module"` or not |
| `foo.d.mts` | ESM | Real ESM semantics |
| `foo.d.cts` | CommonJS | `export =` / `import x = require(...)` |

If you ship a dual package, you need both `.d.mts` and `.d.cts` — a single `.d.ts` will be misinterpreted by one of the two consumers, typically producing "This expression is not callable" on the CJS side. `@arethetypeswrong/cli` exists precisely to catch this class of bug and is worth wiring into `prepublishOnly`.

### Declaration merging: function + namespace for CJS callable exports

Earlier we hit the limitation that `export =` can't coexist with other exports. The real answer for `module.exports = fn; module.exports.Foo = Foo;` is a **declaration merge**:

```ts
declare module "@acme/audit-logger" {
  interface AuditLoggerOptions {
    flushIntervalMs?: number;
    batchSize?:       number;
  }

  class AuditLogger {
    constructor(transportUrl: string, options?: AuditLoggerOptions);
    log(actorId: number, action: string, metadata?: Record<string, unknown>): void;
    flush(): Promise<number>;
    close(): void;
  }

  // The callable part:
  function createLogger(transportUrl: string, options?: AuditLoggerOptions): AuditLogger;

  // The namespace part — merges with the function of the same name,
  // giving createLogger.AuditLogger and createLogger.DEFAULTS:
  namespace createLogger {
    export { AuditLogger, AuditLoggerOptions };
    export const DEFAULTS: Required<AuditLoggerOptions>;
  }

  export = createLogger;
}
```

Now all three usage styles type-check:

```ts
import createLogger = require("@acme/audit-logger");

const logger = createLogger("https://audit.internal", { batchSize: 50 });  // callable ✅
const other  = new createLogger.AuditLogger("https://audit.internal");     // namespace ✅
const size   = createLogger.DEFAULTS.batchSize;                            // namespace ✅
```

### The performance cost of `.d.ts` files you don't need

`@types/node` alone is roughly 100,000 lines of declarations. Every `@types/*` package auto-included by the default `types` behaviour is parsed and (without `skipLibCheck`) fully checked on every compile.

```jsonc
// Measure it — these flags print exactly where time goes:
// npx tsc --noEmit --diagnostics
// npx tsc --noEmit --extendedDiagnostics
// npx tsc --generateTrace ./trace && npx @typescript/analyze-trace ./trace
//
// Typical wins on a mid-size Node service:
//   skipLibCheck: true                       → 30-50% faster type check
//   "types": ["node"] instead of default     → excludes jest/mocha/react globals
//   "lib": ["ES2022"] instead of default     → skips ~40k lines of DOM declarations
```

```jsonc
// A tuned Node-service tsconfig:
{
  "compilerOptions": {
    "lib":          ["ES2022"],       // no DOM — this is a server
    "types":        ["node"],         // no accidental jest/react globals
    "skipLibCheck": true,
    "incremental":  true,             // reuse .tsbuildinfo between runs
    "tsBuildInfoFile": "./node_modules/.cache/tsbuildinfo"
  }
}
```

### A `.d.ts` file can lie, and nothing will tell you

This is the fundamental limitation and it deserves to be stated plainly.

```ts
// src/types/redis-lite.d.ts
declare module "redis-lite" {
  // The real function returns `string | null`. This declaration says `string`.
  export function get(key: string): Promise<string>;
}
```

```ts
// Consumer code — compiles cleanly, crashes at runtime:
import { get } from "redis-lite";

const sessionJson = await get(`session:${authToken}`);
const session = JSON.parse(sessionJson);   // ✅ compiles — sessionJson is `string`
                                           // 💥 TypeError: "null" is not valid JSON
```

Defensive practices:

1. **Read the implementation, not the README.** READMEs omit null returns constantly.
2. **Prefer `unknown` over a confident lie.** `get(key: string): Promise<unknown>` forces the consumer to validate — annoying but honest.
3. **Validate at the boundary.** Pair the declaration with a runtime schema (`zod`, `valibot`) for anything crossing a network or process boundary.
4. **Contribute upstream.** If you wrote a good `.d.ts` for a public package, send it to DefinitelyTyped. The next person — possibly you in six months — inherits it.

```ts
// ✅ Honest declaration + runtime validation at the boundary:
declare module "redis-lite" {
  export function get(key: string): Promise<string | null>;   // matches reality
}

import { get } from "redis-lite";
import { z } from "zod";

const SessionSchema = z.object({
  userId:    z.number().int().positive(),
  authToken: z.string().min(32),
  expiresAt: z.coerce.date(),
});
type Session = z.infer<typeof SessionSchema>;

async function loadSession(authToken: string): Promise<Session | null> {
  const raw = await get(`session:${authToken}`);   // string | null — must be handled
  if (raw === null) return null;
  const parsed = SessionSchema.safeParse(JSON.parse(raw));
  return parsed.success ? parsed.data : null;
}
```

---

## Common mistakes

### Mistake 1 — Writing implementation code in a `.d.ts` file

```ts
// ❌ src/types/config.d.ts — this file emits nothing, so the constant does not exist at runtime
export const DEFAULT_PAGE_SIZE = 50;              // TS1039 in ambient context

export function parseAuthToken(header: string) {  // TS1183: implementation not allowed
  return header.replace(/^Bearer /, "");
}

export interface PaginationOptions {
  pageSize: number;
  cursor?:  string;
}
```

```ts
// ✅ Split them. Runtime values go in a real .ts file:
// src/config.ts
export const DEFAULT_PAGE_SIZE = 50;

export function parseAuthToken(header: string): string {
  return header.replace(/^Bearer /, "");
}

export interface PaginationOptions {   // types can live here too — this is fine
  pageSize: number;
  cursor?:  string;
}
```

```ts
// ✅ A .d.ts only DESCRIBES things that already exist elsewhere:
// src/types/legacy-config.d.ts
declare module "@acme/legacy-config" {
  export const DEFAULT_PAGE_SIZE: number;                    // declared, not defined
  export function parseAuthToken(header: string): string;    // declared, not defined
}
```

The failure mode when you get this wrong is nasty: your editor autocompletes `DEFAULT_PAGE_SIZE`, the build passes, and production throws `ReferenceError` — because there was never any JavaScript emitted for that file.

### Mistake 2 — Installing `@types/x` for a package that ships its own types

```bash
# ❌ zod, prisma, fastify, hono, drizzle-orm all bundle their own .d.ts files
npm i -D @types/zod
# npm WARN deprecated @types/zod@3.x: This is a stub types definition.
#   zod provides its own type definitions, so you do not need this installed.
```

The real damage when a *non-stub* stale `@types` package exists: two declaration sources for the same module, and resolution order decides which wins.

```ts
// ❌ Symptom of a stale @types package shadowing bundled types:
import { z } from "zod";
const UserSchema = z.object({ userId: z.number() });
type User = z.infer<typeof UserSchema>;
//                   ^ TS2339: Property 'infer' does not exist on type 'typeof z'
//                     — the stale @types/zod predates z.infer
```

```bash
# ✅ Check before installing:
node -e "const p=require('./node_modules/zod/package.json'); console.log(p.types ?? p.typings ?? 'none')"
# → ./index.d.ts   → bundled types exist, do NOT install @types/zod

# ✅ Remove any stale stubs you already have:
npm uninstall @types/zod
npm ls @types/node          # check for duplicated/nested copies while you're here
```

### Mistake 3 — Your `.d.ts` file exists but tsc never reads it

```jsonc
// ❌ tsconfig.json — .d.ts lives in ./types/ but include only covers ./src/
{
  "compilerOptions": { "strict": true },
  "include": ["src/**/*.ts"]
}
// types/legacy-cache.d.ts is outside the include glob → never loaded
// → TS7016: Could not find a declaration file for module '@acme/legacy-cache'
```

```jsonc
// ✅ Option A — move the file under src/ (simplest, works with the existing glob)
//    src/types/legacy-cache.d.ts

// ✅ Option B — extend the include glob
{
  "compilerOptions": { "strict": true },
  "include": ["src/**/*.ts", "types/**/*.d.ts"]
}

// ✅ Option C — reference it explicitly (survives any include config)
{
  "compilerOptions": { "strict": true },
  "files":   ["types/legacy-cache.d.ts"],
  "include": ["src/**/*.ts"]
}
```

```bash
# ✅ Always verify rather than assume — this is the debugging command to remember:
npx tsc --noEmit --listFiles | grep legacy-cache
# no output → tsc is not reading your declaration file
```

The trap within the trap: if your `.d.ts` accidentally contains a top-level `import` or `export`, it becomes a **module** rather than a global script, and its `declare module "..."` block stops being ambient. See `50 — Ambient declarations` for the full explanation.

### Mistake 4 — Publishing a package whose `.d.ts` files never ship

```jsonc
// ❌ package.json — "types" points at a file npm never packed
{
  "name":  "@acme/api-kit",
  "main":  "./dist/index.js",
  "types": "./dist/index.d.ts",
  "files": ["dist/**/*.js"]      // ← .d.ts excluded by the glob!
}
// Consumers: TS7016 "Could not find a declaration file for module '@acme/api-kit'"
```

```jsonc
// ✅ Ship the whole dist directory:
{
  "name":  "@acme/api-kit",
  "main":  "./dist/index.js",
  "types": "./dist/index.d.ts",
  "files": ["dist"],
  "scripts": { "prepublishOnly": "npm run build && npx publint && npx @arethetypeswrong/cli --pack ." }
}
```

```bash
# ✅ Verify what actually gets published — before you publish:
npm pack --dry-run          # lists every file that will be in the tarball
npx publint                 # lints package.json main/types/exports wiring
npx @arethetypeswrong/cli --pack .   # simulates resolution from CJS and ESM consumers
```

### Mistake 5 — Turning off `noImplicitAny` to silence TS7016

```jsonc
// ❌ "The untyped import is annoying, let me just..."
{ "compilerOptions": { "strict": true, "noImplicitAny": false } }
```

```ts
// Now this compiles — and every value derived from it is `any`, silently:
import { createLimiter } from "@acme/rate-limiter";   // any
const limiter = createLimiter({ maxRequets: 100 });   // any — typo invisible
limiter.rest(userId);                                 // any — crashes at runtime
```

`noImplicitAny: false` doesn't just affect that one import. It disables implicit-any errors across your entire codebase, including untyped function parameters and untyped catch bindings. You traded one localized error for global blindness.

```ts
// ✅ Fix the actual hole. Either write a real declaration...
declare module "@acme/rate-limiter" {
  export interface LimiterOptions { windowMs: number; maxRequests: number; keyPrefix?: string }
  export interface RateLimiter {
    check(key: string): { allowed: boolean; remaining: number; resetAt: Date };
    reset(key: string): void;
  }
  export function createLimiter(options: LimiterOptions): RateLimiter;
}

// ...or, if you genuinely can't yet, scope the escape hatch to ONE module:
// src/types/shims.d.ts
declare module "@acme/rate-limiter";   // this module only is `any` — everything else stays strict
```

The second form is honest: it is grep-able, it is one line, and it doesn't cost you type safety anywhere else.

---

## Practice exercises

### Exercise 1 — easy

An internal package `@acme/cursor-paginate` ships as plain JavaScript with no types. Here is its complete source:

```js
// node_modules/@acme/cursor-paginate/index.js
exports.encodeCursor = function (record, sortField) {
  return Buffer.from(JSON.stringify({ v: record[sortField], id: record.id })).toString("base64url");
};

exports.decodeCursor = function (cursor) {
  try { return JSON.parse(Buffer.from(cursor, "base64url").toString("utf8")); }
  catch { return null; }        // returns null on malformed input
};

exports.buildPage = function (rows, limit, sortField) {
  const hasMore = rows.length > limit;
  const items   = hasMore ? rows.slice(0, limit) : rows;
  return {
    items,
    hasMore,
    nextCursor: hasMore ? exports.encodeCursor(items[items.length - 1], sortField) : null,
  };
};

exports.MAX_PAGE_SIZE = 200;
```

Write a declaration file `src/types/acme__cursor-paginate.d.ts` that types this package. Requirements:

1. Use `declare module` with the correct module specifier.
2. `encodeCursor` and `buildPage` must be **generic** over the record type, constrained to records that have an `id: number`.
3. `decodeCursor` must return `{ v: unknown; id: number } | null` — model the null path honestly.
4. Export an interface `Page<TRecord>` describing the `buildPage` return value and use it as the return type.
5. `MAX_PAGE_SIZE` must be declared as a `number` constant.

Then write a consuming file `src/repositories/order-repository.ts` that imports it and implements `listOrdersPage(userId: number, limit: number, cursor?: string): Promise<Page<Order>>`, where `Order` is `{ id: number; userId: number; totalCents: number; createdAt: Date }`. Show at least three call-site mistakes the declaration file now catches, as comments.

```ts
// Write your code here
```

### Exercise 2 — medium

You are publishing `@acme/queue-kit`, a shared job-queue wrapper consumed by six backend services. Build the complete publishable setup **from scratch**.

Write the source in `src/`:

- `src/types.ts` — a `JobPayloadMap` interface mapping job names to payload shapes (`"email.send"`, `"report.generate"`, `"user.reindex"`), a `JobName = keyof JobPayloadMap` type, a `JobOptions` interface (`delayMs?`, `maxAttempts?`, `priority?: "low" | "normal" | "high"`), and a `JobHandle` interface (`jobId: string`, `enqueuedAt: Date`, `name: JobName`).
- `src/queue.ts` — a `JobQueue` class with a `private` connection field, `enqueue<TName extends JobName>(name: TName, payload: JobPayloadMap[TName], options?: JobOptions): Promise<JobHandle>`, `registerHandler<TName extends JobName>(name: TName, handler: (payload: JobPayloadMap[TName]) => Promise<void>): void`, and `start(): Promise<void>`.
- `src/index.ts` — the public surface, re-exporting values and types separately.

Then produce, as commented blocks:

1. `tsconfig.build.json` with `declaration`, `declarationMap`, `outDir`, `rootDir`, and appropriate `exclude`.
2. The `.d.ts` you expect `tsc` to emit for `src/queue.ts` — write it by hand and reason about what happens to the `private` field and the generic signatures.
3. A `package.json` with `main`, `types`, a full `exports` map including a `./types` subpath, `files`, and a `prepublishOnly` script.
4. A consumer file showing type-safe usage — and at least four compile errors the types now catch, including one where the payload doesn't match the job name.

Finally, explain in comments why `enqueue("email.send", { reportId: 7 })` is an error and exactly which type mechanism produces it.

```ts
// Write your code here
```

### Exercise 3 — hard

Your team inherits a legacy internal SDK, `@acme/legacy-billing`, published only as CommonJS JavaScript. Here is its full runtime shape:

```js
// node_modules/@acme/legacy-billing/index.js
function BillingClient(apiKey, opts) {
  if (!(this instanceof BillingClient)) return new BillingClient(apiKey, opts);
  this.apiKey  = apiKey;
  this.baseUrl = (opts && opts.baseUrl) || "https://billing.acme.internal";
}
BillingClient.prototype.charge      = function (customerId, amountCents, currency, cb) {};  // node-style callback
BillingClient.prototype.refund      = function (paymentId, cb) {};
BillingClient.prototype.getInvoice  = function (invoiceId, cb) {};

function withRetry(client, maxAttempts) { /* returns a proxied BillingClient */ }

module.exports          = BillingClient;   // callable AND newable
module.exports.withRetry = withRetry;
module.exports.CURRENCIES = ["usd", "eur", "gbp", "inr"];
module.exports.VERSION    = "1.4.2";
```

Deliver all of the following:

1. **A complete ambient declaration** `src/types/acme__legacy-billing.d.ts` that models: the dual callable/newable constructor, the prototype methods with Node-style `(err, result)` callbacks, the `withRetry` helper preserving the client type, `CURRENCIES` as a `readonly` tuple of literal types (not `string[]`), and `VERSION`. Use the function + namespace declaration-merging pattern and `export =`.

2. **A typed promise wrapper** `src/billing/billing-client.ts` that wraps the callback API in promises, exposing `chargeCustomer`, `refundPayment`, and `fetchInvoice` returning a discriminated `Result<T, BillingError>` union rather than throwing. Define `Currency` as `(typeof CURRENCIES)[number]` derived from your declaration.

3. **A published-package configuration** for your wrapper: `tsconfig.build.json` with `emitDeclarationOnly: true` (esbuild builds the JS), plus the `package.json` `exports` map with correct `types`-first ordering and both `.d.mts`/`.d.cts` outputs for a dual package.

4. **Written analysis, in comments**, covering: (a) why `export =` cannot coexist with named `export` declarations and how the namespace merge works around it; (b) what happens to the emitted `.d.ts` if `@acme/legacy-billing` is a `devDependency` rather than a `dependency`, and which error code you would see; (c) why `skipLibCheck: true` might hide a real bug in your own declaration file and what CI step you would add to catch it; (d) how you would verify the whole package resolves correctly for both a CJS and an ESM consumer.

```ts
// Write your code here
```

---

## Quick reference cheat sheet

```ts
// ── Declaring types in a .d.ts ────────────────────────────────────────────────
export interface UserSchema { userId: number; email: string }  // no `declare` needed
export type AuthRole = "admin" | "member";                     // no `declare` needed
export declare const API_VERSION: string;                      // value → needs `declare`
export declare function verifyToken(authToken: string): boolean;
export declare class UserRepository { findById(userId: number): Promise<UserSchema> }

// ── Typing a package you don't own ────────────────────────────────────────────
declare module "@acme/untyped-pkg" {           // specifier must match the import exactly
  export function doThing(input: string): number;
}
declare module "@acme/really-untyped";         // no body → whole module is `any`

// ── CommonJS `module.exports = fn` ────────────────────────────────────────────
declare module "some-cjs-pkg" {
  function main(options?: object): void;
  namespace main { export const VERSION: string }   // merges → main.VERSION
  export = main;
}
```

```jsonc
// ── tsconfig.json ─────────────────────────────────────────────────────────────
{
  "compilerOptions": {
    "declaration":          true,     // emit .d.ts
    "declarationMap":       true,     // emit .d.ts.map → go-to-def hits source
    "declarationDir":       "./dist/types",
    "emitDeclarationOnly":  false,    // true → only .d.ts (esbuild/swc emits JS)
    "isolatedDeclarations": false,    // true → require explicit types on all exports
    "skipLibCheck":         true,     // don't check inside .d.ts files
    "typeRoots":            ["./node_modules/@types"],
    "types":                ["node"], // ONLY these are auto-included globally
    "lib":                  ["ES2022"]
  },
  "include": ["src/**/*.ts"]          // must cover your .d.ts files
}
```

```jsonc
// ── package.json ──────────────────────────────────────────────────────────────
{
  "main":    "./dist/index.js",
  "types":   "./dist/index.d.ts",     // "typings" = legacy alias, same meaning
  "exports": { ".": { "types": "./dist/index.d.ts", "default": "./dist/index.js" } },
  "files":   ["dist"]                 // .d.ts must be inside a packed path
}
```

| Thing | What it does |
|---|---|
| `.d.ts` | Types only, no implementation, emits nothing |
| `declare` | "This value exists at runtime — here is its type" |
| `@types/express` | Community types for `express` from DefinitelyTyped |
| `@types/acme__logger` | Scoped package `@acme/logger` — `/` becomes `__` |
| `declaration: true` | tsc emits `.d.ts` from your `.ts` sources |
| `declarationMap: true` | Go-to-definition lands in `.ts`, not `.d.ts` |
| `emitDeclarationOnly` | tsc emits only types; another tool builds the JS |
| `isolatedDeclarations` | Forces explicit types on exports → parallel `.d.ts` emit |
| `"types"` in package.json | Entry point for consumers' type resolution |
| `"exports"` `types` key | Must be listed **first** in each condition object |
| `typeRoots` | Directories searched for auto-included type packages |
| `types: []` | Disables automatic `@types/*` global inclusion |
| `skipLibCheck: true` | Skip checking inside `.d.ts` — faster, hides dep errors |
| `TS7016` | "Could not find a declaration file for module 'x'" |
| `TS4023` | Exported type "cannot be named" → annotate explicitly |
| `TS1183` | "An implementation cannot be declared in ambient contexts" |
| `tsc --listFiles` | Prove which `.d.ts` files the compiler actually loaded |
| `@arethetypeswrong/cli` | Verify your published types resolve for CJS **and** ESM |

---

## Connected topics

- **50 — Ambient declarations** — the `declare` keyword in depth, `declare global`, and why a stray `export {}` changes whether your `.d.ts` is a script or a module.
- **51 — Module augmentation** — adding properties to types that already exist in `@types/*` packages, like putting `req.user` on Express's `Request`.
- **03 — tsconfig in depth** — the full picture on `include`, `files`, `typeRoots`, `lib`, and `moduleResolution` that governs which `.d.ts` files get loaded.
- **14 — Interfaces** — the declaration form you will write most inside `.d.ts` files, and the only one that supports the merging that augmentation depends on.
