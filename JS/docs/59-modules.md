# 59 — Modules

## What is this?

A **module** is a file that has its own isolated scope and explicitly declares what it shares with the outside world (`export`) and what it needs from other files (`import`). Think of modules like well-designed electrical appliances: each appliance (module) does one job, has a specific plug-socket interface (exports/imports), and its internal wiring (private variables) is hidden from everything else. You can swap out one appliance without rewiring the whole house.

Before modules, all JavaScript ran in a single shared global scope — variables from one `<script>` tag leaked into every other. Modules solved this: each file gets its own scope, and only what you explicitly export is visible to other files.

---

## Why does it matter?

- **Encapsulation** — private variables stay private; no accidental global pollution
- **Dependency clarity** — `import` statements at the top of a file tell you exactly what it depends on
- **Tree shaking** — bundlers (Webpack, Rollup, esbuild) can remove unused exports — smaller bundles
- **Reusability** — write once, import anywhere
- **Testability** — import individual functions in isolation
- **Node.js backend** — every file is a module; how you structure exports determines your API surface

---

## Syntax

```js
// ── Named exports ──────────────────────────────────────────────
// math.js
export function add(a, b)      { return a + b; }
export function subtract(a, b) { return a - b; }
export const PI = 3.14159;

// ── Named imports ──────────────────────────────────────────────
import { add, PI }             from "./math.js";
import { add as sum, subtract } from "./math.js";    // rename on import
import * as MathUtils          from "./math.js";     // namespace import

// ── Default export (one per file) ─────────────────────────────
// user.js
export default class User {
  constructor(name) { this.name = name; }
}

// ── Default import (any name you want) ────────────────────────
import User       from "./user.js";
import MyUser     from "./user.js";   // same thing, different name

// ── Mixed (default + named from same file) ─────────────────────
import User, { createUser, USER_ROLES } from "./user.js";

// ── Dynamic import (returns Promise) ───────────────────────────
const { add } = await import("./math.js");
```

---

## How it works — the module system

### 1. Module scope

Every ES module file has its own scope. Variables declared at the top level of a module are NOT global:

```js
// counter.js
let count = 0;   // private to this module — NOT window.count

export function increment() { count++; }
export function getCount()  { return count; }
```

```js
// main.js
import { increment, getCount } from "./counter.js";

increment();
increment();
console.log(getCount());   // 2
console.log(count);        // ReferenceError: count is not defined
```

### 2. Live bindings — exports are live, not copies

```js
// timer.js
export let seconds = 0;

setInterval(() => { seconds++; }, 1000);
```

```js
// main.js
import { seconds } from "./timer.js";

setTimeout(() => {
  console.log(seconds);  // 3 — reflects the current value (live binding)
}, 3000);
// If this were a copy: seconds would always be 0
```

This is fundamentally different from CommonJS (`require`) which copies values.

### 3. Modules are singletons — loaded once, shared everywhere

```js
// state.js
export const state = { users: [] };
export function addUser(user) { state.users.push(user); }
```

```js
// service-a.js
import { state, addUser } from "./state.js";
addUser({ id: 1, name: "Alice" });
```

```js
// service-b.js
import { state } from "./state.js";
console.log(state.users);  // [{ id: 1, name: "Alice" }]
// Same module instance — state is shared
```

Even if 10 files import `state.js`, the module is only evaluated ONCE. All imports get a reference to the same module namespace.

### 4. Static analysis — imports are resolved at parse time

ES module `import` declarations are **static**: they must be at the top of the file, not inside conditions or functions. This restriction is intentional — it allows bundlers to do tree shaking.

```js
// VALID — static imports at top level
import { foo } from "./foo.js";

// INVALID — conditional/dynamic static import (SyntaxError)
if (condition) {
  import { foo } from "./foo.js";  // SyntaxError
}

// VALID — dynamic import() for conditional loading
if (condition) {
  const { foo } = await import("./foo.js");  // fine — import() is a function call
}
```

---

## Named exports — in depth

```js
// utils.js

// Inline exports
export const VERSION    = "1.0.0";
export function greet(name) { return `Hello, ${name}`; }
export class Logger { /* ... */ }

// OR — declare first, then export at the bottom (often cleaner)
const VERSION = "1.0.0";
function greet(name) { return `Hello, ${name}`; }
class Logger { /* ... */ }

export { VERSION, greet, Logger };

// Rename on export
export { greet as sayHello, Logger as LogService };
```

```js
// Rename on import
import { sayHello as greet } from "./utils.js";

// Import everything as a namespace object
import * as Utils from "./utils.js";
Utils.greet("Alice");   // "Hello, Alice"
Utils.VERSION;          // "1.0.0"
// Utils itself is not a plain object — it's a Module Namespace Object (sealed, can't add props)
```

---

## Default export — in depth

```js
// A file can have exactly ONE default export
// Common for: main class, main function of a file, React components

// Approach 1 — inline
export default function formatDate(date) {
  return date.toISOString().slice(0, 10);
}

// Approach 2 — declare then export default
function formatDate(date) { ... }
export default formatDate;

// Approach 3 — anonymous
export default (date) => date.toISOString().slice(0, 10);
```

```js
// Importing default — any name works
import formatDate    from "./format.js";
import dateFormatter from "./format.js";  // same export, different alias

// Import default + named in one line
import formatDate, { FORMAT_ISO, FORMAT_US } from "./format.js";
```

---

## Re-exports — building a barrel file

A **barrel file** (`index.js`) collects and re-exports from multiple files:

```js
// users/user-model.js
export class User { /* ... */ }
export class UserRepository { /* ... */ }

// users/user-service.js
export class UserService { /* ... */ }

// users/user-controller.js
export class UserController { /* ... */ }

// users/index.js  ←  barrel file
export { User, UserRepository }  from "./user-model.js";
export { UserService }           from "./user-service.js";
export { UserController }        from "./user-controller.js";

// Rename on re-export
export { UserService as default } from "./user-service.js";  // re-export as default

// Re-export everything
export * from "./user-model.js";
export * from "./user-service.js";

// Namespace re-export
export * as UserModule from "./user-model.js";
```

```js
// Now consumers import from the barrel — clean API surface
import { User, UserService, UserController } from "./users/index.js";
// Instead of:
import { User }           from "./users/user-model.js";
import { UserService }    from "./users/user-service.js";
import { UserController } from "./users/user-controller.js";
```

---

## Dynamic import — `import()`

Dynamic `import()` is a function (actually a keyword that looks like a function call) that returns a Promise resolving to the module namespace object:

```js
// Lazy loading — only load when needed
async function loadChartLibrary() {
  const { Chart } = await import("./chart.js");
  return new Chart();
}

// Conditional import
async function loadPlugin(name) {
  try {
    const plugin = await import(`./plugins/${name}.js`);
    return plugin.default ?? plugin;   // handle default or named export
  } catch {
    throw new Error(`Plugin "${name}" not found`);
  }
}

// Route-based code splitting (React / Vue pattern)
const router = {
  "/":       () => import("./pages/Home.js"),
  "/about":  () => import("./pages/About.js"),
  "/users":  () => import("./pages/Users.js"),
};

async function navigate(path) {
  const loader = router[path];
  if (!loader) return loadPage("404");
  const { default: Page } = await loader();
  render(Page);
}
```

---

## ES Modules vs CommonJS

Node.js supports two module systems. Understanding both is essential for backend work:

```
┌─────────────────┬──────────────────────────┬──────────────────────────┐
│ Feature         │ ES Modules (ESM)          │ CommonJS (CJS)           │
├─────────────────┼──────────────────────────┼──────────────────────────┤
│ Syntax          │ import / export           │ require() / module.exports│
│ Loading         │ Async (spec) / sync (file)│ Synchronous              │
│ Bindings        │ Live (references)         │ Copies at require() time │
│ Execution       │ Static analysis at parse  │ Dynamic at runtime       │
│ Top-level await │ Yes                       │ No                       │
│ __dirname       │ Not available             │ Available                │
│ __filename      │ Not available             │ Available                │
│ Circular deps   │ Handled (live bindings)   │ Partial (gets incomplete │
│                 │                           │ object)                  │
│ Tree shaking    │ Yes (bundlers can)        │ No                       │
│ File extension  │ .js (with type:module)    │ .js (default in Node)    │
│                 │ .mjs                      │ .cjs                     │
│ require()       │ Not available             │ Yes                      │
│ import()        │ Yes                       │ Yes (dynamic only)       │
└─────────────────┴──────────────────────────┴──────────────────────────┘
```

---

## Configuring Node.js for ES Modules

```json
// package.json — option 1: treat all .js files as ESM
{
  "type": "module"
}
```

```
File extensions:
  .mjs  → always ES module (regardless of package.json type)
  .cjs  → always CommonJS (regardless of package.json type)
  .js   → depends on package.json "type" field
            "type": "module"  → .js files are ESM
            (no type / "type": "commonjs") → .js files are CJS
```

```js
// ES module in Node.js — things that change:
// 1. No __dirname or __filename — use import.meta.url instead
import { fileURLToPath } from "url";
import { dirname, join }  from "path";

const __filename = fileURLToPath(import.meta.url);
const __dirname  = dirname(__filename);
const configPath = join(__dirname, "config.json");

// 2. Must include file extensions in local imports
import { add } from "./math.js";    // ✓ — extension required
import { add } from "./math";       // ✗ — works in CJS, NOT in ESM

// 3. JSON imports (Node 22+ with --experimental-json-modules or import assertions)
import data from "./data.json" assert { type: "json" };

// 4. require() is not available in ESM
// If you need it, create it:
import { createRequire } from "module";
const require = createRequire(import.meta.url);
const pkg = require("./package.json");
```

---

## Mixing CJS and ESM in Node.js

```js
// Importing ESM from CJS — NOT directly supported
// CJS files cannot use import/export syntax
// require() cannot load .mjs files

// Solution: use dynamic import() inside an async function in CJS
// file: old-cjs-file.cjs
async function loadEsmModule() {
  const { myFunction } = await import("./new-esm-module.mjs");
  return myFunction();
}

// Importing CJS from ESM — supported directly
// file: new-esm-module.mjs
import oldLibrary from "./old-cjs-file.cjs";           // works
import { namedExport } from "./old-cjs-file.cjs";      // works — CJS default export destructured
```

---

## Real world: Node.js backend file structure with modules

```
src/
├── index.js              ← entry point
├── config/
│   ├── database.js       ← DB connection (singleton)
│   └── index.js          ← barrel: export { db, redis, ... }
├── models/
│   ├── user.model.js
│   ├── post.model.js
│   └── index.js          ← barrel
├── services/
│   ├── user.service.js
│   ├── email.service.js
│   └── index.js          ← barrel
├── controllers/
│   ├── user.controller.js
│   └── index.js          ← barrel
├── middleware/
│   ├── auth.js
│   ├── validate.js
│   └── index.js          ← barrel
├── routes/
│   ├── user.routes.js
│   └── index.js          ← barrel
└── utils/
    ├── errors.js
    ├── logger.js
    └── index.js          ← barrel
```

```js
// config/database.js — singleton pattern with modules
import mongoose from "mongoose";

let connection = null;

export async function connect(uri) {
  if (connection) return connection;   // already connected — return same instance

  connection = await mongoose.connect(uri, {
    maxPoolSize:          10,
    serverSelectionTimeoutMS: 5000,
  });

  console.log("MongoDB connected");
  return connection;
}

export async function disconnect() {
  await mongoose.disconnect();
  connection = null;
}

export { connection };
```

```js
// services/user.service.js
import { UserRepository } from "../models/index.js";
import { hash, compare }  from "../utils/crypto.js";
import { sendEmail }      from "./email.service.js";

export class UserService {
  async createUser(data) {
    const exists = await UserRepository.findByEmail(data.email);
    if (exists) throw new ValidationError("Email already registered");

    const hashedPassword = await hash(data.password);
    const user = await UserRepository.create({ ...data, password: hashedPassword });

    await sendEmail({
      to:      user.email,
      subject: "Welcome!",
      template: "welcome",
      data:     { name: user.name },
    });

    return user;
  }
}
```

---

## Circular dependencies — what they are, what happens

```js
// a.js
import { b } from "./b.js";
export const a = "A - " + b;   // b might be undefined here!

// b.js
import { a } from "./a.js";
export const b = "B - " + a;   // a might be undefined here!
```

What happens:
1. Module `a` starts loading → sees `import { b } from "./b.js"` → starts loading `b`
2. Module `b` starts loading → sees `import { a } from "./a.js"` → module `a` is already being loaded (in-progress)
3. `a`'s current exports are returned — but `a` hasn't finished yet → `a` is `undefined` at this point
4. `b` finishes loading: `b = "B - undefined"`
5. `a` finishes loading: `a = "A - B - undefined"`

**Fix: use functions to defer access until all modules are loaded:**

```js
// a.js
import { getB } from "./b.js";
export function getA() { return "A - " + getB(); }  // getB() called lazily at call time

// b.js
import { getA } from "./a.js";
export function getB() { return "B - " + getA(); }  // getA() called lazily at call time
```

When called at runtime (not at module evaluation time), both functions are already defined.

**General rule: circular dependencies are a code smell — restructure to extract shared code into a third module both can import.**

---

## `import.meta`

Available in ES modules, gives context about the current module:

```js
// import.meta.url — the URL/path of the current module
console.log(import.meta.url);
// file:///home/user/project/src/utils.js  (Node.js)
// http://localhost:3000/src/utils.js      (browser)

// Common Node.js pattern to get __dirname equivalent:
import { fileURLToPath } from "url";
import { dirname }       from "path";

const __dirname = dirname(fileURLToPath(import.meta.url));

// import.meta.env — in Vite/bundler contexts
console.log(import.meta.env.VITE_API_URL);
console.log(import.meta.env.MODE);         // "development" | "production"

// import.meta.resolve — resolve a module specifier to a URL
const mathURL = import.meta.resolve("./math.js");
```

---

## Tricky things

### 1. Named imports are not destructuring — they are live bindings

```js
// This LOOKS like destructuring but is not the same:
import { count } from "./counter.js";

// Destructuring would give you a copy.
// import gives you a live binding — count updates when the module updates it.

// You CANNOT reassign a named import:
count = 5;  // TypeError: Assignment to constant variable
// The module that exports count is the only one that can change it.
```

### 2. Default export has no name unless you give it one

```js
// This creates an anonymous function as the default export:
export default function() { return 42; }

// The function's .name property will be "" or "default"
// This makes stack traces harder to read

// Better — always name your default exports:
export default function getAnswer() { return 42; }
// Or:
function getAnswer() { return 42; }
export default getAnswer;
```

### 3. Module evaluation order

```js
// Modules are evaluated in dependency order — deepest dependency first.
// If A imports B and C, B imports D:
// Evaluation order: D → B → C → A

// This matters for side effects:
// side-effects.js
console.log("side-effects.js evaluated");
export const x = 1;

// main.js
import { x } from "./side-effects.js";
console.log("main.js running");

// Output:
// side-effects.js evaluated
// main.js running
```

### 4. Top-level await blocks the module graph

```js
// database.js
export const db = await connectToDatabase();  // blocks until connected
// Any module that imports from database.js will wait for this to resolve
// All modules dependent on database.js are delayed

// This is powerful but use carefully:
// ✓ Good: one-time async setup at module load
// ✗ Bad: long operations that delay your entire app startup
```

### 5. `import * as X` gives a Module Namespace Object, not a plain object

```js
import * as Utils from "./utils.js";

// You cannot:
Utils.newProp = "value";   // TypeError: Cannot add property — namespace is sealed
JSON.stringify(Utils);     // "{}" — own properties are not enumerable in the usual sense

// You CAN:
for (const key of Object.keys(Utils)) { /* iterate exports */ }
Utils.existingExport;      // access exports by name
```

### 6. Extensions are REQUIRED in Node.js ESM (unlike CJS)

```js
// CJS — extensions optional
const { add } = require("./math");     // works

// ESM — extensions required in Node.js
import { add } from "./math";          // ERR_MODULE_NOT_FOUND
import { add } from "./math.js";       // ✓ correct
```

---

## Common mistakes

### Mistake 1 — Mixing default and named export confusion

```js
// WRONG — exporting a named export when you mean default
// math.js
export { add as default };   // confusing — use export default directly

// RIGHT
export default add;

// WRONG — importing default as if it were named
import { add } from "./math.js";   // if add is the DEFAULT export, this fails

// RIGHT
import add from "./math.js";       // default import
import { add } from "./math.js";   // named import (only if add is a named export)
```

### Mistake 2 — Forgetting file extensions in ESM

```js
// WRONG (in Node.js ESM)
import { helper } from "./helper";       // ERR_MODULE_NOT_FOUND
import { helper } from "./helper/";      // ERR_MODULE_NOT_FOUND

// RIGHT
import { helper } from "./helper.js";    // explicit extension
import { helper } from "./helper/index.js"; // explicit index file
```

### Mistake 3 — Re-exporting default incorrectly

```js
// utils/formatter.js has a default export (formatDate)

// WRONG — this doesn't re-export the default
export * from "./formatter.js";   // export * skips the default export!

// RIGHT — explicitly re-export the default
export { default as formatDate } from "./formatter.js";
// or
export { default } from "./formatter.js";
```

### Mistake 4 — Using `__dirname` in ESM without a shim

```js
// WRONG — not available in ESM
const configPath = path.join(__dirname, "config.json");  // ReferenceError: __dirname is not defined

// RIGHT — derive it from import.meta.url
import { fileURLToPath } from "url";
import { dirname, join }  from "path";
const __dirname  = dirname(fileURLToPath(import.meta.url));
const configPath = join(__dirname, "config.json");
```

---

## Practice exercises

### Exercise 1 — easy

Create two files:
- `math.js` — exports named functions: `add`, `subtract`, `multiply`, `divide`, and a constant `TAX_RATE = 0.2`
- `main.js` — imports all of them, calls each function, also imports all as a namespace `* as Math` and calls `Math.add(1, 2)`

Then create `index.js` that re-exports everything from `math.js` via a barrel, plus re-exports `divide` as both `divide` and `divideNumbers`.

```js
// Write your code here
```

### Exercise 2 — medium

Build a small module system for a Node.js API:

- `config/env.js` — reads `process.env` and exports: `PORT`, `DB_URI`, `JWT_SECRET`, `NODE_ENV`. Throw an error at module load time if any required env var is missing (this is a module-level side effect).

- `utils/logger.js` — exports a `logger` object with `info`, `warn`, `error` methods. Format: `[timestamp] [LEVEL] message`. In production (`NODE_ENV === "production"`), skip `info` logs.

- `utils/index.js` — barrel re-exporting both.

- `app.js` — uses dynamic `import()` to conditionally load `./dev-tools.js` only when `NODE_ENV === "development"`. Import `logger` and `PORT` from the barrels and log startup info.

```js
// Write your code here
```

### Exercise 3 — hard

Implement a **plugin system** using dynamic imports:

```
plugins/
├── json-formatter.js   — exports default: { name, format(data) }
├── csv-formatter.js    — exports default: { name, format(data) }
└── xml-formatter.js    — exports default: { name, format(data) }

PluginLoader class:
  - #plugins: Map<name, plugin>
  - async load(pluginPath)   — dynamically imports, validates has name+format, registers
  - async loadAll(paths[])   — loads all in parallel using Promise.all
  - get(name)                — returns plugin or throws if not found
  - list()                   — returns array of plugin names
  - async unload(name)       — removes from registry (can't actually unload ESM, but remove from Map)

DataFormatter class:
  - constructor(loader)
  - async format(data, formatName)  — get plugin from loader, call plugin.format(data), return result
  - async formatAll(data)           — run ALL loaded plugins on the data and return { name: result } map
```

All plugin files should be ES modules with default exports. The loader should validate the plugin interface before registering.

```js
// Write your code here
```

---

## Quick reference (cheat sheet)

| Concept | Syntax |
|---|---|
| Named export (inline) | `export const x = 1;` / `export function f() {}` |
| Named export (batch) | `export { a, b, c };` |
| Named export (renamed) | `export { a as myA };` |
| Default export | `export default value;` |
| Named import | `import { a, b } from "./mod.js";` |
| Named import (renamed) | `import { a as myA } from "./mod.js";` |
| Default import | `import anything from "./mod.js";` |
| Default + named | `import Default, { a, b } from "./mod.js";` |
| Namespace import | `import * as Mod from "./mod.js";` |
| Dynamic import | `const mod = await import("./mod.js");` |
| Re-export named | `export { a } from "./mod.js";` |
| Re-export renamed | `export { a as b } from "./mod.js";` |
| Re-export all named | `export * from "./mod.js";` (skips default) |
| Re-export default | `export { default } from "./mod.js";` |
| Re-export default as named | `export { default as name } from "./mod.js";` |
| Live binding | Named imports reflect changes — not copies |
| Module singleton | Module runs once — all imports share same instance |
| `import.meta.url` | URL of current module file |
| `__dirname` in ESM | `dirname(fileURLToPath(import.meta.url))` |
| ESM in Node.js | `"type": "module"` in package.json, or `.mjs` |
| CJS in ESM | `import cjsLib from "./lib.cjs"` — works |
| ESM in CJS | `const m = await import("./lib.mjs")` inside async fn |
| Static import location | Top of file only — not in blocks or conditionals |
| Top-level await | Only in ES modules (`type: module`) |

---

## Connected topics

- **52 — Event loop** — understanding how module evaluation fits into the JS execution model
- **56 — async/await** — top-level await is an ES module feature
- **62 — Generators** — generator functions are commonly exported from modules
- **75 — Module pattern** — the old pre-ESM pattern modules replaced (IIFE-based encapsulation)
