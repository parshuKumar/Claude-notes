# 126 — Technical Strategy: Build vs Buy vs Adopt

## Phase: 12 — Principal Track
## Category: ELITE
## Java baseline: 21  |  Notes features from: 21/25
## Project spine: a five-year total-cost evaluation of one framework choice for `orderflow` — including the cost of getting off it

---

## Mechanical statement

A framework's cost is not its benchmark.

> **Every dependency is a multi-year subscription, paid in engineer-time, in a
> currency no benchmark measures: upgrade effort, onboarding and hiring cost, the
> blast radius of its failure modes, and the cost of getting off it.**

A benchmark measures one workload for one hour on one machine. The subscription
runs for roughly 260 weeks. When people say "we chose X because it was faster",
they have priced one hour and signed for 260 weeks.

The mechanic you are learning has four moving parts, and only the last one is
genuinely hard:

1. **Acquisition** — what it costs to get it working. Everyone estimates this.
   Everyone underestimates it, but at least it gets estimated.
2. **Recurring** — what it costs every year to keep it working, on a calendar you
   do not control. Rarely estimated.
3. **Blast radius** — what it costs when it fails, times how often. Almost never
   estimated.
4. **Exit** — what it costs to remove it. Essentially never estimated, and it is
   the number that determines whether a wrong choice is a bad quarter or a bad
   two years.

The artefact you produce in this topic is the first time you will have written
all four down for the same decision. That is the whole exercise.

---

## The bridge from what you know

### What transfers — completely, and I will not re-teach it

You have evaluated dependencies before, and the following all transfer 1:1:

- **Bus factor.** Counting maintainers, reading the commit history for the last
  twelve months, checking whether issues get answered. Same skill, same signals.
- **Licence review.** Reading the actual `LICENSE`, not the README badge, and
  knowing your organisation's policy. Same skill.
- **"Is this maintained or merely popular?"** You already know that star count is
  a lagging indicator of a decision made three years ago by people who have since
  left. Same skill.
- **Writing the evaluation document and getting it approved.** You have done
  this. It transfers whole.
- **Build-vs-buy as a general commercial argument.** You have made this argument.
  Nothing about it is Java-specific.

Do not spend space in your artefact re-deriving any of that. Spend it on what
follows.

### What does not transfer — the Java-specific cost drivers

| You know (Node/npm) | Java/JVM reality | Verdict |
|---|---|---|
| npm nests dependencies; two versions of a library coexist happily | **Maven puts exactly one version of an artifact on a flat classpath, nearest-wins** | **NO ANALOGUE.** Adopting a library means adopting its transitive version pins *into your classpath*. Two libraries that disagree about Jackson or Netty produce a conflict you own. |
| A version conflict shows up at install time | A version conflict shows up as `NoSuchMethodError` **at runtime, after a clean compile**, sometimes only on a code path you exercise monthly | **NO ANALOGUE** — and it changes what "risk" means in an adoption decision. |
| Frameworks upgrade independently | **The Spring portfolio moves as a unit**, version-managed by a BOM. A dependency inside the portfolio upgrades for free with Boot; one outside it is a separate decision at every Boot bump | **PARTIAL** — the concept of a curated version set exists in Node; the gravity does not. |
| An abandoned package can be forked and patched in an afternoon | An abandoned Java library that never migrated `javax` → `jakarta` is a **hard blocker on the entire framework upgrade** for everyone who depends on it | **NO ANALOGUE** — an ecosystem-wide forced rename with no incremental path. |
| A library is just code you call | Many Java libraries are **proxy-based, reflective, or JVM agents**. Those interact with each other, with AOP (Topic 40), with native image (Topic 83), and with your observability agent | **NO ANALOGUE** — the adoption surface is larger than the API surface. |
| Runtime cost is mostly CPU and event-loop time | A JVM dependency can add **heap live set, threads, a second connection pool, and startup time** — each of which is a line in the capacity model of Topic 129 | **PARTIAL** |
| "We can always swap it" | Persistence and serialization libraries write their assumptions into your **domain model and your stored data**. That is not a swap, it is a migration | **PARTIAL** — worse in Java because the coupling is expressed in annotations spread across every entity. |

Four of those rows are load-bearing for the artefact you will write. In order of
how often they surprise people:

**The flat classpath.** When you adopt library X, you are not adding X. You are
adding X and its opinion about the version of every library it depends on, merged
into a single flat namespace where exactly one version wins. This is why
"lightweight library, small API surface" can still be an expensive adoption. Run
`mvn dependency:tree` on the candidate before you write a single line of the
evaluation. If it drags in a different major version of something you already
have, that is not a footnote; it may be the whole answer.

**Portfolio gravity.** A dependency managed by the Spring Boot BOM costs you
approximately nothing at upgrade time — the version moves when Boot moves, and
someone else did the compatibility testing. A dependency outside the BOM costs
you a compatibility decision at every Boot upgrade, forever. That difference is
worth more, over five years, than almost any performance delta.

**The `jakarta` precedent.** `javax` → `jakarta` was a namespace rename applied to
the entire enterprise Java ecosystem. Mechanically simple; vast. Every library
had to move, and the ones that did not became dead ends — not "less good", but
*incompatible with any modern framework*. That happened once. It is the clearest
possible evidence that "abandoned upstream" in Java is a different severity of
risk than in npm, because you cannot route around it with a nested version.

**Model and data coupling.** A persistence framework does not sit behind your
code; it sits *inside* your domain model, as annotations on every entity, as
lifecycle assumptions in every service method, and as a schema shape in the
database. Topic 48 taught you that the persistence context holds a snapshot of
every loaded entity and writes on dirty check. That behaviour is load-bearing for
how your services are written. Removing it is not a dependency swap.

---

## What is this?

Three options, and two that people forget.

**Build** — write it yourself, own it forever. You control the roadmap and the
API. You pay for every feature, every bug, every security patch, and every
onboarding conversation, for as long as it exists.

**Buy** — pay a vendor. Either a hosted service, or a commercial licence, or
commercial support for something otherwise open source. You convert engineer-time
into money, and you acquire a supplier relationship with a renewal date and a
negotiating position.

**Adopt** — take an open-source dependency. You get the code for free and pay in
integration, upgrades, and dependence on a community whose priorities are not
yours.

The two that get forgotten:

**Do nothing (yet)** — the status quo is always an option and is frequently the
right one. It has a cost too, and the cost must be computed with the same
rigour, or the comparison is invalid. This is the most common flaw in evaluation
documents: the new option is priced carefully and the status quo is described as
"painful".

**Adopt and own** — take the open-source dependency knowing you will fork or
contribute upstream. This is a legitimate strategy and it is sometimes the right
one, but it must be a *decision*, made in advance, with named capacity. What
usually happens instead is that a team adopts, upstream goes quiet, and they
back into ownership eighteen months later with no plan and no capacity. That is
the same outcome arrived at accidentally, which is much more expensive.

### The four cost buckets, defined

Define each precisely, because the artefact depends on them being distinct.

**1. Acquisition cost.** One-time. Engineer-days to integrate, plus licence or
subscription cost in year one, plus the cost of the learning curve for the people
doing the integration. Include the cost of the *decision itself* if it is large —
a three-month evaluation is real money.

**2. Recurring cost.** Per year, forever, until exit. Made of:
- **Upgrade effort.** How often does it release, how coupled is it to the JDK and
  to Boot, is it in the BOM. A library in the Spring BOM is close to zero. A
  library outside it that has a strong opinion about Netty or Jackson is not.
- **Operational cost.** Extra memory in the live set, extra threads, an extra
  pool, extra startup time, an extra thing on the dashboard, an extra thing on
  the on-call rotation. Each of these is an input to Topic 129's model.
- **Cognitive cost.** Every engineer who touches this area now needs to know it.

**3. Blast-radius cost.** Expected annual cost of its failure modes. Ask, for
each failure mode: what fails, how visible is it, how long to detect, how long to
recover, and what does that cost. You do not need precision. You need the *order
of magnitude* and, crucially, the answer to "does this take down `orderflow`, or
degrade one endpoint?" A dependency in the order-placement path and a dependency
in the admin reporting path are not the same risk even if they are the same code.

**4. Exit cost.** What it takes to remove it. Broken into four sub-questions,
because "hard to remove" is not a number:

- **Surface** — how many files import it? Is it behind one interface you own, or
  is it spread across the codebase?
- **Semantic coupling** — does the rest of your code depend on its *behaviour*,
  not just its API? Lazy loading, dirty checking, implicit flushing, transaction
  binding, proxying: these leak into how the surrounding code is written, and
  they do not come out with a find-and-replace.
- **Data and format lock-in** — does it own a schema, a serialization format, or
  persisted state? If yes, exit includes a data migration, which is a different
  order of work.
- **Alternative existence** — is there a second implementation of the same
  abstraction? If the only way out is "write it yourself", exit cost is
  effectively "build cost, later, under pressure".

**The rule:** if you cannot write a number and a method for exit cost, you have
not evaluated the option. You have described it.

---

## Why does it matter?

**1. Because the wrong bet is not recoverable on the timescale people assume.**
A senior engineer who picks a slower library costs the company two instances. A
principal engineer who picks a dead-end library costs the company a quarter, and
then costs it *again* every time the framework moves. The asymmetry is the entire
reason this topic exists: the downside of a bad adoption is bounded by exit cost,
and exit cost is the number nobody computed.

**2. Because "we'll wrap it behind an interface" is said far more often than it is
done.** The wrapper is real work, it is the least interesting work in the
project, and it is the first thing cut when the deadline moves. A principal
engineer either funds the wrapper explicitly or prices the option without it —
and does not let the team have the comfort of an abstraction that does not exist.

**3. Because these decisions are how a team's hiring and onboarding cost is set,
silently, years in advance.** Every unusual choice narrows the pool of engineers
who can be productive in week one and lengthens the ramp for everyone else.
Sometimes that is worth it. It is never free, and the person who made the choice
is usually not the person who pays.

**4. Because the status quo has a cost and nobody computes it.** Staying on
something out of support is a decision, with a price: no free CVE patches. Spring
Boot 3.5 left OSS support in June 2026 — after that date, "we did not decide" and
"we decided to run unpatched" are the same thing operationally, and only one of
them is defensible in an audit.

---

## The decision, framed

> **We have a need. Do we build it, buy it, adopt something, or leave it alone —
> and what would that choice cost us over five years, including the cost of
> reversing it?**

Work it in this order. Steps 1 and 2 kill most proposals cheaply.

### Step 1 — State the need without naming a solution

Write the requirement as a sentence with no product names in it, and attach
evidence from `orderflow`: a Topic 65 baseline number, a Topic 118 metric, a
drill outcome, an incident, or a line from the Topic 124 readiness review.

If you cannot attach evidence, you have a preference, not a need. That is not a
disqualification — sometimes you genuinely foresee a need — but you must label it
as a forecast and say what would confirm it.

The reason this step is first: an evaluation that begins with two candidate
products has already made the important decision (that a product is the answer)
without examining it.

### Step 2 — Is this commodity or is this us?

Two questions:

- **Is this differentiating?** Would a customer ever notice, or care, that we do
  this ourselves? For `orderflow`, connection pooling is not differentiating.
  Pricing and promotion logic is.
- **Is this a solved problem with multiple mature implementations?** If three
  well-maintained libraries do it, building it means competing with three teams
  who do only that.

Building commodity infrastructure is the most reliably expensive mistake in this
topic, and it is expensive in a delayed and invisible way — the bill arrives as
slow onboarding and an unmaintained internal framework, three years later, when
its author has left.

Building the differentiating thing is usually correct, and adopting there is
usually a mistake in the other direction: you contort your domain to fit
somebody else's model.

### Step 3 — Enumerate options including the two that get forgotten

Minimum: build, adopt (name at least two candidates), buy, do nothing. Kill the
obviously bad ones in a sentence each — but *write the sentence*, because a
reviewer's first question is always "why not Y?" and an unstated rejection reads
as an unexamined one.

### Step 4 — Price all four buckets, for every surviving option, including the status quo

This is the artefact. Tables are in the Artefact section below. Two rules:

- **The status quo gets the same table as everything else.** If you cannot fill
  in "do nothing" you cannot compare anything to it.
- **Every number carries a method.** "Roughly 15 engineer-days, estimated from
  the last time we did a comparable integration, which took 12" is a usable
  number. "About two weeks" is not.

### Step 5 — Find the decision's reversibility class

Not all decisions deserve the same rigour. Classify:

- **Reversible in a sprint** — a small library behind one interface. Decide fast,
  do not write a six-page document, get on with it.
- **Reversible in a quarter** — a significant library with wide surface but no
  data lock-in.
- **Effectively irreversible** — anything that owns your data format, your domain
  model, or your deployment topology.

Spend evaluation effort in proportion. A principal engineer who runs a two-month
evaluation on a sprint-reversible decision has wasted two months and taught the
team that decisions are slow. One who runs a two-week evaluation on an
irreversible one has gambled.

### Step 6 — State the decision, the owner, the date, and the review trigger

Same discipline as Topic 125: a verb, an owner, a date, and a falsifiable
condition that would reopen it.

---

## Example 1 — a minimal illustration

The smallest version: **should we add a dependency, or write forty lines?**

### The situation

`orderflow` calls the payment gateway. Topic 111 taught you that naive retries
amplify an outage — three retries is a 4× load multiplier on an already-failing
dependency — and that the minimum correct set is exponential backoff **with
jitter**, plus a circuit breaker, plus a bulkhead.

An engineer proposes adding a small third-party retry library, because writing
backoff-with-jitter correctly is fiddly and the library does it well.

### The bad version of this conversation

"It's a tiny library, it has no dependencies, let's just add it." Decision made
in ninety seconds, which for a genuinely sprint-reversible decision is
approximately the right amount of time — but the reasoning is wrong, and the
wrong reasoning will be reused on a decision that is not sprint-reversible.

### The five-minute version

**Need, without a product name:** "retry the payment-gateway call with
exponential backoff and jitter, bounded by a concurrency budget, and stop
retrying when the breaker is open." Evidence: the Topic 111 drill, where 200
concurrent requests retried in lockstep against a failing stub and produced a
synchronised retry storm you could see in the request-rate graph.

**Commodity or us?** Commodity. Nobody buys `orderflow` for its retry algorithm.

**Options:**

1. **Build it** — roughly forty lines plus tests. Correct jitter is easy to get
   subtly wrong; the failure mode of getting it wrong is invisible until an
   outage.
2. **Adopt the small library** — solves exactly this.
3. **Adopt Resilience4j** — which you already have, from Topic 111, and which
   already provides retry, circuit breaker, bulkhead and rate limiter as a
   coherent set with a defined decorator order.
4. **Do nothing** — no retries. Real option; sometimes correct for a
   non-idempotent call.

**And option 3 ends the conversation.** The decisive question was never "is this
library good?" It was "do we already have something in the codebase that solves
this, that the team already knows, that is already on the dashboard, and that is
already in our upgrade path?" Adding a second retry mechanism costs: two ways to
do the same thing, two sets of metrics, two things in the onboarding document,
and a future engineer wondering which one is authoritative.

**The five-year cost of option 2 is not its API. It is the sentence "we use
Resilience4j, except in the payments module, where we use something else."**
That sentence costs more than forty lines of jitter code, and it never goes
away by itself.

### What you write down

> Use Resilience4j's retry with jitter, decorated in the order retry-outside-
> breaker, bulkhead innermost (Topic 111). We already depend on it, it is
> version-managed alongside our Spring upgrades, its metrics are already on the
> USE dashboard, and every engineer who has touched the payment path knows it.
> The alternative library is good and irrelevant: adding a second retry
> abstraction costs us a permanent "except in payments" clause for a marginal API
> preference.

Notice the shape. The winning argument is not about quality. It is about
**consistency, upgrade coupling, and cognitive cost** — three of the four buckets,
none of which appear in a benchmark. That shape scales up unchanged to the real
decision below.

---

## Example 2 — the real decision on the project spine

### The setting, with real constraints

`orderflow` is a production-shaped service on Java 21 (running JDK 25), Spring
Boot 4.1 / Framework 7.0, persisting to Postgres through Spring Data JPA and
Hibernate. You have:

- a recorded p50/p95/p99 baseline from Topic 65, on a realistic dataset — at
  least 100k products, 1M orders, 5M order lines, with skew;
- six topics' worth of scar tissue on Hibernate specifically: the persistence
  context and its per-entity snapshot (48), lazy loading and
  `LazyInitializationException` (49), N+1 detection by query counting (50),
  second-level cache invalidation (51), optimistic and pessimistic locking on
  inventory and wallet (52), and the `IDENTITY`-disables-batching trap (53);
- a connection pool tuned against that baseline, where you learned the pool — not
  CPU — is usually the ceiling (109);
- RED and USE metrics (118) and a production-readiness review (124).

### The proposal on the table

An engineer proposes moving `orderflow`'s persistence layer off JPA/Hibernate to
a SQL-centric library — jOOQ, or Spring Data JDBC, or `JdbcClient` directly. The
case made in the proposal:

- the read paths are the hot paths (70% of the Topic 65 scenario mix is catalogue
  read, 20% order read);
- most of the Phase 5 pain — N+1, lazy exceptions, cache invalidation, snapshot
  memory — is *caused by* the persistence context, not by SQL;
- generated SQL is opaque; hand-written SQL is inspectable;
- a benchmark shows the alternative is substantially faster on a query-heavy
  workload.

Every one of those points is true. That is what makes this a real decision and
not a strawman.

### Step 1 — the need, without a product name

> "Reduce p95 on `GET /orders` and `GET /products`, and reduce the recurring cost
> of persistence-layer defects, which have been the largest single category in
> our last two quarters of incidents."

Evidence to attach — and you have all of it:

- the Topic 65 baseline p95 for those two endpoints;
- the query counts from the Topic 50 drill, before and after the N+1 fix;
- the Topic 124 readiness review's top-three risks, if persistence is among them;
- an actual count of persistence-related defects from your issue tracker. If that
  count is small, the second half of the need is fiction and must be struck.

**This is where most proposals of this kind quietly die**, and it is worth
sitting with. "Persistence bugs are our biggest category" is an extremely common
claim and is often false. Go and count. If it is false, the whole document
collapses to a narrower and much better proposal about two endpoints.

### Step 2 — commodity or us?

Object-relational mapping is commodity. Nobody buys `orderflow` for its data
access. So building is out; the real choice is between two adoptions and the
status quo.

### Step 3 — the option set

1. **Status quo** — JPA/Hibernate everywhere.
2. **Full replacement** — remove JPA, use a SQL-centric library for everything.
3. **Split by direction (CQRS-lite)** — keep JPA for writes, where the
   persistence context, dirty checking and optimistic locking (Topic 52) are
   genuinely doing work; use a SQL-centric library for read paths, behind the
   existing repository interfaces.
4. **Status quo plus targeted fixes** — keep JPA, and fix the specific read paths
   with projections and entity graphs (Topic 47, Topic 50), which you already
   know how to do.

Options 3 and 4 are the ones a senior evaluation usually omits, because the
proposal arrived framed as a binary. Reframing a binary into four options is one
of the highest-value things a principal engineer does to a document, and it costs
one paragraph.

### Step 4 — the four buckets, per option

Do not fill these in from my imagination — the tables in the Artefact section are
blank for exactly that reason. What follows is the *reasoning* you must do for
each cell, on your own numbers.

**Acquisition.** For option 2, this is not "learn a new library". It is: rewrite
every entity, every repository, every service method that relies on dirty
checking, and every test that relies on transactional rollback semantics. Count
the files. `grep` for `@Entity`, for `@Transactional`, for repository interfaces.
That count is your acquisition estimate's denominator, and it is usually two to
five times what people guess.

For option 3, acquisition is bounded by the read paths only, and — critically —
by whether your repository interfaces are already an abstraction you own. If
services depend on `OrderRepository` and not on `EntityManager`, option 3 is
cheap. If services take `EntityManager` directly anywhere, that is a leak you
must price.

**Recurring.** Now the Java-specific driver: **is the candidate in the Spring Boot
BOM?** Spring Data JPA and Spring Data JDBC ride along with Boot upgrades — the
compatibility testing is done by someone else. A third-party library outside the
BOM is a compatibility decision at every Boot upgrade, forever, and a potential
blocker on a security-driven upgrade. Over five years and roughly one major Boot
upgrade plus several minors, that difference is not small, and it is invisible in
every benchmark.

Also recurring: code generation. Some SQL-centric libraries generate type-safe
Java from your schema at build time. That is a genuine benefit and a genuine
build-pipeline dependency: the build now needs a database or a schema dump. Price
the CI change. Ask what happens when the generation step breaks on a Friday.

**Blast radius.** Compare honestly. Hibernate's failure modes you *know*, in
detail, and you have runbooks for them from Phases 5 and 11. That is worth
something real: a known failure mode with a runbook is cheaper than an unknown
one, even if the unknown one is rarer. Write that down explicitly, because it is
the argument the proposal will not have made.

The other side of the ledger: Hibernate's failure modes are *silent*. An N+1 does
not throw; it just makes p95 worse under load. A lazy-load leak does not throw
until the session closes. A second-level cache serving stale data after a native
write (Topic 51) is silently wrong. Silent failure modes have long detection
times, and detection time is most of incident cost.

**Exit.** The number the proposal will not contain, computed for each option.

- **Exit from the status quo (JPA)**: this is option 2's acquisition cost. They
  are the same number. Write it once and use it twice — and note that this means
  *you already know the status quo's exit cost*, which is unusual and useful.
- **Exit from option 2**: back to JPA, which means re-adding entity mappings you
  deleted. High, and made worse by the fact that any team that did this once will
  be politically unable to do it again.
- **Exit from option 3**: low. Read paths are behind repository interfaces you
  own; reverting one read path is one class. **That is the whole argument for
  option 3**, and it is a reversibility argument, not a performance one.

### Step 5 — reversibility class

- Option 2: **effectively irreversible** within the planning horizon. Domain model
  coupling plus organisational commitment.
- Option 3: **reversible in a sprint, per endpoint**, if and only if the
  repository interfaces hold.
- Option 4: reversible immediately.

### Step 6 — the position

A defensible conclusion, written the way it goes in the document:

> **Recommendation: option 4 first, then option 3 for the two endpoints where it
> still pays. Not option 2.**
>
> The performance case is real but narrow: it applies to two read endpoints that
> are 90% of our request mix. We have not yet done the cheap version — projections
> and explicit entity graphs on those two endpoints (Topics 47, 50) — and we
> should not adopt a new persistence stack before we have measured what the
> cheap version buys. That is two engineer-days and it is measurable against the
> Topic 65 baseline directly.
>
> If the cheap version leaves us short of the p95 budget from Topic 129, we adopt
> a SQL-centric library for **read paths only**, behind the existing repository
> interfaces. That is option 3. Its exit cost is one class per endpoint, which is
> the property that makes it safe to try.
>
> We should not do option 2. The write paths depend on behaviour we would have to
> reimplement — optimistic locking on inventory and wallet from Topic 52, and
> transactional write ordering — and a full replacement converts a bounded
> performance question into an unbounded domain-model rewrite whose exit cost is
> the same rewrite in reverse. The performance benchmark cited compares a
> query-heavy synthetic workload; our mix is 70% catalogue read against a cached
> hot set (Topic 110), so it does not describe our access pattern.
>
> **Owner:** the payments/orders team lead. **Date:** measurement complete within
> two weeks.
> **Review trigger:** if after option 4 the p95 on either endpoint is still above
> budget, we move to option 3 on that endpoint only. If we ever find ourselves
> proposing option 3 on more than four endpoints, that is the signal to re-open
> the full question with better evidence than we have today.

Notice the moves:

- The binary became four options.
- The cheap option is tried first, because it is measurable and reversible.
- The expensive option is scoped to where its exit cost is lowest.
- The benchmark is not dismissed; it is **relocated** — "it does not describe our
  access pattern" is a specific, falsifiable objection, not a general suspicion of
  benchmarks.
- The status quo is given its genuine strength (known failure modes, existing
  runbooks) rather than being described as pain.
- There is a trigger that would reopen the big question, so nobody has to have
  this argument informally in six months.

---

## Wrong approach → exact symptom → root cause → fix

Organisational symptoms. No stack traces. What you would have *seen*.

---

### Wrong approach 1 — adopting on a benchmark, from a project with one maintainer and no LTS policy

**Wrong:** a library is adopted because it wins a published benchmark by a wide
margin. Nobody checks the maintainer count or whether the project has ever
published a support policy.

**Exact symptom — what you would have SEEN:**

- Eleven months in, a CVE lands in one of the library's transitive dependencies.
  You go to file an issue and find the last upstream commit was fourteen months
  ago, there are 60 open issues, and three of them are the same CVE reported by
  three different companies.
- Your `pom.xml` acquires a `<dependency>` with a version like `2.4.1-orderflow-1`
  and a comment saying "forked, see PLAT-2291". That comment is now the most
  important line in the build file and nobody outside the platform team knows it
  exists.
- The next Spring Boot upgrade takes six weeks instead of one, because the forked
  library pins a transitive dependency at a version the new Boot BOM has moved
  past, and Maven's flat classpath means one of the two has to lose.
- In the retrospective someone says "we should check maintenance status before
  adopting", it is written down, and it does not become a check anywhere.

**Root cause:** the evaluation measured the one dimension that was easy to
measure. Performance is a number; governance is a judgment, and judgments lose to
numbers in meetings unless someone forces them onto the same page. Nobody asked
the two governance questions: how many people can merge to this repository, and
has this project ever stated a support policy?

**Fix:**

1. Make governance a **gate, not a factor**. Before performance is discussed at
   all: how many committers merged in the last twelve months; is there a written
   support or LTS policy; is it in the Spring BOM or a comparable curated set; and
   what happened the last time the ecosystem forced a breaking change on
   everyone — did this project make the `jakarta` move promptly, or late, or not
   at all? That last question is a superb, cheap, historical test of a project's
   responsiveness, and the answer is public.
2. Price the fork explicitly in the evaluation: "if upstream goes quiet, we own
   this; that is N engineer-days per year." If nobody will fund N, the option
   fails the gate.
3. Prefer, all else close to equal, the dependency that upgrades for free with
   your framework. That preference is worth more than a benchmark delta, and you
   should be able to say why in one sentence.

---

### Wrong approach 2 — "we'll wrap it behind an interface" and then not

**Wrong:** the evaluation's risk section says exit cost is low because the
library will sit behind an interface the team owns. The interface is planned for
"after the first milestone".

**Exact symptom — what you would have SEEN:**

- The abstraction exists, and its method signatures use the vendor's types.
  `Optional<VendorResult> execute(VendorQuery q)`. It is not an abstraction; it is
  a package-private alias.
- Or the interface exists for 80% of usage and there are eleven direct imports
  bypassing it, all added under deadline pressure, all with a review comment
  saying "we'll clean this up".
- The exit-cost estimate in the original document said "two weeks". The actual
  attempt, when it comes, stalls in week three with a spreadsheet of 140 call
  sites.
- The tell you can see today, without waiting: **`grep` for the library's package
  name across the codebase.** If it appears outside the one module that is
  supposed to own it, the abstraction is fictional and the exit estimate is
  fictional with it.

**Root cause:** the abstraction was a *promise made in a document*, and documents
do not compile. Under pressure, teams do the thing that ships, and going through
an indirection that adds no immediate value is the first thing dropped. The
evaluation credited a discount for work that was never funded or enforced.

**Fix:**

1. If exit cost depends on an abstraction, **the abstraction ships in the same
   pull request as the adoption**, or the evaluation prices the option without the
   discount. No IOUs.
2. Enforce it mechanically: an architecture test that fails the build when the
   library's package is imported outside its owning module. This is Topic 132's
   machinery, and this is the second time in Phase 12 you have needed it.
3. Design the abstraction around **your domain's concepts**, not the vendor's. If
   the interface method is `chargeWallet(WalletId, Money)` you have an
   abstraction. If it is `executeVendorCommand(...)` you have a rename.

---

### Wrong approach 3 — building commodity infrastructure

**Wrong:** the team builds an internal framework — a bespoke configuration
system, a homegrown ORM layer, a custom HTTP client wrapper — because the
existing options did not quite fit.

**Exact symptom — what you would have SEEN:**

- A 9,000-line internal module with one confident expert and a wiki page last
  edited two years ago.
- New joiners taking three months to be productive instead of three weeks, and
  the exit interviews of the ones who leave mentioning "the internal framework"
  by name.
- A job posting that cannot list a marketable skill for a third of the codebase.
- A recurring "we should document the framework" item that appears in every
  quarterly plan and is never done, because documenting it is not shipping.
- The original author has left, and the module's last substantive commit predates
  their departure.
- The decisive one: a feature that would be a one-line configuration change in a
  mainstream library takes four days, because it has to be built.

**Root cause:** the requirement was 90% commodity and 10% specific, and the
decision was made on the 10%. The 90% is where all the maintenance cost lives —
edge cases, thread safety, observability, documentation, upgrade compatibility —
and it is precisely the part that a mature library has already paid for. The team
priced the build against the 10% they cared about and inherited the 90% they did
not.

**Fix:**

1. Split the requirement: what is genuinely ours, and what is commodity? Adopt
   for the commodity part and build a thin layer for the specific part. Almost
   every "nothing quite fits" is actually "one thing does not fit".
2. Apply the hiring test explicitly in the evaluation: *how many engineers we
   could hire already know this?* If the answer for a build is "none, by
   definition", that is a permanent cost line in the recurring bucket, and it
   should be stated in engineer-days of ramp per hire, per year, at your expected
   hiring rate.
3. If you do build, decide up front what would make you stop and adopt later, and
   write the trigger down. Internal frameworks almost never get retired because
   nobody ever defined the condition for retiring them.

---

### Wrong approach 4 — buying without pricing integration or exit

**Wrong:** a managed service is bought because the subscription is cheaper than
the estimated build cost. The comparison is subscription-versus-build.

**Exact symptom — what you would have SEEN:**

- The integration takes three times the estimate, because the estimate covered the
  happy path and not authentication, network policy, data residency, the on-call
  runbook, the staging environment, or the failure behaviour when the vendor is
  degraded but not down.
- A new dashboard and a new alert nobody owns, and an incident where the first
  twenty minutes are spent establishing whether the vendor is down, because there
  is no dependency-health signal in your own monitoring.
- At renewal, the price rises substantially and the negotiation goes badly,
  because everyone in the room knows there is no alternative ready. You have no
  BATNA — no walk-away option — and the vendor knows it too.
- Your data is in the vendor's format, and the export API is paginated, rate
  limited, and lossy in exactly the field you need.

**Root cause:** buy-versus-build was priced as subscription-versus-salary.
Integration cost, operational surface and exit cost were all zero in the model.
And exit cost was zero not because it is small but because nobody asked.

**Fix:**

1. Price the **whole** buy: subscription, plus integration, plus operational
   surface, plus the vendor's place in your incident response, plus the exit.
2. Before signing, answer in writing: *if we had to leave in 90 days, what would
   we do?* If the answer is "we could not", you are not buying a service, you are
   acquiring a dependency with a renewal date, and the price at renewal is
   whatever they say it is.
3. Keep the exit cheap deliberately where it matters: own your data in your own
   format, use an interface that describes your domain rather than theirs, and —
   for anything genuinely critical — keep a documented degraded mode that does not
   require the vendor at all.

---

### Wrong approach 5 — never deciding, and running two of everything

**Wrong:** a decision is repeatedly deferred, so different teams and different
modules each pick their own answer. Nobody ever says no.

**Exact symptom — what you would have SEEN:**

- Two HTTP clients, two JSON libraries and two metrics facades in the same
  deployable, and a `mvn dependency:tree` output that takes a full screen to read.
- A `NoSuchMethodError` in production, from a clean build, because a transitive
  bump changed which version won on Maven's flat classpath (Topic 32). It appears
  on one endpoint, intermittently, and takes a day to diagnose because the stack
  trace names a method that exists perfectly well in the source you are reading.
- An onboarding document containing the phrase "we use X, except in the payments
  module".
- A CVE response that has to be done twice, because there are two libraries doing
  the same job and the scanner found the vulnerable one in the module nobody
  remembered.
- Two people who each believe they are following the standard, and both are.

**Root cause:** avoiding a decision feels lower-risk than making one, because the
cost of no decision is diffuse and delayed while the cost of a wrong decision is
concentrated and attributable. That incentive is real and it is why this failure
is so common. It is also why "we should standardise" as a suggestion never works:
suggestion is exactly what produced the mess.

**Fix:**

1. Make the choice, write it down as a standard with a rationale, and — this is
   the part that matters — **give the losing option a migration path and a
   deadline**. A standard without a migration path is an opinion.
2. Enforce at the build: a dependency-convergence rule and a banned-dependency
   list that fails the build, not a wiki page. Topic 132 covers introducing this
   on a legacy codebase without a three-month freeze; the short version is that
   you gate new and changed code first with a ratcheting baseline.
3. Accept that some decisions are wrong and are still better than no decision.
   One consistent, slightly suboptimal choice beats two good ones, because the
   cost you are avoiding is not technical, it is the permanent tax of every
   engineer needing to know which world they are in.

---

## Artefact — what you must produce

### Specification

**Title:** `<Framework/library choice> for orderflow — five-year evaluation`

**Length:** 1,500–2,500 words, plus tables. Tables do not count toward the word
budget; prose does. If the prose exceeds 2,500 words you are explaining rather
than deciding.

**Subject:** one framework-scale choice for `orderflow`. It must be big enough
that exit cost is a real number. Candidates that qualify:

- persistence: JPA/Hibernate versus a SQL-centric library (Example 2's decision);
- messaging: Kafka versus a simpler queue for the outbox relay (Topics 113–115);
- caching: the current Spring Cache + Redis arrangement versus an in-process
  cache, or both (Topic 110);
- resilience: Resilience4j versus the platform's service mesh doing it (Topic 111,
  Topic 112);
- deployment/runtime shape: JVM versus native image, which is simultaneously a
  cost, startup and capability decision (Topic 83, and it feeds Topic 129);
- observability: self-hosted metrics and tracing versus a vendor (Topics 118–119).

Things that do **not** qualify because their exit cost is trivial: a utility
library, a test assertion library, a code formatter. Those are good decisions to
make quickly and bad subjects for this artefact.

**Required sections, in this order:**

**1. The need, with evidence.** One sentence, no product names. Then the
evidence: Topic 65 baseline numbers, Topic 118 metrics, a drill outcome, a
readiness-review risk from Topic 124, or a defect count from your tracker. If the
evidence is a forecast rather than a measurement, label it as a forecast and say
what would confirm it.

**2. Commodity or differentiating.** Two paragraphs. Would a customer notice?
How many mature implementations exist?

**3. Options considered.** At least four, and the status quo is always one of
them. One paragraph each for the ones you reject early, stating the reason.

**4. Cost tables.** One table per surviving option, **including the status quo**.
Fill these in from your own estimates and your own measurements. Every number
carries a method in the Method column — that column is not optional and it is the
first thing I read.

*Table A — Acquisition (one-time)*

| Item | Estimate | Method / basis | Confidence |
|---|---|---|---|
| Integration engineering (engineer-days) | | | |
| Migration of existing code (engineer-days) | | | |
| Test rewrite (engineer-days) | | | |
| Licence / subscription, year 1 | | | |
| Learning curve for the implementing team | | | |
| Cost of the evaluation itself | | | |
| **Total** | | | |

*Table B — Recurring (per year)*

| Item | Estimate | Method / basis | Confidence |
|---|---|---|---|
| Upgrade effort per year (engineer-days) | | | |
| — is it version-managed by the Spring BOM? (yes/no) | | | |
| — how many releases per year, and are they breaking? | | | |
| Operational surface: extra memory in live set | | measured, not guessed — see Topic 70 | |
| Operational surface: extra threads / pools | | | |
| Operational surface: startup time delta | | Topic 122 | |
| Dashboards / alerts added, and their owner | | | |
| Onboarding delta (days per new engineer × hires/year) | | | |
| Licence / subscription, per year | | | |
| **Total per year** | | | |

*Table C — Blast radius*

| Failure mode | What breaks in `orderflow` | Detection time | Recovery time | Frequency estimate | Basis |
|---|---|---|---|---|---|
| | | | | | |
| | | | | | |
| | | | | | |

Include at least one *silent* failure mode — one that does not throw. Those are
the expensive ones, and a table with only loud failures is a table that has not
thought hard enough.

*Table D — Exit*

| Question | Answer | Basis |
|---|---|---|
| Files importing it directly (count) | | `grep` output, actual number |
| Is it behind an interface we own? | | and is that enforced by a build rule? |
| Semantic coupling (what behaviour does surrounding code rely on?) | | |
| Data / format lock-in? | | does exit require a data migration? |
| Does a second implementation of this abstraction exist? | | |
| **Exit estimate (engineer-weeks)** | | |
| Who has actually done this migration, and what did it cost them? | | |

**5. Five-year summary.** One table comparing options on: total five-year cost,
reversibility class, the largest single risk, and the thing that would most
change the answer.

**6. Recommendation.** Verb, scope, owner, date.

**7. Review trigger.** What fact, with a threshold or a date, reopens this?

**8. What I am least sure about.** One paragraph, and write it honestly. Naming
your own weakest number is the strongest thing in the document, because it tells
the reviewer you know where the document is thin — and it means the review can
start there instead of spending twenty minutes finding it.

### Constraints

- **No invented numbers.** Every figure is measured, quoted with attribution, or
  estimated with a stated method. "Estimated at 15 days by analogy with the
  Testcontainers migration, which took 12" is legitimate. "About three weeks" is
  not.
- **No vendor benchmarks as load-bearing evidence.** You may cite one, attributed
  as the vendor's claim, and you must state the workload it measured and whether
  it resembles the Topic 65 scenario mix. If it does not resemble it, say so and
  stop using it.
- **The status quo gets a full table.** Not a paragraph of complaints.
- **Popularity is not an input.** No star counts, no "everyone is using it". If a
  large user base matters to you, say *why* — usually it is a proxy for "bugs get
  found by someone other than us", which is a real argument you should make
  directly.

---

## How I will review it

Three questions break a document of this kind. I ask them in this order.

### Question 1 — "What is your exit cost, as a number, and how did you get it?"

This is where most of these documents fail, and they fail in a predictable way:
the exit section exists, and it contains adjectives.

What I am testing: whether you have modelled this as a reversible bet or an
irreversible one, and whether you know which.

**How the document fails:**
- "Exit cost is low because we'll wrap it behind an interface" — with no
  interface in the plan, no build rule enforcing it, and no line item funding it.
  I will ask which pull request contains the interface. If the answer is "the
  follow-up one", I strike the discount and ask for the number without it.
- "Exit cost is moderate" — an adjective where a number belongs.
- An exit cost computed as "remove the library and put back the old one",
  ignoring that the old one's mental model has left the team along with the
  engineers who knew it.

**What a strong answer looks like:** a count of import sites from an actual
`grep`, a named enforcement mechanism, an engineer-week estimate with a method,
and an honest note on semantic coupling — the behaviour the surrounding code
relies on that does not appear in any import statement. Lazy loading. Dirty
checking. Transaction binding. Implicit flush ordering. Those are the things that
make an exit a rewrite instead of a refactor, and naming them is the difference
between a real estimate and a hopeful one.

### Question 2 — "Your performance claim: what workload, at what data size, on whose hardware, and does it match our Topic 65 profile?"

What I am testing: whether the number that motivated this document describes
`orderflow` or describes something else.

**How the document fails:**
- A benchmark from the project's own site, with no workload description.
- A benchmark whose access pattern is nothing like yours — for instance a
  write-heavy or a small-dataset benchmark, when your Topic 65 mix is 70%
  catalogue read over 100k products with a hot skew and a cache in front of it
  (Topic 110).
- A benchmark with no concurrency stated, when you learned in Topic 65 that
  throughput without concurrency and percentile is not a measurement, and that
  coordinated omission makes naive load tests under-report tail latency badly.
- A performance argument that survives after you notice the bottleneck is
  elsewhere. You already know from Topic 109 that the pool is usually the ceiling
  and from Topic 55 that a transaction holding a connection through an HTTP call
  can take out the whole service. If the candidate makes CPU work faster and CPU
  was not the constraint, the improvement is zero and the document should say so.

**What a strong answer looks like:** either a measurement you took on your own
workload — even a rough one on the Topic 65 harness, which you already have — or
an explicit statement that the performance case is unverified and the decision
does not rest on it. Both are respectable. What is not respectable is a
benchmark used as evidence for a workload it never measured.

**The strongest version:** "we ran the candidate against our own load harness on
the two endpoints that matter and the delta was X at p95, measured this way." You
have that harness. Two days of work converts the weakest part of the document
into the strongest.

### Question 3 — "What is the five-year cost of the option you rejected, computed exactly the same way?"

What I am testing: whether this is an evaluation or an argument for a conclusion
you had already reached.

**How the document fails:** the preferred option has four tables and a method
column. The status quo has a paragraph describing pain. That is not a comparison;
that is a case. And because it is a case, the reviewer cannot tell whether the
conclusion follows from the analysis or the analysis followed from the
conclusion.

**What a strong answer looks like:** the status quo has the same tables, and its
strengths are stated in its own voice — known failure modes, existing runbooks,
existing expertise, zero migration risk, existing metrics coverage. Then the
recommendation beats it on the numbers, or the recommendation is the status quo,
which is a completely legitimate and under-produced outcome of an evaluation.

A document that concludes "keep what we have, and here are the two targeted fixes
that address the actual complaint" is often the best possible output of this
process. It is also the one people are least willing to write, because it feels
like it wasted the effort. It did not: it converted an argument into a decision,
and it stopped a quarter of work.

### Smaller things I will also pick at

- **Onboarding and hiring cost missing from Table B.** Almost always absent.
  This is a real recurring cost of every unusual choice and it compounds with
  headcount growth.
- **No silent failure mode in Table C.** If everything in your blast-radius table
  throws an exception, you have not thought about detection time.
- **Transitive dependency conflicts unexamined.** Did you run `mvn
  dependency:tree` on the candidate? If it brings a different major version of
  something you already have, on Maven's flat classpath that is a real cost and
  possibly a blocker (Topic 32).
- **Sunk cost.** If any sentence begins "we've already invested", I will strike
  it. Money already spent is not an input to a decision about the future, and its
  appearance is a reliable signal that the decision has already been made
  emotionally.
- **Two decisions in one document.** "Adopt X, and also restructure the payment
  module." Split it.
- **No confidence column.** A number without a confidence level cannot be
  challenged proportionally, so all challenges land equally and the review is
  worse.

### What I will not attack

Prose, formatting, table styling, or the conclusion itself. I have no stake in
which option wins. I am attacking the weakest assumption the recommendation rests
on. If your weakest assumption is clearly labelled in section 8 and you have
already priced the risk of it being wrong, the review will be short — and that is
the outcome you are working toward.

---

## Interview questions (Senior → Principal)

> The rubric line from the master plan:
> *"library X is faster in benchmarks" → "X is faster on a synthetic benchmark
> that doesn't match our access pattern; it has one maintainer, no LTS policy,
> and no migration path off it. Y is 15% slower and I can hire people who know
> it. The 15% costs us two instances; the wrong bet costs us a quarter."*

---

### Q1 — "This library benchmarks three times faster than what we use. Should we switch?"

**A senior answer sounds like:** "It depends on whether that benchmark reflects
our workload. I'd want to run it against our own load test before deciding. And
I'd check whether the library is well maintained, and how much of our code would
have to change."

That is a genuinely good answer. Everything in it is correct.

**A principal answer sounds like:** "Three questions, and the first one might end
it.

What is the constraint we are actually hitting? From our Topic 109 work,
`orderflow` is pool-bound before it is CPU-bound on the order path, and from
Topic 55 we know a transaction that holds a connection through a downstream call
is worth more than any library delta. If the candidate makes CPU faster and CPU is
not our ceiling, three times faster is three times faster at something we are not
waiting on, and the whole conversation is moot. That is a two-hour check against
data we already have.

Second, if it *is* our constraint: what does that benchmark measure? Our mix from
Topic 65 is 70% catalogue read over a hot skew with a cache in front of it. A
benchmark on a cold, uniformly-distributed, write-heavy workload is a different
program. I would rather spend two days running the candidate on our own harness
than two weeks arguing about somebody else's numbers, and we already have the
harness.

Third, the part that is not about performance at all: is it version-managed with
our framework, how many people can merge to it, and what did it do when the
ecosystem forced `javax` to `jakarta` on everyone? Because our real risk is not
being 15% slower. Being 15% slower costs us a couple of instances, and I can put
a number on that from our cost model. Adopting a dead end costs us a quarter, and
it costs it at the worst possible time — when a CVE forces an upgrade we cannot
do."

**What separates them:** three things. The senior answer validates the benchmark;
the principal answer first asks whether performance is the binding constraint at
all — which is the Topic 129 habit of finding the ceiling before optimising.
Second, the principal answer prices both sides of the asymmetry in the same units:
slow is a recurring instance cost, wrong is a one-off quarter. Third, the
governance question is a gate, not a factor.

**Adversarial follow-up:** *"You've reframed it as a governance question. What if
the project has one maintainer who is excellent and extremely responsive?"*

The honest answer accepts that this is a real and common case, and does not
pretend a bus factor of one is automatically disqualifying. It shifts to: what is
our plan when that person stops? Can we fork and maintain it — do we have someone
who could, and would we fund it? If yes, adopt with eyes open and write the
contingency into the document. If no, the answer is no regardless of how good the
maintainer is, because their excellence is not a property we control.

---

### Q2 — "We're on a framework line that just went out of open-source support. Do we buy commercial support or upgrade?"

This is a live decision, not a hypothetical: Spring Boot 3.5 left OSS support in
June 2026, which means no free CVE patches for anyone still on it.

**A senior answer sounds like:** "We should upgrade — running unsupported means
no security patches. Commercial support is a stopgap. I'd plan the upgrade for
next quarter."

**A principal answer sounds like:** "Both, in that order, and they are answers to
different questions.

Commercial support buys *time*, and time is what the upgrade needs. So the
question is not which one; it is how much time do we need and what is it worth.
If the upgrade is three months of work across eleven services, then a support
contract that covers those three months is cheap insurance against being the
company that shipped an unpatched CVE because the migration slipped. Buying it is
not an alternative to upgrading; it is a way of not being forced to rush the
upgrade, which is how upgrades break things.

But I would put a hard expiry on it in the same decision, in writing, because
the failure mode here is very well known: support gets renewed, the upgrade gets
deprioritised because the pain went away, and three years later you are two major
versions behind with a much bigger migration and a much worse negotiating
position at renewal. That is exactly the stall pattern from our migration
planning — teams stop when the pain stops. So the contract is bought *with* a
migration plan that has a deadline enforced in the build, not alongside a
migration intention.

And I would size the upgrade properly before choosing the contract length, not
after. Ordering matters: the sequence is JDK first while staying on the current
framework, then the mechanical namespace work, then the framework. Each of those
is independently valuable and independently revertible, which is what makes the
estimate believable."

**What separates them:** the senior answer treats it as a fork in the road. The
principal answer recognises that the two options operate on different variables —
one buys risk coverage, one removes the risk — and combines them with a mechanism
that prevents the well-known failure mode of the combination. Also note the last
paragraph: sizing before committing to a contract length, rather than picking a
term and hoping.

**Adversarial follow-up:** *"Your CFO says the support contract is expensive and
asks why we can't just be careful. What do you say?"*

The wrong answer is technical. The right answer converts it: without patches, a
critical CVE means an emergency upgrade under time pressure, which is the most
expensive and highest-risk way to do the work we already have to do. The contract
prices that risk down for a known period. Then give the number of engineer-weeks
in the plan, so the CFO can see the contract is buying a specific, bounded amount
of schedule, not an indefinite subscription.

---

### Q3 — "When should a team build something in-house?"

**A senior answer sounds like:** "When it's core to our business, when nothing
off-the-shelf fits our requirements, or when the existing options are too heavy
for what we need."

**A principal answer sounds like:** "I use two tests, and then I check the answer
against a third thing that people miss.

First test: would a customer ever notice we do this ourselves? For `orderflow`,
pricing and promotion logic — yes, that is us, we should own it, and adopting
there means contorting our domain to somebody else's model. Connection pooling —
no, and building it would be an act of vandalism.

Second test: how many mature implementations already exist? If three, building
means competing with three teams who do only that, and we will lose slowly and
invisibly.

Then the thing people miss: 'nothing quite fits' is almost always 'one thing does
not fit'. The requirement is 90% commodity and 10% specific, and the build gets
justified on the 10% while the maintenance bill comes from the 90% — thread
safety, edge cases, observability, documentation, upgrade compatibility. The
right move is nearly always adopt the commodity part and build a thin layer for
the specific part.

And if we do build, I want the retirement condition written down at the start:
what would make us stop and adopt instead. Internal frameworks do not get retired
because nobody ever defined what retiring them would look like, and the cost
shows up years later as onboarding time and as a hiring pitch we cannot make."

**What separates them:** the senior answer gives correct criteria. The principal
answer gives criteria, then names the specific reasoning error that makes teams
apply them wrongly, then adds an exit condition to a build decision — which
almost nobody does, because a build feels permanent by nature. Treating your own
build as a dependency with an exit cost is the move.

**Adversarial follow-up:** *"You built the thin layer. Two years later it's
4,000 lines. What happened and what do you do?"*

What happened is that thin layers accrete, one reasonable request at a time. What
you do is re-run this evaluation with the layer as the status quo — which means
you now have real data on its cost, which is better data than you had originally.
The mistake is defending it because you built it. Sunk cost gets struck here just
like anywhere else.

---

### Q4 — "How do you evaluate a dependency's long-term risk?"

**A senior answer sounds like:** "I look at maintenance activity, the number of
contributors, release frequency, how quickly issues get closed, whether it has a
security policy, and the licence."

Correct, complete, and this is a good senior answer.

**A principal answer sounds like:** "That list, and then two Java-specific things
that dominate it.

The first is coupling to our framework's release train. A dependency inside the
Spring Boot BOM upgrades for free when Boot upgrades, and somebody else did the
compatibility testing. One outside it is a compatibility decision we make at every
Boot bump, forever, and a potential blocker on a security-driven upgrade we cannot
delay. Over five years that dwarfs almost any other factor, and it never appears
in a maintenance-activity check.

The second is the flat classpath. Because Maven resolves one version per artifact,
adopting a library means adopting its opinion about the version of everything it
depends on. So `mvn dependency:tree` is part of the evaluation, not part of the
implementation. If the candidate pins an older major version of something we
already use, that is a live conflict that shows up as `NoSuchMethodError` at
runtime after a clean compile — which is a diagnosis that costs a day, on an
endpoint, in production.

And one historical test that is cheap and very informative: what did this project
do when the ecosystem forced `javax` to `jakarta` on everybody? That was a
mandatory, mechanical, unavoidable migration applied to every library at once. A
project that moved promptly demonstrated it had maintainers who respond to
ecosystem pressure. A project that moved late told you something. A project that
never moved is a dead end, and that is publicly checkable in ten minutes."

**What separates them:** the senior answer evaluates the project. The principal
answer evaluates the project *in the context of the specific ecosystem's
mechanics* — release-train coupling and single-version resolution — both of which
are invisible if you carry npm intuitions across. And the `jakarta` test is a
real, cheap, historical stress test that already happened, which beats any
forward-looking guess about responsiveness.

**Adversarial follow-up:** *"We need something that isn't in the BOM and has no
good alternative. Now what?"*

The answer is not to refuse. It is to name the cost explicitly — this dependency
adds N days per Boot upgrade and is a potential upgrade blocker — assign an owner
for that recurring cost, isolate it behind an interface enforced by a build rule,
and set a review trigger for when an in-BOM alternative appears. The point of the
analysis is never to prevent adoption. It is to make sure the recurring bill has
a name on it.

---

### Q5 — "You championed a technology two years ago. It's not working out. What do you do?"

**A senior answer sounds like:** "I'd acknowledge it, gather the data on what's
not working, and propose a plan to migrate off it."

**A principal answer sounds like:** "The technical part is straightforward and it
is not the hard part.

The hard part is that I championed it, so everyone is watching how I handle
being wrong, and that is more consequential than the migration itself. If I get
defensive, the next person with bad news about their own decision will sit on it,
and I will have made the organisation worse at correcting errors — which is worth
more than any single technology choice.

So: I say it plainly and early, with the evidence, and without the softening
language. Then I separate two questions that get conflated. Is it not working, or
is it not working *yet*? Some adoptions are genuinely slow to pay off, and I need
to be honest about whether I am now over-correcting out of embarrassment, which
is a real risk and points the wrong way just as reliably.

Then I re-run the evaluation with what we now know — and we know a great deal we
did not know then, including real numbers on its recurring and operational cost.
That is a better evaluation than the original by a wide margin. Notably, the exit
cost is no longer an estimate; we can count the import sites today.

And I would write down what I got wrong in the original evaluation and why —
specifically, which assumption failed. If it was 'we'll wrap it behind an
interface' and we never did, that is not a technology lesson, it is a lesson about
crediting discounts for unfunded work, and it changes how I write the next
evaluation. That note is worth more to the next five decisions than the migration
is."

**What separates them:** the senior answer handles the decision. The principal
answer handles the decision *and* the organisational effect of how the reversal is
performed, guards against over-correction, and extracts a transferable lesson
about the evaluation method rather than about the technology. This is Topic 133's
blameless-postmortem discipline applied to your own judgment, and Topic 135 will
ask you a version of this question directly: *tell me about a technical decision
you reversed.*

**Adversarial follow-up:** *"Your team is attached to it. Half of them will read
this as you throwing their work away."*

The answer that works is not persuasion, it is participation: the people who
built it know its failure modes better than anyone and should own the evaluation
of what replaces it. And be specific about what was genuinely gained — the
knowledge, the tests, the operational understanding — rather than offering a
generic consolation. Topic 134 is this problem in its general form.

---

## Mental model checkpoint

Reason these out in writing.

1. Exit cost is the number nobody computes. Give the *structural* reason — not
   "people are lazy" — why it is systematically omitted. What would have to be
   true about how decisions are reviewed for it to be routinely included?

2. A dependency inside the Spring Boot BOM upgrades for free. Name the cost of
   that convenience. Under what circumstance is being outside the BOM actually an
   advantage?

3. You argued in Q3 that "nothing quite fits" is usually "one thing does not
   fit". Construct the strongest counterexample you can — a case where the 10%
   that does not fit genuinely justifies building the whole thing. What makes
   your counterexample different?

4. Buying converts engineer-time into money and acquires a renewal negotiation.
   Adopting converts it into dependence on a community. Building converts it into
   permanent ownership. Rank these three by *how the cost changes if the company
   grows 5×*, and justify the ranking.

5. The `javax` → `jakarta` rename was mechanically simple and ecosystem-wide.
   What made it survivable, and what would a *non*-survivable version of the same
   event look like? Is your dependency set exposed to that shape of risk today?

6. Someone shows you a five-year evaluation where the recommended option wins on
   every one of the four cost buckets. What is your first suspicion, and what
   specifically would you check?

7. You have been asked to make this decision in two days instead of two weeks.
   Which parts of the artefact do you drop, which do you keep, and what is the
   principle behind that split? Now: which decisions should *never* be made in
   two days, and how do you tell?

---

## Quick reference card

### The four buckets

| Bucket | Question | Usually |
|---|---|---|
| Acquisition | What to get it working? | Underestimated 2–5× |
| Recurring | What per year, on whose calendar? | Omitted |
| Blast radius | What when it fails × how often? | Omitted |
| **Exit** | What to remove it? | **Never computed** |

### Governance gate — before performance is discussed

- [ ] How many people merged to this in the last 12 months?
- [ ] Is there a written support / LTS policy?
- [ ] Is it version-managed by the Spring Boot BOM or a comparable curated set?
- [ ] What did it do when the ecosystem forced `javax` → `jakarta`?
- [ ] `mvn dependency:tree` — does it drag in a conflicting major version?
- [ ] What is the licence, at the version we would use, in the actual `LICENSE`
      file? Does the project have a CLA that would permit a future relicence?
- [ ] Is it reflective / proxy-based / a JVM agent? If so: how does it interact
      with our AOP (Topic 40), our observability agent, and native image (Topic 83)?

### Exit-cost checklist

- [ ] Count of files importing it — from an actual `grep`, not an impression
- [ ] Is there an interface we own, and is it enforced by a build rule?
- [ ] Semantic coupling: what behaviour does surrounding code rely on?
- [ ] Data or format lock-in: does exit require a data migration?
- [ ] Does a second implementation of the abstraction exist?
- [ ] Has anyone actually done this migration? What did it cost them?

### Reversibility classes

| Class | Example | Evaluation effort |
|---|---|---|
| Sprint-reversible | small library behind one interface | hours |
| Quarter-reversible | wide API surface, no data lock-in | days |
| Effectively irreversible | owns your data format, domain model, or topology | weeks, and get it reviewed |

### Java-specific cost drivers your Node instincts will miss

1. **Flat classpath** — you adopt the library *and its version opinions*; conflicts
   surface as `NoSuchMethodError` at runtime after a clean compile.
2. **Release-train coupling** — in the BOM is nearly free; outside it is a
   decision at every framework bump, forever.
3. **`jakarta` precedent** — abandoned upstream in Java can mean *hard blocker*,
   not merely *stale*.
4. **Reflection and agents** — the adoption surface exceeds the API surface, and
   it forecloses options like native image.
5. **Model and data coupling** — persistence and serialization write themselves
   into your domain model and your stored data.

### Verified facts you may rely on

| Fact | Value |
|---|---|
| Java 25 | LTS, GA 16 September 2025 — current LTS |
| Java 26 | GA 17 March 2026, non-LTS |
| Spring Framework 7.0 | GA 13 Nov 2025; JDK 17 baseline, JDK 25 recommended; Jakarta EE 11 |
| Spring Boot 4.0.0 / 4.1.0 | GA 20 Nov 2025 / 10 Jun 2026 |
| Spring Boot 3.5.x | **OSS support ended June 2026** — commercial support only |

Any other date, look up. Primary sources: `spring.io/projects/spring-boot#support`
and `openjdk.org/projects/jdk/` plus your JDK vendor's support page.

---

## When would I use this at work?

**1. An engineer opens a pull request adding a dependency.**
The right review question is not "is this a good library". It is: do we already
have something that does this, is it in our BOM, what does `dependency:tree` say
about the version conflicts it brings, and how many files will import it in a
year. Four questions, ninety seconds, and it prevents the most common slow leak
in a Java codebase — the second implementation of something you already have.
This is the highest-frequency application of the topic and it is nearly free once
the questions are habitual.

**2. A vendor renewal, or an out-of-support framework line.**
The renewal conversation goes well or badly depending entirely on whether you
have a credible walk-away option, and that is determined months earlier by
whether you kept the exit cheap. Being the person who says "here is what leaving
would cost us, here is what we would do instead, and here is the number I am
willing to pay" is a materially different position from "we need this, what is
the price". Same for support contracts on an EOL framework line: you buy time, and
you buy it with a plan that has a deadline attached.

**3. Someone proposes a rewrite.**
Rewrites arrive framed as binaries: keep the pain, or replace it. The most
valuable ninety minutes you can spend is turning that binary into four options —
status quo, targeted fix, partial adoption behind an interface you own, full
replacement — and pricing exit cost for each. Very often the targeted fix is two
engineer-days and gets 80% of the benefit, and nobody had checked because the
conversation started at the wrong altitude. Doing this well, once, in public,
changes how your organisation frames the next five of these.

---

## Connected topics

**Prerequisites — the evidence you will cite:**

- **Topic 32 — dependency resolution.** Maven's flat, nearest-wins classpath is
  the mechanic behind half the Java-specific cost in this topic. Read it again
  before you write the artefact.
- **Topic 34 — supply chain, SBOM, CVE triage.** Governance and licence checks
  are the gate; CVE triage is what happens when the gate was not applied.
- **Topics 48–53 — Hibernate.** The failure modes that make the persistence
  decision in Example 2 a real one rather than a preference.
- **Topic 65 — the load baseline.** Every performance claim in your artefact is
  checked against this or it is not evidence.
- **Topic 83 — native image / AOT.** A cost-and-startup lever that is itself a
  build/buy/adopt decision, and one that some dependencies quietly foreclose.
- **Topic 109 — the pool as the real ceiling.** Before accepting any performance
  argument, establish what the binding constraint actually is.
- **Topic 111 — Resilience4j**, **112 — Spring Cloud**, **118 — metrics.** Each is
  a concrete adopt-or-not decision you have already lived through.
- **Topic 124 — the readiness review.** Its top-three risks are the legitimate
  starting points for a need statement.

**This unlocks:**

- **127 — migration planning I.** Once you have decided to change something,
  sequencing it is the next problem. The upgrade cost you estimated in Table B is
  the input to that plan.
- **128 — migration planning II.** The strangler is the technique that makes a
  large adoption reversible per step, which is the property Example 2's option 3
  relies on.
- **129 — capacity and cost.** The operational lines in Table B — memory, threads,
  startup, instance count — are computed there. If your Table B has a
  "extra memory" row and you guessed the number, 129 is where you learn to derive
  it instead.
- **131 — design docs.** This artefact *is* a design doc of a particular kind.
  The alternatives-with-reasons and the reversibility statement are the sections
  131 attacks hardest.
- **132 — engineering standards.** The build rules this topic keeps demanding —
  architecture tests, dependency-convergence enforcement, banned-dependency lists
  — are introduced there without a three-month freeze.
- **134 — influence without authority.** A five-year evaluation is only worth
  writing if it changes what people do. Making adoption cheaper than
  non-adoption, rather than arguing for it, is 134's subject.
- **135 — the interview simulation.** "Tell me about a technical decision you
  reversed" is Q5 with the safety off.

---

*Java baseline 21, running on JDK 25, Spring Boot 4.1 / Framework 7.0. The cost
mechanics in this topic — flat classpath resolution, BOM-managed release trains,
the exit-cost asymmetry — are properties of the ecosystem and are stable. The
specific version and support dates are not; verify them at
`spring.io/projects/spring-boot#support` and your JDK vendor's support page
before they go in a document with your name on it.*
