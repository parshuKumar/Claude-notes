# 58 — API Gateway
## Category: HLD Components

---

## What is this?

An **API Gateway** is the single front door to a system made of many backend services. Every client request enters through it. It checks who you are, decides whether you're allowed in, rate-limits you if you're greedy, figures out which internal service should handle you, sends the request there, and shapes the response on the way back.

Think of the **concierge desk at a large hotel**. You don't wander the building hunting for housekeeping, the kitchen, or the spa. You go to one desk. The concierge checks your key card once (auth), notes you've already ordered four room-service meals today (rate limiting), knows the kitchen speaks Italian while you speak English (protocol translation), and — if you ask "what's happening tonight?" — calls the restaurant, the spa, and the events team on your behalf and hands you **one** answer (aggregation). You never learn the hotel's internal phone extensions.

---

## Why does it matter?

Once you split a monolith into microservices ([71 — Microservices vs Monolith](./71-microservices-vs-monolith.md)), you immediately create a nasty question: **how does the client talk to 20 services?**

Without a gateway, the answer is ugly:
- The mobile app has to know 20 hostnames, and hardcodes them. Rename a service and you break every app version in the wild — including the ones users refuse to update.
- Every one of the 20 services has to implement JWT validation, rate limiting, CORS, request logging, and TLS. **Twenty times.** Twenty chances to get it wrong. One team forgets to check the token expiry and you're breached.
- The mobile app's home screen needs data from 8 services, so it makes **8 round-trips** over a flaky 4G connection. On a 200ms mobile RTT that's 1.6 seconds *just in network time*, before any server does any work.
- You can't do anything centrally: no global rate limit, no unified metrics, no canary routing, no single place to shut off abusive traffic.

**Interview angle:** "How does the client talk to your microservices?" is asked in nearly every microservices design round. "API Gateway" is the expected first word. The *good* answer continues: what it does, how it differs from a reverse proxy and a load balancer, and — the senior signal — **why it's dangerous** (it becomes a monolith and a single point of failure).

**Real work angle:** You will configure one. It's the natural home for cross-cutting concerns, and it's also where badly-run teams quietly put business logic until the gateway becomes the most terrifying deploy in the company.

---

## The core idea — explained simply

### The Hotel Concierge Analogy

You check into a 40-floor hotel. Inside are dozens of independent departments: the restaurant, the spa, housekeeping, the gym, valet parking, the business centre. Each is genuinely independent — different staff, different systems, some are even outsourced contractors.

**Hotel WITHOUT a concierge (no gateway):**

You want dinner and a spa slot and your car brought round. So you:
- Take the lift to floor 3, find the restaurant, show your key card, book a table.
- Take the lift to floor 12, find the spa, show your key card **again**, book a slot.
- Walk to the basement, find valet, show your key card **again**.

Three trips. You had to know where everything was. **Every department had to independently learn how to verify key cards** — and the outsourced spa contractor implemented it badly, accepting cards that expired last week. If the hotel renumbers floors, all guests are lost.

**Hotel WITH a concierge (gateway):**

You go to one desk in the lobby.
- The concierge checks your key card **once** (authentication at the edge).
- She sees you've already made 12 requests in an hour and politely asks you to slow down (rate limiting).
- She knows the restaurant is on floor 3 and the spa on 12 (routing).
- She notices the spa's booking system only speaks fax while you speak English, and translates (protocol translation).
- You ask "what's my evening?" — she calls the restaurant, the spa, **and** valet, waits for all three, and hands you **one** printed itinerary (aggregation).
- Internally, she stamps every request "guest verified, room 402, loyalty tier gold" so the departments can just **trust it** and get on with their work.

Now: **the departments never check key cards.** They trust the concierge's stamp, because the only door into the building is the lobby. That's the deepest idea in this whole topic.

**But watch the danger.** Over the years, management keeps adding to the concierge's job. "Also calculate the loyalty discount." "Also apply the seasonal pricing rules." "Also decide which guests get free breakfast." Now the concierge desk holds business logic that belongs to *departments*, every rule change requires retraining her, she's the busiest person in the hotel, **and if she goes home sick, no guest can do anything at all.** The concierge has become a monolith and a single point of failure.

### Mapping it back

| Hotel | System |
|---|---|
| The lobby concierge desk | API Gateway |
| Departments on different floors | Microservices |
| Checking your key card once | JWT validation at the edge |
| The "verified, room 402, gold" stamp | Trusted internal header (`X-User-Id`) or an internal signed token |
| Departments trusting the stamp | Downstream services skip auth — the network is closed |
| "Slow down, you've asked 12 times" | Rate limiting |
| Knowing which floor is which | Routing |
| Translating fax ↔ English | Protocol translation (REST → gRPC) |
| One printed itinerary from 3 calls | Response aggregation / BFF |
| Guests never see floor numbers | Internal topology hidden |
| Concierge doing pricing rules | **Business logic leaking into the gateway — the anti-pattern** |
| Concierge sick → hotel unusable | Gateway as a single point of failure |

---

## Key concepts inside this topic

### 1. Routing — the baseline job

The simplest thing a gateway does: map a public path to an internal service.

```
PUBLIC (what the client sees)          INTERNAL (what actually exists)
─────────────────────────────          ────────────────────────────────
api.example.com/users/*         →      user-service.internal:3001
api.example.com/orders/*        →      order-service.internal:3002
api.example.com/products/*      →      product-service.internal:3003
api.example.com/payments/*      →      payment-service.internal:3004  (Go, gRPC)
api.example.com/search/*        →      search-service.internal:3005   (Python)
```

The client sees **one** hostname. It cannot tell there are five services, what languages they're in, or that you split `orders` into two services last Tuesday. That decoupling is the point: **you can re-shape your backend without shipping a new mobile app.** Given that a meaningful chunk of your users will never update the app, this is not a nice-to-have.

Routing also plugs into [72 — Service Discovery](./72-service-discovery.md): in a world where instances come and go, the gateway asks the registry "who's healthy for `order-service` right now?" rather than reading a static IP from a file.

### 2. Authentication & authorization — validate the JWT ONCE, at the edge

This is the single most valuable thing a gateway does, and the reasoning is worth getting exactly right.

A **JWT** (JSON Web Token) is a signed blob the client sends on every request. Validating it means: check the signature, check it hasn't expired, check the issuer. That's cryptographic work, and it requires the key.

**Without a gateway,** all 20 services must validate the token. That means: 20 copies of the validation code (in 5 different languages), 20 services that need the public key distributed to them, and — the real killer — **20 opportunities to get it wrong.** One team forgets the expiry check. Another accepts the `none` algorithm. You are now breached, and the breach is in a service nobody has looked at in a year.

**With a gateway,** validation happens **once**, at the edge. The gateway verifies the token, extracts the claims, and forwards the request to the internal service with the identity attached:

```
Client                 Gateway                      Order Service
  │                       │                              │
  │ GET /orders           │                              │
  │ Authorization:        │                              │
  │   Bearer eyJhbGc...   │                              │
  ├──────────────────────▶│                              │
  │                       │ 1. verify signature ✓        │
  │                       │ 2. check exp ✓               │
  │                       │ 3. extract claims            │
  │                       │ 4. STRIP the raw token       │
  │                       │ 5. attach identity           │
  │                       │                              │
  │                       │ GET /orders                  │
  │                       │ X-User-Id: 4021              │
  │                       │ X-User-Roles: customer       │
  │                       ├─────────────────────────────▶│
  │                       │                              │ Just trusts it.
  │                       │                              │ ZERO auth code here.
  │                       │◀─────────────────────────────┤
  │◀──────────────────────┤                              │
```

The order service contains **no authentication logic at all.** It reads `X-User-Id` and gets on with its job.

**"But isn't blindly trusting a header insanely dangerous?"** — Yes, and you should raise this yourself in an interview, because it's the follow-up. It is only safe if **the gateway is the only way in.** Two things must be true:

1. **Network isolation.** Internal services live on a private network and are not routable from the internet. There is no other door.
2. **The gateway strips the header on the way in.** If a client sends `X-User-Id: 1` (the admin), the gateway must **delete it** before forwarding — otherwise anyone can impersonate anyone. This is a real, exploited vulnerability class. *Always strip inbound copies of your trusted headers.*

If you don't fully trust the network (zero-trust architectures, service meshes), the gateway instead mints a **short-lived internal signed token** and services verify *that* — cheaper and simpler than full user-JWT validation, but not blind trust. Naming this nuance is a strong senior signal.

**Also note the split:** the gateway does **coarse** authorization ("is this a valid, non-expired user? does this token have the `admin` scope for `/admin/*`?"). It must **not** do fine-grained authorization ("can user 4021 read order 9987?") — that requires knowing that order 9987 belongs to user 4021, which is **business logic**, and it lives in the order service. Coarse at the edge, fine in the service.

### 3. The N+1 client-calls problem, aggregation, and BFF

Here is the concrete pain. A mobile app's home screen needs:

```
1. GET /users/me                → profile
2. GET /users/me/notifications  → unread count
3. GET /orders?recent=5         → recent orders
4. GET /cart                    → cart items
5. GET /products/recommended    → recommendations
6. GET /promotions/active       → banners
7. GET /loyalty/points          → points balance
8. GET /support/open-tickets    → ticket badge
```

**Eight round-trips.** On mobile, network round-trip time dominates everything. Do the arithmetic:

```
Mobile RTT (4G, realistic):     ~150–200 ms
8 sequential calls × 200 ms  =  1,600 ms  ← just network. Zero server time.
Add ~50ms server time each   =    400 ms
──────────────────────────────────────────
Home screen:                  ~2,000 ms  → the app feels broken
```

Even parallelised, the client pays 8 TLS/TCP setups, 8 sets of headers, and burns battery and radio time. And it gets *worse* on a bad connection, where a single dropped packet on any of the 8 stalls the whole screen.

**With gateway aggregation** — the client makes ONE call:

```
1. GET /mobile/home   →  gateway fans out to all 8 services IN PARALLEL
                         over the fast internal network (~5–10 ms each)
                         and returns ONE combined JSON payload.

Mobile RTT:                       200 ms  (one round trip)
Gateway internal fan-out:         ~60 ms  (parallel, slowest wins + overhead)
──────────────────────────────────────────
Home screen:                     ~260 ms  → 7.7x faster
```

The client did one round-trip. The eight round-trips still happened, but across a **datacenter LAN** where they cost 5ms each, not 200ms.

**Backend-for-Frontend (BFF).** Now notice: the mobile home screen and the web home screen and the smart-TV home screen all want *different* data. If you cram all of them into one gateway endpoint, it bloats.

The **BFF pattern** says: give each client type **its own gateway**, tailored to it.

```
        ┌──────────┐  ┌──────────┐  ┌──────────┐
        │ iOS/     │  │  Web     │  │ Partner  │
        │ Android  │  │  SPA     │  │  API     │
        └────┬─────┘  └────┬─────┘  └────┬─────┘
             ▼             ▼             ▼
     ┌──────────────┐ ┌───────────┐ ┌──────────────┐
     │ Mobile BFF   │ │  Web BFF  │ │ Public API   │
     │ tiny payload │ │ rich data │ │ strict       │
     │ few round-   │ │ fine to   │ │ versioning,  │
     │ trips        │ │ make 3-4  │ │ hard rate    │
     │              │ │ calls     │ │ limits       │
     └──────┬───────┘ └─────┬─────┘ └──────┬───────┘
            └───────────────┼──────────────┘
                            ▼
                  the same microservices
```

The mobile BFF is owned by the mobile team and returns tiny, pre-shaped payloads. The web BFF returns richer data. Neither compromises for the other. **Netflix built exactly this**, because their device-specific needs (TV, phone, browser, game console) diverged so wildly that one generic API served nobody well.

### 4. The other gateway responsibilities

**Rate limiting.** One place to enforce "100 requests/minute per API key." Doing it at the edge means abusive traffic never touches your services at all. Counters live in Redis so all gateway instances share them (recall from [56](./56-horizontal-vs-vertical-scaling.md): an in-memory counter breaks the moment you run a second gateway instance). See [70 — Rate Limiting](./70-rate-limiting.md).

**Request/response transformation.** The internal service returns `{ user_id, created_ts }`; your public API contract promises `{ userId, createdAt }`. The gateway renames. More importantly it can **strip fields** — the user service's response includes `passwordHash` and `internalRiskScore`, and the gateway removes them before they reach a client. Never rely on this as your only defence, but it's a good backstop.

**Protocol translation.** Outside your network, clients speak **REST/JSON over HTTP** because that's what browsers and mobile SDKs do well. Inside, services often speak **gRPC** — a binary protocol that's substantially faster and gives you strongly-typed contracts. The gateway is the translation point: REST in, gRPC out. Same for WebSocket → internal message queue, or GraphQL → many REST calls. See [69 — API Design](./69-api-design-rest-graphql-grpc.md).

**Caching.** Cache `/products/123` at the gateway for 60 seconds and the product service never sees the repeat traffic. Same idea as the reverse-proxy cache in [57](./57-reverse-proxy.md), but the gateway can cache **per-user** because it knows who the user is — something a dumb proxy cannot do.

**API versioning.** `/v1/*` and `/v2/*` route to different service versions, letting you run both while old mobile clients drain away. The gateway is where you deprecate.

**Metrics/logging/tracing.** Every request passes through, so it's the natural place to emit request counts, latencies, and error rates per route — and to stamp a **trace ID** onto every request that then flows through all downstream services, letting you reconstruct a single user's journey across 8 services in your tracing tool. See [80 — Monitoring and Observability](./80-monitoring-and-observability.md).

**Canary routing.** Send 5% of traffic to `order-service-v2` and 95% to `v1`. Watch v2's error rate. If it's clean, shift to 50%, then 100%. If it spikes, shift back to 0% instantly — **no deploy needed**, just a config change at the gateway. This makes risky releases genuinely safe, and it's a great thing to volunteer in an interview.

### 5. Gateway vs Reverse Proxy vs Load Balancer

The three get muddled constantly. The clean mental model:

```
┌──────────────────────────────────────────────────────────────────┐
│  API GATEWAY          ← API-aware. Knows about USERS, TOKENS,    │
│  ┌───────────────────────┐  API KEYS, QUOTAS, and BUSINESS-      │
│  │  REVERSE PROXY        │  ADJACENT concerns. Aggregates.       │
│  │  ┌─────────────────┐  │  ← Protocol-level. Knows about HTTP:  │
│  │  │  LOAD BALANCER  │  │    TLS, headers, paths, compression,  │
│  │  │                 │  │    caching, buffering.                │
│  │  │  ← One job:     │  │                                       │
│  │  │    spread the   │  │                                       │
│  │  │    traffic.     │  │                                       │
│  │  └─────────────────┘  │                                       │
│  └───────────────────────┘                                       │
└──────────────────────────────────────────────────────────────────┘
        Each outer ring does everything the inner ring does, plus more.
```

| | Load Balancer | Reverse Proxy | API Gateway |
|---|---|---|---|
| **Layer of concern** | Connections | HTTP protocol | **APIs and identity** |
| **Core question** | "Which backend is least busy?" | "How do I front these servers efficiently?" | "Who are you, are you allowed, and where does this API call go?" |
| **Spread traffic** | ✅ its only job | ✅ | ✅ |
| **TLS termination** | L7 ones, yes | ✅ | ✅ |
| **Caching, compression, static files** | ❌ | ✅ | ✅ |
| **Buffer slow clients** | sometimes | ✅ | ✅ |
| **Knows what a JWT is** | ❌ | ❌ | ✅ |
| **Rate limit per API key / user** | ❌ | crude (per IP) | ✅ per user, per plan, per endpoint |
| **Aggregate 8 calls into 1** | ❌ | ❌ | ✅ |
| **REST → gRPC translation** | ❌ | ❌ | ✅ |
| **API versioning, deprecation** | ❌ | ❌ | ✅ |
| **Usage metering / billing** | ❌ | ❌ | ✅ |
| **Examples** | AWS NLB, LVS | Nginx, Varnish | Kong, AWS API Gateway, Zuul, Apigee |

**The one-liner:** *A load balancer is dumb about your app. A reverse proxy is smart about HTTP. An API gateway is smart about your API.* A reverse proxy sees "a POST to `/orders` with some bytes." A gateway sees "**customer 4021**, on the **free plan**, who has used **97 of their 100** monthly order-creations, calling the **v2 orders API**."

Practically, the boundary is fuzzy — Nginx with the right modules edges into gateway territory, and Kong is literally *built on* Nginx. Don't fight over labels; describe capabilities.

### 6. A minimal API Gateway in Express

Real, runnable, and small enough to read end to end. This is a great thing to be able to sketch in an interview.

```javascript
import express from 'express';
import jwt from 'jsonwebtoken';
import { createProxyMiddleware } from 'http-proxy-middleware';
import { createClient } from 'redis';

const app = express();
const redis = createClient({ url: process.env.REDIS_URL });
await redis.connect();

// ── Where the internal services actually live ────────────────────────────────
// In production these come from service discovery, not a constant.
const SERVICES = {
  users:    'http://user-service.internal:3001',
  orders:   'http://order-service.internal:3002',
  products: 'http://product-service.internal:3003',
  cart:     'http://cart-service.internal:3004',
};

// ── MIDDLEWARE 1 — Trace ID. Stamped here, flows through every service, so one
//    user's journey across 8 services can be reconstructed in the logs. ───────
app.use((req, res, next) => {
  req.traceId = req.headers['x-trace-id'] ?? crypto.randomUUID();
  res.setHeader('X-Trace-Id', req.traceId);
  next();
});

// ── MIDDLEWARE 2 — SECURITY: strip forged identity headers.
//    A client could send `X-User-Id: 1` and impersonate the admin. Downstream
//    services TRUST these headers, so only WE may ever set them. Must run
//    BEFORE auth. Forgetting this is a real, exploited vulnerability. ─────────
app.use((req, _res, next) => {
  delete req.headers['x-user-id'];
  delete req.headers['x-user-roles'];
  delete req.headers['x-user-plan'];
  next();
});

// ── MIDDLEWARE 3 — AUTHENTICATION. Validate the JWT exactly ONCE, here.
//    Downstream services contain zero auth code. ─────────────────────────────
const PUBLIC_ROUTES = [/^\/products/, /^\/auth\/login/, /^\/health/];

function authenticate(req, res, next) {
  if (PUBLIC_ROUTES.some(rx => rx.test(req.path))) return next();

  const header = req.headers.authorization ?? '';
  const token = header.startsWith('Bearer ') ? header.slice(7) : null;
  if (!token) return res.status(401).json({ error: 'Missing token', traceId: req.traceId });

  try {
    // jwt.verify checks signature AND expiry. Pin the algorithm explicitly —
    // omitting `algorithms` historically allowed the `alg: none` bypass attack.
    const claims = jwt.verify(token, process.env.JWT_PUBLIC_KEY, { algorithms: ['RS256'] });
    req.user = { id: claims.sub, roles: claims.roles ?? [], plan: claims.plan ?? 'free' };
    next();
  } catch (err) {
    const expired = err.name === 'TokenExpiredError';
    return res.status(401).json({ error: expired ? 'Token expired' : 'Invalid token' });
  }
}
app.use(authenticate);

// ── MIDDLEWARE 4 — RATE LIMITING. Counters live in Redis, not memory, so ALL
//    gateway instances share one count. (An in-memory counter silently
//    multiplies your limit by the number of gateway instances — topic 56.) ────
const LIMITS = { anonymous: 20, free: 100, pro: 1000, enterprise: 10000 }; // per minute

async function rateLimit(req, res, next) {
  const plan = req.user?.plan ?? 'anonymous';
  const identity = req.user?.id ?? req.ip;          // req.ip needs `trust proxy`
  const window = Math.floor(Date.now() / 60_000);   // fixed 1-minute window
  const key = `rl:${identity}:${window}`;

  // INCR + EXPIRE atomically in one round-trip. INCR on a missing key returns 1.
  const [count] = await redis.multi().incr(key).expire(key, 120).exec();

  const limit = LIMITS[plan];
  res.setHeader('X-RateLimit-Limit', limit);
  res.setHeader('X-RateLimit-Remaining', Math.max(0, limit - count));

  if (count > limit) {
    return res.status(429).json({ error: 'Rate limit exceeded', retryAfter: 60, plan });
  }
  next();
}
app.use(rateLimit);

// ── MIDDLEWARE 5 — ROUTING. Forward to the right service, attaching the
//    identity the service will trust. We forward CLAIMS, not the raw token. ───
function attachIdentity(proxyReq, req) {
  proxyReq.setHeader('X-Trace-Id', req.traceId);
  if (req.user) {
    proxyReq.setHeader('X-User-Id', req.user.id);
    proxyReq.setHeader('X-User-Roles', req.user.roles.join(','));
    proxyReq.setHeader('X-User-Plan', req.user.plan);
  }
}

for (const [name, target] of Object.entries(SERVICES)) {
  app.use(
    `/${name}`,
    createProxyMiddleware({
      target,
      changeOrigin: true,
      pathRewrite: { [`^/${name}`]: '' },   // /orders/123 → order-service /123
      onProxyReq: attachIdentity,
      proxyTimeout: 5000,
      onError: (err, req, res) => {         // a dead service must not kill the gateway
        console.error({ traceId: req.traceId, service: name, err: err.message });
        res.status(503).json({ error: `${name} unavailable`, traceId: req.traceId });
      },
    })
  );
}

// ── AGGREGATION — the N+1 fix. ONE client call, N parallel internal calls.
//    This is a BFF endpoint: it exists purely to serve the mobile home screen. ─
async function fetchOrNull(url, req, fallback) {
  // Every sub-call MUST be independently failure-tolerant. If recommendations
  // are down, the user should still get their profile and cart — a degraded
  // home screen beats a 500. This is graceful degradation at the edge.
  try {
    const r = await fetch(url, {
      headers: { 'X-User-Id': req.user.id, 'X-Trace-Id': req.traceId },
      signal: AbortSignal.timeout(800),   // one slow service must not stall the screen
    });
    return r.ok ? await r.json() : fallback;
  } catch {
    return fallback;
  }
}

app.get('/mobile/home', async (req, res) => {
  const uid = req.user.id;

  // Promise.all → all 5 calls go out IN PARALLEL over the fast internal LAN.
  // Total time ≈ the SLOWEST call, not the sum. This is the whole trick.
  const [profile, orders, cart, recommended, promos] = await Promise.all([
    fetchOrNull(`${SERVICES.users}/${uid}`,               req, null),
    fetchOrNull(`${SERVICES.orders}?userId=${uid}&limit=5`, req, []),
    fetchOrNull(`${SERVICES.cart}/${uid}`,                 req, { items: [] }),
    fetchOrNull(`${SERVICES.products}/recommended`,        req, []),   // nice-to-have
    fetchOrNull(`${SERVICES.products}/promotions`,         req, []),   // nice-to-have
  ]);

  if (!profile) return res.status(503).json({ error: 'Profile unavailable' }); // essential

  // Shape the payload for THIS client. Mobile wants small — send only what it renders.
  res.json({
    user:        { id: profile.id, name: profile.name, avatar: profile.avatarUrl },
    cartCount:   cart.items.length,
    recentOrders: orders.map(o => ({ id: o.id, status: o.status, total: o.total })),
    recommended: recommended.slice(0, 10),
    promotions:  promos,
    degraded:    recommended.length === 0,  // tell the client if it's a partial response
  });
});

app.get('/health', (_req, res) => res.json({ ok: true }));

app.listen(8080, () => console.log('API Gateway on :8080'));
```

**Notice what is NOT in this file:** no pricing rules, no order-state machine, no inventory checks. Every line is a **cross-cutting concern** — auth, rate limits, routing, tracing, aggregation. That discipline is the whole game, and it's the subject of the next section.

### 7. The risks — how gateways go wrong

**Risk 1: It becomes a monolith.** The gateway starts as thin routing. Then someone adds "just a small discount calculation" — it's convenient, it's right there. Then a currency conversion. Then a special case for enterprise customers. Two years later the gateway is 40,000 lines, **every team must change it to ship anything**, and it has become the exact coupled monolith that microservices were supposed to eliminate. You've re-centralised your organisation's bottleneck at the network edge.

**The rule, and say it in the interview: business logic does NOT go in the gateway.** Cross-cutting concerns only. The test: *"would every service need this, regardless of what it does?"* Auth — yes, every service needs it. Rate limiting — yes. "Apply 10% off for gold-tier customers on Tuesdays" — obviously not. That belongs in the pricing service.

**Risk 2: Single point of failure.** Everything goes through it. If it's down, your **entire API** is down — not one service, all of them. Mitigations, all of which you should name:
- Run **multiple gateway instances** behind a load balancer. Stateless (see [56](./56-horizontal-vs-vertical-scaling.md)) so any instance serves any request.
- Deploy across **multiple availability zones**.
- Keep it **fast and boring** — no heavy computation, no synchronous DB calls in the hot path. Cache JWT public keys; don't hit the auth DB per request.
- **Circuit breakers** ([73](./73-circuit-breaker.md)) so one dead downstream service can't exhaust the gateway's connection pool and take down routing for *every* service. This cascading failure is a genuine, common outage pattern.
- **Graceful degradation** on aggregation, as in the `fetchOrNull` code above.

**Risk 3: A performance bottleneck.** Every request pays the gateway's latency. Keep it under ~5–10ms of added overhead. Watch out for: synchronous crypto on a Node event loop, per-request database lookups, and — the classic — an aggregation endpoint that fans out **sequentially** instead of in parallel. (`await a; await b; await c;` instead of `Promise.all`. This mistake instantly turns your 60ms fan-out into 500ms.)

**Risk 4: Organisational bottleneck.** If one team owns the gateway and every other team must queue for their review to ship a route change, you've created a human single point of failure. Fix it with **declarative config** (routes as YAML in each team's repo) or with **BFFs owned by the client teams**.

---

## Visual / Diagram description

### Diagram 1 — The API Gateway in a microservices architecture

```
   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
   │  Mobile  │   │   Web    │   │  Smart   │   │ Partner  │
   │   App    │   │   SPA    │   │    TV    │   │   API    │
   └────┬─────┘   └────┬─────┘   └────┬─────┘   └────┬─────┘
        │ HTTPS/REST   │              │              │
        └──────────────┴──────┬───────┴──────────────┘
                              │  ONE public hostname: api.example.com
                              ▼
        ╔═════════════════════════════════════════════════════╗
        ║                   API  GATEWAY                       ║
        ║              (2+ instances, behind an LB)            ║
        ║ ┌─────────────────────────────────────────────────┐ ║
        ║ │ 1. Strip forged X-User-* headers   ◀── security  │ ║
        ║ │ 2. Validate JWT  ─── ONCE, HERE ───▶ 401 if bad  │ ║
        ║ │ 3. Rate limit (Redis, per user/plan) ▶ 429       │ ║
        ║ │ 4. Route by path  /orders/* → order-service      │ ║
        ║ │ 5. Translate protocol  REST ──▶ gRPC             │ ║
        ║ │ 6. Aggregate  (1 call in, N calls out)           │ ║
        ║ │ 7. Cache, transform, version                     │ ║
        ║ │ 8. Trace ID + metrics + logs                     │ ║
        ║ │ 9. Circuit breakers on every downstream          │ ║
        ║ └─────────────────────────────────────────────────┘ ║
        ╚══╤════════════╤═══════════╤═══════════╤═════════╤═══╝
           │            │           │           │         │
   forwards X-User-Id / X-User-Roles / X-Trace-Id downstream
           │            │           │           │         │
  ┌────────▼───┐ ┌──────▼─────┐ ┌───▼──────┐ ┌──▼──────┐ ┌▼──────────┐
  │   User     │ │   Order    │ │ Product  │ │ Payment │ │  Search   │
  │  Service   │ │  Service   │ │ Service  │ │ Service │ │  Service  │
  │  (Node)    │ │  (Node)    │ │ (Node)   │ │  (Go,   │ │ (Python)  │
  │            │ │            │ │          │ │  gRPC)  │ │           │
  │ NO auth    │ │ NO auth    │ │ NO auth  │ │ NO auth │ │ NO auth   │
  │ NO rate    │ │ NO rate    │ │ NO rate  │ │ NO rate │ │ NO rate   │
  │ limiting   │ │ limiting   │ │ limiting │ │ limiting│ │ limiting  │
  │ code       │ │ code       │ │ code     │ │ code    │ │ code      │
  └─────┬──────┘ └─────┬──────┘ └────┬─────┘ └────┬────┘ └─────┬─────┘
        │              │             │            │            │
     ┌──▼──┐        ┌──▼──┐      ┌───▼──┐     ┌───▼──┐     ┌───▼───┐
     │Users│        │Order│      │Prod  │     │ Pay  │     │ Elastic│
     │ DB  │        │ DB  │      │ DB   │     │  DB  │     │ search │
     └─────┘        └─────┘      └──────┘     └──────┘     └────────┘

     ◀────────────── PRIVATE NETWORK ──────────────────────────▶
     Not routable from the internet. The gateway is the ONLY door.
     THAT is why services can safely trust X-User-Id.
```

### Diagram 2 — The N+1 problem and how aggregation kills it

```
 WITHOUT AGGREGATION                     WITH AGGREGATION (BFF)
 ═══════════════════                     ══════════════════════

  Mobile (4G, 200ms RTT)                  Mobile (4G, 200ms RTT)
      │                                        │
      │──── 200ms ───▶ /users/me               │──── 200ms ───▶ /mobile/home
      │◀──────────────                         │◀──────────────
      │──── 200ms ───▶ /notifications          │            │
      │◀──────────────                         │            ▼
      │──── 200ms ───▶ /orders            ┌─────────────────────────────┐
      │◀──────────────                    │         GATEWAY              │
      │──── 200ms ───▶ /cart              │   Promise.all([ ... ])       │
      │◀──────────────                    │                              │
      │──── 200ms ───▶ /recommended       │    ├─5ms─▶ users             │
      │◀──────────────                    │    ├─8ms─▶ notifications     │
      │──── 200ms ───▶ /promotions        │    ├─6ms─▶ orders   PARALLEL │
      │◀──────────────                    │    ├─4ms─▶ cart     over the │
      │──── 200ms ───▶ /loyalty           │    ├─9ms─▶ recommend  LAN    │
      │◀──────────────                    │    ├─5ms─▶ promotions        │
      │──── 200ms ───▶ /tickets           │    ├─3ms─▶ loyalty           │
      │◀──────────────                    │    └─4ms─▶ tickets           │
      ▼                                   │                              │
  ══════════════                          │  total ≈ 9ms (the slowest,   │
  8 round-trips, 8 TLS setups             │  not the sum) + ~50ms overhead│
  ~1,600ms network + ~400ms server        └─────────────────────────────┘
  ─────────────                                     ▼
  ≈ 2,000 ms                                 1 round-trip ≈ 260 ms
                                             ──── 7.7x faster ────
```

**How to read these:** In Diagram 1, trace one request left to right. It hits **one** box that does nine things, then fans out to services that do **zero** of those things. In Diagram 2, count the arrows crossing the mobile network — 8 versus 1. The 8 internal calls didn't disappear; they moved to a network where they cost 5ms instead of 200ms.

---

## Real world examples

### 1. AWS API Gateway — the fully managed gateway

A managed service where you declare routes and it handles the rest. It integrates natively with **AWS Lambda** (a route can invoke a function with no server at all — see [89 — Serverless](./89-serverless-architecture.md)), does **authorization** via Cognito or a custom "Lambda authorizer" (your own function that validates the token), enforces **usage plans and API keys** (the metering that lets you *sell* your API), throttles per-key, and caches responses.

Its trade-off is instructive: you get enormous capability with zero servers to run, but you pay **per request**, which at very high volume becomes expensive relative to running your own Nginx/Kong — and you accept vendor lock-in on your routing configuration. It's the right default for low-to-medium volume and for serverless backends.

### 2. Kong — the open-source, plugin-based gateway

Kong is built **on top of Nginx** (with Lua scripting via OpenResty), which is a beautiful illustration of this topic's central idea: **an API gateway is a reverse proxy with API-awareness bolted on.**

Its model is **plugins**. You declare routes and services, then attach plugins: `jwt`, `rate-limiting`, `cors`, `request-transformer`, `prometheus`, `key-auth`. Each is a middleware in the request pipeline — conceptually identical to the Express middleware chain in the code above, just configured as data rather than written as code. You can run it self-hosted (no per-request cost) and it's a common choice for on-prem and Kubernetes deployments.

### 3. Netflix Zuul — the gateway that popularised the pattern

Netflix built **Zuul** as the front door to their enormous microservices fleet, and its story is the canonical case study.

Two things they're known for:
- **Dynamic filters.** Zuul's routing rules were loaded as Groovy scripts that could be pushed live, so operators could re-route or block traffic **without a deploy** — invaluable during an incident.
- **The BFF realisation.** Netflix streams to TVs, phones, browsers, and game consoles, and their needs diverged so hard that a single generic API served none of them well. This pressure is precisely what popularised **Backend-for-Frontend** — device teams owning their own tailored API layer, rather than one team's gateway trying to please everyone.

The pattern's biggest lesson also came from Netflix-scale operations: when everything flows through one component, **resilience at that component is everything.** Hence the heavy investment in circuit breakers (Hystrix) and fallbacks — a dead downstream service must degrade one part of the screen, never take down the front door.

---

## Trade-offs

| Benefit | The cost |
|---|---|
| Auth in one place, done once, done right | The gateway now holds keys — it's a high-value target |
| Services carry zero auth/rate-limit code | Services **blindly trust** headers — safe **only** if the network is closed and forged headers are stripped |
| 8 mobile round-trips → 1 (7.7x faster) | Aggregation logic must be failure-tolerant, or one dead service breaks the whole screen |
| Clients decoupled from internal topology | The gateway is coupled to *everything* — it changes whenever any service's contract does |
| Central metrics, tracing, canary routing | Every request pays the gateway's latency (aim for <10ms) |
| Rate limiting before traffic reaches services | Requires shared state (Redis) — another dependency, another failure mode |
| One place to enforce policy | **One place to take the whole API down** |
| Protocol translation (REST↔gRPC) | Adds serialization overhead on the hot path |

| Anti-pattern | What it looks like | The fix |
|---|---|---|
| **Gateway-as-monolith** | Pricing rules, discounts, workflow logic living in the gateway | Cross-cutting concerns only. Test: "does *every* service need this?" |
| **Gateway as SPOF** | One instance, one AZ | 2+ stateless instances, multi-AZ, behind an LB |
| **Cascading failure** | One dead service exhausts the gateway's connection pool → all routes die | Circuit breakers + per-downstream timeouts |
| **Sequential fan-out** | `await a; await b; await c;` in an aggregate endpoint | `Promise.all` — parallel, always |
| **Trusting inbound headers** | Client sends `X-User-Id: 1`, becomes admin | **Strip** all trusted headers on ingress, before auth |
| **Organisational bottleneck** | Every team queues on the gateway team to ship a route | Declarative per-team route config, or BFFs owned by client teams |

**Rule of thumb:** Use an API gateway the moment you have **more than a handful of services** or **more than one client type**. Put **cross-cutting concerns** in it — auth, rate limiting, routing, tracing, aggregation — and **never business logic**. Run at least two instances. Keep it thin, fast, and boring: the best gateway is one nobody thinks about. If you have exactly one backend service, **you don't need a gateway yet — you need a reverse proxy** ([57](./57-reverse-proxy.md)).

---

## Common interview questions on this topic

### Q1: "How is an API gateway different from a reverse proxy?"
**Hint:** A reverse proxy is **protocol-level** — it understands HTTP (TLS, headers, paths, caching, compression, buffering) but not your application. An API gateway is **API-aware** — it understands JWTs, users, API keys, plans, quotas, and versions. A reverse proxy sees "a POST to `/orders` with some bytes." A gateway sees "customer 4021, on the free plan, who has used 97 of 100 monthly order-creations." A gateway does everything a reverse proxy does, plus auth, per-user rate limiting, aggregation, and protocol translation. Kong is literally built on Nginx — that's the relationship in one fact.

### Q2: "Should downstream services also validate the JWT, or just trust the gateway?"
**Hint:** In a **closed network** (services on a private subnet, unreachable from the internet), trusting the gateway's headers is correct and standard — that's the whole point of validating once. But two conditions are non-negotiable: (1) the gateway must be the **only** ingress, and (2) the gateway must **strip** any inbound `X-User-*` headers, or a client can forge `X-User-Id: 1` and become the admin. In a **zero-trust** setup, the gateway instead mints a short-lived internal signed token that services verify — cheap, but not blind trust. Also flag the split: **coarse** authz at the gateway ("valid user? has `admin` scope?"), **fine-grained** authz in the service ("does user 4021 own order 9987?") — the latter is business logic and must not live at the edge.

### Q3: "The gateway is a single point of failure. Defend the design."
**Hint:** Concede it immediately — it's true and the interviewer knows. Then mitigate: run **2+ stateless instances** behind a load balancer, across **multiple AZs**; keep it thin and fast (no synchronous DB calls, cache the JWT public keys); add **circuit breakers** so one dead downstream can't exhaust the connection pool and cascade into total failure; use **graceful degradation** on aggregation (a dead recommendations service should blank one carousel, not 500 the home screen). Then make the counter-argument: the alternative — 20 services each independently exposed to the internet with 20 copies of auth code — is *far* worse for both security and availability.

### Q4: "A mobile app makes 8 API calls to render its home screen. Fix it."
**Hint:** Do the math out loud: 8 calls × ~200ms mobile RTT = **1.6s of pure network**. Then: add a **gateway aggregation endpoint** (`GET /mobile/home`) that fans out to all 8 services **in parallel** (`Promise.all`) over the internal LAN where each call costs ~5ms. One round-trip, ~260ms — a **7.7x** win. Then upgrade the answer: this is really the **Backend-for-Frontend (BFF)** pattern — give each client type its own gateway so mobile gets small payloads and web gets rich ones without compromise. Then name the trap: every sub-call needs its own **timeout and fallback**, or one slow service stalls the whole screen.

### Q5: "What should NEVER go in an API gateway?"
**Hint:** **Business logic.** Pricing rules, discount calculations, order state machines, inventory checks — all of it belongs in the owning service. The test: *"would every service need this, regardless of what it does?"* Auth, rate limiting, routing, tracing, CORS → yes, cross-cutting, put them in. "10% off for gold tier on Tuesdays" → no. If you don't hold this line, the gateway becomes a 40,000-line monolith that every team must modify to ship anything — you will have recreated the exact bottleneck microservices were meant to remove, and put it on your most critical path.

---

## Practice exercise

### Build a Gateway for a Food Delivery App

You have four microservices, each already running and each with **zero** auth code:

- `restaurant-service:3001` — `GET /restaurants`, `GET /restaurants/:id`, `GET /restaurants/:id/menu`
- `order-service:3002` — `POST /orders`, `GET /orders?userId=X`, `GET /orders/:id`
- `user-service:3003` — `GET /users/:id`, `GET /users/:id/addresses`
- `delivery-service:3004` — `GET /deliveries/:orderId/eta`

Clients: a **mobile app** and a **restaurant-owner web dashboard**.

**Build (in Express, ~30–40 minutes):**

1. **Routing** — map `/restaurants/*`, `/orders/*`, `/users/*`, `/delivery/*` to the four services.
2. **Auth** — validate a JWT; forward `X-User-Id` and `X-User-Roles` downstream. Make `GET /restaurants` and `GET /restaurants/:id/menu` **public** (browsing before login), everything else authenticated.
3. **The security bug** — write the middleware that strips inbound `X-User-*` headers. Then **prove the vulnerability exists without it**: `curl -H "X-User-Id: 1" localhost:8080/orders` with no token and watch yourself become user 1. Then add the strip and prove it's fixed. *Do this one — it's the exercise's real lesson.*
4. **Rate limiting** — Redis-backed: 20/min anonymous, 100/min authenticated. Return `429` with `X-RateLimit-Remaining`.
5. **Aggregation** — build `GET /mobile/order-tracking/:orderId` that returns, from ONE client call:
   - the order (order-service)
   - the restaurant's name + phone (restaurant-service)
   - the delivery ETA (delivery-service)
   - the user's delivery address (user-service)

   Use `Promise.all`. Give **every** sub-call a timeout. If `delivery-service` is down, the endpoint must **still return the order** with `eta: null` — not a 500.

**Then answer, in writing:**

- Sketch a **second, separate BFF** for the restaurant-owner dashboard. What does it need that mobile doesn't? What does it *not* need? Why is one shared endpoint the wrong call?
- **Kill `delivery-service`** and hit your aggregation endpoint 50 times. What happens to gateway latency? Now add a **circuit breaker** (after 5 failures, skip the call entirely for 30s and return `eta: null` immediately). Measure the difference. That gap is why circuit breakers exist.
- List three things a junior might be tempted to add to this gateway that would be **business logic**, and say which service each belongs in instead.

---

## Quick reference cheat sheet

- **API Gateway = the single front door** to a microservices system. One public hostname, N private services.
- **Its jobs:** routing, auth, rate limiting, transformation, protocol translation, caching, versioning, metrics/tracing, canary routing, aggregation.
- **Validate the JWT ONCE, at the edge.** Downstream services then contain **zero** auth code — they trust `X-User-Id`.
- **That trust is only safe if** (a) the internal network is closed, and (b) the gateway **strips inbound `X-User-*` headers**. Forgetting (b) lets any client become admin.
- **Coarse authz at the gateway** ("valid user? has the scope?"). **Fine-grained authz in the service** ("does user 4021 own order 9987?") — that's business logic.
- **N+1 client calls:** 8 mobile calls × 200ms RTT = 1.6s. Aggregate into 1 call + parallel LAN fan-out = ~260ms. **7.7x.**
- **Always `Promise.all`** in an aggregation endpoint. Sequential `await`s silently destroy the entire benefit.
- **Every sub-call needs a timeout and a fallback.** A dead recommendations service should blank one carousel, not 500 the home screen.
- **BFF (Backend-for-Frontend):** one gateway *per client type*. Mobile gets tiny payloads; web gets rich ones. Netflix popularised it.
- **Gateway vs reverse proxy:** the gateway is **API-aware** (users, tokens, quotas, versions); the proxy is **protocol-level** (HTTP, TLS, caching). Kong is built on Nginx — that's literally the relationship.
- **Gateway vs load balancer:** an LB just spreads traffic. The gateway does that plus everything else.
- **The #1 anti-pattern: business logic in the gateway.** It becomes a monolith every team must edit. Test: *"would every service need this?"*
- **The #1 risk: single point of failure.** Run 2+ stateless instances, multi-AZ, with **circuit breakers** so one dead service can't cascade.
- **Real ones:** Kong (Nginx+Lua, self-hosted), AWS API Gateway (managed, pay-per-request, Lambda-native), Netflix Zuul (dynamic filters, birthplace of BFF), Apigee, Envoy.
- **One service? You don't need a gateway — you need a reverse proxy.**

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [57 — Reverse Proxy](./57-reverse-proxy.md) — the protocol-level component an API gateway is built on top of |
| **Next** | [59 — Caching in Depth](./59-caching-in-depth.md) — the gateway is one of the best places to cache, and it can cache per-user |
| **Related** | [70 — Rate Limiting](./70-rate-limiting.md) — the algorithms (token bucket, sliding window) behind the gateway's limiter |
| **Related** | [71 — Microservices vs Monolith](./71-microservices-vs-monolith.md) — the architecture that makes a gateway necessary in the first place |
