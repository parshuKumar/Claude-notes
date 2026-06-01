# 05 — The process Object

## What is this?

`process` is a global object that Node.js injects into every running script automatically — no `require()` needed. It is your window into the **running Node.js process itself**: what environment it is running in, what arguments were passed to it, what its exit code should be, and how to react to OS-level signals. Think of it like the cockpit of your Node.js application — it shows you everything about the current runtime state and lets you control how the process starts, runs, and shuts down.

## Why does it matter for backend development?

Backend developers use `process` constantly:
- Reading `process.env` to get database URLs, API keys, and port numbers without hardcoding them
- Reading `process.argv` to build CLI tools and scripts that accept arguments
- Calling `process.exit(1)` to shut down with an error code in scripts and CI pipelines
- Listening to `process.on('SIGTERM')` for graceful server shutdown (critical in Docker/Kubernetes)
- Using `process.cwd()` to resolve file paths relative to where the app was launched
- Checking `process.pid` for logging and monitoring

---

## Syntax / API

```js
// process is global — no require needed
// but you can still do: const process = require('process'); — same object

// ── Environment variables ────────────────────────────────────────────────────
process.env.PORT          // string | undefined — any env var set before running
process.env.NODE_ENV      // 'development' | 'production' | 'test' — convention

// ── Command-line arguments ───────────────────────────────────────────────────
process.argv              // Array<string> — all CLI args including node and script path
process.argv[0]           // path to node executable
process.argv[1]           // path to the running script
process.argv[2]           // first user-supplied argument

// ── Working directory and file paths ────────────────────────────────────────
process.cwd()             // string — current working directory (where you ran node)
process.chdir('/tmp')     // change the working directory of the process

// ── Process metadata ─────────────────────────────────────────────────────────
process.pid               // number — OS process ID (e.g. 12345)
process.ppid              // number — parent process ID
process.version           // string — Node.js version e.g. 'v20.11.0'
process.platform          // string — 'linux' | 'darwin' | 'win32'
process.arch              // string — 'x64' | 'arm64' | 'ia32'
process.title             // string — name shown in process list (you can set this)

// ── Exiting ──────────────────────────────────────────────────────────────────
process.exit(0)           // exit with success code
process.exit(1)           // exit with failure code (non-zero = error in CI/scripts)

// ── Standard I/O streams ─────────────────────────────────────────────────────
process.stdin             // Readable stream — keyboard / piped input
process.stdout            // Writable stream — console output
process.stderr            // Writable stream — error output (separate from stdout)

// ── Event listeners ──────────────────────────────────────────────────────────
process.on('exit', (code) => {})            // fires synchronously when process exits
process.on('uncaughtException', (err) => {}) // last-resort error handler
process.on('unhandledRejection', (reason) => {}) // unhandled Promise rejection
process.on('SIGTERM', () => {})             // OS signal — Docker stop / k8s shutdown
process.on('SIGINT', () => {})              // Ctrl+C from terminal

// ── Memory and performance ───────────────────────────────────────────────────
process.memoryUsage()     // { rss, heapTotal, heapUsed, external, arrayBuffers }
process.hrtime.bigint()   // high-resolution timestamp in nanoseconds (for benchmarking)
process.uptime()          // seconds the process has been running
```

---

## How it works — line by line

`process` is not a regular JavaScript object — it is provided by Node.js's C++ bindings and wraps OS-level primitives:

- **`process.env`** — Node reads the OS environment variables when the process starts and exposes them as a plain JS object. Mutating `process.env` only affects the current process (and its children), not the OS.
- **`process.argv`** — the OS passes command-line arguments as a raw string array when the process is created. Node splits them and exposes them as `process.argv`.
- **`process.exit(code)`** — tells libuv to shut down the event loop and tell the OS the exit code. Code `0` = success. Any non-zero = failure. CI systems (GitHub Actions, Jenkins) use this to detect build failures.
- **`process.on('SIGTERM')`** — the OS sends signals to processes. `SIGTERM` is the polite shutdown signal. Docker and Kubernetes send it when stopping a container. Listening lets you close DB connections and finish in-flight requests before exiting.
- **`process.stdout.write`** — `console.log` is just a wrapper around `process.stdout.write` with a newline added. `process.stderr.write` writes to the error stream, which can be piped separately in shell scripts.

---

## Example 1 — basic

```js
// File: process-info.js
// Run: node process-info.js hello world

// Argv: [node-path, script-path, 'hello', 'world']
console.log('=== process info ===');
console.log('Node version :', process.version);
console.log('Platform     :', process.platform);
console.log('Architecture :', process.arch);
console.log('PID          :', process.pid);
console.log('Uptime (s)   :', process.uptime().toFixed(3));
console.log('CWD          :', process.cwd());

console.log('\n=== argv ===');
// Slice off the first two (node path + script path) to get user args
const userArgs = process.argv.slice(2);
console.log('User args:', userArgs);   // ['hello', 'world']

console.log('\n=== env ===');
// Never log ALL of process.env in production — it may contain secrets
const port = process.env.PORT || '3000';
const nodeEnv = process.env.NODE_ENV || 'development';
console.log('PORT     :', port);
console.log('NODE_ENV :', nodeEnv);

console.log('\n=== memory ===');
const mem = process.memoryUsage();
// rss = Resident Set Size — total memory allocated for the process
console.log('Heap used :', Math.round(mem.heapUsed / 1024 / 1024), 'MB');
console.log('RSS       :', Math.round(mem.rss / 1024 / 1024), 'MB');
```

---

## Example 2 — real world backend use case

```js
// File: server-entry.js
// Production-grade server entry point using process correctly.
// This is the pattern you will use in every real Node.js backend.

// ── 1. Read config from environment (never hardcode) ────────────────────────
const port     = parseInt(process.env.PORT, 10) || 3000;
const nodeEnv  = process.env.NODE_ENV || 'development';
const dbUrl    = process.env.DATABASE_URL;   // must be set — checked below

if (!dbUrl) {
  // Missing required env var — exit immediately with error code 1
  // Scripts and CI pipelines will detect non-zero exit as a failure
  process.stderr.write('[FATAL] DATABASE_URL environment variable is not set\n');
  process.exit(1);
}

// ── 2. Set a descriptive process title (shows in `ps`, `top`, Activity Monitor) ─
process.title = `my-api-server (${nodeEnv})`;

// ── 3. Graceful shutdown — critical for Docker / Kubernetes ─────────────────
let httpServer;   // will hold the server instance once started

function gracefulShutdown(signal) {
  process.stdout.write(`\n[${signal}] Shutdown signal received — closing server\n`);

  if (!httpServer) {
    process.exit(0);
    return;
  }

  // Stop accepting new connections, wait for existing ones to finish
  httpServer.close(() => {
    process.stdout.write('[shutdown] HTTP server closed — process exiting\n');
    // Close DB connections here in a real app
    process.exit(0);
  });

  // Force exit after 10 seconds if graceful shutdown hangs
  setTimeout(() => {
    process.stderr.write('[shutdown] Forced exit after timeout\n');
    process.exit(1);
  }, 10_000).unref();   // .unref() so this timer doesn't keep the process alive
}

// Docker sends SIGTERM when stopping a container
process.on('SIGTERM', () => gracefulShutdown('SIGTERM'));
// Ctrl+C from terminal
process.on('SIGINT',  () => gracefulShutdown('SIGINT'));

// ── 4. Last-resort error handlers ───────────────────────────────────────────
// These should log and exit — never silently swallow errors
process.on('uncaughtException', (err) => {
  process.stderr.write(`[uncaughtException] ${err.stack}\n`);
  process.exit(1);   // always exit after uncaught exception — process state is unknown
});

process.on('unhandledRejection', (reason) => {
  process.stderr.write(`[unhandledRejection] ${reason}\n`);
  process.exit(1);
});

// ── 5. Startup log ───────────────────────────────────────────────────────────
const http = require('http');

httpServer = http.createServer((req, res) => {
  res.writeHead(200, { 'Content-Type': 'application/json' });
  res.end(JSON.stringify({ status: 'ok', pid: process.pid, env: nodeEnv }));
});

httpServer.listen(port, () => {
  process.stdout.write(
    `[startup] Server running | env=${nodeEnv} port=${port} pid=${process.pid}\n`
  );
});
```

---

## Common mistakes

### Mistake 1 — Reading process.env values as numbers without parsing

```js
// ❌ WRONG — process.env always returns STRINGS, never numbers
const port = process.env.PORT;
const server = app.listen(port);
// port is "3000" (string) — app.listen() coerces it, but arithmetic fails:
console.log(port + 1);   // "30001" (string concat) not 3001

// ✅ CORRECT — always parse numeric env vars explicitly
const port = parseInt(process.env.PORT, 10) || 3000;
console.log(port + 1);   // 3001 ✓
```

### Mistake 2 — Using process.exit() inside an async server handler

```js
// ❌ WRONG — calling process.exit() directly inside a request kills the entire server
app.get('/admin/restart', (req, res) => {
  res.send('Restarting...');
  process.exit(0);   // kills the process immediately — all other requests die too
});

// ✅ CORRECT — use graceful shutdown, or never call exit inside a live request handler
// If you need to restart, use a process manager (PM2, systemd) that auto-restarts
// and trigger shutdown via a signal:
app.get('/admin/restart', (req, res) => {
  res.send('Graceful shutdown initiated');
  process.kill(process.pid, 'SIGTERM');   // sends signal to self → triggers gracefulShutdown()
});
```

### Mistake 3 — Logging process.env directly (leaking secrets)

```js
// ❌ WRONG — logs ALL environment variables including secrets to stdout
console.log('Starting with env:', process.env);
// Output includes DATABASE_URL, JWT_SECRET, AWS_SECRET_ACCESS_KEY, etc.

// ✅ CORRECT — log only the safe, non-sensitive vars you explicitly choose
console.log('Starting with config:', {
  port:    process.env.PORT,
  nodeEnv: process.env.NODE_ENV,
  // Never log: DATABASE_URL, JWT_SECRET, API_KEY, etc.
});
```

---

## Practice exercises

### Exercise 1 — easy

Write a script `system-report.js` that uses only `process` (no `require`) to print:
- Node.js version
- Platform and architecture
- Process ID
- How long the process has been running (in milliseconds, rounded)
- Heap memory used in MB (from `process.memoryUsage()`)
- The value of `process.env.NODE_ENV` — if not set, print `"not set"`

```js
// Write your code here
```

---

### Exercise 2 — medium

Build a CLI argument parser. Write a script `cli-tool.js` that:

1. Reads arguments from `process.argv` (slice off the first two)
2. Supports this interface:
   ```
   node cli-tool.js --userId=user_42 --action=delete --verbose
   ```
3. Parses all `--key=value` args into an object and `--flag` args into booleans
4. Logs the parsed config object
5. If `--userId` or `--action` is missing, write an error to `process.stderr` and call `process.exit(1)`

Test with:
```
node cli-tool.js --userId=user_99 --action=export --verbose
node cli-tool.js --action=export
```

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a production-style server entry point. Requirements:

1. Read `PORT` (default 4000) and `NODE_ENV` (default `'development'`) from `process.env`
2. If `NODE_ENV` is `'production'` and `PORT` is still the default 4000, write a warning to `process.stderr` (production should always have an explicit port)
3. Set `process.title` to `"practice-server (NODE_ENV)"`
4. Create a basic `http` server that responds with a JSON body containing `{ pid, uptime, memoryUsedMB, nodeEnv }`
5. Implement graceful shutdown on both `SIGTERM` and `SIGINT` — log the signal received, close the server, then exit with code 0
6. Add an `uncaughtException` handler that logs the error stack to stderr and exits with code 1
7. Log a startup line to stdout when the server starts listening

Test graceful shutdown by running the server and pressing Ctrl+C.

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
GLOBAL — no require() needed (but require('process') works too)

KEY PROPERTIES
  process.env          → object of all environment variables (values are always strings)
  process.argv         → ['node', 'script.js', ...userArgs] — slice(2) for user args
  process.pid          → OS process ID number
  process.ppid         → parent process ID
  process.version      → Node.js version string e.g. 'v20.11.0'
  process.versions     → { node, v8, uv, ... } — all component versions
  process.platform     → 'linux' | 'darwin' | 'win32'
  process.arch         → 'x64' | 'arm64' | 'ia32'
  process.title        → process name in OS task list (writable)
  process.env.NODE_ENV → 'development' | 'production' | 'test' (convention, not enforced)

KEY METHODS
  process.cwd()              → current working directory
  process.chdir(path)        → change working directory
  process.exit(code)         → 0 = success, non-zero = error
  process.kill(pid, signal)  → send a signal to a process (can target itself)
  process.uptime()           → seconds since process started
  process.memoryUsage()      → { rss, heapTotal, heapUsed, external, arrayBuffers }
  process.hrtime.bigint()    → nanosecond-precision timestamp (for benchmarking)
  process.nextTick(fn)       → schedule fn in nextTick queue (covered in Topic 04)

STREAMS
  process.stdin   → Readable  — pipe in data from terminal or another process
  process.stdout  → Writable  — console.log writes here
  process.stderr  → Writable  — errors go here, separate from stdout

IMPORTANT SIGNALS
  SIGTERM → polite shutdown (Docker stop, Kubernetes pod termination)
  SIGINT  → Ctrl+C in terminal
  SIGHUP  → terminal closed / reload config (common in daemons)

PROCESS EVENTS
  'exit'               → fires on exit (sync only — no async here)
  'uncaughtException'  → unhandled thrown error — log and exit(1)
  'unhandledRejection' → unhandled Promise rejection — log and exit(1)
  'SIGTERM'            → graceful shutdown trigger
  'SIGINT'             → Ctrl+C graceful shutdown trigger

RULES
  - process.env values are ALWAYS strings — parseInt() / === comparison carefully
  - Never log process.env wholesale — it contains secrets
  - Always exit(1) after uncaughtException — process state is corrupted
  - Use .unref() on force-exit timers so they don't keep the process alive
```

---

## Connected topics

- **06 — \_\_dirname and \_\_filename** — sister globals to `process`, also injected by Node, used for file path resolution
- **10 — Environment variables** — deep dive into `.env` files, `dotenv`, and secrets management built on `process.env`
- **08 — Errors and error handling** — `process.on('uncaughtException')` and `process.on('unhandledRejection')` in full detail
