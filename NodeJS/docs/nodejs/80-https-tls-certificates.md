# 80 — HTTPS, TLS, certificates

## What is this?

HTTPS is plain HTTP wrapped inside TLS (Transport Layer Security) — an encryption layer that scrambles traffic between client and server so nobody in between can read or tamper with it. A TLS certificate is the piece of paper that proves "this server really is api.myapp.com" — issued and digitally signed by a trusted authority. Think of a certificate like a notarized ID card: the browser doesn't just trust your claim that you're the real server, it checks that a trusted notary (a Certificate Authority like Let's Encrypt) vouched for you.

## Why does it matter for backend development?

Every production API, login form, and payment endpoint must run over HTTPS — browsers now flag plain HTTP as "Not Secure," mobile apps refuse plaintext connections by default (App Transport Security / Network Security Config), and any data sent over HTTP (passwords, tokens, cookies) can be read or modified by anyone on the network path. A backend developer needs to know how to generate a self-signed certificate for local development, request a free trusted certificate from Let's Encrypt for production, and — critically — understand that in most real deployments Node itself never terminates TLS at all; a load balancer, reverse proxy, or CDN (Nginx, AWS ALB, Cloudflare) decrypts HTTPS traffic and forwards plain HTTP to Node internally. Knowing which model you're in changes how you configure your server and how you reason about security.

---

## Syntax / API

```js
// The built-in https module mirrors the http module's API, but requires TLS credentials
const https = require('https');
const fs    = require('fs');

// TLS options — the certificate + private key pair that proves server identity
const tlsOptions = {
  key:  fs.readFileSync('./certs/server.key'),   // private key — NEVER commit this to git
  cert: fs.readFileSync('./certs/server.crt'),   // public certificate — signed proof of identity
  // ca: fs.readFileSync('./certs/ca-bundle.crt'), // optional: intermediate CA chain (needed for some CAs)
};

// Create an HTTPS server — same request handler signature as http.createServer
const server = https.createServer(tlsOptions, (req, res) => {
  res.writeHead(200, { 'Content-Type': 'application/json' }); // set headers as usual
  res.end(JSON.stringify({ message: 'Encrypted response' }));  // send response body
});

// Listen on the standard HTTPS port (443 in prod, often 8443 or 4000 in local dev)
server.listen(4000, () => {
  console.log('HTTPS server running on https://localhost:4000'); // confirm startup
});
```

---

## How it works — line by line

1. **`require('https')`** — Node's built-in module for creating TLS-encrypted HTTP servers. It has the exact same API shape as `http`, but its `createServer()` needs credentials.
2. **`tlsOptions.key`** — the server's private key file. This is the secret half of a public/private key pair and must never leave the server or be committed to source control.
3. **`tlsOptions.cert`** — the server's public certificate. It contains the server's public key plus a digital signature from a Certificate Authority (CA) confirming the domain ownership.
4. **`https.createServer(tlsOptions, handler)`** — during the TLS handshake, the server presents `cert` to the connecting client. The client's browser/HTTP library checks the signature against a list of CAs it already trusts (baked into the OS or browser).
5. **The request handler function** — runs exactly like a plain `http` handler; by the time your code sees `req`, TLS decryption has already happened. Your application code never touches encrypted bytes directly.
6. **`server.listen(4000, ...)`** — starts listening for encrypted connections on port 4000; a client connecting here performs a TLS handshake first, then sends the HTTP request over the now-encrypted channel.

---

## Example 1 — basic

```js
// File: src/dev-server.js
// A self-signed HTTPS server for local development only.
// Browsers will show a security warning because no public CA signed this cert —
// that is expected and fine for localhost testing.

const https = require('https');
const fs    = require('fs');
const path  = require('path');

// Load the self-signed key/cert generated with openssl (see cheat sheet below)
const tlsOptions = {
  key:  fs.readFileSync(path.join(__dirname, '..', 'certs', 'localhost-key.pem')),
  cert: fs.readFileSync(path.join(__dirname, '..', 'certs', 'localhost-cert.pem')),
};

// Simple request handler — no framework needed to demonstrate TLS itself
const server = https.createServer(tlsOptions, (req, res) => {
  res.writeHead(200, { 'Content-Type': 'text/plain' }); // plain text response
  res.end(`Secure hello over ${req.socket.encrypted ? 'TLS' : 'plain TCP'}\n`); // confirm encryption
});

// Bind to a local dev port (443 requires root/admin privileges, so avoid it locally)
server.listen(4443, () => {
  console.log('Dev HTTPS server: https://localhost:4443'); // → accept the browser warning to test
});
```

---

## Example 2 — real world backend use case

```js
// File: src/server.js
// A realistic pattern: Node serves plain HTTP, TLS is terminated upstream.
// This is how most production Node apps behind Nginx / AWS ALB / Cloudflare actually run —
// Node never sees a certificate at all; it just trusts a header from the proxy.

const http    = require('http');
const express = require('express');

const app = express();

// Trust the reverse proxy (Nginx, ALB, etc.) so req.secure and req.protocol are accurate
app.set('trust proxy', 1); // 1 = trust the first hop (the load balancer) — critical for correctness

// Middleware: force HTTPS by checking the header the proxy sets after terminating TLS
function requireHttps(req, res, next) {
  // 'x-forwarded-proto' is set by the proxy to 'https' once it decrypted the real request
  if (req.headers['x-forwarded-proto'] !== 'https' && process.env.NODE_ENV === 'production') {
    // Redirect plain HTTP requests to the HTTPS version of the same URL
    return res.redirect(301, `https://${req.headers.host}${req.originalUrl}`);
  }
  next(); // already secure (or running locally) — continue to the next middleware
}

app.use(requireHttps); // apply the redirect check to every route

app.get('/api/profile', (req, res) => {
  // req.secure now correctly reflects the ORIGINAL client connection, not the internal hop
  res.json({ userId: req.query.userId, secure: req.secure, protocol: req.protocol });
});

// Node listens on plain HTTP internally — port 3000, never exposed directly to the internet
const PORT = process.env.PORT || 3000;
http.createServer(app).listen(PORT, () => {
  console.log(`App server listening on http://127.0.0.1:${PORT} (TLS terminated upstream)`);
});

// Nginx (or ALB) in front of this handles:
//   1. Accepting HTTPS on port 443 with the real Let's Encrypt certificate
//   2. Decrypting traffic
//   3. Forwarding plain HTTP to Node on 127.0.0.1:3000
//   4. Setting X-Forwarded-Proto: https and X-Forwarded-For: <real client IP>
```

---

## Common mistakes

### Mistake 1 — Committing private keys or certificates to git

```js
// ❌ WRONG — server.key checked into the repo; anyone with repo access can impersonate your server
// project structure:
//   certs/server.key   ← committed by accident
//   certs/server.crt

const tlsOptions = {
  key:  fs.readFileSync('./certs/server.key'),
  cert: fs.readFileSync('./certs/server.crt'),
};
// If this leaks, an attacker can decrypt traffic or run a fake server as "you"

// ✅ CORRECT — keep keys out of git and load them from a secrets path or env-mounted volume
// .gitignore:
//   certs/*.key
//   certs/*.pem

const tlsOptions = {
  key:  fs.readFileSync(process.env.TLS_KEY_PATH),   // path injected by deployment platform
  cert: fs.readFileSync(process.env.TLS_CERT_PATH),  // e.g. mounted secret volume in Kubernetes
};
```

### Mistake 2 — Using self-signed certificates in production

```js
// ❌ WRONG — self-signed cert deployed to a real public domain
// Browsers show "Your connection is not private" to every real user, killing trust
const tlsOptions = {
  key:  fs.readFileSync('./certs/self-signed-key.pem'),
  cert: fs.readFileSync('./certs/self-signed-cert.pem'),
};
https.createServer(tlsOptions, app).listen(443);

// ✅ CORRECT — use a free, trusted certificate from Let's Encrypt (via Certbot / ACME)
// Certbot auto-renews and gives you a cert signed by a CA every browser already trusts:
//   sudo certbot certonly --standalone -d api.myapp.com
// Then load the CA-signed files it produces (usually via a reverse proxy, not Node directly):
const tlsOptions = {
  key:  fs.readFileSync('/etc/letsencrypt/live/api.myapp.com/privkey.pem'),
  cert: fs.readFileSync('/etc/letsencrypt/live/api.myapp.com/fullchain.pem'), // includes CA chain
};
```

### Mistake 3 — Disabling certificate validation to "fix" errors

```js
// ❌ WRONG — silences a real security check instead of fixing the underlying cert problem
// Common when an outgoing HTTPS request to another API fails with a cert error
process.env.NODE_TLS_REJECT_UNAUTHORIZED = '0'; // disables ALL TLS verification, globally
const response = await fetch('https://payments-provider.example.com/charge', {
  method: 'POST',
  body: JSON.stringify({ amount: 500, userId: 'user_42' }),
});
// Now your app accepts ANY certificate, including one from an attacker doing a man-in-the-middle attack

// ✅ CORRECT — fix the actual cause (expired cert, missing CA bundle, wrong hostname)
// and never disable verification outside of a throwaway local script
const https = require('https');
const agent = new https.Agent({
  ca: fs.readFileSync('./certs/internal-ca-bundle.crt'), // trust your org's internal CA explicitly
});
const response = await fetch('https://payments-provider.example.com/charge', {
  method: 'POST',
  agent, // supply the correct trust chain instead of turning off checks
  body: JSON.stringify({ amount: 500, userId: 'user_42' }),
});
```

---

## Practice exercises

### Exercise 1 — easy

Generate a self-signed certificate locally using OpenSSL (or find one already generated), then write a Node script that:
1. Starts an `https.createServer()` on port `4443` using that key/cert pair
2. Has one route that responds with JSON `{ secure: true, path: req.url }`
3. Logs a message on startup showing the URL to visit
4. Logs `req.socket.encrypted` for each incoming request to confirm the connection is actually TLS

```js
// Write your code here
```

---

### Exercise 2 — medium

Build a small Express app with two behaviors controlled by `process.env.NODE_ENV`:
1. In `development`, the app starts an HTTPS server directly using a local self-signed cert (paths read from env vars `TLS_KEY_PATH` and `TLS_CERT_PATH`)
2. In `production`, the app starts a plain `http.createServer()` on `process.env.PORT`, assuming TLS is terminated by a reverse proxy in front of it
3. Add `app.set('trust proxy', 1)` and a middleware that reads `x-forwarded-proto` to redirect any non-HTTPS request to HTTPS when in production
4. Add a `/whoami` route that returns `{ protocol: req.protocol, secure: req.secure, host: req.headers.host }`

```js
// Write your code here
```

---

### Exercise 3 — hard

Design (in code, as comments/pseudocode plus real Node where applicable) a `CertificateWatcher` module that:
1. Accepts a directory path where Let's Encrypt style certs live (`privkey.pem`, `fullchain.pem`)
2. Loads the cert/key pair into an `https.Server` created with `https.createServer(options, app)`
3. Uses `fs.watch()` on the cert directory to detect when Certbot renews the certificate (files get rewritten)
4. On change, reloads the new key/cert into the *running* server without dropping active connections — research and use `server.setSecureContext(newOptions)` (available on Node's `tls.Server`, which `https.Server` extends)
5. Logs every renewal event with a timestamp and the certificate's new expiry date (hint: use the `crypto` or a cert-parsing approach, or simply log that a reload occurred if full parsing is out of scope)
6. Exposes a `getCertExpiry()` method for a health check endpoint to report how many days remain before expiry

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
WHAT HTTPS/TLS ACTUALLY IS
  HTTP           → plain text over TCP, anyone on the network path can read/modify it
  HTTPS          → HTTP + TLS encryption layer — same HTTP semantics, encrypted transport
  Certificate    → signed proof that a public key belongs to a specific domain
  CA             → Certificate Authority — trusted third party that signs certificates
  Handshake      → client + server negotiate keys and verify the cert before any HTTP data flows

GENERATING A SELF-SIGNED CERT (LOCAL DEV ONLY)
  openssl req -x509 -newkey rsa:2048 -nodes \
    -keyout localhost-key.pem -out localhost-cert.pem \
    -days 365 -subj "/CN=localhost"
  → browsers WILL warn "not trusted" — expected, fine for dev only

LET'S ENCRYPT (FREE, TRUSTED, PRODUCTION-READY)
  Uses the ACME protocol to prove you control a domain, then issues a real trusted cert
  Certbot is the standard client:
    sudo certbot certonly --standalone -d api.myapp.com
    sudo certbot renew          → certs expire every 90 days, must auto-renew
  Output files:
    /etc/letsencrypt/live/<domain>/privkey.pem     → private key
    /etc/letsencrypt/live/<domain>/fullchain.pem   → cert + intermediate chain

SSL TERMINATION: WHERE DOES DECRYPTION HAPPEN?
  Option A — Terminate at Node (https.createServer with key/cert)
    + Full end-to-end encryption, no unencrypted hop anywhere
    - Node manages certs and renewal itself; more CPU spent on TLS in the app process
    - Rare in real production setups except very simple single-server deployments

  Option B — Terminate at a reverse proxy / load balancer (Nginx, AWS ALB, Cloudflare)  [MOST COMMON]
    + Proxy handles TLS, cert renewal, and HTTPS termination centrally
    + Node runs plain http.createServer() — simpler, faster, no cert management in app code
    + Proxy sets X-Forwarded-Proto / X-Forwarded-For headers for the app to read
    - Traffic between proxy and Node is unencrypted (acceptable inside a private network/VPC)
    - Must set app.set('trust proxy', 1) in Express or req.secure/req.ip will be wrong

NEVER DO
  Commit private keys (.key/.pem) to git
  Deploy self-signed certs to a public production domain
  Set NODE_TLS_REJECT_UNAUTHORIZED=0 to silence cert errors
  Forget 'trust proxy' when TLS terminates upstream — breaks req.secure, req.ip, redirects

COMMON PATTERNS
  Local dev HTTPS:        https.createServer({ key, cert }, app).listen(4443)
  Prod behind proxy:      http.createServer(app).listen(PORT)  // proxy handles TLS
  Force HTTPS redirect:   if (req.headers['x-forwarded-proto'] !== 'https') redirect
  Trust the proxy hop:    app.set('trust proxy', 1)
```

---

## Connected topics

- **21 — https module** — the full API surface (`https.createServer`, `https.Agent`, `https.request`) this topic's syntax section is built on
- **109 — Reverse proxy with Nginx** — the concrete config for terminating SSL at Nginx and forwarding plain HTTP to Node, the pattern shown in Example 2
- **78 — XSS, CSRF, Clickjacking** — HTTPS is a prerequisite for secure cookies (`Secure` flag, `SameSite`) covered alongside these browser-facing attacks
