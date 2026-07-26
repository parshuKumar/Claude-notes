# 16 — fs module (promises API)

## What is this?

`fs/promises` is the promise-based version of Node's file system module — the same file operations as the classic `fs` module (read, write, delete, rename, list directories) but returning Promises instead of taking error-first callbacks. This means you can `await` every file operation and write it top-to-bottom like normal synchronous-looking code, while the actual work still happens off the main thread. Think of it as ordering food through an app instead of yelling your order across a kitchen and waiting for someone to shout back — you get a receipt (a Promise) immediately, and you check on it (`await`) whenever you're ready, without blocking anyone else in line.

## Why does it matter for backend development?

Nearly every backend service touches the disk — reading config, writing logs, saving uploaded files, generating reports, caching data to local files. The old callback-style `fs` API works fine for one file, but it gets ugly fast the moment you need to read three files in sequence, or read ten files at once and wait for all of them. `fs/promises` combined with `async/await` and `Promise.all` lets you write that logic cleanly, handle errors with plain `try/catch`, and run independent file operations in parallel for real speed gains. This is the API modern Node.js backends actually use — callbacks (Topic 15) are mostly legacy code you'll only read, not write.

---

## Syntax / API

```js
// Import the promise-based fs API — note the '/promises' suffix
const fs = require('fs/promises');

// Every method below returns a Promise — use await inside an async function
async function demo() {
  // Read an entire file's contents (utf8 = get a string back, not a Buffer)
  const data = await fs.readFile('./config.json', 'utf8');

  // Write a file — creates it if missing, overwrites if it exists
  await fs.writeFile('./output.txt', 'Hello from Node\n', 'utf8');

  // Append to an existing file without overwriting it
  await fs.appendFile('./app.log', 'New log line\n', 'utf8');

  // Delete a file
  await fs.unlink('./temp.txt');

  // Create a directory (recursive: true also creates missing parent folders)
  await fs.mkdir('./uploads/avatars', { recursive: true });

  // List all entries (files + folders) inside a directory
  const entries = await fs.readdir('./uploads');

  // Get metadata about a file/folder — size, type, timestamps
  const stats = await fs.stat('./config.json');

  // Rename or move a file (same function does both)
  await fs.rename('./old-name.txt', './new-name.txt');

  // Check existence without throwing (fs/promises has no existsSync equivalent —
  // the idiomatic way is to try the operation and catch ENOENT)
  try {
    await fs.access('./maybe-missing.json');
    console.log('File exists');
  } catch {
    console.log('File does not exist');
  }
}
```

---

## How it works — line by line

`require('fs/promises')` loads a version of the `fs` module where every function already returns a Promise — Node built this for you, so you don't need `util.promisify` (Topic 24) to wrap anything yourself.

When you call `await fs.readFile(...)`, three things happen: (1) Node hands the actual disk read to libuv's thread pool so your main thread is never blocked, (2) your `async` function pauses at that line without freezing the whole app — other requests keep being handled, (3) when the disk read finishes, the Promise resolves with the file's contents and your function resumes exactly where it left off.

If anything goes wrong — file missing, no permission, disk error — the Promise **rejects** instead of resolving. Because you used `await`, that rejection turns into a thrown JavaScript error, which a normal `try/catch` block around your `await` calls can catch. This is the biggest win over callbacks: no more checking `if (err)` after every single call — one `try/catch` (or one `.catch()`) covers a whole sequence of file operations.

```
fs.readFile('a.txt')     fs.readFile('b.txt')     fs.readFile('c.txt')
        │                        │                        │
        ▼                        ▼                        ▼
  handed to libuv          handed to libuv          handed to libuv
  thread pool               thread pool               thread pool
        │                        │                        │
        └────────── all run at the same time ─────────────┘
                          (if using Promise.all)
```

`Promise.all([...])` is what turns three separate `await`s into one parallel batch — instead of waiting for file A, then B, then C one after another, you fire off all three reads at once and wait for the slowest one to finish. For independent file operations, this can cut total wait time from "sum of all reads" down to "the single longest read."

---

## Example 1 — basic

```js
// File: src/scripts/read-config.js

const fs = require('fs/promises');   // promise-based fs API
const path = require('path');

async function readConfig() {
  // Build a safe path relative to this file (Topic 06)
  const configPath = path.join(__dirname, '..', 'config', 'app.json');

  try {
    // await pauses this function until the disk read finishes
    const rawData = await fs.readFile(configPath, 'utf8');

    // Parse the JSON string into a real JavaScript object
    const config = JSON.parse(rawData);

    console.log('Config loaded:', config);
    return config;

  } catch (error) {
    // Runs if the file is missing, unreadable, or contains invalid JSON
    console.error('Failed to load config:', error.message);
    throw error;   // re-throw so the caller knows something failed
  }
}

// Top-level call — Node supports top-level await in ESM, but in CommonJS
// we call the async function and handle its promise like this:
readConfig().catch(() => process.exit(1));
```

---

## Example 2 — real world backend use case

```js
// File: src/services/reportService.js
// A backend job that reads several data files in parallel, merges them,
// and writes a combined report — the kind of task a Node backend runs on a schedule.

const fs = require('fs/promises');
const path = require('path');

const DATA_DIR   = path.join(__dirname, '..', 'data');
const REPORT_DIR = path.join(__dirname, '..', 'reports');

async function generateDailyReport(reportDate) {
  // Build the three input file paths this report depends on
  const usersPath   = path.join(DATA_DIR, 'users.json');
  const ordersPath  = path.join(DATA_DIR, 'orders.json');
  const paymentsPath = path.join(DATA_DIR, 'payments.json');

  let usersRaw, ordersRaw, paymentsRaw;

  try {
    // Promise.all runs all three reads in PARALLEL, not one after another —
    // total wait time ≈ the slowest single read, not the sum of all three
    [usersRaw, ordersRaw, paymentsRaw] = await Promise.all([
      fs.readFile(usersPath, 'utf8'),
      fs.readFile(ordersPath, 'utf8'),
      fs.readFile(paymentsPath, 'utf8'),
    ]);
  } catch (error) {
    // If ANY of the three reads fails, Promise.all rejects immediately
    console.error(`[report] Could not read source data: ${error.message}`);
    throw new Error('Report generation failed: missing source data');
  }

  // Parse all three JSON payloads now that we know all reads succeeded
  const users    = JSON.parse(usersRaw);
  const orders   = JSON.parse(ordersRaw);
  const payments = JSON.parse(paymentsRaw);

  // Build the report object from the merged data
  const report = {
    date: reportDate,
    totalUsers: users.length,
    totalOrders: orders.length,
    totalRevenue: payments.reduce((sum, payment) => sum + payment.amount, 0),
  };

  // Make sure the output directory exists before writing (recursive = safe to call always)
  await fs.mkdir(REPORT_DIR, { recursive: true });

  // Build the output file path and write the finished report as JSON
  const reportPath = path.join(REPORT_DIR, `report-${reportDate}.json`);
  await fs.writeFile(reportPath, JSON.stringify(report, null, 2), 'utf8');

  console.log(`[report] Saved report to ${reportPath}`);
  return report;
}

module.exports = { generateDailyReport };

// Usage:
// const { generateDailyReport } = require('./services/reportService');
// generateDailyReport('2026-07-27').catch(console.error);
```

---

## Common mistakes

### Mistake 1 — Awaiting file reads sequentially when they don't depend on each other

```js
// ❌ WRONG — each await blocks the next one from even starting
// If each read takes 200ms, this takes ~600ms total
const usersRaw    = await fs.readFile(usersPath, 'utf8');
const ordersRaw   = await fs.readFile(ordersPath, 'utf8');
const paymentsRaw = await fs.readFile(paymentsPath, 'utf8');

// ✅ CORRECT — independent reads run in parallel with Promise.all
// All three start at the same time — total time ≈ 200ms, not 600ms
const [usersRaw, ordersRaw, paymentsRaw] = await Promise.all([
  fs.readFile(usersPath, 'utf8'),
  fs.readFile(ordersPath, 'utf8'),
  fs.readFile(paymentsPath, 'utf8'),
]);
```

### Mistake 2 — Forgetting try/catch around await, letting rejections crash the process

```js
// ❌ WRONG — if the file is missing, this throws an unhandled rejection
// and can crash the whole Node process (Topic 07)
async function loadSessionFile(sessionId) {
  const data = await fs.readFile(`./sessions/${sessionId}.json`, 'utf8');
  return JSON.parse(data);
}

// ✅ CORRECT — wrap the await in try/catch and return a safe fallback
async function loadSessionFile(sessionId) {
  try {
    const data = await fs.readFile(`./sessions/${sessionId}.json`, 'utf8');
    return JSON.parse(data);
  } catch (error) {
    if (error.code === 'ENOENT') {
      return null;   // session file just doesn't exist yet — not a crash
    }
    throw error;      // any other error (permissions, corrupt JSON) still surfaces
  }
}
```

### Mistake 3 — Mixing the callback fs module with fs/promises by accident

```js
// ❌ WRONG — require('fs') gives you the CALLBACK version, not promises
// This does NOT return a Promise, so `await` does nothing useful here
const fs = require('fs');
const data = await fs.readFile('./config.json', 'utf8');
// TypeError: fs.readFile(...) is not a thenable — await just passes it through

// ✅ CORRECT — require the '/promises' variant explicitly
const fs = require('fs/promises');
const data = await fs.readFile('./config.json', 'utf8');   // works as expected

// Alternative: keep both available under different names in the same file
// const fsCallback = require('fs');
// const fsPromises = require('fs/promises');
```

---

## Practice exercises

### Exercise 1 — easy

Write an async function `writeAndReadBack(filePath, content)` that:
1. Writes `content` to `filePath` using `fs.writeFile`
2. Immediately reads the same file back using `fs.readFile`
3. Logs the content that was read back
4. Wraps everything in a `try/catch` and logs a clear error message if either step fails

Call it with a path inside a `scratch/` folder next to your script and any test string.

```js
// Write your code here
```

---

### Exercise 2 — medium

Write an async function `getDirectorySummary(dirPath)` that:
1. Reads all entries in `dirPath` using `fs.readdir`
2. For every entry, calls `fs.stat` to get its size and whether it's a file or directory
3. Runs all the `fs.stat` calls **in parallel** using `Promise.all` (not one at a time in a loop with `await` inside)
4. Returns an array of objects like `{ name, sizeInBytes, isDirectory }`
5. Handles the case where `dirPath` does not exist by catching the error and returning an empty array

Test it against any folder in your project and log the result.

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a `JsonFileStore` class backed by `fs/promises` that:
1. Takes a `storeDir` path in its constructor and ensures it exists (create it with `fs.mkdir({ recursive: true })` lazily, the first time it's needed — not in the constructor, since constructors can't be `async`)
2. Has an async method `save(key, data)` that writes `data` (an object) as pretty-printed JSON to `{storeDir}/{key}.json`
3. Has an async method `load(key)` that reads and parses that file, returning `null` (not throwing) if the file doesn't exist
4. Has an async method `remove(key)` that deletes the file, silently succeeding if it was already gone
5. Has an async method `loadAll()` that reads every `.json` file in `storeDir` **in parallel** and returns an array of `{ key, data }` objects
6. Every method must correctly distinguish "file not found" (`error.code === 'ENOENT'`) from real errors, which should still be thrown

Test it:
```js
const store = new JsonFileStore('./data/sessions');
await store.save('user_42', { userId: 'user_42', authToken: 'abc123' });
console.log(await store.load('user_42'));
console.log(await store.load('does_not_exist'));   // should log null, not throw
console.log(await store.loadAll());
await store.remove('user_42');
```

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
IMPORT
  const fs = require('fs/promises');   // promise-based API — always with async/await

CORE METHODS (all return a Promise)
  fs.readFile(path, 'utf8')            → string contents (omit encoding → Buffer)
  fs.writeFile(path, data, 'utf8')     → creates or OVERWRITES the file
  fs.appendFile(path, data, 'utf8')    → adds to the end without overwriting
  fs.unlink(path)                      → deletes a file
  fs.mkdir(path, { recursive: true })  → creates dir + missing parents, no error if exists
  fs.rmdir / fs.rm(path, {recursive})  → removes a directory (fs.rm is the modern one)
  fs.readdir(path)                     → array of entry names in a directory
  fs.stat(path)                        → { size, isFile(), isDirectory(), mtime, ... }
  fs.rename(oldPath, newPath)          → renames/moves a file
  fs.access(path)                      → resolves if reachable, rejects if not (no existsSync equivalent)

ERROR HANDLING
  Wrap awaited calls in try/catch — rejections become thrown errors
  error.code === 'ENOENT'   → file/dir does not exist
  error.code === 'EACCES'   → permission denied
  error.code === 'EEXIST'   → already exists (e.g. mkdir without recursive)

PARALLEL VS SEQUENTIAL
  Sequential (slow, only when each step depends on the previous result):
    const a = await fs.readFile(pathA);
    const b = await fs.readFile(pathB);

  Parallel (fast, when reads are independent):
    const [a, b] = await Promise.all([fs.readFile(pathA), fs.readFile(pathB)]);

GOTCHAS
  require('fs') vs require('fs/promises')  → callbacks vs promises, don't mix them up
  writeFile OVERWRITES by default          → use appendFile to add without wiping
  Promise.all rejects on the FIRST failure → use Promise.allSettled if partial success is OK
  No fs.existsSync equivalent in promises  → use fs.access() + try/catch instead
```

---

## Connected topics

- **15 — fs module (callbacks)** — the older error-first callback API that `fs/promises` wraps and replaces for everyday backend code
- **46 — Promises in Node context** — `Promise.all`, `Promise.allSettled`, `Promise.race` explained in depth beyond just file reads
- **17 — fs module — streams and large files** — when a file is too large to safely `readFile` into memory all at once, streams are the next step
