# 52 — Building a REST API with raw http

## What is this?

Building a REST API with raw `http` means using Node's built-in `http` module — no Express, no Fastify, nothing installed from npm — to accept requests, figure out which route was hit, read the request body, and send back a JSON response. It's like building a restaurant kitchen with just a stove and a knife instead of buying a meal-kit service: slower to set up, but you see and control every single step. Every framework you'll learn later (Express, Fastify) is just a nicer wrapper around exactly this.

---

## Why does it matter for backend development?

You will almost never ship a production API built this way — frameworks exist precisely because manual routing gets tedious fast. But every backend developer should build one raw `http` API at least once, because it strips away the "magic." When Express calls `req.body` or `res.json()`, you'll know exactly what's happening underneath: streams being collected, buffers being parsed, headers being set by hand. This matters when you debug a weird framework behavior, when you evaluate whether a framework is worth the dependency, or when you work somewhere with a "no framework, minimal dependencies" policy (common in some microservices and serverless functions where every millisecond of cold-start matters).

---

## Syntax / API

```js
// Import Node's built-in HTTP module — no npm install needed
const http = require('http');

// createServer takes a callback that runs on EVERY incoming request
const server = http.createServer((req, res) => {
  // req.method → 'GET', 'POST', 'PUT', 'DELETE', etc.
  // req.url    → the path + query string, e.g. '/users/42?active=true'

  // Set the response header BEFORE writing any body content
  res.setHeader('Content-Type', 'application/json');

  // Manually decide status code — Express does this via res.status() for you
  res.statusCode = 200;

  // end() sends the response and closes it — nothing can be sent after this
  res.end(JSON.stringify({ message: 'hello from raw http' }));
});

// listen() starts the TCP server on the given port
server.listen(3000, () => {
  console.log('Server running on http://localhost:3000');
});
```

---

## How it works — line by line

`http.createServer()` takes one function and calls it automatically, once, every time a new HTTP request arrives — you never call it yourself. That function receives two objects: `req` (the incoming request — a readable stream) and `res` (the outgoing response — a writable stream).

Because `req` is a **stream**, the request body does not arrive as a ready-made string or object. It arrives in small chunks over time (this matters for large uploads). To read it, you listen for `'data'` events (each chunk of bytes) and a `'end'` event (no more chunks are coming), then glue the chunks together and run `JSON.parse()` on the result yourself — Express's `express.json()` middleware does exactly this behind the scenes.

Routing is manual: there's no `app.get('/users', ...)` — instead you read `req.method` and `req.url` yourself and write `if`/`else` or a lookup table to decide what to run. Sending a response means calling `res.end()` — nothing is sent to the client until you call it, and calling it twice throws an error.

---

## Example 1 — basic

```js
// Import the built-in http module
const http = require('http');

// createServer's callback runs once per incoming request
const server = http.createServer((req, res) => {
  // Log the method and URL for every request that comes in
  console.log(`${req.method} ${req.url}`);

  // Tell the client we're sending JSON back
  res.setHeader('Content-Type', 'application/json');

  // Manual routing — check method AND url together
  if (req.method === 'GET' && req.url === '/health') {
    // 200 OK — the standard "everything is fine" status
    res.statusCode = 200;
    res.end(JSON.stringify({ status: 'ok' }));
    return; // stop here so we don't fall through to the 404 below
  }

  if (req.method === 'GET' && req.url === '/') {
    res.statusCode = 200;
    res.end(JSON.stringify({ message: 'Welcome to the raw http API' }));
    return;
  }

  // No route matched — respond with 404 Not Found
  res.statusCode = 404;
  res.end(JSON.stringify({ error: 'Route not found' }));
});

// Start listening on port 3000
server.listen(3000, () => {
  console.log('Server listening on http://localhost:3000');
});
```

---

## Example 2 — real world backend use case

```js
// File: src/server.js
// A tiny in-memory "users" REST API built with only the http module —
// this is the kind of exercise interviewers ask to test raw Node knowledge.

const http = require('http');

// In-memory "database" — resets every time the server restarts
let users = [
  { userId: 1, name: 'Asha Rao' },
  { userId: 2, name: 'Vikram Shah' },
];
let nextUserId = 3; // simple auto-increment counter for new users

// Helper — reads and parses the JSON body from a request stream
function readRequestBody(req) {
  return new Promise((resolve, reject) => {
    let rawBody = ''; // accumulates incoming chunks as a string

    // Fired every time a new chunk of the body arrives
    req.on('data', (chunk) => {
      rawBody += chunk; // chunk is a Buffer, but += converts it via toString()
    });

    // Fired once the client has finished sending the body
    req.on('end', () => {
      if (!rawBody) return resolve({}); // no body sent — treat as empty object
      try {
        resolve(JSON.parse(rawBody)); // convert JSON text into a real object
      } catch (err) {
        reject(new Error('Invalid JSON body')); // malformed JSON — reject the promise
      }
    });

    // Fired if the connection breaks mid-stream
    req.on('error', reject);
  });
}

// Helper — sends a consistent JSON response with the right headers
function sendJson(res, statusCode, payload) {
  res.statusCode = statusCode;                 // e.g. 200, 201, 404, 500
  res.setHeader('Content-Type', 'application/json');
  res.end(JSON.stringify(payload));            // serialize and close the response
}

const server = http.createServer(async (req, res) => {
  const { method, url } = req; // destructure for readability

  try {
    // GET /users — list all users
    if (method === 'GET' && url === '/users') {
      return sendJson(res, 200, { users });
    }

    // GET /users/:id — fetch one user by id (manual param parsing)
    if (method === 'GET' && url.startsWith('/users/')) {
      const userId = Number(url.split('/')[2]);      // extract the id segment
      const user = users.find((u) => u.userId === userId);
      if (!user) return sendJson(res, 404, { error: 'User not found' });
      return sendJson(res, 200, { user });
    }

    // POST /users — create a new user from the JSON request body
    if (method === 'POST' && url === '/users') {
      const requestBody = await readRequestBody(req); // { name: '...' }
      if (!requestBody.name) {
        return sendJson(res, 400, { error: 'name is required' }); // 400 Bad Request
      }
      const newUser = { userId: nextUserId++, name: requestBody.name };
      users.push(newUser);
      return sendJson(res, 201, { user: newUser }); // 201 Created
    }

    // DELETE /users/:id — remove a user by id
    if (method === 'DELETE' && url.startsWith('/users/')) {
      const userId = Number(url.split('/')[2]);
      const beforeCount = users.length;
      users = users.filter((u) => u.userId !== userId);
      if (users.length === beforeCount) {
        return sendJson(res, 404, { error: 'User not found' });
      }
      return sendJson(res, 204, undefined); // 204 No Content — nothing to send back
    }

    // Nothing matched any route above
    return sendJson(res, 404, { error: 'Route not found' });
  } catch (err) {
    // JSON.parse failures or unexpected errors land here
    return sendJson(res, 500, { error: err.message }); // 500 Internal Server Error
  }
});

server.listen(3000, () => {
  console.log('Users API running on http://localhost:3000');
});

module.exports = server; // exported so tests can start/stop it programmatically
```

---

## Common mistakes

### Mistake 1 — Calling res.end() before the body has finished streaming in

```js
// ❌ WRONG — req.on('data') is async; this runs before any chunks arrive,
// so requestBody is always empty/undefined
const server = http.createServer((req, res) => {
  let rawBody = '';
  req.on('data', (chunk) => { rawBody += chunk; });
  res.end(JSON.stringify({ received: rawBody })); // rawBody is still '' here!
});

// ✅ CORRECT — only respond inside (or after) the 'end' event,
// once every chunk has actually arrived
const server2 = http.createServer((req, res) => {
  let rawBody = '';
  req.on('data', (chunk) => { rawBody += chunk; });
  req.on('end', () => {
    res.end(JSON.stringify({ received: rawBody })); // now rawBody is complete
  });
});
```

### Mistake 2 — Forgetting Content-Type, so clients misinterpret the response

```js
// ❌ WRONG — no Content-Type header; some clients (and browsers) will treat
// the JSON string as plain text instead of parsing it as JSON
const server = http.createServer((req, res) => {
  res.end(JSON.stringify({ ok: true })); // client sees raw text, not JSON
});

// ✅ CORRECT — always set Content-Type before sending JSON
const server2 = http.createServer((req, res) => {
  res.setHeader('Content-Type', 'application/json'); // tells client "this is JSON"
  res.end(JSON.stringify({ ok: true }));
});
```

### Mistake 3 — Not calling res.end(), or calling it more than once

```js
// ❌ WRONG — matched branch never calls res.end(), so the client hangs forever
// waiting for a response that never comes
const server = http.createServer((req, res) => {
  if (req.url === '/ping') {
    res.statusCode = 200;
    // missing res.end() — request never completes!
  }
});

// ❌ ALSO WRONG — calling res.end() twice throws:
// "Error: Cannot set headers after they are sent to the client"
const server2 = http.createServer((req, res) => {
  res.end(JSON.stringify({ step: 1 }));
  res.end(JSON.stringify({ step: 2 })); // crashes — response already closed
});

// ✅ CORRECT — every code path ends in exactly one res.end() call, and return
// immediately after sending to avoid falling through to more code
const server3 = http.createServer((req, res) => {
  if (req.url === '/ping') {
    res.statusCode = 200;
    res.setHeader('Content-Type', 'application/json');
    res.end(JSON.stringify({ pong: true })); // exactly one end() call
    return;
  }
  res.statusCode = 404;
  res.end(JSON.stringify({ error: 'Not found' }));
});
```

---

## Practice exercises

### Exercise 1 — easy

Build a raw `http` server on port `4000` with exactly two routes:
1. `GET /` — responds with status `200` and JSON `{ message: 'API is running' }`
2. `GET /time` — responds with status `200` and JSON `{ serverTime: <current ISO timestamp> }`

Any other method or path should respond with status `404` and JSON `{ error: 'Route not found' }`. Make sure `Content-Type: application/json` is set on every response.

```js
// Write your code here
```

---

### Exercise 2 — medium

Build a raw `http` server that manages an in-memory list of `notes` (each note is `{ noteId, title, content }`). Implement:
1. `GET /notes` — return all notes
2. `GET /notes/:id` — return one note by id, or `404` if not found
3. `POST /notes` — read the JSON body, validate that `title` and `content` are both present (respond `400` if either is missing), create the note with an auto-incrementing `noteId`, and respond `201` with the created note
4. `PUT /notes/:id` — read the JSON body and update the matching note's `title`/`content`, or `404` if the id doesn't exist

Write a small helper function to parse the JSON request body so you aren't repeating the `data`/`end` listener logic in every route.

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a raw `http` router system from scratch — no framework — that supports dynamic path parameters, then use it to build a small `orders` API.

Requirements:
1. Write a `createRouter()` function that returns an object with `.get(path, handler)`, `.post(path, handler)`, `.put(path, handler)`, `.delete(path, handler)`, and `.handle(req, res)` methods
2. Support path parameters like `/orders/:orderId` — when a request comes in for `/orders/77`, the matched handler should receive `req.params.orderId === '77'`
3. Support JSON body parsing automatically before calling the matched handler (so handlers can just read `req.body`)
4. If no route matches, `.handle()` should send a `404` JSON response automatically
5. If a handler throws (sync or async), `.handle()` should catch it and send a `500` JSON response with the error message — the server must never crash from a bad request
6. Use your router to build `GET /orders`, `GET /orders/:orderId`, `POST /orders`, and `DELETE /orders/:orderId` for an in-memory `orders` array (`{ orderId, item, quantity }`)

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
CREATE A SERVER
  const http = require('http');
  const server = http.createServer((req, res) => { ... });
  server.listen(3000, () => console.log('running'));

READING THE REQUEST
  req.method            → 'GET' | 'POST' | 'PUT' | 'DELETE' | ...
  req.url               → path + query string, e.g. '/users/42?active=true'
  req.headers           → object of all request headers (lowercase keys)

READING A JSON BODY (manual, since req is a stream)
  let rawBody = '';
  req.on('data', chunk => rawBody += chunk);   // collect chunks
  req.on('end', () => {
    const requestBody = JSON.parse(rawBody);   // parse once fully received
  });

SENDING A RESPONSE
  res.statusCode = 200;                                // set status FIRST
  res.setHeader('Content-Type', 'application/json');   // set headers before end()
  res.end(JSON.stringify({ ... }));                    // send + close (call ONCE)

COMMON STATUS CODES
  200 OK                  → successful GET/PUT
  201 Created              → successful POST that creates a resource
  204 No Content            → successful DELETE, nothing to return
  400 Bad Request           → client sent invalid/missing data
  404 Not Found             → route or resource doesn't exist
  500 Internal Server Error → unexpected server-side failure

MANUAL ROUTING PATTERN
  if (req.method === 'GET' && req.url === '/users') { ... return; }
  if (req.method === 'POST' && req.url === '/users') { ... return; }
  // fallback:
  res.statusCode = 404; res.end(...);

GOTCHAS
  - req body arrives in chunks — never read it before the 'end' event
  - res.end() can only be called once per request — always `return` after it
  - JSON.parse() on bad input throws — always wrap body parsing in try/catch
  - no built-in query string parsing — use the `url` module (Topic 22) for that
  - no built-in path params — you must split/match req.url yourself
```

---

## Connected topics

- **20 — http module** — the deeper reference for `http.createServer`, req/res internals, and manual routing that this topic builds on directly
- **22 — url module** — parsing query strings and building URLs properly instead of manually splitting `req.url` strings
- **53 — Express.js fundamentals** — the very next topic; Express automates everything done manually here (routing, JSON body parsing, response helpers) via middleware
