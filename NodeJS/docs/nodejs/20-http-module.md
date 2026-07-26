# 20 — http module

## What is this?

The `http` module is Node's built-in toolkit for speaking the HTTP protocol directly — no Express, no Fastify, nothing installed from npm. It lets you create a server that listens for incoming requests, inspect exactly what the client sent, and hand back a response byte by byte. Think of it like being the switchboard operator at an old telephone exchange: every call (request) comes in on the same line, and it is entirely your job to look at who's calling and what they want, then manually plug the connection through to the right response — there is no automatic routing system doing it for you.

## Why does it matter for backend development?

Every framework you will ever use — Express, Fastify, Koa, NestJS — is built on top of this exact module. `app.listen()` in Express is `http.createServer()` wearing a costume. Understanding the raw `http` module means you understand what a "request", a "response", a "route", and a "status code" actually *are* at the protocol level, instead of just memorizing framework syntax. Backend developers reach for raw `http` directly when writing tiny microservices, health-check servers, proxies, or webhook receivers where pulling in a full framework is unnecessary overhead — and reach for the *understanding* of it constantly, every time they debug why a response hung, why a body came back empty, or why a header didn't apply.

---

## Syntax / API

```js
// Import Node's built-in http module — no npm install needed
const http = require('http');

// createServer() takes a callback that runs on EVERY incoming request
// req = the incoming request (what the client sent)
// res = the outgoing response (what you send back)
const server = http.createServer((req, res) => {
  // req.method → 'GET', 'POST', 'PUT', 'DELETE', etc.
  // req.url    → the path + query string, e.g. '/users?active=true'
  // req.headers → an object of all request headers, lowercase keys

  // Set the HTTP status code and response headers in one call
  res.writeHead(200, { 'Content-Type': 'application/json' });

  // Write the response body — can be called multiple times before end()
  res.write(JSON.stringify({ message: 'Server is running' }));

  // end() finalizes and sends the response — MUST be called or the request hangs
  res.end();
});

// listen() starts the server on a port and optional host
server.listen(3000, () => {
  console.log('Server listening on http://localhost:3000');
});

// Common shortcuts on res:
res.statusCode = 200;                        // set status without writeHead
res.setHeader('Content-Type', 'text/plain');  // set a single header
res.end('plain text body');                   // write + end in one call
```

---

## How it works — line by line

`http.createServer()` builds a server object and registers your callback function to fire every single time a client connects and sends a request — this callback is called the **request handler**. Node does not call your handler once per server; it calls it fresh, every time, for every request, with a brand new `req` and `res` pair.

The `req` object is a **readable stream** — the request body (if any) does not arrive all at once, it arrives in chunks over time, exactly like a file being read piece by piece. This is why you cannot just do `req.body` and expect data — you have to listen for `'data'` events as chunks arrive and a `'end'` event when the client has finished sending.

The `res` object is a **writable stream** going the other direction — you push data into it with `res.write()`, and nothing is actually sent over the network as a *complete* response until you call `res.end()`. If you forget `res.end()`, the connection just sits open forever from the client's point of view — this is one of the most common raw-`http` bugs.

`server.listen(port)` tells the operating system "bind to this port and hand me every incoming connection." From that point on, the process stays alive specifically because it is listening — this is why a bare `http` server script does not exit like a normal script would.

There is no router built in. `req.url` and `req.method` are just strings you have to inspect yourself with `if`/`switch` statements — this manual comparison *is* "routing" at this level. Frameworks exist specifically to replace this manual `if (req.method === 'GET' && req.url === '/users')` pattern with something declarative like `router.get('/users', handler)`.

---

## Example 1 — basic

```js
// File: src/servers/basic-server.js

const http = require('http');   // built-in module, no install required

// createServer runs this function for every single incoming request
const server = http.createServer((req, res) => {
  console.log(`${req.method} ${req.url}`);  // log every request for visibility

  // req.url includes the path — compare it directly for simple routing
  if (req.url === '/' && req.method === 'GET') {
    res.writeHead(200, { 'Content-Type': 'text/plain' });  // status + header together
    res.end('Welcome to the home page');                    // write body + close response
    return;                                                  // stop here, don't fall through
  }

  if (req.url === '/health' && req.method === 'GET') {
    res.writeHead(200, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify({ status: 'ok', uptime: process.uptime() }));
    return;
  }

  // Nothing matched above — this is the fallback "route not found" case
  res.writeHead(404, { 'Content-Type': 'application/json' });
  res.end(JSON.stringify({ error: 'Not found' }));
});

// Start listening on port 3000, on all local network interfaces
server.listen(3000, () => {
  console.log('Listening on http://localhost:3000');
});
```

---

## Example 2 — real world backend use case

```js
// File: src/servers/user-api-server.js
// A tiny in-memory "users" API with manual routing and body parsing —
// the pattern you'd see before a team adopts Express.

const http = require('http');

// In-memory "database" — resets every restart, good enough to demonstrate the pattern
const users = [
  { userId: 'u_1', name: 'Asha Rao', email: 'asha@example.com' },
  { userId: 'u_2', name: 'Rohit Sen', email: 'rohit@example.com' },
];

// Helper: reads the full request body from a stream of chunks and parses it as JSON
function readRequestBody(req) {
  return new Promise((resolve, reject) => {
    let requestBody = '';                        // accumulates incoming chunks as a string

    req.on('data', (chunk) => {
      requestBody += chunk;                       // Buffer chunk auto-converts to string here
    });

    req.on('end', () => {
      if (!requestBody) return resolve({});       // empty body (e.g. GET request) → empty object
      try {
        resolve(JSON.parse(requestBody));         // parse once all chunks have arrived
      } catch (err) {
        reject(new Error('Invalid JSON body'));   // client sent malformed JSON
      }
    });

    req.on('error', reject);                      // network-level failure while reading
  });
}

// Helper: sends a consistent JSON response with correct headers and status
function sendJson(res, statusCode, payload) {
  res.writeHead(statusCode, { 'Content-Type': 'application/json' });
  res.end(JSON.stringify(payload));
}

const server = http.createServer(async (req, res) => {
  const authToken = req.headers['authorization'];   // read a header — always lowercase key

  // Simple auth gate applied to every request except the health check
  if (req.url !== '/health' && authToken !== 'Bearer secret-token') {
    return sendJson(res, 401, { error: 'Missing or invalid auth token' });
  }

  // GET /users — list all users
  if (req.method === 'GET' && req.url === '/users') {
    return sendJson(res, 200, { users });
  }

  // GET /users/u_1 — extract the id from the path manually (no framework param parsing)
  if (req.method === 'GET' && req.url.startsWith('/users/')) {
    const userId = req.url.split('/')[2];             // '/users/u_1' → ['', 'users', 'u_1']
    const user = users.find((u) => u.userId === userId);

    if (!user) return sendJson(res, 404, { error: `No user with id ${userId}` });
    return sendJson(res, 200, { user });
  }

  // POST /users — create a user from the request body
  if (req.method === 'POST' && req.url === '/users') {
    try {
      const requestBody = await readRequestBody(req);  // wait for full body to arrive

      if (!requestBody.name || !requestBody.email) {
        return sendJson(res, 400, { error: 'name and email are required' });
      }

      const newUser = {
        userId: `u_${users.length + 1}`,
        name: requestBody.name,
        email: requestBody.email,
      };
      users.push(newUser);
      return sendJson(res, 201, { user: newUser });     // 201 = resource created
    } catch (err) {
      return sendJson(res, 400, { error: err.message });
    }
  }

  if (req.url === '/health') {
    return sendJson(res, 200, { status: 'ok' });
  }

  // Nothing matched any route above
  sendJson(res, 404, { error: 'Route not found' });
});

server.listen(4000, () => {
  console.log('User API listening on http://localhost:4000');
});
```

---

## Common mistakes

### Mistake 1 — Forgetting to call res.end()

```js
// ❌ WRONG — writeHead + write, but no end() — the client's request hangs forever
const server = http.createServer((req, res) => {
  res.writeHead(200, { 'Content-Type': 'text/plain' });
  res.write('Hello');
  // no res.end() here — browser/client spinner never stops
});

// ✅ CORRECT — always call end(), even with no extra body content
const server = http.createServer((req, res) => {
  res.writeHead(200, { 'Content-Type': 'text/plain' });
  res.write('Hello');
  res.end();   // finalizes and flushes the response — request completes
});
```

### Mistake 2 — Reading req.body directly instead of consuming the stream

```js
// ❌ WRONG — req.body does not exist on the raw http module; it's always undefined here
const server = http.createServer((req, res) => {
  console.log(req.body);          // undefined — req is a stream, not a parsed object
  res.end('done');
});

// ✅ CORRECT — collect chunks from the 'data' event until 'end' fires
const server = http.createServer((req, res) => {
  let requestBody = '';
  req.on('data', (chunk) => { requestBody += chunk; });  // accumulate raw bytes as string
  req.on('end', () => {
    const parsed = requestBody ? JSON.parse(requestBody) : {};
    console.log(parsed);          // now this actually has the client's data
    res.end('done');
  });
});
```

### Mistake 3 — Sending an object without stringifying it, or with the wrong Content-Type

```js
// ❌ WRONG — writing a raw object sends "[object Object]" as text, and the header lies about it
const server = http.createServer((req, res) => {
  res.writeHead(200, { 'Content-Type': 'application/json' });
  res.end({ status: 'ok' });      // res.end expects a string or Buffer, not an object
});

// ✅ CORRECT — JSON.stringify the payload, and match the Content-Type to what you actually send
const server = http.createServer((req, res) => {
  res.writeHead(200, { 'Content-Type': 'application/json' });
  res.end(JSON.stringify({ status: 'ok' }));   // clients can now safely JSON.parse the response
});
```

---

## Practice exercises

### Exercise 1 — easy

Build a raw `http` server on port `3000` that:
1. Responds to `GET /` with plain text `"Home page"` and status `200`
2. Responds to `GET /about` with plain text `"About page"` and status `200`
3. Responds to any other route with status `404` and the JSON body `{ "error": "Not found" }`
4. Logs the method and URL of every incoming request to the console before responding

```js
// Write your code here
```

---

### Exercise 2 — medium

Build a raw `http` server that manages an in-memory list of `products` (each with `productId`, `name`, `price`):
1. `GET /products` → returns the full list as JSON with status `200`
2. `GET /products/:id` (parse the id manually from `req.url`) → returns the matching product, or `404` if no product has that id
3. Any route that isn't recognized → `404` with a JSON error message
4. Every response must have the header `Content-Type: application/json`

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a full in-memory CRUD API for a `todos` resource (each todo has `todoId`, `title`, `completed`) using only the raw `http` module — no frameworks:
1. `GET /todos` — list all todos, status `200`
2. `GET /todos/:id` — get one todo by id, `404` if missing
3. `POST /todos` — read and parse the JSON request body, require a `title` field (`400` if missing), create a new todo with `completed: false`, respond `201` with the created todo
4. `PUT /todos/:id` — read and parse the JSON body, update `title` and/or `completed` on the matching todo, `404` if the id doesn't exist, `200` with the updated todo on success
5. `DELETE /todos/:id` — remove the todo from the in-memory list, `404` if it doesn't exist, `204` (no body) on success
6. Wrap all JSON body parsing in a way that returns `400` with an error message if the client sends malformed JSON, instead of crashing the server

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
CREATING A SERVER
  const http = require('http');
  const server = http.createServer((req, res) => { ... });
  server.listen(port, callback);

REQUEST OBJECT (req) — a READABLE STREAM
  req.method           → 'GET' | 'POST' | 'PUT' | 'DELETE' | ...
  req.url              → path + query string, e.g. '/users?active=true'
  req.headers          → object of headers, ALL KEYS LOWERCASE
  req.on('data', cb)   → fires per chunk of the incoming body (Buffer)
  req.on('end', cb)    → fires once the body has fully arrived
  req.on('error', cb)  → fires on a network-level read failure

RESPONSE OBJECT (res) — a WRITABLE STREAM
  res.writeHead(status, headersObj)   → set status code + headers together
  res.statusCode = 200                → set status code alone
  res.setHeader(name, value)          → set one header alone
  res.write(chunk)                    → send part of the body (can call many times)
  res.end([chunk])                    → finalize and send the response — REQUIRED

READING A JSON BODY (manual pattern)
  let requestBody = '';
  req.on('data', chunk => requestBody += chunk);
  req.on('end', () => {
    const parsed = JSON.parse(requestBody);   // wrap in try/catch — client input is untrusted
  });

MANUAL ROUTING PATTERN
  if (req.method === 'GET'  && req.url === '/users')      { ... }
  if (req.method === 'POST' && req.url === '/users')      { ... }
  if (req.method === 'GET'  && req.url.startsWith('/users/')) {
    const userId = req.url.split('/')[2];                 // crude param extraction
  }

COMMON STATUS CODES
  200 OK                  → successful GET/PUT
  201 Created              → successful POST that made a new resource
  204 No Content            → successful DELETE, no body returned
  400 Bad Request           → malformed input from the client
  401 Unauthorized          → missing/invalid auth credentials
  404 Not Found              → route or resource doesn't exist
  500 Internal Server Error → unhandled error on the server side

NEVER FORGET
  res.end() must always be called, or the request hangs forever
  req.body does NOT exist — you must read the stream yourself
  res.end() only accepts a string or Buffer, never a raw object — JSON.stringify first
```

---

## Connected topics

- **19 — events module** — `http.createServer()` returns an `EventEmitter` under the hood; `req` and `res` themselves emit `'data'`, `'end'`, and `'error'` events you listen to directly
- **22 — url module** — replaces crude `req.url.split('/')` and manual query-string slicing with `new URL()` and `searchParams` for reliable parsing
- **52 — Building a REST API with raw http** — takes everything in this topic further into a real manual router, JSON body parsing helpers, and consistent response shaping without a framework
