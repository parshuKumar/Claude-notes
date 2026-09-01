# 133 — Incident Command and Blameless Postmortems

## Phase: 12 — Principal Track
## Category: ELITE
## Java baseline: 21  |  Notes features from: 21/25
## Project spine: a full blameless postmortem for one of your Phase 8/9 drills, treated as a real incident — timeline, contributing factors, and an action-item chain where every item has an owner, a due date, and a stated preventive effect.

---

## Before anything else — what is and is not in this document

**What is in it.** A method for writing a postmortem whose output is a chain of
action items that each prevent a *class* of incident. A blank timeline and action
table you fill in. One fully worked example built on a drill you actually ran. Five
ways postmortems fail organisationally, and the attacks I will make on yours.

**What is not in it, deliberately.**

- **No invented incident.** There is no fictional outage in this document with
  timestamps, customer-impact figures, or revenue numbers. The worked example uses
  the **Topic 101 virtual-thread pinning drill** — something you produced yourself,
  on purpose, and watched. Every clock time in the worked example is written as a
  relative marker (`T+0`, `T+8m`) or a blank you fill from your own drill notes. If
  I wrote "14:32 — p99 hit 4.2s", you would remember the number, and the number
  would be fiction.
- **No impact statistics.** No "the average incident costs $X". No "teams that write
  postmortems reduce MTTR by Y%". Those numbers exist in vendor marketing and I am
  not going to launder them into your head.
- **No claim of a canonical format.** Blameless postmortems are a widely used
  industry practice, popularised largely by Google's SRE literature and by the
  broader web-operations community over the last fifteen years. The norm — separate
  the person from the system, look for contributing factors rather than a culpable
  individual — is theirs and is well established. The specific structure below is
  mine, built from this curriculum's drills. I am not quoting anyone.
- **No code.** The artefact is a document. There is no coding exercise here.

**What I am asking you to trust.** That "root cause: someone forgot" is not a
finding, and that the difference between a postmortem that changes a system and one
that produces a wiki page is almost entirely in the action-item chain.

---

## Mechanical statement

**A postmortem's output is a chain of action items that each prevent a CLASS of
incident. "Root cause: someone forgot" prevents nothing, because the next person
will also forget.**

Three consequences, and they are the document:

1. **Contributing factors, not a root cause.** Every non-trivial incident has
   several conditions that all had to be true. Naming one of them "the root cause"
   is a choice about where to stop looking, and the place people stop is almost
   always at a human decision — because human decisions are legible and system
   properties are not.
2. **Each action item must name the class it prevents.** Not "fix the bug". "Prevent
   any future code path from blocking a carrier thread inside a monitor" is a class.
   The test is: *would this action item have prevented three other incidents you
   have not had yet?* If the answer is no, you have written a repair, not a
   preventive.
3. **Owner, date, and preventive effect, or it is not an action item.** All three.
   A list of good intentions with no owner is the single most common way a good
   postmortem produces nothing.

The deeper mechanic, and the one that makes this a Principal topic: **the strength
of an action item is a property of where it sits in the control hierarchy, not of
how well it is written.** Documentation is the weakest control. A checklist is
stronger. An alert is stronger still. A gate in CI is stronger than that. A wrapper
that makes the bug *unwritable* is the strongest. Choosing the highest achievable
level for each contributing factor is the skill.

---

## The bridge from what you know

### What transfers — completely, and I will not re-teach it

You have been on call. You have run and written incident reviews in TypeScript-land.
You already know:

- How to build a timeline from logs, alerts and chat scrollback.
- That "who deployed it" is a useless question and "what let it reach production" is
  a useful one.
- How to run the meeting: no blame, facts before conclusions, one person writing.
- That the person closest to the incident is the person with the most information
  and the least willingness to speak if they feel accused.
- That MTTR is dominated by *time to correct diagnosis*, not time to fix.
- How to chase action items, and that most of them die quietly.

None of that changes because the runtime is a JVM. If you have written good incident
reviews before, the meeting-running half of this topic is already yours.

### What is genuinely harder at Principal level

**One: separating contributing factors from a single "root cause".** This is the
skill, and it is harder than it sounds because a single cause is *satisfying*. It
closes the discussion, it fits in a summary line, and management asks for it by
name. Producing four contributing factors instead requires you to keep an
uncomfortable conversation open past the point where everyone wants it closed, and
to resist the specific gravitational pull toward the last human action in the chain.

**Two: writing an action item that prevents a class.** Most engineers can fix the
instance. Fewer can name the class. Fewer still can identify the strongest available
control for that class rather than the most obvious one. "Add a code-review checklist
item" is what you write when you do not know the hierarchy; a factory that refuses
to construct the dangerous object is what you write when you do. Topic 132 taught
you the hierarchy; this topic is where you have to choose from it under time
pressure, with an audience that wants the meeting to end.

**Three: keeping it blameless while still being specific.** Blameless does not mean
vague. A postmortem that will not say what happened, in order, because naming
actions feels like naming people, is useless. The discipline is to describe actions
precisely and attribute causes to *conditions* — the default that was unsafe, the
signal that did not exist, the review that had no way to catch it.

### The Java-shaped part

Your incidents are JVM-shaped, and the contributing factors have Java-specific
forms that a Node engineer's postmortem vocabulary does not contain:

- **Unsafe library defaults.** `Executors.newFixedThreadPool` gives you an unbounded
  queue. `LinkedBlockingQueue` with no capacity argument is unbounded. Neither is a
  mistake by the person who called it; both are defaults that make the dangerous
  option the easy one. That is a contributing factor, and the action item that
  addresses it is a wrapper, not a warning.
- **Failure modes with no external signal until they are terminal.** F2's plateau,
  the unbounded `Flux` buffer that grows silently until OOM, the pinned carrier that
  presents as idle CPU. Each is a contributing factor of the form "there was no
  signal", which is a different kind of finding from "there was a bug".
- **Diagnostics that must be captured before the mitigation destroys them.** A
  restart clears a deadlock, a pinning stall and a pool exhaustion equally well, and
  it destroys the evidence for all three. "We restarted and it recovered" is how an
  incident becomes unlearnable, and preventing that is itself an action item.

---

## What is this?

A **postmortem** is a written account of an incident, produced after recovery, whose
purpose is to change the system so that a class of incident becomes less likely,
less severe, or faster to detect.

Terms, defined once and used precisely.

**Incident.** An unplanned interruption or degradation of a service that mattered to
someone outside the team. For this exercise, a drill counts, because you are
practising the artefact.

**Blameless.** A property of the *analysis*, not of the tone. It means: assume every
person acted reasonably given the information and incentives they had, and treat
their action as evidence about the system rather than about them. If an engineer
made a mistake that was easy to make, the finding is that it was easy to make.

Blameless does not mean nobody is named, does not mean no action items land on
individuals, and does not mean avoiding uncomfortable facts. It means the *causal*
account stops attributing to persons what belongs to conditions.

**Timeline.** An ordered list of events with timestamps, drawn from evidence and not
from memory. It contains three kinds of entry: what the system did, what the humans
did, and what the humans *believed* at that moment. The third is the one people omit
and it is where most of the learning is.

**Contributing factor.** A condition without which the incident would not have
happened, or would have been less severe or shorter. Incidents have several. Some
are technical, some are procedural, some are organisational.

**Root cause.** A term I would like you to stop using in this document. Its real
meaning is "the place we decided to stop asking why", and it hides that decision.
Where you must produce one for an external template, write "primary contributing
factor" underneath it and list the others.

**Trigger.** The event that started this occurrence. Different from a contributing
factor: the trigger is why it happened *today*; the factors are why it was possible
at all. A deploy is usually a trigger and almost never a cause.

**Action item.** A change with an owner, a due date, and a stated preventive effect,
which names the class of incident it addresses.

**Detection time / diagnosis time / mitigation time.** Three separate durations, and
you must measure them separately, because they have different fixes. Detection is an
alerting problem. Diagnosis is a runbook and observability problem. Mitigation is an
architecture and tooling problem. Teams that report only "MTTR" cannot tell which
one to invest in.

### The control hierarchy — the thing that determines action-item strength

Weakest to strongest:

1. **Documentation.** "We wrote it up." Prevents nothing on its own.
2. **Training / awareness.** Decays with staff turnover.
3. **Checklist item in review.** Depends on a human noticing, every time.
4. **Alert.** Does not prevent; shortens. Still valuable, and often the cheapest
   real improvement.
5. **Automated test / CI gate.** Prevents the class from reaching production, if the
   test can express the class.
6. **Safe-by-default API or wrapper.** Makes the bug hard to write.
7. **Structural elimination.** Makes the bug *impossible* — the dangerous capability
   no longer exists in the codebase.

The Principal move is to ask, for each contributing factor, "what is the highest
level achievable here at acceptable cost?" and to write that. Most postmortems stop
at level 1 or 3 because those are the cheapest to write and the most invisible to
fail.

---

## Why does it matter?

### 1. Because "someone forgot" is a prediction that the next person will also forget

If your finding is that an engineer failed to bound a queue, your implicit model is
that the system is fine and the human was defective. That model produces exactly one
action item — be more careful — and it has a known failure rate: everybody's.

The alternative model is that a system which permits an unbounded queue by default,
with no saturation alert and no overload scenario in the load suite, *will* produce
this incident eventually regardless of who is at the keyboard. That model produces
three action items, each of which is checkable.

### 2. Because the postmortem is where an incident is converted into an asset or wasted

You paid for the incident. The price is already sunk. The only variable is whether
you get anything for it. A postmortem that produces two strong action items has
bought a permanent reduction in a class of risk. One that produces a wiki page has
bought a wiki page.

### 3. Because it is the primary mechanism by which engineering organisations learn

There are very few of these mechanisms. Design review catches problems before they
exist. Code review catches them at the line level. The postmortem is the only one
that operates on evidence from production reality. Weakening it — by rushing it, by
making it blameful, by letting action items rot — removes the organisation's main
feedback loop from the world.

### 4. Because blame makes the next incident longer

This is the practical argument, and it is more persuasive to sceptical managers than
the cultural one. In a blaming culture, the engineer who notices the anomaly at
02:40 waits twenty minutes before speaking, to be sure. That is twenty minutes of
detection time, purchased with fear, on every incident, forever. Blamelessness is
not kindness; it is latency reduction.

### 5. Because how you write the action items is visible evidence of your seniority

Give a strong Senior engineer an incident and they will fix it correctly. Give a
Principal engineer the same incident and the fix is the same, but the four items
around it — the alert, the default that changes, the load-test scenario, the runbook
confirmation step — are what stop the next three incidents you would otherwise
have had. Interviewers know this, which is why "walk me through a postmortem you
wrote" is a standard Principal question, and why the follow-up is always about the
action items.

---

## The decision, framed

The decision this topic trains is **where to stop asking why, and what to write
down when you stop.**

That decision is under real ambiguity, because there is no natural stopping point.
Every incident's causal chain runs backwards indefinitely: the queue was unbounded
because the factory default is unbounded, because the API was designed in 2004,
because the JDK's authors optimised for the common case, because... At some point
you stop. The question is what governs the stopping.

### The stopping rule

**Stop when the next "why" produces no action item you could take.**

Applied to the unbounded-queue example:

- *Why did the service OOM?* The work queue grew without bound. → **Actionable**: it
  could be bounded.
- *Why was it unbounded?* The executor was created with a factory whose default
  queue is unbounded. → **Actionable**: the factory can be replaced with one that
  requires an explicit bound.
- *Why did nobody notice?* There was no queue-depth metric and no saturation alert.
  → **Actionable**: add both.
- *Why did testing not find it?* The load suite has no overload scenario; it tests
  the happy arrival rate. → **Actionable**: add one.
- *Why does the JDK default to unbounded?* Historical API design. → **Not
  actionable.** Stop.

Four contributing factors, four action items, and a clean stopping point that is a
property of the analysis rather than of anyone's patience.

### The three framings that change the output

**Framing 1: "what went wrong" versus "what allowed this".** The first produces a
story. The second produces a list of conditions. Only the second is a design input.

**Framing 2: counterfactual, applied honestly.** For each candidate factor, ask: *if
this one condition had been different, would the incident have happened?* If the
answer is "yes, just later" it is not a contributing factor, it is context. If it is
"no", or "yes but half as bad", it is a factor. This is the test that keeps the list
honest and stops it becoming a general critique of the codebase.

**Framing 3: the class, not the instance.** For each factor, ask "what else in this
system has the same shape?" The unbounded queue is one instance of *unbounded
buffering*, which also lives in the reactive path (Topic 105) and in the outbox
relay's batch. Naming the class is what turns one fix into three.

### What makes this hard in a real meeting

Three forces push against you, all of them social:

- **The desire for closure.** A single cause ends the meeting. Four factors mean
  four owners, and people are busy.
- **The presence of the person who wrote the line of code.** Their instinct is to
  accept blame, because accepting it is faster than the alternative and it makes the
  room comfortable. Letting them do it is the easiest mistake in the room and it
  costs you the analysis.
- **The manager who wants "the" cause for a status update.** This one is legitimate
  and you should serve it: give them a one-line primary factor for the summary, and
  keep the list underneath. Refusing the summary line is a fight you do not need.

---

## Example 1 — a minimal illustration

The smallest version, so the mechanic is visible before the real one.

### The incident

A scheduled job that emails order receipts stopped sending. Discovered when a
customer complained. The job's thread had died on an unhandled exception three days
earlier; the scheduler did not restart it.

### Version A — the postmortem most teams write

> **Root cause:** an unhandled `NullPointerException` in the receipt formatter
> caused the scheduled task's thread to die.
> **Fix:** added a null check.
> **Action items:** be careful with nullable fields.

Everything in it is true. It prevents approximately nothing. Next quarter a
different unhandled exception in a different scheduled task will do the same thing,
and the postmortem for that one will say "root cause: an unhandled
`IllegalStateException`".

### Version B — contributing factors and classes

> **Trigger.** An order with no billing address reached the formatter for the first
> time.
>
> **Contributing factors.**
> - **CF1.** The formatter dereferenced a nullable field without handling absence.
>   *Class:* nullable-field dereference on a rarely-populated column.
> - **CF2.** `ScheduledExecutorService` silently cancels a task when its `Runnable`
>   throws — a documented behaviour, and a surprising one. Nothing in the codebase
>   wrapped scheduled tasks to catch and log. *Class:* any scheduled task dying
>   silently on any exception.
> - **CF3.** There was no metric for "receipts sent in the last hour" and no alert on
>   it going to zero. *Class:* any periodic job whose absence is invisible.
> - **CF4.** Detection came from a customer after three days. *Class:* our detection
>   for asynchronous work is customer-driven.
>
> **Action items.**
> | # | Action | Prevents (class) | Owner | Due | Strength |
> |---|---|---|---|---|---|
> | A1 | Null-safe formatter with a test for the absent-address case | This instance only | <name> | <date> | Test |
> | A2 | A `SafeScheduledTask` wrapper that catches, logs and re-schedules; make it the only sanctioned way to schedule | CF2 for every job | <name> | <date> | Safe-by-default wrapper |
> | A3 | Heartbeat metric per scheduled job + alert on absence | CF3 for every job | <name> | <date> | Alert |
> | A4 | NullAway on the formatter package | CF1 as a class | <name> | <date> | CI gate |

Note what changed. A1 is the fix everyone would write, and it is the *weakest* item
in the table. A2 is the one that prevents the most future incidents, and it exists
only because the analysis asked why a dead thread was silent. A3 converts a
three-day detection into minutes for every job, not just this one.

Note also the honest labelling of A1's preventive scope as "this instance only".
Saying so is what stops the table from looking stronger than it is.

---

## Example 2 — the real postmortem on the project spine

The worked artefact. The incident is the **Topic 101 drill**: a `synchronized` block
held across the payment-gateway call, under virtual threads, producing carrier
starvation.

I am treating a drill as a real incident. That is the exercise. **Every timestamp
and number below is a blank for you to fill from your own drill run.** I do not know
what your numbers were and I will not invent them.

### PM header

```markdown
# Postmortem — orderflow: throughput collapse with idle CPU

- Incident ID:
- Date of incident:
- Author:
- Reviewers:
- Status: [ ] Draft  [ ] Reviewed  [ ] Actions assigned  [ ] Closed
- Severity (and the rule that assigned it):
- Customer impact (what a user experienced, in their words):
- Duration: detection ___  diagnosis ___  mitigation ___  total ___
- Error budget consumed (Topic 130), if you have a budget:
```

### Summary — five sentences, no jargon

Write it so a product manager understands it. A worked shape:

> *During the load run on <date>, order placement slowed to the point of timing out
> for approximately <duration>. Customers attempting to place orders saw <what>. The
> service was running, using little CPU, and accepting connections — it simply was
> not making progress. The cause was a lock held across a slow external call, which
> under our new threading model consumed the whole scheduler. We recovered by
> <action>; the underlying change is <action item>.*

**Strong summary:** names customer experience first, admits the confusing part (low
CPU), and does not use the words "pinning" or "carrier" before they are explained.

**Weak summary:** "Virtual thread pinning caused carrier starvation." True, and
unreadable to two thirds of the audience, and it presents the mechanism as if it
were the finding.

### Severity, and the rule that assigned it

Severity labels are worth very little on their own. "Sev 2" means whatever your
organisation's culture currently means by it, and that drifts. What has content is
the *rule*, written next to the label.

Two rules that carry real information, both of which you can compute:

- **Budget-based (Topic 130).** "This incident consumed <blank>% of the 28-day error
  budget for the order-placement SLO." That is a number, it is comparable across
  incidents, and it makes the prioritisation argument for your action items without
  you having to make it.
- **Journey-based.** "Order placement was unavailable; browsing and order reads were
  unaffected." A user-facing statement, understandable by anyone, and it prevents
  the common inflation where any production anomaly becomes a major incident.

```markdown
| Field | Value |
|---|---|
| Severity label |  |
| Rule that assigned it |  |
| Journeys affected |  |
| Journeys unaffected |  |
| Error budget consumed (if you have one) |  |
| Customers who noticed, and how we know |  |
```

**Strong:** the "how we know" cell says something checkable — support tickets, a
drop in a business metric, a client-side error rate. **Weak:** "minimal customer
impact", which is an assertion that costs nothing to make and is usually made by the
person least able to observe it.

### Timeline — blank, fill from your drill notes

Three columns of entry type: system, human action, human belief. The belief column
is the one that teaches.

```markdown
| Time | Source (evidence) | What happened / what we believed |
|---|---|---|
| T-___ | deploy log | Change deployed: `spring.threads.virtual.enabled=true` |
| T+0 | | Load run begins at baseline arrival rate |
| T+___ | metric: | First symptom appears: |
| T+___ | alert: | Alert fires (or: no alert fired — record that) |
| T+___ | human | First responder acknowledges. Believed at that moment: |
| T+___ | human | Checked CPU. Belief updated to: |
| T+___ | human | Ruled out: |
| T+___ | evidence: | Diagnostic captured (`jcmd Thread.print` / JFR / `jdk.tracePinnedThreads`) |
| T+___ | human | Correct diagnosis reached |
| T+___ | human | Mitigation applied: |
| T+___ | metric: | Recovery confirmed by: |
| T+___ | | Incident closed |
```

**Rules for the timeline, and I check all four:**

1. Every row cites evidence. "We think around then" is a row that says
   `EVIDENCE: none — reconstructed from memory`, which is honest and also a finding
   about your logging.
2. The belief column is filled for every human row. *"We believed it was the
   database, because the last three incidents were the database"* is worth more than
   any technical row in the table, because it explains the diagnosis time and points
   at a real action item.
3. Detection, diagnosis and mitigation are marked as separate spans.
4. Dead ends stay in. The forty minutes spent on the wrong hypothesis is the most
   valuable data in the document; deleting it makes the team look competent and
   makes the postmortem worthless.

### Contributing factors — the core section

Each factor gets: the condition, the evidence, the counterfactual test, the class,
and whether it is technical, procedural or organisational.

**CF1 — A lock was held across a network call.**
*Condition:* the inventory-and-payment path took a monitor and then made the
payment-gateway call inside it.
*Counterfactual:* had the call been outside the lock, no carrier would have been
occupied while blocked; the incident does not happen.
*Class:* **any blocking operation performed inside a monitor.** Not just this call
site, and not just this gateway.
*Type:* technical.

**CF2 — The threading model changed and nothing re-examined the blocking paths.**
*Condition:* enabling virtual threads changed the consequence of an existing,
previously-harmless pattern. The `synchronized` block was not new. Its cost was.
*Counterfactual:* had the switch been accompanied by an audit of monitors around
blocking calls, the site is found before it matters.
*Class:* **any configuration change that alters the cost of an existing pattern.**
This class is much broader than virtual threads and is the most valuable finding in
the document.
*Type:* procedural.

**CF3 — The scheduler's parallelism is small and invisible.**
*Condition:* the carrier pool is sized from core count by default. A handful of
pinned threads exhausts it. Nothing in the system displayed carrier availability.
*Counterfactual:* with a pinned-event signal, detection is minutes rather than
however long yours took.
*Class:* **saturation of a resource we do not measure.** Same class as the pool
saturation of Topic 109 and the queue depth of Topic 90.
*Type:* technical/observability.

**CF4 — The symptom pointed away from the cause.**
*Condition:* low CPU plus rising latency reads as "downstream is slow" to every
engineer's trained intuition, and that intuition was correct on the previous
incidents. There was no runbook entry mapping this symptom to this cause.
*Counterfactual:* with a runbook entry, diagnosis time collapses.
*Class:* **symptoms with multiple candidate causes and no distinguishing step.**
Requests-hang and latency-up-CPU-idle are both in this class.
*Type:* procedural.

**CF5 — The load suite did not cover a slow downstream.**
*Condition:* load tests ran against a fast payment stub. The pathological behaviour
requires the gateway to be slow, so no pre-production run could have produced it.
*Counterfactual:* a run with an injected downstream delay finds it before release.
*Class:* **failure modes that require a degraded dependency to appear.** Includes
the retry storm of Topic 111 and the backpressure failure of Topic 105.
*Type:* procedural.

Optionally, if it is true of your run:

**CF6 — The first mitigation destroyed the evidence.**
*Condition:* restarting the service cleared the symptom. Had a diagnostic not been
captured first, the incident would have recurred with nothing learned.
*Class:* **mitigations that clear state needed for diagnosis.** Same class as
restarting on a deadlock or a pool exhaustion.
*Type:* procedural.

**Note what is not on this list:** the name of whoever wrote the `synchronized`
block, and the name of whoever enabled virtual threads. Both people acted
reasonably. The monitor was correct when written. The config change was a good
change with a measured benefit. The incident is what happened when two reasonable
decisions met, with no mechanism between them.

### The counterfactual test, applied

I will ask you to run this on your own list. For each factor:

| Factor | If this alone had been different, does the incident happen? | Verdict |
|---|---|---|
| CF1 | No — the carrier is never held | Contributing factor |
| CF2 | No — found before release | Contributing factor |
| CF3 | Yes, but detected in minutes | Contributing factor (severity/duration) |
| CF4 | Yes, but diagnosed far faster | Contributing factor (duration) |
| CF5 | No — found in test | Contributing factor |
| "The payment gateway was slow" | Yes — gateways are slow sometimes | **Context, not a factor** |

That last row matters. A slow dependency is a normal condition of the world, not a
defect. Listing it as a cause outsources your incident to someone else and produces
no action item you own.

### Action items — the chain

Every row has an owner (a person, never a team), a due date, the class it prevents,
and its strength on the control hierarchy.

```markdown
| # | Action | Addresses | Prevents (class) | Strength | Owner | Due | Verified how |
|---|---|---|---|---|---|---|---|
| A1 | Move the gateway call outside the monitor; replace with `ReentrantLock` where mutual exclusion is still required | CF1 | This site only | Fix | | | Re-run of the drill shows no pinning |
| A2 | Detect monitors around blocking calls: an ErrorProne/ArchUnit check on `synchronized` blocks containing an I/O call, new code first with a baseline | CF1 | Every future occurrence of the class | CI gate | | | Check fails on a deliberately reintroduced case |
| A3 | JFR `VirtualThreadPinned` event exported as a metric, with an alert on sustained pinning | CF3 | Any pinning, any site | Alert | | | Alert fires during a re-run of the drill |
| A4 | Carrier-scheduler saturation on the USE dashboard alongside Hikari and executor queues | CF3 | Saturation of an unmeasured resource | Alert/dashboard | | | Signal moves under the drill |
| A5 | Add a slow-downstream scenario to the load suite: gateway latency injected at Nx normal | CF5 | Every failure requiring a degraded dependency | Automated test | | | Suite reproduces the original symptom before A1 |
| A6 | Runbook entry: "latency up, CPU idle" — with the three candidates and the distinguishing command for each | CF4 | Diagnosis time on a whole symptom class | Runbook | | | A colleague who did not work the incident reaches the diagnosis using only the runbook |
| A7 | Add "audit blocking paths under monitors" to the threading-model change checklist; more strongly, make A2 the gate so the checklist is unnecessary | CF2 | Config changes that alter the cost of existing patterns | Checklist → superseded by gate | | | A2 shipped |
| A8 | Capture-before-mitigate: a one-command diagnostic bundle (thread dump + JFR snapshot + pool stats) in the runbook preamble | CF6 | Evidence destruction on any restart-clears-it incident | Runbook + tooling | | | Bundle runs in under 30s on a healthy pod |
```

**Read the chain, not the rows.** A1 alone is the fix a good engineer produces in an
hour. A2 is what makes the class unwritable. A3 and A4 shorten every future
occurrence. A5 moves discovery before production. A6 shortens diagnosis for a whole
symptom family. A8 protects the next investigation. Only A1 is about this incident.

**The verification column is not decoration.** An action item you cannot verify is
one that will be marked "done" when a pull request merges, whether or not it works.
"Alert fires during a re-run of the drill" is a completion criterion; "add alert" is
a wish.

### What we got lucky about

A section most templates omit and every good postmortem has.

> *Recovery was fast because someone recognised the low-CPU signature from the
> pre-production run. Had that person been unavailable, diagnosis would have taken
> substantially longer, since no runbook entry existed. **The speed of this recovery
> was a property of who was on call, not of the system.** A6 addresses that.*

Naming luck is important because it stops a fast recovery from being read as
evidence that the system is fine. Fast recoveries that depend on one person are a
risk, not a strength — and this is exactly the organisational risk that Topic 124's
review asks you to name.

### Where the register was, and was not

Cross-check against Topic 124's failure-mode register:

- Was this failure mode already in the register (F14)? If yes: it was known and
  undetected, and the question is why the detection item was not prioritised. That is
  a *better* finding than a novel failure, and it belongs in the postmortem.
- If it was not in the register: how would a drill have found it, and does that
  suggest a class of drill you are not running?

---

## Wrong approach → exact symptom → root cause → fix

Five. Organisational symptoms, each recognisable in a document you are holding.

---

### Wrong approach 1 — the root cause is "the developer forgot to bound the queue"

**Exact symptom.** The postmortem is one page. It has a root-cause line naming a
human action, a fix that is a single pull request, and an action item that reads
"be careful with executor configuration". The meeting is short and everyone leaves
satisfied. Five months later a different service OOMs under overload, and its
postmortem's root-cause line names a different engineer and the same mistake. Nobody
connects the two, because neither document names a class. The engineer from the
first incident has become quietly more cautious and slightly less willing to speak
up early, which costs detection time on unrelated incidents.

**Root cause.** Human error is the most legible node in any causal chain and the
easiest place to stop. It also produces a *complete-feeling* explanation: the story
has an agent, a decision, and a consequence. System conditions do not tell a story;
they are a list. Under time pressure, the story wins. There is also a quieter
incentive: a human cause implies no engineering work, and the meeting ends on time.

**Fix.** Apply the substitution test out loud in the meeting: *"if we replaced this
engineer with any other engineer on the team, would the outcome be different?"* If
no — and it almost always is no — the finding is about the system. Then run the
five-why chain past the human node using the stopping rule, and count the action
items. One item means you stopped at the human. The unbounded-queue example produces
four: the executor factory default, the missing saturation alert, the missing
overload scenario in the load suite, and the review that had no way to catch it. Then
write "the next person will also forget" into the document explicitly as the reason
the fix is a wrapper rather than a warning.

---

### Wrong approach 2 — action items with no owner and no due date

**Exact symptom.** The postmortem is genuinely good: five contributing factors, six
strong action items, class-level thinking throughout. It is circulated and praised.
The action items are recorded as a bullet list at the bottom of the document, owned
by "the platform team", due "next quarter". Ninety days later, one of the six has
been done — the code fix, which was done during the incident. At the next incident of
the same class, someone finds the earlier postmortem and reads out the action item
that would have prevented it. The room goes quiet. The credibility of every future
postmortem drops, including the ones whose items *are* being tracked.

**Root cause.** Assigning an owner requires someone in the room to accept work, and
naming a date requires someone to defend a priority against their existing
commitments. Both are uncomfortable, both are avoidable by writing a team name and a
vague quarter, and neither the author nor the meeting has authority over anyone's
backlog. The document is written by the incident process and executed by the
planning process, and nothing connects them.

**Fix.** Three mechanical rules, all cheap. **One:** an owner is a person, never a
team; if no person will take it in the meeting, the item is recorded as "unowned —
requires prioritisation by <manager name>", which is honest and escalates itself.
**Two:** the items go into the same tracker as feature work, in the same sprint
process, with the postmortem linked — a list at the bottom of a document is not a
commitment system. **Three:** review open action items at the *start* of the next
postmortem meeting, every time. That single habit does more for completion than any
amount of process, because it makes non-completion visible in the room where it is
most expensive. Additionally: cut the number. Six items with three owners will
outperform twelve items with one, and you should be willing to drop the weakest four
in the meeting to make the top two real.

---

### Wrong approach 3 — the timeline is written from memory and the dead ends are edited out

**Exact symptom.** The timeline reads cleanly: alert at 02:14, diagnosis at 02:31,
mitigation at 02:40. Total 26 minutes, which sounds good. What actually happened is
that the first responder spent 35 minutes on the database because the last three
incidents were the database, and the timeline starts at the moment they gave up on
that hypothesis. Six months later a similar incident takes the same 35 minutes on the
same wrong hypothesis, because the document that would have prevented it was edited
to look competent. Worse: nobody ever built the runbook entry that maps the symptom
to its three candidates, because the document contained no evidence that anyone had
struggled.

**Root cause.** The postmortem has two audiences with opposite needs. Management
reads it as a report card; engineers need it as a learning document. When the author
believes the first audience dominates, dead ends become liabilities and get trimmed.
This is a *culture* symptom presenting as a document defect, and it is usually
correct behaviour given the incentives — which makes it a systems problem rather than
an author problem.

**Fix.** Make the belief column structural: every human row in the timeline records
what the responder believed at that moment and why. That reframes a dead end as data
about the system's legibility rather than as a personal failing — "we believed the
database because the symptom is indistinguishable without a thread dump" is a
statement about missing tooling. Then measure detection, diagnosis and mitigation
separately and report all three, so that a long diagnosis is visibly an
observability problem rather than a competence problem. And build the timeline from
evidence during the incident, not after: one person in the incident channel posting
timestamped observations, which is a role, not a discipline. Where evidence is
missing, write `EVIDENCE: none — reconstructed`, which is itself a finding about
logging.

---

### Wrong approach 4 — the postmortem produces documentation where a wrapper was available

**Exact symptom.** Every action item is of the form "document X", "add a review
checklist item for Y", "share a learning session on Z". The document is thorough and
its items are all level 1 to 3 on the control hierarchy. They are completed on time,
which looks like success. Eighteen months later the same class recurs; the checklist
item exists and was not applied, because the reviewer was junior and did not know
what they were looking at, and because a checklist of forty items is a checklist
nobody reads.

**Root cause.** Writing documentation is fast, uncontroversial, and needs no
engineering time approval. Building a wrapper or a CI gate is a week of work that
must be argued for against feature delivery. Under the incentive to close the
postmortem quickly, the author picks the item they can complete rather than the item
that works. Underneath that is a knowledge gap: many engineers do not have the
control hierarchy in their head, so "add a checklist item" genuinely feels like the
available option.

**Fix.** For every contributing factor, write the hierarchy level of the proposed
item next to it, and ask explicitly what a level-6 item would look like. Often it is
smaller than it sounds: for the unbounded queue, a `BoundedExecutors` factory whose
methods require a capacity argument is an afternoon, and it removes the class
permanently for all future code. Where the strong item genuinely is expensive, write
*both* — the cheap control now and the strong one as a dated item with its cost
estimated — so the trade is visible instead of silently resolved in favour of the
cheap one. Topic 132 is where you learn which controls are available in Java
specifically; this is where you have to choose one under time pressure.

---

### Wrong approach 5 — the postmortem is used to decide whether the on-call engineer performed well

**Exact symptom.** Somewhere in the meeting, someone asks why it took 35 minutes to
check the thread dump. The tone is reasonable and the question is even fair. The
engineer explains, slightly defensively. The remaining twenty minutes are about
response quality rather than about the system. The document that comes out has one
action item about "improving on-call readiness". Three months later, at the next
incident, the first responder does not post their early hypothesis in the channel,
because last time a hypothesis became a topic of review. Detection and diagnosis both
get slower, permanently, and nobody attributes the slowdown to the meeting that
caused it.

**Root cause.** Blamelessness is usually understood as a rule about *tone*, so people
police words like "fault" while asking questions whose entire function is
performance evaluation. The failure is structural: the same meeting is being used for
two incompatible purposes — learning about a system, and assessing a person. The
second purpose contaminates the first irreversibly, because people cannot supply
candid information to a process that evaluates them.

**Fix.** State the separation explicitly at the top of the document and the start of
the meeting: *this document is not an input to anyone's performance review, and the
people in it acted reasonably given what they knew.* Then convert every
response-quality question into a system question, in the room, out loud: "why did it
take 35 minutes" becomes "what would have made the thread dump the obvious first
step" — which produces A6, the runbook entry, instead of producing defensiveness. If
someone genuinely handled an incident badly, that is a management conversation held
somewhere else, by their manager, and it must not happen here. Protecting that
boundary is one of the highest-value things a Principal engineer does in an
organisation, and it is nearly invisible when it works.

---

## Artefact — what you must produce

**One postmortem** for one Phase 8 or Phase 9 drill, treated as a real incident.
`docs/java/postmortems/<date>-<slug>.md`, committed.

**Choose one:**

- **Topic 79 — the unbounded static cache OOM.** Best if you want the clearest
  example of a contributing factor that is a library default.
- **Topic 94 — the wallet/inventory lock-ordering deadlock.** Best if you want to
  exercise the "symptom with multiple candidate causes" analysis, because it collides
  with Topic 109's pool deadlock.
- **Topic 101 — virtual-thread pinning and carrier starvation.** Worked above; pick
  it only if you will do the analysis independently rather than copying mine.

### Format and length

- **3 to 6 pages.** Under 3 and the contributing-factor section is thin. Over 6 and
  you have written a narrative; the analysis is what matters, not the story.
- Written so that the **Summary**, the **Contributing factors**, and the **Action
  items** can be read alone and still be accurate.
- Non-engineers must be able to read the summary. Jargon is defined at first use or
  removed.

### Required sections

1. **Header** — severity with the rule that assigned it, customer impact in user
   language, and detection / diagnosis / mitigation / total as four separate
   durations.
2. **Summary** — five sentences, no undefined jargon.
3. **Timeline** — evidence-cited, with a belief column on every human row, and the
   dead ends left in.
4. **Contributing factors** — **at least four**, each with condition, evidence,
   counterfactual, class, and type (technical / procedural / organisational). At
   least one must be procedural or organisational.
5. **What we got lucky about** — at least one entry, honestly.
6. **Action items** — the table, with owner (a person), due date, class prevented,
   control-hierarchy strength, and a verification criterion.
7. **Register cross-check** — was this in Topic 124's register? What changes there
   now?
8. **What we are explicitly not doing** — the items you considered and rejected,
   with the reason. This is as important as the items you took.

### Required content — the specific things I will check for

- The phrase "root cause" does not appear, or appears once as "primary contributing
  factor" for an external template with the full list underneath.
- **At least one action item at control-hierarchy level 5 or higher** (test, gate,
  wrapper, or elimination). If everything is documentation and alerts, you did not
  look for the strong control.
- At least one contributing factor that is a **default, a missing signal, or a
  missing test** rather than a code defect.
- At least one action item whose preventive scope is honestly labelled "this instance
  only" — usually the fix itself. Labelling it stops the table from overclaiming.
- A verification criterion for every action item that is checkable by someone other
  than the owner.
- No named individual anywhere in a causal position.
- The counterfactual table, including at least one candidate you rejected as
  **context rather than a factor**.
- A statement of the total time from detection to correct diagnosis, and one sentence
  about what would have shortened it.

---

## How I will review it

I will not comment on your prose. I will attack the analysis: the factor you stopped
at, the class you did not name, and the action item that will not survive contact
with a sprint planning meeting.

### The three questions that usually break a postmortem

**Question 1: "Which of these action items would have prevented the other three
incidents you have not had yet — and which one only fixes this one?"**

I want you to rank your own items by preventive scope, out loud, and to tell me which
of them is weak.

This breaks most postmortems because the items were written as a list of good things
to do rather than as a chain with different jobs. The author has usually never
compared them to each other, so the fix that closes this instance and the gate that
closes the class sit side by side with equal weight, and the sprint takes the cheap
one.

What a good answer sounds like: *"A2 — the check for blocking calls inside monitors —
is the only item that prevents the class, and it is the one I would fight for. A1 is
the fix and prevents nothing beyond this call site; I have labelled it as such. A3
and A4 do not prevent anything, they shorten — which is worth having, and I would
trade both for A2 if I could only have one. A5 is the highest-leverage of the cheap
items because it changes where we find this family of bug, not just this bug."*

Follow-ups:

- "You have eight items. Which four would you drop to get the top two done this
  sprint?" If the answer is "none, they are all important", the list will be
  delivered at whatever rate the backlog allows, which is usually zero.
- "A2 is a static-analysis check. What is its false-positive rate on your existing
  code, and what happens in week one?" This is Topic 132's whole lesson, and an
  action item that fails 400 existing cases gets switched off, taking your
  credibility with it.
- "Who has to approve the engineering time for A2, and have you asked them?"

**Question 2: "What is the contributing factor you decided not to write down?"**

Every real postmortem has one. Usually it is organisational: the team was
understaffed, the change was rushed for a date, one person owns the component, the
alert had been noisy so it was silenced, or someone raised this exact risk in a
review and was overruled.

The failure mode: an author who says there is not one. That either means they have
not looked, or means the culture makes it unwriteable — and if it is the second, that
is the most important thing in the incident and I will spend the rest of the review
there.

What a good answer sounds like: *"Two. First, the virtual-thread switch shipped in a
week where two of the three people who understand the concurrency paths were on
leave, and the review was thinner than it should have been. Second, the pinning risk
was mentioned in the change discussion and dismissed as theoretical because JDK 24
removed most `synchronized` pinning — nobody checked which JDK we run in production.
I have written the second one as CF7 and I have raised the first with my manager
rather than putting a staffing critique in a document that goes to the whole
organisation."*

Follow-ups:

- "How do you write a staffing contributing factor without it reading as a
  complaint?" There is a real technique: state it as a condition and pair it with the
  system change that would tolerate it — "the change process assumed reviewer
  availability that we cannot guarantee; a checklist gate does not depend on who is
  in the room".
- "Someone raised this and was overruled. What do you write?" You write it, in
  neutral form, as a factor about the *decision process* — because the alternative is
  that raising risks stops being worth doing.

**Question 3: "Six months from now, how will you know whether this postmortem
worked?"**

Not "were the items done". Done is an input.

The failure mode: an author who equates completed action items with success. All
eight could be shipped and the class could recur, and the document would still be
marked closed.

What a good answer sounds like: *"Three checks. One: did the gate in A2 catch a real
case before it shipped? I will check its violation log at the next quarterly review;
if it has never fired, either the class is rarer than I thought or the check is not
matching. Two: at the next incident with the 'latency up, CPU idle' symptom, did the
runbook get us to diagnosis faster? That is measurable from the timeline. Three: is
F14 still marked 'no detection' in the readiness register? If A3 shipped and the
register was not updated, our register is drifting from reality and that is its own
problem."*

Follow-ups:

- "Your gate never fired. Is that good news or bad news?" A genuinely ambiguous
  question, and a candidate who answers instantly in either direction is not
  thinking. The honest answer is that you cannot tell without checking whether the
  check matches the pattern you meant, which is a five-minute test.
- "The class recurred in a different service that does not use your wrapper. What
  does that tell you about the action item?" That it was scoped to a codebase rather
  than to the organisation — which is where Topic 134 begins.

### The other attacks, in order

**On the contributing factors:**

- "CF1 is a code defect. Which of your factors is about a default, and which is about
  a missing signal?" A list of four code defects is one factor written four ways.
- "You listed the slow gateway as a factor. Is a slow dependency a defect or a
  condition of the world?" If it is a condition, it produces no action item you own,
  and its presence in the list is a way of not looking at your own system.
- "What else in this codebase has the same shape as CF1?" If the answer is "nothing",
  you have not looked; if it is a list, that list should have generated an action
  item.

**On the timeline:**

- "Show me the row where somebody was wrong." A timeline with no wrong hypothesis was
  edited.
- "Detection was fast. Was that the alert, or was it someone happening to look?"
- "What destroyed evidence during mitigation, and what would you capture first next
  time?"

**On the action items:**

- "This one is owned by a team. Which person?"
- "This one is due 'next quarter'. What is the date, and what does it displace?"
- "How will the owner know they are done?" If the criterion is "merged", it is not a
  criterion.
- "Which of these is at hierarchy level 6, and if none, why was none available?"

**On the register cross-check:**

- "This failure mode was already F14 in your readiness review, marked as having no
  detection. Why was that not prioritised, and what changes about how you rank
  detection gaps?" This is the strongest available finding in a postmortem and most
  authors skip it, because it implicates a prior decision they made.
- "Your register now gains a row. What else should it gain that you have not
  drilled?" One incident usually implies a family, and the family is the useful
  output.

**On the summary:**

- "Read me the customer impact sentence." If it contains the word "pinning", it was
  written for engineers and the product side will not read the document.

### What I will not attack

- The severity you assigned, if the rule that assigned it is stated.
- A short list of action items. Four real ones beat twelve aspirational ones and I
  would rather review the four.
- An honest "we do not know" in the timeline, where evidence was not captured. That
  is a finding, and it usually produces a good action item.
- Contributing factors you found and could not fix, provided they are recorded with
  the reason.

---

## Interview questions (Senior → Principal)

### Q1 — "Walk me through a postmortem you wrote."

**A Senior answer.** Tells the incident story: what broke, how it was found, what the
fix was. It is a good story, accurately told, and it is mostly narrative. The action
items appear at the end as a list.

**A Principal answer.** Spends thirty seconds on the story and the rest on the
analysis. *"The trigger was a config change. The interesting part is that the config
change was correct — it made the system faster — and it changed the cost of a pattern
that had been safe for two years. That is the class I care about: changes that alter
the consequences of existing code rather than introducing new code. Four contributing
factors; the one I would defend hardest is that we had no signal for the resource
that saturated. The action item chain has one fix, one gate, two detections and one
test-coverage change, and only the gate prevents recurrence — the rest shorten."*

**What separates them.** The Senior answer is about the incident. The Principal
answer is about the *class* and about which action item does which job. Notice also
that the Principal answer volunteers the weakest item, which reads as confidence
rather than as a concession.

**Adversarial follow-up.** *"You said the config change was correct. Should you have
rolled it back?"* The strong answer distinguishes mitigation from cause: rolling back
would have restored service and would also have discarded a measured improvement to
solve a problem that lived in one call site. The right sequence is to mitigate by the
cheapest reversible means available, then fix the site, then re-apply. If the
candidate says "yes, always roll back", ask what they do when the change is a
security fix.

---

### Q2 — "What's the difference between a root cause and a contributing factor?"

**A Senior answer.** "Root cause is the fundamental reason; contributing factors made
it worse." Directionally fine, and it treats root cause as a real thing that exists
in the world and can be found.

**A Principal answer.** *"'Root cause' names the place we chose to stop asking why,
and the phrase hides that it was a choice. Real incidents need several conditions to
all hold. If I pick one and call it the root, I get one action item; if I list the
conditions, I get four, and the four are usually cheaper in aggregate than the one.
The stopping rule I use is: keep asking why until the next answer produces no action
I could take. And the reason this matters practically is that the gravitational pull
of root-cause thinking is toward the last human decision in the chain, which is the
one node in the whole system that cannot be engineered."*

**What separates them.** The Senior answer treats root cause as discoverable. The
Principal answer treats it as an *editorial decision with consequences*, and has an
explicit rule for making it.

**Adversarial follow-up.** *"Your VP wants one line for the exec summary. What do you
write?"* The answer that fails is refusing on principle. The answer that works gives
the line — "we held a lock across a slow external call, which under our new threading
model stalled the whole service" — and puts the four factors immediately underneath.
Serving the summary need costs nothing and buys you the space for the analysis. A
candidate who fights this is optimising for being right over being effective.

---

### Q3 — "How do you keep a postmortem blameless when someone genuinely made a mistake?"

**A Senior answer.** "We focus on the system, not the person, and avoid naming
individuals."

**A Principal answer.** *"I separate the causal account from the personnel question
completely, and I say so at the top of the document. In the causal account, an
individual's action is evidence about the system: if the mistake was easy to make,
the finding is that it was easy to make, and the action item changes that. The test I
use out loud in the meeting is the substitution test — if any other engineer had been
in that seat, would the outcome differ? Almost always no, and the room hears why the
analysis moves on. If the answer were genuinely yes — someone bypassed a control
knowingly — that is a management conversation held elsewhere by their manager, and it
must not happen in this meeting, because the moment this meeting can affect someone's
review, nobody will tell it the truth again."*

**What separates them.** The Senior answer describes a norm. The Principal answer
supplies the mechanism (the substitution test), states the boundary condition
(genuine misconduct), and explains the *cost* of getting it wrong in terms the
organisation understands — future detection latency.

**Adversarial follow-up.** *"The same person has now caused three incidents. Still
blameless?"* Yes, in the document, every time. And separately: three incidents from
one person is a signal about onboarding, about the review process, or about someone
being placed in a role they were not set up for. That is their manager's work, and it
is real work that should happen. The two things are not in tension; they are in
different rooms.

---

### Q4 — "Your action items never get done. What do you change?"

**A Senior answer.** Better tracking, more follow-up, escalate to management.

**A Principal answer.** *"First I check whether the problem is completion or volume.
Twelve items with one owner is a planning failure disguised as a discipline failure —
I would cut to three and make them real. Then three mechanisms. One: owners are
people and dates are dates, in the meeting, or the item is recorded as unowned and
needing prioritisation by a named manager, which escalates itself without me having
to escalate. Two: the items live in the same backlog as feature work, because a list
at the bottom of a document is not a commitment system. Three: every postmortem
meeting opens by reviewing the open items from previous ones — that single habit does
more than any process, because non-completion becomes visible in the room where it is
most expensive. And if items still do not land, the honest finding is that the
organisation has decided this class of risk is acceptable, and I would rather that
decision be explicit than pretend the tracker is the problem."*

**What separates them.** The Senior answer applies more effort to the same
mechanism. The Principal answer changes the mechanism, and is willing to name the
possibility that non-completion is a real priority decision rather than a failure of
diligence.

**Adversarial follow-up.** *"You cut twelve items to three. What happens when the
class you dropped causes an incident?"* The strong answer does not pretend this is
avoidable: you wrote the dropped items down in the "explicitly not doing" section
with the reason, so the decision is recoverable and the next postmortem references
it. That section exists precisely for this.

---

### Q5 — "You are new and the team has never written a postmortem. Where do you
start?"

**A Senior answer.** Introduce the template and run the process at the next incident.

**A Principal answer.** *"Not with a template. With one postmortem, written by me,
for an incident that has already happened, showing rather than proposing. I would
pick one where the honest analysis produces an action item that visibly helps the
on-call engineer — a runbook entry or a missing alert — so the first thing the team
sees is the process paying them rather than costing them. I would run the meeting
without calling it a postmortem, keep it under forty minutes, and produce three
items with real owners. Then I would do the second one with someone else writing and
me reviewing. Templates come fourth, if at all. And I would ask the manager for one
thing in advance: an explicit statement that this is not an input to performance
review — asked for privately, before the first meeting, because if it is not true I
need to know before I invite people to be candid."*

**What separates them.** Sequencing and adoption cost. The Senior answer introduces
a process; the Principal answer makes adoption cheaper than non-adoption by
demonstrating value first — which is Topic 134 appearing inside a technical answer —
and secures the cultural precondition before relying on it.

**Adversarial follow-up.** *"The manager says they cannot promise that."* Then you
have learned the most important fact about the environment, and you adapt: write
postmortems that are strictly about system conditions and contain no human actions at
all. Weaker, and still worth doing. Say that out loud rather than pretending the
process will work as designed.

---

## Mental model checkpoint

Answer without looking. More than two sentences means you are reconstructing.

1. Why does "root cause: the developer forgot to bound the queue" prevent nothing,
   and what four action items does the same incident produce under a contributing-
   factor analysis?

2. State the stopping rule for the why-chain, and apply it to the JDK's unbounded
   default queue. Where exactly do you stop and why?

3. Name the seven levels of the control hierarchy from weakest to strongest, and say
   which level "add a code-review checklist item" occupies and why that matters.

4. What is the counterfactual test, and use it to explain why "the payment gateway
   was slow" is context rather than a contributing factor.

5. Your timeline shows 26 minutes from alert to recovery, and no wrong hypotheses.
   What are you looking at, and what is the cost of leaving it that way?

6. What is the substitution test, when do you say it out loud, and what is the one
   situation where the answer is genuinely "yes, the outcome would differ"?

7. Six months after a postmortem, all eight action items are marked done. Name three
   checks that tell you whether the postmortem actually worked.

---

## Quick reference card

### The mechanic, in one line

> The output is a chain of action items that each prevent a class. "Someone forgot"
> predicts that the next person will also forget.

### The stopping rule

> Keep asking why until the next answer produces no action you could take. Then stop,
> and write down where you stopped.

### Control hierarchy — weakest to strongest

1. Documentation → 2. Training → 3. Review checklist → 4. Alert →
5. Test / CI gate → 6. Safe-by-default wrapper → 7. Structural elimination

Ask for every factor: *what is the highest level achievable at acceptable cost?*

### Action item — required fields

person owner · date · class prevented · hierarchy level · verification criterion

### Three durations, measured separately

- **Detection** — alerting problem.
- **Diagnosis** — runbook and observability problem.
- **Mitigation** — architecture and tooling problem.

Reporting only MTTR hides which one to invest in.

### The tests

- **Substitution test** — would another engineer have done differently? Usually no →
  the finding is about the system.
- **Counterfactual test** — if only this condition were different, does the incident
  happen? Yes-just-later → context, not a factor.
- **Class test** — what else in the system has this shape?

### Timeline rules

Evidence-cited rows · a belief column on every human row · dead ends left in ·
`EVIDENCE: none — reconstructed` where memory was the source.

### Sections most templates omit and you must not

- What we got lucky about.
- What we are explicitly **not** doing, with reasons.
- Register cross-check against Topic 124.

### Anti-patterns, one line each

- A human action as the root cause.
- Action items owned by a team.
- A timeline with no wrong hypothesis.
- Every item at hierarchy level 1–3.
- A dependency's slowness listed as a cause.
- The meeting used to assess the responder.
- Twelve items, none prioritised.

---

## When would I use this at work?

**1. On a near miss, which is the cheapest postmortem you will ever write.** Something
almost broke: the alert fired, someone caught it, no customer noticed. Most teams
celebrate and move on. A forty-minute analysis on a near miss produces the same
class-level findings as a real incident, with none of the emotional load, no
executive audience, and nobody defensive. It is also the easiest place to introduce
the practice to a team that has never done it, because there is nothing at stake.

**2. Right after any incident where the mitigation was a restart.** A restart clears a
deadlock, a pool exhaustion, a pinning stall, a thread leak and an unbounded buffer
identically, and destroys the evidence for all of them. That means you are guaranteed
not to know which one it was unless someone captured a diagnostic first. The
postmortem's most valuable output in that case is not a fix at all — it is the
capture-before-mitigate bundle, which is the action item that makes every future
incident of that shape learnable.

**3. When you join a team and want to know what is actually true about its systems.**
Read the last six months of postmortems before reading the architecture docs. The
architecture docs describe the intent; the postmortems describe the behaviour. You
will also learn a great deal about the organisation from the shape of the documents:
whether action items get owners, whether dead ends survive editing, and whether the
word "forgot" appears. Those three tell you more about how the team works than any
onboarding conversation will.

---

## Connected topics

**Prerequisites — the drills that are your raw material:**

- **79 — Memory leaks and heap dumps.** The unbounded static cache OOM, and the
  `ThreadLocal`-on-a-pooled-thread retention. The first is the cleanest example of a
  contributing factor that is a library default; the second is the cleanest example
  of a failure with no signal until it is terminal.
- **90 — Executors, pool sizing, bounded queues.** The unbounded-queue overload. The
  canonical "someone forgot" incident, and the one whose strongest control is a
  factory rather than a warning.
- **94 — Explicit locks.** The wallet/inventory lock-ordering deadlock, and its
  symptom collision with Topic 109's pool deadlock — which is where the runbook
  confirmation step earns its place.
- **98 — Concurrency bug taxonomy.** The thread leak. Useful because its timeline is
  slow and its detection story is entirely about a signal nobody had.
- **101 — Virtual threads and pinning.** The worked example above: an incident where
  a correct change altered the cost of existing correct code.
- **105 — Backpressure.** The unbounded `Flux` buffer. Same class as 90, different
  paradigm, which is exactly the kind of cross-paradigm class a good postmortem
  names.
- **109 — HikariCP and the pool deadlock.** The pool-vs-pool cycle, and the incident
  where the service is up and serving nothing.
- **111 — Resilience4j.** The retry storm: an incident where your own mitigation
  amplified someone else's outage.
- **115 — The outbox.** The lost event on `kill -9`: an incident with no error, no
  alert, and a customer-visible consequence days later.
- **118, 119 — Metrics and tracing.** Your evidence base for the timeline. A timeline
  is only as good as the signals that were being recorded when nobody was looking.
- **123 — Graceful shutdown.** Dropped requests during deploys: the incident class
  that is invisible unless you measure it, and that recurs on every release.

**Built on:**

- **124 — GATE: the production-readiness review.** The register is the before and the
  postmortem is the after. An incident already in the register asks a better question
  than a novel one: why was the detection item not prioritised? Every postmortem
  should update the register, and a register that never changes is not being used.
- **130 — SLOs and error budgets.** Budget consumed is the honest severity measure.
  "This incident cost <n>% of the window's budget" is a better severity statement than
  any label, and it makes the action-item prioritisation argument for you.
- **132 — Engineering standards.** The control hierarchy comes from here, and this is
  where you find out which controls Java actually offers for a given bug class —
  ErrorProne, ArchUnit, a wrapper, a ratcheting baseline. "Add a checklist item" is
  what you write when you have not read Topic 132.

**This unlocks:**

- **134 — Influence without authority.** An action item that fixes your codebase and
  not the organisation's is scoped too narrowly. Getting the wrapper adopted by three
  other teams is an adoption problem, and the postmortem is the most persuasive
  artefact you will ever have for it — because it is evidence rather than opinion.
- **135 — The capstone.** "Walk me through a postmortem you wrote" is a standard
  Principal prompt, and the follow-up chain is always about the action items: which
  one prevents a class, which one you would drop, and who approved the time. This
  document is one of the artefacts you will defend under sustained hostile
  questioning.

---

*Every timestamp, duration and impact figure in this document is a blank. The worked
example uses the Topic 101 drill you ran yourself; fill its timeline from your own
notes, not from anything here. Blameless postmortems are a widely used industry
practice, popularised largely by Google's SRE literature and the web-operations
community; the norm is theirs, the template is this curriculum's, and no incident,
statistic or quotation in this document is invented or borrowed.*
