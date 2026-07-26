# 70 — Rate limiting

## What is this?

Rate limiting is the practice of capping how many requests a client (a user, an IP address, or an API key) can make to your server within a given time window. Once the cap is hit, the server rejects further requests — usually with a `429 Too Many Requests` status — until the window resets. Think of it like a nightclub bouncer with a strict headcount: once the room is full, people are turned away at the door regardless of how politely they ask, and only let back in once someone else leaves.

## Why does it matter for backend development?

Without rate limiting, a single misbehaving client — a buggy frontend retry loop, a scraper, or a brute-force login attempt — can exhaust your server's CPU, database connections, or third-party API quota, degrading service for every other user. Backend developers add rate limiting to login endpoints (to slow down credential stuffing), public APIs (to enforce pricing tiers), and any expensive route (search, file uploads, password reset emails) to keep the system fair and available. In production systems running on multiple server instances, rate limiting must be backed by a shared store like Redis, otherwise each instance enforces its own separate limit and the real limit becomes `limit × number of instances`.

---

## Syntax / API

```js
// npm install express express-rate-limit
const express = require('express');
const rateLimit = require('express-rate-limit'); // middleware factory for rate limiting

const app = express();

// Create a limiter — this is middleware, not a standalone function
const apiLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // the time window: 15 minutes, in milliseconds
  max: 100,                 // max requests allowed per client within the window
  standardHeaders: true,    // send RateLimit-* headers (modern standard)
  legacyHeaders: false,     // disable old X-RateLimit-* headers
  message: { error: 'Too many requests, please try again later.' }, // 429 body
});

// Apply the limiter to every route on the app
app.use(apiLimiter);

// Or apply it to only one route/group of routes
app.use('/api', apiLimiter);

// Or scope it to a single sensitive endpoint (e.g. login)
const loginLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minute window
  max: 5,                   // only 5 login attempts allowed per window
  keyGenerator: (req) => req.body.email || req.ip, // limit per email, fallback to IP
});
app.post('/auth/login', loginLimiter, (req, res) => {
  // ... login logic
});
```

---

## How it works — line by line

- `rateLimit({...})` is a factory function — you call it once with a configuration object, and it hands back an Express middleware function you can plug into `app.use()` or a specific route.
- `windowMs` defines the size of the time bucket — every client gets a "budget" of requests that resets after this many milliseconds pass.
- `max` is the ceiling — the maximum number of requests a single client can make before the window resets.
- Internally, on every incoming request the middleware looks up a key (by default the client's IP address) in a store, increments a counter for that key, and checks if the counter has crossed `max`. If it has, the middleware short-circuits the request and sends a `429` response instead of calling `next()`.
- `standardHeaders: true` tells the middleware to attach `RateLimit-Limit`, `RateLimit-Remaining`, and `RateLimit-Reset` headers to every response, so clients can see how much budget they have left.
- `keyGenerator` lets you change what identifies a "client" — instead of IP, you can limit per logged-in user, per API key, or (as in the login example) per email address attempted, which stops attackers from spreading a brute-force attack across many IPs.
- By default, `express-rate-limit` stores counters **in memory** on that one process. This works for a single-instance app but resets on restart and does not share state across multiple server processes — that's where a Redis-backed store comes in (covered in Example 2).

---

## Example 1 — basic

```js
// File: src/server.js
// A minimal Express server with a global rate limit on every route.

const express = require('express');       // web framework
const rateLimit = require('express-rate-limit'); // rate limiting middleware

const app = express();

// Define the limiter: 100 requests per IP per 15-minute window
const globalLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,  // 15 minutes in milliseconds
  max: 100,                  // allow up to 100 requests in that window
  standardHeaders: true,     // return RateLimit-* headers so clients can self-throttle
  legacyHeaders: false,      // skip the deprecated X-RateLimit-* headers
  handler: (req, res) => {
    // Custom handler runs instead of the default message when the limit is hit
    res.status(429).json({
      error: 'Rate limit exceeded',
      retryAfterSeconds: Math.ceil(req.rateLimit.resetTime - Date.now()) / 1000,
    });
  },
});

app.use(globalLimiter); // applies to every incoming request on this app

// A simple test route
app.get('/ping', (req, res) => {
  res.json({ message: 'pong' }); // just proves the server is alive
});

app.listen(3000, () => {
  console.log('Server running on port 3000'); // startup log
});

// Try hammering GET http://localhost:3000/ping more than 100 times
// within 15 minutes — the 101st request returns 429 automatically.
```

---

## Example 2 — real world backend use case

```js
// File: src/middleware/rateLimiters.js
// Production pattern: Redis-backed rate limiting so limits are enforced
// correctly across multiple server instances behind a load balancer.

// npm install express-rate-limit rate-limit-redis ioredis
const rateLimit = require('express-rate-limit');
const { RedisStore } = require('rate-limit-redis'); // Redis-backed store adapter
const Redis = require('ioredis');                   // Redis client

// One shared Redis connection reused by all limiters in this app
const redisClient = new Redis(process.env.REDIS_URL || 'redis://localhost:6379');

// General API limiter — moderate limit for authenticated users
const apiLimiter = rateLimit({
  windowMs: 60 * 1000,     // 1-minute sliding window
  max: 60,                 // 60 requests per minute per user
  standardHeaders: true,
  legacyHeaders: false,
  // Key by authenticated userId when available, else fall back to IP
  keyGenerator: (req) => req.user?.userId || req.ip,
  store: new RedisStore({
    // sendCommand bridges rate-limit-redis to the ioredis client's API
    sendCommand: (...args) => redisClient.call(...args),
    prefix: 'rl:api:', // namespace keys so they don't collide with cache keys
  }),
});

// Strict limiter for password reset — prevents email-bombing attacks
const passwordResetLimiter = rateLimit({
  windowMs: 60 * 60 * 1000, // 1-hour window
  max: 3,                   // only 3 reset requests per hour
  keyGenerator: (req) => req.body.email, // limit per target email, not per IP
  store: new RedisStore({
    sendCommand: (...args) => redisClient.call(...args),
    prefix: 'rl:reset:',
  }),
  message: { error: 'Too many password reset requests. Try again in an hour.' },
});

// A token-bucket-style limiter for a paid tier that allows short bursts
// (express-rate-limit itself is fixed/sliding window; true token-bucket
// logic is implemented manually against Redis for burst-friendly APIs)
async function tokenBucketLimiter(req, res, next) {
  const apiKey = req.headers['x-api-key'];           // identify the client by API key
  const bucketKey = `rl:bucket:${apiKey}`;            // Redis key for this client's bucket
  const capacity = 50;                                // max tokens the bucket can hold
  const refillRatePerSec = 5;                         // tokens added back per second
  const now = Date.now();

  // Lua script executed atomically in Redis so concurrent requests don't race
  const script = `
    local bucket = redis.call('HMGET', KEYS[1], 'tokens', 'timestamp')
    local tokens = tonumber(bucket[1]) or tonumber(ARGV[1])
    local timestamp = tonumber(bucket[2]) or tonumber(ARGV[3])
    local delta = math.max(0, tonumber(ARGV[3]) - timestamp)
    tokens = math.min(tonumber(ARGV[1]), tokens + delta * tonumber(ARGV[2]))
    if tokens < 1 then
      return 0
    end
    tokens = tokens - 1
    redis.call('HMSET', KEYS[1], 'tokens', tokens, 'timestamp', ARGV[3])
    redis.call('EXPIRE', KEYS[1], 3600)
    return 1
  `;

  const allowed = await redisClient.eval(
    script, 1, bucketKey, capacity, refillRatePerSec, now
  );

  if (allowed === 1) {
    return next(); // token consumed successfully — let the request through
  }
  res.status(429).json({ error: 'Rate limit exceeded, bucket empty' }); // no tokens left
}

module.exports = { apiLimiter, passwordResetLimiter, tokenBucketLimiter };
```

```js
// File: src/server.js
// Wiring the limiters into real routes
const express = require('express');
const {
  apiLimiter,
  passwordResetLimiter,
  tokenBucketLimiter,
} = require('./middleware/rateLimiters');

const app = express();
app.use(express.json());

app.use('/api', apiLimiter);                              // general API traffic
app.post('/auth/password-reset', passwordResetLimiter, (req, res) => {
  res.json({ message: 'Reset email sent if account exists' });
});
app.get('/api/v1/reports', tokenBucketLimiter, (req, res) => {
  res.json({ report: 'expensive report data here' }); // burst-friendly paid endpoint
});

app.listen(3000, () => console.log('Server running on port 3000'));
```

---

## Common mistakes

### Mistake 1 — Using in-memory rate limiting behind a load balancer

```js
// ❌ WRONG — default in-memory store, but the app runs as 4 PM2 cluster workers
const rateLimit = require('express-rate-limit');
const limiter = rateLimit({ windowMs: 60000, max: 100 });
app.use(limiter);
// Each of the 4 worker processes counts independently — a client can actually
// make 400 requests/minute (100 per worker) before ever being blocked.

// ✅ CORRECT — share the counter across all instances via Redis
const { RedisStore } = require('rate-limit-redis');
const Redis = require('ioredis');
const redisClient = new Redis(process.env.REDIS_URL);

const limiter = rateLimit({
  windowMs: 60000,
  max: 100, // now this is a true, cluster-wide limit of 100/minute
  store: new RedisStore({ sendCommand: (...args) => redisClient.call(...args) }),
});
app.use(limiter);
```

### Mistake 2 — Rate limiting by IP alone when clients share an IP

```js
// ❌ WRONG — corporate offices, mobile carriers, and NAT gateways put many
// users behind the SAME public IP; one heavy user blocks everyone else on it
const limiter = rateLimit({ windowMs: 60000, max: 20 }); // keyGenerator defaults to req.ip
app.use('/api', limiter);

// ✅ CORRECT — key by authenticated user or API key when the client is known,
// only fall back to IP for fully anonymous/unauthenticated traffic
const limiter = rateLimit({
  windowMs: 60000,
  max: 20,
  keyGenerator: (req) => req.user?.userId || req.headers['x-api-key'] || req.ip,
});
app.use('/api', limiter);
```

### Mistake 3 — Applying one blanket limit to every route regardless of cost

```js
// ❌ WRONG — a cheap GET /ping and an expensive POST /reports/generate
// (which triggers a heavy DB aggregation) share the exact same limit
const limiter = rateLimit({ windowMs: 60000, max: 1000 });
app.use(limiter); // 1000/min is fine for /ping but way too generous for /reports/generate

// ✅ CORRECT — scope stricter limits to expensive or sensitive routes,
// and looser limits to cheap, high-frequency routes
const cheapLimiter = rateLimit({ windowMs: 60000, max: 1000 });
const expensiveLimiter = rateLimit({ windowMs: 60000, max: 5 });

app.get('/ping', cheapLimiter, (req, res) => res.json({ ok: true }));
app.post('/reports/generate', expensiveLimiter, (req, res) => {
  // ... heavy aggregation logic, protected by a much tighter cap
});
```

---

## Practice exercises

### Exercise 1 — easy

Build a small Express app with two routes: `GET /health` (no rate limit) and `GET /search`. Add rate limiting to `/search` only, using `express-rate-limit`, allowing a maximum of 10 requests per 1-minute window per IP. Configure it to return `standardHeaders: true` and a custom JSON error body on the 429 response containing an `error` message and the `windowMs` value used. Test it by firing more than 10 requests in a row and confirming the 11th is rejected.

```js
// Write your code here
```

---

### Exercise 2 — medium

Write a `keyGenerator` function and a full rate limiter configuration for a `POST /auth/login` route where:
1. The limit is 5 attempts per 10-minute window.
2. The key must be a combination of the submitted `email` from the request body AND the client's IP address (so the same email can't be brute-forced from different IPs, and the same IP can't brute-force many different emails without eventually being caught too — think about how to combine both signals).
3. On the 6th attempt within the window, respond with `429` and a JSON body containing `error` and a `retryAfterSeconds` field computed from `req.rateLimit.resetTime`.
4. Successful logins (status 200) should NOT count against future attempts — read the `express-rate-limit` docs for the option that controls this and use it.

```js
// Write your code here
```

---

### Exercise 3 — hard

Implement a token-bucket rate limiter **from scratch** (no `express-rate-limit`, but you may use `ioredis`) as a piece of Express middleware called `createTokenBucketLimiter(options)` where `options` has `capacity` (max tokens), `refillRatePerSec` (tokens added per second), and `identify(req)` (a function returning the bucket key for a request). Requirements:
1. Store each bucket's `tokens` and `lastRefillTimestamp` in Redis as a hash.
2. On each request, atomically (using a Lua script via `redisClient.eval`) calculate how many tokens should have been refilled since the last check, cap the total at `capacity`, and attempt to consume 1 token.
3. If a token is available, let the request through and attach a header `X-RateLimit-Tokens-Remaining` with the resulting token count.
4. If no token is available, respond `429` with a JSON body containing `error` and the number of seconds until at least 1 token will be available again.
5. Demonstrate it protecting a `POST /api/v1/export` route with `capacity: 10`, `refillRatePerSec: 1`, keyed by `req.headers['x-api-key']`.

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
CORE CONCEPTS
  Rate limiting  → cap requests per client per time window, reject the rest with 429
  Fixed window   → counter resets entirely at fixed intervals (simple, can burst at edges)
  Sliding window → tracks requests within a rolling time span (smoother, more accurate)
  Token bucket   → bucket refills at a steady rate, requests consume tokens, allows bursts
                   up to bucket capacity while enforcing a long-term average rate

express-rate-limit BASICS
  npm install express-rate-limit
  rateLimit({ windowMs, max, standardHeaders, legacyHeaders, keyGenerator, handler, message })
  windowMs        → size of the time window in milliseconds
  max             → max requests allowed per key within the window
  keyGenerator    → (req) => string — defaults to req.ip, override for per-user/per-key limits
  standardHeaders → sends RateLimit-Limit / RateLimit-Remaining / RateLimit-Reset headers
  handler         → custom function to run instead of default 429 response
  skipSuccessfulRequests → don't count 2xx/3xx responses against the limit (good for login)

REDIS-BACKED (multi-instance apps)
  npm install rate-limit-redis ioredis
  store: new RedisStore({ sendCommand: (...args) => redisClient.call(...args) })
  Required whenever the app runs as more than one process/instance —
  in-memory store means each instance enforces its OWN separate limit

CHOOSING A KEY
  req.ip                          → anonymous public traffic (weak: shared IPs collide)
  req.user.userId                 → authenticated requests (strong: per-account fairness)
  req.headers['x-api-key']        → third-party API consumers (per API key/tier)
  email + ip combined             → login/auth endpoints (stops both IP-spread and
                                     multi-account brute-force attacks)

GOTCHAS
  In-memory store + PM2 cluster/multiple pods → real limit multiplies by instance count
  Rate limiting by IP only → shared NAT/corporate IPs punish innocent users
  One global limit for all routes → cheap and expensive routes need different caps
  Not excluding successful logins → legit users get locked out after password typos
  Forgetting to set an EXPIRE on Redis keys → stale counters/buckets leak memory forever

TYPICAL HTTP RESPONSE ON LIMIT HIT
  Status: 429 Too Many Requests
  Headers: RateLimit-Limit, RateLimit-Remaining: 0, RateLimit-Reset
  Body: { "error": "Too many requests, please try again later." }
```

---

## Connected topics

- **69 — Caching strategies** — both rely on Redis as a shared, fast, TTL-aware store; rate-limit counters and cache entries often live in the same Redis instance
- **92 — Connecting to Redis** — `ioredis` connection setup and command patterns used directly by `RedisStore` and the token-bucket Lua script
- **50 — Concurrency control** — p-limit/p-queue control how many operations *you* run at once; rate limiting controls how many requests *clients* can send you — complementary throttling at opposite ends of the request lifecycle
