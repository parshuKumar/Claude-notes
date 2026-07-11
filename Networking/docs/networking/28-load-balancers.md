# 28 — Load Balancers

> **Phase 5 — Topic 1 of 7**
> Prerequisites: `01-how-the-internet-works.md`, `08-tcp-in-depth.md`, `10-http1.1-in-depth.md`

---

## ELI5 — The Simple Analogy

Imagine a popular restaurant with one front door but ten identical kitchens in the back.

- If every customer had to know which kitchen is free and walk straight to it, chaos. Some kitchens would be swamped while others sat idle. And if a kitchen caught fire, customers walking toward it would just... stand there confused.
- Instead, there's a **host at the front door** (the **load balancer**). Every customer talks only to the host. The host says "Kitchen 4 is free, go there," and quietly keeps track of which kitchens are busy, which are broken, and which are fastest today.
- If Kitchen 4 catches fire, the host simply stops sending anyone there. Customers never notice. When Kitchen 4 is fixed, the host starts using it again.

The customers only ever memorize **one address**: the front door. The kitchens can be added, removed, or replaced at any time and nobody outside ever needs to know.

That front-door host is a load balancer. The single memorized address is the **VIP** (virtual IP). The kitchens are your **backend servers**.

---

## Where This Lives in the Network Stack

A load balancer can operate at **two different layers**, and which layer it picks changes *everything* about what it can do.

```
┌─────────────────────────────────────────────────────────────────┐
│  Layer 7 — Application   (HTTP, gRPC, WebSocket)                │  ◄── L7 LB works HERE
│                          reads Host, path, headers, cookies      │      (ALB, nginx, HAProxy,
│  Layer 6 — Presentation  (TLS/SSL)                              │       Envoy, Traefik)
│  Layer 5 — Session                                              │
│  Layer 4 — Transport     (TCP, UDP — ports)                     │  ◄── L4 LB works HERE
│                          reads IP:port only, never the payload   │      (AWS NLB, IPVS/LVS,
│  Layer 3 — Network       (IP)                                   │       most hardware LBs)
│  Layer 2 — Data Link     (Ethernet)                             │
│  Layer 1 — Physical                                             │
└─────────────────────────────────────────────────────────────────┘
```

- An **L4 load balancer** makes its decision by looking only at Layer 3/4 information — source/destination IP and port. It never reads the bytes inside. It is a very fast, very dumb traffic cop.
- An **L7 load balancer** *terminates* the connection, reads the actual HTTP request (Layer 7), and can route on the URL path, `Host` header, a cookie, or anything else in the request. It is slower per request but vastly smarter.

The rest of this topic is largely about the consequences of that one choice.

---

## What Is This?

A **load balancer** (LB) is a component that sits in front of a pool of identical backend servers and distributes incoming client requests across them. It does three jobs:

1. **Scale** — spread load across N servers so no single machine is the bottleneck. You handle traffic no single box could.
2. **Availability** — continuously health-check the backends and stop sending traffic to any that are down. One server dying no longer means an outage.
3. **A single stable entry point** — clients connect to one address (the **VIP**), not to individual servers. You can add, remove, patch, or replace backends behind the LB with zero changes on the client side.

```
                        ┌──────────────────────┐
   Client A ───────┐    │   LOAD BALANCER      │      ┌──► Backend 1  (10.0.1.10)  ✓ healthy
   Client B ───────┼───►│   VIP: 203.0.113.9   │──────┼──► Backend 2  (10.0.1.11)  ✓ healthy
   Client C ───────┘    │   (one stable addr)  │      ├──► Backend 3  (10.0.1.12)  ✗ DOWN (removed)
                        │   + health checks    │      └──► Backend 4  (10.0.1.13)  ✓ healthy
                        └──────────────────────┘
                                  │
                          decides WHICH backend
                          gets each connection
```

The clients only ever know `203.0.113.9`. Behind it, the backend pool is fluid — that's the whole point.

---

## Why Does It Matter for a Backend Developer?

Load balancers are not "ops stuff you can ignore." They change how your code behaves in production:

- **Your app almost never sees the real client IP.** Behind an L7 LB, `req.socket.remoteAddress` is the *load balancer's* IP, not the user's. Rate limiting, geolocation, audit logging, and IP allowlists all silently break unless you read `X-Forwarded-For`. (This is the #1 surprise — see Example 2.)
- **"It works on my machine" but 502s in prod.** A bad health check path or a missing connection-drain on deploy causes intermittent 502/504 errors that never happen locally.
- **Sticky sessions vs stateless design.** If your app stores session state in local memory, you *need* the LB to pin each user to one backend — which cripples load distribution. Understanding this pushes you toward stateless design (JWT / shared session store).
- **Deploys and zero-downtime.** Rolling deploys, blue-green, and canary releases are all orchestrated *at the load balancer*. You need to understand draining to deploy without dropping requests.
- **Cost and latency trade-offs.** L7 is more expensive and adds a hop of latency; L4 is cheaper and faster but can't do routing/TLS. Picking wrong wastes money or blocks a feature you need later.

If you've ever seen "why is every request logged as coming from `10.0.0.5`?" — that's this topic.

---

## The Packet/Protocol Anatomy

The single most important thing to understand: **an L7 load balancer creates TWO separate TCP connections.**

```
        CONNECTION A                             CONNECTION B
  (client  ◄──►  load balancer)          (load balancer  ◄──►  backend)

┌──────────┐                      ┌──────────────┐                    ┌───────────┐
│  Client  │  TCP conn #1         │ L7 Load Bal. │  TCP conn #2       │  Backend  │
│ 198.51.  │◄────────────────────►│  VIP:        │◄──────────────────►│ 10.0.1.11 │
│ 100.7    │  src=198.51.100.7    │  203.0.113.9 │  src=203.0.113.9   │           │
│          │  dst=203.0.113.9     │              │  dst=10.0.1.11     │           │
└──────────┘                      └──────────────┘                    └───────────┘
       │                                 │                                   │
       │  full TCP handshake here        │  SEPARATE TCP handshake here      │
       │  TLS terminated here            │  (may be plain HTTP or re-encrypt)│
       └─────────────────────────────────┴───────────────────────────────────┘

  Result: the backend's TCP socket shows src IP = 203.0.113.9 (the LB), NOT the client.
          The real client IP survives ONLY inside an HTTP header the LB adds:
              X-Forwarded-For: 198.51.100.7
```

Compare that to an **L4 load balancer**, which has two sub-modes:

```
L4 mode 1 — NAT / proxy (most common):
  The LB rewrites the destination IP (and often source IP) and forwards packets.
  One connection is "spliced" — the LB does NOT terminate TCP application-layer,
  but it IS in the return path. Backend may still see LB IP as source (SNAT),
  unless the LB preserves the client IP (see PROXY protocol / topic 29).

  Client ──[dst=VIP]──► LB ──[dst=backend, src rewritten]──► Backend
         ◄─────────────    ◄──────────────────────────────

L4 mode 2 — Direct Server Return (DSR):
  Request goes THROUGH the LB, but the response goes STRAIGHT back to the client,
  bypassing the LB entirely. Massively higher throughput (LB only sees inbound).

  Client ──[dst=VIP]──► LB ──[rewrite L2 MAC only]──► Backend
     ▲                                                   │
     └───────────────── response goes direct ────────────┘
   (backend answers with src=VIP so the client accepts it)
```

Key protocol facts:
- **L4 forwards, L7 terminates.** L4 shovels TCP/UDP segments; it never speaks HTTP. L7 fully reads and re-emits the HTTP request.
- **L7 can therefore see and modify HTTP** — route by path, rewrite headers, add `X-Forwarded-For`, terminate TLS, compress, etc. L4 sees an opaque byte stream and cannot.
- **Source IP preservation is the crux.** By default the backend sees the LB's IP. Recovering the real client IP requires `X-Forwarded-For` (L7, HTTP header) or the **PROXY protocol** (L4, a tiny header prepended to the TCP stream) — both covered in `29-reverse-proxies.md`.

---

## How It Works — Step by Step

Let's trace one request through an **L7 load balancer** doing TLS termination and path-based routing.

```
Client:  GET https://api.myapp.com/orders/42   →  VIP 203.0.113.9
```

### Step 1 — DNS resolves to the VIP, not a server
```
api.myapp.com  →  203.0.113.9   (the load balancer's virtual IP)
The client never learns any backend's IP. It only ever talks to the VIP.
```

### Step 2 — TCP handshake with the load balancer
```
Client ── SYN ───────► LB:443
Client ◄── SYN-ACK ─── LB:443
Client ── ACK ───────► LB:443
Connection #1 (client ↔ LB) is now ESTABLISHED.
```

### Step 3 — TLS terminates AT the load balancer
```
Client ── ClientHello ──► LB
LB presents the certificate for api.myapp.com (cert + private key live ON the LB)
Shared secret negotiated; TLS tunnel established between client and LB.
→ The LB can now DECRYPT and read the plaintext HTTP request.
```

### Step 4 — LB reads the HTTP request and makes a routing decision
```
Decrypted request:
    GET /orders/42 HTTP/1.1
    Host: api.myapp.com
    Cookie: session=abc123

LB routing rules:
    /orders/*   → orders-service pool   (backends 10.0.2.x)
    /users/*    → users-service pool    (backends 10.0.3.x)
    /*          → web pool              (backends 10.0.1.x)

Path is /orders/42  →  choose orders-service pool.
```

### Step 5 — LB picks a specific backend using its algorithm
```
orders-service pool: [10.0.2.10 (2 conns), 10.0.2.11 (5 conns), 10.0.2.12 (1 conn)]
Algorithm = least-connections  →  pick 10.0.2.12 (fewest active connections)
```

### Step 6 — LB opens (or reuses) a SEPARATE connection to the backend
```
LB ── SYN ───────► 10.0.2.12:8080     ← Connection #2 (LB ↔ backend), brand new TCP
LB ◄── SYN-ACK ───
LB ── ACK ───────►
```

### Step 7 — LB adds forwarding headers and forwards the request
```
LB sends to backend (often plain HTTP inside the private network):
    GET /orders/42 HTTP/1.1
    Host: api.myapp.com
    X-Forwarded-For: 198.51.100.7        ← the REAL client IP, preserved here
    X-Forwarded-Proto: https             ← "original request was HTTPS"
    X-Real-IP: 198.51.100.7
    Cookie: session=abc123
```

### Step 8 — Backend responds; LB relays it back over connection #1
```
Backend ── HTTP 200 + JSON ──► LB ── (re-encrypt with TLS) ──► Client
```

### Step 9 — Health checks run continuously in the background
```
Every 10s, independently of client traffic:
    LB ── GET /health ──► 10.0.2.10   → 200  ✓ keep in pool
    LB ── GET /health ──► 10.0.2.11   → 200  ✓ keep in pool
    LB ── GET /health ──► 10.0.2.12   → 200  ✓ keep in pool
If a backend returns non-200 (or times out) N times in a row → removed from pool.
```

An **L4 load balancer** skips steps 3, 4, 7 entirely — it never decrypts, never reads HTTP, never adds headers. It just picks a backend (step 5) based on IP:port and splices the TCP stream through.

---

## Exact Syntax Breakdown

### An nginx L7 load balancer — the minimal, annotated config

```nginx
# ── Define the backend pool ("upstream") ────────────────────────────
upstream orders_backend {
    least_conn;                       # ← routing algorithm (default is round-robin)

    server 10.0.2.10:8080 weight=3;   # ← weight=3 → gets 3x the traffic of a weight=1 peer
    server 10.0.2.11:8080;            # ← default weight=1
    server 10.0.2.12:8080 max_fails=3 fail_timeout=30s;
    #                     │           └── after 3 fails, quarantine for 30s (passive health check)
    #                     └── passive failure counting

    server 10.0.2.13:8080 backup;     # ← only used if all primaries are down
    keepalive 32;                     # ← keep up to 32 idle conns to backends (reuse, faster)
}

# ── The listener that clients connect to ────────────────────────────
server {
    listen 443 ssl;                   # ← terminate TLS here (this makes it L7)
    server_name api.myapp.com;

    ssl_certificate     /etc/ssl/api.crt;   # ← cert + key live ON the LB
    ssl_certificate_key /etc/ssl/api.key;

    location /orders/ {
        proxy_pass http://orders_backend;    # ← forward to the pool defined above

        # Preserve the real client info (backend would otherwise see nginx's IP):
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;              # real client IP
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;                   # http or https

        proxy_http_version 1.1;               # ← required to reuse keepalive conns
        proxy_set_header Connection "";        # ← clear "close" so keepalive works
    }
}
```

Line-by-line meaning:
```
upstream <name> { ... }   Declares a named pool of backend servers.
least_conn;               Routing algorithm. Omit → round-robin. Others: ip_hash, random, hash.
server IP:PORT ...        One backend. Options: weight=, max_fails=, fail_timeout=, backup, down.
proxy_pass http://<name>; Forward matched requests to the named upstream pool.
proxy_set_header ...      Inject/overwrite a header sent to the backend (this is where XFF lives).
listen 443 ssl;           TLS termination → nginx decrypts → operates at L7.
```

### The AWS equivalent, conceptually

```
AWS ALB (Application Load Balancer)  = L7. Routes on host/path/header, terminates TLS,
                                        injects X-Forwarded-For automatically.
   Listener (:443, TLS cert from ACM)
     └─ Rule: path /orders/* → Target Group "orders" (health check: GET /health, 200)
     └─ Rule: path /users/*  → Target Group "users"
     └─ default              → Target Group "web"

AWS NLB (Network Load Balancer)      = L4. Routes on IP:port, does NOT read HTTP.
                                        Ultra-low latency, can preserve client IP,
                                        supports PROXY protocol, handles millions of conns.
   Listener (:443, TCP)
     └─ Target Group "web" (health check: TCP connect OR HTTP GET /health)
```

---

## Example 1 — Basic

Run a real L7 load balancer locally with nginx in front of two tiny backends. You'll watch requests alternate between servers.

**Start two backends** (each just echoes which server it is):

```bash
# Terminal 1 — backend "A" on port 8081
while true; do echo -e "HTTP/1.1 200 OK\r\nContent-Length: 12\r\n\r\n>> SERVER A <<" | nc -l 8081; done

# Terminal 2 — backend "B" on port 8082
while true; do echo -e "HTTP/1.1 200 OK\r\nContent-Length: 12\r\n\r\n>> SERVER B <<" | nc -l 8082; done
```

**Minimal nginx config** (`/tmp/lb.conf`):

```nginx
events {}
http {
    upstream pool {
        server 127.0.0.1:8081;   # backend A
        server 127.0.0.1:8082;   # backend B
    }
    server {
        listen 8080;
        location / {
            proxy_pass http://pool;   # round-robin by default
        }
    }
}
```

**Run nginx and hammer the VIP:**

```bash
nginx -c /tmp/lb.conf              # start the load balancer on :8080

for i in 1 2 3 4; do curl -s http://localhost:8080/; echo; done
```

**Output — watch it alternate (round-robin):**

```
>> SERVER A <<
>> SERVER B <<
>> SERVER A <<
>> SERVER B <<
```

You just proved the core idea: **one address (`localhost:8080`), traffic spread across two backends.** Kill backend A (Ctrl-C in terminal 1) and re-run the loop — nginx notices the connection fails and sends everything to B. That's availability in action.

---

## Example 2 — Production Scenario: "Every request is logged as coming from the load balancer"

**The situation.** Your Node.js API used to run on one box. You put an AWS ALB (L7) in front of three instances for scale. Immediately:

- Your **rate limiter** stops working — it's now throttling *all* users as if they were one, because every request appears to come from the same IP.
- Your **audit log** shows `10.0.0.42` for every user.
- Your **geolocation** feature thinks everyone is in `us-east-1`.

**Step 1 — Confirm what the app sees.** Add a debug endpoint and hit it through the LB:

```bash
curl -s https://api.myapp.com/debug/ip
```

```json
{
  "socket_remote_address": "10.0.0.42",     ← the LOAD BALANCER's private IP, not the user
  "x_forwarded_for": "198.51.100.7, 10.0.0.42",
  "x_real_ip": null
}
```

There it is. `req.socket.remoteAddress` is the LB. The real client IP (`198.51.100.7`) is sitting in **`X-Forwarded-For`**, which the ALB injected automatically.

**Why:** as shown in the Packet Anatomy section, the L7 LB terminates the client's TCP connection and opens a *new* one to your backend. From your app's socket's point of view, the peer genuinely is the LB.

**Step 2 — The fix (Express example).** Trust the proxy so the framework reads `X-Forwarded-For` for you:

```javascript
// Tell Express there is ONE trusted proxy (the ALB) in front of it.
// Now req.ip returns the client IP from X-Forwarded-For instead of the socket IP.
app.set('trust proxy', 1);

app.get('/debug/ip', (req, res) => {
  res.json({ client_ip: req.ip });   // → "198.51.100.7"  ✓ correct now
});
```

**Verify:**

```bash
curl -s https://api.myapp.com/debug/ip
# { "client_ip": "198.51.100.7" }   ✓ rate limiter, geo, and logs all correct again
```

**Critical security note.** Only trust `X-Forwarded-For` when you *know* a trusted LB set it. If your backend is reachable directly (not only via the LB), an attacker can forge `X-Forwarded-For: 1.2.3.4` to bypass IP rate limits. Trust *only* the specific hop(s) you control — that's what `trust proxy: 1` (trust exactly one hop) means. Full detail in `29-reverse-proxies.md`.

**Bonus failure from the same migration — a deploy causes a burst of 502s.** During a rolling deploy, instances were terminated the instant new ones came up, but the ALB was still routing in-flight requests to the dying instances. Fix: enable **connection draining** (AWS calls it *deregistration delay*, default 300s). On deploy, the target is marked `draining` — the LB stops sending it *new* connections but lets existing ones finish before the instance goes away:

```
Deploy without draining:              Deploy WITH draining:
  instance killed at T=0                instance marked "draining" at T=0
  12 in-flight requests → 502           LB stops NEW conns to it
                                        12 in-flight requests complete (up to 300s)
                                        instance removed cleanly → 0 errors
```

```bash
# AWS: set the deregistration delay on the target group
aws elbv2 modify-target-group-attributes \
  --target-group-arn arn:aws:...:targetgroup/orders/abc123 \
  --attributes Key=deregistration_delay.timeout_seconds,Value=30
```

---

## Common Mistakes

### Mistake 1: Needing L7 features but deploying an L4 balancer (or paying for L7 when L4 suffices)
```
Symptom: "I put an AWS NLB in front and now I can't route /api to one pool
          and /images to another, and I can't terminate TLS at the LB."
```
**Root cause:** NLB is L4 — it only sees IP:port and cannot read the URL path, `Host` header, or cookies. Path/host routing, header rewriting, and TLS termination are **L7-only** features.
**Fix:** Use an L7 LB (ALB / nginx / HAProxy / Envoy) when you need to route on HTTP content. Conversely, if you're just forwarding raw TCP (a database proxy, a game server, gRPC without routing) and want maximum throughput + client-IP preservation, L4 (NLB) is cheaper and faster — don't pay the L7 tax for nothing.

---

### Mistake 2: Forgetting the backend sees the LB's IP, not the client's
```javascript
// BROKEN behind any L7 load balancer:
const clientIp = req.socket.remoteAddress;   // → "10.0.0.42" (the LB) for EVERYONE
rateLimiter.check(clientIp);                  // throttles all users as one
```
**Root cause:** The L7 LB terminates the client connection and opens a new one to you; your socket's peer is the LB.
**Fix:** Read the real IP from `X-Forwarded-For` (via `trust proxy` / `X-Real-IP`), and **only** trust it from hops you control.

---

### Mistake 3: No health check, wrong health-check path, or a health check that flaps
```
Health check hits GET /  → returns 200 even when the DB is down
   → LB keeps routing traffic to a backend that 500s every real request.

OR: health check hits GET /health → but /health requires auth → returns 401
   → LB thinks EVERY backend is unhealthy → removes them all → total outage.

OR: interval=1s, unhealthy_threshold=1, timeout=2s under GC pause
   → one slow response → instantly ejected → then re-added → "flapping"
```
**Root cause:** The health check doesn't actually reflect the app's ability to serve, or its thresholds are too twitchy.
**Fix:** Expose a dedicated unauthenticated `/health` (or `/healthz`) that checks real dependencies (DB, cache) and returns 200 only when the instance can truly serve. Use sane thresholds (e.g. interval 10s, healthy 2, unhealthy 3, timeout 5s) so a single GC pause doesn't eject a good node.

---

### Mistake 4: Sticky sessions silently defeating even load distribution
```
Sticky sessions ON + a few long-lived clients (WebSockets, big customers):

   Backend 1: ████████████████████  80% load  (pinned whales)
   Backend 2: ██                     8%
   Backend 3: ██                     8%
```
**Root cause:** Session affinity pins each client to one backend for its whole session. A handful of heavy or long-lived clients concentrate on one node, and least-connections/round-robin can no longer rebalance them.
**Fix:** Prefer **stateless** backends (store session in Redis/JWT — see `19-cookies-and-sessions.md`) so *any* backend can serve *any* request and stickiness isn't needed at all. If you must be sticky, keep sessions short and combine with least-connections.

---

### Mistake 5: Not draining connections on deploy → burst of 502/504s
```
Rolling deploy kills an instance with 15 in-flight requests still running.
LB was still sending those; they die mid-response → clients get 502.
```
**Root cause:** The instance was removed *before* its existing connections finished.
**Fix:** Enable connection draining / deregistration delay. On deploy, mark the target `draining` first: LB stops new connections but lets in-flight ones complete, *then* the instance shuts down.

---

### Mistake 6: Assuming the LB preserves the client IP by default
```
"I'll just read the source IP on the backend — it'll be the user's."
Reality: by default it's the LB's IP. Preservation is OPT-IN:
   L7 → X-Forwarded-For header (app must read it, and trust it correctly)
   L4 → PROXY protocol, or NLB client-IP preservation (must be enabled + parsed)
```
**Fix:** Decide explicitly how you'll recover the client IP, enable it on the LB, and parse it in the app. Never assume it's there.

---

## Hands-On Proof

```bash
# 1. Stand up the Example-1 nginx LB, then PROVE round-robin distributes:
for i in $(seq 1 10); do curl -s http://localhost:8080/; echo; done
#   → alternating "SERVER A" / "SERVER B" lines. That's distribution, live.

# 2. Prove availability: kill backend A, re-run — 100% now goes to B, no errors.
#    Restart A — traffic returns to both. That's the LB removing/re-adding a node.

# 3. See what the backend actually receives (run a header echo backend):
#    nc -l 8081 and read the raw request — look for the LB's added headers:
#      X-Forwarded-For, X-Real-IP, X-Forwarded-Proto, Host

# 4. Inspect a REAL production LB's fingerprints on any big site:
curl -sI https://www.cloudflare.com | grep -iE 'server|cf-ray|via'
#   Server: cloudflare      ← you're talking to an L7 LB/proxy, not the origin
#   cf-ray: ...             ← the edge node that handled you

# 5. Watch the TWO connections of an L7 proxy with ss/netstat (topic 24):
#    On the LB host during a request you'll see:
ss -tnp | grep -E ':8080|:8081|:8082'
#    → one client↔LB conn AND one LB↔backend conn, simultaneously.

# 6. See the LB's TLS cert vs the origin's (L7 LB terminates TLS itself):
curl -v https://api.myapp.com/ 2>&1 | grep -E 'subject:|issuer:'
#    The cert is served BY the load balancer, not your app server.
```

---

## Practice Exercises

### Exercise 1 — Easy: Prove distribution and failover
```bash
# Using the Example-1 setup (nginx + two nc backends):
# 1. Send 10 requests. Record how many hit A vs B. Is it exactly 50/50? Why/why not?
# 2. Kill backend A. Send 10 more. Where do they go? Any errors on the first request?
# 3. Add `weight=3;` to backend A's line, reload nginx (nginx -s reload).
#    Send 8 requests. What's the new A:B ratio? Explain it.

# Questions:
# - What did the client have to change when a backend died? (Answer: nothing.)
# - Which routing algorithm is nginx using here by default?
```

### Exercise 2 — Medium: Recover the real client IP
```bash
# 1. Replace the nc backends with a tiny server that prints request headers, e.g.:
python3 -c "
import http.server
class H(http.server.BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200); self.end_headers()
        self.wfile.write(('remote=%s XFF=%s\n' % (
            self.client_address[0],
            self.headers.get('X-Forwarded-For'))).encode())
http.server.HTTPServer(('127.0.0.1', 8081), H).serve_forever()
"
# 2. Point nginx at it. Add proxy_set_header X-Forwarded-For / X-Real-IP lines.
# 3. curl the LB. 
# Questions:
# - What IP does the backend's client_address show? (It's nginx's — 127.0.0.1.)
# - What's in X-Forwarded-For, and where did it come from?
# - Remove the proxy_set_header lines and repeat. What's lost?
```

### Exercise 3 — Hard (Production Simulation): Design the routing for a real system
```
You run an e-commerce platform. Design the load-balancing layer. Decide for EACH:
L4 or L7, which routing algorithm, sticky or stateless, and health-check strategy.

  a) Public REST API (stateless, JWT auth), /api/v1/*, needs TLS termination
     and path routing to 4 microservice pools. Traffic is spiky.

  b) A WebSocket service for live order tracking — connections last 20+ minutes.

  c) An internal Postgres read-replica pool that services connect to over raw TCP;
     you need the DB to see the real client IP for its pg_hba rules.

  d) A legacy monolith that stores session state in local memory (can't change it
     this quarter).

# For each, justify:
# - L4 vs L7 and why
# - routing algorithm (round-robin? least-conn? ip-hash? consistent hash?)
# - sticky sessions yes/no
# - what a correct health check looks like
# - how you'd preserve the client IP if needed
#
# (Reference answers: (a) L7, least-conn, stateless, GET /health; TLS at LB.
#  (b) L7 or L4; least-conn strongly (long-lived conns concentrate);
#  (c) L4, least-conn, PROXY protocol / client-IP preservation, TCP health check.
#  (d) L7 with cookie-based stickiness as a stopgap — then fix the state, see topic 19.)
```

---

## Mental Model Checkpoint

Answer from memory. If you can't, re-read the relevant section.

1. **In one sentence each, what's the fundamental difference between an L4 and an L7 load balancer, and what can L7 do that L4 fundamentally cannot?**

2. **Behind an L7 load balancer, how many TCP connections exist for a single client request, and what source IP does the backend's socket report?**

3. **You put an ALB in front of your API and your per-IP rate limiter starts throttling everyone at once. What happened, and what's the exact fix?**

4. **Name four routing algorithms. Which one gives cache affinity (same key → same backend), and why does that matter for a cache tier?**

5. **What's the difference between active and passive health checks? Give one concrete failure mode of a badly configured health check.**

6. **A rolling deploy produces a burst of 502s. What load-balancer feature was missing, and what does it actually do?**

7. **Why are sticky sessions considered a code smell, and what design choice removes the need for them entirely?**

---

## Quick Reference Card

### L4 vs L7 — the decision table

| Dimension | **L4 (Transport)** | **L7 (Application)** |
|---|---|---|
| Operates on | IP + port | Full HTTP (host, path, headers, cookies) |
| Reads payload? | No (opaque byte stream) | Yes (terminates & parses) |
| TLS termination | No (passes through) | Yes (cert lives on the LB) |
| Routing granularity | Per connection, by IP:port | Per request, by URL/host/header/cookie |
| Performance | Extremely fast, low latency | Slower, +1 hop, more CPU |
| Client-IP preservation | Possible (DSR / PROXY protocol) | Needs `X-Forwarded-For` header |
| Header manipulation | No | Yes |
| Content-based routing | No | Yes (path/host → different pools) |
| Cost | Cheaper | More expensive |
| Examples | AWS NLB, IPVS/LVS, HW LBs | AWS ALB, nginx, HAProxy, Envoy, Traefik |

### Routing algorithms

| Algorithm | Picks the backend by | Best for |
|---|---|---|
| Round robin | Next in rotation | Uniform backends, short requests |
| Weighted round robin | Rotation × weight | Mixed backend sizes |
| Least connections | Fewest active conns | Long-lived / uneven request cost |
| Least response time | Lowest latency + conns | Latency-sensitive, heterogeneous nodes |
| IP hash | hash(client IP) → backend | Cheap stickiness without cookies |
| Consistent hashing | hash(key) on a ring | Cache affinity; minimal reshuffle on scale |
| Random (+ two choices) | Random / best of 2 random | Simple, scales well, avoids herd |

### Health checks & stickiness cheat block
```
ACTIVE health check   : LB probes GET /health every N sec. Non-200/timeout ×threshold → eject.
PASSIVE health check  : LB watches real traffic; too many errors → eject (nginx max_fails).
DRAINING              : stop NEW conns to a target, let existing finish, THEN remove.
                        (AWS: "deregistration delay", default 300s.)
STICKY (affinity)     : pin a client to one backend.
   cookie-based       : LB sets its own cookie (AWSALB) OR uses your app cookie.
   source-IP hash     : same client IP → same backend (breaks behind NAT/CGNAT).
STATELESS > STICKY    : store session in Redis/JWT → any backend serves any request.

CLIENT IP RECOVERY:
   L7 →  X-Forwarded-For / X-Real-IP   (trust ONLY hops you control)
   L4 →  PROXY protocol header OR NLB client-IP preservation
```

### nginx quick commands
```bash
nginx -c /path/lb.conf     # start with a specific config
nginx -t                   # test config syntax before reload
nginx -s reload            # hot-reload config, zero dropped connections
```

---

## When Would I Use This at Work?

### Scenario 1: "We're getting 502s during every deploy"
You immediately suspect **connection draining** isn't configured. Instances are being killed with in-flight requests still routed to them. You enable a deregistration delay (or `preStop` + graceful shutdown), so the LB stops new connections to the dying instance and lets existing ones finish. 502s vanish.

### Scenario 2: "One server is pinned at 100% CPU while the others idle"
Classic **sticky sessions + long-lived connections + uneven clients** (Common Mistake 4). You check the affinity config, switch the algorithm to **least-connections**, and — the real fix — start moving session state out of local memory so you can drop stickiness entirely (`19-cookies-and-sessions.md`).

### Scenario 3: "Rate limiting / geo / audit logs all show one IP"
You know instantly: the backend is seeing the **LB's IP**. You set `trust proxy` and read `X-Forwarded-For`, being careful to trust only the hop you control (`29-reverse-proxies.md`).

### Scenario 4: "Do we need an ALB or an NLB for this new service?"
You reach for the L4-vs-L7 table. Need path/host routing, TLS termination, or header rules? → **L7 (ALB)**. Just forwarding raw TCP at massive scale with client-IP preservation (a DB proxy, gRPC pass-through, a game server)? → **L4 (NLB)**, cheaper and faster.

### Scenario 5: "We're adding a caching tier and want request affinity"
You pick **consistent hashing** on the cache key so the same key lands on the same cache node — and, crucially, so adding a node only reshuffles ~1/N of keys instead of every key. This is why consistent hashing (not plain IP hash) is the standard for cache and shard routing.

---

## Connected Topics

**Study before this:**
- `08-tcp-in-depth.md` — the handshake and connection model; an L7 LB's "two connections" only makes sense once you own this.
- `10-http1.1-in-depth.md` — L7 routing reads the request line, `Host`, and headers you learned here.

**Study next (in order):**
- `29-reverse-proxies.md` — the mechanics of `X-Forwarded-For`, `X-Real-IP`, and the PROXY protocol; why your app sees `127.0.0.1`. A load balancer *is* a specialized reverse proxy.
- `34-api-gateways.md` — an L7 LB plus auth, rate limiting, and request transformation, all in one hop.

**Also connects to:**
- `19-cookies-and-sessions.md` — why stateless design lets you drop sticky sessions.
- `30-cdns.md` — a CDN is a globally distributed L7 load-balancing/caching layer.
- `32-firewalls-and-security-groups.md` — the LB is usually the only thing exposed to the internet; backends sit private behind it.

---

*Doc saved: `/docs/networking/28-load-balancers.md`*
