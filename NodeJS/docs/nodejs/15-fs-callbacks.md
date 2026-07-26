# 15 — fs module (callbacks)

## What is this?

The `fs` (File System) module is Node's built-in toolkit for reading, writing, deleting, and inspecting files and folders on disk. The callback API is the **original**, non-blocking way to use it — you call a function, hand it a callback, and Node tells your code when the operation finishes instead of freezing everything until it's done. Think of it like dropping a letter in a mailbox: you don't stand there until it's delivered, you walk away and get notified (via the callback) once it arrives.

---

## Why does it matter for backend development?

Almost every backend service touches the filesystem at some point: reading config files, writing logs, saving uploaded images, generating reports, or storing session data on disk before a database exists. Because Node runs on a single thread, using the **synchronous** fs methods (`readFileSync`, `writeFileSync`) blocks that thread — one slow disk read freezes every other request your server is handling. The callback-based fs API keeps the event loop free, so your server can serve thousands of other requests while a file operation happens in the background via libuv's thread pool. Every backend developer needs to recognize this API even though modern code usually prefers the promise-based version (Topic 16) — you will still encounter callback-style fs in legacy code, third-party libraries, and Node's own internals.

---

## Syntax / API

```js
// Import the callback-based fs module — no special import needed, it's built in
const fs = require('fs');

// ── Reading a file ──────────────────────────────────────────────────────────
// fs.readFile(path, encoding, callback) — callback receives (error, data)
fs.readFile('./data/users.json', 'utf8', (err, data) => {
  if (err) return console.error('Read failed:', err.message); // always check err first
  console.log(data); // data is a string because we passed 'utf8' encoding
});

// ── Writing a file (overwrites if it exists, creates if it doesn't) ────────
// fs.writeFile(path, content, callback) — callback receives just (error)
fs.writeFile('./data/output.txt', 'Hello backend!', (err) => {
  if (err) return console.error('Write failed:', err.message);
  console.log('File written successfully');
});

// ── Appending to a file (adds to the end, never overwrites) ────────────────
fs.appendFile('./logs/app.log', 'New log line\n', (err) => {
  if (err) return console.error('Append failed:', err.message);
});

// ── Deleting a file ─────────────────────────────────────────────────────────
fs.unlink('./data/tempFile.txt', (err) => {
  if (err) return console.error('Delete failed:', err.message);
  console.log('File deleted');
});

// ── Creating a directory ────────────────────────────────────────────────────
// { recursive: true } creates nested folders and does not error if it exists
fs.mkdir('./uploads/avatars', { recursive: true }, (err) => {
  if (err) return console.error('Mkdir failed:', err.message);
});

// ── Removing a directory ────────────────────────────────────────────────────
// fs.rmdir only removes EMPTY folders; use fs.rm for folders with contents
fs.rmdir('./uploads/tempDir', (err) => {
  if (err) return console.error('Rmdir failed:', err.message);
});

// ── Reading the contents of a directory ─────────────────────────────────────
fs.readdir('./uploads', (err, files) => {
  if (err) return console.error('Readdir failed:', err.message);
  console.log(files); // array of file/folder names, e.g. ['avatar1.png', 'avatar2.png']
});

// ── Getting file/folder metadata ────────────────────────────────────────────
fs.stat('./data/users.json', (err, stats) => {
  if (err) return console.error('Stat failed:', err.message);
  console.log(stats.size);         // file size in bytes
  console.log(stats.isFile());     // true if it's a file
  console.log(stats.isDirectory()); // true if it's a folder
});

// ── Checking existence — the ONLY common sync method still used freely ─────
// existsSync is fine because it's a cheap, one-shot check with no callback API
if (fs.existsSync('./data/users.json')) {
  console.log('File exists');
}
```

---

## How it works — line by line

Every callback-based fs function follows the same shape: you give it a path, sometimes some options, and always a function to run when the work is done. Node hands your operation off to libuv's background thread pool (so the main thread stays free), and when the disk finishes reading or writing, Node puts your callback at the front of the line to run.

The callback always follows the **error-first** convention: the first argument is either `null` (success) or an `Error` object (failure), and any actual result comes after it. This is why every example above starts with `if (err) return ...` — checking the error first is not optional style, it's the contract every Node core function follows (covered in depth in Topic 07 and Topic 45).

`fs.readFile` and `fs.writeFile` work with whole files at once — they load the entire file into memory before your callback runs, which is fine for config files and small JSON but dangerous for multi-gigabyte files (that's what streams, Topic 17, are for). `fs.stat` doesn't touch file contents at all — it just asks the operating system for metadata: size, type, timestamps, permissions. `fs.existsSync` is the one exception to "avoid sync fs calls" — because checking existence is instant and there's no useful "wait for this" moment, Node's own docs consider it acceptable to call synchronously, though even here `fs.access()` (callback-based) is the technically more idiomatic choice in async code.

---

## Example 1 — basic

```js
// File: src/examples/fs-basics.js
const fs = require('fs');
const path = require('path');

// Build a safe, absolute path using __dirname (Topic 06) + path.join (Topic 14)
const filePath = path.join(__dirname, 'notes.txt');

// Step 1 — write a file (creates it since it doesn't exist yet)
fs.writeFile(filePath, 'First line of notes\n', (err) => {
  if (err) return console.error('Write error:', err.message);
  console.log('notes.txt created');

  // Step 2 — append more content to the same file (nested to guarantee order)
  fs.appendFile(filePath, 'Second line, appended later\n', (err) => {
    if (err) return console.error('Append error:', err.message);
    console.log('Line appended');

    // Step 3 — read the file back to confirm both lines exist
    fs.readFile(filePath, 'utf8', (err, data) => {
      if (err) return console.error('Read error:', err.message);
      console.log('File contents:\n' + data);

      // Step 4 — check metadata: how big is the file now?
      fs.stat(filePath, (err, stats) => {
        if (err) return console.error('Stat error:', err.message);
        console.log('File size (bytes):', stats.size);

        // Step 5 — clean up: delete the file we created
        fs.unlink(filePath, (err) => {
          if (err) return console.error('Delete error:', err.message);
          console.log('notes.txt deleted');
        });
      });
    });
  });
});
```

---

## Example 2 — real world backend use case

```js
// File: src/services/uploadLogger.js
// A backend pattern: every time a user uploads a file, log it to a per-day
// audit file and make sure the logs directory exists first.

const fs = require('fs');
const path = require('path');

const LOGS_DIR = path.join(__dirname, '..', '..', 'logs'); // project_root/logs

function logUpload(userId, fileName, fileSizeBytes) {
  // Step 1 — make sure the logs directory exists (recursive: true = no error if present)
  fs.mkdir(LOGS_DIR, { recursive: true }, (err) => {
    if (err) return console.error('[uploadLogger] Failed to create logs dir:', err.message);

    // Step 2 — build today's log file path, e.g. logs/2026-07-27.log
    const today = new Date().toISOString().slice(0, 10);   // 'YYYY-MM-DD'
    const logFilePath = path.join(LOGS_DIR, `${today}.log`);

    // Step 3 — build the log line (JSON per line is a common log format)
    const logEntry = JSON.stringify({
      timestamp: new Date().toISOString(),
      userId,
      fileName,
      fileSizeBytes,
    }) + '\n';

    // Step 4 — append the entry; never overwrite, since other uploads log too
    fs.appendFile(logFilePath, logEntry, (err) => {
      if (err) return console.error('[uploadLogger] Failed to write log:', err.message);
      console.log(`[uploadLogger] Logged upload for user ${userId}`);
    });
  });
}

// Usage when a user finishes an upload request:
// logUpload('user_42', 'resume.pdf', 204800);

function listTodaysUploads(callback) {
  const today = new Date().toISOString().slice(0, 10);
  const logFilePath = path.join(LOGS_DIR, `${today}.log`);

  // existsSync is fine here — it's a quick guard before an async read
  if (!fs.existsSync(logFilePath)) {
    return callback(null, []); // no uploads logged today yet
  }

  fs.readFile(logFilePath, 'utf8', (err, data) => {
    if (err) return callback(err);

    // Each line is a separate JSON object — split, filter blanks, parse each
    const entries = data
      .split('\n')
      .filter((line) => line.trim().length > 0)
      .map((line) => JSON.parse(line));

    callback(null, entries);
  });
}

module.exports = { logUpload, listTodaysUploads };
```

---

## Common mistakes

### Mistake 1 — Forgetting to check the error argument

```js
// ❌ WRONG — ignoring err means silent failures go unnoticed
fs.readFile('./config/database.json', 'utf8', (err, data) => {
  const config = JSON.parse(data); // crashes with a confusing error if data is undefined
  console.log(config);
});

// ✅ CORRECT — always check err first, before touching the result
fs.readFile('./config/database.json', 'utf8', (err, data) => {
  if (err) {
    console.error('Could not load config:', err.message);
    return; // stop here — don't touch `data`, it will be undefined
  }
  const config = JSON.parse(data);
  console.log(config);
});
```

### Mistake 2 — Using fs.rmdir on a non-empty directory

```js
// ❌ WRONG — fs.rmdir only deletes EMPTY directories
// Throws: Error: ENOTEMPTY: directory not empty
fs.rmdir('./uploads/sessionCache', (err) => {
  if (err) console.error(err.message); // will log the ENOTEMPTY error
});

// ✅ CORRECT — use fs.rm with { recursive: true } to delete a folder and its contents
fs.rm('./uploads/sessionCache', { recursive: true, force: true }, (err) => {
  if (err) return console.error('Cleanup failed:', err.message);
  console.log('sessionCache removed, including all files inside it');
});
```

### Mistake 3 — Nesting callbacks too deeply ("callback hell")

```js
// ❌ WRONG — each operation nests inside the last, forming an unreadable pyramid
fs.readFile('./data/userA.json', 'utf8', (err, dataA) => {
  if (err) return console.error(err);
  fs.readFile('./data/userB.json', 'utf8', (err, dataB) => {
    if (err) return console.error(err);
    fs.writeFile('./data/merged.json', dataA + dataB, (err) => {
      if (err) return console.error(err);
      console.log('Merged!');
      // ...imagine 3 more levels here — this is "the pyramid of doom"
    });
  });
});

// ✅ CORRECT — use the promise-based fs API (Topic 16) with async/await instead
const fsPromises = require('fs/promises');

async function mergeUserFiles() {
  try {
    const dataA = await fsPromises.readFile('./data/userA.json', 'utf8');
    const dataB = await fsPromises.readFile('./data/userB.json', 'utf8');
    await fsPromises.writeFile('./data/merged.json', dataA + dataB);
    console.log('Merged!');
  } catch (err) {
    console.error('Merge failed:', err.message);
  }
}
// Callback fs is still worth knowing — it's what fs/promises wraps internally,
// and you'll meet it in older codebases and some libraries.
```

---

## Practice exercises

### Exercise 1 — easy

Write a script that:
1. Creates a folder called `temp-data` in the same directory as your script (use `fs.mkdir` with `{ recursive: true }`)
2. Writes a file `hello.txt` inside it containing your name
3. Reads the file back and logs its contents to the console
4. Uses `fs.existsSync` to confirm the file exists before reading it, and logs `"File found"` or `"File missing"` accordingly

```js
// Write your code here
```

---

### Exercise 2 — medium

Build a small "session store" module using only callback-based fs methods:
1. A function `saveSession(sessionId, sessionData, callback)` that writes `sessionData` (an object) as JSON into a file named `${sessionId}.json` inside a `sessions/` folder (create the folder if missing)
2. A function `loadSession(sessionId, callback)` that reads that file back, parses the JSON, and passes it to the callback as `(err, data)` — if the file doesn't exist, pass back a custom error like `new Error('Session not found')` instead of letting the raw fs error leak out
3. A function `deleteSession(sessionId, callback)` that deletes the session file
4. Test all three by saving a session for `userId: 'user_101'`, loading it back and logging it, then deleting it and confirming with `existsSync` that it's gone

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a `LogRotator` class that manages daily log files without ever loading an entire log history into memory at once:
1. Constructor takes a `logDir` path and creates the directory (recursive) if it doesn't exist
2. A method `write(message)` that appends a timestamped line to today's log file (`YYYY-MM-DD.log`)
3. A method `listLogFiles(callback)` that uses `fs.readdir` to return all `.log` file names in `logDir`, sorted oldest to newest
4. A method `deleteOldLogs(maxFilesToKeep, callback)` that: reads all log files, uses `fs.stat` on each to get its creation time, sorts them oldest-first, and deletes (via `fs.unlink`) enough of the oldest files so only `maxFilesToKeep` remain — calling `callback(err, deletedFileNames)` when done
5. Handle the callback nesting carefully — you may use a simple counter to know when all async `fs.stat`/`fs.unlink` calls have finished, since you cannot use `await` with callback-style fs directly

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
CORE CALLBACK METHODS
  fs.readFile(path, encoding, cb)        → cb(err, data)      — read whole file
  fs.writeFile(path, content, cb)        → cb(err)            — overwrite/create file
  fs.appendFile(path, content, cb)       → cb(err)            — add to end of file
  fs.unlink(path, cb)                    → cb(err)            — delete a file
  fs.mkdir(path, opts, cb)               → cb(err)            — create a directory
  fs.rmdir(path, cb)                     → cb(err)            — remove EMPTY directory
  fs.rm(path, opts, cb)                  → cb(err)            — remove file/dir (use recursive:true)
  fs.readdir(path, cb)                   → cb(err, files[])   — list directory contents
  fs.stat(path, cb)                      → cb(err, stats)     — get file/folder metadata
  fs.existsSync(path)                    → boolean            — the ONE sync call that's fine to use

ERROR-FIRST CALLBACK RULE
  Every fs callback: (err, result) => { ... }
  ALWAYS check err before touching result — result is undefined when err exists

READFILE ENCODING
  fs.readFile(path, 'utf8', cb)   → data is a STRING
  fs.readFile(path, cb)           → data is a Buffer (raw bytes) — no encoding given

MKDIR / RMDIR GOTCHAS
  mkdir without { recursive: true }  → fails if parent folder missing
  mkdir with    { recursive: true }  → creates nested dirs, no error if already exists
  rmdir → only works on EMPTY directories, throws ENOTEMPTY otherwise
  rm({ recursive: true, force: true }) → deletes folder + all contents, no error if missing

STATS OBJECT (from fs.stat)
  stats.size           → size in bytes
  stats.isFile()        → true/false
  stats.isDirectory()   → true/false
  stats.birthtime       → creation time
  stats.mtime           → last modified time

WHEN TO USE THIS API
  Legacy codebases, some third-party libraries, learning how Node core works
  For NEW code: prefer fs/promises + async/await (Topic 16) — same methods, no callback nesting

NEVER DO
  Use fs.readFileSync/writeFileSync inside a request handler → blocks the event loop for ALL users
  Ignore the err argument                                     → silent, confusing failures
  Use fs.rmdir on a non-empty folder                          → throws ENOTEMPTY, use fs.rm instead
  Nest callbacks 4+ levels deep                                → switch to fs/promises + async/await
```

---

## Connected topics

- **14 — path module** — `path.join()` builds the safe file paths you pass into every fs function shown here
- **16 — fs module (promises API)** — the modern async/await version of every method in this doc, avoiding callback nesting entirely
- **17 — fs module (streams and large files)** — when files are too big to load fully into memory with `readFile`/`writeFile`, streams process them in chunks instead
