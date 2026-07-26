# 36 — HTTP deep dive — headers, methods, status codes

## What is this?

HTTP is the request/response contract every backend speaks with every client — browsers, mobile apps, other servers. A request is a method (what you want to do), a URL (where), headers (metadata about the request), and an optional body (the data). A response is a status code (what happened), headers (metadata about the response), and an optional body (the result). Think of it like sending a package through a courier: the method is the instruction ("deliver", "pick up", "destroy"), the headers are the shipping label (fragile, insured, contents type), and the status code is the courier's receipt back to you (delivered, address not found, refused).

## Why does it matter for backend development?

Every REST API you build lives or dies on getting this contract right. Pick the wrong method and clients cache or retry requests incorrectly. Forget `Content-Type` and the client can't parse your response. Skip proper status codes and every error looks like a 200 to automated tooling, breaking monitoring and retries. Get `Authorization` wrong and you leak security holes. A backend developer who deeply understands headers, methods, idempotency, and status codes writes APIs that behave predictably under load balancers, caches, proxies, retries, and third-party integrations — instead of APIs that "work on my machine" and break in production.

---

## Syntax / API

```js
const http = require('http'); // Node's built-in HTTP module — no install needed

const server = http.createServer((req, res) => {
  // req.method → the HTTP verb the client sent: 'GET', 'POST', 'PUT', 'PATCH', 'DELETE'
  console.log('Method:', req.method);

  // req.url → the path + query string the client requested, e.g. '/users/42?include=orders'
  console.log('URL:', req.url);

  // req.headers → a plain object of all request headers, lowercased keys
  console.log('Headers:', req.headers);
  // e.g. { 'content-type': 'application/json', authorization: 'Bearer abc123' }

  // Reading a specific header — always lowercase, HTTP header names are case-insensitive
  const contentType = req.headers['content-type'];
  const authHeader   = req.headers['authorization'];
  const acceptHeader = req.headers['accept'];

  // Setting response headers BEFORE writing the status/body
  res.setHeader('Content-Type', 'application/json'); // tells client how to parse the body
  res.setHeader('X-Request-Id', 'req_12345');         // custom header — any X- or custom name works

  // writeHead sets the status code and can set headers in one call
  res.writeHead(200, { 'Content-Type': 'application/json' });

  // res.end() sends the body and closes the response — always call it exactly once
  res.end(JSON.stringify({ message: 'ok' }));
});

server.listen(3000, () => {
  console.log('Server listening on port 3000');
});
```

---

## How it works — line by line

- `req.method` tells you what the client wants to do. `GET` means "give me data", `POST` means "create something", `PUT`/`PATCH` mean "update something", `DELETE` means "remove something".
- `req.url` is only the path and query string — never the full URL with domain. To get query params properly, you parse it with the `url` module (Topic 22).
- `req.headers` arrives as a JavaScript object where Node has already lowercased every header name for you, because HTTP headers are officially case-insensitive (`Content-Type` and `content-type` mean the same thing on the wire).
- `Content-Type` on a **request** tells the server what format the incoming body is in (`application/json`, `multipart/form-data`, etc.). `Content-Type` on a **response** tells the client what format the outgoing body is in.
- `Accept` is the client telling the server what format it *wants back* (e.g. `application/json` vs `text/html`). A well-behaved API can inspect this and respond differently.
- `Authorization` carries credentials — usually `Bearer <token>` for JWTs/API keys, or `Basic <base64>` for username:password. The server reads this to identify who is calling.
- `res.writeHead(statusCode, headers)` is how the server tells the client "here's what happened" (status code) and "here's how to read what follows" (headers). It must be called before any body is written.
- `res.end(body)` flushes the body and closes the connection for this request — after this, nothing more can be sent for that response.
- The full lifecycle per request: client connects → sends method + URL + headers + (maybe) body → server reads it all → server processes → server sends status + headers + body → connection is kept alive or closed depending on `Connection` header/HTTP version.

---

## Example 1 — basic

```js
const http = require('http');

const server = http.createServer((req, res) => {
  // Only handle GET requests to '/status' — anything else gets a 404
  if (req.method === 'GET' && req.url === '/status') {
    // Inspect what format the client wants back
    const acceptsJson = req.headers['accept'] === 'application/json';

    if (acceptsJson) {
      // Respond with JSON — 200 OK means "request succeeded, here is your data"
      res.writeHead(200, { 'Content-Type': 'application/json' });
      res.end(JSON.stringify({ status: 'healthy', uptime: process.uptime() }));
    } else {
      // Fallback for browsers or clients that didn't ask for JSON
      res.writeHead(200, { 'Content-Type': 'text/plain' });
      res.end('Server is healthy');
    }
    return; // stop here — response already sent
  }

  // Any other route → 404 Not Found, the resource doesn't exist
  res.writeHead(404, { 'Content-Type': 'application/json' });
  res.end(JSON.stringify({ error: 'Route not found' }));
});

server.listen(4000, () => {
  console.log('Basic status server on http://localhost:4000');
});

// Test with:
// curl -H "Accept: application/json" http://localhost:4000/status
// curl http://localhost:4000/unknown   → 404
```

---

## Example 2 — real world backend use case

```js
// A minimal "users" endpoint showing correct method handling, headers,
// authorization checks, and REST-appropriate status codes.

const http = require('http');

// Fake in-memory "database" for demonstration
const users = new Map([
  ['user_1', { userId: 'user_1', name: 'Asha Rao' }],
]);

// Fake API key check — in a real app this would verify a JWT or session
const VALID_API_KEY = 'sk_live_abc123';

function sendJson(res, statusCode, payload) {
  // Reusable helper so every response uses the same header + serialization logic
  res.writeHead(statusCode, { 'Content-Type': 'application/json' });
  res.end(JSON.stringify(payload));
}

const server = http.createServer((req, res) => {
  // 1. Authentication check — every route below requires a valid Authorization header
  const authHeader = req.headers['authorization']; // e.g. "Bearer sk_live_abc123"
  const apiKey = authHeader && authHeader.startsWith('Bearer ')
    ? authHeader.slice(7) // strip "Bearer " prefix to get the raw key
    : null;

  if (!apiKey || apiKey !== VALID_API_KEY) {
    // 401 Unauthorized — the client did not prove who it is (missing/invalid credentials)
    return sendJson(res, 401, { error: 'Missing or invalid API key' });
  }

  // 2. Route: GET /users/:id — fetch a single user (safe, read-only, idempotent)
  if (req.method === 'GET' && req.url.startsWith('/users/')) {
    const userId = req.url.split('/users/')[1]; // extract the id segment
    const user = users.get(userId);

    if (!user) {
      // 404 Not Found — the requested resource does not exist
      return sendJson(res, 404, { error: `User ${userId} not found` });
    }
    // 200 OK — resource found and returned
    return sendJson(res, 200, user);
  }

  // 3. Route: POST /users — create a new user (not idempotent — calling twice creates two users)
  if (req.method === 'POST' && req.url === '/users') {
    // Reject bodies that aren't declared as JSON
    if (req.headers['content-type'] !== 'application/json') {
      // 415 Unsupported Media Type — server refuses because Content-Type isn't acceptable
      return sendJson(res, 415, { error: 'Content-Type must be application/json' });
    }

    let requestBody = '';
    req.on('data', (chunk) => { requestBody += chunk; }); // accumulate body chunks

    req.on('end', () => {
      let parsedBody;
      try {
        parsedBody = JSON.parse(requestBody); // parse the accumulated JSON body
      } catch (err) {
        // 400 Bad Request — client sent malformed JSON, not the server's fault
        return sendJson(res, 400, { error: 'Invalid JSON body' });
      }

      const newUserId = `user_${users.size + 1}`;
      const newUser = { userId: newUserId, name: parsedBody.name };
      users.set(newUserId, newUser);

      // 201 Created — new resource successfully created, not just 200
      res.setHeader('Location', `/users/${newUserId}`); // where to find the new resource
      return sendJson(res, 201, newUser);
    });
    return; // response is sent async inside 'end' handler above
  }

  // 4. Route: DELETE /users/:id — remove a user (idempotent — deleting twice is still "gone")
  if (req.method === 'DELETE' && req.url.startsWith('/users/')) {
    const userId = req.url.split('/users/')[1];
    users.delete(userId); // Map.delete is a no-op if the key isn't there — safe to call repeatedly

    // 204 No Content — success, but there is nothing to send back
    res.writeHead(204);
    return res.end();
  }

  // 5. Anything unmatched — method not implemented for this path
  sendJson(res, 405, { error: `${req.method} not allowed on ${req.url}` });
});

server.listen(5000, () => {
  console.log('Users API on http://localhost:5000');
});
```

---

## Common mistakes

### Mistake 1 — Using GET for actions that change data

```js
// ❌ WRONG — GET requests must never mutate state.
// Browsers, proxies, and crawlers can pre-fetch or retry GETs automatically,
// so a GET that deletes data can be triggered accidentally.
if (req.method === 'GET' && req.url === '/users/delete') {
  users.delete(req.headers['user-id']);
  res.end('deleted');
}

// ✅ CORRECT — mutations belong on POST/PUT/PATCH/DELETE.
// GET stays safe and idempotent — calling it never changes anything.
if (req.method === 'DELETE' && req.url.startsWith('/users/')) {
  const userId = req.url.split('/users/')[1];
  users.delete(userId);
  res.writeHead(204);
  res.end();
}
```

### Mistake 2 — Sending 200 OK for every response, including errors

```js
// ❌ WRONG — always returning 200 makes errors invisible to monitoring,
// load balancers, and client retry logic. The body has to be parsed just to know it failed.
res.writeHead(200, { 'Content-Type': 'application/json' });
res.end(JSON.stringify({ success: false, error: 'User not found' }));

// ✅ CORRECT — use the status code that matches what actually happened.
// Clients, proxies, and monitoring tools can react correctly without parsing the body.
res.writeHead(404, { 'Content-Type': 'application/json' });
res.end(JSON.stringify({ error: 'User not found' }));
```

### Mistake 3 — Trusting Content-Type or Authorization without validating them

```js
// ❌ WRONG — assumes the header exists and is well-formed; crashes on missing/malformed input
const apiKey = req.headers['authorization'].split(' ')[1]; // TypeError if header is undefined
const requestBody = JSON.parse(rawBody); // throws if Content-Type lied or body is malformed

// ✅ CORRECT — always guard before trusting untrusted client input
const authHeader = req.headers['authorization'];
const apiKey = authHeader && authHeader.startsWith('Bearer ')
  ? authHeader.slice(7)
  : null;

if (!apiKey) {
  res.writeHead(401, { 'Content-Type': 'application/json' });
  return res.end(JSON.stringify({ error: 'Authorization header required' }));
}

let requestBody;
try {
  requestBody = JSON.parse(rawBody);
} catch (err) {
  res.writeHead(400, { 'Content-Type': 'application/json' });
  return res.end(JSON.stringify({ error: 'Malformed JSON body' }));
}
```

---

## Practice exercises

### Exercise 1 — easy

Build a raw `http` server on port `6000` with a single route `GET /ping`. It should:
1. Read the `Accept` header from the request.
2. If `Accept` includes `application/json`, respond with status `200` and JSON body `{ "message": "pong" }`.
3. Otherwise respond with status `200` and plain text body `"pong"`.
4. Any other method or path should respond with status `404` and a JSON error body.

Test both cases with `curl -H "Accept: application/json" http://localhost:6000/ping` and plain `curl http://localhost:6000/ping`.

```js
// Write your code here
```

---

### Exercise 2 — medium

Build a `notes` API on port `7000` with an in-memory array store. Implement:
1. `POST /notes` — reads a JSON body `{ "title": "...", "body": "..." }`, rejects with `415` if `Content-Type` isn't `application/json`, rejects with `400` if the JSON is malformed or `title` is missing, otherwise creates the note and responds `201` with a `Location` header pointing to `/notes/:id`.
2. `GET /notes/:id` — responds `200` with the note if found, `404` if not.
3. `PUT /notes/:id` — fully replaces the note (requires both `title` and `body` in the request), responds `200` with the updated note, or `404` if the note doesn't exist.
4. `DELETE /notes/:id` — removes the note, responds `204` with no body, and is idempotent (deleting a missing note still responds `204`, not an error).
5. Any unmatched method/path combination responds `405` with an `Allow` header listing the supported methods for that path.

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a small API-key-protected `orders` API on port `8000` that demonstrates full REST semantics and idempotency handling:
1. Every route requires `Authorization: Bearer <apiKey>`; missing or wrong key → `401`.
2. `POST /orders` creates an order from a JSON body `{ "userId": "...", "amount": ... }`. Support an optional `Idempotency-Key` request header: if the same key is sent twice, return the **original** created order with status `200` (not a duplicate `201`) instead of creating a second order. Store a map of idempotency keys to the orders they created.
3. `GET /orders/:id` returns the order or `404`.
4. `PATCH /orders/:id` partially updates only the fields present in the JSON body (e.g. just `{ "amount": 50 }` should not erase `userId`). Returns `200` with the merged order or `404` if missing.
5. `DELETE /orders/:id` cancels the order (idempotent — always `204`, even if already cancelled or missing).
6. Any request with `Content-Type` other than `application/json` on a body-carrying route → `415`.
7. Any malformed JSON body → `400` with an error explaining what went wrong.

Manually test the idempotency behavior by sending the same `POST /orders` request twice with the same `Idempotency-Key` header and confirming only one order exists.

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
REQUEST LIFECYCLE
  Client → TCP connect → send method + URL + headers + (body) →
  Server reads request → processes → sends status + headers + (body) → connection kept-alive/closed

HTTP METHODS (semantics)
  GET     → read data, safe, idempotent, cacheable, no body expected
  POST    → create/trigger action, NOT idempotent (calling twice = two side effects)
  PUT     → full replace of a resource, idempotent (same call twice = same end state)
  PATCH   → partial update of a resource, idempotent in practice (usually)
  DELETE  → remove a resource, idempotent (deleting twice still ends "gone")
  HEAD    → like GET but headers only, no body — used to check existence/size
  OPTIONS → asks what methods/headers are allowed (used in CORS preflight)

IDEMPOTENT vs NOT
  Idempotent (safe to retry blindly): GET, PUT, DELETE, HEAD, OPTIONS
  NOT idempotent (retry can duplicate effects): POST, sometimes PATCH
  Use an Idempotency-Key header on POST when retries must not double-create

KEY REQUEST HEADERS
  Content-Type    → format of the BODY BEING SENT (application/json, multipart/form-data, ...)
  Accept          → format the client WANTS BACK (application/json, text/html, */*)
  Authorization   → credentials: "Bearer <token>" (JWT/API key) or "Basic <base64>"
  Content-Length  → size in bytes of the request body
  User-Agent      → identifies the calling client software

KEY RESPONSE HEADERS
  Content-Type    → format of the response body — client parses accordingly
  Location        → path to a newly created resource (used with 201)
  Allow           → lists supported methods when responding 405
  Set-Cookie      → sets a cookie on the client
  Cache-Control   → caching rules (max-age, no-store, private, public)

STATUS CODE FAMILIES
  1xx  Informational  → 100 Continue
  2xx  Success         → 200 OK, 201 Created, 204 No Content
  3xx  Redirection     → 301 Moved Permanently, 302 Found, 304 Not Modified
  4xx  Client error    → 400 Bad Request, 401 Unauthorized, 403 Forbidden,
                          404 Not Found, 405 Method Not Allowed, 409 Conflict,
                          415 Unsupported Media Type, 429 Too Many Requests
  5xx  Server error    → 500 Internal Server Error, 502 Bad Gateway, 503 Service Unavailable

COMMON MIX-UPS
  401 vs 403 → 401 = "who are you?" (no/bad credentials), 403 = "I know you, still no"
  200 vs 204 → 200 has a body, 204 explicitly has none
  400 vs 422 → 400 = malformed request, 422 = well-formed but semantically invalid

NEVER DO
  Return 200 for every response regardless of outcome
  Mutate state on GET requests
  Trust headers (Content-Type, Authorization) without validating/guarding them
  Send a body with a 204 or 304 response
```

---

## Connected topics

- **20 — http module** — the raw `http.createServer` API this document builds directly on top of.
- **22 — url module** — parsing `req.url` and query strings properly instead of manual string splitting.
- **52 — Building a REST API with raw http** — applies these header/method/status-code rules into a full manual router.
