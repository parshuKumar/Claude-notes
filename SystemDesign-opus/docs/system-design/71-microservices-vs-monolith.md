# 71 — Microservices vs Monolith Architecture
## Category: HLD Components

---

## What is this?

A **monolith** is one application: one codebase, one deployable artifact, one database. You run `npm start` and the whole product is up. A **microservices** architecture splits that same product into many small, independently deployable applications that talk to each other over the network, each owning its own database.

Think of it as the difference between **one big restaurant kitchen** where every chef works side by side at the same counter, and **a food court** where each stall is its own business with its own kitchen, its own supplier, and its own opening hours. The food court can scale the busy pizza stall without touching the salad stall — but now the stalls have to *phone each other* to coordinate a combo meal, and phone calls drop.

---

## Why does it matter?

This is the single most over-hyped and most badly-advised decision in modern backend engineering. Beginners are told "microservices = modern, monolith = legacy." That is **wrong**, and believing it will cost your team a year.

**What breaks if you get this wrong:**
- **Split too early:** A 4-person team ends up operating 12 services, 12 CI pipelines, 12 dashboards, and a distributed debugging nightmare — to serve 500 users. Every feature now spans three repos and three deploys. Velocity collapses.
- **Split too late:** A 200-engineer team all committing to one repo, one deploy pipeline, one release train. Every deploy is a 6-hour war room. One team's memory leak takes down checkout.
- **Split wrong:** You get the **distributed monolith** — all the operational pain of microservices with none of the independence. This is the most common real-world outcome and we'll spend real time on it below.

**Interview angle:** Interviewers *love* to see you push back. If you're asked "would you use microservices here?" and you say "yes, obviously" with no discussion of team size, deployment coupling, or data consistency — you've failed a senior signal. The strong answer names the *specific pain* that justifies the split.

**Real-work angle:** You will inherit one of these two worlds and be asked to evolve it. Knowing the honest cost of each is how you avoid a rewrite that kills the company.

---

## The core idea — explained simply

### The Growing Restaurant Analogy

You open a restaurant. Day one, you're the only chef. One kitchen, one menu, one fridge.

**Stage 1 — the monolith (and it's great).** Everything is in arm's reach. Need a tomato for the pasta? Turn around, open the fridge, grab it. That is an **in-process function call** — it takes nanoseconds, it never fails, and if you drop the tomato you know instantly. When you change the pasta recipe, you change it once and it's live for everyone at dinner service. That's **one deploy**.

You hire 3 more chefs. Still fine. The kitchen is busy but everyone can shout across it.

**Stage 2 — the pain begins.** You now have 40 chefs in one kitchen. Problems appear that have *nothing* to do with cooking:

- Only one person can use the pass at a time, so plating is a queue. (**One deploy pipeline — everyone waits their turn to release.**)
- The dessert chef sets a pan on fire and the whole kitchen evacuates, including the people making salads. (**One bad module OOMs the process and kills every endpoint.**)
- Friday nights, the pizza oven is overwhelmed but the salad station is idle. You can't hire "half a kitchen" — you have to duplicate the *entire* kitchen. (**Can't scale one hot component independently.**)
- A chef wants to use a sous-vide machine, but the kitchen wiring only supports gas. (**Tech-stack lock-in — everything must be Node, or everything must be Java.**)

**Stage 3 — the food court (microservices).** You give the pizza team their own stall, their own oven, their own fridge, their own supplier. Now:

- Pizza can renovate their stall on a Tuesday without closing the salad stall. (**Independent deploy.**)
- Pizza can add three ovens on Friday. Salad adds none. (**Independent scaling.**)
- Pizza's fire only closes pizza. (**Fault isolation.**)
- Pizza can use an induction cooktop if they want. (**Polyglot.**)

But — and this is the part nobody tells you — **the tomato is now in someone else's fridge.** To make a pizza-and-salad combo you must *call the salad stall on the phone*. The phone can be busy. The phone can ring forever. The salad stall might have closed for maintenance. And your two fridges now disagree about how many tomatoes exist.

**Mapping the analogy:**

| In the restaurant | In your system |
|---|---|
| One kitchen, one fridge | Monolith: one process, one database |
| Grabbing a tomato from the fridge | In-process function call — fast, reliable, transactional |
| Phoning the salad stall | Network call — slow, can fail, can time out |
| One pass, everyone queues to plate | One deploy pipeline, one release train |
| Dessert fire evacuates everyone | One module's crash/leak kills the whole process |
| Duplicating the whole kitchen for pizza night | Scaling the whole monolith to scale one hot endpoint |
| Each stall has its own fridge | Database-per-service |
| Two fridges disagree on tomato count | Eventual consistency / data duplication |
| "Table 4 wants a combo, coordinate!" | Distributed transaction → SAGA |
| A stall that can't open unless 3 others are open | **Distributed monolith** (the disaster) |

The single sentence to remember: **microservices trade an in-process problem (a big codebase) for a distributed-systems problem (a network between your modules).** Distributed systems problems are strictly harder.

---

## Key concepts inside this topic

### 1. The monolith — what it actually is, and why it's the right default

A monolith is not "bad code." A monolith is a **deployment topology**: one artifact, one process (replicated N times behind a load balancer), one primary database.

```
                 ┌──────────────────────────────────────┐
   Clients ────▶ │            LOAD BALANCER             │
                 └────────┬──────────┬─────────┬────────┘
                          │          │         │
                 ┌────────▼───┐ ┌────▼─────┐ ┌─▼────────┐
                 │ APP :3000  │ │APP :3000 │ │APP :3000 │   same artifact
                 │ ┌────────┐ │ │          │ │          │   x3 replicas
                 │ │ users  │ │ │          │ │          │
                 │ │ orders │ │ │  (same)  │ │  (same)  │
                 │ │payments│ │ │          │ │          │
                 │ │ search │ │ │          │ │          │
                 │ └────────┘ │ │          │ │          │
                 └──────┬─────┘ └────┬─────┘ └────┬─────┘
                        └────────────┼────────────┘
                              ┌──────▼───────┐
                              │  PostgreSQL  │  ONE database
                              └──────────────┘
```

Note: a monolith **is horizontally scalable**. This is the biggest misconception. You can run 200 replicas of a monolith behind a load balancer. What you *cannot* do is scale `search` without also scaling `payments`.

**What the monolith genuinely gives you — and you should not give these up lightly:**

**a) One deploy.** Ship a feature that spans users + orders + payments? One commit, one CI run, one deploy. In microservices that's three PRs, three reviews, three deploys, and a *deploy ordering problem*.

**b) Real ACID transactions.** This is the giant one.

```javascript
// MONOLITH — the database guarantees this is all-or-nothing.
// If charging the card throws, the order row is rolled back. Automatically.
async function placeOrder(userId, items) {
  return db.transaction(async (tx) => {
    const order = await tx.orders.insert({ userId, status: 'PENDING' });
    await tx.inventory.decrement(items);       // reserve stock
    await tx.payments.charge(userId, total(items));
    await tx.orders.update(order.id, { status: 'CONFIRMED' });
    return order;                              // COMMIT — or nothing happened at all
  });
}
```

Split `inventory` and `payments` into separate services with separate databases and **this transaction no longer exists**. There is no `BEGIN` that spans two databases over HTTP. You now need a SAGA (see #6), which is 5x the code and introduces states like "payment succeeded but inventory reservation failed, now issue a refund."

**c) Easy local dev.** `git clone && npm install && npm run dev`. The whole product runs on your laptop. In microservices, running the full product locally means Docker Compose with 14 containers, 6 GB of RAM, and a `docker-compose.yml` that's out of date.

**d) Easy refactoring.** Moving a function from the `orders` module to the `billing` module is a rename. Your IDE does it. The compiler/linter catches every caller. Moving a function *between services* is an API design exercise, a versioning problem, a migration, and a coordinated deploy.

**e) No network between your modules.** An in-process call cannot time out, cannot be partially applied, cannot return a 503, and adds ~0.0001 ms. A network call adds 1–50 ms *and can fail in ways your code must handle*.

### 2. Where the monolith genuinely breaks down

Be honest — these are real, and at scale they are severe.

| Breaking point | What it feels like day-to-day |
|---|---|
| **Deploy coordination** | 60 engineers, one pipeline. Your PR sits behind a broken build from another team. Releases become weekly "trains" — you miss the train, you wait a week. Rollback reverts *everyone's* changes. |
| **Blast radius** | The reporting module runs an unbounded query, allocates 4 GB, and the Node process OOMs. Checkout goes down. Login goes down. Everything goes down — because it's **one process**. |
| **Independent scaling** | Image thumbnailing is CPU-bound and eats 90% of your CPU. To give it more CPU you must scale the *entire* app — including the 40 modules that were perfectly happy. You pay for idle capacity. |
| **Tech lock-in** | The ML team wants Python. The video team wants Go for its concurrency. Everyone is stuck in Node because it's one process. Also: upgrading the framework is an all-or-nothing, 6-month project. |
| **Cognitive load** | 800k lines. Nobody understands it. Every change has unknowable side effects because module boundaries were never enforced — anyone can `require()` anything. |
| **Build/test time** | The test suite takes 45 minutes. CI is a bottleneck for the whole company. |

Notice: **four of these six are about *people and teams*, not technology.** Hold that thought — it's Conway's Law, coming in #4.

### 3. Microservices — the real benefits and the hidden costs

```
                        ┌─────────────────┐
     Clients ─────────▶ │   API GATEWAY   │
                        └──┬────┬─────┬───┘
              ┌────────────┘    │     └───────────┐
      ┌───────▼──────┐  ┌───────▼──────┐  ┌───────▼──────┐
      │ ORDER SVC    │  │ PAYMENT SVC  │  │ INVENTORY SVC│
      │ (Node)       │  │ (Java)       │  │ (Go)         │
      │ x3 replicas  │  │ x2 replicas  │  │ x8 replicas  │  ← scaled independently
      └───────┬──────┘  └───────┬──────┘  └───────┬──────┘
              │                 │                 │
       ┌──────▼─────┐    ┌──────▼─────┐    ┌──────▼─────┐
       │ orders_db  │    │ payments_db│    │ inv_db     │  ← DB per service
       │ (Postgres) │    │ (Postgres) │    │ (DynamoDB) │
       └────────────┘    └────────────┘    └────────────┘
              │                 │                 │
              └────────┬────────┴─────────────────┘
                ┌──────▼────────┐
                │ Kafka / event │   ← async backbone
                │     bus       │
                └───────────────┘
```

**The real benefits (they are real):**

1. **Independent deploy.** The payments team ships 8 times a day. The search team ships once a month. Neither blocks the other. *This is the #1 reason to do microservices.*
2. **Independent scaling.** Inventory is read 100x more than payments — run 8 inventory pods and 2 payment pods.
3. **Team autonomy.** A team owns a service end to end: code, DB, on-call, SLO. No cross-team coordination for routine changes.
4. **Fault isolation.** Search dies → search results are empty, but checkout still works (*if* you built a fallback — see [73 — Circuit Breaker](./73-circuit-breaker.md)).
5. **Polyglot.** ML in Python, video in Go, API in Node.

**The costs nobody puts on the slide:**

| Hidden cost | Concretely, what you now have to build |
|---|---|
| **Distributed transactions die** | Every multi-service write becomes a **SAGA** with compensating actions. Your "place order" function goes from 8 lines to 300 lines plus a state machine. |
| **Every call can fail** | `orderService.getUser(id)` was a function call. It's now an HTTP call that can time out, return 503, be slow, or succeed twice. Every single call site needs timeouts, retries, circuit breakers, and idempotency keys. |
| **Debugging** | A user reports a 500. Which of the 9 services in the chain failed? You **must** have distributed tracing (correlation IDs propagated through every hop — see [80 — Monitoring & Observability](./80-monitoring-and-observability.md)) or you're blind. |
| **Testing** | You cannot integration-test against 14 live services. You need **contract tests** (Pact-style): "Order Service promises to call `/charge` with `{userId, amountCents}`; Payment Service promises to accept that shape." Without them, a Payment refactor silently breaks Orders in prod. |
| **Ops explosion** | 14 services × (CI pipeline + deploy config + dashboard + alerts + on-call runbook + secrets + TLS certs). Plus service discovery, plus a gateway, plus a mesh. You now need a platform team. |
| **Data duplication** | Order Service needs the user's email. It can't `JOIN users`. So it either calls User Service on every request (latency + coupling) or **caches a copy** of email locally — which goes stale. |
| **Eventual consistency everywhere** | User changes their email. Order Service still has the old one for 400 ms. Or 4 hours, if the event consumer is lagging. Product will ask "why does the receipt show the old email?" and the answer is "architecture." |

**Latency, concretely.** A single page load that used to be 5 in-process calls:

```
Monolith:      5 calls  × 0.001 ms  =   0.005 ms  of call overhead
Microservices: 5 calls  ×    12 ms  =      60 ms  of network overhead
               (and if 3 are chained sequentially, they ADD up)
```

And availability compounds. If each service is 99.9% available and a request touches 5 of them **synchronously in a chain**:

```
0.999^5 = 0.995  →  99.5% availability  →  ~3.6 hours of downtime per month
```

You made the system *less* reliable by splitting it. Fault isolation only helps if you actually degrade gracefully instead of chaining hard dependencies.

### 4. Conway's Law — the law that actually decides your architecture

> "Organizations which design systems are constrained to produce designs which are copies of the communication structures of these organizations." — Melvin Conway, 1967

Plain English: **your architecture will end up shaped like your org chart, whether you like it or not.**

If you have a frontend team, a backend team, and a DBA team, you will get a three-tier architecture — because that's where the communication lines are. If you have 8 product teams that each own a business capability, you'll naturally get 8 services.

The **Inverse Conway Manoeuvre** is the useful corollary: *if you want a certain architecture, first organize your teams that way.* Want independent services? First create teams that own a business capability end-to-end. If you draw microservice boundaries that cut across team boundaries, every change will require two teams to coordinate — and you've gained nothing.

**The practical rule:** a service should be owned by **exactly one team**. If two teams must both change a service to ship anything, that's not a service boundary, it's a shared library with extra steps.

This also gives you the honest answer to "when should we split?" — **when team coordination on the shared deploy pipeline becomes the bottleneck.** Not at a user count. Not at a line count. When your engineers spend more time waiting on each other than writing code.

### 5. Finding boundaries — and the #1 mistake

**The #1 mistake: splitting by technical layer.**

```
✗ WRONG — split by technical layer (this is a distributed monolith in waiting)

   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
   │ Controller   │──▶│ Business     │──▶│ Data Access  │
   │ Service      │   │ Logic Service│   │ Service      │
   └──────────────┘   └──────────────┘   └──────────────┘

   Add one field to "Order"?  →  You must change ALL THREE services
                                 and deploy them in the right order.
                                 You gained NOTHING and paid for everything.
```

```
✓ RIGHT — split by business capability (a vertical slice, top to bottom)

   ┌────────────────┐  ┌────────────────┐  ┌────────────────┐
   │ ORDERS         │  │ PAYMENTS       │  │ SHIPPING       │
   │  ├ API         │  │  ├ API         │  │  ├ API         │
   │  ├ logic       │  │  ├ logic       │  │  ├ logic       │
   │  └ its own DB  │  │  └ its own DB  │  │  └ its own DB  │
   └────────────────┘  └────────────────┘  └────────────────┘

   Add one field to "Order"?  →  Change ONE service. Deploy it. Done.
```

**The test for a good boundary:** *"Can I ship a typical feature for this capability by changing only this one service?"* If the answer is usually no, the boundary is wrong.

**Domain-Driven Design and bounded contexts.** DDD gives you the vocabulary. A **bounded context** is a region of your system inside which a word means exactly one thing. This sounds abstract until you see it:

> The word **"Customer"** means different things in different parts of the business.
> - To **Sales**, a Customer is a lead: name, company, deal size, pipeline stage.
> - To **Billing**, a Customer is a payer: tax ID, payment method, invoice address, credit limit.
> - To **Support**, a Customer is a ticket-raiser: contact email, plan tier, open issues.
>
> Trying to build ONE shared `Customer` object that satisfies all three gives you a 60-field god object that every team must agree on to change. That's the coupling you're trying to escape.

Each of those is a **bounded context** → each is a candidate service, each with its *own* narrow model of "Customer", linked only by a shared `customer_id`. **Duplication of the concept is the point, not a bug.**

**How to actually find them:**
1. **Event storming** — put the whole team in a room, write every business event on a sticky note (`OrderPlaced`, `PaymentCaptured`, `ItemShipped`), then cluster them. The clusters are your contexts.
2. **Follow the nouns of the business**, not the nouns of your codebase.
3. **Look for high cohesion, low coupling** — things that change together should live together.
4. **Start from the org chart** (Conway) — a boundary that doesn't map to a team will not survive.

### 6. The distributed monolith — the anti-pattern that eats teams

This is what you get when you split a monolith without fixing the coupling. It is the **worst of both worlds**, and it is by far the most common real-world microservices outcome.

You have a distributed monolith if **any** of these are true:

```
┌───────────────────────────────────────────────────────────────────┐
│  SYMPTOMS OF A DISTRIBUTED MONOLITH                               │
├───────────────────────────────────────────────────────────────────┤
│  ✗ Services must be deployed together, in a specific order        │
│  ✗ Multiple services read/write the SAME database tables          │
│  ✗ A single user request fans out through a long SYNCHRONOUS chain│
│  ✗ Changing one service's API forces changes in 4 others          │
│  ✗ You can't run or test a service alone — it needs the others up │
│  ✗ Services share a common library that contains business logic   │
└───────────────────────────────────────────────────────────────────┘
```

The synchronous chain is the killer:

```
Client ──▶ [A] ──▶ [B] ──▶ [C] ──▶ [D] ──▶ [E]
           each hop: +15ms latency, ×0.999 availability, +1 failure mode

  Latency:      5 × 15ms = 75ms of pure overhead
  Availability: 0.999^5  = 99.5%
  D goes slow   →  C's connection pool fills waiting on D
                →  B's pool fills waiting on C
                →  A's pool fills waiting on B
                →  ENTIRE SYSTEM DOWN because ONE leaf service got slow.
```

That's a **cascading failure**, and it's exactly what [73 — Circuit Breaker](./73-circuit-breaker.md) exists to stop.

You have all the costs — network, ops, tracing, no transactions — and **none** of the benefits: you can't deploy independently, can't scale independently, can't fail independently. You would have been strictly better off with a monolith.

**The root cause is almost always a shared database.** Which brings us to:

### 7. Database-per-service — what it forces on you

The rule: **a service's database is private. No other service touches it. Ever. Not even read-only.**

If Order Service can `SELECT * FROM users`, then the User team can never change the `users` schema — because they don't know who depends on it. You have re-created the monolith's coupling *with a network in the middle*. The shared database is the number one cause of distributed monoliths.

So `JOIN` across services is gone. Here's what replaces it:

```javascript
// ✗ IMPOSSIBLE now. There is no cross-database JOIN over HTTP.
//    SELECT o.*, u.email FROM orders o JOIN users u ON o.user_id = u.id;

// ── Option A: API composition (synchronous) ─────────────────────────
// Simple. But: extra latency, and Order Service is now DOWN if User Service is down.
async function getOrderWithUser(orderId) {
  const order = await orderRepo.findById(orderId);                  // local DB
  const user  = await userClient.getUser(order.userId);             // network hop!
  return { ...order, userEmail: user.email };
}

// ── Option B: local replica via events (asynchronous) ───────────────
// Order Service keeps its OWN slim copy of the fields it needs.
// It is now readable even if User Service is completely offline.
// The cost: the copy is EVENTUALLY consistent (stale for ~ms to ~s).
userEvents.on('UserEmailChanged', async (evt) => {
  await orderDb.userCache.upsert({
    userId: evt.userId,
    email: evt.newEmail,
    updatedAt: evt.occurredAt,   // keep this — you need it to resolve out-of-order events
  });
});

async function getOrderWithUser(orderId) {
  const order = await orderRepo.findById(orderId);
  const cached = await orderDb.userCache.find(order.userId);  // local read. No network. Fast.
  return { ...order, userEmail: cached.email };               // possibly a few seconds stale
}
```

Option B is what mature microservice systems do. Notice what you just accepted: **deliberate data duplication and eventual consistency**, in exchange for independence. That is the whole trade in one code block.

And multi-service writes become SAGAs (covered fully in [77 — Distributed Transactions](./77-distributed-transactions.md)):

```javascript
// A SAGA: a sequence of local transactions, each with a COMPENSATING action
// to undo it if a later step fails. There is no rollback — only "undo forward".
class PlaceOrderSaga {
  constructor(orders, payments, inventory) {
    this.steps = [
      { do: () => orders.create(this.ctx),      undo: () => orders.cancel(this.ctx.orderId) },
      { do: () => inventory.reserve(this.ctx),  undo: () => inventory.release(this.ctx.orderId) },
      { do: () => payments.charge(this.ctx),    undo: () => payments.refund(this.ctx.orderId) },
    ];
    Object.assign(this, { orders, payments, inventory });
  }

  async execute(ctx) {
    this.ctx = ctx;
    const completed = [];
    try {
      for (const step of this.steps) {
        await step.do();
        completed.push(step);
      }
      return { status: 'CONFIRMED' };
    } catch (err) {
      // Unwind in REVERSE order. Every compensation must be IDEMPOTENT,
      // because this unwind itself can crash and be retried.
      for (const step of completed.reverse()) {
        await step.undo();
      }
      return { status: 'FAILED', reason: err.message };
    }
  }
}
```

Compare that to the 8-line `db.transaction()` from #1. That difference — multiplied across every write path in your product — is the true price of microservices.

### 8. The modular monolith — the underrated middle ground

Here's the move almost nobody teaches beginners: **you can get most of the benefits of good boundaries without any of the network.**

A **modular monolith** is one deployable, one process, one database — but with *enforced internal module boundaries*. Each module has a public interface. Modules may only call each other through that interface. Each module owns its own tables and no other module may query them.

```
┌───────────────────────── ONE PROCESS, ONE DEPLOY ────────────────────────┐
│                                                                          │
│  ┌────────────────┐   ┌────────────────┐   ┌────────────────┐            │
│  │  ORDERS module │   │ PAYMENTS module│   │ SHIPPING module│            │
│  │                │   │                │   │                │            │
│  │  index.js  ◀───┼───┼── ONLY the     │   │                │            │
│  │  (public API)  │   │   public API    │   │                │            │
│  │  ─ ─ ─ ─ ─ ─ ─ │   │  ─ ─ ─ ─ ─ ─ ─ │   │  ─ ─ ─ ─ ─ ─ ─ │            │
│  │  internal/     │   │  internal/     │   │  internal/     │            │
│  │  orders tables │   │  payment tables│   │  ship tables   │            │
│  └────────────────┘   └────────────────┘   └────────────────┘            │
│         ▲ tables are PRIVATE to their module — enforced by review/lint    │
└──────────────────────────────────────────────────────────────────────────┘
                              ┌──────────────┐
                              │  PostgreSQL  │  ← one DB, but schema-per-module
                              └──────────────┘
```

```javascript
// modules/orders/index.js — the ONLY thing other modules may import.
// This is the "public API" of the module. Enforce it with ESLint
// (eslint-plugin-boundaries) or dependency-cruiser so a bad import fails CI.
export { placeOrder, getOrder, cancelOrder } from './internal/order-service.js';
// Everything under ./internal/ is off-limits to the rest of the app.

// modules/shipping/internal/shipping-service.js
import { getOrder } from '../../orders/index.js';   // ✓ allowed: public interface
// import { orderRepo } from '../../orders/internal/order-repo.js';  ✗ CI FAILS
```

**Why this is so good:**
- You still have ACID transactions across modules (same DB).
- You still have one deploy, one local dev setup, easy refactoring.
- But you've *practised* the boundaries. If a module later needs to become a service, the seam is already cut — you replace the in-process import with an HTTP client and you're done.
- If you got the boundary **wrong**, fixing it is a refactor, not a distributed migration. **This is the killer feature.** Boundaries are hard to get right the first time; a modular monolith lets you be wrong cheaply.

**Shopify** is the flagship example: a very large, very successful Ruby monolith, deliberately kept as a monolith and instead made rigorously modular. Their public position has been that componentization inside the monolith gave them the boundaries they needed without distributed-systems tax.

### 9. Migration: the strangler fig pattern

Never do a big-bang rewrite. The **strangler fig** (named after the vine that grows around a tree and gradually replaces it) is the safe path:

```
STEP 1: put a proxy/gateway in front of the monolith. Nothing else changes.
     Client ──▶ [ PROXY ] ──▶ [ MONOLITH ]

STEP 2: build ONE new service for ONE capability. Route just that path to it.
     Client ──▶ [ PROXY ] ──┬─ /payments/* ─▶ [ PAYMENT SVC ] (new)
                            └─ everything else ─▶ [ MONOLITH ]

STEP 3: repeat, one capability at a time. Each step is independently
        deployable, independently revertible (just flip the route back).

STEP 4: the monolith shrinks until it's gone — or, very often, until it's
        small enough that you happily keep it forever.
```

Rules: extract **the highest-value, most-independent capability first** (usually the one with a clear boundary and its own scaling need). Keep the route flippable so you can roll back instantly. Never extract two things at once. Full treatment in [139 — Evolutionary Architecture](./139-evolutionary-architecture.md).

### 10. Sync vs async between services

Once you're distributed, *how* services talk determines whether you're resilient or a distributed monolith.

| | **Synchronous** (REST/gRPC) | **Asynchronous** (events/queue) |
|---|---|---|
| **Caller** | Blocks, waits for a reply | Fires and forgets |
| **Coupling** | Temporal — callee must be **up right now** | Decoupled — callee can be down; queue holds the message |
| **Failure** | Caller fails too (unless circuit-broken) | Message waits; consumer catches up |
| **Consistency** | Immediate | Eventual |
| **Debuggability** | Easy — one stack trace | Hard — flow is spread across time |
| **Use when** | You need the answer to respond (e.g. "is this card valid?") | You need to *notify* others (e.g. "order was placed") |

**The rule of thumb: use synchronous calls for queries you need answered right now, and asynchronous events for everything else.** If Order Service must synchronously call Email, Analytics, Loyalty, and Warehouse, you've built a 5-hop chain that dies when any leaf dies. Instead, Order Service writes to its DB and publishes `OrderPlaced`. The other four subscribe. Order Service doesn't even know they exist — and stays up if all four are down.

```javascript
// ✗ SYNCHRONOUS FAN-OUT — Order is now coupled to 4 services' uptime.
async function placeOrder(dto) {
  const order = await orderRepo.save(dto);
  await emailService.send(order);      // if this is down...
  await analyticsService.track(order); // ...order placement FAILS
  await loyaltyService.addPoints(order);
  await warehouseService.notify(order);
  return order;
}

// ✓ ASYNCHRONOUS EVENT — Order commits, publishes, returns. Done in 20ms.
//   The other four consume at their own pace. If loyalty is down for an hour,
//   the message waits in the queue and is processed when it returns.
async function placeOrder(dto) {
  const order = await orderRepo.save(dto);
  await eventBus.publish('OrderPlaced', {
    orderId: order.id, userId: order.userId, totalCents: order.totalCents,
  });
  return order;
}
```

(Production note: saving to the DB and publishing to the bus are *two* systems — the process can crash between them. The fix is the **transactional outbox**: write the event into an `outbox` table *inside the same DB transaction* as the order, and have a separate relay poll that table and publish. See [67 — Message Queues](./67-message-queues.md).)

---

## Visual / Diagram description

### Diagram 1: The three architectures, side by side

```
 ┌──────────────────────┐   ┌──────────────────────┐   ┌──────────────────────┐
 │   1. MONOLITH        │   │ 2. MODULAR MONOLITH  │   │  3. MICROSERVICES    │
 ├──────────────────────┤   ├──────────────────────┤   ├──────────────────────┤
 │ ┌──────────────────┐ │   │ ┌────┐┌────┐┌──────┐ │   │ ┌────┐ ┌────┐ ┌────┐ │
 │ │ users  orders    │ │   │ │ORD ││PAY ││ SHIP │ │   │ │ORD │ │PAY │ │SHIP│ │
 │ │ payments shipping│ │   │ │  ▲ ││  ▲ ││   ▲  │ │   │ │ ▲  │ │ ▲  │ │ ▲  │ │
 │ │  (all tangled)   │ │   │ └──┼─┘└──┼─┘└───┼──┘ │   │ └─┼──┘ └─┼──┘ └─┼──┘ │
 │ └──────────────────┘ │   │  public interfaces   │   │  HTTP/gRPC/events    │
 │   ONE PROCESS        │   │   ONE PROCESS        │   │  3 PROCESSES         │
 │        │             │   │        │             │   │   │      │      │    │
 │  ┌─────▼─────┐       │   │  ┌─────▼─────┐       │   │ ┌─▼─┐  ┌─▼─┐  ┌─▼─┐  │
 │  │  ONE DB   │       │   │  │ONE DB,    │       │   │ │DB │  │DB │  │DB │  │
 │  │ (shared   │       │   │  │schema per │       │   │ └───┘  └───┘  └───┘  │
 │  │  tables)  │       │   │  │ module    │       │   │                      │
 │  └───────────┘       │   │  └───────────┘       │   │                      │
 ├──────────────────────┤   ├──────────────────────┤   ├──────────────────────┤
 │ ✓ ACID txns          │   │ ✓ ACID txns          │   │ ✗ SAGAs only         │
 │ ✓ 1 deploy           │   │ ✓ 1 deploy           │   │ ✓ N deploys          │
 │ ✗ no boundaries      │   │ ✓ real boundaries    │   │ ✓ hard boundaries    │
 │ ✗ scale all-or-none  │   │ ✗ scale all-or-none  │   │ ✓ scale per service  │
 │ ✗ 1 crash = all down │   │ ✗ 1 crash = all down │   │ ✓ fault isolation    │
 │ Team size: 1–20      │   │ Team size: 10–60     │   │ Team size: 50+       │
 └──────────────────────┘   └──────────────────────┘   └──────────────────────┘
      START HERE                MOVE HERE NEXT            ONLY WHEN IT HURTS
```

Read it left to right — that is the recommended *evolution path*, and most companies should stop at box 2.

### Diagram 2: The distributed monolith — how to spot it on a whiteboard

```
      ┌──────────────────────── THE TRAP ─────────────────────────┐
      │                                                           │
 Client ──▶ [ORDER] ──sync──▶ [USER] ──sync──▶ [PRICING] ──sync──▶ [TAX]
                │                                                    │
                └────────────────┐              ┌────────────────────┘
                                 ▼              ▼
                          ┌────────────────────────┐
                          │   ONE SHARED DATABASE   │   ◀── the red flag
                          │  (all 4 read & write    │
                          │   the same tables)      │
                          └────────────────────────┘

  What you have:                        What you wanted:
   ✗ must deploy all 4 together          ✓ deploy any one alone
   ✗ TAX slow  → ORDER dies              ✓ TAX slow  → degrade gracefully
   ✗ schema change → 4 teams meet        ✓ schema change → 1 team
   ✗ 4 CI pipelines, 4 dashboards        (you kept the cost, lost the benefit)
   ✓ ...nothing
```

If you can draw a shared database touched by more than one service, or a synchronous chain more than two hops deep, you are looking at a distributed monolith. Say so in the interview — it's a strong signal.

---

## Real world examples

### 1. Amazon — the canonical split (and the rule that made it work)

Around 2002, Amazon's `Obidos` monolith had become a deployment bottleneck for a fast-growing engineering org. Bezos's now-famous internal mandate required all teams to expose their data and functionality **only** through service interfaces over the network, with **no direct database access between teams** — and to design those interfaces as if they'd be externalized to third parties.

The critical detail beginners miss: the mandate was as much **organizational** as technical. Pair it with the "two-pizza team" idea (a team small enough to be fed by two pizzas) and you get Conway's Law applied deliberately: small autonomous teams → small autonomous services. The no-shared-database rule is precisely what prevented a distributed monolith. That externalization discipline is also what made AWS possible.

### 2. Netflix — microservices plus the resilience tooling that makes them survivable

Netflix moved from a monolithic DVD-rental datacenter application to a large cloud microservices estate on AWS, prompted in part by a 2008 database corruption incident that halted shipping for days.

But the *lesson* is what they had to build alongside it: **Hystrix** (circuit breakers), **Ribbon** (client-side load balancing), **Eureka** (service discovery), **Zuul** (gateway), and **Chaos Monkey** (deliberately killing instances in production to prove the system tolerates it). None of that is needed in a monolith. That entire toolbox is the *tax* microservices imposed — and Netflix open-sourced it precisely because everyone who follows them has to pay it too.

### 3. Shopify — the deliberate modular monolith

Shopify runs one of the largest Rails codebases in the world and, rather than shattering it into microservices, invested in **modularization inside the monolith** — carving the codebase into components with enforced boundaries and explicit public interfaces, with tooling that fails the build on illegal cross-component calls.

This is the strongest public counter-example to "you must go microservices to scale." They scale to enormous Black Friday traffic on a modular monolith (with some services extracted where genuinely warranted). The takeaway: **the boundaries are the valuable thing — the network is just one (expensive) way to enforce them.**

---

## Trade-offs

| Dimension | Monolith | Microservices |
|---|---|---|
| **Deploy** | One pipeline; simple but a shared bottleneck | Independent per service; complex but unblocks teams |
| **Transactions** | Real ACID, free | SAGAs + compensations; you write the rollback logic |
| **Consistency** | Strong by default | Eventual by default |
| **Scaling** | All-or-nothing; pay for idle capacity | Per-service; efficient |
| **Fault isolation** | None — one crash kills everything | Good — *if* you add circuit breakers + fallbacks |
| **Latency** | ~0 ms between modules | 1–50 ms per hop, and hops compound |
| **Availability math** | Single 99.9% component | 0.999^N for an N-hop sync chain — *worse*, not better |
| **Local dev** | `npm run dev` | Docker Compose with 14 containers |
| **Debugging** | One stack trace | Distributed tracing required |
| **Testing** | Integration tests just work | Contract tests mandatory |
| **Refactoring boundaries** | Cheap — an IDE rename | Expensive — API versioning + data migration |
| **Tech choice** | Locked to one stack | Polyglot |
| **Ops headcount** | Low | High — you likely need a platform team |
| **Best team size** | 1–20 engineers | 50+ engineers in autonomous teams |

**Rule of thumb:** **Start with a modular monolith. Always.** Split out a service only when you can name the *specific pain* it relieves — "the payments team is blocked by the release train," or "thumbnailing needs 8x the CPU of everything else," or "the ML model must run in Python." If you cannot name the pain in one sentence, you are buying distributed-systems complexity for nothing. And if you do split: **one team per service, one database per service, async by default.** Break any of those three and you've built a distributed monolith.

---

## Common interview questions on this topic

### Q1: "Would you build this system as microservices or a monolith?"
**Hint:** Never answer without asking about **team size and deploy cadence** first. The strong answer: "For a team of this size I'd start with a modular monolith — one deploy, real transactions, cheap to refactor while the domain is still uncertain. I'd enforce module boundaries with private schemas and public interfaces so that when a specific capability needs independent deploy or scaling, the seam is already cut. I'd extract via strangler fig, starting with [X], because [concrete pain]." Naming the *pain* is the senior signal.

### Q2: "What is a distributed monolith and how do you avoid it?"
**Hint:** Services that must be deployed together, share a database, and call each other in synchronous chains — all the costs of distribution, none of the benefits. Avoid it with three rules: (1) database-per-service, no exceptions — a shared DB is the #1 cause; (2) async events for notification, sync only for queries you need answered right now; (3) split by business capability, never by technical layer. Test: "can I deploy this service alone, at 3am, without telling anyone?" If no, it isn't a service.

### Q3: "You split into microservices. How do you handle a transaction that spans three of them?"
**Hint:** You can't — there's no distributed `BEGIN`. Use a **SAGA**: a sequence of local transactions, each with a compensating action, unwound in reverse on failure. Choreography (each service reacts to events — simple, but the flow is invisible) vs orchestration (a coordinator drives the steps — explicit and debuggable, but a central component). Compensations must be **idempotent** because the unwind can itself be retried. Then say the honest thing: "and the fact that this is now 300 lines instead of an 8-line DB transaction is a real argument for keeping these three in one service."

### Q4: "Explain Conway's Law and why it matters for service boundaries."
**Hint:** Your architecture inevitably mirrors your org's communication structure. Consequence: a service boundary that cuts across team boundaries will force constant cross-team coordination — you paid for microservices and got a monolith with latency. Corollary (the Inverse Conway Manoeuvre): design the teams first, and the architecture follows. Practical rule: **one team owns one service, end to end** — code, DB, on-call.

### Q5: "Microservices give you fault isolation. Do they make the system more available?"
**Hint:** This is a trap — **not automatically, and often the opposite.** If a request synchronously chains through 5 services at 99.9% each, availability is 0.999^5 ≈ 99.5% — *worse* than the monolith. You only get the isolation benefit if you actually build for it: timeouts, circuit breakers, bulkheads, and **fallbacks** (search dies → return empty results, not a 500). Fault isolation is something you *engineer*, not something the topology hands you.

---

## Practice exercise

### The Boundary Audit

Take this (deliberately messy) e-commerce monolith and design its evolution. Budget ~30 minutes.

**The system as it exists:**
- Node/Express monolith, one Postgres DB, 120k lines.
- 14 engineers in 3 teams: *Storefront* (browse, search, cart), *Fulfilment* (orders, inventory, shipping), *Money* (payments, refunds, invoicing).
- Pain: search re-indexing is CPU-heavy and slows the whole app during peak; the Money team is PCI-audited and needs to ship on its own schedule but is stuck behind the shared weekly release train; a bad reporting query OOM'd the process last month and took down checkout for 20 minutes.

**Produce:**

1. **Draw the modular monolith.** List the modules, and for each one: the tables it owns privately, and its 3–5 public interface functions. (Hint: use the three teams as your starting seams — Conway.)
2. **Pick exactly ONE service to extract first.** Justify it in one sentence naming the *specific pain* it relieves. Then draw the strangler-fig routing for it (before / during / after).
3. **Find the cross-boundary write.** "Place order" touches orders, inventory, and payments. Once your extracted service is out, write the SAGA: list each step and its compensating action. What happens if the compensation itself fails?
4. **Find the cross-boundary read.** Fulfilment needs the customer's email for the shipping notification, but that lives in Storefront's module. Write down both options (sync API call vs. event-driven local replica), and pick one — say what you're giving up.
5. **Anti-pattern check.** Write down the one design decision that would turn your answer into a distributed monolith. (If you can't find one, you haven't looked hard enough — look at the database.)

Deliverable: one page of boxes-and-arrows plus five short written answers. Compare your first-extraction choice to what the pain statement actually justifies — most people pick the *interesting* service, not the *painful* one.

---

## Quick reference cheat sheet

- **Monolith = deployment topology, not bad code.** One artifact, one process, one DB. It *is* horizontally scalable.
- **The monolith is the right default.** One deploy, real ACID transactions, trivial local dev, cheap refactoring, zero network between modules.
- **Split when team coordination is the bottleneck** — not at a user count, not at a line count. Name the pain or don't split.
- **Microservices' real win is independent deploy.** Everything else (scaling, polyglot, isolation) is secondary and partly achievable other ways.
- **The hidden costs:** SAGAs replace transactions, every call can fail, distributed tracing becomes mandatory, contract tests become mandatory, data gets duplicated, everything is eventually consistent.
- **Conway's Law:** your architecture will mirror your org chart. One team per service, or you gained nothing.
- **Split by business capability, never by technical layer.** A layer-split service means every feature touches every service.
- **Bounded context:** a region where a word means one thing. "Customer" means three different things to Sales, Billing, and Support — that's three services, and duplicating the concept is the *point*.
- **Distributed monolith = deploy-together + shared DB + synchronous chains.** All the cost, none of the benefit. The most common failure mode.
- **Database-per-service is non-negotiable.** A shared DB is the #1 cause of distributed monoliths. No cross-service `JOIN` — use API composition or an event-fed local replica.
- **Availability compounds badly:** 0.999^5 ≈ 99.5%. A sync chain of 5 services is *less* available than a monolith unless you add circuit breakers and fallbacks.
- **Async by default, sync only when you need the answer now.** Publish `OrderPlaced`; don't call four services in a row.
- **Modular monolith is the underrated answer:** real boundaries, zero network. Get the boundaries wrong cheaply, before they cost you a migration.
- **Migrate with the strangler fig,** never a big-bang rewrite. One capability at a time, behind a proxy, always revertible.

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [70 — Rate Limiting](./70-rate-limiting.md) — protecting a service's boundary once you have one |
| **Next** | [72 — Service Discovery](./72-service-discovery.md) — once you have services, how do they find each other? |
| **Related** | [77 — Distributed Transactions — 2PC and SAGA](./77-distributed-transactions.md) — the price you pay for database-per-service |
| **Related** | [73 — Circuit Breaker Pattern](./73-circuit-breaker.md) — stopping the cascading failure that sync chains create |
| **Related** | [139 — Evolutionary Architecture](./139-evolutionary-architecture.md) — the strangler fig migration in full |
