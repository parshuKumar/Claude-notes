# 77 — Monorepo basics with TypeScript

## What is this?

A **monorepo** is one Git repository containing several independently-buildable packages, wired together by your package manager so they can import each other by name.

```
acme-platform/                      ← ONE git repo, ONE node_modules install
├── package.json                    ← the "workspace root": lists members, holds shared devDeps
├── pnpm-workspace.yaml             ← (pnpm) declares which folders are packages
├── tsconfig.base.json              ← shared compiler settings every package extends
├── tsconfig.json                   ← solution file: references only, emits nothing
├── apps/
│   ├── api/                        ← @acme/api       — Express service
│   └── worker/                     ← @acme/worker    — BullMQ consumer
└── packages/
    ├── types/                      ← @acme/types     — shared domain shapes
    ├── config/                     ← @acme/config    — env parsing
    └── db/                         ← @acme/db        — query helpers
```

Two separate mechanisms are doing the work, and conflating them is the single biggest source of monorepo confusion:

1. **Workspaces** are a *package manager* feature (npm, pnpm, yarn, bun). They make `import { User } from "@acme/types"` resolve — by symlinking `node_modules/@acme/types` → `packages/types`. The compiler has no idea workspaces exist.

2. **Project references** are a *TypeScript* feature (`composite: true` + `references`). They make `tsc -b` understand that `apps/api` depends on `packages/types`, build them in the right order, skip packages whose inputs haven't changed, and give you cross-package go-to-definition. The package manager has no idea project references exist.

```
  pnpm-workspace.yaml  →  node_modules symlinks  →  RUNTIME resolution works
  tsconfig references  →  build graph + .d.ts    →  COMPILE ordering + types work
```

You need both. Get one and not the other and you land in one of two classic broken states: types resolve but the build order is wrong (stale `.d.ts`), or the build is ordered but Node throws `ERR_MODULE_NOT_FOUND` at runtime.

---

## Why does it matter?

Because the alternative — a repo per service — makes shared shapes rot in a very specific, very expensive way.

You have an API that writes jobs onto a queue and a worker that reads them. The job payload is a shape. In a multi-repo setup that shape exists twice: once in `api/src/types/job.ts`, once in `worker/src/types/job.ts`. They start identical. Then:

- Someone adds `priority` to the API's copy. The worker's copy doesn't have it. Nothing fails to compile — the worker just silently ignores a field it doesn't know about.
- Someone renames `userId` → `accountId` in the worker. The API still sends `userId`. `undefined` flows into a database column. You find out from a support ticket three weeks later.
- Someone changes `status` from `string` to a union `"queued" | "running" | "done"`. The API still sends `"pending"`. Type-safe on both sides. Broken between them.

Every one of those is a runtime bug that a compiler *could* have caught, if only one compiler had seen both sides.

A monorepo with a shared `packages/types` makes the shape a **single artifact**. Change it, and `tsc -b` immediately fails in every consumer that no longer matches. The error arrives at the moment of the change, in the editor, from the person who made it — not in production, at 3am, for someone else.

The secondary wins are real but smaller: one `npm install`, one lint config, one CI pipeline, atomic cross-service commits and reverts, and shared tooling that only has to be configured once.

The costs are also real, and I'll be honest about them in "Going deeper" — a monorepo is not free.

---

## The JavaScript way vs the TypeScript way

```js
// ══════════════════════════════════════════════════════════════════════════════
// THE JAVASCRIPT / MULTI-REPO WAY — the shape lives in two repos and drifts
// ══════════════════════════════════════════════════════════════════════════════

// ── repo: acme-api ───────────────────────────────────────────────────────────
// src/jobs/emailJob.js
//
// "The email job payload. Keep this in sync with the worker!"   ← a comment is
//                                                                  not a contract
function enqueueWelcomeEmail(queue, user) {
  return queue.add("welcome-email", {
    userId: user.id,
    email: user.email,
    locale: user.locale ?? "en",
    priority: "high",          // ← added last Tuesday. The worker has never heard of it.
  });
}

// ── repo: acme-worker (a DIFFERENT git repo, cloned into a different folder) ──
// src/jobs/emailJob.js
//
// "The email job payload. Keep this in sync with the API!"
async function handleWelcomeEmail(job) {
  const { userId, emailAddress, locale } = job.data;
  //             ^^^^^^^^^^^^ someone renamed this here six months ago.
  //                          The API still sends `email`.
  //                          emailAddress === undefined.
  //                          sendMail(undefined) → silently drops the email.

  await sendMail(emailAddress, renderWelcome(locale));
}

// Nothing crashes. Nothing warns. No test covers the wire between the two repos,
// because no test *can* — they're separate builds with separate CI.
// The bug surfaces as "some users don't get welcome emails" in a metrics dashboard.
```

Now the same drift, with a plain `import` but still no shared package — the "I'll just copy-paste the interface" phase people hit right after adopting TypeScript:

```ts
// ══════════════════════════════════════════════════════════════════════════════
// THE HALFWAY HOUSE — types exist, but duplicated. Worse than useless.
// ══════════════════════════════════════════════════════════════════════════════

// acme-api/src/types/job.ts
export interface WelcomeEmailJob {
  userId: string;
  email: string;
  locale: string;
  priority: "high" | "normal";   // ← the API's copy
}

// acme-worker/src/types/job.ts   ← copy-pasted three months ago, then edited
export interface WelcomeEmailJob {
  userId: string;
  emailAddress: string;          // ← diverged
  locale: string;
  // priority: never added
}

// Both repos compile CLEANLY with strict: true.
// Both are internally consistent. Together they are wrong.
//
// This is the trap: duplicated types give you the *feeling* of type safety while
// removing the only thing that made it real — a single source of truth.
```

```ts
// ══════════════════════════════════════════════════════════════════════════════
// THE TYPESCRIPT / MONOREPO WAY — one shape, two consumers, one compiler
// ══════════════════════════════════════════════════════════════════════════════

// ── packages/types/src/jobs.ts — the single source of truth ──────────────────
export interface WelcomeEmailJob {
  userId: string;
  email: string;
  locale: "en" | "de" | "fr";
  priority: "high" | "normal";
}

// ── apps/api/src/jobs/enqueue.ts ─────────────────────────────────────────────
import type { WelcomeEmailJob } from "@acme/types";   // ← a REAL package name,
                                                      //   symlinked by the workspace

export async function enqueueWelcomeEmail(queue: Queue, user: User): Promise<void> {
  const payload: WelcomeEmailJob = {
    userId: user.id,
    email: user.email,
    locale: user.locale,
    priority: "high",
  };
  await queue.add("welcome-email", payload);
}

// ── apps/worker/src/jobs/welcomeEmail.ts ─────────────────────────────────────
import type { WelcomeEmailJob } from "@acme/types";   // ← the SAME file. Not a copy.

export async function handleWelcomeEmail(data: WelcomeEmailJob): Promise<void> {
  await sendMail(data.email, renderWelcome(data.locale));
  //             ^^^^^^^^^^ rename this in packages/types and BOTH sides
  //                        fail to compile, immediately, in the same commit.
}

// Now rename `email` → `emailAddress` in packages/types and run `tsc -b`:
//
//   apps/api/src/jobs/enqueue.ts:8:5 - error TS2353:
//     Object literal may only specify known properties, and 'email' does not
//     exist in type 'WelcomeEmailJob'.
//
//   apps/worker/src/jobs/welcomeEmail.ts:4:23 - error TS2339:
//     Property 'email' does not exist on type 'WelcomeEmailJob'.
//
// The drift is now a compile error instead of a support ticket.
```

The revelation: **a monorepo isn't about code sharing, it's about making cross-service contracts checkable.** The convenience of one `npm install` is a nice side effect. The actual value is that the compiler can finally see both ends of the wire at once.

---

## Syntax

```jsonc
// ── package.json (workspace ROOT) — npm / yarn / bun ─────────────────────────
{
  "name": "acme-platform",
  "private": true,                 // MANDATORY at the root — prevents accidental publish
  "workspaces": [                  // globs, relative to the root
    "apps/*",
    "packages/*"
  ],
  "scripts": {
    "build": "tsc -b",                        // build every referenced project, in order
    "build:clean": "tsc -b --clean && tsc -b",
    "typecheck": "tsc -b --dry",              // report what would build, no emit
    "dev": "npm run dev --workspaces --if-present"
  },
  "devDependencies": {
    "typescript": "^5.6.0",        // ONE typescript version for the whole repo
    "@types/node": "^22.0.0"
  }
}
```

```yaml
# ── pnpm-workspace.yaml — pnpm declares members here, NOT in package.json ─────
packages:
  - "apps/*"
  - "packages/*"
  - "!**/dist/**"        # never treat build output as a package
```

```jsonc
// ── packages/types/package.json — a workspace member ─────────────────────────
{
  "name": "@acme/types",           // ← this string is what consumers import
  "version": "0.0.0",
  "private": true,                 // internal-only: never goes to a registry
  "type": "module",
  "main": "./dist/index.js",       // CommonJS fallback entry
  "types": "./dist/index.d.ts",    // where tsc looks for the type surface
  "exports": {                     // modern resolution (Node16/NodeNext/bundler)
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.js",
      "default": "./dist/index.js"
    },
    "./jobs": {                    // subpath export: import ... from "@acme/types/jobs"
      "types": "./dist/jobs.d.ts",
      "import": "./dist/jobs.js"
    }
  },
  "scripts": { "build": "tsc -b" }
}
```

```jsonc
// ── apps/api/package.json — depends on the shared package BY NAME ────────────
{
  "name": "@acme/api",
  "private": true,
  "type": "module",
  "dependencies": {
    "@acme/types": "workspace:*",  // pnpm/yarn/bun protocol — "use the local one"
    // "@acme/types": "*",         // npm workspaces syntax (npm has no workspace: protocol)
    "express": "^4.19.0"
  }
}
```

```jsonc
// ── tsconfig.base.json — settings every package inherits ─────────────────────
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "strict": true,
    "composite": true,             // ← enables project references. Required on any
                                   //   project that is REFERENCED by another.
    "declaration": true,           // implied by composite; emit .d.ts
    "declarationMap": true,        // ← emit .d.ts.map → go-to-definition lands on .ts
    "sourceMap": true,
    "incremental": true,           // implied by composite; write .tsbuildinfo
    "skipLibCheck": true,
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true
  }
}
```

```jsonc
// ── packages/types/tsconfig.json — a leaf project (no dependencies) ──────────
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "rootDir": "src",
    "outDir": "dist",
    "tsBuildInfoFile": "dist/.tsbuildinfo"   // where the incremental state lives
  },
  "include": ["src/**/*"]
}
```

```jsonc
// ── apps/api/tsconfig.json — references what it depends on ───────────────────
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": { "rootDir": "src", "outDir": "dist" },
  "include": ["src/**/*"],
  "references": [
    { "path": "../../packages/types" }   // ← path to the DIRECTORY (or its tsconfig)
  ]
}
```

```jsonc
// ── tsconfig.json (ROOT) — the "solution" file. Builds everything. ───────────
{
  "files": [],                     // ← empty: this project itself compiles nothing
  "references": [
    { "path": "packages/types" },
    { "path": "packages/config" },
    { "path": "packages/db" },
    { "path": "apps/api" },
    { "path": "apps/worker" }
  ]
}
```

```bash
# ── tsc -b (build mode) — the only command that understands references ────────
tsc -b                       # build all referenced projects, in dependency order
tsc -b apps/api              # build apps/api and everything it depends on
tsc -b --watch               # incremental watch across the whole graph
tsc -b --clean               # delete all outputs + .tsbuildinfo files
tsc -b --force               # rebuild everything, ignoring .tsbuildinfo
tsc -b --dry                 # print what WOULD be built — great for debugging staleness
tsc -b --verbose             # explain why each project was or wasn't rebuilt
```

```bash
# ── package manager commands ─────────────────────────────────────────────────
pnpm install                            # installs + symlinks every workspace member
pnpm --filter @acme/api dev             # run a script in one package
pnpm --filter @acme/api... build        # that package AND its dependencies ("...") 
pnpm --filter ...@acme/types build      # that package and its DEPENDENTS
pnpm -r build                           # recursively, topologically ordered
pnpm add express --filter @acme/api     # add a dep to one member
pnpm add -Dw typescript                 # add a devDep to the workspace ROOT

npm install                             # same idea for npm workspaces
npm run build --workspace @acme/api
npm run build --workspaces --if-present
npm install express -w apps/api
```

---

## How it works — concept by concept

### Concept 1 — Workspaces: how `@acme/types` becomes importable

A workspace is nothing more than **a symlink your package manager creates for you.**

When you declare `"workspaces": ["apps/*", "packages/*"]` and run install, the package manager reads every member's `package.json`, notes its `name`, and links it into the root `node_modules`:

```
node_modules/
├── @acme/
│   ├── types  →  ../../packages/types      (symlink)
│   ├── config →  ../../packages/config     (symlink)
│   └── db     →  ../../packages/db         (symlink)
├── express/                                 (real, hoisted from the registry)
└── typescript/
```

From that point, standard Node resolution does the rest — `import "@acme/types"` walks up looking for `node_modules/@acme/types`, finds the symlink, follows it into `packages/types`, reads that folder's `package.json`, and uses its `exports`/`main`/`types` fields.

```ts
// apps/api/src/index.ts
import type { User } from "@acme/types";

// Resolution trace:
//  1. "@acme/types" is bare → node_modules lookup
//  2. apps/api/node_modules/@acme/types      → not found
//  3. apps/node_modules/@acme/types          → not found
//  4. <root>/node_modules/@acme/types        → FOUND (symlink → packages/types)
//  5. read packages/types/package.json
//  6. "types" / exports["."]["types"] → packages/types/dist/index.d.ts
//     ^^^^ note: dist/. The package must have been BUILT, or this file doesn't exist.
```

That last line is the thing to internalise. **Workspaces resolve to build output, not source.** A fresh clone with no build produces:

```
error TS2307: Cannot find module '@acme/types' or its corresponding type declarations.
```

...not because the symlink is missing, but because `packages/types/dist/index.d.ts` doesn't exist yet. Run `tsc -b` first. (Concept 6 covers the "point at source instead" alternative and why I don't recommend it as the default.)

One more critical rule: the dependency must be **declared**.

```jsonc
// ❌ apps/api imports @acme/types but doesn't list it
{ "name": "@acme/api", "dependencies": { "express": "^4.19.0" } }
// It STILL WORKS locally, because the symlink is in the hoisted root node_modules.
// This is called a "phantom dependency". It breaks the moment you:
//   - deploy apps/api alone
//   - switch to pnpm's strict linking
//   - run `pnpm deploy` / `npm pack`

// ✅ Declare it. Always.
{
  "name": "@acme/api",
  "dependencies": { "@acme/types": "workspace:*", "express": "^4.19.0" }
}
```

`workspace:*` (pnpm, yarn 2+, bun) means "resolve this from the workspace, and fail loudly if there's no local package by that name" — it will never silently fall back to a registry package. npm workspaces lack the protocol; use `"*"` or the member's actual version.

### Concept 2 — `composite: true`: what it actually turns on

`composite` is a bundle of guarantees that make a project safe to be *depended upon* by another project.

```jsonc
{
  "compilerOptions": {
    "composite": true
    // implies, and will error if you contradict:
    //   declaration: true        — must emit .d.ts (that IS the public surface)
    //   incremental: true        — must write .tsbuildinfo
    //   rootDir defaults to the tsconfig's directory (set it explicitly anyway)
    // additionally enforces:
    //   every input file must be matched by `include`/`files` — no implicit pickup
  }
}
```

Why each piece is required:

```
declaration: true
  A referencing project never reads your .ts files. It reads your .d.ts.
  That is the whole point — apps/api typechecks against packages/types/dist/index.d.ts,
  a tiny file, instead of re-parsing every source file in the dependency.
  On a 40-package repo this is the difference between a 4-second and a 90-second build.

incremental: true
  tsc must be able to answer "did anything in this project change since last build?"
  without redoing the work. .tsbuildinfo stores file hashes + the emit signature.

full `include` coverage
  If a file could be silently pulled in by an import, tsc couldn't know the project's
  true input set, so staleness detection would be unsound.
```

The practical consequence you'll hit first:

```
error TS6059: File '/repo/packages/types/src/user.ts' is not under 'rootDir'
              '/repo/apps/api/src'. 'rootDir' is expected to contain all source files.

error TS6307: File '/repo/packages/types/src/user.ts' is not listed within the file
              list of project '/repo/apps/api/tsconfig.json'. Projects must list all
              files or use an 'include' pattern.
```

Both mean the same thing: **you imported across a package boundary by relative path.** `import { User } from "../../../packages/types/src/user.js"` reaches into another project's source. Fix it by importing the package name (Concept 6).

### Concept 3 — `references`: the build graph

`references` tells tsc "this project consumes the *output* of that project."

```jsonc
// apps/api/tsconfig.json
{
  "references": [
    { "path": "../../packages/types" },   // directory containing a tsconfig.json
    { "path": "../../packages/db" },
    { "path": "../../packages/config/tsconfig.build.json" }  // or an explicit file
  ]
}
```

Three distinct effects, all of which people expect from workspaces alone and don't get:

```
1. BUILD ORDER
   `tsc -b apps/api` builds packages/types and packages/db FIRST, then apps/api.
   Without references, `tsc -p apps/api` compiles against whatever stale .d.ts
   happens to be sitting in packages/types/dist — possibly from last week.

2. STALENESS PROPAGATION
   Touch packages/types/src/user.ts → tsc knows apps/api and apps/worker are now
   out of date, and rebuilds exactly those. Untouched packages are skipped entirely.

3. EDITOR BEHAVIOUR
   The language server loads referenced projects, so "Find All References" on a
   type in packages/types finds usages in apps/api — across the package boundary.
```

An important subtlety: **references and `dependencies` must agree.** They're two views of the same graph and nothing automatically keeps them in sync.

```jsonc
// package.json says apps/api depends on @acme/db ...
{ "dependencies": { "@acme/types": "workspace:*", "@acme/db": "workspace:*" } }

// ... but tsconfig.json forgot it:
{ "references": [{ "path": "../../packages/types" }] }   // ❌ missing db

// Symptom: `tsc -b apps/api` succeeds against a STALE packages/db/dist/index.d.ts,
// or fails with "Cannot find module '@acme/db'" on a clean checkout.
```

Tooling exists to enforce this (`@monorepo-utils/workspaces-to-typescript-project-references`, Nx's `@nx/dependency-checks`, or a 20-line CI script). At minimum, make it a code-review habit: *add a dependency, add a reference.*

### Concept 4 — `tsc -b`: build mode, and how it differs from `tsc -p`

```bash
tsc -p apps/api/tsconfig.json    # "compile THIS project's files, now"
                                 # - ignores `references` for the purposes of BUILDING
                                 # - reads referenced projects' .d.ts as-is (stale or not)
                                 # - will refuse to emit if it needs a missing .d.ts

tsc -b apps/api                  # "make this project up to date"
                                 # - walks the reference graph
                                 # - topologically sorts it
                                 # - for each project: compare inputs vs .tsbuildinfo,
                                 #   skip if unchanged, otherwise compile
                                 # - stops the whole build on the first project that fails
```

`--verbose` is how you learn what it's doing:

```bash
$ tsc -b --verbose
Projects in this build:
    * packages/types/tsconfig.json
    * packages/db/tsconfig.json
    * apps/api/tsconfig.json

Project 'packages/types/tsconfig.json' is out of date because output
  'packages/types/dist/.tsbuildinfo' is older than input 'packages/types/src/user.ts'
Building project '/repo/packages/types/tsconfig.json'...

Project 'packages/db/tsconfig.json' is up to date with .d.ts files from its dependencies
  ↑ THIS LINE IS THE PAYOFF. types changed, but its emitted .d.ts did NOT change
    (you edited a comment, or a function body), so db doesn't need rebuilding.

Project 'apps/api/tsconfig.json' is out of date because output file
  'apps/api/dist/.tsbuildinfo' is older than input 'apps/api/src/routes/users.ts'
Building project '/repo/apps/api/tsconfig.json'...
```

That "up to date with .d.ts files from its dependencies" check is called **emit-signature-based invalidation**: tsc hashes the *emitted declarations*, so a change that doesn't alter the public type surface doesn't cascade. Turning on `"assumeChangesOnlyAffectDirectDependencies": true` makes this more aggressive (and slightly less sound) on very large graphs.

`--dry` is your staleness debugger:

```bash
$ tsc -b --dry
# lists every project it would build. If a package you just edited is NOT listed,
# your `include` globs don't cover the file you changed.
```

### Concept 5 — `declarationMap`: go-to-definition into source

Without it, cross-package navigation is a dead end:

```ts
// apps/api/src/routes/users.ts
import type { User } from "@acme/types";

const u: User = ...;
//       ^^^^ Cmd+Click here
```

```
declarationMap: false  →  lands in packages/types/dist/index.d.ts
                          A generated file. Read-only in spirit. No JSDoc bodies,
                          no implementation, no comments you can edit.
                          You then manually navigate to src/ and find the type by hand.

declarationMap: true   →  lands in packages/types/src/user.ts, exact line and column.
                          Editable. Rename-refactor works ACROSS packages.
                          Find-All-References returns real source locations.
```

The mechanism mirrors source maps exactly:

```jsonc
// packages/types/tsconfig.json
{ "compilerOptions": { "declaration": true, "declarationMap": true } }
```

```ts
// emitted: packages/types/dist/index.d.ts
export interface User { id: string; email: string; }
//# sourceMappingURL=index.d.ts.map
```

```jsonc
// emitted: packages/types/dist/index.d.ts.map
{
  "version": 3,
  "file": "index.d.ts",
  "sourceRoot": "",
  "sources": ["../src/index.ts"],   // ← relative to the .map file
  "names": [],
  "mappings": "AAAA,MAAM,WAAW,IAAI..."
}
```

Two rules follow:

```
RULE 1: the .d.ts.map must be NEXT TO the .d.ts, and ../src/index.ts must exist
        on disk. Inside a Docker image that copied only dist/, navigation dies —
        harmless, since you don't Cmd+Click inside a container.

RULE 2: if you PUBLISH the package, you must ship src/ too, or consumers get a
        broken map. See Concept 10.
```

`declarationMap: true` in `tsconfig.base.json` costs almost nothing at build time and is the single highest-value quality-of-life setting in a TypeScript monorepo. Turn it on before anything else.

### Concept 6 — Path aliases vs real workspace package names

This is the fork in the road, and the wrong choice is very common.

```jsonc
// ── APPROACH A: path aliases (tsconfig `paths`) ──────────────────────────────
// tsconfig.base.json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@acme/types": ["packages/types/src/index.ts"],
      "@acme/types/*": ["packages/types/src/*"]
    }
  }
}
```

```ts
// apps/api/src/index.ts
import type { User } from "@acme/types";   // compiles fine — tsc rewrites the lookup
```

It looks identical to the real thing, and for typechecking it *is*. Then you run it:

```bash
$ node dist/index.js
Error [ERR_MODULE_NOT_FOUND]: Cannot find package '@acme/types' imported from /repo/apps/api/dist/index.js
```

Because **`paths` is a compile-time-only fiction.** tsc resolves it, then emits `import "@acme/types"` verbatim into the JavaScript. Node has no `paths`. Neither does Jest, or `node --test`, or your bundler unless separately configured.

```
Every consumer of your code must be told about `paths` SEPARATELY:
  tsc          ✅ reads tsconfig paths
  node         ❌ needs tsconfig-paths / tsx / a bundler / imports-field mapping
  jest         ❌ needs moduleNameMapper duplicating every alias
  vitest       ❌ needs resolve.alias duplicating every alias
  eslint       ❌ needs eslint-import-resolver-typescript
  esbuild/rollup ❌ needs a plugin
  webpack      ❌ needs resolve.alias

That is six-plus places to keep one alias list in sync. Every new alias is six edits.
```

```jsonc
// ── APPROACH B: real workspace package names (RECOMMENDED) ───────────────────
// packages/types/package.json
{ "name": "@acme/types", "types": "./dist/index.d.ts", "exports": { ... } }

// apps/api/package.json
{ "dependencies": { "@acme/types": "workspace:*" } }

// apps/api/tsconfig.json
{ "references": [{ "path": "../../packages/types" }] }
// no `paths`, no `baseUrl`
```

```
Every consumer resolves it with ZERO extra configuration:
  tsc          ✅ node_modules symlink + package.json "types"/"exports"
  node         ✅ node_modules symlink
  jest         ✅
  vitest       ✅
  eslint       ✅
  any bundler  ✅
  Docker       ✅ (copy node_modules, or `pnpm deploy`)
```

**Package names win.** The rule: use `paths` for *intra*-package convenience (`"#/*": ["./src/*"]` inside one package, and even then prefer Node's native `imports` field), never for *inter*-package resolution.

The one honest counter-argument: package names require a build before cross-package types resolve, which slows the first-clone experience. The mitigation is not `paths` — it's `tsc -b` in a `postinstall`/`prepare` script, or the "publishConfig swap" pattern:

```jsonc
// packages/types/package.json — point at SOURCE for local dev, DIST when published
{
  "name": "@acme/types",
  "main": "./src/index.ts",              // internal consumers get source directly
  "types": "./src/index.ts",             //   → no build needed, instant feedback
  "publishConfig": {                     // npm rewrites these fields on `npm publish`
    "main": "./dist/index.js",
    "types": "./dist/index.d.ts"
  }
}
```

This works beautifully **if every consumer transpiles the dependency too** (tsx, bundler, Vite). It does *not* work if `apps/api` is compiled by plain `tsc` to `dist/` and run by `node`, because Node will try to `require("@acme/types/src/index.ts")`. Know which world you're in before adopting it. For a plain Node backend, stick with `dist` + project references.

### Concept 7 — Incremental builds and `tsBuildInfoFile`

`.tsbuildinfo` is the cache that makes `tsc -b` fast. It holds, per project:

- the resolved list of input files and their content hashes,
- the compiler options used (any change → full rebuild),
- the *emit signature* of each referenced project's `.d.ts`,
- enough semantic info to skip re-checking untouched files.

```jsonc
{
  "compilerOptions": {
    "incremental": true,                      // implied by composite
    "tsBuildInfoFile": "dist/.tsbuildinfo"    // explicit > default
  }
}
```

Where it lands by default is a genuine footgun:

```
outDir set, tsBuildInfoFile unset  →  <outDir>/<tsconfig-basename>.tsbuildinfo
                                      e.g. dist/tsconfig.tsbuildinfo   ✅ fine

noEmit: true, tsBuildInfoFile unset →  next to the tsconfig, as
                                      .tsbuildinfo  — easy to forget to .gitignore

multiple tsconfigs sharing an outDir →  tsconfig.tsbuildinfo COLLIDES.
                                        tsconfig.json and tsconfig.build.json both
                                        writing dist/tsconfig*.tsbuildinfo will
                                        thrash each other into permanent rebuilds.
```

Always set it explicitly, one per project config:

```jsonc
// packages/types/tsconfig.json
{ "compilerOptions": { "outDir": "dist", "tsBuildInfoFile": "dist/.tsbuildinfo" } }

// packages/types/tsconfig.test.json
{ "compilerOptions": { "noEmit": true, "tsBuildInfoFile": "dist/.tsbuildinfo.test" } }
```

```gitignore
# .gitignore — never commit these
**/dist/
**/*.tsbuildinfo
**/.tsbuildinfo*
```

And the CI caching payoff:

```yaml
# .github/workflows/ci.yml (fragment)
- uses: actions/cache@v4
  with:
    path: |
      **/dist
      **/.tsbuildinfo
    key: tsbuild-${{ runner.os }}-${{ hashFiles('**/package-lock.json') }}-${{ github.sha }}
    restore-keys: tsbuild-${{ runner.os }}-${{ hashFiles('**/package-lock.json') }}-
# A PR touching only apps/worker then rebuilds only apps/worker.
```

Two failure modes worth memorising:

```bash
# "It says up to date but the output is obviously wrong"
tsc -b --force            # ignore .tsbuildinfo, rebuild everything

# "Something is deeply confused after a branch switch / dependency bump"
tsc -b --clean && tsc -b  # delete all outputs and buildinfo, start clean
```

`.tsbuildinfo` tracks file *timestamps and hashes*, so `git checkout` of an older branch can produce files newer than the buildinfo (fine) or older (also fine) — but a `git stash` cycle plus a partially-cleaned `dist/` can leave orphaned `.js` files that nothing rebuilds. `--clean` exists for exactly that.

### Concept 8 — `tsconfig.base.json` and the extends chain

One base, many thin leaves. The rule for what goes where:

```
tsconfig.base.json          → everything PATH-INDEPENDENT
                              target, module, strict flags, lib, composite,
                              declarationMap, skipLibCheck, esModuleInterop

<package>/tsconfig.json     → everything PATH-DEPENDENT
                              rootDir, outDir, tsBuildInfoFile, include, exclude,
                              references, and per-package overrides (types, lib)
```

Why: `extends` resolves relative paths in the *base* file relative to the **base file's own directory**, which is almost never what you want for `outDir`.

```jsonc
// ❌ tsconfig.base.json
{ "compilerOptions": { "outDir": "dist", "rootDir": "src" } }
// Every package now emits to <repo-root>/dist, all on top of each other.
// (Relative paths in `extends`-ed compilerOptions are resolved from the BASE file.)
```

```jsonc
// ✅ tsconfig.base.json — no paths at all
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "lib": ["ES2022"],
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "exactOptionalPropertyTypes": true,
    "composite": true,
    "declarationMap": true,
    "sourceMap": true,
    "skipLibCheck": true,
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "verbatimModuleSyntax": true
  }
}
```

```jsonc
// ✅ each package sets its own paths
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "rootDir": "src",
    "outDir": "dist",
    "tsBuildInfoFile": "dist/.tsbuildinfo"
  },
  "include": ["src/**/*"],
  "exclude": ["dist", "**/*.test.ts"]
}
```

Layering more than one base is fine and often useful:

```
tsconfig.base.json            strict + composite + emit settings
  └─ tsconfig.node.json       + "types": ["node"], lib ES2022        → apps/api, apps/worker, packages/db
  └─ tsconfig.lib.json        + "types": [], stricter isolatedDeclarations → packages/types
```

```jsonc
// tsconfig.node.json
{
  "extends": "./tsconfig.base.json",
  "compilerOptions": { "types": ["node"], "module": "NodeNext" }
}

// apps/api/tsconfig.json
{ "extends": "../../tsconfig.node.json", "compilerOptions": { "outDir": "dist", "rootDir": "src" } }
```

Note `extends` accepts an array in TS 5.0+, applied left to right with later entries winning:

```jsonc
{ "extends": ["../../tsconfig.base.json", "../../tsconfig.node.json"] }
```

### Concept 9 — Build ordering and circular dependency traps

The reference graph must be a **DAG**. tsc enforces this loudly:

```
error TS6202: Project references may not form a circular graph. Cycle detected:
  /repo/packages/db/tsconfig.json
  /repo/packages/types/tsconfig.json
  /repo/packages/db/tsconfig.json
```

How the cycle usually appears — and it's always the same story:

```ts
// packages/types/src/user.ts
import type { QueryResult } from "@acme/db";   // ❌ types now depends on db
export interface User { id: string; row: QueryResult<"users">; }

// packages/db/src/query.ts
import type { User } from "@acme/types";       // ❌ db already depends on types
export async function findUser(id: string): Promise<User> { ... }
```

Three fixes, in order of preference:

```
FIX 1 — Push the shared thing DOWN into the leaf.
        Move QueryResult into @acme/types. `types` depends on nothing;
        `db` depends on `types`. Cycle gone.

        BEST RULE: packages/types must have ZERO workspace dependencies.
        Make it a hard, enforced constraint. It is the root of your graph.

FIX 2 — Extract a new lower package.
        types ──┐
                ├──> @acme/core-primitives   (Brand, Result, Id, Timestamp)
        db ─────┘

FIX 3 — Invert with a generic parameter, so the low package never names the high one.
        // packages/types/src/user.ts — no import at all
        export interface User<TRow = unknown> { id: string; row: TRow; }
        // packages/db names the concrete type; types stays a leaf.
```

Layering discipline is the real cure. Write it down and lint it:

```
Layer 0  packages/types      pure shapes, zero workspace deps
Layer 1  packages/config     depends on: types
Layer 2  packages/db         depends on: types, config
Layer 3  packages/domain     depends on: types, config, db
Layer 4  apps/api            depends on: anything below
Layer 4  apps/worker         depends on: anything below

Rule: an import may only point DOWNWARD. Never sideways between apps.
```

```jsonc
// enforce it with eslint-plugin-import (or dependency-cruiser / eslint-plugin-boundaries)
{
  "rules": {
    "import/no-restricted-paths": ["error", {
      "zones": [
        { "target": "./packages/types", "from": "./packages", "except": ["./types"],
          "message": "packages/types must be a leaf — it may not import other packages." },
        { "target": "./apps/api", "from": "./apps/worker",
          "message": "Apps must not import each other. Extract a package." }
      ]
    }]
  }
}
```

A separate trap: **file-level cycles within one package** are legal for TypeScript (types are erased) but can explode at runtime with ESM/CJS interop — `undefined` at module-evaluation time. `import type` is always safe (it's erased entirely); a value import in a cycle is not.

```ts
// ✅ safe even in a cycle — erased at compile time
import type { User } from "./user.js";

// ⚠️ a runtime cycle — evaluation order now matters
import { createUser } from "./user.js";
```

Setting `verbatimModuleSyntax: true` forces you to write `import type` explicitly, which makes these cycles visible instead of accidental.

### Concept 10 — Publishing vs internal-only packages

Decide this **per package**, and encode the decision in `package.json`.

```jsonc
// ── INTERNAL-ONLY (the default for a backend monorepo) ───────────────────────
{
  "name": "@acme/types",
  "version": "0.0.0",          // never bumped; nobody consumes a version
  "private": true,             // ← npm publish REFUSES to run. This is the guard.
  "type": "module",
  "main": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": { ".": { "types": "./dist/index.d.ts", "import": "./dist/index.js" } }
}
// Consumers are only ever other workspace members, resolved via symlink.
// You never think about semver, changelogs, or registry auth.
```

```jsonc
// ── PUBLISHED (an SDK, a client library, an open-source utility) ─────────────
{
  "name": "@acme/api-client",
  "version": "2.4.1",          // real semver, bumped on every release
  // NO "private": true
  "type": "module",
  "main": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": { "types": "./dist/index.d.ts", "import": "./dist/index.js", "require": "./dist/index.cjs" }
  },
  "files": [                   // ← EXACTLY what gets tarballed
    "dist",
    "src"                      // ← ship src too, or declarationMap points at nothing
  ],
  "sideEffects": false,        // lets bundlers tree-shake consumers
  "publishConfig": { "access": "public", "provenance": true },
  "peerDependencies": { "zod": "^3.23.0" },
  "scripts": {
    "prepublishOnly": "tsc -b --clean && tsc -b"   // never publish a stale dist/
  }
}
```

The `workspace:*` protocol matters here. If a published package depends on another workspace package:

```jsonc
// packages/api-client/package.json (source, in git)
{ "dependencies": { "@acme/types": "workspace:^" } }

// what pnpm/yarn actually publishes to the registry:
{ "dependencies": { "@acme/types": "^0.4.2" } }
//                                  ^^^^^^^ rewritten at pack time to the real
//                                  version of the local package.
// ⚠️ Which means @acme/types must ALSO be published, and NOT be `private: true`.
// Publishing a package that depends on a private workspace package produces a
// tarball nobody can install. This is the #1 monorepo publishing failure.
```

The checklist that prevents it:

```
Before publishing package X:
  □ X is not `private: true`
  □ every workspace dependency of X is also published (recursively)
  □ `files` includes dist/ — and src/ if declarationMap is on
  □ `exports` has a "types" condition FIRST in each entry (order matters!)
  □ `npm pack --dry-run` output contains exactly what you expect
  □ prepublishOnly does a clean build
  □ a changeset / version bump exists
```

For versioning across many published packages, use **Changesets** (`@changesets/cli`): contributors write a markdown file describing the change and its semver impact, CI aggregates them into version bumps + changelogs + a publish. It is the de facto standard and worth adopting the day your first package goes public.

```bash
pnpm add -Dw @changesets/cli
pnpm changeset init
pnpm changeset            # interactive: pick packages + bump type, write a summary
pnpm changeset version    # apply bumps, rewrite changelogs, update internal ranges
pnpm changeset publish    # build then npm publish everything that changed
```

---

## Example 1 — basic

Three packages: shared types, an API that produces jobs, a worker that consumes them. Everything below is a complete, working set of files.

```
acme-basic/
├── package.json
├── pnpm-workspace.yaml
├── tsconfig.base.json
├── tsconfig.json
├── packages/types/
│   ├── package.json
│   ├── tsconfig.json
│   └── src/index.ts
├── apps/api/
│   ├── package.json
│   ├── tsconfig.json
│   └── src/index.ts
└── apps/worker/
    ├── package.json
    ├── tsconfig.json
    └── src/index.ts
```

```jsonc
// ── package.json (root) ─────────────────────────────────────────────────────
{
  "name": "acme-basic",
  "private": true,                     // root is never published
  "type": "module",
  "scripts": {
    "build": "tsc -b",
    "clean": "tsc -b --clean",
    "typecheck": "tsc -b --dry",
    "start:api": "node apps/api/dist/index.js",
    "start:worker": "node apps/worker/dist/index.js"
  },
  "devDependencies": {
    "typescript": "^5.6.0",            // ONE compiler version, at the root
    "@types/node": "^22.0.0"
  }
}
```

```yaml
# ── pnpm-workspace.yaml ──────────────────────────────────────────────────────
packages:
  - "apps/*"
  - "packages/*"
```

```jsonc
// ── tsconfig.base.json — no paths, only path-INDEPENDENT settings ───────────
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["ES2022"],
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "types": ["node"],

    "strict": true,
    "noUncheckedIndexedAccess": true,
    "verbatimModuleSyntax": true,      // forces explicit `import type`

    "composite": true,                 // ← makes every package referenceable
    "declaration": true,
    "declarationMap": true,            // ← Cmd+Click lands on .ts, not .d.ts
    "sourceMap": true,

    "skipLibCheck": true,
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true
  }
}
```

```jsonc
// ── tsconfig.json (root solution file) ──────────────────────────────────────
{
  "files": [],                         // this project compiles nothing itself
  "references": [
    { "path": "packages/types" },
    { "path": "apps/api" },
    { "path": "apps/worker" }
  ]
}
// `tsc -b` at the root now builds all three, in dependency order.
```

```jsonc
// ── packages/types/package.json ─────────────────────────────────────────────
{
  "name": "@acme/types",               // ← the import specifier consumers use
  "version": "0.0.0",
  "private": true,                     // internal-only
  "type": "module",
  "main": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",    // "types" must come FIRST
      "import": "./dist/index.js"
    }
  }
}
```

```jsonc
// ── packages/types/tsconfig.json — a LEAF: no references ────────────────────
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "rootDir": "src",
    "outDir": "dist",
    "tsBuildInfoFile": "dist/.tsbuildinfo"
  },
  "include": ["src/**/*"]
}
```

```ts
// ── packages/types/src/index.ts — the single source of truth ────────────────
// This package has ZERO workspace dependencies. That is a deliberate rule:
// it is the root of the dependency graph, so nothing can create a cycle with it.

/** A user as stored and as returned by the API. */
export interface User {
  readonly id: string;
  readonly email: string;
  readonly displayName: string;
  readonly locale: Locale;
  readonly createdAt: string;          // ISO-8601; JSON has no Date
}

export type Locale = "en" | "de" | "fr";

/** The payload written to the queue by the API and read by the worker. */
export interface WelcomeEmailJob {
  readonly userId: string;
  readonly email: string;
  readonly locale: Locale;
  readonly priority: "high" | "normal";
}

/** Every job name the system knows about, mapped to its payload type. */
export interface JobPayloads {
  "welcome-email": WelcomeEmailJob;
  "password-reset": { readonly userId: string; readonly token: string };
}

export type JobName = keyof JobPayloads;
```

```jsonc
// ── apps/api/package.json ───────────────────────────────────────────────────
{
  "name": "@acme/api",
  "version": "0.0.0",
  "private": true,
  "type": "module",
  "main": "./dist/index.js",
  "dependencies": {
    "@acme/types": "workspace:*"       // ← DECLARE it; don't rely on hoisting
  },
  "scripts": { "build": "tsc -b" }
}
```

```jsonc
// ── apps/api/tsconfig.json — references what it depends on ──────────────────
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "rootDir": "src",
    "outDir": "dist",
    "tsBuildInfoFile": "dist/.tsbuildinfo"
  },
  "include": ["src/**/*"],
  "references": [
    { "path": "../../packages/types" } // ← mirrors the `dependencies` entry above
  ]
}
```

```ts
// ── apps/api/src/index.ts ───────────────────────────────────────────────────
import type { JobName, JobPayloads, User, WelcomeEmailJob } from "@acme/types";
//            ^^^^^^^^ resolved via node_modules/@acme/types → packages/types
//                     Cmd+Click lands on packages/types/src/index.ts (declarationMap)

/** A tiny stand-in for BullMQ so the example runs with zero dependencies. */
const queue: Array<{ name: JobName; data: JobPayloads[JobName] }> = [];

/**
 * Note the generic: the payload type is derived from the job NAME.
 * Pass "welcome-email" with a password-reset payload and it won't compile.
 */
function enqueue<N extends JobName>(name: N, data: JobPayloads[N]): void {
  queue.push({ name, data });
  console.log(`[api] enqueued ${name}`, data);
}

export function registerUser(user: User): void {
  const job: WelcomeEmailJob = {
    userId: user.id,
    email: user.email,
    locale: user.locale,
    priority: "high",
  };
  enqueue("welcome-email", job);

  // ❌ enqueue("welcome-email", { userId: user.id, token: "abc" });
  //    error TS2345: Argument of type '{ userId: string; token: string; }' is not
  //    assignable to parameter of type 'WelcomeEmailJob'.
  //    ← this is the error a multi-repo setup can never produce
}

registerUser({
  id: "usr_1",
  email: "ada@example.com",
  displayName: "Ada",
  locale: "en",
  createdAt: new Date().toISOString(),
});
```

```jsonc
// ── apps/worker/package.json ────────────────────────────────────────────────
{
  "name": "@acme/worker",
  "version": "0.0.0",
  "private": true,
  "type": "module",
  "main": "./dist/index.js",
  "dependencies": { "@acme/types": "workspace:*" },
  "scripts": { "build": "tsc -b" }
}
```

```jsonc
// ── apps/worker/tsconfig.json ───────────────────────────────────────────────
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "rootDir": "src",
    "outDir": "dist",
    "tsBuildInfoFile": "dist/.tsbuildinfo"
  },
  "include": ["src/**/*"],
  "references": [{ "path": "../../packages/types" }]
}
```

```ts
// ── apps/worker/src/index.ts ────────────────────────────────────────────────
import type { JobPayloads, Locale, WelcomeEmailJob } from "@acme/types";
//                                  ^^^^^^^^^^^^^^^ the SAME interface the API used.
//                                  Not a copy. Not "kept in sync". The same file.

const templates: Record<Locale, string> = {
  en: "Welcome, %s!",
  de: "Willkommen, %s!",
  fr: "Bienvenue, %s !",
  // Add "es" to Locale in packages/types and THIS OBJECT stops compiling:
  //   error TS2741: Property 'es' is missing in type ...
  // Exhaustiveness across a package boundary — the whole point of the monorepo.
};

export async function handleWelcomeEmail(data: WelcomeEmailJob): Promise<void> {
  const subject = templates[data.locale].replace("%s", data.email);
  console.log(`[worker] priority=${data.priority} → ${subject}`);
}

export async function dispatch<N extends keyof JobPayloads>(
  name: N,
  data: JobPayloads[N],
): Promise<void> {
  switch (name) {
    case "welcome-email":
      return handleWelcomeEmail(data as WelcomeEmailJob);
    default:
      throw new Error(`unhandled job: ${String(name)}`);
  }
}
```

```bash
# ── Run it ──────────────────────────────────────────────────────────────────
$ pnpm install
# creates node_modules/@acme/types → packages/types symlink

$ pnpm build            # == tsc -b at the root
# builds packages/types, THEN apps/api and apps/worker (which now have a .d.ts)

$ node apps/api/dist/index.js
[api] enqueued welcome-email { userId: 'usr_1', email: 'ada@example.com', ... }

# Now prove the contract. Edit packages/types/src/index.ts:
#   locale: Locale  →  language: Locale
$ pnpm build
apps/api/src/index.ts:22:5 - error TS2353: Object literal may only specify known
  properties, and 'locale' does not exist in type 'WelcomeEmailJob'.
apps/worker/src/index.ts:19:38 - error TS2339: Property 'locale' does not exist
  on type 'WelcomeEmailJob'.

# Two errors, one commit, zero production incidents.
```

---

## Example 2 — real world backend use case

A five-package platform: shared types, env config, a DB layer, an Express API, and a BullMQ worker — with a realistic build pipeline, dev loop, test setup, Docker deploy, and CI.

```
acme-platform/
├── package.json
├── pnpm-workspace.yaml
├── tsconfig.base.json          # strict + composite + declarationMap
├── tsconfig.node.json          # + node types      (extends base)
├── tsconfig.json               # solution file
├── .gitignore
├── packages/
│   ├── types/                  # @acme/types   — layer 0, zero workspace deps
│   ├── config/                 # @acme/config  — layer 1, deps: types
│   └── db/                     # @acme/db      — layer 2, deps: types, config
└── apps/
    ├── api/                    # @acme/api     — layer 3
    └── worker/                 # @acme/worker  — layer 3
```

```jsonc
// ══ package.json (root) ═════════════════════════════════════════════════════
{
  "name": "acme-platform",
  "private": true,
  "type": "module",
  "engines": { "node": ">=20.11" },
  "packageManager": "pnpm@9.12.0",       // pins the PM via corepack — do this
  "scripts": {
    // ── build ──────────────────────────────────────────────────────────────
    "build": "tsc -b",
    "build:force": "tsc -b --force",
    "build:clean": "tsc -b --clean",
    "typecheck": "tsc -b --dry",

    // ── dev: run both apps concurrently, each in watch mode ────────────────
    // `tsc -b --watch` in one terminal keeps every package's dist/ fresh,
    // while node --watch restarts the apps when dist/ changes.
    "dev": "concurrently -n tsc,api,worker -c blue,green,magenta \"pnpm dev:tsc\" \"pnpm dev:api\" \"pnpm dev:worker\"",
    "dev:tsc": "tsc -b --watch --preserveWatchOutput",
    "dev:api": "node --watch --enable-source-maps apps/api/dist/index.js",
    "dev:worker": "node --watch --enable-source-maps apps/worker/dist/index.js",

    // ── quality ────────────────────────────────────────────────────────────
    "test": "vitest run",
    "lint": "eslint . --max-warnings 0",

    // ── targeted work: build/run one app AND its dependencies ("...") ──────
    "api": "pnpm --filter @acme/api...",
    "worker": "pnpm --filter @acme/worker..."
  },
  "devDependencies": {
    "typescript": "^5.6.0",
    "@types/node": "^22.0.0",
    "concurrently": "^9.0.0",
    "vitest": "^2.1.0",
    "eslint": "^9.12.0"
  }
}
```

```jsonc
// ══ tsconfig.json (root solution) ═══════════════════════════════════════════
// Order here is cosmetic — tsc topologically sorts from the reference graph —
// but listing by layer documents the architecture.
{
  "files": [],
  "references": [
    { "path": "packages/types" },
    { "path": "packages/config" },
    { "path": "packages/db" },
    { "path": "apps/api" },
    { "path": "apps/worker" }
  ]
}
```

```jsonc
// ══ tsconfig.node.json ══════════════════════════════════════════════════════
{
  "extends": "./tsconfig.base.json",
  "compilerOptions": {
    "types": ["node"],
    "lib": ["ES2022"]
  }
}
```

```ts
// ══ packages/types/src/index.ts ═════════════════════════════════════════════
export * from "./user.js";      // .js extension: required by NodeNext resolution
export * from "./order.js";     // even though the file on disk is .ts
export * from "./jobs.js";
export * from "./result.js";
```

```ts
// ══ packages/types/src/user.ts ══════════════════════════════════════════════
export type UserId = string & { readonly __brand: "UserId" };
export type OrderId = string & { readonly __brand: "OrderId" };

export const asUserId = (raw: string): UserId => raw as UserId;

export type Locale = "en" | "de" | "fr";

export interface User {
  readonly id: UserId;
  readonly email: string;
  readonly displayName: string;
  readonly locale: Locale;
  readonly createdAt: string;
}

/** What the HTTP layer accepts. Deliberately different from `User`. */
export interface CreateUserRequest {
  readonly email: string;
  readonly displayName: string;
  readonly locale?: Locale;
}

/** What the HTTP layer returns — no internal fields leak. */
export interface UserResponse {
  readonly id: string;
  readonly email: string;
  readonly displayName: string;
}
```

```ts
// ══ packages/types/src/jobs.ts ══════════════════════════════════════════════
import type { Locale, OrderId, UserId } from "./user.js";

export interface WelcomeEmailJob {
  readonly userId: UserId;
  readonly email: string;
  readonly locale: Locale;
}

export interface OrderSettlementJob {
  readonly orderId: OrderId;
  readonly attempt: number;
}

/**
 * The registry that makes the whole queue type-safe.
 * The API can only enqueue a payload matching the name; the worker's handler
 * map must cover every key. Add a job here and BOTH sides fail to compile
 * until they're updated — which is exactly what you want.
 */
export interface JobPayloads {
  "welcome-email": WelcomeEmailJob;
  "order-settlement": OrderSettlementJob;
}

export type JobName = keyof JobPayloads;
```

```ts
// ══ packages/types/src/result.ts ════════════════════════════════════════════
export type Result<T, E = Error> =
  | { readonly ok: true; readonly value: T }
  | { readonly ok: false; readonly error: E };

export const ok = <T>(value: T): Result<T, never> => ({ ok: true, value });
export const err = <E>(error: E): Result<never, E> => ({ ok: false, error });
```

```jsonc
// ══ packages/config/package.json ════════════════════════════════════════════
{
  "name": "@acme/config",
  "version": "0.0.0",
  "private": true,
  "type": "module",
  "main": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": { ".": { "types": "./dist/index.d.ts", "import": "./dist/index.js" } },
  "dependencies": { "@acme/types": "workspace:*", "zod": "^3.23.0" }
}
```

```jsonc
// ══ packages/config/tsconfig.json ═══════════════════════════════════════════
{
  "extends": "../../tsconfig.node.json",
  "compilerOptions": {
    "rootDir": "src",
    "outDir": "dist",
    "tsBuildInfoFile": "dist/.tsbuildinfo"
  },
  "include": ["src/**/*"],
  "exclude": ["**/*.test.ts"],
  "references": [{ "path": "../types" }]     // mirrors dependencies above
}
```

```ts
// ══ packages/config/src/index.ts ════════════════════════════════════════════
import { z } from "zod";
import type { Result } from "@acme/types";
import { err, ok } from "@acme/types";
//     ^^^^^^^^ a VALUE import (not `import type`) — needs the built dist/index.js
//              at runtime, which is why references + build order matter.

const schema = z.object({
  NODE_ENV: z.enum(["development", "test", "production"]).default("development"),
  PORT: z.coerce.number().int().positive().default(3000),
  DATABASE_URL: z.string().url(),
  REDIS_URL: z.string().url(),
  LOG_LEVEL: z.enum(["debug", "info", "warn", "error"]).default("info"),
});

/** The single, shared, validated config shape for every app in the repo. */
export type AppConfig = z.infer<typeof schema>;

export function loadConfig(env: NodeJS.ProcessEnv): Result<AppConfig, string> {
  const parsed = schema.safeParse(env);
  if (!parsed.success) {
    const issues = parsed.error.issues
      .map((i) => `${i.path.join(".")}: ${i.message}`)
      .join("; ");
    return err(`invalid environment: ${issues}`);
  }
  return ok(parsed.data);
}

/** Fail-fast variant for app entry points. */
export function loadConfigOrExit(env: NodeJS.ProcessEnv): AppConfig {
  const result = loadConfig(env);
  if (!result.ok) {
    console.error(result.error);
    process.exit(1);
  }
  return result.value;
}
```

```jsonc
// ══ packages/db/package.json ════════════════════════════════════════════════
{
  "name": "@acme/db",
  "version": "0.0.0",
  "private": true,
  "type": "module",
  "main": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": { "types": "./dist/index.d.ts", "import": "./dist/index.js" },
    "./testing": {                       // subpath: test helpers, not in the main surface
      "types": "./dist/testing/index.d.ts",
      "import": "./dist/testing/index.js"
    }
  },
  "dependencies": {
    "@acme/types": "workspace:*",
    "@acme/config": "workspace:*",
    "pg": "^8.13.0"
  },
  "devDependencies": { "@types/pg": "^8.11.0" }
}
```

```jsonc
// ══ packages/db/tsconfig.json ═══════════════════════════════════════════════
{
  "extends": "../../tsconfig.node.json",
  "compilerOptions": {
    "rootDir": "src",
    "outDir": "dist",
    "tsBuildInfoFile": "dist/.tsbuildinfo"
  },
  "include": ["src/**/*"],
  "exclude": ["**/*.test.ts"],
  "references": [
    { "path": "../types" },
    { "path": "../config" }
  ]
}
```

```ts
// ══ packages/db/src/index.ts ════════════════════════════════════════════════
import pg from "pg";
import type { AppConfig } from "@acme/config";
import type { User, UserId } from "@acme/types";
import { asUserId } from "@acme/types";

export interface UserRepository {
  findById(id: UserId): Promise<User | null>;
  insert(user: Omit<User, "createdAt">): Promise<User>;
}

/** The row shape as it comes back from Postgres — snake_case, all strings. */
interface UserRow {
  id: string;
  email: string;
  display_name: string;
  locale: string;
  created_at: Date;
}

/**
 * The mapping from DB row → domain type lives HERE, once, in the package that
 * owns the database. apps/api and apps/worker never see snake_case.
 */
function toUser(row: UserRow): User {
  return {
    id: asUserId(row.id),
    email: row.email,
    displayName: row.display_name,
    locale: row.locale as User["locale"],
    createdAt: row.created_at.toISOString(),
  };
}

export function createUserRepository(config: AppConfig): UserRepository {
  const pool = new pg.Pool({ connectionString: config.DATABASE_URL });

  return {
    async findById(id) {
      const { rows } = await pool.query<UserRow>(
        "select id, email, display_name, locale, created_at from users where id = $1",
        [id],
      );
      const row = rows[0];
      return row ? toUser(row) : null;
    },

    async insert(user) {
      const { rows } = await pool.query<UserRow>(
        `insert into users (id, email, display_name, locale)
         values ($1, $2, $3, $4)
         returning id, email, display_name, locale, created_at`,
        [user.id, user.email, user.displayName, user.locale],
      );
      return toUser(rows[0]!);
    },
  };
}
```

```ts
// ══ packages/db/src/testing/index.ts — an in-memory repo, shared by both apps' tests ══
import type { User, UserId } from "@acme/types";
import type { UserRepository } from "../index.js";

export function createInMemoryUserRepository(seed: User[] = []): UserRepository {
  const store = new Map<string, User>(seed.map((u) => [u.id, u]));
  return {
    async findById(id: UserId) { return store.get(id) ?? null; },
    async insert(user) {
      const full: User = { ...user, createdAt: new Date().toISOString() };
      store.set(full.id, full);
      return full;
    },
  };
}
// Exposed via the "./testing" subpath export so it never leaks into the
// production import surface — `import "@acme/db"` cannot reach it.
```

```jsonc
// ══ apps/api/package.json ═══════════════════════════════════════════════════
{
  "name": "@acme/api",
  "version": "0.0.0",
  "private": true,
  "type": "module",
  "main": "./dist/index.js",
  "dependencies": {
    "@acme/types": "workspace:*",
    "@acme/config": "workspace:*",
    "@acme/db": "workspace:*",
    "express": "^4.19.0",
    "bullmq": "^5.13.0"
  },
  "devDependencies": { "@types/express": "^4.17.0" },
  "scripts": { "build": "tsc -b", "start": "node --enable-source-maps dist/index.js" }
}
```

```jsonc
// ══ apps/api/tsconfig.json ══════════════════════════════════════════════════
{
  "extends": "../../tsconfig.node.json",
  "compilerOptions": {
    "rootDir": "src",
    "outDir": "dist",
    "tsBuildInfoFile": "dist/.tsbuildinfo"
  },
  "include": ["src/**/*"],
  "exclude": ["**/*.test.ts"],
  "references": [
    { "path": "../../packages/types" },
    { "path": "../../packages/config" },
    { "path": "../../packages/db" }
    // NOTE: no reference to apps/worker. Apps must never depend on each other.
  ]
}
```

```ts
// ══ apps/api/src/queue.ts ═══════════════════════════════════════════════════
import { Queue } from "bullmq";
import type { AppConfig } from "@acme/config";
import type { JobName, JobPayloads } from "@acme/types";

/**
 * A thin typed façade over BullMQ. The generic ties the job NAME to its PAYLOAD
 * using the registry in @acme/types — the exact shape the worker will read.
 */
export interface TypedQueue {
  enqueue<N extends JobName>(name: N, payload: JobPayloads[N]): Promise<void>;
  close(): Promise<void>;
}

export function createQueue(config: AppConfig): TypedQueue {
  const queue = new Queue("acme", { connection: { url: config.REDIS_URL } });
  return {
    async enqueue(name, payload) {
      await queue.add(name, payload, { attempts: 3, backoff: { type: "exponential", delay: 1000 } });
    },
    close: () => queue.close(),
  };
}
```

```ts
// ══ apps/api/src/index.ts ═══════════════════════════════════════════════════
import express, { type Request, type Response } from "express";
import { loadConfigOrExit } from "@acme/config";
import { createUserRepository } from "@acme/db";
import type { CreateUserRequest, UserResponse } from "@acme/types";
import { asUserId } from "@acme/types";
import { createQueue } from "./queue.js";

const config = loadConfigOrExit(process.env);
const users = createUserRepository(config);
const queue = createQueue(config);

const app = express();
app.use(express.json());

app.post("/users", async (req: Request, res: Response) => {
  const body = req.body as CreateUserRequest;   // validate for real with zod in prod

  const user = await users.insert({
    id: asUserId(crypto.randomUUID()),
    email: body.email,
    displayName: body.displayName,
    locale: body.locale ?? "en",
  });

  // The compiler checks this payload against the SAME interface the worker reads.
  await queue.enqueue("welcome-email", {
    userId: user.id,
    email: user.email,
    locale: user.locale,
  });

  // ❌ await queue.enqueue("welcome-email", { orderId: "o_1", attempt: 1 });
  //    error TS2345: ... not assignable to parameter of type 'WelcomeEmailJob'.

  const response: UserResponse = {
    id: user.id,
    email: user.email,
    displayName: user.displayName,
    // ❌ adding `locale: user.locale` here errors — UserResponse deliberately
    //    omits it, so internal fields can't leak into the HTTP surface.
  };
  res.status(201).json({ data: response });
});

app.listen(config.PORT, () => console.log(`[api] listening on ${config.PORT}`));
```

```jsonc
// ══ apps/worker/tsconfig.json ═══════════════════════════════════════════════
{
  "extends": "../../tsconfig.node.json",
  "compilerOptions": {
    "rootDir": "src",
    "outDir": "dist",
    "tsBuildInfoFile": "dist/.tsbuildinfo"
  },
  "include": ["src/**/*"],
  "exclude": ["**/*.test.ts"],
  "references": [
    { "path": "../../packages/types" },
    { "path": "../../packages/config" },
    { "path": "../../packages/db" }
  ]
}
```

```ts
// ══ apps/worker/src/index.ts ════════════════════════════════════════════════
import { Worker } from "bullmq";
import { loadConfigOrExit } from "@acme/config";
import { createUserRepository } from "@acme/db";
import type { JobName, JobPayloads } from "@acme/types";

const config = loadConfigOrExit(process.env);
const users = createUserRepository(config);

/**
 * A handler for EVERY job name — enforced by the mapped type.
 * Add "invoice-generation" to JobPayloads in @acme/types and this object
 * stops compiling until you write the handler. Exhaustiveness across packages.
 */
type Handlers = { [N in JobName]: (payload: JobPayloads[N]) => Promise<void> };

const handlers: Handlers = {
  "welcome-email": async ({ userId, email, locale }) => {
    //               ^^^^ inferred as WelcomeEmailJob — no annotation needed
    const user = await users.findById(userId);
    if (!user) return;                 // user deleted between enqueue and process
    console.log(`[worker] welcome ${email} (${locale})`);
  },

  "order-settlement": async ({ orderId, attempt }) => {
    console.log(`[worker] settling ${orderId}, attempt ${attempt}`);
  },
};

new Worker(
  "acme",
  async (job) => {
    const name = job.name as JobName;
    const handler = handlers[name] as (p: JobPayloads[JobName]) => Promise<void>;
    if (!handler) throw new Error(`unknown job: ${job.name}`);
    await handler(job.data);
  },
  { connection: { url: config.REDIS_URL }, concurrency: 5 },
);

console.log("[worker] started");
```

```ts
// ══ apps/api/src/users.test.ts — tests reuse the shared in-memory repo ══════
import { describe, expect, it } from "vitest";
import { createInMemoryUserRepository } from "@acme/db/testing";  // ← subpath export
import { asUserId } from "@acme/types";

describe("user repository contract", () => {
  it("round-trips a user", async () => {
    const repo = createInMemoryUserRepository();
    const created = await repo.insert({
      id: asUserId("usr_1"),
      email: "ada@example.com",
      displayName: "Ada",
      locale: "en",
    });
    expect(await repo.findById(created.id)).toEqual(created);
  });
});
// Because the fake and the real implementation share the `UserRepository`
// interface from the same package, a signature change breaks BOTH at compile
// time. In a multi-repo world the fake silently drifts from the real thing.
```

```dockerfile
# ══ apps/api/Dockerfile — deploying ONE app out of a monorepo ═══════════════
# The hard part: the build needs the whole workspace, the runtime needs only
# apps/api plus its transitive workspace deps. Two stages solve it.

FROM node:20-slim AS build
WORKDIR /repo
RUN corepack enable

# 1. Copy ONLY manifests first — this layer caches until a package.json changes.
COPY package.json pnpm-lock.yaml pnpm-workspace.yaml ./
COPY packages/types/package.json   packages/types/
COPY packages/config/package.json  packages/config/
COPY packages/db/package.json      packages/db/
COPY apps/api/package.json         apps/api/
RUN pnpm install --frozen-lockfile

# 2. Copy sources + tsconfigs and build the whole reference graph for this app.
COPY tsconfig*.json ./
COPY packages ./packages
COPY apps/api ./apps/api
RUN pnpm --filter @acme/api... exec tsc -b
#          ^^^^^^^^^^^^^^^^^^^ the trailing "..." means "and its dependencies"

# 3. `pnpm deploy` produces a SELF-CONTAINED folder: apps/api's dist/ plus a
#    node_modules with the workspace deps COPIED IN (not symlinked).
RUN pnpm --filter @acme/api --prod deploy /out

FROM node:20-slim AS runtime
WORKDIR /app
ENV NODE_ENV=production
COPY --from=build /out ./
# ⚠️ Copy the WHOLE dist/, not dist/*.js — otherwise .js.map and .d.ts.map are
#    dropped and --enable-source-maps silently stops working (see doc 75).
USER node
CMD ["node", "--enable-source-maps", "dist/index.js"]
```

```yaml
# ══ .github/workflows/ci.yml ════════════════════════════════════════════════
name: ci
on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }          # full history, for changed-package detection

      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: pnpm }

      - run: pnpm install --frozen-lockfile

      # Restore incremental state so untouched packages are skipped entirely.
      - uses: actions/cache@v4
        with:
          path: |
            **/dist
            **/*.tsbuildinfo
          key: tsb-${{ runner.os }}-${{ github.sha }}
          restore-keys: tsb-${{ runner.os }}-

      - name: Build all projects in dependency order
        run: pnpm tsc -b --verbose        # --verbose shows which were skipped

      - name: Guard against phantom dependencies
        run: pnpm -r exec node -e "1"     # placeholder; use depcheck/manypkg here

      - run: pnpm lint
      - run: pnpm test

      # Only run the expensive stuff for packages affected by this PR:
      #   pnpm --filter "...[origin/main]" test
      # "...[ref]" = packages changed since ref, PLUS everything depending on them.
      - name: Affected tests only (PRs)
        if: github.event_name == 'pull_request'
        run: pnpm --filter "...[origin/main]" test
```

---

## Going deeper

### When a monorepo is worth it — honestly

A monorepo has real, permanent costs. Weigh them before you restructure anything.

**Worth it when:**

```
✅ Two or more services share a data contract that changes regularly
   (queue payloads, HTTP request/response shapes, DB row types, event schemas).
   This is the single strongest signal. If your services never talk, skip it.

✅ The same team owns all the pieces and ships them together.
   Atomic cross-service commits are a superpower only if one PR can span them.

✅ You want changes to fail at compile time in every consumer.
   Nothing else gives you this.

✅ Tooling setup is expensive and duplicated — eslint, tsconfig, CI, Docker,
   test setup, and you're copy-pasting them between repos today.

✅ You have a frontend and a backend that share types (tRPC, zod schemas, DTOs).
   Probably the most common good reason in practice.
```

**Not worth it when:**

```
❌ You have one service. A monorepo with one app is pure overhead: an extra
   tsconfig layer, an extra build mode, and a class of errors (TS6059, TS6307,
   stale .d.ts) you'd otherwise never meet.

❌ Different teams own different services on different release cadences.
   You'll end up with a shared repo that nobody feels ownership of, and CI
   that blocks team A on team B's failing tests.

❌ The "shared" code is genuinely stable and rarely changes.
   Publish it to a private registry and version it. Semver exists for this.

❌ You'd need to share code across languages/runtimes anyway. A TS monorepo
   does nothing for your Go service. Generate types from a schema (OpenAPI,
   protobuf, JSON Schema) instead — that's the cross-language answer.

❌ You cannot invest in build tooling. At ~10 packages, plain `tsc -b` and pnpm
   filters stop being enough and you need Turborepo/Nx for caching and task
   graphs. If nobody will own that, the repo gets slow and people start
   skipping CI.
```

**The costs, stated plainly:**

- Every developer builds every package on clone. That's a slower first day.
- CI must be made "affected-aware" or it grows linearly with repo size.
- Deploys get harder — you need per-app Dockerfiles and `pnpm deploy`-style pruning.
- Git history, blame, and `git bisect` get noisier.
- Editor performance degrades on very large graphs; you may need `disableSourceOfProjectReferenceRedirect` or a partial workspace.
- A bad `packages/*` boundary is much harder to undo than a bad file boundary — extracting a package is easy, un-extracting it after 30 imports is not.

**Practical middle ground:** start with a *single* shared package (`packages/types`) and two apps. Add packages only when you can name the specific duplication they remove. Most repos never need more than four or five.

### `tsc -b --watch` vs per-package watchers

```bash
# ✅ ONE watcher for the whole graph
tsc -b --watch --preserveWatchOutput
# - a change in packages/types rebuilds types, then rebuilds only its dependents
# - --preserveWatchOutput stops it clearing your terminal on each pass
# - pair with `node --watch apps/api/dist/index.js` to restart on output change

# ❌ N independent watchers (pnpm -r --parallel dev)
# - each package watches only itself
# - nothing coordinates ordering, so apps/api can recompile against a half-written
#   packages/types/dist and produce phantom errors that vanish on the next save
```

If you want the *fast* dev loop instead, run the app through `tsx`, which transpiles on the fly and follows the workspace symlinks into each package's source — no build step at all — while `tsc -b --watch` runs separately purely as a typechecker. That is the best of both:

```jsonc
{
  "scripts": {
    "dev": "concurrently \"tsc -b --watch --preserveWatchOutput\" \"tsx watch apps/api/src/index.ts\""
  }
}
```

(For that to work, `@acme/types` must resolve to something tsx can load — either a built `dist/`, or the `main: "./src/index.ts"` + `publishConfig` swap from Concept 6.)

### When plain `tsc -b` isn't enough: Turborepo and Nx

`tsc -b` only understands *TypeScript* build order. It knows nothing about your lint task, your test task, your codegen step, or remote caching. Past roughly ten packages, that gap starts to hurt.

```jsonc
// turbo.json — a task graph on top of the package graph
{
  "$schema": "https://turbo.build/schema.json",
  "tasks": {
    "build": {
      "dependsOn": ["^build"],           // "^" = run this task in DEPENDENCIES first
      "outputs": ["dist/**"],            // what to cache
      "inputs": ["src/**", "tsconfig.json", "package.json"]
    },
    "test": { "dependsOn": ["build"], "outputs": ["coverage/**"] },
    "lint": { "dependsOn": [] },
    "dev": { "cache": false, "persistent": true }
  }
}
```

```bash
turbo run build              # topological, parallel, cached
turbo run test --filter=...[origin/main]   # only affected packages
turbo run build --dry=json   # inspect the task graph
```

The headline feature is **remote caching**: CI builds a package once; every developer and every subsequent CI run downloads the artifact instead of rebuilding. On a large repo this turns a 6-minute CI into a 40-second one.

Nx does the same and more (generators, module boundary lint rules, a project graph visualiser) at the cost of more configuration. Rough guidance: **plain `tsc -b` + pnpm filters up to ~10 packages; Turborepo from there; Nx when you also want enforced boundaries and code generation.** Both sit *on top of* project references — they don't replace them.

### `disableSourceOfProjectReferenceRedirect` and editor performance

By default, when the language server sees `import { User } from "@acme/types"` and `packages/types` is a referenced composite project, it **redirects** to `packages/types/src/index.ts` rather than the emitted `.d.ts`. That's what makes go-to-definition work even before you've built.

On a large repo it also means the editor loads every package's full source, which can be slow.

```jsonc
// packages/types/tsconfig.json
{ "compilerOptions": { "disableSourceOfProjectReferenceRedirect": true } }
// The editor now reads dist/*.d.ts instead of src/*.ts.
// ✅ Much faster on big graphs.
// ❌ You must build before edits in the dependency show up downstream.
// ❌ declarationMap becomes essential — without it you lose source navigation entirely.
```

Related knobs for very large repos:

```jsonc
{
  "compilerOptions": {
    "disableReferencedProjectLoad": true,          // don't eagerly load referenced projects
    "disableSolutionSearching": true,              // exclude this project from "find all refs"
    "assumeChangesOnlyAffectDirectDependencies": true  // faster, slightly less sound rebuilds
  }
}
```

### Type-only packages and `isolatedDeclarations`

If `packages/types` contains *only* types, it emits `.d.ts` files and essentially empty `.js` files. Two useful refinements:

```jsonc
// TS 5.5+: guarantee every export has an explicit annotation, so .d.ts can be
// generated from a single file without full type inference. This unlocks
// parallel/fast declaration emit in tools like esbuild and swc.
{ "compilerOptions": { "isolatedDeclarations": true } }
```

```ts
// ❌ with isolatedDeclarations, this errors:
export function makeUser(email: string) {          // return type inferred
  return { id: crypto.randomUUID(), email };
}
// error TS9007: Function must have an explicit return type annotation with --isolatedDeclarations.

// ✅
export function makeUser(email: string): { id: string; email: string } {
  return { id: crypto.randomUUID(), email };
}
```

Annoying at first; genuinely worth it in a package that everything else depends on, because it makes the public surface explicit and reviewable rather than inferred.

### `exports` field ordering — a real bug source

Condition order in `exports` is **significant**; Node and TypeScript take the first match.

```jsonc
// ❌ "types" AFTER "import" — TS may resolve the .js and report "no declarations"
{ "exports": { ".": { "import": "./dist/index.js", "types": "./dist/index.d.ts" } } }

// ✅ "types" always first, "default" always last
{
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.js",
      "require": "./dist/index.cjs",
      "default": "./dist/index.js"
    }
  }
}
```

And if a consumer uses `moduleResolution: "node"` (the legacy TS ≤4 default), `exports` is ignored entirely — it falls back to `main`/`types`. Keep both fields populated, and set `moduleResolution` to `NodeNext` or `Bundler` repo-wide so everyone reads `exports`.

Use `@arethetypeswrong/cli` to check a package's resolution surface:

```bash
pnpm dlx @arethetypeswrong/cli --pack packages/api-client
# Reports: masquerading CJS/ESM, missing types conditions, resolution failures
# under every combination of moduleResolution and import style.
```

### Keeping `dependencies` and `references` in sync automatically

The manual duplication in Concept 3 is the most common source of monorepo rot. Automate it.

```bash
# Option 1 — generate references from package.json dependencies
pnpm dlx @monorepo-utils/workspaces-to-typescript-project-references --check
# CI-friendly: exits non-zero if any tsconfig's references don't match its deps.

# Option 2 — validate the workspace itself
pnpm dlx @manypkg/cli check
# Catches: mismatched dependency versions across packages, missing workspace:
# protocols, packages not covered by the workspace globs.

# Option 3 — visualise / enforce the whole import graph
pnpm dlx dependency-cruiser --validate .dependency-cruiser.js src
```

### Migrating an existing repo — the order that works

```
1. Create the workspace root: package.json with `workspaces`, `private: true`.
2. Move the existing service to apps/<name>/ UNCHANGED. Verify it still builds
   and runs. Commit. (Do not touch types yet.)
3. Add tsconfig.base.json; make apps/<name>/tsconfig.json extend it. Commit.
4. Create packages/types with composite + declarationMap. Move ONE interface
   into it — the one that hurts most. Add the dependency and the reference.
   Switch the root build to `tsc -b`. Commit.
5. Repeat step 4 one interface at a time. Each move is a small, revertible PR.
6. Only once that's stable: add the second app, then extract config/db.
```

The mistake is doing all of it in one branch. You'll hit TS6059, phantom deps, and stale `dist/` simultaneously and won't know which change caused what.

---

## Common mistakes

### Mistake 1 — Importing across packages by relative path

```ts
// ❌ apps/api/src/routes/users.ts
import type { User } from "../../../../packages/types/src/user.js";
//                        ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
// Reaches into another project's SOURCE. Errors you'll get:
//
//   error TS6059: File '/repo/packages/types/src/user.ts' is not under 'rootDir'
//                 '/repo/apps/api/src'.
//   error TS6307: File ... is not listed within the file list of project
//                 '/repo/apps/api/tsconfig.json'.
//
// And if you "fix" it by widening rootDir or adding packages/** to `include`:
// apps/api now COMPILES packages/types itself, emitting a second copy into
// apps/api/dist/packages/types/. Two builds of the same file, two different
// nominal types, and dist/ layout that no longer matches your entry point.
```

```ts
// ✅ Import the package name. That's what workspaces are for.
import type { User } from "@acme/types";

// Requires all three, every time:
//   apps/api/package.json    "dependencies": { "@acme/types": "workspace:*" }
//   apps/api/tsconfig.json   "references": [{ "path": "../../packages/types" }]
//   packages/types/tsconfig.json  "composite": true
```

### Mistake 2 — Using tsconfig `paths` instead of workspace package names

```jsonc
// ❌ tsconfig.base.json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": { "@acme/*": ["packages/*/src"] }
  }
}
// tsc is happy. Then:
//
//   $ node apps/api/dist/index.js
//   Error [ERR_MODULE_NOT_FOUND]: Cannot find package '@acme/types'
//
// ...because tsc emitted the specifier verbatim and Node has never heard of `paths`.
// You then bolt on tsconfig-paths, plus jest moduleNameMapper, plus vitest
// resolve.alias, plus an eslint resolver — four more places to keep in sync,
// forever, and they will drift.
```

```jsonc
// ✅ No paths. Real package names, declared deps, real references.
// packages/types/package.json → { "name": "@acme/types", "types": "./dist/index.d.ts" }
// apps/api/package.json       → { "dependencies": { "@acme/types": "workspace:*" } }
// apps/api/tsconfig.json      → { "references": [{ "path": "../../packages/types" }] }
//
// Now tsc, node, jest, vitest, eslint and every bundler resolve it identically,
// with ZERO additional configuration, because it's just node_modules resolution.
```

If you truly want short intra-package specifiers, use Node's native `imports` field, which *is* understood at runtime:

```jsonc
// apps/api/package.json
{ "imports": { "#/*": "./dist/*" } }   // then: import { x } from "#/services/user.js"
```

### Mistake 3 — Forgetting `composite: true` on a referenced project

```jsonc
// ❌ packages/types/tsconfig.json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": { "outDir": "dist", "rootDir": "src" }   // no composite
}
```

```
$ tsc -b
error TS6306: Referenced project '/repo/packages/types' must have setting
              "composite": true.
error TS6305: Output file '/repo/packages/types/dist/index.d.ts' has not been
              built from source file '/repo/packages/types/src/index.ts'.
```

```jsonc
// ✅ Put composite in the BASE so no package can forget it:
// tsconfig.base.json
{ "compilerOptions": { "composite": true, "declaration": true, "declarationMap": true } }

// packages/types/tsconfig.json — inherits composite automatically
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": { "rootDir": "src", "outDir": "dist", "tsBuildInfoFile": "dist/.tsbuildinfo" },
  "include": ["src/**/*"]
}
```

Note TS6305 also appears when the `.d.ts` genuinely *is* stale — run `tsc -b --force` once to distinguish a config problem from a staleness problem.

### Mistake 4 — Running `tsc -p` (or bare `tsc`) instead of `tsc -b`

```jsonc
// ❌ package.json
{ "scripts": { "build": "tsc -p apps/api/tsconfig.json" } }
```

```
`tsc -p` compiles ONE project. It reads the referenced projects' emitted .d.ts
but never BUILDS them. Consequences:

  - Clean checkout → "Cannot find module '@acme/types'" (dist/ doesn't exist yet)
  - Edited packages/types → apps/api compiles against LAST WEEK's .d.ts and
    "passes", then explodes at runtime
  - Fails with TS6305 as soon as tsc notices the mismatch, with a message most
    people mistake for a config error
```

```jsonc
// ✅ Build mode, always, from the root:
{
  "scripts": {
    "build": "tsc -b",                       // whole solution
    "build:api": "tsc -b apps/api",          // api AND its dependencies
    "typecheck": "tsc -b --dry",
    "clean": "tsc -b --clean"
  }
}
```

Corollary: `tsc --noEmit` — the classic "just typecheck" command — **does not work in build mode**. Use `tsc -b --dry` or give each package a `tsconfig.typecheck.json` with `noEmit: true` and its own `tsBuildInfoFile`.

### Mistake 5 — Sharing one `outDir` or one `.tsbuildinfo` between projects

```jsonc
// ❌ tsconfig.base.json
{ "compilerOptions": { "outDir": "dist", "composite": true } }
//                                ^^^^^^ relative to the BASE file's directory,
//                                       i.e. <repo-root>/dist — for EVERY package.
//
// packages/types → /repo/dist/index.js
// packages/db    → /repo/dist/index.js   ← overwrites it
// apps/api       → /repo/dist/index.js   ← overwrites it again
//
// And all three write /repo/dist/tsconfig.tsbuildinfo, so each build invalidates
// the others and NOTHING is ever incremental. Symptom: "tsc -b rebuilds
// everything, every time, even with no changes."
```

```jsonc
// ✅ Base holds only path-independent options; each package owns its paths.
// tsconfig.base.json
{ "compilerOptions": { "target": "ES2022", "strict": true, "composite": true, "declarationMap": true } }

// packages/types/tsconfig.json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "rootDir": "src",
    "outDir": "dist",                       // → packages/types/dist  ✅
    "tsBuildInfoFile": "dist/.tsbuildinfo"  // → packages/types/dist/.tsbuildinfo ✅
  }
}
```

### Mistake 6 — Circular package dependencies

```ts
// ❌ packages/types/src/user.ts
import type { Row } from "@acme/db";        // types → db

// ❌ packages/db/src/query.ts
import type { User } from "@acme/types";    // db → types
```

```
$ tsc -b
error TS6202: Project references may not form a circular graph. Cycle detected:
  /repo/packages/db/tsconfig.json
  /repo/packages/types/tsconfig.json
  /repo/packages/db/tsconfig.json
```

```ts
// ✅ Enforce a strict layering rule: packages/types imports NOTHING.
// packages/types/src/user.ts — no workspace imports at all
export interface User { readonly id: string; readonly email: string; }
export interface Row<T> { readonly data: T; readonly rowCount: number; }
//                ^^^^^^ the shape db needed now lives at the bottom of the graph

// packages/db/src/query.ts — imports downward only
import type { Row, User } from "@acme/types";
export async function findUser(id: string): Promise<Row<User>> { ... }
```

```jsonc
// And make it un-breakable with a lint rule, not a wiki page:
{
  "rules": {
    "import/no-restricted-paths": ["error", {
      "zones": [{
        "target": "./packages/types",
        "from": "./packages",
        "except": ["./types"],
        "message": "packages/types is layer 0 and must have no workspace dependencies."
      }]
    }]
  }
}
```

### Mistake 7 — Undeclared ("phantom") dependencies

```jsonc
// ❌ apps/worker/package.json — imports @acme/db but never declares it
{
  "name": "@acme/worker",
  "dependencies": { "@acme/types": "workspace:*" }
}
```

```ts
// apps/worker/src/index.ts
import { createUserRepository } from "@acme/db";
// Works locally! Because @acme/db is hoisted into the ROOT node_modules by
// some OTHER package that did declare it.
//
// Breaks when:
//   - apps/api stops depending on @acme/db → the hoisted copy disappears
//   - you run `pnpm --filter @acme/worker --prod deploy /out` → db not included
//   - CI runs with a stricter node-linker → MODULE_NOT_FOUND in production only
```

```jsonc
// ✅ Declare every package you import, in the package that imports it.
{
  "name": "@acme/worker",
  "dependencies": {
    "@acme/types": "workspace:*",
    "@acme/db": "workspace:*",
    "bullmq": "^5.13.0"
  }
}
```

pnpm's strict `node_modules` layout makes phantom deps fail immediately, which is one of the better reasons to prefer pnpm for monorepos. With npm/yarn hoisting, add a check (`depcheck`, `@manypkg/cli`, `eslint-plugin-import`'s `no-extraneous-dependencies`) to CI.

### Mistake 8 — Publishing a package that depends on a private one

```jsonc
// ❌ packages/api-client/package.json — published
{ "name": "@acme/api-client", "version": "2.4.1",
  "dependencies": { "@acme/types": "workspace:*" } }

// packages/types/package.json
{ "name": "@acme/types", "private": true }   // ← never published

// `pnpm publish` rewrites workspace:* → "^0.0.0" and uploads the tarball.
// Consumers run `npm i @acme/api-client` and get:
//   404 Not Found - GET https://registry.npmjs.org/@acme%2ftypes
```

```jsonc
// ✅ Option A — publish the dependency too (drop `private`, give it real semver)
{ "name": "@acme/types", "version": "1.2.0", "files": ["dist", "src"] }

// ✅ Option B — inline the types into the published package at build time
//    (api-extractor / tsup --dts-resolve / rollup-plugin-dts) so the tarball
//    has no workspace dependency at all.

// ✅ Option C — don't publish. Keep everything internal and deploy from source.
//    For a backend-only monorepo this is almost always the right answer.
```

---

## Practice exercises

### Exercise 1 — easy

Build a two-package workspace from scratch and prove that a shared type is genuinely shared.

Concretely:

1. `mkdir ts-mono-lab && cd ts-mono-lab && git init && npm init -y`. Set `"private": true` and `"workspaces": ["apps/*", "packages/*"]` in the root `package.json`. Install `typescript` and `@types/node` as root devDependencies.
2. Write `tsconfig.base.json` containing **only path-independent options**: `target: "ES2022"`, `module: "NodeNext"`, `moduleResolution: "NodeNext"`, `strict: true`, `composite: true`, `declaration: true`, `declarationMap: true`, `sourceMap: true`, `skipLibCheck: true`.
3. Create `packages/types` with a `package.json` named `@acme/types` (`private: true`, `type: "module"`, `main: "./dist/index.js"`, `types: "./dist/index.d.ts"`), a `tsconfig.json` extending the base with `rootDir`, `outDir`, and an explicit `tsBuildInfoFile`, and `src/index.ts` exporting `interface User { id: string; email: string; locale: "en" | "de" }`.
4. Create `apps/api` with `@acme/types` in its `dependencies` and a `references` entry pointing at `../../packages/types`.
5. Write a root `tsconfig.json` with `"files": []` and references to both projects.
6. Run `npm install`, then `npx tsc -b`. Confirm `packages/types/dist/index.d.ts`, `index.d.ts.map`, and `.tsbuildinfo` all exist.
7. In `apps/api/src/index.ts`, import `User` and Cmd+Click it. Confirm you land in `packages/types/src/index.ts` — **not** in `dist/index.d.ts`. Then set `declarationMap: false`, rebuild, and confirm the difference.
8. Rename `email` → `emailAddress` in `packages/types` and run `tsc -b`. Record the exact error and which file it points at.
9. Run `tsc -b --verbose` twice in a row. Explain, in a comment, why the second run says "up to date".

```
// Write your code here
```

### Exercise 2 — medium

Grow the lab into a four-package platform with a real API/worker contract, a proper dev loop, and tests that share fixtures across packages.

Concretely:

1. Add `packages/config` (depends on `types`, validates env with zod, exports `loadConfig`) and `apps/worker`. Wire `dependencies` **and** `references` for every edge, and add all four to the root solution file.
2. In `packages/types`, define a `JobPayloads` registry interface mapping at least three job names to payload types. In `apps/api`, write a `enqueue<N extends JobName>(name: N, payload: JobPayloads[N])` function. In `apps/worker`, write a `Handlers = { [N in JobName]: (p: JobPayloads[N]) => Promise<void> }` object.
3. Prove the contract three ways: (a) add a fourth job name to the registry and confirm the worker's handler map fails to compile; (b) pass the wrong payload shape to `enqueue` and record the error; (c) add a variant to a payload union and confirm a `switch` in the worker becomes non-exhaustive.
4. Add a `./testing` subpath export to one package exposing an in-memory fake, and consume it from a test in *both* apps. Confirm `import "@acme/db"` cannot reach the fake.
5. Set up a dev loop: `tsc -b --watch --preserveWatchOutput` plus `node --watch` for each app, run together with `concurrently`. Edit a type in `packages/types` and time how long until both apps report the error.
6. Add `.gitignore` entries for `**/dist` and `**/*.tsbuildinfo`. Then deliberately break incrementality by giving two tsconfigs in the same package the same `tsBuildInfoFile`, run `tsc -b --verbose` twice, and document what you observe.
7. Try the wrong way on purpose: replace one package-name import with a relative `../../packages/types/src/index.js` import. Record TS6059/TS6307 verbatim, then revert.
8. Add `pnpm --filter @acme/api... build` (or the npm equivalent) and explain what the `...` suffix changed.

```
// Write your code here
```

### Exercise 3 — hard

Take the platform to production: enforced boundaries, cached CI, a single-app Docker deploy, and one genuinely published package.

Concretely:

1. **Boundaries.** Introduce a deliberate cycle (`packages/types` imports from `packages/db`). Capture the TS6202 output. Fix it three different ways — push the shape down into `types`, extract a new `packages/core` leaf, and invert with a generic parameter — and write two sentences on which you'd ship and why. Then add an `eslint-plugin-import` `no-restricted-paths` rule (or `dependency-cruiser`) that makes the cycle impossible to reintroduce, and prove it fails.
2. **Dependency/reference sync.** Write a Node script (or wire up `@monorepo-utils/workspaces-to-typescript-project-references --check`) that fails CI when a package's `tsconfig.references` doesn't match its workspace `dependencies`. Prove it catches both a missing reference and an extra one.
3. **Docker.** Write `apps/api/Dockerfile` as a multi-stage build: copy manifests first for layer caching, `pnpm install --frozen-lockfile`, build only `@acme/api...`, then `pnpm --filter @acme/api --prod deploy /out` into a slim runtime stage. Verify the final image does **not** contain `packages/worker` or any devDependency. Then break it by copying `dist/*.js` instead of `dist` and document how source maps stop working (cross-reference doc 75).
4. **CI.** Write a GitHub Actions workflow that caches `**/dist` and `**/*.tsbuildinfo`, runs `tsc -b --verbose`, and runs tests only for packages affected since `origin/main` (`pnpm --filter "...[origin/main]" test`). Push a PR touching only `apps/worker` and confirm from the logs that `packages/types` was skipped.
5. **Publishing.** Add `packages/api-client` that depends on `@acme/types`. Try to publish it with `@acme/types` still `private: true` (use `npm pack --dry-run` and a local Verdaccio registry — do not publish to npmjs). Document the failure. Then fix it two ways: publish `@acme/types` properly with `files: ["dist", "src"]` and semver, and alternatively bundle the declarations with `tsup --dts-resolve` so the tarball has no workspace dependency. Run `@arethetypeswrong/cli --pack` on both and fix every issue it reports.
6. **Scale.** Add Turborepo. Write a `turbo.json` with `build`, `test`, and `lint` tasks, correct `dependsOn: ["^build"]` wiring, and accurate `outputs`. Measure a cold `turbo run build` vs a fully-cached one. Then explain in a comment what Turborepo caches that `tsc -b`'s `.tsbuildinfo` does not.
7. **Editor performance.** Enable `disableSourceOfProjectReferenceRedirect` on `packages/types`. Document exactly what changes about go-to-definition, and what you must now do after editing `packages/types` before downstream packages see the change.
8. Finally, write `MONOREPO.md`: the layering rules, the "add a dependency → add a reference" checklist, the build/dev/test commands, the publish checklist, and a symptom → cause → fix table covering at least eight failure modes you actually hit in steps 1–7.

```
// Write your code here
```

---

## Quick reference cheat sheet

```jsonc
// ── Workspace declaration ───────────────────────────────────────────────────
// package.json (npm / yarn / bun)
{ "private": true, "workspaces": ["apps/*", "packages/*"] }

// pnpm-workspace.yaml (pnpm — NOT package.json)
// packages:
//   - "apps/*"
//   - "packages/*"
```

```jsonc
// ── Package manifest of a workspace member ──────────────────────────────────
{
  "name": "@acme/types",              // the import specifier
  "private": true,                    // internal-only guard
  "type": "module",
  "main": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": { ".": { "types": "./dist/index.d.ts",   // "types" FIRST
                      "import": "./dist/index.js" } },
  "files": ["dist", "src"]            // publishing only; src for declarationMap
}
```

```jsonc
// ── tsconfig.base.json — path-INDEPENDENT only ──────────────────────────────
"composite": true          // required on any REFERENCED project
"declaration": true        // implied by composite — the .d.ts IS the surface
"declarationMap": true     // Cmd+Click lands on .ts, not .d.ts  ← always on
"incremental": true        // implied by composite
"module"/"moduleResolution": "NodeNext"
"strict": true
"skipLibCheck": true
```

```jsonc
// ── <package>/tsconfig.json — path-DEPENDENT only ───────────────────────────
"extends": "../../tsconfig.base.json"
"rootDir": "src"
"outDir": "dist"
"tsBuildInfoFile": "dist/.tsbuildinfo"     // set explicitly, one per config
"include": ["src/**/*"]
"references": [{ "path": "../types" }]     // mirror package.json dependencies
```

```jsonc
// ── root tsconfig.json — the solution file ──────────────────────────────────
{ "files": [], "references": [{ "path": "packages/types" }, { "path": "apps/api" }] }
```

```bash
# ── Build mode ──────────────────────────────────────────────────────────────
tsc -b                  # build the whole graph, in order, incrementally
tsc -b apps/api         # that project + its dependencies
tsc -b --watch --preserveWatchOutput
tsc -b --dry            # what WOULD build — staleness debugger
tsc -b --verbose        # why each project was / wasn't rebuilt
tsc -b --force          # ignore .tsbuildinfo
tsc -b --clean          # delete all outputs + buildinfo
# NOTE: `tsc --noEmit` does NOT work with -b. Use --dry.
```

```bash
# ── pnpm ────────────────────────────────────────────────────────────────────
pnpm install
pnpm -r build                       # recursive, topological
pnpm --filter @acme/api build       # one package
pnpm --filter @acme/api... build    # + its DEPENDENCIES
pnpm --filter ...@acme/types test   # + its DEPENDENTS
pnpm --filter "...[origin/main]" test   # affected since a git ref
pnpm add express --filter @acme/api
pnpm add -Dw typescript             # root devDependency
pnpm --filter @acme/api --prod deploy /out   # self-contained deploy folder

# ── npm workspaces ──────────────────────────────────────────────────────────
npm install
npm run build --workspaces --if-present
npm run build --workspace @acme/api
npm install express -w apps/api
```

| Symptom | Cause | Fix |
|---|---|---|
| `Cannot find module '@acme/types'` | dependency not built, or not declared | `tsc -b`; add `"@acme/types": "workspace:*"` |
| `TS6306: must have "composite": true` | referenced project isn't composite | add `composite` to `tsconfig.base.json` |
| `TS6305: output not built from source` | stale `.d.ts` or missing `composite` | `tsc -b --force`, then check `composite` |
| `TS6059: not under rootDir` | relative import across packages | import the package name instead |
| `TS6307: not listed in the file list` | same as above, or bad `include` | package-name import; widen `include` |
| `TS6202: circular graph` | package cycle | push shared shapes down into the layer-0 package |
| `ERR_MODULE_NOT_FOUND` at runtime | used tsconfig `paths` for cross-package imports | use workspace package names |
| Everything rebuilds every time | shared `outDir` / colliding `tsBuildInfoFile` | one `outDir` + one buildinfo per project |
| Cmd+Click lands in `.d.ts` | `declarationMap` off | `"declarationMap": true` |
| Works locally, 404 after publish | published package depends on a `private` one | publish the dep, or bundle its declarations |
| Works locally, breaks in Docker | phantom (undeclared) dependency | declare every imported package |
| Editor slow on a big repo | eager project loading | `disableSourceOfProjectReferenceRedirect` |

```
Layering rule of thumb:
  packages/types   → ZERO workspace dependencies (the root of the graph)
  packages/*       → may import packages below them, never apps
  apps/*           → may import any package, NEVER another app
```

---

## Connected topics

- **03 — tsconfig in depth** — `composite`, `incremental`, `declaration`, `rootDir`/`outDir`, and the `extends` resolution rules that make `tsconfig.base.json` work.
- **04 — TypeScript Node.js project setup** — the single-package baseline this doc scales up from; `module: "NodeNext"` and the `.js`-extension import rule apply identically inside every workspace member.
- **07 — Modules and imports** — `exports`, `imports`, subpath exports, and why condition order in `exports` decides whether TypeScript finds your declarations.
- **58 — Typing API responses** — the request/response shapes that belong in `packages/types` so the API and its clients can never disagree.
- **61 — Result type pattern** — a natural resident of a layer-0 shared package: every service returns the same `Result<T, E>`.
- **71 — Triple-slash directives** — the pre-project-references way of composing TypeScript projects, and why `references` replaced it.
- **75 — Debugging TypeScript in VS Code** — multi-root `outFiles` globs (`packages/*/dist/**/*.js`), and how `declarationMap` + `sourceMap` together let Step Into land on another package's `.ts` source.
- **76 — Building for production** — multi-stage Docker builds, `pnpm deploy` pruning, and shipping source maps out of a monorepo.

---

### You've finished all 77 docs 🎉

That's the whole series — from `string` and `number`, through generics, conditional types and declaration files, to strict null checks, the repository pattern, production builds, and now a full typed monorepo.

You started as someone who knew JavaScript. You now know how to make a compiler prove things about a backend: that an env var exists before you read it, that an API response can't leak an internal field, that a queue payload means the same thing on both ends of the wire, and that a change to a shared type fails loudly in every consumer the moment you make it.

The only thing left is to use it. **Build a real typed backend.** Not a tutorial — something you'd actually run: an HTTP API with validated input, a database layer with mapped row types, a background worker, real error handling with a `Result` type, and `strict: true` from the first commit. Put the shared shapes in a `packages/types`, wire up project references, and make CI run `tsc -b`.

You'll hit things this series didn't cover. That's the point — you now have the vocabulary to read the compiler's error messages, the TypeScript handbook, and other people's `.d.ts` files, and work it out.

Go build it.
