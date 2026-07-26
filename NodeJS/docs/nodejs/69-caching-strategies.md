# 69 — Caching strategies

## What is this?

Caching means storing the result of an expensive operation — a database query, an API call, a computed value — somewhere fast so the next request can skip redoing the work. It's like a restaurant kitchen prepping sauces in the morning instead of cooking one from scratch for every single order — the "expensive" step happens once, and every subsequent order just grabs the pre-made result. In Node.js backends, caching usually means storing data either **in-memory** (inside the running process, using something like `node-cache`) or in an **external store like Redis** (shared across multiple server instances).

## Why does it matter for backend development?

Databases are almost always the slowest part of a request — a query that takes 50ms feels instant to a human but is an eternity compared to reading the same value from memory (sub-millisecond). When the same data (a user profile, a product listing, a config value) is requested thousands of times per minute, hitting the database every single time wastes CPU, connections, and money. Caching lets a backend serve repeated reads almost instantly, survive traffic spikes without crushing the database, and reduce load on downstream APIs that may charge per request or rate-limit you. Every production backend that scales beyond a toy project uses some form of caching.

---

## Syntax / API

```js
// ── In-memory caching with node-cache ──────────────────────────────────────
// Install: npm install node-cache
const NodeCache = require('node-cache');

// stdTTL = default time-to-live in seconds for every key (0 = never expire)
// checkperiod = how often (seconds) the cache scans for expired keys and deletes them
const memoryCache = new NodeCache({ stdTTL: 60, checkperiod: 30 });

memoryCache.set('userId:101', { name: 'Asha' });   // store a value under a key, uses stdTTL
memoryCache.set('userId:102', { name: 'Ravi' }, 120); // override TTL for this one key (120s)

const cachedUser = memoryCache.get('userId:101');  // returns value, or undefined if missing/expired
memoryCache.del('userId:101');                     // manually remove a key (invalidate it)
memoryCache.flushAll();                            // wipe the entire cache

// ── Redis caching (shared cache across multiple Node processes) ───────────
// Install: npm install ioredis
const Redis = require('ioredis');
const redisClient = new Redis(process.env.REDIS_URL || 'redis://127.0.0.1:6379');

// SET with EX sets a value AND an expiry (seconds) in one atomic command
await redisClient.set('userId:101', JSON.stringify({ name: 'Asha' }), 'EX', 60);

const raw = await redisClient.get('userId:101');   // returns a string or null
const user = raw ? JSON.parse(raw) : null;         // Redis only stores strings — parse JSON yourself

await redisClient.del('userId:101');               // manually invalidate a key
```

---

## How it works — line by line

`node-cache` keeps a plain JavaScript object in memory inside your Node process. `set()` stores a key/value pair and, behind the scenes, records a timestamp for when that key should expire. A background timer (`checkperiod`) periodically sweeps the store and deletes anything past its TTL, so `get()` on an expired key simply returns nothing — as if it was never there.

Redis works the same way conceptually but lives in a **separate process**, often on a separate machine. Every read/write is a network call (fast, usually under 1ms on a local network, but still a round trip). Because Redis is external, every one of your Node server instances — even ten copies running behind a load balancer — sees the exact same cached data. `node-cache`, by contrast, is per-process: if you run three Node instances, each has its own independent memory cache with no idea what the others cached.

TTL (time-to-live) is the core safety mechanism in both: it guarantees that even if you forget to invalidate a key, stale data eventually disappears on its own instead of lingering forever.

---

## Example 1 — basic

```js
// File: cache/memoryCache.js
const NodeCache = require('node-cache');

// Create a cache: values expire after 30 seconds by default
const memoryCache = new NodeCache({ stdTTL: 30 });

function getExpensiveValue() {
  console.log('Running expensive computation...');   // simulate slow work
  return { total: 42, computedAt: Date.now() };        // pretend this took a long time
}

function getCachedValue(cacheKey) {
  const cached = memoryCache.get(cacheKey);            // try the cache first
  if (cached !== undefined) {
    console.log('Cache HIT');                          // found it — no recomputation needed
    return cached;
  }

  console.log('Cache MISS');                           // not found or expired
  const freshValue = getExpensiveValue();               // do the expensive work
  memoryCache.set(cacheKey, freshValue);                // store it for next time
  return freshValue;
}

getCachedValue('report:daily');   // logs "Cache MISS", then "Running expensive computation..."
getCachedValue('report:daily');   // logs "Cache HIT" — instant, no recomputation
```

---

## Example 2 — real world backend use case

```js
// File: services/userService.js
// The "cache-aside" pattern: the APPLICATION checks the cache first,
// and on a miss, reads the database and populates the cache itself.
// Redis is used here because this API runs on multiple server instances.

const Redis = require('ioredis');
const redisClient = new Redis(process.env.REDIS_URL);

const USER_CACHE_TTL = 300; // seconds — 5 minutes, tune based on how often user data changes

// Simulates a real database call — replace with your actual query (pg, mongoose, etc.)
async function fetchUserFromDb(userId) {
  console.log(`[DB] Querying user ${userId}...`);
  // e.g. return dbConnection.query('SELECT * FROM users WHERE id = $1', [userId]);
  return { userId, name: 'Priya Sharma', email: 'priya@example.com' };
}

async function getUserById(userId) {
  const cacheKey = `user:${userId}`;                     // namespaced key prevents collisions

  const cachedRaw = await redisClient.get(cacheKey);      // 1. check cache first (cache-aside)
  if (cachedRaw) {
    console.log('[Cache] HIT for', cacheKey);
    return JSON.parse(cachedRaw);                         // Redis stores strings — parse back to object
  }

  console.log('[Cache] MISS for', cacheKey);
  const user = await fetchUserFromDb(userId);             // 2. fall back to the source of truth (DB)

  // 3. populate the cache so the NEXT request is fast — SET with EX for automatic expiry
  await redisClient.set(cacheKey, JSON.stringify(user), 'EX', USER_CACHE_TTL);

  return user;
}

// Invalidate the cache whenever the underlying data changes — critical for correctness
async function updateUserEmail(userId, newEmail) {
  // e.g. await dbConnection.query('UPDATE users SET email = $1 WHERE id = $2', [newEmail, userId]);
  console.log(`[DB] Updated email for user ${userId}`);

  // Delete the stale cached entry so the next getUserById() call re-reads the fresh row
  await redisClient.del(`user:${userId}`);
}

module.exports = { getUserById, updateUserEmail };
```

---

## Common mistakes

### Mistake 1 — Never invalidating the cache after a write

```js
// ❌ WRONG — update the database but leave the old cached value in place
async function updateUserName(userId, newName) {
  await dbConnection.query('UPDATE users SET name = $1 WHERE id = $2', [newName, userId]);
  // Cache still has the OLD name — every read for the next 5 minutes returns stale data
}

// ✅ CORRECT — always invalidate (or update) the cache in the same operation as the write
async function updateUserName(userId, newName) {
  await dbConnection.query('UPDATE users SET name = $1 WHERE id = $2', [newName, userId]);
  await redisClient.del(`user:${userId}`);   // force the next read to be a cache MISS and refetch
}
```

### Mistake 2 — Caching data with no TTL at all

```js
// ❌ WRONG — no expiry means a bug in your invalidation logic causes PERMANENTLY stale data
memoryCache.set('productList', products);        // no TTL passed — never expires with stdTTL: 0
await redisClient.set('productList', JSON.stringify(products));  // no EX — lives forever in Redis

// ✅ CORRECT — always set a TTL as a safety net, even if you also invalidate manually
memoryCache.set('productList', products, 120);                    // expires in 2 minutes regardless
await redisClient.set('productList', JSON.stringify(products), 'EX', 120); // same safety net in Redis
```

### Mistake 3 — Caching data that is different per user, under one shared key

```js
// ❌ WRONG — one cache key shared across every user returns someone ELSE's private data
async function getDashboard(userId) {
  const cached = await redisClient.get('dashboard');   // same key no matter who asks!
  if (cached) return JSON.parse(cached);
  const data = await buildDashboard(userId);
  await redisClient.set('dashboard', JSON.stringify(data), 'EX', 60);
  return data;   // user B gets user A's cached dashboard — a serious data leak
}

// ✅ CORRECT — namespace the cache key with the identifying value (userId, requestId, etc.)
async function getDashboard(userId) {
  const cacheKey = `dashboard:${userId}`;              // unique per user
  const cached = await redisClient.get(cacheKey);
  if (cached) return JSON.parse(cached);
  const data = await buildDashboard(userId);
  await redisClient.set(cacheKey, JSON.stringify(data), 'EX', 60);
  return data;
}
```

---

## Practice exercises

### Exercise 1 — easy

Using `node-cache`, write a function `getProductPrice(productId)` that:
1. Checks an in-memory cache for a key like `price:{productId}`
2. On a cache miss, calls a fake `fetchPriceFromDb(productId)` function (just return a hardcoded number after logging `"Fetching from DB..."`) and stores the result with a TTL of 10 seconds
3. On a cache hit, logs `"Served from cache"` and returns the cached value
4. Call it twice in a row for the same `productId` and confirm the second call hits the cache (check the console output)

```js
// Write your code here
```

---

### Exercise 2 — medium

Using `ioredis`, build a small cache-aside layer for a `getOrderById(orderId)` function:
1. Cache key format: `order:{orderId}`, TTL of 60 seconds
2. On a miss, simulate a DB call (`fetchOrderFromDb`) that returns an object like `{ orderId, status: 'shipped', total: 499 }`
3. Store the JSON-stringified result in Redis with `EX`
4. Write a second function `cancelOrder(orderId)` that simulates updating the DB status to `'cancelled'` and then **invalidates** the cached entry so the next `getOrderById` call reflects the change
5. Demonstrate the flow: call `getOrderById` (miss), call it again (hit), call `cancelOrder`, then call `getOrderById` again and confirm it's a fresh miss

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a small reusable `CacheService` class that supports **two backends** (in-memory via `node-cache`, and Redis via `ioredis`) behind one interface, chosen by a constructor flag:
1. Constructor: `new CacheService({ backend: 'memory' | 'redis', ttlSeconds })`
2. `async get(key)` — returns the parsed value or `null` if missing/expired (works the same regardless of backend)
3. `async set(key, value)` — stringifies and stores the value with the configured TTL
4. `async invalidate(key)` — deletes the key from whichever backend is active
5. `async getOrSet(key, fetchFn)` — implements the cache-aside pattern generically: checks cache, and on a miss calls `fetchFn()` (an async function), stores the result, and returns it
6. Test it with both `backend: 'memory'` and `backend: 'redis'` using the same `getOrSet('userId:501', fetchUserFromDb)` call and confirm both report a MISS then a HIT

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
CACHING STRATEGY OVERVIEW
  In-memory (node-cache)  → fastest, per-process only, lost on restart, no cross-instance sharing
  Redis (external)        → slightly slower (network hop), SHARED across all app instances, survives app restarts

NODE-CACHE BASICS
  new NodeCache({ stdTTL: seconds, checkperiod: seconds })
  .set(key, value, ttlOverride?)   → store, optional per-key TTL
  .get(key)                       → value, or undefined if missing/expired
  .del(key)                       → remove one key
  .flushAll()                     → wipe everything

REDIS (ioredis) BASICS
  .set(key, value, 'EX', seconds) → store as a string WITH expiry, atomically
  .get(key)                       → string or null — JSON.parse() it yourself
  .del(key)                       → remove one key
  Redis only stores strings — always JSON.stringify() on write, JSON.parse() on read

CACHE-ASIDE PATTERN (the standard approach)
  1. App checks cache for the key
  2. HIT  → return cached value immediately
  3. MISS → read from the real source (DB/API), store result in cache, THEN return it
  App owns the caching logic — the database knows nothing about the cache

TTL (TIME-TO-LIVE)
  Always set one — it's your safety net against forgotten invalidation
  Short TTL  (seconds–minutes) → fresher data, more DB hits
  Long TTL   (hours+)          → fewer DB hits, higher risk of staleness
  Pick TTL based on how often the underlying data actually changes

CACHE INVALIDATION
  Delete (or update) the cache key in the SAME operation as any write/update to the source data
  "There are only two hard things in computer science: cache invalidation and naming things"
  Always namespace keys per entity: user:{id}, order:{id} — never one shared key for per-user data

GOTCHAS
  Stale data          → missing invalidation after writes
  Data leak           → shared cache key across different users/requests
  Unbounded memory     → node-cache with no TTL growing forever inside your process
  Cache stampede      → many requests miss at once and hammer the DB simultaneously (needs locking/queueing)
```

---

## Connected topics

- **92 — Connecting to Redis** — the ioredis client setup, connection lifecycle, and pub/sub basics that back Redis caching in depth
- **70 — Rate limiting** — Redis-backed rate limiting reuses the exact same TTL and key-expiry mechanics as caching
- **68 — Memory management** — an unbounded `node-cache` instance with no TTL is a classic in-process memory leak
