# 139 — Evolutionary Architecture — Designing for Change
## Category: Advanced

---

## What is this?

**Evolutionary architecture** is the practice of designing systems so that *change is cheap*, because change is the one thing you can guarantee will happen. Requirements shift, traffic grows 100x, teams reorganize, and the "obviously correct" datastore you picked in year one becomes the bottleneck in year three.

Think of it like a **city, not a building**. A building is designed once, to a fixed blueprint, and then it's finished. A city is never finished — roads get widened, old districts get demolished and rebuilt, new neighborhoods sprawl outward, and all of it happens *while people are still living there*. Nobody bulldozes the whole city to build version 2. Good architecture works the same way: it's a stream of small, reversible decisions, not a single perfect blueprint carved in stone on day one.

The senior insight is this: **no design survives contact with reality unchanged.** The goal is not the perfect architecture. The goal is the architecture that can *evolve* cheaply.

---

## Why does it matter?

If you don't internalize this, you fail in one of two expensive directions:

- **You over-engineer.** You try to predict the future and build for it. You add a plugin system, a generic rules engine, and support for five databases you don't use — all for requirements that never arrive. This is speculative over-engineering (recall [19 — YAGNI & Speculative Generality](./24-coupling-and-cohesion.md) territory: "You Aren't Gonna Need It"). You paid for flexibility you never cashed in, and that flexibility made the code harder to read and change *today*.
- **You under-engineer.** You wire everything directly to everything else. When the one change you actually needed arrives, it touches 40 files, and each touch risks a new bug. This is the "big ball of mud" — a system so rigid that changing it is more expensive than living with it broken.

**Interview angle:** Senior and staff interviews almost always include *"How would this design evolve if X changed?"* — traffic 10x'd, a new feature type appeared, you had to swap the database. A strong answer doesn't redesign from scratch; it points at the **seams** that make the change cheap and names what was deliberately kept flexible versus deliberately deferred.

**Real-work angle:** Every legacy system you'll ever inherit is a monument to somebody's failure to design for change. The skill of *safely evolving* a running system — without a doomed big-bang rewrite — is one of the highest-leverage things a senior engineer does.

---

## The core idea — explained simply

### The Renovating-While-Occupied Analogy

Imagine you own a house that your family lives in every single day. Over ten years you want to: modernize the kitchen, add a second floor, replace the plumbing, and knock down a wall. You have one hard constraint — **the family never moves out.** The house must be livable every night.

You would never demolish the whole house and rebuild. You'd do it room by room. You put up a temporary wall (a **facade**), gut one room behind it, finish it, move the family's activity into the new room, then tear into the next. Some choices are easy to undo — paint color, furniture. Some are terrifying to undo — moving the load-bearing wall, re-routing the main sewer line. You agonize over the load-bearing wall and you don't lose a minute of sleep over the paint.

That's evolutionary architecture in one picture.

| Analogy | Technical concept |
|---------|------------------|
| The house | Your production system |
| The family living there | Live users / traffic that can't stop |
| Renovating room by room | Incremental migration (Strangler Fig) |
| The temporary wall | A facade / proxy / API gateway |
| Paint color, furniture | Easily reversible decisions (two-way doors) |
| The load-bearing wall, the sewer line | Hard-to-reverse decisions (one-way doors) |
| A building inspector's checklist | Fitness functions (automated architecture tests) |
| A shortcut you take now, fix later | Deliberate technical debt |
| Rot you didn't notice | Accidental technical debt |

The whole discipline reduces to two habits: **spend your caution on the decisions that are hard to undo**, and **keep the seams between parts clean so the next change is a room, not the whole house.**

---

## Key concepts inside this topic

### 1. You cannot predict the future — so stop designing the final system

The instinct of a junior engineer handed a design problem is to design *the end state* — the system as it'll look when the company is huge. This is almost always wrong, because:

- You don't know which features will succeed. Most won't.
- You don't know where the real load will land until real users arrive.
- Every piece of speculative flexibility is code you must carry, understand, and debug — *forever* — whether or not you ever use it.

The evolutionary stance: **build the simplest thing that meets today's real needs AND is cheap to change tomorrow.** Those are two requirements, not one. "Simplest thing that works" alone can produce a rigid mess. "Cheap to change" alone can produce an over-engineered abstraction cathedral. You need both.

Architecture is a **stream of decisions made over time**, not a one-time blueprint. You will make the important calls with *more* information later, when the future has partly arrived. Deferring a decision until you actually need to make it is not laziness — it's buying an option with better data.

```javascript
// OVER-ENGINEERED (day 1): a generic, pluggable notification "engine"
// for a product that today only sends one email.
class NotificationEngine {
  constructor(channelRegistry, templateResolver, rulePipeline, retryPolicy) { /* ... */ }
  registerChannel(name, adapter) { /* SMS, push, webhook, carrier pigeon... */ }
  // 400 lines of flexibility nobody asked for.
}

// RIGHT-SIZED (day 1): do the one thing, behind a clean seam.
async function sendWelcomeEmail(user) {
  await emailClient.send({ to: user.email, template: "welcome", data: { name: user.name } });
}
// It's ONE function behind ONE name. When SMS actually becomes a requirement,
// you introduce a `Notifier` interface THEN — cheaply, because there's one call site.
```

The second version looks less impressive. It is the senior choice.

### 2. The reversibility principle — one-way vs two-way doors

Jeff Bezos framed decisions as **doors**:

- A **two-way door** is easy to walk back through. If you make the wrong call, you notice, you reverse it, and the cost is small. Most decisions are two-way doors.
- A **one-way door** slams shut behind you. Reversing it is very expensive or impossible.

The skill is *telling them apart* and spending your limited "caution budget" almost entirely on the one-way doors.

| Decision | Door | Why |
|----------|------|-----|
| Your core data model / schema shape | One-way | Data outlives code; migrating years of production data is brutal |
| Your primary datastore (SQL vs a specific NoSQL) | One-way | Every query, index, and transaction assumption gets baked in |
| A **public** API contract | One-way | External clients you don't control depend on it forever |
| Splitting a monolith into microservices | One-way | Network boundaries and distributed transactions are hard to un-draw |
| Which logging library you use | Two-way | Wrapped behind one interface; swap in an afternoon |
| An internal function's signature | Two-way | You own every caller; change them all in one commit |
| A caching layer's eviction policy | Two-way | Tunable config, no external contract |

The failure mode juniors have is *inverted caution*: they agonize for a week over a two-way door (which linting config? which folder structure?) and blow through a one-way door in an afternoon ("eh, let's just make the user ID a random string, we'll never need to change it"). Reverse that. On two-way doors: decide fast, move on, fix later if wrong. On one-way doors: slow down, write the doc, get a second opinion.

### 3. Design for change — the enabling practices

Evolvability isn't a wish; it's the *result* of specific engineering practices you've already met:

**Loose coupling + high cohesion (recall [24 — Coupling & Cohesion](./24-coupling-and-cohesion.md)).** This is the foundation. If module A knows nothing about module B's internals, you can rewrite B without touching A. **Program to interfaces, not implementations** (recall the Dependency Inversion Principle, 18) so an implementation can be swapped without its consumers noticing.

```javascript
// Consumers depend on the SHAPE (interface), never the concrete class.
class PaymentGateway {                 // the "interface" (base class = contract)
  async charge(amountCents, token) { throw new Error("not implemented"); }
}
class StripeGateway extends PaymentGateway {
  async charge(amountCents, token) { /* Stripe SDK */ }
}
class AdyenGateway extends PaymentGateway {   // added YEARS later
  async charge(amountCents, token) { /* Adyen SDK */ }
}
// Checkout code took a PaymentGateway. Swapping Stripe -> Adyen touches ONE
// wiring line, not the 30 places that call charge(). That's a clean seam.
```

**Well-defined contracts / APIs between components (recall 69, and contract testing 138).** A contract is a promise about inputs and outputs. As long as a component keeps its promise, it's free to change *everything* inside. And when a contract itself must evolve, you **version** it (`/v1/orders`, `/v2/orders`) so existing clients keep working while new ones opt in — you evolve the contract without breaking anyone.

**Modularity — a modular monolith beats premature microservices (recall [71 — Microservices vs Monolith](./71-microservices-vs-monolith.md)).** Clear module boundaries *inside* one deployable give you most of the evolvability of microservices with almost none of the distributed-systems pain. And they leave the door open: if one module truly needs independent scaling later, its clean boundary is exactly the seam you extract a service along. Draw the boundaries first; split the deployment only when reality forces you to.

**Event-driven decoupling (recall [68 — Event-Driven Architecture](./68-event-driven-architecture.md)).** When a producer emits an event ("OrderPlaced") instead of calling consumers directly, adding a *new* consumer (say, a fraud-check service) requires **zero changes to the producer**. The producer doesn't even know the new consumer exists. That is evolvability as a structural property.

### 4. Fitness functions — making "good architecture" continuously verifiable

Here's the problem with a one-time architecture review: architecture **rots**. The reviewer blesses the design, everyone goes home, and six months of deadline-driven commits quietly punch holes in every rule. By the time anyone notices, the "layered architecture" is a colander.

A **fitness function** (the central idea from the book *Building Evolutionary Architecture*) is an **automated, objective test that guards an architectural quality you care about.** It runs in CI, and it *fails the build* when the architecture drifts from what you decided. It turns "good architecture" from a one-time opinion into a continuously enforced invariant — the same way unit tests turned "correct behavior" from hope into a check.

Concrete examples you can actually write:

```javascript
// FITNESS FUNCTION 1 — forbidden dependency direction.
// Rule: the `domain` layer must NEVER import from the `web` layer.
// (Recall dependency inversion: business logic shouldn't know about HTTP.)
const { execSync } = require("child_process");
test("domain layer does not depend on web layer", () => {
  const hits = execSync(
    `grep -rl "require(.*/web/" src/domain || true`
  ).toString().trim();
  expect(hits).toBe("");   // any hit fails the build
});
```

```javascript
// FITNESS FUNCTION 2 — latency budget as a build gate.
// Rule: p95 of the checkout path must stay under 200ms.
test("checkout p95 latency stays within budget", async () => {
  const samples = [];
  for (let i = 0; i < 200; i++) {
    const t0 = performance.now();
    await runCheckoutSmokeTest();
    samples.push(performance.now() - t0);
  }
  samples.sort((a, b) => a - b);
  const p95 = samples[Math.floor(samples.length * 0.95)];
  expect(p95).toBeLessThan(200);   // a perf regression breaks the PR, not prod
});
```

Other fitness functions people run in the real world:

- **Layering rule:** UI never imports from the persistence layer directly.
- **Coupling ceiling:** no module may have more than N inbound dependencies (fails if coupling creeps up).
- **No cyclic dependencies** between modules (a tool like `dependency-cruiser` or `madge` in the Node world).
- **Bundle size budget:** the frontend build fails if the JS bundle grows past X KB.
- **Security:** no dependency with a known critical CVE (`npm audit` as a gate).

The magic isn't any single check. It's that these run on **every commit**, so drift is caught in the hour it's introduced, not in a horrified review two years later.

### 5. The Strangler Fig pattern — evolving a legacy system without a rewrite

This is the most practically valuable idea in the whole topic, so slow down here.

**The name comes from the strangler fig tree.** In a rainforest, a strangler fig seed germinates in the canopy and sends roots *down around* an existing host tree. Over years the fig grows a complete lattice around the host. Eventually the fig is fully self-supporting — and the original host tree, now enclosed, dies and rots away, leaving a hollow, fully-formed fig standing exactly where the old tree was. The new structure grew *around* the old one and gradually took over its job, with no moment where nothing was holding up the canopy.

Applied to software, coined by Martin Fowler, the **Strangler Fig pattern** replaces a legacy system incrementally:

1. **Put a facade in front of the old system.** A proxy / API gateway that every request now flows through. On day one it routes 100% of traffic straight to the legacy system. Nothing has changed for users — you've just inserted a seam you control.
2. **Build new functionality (or reimplement one old piece) behind that facade** as a new service.
3. **Route that one slice of traffic to the new service.** Everything else still hits legacy. You've strangled one branch.
4. **Repeat, slice by slice.** Each iteration is small, independently shippable, and independently reversible — if the new slice misbehaves, flip the route back to legacy in seconds.
5. **When the last slice is migrated, the legacy system is fully "strangled."** No traffic reaches it. You delete it. The fig stands where the tree was.

**Contrast: the big-bang rewrite** — freeze the old system, build a shiny replacement from scratch, flip everyone over on launch day. It is one of the most reliable ways to destroy a project, and here's *why rewrites fail*:

- **You lose years of embedded edge-case knowledge.** The old code is ugly because reality is ugly. Every weird `if` is a bug someone hit, a tax rule, a customer who does something insane. A rewrite throws all of that hard-won knowledge away and rediscovers each edge case *as a production incident.*
- **You ship nothing for months (or years).** During the rewrite the business gets zero new features while competitors ship. Pressure mounts, corners get cut, and the rewrite becomes the very mess it was meant to replace.
- **The old system keeps changing underneath you.** It's still in production, still getting urgent fixes and features. Your rewrite is aiming at a target that moves every week, so you're never actually done.

The Strangler Fig sidesteps all three: knowledge is carried over piece by piece with the old system right there to compare against, value ships every iteration, and you migrate *current* behavior because the old system isn't frozen — it's being retired gradually.

### 6. The two failure modes and the balance between them

| Failure mode | What it looks like | Root cause |
|--------------|-------------------|------------|
| **Over-engineering** | Plugin systems, generic frameworks, config for cases that never happen | Trying to predict the future (violates YAGNI, recall 19) |
| **Under-engineering** | Big ball of mud, everything wired to everything, no seams | Never thinking past today's ticket |

The balance is a single sentence: **build for today, but keep the seams clean so tomorrow is cheap.** You do *not* build the notification engine before you send two kinds of notification. But you *do* put the one email send behind a named function, so introducing that engine later has one obvious place to live. You defer the flexibility; you don't defer the *cleanliness of the boundary.* Clean seams are cheap now and priceless later; speculative flexibility is expensive now and usually wasted.

### 7. Technical debt — a real, manageable thing

**Technical debt** is the future cost you incur by choosing a faster-but-worse solution now. The metaphor is a financial loan, and like a loan it can be *smart* or *ruinous* depending on how you manage it.

- **Deliberate debt** is a conscious, tracked shortcut taken to ship on time. *"We'll hardcode the tax rate for launch and pull it into a config service in Q3 — ticket JIRA-412 filed."* This is a sensible loan. You knew the interest, you wrote it down, you have a repayment date. Deliberate, tracked debt is a legitimate engineering tool.
- **Accidental debt** comes from neglect and ignorance: copy-paste that drifts, a dependency three major versions behind, a module everyone's afraid to touch. Nobody chose it; it accreted. This is the debt that compounds.

Why unmanaged debt is dangerous: **it compounds.** Each shortcut makes the *next* change slightly harder, which makes the next shortcut more tempting, until one day a one-line feature takes three weeks and everyone shrugs "that's just how this codebase is." The system has become un-evolvable — the exact opposite of the goal. The discipline is not "never take on debt" (that's paralysis); it's **take debt deliberately, track it, and repay it before it compounds.**

---

## Visual / Diagram description

**Diagram A — The Strangler Fig migration over time.** All client traffic flows through a facade you control. Over three phases, the facade routes an ever-larger share to the new system, until the legacy system is dead.

```
        PHASE 1: facade in, 100% still legacy
        ┌──────────┐
        │ Clients  │
        └────┬─────┘
             ▼
        ┌──────────────┐
        │   FACADE     │   (proxy / API gateway — the seam you own)
        │  (router)    │
        └────┬─────────┘
             │ 100%
             ▼
        ┌──────────────┐
        │ LEGACY SYSTEM│   (still doing everything)
        └──────────────┘

        PHASE 2: one slice strangled
        ┌──────────┐
        │ Clients  │
        └────┬─────┘
             ▼
        ┌──────────────┐
        │   FACADE     │
        └───┬──────┬───┘
        70% │      │ 30%  (e.g. /search now goes new)
            ▼      ▼
   ┌──────────┐  ┌──────────────┐
   │  LEGACY  │  │ NEW: Search  │
   └──────────┘  └──────────────┘

        PHASE 3: fully strangled, legacy deleted
        ┌──────────┐
        │ Clients  │
        └────┬─────┘
             ▼
        ┌──────────────┐
        │   FACADE     │
        └───┬────┬─────┘
        ▼   ▼    ▼   (100% across new services)
  ┌───────┐ ┌───────┐ ┌───────┐
  │Search │ │Orders │ │ Users │
  └───────┘ └───────┘ └───────┘
        (LEGACY SYSTEM  ──▶  deleted)
```

Every request always has a home. There is never a "big bang" cutover night. If the new Search service breaks, the facade flips that 30% back to legacy in seconds — the migration is reversible at every step.

**Diagram B — the one-way vs two-way door decision quadrant.** Sort every decision by two axes: how hard it is to reverse, and how big the blast radius is if you're wrong. Spend your deliberation where both are high.

```
   HIGH  ┌─────────────────────┬─────────────────────┐
         │  DELIBERATE, but     │  STOP. Write a doc.  │
  blast  │  reversible enough    │  Get review.         │
 radius  │  to correct           │  (data model,        │
   if    │  (caching strategy)   │   primary datastore, │
  wrong  │                       │   public API,        │
         │                       │   micro-split)       │
         ├─────────────────────┼─────────────────────┤
         │  Decide in seconds.   │  Decide fast, but    │
   LOW   │  Don't even discuss.  │  keep it behind a    │
         │  (lint rules, folder  │  seam so you CAN     │
         │   names, log lib)     │  reverse it          │
         └─────────────────────┴─────────────────────┘
              LOW  ◀── hard to reverse? ──▶  HIGH
```

The junior mistake is living in the bottom-left box for a week. The senior move is recognizing the top-right box and *slowing down only there.*

---

## Real world examples

### Amazon — one-way vs two-way doors as company doctrine

The two-way/one-way door framing comes from Jeff Bezos's shareholder letters, and Amazon uses it as an actual decision-making rule (representative of how they describe it publicly): most decisions are two-way doors and should be made quickly by small teams without heavy process, precisely *so that* the rare one-way doors get the deliberation they deserve. The cultural payoff is speed — you don't get organizational gridlock when 95% of decisions are correctly recognized as cheap to reverse.

### Shopify / GitHub — the modular monolith as an evolutionary bet

Both companies are well-known (representative) examples of scaling a large **modular monolith** rather than shattering everything into microservices early. The public lesson is consistent: enforce strict internal boundaries between modules — often with automated checks (fitness functions) that fail the build if a forbidden cross-module dependency appears — so the codebase stays evolvable and *extractable* without paying the distributed-systems tax before it's actually needed.

### Legacy migrations everywhere — Strangler Fig in production

The Strangler Fig is the *default* pattern for large migrations across the industry — moving a monolith to services, moving on-prem to cloud, replacing an aging payments core. The recurring, representative story is the same shape: stand up a facade/gateway, carve off the highest-value or lowest-risk slice first, route it to a new service, prove it, then repeat — with the ability to route back instantly if a slice misbehaves. Teams that instead attempted big-bang rewrites are the cautionary tales.

---

## Trade-offs

**Evolutionary approach vs big up-front design**

| | Evolutionary architecture | Big up-front design |
|---|---|---|
| Speed to first value | Fast — ship the simple thing now | Slow — design the whole thing first |
| Handles unknown future | Well — decide later with real data | Badly — bets on guesses |
| Risk of over-engineering | Low | High |
| Requires discipline | High — needs clean seams + fitness functions | Lower day-to-day, huge upfront |

**Strangler Fig vs big-bang rewrite**

| | Strangler Fig | Big-bang rewrite |
|---|---|---|
| Ships value during migration | Yes, every iteration | No, nothing for months/years |
| Reversibility | High — flip a route back | Near zero — all-in on launch day |
| Edge-case knowledge | Carried over piece by piece | Rediscovered as incidents |
| Upfront complexity | The facade + dual-running | "Just" a new codebase (a trap) |
| Historical success rate | High | Low |

**Technical debt**

| | Deliberate debt | Accidental debt |
|---|---|---|
| Chosen consciously | Yes | No |
| Tracked / has repayment plan | Yes | No |
| Effect over time | Repaid before it compounds | Compounds until change is impossible |

**The sweet spot:** build the simplest thing that solves *today's* problem, keep the boundaries between parts clean and cheap to cut along, protect those boundaries with automated fitness functions, and take on debt only when it's deliberate and written down. That's an architecture that can keep evolving for a decade.

---

## Common interview questions on this topic

### Q1: "How would this design evolve if traffic went 100x?"

**Hint:** Don't redesign from scratch. Walk the request path and name the *seams* where change is cheap: the stateless app tier scales horizontally behind the load balancer; the cache absorbs read growth; the one part that *doesn't* scale trivially (say, the single write-primary database) is the one-way door you'd deliberately address — via sharding or read replicas — and you'd say so explicitly. Naming what stays flexible vs what needs real work is the whole answer.

### Q2: "You've inherited a 10-year-old monolith you must modernize. What's your plan?"

**Hint:** Strangler Fig. Facade in front, migrate the highest-value or lowest-risk slice first, route it to a new service, keep the ability to route back, repeat. Explicitly reject the big-bang rewrite and give the three reasons: lost edge-case knowledge, months of shipping nothing, and the old system moving underneath you.

### Q3: "What's a two-way door vs a one-way door? Give an example of each from a design."

**Hint:** Two-way = cheap to reverse (logging library, an internal function signature, a cache eviction policy). One-way = expensive/impossible to reverse (core data model, primary datastore, a public API contract, splitting into microservices). The point you're demonstrating: you spend deliberation budget on the one-way doors and move fast on the two-way ones.

### Q4: "How do you keep an architecture from rotting over time?"

**Hint:** Fitness functions — automated tests in CI that fail the build when an architectural rule is violated: forbidden dependency direction, a latency budget, a coupling ceiling, no cyclic dependencies, bundle-size limits. They turn a one-time review into a continuously enforced invariant.

### Q5: "Is technical debt always bad?"

**Hint:** No. Deliberate, tracked debt taken to hit a deadline is a legitimate tool — a smart loan with a known interest rate and a repayment ticket. What's dangerous is *accidental* debt from neglect, and *any* debt left untracked, because it compounds until change becomes impossibly expensive.

---

## Practice exercise

### Write a fitness function and plan a strangle

Take any small Node service you have (or scaffold a two-folder project: `src/domain/` and `src/web/`). Do both parts:

**Part 1 — a real fitness function (~15 min).** Write a Jest test that fails the build if any file in `src/domain/` imports from `src/web/` (business logic must not depend on the HTTP layer). Use `grep -rl` via `child_process`, exactly like the example in Key Concept 4. Deliberately add a bad `require("../web/router")` in a domain file and watch the test go red, then remove it and watch it go green.

**Part 2 — a strangle plan (~15 min).** Pick one legacy endpoint in the same service (e.g. `/search`). On paper, write the four-step Strangler Fig plan to replace it: (1) where the facade/router lives, (2) which slice you migrate first and why, (3) how you'd route 10% of traffic to the new implementation, (4) how you'd instantly route back if it misbehaves.

**What to produce:** the passing/failing test file, plus a short markdown plan naming your facade, your first slice, and your rollback switch. If you can articulate the rollback in one sentence, you understand reversibility.

---

## Quick reference cheat sheet

- **Evolutionary architecture:** design so *change is cheap*, because change is guaranteed. City, not building.
- **Core stance:** you can't predict the future — build the simplest thing that meets today's needs AND is cheap to change.
- **Architecture is a stream of decisions**, not a one-time blueprint. Defer big calls until you have real data.
- **Two-way door:** cheap to reverse (log lib, internal signature) — decide fast, don't agonize.
- **One-way door:** expensive to reverse (data model, datastore, public API, micro-split) — slow down, write a doc.
- **Enabling practice — loose coupling / high cohesion (24):** the foundation; program to interfaces (DIP) so implementations swap.
- **Enabling practice — contracts + versioning (69, 138):** internals change freely while the promise holds; version to evolve without breaking clients.
- **Enabling practice — modular monolith (71):** often *more* evolvable than premature microservices; clean boundaries = future extraction seams.
- **Enabling practice — event-driven (68):** a new consumer needs zero producer changes.
- **Fitness function:** automated CI test that fails the build when an architectural quality drifts (forbidden dep, latency budget, coupling ceiling, no cycles).
- **Strangler Fig:** facade in front of legacy, migrate slice by slice, route back instantly on failure, delete legacy when fully strangled.
- **Why rewrites fail:** lost edge-case knowledge, ship nothing for months, old system moves underneath you.
- **Two failure modes:** over-engineering (flexibility you never use) vs under-engineering (rigid ball of mud). Balance: build for today, keep seams clean.
- **Technical debt:** deliberate + tracked = a smart loan; accidental + unmanaged = compounds until change is impossible.

---

## Connected topics

| Direction | Topic | Why |
|-----------|-------|-----|
| **Previous** | [135 — Zero-Downtime Deployments](./135-zero-downtime-deployments.md) | The mechanics (blue-green, canary) that let you migrate and route traffic gradually — the delivery muscle Strangler Fig relies on |
| **Next** | [140 — System Design Interview Playbook](./140-system-design-interview-playbook.md) | Where "how would this evolve if X changed?" becomes a scored question — this topic is your answer |
| **Related** | [24 — Coupling & Cohesion](./24-coupling-and-cohesion.md) | The foundational property that makes any part cheap to change independently — the bedrock of evolvability |
| **Related** | [71 — Microservices vs Monolith](./71-microservices-vs-monolith.md) | The modular monolith is often the most evolvable choice; its boundaries are the seams you extract services along later |
| **Related** | [68 — Event-Driven Architecture](./68-event-driven-architecture.md) | Structural decoupling: adding a new consumer touches nothing upstream — evolvability baked into the topology |
