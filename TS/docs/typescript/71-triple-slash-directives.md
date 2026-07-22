# 71 — Triple slash directives

## What is this?

A **triple slash directive** is a single-line comment that begins with three slashes and contains an XML tag. It is the only comment in TypeScript that the *compiler* reads and acts on.

```ts
/// <reference types="node" />
/// <reference path="./globals.d.ts" />
/// <reference lib="es2022.array" />
/// <reference no-default-lib="true" />
```

To every JavaScript tool on earth those are just comments. To `tsc` they are **instructions that change which files get compiled and which global types exist in your program**.

There are four forms you will actually meet:

| Directive | Means |
|---|---|
| `/// <reference path="..." />` | "Also compile this *file*, and compile it before me." |
| `/// <reference types="..." />` | "Also pull in the *package* `@types/...` (or its bundled types) for its **globals**." |
| `/// <reference lib="..." />` | "Also include this built-in TypeScript library file (`lib.es2022.array.d.ts`, `lib.dom.d.ts`, …)." |
| `/// <reference no-default-lib="true" />` | "This file *is* the standard library. Do not load the default one." |

They must be at the **very top of the file** — above all statements, above imports, with nothing but comments before them. Move one below an `import` and it silently stops being a directive and becomes an ordinary comment. No error. No warning. Your types just disappear.

Triple slash directives are older than ES modules. They come from the era when TypeScript files were *global scripts* concatenated together with `--out`, and `path` references were the only way to say "this file depends on that file". Modern code almost never needs them — `import`, `tsconfig.json#include`, and `tsconfig.json#compilerOptions.types` cover the same ground with better tooling. But they are still load-bearing in exactly two places: **`.d.ts` declaration files**, and **files that need a global type that no `import` can bring in** (`vite/client`, `jest`, `@types/node` in a package that deliberately does not list it globally).

This doc explains each directive, what it does to compilation order and file inclusion, why the ordering rule exists, what `types` / `typeRoots` / `include` do instead, and the handful of situations where writing one by hand is still correct.

---

## Why does it matter?

Three concrete reasons a backend TypeScript developer runs into this.

**1. Something invisible is providing your globals, and you don't know what.**

You write `process.env.DATABASE_URL` and it type-checks. Why? Nobody imported `process`. There is no `import process from "process"` at the top of your file. The answer is that `@types/node` declares `process` as a **global**, and something — either `tsconfig.json#types`, or automatic `@types` inclusion, or a `/// <reference types="node" />` inside another `.d.ts` in your program — pulled it in. When it *stops* working (a `types: []` added to tsconfig, a monorepo package that doesn't depend on `@types/node`), you need to know which lever to pull.

**2. `.d.ts` files cannot use `import` for globals.**

The moment a `.d.ts` file contains a top-level `import` or `export`, it becomes a **module**, and everything declared in it stops being global. That is the single most common "why are my ambient types not visible anywhere?" bug. If a global declaration file needs types from another package, `/// <reference types="..." />` is the *only* way to get them without destroying its globalness. This is not legacy — this is the current, correct, unavoidable pattern.

**3. Compilation order and file inclusion.**

`/// <reference path="..." />` adds a file to the program even if `include`/`files` never mentioned it, and guarantees it is emitted *before* the referencing file when you use `outFile`. In a modern `"module": "nodenext"` backend that ordering is irrelevant — but the *inclusion* effect is not. A stray `path` reference can drag a whole directory of legacy declarations into your build, slow down `tsc`, and produce errors from files you thought were excluded.

Getting this wrong produces the worst class of TypeScript bug: **code that compiles on your machine and fails in CI**, or types that vanish when someone reorders imports.

---

## The JavaScript way vs the TypeScript way

There is no JavaScript equivalent of a triple slash directive — JavaScript has no notion of "compile these files together in this order". The honest contrast is between **the old global-script way of composing TypeScript files** (which is what triple slash directives were built for) and **the modern module + tsconfig way**.

```ts
// ══════════════════════════════════════════════════════════════════════════════
// THE OLD WAY — global scripts stitched together by reference paths
// Circa TypeScript 1.x. No import/export anywhere. Everything is global.
// ══════════════════════════════════════════════════════════════════════════════

// ── models.ts ────────────────────────────────────────────────────────────────
// No `export` keyword. This interface is GLOBAL — visible in every file
// in the program, with no import needed.
interface UserRecord {
  userId: number;
  email: string;
  role: string;
}

// ── db.ts ────────────────────────────────────────────────────────────────────
/// <reference path="./models.ts" />
// ^ This line does two things:
//   1. Adds models.ts to the compilation, even if tsconfig never listed it.
//   2. Guarantees models.ts is emitted BEFORE db.ts in the concatenated output.

function findUserById(userId: number): UserRecord | null {
  //                                   ^^^^^^^^^^ resolved because of the
  //                                   reference above — not because of an import
  return null;
}

// ── server.ts ────────────────────────────────────────────────────────────────
/// <reference path="./db.ts" />
// db.ts already references models.ts, so the transitive closure is:
//   models.ts → db.ts → server.ts
// and tsc --outFile bundle.js emits them in exactly that order.

const currentUser = findUserById(42);
```

```jsonc
// tsconfig.json for the old way — note `outFile`, which only works with
// "module": "none" | "system" | "amd". The whole program becomes ONE .js file,
// and reference order is what makes the output valid.
{
  "compilerOptions": {
    "module": "none",
    "outFile": "./dist/bundle.js"
  },
  "files": ["src/server.ts"]
  // ^ ONE entry point. Everything else is discovered through reference paths.
}
```

Problems with that model, all of which modern TypeScript solved:

```ts
// 1. No encapsulation. Every declaration in every file is global.
//    Two files both declaring `function parseToken()` = duplicate identifier error.

// 2. Ordering is manual and fragile. Forget a reference and you get a runtime
//    "X is not defined" even though it compiled fine.

// 3. No tree shaking, no lazy loading, no per-file caching, no bundler support.

// 4. Renaming a file means editing every reference path by hand — no editor
//    refactoring understood these comments for years.
```

```ts
// ══════════════════════════════════════════════════════════════════════════════
// THE MODERN WAY — ES modules + tsconfig
// ══════════════════════════════════════════════════════════════════════════════

// ── models.ts ────────────────────────────────────────────────────────────────
export interface UserRecord {
  userId: number;
  email: string;
  role: UserRole;
}

export type UserRole = "admin" | "support" | "customer";

// ── db.ts ────────────────────────────────────────────────────────────────────
import type { UserRecord } from "./models.js";
// The import IS the dependency edge. tsc resolves it, adds models.ts to the
// program automatically, and the bundler/Node runtime handles ordering.

export function findUserById(userId: number): UserRecord | null {
  return null;
}

// ── server.ts ────────────────────────────────────────────────────────────────
import { findUserById } from "./db.js";

const currentUser = findUserById(42);
```

```jsonc
// tsconfig.json for the modern way
{
  "compilerOptions": {
    "module": "nodenext",
    "moduleResolution": "nodenext",
    "outDir": "./dist",
    "strict": true,

    // This is the modern replacement for `/// <reference types="..." />`
    // when you want a global type package in EVERY file of the program:
    "types": ["node"]
  },
  "include": ["src/**/*.ts"]
  // ^ Replaces `files` + reference-path discovery entirely.
}
```

The revelation: **`import` replaced `/// <reference path="" />`, and `tsconfig#types` replaced program-wide `/// <reference types="" />`.** What neither replaced is the *file-scoped* need for globals inside `.d.ts` files, and the ability of a single file to opt into a type package that the rest of the program doesn't get. That residue is why the directives still exist and still matter.

---

## Syntax

```ts
// ══════════════════════════════════════════════════════════════════════════════
// ALL FOUR FORMS — and the exact rules about placement
// ══════════════════════════════════════════════════════════════════════════════

// ── 1. path — include another FILE by relative or absolute path ──────────────
/// <reference path="./types/globals.d.ts" />
/// <reference path="../shared/api-contracts.d.ts" />
/// <reference path="/abs/path/legacy.d.ts" />

// ── 2. types — include a type PACKAGE for its globals ────────────────────────
/// <reference types="node" />          // → node_modules/@types/node
/// <reference types="jest" />          // → node_modules/@types/jest
/// <reference types="vite/client" />   // → node_modules/vite/client.d.ts
/// <reference types="@acme/env-types" /> // scoped packages work too

// ── 3. lib — include a built-in TypeScript library file ──────────────────────
/// <reference lib="es2022" />          // → lib.es2022.d.ts
/// <reference lib="es2022.array" />    // → lib.es2022.array.d.ts (one slice)
/// <reference lib="dom" />             // → lib.dom.d.ts
/// <reference lib="esnext.asynciterable" />

// ── 4. no-default-lib — suppress lib.d.ts entirely ───────────────────────────
/// <reference no-default-lib="true" />
// Only ever correct inside TypeScript's OWN lib files, or a hand-written
// replacement standard library. You will (almost certainly) never write this.
```

```ts
// ══════════════════════════════════════════════════════════════════════════════
// PLACEMENT RULES — this is where people get burned
// ══════════════════════════════════════════════════════════════════════════════

// ✅ VALID — directives first, only comments allowed above them
// Copyright (c) 2026 Acme Corp.       ← a plain comment above is FINE
/// <reference types="node" />
/// <reference path="./env.d.ts" />

import express from "express";
const port = process.env.PORT;

// ─────────────────────────────────────────────────────────────────────────────

// ❌ INVALID — a statement precedes the directive
import express from "express";
/// <reference types="node" />   // ← now just a comment. Silently ignored.
                                 //   No error. No warning. Types missing.

// ❌ INVALID — even a `const` counts as a statement
const SERVICE_NAME = "orders-api";
/// <reference types="node" />   // ← ignored

// ⚠️ SUBTLE — a directive inside a function body or after a blank statement
// is likewise inert; only the leading comment block of the FILE is scanned.
```

```ts
// ══════════════════════════════════════════════════════════════════════════════
// SHAPE RULES
// ══════════════════════════════════════════════════════════════════════════════

/// <reference types="node" />     // ✅ self-closing, exactly three slashes
//// <reference types="node" />    // ❌ four slashes — plain comment
// / <reference types="node" />    // ❌ space breaks it
/// <reference types='node' />     // ✅ single quotes are accepted
/// <reference types="node">       // ❌ not self-closing → error TS1005 in .ts
/// <reference typos="node" />     // ❌ error TS1435: Unknown directive attribute
/// <reference />                  // ❌ error TS1451 / invalid reference
```

```ts
// ══════════════════════════════════════════════════════════════════════════════
// TWO MORE DIRECTIVES YOU WILL SEE (same syntax family, different jobs)
// ══════════════════════════════════════════════════════════════════════════════

// AMD module naming — legacy, for `--module amd` only:
/// <amd-module name="OrderService" />

// AMD dependency injection — legacy:
/// <amd-dependency path="legacy/jquery" name="$" />

// Neither has anything to do with references; both are effectively dead in
// modern Node backends. Mentioned only so you recognise them in old code.
```

```jsonc
// ══════════════════════════════════════════════════════════════════════════════
// THE MODERN ALTERNATIVES — tsconfig.json
// ══════════════════════════════════════════════════════════════════════════════
{
  "compilerOptions": {
    // Program-wide replacement for `/// <reference types="..." />`.
    // When present, ONLY these @types packages are auto-included.
    "types": ["node", "jest"],

    // Where to look for @types packages. Default walks up node_modules/@types
    // from the tsconfig's directory to the filesystem root.
    "typeRoots": ["./node_modules/@types", "./src/types"],

    // Program-wide replacement for `/// <reference lib="..." />`.
    "lib": ["ES2022"],

    // Replacement for `no-default-lib` at the program level.
    "noLib": false
  },

  // Program-wide replacement for `/// <reference path="..." />`.
  "include": ["src/**/*.ts", "src/**/*.d.ts"],
  "files": ["src/index.ts", "src/types/global.d.ts"]
}
```

---

## How it works — concept by concept

### Concept 1 — `/// <reference path="" />`: file inclusion and ordering

A `path` reference does exactly two things, and it is important to keep them separate in your head:

1. **Inclusion.** The referenced file is added to the compilation *program*, transitively, even if `include`/`files` never matched it.
2. **Ordering.** When emitting to a single file (`outFile`), the referenced file is emitted first.

```ts
// ── src/types/domain.d.ts ────────────────────────────────────────────────────
// A global declaration file. NO top-level import/export, so it stays global.
declare interface OrderRecord {
  orderId: number;
  userId: number;
  totalCents: number;
  status: string;
}

declare interface AuthToken {
  userId: number;
  role: "admin" | "support" | "customer";
  expiresAt: number;
}
```

```ts
// ── src/services/order-service.ts ────────────────────────────────────────────
/// <reference path="../types/domain.d.ts" />
// Even if tsconfig's `include` was ["src/services/**/*"] and never matched
// src/types, this line pulls domain.d.ts into the program.

export function summarise(order: OrderRecord, authToken: AuthToken): string {
  //                             ^^^^^^^^^^^  ^^^^^^^^^ both global, no import
  return `Order ${order.orderId} for user ${authToken.userId}`;
}
```

Now the ordering half, which only shows up with `outFile`:

```ts
// ── src/legacy/config.ts ─────────────────────────────────────────────────────
// A global script (no import/export), so it declares a real runtime global.
const SERVICE_CONFIG = {
  port: 3000,
  databaseUrl: "postgres://localhost/orders",
};
```

```ts
// ── src/legacy/bootstrap.ts ──────────────────────────────────────────────────
/// <reference path="./config.ts" />

// Because of the reference, tsc --outFile emits config.ts BEFORE bootstrap.ts,
// so `SERVICE_CONFIG` is initialised by the time this line runs.
console.log(`Starting on port ${SERVICE_CONFIG.port}`);
```

```js
// ── dist/bundle.js — the emitted single file ─────────────────────────────────
var SERVICE_CONFIG = {
    port: 3000,
    databaseUrl: "postgres://localhost/orders",
};
console.log("Starting on port ".concat(SERVICE_CONFIG.port));
// Remove the reference and tsc is free to emit bootstrap.ts first →
// ReferenceError: Cannot access 'SERVICE_CONFIG' before initialization
```

Key rules for `path` references:

```ts
// 1. The path is resolved RELATIVE TO THE REFERENCING FILE, not to tsconfig.
//    src/services/order-service.ts referencing "../types/domain.d.ts"
//    resolves to src/types/domain.d.ts.

// 2. References are TRANSITIVE. A → B → C puts all three in the program.

// 3. The compiler DEDUPLICATES by resolved, canonicalised path. Referencing
//    the same file from 40 files costs nothing.

// 4. Circular references are allowed and harmless for type checking.
//    They are only a problem for `outFile` ordering, where tsc picks an order.

// 5. With `noResolve: true`, path references are NOT followed — the file is
//    not added to the program, and its types are missing.
```

```jsonc
// The modern replacement — just list the directory in `include`:
{
  "include": ["src/**/*.ts", "src/**/*.d.ts"]
}
// Now every .d.ts under src is in the program, no reference comments needed.
// This is why you almost never write a path reference in a modern repo:
// `include` already did it, and it did it for every file at once.
```

---

### Concept 2 — `/// <reference types="" />`: pulling in a package's globals

This is the form you will actually write in 2026. It says: *resolve this **package** the way `@types` resolution works, and include its declarations for their **global** effects.*

```ts
// ── The problem it solves ────────────────────────────────────────────────────

// src/types/environment.d.ts
// This file needs to augment NodeJS.ProcessEnv — a namespace declared by
// @types/node. But this file must remain a GLOBAL declaration file.

/// <reference types="node" />
// ^ Without this, `NodeJS` is unknown here in any config where @types/node
//   isn't auto-included (e.g. `"types": []`, or a package that doesn't
//   depend on @types/node directly).

declare namespace NodeJS {
  interface ProcessEnv {
    DATABASE_URL: string;
    REDIS_URL: string;
    JWT_SECRET: string;
    PORT?: string;
    NODE_ENV: "development" | "test" | "production";
  }
}

// Now, in every file of the program:
//   process.env.DATABASE_URL  → string
//   process.env.PORT          → string | undefined
//   process.env.DATABSE_URL   → ❌ Property 'DATABSE_URL' does not exist
```

Why can't you just `import`? Because importing would make the file a module:

```ts
// ❌ WRONG — this file is now a MODULE. The `declare namespace NodeJS` below
//    is scoped to the module and augments nothing globally.
import type {} from "node";

declare namespace NodeJS {
  interface ProcessEnv {
    DATABASE_URL: string;
  }
}
// process.env.DATABASE_URL is still `string | undefined` everywhere else.
// Nothing errored. Nothing warned. The augmentation just did not happen.
```

```ts
// ✅ RIGHT — reference directive keeps the file a global script
/// <reference types="node" />

declare namespace NodeJS {
  interface ProcessEnv {
    DATABASE_URL: string;
  }
}
```

How `types="X"` resolves:

```ts
// /// <reference types="node" />
//
// 1. Look for node_modules/@types/node/  in each `typeRoots` entry
//    (default: walk node_modules/@types upward from the containing directory).
// 2. Read its package.json "types"/"typings" field → index.d.ts.
// 3. Include that file (and its own reference/import graph) in the program.
//
// /// <reference types="vite/client" />
//
// 1. "vite/client" has a slash, so it is treated as a package SUBPATH.
// 2. Resolve node_modules/vite/client.d.ts (via exports map or direct path).
// 3. Include it. This is how Vite ships `import.meta.env` and
//    `*.svg` module declarations.
```

A crucial asymmetry between `.ts` and `.d.ts` files:

```ts
// In a .ts FILE (your application source):
//   /// <reference types="jest" />
//   → includes @types/jest for THIS COMPILATION. Same effect as adding it
//     to tsconfig#types, but scoped by having written it here.

// In a .d.ts FILE that you PUBLISH in an npm package:
//   /// <reference types="node" />
//   → becomes a DEPENDENCY DECLARATION. Anyone who consumes your package and
//     reads your .d.ts will have @types/node pulled into THEIR program.
//     If they don't have @types/node installed, they get:
//       error TS2688: Cannot find type definition file for 'node'.
//
//   This is why publishing a `/// <reference types="node" />` obligates you to
//   list @types/node as a real `dependency` (not devDependency) in package.json.
```

TypeScript can also **auto-generate** these when it emits declarations:

```ts
// ── src/http-client.ts ───────────────────────────────────────────────────────
import type { IncomingMessage } from "node:http";

export function readBody(requestBody: IncomingMessage): Promise<string> {
  return new Promise((resolve) => {
    let data = "";
    requestBody.on("data", (chunk: Buffer) => { data += chunk.toString(); });
    requestBody.on("end", () => resolve(data));
  });
}
```

```ts
// ── dist/http-client.d.ts — EMITTED by tsc --declaration ─────────────────────
/// <reference types="node" />
//  ^^^^^^^^^^^^^^^^^^^^^^^^^ tsc added this itself, because the emitted
//  declarations mention `Buffer`, a global from @types/node.

import type { IncomingMessage } from "node:http";
export declare function readBody(requestBody: IncomingMessage): Promise<string>;
```

```jsonc
// You can suppress that auto-emission per-package:
{
  "compilerOptions": {
    "types": ["node"],
    // Do not emit `/// <reference types="node" />` into .d.ts output:
    "typesVersions": {}, // unrelated, shown for contrast
  }
}
// The real flag is:
//   "compilerOptions": { "disableReferencedProjectLoad": false }   ← not this
// Actually: `--emitDeclarationOnly` does not change it; the supported way is
//   "compilerOptions": { "types": [...] } plus
//   // @ts-ignore-free code, or the `noResolve`/bundler post-processing step.
// In practice most libraries KEEP the emitted reference — it is correct.
```

---

### Concept 3 — `/// <reference lib="" />`: pulling in built-in libraries

TypeScript ships a set of `.d.ts` files describing JavaScript itself and the DOM. They live next to the compiler in `node_modules/typescript/lib/`:

```
lib.es5.d.ts              lib.es2015.d.ts        lib.es2022.d.ts
lib.es2015.core.d.ts      lib.es2022.array.d.ts  lib.es2023.array.d.ts
lib.dom.d.ts              lib.dom.iterable.d.ts  lib.webworker.d.ts
lib.esnext.d.ts           lib.decorators.d.ts    lib.esnext.disposable.d.ts
```

Normally you select these with `compilerOptions.lib`. A `lib` reference selects one **for a single file**, adding it to the program.

```ts
// ── The problem ──────────────────────────────────────────────────────────────
// tsconfig says:
//   { "compilerOptions": { "target": "ES2020", "lib": ["ES2020"] } }
// so Array.prototype.at() (ES2022) is not typed.

const userIds = [101, 102, 103];
const lastUserId = userIds.at(-1);
// ❌ Property 'at' does not exist on type 'number[]'.
//    Do you need to change your target library?
```

```ts
// ── The file-scoped fix ──────────────────────────────────────────────────────
/// <reference lib="es2022.array" />

const userIds = [101, 102, 103];
const lastUserId = userIds.at(-1);   // ✅ number | undefined
// The whole PROGRAM now has lib.es2022.array.d.ts loaded — `lib` references,
// like `types` references, affect the program, not just this file.
```

```ts
// ── Where this is genuinely used: library .d.ts files ────────────────────────
// A package that uses AsyncIterable in its public API but wants to work for
// consumers whose `lib` is only ES2015:

/// <reference lib="esnext.asynciterable" />

export declare function streamOrderEvents(
  userId: number,
): AsyncIterableIterator<{ orderId: number; status: string }>;
// Without the lib reference, consumers on old `lib` settings get
// "Cannot find name 'AsyncIterableIterator'".
```

Important nuances:

```ts
// 1. `lib` references are RESOLVED AGAINST THE COMPILER'S lib folder, not
//    node_modules. `lib="es2022.array"` → typescript/lib/lib.es2022.array.d.ts.
//    A typo gives:  error TS6231 / "Cannot find lib definition for 'es2022.arry'".

// 2. Names are the tsconfig `lib` values, lowercased, without the
//    "lib." prefix and ".d.ts" suffix:
//        tsconfig "lib": ["ES2022.Array"]  ↔  /// <reference lib="es2022.array" />

// 3. A lib reference ADDS; it never removes. You cannot use it to drop `dom`.
//    To drop DOM types you must set tsconfig `lib` explicitly:
//        { "lib": ["ES2022"] }         // no "DOM" entry
//    which matters a lot on backends — you do NOT want `fetch`'s DOM typing,
//    `document`, or `localStorage` autocompleting in a Node service.

// 4. Because it affects the whole program, one file's lib reference silently
//    grants ES2022 arrays to every other file. That is usually not what you
//    wanted; prefer tsconfig `lib` for program-wide intent.
```

```jsonc
// ── The modern, program-level way ────────────────────────────────────────────
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["ES2022"],            // Node backend: no DOM
    "types": ["node"]
  }
}
// If `lib` is omitted, it is DERIVED from `target`. Setting `target: "ES2022"`
// implies `lib: ["ES2022", "DOM", "DOM.Iterable", "ScriptHost"]` — note DOM is
// in there by default. Backend projects should set `lib` explicitly to exclude it.
```

---

### Concept 4 — `/// <reference no-default-lib="true" />`

This one you read about and then never write.

```ts
// ── What it does ─────────────────────────────────────────────────────────────
/// <reference no-default-lib="true" />
// "Treat this file as a default library. Do NOT load lib.d.ts for this program."
//
// Semantically identical to setting `"noLib": true` in tsconfig, but triggered
// by the presence of the directive in ANY file that ends up in the program.
```

```ts
// ── Where it actually appears ────────────────────────────────────────────────
// The very first lines of typescript/lib/lib.es5.d.ts look like this:

/// <reference no-default-lib="true"/>

interface Array<T> { /* ... */ }
interface String { /* ... */ }
interface Object { /* ... */ }
// Without the directive, loading lib.es5.d.ts would recursively load the
// default lib, which is itself lib.es5.d.ts. The directive breaks that cycle.
```

```ts
// ── What happens if you write it by accident ─────────────────────────────────
/// <reference no-default-lib="true" />

const email: string = "ops@acme.dev";
// ❌ error TS2318: Cannot find global type 'String'.
// ❌ error TS2318: Cannot find global type 'Array'.
// ❌ error TS2318: Cannot find global type 'Boolean'.
// ❌ error TS2318: Cannot find global type 'Function'.
// ❌ error TS2318: Cannot find global type 'Number'.
// ❌ error TS2318: Cannot find global type 'Object'.
// ❌ error TS2318: Cannot find global type 'RegExp'.
// ❌ error TS2318: Cannot find global type 'IArguments'.
//
// You have deleted JavaScript. Every primitive type is gone, program-wide.
```

```ts
// ── The two legitimate uses ──────────────────────────────────────────────────
// 1. You are the TypeScript team, writing lib.*.d.ts.
// 2. You are targeting an exotic runtime with no JS standard library at all
//    (some embedded/AssemblyScript-adjacent setups) and are shipping a
//    complete hand-written replacement standard library.
//
// If neither applies — and it doesn't — never write it. And if you inherit a
// repo with mysterious TS2318 errors, grep for it first:
//    grep -rn "no-default-lib" src/ node_modules/@types/
```

---

### Concept 5 — Why they must be at the very top

The rule is mechanical, and understanding the mechanism makes it impossible to forget.

```ts
// ── The compiler's preprocessing phase ───────────────────────────────────────
//
// Before type checking — before even full parsing — tsc runs a lightweight
// "preprocess" pass over each file to discover its dependencies. It needs to
// answer "what other files must I load?" as cheaply as possible.
//
// That pass reads the LEADING TRIVIA of the file: the run of comments and
// whitespace before the first real token. It scans those comments for
// reference directives, and stops the moment it hits a token that is not
// a comment.
//
// So the rule is not arbitrary — the scanner literally stops looking.
```

```ts
// ✅ Everything above the first token is scanned
/*
 * Order service — handles state transitions and refunds.
 * Copyright (c) 2026 Acme Corp.
 */
// eslint-disable-next-line @typescript-eslint/no-explicit-any
/// <reference types="node" />
/// <reference path="./domain.d.ts" />

import express from "express";  // ← first real token. Scanning stopped here.
```

```ts
// ❌ The directive is below a token, so it is never seen
import express from "express";

/// <reference types="node" />
// This is now, to the compiler, a comment containing the text
// "/ <reference types=\"node\" />". It has zero effect.
//
// Note the failure mode: it is SILENT. There is no
//   "warning: reference directive ignored because it is not at the top"
// You discover it when `process` is `any` or missing.
```

```ts
// ⚠️ A subtle trap: `"use strict"` and shebangs
#!/usr/bin/env node
/// <reference types="node" />
// ✅ A shebang on line 1 is fine — it is treated as trivia.

"use strict";
/// <reference types="node" />
// ❌ A string-literal directive prologue IS a statement. Reference ignored.
```

```ts
// ⚠️ Another trap: JSDoc that looks like a directive
/**
 * /// <reference types="node" />
 */
// ❌ Inside a block comment, not a line comment. Not a directive.
// Directives must be `//`-style line comments with exactly three slashes.
```

There is one more consequence worth internalising:

```ts
// Because directives are discovered in the preprocessing pass, they affect
// the SET OF FILES IN THE PROGRAM. That happens before any file is checked.
//
// Which means:
//   - A reference in ONE file changes the global environment of EVERY file.
//   - There is no such thing as a "file-local" reference effect for
//     `types` or `lib`. The globals land everywhere.
//   - Order of files does not matter for correctness; only for `outFile` emit.
```

---

### Concept 6 — `tsconfig#types`, `typeRoots`, and automatic `@types` inclusion

This is the machinery that makes `process.env` work without you writing anything, and the machinery you reach for instead of directives.

```jsonc
// ── The default behaviour (no `types`, no `typeRoots`) ───────────────────────
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "nodenext"
  }
}
```

```ts
// With that config, tsc does AUTOMATIC @types INCLUSION:
//
// 1. Start at the directory containing tsconfig.json.
// 2. Look for ./node_modules/@types/
// 3. Include the package.json "types" entry of EVERY subdirectory found.
// 4. Walk up one directory. Repeat until the filesystem root.
//
// So if node_modules/@types contains:
//     node/  jest/  express/  lodash/  cors/  pg/
// then ALL SIX are added to your program's global scope, in every file,
// whether you use them or not.
//
// This is why `describe()` and `it()` autocomplete in your production
// source files, not just your test files. @types/jest declared them globally
// and automatic inclusion put them everywhere.
```

```jsonc
// ── `types`: an explicit allowlist ───────────────────────────────────────────
{
  "compilerOptions": {
    "types": ["node"]
  }
}
// Now ONLY @types/node is auto-included. @types/jest, @types/express, etc.
// are no longer global — though they are still used when you `import` from
// the package they describe. `types` controls GLOBAL inclusion only.
```

```ts
// The distinction that trips everyone:
//
// import express from "express";
//   → resolves @types/express through normal module resolution.
//   → WORKS regardless of `types`. `types` does not gate imports.
//
// declare const describe: (name: string, fn: () => void) => void;
//   → @types/jest declares this GLOBALLY, with no module to import from.
//   → Only appears if @types/jest is auto-included, listed in `types`,
//     or pulled in by a `/// <reference types="jest" />`.
```

```jsonc
// ── The canonical backend setup: separate tsconfig for tests ─────────────────

// tsconfig.json — production build. No test globals leak into src.
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["ES2022"],
    "module": "nodenext",
    "moduleResolution": "nodenext",
    "outDir": "./dist",
    "strict": true,
    "types": ["node"]        // ← jest globals deliberately excluded
  },
  "include": ["src/**/*.ts"],
  "exclude": ["src/**/*.test.ts", "src/**/*.spec.ts"]
}

// tsconfig.test.json — extends it and re-adds jest.
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "noEmit": true,
    "types": ["node", "jest"]   // ← NOT merged; `types` is REPLACED wholesale
  },
  "include": ["src/**/*.ts", "test/**/*.ts"]
}
```

```jsonc
// ── The alternative: keep one tsconfig, use a directive in test files ────────
// tsconfig.json has "types": ["node"] only.
```

```ts
// test/order-service.test.ts
/// <reference types="jest" />

import { OrderService } from "../src/services/order-service.js";

describe("OrderService", () => {          // ✅ jest globals available
  it("rejects transitions from terminal states", async () => {
    expect(true).toBe(true);
  });
});
// Trade-off: because reference directives are program-wide, this ALSO makes
// `describe` available in src/*.ts. It is less isolated than a second
// tsconfig, but far less config to maintain. Pick your poison knowingly.
```

```jsonc
// ── `typeRoots`: WHERE to look for those packages ────────────────────────────
{
  "compilerOptions": {
    "typeRoots": [
      "./node_modules/@types",   // the usual place
      "./src/types",             // your own hand-written global type packages
      "../../node_modules/@types" // monorepo hoisted root
    ]
  }
}
// Every immediate subdirectory of each typeRoot is treated as a type package.
// So ./src/types/acme-env/index.d.ts becomes includable as:
//     /// <reference types="acme-env" />
// or  "types": ["node", "acme-env"]
//
// ⚠️ Setting `typeRoots` REPLACES the default upward walk. In a monorepo where
// @types are hoisted to the repo root, a naive
//     "typeRoots": ["./node_modules/@types"]
// in a package silently loses every hoisted type package.
```

```ts
// ── Decision guide ───────────────────────────────────────────────────────────
//
// Want a global type package in EVERY file of a project?
//     → tsconfig "types": [...]                       (preferred)
//
// Want it in a .d.ts file that must stay global?
//     → /// <reference types="..." />                 (only option)
//
// Want it in ONE .ts file for a one-off (a script, a test)?
//     → /// <reference types="..." />                 (works, but leaks
//                                                      program-wide)
//
// Publishing a library whose .d.ts mentions Buffer/NodeJS.*?
//     → /// <reference types="node" /> in the .d.ts, AND
//       @types/node in package.json dependencies      (required)
```

---

### Concept 7 — Directives inside `.d.ts` files: the case that still matters

Everything above converges here. `.d.ts` files are where triple slash directives remain genuinely necessary, because of one hard rule:

> **A `.d.ts` file with a top-level `import` or `export` is a module. All its declarations are module-scoped. A `.d.ts` file without one is a global script, and everything it declares is global.**

```ts
// ══════════════════════════════════════════════════════════════════════════════
// CASE A — a global declaration file that needs another package's types
// ══════════════════════════════════════════════════════════════════════════════

// src/types/express.d.ts
/// <reference types="express" />
/// <reference types="node" />

// Augment Express's Request with the fields our auth middleware attaches.
declare namespace Express {
  interface Request {
    authToken?: {
      userId: number;
      role: "admin" | "support" | "customer";
      expiresAt: number;
    };
    requestId: string;
  }
}

// ⚠️ Note: for @types/express specifically, the modern idiom is
// module augmentation with `declare module "express-serve-static-core"`,
// which REQUIRES the file to be a module (so it needs an import/export).
// The global `declare namespace Express` form above works because
// @types/express declares that namespace globally. Know both; see 51.
```

```ts
// ══════════════════════════════════════════════════════════════════════════════
// CASE B — declaring globals your runtime injects
// ══════════════════════════════════════════════════════════════════════════════

// src/types/runtime-globals.d.ts
/// <reference types="node" />
// ^ needed for `NodeJS.Timeout` and `Buffer` below

declare global {
  // ❌ WRONG in this file — `declare global` requires the file to be a MODULE.
  //    In a global script it is:
  //      error TS2669: Augmentations for the global scope can only be
  //      directly nested in external modules or ambient module declarations.
}

// ✅ In a global script, just declare at the top level — it IS global:
declare const __BUILD_SHA__: string;
declare const __BUILD_TIME__: number;

declare function registerShutdownHook(
  handler: () => Promise<void>,
): NodeJS.Timeout;
//         ^^^^^^^^^^^^^^^^ from @types/node, available thanks to the reference

interface CachedResponse {
  body: Buffer;
  //    ^^^^^^ also from @types/node
  cachedAt: number;
  ttlSeconds: number;
}
```

```ts
// ══════════════════════════════════════════════════════════════════════════════
// CASE C — the module version, for contrast
// ══════════════════════════════════════════════════════════════════════════════

// src/types/express-augment.d.ts
import type { UserRole } from "../models/user.js";
// ^ This import makes the file a MODULE. Every top-level declaration is now
//   module-scoped. To reach global scope you must use `declare global`
//   or `declare module`.

declare module "express-serve-static-core" {
  interface Request {
    authToken?: {
      userId: number;
      role: UserRole;
      expiresAt: number;
    };
  }
}

declare global {
  // ✅ Allowed here, because this file is a module.
  var __REQUEST_COUNTER__: number;
}

export {};
// ^ Belt and braces: guarantees module-ness even if the import is removed.
```

```ts
// ══════════════════════════════════════════════════════════════════════════════
// THE DECISION TREE FOR .d.ts FILES
// ══════════════════════════════════════════════════════════════════════════════
//
// Does the file need types from another package?
//   │
//   ├── Does it also need to declare/augment GLOBAL things?
//   │     │
//   │     ├── YES → keep it a global script.
//   │     │         Use /// <reference types="..." /> for the dependency.
//   │     │         NEVER add a top-level import/export.
//   │     │
//   │     └── NO  → make it a module. Use `import type`.
//   │               Reference directives are unnecessary.
//   │
//   └── NO → no directive needed either way.
```

```ts
// ══════════════════════════════════════════════════════════════════════════════
// THE `import type {}` ESCAPE HATCH — a modern middle ground
// ══════════════════════════════════════════════════════════════════════════════

// You can load a package's globals from a MODULE .d.ts without importing
// any specific symbol, using an empty type-only import:
import type {} from "node";

// This is equivalent in effect to `/// <reference types="node" />` for the
// purpose of loading global declarations, but only inside a module file —
// and it is properly tracked by editors, bundlers, and `verbatimModuleSyntax`.
//
// Use it when the file is a module anyway. Use the reference directive when
// the file must stay a global script.
```

---

### Concept 8 — Compilation order, `files`, `include`, and the program graph

The final piece: how directives interact with the rest of the file-inclusion machinery.

```ts
// ══════════════════════════════════════════════════════════════════════════════
// HOW tsc BUILDS THE PROGRAM — in order
// ══════════════════════════════════════════════════════════════════════════════
//
// 1. ROOT FILES
//    Union of:
//      - tsconfig "files"    (explicit list; no globs; every entry must exist)
//      - tsconfig "include"  (globs) minus "exclude" (globs)
//      - files passed on the CLI
//    Note: "exclude" filters "include" ONLY. It never removes a file listed
//    in "files", and it never removes a file reached by import/reference.
//
// 2. AUTOMATIC @types INCLUSION
//    Every package under each typeRoot (or the default upward walk),
//    filtered by "types" if present.
//
// 3. DEFAULT LIB
//    lib.*.d.ts chosen by "lib" (or derived from "target"),
//    unless "noLib" or a no-default-lib directive is present.
//
// 4. CLOSURE
//    Repeatedly, for every file now in the program:
//      - follow its `import` / `export ... from` specifiers
//      - follow its `/// <reference path="" />`   → adds a FILE
//      - follow its `/// <reference types="" />`  → adds a PACKAGE
//      - follow its `/// <reference lib="" />`    → adds a LIB FILE
//      - follow `import("...")` type positions
//    until nothing new is added.
//
// The program is the fixed point of that closure. `exclude` cannot shrink it.
```

```jsonc
// ── The classic surprise ─────────────────────────────────────────────────────
{
  "include": ["src/**/*.ts"],
  "exclude": ["src/legacy/**"]
}
```

```ts
// src/services/report-service.ts
/// <reference path="../legacy/old-types.d.ts" />
// ⚠️ src/legacy IS excluded from `include` globs — but this reference pulls
//    old-types.d.ts into the program anyway. `exclude` did nothing.
//    Same thing happens with a plain `import "../legacy/helper.js"`.
//
// Excluding a directory does not make it unreachable. It only makes it
// not-a-root. Reachability wins.
```

```ts
// ══════════════════════════════════════════════════════════════════════════════
// EMIT ORDER — where `path` references still have teeth
// ══════════════════════════════════════════════════════════════════════════════
//
// With "outFile" (only valid for "module": "none" | "amd" | "system"):
//   tsc emits files in a topological order derived from:
//     1. the order of tsconfig "files"
//     2. the transitive `/// <reference path="" />` graph — dependencies first
//
// With "outDir" (every modern project):
//   Each file is emitted independently. Reference order is IRRELEVANT to emit.
//   Runtime ordering is the module system's problem, driven by `import`.
//
// So: if your tsconfig has "outDir" and "module": "nodenext" — which it does —
// then the ORDERING half of `path` references is dead weight. Only the
// INCLUSION half still does anything, and `include` does that better.
```

```jsonc
// ══════════════════════════════════════════════════════════════════════════════
// PROJECT REFERENCES — the modern, unrelated thing with a confusingly
// similar name
// ══════════════════════════════════════════════════════════════════════════════
// tsconfig.json — NOT a triple slash directive. Completely different feature.
{
  "compilerOptions": {
    "composite": true,
    "declaration": true
  },
  "references": [
    { "path": "../shared" },
    { "path": "../db" }
  ]
}
// "references" in tsconfig = PROJECT references: build orchestration between
// separate tsconfigs, with incremental builds and .tsbuildinfo.
//
// /// <reference path="" /> = FILE references inside one program.
//
// They share a word and nothing else. If someone says "add a reference",
// clarify which one they mean.
```

```ts
// ══════════════════════════════════════════════════════════════════════════════
// `noResolve` — turning the closure off
// ══════════════════════════════════════════════════════════════════════════════
// { "compilerOptions": { "noResolve": true } }
//
// tsc does NOT follow imports or reference directives. Only root files are
// compiled. Any type not found in a root file becomes an error.
//
// Used almost exclusively by tooling that manages the file list itself
// (some bundler plugins, some test harnesses). If you turn it on by accident,
// every reference directive in your codebase stops working at once.
```

---

## Example 1 — basic

```ts
// ══════════════════════════════════════════════════════════════════════════════
// A tiny Node service that needs THREE different kinds of global type,
// each obtained a different way. Watch which lever solves which problem.
// ══════════════════════════════════════════════════════════════════════════════

// ── File 1: src/types/env.d.ts ───────────────────────────────────────────────
// A GLOBAL declaration file. It must not contain a top-level import/export.
// It needs the `NodeJS` namespace, which lives in @types/node.

/// <reference types="node" />

declare namespace NodeJS {
  interface ProcessEnv {
    // Required — if missing at runtime, the app should refuse to boot.
    DATABASE_URL: string;
    JWT_SECRET: string;

    // Optional — note the `?`, which makes the type `string | undefined`.
    PORT?: string;
    LOG_LEVEL?: "debug" | "info" | "warn" | "error";

    // Literal union so typos in NODE_ENV comparisons are compile errors.
    NODE_ENV: "development" | "test" | "production";
  }
}
```

```ts
// ── File 2: src/types/build-globals.d.ts ─────────────────────────────────────
// Globals injected by the build step (esbuild `define`, webpack DefinePlugin).
// No external types needed → no reference directive needed.

declare const __BUILD_SHA__: string;
declare const __BUILD_TIMESTAMP__: number;
declare const __SERVICE_NAME__: string;
```

```ts
// ── File 3: src/config.ts ────────────────────────────────────────────────────
// Ordinary application code. No directives at all — `include` already put
// both .d.ts files in the program, and their declarations are global.

export interface ServiceConfig {
  readonly serviceName: string;
  readonly buildSha: string;
  readonly port: number;
  readonly databaseUrl: string;
  readonly jwtSecret: string;
  readonly logLevel: "debug" | "info" | "warn" | "error";
  readonly isProduction: boolean;
}

export function loadConfig(): ServiceConfig {
  // process.env.DATABASE_URL is `string` — not `string | undefined` —
  // because env.d.ts narrowed it. That came from the .d.ts, which only
  // worked because of the /// <reference types="node" /> at its top.
  const databaseUrl = process.env.DATABASE_URL;
  const jwtSecret = process.env.JWT_SECRET;

  if (!databaseUrl) {
    throw new Error("DATABASE_URL is required but was empty");
  }
  if (!jwtSecret || jwtSecret.length < 32) {
    throw new Error("JWT_SECRET must be at least 32 characters");
  }

  return {
    serviceName: __SERVICE_NAME__,   // from build-globals.d.ts
    buildSha: __BUILD_SHA__,         // from build-globals.d.ts
    port: Number(process.env.PORT ?? 3000),
    databaseUrl,
    jwtSecret,
    logLevel: process.env.LOG_LEVEL ?? "info",
    isProduction: process.env.NODE_ENV === "production",
    //                                  ^^^^^^^^^^^^^^ typo here would be
    //                                  a compile error, thanks to the union
  };
}
```

```ts
// ── File 4: src/legacy/report-formatter.ts ───────────────────────────────────
// One file needs an ES2023 array method, but tsconfig targets ES2020.
// A file-scoped lib reference solves it without changing the whole project.

/// <reference lib="es2023.array" />

interface OrderSummary {
  orderId: number;
  userId: number;
  totalCents: number;
  createdAt: Date;
}

export function mostRecentOrder(orders: OrderSummary[]): OrderSummary | undefined {
  // findLast() is ES2023. Without the lib reference:
  //   ❌ Property 'findLast' does not exist on type 'OrderSummary[]'.
  return orders.findLast((order) => order.totalCents > 0);
}

export function orderIdsNewestFirst(orders: OrderSummary[]): number[] {
  // toReversed() is also ES2023 — and non-mutating, unlike reverse().
  return orders.toReversed().map((order) => order.orderId);
}
```

```jsonc
// ── File 5: tsconfig.json ────────────────────────────────────────────────────
{
  "compilerOptions": {
    "target": "ES2020",
    "lib": ["ES2020"],              // deliberately no "DOM" — this is a backend
    "module": "nodenext",
    "moduleResolution": "nodenext",
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "declaration": true,

    // Program-wide global type packages. Note: this is the modern equivalent
    // of writing `/// <reference types="node" />` in every source file.
    // The directive in env.d.ts is STILL needed, because that file must be
    // self-contained if it is ever consumed outside this tsconfig.
    "types": ["node"],

    "typeRoots": ["./node_modules/@types", "./src/types-packages"]
  },
  "include": ["src/**/*.ts", "src/**/*.d.ts"],
  "exclude": ["node_modules", "dist"]
}
```

```ts
// ══════════════════════════════════════════════════════════════════════════════
// WHAT WOULD BREAK, AND WHY — walk through each removal
// ══════════════════════════════════════════════════════════════════════════════
//
// Remove `/// <reference types="node" />` from env.d.ts:
//   → Still compiles HERE, because tsconfig "types": ["node"] already loaded
//     @types/node into the program. But the file is no longer self-contained.
//     Copy it into a package with "types": [] and it breaks with
//     "error TS2503: Cannot find namespace 'NodeJS'".
//
// Remove "types": ["node"] from tsconfig:
//   → Still compiles, because env.d.ts's directive pulls @types/node in.
//     This is the redundancy that makes people delete the wrong one.
//
// Remove BOTH:
//   → error TS2503: Cannot find namespace 'NodeJS'.
//   → error TS2580: Cannot find name 'process'.
//
// Add an `export {}` to the bottom of env.d.ts:
//   → File becomes a module. `declare namespace NodeJS` no longer augments
//     the global NodeJS namespace. process.env.DATABASE_URL silently reverts
//     to `string | undefined`. NO ERROR ANYWHERE. Worst failure mode in the doc.
//
// Remove `/// <reference lib="es2023.array" />`:
//   → error TS2550: Property 'findLast' does not exist on type
//     'OrderSummary[]'. Do you need to change your target library?
//
// Move the lib reference below the `interface OrderSummary` declaration:
//   → Same error. The directive became a plain comment.
```

---

## Example 2 — real world backend use case

```ts
// ══════════════════════════════════════════════════════════════════════════════
// A published npm package: @acme/order-client
//
// Constraints that force real directive decisions:
//   - It exposes Node types (Buffer, Readable) in its PUBLIC API.
//   - It ships hand-written .d.ts for a legacy untyped dependency.
//   - It declares one global for consumers who use the CDN build.
//   - Its own tests need jest globals, but its src must NOT see them.
//   - Consumers may be on old `lib` settings, so it must self-declare
//     the lib slices its API depends on.
// ══════════════════════════════════════════════════════════════════════════════


// ═════ 1. package.json — the contract behind the directives ══════════════════
```

```jsonc
{
  "name": "@acme/order-client",
  "version": "3.1.0",
  "type": "module",
  "main": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": { "types": "./dist/index.d.ts", "import": "./dist/index.js" },
    "./globals": { "types": "./dist/globals.d.ts" }
  },
  "dependencies": {
    // ⚠️ NOT devDependencies. Our published .d.ts contains
    //    /// <reference types="node" />, so every consumer needs this
    //    resolvable or they get TS2688.
    "@types/node": "^22.0.0",
    "undici": "^7.0.0"
  },
  "devDependencies": {
    "@types/jest": "^29.5.0",
    "jest": "^29.7.0",
    "typescript": "^5.8.0"
  },
  "files": ["dist"]
}
```

```ts
// ═════ 2. src/types/legacy-retry.d.ts ════════════════════════════════════════
// Hand-written declarations for `flaky-retry`, a JS-only dependency with no
// types on npm. This file is a MODULE-declaring global script: it uses
// `declare module`, which is only legal at the top level of a global script
// OR a module. We keep it a global script so it applies without importing it.

/// <reference types="node" />
// ^ Required: the declarations below mention NodeJS.Timeout.

declare module "flaky-retry" {
  export interface RetryOptions {
    maxAttempts: number;
    baseDelayMs: number;
    maxDelayMs?: number;
    jitter?: boolean;
    onAttempt?: (attemptNumber: number, lastError: Error) => void;
  }

  export interface RetryHandle {
    cancel(): void;
    readonly timer: NodeJS.Timeout | null;
    //              ^^^^^^^^^^^^^^ this is why the reference exists
  }

  export function retry<T>(
    operation: () => Promise<T>,
    options: RetryOptions,
  ): Promise<T>;

  export function retryWithHandle<T>(
    operation: () => Promise<T>,
    options: RetryOptions,
  ): { promise: Promise<T>; handle: RetryHandle };
}
```

```ts
// ═════ 3. src/types/globals.d.ts ═════════════════════════════════════════════
// The CDN/script-tag build attaches a global. Consumers who use that build
// import "@acme/order-client/globals" to get the declaration.
// Global script — absolutely no top-level import/export.

/// <reference types="node" />
/// <reference lib="esnext.asynciterable" />
// ^ Our streaming API returns AsyncIterableIterator. Consumers whose tsconfig
//   lib is only ES2018 would otherwise get "Cannot find name
//   'AsyncIterableIterator'". Self-declaring the lib slice makes our .d.ts
//   portable across consumer configurations.

declare const AcmeOrderClient: {
  readonly version: string;
  create(options: { baseUrl: string; authToken: string }): {
    fetchOrder(orderId: number): Promise<unknown>;
  };
};

declare namespace AcmeOrderClientTypes {
  interface ApiErrorShape {
    statusCode: number;
    code: string;
    message: string;
    requestId: string;
  }
}
```

```ts
// ═════ 4. src/http/transport.ts ══════════════════════════════════════════════
// Ordinary module source. NO directives — it uses `import`, which is correct
// for a module. Note `import type {} from "node"` is unnecessary here because
// we import concrete things from node: specifiers.

import { Readable } from "node:stream";
import { setTimeout as sleep } from "node:timers/promises";
import { retry, type RetryOptions } from "flaky-retry";
//                    ^^^^^^^^^^^^ resolved from our hand-written .d.ts

export type UserRole = "admin" | "support" | "customer";

export interface AuthToken {
  readonly value: string;
  readonly userId: number;
  readonly role: UserRole;
  readonly expiresAt: number;
}

export type ApiResponse<T> =
  | { readonly ok: true; readonly statusCode: number; readonly data: T }
  | {
      readonly ok: false;
      readonly statusCode: number;
      readonly code: string;
      readonly message: string;
    };

export interface TransportOptions {
  readonly baseUrl: string;
  readonly authToken: AuthToken;
  readonly timeoutMs?: number;
  readonly retry?: Partial<RetryOptions>;
}

const DEFAULT_RETRY: RetryOptions = {
  maxAttempts: 3,
  baseDelayMs: 200,
  maxDelayMs: 5_000,
  jitter: true,
};

export class HttpTransport {
  private readonly baseUrl: string;
  private readonly authToken: AuthToken;
  private readonly timeoutMs: number;
  private readonly retryOptions: RetryOptions;

  constructor(options: TransportOptions) {
    this.baseUrl = options.baseUrl.replace(/\/+$/, "");
    this.authToken = options.authToken;
    this.timeoutMs = options.timeoutMs ?? 10_000;
    this.retryOptions = { ...DEFAULT_RETRY, ...options.retry };
  }

  private isExpired(): boolean {
    return this.authToken.expiresAt <= Date.now();
  }

  async requestJson<T>(
    method: "GET" | "POST" | "PATCH" | "DELETE",
    path: string,
    requestBody?: unknown,
  ): Promise<ApiResponse<T>> {
    if (this.isExpired()) {
      return {
        ok: false,
        statusCode: 401,
        code: "token_expired",
        message: `Auth token for user ${this.authToken.userId} has expired`,
      };
    }

    return retry(async () => {
      const controller = new AbortController();
      const timer = globalThis.setTimeout(() => controller.abort(), this.timeoutMs);
      //            ^^^^^^^^^^^^^^^^^^^^^ typed by @types/node, which our
      //            tsconfig "types" provides for the build

      try {
        const response = await fetch(`${this.baseUrl}${path}`, {
          method,
          signal: controller.signal,
          headers: {
            "content-type": "application/json",
            authorization: `Bearer ${this.authToken.value}`,
          },
          body: requestBody === undefined ? undefined : JSON.stringify(requestBody),
        });

        if (!response.ok) {
          const errorBody = (await response.json().catch(() => ({}))) as {
            code?: string;
            message?: string;
          };
          return {
            ok: false as const,
            statusCode: response.status,
            code: errorBody.code ?? "unknown_error",
            message: errorBody.message ?? response.statusText,
          };
        }

        return {
          ok: true as const,
          statusCode: response.status,
          data: (await response.json()) as T,
        };
      } finally {
        globalThis.clearTimeout(timer);
      }
    }, this.retryOptions);
  }

  // Buffer and Readable appear in the PUBLIC signature — this is what forces
  // `/// <reference types="node" />` into the emitted .d.ts.
  async downloadInvoice(orderId: number): Promise<Buffer> {
    const response = await fetch(`${this.baseUrl}/orders/${orderId}/invoice`, {
      headers: { authorization: `Bearer ${this.authToken.value}` },
    });
    const arrayBuffer = await response.arrayBuffer();
    return Buffer.from(arrayBuffer);
  }

  streamOrderEvents(userId: number): Readable {
    const stream = new Readable({ objectMode: true, read() {} });
    void (async () => {
      for (let page = 0; page < 10; page += 1) {
        const result = await this.requestJson<{ events: unknown[] }>(
          "GET",
          `/users/${userId}/events?page=${page}`,
        );
        if (!result.ok) break;
        for (const event of result.data.events) stream.push(event);
        await sleep(250);
      }
      stream.push(null);
    })();
    return stream;
  }
}
```

```ts
// ═════ 5. dist/http/transport.d.ts — WHAT tsc EMITS ══════════════════════════
// You do not write this file. tsc generates it, and it AUTOMATICALLY inserts
// the reference directive because the public API mentions `Buffer`.

/// <reference types="node" />
//  ^^^^^^^^^^^^^^^^^^^^^^^^^^ compiler-generated

import { Readable } from "node:stream";
import { type RetryOptions } from "flaky-retry";

export type UserRole = "admin" | "support" | "customer";

export interface AuthToken {
  readonly value: string;
  readonly userId: number;
  readonly role: UserRole;
  readonly expiresAt: number;
}

export type ApiResponse<T> =
  | { readonly ok: true; readonly statusCode: number; readonly data: T }
  | {
      readonly ok: false;
      readonly statusCode: number;
      readonly code: string;
      readonly message: string;
    };

export declare class HttpTransport {
  constructor(options: TransportOptions);
  requestJson<T>(
    method: "GET" | "POST" | "PATCH" | "DELETE",
    path: string,
    requestBody?: unknown,
  ): Promise<ApiResponse<T>>;
  downloadInvoice(orderId: number): Promise<Buffer>;
  streamOrderEvents(userId: number): Readable;
}

export interface TransportOptions {
  readonly baseUrl: string;
  readonly authToken: AuthToken;
  readonly timeoutMs?: number;
  readonly retry?: Partial<RetryOptions>;
}

// ⚠️ THE CONSUMER CONSEQUENCE
// A consumer with "types": [] in their tsconfig, or without @types/node
// installed, now sees:
//     node_modules/@acme/order-client/dist/http/transport.d.ts:1:1
//     error TS2688: Cannot find type definition file for 'node'.
//
// Which is exactly why @types/node is in our `dependencies`, not
// `devDependencies`. Getting this wrong is one of the most common
// bug reports on typed npm packages.
```

```jsonc
// ═════ 6. tsconfig.json — the build ══════════════════════════════════════════
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["ES2022"],
    "module": "nodenext",
    "moduleResolution": "nodenext",
    "outDir": "./dist",
    "rootDir": "./src",
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "strict": true,
    "verbatimModuleSyntax": true,
    "isolatedModules": true,
    "skipLibCheck": true,

    // Only @types/node is global in src. @types/jest is deliberately absent,
    // so `describe` / `it` / `expect` are compile errors in production code.
    "types": ["node"]
  },
  "include": ["src/**/*.ts", "src/**/*.d.ts"],
  "exclude": ["src/**/*.test.ts", "dist", "node_modules"]
}
```

```jsonc
// ═════ 7. tsconfig.test.json — tests get jest globals ════════════════════════
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "noEmit": true,
    // ⚠️ `types` is REPLACED, not merged, by `extends`. You must relist "node".
    "types": ["node", "jest"]
  },
  "include": ["src/**/*.ts", "src/**/*.d.ts", "test/**/*.ts"],
  "exclude": ["dist", "node_modules"]
}
```

```ts
// ═════ 8. test/transport.test.ts ═════════════════════════════════════════════
// If you were NOT maintaining a separate test tsconfig, you would put a
// directive here instead. Shown so you can see both shapes side by side.

/// <reference types="jest" />
// ^ Redundant given tsconfig.test.json, but harmless and self-documenting.
//   Some teams keep it so the file type-checks even under the prod tsconfig.

import { HttpTransport, type AuthToken } from "../src/http/transport.js";

const validToken: AuthToken = {
  value: "eyJhbGciOiJIUzI1NiJ9.test",
  userId: 4711,
  role: "admin",
  expiresAt: Date.now() + 60_000,
};

const expiredToken: AuthToken = { ...validToken, expiresAt: Date.now() - 1_000 };

describe("HttpTransport", () => {
  it("short-circuits when the auth token has expired", async () => {
    const transport = new HttpTransport({
      baseUrl: "https://api.acme.dev",
      authToken: expiredToken,
    });

    const response = await transport.requestJson<{ orderId: number }>(
      "GET",
      "/orders/9001",
    );

    expect(response.ok).toBe(false);
    if (!response.ok) {
      expect(response.statusCode).toBe(401);
      expect(response.code).toBe("token_expired");
      expect(response.message).toContain(String(expiredToken.userId));
    }
  });

  it("normalises a trailing slash in baseUrl", () => {
    const transport = new HttpTransport({
      baseUrl: "https://api.acme.dev///",
      authToken: validToken,
    });
    expect(transport).toBeInstanceOf(HttpTransport);
  });
});
```

```ts
// ═════ 9. THE AUDIT — every directive in this package, justified ═════════════
//
// src/types/legacy-retry.d.ts
//   /// <reference types="node" />
//   ✅ NECESSARY. Global script (declare module), mentions NodeJS.Timeout.
//      An `import type {} from "node"` would make it a module and break
//      the ambient `declare module "flaky-retry"` for consumers.
//
// src/types/globals.d.ts
//   /// <reference types="node" />
//   ✅ NECESSARY. Same reason.
//   /// <reference lib="esnext.asynciterable" />
//   ✅ NECESSARY. Published .d.ts must be portable to consumers with old `lib`.
//
// src/http/transport.ts
//   (no directives)
//   ✅ CORRECT. It is a module; `import` covers everything.
//
// dist/**/*.d.ts
//   /// <reference types="node" />
//   ✅ COMPILER-GENERATED and correct. Backed by @types/node in `dependencies`.
//
// test/transport.test.ts
//   /// <reference types="jest" />
//   ⚠️ OPTIONAL. tsconfig.test.json already supplies it. Kept for robustness;
//      a team that dislikes redundancy would delete it.
//
// ANYWHERE
//   /// <reference path="..." />
//   ❌ NOT USED. `include` covers file inclusion, `import` covers dependencies,
//      and `outDir` means emit order is irrelevant. Correct for a modern package.
```

---

## Going deeper

### 1. `types` vs `import type {} from` vs a reference directive

Three ways to load a package's globals. They are not interchangeable.

```ts
// ── (a) tsconfig "types" ─────────────────────────────────────────────────────
// Scope:     entire program
// Location:  tsconfig.json
// Works in:  everything
// Downside:  a per-tsconfig decision; a file copied elsewhere loses it.

// ── (b) /// <reference types="node" /> ───────────────────────────────────────
// Scope:     entire program (despite being written in one file)
// Location:  top of any .ts or .d.ts
// Works in:  global scripts AND modules
// Downside:  invisible to bundlers' dependency graphs; silently inert if
//            it drifts below an import; leaks globals program-wide.
// UNIQUE:    the only option inside a .d.ts that must stay a global script.

// ── (c) import type {} from "node" ───────────────────────────────────────────
// Scope:     entire program (globals land globally once the file is loaded)
// Location:  top of a module file
// Works in:  module files only — using it MAKES the file a module
// Upside:    real module syntax; understood by every tool; survives
//            `verbatimModuleSyntax`; movable within the file.
// Downside:  destroys globalness of a .d.ts.

// Rule of thumb for a modern backend:
//   program-wide need  → (a)
//   global .d.ts       → (b)
//   module file need   → (c)
```

### 2. `skipLibCheck` and why it hides directive bugs

```jsonc
{ "compilerOptions": { "skipLibCheck": true } }
```

```ts
// skipLibCheck: true tells tsc not to TYPE CHECK the contents of any .d.ts.
// It does NOT stop tsc from RESOLVING reference directives inside them.
//
// So with skipLibCheck on:
//   - A broken `/// <reference types="does-not-exist" />` inside a dependency's
//     .d.ts STILL produces TS2688. Resolution is not skipped.
//   - But conflicting global declarations between two @types packages
//     (the classic @types/node vs @types/bun vs DOM `fetch` clash) go quiet.
//
// Consequence: skipLibCheck makes over-broad automatic @types inclusion
// FEEL fine, right up until a real conflict reaches your own source files.
// Turning it off temporarily is a great way to audit what your program
// actually pulled in.
```

### 3. Diagnosing what got included: `--traceResolution` and `--listFiles`

```bash
# Every file in the program, in inclusion order:
npx tsc --noEmit --listFiles

# Only the files, without checking (fast):
npx tsc --noEmit --listFilesOnly

# Why a `types` reference resolved (or didn't) — very verbose:
npx tsc --noEmit --traceResolution | grep -i "type reference"

# Show the effective config after `extends` merging:
npx tsc --showConfig

# Find every directive in your own source:
grep -rn '/// *<reference' src/

# Find directives in your dependencies' published types (the ones that
# silently expand YOUR program):
grep -rn '/// *<reference types' node_modules/*/dist/*.d.ts 2>/dev/null | head -50
```

```ts
// Typical revelation from --listFiles on a "small" service:
//   node_modules/@types/node/index.d.ts
//   node_modules/@types/node/globals.d.ts
//   ... 60 more @types/node files ...
//   node_modules/@types/jest/index.d.ts        ← why is this here?
//   node_modules/@types/express/index.d.ts
//   node_modules/@types/serve-static/index.d.ts
//   node_modules/@types/qs/index.d.ts          ← transitively, via express
//   node_modules/@types/range-parser/index.d.ts
//
// Setting "types": ["node"] cuts that list dramatically and speeds up tsc,
// because automatic inclusion is what dragged most of it in.
```

### 4. The `vite/client` and `vitest/globals` pattern

```ts
// ── src/vite-env.d.ts — created by `npm create vite` ─────────────────────────
/// <reference types="vite/client" />

// That one line gives you:
//   - `import.meta.env.VITE_API_URL` typed
//   - `import.meta.hot` typed
//   - module declarations for `*.svg`, `*.css`, `*.png`, `?raw`, `?url`
//
// It is a package SUBPATH reference: vite/client resolves to
// node_modules/vite/client.d.ts. This is the most common reference directive
// a working developer will ever see.

// You extend it in the same file, which must stay a global script:
interface ImportMetaEnv {
  readonly VITE_API_BASE_URL: string;
  readonly VITE_SENTRY_DSN?: string;
}

interface ImportMeta {
  readonly env: ImportMetaEnv;
}
```

```ts
// ── Vitest globals, the two ways ─────────────────────────────────────────────

// Way 1 — tsconfig:
//   { "compilerOptions": { "types": ["vitest/globals"] } }

// Way 2 — a directive in a .d.ts:
/// <reference types="vitest/globals" />

// Both make `describe`/`it`/`expect` global, and both require
// `globals: true` in vitest.config.ts to actually exist at runtime.
// The directive form is common in `src/vitest-env.d.ts`.
```

### 5. Monorepos: where `typeRoots` bites

```jsonc
// packages/orders-api/tsconfig.json
{
  "compilerOptions": {
    "typeRoots": ["./node_modules/@types"]
  }
}
// ❌ In a pnpm/yarn workspace, @types/node is usually hoisted to
//    <repo-root>/node_modules/@types. This config REPLACES the default
//    upward walk with a single directory that doesn't contain it.
//    Result: "Cannot find name 'process'" in one package but not others.
```

```jsonc
// ✅ Either omit typeRoots entirely (let the upward walk work):
{
  "compilerOptions": {
    "types": ["node"]
  }
}

// ✅ Or list every root explicitly, including the workspace root:
{
  "compilerOptions": {
    "typeRoots": [
      "./node_modules/@types",
      "../../node_modules/@types",
      "./src/local-types"
    ],
    "types": ["node", "acme-internal-globals"]
  }
}
```

```ts
// The other monorepo bite: a path reference that escapes rootDir.
//
// packages/orders-api/src/index.ts
//   /// <reference path="../../shared/src/globals.d.ts" />
//
// → error TS6059: File '.../shared/src/globals.d.ts' is not under 'rootDir'.
//   'rootDir' is expected to contain all source files.
//
// Fix: use PROJECT references (tsconfig "references") instead. That is the
// feature designed for cross-package composition — file reference paths
// are not.
```

### 6. Reference directives and `verbatimModuleSyntax` / `isolatedModules`

```ts
// Reference directives are pure comments to any transpiler. That has two faces:
//
// GOOD: esbuild, swc, Babel, Bun, and Node's type-stripping all ignore them
//       harmlessly. Nothing breaks at runtime. Unlike `const enum`, a
//       reference directive is 100% erasable syntax.
//
// BAD:  a single-file transpiler cannot ACT on them either. If a directive
//       is what makes your types resolve, `tsc --noEmit` is the only thing
//       that will ever notice a broken one. Your bundler will happily ship
//       a build whose types never checked.
//
// Practical rule: never rely on a directive for correctness of RUNTIME
// behaviour (only `path` references with `outFile` ever did, and that
// combination is dead). Rely on them only for type resolution, and run
// `tsc --noEmit` in CI so broken ones are caught.
```

### 7. What happens when a reference cannot be resolved

```ts
/// <reference types="does-not-exist" />
// ❌ error TS2688: Cannot find type definition file for 'does-not-exist'.
//    The file is in the program because of the 'types' reference.

/// <reference path="./missing.d.ts" />
// ❌ error TS6053: File './missing.d.ts' not found.

/// <reference lib="es2099" />
// ❌ error TS6231-ish: Cannot find lib definition for 'es2099'.

/// <reference types="node" />   // when @types/node is genuinely absent
// ❌ error TS2688: Cannot find type definition file for 'node'.
//    Fix: npm i -D @types/node   (or -S if you publish .d.ts that reference it)

// Note the important asymmetry: these ARE errors. Unlike a misplaced
// directive, which is silent, an unresolvable one is loud. If you want the
// safety, that argues for keeping directives — but only where they're needed.
```

### 8. `/// <reference path>` on a `.js` file, and `allowJs`

```ts
// With "allowJs": true and "checkJs": true, reference directives work in
// .js files too — this is how large JSDoc-typed JavaScript codebases
// (before migrating to .ts) get their ambient types:

// ── src/legacy/worker.js ─────────────────────────────────────────────────────
/// <reference path="../types/domain.d.ts" />
/// <reference types="node" />

/**
 * @param {number} userId
 * @param {OrderRecord[]} orders   ← global type from domain.d.ts
 * @returns {Promise<Buffer>}      ← global type from @types/node
 */
async function buildOrderReport(userId, orders) {
  const totalCents = orders.reduce((sum, order) => sum + order.totalCents, 0);
  return Buffer.from(JSON.stringify({ userId, totalCents }));
}

module.exports = { buildOrderReport };
```

### 9. The historical arc, in one table

| Era | How files were composed | How globals arrived |
|---|---|---|
| TS 0.8–1.4 (2012–2015) | `/// <reference path>` + `--out` | `/// <reference path>` to a `.d.ts` |
| TS 1.5–1.8 (2015–2016) | `import`/`export` (ES modules) arrive; `tsconfig#files` | `typings` CLI + `/// <reference path>` |
| TS 2.0 (2016) | `tsconfig#include`/`exclude` | `@types` on npm; automatic inclusion; `types`/`typeRoots`; `/// <reference types>` introduced |
| TS 3.0 (2018) | Project references (`tsconfig#references`) | unchanged |
| TS 3.x–5.x | `moduleResolution: node16/nodenext/bundler`, `exports` maps | `import type {} from "pkg"`; `types` still king |

```ts
// The takeaway: `/// <reference types>` was introduced in 2016 SPECIFICALLY
// because @types packages needed a way to declare dependencies on each other
// without becoming modules. That is still exactly the job it does. Everything
// else about triple slash directives is a fossil.
```

### 10. When you still legitimately write one — the complete list

```ts
// 1. In a global .d.ts that needs another package's types.
//      /// <reference types="node" />
//    Cannot be replaced by an import without destroying globalness.
//    THE dominant real case.

// 2. In a published library's .d.ts that uses lib features newer than a
//    consumer's `lib` setting.
//      /// <reference lib="esnext.asynciterable" />

// 3. To opt a scaffolded project into a framework's ambient types.
//      /// <reference types="vite/client" />
//      /// <reference types="astro/client" />
//      /// <reference types="@cloudflare/workers-types" />
//    Convention-driven; every one of these could be tsconfig `types` instead,
//    but the ecosystem standardised on the directive.

// 4. In a one-off script or test file that needs globals the project-wide
//    tsconfig deliberately excludes.
//      /// <reference types="jest" />

// 5. Inside TypeScript's own lib files.
//      /// <reference no-default-lib="true" />
//    Not you.

// ❌ NOT on this list: `/// <reference path="..." />` in application code.
//    If you are writing one of those in a project with `include` and
//    `outDir`, you have a tsconfig problem, not a directive problem.
```

---

## Common mistakes

### Mistake 1 — Putting the directive below an import (or any statement)

```ts
// ❌ WRONG — silently inert
import express from "express";
import { Router } from "express";

/// <reference types="node" />
// ^ Just a comment. tsc's preprocessor stopped scanning at the first import.

const uploadDir = process.env.UPLOAD_DIR;
// ❌ error TS2580: Cannot find name 'process'. Do you need to install
//    type definitions for node? Try `npm i --save-dev @types/node`.
//
// And the failure mode is worse in a .d.ts: the reference is ignored, the
// types it needed are missing, and the resulting error message points at the
// USE of the type, never at the misplaced directive.
```

```ts
// ✅ RIGHT — directives before every statement; only comments may precede
/// <reference types="node" />

import express from "express";
import { Router } from "express";

const uploadDir = process.env.UPLOAD_DIR;   // ✅ string | undefined
```

```ts
// ✅ ALSO RIGHT — comments above are fine, including license headers,
//    eslint directives, and @ts-check
/*!
 * @acme/uploads — Copyright (c) 2026 Acme Corp. MIT licensed.
 */
// @ts-check
/// <reference types="node" />

import express from "express";
```

### Mistake 2 — Adding an `import` or `export` to a global `.d.ts`

```ts
// ❌ WRONG — the file becomes a module; every declaration stops being global
// src/types/env.d.ts

import type { UserRole } from "../models/user.js";
//     ^^^^^^ this single line converts the file into a module

declare namespace NodeJS {
  interface ProcessEnv {
    DATABASE_URL: string;
    DEFAULT_ROLE: UserRole;
  }
}
// The `declare namespace NodeJS` now creates a namespace INSIDE this module.
// It augments nothing. `process.env.DATABASE_URL` remains
// `string | undefined` throughout the codebase.
//
// NO ERROR IS REPORTED. This is the single nastiest bug in this document.
```

```ts
// ✅ RIGHT (option A) — keep it a global script; get external types via a
//    reference directive, and inline or re-declare what you need.
// src/types/env.d.ts
/// <reference types="node" />

declare namespace NodeJS {
  interface ProcessEnv {
    DATABASE_URL: string;
    DEFAULT_ROLE: "admin" | "support" | "customer";
    //            ^ inlined, because importing UserRole would break globalness
  }
}
```

```ts
// ✅ RIGHT (option B) — make it a module ON PURPOSE and use `declare global`
// src/types/env.d.ts
import type { UserRole } from "../models/user.js";

declare global {
  namespace NodeJS {
    interface ProcessEnv {
      DATABASE_URL: string;
      DEFAULT_ROLE: UserRole;
    }
  }
}

export {};
// `declare global` is legal ONLY in a module, which is why option B needs
// the import (or the trailing `export {}`) to work at all.
// Both options are correct; the mistake is doing half of each.
```

### Mistake 3 — Using `/// <reference path>` where `include` belongs

```ts
// ❌ WRONG — hand-maintained dependency graph in a modern project
// src/index.ts
/// <reference path="./types/domain.d.ts" />
/// <reference path="./types/env.d.ts" />
/// <reference path="./types/build-globals.d.ts" />
/// <reference path="./types/express.d.ts" />
/// <reference path="./types/legacy-retry.d.ts" />

import { startServer } from "./server.js";
startServer();

// Problems:
//   - Add a sixth .d.ts and you must remember to add a sixth line.
//   - Rename a file and this breaks with TS6053 at a distance.
//   - Any OTHER entry point (a worker, a migration script, a test) needs the
//     same five lines copy-pasted, or it silently lacks the types.
//   - The paths are relative to THIS file, so moving index.ts breaks all five.
//   - `tsc --noEmit` on a single file no longer reflects the real program.
```

```jsonc
// ✅ RIGHT — declare the file set once, in tsconfig
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["ES2022"],
    "module": "nodenext",
    "moduleResolution": "nodenext",
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "types": ["node"]
  },
  "include": ["src/**/*.ts", "src/**/*.d.ts"],
  "exclude": ["node_modules", "dist"]
}
```

```ts
// ✅ src/index.ts — no directives at all
import { startServer } from "./server.js";
startServer();
// Every .d.ts under src/ is already in the program, for every entry point,
// forever, with no maintenance.
```

### Mistake 4 — Publishing `/// <reference types="node" />` without a real dependency

```jsonc
// ❌ WRONG — package.json
{
  "name": "@acme/order-client",
  "types": "./dist/index.d.ts",
  "devDependencies": {
    "@types/node": "^22.0.0"     // ← devDependency only
  }
}
```

```ts
// dist/index.d.ts (emitted, because the API returns Buffer)
/// <reference types="node" />
export declare function downloadInvoice(orderId: number): Promise<Buffer>;

// A consumer installs @acme/order-client. npm does NOT install our
// devDependencies. Their build fails with:
//     node_modules/@acme/order-client/dist/index.d.ts:1:1 -
//     error TS2688: Cannot find type definition file for 'node'.
//
// And they cannot fix it without understanding a feature they've never used.
```

```jsonc
// ✅ RIGHT (option A) — make it a real dependency
{
  "dependencies": {
    "@types/node": "^22.0.0"
  }
}
```

```ts
// ✅ RIGHT (option B) — don't leak Node types into the public API at all
export declare function downloadInvoice(orderId: number): Promise<Uint8Array>;
//                                                        ^^^^^^^^^^ a standard
// ES2015 type from lib.es2015.d.ts. No @types/node needed by consumers.
// tsc emits no reference directive. Best choice when it's viable — Buffer
// IS a Uint8Array at runtime, so this is a free win for most APIs.
```

### Mistake 5 — Assuming a directive's effect is file-local

```ts
// ❌ WRONG MENTAL MODEL
// test/order-service.test.ts
/// <reference types="jest" />
// "This makes jest globals available in THIS file."

// src/services/order-service.ts   — no directive
export function processRefund(orderId: number): void {
  describe("oops", () => {});
  //  ^^^^^^^^ ✅ COMPILES. `describe` is global here too.
  //  The directive in the test file added @types/jest to the PROGRAM,
  //  and @types/jest declares its helpers globally. There is no scoping.
}
// You can now ship test helpers into production code and tsc will not stop you.
```

```jsonc
// ✅ RIGHT — enforce the boundary with separate tsconfigs
// tsconfig.json (production)
{
  "compilerOptions": { "types": ["node"] },
  "include": ["src/**/*.ts"],
  "exclude": ["src/**/*.test.ts"]
}
```

```jsonc
// tsconfig.test.json
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "noEmit": true,
    "types": ["node", "jest"]
  },
  "include": ["src/**/*.ts", "test/**/*.ts"]
}
```

```ts
// ✅ Now, under the production tsconfig:
// src/services/order-service.ts
export function processRefund(orderId: number): void {
  describe("oops", () => {});
  // ❌ error TS2304: Cannot find name 'describe'.
  // Exactly the guardrail you wanted.
}
```

### Mistake 6 — Setting `typeRoots` and losing everything else

```jsonc
// ❌ WRONG — you wanted to ADD a local type package, so you set typeRoots.
{
  "compilerOptions": {
    "typeRoots": ["./src/types-packages"]
  }
}
// You just replaced the default upward walk. node_modules/@types is no longer
// searched at all. Result:
//   error TS2688: Cannot find type definition file for 'node'.
//   error TS2580: Cannot find name 'process'.
//   error TS2304: Cannot find name 'describe'.
// ...and hundreds more, all at once, for a "small" config change.
```

```jsonc
// ✅ RIGHT — typeRoots is a REPLACEMENT list, so list everything you need
{
  "compilerOptions": {
    "typeRoots": [
      "./node_modules/@types",
      "./src/types-packages"
    ],
    "types": ["node", "acme-internal-globals"]
  }
}
```

```ts
// ✅ ALTERNATIVE — skip typeRoots entirely and just include the .d.ts files
//    via `include`. A local global .d.ts does not need to be a "type package".
//
// tsconfig.json:
//   { "include": ["src/**/*.ts", "src/**/*.d.ts"] }
//
// src/types/acme-globals.d.ts:
//   /// <reference types="node" />
//   declare const __ACME_REGION__: "us-east-1" | "eu-west-1";
//
// Simpler, fewer moving parts, and it survives being moved between repos.
```

---

## Practice exercises

### Exercise 1 — easy

Set up the ambient type layer for a small Node service, using each directive exactly where it belongs.

Requirements:

1. Create `src/types/env.d.ts` as a **global script** (no top-level import/export). Give it a `/// <reference types="node" />` and augment `NodeJS.ProcessEnv` with: `DATABASE_URL: string`, `REDIS_URL: string`, `JWT_SECRET: string`, `PORT?: string`, and `NODE_ENV: "development" | "test" | "production"`.
2. Create `src/types/build-globals.d.ts` declaring `__BUILD_SHA__: string`, `__DEPLOYED_AT__: number`, and a global `interface HealthCheckResult { healthy: boolean; checkedAt: number; details: Record<string, string> }`. Work out whether this file needs a directive, and be able to explain why.
3. Write `src/config.ts` exporting `interface AppConfig` (readonly fields for `port`, `databaseUrl`, `redisUrl`, `jwtSecret`, `buildSha`, `isProduction`) and `loadConfig(): AppConfig`, which throws a descriptive `Error` if `JWT_SECRET` is shorter than 32 characters.
4. Write `src/health.ts` exporting `buildHealthCheck(dependencies: Record<string, boolean>): HealthCheckResult` — using the global interface, with no import.
5. Write the `tsconfig.json`: `target`/`lib` ES2022, `module`/`moduleResolution` nodenext, `strict`, `outDir` `./dist`, `types: ["node"]`, and an `include` that picks up both `.ts` and `.d.ts` under `src`. Deliberately do **not** include the DOM lib.
6. In a comment at the bottom of `config.ts`, state exactly what error appears if you (a) delete the reference directive from `env.d.ts` *and* remove `types` from tsconfig, and (b) add `export {}` to the bottom of `env.d.ts`.

```ts
// Write your code here
```

### Exercise 2 — medium

Take a legacy service that composes its files with `/// <reference path="" />` and `outFile`, and migrate it to modules — then decide, per directive, which ones survive.

Starting point (rewrite it yourself — do not copy mechanically):

```ts
// src/models.ts        — global script, no exports
//   interface OrderRecord { orderId: number; userId: number; totalCents: number;
//                           status: string; createdAt: Date }
//   interface AuthToken  { userId: number; role: string; expiresAt: number }
//
// src/db.ts
//   /// <reference path="./models.ts" />
//   var CONNECTION_STRING = "postgres://localhost/orders";
//   function findOrderById(orderId: number): OrderRecord | null { ... }
//   function findOrdersByUser(userId: number): OrderRecord[] { ... }
//
// src/auth.ts
//   /// <reference path="./models.ts" />
//   function verifyToken(rawToken: string): AuthToken | null { ... }
//
// src/server.ts
//   /// <reference path="./db.ts" />
//   /// <reference path="./auth.ts" />
//   function handleGetOrder(orderId: number, rawToken: string) { ... }
//
// tsconfig.json
//   { "compilerOptions": { "module": "none", "outFile": "./dist/bundle.js" },
//     "files": ["src/server.ts"] }
```

Write, from scratch:

1. `src/models.ts` as a real module exporting `OrderRecord`, `AuthToken`, `OrderStatus` (a literal union: `"pending" | "paid" | "shipped" | "cancelled"`) and `UserRole` (`"admin" | "support" | "customer"`). Note that `status` and `role` must now be properly typed, not `string`.
2. `src/db.ts` as a module with `import type` for the models, exporting `findOrderById`, `findOrdersByUser`, and a `CONNECTION_STRING` read from `process.env.DATABASE_URL`.
3. `src/auth.ts` as a module exporting `verifyToken(rawToken: string): AuthToken | null` and `requireRole(authToken: AuthToken, role: UserRole): void` (throws on mismatch).
4. `src/server.ts` as a module exporting `handleGetOrder(orderId: number, rawToken: string): ApiResponse<OrderRecord>`, where `ApiResponse<T>` is a discriminated union on `ok`.
5. `src/types/env.d.ts` — the ONE file where a triple slash directive survives the migration. It must augment `NodeJS.ProcessEnv` with `DATABASE_URL: string`.
6. The new `tsconfig.json`: `outDir` instead of `outFile`, `include` instead of `files`, `module: "nodenext"`, `types: ["node"]`.
7. A comment block titled `// DIRECTIVE AUDIT` listing every original directive and stating, for each, whether it was **deleted** (and what replaced it) or **kept** (and why nothing else could do the job).
8. Bonus: explain in a comment why the original `outFile` build needed the reference in `server.ts` to `db.ts`, and why the module version does not — referencing what changed about emit order.

```ts
// Write your code here
```

### Exercise 3 — hard

Build the complete type surface of a **publishable** npm package, `@acme/job-queue`, where every reference directive is either provably necessary or provably removable.

Part A — the package source:

```ts
// Implement, with no `any`:
//
// src/types/legacy-amqp.d.ts
//   Hand-written ambient declarations for `flaky-amqp`, an untyped JS
//   dependency. Must be a GLOBAL SCRIPT using `declare module "flaky-amqp"`.
//   The declared API must mention NodeJS.Timeout and Buffer, forcing a
//   /// <reference types="node" /> that you must justify.
//     - interface AmqpChannel { publish(exchange, routingKey, payload: Buffer): boolean;
//                               consume(queue, handler: (msg: Buffer) => void): string;
//                               readonly heartbeatTimer: NodeJS.Timeout | null }
//     - function connect(url: string): Promise<AmqpChannel>
//
// src/types/globals.d.ts
//   A global script declaring `__QUEUE_BUILD_SHA__: string` and a global
//   namespace `AcmeJobQueue` containing `interface TelemetryHook {
//     onEnqueue(jobId: string, queueName: string): void;
//     onFailure(jobId: string, error: Error, attemptNumber: number): void }`.
//   Decide whether it needs a directive; justify either way.
//
// src/job.ts  (a MODULE)
//   - type JobStatus = "queued" | "running" | "succeeded" | "failed" | "dead"
//   - type QueueName = "emails" | "invoices" | "webhooks"
//   - interface JobRecord<TPayload> { jobId: string; queueName: QueueName;
//       payload: TPayload; status: JobStatus; attemptNumber: number;
//       maxAttempts: number; enqueuedAt: number; lastError?: string }
//   - type JobHandler<TPayload> = (job: JobRecord<TPayload>) => Promise<void>
//   - a discriminated ApiResponse<T> on `ok`
//
// src/queue.ts  (a MODULE)
//   - class JobQueue with:
//       constructor(options: { amqpUrl: string; telemetry?: AcmeJobQueue.TelemetryHook })
//       enqueue<TPayload>(queueName: QueueName, payload: TPayload,
//                         opts?: { maxAttempts?: number }): Promise<JobRecord<TPayload>>
//       register<TPayload>(queueName: QueueName, handler: JobHandler<TPayload>): void
//       drain(queueName: QueueName): Promise<number>
//       streamCompleted(queueName: QueueName): AsyncIterableIterator<JobRecord<unknown>>
//       exportDeadLetters(queueName: QueueName): Promise<Buffer>
//   - Exhaustive switch over JobStatus with an assertNever default.
//
// src/index.ts — the public barrel.
```

Part B — the directive engineering:

```ts
// 1. Write BOTH tsconfigs: tsconfig.json (build, types: ["node"], declaration:
//    true, isolatedModules, verbatimModuleSyntax) and tsconfig.test.json
//    (extends it, types: ["node", "jest"], noEmit).
//
// 2. Hand-write what `tsc --declaration` WILL emit as the first line of
//    dist/queue.d.ts, and explain which member of JobQueue forces it.
//
// 3. Write the package.json, correctly deciding whether @types/node belongs
//    in `dependencies` or `devDependencies`. Justify in a comment with the
//    exact error a consumer would see if you get it wrong.
//
// 4. `streamCompleted` returns AsyncIterableIterator. Add whatever directive
//    makes the published .d.ts portable to a consumer whose tsconfig `lib`
//    is only ES2018 — and explain why tsconfig `lib` alone cannot solve this
//    for consumers.
//
// 5. Produce a REMOVAL PLAN: rewrite `exportDeadLetters` so the package no
//    longer forces `/// <reference types="node" />` into its public .d.ts at
//    all, and state what else would have to change in legacy-amqp.d.ts to
//    finish the job. Explain the trade-off you're making.

// Part C — prove it:
// - Show the exact error when a top-level `export {}` is added to globals.d.ts.
// - Show the exact error when the reference in legacy-amqp.d.ts is moved
//   below the `declare module` line.
// - Show the exact error a consumer with "types": [] sees before your fix in
//   Part B.5, and show it disappearing after.
// - Show that `describe` is a compile error in src/queue.ts under
//   tsconfig.json but not under tsconfig.test.json.
```

```ts
// Write your code here
```

---

## Quick reference cheat sheet

```ts
// ══ THE FOUR DIRECTIVES ══════════════════════════════════════════════════════
/// <reference path="./types/domain.d.ts" />   // include a FILE (+ emit order)
/// <reference types="node" />                 // include a @types PACKAGE (globals)
/// <reference types="vite/client" />          // package SUBPATH also works
/// <reference lib="es2023.array" />           // include a built-in LIB file
/// <reference no-default-lib="true" />        // suppress lib.d.ts — never write this

// ══ PLACEMENT ════════════════════════════════════════════════════════════════
// ✅ Top of file. Only comments / shebang may precede.
// ❌ Below ANY statement (including "use strict" and imports) → silent no-op.
// ❌ Inside a /* block comment */ → not a directive.
// ❌ Four slashes, or a space after the slashes → not a directive.

// ══ SCOPE ════════════════════════════════════════════════════════════════════
// All directives affect the WHOLE PROGRAM, not just the file they're in.
// A `types="jest"` in one test file makes `describe` global in src/ too.

// ══ .d.ts MODULE vs GLOBAL SCRIPT — the rule everything hinges on ════════════
// Top-level import/export present  → MODULE  → declarations are module-scoped
//                                            → use `declare global` / `declare module "x"`
//                                            → use `import type {} from "node"`
// No top-level import/export       → GLOBAL  → declarations are global
//                                            → MUST use /// <reference types="..." />
```

```jsonc
// ══ MODERN EQUIVALENTS (tsconfig.json) ═══════════════════════════════════════
{
  "compilerOptions": {
    "types": ["node", "jest"],       // ↔ /// <reference types="..." />  (program-wide)
    "typeRoots": ["./node_modules/@types", "./src/types"],  // WHERE to look; REPLACES default
    "lib": ["ES2022"],               // ↔ /// <reference lib="..." />
    "noLib": false,                  // ↔ /// <reference no-default-lib="true" />
    "noResolve": false,              // true = stop following imports AND directives
    "skipLibCheck": true             // skips CHECKING .d.ts, still RESOLVES their directives
  },
  "files": ["src/index.ts"],         // explicit roots, no globs
  "include": ["src/**/*.ts", "src/**/*.d.ts"],  // ↔ /// <reference path="..." />
  "exclude": ["dist"],               // filters `include` ONLY; cannot un-reach a file
  "references": [{ "path": "../shared" }]  // PROJECT refs — a totally different feature
}
```

```ts
// ══ DEFAULTS WORTH KNOWING ═══════════════════════════════════════════════════
// No "types" key      → ALL packages under every typeRoot are auto-included.
// No "typeRoots" key  → walk node_modules/@types upward to the filesystem root.
// No "lib" key        → derived from "target" (and INCLUDES "DOM" — set it
//                       explicitly on a backend to keep DOM types out).
// "extends"           → "types" and "lib" are REPLACED, never merged.

// ══ DIAGNOSTICS ══════════════════════════════════════════════════════════════
// npx tsc --noEmit --listFilesOnly            what's actually in the program
// npx tsc --noEmit --traceResolution          why a types= reference resolved
// npx tsc --showConfig                        effective config after `extends`
// grep -rn '/// *<reference' src/             every directive you own

// ══ ERRORS ═══════════════════════════════════════════════════════════════════
// TS2688  Cannot find type definition file for 'X'    → types= unresolved
// TS6053  File 'X' not found                          → path= unresolved
// TS2503  Cannot find namespace 'NodeJS'              → missing @types/node
// TS2580  Cannot find name 'process'                  → missing @types/node
// TS2304  Cannot find name 'describe'                 → @types/jest not included
// TS2318  Cannot find global type 'String'            → no-default-lib is active
// TS2669  Augmentations for the global scope can only be directly nested in
//         external modules  → you used `declare global` in a global script
// TS6059  File is not under 'rootDir'                 → path= escaped rootDir
// TS1435  Unknown directive attribute                 → typo in the tag
```

### Decision table

| You want… | `path` | `types` | `lib` | `include` | tsconfig `types` | `import type {}` |
|---|---|---|---|---|---|---|
| Add a file to the program | ✅ | ❌ | ❌ | ✅ preferred | ❌ | ⚠️ via import graph |
| Control `outFile` emit order | ✅ only way | ❌ | ❌ | ⚠️ partly | ❌ | ❌ |
| Globals from `@types/node` program-wide | ❌ | ✅ | ❌ | ❌ | ✅ preferred | ✅ in modules |
| Globals inside a **global** `.d.ts` | ⚠️ if local file | ✅ only way | ❌ | ❌ | ⚠️ config-dependent | ❌ breaks globalness |
| Newer JS lib types for one file | ❌ | ❌ | ✅ | ❌ | ✅ via `lib` | ❌ |
| Make a published `.d.ts` self-contained | ⚠️ fragile | ✅ | ✅ | ❌ | ❌ | ✅ if a module |
| Keep jest globals out of `src/` | ❌ | ❌ | ❌ | ✅ | ✅ + 2nd tsconfig | ✅ |
| Works with esbuild / swc / node strip-types | ✅ inert | ✅ inert | ✅ inert | ✅ | ✅ | ✅ |
| **Default recommendation** | avoid | global `.d.ts` only | published `.d.ts` only | **yes** | **yes** | modules |

---

## Connected topics

- **49 — Declaration files** — what `.d.ts` files are, how `declaration: true` emits them, and where the auto-generated `/// <reference types="node" />` in your build output comes from.
- **50 — Ambient declarations** — `declare const`, `declare function`, `declare module`, and the global-script-vs-module rule that makes reference directives necessary in the first place.
- **51 — Module augmentation** — `declare global` and `declare module "express-serve-static-core"`, the module-shaped alternative to global `.d.ts` files, and why it needs an `import`/`export` present.
- **48 — Modules in TypeScript** — `import`/`export`, `moduleResolution`, and the model that replaced `/// <reference path="" />` for expressing file dependencies.
- **03 — tsconfig in depth** — `types`, `typeRoots`, `lib`, `noLib`, `noResolve`, `files`, `include`, `exclude`, and project `references`; the modern home for everything a directive used to do.
- **55 — Typing environment variables** — the `NodeJS.ProcessEnv` augmentation used throughout this doc, and the global `.d.ts` that carries it.
- **53 — Extending Express Request** — the concrete case where you must choose between a global `.d.ts` with a `types` reference and a module with `declare module`.
- **02 — Setting up TypeScript** — installing `@types/*` packages, where they live, and how automatic `@types` inclusion picks them up without you writing anything.
