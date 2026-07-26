# 27 — child_process module

## What is this?

`child_process` is Node's built-in module for launching other programs — shell commands, executables, or even other Node.js scripts — as separate operating system processes, and talking to them from your Node code. Think of your Node server as a restaurant manager: it doesn't cook every dish itself, it delegates tasks like "run this report script" or "convert this video" to specialists (child processes) and waits for them to report back with results. Node stays free to keep handling requests while the specialist works in the background.

---

## Why does it matter for backend development?

Node is single-threaded and great at I/O, but terrible at CPU-heavy or OS-level tasks — resizing images, running `ffmpeg`, compiling code, running a Python data-science script, calling `git`, or pinging a database CLI tool. Rather than blocking Node's event loop with heavy work, backend developers spawn a child process to do it, freeing Node to keep serving other requests. `child_process` is also how Node apps integrate with the wider OS ecosystem — running migrations, calling ImageMagick, invoking a shell script during deployment, or launching worker scripts that need their own process (and their own crash boundary, so a crash in the child doesn't take down your API server).

---

## Syntax / API

```js
// child_process is a Node core module — no npm install needed
const { exec, execFile, spawn, fork } = require('child_process');

// ── exec() — runs a command through a shell, buffers ALL output in memory ──
exec('ls -la', (error, stdout, stderr) => {
  // error is set if the command fails (non-zero exit code) or can't run at all
  if (error) {
    console.error('exec failed:', error.message);
    return;
  }
  console.log('stdout:', stdout); // full output as one string
  console.error('stderr:', stderr); // warnings/errors printed by the command
});

// ── execFile() — runs an executable directly, NO shell involved ───────────
execFile('node', ['--version'], (error, stdout, stderr) => {
  if (error) return console.error(error);
  console.log('Node version:', stdout.trim()); // e.g. "v20.11.0"
});

// ── spawn() — runs a command and streams output as it happens ─────────────
const child = spawn('ping', ['-c', '4', 'google.com']); // args passed as an array

child.stdout.on('data', (chunk) => {
  console.log('output chunk:', chunk.toString()); // arrives in real time, not all at once
});

child.stderr.on('data', (chunk) => {
  console.error('error chunk:', chunk.toString());
});

child.on('close', (exitCode) => {
  console.log('process finished with code:', exitCode); // 0 = success, non-zero = failure
});

// ── fork() — spawns a NEW NODE.JS PROCESS, with a built-in message channel ─
const workerProcess = fork('./worker.js'); // must be a .js file run by node

workerProcess.send({ task: 'processOrder', orderId: 501 }); // send data to child

workerProcess.on('message', (result) => {
  console.log('reply from worker:', result); // child can send data back
});
```

---

## How it works — line by line

- `require('child_process')` loads Node's built-in tools for launching other programs — no package to install, it ships with Node.
- `exec('ls -la', callback)` opens a shell (like bash), types the command into it, and waits for the whole command to finish before calling the callback once with everything the command printed.
- The callback gets three things: `error` (null if it worked), `stdout` (normal output, as one big string), and `stderr` (error/warning output, also one big string).
- `execFile('node', ['--version'], callback)` runs the `node` program directly by its name, passing `['--version']` as arguments — it does **not** open a shell first, so it's faster and safer against certain attacks.
- `spawn('ping', ['-c', '4', 'google.com'])` starts the `ping` program immediately and returns a handle (`child`) to it right away — it does not wait for the program to finish.
- `child.stdout.on('data', ...)` listens for chunks of output as the program produces them — this is a stream, so you get data piece by piece in real time instead of waiting for everything at once.
- `child.on('close', ...)` fires once the program has fully exited, telling you its exit code — `0` conventionally means success, anything else means something went wrong.
- `fork('./worker.js')` is a special version of spawn built specifically for launching another Node.js file — it automatically sets up a communication channel between the parent and the child.
- `workerProcess.send({ ... })` pushes a JavaScript object over to the child process — Node serializes it for you, so you can send objects directly instead of raw text.
- `workerProcess.on('message', ...)` listens for objects the child sends back the same way, letting parent and child have a real conversation instead of just reading stdout text.

---

## Example 1 — basic

```js
// File: examples/run-shell-command.js
const { exec } = require('child_process'); // import the exec helper

// Run a simple shell command and wait for the full result
exec('node --version', (error, stdout, stderr) => {
  // error is non-null if the command could not run or exited with a failure code
  if (error) {
    console.error('Command failed:', error.message); // log a readable message
    return; // stop here, nothing useful to do with stdout/stderr
  }

  // stderr can contain warnings even when the command "succeeds" — log it separately
  if (stderr) {
    console.warn('Warnings:', stderr); // e.g. deprecation notices from some CLIs
  }

  // stdout holds the actual output of the command as a single string
  console.log('Node version installed:', stdout.trim()); // trim() removes trailing newline
});

console.log('This line runs BEFORE the exec callback');
// exec is asynchronous — Node moves on immediately and reports back later
```

---

## Example 2 — real world backend use case

```js
// File: src/services/imageProcessingService.js
// A backend service that converts an uploaded image to a thumbnail
// using ImageMagick's "convert" CLI tool, without blocking the event loop.

const { spawn } = require('child_process');
const path = require('path');

/**
 * Generates a thumbnail for an uploaded file using the "convert" CLI tool.
 * Uses spawn() (not exec) because image data can be large and we want
 * to stream stderr for logging without buffering huge strings in memory.
 */
function generateThumbnail(filePath, outputDir) {
  return new Promise((resolve, reject) => {
    const fileName = path.basename(filePath, path.extname(filePath)); // strip extension
    const outputPath = path.join(outputDir, `${fileName}-thumb.jpg`); // build output path

    // spawn the "convert" binary with explicit arguments (no shell, no injection risk)
    const convertProcess = spawn('convert', [
      filePath,          // source image
      '-resize', '200x200', // resize flag understood by ImageMagick
      outputPath,        // destination file
    ]);

    let errorOutput = ''; // collect stderr in case the process fails

    convertProcess.stderr.on('data', (chunk) => {
      errorOutput += chunk.toString(); // accumulate error text as it streams in
    });

    convertProcess.on('error', (err) => {
      // fires if the "convert" binary itself doesn't exist on this machine
      reject(new Error(`Failed to start convert: ${err.message}`));
    });

    convertProcess.on('close', (exitCode) => {
      if (exitCode === 0) {
        resolve(outputPath); // success — hand back the thumbnail path
      } else {
        reject(new Error(`convert exited with code ${exitCode}: ${errorOutput}`));
      }
    });
  });
}

// Usage inside an upload route handler:
// const thumbPath = await generateThumbnail(uploadedFile.path, './uploads/thumbs');
// res.json({ thumbnailUrl: `/thumbs/${path.basename(thumbPath)}` });

module.exports = { generateThumbnail };
```

```js
// File: src/services/reportService.js
// Delegates CPU-heavy report generation to a separate Node process via fork(),
// so a big report never blocks the main API server's event loop.

const { fork } = require('child_process');
const path = require('path');

function generateReportInBackground(reportType, userId) {
  return new Promise((resolve, reject) => {
    // fork() launches reportWorker.js as its OWN Node.js process
    const worker = fork(path.join(__dirname, 'reportWorker.js'));

    const requestId = `${userId}-${Date.now()}`; // unique id to match the reply

    worker.send({ reportType, userId, requestId }); // send the job to the worker

    worker.on('message', (result) => {
      if (result.requestId === requestId) {
        resolve(result.rows); // got our answer
        worker.kill(); // free the child process now that we're done with it
      }
    });

    worker.on('error', reject); // worker crashed before it could reply
  });
}

module.exports = { generateReportInBackground };

// File: src/services/reportWorker.js (the forked child — runs as its own process)
// process.on('message', (job) => {
//   const rows = buildHeavyReport(job);          // pretend this is CPU-heavy work
//   process.send({ rows, requestId: job.requestId }); // reply back to the parent
// });
```

---

## Common mistakes

### Mistake 1 — Using exec() with untrusted user input (shell injection)

```js
// ❌ WRONG — exec() runs through a shell, so user input can inject extra commands
const { exec } = require('child_process');

function lookupDomain(userInput) {
  // If userInput is "google.com; rm -rf /", the shell executes BOTH commands
  exec(`nslookup ${userInput}`, (error, stdout) => {
    console.log(stdout);
  });
}

// ✅ CORRECT — use execFile/spawn with an args array, no shell parsing involved
const { execFile } = require('child_process');

function lookupDomainSafely(userInput) {
  // userInput is passed as a single argument, never interpreted as shell syntax
  execFile('nslookup', [userInput], (error, stdout) => {
    console.log(stdout);
  });
}
```

### Mistake 2 — Using exec() for large output and hitting the buffer limit

```js
// ❌ WRONG — exec() buffers all output in memory; default maxBuffer is 1MB
const { exec } = require('child_process');

exec('find / -type f', (error, stdout) => {
  // Large output crashes with: "Error: stdout maxBuffer length exceeded"
  console.log(stdout);
});

// ✅ CORRECT — use spawn() and stream the output instead of buffering it
const { spawn } = require('child_process');

const findProcess = spawn('find', ['/', '-type', 'f']);

findProcess.stdout.on('data', (chunk) => {
  // process each chunk as it arrives — memory stays flat even for huge output
  processChunk(chunk.toString());
});

function processChunk(text) {
  // e.g. write to a log file or count lines, instead of holding everything in RAM
}
```

### Mistake 3 — Forgetting that child_process calls are async and not awaiting/handling them

```js
// ❌ WRONG — treating exec() like a synchronous call, response used before it exists
const { exec } = require('child_process');

function getGitCommit() {
  let commitHash;
  exec('git rev-parse HEAD', (error, stdout) => {
    commitHash = stdout.trim(); // this runs LATER, after the function already returned
  });
  return commitHash; // always undefined here — the callback hasn't fired yet
}

// ✅ CORRECT — wrap it in a Promise and await it, or use the promisified version
const { promisify } = require('util');
const execAsync = promisify(require('child_process').exec); // util.promisify — Topic 24

async function getGitCommitAsync() {
  const { stdout } = await execAsync('git rev-parse HEAD'); // properly waits for completion
  return stdout.trim(); // safe to use — the command has actually finished
}
```

---

## Practice exercises

### Exercise 1 — easy

Write a script that uses `exec()` to run the `node --version` and `npm --version` commands (two separate `exec()` calls). Log both results with clear labels, and log an error message (without crashing) if either command fails. Also log one line before the `exec()` calls to prove they run asynchronously.

```js
// Write your code here
```

---

### Exercise 2 — medium

Build a function `runCommandStreaming(command, args)` using `spawn()` that:
1. Spawns the given command with the given arguments array
2. Collects `stdout` chunks into one string as they arrive, logging each chunk as it's received (to prove streaming, not buffering)
3. Collects `stderr` chunks into a separate string the same way
4. Returns a Promise that resolves with `{ stdout, stderr, exitCode }` once the process closes
5. Rejects the Promise if the command fails to start at all (e.g. binary not found)

Test it by running `spawn`-friendly commands like listing a directory or checking a tool's version, and log the final resolved object.

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a small "background job runner" using `fork()`:
1. Create a worker file `jobWorker.js` that listens for `message` events shaped like `{ jobId, numbers }`, computes the sum of `numbers` (simulate delay with a busy loop or `setTimeout`), and sends back `{ jobId, result }`
2. In a main file, write a `runJob(numbers)` function that forks `jobWorker.js`, sends it a job with a unique `jobId`, waits for the matching reply via a Promise, then kills the worker
3. Add a timeout — if the worker doesn't reply within 3 seconds, reject the Promise with a timeout error and kill the worker anyway
4. Run three jobs with different number arrays sequentially and log each result, plus log what happens when you intentionally make the worker never reply (to see the timeout fire)

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
FOUR WAYS TO RUN A CHILD PROCESS
  exec(command, callback)          → runs via shell, buffers ALL output, callback once at end
  execFile(file, args, callback)   → runs a binary directly, NO shell, safer, buffers output
  spawn(command, args)             → runs via streams, no buffering, best for large/live output
  fork(modulePath)                 → spawn() specialized for launching another Node.js file,
                                      auto-creates a message channel (send/on('message'))

WHEN TO USE WHICH
  exec       → quick shell commands, small/known output, need shell features (pipes, globs)
  execFile   → running a known binary with args, no shell needed, safer against injection
  spawn      → long-running processes, large output, real-time streaming (e.g. ping, ffmpeg)
  fork       → offloading CPU-heavy JS work to another Node process, parent/child messaging

STDOUT / STDERR HANDLING
  exec/execFile → callback(error, stdout, stderr) — both are complete strings
  spawn         → child.stdout.on('data', chunk) / child.stderr.on('data', chunk) — streams
  fork          → same as spawn for stdout/stderr, PLUS send()/on('message') for objects

EVENTS ON A spawn()/fork() CHILD
  'data'    (on stdout/stderr) → a chunk of output arrived
  'error'   → the process could not be started at all (e.g. binary missing)
  'close'   → process fully exited, gives you the exit code
  'exit'    → process exited but streams may still be flushing (close is usually safer)
  'message' → (fork only) the child sent structured data via process.send()

SECURITY RULE
  NEVER interpolate user input into exec()'s command string — shell injection risk
  Use execFile()/spawn() with an args ARRAY instead — arguments are never shell-parsed

GOTCHAS
  exec() has a default maxBuffer of 1MB — large output throws "maxBuffer exceeded"
  All of these are ASYNCHRONOUS — results only exist inside the callback/event/Promise
  fork() only works for .js files run by node — not for arbitrary executables
  Always handle the 'error' event on spawn()/fork() — a missing binary won't throw normally
  Kill long-lived child processes you no longer need with child.kill() to avoid leaks

PROMISE-BASED VERSION
  const { promisify } = require('util');
  const execAsync = promisify(require('child_process').exec);
  const { stdout } = await execAsync('git rev-parse HEAD');
```

---

## Connected topics

- **28 — worker_threads module** — the alternative to `child_process` for CPU-heavy work inside the same process, with lower overhead than forking a full Node process
- **24 — util module** — `util.promisify()` turns the callback-based `exec()` into an awaitable function, as shown in Mistake 3
- **19 — events module** — `spawn()`/`fork()` return an `EventEmitter`-based child object, so everything learned about `.on()` and event handling applies directly here
