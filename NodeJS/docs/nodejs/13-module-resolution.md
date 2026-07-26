# 13 — Module resolution algorithm

## What is this?

The module resolution algorithm is the exact step-by-step process Node.js follows to turn a string like `require('./config')` or `require('express')` into an actual file on disk that it can load. Think of it like a postal system with a strict delivery order: Node first checks if the address is a "local landmark" (a core module like `fs`), then checks if it's a relative address (`./` or `../`), and if neither, it starts walking up the folder tree looking inside every `node_modules` directory until it finds a match or runs out of road.

## Why does it matter for backend development?

Backend projects grow into dozens or hundreds of files with deeply nested folders, and developers constantly hit confusing errors like `Cannot find module './utils'` or accidentally import the wrong package because two `node_modules` folders exist at different levels. Understanding the resolution order tells you exactly why Node picked the file it picked, why a core module always wins over an npm package with the same name, and why `index.js` sometimes loads silently when you only required a folder. This knowledge turns "it just works" or "it mysteriously broke" into something you can predict and debug in seconds.

---

## Syntax / API

```js
// require() itself has no special syntax for resolution — you just call it normally.
// The resolution algorithm runs internally based on WHAT STRING you pass in.

// 1. CORE MODULE — matches Node's built-in module names exactly, resolved FIRST, always
const fs = require('fs');
const path = require('path');

// 2. RELATIVE PATH — starts with './' or '../', resolved relative to the CURRENT FILE
const dbConnection = require('./db/connection');
const authHelper = require('../shared/authHelper');

// 3. ABSOLUTE PATH — starts with '/' (rare in app code), used as-is, no searching
const sharedConfig = require('/etc/myapp/config.json');

// 4. NODE_MODULES PACKAGE — no './', '../', or leading '/' — triggers the node_modules search
const express = require('express');

// You can inspect what Node actually resolved a specifier to, without loading it:
console.log(require.resolve('express'));
// → "/app/node_modules/express/index.js" (the exact file Node would load)

// require.resolve.paths() shows every node_modules folder Node will search, in order
console.log(require.resolve.paths('express'));
// → ['/app/src/node_modules', '/app/node_modules', '/node_modules']
```

---

## How it works — line by line

When you call `require('something')`, Node checks the string against four categories, in this exact priority order:

1. **Core module check** — Node first asks: "Is this one of my own built-in modules (`fs`, `path`, `http`, `crypto`, etc.)?" If yes, it loads the compiled-in native module immediately and stops — nothing on disk is ever searched, even if you have a folder named `fs` in `node_modules`. Core modules always win.
2. **Relative or absolute path check** — If the string starts with `./`, `../`, or `/`, Node treats it as a literal file system path (relative to the current file for `./`/`../`, or absolute for `/`). It does NOT search `node_modules` at all in this case.
3. **File extension resolution** — For path-based requires, Node tries the exact string first, then appends `.js`, then `.json`, then `.node` (native addons), in that order, until one exists.
4. **Directory / index.js fallback** — If the path points to a folder instead of a file, Node looks for a `package.json` with a `"main"` field inside that folder and loads whatever file it points to. If there is no `package.json` or no `"main"` field, Node falls back to loading `index.js` inside that folder.
5. **node_modules walk-up** — If the string is a bare name like `express` (no `./`, `../`, `/`), Node looks for a `node_modules/express` folder starting in the current file's directory, then the parent directory, then the grandparent, and so on, all the way up to the filesystem root. The FIRST match found wins — this is why a package installed in a nested `node_modules` shadows one installed higher up.
6. **Not found** — If none of the above match after reaching the filesystem root, Node throws `Error: Cannot find module 'X'` (code `MODULE_NOT_FOUND`).

```
The node_modules walk-up, visualized for a file at /app/src/routes/user.js
requiring `require('lodash')`:

  Step 1: /app/src/routes/node_modules/lodash   → not found
  Step 2: /app/src/node_modules/lodash          → not found
  Step 3: /app/node_modules/lodash              → FOUND ✓  → load this one
  (Node would keep going up to /node_modules if step 3 had failed)
```

---

## Example 1 — basic

```js
// File: src/index.js

// 1. CORE MODULE — resolved instantly, no disk search, always wins over any package
const path = require('path');

// 2. RELATIVE PATH — Node looks for these files, in this exact order, next to index.js:
//    ./mathUtils.js  → ./mathUtils.json  → ./mathUtils.node  → ./mathUtils/ (folder)
const mathUtils = require('./mathUtils');

// 3. RELATIVE PATH TO A FOLDER — Node looks inside ./services/ for:
//    a) services/package.json with a "main" field, OR
//    b) services/index.js as the fallback
const paymentService = require('./services');

// 4. BARE NAME — no './', '../', or '/' prefix — triggers the node_modules search
//    Node checks: src/node_modules/uuid → node_modules/uuid (project root) → ...
const { v4: generateId } = require('uuid');

console.log('Resolved uuid entry file:', require.resolve('uuid'));
// → e.g. /app/node_modules/uuid/dist/index.js

console.log('math util result:', mathUtils.add(2, 3));  // uses the resolved file above
```

---

## Example 2 — real world backend use case

```js
// File: src/services/index.js
// A common backend pattern: a folder of services with a single entry point.
// Resolution relies on the directory/index.js fallback rule.

const fs = require('fs');
const path = require('path');

// This file IS the "index.js fallback" for `require('./services')` elsewhere in the app.
// Node found it because there is no services/package.json with a "main" field.

const authService = require('./authService');       // ./services/authService.js
const paymentService = require('./paymentService');  // ./services/paymentService.js
const emailService = require('./emailService');      // ./services/emailService.js

module.exports = { authService, paymentService, emailService };

// ─────────────────────────────────────────────────────────────────────────
// File: src/server.js
// Elsewhere in the app, requiring the FOLDER (not a file) triggers the fallback:

const { authService, paymentService } = require('./services');
// Node's resolution here:
//   1. './services' is a relative path → not a core module, not node_modules
//   2. './services' is not a file (no services.js) → try appending extensions → none exist
//   3. './services' IS a directory → look for services/package.json "main" → none
//   4. Fall back to services/index.js → FOUND → load that file

async function authenticateRequest(authToken) {
  // Uses authService, which was resolved through the directory fallback above
  const userId = await authService.verifyToken(authToken);
  return userId;
}

module.exports = { authenticateRequest };

// ─────────────────────────────────────────────────────────────────────────
// Debugging a real MODULE_NOT_FOUND error with require.resolve.paths():
// console.log(require.resolve.paths('paymentService'));
// Shows every node_modules directory Node WOULD search — useful when a package
// installed in a monorepo sub-package isn't being found by a sibling package.
```

---

## Common mistakes

### Mistake 1 — Forgetting the `./` prefix for local files

```js
// ❌ WRONG — no './' prefix means Node treats "authHelper" as a BARE NAME
// It searches node_modules for a package called "authHelper" — never finds your local file
const authHelper = require('authHelper');
// Error: Cannot find module 'authHelper'
// Require stack: /app/src/routes/user.js

// ✅ CORRECT — './' tells Node explicitly: "this is a relative file path, not a package"
const authHelper = require('./authHelper');
// Node resolves: ./authHelper.js relative to the CURRENT file — found immediately
```

### Mistake 2 — Assuming a package version is unique across the project

```js
// ❌ WRONG ASSUMPTION — in large projects/monorepos, MULTIPLE copies of the
// same package can exist at different node_modules levels, causing subtle bugs
// (e.g. two different Mongoose instances, so `instanceof` checks silently fail)
const mongoose = require('mongoose');   // might resolve to a DIFFERENT copy
// than the one a sibling package uses, if node_modules is nested per-package

// ✅ CORRECT — verify exactly which file Node resolved before debugging further
console.log('mongoose loaded from:', require.resolve('mongoose'));
// → compare this path across files/packages to catch duplicate-copy bugs
// Fix at the root cause: dedupe with `npm dedupe`, hoist deps, or use workspaces
```

### Mistake 3 — Requiring a folder and expecting a specific file to load

```js
// ❌ WRONG EXPECTATION — requiring a folder does NOT automatically pick "the main file
// you meant"; it follows the strict package.json "main" → index.js fallback rule only
const dbConnection = require('./db');
// If db/ has NO package.json and NO index.js, this throws Cannot find module './db'
// even though db/connection.js clearly exists inside that folder

// ✅ CORRECT — either be explicit about the file, or add an index.js that re-exports it
const dbConnection = require('./db/connection');   // explicit — always works

// OR create db/index.js so `require('./db')` resolves via the fallback rule:
// File: db/index.js
// module.exports = require('./connection');
```

---

## Practice exercises

### Exercise 1 — easy

Create a small project folder with this structure:
```
project/
  index.js
  greeting.js
```
In `index.js`, `require` the sibling file using the correct relative syntax and call a function it exports. Then use `require.resolve()` to print the exact absolute path Node resolved `'./greeting'` to. Finally, print the result of `require.resolve.paths('lodash')` (even if `lodash` is not installed) to see the list of `node_modules` directories Node would search from that file's location.

```js
// Write your code here
```

---

### Exercise 2 — medium

Build this folder structure:
```
project/
  src/
    services/
      index.js
      userService.js
      orderService.js
    app.js
```
`userService.js` and `orderService.js` should each export one simple function (e.g. `getUserId()`, `getOrderId()`). `services/index.js` should require both and re-export them as a single object (this is the index.js fallback pattern from Example 2). In `app.js`, require the whole `./services` folder (not individual files) and call both functions. Add a comment above the require line explaining, step by step, which resolution rule Node applies to find `services/index.js`.

```js
// Write your code here
```

---

### Exercise 3 — hard

Write a small standalone function `resolveModuleType(specifier)` (no `require()` calls needed inside it — pure string logic) that takes a module specifier string and returns which resolution category it falls into, based on the rules from this topic:
1. Return `'core'` if the specifier matches a hardcoded list of core module names you define (e.g. `['fs', 'path', 'http', 'crypto', 'os', 'events']`)
2. Return `'relative'` if it starts with `./` or `../`
3. Return `'absolute'` if it starts with `/`
4. Return `'package'` for everything else (bare names like `express`, `lodash`, scoped names like `@babel/core`)

Test it against at least 8 different specifier strings covering all four categories, including at least one scoped package name (`@something/package`) and one edge case (an empty string, or a string that is just `.`). Log each input alongside the category your function returned, and manually verify each result is correct.

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
RESOLUTION PRIORITY ORDER (checked top to bottom, first match wins)
  1. Core module         (fs, path, http, crypto, ...) → ALWAYS wins, no disk search
  2. Relative path        './x' or '../x'               → relative to CURRENT FILE
  3. Absolute path         '/x'                          → used literally, no search
  4. Bare package name    'express', '@scope/pkg'       → triggers node_modules walk-up

FILE RESOLUTION ORDER (for a relative/absolute path)
  ./mathUtils           → try exact match
  ./mathUtils.js         → try appending .js
  ./mathUtils.json       → try appending .json
  ./mathUtils.node       → try appending .node (native addon)
  ./mathUtils/           → if it's a directory, apply DIRECTORY rules below

DIRECTORY RESOLUTION (index.js fallback)
  1. Look for package.json inside the folder
  2. If found, load the file named in its "main" field
  3. If no package.json or no "main" field → load index.js
  4. If none of the above exist → Cannot find module error

NODE_MODULES WALK-UP (for bare package names)
  Starting in the current file's directory, check:
    <dir>/node_modules/<pkg>
    <parent>/node_modules/<pkg>
    <grandparent>/node_modules/<pkg>
    ... all the way up to filesystem root
  FIRST match wins — nested node_modules SHADOWS higher-level ones

USEFUL DEBUG TOOLS
  require.resolve('pkg')        → exact absolute file path Node would load
  require.resolve.paths('pkg')  → list of node_modules dirs Node will search
  require.cache                 → inspect what's already loaded (Topic 08)

COMMON ERRORS
  Cannot find module 'X'         → typo, missing './', or package not installed
  Cannot find module './x'       → missing file extension issue or wrong relative path
  MODULE_NOT_FOUND (code)        → the standard error code for all of the above

GOTCHAS
  Core module names are RESERVED — you cannot shadow 'fs' or 'http' with your own file
  Two node_modules at different levels CAN hold different versions of the same package
  ESM (import) uses a DIFFERENT, stricter algorithm — no automatic extension guessing
```

---

## Connected topics

- **08 — CommonJS modules** — `require()` and `module.exports` are the API surface; this topic explains the internal algorithm behind every `require()` call
- **11 — npm in depth** — understanding `node_modules` structure and nested installs explains why the walk-up search can find different package versions at different levels
- **14 — path module** — building the relative/absolute path strings that feed into the resolution algorithm correctly across operating systems
