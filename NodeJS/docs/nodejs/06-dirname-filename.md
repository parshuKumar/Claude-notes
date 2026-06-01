# 06 — __dirname and __filename

## What is this?

`__dirname` and `__filename` are two variables that Node.js automatically injects into every CommonJS module. `__filename` is the **absolute path to the current file** being executed. `__dirname` is the **absolute path to the folder that file lives in**. They never change regardless of where you run the `node` command from. Think of them as a GPS coordinate baked into each file — no matter how your app is deployed or from what directory it is launched, these always point to the file's own location on disk.

## Why does it matter for backend development?

Backend apps read config files, templates, static assets, and migration scripts using file paths. If you build those paths with string concatenation or relative paths like `'./config.json'`, they break the moment you run the server from a different directory. `__dirname` solves this permanently — it is always the directory of the source file, so paths built with it work from anywhere. Every Node.js backend developer uses `__dirname` daily with `path.join()` to build safe, portable paths.

---

## Syntax / API

```js
// Both are available in every CommonJS (.js) file automatically — no require needed

console.log(__filename);
// → "C:\Users\Parsh\Desktop\NOTES\NodeJS\src\server.js"  (absolute path to THIS file)

console.log(__dirname);
// → "C:\Users\Parsh\Desktop\NOTES\NodeJS\src"  (folder this file lives in)

// ── Building paths safely ────────────────────────────────────────────────────
const path = require('path');

// Correct way — always use path.join() with __dirname, never string concatenation
const configPath  = path.join(__dirname, 'config.json');
const uploadsDir  = path.join(__dirname, '..', 'uploads');     // go up one level
const templateDir = path.join(__dirname, 'templates', 'email'); // go deeper

// ── ESM equivalent (import/export syntax) ───────────────────────────────────
// __dirname and __filename do NOT exist in ES Modules (.mjs or "type":"module")
// Use import.meta.url instead:
import { fileURLToPath } from 'url';
import { dirname } from 'path';

const __filename = fileURLToPath(import.meta.url);   // reconstruct __filename
const __dirname  = dirname(__filename);               // reconstruct __dirname
```

---

## How it works — line by line

When Node.js loads a CommonJS module, it wraps your file's code inside a function before executing it:

```js
// Node internally wraps every CommonJS file in this function wrapper:
(function(exports, require, module, __filename, __dirname) {
  // YOUR FILE'S CODE IS PLACED HERE
});
```

So `__filename` and `__dirname` are not globals — they are **function parameters** injected by Node's module loader for each file individually. This is why they always point to the current file, not to wherever `node` was run from.

This also means:
- They do **not** exist in the Node REPL (no file to point to)
- They do **not** exist in ES Modules (different module system, no wrapper function)
- Each file gets its own values — `__dirname` in `src/routes/user.js` differs from `__dirname` in `src/server.js`

### The CWD trap

```
Project structure:
  /app/
    src/
      server.js      ← file reads './config.json'
    config.json

Run from /app/src/  →  node server.js        → finds config.json ✓
Run from /app/     →  node src/server.js    → CANNOT find './config.json' ✗
Run from /         →  node app/src/server.js → CANNOT find './config.json' ✗

With __dirname:
Run from anywhere  →  path.join(__dirname, '../config.json') → ALWAYS finds it ✓
```

---

## Example 1 — basic

```js
// File: src/utils/logger.js

const path = require('path');

// __filename → absolute path to THIS file (logger.js)
console.log('This file  :', __filename);

// __dirname → folder this file lives in (utils/)
console.log('This folder:', __dirname);

// Build path to a sibling config file — works no matter where node is run from
const configPath = path.join(__dirname, '..', 'config', 'app.json');
console.log('Config path:', configPath);
// → /app/src/config/app.json — always correct, never depends on cwd

// Get just the filename without directory
console.log('File name  :', path.basename(__filename));     // 'logger.js'

// Get just the file extension
console.log('Extension  :', path.extname(__filename));      // '.js'

// Get the name without extension
const moduleName = path.basename(__filename, path.extname(__filename));
console.log('Module name:', moduleName);   // 'logger'
```

---

## Example 2 — real world backend use case

```js
// File: src/config/index.js
// A config loader that finds files relative to itself — never relative to cwd.
// This is the pattern used in Express apps, CLI tools, and any multi-file backend.

const path = require('path');
const fs   = require('fs');

// Resolve the project root — go up from src/config/ to the project root
const PROJECT_ROOT = path.join(__dirname, '..', '..');

// All paths built from __dirname — safe regardless of where the app is launched
const paths = {
  root:       PROJECT_ROOT,
  src:        path.join(PROJECT_ROOT, 'src'),
  config:     path.join(PROJECT_ROOT, 'config'),
  uploads:    path.join(PROJECT_ROOT, 'uploads'),
  logs:       path.join(PROJECT_ROOT, 'logs'),
  publicDir:  path.join(PROJECT_ROOT, 'public'),
};

// Load the correct config file based on NODE_ENV
const nodeEnv    = process.env.NODE_ENV || 'development';
const configFile = path.join(paths.config, `${nodeEnv}.json`);  // e.g. development.json

function loadConfig() {
  if (!fs.existsSync(configFile)) {
    // process.stderr and exit — covered in Topic 05
    process.stderr.write(`[config] Missing config file: ${configFile}\n`);
    process.exit(1);
  }

  const raw    = fs.readFileSync(configFile, 'utf8');   // safe: startup only, sync is OK
  const config = JSON.parse(raw);

  return { ...config, paths };   // merge file config with resolved paths
}

module.exports = { loadConfig, paths, PROJECT_ROOT };

// Usage in server.js:
// const { loadConfig } = require('./config');
// const config = loadConfig();
// console.log(config.paths.uploads);  // → /app/uploads — always correct
```

---

## Common mistakes

### Mistake 1 — Using relative paths instead of __dirname

```js
// ❌ WRONG — relative path is resolved from process.cwd(), not the file's location
const fs = require('fs');

// Works when you run: node src/server.js (from project root)
// BREAKS when you run: node server.js (from inside src/)
const config = fs.readFileSync('./config/app.json', 'utf8');

// ✅ CORRECT — __dirname is always the file's own directory
const path = require('path');
const config = fs.readFileSync(path.join(__dirname, 'config', 'app.json'), 'utf8');
// Works from any directory, always
```

### Mistake 2 — String-concatenating paths instead of using path.join

```js
// ❌ WRONG — string concatenation breaks on Windows (backslash vs forward slash)
// and fails when __dirname already has a trailing slash
const uploadsDir = __dirname + '/uploads/' + userId + '/avatar.png';
// On Windows: "C:\app\src/uploads/user_1/avatar.png" — mixed separators

// ✅ CORRECT — path.join handles separators for the current OS
const path = require('path');
const avatarPath = path.join(__dirname, 'uploads', userId, 'avatar.png');
// On Windows: "C:\app\src\uploads\user_1\avatar.png" ✓
// On Linux:   "/app/src/uploads/user_1/avatar.png"   ✓
```

### Mistake 3 — Expecting __dirname to exist in ES Modules

```js
// ❌ WRONG — __dirname does not exist in .mjs files or files with "type":"module"
// ReferenceError: __dirname is not defined
import fs from 'fs';
const configPath = path.join(__dirname, 'config.json');   // crashes

// ✅ CORRECT — reconstruct __dirname from import.meta.url in ESM
import { fileURLToPath } from 'url';
import { dirname, join }  from 'path';
import fs from 'fs';

const __filename = fileURLToPath(import.meta.url);
const __dirname  = dirname(__filename);

const configPath = join(__dirname, 'config.json');   // works correctly
```

---

## Practice exercises

### Exercise 1 — easy

Create a file at any path inside the project. In that file, use `__filename` and `__dirname` to print:
1. The absolute path of the file itself
2. The name of the folder it lives in (just the folder name, not the full path — hint: `path.basename`)
3. The file name without extension
4. A constructed path to an imaginary `logs/app.log` file two levels up from the current file

```js
// Write your code here
```

---

### Exercise 2 — medium

Write a function `resolveProjectPath(...segments)` that:
1. Assumes it lives inside a `src/utils/` folder
2. Navigates up to the **project root** (two levels up from `__dirname`)
3. Joins the project root with any additional path segments passed in
4. Returns the resolved absolute path

Then call it with:
- `resolveProjectPath('config', 'database.json')`
- `resolveProjectPath('uploads', 'avatars')`
- `resolveProjectPath()` — should return the project root itself

Log all three results and verify they look correct.

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a `FileStore` class that:
1. Takes a `storeName` string in its constructor (e.g. `'sessions'`, `'uploads'`)
2. Resolves its storage directory as `__dirname/../../data/{storeName}` — so it always works regardless of cwd
3. Creates the directory if it does not exist (use `fs.mkdirSync` with `{ recursive: true }`)
4. Has a `write(fileName, content)` method that writes a file into that directory
5. Has a `read(fileName)` method that reads and returns the file content
6. Has a `list()` method that returns an array of all file names in the store directory
7. Has a `getStorePath()` method that returns the absolute path to the store directory

Test it:
```js
const sessionStore = new FileStore('sessions');
sessionStore.write('user_42.json', JSON.stringify({ userId: 'user_42', token: 'abc123' }));
console.log(sessionStore.read('user_42.json'));
console.log(sessionStore.list());
console.log(sessionStore.getStorePath());
```

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
WHAT THEY ARE
  __filename  → absolute path to the CURRENT FILE being executed
  __dirname   → absolute path to the FOLDER the current file lives in

HOW THEY WORK
  Node wraps every CJS file in:
  (function(exports, require, module, __filename, __dirname) { ... })
  They are function parameters, not true globals

AVAILABILITY
  CommonJS (.js with no "type":"module")  → available automatically ✓
  ES Modules (.mjs or "type":"module")    → NOT available ✗ — use import.meta.url
  Node REPL                               → NOT available ✗ — no file context

ESM EQUIVALENT
  import { fileURLToPath } from 'url';
  import { dirname } from 'path';
  const __filename = fileURLToPath(import.meta.url);
  const __dirname  = dirname(__filename);

ALWAYS USE WITH path.join()
  path.join(__dirname, 'config.json')          → sibling file
  path.join(__dirname, '..', 'config.json')    → parent directory
  path.join(__dirname, '..', '..', 'root.js')  → two levels up

NEVER DO
  __dirname + '/file.json'          → breaks on Windows
  './relative-path.json'            → breaks if cwd changes
  __dirname + '/' + subDir + '/'    → use path.join instead

COMMON PATTERNS
  Project root from src/utils/foo.js:
    const ROOT = path.join(__dirname, '..', '..');

  Load config relative to file:
    const cfg = require(path.join(__dirname, 'config.json'));

  Serve static files in Express:
    express.static(path.join(__dirname, 'public'))
```

---

## Connected topics

- **14 — path module** — `path.join()`, `path.resolve()`, `path.dirname()` — the tools you always combine with `__dirname`
- **15 — fs module (callbacks)** — `fs.readFile`, `fs.writeFile` — where `__dirname`-built paths are actually used
- **09 — ES Modules in Node** — why `__dirname` doesn't exist in ESM and the `import.meta.url` pattern in full detail
