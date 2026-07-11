# 34 — API Gateways

> **Phase 5 — Topic 7 of 7**
> Prerequisites: `01-how-the-internet-works.md`, `18-rate-limiting-at-http-level.md`, `28-load-balancers.md`

---

## ELI5 — The Simple Analogy

Imagine a big office building with 50 different departments inside — HR, Payroll, Legal, IT, Shipping. Now imagine visitors could walk straight in through 50 different side doors, and *every* department had to hire its own security guard to check IDs, its own receptionist to log who came in, and its own rulebook for "you can only ask 3 questions per minute."

That's chaos. Every department reinvents security, badly, and no two do it the same way.

So the building installs **one front desk** at the main entrance:

- You show your ID *once* at the front desk (**authentication**).
- The guard checks you're allowed in and how often you're allowed to visit (**rate limiting**).
- The receptionist looks at your request slip — "I need Payroll" — and points you to the right hallway (**routing**).
- The front desk stamps a visitor badge on you that says *"verified: employee #4471"* so the department doesn't have to re-check your ID (**identity injection**).

The departments now trust anyone wearing a badge, because they know the only way to get one is through the front desk. They stop worrying about security and just do their jobs.

An **API gateway** is that front desk for your microservices. One entry point. All the cross-cutting concerns — auth, rate limits, logging, TLS — handled *once*, at the door, instead of 50 times inside.

---

## Where This Lives in the Network Stack

An API gateway is a **Layer 7 (Application) reverse proxy**. It reads HTTP — the method, path, headers, and often the body — to make decisions. It is not a dumb packet forwarder.

```
┌─────────────────────────────────────────────────────────────────┐
│  Layer 7 — Application   (HTTP)   ◄── API GATEWAY LIVES HERE     │  ← reads paths, headers, JWTs
│  Layer 6 — Presentation  (TLS)    ◄── terminates client TLS too │  ← decrypts to inspect L7
│  Layer 5 — Session                                              │
│  Layer 4 — Transport     (TCP — the gateway owns TWO sockets)   │  ← client↔gw and gw↔backend
│  Layer 3 — Network       (IP — backends see the GATEWAY's IP)   │
│  Layer 2 — Data Link                                            │
│  Layer 1 — Physical                                             │
└─────────────────────────────────────────────────────────────────┘
```

Because it must read the HTTP request to route on path (`/orders` vs `/users`) and to validate a `Bearer` token, it *must* terminate TLS. That means it operates as two connection hops — exactly like the reverse proxy in Topic 29 — but with API-awareness bolted on top.

---

## What Is This?

An **API gateway** is a single, managed entry point that sits in front of many backend services and centralizes the concerns that would otherwise be duplicated in every service.

```
                          ┌──────────────────────────────────────┐
   Clients                │            API GATEWAY               │      Backend services
                          │  (single public entry point)         │      (private network)
┌──────────┐              │                                      │      ┌──────────────┐
│ Mobile   │──────┐       │  1. Terminate TLS                    │  ┌──►│ user-service │
└──────────┘      │       │  2. Authenticate  (validate JWT)     │  │   │  10.0.1.10   │
┌──────────┐      ├──HTTPS│  3. Rate limit    (100 req/min)      │  │   └──────────────┘
│ Browser  │──────┤       │  4. Route by path (/orders → svc)    │──┤   ┌──────────────┐
└──────────┘      │       │  5. Transform     (rewrite headers)  │  ├──►│order-service │
┌──────────┐      │       │  6. Log / trace   (request-id)       │  │   │  10.0.1.11   │
│ Partner  │──────┘       │                                      │  │   └──────────────┘
│  API     │              │       ONE place for cross-cutting    │  │   ┌──────────────┐
└──────────┘              │       concerns, not N places         │  └──►│ pay-service  │
                          └──────────────────────────────────────┘      │  10.0.1.12   │
                                                                         └──────────────┘
```

The clients talk to *one* address. Behind the gateway, you can have 3 services or 300, split them, merge them, rename them, or move them to a different subnet — and no client ever notices. The gateway **decouples the client from your internal topology**.

**Common implementations:**

| Category | Examples |
|----------|----------|
| Managed (cloud) | AWS API Gateway, Google Apigee, Azure API Management |
| Self-hosted / OSS | Kong, Tyk, KrakenD, Envoy-based (Gloo, Contour) |
| Lightweight / DIY | nginx, Traefik, HAProxy configured with auth + rate-limit modules |
| Kubernetes | Ingress controllers (nginx-ingress, Traefik), and the Gateway API |

---

## Why Does It Matter for a Backend Developer?

Without a gateway, **every microservice re-implements the same boring, security-critical plumbing** — and each one gets it slightly wrong:

```
WITHOUT a gateway (every service does everything):

┌────────────────┐   ┌────────────────┐   ┌────────────────┐
│  user-service  │   │  order-service │   │  pay-service   │
│  ┌──────────┐  │   │  ┌──────────┐  │   │  ┌──────────┐  │
│  │ TLS certs│  │   │  │ TLS certs│  │   │  │ TLS certs│  │  ← 3× cert management
│  │ JWT check│  │   │  │ JWT check│  │   │  │ JWT check│  │  ← 3× auth logic (one is buggy)
│  │ rate lim │  │   │  │ rate lim │  │   │  │ rate lim │  │  ← 3× rate limit, no shared count
│  │ CORS     │  │   │  │ CORS     │  │   │  │ CORS     │  │  ← 3× CORS headers (one forgotten)
│  │ logging  │  │   │  │ logging  │  │   │  │ logging  │  │  ← 3× log formats, hard to correlate
│  └──────────┘  │   │  └──────────┘  │   │  └──────────┘  │
└────────────────┘   └────────────────┘   └────────────────┘
      ▲                     ▲                     ▲
      │                     │                     │
   Clients must know all 3 public addresses and each service's quirks.
```

Concretely, as a backend developer a gateway lets you:

- **Ship a new service without touching auth.** Auth is validated at the edge; your service just reads `X-User-Id`.
- **Enforce one rate limit across all services** — a client burning 100 req/min can't dodge the limit by spreading calls across services.
- **Refactor internals freely.** Split `user-service` into `profile-service` + `auth-service` and the public API `/users` stays identical.
- **Debug faster.** One access log, one place to add a `request-id`, one place to see the 429s and 401s.
- **Present a stable public contract** while everything behind it churns.

---

## The Packet/Protocol Anatomy

The single most important network fact about a gateway: **it is two TCP connections, not one.** The client's socket ends *at the gateway*. The gateway opens a brand-new socket to the backend.

```
    CONNECTION HOP 1                          CONNECTION HOP 2
    (public internet, TLS)                    (private network, often plain HTTP)

┌────────┐                        ┌────────────────┐                    ┌──────────────┐
│ Client │== TLS 443, TCP =======►│  API Gateway   │── TCP 8080, HTTP ─►│  Backend svc │
│203.0.  │◄══════════════════════ │  10.0.0.5      │◄───────────────────│  10.0.1.11   │
│113.7   │   src=client_ip        │                │    src=GATEWAY_ip  │              │
└────────┘   dst=gateway_ip       └────────────────┘    dst=backend_ip  └──────────────┘

Backend's kernel sees:  src IP = 10.0.0.5  (the GATEWAY), NOT 203.0.113.7 (the client)
```

Because the backend sees the *gateway's* IP as the source, the real client IP is lost at Layer 3. The gateway preserves it at Layer 7 by injecting a header — the same `X-Forwarded-For` mechanism from Topic 29:

```
What the CLIENT sends over the wire (hop 1):        What the BACKEND receives (hop 2):
┌──────────────────────────────────────┐            ┌──────────────────────────────────────┐
│ GET /orders/42 HTTP/1.1               │            │ GET /42 HTTP/1.1                      │ ← path rewritten
│ Host: api.myapp.com                   │  gateway   │ Host: order-service.internal          │ ← host rewritten
│ Authorization: Bearer eyJhbGci...     │  ───────►  │ X-User-Id: 4471                       │ ← identity INJECTED
│ Origin: https://app.myapp.com         │            │ X-User-Roles: admin                   │ ← from decoded JWT
│                                       │            │ X-Forwarded-For: 203.0.113.7          │ ← real client IP
│                                       │            │ X-Request-Id: 8f3c-...                 │ ← trace id
│                                       │            │ (Authorization header STRIPPED)       │ ← token removed
└──────────────────────────────────────┘            └──────────────────────────────────────┘
```

The gateway **consumed** the `Authorization` token (validated the JWT signature and expiry), **translated** it into a small trusted `X-User-Id` header, and **stripped** the original. The backend never parses a JWT. It just trusts the header — *because the backend is on a private network and the gateway is the only thing that can reach it.* That "because" is load-bearing and we'll return to it in Common Mistakes.

---

## How It Works — Step by Step

Trace a single request `GET https://api.myapp.com/orders/42` from a mobile client through the gateway to `order-service`.

```
CLIENT:  fetch("https://api.myapp.com/orders/42", { headers: { Authorization: "Bearer eyJ..." }})
```

### Step 1 — DNS + TCP + TLS to the gateway
```
1a. DNS: api.myapp.com → 203.0.113.50  (the GATEWAY's public IP — not any service)
1b. TCP 3-way handshake to 203.0.113.50:443
1c. TLS 1.3 handshake — gateway presents the *.myapp.com certificate
    ► TLS is TERMINATED here. The gateway now holds the plaintext HTTP request.
```

### Step 2 — Authentication (offloaded at the edge)
```
2a. Gateway reads:  Authorization: Bearer eyJhbGci...
2b. Validates the JWT: signature (against a known public key / JWKS), exp, iss, aud
2c. FAIL  → gateway returns 401 immediately. Backend is never contacted. ◄── cheap rejection
2d. PASS  → gateway extracts claims: sub=4471, roles=["admin"]
```
*Auth-over-HTTP details (Bearer, JWT structure) live in Topic 16.*

### Step 3 — Rate limiting & quotas
```
3a. Identify the caller (by X-User-Id, API key, or client IP)
3b. Check the counter in a shared store (Redis): "user 4471 → 87 requests this minute"
3c. OVER LIMIT → return 429 Too Many Requests + Retry-After: 12   ◄── backend protected
3d. UNDER LIMIT → increment counter, continue
```
*The 429 / Retry-After / X-RateLimit-* contract lives in Topic 18.*

### Step 4 — Routing (path/host-based dispatch)
```
4a. Match the request path against the route table:
        /users/**   → user-service   (10.0.1.10:8080)
        /orders/**  → order-service  (10.0.1.11:8080)   ◄── matches
        /pay/**     → pay-service    (10.0.1.12:8080)
4b. Selected upstream: order-service. (This may itself be a pool → load balanced, Topic 28.)
```

### Step 5 — Request transformation
```
5a. Rewrite path:    /orders/42  →  /42         (strip the routing prefix)
5b. Rewrite Host:    api.myapp.com → order-service.internal
5c. Inject identity: X-User-Id: 4471, X-User-Roles: admin
5d. Inject tracing:  X-Request-Id: 8f3c...,  X-Forwarded-For: 203.0.113.7
5e. Strip secrets:   remove Authorization header (backend doesn't need the raw token)
```

### Step 6 — Open the second connection & forward
```
6a. Gateway opens a NEW TCP connection to 10.0.1.11:8080 (private network, plain HTTP)
    (Usually reused from a keep-alive connection pool — no fresh handshake each time.)
6b. Forwards the transformed request.
6c. order-service trusts X-User-Id: 4471 and does its work. NO JWT parsing.
```

### Step 7 — Response transformation & return
```
7a. Backend responds:  200 OK { "id": 42, "total": 19.99 }
7b. Gateway may add:   X-RateLimit-Remaining: 12, Cache-Control, CORS headers
7c. Gateway may strip: internal headers (X-Internal-Debug, Server version leak)
7d. Response encrypted back over TLS on hop 1 → client
7e. Gateway logs one line: method, path, user, upstream, status, latency
```

### The complete picture
```
┌────────┐   HTTPS    ┌───────────────────────────────────────┐   HTTP    ┌──────────────┐
│ Mobile │──443──────►│ GATEWAY 203.0.113.50                  │──8080────►│order-service │
│ client │            │ ─────────────────────────────────────  │           │ 10.0.1.11    │
│        │            │ TLS term → JWT check → rate limit →     │           │ (no auth,    │
│        │◄───────────│ route → transform → forward → log       │◄──────────│  no TLS,     │
└────────┘  200 + TLS └───────────────────────────────────────┘   200     │  trusts hdr) │
                       ▲                                                    └──────────────┘
              401 (bad token) / 429 (over limit) returned HERE — backend never sees the request
```

---

## Exact Syntax Breakdown

There is no single "API gateway protocol" — each implementation has its own config. Below is **Kong** (a widely used OSS gateway), because its model — *services*, *routes*, and *plugins* — maps cleanly onto the concepts. A declarative Kong config:

```yaml
# kong.yml — declarative config
_format_version: "3.0"

services:                              # ── an upstream backend the gateway forwards to
  - name: order-service
    url: http://10.0.1.11:8080         # where the real service lives (private IP)
    routes:                            # ── which incoming requests map to this service
      - name: orders-route
        paths:
          - /orders                    # match any request whose path starts with /orders
        strip_path: true               # /orders/42  →  forwarded as  /42
    plugins:                           # ── cross-cutting concerns, applied in order
      - name: jwt                      # (1) validate the Bearer JWT; reject with 401 if bad
      - name: rate-limiting            # (2) throttle callers
        config:
          minute: 100                  #     100 requests per minute...
          policy: redis                #     ...counted in a SHARED redis store
          redis_host: 10.0.2.5         #     (so the limit holds across all gateway nodes)
      - name: request-transformer      # (3) reshape the request before forwarding
        config:
          add:
            headers:
              - "X-Gateway: kong"      #     inject a header the backend can trust
          remove:
            headers:
              - "Authorization"        #     strip the raw token; backend doesn't need it
```

Annotated, piece by piece:

```
services:        WHERE traffic goes    → the backend URL (private address).
  routes:        WHICH traffic matches → paths / hosts / methods that select this service.
    strip_path:  path REWRITE          → remove the matched prefix before forwarding.
  plugins:       WHAT happens en route → ordered middleware: auth, rate-limit, transform, log.
```

The same three ideas in **AWS API Gateway** vocabulary:

```
Resource   (/orders/{id})        ≈  Kong route path
Method     (GET, with Authorizer)≈  Kong jwt plugin + method match
Integration(HTTP_PROXY → 10.0..) ≈  Kong service url
Usage Plan (100 req/min, API key)≈  Kong rate-limiting plugin
```

And in **Kubernetes Ingress** (the gateway-lite that ships with every cluster):

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/limit-rpm: "100"        # rate limit
    nginx.ingress.kubernetes.io/auth-url: "http://auth.default.svc/verify"  # auth offload
spec:
  rules:
    - host: api.myapp.com
      http:
        paths:
          - path: /orders                                # route by path
            backend:
              service:
                name: order-service                      # → forward here
                port: { number: 8080 }
```

---

## Example 1 — Basic

Run a real gateway locally with nginx acting as a minimal API gateway in front of two "services." No cloud account needed.

**Two toy backends** (start them in two terminals):
```bash
# "user-service" on 9001 — echoes the headers it receives
python3 -c "
import http.server, json
class H(http.server.BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200); self.send_header('Content-Type','application/json'); self.end_headers()
        self.wfile.write(json.dumps({'service':'user','path':self.path,'got_user':self.headers.get('X-User-Id')}).encode())
http.server.HTTPServer(('127.0.0.1',9001),H).serve_forever()"

# "order-service" on 9002 — same, change 'user'→'order' and 9001→9002
```

**Gateway config** (`gateway.conf`):
```nginx
events {}
http {
  server {
    listen 8080;

    # ── ROUTE by path prefix ──
    location /users/ {
      proxy_pass http://127.0.0.1:9001/;   # trailing / strips the /users prefix
      # ── TRANSFORM: inject a trusted identity header (auth offload, simplified) ──
      proxy_set_header X-User-Id 4471;
      proxy_set_header X-Forwarded-For $remote_addr;
    }
    location /orders/ {
      proxy_pass http://127.0.0.1:9002/;
      proxy_set_header X-User-Id 4471;
    }
  }
}
```

**Run and test:**
```bash
nginx -c "$PWD/gateway.conf" -g 'daemon off;' &   # start the gateway on :8080

curl -s http://localhost:8080/users/profile | python3 -m json.tool
```
```json
{
    "service": "user",
    "path": "/profile",          ◄── /users/profile was rewritten to /profile before forwarding
    "got_user": "4471"           ◄── the gateway INJECTED X-User-Id; the backend just read it
}
```

```bash
curl -s http://localhost:8080/orders/42 | python3 -m json.tool
```
```json
{ "service": "order", "path": "/42", "got_user": "4471" }
```

One address (`localhost:8080`), two services, path-based routing, and header injection — the core of a gateway in 20 lines.

---

## Example 2 — Production Scenario: "A client is bypassing our gateway"

**The situation.** You run 8 microservices behind an AWS API Gateway that enforces JWT auth and a 1,000 req/min quota per API key. Everything works — until Security notices that `payment-service` is receiving *unauthenticated* requests with no rate limiting, from an IP that isn't the gateway. Money is moving without auth. Alarms.

**Step 1 — Confirm the gateway is doing its job.**
```bash
# Through the gateway: no token → 401. Good.
curl -s -o /dev/null -w "%{http_code}\n" https://api.myapp.com/pay/charge
# 401
```

**Step 2 — Find the leak. Does the service have a public IP of its own?**
```bash
# Someone shared the raw service hostname. Try hitting it directly.
curl -s -o /dev/null -w "%{http_code}\n" http://payment-service.myapp.com:8080/charge
# 200   ◄── DISASTER. The service answered directly, no token, no rate limit.
```

The service was launched on a **public subnet** with a security group allowing `0.0.0.0/0` on port 8080. The gateway was one door; this was an unlocked window. Because auth was *offloaded* to the gateway, the service itself does zero auth — so anyone who reaches it directly is trusted.

**Step 3 — Prove the client bypass in the access logs.**
```bash
# Gateway sees the client IP. Direct hits show the real caller, not the gateway IP.
aws logs filter-log-events --log-group-name /app/payment-service \
  --filter-pattern '"POST /charge"' --max-items 5 --query 'events[].message' --output text
```
```
203.0.113.99 POST /charge 200  X-Forwarded-For=-      ◄── no XFF header = NOT via gateway
203.0.113.99 POST /charge 200  X-Forwarded-For=-      ◄── direct, unauthenticated
10.0.0.5     POST /charge 200  X-Forwarded-For=198..  ◄── this one IS from the gateway (10.0.0.5)
```
The tell: legitimate traffic comes from the gateway's internal IP (`10.0.0.5`) and carries `X-Forwarded-For`. The bypass traffic comes straight from `203.0.113.99` with no forwarded header.

**Step 4 — The fix: make the backend unreachable except through the gateway.**

The gateway model *only* works if the backends are private. Two layers:

```bash
# (a) Move the service off the public subnet — no public IP at all (Topics 31/32).
#     Now payment-service.myapp.com has no A record to a routable address.

# (b) Lock the security group: allow port 8080 ONLY from the gateway's security group,
#     not from 0.0.0.0/0.
aws ec2 authorize-security-group-ingress \
  --group-id sg-payment \
  --protocol tcp --port 8080 \
  --source-group sg-apigateway        # ◄── only the gateway SG may connect
aws ec2 revoke-security-group-ingress \
  --group-id sg-payment \
  --protocol tcp --port 8080 --cidr 0.0.0.0/0   # ◄── remove the open-to-world rule
```

**Step 5 — Verify the hole is closed.**
```bash
curl -s -o /dev/null -w "%{http_code}\n" --max-time 5 http://payment-service.myapp.com:8080/charge
# 000  (connection timed out — the port no longer accepts anything but the gateway)

curl -s -o /dev/null -w "%{http_code}\n" https://api.myapp.com/pay/charge \
  -H "Authorization: Bearer $VALID_TOKEN" -X POST
# 200  ◄── the ONLY path that works now runs through auth + rate limiting
```

**The lesson.** Auth offloading is a bargain: services skip auth *in exchange for* being unreachable except through the gateway. If you take the first half (skip auth) without the second (make private), you've built an open door with a sign that says "please use the front desk."

---

## Common Mistakes

### Mistake 1: Leaving backend services publicly reachable
```
Belief:  "The gateway does auth, so we're protected."
Reality: If the backend has a public IP / open security group, attackers skip the
         gateway entirely and hit the service directly — with no auth, no rate limit.
```
**Fix:** Backends live on a private subnet (Topic 31). Their security group allows inbound *only* from the gateway's security group (Topic 32), never `0.0.0.0/0`. The gateway must be the *only* reachable path. Verify with a direct `curl` from outside — it must time out.

---

### Mistake 2: Doing heavy synchronous work inside the gateway
```
Belief:  "The gateway is where requests pass through — put the logic there."
Reality: The gateway is a shared chokepoint. Every request goes through it. A slow
         plugin (a synchronous DB call, image resizing, a blocking external HTTP call)
         adds latency to EVERY route and turns the gateway into a bottleneck + SPOF.
```
**Fix:** Keep the gateway thin: auth check, rate-limit lookup, route, forward. Cheap, fast, stateless (state lives in Redis). Put business logic in the services. If a plugin must call out, make it async and cache aggressively. Run multiple gateway replicas behind a load balancer so it isn't a single point of failure.

---

### Mistake 3: Trusting injected identity headers without locking down direct access
```
Belief:  "The backend reads X-User-Id, so it knows who the user is."
Reality: X-User-Id is just a header. Anyone who can reach the backend can SET it.
         curl http://svc:8080/admin -H "X-User-Id: 1"  ← instant impersonation
         if the backend is directly reachable.
```
**Fix:** The injected-identity pattern is *only* safe when the backend cannot be reached directly (Mistake 1) — private network + gateway-only security group. Additionally, have the gateway **strip** any client-supplied `X-User-Id` before injecting its own, so a client can't smuggle a forged one through. For higher assurance, use mTLS between gateway and services (Topic 36) so the backend cryptographically verifies the caller is the gateway.

---

### Mistake 4: No rate limiting — one client starves everyone
```
Belief:  "We'll add rate limits later; traffic is low."
Reality: One buggy client in a retry loop (or one abusive partner) sends 10k req/s.
         Your services and shared database saturate. Every OTHER client gets errors.
         This is a self-inflicted DoS.
```
**Fix:** Set a default quota at the gateway from day one (Topic 18). Per-API-key and per-user limits so one caller's burst can't consume the whole pool. Return `429 Too Many Requests` + `Retry-After`. The limit must be counted in a *shared* store (Redis) so it holds across all gateway replicas — a per-instance in-memory counter multiplies the real limit by the number of gateway nodes.

---

### Mistake 5: Conflating an API gateway with a plain load balancer
```
Belief:  "We have an ALB, so we have a gateway."
Reality: A plain L4/L7 load balancer distributes traffic across identical backends.
         It does NOT validate JWTs, enforce per-user quotas, transform requests,
         translate REST↔gRPC, or manage API keys. Those are gateway features.
```
**Fix:** Know what you actually need. If you only need to spread load across identical replicas, a load balancer (Topic 28) is enough. The moment you need auth offloading, per-consumer quotas, request transformation, or a stable public API over churning internals, you need a gateway. They overlap — a gateway *is* a specialized L7 proxy/LB — but the API-aware features are the difference.

---

### Mistake 6: Duplicating full auth in both the gateway and every service
```
Belief:  "Defense in depth — validate the JWT at the gateway AND in every service."
Reality: If done as full re-validation everywhere, you pay JWKS fetches + signature
         checks N times, and you've un-done the whole point of offloading. It's slow
         and every service re-implements the auth you centralized.
```
**Fix:** Validate the token *once* at the gateway; services trust the injected identity over a locked-down private network. "Defense in depth" here means network isolation + mTLS, not re-parsing the JWT in every service. (A lightweight sanity check in the service — "is `X-User-Id` present?" — is fine; a full signature re-verification on every hop is the anti-pattern.)

---

## Hands-On Proof

```bash
# 1. See a real managed gateway in action (httpbin sits behind one).
#    Watch which headers a proxy/gateway ADDS on your behalf:
curl -s https://httpbin.org/headers | python3 -m json.tool
#    Look for X-Forwarded-For, X-Forwarded-Host — injected by the edge proxy,
#    exactly like a gateway injects identity headers.

# 2. Prove the gateway TERMINATES TLS and opens its own connection:
#    The cert you get belongs to the gateway/edge, not your origin service.
curl -v https://httpbin.org/get 2>&1 | grep -Ei "subject:|issuer:|TLSv"

# 3. See a rate-limited endpoint return 429 + Retry-After (github's is real):
for i in $(seq 1 5); do
  curl -s -o /dev/null -w "req $i → %{http_code}\n" https://api.github.com/users/torvalds
done
curl -sD - -o /dev/null https://api.github.com/users/torvalds | grep -i "x-ratelimit"
#    X-RateLimit-Limit / X-RateLimit-Remaining are gateway-enforced quota headers.

# 4. Watch path-based routing decide the backend (httpbin echoes the path it saw):
curl -s https://httpbin.org/anything/orders/42 | python3 -c "import sys,json;print(json.load(sys.stdin)['url'])"

# 5. Prove header injection/stripping with a local nginx gateway (see Example 1),
#    then confirm the backend sees the INJECTED X-User-Id, not your Authorization:
curl -s -H "Authorization: Bearer secret" http://localhost:8080/users/me
#    → backend reports got_user=4471; the raw 'secret' token never reached it.
```

---

## Practice Exercises

### Exercise 1 — Easy: Route two services behind one nginx gateway
```bash
# Using the Example 1 setup, add a THIRD backend "pay-service" on port 9003
# and a /pay/ route. Then answer:
#
# 1. What single command lists all three services from ONE address?
# 2. When you curl http://localhost:8080/pay/charge, what path does pay-service see?
#    (Hint: what does the trailing slash in proxy_pass do?)
# 3. Add proxy_set_header X-Request-Id $request_id; to all three routes.
#    Confirm each backend receives a unique id per request.
```

### Exercise 2 — Medium: Prove the auth-offload trust boundary
```bash
# Extend Example 1 so /orders/ requires a header the client must NOT be able to forge.
#
# 1. Make the gateway STRIP any client-supplied X-User-Id, then inject its own:
#      proxy_set_header X-User-Id 4471;   # already overwrites — verify it does
#    Test: curl -H "X-User-Id: 9999" http://localhost:8080/orders/1
#    Does the backend see 9999 or 4471? Why does the ORDER of processing matter?
#
# 2. Now start order-service so it ALSO listens on a public-ish port and curl it
#    directly with -H "X-User-Id: 1". What stops this in production?
#    Write one sentence describing the network control that closes this hole.
```

### Exercise 3 — Hard: Design the bypass fix from Example 2 end-to-end
```bash
# Scenario: 8 services behind a gateway; one (payment) is directly reachable.
#
# 1. Write the security-group rule (AWS syntax) that allows port 8080 into
#    payment-service ONLY from the gateway's security group, and the rule that
#    REVOKES 0.0.0.0/0. (See Example 2, Step 4.)
#
# 2. Explain how you'd detect ANY future bypass automatically:
#    which single field in the service access log distinguishes gateway traffic
#    from a direct hit? Write the log filter you'd alert on.
#
# 3. Bonus: the injected X-User-Id is still a plaintext header on the private net.
#    Which Topic-36 mechanism would let the backend CRYPTOGRAPHICALLY verify the
#    caller is the gateway, not just trust the source IP? Name it and say in one
#    line what it adds over "private subnet + security group."
```

---

## Mental Model Checkpoint

Answer from memory. If you can't, re-read the relevant section.

1. **How many TCP connections exist when a client's request flows through a gateway to a backend, and what source IP does the backend's kernel see?**

2. **A request arrives with an expired JWT. At which step does the gateway reject it, what status code does it return, and does the backend service ever see the request?**

3. **Why is it safe for a backend to trust an `X-User-Id` header the gateway injects — and what one network condition, if violated, makes it catastrophically unsafe?**

4. **Give three concrete features an API gateway has that a plain L4 load balancer does not.**

5. **Your gateway runs 4 replicas, each enforcing "100 req/min" from its own in-memory counter. What is the *actual* rate limit a single client experiences, and how do you fix it?**

6. **You want to split `user-service` into two services without any client noticing. Which gateway responsibility makes this possible, and what stays constant from the client's view?**

7. **A partner reports they're hitting your `payment-service` with no token and it works. What is almost certainly misconfigured, and what are the two fixes (one network, one gateway/header)?**

---

## Quick Reference Card

| Responsibility | What the gateway does | Ties to |
|----------------|----------------------|---------|
| Routing | Path/host-based dispatch to the right service | Topic 28 (LB) |
| Auth offload | Validate JWT/API key once at the edge; inject `X-User-Id` | Topic 16 |
| Rate limiting | Per-user/key quotas from a shared store; return 429 | Topic 18 |
| Transformation | Rewrite paths/headers/bodies; REST↔gRPC translation | — |
| Aggregation / BFF | Fan out one client call to several services, merge results | Topic 36 |
| TLS termination | Decrypt client TLS so L7 can be inspected | Topic 09 |
| Observability | One access log, `X-Request-Id`, metrics, tracing | Topic 27 |
| API key mgmt | Issue/revoke keys, map key → quota/plan | — |

**Gateway vs Load Balancer vs Reverse Proxy:**

| | Load Balancer | Reverse Proxy | API Gateway |
|---|---|---|---|
| Layer | L4 or L7 | L7 | L7 (API-aware) |
| Core job | Spread load across *identical* backends | Generic front for one/many servers | Managed API entry point |
| Auth / JWT | No | Rarely | **Yes** (offload) |
| Per-user rate limit | No | Basic | **Yes** (quotas) |
| Request transform | No | Some (headers) | **Yes** (paths/body/protocol) |
| API keys / plans | No | No | **Yes** |
| Relationship | — | A gateway *is* a specialized reverse proxy… | …with LB + API features on top |

**Cheat block:**
```
Two connection hops:    client ──TLS──► GATEWAY ──HTTP──► backend
Backend sees:           src IP = gateway (real client in X-Forwarded-For)
Reject early, cheaply:  401 (bad token) and 429 (over quota) never reach the backend
Auth offload deal:      services skip auth  ⇔  services MUST be private (gateway-only)
Injected identity:      X-User-Id is trusted ONLY because backend is unreachable directly
Keep the gateway thin:  auth + rate-limit + route + forward; NO heavy sync work
Rate limit correctly:   shared Redis counter, not per-replica in-memory
Implementations:        AWS API Gateway · Kong · Apigee · Envoy · nginx/Traefik · k8s Ingress
```

---

## When Would I Use This at Work?

### Scenario 1: "We're breaking the monolith into microservices"
The first piece of shared infrastructure you stand up is a gateway. Put it in front of the monolith today (single route: `/** → monolith`). As you carve out services, add routes (`/orders/** → order-service`) without clients ever changing their base URL. The gateway is what lets the migration be invisible.

### Scenario 2: "Every service team keeps reimplementing auth, and one did it wrong"
Move JWT validation to the gateway. Services read `X-User-Id` from a locked-down private network. Auth logic now lives in exactly one place, is tested once, and a new service ships with auth "for free." Pair it with private subnets + security groups so the injected header is trustworthy.

### Scenario 3: "A partner's integration is hammering us and taking everyone down"
Issue that partner an API key at the gateway and attach a usage plan — say 600 req/min. Their bursts now hit a `429` at the edge instead of your database. Other clients are unaffected because the quota is per-key. No code change in any service.

### Scenario 4: "We need to refactor the internal API but external clients depend on the old shape"
Use request/response transformation at the gateway to keep the public contract stable — rewrite the old `/v1/orders/{id}` path and reshape the JSON — while the internal service moves to a new path and schema. The gateway absorbs the impedance mismatch so no client has to migrate on your schedule.

---

## Connected Topics

**Study before this:**
- `16-authentication-over-http.md` — Bearer tokens and JWTs are what the gateway validates and offloads
- `18-rate-limiting-at-http-level.md` — the 429 / Retry-After / quota contract the gateway enforces
- `28-load-balancers.md` — a gateway routes to backend *pools* that are themselves load balanced
- `29-reverse-proxies.md` — a gateway *is* a reverse proxy; `X-Forwarded-For` and two-hop connections come from here

**Study next / alongside:**
- `31-vpns-and-private-networks.md` — backends must be private for auth offload to be safe
- `32-firewalls-and-security-groups.md` — the security-group rule that makes the gateway the *only* way in
- `36-service-to-service-communication.md` — mTLS and service mesh, the next step past "trust the injected header"

**The one-sentence tie-together:** an API gateway is an API-aware L7 reverse proxy that terminates client TLS, authenticates and rate-limits at the edge, and forwards transformed requests to private backends over a second connection — consolidating every cross-cutting concern into one managed front door.

---

*Doc saved: `/docs/networking/34-api-gateways.md`*
