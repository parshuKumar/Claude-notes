# 58 — Authentication patterns — Sessions

## What is this?

Session-based authentication is a way of remembering "who is logged in" by storing a small piece of state on the server and giving the browser a matching ID in a cookie. Think of it like a coat check at a restaurant — you hand over your coat (your credentials, once, at login) and get a numbered ticket (a cookie with a session ID). Every time you come back to the counter, you just show the ticket; the staff looks up your coat by that number. The server, not the client, holds the actual data — the browser only ever carries the ticket number.

## Why does it matter for backend development?

Sessions are the classic, battle-tested way to keep a user logged in across requests in server-rendered apps, admin dashboards, and any backend where you control both the server and can maintain state. Unlike JWTs (Topic 57), sessions can be **instantly revoked** — delete the row in the store and the user is logged out everywhere immediately, which matters for "log out all devices," banning users, or handling stolen credentials. Backend developers reach for sessions when they need traditional login flows with server-side logout control, and they must pair sessions with a shared store like Redis the moment they run more than one server instance, plus CSRF protection since cookies are sent automatically by the browser on every request.

---

## Syntax / API

```js
// npm install express-session connect-redis ioredis csurf
const express = require('express');
const session = require('express-session');       // session middleware for Express
const RedisStore = require('connect-redis').default; // session store backed by Redis
const Redis = require('ioredis');                 // Redis client

const app = express();

// ── 1. Create a Redis client the session store will use ────────────────────
const redisClient = new Redis(process.env.REDIS_URL); // connects to Redis (Topic 92)

// ── 2. Configure express-session middleware ─────────────────────────────────
app.use(session({
  store: new RedisStore({ client: redisClient }), // persist sessions in Redis, not memory
  secret: process.env.SESSION_SECRET,             // used to sign the session ID cookie
  resave: false,                                  // don't re-save if nothing changed
  saveUninitialized: false,                       // don't create a session until data is set
  cookie: {
    httpOnly: true,                               // JS on the page cannot read this cookie
    secure: process.env.NODE_ENV === 'production',// only sent over HTTPS in production
    sameSite: 'lax',                              // basic CSRF mitigation for top-level nav
    maxAge: 1000 * 60 * 60 * 24,                  // cookie/session lives 24 hours
  },
}));

// ── 3. Reading and writing session data in a route ──────────────────────────
app.post('/login', (req, res) => {
  // ...verify username/password against the database first...
  req.session.userId = 'user_42';                 // store data on req.session — Node saves it
  res.json({ message: 'Logged in' });
});

app.get('/me', (req, res) => {
  if (!req.session.userId) {                       // no userId means no active session
    return res.status(401).json({ error: 'Not logged in' });
  }
  res.json({ userId: req.session.userId });        // read data back on later requests
});

app.post('/logout', (req, res) => {
  req.session.destroy(() => {                      // remove the session from the store
    res.clearCookie('connect.sid');                // remove the cookie from the browser too
    res.json({ message: 'Logged out' });
  });
});
```

---

## How it works — line by line

- `new Redis(process.env.REDIS_URL)` opens a connection to a Redis server — a fast in-memory database perfect for storing millions of small session records.
- `new RedisStore({ client: redisClient })` wraps that Redis connection in an adapter that `express-session` knows how to talk to — read, write, and expire session data.
- `session({ ... })` returns middleware that runs on **every request**. On the first request it reads the cookie named `connect.sid`, checks whether a matching session exists in the store, and if so, loads that data onto `req.session`.
- `secret` is used to **sign** the cookie (add an HMAC signature), so a client cannot forge or tamper with the session ID — it does not encrypt the session data itself, since the data never leaves the server.
- `resave: false` stops the middleware from writing the session back to the store on every single request if nothing changed — this reduces unnecessary writes to Redis.
- `saveUninitialized: false` means an empty, unmodified session (e.g. an anonymous visitor who never logs in) is never saved or given a cookie — this avoids filling your store with junk sessions.
- `cookie.httpOnly: true` tells the browser to hide this cookie from `document.cookie` in JavaScript, which blocks a whole class of XSS-based cookie theft.
- `cookie.secure: true` (in production) tells the browser to only ever send this cookie over an HTTPS connection, never plain HTTP.
- `cookie.sameSite: 'lax'` tells the browser not to send this cookie on most cross-site requests, which is the first line of defense against CSRF.
- `req.session.userId = 'user_42'` simply assigns a property — `express-session` proxies this object and automatically persists it to the store (Redis) at the end of the request.
- `req.session.destroy(callback)` deletes the session record from the store entirely, so even if someone still has the old cookie, it no longer matches anything.
- `res.clearCookie('connect.sid')` tells the browser to delete the cookie on its side too, for good measure.

---

## Example 1 — basic

```js
// File: src/basic-session-server.js
// Minimal session demo using the default in-memory store — fine for learning, NOT production.

const express = require('express');
const session = require('express-session'); // brings in the session middleware

const app = express();
app.use(express.json()); // parse JSON request bodies

// Configure sessions — default store keeps data in server RAM (lost on restart)
app.use(session({
  secret: 'dev-only-secret-change-me', // signs the cookie — use env var in real apps
  resave: false,                       // don't rewrite unchanged sessions
  saveUninitialized: false,            // don't save empty sessions
  cookie: { maxAge: 1000 * 60 * 15 },  // session cookie expires after 15 minutes
}));

// Track how many times this particular visitor has hit the page
app.get('/visit', (req, res) => {
  req.session.visitCount = (req.session.visitCount || 0) + 1; // increment or start at 1
  res.json({ visits: req.session.visitCount });                // send the running count back
});

// Simple fake login that stores a userId in the session
app.post('/login', (req, res) => {
  req.session.userId = 'user_7';        // pretend this user just authenticated
  res.json({ message: 'session created' });
});

// Route that only works if a session with userId exists
app.get('/dashboard', (req, res) => {
  if (!req.session.userId) {                        // guard clause — no session, no access
    return res.status(401).json({ error: 'Please log in' });
  }
  res.json({ welcome: req.session.userId });         // greet the logged-in user
});

app.listen(3000, () => console.log('Server on http://localhost:3000'));
```

---

## Example 2 — real world backend use case

```js
// File: src/auth/session-auth.js
// Production-shaped session setup: Redis store, CSRF protection, and a requireAuth guard
// used across multiple protected routes in a real Express backend.

const express = require('express');
const session = require('express-session');
const RedisStore = require('connect-redis').default;
const Redis = require('ioredis');
const csurf = require('csurf'); // CSRF token middleware

const app = express();
app.use(express.json());
app.use(express.urlencoded({ extended: true })); // needed for form-posted CSRF tokens

const redisClient = new Redis(process.env.REDIS_URL); // shared Redis connection

redisClient.on('error', (err) => {
  console.error('[redis] connection error:', err.message); // never let a Redis crash go silent
});

// ── Session middleware, backed by Redis so it works across multiple server instances ──
app.use(session({
  store: new RedisStore({ client: redisClient, prefix: 'sess:' }), // namespace the keys
  secret: process.env.SESSION_SECRET,
  resave: false,
  saveUninitialized: false,
  cookie: {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'lax',
    maxAge: 1000 * 60 * 60 * 2, // 2-hour rolling session lifetime
  },
}));

// ── CSRF protection — issues and verifies a token tied to the session ─────────────
const csrfProtection = csurf(); // uses the session to store the expected token

// Endpoint the frontend calls once to get a CSRF token before submitting forms
app.get('/csrf-token', csrfProtection, (req, res) => {
  res.json({ csrfToken: req.csrfToken() }); // send token for the client to echo back
});

// ── Reusable guard middleware for any route that requires a logged-in user ────────
function requireAuth(req, res, next) {
  if (!req.session.userId) {                             // no active session
    return res.status(401).json({ error: 'Authentication required' });
  }
  next(); // session is valid — continue to the actual route handler
}

// Login route — verifies credentials, then creates the session
app.post('/login', async (req, res) => {
  const { email, password } = req.body;               // credentials sent by the client
  const dbUser = await findUserByEmail(email);         // pretend DB lookup (Topic 90/91)

  if (!dbUser || !(await verifyPassword(password, dbUser.passwordHash))) {
    return res.status(401).json({ error: 'Invalid email or password' });
  }

  req.session.regenerate((err) => {                    // rotate session ID on login (prevents fixation)
    if (err) return res.status(500).json({ error: 'Login failed' });
    req.session.userId = dbUser.id;                    // store the authenticated user's ID
    res.json({ message: 'Logged in', userId: dbUser.id });
  });
});

// Protected route — mutating request, so CSRF protection is applied too
app.post('/account/email', requireAuth, csrfProtection, async (req, res) => {
  const { newEmail } = req.body;
  await updateUserEmail(req.session.userId, newEmail); // apply the change in the database
  res.json({ message: 'Email updated' });
});

// Logout — destroys the server-side session so the ID can never be reused
app.post('/logout', requireAuth, (req, res) => {
  req.session.destroy(() => {
    res.clearCookie('connect.sid');
    res.json({ message: 'Logged out' });
  });
});

// Stub helpers used above — real implementations would query a database
async function findUserByEmail(email) { /* ...db lookup... */ }
async function verifyPassword(password, hash) { /* ...bcrypt.compare... */ }
async function updateUserEmail(userId, newEmail) { /* ...db update... */ }

app.listen(3000, () => console.log('Auth server on http://localhost:3000'));
```

---

## Common mistakes

### Mistake 1 — Using the default MemoryStore in production

```js
// ❌ WRONG — express-session's built-in MemoryStore leaks memory and
// is wiped every time the server restarts or you scale to 2+ instances
const session = require('express-session');
app.use(session({
  secret: process.env.SESSION_SECRET,
  resave: false,
  saveUninitialized: false,
  // no `store` set → falls back to in-memory storage, unusable in production
}));

// ✅ CORRECT — back sessions with Redis so they survive restarts and
// are shared across every instance of your app behind a load balancer
const RedisStore = require('connect-redis').default;
const Redis = require('ioredis');
const redisClient = new Redis(process.env.REDIS_URL);

app.use(session({
  store: new RedisStore({ client: redisClient }), // shared, persistent session storage
  secret: process.env.SESSION_SECRET,
  resave: false,
  saveUninitialized: false,
}));
```

### Mistake 2 — Forgetting cookie security flags

```js
// ❌ WRONG — cookie is readable by client-side JS and sent over plain HTTP,
// making it an easy target for XSS-based theft and network sniffing
app.use(session({
  secret: process.env.SESSION_SECRET,
  cookie: {}, // httpOnly/secure/sameSite all left at insecure defaults
}));

// ✅ CORRECT — lock the cookie down so it can only travel the way you intend
app.use(session({
  secret: process.env.SESSION_SECRET,
  cookie: {
    httpOnly: true,                              // blocks document.cookie access
    secure: process.env.NODE_ENV === 'production', // HTTPS only in prod
    sameSite: 'lax',                              // blocks most cross-site sending
    maxAge: 1000 * 60 * 60,                       // always set an explicit expiry
  },
}));
```

### Mistake 3 — Skipping CSRF protection on state-changing routes

```js
// ❌ WRONG — any site the user visits while logged in can silently POST here,
// because the browser attaches the session cookie automatically
app.post('/account/delete', requireAuth, (req, res) => {
  deleteAccount(req.session.userId); // no CSRF check — vulnerable to a forged form
  res.json({ message: 'Account deleted' });
});

// ✅ CORRECT — require a CSRF token that only your own frontend could have fetched
const csrfProtection = require('csurf')();

app.post('/account/delete', requireAuth, csrfProtection, (req, res) => {
  deleteAccount(req.session.userId); // csurf already verified the token before this runs
  res.json({ message: 'Account deleted' });
});
```

---

## Practice exercises

### Exercise 1 — easy

Build a small Express server with `express-session` using the default in-memory store (this is fine for this exercise only). Create:
1. A `GET /counter` route that increments `req.session.visits` on every request and returns the current count
2. A `POST /reset` route that resets `req.session.visits` back to `0`
3. Cookie options that set `httpOnly: true` and a `maxAge` of 5 minutes

Test it by hitting `/counter` several times in a browser or with curl (keeping cookies between requests) and confirm the count increases per-browser, not globally.

```js
// Write your code here
```

---

### Exercise 2 — medium

Build a login system using `express-session` with:
1. A hardcoded in-memory array of users (email + plaintext password is fine for this exercise)
2. A `POST /login` route that checks the email/password, and on success stores `req.session.userId` and calls `req.session.regenerate()` before setting it
3. A `requireAuth` middleware function that checks `req.session.userId` and returns `401` if missing
4. A `GET /profile` route protected by `requireAuth` that returns the logged-in user's email
5. A `POST /logout` route that destroys the session and clears the cookie

Test the full flow: log in, call `/profile` successfully, log out, then call `/profile` again and confirm it now returns `401`.

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a production-shaped session auth module that:
1. Uses `connect-redis` and `ioredis` as the session store (you can point it at a local Redis instance, e.g. via Docker)
2. Sets all recommended cookie flags (`httpOnly`, `secure` conditional on `NODE_ENV`, `sameSite: 'lax'`, and a `maxAge`)
3. Adds `csurf` CSRF protection on a `GET /csrf-token` route and enforces it on every state-changing route (`POST`, `PUT`, `DELETE`)
4. Implements a `requireAuth` middleware and a `requireRole(role)` middleware factory that both check `req.session`
5. Adds a `POST /admin/revoke-all-sessions` route that, using the Redis client directly, deletes **every** session key matching the `sess:` prefix — simulating a "force logout everyone" security action
6. Handles Redis connection errors gracefully (log them, don't crash the whole server)

Test it end-to-end: log in, fetch a CSRF token, hit a protected `POST` route successfully, then call the revoke-all-sessions route and confirm your original session cookie no longer works.

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
CORE PACKAGES
  express-session   → session middleware, manages req.session + cookie
  connect-redis      → adapter so express-session can store data in Redis
  ioredis / redis    → Redis client library used by the store
  csurf              → CSRF token generation and verification middleware

KEY OPTIONS (express-session)
  secret              → signs the session ID cookie (use a long random env var)
  store               → WHERE session data lives (Redis in production, never MemoryStore)
  resave: false       → don't re-save session if nothing changed
  saveUninitialized: false → don't create a session/cookie until data is set
  cookie.httpOnly     → hides cookie from client-side JS (XSS defense)
  cookie.secure       → cookie only sent over HTTPS
  cookie.sameSite     → 'lax'/'strict' — blocks most cross-site cookie sending
  cookie.maxAge       → session lifetime in milliseconds

SESSION LIFECYCLE
  req.session.KEY = value       → write data, auto-persisted to the store
  req.session.KEY               → read data back on later requests
  req.session.regenerate(cb)    → rotate session ID (call on login — prevents fixation)
  req.session.destroy(cb)       → delete session from store (call on logout)
  res.clearCookie('connect.sid')→ remove cookie from the browser after destroy

SESSIONS VS JWT (Topic 57)
  Sessions → state lives on server, instantly revocable, needs shared store to scale
  JWT      → state lives in the token, stateless, cannot be revoked without a blocklist

CSRF DEFENSE LAYERS
  cookie.sameSite: 'lax'   → browser-level, blocks most cross-site sends
  csurf token              → app-level, verifies a per-session secret token
  Never trust GET routes for state changes → CSRF only matters for mutating requests

NEVER DO
  Default MemoryStore in production        → memory leak + no multi-instance support
  Skip httpOnly/secure/sameSite on cookies  → cookie theft, session hijacking
  Skip CSRF tokens on POST/PUT/DELETE       → forged cross-site requests succeed
  Reuse the same session ID after login    → session fixation attack
```

---

## Connected topics

- **57 — Authentication patterns — JWT** — the stateless alternative to sessions; understand the trade-offs (revocability vs scalability) between the two approaches
- **92 — Connecting to Redis** — the store that backs `connect-redis` in production; covers `ioredis` connection setup, expiry, and pub/sub in depth
- **78 — XSS, CSRF, Clickjacking** — goes deeper into `helmet.js`, `SameSite` cookie configuration, and CSRF defenses beyond what a session middleware provides alone
