# 125 — Reading the Source: OpenJDK, Spring, JEPs

## Phase: 12 — Principal Track
## Category: ELITE
## Java baseline: 21  |  Notes features from: 21/25
## Project spine: a written analysis of one unreleased or preview JEP, ending in a position on whether `orderflow` should adopt it — and a written statement of what would change that position

---

> **Phase 12 is not more Java.** Phases 1–11 made you someone who can build and
> operate `orderflow`. Phase 12 is about the decisions you make *before* anyone
> writes code, and about being trusted with them. Every topic here produces an
> **artefact**, and I review the artefact by attacking its weakest assumption.
> There are no easy/medium/hard exercises. There is a document, and there is a
> hostile reviewer.

---

## Mechanical statement

A JEP is not an announcement. It is a **design document with a fixed shape**:
a Summary, a **Goals** section, an explicit **Non-Goals** section, a Motivation,
a Description, and — this is the part almost nobody reads — an **Alternatives**
section where the authors say what else they considered and why they rejected it.

The mechanic you are learning:

> **The Non-Goals and the Alternatives tell you whether a feature fits your
> problem. The Description only tells you what the feature does.**

A blog post is written by someone who found the feature interesting. A Non-Goals
section is written by the people who will maintain the feature for twenty years,
listing the things they have *decided not to solve*. If your problem is on that
list, the feature will never solve it, no matter how many releases you wait.

The same mechanic applies one level down, to source code:

> **When documentation is ambiguous, the source is the authority — but only the
> parts of it that are specified. Everything else is an implementation detail
> that is allowed to change under you.**

Reading source to answer a question is a five-minute skill. Knowing which lines
you are *allowed to depend on* is the principal-level part.

---

## The bridge from what you know

### What transfers — completely, and I will not re-teach it

You have done all of this in TypeScript, and it transfers 1:1:

- **Reading library source to resolve a question the docs do not answer.** You
  have opened `node_modules` and read the actual implementation. Identical skill.
- **Tracking a language proposal before it ships.** TC39 has stages 0–4. You
  have almost certainly had an opinion on decorators, or on `Array.prototype.at`,
  before either shipped. Identical skill.
- **Deciding whether to adopt an unstable API.** You have made the call on a
  Node experimental flag, or on a `next`-tagged npm package. Identical decision
  shape: reversal cost versus benefit, weighted by how likely the API is to move.
- **Writing a position document and defending it.** You have done this. It
  transfers whole.

None of the above is what makes this topic hard in Java. Skip straight to the
next part.

### What does not transfer — and this is the whole topic

| You know (Node/TS) | Java/JVM reality | Verdict |
|---|---|---|
| TC39 stages 0–4, driven by champions and browser implementations | **JEP states**: Draft → Submitted → Candidate → Proposed to Target → Integrated → Delivered (or Closed/Withdrawn) | **PARTIAL** — the ladder rhymes, the governance does not. A JEP is targeted at a *specific release* long before it lands, and can be un-targeted late. |
| Node ships when it ships; you upgrade when you like | **A strict six-month train.** March and September, every year, no slips. Features that miss the train wait six months. | **NO ANALOGUE** — the cadence is the single most important scheduling fact in the ecosystem. |
| Node LTS lines, ~30 months | **JDK LTS releases** with vendor-dependent support windows that are *much* longer, and a large installed base that only ever runs LTS | **PARTIAL** — the concept exists; the timescales and the vendor fragmentation do not. |
| `--experimental-*` flags; you can ship them if you are brave | **Preview features** are compiled into class files that are *stamped* with the release that produced them and **refuse to run on any other release** | **NO ANALOGUE** — this is a hard, mechanical lock-in that has no npm equivalent. |
| You can monkey-patch or fork a dependency in an afternoon | The JDK is one artefact; you cannot patch `java.base` in production without a custom runtime image | **NO ANALOGUE** |
| Semver, and a `BREAKING CHANGE` footer | The JDK's compatibility policy is stronger than semver for *specified* API, and offers you nothing for internal API | **PARTIAL** — the guarantee is stronger where it exists and weaker where it does not. |

Two of those rows are the ones that will actually bite you: **the six-month
train** and **the preview class-file stamp**. Everything else you can reason
about from experience.

### The one habit you must break

In Node you resolve ambiguity by experiment: run it, see what happens, ship. That
works because the runtime and the library are usually the same version forever.

In Java, "run it and see" tells you what **this JDK build** does, not what the
**specification** requires. Those diverge, and the divergence is exactly where
upgrade breakage lives. A principal engineer answers "does X happen?" with
"the spec requires it" or "this implementation happens to do it, and I would not
depend on that". Those are different answers with different blast radii.

---

## What is this?

Three related things, taught together because a principal-level answer usually
needs all three.

### 1. The JEP process

A **JEP** (JDK Enhancement Proposal) is the design document for any significant
change to the JDK — a language feature, an API, a GC, a tooling change. They live
at `openjdk.org/jeps/<number>`, and the index of all of them is at
`openjdk.org/jeps/0`.

The states a JEP moves through:

| State | What it means for you |
|---|---|
| **Draft** | Someone is thinking out loud. Read it for direction, not for planning. |
| **Submitted** | Author believes it is ready for review. Still not a plan. |
| **Candidate** | Accepted as something the project *wants*. Still no release attached. |
| **Proposed to Target** | A specific release has been proposed. This is the first point where a date is meaningful — and it can still be pulled. |
| **Targeted** | It is going into release N unless something goes wrong. |
| **Integrated** | The code is in the mainline repository. |
| **Completed / Delivered** | It shipped in a release. |
| **Closed / Withdrawn** | It is not happening in that form. |

The state that matters most for planning is **Proposed to Target / Targeted**,
because before that there is no date at all, and people who quote dates before
that are quoting a hope.

### 2. Preview, incubator, experimental — three different things

These three words get used interchangeably in conversation and they are not the
same mechanism. Getting them wrong in a design review is a tell.

| Kind | What it covers | How you turn it on | The catch |
|---|---|---|---|
| **Preview feature** | A **language or VM feature** that is complete and specified but not permanent | `--enable-preview` at *both* `javac` and `java`, plus `--release N` | The produced class file is marked as a preview class file for release N. It will **refuse to load on release N+1**, even though N+1 is newer. Recompilation is mandatory on every JDK upgrade. |
| **Incubator module** | A **library API** that is not yet final, shipped in a `jdk.incubator.*` module | `--add-modules jdk.incubator.<name>` | The package name itself changes when it graduates. Every import in your codebase changes. |
| **Experimental VM option** | A HotSpot flag not considered production-ready | `-XX:+UnlockExperimentalVMOptions` before the flag | No compatibility promise at all; can vanish in a patch release. |

**Define the terms once, precisely:**

- **Preview** = the design is finished; the JDK team wants real-world feedback
  before making it permanent. It may still change between previews, and it has
  done so repeatedly.
- **Incubator** = the design is *not* finished. The API is expected to change.
- **Experimental** = the implementation is not trusted. Usually a VM flag.

The preview class-file stamp is the important operational fact. Say it plainly to
your team: **if any module in the build uses a preview feature, every JDK upgrade
becomes a mandatory recompile-and-retest of that module, on a schedule set by
someone outside your company.** That is a real, recurring cost. Sometimes it is
worth it. It is never free.

### 3. Reading source as an answer-finding technique

Two source trees matter to you.

**OpenJDK** — `github.com/openjdk/jdk`. The layout you need:

```
src/java.base/share/classes/java/util/concurrent/   the concurrency library
src/java.base/share/classes/java/lang/              String, Integer, Thread, ...
src/hotspot/share/                                  the VM itself, in C++
src/hotspot/share/gc/g1/                            G1, from Topic 71
test/jdk/java/util/concurrent/                      the JDK's own tests
test/hotspot/jtreg/                                 VM tests
```

**Spring** — `github.com/spring-projects/spring-framework` and
`github.com/spring-projects/spring-boot`. The classes you will actually open,
because they answer the questions people actually ask:

```
spring-tx/.../transaction/interceptor/TransactionAspectSupport.java
        -> what @Transactional actually does around your method (Topic 54)
spring-tx/.../transaction/interceptor/RuleBasedTransactionAttribute.java
        -> which exceptions roll back and which do not
spring-beans/.../factory/support/AbstractAutowireCapableBeanFactory.java
        -> the bean lifecycle from Topic 37, as executable code
spring-context/.../annotation/ConfigurationClassPostProcessor.java
        -> how @Configuration classes are parsed and why @Bean methods are proxied
spring-boot-autoconfigure/  (Boot 3.x) / the split modules (Boot 4.x)
        -> the @Conditional evaluations from Topic 42
```

And the underrated one: **Spring's tests are better documentation than Spring's
documentation.** For nearly every behavioural question, there is a test class next
to the implementation whose name is the question. `TransactionAspectSupport` has
tests that enumerate propagation and rollback combinations. Reading those is
faster than reading the reference manual and strictly more reliable.

### The distinction that separates senior from principal

Not every line you can read is a line you can depend on.

| Thing you read | Can you depend on it? |
|---|---|
| The javadoc's `@implSpec` / normative prose on a public JDK API | **Yes.** This is the specification. |
| The JLS / JVMS | **Yes.** This is the specification. |
| The body of a public JDK method that goes beyond the javadoc | **No.** Implementation detail. |
| Anything in `sun.*`, `jdk.internal.*`, `com.sun.*` | **No**, and since JDK 16 strong encapsulation makes most of it inaccessible anyway. |
| A public Spring class documented in the reference manual | **Yes**, within its stated semver. |
| A Spring class marked `@Internal`, in an `.support` package, or undocumented | **Treat as no.** Spring is explicit that these can move. |
| Behaviour you observed by running it | **No.** That is one data point on one build. |

Write this table into your own head. It is the difference between "I read the
source" and "I read the source and I know which half of it is a contract".

---

## Why does it matter?

Three concrete reasons, in increasing order of career impact.

**1. Speed of resolution.** A question like "does `@Transactional` roll back on a
checked exception thrown from a `@Async` method?" takes forty minutes of blog
archaeology and produces an answer you do not trust. It takes eight minutes of
source reading and produces an answer you can put your name on, plus a permalink
to a specific line for the design doc. That ratio, applied hundreds of times a
year, is a visible difference in how much you get done.

**2. Position before the herd.** When a feature previews, there is a window of
roughly two to four releases where nobody has written the definitive blog post
yet. In that window, the person who has read the JEP — including the Non-Goals —
is the most informed person in every room. That is how technical influence
actually accumulates (Topic 134). Not by being loud. By being early and correct.

**3. Knowing what you signed up for.** Adoption decisions in Java have long
tails, because the JDK's cadence is not yours to control. A preview feature
commits you to a recompile on every JDK bump. An incubator module commits you to
a package rename. A JEP that is Candidate but not Targeted commits you to
nothing but has no date. A principal engineer states the commitment out loud
*before* the team takes it on, because the team will be living with it for years
and the person who proposed it may not be.

---

## The decision, framed

Here is the decision this topic teaches, stated as a decision and not as a topic.

> **A feature exists that is not yet final. Should `orderflow` adopt it, and what
> would make you change your answer?**

That question has five inputs. Work them in this order — the order matters,
because the first two can end the analysis before you do any work.

### Input 1 — Do we have the problem this JEP solves?

Read **Motivation** and **Goals**. Then read **Non-Goals** twice. If your problem
is a Non-Goal, stop. You have your answer and it took ten minutes. This is the
single highest-leverage step and almost everybody skips it.

State your problem in one sentence *first*, before you read, so you cannot
retro-fit it to the feature. This is the same discipline as writing the test
before the implementation, and it fails the same way if you cheat.

### Input 2 — What is the honest alternative today?

The JEP's own **Alternatives** section is written from the JDK's perspective:
"why not solve this a different way in the platform". Yours is different: "what
do we do today, and how bad is it really?"

If the answer is "we write fifteen more lines of `CompletableFuture` composition
and it works", the feature is a convenience, and conveniences do not justify
non-final dependencies.

### Input 3 — What is the reversal cost?

Not the adoption cost. The **reversal** cost. If in eighteen months the API
changes shape or the JEP is withdrawn, what do we have to do?

Ask it structurally:

- How many call sites?
- Are they behind an interface we own, or scattered?
- Is the feature *load-bearing for correctness*, or is it a nicer way to spell
  something we could spell otherwise?

A feature used in three places behind one internal interface is a cheap bet. The
same feature used in two hundred places across five modules is a quarter of work
to undo.

### Input 4 — What is the recurring cost, on whose schedule?

This is the Java-specific one.

- **Preview feature**: mandatory recompile on every JDK upgrade, plus the risk of
  an API change between previews. The schedule is OpenJDK's, not yours.
- **Incubator module**: a package rename when it graduates, plus the same
  recompile risk.
- **Targeted-but-unreleased**: you cannot use it at all yet; the question is only
  whether to *design toward* it.

Say the recurring cost in units of engineer-days per year. If you cannot, you do
not yet understand the commitment.

### Input 5 — What would change my mind?

Write it down as a **falsifiable trigger**, not a feeling. Examples of the right
shape:

- "If the API is unchanged across two consecutive previews, the change risk has
  dropped enough to adopt in the payment fan-in."
- "If it is Targeted for an LTS release, we adopt on that LTS."
- "If our p99 on the fan-in path exceeds our budget from Topic 129 and profiling
  shows the cost is in our hand-rolled orchestration rather than downstream, the
  benefit side has grown enough to reconsider."

Examples of the wrong shape: "if it seems more stable", "if the community
adopts it", "if we have more time". Those are not triggers; they are ways of
never deciding.

**The trigger is the part of the artefact I attack hardest.** A document without
one is a document that will be re-argued every quarter forever.

---

## Example 1 — a minimal illustration

The smallest possible version of the skill: **one ambiguous question, resolved
from source, turned into one sentence you can defend.**

### The question

A colleague writes this in the `orderflow` product-catalogue warm-up (Topic 37):

```java
private final ConcurrentHashMap<String, Product> bySku = new ConcurrentHashMap<>();

Product load(String sku) {
    return bySku.computeIfAbsent(sku, this::fetchFromDatabase);
}
```

In review, someone asks: "is that safe if two threads ask for the same SKU at
once? And what if `fetchFromDatabase` ends up touching the same map?"

Three unsatisfying ways this normally goes:

1. "`ConcurrentHashMap` is thread-safe, so it's fine." — true and irrelevant.
2. "I ran it with two threads and it was fine." — one data point, one build.
3. Someone links a 2014 blog post about `ConcurrentHashMap` in Java 7, which had
   a completely different internal design (segments, not bins with per-bin
   locking). Wrong version, confidently cited.

### The five-minute resolution

Open the **javadoc for the method you are actually calling** — not the class.
`ConcurrentHashMap.computeIfAbsent(K, Function)`. Read its normative prose. Then
open the source:

```
src/java.base/share/classes/java/util/concurrent/ConcurrentHashMap.java
```

and find `computeIfAbsent`. You are looking for two things, and only two:

1. **What does the javadoc promise about atomicity?** The method's own
   documentation states that the entire method invocation is performed
   atomically, and that some attempted update operations on the map by other
   threads may be blocked while computation is in progress. That is normative —
   it is the contract, and you may depend on it.
2. **What does the source do when the mapping function re-enters the map?** Find
   the check that throws on a recursive update. The javadoc also warns that the
   mapping function must not attempt to update any other mappings of this map.

The distinction you are practising: point 1 is a **contract**. The exact bin
locking that implements it is an **implementation detail** that changed between
Java 7 and Java 8 and could change again.

### What you write down

Not "I checked, it's fine." This:

> `computeIfAbsent` is atomic per key by contract, so two threads asking for the
> same SKU will produce exactly one `fetchFromDatabase` call — this is documented
> behaviour, not an implementation accident. The cost is that the mapping
> function runs while holding the bin, so a slow database call blocks other
> threads that hash to the same bin. Our warm-up runs at startup with no
> concurrent traffic, so that cost is irrelevant here — but it would be a real
> problem if we used the same pattern in the request path, and we should not.
> Also: `fetchFromDatabase` must not touch `bySku`, or we get an
> `IllegalStateException` for recursive update. Verified against
> `ConcurrentHashMap` on JDK 25.

That paragraph is worth more than the fix. It states what is contractual, what is
incidental, where the pattern would stop being safe, and what version it was
checked against. A reviewer can attack it — and if they do, they have to attack
it with a source reference, which is exactly the conversation you want.

> **Do this yourself before continuing.** Open the file, find the method, and
> confirm the two facts above on the JDK you actually run. The point of the
> exercise is not the answer; it is that you now know how long it takes. It is
> shorter than you think.

---

## Example 2 — the real decision on the project spine

Now the artefact-shaped version, on `orderflow`, with real constraints.

### The setting

`orderflow` has, by Topic 108, a **payment-callback fan-in**: when an order is
placed, the service calls the payment gateway, the fraud-scoring service, and the
wallet-balance service, and needs all three before it can respond. Today that is
`CompletableFuture` composition on a bounded executor (Topics 91, 92), with an
overall timeout and per-call circuit breakers (Topic 111).

The code works. It is also the part of the codebase where every incident review
has found the same shape of problem:

- a failed sub-call that leaves the other two running to completion, burning
  downstream capacity for a response nobody will read;
- a timeout on the outer future that does not actually cancel the inner ones;
- a stack trace on failure that shows the executor thread, not the request path,
  so you cannot tell which order it was (Topic 119's context propagation problem
  in a different costume).

### The candidate

**Structured concurrency** — the `java.util.concurrent` API built around
`StructuredTaskScope`, in which concurrent subtasks are confined to a lexical
scope, the scope does not exit until all subtasks are done, and a failure in one
subtask cancels its siblings.

> **Flag, per this document's own rules:** structured concurrency has been
> previewed across several JDK releases and its API changed shape between
> previews — notably the move away from constructing a scope directly toward a
> static factory with a joiner. I am working from release notes I read, and this
> is precisely the kind of fact you must verify rather than inherit. **Go to
> `openjdk.org/jeps/0`, search for "structured concurrency", and read the JEP
> that is current for your JDK.** Get the number and the state from there, not
> from me. The fact that its API moved between previews is not a footnote in this
> analysis — it is the single most important input to it.

### Working the five inputs

**Input 1 — do we have the problem?**

Our problem, stated before reading: *"a sub-call that fails or times out does not
reliably cancel its siblings, and the failure loses request context."*

Read the JEP's Goals. Cancellation propagation and treating a group of subtasks
as a unit are squarely in scope. Read the **Non-Goals**. This is where the
analysis gets sharp. Look specifically for whether the JEP declines to:

- replace or subsume `ExecutorService` for long-lived pools;
- provide a distributed or cross-process cancellation story;
- change the behaviour of blocking I/O in any way;
- solve observability or context propagation as such (that is a *different*
  feature — scoped values — with its own JEP).

If context propagation is a Non-Goal of the structured concurrency JEP, then
**one of our three problems is not solved by this feature at all**, and I need to
say so in the artefact rather than let the good parts carry the bad. That single
sentence is the difference between an analysis and an advertisement.

**Input 2 — what is the honest alternative today?**

We can do sibling cancellation by hand: keep references to the three futures,
and on the first failure call `cancel(true)` on the others. It is roughly twenty
lines, it is easy to get subtly wrong, and — importantly — `Future.cancel` on a
task already blocked in a socket read does not reliably stop it, which is a
limitation the JDK feature does not remove either.

So the honest statement is: **the alternative is not "nothing", it is "twenty
lines of error-prone code that has the same underlying limitation on
uninterruptible blocking".** That materially weakens the case for adoption, and
it belongs in the document. If your analysis makes the alternative look worse
than it is, the reviewer will find it and everything else you wrote loses value.

**Input 3 — reversal cost.**

Count the call sites. In `orderflow` there are two places that fan out: the
payment fan-in and the catalogue enrichment path. Both are behind service
interfaces we own.

That is the good case: **two call sites, both behind our own interfaces.** If the
API changes shape, we change two files. Write the number down — "two call sites,
one internal interface, estimated one engineer-day to revert" — because a number
survives a hostile review and an adjective does not.

Now be honest about the bad case: if we adopt it and it *works*, engineers will
use it in new code, because it is nicer. In eighteen months it will not be two
call sites. So the reversal cost is not static, and the artefact must say how we
bound it — for example, a lint rule or an architecture test (Topic 132's
territory) that restricts the API to a named package until it is final.

**Input 4 — recurring cost, on whose schedule.**

Preview features force `--enable-preview` in both compilation and runtime, and
the produced class files will not run on the next JDK release. `orderflow` runs
on JDK 25, an LTS. If we take a preview dependency:

- every JDK upgrade — including the six-monthly non-LTS releases if we ever
  evaluate one — requires recompiling and retesting that module;
- our container image and our build must agree on the exact release, permanently;
- Topic 83's native-image and AOT-cache options need re-verification against a
  preview feature, because they are the most sensitive to anything non-standard.

Express it as a number: "approximately N engineer-days per JDK upgrade, at an
upgrade cadence we do not control." If you do not know N, say you do not know N
and say what you would do to find out — for example, do the adoption on a branch,
recompile against the next early-access build, and measure. Naming the experiment
is better than guessing the number.

**Input 5 — what would change my mind.**

Write two triggers, both falsifiable, one in each direction:

- **Adopt when:** the API is unchanged across two consecutive JDK releases, *or*
  it reaches final in an LTS release. Whichever comes first.
- **Reconsider earlier when:** we have a second production incident whose
  contributing factor is uncancelled sibling calls after a fan-in failure. Two
  incidents is evidence that the twenty-lines-by-hand alternative is not holding.
- **Abandon when:** the JEP goes back to Draft, or a subsequent preview changes
  the API in a way that breaks the two call sites.

### The position

A defensible position for `orderflow`, written the way it would go in the doc:

> **Recommendation: do not adopt in production now; prototype on a branch and set
> a review trigger.**
>
> The feature addresses one of our three fan-in problems well (sibling
> cancellation), one partially (failure clarity), and one not at all (request
> context propagation, which is a Non-Goal and belongs to a separate feature).
> Adoption today buys us roughly twenty lines of hand-written cancellation logic
> in two call sites, in exchange for a preview dependency that makes every JDK
> upgrade a mandatory recompile of that module on OpenJDK's schedule rather than
> ours.
>
> That trade is bad today because our two call sites are small and stable. It
> becomes good when the recurring cost goes to zero — which happens on final
> release, not before.
>
> **Action now:** one engineer, three days, prototype the payment fan-in on a
> branch against the current preview. The deliverable is not a merge; it is a
> written note saying whether the cancellation semantics actually fix the
> uncancelled-sibling incident we had, and what the diff size was. If the answer
> is no, we have saved ourselves the whole conversation permanently.
>
> **Review trigger:** unchanged API across two consecutive releases, or final in
> an LTS. Re-open early on a second uncancelled-sibling incident.

Notice what that position is *not*. It is not "let's wait and see". "Wait and
see" has no trigger, produces no information, and will be re-argued next quarter
by someone who read a conference talk. The recommendation above spends three
engineer-days to buy a fact, and names the condition under which the answer
changes.

---

## Wrong approach → exact symptom → root cause → fix

The symptoms in Phase 12 are **organisational**. There is no stack trace. What
you are learning to recognise is a shape in how a team behaves.

---

### Wrong approach 1 — adopting a preview feature because the talk was good

**Wrong:** an engineer returns from a conference, uses a preview language feature
in the `orderflow` payment module, and the PR is approved because the code is
genuinely nicer to read.

**Exact symptom — what you would have SEEN:**

- The build file grows `--enable-preview` in `maven-compiler-plugin`, and the
  container entrypoint grows `--enable-preview` to match. Two places, in two
  repositories.
- Eight months later, the platform team's JDK upgrade PR is red. The failure at
  runtime is a class-file version rejection: the class was compiled as a preview
  class file for the previous release and the new JVM refuses to load it.
- The JDK upgrade — which was supposed to be a one-line base-image change for
  eleven services — becomes "blocked on payments" in the migration tracker, and
  stays there for six weeks.
- Sixteen months later, a PR titled "adapt to the new API shape" touches forty
  files, because the preview changed between previews.

**Root cause:** nobody asked "what is the recurring cost and whose schedule is it
on?" The adoption decision was made on code aesthetics — a real benefit — with
the cost side of the ledger left blank. The reviewer who approved it was
evaluating the diff, which is a senior-level review. Nobody evaluated the
*commitment*, which is the principal-level review.

**Fix:**

1. A written rule: **preview and incubator features are allowed in prototypes and
   in one named sandbox module, and are forbidden in modules on the request
   path**, until final. Enforce it with an architecture test or a build check
   (Topic 132), not with a code-review convention, because conventions decay.
2. Every preview adoption carries an owner and a review trigger in the same PR
   description.
3. The JDK upgrade cost goes on the team's roadmap as a recurring item, visible
   to the same manager who approved the feature work.

---

### Wrong approach 2 — answering a behavioural question from a blog post

**Wrong:** "Does Spring retry the transaction if the commit fails?" Someone
searches, finds a well-written post, and the answer goes into a design document
and then into an on-call runbook.

**Exact symptom — what you would have SEEN:**

- A design doc containing "Spring will retry" with no link, or with a link to a
  post whose byline year is six major versions ago.
- A review comment that says "source?" and gets no reply, and the doc is approved
  anyway because the meeting was ending.
- Nine months later, a postmortem (Topic 133) whose timeline includes: *"the
  on-call engineer waited for the automatic retry described in the runbook. There
  is no automatic retry."* Twenty-two minutes of unnecessary outage, attributed
  in the postmortem to a belief nobody could trace to a source.

**Root cause:** a blog post is a snapshot of one person's understanding of one
version. Spring's behaviour here lives in `TransactionAspectSupport` and the
`PlatformTransactionManager` implementation, and it has version-specific detail.
The team treated a secondary source as authoritative because it was easier to
read than the primary one.

**Fix:** a norm you can actually enforce in review: **any behavioural claim about
a framework in a design doc or a runbook carries either a link to the
specification, or a permalink to a specific line of source at a specific tag.**
Not a link to the repository — a permalink to the line. It takes twenty seconds
to produce and it makes the claim falsifiable forever after.

---

### Wrong approach 3 — depending on what the source does rather than what it promises

**Wrong:** an engineer reads the source, finds that a Spring internal class
happens to expose a `Map` of bean definitions, and builds an `orderflow`
diagnostic endpoint on top of it. It works beautifully and it is genuinely
useful.

**Exact symptom — what you would have SEEN:**

- The minor-version Spring upgrade fails to compile, with a missing method or a
  changed signature on a class nobody outside the framework was supposed to call.
- An issue filed upstream, closed politely as "this is internal API".
- The `orderflow` upgrade is now blocked on rewriting a feature that was never
  asked for, and the engineer who wrote it has moved teams. The endpoint gets
  deleted eight weeks later, having produced a quarter of upgrade friction for a
  diagnostic nobody had used in five months.

**Root cause:** confusing *"I can call it"* with *"I am allowed to call it"*. The
compiler will happily let you call any public method. Only the documentation,
the package name, and the project's stated compatibility policy tell you whether
that method is a contract.

**Fix:**

1. Before depending on anything you found by reading source, answer in writing:
   is this in the reference documentation, is the package part of the public API
   surface, is it annotated as internal?
2. If the answer is "no" and you still need it, **isolate it behind one interface
   you own**, with a comment naming the exact version it was verified against and
   what breaks if it changes. That converts an unbounded upgrade risk into one
   file.
3. Better: file the upstream feature request. Sometimes you get a supported API,
   which is a much better outcome than a clever workaround.

---

### Wrong approach 4 — an analysis with no falsification condition

**Wrong:** a thorough, well-written six-page evaluation of a JEP that concludes
"we should keep monitoring this".

**Exact symptom — what you would have SEEN:**

- The same feature name on the architecture-forum agenda in Q1, Q3, and the
  following Q1.
- Three different documents in the wiki about it, by three different authors,
  reaching compatible conclusions, none of which changed any behaviour.
- Engineers privately deciding for themselves, so the codebase acquires it in one
  team's module anyway, without a decision.

**Root cause:** "keep monitoring" is not a decision. It has no trigger, no owner
and no expiry, so it cannot be completed, only repeated. The document produced
understanding, which felt like progress, but it produced no *state change*.

**Fix:** every adoption analysis ends with exactly one of four verbs, and each
carries an owner and a date:

- **Adopt** — with the scope and the rollback plan.
- **Prototype** — with the specific question the prototype answers, a time box,
  and what you will do with each possible answer.
- **Defer with a trigger** — the trigger is a *fact about the world* (release
  state, a repeated incident, a measured budget breach), not a feeling.
- **Reject** — with the condition that would reopen it.

If none of those four fits, you have not finished the analysis.

---

### Wrong approach 5 — tracking everything and adopting nothing

**Wrong:** a team with a genuinely excellent JEP-watching habit, a shared
document, and a monthly discussion. Also, an estate on an old JDK and an
out-of-support framework line.

**Exact symptom — what you would have SEEN:**

- A well-maintained "upcoming Java features" page, last updated last week.
- A dependency dashboard showing the runtime on a JDK two LTS releases behind and
  a Spring Boot line that left OSS support months ago — meaning **no free CVE
  patches** (Spring Boot 3.5 left OSS support in June 2026; that is a verified
  date, and it is exactly the kind of date that turns an abstract policy into a
  security finding).
- A security audit finding that is answered with "we are evaluating our upgrade
  path", in the same words as the previous audit.

**Root cause:** curiosity substituting for judgment. Reading about the future is
more pleasant than migrating the present, and it produces the same feeling of
technical engagement. The team optimised for the pleasant activity.

**Fix:** tie the two together explicitly. The *same* document that tracks upcoming
features carries the current support status of everything you run, with the end-
of-support date in a column. Feature evaluation is a discretionary activity;
staying in support is not. If the support column has a red cell, feature
evaluation is not what this month's meeting is about. Topic 127 is the plan for
getting the red cells green.

---

## Artefact — what you must produce

### Specification

**Title:** `<Feature name> — adoption analysis for orderflow`

**Length:** 1,000–1,600 words. Not longer. A principal-level analysis that cannot
fit in 1,600 words is usually an analysis that has not decided anything. If you
need more room, the extra material goes in an appendix that the reader may skip
without losing the decision.

**Subject:** exactly one JEP that is **not yet final** — Draft, Candidate,
Targeted, or in Preview/Incubator. Not a finalised feature. The whole point is
reasoning under uncertainty about something whose shape can still change.

Suggested candidates, all genuinely relevant to `orderflow` — verify the current
number and state yourself at `openjdk.org/jeps/0`:

- **Structured concurrency** — directly applicable to the payment fan-in
  (Topics 92, 108).
- **Scoped values** — directly applicable to the request-context propagation
  problem you met in Topics 91, 101 and 119.
- Any **Project Leyden** JEP on AOT caching and startup — directly applicable to
  Topics 83 and 122, and to the cost model you build in Topic 129.
- Any **Project Valhalla** JEP on value types — applicable to Topics 01 and 69,
  and the longest-horizon bet on this list.

**Required sections, in this order:**

**1. The problem, in one sentence, written before you read the JEP.**
State it as a thing that has cost you something in `orderflow`. Cite the topic
where it bit you, and the observable evidence — a drill, a metric, an incident.
If you cannot name the evidence, choose a different feature.

**2. What the JEP says it does.** Three to five sentences. Summary and Goals,
in your words. If you cannot summarise it in five sentences you have not read it
carefully enough to have an opinion.

**3. Non-Goals — and which of yours are on that list.** Quote the Non-Goals
verbatim, then map each of your problems onto "solved / partly solved / explicitly
not solved". **If nothing lands in the third column, I will assume you did not
read the section, because almost every JEP declines to solve something adjacent
that people expect it to solve.**

**4. Alternatives — theirs and yours.** Summarise the JEP's Alternatives section.
Then, separately, describe what `orderflow` does today and how bad it actually
is. Be fair to the status quo. Give it its best argument.

**5. Reversal cost.** A number, not an adjective. Call sites, modules, whether
they are behind an interface you own, and an engineer-day estimate to undo.
Include how the number grows if adoption is *successful* and spreads.

**6. Recurring cost and whose schedule it is on.** Preview stamp, incubator
package rename, or none. Express as engineer-days per JDK upgrade, and state the
upgrade cadence. If you do not know the number, name the experiment that would
tell you.

**7. Interaction with what we already run.** At minimum: does this interact with
the container limits from Topic 82, the native-image/AOT choices from Topic 83,
the observability from Topics 118 and 119, or the readiness posture from Topic
124? Most features touch at least one. A feature that touches none is probably
not important enough to write about.

**8. Position.** One of: Adopt / Prototype / Defer with trigger / Reject. With
scope, owner and date.

**9. What would change my mind.** Two triggers minimum, both falsifiable, at
least one in each direction. A trigger is a fact about the world with a date or a
threshold attached.

**10. Sources.** Permalinks. The JEP URL, the relevant `openjdk.org` mail-list
thread if you found one, and any source-code permalink at a specific tag. No
blog posts as load-bearing citations. Blogs are allowed as "here is where I first
heard about it", clearly labelled as such.

### Constraints

- **No benchmark numbers you did not run.** If you quote a performance claim,
  it is either from the JEP itself — attributed as the authors' claim, not as
  fact — or from a measurement you took on your own hardware, with the method
  stated. Anything else is fabrication, and I will find it.
- **No appeal to popularity.** GitHub stars, conference-talk count and "everyone
  is moving to it" are not inputs. If they appear, I will strike them and ask
  what is left.
- **Name the version you read.** JEPs get revised. Say which revision, and when
  you read it.

---

## How I will review it

I am reading as a staff-level reviewer. I am not reading for prose. Here are the
three questions that break a document of this kind, in the order I ask them.

### Question 1 — "Which of your problems does the Non-Goals section refuse to solve, and what is your plan for that one?"

This is the single highest-yield attack on an adoption analysis, and it lands
most of the time.

What I am testing: whether you read the JEP or read *about* the JEP. Secondary
coverage is written to be interesting, and Non-Goals are not interesting. They
are, however, the section where the authors tell you the boundaries of the
feature in their own words.

**How the document fails:** section 3 says "no significant non-goals for our use
case". Almost never true. When I then read the Non-Goals myself and find one that
maps directly onto problem two in your section 1, the whole document loses
credibility — not because you were wrong, but because you were confident about
something you had not checked.

**What a strong answer looks like:** you already wrote it in section 3, you name
the unsolved problem explicitly, and section 8 either scopes the adoption to the
part that *is* solved, or names the separate feature that addresses the rest.

### Question 2 — "What is the reversal cost if this is adopted and then succeeds?"

Note the word *succeeds*. Everyone estimates the reversal cost of a failed
adoption, where usage stays small. Nobody estimates the reversal cost of a
successful one, where engineers reach for it because it is genuinely better and
it spreads into forty files across five modules in a year.

What I am testing: whether you modelled adoption as a static decision or as a
process with a feedback loop.

**How the document fails:** section 5 says "two call sites, one day to revert",
full stop. I ask what happens in eighteen months if it is nice to use, and there
is no answer — because there is no containment mechanism.

**What a strong answer looks like:** the number, *plus* a stated containment
mechanism with teeth. An architecture test that fails the build if the API is
imported outside a named package. A lint rule. A module boundary. Something the
build enforces, so the containment survives the departure of everyone who
remembers why it exists. Topic 132 is the machinery; this is the first place you
need it.

### Question 3 — "What specifically would make you change this recommendation, and how would you find out that it happened?"

Two halves, and people usually have only the first.

What I am testing: whether the trigger is falsifiable, and whether anyone will
ever notice it firing.

**How the document fails:** the trigger exists — "we will adopt when it is
final" — but there is no mechanism. Nobody is subscribed to anything, nothing is
on a calendar, no owner is named. The trigger will fire and nobody will be
looking. Eighteen months later the feature is final, `orderflow` still has the
hand-rolled version, and the document is in a wiki nobody opens.

**What a strong answer looks like:** the trigger, the owner, and the **detection
mechanism**: a calendar entry on the JDK release date, a subscription to the
relevant OpenJDK mailing list, a recurring item in the team's quarterly technical
review, or a line in the same dependency dashboard that already tracks support
dates. It does not have to be sophisticated. It has to exist.

### Smaller things I will also pick at

- **A number with no method.** Any figure without "here is how I got this" gets
  struck and I ask what the argument is without it.
- **The status quo strawmanned.** If your "what we do today" section makes the
  current approach sound worse than it is, I stop trusting the rest. Write the
  status quo's best case and beat it honestly, or do not beat it.
- **A hidden second decision.** Many adoption analyses smuggle in an unrelated
  change — "and while we are at it we should also restructure the payment
  module". Two decisions in one document means neither gets reviewed properly. I
  will ask you to split it.
- **Version drift.** If you read the JEP six months ago and did not re-check its
  state before writing, the document is describing a world that has moved.

### What I will not attack

Prose quality, structure preferences, length within the stated range, whether I
personally like the feature. I am attacking the assumption load-bearing for your
recommendation, and nothing else. If I have nothing to attack, I will say so —
and that is the outcome you are working toward.

---

## Interview questions (Senior → Principal)

> The rubric line from the master plan for this topic:
> *"the docs say..." → "the docs are ambiguous here, so I read
> `TransactionAspectSupport`; the behaviour comes from **this** branch, and it
> changed in this commit for this reason."*

---

### Q1 — "How do you determine whether `@Transactional` will roll back for a particular exception?"

**A senior answer sounds like:** "By default Spring rolls back on unchecked
exceptions — `RuntimeException` and `Error` — and commits on checked exceptions.
You override it with `rollbackFor`. It is a common bug: a checked exception
escapes a `@Transactional` method and the transaction commits anyway."

That is correct and complete for day-to-day work. It is a good senior answer.

**A principal answer sounds like:** "Same default, and then I would say how I
know. The rollback decision lives in `TransactionAspectSupport`'s completion
handling, which asks the transaction attribute whether to roll back.
`DefaultTransactionAttribute` implements the unchecked-only default;
`RuleBasedTransactionAttribute` is what `rollbackFor`/`noRollbackFor` populate,
and when several rules could match it picks the most specific one by depth in the
exception hierarchy. I would open the class rather than rely on my memory of the
ordering, because that ordering is exactly the part people get wrong and it is
twenty seconds to check.

Then the bit that matters more than the mechanics: this behaviour is why our
error-handling convention says domain failures are unchecked. Relying on
`rollbackFor` annotations means the correctness of a transaction depends on
someone remembering an annotation at every call site, and that is not a property
that survives three years of team turnover."

**What separates them:** the senior answer states the rule. The principal answer
states the rule, names where the rule is implemented and how they would verify it
right now, and then converts the mechanic into a **standard** that removes the
class of bug rather than the instance. That last move is the Phase 12 move, and
it is the same one you make in Topic 133 when you write contributing factors
instead of a root cause.

**Adversarial follow-up:** *"You said you'd read the class. What in that class is
a contract you can depend on, and what is an implementation detail that could
change in a minor version?"*

They are testing the table from earlier in this document. The honest answer names
the reference documentation and the public annotation semantics as the contract,
and treats the internal ordering logic as implementation — verifiable, useful for
debugging, not something to build an architecture on.

---

### Q2 — "There's a preview feature that would make a chunk of our code much simpler. Do we use it?"

**A senior answer sounds like:** "I'd try it on a branch and see how it looks. If
the code is clearly better and the tests pass, I'd propose it. We can always
revert it if it changes."

The instinct is right and the prototype is right. What is missing is the cost
model.

**A principal answer sounds like:** "Three questions before the branch.

First, what problem is it solving for us, and is that problem on the JEP's
Non-Goals list? I have been surprised by that more than once.

Second, what does the preview stamp cost us? Preview class files will not load on
the next JDK release, so we are committing to recompile and retest that module on
every JDK upgrade, on OpenJDK's six-month cadence rather than ours. For us that
is one module, maybe two engineer-days per upgrade, and it means the module can
never be the thing blocking a security-driven JDK bump.

Third, what does success cost? If it is genuinely nicer, people will use it, and
in a year 'we can always revert it' means forty files instead of two. So if we
adopt, we adopt into one named package with an architecture test that fails the
build on imports elsewhere, and we lift that when it goes final.

Given all that, my default is: prototype now, adopt on final release, unless the
prototype shows it fixes a specific production failure we keep having — in which
case the calculation changes and I would want to see the incident record."

**What separates them:** the senior answer treats reversibility as a property of
the code. The principal answer treats reversibility as a property that **decays
over time** unless something structural holds it in place, and prices the
recurring cost in someone else's calendar. Also: the principal answer has a
default and a named condition that overrides it, so the decision is made rather
than deferred.

**Adversarial follow-up:** *"Your architecture test is a rule engineers will
route around when they are under deadline pressure. What then?"*

The good answer does not defend the rule harder. It accepts the premise and moves
to consequence: the test failing is a *conversation trigger*, not a wall; if
someone needs to route around it, that is a signal the containment assumption was
wrong and the analysis should be re-run. A rule that is quietly deleted is a
failure of the rule; a rule that gets loudly argued about is doing its job.

---

### Q3 — "How do you decide when to move to a new JDK version?"

**A senior answer sounds like:** "We stay on LTS releases and upgrade when there
is a reason — a security patch, a feature we want, or the current one going out
of support. We test in staging first."

**A principal answer sounds like:** "I separate three decisions that people
collapse into one.

The **runtime** decision: what JDK do we run in production. That one is driven by
support windows and CVE patching, not by features. The JDK ships every six months
in March and September; LTS releases are the ones with long vendor support, and
Java 25 has been the current LTS since September 2025. Java 26 arrived in March
2026 and is non-LTS. So the runtime question for us is 'are we on a supported
LTS', and the answer must never be no.

The **language level** decision: what `--release` do we compile against. That is
separate, it is reversible per module, and it can lag the runtime. Running on a
newer JDK while compiling to an older language level is a completely normal and
very useful state — it is exactly the sequencing I would use in a large migration,
and it is why Topic 127's plan upgrades the JDK first while staying on the old
framework.

The **feature adoption** decision: which new language features we actually use.
That is a standards question with a rollout, not a version question.

Collapsing those three is why upgrades feel scary. Separated, the runtime upgrade
is usually a base-image change plus a test run, and the risky parts are isolated
to where they belong."

**What separates them:** the senior answer treats "upgrading Java" as one action.
The principal answer decomposes it into three independently valuable, independently
revertible decisions — which is the same mechanic that makes migrations succeed in
Topic 127 and the same mechanic that makes a strangler work in Topic 128. Once you
see the pattern, you see it everywhere in this phase.

**Adversarial follow-up:** *"Your team is on an LTS that goes out of support in
nine months and the roadmap is full. What do you actually do?"*

They want to know whether you can convert a technical fact into a scheduling
argument. The weak answer asks for cooperation. The strong answer puts a date in
CI and a line in the risk register, and frames it as: unpatched CVEs are an
incident we have chosen to have later. Topic 127 and Topic 134 are both in this
answer.

---

### Q4 — "You read the OpenJDK source and it contradicts the javadoc. What do you do?"

**A senior answer sounds like:** "I'd trust the source, since that is what
actually runs, and probably file a documentation bug."

**A principal answer sounds like:** "I would slow down first, because in my
experience that contradiction is usually me misreading one of the two.

Order of operations. Am I reading the right version — the source at the tag my
runtime is built from, not mainline? Is the javadoc sentence normative or
informative; `@implSpec` and `@implNote` mean different things and only one of
them is a promise? Is there a JDK bug ID in the git history for that method
explaining a deliberate divergence?

If it really is a contradiction, then the practical answer is: for *today's*
behaviour, the source is what runs, so I code to it defensively. For *durable*
behaviour, the specification is what future versions will preserve, so I do not
build an architecture on the divergence. And I would file it — with a test case,
because a bug report with a reproducer gets fixed and a bug report without one
does not.

Then the thing that actually protects us: I put a test in our codebase that
asserts the behaviour we depend on. If a future JDK resolves the contradiction in
the other direction, our build tells us on the upgrade branch instead of our
users telling us in production."

**What separates them:** the senior answer picks a winner. The principal answer
distinguishes *what is true now* from *what will keep being true*, and then leaves
behind an artefact — a test — that converts an unknown into an alarm. Leaving
behind a mechanism rather than a conclusion is most of what this phase is.

**Adversarial follow-up:** *"That test now fails on every JDK upgrade for a
behaviour we do not really care about. Is that a good test?"*

Fair attack. The answer is that the test's assertion message must say **why** the
behaviour matters and what to do if it fails — otherwise it becomes noise and
gets deleted, which is worse than never writing it. A test with no explanation is
a future engineer's annoyance.

---

### Q5 — "A respected engineer on another team is pushing hard for a technology based on a JEP. You disagree. How do you handle it?"

**A senior answer sounds like:** "I'd share my concerns, maybe write up the
trade-offs, and escalate to an architecture forum if we could not agree."

**A principal answer sounds like:** "First I would check whether I actually
disagree or whether I have different constraints. Half the time it is the second,
and then the conversation is short and friendly: their service is scale-to-zero
and startup-dominated, ours is long-running and throughput-dominated, and the
same feature is a good bet for them and a bad one for us. That is not a
disagreement, it is two correct answers.

If it is a real disagreement, I would go to the primary source together rather
than trading opinions. In my experience most of these arguments are two people
having read two different secondary sources, and reading the Non-Goals section
together resolves it in ten minutes.

If we still disagree after that, I would write down what evidence would settle
it and see if it is cheap to get. A three-day prototype that answers the actual
question is worth more than a forum debate, and it means the decision is made by
a fact rather than by whoever is more senior or more persistent.

And I would take seriously that they might be right. The last time I was sure
about something like this, the person disagreeing with me had a constraint I did
not know about, and that constraint changed the design. That is also why the
other teams trusted the outcome."

**What separates them:** the senior answer manages the disagreement. The principal
answer tries to dissolve it — first by separating constraints from opinions, then
by making it empirical, then by being genuinely open to losing. Escalation is
present but last. And notice the final paragraph: a principal engineer's
credibility is partly built out of publicly changing their mind when the evidence
warrants it. This is the same skill Topic 134 is entirely about, and Topic 135
will test it under sustained pressure.

**Adversarial follow-up:** *"The prototype comes back ambiguous. Now what?"*

Ambiguous is a result. The strong answer says so: an ambiguous prototype means the
benefit is not large enough to be visible, which is itself evidence against
adoption under uncertainty. The weak answer runs another prototype.

---

## Mental model checkpoint

Reason these out in writing. Do not look them up, and do not answer in one line.

1. The Non-Goals section of a JEP is written by people who want the feature to
   succeed. What incentive do they have to write it honestly, and what would you
   expect to be *missing* from even an honest Non-Goals section? How would you
   detect the missing thing?

2. Preview features force a recompile on every JDK release. That is obviously a
   cost. Argue that it is also a *feature* of the process — what does it prevent
   from happening to the ecosystem? Then argue the other side.

3. You have two ways to answer a behavioural question: read the source, or write
   a test that asserts the behaviour. Both are better than a blog post. When is
   each one the *wrong* choice? Give a concrete case for each.

4. A JEP has been at Candidate for four years. What are the three most likely
   explanations, and does the right adoption posture differ between them?

5. You wrote an adoption analysis eighteen months ago recommending Defer. The
   trigger has now fired. Nobody has noticed. Whose failure is that, and what
   would you change about the artefact — not about the people — so it does not
   recur?

6. Reading source is a skill that transfers from TypeScript almost perfectly, yet
   Java engineers do it far more than Node engineers do. Why? What is different
   about the two ecosystems that makes the habit more valuable in one? Does that
   difference argue for or against reading source more in Node?

7. Suppose a preview feature would remove a genuine class of production bug in
   `orderflow` — not a convenience, a correctness win. Does that change the
   adoption calculus, or does it only change the weighting? Write the decision
   rule you would apply, and then find the case where your own rule gives an
   answer you would not accept.

---

## Quick reference card

### Primary sources — bookmark all of these

| What | Where |
|---|---|
| All JEPs, by state | `openjdk.org/jeps/0` |
| One JEP | `openjdk.org/jeps/<number>` |
| JDK source | `github.com/openjdk/jdk` |
| JDK bug database | `bugs.openjdk.org` (bug IDs look like `JDK-8123456`) |
| OpenJDK mailing lists | `mail.openjdk.org` — `loom-dev`, `amber-dev`, `core-libs-dev`, `hotspot-gc-dev` |
| JDK release schedule and support | `openjdk.org/projects/jdk/` plus your **vendor's** support page |
| Spring Framework source | `github.com/spring-projects/spring-framework` |
| Spring Boot source | `github.com/spring-projects/spring-boot` |
| Spring support timelines | `spring.io/projects/spring-boot#support` |

### JEP anatomy — read in this order

1. **Non-Goals** — does it decline to solve your problem? If yes, stop.
2. **Motivation** — is their problem your problem?
3. **Alternatives** — what did they reject, and why?
4. **Goals** — what will it actually do?
5. **Description** — how. Read last. This is the part blogs cover, so it is the
   part you least need the primary source for.

### Preview / incubator / experimental

| | Preview | Incubator | Experimental |
|---|---|---|---|
| Covers | language & VM features | library APIs | VM behaviour |
| Enable | `--enable-preview` on `javac` **and** `java`, with `--release N` | `--add-modules jdk.incubator.X` | `-XX:+UnlockExperimentalVMOptions` |
| Package/name changes on graduation | no | **yes** — `jdk.incubator.*` disappears | n/a |
| Class file portable to next release | **no** | yes | n/a |
| Design considered final | yes | **no** | n/a |

### Verified version facts you may rely on in an artefact

| Fact | Value |
|---|---|
| Java 21 | LTS, September 2023 |
| Java 25 | **LTS, GA 16 September 2025** — current LTS |
| Java 26 | GA 17 March 2026, **non-LTS** |
| Spring Framework 7.0 | GA 13 November 2025. JDK 17 baseline, JDK 25 recommended, Jakarta EE 11 |
| Spring Boot 4.0.0 | GA 20 November 2025 |
| Spring Boot 4.1.0 | GA 10 June 2026 |
| Spring Boot 3.5.x | **OSS support ended June 2026** — commercial support only |

Anything not in that table, you look up. Do not quote a date from memory in a
document with your name on it.

### The six questions for any not-yet-final feature

1. What is my problem, in one sentence, written before I read anything?
2. Is my problem in the **Non-Goals**?
3. What do we do today, at its honest best?
4. What does reversal cost — if it fails, *and* if it succeeds?
5. What is the recurring cost, and whose calendar controls it?
6. What fact, with a date or threshold, would change my answer?

### Source-reading checklist

- [ ] Am I on the tag my runtime is actually built from?
- [ ] Is this sentence in the javadoc normative, `@implSpec`, or `@implNote`?
- [ ] Is this package public API, or `internal`/`support`/`sun.*`?
- [ ] Is there a test next to it that asserts the behaviour I care about?
- [ ] Have I captured a **permalink to the line**, not the file?
- [ ] If I depend on this, is it isolated behind one interface I own?

---

## When would I use this at work?

**1. A production question at 11pm that nobody can answer.**
An `orderflow` payment reconciliation job is double-processing under a specific
retry condition, and the argument in the incident channel is about what a
framework annotation does in that case. Two people have opinions. You open the
class, find the branch, paste a permalink, and the argument ends in four minutes
with a fact instead of in forty with a consensus. This is the most common way the
skill pays for itself, and it costs nothing once you know where things live.

**2. An architecture forum where a technology is being sold.**
Someone proposes adopting something on the strength of a benchmark and a
conference talk. You have read the Non-Goals, so you can ask one specific question
that reframes the discussion — not "I disagree", but "the JEP says explicitly it
does not address X; X is 60% of our use case, so what is the plan for that?" One
question, asked from primary sources, changes the outcome of the meeting. This is
how you become the person whose opinion is sought before the meeting rather than
during it.

**3. Planning the year.**
Your manager asks what is coming in Java that would change what `orderflow` looks
like in two years. The answer that lands is not a feature list — it is: "two
things matter to us. One is targeted for an LTS and I have a prototype that says
it removes a class of failure we keep having. The other is Candidate with no date
and I would not plan around it. Here is what I would fund, and here is the trigger
that would make me revisit the second." That answer is short, specific and
actionable, and it is only possible if you read primary sources during the year
rather than the week before the planning meeting.

---

## Connected topics

**Prerequisites — the evidence you will cite:**

- **Topic 65 — the load baseline.** Any performance claim in your artefact is
  measured against this or it is not a claim.
- **Topic 76 — bytecode and `javap`.** Sometimes the fastest way to resolve "what
  does the compiler actually do here" is to look at the bytecode, especially for
  a language feature.
- **Topic 82 — container limits** and **Topic 83 — native image / AOT.** Most JDK
  features that matter to a service interact with one of these. Your artefact's
  section 7 lives here.
- **Topics 91, 92, 101, 108, 119 — the concurrency and context-propagation
  problems** that make structured concurrency and scoped values relevant to
  `orderflow` rather than merely interesting.
- **Topic 124 — the production-readiness review.** The risks named there are the
  problems your adoption analysis should be trying to solve. If a feature does
  not touch anything in that document, ask why you are writing about it.

**This unlocks:**

- **126 — build vs buy vs adopt.** The same evaluation discipline applied to a
  whole framework rather than a language feature, over a five-year horizon,
  including the exit cost. This topic is the small version; 126 is the large one.
- **127 — migration planning.** The JDK's cadence and LTS windows, which you
  learned to read here, are the calendar the entire migration plan is built on.
- **131 — design docs.** The artefact you just wrote is a design doc with a
  narrow subject. The Alternatives and "what would change my mind" sections are
  the same sections, and they are the ones 131 attacks hardest.
- **132 — engineering standards.** The containment mechanism from review Question
  2 — an architecture test that keeps a non-final API inside one package — is a
  standard, and 132 is how you roll one out without stalling delivery.
- **134 — influence without authority.** Being early and correct from primary
  sources is one of the two reliable ways to accumulate technical influence. The
  other is building the thing, which is 134's subject.
- **135 — the interview simulation.** You will be asked to defend this artefact
  under sustained hostile questioning. The Non-Goals mapping and the falsification
  trigger are where the pressure will be applied.

---

*Java baseline 21, running on JDK 25. The mechanics in this topic — the JEP
process, the preview class-file stamp, the six-month train — have been stable for
years and are not expected to change. The specific JEPs referenced will move; that
is the point of the topic. Verify every JEP number and state at `openjdk.org/jeps/0`
before it goes in a document with your name on it.*
