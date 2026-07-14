# 53 — Client-Server Architecture

## Category: HLD Components

---

## What is this?

Client-server architecture is the model where one program **asks** (the client) and another program **answers** (the server). Your browser asks `google.com` for a page; Google's servers answer with HTML. That's it. That's the whole model.

Think of a **restaurant**. You (the client) sit at a table and place an order with the waiter. The kitchen (the server) does the actual work and sends food back. You don't cook. The kitchen doesn't decide what you want. Two roles, one conversation, repeated millions of times a day.

Every single thing in the rest of this curriculum — load balancers, caches, databases, message queues, CDNs — is an elaboration on this one idea: *someone asks, someone answers, and we make the answering fast and reliable.*

---

## Why does it matter?

If you don't have this model crisp in your head, everything downstream is fog.

**What breaks without it:**
- You'll store user state in a variable inside your Node process, deploy a second server, and half your users will randomly get logged out. (This is the #1 real-world bug caused by not understanding stateless servers.)
- You'll put business logic in the browser where anyone can edit it, and someone will set `price = 0` in DevTools.
- You'll try to "just add another server" and find your architecture physically cannot be scaled horizontally.

**Interview angle:** Almost every system design interview implicitly starts here. When you draw a box labeled "App Server" behind a load balancer, the interviewer's very next question is often *"where does the session live?"* — and that question is a direct test of whether you understand stateless servers. If you answer "in memory on the server," the interview goes downhill fast.

**Real-work angle:** The moment your app goes from 1 server to 2, statelessness stops being theory and becomes a production incident. Every autoscaling group, every Kubernetes deployment, every blue-green rollout assumes your servers are interchangeable. They only are if they're stateless.

---

## The core idea — explained simply

### The Restaurant Analogy

Walk into a restaurant. Here's what actually happens:

1. You sit down. You **request** a menu.
2. The waiter **responds** with the menu.
3. You **request** the pasta.
4. The kitchen cooks. The waiter **responds** with the pasta.
5. You **request** the bill. Waiter **responds** with the bill.
6. You leave. The table is wiped clean.

Notice five things:

**a) The client initiates. Always.** The kitchen never randomly sends you food you didn't order. In classic client-server, the server *never* speaks first. (WebSockets and push notifications bend this rule — we'll get to that.)

**b) The client doesn't know how the kitchen works.** You don't know if there are 3 chefs or 30, whether they use a gas or induction stove, whether the pasta was frozen. You only know the **interface**: the menu. This is the API contract.

**c) The kitchen doesn't remember you.** Once you leave, the table is wiped. If you come back tomorrow, you're a stranger. If the kitchen *did* remember every customer forever, it would run out of memory. This is **statelessness**.

**d) Your table number is how they find you.** They don't remember *you*; they remember "order #14 goes to table 7." The identifying info travels *with the order*. This is exactly how a session token works.

**e) One kitchen, many tables.** One server, many clients. That's the whole point — centralize the expensive thing (the kitchen, the database), replicate the cheap thing (the tables, the browsers).

### Mapping the analogy

| Restaurant | Client-Server System |
|---|---|
| You, the diner | The client (browser, mobile app, another service) |
| The order you place | The HTTP request |
| The waiter | The network / transport layer (HTTP over TCP) |
| The menu | The API contract (endpoints, request/response shapes) |
| The kitchen | The server (your Node.js process) |
| The pantry / fridge | The database |
| The plated food | The HTTP response |
| Your table number written on the ticket | The session token / JWT sent with each request |
| Wiping the table when you leave | Statelessness — the server keeps nothing between requests |
| Adding a second kitchen for a busy night | Horizontal scaling |

---

## Key concepts inside this topic

### 1. The request-response lifecycle

When you type `https://api.shop.com/products/42` and hit enter, this is the full journey. Memorize it — interviewers love asking "what happens when you type a URL into a browser?"

```
 1. DNS lookup        api.shop.com  →  93.184.216.34        (see topic 54)
 2. TCP handshake     SYN → SYN-ACK → ACK                   (~1 RTT, ~20-100ms)
 3. TLS handshake     cert exchange, key agreement          (~1 RTT with TLS 1.3)
 4. HTTP request      GET /products/42 HTTP/1.1
                      Host: api.shop.com
                      Authorization: Bearer eyJhbG...
 5. Load balancer     picks a healthy app server            (see topic 55)
 6. Server work       parse → auth → query DB → serialize   (~5-200ms)
 7. HTTP response     200 OK
                      Content-Type: application/json
                      {"id":42,"name":"Keyboard","price":4999}
 8. Client renders    JSON → DOM update
 9. Connection reuse  keep-alive: next request skips steps 2-3
```

The three costs to keep in your head:
- **Network round trip (RTT):** same city ~5ms, cross-country ~40ms, cross-ocean ~150ms.
- **Handshakes:** TCP + TLS costs you ~2 RTTs *before a single byte of your data moves*. This is why keep-alive and HTTP/2 connection reuse matter so much.
- **Server time:** the only part you control with code.

If your server takes 10ms but the user is in Sydney and your server is in Virginia, the user experiences ~310ms. That's why CDNs (topic 60) exist.

### 2. A minimal server, in Node

No framework. Just the `http` module, so you can see the raw shape of client-server.

```javascript
// server.js — the smallest honest client-server example
import http from 'node:http';

// Pretend database. In real life this is Postgres.
const products = new Map([
  [42, { id: 42, name: 'Mechanical Keyboard', price: 4999 }],
  [43, { id: 43, name: 'Wireless Mouse', price: 1999 }],
]);

const server = http.createServer(async (req, res) => {
  // A "request" is just: a method, a path, headers, and (maybe) a body.
  const url = new URL(req.url, `http://${req.headers.host}`);

  if (req.method === 'GET' && url.pathname.startsWith('/products/')) {
    const id = Number(url.pathname.split('/')[2]);
    const product = products.get(id);

    if (!product) {
      res.writeHead(404, { 'Content-Type': 'application/json' });
      return res.end(JSON.stringify({ error: 'Not found' }));
    }

    // A "response" is just: a status code, headers, and a body.
    res.writeHead(200, {
      'Content-Type': 'application/json',
      // Tell the client it may reuse this answer for 60s — free scaling.
      'Cache-Control': 'public, max-age=60',
    });
    return res.end(JSON.stringify(product));
  }

  res.writeHead(404).end();
});

// The server does NOT go looking for clients. It waits.
server.listen(3000, () => console.log('listening on :3000'));
```

Two things to notice:

1. **The server is passive.** `listen()` means "wait." It never initiates.
2. **The handler holds no memory between calls.** Nothing about request N leaks into request N+1. That property is what will let us run 50 copies of this file.

### 3. Thin clients vs thick clients

The question is: **how much work happens on the client's machine vs the server's?**

**Thin client — the server does almost everything.**
The client is basically a display. Server-side rendered (SSR) web pages: PHP, Rails, Django, Next.js with SSR. The server builds the full HTML and ships it. The browser just paints.

```
Browser  ──GET /profile──▶  Server: fetch user, render HTML template
Browser  ◀──<html>...──────  Server returns finished HTML
Browser: paint. That's all it does.
```

**Thick (fat) client — the client does a lot.**
Single-page apps (React, Vue) and mobile apps. The server ships JSON; the client owns routing, rendering, validation, optimistic updates, and often a local cache/DB.

```
Browser  ──GET /app.js────▶  Server: static bundle (from a CDN)
Browser  ──GET /api/user──▶  Server: JSON only
Browser: builds the entire UI itself, manages its own state.
```

**Three modern examples, concretely:**

| | SSR web page | SPA (React) | Mobile app |
|---|---|---|---|
| **What the server sends** | Full HTML per navigation | JS bundle once, then JSON | JSON only (app is pre-installed) |
| **Where routing happens** | Server | Client | Client |
| **First paint** | Fast (HTML arrives ready) | Slow (must download + run JS) | Instant (app already installed) |
| **Later navigations** | Full page load each time | Instant, no full reload | Instant |
| **Works offline?** | No | Partially (service worker) | Yes (local SQLite/Realm) |
| **SEO** | Excellent | Needs SSR/prerender to be good | N/A |
| **Client CPU/battery** | Low | High | Medium-high |
| **Deploy a fix** | Instant (server deploy) | Instant (new bundle) | Days (app store review) |

**The trap:** a thick client is *not* a security boundary. Validation in React is a UX nicety. **Every rule must be re-checked on the server**, because the client is a machine you do not control. Someone will `curl` your API directly.

**The modern answer** is usually hybrid: Next.js/Remix render the first page on the server (fast first paint, good SEO) then hydrate into a thick client (instant later navigation). You get both.

### 4. Stateless vs stateful servers — the key idea

This is the most important section in this doc. Read it twice.

**Stateful server:** the server keeps information about the client *in its own memory* between requests.

```javascript
// ❌ BAD — stateful. This works perfectly on one server and breaks on two.
const sessions = new Map(); // lives in THIS process's RAM

app.post('/login', (req, res) => {
  const sid = crypto.randomUUID();
  sessions.set(sid, { userId: 7, cart: [] }); // state lives here
  res.setHeader('Set-Cookie', `sid=${sid}`);
  res.end('ok');
});

app.get('/cart', (req, res) => {
  const session = sessions.get(req.cookies.sid);
  if (!session) return res.status(401).end(); // ← random 401s in production
  res.json(session.cart);
});
```

Now put two of these behind a load balancer:

```
                    ┌──────────────┐
  login  ──────────▶│ Load Balancer│──────▶ Server A   sessions = { abc: {...} }  ✅
                    └──────────────┘
                    ┌──────────────┐
  GET /cart ───────▶│ Load Balancer│──────▶ Server B   sessions = { }             ❌ 401
                    └──────────────┘
```

The user logged in on Server A. Their next request landed on Server B, which has never heard of them. **This is why stateless servers are the key enabler of horizontal scaling.**

**Stateless server:** the server keeps *nothing* between requests. Every request carries everything needed to serve it. Any server can serve any request. Servers become **interchangeable and disposable** — which means:

- You can add servers freely (autoscaling).
- You can kill servers freely (a crash loses nothing).
- You can deploy by replacing servers one at a time (rolling deploys).
- The load balancer can route anywhere (simple, and health-check-driven).

Stateless doesn't mean "no state exists." It means **the state doesn't live in the app server's memory.** It moves somewhere shared. So: *where does it go?*

### 5. Where session state actually goes — the three options

**Option A — Sticky sessions (session affinity)**

Keep state in server memory, but make the load balancer *always* send a given user back to the same server (usually by hashing the client IP, or by setting a cookie the LB reads).

```
User X ──▶ LB ──(always)──▶ Server A   [X's session in RAM]
User Y ──▶ LB ──(always)──▶ Server B   [Y's session in RAM]
```

- **Pro:** near-zero latency (RAM lookup), no extra infrastructure, easiest retrofit to a legacy app.
- **Con:** Server A dies → all its users are logged out. Load becomes uneven (one "sticky" server can get all the heavy users). Autoscaling is broken: a *new* server gets no existing traffic, and scaling *down* kicks users out. Rolling deploys log everyone out.
- **Verdict:** a legacy crutch. Acceptable short-term; never the answer you lead with in an interview.

**Option B — Shared session store (Redis)**

Servers stay stateless. The session lives in Redis (topic 59). The client holds only an opaque session ID cookie.

```javascript
// ✅ GOOD — stateless app servers, shared store
import { createClient } from 'redis';
const redis = createClient({ url: process.env.REDIS_URL });
await redis.connect();

const SESSION_TTL = 60 * 60 * 24 * 7; // 7 days

async function createSession(userId) {
  const sid = crypto.randomUUID();
  // The state leaves the app process entirely.
  await redis.setEx(`sess:${sid}`, SESSION_TTL, JSON.stringify({ userId }));
  return sid;
}

async function loadSession(sid) {
  if (!sid) return null;
  const raw = await redis.get(`sess:${sid}`);
  return raw ? JSON.parse(raw) : null;
}

// Any server, any request. No affinity needed.
app.get('/cart', async (req, res) => {
  const session = await loadSession(req.cookies.sid);
  if (!session) return res.status(401).end();
  res.json(await getCart(session.userId));
});

// Logout is trivial and INSTANT — that matters, see Option C.
app.post('/logout', async (req, res) => {
  await redis.del(`sess:${req.cookies.sid}`);
  res.clearCookie('sid').end();
});
```

- **Pro:** true statelessness. Any server serves any request. Instant revocation (delete the key). Session can be large (a full cart, permissions, feature flags). Servers can die freely.
- **Con:** one extra network hop per request (~0.5–2ms — cheap, but not free). Redis is now a dependency: if Redis is down, *everyone* is logged out. You must run Redis in HA (replica + sentinel/cluster).
- **Verdict:** the **default correct answer** for most web apps.

**Option C — JWT (stateless tokens, no server-side session at all)**

Don't store the session anywhere. Put the state *in the token itself*, signed so it can't be forged. The client carries it on every request.

```javascript
import jwt from 'jsonwebtoken';

// On login: sign a token containing the claims.
function issueTokens(user) {
  const accessToken = jwt.sign(
    { sub: user.id, role: user.role },   // the "state" travels with the client
    process.env.JWT_SECRET,
    { expiresIn: '15m' }                 // SHORT — this is the revocation window
  );
  // Refresh token IS stored server-side (in Redis/DB) so it CAN be revoked.
  const refreshToken = crypto.randomUUID();
  return { accessToken, refreshToken };
}

// On every request: verify. No DB call, no Redis call. Pure CPU.
function authMiddleware(req, res, next) {
  const token = req.headers.authorization?.slice('Bearer '.length);
  try {
    req.user = jwt.verify(token, process.env.JWT_SECRET);
    next();
  } catch {
    res.status(401).json({ error: 'invalid or expired token' });
  }
}
```

- **Pro:** **zero lookups.** No Redis, no DB, no shared store at all. Perfect for microservices — service B can verify a token minted by service A without calling A. Scales infinitely.
- **Con:** **you cannot revoke it.** If a token is stolen, or you ban a user, or they change their password, that signed token remains valid until it expires. There is no delete button. Mitigations: short expiry (5-15 min) + a refresh token that *is* server-side revocable; or a denylist of revoked JTIs in Redis — but then you're doing a Redis lookup, and you've reinvented Option B with extra steps. Also: tokens are sent on *every* request, so keep them small (a 4KB JWT in a header on every call is real bandwidth).
- **Verdict:** the right choice for **APIs and microservices**. Riskier for classic sessions where instant logout matters (banking, admin panels).

**Side by side:**

| | Sticky sessions | Redis store | JWT |
|---|---|---|---|
| App server stateless? | No | Yes | Yes |
| Extra hop per request | None | ~1ms (Redis) | None |
| Instant revocation | Yes (kill the server 😐) | **Yes** | **No** |
| Survives server death | **No** | Yes | Yes |
| Works with autoscaling | Poorly | Yes | Yes |
| Extra infrastructure | None | Redis (must be HA) | None |
| Payload size on the wire | Tiny cookie | Tiny cookie | Larger token |
| Good for microservices | No | OK | **Best** |

**Rule of thumb:** Redis sessions for user-facing web apps (you will eventually need to force-logout someone). Short-lived JWTs + revocable refresh tokens for APIs and service-to-service. Sticky sessions only when you inherited a codebase you can't change yet.

### 6. Tiers: 1-tier, 2-tier, 3-tier, n-tier

A "tier" is a **physically separable** layer — something that could run on its own machine. (Contrast with "layer," which is a logical code separation. Three layers can run in one tier.)

**1-tier (monolithic / standalone):** everything on one machine. UI, logic, and data in one process.
*Example:* Microsoft Excel, SQLite-backed desktop apps, a local dev server.
*Breaks when:* more than one user needs the data.

**2-tier (client-server, classic):** a fat client talks directly to a database.
*Example:* a 1990s desktop app with an embedded Oracle connection string; a Java Swing app hitting MySQL.
*Breaks when:* every client needs DB credentials (security nightmare), business logic is duplicated in every client, and 5,000 clients each hold a DB connection (databases die around a few hundred connections — topic 65).

**3-tier — the canonical web architecture. This is what you draw in interviews.**

| Tier | Name | Responsibility | Typical tech |
|---|---|---|---|
| 1 | **Presentation** | Render UI, capture input | Browser, React, mobile app |
| 2 | **Application / Logic** | Business rules, auth, validation, orchestration | Node.js/Express, Django, Spring |
| 3 | **Data** | Persist and query | PostgreSQL, Redis, S3 |

The critical rule: **the presentation tier never talks to the data tier directly.** All access flows through the logic tier, which is where authorization, validation, and rate limiting live. That single rule is why 3-tier won.

Why it beats 2-tier:
- **Security:** DB credentials exist only in tier 2, never on a user's device.
- **Scaling:** tier 2 is stateless → scale it horizontally to N boxes. Tier 3 scales differently (replicas, sharding).
- **Connection pooling:** 10,000 clients → 20 app servers → 200 DB connections. Tier 2 acts as a funnel.
- **Change safety:** swap Postgres for DynamoDB and no client needs to change.

**n-tier:** just 3-tier with more slices, once the middle gets fat.
```
Client → CDN → Load Balancer → API Gateway → [Auth svc | Order svc | Search svc]
       → Cache (Redis) → Message Queue (Kafka) → Workers → Database → Blob store
```
Each new tier buys you one thing (a CDN buys latency; a queue buys decoupling) and costs you one thing (a hop, a failure mode, an on-call page). Don't add a tier you can't justify in one sentence.

### 7. Peer-to-peer — the contrast

In P2P, there is **no permanent server**. Every node is both client and server. Nodes discover each other and exchange data directly.

```
Client-Server:                   Peer-to-Peer:

   C   C   C                        P ───── P
    \  |  /                         │ ╲   ╱ │
     \ | /                          │   ╳   │
      ▼▼▼                           │ ╱   ╲ │
    [SERVER]                        P ───── P
    (bottleneck,                    (no center; capacity
     single owner)                   GROWS with users)
```

| | Client-Server | Peer-to-Peer |
|---|---|---|
| Capacity as users grow | **Degrades** (server saturates) | **Improves** (each peer adds upload capacity) |
| Consistency / source of truth | Easy — one authority | Hard — no authority |
| Cost to operator | High (you pay for all bandwidth) | Near zero (peers pay) |
| Censorship / takedown | Easy | Very hard |
| Discovery | Trivial (a DNS name) | Hard (needs a tracker, DHT, or bootstrap nodes) |
| Security | Server enforces rules | Everyone is untrusted |
| Examples | Every web app you use | BitTorrent, IPFS, blockchains, WebRTC calls |

**The pragmatic reality: almost everything is hybrid.** BitTorrent needs a *tracker* (a server) to find peers. WebRTC video calls send media peer-to-peer but need a **signaling server** to introduce the peers and a **TURN relay server** when NAT blocks the direct path — so ~15% of calls end up relayed through a server anyway. Skype started P2P and moved to servers. The lesson: P2P is a bandwidth optimization, not usually a whole architecture.

---

## Visual / Diagram description

### Diagram 1: The 3-tier architecture (draw this one from memory)

```
   PRESENTATION TIER              APPLICATION TIER                DATA TIER
 ┌────────────────────┐      ┌─────────────────────────┐    ┌──────────────────┐
 │  ┌──────────────┐  │      │                         │    │                  │
 │  │   Browser    │  │      │   ┌─────────────────┐   │    │  ┌────────────┐  │
 │  │  (React SPA) │──┼──┐   │ ┌▶│  App Server 1   │───┼──┬─┼─▶│ PostgreSQL │  │
 │  └──────────────┘  │  │   │ │ │  (Node.js)      │   │  │ │  │  primary   │  │
 │                    │  │   │ │ │  STATELESS      │   │  │ │  └─────┬──────┘  │
 │  ┌──────────────┐  │  │   │ │ └─────────────────┘   │  │ │        │ replicate│
 │  │  Mobile App  │──┼──┤   │ │                       │  │ │  ┌─────▼──────┐  │
 │  │  (iOS/Android│  │  │   │ │ ┌─────────────────┐   │  │ │  │ PostgreSQL │  │
 │  └──────────────┘  │  │   │ ├▶│  App Server 2   │───┼──┤ │  │  replica   │  │
 │                    │  │   │ │ │  (Node.js)      │   │  │ │  └────────────┘  │
 │  ┌──────────────┐  │  │   │ │ │  STATELESS      │   │  │ │                  │
 │  │ Partner API  │──┼──┤   │ │ └─────────────────┘   │  │ │  ┌────────────┐  │
 │  │  (curl/SDK)  │  │  │   │ │                       │  ├─┼─▶│   Redis    │  │
 │  └──────────────┘  │  │   │ │ ┌─────────────────┐   │  │ │  │ (sessions, │  │
 │                    │  │   │ ├▶│  App Server 3   │───┼──┘ │  │  cache)    │  │
 └────────────────────┘  │   │ │ │  (Node.js)      │   │    │  └────────────┘  │
                         │   │ │ │  STATELESS      │   │    │                  │
                         │   │ │ └─────────────────┘   │    │  ┌────────────┐  │
                         │   │ │                       │    │  │  S3 / Blob │  │
                         │   │ │  ┌─────────────────┐  │    │  │  (images)  │  │
                         └───┼─┴──│  LOAD BALANCER  │  │    │  └────────────┘  │
                             │    │  (topic 55)     │  │    │                  │
                             │    └─────────────────┘  │    └──────────────────┘
                             └─────────────────────────┘

  Rule: presentation NEVER reaches past the app tier to the data tier.
  Rule: app servers hold NO user state → any of the 3 can serve any request.
  Rule: kill App Server 2 right now → zero users notice.
```

**What this shows:** three clients of different shapes (browser, mobile, partner integration) all speak the *same* HTTP/JSON contract to a *pool* of identical, disposable app servers. Because those servers are stateless, the load balancer is free to send request #1 to server 1 and request #2 to server 3. The state that used to live in server RAM has been pushed rightward into Redis and Postgres, which are *designed* to be shared.

### Diagram 2: Where the state went

```
  STATEFUL (breaks at 2 servers)          STATELESS (scales to 200 servers)

   ┌─────────────┐                         ┌─────────────┐   ┌─────────────┐
   │  Server A   │                         │  Server A   │   │  Server B   │
   │             │                         │             │   │             │
   │ sessions={  │  ← user's cart          │  (empty)    │   │  (empty)    │
   │   abc: {...}│    lives HERE           │             │   │             │
   │ }           │                         └──────┬──────┘   └──────┬──────┘
   └─────────────┘                                │                 │
   ┌─────────────┐                                └────────┬────────┘
   │  Server B   │                                         ▼
   │ sessions={} │  ← ...and NOT here            ┌──────────────────┐
   └─────────────┘     → random 401s             │      REDIS       │
                                                 │  sess:abc → {...}│
                                                 └──────────────────┘
                                                   ← one shared truth
```

---

## Real world examples

### 1. Netflix — thick clients, stateless services

Netflix's apps (TV, phone, browser) are deliberately **thick clients**: they hold the UI, the playback buffer, adaptive bitrate logic, and a local cache of your rows. The backend is a large set of **stateless microservices** behind an edge gateway (their open-sourced Zuul), fronted by AWS load balancers. Because the services hold no session state, Netflix can (and famously does, via Chaos Monkey) randomly terminate production instances during business hours — if killing a server logged users out, that practice would be impossible. It's the clearest proof that statelessness isn't academic: it's what makes "servers are cattle, not pets" true.

### 2. Amazon.com — the 3-tier rule taken seriously

Amazon's well-documented internal mandate (the "Bezos API mandate") forced every team to expose data *only* through service interfaces — no team may reach into another team's datastore directly. That is the 3-tier rule ("presentation never touches the data tier") generalized to hundreds of teams. The cost is real: more network hops, more failure modes, more latency budgets. The payoff is that any team can rewrite its storage without breaking anyone, which is what eventually made AWS possible as an external product.

### 3. WhatsApp — the server-first exception

WhatsApp's messages are end-to-end encrypted, so you might expect P2P. It isn't — messages route through WhatsApp's servers, which store them only until delivery. Why keep a server at the center at all? **Because clients are unreliable.** Your phone is offline, in a tunnel, out of battery. A server is a rendezvous point that is *always awake*. This is the honest argument for client-server over P2P: not that servers are better computers, but that they are *reliably reachable*, and someone must hold the message while the recipient's phone is dead.

---

## Trade-offs

**Thin client vs thick client**

| | Thin (SSR) | Thick (SPA / mobile) |
|---|---|---|
| First-load speed | **Fast** | Slow (download + parse JS) |
| Subsequent navigation | Slow (full reload) | **Instant** |
| Server CPU cost | **High** (renders every page) | Low (serves JSON) |
| Client CPU / battery | Low | High |
| Offline capability | None | **Good** |
| SEO | **Native** | Needs extra work |
| Ship a bugfix | **Instantly** | Instantly (web) / days (app store) |
| Security posture | Logic hidden on server | **Logic visible — must re-validate server-side** |

**Stateless vs stateful servers**

| | Stateless | Stateful |
|---|---|---|
| Horizontal scaling | **Trivial** | Painful (sticky sessions or bust) |
| Server crash impact | Zero (retry elsewhere) | Users lose their session |
| Rolling deploy / autoscale | **Free** | Kicks users off |
| Latency per request | +1 hop to fetch state | **0 (it's in RAM)** |
| Infra needed | Redis or JWT | None |
| When it's genuinely right | ~Always for HTTP APIs | Long-lived connections (WebSocket/game servers), where the connection *is* the state |

**Note the honest exception:** a WebSocket chat server or a multiplayer game server *is* stateful — the open socket lives in one process's memory. You can't make that stateless. What you do instead is make the state **recoverable** (persist to Redis) and **re-routable** (a connection registry that says "user 7 is on box 4"), so a crash costs a reconnect, not a data loss.

**The sweet spot:** Make your HTTP tier ruthlessly stateless and push all session state into Redis (or a short-lived JWT). Use a hybrid client (SSR first paint → hydrate to SPA). Keep 3 tiers until a specific, measured pain forces you to add a fourth.

---

## Common interview questions on this topic

### Q1: "What happens when you type a URL in the browser and press Enter?"
**Hint:** Walk the lifecycle: DNS resolution (browser cache → OS → resolver → root → TLD → authoritative) → TCP handshake → TLS handshake → HTTP GET → load balancer picks a server → app server runs auth + DB query → response → browser parses HTML, fetches sub-resources, renders. Mention keep-alive and that CDNs may short-circuit steps 1–6 entirely. Interviewers use this to see how many layers you actually know.

### Q2: "You have one server holding sessions in memory. You add a second server and users start getting randomly logged out. Why, and how do you fix it?"
**Hint:** The LB round-robins requests, so a user's second request hits a server whose `sessions` Map has never seen their ID. Three fixes, and *name the trade-off of each*: (1) sticky sessions — quick, but breaks on server death and autoscaling; (2) Redis session store — correct default, costs one hop and a new HA dependency; (3) JWT — no lookup at all, but you can't revoke it, so pair a 15-minute access token with a revocable refresh token. Lead with Redis.

### Q3: "Why can't you just store state on the server? Isn't RAM faster than Redis?"
**Hint:** Yes — RAM is ~100ns, Redis is ~1ms, so in-memory is literally 10,000× faster *per lookup*. That's not the point. The point is **fault tolerance and elasticity**: an in-memory session dies with the process, and processes die constantly (deploys, autoscale-down, OOM, spot-instance reclaim). You're trading 1ms of latency for the ability to lose any server at any time without a user noticing. That trade is almost always correct.

### Q4: "When would you choose P2P over client-server?"
**Hint:** When bandwidth is the dominant cost and there's no need for a single source of truth: large-file distribution (BitTorrent), real-time media between two parties (WebRTC), censorship-resistant storage (IPFS). Note that capacity *grows* with users instead of degrading. Then be honest about why almost nobody does it: no authority means hard consistency, hard moderation, hard discovery, and NAT traversal fails often enough that you end up running relay servers anyway.

### Q5: "Is a JWT more secure than a session cookie?"
**Hint:** No — this is the trap. A JWT is *stateless*, not *more secure*. A session ID in an `HttpOnly; Secure; SameSite` cookie is revocable instantly and reveals nothing if leaked in a log. A JWT in localStorage is XSS-readable and cannot be revoked. JWT's real win is **avoiding a shared lookup across services**, which matters in microservices and matters very little in a monolith.

---

## Practice exercise

### Make a stateful server stateless

**Time: ~30-40 minutes. Produce working code.**

**Part A — build the broken version (10 min).**
Write a Node `http` (or Express) server with an in-memory `Map` for sessions and these three routes:
- `POST /login` → creates a session, sets a `sid` cookie
- `POST /cart` → adds an item to that session's cart
- `GET /cart` → returns the cart

Run **two copies** on ports 3001 and 3002. Log in against 3001, then `curl` `GET /cart` against 3002 with the same cookie. Watch it 401. **Feel the bug.** This is the exact production incident you'd otherwise learn about at 2am.

**Part B — fix it with Redis (15 min).**
Replace the `Map` with Redis (`docker run -p 6379:6379 redis`). Keys: `sess:<sid>`, value: JSON, TTL 7 days. Now both ports work with the same cookie. Then kill port 3001 mid-session and confirm 3002 still serves the cart.

**Part C — fix it with a JWT instead (10 min).**
Drop Redis. Sign a 15-minute JWT at login containing `{ sub: userId }`. Store the cart in Postgres/a file keyed by `userId` (state has to live *somewhere* — the point is it's not in the app process). Verify the token on each request.

**Part D — write four sentences answering:**
1. Which version could you autoscale from 2 → 50 servers without touching code?
2. In the JWT version, how would you force-logout a user whose account you just banned? What does that cost you?
3. Which version has the lowest p99 latency, and by roughly how much?
4. Which would you actually ship, and why?

---

## Quick reference cheat sheet

- **Client-server** = the client always initiates; the server waits and responds. It is the base model for everything else in this curriculum.
- **Request-response lifecycle:** DNS → TCP handshake → TLS handshake → HTTP request → LB → server work → response → render. Two RTTs are burned *before* your data moves — reuse connections.
- **Thin client** = server renders (SSR); fast first paint, great SEO, high server CPU. **Thick client** = client renders (SPA/mobile); instant navigation, offline-capable, but logic is public.
- **Never trust the client.** Client-side validation is UX. Server-side validation is security. Re-check everything.
- **Stateless server** = keeps nothing about the client between requests. This is *the* enabler of horizontal scaling — it makes servers interchangeable and disposable.
- **Sticky sessions:** fastest, zero infra, but breaks on server death, rolling deploys, and autoscaling. A legacy crutch.
- **Redis session store:** the default correct answer. Costs ~1ms/request and an HA Redis. Gives you instant revocation.
- **JWT:** zero lookups, perfect for microservices, but **cannot be revoked**. Use short expiry (15 min) + a revocable refresh token.
- **3-tier** = presentation / application / data. The golden rule: **presentation never touches the data tier.**
- **2-tier dies** because clients hold DB credentials and each one holds a DB connection.
- **A "tier" is physical** (separate machine); a "layer" is logical (separate code module). Three layers can live in one tier.
- **P2P** capacity *grows* with users; client-server capacity *degrades*. But P2P has no source of truth and no reliable rendezvous — which is why WhatsApp still uses servers.
- **The honest exception to statelessness:** WebSocket/game servers are inherently stateful. Make that state *recoverable* and *re-routable* instead.

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [52 — Concurrency Fundamentals](./52-concurrency-fundamentals.md) — how a single Node server handles many clients at once with its event loop |
| **Next** | [54 — DNS Deep Dive](./54-dns-deep-dive.md) — step 1 of the request lifecycle: how the client finds the server's IP in the first place |
| **Related** | [55 — Load Balancing](./55-load-balancing.md) — the component that makes a pool of stateless servers look like one server |
| **Related** | [56 — Horizontal vs Vertical Scaling](./56-horizontal-vs-vertical-scaling.md) — statelessness is the precondition; this is what it buys you |
