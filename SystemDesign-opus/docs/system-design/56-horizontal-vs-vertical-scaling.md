# 56 — Horizontal vs Vertical Scaling
## Category: HLD Components

---

## What is this?

Your app is slow because too many people are using it. You have exactly two options: **make the machine bigger** (vertical scaling — "scale up") or **add more machines** (horizontal scaling — "scale out").

Think of a restaurant where the queue is out the door. Vertical scaling is hiring a superhuman chef who cooks four times faster and buying a bigger stove. Horizontal scaling is opening three more kitchens and hiring a host at the front to send each customer to whichever kitchen is free. Both feed more people. They fail in completely different ways, and they cost completely different amounts.

---

## Why does it matter?

Every scaling conversation in your career — and every system design interview — eventually lands on this fork in the road. "We're at 80% CPU, what do we do?" has exactly two answers, and choosing wrong is expensive.

**What breaks if you don't understand this:**
- You horizontally scale an app that keeps user sessions in local memory, and half your users get randomly logged out. (This is *the* classic production incident.)
- You vertically scale into the biggest instance your cloud offers, hit the ceiling on a Friday, and now have no move left.
- You over-engineer a startup MVP into 12 microservices behind an autoscaling group when a single $200/month server would have carried you to 100,000 users.

**Interview angle:** Interviewers use this to test whether you understand *statelessness*. If you say "just add more servers," the very next question is "okay — where do the user sessions live now?" If you don't have an answer, you fail the round.

**Real work angle:** This is your on-call decision tree. Vertical scaling is a 10-minute fix with 5 minutes of downtime. Horizontal scaling is an architecture change. Knowing which one the moment demands is a senior-engineer skill.

---

## The core idea — explained simply

### The Coffee Shop Analogy

You run a coffee shop. One barista, one espresso machine. You can serve 60 customers an hour. The queue is now 200 customers an hour. What do you do?

**Option A — Scale UP (vertical):**

You fire the barista and hire a world-champion barista who works twice as fast. You replace the single-group espresso machine with a triple-group commercial monster. Now you serve 180/hour.

What happened:
- The shop layout didn't change. One counter, one queue, one person.
- Nobody had to learn anything new. Same workflow, just faster.
- But you had to **close the shop for a day** to install the new machine.
- And the champion barista costs 6x the old one, not 2x.
- And when she calls in sick, **the shop is closed.** Zero coffee.
- And there is no barista alive who can do 400/hour. There is a ceiling.

**Option B — Scale OUT (horizontal):**

You keep the ordinary barista and the ordinary machine. You open three more identical counters, hire three more ordinary baristas, and put a **host at the door** who says "counter 2 is free, go there."

What happened:
- You now serve 240/hour, and you can reach 1,000/hour by adding more counters. **No ceiling in sight.**
- If one barista calls in sick, you serve at 75% capacity. **The shop stays open.**
- You can add a counter *while the shop is open*. No downtime.
- But now you have brand-new problems you never had before:
  - You need the **host** (a load balancer). If the host quits, chaos.
  - A customer who ordered at counter 1 and comes back to collect at counter 3 — counter 3 has never heard of them. **State is now a problem.**
  - The loyalty punch-card can't live in one barista's apron pocket anymore. It has to move to a **shared system** everyone can read.
  - Two baristas might both give away the last croissant. **Consistency is now a problem.**

That last block is the whole lesson. **Horizontal scaling doesn't just add machines — it converts your single-machine program into a distributed system, and distributed systems have entirely new categories of bugs.**

### Mapping the analogy

| Coffee shop | System design |
|---|---|
| One barista, one machine | One server process on one host |
| Champion barista + bigger machine | Vertical scaling: more CPU/RAM/faster disk |
| Closing the shop to install the machine | Downtime during instance resize/reboot |
| Barista calls in sick → shop closed | Single point of failure (SPOF) |
| "No barista can do 400/hour" | Hard ceiling: the largest instance your cloud sells |
| Four identical counters | Horizontal scaling: N identical app instances |
| The host at the door | Load balancer ([55 — Load Balancing](./55-load-balancing.md)) |
| Customer returns to a counter that doesn't know them | Session state stored in local memory — breaks |
| The shared loyalty-card system | Shared session/state store (Redis, DB) |
| Two baristas sell the same last croissant | Race condition / consistency across nodes |
| Adding a counter while open | Zero-downtime scaling |

---

## Key concepts inside this topic

### 1. Vertical scaling — and its four hard limits

Vertical scaling means: same machine count (one), bigger machine. You go from 4 vCPU / 16 GB RAM to 16 vCPU / 64 GB RAM. Your code does not change. Not one line. That is its superpower.

But it hits four walls, and you should be able to recite all four in an interview:

**Wall 1 — There is a physically largest machine.**
Cloud instances top out. On AWS today the general-purpose and memory-optimized ceilings are in the range of ~128 vCPU / ~1 TB RAM for common families, with specialty high-memory instances going into the multi-terabyte range. Whatever the exact number is when you read this, the point stands: **there is a number, and you can hit it.** When you do, you have no next step — you have to re-architect under pressure, which is the worst time to re-architect.

**Wall 2 — Cost grows superlinearly.**
Doubling the machine more than doubles the price, because the top-end SKUs carry a scarcity premium. Rough, illustrative on-demand shape (numbers vary by region/year — the *shape* is what matters):

```
 vCPU │ RAM   │ ~$/month │ $ per vCPU  │ notes
──────┼───────┼──────────┼─────────────┼─────────────────────────
   2  │   8GB │    ~$60  │   ~$30      │ cheap, linear zone
   4  │  16GB │   ~$120  │   ~$30      │ still linear
   8  │  32GB │   ~$250  │   ~$31      │ still basically linear
  16  │  64GB │   ~$500  │   ~$31      │ linear ends about here
  32  │ 128GB │  ~$1,000 │   ~$31      │
  64  │ 256GB │  ~$2,100 │   ~$33      │ premium creeping in
 128  │ 512GB │  ~$4,600 │   ~$36      │ premium is real
 448  │  12TB │ ~$80,000+│   ~$180     │ high-memory SKU: 6x $/vCPU
```

Read the `$ per vCPU` column. It is flat in the commodity range and then bends sharply upward at the top. **That bend is the economic argument for scaling out.** Eight 16-vCPU boxes give you 128 vCPU for ~$4,000; one 128-vCPU box costs ~$4,600 *and* is a single point of failure. Horizontal wins on both axes once you're past the knee.

But notice the flip side that juniors miss: **below the knee, vertical is cheaper**, because horizontal has fixed overheads (load balancer, extra ops, more monitoring, more failure modes).

**Wall 3 — It is still a single point of failure.**
A 128-vCPU machine that dies is exactly as dead as a 2-vCPU machine that dies. Vertical scaling buys you *capacity*, never *availability*. You cannot reach four nines of uptime on one box, no matter how big the box is.

**Wall 4 — Resizing requires downtime.**
Changing an EC2 instance type means: stop the instance, change type, start it. That's typically 1–5 minutes of hard downtime. You can hide it behind a blue/green swap, but that swap is itself a horizontal-ish trick (you briefly have two machines). Pure vertical scaling on one box = a maintenance window.

### 2. Horizontal scaling — and the tax it charges you

Horizontal scaling means: N identical machines, a load balancer in front, traffic spread across them.

The benefits are the mirror image of vertical's walls: **near-unlimited ceiling, no single point of failure, linear cost, and you can add capacity with zero downtime.**

The tax you pay is that you are now writing a distributed system. Concretely, four new problems appear the moment N > 1:

**a) You need a load balancer.** Something has to decide which instance handles each request. That's [55 — Load Balancing](./55-load-balancing.md). And the load balancer itself must not become the new SPOF — so it gets replicated too, usually via DNS or a managed LB (ALB/ELB/GCLB) that's redundant internally.

**b) You need service discovery.** With one server, its address was a constant in a config file. With an autoscaling group where instances are born and die every few minutes, "where are my healthy backends right now?" becomes a live question with a live answer. See [72 — Service Discovery](./72-service-discovery.md).

**c) You need to get state out of the process.** This is the big one — next section.

**d) You inherit consistency problems.** Two instances can now process two requests for the same user *at the same time*. Anything that was safe because "there's only one of me" — an in-memory counter, a `if (!exists) create()` check, a scheduled cron job — is now a race. You'll reach for [84 — Distributed Locking](./84-distributed-locking.md) and [85 — Idempotency](./85-idempotency.md).

### 3. Why horizontal scaling REQUIRES statelessness

Recall from [53 — Client-Server Architecture](./53-client-server-architecture.md) that a **stateless** server keeps no memory of previous requests from a client — every request carries everything needed to serve it. This isn't a stylistic preference. It is the *precondition* for horizontal scaling.

Here's the bug, in code. This works perfectly on one server and shatters on two:

```javascript
// ❌ BAD — state lives in the Node.js process's heap.
// Works flawlessly with 1 instance. Breaks randomly with 2+.
const sessions = new Map(); // <-- lives in THIS process only

app.post('/login', async (req, res) => {
  const user = await db.verifyPassword(req.body.email, req.body.password);
  const sessionId = crypto.randomUUID();
  sessions.set(sessionId, { userId: user.id, cart: [] });
  res.cookie('sid', sessionId).json({ ok: true });
});

app.get('/cart', (req, res) => {
  const session = sessions.get(req.cookies.sid);
  if (!session) return res.status(401).json({ error: 'Not logged in' }); // 💥
  res.json(session.cart);
});
```

What happens with two instances behind a round-robin load balancer:

```
Request 1: POST /login  → LB sends to Instance A → A stores session in ITS map → 200 OK
Request 2: GET  /cart   → LB sends to Instance B → B's map is empty          → 401 💥
```

The user logged in successfully and is immediately told they're not logged in. Refresh, and it works again (back on A). Refresh again, broken. This is the bug that makes engineers lose a weekend.

The fix is to move state to a place **all instances share**:

```javascript
// ✅ GOOD — state lives OUTSIDE the process, in a store every instance can reach.
import { createClient } from 'redis';
const redis = createClient({ url: process.env.REDIS_URL });
await redis.connect();

app.post('/login', async (req, res) => {
  const user = await db.verifyPassword(req.body.email, req.body.password);
  const sessionId = crypto.randomUUID();
  // TTL matters: without it, dead sessions accumulate in Redis forever.
  await redis.set(`sess:${sessionId}`, JSON.stringify({ userId: user.id, cart: [] }), { EX: 3600 });
  res.cookie('sid', sessionId, { httpOnly: true }).json({ ok: true });
});

app.get('/cart', async (req, res) => {
  const raw = await redis.get(`sess:${req.cookies.sid}`);
  if (!raw) return res.status(401).json({ error: 'Not logged in' });
  res.json(JSON.parse(raw).cart); // ✅ Any instance can serve this
});
```

Now every instance is **interchangeable**. That word is the whole point of horizontal scaling: *any request can be served by any instance*. If your instances aren't interchangeable, adding more of them doesn't help — it hurts.

**Where the state actually goes** (you should name these in an interview):

| Kind of state | Wrong home (breaks scale-out) | Right home |
|---|---|---|
| Sessions | in-process `Map` | Redis, or a signed JWT in the cookie (no server state at all) |
| Uploaded files | local disk `/uploads` | S3 / blob storage ([78](./78-blob-storage.md)) |
| Cache | in-process LRU | Redis / Memcached (or accept per-instance caches + short TTL) |
| Rate-limit counters | in-process counter | Redis `INCR` ([70](./70-rate-limiting.md)) |
| WebSocket connections | inherently per-instance | keep them, but add a Redis pub/sub backplane so instances can fan messages out to each other |
| Scheduled jobs / cron | `setInterval` in every instance (runs N times!) | a single scheduler, or a distributed lock ([84](./84-distributed-locking.md)) |

**The sticky-sessions escape hatch:** load balancers can pin a client to one instance (via cookie or IP hash) so in-memory state "works." It is a legitimate short-term band-aid, and it is a trap: you lose even load distribution, and when that instance dies, those users lose their state anyway. Treat it as debt, not a design.

### 4. Auto-scaling — the machinery that makes horizontal automatic

Auto-scaling watches a metric and adds or removes instances. The three things that actually matter:

**a) Pick the right metric.** CPU is the default and is often wrong for Node.js, because Node is single-threaded and frequently **I/O-bound** — it can be 100% saturated on latency while showing 30% CPU (it's just waiting on the DB). Better signals:

| Metric | Good for | Watch out for |
|---|---|---|
| CPU utilisation | CPU-bound work (image resizing, JSON crunching) | Misses I/O-bound saturation entirely |
| Requests per instance | Uniform, cheap requests | Wrong if one endpoint is 100x heavier |
| **p95 latency** | The thing users actually feel | Can be caused by a slow DB — scaling app servers won't help |
| Queue depth (SQS/Kafka lag) | **The best signal for worker fleets** | Requires a queue |
| Event-loop lag (Node-specific) | Detecting a blocked event loop | Needs custom instrumentation |

**b) Scale-up and scale-down must be ASYMMETRIC.** This is the point almost everyone misses.

- **Scale up FAST and AGGRESSIVELY.** The cost of being under-provisioned is dropped requests and angry users. Trigger quickly (e.g. 2 minutes above threshold), and add several instances at once, not one.
- **Scale down SLOWLY and TIMIDLY.** The cost of being over-provisioned is a few dollars. Wait a long time (e.g. 15 minutes below threshold), and remove one instance at a time.

If you make them symmetric, you get **flapping**: traffic dips, you kill 3 instances, load per instance jumps, you add 3 instances, they take 3 minutes to warm up, latency spikes, repeat forever. Long scale-down cooldowns are the cure.

**c) Warm-up time is real and it is the enemy.** From "scaling decision made" to "instance serving traffic at full speed":

```
  0s  ── autoscaler decides to add an instance
 +30s ── VM provisioned and booting        (container: ~2s)
 +60s ── OS up, container image pulled
 +75s ── Node process started, deps loaded
 +90s ── health check passes → LB starts sending traffic
+120s ── JIT warmed, connection pool filled, caches populated → actually fast
────────────────────────────────────────────────────────────────
 ~2 minutes from decision to useful. Your traffic spike happens in 10 seconds.
```

This is why you (1) scale on **leading** indicators, not lagging ones, (2) keep **headroom** — run at 50–60% utilisation, not 90%, so you can absorb the spike while the new instances boot, and (3) use **scheduled scaling** for predictable spikes (scale up at 08:45 because everyone logs in at 09:00 — don't wait for the metric).

A minimal health-check endpoint, so the load balancer only sends traffic to instances that are genuinely ready:

```javascript
let isReady = false;

// Liveness: "is the process alive?" — cheap, never touches dependencies.
// If this fails, the orchestrator RESTARTS the container.
app.get('/healthz', (_req, res) => res.status(200).send('ok'));

// Readiness: "should I get traffic?" — checks dependencies.
// If this fails, the LB just stops routing to me. No restart.
app.get('/readyz', async (_req, res) => {
  if (!isReady) return res.status(503).send('warming up');
  try {
    await Promise.all([db.query('SELECT 1'), redis.ping()]);
    res.status(200).send('ready');
  } catch {
    res.status(503).send('dependency down');
  }
});

async function warmUp() {
  await db.connect();          // fill the connection pool BEFORE taking traffic
  await redis.connect();
  await preloadHotConfig();    // whatever your first request would otherwise pay for
  isReady = true;
}
warmUp();

// Graceful shutdown — the other half of autoscaling.
// When the autoscaler scales DOWN, in-flight requests must not be killed.
process.on('SIGTERM', async () => {
  isReady = false;                      // fail readiness → LB drains me
  await new Promise(r => setTimeout(r, 10_000)); // let the LB notice (drain window)
  server.close(async () => {            // finish in-flight requests
    await db.end();
    process.exit(0);
  });
});
```

That `SIGTERM` handler is not optional. Without it, every scale-down event drops in-flight requests, and you'll see a mysterious 0.1% error rate that correlates with traffic *decreasing*.

### 5. Read replicas — scaling the database horizontally (sort of)

Your app tier scales out easily once it's stateless. The database is the hard part, because the database is *pure state*. The first and cheapest database scaling move is **read replicas**.

Most apps are read-heavy — a typical social or e-commerce workload is somewhere around **90–99% reads**. So: keep ONE primary that takes all writes, and add N replicas that stream changes from it and serve only reads.

```javascript
// Two connection pools: one primary for writes, a pool of replicas for reads.
const primary  = new Pool({ host: 'db-primary.internal' });
const replicas = [
  new Pool({ host: 'db-replica-1.internal' }),
  new Pool({ host: 'db-replica-2.internal' }),
];
let rr = 0;
const pickReplica = () => replicas[rr++ % replicas.length];

export async function read(sql, params)  { return pickReplica().query(sql, params); }
export async function write(sql, params) { return primary.query(sql, params); }

// ⚠️ The read-your-own-writes trap:
async function createPostBroken(userId, body) {
  await write('INSERT INTO posts(user_id, body) VALUES($1,$2)', [userId, body]);
  // Replication lag is typically 10–500ms. This read may hit a replica
  // that hasn't received the INSERT yet → the user posts and doesn't see their post.
  return read('SELECT * FROM posts WHERE user_id=$1 ORDER BY id DESC', [userId]);
}

// ✅ Fix: for a short window after a user writes, route THAT user's reads to the primary.
const recentWriters = new Map(); // in prod this is Redis, not a Map (see §3!)
async function createPost(userId, body) {
  const row = await write(
    'INSERT INTO posts(user_id, body) VALUES($1,$2) RETURNING *', [userId, body]
  );
  recentWriters.set(userId, Date.now() + 2000); // 2s > expected replication lag
  return row;
}
async function readForUser(userId, sql, params) {
  const until = recentWriters.get(userId) ?? 0;
  return Date.now() < until ? primary.query(sql, params) : pickReplica().query(sql, params);
}
```

**What read replicas buy you:** read throughput scales roughly linearly with replica count, plus a hot standby for failover.

**What they do NOT buy you:** write throughput. Every replica must apply every write, so a write-saturated primary is not helped at all — *adding replicas makes the primary's job slightly harder.* When writes are the bottleneck, replicas are the wrong tool and you need [64 — Database Sharding](./64-database-sharding.md). Knowing that distinction is the difference between a mid-level and a senior answer.

### 6. The honest advice: vertical scaling is badly underrated

Junior engineers dramatically underestimate one machine. A single modern server — say 32 vCPU, 128 GB RAM, NVMe SSD — running a well-written Node.js service, will comfortably do:

- **10,000–50,000 simple HTTP requests/second** across a cluster of Node workers (`node:cluster`, one worker per core — this is *vertical* scaling done right: use the cores you already paid for).
- **Postgres on that box:** tens of thousands of simple indexed reads/second, thousands of writes/second.
- **Redis on that box:** ~100,000+ ops/second on a single core.

Do the arithmetic that interviewers love:

```
1,000,000 daily active users
× 20 requests each per day            = 20,000,000 requests/day
÷ 86,400 seconds/day                  = ~230 requests/second average
× 3 (peak is ~3x average)             = ~700 requests/second at peak
```

**700 req/s.** One decent box, correctly configured, handles that without breathing hard. A million DAU does not require a fleet. It requires one server, one database, and a cache.

So the sane progression is:

```
1. Make the code faster.          (add the missing index. Free. Do this first, always.)
2. Add a cache.                   (Redis. Cheap. 10x on read-heavy paths.)
3. Scale VERTICALLY.              (2 clicks. No code change. Buys you 4–8x.)
4. Scale HORIZONTALLY (stateless app tier + LB). (Now you're a distributed system.)
5. Read replicas.                 (Scales reads.)
6. Shard.                         (Last resort. Genuinely hard. Avoid until forced.)
```

Most companies never get past step 4. **Stack Overflow famously served enormous traffic from a handful of very large, mostly hand-tuned servers** — a public, well-documented demonstration that vertical scaling plus good engineering goes much further than the microservices discourse suggests.

**But say this in the interview too:** you scale horizontally *before* you're forced to, for one reason that has nothing to do with capacity — **availability**. Even at 100 req/s, you want at least 2 instances so that one dying doesn't take you down. The first "horizontal" step is often for redundancy, not throughput. Two instances at 20% CPU each is a perfectly rational design.

---

## Visual / Diagram description

### Diagram 1 — Vertical scaling (scale UP)

```
        BEFORE                                AFTER
        ══════                                ═════

   ┌───────────┐                        ┌───────────┐
   │  Clients  │                        │  Clients  │
   └─────┬─────┘                        └─────┬─────┘
         │                                    │
         │ 700 req/s                          │ 2,800 req/s
         ▼                                    ▼
  ┌──────────────┐                    ┌──────────────────────┐
  │   SERVER     │                    │       SERVER          │
  │              │   resize + reboot  │                       │
  │  4 vCPU      │  ═══════════════▶  │   32 vCPU             │
  │  16 GB RAM   │   (~3 min DOWN)    │   128 GB RAM          │
  │  ~$120/mo    │                    │   ~$1,000/mo          │
  └──────┬───────┘                    └──────────┬────────────┘
         │                                       │
         ▼                                       ▼
   ┌───────────┐                          ┌───────────┐
   │    DB     │                          │    DB     │
   └───────────┘                          └───────────┘

  ▲ Same architecture. Zero code changes.
  ▲ Still ONE box → if it dies, you are 100% down.  ← SPOF
  ▲ There IS a biggest box. When you reach it, this path ends.
```

### Diagram 2 — Horizontal scaling (scale OUT)

```
                      ┌───────────┐
                      │  Clients  │
                      └─────┬─────┘
                            │ 2,800 req/s
                            ▼
                 ┌──────────────────────┐
                 │    LOAD BALANCER     │   ◀── new component (55)
                 │  round-robin +       │       must itself be HA
                 │  health checks       │
                 └──┬────────┬────────┬─┘
          ~700/s    │        │        │   ~700/s
         ┌──────────┘        │        └──────────┐
         ▼                   ▼                   ▼
  ┌────────────┐      ┌────────────┐      ┌────────────┐
  │  App #1    │      │  App #2    │      │  App #3    │  ... #4
  │  4 vCPU    │      │  4 vCPU    │      │  4 vCPU    │
  │ STATELESS  │      │ STATELESS  │      │ STATELESS  │  ◀── interchangeable
  │  ~$120/mo  │      │  ~$120/mo  │      │  ~$120/mo  │
  └─────┬──────┘      └─────┬──────┘      └─────┬──────┘
        │                   │                   │
        └─────────┬─────────┴─────────┬─────────┘
                  │                   │
                  ▼                   ▼
        ┌──────────────────┐  ┌─────────────────────────────┐
        │  Redis           │  │        DATABASE              │
        │  sessions,cache, │  │  ┌─────────┐   replication   │
        │  rate limits     │  │  │ PRIMARY │──┬──────────┐   │
        │                  │  │  │(writes) │  │          │   │
        │  ◀── the state   │  │  └─────────┘  ▼          ▼   │
        │      that used   │  │            ┌──────┐  ┌──────┐│
        │      to live in  │  │            │Replica│ │Replica│
        │      the process │  │            │(reads)│ │(reads)│
        └──────────────────┘  │            └──────┘  └──────┘│
                              └─────────────────────────────┘

  ▲ 4 × $120 = $480/mo for the same 128 vCPU that cost $1,000 vertically.
  ▲ One app instance dies → you run at 75%. STILL UP.  ← no SPOF in the app tier
  ▲ Add instance #5 with ZERO downtime.
  ▲ Ceiling: effectively none (until the DB becomes the bottleneck — it will).
  ▲ COST: you now own an LB, a shared state store, service discovery,
    and every consistency bug in the distributed-systems textbook.
```

**How to read these:** the vertical diagram has one box that gets fatter. The horizontal diagram *grows new box types* — the load balancer and the shared state store didn't exist before. That's the entire trade: vertical changes a number, horizontal changes the shape of your architecture.

---

## Real world examples

### 1. Stack Overflow — the vertical-scaling poster child

Stack Overflow has been unusually public about its architecture, and for years it served a top-50 website from a strikingly small number of physical servers — a handful of very powerful web servers and a small set of extremely large SQL Server boxes with enormous RAM (hundreds of GB), so that the working set of the database sat entirely in memory.

The lesson their engineers repeatedly emphasise: **they optimised the code and the queries, then bought big machines, and mostly stopped there.** They chose vertical scaling deliberately because it kept the system simple enough for a small team to reason about — no distributed consensus, no sharding, no eventual consistency. It is the strongest available argument against reflexive scale-out.

### 2. Netflix — the horizontal-scaling poster child

Netflix runs on AWS with thousands of stateless service instances behind autoscaling groups. Their traffic is enormously predictable and enormously spiky (evenings, weekends, new-release drops), so they scale out and back in continuously.

Two concrete practices worth naming:
- Their services are built to be **disposable** — any instance can vanish at any moment. They enforced this with **Chaos Monkey**, a tool that randomly terminates production instances during business hours, so that "an instance died" is a boring non-event rather than an incident.
- They **pre-scale** for known events rather than reacting to metrics, precisely because of the warm-up problem: by the time CPU crosses a threshold, the spike has already hurt users.

### 3. A typical B2B SaaS — the boring, correct middle

The overwhelmingly common real architecture, and the one you'll actually build:

- **2–4 stateless Node.js app instances** behind a managed load balancer (ALB). Two is the minimum — not for throughput, for redundancy.
- **One vertically-scaled managed Postgres** (e.g. 8–16 vCPU) with **one read replica**, which doubles as the failover standby.
- **One Redis** for sessions, cache, and rate limits.
- Autoscaling configured but rarely firing.

Total: a few hundred dollars a month, serves millions of requests a day, and one engineer can hold the whole thing in their head. **Vertical where it's easy (the DB), horizontal where it's cheap (the app tier).** That hybrid is the real answer to this topic, and saying it out loud in an interview signals maturity.

---

## Trade-offs

| Dimension | Vertical (scale UP) | Horizontal (scale OUT) |
|---|---|---|
| **Code changes needed** | None | Must be stateless — sometimes a big refactor |
| **Operational complexity** | Very low | High: LB, discovery, shared state, distributed bugs |
| **Ceiling** | Hard — the biggest instance sold | Effectively unlimited (app tier) |
| **Failure impact** | Total outage (SPOF) | Degraded capacity, still serving |
| **Downtime to scale** | Yes — resize needs a reboot | No — add an instance live |
| **Cost at small scale** | Cheaper (no LB, no extra ops) | More expensive (fixed overheads) |
| **Cost at large scale** | Much more expensive (superlinear premium) | Cheaper (commodity boxes, linear) |
| **Speed to execute** | Minutes | Days–weeks (first time), then minutes |
| **Debuggability** | Easy — one process, one log stream | Hard — correlate logs across N instances |
| **Best fit** | Databases, stateful systems, small/medium apps | Stateless web/API tiers, workers, anything spiky |

| What you give up by choosing... | The price |
|---|---|
| **Vertical** | Availability (SPOF), a future (the ceiling), zero-downtime scaling |
| **Horizontal** | Simplicity, strong consistency by default, easy debugging, low ops cost |

**Rule of thumb:** Scale **vertically** until it hurts — it's a config change, not an architecture change, and one modern box goes far further than you think. Scale **horizontally** when you hit any one of: (a) the instance-size ceiling, (b) the cost knee where $/vCPU bends upward, or (c) an availability requirement that a single box can't meet — and in practice (c) arrives first, which is why even small apps run 2 instances. **The real-world answer is almost always hybrid: horizontal stateless app tier, vertically-scaled database, with read replicas.**

---

## Common interview questions on this topic

### Q1: "Your API is at 90% CPU. Walk me through your options."
**Hint:** Don't jump to "add servers." Go in order: (1) *Is it the code?* Profile first — a missing DB index or an accidental O(n²) is free to fix. (2) *Can I cache?* Read-heavy endpoints often drop 10x. (3) *Scale vertically* — a 2-click, no-code-change 4x. (4) *Scale horizontally* — but first answer "is my app stateless?" (5) Then check whether the **DB** is actually the bottleneck, because scaling app servers against a saturated DB just adds more clients hammering the same overloaded database — it makes things *worse*. Naming that last trap is what separates a strong answer.

### Q2: "What has to be true about your app before you can horizontally scale it?"
**Hint:** It must be **stateless** — no sessions in process memory, no uploads on local disk, no in-memory rate-limit counters, no in-memory caches you depend on for correctness, no `setInterval` cron jobs (they'd run N times). All of that state moves to a shared store (Redis / S3 / the DB) or into the request itself (a signed JWT). The test: "can any instance serve any request, and can I kill any instance at random without a user noticing?"

### Q3: "Why not just always scale horizontally? Isn't it strictly better?"
**Hint:** No. Horizontal scaling converts a simple program into a **distributed system**, and you inherit the entire distributed-systems problem set: consistency, partial failure, clock skew, race conditions, distributed tracing, and a much larger ops surface. Below the cost knee it's also *more* expensive. And some things simply don't scale out easily — a relational DB's write path is the classic example, which is why even the most horizontal companies still run a big vertical primary.

### Q4: "What metric would you auto-scale a Node.js API on, and why not CPU?"
**Hint:** CPU is a poor signal for Node because Node is single-threaded and usually **I/O-bound** — it can be saturated on latency while showing 30% CPU (it's blocked waiting on the DB). Prefer **p95 latency**, **requests-per-instance**, or **event-loop lag**. For worker fleets, **queue depth / consumer lag** is the best signal of all. Then add: scale up fast and aggressively, scale down slowly, or you'll flap — and remember instances take ~2 minutes to warm up, so keep headroom and scale on leading indicators.

### Q5: "You added 5 read replicas and the site is still slow. What happened?"
**Hint:** Read replicas scale **reads only**. If the bottleneck is the **write** path, replicas don't just fail to help — each replica must replay every write, so you've added load to the primary. Diagnose the read/write mix first. If writes are the bottleneck, the tools are: batch/queue the writes, add a write-through cache, or shard ([64](./64-database-sharding.md)). Bonus point: also mention **replication lag** causing read-your-own-writes bugs, and pinning a recent writer's reads to the primary for a couple of seconds.

---

## Practice exercise

### The Scaling Decision Memo

You're the engineer on call for a Node.js e-commerce API. Current state:

- **1 app server:** 4 vCPU / 16 GB, running Express. Sessions in an in-process `Map`. Product images written to `/var/www/uploads`. A `setInterval` that emails abandoned carts every hour.
- **1 Postgres:** 8 vCPU / 32 GB. Currently 40% CPU.
- Traffic: 500,000 DAU, ~15 requests/user/day. App server is at **85% CPU at peak** and p95 latency has gone from 120ms to 900ms.
- Black Friday is in 6 weeks and marketing promises **5x traffic**.

**Produce a one-page memo containing:**

1. **The math.** Compute current average req/s and peak req/s from the DAU numbers (show every step). Then compute the Black Friday peak. State the target req/s you must support.
2. **A vertical-only plan.** Which instance size, what it costs, how much downtime the resize needs, and — critically — **what would still be broken** after doing it.
3. **A horizontal plan.** List *every* piece of state in the current app that must move before you can run 2+ instances, and say exactly where each one goes. (There are at least four. The `setInterval` one is the sneaky one — what happens when it runs on 4 instances?)
4. **Your actual recommendation**, given that you have 6 weeks. Defend the choice on three axes: risk, cost, and time-to-deliver.
5. **Draw both target architectures** as ASCII diagrams, in the style of the two diagrams above.

**What good looks like:** a strong answer notices the DB is only at 40% (so it is *not* the bottleneck — yet), does the 5x math on the DB too to check whether it *becomes* the bottleneck, and proposes a hybrid: make the app stateless, run 4–6 instances behind an LB, and vertically bump Postgres one size as insurance. Time-box it: 30–40 minutes.

---

## Quick reference cheat sheet

- **Vertical = scale UP** = bigger machine. **Horizontal = scale OUT** = more machines.
- **Vertical's four walls:** hard size ceiling, superlinear cost at the top end, still a single point of failure, needs downtime to resize.
- **Horizontal's tax:** you now own a load balancer, service discovery, shared state, and every distributed-systems bug there is.
- **Horizontal REQUIRES statelessness.** Sessions in a process `Map` is the #1 bug when you add the second instance.
- **State's real homes:** sessions → Redis or JWT; files → S3; cache → Redis; counters → Redis `INCR`; cron → one scheduler or a distributed lock.
- **Sticky sessions** make in-memory state "work." They are debt, not a design.
- **Vertical is underrated.** One 32-vCPU box handles thousands of req/s. 1M DAU ≈ 700 req/s at peak — that is *one server*.
- **Order of moves:** fix the code → cache → scale up → scale out → read replicas → shard (last resort).
- **You still run 2+ instances at low traffic** — not for throughput, for **availability**. That's the honest reason most apps go horizontal early.
- **Auto-scaling is asymmetric:** scale up fast and big, scale down slow and small, or you flap.
- **Don't autoscale Node on CPU alone** — it's I/O-bound. Use p95 latency, req/instance, event-loop lag, or queue depth.
- **Warm-up is ~2 minutes** from decision to useful. Keep headroom; pre-scale for known spikes.
- **Graceful SIGTERM shutdown** is mandatory or every scale-down drops in-flight requests.
- **Read replicas scale reads, never writes.** Beware replication lag → read-your-own-writes bugs.
- **The real answer is hybrid:** horizontal stateless app tier + vertically-scaled DB + read replicas.

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [55 — Load Balancing](./55-load-balancing.md) — the component that makes horizontal scaling possible in the first place |
| **Next** | [57 — Reverse Proxy](./57-reverse-proxy.md) — the thing sitting in front of your scaled-out fleet, doing TLS, caching, and routing |
| **Related** | [53 — Client-Server Architecture](./53-client-server-architecture.md) — where statelessness is defined; the precondition for scaling out |
| **Related** | [64 — Database Sharding](./64-database-sharding.md) — how you scale the one tier that refuses to scale out easily |
