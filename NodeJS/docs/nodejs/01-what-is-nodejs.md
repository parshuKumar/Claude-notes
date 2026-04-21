# 01 — What is Node.js

## What is this?

Node.js is a **runtime environment** that lets you run JavaScript outside the browser — on a server, your laptop, or any machine. It is built on top of **V8** (Google's JS engine that also powers Chrome) and a C++ library called **libuv** that handles operating-system-level tasks like file access, networking, and timers. Think of Node.js as a container: it takes your JavaScript code and gives it the ability to talk to the operating system, which plain browser JS can never do.

## Why does it matter for backend development?

Before Node.js, JavaScript could only run in a browser. Backend code had to be written in Java, Python, Ruby, PHP, etc. Node.js changed that — now one language (JS) runs everywhere. Backend developers use Node.js to build HTTP servers, REST APIs, real-time applications, CLI tools, and background job processors. It is especially powerful for I/O-heavy workloads (reading files, making database queries, calling external APIs) because of how it handles concurrency without blocking.

---

## Syntax / API

```js
// Node.js is not a special syntax — it is the same JavaScript you already know.
// What changes is the ENVIRONMENT: what global objects and modules are available.

// --- Available in the BROWSER (NOT in Node) ---
// document.getElementById('app')   // DOM access — does not exist in Node
// window.location.href              // browser window object — does not exist in Node
// localStorage.setItem(...)         // browser storage — does not exist in Node

// --- Available in NODE (NOT in the browser) ---
const fs = require('fs');                      // file system access — only in Node
const path = require('path');                  // OS path utilities — only in Node
console.log(process.env.PORT);                 // process info — only in Node
console.log(__dirname);                        // current directory — only in Node

// --- Available in BOTH ---
console.log('hello');                          // logging works everywhere
const nums = [1, 2, 3].map(n => n * 2);       // array methods — same in both
Promise.resolve(42).then(console.log);         // Promises — same in both
```

---

## How it works — line by line

**What V8 does:**  
V8 is a C++ program that reads your JavaScript text, compiles it to machine code (not interpreted line by line), and executes it at near-native speed. Node.js *embeds* V8 — that is why Node can run JS.

**What libuv adds:**  
V8 can execute JS logic, but it has no idea how to open a file or listen on a network port — those are operating system concepts. libuv is the layer between Node.js and the OS. It provides:
- An **event loop** — the mechanism that keeps Node alive and processes callbacks
- A **thread pool** — a set of background OS threads for operations the OS can't do asynchronously (like reading a file on some systems)
- Async I/O wrappers for networking, file system, DNS, and timers

**What Node.js itself adds on top:**
- The `require()` / `import` module system
- Built-in modules: `fs`, `http`, `path`, `crypto`, `events`, `stream`, etc.
- The `process` global object (access to environment variables, CLI args, exit codes)
- `__dirname` and `__filename` globals
- npm ecosystem compatibility

**The difference from browser JS:**  
| Feature | Browser JS | Node.js |
|---|---|---|
| Runs where? | Inside a browser tab | On any machine (server, laptop, cloud) |
| Global object | `window` | `global` (or `globalThis`) |
| DOM access | Yes | No |
| File system access | No | Yes (`fs` module) |
| Network (raw TCP) | No | Yes (`net` module) |
| Module system | ES Modules (native) | CommonJS (default) + ES Modules |
| Built-in HTTP server | No | Yes (`http` module) |

---

## Example 1 — basic

```js
// File: hello-node.js
// Run with: node hello-node.js

// 'process' is a global object Node gives you automatically — no require needed
console.log('Node version:', process.version);       // e.g. "v20.11.0"
console.log('Platform:', process.platform);          // "win32", "linux", "darwin"
console.log('Working directory:', process.cwd());    // full path to where you ran the command

// __dirname is injected by Node's module system — it is the folder this file lives in
console.log('This file lives in:', __dirname);

// This is plain JavaScript — Node runs it exactly like V8 would
const greet = (userName) => `Hello, ${userName}! Welcome to Node.js.`;
console.log(greet('Parsh'));
```

**Output (example):**
```
Node version: v20.11.0
Platform: win32
Working directory: C:\Users\Parsh\Desktop\NOTES\NodeJS
This file lives in: C:\Users\Parsh\Desktop\NOTES\NodeJS
Hello, Parsh! Welcome to Node.js.
```

---

## Example 2 — real world backend use case

```js
// File: server-info.js
// A backend script that logs environment and system info at startup.
// This is the kind of code you write at the TOP of your server's entry point
// to verify the environment before the app starts accepting requests.

const os = require('os');      // built-in Node module for OS info
const path = require('path');  // built-in Node module for file paths

// Read the PORT from environment variables — never hardcode port numbers
const port = process.env.PORT || 3000;

// Read the environment name (development / staging / production)
const nodeEnv = process.env.NODE_ENV || 'development';

// Log startup info — real servers always log this at boot
console.log('=== Server Starting ===');
console.log(`Environment : ${nodeEnv}`);
console.log(`Port        : ${port}`);
console.log(`Node.js     : ${process.version}`);
console.log(`OS Platform : ${os.platform()}`);       // "linux" on most servers
console.log(`CPU Cores   : ${os.cpus().length}`);    // used for clustering decisions
console.log(`Free Memory : ${Math.round(os.freemem() / 1024 / 1024)} MB`);
console.log(`Entry file  : ${path.basename(__filename)}`);  // "server-info.js"
console.log('======================');

// In a real app, the next line would be: app.listen(port, ...)
// We will build that in Topic 20 — http module
```

**Why this is real:** Every production Node server logs its environment at startup. This tells you immediately if the wrong `NODE_ENV` was set, the wrong port was bound, or memory is dangerously low before a single request is handled.

---

## Common mistakes

### Mistake 1 — Thinking Node.js is a framework

```js
// ❌ WRONG mental model:
// "I'll use Node.js like I use React or Express — it's a framework with helpers"

// ✅ CORRECT mental model:
// Node.js is a RUNTIME — it is the environment that runs your JS.
// Express, Fastify, NestJS are FRAMEWORKS that run ON TOP of Node.
// Just like how Chrome is a runtime and React is a library that runs in Chrome.
```

### Mistake 2 — Using browser globals in Node

```js
// ❌ WRONG — these crash in Node because the browser environment doesn't exist
const userInput = prompt('Enter your name:');   // ReferenceError: prompt is not defined
document.title = 'My App';                      // ReferenceError: document is not defined

// ✅ CORRECT — use Node's equivalents
const readline = require('readline');           // readline module for terminal input
// (covered in Topic 31)
// There is no Node equivalent of document — Node doesn't render HTML
```

### Mistake 3 — Confusing `global` with `window`

```js
// ❌ WRONG — trying to use window in Node
window.myConfig = { debug: true };   // ReferenceError: window is not defined

// ✅ CORRECT — Node's global object is called 'global' (or use globalThis in both)
global.myConfig = { debug: true };   // works in Node
// But: setting things on global is almost always a bad idea — use modules instead
// globalThis works in BOTH Node and browser — prefer it when writing isomorphic code
console.log(globalThis === global);  // true in Node
```

---

## Practice exercises

### Exercise 1 — easy

Write a Node.js script that prints the following information to the console using only `process` (no `require` needed):
- The current Node.js version
- The operating system platform
- The process ID (the unique number the OS assigns to this running process)
- The current working directory

```js
// Write your code here
```

---

### Exercise 2 — medium

Write a Node.js script that:
1. Reads a `PORT` environment variable using `process.env`. If it is not set, default to `8080`.
2. Reads a `APP_NAME` environment variable. If not set, default to `"MyApp"`.
3. Prints a startup banner in this exact format:

```
=============================
  APP_NAME is starting...
  Listening on port: PORT
  Node: vXX.XX.X
  PID: XXXXX
=============================
```

Run it once normally (`node script.js`) and once with the variables set:
- Windows: `$env:PORT=5000; $env:APP_NAME="BackendAPI"; node script.js`
- Linux/Mac: `PORT=5000 APP_NAME="BackendAPI" node script.js`

```js
// Write your code here
```

---

### Exercise 3 — hard

You are a backend developer joining a team. On your first day you need to write a **system diagnostics reporter** — a script that collects and displays everything a senior dev would want to know about the environment the app is running in.

Requirements:
1. Use `require('os')` and `require('path')` (look up the docs or experiment — you haven't formally learned these yet, but a backend dev must be comfortable exploring)
2. Print: Node version, platform, architecture (x64/arm64), total RAM in GB (rounded to 1 decimal), free RAM in GB, number of CPU cores, the hostname of the machine, the home directory, and the absolute path of the running script
3. Add a warning line at the end: if free RAM is less than 20% of total RAM, print `"⚠ WARNING: Low memory"`, otherwise print `"✓ Memory OK"`
4. All output must be formatted in a clean aligned table (pad with spaces or use template literals)

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
WHAT NODE IS
  - JavaScript runtime built on V8 (compiles JS) + libuv (handles OS/async)
  - NOT a framework — a runtime environment
  - Runs on servers, laptops, cloud functions — anywhere but a browser

WHAT NODE ADDS TO JS
  - process global        → env vars, CLI args, exit codes, PID
  - __dirname             → absolute path to the current file's folder
  - __filename            → absolute path to the current file
  - require() / module    → CommonJS module system
  - Built-in modules      → fs, http, path, crypto, events, stream, ...
  - global object         → like window, but for Node (use globalThis for isomorphic)

WHAT IS NOT IN NODE
  - window, document, DOM  → browser only
  - localStorage            → browser only
  - fetch (built-in)        → available from Node 18+, not in older versions
  - alert / prompt          → browser only

KEY COMPARISONS
  V8       → executes your JavaScript
  libuv    → event loop + thread pool + OS I/O
  Node.js  → glues V8 + libuv together + adds standard library + module system
  npm      → package manager that ships with Node (separate project)

COMMON REAL-WORLD USES
  - REST APIs (Express, Fastify)
  - Real-time apps (Socket.IO, WebSockets)
  - CLI tools (npm packages, build tools like webpack, vite)
  - Serverless functions (AWS Lambda, Vercel, Cloudflare Workers)
  - BFF (Backend For Frontend)
  - Microservices
  - Background job processors

VERSION CHECK
  node --version    → check Node version
  node script.js    → run a file
  node              → open REPL (interactive shell)
```

---

## Connected topics

- **02 — How Node.js works internally** — now that you know what Node is, learn *why* it handles 10,000 concurrent connections on a single thread
- **05 — The process object** — deep dive into the `process` global you used in every example above
- **08 — CommonJS modules** — how `require()` works under the hood, which you'll use starting from the very next topic
