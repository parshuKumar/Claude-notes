# 84 — Distributed Locking — Coordination Across Services
## Category: HLD Components

---

## What is this?

A **distributed lock** is a "only one of you at a time" token that lives on a machine *outside* all the machines competing for it. Think of a shared office bathroom with a single physical key hanging by the door: whoever takes the key gets the room, and everyone else waits by the door. The key is not inside anybody's pocket-universe — it's on a hook everyone can see.

In software, your service runs on 10 servers. When you need exactly one of those servers to run the nightly billing job, or exactly one worker to process order #4471, you need a lock that all 10 can see. That lock lives in Redis, ZooKeeper, etcd, or your database — never in one server's memory.

---

## Why does it matter?

**Recall from [52 — Concurrency Fundamentals](./52-concurrency-fundamentals.md) that a mutex protects a critical section *within one process*.** That is exactly the limit of its power. A `Mutex` object in server A's heap is a few bytes of RAM in server A. Server B has no idea it exists. If both servers run `if (!lock.isHeld()) { lock.acquire(); chargeCard(); }`, **both will see an unlocked mutex** — their own — and both will charge the card.

What breaks without distributed locking:

- **Double charges.** Two workers pick up the same payment job from a queue. The customer pays twice.
- **Duplicate cron runs.** A "send weekly digest" job scheduled on every instance of a 10-pod deployment sends 10 emails to every user.
- **Overselling.** Two users click "Buy" on the last concert seat at the same millisecond; both get a confirmation; one shows up to a taken chair.
- **Corrupted state.** Two processes both do `read counter → add 1 → write counter`. Two increments, one result.

**In interviews:** Distributed locking shows up in *every* ticket-booking, payment, and inventory question. The naive answer is "use a Redis lock." The senior answer is "a lock alone cannot guarantee correctness — here's why, and here's what I'd actually do." That gap is the whole point of this document.

**At work:** You will reach for a distributed lock the first week you scale from one server to two. Getting it *slightly* wrong produces bugs that appear once a month, under load, in production, and never reproduce locally.

---

## The core idea — explained simply

### The Single Bathroom Key Analogy

An office has one bathroom and one key on a hook in the hallway.

**Version 1 — take the key.** You walk up, the key is there, you take it, you go in. Someone else arrives, sees the empty hook, waits. Works fine — until you take the key home in your pocket by accident. Now the hook is empty *forever* and nobody can ever use the bathroom again. **The system is wedged.**

**Version 2 — the key dissolves.** The office buys keys made of ice. If you keep one for more than 30 minutes, it melts and the hook magically gets a fresh key. Crashes no longer wedge the system. But now: you're in the bathroom, you've been in there 35 minutes, your key melted, someone else grabbed the new key — **and walks in on you.** The mutual exclusion you paid for is gone.

**Version 3 — numbered keys.** Every key is stamped with a number, higher each time: #1, #2, #3. You come out, go to hang your key back — but the hook holds key #2 now, not your #1. You must check the number before hanging it back, or you'd be returning *someone else's* key while they're still inside.

**Version 4 — the door checks the number too.** The bathroom door has a display showing the highest key number that has ever entered. If you walk up holding key #1 and the door says "highest seen: 2," the door refuses to open. Now even if you were frozen in time for an hour and woke up believing you still had the lock, **the door itself stops you.** This is the only version that is actually safe.

Mapping it back:

| Analogy | Technical concept |
|---|---|
| The key on the hook | A row/key in a shared store (Redis, etcd, DB) |
| Taking the key | `SET lock owner NX` — set only if not exists |
| Taking it home by accident | The lock holder crashes while holding the lock |
| The ice key melting | TTL / lease expiry on the lock |
| Being walked in on | Two holders at once — the lock's guarantee has failed |
| The number stamped on the key | A **fencing token** (monotonically increasing) |
| Checking the number before hanging it back | Compare-and-delete on release (must be atomic) |
| The door checking the number | The **protected resource** rejecting stale tokens |

**The lesson hiding in this analogy:** a lock is a *request* that others stay out. Only the door — the database, the storage bucket, the payment API — can actually *enforce* it.

---

## Key concepts inside this topic

### 1. Why local locks don't work — seeing it fail

Here is a lock that works perfectly on one machine and is worthless on ten.

```js
// BAD — a local, in-process lock. Correct on 1 server, useless on N.
class LocalLock {
  #held = false;
  #waiters = [];

  async acquire() {
    if (!this.#held) { this.#held = true; return; }
    await new Promise(resolve => this.#waiters.push(resolve));
    this.#held = true;
  }

  release() {
    this.#held = false;
    const next = this.#waiters.shift();
    if (next) next();
  }
}

const lock = new LocalLock();

async function processPayment(orderId) {
  await lock.acquire();               // <-- protects only THIS Node process
  try {
    const order = await db.orders.findById(orderId);
    if (order.status === 'PAID') return;      // "already done" check
    await stripe.charge(order.amount);        // side effect on the outside world
    await db.orders.update(orderId, { status: 'PAID' });
  } finally {
    lock.release();
  }
}
```

Run this on one server: safe. Deploy it to `api-1` … `api-10` behind a load balancer, and have two requests for order #4471 land on `api-3` and `api-7` simultaneously:

```
api-3: lock.#held === false  -> acquires ITS OWN lock  -> reads status=PENDING -> charges
api-7: lock.#held === false  -> acquires ITS OWN lock  -> reads status=PENDING -> charges
```

Both locks are "held." Neither knows about the other. The customer is charged twice. **The lock must live outside every process that competes for it.**

Three concrete jobs that need this:

1. **A cron job that must run once across a fleet.** Every pod has the same crontab. At 02:00, all 10 wake up. Only one may send the invoices.
2. **Only one worker per order.** A queue can deliver the same message twice (at-least-once delivery is the norm). Two workers must not both fulfill it.
3. **The last seat.** Two users, one seat, and a `read → check → write` sequence that is not atomic.

---

### 2. The naive Redis lock, and the four bugs inside it

Everyone's first distributed lock:

```js
// BUG-RIDDEN. Do not ship. We fix it bug by bug below.
async function withLock(key, fn) {
  const alreadyLocked = await redis.get(key);
  if (alreadyLocked) throw new Error('busy');
  await redis.set(key, '1');           // "I have the lock"
  try { return await fn(); }
  finally { await redis.del(key); }    // "I release it"
}
```

Even the `GET`-then-`SET` is a race (two clients can both read `null`), so let's be generous and assume we start from the atomic version, `SET key 1 NX` ("set if Not eXists"), which returns `OK` for the winner and `null` for the loser. Now walk the bugs.

---

#### Bug 1 — the holder crashes and the lock is held forever

```
t=0   Worker A: SET job:invoices 1 NX  -> OK. A holds the lock.
t=3   Worker A: process crashes / pod is OOM-killed / node is drained.
      The DEL never runs.
t=∞   Every other worker: SET ... NX -> null. Busy. Forever.
```

The invoice job never runs again — not tonight, not next week — until a human notices and manually deletes the key at 3am. This is a **wedge**: a permanent, self-inflicted outage.

**The fix: a TTL (a lease, not a lock).** Don't grant the lock forever, grant it for 30 seconds.

```js
// SET key value NX PX 30000
//   NX      -> only set if the key does not exist (this is the atomic "acquire")
//   PX 30000 -> auto-delete after 30,000 ms, even if we crash
const ok = await redis.set('job:invoices', '1', 'NX', 'PX', 30000);
```

**Critical detail:** this must be ONE command. This two-command version is a bug:

```js
// BAD — two round trips. If we crash between them, we're back to Bug 1.
await redis.setnx('job:invoices', '1');   // acquired...
// <-- process dies here. Key exists with NO expiry. Wedged forever.
await redis.expire('job:invoices', 30);
```

Redis guarantees `SET ... NX PX` is atomic; `SETNX` followed by `EXPIRE` is two separate operations with a fatal window between them. **Always use the single-command form.**

---

#### Bug 2 — the TTL expires while you're still working

You picked 30 seconds. Worker A's job usually takes 5. But today the database is slow, or the V8 garbage collector does a long stop-the-world pause, or the pod gets CPU-throttled by the Kubernetes scheduler.

```
t=0    A acquires lock, TTL = 30s.
t=2    A starts a slow DB query.
t=30   Lock expires in Redis. Nobody deleted it — Redis did.
t=31   B: SET ... NX -> OK. B legitimately holds the lock. B starts working.
t=33   A's query finally returns. A keeps going, believing it holds the lock.
       *** A and B are now BOTH in the critical section. ***
```

You built the lock precisely to prevent this, and it happened anyway. Note carefully: **nobody made a coding mistake here.** The TTL is a *guess* about how long the work takes, and guesses are wrong.

---

#### Bug 3 — you delete somebody else's lock

Continuing the timeline above, it gets worse:

```
t=35   A finishes. A calls: DEL job:invoices
       But that key now belongs to B! A just released B's lock.
t=36   C: SET ... NX -> OK. C acquires. Now B and C are both working.
```

One expired lease has now cascaded into an ongoing free-for-all. The bug is that `DEL` is unconditional — it doesn't ask *whose* lock this is.

**The fix: an owner token.** Store a unique random value with the lock, and only delete the key if the value still matches yours.

```js
import { randomUUID } from 'node:crypto';

const token = randomUUID();                                   // unique to THIS acquisition
await redis.set(key, token, 'NX', 'PX', 30000);

// BAD RELEASE — still broken! Read-then-delete is not atomic.
const current = await redis.get(key);
if (current === token) {
  // <-- the lock can expire RIGHT HERE, and someone else can acquire it,
  //     and then our DEL below kills their lock. Same bug, smaller window.
  await redis.del(key);
}
```

Shrinking a race window is not closing it. The compare and the delete must happen as **one atomic operation on the Redis server**, which means a Lua script (Redis runs scripts atomically, single-threaded, with nothing interleaved):

```lua
-- release.lua — delete the key ONLY if we still own it
if redis.call("GET", KEYS[1]) == ARGV[1] then
  return redis.call("DEL", KEYS[1])
else
  return 0
end
```

---

#### Bug 4 — long jobs need a watchdog

Bug 2 (the lease expiring under a slow worker) isn't solved by the token — the token only stops you from *stomping on* the new holder. To actually keep the lock while you're alive and working, run a **watchdog**: a background timer that extends the TTL every `ttl/3` ms, but only while you still own it.

```lua
-- extend.lua — push the expiry out ONLY if we still own the key
if redis.call("GET", KEYS[1]) == ARGV[1] then
  return redis.call("PEXPIRE", KEYS[1], ARGV[2])
else
  return 0
end
```

If the process dies, the timer dies with it, the TTL stops being renewed, and the lock expires naturally. That's the property you want: **liveness comes from the TTL, safety comes from the token.**

---

### 3. The full, correct Redis lock in Node.js

```js
import { randomUUID } from 'node:crypto';

const RELEASE_LUA = `
if redis.call("GET", KEYS[1]) == ARGV[1] then
  return redis.call("DEL", KEYS[1])
else
  return 0
end`;

const EXTEND_LUA = `
if redis.call("GET", KEYS[1]) == ARGV[1] then
  return redis.call("PEXPIRE", KEYS[1], ARGV[2])
else
  return 0
end`;

class RedisLock {
  /**
   * @param {import('ioredis').Redis} redis
   * @param {string} key      e.g. "lock:order:4471"
   * @param {number} ttlMs    the lease length. Long enough that a healthy worker
   *                          renews before it lapses; short enough that a crash
   *                          doesn't wedge the system for ages.
   */
  constructor(redis, key, { ttlMs = 30_000 } = {}) {
    this.redis = redis;
    this.key = key;
    this.ttlMs = ttlMs;
    this.token = null;      // proves WE are the owner; null means "not held"
    this.watchdog = null;
  }

  /** Try once. Returns true if we got it. */
  async tryAcquire() {
    const token = randomUUID();
    // ONE atomic command: set-if-absent AND set the expiry together.
    const res = await this.redis.set(this.key, token, 'NX', 'PX', this.ttlMs);
    if (res !== 'OK') return false;

    this.token = token;
    this.#startWatchdog();
    return true;
  }

  /** Retry with jittered backoff so 50 workers don't stampede in lockstep. */
  async acquire({ waitMs = 5_000, retryMs = 100 } = {}) {
    const deadline = Date.now() + waitMs;
    while (Date.now() < deadline) {
      if (await this.tryAcquire()) return true;
      const jitter = Math.random() * retryMs;   // jitter breaks up thundering herds
      await new Promise(r => setTimeout(r, retryMs + jitter));
    }
    return false;
  }

  /** Delete ONLY if we still own it. Atomic on the server. */
  async release() {
    if (!this.token) return false;
    this.#stopWatchdog();
    const token = this.token;
    this.token = null;                                  // release is one-shot
    const deleted = await this.redis.eval(RELEASE_LUA, 1, this.key, token);
    return deleted === 1;   // 0 means the lease had already lapsed — see below
  }

  #startWatchdog() {
    // Renew at 1/3 of the TTL: two renewals may fail before we actually lose it.
    this.watchdog = setInterval(async () => {
      if (!this.token) return;
      const ok = await this.redis
        .eval(EXTEND_LUA, 1, this.key, this.token, String(this.ttlMs))
        .catch(() => 0);
      if (ok !== 1) {
        // We LOST the lock (expired, or someone else holds it). We are now a
        // zombie. Stop renewing and make it visible — never keep working silently.
        this.token = null;
        this.#stopWatchdog();
      }
    }, Math.floor(this.ttlMs / 3));
    this.watchdog.unref?.();   // don't keep the Node event loop alive
  }

  #stopWatchdog() {
    if (this.watchdog) clearInterval(this.watchdog);
    this.watchdog = null;
  }

  /** True only if we believe we hold it. Belief is not proof — see section 4. */
  get isHeld() { return this.token !== null; }
}

/** Convenience wrapper: acquire, run, always release. */
export async function withLock(redis, key, fn, opts = {}) {
  const lock = new RedisLock(redis, key, opts);
  if (!(await lock.acquire(opts))) {
    throw new Error(`could not acquire lock: ${key}`);
  }
  try {
    return await fn(lock);          // pass the lock in so fn can check isHeld
  } finally {
    await lock.release();           // safe even if the lease already lapsed
  }
}
```

Usage:

```js
await withLock(redis, 'cron:daily-invoices', async (lock) => {
  for (const account of await getAccountsToBill()) {
    if (!lock.isHeld) throw new Error('lost the lock mid-job — aborting');
    await billAccount(account);
  }
}, { ttlMs: 30_000, waitMs: 0 });   // waitMs: 0 -> cron losers just exit quietly
```

That `if (!lock.isHeld) throw` inside the loop is the single most under-used line in distributed locking code. Check between units of work, not just at the start.

---

### 4. The unfixable problem — and why fencing tokens are the real answer

Here is the honest truth, and it's the most important paragraph in this document:

> **No lock with a timeout can guarantee mutual exclusion if the lock holder can be paused.**

And your lock holder *can* be paused. A V8 major GC can stop the world. A VM can be live-migrated. The kernel can deschedule your process. A network call to Redis can take 40 seconds and then succeed. Your Kubernetes node can be CPU-throttled to 5% for a minute. None of these are bugs — they are normal operating conditions.

```
     Worker A                        Redis                    Worker B
        │                              │                          │
 t=0    │──── SET lock A1 NX PX 30s ──▶│  lock=A1 (30s lease)     │
        │◀──────────── OK ─────────────│                          │
        │                              │                          │
 t=1    │  reads order, prepares       │                          │
        │  to write...                 │                          │
        │                              │                          │
 t=2    ╳══════════════════════════════════════════════╗          │
        ║   *** STOP-THE-WORLD GC PAUSE — 40 seconds ***║         │
        ║   A is frozen. It cannot renew. It cannot     ║         │
        ║   even know that time is passing.             ║         │
 t=32   ║                              │ lease EXPIRES  ║         │
        ║                              │ lock deleted   ║         │
 t=33   ║                              │◀── SET lock B1 NX PX 30s ┤
        ║                              │────── OK ───────────────▶│
        ║                              │                          │
 t=34   ║                              │        B is now the LEGITIMATE holder.
        ║                              │        B reads and writes. All correct.
        ║                              │                          │
 t=42   ╚══════════════════════════════════════════════╝          │
        │  A wakes up. Its last observation was "I hold the lock."│
        │  It has no way to know 40 seconds passed.               │
        │                              │                          │
 t=42   │────────────── WRITE (stale data, from t=1) ────────────────▶ DATABASE
        │                              │                          │
                *** A and B have both written. Corruption. ***
```

Checking `lock.isHeld` right before the write does **not** save you: the pause can land between the check and the write. This is a TOCTOU (time-of-check to time-of-use) gap, and you cannot close it from inside the client.

#### The fix: fencing tokens

Make the lock service hand out a **monotonically increasing number** with every grant, and make the **protected resource** — the database, the object store, the API — reject any write carrying a token lower than the highest it has already accepted.

```
 Lock grants:   A gets token 33.   B (later) gets token 34.

 t=42  A writes with token 33  ─▶  Storage: "highest seen = 34.  33 < 34.  REJECT."
 t=34  B writes with token 34  ─▶  Storage: "34 >= 34.  ACCEPT. highest = 34."
```

The zombie's write is killed **at the destination**, by a resource that has no idea a pause happened and doesn't need to. It's just comparing numbers.

```js
// Redis gives you monotonic numbers for free: INCR is atomic.
async function acquireWithFence(redis, key, ttlMs = 30_000) {
  const token = randomUUID();
  const ok = await redis.set(key, token, 'NX', 'PX', ttlMs);
  if (ok !== 'OK') return null;
  const fence = await redis.incr(`${key}:fence`);   // 1, 2, 3, ... never reused
  return { token, fence };
}

// The RESOURCE enforces the ordering. This is the part people skip.
// Conditional update: the write only lands if our fence is the newest one seen.
async function writeWithFence(db, orderId, fence, patch) {
  const { rowCount } = await db.query(
    `UPDATE orders
        SET status = $1, last_fence = $2
      WHERE id = $3
        AND (last_fence IS NULL OR last_fence <= $2)`,   // <-- the guard
    [patch.status, fence, orderId]
  );
  if (rowCount === 0) {
    // A newer holder has already written. We are a zombie. Do nothing.
    throw new Error(`fenced out: token ${fence} is stale for order ${orderId}`);
  }
}
```

**The takeaway to memorize: the lock alone is never enough. The protected resource must participate.** (This is Martin Kleppmann's argument in "How to do distributed locking," 2016 — and it has held up.)

If your resource *cannot* do this — say it's a third-party payment API with no version check — then you cannot get correctness from a lock, and you must get it somewhere else: an idempotency key on the request (see [85 — Idempotency](./85-idempotency.md)), which Stripe and every serious payments API support precisely because of this problem.

---

### 5. Redlock — the multi-node algorithm, and the famous fight about it

A single Redis master is a single point of failure. If it dies and a replica is promoted, the replica may not have received the lock key yet (Redis replication is asynchronous), so **two clients can hold the same lock across a failover**. Redlock, proposed by Redis's author antirez, tries to fix this with N independent Redis masters (typically 5, no replication between them):

1. Get the current time `T1`.
2. Try to `SET key token NX PX ttl` on all N masters, in sequence, with a short per-node timeout (say 50ms) so a dead node can't stall you.
3. Count the successes. You hold the lock **only if** you got a majority (`≥ N/2 + 1`, i.e. 3 of 5) **and** the total elapsed time `T2 - T1` is less than the TTL.
4. The lock's *effective* validity is `ttl - (T2 - T1) - clockDriftAllowance` — you must subtract the time acquisition itself burned.
5. If you failed, release on **all** N nodes (including ones you might have set), immediately.

```js
async function redlockAcquire(clients, key, ttlMs) {
  const token = randomUUID();
  const start = Date.now();
  const quorum = Math.floor(clients.length / 2) + 1;

  const results = await Promise.all(
    clients.map(c =>
      c.set(key, token, 'NX', 'PX', ttlMs).then(r => r === 'OK').catch(() => false)
    )
  );
  const votes = results.filter(Boolean).length;

  // Time spent acquiring is time already burned off the lease.
  const drift = Math.floor(ttlMs * 0.01) + 2;      // allowance for clock drift
  const validityMs = ttlMs - (Date.now() - start) - drift;

  if (votes >= quorum && validityMs > 0) return { token, validityMs };

  await Promise.all(clients.map(c => releaseOn(c, key, token).catch(() => {})));
  return null;
}
```

#### The critique, stated honestly

Kleppmann's objection is not "Redlock has a bug you can patch." It's that Redlock's safety **depends on timing assumptions that distributed systems do not provide**:

- **Bounded clock drift.** Step 3/4 assumes each node's clock ticks at roughly the same rate. But an NTP correction, a leap-second smear, or an admin running `date -s` can jump a clock forward — expiring a lock early on that node — and now a second client can win a majority while the first still legitimately holds it.
- **Bounded pauses.** The `validityMs` calculation assumes not much time passes between "I computed validity" and "I did the work." A 40-second GC pause makes that assumption false — and, as section 4 showed, **five Redis nodes don't fix a pause in the client.**
- **Bounded network delay.** A write that was sent before the lease expired can *arrive* after.

antirez's rebuttal is essentially: in practice, pauses that long are rare, and Redlock is far safer than a single Redis instance across failover. Both are right about what they're each measuring. The debate (2016, and still worth reading both posts in full) is famous because it forces the real question: **what are you using the lock for?**

#### The practical verdict — the only framing you need

| You're locking for... | Example | What a Redis/Redlock lock gives you | What you should do |
|---|---|---|---|
| **Efficiency** | Don't regenerate the same cached report twice; don't send a duplicate marketing email; don't re-crawl a URL | Good enough. A rare double-run costs you money/annoyance, not correctness. | A single Redis lock with TTL + token + watchdog. Ship it. |
| **Correctness** | Never double-charge; never oversell the last seat; never corrupt a balance | **Not enough. Ever.** Not even with 5 nodes. | A DB transaction with the right isolation, a UNIQUE constraint, or a conditional/CAS write — plus fencing tokens and idempotency. |

If a duplicate is *annoying*, lock. If a duplicate is *unacceptable*, the database must be the thing that says no.

---

### 6. ZooKeeper and etcd locks — sessions instead of TTL guesses

ZooKeeper (and etcd, with leases) offers a cleaner primitive. Instead of guessing a TTL, the client holds a **session** kept alive by heartbeats. The lock node is **ephemeral**: when the session dies — because the client crashed, or its heartbeats stopped — **ZooKeeper itself deletes the node.** No expiry guess, no watchdog you have to write.

The standard recipe uses **ephemeral sequential znodes**:

```
1. Create an EPHEMERAL_SEQUENTIAL node under /locks/order-4471/
   ZooKeeper appends a monotonically increasing counter:
      /locks/order-4471/lock-0000000017

2. List the children of /locks/order-4471/.
   If YOUR node has the LOWEST sequence number, you hold the lock.

3. Otherwise, set a WATCH on the node immediately before yours
   (0000000016 — not on all of them; that's the "herd effect" mistake)
   and wait for it to disappear.

4. Release = delete your node. The next in line gets a single
   notification and re-checks step 2.
```

```js
// Conceptual — using the `node-zookeeper-client` API shape
async function zkLock(zk, path) {
  const myNode = await zk.create(
    `${path}/lock-`,
    null,
    CreateMode.EPHEMERAL_SEQUENTIAL   // dies with the session; numbered for free
  );
  const mySeq = seqOf(myNode);        // e.g. 17

  for (;;) {
    const children = (await zk.getChildren(path)).map(seqOf).sort((a, b) => a - b);
    if (children[0] === mySeq) {
      // Lowest sequence number == the lock. And it doubles as a FENCING TOKEN.
      return { node: myNode, fence: mySeq };
    }
    const predecessor = children[children.indexOf(mySeq) - 1];
    await waitForDeletion(zk, `${path}/lock-${pad(predecessor)}`);  // watch ONE node
  }
}
```

Two things fall out of this design for free:

- **The sequence number IS a fencing token.** It's monotonically increasing by construction. Pass it downstream.
- **Fairness.** Waiters are served in arrival order (a FIFO queue), unlike Redis where retry timing decides the winner.

The cost: ZooKeeper/etcd are consensus systems (ZAB/Raft — see [83 — Leader Election](./83-leader-election.md)). Every write goes through a quorum, so a lock acquisition is milliseconds, not microseconds, and you now operate a 3- or 5-node cluster. That's a real price, and you pay it for correctness guarantees Redis doesn't offer.

**And even here, section 4 still applies.** ZooKeeper can expire your session while your paused process still thinks it holds the lock. The ephemeral node is *safer* than a TTL guess, not *immune*. Use the sequence number as a fence.

---

### 7. The best distributed lock is no lock at all

Most of the time you reach for a lock, you're using it to make a `read → check → write` sequence atomic. But your database **already** has atomic primitives that do that, without any of the pauses, clocks, or quorums above. Prefer them.

**a) A UNIQUE constraint — let the DB reject the duplicate.**

```js
// Instead of: lock, check if a payment exists, create it, unlock.
// Just try to insert, and let the database be the arbiter.
try {
  await db.query(
    `INSERT INTO payments (idempotency_key, order_id, amount) VALUES ($1, $2, $3)`,
    [idemKey, orderId, amount]              // UNIQUE INDEX on idempotency_key
  );
  await stripe.charge({ amount, idempotencyKey: idemKey });
} catch (e) {
  if (e.code === '23505') return;           // 23505 = unique_violation. Someone won. Fine.
  throw e;
}
```

No lock. No TTL. No zombie. The uniqueness is enforced by the storage engine, which cannot be GC-paused out of correctness.

**b) A conditional / compare-and-swap update — the last seat, done right.**

```js
// The "check" and the "write" are ONE statement. The DB does the locking,
// at row level, inside the transaction, for the microseconds it actually needs.
const { rowCount } = await db.query(
  `UPDATE events
      SET seats_left = seats_left - 1
    WHERE id = $1
      AND seats_left > 0`,                 // <-- the guard IS the mutual exclusion
  [eventId]
);
if (rowCount === 0) throw new SoldOutError();
```

Two users, one seat, ten servers: exactly one `UPDATE` gets `rowCount === 1`. There is no window. (Optimistic locking is the same idea with a version column: `UPDATE ... SET version = version + 1 WHERE id = $1 AND version = $2` — see [66 — Database Transactions and Isolation](./66-database-transactions-and-isolation.md).)

**c) Make the operation idempotent — so a duplicate is harmless.**

If running the job twice produces the same result as running it once, you don't need to prevent the second run; you just need to *absorb* it. That's [85 — Idempotency](./85-idempotency.md), and it's the most robust answer of the three because it doesn't require anything to be prevented at all.

**Reach for a distributed lock only when you genuinely cannot do any of the above** — typically when the protected resource is *not* a database you control: a third-party API with no idempotency key, a legacy system, a physical device, a fleet-wide cron with no natural uniqueness key.

---

## Visual / Diagram description

**Diagram 1 — where the lock lives.**

```
        ┌──────────┐   ┌──────────┐   ┌──────────┐
        │  api-1   │   │  api-3   │   │  api-7   │   <- 10 identical Node processes
        │          │   │          │   │          │
        │ Mutex ✗  │   │ Mutex ✗  │   │ Mutex ✗  │   <- local mutex: INVISIBLE to peers
        └────┬─────┘   └────┬─────┘   └────┬─────┘
             │              │              │
             │  SET lock:order:4471 <tok>  │
             │  NX PX 30000                │
             └──────────────┼──────────────┘
                            ▼
                  ┌───────────────────────┐
                  │        REDIS          │   <- the lock lives HERE, outside
                  │  lock:order:4471=tok  │      everyone. One winner, N losers.
                  │  (expires in 30s)     │
                  └───────────┬───────────┘
                              │  winner proceeds
                              ▼
                  ┌───────────────────────┐
                  │  POSTGRES  (orders)   │   <- the RESOURCE. It must also check
                  │  last_fence >= token? │      the fence, or a zombie gets through.
                  └───────────────────────┘
```

Read it top to bottom: the three servers each have their own useless local mutex; they all talk to one Redis, which picks exactly one winner; but the *last* box is the one that actually enforces safety when things go wrong.

**Diagram 2 — the zombie write, fenced out.**

```
  time ──▶
                    lease 33                    lease 34
  A  ══[acquire]════════════╳╌╌╌╌╌╌ PAUSED ╌╌╌╌╌╌▶[wake]──write(33)──▶ ✗ REJECTED
                            │                                            (33 < 34)
  Redis  lock=A ────────────┘ expires
                             lock=B ─────────────────────────────────────────
  B                          ═══[acquire]══════════════════[write(34)]─▶ ✓ ACCEPTED
                                                                          (highest=34)

  Storage's rule: accept a write only if fence >= highest fence ever seen.
```

Draw this on a whiteboard with two horizontal lifelines and a fat gap in A's. The pause is the whole story; the fence is the whole fix.

---

## Real world examples

### Redis / Redlock in the wild

Redis's own documentation ships the Redlock algorithm, and client libraries exist in most languages (`redlock` and `node-redlock` on npm). It is very widely used for *efficiency* locks: deduplicating cron runs across a Kubernetes deployment, preventing two workers from regenerating the same expensive cached report, single-flighting cache stampedes. The 2016 Kleppmann/antirez exchange is the canonical reading, and its practical conclusion is the table in section 5: use it where a rare duplicate is survivable.

### Kafka's consumer group protocol — locking without a lock

Kafka never asks a consumer to take a lock on a partition. Instead, the group coordinator *assigns* each partition to exactly one consumer in the group, and every assignment carries a **generation id** that increases on each rebalance. If a consumer is paused past its `max.poll.interval.ms`, it is kicked from the group and its partitions are reassigned. When the zombie wakes up and tries to commit an offset with the old generation id, **the broker rejects the commit.** That is a fencing token, in production, at enormous scale — and it's why Kafka's "exactly one consumer per partition" guarantee actually holds. (Its transactional producers use the same trick with an epoch number to fence zombie producers.)

### HDFS, and the bug that motivated the whole argument

Hadoop's HDFS NameNode high-availability setup famously had to solve exactly this: an old active NameNode, paused or partitioned, must not write to the shared edit log after a new one has taken over. The fix is fencing — the journal nodes reject writes from a stale epoch, and the design also supports crude out-of-band fencing (STONITH: literally power-cycling the old node, or `fuser`-killing the process). It's the same lesson in a different costume: **the shared resource has to be the one that says no.**

---

## Trade-offs

| Approach | Safety | Latency | Ops cost | Use when |
|---|---|---|---|---|
| **Single Redis lock** (NX PX + token + Lua release) | Weak — fails on failover, on pauses, on clock jumps | ~0.5–2 ms | Low (you likely already run Redis) | Efficiency locks: dedupe crons, single-flight caches |
| **Redlock (5 masters)** | Better than one node, but still assumes bounded clocks and pauses | ~2–10 ms | Medium (5 independent Redis nodes) | You need the lock to survive a Redis failure, and duplicates are still survivable |
| **ZooKeeper / etcd** (ephemeral sequential) | Strong: consensus-backed, session-based, gives you a fence for free | ~5–50 ms | High (operate a 3–5 node quorum) | Leader election, config, correctness-adjacent coordination |
| **DB UNIQUE constraint** | Strong — the storage engine enforces it | ~1–5 ms | Zero extra | Deduping: "this payment already exists" |
| **DB conditional / CAS update** | Strong — check and write are one atomic statement | ~1–5 ms | Zero extra | Counters, inventory, "the last seat," optimistic concurrency |
| **Idempotency** (no lock at all) | Strongest — duplicates simply don't matter | Zero | Design cost only | Anything you can make replay-safe |

| Design decision | You gain | You give up |
|---|---|---|
| Add a TTL | No permanent wedge on crash | Mutual exclusion under long pauses |
| Longer TTL | Fewer premature expiries | Slower recovery after a crash |
| Shorter TTL | Fast recovery | More zombie windows; more watchdog traffic |
| Add a watchdog | Long jobs keep their lock | A partitioned-but-alive client keeps renewing something it can't safely use |
| Add fencing tokens | Real safety, even against zombies | The downstream resource must be modified to check them |
| Move from Redis to ZooKeeper | Real consensus, sessions, free fences | 10× the latency and a cluster to operate |

**Rule of thumb:** Ask "if this runs twice, does someone lose money or a seat?" **No** → a single Redis lock with TTL + token + Lua release is correct enough; ship it and move on. **Yes** → the lock is an *optimization to reduce contention*, not your safety mechanism. Your safety must come from the database: a unique constraint, a conditional update, or a fencing token the resource checks. And whatever you build, make the operation idempotent so a duplicate is boring.

---

## Common interview questions on this topic

### Q1: "Why can't you just use a mutex? Your service already has one."

**Hint:** Because a mutex is memory inside one process. With 10 pods you have 10 independent mutexes, each of which happily reports "unlocked" to its own process. Mutual exclusion requires a *single* shared arbiter that all contenders can see, which means it must live outside all of them — Redis, ZooKeeper, or the database. Then immediately note that even a shared lock doesn't give you correctness on its own (Q3).

### Q2: "Walk me through building a Redis lock. What's wrong with `SET key 1` then `DEL key`?"

**Hint:** Enumerate the bugs in order, and fix each: (1) crash → held forever → add a TTL, using the *single* command `SET k v NX PX 30000` (`SETNX` + `EXPIRE` is two commands and re-opens the same hole). (2) TTL expires mid-job → two holders. (3) You then `DEL` someone else's lock → store a random token as the value and compare-and-delete in a **Lua script**, because a `GET` then `DEL` from the client races. (4) Long jobs → a watchdog that `PEXPIRE`s while you still own the key. Naming the bugs in this order is the answer; jumping straight to "use redlock" is not.

### Q3: "Your worker holds the lock, then GC-pauses for 40 seconds. What happens, and how do you fix it?"

**Hint:** The lease expires; another worker legitimately acquires; the first wakes up still believing it holds the lock and writes stale data. **No lock with a timeout can prevent this** — not Redlock, not ZooKeeper — because the client cannot detect that it was frozen. The fix is **fencing tokens**: the lock grants a monotonically increasing number, the worker passes it to the resource, and the resource rejects any write with a token lower than the highest it has seen. The lock alone is never enough; the protected resource must participate. Cite Kafka's generation id as a real implementation.

### Q4: "Is Redlock safe? Would you use it?"

**Hint:** Present it fairly (majority of N independent masters, subtract elapsed time from the validity), then the critique: its safety rests on bounded clock drift and bounded pauses, and distributed systems guarantee neither — an NTP jump or a long GC pause breaks it. Then give the verdict that matters: for **efficiency** (a duplicate is annoying) a plain Redis lock is fine and Redlock is overkill; for **correctness** (a duplicate is a double charge) *no* lock is sufficient, and you must use a DB transaction, a unique constraint, or a CAS write plus idempotency.

### Q5: "Two users book the last seat at the same instant. Design it."

**Hint:** The interviewer wants to see if you reach for a lock. Don't — reach for the database: `UPDATE events SET seats_left = seats_left - 1 WHERE id = $1 AND seats_left > 0` and check `rowCount`. Check-and-write in one atomic statement; exactly one user wins; no TTL, no zombie, no clock. Add a `UNIQUE(event_id, seat_id)` on the bookings table so a seat can never be double-assigned even under retries, and an idempotency key so the user's double-click doesn't create two bookings. Mention a Redis lock only as a *contention reducer* in front of a hot event, never as the safety mechanism. (See [110 — HLD: Ticketmaster](./110-hld-ticket-master.md).)

---

## Practice exercise

### Build a lock, then break it, then fence it

**Part 1 — build it (~15 min).** Implement the `RedisLock` class from section 3 against a local Redis (`docker run -p 6379:6379 redis`) using `ioredis`. It must have: `tryAcquire()` using a single `SET ... NX PX`, a Lua-scripted `release()` that only deletes if the token matches, and a watchdog that renews at `ttl/3`. Write a script that spawns 5 concurrent workers all trying to increment a `counter` key with a 200ms sleep between read and write. With the lock, the final counter must be exactly 5.

**Part 2 — break it (~10 min).** Set `ttlMs` to 1000 and **disable the watchdog**. Make the critical section take 3 seconds (`await sleep(3000)`). Run two workers. Log, from each worker, the moment it enters and exits the critical section, and add a `console.log` inside `release()` reporting whether the Lua script returned 1 or 0. You should observe: both workers in the critical section at once, and worker A's release returning **0** — proof that its lease had already lapsed and it correctly refused to delete B's lock. Now temporarily replace the Lua release with a plain `redis.del(key)` and watch A destroy B's lock. **Save both logs.** This is the bug, live.

**Part 3 — fence it (~15 min).** Add an `INCR lock:counter:fence` to acquisition. Replace the shared counter with a small object `{ value, lastFence }` and write a `writeFenced(fence, newValue)` function that **rejects** any write whose fence is lower than `lastFence`. Re-run the broken scenario from Part 2. The zombie's write must now be rejected.

**Produce:** the three log outputs, and a two-sentence answer to: *"Which part of my system actually guaranteed correctness — the lock, or the fence check?"*

---

## Quick reference cheat sheet

- **A local mutex is invisible to other processes.** With N servers you have N locks and zero mutual exclusion. The lock must live outside all contenders.
- **Always acquire with one atomic command:** `SET key <random-token> NX PX <ttl>`. `SETNX` + `EXPIRE` is two commands and a crash between them wedges the lock forever.
- **The TTL is for liveness, not safety.** It exists so a crash doesn't wedge the system — it does *not* mean your work finished in time.
- **Never `DEL` unconditionally.** Store a unique token as the value and release with a **Lua script** that compares then deletes atomically. A client-side `GET` then `DEL` races.
- **Watchdog:** renew at `ttl/3` while you still own the key, and if a renewal fails, *stop working* — you are a zombie.
- **No lock with a timeout survives a paused holder.** A GC pause, a VM migration, or CPU throttling can freeze you past your lease. This is unfixable at the lock layer.
- **Fencing tokens are the only real fix.** The lock hands out a monotonically increasing number; the *resource* rejects any write with a lower token than it has already seen.
- **The protected resource must participate.** If your DB/storage/API can't check a token or an idempotency key, a lock cannot give you correctness. Full stop.
- **Redlock** = acquire on a majority of N independent Redis masters within the TTL, and subtract elapsed time from the validity. Its safety assumes bounded clocks and bounded pauses; neither is guaranteed.
- **Efficiency vs correctness** is the framing that wins interviews: duplicate work is annoying (a lock is fine) vs duplicate work is a double charge (a lock is *never* enough).
- **ZooKeeper/etcd** use ephemeral sequential nodes: the session dies with the client (cleaner than a TTL guess), waiters are FIFO-fair, and the sequence number is a free fencing token. Cost: consensus latency and a cluster to run.
- **Watch the predecessor node, not all of them** — watching all children creates a thundering herd on every release.
- **Prefer no lock:** a `UNIQUE` constraint, a conditional update (`UPDATE ... WHERE seats_left > 0` / `WHERE version = ?`), or plain idempotency beats any distributed lock on safety, latency, and ops cost.
- **Reach for a distributed lock only when the resource isn't yours to guard** — a third-party API, a legacy system, a fleet-wide cron with no natural uniqueness key.

---

## Connected topics

| Direction | Topic |
|---|---|
| **Previous** | [83 — Leader Election](./83-leader-election.md) — the same coordination problem, one level up: electing a single leader *is* acquiring a long-lived distributed lock, and the systems that do it (ZooKeeper, etcd) are the ones you'd use here. |
| **Next** | [85 — Idempotency](./85-idempotency.md) — the technique that makes a duplicate execution *harmless*, which is strictly better than trying to prevent it. Read this next; it's the other half of the answer. |
| **Related** | [52 — Concurrency Fundamentals](./52-concurrency-fundamentals.md) — mutexes, race conditions, and critical sections inside one process. Everything here is that, minus shared memory, plus the network. |
| **Related** | [66 — Database Transactions and Isolation](./66-database-transactions-and-isolation.md) — where you should be getting your correctness from: `SERIALIZABLE`, `SELECT ... FOR UPDATE`, unique constraints, and optimistic version checks. |
| **Related** | [110 — HLD: Ticketmaster](./110-hld-ticket-master.md) — the canonical "two people, one seat" system, where every idea in this doc gets applied under real load. |
