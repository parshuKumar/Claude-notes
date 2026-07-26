# 18 — os module

## What is this?

The `os` module is Node's built-in way of asking the **operating system** questions about the machine your code is actually running on — what OS is this, how many CPU cores does it have, how much memory is free, what's this machine's hostname. Think of it like a dashboard gauge panel in a car: it doesn't drive the car, it just tells you the engine temperature, fuel level, and speed so you can make decisions. `os` doesn't do any work for you — it just reports facts about the hardware and environment underneath your Node process.

---

## Why does it matter for backend development?

Production backends don't run on your laptop — they run on servers, containers, and cloud instances that vary in CPU count, memory, and OS. A backend dev uses `os` to write code that **adapts to its environment** instead of hardcoding assumptions: spinning up one worker process per CPU core (`os.cpus().length`), refusing to accept new jobs when memory is critically low (`os.freemem()`), tagging logs with the machine's hostname so you know which server produced them in a multi-server deployment, or writing temp files to the correct OS-specific temp directory. Without `os`, you'd be guessing at server capacity or hardcoding values that break the moment you deploy to a different machine.

---

## Syntax / API

```js
// Import the os module — built into Node, no npm install needed
const os = require('os');

// ── Platform and architecture ───────────────────────────────────────────────
console.log(os.platform());
// → 'linux', 'darwin' (macOS), or 'win32' — the OS Node is running on

console.log(os.arch());
// → 'x64', 'arm64' — the CPU architecture

// ── CPU information ──────────────────────────────────────────────────────────
console.log(os.cpus());
// → array of objects, one per logical core, each with model/speed/times

console.log(os.cpus().length);
// → number of logical CPU cores available — used to size worker pools

// ── Memory information (in bytes) ────────────────────────────────────────────
console.log(os.totalmem());
// → total system RAM in bytes (not just what Node can use — the whole machine)

console.log(os.freemem());
// → currently free system RAM in bytes — changes constantly

// ── Identity of the machine ─────────────────────────────────────────────────
console.log(os.homedir());
// → absolute path to the current user's home directory, e.g. '/home/deploy'

console.log(os.hostname());
// → the machine's network hostname, e.g. 'api-server-03' or 'MacBook-Pro.local'

console.log(os.tmpdir());
// → the OS-specific temp directory, e.g. '/tmp' on Linux, 'C:\Users\...\Temp' on Windows

console.log(os.uptime());
// → seconds the OS itself has been running since last reboot (not the Node process)

console.log(os.type());
// → 'Linux', 'Darwin', 'Windows_NT' — a more verbose OS name

console.log(os.EOL);
// → the OS-specific line-ending character: '\n' on Linux/macOS, '\r\n' on Windows
```

---

## How it works — line by line

`os` is a **synchronous, read-only reporting module** — every function returns an answer immediately, no callbacks or promises needed, because reading OS-level system info doesn't require disk or network I/O the way file operations do.

- `os.platform()` and `os.arch()` never change while your process runs — they describe the machine, not something that fluctuates.
- `os.cpus()` returns one object **per logical core** (not physical core — a machine with hyperthreading reports double the physical count). Each entry has `model` (CPU name), `speed` (MHz), and `times` (milliseconds spent in user/nice/sys/idle/irq states) — most backend code only cares about the **array length**.
- `os.totalmem()` is fixed for the life of the machine, but `os.freemem()` is a live snapshot — it changes every time you call it because other processes on the machine are also using RAM. Both return **bytes**, so you divide by `1024 * 1024` to get MB or `1024 ** 3` for GB.
- `os.homedir()` and `os.tmpdir()` respect environment variables (`$HOME`, `$TMPDIR` on Unix; `%USERPROFILE%`, `%TEMP%` on Windows) so they're always correct regardless of OS.
- `os.hostname()` reflects whatever the OS's hostname is set to — in Docker containers this is usually the container ID unless overridden.

None of these functions take arguments or return promises — they are cheap, synchronous, safe to call as often as you want.

---

## Example 1 — basic

```js
// File: src/scripts/system-info.js
// A quick script to print a snapshot of the machine's specs

const os = require('os'); // built-in module, no install needed

// Basic identity of the machine
console.log('Platform :', os.platform());   // e.g. 'darwin'
console.log('Arch     :', os.arch());       // e.g. 'arm64'
console.log('Hostname :', os.hostname());   // e.g. 'dev-machine.local'
console.log('Home dir :', os.homedir());    // e.g. '/Users/dev'
console.log('Temp dir :', os.tmpdir());     // e.g. '/tmp'

// CPU info — how many cores this machine has
const cpuCount = os.cpus().length;
console.log('CPU cores:', cpuCount);        // e.g. 8

// Memory info — convert bytes to gigabytes for readability
const totalMemGB = (os.totalmem() / 1024 ** 3).toFixed(2);
const freeMemGB  = (os.freemem() / 1024 ** 3).toFixed(2);
console.log(`Total RAM: ${totalMemGB} GB`);
console.log(`Free RAM : ${freeMemGB} GB`);

// Uptime — how long the OS itself has been running
const uptimeHours = (os.uptime() / 3600).toFixed(1);
console.log(`OS uptime: ${uptimeHours} hours`);
```

---

## Example 2 — real world backend use case

```js
// File: src/monitoring/healthMetrics.js
// Powers a /health/metrics endpoint that ops teams and load balancers
// query to decide whether this instance is healthy enough to keep serving traffic.

const os = require('os');

// Build a snapshot of current system health — used by monitoring dashboards
function getSystemHealth() {
  const totalMem = os.totalmem();               // total RAM in bytes
  const freeMem  = os.freemem();                 // free RAM right now, in bytes
  const usedMem  = totalMem - freeMem;           // derive used memory
  const memUsagePercent = ((usedMem / totalMem) * 100).toFixed(1); // % used

  // Average CPU load over 1, 5, and 15 minutes (Unix-style load average)
  // Not available on Windows — returns [0, 0, 0] there
  const [load1m, load5m, load15m] = os.loadavg();

  return {
    hostname:       os.hostname(),               // identifies WHICH server this is
    platform:       os.platform(),                // 'linux' in most production deployments
    cpuCores:       os.cpus().length,             // total logical cores available
    memory: {
      totalMB: Math.round(totalMem / 1024 / 1024),
      freeMB:  Math.round(freeMem / 1024 / 1024),
      usagePercent: Number(memUsagePercent),
    },
    loadAverage: { load1m, load5m, load15m },      // rising load = server getting busy
    uptimeSeconds: Math.round(os.uptime()),        // how long the host has been up
  };
}

// Express-style route handler — ops/monitoring tools poll this endpoint
function healthMetricsHandler(req, res) {
  const health = getSystemHealth();

  // Flag the instance unhealthy if memory usage is dangerously high —
  // an orchestrator (Kubernetes, PM2) can use this to stop routing traffic here
  const isHealthy = health.memory.usagePercent < 90;

  res.status(isHealthy ? 200 : 503).json({
    status: isHealthy ? 'ok' : 'degraded',
    checkedAt: new Date().toISOString(),
    ...health,
  });
}

// Decide worker pool size at startup — one worker per CPU core is a common default
// (used later with the cluster module — Topic 29)
function getRecommendedWorkerCount() {
  const cpuCount = os.cpus().length;
  // Leave one core free for the OS and other processes on small machines
  return cpuCount > 1 ? cpuCount - 1 : 1;
}

module.exports = { getSystemHealth, healthMetricsHandler, getRecommendedWorkerCount };
```

---

## Common mistakes

### Mistake 1 — Treating os.freemem() as "memory available to my Node process"

```js
// ❌ WRONG — os.freemem() is SYSTEM-WIDE free RAM, not what Node/your process can use.
// A server can have 500MB free system-wide but your process's heap could still
// be near its own limit — these are two completely different numbers.
const os = require('os');
if (os.freemem() > 100 * 1024 * 1024) {
  // "There's 100MB free, so my process is fine" — WRONG assumption
  acceptNewRequest();
}

// ✅ CORRECT — check the actual Node process's memory usage separately
const os = require('os');
const memUsage = process.memoryUsage(); // Node-specific, not OS-wide (Topic 05)

if (os.freemem() > 100 * 1024 * 1024 && memUsage.heapUsed < memUsage.heapTotal * 0.9) {
  // Now checking BOTH system-wide headroom AND this process's own heap pressure
  acceptNewRequest();
}
```

### Mistake 2 — Assuming os.cpus().length equals physical CPU cores

```js
// ❌ WRONG — assuming this number maps 1:1 to physical cores when sizing a
// database connection pool or worker count based on "physical" hardware capacity
const os = require('os');
const dbPoolSize = os.cpus().length * 4; // may be counting hyperthreaded logical cores, doubling the real count

// ✅ CORRECT — treat it as "logical cores available to schedule work on" and
// size pools conservatively, testing under real load rather than guessing
const os = require('os');
const cpuCount = os.cpus().length;               // logical cores (may include hyperthreading)
const dbPoolSize = Math.min(cpuCount * 2, 20);   // cap it — don't let pool size explode on big machines
```

### Mistake 3 — Using os.tmpdir() or os.homedir() without path.join()

```js
// ❌ WRONG — string concatenation breaks path separators across OSes,
// same trap as with __dirname (Topic 06)
const os = require('os');
const tempFilePath = os.tmpdir() + '/' + 'upload_' + userId + '.tmp';
// On Windows: 'C:\Users\deploy\Temp/upload_user_42.tmp' — mixed separators

// ✅ CORRECT — always combine os.tmpdir()/os.homedir() with path.join()
const os   = require('os');
const path = require('path');
const tempFilePath = path.join(os.tmpdir(), `upload_${userId}.tmp`);
// Correct separators on every OS: '/tmp/upload_user_42.tmp' or 'C:\Users\deploy\Temp\upload_user_42.tmp'
```

---

## Practice exercises

### Exercise 1 — easy

Write a script that prints a clean, labeled report of the current machine using the `os` module. It must include:
1. Platform and architecture
2. Number of CPU cores
3. Total and free memory, converted to megabytes (not raw bytes)
4. The hostname
5. The OS-specific temp directory path

```js
// Write your code here
```

---

### Exercise 2 — medium

Write a function `getMemoryStatus()` that:
1. Calculates the percentage of system memory currently in use (`used / total * 100`)
2. Returns an object `{ percentUsed, status }` where `status` is:
   - `'healthy'` if usage is below 70%
   - `'warning'` if usage is between 70% and 90%
   - `'critical'` if usage is 90% or above
3. Round `percentUsed` to 1 decimal place

Then call it every 5 seconds using `setInterval` and log the result, so it behaves like a lightweight memory monitor a backend service could run in the background.

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a `SystemMonitor` class that:
1. In its constructor, accepts an options object `{ memoryThresholdPercent = 85, checkIntervalMs = 10000 }`
2. Has a `getSnapshot()` method returning `{ hostname, platform, cpuCores, memoryUsedPercent, uptimeSeconds, timestamp }`
3. Has a `start()` method that begins polling `getSnapshot()` on the configured interval and stores the last 10 snapshots in an internal array (drop the oldest when a new one arrives — a rolling window)
4. Has a `stop()` method that stops the polling
5. Has a `isOverThreshold()` method that returns `true` if the **most recent** snapshot's memory usage exceeds `memoryThresholdPercent`
6. Emits a warning to `console.warn` automatically the moment usage crosses the threshold (only once per crossing, not on every single check while it stays above threshold)

Test it by starting the monitor with a very low `memoryThresholdPercent` (like `1`) so it triggers immediately, and confirm the warning fires exactly once even after several checks.

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
IMPORT
  const os = require('os');   // built-in, no npm install

IDENTITY
  os.platform()   → 'linux' | 'darwin' | 'win32'
  os.type()       → 'Linux' | 'Darwin' | 'Windows_NT'  (verbose name)
  os.arch()       → 'x64' | 'arm64'
  os.hostname()   → machine's network hostname

CPU
  os.cpus()          → array, one entry per LOGICAL core (model, speed, times)
  os.cpus().length   → core count — common use: size worker/cluster pools
  os.loadavg()       → [1m, 5m, 15m] average load — Unix only, [0,0,0] on Windows

MEMORY (always in BYTES — divide to get MB/GB)
  os.totalmem()   → total system RAM (fixed for machine lifetime)
  os.freemem()    → free system RAM RIGHT NOW (changes constantly, system-wide!)
  NOT the same as process.memoryUsage() — that's YOUR Node process only (Topic 05)

PATHS
  os.homedir()    → current user's home dir, respects $HOME / %USERPROFILE%
  os.tmpdir()     → OS temp dir, respects $TMPDIR / %TEMP%
  ALWAYS combine with path.join() — never string-concatenate

MISC
  os.uptime()     → seconds since the OS itself booted (not your Node process)
  os.EOL          → '\n' on Linux/macOS, '\r\n' on Windows

GOTCHAS
  - os.cpus().length counts LOGICAL cores (hyperthreading inflates this)
  - os.freemem() is system-wide, NOT your process's available memory
  - Everything here is synchronous — no callbacks, no promises needed
  - os.hostname() inside Docker is usually the container ID, not the host machine's name

COMMON BACKEND USES
  - Size a cluster/worker pool: os.cpus().length
  - Health check endpoint: os.freemem() / os.totalmem() for memory pressure
  - Tag logs by server: os.hostname()
  - Safe temp file storage: path.join(os.tmpdir(), fileName)
```

---

## Connected topics

- **29 — cluster module** — uses `os.cpus().length` to decide how many worker processes to fork across CPU cores
- **43 — Temporary files and directories** — `os.tmpdir()` is the foundation for safely writing temp files across any OS
- **05 — The process object** — `process.memoryUsage()` and `process.cwd()` complement `os`'s machine-wide facts with process-specific ones; easy to confuse the two
