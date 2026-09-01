# 124 — **GATE** — Production-Readiness Review of `orderflow`

## Phase: 11 — GATE
## Category: DIFFERENTIATOR
## Java baseline: 21  |  Notes features from: 21/25
## Project spine: a written production-readiness review of `orderflow` — SLOs and attainment, capacity headroom at the Topic 65 baseline, every Phase 8–11 failure mode with its mitigation and runbook entry, observability gaps, and the top three risks with named owners.

---

## Before anything else — what is and is not in this document

**What is in it.** A method for writing a production-readiness review, a blank
template you fill in, a register that maps every failure drill you ran in Phases
8–11 to a named failure mode and a runbook entry, five ways this document fails in
real organisations, and the specific attacks I will make on yours.

**What is not in it, deliberately.**

- **No numbers about your system.** Not one. No p99, no throughput, no headroom
  percentage, no attainment figure, no capacity ceiling. Every table in the template
  is blank. The numbers come from *your* Topic 65 baseline, *your* Topic 118 metrics,
  and *your* drill history. If I supplied a plausible-looking number here you would
  anchor on it, and the anchor would be fiction.
- **No industry benchmarks.** I will not tell you what "good" p99 is for an order
  API. It depends on your product, your users, and your competition, and any figure
  I invented would be worse than no figure.
- **No claim that this is a standard format.** Production-readiness reviews are a
  widely used practice; Google's SRE literature popularised the term and the related
  error-budget model, and many large engineering organisations run something like
  it under names such as "launch review", "operational readiness review", or
  "go-live checklist". The specific template below is mine, built from this
  curriculum's drills. I am not quoting anyone and I am not reproducing any
  company's internal checklist.
- **No code.** The deliverable is a written document. There is no coding exercise in
  this topic. That is not a softening of the gate; it is the point of it.

**What I am asking you to trust.** That a review whose every claim names its
evidence is worth more than a longer review whose claims are confident. The rest of
this document is an argument for that, plus the machinery to do it.

---

## THE GATE RULE — read this first

> **If a claim in your review cannot name the measurement that supports it and the
> measurement that would falsify it, delete the claim or go and take the
> measurement. Phase 12 is not startable on a review made of assertions.**

That is not style advice. Phase 12 is eleven topics of judgment under ambiguity —
capacity models, migration plans, SLO negotiations, postmortems, adoption plans.
Every one of them is a document whose value is exactly the quality of the evidence
underneath it. If you can write "we handle load fine" without flinching, you will
write a capacity model with the same reflex, and it will be wrong in a way nobody
can check.

This gate is the rehearsal. It is the last topic where I hand you the structure. In
Phase 12 you build the structure yourself.

---

## Mechanical statement

**A production-readiness review is an artefact, not an opinion. Every claim in it
must name the evidence that supports it and the measurement that would falsify it.**

Three consequences follow, and they are the whole document:

1. **A claim with no evidence is a risk entry, not a finding.** "We handle the load"
   with nothing behind it does not become true by being written down. It becomes a
   line in the risk register that reads: *we do not know whether we handle the load,
   owner X, measurement due Y.* That is a more useful document than the confident
   one, and it is shorter.
2. **A claim with no falsifier is unmaintainable.** Six months from now the system
   has changed. If the review says "p99 is within budget" but does not say *at what
   arrival rate, against what dataset, measured where*, nobody can tell whether it is
   still true. A review that cannot rot visibly will rot invisibly.
3. **Every known failure mode must appear, mitigated or accepted, with a named
   owner.** A failure you produced on purpose in a drill and then left out of the
   review is worse than one you never found. You knew.

Everything else here is consequence.

---

## The bridge from what you know

### What transfers directly — and I am not going to re-teach it

You have shipped Node services to production. You have been on call. You already
know, and I will not spend a line teaching you:

- What a percentile is and why an average hides the tail.
- That "it works on staging" is not evidence about production.
- How to run a review meeting, take an action list out of it, and chase it.
- That a risk with no owner is not tracked.
- That a rollback plan you have never executed is a hope.
- How to write a document for an audience that will skim it.

All of that carries over unchanged. Writing this review is not a new *writing*
skill. If you have produced a launch checklist for a NestJS service, the shape is
familiar.

### What is genuinely harder here

Two things, and they are the reason this is a gate rather than a checklist.

**One: making the claim legible instead of confident.** A Senior engineer writes a
readiness review that is *accurate*. A Principal engineer writes one where a reader
who disagrees can find the exact thing to attack. That means every assertion carries
its source and its falsifier. It reads less impressively and it is worth far more,
because it survives contact with a hostile reviewer, and because in six months it
tells the truth about whether it is still true.

This is the skill that transfers *least* from a TypeScript launch checklist,
because most launch checklists are tick-boxes. A tick-box records that somebody
looked. It does not record what they saw.

**Two: the failure modes are JVM-shaped and you cannot reason about them from
first principles.** This is the honest reason a Node engineer's readiness review of
a Java service is usually thin. In Node, your failure vocabulary is: event-loop
block, unbounded memory growth, socket exhaustion, unhandled rejection. It is a
short list and you know it by heart.

The JVM's list is longer and less intuitive, and several entries have no Node
analogue at all:

- A stop-the-world pause that is long because of **time-to-safepoint**, not because
  of garbage collection work.
- A **humongous allocation** that fragments a G1 heap and triggers a full collection.
- **Two pools deadlocking against each other** — a request thread holding a database
  connection while waiting for a second one.
- A `synchronized` block **pinning a virtual thread to its carrier** and starving a
  scheduler sized to your core count, while CPU sits idle.
- **`ThreadLocal` retention on a pooled thread** that grows to a bound and stops,
  which is exactly why it gets dismissed as "not a leak".
- A **rebalance loop** because a consumer took longer than its poll interval.
- **Metric cardinality** taking down the monitoring system rather than the service.

You produced every one of those on purpose in Phases 8–11. That is what makes you
able to write this review and makes a Node engineer who has not done the drills
unable to. The register in "Machine-level reality" is that vocabulary, written out.

### The Java-specific evidence base you already built

- **Topic 65** — the baseline. Every capacity and latency claim traces to it.
- **Topic 109** — the pool ceiling. Your recorded latency-versus-pool-size curve is
  the single most load-bearing number in the capacity section.
- **Topics 118 and 119** — metrics and traces. The evidence base for attainment,
  for saturation, and for attributing latency to a dependency.
- **Topic 121** — probes. Whether readiness is honest is a readiness-review item in
  the most literal sense.
- **Topic 123** — graceful shutdown. Deploy behaviour is a reliability property, and
  at high targets it is most of the budget.

---

## What is this?

A **production-readiness review** (PRR) is a written assessment, produced before or
shortly after a service takes production traffic, that answers one question:

> *What do we know about how this system behaves, what do we not know, and what are
> we choosing to accept?*

It has five obligatory parts. Define each precisely, because vagueness in the
definitions is where these documents go soft.

**1. Service level objectives and current attainment.** What the system promises,
measured how, over what window — and what it is actually delivering. The gap
between the two is the interesting number. A PRR that lists targets without
attainment has told you nothing.

**2. Capacity headroom.** How far the system is from its ceiling, at a *stated*
load, with the ceiling's *cause* named. "We have headroom" is not a capacity claim.
"At the Topic 65 arrival rate the pool is the binding constraint at N in-flight
requests, so headroom to the next constraint is X" is one.

**3. The failure-mode register.** Every way you know this system breaks, each with:
what it looks like from outside, what detects it, what mitigates it, and where the
runbook entry is. "Every way you know" is the operative phrase — it is not every
way it *can* break, which is unknowable. It is everything you have observed, and
you have observed a great deal, because Phases 8–11 were nothing but that.

**4. Observability coverage gaps.** Not "we have monitoring". Specifically: for each
failure mode in the register, is there a signal that would show it, and would that
signal reach a human? A failure mode with no detection is not mitigated, however
good the fix is, because nobody will apply the fix.

**5. Top three risks, with named owners.** Three, not fifteen. A risk register of
fifteen items is a list nobody reads and nobody owns. Three is a number a director
can hold in their head, and each one must have a person's name, a date, and a
statement of what would make it worse.

**What a PRR is not:**

- Not a design doc. It does not argue for a design; it assesses one that exists.
  (Topic 131 is the design doc.)
- Not a checklist. A checklist records that someone looked. A review records what
  they saw.
- Not a sign-off ritual. If the only possible outcome is "approved", it is theatre,
  and everyone in the room knows it.
- Not a promise of correctness. It is a statement of what is known, and the *known
  unknowns* section is the part senior readers turn to first.

### Terms used precisely in this document

- **Evidence** — a number, a log, a trace, a drill result, or a document you can
  point at. A memory of a conversation is not evidence.
- **Falsifier** — the specific measurement that, if it came out differently, would
  make the claim false. Every claim has one or it is not a claim.
- **Failure mode** — a named way the system misbehaves, defined by its *external
  symptom*, not by its internal cause. "p99 climbs while throughput stays flat and
  CPU sits idle" is a failure mode. "Pinning" is a cause.
- **Mitigation** — a change that makes the failure less likely, less severe, or
  faster to detect. Notice detection counts.
- **Runbook entry** — a written procedure that a person who did not build the system
  can follow at 03:00 to confirm the failure and act on it.
- **Accepted risk** — a known failure mode with no mitigation, recorded with the
  reason and the person who accepted it. Accepting risk is legitimate. Accepting it
  silently is not.

---

## Why does it matter?

### 1. Because the alternative is discovering your failure modes in front of customers

You already know every failure mode in this register, because you built each one
deliberately, watched it, and fixed it. That is an unusual position. Most teams meet
these for the first time during an incident, at 02:40, with a director in the
channel.

The review is how that knowledge stops living in your head. When you go on leave,
the register is what remains.

### 2. Because "we handle load" is a claim with no content, and it is the most common sentence in these documents

Handle what load? At what arrival rate? Against what dataset size? With what cache
state? Measured client-side or server-side? Was the load generator itself saturated?

Topic 65 exists so that you can answer all six. A readiness review that does not use
that answer is wasting the most expensive artefact in the curriculum.

### 3. Because it converts drills into organisational memory

A drill you ran and fixed teaches you something. A drill you ran, fixed, and wrote
into a register with a detection signal and a runbook entry teaches *the team*
something, permanently. The difference between a strong engineer and a strong
engineering organisation is almost entirely this conversion.

### 4. Because it is the rehearsal for every Phase 12 artefact

Look at what Phase 12 asks for: a capacity model (Topic 129), an SLO document with a
negotiation position (130), a design doc with alternatives and reversibility (131), a
standards rollout plan (132), a postmortem with contributing factors (133), an
adoption plan (134). Every one is a document whose value equals the evidence beneath
it. If you can write this review honestly, you can write those. If you cannot, you
will write those in the confident register and they will be unfalsifiable.

### 5. Because the risk section is where you find out whether the organisation is honest

Writing "the top risk is that we have never tested a Postgres failover, owner: me,
by the 30th" is easy in a document and hard in a room. If the culture punishes that
sentence, you have learned something more important than the answer.

---

## Machine-level reality

This section is the JVM-specific core of the review, and the thing a readiness
review of a Java service must contain that a generic template will not prompt you
for.

Each Phase 8–11 drill produced a **named failure mode**. A failure mode is defined by
its external symptom, because that is what an operator sees. The cause is what the
runbook helps them confirm.

The register below names the modes. **The mitigation, detection and runbook columns
are yours to fill** — I do not know what you built, and inventing it would defeat
the exercise.

### The register — Phase 8 (memory, GC, JVM behaviour)

| # | Failure mode (external symptom) | Cause you produced | Drill |
|---|---|---|---|
| F1 | Heap grows across the run, then `OutOfMemoryError`; restart resets it | Unbounded `static Map` cache retaining every order | 79 |
| F2 | Memory sits high but stable at pool-size × context-size; dismissed as "not a leak" | `ThreadLocal` request context never removed on a pooled thread | 79 |
| F3 | RSS grows while heap stays flat; container OOMKilled with a healthy heap graph | Direct buffers / native allocation outside the heap | 80 |
| F4 | One multi-second pause under load, unrelated to heap fullness | Humongous allocation larger than half the G1 region size | 71 |
| F5 | A pause much longer than the reported GC work | Time-to-safepoint: a counted `int` loop with no safepoint poll | 73 |
| F6 | Throughput drops after a deploy that changed no hot code | Deoptimisation after a call site became bimorphic | 74 |
| F7 | Container killed at start-up or default flags chosen wrongly | JVM not container-aware / no explicit heap and CPU flags | 82 |
| F8 | Works on the JVM, fails at runtime on the native build | Reflection without reachability metadata | 83 |

### The register — Phase 9 (concurrency)

| # | Failure mode (external symptom) | Cause you produced | Drill |
|---|---|---|---|
| F9 | Latency climbs, throughput flat, then OOM under sustained overload | Unbounded work queue in front of a fixed pool | 90 |
| F10 | Unrelated parallel work stalls whenever one endpoint is slow | Blocking JDBC submitted to the common ForkJoinPool | 91 |
| F11 | A wallet is debited twice for one idempotency key | Check-then-act race on the idempotency cache | 92 |
| F12 | Requests hang indefinitely; thread dump shows two threads blocked forever | Lock-ordering deadlock between wallet and inventory | 94 |
| F13 | Thread count grows monotonically until `OutOfMemoryError: unable to create native thread` | An `ExecutorService` constructed per request | 98 |
| F14 | Throughput collapses while CPU is idle and requests queue | Virtual threads pinned by `synchronized` across the payment-gateway call, starving carriers | 101 |
| F15 | Contention shows as monitor-blocked time on one hot path | `synchronized` on the inventory decrement | 85 |
| F16 | A shutdown flag is set and the loop never exits | Non-`volatile` stop flag; visibility, not liveness | 86/87 |

### The register — Phase 10 (reactive and backpressure)

| # | Failure mode (external symptom) | Cause you produced | Drill |
|---|---|---|---|
| F17 | Heap grows during a downstream slowdown until OOM, with no error until the end | Unbounded push source bridged into a `Flux` with a slow consumer | 105 |
| F18 | Logs for one request have no correlation ID after an async boundary | Context lost across `flatMap` / thread hand-off | 108 |

### The register — Phase 11 (distributed and production)

| # | Failure mode (external symptom) | Cause you produced | Drill |
|---|---|---|---|
| F19 | All request threads parked in `getConnection`; the service is up and serving nothing | Pool-vs-pool deadlock: nested `REQUIRES_NEW` with pool size ≤ concurrency | 109 |
| F20 | A downstream blip becomes a sustained outage; load on the dependency rises as it recovers | Retry storm without jitter, no breaker, no bulkhead | 111 |
| F21 | Consumer lag grows while the group rebalances repeatedly | Processing slower than `max.poll.interval.ms` | 113 |
| F22 | An order exists in the database with no corresponding event downstream | `kill -9` between commit and publish — the dual-write gap | 115 |
| F23 | The monitoring system degrades or the scrape fails; the service is fine | Metric cardinality explosion from an unbounded tag | 118 |
| F24 | A child span appears as a new trace; the trace is broken at an async hop | Context not propagated across an executor boundary | 119 |
| F25 | Every pod restarts together when the database blips | Liveness probe checking a database-dependent health group | 121 |
| F26 | Requests dropped during every rolling deploy | Readiness not false before drain, or grace period shorter than p99 | 123 |

### Why the register must be organised by symptom

At 03:00 the operator has a symptom, not a cause. They see "throughput collapsed and
CPU is idle". A register indexed by cause ("pinning") is unusable to them; they
would have to already know the answer to find it.

So the register is a **symptom-to-runbook index**. That is its operational purpose.
Its review purpose is different: it forces you to state, for each mode you have
personally produced, whether anything currently detects it.

That second purpose is where readiness reviews earn their keep. It is very common to
have a good mitigation and no detection. F2 is the canonical example: retention
bounded at pool-size × context-size will never page anyone, and will quietly consume
heap you needed for something else.

### Failure modes you should notice are absent

Two categories, both worth stating explicitly in your review:

- **Failure modes you have not drilled.** Postgres failover. An availability-zone
  loss. A certificate expiry. A poison message that fails deterministically forever.
  A schema migration that locks a hot table. You have not produced these. Say so.
  They belong in "known unknowns", not in the register as though you had checked.
- **Failure modes with no runbook because the mitigation is structural.** If you
  fixed F11 with `putIfAbsent`, the race is gone rather than mitigated. Those still
  belong in the register — with the mitigation named as structural and a note about
  what would reintroduce it (a new code path that reads then writes). This is how a
  register stops being a museum and starts being a review item.

---

## Example 1 — a minimal illustration

The smallest possible version of the mechanic, so the shape is clear before the big
template arrives.

### The service

A single endpoint that returns a product by ID from Postgres, with a Redis cache in
front. One instance. No queue, no events.

### Version A — the review as most people write it

> **Capacity:** the service handles our expected traffic with headroom.
> **Reliability:** we have a cache, so database load is low.
> **Monitoring:** we have dashboards and alerts.
> **Risks:** none blocking.

Every sentence is defensible in a meeting and none of them is checkable. There is
nothing here a reviewer can attack, which people mistake for strength. It is the
opposite: it is a document that cannot be wrong because it does not say anything.

Notice also that it cannot rot. In six months, when a second caller has doubled the
read rate, every sentence above is still exactly as true as it was — which is to
say, still contentless.

### Version B — the same review, with evidence and falsifiers

> **Capacity.** At the arrival rate recorded in `baselines/<date>/baseline.md`
> (`MEASURED`), the binding constraint is the connection pool: `hikaricp.connections.pending`
> becomes non-zero before CPU reaches its limit. Headroom to the next constraint is
> therefore a pool question, not a CPU question.
> *Falsifier:* if a re-run at 1.5× arrival rate shows CPU saturating before
> `pending` becomes non-zero, this claim is wrong and the capacity model changes.
>
> **Cache.** Hit ratio over the measured window is in `baseline.md` (`MEASURED`).
> Database load under a **cold** cache has not been measured (`GAP`). The stampede
> path on a mass eviction is untested.
> *Falsifier:* a cold-start run at baseline arrival rate that does not exceed the
> pool's capacity would close this gap.
>
> **Detection.** Pool saturation is alerted on `pending > 0` sustained for 1 minute.
> Cache-miss-rate change is on a dashboard but **not** alerted (`GAP`).
>
> **Top risk.** Cold-cache behaviour is unknown and is the most likely cause of a
> first outage. Owner: <name>. Test scheduled: <date>. Worsens if: a second consumer
> starts reading the catalogue, or the cache TTL is made uniform across keys.

Version B is longer, less impressive, and enormously more useful. Three things
changed:

1. Every claim names its source and its **falsifier**.
2. The gaps are marked `GAP` rather than smoothed over, so the document is honest
   about its own edges.
3. The risk has a name, a date, and a statement of what would make it worse — which
   is what turns it from a worry into a tracked item.

Also note what Version B does **not** do: it does not say "the cache is fine". It
says the hit ratio is measured and the cold path is not. Those are different
sentences and only one of them is true.

---

## Example 2 — the real decision on the project spine

Now the real artefact: the readiness review of `orderflow`.

Below is the **template**, section by section, with guidance on what a strong and a
weak answer look like in each. **Every table is blank.** Fill them from your own
Topic 65 baseline, your own Topic 118 dashboards, and your own drill notes.

---

### Header block

```markdown
# orderflow — Production Readiness Review

- Review date:
- Reviewed release / git commit:
- Author:
- Reviewers (name, role, date reviewed):
- Decision owner (the person who can say "not yet"):
- Baseline this review references: docs/java/baselines/<date>-run-NN/
- Next review due (date or trigger event):
- Status: [ ] Draft  [ ] In review  [ ] Accepted  [ ] Accepted with conditions
```

**Strong:** a named decision owner who is not the author, and a *next review
trigger* that is an event ("before the payments extraction lands") rather than only
a date. A review with an event trigger stays alive.

**Weak:** author reviews their own document; "next review: in six months" with no
event; no commit recorded, so nobody can tell later what was actually assessed.

---

### Section 1 — Summary and recommendation

Five sentences maximum. Must contain the recommendation, the single largest risk,
and the condition attached to the recommendation if there is one.

**Strong:** *"Recommend go-live with two conditions: the saturation alert on the
notification pool ships first, and Postgres failover is exercised within four weeks.
The largest risk is that we have never tested failover; if it does not work we have
no recovery procedure, only a hope. Everything else in the register is mitigated and
detected."*

**Weak:** *"orderflow is production ready."* No conditions, no largest risk. A
summary with no condition in it is a summary that did not look hard.

---

### Section 2 — SLOs and current attainment

Blank table. Fill from your own SLO definitions (Topic 130 refines these) and your
own measured attainment.

```markdown
| Journey | SLI (good / valid, stated precisely) | Target | Window | Measured where | Current attainment | Evidence link | Falsifier |
|---|---|---|---|---|---|---|---|
| Place an order |  |  |  |  |  |  |  |
| Read an order |  |  |  |  |  |  |  |
| Browse catalogue |  |  |  |  |  |  |  |
| Payment callback processed |  |  |  |  |  |  |  |
| Order-placed event delivered |  |  |  |  |  |  |  |
```

```markdown
### If any target is not currently met

| Journey | Target | Attainment | Gap | Cause (evidenced) | Owner | Plan and date |
|---|---|---|---|---|---|---|
|  |  |  |  |  |  |  |
```

**Strong:** the SLI is written as an explicit ratio with the valid-event set
defined; "measured where" says client-side or server-side and names the metric;
attainment is a real computed number over a real window, and where it is not
measurable yet the cell says `NOT MEASURED` rather than being left ambiguous.

**Weak:** targets with no attainment column. This is the single most common defect
in real readiness reviews. A target is an intention; attainment is a fact; a document
with only intentions has told the reader nothing about the system.

**Also weak:** an SLI defined only by HTTP status. Topic 92's double-debit and Topic
115's lost event are both 200s. A journey with a correctness failure mode needs a
correctness SLI or you must state that you are accepting blindness there.

---

### Section 3 — Capacity headroom at the Topic 65 baseline

```markdown
### 3.1 The baseline this section rests on

- Baseline run:
- Date:
- Dataset size (orders / products / lines):
- Arrival rate (open model):
- Warm-up discarded / measured window:
- Re-run within ±10%? [ ] yes  [ ] no — if no, this section is not usable
```

```markdown
### 3.2 Where the ceiling is, and what causes it

| Resource | Signal used | Value at baseline | Limit | Utilisation | Binding at what multiple of baseline? |
|---|---|---|---|---|---|
| App CPU |  |  |  |  |  |
| App heap / allocation rate |  |  |  |  |  |
| Connection pool (active / pending) |  |  |  |  |  |
| Postgres connections / CPU |  |  |  |  |  |
| Redis |  |  |  |  |  |
| Kafka consumer lag |  |  |  |  |  |
| Notification executor queue depth |  |  |  |  |  |

**The binding constraint at baseline is:**
**The next constraint after that is:**
**Evidence for both:**
```

```markdown
### 3.3 Pool sizing (Topic 109)

| Pool size | p95 | p99 | Throughput | pending peak | Timeouts |
|---|---|---|---|---|---|
|  |  |  |  |  |  |
|  |  |  |  |  |  |
|  |  |  |  |  |  |

- Chosen size and why:
- What breaks if it is raised:
- What breaks if it is lowered:
```

```markdown
### 3.4 Headroom statement

- Headroom to the binding constraint, as a multiple of baseline arrival rate:
- What we would do at 80% of that: 
- Lead time on that action (scale-out minutes / replica provisioning days):
- Is the growth rate known? [ ] yes, source:  [ ] no — this is a GAP
```

**Strong:** the binding constraint is named with the signal that proves it, and the
headroom number is a *multiple of a measured arrival rate*, not a percentage of an
abstract capacity. The lead time is stated — headroom is meaningless if the mitigation
takes three weeks to provision. Topic 109's curve is present, which means the pool
number was chosen rather than inherited.

**Weak:** a headroom multiple asserted with no ceiling cause. Headroom against what? If the
ceiling is the pool and the pool is bound by the database's own connection limit,
adding instances moves you closer to the wall, not further from it. That is exactly
the reasoning Topic 129 will demand of you, and this is the rehearsal.

**Also weak:** capacity claimed from a run where the load generator was itself
saturated, or against a small dataset. Topic 65 traps 1 and 4. If the baseline is
not trustworthy, say so here and stop; every number downstream inherits the error.

---

### Section 4 — Failure-mode register

This is the heart of the review. **One row per mode from the Machine-level reality
register**, plus any others your service has.

```markdown
| # | Failure mode (external symptom) | Drill | Currently mitigated? | Mitigation (what, where) | Detection signal | Alert? | Runbook entry | Residual risk | Owner |
|---|---|---|---|---|---|---|---|---|---|
| F1 | Heap grows to OOM under sustained load | 79 |  |  |  |  |  |  |  |
| F2 | Memory plateaus at pool-size × context-size | 79 |  |  |  |  |  |  |  |
| F3 | RSS grows while heap is flat; container OOMKilled | 80 |  |  |  |  |  |  |  |
| F4 | Single multi-second pause unrelated to heap fullness | 71 |  |  |  |  |  |  |  |
| F5 | Pause longer than the reported GC work | 73 |  |  |  |  |  |  |  |
| F6 | Throughput drop after a deploy that changed no hot code | 74 |  |  |  |  |  |  |  |
| F7 | Wrong defaults / OOMKill under container limits | 82 |  |  |  |  |  |  |  |
| F8 | Runtime reflection failure on the native build | 83 |  |  |  |  |  |  |  |
| F9 | Latency climbs, throughput flat, then OOM under overload | 90 |  |  |  |  |  |  |  |
| F10 | Unrelated parallel work stalls when one endpoint is slow | 91 |  |  |  |  |  |  |  |
| F11 | Double wallet debit for one idempotency key | 92 |  |  |  |  |  |  |  |
| F12 | Requests hang; two threads blocked forever | 94 |  |  |  |  |  |  |  |
| F13 | Thread count grows to native-thread OOM | 98 |  |  |  |  |  |  |  |
| F14 | Throughput collapses with idle CPU | 101 |  |  |  |  |  |  |  |
| F15 | Monitor-blocked time on the inventory path | 85 |  |  |  |  |  |  |  |
| F16 | Shutdown flag set, loop does not exit | 86/87 |  |  |  |  |  |  |  |
| F17 | Heap grows during downstream slowdown, no error until OOM | 105 |  |  |  |  |  |  |  |
| F18 | Logs lose correlation ID after an async boundary | 108 |  |  |  |  |  |  |  |
| F19 | All threads parked in getConnection; service up, serving nothing | 109 |  |  |  |  |  |  |  |
| F20 | Downstream blip becomes sustained outage | 111 |  |  |  |  |  |  |  |
| F21 | Consumer lag grows with repeated rebalances | 113 |  |  |  |  |  |  |  |
| F22 | Order exists with no downstream event | 115 |  |  |  |  |  |  |  |
| F23 | Monitoring degrades while the service is healthy | 118 |  |  |  |  |  |  |  |
| F24 | Trace broken at an async hop | 119 |  |  |  |  |  |  |  |
| F25 | All pods restart together when the database blips | 121 |  |  |  |  |  |  |  |
| F26 | Requests dropped during every rolling deploy | 123 |  |  |  |  |  |  |  |
```

**Rules for filling this in, and I will check all four:**

1. **"Currently mitigated?" is yes / no / partial — never blank.** Partial requires a
   sentence saying which part.
2. **Detection is a named signal**, not "monitoring". `hikaricp.connections.pending`
   is a signal. "Grafana" is not.
3. **Alert? is yes / no.** A signal on a dashboard nobody is looking at during an
   incident is not detection. Marking it "no" is honest and useful.
4. **Runbook entry is a link or a path.** "Ask the team" is not a runbook entry;
   it is a single point of failure with a human in it.

**Strong:** several rows where detection is `no` and the residual risk is stated
plainly. A register with 26 rows all green is a register that was filled in
optimistically, and I will find the soft one by asking about F2 or F24.

**Weak:** mitigation described as "fixed". Fixed how, where, and what would
reintroduce it? For F11, "fixed with `putIfAbsent`" is good; the residual-risk cell
should say "any new read-then-write path on the idempotency store reintroduces it —
review item".

---

### Section 5 — Runbook coverage

```markdown
| Failure mode | Runbook exists? | Confirm-the-diagnosis step | First mitigating action | Escalation | Last rehearsed |
|---|---|---|---|---|---|
| F1 |  |  |  |  |  |
| F9 |  |  |  |  |  |
| F12 |  |  |  |  |  |
| F14 |  |  |  |  |  |
| F19 |  |  |  |  |  |
| F20 |  |  |  |  |  |
| F21 |  |  |  |  |  |
| F22 |  |  |  |  |  |
| F25 |  |  |  |  |  |
| F26 |  |  |  |  |  |
```

A runbook entry must contain a **confirmation step** — the command or query that
distinguishes this failure from the three others with the same symptom. F12 and F19
both present as "requests hang". The thread dump distinguishes them: monitors held
in a cycle versus every thread parked in `getConnection`. Without the confirmation
step, the runbook is guessing.

**Strong:** "last rehearsed" has dates in it, and at least one entry is honestly
marked *never*.

**Weak:** every runbook is "restart the pod". For F1 that is a valid holding action
and should say so explicitly, with the note that it resets the clock without
fixing anything. For F22 it makes things worse.

---

### Section 6 — Observability coverage gaps

```markdown
### 6.1 Coverage against the register

| Failure mode | Metric | Trace | Log | Alert | Gap statement |
|---|---|---|---|---|---|
| (one row per F#, or per group) |  |  |  |  |  |
```

```markdown
### 6.2 Signal inventory

| Layer | What exists | What is missing |
|---|---|---|
| RED on every endpoint |  |  |
| USE on every pool (Hikari, executors, Kafka) |  |  |
| GC and safepoint logging retained |  |  |
| Heap-dump-on-OOM configured, with a writable path |  |  |
| Trace sampling rate and whether errors are always sampled |  |  |
| Correlation ID present on every log line, across async hops |  |  |
| Cardinality guard (bounded tag values) |  |  |
| Alert routing: who is paged, on what, at what hour |  |  |
```

**Strong:** an explicit statement of the *worst undetected failure*. Every system
has one. Naming it is the single most valuable sentence in the observability
section.

**Weak:** "we have full observability." Nobody has full observability. That sentence
tells me the author did not check coverage against the register — which takes an
hour and is the only way to know.

---

### Section 7 — Deploy, shutdown, and configuration

```markdown
| Item | State | Evidence | Gap |
|---|---|---|---|
| Readiness goes false before drain begins |  |  |  |
| Grace period exceeds p99 request duration (which p99?) |  |  |  |
| Kafka consumer drains before exit |  |  |  |
| Outbox relay finishes its batch before exit |  |  |  |
| Liveness checks only "is this JVM wedged" |  |  |  |
| Rolling deploy under Topic 65 load: dropped requests |  |  |  |
| Rollback tested from the current release |  |  |  |
| Secrets not in the image or in logs |  |  |  |
| Config change requires a restart? Which config? |  |  |  |
| Startup time, and whether it affects scale-out lead time |  |  |  |
```

**Strong:** the grace-period row names *which* p99 it is compared against, and
notices that the longest request is not the p99 request.

**Weak:** "Spring Boot handles graceful shutdown." It does, if configured, and the
ordering with readiness is the part that is usually wrong. Topic 123's drill exists
precisely because the default ordering drops requests.

---

### Section 8 — Data and correctness

```markdown
| Question | Answer | Evidence | Gap |
|---|---|---|---|
| What is the worst wrong outcome for a user, and what prevents it? |  |  |  |
| Is order placement idempotent end to end? Under retry? Under duplicate delivery? |  |  |  |
| Can a wallet be debited twice? What proves it cannot? |  |  |  |
| Is the outbox relay at-least-once, and are consumers idempotent? |  |  |  |
| What happens to a poison message? Bounded retry, DLQ, alert? |  |  |  |
| Backup exists? Restore *tested*, with a recorded restore time? |  |  |  |
| Is there a reconciliation job, and what does it compare? |  |  |  |
```

**Strong:** the restore row says "restore tested on <date>, took <recorded time>",
or honestly says never tested and lists it as a top risk. An untested backup is a
belief, not a control.

**Weak:** "we take nightly backups". That is a statement about a cron job, not about
recovery.

---

### Section 9 — Known unknowns

Explicit. This is the section that makes the review honest.

```markdown
| What we have not tested | Why not | Worst case if it goes wrong | Decision: accept / schedule | Owner | Date |
|---|---|---|---|---|---|
| Postgres failover |  |  |  |  |  |
| Availability-zone loss |  |  |  |  |  |
| Certificate / credential expiry |  |  |  |  |  |
| Schema migration under load |  |  |  |  |  |
| Traffic at N× baseline |  |  |  |  |  |
| Cold-cache mass eviction |  |  |  |  |  |
| Kafka broker loss / partition leadership change |  |  |  |  |  |
```

**Strong:** items marked "accept" with a reason and an accepting person. Accepting
risk deliberately is senior behaviour. Accepting it by omission is not.

**Weak:** an empty section. It means "we have tested everything", which is never
true, and it is the first place I will push.

---

### Section 10 — Top three risks

Three. Not fifteen.

```markdown
| # | Risk (one sentence) | Why it is top-three | Owner (person) | Mitigation | Due | What makes it worse | Review date |
|---|---|---|---|---|---|---|---|
| 1 |  |  |  |  |  |  |  |
| 2 |  |  |  |  |  |  |  |
| 3 |  |  |  |  |  |  |  |
```

**Strong:** each risk is stated as an outcome, not a missing artefact. "We cannot
recover from a Postgres failure within any known time" beats "no failover runbook".
The first states the consequence; the second states a missing document, and missing
documents do not get prioritised.

**Weak:** three risks that are all engineering-hygiene items, when the honest top
risk is organisational — one person understands the outbox relay, and they are the
author. Say that. It is the highest-value sentence in most readiness reviews and
almost nobody writes it.

---

### Section 11 — Recommendation and conditions

```markdown
- Recommendation: [ ] Go  [ ] Go with conditions  [ ] Not yet
- Conditions (each with owner and date):
- What would change this recommendation:
- Signed:              Date:
```

**Strong:** conditions that are checkable and dated, and a recommendation that is
not "go" if the honest answer is "go with conditions".

**Weak:** the review that could only ever have ended in "go", because the go-live
date was fixed before the review started. If that is the situation, the review's
job changes: it documents what is being accepted and by whom. Write that sentence
down. It is the most useful thing an honest reviewer can do inside a decision that
has already been made.

---

## Wrong approach → exact symptom → root cause → fix

Five. The symptoms are organisational, and each is concrete enough that you can
recognise it in a document you are holding.

---

### Wrong approach 1 — the review asserts "we handle load" with no baseline behind it

**Exact symptom.** The capacity section is three sentences long and contains the
words "comfortably", "sufficient headroom", or "no issues observed". Two months
later, traffic grows by a third and p99 doubles. In the incident review someone
pulls up the readiness document and reads the capacity section aloud, and there is
nothing in it to check — no arrival rate, no dataset size, no binding constraint. The
document cannot be wrong, and it also cannot help. The team's confidence in every
other section drops at that moment, including the sections that were rigorous.

**Root cause.** The author had the baseline (Topic 65 exists) and summarised it
instead of citing it, because the summary reads better and nobody asked for more.
Underneath that is a cultural cause: in most organisations the review is read by
people who cannot evaluate the numbers, so confident prose is rewarded and evidence
is not. The author optimised, rationally, for the reader they had.

**Fix.** Make every capacity claim a citation plus a falsifier. The rule is
mechanical: no adjective without a number, no number without a link, no claim
without the measurement that would disprove it. Concretely, the capacity section
becomes Section 3's tables: the baseline reference with its run ID, the binding
constraint with the signal that proves it, headroom as a multiple of a measured
arrival rate, and the lead time on the mitigating action. If a cell cannot be
filled, it becomes a `GAP` row in known unknowns with an owner. The document gets
longer and duller and starts being able to be wrong, which is the property you
wanted.

---

### Wrong approach 2 — the failure-mode register lists mitigations and never asks about detection

**Exact symptom.** Every row in the register says "mitigated". Six weeks later, F2
happens — `ThreadLocal` retention on a pooled thread after someone adds a new
context object. Memory rises to a plateau. No alert fires, because a plateau is not
a leak shape and nothing was watching heap-after-GC as a trend. It is discovered a
month later by an engineer investigating something else, who notices the service is
running with far less usable heap than it should have. Nobody can say when it
started. The register said "mitigated" for a mitigation that had been removed by a
refactor, and nothing noticed.

**Root cause.** Mitigation is satisfying to write and detection is not. A mitigation
is a thing you built; detection is an admission that the mitigation might fail or be
removed. Authors write the register from memory of the fix, not from a coverage
check against signals. Also: nobody validates the register against reality after it
is written, so a mitigation deleted by a refactor stays "mitigated" forever.

**Fix.** Add the two columns — detection signal, and alert yes/no — and refuse to
leave them blank. Then do the coverage pass in Section 6.1: for each failure mode,
open the dashboard and confirm the signal exists and moves. Where it does not,
write `no` rather than something aspirational. Finally, add the strongest available
control from Topic 132's hierarchy: for F2, the strongest control is not an alert
but a wrapper that makes the bug unwritable — a request-context holder with a
mandatory clear in a filter, or a check in CI for `ThreadLocal` fields without a
paired `remove`. Detection is the floor, not the goal.

---

### Wrong approach 3 — thirty risks, none owned

**Exact symptom.** The risk section is a two-page table. It gets a nod in the review
meeting. Ninety days later, none of the thirty items has moved. When one of them
becomes an incident, three people independently say "yes, we knew about that" — and
they are all telling the truth. The document had it. The document had twenty-nine
other things too, ranked by nothing, owned by nobody, due never.

**Root cause.** Listing risks is cheap and prioritising them is expensive, because
prioritising means saying that twenty-seven things are *not* being worked on, and
that sentence has to be defended. The author avoided the defence by listing
everything, which feels thorough and is actually an abdication. A list with no
ranking transfers the decision to whoever reads it, and nobody reads it.

**Fix.** Three risks. Exactly three, with a person's name, a date, and a
"what makes this worse" clause. Everything else goes into the register or the
known-unknowns table, which are reference material and not commitments. If someone
argues an item deserves top-three status, that argument is the productive
conversation the meeting exists for — and it forces the trade to be explicit: which
of the three does it displace? Three is not a stylistic preference; it is the number
that forces ranking to happen out loud.

---

### Wrong approach 4 — a review that reproduces the design doc

**Exact symptom.** Sections 1 to 4 explain the architecture: the outbox, the
breaker, the idempotency store, the event flow. It is well written. A reviewer who
already knows the system reads six pages and learns nothing about *readiness*. The
review passes, because there is nothing in it to object to, and the observability
gaps were never listed — so the first real incident is invisible for forty minutes
while people work out which signal to look at.

**Root cause.** The author confused "describe the system" with "assess the system",
which happens because describing is comfortable and assessing requires stating what
you do not know in front of colleagues. It is also a genre error: design docs and
readiness reviews look similar and answer opposite questions. A design doc argues
for a future; a review reports on a present.

**Fix.** One paragraph of architecture with a link to the design doc, then delete
the rest. Test each remaining paragraph against a single question: *does this tell
the reader something about whether the system is ready, or only about what it is?*
If it is the second, it is documentation and belongs elsewhere. The review's centre
of gravity should be Sections 4, 6, 9 and 10 — register, gaps, known unknowns,
risks. If those four are shorter than the architecture section, the document is the
wrong shape.

---

### Wrong approach 5 — the review is written by the person who cannot fail it

**Exact symptom.** The author is the tech lead, the reviewer is the same tech lead's
manager, the go-live date was announced to customers three weeks ago, and the review
happens on the Tuesday before. Its recommendation is "go". Everyone present knows
the recommendation was fixed before the document was written. The review takes forty
minutes and produces no conditions. Two months later, in an incident, someone asks
"did the readiness review not cover this?" and the answer is that it covered it in
the sense of containing a sentence about it.

**Root cause.** The review was scheduled as a ceremony rather than as an input to a
decision. Nobody in the room had the authority — or the incentive — to say "not
yet". Structurally, the author is the person most invested in the answer being yes,
which is the exact conflict the review exists to protect against.

**Fix.** Two structural changes, and neither requires organisational power to
propose. First, name a decision owner who is not the author, in the header block,
before writing. Second, make "go with conditions" the expected outcome rather than
a failure — conditions with owners and dates are how a review influences a fixed
date without fighting it. If the date genuinely cannot move, the review's job
changes and you should say so in the summary: *"This review does not gate the date.
It records what is being accepted, by whom, and what we will do first when it
bites."* That sentence is uncomfortable, entirely honest, and it makes the document
useful in the incident it is predicting. Topic 134 is about how to get that sentence
accepted rather than resented.

---

## Gate deliverable — what you must produce

**One document.** `docs/java/reviews/<date>-orderflow-prr.md`, committed.

### Format and length

- **6 to 12 pages.** Under 6 and the register is not complete. Over 12 and you have
  written architecture documentation instead of an assessment.
- Written so that reading only the **Summary**, the **Top three risks**, and the
  **Known unknowns** gives an accurate picture. Those three are what a busy reader
  actually reads, and they must not disagree with the body.
- Every table blank in this document must be filled from your own measurements, or
  explicitly marked `NOT MEASURED` with an owner. Both are acceptable. Silence is not.

### Required sections

1. Header block with a **named decision owner who is not you**, and a next-review
   trigger that is an event.
2. Summary and recommendation, five sentences or fewer, containing the largest risk
   and any conditions.
3. SLOs and current attainment, with an explicit not-met table.
4. Capacity headroom at the Topic 65 baseline, including the pool-size curve from
   Topic 109 and a named binding constraint.
5. **The failure-mode register — every one of F1 to F26**, plus anything else your
   service has, with mitigation, detection, alert, runbook and residual risk.
6. Runbook coverage, with a confirm-the-diagnosis step per entry.
7. Observability coverage gaps, including a named **worst undetected failure**.
8. Deploy, shutdown and configuration.
9. Data and correctness, including whether restore has been tested.
10. Known unknowns, with accept-or-schedule decisions.
11. Top three risks with named human owners and dates.
12. Recommendation and conditions.

### Required content — the specific things I will check for

- Every quantitative claim labelled `MEASURED`, `ESTIMATED`, or `ASSUMED`, and every
  `MEASURED` claim linked to a baseline file or a dashboard query.
- **A falsifier on every capacity and attainment claim.** One sentence: what
  measurement, coming out differently, would make this false.
- At least three rows in the register where detection is honestly `no`.
- At least one failure mode marked **accepted**, with the reason and the accepting
  person's name.
- The **worst undetected failure**, named in one sentence.
- At least one risk that is organisational rather than technical — knowledge
  concentration, an unowned component, an untested human procedure.
- A statement of what this review does **not** cover.
- A next-review trigger that is an event, not only a date.

### The gate check

You pass this gate when all four are true:

1. Every row of the F1–F26 register is filled with no blank cells.
2. Every capacity claim traces to a baseline run that you have re-run within ±10%.
3. The document contains at least one sentence that is uncomfortable to write.
4. Someone other than you has read it and named its weakest claim — and you agree
   with their choice, or can say precisely why they are wrong.

Point 3 is not a joke. A readiness review with nothing uncomfortable in it is a
review that stopped at the comfortable edge of what the author knows.

---

## How I will review it

I will not comment on your prose. I will attack the weakest claim, the missing
falsifier, and the failure mode you knew about and left out.

### The three questions that usually break a readiness review

**Question 1: "Show me the measurement behind this sentence."**

I will pick the most confident sentence in the document — usually in the capacity or
reliability section — and ask for the number, the run, and the date.

This breaks most readiness reviews because the confident sentences are exactly the
ones written from memory. The author *did* measure something, six weeks ago, on a
different commit, at a different arrival rate, and has been carrying a rounded
version of it in their head since.

What a good answer sounds like: *"Baseline run 03, that file, that arrival rate,
re-run last Thursday well inside the 10% gate on every percentile. The claim is that the pool binds
before CPU; the evidence is `pending` going non-zero while CPU is still under its
limit. If a 1.5× run showed CPU saturating first, the claim is wrong and the whole
capacity section changes."*

Follow-ups I will pursue:

- "That run is from before the virtual-thread switch. Is it still the binding
  constraint?" Changing the threading model can move the ceiling without changing a
  single line of business logic, and reviews frequently carry a pre-Loom number.
- "Was the load generator saturated?" If you cannot answer, the number is unusable
  and so is everything derived from it.
- "You said `MEASURED`. Measured client-side or server-side, and what is the gap
  between them?" A review that has not looked at that gap has not really read its own
  baseline.

**Question 2: "Which failure mode has no detection, and how long would it take you
to notice it in production?"**

I am not asking whether coverage is complete. I know it is not. I am asking whether
you know *where* it is not, and whether you have thought in units of time.

The failure mode: an author who says "we have good coverage". That answer tells me
they never did the coverage pass, because everyone who does it comes back with a
list.

What a good answer sounds like: *"F2 and F24. F2 — `ThreadLocal` retention on a
pooled thread — plateaus rather than climbs, so no leak alert fires; we would notice
it at the next heap-related incident, which could be months. F24 — a broken trace at
an async hop — is invisible until someone tries to debug a latency spike and finds
the trace ends. Both are in known unknowns; F2 has an owner because it costs us
usable heap silently, and F24 does not, because its cost is only paid during an
investigation."*

Follow-ups:

- "You said months. What is the cost of months?" Forces you to price the gap rather
  than just name it.
- "You alert on `pending > 0` for a minute. What does the operator do when it fires
  at 03:00, and does the runbook distinguish F19 from a slow query?" Alerts without a
  distinguishing runbook step produce a restart, and a restart on F19 clears the
  symptom and destroys the evidence.
- "Which alert in this document would you delete?" A review that has never removed
  an alert has an alerting system that is training its operators to ignore it.

**Question 3: "What is in this document that you were tempted to leave out?"**

Every honest review has one. Usually it is either an untested recovery path, a
component only one person understands, or a number that came out worse than
expected.

This question breaks reviews because the reflex is to answer "nothing". If nothing
was tempting to leave out, the author did not get near the edge of what they know.

What a good answer sounds like: *"Two. We have never tested Postgres failover — I
nearly wrote 'managed service, covered by the provider', which is true about the
mechanism and says nothing about our recovery time. And I am the only person who
understands the outbox relay's `SKIP LOCKED` behaviour under contention; that is the
top organisational risk and I have put my own name on it with a date to write it up
and walk two people through it."*

Follow-ups:

- "Who else could run the failover if it happened tonight?" The answer is usually
  nobody, and that is the actual risk rather than the untested procedure.
- "You put your own name on the knowledge-concentration risk. What is the deliverable
  that closes it, and how do you know it worked?" A document does not close it; two
  other people successfully operating the thing does.
- "What did your reviewer push back on?" If the answer is nothing, the review was not
  reviewed.

### The other attacks, in the order I will make them

**On the register:**

- "F14 says mitigated. What reintroduces it?" If the mitigation was replacing
  `synchronized` with `ReentrantLock` on one path, any new `synchronized` block
  around a blocking call reintroduces it. Is that a review item, a check, or nothing?
- "F22 says the outbox fixes it. What happens if the relay is down for an hour — is
  that detected, and what is the maximum acceptable lag?" A fix that moves the
  failure to a new component needs the new component's failure mode in the register
  too. Registers frequently stop one step short.
- "F23 is a failure of the monitoring system, not the service. Who is paged, and is
  it the same person?" Cardinality incidents are commonly other people's outage
  caused by your service, which changes both the runbook and the ownership.
- "You have marked F16 as not applicable. Why?" If the answer is "we do not use stop
  flags", check the shutdown path first.

**On capacity:**

- "Your headroom number assumes linear scaling. Where does it stop being linear?"
- "If you added two instances tomorrow, what would get worse?" Database connections,
  cache hit ratio during warm-up, and Kafka partition count are the usual answers,
  and a review that has not considered them is describing the app in isolation.
- "The pool-size curve — did you measure it, or reason about it from a formula?"

**On risks:**

- "Risk 1 has an owner. Does the owner know?" I will ask them.
- "What is risk 4, and why did it lose?" Tests whether ranking happened.
- "Which of these three risks would you drop if you were given one week of
  engineering time instead of three?"

**On the recommendation:**

- "Could this review have concluded 'not yet'? What would that have taken?"
- "The conditions have dates. What happens if the date passes and the condition is
  not met?" If the answer is nothing, they are wishes.

### What I will not attack

- Formatting, section ordering, or length within the range.
- The absence of failure modes you have not encountered, *provided* they are in
  known unknowns.
- A `NOT MEASURED` cell with an owner and a date. That is a good document being
  honest, and I would rather have twelve of those than one confident paragraph.
- Conclusions I disagree with, if the evidence and the falsifier are stated. A
  well-evidenced conclusion I think is wrong is a productive conversation. A
  confident conclusion with no evidence is not a conversation at all.

---

## Measurement

How you know the review is working — as a document and as a practice.

### Measuring the document itself

| Question | How to check | What a bad answer means |
|---|---|---|
| Can every claim be traced? | Pick five claims at random; try to reach the evidence in under two minutes each | Untraceable claims mean the document is memory, not measurement |
| Does it have falsifiers? | Count claims with a stated falsifier as a fraction of quantitative claims | Below half means most of the document cannot rot visibly |
| Is the register complete? | Count filled cells against F1–F26 × columns | Blanks are where the author stopped looking |
| Is detection honest? | Count rows where alert = no | Zero means the coverage pass was not done |
| Is it ranked? | Count top risks | More than three means ranking was avoided |
| Is it owned? | Count risks with a human name (not a team) | Team-owned risks are unowned risks |

### Measuring the practice over time

The interesting measurement is not the review. It is what happens after.

| Signal | Where it comes from | What it tells you |
|---|---|---|
| Incidents whose failure mode was already in the register | Postmortems (Topic 133) | High is good for the register, and points at detection or prioritisation failing |
| Incidents whose failure mode was **not** in the register | Postmortems | Each one is a new register row, and a question about why the drill list missed it |
| Time from symptom to correct diagnosis | Incident timelines | Runbook quality, measured directly |
| Top-three risks closed by their due date | The review itself, re-read at the next review | Below half means the review is decoration |
| Conditions attached to "go with conditions" that were actually met | Same | The credibility of the whole practice rests on this |

### The one measurement that matters most

**At the next incident, was the failure mode already in the register, and did the
runbook shorten the diagnosis?**

If yes, the review paid for itself. If the mode was in the register but the runbook
did not help, your registers are good and your runbooks are prose. If the mode was
not in the register at all, you have a new row and a question worth asking: was this
knowable before, and what kind of drill would have found it?

That question is the bridge into Topic 133, which is the same discipline applied
after the fact instead of before it.

---

## Interview questions (Senior → Principal)

### Q1 — "Is this service production ready?"

**A Senior answer.** Walks the architecture: circuit breakers, outbox, idempotency,
tracing, probes. Concludes that the service is well built and ready. Everything said
is true, and it is a list of capabilities.

**A Principal answer.** *"Ready for what load, and what are we accepting? Here is
what I know: at the baseline arrival rate the binding constraint is the connection
pool, not CPU, and headroom to it is X. Here is what I do not know: we have never
tested Postgres failover, so our recovery time is unmeasured. Here are the three
things I would want fixed first, and here is who owns each. My recommendation is go
with two conditions."*

**What separates them.** The Senior answer describes capability; the Principal
answer states a *position* with its evidence and its edges. "Production ready" is
not a property of a system — it is a judgment about a system relative to a load and
a risk appetite. The Principal answer makes the reader able to disagree with a
specific thing.

**Adversarial follow-up.** *"Your two conditions will delay the launch by three
weeks. Product says no. Now what?"* The good answer does not repeat the conditions
louder and does not fold. It re-frames: which of the two is actually a launch
blocker versus a fast-follow, what is the cost if the non-blocker bites, and what
compensating control (an alert, a manual check, a lower initial traffic percentage)
buys back most of the risk for a day of work. That is Topic 134's mechanic —
adoption becomes cheap — applied inside a readiness argument.

---

### Q2 — "Your review says you handle current load with headroom. How do you know?"

**A Senior answer.** "We load tested it. It handled our peak with room to spare."

**A Principal answer.** *"At the recorded baseline — that arrival rate, that dataset
size, open-model generator, generator confirmed unsaturated — the pool binds before
CPU. So my headroom claim is about pool capacity and the database behind it, not
about instance count. Adding instances multiplies connections against a fixed
Postgres limit, so scale-out past a point makes it worse. I have the latency-versus-
pool-size curve. And the claim is falsifiable: a 1.5× run where CPU saturates first
would disprove it."*

**What separates them.** The Senior answer treats headroom as a scalar. The
Principal answer names the *constraint*, knows what happens when the obvious
mitigation is applied, and states the measurement that would prove it wrong. This is
the reasoning Topic 129 formalises.

**Adversarial follow-up.** *"Your baseline is from before you moved to virtual
threads. Is any of it still valid?"* The right answer is to concede immediately and
precisely: latency percentiles may have changed and must be re-measured; the
*shape* of the constraint — pool-bound rather than CPU-bound — is likely to have
become more pronounced, because raising concurrency without raising downstream
capacity moves the queue rather than removing it. Then commit to the re-run. A
candidate who defends a stale number here is showing exactly the failure Topic 135
is about.

---

### Q3 — "What is the worst thing that can happen to this service that you would not detect?"

**A Senior answer.** "We monitor errors, latency and saturation, so we would see
most things." Reasonable, and it is an answer about the monitoring, not about the
gap.

**A Principal answer.** *"Silent wrongness. Both my correctness failure modes return
200. A double wallet debit and a lost order-placed event are invisible to a
status-code SLI, and I do not currently have a reconciliation job. That is my worst
undetected failure: a user charged twice and nothing anywhere goes red. The
mitigation is a reconciliation comparing orders to published events and wallet
ledger entries to order totals, and it is risk two in my review with my name on it."*

**What separates them.** The Senior answer inventories signals. The Principal answer
reasons from *consequence to detection* and finds the class that no standard signal
covers. Silent wrongness is the failure category that separates readiness reviews
that have thought from ones that have listed.

**Adversarial follow-up.** *"How much would that reconciliation cost, and is it worth
it?"* Not a trap — a real question, and "obviously yes" is a weak answer. The strong
answer prices it: a scheduled comparison over a bounded window is small; the
expensive part is deciding what happens when it finds a mismatch, because that is a
human process and a customer-facing decision. Then it names the trigger for building
it — the first customer-reported double charge, or the payments extraction, whichever
comes first — because pre-committing the trigger is what stops it being deferred
forever.

---

### Q4 — "How do you run a readiness review when the launch date is fixed and cannot move?"

**A Senior answer.** Documents the risks and escalates, hoping the date moves. When
it does not, the document sits unread.

**A Principal answer.** *"Then the review is not a gate and I stop pretending it is —
pretending is what destroys its credibility. Its job becomes three things. One:
record precisely what is being accepted and who accepted it, by name, in the summary.
Two: convert as many risks as possible into cheap compensating controls that fit
inside the date — an alert, a dashboard, a manual daily check, launching at 10% of
traffic. Three: pre-write the runbook for the risk I think is most likely to bite, so
that when it does, we are twenty minutes faster. I also fix the review's timing for
next time: a review three weeks before a launch is a ceremony, and moving it earlier
is a smaller ask than moving a date."*

**What separates them.** The Senior answer treats the review as a gate that failed.
The Principal answer adapts the artefact to the decision that actually exists, gets
real value out of a constrained situation, and fixes the process problem separately
rather than fighting the date.

**Adversarial follow-up.** *"Isn't that just rubber-stamping?"* The distinction is
whether the acceptance is *named and recorded*. A rubber stamp has no author; a
recorded acceptance has a person's name against a specific consequence. The second
changes behaviour next quarter, because the person who signed remembers, and because
the postmortem will reference it.

---

### Q5 — "You inherit a service with no readiness review and no baseline. Where do you start?"

**A Senior answer.** Proposes writing the review, starting with the architecture.

**A Principal answer.** *"Not with the review. With one measurement and one list.
The measurement is a baseline — without it every capacity sentence I write is a
guess. The list is the last six months of incidents, because it tells me the real
failure modes rather than the ones I would guess from the code. Those two give me a
register grounded in what has actually happened here. Then I write the review, and
the first version is mostly `NOT MEASURED` with owners, which is fine — the value is
in making the gaps visible and owned rather than in looking complete. And I would
show a draft to the person who has been on call longest before showing it to anyone
senior; they know where the bodies are and their buy-in is what makes the register
real rather than mine."*

**What separates them.** Sequencing and evidence sourcing. The Senior answer starts
with the document. The Principal answer starts with the two inputs that make the
document worth writing, and thinks about *who* makes it credible — which is Topic
134 arriving in the middle of a technical answer.

**Adversarial follow-up.** *"You have two weeks and the team is hostile to process.
What do you actually ship?"* The strong answer shrinks hard: the incident list and a
one-page register with detection gaps, no SLO section, no capacity section, no
recommendation. Then it uses the register to fix one thing that visibly helps the
on-call engineer — usually a missing alert or a runbook confirmation step — so that
the second version of the document is wanted rather than tolerated. Making adoption
cheaper than non-adoption, again.

---

## Mental model checkpoint

Answer without looking. If an answer takes more than two sentences, you are
reconstructing rather than knowing.

1. A readiness review contains the sentence "we handle current load comfortably".
   What are the three things wrong with it, and what does the replacement look like?

2. Why is a failure-mode register organised by external symptom rather than by
   cause? Give two failure modes from your own register that share a symptom and
   name the step that distinguishes them.

3. F2 — `ThreadLocal` retention on a pooled thread — is described as "mitigated".
   What question makes that answer collapse, and why is this failure mode the one
   that most often gets marked green wrongly?

4. Your review lists fifteen risks. Explain, in terms of what happens ninety days
   later, why that is worse than listing three.

5. Two failure modes present as "requests hang and never return". Name them, name
   the diagnostic that distinguishes them, and name the operator action that clears
   the symptom while destroying the evidence.

6. What is a falsifier, why does every capacity claim need one, and what happens to
   a review without them after six months of system change?

7. Your top risk is that only one person understands the outbox relay. Why is this
   harder to write than any technical risk, and what deliverable actually closes it?

---

## Quick reference card

### The gate, in one line

> Every claim names its evidence and its falsifier; every failure mode you have
> produced appears with a detection signal and a runbook entry.

### The five obligatory parts

1. SLOs **and attainment** — targets alone say nothing.
2. Capacity headroom **with the constraint named** — headroom against what.
3. The failure-mode register — mitigation *and* detection *and* runbook.
4. Observability gaps — including the worst undetected failure.
5. Top three risks — human owners, dates, "what makes it worse".

### Claim labels — apply mechanically

- `MEASURED` — number, run ID, date, link.
- `ESTIMATED` — derived from a measurement; state the derivation.
- `ASSUMED` — no evidence; belongs in known unknowns with an owner.
- `GAP` — we know we cannot see this.

### The register columns — none may be blank

symptom · drill · mitigated? · mitigation · detection signal · alert? · runbook ·
residual risk · owner

### Symptom collisions to know cold

| Symptom | Candidates | Distinguishing step |
|---|---|---|
| Requests hang forever | F12 lock deadlock, F19 pool deadlock | Thread dump: monitor cycle vs all parked in `getConnection` |
| Memory grows to OOM | F1 unbounded cache, F13 thread leak, F17 unbounded buffer, F3 off-heap | Heap dump dominator vs thread count vs RSS-minus-heap |
| Latency up, CPU idle | F14 pinning, F19 pool exhaustion, F10 blocked common pool | Pinned-thread trace / pool `pending` / pool thread states |
| Everything restarts at once | F25 liveness on a dependency | Probe config and restart timestamps correlated with the dependency |

### The three questions that break these documents

1. Show me the measurement behind that sentence.
2. Which failure mode has no detection, and how long until you notice?
3. What were you tempted to leave out?

### Anti-patterns, one line each

- Confidence without citation.
- Mitigation without detection.
- Thirty risks, none owned.
- Architecture description instead of assessment.
- A review whose only possible outcome was "go".
- Targets with no attainment column.
- An SLI defined only by HTTP status where correctness can fail silently.
- Team-owned risks — which are unowned risks.

---

## When would I use this at work?

**1. In your first six weeks on a service you have inherited.** You cannot write an
honest review yet, and that is the point — the gaps you cannot fill are the fastest
map of the system you will ever build. Do the incident-history pass, fill what you
can, mark the rest `NOT MEASURED`, and show it to the longest-serving on-call
engineer before anyone senior. You will learn more in that conversation than in a
month of reading code, and you will have produced something useful without having
had to be right about anything yet.

**2. Before any launch, migration cut-over, or significant traffic increase.** Not
because a ceremony is required, but because the register-versus-detection pass is
the cheapest hour in engineering: it reliably finds two or three failure modes that
are mitigated but invisible. The strangler extraction in Topic 128 needs one of
these at every phase boundary, because each phase changes which failure modes are
live.

**3. As the input to on-call training, which is where it pays for itself twice.**
The symptom-to-runbook index is exactly what a new on-call engineer needs and almost
never has. Hand them the register, walk the four symptom collisions, and let them
run the confirmation steps against a healthy system so the commands are familiar
before 03:00. The value of the review then stops depending on anyone re-reading the
document, which is the only way a document survives.

---

## Connected topics

**Prerequisites — the evidence this review is built from:**

- **65 — GATE: the baseline.** Every capacity and latency claim traces here. Without
  a re-runnable baseline, Section 3 is prose. Traps 1 and 4 from that topic
  invalidate a capacity section outright.
- **71, 73, 79, 80, 82, 83 — Phase 8.** Failure modes F1–F8. Humongous allocations,
  time-to-safepoint, retention shapes, off-heap growth, container awareness.
- **85, 86/87, 90, 91, 92, 94, 98, 101 — Phase 9.** Failure modes F9–F16. The
  concurrency vocabulary that a readiness review of a JVM service must contain.
- **105, 108 — Phase 10.** F17–F18. Unbounded buffering and lost context.
- **109 — HikariCP sizing and the pool deadlock.** The single most load-bearing
  number in Section 3, and failure mode F19. The latency-versus-pool-size curve is
  what turns "we have headroom" into a capacity claim.
- **111 — Resilience4j.** F20. Retry storms, and the reason a mitigation must be
  measured under an injected outage rather than assumed.
- **113, 115, 116 — Kafka and the outbox.** F21–F22. The dual-write gap is the
  clearest example of a correctness failure that returns 200.
- **118 — Micrometer, RED/USE, cardinality.** The evidence base for attainment and
  saturation, and F23 — the failure where your service takes down someone else's.
- **119 — Tracing.** How you attribute latency to a dependency, and F24.
- **121 — Probes.** F25, and the honesty of readiness itself.
- **122, 123 — Startup and graceful shutdown.** F26 and Section 7. Deploy behaviour
  is a reliability property.

**This unlocks — every Phase 12 artefact is this discipline applied to a new question:**

- **129 — Capacity, cost and latency budgets.** Section 3 is the seed of the capacity
  model; Topic 129 adds cost per request and the break-even calculation.
- **130 — SLO design and error budgets.** Section 2 is aspirational until Topic 130
  makes the targets defensible and attaches a budget policy with consequences.
- **131 — Design-doc authorship.** The evidence-labelling and falsifier discipline is
  identical; the difference is that a design doc argues for a future and this
  reports on a present.
- **132 — Engineering standards.** Where register mitigations become gates,
  wrappers, or checklist items — and where you learn which of the three is
  available for a given bug class.
- **133 — Incident postmortems.** The register is the before; the postmortem is the
  after. An incident whose failure mode was already in the register asks a different
  and better question than one that was not.
- **134 — Influence without authority.** Getting three teams to write these is an
  adoption problem, not an argument. Give them the template and the register rows
  they can reuse, and adoption becomes cheaper than explaining why they have not.
- **135 — The capstone.** This document is one of the artefacts you will defend
  under sustained hostile questioning. The first question will be "show me the
  measurement", and it will be asked in that tone.

---

*This topic contains no numbers about your system, no benchmarks, and no industry
statistics. Every table is blank by design; fill them from your own Topic 65
baseline, your own Topic 118 dashboards, and your own Phase 8–11 drill notes. The
production-readiness review is a widely used industry practice — Google's SRE
literature popularised the term and the related error-budget model — but the
template here is built from this curriculum's drills, not reproduced from anyone's
internal checklist. A review is worth exactly as much as the evidence behind it and
not one line more.*
