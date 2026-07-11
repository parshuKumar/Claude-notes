# 16 — Authentication over HTTP

> **Phase 3 — Topic 3 of 7**
> Prerequisites: `01-how-the-internet-works.md`, `09-tls-ssl-in-depth.md`, `14-http-headers-in-depth.md`

---

## ELI5 — The Simple Analogy

Imagine a members-only club.

- **Authentication** is the bouncer checking your ID at the door: *"Are you really who you say you are?"* You show your driver's license, he confirms your face matches the photo, and lets you in.
- **Authorization** is what happens *after* you're inside: the VIP lounge has a second guard who checks whether your wristband is gold or silver. You're definitely a real member (authenticated), but you might not be allowed into that room (not authorized).

Now, HTTP has a problem: it has **amnesia**. Every request is a brand-new stranger walking up to the door. The server does not remember that you showed your ID thirty seconds ago. So on *every single request* you must prove who you are again.

There are three common ways to keep proving it:

1. **Basic auth** — you shout your username and password through the door every time. Simple, but everyone standing nearby can hear you unless you're in a soundproof room (TLS).
2. **A token** — the bouncer gives you a numbered ticket. On future visits you just flash the ticket instead of your ID.
3. **A cookie** — the club stamps your hand with invisible ink. Your browser automatically shows the stamp on every visit without you thinking about it.

This topic is about what those three things actually look like **on the wire** — the exact bytes in the HTTP headers.

---

## Where This Lives in the Network Stack

Authentication is a **Layer 7 (Application)** concern. It rides *inside* HTTP, which rides inside TLS, which rides inside TCP/IP.

```
┌─────────────────────────────────────────────────────────────────┐
│  Layer 7 — Application   (HTTP)                                  │
│      ┌──────────────────────────────────────────────────────┐   │
│      │  Authorization: Bearer eyJhbGc...   ← AUTH LIVES HERE │   │  ← just an HTTP header
│      │  Cookie: session=abc123             ← AUTH LIVES HERE │   │
│      └──────────────────────────────────────────────────────┘   │
│  Layer 6 — Presentation  (TLS/SSL)   ← encrypts the header above │  ← makes auth safe
│  Layer 4 — Transport     (TCP — port 443)                       │
│  Layer 3 — Network       (IP)                                   │
│  Layer 2 — Data Link     (Ethernet, Wi-Fi)                      │
│  Layer 1 — Physical      (fiber, radio)                         │
└─────────────────────────────────────────────────────────────────┘
```

**The critical relationship: authentication is worthless without the layer below it.** An `Authorization` header or a session cookie is just plaintext bytes. If it travels over plain HTTP (no TLS), anyone on the path — coffee-shop Wi-Fi, your ISP, a compromised router — reads it and *becomes you*. This is why every section below assumes TLS (Topic 09) is already in place. **Auth over HTTP is really "auth over HTTPS."**

---

## What Is This?

**Authentication** is the process by which a server confirms the identity behind an HTTP request. Because HTTP is **stateless** — the protocol itself carries no memory between requests — the client must attach proof of identity to *every* request that needs it.

The proof is carried in one of two places:

| Carrier | Header on the wire | Who sets it | Typical use |
|---------|--------------------|-------------|-------------|
| **Authorization header** | `Authorization: <scheme> <credentials>` | The client (your code) sets it explicitly | APIs, mobile apps, server-to-server |
| **Cookie header** | `Cookie: name=value` | The **browser** sets it automatically | Websites, browser sessions |

Within those two carriers, the common schemes are:

- **Basic** — `Authorization: Basic base64(user:pass)`
- **Bearer** — `Authorization: Bearer <token>` (token is often a **JWT** or an **opaque** string)
- **Session cookie** — `Cookie: sid=<opaque session id>`

Two big architectural families sit on top of these carriers:

- **Stateful sessions** — the server stores session data; the cookie holds only a meaningless ID that points to it.
- **Stateless tokens (JWT)** — the token *itself* carries the identity claims, signed so the server can trust them without a lookup.

The rest of this doc is the wire-level anatomy of all of the above.

---

## Why Does It Matter for a Backend Developer?

Authentication is where correctness and security collide hardest. Getting it wrong is not a slow endpoint — it's a breach. You need this to:

- **Choose the right scheme.** Session vs JWT is one of the most common (and most botched) design decisions in backend work. Pick wrong and you'll either fight revocation problems for years or bolt on a session store you swore you'd never need.
- **Read the 401 vs 403 distinction correctly.** These are *not* interchangeable. Returning the wrong one leaks information or confuses every client that consumes your API.
- **Set cookie attributes that actually protect users.** A single missing `HttpOnly`, `Secure`, or `SameSite` flag is the difference between "session cookie" and "session cookie any XSS or CSRF can steal."
- **Debug the invisible.** When a cross-origin fetch silently drops your cookie, or a JWT is rejected as expired, or Basic auth loops forever asking for a password — you fix it by reading the exact headers on the wire, not by guessing.
- **Not leak secrets.** Half the industry believes base64 is encryption and that JWT payloads are private. Both beliefs cause real incidents. You should never be in that half.

---

## The Packet/Protocol Anatomy

### Authentication vs Authorization — get this straight first

```
┌────────────────────────────────────────────────────────────────┐
│  AUTHENTICATION  →  "Who are you?"     → proven by credentials   │
│                     Result: identity established (or not)        │
│                     Failure status:  401 Unauthorized            │
│                                                                  │
│  AUTHORIZATION   →  "What may you do?" → checked against perms   │
│                     Result: action allowed (or not)              │
│                     Failure status:  403 Forbidden               │
└────────────────────────────────────────────────────────────────┘
```

Naming trap: the HTTP header is called `Authorization`, but it carries **authentication** credentials. And the status code `401 Unauthorized` actually means **"unauthenticated."** The HTTP spec's vocabulary is historically backwards. Memorize the meanings, not the words.

### 401 vs 403 on the wire

```
401 Unauthorized  →  "I don't know who you are (yet)."
                     You sent NO credentials, or BAD/EXPIRED credentials.
                     The server INVITES you to authenticate.
                     MUST include a WWW-Authenticate header (the "challenge").
                     Retrying WITH valid credentials can succeed.

403 Forbidden     →  "I know exactly who you are. You still can't do this."
                     Your credentials were valid — identity is established.
                     You lack permission for this resource/action.
                     Retrying with the SAME credentials will NEVER succeed.
                     No WWW-Authenticate; there's nothing to re-try.
```

A 401 response from a Basic-auth-protected resource looks like this on the wire:

```
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Basic realm="Admin Area", charset="UTF-8"
Content-Length: 0
```

That `WWW-Authenticate` header is the **challenge**. It tells the client *how* to authenticate (`Basic`) and names a `realm` (a label for the protected area). A browser receiving this pops up the native username/password dialog.

### The Authorization header format

```
Authorization: <scheme> <credentials>
               │        │
               │        └── scheme-specific; for Basic it's base64, for Bearer it's the raw token
               └── "Basic", "Bearer", "Digest", "Negotiate", ...

Examples on the wire:
  Authorization: Basic YWxpY2U6czNjcmV0          ← user:pass, base64-encoded
  Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...  ← a token (opaque or JWT)
```

### Basic auth — base64 is NOT encryption

```
Credentials = base64( username + ":" + password )

  alice:s3cret
      │  encode with base64 (reversible, no key, no secret)
      ▼
  YWxpY2U6czNjcmV0

Header on the wire:
  Authorization: Basic YWxpY2U6czNjcmV0
```

Anyone can reverse it instantly — there is no key involved:

```bash
$ echo -n 'YWxpY2U6czNjcmV0' | base64 -d
alice:s3cret
```

**base64 is an *encoding*, not *encryption*.** Encoding = a reversible transform anyone can undo. Encryption = requires a secret key to undo. Basic auth over plain HTTP means your password travels in effectively-plaintext form on every request. The *only* thing protecting it is the TLS layer wrapping the whole HTTP message. Basic auth without TLS = shouting your password across the room.

### JWT structure on the wire

A JWT (JSON Web Token) is three base64url segments joined by dots:

```
       header            .            payload             .          signature
┌──────────────────┐        ┌──────────────────────┐        ┌──────────────────┐
eyJhbGciOiJIUzI1NiI  .  eyJzdWIiOiIxMjMiLCJleHAiOjE  .  SflKxwRJSMeKKF2QT4fwp
sInR5cCI6IkpXVCJ9        3MzAwMDAwMDB9                    MeJf36POk6yJV_adQssw5c

  base64url(header)      base64url(payload/claims)     signature over header.payload

DECODED header:                 DECODED payload (claims):
{                               {
  "alg": "HS256",   ← algorithm   "sub": "123",     ← subject (the user ID)
  "typ": "JWT"                     "iss": "auth.myapp.com", ← issuer
}                                  "aud": "api.myapp.com",  ← audience
                                   "iat": 1730000000, ← issued-at (unix time)
                                   "exp": 1730003600, ← expiry (unix time)
                                   "role": "admin"    ← custom claim
                                 }
```

**Critical fact: the payload is only base64url-encoded, NOT encrypted.** Anyone holding the token can decode and read every claim. The signature does not hide the payload — it only *proves the payload wasn't tampered with*. So:

```
JWT payload  →  READABLE by anyone (it's just base64)
             →  TRUSTWORTHY because the signature would break if altered
             →  NEVER put passwords, card numbers, or secrets in it
```

### Signature: HS256 vs RS256

```
HS256 (HMAC-SHA256) — SYMMETRIC
   signature = HMAC_SHA256( header.payload , shared_secret )
   Same secret both signs AND verifies.
   ✓ Simple, fast.
   ✗ Every service that verifies must hold the signing secret → can also forge tokens.

RS256 (RSA-SHA256) — ASYMMETRIC
   signature = RSA_sign( header.payload , PRIVATE_key )
   verify with the PUBLIC key.
   ✓ Auth server signs with private key; anyone verifies with public key.
   ✓ Verifiers CANNOT forge tokens (they only have the public key).
   → The standard for OAuth / multi-service systems.
```

### Cookie for auth — Set-Cookie and the round-trip

The server issues a cookie once; the browser stores it and **automatically** attaches it to every subsequent matching request.

```
── Response (server → browser), sets the cookie ──
HTTP/1.1 200 OK
Set-Cookie: session=8f3b2a...; HttpOnly; Secure; SameSite=Lax; Path=/; Max-Age=86400
            │                  │         │       │              │      │
            │                  │         │       │              │      └── lifetime in seconds
            │                  │         │       │              └── which paths send it
            │                  │         │       └── CSRF control (Strict/Lax/None)
            │                  │         └── HTTPS-only (never sent over plain HTTP)
            │                  └── JS cannot read document.cookie for this one (XSS defense)
            └── the cookie name=value (an opaque session id here)

── Every later Request (browser → server), browser resends automatically ──
GET /account HTTP/1.1
Host: myapp.com
Cookie: session=8f3b2a...
        │
        └── note: ONLY name=value comes back. Attributes (HttpOnly, Secure…) are
            instructions to the BROWSER and are NOT echoed back to the server.
```

---

## How It Works — Step by Step

### Flow A — HTTP Basic auth challenge/response

```
Client                                             Server
  │  GET /admin  (no credentials) ──────────────────►│  no Authorization header → challenge
  │◄── 401 Unauthorized ─────────────────────────────│
  │    WWW-Authenticate: Basic realm="Admin Area"    │
  │                                                  │
  │  (browser shows login dialog; user types creds)  │
  │  GET /admin ─────────────────────────────────────►│  base64-decode → alice:s3cret
  │    Authorization: Basic YWxpY2U6czNjcmV0         │  verify against user store
  │◄── 200 OK  (+ the protected content) ────────────│
  │                                                  │
  │  (browser now RESENDS the header on every request to this realm)
```

### Flow B — Bearer token (login once, carry a token)

```
Client                                             Server
  │  POST /login {user, pass} ──────────────────────►│  verify creds ONCE; mint token (JWT/opaque)
  │◄── 200 OK { "access_token": "eyJ...", "exp": …} ─│
  │  (client stores token)                           │
  │  GET /api/orders ────────────────────────────────►│  validate token:
  │    Authorization: Bearer eyJ...                  │    opaque → look up in store
  │◄── 200 OK  [ ...orders... ] ─────────────────────│    JWT    → verify signature + exp
```

### Flow C — Session cookie (stateful)

```
Client (browser)                                   Server + Session Store
  │  POST /login {user, pass} ──────────────────────►│  verify creds; CREATE session:
  │◄── 200 OK ───────────────────────────────────────│    sid "8f3b2a" → {userId:123, role:admin}
  │    Set-Cookie: session=8f3b2a; HttpOnly; Secure  │
  │  (browser stores cookie)                         │
  │  GET /account ───────────────────────────────────►│  LOOK UP sid in store
  │    Cookie: session=8f3b2a   (auto-attached)      │    → {userId:123, role:admin}
  │◄── 200 OK ───────────────────────────────────────│
  │  POST /logout ───────────────────────────────────►│  DELETE sid from store
  │◄── Set-Cookie: session=; Max-Age=0 ──────────────│    → instantly revoked everywhere
```

### Flow D — Stateless JWT verification (no store lookup)

```
Every request carries the whole identity in the token itself:

  Authorization: Bearer eyJhbGci.eyJzdWI..sub=123,role=admin,exp=...  .SIGNATURE

Server verification (NO database, NO session store):
  1. Split into header.payload.signature
  2. Recompute signature over header.payload using the secret/public key
  3. Does it match the signature segment?   ── no ──►  401 (tampered / wrong key)
       │ yes
  4. Is exp in the future?                  ── no ──►  401 (expired)
  5. Are iss / aud what we expect?          ── no ──►  401 (wrong issuer/audience)
       │ all good
  6. Trust the claims: userId=123, role=admin. Proceed.

This is the whole appeal: any server holding the key can verify, with zero shared state.
And this is the whole problem: there's no store to DELETE from → you can't easily revoke.
```

### Flow E — OAuth 2.0 Authorization Code (high level)

OAuth 2.0 is how "Log in with Google/GitHub" works. The app never sees the user's password — it receives an **access token** (usually a Bearer token) it can present to the API.

```
User's Browser        Your App (client)      Auth Server (Google)   Resource Server (API)
  │  click "Login with Google" ─►│                    │                    │
  │◄─ 302 to Google authorize URL │                    │                    │
  │  GET /authorize?client_id=..&redirect_uri=..&scope=.. ─────►│           │
  │  user logs in + consents ─────────────────────────────────►│           │
  │◄─ 302 back to app with ?code=AUTH_CODE ───────────────────│            │
  │  ?code=AUTH_CODE ───────────►│                    │                    │
  │                              │ POST /token  code=AUTH_CODE + client_secret
  │                              │   (back channel, server-to-server) ─────►│
  │                              │◄─ { access_token, refresh_token } ───────│
  │                              │  GET /userinfo   Authorization: Bearer access_token ─►│
  │                              │◄──────────────── 200 OK { profile } ─────────────────│
```

The **authorization code** (short-lived, single-use) is swapped for the **access token** over a secure back channel using the app's `client_secret`. The access token then rides in the `Authorization: Bearer` header on every API call — exactly like Flow B. That's the key takeaway: OAuth is a way to *obtain* a Bearer token; on the wire, using it is just a Bearer header.

---

## Exact Syntax Breakdown

### Basic auth with curl

```
curl  -u alice:s3cret  https://api.myapp.com/admin
│     │  │              │
│     │  │              └── URL (must be https so the base64 creds are encrypted in transit)
│     │  └── username:password  (curl base64-encodes and builds the header for you)
│     └── --user : send HTTP Basic auth
└── transfer URL data

# curl turns -u alice:s3cret into this exact header:
#   Authorization: Basic YWxpY2U6czNjcmV0
```

### Bearer token with curl

```
curl  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiJ9..."  https://api.myapp.com/orders
│     │  │                                                 │
│     │  │                                                 └── URL
│     │  └── the literal header value: scheme "Bearer" + one space + the token
│     └── --header : send a raw custom header verbatim
└── transfer URL data
```

### Cookie jar with curl (-c writes, -b reads)

```
curl  -c cookies.txt  -d 'user=alice&pass=s3cret'  https://myapp.com/login
│     │  │             │                            │
│     │  │             └── POST body (login form)   └── URL
│     │  └── file to SAVE received Set-Cookie into (the "cookie jar")
│     └── --cookie-jar : write cookies to this file after the response
└── ...

curl  -b cookies.txt  https://myapp.com/account
│     │  │             │
│     │  │             └── URL
│     │  └── file to READ cookies from and send as Cookie: header
│     └── --cookie : send cookies from this jar
└── ...

# Do both at once to persist across a chain of requests:
curl -c jar.txt -b jar.txt https://myapp.com/next
#    write updated cookies ─┘  └─ send existing cookies

# You can also send a cookie inline without a file:
curl -b 'session=8f3b2a' https://myapp.com/account
#    -b name=value  →  sends  Cookie: session=8f3b2a
```

### Set-Cookie attribute reference

```
Set-Cookie: name=value; Attr1; Attr2=x; ...

Attribute         Purpose                                            Example
──────────────────────────────────────────────────────────────────────────────────
HttpOnly          JS (document.cookie) CANNOT read it. XSS defense.  HttpOnly
Secure            Sent only over HTTPS, never plain HTTP.            Secure
SameSite=Strict   Never sent on cross-site requests. Max CSRF safe.  SameSite=Strict
SameSite=Lax      Sent on top-level GET navigations only (default).  SameSite=Lax
SameSite=None     Sent cross-site — REQUIRES Secure. For 3rd-party.  SameSite=None
Domain=.myapp.com Which host(s) the cookie is sent to.               Domain=myapp.com
Path=/            Which URL paths the cookie is sent to.             Path=/api
Max-Age=3600      Lifetime in seconds (preferred over Expires).      Max-Age=3600
Expires=<date>    Absolute expiry (older syntax).                    Expires=Wed, 09 Jun 2027 ...
```

---

## Example 1 — Basic

**Goal:** see all three schemes on the wire against a public test server (`httpbin.org`).

### 1a. Basic auth — watch the challenge and the header

```bash
# Hit a Basic-auth-protected endpoint with NO credentials → expect 401 + challenge
curl -v https://httpbin.org/basic-auth/alice/s3cret 2>&1 | grep -Ei 'HTTP/|www-authenticate'
```
```
< HTTP/2 401
< www-authenticate: Basic realm="Fake Realm"    ← the challenge telling you HOW to auth
```

```bash
# Now send the credentials with -u
curl -v -u alice:s3cret https://httpbin.org/basic-auth/alice/s3cret 2>&1 \
  | grep -Ei 'authorization|HTTP/'
```
```
> Authorization: Basic YWxpY2U6czNjcmV0          ← curl built this from -u (base64 of alice:s3cret)
< HTTP/2 200                                      ← accepted
```

Prove base64 is not encryption:
```bash
echo -n 'YWxpY2U6czNjcmV0' | base64 -d
# → alice:s3cret        (fully reversed, no key needed)
```

### 1b. Bearer token — send a raw Authorization header

```bash
# httpbin echoes back the headers it received, so we can confirm what went on the wire
curl -s -H "Authorization: Bearer my-demo-token-123" https://httpbin.org/headers
```
```json
{
  "headers": {
    "Authorization": "Bearer my-demo-token-123",   ← exactly what we sent
    "Host": "httpbin.org",
    "User-Agent": "curl/8.4.0"
  }
}
```

### 1c. Cookies — receive Set-Cookie, resend Cookie

```bash
# Ask httpbin to set a cookie, saving it into a jar
curl -s -c jar.txt "https://httpbin.org/cookies/set?session=8f3b2a" -o /dev/null

# Look at the jar (Netscape cookie-file format)
cat jar.txt
```
```
# Netscape HTTP Cookie File
#HttpOnly_httpbin.org  FALSE  /  FALSE  0  session  8f3b2a
```
```bash
# Now send that jar back; httpbin echoes the cookies it received
curl -s -b jar.txt https://httpbin.org/cookies
```
```json
{ "cookies": { "session": "8f3b2a" } }    ← the browser-style round-trip, done manually
```

---

## Example 2 — Production Scenario: "The cross-site cookie that silently disappears"

**The situation.** You run a SPA at `https://app.myshop.com` that calls an API at `https://api.myshop.com`. Login works locally. In production, users log in, the API returns `200` with a `Set-Cookie`, but the **very next** request to `api.myshop.com` comes back `401` — as if they never logged in. No error in the network tab except the 401. The cookie is just... not being sent.

This is the single most common cookie-auth bug in modern apps. Here is how to diagnose and fix it on the wire.

### Step 1 — Confirm the cookie is being SET but not SENT

Look at the login response in browser DevTools (Network tab → the `/login` request → Response Headers), or reproduce with curl:

```bash
curl -v -c jar.txt -X POST https://api.myshop.com/login \
  -d '{"email":"a@b.com","password":"x"}' -H 'Content-Type: application/json' 2>&1 \
  | grep -i set-cookie
```
```
< set-cookie: session=8f3b2a...; HttpOnly; Path=/; SameSite=Lax
                                                    │
                                                    └── HERE IS THE BUG
```

The cookie is set with `SameSite=Lax`. Note that `app.myshop.com` and `api.myshop.com` share the registrable domain `myshop.com`, so they are *same-site* but *cross-origin*. `Lax` allows same-site requests, so the cookie *would* be sent — **but only if the fetch is told to send credentials at all.** There are two independent failure modes; check both.

### Step 2 — Check the fetch credentials mode (the JS side)

By default, `fetch()` does **not** send cookies on cross-origin requests. The frontend must opt in:

```javascript
// BROKEN — cookie is never attached on a cross-origin call
fetch("https://api.myshop.com/orders")

// FIXED — tells the browser to include cookies
fetch("https://api.myshop.com/orders", { credentials: "include" })
```

Even with `credentials: "include"`, the browser will only *attach* and *accept* the cookie if the cookie's `SameSite` policy and the server's CORS headers both permit it.

### Step 3 — The true cross-site case (different registrable domains)

Now change the scenario: the SPA is `https://dashboard.vendor.io` and the API is `https://api.myshop.com`. These are **genuinely cross-site** (different registrable domains). Here `SameSite=Lax` **blocks the cookie entirely** on the API call. You must use `SameSite=None`, which *requires* `Secure`:

```
BROKEN cookie for a true cross-site API:
  Set-Cookie: session=...; HttpOnly; SameSite=Lax
  → browser refuses to send it on the cross-site fetch → 401

FIXED cookie:
  Set-Cookie: session=...; HttpOnly; Secure; SameSite=None
                                     │        │
                                     │        └── explicitly allow cross-site sending
                                     └── mandatory: browsers reject SameSite=None without Secure
```

And the CORS layer must explicitly allow credentials (see Topic 17). A wildcard origin is **forbidden** with credentials:

```bash
# Reproduce the preflight the browser sends before the credentialed request
curl -v -X OPTIONS https://api.myshop.com/orders \
  -H 'Origin: https://dashboard.vendor.io' \
  -H 'Access-Control-Request-Method: GET' 2>&1 | grep -i access-control
```
```
< access-control-allow-origin: https://dashboard.vendor.io   ← MUST echo exact origin, NOT *
< access-control-allow-credentials: true                     ← MUST be present for cookies
```

If you see `access-control-allow-origin: *` together with credentials, the browser **rejects the response** and the cookie is dropped. The two rules that trip everyone up:

```
With credentials (cookies), the server MUST:
  1. Set  Access-Control-Allow-Origin: <the exact origin>   (never "*")
  2. Set  Access-Control-Allow-Credentials: true
And the cookie MUST be:  Secure; SameSite=None
And the fetch MUST use:  credentials: "include"
ALL FOUR are required. Miss any one → 401 with no obvious error.
```

### Step 4 — The security angle you must not miss

While fixing the "cookie not sent" bug, engineers under pressure often "just make it work" by dropping protections. Do **not**:

```
❌ Dropping HttpOnly to "read the session in JS"  → now any XSS steals the session
❌ Dropping Secure to "test over http://"          → session leaks on any plain-HTTP hop
❌ Using SameSite=None when you didn't need to      → re-opens CSRF you had closed for free
```

The correct production cookie for a genuine cross-site credentialed API is exactly:

```
Set-Cookie: session=8f3b2a...; HttpOnly; Secure; SameSite=None; Path=/; Max-Age=86400
```

...paired with proper CORS and a CSRF token (because `SameSite=None` gives up the free CSRF protection that `Lax`/`Strict` provided).

**The lesson:** a "cookie silently not sent" bug is almost always the intersection of three independent gates — the cookie's `SameSite`/`Secure` attributes, the fetch `credentials` mode, and the server's CORS `Allow-Credentials`/`Allow-Origin`. Read all three on the wire; don't guess.

---

## Common Mistakes

### Mistake 1: Treating base64 as encryption

```
❌ "Basic auth base64-encodes the password, so it's safe to use over http://"
```
base64 is a **reversible encoding with no key**. `echo -n <blob> | base64 -d` reveals the password instantly. Over plain HTTP, Basic auth credentials are effectively plaintext to anyone on the network path.

```
✅ Basic auth is ONLY acceptable over TLS. The TLS layer — not the base64 — is what
   protects the credentials. Same rule for Bearer tokens and session cookies: the
   header/cookie is plaintext; HTTPS is the only thing hiding it.
```

### Mistake 2: Putting secrets in a JWT payload

```
❌  payload = { "sub": "123", "password": "hunter2", "ssn": "555-12-3456" }
```
The payload is **base64url-encoded, not encrypted.** Anyone holding the token — including the client, and anyone who steals it — can decode and read every claim:

```bash
echo 'eyJzdWIiOiIxMjMiLCJyb2xlIjoiYWRtaW4ifQ==' | base64 -d
# → {"sub":"123","role":"admin"}   (readable by anyone, no key needed)
```
```
✅ Put only NON-secret identity claims in a JWT: user id, role, exp, iss, aud.
   The signature makes claims TRUSTWORTHY (tamper-evident), not PRIVATE.
   Secrets belong server-side, never in a token you hand to the client.
```

### Mistake 3: Storing JWTs in localStorage

```
❌  localStorage.setItem("token", jwt)   // convenient... and XSS-lootable
```
`localStorage` is readable by **any** JavaScript on the page. One malicious npm dependency or one XSS hole and `localStorage.getItem("token")` exfiltrates the token to an attacker.

```
✅ Prefer an HttpOnly cookie the browser attaches automatically — JS literally cannot
   read it (document.cookie won't show it), so XSS can't steal it directly.
   Trade-off: cookies bring CSRF exposure, so add SameSite + a CSRF token.
   Rule of thumb: HttpOnly cookie for XSS safety > localStorage for convenience.
```

### Mistake 4: Missing Secure / HttpOnly / SameSite on the session cookie

```
❌  Set-Cookie: session=8f3b2a
      → readable by JS (XSS steals it), sent over http (network sniffing),
        sent on cross-site requests (CSRF rides it).
```
```
✅  Set-Cookie: session=8f3b2a; HttpOnly; Secure; SameSite=Lax; Path=/; Max-Age=86400
      HttpOnly  → XSS can't read it
      Secure    → never leaves over plain HTTP
      SameSite  → CSRF can't silently trigger authenticated cross-site requests
```

### Mistake 5: Confusing 401 and 403

```
❌  Returning 403 when the user sent no token (should invite them to log in)
❌  Returning 401 when a logged-in user hits an admin-only route (they ARE authenticated)
```
```
✅  401 Unauthorized  = "I don't know who you are." Missing/expired/bad credentials.
                        Include WWW-Authenticate. Retry-with-good-creds MIGHT work.
✅  403 Forbidden      = "I know who you are; you're not allowed." Valid creds, no perms.
                        Retrying with the SAME creds will never work.
```

### Mistake 6: No token expiry or rotation

```
❌  JWT minted with no exp claim (or exp years away). If it leaks, it's valid forever.
```
```
✅  Short-lived access tokens (5–15 min) + a long-lived refresh token.
    The access token's blast radius is tiny; the refresh token can be revoked in a store.
    Always set exp. Always have a rotation/refresh strategy.
```

### Mistake 7: Assuming stateless JWTs can be instantly revoked

```
❌  "Ban a user by deleting their JWT."  There's nothing to delete — the token lives
     on the CLIENT. Any server with the key will keep accepting it until exp.
```
```
✅  Accept the trade-off honestly. Options:
    - Keep exp short so leaked/stale tokens die quickly.
    - Maintain a server-side denylist of revoked token IDs (jti) — but now you have
      state again, partially defeating the "stateless" benefit.
    - Use short access tokens + revocable refresh tokens (revoke by refusing to refresh).
    If instant revocation is a hard requirement, sessions may be the better fit.
```

---

## Hands-On Proof

```bash
# 1. See a 401 challenge and the WWW-Authenticate header
curl -sv https://httpbin.org/basic-auth/u/p 2>&1 | grep -i 'www-authenticate\|< HTTP'

# 2. Prove base64 is not encryption — encode then decode a credential
echo -n 'alice:s3cret' | base64            # → YWxpY2U6czNjcmV0
echo -n 'YWxpY2U6czNjcmV0' | base64 -d     # → alice:s3cret  (no key needed!)

# 3. Watch curl build the Basic header from -u
curl -sv -u alice:s3cret https://httpbin.org/headers 2>&1 | grep -i authorization

# 4. Decode a real JWT payload (middle segment) WITHOUT the signing key
TOKEN='eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiIxMjMiLCJyb2xlIjoiYWRtaW4iLCJleHAiOjE3MzAwMDM2MDB9.sig'
echo "$TOKEN" | cut -d. -f2 | tr '_-' '/+' | base64 -d 2>/dev/null; echo
# → {"sub":"123","role":"admin","exp":1730003600}   (proves the payload is public)

# 5. Send a Bearer token and confirm what reached the server
curl -s -H "Authorization: Bearer test-123" https://httpbin.org/headers | grep -i bearer

# 6. Full cookie round-trip: receive Set-Cookie, then resend it
curl -s -c jar.txt "https://httpbin.org/cookies/set?session=abc123" -o /dev/null
cat jar.txt                                 # inspect the stored cookie
curl -s -b jar.txt https://httpbin.org/cookies   # server echoes {"cookies":{"session":"abc123"}}

# 7. Inspect Set-Cookie attributes on a real site
curl -sv https://github.com 2>&1 | grep -i 'set-cookie' | head
# Look for HttpOnly, Secure, SameSite on their session cookies
```

---

## Practice Exercises

### Exercise 1 — Easy: Decode credentials and a token by hand

```bash
# Part A — Basic auth
# This header appeared in a capture:
#   Authorization: Basic Ym9iOnBhc3N3b3JkMTIz
# 1. Decode it. What are the username and password?
echo -n 'Ym9iOnBhc3N3b3JkMTIz' | base64 -d

# Part B — JWT
# Decode the PAYLOAD of this token (the middle dot-separated segment):
#   eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiI0MiIsInJvbGUiOiJ1c2VyIiwiZXhwIjoxNzMwMDAwMDAwfQ.xxxxx
# 2. What is the "sub"? The "role"? The "exp"?
# 3. Could you have read those claims without the signing secret? What does that tell you
#    about what you should NEVER put in a JWT payload?
```

### Exercise 2 — Medium: Reproduce the cookie round-trip and read every attribute

```bash
# 1. Log into a cookie jar against httpbin and inspect the raw Set-Cookie header:
curl -sv -c jar.txt "https://httpbin.org/cookies/set?sid=xyz789" -o /dev/null 2>&1 \
  | grep -i set-cookie

# 2. Send the jar back and confirm the server sees the cookie:
curl -s -b jar.txt https://httpbin.org/cookies

# 3. Now answer:
#    - When you send with -b, does the Cookie header include HttpOnly/Secure, or just name=value? Why?
#    - Add ' -b "sid=xyz789; extra=1" ' inline (no jar). What Cookie header does the server receive?
#    - Which of HttpOnly / Secure / SameSite would you ADD to make this a safe session cookie,
#      and what attack does each one stop?
```

### Exercise 3 — Hard (Production Simulation): Diagnose a cross-site 401

```
Scenario: SPA at https://app.acme.dev calls API at https://api.other-vendor.com.
Login returns 200 with Set-Cookie, but every subsequent API call returns 401.
You captured these on the wire:

  Login response:
    Set-Cookie: session=9a8b7c; HttpOnly; Path=/; SameSite=Lax

  Frontend code:
    fetch("https://api.other-vendor.com/data")

  API preflight response:
    Access-Control-Allow-Origin: *
    (no Access-Control-Allow-Credentials header)

Tasks:
  1. List EVERY reason the cookie is not being sent / accepted. There are at least three
     independent problems above. Name each and the exact line that causes it.
  2. Write the corrected Set-Cookie header.
  3. Write the corrected fetch() call.
  4. Write the two corrected CORS response headers.
  5. After switching to SameSite=None, what protection did you just give up, and how do
     you compensate?
  6. Bonus: would switching from a cookie to an "Authorization: Bearer" token stored in
     memory sidestep this entire class of problem? What new risk does that introduce?
```

---

## Mental Model Checkpoint

Answer from memory. If you can't, re-read the relevant section.

1. **A request arrives with a valid, unexpired token, but for a resource this user doesn't own. Which status code — 401 or 403 — and why? Which one carries `WWW-Authenticate`?**

2. **Your colleague says "Basic auth is fine, the password is base64-encoded so it's encrypted." Give the one-line rebuttal and the one shell command that proves it.**

3. **You decode a JWT payload and see `{"sub":"123","role":"admin","exp":...}`. You did this without any secret key. What does that prove about (a) confidentiality of the payload and (b) what you must never store in it?**

4. **A user's laptop is stolen with an active session. In a stateful session system, how do you revoke it? In a stateless JWT system with 24h expiry and no denylist, how do you revoke it? Which is easier and why?**

5. **Name the three cookie attributes that defend against, respectively: XSS reading the cookie, the cookie leaking over plain HTTP, and CSRF. Give the exact attribute for each.**

6. **A cross-site fetch to your API returns 401 even though login succeeded. List the three independent gates (one in the cookie, one in the JS, one in CORS) that all must be satisfied for the cookie to be sent and accepted.**

7. **In the OAuth 2.0 authorization code flow, what is exchanged for the access token, and over which channel? Once you have the access token, in which HTTP header does it ride on API calls?**

---

## Quick Reference Card

### Schemes at a glance

| Scheme | Header on the wire | Encoding | Safe without TLS? | Typical use |
|--------|--------------------|----------|-------------------|-------------|
| Basic | `Authorization: Basic base64(user:pass)` | base64 (reversible!) | ❌ never | quick internal tools, legacy |
| Bearer (opaque) | `Authorization: Bearer <random-id>` | opaque; needs server lookup | ❌ never | APIs, revocable tokens |
| Bearer (JWT) | `Authorization: Bearer h.p.s` | base64url, signed | ❌ never | stateless APIs, microservices |
| Session cookie | `Cookie: sid=<opaque-id>` | opaque; server stores state | ❌ never | browser web apps |

### 401 vs 403

| | 401 Unauthorized | 403 Forbidden |
|---|------------------|---------------|
| Means | "I don't know who you are" (**unauthenticated**) | "I know you; you're not allowed" (**unauthorized**) |
| Cause | missing / bad / expired credentials | valid creds, insufficient permission |
| Header | `WWW-Authenticate: <scheme> ...` (required) | none |
| Retry with good creds? | might succeed | never — same creds always fail |

### Session vs JWT — the honest trade-off table

| Dimension | Session (stateful) | JWT (stateless) |
|-----------|--------------------|-----------------|
| Where identity lives | server-side store; cookie holds only an ID | inside the token, on the client |
| Server storage | required (Redis/DB per session) | none for verification |
| Size on the wire | tiny (`sid=8f3b2a`, ~30 bytes) | large (whole token, 300B–1KB+ every request) |
| Verification | store lookup on every request | signature check, no lookup |
| Horizontal scaling | needs shared/sticky session store | trivial — any node with the key verifies |
| Revocation / logout | instant (delete from store) | hard (valid until `exp`; needs denylist) |
| Rotating a leaked secret | n/a (no signing secret) | invalidates ALL tokens at once |
| Best when | you need instant revocation, small cookies | you need scale, cross-service, no shared state |

### Cookie attribute cheat block

```
Set-Cookie: session=<id>; HttpOnly; Secure; SameSite=Lax; Path=/; Max-Age=86400

HttpOnly          → JS can't read it (document.cookie hides it)   → blocks XSS theft
Secure            → HTTPS only, never plain HTTP                  → blocks network sniffing
SameSite=Strict   → never sent cross-site                         → strongest CSRF defense
SameSite=Lax      → sent on top-level GET nav only (default)      → good CSRF defense
SameSite=None     → sent cross-site; REQUIRES Secure              → for 3rd-party/cross-site APIs
Domain / Path     → scope: which host / which paths get the cookie
Max-Age / Expires → lifetime; Max-Age=0 deletes the cookie (logout)
```

### curl auth commands

```bash
curl -u user:pass URL                              # Basic auth (curl base64-encodes it)
curl -H "Authorization: Bearer $TOKEN" URL         # Bearer token
curl -c jar.txt URL                                # save received cookies (write jar)
curl -b jar.txt URL                                # send cookies from jar (read jar)
curl -c jar.txt -b jar.txt URL                     # persist across a request chain
curl -b 'session=abc' URL                          # send an inline cookie
echo -n 'user:pass' | base64                       # build a Basic credential by hand
echo <jwt> | cut -d. -f2 | base64 -d               # decode a JWT payload (no key needed)
```

### JWT claim vocabulary

```
sub  subject      → who the token is about (the user id)
iss  issuer       → who minted it (auth.myapp.com)
aud  audience     → who it's for (api.myapp.com)
iat  issued-at    → unix time it was created
exp  expiry       → unix time it stops being valid   ← ALWAYS set this
jti  jwt id       → unique token id (for denylists)
alg  algorithm    → HS256 (symmetric secret) vs RS256 (asymmetric key pair)
```

---

## When Would I Use This at Work?

### Scenario 1: "Should this new service use sessions or JWTs?"
You now reason it out instead of cargo-culting. Internal microservices that call each other with no browser and need to scale horizontally → **JWT (RS256)**: any service verifies with the public key, no shared session store. A classic server-rendered web app where "log out everywhere now" and account-ban must be instant → **sessions**: one `DELETE` from Redis revokes globally. A public API where you want revocable keys → **opaque Bearer tokens** in a store. The trade-off table above is the exact conversation to have in the design review.

### Scenario 2: "Our pentest flagged the session cookie."
The report says `HttpOnly`, `Secure`, or `SameSite` is missing. You know precisely what each one stops (XSS theft, plaintext leakage, CSRF) and can fix it in one line of `Set-Cookie` — and you know *not* to strip `HttpOnly` just because a frontend dev wanted to read the session in JavaScript.

### Scenario 3: "Login works, but the SPA gets 401 on every API call in production."
Instead of flailing, you check the three gates in order: the cookie's `SameSite`/`Secure` attributes, the fetch `credentials: "include"` mode, and the server's `Access-Control-Allow-Credentials` + exact-origin CORS. You read them on the wire with `curl -v` and browser DevTools and find the one that's misconfigured. (This is the single most common auth bug in modern frontends — see Topic 17.)

### Scenario 4: "A token leaked — how bad is it and how do we contain it?"
You immediately ask: is it a JWT (can't be revoked before `exp` without a denylist) or an opaque/session token (delete it server-side, done)? What's the `exp`? Is there a refresh-token rotation you can trip? Your answer shapes the incident response — and retroactively justifies why short access-token lifetimes matter.

---

## Connected Topics

**Study before this:**
- `09-tls-ssl-in-depth.md` — *the* prerequisite. Every scheme here (Basic, Bearer, cookies) is plaintext on the wire; TLS is the only thing protecting it.
- `14-http-headers-in-depth.md` — `Authorization`, `WWW-Authenticate`, `Set-Cookie`, and `Cookie` are just HTTP headers; know how headers work first.

**Study next (in order):**
- `17-cors-in-depth.md` — why credentialed cross-origin requests need exact-origin CORS + `Allow-Credentials`, and how the preflight interacts with cookies (Example 2 lives at this intersection).
- `19-cookies-and-sessions.md` — the deep dive on exactly what `Set-Cookie` puts on the wire, browser storage/resend rules, and every security implication of cookie-based sessions.

**This topic connects forward to:**
- `18-rate-limiting-at-http-level.md` — rate limits are usually keyed on the authenticated identity you established here.
- `34-api-gateways.md` — gateways commonly offload auth (validate the JWT / session once at the edge) before forwarding to your service.
- `36-service-to-service-communication.md` — internal service auth (mTLS, signed JWTs) builds directly on these ideas.

---

*Doc saved: `/docs/networking/16-authentication-over-http.md`*
