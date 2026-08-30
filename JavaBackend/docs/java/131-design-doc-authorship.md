# 131 — RFC / Design-Doc Authorship

## Phase: 12 — Principal Track
## Category: ELITE
## Java baseline: 21  |  Notes features from: 21/25
## Project spine: **you write** a design doc for a significant `orderflow` change, and I review it adversarially

---

## Mechanical statement

A design doc's job is to make a **decision** legible.

Legible means four things are written down and none of them are optional:

1. **What we are trading.** Not what we are building — what we are giving up to
   build it.
2. **What alternatives were rejected, and the specific reason each was rejected.**
3. **How reversible this is**, and what it costs to reverse at each point in time.
4. **What evidence would change the answer.**

A doc that only explains the chosen design cannot be disagreed with usefully. A
reader who dislikes it has nothing to push against except your prose, so the review
becomes a discussion about wording, and the actual decision goes unexamined.

The mechanic, stated as a test: **if a reviewer who disagrees with you cannot point
to the exact paragraph they disagree with, your document has failed regardless of
how good the design is.**

---

## The bridge from what you know

### This transfers. I am not going to re-teach it.

You have written RFCs. You have run design reviews. You have argued for an
architecture in front of people who did not want it, and you have written the
follow-up document afterwards. You know how to structure a document, how to write
for an audience that will skim, and how to take review comments without becoming
defensive.

None of that changes because the implementation language is Java. A design doc is
a design doc. The tooling is the same, the meeting is the same, the political
dynamics are the same, and the skill of writing clearly is the same skill.

**So this document spends no space on how to write well, how to structure a
document, or how to run a review meeting.** You have those.

### What is genuinely harder at Principal level

Three things. All three are about the *decision*, not the *design*.

**One: writing the alternatives well enough that they might win.**

Most engineers write an alternatives section that exists to be dismissed. Each
alternative gets two sentences, the second of which explains why it is bad. A
reviewer reads that and correctly concludes the author decided first and documented
second.

The Principal version is uncomfortable: you write the strongest version of the
alternative you rejected — strong enough that a reader might prefer it — and *then*
say why you did not choose it. This is hard because it requires you to have
genuinely held the alternative in your head as a live option, and because it exposes
you to the possibility that a reviewer says "actually, option 2 is better" and is
right.

That exposure is the point. A document that cannot lose an argument is not a
decision document.

**Two: reversibility as a first-class section.**

Almost nobody writes this and it is the section that most changes how a decision is
made. Some decisions are two-way doors — cheap to walk back, so decide fast and
learn. Some are one-way doors — expensive or impossible to walk back, so decide
slowly with more evidence. Most decisions are two-way doors that *become* one-way
doors at an identifiable moment, and naming that moment is enormously valuable.

For `orderflow`: changing an internal service boundary is reversible in a sprint.
Changing the public API contract so external clients depend on a new order state is
reversible until the first external client ships against it, and effectively
irreversible after that. Same project. Two different decision types, separated by a
date you can name in advance.

**Three: falsification — "what evidence would change the answer".**

Stating in advance what would prove you wrong is the highest-credibility move
available to a document author, and it is rare. It converts your position from an
opinion into a hypothesis. It also protects you: six months later, when someone asks
"why did we do this?", the document says what the decision was contingent on, and
you can check whether the contingency still holds.

### The Java-shaped part

Small but real. Java design docs have a recurring hazard: **the framework hides the
decision.**

A doc that says "we will use `@Transactional` with `REQUIRES_NEW` for the
compensation path" has embedded a connection-pool sizing decision (Topic 109) and a
deadlock risk without naming either. A doc that says "we will move to virtual
threads" has embedded a pinning risk (Topic 101) and a pool-sizing consequence
(Topic 90). A doc that says "we will cache the product catalogue" has embedded an
invalidation contract (Topic 51).

At Senior level you write the framework choice. At Principal level you write the
consequence the framework choice imports, because that consequence is the thing
that will actually hurt and the framework will not warn anyone about it. Naming
those in the doc is a large part of what makes a Java design doc credible to a
reviewer who has been burned before.

---

## What is this?

A **design doc** (also called an RFC, a technical spec, a one-pager, or an ADR
depending on your organisation) is a written artefact whose purpose is to get a
decision made and recorded, before the code exists.

It is not:

- **A specification.** A spec describes what will be built. A design doc explains
  why this rather than something else. You can have a perfect spec and a terrible
  design doc.
- **Documentation.** Documentation describes what exists now and is kept current. A
  design doc is a snapshot of a decision at a moment and is *not* updated afterwards
  — it is superseded by a new document. Trying to keep design docs current destroys
  their value as a record of reasoning.
- **A proposal for consensus.** A design doc seeks a *decision*, from a named
  decider, informed by review. Those are different processes and confusing them is
  the most common way a doc dies (see Wrong Approach 4).

### The three sizes, and picking the right one

| Size | When | Length | Decider |
|---|---|---|---|
| **ADR** (architecture decision record) | one bounded decision with a clear default | half a page | the team |
| **Design doc** | a change that affects other teams, is expensive to reverse, or has real alternatives | 3–8 pages | a named individual |
| **Strategy doc** | a direction across multiple projects and quarters | 5–15 pages | a leader with budget |

Choosing the wrong size is itself a failure. A three-page document about a decision
that has one obvious answer wastes everyone's attention and trains people that your
documents are not worth reading. A half-page ADR about extracting the payments
domain is negligent.

The test: **how expensive is it to be wrong?** If the cost of a wrong choice is a
week, write an ADR and move. If it is a quarter, write a design doc. If it is a
year and a hiring plan, write a strategy doc.

### Status, and why it matters

Every design doc has a status, and the status is on the first line:

```
Status: DRAFT | IN REVIEW | ACCEPTED | REJECTED | SUPERSEDED BY <doc>
```

This looks like bureaucracy and is not. A document without a status generates the
single most expensive failure mode in technical writing: **two engineers, six months
apart, both believing the document represents the current plan, when one of them is
reading a draft that was abandoned.** The status line costs one line and prevents
that entirely.

`SUPERSEDED BY` matters more than the others. It is how the decision record forms a
chain, and the chain is what lets a new joiner reconstruct why the system is the way
it is. Deleting an old doc destroys that; superseding it preserves it.

---

## Why does it matter?

**1. Because at Principal scope you cannot be in every room.**

Your influence is bounded by how many conversations you can attend. A document is
the only mechanism that scales past that. It works while you sleep, it works for
people you have never met, and it works in six months when everyone has forgotten
the conversation.

This is the reason the skill is gated at this level and not earlier. A Senior
engineer's decisions mostly affect code they will personally touch. A Principal
engineer's decisions are executed by people they will never review a PR from.

**2. Because a decision without a recorded reason gets relitigated, forever.**

You have seen this. A choice was made two years ago. Nobody remembers why. Every
new engineer proposes changing it, discovers the reason painfully, and the cycle
repeats. Each iteration costs a week and some goodwill.

The fix is one paragraph, written once, at the moment the reason was fresh. The
value of a design doc is realised almost entirely in the future, which is exactly
why the discipline is hard to maintain — the cost is now and the benefit is later,
to someone else.

**3. Because writing it is how you find out you are wrong.**

This is the underrated one. Writing "alternatives considered" forces you to
construct the alternative properly, and a meaningful fraction of the time the
alternative turns out to be better. Writing "reversibility" forces you to notice
that you were about to make a one-way decision on two-way evidence. Writing "what
would change the answer" forces you to notice that you have no evidence at all.

The document is a thinking tool first and a communication tool second. If you have
never abandoned your own proposal while writing the doc for it, you are probably
writing docs to ratify decisions rather than to make them.

**4. Because it is the artefact your promotion case is made of.**

Uncomfortable but true. Nobody can see the reasoning inside your head. A committee
deciding whether you operate at Principal level will read your documents. A doc
that makes a hard trade legible, names who is affected, states what would reverse
it, and turns out to have been right — that is the evidence. Code is rarely
legible enough to serve as evidence of judgment.

---

## The decision, framed

You need a **significant** `orderflow` change to write about. Significant means all
four of:

- It affects at least one team other than the one implementing it.
- It has at least two genuinely viable alternatives — not one real option and two
  strawmen.
- It is expensive to reverse, or it becomes expensive at an identifiable moment.
- You have evidence from Phases 7–11 that bears on it.

Here are five candidates from your own system. Pick one. I recommend the first, and
I will use it as the worked example.

| # | Candidate change | Why it qualifies | Evidence you already have |
|---|---|---|---|
| **A** | **Order placement without synchronous payment authorisation** — commit the order and reservation locally, authorise the payment asynchronously, return `PENDING_PAYMENT` | Changes the API contract, the user experience, the failure modes, and the SLO shape. Affects the client teams and support. | Topic 55 pool exhaustion; Topic 111 retry storms; Topic 115 outbox; Topic 130 dependency ceiling |
| B | Move inventory decrement from `@Version` optimistic locking to a single atomic conditional UPDATE | Real throughput trade with a real correctness argument on both sides | Topic 52 drill: oversell, then three fixes with measured throughput |
| C | Adopt virtual threads for the whole request path | Affects every service using shared libraries; pinning risk is subtle | Topic 101 pinning drill; Topic 90 bounded pools |
| D | Extract payments into its own service (the Topic 128 strangler) | The data boundary is a quarter of work; the code boundary is a week | Topic 128 artefact |
| E | Introduce a read replica for the order-history read path | Consistency trade; capacity ceiling moves | Topic 129 capacity model |

### Why A is the best choice for this exercise

Because it is the one where the **obvious fix is not the right fix**, and a document
that catches that is doing real work.

The obvious fix is: the payment gateway call sits inside the order transaction, a
slow gateway therefore pins a HikariCP connection for its whole duration, and under
load the pool exhausts and every endpoint fails — including endpoints that touch no
database at all. That is precisely the Topic 55 drill. The obvious fix is to move
the gateway call outside the transaction.

That fix is correct and insufficient, and the gap between "correct" and "sufficient"
is exactly what a design doc is for. Moving the call out of the transaction fixes
the pool-exhaustion amplification. It does not remove the payment gateway from the
*serial availability path* of order placement. If the gateway is down, orders still
fail. Your Topic 130 dependency-ceiling arithmetic says that as long as the gateway
is serial, your order-placement SLO is capped by the gateway's availability, and no
amount of care changes that.

So there are at least three defensible positions — tune the synchronous path,
bulkhead it, or go asynchronous — and reasonable engineers will disagree. That is
what makes it worth a document.

### The framing sentence

Before writing anything else, write one sentence of this shape:

> We are deciding **whether to X or Y**, because **Z**, and the decision is
> **reversible/irreversible at moment M**.

For candidate A:

> We are deciding whether order placement should remain synchronous through payment
> authorisation or return before it, because the payment gateway currently sits on
> the serial availability path and caps our order-placement SLO, and the decision is
> reversible internally but becomes one-way once external clients handle the
> `PENDING_PAYMENT` state.

If you cannot write that sentence, you do not yet know what you are deciding, and
the document will wander. Write it first, put it at the top, and check every section
against it.

---

## Example 1 — a minimal illustration

The smallest possible complete design doc. One decision, one page. It is a real
`orderflow` decision and it is drawn straight from the Topic 53 drill.

---

```
ADR-011: Order-line primary key generation strategy

Status: ACCEPTED
Author: <you>
Decider: orderflow tech lead
Date: <date>
Supersedes: none

DECISION
We generate OrderLine primary keys with GenerationType.SEQUENCE using a
pooled optimiser (allocation size 50), not GenerationType.IDENTITY.

CONTEXT
Order placement writes one Order and N OrderLines in a single
transaction. Bulk import writes order lines in batches of up to 50,000.

We measured the bulk path with IDENTITY and with SEQUENCE+pooled under
the Topic 65 load harness, counting JDBC statements via Hibernate
Statistics.

  | Strategy            | Statements for 50k lines | Wall time |
  |---------------------|--------------------------|-----------|
  | IDENTITY            | <fill in from your run>  | <fill in> |
  | SEQUENCE + pooled   | <fill in from your run>  | <fill in> |

TRADE
We gain JDBC batch inserts. We give up strictly gapless IDs: a pooled
sequence allocates blocks, so an application restart leaves a gap in the
ID range. We accept that because no business rule reads meaning from
order-line IDs; the customer-facing order reference is a separate field.

ALTERNATIVES

1. IDENTITY (the default, and what we had).
   Strongest case for it: it is the least surprising choice, it works
   with every database we might migrate to without configuration, and
   IDs are gapless and monotonic, which makes manual investigation of
   production data easier.
   Rejected because: Hibernate must round-trip per row to obtain the
   generated key, so JDBC batching is disabled entirely regardless of
   hibernate.jdbc.batch_size. This is not tunable; it is structural.
   The bulk path is our worst write path and this makes it as slow as
   it can be.

2. Application-assigned UUIDv7.
   Strongest case for it: IDs can be generated before insert, so
   batching works and the write path never round-trips. UUIDv7 is
   time-ordered, so index locality is acceptable, unlike UUIDv4.
   Rejected because: 16 bytes per key across order lines is our
   highest-row-count table, and every foreign key inherits the width.
   We ran the index-size arithmetic and it was not worth it for a
   single-database system. We would revisit if we sharded.

REVERSIBILITY
Two-way door. Changing the generator changes only new rows; existing
IDs remain valid. Cost to reverse: one migration adding the sequence or
removing it, plus a deploy. Estimated at under a day.

WHAT WOULD CHANGE THIS
- If we sharded the orders table, IDs would need to be globally unique
  without coordination and alternative 2 would win.
- If the measured batching gain on the bulk path turns out to be under
  2x, this is not worth the configuration complexity and we revert.

WHO IS AFFECTED
The data team, whose ETL currently assumes contiguous ID ranges to
detect gaps. They have confirmed they can key on created_at instead.
Confirmed with <name>, <date>.
```

---

Read what that half-page does.

- The **decision** is one sentence and it is at the top.
- The **evidence** is a table from a measurement the author actually ran. The cells
  are blank here because I will not invent your numbers — you run the Topic 53 drill
  and fill them in. That is the point.
- **Alternative 1 is argued for before it is rejected**, and the argument for it is
  genuinely good: least surprising, portable, gapless. A reader can feel the pull of
  it. The rejection is then specific and structural, not "it's slower".
- **Alternative 2 is a real option** with a real advantage, rejected on arithmetic
  the author did.
- **Reversibility** is stated with a cost and a duration.
- **Falsification** is stated twice: one condition that would make a different
  alternative win, and one measurement threshold that would reverse the decision
  entirely.
- **Who is affected** names a real team, a real assumption they held, and a
  confirmation with a name and a date.

That last line is worth dwelling on. "Confirmed with `<name>`, `<date>`" is the
single highest-value line in most design docs, because it converts "I think this is
fine for them" into "I asked them". A large fraction of failed changes fail because
nobody asked.

Now notice what the ADR does *not* have: no background section, no glossary, no
diagram, no implementation plan. The decision is small, so the document is small.
Matching document size to decision cost is itself a skill.

---

## Example 2 — the real decision on the project spine

The full design doc for candidate A. I am giving you the **structure with the
reasoning at each step**, not a finished document you can copy — because copying it
would defeat the exercise, and because the numbers must be yours.

Where a number belongs, there is a blank. Fill blanks from your Topic 65 baseline,
your Topic 118 metrics, your Topic 119 traces, and your Topic 130 SLO document.

---

### The header block

```
DD-004: Asynchronous payment authorisation for order placement

Status: DRAFT
Author: <you>
Decider: <named individual — the orderflow service owner>
Reviewers: orders team lead, payments team lead, mobile client lead,
           support operations lead, SRE on-call representative
Review closes: <date, at least 5 working days out>
Date: <date>
Supersedes: none
Related: DD-001 (outbox pattern), SLO-001 (orderflow SLOs),
         CAP-001 (capacity model)
```

Two things here are decisions, not formatting.

**The reviewer list is a design choice.** Including the support operations lead is
deliberate: this change creates a new customer-visible state, which creates a new
category of support ticket. Excluding them and discovering that in month two is a
classic and avoidable failure. Ask yourself for every design doc: *who will have to
handle the consequences of this that I am not thinking about?* Support, on-call,
finance, and data are the usual answers and are the usual omissions.

**"Review closes" is a decision-forcing mechanism.** Without a date, a doc sits in
review indefinitely. With a date and a named decider, review has a shape. State the
rule explicitly somewhere: *comments after the close date are welcome and will be
addressed in a follow-up, but will not block the decision.*

### Section 1 — Summary

Three to five sentences. Someone who reads only this must be able to state the
decision accurately to a third party.

> Order placement currently calls the payment gateway synchronously inside the
> request. We propose committing the order and the inventory reservation locally,
> returning `PENDING_PAYMENT` to the caller, and authorising the payment
> asynchronously via the existing outbox. This removes the payment gateway from the
> serial availability path of order placement, at the cost of a new customer-visible
> intermediate state and a new class of support case. The change is internally
> reversible until external clients ship handling for `PENDING_PAYMENT`, after which
> reversing requires a client migration.

Note that the summary contains the cost, not only the benefit. A summary that
contains only benefits is marketing and a reviewer will read the rest of the
document more suspiciously because of it.

### Section 2 — Context and problem, with evidence

This section's job is to make the reader agree there is a problem *before* you
propose anything. If you have not established the problem, every argument about the
solution is happening on unstable ground.

Structure it as: the current shape, then the evidence, then the consequence.

```
Current shape:
  POST /orders opens a transaction, reserves inventory, debits the wallet,
  calls the payment gateway, writes the payment record, commits, and
  publishes an order-created event via the outbox.

Evidence:

  E1. Gateway latency contribution.
      From Topic 119 traces over <window>, the gateway span accounts for
      <___>% of POST /orders p95 and <___>% of p99.
      Source: <trace query>

  E2. Connection held for the gateway call.
      The gateway call is inside the transaction, so a pool connection is
      held for its full duration. At pool size <___> and <___> rps, Little's
      Law says we saturate the pool at a gateway latency of <___> ms.
      Our observed gateway p99 is <___> ms.
      Source: Topic 109 sizing, Topic 65 baseline.

  E3. This has already failed once in testing.
      In the Topic 55 drill we introduced a 2-second downstream call inside a
      transaction and every endpoint failed, including endpoints that touch no
      database. The failure mode is not hypothetical; it is reproducible on
      demand with our own load harness.

  E4. Dependency ceiling.
      SLO-001 computes our order-placement availability ceiling as the serial
      product of our dependencies. With the gateway serial, the ceiling is
      <___>. Our proposed target is <___>. The gap is <___>.

Consequence:
  Two distinct problems, and they need separating because they have
  different fixes:
    P1 (amplification) — a slow gateway becomes a service-wide outage
        via pool exhaustion.
    P2 (availability cap) — a down gateway fails order placement, and no
        amount of tuning changes that while it is serial.
```

Separating P1 from P2 is the most important intellectual move in the whole
document, and it is the thing a Senior version of this doc usually misses. They have
different causes, different fixes, and different costs. Conflating them leads
directly to the trap in Wrong Approach 1 of this topic: proposing the big change
when the small one would have solved the problem people actually complained about.

Say this out loud in the document: *"Alternative 3 solves P1 completely and does not
touch P2. If we only care about P1, we should stop there and this document should be
rejected."* Writing that sentence makes you extremely credible, and it is honest.

### Section 3 — Goals and non-goals

```
GOALS
  G1. Remove the payment gateway from the serial availability path of
      order placement.
  G2. Eliminate pool-exhaustion amplification from gateway slowness.
  G3. Keep the customer's mental model coherent: they must always know
      whether their order is confirmed.

NON-GOALS
  N1. Reducing p50 order-placement latency. This may improve as a side
      effect. It is not why we are doing it and must not be used to
      justify it.
  N2. Changing the wallet debit path. Wallet remains synchronous and
      transactional.
  N3. Supporting multiple payment providers. Out of scope; tracked in
      <ticket>.
  N4. Changing inventory reservation semantics. Reservation stays
      synchronous; a customer who gets an order ID has stock held.
```

Non-goals are not filler. Every non-goal is a review comment you will not have to
have. N1 in particular is a discipline move: it forbids you from later claiming a
latency win as justification, which stops the document from becoming a
justification-collector. N4 pre-empts the most likely reviewer question ("so I can
place an order for something out of stock?") by answering it before it is asked.

### Section 4 — Constraints

The things that are true regardless of what you decide. Listing them stops reviewers
proposing options that are already excluded.

```
C1. The payment gateway has no asynchronous authorisation API. Whatever
    we do, someone makes a synchronous HTTP call eventually.
C2. Mobile clients ship on the app-store cycle. Any client-visible
    contract change has a tail of <___> weeks before old clients are
    below <___>% of traffic.
C3. Wallet debit must remain atomic with order creation. Finance
    requires that an order and its debit either both exist or neither
    does.
C4. We have one Kafka cluster and one outbox relay (DD-001). Adding a
    second async path adds no new infrastructure.
```

C2 is doing heavy lifting and is the sort of constraint engineers forget. It is what
converts "reversible" into "reversible until date M" in the reversibility section.

### Section 5 — Proposed design

Keep it shorter than you want to. The design is the part everyone is comfortable
discussing, so it attracts disproportionate review attention, and every paragraph
you add here is a paragraph pulling attention away from the trade.

Cover: the new state machine, the transaction boundary, the failure paths, and the
observability.

```
State machine:
  PENDING_PAYMENT -> CONFIRMED   (authorisation succeeded)
  PENDING_PAYMENT -> FAILED      (authorisation declined)
  PENDING_PAYMENT -> EXPIRED     (no authorisation outcome within <___>)

Transaction boundary:
  T1 (synchronous, in-request): reserve inventory, create order in
      PENDING_PAYMENT, debit wallet, write outbox row. Commit.
      The gateway is not called. No connection is held across a
      network call to a third party.
  T2 (asynchronous, in the relay consumer): call gateway, then a short
      local transaction to move state and write the payment record.

Failure paths, each with its handling:
  F1. Gateway declines -> order FAILED, wallet debit compensated,
      inventory reservation released. This is a saga compensation
      (Topic 117) and its failure mode is itself a case: F1a.
  F1a. Compensation fails -> the order is in a state where money moved
      and no order exists. Requires a reconciliation job and an alert.
      This is the worst state in the design and the document must say so.
  F2. Gateway unreachable -> retry with jitter and a cap (Topic 111).
      After <___> attempts over <___>, the order EXPIRES and F1's
      compensation runs.
  F3. Relay stalled -> orders sit in PENDING_PAYMENT indefinitely.
      Detected by the freshness SLI in SLO-001, not by any availability
      metric.

Observability:
  - Freshness SLI: fraction of orders leaving PENDING_PAYMENT within
    <___> of creation. This is a new SLI and SLO-001 must be amended.
  - A gauge of orders currently in PENDING_PAYMENT, with an age
    histogram. Bounded cardinality: bucket by age, never tag by order ID
    (Topic 118).
  - Trace continuity across the Kafka boundary; the consumer runs in a
    different process, so context must travel in message headers
    (Topic 119).
```

F1a deserves a paragraph of its own in the real document. The honest framing:
*"this design introduces a new state in which the customer has been debited and has
no confirmed order. Today that state cannot occur because everything is in one
transaction. We are trading a rare, loud failure for a rarer, quieter one, and
quieter failures are more dangerous. The mitigation is a reconciliation job with an
alert, and if we are not willing to build that job, we should not do this change."*

That paragraph will be the most attacked paragraph in the document, and it should
be, because it names the real cost. Writing it yourself is far better than having a
reviewer find it.

### Section 6 — Alternatives considered

The heart of the document. Four alternatives, each argued for before it is rejected.

**Alternative 1 — Do nothing; tune the synchronous path.**

*The case for it:* it is free, it is zero-risk, and it may be sufficient. If gateway
p99 is comfortably inside our latency budget and gateway availability is comfortably
above our SLO target, then P1 and P2 are both theoretical and this document is
solving a problem we do not have. This is the alternative most likely to be correct
and it deserves to be listed first for that reason.

*Rejected because:* `<state the specific measurement from E1/E2/E4 that rules it
out>`. If you cannot point at a measurement here, stop writing and go and measure.
A design doc whose "do nothing" rejection is qualitative is a design doc that has
not established a problem.

*What would make it win:* if the gateway's measured contribution to our error budget
over a full quarter is below `<___>`% then the complexity of anything else is not
justified.

**Alternative 2 — Keep synchronous; add a bulkhead, a short timeout, and a circuit
breaker.**

*The case for it:* this is a small, well-understood change using machinery we
already have from Topic 111. A bulkhead caps how many in-flight requests can be
waiting on the gateway, so gateway slowness can no longer consume the whole pool —
it consumes a bounded slice. A short timeout bounds the connection-hold time. A
circuit breaker fails fast when the gateway is unhealthy instead of queueing. It
requires no API change, no new state, no client work, and it is reversible in a
deploy. It fully solves P1.

*Rejected because:* it does not touch P2. When the breaker is open, order placement
fails. The customer experience of "fail fast" is still failure. Our dependency
ceiling is unchanged, so our SLO target remains capped by the gateway.

*What would make it win:* if P2 is acceptable — that is, if the business is content
for order placement to be unavailable whenever the payment provider is unavailable —
then Alternative 2 is strictly better than the proposal: cheaper, smaller,
reversible, no new failure states. **This is the crux of the entire document and it
should be stated exactly that plainly.** The decision is not technical. It is whether
the business accepts the gateway's availability as its own.

**Alternative 3 — Move the gateway call outside the transaction, keep it inside the
request.**

*The case for it:* the smallest change that addresses the pool problem directly. The
transaction commits the reservation and the debit; the gateway call happens after,
still in the request; a second short transaction records the outcome. The connection
is not held across the network call. The API contract does not change at all. No new
state, no client work, no support impact.

*Rejected because:* it solves P1 but introduces a dual-write between the local
commit and the gateway outcome, which is the exact failure DD-001's outbox exists to
prevent (Topic 115: `kill -9` between commit and publish leaves an order with no
event). And it leaves P2 untouched. So it takes on a correctness risk without buying
the availability improvement.

*What would make it win:* if we decide P2 is acceptable *and* we want to reduce
connection-hold time without adopting a bulkhead. That is a narrow window, but it is
real, and a reader who is in that window should be able to find it in this document.

**Alternative 4 — Degrade to asynchronous under pressure; stay synchronous
normally.**

*The case for it:* the best of both. Normally the customer gets a confirmed order
synchronously with no new state to understand. When the gateway is slow or the
breaker opens, we fall back to the asynchronous path, so we stay available. Optimises
the common case and protects the tail.

*Rejected because:* the client must handle `PENDING_PAYMENT` anyway, so we pay the
entire client-side and support-side cost of the new state and get it only rarely.
Rare paths are undertested paths — this one would execute only during incidents,
which is the worst time to exercise a code path for the first time. And it doubles
the number of states the system can be in for any given order, which roughly doubles
the reasoning cost for every future change to this flow.

*What would make it win:* if the client-side cost of the new state turns out to be
near-zero — for example if clients already poll order status for other reasons —
then the argument against it weakens considerably, because the "you pay the cost
anyway" objection is what kills it. Check this with the client teams before
finalising.

Look at what that fourth alternative does. It is rejected, but the rejection is
contingent on a fact the author is not certain about, and the document says so and
names the check. That is what an honest alternatives section looks like.

### Section 7 — The trade, stated plainly

One short section. Do not bury this.

```
WE GAIN
  - The payment gateway leaves the serial availability path. Our ceiling
    rises from <___> to <___>.
  - Gateway slowness cannot exhaust the connection pool.
  - Retry policy for authorisation becomes a server-side concern with
    proper backoff, rather than the customer pressing the button again.

WE GIVE UP
  - A simple mental model. "Order placed" no longer means "paid".
  - A new customer-visible state, which means client work, support
    training, and a new category of "where is my order" contact.
  - A new failure state F1a (debited, no confirmed order) which today is
    impossible. This is the real cost of the change.
  - Test surface: the asynchronous path needs its own failure testing,
    including the compensation and the compensation's failure.

WE ARE NOT SURE ABOUT
  - Whether the support contact volume from PENDING_PAYMENT is small or
    large. We have no way to know before shipping. Mitigation: ship to
    <___>% of traffic first and measure contact rate before proceeding.
```

The "we are not sure about" block is unusual and you should include one in every
design doc you write. It signals that you know the difference between what you have
established and what you are assuming, which is the single most important signal a
document sends about its author.

### Section 8 — Reversibility

```
Two-way door until: the first external client ships a version that
                    handles PENDING_PAYMENT.
One-way door after:  that point.

Cost to reverse, by phase:

  Phase 1 (server-side only, feature-flagged off)
    Reverse cost: flip a flag. Minutes.

  Phase 2 (enabled for internal traffic / <___>% canary)
    Reverse cost: flip the flag, plus draining orders currently in
    PENDING_PAYMENT. Hours.

  Phase 3 (enabled for all traffic, clients updated)
    Reverse cost: clients must ship again; per constraint C2 the tail is
    <___> weeks. Plus a migration for in-flight orders. Weeks.

  Phase 4 (external API consumers depend on the state)
    Reverse cost: a deprecation cycle with third parties. Quarters, and
    partly outside our control.

THE POINT OF NO RETURN is the transition from Phase 2 to Phase 3, and we
should treat that transition as a separate decision with its own
go/no-go, not as a rollout step.
```

That last sentence is the most valuable in this section. It converts one large
irreversible decision into a small reversible one plus a later, better-informed
decision. **Finding a way to defer the irreversible part until you have more
evidence is one of the highest-value things a design doc can do**, and it is almost
never in the Senior version of the same document.

### Section 9 — What would change the answer

```
We would abandon this design if:

  X1. Measured gateway availability over a full quarter turns out to be
      at or above our SLO target, making P2 a non-issue. Check: pull the
      last four quarters from the gateway's status history and our own
      Topic 119 traces.

  X2. The support team's estimate of contact volume from a new pending
      state exceeds <___> contacts per thousand orders. Check: ask them
      for their estimate from a comparable state change, before we build.

  X3. The Phase 2 canary shows the F1a state occurring more than <___>
      times per <___> orders, indicating our compensation is not reliable
      enough to be trusted with money.

We would choose Alternative 2 instead if:

  Y1. The business explicitly accepts that order placement is unavailable
      when the payment provider is unavailable. That is a product
      decision, not an engineering one, and it is the fastest way to
      make this document unnecessary.
```

Y1 is the important one. You have written down the single fact that makes your own
proposal wrong, and you have identified who owns that fact. That is Principal-level
work: it routes the decision to the person who should be making it, and it makes
your document useful even in the case where your proposal loses.

### Section 10 — Who is affected

```
| Group | Impact | Consulted? |
|---|---|---|
| Mobile client team | Must handle a new order state; app-store tail | <name>, <date> |
| Web client team | Must handle a new order state | <name>, <date> |
| Support operations | New contact category; needs a runbook and script | <name>, <date> |
| Finance | New reconciliation requirement for F1a | <name>, <date> |
| On-call / SRE | New freshness alert; new runbook for a stalled relay | <name>, <date> |
| Data team | Order lifecycle events change shape | <name>, <date> |
```

Empty cells in the "consulted" column are the most damning thing in a design doc,
and reviewers look there first. Fill them before you send it. If a group has not
been consulted, write "not yet — will consult before review closes" rather than
leaving it blank; the omission then reads as a plan rather than an oversight.

### Section 11 — Rollout, with rollback points

Four phases, each with an entry gate, an exit gate, and a rollback. Not a timeline
— a sequence of decisions.

```
Phase 1: server-side implementation behind a flag, flag off.
  Exit gate: async path passes the failure suite including F1, F1a, F2,
             F3. Rollback: none needed; flag is off.

Phase 2: enable for <___>% of internal traffic.
  Exit gate: freshness SLI within <___>; zero F1a occurrences over
             <___> orders. Rollback: flip the flag.

  ** GO/NO-GO DECISION ** — this is the point of no return.
     Decider: <named>. Evidence required: the Phase 2 exit metrics plus
     support's readiness confirmation.

Phase 3: clients ship handling; enable for all traffic.
  Exit gate: <___>% of client traffic on a version that handles the
             state. Rollback: expensive; see Section 8.

Phase 4: remove the synchronous code path.
  Entry gate: <___> weeks of Phase 3 stability. Rollback: none.
```

### Section 12 — Open questions

List them. An empty open-questions section on a change this size is a claim nobody
believes.

```
Q1. Does the gateway's API allow us to detect a duplicate authorisation
    attempt idempotently? If not, F2's retry policy is unsafe and the
    design needs an idempotency key negotiated with the provider
    (Topic 116). OWNER: <name>. NEEDED BY: before Phase 1.
Q2. What is the correct EXPIRED timeout? Product decision, not
    engineering. OWNER: <name>. NEEDED BY: before Phase 2.
Q3. Should inventory reservation expire on the same clock as the
    payment? OWNER: <name>. NEEDED BY: before Phase 2.
```

Q1 is a genuine blocker and putting it in writing with an owner and a deadline is
how it gets answered. Left unwritten, it gets discovered in Phase 1 by an engineer
who then makes an assumption.

---

## Wrong approach → exact symptom → root cause → fix

Five. The symptoms are organisational and observable.

---

### Wrong approach 1 — A doc that explains the chosen design and never states the alternatives or what would reverse it

**What it looks like:** A well-written, clear, thorough document. Background,
problem, proposed design in careful detail, a diagram, an implementation plan.
Genuinely good prose. No alternatives section, or a two-line one where each
alternative is dismissed in half a sentence. No reversibility. No falsification.

**Exact symptom:** Three observable things, and all three are diagnostic.

- **Every review comment is about wording, naming, or a detail of the diagram.** Not
  one comment engages with the decision. This is not because reviewers agree — it is
  because you gave them nothing to disagree with, so they commented on the only
  surface available. Count the comments: if zero of them are about *whether* rather
  than *how*, the document has failed.
- **The review meeting ends in under fifteen minutes with "looks good".** Fast
  approval on a significant decision is a warning sign, not a success.
- **Four months later, someone reopens the decision from scratch.** A new engineer
  asks "why didn't we just do X?" and nobody in the room can answer, so the team
  spends a week re-deriving the answer — or, worse, reverses the decision without
  knowing why it was made, and rediscovers the original reason in production.

**Root cause:** The document is a *description*, not a *decision*. It answers "what
are we building" and never answers "why this rather than the other thing". A reader
cannot disagree with a description; they can only fail to be persuaded by it, which
is silent and produces no useful signal.

The deeper cause is usually sequencing: the author decided, then wrote. Writing to
ratify produces a document with no alternatives, because by the time writing started
the alternatives were already dead in the author's head.

**Fix:**

- **Write the alternatives section first, before the proposed design.** Physically
  first, in the draft. It changes what you write in the design section, because you
  will find yourself defending against your own alternatives.
- **Write each alternative's strongest case before its rejection**, in that order.
  Impose a rule on yourself: if you cannot write a paragraph that makes a reasonable
  person want to choose the alternative, you have not understood it well enough to
  reject it.
- **Add a reversibility section with costs per phase**, and look for a way to split
  the decision so the irreversible part happens later with better evidence.
- **Add a "what would change the answer" section** with concrete, checkable
  conditions.
- **Test the doc before sending:** hand it to someone and ask them to argue against
  it. If they cannot find a foothold in under five minutes, add one — because the
  foothold exists and you have hidden it.

---

### Wrong approach 2 — The doc is written after the code

**What it looks like:** The implementation is largely done. A branch exists, maybe a
PR. Now the doc gets written, because the process requires one, or because someone
asked "is there a design doc for this?"

**Exact symptom:**

- Review comments get answers of the form *"that's a good point, but we've already
  built it this way"* or *"we can revisit that later"*. Count these. More than one in
  a review is a diagnosis.
- The alternatives section is unusually weak, and specifically the alternatives are
  ones nobody would have chosen. This is because the author is reconstructing
  alternatives from memory after the fact rather than having held them as live
  options.
- **The most reliable symptom: reviewers stop engaging with your future documents.**
  They comment less, later, and more superficially. They have learned that commenting
  does not change anything, so the rational response is to stop spending the effort.
  This is expensive and slow to repair, and it is the actual damage.

**Root cause:** The document is being used to ratify a decision rather than to make
one. The decision was made at the keyboard.

There is a legitimate version of this and it is worth distinguishing: sometimes you
build a prototype *to inform* the decision, then write the doc with the prototype as
evidence. That is fine and good. The difference is whether you are genuinely willing
to throw the prototype away. If you are not, you are ratifying.

**Fix:**

- **Write at the point of maximum uncertainty**, which is before you build, when
  writing is cheapest and most useful.
- If the design is complex enough that you cannot write it without exploring,
  **write a one-page decision brief first** — problem, two or three options, the
  thing you do not know — and circulate that. It costs an hour and it gets you the
  objections while they are still free.
- If you have already built it, **say so at the top of the document.** "A prototype
  exists at `<branch>`; it informed sections 5 and 6 and is not a commitment." That
  is honest and it lets reviewers calibrate. Concealing it and being found out is far
  worse than disclosing it.
- **Institute a rule for yourself:** the doc goes out before the branch is more than
  a spike. You will violate this occasionally. Notice when you do, and notice whether
  the review got worse.

---

### Wrong approach 3 — Goals with no non-goals

**What it looks like:** A goals section listing what the change achieves. No
non-goals. Scope is implied rather than stated.

**Exact symptom:**

- The review thread accumulates comments of the form *"what about X?"* where X is a
  related but distinct problem. Ten, twenty, forty of them. Each is individually
  reasonable.
- The document goes through three or four revision rounds, growing each time, and
  never reaches a decision. Track the word count across revisions: if it is
  monotonically increasing and the status is still IN REVIEW, this is the failure.
- The author starts each round by addressing the previous round's comments, which
  generates new comments on the new material. This can run forever and sometimes
  does.
- Eventually either the author gives up and implements without approval, or the doc
  is approved out of exhaustion, which means nobody actually reviewed the final
  version.

**Root cause:** Unbounded scope invites unbounded review. Reviewers cannot tell what
is in scope, so they raise everything adjacent, because raising it is cheap and
missing something is embarrassing. The author feels obliged to address each one,
because refusing feels dismissive.

**Fix:**

- **Write non-goals explicitly**, and make some of them slightly surprising. "We are
  not reducing p50 latency" pre-empts a whole class of comment.
- For each non-goal, either **link the ticket** where it is tracked or say "we have
  decided not to solve this". Both are legitimate. Silence is not.
- Adopt a stock response for out-of-scope comments and use it consistently: *"Good
  point and out of scope per N2 — I've raised `<ticket>`."* Consistency is what makes
  it land as boundary-setting rather than dismissal.
- **Put a review deadline in the header.** Scope creep needs time to work; a deadline
  starves it.

---

### Wrong approach 4 — No named decider; the doc seeks consensus

**What it looks like:** The document is circulated widely. Everyone is invited to
comment. There is no "Decider" line, or it says "the team". The implicit goal is
that everyone agrees.

**Exact symptom:**

- The doc sits in `IN REVIEW` for weeks. Six weeks is not unusual. Look at the
  status field's age — that number alone diagnoses this.
- Every comment has been addressed. Nobody has decided. When you ask "so are we
  doing this?", the answer is a shrug or "I think so?".
- Two reviewers hold incompatible positions and neither will move, because there is
  no mechanism to resolve it other than one of them conceding, and neither has a
  reason to.
- The author eventually starts implementing anyway, which teaches everyone that the
  review process is theatre, which degrades every future review.
- Or the doc quietly dies and the problem it addressed resurfaces in six months.

**Root cause:** Consensus is not a decision procedure. It is an outcome you sometimes
get. Requiring it hands a veto to the most stubborn participant and gives nobody the
authority to close.

**Fix:**

- **Name a decider in the header.** One human, by name. Their job is not to be the
  smartest person in the review; it is to close.
- **Set a review-closes date** and state the rule: comments after that date are
  addressed in a follow-up and do not block.
- **Separate "I disagree" from "I object".** Ask disagreeing reviewers to state what
  evidence would change their position. If they can name it, you have a check to run
  and the disagreement is productive. If they cannot, it is a preference, and
  preferences inform the decider but do not block.
- Adopt the **disagree-and-commit** norm explicitly, and write it in the document
  template so it is a property of the process rather than something you invoke when
  losing.
- If two reviewers are genuinely incompatible on a matter of fact, **escalate the
  fact, not the disagreement.** "We disagree about whether the gateway's availability
  is above our target. Here is the measurement that settles it. Running it this
  week." Facts are resolvable; positions are not.

---

### Wrong approach 5 — Assertions with no evidence, in a system that has a baseline

**What it looks like:** "This will improve latency." "The current approach doesn't
scale." "This is more maintainable." Confident, plausible, unmeasured.

**Exact symptom:**

- A reviewer asks "by how much?" and the answer is "we'd have to measure". The
  document is approved anyway, because the reasoning sounds right.
- The change ships. Nobody checks whether the claimed improvement happened. Nobody
  even can, because no target was stated.
- Six months later somebody profiles the system and finds the improvement never
  materialised — or was real but irrelevant, because the bottleneck was somewhere
  else entirely.
- **The strongest symptom is specific to you:** you have a Topic 65 baseline, Topic
  118 metrics, and Topic 119 traces, and the document cites none of them. That is not
  a gap in tooling. It is a choice not to look, and a reviewer who knows the tooling
  exists will read the whole document less charitably because of it.

**Root cause:** The author reasoned about the system's behaviour instead of
observing it. This is a strong habit and it is usually right, which is what makes it
dangerous — being usually right means the times you are wrong are unflagged.

**Fix:**

- **Every quantitative claim gets one of three labels:**
  `MEASURED` (with the source and the query),
  `ESTIMATED` (with the arithmetic shown),
  `ASSUMED` (with what would validate it and when).
  Three labels, mechanically applied. This is the single highest-leverage editing
  pass you can make on a technical document.
- **Cite the Topic 65 baseline by name** for any latency or throughput claim. If the
  baseline does not cover the scenario, say so — that is itself a finding, and it
  may mean the load suite needs a new scenario before the decision can be made.
- **State the target explicitly:** "we expect p99 for POST /orders to fall below
  `<___>` ms; if it does not, this change did not do what we said and we should
  reconsider." Now the change is falsifiable after the fact, which is what makes the
  next document you write more credible.
- For non-quantitative claims like "more maintainable", **convert to something
  observable or delete them.** "Maintainable" usually means "fewer places to change
  when X happens" — say that instead, and name X.

---

## Artefact — what you must produce

Write a **design doc for one significant `orderflow` change.** Candidate A from the
table above unless you have a better one; if you pick another, check it against the
four significance criteria first.

### Format and length

- **3 to 8 pages.** Under 3, it is an ADR and the decision was not big enough. Over
  8, you have written a specification and the decision is buried in it.
- Written so that reading only the Summary, the Trade, and the Reversibility sections
  gives an accurate picture. Those three sections are what a busy senior reader
  actually reads.
- Status, author, decider, reviewers, and review-close date in the header block.

### Required sections

1. **Header block** — status, author, named decider, reviewer list, review-close
   date, related documents.
2. **Summary** — 3 to 5 sentences, containing at least one cost.
3. **Context and problem** — with labelled evidence (E1, E2, …), each citing a real
   source in your system.
4. **Goals and non-goals** — at least three non-goals.
5. **Constraints.**
6. **Proposed design** — including the failure paths, each named and handled.
7. **Alternatives considered** — at least three, including "do nothing". Each with
   its strongest case *before* its rejection, and each with a "what would make this
   win" condition.
8. **The trade** — what we gain, what we give up, what we are not sure about.
9. **Reversibility** — two-way or one-way, the moment it changes, and the cost to
   reverse at each phase.
10. **What would change the answer** — concrete, checkable conditions.
11. **Who is affected** — a table with a consulted column containing names and dates.
12. **Rollout with rollback points** — phases with entry and exit gates, and an
    explicit go/no-go at the point of no return.
13. **Open questions** — each with an owner and a needed-by date.

### Required content — the specific things I will check for

- Every quantitative claim labelled `MEASURED`, `ESTIMATED`, or `ASSUMED`.
- At least one alternative you can argue for persuasively enough that a reader might
  choose it.
- At least one place where you say **"if this is true, this document should be
  rejected"** and name who owns that fact.
- The point at which the decision becomes irreversible, named as a date or an event,
  and treated as its own go/no-go.
- At least one consequence the framework imports that a naive doc would not mention
  — a pool-sizing implication, a pinning risk, a cache-invalidation contract, a
  cardinality constraint.
- A "we are not sure about" block.
- At least one named person in the consulted column who is not an engineer.

---

## How I will review it

I will not comment on your prose. I will attack the weakest assumption, the unstated
alternative, and the missing failure mode.

### The three questions that usually break a design doc

**Question 1: "What is the second-best option, and what would make it the best?"**

I want you to argue for the alternative — properly, with conviction — and then name
the specific condition under which it wins.

This breaks most design docs because most authors cannot do it. The alternatives
section was written to justify a decision already made, so the author has never held
the alternative as a live option and cannot argue it. What you get instead is a
restatement of the rejection.

What a good answer sounds like: *"Alternative 2 — bulkhead plus breaker. It is
smaller, cheaper, fully reversible, introduces no new states, and completely solves
the amplification problem, which is the one that has actually bitten us in testing.
It wins the moment the business says it is acceptable for order placement to be
unavailable when the payment provider is unavailable. I think they will not say
that, but I have not asked, and the person to ask is the head of product. If they
say yes, my proposal is over-engineering and this document should be rejected."*

Follow-ups I will pursue:

- "You said you have not asked. Why are you writing a design doc before asking the
  question that determines whether it is needed?" (Sometimes there is a good answer
  — you need the doc to make the question concrete. Sometimes there is not.)
- "Which alternative would the payments team pick, and why?" (Testing whether you
  have modelled other people's incentives, or only your own reasoning.)

**Question 2: "What would you have to see to reverse this, and what does reversing
cost in three months?"**

Two halves. The first tests falsifiability. The second tests whether you have
thought about time.

The failure mode: an author who says "we'd reverse it if it doesn't work" — which is
not a condition, because "doesn't work" is not observable. And an author who says
reversing is "just a flag" without noticing that by month three there are orders in
the new state, clients shipped against it, and support has trained staff on it.

Follow-ups:

- "Your rollback plan is a feature flag. What happens to the orders currently in
  `PENDING_PAYMENT` when you flip it?" This catches most rollback plans. Rollback of
  code is easy; rollback of *state* is the hard part and is usually unaddressed.
- "You say the point of no return is when clients ship. Who decides when to cross
  it, and what evidence do they need?" If it is not a named decision with named
  evidence, it will be crossed by accident during a rollout.
- "Three months in, the change is fine but not clearly better. Do you keep it or
  revert it? Which does your document tell you to do?" Ambiguous outcomes are the
  common case and almost no document plans for them.

**Question 3: "Who is worse off, and have they read this?"**

Every significant change makes someone worse off. If the document does not name
them, either the author has not looked or the change is not significant.

For candidate A, the answer includes support (new contact category), the client
teams (work they did not ask for), and finance (a new reconciliation obligation).
The document must name them, and the consulted column must have names and dates.

Follow-ups:

- "Support said it was fine. Did you show them the state machine or did you describe
  it?" There is a difference, and only one of them constitutes consultation.
- "The mobile team has a two-week sprint cadence and an app-store tail. Does your
  rollout plan account for that, or does it assume they will drop what they are
  doing?" Plans that assume other teams have free capacity are the most common
  cross-team failure.
- "What is your plan if one of these groups says no?" (Which is Topic 134's whole
  subject, and I will notice whether you have thought about it.)

### The other attacks, in order

**On the problem statement:**

- "Show me the measurement that establishes this is a problem." If the evidence is
  `ASSUMED`, the whole document rests on it and I will spend the rest of the review
  there.
- "You have separated P1 from P2. Which one has actually hurt us in production, and
  which one are you predicting? Are you solving the predicted one because it is more
  interesting?" This is a real and common failure and it is uncomfortable to be
  asked.

**On the design:**

- "Name the failure state that is possible after this change and impossible before
  it." If you cannot, you have not thought about the design's failure modes. For
  candidate A it is F1a and it must be in the document.
- "Which framework behaviour does this design depend on that a reader would not
  know?" I am looking for the imported consequence: the pool implication, the
  proxying implication, the cardinality constraint.
- "How is this tested?" Specifically: how is the compensation tested, and how is the
  compensation's failure tested? Async paths that only execute during incidents are
  the least-tested code in any system.

**On the alternatives:**

- "There is an alternative missing. What is it?" Sometimes there genuinely is one —
  most often the "do nothing but measure more first" option, or the "solve a
  different problem that makes this one irrelevant" option. A good author has
  considered and dismissed the second kind explicitly.
- "You rejected alternative 3 on a correctness risk. Quantify it or admit it is a
  judgment." Both are acceptable answers. Pretending a judgment is a measurement is
  not.

**On the whole document:**

- "Which paragraph is weakest, and why did you leave it in?" A Principal author
  knows. If you say "none of them", I will find one and the review gets worse from
  there.
- "This says `ACCEPTED`. Who accepted it and on what date?" Process hygiene, and it
  reveals whether the decision actually happened or whether the status was
  self-applied.
- "In two years, a new engineer reads this and wants to reverse it. Does the document
  tell them what they would need to check?" That is the entire long-term purpose of
  the artefact, and most documents fail it.

---

## Interview questions (Senior → Principal)

The rubric line for this topic: *a doc that explains the chosen design → a doc that
makes the decision legible: what we're trading, what we'd need to see to reverse it,
and who is affected.*

---

### Q1 — "Walk me through a design doc you wrote."

**Senior answer:** "I wrote the design for our payment retry system. It covered the
problem, the architecture, the data model, the API, and the rollout plan. It went
through review, we got some good feedback about error handling, and we implemented
it roughly as designed."

Fine. It describes a document. It says nothing about a decision.

**Principal answer:** "The most useful one I wrote was about whether order placement
should call the payment gateway synchronously. The interesting part was not the
design — it was that there were two different problems tangled together and the
document's main contribution was separating them. One was that a slow gateway
exhausted our connection pool and took down endpoints that had nothing to do with
payments. The other was that a *down* gateway made order placement unavailable,
which caps our SLO no matter how careful we are.

They have different fixes, and the cheap fix solves only the first. So the document
laid out four options, and I wrote the strongest case for the cheapest one —
bulkhead and circuit breaker — because it genuinely might have been right. It wins
if the business accepts that we are unavailable when our payment provider is
unavailable. I wrote that condition explicitly and said that if the head of product
said yes to it, the document should be rejected.

The other thing I would highlight is the reversibility section. The change was
reversible with a flag until external clients shipped support for a new order state,
and irreversible after. So I split the rollout so that the irreversible transition
was its own go/no-go decision with its own evidence gate, rather than a step in a
rollout plan that someone would cross without noticing.

What I got wrong: I under-consulted support. They came back with a concern about
contact volume that I had no data for, and I had to add a canary phase to measure it
rather than guessing."

**What separates them:** the Principal answer is about a decision, not a design.
It names the intellectual contribution (separating two problems), argues the
rejected option honestly, names the condition that would have killed its own
proposal, treats reversibility as a design lever rather than a section, and
volunteers a mistake without being asked.

**Adversarial follow-up:** *"Separating those two problems sounds like something you
worked out afterwards, in hindsight, and put in the doc to look clever."*

Do not get defensive; that is the actual test. "Fair challenge. The order I actually
worked in was: I started writing the doc intending to propose the async change,
because that was my instinct. Writing the alternatives section is where I noticed
that the bulkhead option solved the thing that had actually broken in our load
testing and my proposal was aimed at a thing that had not broken yet. That is
uncomfortable to notice halfway through writing, and it changed the document from an
advocacy piece into something more honest. If I had decided first and written
second, I would have missed it — that is most of why I write the alternatives
section first now."

That answer concedes the possibility, describes the actual sequence, and converts
the challenge into evidence for the underlying practice.

---

### Q2 — "How long should a design doc be?"

**Senior answer:** "Long enough to cover the design, short enough that people read
it. Maybe three to five pages."

**Principal answer:** "It should be proportional to the cost of being wrong, and it
should be as short as that allows.

Practically I use three sizes. A half-page ADR when the decision is bounded and has
an obvious default — key generation strategy, a library choice with a clear winner.
A three-to-eight-page design doc when the decision affects another team, is
expensive to reverse, or has genuine alternatives. A longer strategy doc when the
thing spans quarters and implies a hiring plan.

Getting the size wrong in either direction is a real failure. Too long for a small
decision and people stop reading your documents, which costs you on the one that
matters. Too short for a big one and you have made an irreversible decision without
a review surface.

The thing that actually controls length is the design section, because it is the
part everyone is comfortable writing and the part reviewers are comfortable
commenting on. I deliberately keep it shorter than feels natural, because every page
of design detail pulls attention away from the trade-off, and the trade-off is the
part the document exists for. Implementation detail belongs in the PR."

**What separates them:** a rule tied to consequence rather than a number, the
observation that document length has a cost paid by *future* documents, and the
specific insight about the design section crowding out the trade.

**Adversarial follow-up:** *"Our team requires a full template for every change. So
your three sizes don't apply here. What do you do?"*

"Then I would follow the template — arguing about process before I have credibility
is a bad trade, and templates exist because someone was burned. But I would fill it
proportionally: the sections that do not apply get one honest line saying so, rather
than padding. 'Reversibility: two-way door, flag flip, minutes' is a complete answer
and it makes the template cheap for small decisions. Then after I had shipped a few
things well, I would raise the ADR-sized variant with evidence — specifically, the
average time from draft to decision, and how much of it was spent on documents where
the decision was never in doubt. That is Topic 132's problem: a standard gets adopted
when following it is cheaper than ignoring it, and a heavy template on a small
decision is a standard people route around."

---

### Q3 — "A reviewer disagrees with your central choice and will not move. What do you
do?"

**Senior answer:** "I'd set up a call to talk it through, try to understand their
concerns, and find a compromise or escalate to the tech lead if we can't agree."

**Principal answer:** "First I would work out which of three things is happening,
because they need different responses.

If we disagree about a **fact** — is the gateway's availability above our target —
that is the good case. Facts are resolvable. I would stop arguing, write down the
measurement that settles it, and go and run it. Converting a position disagreement
into a measurement is the single most useful move available and it usually ends the
conversation.

If we disagree about a **value** — how much we care about a simple mental model
versus availability — no amount of discussion resolves it, because we are weighting
things differently. That goes to the decider, with both positions written down
fairly. My job is to state their position well enough that they would endorse my
statement of it. If they read my summary of their view and object to it, I have not
done that job.

If they have **information I lack** — which is the case often enough to always check
first — then I am wrong and the design should change. I would rather find that in
review than in production. Asking 'what do you know that I don't?' early is cheap
and it is the question people are usually waiting to be asked.

In all three cases I ask them one specific question: what evidence would change your
position? If they can name it, we have work to do and it is productive. If they
cannot, it is a preference, which is legitimate and informs the decider but does not
block. And I would say that out loud rather than treating it as a private
classification."

**What separates them:** the three-way diagnosis, the specific move of converting
fact disagreements into measurements, the standard of stating the opponent's
position well enough that they would endorse it, and treating "they might be right"
as the first hypothesis rather than the last.

**Adversarial follow-up:** *"The decider sided with you. Six weeks later the
reviewer's concern turns out to have been correct and the change causes an incident.
What now?"*

"Then I write it up as a postmortem where a contributing factor is 'a review concern
was correctly raised and incorrectly weighted', and I name that I was the one who
weighted it. Not as self-flagellation — as a contributing factor with an action, and
the action is a process one: for decisions in this class, a reviewer who disagrees
gets their objection recorded in the document as a named risk with a monitoring
signal attached, so that if they are right we find out early rather than through an
incident.

The other thing I would do, immediately and privately, is tell the reviewer they
were right. Publicly if they would want that. The cost of not doing that is that
they stop raising concerns, and I have lost the most valuable reviewer I had."

That answer is Topic 133 and Topic 134 arriving early, which is exactly what a good
interviewer is checking for.

---

### Q4 — "What do you do when you're wrong halfway through writing the doc?"

**Senior answer:** "I'd update the doc to reflect the better approach."

**Principal answer:** "That is the doc working, so first I would notice and say so
rather than quietly rewriting.

Mechanically: if the draft has not circulated, I rewrite and the new design becomes
the proposal — but I keep my original idea in the alternatives section with the real
reason it lost, because that reasoning is the most valuable thing I produced. Future
readers benefit from it far more than from a document that only ever contained the
right answer.

If it has circulated, I say explicitly in the next version: 'Section 5 changed
because of `<reviewer>`'s point about X; the previous proposal is now alternative 4.'
Two reasons. It credits the person, which makes them and everyone watching more
likely to engage next time. And it demonstrates that review changes outcomes, which
is the only thing that keeps a review culture alive — people spend effort on review
exactly to the degree they believe it matters.

The failure mode I watch for in myself is quietly changing the design and presenting
version two as if it were always the plan. It feels smoother and it is corrosive,
because it makes the review look decorative even when it was not."

**What separates them:** treating being wrong as the process succeeding, keeping the
dead idea as recorded reasoning, crediting the reviewer explicitly, and naming the
specific self-serving instinct that must be resisted.

**Adversarial follow-up:** *"Doesn't that undermine your authority? You proposed
something and then abandoned it."*

"The opposite, in my experience, and I would rather be right than look consistent.
The thing that undermines authority is defending a position past the evidence,
because everyone in the room can see you doing it and they update on it permanently.
Changing position when the pushback is right, and saying why, is what makes people
believe you the next time you *don't* change position — which is the currency I
actually need. And there is a practical version of this: I want people to bring me
their disagreement early and cheaply. That only happens if disagreeing with me has
previously worked for them."

---

### Q5 — "Your organisation has no design-doc culture. Would you introduce one?"

**Senior answer:** "Yes — I'd propose a template and ask people to use it for
significant changes."

**Principal answer:** "Not as a policy, and not first. A template imposed on a team
that does not want one produces compliance documents: filled in after the decision,
skimmed by nobody, and resented. Then the practice is poisoned and reintroducing it
later is harder.

What I would do is write two or three docs myself on decisions that genuinely
mattered, and make the review of them visibly useful — specifically, I would make
sure at least one of them visibly changed as a result of a review comment, and I
would credit that comment loudly. Then the argument is not 'we should write
documents', it is 'that thing where Priya caught the pool-sizing problem before we
built it — can we do that again?'

Then I would make it cheap. The template goes in the repo. The half-page ADR variant
exists and is used for small things, so nobody is ever forced to write four pages
about a library choice. If we have a tool, the doc links to the ticket automatically.
Every unit of friction I remove is worth more than any amount of advocacy.

And I would explicitly not require it. A rule you cannot enforce teaches people that
your rules are optional, which costs you on the rule that matters. Better a practice
that a third of the team uses because it visibly works than a policy everyone
performs.

That is really Topic 134's mechanism applied to a process rather than to a technical
decision: make adoption cheaper than non-adoption and the argument becomes
unnecessary."

**What separates them:** refusing the mandate, making the first instance visibly
valuable, removing friction as the primary lever, and the explicit statement that an
unenforced rule has a cost beyond itself.

**Adversarial follow-up:** *"Six months in, two people write docs and nobody else
does. Has it failed?"*

"Depends entirely on which two people and which decisions. If the two people are
writing docs for the decisions that are expensive to reverse, the practice is
working and the coverage number is the wrong metric. If the expensive decisions are
still being made in Slack threads by everyone else, then yes, it has failed on the
thing that mattered.

So I would go and look at the last five expensive decisions and ask whether each one
had a document. That is the measurement. If the answer is two out of five, I would
find out what was different about the three — usually it is time pressure or that
nobody realised it was a big decision until later. The second one is fixable with a
lightweight trigger: 'if you are about to make a change that is expensive to reverse,
write the half-page version first.' The first one is not fixable with process and I
would stop trying."

---

## Mental model checkpoint

1. A design doc is not updated after acceptance; it is superseded. But a
   specification must be kept current. Where exactly is the boundary between the two
   for `orderflow`'s payment flow, and what goes wrong if you get the boundary wrong
   in each direction?

2. You are asked to write the alternatives section for a decision where you genuinely
   believe there is only one viable option. Is the correct response to write weak
   alternatives, to write "no viable alternatives", or to conclude the document
   should be an ADR instead? Argue for one, then make the strongest case against
   yourself.

3. Reversibility is presented here as a property of the decision. Give an example
   where reversibility is instead a property of the *implementation* — where two
   implementations of the same decision have different reversal costs. What does that
   imply about which section it belongs in?

4. "What would change the answer" makes you falsifiable. Name a situation where
   writing that section would be actively harmful to getting the right outcome, and
   say whether you would write it anyway.

5. Your document names a decider. That person is less technically deep than you on
   this subject. What is their actual job in the review, and what should you write
   differently because of it?

6. A design doc's value is realised mostly in the future, by other people. That means
   the author bears the cost and someone else gets the benefit. What organisational
   mechanisms make that trade sustainable, and which of them can a Principal engineer
   create without authority?

7. Compare a design doc's alternatives section with the SLO document's negotiation
   position from Topic 130. Both state a position and what would change it. What is
   structurally different about them, and why does one belong in the published
   document while the other does not?

---

## Quick reference card

### The four things that make a decision legible

1. What we are trading (not what we are building).
2. Alternatives, each with the reason it was rejected.
3. Reversibility, with a cost and a point of no return.
4. What evidence would change the answer.

### Section checklist

- [ ] Header: status, author, **named decider**, reviewers, review-close date
- [ ] Summary in 3–5 sentences, containing at least one cost
- [ ] Problem with labelled evidence, each citing a real source
- [ ] Goals **and at least three non-goals**
- [ ] Constraints
- [ ] Proposed design, including named failure paths
- [ ] Alternatives — strongest case *before* rejection; "what would make this win"
- [ ] The trade: gain / give up / **not sure about**
- [ ] Reversibility with per-phase cost and a named point of no return
- [ ] What would change the answer
- [ ] Who is affected, with a consulted column containing names and dates
- [ ] Rollout with rollback points and a go/no-go at the irreversible step
- [ ] Open questions with owners and dates

### Evidence labels — apply mechanically

| Label | Means | Must include |
|---|---|---|
| `MEASURED` | we observed it | source, query, window |
| `ESTIMATED` | we calculated it | the arithmetic |
| `ASSUMED` | we believe it | what would validate it, and when |

### Document size by cost of being wrong

| Cost of a wrong choice | Artefact | Length | Decider |
|---|---|---|---|
| A week | ADR | half a page | the team |
| A quarter | Design doc | 3–8 pages | a named individual |
| A year plus hiring | Strategy doc | 5–15 pages | a leader with budget |

### Status values

`DRAFT` → `IN REVIEW` → `ACCEPTED` / `REJECTED` → `SUPERSEDED BY <doc>`

Never delete a superseded doc. The chain is the reasoning record.

### Anti-patterns

- Alternatives written to be dismissed.
- Doc written after the code.
- No non-goals, so review never converges.
- No named decider, so nothing closes.
- "Reversible with a feature flag" with no plan for in-flight state.
- Quantitative claims with no label.
- An empty consulted column.
- No open questions on a large change.
- A design section that is longer than everything else combined.

---

## When would I use this at work?

**1. Before a change that is expensive to reverse.**

The trigger is not size and it is not effort — it is reversal cost. A two-week change
that is trivially revertible needs no document. A two-day change that alters a
published API contract needs one. Training yourself to notice reversal cost rather
than implementation cost is most of the skill, and it is the thing that stops you
writing documents nobody needs while missing the one that mattered.

**2. When a decision keeps getting relitigated.**

If the same argument has happened three times, the problem is not that people
disagree — it is that the reasoning was never recorded, so each new participant
starts from zero. Writing it down once, including the rejected alternatives and why,
converts an infinite conversation into a document you can link. The cost is an
afternoon and the saving is unbounded.

**3. When you need someone else to be able to disagree with you.**

This is the underrated use. If you are about to make a decision that affects teams
you cannot talk to individually, the document is how they get a chance to object
before it is expensive. And selfishly: it is how you find out you were wrong while
the cost of being wrong is still a rewrite rather than a rollback. The moment you
notice you are avoiding writing the alternatives section, that is the moment you most
need to.

---

## Connected topics

**Prerequisites — where the evidence in your doc comes from:**

- **65 — GATE: service under load.** Every performance claim in a design doc must
  trace back to this baseline or be labelled `ASSUMED`.
- **118 — Metrics.** Your evidence for "this is a problem", and a source of imported
  constraints (bucket placement, cardinality) that a good doc names.
- **119 — Tracing.** How you attribute latency to a specific dependency, which is
  what turns "the gateway is slow" into a number.
- **124 — Production-readiness review.** That document identified risks with owners;
  several of those risks are the design docs you should be writing next.
- **125 — Reading the source.** A design doc that asserts framework behaviour is
  weaker than one that cites the code path. When a reviewer challenges "does Spring
  actually do that?", reading the source is the answer.
- **126 — Build vs buy vs adopt.** A build/buy decision is a design doc with a
  specific alternatives structure. Everything here applies to it.
- **127, 128 — Migration planning.** Both are design docs at strategy size. The
  reversibility section is the most important part of each, because migrations are
  where irreversible steps hide inside sequences that look routine.
- **129 — Capacity and cost.** Supplies the arithmetic for `ESTIMATED` claims and for
  the cost side of any trade.
- **130 — SLO design.** The dependency-ceiling arithmetic is often the evidence that
  establishes the problem. The SLO document and the design doc are read together.

**Design-relevant failure knowledge your doc should demonstrate:**

- **55** — a network call inside a transaction pins a pool connection. The evidence
  base for candidate A.
- **109** — pool sizing and the pool-vs-thread-pool deadlock. The imported
  consequence of any `REQUIRES_NEW` in a design.
- **111** — retries without jitter amplify outages. Any retry policy in a design doc
  must address this or a reviewer should reject it.
- **115** — the dual-write window. Any design that writes locally and then calls a
  remote system must say what happens to the gap.
- **116** — idempotency. Any retry policy that lacks an idempotency key is unsafe,
  and the design doc is where that gets caught.
- **117** — sagas and compensations. The compensation's own failure mode is the state
  a good design doc names explicitly.

**This unlocks:**

- **132 — Engineering standards.** A standard is proposed with a design doc, and the
  rollout plan section is where the ratcheting-baseline idea lives.
- **133 — Postmortems.** A postmortem is a design doc pointed backwards: contributing
  factors instead of alternatives, action items instead of a rollout plan.
- **134 — Influence without authority.** The document is your main instrument for
  influencing people you will never meet. The "who is affected / consulted" table is
  where Topic 134's work shows up inside Topic 131's artefact.
- **135 — Principal interview simulation.** This design doc is one of the artefacts
  you will defend under sustained hostile questioning. Question 1 will be "what is
  the second-best option, and what would make it the best?"

---

*Nothing in this topic contains invented measurements, adoption statistics, or
described-as-real review histories. Every table cell describing your system is blank
by design. The document you write is worth exactly as much as the evidence you put
behind it — and the single fastest way to lose a design review is to be caught
asserting a number you did not measure.*
