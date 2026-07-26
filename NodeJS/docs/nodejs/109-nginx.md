# 109 — Reverse proxy with Nginx

## What is this?

Nginx is a lightweight, high-performance web server that is most often used in Node.js deployments as a **reverse proxy** — it sits in front of your Node process and forwards incoming requests to it, then sends the response back to the client. Think of Nginx as the receptionist at an office building: visitors (requests) never walk straight into an employee's cubicle (your Node process) — they check in at the front desk first, which handles the building's front door (port 80/443, SSL, security) and then routes them to the right person. Node.js focuses purely on running your application logic; Nginx handles everything around the edges of the network.

## Why does it matter for backend development?

Running `node server.js` directly on port 443 with a client-facing public IP works for a demo, but it is fragile and inefficient in production. Nginx terminates SSL/TLS so your Node code never touches certificates, serves static files (images, CSS, JS bundles) directly from disk without ever bothering the Node event loop, load-balances traffic across multiple Node processes or containers, and shields Node from slow clients, malformed requests, and basic DDoS patterns. Nearly every production Node.js deployment — whether on a bare VM, Docker, or Kubernetes — puts Nginx (or an equivalent like an ALB/Ingress controller) in front of the app. Understanding `proxy_pass` and SSL termination is a baseline expectation for any backend developer who ships real services.

---

## Syntax / API

```nginx
# File: /etc/nginx/sites-available/myapp.conf
# This is the Nginx configuration language — NOT JavaScript.
# Nginx reads config directives inside "blocks" delimited by { }

server {
    listen 80;                      # Listen for plain HTTP traffic on port 80
    server_name api.myapp.com;      # Only handle requests for this hostname

    location / {                              # Match every incoming request path
        proxy_pass http://localhost:3000;     # Forward the request to the Node process
        proxy_set_header Host $host;          # Preserve the original Host header for Node
        proxy_set_header X-Real-IP $remote_addr;          # Pass the client's real IP
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for; # Chain of proxy IPs
        proxy_set_header X-Forwarded-Proto $scheme;       # Tell Node if original request was http/https
    }
}
```

```js
// File: server.js
// Node.js side — the app has NO idea Nginx exists, it just listens on a local port

const http = require('http'); // built-in HTTP module (Topic 20)

const server = http.createServer((req, res) => {
  // req.headers['x-forwarded-for'] gives the REAL client IP, not Nginx's IP
  console.log('Client IP:', req.headers['x-forwarded-for']);
  res.end('Hello from Node behind Nginx');
});

// Bind only to localhost — Nginx is the only thing that should reach this port directly
server.listen(3000, '127.0.0.1', () => {
  console.log('Node listening on 127.0.0.1:3000');
});
```

---

## How it works — line by line

- `listen 80;` — tells Nginx to accept connections arriving on port 80 (plain HTTP), the port browsers use by default when a user types `http://` or just a domain name.
- `server_name api.myapp.com;` — Nginx can host many different sites on one machine; this line says "only apply this configuration block when the request's Host header matches this domain."
- `location / { ... }` — matches every request path (`/`, `/users`, `/api/v1/orders`, everything) and applies the rules inside the block to it. More specific `location` blocks (like `/static/`) can override this for certain paths.
- `proxy_pass http://localhost:3000;` — this is the core directive. It tells Nginx "do not answer this request yourself — forward it to the Node process listening on port 3000, wait for its response, then relay that response back to the original client." The client never talks to Node directly; it only ever sees Nginx.
- `proxy_set_header Host $host;` — by default, forwarded requests would show Node the internal address (`localhost:3000`) as the Host header. This line rewrites it back to the original domain the client actually requested, which many Node apps rely on for building absolute URLs or routing logic.
- `proxy_set_header X-Real-IP $remote_addr;` and `X-Forwarded-For` — without these, every request reaching Node would appear to come from Nginx's own IP address (127.0.0.1), because Nginx is technically the client making the request to Node. These headers preserve the original visitor's IP for logging, rate limiting, and geolocation.
- `proxy_set_header X-Forwarded-Proto $scheme;` — Node needs to know whether the original request was `http` or `https` even though the connection between Nginx and Node is always plain HTTP internally. Frameworks like Express use this header to correctly redirect or set secure cookies.
- On the Node side, `server.listen(3000, '127.0.0.1', ...)` — binding to `127.0.0.1` instead of `0.0.0.0` means the Node process cannot be reached from outside the machine at all, even by accident. Only Nginx, running on the same machine, can talk to it.

---

## Example 1 — basic

```nginx
# File: /etc/nginx/sites-available/simple-proxy.conf
# Minimal reverse proxy for a single Node app — no SSL, no static files yet

server {
    listen 80;                    # accept plain HTTP requests
    server_name myapp.local;      # domain this block responds to

    location / {
        proxy_pass http://127.0.0.1:4000; # forward everything to Node on port 4000
        proxy_http_version 1.1;           # use HTTP/1.1 so keep-alive connections work
        proxy_set_header Connection '';   # clear the Connection header to enable keep-alive
        proxy_set_header Host $host;      # forward the original domain to Node
    }
}

# After saving this file:
#   sudo ln -s /etc/nginx/sites-available/simple-proxy.conf /etc/nginx/sites-enabled/
#   sudo nginx -t              # test the config for syntax errors before reloading
#   sudo systemctl reload nginx  # apply the new config without dropping connections
```

```js
// File: server.js
// The Node app being proxied — completely unaware of Nginx in front of it

const http = require('http'); // core module, no external dependency needed

const PORT = 4000; // must match the port used in proxy_pass above

const server = http.createServer((req, res) => {
  res.writeHead(200, { 'Content-Type': 'text/plain' }); // simple plaintext response
  res.end(`Handled by Node on ${PORT}\n`);               // confirm which process answered
});

server.listen(PORT, '127.0.0.1', () => {
  console.log(`Node app ready on 127.0.0.1:${PORT}`); // only reachable via Nginx
});
```

---

## Example 2 — real world backend use case

```nginx
# File: /etc/nginx/sites-available/api.myapp.com.conf
# Production-style config: SSL termination, static file serving, and proxy to Node

# Block 1 — redirect all plain HTTP traffic to HTTPS
server {
    listen 80;                        # catch insecure requests
    server_name api.myapp.com;        # our production domain
    return 301 https://$host$request_uri; # permanently redirect to the https version
}

# Block 2 — the real HTTPS server
server {
    listen 443 ssl http2;                       # accept HTTPS with HTTP/2 support
    server_name api.myapp.com;                  # domain this cert and block belong to

    ssl_certificate     /etc/letsencrypt/live/api.myapp.com/fullchain.pem; # public cert
    ssl_certificate_key /etc/letsencrypt/live/api.myapp.com/privkey.pem;   # private key
    ssl_protocols TLSv1.2 TLSv1.3;              # disallow old, insecure TLS versions
    ssl_ciphers HIGH:!aNULL:!MD5;               # only allow strong encryption ciphers

    client_max_body_size 10m;                   # reject request bodies larger than 10MB

    # Serve static assets directly from disk — Node never sees these requests
    location /static/ {
        alias /var/www/myapp/public/;            # actual folder on disk holding assets
        expires 30d;                             # tell browsers to cache for 30 days
        add_header Cache-Control "public, immutable"; # skip revalidation while cached
    }

    # Everything else — forward to the Node.js API
    location / {
        proxy_pass http://127.0.0.1:3000;         # Node app listening locally
        proxy_http_version 1.1;                   # required for keep-alive and websockets
        proxy_set_header Upgrade $http_upgrade;    # forward websocket upgrade requests
        proxy_set_header Connection 'upgrade';     # required alongside Upgrade header
        proxy_set_header Host $host;               # preserve original domain
        proxy_set_header X-Real-IP $remote_addr;   # true client IP for logging
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for; # proxy chain
        proxy_set_header X-Forwarded-Proto https;  # tell Node the original request was https
        proxy_connect_timeout 5s;                  # fail fast if Node is unreachable
        proxy_read_timeout 30s;                    # give Node up to 30s to respond
    }
}
```

```js
// File: server.js
// Express app being served behind the config above — reads forwarded headers correctly

const express = require('express'); // popular Node.js web framework (Topic 53)
const app = express();

// Trust the Nginx proxy so Express reads X-Forwarded-* headers correctly
app.set('trust proxy', 1); // "1" means trust exactly one hop (our Nginx box)

app.get('/api/v1/health', (req, res) => {
  // req.secure now correctly reports true because trust proxy reads X-Forwarded-Proto
  res.json({
    status: 'ok',
    secure: req.secure,           // true — Nginx told Express the real request was https
    clientIp: req.ip,             // real visitor IP, not Nginx's internal IP
  });
});

app.get('/api/v1/users/:userId', (req, res) => {
  const userId = req.params.userId;       // path param extracted by Express
  const authToken = req.headers['authorization']; // client's bearer token, if any

  if (!authToken) {
    return res.status(401).json({ error: 'Missing auth token' }); // reject unauthenticated calls
  }

  res.json({ userId, message: 'Fetched via Nginx reverse proxy' }); // normal success response
});

// Bind ONLY to localhost — Nginx is the sole entry point from the outside world
app.listen(3000, '127.0.0.1', () => {
  console.log('Express API listening on 127.0.0.1:3000 (behind Nginx)');
});
```

---

## Common mistakes

### Mistake 1 — Forgetting to forward the Host and X-Forwarded headers

```nginx
# ❌ WRONG — bare proxy_pass with no headers forwarded
location / {
    proxy_pass http://127.0.0.1:3000;
    # Node now sees every client as coming from 127.0.0.1
    # and the Host header is "127.0.0.1:3000" instead of the real domain
}
```

```nginx
# ✅ CORRECT — forward the real client info to Node
location / {
    proxy_pass http://127.0.0.1:3000;
    proxy_set_header Host $host;                       # real domain, not Nginx's own address
    proxy_set_header X-Real-IP $remote_addr;            # real visitor IP
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for; # preserves proxy chain
    proxy_set_header X-Forwarded-Proto $scheme;         # http or https as seen by the client
}
```

### Mistake 2 — Serving static files through Node instead of letting Nginx handle them

```js
// ❌ WRONG — Node reads every image/CSS file from disk on every single request,
// burning event-loop time that should go to real API logic
const express = require('express');
const app = express();
const fs = require('fs');
const path = require('path');

app.get('/static/:fileName', (req, res) => {
  const filePath = path.join(__dirname, 'public', req.params.fileName); // Topic 06/14
  fs.readFile(filePath, (err, data) => {           // blocking-ish, adds latency
    if (err) return res.status(404).end();
    res.end(data);
  });
});
```

```nginx
# ✅ CORRECT — let Nginx serve static files directly from disk, bypassing Node entirely
location /static/ {
    alias /var/www/myapp/public/;   # Nginx (written in C) serves files far faster than Node
    expires 30d;                    # cache aggressively since assets rarely change
    add_header Cache-Control "public, immutable";
}
# Node's app.js no longer needs a /static route at all — one less thing for it to do
```

### Mistake 3 — Binding Node to 0.0.0.0 or the public IP, bypassing Nginx entirely

```js
// ❌ WRONG — Node listens on every network interface, so attackers can hit port 3000
// directly, skipping Nginx's SSL termination, rate limiting, and header sanitization
const server = require('http').createServer(handler);
server.listen(3000, '0.0.0.0'); // reachable from the public internet, unencrypted
```

```js
// ✅ CORRECT — bind only to localhost so Nginx is the ONLY way in
const server = require('http').createServer(handler);
server.listen(3000, '127.0.0.1'); // unreachable except from Nginx on the same machine
// Firewall rule (extra safety net): block external traffic to port 3000 entirely
// e.g. `sudo ufw deny 3000` — defense in depth even if the bind address is ever wrong
```

---

## Practice exercises

### Exercise 1 — easy

Write an Nginx server block that:
1. Listens on port 80 for the domain `shop.local`
2. Forwards every request to a Node app running on `127.0.0.1:5000`
3. Sets `Host`, `X-Real-IP`, and `X-Forwarded-For` headers correctly

Then write the matching minimal Node.js `http` server (listening on `127.0.0.1:5000`) that logs `req.headers['x-forwarded-for']` and responds with a plain text greeting.

```nginx
// Write your code here
```

---

### Exercise 2 — medium

Extend Exercise 1 into a full production-style config:
1. Add a second `server` block that listens on port 80 and permanently redirects (`301`) all traffic to HTTPS
2. Add an HTTPS `server` block on port 443 using placeholder cert paths (`ssl_certificate`, `ssl_certificate_key`)
3. Add a `location /static/` block that serves files from `/var/www/shop/public/` directly from disk, with 7-day browser caching
4. Add a `location /` block proxying everything else to the Node app on `127.0.0.1:5000`, including the `X-Forwarded-Proto` header

Write the Node/Express side too: an endpoint `GET /api/v1/products/:productId` that uses `app.set('trust proxy', 1)` and returns `{ productId, secure: req.secure }`.

```nginx
// Write your code here
```

---

### Exercise 3 — hard

Design a config for a small e-commerce backend with **two Node services behind one Nginx instance**:
1. An **orders service** on `127.0.0.1:4001`, reachable at paths starting with `/api/orders/`
2. A **payments service** on `127.0.0.1:4002`, reachable at paths starting with `/api/payments/`
3. All static assets served from `/var/www/shop/public/` under `/static/`
4. A shared HTTPS block terminating SSL once for the whole domain
5. A `/health` location that returns a static `200 OK` directly from Nginx (using the `return` directive) WITHOUT touching either Node service — useful for load balancer health checks
6. Rate limit the `/api/payments/` location to 5 requests per second per IP using `limit_req_zone` and `limit_req`

Write the full `nginx.conf` server block(s) plus two minimal Express apps (`orders.js` and `payments.js`) that each expose one route confirming which service answered, using `trust proxy`.

```nginx
// Write your code here
```

---

## Quick reference cheat sheet

```
CORE DIRECTIVES
  listen 80 / 443 ssl http2;      → port(s) Nginx accepts connections on
  server_name example.com;        → which domain this block handles
  location /path/ { ... }         → rules applied to matching request paths
  proxy_pass http://host:port;    → forward the request to a backend (Node)
  alias /disk/path/;              → serve static files directly, bypassing Node
  return 301 https://...;         → issue a redirect without touching Node

HEADERS TO ALWAYS FORWARD TO NODE
  proxy_set_header Host $host;
  proxy_set_header X-Real-IP $remote_addr;
  proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  proxy_set_header X-Forwarded-Proto $scheme;

SSL / TLS TERMINATION
  ssl_certificate     fullchain.pem;   → public certificate
  ssl_certificate_key privkey.pem;     → private key (keep permissions strict!)
  ssl_protocols TLSv1.2 TLSv1.3;       → disallow legacy insecure TLS
  Node itself stays on plain http — Nginx handles all encryption/decryption

STATIC FILES
  location /static/ { alias /var/www/app/public/; }
  → Nginx (C, event-driven) serves files far faster than routing through Node
  → add expires / Cache-Control headers here, not in Node

NODE-SIDE CHECKLIST
  app.set('trust proxy', 1)         → Express reads X-Forwarded-* correctly
  server.listen(port, '127.0.0.1')  → never bind to 0.0.0.0 behind a proxy
  req.ip / req.secure                → correct only if trust proxy is set

USEFUL COMMANDS
  sudo nginx -t                → validate config syntax before reloading
  sudo systemctl reload nginx  → apply new config with zero downtime
  sudo systemctl status nginx  → check if Nginx is running
  tail -f /var/log/nginx/error.log → debug proxy failures (502, 504, etc.)

COMMON ERRORS
  502 Bad Gateway   → Node process is down or proxy_pass port is wrong
  504 Gateway Timeout → Node too slow, raise proxy_read_timeout or fix code
  Websockets fail   → missing "Upgrade" / "Connection: upgrade" headers
```

---

## Connected topics

- **80 — HTTPS, TLS, certificates** — the deeper mechanics of SSL/TLS that Nginx handles here via `ssl_certificate` and `ssl_certificate_key`
- **105 — Dockerizing a Node.js app** — Nginx and Node are commonly run as separate containers, linked together in the same Docker Compose network
- **73 — Horizontal scaling with PM2** — Nginx's `proxy_pass` can load-balance across multiple PM2 cluster instances or multiple Node containers behind a single domain
