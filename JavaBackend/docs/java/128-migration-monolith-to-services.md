# 128 — Migration Planning II: Monolith → Services, and the Strangler

## Phase: 12 — Principal Track
## Category: ELITE
## Java baseline: 21  |  Notes features from: 21/25
## Project spine: a strangler plan for extracting `orderflow`'s payments domain — the data-ownership cut, the transaction that has to be replaced, and the rollback at every step

---

## Mechanical statement

> **The hard part of extracting a service is the DATA boundary, not the code
> boundary. If you cannot enforce the boundary inside the monolith — no shared
> tables, no cross-boundary joins, no cross-boundary transaction — then
> extracting it will produce a distributed monolith with network latency in the
> middle.**

The code split is a week's work. Move the packages, define an interface, wire it
up. Any competent engineer can do it and it will look finished.

The data split is a quarter, and it is a quarter whether you plan for it or not.
Because right now, somewhere in `orderflow`, there is a query that joins `orders`
to `payments`, and a transaction that writes to `inventory`, `wallets` and
`payments` and commits or rolls back all three together. Those two things are the
service boundary's real obstacles, and neither of them is in the code you are
planning to move.

Here is the mechanic underneath the mechanical statement. A method call inside one
JVM has exactly one outcome shape: it returns, or it throws, and either way you
know which. A call across a network has **three**: it succeeds, it fails, or it
*succeeds and you do not find out* — the response is lost, the timeout fires, and
your caller cannot distinguish "did not happen" from "happened, and I am about to
retry it". Extraction converts every call across the boundary from the first shape
into the second. Every one. And `@Transactional` cannot span it, so every
invariant that was previously maintained by a database commit now has to be
maintained by you, in application code, in the presence of partial failure.

That is the actual cost of the split, and it is paid in the data layer.

**The test, and it is a real test you can run before committing to anything:**
enforce the boundary *inside* the monolith first. Separate schema, no
cross-boundary foreign keys, no cross-boundary joins, no shared transaction, calls
only through one interface whose failure modes include timeout. If you can do
that and the system still works, extraction is a deployment change. If you cannot
do it — if there is a join you cannot remove or a transaction you cannot break —
then extracting will not fix that. It will take the same coupling and put a
network in the middle of it, where it is slower, less reliable, and much harder to
debug.

---

## The bridge from what you know

### What transfers — completely, and I will not re-teach it

You have strong system-design background and you have split systems before. All of
this transfers and none of it belongs in your artefact:

- **The strangler pattern itself.** Named by Martin Fowler after the strangler fig
  — a new system grows around the old one, taking over call paths one at a time,
  until the old one can be removed. You know it. This is a widely-used industry
  practice, not a proprietary method.
- **Service-boundary design from domain boundaries** rather than from technical
  layers. Bounded contexts, aggregates, ubiquitous language. You have this.
- **Sagas and compensating actions** as the replacement for distributed
  transactions. You know why two-phase commit is not the answer in practice.
- **Idempotency keys, at-least-once delivery, the outbox pattern.** You built
  these in Topics 114–116 and you knew the concepts before that.
- **Contract testing, versioned APIs, backward-compatible evolution.** Same skill,
  same discipline.
- **Conway's law as a planning input.** You know that a boundary that does not
  match a team boundary will erode.
- **The organisational reasons splits are proposed** — deploy independence, team
  autonomy, blast-radius reduction — and the fact that most of them can be
  achieved other ways. You have made this argument.

Spend none of your artefact on any of that. Spend it on what follows.

### What does not transfer — the Java/JVM-specific edges of the cut

| You know (Node/Nest) | Java/JVM reality | Verdict |
|---|---|---|
| A module boundary is an `import` you can grep for; Nest modules declare `imports` explicitly | **Component scanning is classpath-wide and implicit.** Nothing declares which packages a Spring bean may depend on; the boundary exists only in people's heads unless a build rule enforces it | **PARTIAL, and worse for you.** In Nest the module graph is written down. In Spring it is not, so "we have a payments module" is usually a folder name. |
| Your ORM makes you write the join, so removing it is visible | **A lazy `@ManyToOne` is invisible at the call site.** `order.getPayment().getStatus()` looks like a field read and is a SELECT — or, after the split, a network call | **NO ANALOGUE.** The boundary crossings you must find are not syntactically distinguishable from field access. This is the single biggest practical difference. |
| An N+1 in Prisma is an N+1 of queries you can see in the log | **After extraction, an N+1 becomes N HTTP calls**, and no fetch join, entity graph or `@BatchSize` can fix it, because those are database features and the data is no longer in the database | **PARTIAL** — you know N+1; you have not met the version that survives every fix you learned in Topic 50. |
| A transaction spans your ORM calls; you know it does not span an HTTP call | **`@Transactional` is a proxy plus a `PlatformTransactionManager` (Topic 54), and it silently does nothing useful across the new boundary** — the local half commits and the remote half is already gone | **PARTIAL** — the trap is sharper because the annotation stays on the method and looks like it is still doing its job. |
| Two Node services means two connection pools, which is roughly two × the connections | **Two JVMs means two HikariCP pools, and Postgres does a process per connection (Topic 109).** The split can make the database ceiling *worse*, not better | **PARTIAL** — the arithmetic is the same; the cost per connection is much higher. |
| Splitting a service adds a hop of latency | **Splitting adds a hop, and the composite p99 is worse than the sum of the parts** because tail latencies compound across a fan-out | **HONEST ANALOGUE** — you know this. What is new is that you now have a Topic 65 baseline to check it against. |
| Deploying two services means two deploys | **Two JVMs means two warm-ups.** Each has its own JIT profile to rebuild (Topic 74) and its own cold-start cost. Splitting one service into two multiplies the warm-up cost, which shows up as a latency spike on every deploy | **NO ANALOGUE** — Node has no equivalent of losing C2's profile on restart. |

### The four Java-specific things to say out loud in the plan

**1. The boundary must be enforced by the build, or it does not exist.**
Spring's component scanning is implicit and classpath-wide (Topic 36). There is
nothing in a Spring application that says "the orders package may not import from
the payments package". Package-private access (Topic 03) helps within a package
and stops at the package edge. JPMS (Topic 20) would do it properly and almost
nobody uses it for applications. So in practice the enforcement mechanism is an
**architecture test** — an ArchUnit rule in the test suite, or a Maven Enforcer /
banned-imports rule — that fails the build on a cross-boundary import. This is
Topic 132's machinery again and it is the very first step of the plan, not a
later tidy-up.

**2. Lazy associations are invisible boundary crossings.**
Topic 49 taught you that a lazy `@ManyToOne` is a runtime-generated proxy holding
only the identifier, and touching any other field triggers a SELECT.
`order.getPayment().getStatus()` is a database round-trip that reads like a field
access. After extraction it is a *network* round-trip that still reads like a
field access, and there is no compiler error and no lint rule that will find it
for you unless you deliberately break the association first.

Which is why the data cut has a mandatory intermediate move: **replace every
cross-boundary association with a plain identifier field.** `Order` stops holding
`Payment payment` and starts holding `UUID paymentId`. That change is ugly, it is
purely mechanical, and it makes every single boundary crossing appear as an
explicit call at a call site you can count. Do it while still in one process.
Everything after it becomes tractable; nothing before it is.

**3. The N+1 you already fixed comes back in a form you cannot fix.**
Topic 50's `GET /orders` endpoint returns orders with lines, products and payment
status. You fixed its N+1 with a fetch join and you proved it with a query
counter. After extraction, "payment status" lives in another service. The fetch
join cannot reach it. If the new code loops over 50 orders calling the payments
API, you have an N+1 of HTTP calls: 50 network round-trips where there was one
query, each with its own tail latency, and the composite p99 will be dreadful in
a way your unit tests will never show.

There are exactly three honest fixes and the plan must pick one per read path:
a **batch endpoint** (`GET /payments?orderIds=…`), **event-carried state
transfer** (payments publishes status changes; orders keeps a denormalised
read-model column), or **do not show it on that screen**. Picking one is a design
decision with a staleness consequence, and it belongs in the plan, not in a
sprint.

**4. `@Transactional` stops working and does not tell you.**
Order placement today is one method annotated `@Transactional` that reserves
inventory, debits the wallet, creates a payment and publishes an event, and either
all of it happens or none of it does. Move payments out and that method still
compiles, still carries the annotation, and still begins a transaction — which now
covers the inventory reservation and nothing else. The remote payment call inside
it is worse than useless: it is Topic 55's failure exactly, an external call
holding a pool connection for its whole duration, converting a downstream blip
into HikariCP exhaustion across every endpoint in `orderflow`.

So the transactional semantics change is not a consequence of the split. It is a
**prerequisite** for it, and — this is the most important sequencing point in the
topic — it should be made **while still in one process**, behind a feature flag,
so it can be reverted with a config change instead of a redeploy of two services
and a data reconciliation.

---

## What is this?

A strangler plan is a **sequencing document for architectural change**, with the
same two properties as Topic 127's migration plan — every step independently
valuable, every step independently revertible — plus one more that is specific to
this kind of work:

**Every step must be executable while the old path still works.** The strangler's
whole value is that the old system keeps serving traffic while the new one grows
around it. A step that requires the old path to be off is not a strangler step; it
is a cutover with extra vocabulary.

### The five things people mean by "extract a service"

They are not the same and the plan must say which one it is doing, because they
have different costs and different benefits:

1. **A module boundary** — enforced separation inside one deployable. Cheap,
   reversible, and delivers most of the *comprehensibility* benefit.
2. **A data boundary** — separate schema, no cross-boundary joins or FKs. Moderate
   cost, mostly reversible, and this is where the real work is.
3. **A transactional boundary** — the local transaction replaced by a saga or by
   an explicitly-accepted eventual consistency. Expensive, has *product-visible*
   consequences, and is the step people forget is a product decision.
4. **A process boundary** — a separate deployable, separate pool, separate
   lifecycle. This is the one people mean when they say "microservice", and it is
   the *last* one, not the first.
5. **A team boundary** — a different group owns it. Often the actual motivation,
   and it can sometimes be achieved at step 1 without any of the rest.

The plan's opening move is to say which of these five you are actually buying, and
to notice that steps 1 and 2 deliver a surprising fraction of the benefit at a
fraction of the cost. A plan that jumps to 4 without doing 1–3 produces the
distributed monolith, every time, with no exceptions I have seen.

### The distributed monolith, defined precisely

Not a slur — a specific, diagnosable condition. You have one when **two or more
deployables must be deployed together to remain correct.** The observable
symptoms, in the order you meet them:

- A schema change requires both services to ship in a coordinated order, and
  someone maintains a wiki page describing that order.
- Both services connect to the same database. (This one is sufficient on its own.)
- A synchronous call chain: A calls B calls C, in the request path, so the user's
  latency is the sum and the availability is the product.
- One team cannot release without another team's release.
- A rollback of one service requires a rollback of the other.
- The p99 you were trying to improve is worse than it was as a monolith, and the
  difference is the network.
- Debugging any incident requires two log streams and a trace, where before it
  required a stack trace.

You paid the full cost of distribution — network failure modes, partial failure,
operational surface, trace plumbing (Topic 119), doubled connection pools (Topic
109) — and got none of the benefit, because the units are still coupled. This is
strictly worse than the monolith you started with, and it is the default outcome
of splitting the code before splitting the data.

### When *not* to split — say this out loud

A Principal-level plan contains a section on this, and it is not a formality.
Do not split when:

- **The motivation is "we should have microservices".** There is no measurable
  benefit to plan against and no way to tell if it worked.
- **One team owns both sides and will continue to.** You have added distributed-
  systems failure modes to buy an organisational property you already have. The
  boundary will erode because nothing rewards maintaining it.
- **The real problem is deploy friction.** A 40-minute test suite, a weekly
  release train, a shared staging environment. Splitting the service does not fix
  any of those and adds a second instance of each. Fix the pipeline; it is
  cheaper and it is reversible.
- **The domains genuinely share an invariant that must hold synchronously.** If
  the business rule is "an order is never confirmed unless the money moved", and
  the business will not accept a window where that is temporarily untrue, then a
  saga is not a free substitution — it is a change to the product, and the product
  owner has to agree to it.
- **You cannot enforce the boundary inside the monolith.** Repeating this because
  it is the test: if step 1 fails, stop. The failure is telling you something true
  about the domain.

The honest version of this section names the alternatives you would do instead:
enforce the module boundary and stop there; fix the pipeline; split the *database*
without splitting the service; or extract a much smaller, genuinely independent
piece first to learn on.

---

## Why does it matter?

**1. Because the distributed monolith is the modal outcome, not the rare one.**
Most extractions produce it. It is not caused by incompetence; it is caused by
sequencing — the code boundary is visible and satisfying to work on, the data
boundary is invisible and tedious, so teams do the visible one first and then
discover the invisible one is load-bearing. The plan's entire job is to invert
that order.

**2. Because the cost is paid in latency and availability, in production, by
users.** A local method call is nanoseconds and cannot fail. A remote call is
milliseconds and fails three ways. If order placement now makes two synchronous
remote calls, the p99 you recorded in Topic 65 is no longer the number, and the
availability is the product of the parts rather than the availability of one
process. That is a real, quantifiable regression and it belongs in the plan as a
predicted number checked against the baseline (Topic 129), not discovered
afterwards.

**3. Because the transaction change is a product decision disguised as a technical
one.** Today, `orderflow` confirms an order synchronously: the customer's request
returns after inventory, wallet and payment have all committed together. After a
saga, the request returns "accepted" and confirmation arrives shortly after. That
is a user-visible change. If a Principal engineer makes it silently, they have
made a product decision without the product owner, and the first person to notice
will be a customer or a support agent. Surfacing that trade — with the option of
keeping it synchronous and paying for it in availability instead — is exactly the
judgment this topic is testing.

**4. Because "we can always split it later" is true and "we can always merge it
back" is not.** Extraction is close to irreversible in practice: once two teams
own two repositories with two release cadences and two on-call rotations, merging
them back is an organisational change, not a code change. That asymmetry means the
decision deserves the reversibility analysis from Topic 131 and the exit-cost
analysis from Topic 126, and most extraction proposals contain neither.

---

## The decision, framed

Five decisions, in this order. Getting the order right is most of the work.

### Decision 1 — What is the boundary, in terms of data?

Not "what code moves". **Which tables does the new service own, exclusively, as
the only writer and the only direct reader?**

Ownership means three things and all three are required:
- **Sole writer.** No other service writes these tables. Ever. Not "only in the
  batch job".
- **Sole direct reader.** Everyone else goes through the API or through events.
- **Sole schema owner.** Its migrations are its own; nobody else's Flyway touches
  it.

The exercise that produces this: list every table, assign each to exactly one
side, and then **find every query that crosses the line.** Every join, every
foreign key, every transaction that writes to both sides. That list is the plan.
It is also usually four times longer than anyone expected, which is why it is
step 0 and why it is done with `grep`, the SQL log at baseline load, and the
Hibernate `Statistics` machinery from Topic 50 — not from memory.

### Decision 2 — For each crossing, which resolution?

There are exactly five, and each has a cost you must name:

| Resolution | What it is | Cost |
|---|---|---|
| **Move it** | The query belongs entirely on one side; move the code | Cheapest. Do this wherever it is honest. |
| **API call** | The caller asks the owner | A network hop per call, and an N+1 risk if it is in a loop |
| **Batch API call** | The caller asks the owner for many at once | One hop, but a wider API and a coupling on batch size |
| **Event-carried state transfer** | The owner publishes changes; the reader keeps a local denormalised copy | Eventual consistency with a staleness bound you must state, plus a rebuild path |
| **Denormalise at write time** | Copy the field across at the moment it is written | Simplest read; needs a backfill and a repair job |

The wrong answer is "API call" everywhere, because it is the default and it looks
clean. That is precisely how you get the network N+1 and the synchronous chain.
For read paths that render lists — `GET /orders` with payment status is the
canonical one — event-carried state transfer is usually right, and the plan must
state the staleness bound ("payment status may be up to N seconds stale on the
order list; the order detail page reads through") because that bound is a product
statement.

### Decision 3 — What replaces the transaction?

Three options. Name the one you chose and why, and name what you gave up.

**(a) Keep it synchronous.** Orders calls payments in-request and waits. The order
is confirmed or rejected before the response. You keep the user-visible semantics
and you pay:
- availability becomes the product of both services;
- p99 becomes the sum plus tail amplification;
- you must handle the third outcome — timeout with unknown result — with an
  idempotency key and a reconciliation path, because "I don't know if the money
  moved" is now a state your system can be in;
- and you must **never** make that call inside a `@Transactional` method, for
  Topic 55's reason.

**(b) Saga with compensation.** Order is created `PENDING`, an event goes out via
the outbox (Topic 115), payments consumes it idempotently, does its work, and
publishes a result; orders consumes that and either confirms or compensates
(release the inventory reservation, cancel the order, notify). You get
availability and decoupling and you pay in user-visible semantics — "accepted"
now precedes "confirmed" — and in a compensation path that must be built, tested
and monitored, including the case where the compensation itself fails.

**(c) Do not split this operation.** Extract the parts of payments that are not in
the order-placement transaction — the gateway callbacks, the reconciliation, the
refunds, the reporting — and leave the synchronous debit where it is. This is
frequently the best answer and it is almost never on the list, because it does not
feel like a real extraction. It is: it moves most of the code, most of the
operational surface and most of the change traffic, and it leaves the one
invariant that genuinely wants a local transaction alone.

### Decision 4 — What is the granularity of the rollout, and the rollback at each step?

Same rule as Topic 127: the unit is the thing you can put back. Here the steps
have very different rollback costs, and the plan must be honest about where the
cheap ones end:

- Boundary enforcement, association-to-ID, join removal: **revert the commit.**
- Schema separation while still one process: **revert the commit**, plus a schema
  step that must be expand-contract.
- Transaction change to a saga, still one process: **feature flag**, because both
  code paths exist. This is the single highest-value sequencing decision in the
  plan.
- Deploy as a separate process, dual-running: **route flag back to in-process.**
  Requires the data to be reachable from both, which is the expensive part.
- Delete the in-monolith module: **irreversible, deliberately, after a soak.**

### Decision 5 — What is the forcing function that deletes the old path?

Strangler migrations stall in exactly the same shape as version migrations, for
exactly the same reason (Topic 127). Once the new service handles the interesting
traffic, the remaining call sites into the old in-monolith module are the boring
ones — the admin screen, the monthly report, the batch job — and nobody is
suffering from them. So the old module stays, forever, and now you are maintaining
payments logic in two places, which is worse than either alternative.

The mechanism is the same shape: a deprecation on the old module's entry points
enforced in CI, a date on which the old module is deleted, and someone doing the
remaining call-site migrations for the teams rather than asking them to. And a
burndown by call site with an owner, not a percentage.

---

## Example 1 — a minimal illustration

Small enough to hold in your head. It contains the whole mechanic and it takes
about a day.

### The situation

`orderflow` sends notifications: an order-confirmation email, a payment-receipt
email, a low-stock alert to an internal channel. The code lives in
`com.orderflow.notifications`, it is about 2,000 lines, and someone proposes
extracting it as a service because "it does not need to be in the critical path".

### The question that decides it in ten minutes

Not "how do we extract it". **What data does it own, and what does it join to?**

Answer: it owns a `notification_log` table (what was sent, to whom, when, and
whether it succeeded). And it joins to `users` for the email address, to `orders`
for the order summary, and to `payments` for the receipt amount.

So it owns one table and reads three. That is a good sign — a boundary with a
small, read-only crossing surface is a boundary that can be cut.

### The cut, in order

**Step 1.** Enforce the boundary in place. Add an ArchUnit rule: nothing outside
`com.orderflow.notifications` may import from inside it, and nothing inside it may
import from `orders`, `payments` or `users` *entity* packages. The build goes red
in eleven places. Nine are `NotificationService` reaching into `OrderRepository`
directly. Two are an admin controller reaching *into* notifications to re-send.

That red build is the actual finding. Before it, "notifications is a module" was a
folder name. Rollback: delete the rule.

**Step 2.** Replace the reads with an explicit input. Instead of
`notificationService.sendOrderConfirmation(orderId)` — which then loads the order,
the user and the payment itself — the caller passes a
`OrderConfirmationRequest(email, orderNumber, total, lines)`. The notification
module no longer knows what an `Order` is.

This is the association-to-value move, and it is where the design actually
happens: you have just discovered that notifications does not need the order
domain, it needs five fields. The crossing surface shrank from three tables to one
record. Rollback: revert the commit.

**Step 3.** Move `notification_log` to its own schema. No foreign keys out of it —
it holds an `order_number` as a string, not an FK to `orders`. Rollback: the
schema move is additive if you keep the old table until the soak ends.

**Step 4.** Extract the process. Now it is genuinely mechanical: the interface
already exists, the input is already a value object, the data is already separate.
Publish the request as an event rather than calling it synchronously — because the
whole premise was that it is not in the critical path, and if you extract it and
then call it synchronously in the request path, you have made availability worse
for no benefit at all.

Rollback: a flag that routes back to the in-process implementation, which is still
present.

**Step 5.** After a soak, delete the in-process implementation. Irreversible, on
purpose.

### What you write down

Five rows: step, what changes, what it buys, rollback, how you know it worked. And
one sentence that is the whole lesson: **step 2 was the design work; steps 3 and 4
were mechanics.** If step 2 had been impossible — if notifications genuinely
needed to run arbitrary queries against the order domain — you would have stopped,
and stopping would have been the right outcome.

---

## Example 2 — the real decision on the project spine

### The setting

`orderflow` is a production-shaped service (Topic 124). It has a Topic 65 baseline
with recorded p50/p95/p99, RED metrics on every endpoint and USE metrics on every
pool (Topic 118), end-to-end tracing (Topic 119), a transactional outbox (Topic
115), idempotent consumers, and circuit breakers on the payment gateway (Topic
111). It runs as one deployable against one Postgres database.

The proposal on the table: **extract payments into its own service.** The stated
reasons are that payment changes require a full `orderflow` release, that the
payment-gateway integrations have a different change cadence from the order
domain, and that a payments team is being formed.

Those are three good reasons. Two of them are organisational and one of them
(release coupling) might be fixable another way. The plan should say so, and then
proceed, because the team boundary is real and a team boundary that does not match
a service boundary erodes.

### Step 0 — The data-ownership cut, done properly

The tables, and the proposed assignment:

| Table | Proposed owner | Notes |
|---|---|---|
| `orders`, `order_lines` | orders | |
| `products`, `inventory` | orders (catalogue/inventory stay) | |
| `payments` | **payments** | The obvious move |
| `payment_gateway_events` | **payments** | Callback fan-in (Topic 108) |
| `wallets`, `wallet_transactions` | **contested — see below** | The hard call |
| `outbox` | Both, one each | Each service gets its own outbox table; they are not shared |

**The contested one, and why it is the interesting decision.** `wallets` is a
ledger with its own invariants (a balance is the sum of its transactions; it must
never go negative), and it is debited inside the order-placement transaction. Two
defensible answers:

- **Wallets go with payments.** Payments owns all money movement. Order placement
  then makes *one* boundary crossing for money instead of two, which is
  simpler, and the ledger invariant stays inside one transaction boundary where a
  database can enforce it. Cost: wallets is customer-facing (balance display,
  top-up) and those read paths now cross the boundary too.
- **Wallets stay with orders.** Then payments is thin — gateway integration and a
  payment record — and the split buys less, but it buys it cheaply, and the
  order-placement transaction keeps the wallet debit local.

I would take the first, because keeping the ledger's invariant inside one
transactional boundary is worth more than the read-path convenience, and because a
"payments team" that does not own the money ledger will spend its life coordinating
with the team that does. But this is a genuine judgment call and the plan must
state it as one, with the reason and with what would change it: *if wallet balance
is displayed on more than a small number of high-traffic screens, the read-path
cost may flip the answer, and that is measurable from the Topic 118 metrics before
we commit.*

### Step 0, continued — every crossing, enumerated

This is the part people skip. Produced from `grep`, from the SQL log at baseline
load, and from reading the code — not from memory.

| # | Crossing | Where | Kind | Resolution |
|---|---|---|---|---|
| 1 | `orders` JOIN `payments` for payment status | `GET /orders` list (Topic 50's endpoint) | Read, high volume | **Event-carried state transfer** — denormalised `payment_status` on `orders` |
| 2 | `orders` JOIN `payments` for detail | `GET /orders/{id}` | Read, low volume | **API call** — read-through, acceptable at this volume |
| 3 | `Order.payment` lazy `@ManyToOne` | Entity model | Invisible crossing | **Replace with `paymentId`** — mandatory, first |
| 4 | Order placement transaction: inventory + wallet + payment + outbox | `OrderService.placeOrder` | **Write, transactional** | **The transaction decision — see below** |
| 5 | Refund flow writes `orders.status` | `RefundService` | Write, cross-boundary | **Move**: payments publishes `RefundCompleted`; orders reacts |
| 6 | Revenue-by-product report | Admin/reporting | Read, batch, joins `payments` to `order_lines` | **Move the report** to a read store fed by both services' events |
| 7 | Reconciliation job: `payments` vs `wallet_transactions` | Nightly batch | Read, both sides | **Move it** — after the wallet decision, both are in payments |
| 8 | FK `payments.order_id → orders.id` | Schema | Structural | **Drop the FK**, keep the column as an opaque identifier |
| 9 | Support tooling queries both | Internal admin | Read, ad hoc | **Two queries, joined in the tool** — and this one will be unpopular |

Nine crossings. Crossing 3 is done first because it makes the others visible.
Crossing 9 is the one that generates the most complaint and the least engineering
work, and the plan should name it because unaddressed operational-tooling
regressions are a reliable source of "the migration made everything worse".

### The transaction decision, in full

**Today:** `OrderService.placeOrder` is `@Transactional`. Inside one transaction it
reserves inventory (an atomic conditional UPDATE, from Topic 52), debits the
wallet, creates a `payments` row in `PENDING`, and writes an outbox row. Commit or
rollback, all four together. The customer's HTTP response is the confirmed order.

**Option (a) — synchronous remote call.** `placeOrder` reserves inventory locally,
then calls the payments service, then commits.

Rejected, and the reason is specific: to be correct, the remote call must be made
**outside** the transaction, because a remote call inside a `@Transactional`
method holds a HikariCP connection for the call's whole duration — Topic 55's
drill, the one that took every endpoint down when a 2-second downstream call was
inside a transaction. Moving it outside means the inventory reservation and the
payment are no longer atomic, which means we need compensation anyway. So option
(a) buys the old user-visible semantics and still requires the compensation
machinery. It also makes availability the product of two services and adds the
payments p99 to the order-placement p99, on the highest-traffic write path.

If the product requirement genuinely is "the response must be the confirmed
order", option (a) becomes viable again — with the reservation done as a
short-lived local transaction, the payment call outside it, and a confirm-or-
compensate step after. State that as the fallback, with its availability cost.

**Option (b) — saga. Recommended.**

1. `placeOrder` runs one **local** transaction: reserve inventory, create the
   order in `PENDING_PAYMENT`, write an outbox row. Commits. Returns 202 with the
   order id.
2. The outbox relay publishes `OrderPlaced`.
3. Payments consumes it **idempotently**, keyed on the order id (Topic 116). It
   debits the wallet and creates the payment, in one local transaction on its own
   database, and writes its own outbox row.
4. Payments publishes `PaymentSucceeded` or `PaymentFailed`.
5. Orders consumes it. Success: order → `CONFIRMED`, and the denormalised
   `payment_status` column is updated. Failure: **compensate** — release the
   inventory reservation, order → `REJECTED`, notify.
6. A reconciliation job runs on a schedule and finds orders stuck in
   `PENDING_PAYMENT` past a threshold, because step 4's message can be lost and
   the compensation must not depend on it arriving.

**What this costs, stated plainly in the plan:**
- The customer-visible semantics change from "confirmed" to "accepted, then
  confirmed within N seconds". **This is a product decision and requires the
  product owner's agreement, in writing, before step 4 is built.** Naming it as a
  product decision rather than shipping it as a technical detail is the single
  most Principal-shaped move in this document.
- Inventory can be reserved for an order that is later rejected. The reservation
  needs a TTL and a sweeper, which is new code with new failure modes.
- Step 6 must exist, must be monitored, and its "stuck orders" count is an SLI
  (Topic 130) with an alert, because it is the safety net for every lost message.
- The compensation path itself can fail. What happens when the inventory release
  fails? Answer in the plan: retry with backoff, then a dead-letter queue with an
  alert and a documented manual procedure. Do not leave this as an exercise.

**Option (c) — do not split this operation.** Extract the gateway integrations,
the callback fan-in, the refunds, the reconciliation and the reporting; leave the
wallet debit and payment record creation inside the monolith's order transaction,
called through the enforced module boundary.

This is a serious option and the plan must include it. It gets the payments team
the code they change most often, it gets the release-cadence decoupling that was
one of the three stated motivations, and it does not change the user-visible
semantics of order placement at all. What it does not get is a clean data
boundary, which means it is a way-station rather than a destination. If the
programme's real driver is the team boundary and the gateway change cadence, this
delivers most of it in a fraction of the time and with none of the product risk —
and a Principal engineer who cannot bring themselves to put it on the list is
arguing for an outcome rather than evaluating one.

### The sequence

| # | Step | Independently valuable? | Rollback | Verified by |
|---|---|---|---|---|
| 1 | ArchUnit rule: no cross-boundary imports. Baseline existing violations, ratchet down | Yes — the boundary becomes real and cannot regress | Revert commit | Build red count going to zero; rule enforced on new code from day one |
| 2 | Replace `Order.payment` association with `paymentId`; same for the reverse | Yes — every crossing is now a visible call site | Revert commit | `grep` count of crossings; Topic 50 query counter unchanged |
| 3 | Resolve read crossings 1, 2, 6, 9 while in-process | Yes — read paths no longer join | Revert commit | Query log shows no `orders`⋈`payments` join at baseline load |
| 4 | Add denormalised `payment_status` to `orders`, maintained in-process; readers switch to it | Yes — the list endpoint gets faster | Column is additive; readers revert to the join | Baseline p95 on `GET /orders` compared |
| 5 | Separate schema; drop cross-boundary FKs; separate Flyway histories | Yes — schema ownership is now unambiguous | Expand-contract; old schema retained through soak | No cross-schema query in the SQL log at baseline load |
| 6 | **Saga replaces the local transaction — still one process, behind a flag** | Yes — the hard semantic change is proven before distribution | **Config flag back to the local transaction** | Load test at baseline; stuck-order count; compensation exercised deliberately |
| 7 | Deploy payments as a separate process against its own database; route via flag; the in-process module remains | Yes — deploy independence achieved | **Route flag back to in-process**, subject to data reachability | Baseline re-run; composite p99 compared to prediction; trace shows the hop |
| 8 | Migrate remaining call sites; deprecate the in-monolith module in CI | Yes — one implementation, not two | Per call site | Burndown by call site and owner |
| 9 | Delete the in-monolith payments module | Yes — the point of the whole exercise | **None, deliberately, after a soak** | Module gone; no dead code |

**Why step 6 comes before step 7, and why that is the most important row in the
table.** The saga is the change with product-visible consequences and the most
subtle failure modes: lost messages, duplicate consumption, failed compensation,
stuck orders. Doing it while both code paths live in one process means the
rollback is a configuration flag and the blast radius is one deployable. Doing it
*after* the process split means the rollback is "redeploy two services and
reconcile two databases", under pressure, at 3 a.m. The engineering work is
identical; the risk is not remotely comparable.

**The step 7 honesty note.** Step 7's rollback is the weakest in the plan and the
document must say so rather than let a reviewer find it. Once payments is writing
to its own database, routing back to in-process means the in-process path cannot
see the payments written while the flag was on. The mitigations, in decreasing
order of cost and effectiveness: dual-write for the soak window (expensive,
correct); run the new service in shadow mode first, taking traffic and writing
results but not being authoritative (cheaper, and it validates everything except
the write path); or accept a short forward-only window with a documented manual
reconciliation and a named owner (cheapest, honest, and fine if the window is
hours rather than weeks). Pick one, in the plan, and say why.

### The predicted numbers, before you start

From the Topic 65 baseline and the Topic 129 model, the plan states what it expects
to happen — so that "it got slower" is a checked prediction rather than a
surprise:

- Order placement gains one asynchronous hop and loses one synchronous wallet
  write. Predict the direction and the rough magnitude, and check it.
- `GET /orders` loses a join and gains a denormalised column read. Predict it gets
  slightly faster.
- Total Postgres connections becomes the sum of two pools. From Topic 109, the
  pool is the ceiling, and Postgres does a process per connection — so **the
  combined pool budget must not exceed what one database could take before**, or
  you have made the ceiling lower while believing you raised it. Write the two
  pool sizes down and their sum.
- Two JVMs means two warm-ups per deploy (Topic 74) and two live sets (Topic 70).
  The memory footprint of the pair is not the footprint of the one.

Writing these predictions before the work is what converts the migration from a
hope into an experiment.

---

## Wrong approach → exact symptom → root cause → fix

Organisational and architectural symptoms. What you would have *seen*.

---

### Wrong approach 1 — split the code, share the database

**Wrong:** payments is extracted as a separate deployable in three weeks. It
connects to the same Postgres instance and the same schema, because "the data
split can come later".

**Exact symptom — what you would have SEEN:**

- A schema change requires a coordinated deploy of both services in a specific
  order, and there is a Confluence page named "Deploy order for payment schema
  changes" that four people have edited.
- Two Flyway histories against one schema. The second one to run either fails or,
  worse, succeeds against a state the first did not expect.
- Total Postgres connections has doubled — two HikariCP pools where there was one —
  and Postgres does a process per connection, so p99 across *every* endpoint has
  degraded, including endpoints neither service changed. Topic 109's lesson,
  arriving as a surprise.
- Someone in payments adds a column that orders' Hibernate mapping does not know
  about, and a `SELECT *` somewhere breaks. Or an orders migration renames a
  column payments reads, and payments starts throwing at 03:00.
- A production incident requires the on-call engineer from *both* services, and
  the retrospective action is "improve communication between the teams".
- The original motivation — payments changes should not require an `orderflow`
  release — is not met, because payment schema changes still require both.

**Root cause:** the boundary that matters was never cut. The service boundary is a
deployment fact; the coupling is a data fact; and the data fact did not change.
Sharing a database means sharing a schema means sharing a release, which is the
definition of a distributed monolith.

**Fix:**
1. **Data first, always.** Separate schema — separate database instance if you can
   afford it — with no cross-boundary FKs and no cross-boundary queries, achieved
   *before* the process split, while it is still cheap to revert.
2. Enforce it. Separate database users with grants that make a cross-boundary
   query fail, rather than a convention that makes it discouraged. A permission is
   a mechanism; a convention is a memo.
3. Budget the pools against the database's real capacity, as a sum, before the
   split. Two services do not get two pools' worth of headroom; they share one
   database's.
4. If you cannot cut the data, do not do the split. Do the module boundary and
   stop. That outcome is a success, not a retreat.

---

### Wrong approach 2 — the distributed monolith with network latency in the middle

**Wrong:** the split is done properly at the data layer, but every interaction
that used to be a method call becomes a synchronous HTTP call, in the request
path, one for one.

**Exact symptom — what you would have SEEN:**

- `GET /orders` renders 50 orders and makes 50 calls to the payments service to
  fetch each order's status. The flame graph (Topic 78) is a wall of wall-clock
  time in the HTTP client. This is Topic 50's N+1, resurrected in a form where a
  fetch join cannot touch it.
- p99 on the order list is several times the baseline you recorded in Topic 65,
  and the p99 of the *composite* is much worse than the p99 of either service,
  because tail latencies compound: at 50 sequential calls, hitting at least one
  slow response is close to certain.
- The circuit breaker (Topic 111) opens on the payments service during a routine
  deploy, and `GET /orders` — which is a read path with no business need for live
  payment data — returns errors. Availability is now the product of two services
  on a path that used to be one.
- A trace (Topic 119) for a single request has 50 sibling spans and takes eight
  seconds to render in the UI.
- Deploying payments causes a visible latency spike in orders, so the teams start
  coordinating deploy windows — which was the exact thing the split was supposed
  to eliminate.
- Someone proposes fixing it by caching the payments responses in orders. That is
  the right instinct arriving as an incident response rather than as a design.

**Root cause:** every crossing was resolved the same way — "API call" — because it
is the default and the mechanical translation of a method call. The design work of
Decision 2 was skipped: nobody asked, per crossing, whether the reader needs live
data, whether the call is in a loop, and whether the data could be pushed instead
of pulled.

**Fix:**
1. Resolve each crossing deliberately: move / API / **batch** API / event-carried
   state transfer / denormalise. Any crossing inside a loop is automatically
   disqualified from "API call".
2. For list-rendering read paths, event-carried state transfer with a stated
   staleness bound. The denormalised `payment_status` column on `orders` is not a
   hack; it is the correct design, and the staleness bound is a product statement.
3. Add a build-time or test-time rule that fails when a remote client is invoked
   inside an iteration over a collection. It is crude and it catches the case that
   matters.
4. Predict the composite p99 *before* the split, from the baseline and the Topic
   129 model, and check it after. If you did not predict it, you cannot tell
   whether what you got is acceptable.

---

### Wrong approach 3 — extract first, define the boundary from what is easy to move

**Wrong:** the boundary is drawn around the code that was easiest to lift — the
`payments` package — rather than around a data-ownership decision.

**Exact symptom — what you would have SEEN:**

- The new payments service exposes fourteen endpoints, and eleven of them are CRUD
  on payments tables: `GET /payments/{id}`, `PUT /payments/{id}/status`,
  `POST /payments/{id}/notes`. That is not a service; it is a database with HTTP
  in front of it, and the business logic that used to guard those writes is still
  in orders.
- Orders reaches in and sets a payment's status directly, because that is what the
  code did before. The invariant that used to be enforced by a method in the same
  process is now enforced by nothing.
- Every new feature requires a change to both services and a coordinated release,
  and the API grows an endpoint per feature.
- The team says "the API is too chatty" and adds a composite endpoint. Then
  another. Within a year there is a `POST /payments/process-order-placement` that
  takes the entire order as its body, which means the payments service now has an
  order model, which means the boundary has moved back into the middle of the
  order domain.
- Nobody can answer "who owns the rule that a wallet cannot go negative?"

**Root cause:** the boundary was drawn in code space rather than in data-ownership
space. A service that owns tables but not the *rules* about those tables is a
remote database, and it inherits every coupling the original code had plus the
network.

**Fix:**
1. Draw the boundary from Decision 1: which tables, sole writer, sole reader, sole
   schema owner — and, crucially, **which invariants move with them**. If the rule
   "wallet balance never goes negative" moves to payments, then the operation that
   enforces it moves too, and orders never sets a balance.
2. The API is a set of **business operations**, not table access. `POST
   /payments` with an idempotency key, not `PUT /payments/{id}/status`. If you can
   describe an endpoint without using a domain verb, it is probably wrong.
3. Test the boundary before extraction by trying to state the service's purpose in
   one sentence with no reference to tables. If you cannot, the boundary is not a
   boundary.

---

### Wrong approach 4 — the transaction change made after the process split

**Wrong:** the sequence is code split, data split, process split, and *then* "now
we need to figure out consistency".

**Exact symptom — what you would have SEEN:**

- The saga is implemented across two deployed services under time pressure,
  because the split already shipped and the system is currently inconsistent in
  production.
- The first bug: duplicate wallet debits, because the consumer was not idempotent
  and the message was redelivered. This is Topic 92's check-then-act failure with
  a network in front of it, and the customer sees it as being charged twice.
- The second bug: orders stuck in `PENDING_PAYMENT` forever, because
  `PaymentSucceeded` was lost and there was no reconciliation sweep. Discovered by
  a customer support ticket, days later.
- The third: the compensation fails — inventory release throws — and there is no
  dead-letter path, so the reservation is held until someone notices the stock
  discrepancy in a weekly report.
- Rollback is proposed and rejected, because rolling back now means reconciling
  two databases that have diverged. The team is committed to fixing forward on a
  system they do not yet understand, at the worst possible moment.
- A product manager finds out that orders are now "accepted" rather than
  "confirmed" from a customer complaint.

**Root cause:** the riskiest and most semantically significant change was made in
the environment with the most expensive rollback. The saga's difficulty has
nothing to do with the process split — it is about partial failure, idempotency
and compensation — and all of that can be built and proven inside one process.

**Fix:**
1. **Do the transaction change first, in-process, behind a flag.** Both code paths
   exist; the rollback is a config change. This is the highest-value sequencing
   decision available in an extraction.
2. Exercise the failure paths deliberately before the split: kill the process
   between the local commit and the publish (Topic 115's drill), redeliver a
   message, force a compensation, force a compensation failure. Every one of those
   is a Phase 8–11 drill you have already run in another form.
3. Get the product owner's agreement to the semantic change **before** building
   it, in writing, and put the agreed wording in the plan.
4. Build the reconciliation sweep in the same pull request as the saga, not later.
   It is the safety net for every message the system loses, and a saga without one
   is a design that assumes the network is reliable.

---

### Wrong approach 5 — splitting to fix a problem that is not architectural

**Wrong:** the extraction is motivated by "we can't deploy quickly", and the plan
does not check whether the service boundary is what is preventing that.

**Exact symptom — what you would have SEEN:**

- Two quarters and several engineer-years later, deploy frequency is **unchanged**.
  The test suite still takes 40 minutes; now there are two of them. The release
  train is still weekly; now two services are on it.
- On-call load has gone up: two services, two dashboards, two alert sets, a new
  class of incident (partial failure) that did not previously exist, and traces to
  read instead of stack traces.
- The team is smaller in effective capacity than before, because two people are
  now doing platform work that did not exist.
- Someone measures and finds the actual deploy blocker was a shared staging
  environment with a manual sign-off, which nobody had looked at because it was
  not an architecture problem and architecture was what the team was good at.
- The retrospective concludes that "we should have done it more incrementally",
  which is true and is not the lesson.

**Root cause:** the intervention was chosen from the team's competence rather than
from a diagnosis. The problem statement ("we can't deploy quickly") was never
converted into a measurement ("deploys take N hours, of which M is X"), so the
most expensive available intervention was applied to a cause nobody had confirmed.

**Fix:**
1. Before any split, write the problem as a **measurable statement with a current
   value and a target**. "Deploy lead time is N hours; we want it under M." Then
   decompose N. This is the same discipline as Topic 130's SLI derivation, applied
   to an internal process.
2. List the cheapest interventions that could move that number: parallelise the
   test suite, split the *pipeline* rather than the service, add a staging
   environment, remove the manual gate. Price each. Extraction is usually the most
   expensive item on that list and should be justified against the others, not
   assumed.
3. If the real motivation is the team boundary — which is legitimate and often
   the strongest reason — say that plainly instead of dressing it as a deploy
   argument. It changes which steps are necessary: a team boundary may be
   satisfiable at step 1 or step 3, without ever reaching step 7.
4. Define, in advance, what would make you stop. "If after step 5 the payments
   team can release independently and deploy lead time has met the target, we stop
   there and do not do steps 6–9." Extractions almost never have a defined
   stopping point, which is why they always run to completion regardless of
   whether completion was needed.

---

## Artefact — what you must produce

### Specification

**Title:** `Extracting payments from orderflow — strangler plan`

**Length:** 2,000–3,000 words of prose, plus tables. Tables do not count. If the
prose runs long you are describing the target architecture rather than sequencing
the route to it, and the route is the deliverable.

**Audience:** the engineering leadership who will fund it, the orders team, the
new payments team, and the product owner who has to agree to the semantic change.

**Required sections, in this order:**

**1. What we are buying, and how we will know.** Which of the five boundaries
(module / data / transactional / process / team) is the actual objective. The
measurable statement of the problem with a current value and a target. The
alternatives you considered that are cheaper than an extraction, and why they were
rejected — including "enforce the module boundary and stop".

**2. The data-ownership cut.** A table assigning **every** table to exactly one
side, with the contested ones marked and decided, each with a reason and a
"what would change this" line.

**3. The crossing register.** Every query, association, foreign key and
transaction that crosses the line, with its resolution:

| # | Crossing | Where | Kind (read / write / transactional) | Volume | Resolution | Cost accepted |
|---|---|---|---|---|---|---|
| | | | | | | |

This is the heart of the document. It must be produced from evidence — `grep`, the
SQL log at baseline load, the Hibernate `Statistics` counter — and the document
must say which. A crossing register produced from memory is worthless and I will
be able to tell, because it will be too short.

**4. The transaction decision.** All three options — synchronous, saga,
don't-split-this-operation — with what each costs. The chosen one, the
user-visible semantic change stated in the words a customer would see, and an
explicit note that the product owner has agreed (or has not yet, and when the
decision is due).

**5. The sequence.** With these exact columns and every cell filled:

| # | Step | Independently valuable? (what it buys if we stop here) | Rollback (as an action) | Rollback preconditions | Verified by |
|---|---|---|---|---|---|

The transaction change must appear **before** the process split, or the document
must argue explicitly why not.

**6. Predicted effects on the baseline.** Before-and-after predictions for: the
order-placement path, the highest-volume read path, total database connections
across both pools, and total memory footprint across both JVMs. Predictions, with
their basis, from the Topic 65 baseline and the Topic 129 model. Not results —
predictions, so that the results can falsify them.

**7. The failure modes you are adding.** At minimum: lost message, duplicate
message, failed compensation, stuck saga, and partial failure with unknown
outcome. For each: how it is detected, what it costs, and what the operator does.
The reconciliation sweep and its alert are a required line item, not a follow-up.

**8. The stopping point.** What condition would make you stop before the last
step, and who decides. Every extraction plan should have one and almost none do.

**9. The forcing function.** How the old in-monolith module actually gets deleted.
Dates, CI enforcement, who migrates the remaining call sites, and a burndown by
call site and owner.

**10. What I am least sure about.** One paragraph, honestly.

### Constraints

- **The crossing register comes from evidence**, and the document says which
  evidence. `grep` counts, SQL logs, query counters.
- **No step may require the old path to be off**, except the last one. If it does,
  it is a cutover and must be labelled as such.
- **Every rollback is an action with preconditions**, not a reassurance.
- **The user-visible semantic change is quoted in customer-facing language**, not
  in engineering terms. "The order confirmation email arrives a few seconds after
  checkout instead of immediately" is the sentence the product owner is agreeing
  to. "Eventual consistency between the order and payment aggregates" is not.
- **No invented latency or throughput numbers.** Predictions are labelled as
  predictions and carry their basis; measurements come from your own baseline.

---

## How I will review it

### The three questions that usually break a document of this kind

**Question 1 — "Show me the query that joins the two sides, and tell me what
replaces it."**

There is always at least one and it is always the read path that renders a list.
For `orderflow` it is `GET /orders` with payment status — the endpoint from Topic
50 that you already fixed once.

*How the document fails:*
- The crossing register does not exist, or lists three crossings when there are
  nine. A short register means the author worked from memory.
- Every crossing is resolved as "API call", including the ones inside loops. That
  is the network N+1 with a plan attached.
- The lazy associations are not in the register at all, because they do not look
  like queries in the source. `order.getPayment().getStatus()` is a crossing and
  will not appear in a `grep` for `SELECT`.
- Foreign keys are not treated as crossings. A cross-boundary FK is a hard
  structural coupling and it has to be dropped, which has referential-integrity
  consequences the document should name.

*What a strong answer looks like:* an evidenced register, a per-crossing
resolution chosen from the five options with the cost named, and — the thing I
most want to see — the association-to-identifier step scheduled first, with the
explicit reasoning that it is what makes the other crossings visible.

**Question 2 — "What happens to the order-placement transaction, and who agreed to
the change your customers will see?"**

*How the document fails:*
- The transaction is not mentioned. This is common and it is the single most
  serious omission possible in this document, because that transaction is the
  reason the split is hard.
- The saga is described but the semantic change is not, so nobody outside
  engineering knows the product is changing.
- The saga is scheduled *after* the process split, which puts the riskiest change
  in the environment with the most expensive rollback.
- There is no reconciliation sweep, which means the design assumes messages are
  never lost — an assumption Topic 115's drill already falsified for you with a
  `kill -9`.
- The compensation is described and its failure is not. "Release the inventory
  reservation" — and when that throws?

*What a strong answer looks like:* all three options considered, the chosen one
with its costs, the semantic change quoted in customer-facing language with a named
product owner who has agreed, the saga scheduled before the process split behind a
flag, the reconciliation sweep in the same step with an alert and an SLI, and the
compensation's own failure path documented.

**Question 3 — "Which step is not revertible, and what did you do instead of
pretending it is?"**

*How the document fails:*
- Every rollback says "revert the deploy". By step 7 that is false, and a reviewer
  will find it in thirty seconds.
- The data-migration step has no rollback discussion at all.
- The plan asserts a dual-write window without pricing it or saying how long it
  lasts.
- The final deletion step is presented as safe rather than as a deliberate one-way
  door with a soak period in front of it.

*What a strong answer looks like:* naming step 7 as the weak point before I do,
picking a mitigation (dual-write / shadow mode / a short documented forward-only
window with an owner), pricing it, and stating the soak period before the
irreversible deletion. Naming your own weakest step is the strongest move in the
document.

### The other attacks, in order

- **"What is the sum of your two connection pools, and what could the database
  take before?"** From Topic 109, the pool is the ceiling and Postgres does a
  process per connection. If the plan adds pool capacity without checking the
  database's total, it has lowered the ceiling while claiming to raise it.
- **"Who owns the rule that a wallet cannot go negative, after the split?"** If
  the answer is unclear, the boundary is drawn around tables rather than around
  invariants.
- **"Your `payment_status` column is stale by how long, and what does the customer
  see during that window?"** Event-carried state transfer is the right answer and
  it has a bound, and the bound is a product statement.
- **"What does support do now?"** The internal tooling that queried both sides is a
  real regression that will be felt on day one, and unaddressed operational
  regressions poison a migration's reputation.
- **"What is the deploy-frequency number today, and what is it after?"** If the
  motivation was deploy independence, it needs a before and an after, or you
  cannot tell whether it worked.
- **"What would make you stop at step 5?"** If there is no stopping condition, the
  plan will run to completion whether or not completion is needed.
- **"Two JVMs, two warm-ups. What does that do to your p99 during a deploy?"**
  Rarely considered, always real (Topic 74).

### What I will not attack

Prose, the choice of message broker, or your view on whether the wallet belongs
with payments — that is a judgment call and both answers are defensible. I am
attacking the weakest assumption the sequence rests on, and in an extraction plan
that is almost always the transaction, the crossing register, or step 7's
rollback.

---

## Interview questions (Senior → Principal)

> The rubric line from the master plan:
> *"we'll extract payments into its own service" → "the code split is a week; the
> data split is a quarter, because payments and orders currently join. Step one is
> enforcing the boundary inside the monolith with no shared tables — if we can't
> do that, extracting will just give us a distributed monolith with network
> latency in the middle."*

---

### Q1 — "We want to extract payments from our monolith. How would you approach it?"

**A senior answer sounds like:** "I'd start by defining the boundary — what
belongs to payments and what doesn't. I'd introduce an interface for it inside the
monolith first so the calls go through one place, then split the database so
payments owns its tables, then deploy it separately, probably behind a feature
flag so we can route traffic gradually. I'd use the strangler pattern so the old
path keeps working. And we'd need to handle the fact that we can't have a
transaction across the boundary any more — probably a saga."

That is a strong answer. It has the right sequence at a high level, it names the
transaction problem, and most candidates do not name the transaction problem.

**A principal answer sounds like:** "The code split is a week. The data split is a
quarter. So I'd spend the first two weeks not writing any extraction code at all,
and instead producing one artefact: every place where the two domains touch in the
data layer. Every join, every foreign key, every lazy association, and every
transaction that writes to both sides. From `grep`, from the SQL log under our
load baseline, and from the Hibernate query counter — not from memory, because
memory will find about a third of them.

That register is the plan. And the crossings that catch people are not the joins
you can see: a lazy `@ManyToOne` from `Order` to `Payment` is a boundary crossing
that reads like a field access at the call site. So the first code change is
purely mechanical and looks like a regression — replace the association with a
plain identifier field. That makes every crossing a visible call site you can
count, and nothing after it is tractable until you do.

Then the test, and this is the part I'd insist on before we commit to anything:
**enforce the boundary inside the monolith**. Separate schema, no cross-boundary
foreign keys, no cross-boundary joins, calls only through one interface, enforced
by an architecture test that fails the build. If we can do that and the system
still works, extraction becomes a deployment change. If we can't — if there's a
join we can't remove or a transaction we can't break — then extracting won't fix
it. We'll have the same coupling with a network in the middle, which is strictly
worse than what we have now.

The hardest single thing is the order-placement transaction. Today it reserves
inventory, debits the wallet, creates the payment and writes the outbox, and
commits all four together. After the split that has to become a saga with
compensation — and the semantics the customer sees change from 'confirmed' to
'accepted, then confirmed a few seconds later'. That's a product decision, not a
technical one, and I'd want the product owner's agreement in writing before we
build it.

And the sequencing point I care most about: **the saga goes in before the process
split, behind a feature flag, while both paths are still in one process.** Same
engineering work either way. But in-process the rollback is a config change, and
after the split the rollback is redeploying two services and reconciling two
databases at three in the morning. The risk difference is enormous and it costs
nothing to get right."

**What separates them:** four things.

1. The senior answer sequences correctly at the level of *activities*. The
   principal answer sequences by **rollback cost**, and moves the riskiest change
   into the cheapest-to-revert environment. That reordering is free and it is the
   highest-leverage decision in the plan.
2. The senior answer says "split the database". The principal answer knows the
   *specific* Java mechanism that makes the crossings invisible — lazy
   associations — and schedules a mechanical step to surface them.
3. The senior answer treats the boundary as something you design. The principal
   answer treats it as a **falsifiable hypothesis you test in-place before
   committing**, with a defined outcome where the answer is "don't split".
4. The senior answer says "we'd need a saga". The principal answer identifies the
   user-visible consequence and routes it to the person who owns that decision.

**Adversarial follow-up:** *"Your first two weeks produce a spreadsheet and no
code. How do you sell that to a director who's already announced the payments
team?"*

The honest answer does not get defensive about the spreadsheet. It says: the
register is not a delay, it is the estimate. Right now nobody can tell that
director whether this is a six-week project or a two-quarter one, and the
difference is entirely in that register. Two weeks to convert an unbounded
commitment into a bounded one is the cheapest thing on the plan. And it is not
zero-output: the architecture rule that enforces the boundary ships in week one,
which delivers real value on its own — the boundary stops eroding — and it is the
first step of the extraction regardless of what the register says. If the director
still wants visible motion, the association-to-identifier change can run in
parallel; it is mechanical, it is reviewable, and it is required whatever the
answer turns out to be.

---

### Q2 — "What is a distributed monolith, and how do you know you have one?"

**A senior answer sounds like:** "It's when you've split into microservices but
they're still tightly coupled — you have to deploy them together, they share a
database, or there are synchronous call chains everywhere. You get the operational
complexity of microservices without the independence. You avoid it by defining
clear boundaries and using asynchronous communication."

Correct definition, correct causes, correct mitigation.

**A principal answer sounds like:** "The one-line test is: **two or more
deployables that must be deployed together to remain correct.** Everything else is
a symptom of that.

The symptoms I'd look for, roughly in the order they show up: a shared database —
which is sufficient on its own, no further evidence needed. A wiki page describing
the order in which two services must be deployed. A synchronous call chain in the
request path, so the user's latency is the sum and the availability is the
product. A rollback of one requiring a rollback of the other. And the tell that
usually comes first in practice: the p99 you were trying to improve is *worse*
than it was as a monolith, and the difference is the network.

What makes it worse than the monolith you started with is that you paid the entire
cost of distribution — partial failure, two connection pools against one database,
two warm-ups per deploy, traces instead of stack traces, a second on-call rotation
— and bought none of the benefit, because the units are still coupled. It is
strictly dominated by the thing it replaced.

The cause is almost always sequencing rather than ignorance. The code boundary is
visible and satisfying to work on; the data boundary is invisible and tedious. So
teams do the visible one first, ship it, and then find the invisible one was
load-bearing — at which point the cheap fixes are gone. Which is why my sequence
is: enforce the boundary in-process, cut the data, change the transaction, and only
then split the process. The process split should be the boring step. If it is the
exciting one, something earlier was skipped."

**What separates them:** the senior answer describes the condition. The principal
answer gives a **single diagnostic criterion** that resolves any argument about
whether you have one, ranks the symptoms by when you will actually meet them,
explains why it is worse than the starting point rather than merely disappointing,
and — most importantly — attributes the cause to sequencing rather than to a lack
of design skill. That last move is what makes it preventable.

**Adversarial follow-up:** *"We already have one. Two services, shared database,
coordinated deploys. What do you actually do on Monday?"*

The honest answer starts by refusing the obvious move. Merging them back is
usually not available — two teams, two repos, two rotations — so the work is
forward. On Monday: pick the *smallest* set of tables that one service can own
exclusively, and cut those, using the same crossing register. Not the whole
boundary; the smallest cut that removes one coordinated-deploy scenario. Then
measure whether coordinated deploys went down. That gives you an incremental path
with feedback, and it also tells you honestly whether the full cut is worth
funding. The alternative answer — and it should be on the table — is to recombine
into one deployable while keeping the module boundary, which is unpopular,
occasionally correct, and worth saying out loud so that nobody thinks the only
direction is forward.

---

### Q3 — "When would you tell a team not to split their monolith?"

**A senior answer sounds like:** "If the team is small, if the domains are tightly
coupled, or if they're doing it because microservices are fashionable rather than
for a concrete reason. Also if they don't have the operational maturity —
monitoring, tracing, CI — to run multiple services."

Good list, and the operational-maturity point is a real one that many candidates
miss.

**A principal answer sounds like:** "I'd want to hear the problem stated as a
number before I'd form a view at all, because most of these conversations are
about a solution rather than a problem. 'We can't deploy quickly' is not
actionable; 'deploy lead time is N hours, of which M is the test suite' is.

Given that, the four cases where I'd say no:

The real bottleneck is the pipeline, not the architecture. If deploys are slow
because the test suite takes 40 minutes and there's a manual staging sign-off,
splitting the service gives you two 40-minute test suites and two manual sign-offs.
Fix the pipeline. It's cheaper, it's reversible, and it's usually the actual answer
when the complaint is about speed.

One team owns both sides and will continue to. You've bought distributed-systems
failure modes to get an organisational property you already have, and the boundary
will erode because nothing rewards maintaining it. Conway's law works in both
directions.

There's a synchronous invariant the business won't give up. If 'an order is never
confirmed unless the money moved' is non-negotiable, a saga isn't a substitution —
it's a product change, and if the product owner says no, the split is off the table
for that operation. That's a legitimate stop.

And the decisive one: **we can't enforce the boundary in-process.** If we try to
separate the schemas and remove the joins inside the monolith and we can't, that's
not a reason to try harder with a network in the middle. It's information about
the domain, and the right response is to stop and take the module boundary as the
outcome.

The thing I'd add is that 'no' is rarely the whole answer. There's almost always a
cheaper intervention that gets most of what they wanted: enforce the module
boundary and stop; split the *pipeline* rather than the service; extract a smaller,
genuinely independent piece first to learn on. And if the real driver is a team
boundary — which is the most legitimate reason there is — that can often be
satisfied at the module or data step without ever splitting the process."

**What separates them:** the senior answer lists conditions. The principal answer
insists on a **measured problem statement before evaluating any solution**, gives
a falsifiable in-place test whose failure is a definitive stop, and — the part
that most distinguishes it — never leaves "no" as the whole answer. Offering the
cheaper intervention that delivers most of the value is what makes the "no"
acceptable rather than obstructive, and it is a large part of what Topic 134 is
about.

**Adversarial follow-up:** *"You've told them no. They do it anyway — the director
has already committed to it publicly. What now?"*

The honest answer does not sulk and does not sabotage. You have made the argument
once, in writing, clearly; it is on the record and that is what it is for. Now the
useful contribution is to make the version they are going to do as survivable as
possible: insist on the data cut before the process split, insist the transaction
change happens in-process behind a flag, insist on the reconciliation sweep, and
insist on a defined stopping point. Those four things are most of the risk, they
are compatible with the decision that has been made, and getting them costs the
director nothing. Being the person who fights the decision after it is made
spends credibility you will need for the next one — and it does not change the
outcome. Being the person who makes the chosen path safe is both more useful and,
in practice, more persuasive next time.

---

### Q4 — "How do you handle the transaction that spans the boundary?"

**A senior answer sounds like:** "You can't have a distributed transaction — two-
phase commit doesn't work well in practice. So you use a saga: each service does
its local transaction and publishes an event, and if a later step fails you run
compensating actions to undo the earlier ones. You need idempotent consumers
because messages can be redelivered, and the outbox pattern so the database write
and the event publish are atomic."

That is a good, complete answer at the pattern level.

**A principal answer sounds like:** "Three things, and the first one is that the
pattern is the easy part.

Before the saga: **do not put the remote call inside the transaction.** A remote
call inside a `@Transactional` method holds a connection from the pool for the
call's whole duration. We proved that in a drill — a two-second downstream call
inside a transaction exhausted HikariCP and took down every endpoint in the
service, including ones that touched no database. So whatever design we land on,
the remote call is outside the transaction boundary, which by itself means we no
longer have atomicity and need compensation regardless.

Then the saga, and the parts that actually bite in production rather than in the
diagram: the consumer must be idempotent keyed on something stable, because
at-least-once delivery means duplicates and a duplicate wallet debit is a customer
charged twice — that's the check-then-act failure from our concurrency work, with
a network in front of it. The outbox handles the write-then-publish window; we
proved that one with a `kill -9` between the commit and the publish, which left the
order existing and the event lost. And critically: **the compensation itself can
fail.** Release the inventory reservation — and if that throws? So there's a
retry, then a dead-letter queue, then an alert, then a documented manual
procedure with a named owner. A saga without that is a design that assumes the
happy path.

And the safety net that is not optional: a **reconciliation sweep** that finds
sagas stuck past a threshold, because any message can be lost. Its stuck-count is
an SLI with an alert. If the saga's correctness depends on every message arriving,
it is not a design, it is a hope.

The third thing is the one people leave out entirely: the customer-visible
semantics change. The order goes from confirmed-at-checkout to accepted-then-
confirmed. That's a product decision. I'd want the product owner to agree to the
sentence the customer will actually experience, before we build it — and I'd also
put 'don't split this particular operation' on the options list, because extracting
everything *except* the synchronous debit often gets most of the value with none of
the product risk."

**What separates them:** the senior answer knows the patterns. The principal
answer knows the patterns' **failure modes in production** — connection-pool
exhaustion from the call placement, duplicate charges from non-idempotent
consumers, the lost event window, and the compensation's own failure — and it
cites drills rather than doctrine. It also escalates the semantic change to the
product owner and keeps "don't do this part" on the options list. That last move
is the one that most reliably separates the two levels: a Principal engineer's
option set includes not doing it.

**Adversarial follow-up:** *"Your reconciliation sweep finds a stuck order. What
does it do — automatically resolve it, or page someone?"*

The honest answer is that it depends on the direction of the error and you should
decide that per saga, in advance. Where the automatic action is safe and
idempotent — re-emit the event, because the consumer is idempotent and a duplicate
is a no-op — automate it and alert on the *rate*, because a rising rate is a
signal even when each instance self-heals. Where the automatic action moves money
or releases stock and the outcome is genuinely unknown, do not automate it: mark
it, expose it in a queue, and page. And the number that matters for the alert is
not "is there a stuck order" but "how many, and is it growing", because a small
constant background rate is normal in any distributed system and paging on the
first one trains people to ignore the alert.

---

### Q5 — "The extraction is done. How do you know it worked?"

**A senior answer sounds like:** "The payments team can deploy independently, the
services don't share a database, and latency and error rates are within
acceptable bounds. I'd check the dashboards and confirm we haven't regressed."

Reasonable, and it names the right dimensions.

**A principal answer sounds like:** "I'd want to have written the answer down
*before* we started, because 'acceptable' after the fact means whatever we can
live with.

So: the problem statement had a number. If it was deploy lead time, the check is
deploy lead time for payments changes, now versus the baseline we recorded. If it
was change-failure rate, that. One number, decided in advance.

Then the things I predicted in the plan and can now falsify. I predicted the
order-placement path would gain an asynchronous hop and lose a synchronous write,
and I predicted a direction and a magnitude; I check it against the Topic 65
baseline. I predicted `GET /orders` would get slightly faster because it lost a
join and gained a denormalised column read. I predicted a total connection count
across two pools, and I check that against what the database could take before,
because the pool is the real ceiling and it is easy to lower it while believing
you raised it. And I predicted a memory footprint for the pair, which is not the
footprint of the one.

Then the failure modes I added, which are the ones that will actually decide
whether this was a good idea: the stuck-saga count and its trend, the duplicate-
detection rate in the idempotent consumer, the dead-letter depth on the
compensation path, and how many incidents in the first quarter required two people
from two teams. That last one is the distributed-monolith detector — if incidents
routinely need both rotations, the boundary is not real.

And the thing that is easy to miss: did the old in-monolith module actually get
deleted? If it is still there six months later with three call sites, we are
maintaining payments logic in two places and the migration is at 90% forever —
which is the same stall shape as any other migration, for the same reason: the
remaining work stopped hurting anyone."

**What separates them:** the senior answer evaluates against a standard chosen
afterwards. The principal answer **wrote falsifiable predictions before starting**
and checks them, distinguishes the metrics that measure the stated goal from the
metrics that detect the failure mode, and closes the loop on the forcing function —
because an extraction whose old path is never deleted has not finished, it has
merely stopped.

**Adversarial follow-up:** *"Your predictions were wrong. p99 on order placement
got 40% worse and the team says it's fine. Is it?"*

The honest answer separates two questions that are being run together. First: is
the new number acceptable against the SLO? That is the only question that matters
for the users, and it is answerable from the error budget rather than from
anyone's opinion. If the SLO is intact, "worse but fine" may genuinely be fine and
saying otherwise is engineering vanity. Second, and separately: **why was the
prediction wrong?** A 40% miss means the model of where the time goes is wrong,
and that model is what the next decision will be based on. So I would want the
diagnosis regardless of whether the number is acceptable — is it the extra hop, is
it a serialization cost nobody counted, is it the second warm-up, is it pool
contention? The cost of not knowing is that the next prediction is equally wrong
and the next decision is made on the same broken model.

---

## Mental model checkpoint

Reason these out in writing.

1. The mechanical statement says the data boundary is harder than the code
   boundary. Construct the strongest counterexample — a domain where the code
   boundary is genuinely the hard part. What is different about it, and does the
   general rule survive?

2. A lazy `@ManyToOne` is described as an invisible boundary crossing. Explain
   precisely why it is invisible, using what Topic 49 taught you about the proxy.
   Now: name two other Java or Spring constructs that hide a boundary crossing at
   the call site.

3. The plan puts the transaction change before the process split, in-process,
   behind a flag. State the cost of that ordering — there is one — and say why you
   would pay it anyway.

4. Splitting one service into two doubles the connection pools against one
   database. Work through what that does to the ceiling from Topic 109, and state
   the rule you would give a team about pool budgets across a split.

5. "Event-carried state transfer" resolves the `GET /orders` crossing by keeping a
   denormalised `payment_status` on `orders`. Enumerate everything that can go
   wrong with that column, and say which of those failures a customer would
   notice, which an operator would notice, and which nobody would notice.

6. The forcing function for deleting the old in-monolith module is the same shape
   as Topic 127's. Explain why the stall happens for the same reason in both
   cases, in terms of who is feeling which pain.

7. You are told the extraction must ship in six weeks instead of two quarters.
   Which steps do you drop, which do you keep, and what specifically do you tell
   the person asking about what they are now carrying? Now the harder version:
   which single step, if dropped, guarantees a distributed monolith?

---

## Quick reference card

### The five boundaries — say which one you are buying

| Boundary | Cost | Reversible? | Delivers |
|---|---|---|---|
| Module | Days | Yes | Comprehensibility, enforced separation |
| Data | Weeks–months | Mostly | Independent schema evolution |
| Transactional | Weeks + **a product decision** | With a flag, if in-process | Independent failure |
| Process | Days *if the above are done* | Expensively | Independent deploy, isolation |
| Team | Organisational | No | Autonomy — often achievable at Module |

### The distributed-monolith test

**Two or more deployables that must be deployed together to remain correct.**

Sufficient on its own: a shared database.

### The five resolutions for a crossing

| Resolution | Use when | Do not use when |
|---|---|---|
| Move it | It belongs on one side | — |
| API call | Low volume, needs live data | It is inside a loop |
| Batch API call | List rendering, needs live data | The batch size is unbounded |
| Event-carried state transfer | List rendering, staleness acceptable | The read must be strongly consistent |
| Denormalise at write | Single field, rarely changes | The field changes often |

### Sequence, and the one rule that matters most

1. Enforce the boundary in-process (architecture test, ratcheted baseline)
2. **Replace cross-boundary associations with identifiers** — makes crossings visible
3. Resolve read crossings
4. Separate the schema; drop cross-boundary FKs
5. **Transaction change — in-process, behind a flag** ← the rule
6. Split the process; route by flag
7. Migrate remaining call sites; deprecate in CI
8. Delete the old module — irreversible, after a soak

**The rule:** the riskiest change goes in the cheapest-to-revert environment.

### Java-specific things that will surprise you

1. **Component scanning is implicit** — the boundary exists only if the build
   enforces it (ArchUnit / Enforcer).
2. **Lazy associations are invisible crossings** — `order.getPayment()` reads like
   a field access.
3. **Your N+1 fix does not survive** — fetch joins are a database feature; the
   data is no longer in the database.
4. **`@Transactional` silently stops covering the other side**, and a remote call
   inside it exhausts the pool (Topic 55).
5. **Two pools against one database** — Postgres does a process per connection;
   the ceiling can go *down* (Topic 109).
6. **Two JVMs, two warm-ups** — a latency spike per deploy that did not exist
   before (Topic 74).

### Before you commit — the falsifiable test

Can you, inside the monolith:
- [ ] separate the schemas with no cross-boundary foreign keys?
- [ ] remove every cross-boundary join?
- [ ] route every call through one interface whose failure modes include timeout?
- [ ] break the shared transaction and pass the load test?

If any of these is **no**, extraction will not fix it. Stop, and take the module
boundary as the outcome.

---

## When would I use this at work?

**1. The week a "let's extract X into a service" proposal appears.**
It will appear framed as an architecture decision and it is mostly a data
decision. The highest-value thing you can do in that first week is produce the
crossing register — every join, FK, lazy association and shared transaction — and
put it in front of the room. It converts an unbounded commitment into a bounded
estimate, and about a third of the time it ends the proposal, correctly, by
showing that the boundary the team imagined does not exist in the data. Two weeks
of work that saves a quarter is the best trade available to you.

**2. When a system already has services and incidents keep needing two teams.**
That symptom is the distributed-monolith detector and almost nobody names it as
one, because each individual incident has a local explanation. Counting how many
incidents in a quarter required two rotations, and putting that number next to the
"independent services" claim in the architecture diagram, is a small piece of work
with a large effect on what the organisation believes about itself. Then the fix
is incremental: find the smallest set of tables one service can own exclusively,
and cut those.

**3. Any time a distributed transaction is proposed as a design detail.**
Sagas get drawn on whiteboards as three boxes and an arrow. The value you add is
insisting on the four things the whiteboard leaves out: idempotency keyed on
something stable, the outbox for the write-then-publish window, the compensation's
own failure path, and the reconciliation sweep with an alert on its rate. And
then the fifth thing, which is not technical at all: the customer-visible semantic
change, quoted in the words a customer would read, agreed by the person who owns
that decision. Doing that once, in public, changes how your organisation designs
the next five.

---

## Connected topics

**Prerequisites — the evidence and the machinery you will cite:**

- **Topic 03 — package-private as a module boundary**, and **Topic 20 — JPMS.**
  What Java gives you for enforcing a boundary, and why in practice it comes down
  to a build rule.
- **Topic 36 — component scanning.** Why the boundary is not written down
  anywhere by default, which is the whole reason step 1 exists.
- **Topic 49 — lazy associations.** The proxy that makes a boundary crossing look
  like a field access. This is the mechanic behind the plan's second step.
- **Topic 50 — N+1 detection and fixes.** The `GET /orders` endpoint is the
  canonical crossing, and the fix you learned there does not survive the split —
  which is the point.
- **Topics 54–55 — `@Transactional` and the connection pool.** The transaction
  that has to be replaced, and the reason the remote call must never be inside it.
- **Topic 65 — the load baseline.** Every prediction in section 6 is measured
  against this.
- **Topics 109 — pool sizing**, **111 — circuit breakers**, **115 — the outbox**,
  **116 — idempotent consumers**, **119 — tracing.** The extraction's new failure
  modes are exactly the machinery these topics built; the plan assembles them.
- **Topic 124 — the readiness review.** The new service needs its own, and the
  first draft of it belongs in this plan.
- **Topic 127 — migration planning I.** Same two properties (independently
  valuable, independently revertible), same stall shape, same forcing function.
  Read them together; this one is the architectural case of that one.

**This unlocks:**

- **129 — capacity and cost.** Section 6's predictions — the extra hop, the doubled
  pools, the two live sets, the two warm-ups — are computed there. A split changes
  the cost model in ways that are easy to get backwards.
- **130 — SLOs.** The stuck-saga count, the compensation dead-letter depth and the
  staleness bound on the denormalised column are all SLIs. A saga without an SLO
  on its safety net is unmonitored by construction.
- **131 — design docs.** This artefact is a design doc of a particular kind, and
  the reversibility section is the one 131 attacks hardest — with good reason,
  because step 7 is the weak point.
- **132 — engineering standards.** The architecture test in step 1, with its
  ratcheting baseline, is 132's machinery.
- **133 — postmortems.** The first duplicate charge, the first stuck saga and the
  first failed compensation are incidents, and how you write them up decides
  whether the second one happens.
- **134 — influence without authority.** Getting three teams to accept a boundary
  they did not ask for, and to migrate their call sites, is exactly that topic.
- **135 — the capstone.** "Walk me through a service extraction" is a standard
  principal prompt, and the follow-up is always "what did you do about the
  transaction?"

---

*Java baseline 21, running on JDK 25, Spring Boot 4.1 / Framework 7.0. The
mechanics in this topic — the invisible crossing created by a lazy association,
`@Transactional` silently not spanning the boundary, the pool arithmetic against a
single database, the three outcome shapes of a remote call — are properties of the
platform and are stable. The strangler pattern is a widely-used industry practice
named by Martin Fowler; nothing in this topic depends on a proprietary method.*
