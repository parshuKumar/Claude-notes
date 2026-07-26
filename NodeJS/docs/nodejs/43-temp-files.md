# 43 — Temporary files and directories

## What is this?

A temporary file is data you write to disk knowing you will delete it soon — a scratch pad, not permanent storage. Node gives you `os.tmpdir()` to find the OS's designated "junk drawer" folder (`/tmp` on Linux/Mac, `C:\Users\<user>\AppData\Local\Temp` on Windows), and `fs.mkdtemp()`/`fs.mkdtempSync()` to safely carve out a unique, collision-free folder inside it. Think of it like a hotel room: you use it for the duration of your stay, you never assume it's still yours the next day, and a good guest always cleans up and checks out before leaving.

## Why does it matter for backend development?

Backend servers constantly need a "workbench" to process data before it's ready for its final home: extracting a ZIP a user uploaded, converting a video, generating a PDF invoice, resizing an image before pushing it to S3, or buffering a large CSV export before streaming it to the client. Writing directly to your app's permanent storage folders is risky — a crash mid-write leaves corrupt partial files mixed in with real data. Temp files isolate "work in progress" from "done," and because many requests can run concurrently, every temp file or folder must have a **unique name** so two requests never collide or overwrite each other's scratch data.

---

## Syntax / API

```js
const os   = require('os');     // gives access to OS-level info, including the temp dir
const fs   = require('fs');     // synchronous + callback filesystem API
const fsp  = require('fs/promises'); // promise-based filesystem API (preferred in async code)
const path = require('path');   // for building safe, cross-platform paths
const crypto = require('crypto'); // for generating unique random names

// ── Finding the OS temp directory ───────────────────────────────────────────
console.log(os.tmpdir());
// → "/tmp" on Linux/Mac, "C:\Users\Parsh\AppData\Local\Temp" on Windows
// NEVER hardcode "/tmp" yourself — os.tmpdir() is the portable, correct way

// ── Creating a unique temp FILE name (you write to it yourself) ────────────
const uniqueName = `upload-${crypto.randomUUID()}.tmp`;   // e.g. "upload-9f2a...-....tmp"
const tempFilePath = path.join(os.tmpdir(), uniqueName);  // full safe path
fs.writeFileSync(tempFilePath, 'some scratch data');       // write it
fs.unlinkSync(tempFilePath);                                // delete when done

// ── Creating a unique temp DIRECTORY (safer, built into Node) ──────────────
// fs.mkdtemp() appends 6 random characters to your prefix and creates the folder
// atomically — no race condition, no collision, guaranteed unique.
const tempDir = fs.mkdtempSync(path.join(os.tmpdir(), 'myapp-'));
console.log(tempDir);
// → "/tmp/myapp-4kX2aB"  (random suffix appended by Node itself)

// ── Async versions (preferred in real server code) ─────────────────────────
async function useTempDir() {
  const dir = await fsp.mkdtemp(path.join(os.tmpdir(), 'export-')); // create unique dir
  await fsp.writeFile(path.join(dir, 'data.csv'), 'id,name\n1,Alice'); // work inside it
  await fsp.rm(dir, { recursive: true, force: true }); // ALWAYS clean up when done
}

// ── The "tmp" npm package (popular third-party alternative) ────────────────
// npm install tmp
const tmp = require('tmp');
tmp.file((err, filePath, fd, cleanupCallback) => {
  // filePath   → ready-to-use unique temp file path
  // cleanupCallback() → call this to delete the file yourself
  if (err) throw err;
  fs.writeSync(fd, 'hello from tmp package'); // write using the open file descriptor
  cleanupCallback(); // delete the temp file
});
```

---

## How it works — line by line

`os.tmpdir()` doesn't create anything — it just reads an environment variable (`TMPDIR`, `TEMP`, or `TMP` depending on OS) and returns the path the operating system has reserved for temporary data. That folder is periodically cleared by the OS itself (on reboot, or by cleanup jobs), so it is explicitly **not** meant for anything you want to keep.

`fs.mkdtemp(prefix)` is the safe way to get a working folder: you give it a prefix like `'myapp-'`, and Node appends 6 random characters, then creates that exact directory in one atomic filesystem operation. Because the random suffix is generated *and* the directory is created in a single step, there is no window where two concurrent requests could both compute the same "next available name" and collide — which is the exact race condition that hand-rolled naming (like `Date.now()` or a counter) is vulnerable to.

Cleanup is entirely your responsibility. Node does not track which temp files or folders you created, does not delete them automatically when your process exits, and the OS may not clear `/tmp` for hours or days. This is why every real-world usage pattern wraps temp file work in `try/finally` — the `finally` block guarantees the delete call runs even if something inside throws an error.

---

## Example 1 — basic

```js
// File: temp-basics.js

const os   = require('os');
const fs   = require('fs');
const path = require('path');

// Step 1 — ask the OS where its temp folder lives (portable across platforms)
const tempRoot = os.tmpdir();
console.log('OS temp root:', tempRoot);

// Step 2 — create a unique temp directory inside it (6 random chars appended)
const workDir = fs.mkdtempSync(path.join(tempRoot, 'demo-'));
console.log('Created temp dir:', workDir);

// Step 3 — write a scratch file inside our new temp directory
const scratchFile = path.join(workDir, 'notes.txt');
fs.writeFileSync(scratchFile, 'This data is temporary and will be deleted.');

// Step 4 — read it back, proving the file really exists and has our content
const content = fs.readFileSync(scratchFile, 'utf8');
console.log('File content:', content);

// Step 5 — clean up: remove the whole temp directory and everything in it
fs.rmSync(workDir, { recursive: true, force: true });
console.log('Cleaned up:', workDir);

// Step 6 — verify it's really gone
console.log('Still exists?', fs.existsSync(workDir)); // → false
```

---

## Example 2 — real world backend use case

```js
// File: src/services/uploadProcessor.js
// Scenario: a user uploads a ZIP file to an Express endpoint. We need a private
// scratch directory to extract it, validate contents, and move the good files
// to permanent storage — without ever touching the app's real storage folders
// with unverified data, and without leaving junk behind if something fails.

const os     = require('os');
const path   = require('path');
const fsp    = require('fs/promises');
const crypto = require('crypto');

const PERMANENT_STORAGE = path.join(__dirname, '..', '..', 'storage', 'uploads');

async function processUpload(requestBody, userId) {
  // Create a unique temp workspace for THIS request only — no collisions
  // even if 100 users upload at the exact same millisecond.
  const workDir = await fsp.mkdtemp(path.join(os.tmpdir(), `upload-${userId}-`));

  try {
    // Simulate writing the raw uploaded buffer into our scratch directory first
    const rawFilePath = path.join(workDir, 'incoming.zip');
    await fsp.writeFile(rawFilePath, requestBody.fileBuffer); // buffer from multer/busboy

    // Simulate "extracting" — in real code you'd use a library like `unzipper` here
    const extractedDir = path.join(workDir, 'extracted');
    await fsp.mkdir(extractedDir);
    await fsp.writeFile(path.join(extractedDir, 'photo.jpg'), 'fake-image-bytes');

    // Validate: reject anything that isn't an allowed file type (security check)
    const files = await fsp.readdir(extractedDir);
    const safeFiles = files.filter((name) => /\.(jpg|png|pdf)$/i.test(name));

    if (safeFiles.length === 0) {
      throw new Error('No valid files found in upload'); // caught below, temp dir still cleaned
    }

    // Only NOW — after validation passes — do we move files into permanent storage
    await fsp.mkdir(PERMANENT_STORAGE, { recursive: true });
    const savedPaths = [];

    for (const fileName of safeFiles) {
      const finalName = `${userId}-${crypto.randomUUID()}${path.extname(fileName)}`;
      const finalPath = path.join(PERMANENT_STORAGE, finalName);
      await fsp.copyFile(path.join(extractedDir, fileName), finalPath); // safe, verified data only
      savedPaths.push(finalPath);
    }

    return { success: true, savedPaths };
  } finally {
    // This runs whether we succeeded, threw, or returned early —
    // the scratch workspace NEVER survives past this function call.
    await fsp.rm(workDir, { recursive: true, force: true });
  }
}

module.exports = { processUpload };
```

---

## Common mistakes

### Mistake 1 — Hardcoding "/tmp" instead of os.tmpdir()

```js
// ❌ WRONG — "/tmp" doesn't exist on Windows, breaks the app on that platform
const fs = require('fs');
fs.writeFileSync('/tmp/export.csv', 'id,name\n1,Alice');

// ✅ CORRECT — os.tmpdir() resolves to the right folder on every OS
const os   = require('os');
const path = require('path');
const filePath = path.join(os.tmpdir(), 'export.csv');
fs.writeFileSync(filePath, 'id,name\n1,Alice');
```

### Mistake 2 — Forgetting to clean up, leaking disk space over time

```js
// ❌ WRONG — if the server handles 10,000 uploads, this leaves 10,000
// orphaned files that are never deleted, slowly filling the disk
async function handleUpload(requestBody) {
  const tempPath = path.join(os.tmpdir(), `upload-${Date.now()}.zip`);
  await fsp.writeFile(tempPath, requestBody.fileBuffer);
  await processFile(tempPath); // if this throws, tempPath is NEVER deleted
}

// ✅ CORRECT — try/finally guarantees cleanup runs even when errors occur
async function handleUpload(requestBody) {
  const tempPath = path.join(os.tmpdir(), `upload-${crypto.randomUUID()}.zip`);
  await fsp.writeFile(tempPath, requestBody.fileBuffer);
  try {
    await processFile(tempPath);
  } finally {
    await fsp.rm(tempPath, { force: true }); // always runs, success or failure
  }
}
```

### Mistake 3 — Predictable temp file names causing collisions or security holes

```js
// ❌ WRONG — a fixed or guessable name means two concurrent requests can
// overwrite each other's data, and an attacker can predict the path
const tempFilePath = path.join(os.tmpdir(), `session-${userId}.tmp`);
// Two requests from the same user at the same time stomp on one another

// ✅ CORRECT — use fs.mkdtemp (atomic + random) or crypto.randomUUID()
// for a name nobody can predict or collide with
const os     = require('os');
const fs     = require('fs/promises');
const crypto = require('crypto');

const workDir = await fs.mkdtemp(path.join(os.tmpdir(), `session-${userId}-`));
// → e.g. "/tmp/session-42-8kLwZq" — random suffix guarantees uniqueness
```

---

## Practice exercises

### Exercise 1 — easy

Write a script that:
1. Logs the OS's temp directory using `os.tmpdir()`.
2. Creates a temp file inside it with a unique name (use `crypto.randomUUID()`).
3. Writes the text `"session token: abc123"` to that file.
4. Reads the file back and logs its contents.
5. Deletes the file and then logs whether it still exists (should be `false`).

```js
// Write your code here
```

---

### Exercise 2 — medium

Write an async function `createExportBundle(rows)` that:
1. Creates a unique temp directory using `fsp.mkdtemp()` with the prefix `'export-'`.
2. Writes each item in the `rows` array (assume an array of objects) into its own file inside that directory, named `row-0.json`, `row-1.json`, etc.
3. Returns an object `{ dir, fileCount }` describing the created directory and how many files were written.
4. Does **not** delete the directory (the caller decides when to clean up).

Then write a second function `cleanupBundle(dir)` that safely removes the entire directory and everything inside it.

Test both by calling `createExportBundle` with 3 sample row objects, logging the result, then calling `cleanupBundle` and confirming the directory no longer exists.

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a `TempFileManager` class that:
1. Keeps an internal array tracking every temp file/directory path it has created.
2. Has a method `createScratchDir(prefix)` that creates a unique temp directory (via `fs.mkdtempSync`), records it internally, and returns the path.
3. Has a method `createScratchFile(content, extension)` that writes `content` to a uniquely named temp file (using `crypto.randomUUID()` + the given extension) directly inside `os.tmpdir()`, records it, and returns the path.
4. Has a method `cleanupAll()` that iterates over everything it has tracked and deletes it (files with `fs.rmSync(path, { force: true })`, directories with `fs.rmSync(path, { recursive: true, force: true })`), then empties its internal tracking array.
5. Registers a listener on `process.on('exit', ...)` in the constructor that automatically calls `cleanupAll()` as a safety net, so nothing is left behind even if the developer forgets to call it manually.

Test it by creating two scratch files and one scratch directory, logging the manager's tracked paths, then calling `cleanupAll()` and verifying every path no longer exists with `fs.existsSync()`.

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
FINDING THE TEMP DIRECTORY
  os.tmpdir()                        → OS-correct temp folder (never hardcode "/tmp")

CREATING UNIQUE TEMP FILES/DIRS
  fs.mkdtempSync(path.join(os.tmpdir(), 'prefix-'))
      → creates + returns a unique dir path, e.g. "/tmp/prefix-4kX2aB"
      → atomic: no race condition between "pick a name" and "create it"
  crypto.randomUUID()                → good for unique file NAMES (dirs use mkdtemp instead)

SYNC vs ASYNC
  fs.mkdtempSync / fs.writeFileSync / fs.rmSync   → simple scripts, startup code
  fsp.mkdtemp / fsp.writeFile / fsp.rm (fs/promises) → real server code, non-blocking

DELETING TEMP DATA
  fsp.rm(filePath, { force: true })                    → delete a single file
  fsp.rm(dirPath, { recursive: true, force: true })    → delete a dir + everything inside
  ALWAYS wrap in try/finally so cleanup runs even on error

THE "tmp" NPM PACKAGE (alternative to hand-rolling)
  npm install tmp
  tmp.file((err, path, fd, cleanupCallback) => { ... cleanupCallback() })
  tmp.dir((err, path, cleanupCallback) => { ... cleanupCallback() })
  → adds auto-cleanup-on-process-exit as a built-in safety net

GOTCHAS
  - The OS may clear /tmp on reboot but NOT automatically during uptime — don't rely on it
  - Never store secrets/tokens in temp files longer than necessary — same disk, same permissions risk
  - Predictable names (Date.now(), fixed strings) → race conditions + security risk
  - Forgetting cleanup on the error path is the #1 cause of "disk full" bugs in production
  - Temp dirs are still subject to the same disk quota as everything else on that volume
```

---

## Connected topics

- **18 — os module** — `os.tmpdir()` itself lives here; this topic covers the rest of `os` (`platform()`, `homedir()`, `freemem()`) for broader system awareness.
- **16 — fs module — promises API** — `fs/promises` is the async API used for all real-world temp file read/write/delete operations shown above.
- **44 — File uploads** — the natural next step: temp directories are the standard scratch space for validating and processing uploaded files before they reach permanent storage.
