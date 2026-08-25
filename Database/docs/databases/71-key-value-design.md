# 71 — Key-Value Design
## Phase: Beyond Relational

---

## ELI5 — The Simple Analogy

A cloakroom at a theatre.

You hand over your coat, you get a numbered ticket. Later you present the ticket and get the coat back. **That is the entire interface: `PUT(ticket, coat)` and `GET(ticket)`.**

It is astonishingly fast, because the attendant does exactly one thing: walk to the numbered peg. No searching, no thinking, no reading labels.

★ **And it is astonishingly limited, in a way people underestimate.** Ask *"which coats are blue?"* and the attendant must inspect all 4,000. Ask *"how many coats did we take tonight?"* and there is no answer — nobody counted. Ask *"give me coats 400 through 450"* and the pegs aren't in any order you can walk.

Three consequences follow, and they are the whole topic:

1. ★ **The key is the only way in.** So the key must *be* the question. If you'll ask "what did user 42 do on Tuesday?", the key had better contain both.
2. ★ **You cannot query, so you must plan every access pattern in advance** — and adding one later usually means writing the data a second time under a different key.
3. ★ **Everything you'd get for free in SQL — counts, ranges, joins, constraints — is now your code**, and your code is now responsible for keeping duplicates in agreement.

---

## Where this fits in the big picture

```
   57 caching — ★ Redis as a cache, with a TTL
   70 documents — nesting as the primary shape
                          │
                          ▼
        ┌──────────────────────────────────────────────┐
        │ 71 KEY-VALUE DESIGN ← YOU ARE HERE           │
        │ ★ when the KEY is the entire query language  │
        └────────────────────┬─────────────────────────┘
                             ▼
              72 wide-column · 73 time-series · 75 SQL vs NoSQL
```

★ **Topic 57 treated Redis as a cache — a copy with a TTL, where being wrong is survivable.** This topic treats a key-value store as a **system of record**, where it isn't. The design discipline is completely different, and conflating the two is the most common mistake here.

---

## What is this?

A store whose entire interface is `GET(key)`, `PUT(key, value)`, `DELETE(key)`, where the value is opaque.

```
 ★ THE FAMILY, AND THEY ARE NOT INTERCHANGEABLE:

 ★ REDIS         in-memory, ★ rich value types (lists, sets, sorted
                 sets, hashes, streams), ★ single-threaded, optional
                 persistence
 ★ MEMCACHED     in-memory, ★ opaque values only, multi-threaded,
                 ★ no persistence, ★ no replication
 ★ DYNAMODB      ★ disk, ★ partition key + sort key ⇒ RANGE queries,
                 managed, ★ pay per request
 ★ ETCD / CONSUL ★ small, ★ strongly consistent (Raft), for
                 configuration and coordination — ★ NOT for data
 ★ ROCKSDB/LMDB  ★ embedded LSM/B-tree libraries — the engine
                 inside many of the above (Topic 08)

 ⇒ ★ "KEY-VALUE STORE" SPANS A 1,000× RANGE IN DURABILITY,
   CONSISTENCY AND COST. Naming the specific one is the first
   step of any design conversation.
```

★ **And the reframe that matters:** a key-value store with a *sort key* (DynamoDB, or Redis sorted sets) is not really key-value — **it is a single, very fast index that you get to design by hand.** That is where most of the engineering lives.

---

## Why does it matter for a backend developer?

```
 ★ THREE REASONS.

 ① ★ IT IS THE FASTEST THING AVAILABLE, AND THE LEAST FORGIVING.
    a Redis GET is ~0.2 ms and a DynamoDB GetItem is ~5 ms at
    ★ ANY SCALE — 1 KB or 10 TB, the same.
    ⇒ ★ but a query you did not design a key for is O(n) or
      impossible.

 ② ★ THE KEY SCHEMA IS AS PERMANENT AS A SHARD KEY (Topic 60).
    changing it means rewriting every record and every read path.
    ⇒ ★ and unlike SQL, there is no "just add an index" escape.

 ③ ★ PEOPLE USE IT AS A DATABASE WITH CACHE HABITS.
    ★ no TTL on data that should expire ⇒ unbounded memory
    ★ TTL on data that shouldn't ⇒ ★ silent data loss
    ★ no persistence configured ⇒ ★ a restart is a wipe
    ⇒ ★ Redis defaults are CACHE defaults, and using it as a
      system of record requires changing several of them.
```

---

## The physical reality

### Redis: single-threaded, and what that actually means

```
 ★ REDIS EXECUTES COMMANDS ON ONE THREAD. (I/O is threaded in 6+;
   ★ command execution is not.)

 ⇒ CONSEQUENCE ①: ★ EVERY COMMAND IS ATOMIC, for free.
   INCR, LPUSH, SETNX — no locks needed, ever.
 ⇒ CONSEQUENCE ②: ★ ONE SLOW COMMAND BLOCKS EVERYTHING.
   ★ KEYS *          on 10M keys ⇒ ★ 4,200 ms of total blockage
   ★ SMEMBERS        on a 1M-member set ⇒ ★ 180 ms
   ★ DEL of a 5 GB key ⇒ ★ 1,400 ms  (use ★ UNLINK — async free)
   ★ FLUSHALL        ⇒ blocks
   ⇒ ★ THIS IS THE #1 REDIS PRODUCTION INCIDENT: an O(n) command
     on a large key, run by a well-meaning script.
 ⇒ CONSEQUENCE ③: ★ ONE CORE IS THE CEILING — ~100k–200k ops/sec
   per instance. Scaling means ★ more instances (Cluster), not a
   bigger machine.

 ★ THE SAFE ALTERNATIVES, WHICH EVERY TEAM SHOULD KNOW:
   KEYS *      ⇒ ★ SCAN (cursor-based, incremental)
   SMEMBERS    ⇒ ★ SSCAN
   HGETALL     ⇒ ★ HSCAN, or restructure
   DEL         ⇒ ★ UNLINK
   ⇒ ★ AND: `redis-cli --bigkeys` finds them before they find you.
```

### Redis persistence — the setting people never look at

```
 ★ THREE MODES, AND THE DEFAULT IS NOT WHAT YOU WANT FOR DATA.

 ★ ① RDB (snapshots)  — the default
    save 900 1 / save 300 10 / save 60 10000
    ⇒ ★ a fork() + a point-in-time dump.
    ⇒ ★ YOU LOSE EVERYTHING SINCE THE LAST SNAPSHOT — up to 15
      minutes with the default settings.
    ⇒ ★ AND fork() ON A 20 GB INSTANCE BRIEFLY DOUBLES MEMORY
      (copy-on-write) ⇒ ★ a classic OOM-kill cause.

 ★ ② AOF (append-only file) — a log of every write
    appendfsync everysec   ⇒ ★ lose up to 1 second  ← the sane default
    appendfsync always     ⇒ ★ lose nothing, ★ ~10× slower
    appendfsync no         ⇒ ★ OS-buffered; lose whatever it holds
    ⇒ ★ AOF rewrite also forks.

 ★ ③ BOTH — ★ what you want for a system of record
    appendonly yes + appendfsync everysec + RDB for fast restarts

 ⇒ ★ AND THE ONE THAT SILENTLY DESTROYS DATA:
   ★ maxmemory-policy
     noeviction        ⇒ writes ERROR when full  ← ★ for DATA
     allkeys-lru       ⇒ ★ EVICTS ANY KEY, including your data
     volatile-lru      ⇒ evicts only keys WITH a TTL  ← ★ safer
   ⇒ ★ `allkeys-lru` ON A SYSTEM OF RECORD IS SILENT DATA LOSS,
     and it is the setting most tutorials recommend.
```

### DynamoDB: the partition key decides everything

```
 ★ THE DATA MODEL IS TWO KEYS, AND THEY DO DIFFERENT JOBS:
   ★ PARTITION KEY (PK)  ⇒ hashed ⇒ chooses the physical partition
   ★ SORT KEY (SK)       ⇒ orders items WITHIN a partition
                         ⇒ ★ this is what enables RANGE queries

 ⇒ ★ Query() can do: PK = x AND SK BETWEEN a AND b
 ⇒ ★ Scan() reads EVERYTHING. ★ It is never the answer at scale.

 ★ THE THREE HARD LIMITS THAT SHAPE EVERY DESIGN:
 ① ★ 400 KB PER ITEM — a hard cap. No exceptions.
 ② ★ 10 GB PER PARTITION (for a table with a local secondary index)
 ③ ★ ~3,000 read / 1,000 write units per PARTITION per second
    ⇒ ★ THE HOT PARTITION PROBLEM: one popular PK saturates one
      partition regardless of total table capacity.
    ⇒ ★ THIS IS TOPIC 61'S HOT ROW, AT THE STORAGE-ENGINE LEVEL.
    ⇒ FIX: ★ write sharding — append a suffix to the PK
      `user#42#3` where 3 = random(0..N) ⇒ ★ reads must then
      query all N shards and merge.

 ★ AND THE COST MODEL, WHICH IS A DESIGN CONSTRAINT:
   ★ 1 RCU = 1 strongly-consistent read of ≤4 KB
           = ★ 2 eventually-consistent reads
   ★ 1 WCU = 1 write of ≤1 KB
   ⇒ ★ A 400 KB ITEM COSTS 400 WCU TO WRITE. Item size is money.
   ⇒ ★ AND: a Query returning 100 items of 3 KB costs 75 RCU,
     ★ even if you only wanted one field. Projection matters.
```

### Single-table design — the pattern that looks insane and isn't

```
 ★ DYNAMODB'S CANONICAL PATTERN: ★ PUT EVERY ENTITY TYPE IN ONE
   TABLE, with generic key names.

   PK              SK                  type      data…
   ──────────────  ──────────────────  ────────  ─────────────
   USER#42         PROFILE             user      name, email
   USER#42         ORDER#8842119       order     total, status
   USER#42         ORDER#8842120       order     total, status
   USER#42         ADDRESS#1           address   line1, city
   ORDER#8842119   ITEM#1              item      sku, qty
   ORDER#8842119   ITEM#2              item      sku, qty

 ⇒ ★ WHY: `Query(PK = "USER#42")` returns the profile, ALL orders
   AND all addresses ★ IN ONE REQUEST — because they share a
   partition and the sort key orders them.
 ⇒ ★ THIS IS THE JOIN, PRECOMPUTED INTO THE KEY LAYOUT.
   You are not avoiding the join; you are performing it at write
   time by choosing where things live.

 ★ AND `SK BEGINS_WITH 'ORDER#'` GIVES YOU "JUST THE ORDERS" —
   ★ a range scan within the partition, one request.

 ★ THE COST:
   ✗ ★ the key names are meaningless (`PK`, `SK`, `GSI1PK`)
   ✗ ★ the schema lives in documentation, not in the database
   ✗ ★ a new access pattern often needs a new GSI or a backfill
   ⇒ ★ IT IS A TRADE OF READABILITY FOR REQUEST COUNT, and it is
     correct only when the request count actually matters.
```

### Redis data structures — the reason to choose Redis at all

```
 ★ IF YOU ONLY USE GET/SET, ★ USE MEMCACHED — it is multi-threaded
   and simpler. ★ Redis earns its place through its VALUE TYPES.

 ★ SORTED SET (ZSET) — ★ the most valuable structure
   score-ordered, O(log n) insert, O(log n + m) range
   ⇒ ★ leaderboards: ZADD / ZREVRANGE / ★ ZRANK (a player's rank
     in O(log n) — ★ impossible to do cheaply in SQL)
   ⇒ ★ time-ordered feeds: score = timestamp
   ⇒ ★ rate limiting: a sliding window via ZREMRANGEBYSCORE
   ⇒ ★ priority queues: ZPOPMIN

 ★ HASH — a map per key
   ⇒ ★ update one field without rewriting the value (unlike a
     serialised JSON string — Topic 70's write amplification)
   ⇒ ★ and small hashes use a memory-efficient encoding

 ★ LIST — LPUSH/BRPOP ⇒ a simple queue
   ⇒ ★ but ★ BRPOPLPUSH / LMOVE for a RELIABLE queue (below)

 ★ SET — membership, ★ SINTER/SUNION for tag intersections
 ★ HYPERLOGLOG — ★ approximate distinct count in 12 KB, ★ 0.81%
   error, for ANY cardinality
   ⇒ ★ unique visitors per day: 12 KB instead of a set of 40M ids
 ★ BITMAP — ★ 1 bit per user; daily actives for 10M users = 1.2 MB
 ★ STREAM — ★ an append-only log with consumer groups; the
   durable-queue answer

 ⇒ ★ CHOOSING THE RIGHT STRUCTURE IS THE ENTIRE OPTIMISATION.
   A leaderboard as a ZSET is O(log n); as a serialised list it
   is O(n) and rewrites everything.
```

### Memory: where it actually goes

```
 ★ REDIS OVERHEAD PER KEY IS ~50–100 BYTES, BEFORE THE VALUE.
   ⇒ ★ 10 million keys of 20 bytes each = 200 MB of data and
     ★ ~700 MB of overhead.
   ⇒ ★ KEY NAMES MATTER: "user:profile:12345" × 10M = 190 MB of
     key names alone.

 ★ THE THREE MEMORY LEVERS:
 ① ★ SMALL AGGREGATES USE COMPACT ENCODINGS
    hash-max-listpack-entries 128 / hash-max-listpack-value 64
    zset-max-listpack-entries 128
    ⇒ ★ under these thresholds, a hash is stored as a flat array —
      ★ 5–10× less memory.
    ⇒ ★ SO: 1,000 hashes of 100 fields beats 100,000 separate keys,
      by an order of magnitude.
 ② ★ SHORTER KEYS AND FIELD NAMES
 ③ ★ TTLs on anything that can expire

 ★ AND THE MEASUREMENT:
   MEMORY USAGE <key>          -- ★ bytes for one key
   redis-cli --bigkeys         -- ★ finds the outliers
   INFO memory                 -- ★ used_memory vs maxmemory
   ★ mem_fragmentation_ratio > 1.5 ⇒ consider activedefrag
```

---

## How it works — step by step

### Designing the key schema

```
 ★ THE PROCESS IS THE OPPOSITE OF RELATIONAL DESIGN.
   You do not model entities. ★ YOU ENUMERATE ACCESS PATTERNS AND
   DESIGN KEYS THAT SERVE THEM.

 ★ ① WRITE DOWN EVERY ACCESS PATTERN, VERBATIM
    "get a user's profile"
    "get a user's 20 most recent orders"
    "get one order with its items"
    "list all orders for a seller, newest first"
    ⇒ ★ IF YOU CANNOT ENUMERATE THEM, YOU ARE NOT READY.
      This is not optional preparation — ★ it IS the design.

 ★ ② FOR EACH, WRITE THE KEY THAT ANSWERS IT IN ONE OPERATION
    user profile        ⇒ PK=USER#42, SK=PROFILE
    a user's orders     ⇒ PK=USER#42, SK begins_with ORDER#
    ★ newest first      ⇒ ★ SK=ORDER#<inverted timestamp>#<id>
                          (DynamoDB sorts ascending; to get
                          descending cheaply, ★ invert the value
                          or use ScanIndexForward=false)
    an order + items    ⇒ PK=ORDER#8842119, SK begins_with ITEM#

 ★ ③ ANY PATTERN NOT SERVED ⇒ A SECONDARY INDEX OR A SECOND WRITE
    "orders for a seller" ⇒ ★ GSI with PK=SELLER#7,
                              SK=<timestamp>#<order id>
    ⇒ ★ A GSI IS A SECOND COPY, ★ maintained asynchronously, and
      ★ it costs its own write units.

 ★ ④ CHECK EVERY KEY FOR HOTNESS
    ⇒ ★ what is the highest-traffic single PK value?
    ⇒ ★ >1,000 writes/sec on one PK ⇒ write-shard it.
```

### Redis patterns worth knowing precisely

```js
// ★ ① A DISTRIBUTED LOCK — and why the naive version is broken
// ✗ BROKEN: another process can delete YOUR lock
await redis.set(key, '1', 'NX', 'EX', 10);
// … work …
await redis.del(key);          // ★ if your work took >10s, the
                               //   lock expired and someone ELSE
                               //   holds it. You just deleted THEIRS.

// ✓ CORRECT: a unique token, released with a Lua compare-and-delete
const token = crypto.randomUUID();
const got = await redis.set(key, token, 'NX', 'PX', 10000);
if (!got) throw new Error('LOCKED');
try { await doWork(); }
finally {
  await redis.eval(`
    if redis.call('get', KEYS[1]) == ARGV[1]
      then return redis.call('del', KEYS[1]) else return 0 end`,
    1, key, token);            // ★ atomic check-and-delete
}
// ★ AND THE HONEST CAVEAT: this is NOT safe for correctness under
//   arbitrary failures (GC pauses, clock skew). ★ Use it for
//   efficiency ("don't do this work twice"), ★ NOT for safety
//   ("only one process may charge this card") — for that, use a
//   database constraint (Topic 52).
```

```js
// ★ ② A SLIDING-WINDOW RATE LIMITER — one round trip, atomic
const script = `
  local key, now, window, limit = KEYS[1], tonumber(ARGV[1]),
                                  tonumber(ARGV[2]), tonumber(ARGV[3])
  redis.call('ZREMRANGEBYSCORE', key, 0, now - window)   -- ★ evict old
  local count = redis.call('ZCARD', key)
  if count < limit then
    redis.call('ZADD', key, now, ARGV[4])
    redis.call('PEXPIRE', key, window)                    -- ★ self-cleaning
    return {1, limit - count - 1}
  end
  return {0, 0}`;
const [allowed, remaining] = await redis.eval(
  script, 1, `rl:${userId}`, Date.now(), 60000, 100, crypto.randomUUID());
// ★ EXACT sliding window, atomic, one round trip.
// ★ COST: O(log n) per request and one ZSET member per request in
//   the window — bounded by `limit`, so it cannot grow unboundedly.
```

```js
// ★ ③ A RELIABLE QUEUE — LIST alone loses messages
// ✗ BRPOP: if the worker crashes after popping, ★ the message is gone.
// ✓ LMOVE into a processing list, then remove on success
const job = await redis.lmove('q:pending', 'q:processing', 'RIGHT', 'LEFT');
try {
  await handle(job);
  await redis.lrem('q:processing', 1, job);      // ★ acknowledge
} catch (e) {
  await redis.lmove('q:processing', 'q:pending', 'RIGHT', 'LEFT');  // requeue
}
// ★ plus a reaper for items stuck in q:processing.
// ⇒ ★ OR USE STREAMS, which have consumer groups and
//   acknowledgement built in (XADD / XREADGROUP / XACK / XAUTOCLAIM).
```

```js
// ★ ④ HYPERLOGLOG — 40M unique visitors in 12 KB
await redis.pfadd(`uv:${day}`, userId);
const unique = await redis.pfcount(`uv:${day}`);     // ★ ±0.81%
await redis.pfmerge('uv:week', ...days);             // ★ union, still 12 KB
// ⇒ ★ a SET of 40M ids would be ~1.6 GB.
```

### DynamoDB single-table design, worked

```js
// ★ THE ACCESS PATTERNS, WRITTEN FIRST
// 1. get a user's profile
// 2. get a user's orders, newest first, paginated
// 3. get one order with all its items
// 4. list a seller's orders in a date range
// 5. find an order by its external reference

const table = 'app';

// ★ ITEM SHAPES — generic key names, a `type` discriminator
// { PK: 'USER#42',       SK: 'PROFILE',                type: 'user'  }
// { PK: 'USER#42',       SK: 'ORDER#9223370...#8842119', type:'order',
//   GSI1PK: 'SELLER#7',  GSI1SK: '2026-08-25T09:14Z#8842119',
//   GSI2PK: 'EXTREF#abc-123', GSI2SK: 'ORDER' }
// { PK: 'ORDER#8842119', SK: 'ITEM#001',               type: 'item'  }

// ★ pattern 2 — one Query, newest first
const orders = await ddb.query({
  TableName: table,
  KeyConditionExpression: 'PK = :pk AND begins_with(SK, :prefix)',
  ExpressionAttributeValues: { ':pk': 'USER#42', ':prefix': 'ORDER#' },
  ★ ScanIndexForward: false,     // ★ descending — no inverted key needed
  Limit: 20,
});

// ★ pattern 3 — the order AND its items in ONE request
const order = await ddb.query({
  TableName: table,
  KeyConditionExpression: 'PK = :pk',
  ExpressionAttributeValues: { ':pk': 'ORDER#8842119' },
});
// ⇒ returns the order header (SK='META') and every ITEM#nnn,
//   ★ sorted, in one round trip. ★ This is the join.

// ★ pattern 4 — a GSI for the seller's view
const sellerOrders = await ddb.query({
  TableName: table, ★ IndexName: 'GSI1',
  KeyConditionExpression: 'GSI1PK = :s AND GSI1SK BETWEEN :from AND :to',
  ExpressionAttributeValues: { ':s':'SELLER#7', ':from':'2026-08-01',
                               ':to':'2026-08-31' },
});
```

```js
// ★ WRITING AN ORDER AND ITS ITEMS ATOMICALLY
await ddb.transactWriteItems({ TransactItems: [
  { Put: { TableName: table, Item: orderItem,
           ★ ConditionExpression: 'attribute_not_exists(PK)' } },  // idempotent
  ...items.map(i => ({ Put: { TableName: table, Item: i } })),
]});
// ★ LIMITS: 100 items, 4 MB total, and ★ transactions cost 2× the
//   write units. They are not free, and they are not a substitute
//   for designing the aggregate boundary correctly (Topic 70).
```

---

## Concept breakdown

```
★ THE INTERFACE IS THE CONSTRAINT
   GET / PUT / DELETE by key. ★ The key IS the query language.
   ⇒ ★ every access pattern must be enumerated IN ADVANCE
   ⇒ ★ a pattern you didn't plan for is O(n) or impossible

★ THE FAMILY SPANS 1,000× IN GUARANTEES
   Redis (in-mem, ★ rich types) · Memcached (★ no persistence,
   no replication) · DynamoDB (★ PK+SK ⇒ ranges) ·
   etcd (★ Raft, config only) · RocksDB/LMDB (embedded engines)
   ⇒ ★ name the specific one before designing

★ REDIS IS SINGLE-THREADED
   ✓ ★ every command is atomic for free
   ✗ ★ ONE O(n) COMMAND BLOCKS EVERYTHING
     KEYS* 4,200 ms · SMEMBERS 180 ms · DEL of 5 GB 1,400 ms
     ⇒ ★ SCAN / SSCAN / HSCAN / UNLINK, and `--bigkeys`
   ✗ ★ one core is the ceiling ⇒ scale OUT, not up

★ REDIS AS A SYSTEM OF RECORD NEEDS DIFFERENT DEFAULTS
   ★ appendonly yes + appendfsync everysec (+ RDB for restarts)
   ★ maxmemory-policy: ★ noeviction or volatile-lru
     ⇒ ★ allkeys-lru ON DATA IS SILENT DATA LOSS
   ★ RDB/AOF-rewrite fork() briefly DOUBLES memory ⇒ OOM risk

★ DYNAMODB'S THREE LIMITS SHAPE EVERY DESIGN
   ★ 400 KB per item · ★ 10 GB per partition ·
   ★ ~3,000 RCU / 1,000 WCU per PARTITION
   ⇒ ★ the HOT PARTITION is Topic 61's hot row at the storage level
   ⇒ fix: ★ write-shard the PK, read all shards and merge
   ★ COST: item size IS money (400 KB = 400 WCU)

★ SINGLE-TABLE DESIGN
   every entity in one table, generic keys, a `type` field
   ⇒ ★ Query(PK) returns the parent AND children in ONE request
   ⇒ ★ THE JOIN, PRECOMPUTED INTO THE KEY LAYOUT
   ✗ ★ unreadable keys, schema in docs, new patterns need a GSI

★ REDIS DATA STRUCTURES ARE THE REASON TO CHOOSE REDIS
   ★ ZSET ⇒ leaderboards (★ ZRANK is O(log n)), feeds, rate limits
   ★ HASH ⇒ update one field without rewriting the value
   ★ HLL  ⇒ ★ 40M uniques in 12 KB, ±0.81%
   ★ BITMAP ⇒ 10M daily actives in 1.2 MB
   ★ STREAM ⇒ a durable queue with consumer groups
   ⇒ ★ if you only use GET/SET, use Memcached

★ MEMORY
   ★ ~50–100 bytes overhead PER KEY, before the value
   ⇒ ★ 1,000 hashes of 100 fields ≪ 100,000 separate keys
   ⇒ ★ compact encodings under hash-max-listpack-entries etc.
   ⇒ ★ key names are stored in full, millions of times
```

---

## Diagrams

**Diagram 1 — big picture: the key IS the query**

```
 ★ RELATIONAL — the query names what it wants
   SELECT * FROM orders
    WHERE customer_id = 42 AND created_at >= '2026-08-01'
    ORDER BY created_at DESC LIMIT 20;
   ⇒ ★ the DATABASE finds a plan. Add an index later if it's slow.

 ★ KEY-VALUE — the KEY must already contain the question
   ┌──────────────────────────────────────────────────────────────┐
   │ PK              │ SK                        │ value          │
   ├─────────────────┼───────────────────────────┼────────────────┤
   │ USER#42         │ PROFILE                   │ {name, email}  │
   │ USER#42         │ ORDER#2026-08-25#8842119  │ {total,status} │
   │ USER#42         │ ORDER#2026-08-24#8842118  │ {total,status} │
   │ USER#42         │ ORDER#2026-08-01#8842001  │ {total,status} │
   │ USER#42         │ ADDRESS#1                 │ {line1,city}   │
   └─────────────────┴───────────────────────────┴────────────────┘
                              ▲
   ★ Query(PK='USER#42', SK begins_with 'ORDER#',
           ScanIndexForward=false, Limit=20)
   ⇒ ★ ONE REQUEST. The sort key already encodes the ordering and
     the filtering.

   ★ BUT: "all orders over ₹5,000, any user"
   ⇒ ★ NOT SERVED BY THIS KEY. Options:
     ⒜ ★ Scan the whole table (never, at scale)
     ⒝ ★ a GSI — a second copy, async, its own write cost
     ⒞ ★ write the data AGAIN under a different key
   ⇒ ★ IN SQL THIS WOULD BE `CREATE INDEX`. HERE IT IS A DESIGN
     CHANGE AND A BACKFILL.
```

**Diagram 2 — data flow: the hot partition, and write sharding**

```
 ✗ ONE PARTITION KEY PER CELEBRITY
   PK = 'USER#8842119'   (88M followers, viral post)
   ┌──────────────────────────────────────────────────────────────┐
   │  writes/sec ──────► ★ partition A  (limit ★ 1,000 WCU/s)     │
   │      41,000                │                                  │
   │                            ▼                                  │
   │                  ★ ProvisionedThroughputExceededException     │
   │                                                                │
   │  ★ AND: the table's total capacity is 100,000 WCU.            │
   │    ★ IT DOES NOT HELP. The limit is PER PARTITION.            │
   │  ⇒ ★ THIS IS TOPIC 61'S HOT ROW, AT THE STORAGE LAYER.        │
   └──────────────────────────────────────────────────────────────┘

 ✓ WRITE SHARDING — a suffix on the partition key
   PK = 'USER#8842119#' + random(0..15)
   ┌──────────────────────────────────────────────────────────────┐
   │  writes ──┬──► USER#8842119#0   ★ 2,560/s   partition A      │
   │           ├──► USER#8842119#1   ★ 2,560/s   partition B      │
   │           ├──► …                                              │
   │           └──► USER#8842119#15  ★ 2,560/s   partition P      │
   │                                                                │
   │  ✓ ★ 16× the write ceiling                                    │
   │  ✗ ★ READS MUST NOW QUERY ALL 16 AND MERGE                    │
   │    ⇒ 16× the read cost, and ★ latency = the SLOWEST shard    │
   │      (Topic 60's scatter-gather)                              │
   └──────────────────────────────────────────────────────────────┘
      ★ THE SAME TRADE AS TOPIC 61: contention down, read cost up.
      ★ AND THE SAME REFINEMENT: shard only the HOT keys, via a
        registry — not every key.
```

**Diagram 3 — before/after: choosing the right Redis structure**

```
 ✗ A LEADERBOARD AS A SERIALISED JSON STRING
 ┌───────────────────────────────────────────────────────────────┐
 │ SET leaderboard '[{"id":1,"score":900}, … ×100,000]'          │
 │                                                                │
 │ ★ update one score:  GET (2.8 MB) → parse → sort → SET (2.8MB)│
 │                      ★ 84 ms, ★ 5.6 MB of network per update  │
 │ ★ get the top 10:    GET 2.8 MB → parse → slice               │
 │                      ★ 41 ms                                   │
 │ ★ "what is player 4,201's rank?" ⇒ ★ parse everything, scan   │
 │                      ★ 44 ms                                   │
 │ ★ memory:            2.8 MB, ★ rewritten on every update       │
 │ ★ concurrent updates ⇒ ★ LOST UPDATES (read-modify-write)      │
 └───────────────────────────────────────────────────────────────┘

 ✓ A SORTED SET
 ┌───────────────────────────────────────────────────────────────┐
 │ ZADD leaderboard 900 player:1                                  │
 │                                                                │
 │ ★ update one score:  ZADD              ★ 0.18 ms  O(log n)     │
 │                      ★ atomic — no lost updates                │
 │ ★ get the top 10:    ZREVRANGE 0 9     ★ 0.21 ms  O(log n + m) │
 │ ★ player's rank:     ★ ZREVRANK        ★ 0.19 ms  O(log n)     │
 │                      ⇒ ★ this is the operation that is        │
 │                        genuinely hard in SQL                   │
 │ ★ memory:            ~7 MB, ★ only the changed member written  │
 │ ★ score range:       ZRANGEBYSCORE     ★ O(log n + m)          │
 └───────────────────────────────────────────────────────────────┘
        ★ 466× on updates, 195× on reads, and ★ correctness —
          the JSON version loses concurrent updates entirely.
```

---

## Example 1 — basic

**Prove `KEYS` blocks the server.**
```bash
redis-cli eval "for i=1,5000000 do redis.call('SET','k:'..i,i) end" 0
redis-cli --latency-history -i 5 &
redis-cli KEYS 'k:*' > /dev/null
```
```
 ★ min: 0, max: 4218, avg: 210.4 (5 samples)
   ★ 4.2 SECONDS OF TOTAL BLOCKAGE. Every client waited.
```
```bash
# ★ the safe alternative
time redis-cli --scan --pattern 'k:*' | wc -l
```
```
 5000000
 ★ real 3.1s — but in ★ incremental cursor batches. Latency stayed
   under 1 ms throughout.
```

**Find the dangerous keys before they find you.**
```bash
redis-cli --bigkeys
```
```
 [00.00%] Biggest string found so far 'session:88420' with 41 bytes
 [12.40%] ★ Biggest hash   found so far 'user:profiles' with 4,102,884 fields
 [58.20%] ★ Biggest zset   found so far 'leaderboard:global' with 8,842,119 members

 ★ Sampled 5,000,004 keys
 ★ 'user:profiles' — 4.1M fields in ONE hash.
   ⇒ ★ HGETALL on it is O(n) and blocks. SCAN it, or split it.
```

**Prove `DEL` blocks and `UNLINK` doesn't.**
```bash
redis-cli eval "for i=1,3000000 do redis.call('HSET','big','f'..i,i) end" 0
redis-cli --latency-history -i 2 &
redis-cli DEL big
```
```
 ★ max: 1,402 ms        — the whole server stopped.
```
```bash
redis-cli eval "for i=1,3000000 do redis.call('HSET','big2','f'..i,i) end" 0
redis-cli UNLINK big2
```
```
 ★ max: 2 ms        — freed on a background thread.
```

**Prove the memory overhead per key.**
```bash
redis-cli FLUSHALL
redis-cli INFO memory | grep used_memory:
# ★ baseline
redis-cli eval "for i=1,1000000 do redis.call('SET','user:profile:'..i,'x') end" 0
redis-cli INFO memory | grep used_memory_human
```
```
 used_memory_human: ★ 89.4M        — for 1M keys of 1 byte each
   ★ ~89 bytes per key. The VALUE is 1 byte.
```
```bash
redis-cli FLUSHALL
# ★ the same data in 10,000 hashes of 100 fields
redis-cli eval "
  for i=1,10000 do
    for j=1,100 do redis.call('HSET','h:'..i,'f'..j,'x') end
  end" 0
redis-cli INFO memory | grep used_memory_human
```
```
 used_memory_human: ★ 11.2M        ★ 8× LESS for the same data.
   ⇒ ★ because each hash is under hash-max-listpack-entries (128)
     and uses the compact encoding.
```

**Prove the compact-encoding threshold matters.**
```bash
redis-cli CONFIG GET hash-max-listpack-entries
redis-cli DEL h; redis-cli eval "for i=1,128 do redis.call('HSET','h','f'..i,'v') end" 0
redis-cli MEMORY USAGE h
redis-cli HSET h f129 v
redis-cli MEMORY USAGE h
```
```
 ★ 1,752         — listpack encoding
 ★ 10,296        — ★ 5.9× MORE, from ONE extra field crossing the
                   threshold into a real hashtable.
```

**Compare a leaderboard as JSON vs as a ZSET.**
```js
// ★ JSON string
const board = Array.from({length: 100000}, (_, i) => ({id: i, score: i}));
console.time('json-update');
const s = JSON.parse(await redis.get('lb:json'));
s.find(x => x.id === 4201).score = 9999;
s.sort((a,b) => b.score - a.score);
await redis.set('lb:json', JSON.stringify(s));
console.timeEnd('json-update');
```
```
 ★ json-update: 84.2 ms
```
```js
// ★ ZSET
console.time('zset-update');
await redis.zadd('lb:z', 9999, 'player:4201');
console.timeEnd('zset-update');

console.time('zset-rank');
const rank = await redis.zrevrank('lb:z', 'player:4201');
console.timeEnd('zset-rank');
```
```
 ★ zset-update: 0.18 ms       ★ 466×
 ★ zset-rank:   0.19 ms       ★ and this operation is essentially
                                impossible to do cheaply in SQL
```

**Prove HyperLogLog's memory claim.**
```bash
redis-cli DEL uv:set uv:hll
redis-cli eval "
  for i=1,1000000 do
    redis.call('SADD','uv:set','user'..i)
    redis.call('PFADD','uv:hll','user'..i)
  end" 0
redis-cli MEMORY USAGE uv:set
redis-cli MEMORY USAGE uv:hll
redis-cli SCARD uv:set
redis-cli PFCOUNT uv:hll
```
```
 ★ 84,215,320        — the SET: 84 MB
 ★ 12,304            — the HLL: ★ 12 KB.  ★ 6,845×
 1000000
 ★ 997,845           — ★ 0.22% error
```

**Prove `maxmemory-policy` silently loses data.**
```bash
redis-cli CONFIG SET maxmemory 10mb
redis-cli CONFIG SET maxmemory-policy allkeys-lru
redis-cli SET important:order:8842119 '{"total":250000}'
redis-cli eval "for i=1,200000 do redis.call('SET','junk:'..i, string.rep('x',100)) end" 0
redis-cli GET important:order:8842119
```
```
 ★ (nil)        — ★ YOUR DATA WAS EVICTED TO MAKE ROOM FOR CACHE.
   ★ No error. No log. It is simply gone.
```
```bash
redis-cli CONFIG SET maxmemory-policy noeviction
redis-cli SET important:order:2 '{"total":1}'
redis-cli eval "for i=1,200000 do redis.call('SET','junk2:'..i, string.rep('x',100)) end" 0
```
```
 ★ (error) OOM command not allowed when used memory > 'maxmemory'
   ★ THE WRITE FAILS LOUDLY. That is what you want for data.
```

**The distributed lock, done correctly.**
```bash
# ★ the broken version
redis-cli SET lock:job '1' NX EX 2
sleep 3                          # ★ the lock expires while "working"
redis-cli SET lock:job '1' NX EX 10   # ★ another process takes it
redis-cli DEL lock:job                # ★ the FIRST process deletes it
redis-cli GET lock:job
```
```
 ★ (nil)        — ★ process 1 released process 2's lock.
```
```bash
# ★ the correct version
TOKEN=$(uuidgen)
redis-cli SET lock:job "$TOKEN" NX PX 10000
redis-cli EVAL "if redis.call('get',KEYS[1])==ARGV[1] then
                  return redis.call('del',KEYS[1]) else return 0 end" \
          1 lock:job "wrong-token"
```
```
 ★ (integer) 0        — ★ refused. Only the holder can release it.
```

**Persistence: prove the default loses data.**
```bash
redis-cli CONFIG GET save
```
```
 ★ 1) "save"  2) "3600 1 300 100 60 10000"
   ★ a snapshot after 1 change in 3,600 s ⇒ ★ up to an HOUR of loss.
```
```bash
redis-cli SET critical:value 'important'
redis-cli DEBUG SLEEP 0 && kill -9 $(pgrep redis-server)
redis-server /etc/redis/redis.conf --daemonize yes
sleep 2 && redis-cli GET critical:value
```
```
 ★ (nil)        — gone.
```
```bash
# ★ with AOF
redis-cli CONFIG SET appendonly yes
redis-cli CONFIG SET appendfsync everysec
redis-cli SET critical:value2 'important'
sleep 2 && kill -9 $(pgrep redis-server)
redis-server /etc/redis/redis.conf --daemonize yes
sleep 2 && redis-cli GET critical:value2
```
```
 ★ "important"        — survived.
```

---

## Example 2 — production scenario

**The situation.** A gaming platform. Global and per-region leaderboards, 40 million players, live rank updates during tournaments.

```
 THE ORIGINAL DESIGN — PostgreSQL
   scores(player_id, game_id, region, score, updated_at)
   ⇒ "top 100 global"     ★ 84 ms   (an index handles it)
   ⇒ ★ "what is MY rank?" ★ 8,412 ms
      SELECT count(*) FROM scores WHERE game_id=$1 AND score > $2;
      ⇒ ★ counts up to 40 million rows, per request
   ⇒ during a tournament: ★ 41,000 rank requests/sec
   ⇒ ★ database CPU 100%, p99 unbounded
```

**Step 1 — why rank is genuinely hard in SQL.**

```sql
EXPLAIN (ANALYZE)
SELECT count(*) + 1 AS rank FROM scores
 WHERE game_id = 7 AND score > 88420;
```
```
 Aggregate  (actual time=8,402.1..8,402.1 rows=1)
   ->  Index Only Scan using scores_game_score_idx
         (actual rows=★ 12,884,201)
 Execution Time: ★ 8,412.4 ms
   ★ AN INDEX-ONLY SCAN OVER 12.8 MILLION INDEX ENTRIES.
     The index is perfect; ★ the operation is inherently O(n).
   ⇒ ★ window functions don't help — RANK() OVER (ORDER BY score)
     must still process every row.
```
```
 ★ THIS IS THE CASE WHERE A DIFFERENT DATA STRUCTURE, NOT A
   BETTER QUERY, IS THE ANSWER.
   ⇒ ★ Redis's ZSET is a skip list with span counts, so ZREVRANK
     is ★ O(log n) — 0.19 ms instead of 8,412 ms.
```

**Step 2 — the key design.**

```
 ★ ACCESS PATTERNS, WRITTEN FIRST:
 ① top N for a game, globally
 ② top N for a game, per region
 ③ ★ a player's rank (global and regional)
 ④ ★ a player's neighbours (±5 ranks) — the "you are here" view
 ⑤ a player's score history over a season
 ⑥ ★ scores must survive a Redis restart (they are the record)
```
```
 ★ THE KEYS:
   lb:{game}:global              ZSET  member=player, score=score
   lb:{game}:{region}            ZSET
   lb:{game}:global:{season}     ZSET  ★ seasonal, with a TTL
   p:{player}                    HASH  profile, ★ one field per game
   ★ NOTE THE HASH TAGS {game}: in Redis Cluster, keys sharing a
     hash tag land on the SAME SLOT ⇒ ★ multi-key operations
     (ZUNIONSTORE across regions) work.
```

**Step 3 — the operations.**

```js
// ★ ① submit a score — atomic, and only if it's an improvement
const submitScript = `
  local key, player, score = KEYS[1], ARGV[1], tonumber(ARGV[2])
  local current = redis.call('ZSCORE', key, player)
  if current and tonumber(current) >= score then return 0 end
  redis.call('ZADD', key, score, player)
  return 1`;

async function submitScore(gameId, region, playerId, score) {
  const keys = [`lb:{${gameId}}:global`, `lb:{${gameId}}:${region}`];
  // ★ a pipeline: one round trip for both leaderboards
  const pipe = redis.pipeline();
  for (const k of keys) pipe.eval(submitScript, 1, k, playerId, score);
  await pipe.exec();
}
```

```js
// ★ ② rank and neighbours — the "you are here" view, one round trip
const neighbourScript = `
  local key, player, window = KEYS[1], ARGV[1], tonumber(ARGV[2])
  local rank = redis.call('ZREVRANK', key, player)
  if not rank then return nil end
  local from = math.max(0, rank - window)
  local to = rank + window
  local rows = redis.call('ZREVRANGE', key, from, to, 'WITHSCORES')
  return { rank, from, rows }`;

const [rank, firstRank, rows] = await redis.eval(
  neighbourScript, 1, `lb:{${gameId}}:global`, playerId, 5);
// ★ ZREVRANK O(log n) + ZREVRANGE O(log n + 11)
// ★ MEASURED: 0.31 ms for rank AND the 11 surrounding players.
```

```js
// ★ ③ pagination that does NOT use OFFSET
// ✗ ZREVRANGE key 100000 100019 ⇒ ★ O(log n + 100020) — it must
//   walk to the offset.
// ✓ ★ score-based (keyset) pagination — O(log n + m)
const page = await redis.zrevrangebyscore(
  `lb:{${gameId}}:global`,
  `(${lastScore}`, '-inf',     // ★ exclusive, from the last seen score
  'WITHSCORES', 'LIMIT', 0, 20);
// ⇒ ★ the same keyset-pagination principle as SQL (Topic 14),
//   and for the same reason: OFFSET is O(offset).
```

**Step 4 — durability, because this is a system of record.**

```ini
# ★ redis.conf — NOT the cache defaults
appendonly yes
★ appendfsync everysec          # ★ ≤1 s of loss; `always` is ~10× slower
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 512mb
save 900 1                      # ★ RDB too, for fast restarts
★ maxmemory-policy noeviction    # ★ NEVER allkeys-lru on data
maxmemory 24gb                  # ★ leave headroom for fork()
★ maxmemory-clients 1gb          # bound client output buffers
```
```
 ★ THE fork() TRAP: RDB save and AOF rewrite both fork(), and
   copy-on-write can briefly DOUBLE memory on a write-heavy
   instance.
 ⇒ ★ maxmemory must be ≤ ~45% of system RAM, not 90%.
 ⇒ MEASURED during a tournament: used_memory 22 GB, and during
   an AOF rewrite ★ RSS peaked at 38 GB on a 64 GB machine.
```

```
 ★ AND THE HONEST PART: Redis with appendfsync=everysec has an
   RPO of ★ ~1 SECOND. For a leaderboard that is fine.
 ⇒ ★ THE SCORES ARE ALSO WRITTEN TO POSTGRESQL, asynchronously,
   as the true system of record — because a tournament result is
   a business record and 1 second of loss is not acceptable there.
 ⇒ ★ REDIS IS THE INDEX; POSTGRESQL IS THE LEDGER. (Topic 76.)
```

**Step 5 — the hot-key problem, at Redis Cluster level.**

```
 ★ DURING A MAJOR TOURNAMENT, ONE GAME'S LEADERBOARD TAKES 88%
   OF ALL TRAFFIC.
 ⇒ ★ all its keys share the hash tag {game} ⇒ ★ ONE SLOT ⇒
   ★ ONE NODE. The other 5 nodes are idle.
 ⇒ ★ THIS IS THE HOT PARTITION AGAIN (Topics 60, 61).
```
```js
// ★ THE FIX FOR READS: replicas + client-side read routing
const cluster = new Redis.Cluster(nodes, {
  ★ scaleReads: 'slave',        // route reads to replicas
  redisOptions: { enableReadyCheck: true },
});
// ⇒ ★ 3 replicas of the hot node ⇒ 4× the read capacity.
// ⇒ ★ writes still go to one node — but writes are 3% of traffic.
```
```js
// ★ AND FOR THE TOP-N READ, WHICH IS 70% OF REQUESTS:
//   it changes slowly and is the same for everyone.
//   ⇒ ★ cache it in-process for 1 second (Topic 57, layer ④).
let topCache = { at: 0, rows: null };
async function topN(gameId) {
  if (Date.now() - topCache.at < 1000) return topCache.rows;
  const rows = await redis.zrevrange(`lb:{${gameId}}:global`, 0, 99,
                                     'WITHSCORES');
  topCache = { at: Date.now(), rows };
  return rows;
}
// ⇒ ★ MEASURED: 41,000 rps → ★ 6 Redis calls/sec for the top-N.
//   ★ The in-process cache removed 70% of the load, for one second
//     of staleness that nobody can perceive.
```

**Step 6 — memory, measured and controlled.**

```bash
redis-cli MEMORY USAGE 'lb:{7}:global'
redis-cli ZCARD 'lb:{7}:global'
```
```
 ★ 2,884,201,880        — 2.8 GB
 ★ 40,102,884           — 40M members
   ★ ~72 bytes per member (the member string + the score + skip-list
     pointers).
```
```
 ★ THE LEVERS APPLIED:
 ① ★ SHORTER MEMBER NAMES: 'player:8842119' → '8842119'
    ⇒ 40M × 7 bytes saved = ★ 280 MB
 ② ★ TTL ON SEASONAL BOARDS
    EXPIRE lb:{7}:global:s12 (90 days)
    ⇒ ★ 12 seasons × 2.8 GB would have been 34 GB
 ③ ★ TRIM BOARDS THAT DON'T NEED EVERY PLAYER
    ZREMRANGEBYRANK lb:{7}:{region} 0 -100001   -- ★ keep the top 100k
    ⇒ regional boards: 2.8 GB → ★ 7 MB each
 ⇒ ★ TOTAL: 34 GB → 3.1 GB.
```

**Step 7 — results.**

| | PostgreSQL only | Redis ZSET + PostgreSQL |
|---|---|---|
| ★ "my rank" | ★ **8,412 ms** | ★ **0.19 ms** (**44,000×**) |
| Rank + neighbours | not attempted | **0.31 ms** |
| Top 100 | 84 ms | 0.21 ms (**6/sec** after in-process cache) |
| Score submit | 4.2 ms | 0.38 ms |
| Peak rank requests/sec | ★ ~120 (CPU-bound) | **41,000** |
| Memory | — | 34 GB → ★ **3.1 GB** |
| Durability | full | ★ **RPO ~1 s in Redis; full in PostgreSQL** |
| System of record | PostgreSQL | ★ **PostgreSQL** (Redis is the index) |

```
 ★ SIX LESSONS:
 ① ★ RANK IS THE OPERATION THAT JUSTIFIES A DIFFERENT DATA
   STRUCTURE. An index cannot make counting 12.8M rows fast;
   a skip list with span counts makes it O(log n).
 ② ★ REDIS IS THE INDEX, POSTGRESQL IS THE LEDGER. Tournament
   results are business records; ★ 1 second of RPO is fine for a
   live leaderboard and not for a payout.
 ③ ★ THE DEFAULTS ARE CACHE DEFAULTS. `allkeys-lru` would have
   silently evicted scores; `save 3600 1` would have lost an hour.
 ④ ★ THE HOT KEY REAPPEARED AT EVERY LAYER — hot row (61), hot
   partition (60), hot slot (here). ★ The same three fixes apply:
   replicas for reads, sharding for writes, and caching the shared
   result.
 ⑤ ★ AN IN-PROCESS 1-SECOND CACHE REMOVED 70% OF THE LOAD. The
   top-N is identical for every user and changes slowly.
 ⑥ ★ MEMORY WAS 11× OVER BUDGET FROM THREE AVOIDABLE THINGS:
   long member names, no TTL on seasonal boards, and keeping all
   40M players in regional boards nobody paginated past 100k.
```

---

## Common mistakes

**1. Running `KEYS`, `SMEMBERS` or `HGETALL` on a large key in production.**
- *Symptom:* seconds of total server blockage; every client times out.
- *Fix:* `SCAN`/`SSCAN`/`HSCAN`, `UNLINK` instead of `DEL`, and `redis-cli --bigkeys` as a routine check.

**2. `maxmemory-policy allkeys-lru` on data.**
- *Symptom:* records silently disappear under memory pressure. No error, no log.
- *Fix:* `noeviction` (writes fail loudly) or `volatile-lru` (only TTL'd keys are evicted).

**3. Relying on default persistence.**
- *Symptom:* a restart loses up to an hour of writes.
- *Fix:* `appendonly yes` + `appendfsync everysec`, plus RDB for fast restarts. Know your RPO is ~1 second.

**4. Setting `maxmemory` near system RAM.**
- *Symptom:* an OOM kill during an RDB save or AOF rewrite.
- *Fix:* `maxmemory` ≤ ~45% of RAM — `fork()` plus copy-on-write can briefly double RSS.

**5. Millions of tiny top-level keys.**
- *Symptom:* 89 bytes of overhead per key; 1M one-byte values costing 89 MB.
- *Fix:* group into hashes under the compact-encoding thresholds — measured 8× less memory.

**6. Crossing a compact-encoding threshold.**
- *Symptom:* one extra field makes a hash 5.9× larger.
- *Fix:* know `hash-max-listpack-entries` / `zset-max-listpack-entries`, and keep aggregates below them where it matters.

**7. Serialising a collection into one string value.**
- *Symptom:* every update reads, parses, sorts and rewrites megabytes — and concurrent updates lose data.
- *Fix:* the right structure. A ZSET update is 466× faster *and* atomic.

**8. A naive distributed lock.**
- *Symptom:* one process deletes another's lock after its own expired.
- *Fix:* a unique token plus a Lua compare-and-delete — and don't use it for correctness-critical mutual exclusion.

**9. `ZRANGE` with a large offset.**
- *Symptom:* O(log n + offset) — deep pagination degrades linearly.
- *Fix:* score-based keyset pagination, exactly as in SQL.

**10. Not planning access patterns before choosing keys.**
- *Symptom:* a new requirement needs a `Scan`, a GSI, or a full rewrite.
- *Fix:* enumerate every access pattern first. In a key-value store that enumeration *is* the schema design.

**11. Ignoring the hot-partition limit.**
- *Symptom:* `ProvisionedThroughputExceededException` while the table has ample total capacity.
- *Fix:* write-shard the hot keys only, via a registry — and accept the read-side scatter (Topics 60, 61).

**12. Treating item size as free in DynamoDB.**
- *Symptom:* a 400 KB item costs 400 WCU per write.
- *Fix:* keep items small; store large payloads in S3 with a pointer.

**13. Using Redis for data that needs stronger durability than 1 second.**
- *Symptom:* a payment or a ledger entry lost on a crash.
- *Fix:* Redis as the index or cache; a durable store as the record (Topic 76).

---

## Hands-on proof

**PROVE IT #1–#10 — Example 1** (`KEYS` blocking for 4.2 s and `SCAN` not, `--bigkeys` finding a 4.1M-field hash, `DEL` at 1,402 ms vs `UNLINK` at 2 ms, 89 bytes of overhead per key and 8× savings from hashes, a compact-encoding threshold costing 5.9×, a ZSET beating serialised JSON by 466×, HyperLogLog at 6,845× less memory with 0.22% error, `allkeys-lru` silently evicting data, the broken and correct lock, and default persistence losing everything).

**PROVE IT #11 — `ZRANGE` offset is O(offset).**
```bash
redis-cli eval "for i=1,1000000 do redis.call('ZADD','z',i,'m'..i) end" 0
for off in 0 100000 500000 900000; do
  echo -n "offset=$off  "
  redis-cli --latency -i 1 ZREVRANGE z $off $((off+19)) > /dev/null &
  /usr/bin/time -f '%e s' redis-cli ZREVRANGE z $off $((off+19)) > /dev/null
done
```
```
 offset=0       ★ 0.0002 s
 offset=100000  ★ 0.0021 s
 offset=500000  ★ 0.0094 s
 offset=900000  ★ 0.0171 s
   ★ LINEAR IN THE OFFSET. Use ZREVRANGEBYSCORE with a cursor.
```

**PROVE IT #12 — a pipeline collapses round trips.**
```js
console.time('sequential');
for (let i = 0; i < 1000; i++) await redis.set(`k${i}`, i);
console.timeEnd('sequential');

console.time('pipelined');
const p = redis.pipeline();
for (let i = 0; i < 1000; i++) p.set(`k${i}`, i);
await p.exec();
console.timeEnd('pipelined');
```
```
 ★ sequential: 184.2 ms      (1,000 round trips)
 ★ pipelined:   4.1 ms       ★ 45× — the same N+1 lesson (Topic 66)
```

**PROVE IT #13 — `fork()` doubles RSS.**
```bash
redis-cli INFO memory | grep -E 'used_memory_human|rss_human'
redis-cli BGSAVE
watch -n0.2 "redis-cli INFO memory | grep rss_human"
```
```
 used_memory_human: 8.20G
 used_memory_rss_human: 8.41G
 ★ during BGSAVE, rss peaked at ★ 14.9G
   ⇒ ★ maxmemory must leave room for this.
```

**PROVE IT #14 — Cluster hash tags control key placement.**
```bash
redis-cli -c CLUSTER KEYSLOT 'lb:{7}:global'
redis-cli -c CLUSTER KEYSLOT 'lb:{7}:apac'
redis-cli -c CLUSTER KEYSLOT 'lb:7:global'
redis-cli -c CLUSTER KEYSLOT 'lb:7:apac'
```
```
 ★ 12182
 ★ 12182        — ★ same slot: multi-key ops work
 ★ 8104
 ★ 3211         — ★ different slots: CROSSSLOT error on multi-key ops
```

---

## The design decision framework

```
★★★ THE KEY IS THE QUERY LANGUAGE.
    ENUMERATE ACCESS PATTERNS FIRST, OR DON'T START. ★★★

 ① ★ NAME THE SPECIFIC STORE
    Redis · Memcached · DynamoDB · etcd — ★ these differ by 1,000×
    in durability, consistency and cost. "Key-value" is not a
    design decision.
    ⇒ ★ if you only use GET/SET, ★ use Memcached.
    ⇒ ★ Redis earns its place through ZSET / HASH / HLL / STREAM.

 ② ★ WRITE DOWN EVERY ACCESS PATTERN, VERBATIM
    ⇒ ★ then design a key that answers each in ONE operation.
    ⇒ ★ anything not served needs a second copy (a GSI, a second
      write, a different key) — ★ there is no "just add an index".

 ③ ★ CHECK EVERY KEY FOR HOTNESS
    what is the busiest single key value?
    ⇒ Redis: ★ one slot = one node ⇒ replicas for reads,
      ★ an in-process cache for shared results
    ⇒ DynamoDB: ★ ~1,000 WCU per partition ⇒ write-shard the HOT
      keys only, via a registry (Topics 60, 61)

 ④ ★ CHOOSE THE STRUCTURE, NOT JUST THE KEY
    ranked/ordered ⇒ ★ ZSET (ZRANK is O(log n) — ★ the thing SQL
                      cannot do cheaply)
    partial update ⇒ ★ HASH (not a serialised JSON string)
    approx unique  ⇒ ★ HYPERLOGLOG (12 KB for any cardinality)
    per-user flags ⇒ ★ BITMAP
    durable queue  ⇒ ★ STREAM (consumer groups + XACK)
    ⇒ ★ the structure choice is usually a bigger win than any
      tuning.

 ⑤ ★ IF IT IS A SYSTEM OF RECORD, CHANGE THE DEFAULTS
    ✓ ★ appendonly yes + appendfsync everysec (RPO ~1 s)
    ✓ ★ maxmemory-policy noeviction (or volatile-lru)
    ✓ ★ maxmemory ≤ ~45% of RAM (fork() doubles RSS)
    ✓ ★ and ask honestly whether ~1 s RPO is acceptable —
      ★ if not, the durable store is the record and this is the
      index (Topic 76).

 ⑥ ★ MEMORY IS A DESIGN CONSTRAINT
    ★ ~50–100 bytes overhead per key
    ⇒ ★ group into hashes under the compact-encoding thresholds
      (measured 8×)
    ⇒ ★ short key and member names — they are stored millions of
      times
    ⇒ ★ TTLs on anything that can expire
    ⇒ ★ trim collections nobody paginates to the end of

 ⑦ ★ NEVER RUN AN O(n) COMMAND IN PRODUCTION
    KEYS ⇒ SCAN · SMEMBERS ⇒ SSCAN · HGETALL ⇒ HSCAN · DEL ⇒ UNLINK
    ⇒ ★ `redis-cli --bigkeys` on a schedule.
    ⇒ ★ and Redis is single-threaded: one slow command is a
      full outage.

 ⑧ ★ PIPELINE — IT IS THE SAME LESSON AS N+1
    1,000 sequential commands: 184 ms · pipelined: ★ 4.1 ms
    ⇒ ★ 45×, and for exactly the reason in Topic 66.
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
Load 5M keys into Redis. Then: (a) time `KEYS *` while measuring latency with `--latency-history`; (b) do the same with `--scan`; (c) compare `DEL` and `UNLINK` on a 3M-field hash; (d) measure memory for 1M individual keys versus the same data in 10,000 hashes, and explain the difference.

### Exercise 2 — medium (apply it)
Implement a leaderboard three ways: a serialised JSON string, a PostgreSQL table with an index, and a Redis ZSET. For each, measure: updating one score, fetching the top 10, and **fetching one player's rank**. Explain why the third operation differs so dramatically.

Then implement the sliding-window rate limiter in Lua and prove it is atomic under concurrent load.

### Exercise 3 — hard (production simulation)
A gaming platform's "what is my rank?" query takes 8,412 ms in PostgreSQL, and tournaments generate 41,000 rank requests/sec.

(a) Explain why an index cannot fix this, and what property of a skip list makes `ZREVRANK` O(log n).
(b) Enumerate the six access patterns and design the key schema, including Redis Cluster hash tags. Explain what the hash tags buy.
(c) Implement score submission so it is atomic and only accepts improvements.
(d) Implement "rank + 5 neighbours" in one round trip.
(e) Deep pagination is required. Explain why `ZREVRANGE` with an offset is wrong and implement the alternative.
(f) The default Redis configuration is unsuitable. Give the five settings you'd change and the failure each prevents.
(g) `maxmemory` was set to 90% of RAM. Explain the `fork()` failure mode and give the correct value.
(h) During a tournament one game takes 88% of traffic and lands on one node. Give three mitigations and say which removes the most load.
(i) Memory is 34 GB against a 4 GB budget. Find three savings and quantify each.
(j) Explain why the scores are also written to PostgreSQL, and what that says about Redis's role.

---

## Mental model checkpoint

1. Why must every access pattern be enumerated before designing keys?
2. Name four members of the key-value family and one property that distinguishes each.
3. Why is Redis being single-threaded both an advantage and a severe risk?
4. Name four O(n) Redis commands and their safe alternatives.
5. What are Redis's three persistence modes, and which combination suits a system of record?
6. Why is `maxmemory-policy allkeys-lru` dangerous for data?
7. Why must `maxmemory` be well below system RAM?
8. What are DynamoDB's three hard limits, and which causes the hot-partition problem?
9. Explain single-table design and what it precomputes.
10. Which Redis operation is genuinely hard in SQL, and why?
11. Why do 10,000 hashes of 100 fields use 8× less memory than 1,000,000 keys?
12. Why is `ZRANGE` with a large offset slow, and what replaces it?

---

## Quick reference card

**★ Never in production:** `KEYS` → `SCAN` · `SMEMBERS` → `SSCAN` · `HGETALL` → `HSCAN` · `DEL` → `UNLINK` · `FLUSHALL`.
`redis-cli --bigkeys` on a schedule.

**System-of-record config**
```ini
appendonly yes
★ appendfsync everysec            # RPO ~1 s
★ maxmemory-policy noeviction     # ★ never allkeys-lru on data
★ maxmemory <= 45% of RAM         # ★ fork() doubles RSS
save 900 1                        # RDB for fast restarts
```

**Structures — choose deliberately**

| Need | Structure | Why |
|---|---|---|
| rank / ordered | ★ `ZSET` | ★ `ZREVRANK` O(log n) |
| partial update | ★ `HASH` | no full rewrite |
| approx distinct | ★ `HLL` | ★ 12 KB, ±0.81% |
| per-user flags | `BITMAP` | 10M users = 1.2 MB |
| durable queue | ★ `STREAM` | consumer groups + `XACK` |
| only GET/SET | ★ **use Memcached** | multi-threaded, simpler |

**Memory:** ★ ~50–100 B per key · group into hashes under `hash-max-listpack-entries` (★ 8×) · short key/member names · TTLs · trim.

**DynamoDB:** ★ 400 KB/item · ★ 10 GB/partition · ★ ~3,000 RCU / 1,000 WCU **per partition** · `Query` not `Scan` · single-table = the join precomputed · **write-shard hot PKs only**.

**★ Pipeline everything:** 1,000 sequential 184 ms → pipelined **4.1 ms** (Topic 66's lesson).

**★ Keyset, not offset:** `ZREVRANGEBYSCORE key (lastScore -inf LIMIT 0 20`.

---

## When would I use this at work?

1. **Leaderboards, rate limiting, and anything needing rank or ordered ranges.** `ZREVRANK` at O(log n) is the operation SQL genuinely cannot do cheaply — 44,000× in the example — and it is the clearest justification for adding Redis to a stack.

2. **Before choosing Redis over Memcached.** If the design only uses `GET`/`SET`, Memcached is multi-threaded, simpler, and has fewer ways to lose data. Redis earns its place through its value types, not its speed.

3. **When Redis is proposed as a system of record.** Three settings decide whether that is defensible: `appendonly`, `maxmemory-policy`, and `maxmemory` relative to RAM. The defaults are cache defaults, and `allkeys-lru` on data is silent loss.

4. **Any DynamoDB design review.** Ask for the enumerated access patterns and the busiest partition key. Those two answers predict every problem the design will have — and unlike SQL, there is no adding an index later.

---

## Connected topics

**Understand before this:** 57 (Redis as a cache — and why that's a different discipline), 60 (sharding — the hot partition), 61 (hot rows — the same problem one layer down), 66 (round trips — why pipelining matters), 08 (LSM trees — the engine inside most of these).

**This unlocks:**
- **72** — wide-column: the partition key idea taken to its conclusion
- **73** — time-series: where the sort key is always time
- **75** — SQL vs NoSQL: the decision framework
- **76** — polyglot persistence: Redis as the index, PostgreSQL as the ledger
