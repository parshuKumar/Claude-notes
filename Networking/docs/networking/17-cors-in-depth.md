# 17 — CORS in Depth

> **Phase 3 — Topic 4 of 7**
> Prerequisites: `01-how-the-internet-works.md`, `10-http1.1-in-depth.md`, `14-http-headers-in-depth.md`

---

## ELI5 — The Simple Analogy

Imagine a school where each classroom is a **website**. A kid (your JavaScript code) is sitting in classroom A (`https://app.example.com`) and wants to borrow a book from classroom B (`https://api.example.com`).

There's a strict **hall monitor** (the **browser**) standing at every door. The rule is simple: *"You may read things inside your own classroom freely. But if you want to take something FROM another classroom back to yours and READ it, that classroom's teacher must have signed a permission slip saying your classroom is allowed."*

Here's the crucial twist most people get wrong:

- The hall monitor doesn't stop the kid from *walking* to classroom B. The kid goes, the request happens, classroom B does the work.
- The hall monitor stops the kid from **carrying the answer back into classroom A** unless classroom B's teacher pinned a permission slip on the reply.
- If there's no slip, the monitor confiscates the answer and tells the kid: *"blocked by CORS policy."* The kid never gets to read it.

And the biggest twist: **the hall monitor only exists inside the school.** A kid who goes home (a `curl` command, a backend server) has no hall monitor at all — they can read anything from anywhere. CORS is a rule the browser enforces to protect **its own user**, not a lock the server puts on its door.

---

## Where This Lives in the Network Stack

CORS is **not a protocol**. There are no CORS packets. It is a set of **HTTP headers** (Layer 7) plus a **browser-enforced policy** that lives entirely in the browser's networking code — above HTTP, below your JavaScript.

```
┌─────────────────────────────────────────────────────────────────┐
│  Layer 7 — Application                                          │
│                                                                 │
│    ┌───────────────────────────────────────────────────────┐   │
│    │  YOUR JAVASCRIPT   fetch("https://api.example.com")   │   │
│    └───────────────────────────────────────────────────────┘   │
│                          │  the browser intercepts this          │
│                          ▼                                       │
│    ┌───────────────────────────────────────────────────────┐   │
│    │  BROWSER CORS ENGINE  ← Same-Origin Policy lives here │   │
│    │   • decides: simple request or preflight?             │   │
│    │   • reads Access-Control-* response headers            │   │
│    │   • ALLOWS or BLOCKS the response reaching your JS     │   │
│    └───────────────────────────────────────────────────────┘   │
│                          │  actual bytes on the wire            │
│                          ▼                                       │
│    ┌───────────────────────────────────────────────────────┐   │
│    │  HTTP/1.1 or HTTP/2   (headers + body — Topic 10)     │   │
│    └───────────────────────────────────────────────────────┘   │
│                                                                 │
│  Layer 6 — TLS (Topic 09)                                       │
│  Layer 4 — TCP (Topic 08)                                       │
│  Layer 3 — IP                                                   │
└─────────────────────────────────────────────────────────────────┘
```

**Key placement fact:** CORS enforcement happens in the box labeled "BROWSER CORS ENGINE." Everything below it — TCP, TLS, the HTTP request itself — happens *normally*. The request is sent, the server responds. CORS only decides whether your JavaScript is *allowed to see* that response.

---

## What Is This?

**CORS** stands for **Cross-Origin Resource Sharing**. It is a controlled relaxation of a much older, stricter rule called the **Same-Origin Policy (SOP)**.

### The Same-Origin Policy

By default, a browser lets JavaScript on page A read responses from server B **only if A and B share the same origin**. An **origin** is the triple:

```
        scheme  ://  host              :  port
        ──────       ──────────────       ────
        https        app.example.com      443
```

**All three must match exactly** for two URLs to be "same-origin":

| URL A | URL B | Same origin? | Why |
|-------|-------|:---:|-----|
| `https://app.example.com/a` | `https://app.example.com/b` | ✅ | path doesn't count |
| `https://app.example.com` | `http://app.example.com` | ❌ | scheme differs (https ≠ http) |
| `https://app.example.com` | `https://api.example.com` | ❌ | host differs (subdomain counts!) |
| `https://app.example.com` | `https://app.example.com:8443` | ❌ | port differs (443 ≠ 8443) |
| `https://example.com` | `https://www.example.com` | ❌ | host differs (`www` is a subdomain) |

**CORS is the mechanism a server uses to say "it's OK, that other origin may read my responses."** It does this by sending `Access-Control-*` HTTP headers. The browser reads those headers and decides whether to hand the response to the JavaScript or block it.

### The single most important sentence in this entire document

> **CORS is enforced BY THE BROWSER, not by the server. The server just sends headers giving permission. Anything that is not a browser — `curl`, `wget`, Postman, your Node.js backend, another server — completely ignores CORS.**

This is why a `curl` request "works" but the same URL "fails with CORS" in the browser. Nothing is wrong with the server. The browser is doing its job; `curl` has no such job.

---

## Why Does It Matter for a Backend Developer?

You will hit CORS constantly, and almost always it lands on **your** desk because **only the server can fix it**:

- The frontend team pings you: *"Your API is broken, we get 'blocked by CORS policy'."* The API is fine. Your **headers** are missing. They physically cannot fix it in frontend code — you must fix the server.
- You add cookie-based auth (Topic 16) and suddenly every request fails, because credentialed CORS has stricter rules that `Access-Control-Allow-Origin: *` violates.
- You add a custom header like `Authorization` or `X-Request-Id` and now the browser fires a **preflight OPTIONS** request your server never handled — so it 404s and everything breaks.
- You need to understand the security boundary: **CORS does NOT protect your API.** A relaxed CORS policy does not "open a hole" for attackers using `curl`; a strict one does not "secure" your API against them. CORS protects *the browsing user* from a malicious site stealing *their* authenticated data. Real API protection is **authentication** (Topic 16), not CORS.
- You must diagnose fast: is this a CORS error, a network error, or an auth (401/403) error? They look similar in the console but have completely different fixes.

Knowing exactly which header to add — and why `*` breaks the moment credentials appear — is a routine, high-frequency backend skill.

---

## The Packet/Protocol Anatomy

There is no special "CORS packet." CORS is expressed as **HTTP headers** riding inside a normal HTTP request/response (Topic 10 / Topic 14). Here is what actually appears on the wire.

### The request side (browser → server) adds:

```
GET /users HTTP/1.1
Host: api.example.com
Origin: https://app.example.com      ← browser AUTO-ADDS this on cross-origin requests
                                        (you cannot set or remove it from JavaScript)
```

### The preflight request (browser → server) adds two more:

```
OPTIONS /users HTTP/1.1
Host: api.example.com
Origin: https://app.example.com
Access-Control-Request-Method: PUT              ← "the real request will use PUT"
Access-Control-Request-Headers: authorization,content-type
                                                ← "...and will send these headers"
```

### The response side (server → browser) adds the `Access-Control-*` family:

```
HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://app.example.com   ← the permission slip
Access-Control-Allow-Methods: GET, POST, PUT, DELETE   ← (preflight response only)
Access-Control-Allow-Headers: authorization, content-type  ← (preflight response only)
Access-Control-Allow-Credentials: true                 ← allow cookies/auth to be read
Access-Control-Expose-Headers: X-Total-Count           ← let JS read these response headers
Access-Control-Max-Age: 600                            ← (preflight) cache this decision 600s
Vary: Origin                                           ← cache correctness (see later)
```

Visually, where these sit inside the HTTP message:

```
┌──────────────────────────────────────────────────────────────┐
│ HTTP RESPONSE                                                 │
│  ┌──────────────────────────────────────────────────────┐    │
│  │ STATUS LINE:  HTTP/1.1 200 OK                        │    │
│  ├──────────────────────────────────────────────────────┤    │
│  │ HEADERS:                                             │    │
│  │   content-type: application/json                     │    │
│  │   access-control-allow-origin: https://app...   ◄── │────┼─ browser reads
│  │   access-control-allow-credentials: true        ◄── │────┼─ THESE and decides
│  │   vary: Origin                                  ◄── │────┼─ allow / block
│  ├──────────────────────────────────────────────────────┤    │
│  │ BODY:  [{"id":1,"name":"Alice"}]                     │    │  ← delivered to JS
│  └──────────────────────────────────────────────────────┘    │     ONLY if allowed
└──────────────────────────────────────────────────────────────┘
```

---

## How It Works — Step by Step

The browser splits every cross-origin request into one of two paths: **simple request** (no preflight) or **preflighted request** (an extra OPTIONS round-trip first).

### Deciding the path: is it a "simple request"?

A request is **simple** (skips preflight) if **ALL** of these hold:

```
1. Method is one of:   GET, HEAD, POST                (nothing else)

2. Headers are ONLY from the CORS-safelist:
      Accept
      Accept-Language
      Content-Language
      Content-Type       (but restricted — see #3)
      Range              (with limits)
   Any custom header (Authorization, X-Request-Id, X-Api-Key, ...) → NOT simple

3. If Content-Type is set, it is ONLY one of:
      application/x-www-form-urlencoded
      multipart/form-data
      text/plain
   → Content-Type: application/json  breaks "simple". This is the #1 preflight trigger.

4. No ReadableStream body, no event listeners on the upload,
   no unusual fetch options.
```

If any condition fails, the request is **preflighted**.

> ⚠️ **The trap:** almost every real JSON API call is **preflighted**, because sending `Content-Type: application/json` (condition 3) or an `Authorization` header (condition 2) both disqualify "simple." Do not assume your API only ever gets simple requests.

### Path A — Simple request (one round-trip)

```
┌──────────┐                                        ┌──────────────┐
│ BROWSER  │                                        │  api.example │
│(app.exam)│                                        │              │
└────┬─────┘                                        └──────┬───────┘
     │  GET /users HTTP/1.1                                │
     │  Origin: https://app.example.com                   │
     │───────────────────────────────────────────────────►│
     │                                                     │  runs handler,
     │                                                     │  DB query, etc.
     │  HTTP/1.1 200 OK                                    │
     │  Access-Control-Allow-Origin: https://app.example  │
     │  Content-Type: application/json                     │
     │  [{"id":1,"name":"Alice"}]                          │
     │◄───────────────────────────────────────────────────│
     │                                                     │
     ▼
  Browser checks: does A-C-A-Origin match my origin (or is it *)?
     YES → hand the JSON to JavaScript
     NO  → BLOCK. Throw "blocked by CORS policy" in console.
           (JS gets a rejected promise / opaque error — never sees the body)
```

**Notice:** even when blocked, **the server already ran the request and sent the response.** The block happens on the *client side, after the response arrives.* This is why CORS is worthless as server protection — the side effect already happened.

### Path B — Preflighted request (two round-trips)

```
┌──────────┐                                        ┌──────────────┐
│ BROWSER  │                                        │  api.example │
└────┬─────┘                                        └──────┬───────┘
     │                                                     │
     │  ── PREFLIGHT (automatic, you never write this) ──  │
     │                                                     │
     │  OPTIONS /users HTTP/1.1                            │
     │  Origin: https://app.example.com                   │
     │  Access-Control-Request-Method: PUT                │
     │  Access-Control-Request-Headers: content-type,     │
     │                                   authorization     │
     │───────────────────────────────────────────────────►│
     │                                                     │  server answers
     │                                                     │  the preflight
     │  HTTP/1.1 204 No Content                            │  (NO body, NO
     │  Access-Control-Allow-Origin: https://app.example  │   handler logic)
     │  Access-Control-Allow-Methods: GET,POST,PUT,DELETE │
     │  Access-Control-Allow-Headers: content-type,       │
     │                                authorization        │
     │  Access-Control-Max-Age: 600                       │
     │◄───────────────────────────────────────────────────│
     │                                                     │
     │  Browser checks the preflight answer:               │
     │    • is PUT in Allow-Methods?          ✔            │
     │    • are my headers in Allow-Headers?  ✔            │
     │    • does Allow-Origin match?          ✔            │
     │    ALL PASS → send the real request.                │
     │    ANY FAIL → BLOCK. Real request never sent.       │
     │                                                     │
     │  ── THE REAL REQUEST ──                             │
     │                                                     │
     │  PUT /users HTTP/1.1                                │
     │  Origin: https://app.example.com                   │
     │  Content-Type: application/json                     │
     │  Authorization: Bearer eyJ...                       │
     │  {"name":"Bob"}                                     │
     │───────────────────────────────────────────────────►│  NOW the real
     │                                                     │  handler runs
     │  HTTP/1.1 200 OK                                    │
     │  Access-Control-Allow-Origin: https://app.example  │  ← must be present
     │  {"id":2,"name":"Bob"}                              │     AGAIN here
     │◄───────────────────────────────────────────────────│
     ▼
  Browser checks Allow-Origin on the REAL response too → deliver to JS.
```

**Two critical points about preflight:**

1. The preflight (`OPTIONS`) must be answered by your **CORS layer/middleware, not your route handler**. If your `PUT /users` handler is the only thing registered and it doesn't accept `OPTIONS`, the preflight gets a `404`/`405`, the browser blocks, and your real request never even leaves.
2. `Access-Control-Max-Age: 600` tells the browser: *"cache this preflight decision for 600 seconds."* During that window, repeated calls to the same endpoint skip the OPTIONS round-trip. This is a real latency win — without it, **every** non-simple request pays for two round-trips instead of one.

---

## Exact Syntax Breakdown

### Inspecting CORS headers with curl (`-H "Origin:"` + `-v`)

`curl` never enforces CORS, but you can make it *send the Origin header a browser would send* and *read the headers the server returns* — that's exactly how you verify a server is configured correctly.

```
curl  -v   -H "Origin: https://app.example.com"   https://api.example.com/users
│     │    │                                        │
│     │    │                                        └── the endpoint under test
│     │    └── manually add the Origin header a browser would auto-add
│     └── verbose: print request (>) and response (<) headers, incl. Access-Control-*
└── transfer URL
```

What the response tells you:

```
< HTTP/1.1 200 OK
< access-control-allow-origin: https://app.example.com   ← ✅ server permits your origin
< vary: Origin                                           ← ✅ cache-safe
< content-type: application/json
```

If `access-control-allow-origin` is **absent** from the response, a browser would block it. `curl` still shows you the body — because `curl` doesn't care.

### Simulating a preflight with curl (`-X OPTIONS`)

```
curl -v -X OPTIONS \
  -H "Origin: https://app.example.com" \
  -H "Access-Control-Request-Method: PUT" \
  -H "Access-Control-Request-Headers: content-type, authorization" \
  https://api.example.com/users
│  │  │                     │
│  │  │                     └── the two headers a browser attaches to a preflight
│  │  └── (the Origin header, as always)
│  └── force the HTTP method to OPTIONS (the preflight method)
└── inspect what the server answers to the preflight
```

A correctly-configured server replies:

```
< HTTP/1.1 204 No Content
< access-control-allow-origin: https://app.example.com
< access-control-allow-methods: GET, POST, PUT, DELETE
< access-control-allow-headers: content-type, authorization
< access-control-max-age: 600
```

If `access-control-allow-methods` doesn't list `PUT`, or `access-control-allow-headers` doesn't list `authorization`, the real browser request would be **blocked at the preflight stage** — you'd see the failure here, in the OPTIONS response, before the real request is ever attempted.

---

## Example 1 — Basic

**Goal:** see the difference between what `curl` returns and what a browser enforces, using a public echo service.

```bash
# 1. Ask a public API, sending a browser-like Origin header.
#    httpbin echoes back the CORS headers it sends.
curl -v -H "Origin: https://app.example.com" https://httpbin.org/get 2>&1 | grep -i "access-control\|origin"
```

Expected (annotated):

```
> Origin: https://app.example.com                     ← what WE sent (browser would auto-send)
< access-control-allow-origin: *                      ← httpbin allows ANY origin
< access-control-allow-credentials: true              ← (httpbin sets this; see the wildcard trap later)
```

Because `access-control-allow-origin: *`, a real browser on **any** website could read this response. Now compare an endpoint with **no** CORS headers:

```bash
# 2. A server that returns NO Access-Control-Allow-Origin header.
curl -v -H "Origin: https://app.example.com" https://example.com/ 2>&1 | grep -i "access-control"
```

```
(no output — there are NO access-control-* headers)
```

**Interpretation:**

- With `curl`: both requests **succeed and return a body**, because `curl` ignores CORS entirely.
- In a **browser**: request #1's response would be readable by your JS (wildcard allows it). Request #2's response would be **blocked** — `example.com` sends no `Access-Control-Allow-Origin`, so the browser refuses to hand the body to cross-origin JavaScript, and you'd see `blocked by CORS policy` in the console.

Same servers, same bytes — the *only* difference is whether a browser is the one making the call.

---

## Example 2 — Production Scenario

**The situation.** Your React app is served from `https://app.acme.io`. It calls your Node/Express API at `https://api.acme.io`. Users report the dashboard is blank. The frontend team sends you this console screenshot:

```
Access to fetch at 'https://api.acme.io/orders' from origin
'https://app.acme.io' has been blocked by CORS policy: No
'Access-Control-Allow-Origin' header is present on the requested resource.

GET https://api.acme.io/orders net::ERR_FAILED 200 (OK)
```

Note the `200 (OK)` — **the server answered successfully.** The browser blocked the *reading* of a perfectly good response. This is a CORS config problem, not a bug in your handler.

### Step 1 — Reproduce it the way the browser sees it (with curl)

```bash
# Simulate the browser's actual cross-origin GET
curl -s -D - -o /dev/null \
  -H "Origin: https://app.acme.io" \
  https://api.acme.io/orders
```

```
HTTP/1.1 200 OK
content-type: application/json
content-length: 1284
                          ← NO access-control-allow-origin header anywhere
```

Confirmed: the server never sends `Access-Control-Allow-Origin`. The frontend cannot fix this — there is no JavaScript that makes a missing *response* header appear.

### Step 2 — Check the preflight too (this API uses `Authorization` + JSON)

```bash
curl -s -D - -o /dev/null -X OPTIONS \
  -H "Origin: https://app.acme.io" \
  -H "Access-Control-Request-Method: POST" \
  -H "Access-Control-Request-Headers: authorization, content-type" \
  https://api.acme.io/orders
```

```
HTTP/1.1 404 Not Found          ← the OPTIONS preflight isn't handled at all
```

Second problem found: even if we fix `Allow-Origin`, any `POST` with `Authorization` (a preflighted request) will die at the preflight, because `OPTIONS /orders` returns `404`.

### Step 3 — Fix it on the SERVER (the only place it can be fixed)

Using the `cors` middleware in Express, configured correctly:

```javascript
const cors = require("cors");

app.use(cors({
  // Echo back a SPECIFIC origin — required because we use credentials.
  // Never "*" here (see Common Mistakes).
  origin: "https://app.acme.io",

  methods: ["GET", "POST", "PUT", "DELETE"],
  allowedHeaders: ["Authorization", "Content-Type"],

  credentials: true,        // cookies / Authorization allowed to be read
  maxAge: 600,              // cache preflight for 10 min → fewer OPTIONS round-trips
}));

// cors() middleware AUTOMATICALLY answers OPTIONS preflights before your
// routes run — this is what fixes the 404 on the preflight.
```

### Step 4 — Verify the fix with the same curl commands

```bash
# Real request now carries the permission slip:
curl -s -D - -o /dev/null -H "Origin: https://app.acme.io" https://api.acme.io/orders
```
```
HTTP/1.1 200 OK
access-control-allow-origin: https://app.acme.io    ← ✅ specific origin echoed
access-control-allow-credentials: true              ← ✅
vary: Origin                                         ← ✅ cache-correct
content-type: application/json
```

```bash
# Preflight now answered:
curl -s -D - -o /dev/null -X OPTIONS \
  -H "Origin: https://app.acme.io" \
  -H "Access-Control-Request-Method: POST" \
  -H "Access-Control-Request-Headers: authorization, content-type" \
  https://api.acme.io/orders
```
```
HTTP/1.1 204 No Content
access-control-allow-origin: https://app.acme.io
access-control-allow-methods: GET, POST, PUT, DELETE
access-control-allow-headers: Authorization, Content-Type
access-control-allow-credentials: true
access-control-max-age: 600
```

**The lesson to hand the frontend team:** "There was nothing you could do — the fix was three lines of server config. Here's the curl command you can run yourself next time to prove whether it's a CORS/header problem before pinging me."

---

## Common Mistakes

### Mistake 1: Thinking CORS protects your API

```
WRONG belief: "My CORS policy only allows app.acme.io, so attackers can't hit my API."
```
**Reality:** CORS is enforced by *browsers*. An attacker with `curl`, a script, or their own server ignores it completely:
```bash
# This works no matter what your CORS policy says — no Origin restriction applies:
curl https://api.acme.io/orders -H "Authorization: Bearer <stolen-token>"
```
CORS protects **your users' browsers** from *other* websites silently using *their* credentials. It is **not** access control for your API. Real protection = **authentication + authorization** (Topic 16). Do not treat a restrictive CORS policy as a security boundary against non-browser clients.

---

### Mistake 2: Using `*` together with credentials

```javascript
// BROKEN combination:
res.setHeader("Access-Control-Allow-Origin", "*");
res.setHeader("Access-Control-Allow-Credentials", "true");
```
The browser **rejects this by spec**. Console error:
```
The value of the 'Access-Control-Allow-Origin' header in the response must
not be the wildcard '*' when the request's credentials mode is 'include'.
```
**Why:** `*` means "any website may read this." Combined with "and send the user's cookies," that would let *every* site on the internet make authenticated requests as your logged-in user and read the results. The spec forbids it. **Fix:** echo the *specific* requesting origin (validated against an allowlist) and add `Vary: Origin`:
```javascript
const allowed = ["https://app.acme.io", "https://admin.acme.io"];
const origin = req.headers.origin;
if (allowed.includes(origin)) {
  res.setHeader("Access-Control-Allow-Origin", origin);   // echo the exact origin
  res.setHeader("Access-Control-Allow-Credentials", "true");
  res.setHeader("Vary", "Origin");                         // don't cache-poison
}
```

---

### Mistake 3: Forgetting to handle the OPTIONS preflight

```
Symptom: GET works fine, but POST/PUT/DELETE with JSON "randomly" fails with CORS.
Console: Response to preflight request doesn't pass access control check.
Network tab: an OPTIONS request returning 404, 405, or 401.
```
**Root cause:** your framework routes `OPTIONS /resource` to nothing (or to auth middleware that 401s it before CORS runs). The browser sends the preflight, gets a non-2xx, and never sends the real request. **Fix:** ensure your CORS middleware runs **first** and answers `OPTIONS` for all routes. Preflights must **not** require authentication — a browser never sends credentials on a preflight, so gating OPTIONS behind auth guarantees a 401 and a block.

---

### Mistake 4: Forgetting `Vary: Origin` (cache poisoning)

```
Scenario: A CDN or reverse proxy (Topic 29/30) caches your API response.
Request from app.acme.io  → server echoes Access-Control-Allow-Origin: https://app.acme.io
                          → CDN caches THAT response (with that header)
Request from admin.acme.io → CDN serves the CACHED copy
                          → browser sees Allow-Origin: https://app.acme.io ≠ admin.acme.io
                          → BLOCKED, even though the server would have allowed it
```
**Fix:** whenever `Access-Control-Allow-Origin` depends on the request's `Origin` (i.e. you're echoing it, not sending `*`), you **must** send `Vary: Origin`. This tells caches "the response differs per Origin — key the cache on it." Without it, one origin's allow-header gets served to a different origin and CORS breaks intermittently and unreproducibly.

---

### Mistake 5: Trying to fix CORS in frontend code

```javascript
// None of these do anything useful:
fetch(url, { headers: { "Access-Control-Allow-Origin": "*" } });  // wrong side!
fetch(url, { mode: "no-cors" });   // makes the response OPAQUE — you can't read it
```
`Access-Control-Allow-Origin` is a **response** header — only the server can set it; setting it on the request is meaningless. `mode: "no-cors"` doesn't bypass anything; it just makes the browser give you an unreadable opaque response. The Origin header itself is **browser-controlled** and cannot be changed by JS. **There is no frontend fix.** The options are: fix the server's headers, or (for local dev only) route through a same-origin proxy. Disabling CORS in the browser (`--disable-web-security`) is a dev-only hack, never a fix.

---

### Mistake 6: Confusing a CORS error with a network or auth error

```
"blocked by CORS policy"        → header/preflight problem. Server reachable. Fix headers.
"net::ERR_CONNECTION_REFUSED"   → nothing listening / wrong port. NOT CORS.
"net::ERR_NAME_NOT_RESOLVED"    → DNS failed. NOT CORS.
401 Unauthorized / 403 Forbidden→ request reached the app, auth failed. NOT CORS.
                                   (You can READ a 401 body only if CORS headers are present!)
```
**Diagnosis rule:** run the exact request with `curl -v -H "Origin: ..."`. If curl **succeeds and returns a body**, it's a CORS/header problem (browser-only). If curl **also fails** (refused, timeout, DNS, 401), it's a *real* network or auth problem that has nothing to do with CORS. This one test collapses hours of confusion.

---

## Hands-On Proof

```bash
# 1. Prove the server sends a permissive CORS header (httpbin allows all origins)
curl -s -D - -o /dev/null -H "Origin: https://evil.example" https://httpbin.org/get \
  | grep -i access-control
#   → access-control-allow-origin: *   (server permits any origin)

# 2. Prove curl ignores CORS entirely: a "disallowed" origin STILL gets the body
curl -s -H "Origin: https://evil.example" https://httpbin.org/get | head -5
#   → you get JSON back. curl has no Same-Origin Policy. A browser would gate this.

# 3. Simulate a full preflight and read the server's answer
curl -s -D - -o /dev/null -X OPTIONS \
  -H "Origin: https://app.example.com" \
  -H "Access-Control-Request-Method: DELETE" \
  -H "Access-Control-Request-Headers: authorization" \
  https://httpbin.org/anything \
  | grep -i "access-control\|HTTP/"
#   → read which methods/headers the server says it allows

# 4. See the browser side for real: open ANY site, press F12 → Console, paste:
#    fetch("https://api.github.com").then(r=>r.json()).then(console.log)   // works: GitHub sends CORS *
#    fetch("https://example.com").then(r=>r.text())                        // BLOCKED: no CORS header
#    Watch the Network tab: the request SUCCEEDS (200) but JS is blocked from the body.

# 5. Watch a preflight fire in the browser Network tab:
#    fetch("https://httpbin.org/anything", {
#      method: "PUT",
#      headers: { "Content-Type": "application/json" },   // triggers preflight
#      body: JSON.stringify({x:1})
#    })
#    → Network tab shows an OPTIONS request appear BEFORE the PUT.
```

---

## Practice Exercises

### Exercise 1 — Easy: Identify same-origin vs cross-origin

```
For each pair, decide same-origin or cross-origin, and if cross, name which
part of (scheme, host, port) differs. Base page: https://shop.acme.io

  a) https://shop.acme.io/cart          → ?
  b) http://shop.acme.io                → ?
  c) https://api.acme.io                → ?
  d) https://shop.acme.io:8443          → ?
  e) https://www.shop.acme.io           → ?

Then: which of a–e would need CORS headers from the server for JS on the base
page to read their responses?
```

### Exercise 2 — Medium: Predict simple vs preflight

```
For each fetch, predict "simple (no preflight)" or "preflighted (OPTIONS first)",
and give the reason. Verify each by watching the Network tab in a browser.

  a) fetch("https://api.x.io/items")                       // GET, no extra headers
  b) fetch("https://api.x.io/items", {
       method: "POST",
       headers: {"Content-Type": "application/json"},
       body: "{}"
     })
  c) fetch("https://api.x.io/items", {
       method: "POST",
       headers: {"Content-Type": "text/plain"},
       body: "hi"
     })
  d) fetch("https://api.x.io/items", {
       headers: {"Authorization": "Bearer abc"}
     })
  e) fetch("https://api.x.io/items", { method: "DELETE" })

# Hint: re-read the four "simple request" conditions.
```

### Exercise 3 — Hard (Production Simulation): Diagnose and fix a credentialed-CORS failure

```
Setup: Frontend https://app.acme.io calls https://api.acme.io/me with
credentials: "include" (cookie-based session, Topic 16). Users are logged out
instantly. Console shows:

  The value of the 'Access-Control-Allow-Origin' header in the response must
  not be the wildcard '*' when the request's credentials mode is 'include'.

Your tasks:
  1. Reproduce with curl. Which single curl command reveals the bad header?
     (Write it, including the Origin header and -D -.)
  2. The server currently returns:
        access-control-allow-origin: *
        access-control-allow-credentials: true
     Explain in one sentence why the browser rejects this specific combination.
  3. Write the corrected server response headers (list them) that would make a
     credentialed request from app.acme.io succeed AND stay cache-safe behind a CDN.
  4. The preflight for GET /me also 401s because auth middleware runs before CORS.
     Explain why a browser preflight can NEVER pass auth, and state the fix.
  5. Bonus: the frontend also needs to read a custom `X-Session-Expiry` response
     header via JS but gets `undefined`. Which ONE response header fixes that?
```

---

## Mental Model Checkpoint

Answer from memory. If you can't, re-read the relevant section.

1. **Who enforces CORS — the browser or the server? What does the server's role reduce to?** And why does the identical request "work in curl but fail in the browser"?

2. **Define an origin precisely.** Are `https://acme.io` and `https://www.acme.io` the same origin? Are `https://acme.io` and `https://acme.io:8443`?

3. **List the conditions that make a request "simple."** Why does `Content-Type: application/json` almost always force a preflight?

4. **Walk through a preflight:** which request method does the browser use, which two request headers does it add, and which response headers must the server return for the real request to proceed?

5. **Why is `Access-Control-Allow-Origin: *` incompatible with `Access-Control-Allow-Credentials: true`?** What must the server send instead, and which extra header keeps caches correct?

6. **A `POST` with `Authorization` and JSON body fails, but a plain `GET` to the same server works. The Network tab shows an `OPTIONS` returning 404. What is broken and where is the fix?**

7. **Your console shows `blocked by CORS policy`. Give the single curl command that tells you whether this is truly a CORS/header problem versus a real network/auth failure, and how to read its result.**

---

## Quick Reference Card

### Every CORS header

| Header | Direction | Set by | Meaning |
|--------|-----------|--------|---------|
| `Origin` | request | **browser (auto)** | The origin making the request. JS can't set/change it. |
| `Access-Control-Request-Method` | request (preflight) | browser (auto) | "The real request will use this method." |
| `Access-Control-Request-Headers` | request (preflight) | browser (auto) | "The real request will send these headers." |
| `Access-Control-Allow-Origin` | response | **server** | Which origin may read the response. Specific origin, or `*`. |
| `Access-Control-Allow-Methods` | response (preflight) | server | Methods permitted for the real request. |
| `Access-Control-Allow-Headers` | response (preflight) | server | Request headers permitted for the real request. |
| `Access-Control-Allow-Credentials` | response | server | `true` = cookies/Authorization may be sent & response read. |
| `Access-Control-Expose-Headers` | response | server | Which *response* headers JS is allowed to read. |
| `Access-Control-Max-Age` | response (preflight) | server | Seconds the browser may cache the preflight result. |
| `Vary: Origin` | response | server | Cache correctness when `Allow-Origin` echoes the origin. |

### Simple vs Preflight cheat block

```
SIMPLE (no OPTIONS)                 PREFLIGHTED (OPTIONS first)
─────────────────────              ────────────────────────────
method ∈ {GET,HEAD,POST}           method ∈ {PUT,PATCH,DELETE,...}
only safelisted headers            any custom header (Authorization, X-*)
Content-Type ∈ {                   Content-Type: application/json
  x-www-form-urlencoded,           (or any non-safelisted value)
  multipart/form-data,
  text/plain }
```

### The wildcard-vs-credentials rule

```
Allow-Credentials: true   ⇒   Allow-Origin MUST be a specific origin (echoed),
                              NEVER "*"   → and add   Vary: Origin

Allow-Origin: *           ⇒   NO credentials. Cookies/Authorization are ignored,
                              response readable by any origin.
```

### Diagnostic commands

```bash
# Inspect CORS headers as a browser-origin request
curl -v -H "Origin: https://app.example.com" https://api.example.com/path

# Simulate a preflight
curl -v -X OPTIONS \
  -H "Origin: https://app.example.com" \
  -H "Access-Control-Request-Method: PUT" \
  -H "Access-Control-Request-Headers: authorization,content-type" \
  https://api.example.com/path

# curl succeeds + returns body  → CORS/header problem (browser-only). Fix server headers.
# curl also fails (refused/DNS/401) → real network/auth problem, NOT CORS.
```

---

## When Would I Use This at Work?

### Scenario 1: "The frontend team says your API is down but it works in Postman"

Postman and `curl` ignore CORS; the browser doesn't. It's not down — it's missing `Access-Control-Allow-Origin` (or the preflight isn't handled). Reproduce with `curl -v -H "Origin: <their-site>"`, see the missing header, add the server-side CORS config. Hand them the curl one-liner so they can self-diagnose next time.

### Scenario 2: "We switched from Bearer tokens in a header to cookie sessions and everything broke"

Cookie auth means `credentials: "include"`, which activates the strict credentialed-CORS rules. Your old `Access-Control-Allow-Origin: *` is now illegal. Switch to echoing the specific origin from an allowlist, add `Access-Control-Allow-Credentials: true` and `Vary: Origin`. Also confirm your cookie's `SameSite` setting (Topic 16/19) — cross-origin cookies need `SameSite=None; Secure`.

### Scenario 3: "Our API got slow after we added a required request header"

Adding `X-Request-Id` (or any custom header) turned every call into a preflighted request — two round-trips instead of one. On a high-latency link (Topic 01) that doubles perceived latency. Fix: add `Access-Control-Max-Age` so the browser caches the preflight decision, drastically cutting the number of OPTIONS round-trips.

### Scenario 4: "Security asked us to 'lock down CORS' to secure the API"

Educate them: CORS is not an API access control. Non-browser clients bypass it entirely. Tightening `Access-Control-Allow-Origin` protects *your users' browsers* from other sites riding their sessions — it does **not** stop a determined attacker with `curl`. The API's real protection is authentication and authorization (Topic 16). Do both — but don't confuse them.

---

## Connected Topics

**Study before this:**
- `01-how-the-internet-works.md` — why cross-origin latency and preflight round-trips cost real time
- `10-http1.1-in-depth.md` — the request/response and header format CORS rides on; the `OPTIONS` method
- `14-http-headers-in-depth.md` — headers, `Vary`, and content negotiation that CORS builds upon

**Study alongside / next:**
- `16-authentication-over-http.md` — cookies, `SameSite`, and why credentialed CORS is strict; the real API access-control layer CORS is *not*
- `18-rate-limiting-at-http-level.md` — remember to `Expose-Headers` your `X-RateLimit-*` so the frontend can actually read them
- `19-cookies-and-sessions.md` — `SameSite=None; Secure` is the cookie half of the cross-origin credentials story

**Connects forward to:**
- `29-reverse-proxies.md` / `30-cdns.md` — where `Vary: Origin` and preflight caching decisions actually get made in production
- `34-api-gateways.md` — CORS is frequently centralized/offloaded at the gateway rather than each service

---

*Doc saved: `/docs/networking/17-cors-in-depth.md`*
