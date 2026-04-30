# 03 — tsconfig.json in depth

## What is this?

`tsconfig.json` is the configuration file that controls how the TypeScript compiler (`tsc`) behaves. It sits at the root of your project and answers questions like: *Where are my source files? Where should compiled output go? What version of JavaScript should I target? How strict should the type checking be?* Every TypeScript project has one.

## Why does it matter?

Without `tsconfig.json`, the compiler uses defaults that are often wrong for Node.js — wrong module format, wrong output location, no strict checking. A misconfigured `tsconfig.json` can silently let bugs through (by disabling strict checks), break your imports at runtime (wrong `module` setting), or cause your compiled output to land in the wrong place. Understanding every option means you can set up any project correctly from scratch.

## The JavaScript way vs the TypeScript way

```js
// JavaScript — zero configuration needed
// node index.js just works. No compiler, no config file.
// Freedom, but also: no guardrails.

// ❌ JS pain: you discover misconfigured paths, wrong assumptions,
//    and broken imports at RUNTIME, not before deployment.
```

```ts
// TypeScript — tsconfig.json is your contract with the compiler
// tsc reads it and enforces your rules across the entire project.
// One config file protects every single .ts file in your codebase.

// ✅ TS win: misconfigurations are caught at COMPILE time.
//    If your paths, modules, or types are wrong — tsc tells you before you ship.
```

---

## The full tsconfig.json — every important option

```json
{
  "compilerOptions": {
    // ─── OUTPUT CONTROL ──────────────────────────────────────────
    "target": "ES2020",
    "module": "commonjs",
    "outDir": "./dist",
    "rootDir": "./src",

    // ─── TYPE CHECKING STRICTNESS ────────────────────────────────
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitReturns": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,

    // ─── MODULE RESOLUTION ───────────────────────────────────────
    "esModuleInterop": true,
    "moduleResolution": "node",
    "resolveJsonModule": true,
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    },

    // ─── DECLARATION FILES ───────────────────────────────────────
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,

    // ─── MISC ────────────────────────────────────────────────────
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "**/*.spec.ts"]
}
```

---

## How it works — option by option

### `target` — What JavaScript version to compile to

```json
"target": "ES2020"
```

TypeScript compiles your code **down** to whatever JS version you specify. This matters for features like `async/await`, optional chaining (`?.`), and `BigInt`.

```ts
// You write modern TypeScript:
const userId = user?.profile?.id ?? 0;   // optional chaining + nullish coalescing

// With target: "ES2020" → kept as-is (ES2020 supports these)
const userId = user?.profile?.id ?? 0;

// With target: "ES5" → compiled down to older equivalent:
var userId = (_a = (_b = user) === null || _b === void 0 ? void 0 : _b.profile) === null
  || _a === void 0 ? void 0 : _a.id;
var userId_1 = userId !== null && userId !== void 0 ? userId : 0;
```

**For Node.js:** Use `ES2020` or higher. Node.js 18+ supports ES2022 natively, so you don't need to compile down far.

| Target | Use when |
|--------|----------|
| `ES5` | Supporting very old browsers (almost never needed) |
| `ES2017` | Node.js 8-10 support needed |
| `ES2020` | Node.js 14+ (good default) |
| `ES2022` | Node.js 18+ (modern default) |
| `ESNext` | Latest JS features, bleeding edge |

---

### `module` — What module system to output

```json
"module": "commonjs"
```

This controls whether your compiled JS uses `require()` or `import/export`.

```ts
// You write in TypeScript (always use import/export):
import express from 'express';
export function getUser() { ... }
```

```js
// With "module": "commonjs" → outputs:
const express = require('express');
function getUser() { ... }
exports.getUser = getUser;

// With "module": "ESNext" → outputs:
import express from 'express';
export function getUser() { ... }
```

**The rule:**
- Node.js + CommonJS → `"module": "commonjs"` (most common backend setup)
- Node.js + ESM (`"type": "module"` in package.json) → `"module": "NodeNext"`
- Frontend bundler (Webpack/Vite) → `"module": "ESNext"`

---

### `outDir` — Where compiled files go

```json
"outDir": "./dist"
```

```
src/
  index.ts          →   dist/index.js
  routes/
    users.ts        →   dist/routes/users.js
  controllers/
    userController.ts →  dist/controllers/userController.js
```

TypeScript **mirrors your folder structure** from `rootDir` into `outDir`. The relative paths are preserved.

---

### `rootDir` — Where your source files are

```json
"rootDir": "./src"
```

Tells the compiler: "all `.ts` files I should compile live inside `src/`." This prevents compiled output from mirroring an unexpected path.

```
// Without rootDir:
// If you have files in src/ and test/ both,
// tsc might output:
dist/
  src/
    index.js     ← extra nesting you didn't want

// With "rootDir": "./src":
dist/
  index.js       ← clean, flat output
```

---

### `strict` — The most important option

```json
"strict": true
```

This is a **master switch** that enables a bundle of strict checks all at once. It enables:

| Sub-option | What it enforces |
|-----------|-----------------|
| `strictNullChecks` | `null` and `undefined` are not valid where a real value is expected |
| `noImplicitAny` | You cannot have a parameter/variable with an inferred `any` type |
| `strictFunctionTypes` | Function parameter types must match exactly |
| `strictBindCallApply` | `.bind()`, `.call()`, `.apply()` are type-checked correctly |
| `strictPropertyInitialization` | Class properties must be initialized in the constructor |
| `noImplicitThis` | `this` must have a known type |

**Always set `"strict": true`.** The whole point of TypeScript falls apart without it.

```ts
// With strict: false (bad)
function getUserName(user) {   // user is implicitly 'any' — no error
  return user.name;            // TypeScript doesn't check anything
}

// With strict: true (correct)
function getUserName(user) {   // ❌ Error: Parameter 'user' implicitly has an 'any' type
  return user.name;
}
// Forces you to type it properly:
function getUserName(user: { name: string }): string {
  return user.name;  // ✅ Now TypeScript knows this is safe
}
```

---

### `strictNullChecks` — Null safety

Included in `strict: true`, but worth understanding on its own.

```ts
// strictNullChecks: false (dangerous)
let userId: number = null;   // allowed — silently broken
userId.toString();           // runtime crash

// strictNullChecks: true (safe)
let userId: number = null;   // ❌ Error: Type 'null' is not assignable to type 'number'

// If null is a valid value, you must say so explicitly:
let userId: number | null = null;  // ✅ Explicitly allows null
if (userId !== null) {
  userId.toString();  // ✅ TypeScript knows userId is number here, not null
}
```

---

### `noImplicitAny` — No sneaky any types

```ts
// noImplicitAny: true — TypeScript refuses to silently give things 'any' type
function processRequest(requestBody) {   // ❌ Error: 'requestBody' implicitly has type 'any'
  return requestBody.userId;
}

// Fix: explicitly type it
function processRequest(requestBody: { userId: number; email: string }): number {
  return requestBody.userId;  // ✅
}
```

---

### `noImplicitReturns` — Every code path must return

```ts
// noImplicitReturns: true
function getStatusMessage(statusCode: number): string {
  if (statusCode === 200) {
    return "OK";
  }
  // ❌ Error: Not all code paths return a value.
  // What if statusCode is 404? The function returns undefined silently in JS.
}

// Fix: handle all paths
function getStatusMessage(statusCode: number): string {
  if (statusCode === 200) return "OK";
  if (statusCode === 404) return "Not Found";
  return "Unknown Status";  // ✅ All paths covered
}
```

---

### `noUnusedLocals` and `noUnusedParameters`

```ts
// noUnusedLocals: true
function buildQuery(userId: number): string {
  const debugInfo = "some debug";   // ❌ Error: 'debugInfo' is declared but never read
  return `SELECT * FROM users WHERE id = ${userId}`;
}

// noUnusedParameters: true
function formatResponse(data: object, unusedFlag: boolean): string {  // ❌ 'unusedFlag' unused
  return JSON.stringify(data);
}

// Fix: remove unused variables, or prefix with _ to intentionally mark as unused
function formatResponse(data: object, _unusedFlag: boolean): string {  // ✅ _ prefix is convention
  return JSON.stringify(data);
}
```

---

### `esModuleInterop` — Sane import syntax

```ts
// Without esModuleInterop: true
import * as express from 'express';  // required awkward syntax
const app = express();               // works but ugly

// With esModuleInterop: true
import express from 'express';       // ✅ clean default import
const app = express();
```

This makes TypeScript handle the difference between `module.exports = ...` (CommonJS default export) and ES module default exports. Always enable this for Node.js projects.

---

### `moduleResolution` — How TypeScript finds imported files

```json
"moduleResolution": "node"
```

Tells TypeScript to resolve imports the same way Node.js does — check `node_modules`, check `index.ts`, etc.

| Value | Use when |
|-------|----------|
| `"node"` | Classic Node.js + CommonJS (most projects) |
| `"NodeNext"` | Node.js with native ESM (`"type": "module"`) |
| `"bundler"` | Using Vite, Webpack, or esbuild |

---

### `resolveJsonModule` — Import JSON files directly

```json
"resolveJsonModule": true
```

```ts
// With resolveJsonModule: true
import packageJson from '../package.json';  // ✅
console.log(packageJson.version);           // TypeScript knows the shape of the JSON
```

---

### `paths` — Path aliases (import shortcuts)

```json
"baseUrl": ".",
"paths": {
  "@/*": ["src/*"]
}
```

```ts
// Without path aliases (brittle relative paths):
import { UserService } from '../../../services/UserService';
import { authMiddleware } from '../../middleware/auth';

// With @/ path alias:
import { UserService } from '@/services/UserService';   // ✅ clean and stable
import { authMiddleware } from '@/middleware/auth';      // ✅ works from any file depth
```

**Important:** `paths` only works for TypeScript's type checking. At runtime, Node.js still doesn't understand `@/`. You need `tsconfig-paths` or `tsc-alias` to make it work at runtime (covered in topic 74).

---

### `declaration` and `declarationMap` — Generating type definition files

```json
"declaration": true,
"declarationMap": true
```

```
// With declaration: true, tsc also outputs:
dist/
  index.js          ← compiled JS
  index.d.ts        ← type declaration file (for library consumers)
  index.d.ts.map    ← maps .d.ts back to .ts source (with declarationMap: true)
```

Use these when **building a library** or **sharing types** between packages in a monorepo. For a regular API server, you don't need them.

---

### `sourceMap` — Debugging support

```json
"sourceMap": true
```

Generates `.js.map` files that allow debuggers (VS Code, Chrome DevTools) to map runtime errors in the compiled JS back to the original `.ts` line numbers.

```
dist/
  index.js       ← compiled JS
  index.js.map   ← sourcemap: maps line 42 in index.js → line 38 in src/index.ts
```

Without source maps, a stack trace points to line numbers in `dist/index.js` — useless for debugging. With source maps, it points to your original `.ts` file.

---

### `skipLibCheck` — Skip checking node_modules types

```json
"skipLibCheck": true
```

Skips type checking of all `.d.ts` files in `node_modules`. Without this, TypeScript checks every type declaration in every installed package — slow and sometimes causes false errors from poorly typed third-party packages. Always enable this.

---

### `forceConsistentCasingInFileNames`

```json
"forceConsistentCasingInFileNames": true
```

Prevents bugs caused by case-insensitive file systems (macOS/Windows) vs case-sensitive ones (Linux/servers).

```ts
// On macOS this works (filesystem is case-insensitive):
import { getUser } from './UserService';  // actually loads ./userservice.ts

// On Linux (production server) this crashes — file casing must match exactly
// forceConsistentCasingInFileNames: true catches this mismatch at compile time
```

---

### `include` and `exclude` — What to compile

```json
"include": ["src/**/*"],
"exclude": ["node_modules", "dist", "**/*.spec.ts"]
```

- `include` — glob patterns for files to include. `src/**/*` means all files recursively inside `src/`.
- `exclude` — glob patterns to skip. Always exclude `node_modules` and `dist`. Exclude test files if you compile them separately.

```json
// Common production config (no test files in build):
"include": ["src/**/*"],
"exclude": ["node_modules", "dist", "**/*.spec.ts", "**/*.test.ts"]

// Including tests in type-check but not in output:
"include": ["src/**/*", "tests/**/*"],
"exclude": ["node_modules", "dist"]
// Then set "outDir" to only include src/ with rootDir: "src"
```

---

## Example 1 — basic: seeing what strict catches

```ts
// src/example.ts — try this with strict: true vs strict: false

// This function has THREE bugs that strict: true catches:
function findUser(userId) {                  // ❌ noImplicitAny: userId is 'any'
  const user = { id: userId, name: "Parsh" };
  const unused = "debug";                    // ❌ noUnusedLocals
  if (userId > 0) {
    return user;
    // ❌ noImplicitReturns: what if userId <= 0? No return value.
  }
}

// Fixed with strict: true forcing better code:
function findUser(userId: number): { id: number; name: string } | null {
  if (userId <= 0) return null;             // ✅ all paths return
  return { id: userId, name: "Parsh" };     // ✅ typed return
}
```

---

## Example 2 — real world backend use case

**Full production-ready `tsconfig.json` for a Node.js/Express API:**

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "commonjs",
    "lib": ["ES2022"],
    "outDir": "./dist",
    "rootDir": "./src",

    "strict": true,
    "noImplicitReturns": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noUncheckedIndexedAccess": true,

    "esModuleInterop": true,
    "moduleResolution": "node",
    "resolveJsonModule": true,

    "baseUrl": ".",
    "paths": {
      "@/routes/*": ["src/routes/*"],
      "@/controllers/*": ["src/controllers/*"],
      "@/models/*": ["src/models/*"],
      "@/middleware/*": ["src/middleware/*"],
      "@/utils/*": ["src/utils/*"],
      "@/types/*": ["src/types/*"]
    },

    "sourceMap": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "**/*.spec.ts", "**/*.test.ts"]
}
```

**How this protects your backend:**

```ts
// src/controllers/userController.ts
import { Request, Response } from 'express';
import { UserService } from '@/services/UserService';  // clean alias

// noImplicitReturns catches missing else branches in route handlers
async function getUserById(req: Request, res: Response): Promise<void> {
  const userId = parseInt(req.params.id, 10);

  if (isNaN(userId)) {
    res.status(400).json({ error: "Invalid user ID" });
    return;                     // ✅ must explicitly return after sending response
  }

  const user = await UserService.findById(userId);

  if (!user) {
    res.status(404).json({ error: "User not found" });
    return;                     // ✅ all code paths handled
  }

  res.json({ user });           // ✅ final code path
}
```

---

## Common mistakes

### Mistake 1 — Turning off strict to silence errors

```json
// ❌ WRONG — disabling strict to "fix" errors faster
{
  "compilerOptions": {
    "strict": false,
    "noImplicitAny": false
  }
}
// You've now turned TypeScript into a fancy JS syntax highlighter.
// All the protection is gone.

// ✅ RIGHT — fix the actual type errors
// If strict causes errors, those errors are REAL bugs TypeScript found.
// Fix the types, don't silence the checker.
```

### Mistake 2 — Wrong `module` setting for Node.js

```json
// ❌ WRONG for a Node.js project with CommonJS
{
  "compilerOptions": {
    "module": "ESNext"   // outputs: import/export syntax
  }
}
// Node.js then throws: SyntaxError: Cannot use import statement in a module
// (unless you also set "type": "module" in package.json)

// ✅ RIGHT for Node.js + CommonJS (most common)
{
  "compilerOptions": {
    "module": "commonjs"  // outputs: require() / module.exports
  }
}
```

### Mistake 3 — `rootDir` and `outDir` mismatch

```json
// ❌ WRONG — rootDir doesn't match where files actually are
{
  "compilerOptions": {
    "rootDir": "./src",
    "outDir": "./dist"
  },
  "include": ["src/**/*", "tests/**/*"]   // tests/ is outside rootDir!
}
// Error: 'rootDir' is expected to contain all source files.
// File 'tests/user.spec.ts' is not under 'rootDir' './src'.

// ✅ RIGHT — rootDir must contain ALL compiled files
{
  "compilerOptions": {
    "rootDir": ".",          // both src/ and tests/ are under root
    "outDir": "./dist"
  },
  "include": ["src/**/*", "tests/**/*"]
}
// Or: exclude tests entirely from compilation
```

---

## Practice exercises

### Exercise 1 — easy

Create a `tsconfig.json` from scratch (without using `tsc --init`) that:
- Targets `ES2020`
- Uses `commonjs` modules
- Compiles from `src/` to `dist/`
- Has `strict` enabled
- Has `esModuleInterop` enabled
- Excludes `node_modules` and `dist`

Then write a `src/index.ts` that would fail compilation without `strict: true` (e.g., a function with an untyped parameter) and show how you fix it to make `tsc` happy.

```json
// Write your tsconfig.json here
```

```ts
// Write your broken src/index.ts and then the fixed version
```

### Exercise 2 — medium

You have this `tsconfig.json`:

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "noImplicitReturns": true,
    "noUnusedLocals": true,
    "esModuleInterop": true
  },
  "include": ["src/**/*"]
}
```

Write a `src/auth.ts` file with two functions:

1. `validateAuthToken(token: string): boolean` — returns `true` if the token starts with `"Bearer "`, `false` otherwise. Make sure **all code paths return a value** (otherwise `noImplicitReturns` will catch it).
2. `extractUserId(authToken: string): number` — extracts the numeric user ID from a token formatted as `"Bearer userId-123"`. Returns the number `123`. Make sure you have **no unused variables** (otherwise `noUnusedLocals` will catch it).

```ts
// Write your src/auth.ts here
```

### Exercise 3 — hard

Set up path aliases. Add this to your `tsconfig.json`:

```json
"baseUrl": ".",
"paths": {
  "@/utils/*": ["src/utils/*"],
  "@/types/*": ["src/types/*"]
}
```

Then:
1. Create `src/types/index.ts` that exports a type `ApiResponse<T>` with properties: `success: boolean`, `data: T | null`, `error: string | null`.
2. Create `src/utils/responseBuilder.ts` that imports `ApiResponse` using the `@/types/index` alias and exports two functions:
   - `buildSuccess<T>(data: T): ApiResponse<T>` — returns `{ success: true, data, error: null }`
   - `buildError(errorMessage: string): ApiResponse<null>` — returns `{ success: false, data: null, error: errorMessage }`
3. Create `src/index.ts` that imports both functions using `@/utils/responseBuilder` and calls them with realistic data (a user object for success, an error string for failure), logging the results.

Run `npx tsc` and confirm it compiles without errors (path aliases are type-checked by tsc — the runtime resolution issue is a separate topic).

```ts
// Write your three files here
```

---

## Quick reference cheat sheet

| Option | Recommended value | What it does |
|--------|------------------|-------------|
| `target` | `"ES2020"` or `"ES2022"` | JS version to compile down to |
| `module` | `"commonjs"` (Node.js) | Module system in output |
| `outDir` | `"./dist"` | Where compiled JS goes |
| `rootDir` | `"./src"` | Where your TS source lives |
| `strict` | `true` — **always** | Enables all strict type checks |
| `esModuleInterop` | `true` — **always** | Clean `import x from 'x'` syntax |
| `moduleResolution` | `"node"` | How imports are resolved |
| `resolveJsonModule` | `true` | Import `.json` files |
| `sourceMap` | `true` (dev), optional (prod) | Debug stack traces in `.ts` |
| `skipLibCheck` | `true` | Skip checking node_modules types |
| `noImplicitReturns` | `true` | All code paths must return |
| `noUnusedLocals` | `true` | No dead variables |
| `declaration` | `true` (libraries only) | Generate `.d.ts` files |

## Connected topics

- **02 — Setting up TypeScript** — The project setup that uses this `tsconfig.json`.
- **04 — TypeScript with Node.js project setup** — How tsconfig fits into a full backend project.
- **74 — Path aliases in TypeScript** — Making `@/` imports work at runtime with `tsc-alias` or `tsconfig-paths`.
