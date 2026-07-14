# 55 — Load Balancing — Algorithms, L4 vs L7, Health Checks
## Category: HLD Components

---

## What is this?

A **load balancer** (LB) is a piece of software or hardware that sits in front of a group of identical servers and decides, for every incoming request, *which server gets it*. To the client, there is one address. Behind that address, there might be 3 servers or 300.

Think of a busy restaurant. Guests don't wander in and pick a waiter at random — they queue at the door, and a **host** seats them. A good host knows which waiter has four tables and which has one, knows that Table 12's waiter just went on break, and seats you accordingly. The load balancer is that host: **one door, many waiters, an informed decision on every guest.**

The three jobs it does, in order of importance:

1. **Distribute load** — spread traffic across servers so no single one melts.
2. **Remove dead servers from rotation** — if a server stops responding, stop sending it traffic. Users never see it.
3. **Enable zero-downtime deploys** — pull one server out, upgrade it, put it back, repeat. The site never goes down.

Most beginners only know job #1. Jobs #2 and #3 are what actually make the load balancer indispensable.

---

## Why does it matter?

**Without a load balancer, you cannot scale horizontally at all.** Recall from [56 — Horizontal vs Vertical Scaling](./56-horizontal-vs-vertical-scaling.md) that scaling out means adding more machines. But adding a second machine is useless if nothing sends it traffic. The load balancer is the thing that *makes* five servers behave like one big one.

What breaks without one:

- **One server dies → your site is down.** With an LB, one server dies and the LB stops routing to it within a few seconds. Nobody notices.
- **You cannot deploy without downtime.** With a single server, `npm restart` = 30 seconds of 502 errors. With an LB, you drain traffic from server A, deploy, re-add it, then do server B.
- **A traffic spike takes you down.** You can't add capacity mid-spike if there's no front door to add it behind.
- **A slow server poisons the whole system.** Without health checks, a server that's alive-but-broken (DB connection dead, event loop blocked) keeps receiving requests and keeps failing them.

**Interview angle:** Load balancing appears in *literally every* HLD interview. The moment you draw more than one app server, the interviewer expects an LB in front of it — and expects you to say *which algorithm* and *L4 or L7* and *how health checks work*. The follow-up "what if the load balancer itself dies?" separates people who memorized a diagram from people who understand it.

**Real-work angle:** You will configure Nginx, HAProxy, an AWS ALB, or a Kubernetes Service. You will debug why one pod is getting 80% of traffic. You will get paged because a health check endpoint returned 200 while the app couldn't reach the database. These are Tuesday problems, not theory.

---

## The core idea — explained simply

### The Restaurant Host Analogy

Picture a restaurant with 5 waiters. Every waiter is identical in skill (your servers run the same code). Guests arrive at the door continuously.

**A bad host** seats guests strictly in a circle: waiter 1, 2, 3, 4, 5, 1, 2, 3… This is fine if every table takes the same effort. But a table of 12 ordering a tasting menu is not the same as a solo diner ordering coffee. Rotate blindly and waiter 3 ends up with three tasting menus while waiter 5 pours coffee all night.

**A good host** looks at the floor: "Waiter 5 only has one table — you go there." That's *least connections*.

**A great host** also knows things the bad host doesn't:
- Waiter 2 just started their shift and is still finding their feet → send them fewer tables at first (**slow start**).
- Waiter 4 hasn't picked up an order in ten minutes and isn't answering → stop seating them entirely and tell the manager (**health checks + ejection**).
- Waiter 4 is back → don't dump 30 tables on them at once; ease them in (**slow start on re-entry**).
- The couple in the corner had waiter 1 last time and asked for them again → seat them with waiter 1 (**sticky sessions**) — but if waiter 1 quits, that couple's whole tab is lost.

And the deepest question: **what if the host quits?** Nobody gets seated, and the restaurant is closed even though all five waiters are perfectly fine. That's the LB as a single point of failure.

### Mapping the analogy

| Restaurant | System |
|---|---|
| The host at the door | Load balancer |
| Waiters | Backend servers / app instances |
| Guests | Incoming requests |
| Tables currently being served | Active connections per server |
| "Waiter, are you okay?" every 5 seconds | Active health check |
| Waiter dropped 3 orders in a row | Passive health check (error-rate based) |
| Taking a waiter off the floor | Ejecting a server from the pool |
| Easing a returning waiter back in | Slow start / connection draining |
| "Same waiter as last time, please" | Sticky session |
| Host quits, restaurant closed | LB single point of failure |
| A second host on standby with the same clipboard | Active-passive LB pair with a floating IP |

The whole topic is: **how does the host decide, how does the host know a waiter is sick, and who covers for the host?**

---

## Key concepts inside this topic

### 1. The algorithms — how the LB picks a server

This is the part interviewers actually drill. For each one: how it works, when it wins, how it fails.

**Round robin.** Cycle through servers in order: 1, 2, 3, 1, 2, 3…

```js
// The simplest LB algorithm in existence.
class RoundRobin {
  constructor(servers) {
    this.servers = servers;
    this.i = 0;
  }
  pick() {
    const s = this.servers[this.i % this.servers.length];
    this.i++;
    return s;
  }
}
```

*Wins when:* servers are identical AND requests cost roughly the same (static files, simple reads). It's O(1), stateless, and trivially correct.

*Fails when:* requests have different costs — which is almost always true. `GET /health` and `POST /reports/generate` both count as "one request". If server 2 happens to catch three report generations, round robin keeps feeding it more work anyway, because it counts *turns*, not *load*. The distribution is even in **request count** and wildly uneven in **actual work**.

**Weighted round robin.** Give each server a weight; a server with weight 3 gets 3 turns for every 1 turn of a weight-1 server.

```js
class WeightedRoundRobin {
  // servers: [{ url, weight }]
  constructor(servers) {
    // Expand into a flat list once: weight 3 => appears 3 times.
    // Fine for small pools; real LBs use smooth WRR to avoid bursty runs.
    this.slots = servers.flatMap(s => Array(s.weight).fill(s.url));
    this.i = 0;
  }
  pick() {
    return this.slots[this.i++ % this.slots.length];
  }
}
```

*Wins when:* your fleet is **heterogeneous** — you have three 16-core boxes and two 4-core boxes, or you're mid-migration with old and new instance types. Also used for **canary deploys**: give the new version weight 1 and the old version weight 99, so 1% of traffic hits the canary.

*Fails when:* weights are static guesses. A 16-core box that's swapping to disk is now *slower* than the small one, and the weight doesn't know that.

**Least connections.** Send the request to whichever server currently has the fewest open connections.

```js
class LeastConnections {
  constructor(urls) {
    this.active = new Map(urls.map(u => [u, 0])); // url -> in-flight count
  }
  pick() {
    let best = null, min = Infinity;
    for (const [url, n] of this.active) {
      if (n < min) { min = n; best = url; }
    }
    this.active.set(best, min + 1);
    return best;
  }
  release(url) {
    // Called when the response completes — this is what makes it adaptive.
    this.active.set(url, Math.max(0, this.active.get(url) - 1));
  }
}
```

*Wins when:* request durations vary — which, again, is most real workloads. A server stuck on three 10-second report queries has 3 open connections and simply stops being picked. It **self-corrects** without you telling it anything. This is the sane default for most APIs.

*Fails when:* connections aren't a good proxy for load. With HTTP/2 or gRPC, one connection can carry hundreds of multiplexed streams, so "1 connection" might mean 200 in-flight requests. (Modern LBs offer *least requests* / *least outstanding requests* for exactly this.) It also fails right after a server restarts: the fresh server has 0 connections and gets **slammed** with everything at once — which is why slow start exists.

**Least response time.** Pick the server with the lowest (active connections × recent average latency), or simply the fastest recent response time.

*Wins when:* servers degrade gradually rather than dying — a box with a noisy neighbor, a node with a slow disk. Connection count won't reveal that; latency will.

*Fails when:* it creates a feedback loop. Fast server gets more traffic → becomes slower → traffic shifts away → it gets fast again → traffic floods back. This is called **herding** or oscillation. It also needs the LB to track per-server latency stats, which costs memory and CPU.

**IP hash / hash-based routing.** Hash something stable about the request — usually the client IP, sometimes a URL path or a user ID header — and mod it by the number of servers.

```js
function ipHashPick(clientIp, servers) {
  // Deterministic: same IP always lands on the same server (while the pool is stable).
  let h = 0;
  for (const ch of clientIp) h = (h * 31 + ch.charCodeAt(0)) >>> 0;
  return servers[h % servers.length];
}
```

*Wins when:* you need crude stickiness without cookies (raw TCP, or an L4 LB that can't read HTTP), or you want **cache affinity** — the same user hitting the same server means that server's in-memory cache actually gets hits.

*Fails when — and this is the important part — the pool size changes.* Go from 4 servers to 5 and the modulus changes from `% 4` to `% 5`. Almost **every** key remaps to a different server. Every in-memory cache is instantly cold, every sticky session is on the wrong box. Adding one server causes ~80% of clients to move. That's a stampede, and it's the exact reason the next algorithm exists.

**Consistent hashing.** Instead of `hash % N`, place servers and keys on a conceptual **ring** (0 to 2³²). A key goes to the first server clockwise from it. Now adding or removing one server only remaps the keys in *that server's arc* — roughly `1/N` of keys move, not all of them.

That's the whole intuition; the machinery (virtual nodes, ring balance) is a topic on its own — see [74 — Consistent Hashing](./74-consistent-hashing.md). Just know **why** it exists: it fixes IP hash's catastrophic rebalancing. Every distributed cache and sharded datastore leans on it.

**Random, and power-of-two-choices.** Pure random is O(1), needs zero state, and is *surprisingly decent* — but with pure random, the unluckiest server ends up with noticeably more load than average (queueing theory: max load grows like `log n / log log n` above the mean).

**Power of two choices (P2C)** is the trick: pick **two servers at random**, then send the request to whichever of those two has fewer connections.

```js
class PowerOfTwoChoices {
  constructor(urls) {
    this.active = new Map(urls.map(u => [u, 0]));
    this.urls = urls;
  }
  pick() {
    const a = this.urls[Math.floor(Math.random() * this.urls.length)];
    const b = this.urls[Math.floor(Math.random() * this.urls.length)];
    // Compare only two — O(1) instead of scanning all N servers.
    const chosen = this.active.get(a) <= this.active.get(b) ? a : b;
    this.active.set(chosen, this.active.get(chosen) + 1);
    return chosen;
  }
}
```

Comparing just two random servers instead of all of them collapses the max-load-above-average from `log n / log log n` down to `log log n` — an exponential improvement, for the price of one extra random number. It's cheap, it needs no global view (huge when you have many LBs or client-side balancing), and it avoids the "everyone piles onto the one idle server" herding that naive least-connections suffers when multiple LBs each think the same box is idle. Nginx, Envoy, and Finagle all ship it. Mention P2C in an interview and you will sound like you've read past the blog posts.

### 2. L4 vs L7 — where in the network stack the LB lives

This is the other half of the topic, and it's the comparison you'll be asked to make.

**Layer 4 (transport layer)** works with **TCP/UDP packets**. It sees source IP, source port, destination IP, destination port. That's it. It picks a backend, rewrites the destination, and forwards the packets. It never looks inside the payload — it *cannot*, if the traffic is encrypted, and it doesn't want to anyway.

**Layer 7 (application layer)** actually **terminates the connection**, parses the HTTP request, and reads the method, the URL path, the headers, the cookies. Then it opens (or reuses) a connection to a backend and forwards the request. It's a full proxy, not a packet forwarder.

| | **L4 (Transport)** | **L7 (Application)** |
|---|---|---|
| Sees | IP + port only | Full HTTP: method, path, headers, cookies, body |
| Protocol | TCP / UDP (protocol-agnostic) | HTTP/HTTPS, gRPC, WebSocket |
| Speed | Very fast — millions of packets/sec, µs of added latency | Slower — must parse; ms-scale added latency |
| CPU cost | Low | High (parsing + TLS decrypt) |
| Content-based routing | **No** — cannot route `/api` vs `/images` | **Yes** — route by path, host, header, cookie |
| SSL/TLS termination | No (traffic passes through encrypted) | Yes — decrypts at the LB |
| Sticky sessions | Only by IP hash (crude) | By cookie (precise) |
| Header rewriting (e.g. `X-Forwarded-For`) | No | Yes |
| Retries / circuit breaking | Limited | Yes — can retry a failed request on another backend |
| Sees the real client IP at the backend | Backend sees client IP directly (with DSR/proxy protocol) | Backend sees the LB's IP; real IP must be passed in a header |
| Typical products | AWS NLB, HAProxy in TCP mode, LVS/IPVS | AWS ALB, Nginx, Envoy, Traefik, HAProxy in HTTP mode |

**The mental shortcut:** L4 is a **postal sorting machine** — it reads the address on the envelope and throws it in the right bin, fast, without opening it. L7 is a **receptionist** — she opens the envelope, reads the letter, and decides "this is a billing question, send it to Finance." Slower, but she can make decisions the sorting machine never could.

**When you want L4:** raw TCP services (a database proxy, a game server, MQTT), extreme throughput needs, or when you must not decrypt traffic (end-to-end TLS for compliance).

**When you want L7:** anything HTTP, which is most of what you build. You get path-based routing, SSL termination in one place, header injection, per-route timeouts, request retries, and canary routing by header. The CPU cost is real but almost always worth it.

**In practice, big systems use both:** an L4 LB at the edge handling raw connection volume, forwarding to a fleet of L7 LBs that do the smart routing. L4 for scale, L7 for brains.

### 3. Health checks — how the LB knows a server is dead

A load balancer that keeps sending traffic to a dead server is worse than no load balancer, because it fails *some* requests instead of all of them and makes the failure look random.

**Active health checks.** The LB proactively probes each backend on a fixed interval: `GET /health` every 5 seconds, expect a `200` within 2 seconds. If it doesn't get one, the server accumulates a failure.

**Passive health checks** (also called *outlier detection*). The LB watches **real traffic**. If a server returns five `5xx`s or times out repeatedly, eject it — no probe needed. This catches things a probe misses (the probe endpoint is cheap and passes; the real endpoint is timing out) and it costs zero extra requests.

**Use both.** Passive catches real-world failure fast; active is what tells you the server has *recovered* and can come back.

**Liveness vs readiness — the distinction that gets people paged.**

- **Liveness:** "Is the process alive?" If this fails, the fix is **restart the process**. It should be dumb and cheap — `res.status(200).send('ok')`. It must **not** check the database. If liveness checks the DB and the DB blips, every instance is declared dead and *restarted*, and now you have a DB outage *and* a stampede of cold restarts. This is a real, common, self-inflicted outage.
- **Readiness:** "Can this instance *serve traffic right now*?" DB pool connected? Cache warm? Migrations done? Not currently shutting down? If readiness fails, the fix is **stop sending it traffic** — but do *not* kill it. It may recover in 3 seconds.

**The load balancer should route based on readiness. The orchestrator (e.g. Kubernetes) restarts based on liveness.**

```js
import express from 'express';
const app = express();

let shuttingDown = false;

// LIVENESS: is the event loop responsive at all? Nothing else.
// Deliberately checks no dependencies — a DB blip must never restart the process.
app.get('/healthz', (req, res) => res.status(200).send('ok'));

// READINESS: should the LB send this instance traffic right now?
app.get('/readyz', async (req, res) => {
  if (shuttingDown) return res.status(503).json({ ready: false, reason: 'draining' });
  try {
    // Check only the dependencies this instance cannot serve without.
    await Promise.all([
      db.query('SELECT 1'),         // DB pool actually usable
      redis.ping(),                 // session/cache store reachable
    ]);
    res.status(200).json({ ready: true });
  } catch (err) {
    res.status(503).json({ ready: false, reason: err.message });
  }
});

// Graceful shutdown = connection draining, from the app's side.
process.on('SIGTERM', () => {
  shuttingDown = true;              // readiness starts failing immediately
  // Keep serving in-flight requests. Wait longer than the LB's ejection window
  // (interval x unhealthy_threshold) so the LB has definitely stopped routing here.
  setTimeout(() => server.close(() => process.exit(0)), 15_000);
});
```

**Failure thresholds — why N consecutive failures, not 1.** Networks blip. GC pauses happen. If one missed probe ejected a server, you'd be flapping servers in and out of rotation all day, and a brief hiccup across the fleet could eject *everything*. So real config looks like:

```
interval:            5s     # probe every 5 seconds
timeout:             2s     # a probe that takes >2s counts as a failure
unhealthy_threshold: 3      # 3 consecutive failures => eject
healthy_threshold:   2      # 2 consecutive successes => bring it back
```

Worst-case detection time = `interval × unhealthy_threshold` = **15 seconds** of traffic hitting a dead box. Tighten the numbers and you detect faster but flap more. That tension is the whole tuning exercise, and it's a great thing to say out loud in an interview.

**Slow start and connection draining.**

- **Slow start:** when a server re-enters the pool, ramp its share of traffic up over ~30–60s instead of hitting it at full rate. A freshly booted Node process has cold caches, an empty connection pool, and un-JIT-ed code paths. Under least-connections it looks maximally idle (0 connections) and gets *slammed* — then falls over, gets ejected, restarts, and the cycle repeats. Slow start breaks that loop.
- **Draining (a.k.a. connection draining / deregistration delay):** when you *remove* a server (deploy, scale-down), stop sending it **new** requests but let **in-flight** ones finish — typically 30s. Kill it instantly and you've just 500'd every request that was mid-flight. This is exactly what makes zero-downtime deploys possible.

### 4. Sticky sessions — and why they're a trap

**Sticky session** (session affinity) = the LB pins a given client to a given backend for the duration of their session.

Two ways:
1. **Cookie-based (L7):** the LB sets its own cookie (`AWSALB`, or Nginx's `sticky` cookie) naming the backend. Every subsequent request carries it, and the LB honours it. Precise, survives IP changes.
2. **IP hash (L4):** hash the source IP. Crude — everyone behind one corporate NAT or mobile carrier gateway looks like *one* client and lands on *one* server. Mobile users switching Wi-Fi → cellular change IP and lose their affinity mid-session.

**Why they're used:** because the app stored session state (logged-in user, shopping cart) in **local process memory**. Hit a different server and your cart is gone.

**Why they hurt:**

| Problem | What actually happens |
|---|---|
| Uneven distribution | Affinity ignores load. A server that caught a batch of heavy users stays overloaded — the LB can't move them. |
| Session loss on failure | Server dies → every session pinned to it is gone. Users are logged out and their carts are empty *even though the system is "up"*. |
| Blocks clean scale-down | You can't remove a server without dropping its sessions, so autoscaling can only ever scale *up*. |
| Cripples deploys | Every rolling restart evicts the sessions on the box you're restarting. |
| Cache-warming illusion | Feels like a win, but it's fragile — one restart and the "warm" cache is gone anyway. |

**The right answer: make the app stateless.** Don't store session state in process memory; put it in a shared store (Redis) or in a signed token (JWT) the client carries. Then *any* server can serve *any* request, sticky sessions become unnecessary, and everything above (even distribution, failover, scale-down, rolling deploys) just works.

```js
// BAD: session lives in this process's memory. Requires sticky sessions,
// and every restart or crash logs users out.
const sessions = new Map(); // userId -> cart
app.post('/cart/add', (req, res) => {
  const cart = sessions.get(req.userId) ?? [];
  cart.push(req.body.item);
  sessions.set(req.userId, cart);   // <-- lost forever if THIS box dies
  res.sendStatus(200);
});

// GOOD: session lives in shared Redis. Any server can serve any request.
// The LB is free to route this user anywhere, forever.
app.post('/cart/add', async (req, res) => {
  const key = `cart:${req.userId}`;
  await redis.rpush(key, JSON.stringify(req.body.item));
  await redis.expire(key, 60 * 60 * 24 * 7); // sessions should expire
  res.sendStatus(200);
});
```

See [59 — Caching in Depth](./59-caching-in-depth.md) for how Redis is used as that shared store. **Rule: reach for sticky sessions only when you cannot change the app** (a legacy stateful service, or WebSocket connections that are inherently pinned to one box anyway).

### 5. Who load balances the load balancer?

You added an LB so no single server takes the whole system down… and created a single box that takes the whole system down. If it dies, all your healthy backends are unreachable. This is *the* classic interview follow-up. Three real answers:

**(a) Active-passive pair with a floating (virtual) IP.** Two identical LBs. One is active and holds a **virtual IP (VIP)** — the IP that DNS actually points at. The passive one sits idle, exchanging **heartbeats** with the active one (via VRRP, or keepalived). When heartbeats stop, the passive one **claims the VIP** (it broadcasts an ARP update saying "that IP is me now") and takes over in 1–3 seconds. DNS never changes. Clients never notice. Cost: half your LB capacity sits idle.

**(b) Multiple active LBs behind DNS round robin.** Publish several A records for `api.example.com`, one per LB. Clients spread across them. All LBs are doing work — no idle spare. Downside: **DNS is a bad failover mechanism.** Clients and resolvers cache records for the TTL, so when one LB dies, some fraction of users keep trying the dead IP until their cache expires. Lowering the TTL to 60s helps but never eliminates it. (Modern browsers do retry the next A record on connection failure, which softens this a lot.) See [54 — DNS Deep Dive](./54-dns-deep-dive.md).

**(c) Use a cloud-managed LB.** AWS ALB/NLB, GCP Cloud Load Balancing, Cloudflare. They're already distributed across availability zones with health-checked, automatically-replaced nodes; the "single box" isn't a box at all. This is what you should say you'd do in a real design, and then explain (a) and (b) to prove you know *why* it works.

**Anycast** is the fourth, deepest answer: many machines around the world announce the **same IP address** via BGP, and the internet's own routing sends each client to the topologically nearest one. Cloudflare and Google front-ends run on this.

### 6. Global Server Load Balancing (GSLB)

Everything above balances *within* one datacenter. **GSLB balances across regions.** A user in Mumbai should hit `ap-south-1`, not `us-east-1` — that alone saves ~200ms of round-trip time, per request.

How it decides:
- **Geo/latency-based DNS:** the DNS resolver returns a *different IP* depending on where the query came from. AWS Route 53 latency-based routing does exactly this.
- **Anycast:** one IP, BGP routes you to the nearest POP. No DNS trickery, faster failover.
- **Health-aware failover:** if the entire `eu-west-1` region is unhealthy, GSLB stops handing out its IPs and sends European users to `us-east-1` — degraded latency, but *up*.
- **Weighted regional routing:** shift 10% of traffic to a new region to warm it before a full cutover.

Mentally: **GSLB picks the region, the regional LB picks the server.** Two layers, same idea.

---

## Visual / Diagram description

### Diagram 1 — Basic load balancer topology

```
                             ┌──────────────┐
                             │    Client    │
                             └──────┬───────┘
                                    │  https://api.example.com
                                    ▼
                          ┌─────────────────────┐
                          │        DNS          │  api.example.com -> 203.0.113.10
                          └─────────┬───────────┘
                                    │  (returns the VIP)
                                    ▼
        ┌───────────────────────────────────────────────────┐
        │            LOAD BALANCER  (VIP 203.0.113.10)      │
        │                                                   │
        │   algorithm : least connections                   │
        │   health    : GET /readyz every 5s, 3 fails=out   │
        └───┬───────────────┬───────────────┬───────────────┘
            │               │               │
      ┌─────▼─────┐   ┌─────▼─────┐   ┌─────▼─────┐
      │  App #1   │   │  App #2   │   │  App #3   │
      │  HEALTHY  │   │  HEALTHY  │   │  EJECTED  │  <- failed 3 probes
      │ 12 conns  │   │  4 conns  │   │  0 conns  │     no traffic sent
      └─────┬─────┘   └─────┬─────┘   └───────────┘
            │               │
            └───────┬───────┘
                    ▼
            ┌───────────────┐        ┌──────────────┐
            │   PostgreSQL  │        │    Redis     │  <- shared session state
            └───────────────┘        └──────────────┘     (so no sticky sessions)
```

**What it shows:** one public IP, three interchangeable app servers, one of which has been ejected by health checks and is receiving zero traffic. The next request goes to App #2 (4 connections, fewest). State lives in Postgres and Redis — *not* in the app processes — which is precisely why the LB is free to send any request anywhere.

### Diagram 2 — L7 content-based routing

```
                       ┌──────────────┐
                       │    Client    │  GET /api/orders/42
                       └──────┬───────┘  GET /images/logo.png
                              │          GET /ws/chat
                              ▼
      ┌───────────────────────────────────────────────────────┐
      │              L7 LOAD BALANCER  (Nginx / Envoy / ALB)  │
      │                                                       │
      │   1. terminate TLS  (decrypt: now it can read HTTP)   │
      │   2. parse request line + headers                     │
      │   3. add X-Forwarded-For: <real client IP>            │
      │   4. match the path against routing rules  ───────┐   │
      └───────────────────────────────────────────────────┼───┘
                                                          │
        ┌────────────────────┬────────────────────┬───────┴──────────┐
        │  path = /api/*     │  path = /images/*  │  path = /ws/*    │
        ▼                    ▼                    ▼                  ▼
 ┌─────────────┐      ┌─────────────┐      ┌─────────────┐    ┌────────────┐
 │  API POOL   │      │ STATIC POOL │      │  WS POOL    │    │  DEFAULT   │
 │ (Node x6)   │      │ (Nginx x2)  │      │ (Node x4)   │    │  404 page  │
 │ least-conn  │      │ round robin │      │ sticky —    │    └────────────┘
 │ CPU-heavy   │      │ cheap+fast  │      │ long-lived  │
 └─────────────┘      └─────────────┘      └─────────────┘
```

**What it shows:** the *superpower* an L4 LB does not have. Because the L7 LB decrypted and parsed the HTTP request, it can read the **path** and send `/api/*` to a pool of beefy Node servers using least-connections, `/images/*` to two cheap static servers using round robin, and `/ws/*` to a WebSocket pool where connections are inherently pinned to one box. Three different pools, three different algorithms, **one hostname**. An L4 LB sees only "TCP to port 443" and cannot make any of these distinctions.

Redraw both of these on a whiteboard until you can do it from memory. Diagram 1 is the answer to "how do you scale the web tier"; Diagram 2 is the answer to "how would you route static content separately".

---

## Real world examples

### Netflix — Zuul (L7) in front of everything

Netflix's edge gateway, **Zuul**, is an L7 proxy that every client request passes through. Because it terminates HTTP, it can do far more than pick a server: it does dynamic routing (send this device type to that service version), authentication, request throttling, and — famously — **canary and squeeze testing**, where a small slice of live traffic is routed to a new build to measure it under real load before a full rollout. Netflix's Ribbon library also pushed load balancing to the **client side**: the calling service holds the list of healthy instances and picks one itself, using availability-filtered rules, which removes an entire network hop and an entire single point of failure. This is the direct ancestor of the modern service-mesh sidecar model.

### AWS — the ALB/NLB split is the L7/L4 split

AWS ships two load balancers, and they are a textbook illustration of this topic. The **Network Load Balancer (NLB)** is L4: it operates on TCP/UDP, handles millions of requests per second with ultra-low latency, preserves the client's source IP, and cannot look at your URLs. The **Application Load Balancer (ALB)** is L7: it terminates TLS, parses HTTP, and routes by path, host header, HTTP method, query string, or source IP to different target groups. ALB does cookie-based stickiness; NLB can only do flow hashing. Both run health checks with configurable interval, timeout, and healthy/unhealthy thresholds, and both support **deregistration delay** — AWS's name for connection draining. Choosing between them in a design interview, and justifying it, is a very concrete way to show you understand the layer distinction.

### Kubernetes — two load balancers you may not have noticed

In Kubernetes, a **Service** of type ClusterIP is already an L4 load balancer: `kube-proxy` programs iptables/IPVS rules so traffic to the Service's virtual IP is spread (essentially randomly, or with IPVS's least-connection mode) across the healthy Pod IPs. Pods only enter that rotation when their **readiness probe** passes, and they leave it the instant readiness fails — which is exactly the readiness/liveness distinction from section 3, enforced by the platform. An **Ingress controller** (Nginx, Traefik, Envoy) then adds the L7 layer on top: TLS termination and host/path routing to different Services. So a typical cluster is running an L7 LB in front of an L4 LB in front of your pods — the layered pattern described above, whether or not anyone drew it.

---

## Trade-offs

**Algorithm choice**

| Algorithm | Pros | Cons | Use when |
|---|---|---|---|
| Round robin | Trivial, stateless, O(1), predictable | Ignores request cost and server load | Uniform requests, identical servers, static content |
| Weighted RR | Handles mixed hardware; enables canaries | Weights are static guesses; go stale | Heterogeneous fleet, canary deploys |
| Least connections | Self-correcting; handles variable durations | Slams freshly-added servers; misleading under HTTP/2 multiplexing | **Default for most APIs** |
| Least response time | Detects gradual degradation | Can oscillate/herd; needs latency stats | Latency-sensitive, servers degrade slowly |
| IP hash | Free stickiness; cache affinity | Catastrophic rebalance on pool change; NAT skew | L4 stickiness with a fixed pool |
| Consistent hashing | Only ~1/N keys move on a change | More complex; needs virtual nodes to stay balanced | Sharded caches, stateful routing |
| Random / P2C | O(1), no global state, near-optimal balance | Not perfectly even (nothing is) | Many LBs, client-side LB, huge fleets |

**L4 vs L7**

| | L4 | L7 |
|---|---|---|
| Pros | Blazing fast, cheap CPU, protocol-agnostic, preserves client IP, no decryption needed | Content routing, TLS termination, header rewrite, cookie stickiness, retries, per-route policy |
| Cons | Blind to content; no HTTP-aware anything | More CPU and latency; must hold TLS keys; a real proxy to operate |

**Health checks**

| | Pros | Cons |
|---|---|---|
| Active | Detects failure with no user impact; detects recovery | Costs probe traffic; a shallow probe can pass while the app is actually broken |
| Passive | Free; catches real failures a probe misses | Needs real user requests to fail first; can't detect recovery on its own |

**Sticky sessions**

| Pros | Cons |
|---|---|
| Lets a stateful app work behind an LB with no code change | Uneven load, session loss on failure, blocks scale-down, breaks rolling deploys |
| In-memory cache hit rate goes up | Fragile — one restart and the "warmth" is gone |

**The sweet spot:** an **L7 load balancer** (Nginx, Envoy, or a cloud ALB) doing **least connections**, with **active readiness probes** (5s interval, 3 failures to eject, 2 successes to return) **plus passive outlier detection**, **30s connection draining** on removal and **slow start** on re-entry, **no sticky sessions** because the app is stateless with session state in Redis, and **two LBs** — either an active-passive VIP pair or a cloud-managed LB — so the balancer itself is never the thing that takes you down.

**Rule of thumb:** *Least connections unless you have a reason. L7 unless you need raw speed. Stateless unless you literally cannot change the app.*

---

## Common interview questions on this topic

### Q1: "You have 5 app servers behind a load balancer. Which algorithm do you pick, and why?"

**Hint:** Ask first — *do the requests cost roughly the same?* If yes (static assets, uniform reads), **round robin** is fine and free. If request durations vary (the normal case), pick **least connections**, because a server stuck on slow requests holds open connections and automatically stops being chosen. Mention that you'd add **slow start** so a restarted server with 0 connections doesn't get flooded, and mention **P2C** as the cheap approximation used when there are many load balancers and no single global view of connection counts.

### Q2: "What's the difference between L4 and L7 load balancing, and when would you use each?"

**Hint:** L4 forwards TCP/UDP using only IP+port — extremely fast, protocol-agnostic, but blind: it cannot route by URL or read headers. L7 terminates the connection and parses HTTP, so it can route `/api/*` and `/images/*` to different pools, terminate TLS, rewrite headers, and do cookie stickiness — at a higher CPU and latency cost, and it must hold your TLS keys. Use L4 for raw TCP or extreme throughput; use L7 for anything HTTP, which is most things. Big systems chain them: L4 at the edge → L7 behind it.

### Q3: "Your health check returns 200 but users are getting errors. What's happening?"

**Hint:** The probe is **shallow** — it proves the process is alive (liveness), not that it can serve (readiness). Classic cause: the DB connection pool is exhausted or the Redis connection dropped, so `/health` (which touches nothing) returns 200 while every real request 500s. The fix is a **readiness** endpoint that checks the dependencies the app genuinely cannot serve without, and **passive health checking / outlier detection** so the LB ejects servers based on real error rates rather than trusting the probe. But be careful the readiness check doesn't check *too much* — if it checks a non-critical downstream, one flaky dependency takes your whole fleet out of rotation at once.

### Q4: "What happens if the load balancer itself fails?"

**Hint:** Name it as a single point of failure, then give the three fixes: **(1)** an active-passive pair sharing a **floating/virtual IP** with a heartbeat (VRRP/keepalived) — failover in seconds, DNS unchanged, but half your capacity idles; **(2)** multiple active LBs behind **DNS round robin** — no idle spare, but DNS caching means failover is slow and imperfect; **(3)** a **cloud-managed LB** spread across availability zones, which is what you'd actually do. Bonus points for **anycast**: many machines announce the same IP over BGP and the network routes each client to the nearest one.

### Q5: "Why are sticky sessions considered an anti-pattern? When are they still okay?"

**Hint:** They pin a user to one server, which breaks even distribution (the LB can't move an overloaded user), loses the session when that server dies, blocks clean scale-down, and makes rolling deploys evict sessions. The root cause is state in process memory. Fix the cause: move session state to **Redis** or into a signed **JWT** the client carries, and the app becomes stateless — any server can serve any request. They're still legitimate for **WebSockets** (the connection is physically pinned to one box anyway) and for legacy apps you cannot modify.

---

## Practice exercise

### Build a working load balancer in Node — then break it

Roughly 30–40 minutes. Produce a single file, `lb.js`, that runs.

**Part 1 — the backends.** Start three tiny Express servers on ports 4001, 4002, 4003. Each one:
- responds to `GET /work` after a **random delay of 50–2000ms** (this is the point — request costs must vary), returning which port served it;
- responds to `GET /readyz` with 200;
- exposes `POST /kill` which flips an internal flag so `/readyz` starts returning **503**, and `POST /revive` which flips it back.

**Part 2 — the load balancer.** Write a proxy on port 8080 that forwards `/work` to a backend. Implement **three** pluggable algorithms behind one interface: `RoundRobin`, `LeastConnections`, and `PowerOfTwoChoices` (increment an in-flight counter when you dispatch, decrement when the response completes — that bookkeeping *is* the algorithm).

**Part 3 — health checks.** Every 3 seconds, probe each backend's `/readyz` with a 1s timeout. Eject after **3 consecutive failures**; re-admit after **2 consecutive successes**. Log every state transition: `[HEALTH] :4002 HEALTHY -> EJECTED (3 consecutive failures)`.

**Part 4 — the experiment.** Fire 200 concurrent requests at `/work` and record, per algorithm: (a) how many requests each backend served, and (b) the **p50 and p99** total latency. Then `POST /kill` to port 4002 mid-run and confirm that within ~9 seconds it stops receiving traffic and **no client request fails**.

**What to produce:** a short table comparing request distribution and p99 latency across the three algorithms, plus one sentence explaining *why* least-connections and P2C beat round robin here. (They will — because your delays are random, which is the whole point.) Then answer in writing: *what breaks if you kill the load balancer itself, and what would you add to fix it?*

---

## Quick reference cheat sheet

- **A load balancer does three jobs:** distributes load, ejects dead servers, and enables zero-downtime deploys. Most people forget the last two.
- **Round robin** counts turns, not work. Fine for uniform requests; wrong when request cost varies.
- **Least connections** is the sane default for real APIs — it self-corrects, because a server stuck on slow requests holds connections and stops being picked.
- **Power-of-two-choices** — pick 2 at random, take the less loaded. O(1), no global state, near-optimal balance. Cheap and underrated.
- **IP hash rebalances catastrophically:** change the pool size and `hash % N` remaps almost every key. **Consistent hashing** ([74](./74-consistent-hashing.md)) exists to fix exactly this.
- **L4** = IP + port only. Fast, blind, protocol-agnostic. **L7** = parses HTTP. Can route by path/header/cookie, terminate TLS, rewrite headers — at a CPU cost.
- **Liveness ≠ readiness.** Liveness failing means *restart me*; readiness failing means *stop routing to me*. Never check the DB in a liveness probe — a DB blip would restart your entire fleet.
- **Eject after N consecutive failures, not one.** `interval × unhealthy_threshold` = your worst-case detection time (e.g. 5s × 3 = 15s of bad traffic).
- **Active checks** probe on a schedule and detect recovery. **Passive checks** watch real traffic and detect the failures a shallow probe misses. Run both.
- **Slow start** on re-entry, **connection draining** on removal. Without draining there is no zero-downtime deploy.
- **Sticky sessions are a smell** — they break even distribution, lose sessions on failure, and block scale-down. Store session state in Redis and go stateless.
- **The LB is a SPOF.** Fix with an active-passive VIP pair + heartbeat, multiple LBs behind DNS round robin, a cloud-managed LB, or anycast.
- **GSLB picks the region; the regional LB picks the server.** Same idea, two altitudes.
- **Interview reflex:** the second you draw two app servers, draw the LB — and be ready to say the algorithm, the layer, and the health-check config.

---

## Connected topics

| Direction | Topic |
|---|---|
| **Previous** | [54 — DNS Deep Dive](./54-dns-deep-dive.md) — DNS is what points clients at the load balancer's IP, and DNS round robin is itself a crude form of load balancing. |
| **Next** | [56 — Horizontal vs Vertical Scaling](./56-horizontal-vs-vertical-scaling.md) — the load balancer is the thing that *makes* horizontal scaling possible; this is where you decide to scale out at all. |
| **Related** | [57 — Reverse Proxy](./57-reverse-proxy.md) — an L7 load balancer *is* a reverse proxy with a backend pool; the two roles blur in practice (Nginx does both). |
| **Related** | [74 — Consistent Hashing](./74-consistent-hashing.md) — the fix for IP hash's catastrophic rebalancing when a server joins or leaves the pool. |
| **Related** | [59 — Caching in Depth](./59-caching-in-depth.md) — Redis as the shared session store that lets you delete sticky sessions and make the app stateless. |
