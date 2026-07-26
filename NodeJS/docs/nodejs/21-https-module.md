# 21 — https module

## What is this?

The `https` module is Node's built-in tool for creating HTTP servers (and making HTTP requests) that run over an **encrypted** connection using TLS/SSL. It has almost the exact same API as the `http` module — same `req`/`res` objects, same `.listen()`, same everything — except the server needs a certificate and private key to prove its identity and encrypt traffic. Think of `http` as sending a postcard anyone on the network can read, and `https` as sending the same message sealed in a locked, tamper-proof envelope only the intended recipient can open.

## Why does it matter for backend development?

Any API that handles passwords, tokens, payment info, or personal data must never travel as plain text — `https` is what prevents attackers on the network (public wifi, compromised routers, ISPs) from reading or modifying requests in transit. In real production systems, most backend developers do **not** terminate TLS directly in Node — a reverse proxy (Nginx, AWS ALB, Cloudflare) handles the certificate and forwards plain `http` internally. But you still need to understand `https` deeply: for local development with real certs, for internal service-to-service encryption, for tools that must be self-contained (no proxy in front), and to reason correctly about where encryption actually terminates in your infrastructure.

---

## Syntax / API

```js
// Import Node's built-in https module — no npm install needed
const https = require('https');

// Import fs to read the certificate and private key files from disk
const fs = require('fs');

// TLS options: the server's identity and encryption key
const tlsOptions = {
  // The private key — must be kept secret, never committed to git
  key: fs.readFileSync('server-key.pem'),

  // The certificate — proves the server's identity to connecting clients
  cert: fs.readFileSync('server-cert.pem'),
};

// Create an HTTPS server — same request handler signature as http.createServer
const server = https.createServer(tlsOptions, (req, res) => {
  // req → incoming request (method, url, headers) — identical shape to http module
  // res → outgoing response object — identical shape to http module
  res.writeHead(200, { 'Content-Type': 'text/plain' }); // set status + headers
  res.end('Secure response over TLS\n');                // send body and close response
});

// Listen on port 443 (standard HTTPS port) or any port during development
server.listen(443, () => {
  console.log('HTTPS server running on port 443'); // confirms server started
});

// ── Making an outbound HTTPS request (client side) ──────────────────────────
https.get('https://api.example.com/users', (res) => {
  let data = '';                          // accumulate response chunks
  res.on('data', (chunk) => { data += chunk; }); // append each chunk as it arrives
  res.on('end', () => { console.log(JSON.parse(data)); }); // parse once fully received
});
```

---

## How it works — line by line

`https.createServer()` takes one extra argument compared to `http.createServer()` — an options object containing the server's private key and certificate. Everything after that (the request handler, `req`/`res`, `.listen()`) behaves identically to the plain `http` module because `https` is built directly on top of it, adding a TLS encryption layer underneath.

- **`key`** is the server's private key — a secret used to decrypt data encrypted with the matching public key. If this leaks, anyone can impersonate your server.
- **`cert`** is the public certificate — it contains the server's public key plus identity information (domain name, issuer, expiry), signed by a Certificate Authority (CA) so browsers and clients can trust it.
- When a client connects, a **TLS handshake** happens first: client and server agree on encryption algorithms, the server proves its identity by presenting the certificate, and both sides derive a shared secret key used to encrypt all further traffic. This happens before any HTTP request/response is exchanged.
- Once the handshake completes, the connection behaves exactly like a normal HTTP connection — just wrapped in encryption that neither the client nor the server has to think about after setup.
- For outbound requests, `https.get()` / `https.request()` do the same handshake as a client — verifying the server's certificate is valid and signed by a CA the system trusts, rejecting the connection if not (unless you explicitly disable verification, which you should never do in production).

---

## Example 1 — basic

```js
// File: src/https-basic-server.js
// A minimal HTTPS server using a self-signed certificate for local development.

const https = require('https'); // core module for encrypted HTTP servers
const fs    = require('fs');    // to read the key and cert files from disk

// Self-signed cert + key generated locally for dev only (see Exercise 2 for how)
const tlsOptions = {
  key:  fs.readFileSync('./certs/dev-key.pem'),  // private key — dev only, never commit
  cert: fs.readFileSync('./certs/dev-cert.pem'), // matching self-signed certificate
};

// Create the server — request handler is identical to the http module
const server = https.createServer(tlsOptions, (req, res) => {
  console.log(`${req.method} ${req.url}`);       // log incoming request line

  res.writeHead(200, { 'Content-Type': 'application/json' }); // JSON response
  res.end(JSON.stringify({ message: 'Hello over TLS', secure: true }));
});

// Bind to port 8443 — common convention for non-privileged HTTPS dev ports
server.listen(8443, () => {
  console.log('Dev HTTPS server: https://localhost:8443'); // confirms startup
});

// Browsers will show a "not trusted" warning because this cert is self-signed —
// that is expected and fine for local development, never for production.
```

---

## Example 2 — real world backend use case

```js
// File: src/server.js
// A production-style pattern: HTTPS enabled conditionally, redirecting http → https,
// with certs loaded from environment-driven paths (common in Docker/VM deployments).

const http  = require('http');
const https = require('https');
const fs    = require('fs');

const requestHandler = (req, res) => {
  // Shared handler used by both the http redirect server and the real https server
  res.writeHead(200, { 'Content-Type': 'application/json' });
  res.end(JSON.stringify({ status: 'ok', securedBy: 'TLS' }));
};

const useHttps = process.env.USE_HTTPS === 'true'; // toggle via environment variable

if (useHttps) {
  // Load certs from paths supplied by the deployment environment (e.g. Let's Encrypt)
  const tlsOptions = {
    key:  fs.readFileSync(process.env.TLS_KEY_PATH),   // e.g. /etc/letsencrypt/live/api.example.com/privkey.pem
    cert: fs.readFileSync(process.env.TLS_CERT_PATH),  // e.g. .../fullchain.pem
  };

  // Real HTTPS server — this is what actually terminates TLS if no proxy does it
  https.createServer(tlsOptions, requestHandler).listen(443, () => {
    console.log('HTTPS server listening on 443');
  });

  // A tiny plain-http server whose only job is to redirect to https
  http.createServer((req, res) => {
    const secureUrl = `https://${req.headers.host}${req.url}`; // rebuild as https URL
    res.writeHead(301, { Location: secureUrl });                // permanent redirect
    res.end();
  }).listen(80, () => {
    console.log('HTTP → HTTPS redirect server listening on 80');
  });
} else {
  // In most real deployments this branch runs instead — a reverse proxy (Nginx, ALB,
  // Cloudflare) already terminated TLS and forwards plain http to this Node process.
  http.createServer(requestHandler).listen(process.env.PORT || 3000, () => {
    console.log(`Plain HTTP server listening on ${process.env.PORT || 3000} (TLS terminated upstream)`);
  });
}
```

---

## Common mistakes

### Mistake 1 — Disabling certificate verification to silence errors

```js
// ❌ WRONG — turning off cert validation on the client "fixes" the error
// but opens the door to man-in-the-middle attacks on every request
const https = require('https');

https.get(
  'https://internal-api.example.com/data',
  { rejectUnauthorized: false }, // NEVER do this — accepts ANY certificate, even fake ones
  (res) => { /* ... */ }
);

// ✅ CORRECT — fix the actual problem: install the proper CA cert, or use a valid
// publicly-trusted certificate on the server so the default verification passes
const agent = new https.Agent({
  ca: require('fs').readFileSync('./certs/internal-ca.pem'), // trust your internal CA specifically
});
https.get('https://internal-api.example.com/data', { agent }, (res) => { /* ... */ });
```

### Mistake 2 — Terminating TLS in Node when a proxy already does it

```js
// ❌ WRONG — running an https.createServer() behind an Nginx/ALB that ALREADY
// terminates TLS means Node is doing pointless double encryption work,
// and cert renewal has to be handled in two places instead of one
const https = require('https');
const server = https.createServer(tlsOptions, requestHandler).listen(443);
// The proxy in front of this decrypts, then re-encrypts to reach Node — wasted CPU
// and now you must renew/rotate certs on the Node box too.

// ✅ CORRECT — let the proxy terminate TLS, Node just serves plain http internally
const http = require('http');
const server = http.createServer(requestHandler).listen(3000);
// Nginx/ALB handles :443 and the certificate, forwards decrypted traffic to :3000
// req.headers['x-forwarded-proto'] tells Node the original request was https
```

### Mistake 3 — Committing private keys or certs to git

```js
// ❌ WRONG — hardcoding or checking in the key/cert files
const tlsOptions = {
  key:  fs.readFileSync('./key.pem'),   // if this path is inside a committed repo folder,
  cert: fs.readFileSync('./cert.pem'),  // the private key is now in git history forever
};
// Even deleting the file later does not remove it from git history — it is compromised.

// ✅ CORRECT — load paths from environment variables, keep actual files out of git
// .gitignore: certs/*.pem
const tlsOptions = {
  key:  fs.readFileSync(process.env.TLS_KEY_PATH),  // path injected at deploy time
  cert: fs.readFileSync(process.env.TLS_CERT_PATH), // e.g. mounted secret volume in Docker/K8s
};
```

---

## Practice exercises

### Exercise 1 — easy

Using OpenSSL (run this in your terminal, not in Node), generate a self-signed certificate and key for local development:

```
openssl req -nodes -new -x509 -keyout dev-key.pem -out dev-cert.pem -days 365
```

Then write a Node script that:
1. Creates an `https.createServer()` using that key and cert
2. Responds to any request with a JSON body containing your `userId` and the current timestamp
3. Listens on port `8443` and logs a startup message with the full URL to visit

```js
// Write your code here
```

---

### Exercise 2 — medium

Write a script `src/tls-agent-request.js` that:
1. Uses `https.request()` (not `https.get()`) to make a `POST` request to `https://localhost:8443/orders` (pointing at your own server from Exercise 1, or any test HTTPS endpoint you control)
2. Sends a JSON body containing `{ userId, itemId, quantity }`
3. Sets the correct `Content-Type` and `Content-Length` headers
4. Uses a custom `https.Agent` that trusts your self-signed dev certificate via the `ca` option (instead of disabling verification with `rejectUnauthorized: false`)
5. Collects the response body via the `data` and `end` events and logs the parsed JSON

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a small `src/dual-protocol-server.js` module that:
1. Reads an environment variable `TLS_MODE` which can be `"native"`, `"proxied"`, or `"redirect"`
2. In `"native"` mode: starts a real `https.createServer()` on port 8443 using local self-signed certs, and rejects any plain `http` connection attempt on that same port
3. In `"proxied"` mode: starts a plain `http.createServer()` on port 3000, and includes a middleware-style check that reads `req.headers['x-forwarded-proto']` — if it's not `"https"`, respond with `403 Forbidden` and a JSON error explaining the request must arrive via a TLS-terminating proxy
4. In `"redirect"` mode: starts both a `https.createServer()` on 8443 AND a `http.createServer()` on 3000 where the http server 301-redirects every request to the https equivalent URL
5. Exports a `startServer()` function that reads `TLS_MODE` and starts the correct configuration, returning the created server instance(s)
6. Add clear `console.log` statements so running the script in each mode shows exactly what's happening

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
WHAT IT IS
  https module   → same API as http module, adds TLS/SSL encryption underneath
  TLS/SSL        → protocol that encrypts traffic + verifies server identity via certificates

CORE OPTIONS FOR https.createServer(options, handler)
  key   → server's private key (PEM format) — NEVER commit to git, NEVER share
  cert  → server's public certificate — proves identity, signed by a CA (or self-signed for dev)
  ca    → (client-side) trusted CA certificates for verifying servers with custom/internal CAs

SERVER METHODS (identical to http module)
  server.listen(port, callback)   → start listening
  req.method / req.url / req.headers  → same as http
  res.writeHead() / res.write() / res.end()  → same as http

CLIENT METHODS
  https.get(url, callback)             → simple GET request
  https.request(options, callback)     → full control (method, headers, body)
  new https.Agent({ ca, rejectUnauthorized })  → custom trust config for a client

HTTP vs HTTPS
  http   → plain text, port 80, no identity verification, fast, unsafe over public networks
  https  → encrypted, port 443, verified server identity, TLS handshake adds one round trip

WHEN TO USE https DIRECTLY IN NODE
  - Local dev needing real TLS behavior (self-signed certs)
  - No reverse proxy in front (standalone tools, internal services, some serverless setups)
  - Service-to-service encryption inside a private network with its own CA

WHEN TO LET A PROXY TERMINATE TLS INSTEAD (most production APIs)
  - Nginx / AWS ALB / Cloudflare handles the certificate and renewal
  - Node receives plain http, checks req.headers['x-forwarded-proto'] to know origin was https
  - Simpler cert rotation (one place), lower Node CPU usage, easier horizontal scaling

NEVER DO
  { rejectUnauthorized: false }   → disables cert verification, opens MITM risk
  committing .pem key/cert files to git
  running https.createServer() redundantly behind a proxy that already terminates TLS

COMMON PATTERNS
  Self-signed cert for dev:
    openssl req -nodes -new -x509 -keyout key.pem -out cert.pem -days 365

  http → https redirect server:
    http.createServer((req,res)=>{ res.writeHead(301,{Location:`https://${req.headers.host}${req.url}`}); res.end(); })

  Trust internal CA on client:
    new https.Agent({ ca: fs.readFileSync('./internal-ca.pem') })
```

---

## Connected topics

- **20 — http module** — `https` is built directly on top of `http`; the request/response API, routing, and body handling are identical, only the transport is encrypted
- **80 — HTTPS, TLS, certificates** — goes deeper into certificate chains, Let's Encrypt, and the tradeoffs of terminating TLS at the load balancer vs inside Node itself
- **109 — Reverse proxy with Nginx** — the most common production pattern: Nginx terminates TLS and forwards plain `http` to Node, which is why understanding both sides matters
