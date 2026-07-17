# 81 — Security Fundamentals in System Design
## Category: HLD Components

---

## What is this?

Security in system design is the set of controls that answer two questions on **every single request**: *"Who are you?"* and *"Are you allowed to do this?"* — plus the plumbing that keeps the answers honest (encryption, hashing, key management, input validation).

Think of a hotel. The **front desk checks your passport** — that's authentication. The **key card they give you opens room 412 and the gym, but not room 413 and not the manager's office** — that's authorization. The card is a *token*: it doesn't say your name, it just proves the desk already checked you. And if you lose it, the hotel wants a way to deactivate it — that turns out to be the hardest problem in the whole chapter.

---

## Why does it matter?

Every other topic in this curriculum makes your system **faster or bigger**. Security is the topic that decides whether your system still exists next year.

**What breaks if you don't understand this:**
- You store passwords with SHA-256 → a leaked database is fully cracked in hours on a rented GPU.
- You build string-concatenated SQL → one attacker exfiltrates every row you own.
- You issue 30-day JWTs → a stolen token gives an attacker a month of access and you have **no button to press**.
- You check "is the user logged in?" but never "is this *their* order?" → `GET /orders/123` leaks other customers' data. This is the single most common real-world API vulnerability, and it is *invisible* in tests because your test user only ever fetches their own orders.

**Interview angle:** Almost every HLD interview has a security beat. "How do users authenticate?" "Where do you validate the token?" "How do you revoke access?" Answering *"JWT"* and stopping is a junior answer. Answering *"short-lived access JWT validated at the gateway, plus a server-stored refresh token so I can actually revoke"* is a senior answer.

**Work angle:** You will be the person in code review who says "that's a mass-assignment bug" or "that key needs to be hashed at rest." Security is not a team down the hall; it's a property of the code you write today.

---

## The core idea — explained simply

### The Hotel Analogy

You walk into a hotel.

**Step 1 — The front desk (authentication).** You hand over your passport. The clerk compares the photo to your face and the name to the booking. They are establishing **identity**: *are you who you claim to be?* This happens **once**, at the start. It's slow and involves a secret (your passport, your password).

**Step 2 — The key card (authorization).** The clerk programs a card. That card opens: room 412, the gym, the pool. It does **not** open: room 413, the staff corridor, the safe in the back office. The card encodes **permissions**, not identity. Nobody at the gym door re-checks your passport — the door just asks the card *"are you allowed in here?"*

**Step 3 — The card readers (enforcement points).** Every door is a checkpoint. A door that only checks *"is this a valid card?"* and not *"is this card allowed in THIS room?"* is a broken door. That broken door is exactly the IDOR bug.

**Step 4 — Losing the card (revocation).** Two hotel designs:
- **Networked locks** (session-based auth): every door radios the front desk — "card 88F2, room 412, allowed?" If you report the card lost, the desk deletes it and the *next* door beep fails instantly. Cost: every door needs a live line to the desk.
- **Offline locks** (JWT): the card itself carries a cryptographically signed statement — *"valid for room 412 until 11am Sunday"* — and each door verifies the signature locally with no phone call. Fast, scales infinitely, works when the desk is down. Cost: **if you lose the card, the desk cannot un-print it.** It's valid until 11am Sunday, full stop. The only fix is to make the expiry short.

That trade — *instant revocation* versus *no lookup* — is the central tension of this entire topic.

| Hotel | System design |
|-------|---------------|
| Passport check at front desk | **Authentication** (authn) — verify identity, once, with a secret |
| Key card | **Credential / token / session ID** — proof that authn already happened |
| "Which doors does this card open?" | **Authorization** (authz) — checked on *every* request |
| Card reader on each door | Enforcement point (API gateway, service, DB query) |
| Networked lock phoning the desk | **Session-based auth** — server-side state, instant revoke, lookup per request |
| Offline lock verifying a signature | **JWT** — stateless, no lookup, cannot be un-issued |
| Card expiry printed on the strip | Token `exp` claim — your only defence against theft |
| Valet key that opens just the car | **OAuth** — delegate limited access without handing over your master key |

---

## Key concepts inside this topic

### 1. Authentication vs Authorization — nail this first

These get confused constantly, including by people who should know better. Two different questions, two different failure modes.

| | Authentication (authn) | Authorization (authz) |
|---|---|---|
| Question | **Who are you?** | **What may you do?** |
| Answers with | A secret you hold (password, private key, OTP) | A policy about the identity |
| When | Once per session/login | On **every** request, per resource |
| Failure | Attacker logs in as someone else | Logged-in user reads someone else's data |
| HTTP status | `401 Unauthorized` (badly named — it means *unauthenticated*) | `403 Forbidden` |

```javascript
// Express middleware. Two DIFFERENT layers — never collapse them into one.

// AUTHN: who are you? Populates req.user or rejects.
function authenticate(req, res, next) {
  const token = req.headers.authorization?.replace(/^Bearer /, '');
  if (!token) return res.status(401).json({ error: 'missing token' });
  try {
    req.user = verifyToken(token);   // throws on bad signature / expiry
    next();
  } catch {
    return res.status(401).json({ error: 'invalid token' });
  }
}

// AUTHZ: may THIS user touch THIS object? Note it needs the object.
async function authorizeOrderAccess(req, res, next) {
  const order = await orderRepo.findById(req.params.id);
  if (!order) return res.status(404).json({ error: 'not found' });
  // The whole ballgame is this one line:
  if (order.userId !== req.user.sub && req.user.role !== 'admin') {
    return res.status(403).json({ error: 'forbidden' });
  }
  req.order = order;
  next();
}

app.get('/orders/:id', authenticate, authorizeOrderAccess, (req, res) => {
  res.json(req.order);
});
```

Authn is a *gate*. Authz is a *filter that must run at every door*. A system with perfect authn and no authz is a hotel where anyone with a valid card can walk into any room.

### 2. Session-based auth vs token-based (JWT) — the central trade-off

**Session-based (stateful).** On login the server generates a long random **opaque** ID — opaque meaning it carries zero information, it's just a lookup key — and stores the real state server-side (typically Redis).

```javascript
import crypto from 'node:crypto';

async function login(req, res) {
  const user = await authenticateUser(req.body.email, req.body.password);

  // 128 bits of CSPRNG randomness. Never Math.random() — it is predictable.
  const sessionId = crypto.randomBytes(32).toString('base64url');

  // The state lives on the SERVER. The cookie is just a claim ticket.
  await redis.set(`sess:${sessionId}`, JSON.stringify({
    userId: user.id, role: user.role, createdAt: Date.now(),
  }), 'EX', 60 * 60 * 24 * 7);   // 7-day TTL, refreshed on activity

  res.cookie('sid', sessionId, {
    httpOnly: true,   // JS cannot read it → XSS cannot steal it
    secure: true,     // HTTPS only
    sameSite: 'lax',  // blocks most CSRF
    maxAge: 7 * 24 * 3600 * 1000,
  });
  res.json({ ok: true });
}

// Revocation is one line and it is INSTANT.
async function logout(req, res) {
  await redis.del(`sess:${req.cookies.sid}`);
  res.clearCookie('sid').json({ ok: true });
}
```

Every request now costs a Redis GET (~1 ms in-datacenter). And to scale horizontally you need **shared** session storage — if server A holds the session in memory and the load balancer sends the next request to server B, the user is logged out. (Sticky sessions "solve" this and then break the moment a node dies.)

**Token-based / JWT (stateless).** The server signs a token containing the claims. Any server with the key can verify it — **no lookup, no shared store**.

```javascript
import jwt from 'jsonwebtoken';

const token = jwt.sign(
  { sub: user.id, role: user.role },
  process.env.JWT_SECRET,
  { expiresIn: '15m', issuer: 'api.example.com', audience: 'example-web' }
);

// Verification is pure CPU — microseconds, no network. This is the superpower.
const claims = jwt.verify(token, process.env.JWT_SECRET, {
  algorithms: ['HS256'],          // NEVER omit this — see the alg:none attack below
  issuer: 'api.example.com',
  audience: 'example-web',
});
```

**And here is the catch that people gloss over: you cannot revoke a JWT.** Nothing is stored, so there is nothing to delete. User clicks "log out of all devices"? The token is still cryptographically valid. Fire an employee? Their token still works. A stolen token is valid **until it expires**, and the server cannot tell the difference between the real user and the thief — the signature is perfect either way.

Yes, you can bolt on a denylist ("blocklist of revoked JTIs in Redis") — but notice what you just did: you added a per-request lookup to a shared store. **You reinvented sessions, but with worse ergonomics.**

**The standard mitigation — and it's the answer interviewers want:**

- **Access token:** JWT, **5–15 minutes**, sent on every request, verified statelessly. Short life = a stolen token is a small window.
- **Refresh token:** long-lived (days/weeks), opaque random string, **stored server-side (hashed)**, sent *only* to `/auth/refresh`. Because it's stored, it **can be revoked** — deleting the row kills the session within one access-token lifetime.

```javascript
// POST /auth/refresh
async function refresh(req, res) {
  const presented = req.cookies.rt;
  const hash = sha256(presented);                 // stored hashed, like a password
  const row = await db.refreshTokens.findOne({ tokenHash: hash });

  if (!row || row.revoked || row.expiresAt < new Date()) {
    return res.status(401).json({ error: 'invalid refresh token' });
  }

  // Rotation: each refresh burns the old token and issues a new one.
  // If an OLD token is ever presented again, it was stolen and replayed →
  // nuke the whole token family. This is refresh-token reuse detection.
  if (row.usedAt) {
    await db.refreshTokens.updateMany({ familyId: row.familyId }, { revoked: true });
    return res.status(401).json({ error: 'token reuse detected — session killed' });
  }
  await db.refreshTokens.update({ id: row.id }, { usedAt: new Date() });

  const newRefresh = crypto.randomBytes(32).toString('base64url');
  await db.refreshTokens.insert({
    tokenHash: sha256(newRefresh), userId: row.userId, familyId: row.familyId,
    expiresAt: new Date(Date.now() + 30 * 864e5),
  });

  res.cookie('rt', newRefresh, { httpOnly: true, secure: true, sameSite: 'strict', path: '/auth/refresh' });
  res.json({ accessToken: signAccessToken(row.userId) });   // fresh 15-min JWT
}
```

So the honest summary: **you get statelessness on the hot path (every API call) and pay for state only on the cold path (every 15 minutes).** That's the deal. Your "revocation window" is the access-token lifetime — a design knob you turn between security (shorter) and refresh traffic (longer).

**Compare:**

| | Session (opaque ID) | JWT (self-contained) |
|---|---|---|
| Where's the state | Server (Redis/DB) | In the token itself |
| Lookup per request | Yes (~1 ms) | No (pure CPU, µs) |
| Revocation | **Instant** — delete the row | **Impossible** before `exp` |
| Horizontal scaling | Needs shared session store | Trivial — any node can verify |
| Cross-service / mobile | Cookies are awkward across domains | Bearer header works everywhere |
| Size on the wire | ~40 bytes | ~300–800 bytes, on *every* request |
| Failure mode | Redis down = everyone logged out | Leaked key = attacker forges any identity |

### 3. JWT structure — and why the payload is NOT a secret

A JWT is three **base64url** chunks joined by dots:

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9   .   eyJzdWIiOiI0MiIsInJvbGUiOiJhZG1pbiJ9   .   dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk
└──────────── HEADER ────────────┘       └──────────── PAYLOAD ────────────┘       └──────────── SIGNATURE ────────────┘
   {"alg":"HS256","typ":"JWT"}              {"sub":"42","role":"admin",              HMAC-SHA256(
                                             "exp":1735689600}                         base64url(header) + "." + base64url(payload),
                                                                                       secret)
```

**Base64url is encoding, not encryption.** Anyone holding the token — the user, a browser extension, a log aggregator, an attacker — can decode the payload in one line:

```javascript
// No key needed. This is not a hack; this is just... base64.
const [, payload] = token.split('.');
console.log(JSON.parse(Buffer.from(payload, 'base64url').toString()));
// { sub: '42', role: 'admin', exp: 1735689600 }
```

**Therefore: never put anything secret in a JWT.** No password hashes, no API keys, no PII you wouldn't print on a postcard, no internal flags you'd rather not leak. The signature guarantees **integrity** (nobody can *change* the claims without the key), not **confidentiality**.

**The `alg: none` attack.** Early JWT libraries honoured the header's `alg` field. An attacker takes a valid token, rewrites the header to `{"alg":"none"}`, rewrites the payload to `{"sub":"1","role":"admin"}`, strips the signature — and a naive verifier says *"alg is none, so no signature to check, looks good!"* Instant admin.

A sibling attack: a server that verifies with an RSA public key gets handed a token with `alg: HS256`, and the library HMACs it **using the public key as the secret** — which the attacker also has, because it's public.

**The fix, always:** pin the algorithm on the verify side and never trust the header.

```javascript
jwt.verify(token, key, { algorithms: ['RS256'] });   // pinned. non-negotiable.
```

Also always validate `exp` (libraries do this by default — don't disable it), `iss`, and `aud`, so a token minted for your *staging* environment isn't accepted by *production*.

### 4. OAuth 2.0 and OpenID Connect

**The problem OAuth actually solves.** A photo-printing app wants your Google Photos. The naive approach: it asks for your Google **password**. That is catastrophic — the app now has full access to your email, your Drive, everything, forever, and it must store your password to keep working. You cannot revoke it without changing your password everywhere.

OAuth 2.0 is **delegated authorization**: the app gets a *valet key* — an access token scoped to `photos.readonly`, expiring in an hour, revocable from your Google account page — and **never sees your password**.

Say this precisely, because interviewers listen for it:
- **OAuth 2.0 = authorization.** "This app may read my photos." It says nothing about *who you are*.
- **OpenID Connect (OIDC) = a thin identity layer on top of OAuth 2.0.** It adds an **ID token** (a JWT with `sub`, `email`, `name`) and a `/userinfo` endpoint. This is what "Sign in with Google" actually is. Using a raw OAuth *access* token as proof of identity is a classic bug — an access token proves *access was granted*, not *who granted it*.

**The four roles:**

| Role | Who | In the photo-print example |
|---|---|---|
| **Resource Owner** | The human who owns the data | You |
| **Client** | The app requesting access | PrintMyPics.com |
| **Authorization Server** | Authenticates the owner, issues tokens | Google's OAuth server (`accounts.google.com`) |
| **Resource Server** | Holds the data, accepts tokens | Google Photos API |

**Authorization Code flow with PKCE** (say "pixie"). PKCE (Proof Key for Code Exchange) exists because the authorization code travels back through the browser — a redirect — where a malicious app on the same device could intercept it. PKCE makes an intercepted code useless: the client proves it's the same party that *started* the flow. It is now recommended for **all** clients, not just mobile.

```
 Resource Owner        Client                Authorization Server     Resource Server
   (you/browser)   (PrintMyPics)              (Google accounts)       (Photos API)
        │                │                            │                      │
        │  "Connect      │                            │                      │
        │   Google" ────▶│                            │                      │
        │                │ 1. verifier = random(64)   │                      │
        │                │    challenge = SHA256(verifier)                   │
        │                │    (verifier NEVER leaves the client)             │
        │                │                            │                      │
        │◀── 2. redirect to /authorize?             ──┤                      │
        │      client_id=... &redirect_uri=...        │                      │
        │      &scope=photos.readonly &state=xyz      │                      │
        │      &code_challenge=<challenge>            │                      │
        │      &code_challenge_method=S256            │                      │
        │                │                            │                      │
        ├─── 3. login + consent screen ──────────────▶│                      │
        │    "PrintMyPics wants to view your photos"  │                      │
        │    [Allow]  [Deny]                          │                      │
        │                │                            │                      │
        │◀── 4. redirect back with ?code=AUTH_CODE&state=xyz ────────────────┤
        │                │                            │                      │
        ├─ 5. browser hands code to client ──▶│       │                      │
        │                │                            │                      │
        │                ├── 6. POST /token ─────────▶│                      │
        │                │    code=AUTH_CODE          │                      │
        │                │    code_verifier=<verifier>│  server hashes it &  │
        │                │    client_id, redirect_uri │  compares to the     │
        │                │                            │  challenge from (2)  │
        │                │                            │                      │
        │                │◀─ 7. { access_token (1h),  │                      │
        │                │       refresh_token,       │                      │
        │                │       id_token }  ← OIDC   │                      │
        │                │                            │                      │
        │                ├── 8. GET /v1/photos ──────────────────────────────▶
        │                │    Authorization: Bearer <access_token>           │
        │                │◀── 9. your photos ────────────────────────────────┤
        │                │                            │                      │
```

Two details that carry real weight:
- **`state`** is a random value the client generates and checks on return. It ties the callback to *this* browser session and blocks CSRF on the redirect.
- The **authorization code is a one-time, short-lived (~60s) coupon**, exchanged over a *back-channel* (server-to-server POST). The tokens themselves never appear in a URL, browser history, or referrer header. That's the whole reason this flow beats the old, now-discouraged Implicit flow.

### 5. API keys — useful, weak, and constantly leaked

An API key is a **bearer secret**: whoever holds the string is the caller. No signature, no expiry (usually), no proof of possession.

**When they're fine:** server-to-server calls, B2B integrations, machine clients where there is no human to log in and no browser to redirect. Stripe, Twilio, SendGrid all work this way.

**Why they're weak:** they're long-lived, they're copy-pasted, and they end up in git history, CI logs, Slack messages, Postman collections, and client-side JavaScript bundles.

**How to manage them properly:**

```javascript
// 1. PREFIX so secret scanners (GitHub, TruffleHog, gitleaks) can spot a leak.
//    Stripe pioneered this: sk_live_..., pk_test_...
const key = `sk_live_${crypto.randomBytes(24).toString('base64url')}`;

// 2. HASH AT REST. Treat it like a password — your DB should never hold the
//    plaintext. Unlike a user password this is high-entropy, so a fast hash is
//    fine here (no brute-force risk); you do NOT need bcrypt's slowness.
await db.apiKeys.insert({
  keyHash: crypto.createHash('sha256').update(key).digest('hex'),
  prefix: key.slice(0, 12),          // shown in the dashboard: "sk_live_9fA2…"
  scopes: ['orders:read'],           // 3. SCOPE — least privilege, not god-mode
  ownerId: org.id,
  expiresAt: addDays(new Date(), 90),// 4. ROTATE — expiry forces the habit
});
return key;   // shown to the user EXACTLY ONCE, then never again

// Verification: constant-time compare to avoid timing side-channels.
async function verifyApiKey(presented) {
  const hash = crypto.createHash('sha256').update(presented).digest('hex');
  const row = await db.apiKeys.findOne({ keyHash: hash });
  if (!row || row.revoked || row.expiresAt < new Date()) return null;
  await db.apiKeys.touch(row.id);   // last-used timestamp → find dead keys
  return row;
}
```

**Rotation without downtime:** support **two** live keys per account. Issue key B, deploy it, watch key A's `lastUsedAt` go stale, then revoke A. Never a big-bang swap.

### 6. Password storage

Rules, in order of how badly they'll hurt you:

1. **Never plaintext.** Not "temporarily." Not "just in the logs."
2. **Never a fast hash alone** — not MD5, not SHA-1, not SHA-256. They were *designed* to be fast. A modern GPU does billions of SHA-256 guesses per second; a rig cracks the entire common-password space essentially instantly.
3. **Use a slow, salted, memory-hard KDF:** **bcrypt**, **scrypt**, or **Argon2id** (the current preference). "Memory-hard" means the algorithm deliberately needs a big chunk of RAM, which kills the GPU/ASIC advantage — GPUs have thousands of cores but not thousands of independent memory banks.

**Salt** = a unique random value per user, stored alongside the hash. Without a salt, two users with the same password get the same hash — and an attacker precomputes a **rainbow table** (a giant hash→password lookup) *once* and cracks your whole database. With a per-user salt, the attacker must attack every user separately, and precomputation is worthless. bcrypt generates and embeds the salt for you.

**Work factor / cost** = how many times the KDF loops. Tune it so one hash takes **~100–250 ms** on your production hardware. Slow enough that offline cracking is brutally expensive; fast enough that login feels instant. Re-tune it every couple of years as hardware improves.

```javascript
import bcrypt from 'bcrypt';

const COST = 12;   // 2^12 = 4096 rounds ≈ 200-300ms on a typical 2020s server.
                   // Benchmark on YOUR hardware — +1 cost = 2x time.

export async function hashPassword(plain) {
  return bcrypt.hash(plain, COST);   // salt is generated & embedded automatically
}

// Stored value: $2b$12$Kix7Hn5f1jQ0z9pQ8sB3.eOJ9Yx...  ← alg $ cost $ salt+hash
export async function checkPassword(plain, stored) {
  // bcrypt.compare is constant-time and re-derives the salt from `stored`.
  return bcrypt.compare(plain, stored);
}

// Login — note the deliberate dummy hash on the "user not found" path.
export async function login(email, password) {
  const user = await db.users.findByEmail(email);
  if (!user) {
    // Without this, "no such user" returns in 1ms and "wrong password" in 250ms.
    // That timing difference lets an attacker ENUMERATE valid emails.
    await bcrypt.compare(password, DUMMY_HASH);
    throw new AuthError('invalid credentials');   // same message either way
  }
  if (!(await checkPassword(password, user.passwordHash))) {
    throw new AuthError('invalid credentials');
  }
  return user;
}
```

**Node-specific trap (recall topic 52 — the event loop and the libuv threadpool):** a ~250 ms hash is a *long* CPU job. The **synchronous** `bcrypt.hashSync()` blocks the event loop for 250 ms — during which your server answers *zero* other requests. The **async** version runs on the libuv threadpool (default **4** threads), so it doesn't block the loop — but it *does* mean only 4 hashes run concurrently. A login spike of 100 simultaneous logins queues up: 100 ÷ 4 × 250 ms = **6.25 seconds** for the last one. Raise `UV_THREADPOOL_SIZE` (e.g. 8–16, bounded by cores), rate-limit `/login` (topic 70), and never call the sync variants in a request path.

### 7. Encryption in transit vs at rest

**In transit — TLS.** What TLS actually buys you, precisely:
- **Confidentiality** — the coffee-shop wifi, the ISP, and every router in between see ciphertext.
- **Integrity** — nobody can flip bits or inject content undetected.
- **Server authenticity** — the certificate chain proves you're talking to `api.example.com` and not an impostor.

What TLS **does not** buy you: **anything once the bytes land on your server.** TLS terminates at the load balancer or gateway; from there the data is plaintext in memory, in your logs, and in your database. TLS is a tunnel, not a safe. (And note it does not authenticate the *client* — that's mTLS, see §9.)

Non-negotiables: TLS 1.2+ (prefer 1.3), HTTPS everywhere including internal hops, and **HSTS** (`Strict-Transport-Security`) so a browser refuses to downgrade to HTTP.

**At rest.** Three layers, protecting against different things:

| Layer | What it is | What it protects against | What it does NOT protect against |
|---|---|---|---|
| **Full-disk / volume** (LUKS, EBS encryption) | The whole volume is encrypted | Stolen disks, decommissioned hardware, a cloud provider's drive being resold | Anything at all once the machine is booted — the OS sees plaintext |
| **Database-level (TDE)** | The DB encrypts its own files | Same as above, plus stolen backups/snapshots | A compromised app with valid DB credentials — it queries and gets plaintext |
| **Column / field-level** | Your app encrypts specific fields (SSN, card, health data) before writing | Stolen disks **and** a leaked DB dump **and** a curious DBA | An attacker who compromises the app *and* can reach the key |

**Be clear-eyed:** at-rest encryption is mostly about **stolen media, stolen backups, and compliance**. It does almost nothing against a compromised application — because a live app must be able to read its own data. Anyone selling you "encrypted at rest" as protection against SQL injection is confused.

**Key management** is where at-rest encryption is actually won or lost. Keys live in a **KMS** (AWS KMS, GCP KMS, HashiCorp Vault) or an **HSM** (a tamper-resistant hardware box that will *use* a key but never *export* it). The standard pattern is **envelope encryption**: the KMS holds a master key; your app asks it to generate a **data key**, uses that data key to encrypt the row, stores the *encrypted* data key next to the ciphertext, and throws the plaintext data key away. Rotating the master key then doesn't mean re-encrypting terabytes.

```javascript
import crypto from 'node:crypto';

// Envelope encryption with AES-256-GCM. GCM is AUTHENTICATED encryption:
// the auth tag detects tampering. Plain AES-CBC does not — always use GCM.
export async function encryptField(plaintext, kms) {
  const { plaintextKey, encryptedKey } = await kms.generateDataKey('alias/pii');
  const iv = crypto.randomBytes(12);                       // NEVER reuse an IV with GCM
  const c = crypto.createCipheriv('aes-256-gcm', plaintextKey, iv);
  const ct = Buffer.concat([c.update(plaintext, 'utf8'), c.final()]);
  plaintextKey.fill(0);                                    // scrub it from memory
  return { ct, iv, tag: c.getAuthTag(), encryptedKey };    // store all four
}
```

**The golden rule: never commit secrets to git.** Git history is forever — deleting the line in a later commit changes nothing, the blob is still there, and if the repo was ever public assume the key is burned. Use environment variables injected from a secrets manager, add `gitleaks`/`trufflehog` to CI, and **rotate anything that ever touched a repo.**

### 8. The threat checklist every backend engineer must know

**SQL injection** — you built a query by gluing strings together, so the attacker's input became *code* instead of *data*.

```javascript
// VULNERABLE. email = "x' OR '1'='1' --" returns every user.
// email = "x'; DROP TABLE users; --" is not a joke.
const rows = await db.query(
  `SELECT * FROM users WHERE email = '${req.body.email}'`
);

// FIXED — parameterized query. The DB parses the SQL FIRST, then binds the
// value. The input can NEVER be interpreted as SQL, whatever it contains.
const rows = await db.query(
  'SELECT * FROM users WHERE email = $1',
  [req.body.email]
);
```
Escaping-by-hand is not a fix; **parameterize, always.** ORMs do this by default — right up until you drop into `knex.raw()` and glue strings again.

**XSS (cross-site scripting)** — attacker gets *their* JavaScript to run in *your* origin, so it can read the DOM, and read `localStorage` — including any token you stashed there. Fix: **escape on output** (React/Vue do this by default; `dangerouslySetInnerHTML` and `v-html` opt out), set a **Content-Security-Policy** so injected inline script won't execute, and put tokens in **`httpOnly` cookies** — JS literally cannot read an httpOnly cookie, so an XSS payload can't exfiltrate it. (It can still *ride* the cookie with `fetch`, so httpOnly limits theft, it doesn't cure XSS.)

```javascript
app.use(helmet.contentSecurityPolicy({
  directives: { defaultSrc: ["'self'"], scriptSrc: ["'self'"], objectSrc: ["'none'"] },
}));
```

**CSRF (cross-site request forgery)** — `evil.com` renders a form that POSTs to `bank.com/transfer`; the browser **automatically attaches your bank cookie**, and the transfer goes through. Fixes: **`SameSite=Lax|Strict`** cookies (the browser won't send the cookie on cross-site POSTs — this alone kills most CSRF), plus a **CSRF token** (a random value in the form that `evil.com` cannot read due to same-origin policy). Key insight: **`Authorization: Bearer <token>` auth is naturally immune** — browsers auto-attach *cookies*, never custom headers. `evil.com` cannot add your header.

**IDOR / broken object-level authorization** — the #1 real-world API bug. You authenticated the user and then *forgot to check ownership*.

```javascript
// VULNERABLE. Authenticated? Yes. Authorized? Never asked.
// I log in as myself and walk /orders/1, /orders/2, /orders/3... and read
// everyone's orders. Your tests pass. Your logs look normal.
app.get('/orders/:id', authenticate, async (req, res) => {
  res.json(await db.orders.findById(req.params.id));
});

// FIXED — scope the query BY the authenticated user. Don't fetch-then-check;
// make it impossible to fetch at all. Return 404 (not 403) so you don't even
// confirm the row exists.
app.get('/orders/:id', authenticate, async (req, res) => {
  const order = await db.orders.findOne({ id: req.params.id, userId: req.user.sub });
  if (!order) return res.status(404).json({ error: 'not found' });
  res.json(order);
});
```
And never rely on unguessable IDs — "security through obscurity" is not authorization. UUIDs make enumeration *slower*, not *impossible*.

**Mass assignment** — you spread untrusted JSON straight into your model.

```javascript
// VULNERABLE: POST /users {"email":"a@b.c","password":"x","role":"admin","credits":999999}
await db.users.insert({ ...req.body });   // congratulations, they're an admin

// FIXED: allowlist the fields. Never a denylist — you WILL forget a field.
const { email, password, name } = req.body;      // explicit, boring, correct
await db.users.insert({ email, name, passwordHash: await hashPassword(password),
                        role: 'user', credits: 0 });   // server sets privileged fields
```
Use a schema validator (`zod`, `joi`) with `.strict()` so unknown keys are *rejected*, not silently carried along.

**SSRF (server-side request forgery)** — you let the user hand you a URL and you fetch it. They hand you `http://169.254.169.254/latest/meta-data/iam/security-credentials/` — the cloud metadata endpoint — and your server, which *is* inside the trusted network, dutifully fetches your IAM credentials and returns them. Fixes: **allowlist** destination hosts, resolve the DNS name and reject private/link-local IP ranges (10/8, 172.16/12, 192.168/16, 127/8, 169.254/16), block redirects to those ranges (check *after* each redirect — DNS can rebind), require IMDSv2, and where possible route outbound fetches through an egress proxy that enforces the allowlist.

**Dependency / supply-chain risk** — your `node_modules` is thousands of packages you didn't write and haven't read, and they run with your app's full privileges. Discipline: `npm audit` and Dependabot in CI, **commit the lockfile**, `npm ci` (never `npm install`) in builds, pin versions, prefer fewer dependencies, and be paranoid about typosquats (`lodahs`) and brand-new packages with suspicious install scripts (`--ignore-scripts` where you can).

### 9. Defense in depth

No single control holds. Assume each one fails and ask *what's the next wall?*

- **Least privilege.** The app's DB user needs `SELECT/INSERT/UPDATE` on 6 tables — not `DROP TABLE`, not `SUPERUSER`. The service's IAM role needs `s3:GetObject` on **one** bucket prefix, not `s3:*`. Then a full app compromise is *bounded*.
- **Zero trust — don't trust the network.** The old model was a hard shell and a soft, chewy centre: get past the firewall and everything inside talks freely. One compromised node then owns the datacenter. Zero trust says **the network grants nothing**; every service authenticates every caller. In practice that's **mTLS** (mutual TLS — *both* sides present certificates, so the caller is cryptographically identified) or signed service tokens, usually handled by a service mesh so app code doesn't have to.
- **Network segmentation.** The database sits in a private subnet with no route to the internet and a security group that accepts port 5432 from the app tier *only*. Even with valid credentials, an outside attacker can't reach it.
- **Never trust client input.** Not the request body, not headers, not the JWT payload before you've verified the signature, not a hidden form field, not the price the client sent you. Client-side validation is a UX feature. **Server-side validation is the security control.**
- **Validate at every boundary.** HTTP edge, service-to-service call, queue consumer, DB write. Yes it's redundant — that's the entire point. Redundancy is what "depth" means.

### 10. Auth at scale — where do you validate the token?

At 50 services, you do **not** want 50 copies of token-parsing logic, 50 chances to forget `algorithms: [...]`, and 50 teams rolling their own.

The standard pattern: **validate once at the API gateway** (topic 58). The gateway verifies the signature, expiry, issuer and audience, rejects bad tokens at the edge (your services never see them), and then **injects a trusted internal identity** — e.g. `X-User-Id`, `X-User-Roles`, or a short-lived internal token — into the request it forwards.

```javascript
// AT THE GATEWAY: the only place that knows about the public JWT.
async function gatewayAuth(req, res, next) {
  const claims = await verifyJwtWithJwks(bearer(req));   // rotate keys via JWKS
  req.headers['x-user-id'] = claims.sub;
  req.headers['x-user-roles'] = claims.roles.join(',');
  delete req.headers['authorization'];     // don't leak the raw token downstream
  next();
}

// IN A DOWNSTREAM SERVICE: trusts the header — but ONLY because the network
// guarantees the request came from the gateway (mTLS / private subnet).
// If a client can reach this service directly, they just forge x-user-id: 1.
// THAT is the trade-off, stated plainly.
app.get('/internal/orders/:id', async (req, res) => {
  const userId = req.headers['x-user-id'];
  const order = await db.orders.findOne({ id: req.params.id, userId });
  res.json(order ?? null);
});
```

**The trade-off of centralizing:**

| Centralized at gateway | Validated in every service |
|---|---|
| One implementation, one place to fix a CVE | N implementations, N chances to get it wrong |
| No JWT parsing cost in each service | Every service pays the crypto cost (small, but real) |
| Rotate signing keys in one place | Coordinate a key rollout across N teams |
| **Gateway is a single point of failure** and a fat blast radius | Services stay independently secure |
| **Downstream services trust a header** — you MUST prevent direct access | No implicit trust; works even on an open network |

And note the division of labour: the gateway does **authn** (is this token real?) and *coarse* authz (does this token have the `orders` scope?). **Fine-grained, object-level authz — "is order 123 *yours*?" — can only happen in the service that owns the data.** The gateway has no idea who owns order 123. Never assume the gateway did that check for you; that assumption *is* the IDOR bug, at architecture scale.

---

## Visual / Diagram description

### Diagram 1: Where security lives in a request path

```
                                    ┌───────────────────────────────┐
   ┌──────────┐   HTTPS / TLS 1.3   │      API GATEWAY (topic 58)   │
   │ Browser  │════════════════════▶│                               │
   │ / Mobile │  Authorization:     │  1. Terminate TLS             │
   │          │  Bearer <JWT 15m>   │  2. Rate limit (topic 70)     │
   └──────────┘                     │  3. Verify JWT signature+exp  │◀── JWKS (public keys)
        ▲                           │     algorithms: ['RS256']     │
        │                           │  4. Coarse scope check        │
        │ refresh (every 15m)       │  5. Inject x-user-id header   │
        │                           │  6. Strip Authorization       │
   ┌────┴──────────┐                └───────────────┬───────────────┘
   │ /auth/refresh │                                │  mTLS — services
   │  (stateful!)  │                     ┌──────────┴──────────┐  authenticate
   └────┬──────────┘                     │                     │  each other
        │                       ┌────────▼────────┐   ┌────────▼────────┐
   ┌────▼───────────┐           │  Order Service  │   │  User Service   │
   │ refresh_tokens │           │                 │   │                 │
   │ (hashed, in DB)│           │ OBJECT-LEVEL    │   │  bcrypt(cost 12)│
   │  → REVOCABLE   │           │ AUTHZ HAPPENS   │   │  on /login      │
   └────────────────┘           │ HERE, not at    │   └────────┬────────┘
                                │ the gateway     │            │
                                └────────┬────────┘            │
                                         │  parameterized      │
                          ═══════════════▼═════════════════════▼══════════════
                          ║   PRIVATE SUBNET — no internet route             ║
                          ║   ┌──────────────────┐   ┌───────────────────┐   ║
                          ║   │  PostgreSQL      │   │  Redis (sessions/ │   ║
                          ║   │  TDE at rest     │   │  refresh, denylist)│  ║
                          ║   │  least-priv user │   └───────────────────┘   ║
                          ║   └──────────────────┘                           ║
                          ║   PII columns: AES-256-GCM, keys in KMS          ║
                          ═══════════════════════════════════════════════════
```

Read it as layers. TLS protects the wire. The gateway is the one place that understands the public token. Object-level authorization happens **inside the service that owns the object** — the gateway cannot do it. The database sits behind a network boundary *and* a least-privilege user *and* at-rest encryption, so each failed layer still leaves another.

### Diagram 2: The revocation trade-off, drawn

```
  SESSION (stateful)                       JWT (stateless)
  ─────────────────────                    ─────────────────────
  GET /orders                              GET /orders
  Cookie: sid=aX9f...                      Authorization: Bearer eyJ...
        │                                        │
        ▼                                        ▼
  ┌───────────┐                            ┌───────────┐
  │  Server   │                            │  Server   │
  └─────┬─────┘                            └─────┬─────┘
        │ GET sess:aX9f  (~1ms, network)         │ verify signature
        ▼                                        │ (~50µs, pure CPU)
  ┌───────────┐                                  │ NO NETWORK CALL
  │   Redis   │  ← shared state:                 ▼
  │ sess:aX9f │    every node must reach it   ┌──────┐
  └───────────┘                               │ done │
        │                                     └──────┘
        ▼
   LOGOUT / BAN?
   DEL sess:aX9f  ──▶  ✔ effective on the VERY NEXT request
                                             LOGOUT / BAN?
                                             ✘ nothing to delete.
                                             Token stays valid until `exp`.
                                             ──▶ so make exp SHORT (15 min)
                                                 and revoke the REFRESH token,
                                                 which IS stored server-side.
```

---

## Real world examples

### Stripe — API keys done properly

Stripe's keys are the reference implementation of the pattern in §5. They're **prefixed** by kind and environment (`sk_live_`, `sk_test_`, `pk_live_`, plus `rk_` restricted keys), which means GitHub's secret scanner can recognise a leaked Stripe key on sight and notify Stripe, who can proactively revoke it. Keys are shown **once** at creation and only ever displayed as a truncated prefix afterwards, which tells you they are not retrievable in plaintext from Stripe's side. Restricted keys let you scope a key to specific resources and read/write levels rather than handing an integration full account power — least privilege, applied to a credential.

### Google — OAuth 2.0 and OIDC as a product

Every "Sign in with Google" button is the Authorization Code + PKCE flow from §4 terminating in an **OIDC ID token**. Google's authorization server holds the passwords and the second factors; the third-party app never sees them and gets only the scopes you consented to on the screen. Crucially, `myaccount.google.com` lists every app you've granted access to and lets you revoke any of them individually — which is precisely the capability you *lose* if you hand out unrevokable long-lived credentials. That revocation page is the product manifestation of "the refresh token is stored server-side."

### GitHub — supply chain and secret scanning at platform scale

GitHub treats the two threats from §8 as first-class platform features: **Dependabot** watches your lockfile against the advisory database and opens PRs for vulnerable transitive dependencies, and **secret scanning** greps pushes for the known prefixes of dozens of providers, notifying the provider so the credential can be revoked before an attacker finds it. Both exist because the industry learned that (a) the average Node project pulls in thousands of packages nobody audited, and (b) engineers commit secrets constantly, and history is forever.

---

## Trade-offs

| Decision | You gain | You give up |
|---|---|---|
| **Sessions (opaque + Redis)** | Instant revocation; tiny cookie; easy to reason about | A lookup on every request; a shared store you must scale and keep alive |
| **JWT access tokens** | Stateless verification; trivial horizontal scaling; cross-domain and mobile friendly | Revocation before `exp`; bigger payload on every request; a leaked signing key forges *any* identity |
| **Short access + stored refresh** | Both, mostly: fast hot path, revocable sessions | Complexity — rotation, reuse detection, and a revocation window equal to the access-token TTL |
| **Higher bcrypt cost** | Offline cracking gets exponentially more expensive | Login latency and threadpool pressure; a cheap login-flood DoS vector if unrated-limited |
| **Validate JWT at gateway only** | One implementation; no per-service crypto; central key rotation | Downstream services now trust a header — direct network access to them becomes catastrophic |
| **Column-level encryption** | A stolen DB dump is useless; a curious DBA sees ciphertext | You lose indexing, sorting, and `LIKE` on those columns; key management becomes your problem |
| **Zero trust / mTLS everywhere** | One compromised node doesn't own the fleet | Certificate lifecycle, rotation, and a service mesh to operate; real latency and ops cost |
| **Fine-grained scopes on keys** | A leaked key is bounded | More config surface, more support tickets ("why is this 403?") |

**Rule of thumb:**
- **Browser-only, single app, one backend?** Use **sessions** with `httpOnly; Secure; SameSite` cookies. It's simpler and revocation is free. JWT is not the default just because it's fashionable.
- **Multiple services, mobile clients, third parties?** **Short-lived JWT access token (15 min) + server-stored, rotating refresh token.** Validate at the gateway. Do object-level authz in the owning service, always.
- **Server-to-server / B2B?** **Hashed, prefixed, scoped, rotating API keys** — or mTLS if both sides are yours.
- And regardless of any of the above: **parameterize your queries, hash with bcrypt/Argon2, and check ownership on every object read.** Those three lines prevent most of what actually happens.

---

## Common interview questions on this topic

### Q1: "Session vs JWT — which would you pick, and why?"
**Hint:** Don't pick one and stop. State the axis: JWT trades **revocability** for **statelessness**. Then decide from requirements — single web app, one backend, needs instant "log out everywhere"? Sessions in Redis. Many services, mobile clients, or a public API? Short-lived JWT (15 min) + server-stored, rotating refresh token, so you keep the stateless hot path *and* regain revocation. Name the revocation window explicitly: "an attacker with a stolen access token has at most 15 minutes."

### Q2: "Can I store user roles and email in the JWT payload? What about their phone number?"
**Hint:** Roles and email, yes — but understand two consequences. (1) The payload is **base64url, not encrypted** — anyone holding the token reads it, so no secrets and no PII you wouldn't put on a postcard. A phone number is a judgement call; many teams say no. (2) Claims are a **snapshot** at issue time. Demote an admin and their existing token still says `role: admin` until it expires — another argument for short TTLs.

### Q3: "Walk me through the OAuth Authorization Code flow with PKCE. Why does PKCE exist?"
**Hint:** Name the four roles (resource owner, client, authorization server, resource server). Client generates a random `code_verifier`, sends `SHA256(verifier)` as `code_challenge` on the `/authorize` redirect → user authenticates and consents at the auth server → auth server redirects back with a short-lived, one-time `code` → client POSTs `code` + the raw `verifier` to `/token` over the back channel → auth server hashes the verifier, compares to the challenge, returns tokens. **PKCE exists because the code travels through the browser and can be intercepted** — without the verifier, a stolen code is useless. Also mention `state` for CSRF on the redirect.

### Q4: "Our DB leaked and passwords were stored as salted SHA-256. How bad is it?"
**Hint:** Bad. The salt defeats **precomputed rainbow tables**, so that part is fine — but salt does nothing about **speed**, and SHA-256 was designed to be fast: a GPU rig does billions of guesses per second, so weak and medium passwords fall quickly. Correct answer: force a global password reset, and migrate to bcrypt/Argon2id with a work factor tuned to ~100–250 ms. Bonus: you can migrate *without* the plaintext by wrapping the existing hash — store `bcrypt(sha256(password))` and upgrade on next login.

### Q5: "You ship `GET /api/orders/:id`. What's the security bug you should assume is there?"
**Hint:** **IDOR** — you authenticated the caller but never checked that order `:id` belongs to them, so the endpoint happily serves any user's order to anyone logged in. The fix isn't a scarier ID; it's scoping the query: `findOne({ id, userId: req.user.sub })`, returning **404** rather than 403 so you don't confirm existence. Add that **the gateway cannot do this check for you** — it doesn't know who owns order 123 — so object-level authz must live in the owning service.

### Q6: "In a 40-service system, where do you validate the token?"
**Hint:** Once, at the **API gateway** (topic 58): verify signature/`exp`/`iss`/`aud` with pinned algorithms, reject at the edge, inject a trusted `x-user-id`. One implementation, one key rotation. Then immediately name the cost: downstream services now **trust a header**, so if a client can reach them directly they forge `x-user-id: 1`. That trust is only safe behind mTLS or a private network — and fine-grained authz still belongs in the owning service.

---

## Practice exercise

### Audit and fix a broken API

Below is a real-shaped Express endpoint set. It contains **at least six** distinct vulnerabilities from this doc. Spend 20–40 minutes: find each, name it, and rewrite the file.

```javascript
import express from 'express';
import crypto from 'node:crypto';
import jwt from 'jsonwebtoken';

const app = express();
app.use(express.json());
const SECRET = 'supersecret123';

app.post('/signup', async (req, res) => {
  const hash = crypto.createHash('md5').update(req.body.password).digest('hex');
  const user = await db.users.insert({ ...req.body, passwordHash: hash });
  res.json(user);
});

app.post('/login', async (req, res) => {
  const rows = await db.query(
    `SELECT * FROM users WHERE email = '${req.body.email}'`
  );
  const user = rows[0];
  const hash = crypto.createHash('md5').update(req.body.password).digest('hex');
  if (!user) return res.status(404).json({ error: 'no such user' });
  if (user.passwordHash !== hash) return res.status(401).json({ error: 'wrong password' });

  const token = jwt.sign(
    { sub: user.id, email: user.email, ssn: user.ssn, role: user.role },
    SECRET,
    { expiresIn: '30d' }
  );
  res.json({ token });
});

app.get('/invoices/:id', async (req, res) => {
  const claims = jwt.decode(req.headers.authorization.split(' ')[1]);
  if (!claims) return res.status(401).end();
  res.json(await db.invoices.findById(req.params.id));
});
```

**Produce:**
1. A numbered list: vulnerability name → the exact line → what an attacker does with it → the fix. (Look for: SQLi, weak hashing, mass assignment, secrets in the payload, an unverified token, a hardcoded secret, a 30-day non-revocable token, user enumeration via differing error messages.)
2. A rewritten version with all of them fixed.
3. A short paragraph: you now need "log out of all devices." Sketch what you add — and state your revocation window in minutes.

---

## Quick reference cheat sheet

- **Authentication = who are you** (once, with a secret). **Authorization = what may you do** (every request, per object). `401` = unauthenticated, `403` = forbidden.
- **Sessions** = opaque random ID in a cookie, state in Redis → **instant revocation**, but a lookup per request and a shared store to scale.
- **JWT** = signed, self-contained → **no lookup, scales statelessly**, but you **cannot revoke it before `exp`**.
- **The standard answer:** 5–15 min access JWT + long-lived **refresh token stored server-side (hashed) and rotated** → stateless hot path, revocable sessions.
- **A JWT payload is base64url — encoded, NOT encrypted.** Anyone can read it. Never put secrets or sensitive PII in it.
- **Always pin the algorithm** on verify (`algorithms: ['RS256']`). Otherwise: `alg: none`, and the public-key-as-HMAC-secret confusion.
- **OAuth 2.0 = delegated authorization** (App X reads your Google data without your password). **OIDC = the identity layer on top** — that's "Sign in with Google."
- **Auth Code + PKCE:** verifier → SHA256 challenge → one-time code via redirect → code + verifier exchanged on the back channel for tokens. `state` blocks redirect CSRF.
- **API keys:** fine for server-to-server. **Hash at rest, prefix (`sk_live_`) so scanners catch leaks, scope them, rotate them,** support two live keys for zero-downtime rotation.
- **Passwords:** never plaintext, never MD5/SHA-256 alone. **bcrypt / scrypt / Argon2id**, salted, work factor tuned to **~100–250 ms**. In Node, use the async API and mind the 4-thread libuv pool (topic 52).
- **TLS** gives confidentiality + integrity + server authenticity — and **nothing once data is on your server**. At-rest encryption mainly defeats **stolen disks and backups**, not a compromised app.
- **Never commit secrets to git** — history is forever. Secrets manager + KMS + rotation; scan CI with gitleaks.
- **The checklist:** SQLi → **parameterize**. XSS → **escape output + CSP + httpOnly cookies**. CSRF → **SameSite + CSRF token** (bearer-header auth is naturally immune). **IDOR → scope the query by `req.user.sub`** (the #1 real bug). Mass assignment → **allowlist fields**. SSRF → **allowlist hosts, block private IPs**. Dependencies → **lockfile + `npm ci` + Dependabot**.
- **Defense in depth:** least privilege, zero trust (**mTLS** between services), network segmentation, validate at **every** boundary. Client-side validation is UX; server-side validation is security.
- **At scale:** validate the JWT **once at the API gateway** and inject a trusted internal identity — but downstream services now trust a header, so they must be unreachable directly. **Object-level authz can never be centralized** — only the service that owns the object knows who owns it.

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [70 — Rate Limiting](./70-rate-limiting.md) — the other edge control; throttling `/login` is what stops credential stuffing and bcrypt-based DoS |
| **Next** | [58 — API Gateway](./58-api-gateway.md) — the single place where token validation, TLS termination and coarse authz actually live |
| **Related** | [69 — API Design: REST, GraphQL, gRPC](./69-api-design-rest-graphql-grpc.md) — where IDOR, mass assignment and over-fetching are introduced in the first place |
| **Related** | [10 — Networking Fundamentals](./10-networking-fundamentals.md) — TLS, certificates and the handshake that "encryption in transit" actually rests on |
| **Related** | [107 — HLD: Payment System](./107-hld-payment-system.md) — where every control in this doc becomes non-negotiable: PCI scope, tokenization, idempotency and audit trails |
