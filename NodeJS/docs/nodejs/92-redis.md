# 92 — Connecting to Redis

## What is this?

Redis is an in-memory data store — it keeps data in RAM instead of on disk, which makes reading and writing extremely fast (microseconds, not milliseconds). `ioredis` is the most popular Node.js client library for talking to a Redis server — it lets your backend send commands like `GET`, `SET`, and `DEL` over a connection and get results back as JavaScript values. Think of Redis as a whiteboard next to your database's filing cabinet: the filing cabinet (PostgreSQL/MongoDB) is durable and organized but slow to search through repeatedly, while the whiteboard (Redis) holds whatever you need right now for instant access, and gets erased or rewritten constantly.

## Why does it matter for backend development?

Every backend eventually hits the same wall: the primary database is too slow to hit on every single request, especially for data that rarely changes (user profiles, product catalogs, config flags) or data that is inherently temporary (login sessions, OTP codes, rate-limit counters). Redis solves both problems — it caches expensive database results so repeat requests are served from memory, and it stores short-lived data with automatic expiry so you never need manual cleanup jobs. Redis also powers real-time features through pub/sub (notifying multiple server instances instantly) and backs job queues like BullMQ. Any backend developer working on a production API — especially one that needs to scale — will use Redis for caching, sessions, or rate limiting within their first few weeks on the job.

---

## Syntax / API

```js
// Install first: npm install ioredis
const Redis = require('ioredis');

// Create a connection to a local Redis server (default host/port)
const redisClient = new Redis({
  host: process.env.REDIS_HOST || '127.0.0.1', // Redis server address
  port: process.env.REDIS_PORT || 6379,        // default Redis port
  password: process.env.REDIS_PASSWORD,        // undefined if no auth configured
});

// ── Basic key-value operations ───────────────────────────────────────────────
await redisClient.set('userId:101', 'Amit Sharma');   // store a string value
const userName = await redisClient.get('userId:101'); // retrieve it → "Amit Sharma"
await redisClient.del('userId:101');                  // delete the key

// ── Expiry (TTL — time to live) ──────────────────────────────────────────────
await redisClient.set('otp:9876543210', '482913', 'EX', 300); // expires in 300 seconds
await redisClient.expire('sessionId:abc123', 3600);            // set/refresh TTL separately
const secondsLeft = await redisClient.ttl('otp:9876543210');   // → seconds remaining, -1 = no expiry, -2 = gone

// ── Storing objects (Redis only stores strings, so we serialize) ────────────
const requestBody = { userId: 101, email: 'amit@example.com' };
await redisClient.set('session:abc123', JSON.stringify(requestBody), 'EX', 3600);
const raw = await redisClient.get('session:abc123');
const sessionData = raw ? JSON.parse(raw) : null; // parse back into an object

// ── Pub/Sub (publish/subscribe messaging) ────────────────────────────────────
const subscriber = new Redis();                       // subscriber needs its own connection
subscriber.subscribe('order-events');                 // listen on a channel
subscriber.on('message', (channel, message) => {
  console.log(`Received on ${channel}:`, message);     // fires whenever someone publishes
});

const publisher = new Redis();                        // publisher can reuse the main client too
publisher.publish('order-events', JSON.stringify({ orderId: 55, status: 'shipped' }));

// ── Closing the connection ───────────────────────────────────────────────────
await redisClient.quit(); // gracefully close, waits for pending commands to finish
```

---

## How it works — line by line

`new Redis({...})` opens a TCP connection to the Redis server at startup and keeps it open for the life of your app — you do not open and close a connection per request like you might with a file. Every method call (`.set`, `.get`, `.del`) sends a command down that same connection and returns a Promise, because network calls are asynchronous — Redis lives on a separate process, possibly a separate machine entirely.

`SET` with the `'EX', 300` arguments tells Redis "store this value, and automatically delete it after 300 seconds have passed" — Redis handles the countdown internally, you never write a cleanup script. `TTL` asks Redis "how much time is left on this key?" — useful for debugging or showing a countdown to a user (e.g. "resend OTP in 45s").

Since Redis only understands strings (and a few other primitive structures like lists/hashes/sets), storing a JavaScript object means you must `JSON.stringify()` it before saving and `JSON.parse()` it after reading — this is a manual step you must never forget, unlike a database driver that might do this for you.

Pub/Sub works differently from get/set: a **subscriber** connection declares interest in a named "channel" (like `'order-events'`), and any **publisher** connection that calls `.publish()` on that channel instantly pushes the message to every currently-subscribed client — there is no queue, no persistence; if nobody is subscribed when you publish, the message is lost forever. This is why ioredis requires a *separate* connection object for subscribing — once a connection enters subscribe mode, it can only receive messages, not run normal commands like `GET`.

---

## Example 1 — basic

```js
// File: src/redis-basics.js
const Redis = require('ioredis'); // the Redis client library

// Connect to Redis running locally on the default port
const redisClient = new Redis({
  host: '127.0.0.1', // localhost
  port: 6379,         // Redis default port
});

async function runBasics() {
  // SET — store a simple string value under a key
  await redisClient.set('apiKey:demo', 'sk_test_12345');
  console.log('Stored apiKey:demo');

  // GET — retrieve the value back
  const apiKey = await redisClient.get('apiKey:demo');
  console.log('Retrieved value:', apiKey); // → "sk_test_12345"

  // SET with expiry — key auto-deletes after 10 seconds
  await redisClient.set('otp:9998887776', '123456', 'EX', 10);
  console.log('OTP stored, expires in 10s');

  // TTL — check how many seconds remain before expiry
  const remaining = await redisClient.ttl('otp:9998887776');
  console.log('Seconds left:', remaining); // → 10 (or slightly less)

  // EXISTS — check if a key is present (returns 1 or 0)
  const exists = await redisClient.exists('apiKey:demo');
  console.log('Key exists:', exists === 1);

  // DEL — remove a key manually
  await redisClient.del('apiKey:demo');
  const afterDelete = await redisClient.get('apiKey:demo');
  console.log('After delete:', afterDelete); // → null, key is gone

  // Always close the connection when done in a script (not needed in a long-running server)
  await redisClient.quit();
}

runBasics(); // execute the async function
```

---

## Example 2 — real world backend use case

```js
// File: src/cache/userCache.js
// A cache-aside layer for user profiles — checks Redis first,
// falls back to the database only on a cache miss, then repopulates Redis.

const Redis = require('ioredis');
const dbConnection = require('../db/dbConnection'); // pretend Postgres/Mongo wrapper

const redisClient = new Redis({
  host: process.env.REDIS_HOST || '127.0.0.1',
  port: process.env.REDIS_PORT || 6379,
});

const CACHE_TTL_SECONDS = 600; // cache user profiles for 10 minutes

async function getUserProfile(userId) {
  const cacheKey = `user:profile:${userId}`; // namespaced key — avoids collisions

  // 1. Try the cache first — this is the fast path, hit on most requests
  const cached = await redisClient.get(cacheKey);
  if (cached) {
    console.log(`[cache HIT] ${cacheKey}`);
    return JSON.parse(cached); // parse the stored JSON string back into an object
  }

  // 2. Cache miss — fall back to the real database (slow path)
  console.log(`[cache MISS] ${cacheKey} — querying database`);
  const userProfile = await dbConnection.query(
    'SELECT id, name, email FROM users WHERE id = $1',
    [userId]
  );

  if (!userProfile) return null; // user genuinely does not exist

  // 3. Repopulate the cache so the NEXT request is fast
  await redisClient.set(cacheKey, JSON.stringify(userProfile), 'EX', CACHE_TTL_SECONDS);

  return userProfile;
}

// Call this whenever the user's data changes, so stale cache is never served
async function invalidateUserCache(userId) {
  await redisClient.del(`user:profile:${userId}`);
  console.log(`[cache invalidated] user:profile:${userId}`);
}

module.exports = { getUserProfile, invalidateUserCache };

// Usage in an Express route:
// const { getUserProfile } = require('./cache/userCache');
// app.get('/api/users/:userId', async (req, res) => {
//   const userProfile = await getUserProfile(req.params.userId);
//   if (!userProfile) return res.status(404).json({ error: 'User not found' });
//   res.json(userProfile);
// });
```

---

## Common mistakes

### Mistake 1 — Storing an object directly without JSON.stringify

```js
// ❌ WRONG — ioredis silently coerces the object to the string "[object Object]"
const requestBody = { userId: 101, email: 'amit@example.com' };
await redisClient.set('session:abc123', requestBody);
const result = await redisClient.get('session:abc123');
console.log(result); // → "[object Object]" — all data is lost, unrecoverable

// ✅ CORRECT — serialize to JSON before storing, parse it back after reading
await redisClient.set('session:abc123', JSON.stringify(requestBody));
const raw = await redisClient.get('session:abc123');
const sessionData = JSON.parse(raw); // → { userId: 101, email: 'amit@example.com' }
```

### Mistake 2 — Forgetting expiry on temporary data like sessions or OTPs

```js
// ❌ WRONG — no TTL means the key lives FOREVER, slowly filling up Redis memory
// with dead sessions and expired OTPs that were never cleaned up
await redisClient.set('otp:9876543210', '482913');

// ✅ CORRECT — always attach an expiry to anything inherently temporary
await redisClient.set('otp:9876543210', '482913', 'EX', 300); // gone in 5 minutes automatically
// 'EX' takes seconds; use 'PX' for milliseconds if you need finer granularity
```

### Mistake 3 — Reusing one connection for both subscribing and normal commands

```js
// ❌ WRONG — once a connection calls .subscribe(), ioredis puts it into
// "subscriber mode" — calling .get()/.set() on it afterward throws or hangs
const redisClient = new Redis();
redisClient.subscribe('order-events');
await redisClient.get('userId:101'); // Error: Connection is in subscriber mode

// ✅ CORRECT — use a dedicated connection for subscribing, keep another for commands
const subscriber = new Redis();   // dedicated to listening only
const commandClient = new Redis(); // free to run get/set/del normally

subscriber.subscribe('order-events');
subscriber.on('message', (channel, message) => console.log(message));

await commandClient.set('userId:101', 'Amit Sharma'); // works fine, separate connection
```

---

## Practice exercises

### Exercise 1 — easy

Connect to a local Redis instance using `ioredis`. Write a script that:
1. Sets a key `apiKey:test` with the value `"sk_live_9988"`
2. Sets a second key `otp:9090909090` with value `"554433"` that expires in 15 seconds
3. Reads back and logs both values
4. Logs the TTL remaining on the OTP key
5. Deletes `apiKey:test` and confirms with `GET` that it now returns `null`

```js
// Write your code here
```

---

### Exercise 2 — medium

Build a small cache wrapper function `getOrSetCache(cacheKey, ttlSeconds, fetchFunction)` that:
1. Checks Redis for `cacheKey` — if found, parses the JSON and returns it immediately (cache hit)
2. If not found, calls `fetchFunction()` (an async function that returns some data — simulate it with a fake database call using `setTimeout`)
3. Stores the result in Redis under `cacheKey` with the given `ttlSeconds` expiry
4. Returns the freshly fetched result

Test it by calling `getOrSetCache('product:501', 30, fetchProductFromDb)` twice in a row and logging whether each call was a cache hit or miss (hint: log before returning in both branches).

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a `SessionStore` class backed by Redis that:
1. Takes an `ioredis` client instance in its constructor
2. Has a `createSession(userId, sessionData)` method that generates a random `sessionId` (use `crypto.randomUUID()`), stores `sessionData` as JSON under `session:{sessionId}` with a 1-hour expiry, and returns the `sessionId`
3. Has a `getSession(sessionId)` method that returns the parsed session data, or `null` if it does not exist or has expired
4. Has a `refreshSession(sessionId)` method that resets the TTL back to 1 hour (use `EXPIRE`) without changing the stored data — should return `false` if the session no longer exists
5. Has a `destroySession(sessionId)` method that deletes the session (for logout)
6. Also set up a subscriber using a second Redis connection that listens on a channel `'session-destroyed'`, and have `destroySession` publish the `sessionId` to that channel after deleting it, so other server instances can react (e.g. disconnect a live WebSocket tied to that session)

Test it by creating a session, fetching it, refreshing it, then destroying it and confirming a subsequent `getSession` call returns `null`.

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
CONNECTING
  const Redis = require('ioredis');
  const redisClient = new Redis({ host, port, password });
  redisClient.quit()                     → graceful close (waits for pending commands)
  redisClient.disconnect()               → immediate close (drops pending commands)

BASIC COMMANDS (all return Promises)
  redisClient.set(key, value)            → store a string
  redisClient.set(key, value, 'EX', n)   → store + expire after n SECONDS
  redisClient.set(key, value, 'PX', n)   → store + expire after n MILLISECONDS
  redisClient.get(key)                   → returns string or null if missing
  redisClient.del(key)                   → delete a key, returns count deleted
  redisClient.exists(key)                → returns 1 (present) or 0 (absent)
  redisClient.expire(key, seconds)       → set/reset TTL on an existing key
  redisClient.ttl(key)                   → seconds left; -1 = no expiry, -2 = key gone

STORING OBJECTS
  Redis only stores strings/binary — you must:
    SET:  JSON.stringify(obj)
    GET:  JSON.parse(raw)     — check raw is not null before parsing!

PUB/SUB
  const subscriber = new Redis();        → dedicated connection, subscribe-only mode
  subscriber.subscribe(channelName);
  subscriber.on('message', (channel, message) => { ... });
  publisher.publish(channelName, message); → fire-and-forget, no persistence, no history
  NEVER reuse a subscribed connection to run normal commands

WHEN TO USE REDIS
  Cache      → expensive DB queries, computed results, rendered pages (short TTL)
  Sessions   → login state shared across multiple server instances (Topic 58)
  Rate limit → per-user/IP request counters with expiry (Topic 70)
  Pub/Sub    → fan-out notifications across processes (Topic 101)
  Queues     → job/task processing via BullMQ (Topic 100)

GOTCHAS
  - Redis is in-memory: a server restart wipes data unless persistence (RDB/AOF) is enabled
  - Always set a TTL on cache/session/OTP keys — never let temporary data live forever
  - Namespace keys with colons (user:profile:101) to avoid collisions and enable pattern scans
  - Never use redisClient.keys('*') in production — it blocks the server; use SCAN instead
```

---

## Connected topics

- **69 — Caching strategies** — the cache-aside pattern, TTL tuning, and cache invalidation shown in Example 2 in full depth
- **58 — Authentication patterns — Sessions** — using Redis as the session store behind `express-session` instead of storing sessions in memory
- **101 — Pub/Sub with Redis** — a deeper dive into publish/subscribe channels, fan-out patterns, and multi-instance coordination introduced in this doc
