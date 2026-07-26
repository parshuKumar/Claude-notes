# 37 — HTTP/2 in Node

## What is this?

HTTP/2 is a newer version of the HTTP protocol that lets a browser and server send **many requests and responses over a single TCP connection at the same time**, instead of one-at-a-time (or a few parallel connections) like HTTP/1.1. Node's built-in `http2` module lets you build servers and clients that speak this protocol directly. Think of HTTP/1.1 like a single-lane toll booth where cars queue up one behind another, while HTTP/2 is a multi-lane highway through the same tunnel — many cars (requests) travel at once without blocking each other.

---

## Why does it matter for backend development?

Backend developers care about HTTP/2 because it removes a real performance bottleneck: **head-of-line blocking** and the **6-connections-per-host limit** that browsers impose on HTTP/1.1. A page that needs 50 small API calls or assets no longer needs to wait for connections to free up — HTTP/2 multiplexes them all over one connection, cutting latency significantly on high-request-count pages and APIs. It also matters because most cloud load balancers, API gateways, and reverse proxies (Nginx, Cloudflare, AWS ALB) already speak HTTP/2 to clients and may speak HTTP/2 to your Node app too — so understanding it helps you configure servers correctly, debug protocol mismatches, and know when it is actually worth enabling versus when plain HTTP/1.1 (behind a proxy) is simpler and sufficient.

---

## Syntax / API

```js
// The http2 module is built into Node — no npm install needed
const http2 = require('http2');
const fs    = require('fs');

// ── Creating a secure (TLS) HTTP/2 server — the standard, browser-compatible way ──
const secureServer = http2.createSecureServer({
  key:  fs.readFileSync('server-key.pem'),   // private key for TLS
  cert: fs.readFileSync('server-cert.pem'),  // certificate for TLS
});

// 'stream' event fires once per incoming HTTP/2 stream (i.e. per request)
secureServer.on('stream', (stream, headers) => {
  // headers is a plain object — ':path', ':method' are HTTP/2 pseudo-headers
  const requestPath = headers[':path'];        // e.g. '/api/users'
  const method      = headers[':method'];      // e.g. 'GET'

  // stream.respond() sends response headers (like res.writeHead in HTTP/1.1)
  stream.respond({
    'content-type': 'application/json',
    ':status': 200,
  });

  // stream.end() sends the response body and closes the stream
  stream.end(JSON.stringify({ path: requestPath, method }));
});

secureServer.listen(8443);   // HTTP/2 over TLS conventionally uses 8443/443

// ── Creating a plaintext (h2c) HTTP/2 server — for server-to-server, no TLS ──
const plainServer = http2.createServer((req, res) => {
  // This Express-like (req, res) style also works via the compat API
  res.writeHead(200, { 'content-type': 'text/plain' });
  res.end('Hello over HTTP/2 without TLS (h2c)');
});

plainServer.listen(8080);   // browsers do NOT support h2c — only server-to-server clients do
```

---

## How it works — line by line

- `require('http2')` loads Node's built-in HTTP/2 implementation — it ships with Node itself, no package install required.
- `http2.createSecureServer({ key, cert })` creates a server that negotiates HTTP/2 over TLS using a mechanism called **ALPN** (Application-Layer Protocol Negotiation) — during the TLS handshake, the client and server agree to speak `h2` instead of `http/1.1`. This is the only way real browsers will use HTTP/2 with your server.
- The `'stream'` event replaces the familiar `'request'` event from HTTP/1.1. In HTTP/2, one TCP connection can carry many concurrent **streams**, and each stream is roughly equivalent to one request/response pair — so the event fires once per stream, not once per connection.
- `headers` arrives as a plain object where special protocol fields start with a colon (`:path`, `:method`, `:status`, `:scheme`, `:authority`) — these are called **pseudo-headers** and replace the request line and status line that HTTP/1.1 used as raw text.
- `stream.respond({...})` sends the response headers for that specific stream. Multiple streams on the same connection can each call `respond()` independently and their responses will interleave over the wire without waiting for each other.
- `stream.end(body)` writes the response body and finishes that stream — the underlying TCP connection stays open for other streams to keep using.
- `http2.createServer(callback)` (without TLS options) creates a plaintext HTTP/2 server, commonly called **h2c**. Browsers refuse to use h2c for security reasons, but it is common between internal microservices or behind a proxy that already terminates TLS.
- The optional **compatibility API** (`http2.createSecureServer({ ...options }, (req, res) => {...})`) gives you `req`/`res` objects that look almost identical to the classic `http` module, making migration from HTTP/1.1 code much easier.

---

## Example 1 — basic

```js
// File: src/http2-basic-server.js
// A minimal HTTP/2 server using self-signed certs for local testing

const http2 = require('http2');   // Node's built-in HTTP/2 module
const fs    = require('fs');      // to read the TLS key/cert files

// Load a self-signed key/cert pair (generate with openssl for local dev)
const serverOptions = {
  key:  fs.readFileSync('./certs/localhost-key.pem'),   // private key
  cert: fs.readFileSync('./certs/localhost-cert.pem'),  // public certificate
};

// Create the secure HTTP/2 server
const server = http2.createSecureServer(serverOptions);

// Fires once per incoming request (technically once per HTTP/2 "stream")
server.on('stream', (stream, headers) => {
  const requestPath = headers[':path'];   // extract the requested URL path

  // Log every incoming stream — useful while learning how multiplexing looks
  console.log(`Incoming stream for: ${requestPath}`);

  // Send response headers with a 200 status and JSON content type
  stream.respond({
    ':status': 200,
    'content-type': 'application/json',
  });

  // Send the body and end the stream
  stream.end(JSON.stringify({ message: 'Hello over HTTP/2', path: requestPath }));
});

// Listen on port 8443 — the conventional port for HTTP/2 dev servers
server.listen(8443, () => {
  console.log('HTTP/2 server running at https://localhost:8443');
});
```

---

## Example 2 — real world backend use case

```js
// File: src/http2-api-server.js
// A small HTTP/2 API using the compatibility layer (req/res) so existing
// route logic barely has to change — common when upgrading an Express-style API.

const http2 = require('http2');   // built-in HTTP/2 module
const fs    = require('fs');      // to load TLS certs

// In-memory "database" for this example — a real app would query Postgres/Mongo
const usersById = {
  user_1: { userId: 'user_1', name: 'Asha Verma', role: 'admin' },
  user_2: { userId: 'user_2', name: 'Ravi Shah',  role: 'member' },
};

const serverOptions = {
  key:              fs.readFileSync('./certs/localhost-key.pem'),   // TLS private key
  cert:             fs.readFileSync('./certs/localhost-cert.pem'),  // TLS certificate
  allowHTTP1:       true,   // fall back to HTTP/1.1 for old clients that can't negotiate h2
};

// createSecureServer with a (req, res) callback = the compatibility API
const server = http2.createSecureServer(serverOptions, (req, res) => {
  const authToken = req.headers['authorization'];   // read auth header, like HTTP/1.1

  // Simple auth gate — reject requests missing a bearer token
  if (!authToken || !authToken.startsWith('Bearer ')) {
    res.writeHead(401, { 'content-type': 'application/json' });
    return res.end(JSON.stringify({ error: 'Missing or invalid authToken' }));
  }

  // Very small manual router based on method + path
  if (req.method === 'GET' && req.url.startsWith('/api/users/')) {
    const userId = req.url.split('/').pop();     // extract userId from the path
    const user   = usersById[userId];             // look up the user record

    if (!user) {
      res.writeHead(404, { 'content-type': 'application/json' });
      return res.end(JSON.stringify({ error: `User ${userId} not found` }));
    }

    res.writeHead(200, { 'content-type': 'application/json' });
    return res.end(JSON.stringify(user));
  }

  // Fallback for unmatched routes
  res.writeHead(404, { 'content-type': 'application/json' });
  res.end(JSON.stringify({ error: 'Route not found' }));
});

// Start listening — multiple clients can now open many concurrent
// requests to this server over just one TCP+TLS connection each
server.listen(8443, () => {
  console.log('HTTP/2 API server listening on https://localhost:8443');
});
```

---

## Common mistakes

### Mistake 1 — Using the raw `stream` API but writing HTTP/1.1-style headers

```js
// ❌ WRONG — the raw HTTP/2 stream API rejects HTTP/1.1-style status lines
// and forbids setting the 'connection' header (illegal in HTTP/2)
stream.respond({
  'Status': '200 OK',        // wrong key — must be ':status' with a number
  'connection': 'keep-alive', // throws — HTTP/2 has no per-request connection header
});

// ✅ CORRECT — use the ':status' pseudo-header with a numeric value,
// and never send 'connection', 'keep-alive', or 'transfer-encoding'
stream.respond({
  ':status': 200,             // numeric HTTP status code
  'content-type': 'application/json',
});
```

### Mistake 2 — Assuming HTTP/2 works without TLS for browser clients

```js
// ❌ WRONG — createServer() without TLS makes an h2c (plaintext) server,
// but browsers refuse to speak HTTP/2 without TLS, so this silently
// falls back or fails when a browser connects
const server = http2.createServer((req, res) => {
  res.end('This will not work from a browser tab');
});
server.listen(3000);

// ✅ CORRECT — use createSecureServer with real (or dev) TLS certs
// for anything a browser needs to reach
const server = http2.createSecureServer(
  { key: fs.readFileSync('key.pem'), cert: fs.readFileSync('cert.pem') },
  (req, res) => res.end('Works from a browser over https://')
);
server.listen(8443);
```

### Mistake 3 — Forgetting `allowHTTP1` and breaking clients that can't negotiate HTTP/2

```js
// ❌ WRONG — some load balancers, health-check tools, or older clients
// cannot negotiate HTTP/2 over TLS and the connection just fails
const server = http2.createSecureServer({ key, cert }, (req, res) => {
  res.end('API response');
});
// No fallback configured — non-HTTP/2 clients get connection errors

// ✅ CORRECT — set allowHTTP1: true so the server negotiates HTTP/1.1
// automatically for clients that request it, while still using HTTP/2
// for clients that support it
const server = http2.createSecureServer(
  { key, cert, allowHTTP1: true },
  (req, res) => res.end('API response')
);
```

---

## Practice exercises

### Exercise 1 — easy

Generate a self-signed TLS certificate for `localhost` (search "openssl self signed cert localhost" if needed), then build a basic `http2.createSecureServer` that:
1. Listens on port `8443`
2. Handles the `'stream'` event directly (not the compatibility API)
3. Responds to every request with a JSON body containing the requested `:path` and `:method`
4. Logs each incoming stream's path to the console when it arrives

```js
// Write your code here
```

---

### Exercise 2 — medium

Build an HTTP/2 API server using the compatibility API (`req`/`res` style) that manages an in-memory list of `product` objects (`{ productId, name, price }`). It must support:
1. `GET /api/products` — returns the full list as JSON
2. `GET /api/products/:productId` — returns a single product or a 404 JSON error if not found
3. `POST /api/products` — reads the JSON request body and adds a new product to the in-memory list
4. Set `allowHTTP1: true` so the server still works for clients that cannot negotiate HTTP/2

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a small load-comparison tool that demonstrates HTTP/2 multiplexing:
1. Create an HTTP/2 server (with TLS) that has an artificial `setTimeout` delay of 200ms before responding to any request, to simulate slow work
2. Create an HTTP/2 **client** (using `http2.connect()`) that opens a **single connection** to the server and fires 20 concurrent requests over that one connection using `client.request()`
3. Measure and log the total time taken for all 20 requests to complete
4. Compare it against making the same 20 requests sequentially, one after another, over the same connection, and log that total time too
5. Print a summary showing which approach was faster and by roughly how much

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
WHAT HTTP/2 SOLVES
  HTTP/1.1: one request per connection at a time (or up to ~6 parallel
            connections per host) → head-of-line blocking
  HTTP/2:   many requests ("streams") multiplexed over ONE TCP connection
            → no blocking between unrelated requests

CREATING SERVERS
  http2.createSecureServer({ key, cert })   → HTTP/2 over TLS (browser-compatible)
  http2.createServer()                      → plaintext h2c (server-to-server only)
  { allowHTTP1: true }                      → falls back to HTTP/1.1 for old clients

TWO API STYLES
  Raw stream API     → server.on('stream', (stream, headers) => {...})
                        headers[':path'], headers[':method']
                        stream.respond({ ':status': 200, ... })
                        stream.end(body)
  Compatibility API  → createSecureServer(options, (req, res) => {...})
                        req.url, req.method, req.headers — same as HTTP/1.1
                        res.writeHead(), res.end() — same as HTTP/1.1

PSEUDO-HEADERS (HTTP/2 only, replace the old request/status line)
  :method     → GET, POST, etc.
  :path       → /api/users/user_1
  :scheme     → https
  :authority  → host:port (replaces the 'Host' header)
  :status     → response status code (server responses only)

FORBIDDEN HEADERS IN HTTP/2
  connection, keep-alive, transfer-encoding, upgrade — all illegal;
  Node throws if you try to set them via stream.respond()

SERVER PUSH
  stream.pushStream() lets a server proactively send extra resources
  (e.g. a CSS file) before the client even asks for them.
  Deprecated in Chrome and most browsers as of 2022 — rarely used now;
  know it exists, but do not rely on it for new projects.

WHEN TO ACTUALLY USE HTTP/2
  ✓ Public APIs / websites with many small concurrent requests per page
  ✓ Already behind Nginx/Cloudflare/ALB that terminates HTTP/2 for you
  ✗ Simple internal service with few requests — added complexity, little gain
  ✗ Already fronted by a proxy speaking HTTP/1.1 to your Node app — no benefit
    configuring http2 module directly, since the proxy already handles it

CLIENT SIDE
  const client = http2.connect('https://example.com');
  const req = client.request({ ':path': '/api/data' });
  req.on('response', (headers) => {...});
  req.on('data', (chunk) => {...});
  req.end();
```

---

## Connected topics

- **20 — http module** — the HTTP/1.1 foundation; HTTP/2 solves the head-of-line blocking and connection-limit problems this module's protocol has
- **21 — https module** — HTTP/2 in Node almost always runs over TLS, so understanding `https` certificates and TLS setup is required before enabling `http2.createSecureServer`
- **36 — HTTP deep dive: headers, methods, status codes** — HTTP/2 keeps the same methods and status codes but reshapes them into pseudo-headers (`:method`, `:status`) instead of a text request/status line
