# 130 — SLO Design and Error Budgets

## Phase: 12 — Principal Track
## Category: ELITE
## Java baseline: 21  |  Notes features from: 21/25
## Project spine: an SLO document for `orderflow` — the SLIs, the targets, the error-budget policy, and a written negotiation position for the product conversation

---

## Mechanical statement

An SLO is a **promise about a measured user experience, with a stated budget for
breaking it**.

Three parts, and all three must be present:

1. A **measurement** of something the user actually feels.
2. A **target** for that measurement over a stated window.
3. A **budget** — the amount of failure the target permits — and a **written rule
   about what changes when the budget runs out**.

If nothing changes when the budget is exhausted, it is not an SLO. It is a wish
written in the format of an SLO.

That is the whole mechanic. Everything else in this document is consequence.

The second half of the mechanic is less obvious and is where Principal-level
judgment lives: **the target is a purchase, not an aspiration**. Every extra nine
is bought with engineering time, on-call load, architectural constraint, and cash.
Somebody pays. Your job is to make the price visible *before* the number is agreed,
so that the person choosing the number is the person who can authorise the price.

---

## The bridge from what you know

### What transfers directly — and I am not going to re-teach it

You have run production services. You already know:

- What a percentile is, and that averages hide tails.
- That uptime is not binary.
- How to have a scoping conversation with a product manager, hold a position, and
  trade one thing for another.
- That "the dashboard is green and customers are complaining" is a real state a
  system can be in.

All of that transfers. Node or Java makes no difference to any of it. An SLI is
computed the same way over a Prometheus histogram whether the process is a JVM or
a Node event loop.

If you have written SLOs before in TypeScript-land, most of the mechanics carry
over unchanged. Skip ahead if that is you — but read "The decision, framed" and
the wrong-approach section, because those are where SLO documents actually die,
and they die for organisational reasons that have nothing to do with the runtime.

### What is genuinely harder at this level

Three things, and they are the reason this topic sits in Phase 12 rather than
Phase 11.

**One: making the decision legible.** A Senior engineer proposes a target. A
Principal engineer writes down *what the organisation is trading to get that
target*, *who is affected*, and *what evidence would make them change the number*.
The target is the easy half. The trade is the hard half.

**Two: making the budget cost something.** An error budget with no consequence is
decoration. Attaching a consequence means negotiating, in advance, a rule that
will one day stop a feature the product manager wants. That negotiation happens
best when the budget is full and everyone is calm. It happens worst during the
incident. Getting the policy signed while nothing is on fire is the actual skill.

**Three: separating "what we measure" from "what the user feels".** This is the
one that transfers least, because in most teams the measurement already exists and
nobody asks where it was taken from. Availability at the load balancer is easy to
measure and is not what the customer experiences. That gap is a judgment call, not
a tooling problem.

### The Java-specific part

There is one, and it is small but real. The evidence base for your SLIs is the
work you already did:

- **Topic 118** (Micrometer, RED metrics, histograms, cardinality traps) is where
  your SLI numerator and denominator come from. You cannot compute a p99-based SLI
  from a timer that only records a mean. You need a histogram with buckets chosen
  around your target, and you need to remember that **you cannot average p99s
  across pods** — you aggregate buckets and then compute the percentile.
- **Topic 119** (OpenTelemetry tracing) is where you find out *which* dependency
  spent the latency, which is what turns a breached SLO into a decision.
- **Topic 65** is your baseline. Your recorded p50/p95/p99 are the only honest
  starting point for a latency target. A target set without them is a guess.

So: the SLO judgment is language-neutral. The *evidence* is Java-shaped, and you
built it in Phase 11.

---

## What is this?

Four terms. Define them once and use them precisely, because sloppiness here is
the source of most bad SLO documents.

### SLI — Service Level Indicator

A **measurement** of one aspect of the service, expressed as a ratio:

```
SLI = good events / valid events
```

Both halves matter.

- **Good events** — the ones where the user got what they wanted, in a time they
  found acceptable. Note that time is part of "good". A request that succeeds after
  45 seconds is not good.
- **Valid events** — the ones this SLI is a promise about. Requests you deliberately
  rejected because the caller sent malformed input are usually not valid events for
  an availability SLI. Requests you rejected because your validation layer had a bug
  are absolutely valid events, and you must be able to tell the difference.

Writing the SLI as a ratio forces you to define both halves. That is the point. Most
bad SLIs are bad because nobody wrote down the denominator.

### SLO — Service Level Objective

A **target** for an SLI over a **window**.

```
99.9% of valid order-placement requests are good, over a rolling 28 days.
```

Target and window together. A target with no window is meaningless — 99.9% over a
year and 99.9% over an hour are wildly different promises.

### Error budget

The failure the SLO permits.

```
error budget = 1 − SLO target
```

At a 99.9% target, the budget is 0.1% of valid events. Expressed as events, that
is a countable number of failed requests. Expressed as time, it is a countable
number of minutes.

Here is the arithmetic — this is pure maths, not a benchmark, and you can verify
every cell:

| Target | Budget as a fraction | Allowed bad time per 28-day window | Per 30-day month | Per 365-day year |
|---|---|---|---|---|
| 99% | 1 in 100 | 6 h 43 m | 7 h 12 m | 3 d 15 h 36 m |
| 99.5% | 1 in 200 | 3 h 21 m | 3 h 36 m | 1 d 19 h 48 m |
| 99.9% | 1 in 1,000 | 40 m 19 s | 43 m 12 s | 8 h 45 m 36 s |
| 99.95% | 1 in 2,000 | 20 m 10 s | 21 m 36 s | 4 h 22 m 48 s |
| 99.99% | 1 in 10,000 | 4 m 2 s | 4 m 19 s | 52 m 34 s |
| 99.999% | 1 in 100,000 | 24 s | 26 s | 5 m 15 s |

Read the 99.99% row again. **Four minutes in a 28-day window.** A single pod
restart that takes 90 seconds to become ready has spent more than a third of a
month's budget. A rolling deploy that drops in-flight requests for 30 seconds has
spent an eighth. You cannot get there by being careful. You get there by changing
the architecture — and that is the conversation the number is really about.

### Error-budget policy

The **written rule about what changes when the budget is spent**. This is the part
that most teams skip, and skipping it is what turns the whole exercise into
theatre.

A policy has to answer four questions:

1. What happens at each burn threshold? (Not just at zero.)
2. Who decides? (A named role, not "the team".)
3. Who can override, and what does the override cost? (Usually: it is written down
   and reviewed.)
4. When does the budget reset? (Rolling window, or calendar reset.)

### The two other terms you will meet

- **SLA — Service Level Agreement.** An SLO with money or contractual consequence
  attached, offered to a customer. Your internal SLO should always be *stricter*
  than any SLA, so you find out you are in trouble before your customer does.
- **Critical user journey (CUJ).** A sequence of interactions that delivers value
  to a user — "browse a product, add to cart, place an order, get a confirmation".
  SLOs belong on journeys. This is developed further below and it is the single
  highest-leverage idea in the topic.

---

## Why does it matter?

Four reasons, in rough order of how often each one bites.

**1. Without a budget, every reliability argument is a shouting match.**

Product wants the feature. Engineering wants the refactor. Neither has a number, so
the argument is settled by seniority, volume, or whoever spoke to the VP most
recently. An error budget converts that argument into arithmetic: *we have spent
82% of this window's budget; the policy says at 75% we stop shipping
customer-visible change on this service until we are back under.* Now the
conversation is about whether to override a rule everyone agreed to, which is a
much better conversation than the one about who cares more.

**2. Without a budget, reliability work is unfundable.**

"We should make this more reliable" has no size and no stopping condition, so it
loses to every feature with a launch date. "We are burning budget at 3× the
sustainable rate, and the top contributor is the payment-callback path" has a size
and a stopping condition.

**3. Without a budget, you over-invest.**

This one is under-appreciated. An unspent error budget is also a signal — it means
you are more reliable than you promised, which means you may be spending
engineering time on reliability the user did not ask for and would happily have
traded for features. A Principal engineer says this out loud. It builds enormous
credibility, because everyone expects the reliability person to only ever ask for
more reliability.

**4. Because `orderflow` will be asked for a number, and the number will be wrong.**

Somebody — a customer, a sales conversation, a compliance questionnaire — will
ask "what is your uptime commitment?" If you have no considered answer, someone
will invent one, it will contain four nines because four nines sounds serious, and
you will inherit it. Your Topic 124 production-readiness review already listed
SLOs and current attainment as a required section. This topic is where you make
that section defensible.

---

## The decision, framed

Here is the decision, stated the way it actually arrives.

> Product wants to publish a reliability commitment for `orderflow` before the
> enterprise sales conversation next quarter. The proposed number is 99.99%. You
> have a production-shaped service, a recorded Topic 65 baseline, RED metrics from
> Topic 118, and end-to-end traces from Topic 119. You have never run a formal SLO.
>
> What do you say, and what do you write down?

The bad answers are both easy and both common.

- **"Yes, 99.99%."** You have just committed to four minutes of badness per 28
  days without knowing whether a rolling deploy costs you more than that.
- **"No, that's unrealistic."** You have made it a fight about your judgment
  against theirs, with no evidence on the table, and you will lose it or win it for
  the wrong reasons.

The Principal answer is a set of questions that reframe the decision, followed by a
document.

### The five questions that reframe it

**Q1: On what SLI?** "Uptime" is not measurable. Is the promise about order
placement succeeding? About the catalogue being browsable? About a payment
eventually reconciling? These have different failure modes and different costs and
they should not share a number.

**Q2: Measured where?** At the load balancer? At the edge/CDN? From a synthetic
client in the user's region? From the real client, reported back? Each of these
excludes a different set of real failures. The further from the user, the cheaper
to measure and the less it means.

**Q3: Over what window?** A rolling 28-day window has no reset ceremony and no
end-of-month gaming. A calendar month aligns with reporting and resets the budget
on the first, which some organisations prefer for exactly that reason. Pick one and
say why.

**Q4: What are we trading?** This is the question that makes the decision legible.
Name the specific things the extra nine requires. For `orderflow` that is likely to
include: a second region with data replication and a tested failover; a deploy
process that provably drops zero requests (you built this in Topic 123 — do you
have the evidence?); an on-call rotation with a response-time commitment that
implies a certain headcount; and dependency SLOs from Postgres, Kafka, and the
payment gateway that are at least as strong as the number you are promising.

**Q5: What is the revenue at risk?** Product owns this number and you do not. Ask
for it. "What does an hour of failed order placement cost us, and what does the
enterprise deal we are chasing require contractually?" If an hour of downtime costs
less than the annual cost of the second region, 99.99% is a bad business decision
and you should say so with the arithmetic in front of you.

### The dependency ceiling — the argument that ends the conversation

This is the single most useful piece of arithmetic in the topic.

If your request path serially depends on N components, and each is independently
available with probability `a_i`, your ceiling is the product:

```
ceiling = a_1 × a_2 × ... × a_N
```

Three dependencies at 99.9% each gives roughly 99.7%. You cannot promise 99.99% on
top of that by trying harder. You get there only by removing a dependency from the
critical path, adding redundancy so a single failure is not fatal, or degrading
gracefully so the dependency's failure is not the user's failure.

**Fill this in from your own `orderflow` architecture and your providers' published
figures. Do not copy numbers from anywhere, including from me.**

| Dependency on the order-placement path | Published or observed availability | Is it serial (a failure fails the request)? | Can it be made non-serial? How? |
|---|---|---|---|
| Postgres primary | | | |
| HikariCP pool (Topic 109 sizing) | | | |
| Kafka (outbox relay, Topic 115) | | | |
| Payment gateway | | | |
| Redis cache (Topic 110) | | | |
| Kubernetes ingress / LB | | | |
| **Serial product (your ceiling)** | | | |

The last row is the sentence that ends the 99.99% conversation, and it ends it with
arithmetic rather than with your opinion. That is the difference between Senior and
Principal in this topic, stated as concretely as I can state it.

Note the third column carefully. Kafka being down does not have to fail order
placement — with a transactional outbox (Topic 115) the order commits and the event
publishes later. That moves Kafka off the serial path for the *availability* SLI and
onto a *freshness* SLI instead. Designing dependencies off the critical path is how
you buy nines without buying a region.

---

## Example 1 — a minimal illustration

Strip everything away. One endpoint, one SLI, one budget, one policy line.

### The service

A single endpoint that returns a product's current price.

### The SLI, written as a ratio

```
good events  = responses with HTTP status < 500 AND served in under 200 ms
valid events = all requests to GET /products/{sku}/price
               that reached the service
```

Three things to notice, because each is a decision:

1. **Latency is inside the definition of "good".** A 200 that took nine seconds is
   not good. Folding latency into the availability SLI stops the classic failure
   where availability is 100% and the product is unusable.
2. **Status < 500, not status == 200.** A 404 for a SKU that does not exist is the
   service working correctly. It is a valid event and a good event.
3. **"that reached the service"** is an honest admission that this SLI cannot see
   requests that failed before arriving. That limitation is written down rather than
   ignored. Writing down what your measurement cannot see is a large part of what
   makes a document trustworthy.

### The SLO

```
99.9% of valid events are good, over a rolling 28-day window.
```

### The error budget

0.1% of valid events. If the endpoint serves R requests in the window, the budget
is `R / 1000` bad events.

Notice that this is a *request-based* budget, not a time-based one. Request-based
is usually the better choice for a request/response service, because ten minutes of
badness at 3 a.m. costs far fewer users than ten minutes at peak. Time-based budgets
treat those as identical, which does not match anyone's intuition about harm.

**Fill in from your own traffic shape.**

| Quantity | Value from my system | Where I got it |
|---|---|---|
| Requests to this endpoint per 28 days | | |
| Error budget in requests (`requests / 1000`) | | |
| Bad events in the last completed window | | |
| Budget consumed (%) | | |

### The policy — one line, and it has teeth

```
While budget consumed > 100%, no change to this endpoint's request path
ships except a change whose stated purpose is to reduce the burn.
Decider: the service owner. Override: the engineering manager, in writing,
recorded in the SLO document's override log.
```

That is a complete, if tiny, SLO. It has a measurement tied to user experience, a
target, a window, a countable budget, a consequence, a named decider, and a named
override path with a cost (it gets written down).

### Now break it, in one sentence

Change the "good" definition from `status < 500 AND under 200 ms` to just
`status < 500`, and the whole thing becomes worthless the first time a downstream
dependency gets slow instead of failing. Latency degradation is the most common
real-world failure shape for a healthy-looking service, and an SLI that cannot see
it is measuring the wrong thing. You met exactly this in Topic 55: a two-second HTTP
call inside a transaction exhausts HikariCP and makes every endpoint slow. Status
codes stayed fine for a while. The users did not.

---

## Example 2 — the real decision on the project spine

Now the actual artefact. This is `orderflow`, and it is more complicated in the
ways that matter.

### Step 1 — Find the critical user journeys, not the endpoints

`orderflow` has dozens of endpoints. It has a small number of journeys. Do not
write an SLO per endpoint. You will end up with forty SLOs, six of which are
permanently red, and everybody will stop looking at the dashboard.

The journeys, stated as a user would state them:

| # | Critical user journey | The user's failure experience | Endpoints involved |
|---|---|---|---|
| CUJ-1 | "I can browse products and see accurate stock" | Empty or wrong catalogue; can't decide what to buy | `GET /products`, `GET /products/{sku}` |
| CUJ-2 | "I can place an order and it succeeds or clearly fails" | Spinner, then nothing; or a charge with no order | `POST /orders` (inventory reserve + wallet debit + payment) |
| CUJ-3 | "My order status is correct within a short time of paying" | Paid, but the order still says PENDING an hour later | payment callback → outbox → consumer → status update |
| CUJ-4 | "My wallet balance is right" | Money missing; double charge | wallet ledger consistency |

Four journeys. Four SLOs, at most one or two SLIs each. That is a document a human
being will read.

CUJ-2 is the one that matters most and the one everything else is negotiated
against. Say so explicitly in the document; ranking the journeys is itself a
decision and it should be legible.

### Step 2 — Choose the SLI type per journey

There are four SLI shapes and picking the right one is most of the work.

| Shape | Measures | Right for |
|---|---|---|
| **Availability** | fraction of requests that succeeded | CUJ-1, CUJ-2 |
| **Latency** | fraction of requests served faster than a threshold | CUJ-1, CUJ-2 |
| **Freshness** | fraction of items processed within a time bound of the event | CUJ-3 |
| **Correctness** | fraction of records that agree with an independent check | CUJ-4 |

CUJ-3 and CUJ-4 are the ones Senior engineers miss, and they are the ones where
`orderflow`'s real risk lives. You built an outbox in Topic 115 precisely because a
`kill -9` between the database commit and the Kafka publish loses the event. The
outbox closes that window — but the relay can fall behind, and "fallen behind" is
invisible to any availability SLI. It needs a freshness SLI with its own budget.

Correctness is stranger still and worth dwelling on. A correctness SLI needs an
**independent check** — something that recomputes the answer a different way and
compares. For wallets, that is a reconciliation job that sums the ledger entries
and compares against the stored balance. It is a real piece of engineering, and the
SLO document is where you make the case that it is worth building. The Topic 92
drill (check-then-act on a `ConcurrentHashMap` idempotency cache producing a double
wallet charge) is exactly the failure this SLI catches and no other SLI can.

### Step 3 — Write the SLI specifications precisely

Here is CUJ-2, the one that matters, written to the standard I expect.

```
SLI-2a  Order placement availability

  Good events:
    HTTP responses to POST /orders where
      status is 2xx, OR
      status is 4xx AND the rejection is attributable to the caller
        (schema validation failure, unauthenticated, unknown SKU,
         insufficient wallet balance, insufficient stock)

  Valid events:
    All POST /orders requests recorded at the edge proxy,
    including requests that timed out client-side with no response,
    EXCLUDING requests from the synthetic load generator
    (identified by the X-Orderflow-Synthetic header).

  Explicitly counted as BAD:
    - 5xx of any kind
    - 4xx caused by our own validation defect
      (tracked separately; see the note on attribution below)
    - any response later than the latency threshold in SLI-2b
    - a request that returned success but produced no order row
      (detected by the reconciliation check, CUJ-4's mechanism)

  Measured at: the edge proxy, joined with client-reported outcomes
               from the web and mobile clients.
  Excluded from measurement, knowingly: failures before the request
               reaches the edge (DNS, client network, ISP).
               This limitation is accepted and stated.
```

```
SLI-2b  Order placement latency

  Good events:  POST /orders responses served in <= T_place milliseconds
  Valid events: same denominator as SLI-2a
  Measured at:  the edge proxy (server-side duration), with a
                client-side comparison sampled at 1% for drift detection
```

Two things to defend in review here.

**The 4xx attribution problem.** "4xx is the caller's fault" is true right up until
your own validation has a bug and starts rejecting valid orders with a 400. Then
your SLI says everything is fine while orders fail. You need an attribution
mechanism: a distinct error code per rejection reason, tracked as its own time
series, with an alert on any rejection reason whose rate changes sharply. Note the
Topic 118 constraint — the error *reason* is a bounded-cardinality tag and is safe;
the order ID is not and will kill your Prometheus. Do not solve this problem by
tagging the order ID.

**The "success with no order row" case.** This is the nastiest failure and it is
invisible to HTTP-level measurement. It is why the availability SLI has a
correctness clause bolted onto it. You will only detect it by comparing against an
independent source. Whether that is worth building is a real decision and belongs in
the document with its cost.

### Step 4 — Set the targets from your own baseline

**This is the table you fill in. I am not giving you numbers, because numbers I
invent would be worse than useless — they would be a fake anchor for a real
negotiation.**

Your inputs are: the Topic 65 recorded baseline (p50/p95/p99 per endpoint under
load), your Topic 118 RED metrics in production, and your dependency ceiling table
from earlier in this document.

| SLI | Current measured attainment (last 28 d) | Dependency ceiling | Proposed target | Budget in events/minutes | Confidence that we can hold it |
|---|---|---|---|---|---|
| SLI-1a catalogue availability | | | | | |
| SLI-1b catalogue latency (threshold: ___ ms) | | | | | |
| SLI-2a order placement availability | | | | | |
| SLI-2b order placement latency (threshold: ___ ms) | | | | | |
| SLI-3 payment status freshness (bound: ___ minutes) | | | | | |
| SLI-4 wallet correctness (reconciliation window: ___) | | | | | |

Rules for filling it in, which are the actual content of this step:

1. **Never propose a target above the dependency ceiling** without also naming the
   architectural change that raises the ceiling. If the ceiling is 99.7% and you
   propose 99.9%, the document must contain the paragraph explaining how the
   ceiling moves. Otherwise you have promised something arithmetic forbids.

2. **Set the latency threshold from the baseline, not from a round number.** If your
   Topic 65 p95 for `POST /orders` under target load is some measured value, a
   threshold at or slightly above that is defensible and a threshold at "200 ms
   because that sounds fast" is not. Say which percentile of your current
   distribution the threshold sits at, because that tells the reader immediately how
   much slack there is.

3. **Start with an achievable target, and write the aspirational one next to it with
   its price.** Two columns, not one. "Today: X. Aspiration: Y. What Y costs: a
   second region, a tested failover drill quarterly, an on-call rota of N people,
   and moving the payment gateway off the serial path." This is the single most
   valuable paragraph in the whole document, because it is the one that makes the
   trade legible to a non-engineer.

4. **The last column is not optional.** If you would not bet on holding the target
   for three consecutive windows, say so now. Proposing a target you privately
   expect to miss is how engineers lose credibility slowly.

### Step 5 — Write the error-budget policy

This is the part with teeth. Graduated, not binary, because a binary policy is
either ignored or catastrophic.

```
ERROR BUDGET POLICY — orderflow

Scope: SLI-2a and SLI-2b (order placement). Other SLIs are reported but
do not gate delivery in this version of the policy. Reviewed quarterly.

Window: rolling 28 days.

Thresholds and consequences:

  Budget remaining > 50%
    Normal operation. Ship freely. If the budget is consistently
    untouched for three windows, the target is too loose — bring a
    proposal to raise it or to spend the slack on velocity.

  Budget remaining 25%-50%
    Every change to the order-placement request path requires a named
    reviewer from the on-call rotation, in addition to normal review.
    Reliability items enter the next sprint at the top.

  Budget remaining 0%-25%
    Feature work on the order-placement path pauses. The team's default
    work is burn reduction. Changes off that path continue normally.
    A written burn analysis is produced within two working days,
    naming the top three contributors by budget spent.

  Budget exhausted (< 0%)
    No customer-visible change ships to orderflow until the trailing
    7-day burn rate is back under 1x sustainable.
    Exceptions: security patches, incident mitigations, and changes
    whose stated purpose is burn reduction.

Decider: the orderflow service owner.
Override: the Director of Engineering, in writing, in the override log
          in this document, with a stated expiry date. Overrides are
          reviewed at the quarterly SLO review.
Reset: none. The window rolls. There is no "new month, fresh start".
```

Every clause there is doing work. Let me be explicit about three of them, because
they are the ones reviewers challenge.

- **"Changes off that path continue normally."** Without this, the first exhausted
  budget freezes the whole team, everyone hates the policy, and it is repealed. Scope
  the consequence to the thing that is failing. A policy that survives contact with
  reality is worth more than a strict one that gets deleted.
- **"If the budget is consistently untouched, the target is too loose."** This clause
  buys you enormous political capital and costs you nothing. It shows the policy is
  a control loop, not a ratchet in one direction. It is also the clause that makes
  product managers trust the document.
- **The override log with an expiry date.** Overrides are legitimate; business
  reality exists. Untracked overrides are what turn a policy into a formality. The
  cost of an override is that it is visible and it expires.

### Step 6 — Alerting on burn rate, not on threshold

A quick but load-bearing point, and one of the few places where widely-used practice
has a specific shape worth naming.

Do not alert when the SLI dips. Alert on **burn rate** — how fast the budget is
being consumed relative to the rate that would exactly exhaust it over the window.
Burn rate 1× means you will finish the window with exactly zero budget left. Burn
rate 14× means you will exhaust a 28-day budget in two days.

The widely-used practice (associated with Google's SRE material, and adopted
broadly since) is **multi-window, multi-burn-rate** alerting: a fast-burn condition
evaluated over a short window that pages a human, and a slow-burn condition over a
long window that opens a ticket. Two windows per condition — a long one for
sensitivity and a short one to confirm the burn is still happening — so the alert
resolves when the problem stops rather than continuing to fire on old data.

I am deliberately not giving you the specific window and multiplier pairs as though
they were canonical constants. They depend on your traffic volume and your
tolerance, and quoting numbers as authoritative would be exactly the fabrication
this curriculum forbids. Derive them:

| Condition | Budget consumed to trigger | Long window | Short window | Action | Who is woken |
|---|---|---|---|---|---|
| Fast burn | | | | Page | |
| Medium burn | | | | Page (business hours) | |
| Slow burn | | | | Ticket | |

The derivation: pick the fraction of budget you are willing to lose before someone
is woken. A long window of `W` and a burn rate of `B` consumes `B × W / window_length`
of the budget before firing. Solve for the `B` that matches your chosen fraction. Write
that arithmetic in the document so the next person can re-derive it when traffic
changes.

One warning specific to what you learned in Topic 118: burn-rate alerts are computed
from histogram buckets. If your buckets are badly placed relative to your latency
threshold, your latency SLI is quantised to whatever bucket boundary is nearest and
your alert fires at the wrong time. Choose bucket boundaries *around your SLO
threshold* before you publish the SLO. This is a real, concrete, Java-side task that
falls out of the SLO decision, and it is the kind of detail that makes a reviewer
believe the rest of your document.

### Step 7 — Write the negotiation position

This is a separate section of the artefact, and it is the one that most engineers
never write. It is a page you take into the room. It is not the SLO document; it is
your position on the SLO document.

```
NEGOTIATION POSITION — orderflow reliability commitment

What I am asking for:
  Adopt 99.9% on SLI-2a with the graduated budget policy as written.
  Publish nothing externally above that number this quarter.

What I will trade:
  - I will accept a tighter latency threshold on SLI-2b than my
    baseline comfortably supports, in exchange for the budget policy
    being signed as written. Latency I can improve with engineering
    inside this quarter. The policy I cannot create later.
  - I will drop the correctness SLI (CUJ-4) from the gating set this
    quarter and report it informationally, if that is what it takes
    to get CUJ-2's policy agreed.

What I will not trade:
  - A published external number above our measured dependency ceiling.
    That is a promise we can only keep by luck.
  - The absence of an override log. An override is fine. An invisible
    override is not.

The counter-argument I expect, and my answer:
  "The enterprise deal requires 99.99%."
  Answer: then the deal requires the second region, the tested
  failover, and the on-call headcount. Here is the cost. Here is the
  revenue at risk if we promise it and miss. I am not saying no; I am
  saying this is a funded project, not a number in a document. If the
  deal pays for it, I will build it and I will tell you honestly when
  it is real.

What would change my mind:
  - Evidence that our dependency ceiling is higher than I have
    calculated. Bring me the providers' actual measured numbers.
  - A degradation design that removes the payment gateway from the
    serial path, which raises the ceiling without a second region.
  - Revenue-at-risk arithmetic showing an hour of downtime costs more
    than the second region costs per year.

Who is affected by this decision:
  - On-call engineers: the target sets their page volume and their
    response-time expectation.
  - The orders team: the budget policy can pause their feature work.
  - Sales: the published number is a commitment they will be held to.
  - Support: the freshness SLI determines how long "where is my order"
    tickets take to become legitimate.
```

Read the "what would change my mind" block again. That block is the difference
between an engineer with an opinion and a Principal engineer. You are stating, in
advance and in writing, the evidence that would move you. It makes you extremely
hard to argue with, because you have already conceded that you could be wrong and
named the exact conditions. And it makes the eventual decision legible to everyone
who reads the document six months later.

---

## Wrong approach → exact symptom → root cause → fix

Five failures. The symptoms here are **organisational** — things you can observe in
a room, a ticket queue, or a calendar — not stack traces. They are still concrete.

---

### Wrong approach 1 — Measure the SLI where it is easy, not where the user feels it

**What it looks like:** Availability is computed from the load balancer's own
access logs. `5xx count / total count`. It is one query, it already exists, it is
free.

**Exact symptom:**

- During a live incident, the SLO dashboard shows 99.98% for the day while the
  support queue takes 340 "I can't check out" tickets in ninety minutes.
- In the incident review, someone says the sentence that gives it away: *"our
  monitoring didn't catch it — the customer told us."*
- The SLO's monthly report and the support ticket volume have visibly uncorrelated
  shapes when you plot them together. If your reliability metric does not correlate
  with your complaint volume, the metric is measuring something other than the
  user's experience.

**Root cause:** The measurement point excludes the failure domain. A load balancer
cannot see: requests that never arrived (DNS, TLS handshake failure, the client's
network); requests that timed out on the client while the server was still happily
working on them and eventually returned a 200 into a closed socket; responses that
were structurally successful but semantically wrong (a 200 with an empty product
list because the inventory call silently failed and the code fell back to an empty
collection). Every one of those is a real, common `orderflow` failure and every one
of them is invisible at the load balancer.

There is a general form of this mistake and it is worth naming: **the measurement
was chosen for its availability, not for its meaning.** The easy measurement always
exists first, so it always wins by default unless someone deliberately intervenes.

**Fix:**

- Fold a **latency threshold** into the "good" definition. This single change
  catches the "slow enough to be broken" class immediately and costs nothing beyond
  a histogram you already have from Topic 118.
- Fold a **semantic check** in where the failure is silent. If an empty product list
  is a plausible failure, an empty product list on a request that should have
  matched is a bad event, and the application must emit that distinction as a
  metric. That is application work, and the SLO document is where you justify it.
- Add **client-side reporting** for the journeys that matter. Even sampled at a low
  rate, it tells you the size of the gap between what the server thinks and what the
  user got. You do not need every request; you need enough to know whether the gap
  is 0.1% or 4%.
- Add a **synthetic probe** that runs the whole CUJ-2 journey from outside your
  network on a schedule. It has low volume, so it is a poor SLI on its own, but it
  is an excellent detector of the "nothing reached us, so our metrics look perfect"
  failure — the one where the SLI is 100% because the denominator collapsed.
- In the document, **write down what the measurement cannot see.** Stating the
  limitation is what makes the number trustworthy.

---

### Wrong approach 2 — Adopt 99.99% without costing the on-call load and multi-region work it implies

**What it looks like:** The number appears in a slide, then in a document, then in a
sales deck. Nobody ran the arithmetic. It was picked because it sounds like the
level of seriousness the organisation wants to project.

**Exact symptom:** A specific and very recognisable sequence.

- Week 1: the target is announced. No engineering plan changes. No rota changes. No
  budget is allocated. This is the tell: **a real target change always has a visible
  cost somewhere, and there is no cost anywhere.**
- Week 3: the first routine rolling deploy consumes a large fraction of the window's
  budget, because a 90-second unready pod is a meaningful fraction of four minutes.
- Week 5: the SLO is missed. Nothing happens, because there is no policy.
- Week 9: it is missed again. The dashboard is now decoration. Someone quietly
  removes it from the weekly review agenda because it is always red and there is
  nothing to say about it.
- Month 6: a customer cites the number in an escalation, and now it is a commercial
  problem rather than an engineering one.

**Root cause:** The target was treated as an aspiration rather than as a purchase.
Nobody translated it into its implications, so nobody could refuse to pay for it —
and a price nobody was asked to pay is a price nobody paid. The deeper cause is that
the person choosing the number was not the person who would bear the cost, and the
document did not force those to meet.

**Fix:**

- **Cost the target before adopting it.** Produce this table and take it into the
  room. Fill it from your own environment.

  | Implication of the proposed target | Do we have it today? | Cost to get it | Who pays |
  |---|---|---|---|
  | Deploys that provably drop zero requests (Topic 123) | | | |
  | Pod startup fast enough that a restart is not a budget event (Topic 122) | | | |
  | Second region with tested failover | | | |
  | Dependency SLOs at or above the target | | | |
  | On-call rota with a response commitment implying N engineers | | | |
  | Burn-rate alerting that can page within the fast-burn window | | | |
  | Reconciliation for silent-failure detection | | | |

- **Propose the achievable number with the aspirational one beside it, priced.** You
  are not refusing. You are pricing. Those land completely differently in the room,
  and the second one keeps you in the conversation.
- **Show the four-minutes-per-28-days arithmetic out loud.** Non-engineers almost
  never have "99.99%" and "four minutes a month" connected in their heads. Making
  that connection in a meeting is often the entire intervention — I have seen the
  number revise itself the moment the minutes are said aloud.

---

### Wrong approach 3 — An SLO with no error-budget policy

**What it looks like:** A well-written SLO document. Good SLIs, sensible targets,
a real dashboard. No section describing what happens when the budget is spent, or
one that says "we will prioritise reliability work" without a threshold, a decider,
or a consequence.

**Exact symptom:**

- The budget goes to zero. There is a Slack thread. Everyone agrees it is bad.
- **The next sprint's plan is byte-identical to what it would have been.** This is
  the observable symptom, and it is unambiguous: compare the sprint plan to the one
  drafted before the budget was exhausted. If nothing moved, the SLO changed nothing.
- Three windows later, someone asks in a planning meeting "are we still doing the
  SLO thing?" and nobody has a confident answer.
- When you do try to invoke it, the argument is about whether *this particular*
  breach warrants slowing down. That argument is unwinnable, because it is being had
  under deadline pressure with a specific feature at stake and a specific person who
  wants it.

**Root cause:** The policy was never negotiated, so it is being negotiated during
the crisis. Negotiating a general rule while a specific painful case is on the table
is the worst possible time — everyone is arguing about the case, not the rule, and
the loudest advocate for the case wins. The general form: **a rule agreed in the
abstract binds; a rule proposed in the specific is just an opinion about that
specific thing.**

**Fix:**

- **Get the policy signed while the budget is full.** This is the single most
  important sentence in this document. Sign it when nothing is on fire and the
  discussion is cheap.
- **Make it graduated,** so the first consequence is mild (an extra reviewer) and the
  team learns the policy exists before it ever bites hard.
- **Name a decider and an override path.** Policies without a named human are
  policies that dissolve. An override path is not a weakness — it is what makes the
  policy survivable, and therefore what makes it real.
- **Scope the consequence to the failing path,** so the policy is proportionate and
  does not create an incentive to sabotage it.
- **Rehearse it once.** Run a tabletop: "assume we are at 10% budget remaining
  tomorrow — what stops, who says so, what do we tell the product manager?" Ten
  minutes in a team meeting, and it converts the policy from text into something the
  team has actually done once.

---

### Wrong approach 4 — Forty SLOs, one per endpoint

**What it looks like:** Somebody automated it. Every endpoint in `orderflow` got an
availability SLO and a latency SLO from a template. The dashboard is a wall of
sparklines. It looks impressively rigorous.

**Exact symptom:**

- Six SLOs are permanently in breach, every window, and there is a shared
  understanding that "those ones don't count".
- When you ask any engineer on the team "are we meeting our SLOs?", the answer is a
  question: "which ones?" Nobody can say the number from memory. Contrast that with a
  four-journey document, where everyone knows the answer.
- Nobody can tell you which of the forty would actually hurt a customer, so an
  incident triage cannot use the SLOs to prioritise — which was the point of having
  them.
- The permanently-red ones are a low-grade poison: they teach the whole team that a
  red reliability indicator is normal and ignorable. That habit then applies to the
  ones that matter.

**Root cause:** SLOs were generated from the system's structure (endpoints) rather
than from the user's experience (journeys). The system's structure is easy to
enumerate and the user's experience requires judgment, so the enumerable thing won
by default. This is the same underlying error as Wrong Approach 1, wearing a
different costume: **the easy framing displaced the meaningful one.**

**Fix:**

- **One SLO set per critical user journey.** Three to five journeys for a service the
  size of `orderflow`. If you cannot name the journeys, that is the finding — go and
  find out what users actually do before writing any targets.
- **Rank the journeys** and say which one wins when they conflict. That ranking is a
  decision and should be visible.
- Keep endpoint-level metrics — they are your diagnostic layer and Topic 118 already
  gave you them. **They are just not SLOs.** The distinction is: an SLI has a budget
  and a consequence; a metric is evidence you consult during an investigation.
- Apply a hard test to every proposed SLO: *if this is breached and nothing else is,
  can I describe the customer's experience in one sentence?* If not, it is a metric,
  not an SLI.

---

### Wrong approach 5 — "Good" defined only by HTTP status, so silent wrongness is invisible

**What it looks like:** The SLI is `status < 500`. Clean, standard, and it is what
every dashboard template gives you.

**Exact symptom:**

- An incident where the availability SLI stayed above target for its entire
  duration. The Topic 51 drill is exactly this shape: a native SQL write behind an
  L2 cache serves a stale price. Every response is a 200. Every response is wrong.
- The postmortem contains the sentence *"the SLO was not breached during the
  incident"*, and everyone finds it uncomfortable and moves on without changing
  anything.
- Customer-reported issues systematically outnumber SLO-detected issues. Track the
  ratio for a quarter — if most of your real incidents were reported by users rather
  than detected by your SLIs, your "good" definition is too weak. This is a concrete,
  countable, monthly-reportable symptom.

**Root cause:** "Good" was defined by the transport layer because the transport
layer is where the easy signal is. Correctness is a property of the domain, not of
HTTP, and no framework will hand it to you.

`orderflow` has at least four failure modes that return a perfectly healthy status
code, and you have already produced every one of them in a drill:

- Stale price served from the L2 cache after a native-SQL write (Topic 51).
- A double wallet debit from a check-then-act on the idempotency cache (Topic 92).
- An order committed with its event lost, so downstream never learns about it —
  the failure the outbox exists to prevent (Topic 115).
- A rolled-back value served from the cache because `@Cacheable` sat outside
  `@Transactional` (Topic 41).

**Fix:**

- Add a **correctness SLI with an independent check** for the highest-value
  invariant. For `orderflow` that is the wallet ledger: a job that recomputes the
  balance from the ledger and compares. Its SLI is `records that agree / records
  checked`. This is real engineering and the SLO document is the right place to
  justify funding it.
- Add a **freshness SLI** for anything asynchronous. Outbox relay lag and consumer
  lag both belong here. "Order status is correct within N minutes of payment" is a
  promise a user understands, and it is the only SLI that can see a stalled relay.
- Where a **fallback** exists, emit a metric when the fallback fires and count the
  fallback as a degraded event. A circuit breaker opening (Topic 111) is a graceful
  degradation for the system and a bad event for the user. If your SLI cannot see an
  open breaker, it cannot see your most likely failure mode.
- Apply this test to every SLI in the document: **name a concrete way the service
  could be badly broken while this SLI reads 100%.** Every SLI has such a way. Write
  it down. If you cannot think of one, you have not understood the SLI yet.

---

## Artefact — what you must produce

Write an **SLO document for `orderflow`**. This is the deliverable for Topic 130
and it is one of the artefacts you will defend in Topic 135.

### Format and length

- **6 to 12 pages**, or roughly 1,500 to 3,000 words. Long enough to be complete,
  short enough that a product director will read it end to end. If it is 30 pages,
  nobody will read it and you have failed the actual objective.
- Written for a mixed audience: engineers, a product manager, an engineering
  director. Assume the product manager reads only the summary, the targets table,
  and the policy. Write those three sections so that reading only them is enough.
- One page of it must be the **negotiation position**, written for you and not for
  publication.

### Required sections

**1. Summary — half a page, no more.**
What is being promised, to whom, at what number, and what changes when it is
missed. If someone reads only this, they should be able to repeat the commitment
accurately.

**2. Critical user journeys.**
Three to five, named the way a user would describe them, with the failure
experience stated in the user's terms. Ranked, with the ranking justified in one
sentence.

**3. SLI specifications.**
For each journey, one or two SLIs, each with: the good-event definition, the
valid-event definition, the measurement point, and an explicit statement of what
this measurement cannot see. Include the exact query or the metric names it is
computed from — see the review section on why this is mandatory.

**4. Dependency ceiling.**
The table from earlier, filled in, with the serial product computed and the
conclusion stated in one sentence.

**5. Targets.**
The table with your current measured attainment, the ceiling, the proposed target,
the budget in events and in minutes, and your confidence. Two target columns:
achievable-now and aspirational, with the aspirational one priced.

**6. Error-budget policy.**
Graduated thresholds, consequences scoped to the failing path, a named decider, a
named override authority, an override log (empty at first, with the format defined),
and the reset rule.

**7. Alerting.**
Your burn-rate conditions with the arithmetic that produced them. Include the
histogram bucket boundaries you need in Micrometer and note whether you currently
have them — because if you do not, your latency SLI is not computable and the
document is describing a system that does not exist yet.

**8. What we are not promising.**
An explicit list. Things deliberately outside the SLO, with the reason. This section
does more for your credibility than any other, because it proves you understand the
boundaries of your own claim.

**9. Review cadence and owner.**
When this is revisited, by whom, and what evidence triggers an out-of-cycle review.

**10. Negotiation position.** (Separate page.)
What you are asking for. What you will trade. What you will not trade. The
counter-argument you expect and your answer. What would change your mind. Who is
affected.

### Required content — the specific things that must appear

- At least one **freshness** SLI and at least one **correctness** SLI. If you only
  write availability and latency SLIs, you have written the easy half.
- The **dependency ceiling arithmetic**, computed, with the product shown.
- At least one place where you say **"we are deliberately not measuring this"** and
  give the reason.
- At least one place where you **decline** something product might want, with the
  price of the thing you declined.
- A **named human** in the decider and override roles. Not a team. A role held by a
  person.
- An explicit statement of **what evidence would make you raise or lower each
  target.**

---

## How I will review it

I will not comment on your prose. I will attack the weakest assumption.

### The three questions that usually break an SLO document

These three break more SLO documents than everything else combined. Ask them of
your own document before I do.

**Question 1: "Show me the query."**

For every SLI, I want the actual PromQL, or the actual metric names and the
aggregation, that computes it from data that exists in your system today.

This question kills a majority of SLO documents on first contact. The typical
failure: the SLI says "requests served in under T milliseconds", the service emits
a Micrometer `Timer`, and either it publishes no histogram at all or its buckets sit
nowhere near T. The SLI is therefore not computable. The document describes a
measurement that does not exist.

The second-most-common failure: the good-event definition includes a semantic
condition ("the product list was non-empty when it should not have been") that the
application does not emit as a metric at all. That is fine — but then the document
must contain the work item to emit it, with an owner, and the SLO is marked "not
yet in force" until it lands. What is not fine is writing the SLI as though the data
existed.

Third failure, and it is a Topic 118 callback: the aggregation averages percentiles
across pods. That is arithmetically invalid. If your query does
`avg(histogram_quantile(...))` I will stop reading and we will start again.

**Question 2: "The budget is exhausted. It is Tuesday. Walk me through the next
four hours — who decides what, and what specifically does not ship?"**

I want names, a specific decision, and a specific thing that stops. Vagueness here
means the policy is decorative.

The follow-ups I will actually ask:

- "The product manager says this one feature is committed to a customer for Friday.
  What happens?" (If the answer is "we'd discuss it", the policy has no teeth. The
  right answer names the override path and the fact that the override is logged and
  expires.)
- "Who tells sales?" (Almost never answered. The SLO has an external face and
  somebody must own it.)
- "What if the burn was one bad deploy that is already rolled back and the service
  is now healthy?" (Tests whether you understand that a rolling window keeps the
  spend on the books even after recovery — and whether your policy sensibly keys the
  exit condition to *trailing burn rate* rather than to *remaining budget*, since
  remaining budget cannot recover until the window rolls past the incident.)
- "Name the last time a policy like this was actually invoked in an organisation you
  worked in, and what happened." (Testing whether you have thought about the social
  reality or only the mechanism.)

**Question 3: "Multiply your dependencies. Is your target above your ceiling?"**

If the target exceeds the serial product of the dependency availabilities and the
document does not name the architectural change that raises the ceiling, the target
is arithmetic fiction. This is the fastest way to falsify a reliability commitment
and it takes about ninety seconds.

Corollaries I will pursue:

- "Where did each dependency's number come from?" (A vendor's marketing page, a
  contractual SLA, or your own measurement? These differ, sometimes a lot. A
  contractual SLA is a floor with a refund attached, not a prediction.)
- "Which of these are actually serial?" (If you have marked Kafka serial when you
  have an outbox, you have understated your ceiling and you do not understand your
  own architecture. If you have marked it non-serial, show me the freshness SLI that
  covers the asynchronous path — otherwise you have moved the risk somewhere
  unmeasured, which is worse than leaving it visible.)
- "What is the correlation between these failures?" (The multiplication assumes
  independence. If Postgres and Redis are in the same availability zone, they are
  not independent, and your real ceiling is worse than the product. Correlated
  failure is the thing that makes availability arithmetic optimistic.)

### The other attacks, in the order I will make them

**On the SLIs:**

- "Name a way `orderflow` could be badly broken while this SLI reads 100%." Every
  SLI has one. If you have not written it down, you have not finished the SLI.
- "Your denominator excludes 4xx. Show me how you would detect it if a validation
  bug started rejecting valid orders with a 400." If the answer is "we'd notice",
  that is not a mechanism.
- "This latency threshold — which percentile of your current Topic 65 baseline
  distribution does it sit at?" If you cannot answer, the threshold was picked
  because it was a round number.
- "You measure at the edge proxy. What fraction of real user failures happen before
  the edge proxy? How would you find out?" I am not looking for a number. I am
  looking for you to acknowledge the gap and propose a way to size it.

**On the targets:**

- "You propose a target above your current attainment. What specifically changes to
  close the gap, by when, and who is doing it?" A target that assumes future
  improvement without a named plan is a wish with a decimal point.
- "You propose a target at your current attainment. So you have committed to never
  regressing while also shipping features. Is that honest?" Both directions are
  attackable. Have an answer.
- "Your confidence column says high. What would three consecutive missed windows
  mean — that you were unlucky, or that the target is wrong? How would you tell?"

**On the policy:**

- "Show me the incentive to game this." Every policy has one. If the consequence is
  a freeze, the incentive is to weaken the SLI definition or to widen the exclusion
  list. Name the gaming path and say what prevents it.
- "The override log is empty. In twelve months, how many entries do you expect? Zero
  means the policy never binds or nobody is honest. Twenty means it is not a policy."
- "Who is worse off under this policy, and have you talked to them?" Usually the
  feature team. If you have not spoken to them before publishing, the policy will be
  resented and resented policies get repealed.

**On the whole document:**

- "What did you decide not to include, and why?" I am checking whether the scoping
  was deliberate.
- "If I disagreed with your central target, which paragraph would I attack?" A
  Principal author knows exactly where their own document is weakest and has already
  written the counter-argument into it. If you cannot name the paragraph, you have
  not stress-tested your own work.
- "Six months from now, traffic has tripled. Which parts of this document are now
  wrong?" Tests whether the reasoning is written down or only the conclusions.

---

## Interview questions (Senior → Principal)

The rubric line from the master plan for this topic: *"we target 99.99% uptime" →
"on what SLI, measured where, over what window? Availability measured at the load
balancer isn't what the user experiences. And the extra nine costs multi-region and
on-call load — the question for product is whether the revenue at risk justifies
that, and here's the number."*

Everything below builds from there.

---

### Q1 — "What SLO would you set for the order-placement API?"

**Senior answer:** "I'd look at our current p99 and error rate and set something
achievable — probably 99.9% availability with a latency target around our current
p95. Then we'd track it in Grafana and review it monthly."

That answer is correct and it is not enough. It names a target and a tool. It does
not name a promise.

**Principal answer:** "Before a number, three things. First, order placement is a
journey, not an endpoint — the promise has to cover reserve-inventory, debit-wallet
and create-payment as one thing, because succeeding at two of three is a worse
outcome for the user than failing all three. Second, where we measure: at the edge,
plus sampled client reporting, because our load-balancer view can't see requests
that never arrived or responses that arrived after the client gave up. Third, the
good-event definition includes latency and includes a correctness clause — we have a
real failure mode where we return 200 and no order row exists, and a status-code SLI
cannot see it.

Then the number. Our dependency ceiling is the serial product of Postgres, the pool,
and the payment gateway; I'd compute that first, because a target above the ceiling
is fiction. I'd propose the achievable target with the aspirational one beside it and
its price written out — second region, tested failover, on-call headcount. And the
target is worth nothing without the budget policy, which I want signed while the
budget is full, because the same conversation during an incident is unwinnable."

**What separates them:** the Principal answer treats the target as the *last* thing
decided rather than the first. It leads with journey, measurement point, and
good-event definition, then constrains the number with arithmetic, then makes the
policy the actual deliverable. It also names a specific `orderflow` failure mode
that the naive SLI would miss — evidence of a real system rather than a template.

**Adversarial follow-up:** *"You're overcomplicating this. Every other service here
has a simple availability SLO and it works fine. Why are you special?"*

The trap is to become defensive or to concede the whole structure. The good answer
concedes the reasonable part and holds the load-bearing part: "You're right that the
structure should match the risk, and for a read-only service a simple availability
SLO is genuinely correct — I would not add a correctness SLI to the catalogue read
path. Order placement is different for one specific reason: it moves money and
mutates three pieces of state. The failure mode where we return success and lose the
order is not theoretical here; we have produced it in testing. So I want one extra
clause on one SLI. If the reconciliation job to support that clause turns out to
cost more than a sprint, I'd drop it and say so in the document as an accepted risk."

That answer holds a position, concedes a real point, scopes the disagreement to one
clause, and pre-commits to a condition under which it would fold. That is the shape
of every good answer under pressure, and it is what Topic 135 tests.

---

### Q2 — "Product wants 99.99%. Go."

**Senior answer:** "I'd explain that 99.99% is very expensive and probably not
realistic for us, and suggest 99.9% instead, which is more achievable given our
current architecture."

Right instinct. It is an assertion of judgment with no evidence attached, so it
becomes your credibility against theirs, and that is a coin flip.

**Principal answer:** "I'd start by agreeing it's achievable and asking what it
costs — because it is achievable and the answer is a project, not an argument.

99.99% is four minutes of badness per 28 days. Our rolling deploys currently take
longer than that to drain and become ready — that alone is most of the budget, so
step one is proving zero-drop deploys, which is measurable and we may already be
close. Then the dependency ceiling: if our serial dependencies multiply out below
99.99%, no amount of care gets us there, and the only routes are removing a
dependency from the critical path or adding a region. Then on-call: four minutes
means detection and mitigation inside four minutes, which implies a paging rota with
a response commitment, which implies headcount.

So my answer to product is: here is the priced package. And then the question that
is genuinely theirs, not mine — what is the revenue at risk from an hour of failed
checkout? If that number is larger than the annual cost of the package, this is an
easy yes and I'll build it. If it is smaller, we are buying reliability the business
does not need, and I would rather spend that engineering time on the thing they
actually want. I am not saying no. I am saying this is a funded project and here is
the invoice."

**What separates them:** the Principal answer never refuses. It converts an
aspiration into a priced package and hands the decision back to the person who owns
the trade-off, along with the one piece of information only they have. It also
volunteers the possibility that the answer is yes, which is what makes it credible
rather than obstructive.

**Adversarial follow-up:** *"The deal is worth more than the second region. So we're
doing it. How long?"*

Do not flinch, and do not invent a date. "Good — then I want it. I'll come back
within a week with a sequenced plan and the two numbers I don't have yet: how much
budget our current deploy process actually spends, and our real measured dependency
availability rather than the vendors' published figures. The reason I won't give you
a date right now is that the honest answer depends on whether the payment gateway
can come off the serial path, and that is a design question I need two days on. What
I will commit to today is the date I'll give you the date."

That is the correct move and it is worth internalising: **committing to a date for
the estimate is a real commitment, and it is honest.** Inventing the estimate is not.

---

### Q3 — "Your SLO says 99.9% and you're at 99.95%. Are you doing well?"

**Senior answer:** "Yes — we're beating our target with headroom."

**Principal answer:** "Probably, but it is worth interrogating. Three possibilities.

One: the target is right and we have healthy headroom. Fine.

Two: the target is too loose. If we have not touched the budget in three
consecutive windows, we are more reliable than we promised, which means we may be
spending engineering effort on reliability our users did not ask for. That is a real
cost and I would put it on the table — either raise the target, or deliberately
spend the slack on velocity, which might mean shipping faster or with lighter
process on that path.

Three, and this is the one I would check first: the SLI is not seeing our failures.
The strongest evidence would be customer-reported issues outnumbering
SLI-detected ones. If support is raising incidents our dashboard never noticed, then
99.95% is measuring something other than our users' experience, and the number is
worse than useless because it is actively reassuring.

So my answer is: let me compare SLI-detected incidents against customer-reported
ones for the last quarter. That ratio tells us which of the three we are in."

**What separates them:** treating an unspent budget as a signal requiring
investigation rather than as a win. And naming the specific diagnostic — the
detected-versus-reported ratio — rather than gesturing at "we should check". The
willingness to say "we may be over-invested in reliability" is unusual and lands
hard, because it demonstrates the person is optimising for the business rather than
for their own function.

**Adversarial follow-up:** *"So you're saying we should make the service less
reliable?"*

"No — I'm saying we should stop paying for reliability nobody asked for, which is a
different thing. Concretely: if we are consistently beating the target, I would not
reduce reliability; I would reduce *effort spent defending it beyond the target*.
That might mean a lighter review process on a path that has not had an incident in a
year, or not building the second region we were considering. The service stays as
reliable as it is. We just stop spending on making it more so, until the SLO tells us
we need to."

---

### Q4 — "Walk me through an SLO you'd set for something asynchronous."

**Senior answer:** "For the Kafka consumer I'd alert on consumer lag — if lag goes
above a threshold, we page."

That is a monitoring answer to an SLO question. Lag is a system property. The user
does not experience lag.

**Principal answer:** "The user-facing promise is freshness: 'my order status is
correct within N minutes of my payment completing.' So the SLI is: of all payments
completing in the window, the fraction whose resulting order-status update was
applied within N minutes.

That is measured end to end, from the payment event's timestamp to the status
write's timestamp — not from consumer lag, because lag is one contributor among
several. The others: outbox relay latency, rebalance stalls when a consumer is
slower than `max.poll.interval.ms`, and retry backoff on a downstream failure. A lag
alert sees one of those four.

N comes from the business, not from me. It is 'how long before a customer emails
support asking where their order is' — support can tell you that, and it is usually
much longer than engineers assume, which is useful because it buys headroom.

The budget then has a natural consequence: if we are burning freshness budget, the
fix is either more consumer parallelism, or reducing per-message work, or
partitioning differently — and the burn analysis tells us which, because I can
attribute the delay across the four contributors from trace spans.

One thing I would state explicitly as not promised: ordering. Freshness is a
latency promise, not an ordering promise, and conflating them is how people end up
believing things about a Kafka pipeline that are not true."

**What separates them:** an end-to-end user-visible promise instead of a component
metric; the decomposition into contributors so the SLI is actionable; sourcing the
threshold from support rather than from engineering taste; and the explicit
non-promise at the end.

**Adversarial follow-up:** *"Your freshness SLI has been at 100% for months. Why does
it exist?"*

"Because its job is to detect a class of failure that has not happened yet, and the
cost of it not existing is that a stalled relay is invisible until customers
complain. That said, you have a fair point about attention cost. An SLI nobody
looks at is clutter. So I would keep the SLI and the slow-burn ticket alert, drop
the fast-burn page if it has never fired, and put the SLI in the quarterly review
rather than the weekly one. If it stays at 100% for a year and we have no
architectural change on that path, I would demote it to a monitored metric and say
so in the document."

Note what happened: the position held, but the *attention budget* concern was
accepted and produced a real change. That is the difference between defending a
position and reasoning in public.

---

### Q5 — "You inherit a service with a 99.99% SLO that has never been met. What do you
do in your first month?"

**Senior answer:** "Find out why it's failing and fix the biggest causes."

**Principal answer:** "The first thing I would do is find out whether the number was
ever costed, because that changes the whole approach.

If it was costed and funded and we are simply failing to execute, that is an
engineering problem and I would attack the burn: pull the last three windows, rank
contributors by budget spent, and go after the top one. Almost always the top
contributor is something structural and boring — deploys, a single dependency, or one
endpoint.

If it was never costed, and I would bet on that, then the number is fiction and
fixing the burn is the wrong first move. The first move is to make the fiction
visible without embarrassing anyone. I would produce the four-minutes-a-month
arithmetic, the dependency ceiling, and the cost table — and I would deliberately not
frame it as 'the previous team was wrong'. I'd frame it as 'here is what this number
costs, here is what we have, here is the gap'. Then I would propose the achievable
number, the policy, and a written path to the aspirational number.

The political part matters and I'd be deliberate about it. Somebody chose that
number and may still be in the building. The document should let them change their
mind without losing face, which usually means writing it as new information rather
than as a correction. And I would get the person who owns the external commitment —
sales, or whoever quoted it to a customer — into the conversation early, because if
the number is contractual then my options are much narrower and I need to know that
in week one, not week six."

**What separates them:** diagnosing whether the problem is execution or arithmetic
before choosing a strategy; treating the social cost of correcting an inherited
commitment as a first-class design constraint; and finding out in week one whether
the number is contractual, which is the fact that determines everything else.

**Adversarial follow-up:** *"The number is in a signed customer contract. Now what?"*

"Then it is an SLA and my internal SLO must be stricter than it, so we find out
before the customer does. And the conversation is no longer with product — it is
with whoever can talk to the customer about the contract, plus legal, because the
options are: fund the work to actually meet it; renegotiate the commitment; or
accept the financial exposure of the penalty clause and price it. All three are
legitimate and none of them is mine to choose. What is mine is putting an accurate
number on each: the cost to meet it, the current probability of breaching it per
quarter, and the penalty exposure. I'd have those three numbers within two weeks and
I would not editorialise beyond them."

---

## Mental model checkpoint

Open questions. No lookups. Write answers down; several of these will reappear in
Topic 135's hostile deep-dive.

1. Your SLI is request-based. A two-hour outage happens at 3 a.m. when traffic is
   4% of peak. The budget barely moves. A ten-minute outage at peak spends far more.
   Is that the right behaviour? Construct the strongest argument that it is wrong,
   then say which you would actually choose for `orderflow` and why.

2. You have an availability SLI and a latency SLI on the same journey, with separate
   budgets. A request that is slow *and* fails counts as bad in both. Is
   double-counting a defect or a feature? Would a single combined SLI be better, and
   what would you lose?

3. Suppose the error-budget policy works perfectly and stops feature work for two
   weeks. The product manager complies without complaint, and their roadmap slips.
   What is the second-order effect on your relationship with that team, and what
   would you do in advance to prevent the policy from being the thing you are
   remembered for?

4. Your dependency ceiling calculation multiplies availabilities and assumes
   independence. Name three ways `orderflow`'s dependencies are correlated. Does
   correlation make your real ceiling better or worse than the product, and does the
   answer depend on the direction of the correlation?

5. An SLO is a promise. To whom? Write down who you believe the audience is for
   `orderflow`'s SLO, then argue that it is actually a different audience, and say
   what would change in the document if you were wrong about this.

6. You cannot measure a correctness SLI without an independent check, and building
   the check costs a sprint. Under what circumstances would you publish an SLO
   document that says "we cannot currently detect this class of failure" rather than
   delaying the document until you can? What is the cost of each choice?

7. Topic 129 gave you a cost-per-request model. Topic 130 gives you a reliability
   target. Sketch how you would express reliability as a cost-per-request number, so
   the two documents can be read together. What breaks in that conversion, and is
   the resulting number honest enough to put in front of a finance partner?

---

## Quick reference card

### The four terms

| Term | One line | The test that it is real |
|---|---|---|
| SLI | good events / valid events | You can show the query against data that exists today |
| SLO | a target for an SLI over a window | Both the target and the window are stated |
| Error budget | 1 − target, expressed in events or minutes | You can state it as a countable number |
| Budget policy | what changes when the budget is spent | A named human, a specific consequence, an override log |

### Downtime arithmetic (verifiable, not a benchmark)

| Target | Per 28 days | Per 30 days | Per year |
|---|---|---|---|
| 99% | 6 h 43 m | 7 h 12 m | 3 d 15 h 36 m |
| 99.9% | 40 m 19 s | 43 m 12 s | 8 h 45 m 36 s |
| 99.95% | 20 m 10 s | 21 m 36 s | 4 h 22 m 48 s |
| 99.99% | 4 m 2 s | 4 m 19 s | 52 m 34 s |
| 99.999% | 24 s | 26 s | 5 m 15 s |

### The four SLI shapes

| Shape | Ratio | `orderflow` example |
|---|---|---|
| Availability | successful / valid | order placement succeeds |
| Latency | fast enough / valid | order placement inside T ms |
| Freshness | processed in time / total | order status correct within N min of payment |
| Correctness | agreeing / checked | wallet balance matches ledger sum |

### The dependency ceiling

```
ceiling = product of the availabilities of every SERIAL dependency
```

Above the ceiling you need an architectural change, not more care. Ways to raise
the ceiling: remove the dependency from the critical path (outbox); add redundancy
so one failure is survivable; degrade gracefully so the dependency's failure is not
the user's failure.

### Checklist before publishing

- [ ] Every SLI has a runnable query against data that exists today.
- [ ] Every SLI's good-event definition includes latency where latency matters.
- [ ] At least one freshness SLI and one correctness SLI exist.
- [ ] The denominator (valid events) is written down for every SLI.
- [ ] What the measurement cannot see is stated explicitly.
- [ ] The dependency ceiling is computed and the target sits under it, or the
      architectural change that raises it is named.
- [ ] The target has a stated window.
- [ ] The budget is expressed as a countable number of events and of minutes.
- [ ] The policy is graduated, scoped to the failing path, and has a named decider.
- [ ] The override path exists, is logged, and entries expire.
- [ ] Burn-rate alert thresholds are derived, with the arithmetic shown.
- [ ] Histogram buckets exist around the latency threshold (Topic 118).
- [ ] No query averages percentiles across instances.
- [ ] A "what we are not promising" section exists.
- [ ] The negotiation position names what would change your mind.

### Anti-patterns, one line each

- Measuring at the load balancer and calling it the user experience.
- A target with no window.
- A budget with no consequence.
- Status-code-only "good" definitions on a service with silent failure modes.
- One SLO per endpoint instead of one per journey.
- A target above the dependency ceiling with no plan to raise the ceiling.
- Alerting on threshold crossings instead of burn rate.
- Averaging p99 across pods.
- Negotiating the policy during the incident.
- An SLO document with no stated limitations.

---

## When would I use this at work?

**1. The commitment conversation before a sales cycle.**

Someone needs a reliability number for a contract, a security questionnaire, or a
customer conversation, and the deadline is short. Without this topic you say a
number. With it, you say: "on what SLI, measured where, over what window" — then
produce the ceiling arithmetic and the priced package within a week. The
organisational value is not the number. It is that the commitment is now something
the business chose with the price visible, rather than something an engineer guessed
under time pressure. This is the single highest-leverage half-day of work available
to a Principal engineer on a young service.

**2. Settling a recurring reliability-versus-features argument.**

Every service has this argument, quarterly, and it is decided by seniority and
volume. The budget converts it to arithmetic and the policy converts the arithmetic
into a pre-agreed action. The first time the policy actually stops something is
uncomfortable and it is also the moment the whole thing becomes real. Getting the
policy signed while the budget is full is the specific move; it is cheap then and
impossible later.

**3. Deciding whether a piece of reliability engineering is worth doing.**

A team proposes a second region, or a multi-AZ Postgres, or a big resilience
refactor. The SLO document is what makes that decision answerable rather than a
matter of taste: does the proposal move an SLI we are actually missing, by enough to
matter, at a cost proportionate to the budget it recovers? Frequently the honest
answer is no, and being the person who says "we do not need this yet, and here is
the arithmetic" is worth more to your credibility than any amount of advocating for
reliability work. It also gives you the standing to be believed the day you say the
opposite.

---

## Connected topics

**Prerequisites — the evidence base this document is built on:**

- **65 — GATE: service under load.** Your recorded p50/p95/p99 baseline. Every
  latency threshold in the SLO must be traceable to it. A latency target set without
  a baseline is a guess with a decimal point.
- **118 — Micrometer metrics, RED/USE, cardinality.** Where SLI numerators and
  denominators come from. Histogram buckets must be placed around your SLO threshold
  or the SLI is not computable. And the cardinality lesson binds: attribute 4xx by
  bounded reason codes, never by order ID.
- **119 — Distributed tracing with OpenTelemetry.** How you attribute a breached
  latency SLI to a specific dependency. Without traces, a burn analysis is guesswork
  and your budget policy produces no useful action.
- **121 — Actuator, readiness and liveness.** Readiness going false before shutdown
  is what makes a deploy cost zero budget instead of a large fraction of it.
- **123 — Graceful shutdown and rolling deploys.** At 99.99%, deploy behaviour is
  most of your budget. This is where you find out whether the target is reachable
  at all.
- **124 — GATE: production-readiness review.** That review already required "SLOs
  and current attainment". This topic is what makes that section defensible instead
  of aspirational.
- **129 — Capacity, cost and latency budgets.** The cost side of the trade. The SLO
  says how reliable; Topic 129 says what that costs per request. The 99.99%
  conversation is unanswerable without both.

**Failure modes this document must be able to see — all of which you have produced:**

- **41** — `@Cacheable` outside `@Transactional` serving a rolled-back value: a 200
  that is wrong.
- **51** — stale L2 read after a native SQL write: a 200 that is wrong.
- **55** — HTTP call inside a transaction exhausting HikariCP: latency degradation
  that a status-code SLI cannot see.
- **92** — check-then-act on the idempotency cache producing a double wallet debit:
  the case for a correctness SLI.
- **111** — retry storms without jitter: burn concentrated into minutes.
- **115** — event lost between commit and publish: the case for a freshness SLI.
- **121** — liveness probe checking the database, causing a restart cascade: a
  self-inflicted budget event.

**This unlocks:**

- **131 — Design-doc authorship.** The SLO document is a design doc with a
  specialised shape. The alternatives-and-reversibility discipline from 131 applies
  to it directly; the negotiation position is where its trade-offs become legible.
- **132 — Engineering standards.** The budget policy is a standard, and it obeys the
  same adoption law: it is followed when following it is cheaper than ignoring it.
  A policy scoped to one failing path is cheap; a whole-team freeze is a rule people
  route around.
- **133 — Incident postmortems.** Budget spent is how you size an incident, and the
  burn analysis is the first section of the postmortem. "This incident cost 31% of
  the window's budget" is a better severity statement than any severity label.
- **134 — Influence without authority.** Getting three teams to adopt SLOs is
  exactly the adoption problem: give them the dashboard, the query, and the template,
  and adoption becomes cheaper than explaining why they have not.
- **135 — Principal interview simulation.** The SLO document is one of the artefacts
  you will defend under sustained hostile questioning. Question 1 will be "show me
  the query", and it will be asked in that tone.

---

*This topic contains no attainment figures, no benchmark numbers, and no industry
statistics, because inventing them would give you a false anchor for a real
negotiation. The only numbers here are downtime arithmetic you can verify with a
calculator. Every table describing your system is blank by design. Fill them from
your own Topic 65 baseline and your own Topic 118 metrics — the document is worth
exactly as much as the evidence behind it and not one line more.*
