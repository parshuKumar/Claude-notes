# 127 — Migration Planning I: Java 8 → 21/25, Boot 2 → 3 → 4

## Phase: 12 — Principal Track
## Category: ELITE
## Java baseline: 21  |  Notes features from: 21/25
## Project spine: a sequenced migration plan for a hypothetical 400k-line Boot 2.7 / Java 8 estate — with a rollback point at every step and a mechanism that stops it stalling at 60%

---

## Mechanical statement

> **A migration succeeds when each step is independently valuable and
> independently revertible. A migration stalls when the remaining work stops
> being painful enough to prioritise.**

Both halves are load-bearing, and the second half is the one people get wrong.

**Independently valuable** means: if the migration is cancelled the day after this
step lands, the organisation is still better off than it was. Not "we made
progress toward the goal" — better off, in a way you can name.

**Independently revertible** means: if this step causes a production problem, you
can put the system back the way it was, in minutes, without a code change and
without a data migration. Not "we can revert the commit". Reverting a commit is
not a rollback if the schema has moved on.

**And then the stall.** Every large migration has the same shape of failure. The
first services to move are the ones that hurt most: the ones with the CVE, the
ones where the team was already blocked. They move fast, and moving them removes
the pain that motivated the programme. What is left is the 40% of the estate that
nobody was suffering from — the batch job with no owner, the internal admin tool,
the service that depends on a library whose last release predates the rename. For
those, migrating has a cost and no benefit anyone can feel. So it does not get
prioritised. Not once, not in any quarter. And the migration sits at 60%
indefinitely, which is the most expensive possible state, because now you are
paying to operate two baselines, two build configurations, two sets of runbooks,
and two CVE-patching pipelines forever.

The plan you write in this topic is judged on one thing above all others: does it
contain a **mechanism** — something that runs without anyone deciding to run it —
that makes the last 40% happen? A request for cooperation is not a mechanism. A
quarterly reminder is not a mechanism. A build that fails on a date is a
mechanism.

---

## The bridge from what you know

### What transfers — completely, and I will not re-teach it

You have run platform upgrades in TypeScript and Node. The following transfer 1:1
and should not consume a line of your artefact:

- **Staged rollout, canary, percentage deploys, feature flags.** Same technique,
  same reasoning about blast radius. You know this.
- **Batch size as the primary risk control.** You already believe that a hundred
  small pull requests beat one enormous branch, and you already know why: a large
  batch cannot be reviewed, cannot be bisected, and cannot be partially reverted.
- **Getting buy-in for platform work that ships no features.** You have made this
  argument to a product manager. Nothing about it is Java-specific.
- **Codemods.** You have run `jscodeshift` or a `ts-morph` script across a
  repository. The *idea* of automated refactoring transfers whole. What does not
  transfer is what the Java equivalent can do, which is covered below.
- **Reading a changelog for breaking changes and mapping them to your code.**
  Same skill.
- **Deprecation as a communication practice.** You have deprecated an internal
  API before. The practice transfers; the enforcement mechanics are Topic 132.

### What does not transfer — and this is most of the topic

| You know (Node/npm) | Java/JVM reality | Verdict |
|---|---|---|
| Every Node major gets an LTS line; you upgrade every couple of years and the cadence is uniform | **Java has a 6-month release train and an LTS designation on some of them.** 21 (Sept 2023) and 25 (Sept 2025) are LTS; 22, 23, 24, 26 are not. A non-LTS release receives updates for roughly six months and then stops | **NO ANALOGUE.** The upgrade calendar is not yours; picking a non-LTS target is picking a six-month support window. |
| "Node 18 is supported until date X" — one number, from one place | **Support is per-vendor, not per-version.** Oracle, Eclipse Temurin (Adoptium), Amazon Corretto, Azul Zulu, Red Hat and Microsoft each publish their own end-of-support date for the same JDK version, and they differ | **NO ANALOGUE.** "Java 21 is supported until…" is an incomplete sentence. You must name the vendor. |
| A package published for Node 14 usually just runs on Node 20 | **A jar compiled for Java 8 usually just runs on JDK 21.** Class-file compatibility is genuinely strong | **HONEST ANALOGUE, and it is the single fact the whole plan is built on.** It is why you can upgrade the JDK without upgrading the framework. |
| Two versions of a package can coexist in `node_modules` | **Maven puts one version of an artifact on a flat classpath.** You cannot run `javax`-era and `jakarta`-era code side by side in one application | **NO ANALOGUE** — and it is why the rename cannot be done incrementally within one deployable unit. |
| `jscodeshift` operates on a syntax tree with no type information | **OpenRewrite operates on a *type-attributed* Lossless Semantic Tree** — it knows that `DataSource` in this file resolves to `javax.sql.DataSource` (a JDK type) and not to a Jakarta type | **PARTIAL** — and the difference is exactly what makes a 400k-line rename mechanical instead of a six-month manual slog. |
| Upgrading the runtime does not change your garbage collector, because you do not have one to change | **JDK 8's default collector is Parallel; JDK 9 onward it is G1.** A "no code change" JDK upgrade changes your latency profile | **NO ANALOGUE** — the runtime upgrade is a performance change even when it is not a behaviour change. |
| A removed API fails at `npm install` or at first import | **A removed JVM flag makes the JVM refuse to start.** `-XX:+UseConcMarkSweepGC` on JDK 14+ is a hard startup failure, not a warning | **PARTIAL** — the failure is loud, which is good, but it is at container start in production if you did not test the flags. |

### The four Java-specific facts the whole plan hangs on

**1. Class files are forward-compatible; source is not.**
A jar compiled targeting Java 8 bytecode runs on JDK 21 in the overwhelming
majority of cases. This is why you can change the *runtime* without changing a
line of source. It is the foundation of the entire sequencing argument, and it
has no equivalent in the Node world where the runtime and the code are much more
tightly coupled by the module system. The exceptions — and you must enumerate
them for your own estate — are code that reflects into JDK internals, code that
parses the `java.version` string, and bytecode-manipulation libraries (ASM,
cglib, Byte Buddy, older Mockito, older Lombok) that refuse to read a class-file
version they do not recognise.

**2. Strong encapsulation is now the default, and it is a runtime failure.**
Since JDK 16, illegal reflective access into JDK internals is denied rather than
warned about. Code that did `setAccessible(true)` on an internal field now throws
`InaccessibleObjectException`. The escape hatch is `--add-opens` on the command
line, which works and is the correct short-term mitigation, and which you should
treat as a debt register entry rather than a solution. Note that `sun.misc.Unsafe`
memory-access methods are on a deprecation-and-removal path across recent
releases; if anything in your estate uses it directly (or a library does), that
is a blocker to find in step 0, not in month nine.

**3. `javax` → `jakarta` is mechanically simple, vast in scope, and all-or-nothing
per deployable unit.**
When Java EE moved to the Eclipse Foundation, trademark constraints forced a
namespace rename: `javax.persistence` became `jakarta.persistence`,
`javax.servlet` became `jakarta.servlet`, `javax.validation` became
`jakarta.validation`, and so on across the entire enterprise API surface. Spring
Boot 3 requires the `jakarta` namespace. There is no compatibility shim in the
supported path — the flat classpath means you cannot have both. So within a
single deployable artifact, this is not an incremental change: the module and
every library it uses move together, or the module does not move.

**And here is the detail that catches every team that reaches for `sed`:** not
every `javax` package is a Jakarta package. `javax.sql.DataSource`,
`javax.crypto`, `javax.naming`, `javax.management`, `javax.net.ssl`,
`javax.security.auth` and others are **JDK** APIs. They were never Java EE and
they are still `javax` today and will be forever. A regex rename breaks them. A
type-aware tool does not, because it knows which classpath entry the symbol
resolved from. This single distinction is the clearest possible argument for
using OpenRewrite rather than a shell script, and it is the argument I want to
see you make in one sentence in your plan.

**4. OpenRewrite is the tool that makes 400k lines tractable.**
OpenRewrite is an open-source automated-refactoring engine for the JVM ecosystem,
maintained by Moderne. Mechanically: it parses your source into a **Lossless
Semantic Tree** — an AST that preserves formatting *and* carries full type
attribution from the compiled classpath — then applies **recipes**, which are
composable transformations, and prints the tree back out with your formatting
intact. It runs as a Maven or Gradle plugin (`mvn rewrite:run`), so it runs in CI
and it runs on a branch you can review.

What matters for the plan:

- There are published, maintained recipes for the exact migrations you are doing:
  a Java-version upgrade recipe, a `javax`→`jakarta` migration recipe, and Spring
  Boot version-upgrade recipes. You do not write these; you run them and review
  the diff.
- A recipe does more than rename. The Boot upgrade recipes also update property
  keys in `application.yml`, update `pom.xml` dependency coordinates, and apply
  known API migrations.
- It is **not** a complete answer. Expect it to do the overwhelming bulk of the
  mechanical change and leave you the genuinely semantic parts: your own
  abstractions over the framework, anything reflective, anything string-based,
  and any dependency that has no `jakarta` release at all. Those last ones are
  the blockers, and they are the reason for step 0 in the plan below.
- Because the output is a normal diff on a normal branch, every OpenRewrite step
  is reviewable and revertible in the ordinary way. That property is worth a lot
  in this plan.

> **One line of honest uncertainty:** I am confident about what OpenRewrite is and
> how it works. I am not going to quote a specific recipe coverage percentage or
> a specific recipe ID here, because those move. Get the current recipe catalogue
> from `docs.openrewrite.org` before you write it into a plan with your name on it.

---

## What is this?

A migration plan is a **sequencing document**. Its subject is not "how do we
upgrade" — that part is mostly known and mostly mechanical. Its subject is:

- **in what order**, and why that order and not another;
- **at what granularity**, so that each unit of work can succeed or fail alone;
- **with what rollback**, stated as an action someone on call can take at 3 a.m.
  without a code change;
- **with what forcing function**, so that the boring 40% happens.

It is not a project plan with dates for their own sake. Dates appear in it in
exactly one place that matters: on the deprecation deadline, because that date is
the mechanism.

### The estate, defined

The artefact assumes a hypothetical estate. Make it concrete, because a plan
written against "a big codebase" is a plan that has not been tested against
anything:

- ~400,000 lines of Java.
- Spring Boot 2.7, Java 8 source and target, running on a JDK 8 base image.
- Multiple deployable services plus a shared internal library set (a "platform"
  or "commons" set of jars that most services depend on).
- Some services actively developed; some not touched in a year; at least one with
  no clear owning team.
- A CI system, a shared parent POM or Gradle convention plugin, and a container
  base image that someone controls.

Those last three are not incidental. **The shared parent POM, the CI pipeline and
the base image are where the mechanism lives.** A migration plan for an estate
with no shared build configuration has a different — and much worse — stall story,
and if that is your situation, creating the shared configuration is step 0.

### The three things being migrated, and why they are three

People say "we're upgrading" as if it were one thing. It is three, on three
different risk profiles:

| Change | What it is | Risk shape | Revertible by |
|---|---|---|---|
| **The JDK** | The runtime your bytecode executes on | Performance and reflective-access risk; almost no source risk | Changing the base image tag and redeploying |
| **The namespace** (`javax`→`jakarta`) | A mechanical rename across the entire enterprise API surface | Compile-time and dependency-availability risk; near-zero behavioural risk | Redeploying the previous artifact |
| **The framework** (Boot 2 → 3 → 4) | Behavioural changes across Spring, Hibernate, Security, Jackson | Behavioural and performance risk, spread over hundreds of small semantics changes | Redeploying the previous artifact — *if* the schema and the message contracts allow it |

Collapsing these into one step is the most common structural error in migration
planning, and it is a serious one: when p99 moves, you cannot say which of the
three did it. You have destroyed your own ability to diagnose. Topic 65 gave you a
baseline precisely so you could attribute a change to a cause; changing three
things at once throws that away.

### The dependency chain that forces the order

The order is not a preference. It is forced by three facts:

1. Boot 3 requires Java 17 or later.
2. Boot 3 requires the `jakarta` namespace.
3. Boot 2.7 runs on a Java 8-era codebase *and* on a modern JDK.

Fact 3 is the gift. It means the JDK upgrade can be done first, alone, with the
framework held constant — which makes it independently valuable (security
patches, a better collector, container awareness, and the JDK-level performance
work of the last decade) and independently revertible (change the base image
back). Facts 1 and 2 mean the namespace change and the framework change are
**one step**, because you cannot run `jakarta` code on Boot 2.7 and you cannot run
Boot 3 on `javax` code. That is worth stating plainly in your plan, because
people will try to split it and then discover they cannot.

> **Verify this yourself:** Spring Boot 2.7's documented Java support range is 8
> through 17. Later 2.7.x patch releases may tolerate newer JDKs, but "tolerates"
> and "is supported on" are different claims and only one of them is safe to put
> in a plan. Check the exact range for the patch version you are on at
> `spring.io/projects/spring-boot#support` and in that version's release notes.
> This is the one version fact in the whole plan I want you to verify rather than
> take from me.

---

## Why does it matter?

**1. Because staying is not free, and the cost is invisible until it is enormous.**
An out-of-support baseline means no free CVE patches. That is the concrete,
sayable cost — and it is the one that gets a migration funded, because it is a
compliance and security argument rather than an engineering-taste argument. Spring
Boot 3.5 left OSS support in June 2026; Boot 2.x left it well before that. Verify
the exact dates at `spring.io/projects/spring-boot#support` before quoting them,
because "roughly" is not a thing you want in a security conversation. The second
cost is hiring and retention: engineers notice a Java 8 codebase, and the ones
who stay are not the ones you were hoping to keep.

**2. Because the 60% stall is the actual failure mode, and almost every plan
ignores it.** I have seen many migration plans. Nearly all of them are good at
weeks 1 through 12 and silent about months 9 through 24. That silence is where
the money goes. The plan's quality is decided by what it says about the last 40%,
and I will open your document at that section first.

**3. Because sequencing is where a Principal engineer adds value that a strong
Senior does not.** A strong Senior can execute any single step here better than
most people. Deciding that the JDK moves first, alone, on the old framework — and
being able to say why in two sentences that a director understands — is a
different skill. It converts one terrifying project into four boring ones, and
boring projects finish.

**4. Because the rollback story is where migrations actually hurt.** Everyone
plans the forward path. The plans that survive contact with production are the
ones where somebody wrote down, per step, the exact command to run when it goes
wrong, and then checked whether that command actually works given what the
database has done in the meantime. A rollback that is blocked by a Flyway
migration is not a rollback.

---

## The decision, framed

You are not deciding *whether* to migrate. That decision is made for you by
support windows. You are deciding **the sequence, the granularity, the rollback
boundary, and the forcing function.** Four decisions.

### Decision 1 — Sequence

The recommended sequence, and the reasoning for each position:

**Step 0 — Inventory and blocker discovery. (Weeks, not months. Do not skip.)**

Before any code changes, produce three lists:

- **Every deployable unit**, with its owning team, its last deploy date, its
  criticality, and whether it has a load test.
- **Every third-party dependency across the estate**, with its current version,
  whether a `jakarta`-compatible release exists, and whether the project is still
  maintained. `mvn dependency:tree` per module, aggregated. This list contains
  your blockers, and you want them in week two, not month nine.
- **Every place the estate reaches into JDK internals or manipulates bytecode.**
  Run the existing build on the new JDK with `-Xlint`, and run the existing test
  suite on the new JDK before changing anything. The failures you get are your
  actual work list.

The output of step 0 is a **blocker register**: for each dependency with no
`jakarta` release, a named decision — replace it, fork it, vendor it, or delete
the feature that uses it — with an owner. Every one of those decisions takes
weeks of elapsed time and none of them is technically hard. Discovering them
early is most of what step 0 buys.

**Step 1 — Compile with the new JDK, target the old bytecode.**

Set `maven.compiler.release` (or the Gradle equivalent) to 8 while running javac
from JDK 17. Nothing about the produced artifact changes. What changes is that
your build now runs on the new toolchain, so you flush out build-plugin
incompatibilities — the ones that will otherwise ambush you later — while the
output is still bit-for-bit the thing you were shipping.

Independently valuable: modest (a fixed build toolchain). Independently
revertible: trivially (change one property).

**Step 2 — Run on the new JDK. Still Boot 2.7. Still Java 8 language level.**

Change the base image. Deploy to one service. Compare against the Topic 65
baseline. This is the step where you find out about the collector change, the
locale-data change, the reflective-access failures, and any library that cannot
read the new class-file version.

Independently valuable: **very**. You get years of JVM security patches, G1 or ZGC
instead of Parallel, proper container awareness (Topic 82), and the accumulated
JIT and GC improvements. If the programme is cancelled tomorrow, this step alone
justified the quarter.

Independently revertible: **completely**. Change the image tag back. No code
change, no data change, no coordination. This is the best rollback story in the
entire plan and it is why this step goes first.

**Step 3 — Raise the language level.**

Now set `release` to 17. This unlocks records, switch expressions, `var`, text
blocks and sealed types for new code. It does not require rewriting anything.

Independently valuable: moderate, and mostly about developer experience and
recruiting. Independently revertible: yes, but note that once someone writes a
record, reverting the language level means reverting their code too. In practice
this step is a one-way door within a module after the first modern-syntax commit
lands — which is fine, because its risk is compile-time only.

**Step 4 — `javax` → `jakarta` and Boot 2.7 → 3.x, together, one deployable unit
at a time.**

This is the step. It cannot be split, for the reasons above. It *can* be done one
service at a time, and that is the granularity that matters. Per service:

- Run the OpenRewrite Boot upgrade recipe on a branch.
- Review the diff. Expect the recipe to handle the namespace rename, the
  dependency coordinate changes, and the property-key renames; expect to hand-fix
  your own abstractions, your reflective code, and anything string-based.
- Deal with the behavioural changes the recipe cannot see. On Boot 3 that
  includes the Hibernate 6 upgrade (HQL parsing is stricter, some generated SQL
  changes, `@Type` handling changed), the Spring Security 6 configuration idiom
  change, the trailing-slash matching default change in MVC, and the Jakarta
  Validation changes. This is where your test suite earns its existence.
- Run the Topic 65 load harness against the migrated service before shipping it.
  Compare percentiles. This is a framework upgrade; it *will* move some numbers,
  and you want to know which ones on purpose.
- Ship it. Watch it. Then do the next one.

Independently valuable: per service, yes — that service is now on a supported
line. Independently revertible: per service, yes — redeploy the previous
artifact — **provided** the schema and any message contract it publishes are
compatible. That proviso is the whole content of Decision 3 below.

**Step 5 — JDK 21, then JDK 25.**

Once on Boot 3, move the runtime again. Same shape as step 2: base image change,
compare against baseline, revert by tag. On Java 21 you also unlock virtual
threads (Topic 101), which is a separate decision with its own risk profile —
do not bundle it into the JDK step. Turning on `spring.threads.virtual.enabled`
is a change to your concurrency model, and it deserves its own rollout and its
own baseline comparison.

> **Verify:** which JDK versions a given Spring Boot 3.x line supports has moved
> across patch releases. Check the support page for your exact line rather than
> assuming 3.x means "any modern JDK".

**Step 6 — Boot 3.x → Boot 4.x.**

Spring Boot 4.0.0 went GA on 20 November 2025 and 4.1.0 on 10 June 2026; Spring
Framework 7.0 went GA on 13 November 2025 with a JDK 17 baseline, JDK 25
recommended, and Jakarta EE 11. The work here is smaller than step 4 because the
namespace rename is already done — Boot 4 does not redo it — but it is not
trivial. The parts that touch code: Boot 4 splits the codebase into many smaller
jars so the artifact you depend on for a given feature may have moved; Jackson 3
is standard and Jackson 2 support is deprecated; Spring Security 7 changes
filter-chain configuration idioms; the JSpecify null-safety annotations are
adopted portfolio-wide. Run the OpenRewrite Boot 4 recipe, then work the list.

**Step 7 — Decommission the old baseline.**

Not "declare victory". Actually remove the ability to build on the old baseline:
delete the JDK 8 base image, remove the compatibility profile from the parent POM,
turn the CI warning into a CI failure. Until this happens the migration is not
finished, and it can regress.

### Decision 2 — Granularity

The unit of work is the **deployable artifact**, not the commit and not the
module. Reasoning: the deployable artifact is the unit you can roll back. If you
migrate a shared library that six services depend on, you have just coupled six
rollbacks together, and the rollback story for each of them is now "coordinate
with five other teams".

This has a direct consequence for shared libraries: **the shared platform jars
must support both baselines during the transition**, or they must be versioned so
that services can pin the old line while they migrate. Publishing `platform-2.x`
(javax) and `platform-3.x` (jakarta) in parallel for the duration is
unglamorous, is real work, and is what keeps the per-service rollback story
intact. Write it into the plan explicitly; it is one of the most common omissions
and it surfaces in month three as "we're blocked on platform".

### Decision 3 — The rollback boundary

The question is not "can we revert the deploy". It is: **after this step ships,
what is now true of the system that the previous version cannot cope with?**
Three candidates, and you must check all three per step:

1. **Schema.** If the deploy ran a Flyway or Liquibase migration that the old
   version cannot read, your rollback is blocked. The fix is standard and you
   should state it as a rule in the plan: during a migration window, all schema
   changes are **expand-contract** — add the new column, deploy code that writes
   both and reads the new, and only drop the old column a release later, after
   the rollback window has closed. No destructive migration ships in the same
   release as a framework upgrade.
2. **Message and event contracts.** If the migrated service now publishes a
   payload the old consumers cannot parse — and Jackson 3 in step 6 is a live
   risk here — rolling back the producer is fine but rolling back after consumers
   have adapted is not. Keep serialization changes out of the migration steps.
3. **Persisted state shape.** Anything cached, anything in Redis with a
   serialized Java object in it, anything in a session store. Boot upgrades change
   serialization details. Flush or version your cache keys as part of the step.

Write the rollback for each step as a **command or a runbook line**, not as a
sentence. "Roll back: set image tag to `orderflow:2.7-jdk8`, redeploy, no schema
action required" is a rollback. "We can revert if needed" is not.

### Decision 4 — The forcing function

This is the section I will read first and attack hardest. The plan needs a
mechanism that operates without anyone choosing to prioritise the work.

The mechanism has four parts, and it must be all four:

**(a) A published deadline on the old baseline, with a date.**
"From 1 March, the shared parent POM version that supports Java 8 is no longer
published, and the JDK 8 base image is removed from the registry." Not "we
encourage teams to move by March".

**(b) That deadline enforced in CI, on a ratchet.**
Concretely, three stages with dates:
- Stage 1 (from date D1): a build **warning** for any module still on the old
  baseline, plus a check that fails on *new* modules only.
- Stage 2 (from D2): the build **fails** for any module that adds new code on the
  old baseline — the new-and-changed-code ratchet from Topic 132.
- Stage 3 (from D3): the build fails, full stop. The old base image is deleted
  from the registry, which makes it a deployment failure as well as a build one.

A Maven Enforcer rule banning `javax.*` imports (excluding the JDK packages —
`javax.sql`, `javax.crypto`, `javax.naming`, `javax.management`, `javax.net`,
`javax.security.auth`) is the cheapest possible version of this, and it belongs in
the shared parent POM where nobody has to opt in.

**(c) The migration made cheaper than the alternative.**
This is Topic 134's mechanic applied here. The platform team runs the OpenRewrite
recipe, opens the pull request, fixes the compile errors, and hands the owning
team a green build and a diff to review. The cost of adopting becomes "review a
PR". The cost of not adopting becomes "your build fails in six weeks". When those
two are the options, the work happens. When the options are "do a two-week
migration yourself" versus "ignore an email", the work does not happen — and no
amount of escalation changes that arithmetic.

**(d) A burndown that names owners, not percentages.**
"68% complete" is a number nobody is accountable for. A table with one row per
un-migrated service, the owning team, the blocker, and the date is a number four
people are accountable for. Publish it weekly, to the same audience, unchanged in
format. The format's dullness is a feature: it makes the flat weeks visible.

---

## Example 1 — a minimal illustration

Small enough to hold in your head, and it contains the whole mechanic.

### The situation

A single internal service: 12,000 lines, Spring Boot 2.7, Java 8, one Postgres
database, deployed as a container, no other service depends on it. Two engineers
own it. Their manager asks: "How long to get this on Boot 3?"

### The answer that sounds competent and is wrong

"About three weeks. We'll upgrade Java and Spring together on a branch, run the
tests, and deploy."

Three weeks is probably even roughly right. The problem is structural: at the end
of it there is a single deploy that changes the runtime, the namespace and the
framework simultaneously. If p99 doubles — and it might, because JDK 8's default
collector is Parallel and JDK 17's is G1, which is a real change to the latency
profile — you have three suspects and no way to separate them. And if it fails at
2 a.m., the rollback is "redeploy the old artifact", which is fine right up until
somebody notices Flyway ran a migration on the way in.

### The version with the same total effort and a different shape

**Day 1–2.** Run the existing test suite on JDK 17 without changing anything else.
Two failures: an old Mockito version that cannot read the class files it is asked
to mock, and one test that asserts on a formatted date string that changed because
JDK 9 switched the default locale data provider to CLDR. Both are twenty-minute
fixes and both would otherwise have appeared in the middle of a much larger diff.

**Day 3.** Set `maven.compiler.release=8`, build with JDK 17. Byte-identical
behaviour, new toolchain. Merge.

**Day 4.** Change the base image to JDK 17. Deploy to staging. Run the load
harness. p99 moves — G1 has a different pause profile from Parallel — and now you
know that with certainty, because it is the only thing that changed. Set
`-XX:MaxRAMPercentage` properly for the container while you are here (Topic 82).
Deploy to production. **Rollback: previous image tag. No schema change shipped in
this release, by rule.**

The service is now on a supported, patched runtime. If everything else is
cancelled, this was worth doing.

**Week 2.** `release=17`. Merge. Nothing to observe.

**Week 3.** OpenRewrite's Boot 3 recipe on a branch. Review the diff: the
namespace rename is done, the `pom.xml` coordinates are updated, several property
keys are renamed. Hand-fix the Security configuration, which changed idiom, and
two HQL queries that Hibernate 6 parses more strictly. Run the load harness again.
Deploy. **Rollback: previous image tag, and the one schema change in this release
was additive by rule.**

Same three weeks. Four rollback points instead of one, four attributable changes
instead of one compound mystery, and a genuine benefit banked on day four instead
of at the end.

### What you write down

A five-row table. Step, what changes, what it buys if we stop here, how to roll
back, how we know it worked. That table is the entire artefact in miniature —
scale it to 400k lines and add a forcing function, and you have the deliverable.

---

## Example 2 — the real decision on the project spine

### The setting

You are the Principal engineer for a platform group. The estate:

- ~400,000 lines of Java across **23 deployable services** and **6 shared
  platform libraries**.
- All on Spring Boot 2.7, Java 8 source and target, JDK 8 base image.
- `orderflow` is one of the 23 and is the highest-traffic service. It has a Topic
  65 baseline, a Topic 124 readiness review, and load-test coverage.
- Of the other 22: eight are actively developed with load tests; eleven are
  actively developed without load tests; **three have not been deployed in over a
  year and one of those has no owning team.**
- CI is shared. There is a shared parent POM. The base images come from one
  internal registry.
- Security has raised a finding: the baseline is out of open-source support and
  therefore receives no free CVE patches. That finding is what funded this work.

Your director has asked for "the plan". You have two weeks to write it and you
will present it once.

### Step 0 — Inventory, and what it turns up (weeks 1–3)

The inventory is not paperwork. It is where the plan's risk actually lives.

**The dependency sweep** produces the blocker register. In an estate this age you
should expect to find, in the following rough order of frequency:

- Several libraries with a straightforward `jakarta`-compatible release — no
  decision needed, just a version bump the recipe will do.
- One or two libraries that were abandoned before the rename and have no
  `jakarta` release at all. **These are the blockers.** For each one the plan must
  name a decision: replace it with a maintained equivalent, fork and rename it
  yourself, or delete the feature that depends on it. Each of those is a multi-week
  item with its own owner, and each must start in month one, not month nine,
  because the elapsed time is the cost, not the effort.
- Something reflecting into JDK internals — often a serialization helper or an
  older bytecode-manipulation library.
- An old `javax.annotation` usage that is neither JDK nor Jakarta EE in the way
  people assume, which will confuse someone for an afternoon.

**The ownership sweep** produces something more uncomfortable: the service with
no owner. That is not a technical problem and you must not treat it as one. Put
it in the plan as an explicit decision for your director: **assign an owner, or
decommission it.** A migration plan that quietly assumes the platform team will
migrate an unowned service has just absorbed unbounded work, and that is one of
the two or three most reliable ways to reach 60% and stop.

**The test-coverage sweep** tells you the rollout order. Services with a load test
and an owner go first; not because they are the most valuable, but because they
are the ones where you will *learn* the most per unit of risk.

### The sequence, as it appears in the plan

| # | Step | Scope | Independently valuable? | Rollback | Verified by |
|---|---|---|---|---|---|
| 0 | Inventory + blocker register | Estate | Yes — you now know what you own | n/a | Register exists, every blocker has an owner and a decision |
| 1 | Build on JDK 17, `release=8` | Estate, via parent POM | Weak — toolchain fixed | Revert one property | CI green estate-wide; artifacts unchanged |
| 2 | **Run** on JDK 17, still Boot 2.7 | Per service, `orderflow` first | **Strong — CVE patches, G1, container awareness** | **Image tag revert. No schema change in this release.** | Topic 65 baseline re-run, percentiles compared |
| 3 | `release=17` | Per service | Weak — DX, hiring | Revert property (before modern syntax lands) | CI green |
| 4 | Platform libraries dual-published (`-2.x` javax, `-3.x` jakarta) | 6 libraries | Yes — unblocks per-service migration | Consumers stay on `-2.x` | Both lines build and publish |
| 5 | **`jakarta` + Boot 3, together** | Per service | Yes per service — supported line | Image tag revert; expand-contract schema rule in force | Load harness re-run; error rate and p99 compared |
| 6 | JDK 21 | Per service | Yes — LTS, virtual threads available | Image tag revert | Baseline re-run |
| 7 | Virtual threads enabled | Per service, **separately** | Yes — concurrency headroom | Config flag off | Baseline re-run; pinning check (Topic 101) |
| 8 | JDK 25 | Per service | Yes — current LTS | Image tag revert | Baseline re-run |
| 9 | Boot 4.x | Per service | Yes — current line, longest runway | Image tag revert | Baseline re-run; Jackson 3 contract check |
| 10 | Decommission old baseline | Estate | Yes — one baseline to patch, not two | n/a — this is the point of no return, deliberately | Old image deleted; parent POM profile removed |

Two things about that table are worth stating explicitly in the plan, because
reviewers will ask:

**Why is step 7 separate from step 6?** Because virtual threads change your
concurrency model, not your runtime. Enabling them can expose pinning (Topic 101),
can move the bottleneck onto the connection pool (Topic 109), and changes the
shape of your thread dumps. Bundling it with a JDK upgrade means a p99 change with
two suspects. It costs nothing to separate and it buys attribution.

**Why does `orderflow` go first?** Because it has the baseline, the readiness
review and the load harness. The most instrumented service is the one where a
surprise is cheapest, because you will see it. The instinct to migrate the
least-important service first is wrong: you learn nothing from a service nobody
watches.

### The rollback rules, written as rules

These go in the plan as a short numbered list, because they are the things that
have to be true across all 23 services for the per-step rollbacks to be real:

1. **No destructive schema change ships in the same release as a migration step.**
   Expand-contract only. Drops happen at least one release after the rollback
   window closes.
2. **No serialization-format change ships in a migration step.** If Jackson 3
   changes a payload shape in step 9, that change ships separately, with consumer
   coordination.
3. **Every migration step ships behind the normal deploy process**, so the normal
   rollback works. No manual environment surgery, no hand-edited config.
4. **Caches are versioned or flushed** as part of any step that could change
   serialized state.
5. **The rollback is executed at least once, deliberately, in staging**, per step
   type. A rollback nobody has performed is a hypothesis.

### The stall-prevention mechanism, in full

This is the section the plan is judged on. Four parts.

**(a) The deadline.** From the date the plan is approved, publish two dates:
- **D1 (approval + 6 months):** the JDK 8 base image is deprecated. Builds on it
  emit a warning naming the owning team.
- **D2 (approval + 9 months):** the shared parent POM no longer publishes a
  Java-8-compatible line. Any module that has not moved fails its build on any
  change.
- **D3 (approval + 12 months):** the JDK 8 base image is deleted from the
  registry. Services on it can no longer be deployed.

The dates are illustrative of the *shape*; set your own from the inventory, and
set them so the last one is achievable by the teams that have the hardest
blockers. A deadline that is impossible for one team is a deadline everyone
learns to ignore.

**(b) The enforcement, in CI, on the ratchet.** In the shared parent POM:
- An Enforcer rule requiring `maven.compiler.release >= 17`, initially at `warn`,
  flipping to `fail` at D2.
- A banned-import rule for Jakarta-EE `javax.*` packages, explicitly excluding the
  JDK ones (`javax.sql`, `javax.crypto`, `javax.naming`, `javax.management`,
  `javax.net`, `javax.security.auth`, `javax.xml`). Introduce it on
  new-and-changed files only, with the existing violations baselined — this is
  precisely Topic 132's ratchet, and it is the difference between a gate that
  survives and one that gets switched off in a week.
- A CI check that reports, per pipeline run, which baseline the module is on. The
  data feeds the burndown automatically; nobody maintains a spreadsheet.

**(c) Adoption made cheaper than non-adoption.** The platform group commits to
this, in writing, in the plan:
> For any service, on request, we will run the OpenRewrite recipe, resolve the
> compile errors, run the test suite, and open the pull request. The owning team
> reviews and merges. We will pair on the first deploy.

That is a real capacity commitment and it must be costed in the plan — say, two
engineers for the duration. It is the single highest-leverage line in the
document. It converts "do a migration" into "review a PR", and the ratio of those
two costs is the ratio that decides whether you finish.

**(d) The burndown, by owner.** One table, published weekly, format never
changing:

| Service | Owner | Current baseline | Blocker | Target date | Δ since last week |
|---|---|---|---|---|---|
| | | | | | |

Not a percentage. Not a burn-up chart. Names and blockers. When a row does not
change for three weeks, that is visible to the row's owner and to their director
in the same glance, and the conversation that follows is specific ("what is the
blocker?") rather than general ("how is the migration going?").

### The three services that will stall, named in advance

The plan should say this out loud, because saying it out loud is what makes it
manageable:

> Three services are at high risk of stalling: the two that have not been
> deployed in a year, and the one with no owning team. For these, the default is
> **decommission**, not migrate. Migrating a service nobody deploys costs the same
> as migrating one that matters and buys nothing. The decision on each is due by
> D1, and the decision-maker is [named person], not the platform team.

Naming the stall risk in the plan, with a decision owner and a date, is worth more
than any amount of process. It is also the sentence that most distinguishes a
Principal-level plan from a good Senior one: it does not assume the difficulty is
technical.

---

## Wrong approach → exact symptom → root cause → fix

Organisational symptoms. No stack traces. What you would have *seen*.

---

### Wrong approach 1 — the big-bang branch

**Wrong:** one long-lived branch upgrades the JDK, the namespace and the framework
across the whole estate, to be merged when it is green.

**Exact symptom — what you would have SEEN:**

- A branch called `feature/boot3-upgrade` that is four months old, 1,400 files
  changed, and 300 commits behind `main`. The daily merge from `main` takes an
  engineer an hour and sometimes fails.
- A standing agenda item, "Boot 3 branch status", that has said "80%, blocked on
  the reporting library" for nine consecutive weeks.
- Nobody can review the diff. The pull request, when it is finally opened, gets
  two approvals in four minutes from people who scrolled.
- One dependency with no `jakarta` release blocks the *entire estate*, because
  everything is in one branch. Twenty-two services that could have shipped are
  waiting on the one that cannot.
- When it does merge, p99 on `orderflow` moves by a noticeable amount and there
  are three plausible causes. The investigation takes a week and ends
  inconclusively.
- If anything goes wrong in production, the rollback is "revert four months of
  work", which nobody will authorise, so instead you fix forward under pressure —
  which is the worst possible place to be doing framework debugging.

**Root cause:** batch size. Every risk in the migration was aggregated into a
single event, which means the failure probability of that event is the *union* of
every individual risk, and its blast radius is the whole estate. The plan
optimised for "do it once" over "be able to stop".

**Fix:**
1. The unit of work is the deployable artifact. Twenty-three independent
   migrations, not one.
2. The unit of change within a service is the step: runtime, then namespace +
   framework, and never both in one deploy.
3. Nothing lives on a branch longer than a week. If the recipe's diff is too big
   to review in one sitting, split it by module.
4. One blocked dependency blocks one service, not the estate.

---

### Wrong approach 2 — the migration that stalls at 60% and stays there for two years

**Wrong:** the plan is a good technical plan with a communication strategy: an
announcement, a wiki page, a monthly reminder, and a request that teams migrate
"as capacity allows".

**Exact symptom — what you would have SEEN:**

- Months 1–7: excellent progress. Fourteen of 23 services move. The burndown
  chart is a satisfying diagonal line and gets shown at an all-hands.
- Month 8: three services move.
- Months 9–24: **zero.** The chart is flat at 61% for six quarters. It is still on
  the slide deck. Nobody looks at it.
- The remaining nine services are, without exception: the ones with a blocked
  dependency, the ones with no load test whose owners are reasonably scared, the
  ones whose teams are on a product deadline, and the two nobody owns.
- The platform team is now maintaining two parent POM lines, two base images, two
  sets of CI configuration and two runbooks — indefinitely. Nobody scheduled that
  work; it is simply always there.
- Every security finding now has two remediation paths and someone has to
  remember which services need which.
- Eighteen months in, somebody proposes "a Boot 3 migration initiative" as if it
  were new.

**Root cause:** the pain that motivated the migration was concentrated in the
services that moved first. Removing it removed the motivation. What remains has a
real cost to migrate and no *felt* benefit to the team that owns it, so it will
lose every prioritisation conversation against work that has a felt benefit —
correctly, from that team's point of view. **The plan relied on cooperation, and
cooperation is a function of local incentives, which the plan did not change.**

**Fix:**
1. **A date enforced in CI**, ratcheted through warn → fail-on-change → fail. The
   build is the only thing in the organisation that cannot be deprioritised.
2. **Delete the old path.** As long as the JDK 8 base image exists in the
   registry, staying is an option. Removing it is the mechanism; the announcement
   was never the mechanism.
3. **Make adoption cheaper than non-adoption.** Fund a squad that does the work
   and hands over a pull request. This is Topic 134's whole subject and it is not
   optional here.
4. **Burn down by owner, not by percentage.** A flat row with a name on it
   generates a conversation. A flat percentage generates a slide.
5. **Budget for the tail explicitly.** Assume the last 30% costs as much as the
   first 70%, and say so in the plan. A plan whose effort curve is linear is a
   plan that has not met a migration.

---

### Wrong approach 3 — framework first, runtime later

**Wrong:** "Boot 3 is the goal, so let's start with Boot 3." The JDK upgrade is
treated as a prerequisite to be done in the same change, or afterwards.

**Exact symptom — what you would have SEEN:**

- The team discovers in week one that Boot 3 requires Java 17, so the JDK upgrade
  happens anyway — but now inside the same pull request, undiscussed and
  unmeasured.
- The service ships. Two days later, p99 on the order-placement path is
  materially different from the Topic 65 baseline. There are now at least four
  candidate causes: the collector default changed from Parallel to G1, Hibernate 6
  generates different SQL for two queries, the connection pool defaults moved, and
  Spring's request-handling path changed. The investigation is a week long.
- A separate, subtler version: an invoice date renders differently in one locale
  because JDK 9 made CLDR the default locale-data provider. Nobody notices for a
  month; a customer does.
- Somebody proposes rolling back to get a clean measurement, and discovers the
  rollback would also undo a schema change.

**Root cause:** two independent variables changed in one observation. You had a
baseline (Topic 65) specifically so you could attribute changes to causes, and the
sequencing threw that away. The deeper error is treating the JDK upgrade as
overhead on the way to the "real" goal, when it is the step with the best
value-to-risk ratio in the entire programme.

**Fix:**
1. JDK first, alone, on the old framework. Measure against the baseline. This
   works precisely because Java class files are forward-compatible — say that
   sentence in the plan, because it is the justification.
2. One variable per deploy, and a baseline comparison after each.
3. Keep a written list of "known behaviour changes from this step" per step —
   collector default, locale data provider, strong encapsulation — and check the
   observed changes against it before opening an investigation.

---

### Wrong approach 4 — `sed -i 's/javax/jakarta/g'`

**Wrong:** the namespace change is treated as a text substitution, because
mechanically that is what it looks like.

**Exact symptom — what you would have SEEN:**

- The build fails with `cannot find symbol: class DataSource`, because
  `javax.sql.DataSource` is a JDK type that was never Java EE and must not be
  renamed. Same for `javax.crypto`, `javax.naming`, `javax.management`,
  `javax.net.ssl`.
- A string literal in a configuration file — a fully-qualified class name in a
  properties file, a `Class.forName` argument, an XML namespace declaration — is
  either renamed when it should not have been, or not renamed when it should have
  been. The compiler cannot help you with either, so it fails at runtime.
- A comment or a piece of Javadoc is silently rewritten, adding noise to a diff
  that reviewers were already going to skim.
- The reviewer has 1,400 files to look at and no way to tell the correct renames
  from the incorrect ones, so the review is theatre.
- The subtlest one: everything compiles, and one code path that resolves a class
  by name at runtime throws `ClassNotFoundException` the first time it executes —
  in production, on a weekly batch job, eleven days later.

**Root cause:** text substitution has no type information. It cannot distinguish
`javax.persistence.Entity` (a Jakarta EE type that must move) from
`javax.sql.DataSource` (a JDK type that must not), because textually they are the
same prefix. The transformation needed is semantic, so the tool must be semantic.

**Fix:**
1. **OpenRewrite**, which parses to a type-attributed Lossless Semantic Tree and
   therefore knows which classpath entry each symbol resolved from. It also
   preserves your formatting, so the diff contains only real changes and the
   review is meaningful.
2. Add the banned-import Enforcer rule *after* the migration, with the JDK
   `javax` packages explicitly allowed, so the estate cannot regress.
3. For the residue the tool cannot see — strings, reflection, XML namespaces,
   resource files — grep deliberately and check each hit by hand. Write that list
   into the plan as a per-service checklist item, because it is the part that
   escapes to production.

---

### Wrong approach 5 — "we can always roll back", with a Flyway migration in the release

**Wrong:** the rollback plan for every step is "redeploy the previous image tag",
and nobody checked what else shipped in that release.

**Exact symptom — what you would have SEEN:**

- The Boot 3 deploy goes out at 14:00 with a Flyway migration that renames a
  column and drops the old one, because a Hibernate 6 mapping change made it
  convenient to tidy up at the same time.
- At 14:40 an error rate rises on an unrelated endpoint. The on-call engineer does
  the correct thing and redeploys the previous tag.
- The previous version starts, fails on its first query against the renamed
  column, and enters a crash loop. You now have zero working versions, at 14:45,
  under load, in front of customers.
- The recovery is a hand-written SQL migration composed under pressure, which is
  the single most dangerous artifact in software engineering.
- In the postmortem someone writes "root cause: the developer included a
  destructive migration in the upgrade release." That prevents nothing (Topic
  133), because the next person will also do it.

**Root cause:** the rollback boundary was assumed to be the deployable artifact,
but the real boundary was the artifact *plus the state of the database*. Nobody
wrote down what "revertible" meant for this step, so nobody checked whether it was
true.

**Fix:**
1. State the rule in the plan and enforce it in review: **during the migration
   window, all schema changes are expand-contract; no drop, no rename, no
   not-null constraint ships in the same release as a migration step.**
2. Add a CI check that fails a release containing both a migration-step change and
   a destructive Flyway script. Mechanism, not memo.
3. Per step in the plan, write the rollback as a command plus its preconditions,
   and mark which precondition could be violated by an unrelated change.
4. Rehearse one rollback per step type in staging. A rollback nobody has run is a
   hypothesis, and this is a cheap experiment.

---

## Artefact — what you must produce

### Specification

**Title:** `Platform migration plan — Java 8 / Boot 2.7 estate to Java 25 / Boot 4`

**Length:** 2,000–3,000 words of prose, plus tables. Tables do not count toward
the prose budget. If the prose runs past 3,000 words you are explaining the
migration rather than sequencing it, and the sequencing is the deliverable.

**Audience:** an engineering director who will fund it, and the 23 team leads who
will execute it. It must work for both. That means the first page is the sequence,
the dates and the cost; the detail comes after.

**Required sections, in this order:**

**1. The forcing reason, in three sentences.** Why now. The support-window fact
with a date and a source, the security consequence, and the cost of the status
quo. No engineering-taste arguments — "the code would be nicer" does not fund a
year of work and should not be in the document.

**2. Estate inventory.** A table with one row per deployable unit:

| Service | Owning team | Last deploy | Criticality | Has load test? | Has baseline? | Blocking dependencies |
|---|---|---|---|---|---|---|
| | | | | | | |

Plus a count: total lines, total modules, total distinct third-party dependencies.
If any row's owner column is empty, that is a finding, not a formatting problem,
and it belongs in section 7.

**3. Blocker register.** One row per dependency with no compatible release:

| Dependency | Used by | Last upstream release | Decision (replace / fork / vendor / delete feature) | Owner | Due |
|---|---|---|---|---|---|
| | | | | | |

Every row must have a named human and a date. A blocker register with an empty
Owner column is the single most reliable predictor of a stall.

**4. The sequence.** The step table, with these exact columns, and every cell
filled:

| # | Step | Scope (estate / per service) | What it buys if we stop here | Rollback (as a command) | Rollback preconditions | How we verify it worked |
|---|---|---|---|---|---|---|
| | | | | | | |

The "what it buys if we stop here" column is the test of the whole plan. If any
row's answer is "progress toward the next step", that step is not independently
valuable and must be merged with its neighbour or justified explicitly.

**5. Rollback rules.** The estate-wide rules that make the per-step rollbacks real.
Expand-contract schema policy, serialization-change policy, cache-versioning
policy, and the rehearsal commitment. Numbered, short, enforceable.

**6. The forcing function.** All four parts:
- the dates (D1 / D2 / D3) and what happens on each;
- the CI enforcement, described concretely enough that someone could implement it
  — which rule, in which file, on what ratchet;
- the capacity commitment that makes adoption cheaper than non-adoption, with a
  headcount number;
- the burndown format, with the actual column headers.

**7. Named stall risks.** Which services will stall, why, and what the *decision*
is for each — including the ones where the answer is "decommission". Each with a
decision-maker who is not you and a date.

**8. Cost and shape.** Total engineer-months, split between the migration squad
and the owning teams. Elapsed time. What is not being built because of this. Be
honest; a plan that claims to be free is a plan nobody believes.

**9. What I am least sure about.** One paragraph. Name your own weakest estimate.
This is not modesty — it tells the reviewer where to start, which makes the review
shorter and better.

### Constraints

- **Every version and support-window fact carries a source.** `spring.io` support
  page, `openjdk.org`, or your JDK vendor's support page. No remembered dates.
- **No invented effort numbers.** "12 engineer-days, estimated by analogy with the
  Testcontainers upgrade which took 9" is legitimate. "About two weeks" is not.
  Every estimate carries its method.
- **Rollbacks are written as commands with preconditions**, not as reassurances.
- **The forcing function must be a mechanism**, not a communication plan. If any
  part of it depends on someone choosing to prioritise the work, mark it as such
  and explain what happens when they do not.
- **Percentages are banned from the burndown.** Names and blockers only.

---

## How I will review it

### The three questions that usually break a document of this kind

**Question 1 — "Which step is *not* independently revertible, and what is your
plan for that one?"**

Every migration plan has at least one, and the honest ones say so. If the answer
is "all of them are revertible", the plan has not been thought about, and I will
find the counterexample in under a minute — usually the shared-library migration,
or the language-level bump after the first record has been written, or the step
where a cache serialization format changed.

*How the document fails:* a rollback column that says "redeploy previous version"
on every row, with no preconditions column. That is not a rollback plan; it is a
description of how deploys work.

*What a strong answer looks like:* naming the irreversible step, explaining what
makes it irreversible, and stating what you do instead — a longer soak, a canary
at 1%, a dual-write window, a feature flag, or an accepted risk with a named
owner. "Step 4 is not revertible after 24 hours because consumers will have
adapted, so we hold the old consumer path for a week and the flag is X" is a real
answer.

**Question 2 — "It is month nine. You are at 60% and the burndown has been flat
for two months. Show me the thing in this plan, already running, that makes month
ten different."**

This is the question the topic exists for. I will ask it of every plan and most
plans do not have an answer.

*How the document fails:*
- The forcing function is an announcement, a wiki page, a monthly reminder, or an
  escalation path. None of those change the local incentive of the team that has
  a product deadline.
- The deadline exists but nothing enforces it. A date with no build behind it is a
  suggestion with a date on it.
- The CI gate is planned for "after most teams have migrated" — which means it
  arrives after the stall, when it is precisely useless, because the teams that
  remain are the ones who could not move.
- The plan assumes the platform team will do the tail. That may even work, but if
  it is the plan, it must be costed and staffed in section 8, and it usually is
  not.

*What a strong answer looks like:* three dates, a specific CI rule that flips at
each one, the old artifact deleted from the registry at the last one, a funded
squad that opens the pull requests, and a burndown with names. Plus the sentence
I want most: *"the ratchet is on new-and-changed code first, so the gate does not
fail 4,000 existing warnings on day one and get switched off in a week."*

**Question 3 — "Name the dependency that blocks you, and show me the decision you
have already made about it."**

Every 400k-line estate of this age has at least one library that never made the
`jakarta` move. It is always found, and it is almost always found in month seven
rather than week two. It is the difference between a plan and an aspiration.

*How the document fails:*
- No blocker register at all.
- A register with entries and no owners, or entries whose "decision" is
  "investigate".
- A register that lists only direct dependencies. The blockers live in the
  transitive set; if you did not run `mvn dependency:tree` across every module and
  aggregate it, you have not looked.
- The related failure: an unowned service treated as a technical work item rather
  than as a decision your director has to make.

*What a strong answer looks like:* a named dependency, a named replacement or
fork decision, an owner, a date in the first third of the programme, and an
explicit statement of what happens to the services that depend on it if the
decision slips.

### The other attacks, in the order I will make them

- **"What changed in your p99 when you moved from Parallel to G1?"** If the plan
  does not mention that JDK 8's default collector differs from JDK 9+'s, you have
  not thought about step 2 as a performance change. You have a Topic 65 baseline;
  use it.
- **"Your shared libraries — do consumers migrate on your schedule or theirs?"**
  If the platform jars are not dual-published, every consumer's rollback is
  coupled to every other consumer's, and the per-service granularity you claimed
  in section 4 is fictional.
- **"Show me the release where a Flyway migration and a migration step ship
  together."** If you cannot rule it out with a mechanism, your rollback column is
  optimistic.
- **"Which step enables virtual threads?"** If it is bundled with the JDK 21 step,
  I will ask how you will attribute the next latency change.
- **"What is the plan for the service with no owner?"** If the answer is "the
  platform team will handle it", the plan has absorbed unbounded work.
- **"You said OpenRewrite handles the rename. What does it not handle?"** I want
  to hear: strings, reflection, XML namespaces, resource files, your own
  abstractions, and dependencies with no compatible release. A plan that treats
  the tool as complete will discover the residue in production.
- **"What does the estate cost while it is in two states?"** Two parent POMs, two
  base images, two runbooks, two CVE pipelines. That cost is the argument for
  finishing, and stating it strengthens section 1 considerably.

### What I will not attack

Prose, formatting, the choice of target version, or your effort estimates as
numbers — only their *method*. I have no stake in whether you go to 21 or 25
first. I am attacking the weakest assumption the sequence rests on, and in a
migration plan that is almost always either the rollback column or the forcing
function.

---

## Interview questions (Senior → Principal)

> The rubric line from the master plan:
> *"we'll upgrade everything in Q3" → "we upgrade the JDK first while staying on
> Boot 2 — that's independently valuable and independently revertible. Then
> `jakarta` mechanically via OpenRewrite. The stall risk is that teams stop when
> the pain stops; so the plan has a deprecation deadline on the old baseline in
> CI, not a request for cooperation."*

---

### Q1 — "We have a 400,000-line Boot 2.7 / Java 8 estate that's out of support. How would you plan the migration?"

**A senior answer sounds like:** "I'd start with an inventory of the services and
their dependencies to find blockers. Then I'd upgrade Java to 17 and Spring Boot
to 3 — Boot 3 requires Java 17 and the `jakarta` namespace, so those go together.
I'd use OpenRewrite for the mechanical parts, migrate service by service starting
with the least critical, and run the load tests after each one. I'd expect it to
take two or three quarters."

That is a good answer. It is technically correct in every particular, it names the
right tool, it has the right granularity, and most candidates do not get there.

**A principal answer sounds like:** "Three things decide whether this succeeds,
and only one of them is technical.

First, the sequence. Java class files are forward-compatible, so I can move the
*runtime* without touching the framework. That means step one is: run on JDK 17,
still on Boot 2.7, still compiling to Java 8 bytecode. That step is independently
valuable — it gets us years of JVM security patches, G1 instead of Parallel, and
proper container awareness — and it is independently revertible by changing an
image tag. If the programme is cancelled the next week, we banked a real win. Then
language level. Then `jakarta` and Boot 3, which have to be one step because you
cannot run `jakarta` on Boot 2.7 and you cannot run Boot 3 on `javax` — the flat
classpath means there is no both. Then JDK 21, then 25, then Boot 4. And virtual
threads get their own step, separate from the JDK step, because that is a change
to the concurrency model and I want to be able to attribute a latency change to
it.

Second, the granularity. The unit is the deployable artifact, because that is the
unit I can roll back. Twenty-three independent migrations, not one. Which means
the shared platform libraries have to be dual-published — a `javax` line and a
`jakarta` line — for the duration, or every service's rollback is coupled to every
other's. That is unglamorous work that has to be in the plan on day one.

Third, and this is the part that actually decides it: the stall. Every migration
of this size stalls somewhere around 60%, because the services that moved first
were the ones that were hurting, and moving them removed the pain that funded the
programme. What is left is the batch job with no owner and the service whose team
is on a product deadline, and for those teams migrating is pure cost. They will
deprioritise it, correctly, every quarter, forever. So the plan needs a mechanism
rather than a request: a deprecation deadline on the old baseline enforced in CI —
warn, then fail on changed modules, then fail outright — the old base image
deleted from the registry so staying is not an option, and a funded squad that
runs the OpenRewrite recipe and hands each team a pull request, so the cost of
adopting is a code review and the cost of not adopting is a broken build. And the
burndown lists owners and blockers, not a percentage, because a percentage is
something nobody is accountable for.

The blocker register is the last thing, and it goes first in the calendar: the one
library in the estate that never made the `jakarta` move. Find it in week two,
decide in month one — replace, fork, or delete the feature — because the cost is
elapsed time, not effort."

**What separates them:** four things.

1. The senior answer treats the JDK upgrade as a prerequisite. The principal
   answer treats it as **the highest value-to-risk step in the programme** and
   sequences everything else around that property. It also knows *why* it works
   (class-file forward compatibility) rather than just that it does.
2. The senior answer picks the least critical service to go first. The principal
   answer picks the **most instrumented** service, because a surprise you cannot
   see teaches you nothing. That inverts a common instinct and the reasoning is
   what makes it credible.
3. The senior answer's risk model is technical. The principal answer's risk model
   is **organisational**, and it identifies the failure mode — the stall — before
   it happens and puts a mechanism against it. This is the single biggest gap.
4. The senior answer says "I'd use OpenRewrite". The principal answer knows what
   OpenRewrite cannot do, and plans for the residue.

**Adversarial follow-up:** *"Your CI deadline sounds like it will just annoy
people. What happens when a team hits the deadline with a genuine product
commitment and a blocked dependency?"*

The honest answer does not defend the deadline as sacred. It says: the deadline
has an exception process with a named approver — usually the director who funded
the programme, not the platform team — and every exception is time-boxed and
appears on the burndown as an exception with an expiry, so it is visible rather
than quiet. The point of the deadline is not to punish; it is to make "not
migrating" a decision someone has to actively make and be seen making, instead of
a default that happens through inaction. And if three teams all need exceptions
for the same reason, that is information: the dates were wrong, or the squad is
understaffed, and the plan changes. A deadline that generates no exceptions was
probably set too late.

---

### Q2 — "Why not just upgrade to Boot 3 and Java 17 in one release? It's the same total work."

**A senior answer sounds like:** "Doing them separately is safer. If something
breaks, you know which change caused it, and you can roll back a smaller change.
It's the same reason we keep pull requests small."

Correct, and well-reasoned. It generalises the right principle.

**A principal answer sounds like:** "It is not the same total work, and the
difference is not safety in the abstract — it is three specific things.

First: attribution. We have a recorded baseline from our load work. JDK 8's
default collector is Parallel and JDK 9 onward it is G1, so the runtime change
alone will move the pause profile and probably p99. Hibernate 6 in Boot 3 changes
generated SQL for some queries. If both ship together and p99 moves, I have two
suspects and a week-long investigation. Separated, each has one suspect and a
same-day answer. The baseline exists precisely so we can attribute changes to
causes; bundling throws away the thing we already paid for.

Second: the value curve. The JDK step alone delivers the security patches that
funded the programme. If we bundle, that benefit is deferred behind all of the
framework risk. Separated, we bank it in week four. That materially changes the
conversation if the programme gets cut, and programmes do get cut.

Third: the rollback. The JDK step's rollback is an image tag, no code change, no
schema. The framework step's rollback is a redeploy with preconditions about the
schema and the serialization format. Those are different risk classes and they
should not share a blast radius. Bundling means the cheap-to-revert change
inherits the expensive-to-revert change's constraints.

And there is a smaller fourth thing: doing the JDK step first flushes out the
build-plugin and bytecode-library incompatibilities — old Mockito, old Lombok,
anything doing `setAccessible` into JDK internals — while the diff is still small
enough to read."

**What separates them:** the senior answer states a correct general principle. The
principal answer names the **specific** things that would change — collector
default, Hibernate SQL generation, class-file version in bytecode tooling — and
connects the decision to an asset the team already has (the baseline) and to the
programme's funding risk. It also distinguishes the two rollbacks by *class*
rather than treating "rollback" as one thing.

**Adversarial follow-up:** *"Fine, but our team is small and every extra deploy
costs us a day of coordination. At what point does your sequencing become
ceremony?"*

The honest answer concedes there is a real threshold and names it. For a single
service with two owners and a good test suite — Example 1 — the sequence costs
almost nothing because the deploys are cheap. For an estate where each production
deploy needs a change board, the calculus genuinely changes, and the right answer
may be to combine steps 1 and 2, or 2 and 3, while never combining the runtime
step with the framework step, because that is the pair with the worst attribution
problem. The principle to hold is not "always separate everything"; it is "never
put two independent sources of latency change in one deploy". Everything else is
negotiable against deploy cost.

---

### Q3 — "How do you decide which JDK to target — 21 or 25?"

**A senior answer sounds like:** "Both are LTS. 25 is the newer one, so it has a
longer support window. I'd target 25 unless a dependency doesn't support it yet.
I'd check that our framework version supports it."

Right conclusion, right check.

**A principal answer sounds like:** "Two questions decide it and neither is 'which
is newer'.

The first is: **whose support window?** 'Java 21 is supported until X' is an
incomplete sentence — support is a per-vendor commitment. Oracle, Temurin,
Corretto, Zulu and Red Hat each publish their own end date for the same version
and they are not the same. So the question is which vendor's builds we run, what
that vendor commits to for 21 versus 25, and when each of those dates lands
relative to our next planned upgrade. If we are going to move again in three
years anyway, the marginal runway from 25 may be worth less than the fact that
more of our dependency set is battle-tested on 21 today.

The second is: **what is on the path?** Spring Framework 7 has a JDK 17 baseline
and recommends 25, so on Boot 4 the target is 25. But we are not going to Boot 4
first — we are going to Boot 3 first, and Boot 3.x's supported JDK range has moved
across patch releases, so I would check the support page for the exact line rather
than assume. In practice the sequence is 17 to unblock Boot 3, then 21 as an LTS
resting point where virtual threads become available, then 25 alongside Boot 4.
Each of those is an image-tag change with a baseline comparison, so the cost of
doing it in three moves rather than one is small and the risk is much lower.

What I would not do is target a non-LTS release. The 6-month cadence means a
non-LTS version stops receiving updates about six months after it ships, and
committing an estate to a six-month support window is a decision to do this again
immediately."

**What separates them:** the senior answer treats "LTS" as a property of the
version. The principal answer knows it is a property of the **vendor's commitment
to** the version, and that the sentence is incomplete without naming one. It also
treats the JDK target as a waypoint sequence rather than a single choice, and ties
each waypoint to what it unblocks.

**Adversarial follow-up:** *"Our vendor's support for 21 outlasts our planned
lifetime for half these services. Why move to 25 at all?"*

The honest answer takes the point seriously, because it is a good one. If the
support window covers the service's expected life, "stay on 21" is defensible, and
the counter-arguments are not about support: they are that a single estate-wide
baseline is much cheaper to operate than two, that Spring Framework 7 recommends
25 so you are running a less-tested combination, and that the cost of the move is
an image tag and a baseline comparison. If the estate is genuinely heterogeneous
already and the services are genuinely short-lived, staying is a legitimate
answer — and a Principal engineer should be able to say that rather than reflexively
defending the newer number.

---

### Q4 — "Your migration is at 60% and has been flat for two quarters. What do you do?"

**A senior answer sounds like:** "I'd find out what's blocking the remaining
teams and help unblock them. Probably escalate to leadership to get it
prioritised, and offer engineering help to the teams that are struggling."

Reasonable, and it will produce some movement.

**A principal answer sounds like:** "First I would stop treating it as a
motivation problem, because it almost certainly is not. The teams that have not
moved are behaving rationally: for them, migrating is pure cost with no felt
benefit, and they have work in front of them that does have a felt benefit. Any
plan that requires them to choose differently is asking them to be wrong on
purpose.

So I would do four things.

Diagnose by category, not by team. The remaining services will fall into three or
four buckets: a blocked dependency, no test coverage so the team is reasonably
afraid, no owner, or genuinely deprioritised. Those need different interventions
and mixing them wastes everyone's time. The unowned ones are not an engineering
problem at all — they are a decision someone in leadership has to make, and the
answer is often 'decommission', which is a win that looks like a loss.

Change the arithmetic. Fund a squad that runs the recipe, fixes the compile
errors, gets the tests green and opens the pull request. The team reviews and
merges. That converts a two-week task into a two-hour one, and the flat rows
start moving without anyone changing their priorities.

Remove the alternative. Delete the old base image from the registry on a published
date, with a ratchet before it: warn, then fail on changed modules, then fail
outright. As long as the old path exists, staying is free. That is the only part
of this that is a mechanism rather than a request.

And change the reporting. Replace the percentage with a table of service, owner,
blocker, date. A percentage is a number nobody owns. A row with a name on it that
has not changed in three weeks produces a specific conversation instead of a
general one.

The thing I would not do is escalate. Escalation moves one service, once, and
costs credibility I will need again in six months."

**What separates them:** the senior answer treats the stall as a people problem to
be solved with help and pressure. The principal answer treats it as an **incentive
structure** producing exactly the behaviour it rewards, and changes the structure.
The refusal to escalate is a Principal-level instinct: escalation is a
non-renewable resource and it does not scale to nine services.

**Adversarial follow-up:** *"You have no budget for a squad. Now what?"*

Then the honest answer is that you cannot do all four, and you say which one you
drop and what that costs. Without capacity, the ratchet is still available and is
still the highest-leverage single action — but it will produce exceptions and
friction, because you are imposing cost without offsetting it, and you should
predict that out loud rather than be surprised by it. The other move available
without budget is to shrink the scope: decommission the services that do not
justify the migration, and get the denominator down honestly rather than pretend
the estate is smaller than it is. A plan that says "we are migrating 17 services
and retiring 6" is a better plan than one that says "23" and delivers 17.

---

### Q5 — "What breaks when you move a Java 8 application onto a modern JDK without changing any code?"

**A senior answer sounds like:** "Mostly it just works, since class files are
compatible. You can hit issues with reflection into internal APIs since strong
encapsulation, and some old libraries that do bytecode manipulation need
upgrading. Removed APIs like JAXB moved out of the JDK. And some JVM flags were
removed."

That is a genuinely strong answer and it covers the main categories.

**A principal answer sounds like:** "I'd split it into things that fail loudly and
things that change quietly, because the second list is what hurts.

Loud failures, which are fine because you find them in a test run: illegal
reflective access is now denied rather than warned about, so anything doing
`setAccessible` into JDK internals throws — the mitigation is `--add-opens`, which
works and should go on a debt register rather than be treated as a fix. Bytecode
libraries — ASM, cglib, Byte Buddy, older Mockito, older Lombok — refuse class-file
versions they do not know. Removed JVM flags make the JVM refuse to *start*, which
is loud but happens at container start in production if you did not diff your
flags. And APIs that left the JDK — the JAXB and JAX-WS modules — need explicit
dependencies.

Quiet changes, which are the ones that reach customers: the default collector
changed from Parallel to G1, so the latency profile moves and your p99 is not the
number in your baseline any more. Compact strings changed heap footprint. And the
default locale data provider changed to CLDR, so formatted dates, currencies and
number separators can render differently — which surfaces as a failing test if you
are lucky and as a customer complaint about an invoice if you are not.

Which is why the JDK step in my plan is its own deploy with a baseline comparison
against the load harness, not a line item inside a framework upgrade. The whole
point of doing it alone is that when a number moves, there is exactly one thing it
could have been."

**What separates them:** the senior answer produces a correct list. The principal
answer **partitions** it by failure mode — loud versus quiet — and draws the
operational conclusion from the partition. Naming the CLDR locale-data change is
the specific detail that signals having actually done this rather than read about
it, because it is the one that escapes the test suite and reaches a customer.

**Adversarial follow-up:** *"You mentioned `--add-opens` as a mitigation. When
does that stop working?"*

The honest answer: it works today and it is the right immediate mitigation, but it
is a standing exception to a boundary the platform is deliberately closing, and
the direction of travel across recent JDK releases has been to narrow these
escape hatches — the ongoing restriction and deprecation of `sun.misc.Unsafe`
memory access is the clearest signal. So every `--add-opens` in your startup
command is a dated liability, not a solution: it should be in a register with the
owning dependency named and a plan to remove it, and if the dependency that needs
it is unmaintained, that is a blocker-register entry and not a flag. I would also
say plainly that I am not going to quote you a removal date for any specific
escape hatch, because those move — check the JEP index at `openjdk.org/jeps` for
the current state before writing one into a plan.

---

## Mental model checkpoint

Reason these out in writing.

1. The plan's first step is "run on the new JDK with the old framework". State
   the single JVM property that makes this possible. Now construct the case where
   it does *not* hold — what would a codebase have to contain for the JDK-first
   sequence to be unavailable?

2. `javax` → `jakarta` cannot be done incrementally within one deployable unit,
   but *can* be done incrementally across an estate. Explain the mechanism behind
   both halves of that sentence. What property of the JVM's classpath is doing the
   work?

3. You have a shared platform library used by 12 services. Argue for dual-
   publishing (`-2.x` and `-3.x`) and then argue against it. What is the cost you
   are accepting in each direction, and what estate characteristic decides it?

4. The stall at 60% is described as an incentive problem rather than a motivation
   problem. Restate the incentive faced by a team with a product deadline and an
   un-migrated service, from *their* point of view, as if it were correct — because
   it is. Now name the smallest change to that incentive that flips their decision,
   and say what it costs you.

5. A CI gate that fails the build is the plan's forcing function. Name three ways
   that gate can be neutralised by the organisation without anyone technically
   removing it, and what you would do about each.

6. Your rollback for every step is "redeploy the previous artifact". Enumerate the
   conditions that must hold for that to actually work. Which of them can be
   violated by a change that has nothing to do with the migration?

7. The plan sequences the JDK upgrade before the framework upgrade for
   attribution reasons. Construct the strongest argument for the opposite order.
   Under what circumstances would you accept it?

---

## Quick reference card

### The sequence, and why

| Step | Change | Independently valuable? | Rollback |
|---|---|---|---|
| 0 | Inventory + blocker register | Yes — you know what you own | n/a |
| 1 | Build on new JDK, `release=8` | Weak | Revert one property |
| 2 | **Run** on new JDK, old framework | **Strong — patches, G1, cgroups** | **Image tag** |
| 3 | `release=17` | Weak — DX | Property, before modern syntax lands |
| 4 | Dual-publish platform libs | Yes — unblocks per-service work | Consumers stay on old line |
| 5 | **`jakarta` + Boot 3 (atomic)** | Yes, per service | Image tag + schema preconditions |
| 6 | JDK 21 | Yes | Image tag |
| 7 | Virtual threads (**separate**) | Yes | Config flag |
| 8 | JDK 25 | Yes | Image tag |
| 9 | Boot 4.x | Yes | Image tag + serialization preconditions |
| 10 | Delete old baseline | Yes — one estate, not two | Deliberately none |

### The `javax` packages that are NOT Jakarta

Do not rename these. They are JDK APIs and always were:

`javax.sql` · `javax.crypto` · `javax.naming` · `javax.management` ·
`javax.net` / `javax.net.ssl` · `javax.security.auth` · `javax.xml` (JDK parts) ·
`javax.script` · `javax.tools`

This list is the argument for OpenRewrite over `sed` in one glance.

### What a JDK upgrade changes with no code change

| Loud (fails in a test run) | Quiet (reaches production) |
|---|---|
| Illegal reflective access → `InaccessibleObjectException` | Default collector Parallel → G1: latency profile moves |
| Old bytecode libraries reject the class-file version | Compact strings: heap footprint changes |
| Removed JVM flags → JVM refuses to start | CLDR default locale data: dates/currency/number formats render differently |
| Removed JDK modules (JAXB, JAX-WS) → `NoClassDefFoundError` | Default TLS / crypto policy changes |

### Rollback checklist, per step

- [ ] Is the rollback a command, or a reassurance?
- [ ] Did a Flyway/Liquibase migration ship in this release? Is it expand-contract?
- [ ] Did a serialization or message-contract change ship?
- [ ] Is anything cached in a format the old version cannot read?
- [ ] Has this rollback been executed once, in staging, deliberately?

### Stall-prevention, all four required

1. **A date** — D1 warn, D2 fail-on-change, D3 fail and delete the old image.
2. **Enforced in CI**, in the shared parent config, on a new-and-changed-code
   ratchet so it does not fail thousands of existing violations on day one.
3. **Adoption cheaper than non-adoption** — a funded squad that hands over a pull
   request. Costed, with a headcount.
4. **A burndown by owner and blocker.** No percentages.

### Verified facts you may rely on

| Fact | Value |
|---|---|
| Java 21 | LTS, September 2023 |
| Java 25 | **LTS, GA 16 September 2025 — current LTS** |
| Java 26 | GA 17 March 2026, **non-LTS** |
| Spring Framework 7.0 | GA 13 Nov 2025; JDK 17 baseline, JDK 25 recommended; Jakarta EE 11 |
| Spring Boot 4.0.0 / 4.1.0 | GA 20 Nov 2025 / 10 Jun 2026 |
| Spring Boot 3.5.x | **OSS support ended June 2026** — commercial only |

Everything else — per-vendor JDK end-of-support dates, the exact JDK range of a
given Boot patch line, OpenRewrite recipe IDs — look up before writing it down.
Primary sources: `spring.io/projects/spring-boot#support`, `openjdk.org`, your JDK
vendor's support page, `docs.openrewrite.org`.

---

## When would I use this at work?

**1. The day a support-window date lands in your inbox.**
Security sends a finding: your baseline no longer receives free CVE patches. The
default organisational response is to schedule "the upgrade" for next quarter as a
single item. The highest-value ninety minutes you will spend that week is turning
that one item into a sequence where the first step is cheap, valuable and
revertible — because that step can start *this* sprint without a funding
conversation, and it banks the security benefit that the finding was actually
about. The rest of the sequence then gets funded on the strength of a step that
already shipped.

**2. Any time someone proposes a long-lived migration branch.**
This shows up constantly and not only for framework upgrades: a database engine
change, a build-system change, a monorepo split. The question that reframes it is
always the same — "what is the smallest change we can ship to production that is
worth having on its own?" — and the follow-up is "and how do we put it back?"
Asking those two questions in a design review, consistently, is most of what this
topic is for. You will find that half the time the answer reveals a step nobody
had noticed was separable.

**3. When a platform initiative has been at the same percentage for two
quarters.**
This is the most common form of expensive organisational waste and it is nearly
invisible because nobody is doing anything wrong. Being the person who says "this
is an incentive structure, not a motivation problem — here is the mechanism that
changes it, and here are the three services we should decommission instead of
migrate" is a specific, repeatable, high-value contribution. It applies to
migrations, to deprecations, to test-coverage initiatives and to security
remediation programmes, all of which stall in exactly the same shape and for
exactly the same reason.

---

## Connected topics

**Prerequisites — the evidence and the machinery you will cite:**

- **Topic 32 — dependency resolution and BOMs.** Maven's flat, nearest-wins
  classpath is why `javax` and `jakarta` cannot coexist in one deployable unit.
  That single fact determines the plan's granularity.
- **Topic 34 — supply chain, SBOM, CVE triage.** The dependency inventory in step
  0 is an SBOM exercise. The forcing reason in section 1 is a CVE-triage argument.
- **Topic 65 — the load baseline.** Every "how we verify it worked" cell in the
  step table points here. Without it, a migration is a change you cannot measure.
- **Topics 68–72 — allocation, live set, G1, ZGC.** The collector default change
  is the quiet behaviour change in step 2, and the baseline comparison is
  meaningless if you cannot read a GC log.
- **Topic 82 — container awareness.** The JDK step is the right moment to fix
  `MaxRAMPercentage` and `ActiveProcessorCount`, because you are already touching
  the runtime and already re-measuring.
- **Topic 101 — virtual threads.** Step 7 exists as its own step because of what
  this topic taught you about pinning and about where the bottleneck moves.
- **Topic 109 — the pool as the real ceiling.** After a runtime or framework
  upgrade, if throughput did not improve, this is usually why — and it is why the
  baseline comparison must include pool saturation, not just latency.
- **Topic 124 — the readiness review.** Its risk register and runbooks are the
  starting inventory for the estate table.
- **Topic 126 — build vs buy vs adopt.** The blocker register is a series of
  build/buy/adopt decisions made under time pressure. The dependency whose
  upstream died is exactly the risk 126 told you to price at adoption time.

**This unlocks:**

- **128 — monolith to services.** The strangler is the same mechanic applied to
  architecture rather than to versions: each step independently valuable,
  independently revertible, with a forcing function so the old path actually gets
  deleted. If you understood the stall here, you already understand the
  distributed monolith there.
- **129 — capacity and cost.** Every "verified by" cell that says "compare against
  the baseline" needs a model of what the numbers should be. And the JDK step's
  collector change is a direct input to the fleet-sizing model.
- **132 — engineering standards.** The CI ratchet in the forcing function is
  132's machinery, arriving here first because a migration needs it most.
- **133 — postmortems.** When the rollback fails because a Flyway migration
  shipped in the same release, 133 is how you write it up so it prevents the
  class rather than the instance.
- **134 — influence without authority.** Part (c) of the forcing function — the
  squad that hands teams a pull request — *is* 134. This topic is where you first
  need it; that topic is where you learn to build it.
- **135 — the interview simulation.** "Walk me through a migration you planned"
  is a standard principal-loop prompt, and the follow-up is always some version of
  "it stalled — now what?"

---

*Java baseline 21, running on JDK 25, Spring Boot 4.1 / Framework 7.0. The
sequencing mechanics in this topic — class-file forward compatibility, the flat
classpath forcing the namespace change to be atomic per artifact, the stall at the
point where pain stops — are properties of the platform and of organisations, and
they are stable. The specific version numbers and support dates are not. Verify
them at `spring.io/projects/spring-boot#support`, `openjdk.org`, and your JDK
vendor's support page before they go in a document with your name on it.*
