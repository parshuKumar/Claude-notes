# 29 — Reverse Proxies

> **Phase 5 — Topic 2 of 7**
> Prerequisites: `01-how-the-internet-works.md`, `10-http1.1-in-depth.md`, `28-load-balancers.md`

---

## ELI5 — The Simple Analogy

Imagine a fancy hotel. When you arrive, you don't wander into the kitchen, the laundry room, or the manager's office looking for what you need. You walk up to the **front desk (concierge)**. You ask for room service, and the concierge calls the kitchen for you. You ask for a taxi, and the concierge calls the dispatcher. You never see the staff behind the scenes, and they never see you directly — every request goes through the front desk.

A **reverse proxy** is that concierge. It sits at the front of your building (your servers) and is the single face the outside world talks to. Visitors (clients) only ever meet the concierge. Behind the desk, the concierge quietly routes each request to the right department (your app servers), collects the answer, and hands it back to you.

Now here's the twist that trips everyone up. When the concierge calls the kitchen, the kitchen sees the call as coming from **the front desk**, not from you. So if the kitchen keeps a log of "who ordered what," every order looks like it came from the concierge. If the kitchen wants to know *your* name, the concierge has to write it on a sticky note and pass it along. That sticky note is the `X-Forwarded-For` header, and forgetting it is why your app logs show `127.0.0.1` for every single user.

Compare this to a **forward proxy**, which is the *opposite*: that's like a personal assistant *you* hire to make calls on *your* behalf so the outside world never sees you. Same word "proxy," opposite direction. We'll draw that distinction clearly below.

---

## Where This Lives in the Network Stack

A reverse proxy is an **application-layer (Layer 7)** middleman. It terminates the client's TCP + TLS connection, reads the full HTTP request, and then opens a **brand-new** TCP connection to your backend.

```
┌─────────────────────────────────────────────────────────────────┐
│  Layer 7 — Application   (HTTP)   ← reverse proxy reads & rewrites│  ← nginx works HERE
│  Layer 6 — Presentation  (TLS)    ← reverse proxy TERMINATES TLS  │
│  Layer 5 — Session                                              │
│  Layer 4 — Transport     (TCP)    ← TWO separate connections     │
│  Layer 3 — Network       (IP)     ← proxy's IP replaces client's │
│  Layer 2 — Data Link                                             │
│  Layer 1 — Physical                                              │
└─────────────────────────────────────────────────────────────────┘
```

The critical mental model: there is **no single connection** from client to backend. There are **two independent connections** stitched together by the proxy.

```
   CONNECTION A                         CONNECTION B
┌──────────┐   TCP+TLS   ┌───────────┐   plain TCP   ┌──────────┐
│  Client  │◄──────────►│  nginx    │◄────────────►│  Your app│
│ 203.0.113│  port 443   │ reverse   │  port 3000    │127.0.0.1 │
│    .5    │             │  proxy    │               │  :3000   │
└──────────┘             └───────────┘               └──────────┘
     │                        │                            │
 sees: nginx's IP        sees: real client         sees: nginx's IP
 as the "server"         on connection A            as the "client"
                         sees: itself as the
                         client on connection B
```

Because connection B is opened *by nginx*, the backend's TCP socket reports the **source IP as nginx** (often `127.0.0.1` when they're on the same box). This is the single most important fact in this entire topic.

> **Contrast with an L4 load balancer (Topic 28):** an L4 device forwards packets and can preserve the client IP at the TCP layer. A reverse proxy operates at L7 — it does *not* forward packets, it makes its own request — so it must carry the client IP up in an HTTP header.

---

## What Is This?

A **reverse proxy** is a server that accepts client requests, forwards them to one or more **backend (upstream)** servers, and returns the backend's response to the client — while the client believes it is talking directly to a single origin server. The backends are hidden behind it.

### Forward proxy vs reverse proxy — the distinction

These two share a name but point in opposite directions. Get this right and everything else clicks.

**Forward proxy** — sits in front of **CLIENTS**, acts on the client's behalf, is usually **configured on the client**, and is used for *outbound* traffic.

```
              ┌─────────────────┐
  Client A ──►│                 │
  Client B ──►│  FORWARD PROXY  │──► The Internet ──► any website
  Client C ──►│  (corp gateway) │        (facebook.com, github.com…)
              └─────────────────┘
   The proxy hides the CLIENTS from the servers.
   Use cases: corporate egress filtering, caching outbound requests,
              anonymity, bypassing geo-blocks.
   The website sees the PROXY's IP, not the client's.
```

**Reverse proxy** — sits in front of **SERVERS**, acts on the servers' behalf, is **transparent to clients** (they don't configure anything), and is used for *inbound* traffic.

```
                                   ┌─────────────────┐──► App server 1
  any client ──► The Internet ──► │  REVERSE PROXY  │──► App server 2
  (browser, curl, mobile app)     │     (nginx)     │──► App server 3
                                   └─────────────────┘──► static files
   The proxy hides the SERVERS from the clients.
   Use cases: TLS termination, load balancing, caching, routing,
              compression, rate limiting, a single public entry point.
   The client sees ONE server (the proxy). Backends are invisible.
```

| | Forward proxy | Reverse proxy |
|---|---|---|
| Sits in front of | Clients | Servers |
| Acts on behalf of | The client | The server |
| Who configures it | The client (browser/OS proxy settings) | The server operator |
| Client aware of it? | Yes (client points at it) | No (transparent) |
| Traffic direction | Outbound (client → internet) | Inbound (internet → your app) |
| Hides | The clients from the world | The backends from clients |
| Example | Corporate web filter, Squid | nginx, HAProxy, Envoy, Caddy |

**This topic is entirely about reverse proxies.** Whenever you deploy a web app, the thing listening on ports 80/443 is almost always a reverse proxy, and your actual application sits behind it on a high, private port.

The big players: **nginx** (the default choice, our focus), **HAProxy** (L4/L7, load-balancing pedigree), **Envoy** (cloud-native, powers service meshes — Topic 36), **Caddy** (automatic HTTPS), and **Traefik** (auto-discovers containers). They all solve the same problem; nginx is the one you'll meet first.

---

## Why Does It Matter for a Backend Developer?

Because in production your app almost **never** faces the internet directly. Something sits in front of it, and that something changes what your code sees. If you don't understand the proxy, you will:

- Log `127.0.0.1` for every user and wonder why your analytics are useless.
- Build a rate limiter that throttles *all* traffic at once because every request looks like it came from one IP.
- Ship an app that redirects `https://` → `http://` in an infinite loop because it thinks the connection is insecure.
- Watch WebSockets mysteriously fail in production but work locally.
- Get a `502 Bad Gateway` and have no idea whether it's the proxy or your app.

Concretely, a reverse proxy in front of your app buys you:

- **TLS termination** — the proxy handles all the certificate/encryption work (Topic 09), your app speaks plain HTTP internally. One place to renew certs.
- **Serving static files** — nginx serves images/CSS/JS from disk far faster than Node.js, freeing your app for real work.
- **Compression** — gzip/brotli responses on the fly so you don't ship compression code in every service.
- **Caching** — cache responses so identical requests never touch your app.
- **Request buffering** — nginx absorbs slow clients (a phone on 3G) so your app isn't tied up waiting on a trickle of bytes.
- **Load balancing** — spread traffic across many backend instances (Topic 28).
- **Rate limiting** — reject abusive traffic before it reaches your app.
- **Path-based routing** — `/api/*` → service A, `/images/*` → service B, all under one domain.
- **Hiding backend topology** — clients see one host; you can add, remove, and move backends freely.
- **A single public entry point** — one place for TLS, logging, WAF rules, and security headers.

The catch is that every one of these benefits introduces the "who is the real client?" problem, which is the heart of this topic.

---

## The Packet/Protocol Anatomy

Let's look at what's actually on the wire on **both** connections. The client sends a normal HTTPS request. nginx receives it, then constructs a *new* HTTP request to the backend — and this is where it injects the forwarding headers.

```
════════ CONNECTION A: client → nginx (over TLS, port 443) ════════

GET /api/orders HTTP/1.1
Host: shop.example.com
User-Agent: Mozilla/5.0 ...
Accept: application/json
Cookie: session=abc123
        (source IP on the packet: 203.0.113.5 — the real client)


════════ CONNECTION B: nginx → backend (plain HTTP, port 3000) ════

GET /api/orders HTTP/1.1
Host: shop.example.com              ← preserved via proxy_set_header Host
User-Agent: Mozilla/5.0 ...
Accept: application/json
Cookie: session=abc123
X-Real-IP: 203.0.113.5              ← INJECTED by nginx (single client IP)
X-Forwarded-For: 203.0.113.5        ← INJECTED (client, then each proxy hop)
X-Forwarded-Proto: https            ← INJECTED (original scheme was HTTPS)
X-Forwarded-Host: shop.example.com  ← INJECTED (original Host)
X-Forwarded-Port: 443               ← INJECTED (original port)
        (source IP on the packet: 127.0.0.1 — nginx itself)
```

Your app reads the packet's source IP and sees `127.0.0.1`. It must instead read `X-Forwarded-For` to learn the real client is `203.0.113.5`.

### The forwarding headers, explained

| Header | Value | Meaning | Gotcha |
|--------|-------|---------|--------|
| `X-Forwarded-For` | `client, proxy1, proxy2` | Comma-separated **list**: original client first, then each proxy that touched it | Client-settable; only trust the rightmost entries you control |
| `X-Real-IP` | `203.0.113.5` | The single client IP (nginx convention, not a standard) | Only one value; overwrite it at your edge |
| `X-Forwarded-Proto` | `https` or `http` | The scheme the **client** used before TLS termination | Miss this and your app thinks HTTPS traffic is plain HTTP |
| `X-Forwarded-Host` | `shop.example.com` | Original `Host` header the client sent | Used when building absolute URLs |
| `X-Forwarded-Port` | `443` | Original port the client connected to | Rarely needed but useful for URL building |
| `Forwarded` (RFC 7239) | `for=203.0.113.5;proto=https;host=shop.example.com` | The **standardized** single header meant to replace all the `X-Forwarded-*` ones | Adoption is spotty; most stacks still use `X-Forwarded-*` |

The `X-Forwarded-*` family are **de facto standards** (the `X-` prefix means non-standard). RFC 7239 defined `Forwarded` in 2014 to unify them, but the world was already using `X-Forwarded-For` everywhere, so both coexist. In practice you'll configure and read `X-Forwarded-For` far more often than `Forwarded`.

### The PROXY protocol (for L4, no HTTP involved)

When the front layer is an **L4** proxy/load balancer (it forwards TCP, never parses HTTP — think AWS NLB, or nginx `stream {}`), it can't inject an HTTP header because it doesn't read HTTP. Instead it prepends a tiny plaintext line — the **PROXY protocol** header — to the front of the TCP stream, *before* the TLS/HTTP bytes:

```
PROXY TCP4 203.0.113.5 198.51.100.2 56324 443\r\n
      │    │           │            │     └ destination port
      │    │           │            └ source port
      │    │           └ destination (proxy) IP
      │    └ source (real client) IP
      └ address family
<then the normal TLS ClientHello / HTTP bytes follow...>
```

The backend must be told to expect this (nginx: `proxy_protocol` on the `listen` line, plus `set_real_ip_from`). It's how you preserve the client IP through a **TCP** proxy that never sees HTTP headers.

---

## How It Works — Step by Step

A client requests `https://shop.example.com/api/orders`. nginx is the reverse proxy on the public IP; your Node app is on `127.0.0.1:3000`.

### Step 1 — Client connects to nginx
```
DNS resolves shop.example.com → the PUBLIC IP of the nginx box.
Client opens TCP to that IP:443, then does the TLS handshake WITH NGINX.
nginx presents the certificate. The client encrypts to nginx.
→ From the client's view, nginx IS the server.
```

### Step 2 — nginx terminates TLS and reads the full request
```
nginx decrypts the TLS record → now holds plaintext HTTP:
    GET /api/orders HTTP/1.1
    Host: shop.example.com
This is "TLS termination": encryption stops here. Inside your network
it's plain HTTP (unless you re-encrypt for the backend — mTLS, Topic 36).
```

### Step 3 — nginx matches a `location` and picks the upstream
```
location /api/  → proxy_pass http://127.0.0.1:3000;
nginx decides: this request goes to the Node backend.
(With multiple backends, load-balancing algorithm chooses one — Topic 28.)
```

### Step 4 — nginx opens a NEW TCP connection to the backend
```
nginx (as a client) opens TCP from 127.0.0.1:<ephemeral> → 127.0.0.1:3000.
This is a SEPARATE connection. The backend socket's peer address is nginx.
→ From the backend's view, the "client" is 127.0.0.1.  ← THE 127.0.0.1 PROBLEM
```

### Step 5 — nginx builds the upstream request and injects headers
```
It copies the client's request and, per proxy_set_header directives, adds:
    Host: shop.example.com
    X-Real-IP: 203.0.113.5
    X-Forwarded-For: 203.0.113.5
    X-Forwarded-Proto: https
The real client IP now lives in headers, since the socket lost it.
```

### Step 6 — Backend processes and responds
```
Your Node app reads X-Forwarded-For to get 203.0.113.5 (if configured to
trust the proxy). It runs your route handler and returns HTTP 200 + JSON
over connection B.
```

### Step 7 — nginx relays the response to the client
```
nginx buffers the backend's response (proxy buffering), optionally gzips it,
adds/removes headers, re-encrypts it over the client's TLS session,
and sends it back on connection A. The client never knew the backend existed.
```

### The full picture
```
┌────────────┐        ┌──────────────────────────┐        ┌─────────────┐
│  Client    │        │   nginx  (reverse proxy) │        │  Node app   │
│ 203.0.113.5│        │   listen 443 ssl         │        │127.0.0.1:3000│
│            │──TLS──►│ ── terminates TLS ──     │        │             │
│            │  :443  │   reads HTTP request     │        │             │
│            │        │   matches location /api/ │        │             │
│            │        │   opens NEW TCP ─────────┼─plain─►│ reads XFF   │
│            │        │   injects X-Forwarded-*  │  HTTP  │ 203.0.113.5 │
│            │◄───────┤◄─── buffers + gzips ─────┼◄───────┤ returns 200 │
│  sees 200  │  TLS   │     re-encrypts response │        │             │
└────────────┘        └──────────────────────────┘        └─────────────┘
   CONNECTION A (encrypted)              CONNECTION B (plaintext, private)
```

---

## Exact Syntax Breakdown

### The `location` + `proxy_pass` block

```
location /api/ {
│        │
│        └── URI prefix to match: any request path starting with /api/
└── defines routing rules for a set of URIs

    proxy_pass http://127.0.0.1:3000;
    │          │      │         │
    │          │      │         └── backend PORT your app listens on
    │          │      └── backend IP (loopback = same machine)
    │          └── scheme to the BACKEND (http = plaintext internally)
    └── forward matched requests to this upstream
}
```

**Trailing-slash trap in `proxy_pass`** — this one bites everyone:
```
location /api/ { proxy_pass http://127.0.0.1:3000/;  }
   Request /api/orders  →  backend receives /orders     (prefix STRIPPED)

location /api/ { proxy_pass http://127.0.0.1:3000;   }
   Request /api/orders  →  backend receives /api/orders (prefix KEPT)
```
A trailing slash on `proxy_pass` replaces the matched `location` prefix; no slash passes the URI through unchanged.

### The `proxy_set_header` directives

```
proxy_set_header  Host              $host;
│                 │                 │
│                 │                 └── nginx variable = the Host from the request
│                 └── header NAME to set on the UPSTREAM request
└── set/override a header sent to the backend

proxy_set_header  X-Real-IP         $remote_addr;
                                    └── the DIRECT peer's IP (the real client,
                                        if nginx is the first hop)

proxy_set_header  X-Forwarded-Proto $scheme;
                                    └── "http" or "https" — the scheme nginx
                                        received the request on

proxy_set_header  X-Forwarded-For   $proxy_add_x_forwarded_for;
                                    └── the magic APPEND variable, below
```

### `$proxy_add_x_forwarded_for` — the append variable

This is the one to understand deeply. It is defined as:

```
$proxy_add_x_forwarded_for  =  <incoming X-Forwarded-For header>  +  ", "  +  $remote_addr

   ┌─ If the request ALREADY had an X-Forwarded-For (another proxy set it):
   │     incoming: "203.0.113.5, 70.41.3.18"
   │     $remote_addr (the previous proxy): "10.0.0.7"
   │     result: "203.0.113.5, 70.41.3.18, 10.0.0.7"   ← appends the hop
   │
   └─ If there was NO incoming X-Forwarded-For (nginx is the first hop):
         incoming: (empty)
         $remote_addr: "203.0.113.5"
         result: "203.0.113.5"                          ← just the client
```

So `X-Forwarded-For` grows one IP per proxy hop. **Leftmost = original client, each entry to the right = the next proxy in the chain.** This is *exactly* why you must not blindly trust the leftmost value (see Common Mistakes) — a client can pre-populate the header before it ever reaches you.

### Common `$variables` reference

| Variable | Value |
|----------|-------|
| `$host` | Host header (or server name if absent) |
| `$remote_addr` | IP of the **immediate** peer connecting to nginx |
| `$scheme` | `http` or `https` (how the client reached nginx) |
| `$proxy_add_x_forwarded_for` | incoming XFF + `$remote_addr` |
| `$http_x_forwarded_for` | raw incoming `X-Forwarded-For` header, verbatim |
| `$request_uri` | full original URI including query string |

---

## Example 1 — Basic

A minimal reverse proxy: nginx on port 80 forwarding everything to a Node app on port 3000.

**The app** (`app.js`) — deliberately logs the IP it sees:
```javascript
const http = require("http");
http.createServer((req, res) => {
  console.log("socket peer :", req.socket.remoteAddress);         // what the OS sees
  console.log("X-Forwarded-For:", req.headers["x-forwarded-for"]); // what nginx injected
  res.end("hello from backend\n");
}).listen(3000, "127.0.0.1");
```

**The nginx config** (`/etc/nginx/conf.d/basic.conf`):
```nginx
server {
    listen 80;
    server_name localhost;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host            $host;
        proxy_set_header X-Real-IP       $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

**Reload and test:**
```bash
node app.js &                          # start backend on :3000
sudo nginx -t                          # test config syntax
sudo nginx -s reload                   # apply it
curl -s http://localhost/              # hit nginx (:80), NOT the app directly
```

**What the app logs:**
```
socket peer : 127.0.0.1        ← the socket sees NGINX, not you
X-Forwarded-For: 127.0.0.1     ← nginx put your real IP here (loopback in this demo)
```

Now hit the backend **directly**, bypassing nginx, to prove there's no XFF header:
```bash
curl -s http://127.0.0.1:3000/
```
```
socket peer : 127.0.0.1
X-Forwarded-For: undefined     ← no proxy = no header. YOUR APP must handle both cases.
```

The lesson from the basic case: the socket peer address is *always* nginx's address once a proxy is in play. The real client only exists in the header nginx chose to add.

---

## Example 2 — Production Scenario

**The setup:** `shop.example.com` runs behind nginx doing TLS termination. The Node app is on `127.0.0.1:3000`. Three separate bugs land in the same week — all rooted in proxy misconfiguration.

### Bug A — "Every user shows as 127.0.0.1 in our logs, rate limiter, and geo-IP"

The analytics dashboard shows 100% of traffic from `127.0.0.1`. The per-IP rate limiter throttles *everyone* at once because they all look like one IP. Geo-IP puts every user in "localhost."

**Diagnosis** — check what the app receives:
```bash
curl -s https://shop.example.com/whoami
# app returns: {"socketIp":"127.0.0.1","xff":null}   ← no XFF header at all!
```
nginx isn't injecting the header. The `server` block is missing the `proxy_set_header` lines. Fix the nginx config:
```nginx
server {
    listen 443 ssl;
    server_name shop.example.com;
    ssl_certificate     /etc/letsencrypt/live/shop.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/shop.example.com/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;               # ← added
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for; # ← added
        proxy_set_header X-Forwarded-Proto $scheme;                    # ← added
    }
}
```
```bash
sudo nginx -t && sudo nginx -s reload
```
Now the app must be told to **trust** the proxy and read the header. In Express:
```javascript
// Trust exactly ONE proxy hop (nginx). Never use `true` in production.
app.set("trust proxy", 1);
// Now req.ip returns the real client IP from X-Forwarded-For,
// instead of the socket's 127.0.0.1.
app.get("/whoami", (req, res) =>
  res.json({ ip: req.ip, xff: req.headers["x-forwarded-for"] }));
```
```bash
curl -s https://shop.example.com/whoami
# {"ip":"203.0.113.5","xff":"203.0.113.5"}   ← the REAL client. Fixed.
```

Alternatively, do it at the nginx layer with the `real_ip` module so even nginx's own `$remote_addr` logs the true client:
```nginx
set_real_ip_from 127.0.0.1;         # trust XFF only from this hop
real_ip_header   X-Forwarded-For;   # take client IP from this header
real_ip_recursive on;               # skip trusted proxies in the chain
```

### Bug B — "Login redirects loop forever / all our links are http:// on an https:// site"

Users on `https://shop.example.com/login` get bounced in an infinite redirect. Because nginx terminated TLS, the request reaching the app is plain `http://`. The framework sees "insecure," tries to redirect to `https://`, nginx forwards that as `http://` again → loop.

**Diagnosis** — follow the redirects and watch the scheme flip:
```bash
curl -sIL https://shop.example.com/login | grep -i location
# Location: http://shop.example.com/login    ← app emitted http:// !
# Location: http://shop.example.com/login    ← ...and again. Loop.
```

**Root cause:** the app doesn't know the *original* scheme was HTTPS because it only sees connection B, which is plain HTTP. **Fix:** nginx must send `X-Forwarded-Proto: $scheme` (added above) **and** the app must honor it. With Express + `trust proxy` set, `req.protocol` and `req.secure` now read `X-Forwarded-Proto` correctly:
```javascript
app.set("trust proxy", 1);
// req.protocol === "https", req.secure === true  ← now driven by X-Forwarded-Proto
```
```bash
curl -sIL https://shop.example.com/login | grep -i location
# Location: https://shop.example.com/dashboard   ← correct scheme. Loop gone.
```

### Bug C — "WebSockets connect locally but fail behind nginx in production"

The chat feature works on the dev machine (browser → Node directly) but in production the WebSocket handshake returns `400`/`426` and never upgrades.

**Diagnosis** — a WebSocket starts as an HTTP `Upgrade` request (Topic 13):
```bash
curl -sv https://shop.example.com/ws \
  -H "Connection: Upgrade" -H "Upgrade: websocket" \
  -H "Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==" \
  -H "Sec-WebSocket-Version: 13" 2>&1 | grep -i "HTTP/\|upgrade\|connection"
# < HTTP/1.1 400 Bad Request     ← nginx stripped the Upgrade headers, no switch
```

**Root cause:** by default nginx uses HTTP/1.0 to the upstream and does **not** forward the hop-by-hop `Upgrade`/`Connection` headers, so the backend never sees an upgrade request. **Fix** — force HTTP/1.1 and pass the upgrade headers through:
```nginx
location /ws {
    proxy_pass http://127.0.0.1:3000;
    proxy_http_version 1.1;                    # WebSockets require HTTP/1.1
    proxy_set_header Upgrade    $http_upgrade;  # pass the Upgrade: websocket header
    proxy_set_header Connection "upgrade";      # signal a connection upgrade
    proxy_set_header Host       $host;
    proxy_read_timeout 3600s;                   # keep idle WS open (default 60s!)
}
```
```bash
sudo nginx -s reload
# retry the curl → "< HTTP/1.1 101 Switching Protocols"  ← upgrade succeeds
```

Note `proxy_read_timeout 3600s`: nginx closes an upstream connection after 60s of silence by default, which kills idle WebSockets and long-polling. Buffering also matters here — for streaming/SSE and WebSockets you often want `proxy_buffering off;` so nginx doesn't hold bytes waiting to fill a buffer.

**The through-line of all three bugs:** the proxy split one connection into two, and information that lived in the connection (client IP, scheme, upgrade intent) has to be *deliberately* carried across the gap. Miss any piece and something breaks.

---

## Common Mistakes

### Mistake 1: Reading the client IP from the socket instead of `X-Forwarded-For`
```javascript
// WRONG behind a proxy — always logs the proxy's IP:
const ip = req.socket.remoteAddress;   // → "127.0.0.1" for every user

// RIGHT — trust the proxy, then read the forwarded header:
app.set("trust proxy", 1);
const ip = req.ip;                     // → real client from X-Forwarded-For
```
**Why:** the proxy opened its own TCP connection, so the socket's peer is the proxy. The real IP only exists in the header the proxy injected. Your rate limiter, audit logs, and geo-IP all break silently otherwise.

---

### Mistake 2: Blindly trusting `X-Forwarded-For` (it's spoofable)
```
A client can send this BEFORE it ever reaches your proxy:
    curl https://shop.example.com/ -H "X-Forwarded-For: 1.2.3.4, evil"

If your app naively takes the LEFTMOST entry, the attacker just
forged their "real IP" — bypassing IP allowlists, poisoning logs,
and evading per-IP rate limits.
```
**Fix:** only trust the header from proxies **you control**. Configure the *number of trusted hops* — `app.set("trust proxy", 1)` in Express, or `set_real_ip_from <proxy-cidr>` in nginx — and read the correct entry from the **right side** of the list (the last IP your trusted proxy appended), never the attacker-controllable left side.

---

### Mistake 3: Forgetting `X-Forwarded-Proto` → http/https confusion & redirect loops
```
Client → https://... → nginx (TLS terminated) → http://... → app
App sees "http", "helpfully" redirects to https, nginx forwards it as http,
app redirects again → INFINITE LOOP. Also breaks Secure cookies and OAuth
callback URLs that get built as http:// on an https:// site.
```
**Fix:** send `proxy_set_header X-Forwarded-Proto $scheme;` in nginx **and** make the app honor it (`trust proxy` in Express, `ForwardedHeaderFilter` in Spring, `SECURE_PROXY_SSL_HEADER` in Django). The app must learn the *original* scheme, not the internal one.

---

### Mistake 4: Not passing `Upgrade`/`Connection` headers → WebSockets break
```nginx
# BROKEN for WebSockets — nginx defaults to HTTP/1.0 upstream and
# drops the hop-by-hop Upgrade header, so no protocol switch happens:
location /ws { proxy_pass http://127.0.0.1:3000; }

# FIXED:
location /ws {
    proxy_pass http://127.0.0.1:3000;
    proxy_http_version 1.1;
    proxy_set_header Upgrade    $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_read_timeout 3600s;   # or WS dies after 60s idle
}
```
**Why:** `Upgrade` and `Connection` are *hop-by-hop* headers — a proxy strips them unless explicitly told to forward them. Without HTTP/1.1 upstream, the handshake can't complete at all.

---

### Mistake 5: Forgetting the `Host` header → vhost routing & backend links misbehave
```nginx
# Without this, the backend receives Host: 127.0.0.1:3000 (from proxy_pass),
# not the real domain. Virtual-host routing picks the wrong site, absolute
# redirects point at 127.0.0.1, and multi-tenant apps mis-route.
proxy_set_header Host $host;
```
**Why:** by default nginx sets `Host` to the value in `proxy_pass` (the upstream address). Apps that route by hostname, build absolute URLs, or serve multiple tenants need the *original* `Host`. Note `$host` vs `$http_host`: `$host` is normalized (lowercased, port stripped) and falls back to `server_name`; `$http_host` is the raw header verbatim.

---

### Mistake 6: Exposing the backend port directly (bypassing the proxy)
```
If :3000 is open to the world, attackers hit the app directly, skipping
your TLS, rate limiting, WAF rules, and header injection — and they can
forge X-Forwarded-For freely. Bind the app to 127.0.0.1 (not 0.0.0.0) and
firewall the backend port so ONLY the proxy can reach it.
```

---

## Hands-On Proof

```bash
# 0. Confirm nginx is running and which ports it listens on
sudo nginx -t                      # syntax check the config
sudo ss -ltnp | grep nginx         # see nginx on :80 / :443

# 1. Hit through the proxy and see the injected headers reflected back
#    httpbin echoes request headers, so you can SEE what a proxy adds:
curl -s https://httpbin.org/headers
#    Look for "X-Forwarded-For" and "X-Forwarded-Proto" that upstream
#    infrastructure already injected before your request reached the app.

# 2. Prove the two-connection model — the source IP the backend sees.
#    Start a tiny backend that prints the socket peer, then proxy to it:
node -e 'require("http").createServer((q,r)=>{console.log("peer",q.socket.remoteAddress,"xff",q.headers["x-forwarded-for"]);r.end("ok")}).listen(3000,"127.0.0.1")' &
curl -s http://localhost/          # through nginx → peer=127.0.0.1, xff set
curl -s http://127.0.0.1:3000/     # direct → peer=127.0.0.1, xff undefined

# 3. Forge X-Forwarded-For to prove it's client-controllable
curl -s http://localhost/ -H "X-Forwarded-For: 6.6.6.6"
#    Watch what the backend logs. If it trusts the leftmost value blindly,
#    it now believes you are 6.6.6.6. THIS is why trust config matters.

# 4. Watch the proxy pass a WebSocket upgrade (needs the /ws block from Ex.2)
curl -sv http://localhost/ws \
  -H "Connection: Upgrade" -H "Upgrade: websocket" \
  -H "Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==" \
  -H "Sec-WebSocket-Version: 13" 2>&1 | grep -i "HTTP/\|switching"
#    "101 Switching Protocols" = upgrade forwarded correctly.

# 5. Confirm X-Forwarded-Proto behavior on an https request
curl -sI https://shop.example.com/ | grep -i "location\|strict-transport"
#    Redirects should stay https:// if the proxy sends X-Forwarded-Proto.

# 6. Read nginx's own view of who connected (the access log)
sudo tail -f /var/log/nginx/access.log
#    The first field is $remote_addr. Set up log_format with
#    $proxy_add_x_forwarded_for to log the full chain.
```

---

## Practice Exercises

### Exercise 1 — Easy: Prove the 127.0.0.1 problem yourself
```bash
# 1. Start a backend that prints the socket peer AND the XFF header:
node -e 'require("http").createServer((q,r)=>{console.log("peer:",q.socket.remoteAddress,"| xff:",q.headers["x-forwarded-for"]);r.end("ok\n")}).listen(3000,"127.0.0.1")' &

# 2. Put nginx in front on :8080 forwarding to :3000 WITHOUT any
#    proxy_set_header lines. Reload nginx.

# 3. Run:  curl http://localhost:8080/
#    Then: curl http://127.0.0.1:3000/   (direct)

# Answer:
# - What does the backend log for "peer" in each case?
# - Is X-Forwarded-For present in either case? Why/why not?
# - Add `proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;`,
#   reload, and re-run. What changed?
```

### Exercise 2 — Medium: Build the full injection block and read the real IP in your app
```bash
# 1. Write a location block that injects Host, X-Real-IP,
#    X-Forwarded-For, and X-Forwarded-Proto.
# 2. In an Express app, set `trust proxy` to 1 and expose:
#    app.get("/ip", (req,res)=>res.json({ip:req.ip, proto:req.protocol}))
# 3. Hit it through the proxy AND forge a header:
curl -s http://localhost:8080/ip
curl -s http://localhost:8080/ip -H "X-Forwarded-For: 9.9.9.9"

# Answer:
# - With trust proxy = 1, which IP does req.ip return in each call? Why?
# - Change trust proxy to `true` (trust everything). Re-run the forged call.
#   What does req.ip return now, and why is that dangerous?
# - What does req.protocol report, and which header drives it?
```

### Exercise 3 — Hard (Production Simulation): Diagnose a chained-proxy IP + WebSocket outage
```bash
# Scenario: traffic flows  client → CDN → nginx → app.
# The CDN sets X-Forwarded-For: <client>. nginx appends its own hop.
# Two incidents: (a) your rate limiter is throttling the CDN's IP as if it
# were one user; (b) WebSockets 400 in prod only.

# Step 1 — Simulate the chain. Send a request pre-loaded with an upstream XFF:
curl -s http://localhost:8080/ip -H "X-Forwarded-For: 203.0.113.5"
# Inspect what $proxy_add_x_forwarded_for produced (log $http_x_forwarded_for
# and $remote_addr in nginx). How many IPs are in the final list, in what order?

# Step 2 — Your app must extract the CLIENT, not the CDN or nginx.
# Given XFF = "203.0.113.5, 70.41.3.18, 10.0.0.7" and you control the last
# TWO hops (nginx + CDN), which entry is the real client? Set Express
# `trust proxy` to the correct NUMBER and verify req.ip.

# Step 3 — WebSockets. Add a /ws location that forwards Upgrade/Connection,
# forces proxy_http_version 1.1, and sets proxy_read_timeout 3600s. Prove the
# 101 Switching Protocols with the curl upgrade handshake from Hands-On #4.

# Deliverables:
# - The nginx log line showing the full XFF chain.
# - The correct `trust proxy` value and the reasoning.
# - Proof (curl output) that the WS upgrade now returns 101.
# - One sentence: why does taking the LEFTMOST XFF entry let an attacker
#   spoof their IP, and how does counting trusted hops from the RIGHT fix it?
```

---

## Mental Model Checkpoint

Answer from memory. If you can't, re-read the relevant section.

1. **A forward proxy and a reverse proxy are both "proxies." In one sentence each, state whom each one acts on behalf of and in which direction traffic flows.**

2. **Your Node app logs `127.0.0.1` as the client IP for every request, even users in Australia. Explain mechanically why — reference the two-connection model — and name the header that carries the true IP.**

3. **Why is it unsafe to take the leftmost value of `X-Forwarded-For` as the client IP? What is the correct way to pick the real client from the chain?**

4. **An HTTPS site behind nginx keeps redirecting users in an infinite loop. Which single forwarding header is almost certainly missing/ignored, and what does the app wrongly believe without it?**

5. **WebSockets work in local dev but 400 behind nginx in production. Name the two headers that must be forwarded and the one `proxy_*` directive that forces the HTTP version needed for an upgrade.**

6. **What exactly does `$proxy_add_x_forwarded_for` compute, and how does its value differ when nginx is the first hop vs. when a CDN already set the header?**

7. **You have an L4 TCP load balancer (no HTTP parsing) in front of nginx and still want the real client IP. Which mechanism carries it, and why can't you just use `X-Forwarded-For` at that layer?**

---

## Quick Reference Card

| Directive / Concept | What it does |
|---------------------|--------------|
| `proxy_pass http://IP:PORT;` | Forward matched requests to this backend |
| `proxy_set_header Host $host;` | Preserve the original Host for vhost routing |
| `proxy_set_header X-Real-IP $remote_addr;` | Single client IP (immediate peer) |
| `proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;` | Append this hop to the client-IP chain |
| `proxy_set_header X-Forwarded-Proto $scheme;` | Tell the app the original http/https |
| `proxy_http_version 1.1;` | Required for keep-alive & WebSocket upgrades |
| `proxy_set_header Upgrade $http_upgrade;` + `Connection "upgrade";` | Pass through WebSocket upgrade |
| `proxy_read_timeout 3600s;` | Keep idle upstream connections open (default 60s) |
| `proxy_buffering off;` | Stream responses (SSE/WebSockets) without buffering |
| `set_real_ip_from CIDR;` + `real_ip_header X-Forwarded-For;` | Make nginx itself trust & resolve the real client IP |

| Forwarding header | Carries | Trust rule |
|-------------------|---------|-----------|
| `X-Forwarded-For` | client, then each proxy (list) | Trust only hops you control; read from the right |
| `X-Real-IP` | single client IP | Overwrite at your edge |
| `X-Forwarded-Proto` | original scheme (http/https) | Must-have behind TLS termination |
| `X-Forwarded-Host` / `-Port` | original Host / port | For building absolute URLs |
| `Forwarded` (RFC 7239) | `for=;proto=;host=` in one header | The standard; adoption is partial |
| PROXY protocol | client IP at L4 (pre-HTTP bytes) | For TCP proxies that never parse HTTP |

**Cheat block:**
```
# Minimal correct reverse-proxy location:
location / {
    proxy_pass http://127.0.0.1:3000;
    proxy_set_header Host              $host;
    proxy_set_header X-Real-IP         $remote_addr;
    proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
# WebSocket add-ons:  proxy_http_version 1.1;
#                     proxy_set_header Upgrade $http_upgrade;
#                     proxy_set_header Connection "upgrade";

# App side (Express):  app.set("trust proxy", 1);   // NOT true
#   → req.ip = real client,  req.protocol = original scheme

# Golden rules:
#   - Backend socket ALWAYS sees the proxy's IP (often 127.0.0.1).
#   - Real client IP lives in X-Forwarded-For — read it, don't trust it blindly.
#   - X-Forwarded-Proto or infinite redirect loops behind TLS termination.
#   - Bind the app to 127.0.0.1 + firewall the port so only the proxy reaches it.
nginx -t              # validate config
nginx -s reload       # apply without downtime
```

---

## When Would I Use This at Work?

### Scenario 1: "Standing up a new Node/Python service in production"
You never expose the app process to the internet. You put nginx (or Caddy/Envoy) in front on 443, terminate TLS there, bind your app to `127.0.0.1:PORT`, and inject `X-Forwarded-For` / `X-Forwarded-Proto`. Day one you configure `trust proxy` so logs, rate limiting, and auth see real client IPs and the correct scheme.

### Scenario 2: "Our per-IP rate limiter is blocking all users at once"
You immediately suspect the app is keying the limiter on `req.socket.remoteAddress` (= the proxy) instead of the forwarded client IP. You verify with `curl` + logs, set `trust proxy` to the exact number of hops, and key the limiter on `req.ip`.

### Scenario 3: "Auth redirects loop, or OAuth callbacks come back as http:// on our https:// site"
Classic missing/ignored `X-Forwarded-Proto`. You add the header in nginx and make the framework honor it, so absolute URLs, `Secure` cookies, and HSTS behave.

### Scenario 4: "We're adding real-time chat and it works locally but not in prod"
You know the proxy strips hop-by-hop upgrade headers by default. You add `proxy_http_version 1.1`, forward `Upgrade`/`Connection`, and bump `proxy_read_timeout` so idle sockets survive.

### Scenario 5: "Consolidating three services under one domain"
You use path-based routing (`/api` → svc A, `/img` → svc B, `/` → the web app) in a single nginx front door, hiding the backend topology behind one public entry point and one TLS certificate.

---

## Connected Topics

**Study before this:**
- `01-how-the-internet-works.md` — the client→server path the proxy inserts itself into
- `10-http1.1-in-depth.md` — the request/response and headers the proxy reads and rewrites
- `28-load-balancers.md` — L4 vs L7 and how distributing traffic across backends relates to proxying

**Study alongside / after:**
- `09-tls-ssl-in-depth.md` — what "TLS termination" actually terminates
- `13-websockets.md` — the HTTP Upgrade handshake the proxy must pass through
- `18-rate-limiting-at-http-level.md` — why correct client-IP extraction is load-bearing here
- `34-api-gateways.md` — a reverse proxy grown up: adds auth offloading, routing, and request transformation
- `36-service-to-service-communication.md` — Envoy/sidecars: reverse proxies inside a service mesh, with mTLS

**Key takeaway:** a reverse proxy splits one client→server connection into two, so any information bound to the connection — the client IP, the scheme, the upgrade intent — must be *deliberately* carried across the gap in HTTP headers, and *deliberately* trusted only from hops you control.

---

*Doc saved: `/docs/networking/29-reverse-proxies.md`*
