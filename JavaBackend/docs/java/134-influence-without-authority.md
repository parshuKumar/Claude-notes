# 134 — Technical Mentorship and Influence Without Authority

## Phase: 12 — Principal Track
## Category: ELITE
## Java baseline: 21  |  Notes features from: 21/25
## Project spine: a written plan to get one technical decision adopted by three teams that do not report to you — including how you handle the team that says no.

---

## Before anything else — what is and is not in this document

**What is in it.** A method for getting a technical decision adopted across teams
you have no authority over, built on one idea: you engineer the adoption cost down
until the argument becomes unnecessary. A worked plan on `orderflow`. Five ways this
fails organisationally. The attacks I will make on your plan.

**What is not in it, deliberately.**

- **No adoption statistics.** I will not tell you that "X% of migrations stall" or
  that "teams adopt Y times faster when Z". Those numbers are either invented or
  laundered from vendor material. Where I say something is common, it is a claim
  about mechanism, and you should check it against your own organisation.
- **No named methodology, no quoted authority.** There is a large popular literature
  on influence and persuasion. I am not summarising it, quoting it, or attributing
  claims to it. What is here comes from the mechanics of engineering adoption:
  cost, risk, and who pays.
- **No psychological technique.** This is not a document about how to be persuasive.
  It is a document about how to build the thing that makes persuasion redundant. If
  the plan's success depends on how well you present it, the plan is weak.
- **No claim that this always works.** Sometimes the right answer is that the
  decision should not be adopted by that third team, and a good plan says how you
  would find that out. There is a whole section on the team that says no, and its
  first move is to take the objection seriously.

**What I am asking you to trust.** That in engineering organisations, adoption is
almost always a cost problem wearing the costume of a disagreement — and that
treating it as a cost problem is both more effective and more honest.

---

## Mechanical statement

**Influence without authority is engineering, not persuasion. You make adoption
cheaper than non-adoption — a migration tool, a reference implementation, a paved
road — and the argument becomes unnecessary.**

Three consequences:

1. **Adoption is a cost comparison made by each team independently.** Team B adopts
   when the cost of adopting is less than the cost of not adopting, where both costs
   are measured in *their* currency: their sprint, their on-call, their risk, their
   deadline. Your conviction is not in either column.
2. **The lever you control is the cost of adopting.** You cannot easily raise the
   cost of the status quo — that requires authority you do not have, and doing it
   anyway is how you become the person other teams route around. You *can* lower the
   cost of adopting, arbitrarily far, by doing the work yourself.
3. **The remaining disagreement, after the cost is near zero, is real information.**
   If adopting takes an hour and a team still says no, they have a constraint you do
   not know about. That is the most valuable output of the whole exercise, and the
   most commonly wasted.

The corollary, which is the part people resist: **a decision announced in an
architecture forum and not adopted was not rejected. It was priced, and it lost.**

---

## The bridge from what you know

### What transfers — and I am not going to re-teach it

You have led work across teams in TypeScript-land. You already know:

- How to run a design review and take feedback without defending everything.
- How to negotiate scope with a product manager and trade one thing for another.
- That the person who is loudest in the meeting is not the person who decides.
- How to mentor an engineer: give them the problem, not the answer; review the
  approach before the implementation.
- That a shared library nobody wants is worse than three copies of code.
- That reputation compounds and is spent, not accumulated indefinitely.

All of that carries over unchanged. If you have driven a shared-standards effort
across Node services, the human half of this topic is already yours.

### What is genuinely harder at Principal level

**One: making adoption cheaper than non-adoption is a build task, not a
communication task.** This is the reframe, and it is the reason the topic exists. The
Senior instinct on hitting resistance is to explain better: a clearer document, a
better forum presentation, more data. The Principal instinct is to ask what
specifically makes adopting expensive for that team, and then to remove it — usually
by writing code, sometimes by writing a migration script, occasionally by doing the
migration for them. The cost of adopting is a number you can drive down with
engineering effort, and engineering effort is what you have.

**Two: modelling other teams' incentives accurately, including the ones you find
unreasonable.** A team that will not adopt your change because they have a launch in
three weeks is not being obstructive; they are correctly prioritising. A team that
will not adopt because their tech lead was burned by a previous platform migration
that was abandoned halfway is also not being obstructive; they are pricing in a real
risk based on real evidence, and your plan has to address it. Reading these
correctly, rather than as resistance, is most of the job.

**Three: knowing when the answer is that you are wrong.** The team that says no
sometimes has a constraint that invalidates your design. Discovering that is a
success, not a setback — and if your plan has no mechanism for discovering it, you
have written a rollout, not a plan.

### The Java-shaped part

There is a real one, and it is about what "cheap to adopt" can mean in this
ecosystem specifically:

- **A migration tool exists as a real category.** OpenRewrite recipes perform
  mechanical, reviewable, repeatable source transformations. "Run this recipe, review
  the diff" is a completely different ask from "here is a document describing what to
  change". If your decision is expressible as a recipe, writing one collapses the
  adoption cost more than any amount of advocacy.
- **A parent POM or platform BOM is a paved road.** Dependency versions, plugin
  configuration, analysis setup and defaults can be inherited rather than copied.
  Adopting becomes a one-line parent change, and staying off it becomes the thing
  that costs effort.
- **A starter or auto-configuration is a paved road with defaults.** Spring Boot's
  whole model is that the safe configuration is the one you get by doing nothing. A
  `orderflow-observability-starter` that wires the right metrics, the right
  cardinality guards, and the right trace propagation is adopted by adding a
  dependency.
- **A static-analysis check with a ratcheting baseline** (Topic 132) makes the
  decision self-enforcing on new code without a freeze — which is what makes a
  standard survive after your attention moves elsewhere.
- **A reference implementation in a real service** is worth more here than in most
  ecosystems, because Java teams are conservative about frameworks and want to see
  the thing running under production load before adopting it. Your Topic 65 baseline
  is unusually persuasive evidence for exactly this reason.

That list is the actual content of "make adoption cheap" in a Java organisation. In
Node the equivalents exist but are shallower; a shared config package does less work
than a parent POM plus a starter plus a recipe.

---

## What is this?

**Influence without authority** is the ability to change what other teams do when you
cannot tell them to. It is the defining capability of the Principal role, because the
role's scope exceeds its span of control by design — you are responsible for outcomes
across an organisation whose engineers report to other managers.

Terms, defined once.

**Adoption cost.** Everything a team must spend to take your change: engineering
hours, review time, testing, the risk of breaking something, the cognitive cost of
learning a new thing, the on-call cost of a new failure mode, and the opportunity
cost of the work they are not doing instead. Most of these are invisible from
outside the team, which is why estimates made by the proposer are usually wrong and
usually low.

**Non-adoption cost.** What the team spends by *not* taking it. Sometimes real and
immediate (they keep hitting the bug), sometimes real and deferred (they will pay at
the next upgrade), often zero for them and non-zero for you or for the organisation.
That last case — where the benefit accrues elsewhere — is the hardest and most
common shape, and it is where most cross-team proposals die.

**Paved road.** The path that is easiest to walk. Not mandatory; just cheapest. A
paved road is built by making the safe default free and leaving the alternatives
possible but manual. Its defining property is that a team ends up on it *without
having decided to*.

**Reference implementation.** The thing running in a real service, with real load,
that another team can read, copy, or point their own lead at. Not a prototype and
not a snippet. The difference is that a reference implementation has already met the
failure modes.

**Credibility budget.** A finite resource. Every proposal you push spends some; every
proposal that lands well and does not blow up later refills it. Pushing a mandate
through spends a lot. Being right about something inconvenient refills a lot. You
will need it later for something more important, and running the balance to zero on
a medium-value standard is a common Principal mistake.

**Mandate.** A rule imposed by authority. Available to you only by borrowing someone
else's authority, which spends both your credibility and theirs. Sometimes correct —
security, legal, an upgrade with a hard deadline — and almost always the wrong first
move, because a mandate ends the conversation that would have told you what you got
wrong.

### Mentorship is the same mechanic at the scale of one person

The topic title pairs mentorship with influence, and they are not two subjects. They
are the same mechanic applied at different scale.

Mentoring an engineer who does not report to you is an adoption problem where the
adopting unit is one person and the currency is their time, their confidence, and
their standing with their own manager. The lever is identical: make the good path
cheaper than the bad one *for them*.

Concretely, three moves that are engineering rather than encouragement:

- **Review the design, not the code.** A review at the end of the work can only
  produce rework, and rework is the expensive thing. A twenty-minute conversation
  before the code exists costs them nothing and changes the outcome. This is the
  single highest-leverage habit available, and it is the one most senior engineers
  skip because reviewing code is scheduled and reviewing intentions is not.
- **Give the problem, then the watch-list.** The answer teaches one thing. Telling
  them what you would have watched for — "check whether that call happens inside a
  monitor", "check what the pool does when the downstream is slow" — teaches a class,
  which is the same distinction Topic 133 makes about action items.
- **Make the work visible to the person who writes their review.** Mentoring someone
  whose manager cannot see the improvement is a kindness to them and does nothing for
  their career. A sentence from you to their manager, unprompted, costs you two
  minutes and is often worth more than the technical guidance.

The characteristic failure is becoming the person they ask rather than the person who
taught them how to find out. The self-check is mechanical: if they are still bringing
you the same *shape* of question after three months, you have been answering rather
than teaching.

### The three shapes of cross-team decision

They have different mechanics and confusing them is a common failure:

1. **Convergent** — everyone benefits from doing the same thing, and the benefit is
   mostly local. A shared logging format, a standard health-check shape. Easy: the
   cost is the only obstacle.
2. **Externality** — the benefit accrues to someone other than the adopter. Metric
   cardinality discipline protects the monitoring platform, not the emitting team;
   idempotent consumers protect the producer's ability to retry. Hard: you must
   either move the cost or make the externality visible to the adopter.
3. **Coordination** — the value exists only if everyone does it, and partial adoption
   is worse than none. A tracing propagation format, an event schema, a shared
   idempotency key convention. Hardest: you need a critical mass, and a first mover
   pays the full cost for no benefit. Sequencing matters more than persuasion.

Naming which shape you are in is the first analytical step, because it determines
the strategy. An externality problem solved with a convergent-problem strategy
("here's why it's good for you") produces a polite yes and no action.

---

## Why does it matter?

### 1. Because a decision that is not adopted has no value, however correct it is

An architecture decision exists in one of two states: adopted by the systems it
concerns, or not. Correctness is a precondition and not a substitute. A Principal
engineer who is right about everything and lands nothing is producing zero, and
usually cannot see it, because being right is very satisfying.

### 2. Because your scope exceeds your authority permanently

This is not a phase you grow out of. As you become more senior the ratio gets worse:
your responsibility grows across organisations, and your direct authority stays at
zero. Every principal-level outcome you produce for the rest of your career will be
produced through people who do not report to you.

### 3. Because mandates are a loan against a budget you will need later

You can usually get one, by escalating to a director who agrees with you. It works
once. What it costs is that the teams affected learn that your route to agreement is
escalation, and they route around you next time — they stop bringing you problems
early, which is where you did most of your value. A mandate is a real tool and it
should be the last one you reach for.

### 4. Because the team that says no is your best source of design feedback

Their objection is either a constraint you did not model — in which case your design
is wrong and you have just found out cheaply — or a cost you underestimated, in
which case your adoption plan is wrong and you have just found out cheaply. Both
outcomes are worth more than a third yes. Teams that agree with you teach you
nothing.

### 5. Because this is the visible difference between Senior and Principal in an interview

Both candidates will describe a good technical decision. The Senior candidate
describes convincing people. The Principal candidate describes building the thing
that made the decision cheap, and can tell you what the objecting team taught them.
Interviewers listen for exactly this, which is why the standard follow-up to any
cross-team story is "what did you do when they said no".

---

## The decision, framed

The decision this topic trains is: **given a technical position you believe is
right, and three teams that do not report to you, what do you actually build?**

Not what do you argue. What do you build.

### The framing question

> *For each team, what is the single most expensive part of adopting this, and can
> I remove it with engineering work?*

Run it per team, because the answer differs. For one team the expensive part is the
code change. For another it is regression risk on a payment path. For a third it is
that they have a launch in three weeks and no capacity at all — for which the answer
is not to reduce the cost but to change the timing.

### The cost ledger — the analytical core

Write it down, per team, in their currency:

| Cost component | What it means | How you can reduce it |
|---|---|---|
| Code change | Hours of editing | A migration tool; a PR you open yourself |
| Review | Their senior's time | A small, mechanical, reviewable diff |
| Test | Confidence it still works | Tests you write; a canary; a staged rollout |
| Risk | What if it breaks in production | A reference implementation already under load; a rollback that is one line |
| Learning | New concept to hold | A default that requires no understanding to be safe |
| On-call | New failure mode to own | The alert and runbook entry shipped with it |
| Opportunity | The work they are not doing | Timing: land it in their planning cycle, not across it |

The Senior move is to attack the first row, because it is the visible one. The
Principal move is to find out which row is actually binding for that team, which
usually requires asking them, and which is often risk or opportunity rather than
effort.

### Sequencing — who goes first, and why it decides the outcome

Order matters more than argument. The general shape:

1. **Build the reference implementation in a service you control.** For you, that is
   `orderflow`. It must be running under real load, with the failure modes already
   met, before you ask anyone for anything.
2. **Pick the first external team by lowest cost, not by highest need.** The team
   with the simplest service and the most available capacity. Their adoption is your
   proof that the migration path works on code you did not write, which is a
   different claim from "it works in my service".
3. **Do their migration yourself.** Open the pull request. Their cost drops to
   reviewing a diff. You learn what your instructions got wrong, on someone else's
   codebase, before it costs you credibility with a sceptical team.
4. **Use team one as the evidence for team two.** "Team A's service has been running
   this for six weeks; here is their engineer's view; here is the diff" is a
   qualitatively different ask, and it costs you nothing to assemble.
5. **Take the hardest team last**, when the risk row in their ledger has been
   emptied by other people's production experience.

The common mistake is starting with the most important team, because that is where
the value is. They are also the most sceptical and have the most to lose, and losing
there first makes the other two harder.

### What you are willing to lose

Decide this before you start, and write it in the plan:

- **What is the minimum viable adoption?** If two of three adopt, is the value
  realised? For a convergent decision, usually yes. For a coordination decision,
  possibly no — and if it is no, the plan needs a critical-mass strategy rather than
  a sequence.
- **What would make you drop it?** Name the discovery that means you were wrong.
- **What is this worth on the credibility budget?** If the answer is "not much", do
  not spend a mandate on it. Some standards are worth having and not worth fighting
  for, and knowing which is a senior skill.

---

## Example 1 — a minimal illustration

The smallest version, so the mechanic is visible.

### The situation

Four services log in different formats. You want a single structured JSON format
with a correlation-ID field, so that logs are queryable across services during an
incident.

### Version A — the announcement

You write a one-page standard, present it at the architecture forum, get nods, and
send it round. Two months later one service has adopted it — yours.

Nothing went wrong socially. Everyone agreed it was a good idea. It simply cost each
team a day of work for a benefit that arrives during someone else's incident, and a
day of work always loses to whatever is in the sprint.

Notice the shape: this is an **externality** problem. The benefit is realised by
whoever is debugging across services, which is rarely the team paying the cost.
Treating it as convergent — "here's why structured logs are good for you" — produces
agreement without action, which is exactly what happened.

### Version B — the engineered version

1. You write a **logging starter**: a dependency that, when added, configures the
   encoder, the field names, and the correlation-ID propagation with no other
   changes. Adoption is one line in a build file.
2. You add it to your own service first and leave it under load for two weeks, so
   that the first person to hit a problem with it is you.
3. You open the **pull request** on the friendliest team's service yourself. Their
   cost is reviewing a small diff. In doing it you discover their custom appender
   conflicts, and you fix the starter rather than telling them to change.
4. You build the **cross-service log query** that only works when everyone is on the
   format, and you use it in a real incident, and you share what it showed. Now the
   benefit is visible and concrete rather than argued.
5. You add it to the **parent POM** for new services, so the default for anything
   created from now on is the standard, and nobody has to decide.
6. The last team has a **conflict**: they ship logs to a third-party system that
   expects their existing format. That is a real constraint you did not know about.
   The design changes: the starter gains a compatibility encoder. They adopt.

What changed between A and B is not the argument. It is that adopting went from a
day to ten minutes, the risk went from unknown to demonstrated, and one team's
genuine objection improved the design for everyone.

Note step 6 particularly. In version A that team would simply not have adopted, and
you would have recorded them as resistant. The constraint would have stayed
invisible, and the standard would have been quietly wrong.

---

## Example 2 — the real plan on the project spine

The artefact. The decision to be adopted across three teams:

> **All services that consume `orderflow` events must be idempotent consumers, using
> a shared idempotency-key convention and a shared library, and must not rely on
> exactly-once delivery.**

This is a genuine cross-team decision with real teeth, and it is the right shape for
the exercise because the cost falls on the consumers and the benefit falls partly on
you: the outbox relay (Topic 115) is at-least-once, so duplicates are a property of
the system, and consumers that assume otherwise will corrupt data during any retry
or rebalance (Topic 113).

The three teams are illustrative and you should substitute your own:

- **Team A — notifications.** Small service, consumes order-placed, sends emails.
  Low risk, high availability of capacity.
- **Team B — inventory.** Owns stock levels. Consumes order-placed to decrement.
  Duplicates here mean real inventory corruption. Highest need, highest caution.
- **Team C — analytics.** Consumes everything into a warehouse. Believes duplicates
  are harmless for them because their queries deduplicate at read time. This is the
  team that says no.

### Section 1 — The decision, and the shape of the problem

```markdown
- The decision, in one sentence:
- Shape: [ ] convergent  [ ] externality  [ ] coordination
- Who benefits, and who pays:
- What happens if only two of three adopt:
- Minimum viable adoption:
```

For this decision the shape is **externality mixed with coordination**. Team B pays
and benefits. Team A pays and gets a small benefit (no duplicate emails). Team C pays
and — by their own account — gets nothing. And the value of a shared key convention
is partly coordination: if the key format differs per consumer, the shared library
and the cross-service debugging story both weaken.

Naming this early is what tells you Team C needs a different strategy from Team A,
rather than a louder version of the same one.

### Section 2 — The evidence, and the honest scope of it

```markdown
| Claim | Evidence | Label | Falsifier |
|---|---|---|---|
| Duplicates occur in this system |  | MEASURED / ASSUMED |  |
| Duplicate delivery causes user-visible harm |  |  |  |
| Retry after a rebalance produces duplicates |  |  |  |
| The current consumers are not idempotent |  |  |  |
```

Fill from your own drills — Topic 113's rebalance loop and Topic 115's outbox
behaviour are the direct sources. **Do not assert a duplicate rate you have not
measured.** If you have not measured one, the honest claim is structural: at-least-
once delivery means duplicates are possible by construction, and the consumer must
tolerate them regardless of frequency. That argument is actually stronger than a
frequency number, because it does not invite a debate about whether the number is
big enough to care about.

The trap here is real: a plan that leads with "we saw N duplicates last month" gets
into an argument about N. A plan that leads with "the delivery guarantee is
at-least-once, so correctness cannot depend on frequency" does not.

### Section 3 — The cost ledger, per team

Blank. Fill it by **asking them**, not by estimating.

```markdown
| Cost | Team A (notifications) | Team B (inventory) | Team C (analytics) |
|---|---|---|---|
| Code change |  |  |  |
| Review |  |  |  |
| Test |  |  |  |
| Risk |  |  |  |
| Learning |  |  |  |
| On-call |  |  |  |
| Opportunity / timing |  |  |  |
| **Binding cost (the one that decides)** |  |  |  |
| Non-adoption cost, in their currency |  |  |  |
```

**Strong:** the binding cost differs per team and at least one is not "engineering
hours". For Team B it is almost certainly **risk** — they will not accept a change to
a stock-decrement path without strong evidence. For Team C it is **non-adoption cost
is genuinely near zero for them**, which is a different problem entirely and cannot
be solved by making adoption cheaper.

**Weak:** one estimate applied to all three, produced without talking to anyone.

### Section 4 — What you build to collapse the cost

This is the heart of the plan, and it must be a list of artefacts, not activities.

```markdown
| # | Artefact | Which cost row it removes | For which teams | Effort | Status |
|---|---|---|---|---|---|
| B1 | Idempotency library: key extraction, dedupe store, a `@IdempotentConsumer` wrapper |  |  |  |  |
| B2 | Reference implementation in orderflow's own inventory consumer, under Topic 65 load |  |  |  |  |
| B3 | Documented key convention with the failure modes it prevents |  |  |  |  |
| B4 | An OpenRewrite recipe or a scripted PR that wires the wrapper into a standard consumer |  |  |  |  |
| B5 | A duplicate-injection test harness so a team can prove their consumer survives |  |  |  |  |
| B6 | A dashboard panel + alert: duplicates detected and suppressed, per consumer |  |  |  |  |
| B7 | A starter so new consumers get it by default |  |  |  |  |
```

Two observations about this list.

**B5 is the one that usually decides Team B.** Their binding cost is risk, and risk
is not reduced by a smaller diff. It is reduced by the ability to *prove* the change
is safe on their own service. A harness that replays duplicate events against their
consumer in their test environment converts an act of faith into a test result. Build
the thing that addresses the binding cost, not the thing that is most obviously
useful.

**B6 is what makes the decision self-sustaining.** Once duplicates-suppressed is on a
dashboard, the value is visible continuously rather than argued once, and a
regression is detected rather than discovered. This is the same mechanic as Topic
132's ratchet: the standard survives after your attention moves elsewhere.

### Section 5 — Sequence, with what each step buys

```markdown
| Step | Who | What you do | What it buys | Entry condition | Exit condition |
|---|---|---|---|---|---|
| 1 | You | B1, B2 in orderflow under load |  |  |  |
| 2 | Team A | You open the PR yourself |  |  |  |
| 3 | Team A | Soak period; you own any incident it causes |  |  |  |
| 4 | Team B | Harness first, then PR, then their canary |  |  |  |
| 5 | Team C | The conversation, not the PR |  |  |  |
| 6 | New services | Starter in the parent POM |  |  |  |
```

**Step 3 is the one people skip and it is the one that matters.** Publicly owning the
on-call consequence of your change for the first adopter is the single most
credibility-generating act available to you, and it costs you a few weeks of
attention. It also converts Team A's engineer into an advocate, which is worth more
than anything you can say yourself.

**Step 5 is a conversation, not a pull request.** More below.

### Section 6 — The team that says no

Team C — analytics — says duplicates are harmless because their queries deduplicate
at read time.

**Move 1: assume they are right until you have checked.** Their claim is testable.
Read their pipeline. Deduplication at read time is a real technique. If it holds for
every query, their non-adoption cost is genuinely near zero and your proposal is
asking them to pay for someone else's benefit — which is a legitimate thing to ask
for, but you must ask for it honestly rather than pretending it helps them.

**Move 2: look for the case where their claim breaks.** Deduplication at read time
usually holds for counts over a full window and breaks in specific places: an
incremental aggregate that adds rather than recomputes; a first-seen-timestamp field;
a funnel with per-event sequencing; any downstream consumer of their tables who does
not know about the read-time dedupe. If you find one, you have converted an
externality into a convergent problem, and the conversation changes completely.

**Move 3: if their claim holds, negotiate a smaller thing.** They do not need the
full library. What you actually need from them is that the key is *present and
propagated*, so cross-service debugging and any future dedupe remain possible. That
is a much smaller ask, and asking for the smaller thing after understanding their
position is what makes the relationship survive.

**Move 4: if it still does not land, decide whether to escalate — and usually do
not.** Ask two questions. First: is the value of Team C's adoption worth what the
escalation costs you? Second: if I am overruled, what have I lost? For this decision,
partial adoption is fine for the correctness goal; the coordination value is real but
smaller. So you accept, write it down, and set the trigger for revisiting: *"If
analytics ever adds an incremental aggregate, or if any team consumes their tables
directly, this returns."*

**Move 5: write down that you were possibly wrong.** If Team C's objection revealed
that your key convention assumed a synchronous producer context that their pipeline
does not have, the design changes. Say so in the plan. A plan that cannot record its
own defeat is a rollout schedule with a document wrapped round it.

```markdown
### The objection register

| Team | Objection, in their words | Is it a constraint, a cost, or a preference? | What I checked | Outcome | Design change |
|---|---|---|---|---|---|
|  |  |  |  |  |  |
```

The middle column is the analytical work. **Constraint:** something that makes
adoption impossible or unsafe for them — your design must change. **Cost:** it is
possible but expensive — your adoption plan must change. **Preference:** they would
rather not — the conversation is about the organisation's priorities and is
legitimate, but it is a different conversation and should be named as one.

Most proposers hear all three as the same word ("no") and respond to all three with
more argument. Sorting them is most of the skill.

### Section 7 — What you will not do

```markdown
- I will not mandate this unless:
- I will not escalate before:
- I will accept partial adoption if:
- I will abandon this if:
```

**Strong:** an abandonment condition that is real. *"If Team B's canary shows the
dedupe store adds latency beyond their budget, this design is wrong and I will
withdraw it rather than argue for an exception."*

**Weak:** no abandonment condition, which means the plan cannot fail, which means it
is not a plan.

### Section 8 — How you will know it worked

```markdown
| Signal | Source | What it tells you |
|---|---|---|
| Consumers on the library |  |  |
| Duplicates suppressed, per consumer |  |  |
| New consumers adopting without being asked |  |  |
| Someone else answering questions about it |  |  |
| The library changed because a team needed it to |  |  |
```

The last two are the real ones. A standard that only you can explain has not been
adopted; it has been installed. A library that has never changed in response to a
consumer's need is one that consumers are working around.

---

## Wrong approach → exact symptom → root cause → fix

Five. Organisational symptoms, all concrete.

---

### Wrong approach 1 — the decision is announced in an architecture forum and never adopted

**Exact symptom.** You present a well-argued proposal. Nobody objects. Two or three
people say it is a good idea. The document is circulated and linked in a wiki. Three
months later, one service uses it: yours. When you ask, nobody says no — they say
"yes, it's on our list". At the next forum someone proposes something adjacent and
unaware, because your document was read once and filed. Meanwhile, you have started
to be seen as the person with ideas rather than the person who ships them, which is
an expensive reputation to acquire and slow to shed.

**Root cause.** Adoption is a cost comparison, and the proposal changed neither side
of it. Agreement is free; adoption is not. The forum's function is coordination and
awareness, and it is very good at those; it has no mechanism for making work happen
inside teams whose backlogs are owned elsewhere. Treating agreement as the finish
line is the single most common Principal-track mistake, and it feels like progress
right up until the quarter ends.

**Fix.** Invert the order: build first, present second. Arrive at the forum with the
reference implementation running under load, one external team already migrated by a
pull request you opened, and a five-minute path for everyone else. The presentation
stops being a request and becomes an announcement of something that already exists,
which changes both the cost column and the risk column at once. If you cannot build
first — sometimes you genuinely cannot — then the forum output must be a named owner
and a date per team, not a nod, and you should say plainly in the room that a nod
without a date has historically produced nothing.

---

### Wrong approach 2 — the cost is estimated by the proposer rather than asked of the adopter

**Exact symptom.** Your plan says "this is a half-day change per service". Team B
takes six weeks. Not because they are slow: their consumer has a bespoke retry path
that interacts with your dedupe store, their release process requires a change
advisory for anything touching stock levels, and their staging environment cannot
replay events so the only test is production. None of that was visible from outside.
Meanwhile you have told a director it is a half-day change, so Team B now looks
obstructive while doing genuinely careful work, and their trust in you drops in a way
that is difficult to repair.

**Root cause.** The proposer estimates the part they can see — the code — and the
code is usually the smallest cost. Risk, process, test infrastructure and timing are
invisible from outside and often dominate. Underneath is a subtler error: you
estimated the cost for *your* codebase, where you know the paths and own the
deployment, and generalised it to codebases you have never read.

**Fix.** Ask, before writing the plan, and ask in the form that gets a real answer:
not "how long would this take" (which invites a polite guess) but "walk me through
what would have to happen for this to ship in your service". You will hear about the
change advisory board and the unreplayable staging environment in the first two
minutes. Then put their number in the plan, attributed to them, and — critically —
build the artefact that addresses their *binding* cost rather than the code cost. For
Team B that is the replay harness. And when you report upward, report their estimate,
not yours.

---

### Wrong approach 3 — the first team is the most important team

**Exact symptom.** You start with inventory, because that is where duplicate events
actually corrupt data and where the value is highest. They are also the most cautious
team in the organisation, with the most to lose, and they say not yet — reasonably.
Now you have a public no from the highest-stakes team, and Teams A and C read it as a
signal that the change is risky. The proposal is dead in a quarter, and reviving it
costs more than starting it did, because the first fact anyone recalls is that
inventory turned it down.

**Root cause.** Optimising for value per adoption instead of for the probability of
the first adoption. The first adopter's real job is not to capture value; it is to
convert your change from a proposal into a thing that exists in production somewhere
you did not write. That evidence is what makes the second conversation different.
Starting at the top also inverts the risk profile: the team least able to absorb an
unproven change is asked to absorb it first.

**Fix.** Sequence by cost and capacity, explicitly, and write the reason in the plan
so it does not look like you avoided the hard team. Team A first: small service,
available capacity, low blast radius, and you open the PR. Soak it. Then approach
Team B with three things they did not have before — a service other than yours
running it, an engineer they trust who can describe the experience, and the replay
harness that lets them verify rather than trust. If the important team must go
first for a real reason (a deadline, a dependency), then invest much more heavily up
front: run their duplicate-injection test yourself, on their code, before you ask
them for anything.

---

### Wrong approach 4 — the team that says no is treated as an obstacle rather than as information

**Exact symptom.** Team C objects. You respond with a better version of the same
argument, then a longer document, then a meeting with more people in it. They repeat
their position, more briefly each time. Eventually you escalate; a director tells
them to adopt; they comply minimally and late, and the integration is fragile because
they did the smallest thing that satisfied the mandate. Their tech lead stops
bringing you early design questions, which costs you visibility into their systems
for the next two years — a much larger loss than the adoption you won.

**Root cause.** The objection was never classified. "No" was heard as resistance
rather than being sorted into constraint, cost, or preference, and all three were
answered with argument, which only ever addresses preference. Underneath is an
identity problem: once you have publicly proposed something, an objection feels like
a challenge to you rather than data about the world, and the reflex is to defend.

**Fix.** Classify the objection out loud, with them: *"I want to understand whether
this is impossible for you, expensive for you, or something you would rather not
do — because my response is different in each case."* That sentence is disarming
because it is genuine. If it is a constraint, your design changes and you say so
publicly, which costs you nothing and buys a great deal. If it is cost, build the
thing that removes it. If it is preference, name it as an organisational priority
question and take it to the forum as a trade, not as a dispute. And keep an objection
register in the plan, because the pattern across objections usually reveals something
about your design that no single objection does.

---

### Wrong approach 5 — the standard is adopted and then quietly rots

**Exact symptom.** All three teams adopt. Six months later, two services are on an
old version of the library that predates a bug fix; a new service was created without
the starter because it was scaffolded from an older template; and one team has
disabled the dedupe wrapper on a hot path for latency reasons without telling anyone.
Nobody is aware anything has drifted, because nothing measures it. The next duplicate
incident arrives on a service that appears, in the wiki, to be compliant.

**Root cause.** The plan optimised for the moment of adoption and had no mechanism
for persistence. Adoption is a state, not an event, and states drift toward whatever
is cheapest to maintain. Your attention was the only force holding it in place, and
your attention moved to the next thing — correctly, because that is the job.

**Fix.** Ship the persistence mechanism as part of the plan, not as a follow-up.
Three concrete forms, in ascending strength: a **dashboard** showing which services
are on the library and at which version, so drift is visible without anyone
investigating; an **alert** on suppressed-duplicates going to zero on a consumer that
previously had a non-zero rate, which detects a silently disabled wrapper; and a
**parent POM plus starter** so that new services get it without deciding, which is
the only form that survives complete inattention. Then add one social mechanism: the
library must have an owner who is not you, appointed before you stop paying
attention, because an unowned shared library is a future migration nobody has
budgeted.

---

## Artefact — what you must produce

**One written adoption plan.** `docs/java/plans/<date>-<decision>-adoption.md`,
committed.

**The decision must be real and cross-team.** Use the idempotent-consumer decision
above, or one of: a shared tracing propagation and correlation-ID convention across
services; a metric-cardinality standard protecting the monitoring platform; a
mandated JDK or Boot baseline with a CI deadline; a shared resilience configuration
for calls to a common downstream.

Pick one where **at least one team genuinely pays for someone else's benefit**. A
purely convergent decision makes the exercise too easy and teaches you the wrong
lesson.

### Format and length

- **3 to 6 pages.** Under 3 and the per-team cost analysis is missing. Over 6 and you
  are writing a project plan; this is a plan for changing behaviour, not a schedule.
- The **cost ledger**, the **build list**, and the **team-that-says-no** section must
  be readable on their own.
- Three real teams. If you do not have three, use `orderflow`'s consumers and say so.

### Required sections

1. **The decision** in one sentence, plus its shape — convergent, externality, or
   coordination — and who pays versus who benefits.
2. **Evidence**, labelled `MEASURED` / `ESTIMATED` / `ASSUMED`, with a falsifier for
   each claim. Where you have no measurement, the structural argument, stated as
   such.
3. **Cost ledger per team**, with a named binding cost per team, and the non-adoption
   cost in *their* currency. At least one binding cost must be something other than
   engineering hours.
4. **What you will build** — a table of artefacts, each mapped to the cost row it
   removes and the team it removes it for, with effort estimates.
5. **Sequence**, with entry and exit conditions per step, and a stated reason for the
   ordering.
6. **The team that says no** — the objection classified as constraint, cost, or
   preference; what you would check; the smaller ask you would fall back to; and the
   condition under which you would accept the no.
7. **What you will not do** — mandate conditions, escalation conditions, partial-
   adoption acceptance, and an **abandonment condition**.
8. **Persistence** — how the decision survives after your attention moves, including
   a named owner who is not you.
9. **How you will know it worked** — signals, including at least one that is about
   other people rather than about the code.

### Required content — the specific things I will check for

- **At least one artefact you would build that costs you more than a day.** If
  everything in the build list is a document or a snippet, you have written a
  communication plan.
- **A per-team binding cost that you obtained by asking**, marked as such, not
  estimated.
- **A named condition under which your design is wrong**, and what you would do.
- **The smaller ask** you would fall back to with the objecting team.
- **A statement of what you would spend on the credibility budget**, and whether this
  decision is worth it.
- **At least one thing you would do for another team that is not your job** — opening
  their pull request, running their test, owning the on-call consequence.
- **No mandate as a first move.** If a mandate appears, it must have a condition and
  a stated cost.

---

## How I will review it

I will not comment on prose or plan formatting. I will attack the assumption that
your cost estimates are right, the sequence, and the part where you have written
persuasion where engineering was available.

### The three questions that usually break an adoption plan

**Question 1: "What are you building that makes adopting this cheaper than ignoring
it — and how many days is it?"**

If the answer is a document, a presentation, and a wiki page, the plan is a
communication plan wearing an engineering plan's clothes.

This breaks most plans because building the adoption path is real work, unglamorous,
and the proposer usually assumes it belongs to the adopting teams. It does not. It
belongs to whoever wants the change.

What a good answer sounds like: *"Four things, about three weeks of my time. The
library, which is the obvious one. The reference implementation in orderflow under
load, which is what makes it credible. A duplicate-replay harness so a team can
verify safety on their own service without trusting me — that one is specifically for
inventory, because their binding cost is risk, not effort. And the dashboard, because
otherwise this rots. I open the first two external pull requests myself."*

Follow-ups:

- "Three weeks of your time is real. Who approved it, and what did it displace?" A
  Principal who cannot answer this is planning with imaginary capacity.
- "The harness is for inventory. How do you know risk is their binding cost?" If the
  answer is "I assumed", we are on to question two.
- "Which of the four would you cut if you had one week?" The answer should be the
  library last and the dashboard first — persistence is the thing you cut and then
  regret.

**Question 2: "Which team pays the most for this, and what did they tell you about
why?"**

I am checking whether you have modelled other teams' incentives or only your own
reasoning. The signal I am listening for is whether the answer contains something
specific that you could not have known without talking to them.

The failure mode: an answer that describes what the team *should* care about.

What a good answer sounds like: *"Inventory. Not because the code change is large —
it is small — but because it touches the stock-decrement path, which goes through a
change advisory board, and their staging environment cannot replay events so their
only real test is production. Their engineer's words were roughly 'we can't test
this'. That is why I am building the replay harness before I ask them for anything.
Analytics pays least in effort and gets least in benefit, which is a different
problem and needs a different answer."*

Follow-ups:

- "You are asking analytics to pay for someone else's benefit. Say that out loud to
  them — how does that conversation go?" A candidate who cannot say it plainly will
  end up pretending it helps them, which is the thing that destroys trust.
- "Inventory's advisory board adds two weeks. Does your sequence account for it, or
  does your plan assume they will do it when you are ready?" Plans that assume other
  teams have free capacity at your convenience are the most common cross-team
  failure.
- "What would you do if inventory's engineer had said 'we could do it next month but
  not now'?" The right answer is usually to take next month and use the interval to
  make it cheaper, not to argue for now.

**Question 3: "The third team still says no after all of that. What do you do, and
what would make you conclude they are right?"**

Two halves. The first tests whether you have a plan that survives disagreement. The
second tests whether you can be wrong.

The failure mode I am watching for is either of the two Topic 135 failures, showing
up early: a candidate who folds immediately ("then I'd drop it") and one who cannot
imagine being wrong ("I'd escalate"). Both read the same way.

What a good answer sounds like: *"First I classify. If it is a constraint — their
pipeline genuinely cannot carry the key through its ingestion path — my design is
wrong for them and I change it or scope them out, publicly, because the scoping-out
is what makes the other two trust the rest. If it is cost, I build the thing that
removes it. If it is preference, I take a smaller ask: I do not need the full library
from them, I need the key present and propagated. If they still say no, I check
whether partial adoption realises the value — here it does for correctness — accept,
write down the trigger that brings it back, and do not spend an escalation on it. And
the thing that would make me conclude they are right is if their read-time
deduplication genuinely covers every query and no downstream consumer reads their
tables directly. I can check that in an afternoon and I would rather check than
argue."*

Follow-ups:

- "You checked and their dedupe holds everywhere. Do you drop the requirement for
  them?" Yes — and the interesting part is what you write down, because the next
  person will re-propose it in a year.
- "Your director offers to mandate it. Do you take it?" Almost always no, and the
  reason should be about the cost to future information flow, not about principle.
- "If you scope them out, does the coordination value survive?" This is where a
  candidate either notices the difference between the correctness goal and the
  convention goal, or does not.

### The other attacks, in order

**On the evidence:**

- "You claim duplicates cause harm. Measured, or structural?" Both are acceptable;
  confusing them is not.
- "If your duplicate rate is very low, why is this worth three weeks?" The right
  answer is that at-least-once makes it a correctness property rather than a
  frequency one — and if you lead with a rate, you have invited exactly this
  argument.

**On the sequence:**

- "Why is team A first? Say the reason out loud." If the honest reason is that they
  are the easiest, say so; hiding it makes it look like avoidance.
- "What is the exit condition on the soak period, and what do you do if it produces
  an incident?" Owning that incident is the highest-value thing in the plan and it
  should be explicit.

**On persistence:**

- "Who owns the library in a year?" If the answer is you, you have created a
  dependency on yourself and called it a standard.
- "How would you find out that a team disabled it?" If there is no answer, the plan
  has no persistence mechanism.
- "What happens when someone creates a new service?" The starter-and-parent-POM
  answer is the one that survives inattention.

**On the credibility budget:**

- "Is this decision worth a mandate if it comes to that?" A candidate who says yes to
  everything has not thought about the budget.
- "What is the more important thing you are saving credibility for?" There should be
  one.

### What I will not attack

- A plan that accepts partial adoption, with the reasoning stated.
- A decision to abandon, with the discovery that caused it. That is the plan working.
- An objection that changed your design. That is the best outcome available and I
  will say so.
- Timelines that are long because other teams have their own cycles.

---

## Interview questions (Senior → Principal)

### Q1 — "How do you get other teams to adopt a technical decision?"

**A Senior answer.** "I write a proposal, take it to the architecture forum, get
alignment, and follow up with the teams." Reasonable, describes a real process, and
describes the process that produces the wiki page.

**A Principal answer.** *"Mostly by building, not by arguing. Adoption is a cost
comparison each team makes in their own currency, and the lever I control is the cost
of adopting. So: a reference implementation running under real load in a service I
own; then I find the cheapest external team and open their pull request myself, which
also tells me what my instructions got wrong; then I use them as the evidence for the
next team. For the team whose binding cost is risk rather than effort, a smaller diff
does nothing — I build the harness that lets them verify safety themselves. The forum
is where I announce that it exists, not where I ask for permission."*

**What separates them.** The Senior answer treats adoption as a persuasion problem
with a process. The Principal answer treats it as a cost problem with an engineering
solution, and — crucially — differentiates the *binding* cost per team rather than
assuming everyone is deciding on the same axis.

**Adversarial follow-up.** *"That is a lot of your time spent on other teams' code.
Is that the best use of a Principal engineer?"* A real question, not a trap. The
strong answer prices it: three weeks of my time to change how five services handle
duplicate events permanently is a better return than three weeks of my own feature
work, *and* the alternative is not free — the version where I only write documents
costs the same weeks over a longer period and produces nothing. If the answer to
"what does this displace" is something more valuable, then this standard is not worth
doing, and saying so is also a correct answer.

---

### Q2 — "Tell me about a time you couldn't get something adopted."

**A Senior answer.** Describes a proposal that did not land and attributes it to
priorities, reorganisation, or a team that was not receptive. All of those can be
true, and none of them is about anything the speaker controlled.

**A Principal answer.** *"I proposed a shared resilience configuration for calls to a
common downstream. It was correct and it went nowhere for a quarter. The mistake was
mine: I priced it as a two-hour change because it was two hours in my service, and
for the team I most needed it in, the binding cost was that their integration tests
called the real downstream, so any change to timeout behaviour broke fifty tests.
Nobody told me because I never asked in a way that would surface it — I asked how
long it would take and got a polite guess. When I finally sat with their engineer,
that came out in five minutes. I ended up fixing their test harness, which was not
my job and was the actual unblock. What I do differently now is ask 'walk me through
what would have to happen for this to ship in your service' before I write anything
down."*

**What separates them.** The Senior answer locates the cause outside. The Principal
answer locates a specific, correctable error in their own method, and names the
behavioural change. It also demonstrates the willingness to do unglamorous work in
someone else's codebase, which is the actual currency of this skill.

**Adversarial follow-up.** *"Fixing their test harness sounds like you did their work
for them. Does that scale?"* No, and the honest answer says so: doing it once bought
the adoption and taught me the real cost; doing it every time would be a full-time
job. The scalable version is that the second team got a documented pattern for the
same problem and did it themselves, and the third got a starter. If the candidate
claims it scales as-is, they have not thought about their own time as a constrained
resource.

---

### Q3 — "A team refuses your proposal. Walk me through what you do."

**A Senior answer.** Understand their concerns, address them, escalate if necessary.
Correct as a sequence and empty as a method — every step is a category, not an
action.

**A Principal answer.** *"I classify the no into one of three things, and I say the
classification out loud to them, because the sorting is not adversarial. Is this
impossible for you, expensive for you, or something you would rather not do? If it is
impossible — a real constraint I did not model — then my design is wrong for their
case, and I change it or scope them out publicly. Publicly matters: the other teams
watching learn that objections change the design, which is what makes them willing to
raise theirs. If it is expensive, that is my problem to solve with engineering. If it
is preference, then it is a priority question and belongs in a forum as a trade, not
as a dispute between us. And before any of that I check the objection myself, because
the fastest way to end a disagreement is to discover they are right."*

**What separates them.** The Senior answer is a process. The Principal answer is a
*classification with different responses per class*, plus an awareness that the
handling is watched by the other teams and sets the terms for their objections too.

**Adversarial follow-up.** *"They tell you it's impossible and you think they're
wrong. Now what?"* The good answer does not adjudicate by assertion. It proposes to
find out cheaply: "let me try it on a branch of your service" or "let me run your
test suite against it". Being willing to do the work to resolve a factual
disagreement — rather than arguing about it, or deferring to seniority in either
direction — is the specific behaviour being tested. And if it turns out you were
wrong, saying so quickly and publicly is worth more than the adoption was.

---

### Q4 — "When do you use a mandate?"

**A Senior answer.** "When it's important enough, or when it's a security or
compliance issue."

**A Principal answer.** *"Rarely, and I treat it as a loan against a budget I will
need later. There are cases where it is right: a security fix, a legal requirement, a
JDK baseline with a hard end-of-support date, or a coordination decision where
partial adoption is worse than none and someone has to go first. Even then I would
rather ship the migration tool with the mandate, so that complying is cheaper than
negotiating an exception. What I avoid is using a mandate because I could not be
bothered to make adoption cheap — the cost is not the one meeting, it is that the
teams learn my route to agreement is escalation, and they stop bringing me problems
early. That early visibility is where most of my value comes from, and it is
expensive to get back."*

**What separates them.** The Senior answer treats a mandate as a tool with a
threshold. The Principal answer treats it as a purchase with a price paid in future
information flow, names the legitimate cases specifically, and still pairs it with the
cost-reduction work.

**Adversarial follow-up.** *"Your director wants to mandate it tomorrow because they
are impatient. What do you say?"* The strong answer does not refuse and does not
comply silently: it asks for two weeks to land the first team voluntarily, on the
grounds that a mandate applied after one visible success is cheap and one applied
cold is expensive — and offers the mandate as the fallback with a date. That is a
negotiation with your own management, and it is the same mechanic as everything else
in the topic.

---

### Q5 — "How do you mentor an engineer who does not report to you?"

**A Senior answer.** Regular one-to-ones, answer their questions, review their code,
share knowledge.

**A Principal answer.** *"The same mechanic as adoption: I make the good path
cheaper than the bad one for them specifically. Concretely, three things. I review
their design before they write code rather than reviewing the code, because a review
at the end can only produce rework and rework is the expensive thing. I give them the
problem rather than the answer, and then I tell them what I would have watched for —
the answer teaches one thing, the watch-list teaches a class. And I make sure their
work is visible to the person who writes their performance review, because mentoring
someone whose manager cannot see the improvement is a favour to them and does nothing
for their career. The failure mode I watch for in myself is becoming the person they
ask instead of the person who taught them how to find out — if they are still asking
me the same shape of question after three months, I have been answering rather than
teaching."*

**What separates them.** The Senior answer lists activities. The Principal answer
names the mechanism (design review before implementation, class over instance),
includes the organisational half that engineers routinely forget, and states a
self-check for the characteristic failure.

**Adversarial follow-up.** *"Their manager disagrees with the direction you're giving
them. What do you do?"* You do not put the engineer in the middle — that is the one
thing that must not happen, and it is the failure mode of well-meaning cross-team
mentorship. You take the disagreement to the manager directly, and if it is not
resolved, you defer on anything within their team's scope and confine your input to
technique. An engineer receiving contradictory direction from two senior people will
do worse work and will pay the cost of a disagreement that is not theirs.

---

## Mental model checkpoint

Answer without looking.

1. State the adoption inequality in one line, in whose currency each side is
   measured, and which side you can actually move.

2. A decision was presented at an architecture forum, agreed by everyone, and not
   adopted. What happened, and what is wrong with describing it as "rejected"?

3. Name the three shapes of cross-team decision, give an `orderflow` example of each,
   and say why an externality answered with a convergent strategy produces a polite
   yes and no action.

4. What are the three classifications of a "no", and what is the correct response to
   each? Which one is the only one that argument can address?

5. Why do you sequence adoption by lowest cost rather than by highest need, and what
   specifically does the first external adopter buy you that your own service cannot?

6. Your standard was adopted by three teams and has rotted in six months. Name the
   three mechanisms that would have prevented it, in ascending order of strength.

7. What is the credibility budget, what spends it fastest, and what refills it? Name
   the specific future cost of using a mandate to win a medium-value standard.

---

## Quick reference card

### The mechanic, in one line

> Adoption happens when adopting is cheaper than not adopting, in the adopter's
> currency. You cannot raise the cost of the status quo. You can drive the cost of
> adopting to nearly zero with engineering work.

### The three shapes

- **Convergent** — local benefit; cost is the only obstacle.
- **Externality** — someone else benefits; move the cost or make the benefit visible.
- **Coordination** — value requires critical mass; sequence, do not persuade.

### The cost ledger rows

code · review · test · risk · learning · on-call · opportunity

Find the **binding** row per team. It is often not the code.

### What "cheap to adopt" means in Java

OpenRewrite recipe · parent POM / BOM · Spring Boot starter with safe defaults ·
reference implementation under real load · replay or verification harness ·
ratcheting static-analysis check · the pull request you open yourself

### Sequence

1. Reference implementation in your own service, under load.
2. Cheapest external team; you open the PR.
3. Soak; you own the on-call consequence.
4. Highest-need team, armed with someone else's production experience.
5. The objector — a conversation, not a PR.
6. Starter and parent POM for everything new.

### Classifying a "no"

- **Constraint** → your design changes. Say so publicly.
- **Cost** → your adoption plan changes. Build the thing.
- **Preference** → a priority conversation in a forum, as a trade.

Only preference can be addressed by argument. Most people argue at all three.

### Persistence, weakest to strongest

dashboard of who is on it → alert on regression → parent POM + starter (survives
inattention) → a named owner who is not you

### Anti-patterns, one line each

- Announcing in a forum and calling agreement adoption.
- Estimating other teams' costs yourself.
- Starting with the most important team.
- Answering a constraint with an argument.
- A mandate as the first move.
- A shared library only you can explain or maintain.
- No abandonment condition, so the plan cannot fail.

---

## When would I use this at work?

**1. Any time you find yourself writing a second document to explain the first one.**
That is the signal that you are in a cost problem and treating it as a comprehension
problem. Stop writing and ask one team what would have to happen for the change to
ship in their service. The answer is almost always something you can build, and it is
almost never what your document was addressing.

**2. When a postmortem action item needs to apply outside your codebase.** Topic 133
will produce items like "a wrapper that makes this bug unwritable" — which prevents
the class only in the repository that has the wrapper. Getting it into three other
services is precisely this topic, and the postmortem is the most persuasive artefact
you will ever hold, because it is evidence rather than opinion. Use it while it is
recent; its persuasive value decays within weeks.

**3. When you are new and have no political capital at all.** This is the best time
to use the mechanic, because you have nothing else. You cannot escalate, nobody owes
you a favour, and your opinion carries no weight yet. Building the thing that makes
someone's work cheaper is the one move available to a newcomer, and it produces
credibility rather than spending it. Pick something small, unglamorous, and
immediately useful to another team's on-call engineer — that is worth more in the
first three months than any correct opinion you could offer.

---

## Connected topics

**Prerequisites — the decisions you will be trying to get adopted:**

- **114, 115, 116 — Kafka delivery semantics, the outbox, idempotency.** The worked
  example. At-least-once delivery is a structural argument rather than a frequency
  one, which is why it survives the "is the duplicate rate high enough" objection.
- **113 — Consumer groups and rebalancing.** The mechanism by which duplicates
  actually arrive, and therefore the evidence you cite.
- **118 — Metrics and cardinality.** The cleanest externality in the curriculum: the
  cost of a high-cardinality tag falls on the monitoring platform, not on the team
  that emits it. Almost impossible to solve by argument, straightforward to solve
  with a wrapper and a lint check.
- **119 — Tracing and context propagation.** The cleanest coordination problem: a
  propagation convention is worth nothing until enough services follow it, and the
  first adopter pays full cost for no benefit.
- **65 — The baseline.** Unusually persuasive evidence in a Java organisation,
  because it lets a reference implementation be described as "running under this load
  with these numbers" rather than "working on my branch".

**Built on:**

- **126 — Build vs buy vs adopt.** Adoption cost and exit cost are the same analysis
  turned outward; a framework choice imposed on other teams is this topic with a
  larger blast radius.
- **127, 128 — Migration planning.** A migration is the largest possible adoption
  problem, and the stall at 60% has exactly this cause: teams stop when their local
  pain stops. The forcing function there and the persistence mechanism here are the
  same idea.
- **131 — Design docs.** The doc makes the decision legible; this topic makes it
  happen. A design doc with a "who is affected" section and no adoption plan has
  described a change that will not occur.
- **132 — Engineering standards.** The ratchet is the persistence mechanism, and the
  credibility-budget idea comes from there. A gate that fails four thousand existing
  warnings is the canonical example of raising adoption cost by accident.
- **133 — Postmortems.** Action items that need to cross team boundaries arrive here.
  A postmortem is the cheapest adoption argument you will ever have, and it expires.

**This unlocks:**

- **135 — The capstone.** "Tell me how you got something adopted" and "what did you
  do when they said no" are standard Principal prompts, and the follow-up chain goes
  straight to what you built and what the objection taught you. This plan is one of
  the artefacts you will defend under sustained hostile questioning — and the
  reversal question in Topic 135 is often answered from exactly this material,
  because the team that says no is the most common source of a genuine reversal.

---

*This document contains no adoption statistics, no benchmark figures, and no quoted
methodology. Every table describing your organisation is blank by design; fill them
by asking the teams rather than by estimating on their behalf, which is the single
most consequential habit in the topic. Where a practice is widely used — paved roads,
reference implementations, ratcheting standards — it is described as a mechanism, not
attributed to a source I have not read to you.*
