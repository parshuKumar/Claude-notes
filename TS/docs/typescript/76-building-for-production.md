# 76 — Building for production

## What is this?

**Building for production** is the step that turns the `.ts` files on your laptop into an artifact a machine can run with plain `node` — no TypeScript installed, no compiler in the image, no surprises at 3am.

In JavaScript there was no such step. `src/server.js` *was* the artifact. You `git clone`, `npm ci --omit=dev`, `node src/server.js`, done.

In TypeScript there is exactly one non-negotiable rule:

> **Production runs JavaScript. Always. Node cannot execute `.ts`, and the tools that pretend it can are development tools.**

So a production build is a pipeline with four separable jobs, and the entire topic is about *which tool does which job*:

```
   src/**/*.ts
        │
        ├─────────────► 1. TYPE CHECK      "is this program correct?"      tsc --noEmit
        │                                   slow, thorough, no output
        │
        ├─────────────► 2. EMIT JS         "make runnable .js"             tsc | esbuild | swc
        │                                   strips types, downlevels syntax
        │
        ├─────────────► 3. EMIT MAPS       "make crashes readable"         sourceMap: true
        │                                   dist/*.js.map
        │
        └─────────────► 4. EMIT TYPES      "let others import this"        declaration: true
                                            dist/*.d.ts  (libraries only)

   dist/**/*.js  +  package.json  +  production node_modules
        │
        ▼
   node --enable-source-maps dist/server.js
```

`tsc` can do all four. The fast tools (esbuild, swc, tsx) do **2 and 3 only** — they never do 1, and that trade-off is the single most important thing to understand in this doc.

The deliverable at the end is:

- a `dist/` directory of plain JavaScript,
- a `package.json` whose `dependencies` are genuinely enough to run it,
- a container image (or zip, or tarball) that contains **no TypeScript at all**,
- and a set of npm scripts — `dev`, `build`, `start`, `typecheck` — that make the whole thing one command each.

---

## Why does it matter?

Because every failure mode in this list is a real production incident, and every one of them comes from getting the build wrong:

- **`Cannot find module 'typescript'` at container start.** You put `ts-node` in the start script and `typescript` in `devDependencies`. Locally it works because you never ran `npm ci --omit=dev`. In the image, it doesn't exist.
- **`dist/` structure changes shape when you add one file.** You add `scripts/seed.ts` outside `src/`, and suddenly `dist/src/server.js` exists instead of `dist/server.js`, and your `CMD ["node","dist/server.js"]` breaks. That's `rootDir` inference — Concept 3.
- **A deleted file keeps running.** You renamed `userService.ts` → `usersService.ts`. `tsc` wrote the new output but never deleted the old one. Something still imports the stale `dist/userService.js` and it silently works for weeks.
- **A type error ships.** You switched to esbuild for build speed, got a 40× faster build, and quietly stopped type-checking. Six weeks later a `null` reaches production that `tsc` would have caught on day one.
- **Stack traces point at `dist/`.** `at createOrder (/app/dist/services/orderService.js:214:19)` tells you nothing. Line 214 of a file you never wrote.
- **A published package has no types.** Consumers `import { createClient } from "@acme/sdk"` and get `any`, because `types` isn't in `package.json` and `.d.ts` files aren't in `files`.
- **A 1.2 GB image for a 40 MB service.** Because the final stage kept `node_modules` with dev deps, the TypeScript compiler, the test framework, and the `.git` directory.

There is also a *speed* dimension. `tsc` on a 100k-line service takes 30–90 seconds. esbuild takes under a second. Knowing when each is correct — and how to keep type safety while using the fast one — is what makes a build pipeline both fast and trustworthy.

TypeScript's cost is a build step. This doc is how you pay that cost once, correctly, instead of paying it forever in incidents.

---

## The JavaScript way vs the TypeScript way

```js
// ── Plain JavaScript / Node — there is no build ──────────────────────────────
// package.json
{
  "main": "src/server.js",
  "scripts": { "dev": "nodemon src/server.js",
               "start": "node src/server.js" },   // ← the SAME file in dev and prod
  "dependencies": { "express": "^4.19.2" }
}
// Dockerfile:  FROM node:22-alpine / COPY package*.json ./ / RUN npm ci --omit=dev
//              COPY src ./src / CMD ["node","src/server.js"]
//
// Deploy = copy the repo. Stack traces already name the files you wrote. There is no
// dist/, no outDir, no rootDir, no .d.ts, and no "did it type-check?" question.
```

Now the naive TypeScript translation — the one that looks fine and is wrong:

```js
// ── TypeScript, done wrong — four bugs, all shipped ──────────────────────────
{
  "main": "src/server.ts",                   // ❌ 1. Node cannot load .ts
  "scripts": { "dev":   "ts-node src/server.ts",
               "start": "ts-node src/server.ts" },   // ❌ 2. compiling at boot, in prod
  "dependencies":    { "express": "^4.19.2" },
  "devDependencies": { "typescript": "^5.6.0",       // ❌ 3. needed at RUNTIME by ts-node,
                       "ts-node": "^10.9.2" }        //       installed as a DEV dep
}
// Symptoms, in the order you meet them:
//   • npm ci --omit=dev && npm start   →  sh: ts-node: not found
//   • you "fix" it by moving typescript to dependencies → +300 MB image
//   • boot takes 8 s instead of 200 ms — the compiler runs on every cold start
//   • a type error now CRASHES A RUNNING POD instead of failing a CI job   ❌ 4.
//   • RSS spikes: the compiler holds the whole program graph in RAM, in prod
```

```ts
// ── The TypeScript way — build once, ship JavaScript ─────────────────────────
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2022", "module": "NodeNext", "moduleResolution": "NodeNext",
    "rootDir": "src",           // ← input root: fixes dist/ shape       (Concept 3)
    "outDir": "dist",           // ← output root
    "sourceMap": true,          // ← readable prod stack traces          (Concept 8)
    "strict": true, "skipLibCheck": true
  },
  "include": ["src/**/*.ts"],
  "exclude": ["node_modules", "dist", "**/*.test.ts"]
}

// package.json
{
  "main": "dist/server.js",                            // ← .js, not .ts
  "scripts": {
    "dev":       "tsx watch src/server.ts",            // fast loop, NO type check
    "typecheck": "tsc --noEmit",                       // types only, NO output
    "build":     "rm -rf dist && tsc -p tsconfig.build.json",
    "start":     "node --enable-source-maps dist/server.js"   // ← plain node
  },
  "dependencies":    { "express": "^4.19.2" },         // runtime only
  "devDependencies": { "typescript": "^5.6.0", "tsx": "^4.19.0", "@types/express": "^4.17.21" }
}

// Dockerfile (multi-stage — Concept 12)
//   stage "build":   full deps → npm run typecheck && npm run build
//   stage "runtime": prod deps → COPY --from=build /app/dist ./dist
//                                CMD ["node","--enable-source-maps","dist/server.js"]
//
// Result: the runtime image contains ZERO TypeScript. Same startup profile as the
// JavaScript version. Type errors were caught in CI, hours earlier, by a step that
// emits nothing at all.
```

The revelation: **TypeScript is a build-time tool that must leave no trace at runtime.** If `typescript`, `ts-node`, `tsx`, `@types/*`, or a `.ts` file appears in your production image, your build is wrong. The goal is to arrive back at the JavaScript deployment story — a directory of `.js` and a `node` command — having spent the type safety in CI where it's cheap.

---

## Syntax

```jsonc
// ── tsconfig.json — the emit-related options, all in one place ───────────────
{
  "compilerOptions": {
    // ── WHERE things go ─────────────────────────────────────────────────────
    "rootDir": "src",             // input root. Output mirrors paths RELATIVE to this.
    "outDir": "dist",             // output root. tsc writes .js/.d.ts/.map here.
    "outFile": "bundle.js",       // ⚠️ single-file output — only for "AMD"/"System". Never for Node.

    // ── WHAT gets emitted ───────────────────────────────────────────────────
    "noEmit": true,               // type-check only, write NOTHING. The typecheck script.
    "emitDeclarationOnly": true,  // write .d.ts but no .js (esbuild/swc emits the JS)
    "declaration": true,          // emit .d.ts alongside .js — REQUIRED for libraries
    "declarationMap": true,       // emit .d.ts.map — go-to-definition lands on .ts
    "declarationDir": "dist/types", // put .d.ts somewhere other than outDir
    "sourceMap": true,            // emit .js.map — readable prod stack traces
    "inlineSourceMap": true,      // map inside the .js (mutually exclusive with sourceMap)
    "inlineSources": true,        // embed original .ts TEXT in the map (works with either)
    "removeComments": true,       // strip comments from output — small size win
    "importHelpers": true,        // import __awaiter etc. from tslib instead of inlining

    // ── HOW it's emitted ────────────────────────────────────────────────────
    "target": "ES2022",           // syntax level of the emitted JS
    "module": "NodeNext",         // module format: NodeNext | ESNext | CommonJS
    "moduleResolution": "NodeNext",
    "isolatedModules": true,      // forbid constructs esbuild/swc can't transpile file-by-file
    "verbatimModuleSyntax": true, // `import type` is erased; plain `import` is kept, verbatim

    // ── BUILD PERFORMANCE ───────────────────────────────────────────────────
    "incremental": true,          // write .tsbuildinfo, only recheck what changed
    "tsBuildInfoFile": "./node_modules/.cache/tsbuildinfo",
    "composite": true,            // project references (see 77); implies declaration+incremental
    "skipLibCheck": true          // don't type-check node_modules .d.ts — big speed win
  },
  "include": ["src/**/*.ts"],
  "exclude": ["node_modules", "dist", "**/*.test.ts", "**/*.spec.ts"]
}
```

```bash
# ── tsc CLI — the flags that matter for building ─────────────────────────────
tsc                                 # build using ./tsconfig.json
tsc -p tsconfig.build.json          # build using a specific config  (-p = --project)
tsc --noEmit                        # TYPE CHECK ONLY — the CI gate. Exit code 1 on error.
tsc --watch                         # rebuild on change  (-w)
tsc --incremental                   # reuse .tsbuildinfo across runs
tsc --build                         # project-references build (see 77)  (-b)
tsc --build --clean                 # delete outputs produced by --build
tsc --build --force                 # ignore up-to-date checks, rebuild everything
tsc --listEmittedFiles              # print every file written — great for debugging outDir
tsc --listFiles                     # print every file READ — find what dragged in node_modules
tsc --showConfig                    # print the fully-resolved config after `extends` merging
tsc --diagnostics                   # timing + memory; --extendedDiagnostics for more
tsc --generateTrace ./trace         # perf trace for genuinely slow builds
```

```bash
# ── The four npm scripts, canonical form ─────────────────────────────────────
npm run dev        # tsx watch src/server.ts     fast, transpile-only, NO type check
npm run typecheck  # tsc --noEmit                types only, no output, CI gate
npm run build      # rm -rf dist && tsc -p tsconfig.build.json
npm start          # node --enable-source-maps dist/server.js

# ── Transpile-only builders (fast, NO type checking — always gate with tsc) ──
esbuild src/server.ts --bundle --platform=node --target=node22 --outfile=dist/server.js --sourcemap
swc src -d dist --source-maps
tsx src/server.ts                   # dev runner; esbuild under the hood
```

---

## How it works — concept by concept

### Concept 1 — What `tsc` actually does when it emits

`tsc` is not a bundler and not a minifier. Given a `.ts` file it performs exactly three transformations:

```ts
// ── INPUT: src/services/orderService.ts ─────────────────────────────────────
import type { Logger } from "../types/logger.js";   // type-only import
import { db } from "../db/client.js";               // value import

export interface Order {                            // type — erased entirely
  orderId: number;
  totalCents: number;
}

export async function findOrder(orderId: number, log: Logger): Promise<Order | null> {
  log.debug("finding order", { orderId });
  const row = await db.query<Order>("select * from orders where id = $1", [orderId]);
  return row ?? null;
}
```

```js
// ── OUTPUT: dist/services/orderService.js  (target ES2022, module NodeNext) ─
import { db } from "../db/client.js";   // 1. TYPE ERASURE: `import type` line is GONE,
                                        //    `interface Order` is GONE, `: number` is GONE.
export async function findOrder(orderId, log) {
    log.debug("finding order", { orderId });
    const row = await db.query("select * from orders where id = $1", [orderId]);
    return row ?? null;
}
//# sourceMappingURL=orderService.js.map   // 3. SOURCE MAP link
```

The three transformations:

```
1. TYPE ERASURE      — annotations, interfaces, type aliases, `import type`, generics,
                       `as` casts, `satisfies`, `!`: all deleted. Never changes behaviour.
2. SYNTAX DOWNLEVEL  — features newer than `target` are rewritten. ES2022 on modern Node:
                       almost nothing. ES5: async→__awaiter state machine, classes→functions.
3. MODULE TRANSFORM  — import/export rewritten to match `module`.
                       NodeNext + "type":"module" → stays ESM;  CommonJS → require()/exports.x
```

Plus the small set of constructs that actually *generate* runtime code rather than being erased: `enum` (a real object), parameter properties (`constructor(private db: Db)` → `this.db = db`), decorators (`__decorate()` calls), and `namespace` (an IIFE).

Everything else is deletion. That's *why* esbuild and swc can be 40× faster: for a modern `target`, transpiling TypeScript is mostly "parse, delete, print", and none of it requires understanding the whole program.

### Concept 2 — `outDir`: where the output goes

`outDir` is simple on its own: the root directory `tsc` writes into.

```jsonc
{ "compilerOptions": { "outDir": "dist" }, "include": ["src/**/*.ts"] }
//   src/server.ts               →  dist/server.js
//   src/routes/userRoutes.ts    →  dist/routes/userRoutes.js
```

Notes that bite people:

```jsonc
// `outDir` is relative to the tsconfig.json FILE, not to your shell's cwd.
// So a tsconfig at config/tsconfig.build.json with "outDir": "dist"
// writes to config/dist/ — almost never what you want. Use "../dist".

// If you OMIT outDir, tsc writes each .js NEXT TO its .ts:
//   src/server.ts  →  src/server.js      ← pollutes your source tree
//   Then your editor shows both, git shows both, and `import "./server"`
//   becomes ambiguous. Always set outDir for a Node service.

// ALWAYS exclude the outDir from `include`, or tsc will try to compile its
// own output on the second run:
{ "include": ["src/**/*.ts"], "exclude": ["dist", "node_modules"] }
```

### Concept 3 — `rootDir`, and exactly how it surprises you

This is the single most common "why is my build weird?" question in TypeScript.

**`rootDir` is the *input* root. `tsc` computes each output path as `outDir + (inputPath relative to rootDir)`.**

If you don't set `rootDir`, TypeScript **infers** it as the **longest common directory prefix of all input files**. That inferred value changes whenever your file set changes — which is why the shape of `dist/` mutates without you touching any config.

```
── Scenario A: only src/ is compiled ────────────────────────────────────────
Inputs:  src/server.ts, src/routes/userRoutes.ts
Inferred rootDir = "src"       (longest common prefix)

  src/server.ts             →  dist/server.js
  src/routes/userRoutes.ts  →  dist/routes/userRoutes.js

package.json: "main": "dist/server.js"      ✅ works


── Scenario B: you add ONE file outside src/ ────────────────────────────────
Inputs:  src/server.ts, src/routes/userRoutes.ts, scripts/seed.ts
Inferred rootDir = "."         (the common prefix moved UP one level!)

  src/server.ts             →  dist/src/server.js         ← EXTRA "src/" segment
  src/routes/userRoutes.ts  →  dist/src/routes/userRoutes.js
  scripts/seed.ts           →  dist/scripts/seed.js

package.json: "main": "dist/server.js"      ❌ Error: Cannot find module '/app/dist/server.js'
Dockerfile:   CMD ["node","dist/server.js"] ❌ container crash-loops on deploy
```

Nothing about `outDir` changed. You added an unrelated seed script and your entry point moved.

Other ways the same thing sneaks in:

```jsonc
// ❌ Importing a file above rootDir — a shared package in a monorepo
// src/server.ts:  import { Money } from "../../packages/shared/src/money.js";
//   rootDir SET   → error TS6059: File '/repo/packages/shared/src/money.ts' is not
//                   under 'rootDir' '/repo/src'.        ← loud, at build time ✅
//   rootDir UNSET → tsc infers "/repo": dist/src/server.js + dist/packages/shared/…
//                   The build "succeeds" and the deploy breaks.               ❌

// ❌ A test file outside src/ pulled in by `include`     → rootDir "." → dist/src/…
// ❌ "include": ["**/*.ts"] catching jest.config.ts / vitest.config.ts at the repo root
```

The fix is always the same: **set `rootDir` explicitly and let it fail loudly.**

```jsonc
// ✅ Explicit rootDir turns a silent layout change into a compile error.
{
  "compilerOptions": {
    "rootDir": "src",     // I assert: every input lives under src/
    "outDir": "dist"      // therefore dist/ mirrors src/ exactly
  },
  "include": ["src/**/*.ts"],
  "exclude": ["node_modules", "dist", "**/*.test.ts"]
}
// Now dist/server.js is dist/server.js forever. Adding scripts/seed.ts produces
// TS6059 instead of quietly restructuring your output.
```

Note that `rootDirs` (plural) is a *different* feature: it makes several directories look like one for module **resolution** (generated-code overlays) and does not change emit layout. And verify rather than guess — `tsc --listEmittedFiles` prints every output path, `tsc --showConfig` prints `rootDir` after all `extends` merging.

Mental model to keep:

```
outputPath = outDir + relative(rootDir, inputPath)

Change rootDir → every output path shifts.
Don't set rootDir → tsc picks it for you, and re-picks it every build.
```

### Concept 4 — Clean builds, and why stale output is dangerous

`tsc` is **additive**. It writes files. It never deletes files it didn't just write.

```bash
# Monday
src/services/userService.ts   →  dist/services/userService.js       ✅

# Tuesday: you rename the file
git mv src/services/userService.ts src/services/usersService.ts
npm run build

# dist/ now contains BOTH:
dist/services/userService.js     ← ORPHAN. Monday's code. Still importable.
dist/services/usersService.js    ← today's code.
```

Why this is genuinely dangerous, not just untidy:

```
1. STALE IMPORTS KEEP RESOLVING. If a file was deleted from src/ and left in dist/,
   the runtime graph inside dist/ is self-consistent and RUNS. Old code in prod.
2. DELETED ROUTES STAY REGISTERED. dist/routes/legacyAdminRoutes.js survives a delete;
   a glob-based route loader (readdirSync("dist/routes")) re-registers a route you
   believed you removed — auth assumptions and all. That is a security bug.
3. CONFIG DRIFT. Change `target` ES2022→ES2020 with `incremental` on and only CHANGED
   files are re-emitted. dist/ is now a mix of two targets.
4. "WORKS LOCALLY, FAILS IN CI." Your dist/ is a six-month accretion; CI's is built
   from scratch. They are different programs. CI is the honest one.
```

The rule: **`build` always starts by deleting `outDir`.**

```jsonc
{
  "scripts": {
    "build":     "rimraf dist && tsc -p tsconfig.build.json",  // cross-platform (npm i -D rimraf)
    "clean":     "node -e \"require('fs').rmSync('dist',{recursive:true,force:true})\"", // no dep
    "build:sh":  "rm -rf dist && tsc -p tsconfig.build.json"   // POSIX only
  }
}
// With project references (77), tsc can clean itself — but ONLY for composite projects:
//   tsc --build --clean     deletes outputs tracked in .tsbuildinfo
//   tsc --build --force     ignore up-to-date checks
// Plain `tsc -p tsconfig.json` has no clean mode at all.
```

And the incremental-build gotcha:

```jsonc
// `incremental: true` writes tsconfig.tsbuildinfo. If you delete dist/ but NOT
// the .tsbuildinfo, tsc thinks everything is up to date and emits NOTHING —
// you end up with an empty dist/ and a successful exit code.
{ "compilerOptions": { "incremental": true, "tsBuildInfoFile": "./node_modules/.cache/tsbuildinfo" } }
// Putting it under node_modules/.cache means `rimraf dist` can't desync it,
// and it never gets committed.
```

### Concept 5 — Why you must not run ts-node/tsx in production

`ts-node` and `tsx` are **development runtimes**. They hook Node's module loader and compile each `.ts` file the moment it is first `require`d/`import`ed. That's a wonderful dev loop and a bad production strategy, for six independent reasons:

```
1. STARTUP LATENCY — compilation happens on every cold start, not once at build time.
   node dist/ ≈ 150–400 ms │ tsx src/ ≈ 1.5–4 s │ ts-node (type-checking) ≈ 5–15 s
   On Lambda/Cloud Run/K8s scale-up that's user-visible latency on every new instance,
   paid again on every restart, crash, and scale event.

2. MEMORY — the compiler holds the program graph, AST, and checker in RAM for the life
   of the process. 150–400 MB of RSS doing nothing after boot. In a 512 MB container
   that is the difference between running and OOMKilled.

3. TYPESCRIPT BECOMES A RUNTIME DEPENDENCY — either move it to `dependencies`
   (+60 MB image, dev tooling in prod) or leave it in dev and `npm ci --omit=dev`
   produces a broken image. Both are bad; there is no third option.

4. TYPE ERRORS BECOME RUNTIME CRASHES — ts-node (without transpileOnly) checks at load.
   An error CI should have caught takes down a pod, possibly hours later on a rare path.

5. LARGER ATTACK SURFACE — a compiler in the image means arbitrary code generation is
   one file-write away. Prod images should hold the minimum that can run.

6. NON-DETERMINISM — each instance compiles independently, at its own time, against
   whatever tsconfig is on disk. Build once; run the same bytes everywhere.
```

The honest comparison:

| | `tsx`/`ts-node` in prod | Prebuilt `dist/` |
|---|---|---|
| Cold start | 1.5–15 s | 0.15–0.4 s |
| RSS overhead | +150–400 MB | +0 MB |
| `typescript` in image | required | absent |
| Image size | +60–120 MB | baseline |
| Type errors surface | at runtime, per-instance | in CI, once |
| Reproducible | no | yes |

The one legitimate exception, and it's narrow:

```bash
# A short-lived operational script run manually, where you accept the cost:
npx tsx scripts/backfillUserEmails.ts --dry-run
# ...but even here, prefer building it: node dist/scripts/backfillUserEmails.js
# so the script you run is the same code path CI type-checked.
```

### Concept 6 — `tsc --noEmit` as a separate typecheck step

The insight that unlocks fast builds: **type checking and emitting are independent jobs, and you can give them to different tools.**

```bash
tsc --noEmit      # runs the FULL type checker. Writes nothing. Exit 0 or 1.
```

```jsonc
// package.json
{
  "scripts": {
    "typecheck":       "tsc --noEmit -p tsconfig.json",
    "typecheck:watch": "tsc --noEmit --watch --preserveWatchOutput",
    "build":           "rimraf dist && tsc -p tsconfig.build.json"
  }
}
```

Why the separation earns its keep:

```
• The dev loop (tsx watch) does not type-check at all — you get instant reloads.
  `typecheck:watch` runs in a SECOND terminal and reports errors continuously.
  Fast feedback and full correctness, at the same time, without coupling them.

• CI can run typecheck, lint, and test IN PARALLEL. Nothing depends on emit.
  Serial:   typecheck → build → test        = 90 + 60 + 120 = 270 s
  Parallel: (typecheck | lint | test) then build = max(90,20,120) + 60 = 180 s

• The emit tool becomes swappable. Once typecheck is its own gate, you can emit
  with esbuild or swc without losing any safety (Concept 8).

• `--noEmit` type-checks files that are NEVER emitted: tests, config files,
  scripts, .d.ts. Your build config excludes them; your typecheck config shouldn't.
```

Which is why the two configs differ — one wide, one narrow:

```jsonc
// ── tsconfig.json — EDITOR + TYPECHECK. Sees everything. ────────────────────
{
  "compilerOptions": { /* target, module, rootDir, outDir, strict, sourceMap, … */ },
  // WIDE: tests, scripts, and config files are all type-checked (never emitted).
  "include": ["src/**/*.ts", "tests/**/*.ts", "scripts/**/*.ts", "*.config.ts"]
}

// ── tsconfig.build.json — EMIT. Sees only shippable source. ─────────────────
{
  "extends": "./tsconfig.json",     // inherit strictness — never re-specify it
  "compilerOptions": {
    "noEmit": false,                // in case the base sets it
    "declaration": false,           // a SERVICE doesn't need .d.ts (Concept 9)
    "removeComments": true,
    "incremental": false            // CI builds cold; no cache to reuse
  },
  "include": ["src/**/*.ts"],       // NARROW
  "exclude": ["node_modules", "dist", "**/*.test.ts", "**/*.spec.ts",
              "src/**/__tests__/**", "src/**/__mocks__/**", "src/testUtils/**"]
}
```

```
⚠️  The classic bug this prevents: test files landing in dist/.
    Without the exclude, `dist/services/orderService.test.js` ships to production,
    dragging `jest`/`vitest`/`supertest` imports with it. Those are devDependencies,
    so if anything ever loads that file: MODULE_NOT_FOUND in prod.
    Worse, test fixtures with seeded credentials end up inside your image.
```

### Concept 7 — Transpile-only tools, and the trade-off you're accepting

esbuild, swc, and tsx are 20–100× faster than `tsc`. They achieve that by **never building a type graph.** They parse one file, delete the type syntax, print JavaScript. They never open the files you import.

```
tsc:      read ALL files → build symbol table → resolve types across the graph
          → check every expression → emit
          ~30–90 s on 100k lines. Knows everything. Can be wrong about nothing.

esbuild:  read ONE file → parse → delete type syntax → print
          ~0.3–1 s on 100k lines. Knows nothing beyond the current file.
```

The consequence, stated plainly:

```ts
// ── src/services/paymentService.ts ──────────────────────────────────────────
export function chargeCard(amountCents: number, token: string): Promise<Receipt> { /* … */ }

// ── src/routes/checkout.ts ──────────────────────────────────────────────────
import { chargeCard } from "../services/paymentService.js";

// Wrong argument order AND a wrong type:
const receipt = await chargeCard(cardToken, orderTotalCents);
//                               ^^^^^^^^^ string where number expected
//
// tsc:      error TS2345: Argument of type 'string' is not assignable to
//                         parameter of type 'number'.        → build FAILS ✅
// esbuild:  compiles happily in 40 ms. Ships. Charges a customer NaN cents
//           or throws deep inside the payment SDK at 2am.    → build PASSES ❌
```

So the correct architecture is **two tools, two jobs, both mandatory**:

```jsonc
{
  "scripts": {
    "typecheck": "tsc --noEmit",                                  // CORRECTNESS  (slow)
    "build":     "npm run typecheck && npm run build:js",         // gate, then emit
    "build:js":  "esbuild src/server.ts --bundle --platform=node --target=node22 --outfile=dist/server.js --sourcemap --minify=false",
    "ci":        "npm run typecheck && npm run lint && npm test && npm run build:js"
  }
}
```

```
NEVER do this:   "build": "esbuild ..."          ← nothing type-checks. Ever.
ALWAYS do this:  "build": "tsc --noEmit && esbuild ..."
```

Beyond "no type checking", transpile-only tools have concrete limitations. This is what `isolatedModules: true` is for — it makes `tsc` reject the constructs a per-file transpiler cannot handle:

```ts
// ── 1. Re-exporting a type without `export type` ───────────────────────────
export { Order } from "./types.js";        // ❌ esbuild keeps it → runtime re-export of a
                                           //    type → "does not provide an export"
export type { Order } from "./types.js";   // ✅ erasable file-locally

// ── 2. `const enum` — tsc INLINES it using cross-file knowledge esbuild lacks ─
const enum LogLevel { Debug = 0 }                              // ❌
const LogLevel = { Debug: 0, Info: 1 } as const;               // ✅ object + union
type LogLevel = (typeof LogLevel)[keyof typeof LogLevel];

// ── 3. `namespace` merging across files — needs whole-program knowledge (see 70) ❌
// ── 4. `emitDecoratorMetadata` — emitting design:paramtypes requires the TYPE of
//       each parameter, which a per-file transpiler never computes. Breaks NestJS DI,
//       TypeORM column inference, class-validator. swc supports it; esbuild does NOT.
// ── 5. No .d.ts output. For a LIBRARY, pair them:
//       "build": "tsc --emitDeclarationOnly -p tsconfig.build.json && esbuild ..."
```

```jsonc
// Turn all of the above into compile errors instead of surprises:
{
  "compilerOptions": {
    "isolatedModules": true,      // each file must be transpilable in isolation
    "verbatimModuleSyntax": true  // import/export elision is explicit, not inferred
  }
}
```

Choosing:

| Situation | Emit with |
|---|---|
| Service, build speed not a problem | `tsc` — one tool, no caveats |
| Service, slow build hurting CI | `tsc --noEmit` + `esbuild` or `swc` |
| NestJS / TypeORM / `emitDecoratorMetadata` | `tsc` or `swc` (never esbuild) |
| Library published to npm | `tsc --emitDeclarationOnly` + `esbuild`, or plain `tsc` |
| Lambda / single-file artifact | `esbuild --bundle` (+ `tsc --noEmit` gate) |
| Monorepo with project references | `tsc --build` (see 77) |

### Concept 8 — Source maps in production

A build that emits maps does **not** mean a runtime that uses them. Two separate switches.

```jsonc
// SWITCH 1 — build time: produce the maps
{ "compilerOptions": { "sourceMap": true } }        // → dist/**/*.js.map
```

```bash
# SWITCH 2 — run time: tell Node to consult them
node --enable-source-maps dist/server.js
# or
NODE_OPTIONS="--enable-source-maps" node dist/server.js
```

```
Without switch 2:
  Error: order not found
      at findOrder (/app/dist/services/orderService.js:214:19)     ❌ meaningless
      at /app/dist/routes/checkout.js:88:22

With both:
  Error: order not found
      at findOrder (/app/src/services/orderService.ts:47:11)       ✅ actionable
      at handleCheckout (/app/src/routes/checkout.ts:31:18)
```

`--enable-source-maps` is built into Node (stable since v12.12, no flag needed since v20). It is **lazy** — maps are parsed on the first thrown error, so the cost is ~10–50 ms once, not on the hot path. Use it. There is no good reason not to.

`source-map-support` is the pre-`--enable-source-maps` npm package. Still useful when you can't control the node argv (some PaaS/serverless wrappers), when you need programmatic access to the mapping, or on runtimes older than Node 12.12:

```ts
// ── src/bootstrap/sourceMaps.ts ─────────────────────────────────────────────
//   npm i source-map-support         ← `dependencies`, NOT dev: dist/ imports it
//   npm i -D @types/source-map-support
import "source-map-support/register.js";   // MUST be the first import in the entry file:
                                           // it patches Error.prepareStackTrace, and only
                                           // stacks captured AFTER the patch get remapped.
Error.stackTraceLimit = 50;                // default 10 truncates async chains
```

The three requirements — miss any one and it silently no-ops:

```
1. sourceMap: true (or inlineSourceMap) at BUILD time
2. the .js.map files present next to the .js IN THE IMAGE
3. the original .ts reachable, OR inlineSources: true so the map carries the text
```

Requirement 2 is where multi-stage Docker builds break it:

```dockerfile
COPY --from=build /app/dist/*.js ./dist/     # ❌ .map files silently excluded
COPY --from=build /app/dist ./dist           # ✅ maps included
```

Requirement 3 is where slim images break it. Your final stage copies `dist/` but not `src/`, so the map's `sources` entries (`"../../src/services/orderService.ts"`) point at nothing. You still get correct **file names and line numbers** — Node only needs `src/` to print the source *line text*. If you want that too:

```jsonc
{ "compilerOptions": { "sourceMap": true, "inlineSources": true } }
// Embeds the original .ts text in the .map. Your source is now inside the artifact —
// fine for a private service, think twice for a published package or browser bundle.
```

Cost, and the recommended default:

```
sourceMap: true                   → ~+5–15% build time, dist/ ~+30–60% on disk
inlineSourceMap + inlineSources   → dist/ can be 2–4× larger; matters on Lambda
--enable-source-maps at runtime   → ~0 until the first error, then 10–50 ms once

DEFAULT FOR A BACKEND SERVICE:
  tsconfig.build.json  { "sourceMap": true, "inlineSources": false }
  package.json         "start": "node --enable-source-maps dist/server.js"
  Dockerfile           COPY --from=build /app/dist ./dist
```

### Concept 9 — Declaration output: library vs service

`.d.ts` files are the type interface of your compiled JavaScript. Whether you need them is decided by one question:

> **Will anything `import` this package's code as a dependency?**

```jsonc
// ── A SERVICE (API, worker, cron, Lambda) — nothing imports it ──────────────
{ "compilerOptions": {
    "declaration": false,        // ← no .d.ts. Nobody consumes your types. Emitting
    "declarationMap": false,     //   them costs 20–40% of build time for files that
    "sourceMap": true,           //   will never be read.
    "rootDir": "src", "outDir": "dist" } }

// ── A LIBRARY (published to npm, or a shared workspace package) ─────────────
{ "compilerOptions": {
    "declaration": true,         // ← MANDATORY. Without it consumers get `any`.
    "declarationMap": true,      // ← .d.ts.map: go-to-definition lands on YOUR .ts
    "sourceMap": true,
    "stripInternal": true,       // omit members marked /** @internal */ from .d.ts
    "rootDir": "src", "outDir": "dist" } }
```

```
dist/
  index.js         ← the runtime code
  index.js.map     ← maps .js → .ts   (for stack traces / debugging)
  index.d.ts       ← the type surface  (for the consumer's tsc)
  index.d.ts.map   ← maps .d.ts → .ts  (for the consumer's go-to-definition)
```

With `declarationMap` **plus** shipping `src/`, a consumer who hits F12 on `createClient` lands on your actual TypeScript source with comments, not a stripped `.d.ts` signature. That is a genuinely large DX difference for an internal SDK.

Two library-specific traps:

```ts
// ── TRAP 1: TS2742 "The inferred type of X cannot be named without a reference to Y"
//    A public return type references a type from a devDependency, or an unexported one.
import type { Pool } from "pg";          // pg is a devDependency
export function makeRepo(pool: Pool) {   // ❌ the INFERRED return type mentions Pool,
  return { findUser: (id: number) => pool.query("...") };   //  which .d.ts cannot name
}
// ✅ Fix A — annotate the return type so nothing needs inferring:
export interface UserRepo { findUser(id: number): Promise<User | null>; }
export function makeRepo(pool: Pool): UserRepo { /* … */ }
// ✅ Fix B — move the package to `dependencies`/`peerDependencies` so the consumer
//            can resolve the referenced types.

// ── TRAP 2: @types/* in devDependencies leaking into your PUBLIC .d.ts ──────
// dist/index.d.ts says  import { Request } from "express";  but @types/express is a
// devDependency → the CONSUMER gets TS2688: Cannot find type definition file for 'express'.
// ✅ If a type is in your public surface, its @types package belongs in `dependencies`
//    (or the package must be a peerDependency the consumer installs themselves).
```

For published packages, `@microsoft/api-extractor` (or `rollup-plugin-dts`, `tsup --dts`) collapses `dist/**/*.d.ts` into a single `dist/index.d.ts` and can report accidental breaking API changes between versions.

### Concept 10 — `package.json`: `main`, `module`, `exports`, `types`, `files`

These five fields tell Node and the TypeScript compiler what your package *is*. For a service, most are decorative. For a library, getting them wrong means "it doesn't work in ESM", "it has no types", or "you shipped your tests".

```jsonc
// ── SERVICE — minimal and honest ────────────────────────────────────────────
{
  "name": "@acme/api", "version": "1.0.0",
  "private": true,              // ← guards against an accidental `npm publish`
  "type": "module",             // .js files are ESM (matches module: NodeNext + ESM)
  "main": "dist/server.js",     // mostly documentation; nothing imports a service
  "engines": { "node": ">=22" },// npm warns (errors under engine-strict) on mismatch
  "scripts": { "start": "node --enable-source-maps dist/server.js" }
}
// No `exports`, no `types`, no `files` — nothing consumes this package.
```

```jsonc
// ── LIBRARY — the full, modern set ──────────────────────────────────────────
{
  "name": "@acme/sdk", "version": "2.3.1", "type": "module",

  // "exports" is the modern, AUTHORITATIVE field. Node >= 12.7 prefers it, and it
  // ENCAPSULATES: paths not listed here cannot be imported at all —
  // `import "@acme/sdk/dist/internal/secrets.js"` → ERR_PACKAGE_PATH_NOT_EXPORTED
  "exports": {
    ".": {
      "types":   "./dist/index.d.ts",  // ⚠️ MUST BE FIRST — resolution is order-sensitive
      "import":  "./dist/index.js",    // ESM consumers
      "require": "./dist/index.cjs",   // CJS consumers
      "default": "./dist/index.js"
    },
    "./errors": { "types": "./dist/errors.d.ts", "import": "./dist/errors.js" },
    "./package.json": "./package.json" // some tooling reads it; allow it explicitly
  },

  // LEGACY FALLBACKS — for bundlers/tools that ignore "exports":
  "main":   "./dist/index.cjs",   // CommonJS entry — the oldest, most universal field
  "module": "./dist/index.js",    // ESM entry. NOT a Node field: a bundler convention
                                  // (webpack/rollup/vite). Node ignores it entirely.
  "types":  "./dist/index.d.ts",  // root type entry, for older TS resolution modes

  // PUBLISH ALLOWLIST. Without it npm ships almost your whole working directory —
  // src/, tests/, .env.example, coverage/, fixtures with real-looking data.
  "files": ["dist", "src", "README.md", "LICENSE"],  // src/ so declarationMap resolves
  // (package.json/README/LICENSE are always included; .git and node_modules never are.)

  "sideEffects": false,           // tells bundlers unused exports are safe to drop
  "dependencies":     { "zod": "^3.23.8" },
  "peerDependencies": { "pg": "^8.11.0" },
  "devDependencies":  { "typescript": "^5.6.0", "@types/pg": "^8.11.6" },
  "engines": { "node": ">=20" }
}
```

The resolution order that actually matters, and the one rule people get wrong:

```
A modern TS consumer with "moduleResolution": "NodeNext" or "Bundler" resolves:
  1. "exports"  →  the "types" condition inside the matching entry   ← authoritative
  2. "types" / "typings" at the top level                            ← fallback
  3. dist/index.d.ts sitting next to "main"                          ← last resort

⚠️  RULE: inside "exports", "types" must come FIRST in the object.
    Conditions are matched TOP TO BOTTOM and the first match wins.
    Put "import" before "types" and TypeScript resolves index.js, finds no types,
    and your consumers get implicit `any` — with no error to tell them why.
```

Verify instead of hoping:

```bash
npm pack --dry-run                  # lists exactly what would be published
npx publint                         # lints main/module/exports/types consistency
npx @arethetypeswrong/cli --pack .  # checks types resolve in ESM, CJS, bundler, node10
```

### Concept 11 — `dependencies` vs `devDependencies`, and `npm ci --omit=dev`

The rule sounds trivial:

> **`dependencies`** = needed when the built code *runs*.
> **`devDependencies`** = needed only to *build*, *test*, or *lint*.

It breaks constantly because **your laptop always has both installed**, so the mistake is invisible until the production image starts.

```bash
npm ci                 # installs dependencies + devDependencies  (dev / CI build stage)
npm ci --omit=dev      # installs dependencies ONLY               (production stage)
                       # (older, still-working spelling: npm ci --production)
```

The operational test: **grep `dist/` for the import. If it's there, it's a `dependency`.**

```jsonc
{
  "dependencies": {                 // dist/*.js imports each of these at runtime
    "express": "^4.19.2", "pg": "^8.12.0", "pino": "^9.3.2",
    "zod": "^3.23.8",               // schemas RUN at runtime — not a type-only lib
    "source-map-support": "^0.5.21",// imported by dist/bootstrap/sourceMaps.js
    "tslib": "^2.6.3",              // ONLY if importHelpers: true — dist/ requires it
    "reflect-metadata": "^0.2.2"    // imported by a NestJS/TypeORM entry at runtime
  },
  "devDependencies": {              // nothing in dist/ ever imports these
    "typescript": "^5.6.0", "tsx": "^4.19.0",     // compile / dev-run only
    "@types/express": "^4.17.21", "@types/node": "^22.5.0",  // ERASED at compile time
    "vitest": "^2.0.5", "eslint": "^9.9.0",       // tests are excluded from dist/
    "rimraf": "^6.0.1", "esbuild": "^0.23.1"      // used by build scripts only
  }
}
```

The four classic mistakes:

```jsonc
// ❌ MISTAKE 1 — a runtime package parked in devDependencies.
//    Happens with `npm i -D` muscle memory, or with something you first added
//    "just for a script" and later imported from src/.
"devDependencies": { "dotenv": "^16.4.5" }
// dist/config.js does `import "dotenv/config"`.
// npm ci --omit=dev → Error [ERR_MODULE_NOT_FOUND]: Cannot find package 'dotenv'
// ✅ move to dependencies.

// ❌ MISTAKE 2 — @types/* in dependencies "to be safe".
"dependencies": { "@types/express": "^4.17.21" }
// Types are erased at build time; dist/ never imports them. This is pure image
// bloat (a few MB per package, dozens across a project) and slows every install.
// ✅ @types/* belong in devDependencies — UNLESS you are a LIBRARY whose public
//    .d.ts references them (Concept 9, Trap 2).

// ❌ MISTAKE 3 — typescript/ts-node in dependencies to "fix" a broken start script.
"dependencies": { "typescript": "^5.6.0", "ts-node": "^10.9.2" },
"scripts": { "start": "ts-node src/server.ts" }
// This makes the error go away and every problem in Concept 5 permanent.
// ✅ build to dist/ and `start: node dist/server.js`. Both packages go back to dev.

// ❌ MISTAKE 4 — importHelpers: true without tslib in dependencies.
{ "compilerOptions": { "importHelpers": true } }
// dist/*.js now contains `import { __awaiter } from "tslib"` — a REAL runtime import.
// ✅ "dependencies": { "tslib": "^2.6.3" }
```

The single check that catches all four, before deploy:

```bash
# ── scripts/verify-prod-deps.sh — run in CI right after `npm run build` ─────
set -euo pipefail
rm -rf /tmp/prodcheck && mkdir -p /tmp/prodcheck
cp -r dist package.json package-lock.json /tmp/prodcheck/ && cd /tmp/prodcheck
npm ci --omit=dev                                        # ONLY production deps
node --enable-source-maps dist/server.js --smoke-test    # must boot and exit 0
# A devDependency in the runtime path fails HERE, in CI, not in production.

# Adjuncts:
npx depcheck                    # unused deps, and imports with no declared dep
npm ls --omit=dev --all         # the exact production tree
npm ls typescript --omit=dev    # must print "(empty)"
```

### Concept 12 — Multi-stage Docker for a TypeScript Node service

A multi-stage build separates the *machine that compiles* from the *machine that runs*. The compiler, `@types`, tests, and source never enter the shipped image.

Four stages, each with one job. (Example 2 shows the fully hardened version; this is the shape.)

```dockerfile
# ── STAGE 1 — deps: ALL dependencies. Manifests copied FIRST so Docker caches
#    the whole install whenever package.json/lock are unchanged. ─────────────
FROM node:22-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
# `npm ci` (not install): installs the lockfile EXACTLY, wipes node_modules, and
# fails if lock and package.json disagree. --mount=type=cache keeps npm warm.
RUN --mount=type=cache,target=/root/.npm npm ci

# ── STAGE 2 — build: typecheck, then emit dist/ ────────────────────────────
FROM node:22-alpine AS build
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY package.json package-lock.json tsconfig.json tsconfig.build.json ./
COPY src ./src
RUN npm run typecheck        # separate RUN lines: a red build names the real gate
RUN npm run build

# ── STAGE 3 — prod-deps: node_modules with NO dev deps.
#    Its own stage rather than `npm prune`, because pruned files survive in the
#    layer history of the stage we'd otherwise copy from. ────────────────────
FROM node:22-alpine AS prod-deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN --mount=type=cache,target=/root/.npm npm ci --omit=dev && npm cache clean --force

# ── STAGE 4 — runtime: the ONLY stage that ships. Nothing here can compile. ─
FROM node:22-alpine AS runtime
WORKDIR /app
ENV NODE_ENV=production \
    NODE_OPTIONS="--enable-source-maps"   # applies to this process AND its children
# dumb-init as PID 1: reaps zombies and forwards SIGTERM so rolling updates drain
# instead of being SIGKILLed after the grace period.
RUN apk add --no-cache dumb-init
USER node                                  # node:*-alpine ships uid 1000. Never root.
# Dependencies change less often than code → copy them first for layer reuse.
COPY --from=prod-deps --chown=node:node /app/node_modules ./node_modules
COPY --from=build     --chown=node:node /app/dist         ./dist
COPY --chown=node:node package.json ./
# ⚠️ The WHOLE dist directory. `COPY .../dist/*.js` drops the .map files and your
#    production stack traces silently revert to dist/*.js coordinates.
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
  CMD node -e "fetch('http://127.0.0.1:3000/healthz').then(r=>process.exit(r.ok?0:1)).catch(()=>process.exit(1))"
ENTRYPOINT ["dumb-init", "--"]
CMD ["node", "dist/server.js"]

# NOT in the final image: typescript, tsx, ts-node, eslint, vitest, @types/*,
# src/**/*.ts, tsconfig*.json, tests, .git, .env.
# Typical result ~150 MB, vs ~1.1 GB for a naive single-stage build.
```

```dockerignore
# ── .dockerignore — as important as the Dockerfile ──────────────────────────
# Without this, `COPY . .` ships your local node_modules (wrong architecture,
# dev deps, possibly a stale build) and your .git history and .env secrets.
node_modules
dist
.git
.github
.env
.env.*
*.log
coverage
.vscode
.idea
Dockerfile
.dockerignore
README.md
**/*.test.ts
**/*.spec.ts
```

```yaml
# ── docker-compose.yml — dev and prod from ONE Dockerfile, via `target` ─────
services:
  api:
    build: { context: ., target: runtime }   # or target: build for a dev container
    environment: { NODE_ENV: production, NODE_OPTIONS: "--enable-source-maps" }
    ports: ["3000:3000"]
    stop_grace_period: 30s      # let Node finish in-flight requests on SIGTERM
```

### Concept 13 — The four scripts, and what each one is for

Every TypeScript service should expose the same four verbs. Consistency here means any developer, any CI job, and any runbook works on any repo.

| Script | Command | Type-checks? | Emits? | Used by |
|---|---|---|---|---|
| `dev` | `tsx watch src/server.ts` | ❌ no | in-memory | you, all day |
| `typecheck` | `tsc --noEmit` | ✅ full | ❌ nothing | CI gate, 2nd terminal, pre-commit |
| `build` | `rimraf dist && tsc -p tsconfig.build.json` | ✅ full | `dist/` | CI, Docker build stage |
| `start` | `node --enable-source-maps dist/server.js` | ❌ n/a | ❌ | production, `docker CMD` |

```jsonc
{
  "scripts": {
    "dev":           "tsx watch --clear-screen=false src/server.ts",
    "dev:typecheck": "tsc --noEmit --watch --preserveWatchOutput",  // SECOND terminal
    "typecheck":     "tsc --noEmit -p tsconfig.json",
    "lint":          "eslint . --max-warnings=0",
    "test":          "vitest run",
    "verify":        "npm run typecheck && npm run lint && npm run test",
    "clean":         "rimraf dist *.tsbuildinfo",
    "build":         "npm run clean && tsc -p tsconfig.build.json",
    "build:fast":    "npm run typecheck && npm run clean && swc src -d dist --source-maps",
    "start":         "node --enable-source-maps dist/server.js",
    "ci":            "npm run verify && npm run build"
  }
}
```

Two conventions worth adopting deliberately:

```
• `dev` MUST NOT type-check. Coupling them slows the reload loop and tempts you to
  disable typecheck mid-refactor. Instant reloads in terminal 1, continuous type
  errors in terminal 2 — fast feedback AND full correctness, decoupled.

• `start` MUST NOT build. If `start` runs tsc, production compiles on boot and you
  are back to Concept 5. `start` is one `node` invocation that works with only
  `dependencies` installed.
```

Two more npm behaviours that matter:

```jsonc
// `prepare` runs automatically on `npm install` (local) and before `npm publish`.
// For a LIBRARY this guarantees dist/ exists before publishing:
{ "scripts": { "prepare": "npm run build" } }
// ⚠️ For a SERVICE this is a trap: it makes `npm ci` in your Docker prod stage
//    try to compile — and typescript isn't installed there. Use `prepublishOnly`
//    for libraries if you want publish-time-only behaviour.

// `pre`/`post` hooks are automatic for any script name:
//   "prebuild"  runs before "build"
//   "postbuild" runs after  "build"    e.g. copying non-TS assets:
{ "scripts": { "postbuild": "cp -r src/templates dist/templates" } }
// ⚠️ tsc copies ONLY .ts→.js. .sql, .json (without resolveJsonModule), .hbs,
//    .graphql, and .proto files are NOT copied. If dist/ needs them, copy them
//    yourself in postbuild — this is a very common "works locally" bug, because
//    locally you run from src/ where the files are right there.
```

---

## Example 1 — basic

The smallest complete production build for a Node service: source, two tsconfigs, scripts, and the two commands.

```
order-api/
├─ package.json
├─ tsconfig.json            ← editor + typecheck (wide)
├─ tsconfig.build.json      ← emit (narrow)
├─ src/
│  ├─ server.ts
│  └─ services/orderService.ts
└─ dist/                    ← generated, gitignored
```

```jsonc
// ── tsconfig.json — wide: typecheck + editor ────────────────────────────────
{
  "compilerOptions": {
    "target": "ES2022",             // Node 20/22 run this natively — no downlevel
    "lib": ["ES2022"],              // no DOM: this is a server
    "module": "NodeNext", "moduleResolution": "NodeNext",
    "rootDir": "src",               // ← explicit: dist/ mirrors src/ exactly
    "outDir": "dist",
    "strict": true, "noUncheckedIndexedAccess": true,
    "esModuleInterop": true, "skipLibCheck": true,
    "sourceMap": true,              // production stack traces
    "declaration": false,           // a SERVICE — nothing imports it
    "isolatedModules": true, "verbatimModuleSyntax": true,   // esbuild/swc-compatible
    "incremental": true, "tsBuildInfoFile": "./node_modules/.cache/tsbuildinfo"
  },
  "include": ["src/**/*.ts", "tests/**/*.ts", "scripts/**/*.ts"],
  "exclude": ["node_modules", "dist"]
}

// ── tsconfig.build.json — narrow: emit only ─────────────────────────────────
{
  "extends": "./tsconfig.json",     // inherit strictness — never restate it
  "compilerOptions": { "removeComments": true, "incremental": false },
  "include": ["src/**/*.ts"],       // tests and scripts must not reach dist/
  "exclude": ["node_modules", "dist", "**/*.test.ts", "**/*.spec.ts", "src/**/__tests__/**"]
}
```

```jsonc
// ── package.json ────────────────────────────────────────────────────────────
{
  "name": "order-api", "version": "1.0.0",
  "private": true,                  // guards against an accidental `npm publish`
  "type": "module",                 // matches module: NodeNext → emit stays ESM
  "main": "dist/server.js",         // .js, never .ts
  "engines": { "node": ">=22" },
  "scripts": {
    "dev":       "tsx watch src/server.ts",
    "typecheck": "tsc --noEmit",
    "build":     "rimraf dist && tsc -p tsconfig.build.json",
    "start":     "node --enable-source-maps dist/server.js"
  },
  "dependencies":    { "express": "^4.19.2" },              // imported by dist/server.js
  "devDependencies": { "@types/express": "^4.17.21",        // erased at build time
                       "@types/node": "^22.5.0", "rimraf": "^6.0.1",
                       "tsx": "^4.19.0", "typescript": "^5.6.0" }
}
```

```ts
// ── src/services/orderService.ts ────────────────────────────────────────────
export interface OrderLine { sku: string; priceCents: number; quantity: number; }  // erased
export interface Order { orderId: number; lines: readonly OrderLine[]; totalCents: number; }

const ordersById = new Map<number, Order>();
let nextOrderId = 1;

export function createOrder(lines: readonly OrderLine[]): Order {
  if (lines.length === 0) {
    throw new Error("ORDER_HAS_NO_LINES");   // ← names src/…ts in the trace, with maps on
  }
  const totalCents = lines.reduce((sum, l) => sum + l.priceCents * l.quantity, 0);
  const order: Order = { orderId: nextOrderId++, lines, totalCents };
  ordersById.set(order.orderId, order);
  return order;
}

export function findOrder(orderId: number): Order | null {
  return ordersById.get(orderId) ?? null;
}
```

```ts
// ── src/server.ts ───────────────────────────────────────────────────────────
// NOTE the .js extension on a .ts import. Under NodeNext you write the specifier
// as it will exist AT RUNTIME — tsc does not rewrite specifiers.
import express, { type Request, type Response } from "express";
import { createOrder, findOrder, type OrderLine } from "./services/orderService.js";

const app = express();
app.use(express.json());

app.post("/orders", (req: Request, res: Response) => {
  const lines = req.body?.lines as OrderLine[] | undefined;
  if (!Array.isArray(lines)) return res.status(400).json({ error: "LINES_REQUIRED" });
  return res.status(201).json({ data: createOrder(lines) });
});

app.get("/orders/:orderId", (req: Request, res: Response) => {
  const order = findOrder(Number(req.params.orderId));
  if (!order) return res.status(404).json({ error: "NOT_FOUND" });
  return res.json({ data: order });
});

app.get("/healthz", (_req, res) => res.json({ ok: true }));
app.listen(Number(process.env.PORT ?? 3000), () => console.log("listening"));
```

```bash
# ── Build and run ───────────────────────────────────────────────────────────
$ npm run typecheck
# (silent, exit 0 — nothing written)

$ npm run build
# rimraf dist && tsc -p tsconfig.build.json

$ find dist -type f | sort
dist/server.js
dist/server.js.map
dist/services/orderService.js
dist/services/orderService.js.map
# ✅ Mirrors src/ exactly, because rootDir is "src".
# ✅ No .d.ts — declaration is false.
# ✅ No test files — excluded in tsconfig.build.json.

$ npm start
listening on 3000

# Prove the source maps work — force the error path:
$ curl -s -XPOST localhost:3000/orders -H 'content-type: application/json' -d '{"lines":[]}'
# Server log:
#   Error: ORDER_HAS_NO_LINES
#       at createOrder (/app/src/services/orderService.ts:20:11)   ✅ .ts + real line
# Without --enable-source-maps it would read:
#       at createOrder (/app/dist/services/orderService.js:9:15)   ❌
```

```bash
# ── See rootDir bite, in 30 seconds ─────────────────────────────────────────
$ mkdir scripts && echo 'console.log("seed");' > scripts/seed.ts
$ npx tsc -p tsconfig.build.json --listEmittedFiles
# tsconfig.build.json's include is ["src/**/*.ts"] → scripts/seed.ts isn't compiled.
# dist/ is unchanged. ✅  The explicit rootDir + narrow include protected you.

# Now remove "rootDir" and widen include to ["**/*.ts"], rebuild:
# TSFILE: /app/dist/src/server.js          ← dist/src/, not dist/  ❌
# TSFILE: /app/dist/scripts/seed.js
# `npm start` → Error: Cannot find module '/app/dist/server.js'
```

---

## Example 2 — real world backend use case

A production service: dual-config typecheck/build, a fast swc path, non-TS asset copying, a graceful-shutdown entry point, the full multi-stage Dockerfile, a CI pipeline, and a production-deps smoke test.

```
payments-api/
├─ Dockerfile
├─ .dockerignore
├─ package.json
├─ tsconfig.json
├─ tsconfig.build.json
├─ .swcrc
├─ scripts/verify-prod-deps.sh
├─ src/
│  ├─ server.ts
│  ├─ app.ts
│  ├─ config.ts
│  ├─ bootstrap/sourceMaps.ts
│  ├─ db/migrations/001_init.sql       ← NOT copied by tsc — see postbuild
│  ├─ services/paymentService.ts
│  └─ routes/paymentRoutes.ts
└─ tests/paymentService.test.ts
```

```jsonc
// ── package.json ────────────────────────────────────────────────────────────
{
  "name": "@acme/payments-api",
  "version": "3.2.0",
  "private": true,
  "type": "module",
  "main": "dist/server.js",
  "engines": { "node": ">=22.0.0", "npm": ">=10.0.0" },

  "scripts": {
    // ── DEV: fast reload. Does NOT type-check — that's dev:types' job. ─────
    "dev":            "tsx watch --clear-screen=false src/server.ts",
    "dev:debug":      "tsx watch --inspect=0.0.0.0:9229 src/server.ts",
    "dev:types":      "tsc --noEmit --watch --preserveWatchOutput",

    // ── VERIFY: independent gates, runnable in parallel in CI ─────────────
    "typecheck":      "tsc --noEmit -p tsconfig.json",
    "lint":           "eslint . --max-warnings=0",
    "test":           "vitest run --coverage",
    "verify":         "npm run typecheck && npm run lint && npm run test",

    // ── BUILD ─────────────────────────────────────────────────────────────
    "clean":          "rimraf dist coverage *.tsbuildinfo",
    "build":          "npm run clean && tsc -p tsconfig.build.json && npm run copy:assets",
    // Fast path: typecheck with tsc (correctness), emit with swc (speed).
    // 45 s → 4 s on this codebase. Both steps are mandatory.
    "build:fast":     "npm run typecheck && npm run clean && swc src -d dist --strip-leading-paths --source-maps && npm run copy:assets",
    // tsc only emits .ts→.js. SQL/templates/protos must be copied explicitly.
    "copy:assets":    "copyfiles -u 1 \"src/**/*.{sql,hbs,graphql,proto}\" dist",

    // ── RUN ───────────────────────────────────────────────────────────────
    "start":          "node --enable-source-maps dist/server.js",
    "migrate":        "node --enable-source-maps dist/db/migrate.js",

    // ── SAFETY NETS ───────────────────────────────────────────────────────
    "verify:prod-deps": "bash scripts/verify-prod-deps.sh",
    "ci":               "npm run verify && npm run build && npm run verify:prod-deps"
  },

  // Everything dist/ imports at runtime. Nothing else.
  "dependencies": {
    "express": "^4.19.2", "pg": "^8.12.0", "pino": "^9.3.2", "stripe": "^16.8.0",
    "zod": "^3.23.8"               // runtime validation — NOT a type-only package
  },
  // Everything needed only to build, test, or lint.
  "devDependencies": {
    "@swc/cli": "^0.4.0", "@swc/core": "^1.7.14",
    "@types/express": "^4.17.21", "@types/node": "^22.5.0", "@types/pg": "^8.11.6",
    "@vitest/coverage-v8": "^2.0.5", "copyfiles": "^2.4.1", "eslint": "^9.9.0",
    "rimraf": "^6.0.1", "supertest": "^7.0.0",
    "tsx": "^4.19.0", "typescript": "^5.6.0", "vitest": "^2.0.5"
  }
}
```

```jsonc
// ── tsconfig.json — typecheck + editor. Sees EVERYTHING. ────────────────────
{
  "compilerOptions": {
    "target": "ES2022", "lib": ["ES2022"],
    "module": "NodeNext", "moduleResolution": "NodeNext",
    "rootDir": "src", "outDir": "dist",
    "strict": true, "noUncheckedIndexedAccess": true, "exactOptionalPropertyTypes": true,
    "esModuleInterop": true, "skipLibCheck": true, "forceConsistentCasingInFileNames": true,
    "sourceMap": true, "declaration": false,
    "isolatedModules": true,        // required for the swc path to be SAFE
    "verbatimModuleSyntax": true,
    "incremental": true, "tsBuildInfoFile": "./node_modules/.cache/tsbuildinfo",
    "resolveJsonModule": true
  },
  // Tests and scripts ARE type-checked (they just aren't emitted).
  "include": ["src/**/*.ts", "tests/**/*.ts", "scripts/**/*.ts", "*.config.ts"],
  "exclude": ["node_modules", "dist"]
}

// ── tsconfig.build.json — emit only. Sees ONLY shippable source. ────────────
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "removeComments": true, "incremental": false, "sourceMap": true,
    "inlineSources": false,   // we ship src/ nowhere → traces show file+line but not
                              // the source LINE TEXT. Flip to true if you want that.
    "declaration": false
  },
  "include": ["src/**/*.ts"],
  "exclude": ["node_modules", "dist", "**/*.test.ts", "**/*.spec.ts",
              "src/**/__tests__/**", "src/**/__mocks__/**", "src/testUtils/**"]
}
```

```jsonc
// ── .swcrc — the fast emit path. Mirror tsconfig's target/module EXACTLY. ───
// A mismatch here means dev (tsc semantics) and prod (swc semantics) differ,
// which is the worst kind of bug to chase.
{
  "$schema": "https://swc.rs/schema.json",
  "jsc": {
    "parser": { "syntax": "typescript", "decorators": false },
    "target": "es2022",              // ← must match tsconfig "target"
    "loose": false,                  // spec-correct semantics, not fast-and-wrong
    "externalHelpers": false         // true ⇒ tslib becomes a runtime DEPENDENCY
  },
  "module": { "type": "es6" },       // ← must match "type": "module" in package.json
  "sourceMaps": true,
  "exclude": [".*\\.test\\.ts$", ".*\\.spec\\.ts$"]
}
```

```ts
// ── src/bootstrap/sourceMaps.ts ─────────────────────────────────────────────
// --enable-source-maps handles Error.stack. This adds what the flag does NOT:
// a deeper stack limit and structured fatal-error logging.
// If your platform won't let you pass node flags, uncomment the register import
// and add source-map-support to `dependencies` (NOT devDependencies):
// import "source-map-support/register.js";   // must be the VERY FIRST import

Error.stackTraceLimit = 50;   // default 10 truncates async chains past a few awaits

function serialise(err: unknown): Record<string, unknown> {
  const e = err instanceof Error ? err : new Error(String(err));
  return { name: e.name, message: e.message,
           stack: e.stack,   // ← mapped back to src/*.ts by --enable-source-maps
           cause: e.cause instanceof Error ? e.cause.message : undefined };
}

for (const event of ["uncaughtException", "unhandledRejection"] as const) {
  process.on(event, (err: unknown) => {
    console.error(JSON.stringify({ level: "fatal", event, ...serialise(err) }));
    process.exit(1);   // undefined state after either — never continue
  });
}
```

```ts
// ── src/config.ts — validate env at BOOT, not at first use ──────────────────
import { z } from "zod";   // runtime validation ⇒ a real `dependencies` entry

const EnvSchema = z.object({
  NODE_ENV: z.enum(["development", "test", "production"]).default("development"),
  PORT: z.coerce.number().int().positive().default(3000),
  DATABASE_URL: z.string().url(),
  STRIPE_SECRET_KEY: z.string().startsWith("sk_"),
  SHUTDOWN_TIMEOUT_MS: z.coerce.number().int().positive().default(15_000),
});
export type Config = z.infer<typeof EnvSchema>;

export function loadConfig(env: NodeJS.ProcessEnv): Config {
  const parsed = EnvSchema.safeParse(env);
  if (!parsed.success) {
    // Fail FAST and LOUD. A container that exits 1 at boot is a clear signal;
    // one that 500s on 3% of requests three hours later is not.
    console.error(JSON.stringify({ level: "fatal", event: "invalidConfig",
      issues: parsed.error.issues.map((i) => ({ path: i.path.join("."), message: i.message })) }));
    process.exit(1);
  }
  return parsed.data;
}
```

```ts
// ── src/server.ts — production entry point ──────────────────────────────────
import "./bootstrap/sourceMaps.js";   // FIRST: installs handlers before anything can throw

import { createApp } from "./app.js";
import { loadConfig } from "./config.js";
import { createPool, closePool } from "./db/client.js";

const config = loadConfig(process.env);
const pool = createPool(config.DATABASE_URL);
const app = createApp(config, pool);

const server = app.listen(config.PORT, () => {
  console.log(JSON.stringify({
    level: "info",
    event: "listening",
    port: config.PORT,
    nodeEnv: config.NODE_ENV,
    // Baked in at build time — makes "which build is running?" answerable from logs.
    commit: process.env.GIT_COMMIT ?? "unknown",
  }));
});

// ── Graceful shutdown ─────────────────────────────────────────────────────
// Kubernetes/ECS send SIGTERM and wait (default 30s) before SIGKILL. Without
// this handler, in-flight requests are severed mid-response on every deploy.
async function shutdown(signal: NodeJS.Signals): Promise<void> {
  console.log(JSON.stringify({ level: "info", event: "shutdown", signal }));

  const forceExit = setTimeout(() => {
    console.error(JSON.stringify({ level: "error", event: "shutdownTimeout" }));
    process.exit(1);
  }, config.SHUTDOWN_TIMEOUT_MS);
  forceExit.unref();   // don't let this timer itself keep the loop alive

  server.close(async () => {          // stop accepting new connections, drain existing
    await closePool(pool);            // then release DB handles
    clearTimeout(forceExit);
    process.exit(0);
  });
}

process.on("SIGTERM", () => void shutdown("SIGTERM"));   // orchestrator
process.on("SIGINT",  () => void shutdown("SIGINT"));    // Ctrl-C
```

```dockerfile
# ── Dockerfile — same four stages as Concept 12, plus BUILD-TIME ASSERTIONS ──
# Build: docker build --build-arg GIT_COMMIT=$(git rev-parse --short HEAD) -t payments-api .

FROM node:22-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN --mount=type=cache,target=/root/.npm npm ci

FROM node:22-alpine AS build
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY package.json package-lock.json tsconfig.json tsconfig.build.json .swcrc ./
COPY src ./src
RUN npm run typecheck        # separate RUN lines: a red build names the failing gate
RUN npm run build

# ⭐ The assertions. Each one turns a class of production incident into a red build.
RUN test -f dist/server.js     || (echo "dist/server.js missing — check rootDir/outDir"; exit 1)
RUN test -f dist/server.js.map || (echo "source maps missing — check sourceMap: true";  exit 1)
RUN test -f dist/db/migrations/001_init.sql || (echo "assets not copied — copy:assets"; exit 1)
RUN ! find dist -name "*.test.js" | grep -q . \
      || (echo "tests leaked into dist/ — fix tsconfig.build.json exclude"; exit 1)

FROM node:22-alpine AS prod-deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN --mount=type=cache,target=/root/.npm npm ci --omit=dev && npm cache clean --force

FROM node:22-alpine AS runtime
WORKDIR /app
ARG GIT_COMMIT=unknown
ENV NODE_ENV=production NODE_OPTIONS="--enable-source-maps" GIT_COMMIT=${GIT_COMMIT}
RUN apk add --no-cache dumb-init
USER node
COPY --from=prod-deps --chown=node:node /app/node_modules ./node_modules
COPY --from=build     --chown=node:node /app/dist         ./dist   # WHOLE dir: maps + .sql
COPY --chown=node:node package.json ./
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=3s --start-period=15s --retries=3 \
  CMD node -e "fetch('http://127.0.0.1:3000/healthz').then(r=>process.exit(r.ok?0:1)).catch(()=>process.exit(1))"
# dumb-init as PID 1 forwards SIGTERM so the graceful shutdown above actually runs.
ENTRYPOINT ["dumb-init", "--"]
CMD ["node", "dist/server.js"]
```

```bash
#!/usr/bin/env bash
# ── scripts/verify-prod-deps.sh ─────────────────────────────────────────────
# Catches "it works on my machine because I have devDependencies installed"
# BEFORE deploy. Run in CI right after `npm run build`.
set -euo pipefail
WORK="$(mktemp -d)"; trap 'rm -rf "$WORK"' EXIT
cp -r dist package.json package-lock.json "$WORK/" && cd "$WORK"

npm ci --omit=dev --ignore-scripts          # prod tree only; no `prepare` surprises

npm ls typescript --omit=dev 2>/dev/null | grep -q "typescript@" \
  && { echo "✗ typescript reachable from production deps"; exit 1; }

# Import the module graph WITHOUT starting the listener: any missing runtime
# dependency throws ERR_MODULE_NOT_FOUND right here, in CI, not in prod.
NODE_ENV=production DATABASE_URL="postgres://localhost:5432/x" STRIPE_SECRET_KEY="sk_test_x" \
timeout 20s node --enable-source-maps -e '
  import("./dist/app.js")
    .then(() => console.log("✓ module graph resolves with production deps only"))
    .catch((e) => { console.error("✗ " + e.message); process.exit(1); });'
```

```yaml
# ── .github/workflows/ci.yml ────────────────────────────────────────────────
name: CI
on: [push, pull_request]
jobs:
  # typecheck / lint / test are INDEPENDENT because typecheck emits nothing.
  # Running them in parallel turns a ~4-minute serial pipeline into ~2 minutes.
  verify:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false                     # see ALL failures in one run
      matrix: { task: [typecheck, lint, test] }
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: npm }
      - run: npm ci                        # exact lockfile install
      - run: npm run ${{ matrix.task }}

  build:
    needs: verify                          # never build what failed verification
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: npm }
      - run: npm ci
      - run: npm run build
      - run: npm run verify:prod-deps      # ← catches dependency misclassification
      - uses: actions/upload-artifact@v4
        with: { name: dist, path: dist/, retention-days: 7 }

  docker:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/build-push-action@v6
        with:
          context: .
          target: runtime
          build-args: GIT_COMMIT=${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          tags: payments-api:${{ github.sha }}
```

```bash
# ── What you should observe ─────────────────────────────────────────────────
$ npm run build          # tsc path
real  0m47.2s            # full type check + emit

$ npm run build:fast     # tsc --noEmit + swc
real  0m4.1s             # SAME safety (typecheck still runs), ~11× faster emit

$ docker build -t payments-api . && docker images payments-api
payments-api   latest   a1b2c3d4   148MB     # vs ~1.1 GB single-stage

$ docker run --rm payments-api node -e "console.log(require('typescript').version)"
# Error: Cannot find module 'typescript'      ✅ exactly what we want
```

---

## Going deeper

### `tsc --build` and project references

For a monorepo (covered fully in 77), `tsc --build` treats each package as a node in a dependency graph, builds them in topological order, and skips any whose `.tsbuildinfo` says it's up to date.

```jsonc
// tsconfig.json at the repo root — a "solution" file with no files of its own
{ "files": [], "references": [{ "path": "./packages/shared" },
                              { "path": "./services/payments-api" }] }

// packages/shared/tsconfig.json
{
  "compilerOptions": {
    "composite": true,       // required for a REFERENCED project. Implies
                             // declaration + incremental, and rootDir must be explicit.
    "declaration": true, "declarationMap": true,
    "rootDir": "src", "outDir": "dist"
  },
  "include": ["src/**/*.ts"]
}
```

```bash
tsc --build          # build everything in topological order, incrementally  (-b)
tsc --build --watch  # watch the whole graph
tsc --build --clean  # delete every tracked output
tsc --build --dry    # print what WOULD be built

# The key benefit: services/payments-api type-checks against packages/shared's EMITTED
# .d.ts, not its source. Rebuilding the API doesn't re-check shared, and shared's public
# API becomes a real, enforced contract boundary.
```

### `isolatedDeclarations` (TypeScript 5.5+)

The newest lever for library build speed. It forbids any exported declaration whose type must be *inferred*, which means `.d.ts` files can be generated from a single file without a type checker — so esbuild/swc/oxc can emit declarations too.

```jsonc
{ "compilerOptions": { "isolatedDeclarations": true, "declaration": true } }
```

```ts
// ❌ error TS9007: Function must have an explicit return type annotation
//    with --isolatedDeclarations.
export function makeClient(config: Config) {
  return { charge: (cents: number) => api.post("/charge", { cents }) };
}

// ✅ annotate the public surface explicitly
export interface Client { charge(cents: number): Promise<Receipt>; }
export function makeClient(config: Config): Client { /* … */ }
```

More typing up front, but explicit public signatures are better documentation anyway — and `.d.ts` generation drops from tens of seconds to milliseconds.

### Bundling a service: when it's worth it

`esbuild --bundle` collapses your code *and* your `node_modules` into one file.

```bash
esbuild src/server.ts \
  --bundle \
  --platform=node \
  --target=node22 \
  --format=esm \
  --outfile=dist/server.js \
  --sourcemap \
  --external:pg-native \        # optional native dep that isn't always installed
  --external:better-sqlite3     # native .node binaries CANNOT be bundled
```

```
WORTH IT:      Lambda / Cloud Functions (one file, no node_modules, fastest cold start);
               CLI tools shipped as a single artifact; 300 MB node_modules → 4 MB file.

NOT WORTH IT:  A normal containerised service. Layer caching already makes node_modules
               effectively free, and bundling breaks native modules (.node binaries),
               computed dynamic require()/import(), __dirname-relative asset loading, and
               anything monkey-patching a dependency. It also degrades stack traces unless
               the sourcemap is perfect, and blinds SCA scanners that read node_modules.
```

### Making builds reproducible

```dockerfile
# 1. Pin the base image by DIGEST, not tag — `node:22-alpine` moves under you.
FROM node:22.7.0-alpine@sha256:2d07db07a2df6830718ae2a47db6f... AS build
# 2. `npm ci`, never `npm install`. ci installs the lockfile exactly and fails if
#    package.json and the lock disagree; install mutates the lock.
RUN npm ci --ignore-scripts        # --ignore-scripts also cuts a supply-chain vector
# 3. Bake identity in, so a running container can say what it is.
ARG GIT_COMMIT
ARG BUILD_TIME
ENV GIT_COMMIT=${GIT_COMMIT} BUILD_TIME=${BUILD_TIME}
```

```ts
// Expose it — turns "which build is broken?" into a single curl.
app.get("/version", (_req, res) => res.json({
  version: process.env.npm_package_version,
  commit:  process.env.GIT_COMMIT ?? "unknown",
  builtAt: process.env.BUILD_TIME ?? "unknown",
  node:    process.version,
}));
```

### Build performance: measure before you optimise

```bash
tsc --diagnostics
# Files: 1284 │ Lines: 412903 │ Types: 89234   ← huge type counts ⇒ heavy generics
# Check time: 31.42s   ← the type checker
# Emit time:   4.11s   ← writing files (usually small!)
# Total time: 41.09s
tsc --extendedDiagnostics      # more detail, including memory
tsc --generateTrace ./trace    # open trace/trace.json in chrome://tracing
```

```
The usual wins, in order of payoff:
1. "skipLibCheck": true          often 20–40%. Stops checking every node_modules .d.ts.
2. "incremental": true           50–80% on rebuilds (no help in a cold CI runner).
3. Narrow `include`              ["src/**/*.ts"], not ["**/*.ts"].
4. Project references            tsc --build only rebuilds changed packages.
5. Simplify pathological types   deep conditional/recursive types can dominate check
                                 time. Find them with --generateTrace.
6. Split typecheck from emit     tsc --noEmit + swc/esbuild (Concept 7).
```

### `NODE_ENV=production` — what it does and doesn't do

```
DOES:      libs (express, react, …) skip dev-only warnings/validation; `npm install`
           (not `ci`) skips devDependencies; some libs enable caching.
DOES NOT:  change anything about how TypeScript compiled — that already happened;
           strip your debug code; enable source maps; make your app faster by itself.
```

```ts
// NODE_ENV-conditional behaviour in your OWN code is a real runtime branch —
// tsc performs no dead-code elimination:
if (process.env.NODE_ENV !== "production") {
  app.use(verboseRequestLogger);   // present in dist/, just not executed in prod
}
// esbuild CAN eliminate it if you supply the value at build time:
//   esbuild --define:process.env.NODE_ENV='"production"' --minify
```

### Shipping ESM and CJS from one build (libraries)

```jsonc
{
  "type": "module",
  "exports": { ".": { "types": "./dist/index.d.ts",
                      "import": "./dist/esm/index.js",
                      "require": "./dist/cjs/index.cjs" } },
  "scripts": {
    "build":       "npm run build:esm && npm run build:cjs && npm run build:types",
    "build:esm":   "tsc -p tsconfig.esm.json",
    "build:cjs":   "tsc -p tsconfig.cjs.json && node scripts/rename-cjs.mjs",
    "build:types": "tsc -p tsconfig.build.json --emitDeclarationOnly"
  }
}
```

Genuinely fiddly: file extensions, a `type` field per output dir, and the **dual-package hazard** (two copies of your module ⇒ two copies of every singleton and broken `instanceof` identity). Prefer ESM-only unless you must support CJS consumers; if you must, use `tsup` or `unbuild`, which handle the details.

### What actually ends up in the image

```bash
docker run --rm payments-api sh -c 'find . -name "*.ts" | head'   # must be EMPTY
docker run --rm payments-api sh -c 'du -sh node_modules dist'
docker history payments-api --human --format '{{.Size}}\t{{.CreatedBy}}' | head -20
npx dive payments-api          # interactive per-layer breakdown

# Common surprises: node_modules/.cache left in the image; a full `npm ci` (with dev
# deps) in the final stage; .git copied by `COPY . .` with no .dockerignore (leaks
# history and secrets); locally-built node_modules copied in with the wrong CPU arch.
```

---

## Common mistakes

### Mistake 1 — `start` script that runs TypeScript

```jsonc
// ❌ Every problem in Concept 5, permanently installed.
{
  "scripts": { "start": "ts-node src/server.ts" },   // or "tsx src/server.ts"
  "dependencies": { "typescript": "^5.6.0",          // dragged into prod to make it work
                    "ts-node": "^10.9.2" }
}
// Symptoms: 5–15 s cold starts, +300 MB RSS, +100 MB image, type errors as pod
// crashes, and a compiler sitting inside your production attack surface.
```

```jsonc
// ✅ Build once. Run plain node. The compiler goes back to devDependencies.
{
  "scripts": {
    "dev":       "tsx watch src/server.ts",
    "typecheck": "tsc --noEmit",
    "build":     "rimraf dist && tsc -p tsconfig.build.json",
    "start":     "node --enable-source-maps dist/server.js"
  },
  "dependencies":    { "express": "^4.19.2" },
  "devDependencies": { "typescript": "^5.6.0", "tsx": "^4.19.0" }
}
```

### Mistake 2 — Not setting `rootDir`, then hardcoding the entry path

```jsonc
// ❌ rootDir is inferred, so dist/'s shape depends on which files happen to exist.
{ "compilerOptions": { "outDir": "dist" },
  "include": ["**/*.ts"] }                     // catches scripts/, tests/, *.config.ts
// package.json: "main": "dist/server.js"
// Day 1:  dist/server.js       ✅ (inferred rootDir = "src")
// Day 30: someone adds scripts/seed.ts →
//         dist/src/server.js   ❌ (inferred rootDir = ".")
//         → Error: Cannot find module '/app/dist/server.js'  — in production
```

```jsonc
// ✅ Explicit rootDir + narrow include. The layout can never drift silently, and an
//    out-of-root import becomes TS6059 at build time instead of a broken deploy.
{ "compilerOptions": { "rootDir": "src", "outDir": "dist" },
  "include": ["src/**/*.ts"],
  "exclude": ["node_modules", "dist", "**/*.test.ts"] }
// Verify: npx tsc -p tsconfig.build.json --listEmittedFiles
```

### Mistake 3 — Building without cleaning

```jsonc
// ❌ tsc only ADDS files. Renames, deletes, and moves leave orphans forever.
{ "scripts": { "build": "tsc -p tsconfig.build.json" } }
// After six months of refactoring:
//   dist/services/userService.js       ← deleted from src/ in March, still here
//   dist/routes/legacyAdminRoutes.js   ← removed route, still loadable
//   dist/services/emailService.js      ← compiled with an older `target`
// Worst case: a directory-scanning route loader re-registers legacyAdminRoutes.
```

```jsonc
// ✅ Delete outDir first, every time. Non-negotiable.
{ "scripts": { "clean": "rimraf dist *.tsbuildinfo",
               "build": "npm run clean && tsc -p tsconfig.build.json" } }
// ⚠️ Clean the .tsbuildinfo too. Delete dist/ but keep the buildinfo, and
//    `incremental: true` makes tsc think it's up to date → EMPTY dist/, exit 0.
```

### Mistake 4 — Runtime packages in `devDependencies`

```jsonc
// ❌ Works locally (you have everything installed). Breaks in the image.
{
  "dependencies": { "express": "^4.19.2" },
  "devDependencies": {
    "dotenv": "^16.4.5",              // dist/config.js does import "dotenv/config"
    "source-map-support": "^0.5.21",  // dist/bootstrap/sourceMaps.js imports it
    "tslib": "^2.6.3",                // importHelpers: true ⇒ dist/*.js imports it
    "reflect-metadata": "^0.2.2"      // a NestJS entry imports it at runtime
  }
}
// npm ci --omit=dev && node dist/server.js
//   → Error [ERR_MODULE_NOT_FOUND]: Cannot find package 'dotenv' imported from
//     /app/dist/config.js
```

```jsonc
// ✅ Rule: if any file in dist/ contains `import "x"` / `require("x")`, then
//    "x" belongs in dependencies. Full stop.
{
  "dependencies": { "express": "^4.19.2", "dotenv": "^16.4.5",
                    "source-map-support": "^0.5.21", "tslib": "^2.6.3",
                    "reflect-metadata": "^0.2.2" },
  "devDependencies": { "typescript": "^5.6.0", "@types/node": "^22.5.0" }
}
// Prove it in CI — list every bare specifier dist/ actually imports:
//   grep -rhoE "from ['\"][^.][^'\"]*['\"]" dist --include="*.js" | cut -d/ -f1 | sort -u
// …then cross-check against `dependencies`, or just run verify-prod-deps.sh.
```

### Mistake 5 — Switching to esbuild/swc and dropping type checking

```jsonc
// ❌ Build went from 45 s to 2 s. Nothing type-checks anymore, and nobody noticed
//    because the build is GREEN.
{
  "scripts": {
    "build": "esbuild src/server.ts --bundle --platform=node --outfile=dist/server.js"
  }
}
// Six weeks later: a renamed field, a swapped argument order, a null that tsc
// would have caught on the first keystroke — all in production.
```

```jsonc
// ✅ Type checking is a SEPARATE, MANDATORY gate that precedes emit.
{
  "scripts": {
    "typecheck": "tsc --noEmit",
    "build":     "npm run typecheck && npm run clean && esbuild src/server.ts --bundle --platform=node --target=node22 --outfile=dist/server.js --sourcemap",
    "ci":        "npm run typecheck && npm run lint && npm test && npm run build"
  }
}
// And make per-file transpilation safe by construction:
// tsconfig: { "isolatedModules": true, "verbatimModuleSyntax": true }
```

### Mistake 6 — Copying `dist/*.js` in the Dockerfile

```dockerfile
# ❌ The glob matches .js only. Silently drops .map files (and .sql, .hbs, .json).
COPY --from=build /app/dist/*.js ./dist/

# Consequences:
#   1. It only matches the TOP LEVEL — dist/services/, dist/routes/ are missing
#      entirely → MODULE_NOT_FOUND at boot.
#   2. Even if it matched recursively, --enable-source-maps finds no maps and
#      your stack traces stay at dist/*.js coordinates. No error, no warning.
```

```dockerfile
# ✅ Copy the directory. Whole. Always.
COPY --from=build --chown=node:node /app/dist ./dist

# And assert it in the build stage, so a mistake fails the build not the deploy:
RUN test -f dist/server.js.map || (echo "source maps missing" && exit 1)
```

### Mistake 7 — Test files shipping to production

```jsonc
// ❌ tsconfig.build.json with no exclude, or just reusing the wide tsconfig.json.
{ "extends": "./tsconfig.json", "compilerOptions": { "outDir": "dist" } }
// → dist/services/paymentService.test.js exists and imports vitest/supertest, which
//   are devDependencies. If ANYTHING loads it: MODULE_NOT_FOUND. And test fixtures
//   (seeded tokens, sample PANs, staging URLs) ship inside your image.
```

```jsonc
// ✅ Narrow the emit config explicitly.
{
  "extends": "./tsconfig.json",
  "compilerOptions": { "removeComments": true, "incremental": false },
  "include": ["src/**/*.ts"],
  "exclude": ["node_modules", "dist", "**/*.test.ts", "**/*.spec.ts",
              "src/**/__tests__/**", "src/**/__mocks__/**", "src/testUtils/**"]
}
// Assert it in the Docker build:  RUN ! find dist -name "*.test.js" | grep -q .
```

### Mistake 8 — A library with no `types` / no `declaration`

```jsonc
// ❌ Published package. Consumers get `any` everywhere and don't know why.
{ "name": "@acme/sdk", "main": "dist/index.js", "files": ["dist"] }
// tsconfig: { "declaration": false }
// Consumer: import { createClient } from "@acme/sdk";
//   → error TS7016: Could not find a declaration file for module '@acme/sdk'.
//     '.../dist/index.js' implicitly has an 'any' type.
```

```jsonc
// ✅ Emit declarations, point at them, and ship them.
// tsconfig.build.json: { "declaration": true, "declarationMap": true }
{
  "name": "@acme/sdk", "type": "module",
  "exports": { ".": { "types":  "./dist/index.d.ts",   // ← MUST be the FIRST condition
                      "import": "./dist/index.js" } },
  "types": "./dist/index.d.ts",                        // legacy fallback
  "main":  "./dist/index.js",
  "files": ["dist", "src", "README.md", "LICENSE"]     // src/ so declarationMap resolves
}
// Verify before publishing:
//   npm pack --dry-run && npx publint && npx @arethetypeswrong/cli --pack .
```

### Mistake 9 — Assuming `tsc` copies your non-TypeScript files

```jsonc
// ❌ src/db/migrations/001_init.sql and src/templates/receipt.hbs are NOT in dist/.
//    tsc compiles .ts → .js. It copies nothing else.
{ "scripts": { "build": "tsc -p tsconfig.build.json" } }

// Locally you run from src/ (via tsx) so the files are right there and it works.
// In production: ENOENT: no such file or directory, open '/app/dist/db/migrations/001_init.sql'
```

```jsonc
// ✅ Copy assets explicitly as part of the build.
{
  "scripts": {
    "copy:assets": "copyfiles -u 1 \"src/**/*.{sql,hbs,graphql,proto}\" dist",
    "build": "npm run clean && tsc -p tsconfig.build.json && npm run copy:assets"
  }
}
// Also: resolve asset paths from the MODULE, not from process.cwd(), so they
// work regardless of where the process was started:
//   import { fileURLToPath } from "node:url";
//   const here = path.dirname(fileURLToPath(import.meta.url));
//   const sql = await readFile(path.join(here, "migrations/001_init.sql"), "utf8");
```

---

## Practice exercises

### Exercise 1 — easy

Build a scratch service from nothing and produce a runnable `dist/`, then break and fix `rootDir` on purpose.

Concretely:

1. `mkdir ts-build-lab && cd ts-build-lab && npm init -y && npm i -D typescript @types/node rimraf tsx`
2. Set `"type": "module"` in `package.json`.
3. Write `tsconfig.json` with `target: "ES2022"`, `module: "NodeNext"`, `moduleResolution: "NodeNext"`, `rootDir: "src"`, `outDir: "dist"`, `strict: true`, `sourceMap: true`, `declaration: false`, `include: ["src/**/*.ts"]`.
4. Write `src/index.ts` and `src/lib/money.ts`. `money.ts` exports `formatCents(cents: number): string` and throws `new Error("NEGATIVE_AMOUNT")` for negative input. `index.ts` imports it (remember the `.js` extension) and calls it with `-1`.
5. Add scripts: `typecheck` (`tsc --noEmit`), `build` (`rimraf dist && tsc`), `start` (`node --enable-source-maps dist/index.js`).
6. Run `npm run build`, then `find dist -type f`. Confirm you see exactly `dist/index.js`, `dist/index.js.map`, `dist/lib/money.js`, `dist/lib/money.js.map`.
7. Run `npm start`. Confirm the stack trace names `src/lib/money.ts` with a correct line number. Now run `node dist/index.js` **without** `--enable-source-maps` and record the difference.
8. **Break `rootDir`:** delete the `rootDir` line, change `include` to `["**/*.ts"]`, add `scripts/seed.ts` with one `console.log`. Rebuild with `npx tsc --listEmittedFiles`. Record where `index.js` landed and what `npm start` now prints.
9. Fix it two ways, and write one sentence on which you'd standardise on: (a) restore `rootDir: "src"` and the narrow `include`; (b) keep the wide include but update `main`/`start` to the new path.
10. **Prove the stale-output problem:** rename `src/lib/money.ts` → `src/lib/currency.ts`, update the import, and run `npx tsc` **without** cleaning. Show that `dist/lib/money.js` is still there. Then run `npm run build` and show it's gone.

```
// Write your code here
```

### Exercise 2 — medium

Turn that scratch project into a real service with split typecheck/build configs, a fast swc path, asset copying, and a multi-stage Docker image — then prove each safeguard by breaking it.

Concretely:

1. Scaffold an Express service with `src/server.ts`, `src/app.ts`, `src/config.ts` (zod-validated env), `src/services/orderService.ts`, `src/routes/orderRoutes.ts`, and a `tests/orderService.test.ts` using Vitest. Add `src/db/migrations/001_init.sql` containing real SQL that the app reads at boot.
2. Create **two** tsconfigs: a wide `tsconfig.json` that includes `src/`, `tests/`, and `scripts/`, and a narrow `tsconfig.build.json` that extends it, sets `removeComments`, `declaration: false`, `incremental: false`, and excludes all test files.
3. Add the four canonical scripts (`dev`, `typecheck`, `build`, `start`) plus `clean`, `lint`, `test`, and `copy:assets`. `build` must clean first and copy assets last.
4. Prove the exclude works: temporarily remove the test exclusions from `tsconfig.build.json`, rebuild, and show `dist/**/*.test.js` appearing. Restore, rebuild, show it gone. Add a build-time assertion (`! find dist -name "*.test.js" | grep -q .`) that would have caught it.
5. Prove the asset problem: run `npm start` with only `tsc` in the build (no `copy:assets`) and capture the `ENOENT` for the `.sql` file. Then add `copy:assets` and show it boots. Also change the file read to resolve from `import.meta.url` rather than `process.cwd()`, and explain in a comment why that matters when the process is started from `/`.
6. Add a **fast build path**: install `@swc/core` and `@swc/cli`, write `.swcrc` whose `target` and `module` exactly match your tsconfig, and add `"build:fast": "npm run typecheck && npm run clean && swc src -d dist --strip-leading-paths --source-maps && npm run copy:assets"`. Time both. Record the ratio.
7. Demonstrate the trade-off concretely: introduce a real type error (swap two arguments at a call site). Show `npm run build` fails and `swc src -d dist` alone succeeds. Then show `npm run build:fast` catches it because of the `typecheck` gate.
8. Write a four-stage `Dockerfile` (`deps`, `build`, `prod-deps`, `runtime`) plus a `.dockerignore`. The runtime stage must set `NODE_ENV`, `NODE_OPTIONS=--enable-source-maps`, run as the `node` user, and `COPY` the whole `dist` directory.
9. Prove the image is clean: `docker run --rm <img> sh -c 'find . -name "*.ts" | head'` returns nothing, and `docker run --rm <img> node -e "require('typescript')"` fails with MODULE_NOT_FOUND. Record the final image size and compare it to a naive single-stage build.
10. Break the `COPY` on purpose (`COPY --from=build /app/dist/*.js ./dist/`), rebuild, and document the exact runtime symptom.

```
// Write your code here
```

### Exercise 3 — hard

Ship two artifacts from one repo — a service and a publishable library — with a CI pipeline that makes every failure mode in this doc impossible.

Concretely:

1. **Restructure as a workspace.** `packages/shared` (library) and `services/api` (service), wired with npm workspaces. `shared` exports a `Money` type, a `formatCents` function, and an `AppError` class that `api` imports.
2. **Library build.** `packages/shared` emits `declaration: true`, `declarationMap: true`, `sourceMap: true`, `stripInternal: true`. Write its `package.json` with a full `exports` map (`types` first), plus legacy `main`/`module`/`types`, and a `files` allowlist containing `dist`, `src`, `README.md`, `LICENSE`. Mark one helper `/** @internal */` and prove it is absent from the emitted `.d.ts`.
3. **Verify the package surface.** Run `npm pack --dry-run`, `npx publint`, and `npx @arethetypeswrong/cli --pack .`. Fix every reported issue. Then deliberately reorder `exports` so `"import"` precedes `"types"`, re-run the checks, and record exactly what breaks and how it is reported.
4. **Reproduce TS2742.** In `shared`, export a function whose *inferred* return type references a type from a devDependency. Capture the exact error. Fix it twice — once by annotating an explicit return interface, once by moving the package to `dependencies` — and write two sentences on which is right for a published library.
5. **Service build.** `services/api` emits `declaration: false`. Set up project references (`composite: true` on shared, `references` in api) and build the whole graph with `tsc --build`. Show `tsc --build --dry` skipping up-to-date projects, and confirm `tsc --build --clean` removes every output.
6. **Dependency classification audit.** Write `scripts/verify-prod-deps.sh` that copies `dist` + manifests to a temp dir, runs `npm ci --omit=dev --ignore-scripts`, asserts `npm ls typescript --omit=dev` is empty, and imports the module graph to prove nothing is missing. Then deliberately move `zod` to `devDependencies` and show the script failing with the exact `ERR_MODULE_NOT_FOUND`.
7. **Source maps end to end.** Build the service image, run it, trigger a deliberate 500, and show the log stack trace naming `src/**/*.ts`. Then rebuild with `sourceMap: false` and record the difference. Then set `inlineSources: true`, rebuild, and explain what extra information appears and what the size cost was (`du -sh dist` before and after).
8. **Multi-stage Docker with build assertions.** Four stages, digest-pinned base image, `--mount=type=cache` for npm, `GIT_COMMIT` build arg surfaced at `GET /version`, `dumb-init` as PID 1, `USER node`, and a `HEALTHCHECK`. Add in-build assertions for: `dist/server.js` exists, `dist/server.js.map` exists, and no `*.test.js` in `dist`. Break each of the three on purpose and confirm the build (not the deploy) fails.
9. **Graceful shutdown proof.** Implement SIGTERM handling with a forced-exit timeout. Start the container, hold a slow request open, `docker stop` it, and demonstrate the in-flight request completes. Then remove `dumb-init` and show the signal never reaching Node.
10. **CI pipeline.** A GitHub Actions workflow with a parallel `verify` matrix (`typecheck` | `lint` | `test`, `fail-fast: false`), a `build` job that runs `verify:prod-deps` and uploads `dist/` as an artifact, and a `docker` job using BuildKit cache. Add a job that publishes `packages/shared` to a local Verdaccio registry and then installs it into a fresh throwaway TypeScript project to prove types resolve.
11. **Write `BUILDING.md`** documenting: the four scripts and what each does; why `dev` doesn't type-check; why `start` doesn't build; the `rootDir` rule; the clean-build rule; the dependency-classification rule; and a symptom → cause → fix table covering at least eight distinct build failures you actually hit doing steps 1–10.

```
// Write your code here
```

---

## Quick reference cheat sheet

```jsonc
// ── tsconfig emit options ───────────────────────────────────────────────────
"rootDir": "src"              // input root — SET IT EXPLICITLY. Anchors output layout.
"outDir": "dist"              // output root. Relative to the tsconfig FILE.
"noEmit": true                // typecheck only, write nothing (the typecheck script)
"emitDeclarationOnly": true   // .d.ts only — pair with esbuild/swc for the .js
"declaration": true           // .d.ts — libraries yes, services no
"declarationMap": true        // .d.ts.map — consumers' go-to-definition lands on .ts
"sourceMap": true             // .js.map — readable production stack traces
"inlineSources": true         // embed .ts text in the map (slim images)
"removeComments": true        // smaller output
"importHelpers": true         // ⚠️ requires tslib in `dependencies`
"isolatedModules": true       // stay compatible with per-file transpilers
"verbatimModuleSyntax": true  // explicit `import type`; no elision guessing
"incremental": true           // .tsbuildinfo — fast rebuilds (clean it with dist/!)
"composite": true             // project references (implies declaration + incremental)
"skipLibCheck": true          // don't check node_modules .d.ts — 20–40% faster
```

```bash
# ── tsc CLI ─────────────────────────────────────────────────────────────────
tsc --noEmit                   # THE typecheck command
tsc -p tsconfig.build.json     # build with a specific config
tsc --build                    # project-references build  (-b)
tsc --build --clean            # delete tracked outputs
tsc --build --force            # rebuild everything
tsc --listEmittedFiles         # what did it WRITE?  (debug outDir/rootDir)
tsc --listFiles                # what did it READ?   (debug accidental includes)
tsc --showConfig               # fully-resolved config after `extends`
tsc --diagnostics              # timings: check time vs emit time
tsc --generateTrace ./trace    # perf trace for chrome://tracing
```

```jsonc
// ── The four scripts ────────────────────────────────────────────────────────
"dev":       "tsx watch src/server.ts"                    // no type check, instant reload
"typecheck": "tsc --noEmit"                               // types only, no output
"build":     "rimraf dist && tsc -p tsconfig.build.json"  // CLEAN, then emit
"start":     "node --enable-source-maps dist/server.js"   // plain node, prod deps only
```

| Script | Type-checks | Emits | Needs devDeps | Runs in prod |
|---|---|---|---|---|
| `dev` | ❌ | in-memory | ✅ | ❌ never |
| `typecheck` | ✅ full | ❌ | ✅ | ❌ never |
| `build` | ✅ full | `dist/` | ✅ | ❌ (CI/build stage only) |
| `start` | n/a | ❌ | ❌ | ✅ this is the one |

```jsonc
// ── package.json fields ─────────────────────────────────────────────────────
"main":    "./dist/index.cjs"   // CJS entry / legacy fallback
"module":  "./dist/index.js"    // ESM entry — BUNDLER convention, Node ignores it
"types":   "./dist/index.d.ts"  // legacy type entry
"exports": { ".": { "types": "…", "import": "…", "require": "…" } }
//                    ^^^^^^^ MUST BE FIRST — conditions match top-to-bottom
"files":   ["dist", "src", "README.md"]   // publish ALLOWLIST
"type":    "module"             // .js files are ESM
"private": true                 // services: prevents accidental publish
"sideEffects": false            // libraries: enables tree-shaking
```

```bash
# ── dependencies vs devDependencies ─────────────────────────────────────────
# RULE: if any file in dist/ imports it → `dependencies`. Otherwise → `devDependencies`.
dependencies:     express pg zod pino stripe tslib(if importHelpers) reflect-metadata
                  source-map-support(if imported)
devDependencies:  typescript tsx ts-node @types/* eslint vitest jest rimraf esbuild swc

npm ci                  # both  (dev, CI build stage)
npm ci --omit=dev       # prod only — this is where mistakes surface
npm ls typescript --omit=dev    # must be "(empty)"
npx depcheck                    # unused deps + undeclared imports
npm pack --dry-run              # what would be published
npx publint                     # main/module/exports/types sanity
npx @arethetypeswrong/cli --pack .   # do types resolve in ESM/CJS/bundler?
```

```dockerfile
# ── Multi-stage skeleton ────────────────────────────────────────────────────
FROM node:22-alpine AS deps
COPY package*.json ./ && RUN npm ci                # full deps, cached on lockfile

FROM node:22-alpine AS build
COPY --from=deps /app/node_modules ./node_modules
RUN npm run typecheck && npm run build             # separate lines = clearer failures

FROM node:22-alpine AS prod-deps
RUN npm ci --omit=dev                              # clean prod tree, own stage

FROM node:22-alpine AS runtime
ENV NODE_ENV=production NODE_OPTIONS="--enable-source-maps"
USER node
COPY --from=prod-deps /app/node_modules ./node_modules
COPY --from=build     /app/dist         ./dist     # WHOLE dir — not dist/*.js
CMD ["node", "dist/server.js"]
```

| Symptom | Cause | Fix |
|---|---|---|
| `Cannot find module '/app/dist/server.js'` | `rootDir` inferred → `dist/src/` | Set `rootDir: "src"`, narrow `include` |
| `Cannot find module 'typescript'` at boot | `ts-node`/`tsx` in `start` | Build to `dist/`, `start: node dist/…` |
| `ERR_MODULE_NOT_FOUND` for a real package | runtime dep in `devDependencies` | Move to `dependencies`; run the prod-deps check |
| Stack traces show `dist/*.js` | maps not built, not copied, or runtime flag off | `sourceMap: true` + `COPY dist` + `--enable-source-maps` |
| Deleted file still runs | no clean build | `rimraf dist &&` in `build` |
| Empty `dist/`, exit code 0 | stale `.tsbuildinfo` + `incremental` | Delete `*.tsbuildinfo` when cleaning |
| Type error reached production | emit-only build (esbuild/swc) | Add `tsc --noEmit` as a mandatory gate |
| Consumer sees `any` for your package | no `declaration` / no `types` | `declaration: true`; `types` FIRST in `exports` |
| `*.test.js` in `dist/` | wide `include` in the build config | Exclude tests in `tsconfig.build.json` |
| `ENOENT` for a `.sql`/`.hbs` file | `tsc` doesn't copy non-TS files | `copyfiles` in a `postbuild`/`copy:assets` step |
| 1 GB image | single-stage, dev deps kept | Multi-stage; separate `prod-deps` stage |
| `TS6059: not under rootDir` | importing above `rootDir` | Use project references (77), or move the file |
| `TS2742: cannot be named` | inferred type refs a devDependency | Annotate the return type explicitly |
| SIGTERM kills in-flight requests | no PID-1 signal forwarding | `dumb-init` + a `SIGTERM` handler |

---

## Connected topics

- **03 — tsconfig in depth** — `rootDir`, `outDir`, `target`, `module`, `declaration`, and `incremental` are all defined here; a build config is a tsconfig.
- **04 — TypeScript Node.js project setup** — where the `dev` script and the `tsx`/`ts-node` loop come from, and why they stay on the dev side of the line.
- **48 — Modules in TypeScript** — `module: NodeNext`, ESM vs CJS emit, and why import specifiers end in `.js` even in `.ts` files.
- **49 — Declaration files** — what `.d.ts` output actually contains and how consumers resolve it.
- **75 — Debugging TypeScript in VS Code** — the same source maps this doc ships to production are what bind your breakpoints locally.
- **77 — Monorepo basics with TypeScript** — `composite`, `references`, and `tsc --build`: the build pipeline scaled to many packages.
