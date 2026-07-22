# 48 — Modules in TypeScript

## What is this?

A **module** is a file that has its own scope. Anything declared inside it is private to it unless explicitly `export`ed, and anything it needs from elsewhere must be explicitly `import`ed.

```ts
// src/services/userService.ts  — this file is a MODULE, because it has an export.

export interface User {
  userId: number;
  email:  string;
}

const CACHE_TTL_SECONDS = 300;         // private to this file — no other file can see it

export async function findUserById(userId: number): Promise<User | null> {
  return database.users.findOne({ userId });
}
```

```ts
// src/routes/userRoutes.ts

import { findUserById } from "../services/userService.js";
import type { User } from "../services/userService.js";   // type-only import — erased at build
```

TypeScript did not invent modules; it inherits ECMAScript modules (ESM) and understands CommonJS (`require` / `module.exports`) as well. What TypeScript **adds** is:

1. A second kind of thing you can export — **types**, which exist only at compile time and must vanish from the emitted JavaScript.
2. A **module resolution** engine that has to figure out which `.ts` / `.d.ts` file a specifier like `"../services/userService.js"` refers to, and under which module system.
3. Interop syntax (`export =`, `import x = require()`) for describing CommonJS shapes that ESM syntax cannot express.

That combination — a type system layered on top of two competing module systems that Node itself is still reconciling — is why "why won't my import work" is the single most common TypeScript question in Node backends.

## Why does it matter?

Modules are where TypeScript stops being a type checker and starts being a build tool, and it is the area where a wrong `tsconfig.json` line produces runtime errors that the type checker happily approved.

Concretely, in a real backend:

- You import an Express middleware, it compiles clean, and at runtime you get `TypeError: express is not a function` — because `esModuleInterop` was off and the default import did not mean what you thought.
- You migrate to `"type": "module"` in `package.json`, and Node throws `ERR_MODULE_NOT_FOUND: Cannot find module '/app/dist/services/userService'` — because ESM requires file extensions and TypeScript did not add them for you.
- You add `isolatedModules` (required by every bundler, by `ts-node --swc`, by esbuild, by Vite) and suddenly dozens of files fail with *"'User' is a type and must be imported using a type-only import"*.
- You create a `src/index.ts` barrel that re-exports 60 modules, and your cold start time triples because importing one function now loads the entire application graph — including the database driver, in a Lambda that only wanted a validator.
- Two services import each other, and in production you get `Cannot read properties of undefined (reading 'findUserById')` at 3am — a circular import that worked in dev because of load order luck.

None of these are type errors. All of them are module errors. Understanding this doc is what makes them ten-second fixes instead of afternoon-long ones.

## The JavaScript way vs the TypeScript way

```js
// ═══ JavaScript (CommonJS) — the way most Node backends still look ═══════════
// src/services/userService.js

const { database } = require("../db/connection");

const CACHE_TTL_SECONDS = 300;

async function findUserById(userId) {
  return database.users.findOne({ userId });
}

async function createUser(requestBody) {
  // What shape is requestBody? Read the caller. Or the docs. Or guess.
  return database.users.insert(requestBody);
}

module.exports = { findUserById, createUser };
```

```js
// src/routes/userRoutes.js

const express = require("express");
// Is it a function? An object with .Router? Both? Only running it tells you.

const { findUserById, creatUser } = require("../services/userService");
//                     ^^^^^^^^^ typo. `creatUser` is `undefined`.
//                     No error here. No error at import time.
//                     TypeError: creatUser is not a function — at request time, in prod.

const router = express.Router();

router.get("/users/:userId", async (req, res) => {
  const user = await findUserById(req.params.userId);
  //                              ^ this is a STRING. findUserById expects a number.
  //                                Mongo returns null. You debug for an hour.
  res.json(user);
});

module.exports = router;
```

And there is no such thing as importing a *type*, because there are no types. So the "contract" between two files lives entirely in a JSDoc comment nobody updated:

```js
/**
 * @param {Object} requestBody
 * @param {string} requestBody.email
 * @param {string} requestBody.password   <- renamed to passwordHash 6 months ago
 */
```

```ts
// ═══ TypeScript — the same two files ═════════════════════════════════════════
// src/services/userService.ts

import { database } from "../db/connection.js";
import type { CreateUserDto, User } from "../types/user.js";   // types, erased at build

const CACHE_TTL_SECONDS = 300;

export async function findUserById(userId: number): Promise<User | null> {
  return database.users.findOne({ userId });
}

export async function createUser(requestBody: CreateUserDto): Promise<User> {
  return database.users.insert(requestBody);
}
```

```ts
// src/routes/userRoutes.ts

import express from "express";        // compiler KNOWS this is callable and has .Router
import { findUserById, creatUser } from "../services/userService.js";
//                     ^^^^^^^^^^
//     ❌ Error: Module '"../services/userService.js"' has no exported member 'creatUser'.
//        Did you mean 'createUser'?
//        Caught in your editor, before you save. Not at 3am.

const router = express.Router();

router.get("/users/:userId", async (req, res) => {
  const user = await findUserById(req.params.userId);
  //                              ^^^^^^^^^^^^^^^^^
  //     ❌ Error: Argument of type 'string' is not assignable to parameter of type 'number'.
  res.json(user);
});

export default router;
```

The satisfying part is not just the typo catch. It is that **the module graph itself becomes typed**: rename `createUser` and every importer breaks at compile time; delete an export and you find every consumer in one build; change a parameter type and the compiler walks the whole graph for you. The import list stops being a hopeful string and becomes a checked contract.

And the type-only import is the piece that has no JS equivalent at all — a dependency that exists for the checker and is **completely absent from the emitted JavaScript**, so importing a type from a heavy module costs zero runtime bytes and creates zero runtime cycles.

---

## Syntax

```ts
// ══ EXPORTING ═══════════════════════════════════════════════════════════════

// ── Inline named exports ────────────────────────────────────────────────────
export const MAX_PAGE_SIZE = 100;
export function findUserById(userId: number) { /* ... */ }
export class UserRepository {}
export interface User { userId: number; email: string }
export type ApiResponse<T> = { ok: true; data: T } | { ok: false; error: string };
export enum UserRole { Admin = "admin", Member = "member" }

// ── Export list at the bottom (values and types together) ───────────────────
const DEFAULT_TIMEOUT_MS = 5_000;
interface RequestContext { requestId: string; userId: number }
export { DEFAULT_TIMEOUT_MS, RequestContext };

// ── Export list with the `type` modifier (TS 3.8+) ──────────────────────────
export { DEFAULT_TIMEOUT_MS, type RequestContext };   // inline type modifier
export type { RequestContext };                       // whole-clause type export

// ── Renaming on export ──────────────────────────────────────────────────────
export { findUserById as getUser, UserRepository as Repository };

// ── Default export (one per module) ─────────────────────────────────────────
export default function createServer() { /* ... */ }
// or
const router = express.Router();
export default router;

// ── Re-exports (barrel files) ───────────────────────────────────────────────
export { findUserById, createUser } from "./userService.js";
export type { User, CreateUserDto } from "./types.js";
export * from "./orderService.js";              // re-export everything
export * as orders from "./orderService.js";    // re-export as a namespace object
export { default as UserRepository } from "./userRepository.js";

// ── CommonJS-style export (TS-only syntax) ──────────────────────────────────
class LegacyLogger {}
export = LegacyLogger;          // compiles to `module.exports = LegacyLogger`


// ══ IMPORTING ═══════════════════════════════════════════════════════════════

// ── Named imports ───────────────────────────────────────────────────────────
import { findUserById, createUser } from "./userService.js";
import { findUserById as getUser } from "./userService.js";

// ── Default import ──────────────────────────────────────────────────────────
import express from "express";
import router, { type RouteConfig } from "./routes.js";   // default + named + type

// ── Namespace import ────────────────────────────────────────────────────────
import * as fs from "node:fs";
import * as userService from "./userService.js";

// ── Type-only imports ───────────────────────────────────────────────────────
import type { User, ApiResponse } from "./types.js";       // whole clause is types
import { type User, findUserById } from "./userService.js"; // inline type modifier
import type express from "express";                         // type-only default
import type * as Http from "node:http";                     // type-only namespace

// ── Side-effect-only import (runs the module, imports nothing) ──────────────
import "./instrumentation/tracing.js";
import "reflect-metadata";

// ── CommonJS-style import (TS-only syntax) ──────────────────────────────────
import LegacyLogger = require("./legacyLogger");   // pairs with `export =`

// ── Dynamic import — returns a Promise, works in both ESM and CJS output ────
const { PaymentGateway } = await import("./payments/gateway.js");

// ── Type-only dynamic import (in type position, zero runtime cost) ──────────
type Gateway = import("./payments/gateway.js").PaymentGateway;


// ══ THE tsconfig KNOBS THAT CONTROL ALL OF THE ABOVE ════════════════════════
// {
//   "compilerOptions": {
//     "module":                       "nodenext",  // what syntax to EMIT
//     "moduleResolution":             "nodenext",  // how to FIND files
//     "esModuleInterop":              true,        // fix default-importing CJS
//     "allowSyntheticDefaultImports":  true,       // implied by esModuleInterop
//     "isolatedModules":              true,        // each file transpilable alone
//     "verbatimModuleSyntax":         true,        // emit imports exactly as written
//     "resolveJsonModule":            true,        // allow `import cfg from "./x.json"`
//     "allowImportingTsExtensions":   false        // allow `from "./x.ts"` (noEmit only)
//   }
// }
```

---

## How it works — concept by concept

### Concept 1 — What makes a file a module (and the global-scope trap)

TypeScript's rule is mechanical and surprises everyone once:

> A file is a **module** if it contains a top-level `import` or `export`. Otherwise it is a **script**, and everything it declares goes into the global scope.

```ts
// src/constants.ts  — NO import, NO export
const MAX_RETRIES = 3;          // ← this is now a GLOBAL declaration
```

```ts
// src/httpClient.ts — NO import, NO export
const MAX_RETRIES = 5;
// ❌ Error: Cannot redeclare block-scoped variable 'MAX_RETRIES'.
//    Two unrelated files collide, because neither is a module.
```

```ts
// ✅ Add any export and the file gets its own scope:
export const MAX_RETRIES = 3;   // now file-scoped, no collision
```

If you genuinely have a file with no exports (a pure side-effect file, or a `.d.ts` that only augments globals) but want it to be a module, add the empty export marker:

```ts
// src/instrumentation/tracing.ts
import { trace } from "@opentelemetry/api";

trace.getTracer("api").startActiveSpan("boot", (span) => span.end());

export {};   // ← makes this a module even though it exports nothing
```

The inverse matters for declaration files. A `.d.ts` with **no** top-level import/export declares globals; the moment you add one import, everything inside becomes module-scoped and your global augmentation stops working (see doc **49**).

```ts
// types/global.d.ts — SCRIPT: declares real globals
declare global { }                       // not even needed here
declare const APP_VERSION: string;       // ✅ visible everywhere

// types/express.d.ts — MODULE (it has an import), so you must use `declare global`
import type { User } from "../src/types/user.js";

declare global {
  namespace Express {
    interface Request {
      currentUser?: User;
      requestId: string;
    }
  }
}
export {};
```

### Concept 2 — Named exports, default exports, and why backends prefer named

Every module can have any number of **named** exports and at most one **default** export.

```ts
// src/services/paymentService.ts

export const STRIPE_API_VERSION = "2026-01-01";              // named
export interface ChargeRequest { userId: number; amountCents: number }  // named type
export async function chargeCard(req: ChargeRequest) { /* ... */ }      // named
export default class PaymentService { /* ... */ }                       // default
```

```ts
// Importing them:
import PaymentService, { chargeCard, STRIPE_API_VERSION } from "./paymentService.js";
import type { ChargeRequest } from "./paymentService.js";

// The default has no intrinsic name at the import site — you pick one:
import Anything from "./paymentService.js";   // ✅ legal, and a real problem
```

The mechanical differences that matter:

```ts
// 1. Named imports are CHECKED against the export list. Defaults are not named at all.
import { chargeCrad } from "./paymentService.js";  // ❌ compile error: no such export
import PaymentServcie from "./paymentService.js";  // ✅ compiles — it's just a local name

// 2. Rename refactors follow named exports across the codebase.
//    Rename a default export and every importer keeps its own stale local name.

// 3. `export *` re-exports named exports but NEVER the default:
export * from "./paymentService.js";                  // default NOT included
export { default as PaymentService } from "./paymentService.js";  // must be explicit

// 4. Auto-import in editors works far better with named exports —
//    the editor knows the symbol's canonical name.

// 5. Tree-shaking bundlers can drop unused NAMED exports precisely;
//    a default export that is a big object is all-or-nothing.
```

The practical convention in typed Node backends: **named exports everywhere**, with default exports reserved only for frameworks that demand them (Express routers you `app.use()`, some serverless handlers, some config files).

```ts
// ❌ Common but weak — a default object export
export default {
  findUserById,
  createUser,
  deleteUser,
};
// Importers write `userService.findUserById(...)`; nothing is tree-shakeable,
// and `import userSvc from` vs `import userService from` diverge across the repo.

// ✅ Strong — named exports, and a namespace import if a caller wants grouping
export { findUserById, createUser, deleteUser };
// import * as userService from "./userService.js";   // grouping is the caller's choice
```

### Concept 3 — Type-only imports and exports, and *why* they exist

TypeScript erases types. That means for every import, the compiler must decide: **does this import survive into the emitted JavaScript, or not?**

```ts
// src/routes/userRoutes.ts
import { findUserById } from "../services/userService.js";   // a VALUE — must survive
import { User } from "../types/user.js";                     // a TYPE  — must be erased
```

The classic compiler behaviour is **import elision**: TypeScript looks at what each imported binding actually is, and if *every* binding from a specifier turns out to be type-only, it deletes the entire `import` statement from the output.

```js
// Emitted JS from the file above:
import { findUserById } from "../services/userService.js";
// the User import is GONE — the whole statement was elided
```

Elision is convenient and also the source of three real problems:

```ts
// PROBLEM 1 — Side effects disappear.
import { RequestContext } from "./context.js";   // context.js also registers a global hook
// If RequestContext is only a type, the import vanishes and the hook never runs.

// PROBLEM 2 — Single-file transpilers cannot do it.
// esbuild / swc / Babel / ts-node's fast mode compile ONE FILE AT A TIME.
// They cannot know whether `User` is a type or a value without reading the other file.
// So they must guess — and guessing wrong either keeps a dead import (runtime crash
// on a type-only module) or drops a needed one.

// PROBLEM 3 — Decorator metadata / DI containers need the runtime import kept.
```

`import type` solves all three by making the intent **explicit and local**:

```ts
import type { User, ApiResponse } from "../types/user.js";
// GUARANTEED erased. If you try to use `User` as a value, you get a compile error.

import { type User, findUserById } from "../services/userService.js";
// Inline form (TS 4.5+): mixes erased and retained bindings in one statement.
// Emits: import { findUserById } from "../services/userService.js";
```

Type-only exports mirror it:

```ts
// src/types/index.ts
export type { User, CreateUserDto } from "./user.js";     // whole clause is types
export { type OrderStatus, ORDER_STATUSES } from "./order.js";  // mixed, inline modifier
```

An important guarantee: a `import type` binding **cannot be used in a value position**, and the compiler enforces it:

```ts
import type { UserRepository } from "./userRepository.js";

let repo: UserRepository;                 // ✅ type position
const r = new UserRepository();           // ❌ Error: 'UserRepository' cannot be used as a
                                          //    value because it was imported using 'import type'.
```

That error is a feature. It tells you at the import line whether this dependency is a real runtime edge in your module graph or not.

### Concept 4 — `isolatedModules` and `verbatimModuleSyntax`

These two flags exist because of the single-file-transpilation problem above. Almost every modern toolchain (esbuild, swc, Babel, Vite, Bun, `ts-node --swc`, `tsx`) transpiles files independently, without a full type graph.

**`isolatedModules: true`** tells `tsc` to reject any code that *cannot* be correctly compiled one file at a time. It changes no output; it only adds errors.

```ts
// ── What isolatedModules forbids ────────────────────────────────────────────

// ❌ 1. Re-exporting a type without the `type` modifier.
export { User } from "./types/user.js";
// Error: Re-exporting a type when 'isolatedModules' is enabled requires using 'export type'.
// (A single-file transpiler would emit a runtime re-export of a symbol that doesn't exist.)
// ✅
export type { User } from "./types/user.js";

// ❌ 2. `const enum` across files (its values get inlined via cross-file info).
const enum LogLevel { Debug, Info }
// Error: Cannot access ambient const enums when 'isolatedModules' is enabled.
// ✅ Use a plain enum, or a const object + union:
const LOG_LEVELS = { Debug: 0, Info: 1 } as const;
type LogLevel = (typeof LOG_LEVELS)[keyof typeof LOG_LEVELS];

// ❌ 3. A file with no imports/exports at all in some setups is ambiguous.
// ✅ Add `export {}`.

// ❌ 4. Certain namespace/`export =` patterns that require whole-program knowledge.
```

**`verbatimModuleSyntax: true`** (TS 5.0+, replaces the old `importsNotUsedAsValues` and `preserveValueImports`) goes further and states a much simpler rule:

> Any import or export **without** a `type` modifier is emitted **exactly as written**. Any import or export **with** a `type` modifier is **dropped entirely**. No elision, no analysis, no guessing.

```ts
// With verbatimModuleSyntax: true

import { findUserById } from "./userService.js";
import type { User } from "./types/user.js";
import { type CreateUserDto, validateBody } from "./validation.js";

// Emits EXACTLY:
//   import { findUserById } from "./userService.js";
//   import { validateBody } from "./validation.js";
// The `import type` line is gone; `type CreateUserDto` was stripped from the third.
```

Consequences you feel immediately:

```ts
// ❌ Forgetting `type` on a type-only import is now a runtime bug waiting, so TS errors:
import { User } from "./types/user.js";
let u: User;
// Error: 'User' is a type and must be imported using a type-only import
//        when 'verbatimModuleSyntax' is enabled.

// ✅
import type { User } from "./types/user.js";

// ⚠️ And this becomes an error too, because CJS syntax can't be emitted verbatim
//    into an ESM file:
import express = require("express");
// Error: ESM syntax is not allowed in a CommonJS module when 'verbatimModuleSyntax' is enabled
//        (and vice-versa, depending on the file's detected module kind).
```

The recommended modern baseline for a Node backend:

```jsonc
{
  "compilerOptions": {
    "module": "nodenext",
    "moduleResolution": "nodenext",
    "verbatimModuleSyntax": true,   // implies the strictness of isolatedModules for imports
    "isolatedModules": true,        // still worth setting explicitly
    "esModuleInterop": true
  }
}
```

### Concept 5 — CommonJS vs ESM, and what `esModuleInterop` actually does

The two module systems differ in ways that no amount of syntax sugar fully hides.

| | CommonJS | ESM |
|---|---|---|
| Syntax | `require()` / `module.exports` | `import` / `export` |
| Resolution | synchronous, runtime | asynchronous, statically analysed |
| Bindings | a copy of the value at require time | live bindings (re-read on access) |
| Loading | executes on first `require` | full graph parsed, then executed |
| `__dirname` | available | not available (use `import.meta.url`) |
| Top-level `await` | ❌ | ✅ |
| File extension in specifier | optional | **required** |
| Conditional/lazy load | `require()` anywhere | `await import()` only |

The core interop problem: **CommonJS has no concept of a default export.** `module.exports = X` sets the whole namespace object to `X`. ESM's `import X from "m"` asks for the property named `default`.

```js
// node_modules/express/index.js  (CommonJS)
function createApplication() { /* ... */ }
createApplication.Router = function () { /* ... */ };
module.exports = createApplication;
```

```ts
// WITHOUT esModuleInterop:
import * as express from "express";
const app = express();     // ❌ Error: This expression is not callable.
                           //    Type 'typeof import("express")' has no call signatures.
// You had to write:
import express = require("express");   // TS-only syntax
const app = express();                 // ✅ works, but not portable to ESM

// WITH esModuleInterop: true
import express from "express";
const app = express();                 // ✅ compiles AND runs
```

What the flag does, precisely — two independent things:

```ts
// (1) allowSyntheticDefaultImports (type-level, implied by esModuleInterop):
//     "Pretend a CJS module with `export =` also has a `default` export."
//     This ONLY silences the type error. It changes no emit.

// (2) esModuleInterop (emit-level): when emitting CommonJS, wrap imports in helpers:
//        __importDefault  — turns `module.exports = fn` into `{ default: fn }`
//        __importStar     — builds a proper namespace object with a `default` key
```

Look at the emit to make it concrete:

```ts
// Source
import express from "express";
import * as fs from "node:fs";
const app = express();
fs.readFileSync("x");
```

```js
// Emitted CJS WITHOUT esModuleInterop:
const express = require("express");
const fs = require("node:fs");
const app = express.default();          // ❌ express.default is undefined → TypeError
fs.readFileSync("x");

// Emitted CJS WITH esModuleInterop:
var __importDefault = (this && this.__importDefault) || function (mod) {
  return (mod && mod.__esModule) ? mod : { "default": mod };
};
const express_1 = __importDefault(require("express"));
const fs = __importStar(require("node:fs"));
const app = (0, express_1.default)();   // ✅ works
fs.readFileSync("x");
```

The `__esModule` marker is the whole trick: TypeScript (and Babel, and bundlers) set `exports.__esModule = true` on modules they transpile from ESM, so the helper can tell "a real CJS module" from "a transpiled ESM module" and only synthesize a default for the former.

Another effect people miss — `esModuleInterop` also **restricts namespace imports** to be non-callable, which is correct ESM semantics:

```ts
import * as express from "express";
const app = express();
// ❌ With esModuleInterop: Error — a namespace object is never callable.
//    This is intentional: it stops you writing code that breaks under real ESM.
```

**Set `esModuleInterop: true`.** It is on by default in `tsc --init` and required by `module: nodenext`'s ESM emit. The only reason to leave it off is a legacy codebase full of `import x = require()`.

### Concept 6 — Module resolution: `node10`, `node16`, `nodenext`, `bundler`

`moduleResolution` answers one question: given the specifier string `"../services/userService.js"` or `"express"`, **which file on disk do I type-check against?**

```jsonc
// ── "node10" (formerly "node") — classic CommonJS resolution ────────────────
// For "./userService":
//   ./userService.ts → ./userService.tsx → ./userService.d.ts
//   → ./userService/index.ts → ...
// For "express":
//   node_modules/express/package.json "types"/"typings" field
//   → node_modules/express/index.d.ts
//   → node_modules/@types/express/index.d.ts
// IGNORES: package.json "exports", "imports", conditional exports.
// Extensions in specifiers: NOT expected — `"./userService"` is the norm.
```

```jsonc
// ── "node16" / "nodenext" — models real Node ESM+CJS behaviour ──────────────
// Determines each file's module kind FIRST:
//   *.mts / *.mjs                      → ESM
//   *.cts / *.cjs                      → CJS
//   *.ts  / *.js  → nearest package.json "type" field
//                    "module"    → ESM
//                    absent/"commonjs" → CJS
// Then applies THAT system's rules:
//   In an ESM file: extension is REQUIRED, no directory-index resolution,
//                   package.json "exports" respected with the "import" condition.
//   In a CJS file:  classic resolution, "exports" respected with "require" condition.
// nodenext = node16 but tracks the latest Node semantics as they ship.
```

```jsonc
// ── "bundler" (TS 5.0+) — what esbuild/Vite/webpack/Bun actually do ─────────
// Respects package.json "exports"/"imports" (like node16)
// BUT allows extensionless relative imports and directory-index resolution
// (like node10), because bundlers do.
// Requires "module": "esnext" | "preserve".
// Use ONLY when a bundler — not Node — loads your output.
```

The decision table for a backend:

| Situation | `module` | `moduleResolution` |
|---|---|---|
| Node app, CJS output, legacy | `commonjs` | `node10` |
| Node app, want it correct (CJS or ESM) | `nodenext` | `nodenext` |
| Publishing a library with dual CJS/ESM | `nodenext` | `nodenext` |
| Output goes through esbuild/Vite/webpack | `preserve` or `esnext` | `bundler` |
| Type-check only, Bun/Deno runs the TS | `preserve` | `bundler` |

A concrete failure `node10` cannot see, and `nodenext` catches:

```jsonc
// node_modules/some-sdk/package.json
{
  "name": "some-sdk",
  "exports": {
    ".":          { "types": "./dist/index.d.ts",     "import": "./dist/index.mjs" },
    "./webhooks": { "types": "./dist/webhooks.d.ts",  "import": "./dist/webhooks.mjs" }
  }
}
```

```ts
// With moduleResolution: "node10"
import { verifySignature } from "some-sdk/dist/webhooks";
// ✅ Type-checks (node10 walks the folder and finds the .d.ts)
// ❌ Crashes at runtime under Node ESM: "exports" does not expose /dist/webhooks

// With moduleResolution: "nodenext"
import { verifySignature } from "some-sdk/dist/webhooks";
// ❌ Error: Cannot find module 'some-sdk/dist/webhooks'. ... resolution mode is ESM
//    Correct, and it forces you to write:
import { verifySignature } from "some-sdk/webhooks";   // ✅ matches the exports map
```

### Concept 7 — `.js` extensions in ESM TypeScript imports (the rule that offends everyone)

Under Node ESM, `import "./userService"` is an error — the spec requires a full, resolvable path with extension. And TypeScript **never rewrites your specifier strings**. What you write is what gets emitted.

So in a `"type": "module"` project compiled by `tsc`, you must write the extension of the **output** file, not the input:

```ts
// src/routes/userRoutes.ts   (project has "type": "module")

import { findUserById } from "../services/userService.js";   // ✅ points at userService.ts,
                                                             //    emits ../services/userService.js
import { findUserById } from "../services/userService";      // ❌ ERR_MODULE_NOT_FOUND at runtime
import { findUserById } from "../services/userService.ts";   // ❌ TS error by default
```

It feels wrong to write `.js` next to a `.ts` file. The reasoning is consistent once you accept the rule "TypeScript type-checks TS but emits JS and never touches specifiers": you are writing the specifier the *runtime* will see.

The mapping the resolver applies under `nodenext`:

| You write | TS type-checks against | Node loads |
|---|---|---|
| `"./userService.js"` | `userService.ts`, then `userService.d.ts`, then `userService.js` | `userService.js` |
| `"./userService.mjs"` | `userService.mts` / `.d.mts` | `userService.mjs` |
| `"./userService.cjs"` | `userService.cts` / `.d.cts` | `userService.cjs` |
| `"./userService"` | works only in a CJS-mode file | fails in ESM |

Escape hatches, with their costs:

```jsonc
// (a) allowImportingTsExtensions — write `from "./userService.ts"`
{ "compilerOptions": {
    "allowImportingTsExtensions": true,
    "noEmit": true            // REQUIRED — tsc refuses to emit .ts specifiers
}}
// Only viable when something else (bun, tsx, a bundler) produces the runtime output.

// (b) rewriteRelativeImportExtensions (TS 5.7+) — let tsc rewrite .ts → .js on emit
{ "compilerOptions": {
    "allowImportingTsExtensions": true,
    "rewriteRelativeImportExtensions": true
}}
// Now `from "./userService.ts"` emits `from "./userService.js"`. Relative paths only.

// (c) Stay on CommonJS. Extensions stay optional. Simplest, and still the majority
//     of production Node backends.
```

Related ESM-only adjustments you will hit in the same migration:

```ts
// __dirname / __filename do not exist in ESM:
import { fileURLToPath } from "node:url";
import path from "node:path";
const __filename = fileURLToPath(import.meta.url);
const __dirname  = path.dirname(__filename);
// Node 20.11+ / 21.2+ also give you:  import.meta.dirname

// require() does not exist either — create one if you must:
import { createRequire } from "node:module";
const require = createRequire(import.meta.url);
const legacyConfig = require("./legacy.config.cjs");

// JSON imports need an attribute in ESM:
import pkg from "./package.json" with { type: "json" };
// and "resolveJsonModule": true in tsconfig
```

### Concept 8 — `export =` and `import x = require()`

These are **TypeScript-only** syntaxes that exist to describe CommonJS modules whose entire export *is* a single value — something ESM syntax genuinely cannot express.

```ts
// ── Describing a CJS library: `module.exports = <thing>` ────────────────────
// node_modules/@types/express-session/index.d.ts (shape, simplified)

declare function session(options?: session.SessionOptions): RequestHandler;

declare namespace session {
  interface SessionOptions { secret: string; resave?: boolean }
  interface SessionData { userId?: number; authToken?: string }
}

export = session;
// This says: `module.exports` IS the `session` function, and it also carries
// the types declared in the `session` namespace. No `default`, no named exports.
```

```ts
// ── Consuming it ────────────────────────────────────────────────────────────

// The literal-fidelity way (works with esModuleInterop OFF, CJS output only):
import session = require("express-session");
const middleware = session({ secret: process.env.SESSION_SECRET! });
type Options = session.SessionOptions;          // namespace members accessible

// With esModuleInterop: true you can also write the portable ESM form:
import session from "express-session";
import type { SessionOptions } from "express-session";
const middleware = session({ secret: process.env.SESSION_SECRET! });
```

Authoring `export =` in your own code:

```ts
// src/legacy/metricsClient.ts

class MetricsClient {
  constructor(private readonly namespace: string) {}
  increment(metric: string, value = 1): void { /* ... */ }
}

export = MetricsClient;      // emits: module.exports = MetricsClient;

// ⚠️ Hard constraints:
//   - `export =` cannot be combined with any other export in the same file.
//   - Only legal when the file's module kind is CommonJS.
//   - Under `verbatimModuleSyntax` in an ESM file, it is an error.
//   - `import x = require()` likewise requires a CJS-mode file.
```

Rule of thumb: **you consume `export =` constantly (every `@types/*` package for an older CJS library uses it); you should almost never author it.** In new code, write named exports and let the bundler or `tsc` produce whatever module format you need.

### Concept 9 — Barrel files and what they actually cost

A **barrel** is an `index.ts` that re-exports a directory's public surface.

```ts
// src/services/index.ts  — the barrel
export * from "./userService.js";
export * from "./orderService.js";
export * from "./paymentService.js";
export * from "./emailService.js";
export * from "./searchService.js";
export type * from "./types.js";
```

```ts
// The appeal — one import line instead of five:
import { findUserById, createOrder, chargeCard } from "../services/index.js";
```

The costs, in order of how much they hurt:

```ts
// ── COST 1: The whole barrel executes, even for one symbol ──────────────────
import { findUserById } from "../services/index.js";
// Node loads index.js → which loads userService, orderService, paymentService,
// emailService, searchService → which load the pg driver, the Stripe SDK,
// the SES client, the Elasticsearch client...
// A Lambda that only wanted findUserById just paid 800ms of cold start.
// Bundler tree-shaking helps IF every module is side-effect-free AND
// package.json has "sideEffects": false. Node's ESM loader does NOT tree-shake at all.

// ── COST 2: Circular imports become almost guaranteed ──────────────────────
// userService.ts imports { Order } from "../services/index.js"
//   → index.ts imports userService.ts  → cycle.
// Every intra-directory import through the barrel is a cycle back to yourself.

// ── COST 3: Type-checking and editor perf ──────────────────────────────────
// `export *` forces the compiler to resolve every re-exported module to know
// what the barrel exports. Auto-import suggestions balloon; tsserver slows.

// ── COST 4: Ambiguous re-exports fail silently or confusingly ──────────────
// If userService.ts and orderService.ts BOTH export `validate`,
// `export *` drops it from the barrel entirely (ambiguous star exports are excluded),
// with no error at the barrel — only a "no exported member 'validate'" at the importer.
```

The disciplined middle ground:

```ts
// ✅ Barrels at PACKAGE boundaries only (the public API of a module/library),
//    never for intra-package imports.

// src/services/user/index.ts — a real boundary, explicit list, no `export *`
export { findUserById, createUser, deleteUser } from "./userService.js";
export type { User, CreateUserDto, UpdateUserDto } from "./types.js";
// Note: `export type` for the types means importers pay zero runtime cost for them.

// ✅ Inside src/services/user/*, import siblings DIRECTLY, never via the barrel:
import { hashPassword } from "./password.js";       // ✅
// import { hashPassword } from "./index.js";       // ❌ instant cycle

// ✅ If you must keep a wide barrel, make the heavy leaves lazy:
export async function getSearchService() {
  const { SearchService } = await import("./searchService.js");   // loaded on demand
  return new SearchService();
}
```

### Concept 10 — Circular imports: when they work, when they explode

A cycle is `a.ts` importing `b.ts` which imports `a.ts`. Both module systems allow it; both handle it by giving one side a **partially initialised** module.

```ts
// ── ESM: live bindings + hoisting make FUNCTION cycles work ─────────────────

// src/services/userService.ts
import { getOrdersForUser } from "./orderService.js";

export async function findUserById(userId: number) {
  const user = await database.users.findOne({ userId });
  return { ...user, orders: await getOrdersForUser(userId) };
}

// src/services/orderService.ts
import { findUserById } from "./userService.js";

export async function getOrdersForUser(userId: number) {
  const user = await findUserById(userId);        // ✅ works
  return database.orders.find({ userId: user.userId });
}
// ✅ Function DECLARATIONS are hoisted, and ESM bindings are live,
//    so by the time either function is CALLED, both are initialised.
```

```ts
// ── ESM: TOP-LEVEL use of a cyclic import explodes ─────────────────────────

// src/config/database.ts
import { APP_NAME } from "./app.js";
export const POOL_NAME = `${APP_NAME}-pool`;    // evaluated at module init

// src/config/app.ts
import { POOL_NAME } from "./database.js";
export const APP_NAME = "billing-api";
export const DEFAULT_POOL = POOL_NAME;          // evaluated at module init
// ❌ ReferenceError: Cannot access 'APP_NAME' before initialization
//    (whichever module starts the cycle sees the other's `const` in its TDZ)
```

```ts
// ── CommonJS: cycles fail DIFFERENTLY and more quietly ─────────────────────

// userService.cjs-style
const { getOrdersForUser } = require("./orderService");
// If orderService is mid-initialisation, its module.exports is still {},
// so getOrdersForUser is `undefined` — DESTRUCTURED AND FROZEN as undefined.
// → TypeError: getOrdersForUser is not a function, at call time.

// The CJS workaround (why you see this in older code):
const orderService = require("./orderService");   // keep the namespace object
async function findUserById(userId) {
  return orderService.getOrdersForUser(userId);   // property read is DEFERRED
}
```

Types are immune, which is the single most useful fact here:

```ts
// A cycle that exists ONLY in the type graph is completely free —
// `import type` edges do not exist at runtime at all.

// src/types/user.ts
import type { Order } from "./order.js";
export interface User { userId: number; orders: Order[] }

// src/types/order.ts
import type { User } from "./user.js";
export interface Order { orderId: string; user: User }
// ✅ No runtime cycle. No runtime import. Nothing to explode.
```

How to actually fix value cycles:

```ts
// 1. Extract the shared piece into a third module both depend on:
//    userService → shared/entities  ← orderService     (no cycle)

// 2. Invert the dependency with an interface + injection:
export interface OrderLookup { getOrdersForUser(userId: number): Promise<Order[]> }
export function createUserService(orders: OrderLookup) { /* ... */ }

// 3. Make the edge type-only if that's all it was:
import type { OrderService } from "./orderService.js";

// 4. Defer the runtime edge:
async function findUserById(userId: number) {
  const { getOrdersForUser } = await import("./orderService.js");   // lazy
  return getOrdersForUser(userId);
}

// 5. Detect them in CI:
//    npx madge --circular --extensions ts src/
//    or eslint-plugin-import's `import/no-cycle` rule.
```

---

## Example 1 — basic

```ts
// A small, correctly-moduled slice of a Node backend.
// Project: "type": "module", tsconfig module/moduleResolution = "nodenext",
//          verbatimModuleSyntax = true, esModuleInterop = true.

// ════════════════════════════════════════════════════════════════════════════
// src/types/user.ts — TYPES ONLY. This file emits an empty JS file.
// ════════════════════════════════════════════════════════════════════════════

export interface User {
  userId:       number;
  email:        string;
  displayName:  string;
  passwordHash: string;
  createdAt:    Date;
}

export interface CreateUserDto {
  email:       string;
  displayName: string;
  password:    string;
}

export type ApiResponse<T> =
  | { ok: true;  data: T }
  | { ok: false; error: string; statusCode: number };

// ════════════════════════════════════════════════════════════════════════════
// src/db/connection.ts — a runtime module with a side effect on import
// ════════════════════════════════════════════════════════════════════════════

import { Pool } from "pg";                       // default-less named import from CJS pkg

export const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  max: 10,
});

// Side effect: this runs the moment any module imports this file.
pool.on("error", (err) => {
  console.error("Unexpected pool error", err);
});

export async function query<Row>(sql: string, params: unknown[] = []): Promise<Row[]> {
  const result = await pool.query(sql, params);
  return result.rows as Row[];
}

// ════════════════════════════════════════════════════════════════════════════
// src/services/userService.ts — values in, values out, types imported separately
// ════════════════════════════════════════════════════════════════════════════

import { query } from "../db/connection.js";          // VALUE import — survives emit
import type { CreateUserDto, User } from "../types/user.js";  // TYPE import — erased
import bcrypt from "bcryptjs";                         // CJS default via esModuleInterop

export const MAX_USERS_PER_PAGE = 100;

export async function findUserById(userId: number): Promise<User | null> {
  const rows = await query<User>(
    "SELECT user_id AS \"userId\", email, display_name AS \"displayName\", " +
    "password_hash AS \"passwordHash\", created_at AS \"createdAt\" " +
    "FROM users WHERE user_id = $1",
    [userId],
  );
  return rows[0] ?? null;
}

export async function createUser(requestBody: CreateUserDto): Promise<User> {
  const passwordHash = await bcrypt.hash(requestBody.password, 12);
  const rows = await query<User>(
    "INSERT INTO users (email, display_name, password_hash) VALUES ($1, $2, $3) RETURNING *",
    [requestBody.email, requestBody.displayName, passwordHash],
  );
  return rows[0]!;
}

// Renaming on export — internal name vs public name:
async function softDeleteUserInternal(userId: number): Promise<void> {
  await query("UPDATE users SET deleted_at = NOW() WHERE user_id = $1", [userId]);
}
export { softDeleteUserInternal as deleteUser };

// ════════════════════════════════════════════════════════════════════════════
// src/routes/userRoutes.ts — mixes a default import, named, and inline `type`
// ════════════════════════════════════════════════════════════════════════════

import express from "express";                                   // default (CJS interop)
import type { Request, Response } from "express";                // types from the same pkg
import { createUser, findUserById } from "../services/userService.js";
import type { ApiResponse, User } from "../types/user.js";

const router = express.Router();

router.get("/users/:userId", async (req: Request, res: Response) => {
  const userId = Number(req.params.userId);
  if (!Number.isInteger(userId)) {
    const body: ApiResponse<never> = { ok: false, error: "Invalid userId", statusCode: 400 };
    return res.status(400).json(body);
  }

  const user = await findUserById(userId);
  if (!user) {
    const body: ApiResponse<never> = { ok: false, error: "Not found", statusCode: 404 };
    return res.status(404).json(body);
  }

  const { passwordHash, ...safe } = user;
  const body: ApiResponse<Omit<User, "passwordHash">> = { ok: true, data: safe };
  return res.json(body);
});

router.post("/users", async (req: Request, res: Response) => {
  const user = await createUser(req.body);
  const { passwordHash, ...safe } = user;
  return res.status(201).json({ ok: true, data: safe });
});

export default router;   // default export: Express routers are the classic exception

// ════════════════════════════════════════════════════════════════════════════
// src/server.ts — the composition root
// ════════════════════════════════════════════════════════════════════════════

import "./instrumentation/tracing.js";     // side-effect-only import, MUST be first
import express from "express";
import userRoutes from "./routes/userRoutes.js";   // default import, local name is ours
import { pool } from "./db/connection.js";

const app = express();
app.use(express.json());
app.use("/api", userRoutes);

const server = app.listen(Number(process.env.PORT ?? 3000));

// Top-level await: legal in ESM, illegal in CJS. This alone is a reason to migrate.
process.on("SIGTERM", async () => {
  server.close();
  await pool.end();
});
```

```js
// ── What src/services/userService.ts EMITS (verbatimModuleSyntax: true) ─────
import { query } from "../db/connection.js";
import bcrypt from "bcryptjs";
export const MAX_USERS_PER_PAGE = 100;
export async function findUserById(userId) { /* ... */ }
export async function createUser(requestBody) { /* ... */ }
async function softDeleteUserInternal(userId) { /* ... */ }
export { softDeleteUserInternal as deleteUser };
// The `import type { CreateUserDto, User }` line is GONE — zero runtime cost,
// zero runtime dependency on ../types/user.js, zero chance of a cycle through it.
```

---

## Example 2 — real world backend use case

```ts
// ════════════════════════════════════════════════════════════════════════════
// A monorepo service that has to: consume CJS-only SDKs, publish dual CJS/ESM,
// avoid barrel cold-start cost, keep a plugin registry cycle-free, and
// lazy-load heavy optional dependencies.
// ════════════════════════════════════════════════════════════════════════════

// ── tsconfig.json (the service) ─────────────────────────────────────────────
// {
//   "compilerOptions": {
//     "target": "ES2022",
//     "lib": ["ES2022"],
//     "module": "nodenext",
//     "moduleResolution": "nodenext",
//     "esModuleInterop": true,
//     "verbatimModuleSyntax": true,
//     "isolatedModules": true,
//     "resolveJsonModule": true,
//     "strict": true,
//     "outDir": "dist",
//     "rootDir": "src",
//     "declaration": true,
//     "sourceMap": true,
//     "baseUrl": ".",
//     "paths": { "@billing/*": ["./src/*"] }     // see the trap in "Going deeper"
//   },
//   "include": ["src/**/*"]
// }
//
// ── package.json ────────────────────────────────────────────────────────────
// {
//   "name": "@acme/billing-api",
//   "type": "module",
//   "exports": {
//     ".":          { "types": "./dist/index.d.ts",    "import": "./dist/index.js" },
//     "./webhooks": { "types": "./dist/webhooks/index.d.ts",
//                     "import": "./dist/webhooks/index.js" },
//     "./package.json": "./package.json"
//   },
//   "sideEffects": ["./dist/instrumentation/*.js"],
//   "engines": { "node": ">=20" }
// }

// ════════════════════════════════════════════════════════════════════════════
// PART 1 — Consuming a CJS-only SDK from an ESM file
// ════════════════════════════════════════════════════════════════════════════

// src/payments/stripeClient.ts

// Stripe's types use `export = Stripe` with a merged namespace.
// With esModuleInterop:true we can use the portable default-import form:
import Stripe from "stripe";
import type { Request } from "express";

// The namespace-merged types come through as properties on the imported symbol:
export type StripeEvent      = Stripe.Event;
export type StripeCharge     = Stripe.Charge;
export type StripeWebhookSig = string;

const STRIPE_API_VERSION = "2026-01-01" as const;

export const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, {
  apiVersion: STRIPE_API_VERSION as Stripe.LatestApiVersion,
  maxNetworkRetries: 2,
});

export function verifyWebhook(rawBody: Buffer, signature: StripeWebhookSig): StripeEvent {
  return stripe.webhooks.constructEvent(
    rawBody,
    signature,
    process.env.STRIPE_WEBHOOK_SECRET!,
  );
}

// ⚠️ Some CJS-only packages have no ESM build AND no `default`. For those,
//    createRequire is the reliable escape hatch inside an ESM file:
import { createRequire } from "node:module";
const require = createRequire(import.meta.url);
const legacyTaxTables = require("./vendor/tax-tables-2019.cjs") as Record<string, number>;

// ════════════════════════════════════════════════════════════════════════════
// PART 2 — Type-only module boundaries (breaking runtime cycles by design)
// ════════════════════════════════════════════════════════════════════════════

// src/domain/contracts.ts
// PURE TYPES. Emits an (almost) empty JS file. Every service may import it
// with `import type` and create ZERO runtime edges — so no cycles are possible.

export interface Invoice {
  invoiceId:   string;
  userId:      number;
  amountCents: number;
  currency:    "usd" | "eur" | "gbp";
  status:      "draft" | "open" | "paid" | "void";
  issuedAt:    Date;
}

export interface Subscription {
  subscriptionId: string;
  userId:         number;
  planId:         string;
  invoices:       Invoice[];       // cyclic in TYPE space — completely free
  renewsAt:       Date;
}

export interface InvoiceRepository {
  findById(invoiceId: string): Promise<Invoice | null>;
  findForUser(userId: number): Promise<Invoice[]>;
  markPaid(invoiceId: string, paidAt: Date): Promise<Invoice>;
}

export interface SubscriptionRepository {
  findForUser(userId: number): Promise<Subscription[]>;
  cancel(subscriptionId: string): Promise<void>;
}

export interface AuditLogger {
  record(event: string, userId: number, detail: Record<string, unknown>): Promise<void>;
}

// src/services/invoiceService.ts
// Depends on subscriptions ONLY through the interface — a type-only import.
// The concrete SubscriptionService is injected. No runtime edge, no cycle.

import type {
  AuditLogger,
  Invoice,
  InvoiceRepository,
  SubscriptionRepository,
} from "../domain/contracts.js";

export class InvoiceService {
  constructor(
    private readonly invoices: InvoiceRepository,
    private readonly subscriptions: SubscriptionRepository,
    private readonly audit: AuditLogger,
  ) {}

  async settleInvoice(invoiceId: string, paidAt: Date): Promise<Invoice> {
    const invoice = await this.invoices.markPaid(invoiceId, paidAt);
    await this.audit.record("invoice.settled", invoice.userId, { invoiceId });
    return invoice;
  }

  async cancelAllForUser(userId: number): Promise<void> {
    const subs = await this.subscriptions.findForUser(userId);
    for (const sub of subs) await this.subscriptions.cancel(sub.subscriptionId);
    await this.audit.record("subscriptions.cancelled", userId, { count: subs.length });
  }
}

// src/services/subscriptionService.ts
// The mirror image — also type-only on the other side.

import type {
  AuditLogger,
  InvoiceRepository,
  Subscription,
  SubscriptionRepository,
} from "../domain/contracts.js";

export class SubscriptionService implements SubscriptionRepository {
  constructor(
    private readonly invoices: InvoiceRepository,
    private readonly audit: AuditLogger,
  ) {}

  async findForUser(userId: number): Promise<Subscription[]> { /* ... */ return []; }

  async cancel(subscriptionId: string): Promise<void> { /* ... */ }
}

// src/composition.ts — the ONE module that knows about both concretions.
// All the "cycles" collapse into a single acyclic wiring file.

import { InvoiceService } from "./services/invoiceService.js";
import { SubscriptionService } from "./services/subscriptionService.js";
import { PostgresInvoiceRepository } from "./repositories/invoiceRepository.js";
import { ConsoleAuditLogger } from "./infra/auditLogger.js";

export function buildServices() {
  const audit         = new ConsoleAuditLogger();
  const invoiceRepo   = new PostgresInvoiceRepository();
  const subscriptions = new SubscriptionService(invoiceRepo, audit);
  const invoiceSvc    = new InvoiceService(invoiceRepo, subscriptions, audit);
  return { invoiceSvc, subscriptions, audit };
}

// ════════════════════════════════════════════════════════════════════════════
// PART 3 — A lazy plugin registry: heavy modules loaded only when used
// ════════════════════════════════════════════════════════════════════════════

// src/plugins/registry.ts

import type { Invoice } from "../domain/contracts.js";

export interface PaymentPlugin {
  readonly name: string;
  charge(invoice: Invoice, authToken: string): Promise<{ providerRef: string }>;
}

// The map holds LOADERS, not modules. Nothing heavy is imported at boot.
// `() => import(...)` keeps the specifier statically analysable (bundlers can
// still code-split it) while deferring execution.
const PLUGIN_LOADERS = {
  stripe:   () => import("./stripePlugin.js").then((m) => m.stripePlugin),
  paypal:   () => import("./paypalPlugin.js").then((m) => m.paypalPlugin),
  adyen:    () => import("./adyenPlugin.js").then((m) => m.adyenPlugin),
  sandbox:  () => import("./sandboxPlugin.js").then((m) => m.sandboxPlugin),
} satisfies Record<string, () => Promise<PaymentPlugin>>;

export type PluginName = keyof typeof PLUGIN_LOADERS;

const cache = new Map<PluginName, Promise<PaymentPlugin>>();

export function loadPlugin(name: PluginName): Promise<PaymentPlugin> {
  let pending = cache.get(name);
  if (!pending) {
    pending = PLUGIN_LOADERS[name]();
    cache.set(name, pending);
  }
  return pending;
}

// ✅ Cold start loads only the registry (a few hundred bytes of loaders).
// ✅ A request that charges via Stripe pays for the Stripe SDK once, on first use.
// ✅ `loadPlugin("stripee")` is a compile error — PluginName is derived from the map.

// ════════════════════════════════════════════════════════════════════════════
// PART 4 — A deliberately narrow public barrel
// ════════════════════════════════════════════════════════════════════════════

// src/index.ts
// EXPLICIT list, no `export *`. Types re-exported with `export type` so consumers
// importing only types pull in NO runtime code from this package.

export { InvoiceService } from "./services/invoiceService.js";
export { SubscriptionService } from "./services/subscriptionService.js";
export { buildServices } from "./composition.js";
export { loadPlugin, type PluginName, type PaymentPlugin } from "./plugins/registry.js";

export type {
  AuditLogger,
  Invoice,
  InvoiceRepository,
  Subscription,
  SubscriptionRepository,
} from "./domain/contracts.js";

// src/webhooks/index.ts — a SECOND entry point, mapped in package.json "exports".
// A consumer that only handles webhooks never loads the service layer at all.

export { verifyWebhook, type StripeEvent } from "../payments/stripeClient.js";
export { handleInvoicePaid } from "./invoicePaid.js";

// ════════════════════════════════════════════════════════════════════════════
// PART 5 — What consumers see
// ════════════════════════════════════════════════════════════════════════════

// A consumer package, also ESM:
import { buildServices, loadPlugin } from "@acme/billing-api";
import type { Invoice } from "@acme/billing-api";              // zero runtime cost
import { verifyWebhook } from "@acme/billing-api/webhooks";     // separate entry point

// ❌ import { verifyWebhook } from "@acme/billing-api/dist/webhooks/index.js";
//    Error: Cannot find module — "exports" in package.json does not expose /dist/**.
//    Under moduleResolution "nodenext" this is caught at COMPILE time, not in prod.

const { invoiceSvc } = buildServices();
const gateway = await loadPlugin("stripe");
const invoice: Invoice = await invoiceSvc.settleInvoice("in_9f2", new Date());
await gateway.charge(invoice, process.env.AUTH_TOKEN!);
```

---

## Going deeper

### `paths` does not exist at runtime

`compilerOptions.paths` is a **type-checking-only** alias table. `tsc` resolves `@billing/services` for the checker and then emits that exact string, which Node cannot resolve.

```jsonc
// tsconfig.json
{ "compilerOptions": { "baseUrl": ".", "paths": { "@billing/*": ["./src/*"] } } }
```

```ts
import { InvoiceService } from "@billing/services/invoiceService.js";
// ✅ Type-checks.
// Emits:  import { InvoiceService } from "@billing/services/invoiceService.js";
// ❌ Runtime: ERR_MODULE_NOT_FOUND — Node has never heard of @billing.
```

Your options, all of which require a *second* tool or a runtime feature:

```jsonc
// (a) Node subpath imports — real, standard, no tooling. Keys MUST start with "#".
// package.json
{ "imports": { "#services/*": "./dist/services/*" } }
// tsconfig with moduleResolution nodenext understands these natively:
//   import { InvoiceService } from "#services/invoiceService.js";   ✅ compiles AND runs

// (b) A path-rewriting step: tsc-alias, tsup, esbuild, rollup, or SWC's paths support.
// (c) A runtime resolver: `tsconfig-paths/register` (dev only, CJS only, slow).
// (d) Don't use paths. Relative imports are ugly and always work.
```

For a Node backend that `tsc`-compiles and runs the output directly, **`imports` in package.json is the correct answer** and `paths` is a trap.

### Declaration emit forces you to export your types

With `declaration: true`, a public API that mentions a non-exported type fails to compile:

```ts
// src/services/reportService.ts

interface ReportOptions {          // NOT exported
  from: Date;
  to:   Date;
  includeVoided: boolean;
}

export function buildReport(options: ReportOptions): string { /* ... */ return ""; }
// ❌ Error TS4078: Parameter 'options' of exported function has or is using private
//    name 'ReportOptions'.
//    (Under composite/declaration builds; also surfaces as TS2742 with pnpm symlinks.)

// ✅ Export the type too:
export interface ReportOptions { /* ... */ }
```

The related, nastier version is TS2742 — *"The inferred type of X cannot be named without a reference to ..."* — which happens when an inferred return type points at a type inside `node_modules` that your `.d.ts` cannot name. The fix is almost always to write the return type explicitly:

```ts
import type { Redis } from "ioredis";

// ❌ Inferred type mentions a nested ioredis type your d.ts can't reference
export function createCache(client: Redis) {
  return { get: (k: string) => client.get(k), set: (k: string, v: string) => client.set(k, v) };
}

// ✅ Name it yourself
export interface Cache {
  get(key: string): Promise<string | null>;
  set(key: string, value: string): Promise<unknown>;
}
export function createCache(client: Redis): Cache { /* ... */ }
```

### `import type` vs `import { type X }` — a real difference

They are not always interchangeable. The whole-clause form guarantees the statement is fully erased; the inline form only marks that binding.

```ts
// (a) Whole-clause: the ENTIRE import is erased, including any side effect.
import type { User } from "./userService.js";
// Emits: nothing.

// (b) Inline modifier with at least one value binding: the statement SURVIVES.
import { type User, findUserById } from "./userService.js";
// Emits: import { findUserById } from "./userService.js";

// (c) Inline modifier with ONLY type bindings — statement survives as a bare import
//     under verbatimModuleSyntax, preserving side effects:
import { type User } from "./userService.js";
// Emits: import "./userService.js";     ← side effects PRESERVED. Subtle but important.

// So if a module must be loaded for its side effects but you only need its types,
// form (c) — or an explicit `import "./x.js"` next to `import type` — is correct.
```

### `export *` ambiguity is silent

```ts
// src/validation/userRules.ts
export function validate(requestBody: unknown): string[] { return []; }

// src/validation/orderRules.ts
export function validate(requestBody: unknown): string[] { return []; }

// src/validation/index.ts
export * from "./userRules.js";
export * from "./orderRules.js";
// NO ERROR HERE. Per spec, ambiguous star exports are simply excluded.

// src/routes/checkout.ts
import { validate } from "../validation/index.js";
// ❌ Error: Module '"../validation/index.js"' has no exported member 'validate'.
//    The error appears far from the cause. `export * as userRules from ...` avoids it.
```

Note the asymmetry: an **explicit** export always wins over a star export of the same name, silently shadowing it:

```ts
export * from "./userRules.js";                 // has `validate`
export { validate } from "./orderRules.js";     // explicit — this one wins, no warning
```

### Node's `"type"` field is per-directory, and `.mts` / `.cts` override it

Under `moduleResolution: nodenext`, the compiler reproduces Node's algorithm exactly: it walks up from each file to the **nearest** `package.json` and reads `"type"`.

```
packages/api/package.json          { "type": "module" }
packages/api/src/server.ts         → ESM
packages/api/src/scripts/package.json   { "type": "commonjs" }
packages/api/src/scripts/seed.ts   → CJS   (nearest package.json wins)
packages/api/src/build.mts         → ESM   (extension always wins)
packages/api/src/jest.config.cts   → CJS   (extension always wins)
```

This is the standard trick for config files that a CJS-only tool must `require()` inside an ESM package: name them `.cts`, and they emit `.cjs`.

```ts
// jest.config.cts  (in a "type": "module" package)
import type { Config } from "jest";
const config: Config = { preset: "ts-jest/presets/default-esm" };
export = config;         // ✅ legal here — this file IS CommonJS
```

Mixing modes has a hard rule you cannot escape: **CJS cannot `require()` an ESM module** (before Node 22's `require(esm)` support, and even then only for synchronous, non-TLA modules). ESM can always `import` CJS.

```ts
// In a .cts (CJS) file:
import { InvoiceService } from "./invoiceService.js";  // ← that file is ESM
// ❌ Error TS1479: The current file is a CommonJS module whose imports will produce
//    'require' calls; however, the referenced file is an ECMAScript module.
// ✅ Use a dynamic import — it works from CJS:
const { InvoiceService } = await import("./invoiceService.js");
```

### `resolutionMode` assertions for mixed-mode type resolution

Occasionally you need the types of a package as *the other* module system sees them:

```ts
// Force CJS-flavoured types for one import, in an ESM file (TS 4.7+):
import type { Options } from "some-dual-package" with { "resolution-mode": "require" };
import type { Options as EsmOptions } from "some-dual-package" with { "resolution-mode": "import" };

// Same for the require-style form:
type Legacy = import("some-dual-package", { with: { "resolution-mode": "require" } }).Options;
```

This matters for dual-published packages whose CJS and ESM `.d.ts` files diverge — a real and common source of "the types are different in my test file than in my source file".

### `import.meta` is module-system dependent

```ts
// Only available when the file is ESM AND `module` is es2020+/nodenext/preserve.
const here = import.meta.url;         // "file:///app/dist/server.js"
const dir  = import.meta.dirname;     // Node 20.11+ / 21.2+ only
const file = import.meta.filename;

// ❌ In a CJS-mode file: Error TS1343: The 'import.meta' meta-property is only allowed
//    when the '--module' option is 'es2020', 'es2022', 'esnext', 'system',
//    'node16', 'node18', or 'nodenext'.
```

### Dynamic `import()` in CJS output gets downleveled to `require`

A surprising emit detail: with `module: "commonjs"`, `await import("./x.js")` is compiled into a `Promise.resolve().then(() => require("./x.js"))`. That means it can **no longer load real ESM**.

```ts
// module: "commonjs"
const { chalk } = await import("chalk");   // chalk v5 is ESM-only
// Emits: Promise.resolve().then(() => require("chalk"))
// ❌ Runtime: ERR_REQUIRE_ESM

// Fixes:
//   - "module": "node16"/"nodenext" — tsc preserves real dynamic import() in CJS files
//   - or the escape hatch: eval("import('chalk')") — ugly but it survives transpilation
```

### `moduleDetection` controls when a file counts as a module

```jsonc
{ "compilerOptions": { "moduleDetection": "force" } }
// "auto"   (default): module if it has import/export, OR it's a .mts/.cts,
//                      OR "type": "module" is set and it's a .ts file compiled as ESM
// "legacy" : only import/export counts (the pre-4.7 rule)
// "force"  : EVERY non-declaration file is a module. No accidental globals, ever.
```

`"force"` is a small, high-value setting for backends — it removes the entire class of "my two constants files collided in the global scope" bugs.

### Barrels and `sideEffects`

Tree-shaking a barrel only works if bundlers know the modules are pure:

```jsonc
// package.json
{ "sideEffects": false }
// or, precisely:
{ "sideEffects": ["./dist/instrumentation/*.js", "*.css"] }
```

Getting this wrong in either direction is expensive: `false` on a package whose import registers a global (polyfills, `reflect-metadata`, OpenTelemetry instrumentation) means the bundler deletes the import and the feature silently stops working. And **Node's native ESM loader ignores `sideEffects` entirely** — it never tree-shakes. For serverless cold starts, only *fewer imports* helps.

---

## Common mistakes

### Mistake 1 — Omitting `.js` extensions in an ESM project

```ts
// ❌ Compiles under moduleResolution "node10", crashes under Node ESM
// package.json has "type": "module"
import { findUserById } from "../services/userService";
import { pool }        from "../db/connection";
// ✅ tsc says nothing (with node10 resolution)
// ❌ node dist/server.js
//    Error [ERR_MODULE_NOT_FOUND]: Cannot find module '/app/dist/services/userService'
//    imported from /app/dist/routes/userRoutes.js
//    Did you mean to import "../services/userService.js"?

// ✅ Right — write the extension of the OUTPUT file, and use nodenext resolution
//    so the compiler enforces it for you:
import { findUserById } from "../services/userService.js";
import { pool }        from "../db/connection.js";
```

```jsonc
// ✅ The tsconfig that makes the compiler catch this instead of production:
{ "compilerOptions": { "module": "nodenext", "moduleResolution": "nodenext" } }
// Now the extensionless version is:
// ❌ Error TS2835: Relative import paths need explicit file extensions in ECMAScript
//    imports when '--moduleResolution' is 'node16' or 'nodenext'.
```

### Mistake 2 — Importing a type as a value under `verbatimModuleSyntax` / `isolatedModules`

```ts
// ❌ Wrong — no `type` modifier on a type-only binding
import { User, CreateUserDto } from "../types/user.js";
import { UserRole } from "../types/roles.js";

export function toPublicUser(user: User): Omit<User, "passwordHash"> {
  const { passwordHash, ...rest } = user;
  return rest;
}
// ❌ Error TS1484: 'User' is a type and must be imported using a type-only import
//    when 'verbatimModuleSyntax' is enabled.

// Worse, without the flag but WITH esbuild/swc: the import is emitted verbatim,
// Node loads ../types/user.js, finds no export named `User`, and throws:
//   SyntaxError: The requested module './types/user.js' does not provide an export named 'User'

// ✅ Right — separate, or use the inline modifier
import type { CreateUserDto, User } from "../types/user.js";
import { UserRole } from "../types/roles.js";        // an enum IS a value — keep it

// ✅ Or mixed in one statement when a module exports both:
import { type User, findUserById } from "../services/userService.js";
```

```ts
// ⚠️ The reverse mistake: `import type` on something you need at runtime.
import type { UserRepository } from "./userRepository.js";
const repo = new UserRepository();
// ❌ Error TS1361: 'UserRepository' cannot be used as a value because it was
//    imported using 'import type'.
// ✅ Classes and enums are values — import them normally:
import { UserRepository } from "./userRepository.js";
```

### Mistake 3 — Default-importing a CommonJS module without `esModuleInterop`

```ts
// tsconfig: "esModuleInterop": false, "module": "commonjs"

// ❌ Attempt 1
import express from "express";
// ❌ Error TS1259: Module '"express"' can only be default-imported using
//    the 'esModuleInterop' flag.

// ❌ Attempt 2 — "fix" it with a namespace import
import * as express from "express";
const app = express();
// ❌ Error TS2349: This expression is not callable.
//    Type 'typeof e' has no call signatures.

// ❌ Attempt 3 — "fix" it with allowSyntheticDefaultImports only
// { "allowSyntheticDefaultImports": true, "esModuleInterop": false }
import express from "express";
const app = express();
// ✅ Compiles!  ❌ Emits `express_1.default()` → TypeError: undefined is not a function
//    allowSyntheticDefaultImports is a TYPE-ONLY lie. It changes no emit.

// ✅ Right — turn on the flag that changes BOTH the types and the emit:
// { "esModuleInterop": true }
import express from "express";
const app = express();       // ✅ compiles AND runs

// ✅ Or, if you truly cannot enable it (legacy CJS-only codebase):
import express = require("express");
const app = express();
// Correct, but this syntax is illegal in ESM files and under verbatimModuleSyntax.
```

### Mistake 4 — A barrel file that creates a circular import

```ts
// ❌ Wrong — a service importing a sibling through its own directory's barrel

// src/services/index.ts
export * from "./userService.js";
export * from "./orderService.js";

// src/services/orderService.ts
import { findUserById } from "./index.js";        // ← through the barrel = self-cycle
export async function getOrdersForUser(userId: number) {
  const user = await findUserById(userId);
  return [];
}

// Load order: routes → index.js → userService.js → (fine)
//                              → orderService.js → index.js (already loading, returns
//                                                  a PARTIAL namespace)
// Under ESM with function declarations it may work by luck.
// Add one top-level `const DEFAULT_LIMIT = someImportedValue` and it becomes:
//   ReferenceError: Cannot access 'findUserById' before initialization
// ...intermittently, depending on which route file loads first.

// ✅ Right — import siblings DIRECTLY, never via the barrel
// src/services/orderService.ts
import { findUserById } from "./userService.js";
export async function getOrdersForUser(userId: number) {
  const user = await findUserById(userId);
  return [];
}

// ✅ And make the barrel explicit rather than `export *`, so ambiguity is impossible:
// src/services/index.ts
export { findUserById, createUser } from "./userService.js";
export { getOrdersForUser }          from "./orderService.js";

// ✅ Enforce it in CI:
//    eslint: "import/no-cycle": ["error", { maxDepth: Infinity }]
//    or: npx madge --circular --extensions ts src/
```

### Mistake 5 — Trusting `paths` to work at runtime

```ts
// ❌ Wrong
// tsconfig: { "baseUrl": ".", "paths": { "@/*": ["src/*"] } }
import { InvoiceService } from "@/services/invoiceService.js";
// ✅ tsc: clean.  ❌ node dist/server.js:
//    Error [ERR_MODULE_NOT_FOUND]: Cannot find package '@' imported from /app/dist/server.js

// ✅ Right (option A) — Node subpath imports, understood by BOTH tsc and Node
// package.json
// { "imports": { "#services/*": "./dist/services/*" } }
import { InvoiceService } from "#services/invoiceService.js";

// ✅ Right (option B) — keep `paths` but add a rewrite step to the build
// package.json scripts: { "build": "tsc && tsc-alias" }

// ✅ Right (option C) — relative imports. Boring, always correct.
import { InvoiceService } from "../services/invoiceService.js";
```

### Mistake 6 — Re-exporting a type without `export type`

```ts
// ❌ Wrong — under isolatedModules / any single-file transpiler
// src/types/index.ts
export { User, CreateUserDto } from "./user.js";
// ❌ Error TS1205: Re-exporting a type when 'isolatedModules' is enabled requires
//    using 'export type'.
//
// Without the flag but with esbuild, it emits a real runtime re-export of a
// symbol that does not exist in the compiled user.js:
//   SyntaxError: The requested module './user.js' does not provide an export named 'User'

// ✅ Right
export type { CreateUserDto, User } from "./user.js";

// ✅ Mixed values and types in one statement:
export { type OrderStatus, ORDER_STATUSES, createOrder } from "./order.js";

// ✅ Re-export every type from a module (TS 5.0+):
export type * from "./user.js";
export type * as UserTypes from "./user.js";
```

### Mistake 7 — Assuming a side-effect import survives type-only usage

```ts
// ❌ Wrong
// src/entities/user.ts uses decorators that register metadata at import time
import { User } from "./entities/user.js";       // used ONLY in a type position below

export function serialize(user: User): string { return JSON.stringify(user); }
// Import elision deletes the statement. The decorator registration NEVER RUNS.
// Your ORM reports "No metadata for 'User' was found" at runtime.

// ✅ Right — split the runtime edge from the type edge explicitly
import "./entities/user.js";                     // side effect, guaranteed to run
import type { User } from "./entities/user.js";  // type, guaranteed erased

export function serialize(user: User): string { return JSON.stringify(user); }
```

---

## Practice exercises

### Exercise 1 — easy

Convert this CommonJS module pair into correct, modern TypeScript ESM.

```js
// ── GIVEN: src/services/authService.js (CommonJS) ───────────────────────────
const jwt = require("jsonwebtoken");
const { findUserById } = require("./userService");

const TOKEN_TTL_SECONDS = 3600;

function issueToken(userId, role) {
  return jwt.sign({ userId, role }, process.env.JWT_SECRET, { expiresIn: TOKEN_TTL_SECONDS });
}

async function verifyToken(authToken) {
  const payload = jwt.verify(authToken, process.env.JWT_SECRET);
  return findUserById(payload.userId);
}

module.exports = { issueToken, verifyToken, TOKEN_TTL_SECONDS };
```

```js
// ── GIVEN: src/middleware/requireAuth.js (CommonJS) ─────────────────────────
const { verifyToken } = require("../services/authService");

module.exports = function requireAuth(req, res, next) {
  const authToken = (req.headers.authorization || "").replace("Bearer ", "");
  if (!authToken) return res.status(401).json({ error: "Missing token" });
  verifyToken(authToken)
    .then((user) => { req.currentUser = user; next(); })
    .catch(() => res.status(401).json({ error: "Invalid token" }));
};
```

Your task:

1. Rewrite both files as ESM TypeScript targeting `"type": "module"` with `module`/`moduleResolution` set to `nodenext` and `verbatimModuleSyntax: true`.
2. Define `TokenPayload` and `AuthenticatedUser` interfaces in a **types-only** module `src/types/auth.ts`, and import them with `import type`.
3. Use named exports only — no default exports — and correct `.js` extensions on every relative specifier.
4. Import `jsonwebtoken` (a CommonJS package with `export =` types) correctly, assuming `esModuleInterop: true`.
5. Write the `tsconfig.json` `compilerOptions` block these files require.
6. Then write down, for `authService.ts`, exactly which import statements survive into the emitted JavaScript and which are erased — and why.

```ts
// Write your code here
```

### Exercise 2 — medium

Break a real circular dependency without deleting any feature.

```ts
// ── GIVEN: three modules with a runtime cycle ───────────────────────────────

// src/services/userService.ts
import { getOrderCountForUser } from "./orderService.js";
import { sendWelcomeEmail } from "./notificationService.js";

export interface User { userId: number; email: string; displayName: string }

export const USER_TIER_THRESHOLDS = { gold: 50, silver: 10 };

export async function findUserById(userId: number): Promise<User | null> { return null; }

export async function getUserTier(userId: number): Promise<"gold" | "silver" | "bronze"> {
  const orderCount = await getOrderCountForUser(userId);
  if (orderCount >= USER_TIER_THRESHOLDS.gold) return "gold";
  if (orderCount >= USER_TIER_THRESHOLDS.silver) return "silver";
  return "bronze";
}

export async function registerUser(email: string, displayName: string): Promise<User> {
  const user = { userId: 1, email, displayName };
  await sendWelcomeEmail(user);
  return user;
}

// src/services/orderService.ts
import { findUserById, USER_TIER_THRESHOLDS, type User } from "./userService.js";
import { sendOrderConfirmation } from "./notificationService.js";

export interface Order { orderId: string; userId: number; totalCents: number }

// ⚠️ TOP-LEVEL use of an imported const from a cyclic module:
export const VIP_ORDER_THRESHOLD = USER_TIER_THRESHOLDS.gold * 2;

export async function getOrderCountForUser(userId: number): Promise<number> { return 0; }

export async function placeOrder(userId: number, totalCents: number): Promise<Order> {
  const user = await findUserById(userId);
  if (!user) throw new Error("No such user");
  const order = { orderId: "ord_1", userId, totalCents };
  await sendOrderConfirmation(user, order);
  return order;
}

// src/services/notificationService.ts
import type { User } from "./userService.js";
import type { Order } from "./orderService.js";
import { getUserTier } from "./userService.js";

export async function sendWelcomeEmail(user: User): Promise<void> {
  const tier = await getUserTier(user.userId);
  console.log(`Welcome ${user.displayName} (${tier})`);
}

export async function sendOrderConfirmation(user: User, order: Order): Promise<void> {}
```

Your task:

1. Draw (as a comment) the **runtime** dependency graph, distinguishing value edges from type-only edges. Identify every cycle.
2. Explain precisely why `VIP_ORDER_THRESHOLD` can throw `ReferenceError: Cannot access 'USER_TIER_THRESHOLDS' before initialization`, and which entry point triggers it.
3. Restructure the code so **no runtime cycle remains**, using: a shared types-only module, an extracted constants module, and dependency inversion (interfaces + a composition root) for `notificationService`.
4. Keep every public function's signature identical — callers must not change.
5. Add a `src/services/index.ts` barrel that is explicit (no `export *`), re-exports types with `export type`, and does not itself create a cycle.
6. Add the ESLint rule and the `madge` command you would put in CI to prevent regressions.

```ts
// Write your code here
```

### Exercise 3 — hard

Build a dual-published (CJS + ESM) TypeScript package with multiple entry points, lazy-loaded heavy dependencies, and a public API surface that is enforced by the type system.

```ts
// ── SPECIFICATION ──────────────────────────────────────────────────────────
//
// Package: @acme/webhook-kit
// Purpose: verify, parse, and dispatch provider webhooks in a Node backend.
//
// PART A — Package layout & configuration
//   1. Write package.json with:
//      - "type": "module"
//      - dual "exports" for BOTH import and require, for three entry points:
//          "."            -> the core dispatcher
//          "./providers"  -> provider adapters
//          "./testing"    -> in-memory fakes (must NOT be loaded by "." )
//      - correct "types" conditions FIRST in each condition object
//      - a "sideEffects" field that is accurate for this package
//   2. Write TWO tsconfig files:
//      - tsconfig.esm.json  -> module/moduleResolution nodenext, outDir dist/esm
//      - tsconfig.cjs.json  -> module/moduleResolution node16 (CJS), outDir dist/cjs
//      Explain what must go in dist/cjs/package.json and dist/esm/package.json
//      for Node to interpret each tree correctly.
//   3. Both configs must set: verbatimModuleSyntax, isolatedModules,
//      declaration, declarationMap, strict.
//      Note the ONE of these that conflicts with dual emit, and how you resolve it.
//
// PART B — Core types (a types-only module, zero runtime footprint)
//   Define in src/types.ts:
//     - WebhookProvider  = "stripe" | "github" | "shopify" | "twilio"
//     - RawWebhookRequest { provider; rawBody: Buffer; headers: Record<string,string>;
//                           receivedAt: Date }
//     - VerifiedWebhook<TPayload> { provider; eventId: string; eventType: string;
//                                   payload: TPayload; receivedAt: Date }
//     - ProviderAdapter<TPayload> { readonly provider: WebhookProvider;
//                                   verify(req: RawWebhookRequest, secret: string):
//                                     Promise<VerifiedWebhook<TPayload>>; }
//     - ApiResponse<T> = { ok: true; data: T } | { ok: false; error: string;
//                                                  statusCode: number }
//     - A ProviderPayloadMap interface mapping each WebhookProvider to its payload type.
//
// PART C — Lazy provider registry
//   In src/providers/registry.ts:
//     - A const map from WebhookProvider to `() => Promise<ProviderAdapter<...>>`
//       using dynamic import(), so the Stripe/Shopify SDKs are NEVER loaded at boot.
//     - `satisfies` must enforce that every WebhookProvider key is present.
//     - A memoising `loadAdapter<P extends WebhookProvider>(provider: P)` whose return
//       type is `Promise<ProviderAdapter<ProviderPayloadMap[P]>>` — correctly narrowed
//       per provider, NOT a union.
//     - Adding a new provider to WebhookProvider must be a COMPILE ERROR until
//       its loader is added.
//
// PART D — Dispatcher with no runtime cycles
//   In src/dispatcher.ts:
//     - `class WebhookDispatcher` with
//         on<P extends WebhookProvider>(provider: P,
//            handler: (hook: VerifiedWebhook<ProviderPayloadMap[P]>) => Promise<void>): void
//         dispatch(req: RawWebhookRequest, secret: string): Promise<ApiResponse<{ eventId: string }>>
//     - It must depend on the registry ONLY through an injected interface, so
//       src/testing/* can substitute fakes without importing src/providers/*.
//     - Every import of a type must use `import type`; every relative specifier
//       must carry a `.js` extension.
//
// PART E — Consuming a CJS-only dependency
//   One provider adapter must consume a package that only ships
//   `export = verifier` CommonJS types. Show BOTH ways to import it:
//     (a) the esModuleInterop default-import form (preferred), and
//     (b) the `import x = require()` form, and explain why (b) is illegal in this
//         package and what you would rename the file to if you needed it.
//
// PART F — Prove the boundaries
//   Write a src/__checks__/boundaries.ts containing commented-out lines that
//   MUST be compile errors, each with the expected error code/message:
//     - importing "@acme/webhook-kit/dist/esm/dispatcher.js" from outside
//     - `import { VerifiedWebhook } from "../types.js"` without the `type` modifier
//     - an extensionless relative import
//     - `loadAdapter("stripes")`
//     - using an `import type`-ed class as a constructor
//     - handling "stripe" with a handler typed for the shopify payload
//
// PART G — Cold start budget
//   Write a short comment analysing, for a Lambda that ONLY verifies GitHub
//   webhooks, exactly which modules load at cold start under your design, versus
//   under a naive `export *` barrel in src/index.ts. Quantify the difference in
//   terms of which SDKs are and are not required.
```

```ts
// Write your code here
```

---

## Quick reference cheat sheet

```ts
// ══ EXPORT ══════════════════════════════════════════════════════════════════
export const MAX_PAGE_SIZE = 100;                  // named value
export interface User {}                            // named type
export { findUserById, createUser };                // export list
export { findUserById as getUser };                 // renamed
export type { User, CreateUserDto };                // type-only clause
export { type User, findUserById };                 // inline type modifier
export default router;                              // default (max one per module)
export * from "./userService.js";                   // star re-export (NOT the default)
export * as users from "./userService.js";          // namespaced re-export
export type * from "./types.js";                    // type-only star (TS 5.0+)
export { default as Repo } from "./repo.js";        // re-export a default, named
export = LegacyLogger;                              // CJS: module.exports = X (CJS files only)
export {};                                          // marker: "this file is a module"

// ══ IMPORT ══════════════════════════════════════════════════════════════════
import { findUserById } from "./userService.js";    // named
import { findUserById as getUser } from "./x.js";   // renamed
import express from "express";                      // default
import * as fs from "node:fs";                      // namespace
import router, { type Config } from "./r.js";       // default + named + type
import type { User } from "./types.js";             // type-only clause  → fully erased
import { type User, findUser } from "./s.js";       // inline           → statement survives
import { type User } from "./s.js";                 // → emits bare `import "./s.js"`
import type express from "express";                 // type-only default
import "./instrumentation/tracing.js";              // side effect only
import legacy = require("./legacy");                // CJS import (CJS files only)
const mod = await import("./heavy.js");             // dynamic (both systems)
type T = import("./types.js").User;                 // type position import
```

| tsconfig option | What it controls |
|---|---|
| `module` | What module syntax is **emitted** (`commonjs`, `esnext`, `node16`, `nodenext`, `preserve`) |
| `moduleResolution` | How specifiers are **resolved** to files (`node10`, `node16`, `nodenext`, `bundler`) |
| `esModuleInterop` | Emits `__importDefault`/`__importStar` helpers so `import x from` works on CJS |
| `allowSyntheticDefaultImports` | **Type-only** version of the above — silences errors, changes no emit |
| `isolatedModules` | Errors on anything a single-file transpiler cannot compile correctly |
| `verbatimModuleSyntax` | Imports/exports emitted exactly as written; `type` modifier = dropped |
| `moduleDetection` | `auto` / `legacy` / `force` — when a file counts as a module |
| `resolveJsonModule` | Allow `import config from "./config.json"` |
| `allowImportingTsExtensions` | Permit `from "./x.ts"` (requires `noEmit` unless rewriting) |
| `rewriteRelativeImportExtensions` | TS 5.7+: emit `.ts` specifiers as `.js` |
| `paths` / `baseUrl` | **Type-check-time only** aliases — need a rewriter or `imports` to run |
| `declaration` | Emit `.d.ts` — forces you to export every publicly referenced type |

| Symptom | Cause | Fix |
|---|---|---|
| `ERR_MODULE_NOT_FOUND` on a relative path | Missing `.js` extension in ESM | Add `.js`; set `moduleResolution: nodenext` |
| `TypeError: express is not a function` | Default-imported CJS without interop | `esModuleInterop: true` |
| `TS1259: can only be default-imported using esModuleInterop` | Same, at compile time | `esModuleInterop: true` |
| `TS1484: 'X' is a type and must be imported using a type-only import` | `verbatimModuleSyntax` on | `import type { X }` |
| `TS1205: Re-exporting a type requires 'export type'` | `isolatedModules` on | `export type { X } from` |
| `does not provide an export named 'User'` | Type emitted as a runtime import | `import type` |
| `Cannot access 'X' before initialization` | Circular import, top-level value use | Extract shared module / invert dependency |
| `X is not a function` (only in prod) | CJS circular import, destructured too early | Keep the namespace, defer the property read |
| `ERR_REQUIRE_ESM` | CJS trying to `require()` an ESM package | Dynamic `import()`, or go ESM |
| `TS1479: current file is a CommonJS module` | CJS file importing an ESM file | `await import()`, or make the file `.mts` |
| `TS2742: inferred type cannot be named` | `declaration` + unnameable inferred type | Write the return type explicitly |
| `Cannot find package '@'` | `paths` alias at runtime | package.json `imports`, or `tsc-alias` |
| `TS1343: 'import.meta' only allowed when module is...` | `import.meta` in a CJS-mode file | Set `module: nodenext`, or use `__dirname` |
| Slow cold start | Barrel file pulling the whole graph | Explicit narrow barrels + dynamic `import()` |

```jsonc
// ══ THE DEFAULT tsconfig FOR A NEW NODE BACKEND ═════════════════════════════
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["ES2022"],
    "module": "nodenext",
    "moduleResolution": "nodenext",
    "esModuleInterop": true,
    "verbatimModuleSyntax": true,
    "isolatedModules": true,
    "moduleDetection": "force",
    "resolveJsonModule": true,
    "strict": true,
    "declaration": true,
    "sourceMap": true,
    "outDir": "dist",
    "rootDir": "src",
    "skipLibCheck": true
  },
  "include": ["src/**/*"]
}
// + package.json: { "type": "module", "imports": { "#*": "./dist/*" } }
```

---

## Connected topics

- **02 — Setting up TypeScript** — where `module`, `moduleResolution`, and the ESM/CJS decision first appear in a project.
- **03 — tsconfig in depth** — the full option reference behind `esModuleInterop`, `isolatedModules`, `verbatimModuleSyntax`, and `paths`.
- **04 — TypeScript Node.js project setup** — the `package.json` `"type"`, `"exports"`, and `"imports"` fields that drive `nodenext` resolution.
- **49 — Declaration files** — `.d.ts` shapes, `export =`, `declare module`, `declare global`, and why a `.d.ts` with an import stops declaring globals.
- **32 — Utility types** — `Omit`, `Pick`, and friends used above to derive the public shapes a barrel re-exports.
- **47 — keyof and typeof operators** — `keyof typeof PLUGIN_LOADERS` is how the lazy registry derives its provider union from the loader map.
- **67 — Repository pattern** — dependency inversion through interfaces is the structural fix for circular imports shown in Concept 10.
- **52 — Typing Express routes** — the default-export router convention and `import type { Request, Response } from "express"`.
- **69 — Enum vs union types** — `const enum` is banned by `isolatedModules`; the const-object-plus-union pattern is the replacement.
