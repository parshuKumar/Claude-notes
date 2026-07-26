# 40 — Watching files — fs.watch and chokidar

## What is this?

File watching is asking the operating system to tap you on the shoulder the instant a file or folder changes — instead of you repeatedly checking "has it changed yet?" Node gives you two built-in ways to do this (`fs.watch` and `fs.watchFile`), and the ecosystem gives you a much more reliable third way (the `chokidar` library). Think of it like a security guard watching a door versus you walking back to check the door every five minutes — the guard reacts instantly and never gets tired.

## Why does it matter for backend development?

Backend developers use file watching constantly during development: `nodemon` restarts your server when you save a file, webpack/vite rebuild your bundle when source files change, and TypeScript's `--watch` mode recompiles on save. In production, you might watch a config directory to hot-reload settings without restarting the process, watch an uploads folder to trigger processing jobs, or watch a certificates directory to reload TLS certs when they're renewed. Understanding `fs.watch` also matters because almost every dev-tool you rely on (`nodemon`, `ts-node-dev`, live-reload servers) is built directly on top of it.

---

## Syntax / API

```js
const fs = require('fs');

// ── fs.watch() — event-driven, backed by OS-level file system events ────────
// Fires 'change' or 'rename' events, but filenames can be unreliable cross-platform
const watcher = fs.watch(
  './config',          // path to watch (file or directory)
  { recursive: false }, // options: recursive works on macOS/Windows, not Linux
  (eventType, filename) => {
    // eventType is 'change' or 'rename' (rename = created, deleted, or renamed)
    console.log(`Event: ${eventType}, File: ${filename}`);
  }
);

watcher.close(); // stop watching — always clean up when done

// ── fs.watchFile() — polling based, checks file stats on an interval ────────
// Slower and heavier, but works reliably even on network drives where fs.watch fails
fs.watchFile(
  './config/app.json',           // must be a specific FILE, not a directory
  { interval: 1000 },            // poll every 1000ms (default is 5007ms)
  (curr, prev) => {
    // curr and prev are fs.Stats objects — compare mtime to detect real changes
    if (curr.mtime !== prev.mtime) {
      console.log('File was modified');
    }
  }
);

fs.unwatchFile('./config/app.json'); // stop the specific poller

// ── chokidar — the production-grade solution (npm install chokidar) ─────────
const chokidar = require('chokidar');

const watcherPro = chokidar.watch('./src', {
  ignored: /(^|[/\\])\../,  // ignore dotfiles like .git, .env
  persistent: true,        // keep the process alive while watching
  ignoreInitial: true,      // don't fire 'add' events for existing files on startup
});

watcherPro
  .on('add', (filePath) => console.log(`File added: ${filePath}`))
  .on('change', (filePath) => console.log(`File changed: ${filePath}`))
  .on('unlink', (filePath) => console.log(`File removed: ${filePath}`));
```

---

## How it works — line by line

`fs.watch` asks the operating system's kernel to notify Node whenever something happens inside the watched path (Linux uses `inotify`, macOS uses `FSEvents`, Windows uses `ReadDirectoryChangesW`). Because these three systems behave differently, `fs.watch` sometimes fires twice for one save, sometimes doesn't report the filename, and recursive watching only works on macOS and Windows — not Linux. It is fast and cheap on CPU, but not fully consistent.

`fs.watchFile` takes the opposite approach: it doesn't ask the OS anything. Instead, it repeatedly calls `fs.stat()` on the file at a fixed interval and compares the result to the last check. This makes it slow to notice changes (up to `interval` milliseconds of delay) and wasteful of CPU if you watch many files, but it is completely consistent across every operating system and even works on network-mounted drives where OS-level events don't propagate.

`chokidar` is a wrapper library that uses `fs.watch` where it's reliable and silently falls back to polling where it isn't, while also de-duplicating the double-fire bugs and giving you clean, distinct events (`add`, `change`, `unlink`, `addDir`, `unlinkDir`) instead of the ambiguous `'rename'` string. This is why virtually every real dev-tool (webpack, vite, nodemon) depends on it rather than using `fs.watch` directly.

---

## Example 1 — basic

```js
// File: watch-basic.js
const fs = require('fs');
const path = require('path');

// Absolute path to the folder we want to watch — built with __dirname (Topic 06)
const watchDir = path.join(__dirname, 'watched-folder');

// Make sure the folder exists before we try to watch it
if (!fs.existsSync(watchDir)) {
  fs.mkdirSync(watchDir); // create it if missing
}

console.log(`Watching: ${watchDir}`);

// Start watching the directory (non-recursive — only top-level files)
const watcher = fs.watch(watchDir, (eventType, filename) => {
  // filename can be null on some platforms — always guard against that
  const name = filename || '(unknown file)';
  console.log(`[${eventType}] ${name}`);
});

// Handle errors so a bad watch doesn't crash the whole process silently
watcher.on('error', (err) => {
  console.error('Watcher error:', err.message);
});

// Stop watching automatically after 30 seconds — good practice in scripts/demos
setTimeout(() => {
  watcher.close(); // release the OS-level file handle
  console.log('Stopped watching.');
}, 30000);
```

---

## Example 2 — real world backend use case

```js
// File: config-hot-reload.js
// A backend service that watches its own config file and reloads it in memory
// without needing a full server restart — a common production pattern.

const fs = require('fs');
const path = require('path');

const configPath = path.join(__dirname, 'config', 'app.json');

let appConfig = {}; // holds the currently active config in memory

// Load (or reload) the config file into memory
function loadConfig() {
  try {
    const raw = fs.readFileSync(configPath, 'utf8'); // sync read is fine here — small file, rare event
    appConfig = JSON.parse(raw);
    console.log('[config] Reloaded successfully:', appConfig);
  } catch (err) {
    // Never crash the running server over a bad config edit — keep the old config
    console.error('[config] Reload failed, keeping previous config:', err.message);
  }
}

// Load the config once at startup
loadConfig();

// Debounce timer — prevents reloading multiple times for one editor "save"
// (many editors write a file 2-3 times in rapid succession)
let reloadTimer = null;

fs.watch(configPath, (eventType) => {
  if (eventType !== 'change') return; // ignore rename-only events here

  clearTimeout(reloadTimer);          // cancel any pending reload
  reloadTimer = setTimeout(loadConfig, 200); // wait 200ms for writes to settle
});

// Simulate the app using the live config on every "request"
function handleRequest(userId) {
  // Real handlers would use appConfig.rateLimit, appConfig.featureFlags, etc.
  console.log(`Handling request for user ${userId} with maxUploadSize=${appConfig.maxUploadSize}`);
}

module.exports = { loadConfig, handleRequest, getConfig: () => appConfig };
```

---

## Common mistakes

### Mistake 1 — Never closing the watcher (memory and file-handle leak)

```js
// ❌ WRONG — the watcher runs forever, even after the task that needed it is done
function watchUploadsOnce() {
  fs.watch('./uploads', (eventType, filename) => {
    console.log(`New upload activity: ${filename}`);
  });
  // No way to stop this — it leaks a file descriptor for the life of the process
}

// ✅ CORRECT — always keep a reference and close it when you're done
function watchUploadsOnce() {
  const watcher = fs.watch('./uploads', (eventType, filename) => {
    console.log(`New upload activity: ${filename}`);
  });
  return watcher; // caller decides when to call watcher.close()
}

const uploadWatcher = watchUploadsOnce();
process.on('SIGTERM', () => uploadWatcher.close()); // clean up on shutdown (Topic 64)
```

### Mistake 2 — Trusting `filename` from fs.watch to always be present

```js
// ❌ WRONG — filename is not guaranteed on every platform (can be null on some OS/FS combos)
fs.watch('./data', (eventType, filename) => {
  const ext = filename.split('.').pop(); // crashes with TypeError if filename is null
  console.log(`Changed file type: .${ext}`);
});

// ✅ CORRECT — always guard against a missing filename
fs.watch('./data', (eventType, filename) => {
  if (!filename) {
    console.log('A change happened, but the OS did not report which file.');
    return;
  }
  const ext = filename.split('.').pop();
  console.log(`Changed file type: .${ext}`);
});
```

### Mistake 3 — Reacting to every single fs.watch event without debouncing

```js
// ❌ WRONG — one file save can fire 2-4 'change' events (editors write in chunks),
// causing the reload/rebuild logic to run multiple times per save
fs.watch('./src', { recursive: true }, (eventType, filename) => {
  console.log('Rebuilding...');
  rebuildProject(); // runs 3-4 times for a single save — wastes CPU, may cause races
});

// ✅ CORRECT — debounce so rapid-fire events collapse into one action
let rebuildTimer = null;
fs.watch('./src', { recursive: true }, (eventType, filename) => {
  clearTimeout(rebuildTimer);
  rebuildTimer = setTimeout(() => {
    console.log('Rebuilding...');
    rebuildProject(); // runs once, after events settle for 150ms
  }, 150);
});
```

---

## Practice exercises

### Exercise 1 — easy

Write a script that:
1. Creates a folder called `logs` (if it doesn't already exist) using `__dirname` to build the path.
2. Uses `fs.watch()` to watch that folder.
3. Prints `"[eventType] filename"` to the console every time something changes inside it.
4. Includes an `error` event handler on the watcher that logs any errors.

Test it manually by creating, editing, and deleting a file inside `logs` while the script runs.

```js
// Write your code here
```

---

### Exercise 2 — medium

Build a function `watchConfigFile(filePath, onValidChange)` that:
1. Uses `fs.watchFile()` (not `fs.watch`) to poll the given file every 500ms.
2. Reads the file with `fs.readFileSync` whenever a change is detected, and tries `JSON.parse()` on it.
3. Calls `onValidChange(parsedConfig)` only if the JSON parses successfully.
4. If `JSON.parse` throws, logs an error message but does NOT call `onValidChange` (so a broken edit never breaks the running app).
5. Returns a `stopWatching()` function that calls `fs.unwatchFile(filePath)`.

Test it with a real JSON file, editing it a few times (including once with invalid JSON) to confirm bad edits are ignored.

```js
// Write your code here
```

---

### Exercise 3 — hard

Install `chokidar` (`npm install chokidar`) and build a minimal live-reload system:
1. Watch a `public` directory recursively for `add`, `change`, and `unlink` events, ignoring dotfiles.
2. Keep an in-memory array called `connectedClients` (simulate with plain objects like `{ id, notify: fn }` — no real WebSocket needed for this exercise).
3. On any file change event, debounce for 200ms, then call `.notify()` on every entry in `connectedClients` with a message like `{ type: 'reload', file: filePath }`.
4. Add a function `registerClient(id)` that pushes a new client object (with a `notify` that just logs the message) into `connectedClients`, and `unregisterClient(id)` that removes it.
5. Log a startup message showing how many files chokidar found in `public` after the initial scan (hint: use the `ready` event).

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
FS.WATCH() — event-driven (OS-backed)
  fs.watch(path, [options], (eventType, filename) => {})
  eventType        → 'change' or 'rename' (rename = create/delete/rename)
  filename         → may be null/undefined on some platforms — always guard
  options.recursive→ true works on macOS/Windows, NOT reliable on Linux
  Returns           → an FSWatcher — MUST call .close() to release the handle
  Speed             → instant (OS pushes the event)
  Reliability       → inconsistent across OS/filesystems, can double-fire

FS.WATCHFILE() — polling based
  fs.watchFile(path, [options], (curr, prev) => {})
  path              → must be a single FILE, not a directory
  options.interval  → ms between polls (default 5007ms)
  curr / prev       → fs.Stats objects — compare .mtime to detect real change
  fs.unwatchFile(path) → stops watching that file
  Speed             → delayed by up to `interval` ms
  Reliability       → 100% consistent, works on network drives, but wastes CPU

CHOKIDAR (npm install chokidar)
  chokidar.watch(paths, options)
  Events: 'add', 'change', 'unlink', 'addDir', 'unlinkDir', 'ready', 'error'
  options.ignored       → regex/glob to skip files (e.g. dotfiles, node_modules)
  options.ignoreInitial → don't fire 'add' for pre-existing files on startup
  options.persistent    → keep process alive while watching (default true)
  Best for: any real dev-tool or production hot-reload system

GOTCHAS
  - Always debounce — one save can fire multiple change events
  - Always close/unwatch on shutdown — leaked watchers = leaked file descriptors
  - fs.watch's 'rename' fires for BOTH create and delete — check existsSync to tell which
  - Never trust JSON.parse on a file mid-write — wrap in try/catch, ignore bad reads
  - recursive:true in fs.watch is NOT supported on Linux — use chokidar for that

WHEN TO USE WHICH
  Quick local dev script          → fs.watch
  Watching one config file safely → fs.watchFile
  Anything shipped to production or cross-platform → chokidar
```

---

## Connected topics

- **15 — fs module (callbacks)** — `fs.readFile`, `fs.stat`, `fs.existsSync` are the exact APIs used inside every watch callback to react to a change.
- **19 — events module** — `fs.watch()` returns an `FSWatcher`, which is itself an `EventEmitter` — understanding `.on()`/`.close()` here comes straight from that topic.
- **64 — Graceful shutdown** — watchers hold open OS handles, so `watcher.close()` / `fs.unwatchFile()` must be part of your `SIGTERM` cleanup just like DB connections and open sockets.
