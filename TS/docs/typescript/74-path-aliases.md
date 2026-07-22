# 74 — Path aliases in TypeScript

## What is this?

A **path alias** is a made-up module specifier — `@/services/orderService` instead of `../../../services/orderService` — that the TypeScript compiler knows how to resolve to a real file on disk.

You declare the mapping in `tsconfig.json`:

```jsonc
{
  "compilerOptions": {
    "baseUrl": ".",                       // the folder every "paths" pattern is measured from
    "paths": {
      "@/*": ["src/*"]                    // "@/services/x" → "<baseUrl>/src/services/x"
    }
  }
}
```

And then you write:

```ts
import { createOrder } from "@/services/orderService";   // instead of "../../../services/orderService"
```

Here is the single most important fact on this page, and the reason most people's first encounter with path aliases ends in a production incident:

```
  src/routes/orderRoutes.ts
    import { createOrder } from "@/services/orderService"
              │
              │  tsc --noEmit      →  ✅ resolves, type-checks, autocompletes
              │  tsc (emit)        →  ⚠️  writes the specifier out UNCHANGED
              ▼
  dist/routes/orderRoutes.js
    require("@/services/orderService")
              │
              │  node dist/server.js
              ▼
  Error: Cannot find module '@/services/orderService'   ❌ MODULE_NOT_FOUND
```

**`paths` is a compile-time-only feature.** It teaches the type checker where a specifier points. It does **not** rewrite the specifier in the emitted JavaScript, and the TypeScript team has repeatedly and deliberately declined to make it do so (see `microsoft/TypeScript#10866`, open since 2016). So a path alias is only half a solution — the other half has to come from your runtime, your bundler, or Node itself.

This document covers both halves: how `baseUrl`/`paths` work, and every practical way to make the runtime agree with the compiler.

---

## Why does it matter?

Three reasons, in increasing order of importance.

**1. Deep relative imports are unreadable and unrefactorable.**

```ts
// src/modules/billing/handlers/webhooks/stripeWebhookHandler.ts
import { logger }          from "../../../../shared/logging/logger";
import { db }              from "../../../../infra/db/pool";
import { OrderRepository } from "../../../orders/repositories/orderRepository";
import { AppConfig }       from "../../../../config/appConfig";
```

Count the dots. Now move that file one folder deeper and count them again. Every `../` is a hardcoded assertion about where the *importing* file lives, which means the import breaks for reasons that have nothing to do with the thing being imported. Reviewers cannot tell at a glance whether `../../../../shared/logging/logger` is the same module as `../../../shared/logging/logger` in the file below it.

**2. Aliases give you an architectural vocabulary.**

```ts
import { logger }          from "@shared/logging/logger";
import { db }              from "@infra/db/pool";
import { OrderRepository } from "@orders/repositories/orderRepository";
import { AppConfig }       from "@config/appConfig";
```

Now the import list *documents the layering*. And once your imports are labelled by layer, you can enforce the layering — `eslint-plugin-boundaries` or `import/no-restricted-paths` can say "nothing under `@infra/*` may import `@routes/*`" in one rule instead of a regex over `../` soup.

**3. Aliases move a module without touching its consumers.**

`@shared/logging/logger` is a stable name. Relocate the file from `src/shared/logging/logger.ts` to `src/platform/observability/logger.ts` and you change **one line of tsconfig**, not forty import statements. That is the actual payoff, and it is worth the runtime setup cost.

The counterweight, and you should hold it firmly: **aliases are a runtime liability.** Every alias is a promise the compiler cannot keep on its own. A team that adds `paths` without deciding on a runtime story ships a `dist/` that crashes on boot. This is the single most common way TypeScript newcomers break a deploy.

---

## The JavaScript way vs the TypeScript way

```js
// ── Plain JavaScript / Node — one resolution algorithm, no ambiguity ─────────

// src/modules/billing/handlers/webhooks/stripeWebhookHandler.js
const { logger } = require("../../../../shared/logging/logger");
const { db }     = require("../../../../infra/db/pool");

// Node's resolver: take the specifier, if it starts with "./" or "../" or "/",
// resolve it against the current file's directory. Otherwise walk up looking
// for node_modules/<specifier>. That's it. That's the whole algorithm.
//
// ✅ What you write is what runs. No build step, no translation layer.
// ❌ It is genuinely painful. So the JS ecosystem grew a pile of workarounds:
//      - module-alias                  (patches Module._resolveFilename at runtime)
//      - a symlink: node_modules/@app → ../src   (works, but breaks on npm ci)
//      - "dependencies": { "@app": "file:./src" } (installs a copy — awful)
//      - webpack `resolve.alias`       (only if you bundle)
//      - just... living with ../../../ (most common)
```

Now the TypeScript version, and the trap:

```ts
// ── TypeScript with paths — the compiler is happy, the runtime is not ────────

// tsconfig.json
// { "compilerOptions": { "baseUrl": ".", "paths": { "@/*": ["src/*"] } } }

// src/modules/billing/handlers/webhooks/stripeWebhookHandler.ts
import { logger } from "@/shared/logging/logger";     // ✅ resolves in the editor
import { db }     from "@/infra/db/pool";             // ✅ Cmd-click works
                                                       // ✅ autocomplete works
                                                       // ✅ `tsc --noEmit` passes
// $ npx tsc && node dist/server.js
//
// node:internal/modules/cjs/loader:1143
//   throw err;
//   ^
// Error: Cannot find module '@/shared/logging/logger'
// Require stack:
// - /app/dist/modules/billing/handlers/webhooks/stripeWebhookHandler.js
//     at Module._resolveFilename (node:internal/modules/cjs/loader:1140:15)
//   code: 'MODULE_NOT_FOUND'
//
// The emitted JS is byte-for-byte:
//     const logger_1 = require("@/shared/logging/logger");
// tsc verified the path and then wrote it out verbatim. This is by design.
```

```ts
// ── The TypeScript way, done properly — compile-time AND runtime agree ──────

// Option A: don't use tsconfig paths at all. Use Node's own subpath imports.
// package.json:  { "imports": { "#shared/*": "./dist/shared/*" } }
import { logger } from "#shared/logging/logger.js";
// ✅ tsc resolves it (it reads package.json "imports" natively since TS 4.7)
// ✅ node resolves it (built-in since Node 12.19 / 14.6, stable since 16)
// ✅ jest, vitest, esbuild, webpack, rollup, bun, deno all resolve it
// ✅ ZERO runtime dependencies, zero flags, zero post-build steps
// ❌ specifiers must start with "#" — you cannot spell it "@/..."

// Option B: keep tsconfig paths, and add a runtime resolver:
//   dev:    tsx  /  ts-node -r tsconfig-paths/register
//   build:  tsc && tsc-alias        (rewrites the specifiers in dist/)
//   or:     bundle with esbuild/webpack/rollup (rewriting is inherent)
//   or:     node -r tsconfig-paths/register dist/server.js   (in production!)
```

The revelation: **`paths` is not a module system feature — it is a type-checker hint.** In JavaScript there is exactly one resolver and it runs at runtime. In TypeScript there are now *two* resolvers — the compiler's and the runtime's — and `paths` only configures the first one. Every problem in this document is a consequence of that one asymmetry.

---

## Syntax

```jsonc
// ── tsconfig.json — the full surface area of path aliasing ───────────────────
{
  "compilerOptions": {
    // baseUrl: the directory that non-relative specifiers are resolved against,
    // AND the directory that every entry in "paths" is measured from.
    // Relative to the tsconfig.json that declares it.
    "baseUrl": ".",

    // paths: pattern → list of candidate locations. Checked in array order;
    // the first one that exists on disk wins.
    "paths": {
      // 1. Wildcard mapping. Exactly one "*" allowed on each side.
      //    "@/services/orderService" → "<baseUrl>/src/services/orderService"
      "@/*": ["src/*"],

      // 2. Multiple aliases, one per architectural layer.
      "@config/*":  ["src/config/*"],
      "@infra/*":   ["src/infrastructure/*"],
      "@domain/*":  ["src/domain/*"],
      "@shared/*":  ["src/shared/*"],

      // 3. Exact (non-wildcard) mapping — aliases a single module.
      "@config": ["src/config/index.ts"],

      // 4. Fallback list — tries src/, then generated/, then node_modules copy.
      "@generated/*": ["src/generated/*", "build/generated/*"],

      // 5. Redirect a package to a local fork or to source instead of dist.
      "@acme/billing-client": ["../../packages/billing-client/src/index.ts"],

      // 6. Stub out a module entirely (types only) — e.g. for a browser build.
      "node:fs": ["src/types/empty-module.d.ts"]
    },

    // Related but DIFFERENT knobs, often confused with paths:
    "rootDirs": ["src", "generated"],   // treat multiple roots as one virtual dir
                                        // (relative imports can cross between them)
    "typeRoots": ["./node_modules/@types", "./src/types"],  // where @types packages live
    "moduleResolution": "bundler"       // "node16"/"nodenext"/"bundler"/"node10"
                                        // paths works with all of them, but the
                                        // rules AROUND it differ — see Concept 6
  }
}
```

```jsonc
// ── package.json — Node's native alternative: "imports" subpath imports ─────
{
  "name": "orders-api",
  "type": "module",
  "imports": {
    // Keys MUST start with "#". That prefix is what tells Node "this is an
    // internal specifier, don't go looking in node_modules".
    "#shared/*":  "./dist/shared/*",
    "#domain/*":  "./dist/domain/*",
    "#config":    "./dist/config/index.js",

    // Conditional resolution — different target per environment:
    "#db": {
      "development": "./dist/infra/db/pgPool.js",
      "test":        "./dist/infra/db/inMemoryPool.js",
      "default":     "./dist/infra/db/pgPool.js"
    }
  }
}
```

```bash
# ── Runtime resolvers — the CLI surface ──────────────────────────────────────

# 1. tsx — transpile-and-run, honours tsconfig paths automatically
npx tsx src/server.ts
npx tsx watch src/server.ts

# 2. ts-node + tsconfig-paths — the classic CommonJS dev loop
npx ts-node -r tsconfig-paths/register src/server.ts

# 3. tsconfig-paths against COMPILED output (needs a second tsconfig — see Concept 8)
node -r tsconfig-paths/register dist/server.js

# 4. tsc-alias — rewrite the specifiers in dist/ after compiling (recommended)
npx tsc && npx tsc-alias

# 5. A bundler — rewriting is inherent to bundling
npx esbuild src/server.ts --bundle --platform=node --outfile=dist/server.js
```

---

## How it works — concept by concept

### Concept 1 — `baseUrl`: the anchor everything is measured from

`baseUrl` does two separate jobs, and conflating them causes a lot of confusion.

**Job 1: it is the root for every `paths` entry.**

```jsonc
// tsconfig.json at <repo>/tsconfig.json
{ "compilerOptions": { "baseUrl": ".",     "paths": { "@/*": ["src/*"] } } }
//                              ^^^ = <repo>          → <repo>/src/*

{ "compilerOptions": { "baseUrl": "./src", "paths": { "@/*": ["*"] } } }
//                              ^^^^^^^ = <repo>/src  → <repo>/src/*   (same result)
```

Both produce identical resolution. The second form is common but I'd avoid it, because of Job 2.

**Job 2 (the dangerous one): with `baseUrl` set, bare specifiers resolve against it.**

```ts
// With "baseUrl": "./src" and NO paths entry at all:
import { orderService } from "services/orderService";
//                            ^^^^^^^^^^^^^^^^^^^^^ resolves to src/services/orderService.ts
//
// ⚠️ This looks exactly like a node_modules package import. Six months later
//    someone runs `npm i services` for an unrelated reason and resolution
//    silently changes meaning. Worse, at RUNTIME Node WILL look in node_modules.
```

This implicit behaviour is the reason `baseUrl` has fallen out of favour:

```jsonc
// ✅ Modern recommendation: paths WITHOUT baseUrl.
//    Supported since TypeScript 4.1 — paths entries are then resolved
//    relative to the tsconfig.json file itself.
{
  "compilerOptions": {
    "paths": { "@/*": ["./src/*"] }   // note the explicit "./"
  }
}
// You get the aliases, and you do NOT get the "any bare specifier might be a
// local folder" ambiguity. Prefer this in new projects.
```

Note also: `baseUrl` is **not permitted** at all when `moduleResolution` is `"bundler"` in some tooling configs and is discouraged everywhere; and `paths` without `baseUrl` is what `tsc --init` scaffolds in recent versions.

### Concept 2 — `paths`: pattern matching rules

The matching algorithm is small enough to state completely:

```jsonc
"paths": {
  "@shared/*":        ["src/shared/*"],
  "@shared/logger":   ["src/platform/logger.ts"],
  "@generated/*":     ["src/generated/*", "build/generated/*"]
}
```

```ts
// RULE 1 — at most ONE "*" per pattern, on each side.
//   "@a/*/b/*": [...]   ❌ error: Pattern '@a/*/b/*' can have at most one '*'

// RULE 2 — the LONGEST matching prefix wins, not the first declared.
import { logger } from "@shared/logger";
//   Candidates: "@shared/*" (prefix "@shared/", 8 chars) and
//               "@shared/logger" (exact, 14 chars)
//   → the exact match wins → src/platform/logger.ts

// RULE 3 — whatever "*" captured is substituted into the target.
import { AppError } from "@shared/errors/AppError";
//   "*" captured "errors/AppError" → src/shared/errors/AppError
//   then normal extension resolution: .ts, .tsx, .d.ts, /index.ts, ...

// RULE 4 — the array is tried in order; first existing file wins.
import { OrderRow } from "@generated/db-types";
//   1. src/generated/db-types.ts        — exists? use it.
//   2. build/generated/db-types.ts      — else this.
//   3. neither → error TS2307: Cannot find module

// RULE 5 — a pattern with NO "*" is an exact, whole-specifier match only.
//   "@config": ["src/config/index.ts"]
//   "@config"          ✅ matches
//   "@config/database" ❌ does NOT match (add "@config/*" separately)
```

### Concept 3 — what `tsc` actually emits (the whole problem, in one diff)

This is worth seeing literally, because reading it once prevents the mistake permanently.

```ts
// ── src/routes/orderRoutes.ts (input) ───────────────────────────────────────
import { Router } from "express";
import { createOrder } from "@/services/orderService";
import { requireAuth } from "@/middleware/requireAuth";
import { logger } from "./logger";                       // ← relative, for contrast

export const orderRoutes = Router();
orderRoutes.post("/orders", requireAuth, async (req, res) => {
  const order = await createOrder(req.body);
  logger.info({ orderId: order.orderId }, "order created");
  res.status(201).json({ data: order });
});
```

```js
// ── dist/routes/orderRoutes.js (tsc output, module: commonjs) ───────────────
"use strict";
Object.defineProperty(exports, "__esModule", { value: true });
exports.orderRoutes = void 0;
const express_1 = require("express");
const orderService_1 = require("@/services/orderService");   // ❌ UNCHANGED
const requireAuth_1 = require("@/middleware/requireAuth");   // ❌ UNCHANGED
const logger_1 = require("./logger");                        // ✅ relative, still valid
// ...
```

```js
// ── dist/routes/orderRoutes.js (tsc output, module: esnext/nodenext) ────────
import { Router } from "express";
import { createOrder } from "@/services/orderService";       // ❌ UNCHANGED
import { requireAuth } from "@/middleware/requireAuth";      // ❌ UNCHANGED
import { logger } from "./logger.js";
```

Same result for ESM and CJS. The compiler **verified** those specifiers — it opened `src/services/orderService.ts`, read its exports, type-checked the call — and then wrote the specifier out untouched.

Why? Because tsc does not know what your runtime is. Rewriting `@/services/orderService` into a correct relative path requires knowing where the output file will sit relative to the output of the target, which depends on `outDir`, `rootDir`, whether you're bundling, whether files get moved by a Docker `COPY`, and so on. The TypeScript team's position is that this is the build tool's job, not the type checker's. You can dislike that position, but you have to design around it.

```
Mental model:

   paths  ─── configures ───▶  the TYPE CHECKER's resolver     (design time)
                              (and nothing else)

   ??????  ─── must configure ───▶  the RUNTIME's resolver      (execution time)
                                    ^^^^ this is your job
```

### Concept 4 — Runtime option A: `tsx` (and `ts-node` + `tsconfig-paths`) for development

For the dev loop, the easiest fix is a runtime that reads your `tsconfig.json`.

```jsonc
// package.json
{
  "scripts": {
    "dev": "tsx watch src/server.ts"
  }
}
```

`tsx` reads `compilerOptions.paths` and installs a Node loader hook that resolves them. Nothing else to configure.

```bash
npx tsx src/server.ts        # aliases resolve ✅
```

The `ts-node` equivalent needs an explicit helper, because `ts-node` deliberately doesn't touch resolution:

```jsonc
// package.json
{
  "scripts": {
    "dev": "ts-node -r tsconfig-paths/register src/server.ts"
  },
  "devDependencies": {
    "ts-node": "^10.9.2",
    "tsconfig-paths": "^4.2.0"
  }
}
```

```jsonc
// Or bake it into tsconfig so plain `ts-node src/server.ts` works:
{
  "ts-node": {
    "require": ["tsconfig-paths/register"],
    "transpileOnly": true
  }
}
```

What `tsconfig-paths/register` does, mechanically:

```js
// Simplified, but this really is the shape of it:
const Module = require("module");
const originalResolve = Module._resolveFilename;

Module._resolveFilename = function (request, ...rest) {
  const matched = matchPathAgainstTsconfigPaths(request);   // "@/services/x" → "/abs/src/services/x"
  return originalResolve.call(this, matched ?? request, ...rest);
};
// ⚠️ This is CommonJS-only. Module._resolveFilename does not exist in the ESM
//    resolver. For ESM you need `tsconfig-paths/esm` loader or, better, one of
//    the other options below.
```

**Honest trade-offs:**

| | `tsx` | `ts-node` + `tsconfig-paths` |
|---|---|---|
| Setup | zero | two packages + a flag |
| Speed | very fast (esbuild) | slow (full tsc) unless `transpileOnly` |
| Type-checks? | **no** — transpile only | yes (unless `transpileOnly`) |
| ESM support | good | fiddly (`tsconfig-paths/esm`, experimental loader warnings) |
| Production use | not intended | not intended |

Both solve **development only**. Neither of them affects what `tsc` emits. If your deploy runs `node dist/server.js`, you still have an unsolved problem.

### Concept 5 — Runtime option B: rewriting the emitted output (`tsc-alias`)

`tsc-alias` is a post-build pass. It walks `dist/`, finds specifiers that match your `paths`, and rewrites them to correct relative paths.

```bash
npm i -D tsc-alias
```

```jsonc
// package.json
{
  "scripts": {
    "build": "rimraf dist && tsc -p tsconfig.build.json && tsc-alias -p tsconfig.build.json",
    "start": "node dist/server.js"
  }
}
```

```js
// dist/routes/orderRoutes.js — AFTER tsc-alias
const orderService_1 = require("../services/orderService");   // ✅ rewritten
const requireAuth_1  = require("../middleware/requireAuth");  // ✅ rewritten
```

```jsonc
// tsconfig.build.json — tsc-alias reads the same paths/baseUrl/outDir
{
  "extends": "./tsconfig.json",
  "compilerOptions": { "outDir": "dist", "rootDir": "src", "noEmit": false },
  "exclude": ["**/*.test.ts", "**/__tests__/**"],
  "tsc-alias": {
    "resolveFullPaths": true,   // also append ".js" to extensionless relative
                                // specifiers — required for real ESM output
    "verbose": false
  }
}
```

**Honest trade-offs:**

- ✅ The shipped `dist/` is plain, portable JavaScript. `node dist/server.js` works with no flags, no preloads, no extra dependency in `dependencies`.
- ✅ Rewrites `.d.ts` files too, so a published package's types don't leak `@/...`.
- ❌ It is a **third-party regex/AST pass over your build output**. It is well maintained, but it is one more thing that can be wrong, and its failures are silent until runtime.
- ❌ It does not know about specifiers built at runtime: `await import(\`@/jobs/${jobName}\`)` cannot be rewritten. Neither can `require(variableHoldingASpecifier)`.
- ❌ Adds a build step that must be replicated in every place you build (CI, Docker, `npm pack`, a coworker's machine).

### Concept 6 — Runtime option C: `module-alias` (and why to avoid it now)

`module-alias` predates TypeScript's `paths` and patches Node's CommonJS resolver from inside your process.

```jsonc
// package.json
{
  "_moduleAliases": {
    "@": "dist"                    // ⚠️ note: points at the OUTPUT, not src/
  },
  "dependencies": { "module-alias": "^2.2.3" }   // a PRODUCTION dependency
}
```

```ts
// src/server.ts — must be the very first import, before anything aliased
import "module-alias/register";
import { createApp } from "@/app";
```

**Honest trade-offs:**

- ✅ Works without a build-output rewrite pass.
- ❌ **CommonJS only.** It patches `Module._resolveFilename`; ESM's resolver is unreachable from it. If your `package.json` has `"type": "module"`, this is a dead end.
- ❌ You now duplicate the alias table in two places (`tsconfig.paths` and `_moduleAliases`) with different values (`src/*` vs `dist`), and nothing checks that they agree.
- ❌ A runtime dependency in `dependencies`, monkey-patching Node internals, in production.
- ❌ Load-order sensitive: anything imported before `module-alias/register` (including transitive imports) fails.

Treat this as legacy. If you inherit it, it works; don't start with it.

### Concept 7 — Runtime option D: a bundler

If you already bundle — esbuild, webpack, Rollup, Vite, `bun build` — alias resolution is free, because bundling *is* specifier resolution. The alias never survives to runtime; there are no specifiers left at all.

```js
// ── esbuild.config.mjs ──────────────────────────────────────────────────────
import { build } from "esbuild";

await build({
  entryPoints: ["src/server.ts"],
  bundle: true,
  platform: "node",
  target: "node20",
  format: "esm",
  outfile: "dist/server.js",
  sourcemap: true,
  packages: "external",          // don't bundle node_modules — keep the image sane
  // esbuild does NOT read tsconfig "paths" for bundling by default in all
  // setups; the reliable options are either an explicit alias map...
  alias: {
    "@": "./src"                 // esbuild >= 0.17
  },
  // ...or the plugin that reads tsconfig:
  // plugins: [tsconfigPathsPlugin()]   // esbuild-plugin-tsconfig-paths
});
```

```js
// ── webpack.config.js ───────────────────────────────────────────────────────
const path = require("node:path");
module.exports = {
  resolve: {
    extensions: [".ts", ".js"],
    alias: {
      "@": path.resolve(__dirname, "src")   // duplicated from tsconfig ⚠️
    }
    // Or avoid the duplication entirely:
    // plugins: [new (require("tsconfig-paths-webpack-plugin"))()]
  }
};
```

**Honest trade-offs:**

- ✅ Zero runtime cost, zero runtime dependency, one artifact.
- ✅ Also gives you tree-shaking, faster cold starts (huge for Lambda), and a smaller image.
- ❌ The alias map is now defined **twice** — tsconfig for the checker, bundler config for the build — unless you use a plugin that reads tsconfig. Drift between them produces "works in the editor, fails in the build".
- ❌ Bundling a Node *server* has real costs: native modules (`bcrypt`, `sharp`, `better-sqlite3`) need `external` handling, dynamic `require` breaks, stack traces need source maps, and some libraries do `__dirname`-relative file loads that break.
- ❌ Bundling *only* to fix path aliases is a large tail wagging a small dog.

### Concept 8 — Runtime option E: `tsconfig-paths` in production (and why it's a trap)

You will find this suggestion everywhere:

```jsonc
{ "scripts": { "start": "node -r tsconfig-paths/register dist/server.js" } }
```

It can be made to work, but understand what it requires:

```jsonc
// tsconfig.runtime.json — a SEPARATE config whose paths point at dist/, not src/
{
  "compilerOptions": {
    "baseUrl": "./dist",           // ← the compiled tree
    "paths": { "@/*": ["./*"] }    // ← "@/services/x" → dist/services/x
  }
}
```

```jsonc
{
  "scripts": {
    "start": "TS_NODE_PROJECT=tsconfig.runtime.json node -r tsconfig-paths/register dist/server.js"
  },
  "dependencies": { "tsconfig-paths": "^4.2.0" }   // now a PROD dependency
}
```

Why this is fragile, concretely:

1. **You must ship a tsconfig to production.** A slim Docker image that copies only `dist/` and `package.json` will fail, and the failure is `MODULE_NOT_FOUND` at boot with no hint that a JSON file is missing.
2. **Two alias tables that must stay in sync** — one pointing at `src/`, one at `dist/`.
3. **`-r` is CommonJS-only.** For ESM you need `--loader tsconfig-paths/esm`, which emits `ExperimentalWarning` and has changed shape across Node versions.
4. **A dev-tooling package sits in `dependencies`** and runs on every cold start.
5. `NODE_OPTIONS`/`-r` is easy to lose — a process manager, a `Procfile`, a Kubernetes `command:` override, or `npm start` being bypassed all drop the flag silently.

Use it in dev. Don't build your production boot sequence on it.

### Concept 9 — Runtime option F: Node's `imports` field (the one I'd pick)

Node has had a built-in, standardised answer since Node 12.19/14.6: **subpath imports**. TypeScript has understood them natively since 4.7.

```jsonc
// ── package.json ────────────────────────────────────────────────────────────
{
  "name": "orders-api",
  "type": "module",
  "imports": {
    "#shared/*": "./dist/shared/*",
    "#domain/*": "./dist/domain/*",
    "#infra/*":  "./dist/infrastructure/*",
    "#config":   "./dist/config/index.js"
  }
}
```

```ts
// ── src/routes/orderRoutes.ts ───────────────────────────────────────────────
import { logger } from "#shared/logging/logger.js";
import { createOrder } from "#domain/orders/createOrder.js";
import { appConfig } from "#config";
// Note the ".js" extension: with module: "nodenext" you write the extension of
// the EMITTED file, even in a .ts source. See 48 — Modules in TypeScript.
```

Now trace what resolves it:

```
tsc            reads package.json "imports"  ✅  (since 4.7, with moduleResolution node16/nodenext/bundler)
node           reads package.json "imports"  ✅  (built-in, CJS and ESM both)
jest           reads package.json "imports"  ✅  (jest-resolve, v28+)
vitest / vite  reads package.json "imports"  ✅
esbuild        reads package.json "imports"  ✅
webpack 5      reads package.json "imports"  ✅
rollup         reads package.json "imports"  ✅ (@rollup/plugin-node-resolve)
bun / deno     read package.json "imports"   ✅
```

That column of ✅ is the entire argument. Every other option on this page requires configuring N tools to agree; this one requires configuring **zero**, because it is a real specification that everyone already implements.

The mapping targets point at `dist/` — the *runtime* location — and the compiler follows them backwards to source through `outDir`/`rootDir` and declaration maps. In a `tsc`-only project you can also point them at source and let a dual condition handle it:

```jsonc
{
  "imports": {
    "#shared/*": {
      "types":   "./dist/shared/*",     // what tsc reads
      "default": "./dist/shared/*"      // what node reads
    }
  }
}
```

Conditional targets are where this gets genuinely powerful — something `paths` cannot do at all:

```jsonc
{
  "imports": {
    // Swap the database implementation by NODE_ENV-driven condition,
    // resolved by Node, with no dependency-injection framework:
    "#db": {
      "test":    "./dist/infra/db/inMemoryPool.js",
      "default": "./dist/infra/db/pgPool.js"
    },
    // Different implementation for the browser bundle vs the server:
    "#crypto/hash": {
      "browser": "./dist/shared/hash.browser.js",
      "node":    "./dist/shared/hash.node.js",
      "default": "./dist/shared/hash.node.js"
    }
  }
}
```

```bash
node --conditions=test dist/scripts/seed.js     # activates the "test" branch
```

**Honest trade-offs:**

- ✅ No build-time rewrite, no runtime dependency, no preload flag, no tool-specific config.
- ✅ Survives Docker `COPY dist/ package.json` — `package.json` is already there.
- ✅ Works identically in CJS and ESM.
- ❌ **Specifiers must start with `#`.** You cannot spell it `@/...`. Some teams find `#shared/logging/logger` ugly; that is the entire cost.
- ❌ **Only your own package can use them.** A different package in a monorepo cannot import your `#shared/*` — which is correct (that's what `exports` is for), but it means workspace-crossing needs `exports` + workspace deps, not aliases.
- ❌ Targets must be relative paths inside the package, and wildcards are literal `*` substitution — no fallback arrays like `paths` has.

**For a plain Node service, this is the least fragile choice.** It is the only option where the compiler and the runtime read the *same file* and there is nothing to keep in sync.

### Concept 10 — Aliases in Jest

Jest has its own resolver and knows nothing about `tsconfig.paths`. You mirror the map in `moduleNameMapper`.

```js
// ── jest.config.js — manual mapping ─────────────────────────────────────────
/** @type {import('jest').Config} */
export default {
  preset: "ts-jest",
  testEnvironment: "node",
  roots: ["<rootDir>/src"],
  moduleNameMapper: {
    // KEY   = a regular expression matched against the specifier
    // VALUE = replacement; $1 is the first capture group
    "^@/(.*)$":       "<rootDir>/src/$1",
    "^@shared/(.*)$": "<rootDir>/src/shared/$1",
    "^@config$":      "<rootDir>/src/config/index.ts",

    // Order matters — Jest uses the FIRST regex that matches, unlike tsc's
    // longest-prefix rule. Put exact/specific patterns ABOVE the catch-all.
  }
};
```

The duplication is the problem, so derive it instead:

```js
// ── jest.config.js — generated from tsconfig, no drift possible ─────────────
import { readFileSync } from "node:fs";
import { pathsToModuleNameMapper } from "ts-jest";

const { compilerOptions } = JSON.parse(
  readFileSync("./tsconfig.json", "utf8").replace(/\/\*[\s\S]*?\*\/|\/\/.*/g, "")  // strip jsonc comments
);

export default {
  preset: "ts-jest",
  testEnvironment: "node",
  moduleNameMapper: pathsToModuleNameMapper(compilerOptions.paths ?? {}, {
    prefix: "<rootDir>/"        // ← required; without it paths are relative to CWD
  })
};
```

Two Jest-specific gotchas that cost real time:

```js
// GOTCHA 1 — with ESM + moduleResolution nodenext you write ".js" in imports,
//            but the file on disk is ".ts". Jest must be told to strip it:
moduleNameMapper: {
  "^@/(.*)\\.js$": "<rootDir>/src/$1",   // put this BEFORE the general rule
  "^@/(.*)$":      "<rootDir>/src/$1"
}

// GOTCHA 2 — Jest's transform cache keys don't always include jest.config
//            changes. After editing moduleNameMapper:
//   npx jest --clearCache
```

If you use Node's `imports` field instead, **all of this disappears** — `jest-resolve` v28+ reads `package.json#imports` natively and `moduleNameMapper` stays empty.

### Concept 11 — Aliases in Vitest

Vitest uses Vite's resolver, so aliases live under `resolve.alias`.

```ts
// ── vitest.config.ts — manual mapping ───────────────────────────────────────
import { defineConfig } from "vitest/config";
import { fileURLToPath } from "node:url";

export default defineConfig({
  resolve: {
    alias: {
      // Object form: prefix replacement (NOT a regex). Longest key wins.
      "@":       fileURLToPath(new URL("./src", import.meta.url)),
      "@shared": fileURLToPath(new URL("./src/shared", import.meta.url)),
      "@config": fileURLToPath(new URL("./src/config/index.ts", import.meta.url))
    }
  },
  test: {
    environment: "node",
    globals: true,
    include: ["src/**/*.test.ts"]
  }
});
```

```ts
// ── vitest.config.ts — derived from tsconfig, no drift ──────────────────────
import { defineConfig } from "vitest/config";
import tsconfigPaths from "vite-tsconfig-paths";   // npm i -D vite-tsconfig-paths

export default defineConfig({
  plugins: [tsconfigPaths()],   // reads tsconfig.json paths at startup — done.
  test: { environment: "node" }
});
```

```ts
// Array form when you need regex control (e.g. stripping ".js" extensions):
resolve: {
  alias: [
    { find: /^@\/(.*)\.js$/, replacement: fileURLToPath(new URL("./src/$1", import.meta.url)) },
    { find: /^@\//,          replacement: fileURLToPath(new URL("./src/",   import.meta.url)) }
  ]
}
```

Again: with `package.json#imports`, none of this is needed. Vite resolves `#shared/*` out of the box.

### Concept 12 — The decision table

```
Question 1: Do you already bundle for deployment?
  YES → use the bundler's alias (or a tsconfig-paths plugin). Done.
  NO  → continue.

Question 2: Is this a plain Node service you deploy as `node dist/server.js`?
  YES → use package.json "imports" (#shared/*). Fewest moving parts. Done.
  NO  → continue.

Question 3: Do you publish this to npm as a library?
  YES → package.json "imports" for internals + "exports" for the public API.
        (Never ship a package whose dist/ contains unresolvable "@/..." — it
        will break in every consumer's build.)
  NO  → continue.

Question 4: Are you retrofitting an existing large codebase already full of "@/"?
  YES → tsc && tsc-alias. Lowest-churn path; you keep the specifiers you have
        and get portable output.

Development loop, orthogonal to all of the above:
  tsx / ts-node -r tsconfig-paths/register — reads tsconfig paths directly.
```

---

## Example 1 — basic

A small service with one alias, wired end to end, with the failure demonstrated first.

```
order-api/
├── package.json
├── tsconfig.json
└── src/
    ├── server.ts
    ├── config/
    │   └── appConfig.ts
    ├── services/
    │   └── orderService.ts
    └── shared/
        └── logger.ts
```

```jsonc
// ── tsconfig.json ───────────────────────────────────────────────────────────
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "rootDir": "src",
    "outDir": "dist",
    "strict": true,
    "sourceMap": true,
    "esModuleInterop": true,
    "skipLibCheck": true,

    // No baseUrl — paths are resolved relative to THIS file (TS 4.1+).
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include": ["src/**/*.ts"],
  "exclude": ["node_modules", "dist"]
}
```

```ts
// ── src/shared/logger.ts ────────────────────────────────────────────────────
export interface LogFields {
  readonly [field: string]: string | number | boolean | null | undefined;
}

export const logger = {
  info(fields: LogFields, message: string): void {
    console.log(JSON.stringify({ level: "info", message, ...fields }));
  },
  error(fields: LogFields, message: string): void {
    console.error(JSON.stringify({ level: "error", message, ...fields }));
  }
};
```

```ts
// ── src/config/appConfig.ts ─────────────────────────────────────────────────
export interface AppConfig {
  readonly port: number;
  readonly databaseUrl: string;
}

export function loadAppConfig(env: NodeJS.ProcessEnv): AppConfig {
  const databaseUrl = env.DATABASE_URL;
  if (!databaseUrl) throw new Error("DATABASE_URL is required");
  return { port: Number(env.PORT ?? 3000), databaseUrl };
}
```

```ts
// ── src/services/orderService.ts ────────────────────────────────────────────
import { logger } from "@/shared/logger.js";   // ← alias + explicit .js (NodeNext)

export interface OrderLine {
  readonly sku: string;
  readonly quantity: number;
  readonly unitPriceCents: number;
}

export interface Order {
  readonly orderId: string;
  readonly customerId: string;
  readonly totalCents: number;
}

export function createOrder(customerId: string, lines: readonly OrderLine[]): Order {
  const totalCents = lines.reduce((sum, l) => sum + l.quantity * l.unitPriceCents, 0);
  const order: Order = { orderId: crypto.randomUUID(), customerId, totalCents };
  logger.info({ orderId: order.orderId, totalCents }, "order created");
  return order;
}
```

```ts
// ── src/server.ts ───────────────────────────────────────────────────────────
import { loadAppConfig } from "@/config/appConfig.js";
import { createOrder } from "@/services/orderService.js";
import { logger } from "@/shared/logger.js";

const config = loadAppConfig(process.env);

const order = createOrder("cus_8812", [
  { sku: "WIDGET-BLUE", quantity: 2, unitPriceCents: 1_299 },
  { sku: "GASKET-M8", quantity: 10, unitPriceCents: 149 }
]);

logger.info({ port: config.port, orderId: order.orderId }, "boot complete");
```

**Step 1 — prove the compiler is happy and the runtime is not.**

```bash
$ npx tsc --noEmit
# (no output — type-checks cleanly) ✅

$ npx tsc && node dist/server.js
node:internal/modules/esm/resolve:265
  throw new ERR_MODULE_NOT_FOUND(...)
        ^
Error [ERR_MODULE_NOT_FOUND]: Cannot find package '@' imported from /app/dist/server.js
  code: 'ERR_MODULE_NOT_FOUND'                                     ❌
```

Confirm why by looking at the emitted file:

```bash
$ head -3 dist/server.js
import { loadAppConfig } from "@/config/appConfig.js";   # ← verbatim
```

**Step 2 — fix the dev loop with `tsx`.**

```jsonc
// package.json
{
  "type": "module",
  "scripts": {
    "dev": "tsx watch src/server.ts",     // ✅ tsx reads tsconfig paths
    "typecheck": "tsc --noEmit",
    "build": "tsc",
    "start": "node dist/server.js"        // ❌ still broken
  },
  "devDependencies": { "tsx": "^4.19.2", "typescript": "^5.7.2", "@types/node": "^22.10.1" }
}
```

```bash
$ npm run dev
{"level":"info","message":"order created","orderId":"...","totalCents":4088}   ✅
```

**Step 3 — fix the build output with `tsc-alias`.**

```bash
npm i -D tsc-alias
```

```jsonc
{
  "scripts": {
    "build": "rimraf dist && tsc && tsc-alias",
    "start": "node dist/server.js"
  }
}
```

```bash
$ npm run build && head -3 dist/server.js
import { loadAppConfig } from "./config/appConfig.js";   # ✅ rewritten

$ npm start
{"level":"info","message":"order created", ... }          ✅
{"level":"info","message":"boot complete","port":3000, ... }
```

**Step 4 — or skip all of that with Node's `imports`.**

```jsonc
// package.json — delete the tsconfig "paths" block entirely
{
  "type": "module",
  "imports": {
    "#src/*": "./dist/*"
  },
  "scripts": {
    "dev": "tsx watch src/server.ts",
    "build": "tsc",
    "start": "node dist/server.js"      // ✅ no rewrite pass, no flag
  }
}
```

```ts
// src/server.ts
import { loadAppConfig } from "#src/config/appConfig.js";
import { createOrder } from "#src/services/orderService.js";
```

```bash
$ npx tsc --noEmit    # ✅ tsc reads package.json "imports"
$ npx tsc && node dist/server.js   # ✅ node reads the same file. Nothing to sync.
```

---

## Example 2 — real world backend use case

A layered order-management API. Aliases carry architectural meaning, the layering is enforced by lint, and the runtime story is decided once.

```
orders-api/
├── package.json
├── tsconfig.json
├── tsconfig.build.json
├── jest.config.js
├── eslint.config.js
├── Dockerfile
└── src/
    ├── server.ts
    ├── app.ts
    ├── config/
    │   ├── index.ts
    │   └── env.ts
    ├── domain/
    │   ├── orders/
    │   │   ├── Order.ts
    │   │   ├── createOrder.ts
    │   │   └── createOrder.test.ts
    │   └── pricing/
    │       └── calculateOrderTotal.ts
    ├── infrastructure/
    │   ├── db/
    │   │   ├── pool.ts
    │   │   └── orderRepository.ts
    │   └── payments/
    │       └── stripeClient.ts
    ├── http/
    │   ├── routes/orderRoutes.ts
    │   └── middleware/requireAuth.ts
    └── shared/
        ├── logging/logger.ts
        ├── errors/AppError.ts
        └── result/Result.ts
```

### The alias table

```jsonc
// ── tsconfig.json — the ONE source of truth for aliases ─────────────────────
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "rootDir": "src",
    "outDir": "dist",

    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "sourceMap": true,
    "declaration": true,
    "declarationMap": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "verbatimModuleSyntax": true,

    // ── Layer aliases. One per architectural boundary, not one per folder. ──
    // Reading an import list now tells you which layers a file depends on.
    "paths": {
      "@config":     ["./src/config/index.ts"],   // exact match — the config barrel
      "@config/*":   ["./src/config/*"],
      "@domain/*":   ["./src/domain/*"],          // pure business logic, no I/O
      "@infra/*":    ["./src/infrastructure/*"],  // db, http clients, queues
      "@http/*":     ["./src/http/*"],            // express wiring
      "@shared/*":   ["./src/shared/*"],          // cross-cutting, dependency-free
      "@test/*":     ["./src/test-support/*"]     // fixtures, builders, fakes
    }
  },
  "include": ["src/**/*.ts"],
  "exclude": ["node_modules", "dist"]
}
```

### What the imports look like in practice

```ts
// ── src/http/routes/orderRoutes.ts ──────────────────────────────────────────
// Every import declares its layer. An HTTP route may reach into domain, infra
// and shared — but nothing may reach back INTO @http. That rule is now lintable.
import { Router, type Request, type Response } from "express";

import { appConfig } from "@config";
import { createOrder } from "@domain/orders/createOrder.js";
import type { OrderLine } from "@domain/orders/Order.js";
import { orderRepository } from "@infra/db/orderRepository.js";
import { requireAuth } from "@http/middleware/requireAuth.js";
import { AppError } from "@shared/errors/AppError.js";
import { logger } from "@shared/logging/logger.js";

export const orderRoutes = Router();

orderRoutes.post("/orders", requireAuth, async (req: Request, res: Response) => {
  const customerId = req.auth.customerId;              // see 53 — extending Request
  const lines = req.body.lines as OrderLine[];

  const result = await createOrder({ customerId, lines, repository: orderRepository });

  if (!result.ok) {
    logger.error({ customerId, reason: result.error.code }, "create order failed");
    throw new AppError(result.error.code, 422, result.error.message);
  }

  logger.info(
    { customerId, orderId: result.value.orderId, region: appConfig.region },
    "order created"
  );
  res.status(201).json({ data: result.value });
});
```

```ts
// ── src/domain/orders/createOrder.ts ────────────────────────────────────────
// Domain code imports ONLY from @domain and @shared. If a reviewer sees
// "@infra/..." or "@http/..." in this file, the layering has been violated —
// and the lint rule below turns that review comment into a CI failure.
import { calculateOrderTotal } from "@domain/pricing/calculateOrderTotal.js";
import type { Order, OrderLine } from "@domain/orders/Order.js";
import { err, ok, type Result } from "@shared/result/Result.js";   // see 61

export interface OrderRepositoryPort {
  insert(order: Order): Promise<void>;
}

export interface CreateOrderInput {
  readonly customerId: string;
  readonly lines: readonly OrderLine[];
  readonly repository: OrderRepositoryPort;
}

export type CreateOrderError =
  | { readonly code: "EMPTY_ORDER"; readonly message: string }
  | { readonly code: "LINE_QUANTITY_INVALID"; readonly message: string };

export async function createOrder(
  input: CreateOrderInput
): Promise<Result<Order, CreateOrderError>> {
  if (input.lines.length === 0) {
    return err({ code: "EMPTY_ORDER", message: "an order needs at least one line" });
  }
  for (const line of input.lines) {
    if (line.quantity <= 0) {
      return err({
        code: "LINE_QUANTITY_INVALID",
        message: `quantity for ${line.sku} must be positive`
      });
    }
  }

  const order: Order = {
    orderId: crypto.randomUUID(),
    customerId: input.customerId,
    lines: input.lines,
    totalCents: calculateOrderTotal(input.lines),
    createdAt: new Date()
  };
  await input.repository.insert(order);
  return ok(order);
}
```

### Enforcing the layering with the aliases you just created

```js
// ── eslint.config.js — aliases turn architecture into a lint rule ───────────
import tseslint from "typescript-eslint";
import importPlugin from "eslint-plugin-import";

export default tseslint.config({
  files: ["src/**/*.ts"],
  plugins: { import: importPlugin },
  settings: {
    // Make eslint-plugin-import understand tsconfig paths, otherwise every
    // "@domain/..." import is reported as unresolved.
    "import/resolver": { typescript: { project: "./tsconfig.json" } }
  },
  rules: {
    "import/no-unresolved": "error",

    // The payoff: layering rules expressed against alias prefixes.
    "import/no-restricted-paths": ["error", {
      zones: [
        { target: "./src/domain",         from: "./src/infrastructure",
          message: "domain must not depend on infrastructure — inject a port instead" },
        { target: "./src/domain",         from: "./src/http",
          message: "domain must not depend on the HTTP layer" },
        { target: "./src/shared",         from: "./src/domain",
          message: "shared must stay dependency-free" },
        { target: "./src/infrastructure", from: "./src/http",
          message: "infrastructure must not depend on the HTTP layer" }
      ]
    }],

    // Ban relative escapes so the aliases actually get used.
    "no-restricted-imports": ["error", {
      patterns: [{
        group: ["../*", "../../*", "../../../*"],
        message: "Use a path alias (@domain/, @infra/, @shared/) instead of ../"
      }]
    }]
  }
});
```

### The build: `tsc` + `tsc-alias`

```jsonc
// ── tsconfig.build.json ─────────────────────────────────────────────────────
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "outDir": "dist",
    "rootDir": "src",
    "sourceMap": true,
    "removeComments": true
  },
  // Tests must NOT be compiled into the shipped image, and @test/* must not
  // leak into dist/ — an unresolvable alias in prod is exactly the failure mode.
  "exclude": [
    "node_modules", "dist",
    "**/*.test.ts", "**/*.spec.ts", "src/test-support/**"
  ],
  "tsc-alias": {
    "resolveFullPaths": true,   // append .js to extensionless relative specifiers
    "verbose": false
  }
}
```

```jsonc
// ── package.json ────────────────────────────────────────────────────────────
{
  "name": "orders-api",
  "type": "module",
  "engines": { "node": ">=20" },
  "scripts": {
    "dev":        "tsx watch src/server.ts",
    "typecheck":  "tsc --noEmit -p tsconfig.json",
    "lint":       "eslint src --max-warnings=0",
    "test":       "jest",
    "build":      "rimraf dist && tsc -p tsconfig.build.json && tsc-alias -p tsconfig.build.json",
    "verify:dist": "node scripts/assertNoAliasesInDist.mjs",
    "start":      "node --enable-source-maps dist/server.js",
    "prepack":    "npm run build && npm run verify:dist"
  },
  "dependencies": { "express": "^4.21.2", "pg": "^8.13.1" },
  "devDependencies": {
    "tsx": "^4.19.2", "typescript": "^5.7.2", "tsc-alias": "^1.8.10",
    "jest": "^29.7.0", "ts-jest": "^29.2.5", "@types/node": "^22.10.1",
    "eslint-import-resolver-typescript": "^3.7.0", "rimraf": "^6.0.1"
  }
}
```

### The guard rail that catches the failure before deploy does

```js
// ── scripts/assertNoAliasesInDist.mjs ───────────────────────────────────────
// CI gate. Costs 200ms and prevents the entire class of "MODULE_NOT_FOUND on
// boot" incidents that path aliases are famous for. Run it after every build.
import { readFileSync } from "node:fs";
import { glob } from "node:fs/promises";

const ALIAS_PREFIXES = ["@config", "@domain", "@infra", "@http", "@shared", "@test"];
const SPECIFIER_RE = /(?:from\s+|import\s*\(|require\s*\()\s*["'`]([^"'`]+)["'`]/g;

let offences = 0;
for await (const file of glob("dist/**/*.{js,d.ts}")) {
  const source = readFileSync(file, "utf8");
  for (const [, specifier] of source.matchAll(SPECIFIER_RE)) {
    if (ALIAS_PREFIXES.some((p) => specifier === p || specifier.startsWith(`${p}/`))) {
      console.error(`✗ ${file}: unresolved alias "${specifier}"`);
      offences++;
    }
  }
}

if (offences > 0) {
  console.error(`\n${offences} unresolved alias specifier(s) in dist/.`);
  console.error("tsc-alias did not run, or its config does not match tsconfig paths.");
  process.exit(1);
}
console.log("✓ dist/ contains no unresolved path aliases");
```

### Tests: Jest, with the map derived from tsconfig

```js
// ── jest.config.js ──────────────────────────────────────────────────────────
import { readFileSync } from "node:fs";
import { pathsToModuleNameMapper } from "ts-jest";

const raw = readFileSync("./tsconfig.json", "utf8").replace(/\/\*[\s\S]*?\*\/|\/\/.*/g, "");
const { compilerOptions } = JSON.parse(raw);

/** @type {import('jest').Config} */
export default {
  preset: "ts-jest/presets/default-esm",
  testEnvironment: "node",
  extensionsToTreatAsEsm: [".ts"],
  roots: ["<rootDir>/src"],

  moduleNameMapper: {
    // 1. Strip the ".js" that NodeNext makes us write in .ts sources.
    //    MUST come first — Jest takes the first matching regex.
    "^(@(?:config|domain|infra|http|shared|test)/.*)\\.js$": "$1",
    // 2. Then the generated alias map.
    ...pathsToModuleNameMapper(compilerOptions.paths ?? {}, { prefix: "<rootDir>/" }),
    // 3. And relative ESM specifiers.
    "^(\\.{1,2}/.*)\\.js$": "$1"
  },

  transform: {
    "^.+\\.tsx?$": ["ts-jest", { useESM: true, tsconfig: "tsconfig.json" }]
  },
  collectCoverageFrom: ["src/**/*.ts", "!src/**/*.test.ts", "!src/test-support/**"]
};
```

```ts
// ── src/domain/orders/createOrder.test.ts ───────────────────────────────────
import { createOrder, type OrderRepositoryPort } from "@domain/orders/createOrder.js";
import { anOrderLine } from "@test/builders/orderBuilders.js";   // @test/* alias

describe("createOrder", () => {
  const insert = jest.fn<Promise<void>, [unknown]>().mockResolvedValue(undefined);
  const repository: OrderRepositoryPort = { insert } as OrderRepositoryPort;

  it("rejects an order with no lines", async () => {
    const result = await createOrder({ customerId: "cus_8812", lines: [], repository });
    expect(result.ok).toBe(false);
    if (!result.ok) expect(result.error.code).toBe("EMPTY_ORDER");
    expect(insert).not.toHaveBeenCalled();
  });

  it("sums line totals in cents", async () => {
    const result = await createOrder({
      customerId: "cus_8812",
      lines: [
        anOrderLine({ sku: "WIDGET-BLUE", quantity: 2, unitPriceCents: 1_299 }),
        anOrderLine({ sku: "GASKET-M8", quantity: 10, unitPriceCents: 149 })
      ],
      repository
    });
    expect(result.ok).toBe(true);
    if (result.ok) expect(result.value.totalCents).toBe(4_088);
  });
});
```

### Tests: the Vitest equivalent

```ts
// ── vitest.config.ts ────────────────────────────────────────────────────────
import { defineConfig } from "vitest/config";
import tsconfigPaths from "vite-tsconfig-paths";

export default defineConfig({
  plugins: [tsconfigPaths()],       // reads tsconfig paths — nothing duplicated
  test: {
    environment: "node",
    include: ["src/**/*.test.ts"],
    coverage: { provider: "v8", exclude: ["src/test-support/**", "**/*.test.ts"] }
  }
});
```

```ts
// ── vitest.config.ts — explicit form, if you don't want the plugin ──────────
import { defineConfig } from "vitest/config";
import { fileURLToPath } from "node:url";

const at = (p: string) => fileURLToPath(new URL(p, import.meta.url));

export default defineConfig({
  resolve: {
    alias: [
      // Array form so we can strip the ".js" suffix with a regex first.
      { find: /^@domain\/(.*)\.js$/, replacement: `${at("./src/domain/")}$1` },
      { find: /^@infra\/(.*)\.js$/,  replacement: `${at("./src/infrastructure/")}$1` },
      { find: /^@shared\/(.*)\.js$/, replacement: `${at("./src/shared/")}$1` },
      { find: /^@http\/(.*)\.js$/,   replacement: `${at("./src/http/")}$1` },
      { find: /^@test\/(.*)\.js$/,   replacement: `${at("./src/test-support/")}$1` },
      { find: "@config",             replacement: at("./src/config/index.ts") }
    ]
  },
  test: { environment: "node" }
});
```

### And the version of all of this with zero extra tooling

Here is the same service using `package.json#imports` instead. Note what disappears: `tsc-alias`, the `verify:dist` guard, `moduleNameMapper`, `resolve.alias`, `vite-tsconfig-paths`, and the `paths` block itself.

```jsonc
// ── package.json ────────────────────────────────────────────────────────────
{
  "name": "orders-api",
  "type": "module",
  "imports": {
    "#config":   "./dist/config/index.js",
    "#config/*": "./dist/config/*",
    "#domain/*": "./dist/domain/*",
    "#infra/*":  "./dist/infrastructure/*",
    "#http/*":   "./dist/http/*",
    "#shared/*": "./dist/shared/*"
  },
  "scripts": {
    "dev":       "tsx watch src/server.ts",
    "typecheck": "tsc --noEmit",
    "test":      "jest",                       // no moduleNameMapper needed
    "build":     "rimraf dist && tsc -p tsconfig.build.json",   // no second pass
    "start":     "node --enable-source-maps dist/server.js"     // no flag, no dep
  }
}
```

```ts
// ── src/http/routes/orderRoutes.ts ──────────────────────────────────────────
import { appConfig } from "#config";
import { createOrder } from "#domain/orders/createOrder.js";
import { orderRepository } from "#infra/db/orderRepository.js";
import { logger } from "#shared/logging/logger.js";
// tsc resolves these via package.json "imports" (moduleResolution: NodeNext).
// node resolves these via package.json "imports".
// jest resolves these via package.json "imports".
// There is nothing to keep in sync, because there is only one table.
```

```dockerfile
# ── Dockerfile — why "imports" survives a slim image ────────────────────────
FROM node:20-slim AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:20-slim AS runtime
WORKDIR /app
ENV NODE_ENV=production
COPY package*.json ./
RUN npm ci --omit=dev
COPY --from=build /app/dist ./dist
# package.json is already here (needed for npm ci), and it carries the alias
# table. With tsconfig-paths you would ALSO need to COPY tsconfig.runtime.json
# and keep tsconfig-paths in "dependencies" — two more things to forget.
CMD ["node", "--enable-source-maps", "dist/server.js"]
```

---

## Going deeper

### `paths` in a monorepo: the wrong tool for the job

The tempting move in a workspace repo:

```jsonc
// ❌ Root tsconfig.json — reaching across package boundaries with paths
{
  "compilerOptions": {
    "paths": {
      "@acme/billing/*": ["./packages/billing/src/*"],
      "@acme/shared/*":  ["./packages/shared/src/*"]
    }
  }
}
```

This compiles, and it breaks everything downstream: `tsc` inlines the other package's *source* into the consumer's compilation, so incremental builds don't work, `dist/` contains specifiers no runtime resolves, and publishing any of these packages produces broken artifacts.

```jsonc
// ✅ Use workspaces + project references instead (see 77).
// packages/shared/package.json
{
  "name": "@acme/shared",
  "type": "module",
  "exports": { ".": "./dist/index.js", "./logging": "./dist/logging/index.js" },
  "types": "./dist/index.d.ts"
}

// services/orders-api/package.json
{ "dependencies": { "@acme/shared": "workspace:*" } }

// services/orders-api/tsconfig.json
{ "references": [{ "path": "../../packages/shared" }] }
```

Then `import { logger } from "@acme/shared/logging"` is a **real package import**, resolved identically by tsc, node, jest, and every bundler — and it survives publishing. Keep `paths`/`imports` for *intra*-package aliasing only.

The one legitimate monorepo use of `paths` is the "go to source, not dist" trick during development:

```jsonc
// tsconfig.dev.json — used only by the editor and the dev server, never to build
{
  "compilerOptions": {
    "paths": {
      "@acme/shared":   ["./packages/shared/src/index.ts"],
      "@acme/shared/*": ["./packages/shared/src/*"]
    }
  }
}
// Effect: Cmd-click goes to the .ts source and edits are picked up without
// rebuilding the dependency. Just never point your BUILD config at this.
```

### `paths` inside a published npm package: don't

If you `npm publish` a package whose `dist/` contains `require("@/shared/logger")`, every consumer's build breaks with an unresolvable specifier — and they cannot fix it, because `@/*` means something different in their repo (or nothing at all).

Rules:

1. If you use `paths` in a library, you **must** run `tsc-alias` (or bundle) before publishing.
2. `tsc-alias` also rewrites `.d.ts` — verify this, because unresolvable aliases in type declarations produce `any` in consumers, silently.
3. Add a check to `prepack`, not just to CI:

```jsonc
{ "scripts": { "prepack": "npm run build && node scripts/assertNoAliasesInDist.mjs" } }
```

4. Prefer `package.json#imports` in libraries. `#` specifiers are *scoped to the defining package* by specification, so a consumer's `#shared/*` and yours never collide, and they resolve correctly inside `node_modules` with no rewrite.

### `rootDirs` — the sibling option nobody knows about

`rootDirs` is not an alias mechanism; it merges several directories into one *virtual* directory for the purpose of **relative** import resolution.

```jsonc
{
  "compilerOptions": {
    "rootDirs": ["src", "generated"]
  }
}
```

```
src/
  http/routes/orderRoutes.ts
generated/
  http/graphqlTypes.ts        ← emitted by codegen into a separate tree
```

```ts
// src/http/routes/orderRoutes.ts
import type { OrderInput } from "./graphqlTypes.js";
//                               ^^^^^^^^^^^^^^^^^ a SIBLING in the virtual dir,
//                               physically in generated/http/graphqlTypes.ts
```

Use it when a codegen step writes into a parallel tree that will be *merged* at build time (both compile into the same `outDir`, so the relative path becomes correct in the output). Use `paths` when the trees stay separate.

### Aliases and `import type` / `verbatimModuleSyntax`

A pleasant consequence worth knowing: type-only imports are **erased entirely** at emit, so an alias used only for types can never cause a runtime failure.

```ts
import type { Order } from "@domain/orders/Order.js";   // erased — dist/ has no
                                                        // specifier at all ✅
import { createOrder } from "@domain/orders/createOrder.js";   // survives ❌
```

With `verbatimModuleSyntax: true` this becomes predictable rather than accidental: what you wrote with `import type` is erased, what you wrote as a value import is emitted. During a migration you can safely leave alias-imported *types* alone and only fix value imports — but don't rely on this as a strategy; it hides the problem instead of solving it.

### Alias-aware editor and tooling checklist

Things that need to know about your aliases, and how they find out:

| Tool | Reads tsconfig `paths`? | Reads `package.json#imports`? |
|---|---|---|
| VS Code / tsserver | ✅ automatically | ✅ automatically |
| `tsc` | ✅ | ✅ (TS 4.7+, node16/nodenext/bundler) |
| `tsx` | ✅ | ✅ |
| `ts-node` | ❌ needs `tsconfig-paths` | ✅ |
| `node` | ❌ never | ✅ built-in |
| Jest | ❌ needs `moduleNameMapper` | ✅ (v28+) |
| Vitest / Vite | ❌ needs `resolve.alias` or plugin | ✅ |
| esbuild | ❌ needs `alias` or plugin | ✅ |
| webpack 5 | ❌ needs `resolve.alias` or plugin | ✅ |
| Rollup | ❌ needs `@rollup/plugin-alias` | ✅ (node-resolve) |
| eslint-plugin-import | ❌ needs `eslint-import-resolver-typescript` | ✅ |
| `nodemon` + `node` | ❌ | ✅ |
| Bun / Deno | ✅ (bun) | ✅ |

Read down the two columns. The right-hand one is why the recommendation at the top of this document is what it is.

### Migrating an existing codebase to aliases

Do it mechanically, in this order, and keep every step independently revertable:

```bash
# 1. Decide the runtime story FIRST. Nothing else matters until this is settled.
#    (imports field / tsc-alias / bundler — pick one, write it down.)

# 2. Add the paths block. Change no source yet. Confirm `tsc --noEmit` still passes.

# 3. Add the CI guard that fails on unresolved aliases in dist/.
#    Add it BEFORE any imports exist, so it never has a chance to be "temporarily skipped".

# 4. Convert ONE leaf module's imports. Build, run, test, deploy to staging.

# 5. Bulk-convert with a codemod, one alias prefix at a time:
npx jscodeshift -t ./codemods/relativeToAlias.ts src/ --alias='@shared' --dir='src/shared'

# 6. Turn on the lint rule that bans "../../" so the change is permanent.
```

The order matters: teams that convert imports first and think about runtime last are the ones who find out in production.

### Why the compiler will not rewrite specifiers (the argument, fairly stated)

The TypeScript team's reasoning, so you can stop hoping this changes:

1. **tsc is a type checker with an emitter, not a bundler.** Rewriting specifiers means computing output-relative paths, which requires knowing the deployment layout — information tsc does not and should not have.
2. **The correct rewrite is ambiguous.** `@/shared/logger` might need to become `../shared/logger.js`, `../shared/logger/index.js`, or a bare package specifier, depending on `module`, `moduleResolution`, and the consumer.
3. **`paths` was designed to describe an existing runtime**, not to create one. Its original purpose was "my bundler/AMD loader already does this remapping — tell the checker about it so types line up." Using it to *invent* a mapping the runtime doesn't have is using it backwards.

Point 3 is the useful one. `paths` is a *description* of your runtime's behaviour. If your runtime doesn't behave that way, the description is a lie, and lies compile fine.

---

## Common mistakes

### Mistake 1 — Adding `paths` and shipping without a runtime resolver

```jsonc
// ❌ tsconfig.json
{ "compilerOptions": { "baseUrl": ".", "paths": { "@/*": ["src/*"] } } }
```
```jsonc
// ❌ package.json — `dev` works, so everyone assumes `start` will too
{
  "scripts": {
    "dev":   "tsx watch src/server.ts",   // ✅ tsx resolves aliases
    "build": "tsc",                       // emits "@/..." verbatim
    "start": "node dist/server.js"        // 💥 MODULE_NOT_FOUND on boot
  }
}
```

The reason this reaches production: the dev loop, the type check, the editor, and the tests can *all* pass. Only `node dist/server.js` fails, and many teams never run that locally.

```jsonc
// ✅ Fix A — rewrite the output
{
  "scripts": {
    "build": "rimraf dist && tsc && tsc-alias",
    "start": "node dist/server.js",
    "prestart": "node scripts/assertNoAliasesInDist.mjs"   // and prove it
  }
}

// ✅ Fix B — use a mechanism the runtime already understands
// package.json: { "imports": { "#shared/*": "./dist/shared/*" } }
// delete the "paths" block; `node dist/server.js` just works.
```

Whatever you pick, add a smoke test that actually boots the built artifact:

```yaml
# .github/workflows/ci.yml
- run: npm run build
- run: node scripts/assertNoAliasesInDist.mjs
- run: timeout 15s node dist/server.js --health-check || exit 1
```

### Mistake 2 — Assuming Jest and Vitest inherit the aliases

```ts
// ❌ src/domain/orders/createOrder.test.ts
import { createOrder } from "@domain/orders/createOrder.js";
```
```
$ npx jest
 ● Test suite failed to run
   Cannot find module '@domain/orders/createOrder.js' from 'src/domain/orders/createOrder.test.ts'
```

Jest resolves modules itself and has never read `tsconfig.paths`. Same for Vitest (it reads Vite's config, not tsconfig).

```js
// ✅ Jest — derive the map so it cannot drift
import { pathsToModuleNameMapper } from "ts-jest";
import { readFileSync } from "node:fs";
const { compilerOptions } = JSON.parse(
  readFileSync("./tsconfig.json", "utf8").replace(/\/\*[\s\S]*?\*\/|\/\/.*/g, "")
);
export default {
  preset: "ts-jest",
  moduleNameMapper: {
    "^(@\\w+/.*)\\.js$": "$1",                                       // strip .js first
    ...pathsToModuleNameMapper(compilerOptions.paths, { prefix: "<rootDir>/" })
  }
};
```

```ts
// ✅ Vitest — one plugin, no duplication
import { defineConfig } from "vitest/config";
import tsconfigPaths from "vite-tsconfig-paths";
export default defineConfig({ plugins: [tsconfigPaths()] });
```

### Mistake 3 — `moduleNameMapper` / `resolve.alias` ordering

```js
// ❌ The catch-all swallows the specific rule. Jest takes the FIRST regex
//    that matches — unlike tsc, which takes the LONGEST prefix.
moduleNameMapper: {
  "^@/(.*)$":       "<rootDir>/src/$1",         // matches "@/config" too
  "^@/config$":     "<rootDir>/src/config/index.ts"   // ← never reached
}
// Result: "@/config" resolves to src/config (the directory), which may or may
// not have an index — a confusing "Cannot find module" for a path that exists.
```

```js
// ✅ Most specific first.
moduleNameMapper: {
  "^@/config$":     "<rootDir>/src/config/index.ts",
  "^@/(.*)\\.js$":  "<rootDir>/src/$1",
  "^@/(.*)$":       "<rootDir>/src/$1"
}
```

The same trap exists in Vitest's **array** form (first `find` wins), though not in its object form (longest key wins). If you're using arrays, order them specific-to-general.

### Mistake 4 — Pointing runtime aliases at `src/` instead of `dist/`

```jsonc
// ❌ package.json — copied straight from the tsconfig paths block
{
  "imports": { "#shared/*": "./src/shared/*" },
  "scripts": { "start": "node dist/server.js" }
}
// dist/server.js does  import("#shared/logging/logger.js")
//   → resolves to ./src/shared/logging/logger.js
//   → that file is TypeScript. Node: ERR_UNKNOWN_FILE_EXTENSION ".ts"
//   → or, in a slim Docker image, src/ isn't even there: ERR_MODULE_NOT_FOUND
```

```jsonc
// ✅ Runtime aliases point at what the runtime executes.
{
  "imports": { "#shared/*": "./dist/shared/*" }
}
// tsc still resolves them correctly: it maps dist/shared/x.js back to
// src/shared/x.ts via outDir/rootDir. This is exactly the intended usage.
```

The same bug with `tsconfig-paths` in production:

```jsonc
// ❌ node -r tsconfig-paths/register dist/server.js  using the DEV tsconfig
{ "compilerOptions": { "baseUrl": ".", "paths": { "@/*": ["src/*"] } } }

// ✅ A separate runtime config anchored on dist/
// tsconfig.runtime.json
{ "compilerOptions": { "baseUrl": "./dist", "paths": { "@/*": ["./*"] } } }
```

### Mistake 5 — Relying on `baseUrl` for bare specifiers

```jsonc
// ❌ tsconfig.json
{ "compilerOptions": { "baseUrl": "./src" } }   // no paths at all
```
```ts
// ❌ src/http/routes/orderRoutes.ts
import { createOrder } from "domain/orders/createOrder";
//                           ^^^^^^^^^^^^^^^^^^^^^^^^^ resolves to src/domain/...
//
// Problems, in order of severity:
//  1. Node WILL look for node_modules/domain — MODULE_NOT_FOUND at runtime.
//  2. Indistinguishable from a real package import when reading the file.
//  3. If anyone ever installs a package literally named "domain" or "config"
//     or "shared" (all exist on npm), resolution silently changes target.
```

```ts
// ✅ Use a prefix that can never be a package name.
import { createOrder } from "@/domain/orders/createOrder.js";   // "@/" is not a
                                                                 // valid pkg name
// ✅ Or Node's, which is reserved by specification:
import { createOrder } from "#domain/orders/createOrder.js";
```

```jsonc
// ✅ And drop baseUrl entirely (TS 4.1+):
{ "compilerOptions": { "paths": { "@/*": ["./src/*"] } } }
```

Note that `@acme/*` (a real npm scope prefix) is a *worse* choice than `@/*` for the same reason — it collides with published scoped packages. `@/` and `#` are the two safe conventions.

### Mistake 6 — `extends` and the surprising re-anchoring of `paths`

```jsonc
// ❌ packages/orders-api/tsconfig.json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": { "outDir": "dist" }
}

// tsconfig.base.json (at the repo root)
{ "compilerOptions": { "baseUrl": ".", "paths": { "@/*": ["src/*"] } } }
//
// Historically `baseUrl` in an extended config resolved relative to the BASE
// file's directory — so "@/x" meant <repo-root>/src/x, not
// <repo>/packages/orders-api/src/x. Every alias silently pointed at the wrong
// package, or at nothing.
```

```jsonc
// ✅ Re-declare paths in the child config, where the files actually live:
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "outDir": "dist",
    "paths": { "@/*": ["./src/*"] }    // relative to THIS file (no baseUrl)
  }
}
```

Rule of thumb: keep compiler *behaviour* (strictness, target, lib) in the shared base; keep anything **path-shaped** (`baseUrl`, `paths`, `outDir`, `rootDir`, `include`) in the leaf config that owns those files. Also note `paths` does not merge across `extends` — a child's `paths` **replaces** the parent's wholesale.

### Mistake 7 — Aliasing test-only code into production output

```jsonc
// ❌ tsconfig.json used for the build
{ "compilerOptions": { "paths": { "@test/*": ["./src/test-support/*"] } } }
```
```ts
// ❌ Someone imports a test fixture from production code "just for the seed script"
import { anOrderLine } from "@test/builders/orderBuilders.js";
// tsconfig.build.json excludes src/test-support/** → the file is never compiled
// → dist/ imports a module that does not exist → MODULE_NOT_FOUND, in prod only.
```

```jsonc
// ✅ Make it structurally impossible: test aliases live in the test config only.
// tsconfig.json (dev + editor)   → has @test/*
// tsconfig.build.json            → excludes test-support AND drops @test/*
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "outDir": "dist",
    "paths": {
      "@config": ["./src/config/index.ts"],
      "@domain/*": ["./src/domain/*"],
      "@infra/*": ["./src/infrastructure/*"],
      "@http/*": ["./src/http/*"],
      "@shared/*": ["./src/shared/*"]
      // no @test/* → importing one is a compile error in the build ✅
    }
  },
  "exclude": ["**/*.test.ts", "src/test-support/**"]
}
```

---

## Practice exercises

### Exercise 1 — easy

Set up path aliases in a scratch project, then **prove to yourself** that `tsc` does not rewrite them.

Concretely:

1. `mkdir alias-lab && cd alias-lab && npm init -y && npm i -D typescript @types/node tsx`. Set `"type": "module"` in `package.json`.
2. Write a `tsconfig.json` with `target: "ES2022"`, `module: "NodeNext"`, `moduleResolution: "NodeNext"`, `rootDir: "src"`, `outDir: "dist"`, `strict: true`, and `paths: { "@/*": ["./src/*"] }` — **no `baseUrl`**.
3. Create `src/shared/logger.ts` exporting a `logger` object with `info(fields, message)`, and `src/services/inventoryService.ts` exporting `reserveStock(sku: string, quantity: number): { sku: string; reserved: number }` which calls `logger.info`.
4. Create `src/server.ts` that imports both **via `@/...`** and calls `reserveStock("WIDGET-BLUE", 3)`.
5. Run `npx tsc --noEmit`. Record that it passes.
6. Run `npx tsc`, then open `dist/server.js` and **read the first three lines**. Write down, in a comment in your notes, the exact specifier that was emitted.
7. Run `node dist/server.js`. Copy the full error, including the `code:` field.
8. Now run `npx tsx src/server.ts`. It works. Explain in two sentences why the same import succeeds under `tsx` and fails under `node`.
9. Add `"@shared/*": ["./src/shared/*"]` and an **exact** alias `"@logger": ["./src/shared/logger.ts"]`. Import the logger three different ways (`@/shared/logger.js`, `@shared/logger.js`, `@logger`) and confirm all three type-check.

```
// Write your code here
```

### Exercise 2 — medium

Take the Exercise 1 project and make the **compiled output** run, three different ways. Compare them honestly.

Concretely:

1. Expand the project into layers: `src/config/index.ts`, `src/domain/pricing/calculateOrderTotal.ts`, `src/infrastructure/db/inMemoryOrderRepository.ts`, `src/http/routes/orderRoutes.ts` (Express), `src/shared/logging/logger.ts`, `src/server.ts`. Give each layer its own alias: `@config`, `@domain/*`, `@infra/*`, `@http/*`, `@shared/*`.
2. **Approach A — `tsc-alias`.** `npm i -D tsc-alias`. Add `tsconfig.build.json` with `resolveFullPaths: true`. Make `npm run build && npm start` work. Then inspect `dist/http/routes/orderRoutes.js` and record exactly what each alias became.
3. **Approach B — `tsconfig-paths` at runtime.** Write `tsconfig.runtime.json` with `baseUrl: "./dist"` and paths pointing at `./*`. Get `node -r tsconfig-paths/register dist/server.js` to boot (you'll need CommonJS output for `-r`, or `--loader tsconfig-paths/esm` for ESM — try both and record which warnings you get). Then **delete `tsconfig.runtime.json`** and observe the failure mode.
4. **Approach C — `package.json#imports`.** Delete the `paths` block entirely. Add an `imports` map with `#config`, `#domain/*`, `#infra/*`, `#http/*`, `#shared/*` pointing at `./dist/...`. Rewrite every import to `#`-prefixed specifiers. Confirm that `npx tsc --noEmit` passes **and** `node dist/server.js` boots with no flags, no preload, and no extra dependency.
5. Write `scripts/assertNoAliasesInDist.mjs` that scans `dist/**/*.js` for any specifier starting with `@config`/`@domain`/`@infra`/`@http`/`@shared` and exits non-zero. Wire it as `prestart`. Break Approach A on purpose (remove the `tsc-alias` step) and confirm the guard catches it.
6. Add tests. Write `src/domain/pricing/calculateOrderTotal.test.ts` and get it green under **both** Jest (`moduleNameMapper` via `pathsToModuleNameMapper`) and Vitest (`vite-tsconfig-paths`). Then switch the project to Approach C and **delete** both mapping configurations — confirm the tests still pass.
7. Write a short table comparing A, B and C on: number of config files touched, extra production dependencies, whether it works in a Docker image containing only `dist/` + `package.json`, and what happens if someone bypasses `npm start` and runs `node dist/server.js` directly.

```
// Write your code here
```

### Exercise 3 — hard

Build a realistic multi-package repo, enforce architecture with aliases, and reproduce every alias failure mode deliberately.

Concretely:

1. **Scaffold.** npm workspaces with `packages/shared` (published-style package with `exports`) and `services/orders-api` (the app). `services/orders-api` depends on `@acme/shared` as a **workspace dependency with a real `exports` map** — not via `paths`. Prove the distinction by first doing it *wrong* with `paths: { "@acme/shared/*": ["../../packages/shared/src/*"] }`, building, and documenting exactly what breaks in `dist/`.
2. **Intra-package aliases.** Inside `services/orders-api`, use `package.json#imports` with `#config`, `#domain/*`, `#infra/*`, `#http/*`, `#shared/*`. Implement: an order-creation use case returning a `Result` type, a Postgres repository behind a port interface, an Express route, and a Zod-validated config loader.
3. **Conditional imports.** Use the conditional form of `imports` so that `#db` resolves to `./dist/infra/db/pgPool.js` normally and `./dist/infra/db/inMemoryPool.js` under `--conditions=test`. Write one integration test that runs with `node --conditions=test` and confirm no dependency-injection wiring was needed.
4. **Layer enforcement.** Configure `eslint-plugin-import` with `no-restricted-paths` zones so that `domain` cannot import `infra` or `http`, and `shared` cannot import anything else. Add `no-restricted-imports` banning `../*` patterns. Write one deliberately illegal import per rule and confirm each fails `npm run lint` with your custom message.
5. **Reproduce five failures**, and for each one record the *exact* error text and the minimum fix:
   - a. `paths` alias in `dist/` with no rewrite pass → boot failure.
   - b. Alias map pointing at `src/` while running `dist/` → `ERR_UNKNOWN_FILE_EXTENSION`.
   - c. Jest with no `moduleNameMapper` → "Cannot find module … from …".
   - d. `moduleNameMapper` ordering bug: a catch-all shadowing an exact alias.
   - e. A child `tsconfig.json` that `extends` a base whose `paths` re-anchor to the base's directory.
6. **Docker.** Two-stage `Dockerfile`. The runtime stage copies only `package*.json` and `dist/`. Prove the `imports`-based service boots. Then convert one alias back to `tsconfig-paths/register`, rebuild the image *without* copying the tsconfig, and document the failure. Restore.
7. **Publishing.** Add `packages/shared` build with `tsc-alias`, and a `prepack` script running your `assertNoAliasesInDist.mjs` over both `dist/**/*.js` **and** `dist/**/*.d.ts`. Deliberately publish-pack (`npm pack`) a version with an unrewritten alias in a `.d.ts`, install the tarball into a scratch consumer, and describe what the consumer sees (hint: it is worse than an error).
8. **Codemod.** Write a `jscodeshift` transform that converts relative imports of depth ≥ 2 (`../../`) into the correct `#`-prefixed specifier, based on the `imports` map in `package.json`. Run it over the whole repo and verify with `tsc --noEmit`, `npm test`, and your dist guard.
9. Finally, write `docs/module-resolution.md` for the repo: the alias table, the chosen runtime mechanism and *why*, the list of tools that had to be configured (should be short), and a symptom → cause → fix table covering at least eight distinct failures you actually hit.

```
// Write your code here
```

---

## Quick reference cheat sheet

```jsonc
// tsconfig.json — compile-time only
"baseUrl": "."                  // anchor for paths; also makes bare specifiers
                                // resolve locally — prefer omitting it (TS 4.1+)
"paths": {
  "@/*":       ["./src/*"],     // wildcard — one "*" per side, max
  "@config":   ["./src/config/index.ts"],       // exact, whole-specifier match
  "@gen/*":    ["./src/gen/*", "./build/gen/*"] // fallback list, first existing wins
}
"rootDirs": ["src", "generated"] // merge trees for RELATIVE imports (not aliases)
// Longest matching prefix wins. paths does NOT merge across "extends" — it replaces.
```

```jsonc
// package.json — runtime + compile-time, one table
"imports": {
  "#shared/*": "./dist/shared/*",              // keys MUST start with "#"
  "#config":   "./dist/config/index.js",
  "#db": { "test": "./dist/db/fake.js", "default": "./dist/db/pg.js" }
}
// node --conditions=test dist/server.js   ← activates the "test" branch
```

```bash
# Runtime resolution — pick one and write it down
npx tsx src/server.ts                              # dev: reads tsconfig paths
npx ts-node -r tsconfig-paths/register src/x.ts    # dev: CJS only
npx tsc && npx tsc-alias                           # build: rewrites dist/ specifiers
node -r tsconfig-paths/register dist/server.js     # prod: needs a dist-anchored tsconfig
node dist/server.js                                # prod: works only with #imports or after rewriting
npx esbuild src/server.ts --bundle --platform=node # bundling resolves aliases inherently
```

```js
// Jest
moduleNameMapper: {
  "^(@\\w+/.*)\\.js$": "$1",                                 // strip .js FIRST
  ...pathsToModuleNameMapper(paths, { prefix: "<rootDir>/" })  // from "ts-jest"
}
// First matching regex wins → put specific patterns above catch-alls.
// After editing: npx jest --clearCache
```

```ts
// Vitest
plugins: [tsconfigPaths()]                        // vite-tsconfig-paths — easiest
resolve: { alias: { "@": fileURLToPath(new URL("./src", import.meta.url)) } }
resolve: { alias: [{ find: /^@\/(.*)\.js$/, replacement: "/abs/src/$1" }] }  // regex form
```

| Runtime option | Prod dep | Extra build step | ESM | Works with only `dist/`+`package.json` | Verdict |
|---|---|---|---|---|---|
| `package.json#imports` | none | none | ✅ | ✅ | **best for plain Node** |
| `tsc-alias` | none | ✅ post-build | ✅ | ✅ | best retrofit |
| Bundler | none | ✅ bundle | ✅ | ✅ | free if already bundling |
| `tsconfig-paths` at runtime | ✅ | none | ⚠️ loader | ❌ needs tsconfig shipped | dev only |
| `module-alias` | ✅ | none | ❌ CJS only | ✅ | legacy |
| `tsx` / `ts-node` | dev only | none | ✅ | n/a | dev only |

| Symptom | Cause | Fix |
|---|---|---|
| `Cannot find module '@/x'` at boot | tsc emitted the alias verbatim | `tsc-alias`, bundle, or switch to `#imports` |
| `ERR_UNKNOWN_FILE_EXTENSION ".ts"` | runtime alias points at `src/` | point it at `dist/` |
| Works in dev, fails in prod | `tsx` resolves aliases, `node` doesn't | fix the build, not the dev script |
| Jest: `Cannot find module '@domain/…'` | no `moduleNameMapper` | `pathsToModuleNameMapper` |
| Exact alias resolves to the wildcard | mapper regex order | specific patterns first |
| Alias resolves to the wrong package | `paths` in an extended base config | re-declare `paths` in the leaf config |
| Editor OK, `tsc` fails | different tsconfig selected by tsserver | check the "TypeScript: Select Version"/project |
| Consumers of your package get `any` | unrewritten aliases in `.d.ts` | run `tsc-alias`, gate in `prepack` |
| `MODULE_NOT_FOUND` only in Docker | tsconfig not copied into the image | use `#imports`, or COPY the tsconfig |
| Bare specifier hits `node_modules` | `baseUrl` without a `@`/`#` prefix | always prefix aliases |

---

## Connected topics

- **03 — tsconfig in depth** — `baseUrl`, `paths`, `rootDirs`, `moduleResolution`, and how `extends` re-anchors path-shaped options.
- **04 — TypeScript Node.js project setup** — where `tsx`, `tsc-alias`, and the `build`/`start` scripts in this document actually live.
- **48 — Modules in TypeScript** — `moduleResolution: node16/nodenext/bundler`, why NodeNext makes you write `.js` in `.ts` sources, and how `exports`/`imports` fit the module system.
- **49 — Declaration files** — why an unrewritten alias inside a shipped `.d.ts` silently degrades consumers' types to `any`.
- **75 — Debugging TypeScript in VS Code** — `outFiles` and source maps still have to line up after `tsc-alias` or a bundler has moved your specifiers around.
- **77 — Monorepo basics with TypeScript** — project references and workspace `exports`, which are the right tool for cross-package imports that `paths` is so often misused for.
