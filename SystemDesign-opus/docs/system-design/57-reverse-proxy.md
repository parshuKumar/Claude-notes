# 57 — Reverse Proxy
## Category: HLD Components

---

## What is this?

A **reverse proxy** is a server that sits in front of your application servers and takes every incoming request on their behalf. The client thinks it's talking to your app. It's actually talking to the proxy, which then forwards the request to whichever real server should handle it — and forwards the response back.

Think of the **receptionist at a corporate office**. You walk in and say "I need to see someone about a refund." You don't know, and don't need to know, that Refunds is on floor 7, desk 12, and that his name is Dave. The receptionist knows. She checks your ID, takes your coat, tells you Dave's out today so Priya will handle it, and sends you to the right place. From the street, the entire company looks like one desk with one person at it. That receptionist is a reverse proxy.

---

## Why does it matter?

**Interview angle:** "What's the difference between a forward proxy and a reverse proxy?" is one of the most common warm-up questions in infrastructure interviews, and it is a **trap** — the words are almost identical, and candidates who half-remember them get it backwards. The follow-up, "how is a reverse proxy different from a load balancer?", separates people who've deployed things from people who've only read about them.

**Real work angle:** Essentially every production web application on earth has a reverse proxy in front of it. If you deploy a Node.js app, you put Nginx (or a cloud ALB, or Cloudflare, or Envoy) in front of it — and if you can't explain *why*, you'll do it as cargo cult. The concrete things it buys you:

- Your Node process doesn't have to handle TLS, which is CPU-expensive and security-sensitive.
- Your Node process doesn't waste its single thread serving `logo.png`.
- A user on a 2G connection can't tie up one of your app's precious connections for 40 seconds.
- You can swap, restart, and redeploy your app servers without dropping a single client connection.

Without it, your app is exposed directly to the internet, doing jobs it's bad at, with your internal topology visible to anyone with `curl`.

---

## The core idea — explained simply

### The Two Receptionists Analogy

There are two completely different kinds of middlemen, and the *only* thing that distinguishes them is **which side they're protecting.**

**The forward proxy = your company's mailroom (protects/serves the CLIENT).**

You work at a company. You want to order a book. Company policy says all outbound mail goes through the mailroom. You hand your order to the mailroom clerk; *he* posts it, using the company's return address. The bookstore receives an order from "Acme Corp, 1 Business Park" — it has no idea *you* specifically ordered it. The clerk can also refuse: "Sorry, we don't allow orders from gambling sites." And if three colleagues order the same book, he might just photocopy the one he already has.

The mailroom clerk **works for you and the other employees**. He knows who you all are. The outside world doesn't. He is a **forward proxy** — a VPN, a corporate web filter, your company's outbound HTTP proxy.

**The reverse proxy = the corporate receptionist (protects/serves the SERVER).**

Now you're a visitor arriving at that same company. You give your request to the receptionist. She decides which of the 40 employees actually handles it. You never learn how many employees there are, what floor they're on, or their names. If Dave is sick, she quietly routes you to Priya and you never notice. She also handles the boring universal stuff so the employees don't have to: signs you in, checks your ID, gives you the standard brochure from the stack on her desk instead of bothering an employee for it.

The receptionist **works for the company**. She knows all the employees. You, the visitor, don't. She is a **reverse proxy**.

### The one sentence that fixes it forever

> A **forward** proxy hides the **client** from the server.
> A **reverse** proxy hides the **server** from the client.

If you remember nothing else, remember: **whoever the proxy is standing in front of, is who it hides.** A forward proxy stands in front of clients. A reverse proxy stands in front of servers.

### Mapping the analogy

| Office | Forward proxy | Reverse proxy |
|---|---|---|
| Who installs it | The **client's** organisation (or the client themselves) | The **server's** organisation |
| Who it serves | The client | The server |
| Who it hides | The **client's** identity/IP | The **servers'** identity/count/topology |
| Who knows it exists | The client (configured it deliberately) | Nobody — the client thinks it *is* the server |
| Real examples | VPN, corporate web filter, Tor, `squid`, an SSH tunnel | Nginx, HAProxy, Envoy, Cloudflare, AWS ALB |
| Typical motive | Privacy, bypassing geo-blocks, corporate policy, caching outbound | Security, TLS offload, caching, load balancing, hiding topology |

---

## Key concepts inside this topic

### 1. Forward proxy vs reverse proxy — the distinction, precisely

**Forward proxy.** The client is *configured* to use it. Your browser or OS says "send all HTTP through `proxy.acme.com:3128`." The client knows the proxy is there — it deliberately opted in. The destination server sees the request arriving from the proxy's IP and has no idea who the real client is.

Uses: corporate egress filtering ("no social media at work"), VPNs (hide your IP from websites), bypassing geo-restrictions, caching outbound requests so 500 employees downloading the same OS update only fetch it once, and logging what employees access.

**Reverse proxy.** The client is *not* configured for anything and has no idea it exists. It types `https://api.example.com`, DNS resolves that name to **the reverse proxy's IP**, and the proxy answers. The real application servers sit on a private network with private IPs that are not routable from the internet at all. The client cannot reach them even if it wanted to.

Uses: everything in section 3 below.

**The clean test:** *Who chose to put the proxy there?*
- The client chose → forward proxy.
- The server owner chose → reverse proxy.

The same piece of software (Nginx, Squid, Envoy) can do both jobs. The words describe the **position and purpose**, not the binary.

### 2. What a reverse proxy actually does — the eight jobs

**a) TLS/SSL termination.** The proxy holds the certificate and private key, does the expensive TLS handshake, decrypts the request, and forwards **plain HTTP** to your app over the private network. Your Node app never sees a certificate.

Why this matters: TLS handshakes are CPU-heavy (asymmetric crypto). Nginx does them in optimised C with hardware acceleration; Node does them in JavaScript-land on its single event loop. Also: certificate renewal, cipher-suite policy, and TLS-version deprecation now happen in **one** place instead of in every service. When a new TLS vulnerability lands, you patch one box.

**b) Compression.** The proxy gzip/brotli-compresses responses before sending them. A 200 KB JSON response becomes ~20 KB — a **10x** bandwidth reduction, and much faster for mobile users. Compression is CPU work; better it happens on the proxy's spare cores than on your app's event loop.

**c) Static file serving.** `logo.png`, `app.js`, `styles.css` — Nginx serves these directly from disk using `sendfile()`, a kernel call that copies the file straight from the page cache to the socket **without the data ever entering user space.** Node.js reading a file and streaming it is orders of magnitude more expensive. Never make Node serve static assets in production.

**d) Caching.** The proxy can store responses and serve repeats itself, never touching your app. Cache `/api/products` for 60 seconds and a 10,000 req/s endpoint becomes ~16 req/s hitting Node. See [59 — Caching in Depth](./59-caching-in-depth.md).

**e) Request routing.** Look at the path/host/header and send the request to a different backend. `/api/*` → the API servers. `/admin/*` → the admin service. `/ws` → the WebSocket service. Everything else → the static SPA. One public hostname, many internal services.

**f) Header manipulation.** Add, strip, or rewrite headers. Add `X-Forwarded-For` (so your app can still learn the client's real IP — critical, see below). Add security headers (`Strict-Transport-Security`, `X-Content-Type-Options`). **Strip** headers your app leaks, like `X-Powered-By: Express` or `Server: Node.js`, which tell attackers exactly what to exploit.

**g) Hiding internal topology.** The client sees one IP and one hostname. It cannot tell whether there are 2 servers or 200 behind it, what ports they're on, or that the payments service is a Go binary while the rest is Node. Your private subnet is genuinely unreachable from the internet. This shrinks your attack surface enormously.

**h) Buffering slow clients.** This one is subtle, it's the most Node-specific benefit, and it is a *great* thing to bring up unprompted in an interview.

Imagine a user on a terrible 2G connection uploading a 5 MB file. Without a proxy, that upload trickles into your Node server over **40 seconds**, holding an open connection and an in-flight request handler the entire time. A few thousand such clients (accidental or malicious — this is literally the *Slowloris* DoS attack) and your app is exhausted, unable to serve anyone.

With a proxy in front: **Nginx** absorbs the slow trickle. It patiently buffers the whole request in its own memory/disk over those 40 seconds — Nginx is event-driven C and can hold tens of thousands of idle connections for almost nothing. Only when the request is **complete** does it forward it to Node, over the fast local network, in **2 milliseconds**. Node's connection is occupied for 2ms instead of 40,000ms.

The same thing happens in reverse: Node writes the response to Nginx instantly and moves on; Nginx dribbles it out to the slow client. Your expensive app worker is freed immediately.

**That is a ~20,000x reduction in how long a slow client can occupy your application.** This alone justifies the proxy.

### 3. Nginx configuration — real and readable

Here is a production-shaped Nginx config. Read the comments; every block is one of the eight jobs above.

```nginx
# ── The pool of Node.js app servers we proxy to ──────────────────────────────
upstream node_app {
    least_conn;                       # send each request to the backend with fewest active conns
    server 10.0.1.10:3000 max_fails=3 fail_timeout=30s;
    server 10.0.1.11:3000 max_fails=3 fail_timeout=30s;
    server 10.0.1.12:3000 max_fails=3 fail_timeout=30s;
    keepalive 64;                     # reuse TCP conns to Node — avoids a handshake per request
}

# ── Response cache: 100MB of keys in RAM, up to 10GB of bodies on disk ───────
proxy_cache_path /var/cache/nginx keys_zone=api_cache:100m max_size=10g inactive=60m;

# ── Port 80: do nothing but push everyone to HTTPS ───────────────────────────
server {
    listen 80;
    server_name example.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name example.com;

    # ── (a) TLS TERMINATION — the cert lives HERE, never in Node ─────────────
    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    ssl_protocols       TLSv1.2 TLSv1.3;     # TLS 1.0/1.1 are dead; do not enable them
    ssl_session_cache   shared:SSL:10m;      # resume handshakes → big CPU saving on repeat visitors

    # ── (b) COMPRESSION ──────────────────────────────────────────────────────
    gzip              on;
    gzip_types        application/json application/javascript text/css text/plain;
    gzip_min_length   1024;   # below ~1KB, compression costs more than it saves

    # ── (f) HEADER MANIPULATION: security headers + hide what we're running ───
    add_header Strict-Transport-Security "max-age=31536000" always;
    add_header X-Content-Type-Options    "nosniff" always;
    server_tokens off;                       # don't advertise the Nginx version

    client_max_body_size 10m;                # reject huge uploads AT THE EDGE, before Node sees them

    # ── (c) STATIC FILES — served by Nginx via sendfile(), Node never involved ─
    location /static/ {
        root /var/www;
        expires 30d;                         # let the browser cache these for a month
        access_log off;                      # don't spam the log with 10k asset hits
    }

    # ── (d) CACHED, cacheable API reads ──────────────────────────────────────
    location /api/products {
        proxy_cache            api_cache;
        proxy_cache_valid      200 60s;      # cache successful responses for 60 seconds
        proxy_cache_use_stale  error timeout updating;  # serve stale if Node is down — huge for uptime
        proxy_cache_lock       on;           # on a cache miss, only ONE request goes to Node,
                                             # the rest wait. This prevents cache stampede.
        add_header X-Cache-Status $upstream_cache_status;  # HIT / MISS / STALE — invaluable for debugging
        proxy_pass http://node_app;
    }

    # ── (e) ROUTING + (h) BUFFERING: everything else goes to Node ────────────
    location / {
        proxy_pass http://node_app;

        # Node MUST be told the real client details, because from its point of
        # view every request now arrives from Nginx's private IP.
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;   # so Node knows it was HTTPS

        # (h) BUFFERING — the slow-client shield.
        proxy_request_buffering  on;   # read the WHOLE request from the slow client first
        proxy_buffering          on;   # take the WHOLE response from Node, then dribble it out
        proxy_buffers            16 4k;

        proxy_connect_timeout 5s;
        proxy_read_timeout   60s;
    }

    # ── WebSockets need the HTTP/1.1 Upgrade dance ───────────────────────────
    location /ws {
        proxy_pass http://node_app;
        proxy_http_version 1.1;
        proxy_set_header Upgrade    $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_read_timeout 3600s;       # WS connections are long-lived; don't kill them at 60s
    }
}
```

And the Node side, which must be told to **trust** what the proxy says:

```javascript
import express from 'express';
const app = express();

// Without this, req.ip is Nginx's private IP (10.0.1.1) for EVERY request —
// so rate limiting by IP would throttle all your users as if they were one person,
// and your logs would be useless. This tells Express to read X-Forwarded-For.
app.set('trust proxy', 1);   // 1 = trust exactly one hop (our Nginx). NEVER use `true`
                             // on a public-facing app: a client could forge X-Forwarded-For.

app.disable('x-powered-by'); // belt and braces — Nginx strips it too

app.get('/api/products', async (req, res) => {
  const products = await db.getProducts();
  // Tell Nginx (and browsers) this is cacheable. Nginx honours Cache-Control.
  res.set('Cache-Control', 'public, max-age=60');
  res.json(products);
});

app.get('/whoami', (req, res) => {
  res.json({
    ip: req.ip,                    // real client IP — thanks to trust proxy
    secure: req.secure,            // true, read from X-Forwarded-Proto
  });
});

// Bind to the PRIVATE interface only. This app must be unreachable from the
// internet; the only way in is through Nginx.
app.listen(3000, '10.0.1.10');
```

The `trust proxy` line is a genuinely common production bug. Get it wrong in one direction and every user shares one IP (rate limiting breaks, geolocation breaks, abuse logging breaks). Get it wrong in the other direction — `trust proxy: true` with no proxy in front — and any attacker can spoof `X-Forwarded-For` to bypass your IP rate limits entirely.

### 4. Reverse proxy vs load balancer — the overlap explained

This confuses people because the answer is genuinely "they overlap." Here's the precise version:

> **A load balancer does exactly one job: distribute traffic across many backends. A reverse proxy does that job *and* seven others. Load balancing is a feature of reverse proxies.**

So:
- **Every load balancer is (or acts as) a reverse proxy** — it sits in front of your servers and answers on their behalf. That's the definition.
- **Not every reverse proxy load-balances.** A reverse proxy in front of exactly **one** backend is still a reverse proxy — it's still terminating TLS, compressing, caching, and buffering. It just has nothing to balance.

| | Load balancer | Reverse proxy |
|---|---|---|
| **Core purpose** | Spread traffic; keep the fleet healthy | Be the front door; do everything a server shouldn't |
| **Needs 2+ backends?** | Yes — that's the whole point | No — useful with one |
| **TLS termination** | Usually yes (L7 ones) | Yes |
| **Caching** | Rarely | Yes |
| **Static files** | No | Yes |
| **Compression** | Sometimes | Yes |
| **Slow-client buffering** | Sometimes | Yes |
| **Can operate at L4 (TCP)?** | Yes — many do, for raw speed | Usually L7 (it needs to *read* HTTP to do its jobs) |
| **Examples** | AWS NLB, HAProxy (in LB mode), keepalived/LVS | Nginx, Envoy, Varnish, Cloudflare |

**Nginx is both, simultaneously.** In the config above, the `upstream node_app` block with three servers and `least_conn` **is** load balancing. The `gzip`, `proxy_cache`, `ssl_certificate`, and `location /static/` blocks are the reverse-proxy jobs. Same process, same request, both hats.

The honest interview answer: *"A load balancer is one job that a reverse proxy can do. Nginx is a reverse proxy that happens to also load balance. An AWS NLB is a load balancer that doesn't do the other reverse-proxy jobs. In practice the words are used loosely, and I'd clarify which capabilities we actually need rather than argue about the label."*

### 5. Why you put Nginx in front of Node.js specifically

Node deserves its own section because the reasons are sharper than for other runtimes.

**a) Node is single-threaded.** One event loop per process. **Every millisecond spent on TLS handshakes, gzip, or reading files off disk is a millisecond not spent running your business logic.** Nginx is multi-process, event-driven C, engineered for exactly this work. You are moving cheap work off your most expensive thread.

**b) Slow clients are lethal to an event loop.** As covered in §2h — a slow client holds a Node connection open for its entire duration. Nginx's buffering makes Node's exposure to a 40-second slow client about 2 milliseconds. For an event-loop runtime this is not an optimisation; it's a survival mechanism.

**c) Zero-downtime deploys.** Nginx health-checks your Node instances. You restart instance #1; Nginx notices it's down and routes everything to #2 and #3; instance #1 comes back; Nginx resumes sending to it. Repeat. **No client ever sees an error.** Your Node process, alone, cannot do this for itself.

**d) `sendfile()` for static assets.** Node reading a file means: disk → kernel buffer → V8 Buffer (user space) → back to the kernel → socket. Nginx's `sendfile()` goes disk → socket, inside the kernel, never crossing into user space. It's not a little faster, it's a different category.

**e) Serve stale when Node is dead.** `proxy_cache_use_stale error timeout` means: if every Node instance is down, Nginx serves the last cached response rather than a 502. Your users see slightly old data instead of an error page. That's a real availability win for free.

**f) Node should never be internet-facing.** Node's HTTP parser is fine, but Nginx has had two decades of hostile-internet hardening: malformed request handling, header-size limits, connection floods, request-size limits (`client_max_body_size` rejects a 2 GB upload **before a single byte reaches Node**). Let the hardened C proxy meet the internet.

**The one honest counterpoint:** if you deploy to a cloud load balancer (AWS ALB), a PaaS (Heroku, Render, Fly), or a Kubernetes ingress, **you already have a reverse proxy** — it's just managed for you, and it's already doing TLS, buffering, and health checks. Adding Nginx *inside* your container as well is usually redundant. The rule is "always have a reverse proxy," not "always run Nginx yourself."

---

## Visual / Diagram description

### Diagram 1 — FORWARD proxy (sits in front of the CLIENT, hides the CLIENT)

```
      ┌── YOUR COMPANY'S NETWORK (the proxy's owner) ──┐
      │                                                 │
      │  ┌──────────┐                                   │
      │  │ Laptop A │──┐                                │
      │  │ 10.0.0.5 │  │                                │            THE INTERNET
      │  └──────────┘  │                                │
      │                │   ┌──────────────────────┐     │        ┌──────────────────┐
      │  ┌──────────┐  ├──▶│    FORWARD PROXY     │─────┼───────▶│  google.com      │
      │  │ Laptop B │──┤   │   proxy.acme.com     │     │        │                  │
      │  │ 10.0.0.6 │  │   │                      │     │        │ Sees ONE client: │
      │  └──────────┘  │   │ • knows all clients  │     │        │  203.0.113.9     │
      │                │   │ • filters/blocks     │     │        │ (the proxy's IP) │
      │  ┌──────────┐  │   │ • caches outbound    │     │        │                  │
      │  │ Laptop C │──┘   │ • logs who went where│     │        │ Has NO IDEA that │
      │  │ 10.0.0.7 │      │                      │     │        │ Laptop A exists. │
      │  └──────────┘      │ Public IP:203.0.113.9│     │        └──────────────────┘
      │                    └──────────────────────┘     │
      └─────────────────────────────────────────────────┘

  ▲ The CLIENTS configured this. They know it's there.
  ▲ It HIDES THE CLIENTS from the server.
  ▲ Real: VPN, corporate web filter, Tor, Squid.
```

### Diagram 2 — REVERSE proxy (sits in front of the SERVER, hides the SERVERS)

```
        THE INTERNET                    ┌── YOUR PRIVATE NETWORK (the proxy's owner) ──┐
                                        │                                               │
  ┌──────────┐                          │                        ┌──────────────────┐   │
  │ Browser  │──┐                       │                   ┌───▶│  Node App #1     │   │
  │ (Tokyo)  │  │                       │                   │    │  10.0.1.10:3000  │   │
  └──────────┘  │   ┌───────────────────┴────┐              │    └──────────────────┘   │
                │   │    REVERSE PROXY       │              │                           │
  ┌──────────┐  │   │       (Nginx)          │              │    ┌──────────────────┐   │
  │  Mobile  │──┼──▶│   example.com          │──────────────┼───▶│  Node App #2     │   │
  │ (Berlin) │  │   │   198.51.100.7 :443    │              │    │  10.0.1.11:3000  │   │
  └──────────┘  │   │                        │              │    └──────────────────┘   │
                │   │  (a) TLS termination   │              │                           │
  ┌──────────┐  │   │  (b) gzip / brotli     │              │    ┌──────────────────┐   │
  │   API    │──┘   │  (c) static files      │              └───▶│  Node App #3     │   │
  │  client  │      │  (d) response cache    │                   │  10.0.1.12:3000  │   │
  └──────────┘      │  (e) routing           │                   └──────────────────┘   │
                    │  (f) header rewrite    │                                          │
   Clients see      │  (g) hides topology    │                   ┌──────────────────┐   │
   ONE server:      │  (h) buffers slow      │──────────────────▶│  /var/www/static │   │
   example.com      │      clients           │   sendfile()      │  (served direct) │   │
                    └────────────────────────┘                   └──────────────────┘   │
                                        │                                               │
                                        │  These IPs are NOT routable from the internet.│
                                        │  There is NO other way in.                    │
                                        └───────────────────────────────────────────────┘

  ▲ The SERVER OWNER installed this. Clients have no idea it exists.
  ▲ It HIDES THE SERVERS from the client.
  ▲ Real: Nginx, HAProxy, Envoy, Cloudflare, AWS ALB.
```

### Diagram 3 — The slow-client buffering win

```
  WITHOUT a reverse proxy                    WITH a reverse proxy
  ═════════════════════════                  ══════════════════════

  Slow 2G client                             Slow 2G client
       │                                          │
       │  5MB upload, dribbling                   │  5MB upload, dribbling
       │  ....................                    │  ....................
       │       40 SECONDS                         │       40 SECONDS
       ▼                                          ▼
  ┌─────────────────┐                       ┌──────────────────────────┐
  │   NODE.JS       │                       │        NGINX              │
  │                 │                       │  patiently buffers the    │
  │ ✗ connection    │                       │  whole 5MB. Costs almost  │
  │   held OPEN     │                       │  nothing — event-driven C │
  │   40,000 ms     │                       │  holds 10k idle conns.    │
  │                 │                       └──────────┬───────────────┘
  │ ✗ event loop    │                                  │
  │   slot occupied │                                  │ forwards the COMPLETE
  │                 │                                  │ request over the fast LAN
  │ 5,000 such      │                                  ▼
  │ clients = DEAD  │                       ┌──────────────────────────┐
  │ (Slowloris)     │                       │        NODE.JS            │
  └─────────────────┘                       │  connection held: 2 ms    │
                                            │  ✓ event loop free        │
                                            └──────────────────────────┘

               40,000 ms  ──────▶  2 ms       (~20,000x less exposure)
```

**How to read these:** in Diagram 1, the proxy is inside the *client's* network boundary. In Diagram 2, it's inside the *server's*. That boundary is the entire distinction — everything else follows from it.

---

## Real world examples

### 1. Cloudflare — a reverse proxy in front of a large fraction of the web

Cloudflare's core product is a globally distributed reverse proxy. You point your domain's DNS at Cloudflare's IPs instead of your server's. Every request from every user now hits a Cloudflare edge server first.

From there Cloudflare does the classic reverse-proxy jobs at planetary scale: it terminates TLS at the edge (so the handshake happens ~20ms from the user instead of across an ocean), caches static assets and cacheable responses, compresses responses, filters malicious traffic (a WAF and DDoS scrubbing), and only then forwards what's left to your origin server.

Crucially, it **hides your origin's IP address entirely.** An attacker who wants to DDoS you can't find the machine — they can only hit Cloudflare, which is engineered to absorb it. That's the "hides the servers" property turned into a security product.

### 2. Nginx in a standard Node.js deployment

The single most common production topology on the internet, and the one you'll build:

```
Internet → Nginx (:443, TLS, gzip, static, cache, buffering)
              ↓ plain HTTP over localhost/private LAN
           PM2 / systemd running N Node.js processes (:3000, :3001, :3002)
              ↓
           Postgres + Redis (private network only)
```

Nginx terminates TLS with a Let's Encrypt cert (auto-renewed by certbot), serves the React build folder directly from disk, proxies `/api/*` to the Node cluster, and buffers every slow mobile client so the event loop stays free. The Node processes bind to a private interface and are unreachable from the internet. This is the boring, correct default.

### 3. Kubernetes Ingress — the reverse proxy as a platform primitive

In Kubernetes you don't hand-write Nginx configs; you declare an **Ingress** resource ("host `api.example.com`, path `/api` → service `api-svc`"). An **ingress controller** — very often *literally Nginx*, or Envoy via Istio/Contour — reads that declaration and generates the reverse-proxy configuration for you.

The interesting part: the reverse proxy has become so fundamental that Kubernetes made it a first-class API object. When people talk about a "service mesh," they're describing a reverse proxy (Envoy) placed next to *every single service* as a sidecar, doing TLS, retries, routing, and metrics for east-west traffic too. See [88 — Containers and Orchestration](./88-containers-and-orchestration.md).

---

## Trade-offs

| Benefit of a reverse proxy | The cost you pay |
|---|---|
| TLS offload — app never touches certs | One more component to run, patch, and monitor |
| Caching — massive load reduction | Cache invalidation is genuinely hard; stale data bugs |
| Static file serving via `sendfile()` | Static assets must be reachable from the proxy box |
| Buffers slow clients (~20,000x less app exposure) | Adds a small latency hop (~1–2ms) to every request |
| Hides internal topology; shrinks attack surface | The proxy is now the new single point of failure — it must be HA |
| Central place for security headers, WAF, rate limits | A misconfigured proxy takes the *whole* site down, not one service |
| Health checks → zero-downtime deploys | `X-Forwarded-For` / `trust proxy` misconfigurations are a classic bug class |
| Compression — ~10x less bandwidth | CPU cost on the proxy; not worth it below ~1 KB responses |

| Where to put it | When that's right |
|---|---|
| **Nginx on your own box** | VPS/bare metal; you want caching, static serving, full control |
| **Cloud LB (ALB / GCLB)** | You're already in the cloud — TLS, health checks, buffering are managed for you |
| **CDN (Cloudflare, Fastly)** | Global audience; you want edge caching + DDoS protection ([60 — CDN](./60-cdn.md)) |
| **K8s Ingress / service mesh** | You're on Kubernetes; use the platform primitive |
| **None** | Local development only. Never production. |

**Rule of thumb:** **Always** have a reverse proxy in front of your app — but you often don't have to *run* one yourself. If you're on a cloud LB, a PaaS, or a K8s ingress, you already have one; adding Nginx inside your container too is usually redundant complexity. If you're on a plain VPS with Node, put Nginx in front. And never, ever expose a Node process directly to the internet.

---

## Common interview questions on this topic

### Q1: "What's the difference between a forward proxy and a reverse proxy?"
**Hint:** The one-liner: **a forward proxy hides the client from the server; a reverse proxy hides the server from the client.** A forward proxy sits in front of clients, is installed by the client's side, and the client knows it's there (VPN, corporate filter). A reverse proxy sits in front of servers, is installed by the server owner, and the client has no idea it exists (Nginx, Cloudflare, ALB). The test: *who chose to put it there?*

### Q2: "How is a reverse proxy different from a load balancer?"
**Hint:** They overlap heavily. **Load balancing is one of the jobs a reverse proxy can do.** A reverse proxy also does TLS termination, caching, compression, static files, buffering, and header manipulation. A reverse proxy in front of a *single* backend is still useful; a load balancer with one backend is pointless. Nginx is both at once. AWS NLB is a pure load balancer (L4, doesn't read HTTP). Don't argue the label — clarify which capabilities are needed.

### Q3: "Why do people put Nginx in front of a Node.js app? Node can serve HTTP itself."
**Hint:** Node is **single-threaded** — every ms on TLS, gzip, or file I/O is stolen from your business logic. Nginx is event-driven C built for exactly that work. The killer reason is **buffering slow clients**: without a proxy, a 2G client uploading 5 MB occupies a Node connection for 40 seconds (this is the Slowloris attack); with Nginx buffering, Node's exposure is ~2ms. Add: `sendfile()` for static assets, zero-downtime deploys via health checks, `serve-stale` when Node is down, and keeping an internet-hardened C proxy — not Node — as the thing facing hostile traffic.

### Q4: "Your app is behind Nginx and suddenly every user appears to have the same IP address. What happened?"
**Hint:** Node is seeing Nginx's private IP (e.g. `10.0.1.1`), because that's genuinely who the TCP connection is from. Nginx must send `X-Forwarded-For: $proxy_add_x_forwarded_for`, and Express must be told to trust it: `app.set('trust proxy', 1)`. The `1` matters — `trust proxy: true` on a public app lets an attacker **forge** `X-Forwarded-For` and walk straight through your IP-based rate limiter. Set the hop count to exactly the number of proxies you control.

### Q5: "Doesn't the reverse proxy just become a new single point of failure?"
**Hint:** Yes — and you must say so before the interviewer does. You fix it the same way you fix any SPOF: run **at least two** proxy nodes, either behind a floating/virtual IP (keepalived), behind DNS round-robin with health checks, or by using a managed proxy that's internally redundant (ALB, Cloudflare). The trade is worth it: one hardened, replicated component at the edge is far safer than exposing N app servers directly.

---

## Practice exercise

### Build and Prove a Reverse Proxy

Spend 30–40 minutes. You will end with a working local setup and a measured before/after.

**Setup:**
1. Write a tiny Node/Express app on port `3000` with three routes:
   - `GET /api/time` → returns `{ now: Date.now(), servedBy: process.env.INSTANCE }`
   - `GET /api/slow` → `await` a 2-second delay, then respond
   - `GET /whoami` → returns `req.ip` and `req.secure`
2. Run **two** copies of it (ports 3000 and 3001) with different `INSTANCE` env values.
3. Put Nginx in front on port 8080 (Docker is fine: `nginx:alpine` with a mounted config).

**Configure Nginx to do all of these — one config block per item:**
- `upstream` with **both** Node instances, `least_conn`.
- Proxy `/api/*` to the upstream, setting `X-Real-IP`, `X-Forwarded-For`, `X-Forwarded-Proto`.
- **Cache** `/api/time` for 10 seconds, and add `X-Cache-Status $upstream_cache_status` so you can see HIT/MISS.
- **gzip** JSON responses.
- Serve a `/static/hello.txt` file directly from disk — **Node must never see the request.**
- Set `client_max_body_size 1m`.

**Then prove each one:**
1. `curl -i localhost:8080/api/time` **six times.** Watch `X-Cache-Status` go `MISS` then `HIT, HIT, HIT...`, and confirm the `now` value **freezes** for 10 seconds. Check both Node logs: only ONE of them saw a request.
2. Hit `/api/time` many times with the cache off — confirm `servedBy` alternates between your two instances. **That's load balancing.**
3. Add `app.set('trust proxy', 1)` to Express. Compare `GET /whoami` **before and after** — watch `req.ip` change from Nginx's IP to your real one.
4. `curl -H "Accept-Encoding: gzip" -i localhost:8080/api/time` — confirm `Content-Encoding: gzip`.
5. Try to POST a 5 MB body. Confirm Nginx returns **413** and that **nothing at all appears in the Node logs** — the request died at the edge.
6. **Kill both Node processes.** Now request `/api/time` again with `proxy_cache_use_stale error timeout;` set. Confirm you get a **stale 200 instead of a 502.** Feel how good that is.

**Deliverable:** your `nginx.conf`, plus a short list of what each of the 6 tests proved.

---

## Quick reference cheat sheet

- **Forward proxy hides the CLIENT** from the server (VPN, corporate filter). The **client** installs it and knows it's there.
- **Reverse proxy hides the SERVER** from the client (Nginx, Cloudflare, ALB). The **server owner** installs it; the client has no idea.
- **The test:** who chose to put the proxy there? Client → forward. Server owner → reverse.
- **The eight jobs:** TLS termination, compression, static file serving, caching, routing, header manipulation, hiding topology, buffering slow clients.
- **Load balancing is ONE job a reverse proxy can do.** A reverse proxy does more. **Nginx is both.**
- **Buffering slow clients** is the killer feature for Node: a 40-second slow upload occupies Node for ~2ms instead of 40,000ms. Defends against Slowloris.
- **Node is single-threaded** — never make it do TLS, gzip, or serve static files. That's stolen event-loop time.
- **`sendfile()`** lets Nginx push a file from disk to socket entirely inside the kernel. Node cannot come close.
- **`proxy_cache_use_stale`** serves the last cached response when your app is completely down. Free availability.
- **`X-Forwarded-For` + `app.set('trust proxy', 1)`** — without both, every user looks like one IP and rate limiting breaks. `trust proxy: true` on a public app is a security hole (clients can forge the header).
- **`client_max_body_size`** rejects oversized uploads at the edge, before a byte reaches your app.
- **The proxy is a new SPOF** — run 2+, behind a virtual IP or a managed LB.
- **Never expose Node directly to the internet.** Always something hardened in front.
- **You often already have one** — an ALB, a PaaS router, or a K8s ingress *is* a reverse proxy. Don't stack a redundant Nginx inside it.

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [56 — Horizontal vs Vertical Scaling](./56-horizontal-vs-vertical-scaling.md) — the reverse proxy is what fronts the fleet you just scaled out |
| **Next** | [58 — API Gateway](./58-api-gateway.md) — a reverse proxy that becomes API-aware: auth, rate limits, aggregation |
| **Related** | [55 — Load Balancing](./55-load-balancing.md) — the one job a reverse proxy shares with a dedicated LB |
| **Related** | [60 — CDN](./60-cdn.md) — a globally distributed reverse-proxy cache sitting close to your users |
