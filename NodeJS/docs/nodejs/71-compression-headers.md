# 71 — Compression and performance headers

## What is this?

Compression shrinks the response body (HTML, JSON, CSS, JS) before it travels over the network, so the same data uses fewer bytes on the wire — like vacuum-sealing clothes into a suitcase, they take up less space in transit and get "unpacked" (decompressed) at the destination. Performance headers (`ETag`, `Cache-Control`, `Last-Modified`) are labels your server attaches to a response that tell the browser or CDN "here's how fresh this is and when you're allowed to reuse your last copy instead of asking me again." Together they reduce both the size of each response and the number of responses your server has to send at all.

## Why does it matter for backend development?

A backend that sends every response uncompressed and with no caching hints re-sends the full payload on every single request, even to the same client asking for the same unchanged data seconds later. That wastes bandwidth, slows down page loads (especially on mobile networks), and burns unnecessary CPU and I/O on your server under load. Every production Express/Fastify API uses `compression` middleware for gzip/brotli responses, and every API serving static assets, images, or rarely-changing JSON uses `ETag`/`Cache-Control`/`Last-Modified` so clients and CDNs can skip round trips entirely with a cheap `304 Not Modified` instead of re-downloading the whole body. This is one of the highest-leverage, lowest-effort performance wins a backend developer can ship.

---

## Syntax / API

```js
// Install once: npm install express compression
const express    = require('express');
const compression = require('compression');   // gzip/brotli middleware for Express

const app = express();

// ── 1. Compression middleware ───────────────────────────────────────────────
// Mount early — it wraps res.write/res.end to compress everything sent after it
app.use(compression({
  threshold: 1024,   // don't bother compressing bodies smaller than 1 KB (not worth the CPU)
  level: 6,           // gzip level 1 (fastest, weakest) to 9 (slowest, smallest) — 6 is the default balance
}));

// ── 2. Cache-Control header ──────────────────────────────────────────────────
app.get('/api/products/:productId', (req, res) => {
  // max-age is in SECONDS — browser/CDN can reuse this response for 300s without asking again
  res.set('Cache-Control', 'public, max-age=300');
  res.json({ productId: req.params.productId, name: 'Wireless Mouse' });
});

// ── 3. ETag — a fingerprint of the response body ────────────────────────────
// Express generates weak ETags automatically by default (app.set('etag', 'weak'))
// The client sends it back next time as "If-None-Match" — server compares and can reply 304

// ── 4. Last-Modified — a timestamp of when the resource last changed ───────
const fs = require('fs');
app.get('/downloads/report.pdf', (req, res) => {
  const stats = fs.statSync('./files/report.pdf');
  res.set('Last-Modified', stats.mtime.toUTCString());   // e.g. "Tue, 01 Jul 2025 10:00:00 GMT"
  res.sendFile('./files/report.pdf', { root: __dirname }); // res.sendFile handles ETag + 304 for you
});

app.listen(3000);   // covered in Topic 53 — Express fundamentals
```

---

## How it works — line by line

- `app.use(compression({ ... }))` wires a piece of middleware into every request. Once mounted, it intercepts the outgoing response, checks the client's `Accept-Encoding` header (does this browser support gzip/br?), compresses the body if so, and sets `Content-Encoding: gzip` (or `br`) on the response so the client knows how to decompress it.
- `threshold: 1024` tells the middleware "skip compression for tiny responses" — compressing a 50-byte JSON reply wastes more CPU than it saves in bytes.
- `level: 6` controls the compression/CPU tradeoff. Higher levels squeeze out more bytes but take longer to compute — under heavy traffic, a very high level can actually slow your server down.
- `res.set('Cache-Control', 'public, max-age=300')` attaches an instruction to the response: "any cache (browser, CDN, proxy) may store this and reuse it for up to 300 seconds without contacting the server again."
- An `ETag` is a hash (fingerprint) computed from the response body. The server sends it once; the browser stores it. Next time the browser asks for the same URL, it sends the ETag back in an `If-None-Match` header. If the server computes the same hash (the content hasn't changed), it replies with an empty `304 Not Modified` instead of re-sending the whole body.
- `Last-Modified` works the same way but with a timestamp instead of a hash. The browser sends it back as `If-Modified-Since`; if the file hasn't changed since that timestamp, the server replies `304`.
- `res.sendFile()` in Express automatically computes both `ETag` and `Last-Modified` for the file being sent and handles the `304` logic for you — you rarely need to do this by hand for static files.

---

## Example 1 — basic

```js
// File: src/basic-compression.js
const express     = require('express');
const compression = require('compression');   // gzip middleware

const app = express();

// Mount compression FIRST, before any routes — it must wrap every later response
app.use(compression());

// A route returning a large JSON array — a good candidate for compression
app.get('/api/orders', (req, res) => {
  // Build a chunky payload to make the compression benefit visible
  const orders = Array.from({ length: 500 }, (_, i) => ({
    orderId: `order_${i}`,
    status: 'delivered',
    total: 49.99,
  }));

  res.json(orders);   // compression middleware gzips this automatically before sending
});

app.listen(3000, () => {
  console.log('Server listening on port 3000');   // try: curl -H "Accept-Encoding: gzip" -v http://localhost:3000/api/orders
});

// Result: response headers will include "Content-Encoding: gzip"
// and the body over the wire is dramatically smaller than the raw JSON
```

---

## Example 2 — real world backend use case

```js
// File: src/api-cache-headers.js
// A realistic API endpoint: compress everything, and let CDNs/browsers
// skip re-fetching a user profile that hasn't changed.

const express     = require('express');
const compression = require('compression');
const crypto      = require('crypto');   // for computing our own content hash

const app = express();

app.use(compression());   // compress all responses (JSON, HTML, etc.)

// Fake "database" lookup — in real life this hits Postgres/Mongo
function findUserById(userId) {
  return {
    userId,
    displayName: 'Riya Sharma',
    email: 'riya@example.com',
    updatedAt: '2026-06-10T08:00:00.000Z',
  };
}

app.get('/api/users/:userId', (req, res) => {
  const { userId } = req.params;
  const userRecord  = findUserById(userId);          // simulate a DB read

  // Build a stable JSON string so the ETag is deterministic
  const responseBody = JSON.stringify(userRecord);

  // Compute our own ETag from the body content (Express does this automatically
  // by default, but this shows what's happening under the hood)
  const etag = crypto
    .createHash('sha1')
    .update(responseBody)
    .digest('hex');

  res.set('ETag', `"${etag}"`);                       // ETags are conventionally quoted
  res.set('Cache-Control', 'private, max-age=60');     // private: only THIS user's browser may cache it
  res.set('Last-Modified', new Date(userRecord.updatedAt).toUTCString());

  // If the client already has this exact version, skip re-sending the body
  const clientEtag = req.headers['if-none-match'];
  if (clientEtag === `"${etag}"`) {
    return res.status(304).end();   // 304 = "your cached copy is still correct"
  }

  res.type('application/json').send(responseBody);   // full body only sent when it actually changed
});

app.listen(3000, () => {
  console.log('API server with cache headers running on port 3000');
});
```

---

## Common mistakes

### Mistake 1 — Compressing already-compressed files (images, videos, zips)

```js
// ❌ WRONG — running compression on every route, including binary assets
// that are ALREADY compressed (JPEG, PNG, MP4, ZIP). This wastes CPU for
// zero size benefit — sometimes the "compressed" file is even bigger.
const express     = require('express');
const compression = require('compression');
const app = express();

app.use(compression());                       // applies to EVERYTHING, including /images
app.use('/images', express.static('uploads')); // JPEGs get needlessly re-compressed

// ✅ CORRECT — use compression's filter option to skip already-compressed types
app.use(compression({
  filter: (req, res) => {
    // Skip compression for binary/media routes — they gain nothing from gzip
    if (req.path.startsWith('/images') || req.path.startsWith('/videos')) {
      return false;
    }
    return compression.filter(req, res);   // fall back to default rules for everything else
  },
}));
app.use('/images', express.static('uploads'));
```

### Mistake 2 — Caching a response that contains user-specific or sensitive data as `public`

```js
// ❌ WRONG — marking a per-user response as "public" lets shared caches
// (corporate proxies, CDNs) serve one user's private data to another user
app.get('/api/account/balance', (req, res) => {
  res.set('Cache-Control', 'public, max-age=600');   // DANGEROUS for private data
  res.json({ userId: req.user.userId, balance: 4200 });
});

// ✅ CORRECT — use "private" so only the requesting browser may cache it,
// and keep the TTL short for data that changes often
app.get('/api/account/balance', (req, res) => {
  res.set('Cache-Control', 'private, max-age=0, must-revalidate');
  res.json({ userId: req.user.userId, balance: 4200 });
});
```

### Mistake 3 — Setting `Last-Modified`/`ETag` but never checking the incoming conditional headers

```js
// ❌ WRONG — the server sends ETag/Last-Modified but ignores the client's
// "If-None-Match" / "If-Modified-Since" on the next request, so it always
// resends the full body — the caching headers are doing nothing
app.get('/reports/:reportId', (req, res) => {
  const report = loadReport(req.params.reportId);
  res.set('ETag', `"${report.hash}"`);
  res.json(report);   // full body sent EVERY time, even if unchanged
});

// ✅ CORRECT — actually compare the incoming header and short-circuit with 304
app.get('/reports/:reportId', (req, res) => {
  const report = loadReport(req.params.reportId);
  const etag   = `"${report.hash}"`;

  res.set('ETag', etag);

  if (req.headers['if-none-match'] === etag) {
    return res.status(304).end();   // tell the client: your cached copy is still valid
  }

  res.json(report);   // only sent when the content actually differs
});
```

---

## Practice exercises

### Exercise 1 — easy

Build a small Express server with two routes:
1. `GET /api/heavy-data` — returns a JSON array of at least 300 objects (any shape you like)
2. Mount the `compression` middleware so the response is gzip-compressed

Use `curl -H "Accept-Encoding: gzip" -v http://localhost:3000/api/heavy-data` (or a similar HTTP client) to confirm the response includes a `Content-Encoding: gzip` header.

```js
// Write your code here
```

---

### Exercise 2 — medium

Build an endpoint `GET /api/products/:productId` that:
1. Looks up a product from an in-memory array (simulate a database)
2. Sets a `Cache-Control` header with `public, max-age=120`
3. Computes an `ETag` from the JSON body using `crypto.createHash('sha1')`
4. Checks the incoming `If-None-Match` header — if it matches, respond with `304` and no body
5. Otherwise, send the full JSON with a `200` status and the `ETag` header set

Test it by making the same request twice with the same client and confirming the second request gets a `304`.

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a static file server for a `/downloads` folder that:
1. Serves files by name, e.g. `GET /downloads/invoice.pdf`
2. Reads the file's `mtime` (modification time) with `fs.statSync` and sets it as the `Last-Modified` header
3. Compares the incoming `If-Modified-Since` header against the file's `mtime` — if the file has not changed since that timestamp, responds with `304` and no body
4. Otherwise streams the file to the client with the correct `Content-Type` based on file extension (at least handle `.pdf`, `.json`, `.txt`)
5. Wraps the whole app in `compression` middleware, but excludes `.pdf` files from compression using the `filter` option (PDFs are already compressed internally)
6. Handle the case where the requested file does not exist — respond with `404`

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
COMPRESSION
  npm install compression
  app.use(compression())                  → gzip/brotli every response after this line
  compression({ threshold: 1024 })        → skip tiny bodies, not worth the CPU
  compression({ level: 1-9 })             → 1 = fastest/weakest, 9 = slowest/smallest, 6 = default
  compression({ filter: fn })             → skip already-compressed types (images, video, zip)
  Response header set                     → Content-Encoding: gzip (or br)

CACHE-CONTROL DIRECTIVES
  public                    → any cache (browser, CDN, proxy) may store it
  private                   → only the requesting browser may store it (use for per-user data)
  max-age=<seconds>         → how long the cached copy is considered fresh
  no-cache                  → may store, but MUST revalidate with server before reuse
  no-store                  → never cache at all (use for auth tokens, passwords, secrets)
  must-revalidate           → once stale, must check with server, never serve stale on error

ETAG
  What it is        → a fingerprint (hash) of the response body
  Server sends      → ETag: "abc123"
  Client sends back → If-None-Match: "abc123"        (on the next request)
  Match             → server replies 304 Not Modified, empty body
  No match          → server replies 200 with full new body + new ETag
  Express default   → auto-generated "weak" ETag for every response (app.get('etag'))

LAST-MODIFIED
  What it is        → timestamp of when the resource last changed
  Server sends      → Last-Modified: Tue, 01 Jul 2025 10:00:00 GMT
  Client sends back → If-Modified-Since: Tue, 01 Jul 2025 10:00:00 GMT
  Match/unchanged   → server replies 304 Not Modified
  Less precise than ETag (second-level granularity) but cheaper to compute

WHEN TO USE WHAT
  Static assets (CSS/JS/images)  → long max-age + ETag (filenames often hashed too)
  Per-user API data              → private, short max-age, ETag for revalidation
  Auth/session/secret data       → no-store, never cache
  Large JSON/HTML responses      → always compress
  Already-compressed binaries    → skip compression, still send caching headers

COMMON PATTERNS
  res.sendFile()                    → Express auto-sets ETag + Last-Modified + handles 304
  express.static()                  → built-in Cache-Control + ETag for static folders
  res.status(304).end()             → correct way to short-circuit a conditional request
```

---

## Connected topics

- **33 — zlib module** — the raw Node.js compression primitives (`gzip`, `deflate`) that the `compression` middleware wraps under the hood
- **36 — HTTP deep dive — headers, methods, status codes** — the full picture of request/response headers, including `Accept-Encoding`, `If-None-Match`, and status code semantics used here
- **69 — Caching strategies** — application-level caching (Redis, in-memory) that complements HTTP caching headers for reducing database load
