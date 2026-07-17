# 129 — Design an API Gateway (Class Level)
## Category: LLD Case Study

---

## What is this?

An **API gateway** is the single front door to a backend made of many services. Every request goes through it first, gets checked and shaped by a series of steps, and only then gets **routed** to the real service that answers it. This doc is the **class-level** (LLD) companion to the higher-level design in [58 — API Gateway](./58-api-gateway.md): here we zoom all the way down to the objects and methods.

Think of it like the **security-and-reception desk of an office building**. A visitor doesn't walk straight to the person they want. They pass the guard (are you allowed in?), sign the visitor log (record who came), get a badge (rate-limited passes), and only then does reception direct them to the right floor. Each desk does one job and hands the visitor to the next desk.

---

## Why does it matter?

Without a gateway, every backend service has to re-implement the boring, dangerous, cross-cutting stuff itself: authentication, rate limiting, logging, request shaping. Ten services means ten copies of auth code — ten places to get it wrong.

**Interview angle:** "Design an API gateway" is a beloved LLD prompt because it is *the* canonical **Chain of Responsibility** problem. If you can name the pattern, draw the middleware chain, and code a 10-line engine that runs it, you've demonstrated real pattern fluency. Bonus points for connecting it to how Express/Koa actually work.

**Real-work angle:** Every Node backend you've touched already does this. `app.use(...)` in Express IS a chain of middleware. Understanding the LLD means you can write custom middleware confidently, reason about **order** (why auth must run before routing), and debug the "why did my request stop halfway?" bugs.

---

## The core idea — explained simply

### Analogy: the airport security line

You arrive at the airport and walk a **single line** of checkpoints, in a fixed order:

1. **Ticket check** — no valid ticket? You're turned away *right here*. You never reach the gate.
2. **Rate/queue control** — too many people from one tour group at once? Some are told to wait.
3. **Logging desk** — your passage is recorded (in and out).
4. **Gate routing** — finally, you're directed to the correct gate (your actual flight).

Two things make this powerful. First, **order matters**: ticket check must come before the gate, or unticketed people reach planes. Second, any checkpoint can **short-circuit** you — fail the ticket check and you're ejected immediately; the later checkpoints never see you.

That is *exactly* an API gateway, and the code shape that models it is the **Chain of Responsibility** pattern (recall [47 — Chain of Responsibility](./47-pattern-chain-of-responsibility.md)).

| Airport thing | API gateway thing | Code thing |
|---|---|---|
| A checkpoint | A **middleware** / handler | `class AuthMiddleware { handle(req, res, next) }` |
| The single line | The **ordered pipeline** | `Gateway.middlewares = [auth, rateLimit, log, route]` |
| "Pass to next desk" | Call `next()` | `next()` invokes the next handler |
| "Turned away here" | **Short-circuit** with a response | write `res` and DON'T call `next()` |
| The gate director | The **router** | `RoutingMiddleware` matches path → service |
| "Which line/desk?" policy | The **RoutingStrategy** | pluggable match/load-balance policy |

Every part maps cleanly. Hold this table in your head; the rest is implementation.

---

## Key concepts inside this topic

This is an LLD case study, so we follow the **7-step LLD method** (recall [111 — LLD Approach Framework](./111-lld-approach-framework.md)): requirements → nouns → behaviours → core design → class diagram → code → patterns.

### 1. Requirements & clarifying questions

**Functional requirements.** The gateway must:
- **Receive** an incoming request (method, path, headers, body).
- Run it through a **pipeline of cross-cutting concerns**: authentication, rate limiting, logging, request transformation.
- **Route** the surviving request to the correct backend service.
- Pass the backend's **response** back to the caller (possibly transformed).

**Non-functional requirements.** The pipeline must be:
- **Composable** — add/remove a concern without touching the others.
- **Order-dependent** — auth before rate-limit before logging before routing. Order is a first-class design decision, not an accident.
- **Short-circuitable** — a failed check (bad token, over rate limit) stops the pipeline and returns immediately.

**Clarifying questions to ask the interviewer** (asking these is half the score):
- *Are handlers synchronous or async?* Real backends do async I/O (auth lookups, network calls), so we design `handle` to be `async` and `await next()`.
- *Per-route middleware, or one global chain?* We build a global chain first, then show per-route as an extension (step 7).
- *Do we need response transformation, or just request handling?* We keep a `Response` object flowing through so any middleware can shape the response on the way out (logging does exactly this).
- *One backend per path, or load-balance across instances?* Start with one; make the choice pluggable via a **RoutingStrategy**.

### 2. Identify the core objects (nouns)

Underline the nouns in the requirements:

| Object | Responsibility |
|---|---|
| `Gateway` | Holds the **ordered** middleware list; builds and kicks off the chain per request. |
| `Request` | Immutable-ish carrier of method, path, headers, body. |
| `Response` | status + body + a `send()` helper + a `sent` flag (so we know if someone short-circuited). |
| `Middleware` (base) | Abstract `handle(req, res, next)` — one link in the chain. |
| `AuthMiddleware` | Concern: is the caller authenticated? |
| `RateLimitMiddleware` | Concern: is the caller under their quota? |
| `LoggingMiddleware` | Concern: record method+path in, status out. |
| `RoutingMiddleware` | Concern: match path → backend, call it, fill the response. |
| `Route` / `RouteTable` | Mapping of path-prefix → backend service. |
| `RoutingStrategy` | Pluggable policy for *how* to pick a backend (prefix match, round-robin). |
| `Service` (stubs) | Backend services that return a canned response. |

### 3. Behaviours + ownership (who does what)

- **`Gateway.handle(request)`** owns *orchestration only*: it constructs the `next` chain and starts it. It does **no** business logic. High cohesion — it's a tiny engine.
- **Each middleware owns exactly one concern** (Single Responsibility Principle). `AuthMiddleware` knows nothing about routing; `RoutingMiddleware` knows nothing about tokens. Each either does its job and calls `next()` to pass control **down** the chain, or **short-circuits** by writing a response and *not* calling `next()`.
- **`RoutingMiddleware`** owns the match-and-dispatch step, but delegates the *policy* of picking a backend to a **`RoutingStrategy`** (Strategy pattern — recall [42 — Strategy](./42-pattern-strategy.md)).
- **`Service`** owns producing the actual answer.

This clean split is why you can drop in a new concern (CORS, caching) as a new middleware and change *nothing else*.

### 4. The core design — CHAIN OF RESPONSIBILITY (the star)

This is the heart of the whole design, so slow down here.

**Chain of Responsibility** (from [47](./47-pattern-chain-of-responsibility.md)) says: give a request to a **chain of handlers**; each handler decides whether to process it and/or pass it along. In a gateway, *every* handler processes and (usually) passes along — but any handler may stop the chain.

Each `Middleware` exposes:

```
async handle(request, response, next)
```

It does its work, then either:
- **calls `await next()`** — hands control to the *next* middleware in the chain, OR
- **short-circuits** — writes to `response` (e.g. `response.send(401, ...)`) and **returns without calling `next()`**. Everything downstream is skipped.

**This is literally how Express/Koa middleware works.** When you write:

```js
app.use((req, res, next) => {
  if (!req.headers.authorization) return res.status(401).send('nope'); // short-circuit
  next(); // pass control down
});
```

...that `(req, res, next) => {}` is a Chain-of-Responsibility handler. `next()` advances the chain; returning without it stops the request cold. The gateway we build IS a tiny Express.

**The chain, drawn:**

```
             ┌───────────────────────────────────────────────────────────┐
 request ──▶ │  Auth  ─next→  RateLimit  ─next→  Logging  ─next→  Routing  │ ──▶ Service
             └───────────────────────────────────────────────────────────┘
                 │                │                                  ▲
        fail? ✗  ▼        fail? ✗ ▼                                  │
              401             429                              response flows
           (short-circuit)  (short-circuit)                    back up the chain
```

Control flows **left to right** via `next()`. A short-circuit at any link sends the response straight back without touching the links to its right. On success, `Routing` calls the backend, fills the response, and control **unwinds back up** — which is how `Logging` can log the *final status* after `next()` returns.

**Strategy for routing.** *Which* backend a path maps to, and how to pick among instances, is a policy that can vary (exact-prefix match today; round-robin load balancing tomorrow — recall [70 — Rate Limiting](./70-rate-limiting.md) for the counting idea we reuse, and load-balancing basics). We inject a **`RoutingStrategy`** so `RoutingMiddleware` never hard-codes the selection logic. That's the Strategy pattern (recall [42](./42-pattern-strategy.md)).

### 5. Class diagram (ASCII)

```
                         ┌──────────────────────────────┐
                         │            Gateway            │
                         │  - middlewares: Middleware[]  │  (ORDERED)
                         │  + use(mw)                    │
                         │  + handle(request): Response  │  ← builds next-chain, kicks off
                         └───────────────┬──────────────┘
                                         │ runs in order
                 ┌───────────────────────┼───────────────────────────┐
                 ▼                        ▼                           ▼
        ┌─────────────────┐   ┌──────────────────────┐   ┌────────────────────────┐
        │   «abstract»    │   │   «abstract»         │   │   «abstract»           │
        │   Middleware    │◀──│   Middleware         │◀──│   Middleware           │
        │ + handle(req,   │   │  (base for all)      │   │                        │
        │   res, next)    │   └──────────────────────┘   └────────────────────────┘
        └────────┬────────┘
                 │ extends
   ┌─────────────┼───────────────┬────────────────────┬─────────────────────────┐
   ▼             ▼               ▼                    ▼                         ▼
┌────────┐ ┌──────────────┐ ┌──────────────┐ ┌────────────────────────────────────┐
│ Auth   │ │ RateLimit    │ │ Logging      │ │ RoutingMiddleware                  │
│ Middle │ │ Middleware   │ │ Middleware   │ │  - routeTable: RouteTable          │
│ ware   │ │ (token       │ │              │ │  - strategy: RoutingStrategy ──────┼──▶ «abstract»
└────────┘ │  bucket)     │ └──────────────┘ │  + handle(...)                     │    RoutingStrategy
           └──────────────┘                  └───────────────┬────────────────────┘   + pick(routes,req)
                                                             │ dispatches to           ▲
                                                             ▼                          │ implements
   ┌──────────────┐         ┌──────────────────┐   ┌──────────────────┐   ┌────────────────────────┐
   │ Request      │         │ Response         │   │  «abstract»      │   │ PathPrefixStrategy     │
   │ method,path, │         │ status, body,    │   │  Service         │   │ RoundRobinStrategy     │
   │ headers,body │         │ sent, send()     │   │  + handle(req)   │   └────────────────────────┘
   └──────────────┘         └──────────────────┘   └────────┬─────────┘
                                                            │ extends
                                              ┌─────────────┴──────────────┐
                                              ▼                            ▼
                                        ┌────────────┐              ┌────────────┐
                                        │ UserService│              │OrderService│
                                        └────────────┘              └────────────┘
```

### 6. Full JavaScript implementation

Complete and runnable — save as `gateway.js` and run `node gateway.js`.

```js
'use strict';

// ─────────────────────────────────────────────────────────────
// Request & Response — the two objects that flow through the chain
// ─────────────────────────────────────────────────────────────

class Request {
  constructor(method, path, headers = {}, body = null) {
    this.method = method;         // "GET", "POST", ...
    this.path = path;             // "/users/42"
    this.headers = headers;       // { authorization: "token-abc", "x-client-id": "c1" }
    this.body = body;             // parsed body, if any
  }
}

class Response {
  constructor() {
    this.status = null;   // set on send()
    this.body = null;
    this.sent = false;    // did some middleware already respond? (short-circuit flag)
  }

  // send() writes the response ONCE. The `sent` flag lets the engine and
  // later middleware know the chain has produced an answer.
  send(status, body) {
    if (this.sent) return this;  // ignore double-send (defensive)
    this.status = status;
    this.body = body;
    this.sent = true;
    return this;
  }
}

// ─────────────────────────────────────────────────────────────
// Middleware base — one link in the Chain of Responsibility
// ─────────────────────────────────────────────────────────────

class Middleware {
  // Subclasses MUST override. Signature mirrors Express: (req, res, next).
  // Do work, then EITHER `await next()` to pass control down the chain,
  // OR write to `res` (short-circuit) and return without calling next().
  async handle(req, res, next) {
    throw new Error('Middleware.handle() is abstract — override it');
  }
}

// ─────────────────────────────────────────────────────────────
// AuthMiddleware — concern: is the caller authenticated?
// ─────────────────────────────────────────────────────────────

class AuthMiddleware extends Middleware {
  constructor(validTokens) {
    super();
    this.validTokens = new Set(validTokens); // e.g. Set(["token-abc", "token-xyz"])
  }

  async handle(req, res, next) {
    const token = req.headers.authorization;
    if (!token || !this.validTokens.has(token)) {
      // SHORT-CIRCUIT: bad/missing token. Respond 401 and DO NOT call next().
      // Everything downstream (rate-limit, logging-of-route, routing) is skipped.
      res.send(401, { error: 'Unauthorized: missing or invalid token' });
      return;
    }
    // authenticated → attach identity for downstream middleware, then continue
    req.headers['x-authenticated'] = 'true';
    await next();
  }
}

// ─────────────────────────────────────────────────────────────
// RateLimitMiddleware — concern: is the caller under quota?
// Simple token-bucket per client (reuse the idea from 70 — Rate Limiting).
// ─────────────────────────────────────────────────────────────

class RateLimitMiddleware extends Middleware {
  constructor({ capacity = 3, refillPerSec = 1 } = {}) {
    super();
    this.capacity = capacity;         // max tokens in a bucket
    this.refillPerSec = refillPerSec; // tokens added per second
    this.buckets = new Map();         // clientId -> { tokens, lastRefill }
  }

  _clientId(req) {
    return req.headers['x-client-id'] || 'anonymous';
  }

  _take(clientId) {
    const now = Date.now();
    let b = this.buckets.get(clientId);
    if (!b) {
      b = { tokens: this.capacity, lastRefill: now };
      this.buckets.set(clientId, b);
    }
    // refill based on elapsed time, capped at capacity
    const elapsedSec = (now - b.lastRefill) / 1000;
    b.tokens = Math.min(this.capacity, b.tokens + elapsedSec * this.refillPerSec);
    b.lastRefill = now;

    if (b.tokens >= 1) {
      b.tokens -= 1;
      return true;   // allowed
    }
    return false;    // over limit
  }

  async handle(req, res, next) {
    const clientId = this._clientId(req);
    if (!this._take(clientId)) {
      // SHORT-CIRCUIT: over the limit → 429. Do not call next().
      res.send(429, { error: `Too Many Requests for client '${clientId}'` });
      return;
    }
    await next();
  }
}

// ─────────────────────────────────────────────────────────────
// LoggingMiddleware — concern: record request in, status out.
// Note it logs BEFORE next() and AFTER next() returns — the "unwind".
// ─────────────────────────────────────────────────────────────

class LoggingMiddleware extends Middleware {
  async handle(req, res, next) {
    const start = Date.now();
    console.log(`  [LOG] --> ${req.method} ${req.path}`);
    await next();                       // let the rest of the chain run
    const ms = Date.now() - start;
    // After next() returns, res is fully populated — log the OUTCOME.
    console.log(`  [LOG] <-- ${req.method} ${req.path}  status=${res.status}  (${ms}ms)`);
  }
}

// ─────────────────────────────────────────────────────────────
// Routing — Strategy pattern for HOW to pick a backend (recall 42)
// ─────────────────────────────────────────────────────────────

class RoutingStrategy {
  // routes: array of { prefix, services: Service[] }
  pick(routes, req) {
    throw new Error('RoutingStrategy.pick() is abstract');
  }
}

// Match by longest path-prefix; use the first backend instance.
class PathPrefixStrategy extends RoutingStrategy {
  pick(routes, req) {
    const matches = routes
      .filter(r => req.path.startsWith(r.prefix))
      .sort((a, b) => b.prefix.length - a.prefix.length); // longest prefix wins
    const route = matches[0];
    return route ? route.services[0] : null;
  }
}

// Same matching, but round-robin across a route's backend instances (load balance).
class RoundRobinPrefixStrategy extends RoutingStrategy {
  constructor() { super(); this._cursor = new Map(); }
  pick(routes, req) {
    const route = routes
      .filter(r => req.path.startsWith(r.prefix))
      .sort((a, b) => b.prefix.length - a.prefix.length)[0];
    if (!route) return null;
    const i = (this._cursor.get(route.prefix) || 0) % route.services.length;
    this._cursor.set(route.prefix, i + 1);
    return route.services[i];
  }
}

// A tiny route table: prefix -> one or more backend instances.
class RouteTable {
  constructor() { this.routes = []; }         // [{ prefix, services }]
  add(prefix, ...services) {
    this.routes.push({ prefix, services });
    return this;
  }
}

class RoutingMiddleware extends Middleware {
  constructor(routeTable, strategy = new PathPrefixStrategy()) {
    super();
    this.routeTable = routeTable;
    this.strategy = strategy; // pluggable policy (Strategy pattern)
  }

  async handle(req, res, next) {
    const service = this.strategy.pick(this.routeTable.routes, req);
    if (!service) {
      res.send(404, { error: `No route for ${req.path}` });
      return; // terminal — nothing downstream to call anyway
    }
    // Dispatch to the backend and fill the response.
    const backendResponse = await service.handle(req);
    res.send(backendResponse.status, backendResponse.body);
    // Routing is usually the LAST link; but we still call next() so any
    // trailing middleware (e.g. response-transform) can run. Harmless if none.
    await next();
  }
}

// ─────────────────────────────────────────────────────────────
// Backend Service stubs
// ─────────────────────────────────────────────────────────────

class Service {
  constructor(name) { this.name = name; }
  async handle(req) { throw new Error('Service.handle() is abstract'); }
}

class UserService extends Service {
  async handle(req) {
    return { status: 200, body: { service: this.name, msg: `user data for ${req.path}` } };
  }
}

class OrderService extends Service {
  async handle(req) {
    return { status: 200, body: { service: this.name, msg: `orders for ${req.path}` } };
  }
}

// ─────────────────────────────────────────────────────────────
// Gateway — the mini Express engine. Holds ORDERED middleware and
// runs the Chain of Responsibility. THIS is the payoff.
// ─────────────────────────────────────────────────────────────

class Gateway {
  constructor() { this.middlewares = []; }

  // Register middleware IN ORDER — like Express app.use(...).
  use(middleware) { this.middlewares.push(middleware); return this; }

  // Build the `next` chain and kick it off. ~10 lines — this is the whole engine.
  async handle(request) {
    const res = new Response();
    const stack = this.middlewares;

    // dispatch(i) invokes middleware i, giving it a `next` that dispatches i+1.
    const dispatch = async (i) => {
      if (i >= stack.length) return;       // end of chain
      const mw = stack[i];
      // next = run the following middleware. A middleware that short-circuits
      // simply never calls this, so the rest of the chain is skipped.
      await mw.handle(request, res, () => dispatch(i + 1));
    };

    await dispatch(0);

    // Safety net: nobody produced a response.
    if (!res.sent) res.send(500, { error: 'No middleware handled the request' });
    return res;
  }
}

// ─────────────────────────────────────────────────────────────
// main() — wire it up and prove the three behaviours
// ─────────────────────────────────────────────────────────────

async function main() {
  // Route table: two backends behind path prefixes.
  const routes = new RouteTable()
    .add('/users', new UserService('UserService'))
    .add('/orders', new OrderService('OrderService'));

  // Register middleware IN ORDER — auth → rate-limit → logging → routing.
  const gateway = new Gateway()
    .use(new AuthMiddleware(['token-abc']))
    .use(new RateLimitMiddleware({ capacity: 3, refillPerSec: 1 }))
    .use(new LoggingMiddleware())
    .use(new RoutingMiddleware(routes, new PathPrefixStrategy()));

  const authed = { authorization: 'token-abc', 'x-client-id': 'c1' };

  // (a) Valid authenticated request — flows through the WHOLE chain to UserService.
  console.log('\n=== (a) valid request → reaches backend ===');
  let r = await gateway.handle(new Request('GET', '/users/42', authed));
  console.log('  RESULT:', r.status, JSON.stringify(r.body));

  // (b) No token — short-circuited at Auth with 401. Router never runs.
  console.log('\n=== (b) missing token → short-circuit 401 at auth ===');
  r = await gateway.handle(new Request('GET', '/users/42', { 'x-client-id': 'c2' }));
  console.log('  RESULT:', r.status, JSON.stringify(r.body));

  // (c) Flood from one client — bucket capacity 3, so the 4th+ hit 429.
  console.log('\n=== (c) flood from client c1 → rate limit 429 ===');
  for (let n = 1; n <= 5; n++) {
    r = await gateway.handle(new Request('GET', '/orders/9', authed));
    console.log(`  request #${n} → RESULT:`, r.status, JSON.stringify(r.body));
  }
}

main();
```

**Expected output (abridged):**

```
=== (a) valid request → reaches backend ===
  [LOG] --> GET /users/42
  [LOG] <-- GET /users/42  status=200  (0ms)
  RESULT: 200 {"service":"UserService","msg":"user data for /users/42"}

=== (b) missing token → short-circuit 401 at auth ===
  RESULT: 401 {"error":"Unauthorized: missing or invalid token"}   ← no [LOG] lines: auth stopped it before logging

=== (c) flood from client c1 → rate limit 429 ===
  request #1 → RESULT: 200 {...}
  request #2 → RESULT: 200 {...}
  request #3 → RESULT: 200 {...}
  request #4 → RESULT: 429 {"error":"Too Many Requests for client 'c1'"}
  request #5 → RESULT: 429 {"error":"Too Many Requests for client 'c1'"}
```

Notice in (b) there are **no log lines** — auth short-circuited *before* the logging middleware, proving the chain really stops. In (c), the first 3 pass (bucket capacity 3), then 429s kick in. All three behaviours — full flow, short-circuit, rate-limit — come from the same 10-line engine.

### 7. Design patterns used and WHY

| Pattern | Where | Why |
|---|---|---|
| **Chain of Responsibility** (star) | The middleware pipeline: `Gateway.handle` + each `Middleware.handle(req, res, next)` | Decouples the *sequence* of cross-cutting concerns from each concern's logic. Any link can process-and-pass or short-circuit. This IS Express/Koa middleware. Recall [47](./47-pattern-chain-of-responsibility.md). |
| **Strategy** | `RoutingStrategy` (`PathPrefixStrategy`, `RoundRobinPrefixStrategy`) | The *policy* for picking a backend varies (exact match vs. load-balanced). Injecting it keeps `RoutingMiddleware` closed to modification. Recall [42](./42-pattern-strategy.md). |
| **Template Method (light)** | `Middleware` / `Service` abstract bases | Define the contract (`handle`) once; subclasses fill in behaviour. |

**Connection to the HLD ([58 — API Gateway](./58-api-gateway.md)):** this doc is the *class design of the request pipeline*. The HLD covers the bigger picture — authentication at the edge, request **aggregation** (fanning one client call into several backend calls), TLS termination, and how you scale many gateway instances behind a load balancer. Here we've built the engine that lives *inside* one gateway instance.

---

## Visual / Diagram description

The sequence of one **valid** request unwinding through the chain:

```
Client        Gateway        Auth        RateLimit      Logging      Routing      UserService
  │   handle()   │             │             │             │            │              │
  │─────────────▶│  dispatch(0)│             │             │            │              │
  │              │────────────▶│ ok, next()  │             │            │              │
  │              │             │────────────▶│ ok, next()  │            │              │
  │              │             │             │────────────▶│ log -->    │              │
  │              │             │             │             │───────────▶│ pick()       │
  │              │             │             │             │            │─────────────▶│ handle()
  │              │             │             │             │            │◀─────────────│ 200
  │              │             │             │             │◀───────────│ res.send(200)│
  │              │             │             │             │ log <-- 200│  (unwind)    │
  │◀─────────────│  Response(200)                                       │              │
```

Read top-to-bottom for the **call** direction (`next()` going deeper), and note the `◀` arrows are the **unwind** — the same reason `LoggingMiddleware` can log the final status *after* `await next()` returns. A short-circuit (401/429) simply means one of the earlier arrows turns back left immediately without reaching the ones to its right.

---

## Real world examples

### Express.js (Node) — representative

Express's entire request handling is a Chain of Responsibility. `app.use(fn)` pushes `fn` onto an ordered stack; each `fn(req, res, next)` calls `next()` to continue or ends the response to short-circuit. Popular middleware like `express-rate-limit`, `morgan` (logging), and `passport` (auth) are just links you insert — exactly our `RateLimitMiddleware`, `LoggingMiddleware`, `AuthMiddleware`. Our `Gateway.dispatch` is a stripped-down copy of Express's router `next()` engine.

### Kong / NGINX-based gateways — representative

Kong (built on OpenResty/NGINX) runs **plugins** in ordered phases (auth, rate-limiting, logging, transformation) around proxying to upstream services. Conceptually identical: an ordered chain of pluggable concerns wrapping a routing/proxy step. Each plugin can terminate the request early (e.g. the key-auth plugin returning 401).

### AWS API Gateway — representative

AWS API Gateway lets you attach authorizers, throttling (rate limits), request/response mapping templates (transformation), and then integrates (routes) to Lambda or an HTTP backend. The *order* — authorize, throttle, transform, integrate — mirrors our chain. Throttling short-circuits with 429; a failing authorizer short-circuits with 401/403, before your backend is ever invoked.

---

## Trade-offs

| Aspect | Pro | Con |
|---|---|---|
| Chain of Responsibility | Add/remove/reorder concerns trivially (OCP); each link testable in isolation | Order is implicit and easy to get wrong (put logging before auth and you log rejected calls); deep chains can be hard to trace |
| Single global chain | Simple; one place to reason about the pipeline | Every route pays for every middleware unless you branch (see per-route extension) |
| Strategy for routing | Swap match/LB policy without touching routing code | One more indirection to understand |
| Short-circuiting | Fast rejection; backends protected from bad traffic | A buggy middleware that forgets `next()` silently stalls requests — the classic Express bug |
| Middleware objects (vs. bare functions) | Can hold state (rate-limit buckets), easy to unit test | Slightly more ceremony than `(req,res,next)=>{}` |

**The sweet spot:** a single **ordered** chain of small, single-responsibility middleware, with routing policy behind a Strategy. It's exactly what Express, Koa, Kong, and the cloud gateways converged on — because it's composable, order-explicit, and cheap to extend.

---

## Common interview questions on this topic

### Q1: "Which design pattern models a middleware pipeline, and why?"
**Hint:** Chain of Responsibility. Each handler gets `(req, res, next)`; it processes then either calls `next()` to pass control down, or short-circuits by responding without calling `next()`. It decouples the *order* of concerns from each concern's logic. Name Express `app.use` as the canonical real-world instance.

### Q2: "Why does middleware order matter? Give a concrete bug."
**Hint:** Concerns depend on each other. Auth must run before rate-limiting per-user (you need the identity), and both must run before routing (never dispatch an unauthenticated/over-quota call to a backend). Put logging *before* auth and you'll log-and-count requests that get rejected; put routing first and short-circuits become impossible.

### Q3: "How does a middleware stop the chain?"
**Hint:** It writes the response (`res.send(401, ...)`) and **returns without calling `next()`**. In our engine, `next` is `() => dispatch(i+1)`; not calling it means `dispatch` for the remaining links never runs. This is the single most common Express bug when done by accident (forgetting `next()` stalls the request).

### Q4: "Add per-route middleware. How?"
**Hint:** Attach a middleware list to each `Route`; in `RoutingMiddleware` (or a small router), after matching the route, run its route-specific chain before/around dispatch — reusing the same `dispatch` engine. The global chain is unchanged (OCP).

### Q5: "Where does load balancing across backend instances live?"
**Hint:** In the `RoutingStrategy`. Swap `PathPrefixStrategy` for `RoundRobinPrefixStrategy` (already in the code) — `RoutingMiddleware` needs zero changes. That's the Strategy pattern earning its keep.

---

## Practice exercise

### Build and extend the gateway

1. Copy the full implementation into `gateway.js` and run it (`node gateway.js`). Confirm you see: the valid request reaching `UserService`, the tokenless request short-circuiting at 401 **with no log lines**, and the flood hitting 429 on the 4th request.
2. **Add a `RequestIdMiddleware`** that generates a random `x-request-id`, attaches it to `req.headers`, and prints it in the log. Insert it **first** in the chain. Prove existing middleware needed **zero** changes (that's the Open/Closed Principle).
3. **Add a `CachingMiddleware`** placed *after* auth+rate-limit but *before* routing: for `GET` requests, if you've seen this exact path before, short-circuit with the cached response (status 200, `x-cache: HIT`); otherwise call `next()` and cache what routing produced.
4. Switch the routing strategy to `RoundRobinPrefixStrategy`, register **two** `UserService` instances for `/users`, and confirm alternating instances answer.

Produce: the modified `gateway.js` plus a short note on where each new middleware sits in the order and why.

---

## Quick reference cheat sheet

- **API gateway** — single front door; runs cross-cutting concerns, then routes to a backend.
- **Chain of Responsibility** — THE pattern here: ordered handlers, each `handle(req, res, next)`.
- **`next()`** — call it to pass control **down** the chain; don't call it to **short-circuit**.
- **Short-circuit** — write a response and return without `next()` (auth → 401, rate-limit → 429).
- **Order matters** — auth → rate-limit → logging → routing. Wrong order = real bugs.
- **`Gateway.dispatch(i)`** — the ~10-line engine; `next = () => dispatch(i+1)`. This IS mini-Express.
- **SRP** — one concern per middleware; that's what makes them composable.
- **OCP** — add a new concern by inserting a middleware; existing code untouched.
- **Strategy** — routing/load-balancing policy is pluggable (`RoutingStrategy`).
- **Unwind** — logging logs the final status *after* `await next()` returns.
- **Express parallel** — `app.use((req,res,next)=>{})` is a Chain-of-Responsibility handler.
- **Forgot `next()`?** — the classic stalled-request bug; the chain silently dead-ends.
- **HLD vs LLD** — [58](./58-api-gateway.md) is edge/aggregation/scale; this is the pipeline's classes.

---

## Connected topics

| Direction | Topic | Why |
|---|---|---|
| **Previous** | [47 — Chain of Responsibility](./47-pattern-chain-of-responsibility.md) | The pattern the whole gateway is built on — learn it first. |
| **Next** | [58 — API Gateway (HLD)](./58-api-gateway.md) | Zoom back out: edge auth, request aggregation, and scaling many gateway instances. |
| **Related** | [42 — Strategy](./42-pattern-strategy.md) | The pattern behind the pluggable `RoutingStrategy` (prefix match vs. round-robin). |
| **Related** | [70 — Rate Limiting](./70-rate-limiting.md) | The token-bucket idea reused inside `RateLimitMiddleware`. |
| **Related** | [111 — LLD Approach Framework](./111-lld-approach-framework.md) | The 7-step method (requirements → nouns → behaviours → design → diagram → code → patterns) we followed. |
