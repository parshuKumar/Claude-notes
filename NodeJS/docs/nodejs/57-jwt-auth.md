# 57 — Authentication patterns — JWT

## What is this?

A JSON Web Token (JWT) is a compact, signed string that proves "this user already logged in and here is who they are" — without the server needing to look anything up in a database on every request. It's like a wristband at a concert: security checks your ID once at the gate, stamps a tamper-proof wristband on your wrist, and every guard inside just glances at the wristband instead of re-checking your ID card each time. The `jsonwebtoken` npm package is the standard tool for creating (signing) and checking (verifying) these wristbands in Node.js.

## Why does it matter for backend development?

Almost every backend API needs to know "who is making this request" — a user shouldn't be able to see someone else's orders or delete another person's account. JWT-based authentication lets you build **stateless** APIs: the server doesn't store session data in memory, so any server instance behind a load balancer can verify the same token, which is exactly what horizontally scaled backends need. Every backend developer building REST APIs, mobile app backends, or microservices needs to sign tokens at login, verify them on protected routes, and handle the tricky parts — token expiry, refresh tokens, and where to safely store the token on the client — correctly, because getting this wrong is one of the most common real-world security breaches.

---

## Syntax / API

```js
// npm install jsonwebtoken
const jwt = require('jsonwebtoken');

// ── Signing a token (creating it, usually at login) ─────────────────────────
const authToken = jwt.sign(
  { userId: 'user_42', role: 'customer' },   // payload — data embedded in the token
  process.env.JWT_SECRET,                    // secret key — only the server knows this
  { expiresIn: '15m' }                       // options — token expires in 15 minutes
);
// → "eyJhbGciOiJIUzI1NiIs...header.payload.signature" (3 dot-separated parts)

// ── Verifying a token (checking it, on every protected request) ────────────
try {
  const decodedPayload = jwt.verify(authToken, process.env.JWT_SECRET);
  // decodedPayload → { userId: 'user_42', role: 'customer', iat: ..., exp: ... }
  console.log(decodedPayload.userId);   // 'user_42' — safe to trust, signature checked
} catch (err) {
  // err.name === 'TokenExpiredError' or 'JsonWebTokenError' (invalid/tampered)
  console.log('Invalid or expired token:', err.message);
}

// ── Decoding WITHOUT verifying (never trust this for auth decisions!) ──────
const unsafePayload = jwt.decode(authToken);
// Reads the payload but does NOT check the signature — only for debugging/inspection
```

---

## How it works — line by line

`jwt.sign(payload, secret, options)` takes your data, converts it to JSON, and produces three base64url-encoded pieces joined by dots: a **header** (says "I'm HMAC-SHA256"), the **payload** (your data plus `iat`/`exp` timestamps), and a **signature**. The signature is created by hashing the header and payload together with your secret key. Anyone can read the header and payload (base64 is not encryption, just encoding), but nobody can create a valid signature without knowing the secret — that's what makes the token tamper-proof, not private.

`jwt.verify(token, secret)` re-computes that same signature using the secret and compares it against the signature attached to the token. If they match, the token wasn't altered and really was issued by your server, so `verify()` returns the decoded payload. If they don't match, or the `exp` timestamp is in the past, `verify()` throws an error instead of returning data — this is why you always wrap it in `try/catch`.

`jwt.decode()` just base64-decodes the payload without touching the signature at all — useful for logging or reading a token you already trust, but **never** for making an authentication decision, since anyone could hand you a decoded payload with `role: 'admin'` in it and it would "work" without ever being cryptographically checked.

---

## Example 1 — basic

```js
// File: src/auth/token-basics.js
// Demonstrates signing and verifying a JWT end to end.

const jwt = require('jsonwebtoken');

// In real apps this comes from process.env.JWT_SECRET — hardcoded here only to show shape
const jwtSecret = 'dev-only-secret-change-me';

// Data we want to embed in the token — keep this small, it travels with every request
const tokenPayload = { userId: 'user_42', email: 'vakeelsaab791213@gmail.com' };

// Sign the payload — creates the token string, expires in 1 hour
const authToken = jwt.sign(tokenPayload, jwtSecret, { expiresIn: '1h' });
console.log('Issued token:', authToken);

// Verify the token — throws if invalid/expired, returns payload if valid
function verifyToken(token) {
  try {
    const decoded = jwt.verify(token, jwtSecret);   // checks signature + expiry
    return { valid: true, data: decoded };
  } catch (err) {
    return { valid: false, reason: err.message };   // e.g. 'jwt expired'
  }
}

const result = verifyToken(authToken);
console.log('Verification result:', result);
// → { valid: true, data: { userId: 'user_42', email: '...', iat: ..., exp: ... } }

// Tampering test — flip one character in the token and verify again
const tamperedToken = authToken.slice(0, -1) + 'X';
console.log('Tampered check:', verifyToken(tamperedToken));
// → { valid: false, reason: 'invalid signature' }
```

---

## Example 2 — real world backend use case

```js
// File: src/middleware/auth.js
// Reusable Express middleware + login route pattern with access + refresh tokens.
// This is the shape used in production Node APIs.

const jwt = require('jsonwebtoken');

// Two separate secrets — never reuse the access secret for refresh tokens
const ACCESS_SECRET  = process.env.JWT_ACCESS_SECRET;
const REFRESH_SECRET = process.env.JWT_REFRESH_SECRET;

// Short-lived access token — sent with every API request, stolen ones expire fast
function signAccessToken(user) {
  return jwt.sign(
    { userId: user.id, role: user.role },     // minimal payload — no passwords, no PII
    ACCESS_SECRET,
    { expiresIn: '15m' }                      // short window limits damage if leaked
  );
}

// Long-lived refresh token — only used to mint new access tokens, kept server-side too
function signRefreshToken(user) {
  return jwt.sign(
    { userId: user.id, tokenVersion: user.tokenVersion },   // tokenVersion lets us revoke
    REFRESH_SECRET,
    { expiresIn: '7d' }
  );
}

// Login route — issues both tokens after checking credentials (hashing covered in Topic 23)
async function loginHandler(requestBody, dbConnection) {
  const { email, password } = requestBody;
  const user = await dbConnection.users.findByEmail(email);   // look up user record
  const passwordOk = user && (await user.comparePassword(password));

  if (!passwordOk) {
    throw new Error('Invalid email or password');             // never reveal which one is wrong
  }

  const accessToken  = signAccessToken(user);
  const refreshToken = signRefreshToken(user);

  return { accessToken, refreshToken, userId: user.id };
}

// Auth middleware — protects any route placed after it in the chain
function requireAuth(req, res, next) {
  const authHeader = req.headers.authorization;             // expects "Bearer <token>"

  if (!authHeader || !authHeader.startsWith('Bearer ')) {
    return res.status(401).json({ error: 'Missing auth token' });
  }

  const authToken = authHeader.split(' ')[1];                // pull the token out of "Bearer xyz"

  try {
    const payload = jwt.verify(authToken, ACCESS_SECRET);    // throws if invalid/expired
    req.user = { userId: payload.userId, role: payload.role }; // attach identity to request
    next();                                                    // pass control to the route handler
  } catch (err) {
    if (err.name === 'TokenExpiredError') {
      return res.status(401).json({ error: 'Token expired', code: 'TOKEN_EXPIRED' });
    }
    return res.status(401).json({ error: 'Invalid token' });
  }
}

// Refresh route — client calls this when the access token expires, using the refresh token
function refreshHandler(requestBody) {
  const { refreshToken } = requestBody;
  const payload = jwt.verify(refreshToken, REFRESH_SECRET);   // throws if expired/invalid
  // In production, also check payload.tokenVersion against the DB to support instant revocation
  const newAccessToken = jwt.sign(
    { userId: payload.userId },
    ACCESS_SECRET,
    { expiresIn: '15m' }
  );
  return { accessToken: newAccessToken };
}

module.exports = { requireAuth, loginHandler, refreshHandler, signAccessToken, signRefreshToken };

// Usage in server.js:
// app.post('/login', async (req, res) => { const tokens = await loginHandler(req.body, db); res.json(tokens); });
// app.get('/profile', requireAuth, (req, res) => res.json({ userId: req.user.userId }));
```

---

## Common mistakes

### Mistake 1 — Hardcoding the JWT secret or committing it to git

```js
// ❌ WRONG — secret is visible in source code and in git history forever
const authToken = jwt.sign({ userId }, 'my-super-secret-123', { expiresIn: '1h' });
// Anyone with repo access (or a leaked commit) can now forge valid tokens

// ✅ CORRECT — secret loaded from environment variables, never committed
// .env file (gitignored): JWT_SECRET=<long random string, 32+ bytes>
require('dotenv').config();
const authToken = jwt.sign({ userId }, process.env.JWT_SECRET, { expiresIn: '1h' });
// Secrets management covered fully in Topic 79
```

### Mistake 2 — Storing JWTs in localStorage on the frontend

```js
// ❌ WRONG — localStorage is readable by ANY JavaScript running on the page,
// so a single XSS vulnerability lets an attacker steal every user's token
localStorage.setItem('authToken', accessToken);

// ✅ CORRECT — store the token in an httpOnly, Secure, SameSite cookie instead
// (set from the backend response, JavaScript on the page can never read it)
res.cookie('authToken', accessToken, {
  httpOnly: true,     // JS on the page cannot access this cookie — blocks XSS token theft
  secure: true,       // only sent over HTTPS
  sameSite: 'strict', // not sent on cross-site requests — blocks CSRF-style abuse
  maxAge: 15 * 60 * 1000,  // matches the token's own expiresIn
});
```

### Mistake 3 — Never expiring tokens, or using one token forever

```js
// ❌ WRONG — no expiresIn means the token is valid forever;
// a leaked token can be used by an attacker indefinitely
const authToken = jwt.sign({ userId }, process.env.JWT_SECRET);

// ✅ CORRECT — short-lived access token + separate refresh token flow
const accessToken  = jwt.sign({ userId }, process.env.JWT_ACCESS_SECRET, { expiresIn: '15m' });
const refreshToken = jwt.sign({ userId }, process.env.JWT_REFRESH_SECRET, { expiresIn: '7d' });
// A stolen access token becomes useless within 15 minutes;
// the refresh token is stored more carefully and can be revoked server-side
```

---

## Practice exercises

### Exercise 1 — easy

Write a small script that:
1. Signs a JWT containing `{ userId: 'user_7', role: 'admin' }` with a secret string of your choice, expiring in 30 seconds
2. Immediately verifies it and logs the decoded payload
3. Uses `setTimeout` to wait 35 seconds, then tries verifying the same token again
4. Catches the error and logs whether it was `TokenExpiredError` or something else

```js
// Write your code here
```

---

### Exercise 2 — medium

Build an Express-free auth module with two functions:
1. `signUserToken(user)` — takes a `{ id, role }` object and returns a signed access token (`expiresIn: '10m'`) using a secret from `process.env.JWT_SECRET`
2. `verifyUserToken(authToken)` — verifies the token and returns `{ ok: true, payload }` on success or `{ ok: false, error }` on failure, distinguishing between an expired token and any other invalid token in the `error` message

Then write a small test harness (no framework needed) that:
- Signs a token for `{ id: 'user_10', role: 'editor' }`
- Verifies it and prints the result
- Verifies a deliberately corrupted copy of the token and prints that result too

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a complete in-memory auth system (no real database, use a plain object as your "users table") that supports:
1. `register(email, passwordPlainText)` — stores the user with a hashed password (use Node's built-in `crypto` module's `scryptSync`, not plain text) and a `tokenVersion` starting at `0`
2. `login(email, passwordPlainText)` — checks the password, and if correct, issues an access token (15 min) and a refresh token (7 days) that both embed `tokenVersion`
3. `refresh(refreshToken)` — verifies the refresh token, checks that its `tokenVersion` matches the current stored `tokenVersion` for that user (reject if it doesn't — this is how you support logout), and issues a new access token
4. `logoutAll(userId)` — increments that user's `tokenVersion`, which instantly invalidates every previously issued refresh token for that user, even ones you never see again
5. A `requireAuth`-style function that verifies an access token and returns the payload, or throws

Test the full flow: register a user, log in, refresh the access token successfully, call `logoutAll`, then attempt to refresh again with the old refresh token and confirm it is rejected.

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
INSTALL
  npm install jsonwebtoken

SIGNING (create a token)
  jwt.sign(payload, secret, { expiresIn: '15m' })
  payload    → plain object, keep it small (userId, role — never passwords)
  secret     → long random string from process.env, never hardcoded
  expiresIn  → '15m', '1h', '7d' — always set one

VERIFYING (check a token)
  jwt.verify(token, secret)         → returns payload or THROWS
  jwt.decode(token)                 → reads payload, NO signature check — never trust for auth

TOKEN SHAPE
  header.payload.signature   (3 base64url parts, dot-separated)
  Base64 = encoding, NOT encryption — never put secrets/PII in the payload

ERROR NAMES FROM jwt.verify()
  TokenExpiredError    → exp timestamp has passed
  JsonWebTokenError    → bad signature / malformed token
  NotBeforeError       → used before its "nbf" (not-before) time

ACCESS vs REFRESH TOKENS
  Access token   → short-lived (10–15m), sent on every request, in Authorization header
  Refresh token  → long-lived (days), used ONLY to mint new access tokens
  Use DIFFERENT secrets for access vs refresh tokens

WHERE TO STORE TOKENS (client side)
  httpOnly cookie   → BEST — JS can't read it, blocks XSS token theft
  localStorage      → AVOID — readable by any script, vulnerable to XSS
  Memory (JS var)   → OK for access token in SPAs, lost on page refresh

MIDDLEWARE PATTERN
  Authorization: Bearer <token>  → split header, verify, attach req.user, call next()

REVOCATION (JWTs can't be "deleted" — they're stateless)
  tokenVersion field + DB check   → bump version to invalidate all refresh tokens
  Short expiresIn on access token → natural, automatic revocation window

NEVER DO
  Sign with no expiresIn
  Store JWT secret in code or git
  Trust jwt.decode() output for auth decisions
  Put passwords or sensitive PII inside the payload
```

---

## Connected topics

- **23 — crypto module** — hashing passwords with `scryptSync`/`randomBytes` before ever issuing a token; JWT auth always sits on top of safe password storage
- **58 — Authentication patterns — Sessions** — the alternative, stateful approach (server-stored sessions + cookies) and when it's a better fit than JWT
- **79 — Secrets management** — where `JWT_SECRET`/`JWT_ACCESS_SECRET` actually live in production (never hardcoded, never in git)
