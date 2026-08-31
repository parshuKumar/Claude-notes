# 132 — Engineering Standards and Rollout Without Stalling Delivery

## Phase: 12 — Principal Track
## Category: ELITE
## Java baseline: 21  |  Notes features from: 21/25
## Project spine: a code-review guideline for the `orderflow` team, plus a plan to introduce a static-analysis gate on a legacy codebase without a three-month freeze

---

## Mechanical statement

> **A standard is adopted when following it is cheaper than ignoring it.**
>
> **Gates on new and changed code with a ratcheting baseline are cheap. Gates on
> an entire legacy codebase are a freeze, and freezes get switched off.**

Both halves are mechanical claims about cost, not claims about culture.

**The first half.** Every standard has an adoption cost (learn it, follow it, fix
the thing it flags) and an ignoring cost (a build failure, a review comment, an
awkward conversation, a production incident). Engineers, under deadline, do the
cheaper one. That is not a discipline failure; it is arithmetic, and it is the
same arithmetic that decides whether a migration stalls (Topic 127). If your
standard is a wiki page, the cost of ignoring it is approximately zero, so it will
be ignored — not by bad engineers, but by good ones with a deadline. If your
standard is a formatter that runs on save, the cost of following it is
approximately zero, so it is followed universally without anyone deciding to.

**The second half.** A gate is a step change in the cost of ignoring. Applied to
new and changed code, that cost lands on one file at a time and is proportional to
what the author was already doing. Applied to an entire legacy codebase, it lands
on everyone at once, as thousands of pre-existing violations that nobody's current
work created — which means nobody can merge anything until an unbounded amount of
unrelated work is done. That is a freeze. And freezes get switched off, quickly,
usually by whoever is trying to ship a fix on a Friday, with a build flag that
then survives forever.

**The consequence you must internalise.** When a gate is switched off in week one,
you have not merely failed to install the gate. You have spent the organisation's
willingness to let you install gates — for a year, maybe longer, and it attaches
to you personally. The next time you propose a check, the room remembers the last
one. **You get roughly one attempt.** So the design of the rollout matters more
than the choice of tool, and the whole content of this topic is: how to spend that
one attempt so that it works.

---

## The bridge from what you know

### What transfers — completely, and I will not re-teach it

You have run code review and set up linting in TypeScript. All of this transfers
and none of it belongs in your artefact:

- **Running a code review.** Tone, size limits, turnaround expectations,
  distinguishing blocking from non-blocking comments, not being a jerk. You know
  this and it is not language-specific.
- **Automating formatting so it never appears in review.** Prettier taught the
  industry this lesson permanently. Java's equivalent is a formatter in the build;
  the principle is identical.
- **Errors versus warnings, and warning fatigue.** You know that a warning nobody
  fixes is worse than no warning, because it trains people to scroll past output.
- **CI gates and required checks.** Same mechanism.
- **The "clean as you code" / new-code-quality-gate idea.** This is a widely used
  industry practice — SonarQube's new-code period is probably its best-known
  implementation — and you have likely used something in this shape. The *idea*
  transfers whole. What does not transfer is how you implement the ratchet in a
  Java toolchain, which is most of this topic.
- **Deprecating an internal API with a migration path and a deadline.** Same
  practice; Topic 127 gave you the enforcement mechanics.

### What does not transfer — the Java static-analysis landscape

| You know (TS/JS) | Java reality | Verdict |
|---|---|---|
| One dominant linter (ESLint) that does nearly everything, plus Prettier | **A layered set of tools that do genuinely different things** at different points in the build, with different failure modes and different baseline mechanisms | **NO ANALOGUE** — you must choose a stack, and the choice determines what a "gate" even means. |
| Lint runs as a separate step; a violation is a lint error | **Error Prone runs *inside* javac.** A violation at ERROR severity is a **compile failure** — heavier, earlier, and impossible to skip past | **NO ANALOGUE**, and it makes the legacy-codebase problem sharper, not softer. |
| `strictNullChecks` is a compiler flag you can enable per-file with a pragma | **Java has no null checking.** NullAway approximates it, driven by `@Nullable` annotations, and — crucially — is configured **per package** via `AnnotatedPackages` | **PARTIAL** — and the per-package opt-in is the single best ratchet mechanism in the Java ecosystem. |
| Type information is available to the linter via the TS compiler | **Error Prone has full javac type information; SpotBugs analyses bytecode after compilation.** Those see different bugs and produce different false positives | **PARTIAL** — the layering is the point. |
| `// eslint-disable-next-line` | `@SuppressWarnings("...")`, Error Prone's `@SuppressWarnings` support, SpotBugs exclusion filter XML | **HONEST ANALOGUE** — except the SpotBugs filter file is external, which makes it a natural baseline artifact. |
| Architecture rules are a lint plugin, if you have them at all | **ArchUnit** — architecture rules written as ordinary JUnit tests, so they run in the normal test phase and fail like a test | **NO ANALOGUE** — and it is how a module boundary becomes real (Topic 128). |
| Deprecation is a JSDoc tag and a lint rule | **`@Deprecated(since="…", forRemoval=true)`** makes javac emit a *removal* warning, which `-Werror` converts to a build failure — a language-level deprecation mechanism with teeth | **PARTIAL** — Java gives you a real forcing function you should be using. |

### The tools, what each is actually for, and its baseline mechanism

You must know this to write the rollout plan, because the rollout plan is mostly a
statement about baselines.

**Error Prone** — a javac plugin (from Google) that runs during compilation with
full type information. It catches bug patterns rather than style: `==` on boxed
types, format-string mismatches, mistaken equals, misused mocks, unused return
values on `@CheckReturnValue` methods. Severity is per-check and configurable
(`-Xep:CheckName:OFF|WARN|ERROR`), which is your primary rollout dial. It also has
a **patch mode** that generates fixes automatically for many checks — which is the
mechanism that makes a burndown cheap rather than manual.

**NullAway** — an Error Prone plugin doing fast, practical null analysis driven by
`@Nullable` annotations. Its rollout property is the important one: it is enabled
for a configured set of **annotated packages**, so you can turn it on for one
package, get it clean, and move to the next. That is a ratchet built into the tool
rather than bolted on. Note also that Spring Boot 4 adopts JSpecify null
annotations portfolio-wide, which means the annotations you need are increasingly
present on the framework side of your call boundaries.

**SpotBugs** — analyses compiled bytecode after the compile phase. It sees things a
source analyser cannot and has a different false-positive profile. Its baseline
mechanism is an **exclusion filter file** (XML), which is external to the source
and therefore natural to treat as a shrinking, version-controlled artifact.

**Checkstyle** — style and structure. Mostly superseded for formatting by a
formatter that runs in the build; still useful for structural rules a formatter
does not express.

**Spotless** — applies formatting (and simple content rules) in the build, with a
`ratchetFrom` setting that restricts it to files changed relative to a git
reference. That single option is the cleanest ratchet in the Java build ecosystem
and it is worth naming explicitly in your plan.

**ArchUnit** — architecture rules as JUnit tests: no cycles between packages, this
package may not import that one, controllers may not reach repositories directly.
Runs as a test, fails as a test, and supports a **freeze/violation-store** mode
that records existing violations and fails only on new ones — a ratchet built into
the tool.

**SonarQube** (or an equivalent platform) — aggregates and adds a "new code" quality
gate, which is the productised version of the ratchet. If your organisation already
runs one, use its new-code period rather than building a parallel mechanism.

> **One line of honest uncertainty:** the specific flag names, configuration keys
> and available checks in all of the above change between versions. I am confident
> about what each tool is and about the shape of its baseline mechanism; verify the
> exact configuration syntax against the current documentation for the version you
> pin, before it goes in a plan with your name on it. Primary sources:
> `errorprone.info`, `github.com/uber/NullAway`, `spotbugs.readthedocs.io`,
> `github.com/diffplug/spotless`, `archunit.org`.

---

## What is this?

Two artefacts that look unrelated and are the same problem.

**A code-review guideline** is a written statement of what review is *for* on this
team, what a reviewer is expected to check, what is out of scope because a machine
checks it, and what the service levels are. Its purpose is to make review
consistent and fast, and to stop it being a negotiation about taste.

**A static-analysis rollout plan** is a sequenced plan for making some class of
defect impossible to merge, without stopping delivery.

They are the same problem because **every rule you can automate should leave the
review guideline and become a gate**, and every gate you cannot automate stays in
the guideline. The guideline is the residue: the things that genuinely require a
human. If your review guideline contains "use four spaces" or "prefer
`StringBuilder` in loops", you have humans doing a machine's job, and the machine's
job is the part humans do worst — consistently, at 6pm, on the fortieth file.

### The three things review is actually for

Say this at the top of the guideline, because it settles most arguments:

1. **Correctness a machine cannot check.** Does this code do what the ticket says?
   Is the concurrency safe? Does the transaction boundary match the invariant? Is
   the failure mode acceptable? These require understanding intent, and no
   analyser has intent.
2. **Design and reversibility.** Is this the right shape? Will it be expensive to
   change? Does it create a coupling we will regret? This is where a senior
   reviewer earns their time.
3. **Knowledge transfer.** Someone other than the author now knows this code
   exists. This is the benefit that survives even when the review finds nothing,
   and it is the reason review is not optional on low-risk changes.

Everything else — formatting, import order, naming conventions, obvious bug
patterns, banned APIs, architectural boundaries — is a machine's job. If a human is
doing it, that is a defect in your tooling, and it should appear on the rollout
plan.

### The ratchet, defined precisely

A ratchet has three properties. All three are required; two out of three is a
mechanism that quietly stops working.

1. **A recorded baseline.** The current violation count or set, committed to the
   repository so it is visible in diffs and cannot drift unnoticed.
2. **A gate that fails when the number goes up.** New violations are blocked. This
   is the part that delivers value on day one, and it is the part that is cheap
   because it only ever costs the author of a change something proportional to the
   change.
3. **A mechanism that lowers the number over time**, with an owner and a schedule.
   Without this, the baseline is permanent, the codebase is permanently split into
   "checked" and "grandfathered", and in two years the number is unchanged. This
   is the part everyone skips, and skipping it is why so many baselines have the
   same value they had at creation.

The third property is where the plan is judged, exactly as the stall-prevention
mechanism is where a migration plan is judged. Same failure shape, same cause: the
remaining work stops hurting anyone.

### Credibility as a budget

This is the Principal-level frame and it does not appear in tool documentation.

Introducing a gate spends organisational credibility. If it works — if it catches
real bugs and costs almost nothing — you get more. If it fails — if it blocks a
release and gets disabled — you get much less, for a long time, and the failure
attaches to you rather than to the tool. Two consequences follow, and both are
counterintuitive:

- **Start with checks that are nearly free and unarguably right.** Not the ones
  that would catch the most bugs. The first gate's job is not to catch bugs; it is
  to demonstrate that gates are cheap. A check that fires rarely, has essentially
  no false positives, and catches something everyone agrees is a bug is worth more
  as an opening move than a high-yield check with a 20% false-positive rate.
- **The first gate must never block an urgent fix.** If someone cannot ship a
  production hotfix because of your check, the check is gone by the end of the day
  and so is your next proposal. Design the escape hatch deliberately, make it
  loud, and make it logged — do not leave it to be invented under pressure.

---

## Why does it matter?

**1. Because the failure mode is specific, common, and permanently damaging.** The
sequence is: enable the tool across the codebase, 4,000 pre-existing violations,
nobody can merge, someone adds a skip flag on Friday afternoon, the flag is still
there two years later, and the person who proposed it has spent their credibility
for nothing. This happens constantly. It is entirely avoidable and the avoidance
is a five-minute design decision made before the first commit.

**2. Because a code-review guideline is one of the highest-leverage documents a
Principal engineer can write, and almost nobody writes one.** Review is where a
team's standards are actually transmitted, and by default that transmission is
inconsistent, personality-driven, and biased toward whatever the loudest reviewer
cares about — which is usually style, because style is easy to see. A written
guideline converts that into something teachable, and it makes it possible for a
junior engineer to review well, which is the thing that scales.

**3. Because in Java the review checklist is genuinely long and genuinely
non-obvious.** The bug classes from Phases 5, 8, 9 and 11 — the checked-exception
rollback, `@Transactional` self-invocation, the mutable `HashMap` key, the
unbounded queue, the `ThreadLocal` on a pooled thread, `synchronized` around I/O
under virtual threads, the high-cardinality metric tag — are all invisible in a
diff to someone who has not been bitten. Encoding them is how the team stops
rediscovering them one incident at a time. This is also where Topic 133's action
items land: a postmortem action item that says "add a review checklist item" is
weak; one that says "add an Error Prone check" is strong; and knowing which is
available for a given bug class is this topic's job.

**4. Because gates are how a standard survives your absence.** A rule you enforce
in review is a rule that exists while you are reviewing. A rule in the build is a
rule that exists on a Tuesday when you are on leave and a contractor is shipping
under pressure. Converting your judgment into a mechanism is the single most
durable thing you can do with it, and it is a large part of what distinguishes
Principal-level impact from very good Senior-level impact.

---

## The decision, framed

Four decisions.

### Decision 1 — Which rules become gates, and which stay human

Apply this test to every rule you would like the team to follow:

| Question | If yes | If no |
|---|---|---|
| Can a machine decide it with essentially no false positives? | Gate it | Keep it in the guideline |
| Would a violation be a bug, or merely a preference? | Gate it | Do not gate it — preferences in a gate generate resentment and get disabled |
| Is the fix mechanical? | Gate it, and use the tool's patch mode for the burndown | Gate it, but expect a slower burndown |
| Does it require knowing the author's intent? | Never gate it | Guideline |

The failure to avoid is gating a preference. A gate that enforces a taste
disagreement will be argued about, the argument will be about the gate rather than
the code, and eventually someone will win the argument by turning it off.

### Decision 2 — The severity dial, and where you start

For every check, three positions:

- **OFF** — not run.
- **WARN** — reported, does not fail the build.
- **ERROR** — fails the build. In Error Prone's case, fails **compilation**.

The rollout is a sequence of moves along this dial, per check, not a single
switch for the tool. Start with a small set at ERROR, a larger set at WARN, and
the rest OFF. A common and effective opening position: everything the tool would
flag at WARN so the count is visible, plus a hand-picked handful at ERROR chosen
because they have close to zero existing violations and are unarguably bugs.

### Decision 3 — The baseline mechanism

Pick per tool, and the choice determines how the ratchet works:

| Tool | Baseline mechanism | Ratchet shape |
|---|---|---|
| Error Prone | Per-check severity; `@SuppressWarnings` at the site | Move checks OFF → WARN → ERROR one at a time; suppressions are visible in the diff |
| NullAway | `AnnotatedPackages` configuration | **Opt in one package at a time** — the cleanest ratchet available |
| SpotBugs | Exclusion filter XML, committed | Entries removed over time; the file's line count is the burndown metric |
| Spotless | `ratchetFrom <git ref>` | Only changed files are checked, automatically |
| ArchUnit | Violation store / freeze | Existing violations recorded; only new ones fail |
| Sonar-style platform | New-code period | Gate applies to code changed in the period |

Two things to notice. First, several of these ratchets are **built into the tool**,
which means the mechanism is maintained by someone else and does not rot. Prefer
those. Second, the mechanism you choose determines what "the number" is in your
burndown — a filter-file line count, a package count, a violation-store size — and
you should say which number you are tracking before you start, because a burndown
without a defined metric drifts into anecdote.

### Decision 4 — The escape hatch, designed rather than improvised

There will be an urgent change that trips the gate. Decide now what happens,
because if you do not, the answer will be invented at 5pm on a Friday and it will
be a global skip flag.

A good escape hatch has four properties:
- **Local, not global.** A suppression at the site, not a build flag that turns the
  tool off everywhere.
- **Visible in the diff.** A reviewer sees it and can ask about it.
- **Requires a reason.** `@SuppressWarnings("SomeCheck") // PLAT-1234: …` — the
  comment is the point.
- **Counted.** Suppressions go on the burndown alongside violations, or they become
  a second, invisible baseline.

Say all four in the plan. The escape hatch is not a weakness in the gate; it is
what allows the gate to be strict everywhere else.

---

## Example 1 — a minimal illustration

Small, real, and it contains the whole mechanic.

### The situation

Someone on the team keeps writing `if (order.getStatus() == OrderStatus.PAID)`
comparisons on boxed types elsewhere in the codebase — `if (quantity == someInteger)`
where both are `Integer`. Topic 01 taught you why this is a latent bug: outside the
−128..127 cache, `==` compares references. It has caused one production defect
already.

### The wrong move, which is also the obvious one

Add "don't use `==` on boxed types" to the review checklist and mention it in
standup.

What happens: it is caught in review perhaps two thirds of the time, depending on
who reviews and how tired they are. The remaining third ships. The rule is not
written down anywhere a new joiner will find in their first month. And the person
who mentioned it becomes the human linter for this rule — a role that is boring,
unrewarded, and disappears when they go on leave.

### The right move, and the sequence

**Step 1 — measure.** Enable the relevant Error Prone check at WARN and build.
Count the existing violations. Suppose the number is small — this is a check that
tends to have few existing hits, because most instances have already been fixed as
bugs.

**Step 2 — decide from the count.** If the count is small enough that one person
can fix it in an afternoon, fix it and go straight to ERROR. That is the ideal
case and you should look for these deliberately when choosing your opening set.

If the count is large, you do not go to ERROR. You go to: WARN globally, ERROR on
new and changed code, and a recorded baseline.

**Step 3 — the gate.** At ERROR, the build fails on any new occurrence. A compile
failure, immediately, at the author's desk, before review. Cost of following the
rule: zero, because the author simply writes `.equals()`. Cost of ignoring: the
code does not compile. The arithmetic is now on the standard's side, permanently,
without anyone needing to remember it.

**Step 4 — remove it from the review checklist.** This step is not optional and it
is the one people forget. If the machine checks it, humans must stop, or you have
added work rather than moved it. The guideline gets *shorter* every time a gate
lands, and that shrinkage is the visible evidence that the programme is working.

### What you write down

Four lines:

> Check: `<the boxed-equality check>`. Existing violations: `<count>`. Decision:
> fix all, enable at ERROR. Review-checklist item removed: "don't use `==` on
> boxed types". Owner: `<name>`. Date: `<date>`.

That is the atomic unit of the whole rollout plan. Example 2 is this, forty times,
in a sensible order.

---

## Example 2 — the real decision on the project spine

### The setting

`orderflow` is a production-shaped service (Topic 124) with roughly a decade of
Java-8-era habits still visible in the older packages. The team is six engineers.
There is a formatter but it is not enforced. There is no static analysis. Code
review is inconsistent: two reviewers are thorough about concurrency, two are
thorough about naming, and nobody has written down which is expected.

Recent history that matters, because it is the argument:

- The Topic 79 drill found an unbounded `static Map` cache that caused an
  `OutOfMemoryError` under load.
- The Topic 90 drill found an unbounded queue on a `newFixedThreadPool` that
  converted overload into latency and then OOM.
- The Topic 54 drill found a checked exception thrown from a `@Transactional`
  method, which **committed** the write.
- The Topic 94 drill found a wallet/inventory lock-ordering deadlock.
- The Topic 101 drill found `synchronized` held across the payment-gateway call,
  pinning carriers under virtual threads.
- The Topic 118 drill found an order ID used as a metric tag, exploding series
  cardinality.

Six known bug classes, all reproduced, all documented. That history is the
strongest possible material for this plan, because each one answers "why this
check" with an incident rather than an opinion.

### Part 1 — The code-review guideline

The guideline has five sections. Keep it to two pages; a review guideline nobody
reads is the same as no guideline.

**Section 1 — What review is for.** The three purposes from above, stated in three
sentences. Plus the exclusion, stated explicitly because it saves the most time:
*formatting, import order and style are not reviewed; they are enforced by the
build. If you find yourself commenting on them, file a bug against the build.*

**Section 2 — Service levels.** These make review predictable, which is what makes
it fast:

| Expectation | Value |
|---|---|
| First response to a review request | within `<your team's agreed time>` |
| Maximum change size for a normal review | `<agreed lines / files>` — larger needs a heads-up and probably a design doc (Topic 131) |
| Who may approve | `<rule>` — and which changes need a second approver |
| Comment labels | `blocking:` / `question:` / `nit:` — **`nit:` is never blocking** |
| Disagreement escalation | Author and reviewer disagree twice → a third person decides; nobody sits in a loop |

The comment labelling convention is a small thing that changes review culture more
than anything else in the document. Most review friction comes from ambiguity about
whether a comment must be addressed.

**Section 3 — What the reviewer checks.** This is the Java-specific list, and every
item names the bug class rather than the rule, because the bug class is what makes
it memorable. The `orderflow` version:

*Correctness and data integrity*
- Does any class used as a `Map` key or `Set` element have a mutable field
  participating in `equals`/`hashCode`? (Topic 13 — the entry becomes unreachable
  and still counts toward `size`.)
- Does `equals` have a matching `hashCode`, and do both use the same fields?

*Transactions and persistence*
- Is a `@Transactional` method called from a sibling method in the same class?
  (Topic 40 — self-invocation bypasses the proxy; the annotation does nothing.)
- Does a `@Transactional` method throw a **checked** exception where rollback is
  expected? (Topic 54 — default rollback covers unchecked only; this commits.)
- Is there any network call, or any slow non-database work, inside a transaction?
  (Topic 55 — the connection is pinned for the transaction's whole life, and this
  is how one slow downstream takes out every endpoint.)
- Does a read path return entities rather than projections, and does anything
  touch a lazy association after the session closes? (Topics 49, 50.)
- Does a new query in a loop create an N+1? Is there a query-count assertion?

*Concurrency*
- Is any queue or pool unbounded? `newFixedThreadPool` has an unbounded queue.
  (Topic 90.)
- Is a `ThreadLocal` set on a pooled thread without a `remove()` in a `finally`?
  (Topic 79b.)
- Do two code paths acquire the same two locks in different orders? (Topic 94.)
- Is anything `synchronized` around blocking I/O, with virtual threads enabled?
  (Topic 101 — pinning.)
- Is a check-then-act sequence used on a concurrent map where an atomic operation
  exists? (Topic 92.)

*Operations*
- Does any metric tag carry an unbounded value — an order ID, a user ID, a URL
  path with an identifier in it? (Topic 118 — this kills the monitoring system.)
- Is a new external call covered by a timeout, a circuit breaker and a bounded
  retry with jitter? (Topics 111, 117.)
- Does a liveness probe depend on a downstream? (Topic 121.)

*Design*
- Is the failure mode of this code stated anywhere — in a test, a comment, or the
  design doc?
- Is this change reversible? If not, does it need a design doc?

Every one of those items is a candidate for a gate, and part 2 of the artefact is
the plan to move as many as possible out of this list and into the build. The
guideline should say so explicitly, so the team understands that the list is
supposed to shrink.

**Section 4 — What the author does before requesting review.** Short: run the
build, write the description (what and why, not how), keep the change to one
concern, and say what you tested. A change description that says why the change
exists is worth more reviewer attention than any checklist.

**Section 5 — Escalation and exceptions.** Who to ask when the guideline and the
deadline conflict, and the explicit statement that shipping under an exception is
acceptable and shipping under a silent exception is not.

### Part 2 — The static-analysis rollout plan

Six phases. The whole plan is designed so that **phase 2 delivers value and no
disruption**, because phase 2 is where credibility is won or lost.

**Phase 0 — Measure, do not enable. (Days.)**

Run each tool in report-only mode. Produce one table:

| Tool | Check | Existing violations | False-positive rate (sampled) | Bug class | Fix mechanical? | Related incident |
|---|---|---|---|---|---|---|
| | | | | | | |

The last column is what turns this from a tooling proposal into an argument. Six
of your rows will point at a drill from Phases 8–11. That is the plan's funding
case and it should be on page one.

Sample the false-positive rate by hand on a subset. A check with a high
false-positive rate is disqualified from the opening set regardless of how valuable
it looks, because false positives are what teach people that the tool is noise.

**Phase 1 — Formatting first, ratcheted. (One pull request.)**

Enable the formatter in the build with `ratchetFrom` pointed at your main branch,
so it applies only to files changed relative to that reference. No mass reformat —
a mass reformat destroys `git blame`, creates conflicts with every open branch,
and is exactly the "big bang" pattern Topic 127 warned you about.

Why formatting first: it is unarguable, it is mechanical, it has no false
positives, and it immediately removes an entire category of comment from review.
It is the cheapest possible demonstration that a gate can be free. Ship it alone,
so its cheapness is visible and not confounded with anything else.

**Phase 2 — The opening gate: a handful of checks at ERROR. (One pull request.)**

Choose from the Phase 0 table:
- checks with **zero or near-zero** existing violations,
- with essentially no false positives,
- where a violation is unarguably a bug.

Fix the handful of existing violations in the same pull request. Enable those
checks at ERROR. Everything else the tool offers goes to WARN so the counts become
visible in build output without failing anything.

This phase is the one that decides the programme. It must be boring. Nobody's
build breaks, nobody has to fix unrelated code, and from now on a specific class of
bug cannot be merged. If someone asks "what did that cost us?", the honest answer
is "nothing", and that answer is the asset you are building.

**Phase 3 — New-and-changed-code gating for the rest. (Weeks.)**

Now the large-violation-count checks. Never enabled globally at ERROR. Instead:

- **NullAway**: enable for one package via `AnnotatedPackages`. Choose a package
  that is small, actively developed, and important — for `orderflow`, the payments
  or wallet package, because that is where a null defect costs the most. Annotate,
  get it clean, ship. Then the next package. The tool's own configuration is the
  ratchet.
- **SpotBugs**: generate the exclusion filter from the current violations, commit
  it, and fail on anything not in the filter. The filter is the baseline; its size
  is the burndown metric.
- **ArchUnit**: write the boundary rules you want (this is where Topic 128's
  extraction boundary is enforced), record existing violations in the violation
  store, and fail only on new ones.
- **Diff-scoped checks**: for anything without a built-in ratchet, run the analysis
  against the merge base and fail only on violations in changed lines. Prefer the
  tool's own mechanism when it has one; a home-grown diff script is a thing you now
  maintain.

**Phase 4 — The ratchet becomes a rule.**

Commit the baseline counts. Add a build check: **the number may not increase.** A
pull request that adds a suppression or a filter entry fails unless it removes one
elsewhere, or unless a reviewer explicitly approves the increase with a reason in
the diff.

This is the property that stops a baseline becoming permanent. Without it, phase 3
produces a codebase permanently divided into "checked" and "grandfathered", and
the grandfathered part grows quietly as files are copied and patterns are imitated.

**Phase 5 — Burn down, with the mechanical ones automated. (Ongoing, with an
owner and a schedule.)**

- Use Error Prone's patch mode to generate fixes for the mechanically fixable
  checks. Review the patch like any other change, in reviewable batches, per
  package or per check — never one enormous "fix all warnings" pull request,
  which cannot be reviewed and will be approved without being read.
- Assign the non-mechanical ones by package owner, with a target per iteration.
- Publish the burndown **by package and owner**, not as a percentage. Same rule as
  Topic 127, same reason: a percentage is a number nobody is accountable for.
- The person or squad proposing this does the first two rounds themselves. Making
  adoption cheaper than non-adoption (Topic 134) applies here exactly as it does
  in a migration.

**Phase 6 — Retire the baseline.**

When a check's baseline reaches zero, promote it to ERROR globally and **delete the
baseline entry**. Deleting it matters: a baseline file with stale entries is a
place where new violations can hide, and its size is your burndown metric.

And the step that closes the loop back to part 1: **each time a check goes to
ERROR, remove the corresponding line from the code-review guideline.** The
guideline shrinks. That shrinkage is the visible evidence that the machine is
taking over the machine's work, and it is worth calling out in the team's review of
the programme.

### The exceptions register

One table, in the repository, next to the baseline:

| Suppression / filter entry | Where | Reason | Ticket | Owner | Expiry |
|---|---|---|---|---|---|
| | | | | | |

Suppressions with no expiry become permanent. Suppressions with an expiry that
nobody checks also become permanent — so the expiry is enforced by a build check
that fails when an entry is past its date, which turns it into the same ratchet as
everything else.

---

## Wrong approach → exact symptom → root cause → fix

Organisational symptoms. What you would have *seen*.

---

### Wrong approach 1 — the gate switched on across the whole legacy codebase

**Wrong:** the tool is enabled at ERROR for all checks, everywhere, in one pull
request, because "we should hold ourselves to a high standard".

**Exact symptom — what you would have SEEN:**

- The enabling pull request's own build fails with several thousand violations.
  The author spends two days trying to fix them and gets perhaps a fifth of the
  way.
- It is merged anyway — with the tool at ERROR — after someone argues that
  "otherwise we'll never do it". Every open branch in the repository is now red.
- Three engineers are blocked on unrelated work. One of them is fixing a production
  bug and cannot merge because a file they touched has fourteen pre-existing
  violations that have nothing to do with their change.
- Within about a day, someone adds a global disable — a build property, a skip
  flag, `-XepAllErrorsAsWarnings`, `<skip>true</skip>` — with the commit message
  "unblock build". It is the correct decision under the circumstances.
- That property is still in the build file two years later. Nobody remembers what
  it is for. Removing it is now a scary change nobody will schedule.
- The person who proposed the gate is now the person who broke the build for a
  week. The next time they propose a check — even a good, cheap one — the room's
  first response is "is this going to be like last time?" **The credibility is
  spent and it does not come back quickly.**
- The eventual retrospective concludes "we should have done it incrementally",
  which is true and is not actionable, because nobody wrote down what incrementally
  means.

**Root cause:** the cost of the gate was imposed on everyone at once, in proportion
to the codebase's history rather than in proportion to anyone's current work. That
converts a quality mechanism into a freeze, and a freeze is a thing that must be
removed for the organisation to function — so it is removed. The tool was fine; the
rollout ignored who pays and when.

**Fix:**
1. **New and changed code only, with a ratcheting baseline.** The cost lands on the
   author of a change, proportional to the change. Nobody is asked to pay for
   history they did not write.
2. Use the tool's own ratchet where it has one: NullAway's `AnnotatedPackages`,
   SpotBugs' exclusion filter, ArchUnit's violation store, Spotless' `ratchetFrom`,
   the platform's new-code period. Built-in ratchets are maintained by someone else
   and do not rot.
3. Phase the severity dial per check, never per tool. WARN makes the count visible
   at zero cost; ERROR is earned per check.
4. Open with checks that have near-zero existing violations, so phase 2 costs
   nothing and demonstrates that gates are cheap. Spend the first attempt on
   credibility, not on yield.
5. Design the escape hatch before you need one: local, visible in the diff,
   reasoned, counted. If you do not, a global one will be invented under pressure.

---

### Wrong approach 2 — a review guideline made of style opinions

**Wrong:** the guideline is a list of preferences — naming, ordering, comment
style, method length — written by whoever felt strongly.

**Exact symptom — what you would have SEEN:**

- Review threads of a dozen comments, eleven of them about naming and formatting,
  and the twelfth — if it appears at all — noting that the new executor has an
  unbounded queue.
- Two reviewers giving contradictory instructions on the same pull request, both
  citing the guideline, both correct, because the guideline has two sections that
  disagree.
- Review latency rising. Authors batching changes to avoid the process, which makes
  changes larger, which makes review worse, which makes latency rise further.
- Junior engineers learning that review is about surviving taste rather than about
  correctness, and reviewing that way themselves within a quarter.
- A production incident whose cause was visible in a diff that had four approvals
  and nineteen comments, all about style.
- The guideline is not read by new joiners because it is long and its first page is
  about braces.

**Root cause:** the guideline documented what was easy to see rather than what was
expensive to get wrong. Style is highly visible in a diff and requires no domain
knowledge, so it attracts reviewer attention by default; correctness requires
understanding intent and is therefore quietly skipped, especially by reviewers who
do not know the area.

**Fix:**
1. **Automate every mechanical rule and delete it from the guideline.** Formatter in
   the build with a ratchet, and no formatting comments in review — ever, by rule.
2. Rewrite the guideline around bug classes you have actually experienced, each
   citing its incident. The `orderflow` list above is six drills long and every
   item earns its place.
3. Add the comment labels: `blocking:` / `question:` / `nit:`, with `nit:` never
   blocking. This one convention removes most review friction because it removes
   ambiguity about obligation.
4. Cap the guideline at two pages. If it grows, something must be automated or
   removed. Length is the signal that the automation programme has stalled.

---

### Wrong approach 3 — a standard announced with no mechanism

**Wrong:** the standard is a wiki page, an architecture-forum presentation, and a
message in the team channel. Compliance is expected.

**Exact symptom — what you would have SEEN:**

- Compliance sits somewhere well short of universal and never improves. Nobody
  measures it, so the number is an impression rather than a fact.
- The rule is enforced inconsistently in review, depending entirely on which
  reviewer and how busy they are. Authors learn which reviewers care and, entirely
  rationally, request the ones who do not.
- One senior engineer becomes the human enforcement mechanism, comments on the same
  thing weekly, is privately resented for it, and eventually stops — at which point
  compliance drops to zero with no announcement.
- New joiners have no idea the standard exists. It is not in the build, not in a
  template, not in the code they read.
- Six months later somebody proposes the same standard, unaware, and is told "we
  already decided that", which is true and has had no effect.
- The wiki page's last edit is the day it was created.

**Root cause:** the cost of ignoring the standard was zero. Announcing a standard
changes what people *know*; it does not change what is cheaper. Under a deadline,
knowledge loses to arithmetic every time, and blaming that on discipline is both
inaccurate and unhelpful.

**Fix:**
1. Every standard ships with a mechanism, or it does not ship. Mechanisms, in
   descending order of durability: a build failure; a test failure; a template or
   scaffold that makes the right thing the default; a required review checklist
   item; a document. Anything below the third line is decoration.
2. If no mechanism is possible today, say so explicitly and write the standard as
   guidance rather than as a rule. An honest "we prefer X and cannot enforce it" is
   much better than a rule that everyone learns to ignore, because the second one
   teaches people that rules here are optional — and that lesson generalises to
   your rules that *are* enforceable.
3. Make the right thing the default in the scaffolding: a project template with the
   formatter, the checks and the bounded-queue executor factory already wired.
   Defaults outperform rules by a wide margin because they cost nothing to follow.

---

### Wrong approach 4 — a baseline that never ratchets

**Wrong:** the gate is correctly scoped to new code, with a baseline file recording
existing violations. Nobody owns lowering it.

**Exact symptom — what you would have SEEN:**

- The baseline file has the same number of entries two years later. Possibly more,
  because entries were added during "urgent" changes and never removed.
- The codebase is permanently two codebases: the checked part and the
  grandfathered part. New engineers cannot tell which is which without opening the
  filter file.
- A new file is created by copying an old one, inherits its patterns, and its
  violations get added to the baseline because "it was already like that" — so the
  grandfathered region grows.
- The burndown, if it exists, is a percentage on a slide that has not moved in four
  quarters. Nobody looks at it.
- The original bug class the check was for recurs — in the grandfathered region,
  which is exactly where it was always going to recur, because that is where the
  code that predates the check lives.
- Someone proposes "a quarter to clean up the baseline", which is unfundable
  because it delivers no visible value, so it is never scheduled.

**Root cause:** the mechanism had two of the three ratchet properties. Blocking new
violations delivered immediate value, which relieved the pain — and the remaining
work then had a cost and no felt benefit, so it lost every prioritisation
conversation. Precisely the stall from Topic 127, in a different costume.

**Fix:**
1. The baseline is a **committed number that may only go down**, enforced by a
   build check. An increase requires an explicit approval visible in the diff.
2. A named owner per package and a target per iteration, published as a table of
   packages and owners — not a percentage.
3. Automate the mechanical fixes (Error Prone patch mode) and land them as
   reviewable per-package batches. Most baselines are majority-mechanical and could
   be halved in a week by someone who knows the tool.
4. Delete baseline entries as they are fixed, and promote the check to ERROR when
   its entry count reaches zero. A visible, shrinking artifact is what makes
   progress feel real; a static file is what makes it feel pointless.
5. Budget for the tail explicitly. Assume the last portion costs as much as the
   first, and say so, rather than discovering it.

---

### Wrong approach 5 — a blocking gate that is slow or flaky

**Wrong:** the full analysis suite — every tool, every check, plus a coverage
report — runs on every pull request as a required check.

**Exact symptom — what you would have SEEN:**

- Pull-request build time grows substantially. Engineers push and go do something
  else, so the feedback loop that made the gate valuable is gone; a violation is
  discovered twenty minutes after the context was lost.
- The analysis is occasionally flaky — a timeout, a resource limit, an
  order-dependent check — so "re-run the build" becomes a normal thing to say, and
  people stop reading failures carefully because failures are often noise.
- Somebody adds a skip label or flag "for urgent fixes". Within a couple of months
  it is used routinely, because everything feels urgent when the alternative is
  twenty minutes.
- Batching returns: engineers make larger changes to amortise the build wait, which
  makes review worse, which is the opposite of what the programme was for.
- The gate is measurably slowing delivery, and now someone with authority is
  looking at it — and the honest answer is that they are right.

**Root cause:** all checks were treated as one thing. They have very different
cost-to-value ratios, and the slow, occasionally-noisy ones dragged the fast,
reliable ones into disrepute.

**Fix:**
1. **Two tiers.** Fast and deterministic checks block the pull request:
   compilation, Error Prone, the formatter, ArchUnit, unit tests. Slow or
   probabilistic analysis runs on the main branch or nightly and files issues
   rather than blocking.
2. Set an explicit budget for the blocking tier — a number of minutes the team
   agrees to — and treat exceeding it as a bug in the build with an owner.
3. **Zero tolerance for flaky blocking checks.** A flaky required check must be
   fixed or demoted to the non-blocking tier the same week. One flaky check
   destroys the credibility of every reliable one, because it teaches people that
   red does not mean broken.
4. Cache aggressively and run analysis incrementally where the tool supports it.
5. If a skip mechanism exists, count its use and review the count monthly. A skip
   used routinely is a signal about the gate, not about the people using it.

---

## Artefact — what you must produce

Two documents. They are related and they are not the same length.

### Document A — `orderflow` code-review guideline

**Length:** two pages. Hard limit. A review guideline longer than two pages is not
read, and unread guidance is indistinguishable from none.

**Required sections:**

1. **What review is for** — three sentences, plus the explicit exclusion of
   anything the build checks.
2. **Service levels** — response time, change size, approver rules, comment labels
   (`blocking:` / `question:` / `nit:`), disagreement escalation.
3. **The reviewer's checklist** — organised by bug class, not by rule. **Every item
   must name the failure it prevents**, and at least six must cite a specific
   `orderflow` drill from Phases 5, 8, 9 or 11. Items a machine could check must be
   marked with the check that will replace them and the phase of the rollout plan
   in which that happens.
4. **The author's pre-review checklist** — short.
5. **Exceptions** — who approves shipping against the guideline, and the rule that
   silent exceptions are not acceptable.

### Document B — Static-analysis rollout plan

**Length:** 1,200–2,000 words plus tables.

**Required sections:**

1. **The case, from incidents.** The bug classes you have actually experienced,
   each with the drill or incident that produced it, and which check would have
   caught it. This is the funding argument and it goes first.

2. **Tool selection**, with the reason for each and what it sees that the others do
   not. Compile-time versus bytecode analysis, and why you need more than one.

3. **The Phase 0 measurement table** — every check, its existing violation count,
   its sampled false-positive rate, whether the fix is mechanical, and the related
   incident. Real counts from a real run, or the plan is speculative.

4. **The opening set** — the checks going to ERROR in phase 2, with their violation
   counts, and an explicit statement that these were chosen for **near-zero
   existing violations and near-zero false positives**, not for maximum yield.

5. **The ratchet, per tool** — which baseline mechanism, what "the number" is, and
   how the build fails when it increases. Prefer built-in ratchets and say when you
   are using one.

6. **The burndown** — by package and owner, with a target per iteration and a named
   owner for the mechanical automation. No percentages.

7. **The escape hatch** — local, visible, reasoned, counted, with an expiry
   enforced by a check.

8. **The build-time budget** — the blocking tier, the non-blocking tier, the minute
   budget, and the rule for flaky checks.

9. **The credibility plan** — one paragraph, and I want it explicitly. What is the
   first thing the team will notice, what will it cost them, and what is your
   answer if someone asks in week two what this has cost. If the honest answer to
   that question is anything other than "nothing yet", reconsider the opening set.

10. **What I am least sure about.**

### Constraints

- **Real counts.** The Phase 0 table comes from an actual run, not an estimate. A
  plan built on guessed violation counts is a plan whose central decision — which
  checks open — is unfounded.
- **No check enters the opening set on the strength of how valuable it looks.**
  Only on near-zero violations and near-zero false positives.
- **Every guideline item that a machine could check is marked** with its
  replacement check and rollout phase. The guideline is designed to shrink.
- **No global disable exists in the plan.** If you have written one, you have
  written the thing that will be used.
- **Every check that goes to ERROR removes an item from Document A.** State the
  correspondence explicitly; it is the evidence the programme is working.

---

## How I will review it

### The three questions that usually break a document of this kind

**Question 1 — "What is the existing violation count for each check you are
enabling, and what happens to the engineer who has to merge a hotfix on Friday?"**

This is the question the whole topic exists for, and it is the one that separates a
plan from an intention.

*How the document fails:*
- No counts at all. The plan proposes checks by name and value, having never run
  them. Every subsequent decision is unfounded.
- Counts exist and the opening set includes a check with a large count "because
  it's important". That is the freeze, with a plan attached.
- No escape hatch, or a global one. A global disable is the mechanism by which your
  gate is removed; including it in the plan is including its own defeat.
- The escape hatch exists and is not counted, so it becomes a second invisible
  baseline that grows forever.

*What a strong answer looks like:* real counts from a real run; an opening set
chosen explicitly for near-zero violations; and an escape hatch that is local,
visible in the diff, requires a reason, is counted on the burndown, and has an
expiry enforced by a build check. Plus the sentence I want most: *"in week two, the
honest answer to 'what has this cost us' is 'nothing', and that is the point."*

**Question 2 — "Your baseline has entries in it. Who lowers it, by when, and what
fails if they do not?"**

*How the document fails:*
- A baseline with no owner. This is the most common single defect and it produces
  Wrong Approach 4 with certainty.
- A ratchet that blocks increases and has no mechanism for decreases. Two of three
  properties; permanently stalled.
- A burndown expressed as a percentage. Nobody owns a percentage.
- No automation plan, so the burndown is manual, so it is slow, so it stops. Most
  baselines are majority-mechanical and the plan should say what fraction and how
  it will be patched.
- No expiry on suppressions, so the exceptions register grows monotonically.

*What a strong answer looks like:* a committed number that may only go down,
enforced by a check; a named owner per package; a target per iteration; the
mechanical fixes automated via patch mode and landed as reviewable per-package
batches; the proposing engineer doing the first two rounds themselves; and the
baseline entry deleted and the check promoted to ERROR when it reaches zero.

**Question 3 — "Which items in your review guideline could a machine check, and
why are humans still doing them?"**

This tests whether the two documents are actually connected or merely delivered
together.

*How the document fails:*
- The guideline contains formatting, naming or import-order items. Those are a
  tooling defect showing up as human work.
- The guideline contains a bug class that a check in the rollout plan covers, and
  the correspondence is not stated — so the guideline will never shrink, because
  nobody knows which line to remove.
- The guideline is long. Length is the symptom; the cause is that nothing has been
  automated away.
- Items are stated as rules ("always bound your queues") rather than as failures
  ("an unbounded queue converts overload into latency and then OOM — Topic 90").
  Rules are forgotten; failures are remembered.

*What a strong answer looks like:* each guideline item marked with either "machine:
`<check>`, phase `<n>`" or "human: requires intent"; a two-page limit; and the
explicit statement that the guideline shrinks as the rollout proceeds, with the
first few removals already identified.

### The other attacks, in order

- **"What does the blocking tier cost in minutes, and what is your flaky-check
  policy?"** A slow or flaky required check will destroy the programme and the
  plan usually does not mention build time at all.
- **"Which of these checks would have caught which of our incidents?"** If the
  answer is none, the funding case is theoretical and the plan will lose its first
  prioritisation conversation.
- **"What is your false-positive rate, and how did you measure it?"** If it was not
  sampled by hand, it was not measured.
- **"You have one attempt. What is the very first thing the team experiences?"** If
  the answer involves anyone fixing code they did not write, reconsider.
- **"What happens when the formatter reformats a file someone has an open branch
  on?"** If the plan includes a mass reformat, it includes a conflict storm and a
  destroyed `git blame`.
- **"Who reviews the exceptions register, and when?"** Unreviewed exceptions
  registers grow.
- **"What do you do if the team votes against a check?"** The plan should have an
  answer other than escalation (Topic 134).

### What I will not attack

Your choice of tools, your formatting conventions, or the specific contents of the
review checklist beyond whether each item names a failure. I have no stake in
whether you use SpotBugs or something else. I am attacking the rollout's cost
distribution — who pays, when, and whether they created the problem they are being
asked to fix.

---

## Interview questions (Senior → Principal)

> The rubric line from the master plan:
> *"we should enable NullAway" → "on new code only, with the existing violations
> baselined and the baseline ratcheting down. A gate that fails 4000 existing
> warnings gets turned off in a week, and then we've spent our credibility."*

---

### Q1 — "You want to introduce static analysis on a ten-year-old Java codebase. How do you do it?"

**A senior answer sounds like:** "I'd start by running the tools in report-only
mode to see how many violations we have. Then enable them gradually rather than all
at once — probably start with warnings, fix the worst categories, and only make it
a build failure once we're close to clean. I'd get the team's buy-in first and
choose checks that catch real bugs rather than style issues."

That is a good answer. Report-only first, gradual severity, real bugs over style,
buy-in. Most people do not get to "report-only first".

**A principal answer sounds like:** "The technical rollout is straightforward. The
thing I'd design around is that I get **one attempt**. If this gate blocks someone's
hotfix in week one, it gets disabled with a build flag, that flag lives in the
build file for two years, and the next check I propose — even a free one — gets
'is this going to be like last time?'. So the first phase's job is not to catch
bugs. It is to demonstrate that gates are cheap.

Concretely. Phase zero is measurement: every check, its existing violation count,
and a hand-sampled false-positive rate. That table decides everything and it takes
a couple of days.

Phase one is the formatter, ratcheted to changed files only — no mass reformat,
because that destroys `git blame` and conflicts with every open branch. It is
unarguable, it has no false positives, and it removes a whole category of comment
from review. Shipped alone, so its cheapness is visible.

Phase two is the opening gate, and the selection criterion is the part people get
backwards: I pick checks with **near-zero existing violations and near-zero false
positives**, not the ones with the highest yield. Fix the handful that exist in the
same pull request. Now a class of bug cannot be merged and nobody's build broke. If
someone asks in week two what this cost, the answer is 'nothing', and that answer
is the asset.

Phase three is where the big counts get handled, and never globally at ERROR.
New-and-changed-code only, with a ratcheting baseline — and I'd use the tools' own
ratchets rather than building one. NullAway is configured per annotated package, so
you turn it on one package at a time; I'd start with payments, because a null
defect there costs the most. SpotBugs baselines through an exclusion filter file.
ArchUnit has a violation store. The formatter has a git ratchet. Built-in ratchets
are maintained by someone else and don't rot.

Phase four is the property that stops the baseline being permanent: the number is
committed and **may only go down**, enforced by a check. Otherwise you've built two
of the three ratchet properties, the pain of new violations stops, and the baseline
looks exactly the same in two years.

And phase five is the burndown — automated where the tool has a patch mode, landed
as reviewable per-package batches, published as a table of packages and owners
rather than a percentage, and I do the first two rounds myself. The whole
programme's arithmetic is: make following cheaper than ignoring. If I hand a team
a red build and a list, ignoring is cheaper. If I hand them a green build and a
diff, following is cheaper.

One more thing that is not about tools: every check that goes to ERROR deletes a
line from the code-review guideline. The guideline is supposed to shrink, and that
shrinkage is how the team can see the programme working."

**What separates them:** five things.

1. The senior answer optimises the rollout for correctness. The principal answer
   optimises it for **credibility**, treats it as a finite budget, and designs the
   first phase to cost nothing rather than to catch the most.
2. The senior answer says "gradually". The principal answer names the **specific
   ratchet mechanism per tool**, and prefers the ones built into the tools —
   which is Java-specific knowledge, not a general principle.
3. The senior answer plans to enable at ERROR "once we're close to clean". The
   principal answer knows you will never get close to clean before enabling,
   because the burndown is only funded once the gate exists.
4. The principal answer names the third ratchet property — the number may only go
   down — which is the one everyone omits and the one that determines whether the
   baseline is temporary or permanent.
5. The principal answer connects the gate to the review guideline and makes the
   guideline shrink, which converts a tooling change into a visible reduction in
   human work.

**Adversarial follow-up:** *"Your phase two catches almost nothing. You've spent a
month for a formatter and three checks. Why is that a good use of your time?"*

The honest answer accepts the premise rather than arguing with it. Phase two is not
where the value is; it is the entry fee for phases three to five, which are where
the value is, and which are unavailable if phase two goes wrong. The comparison is
not "a month for three checks" versus "a month for forty checks" — it is "three
checks and a working programme" versus "forty checks for one week and no gates for
a year". I would also point out that the formatter is not nothing: it removes
formatting from every review permanently, which is a real recurring saving across
the team, and it is measurable if anyone wants to count review comments before and
after. And if the organisation genuinely will not fund the sequence, that is worth
knowing early — because then the right answer is a much smaller programme, not the
same programme done riskily.

---

### Q2 — "What should a code review actually check?"

**A senior answer sounds like:** "Correctness first — does it do what it's meant
to, are the edge cases handled, are there tests. Then design: is it maintainable,
does it fit the existing architecture. Style should be automated so it doesn't come
up. And review is also how knowledge spreads through the team."

That is a good answer and it contains the right exclusion.

**A principal answer sounds like:** "Three things, and the useful part is what
falls out of each.

Correctness a machine cannot check — which means intent. Does this do what the
ticket says, is the transaction boundary the right one for the invariant, is the
failure mode acceptable. No analyser has intent, so this is irreducibly human.

Design and reversibility. Is this shape going to be expensive to change, does it
create a coupling we'll regret. This is where a senior reviewer's time is actually
worth something, and it's the part most reviews skip because it requires reading
beyond the diff.

And knowledge transfer, which is the benefit that survives even when the review
finds nothing — it's why review isn't optional on low-risk changes.

Everything else belongs to a machine, and I'd be explicit that it's a **tooling
defect** when a human does it. Formatting, import order, obvious bug patterns,
banned APIs, architectural boundaries — if those appear in review, file a bug
against the build.

The Java-specific part is that the human list is long and non-obvious, and I'd
write it as bug classes with the failure named rather than as rules. Not 'bound
your queues' but 'an unbounded queue turns overload into latency and then OOM — we
have the drill'. Not 'be careful with `@Transactional`' but 'a checked exception
thrown from a `@Transactional` method commits the write, and that's the most common
silent data bug in Spring codebases'. Same for self-invocation bypassing the proxy,
a mutable field in a `hashCode`, a `ThreadLocal` not removed on a pooled thread,
`synchronized` around I/O with virtual threads on, and an unbounded value in a
metric tag. Rules get forgotten; failures get remembered, especially when the
reviewer watched that failure happen.

Two mechanics I'd insist on. Comment labels — `blocking:`, `question:`, `nit:` —
with `nit:` never blocking, because most review friction is ambiguity about whether
a comment must be addressed. And a two-page limit on the guideline, with every
machine-checkable item marked for deletion when its gate lands. The guideline
shrinking over time is the evidence that the automation programme is real."

**What separates them:** the senior answer states the right categories. The
principal answer gives the **test** that assigns an item to a category (does it
require intent), names the human list as bug classes with their failures rather
than as rules, treats a style comment in review as a bug in the build rather than a
lapse in discipline, and adds the two small conventions — comment labels and the
length limit — that do more for review culture than the content does.

**Adversarial follow-up:** *"Your reviewers don't know most of those Java bug
classes. The checklist doesn't help someone who can't see the bug."*

That is correct and it is the strongest objection to a checklist. A checklist
raises the ceiling for someone who knows the class and does nothing for someone who
does not. Three responses, in order of value: first, convert every item you can
into a gate, because a gate works regardless of who is reviewing — that is what the
rollout plan is for and it is why the two documents are one artefact. Second, for
the items that stay human, the checklist item should link to the drill, so the
reviewer who does not know it has a two-minute path to seeing the failure rather
than a rule to memorise. Third — and this is the honest limit — some of these
require someone who has been bitten, which is why the review policy should route
changes in the contended areas (the order-placement transaction, the inventory
decrement, anything touching pools or executors) to a reviewer who knows them. That
is not a checklist; it is a routing rule, and it belongs in the guideline's service
levels section.

---

### Q3 — "How do you deprecate an internal API across five teams?"

**A senior answer sounds like:** "Announce the deprecation with a timeline, mark it
`@Deprecated`, provide a migration guide, and give teams a reasonable window. Then
follow up before the removal date and help anyone who's behind."

Reasonable and it will move some teams.

**A principal answer sounds like:** "The announcement is the least important part.
The question is whether migrating is cheaper than not migrating, and by default it
is not — for a team with a product deadline, migrating is pure cost with no felt
benefit, and they will deprioritise it correctly, every quarter, forever. That is
the same stall as any migration.

So: make it cheaper. Ship the replacement first and make it obviously better. Write
the migration as a tool — an OpenRewrite recipe if the change is mechanical, which
internal API renames usually are — and then **run it for them and open the pull
request**. Now their cost is reviewing a diff. Migrate one team's service myself,
so there is a worked example and I know what the tool misses.

Then make ignoring expensive, and Java gives you the mechanism directly:
`@Deprecated(since=…, forRemoval=true)` makes javac emit a removal warning, and
`-Werror` turns it into a build failure. So the ratchet is the same shape as any
gate — warn, then fail on modules that change, then fail outright, then delete the
API. And the point of the deadline is not to punish; it is to make 'not migrating'
a decision someone has to actively make and be seen making, instead of a default
that happens through inaction. Exceptions go through a named approver with an
expiry, and they appear on the burndown as exceptions so they are visible rather
than quiet.

And the burndown lists teams and call sites, not a percentage. A row with a name
that hasn't changed in three weeks produces a specific conversation. A percentage
produces a slide.

The last thing, and it is the one that decides whether the second deprecation goes
well: the team that pushes back hardest usually has a real constraint I have
missed — a use case the replacement does not cover, a version they cannot move
from. Finding that out early changes the design, and it is also why the other
teams trust the migration. If I treat their objection as resistance rather than as
information, I get compliance from four teams and an enemy on the fifth, and next
time nobody tells me anything early."

**What separates them:** the senior answer runs a communication process. The
principal answer changes the **arithmetic** — builds the tool, runs it for people,
uses Java's own deprecation-and-removal machinery as the ratchet — and treats the
dissenting team as a source of design information rather than an obstacle. The
distinction between "punish" and "make the decision visible" is also a Principal-level
framing: a deadline's function is to convert inaction into a decision with an owner.

**Adversarial follow-up:** *"One team has a legitimate reason they can't migrate,
and it will take them two quarters. Do you slip the removal date for everyone?"*

Not necessarily, and the choice depends on what the old API costs to keep. If
maintaining it is cheap — it compiles, it is stable, nobody is paying much — then
keeping it for that one team behind a documented, dated exception is fine, and it
costs the other four teams nothing. If maintaining it is expensive, or if its
existence is what is blocking something else (a framework upgrade, a boundary you
are trying to enforce), then the cost of the exception is real and someone should
be making that trade explicitly rather than me absorbing it quietly. Either way, the
mistake would be slipping the date silently for everyone, because that teaches the
four teams that moved on time that moving on time was unnecessary — and the next
deadline will be met by nobody. Publish the exception, name its owner, name its
expiry, and keep the date for everyone else.

---

### Q4 — "The team pushes back on a check you want to enable. What do you do?"

**A senior answer sounds like:** "I'd listen to their concerns and try to
understand the objection. If it's a false-positive problem I'd tune the
configuration. If they still don't want it, I'd probably drop it rather than force
it — a check the team resents won't survive anyway."

Good instincts, and the last sentence is genuinely true.

**A principal answer sounds like:** "First I'd want to know which objection it is,
because there are three and they need completely different responses.

'This check has too many false positives.' They are right and I should thank them,
because a noisy check is worse than no check — it teaches people to scroll past
build output, which damages every check I have. The response is to measure the rate
honestly, tune or drop the check, and say publicly that I dropped it. Dropping a
check because the team was right is one of the cheapest ways to buy credibility for
the next one.

'This will slow us down.' Usually a prediction about the rollout rather than the
check. If they are imagining a red build and a list of pre-existing violations,
they are objecting to something I am not proposing, and the answer is to show the
new-and-changed-code scoping and the count from phase zero. If they are right —
if it will genuinely slow things down — then either the scoping is wrong or the
check is not worth it.

'We disagree that this is a bug.' This is the interesting one, and it is a real
signal. If a competent engineer thinks the pattern is fine, either they are missing
something or I am. Sometimes the check is enforcing a preference wearing a bug's
clothes, and gating a preference is how you get an argument about the gate instead
of about the code. Sometimes there is a legitimate use case that needs a documented
suppression pattern. Either way I'd want the specific example rather than the
general objection.

What I would not do is escalate. A check imposed against the team's judgment gets
worked around — a wrapper, a suppression pattern, a habit of not touching those
files — and I have spent authority to get compliance rather than adoption. The
compliance is worth less and the authority does not come back.

The thing that usually resolves it: adopt for one package rather than the whole
codebase, run it for a few weeks, and look at what it actually caught and what it
actually cost. Most of these disagreements are about predictions, and a small
experiment converts a prediction into evidence more cheaply than an argument
does."

**What separates them:** the senior answer treats pushback as a single thing to be
listened to. The principal answer **partitions it into three objections with three
different correct responses**, recognises that one of them is evidence the check is
wrong, refuses escalation on the grounds that compliance is worth less than
adoption, and converts the disagreement into a bounded experiment. Publicly
dropping a check because the team was right — treating that as a credibility gain
rather than a loss — is the move that most distinguishes the levels.

**Adversarial follow-up:** *"The check would have caught a production incident last
quarter, and the team still says no. Do you overrule them?"*

That is the case where I would push hardest, and I would still not overrule. The
incident is strong evidence and it changes the conversation, so I would put it in
front of them directly — this check, this incident, this line of code — and ask
what they would do instead, because "no" to my proposal is not "no" to the problem.
Very often the answer is a different, better check, or a test, or a bounded-queue
factory that makes the bug unwritable, and that outcome is superior to mine. If
they still say no with no alternative, then the right move is to write down the
disagreement — the incident, the proposed check, the objection, the decision, and
who made it — and let it stand. If the class recurs, the record does the arguing
for me and it does it much better than I could, because by then it is evidence
rather than prediction. Overruling gets me the check and costs me the next five
conversations.

---

### Q5 — "How do you know your standards programme is working?"

**A senior answer sounds like:** "The violation count goes down, the checks stay
enabled, and we see fewer of those bug classes in production. I'd also expect code
review to get faster and more consistent."

Right dimensions.

**A principal answer sounds like:** "Four signals, and one of them is the one I'd
actually watch.

The baseline is going down. Not 'has been baselined' — going down, with a table of
packages and owners, and the trend is what matters rather than the level.

No global disables exist in the build. That is a binary and it is the honest test of
whether the rollout was designed correctly. One global skip flag means the gate
failed and nobody said so.

The review guideline is getting **shorter**. Every check that reaches ERROR removes
a line. If the guideline is the same length after six months, nothing has actually
moved from human to machine, which was the whole point.

And the one I'd watch above the others: **what happens when someone proposes the
next check.** If the response is 'sure, what does phase zero say' then the
programme worked, because the organisation now believes gates are cheap. If the
response is a wince, it did not, regardless of what the violation counts say. That
is a cultural reading rather than a metric, but it is the one that predicts whether
any of this survives me leaving.

Two leading indicators I'd also track: the number of style comments in code review,
which should go to approximately zero after the formatter and stay there; and the
recurrence rate of the specific bug classes the checks target, which is the only
thing that connects this programme back to the incidents that justified it. If we
enabled a check for a class we had an incident in, and that class recurs, either
the check does not cover the real case or the recurrence is in the grandfathered
region — and both of those are actionable findings rather than disappointments."

**What separates them:** the senior answer measures the programme's outputs. The
principal answer adds two things: a **binary failure test** (does a global disable
exist), and a **cultural leading indicator** (what happens when the next check is
proposed) which is the only one that predicts durability. The guideline-shrinking
metric is the connection between the two artefacts and is the clearest evidence
that human work was replaced rather than added to.

**Adversarial follow-up:** *"Bug-class recurrence hasn't moved. Was the whole thing
a waste?"*

Not automatically, and the diagnosis matters more than the verdict. Three
possibilities and they have different answers. The check may not cover the real
case — the incident's pattern is subtly different from what the check matches,
which is a finding and a fixable one. The recurrence may be in the grandfathered
region, which means the ratchet is working exactly as designed and the burndown is
too slow, which is a funding question. Or the check may genuinely not be worth its
cost, in which case I should say so and remove it, because carrying checks that do
not pay is how the whole set loses credibility. What I would not do is defend the
programme on its process metrics — a shrinking baseline that does not reduce
incidents is a well-run activity, not a result, and being able to say that about my
own initiative is most of what makes the next one trustworthy.

---

## Mental model checkpoint

Reason these out in writing.

1. The mechanical statement says a standard is adopted when following it is
   cheaper than ignoring it. Take one standard your current team has that is *not*
   followed, and compute both costs honestly. What is the smallest change that
   flips the inequality?

2. A ratchet needs three properties. Name a mechanism you have used that had only
   two, say which one was missing, and predict — from the mechanic, not from
   memory — what happened over the following year.

3. Error Prone runs inside javac, so a violation at ERROR is a compile failure.
   SpotBugs runs on bytecode after compilation. Explain how that difference changes
   the rollout design for each, and which one is riskier to enable aggressively.

4. NullAway's `AnnotatedPackages` configuration is described as the cleanest ratchet
   in the Java ecosystem. Say precisely why, in terms of who pays the cost and
   when. Now: what is the cost of that design — what does per-package adoption make
   harder?

5. You have one credibility attempt. Construct the opening set of checks for
   `orderflow` from the drill history in Example 2, and justify each choice on
   violation count and false-positive rate rather than on value. Which
   high-value check did you deliberately leave out, and when does it enter?

6. Every check promoted to ERROR removes a line from the review guideline. Work out
   what the guideline converges to if the automation programme runs indefinitely.
   Is that a good end state? What is irreducibly human?

7. A check would have caught a production incident, and the team votes it down.
   Argue both sides properly, then say what you actually do — and what you do
   differently if it is the *second* time this class has caused an incident.

---

## Quick reference card

### The two costs

| | Cost of following | Cost of ignoring |
|---|---|---|
| Wiki page | low | **zero** → ignored |
| Review comment | low | low → inconsistent |
| Formatter in build | **zero** | build fails → universal |
| Gate on new code | proportional to your change | build fails → adopted |
| Gate on whole legacy codebase | **unbounded, unrelated to your change** | build fails → **switched off** |

### The three ratchet properties — all required

1. A **recorded baseline**, committed to the repository.
2. A **gate that fails when the number goes up**.
3. A **mechanism that lowers it**, with an owner and a schedule.

Two out of three is a permanently split codebase.

### Tools and their baseline mechanisms

| Tool | Runs | Sees | Baseline / ratchet |
|---|---|---|---|
| Error Prone | inside javac | source + full types | per-check severity OFF/WARN/ERROR; patch mode for burndown |
| NullAway | Error Prone plugin | nullness via annotations | **`AnnotatedPackages` — opt in per package** |
| SpotBugs | after compile | bytecode | exclusion filter XML, committed |
| Spotless | build | formatting | **`ratchetFrom <git ref>`** |
| ArchUnit | test phase | package/class structure | violation store / freeze |
| Sonar-style platform | CI | aggregate | new-code period |

Verify current configuration syntax against each tool's own docs before writing it
into a plan.

### The rollout, in order

0. **Measure** — counts and sampled false-positive rates. Days.
1. **Formatter**, ratcheted to changed files. No mass reformat.
2. **Opening gate** — near-zero violations, near-zero false positives. Costs nothing.
3. **New-and-changed only** for the big counts, using each tool's own ratchet.
4. **The number may only go down**, enforced.
5. **Burn down** — automated where possible, by package and owner, no percentages.
6. **Promote to ERROR globally, delete the baseline entry, delete the guideline line.**

### The escape hatch — four properties

Local (not global) · Visible in the diff · Requires a reason · Counted, with an
enforced expiry.

If you do not design one, a global one will be invented at 5pm on a Friday.

### Review guideline — the shape

- **Two pages, hard limit.**
- Three purposes: intent-level correctness, design and reversibility, knowledge
  transfer.
- Everything mechanical is excluded by rule; a style comment in review is a bug in
  the build.
- Items written as **bug classes with the failure named**, not as rules.
- Comment labels: `blocking:` / `question:` / `nit:` — `nit:` is never blocking.
- Every machine-checkable item marked with its replacement check and phase.

### `orderflow` review checklist — the six that came from drills

| Check for | Failure it prevents | Topic |
|---|---|---|
| Unbounded queue or pool | overload → latency → OOM | 90 |
| Checked exception from `@Transactional` | the write **commits** | 54 |
| Network call inside a transaction | pool exhaustion across all endpoints | 55 |
| `ThreadLocal` not removed on a pooled thread | permanent retention | 79b |
| `synchronized` around I/O with virtual threads | carrier starvation (pinning) | 101 |
| Unbounded value in a metric tag | cardinality kills the monitoring system | 118 |

---

## When would I use this at work?

**1. In the first month on a new team, before you have opinions worth imposing.**
Read the last quarter's incidents, ask which bug classes recur, and write the
review guideline from *that* rather than from your preferences. It is one of the
few artefacts a newcomer can produce that is immediately useful and not
presumptuous, because every item is sourced from something the team already
experienced. It also teaches you the codebase faster than reading it would.

**2. As the second half of every postmortem action item.** Topic 133 will teach you
that an action item must prevent a class rather than an instance. This topic is
where you find out what is *available* to prevent a class: a gate is stronger than
a checklist item, a wrapper that makes the bug unwritable is stronger than a gate,
and knowing which of the three is achievable for a given bug class is the
difference between an action item that works and one that reads well. "Add a review
checklist item" is what you write when you do not know this topic.

**3. Any time you are about to become a human linter.** The moment you notice you
are leaving the same review comment for the third time, stop and ask whether a
machine can make it. If it can, that is a two-hour piece of work that removes the
comment forever, for everyone, including on the days you are not reviewing. If it
cannot, it belongs in the guideline so that other reviewers catch it too. Either
way, the answer is not to keep leaving the comment — that is a role that ends when
you go on leave, and it does not scale past you.

---

## Connected topics

**Prerequisites — the bug classes your checklist and your checks are built from:**

- **Topic 01 — boxing and the Integer cache.** The canonical near-zero-violation,
  zero-false-positive opening check.
- **Topic 13 — the `equals`/`hashCode` contract.** The mutable-key failure, which is
  silent data loss and is a review item that no analyser fully covers.
- **Topics 40, 54, 55 — proxying, `@Transactional`, the connection pool.** Three of
  the highest-value human checklist items, and the drills that justify them.
- **Topics 49–50 — lazy loading and N+1.** Review items with a testing counterpart:
  a query-count assertion is a gate.
- **Topics 79, 79b — leaks and `ThreadLocal` retention.** Checklist items that
  come directly from a heap dump you took.
- **Topic 90 — pool sizing and bounded queues.** The best example of a bug class
  where the strongest fix is neither a gate nor a checklist but a **factory that
  makes the bug unwritable**.
- **Topic 94 — lock ordering**, **101 — virtual-thread pinning**, **92 — atomic
  composition.** Concurrency review items that require a reviewer who has been
  bitten, which is why the guideline needs a routing rule as well as a checklist.
- **Topic 118 — metric cardinality.** A review item that protects a system other
  than the one being reviewed.
- **Topic 124 — the readiness review.** Its observability and risk sections are
  where several checklist items originate.

**This connects to:**

- **Topic 127 — migration planning.** The CI ratchet is the same mechanism, and the
  stall has the same cause. Read the two forcing-function sections together.
- **Topic 128 — monolith to services.** The ArchUnit boundary rule with a violation
  store is the first step of the extraction, and it is this topic's machinery.
- **Topic 131 — design docs.** The review guideline states when a change needs one;
  the rollout plan is itself a small design doc.
- **Topic 133 — postmortems.** Every action item ends up here, and this topic
  determines which of gate / wrapper / checklist / documentation is available.
- **Topic 134 — influence without authority.** Making adoption cheaper than
  non-adoption is the whole mechanic of a successful rollout, and the credibility
  budget is the currency both topics spend.
- **Topic 135 — the capstone.** "Tell me about a standard you introduced" is a
  standard principal prompt, and the follow-up is always "what did it cost the
  team in week one".

---

*Java baseline 21, running on JDK 25, Spring Boot 4.1 / Framework 7.0. The
mechanics in this topic — the two costs, the three ratchet properties, the
credibility budget — are properties of organisations and are stable. The tools are
not: check names, configuration keys and available checks change between versions.
Verify against `errorprone.info`, `github.com/uber/NullAway`,
`spotbugs.readthedocs.io`, `github.com/diffplug/spotless` and `archunit.org` for
the versions you pin before any of it goes into a plan with your name on it. The
"clean as you code" new-code-gate idea is a widely used industry practice, best
known from SonarQube; nothing here depends on a proprietary method.*
