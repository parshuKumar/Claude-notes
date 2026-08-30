# 99 — jcstress: Actually Testing Concurrency Correctness

## Phase: 9 — Concurrency
## Category: ELITE
## Java baseline: 21  |  Notes features from: 21 (jcstress itself is a separate OpenJDK tool, versioned independently of the JDK)
## Project spine: no new `orderflow` capability. This topic supplies the *evidence* for the concurrency claims you have been making since Topic 84 — specifically for the inventory-reservation and wallet-debit critical sections, which are the two contended writes in the system (Topic 52).

---

## Mechanical statement

Read this twice. Everything below is elaboration.

> **jcstress runs a tiny pair of `@Actor` methods on two real threads, over the same
> tiny `@State` object, millions of times, in several separately-forked JVMs with
> different JIT settings. After both actors finish, an `@Arbiter` reads the state into
> a result object. jcstress tallies how many times each distinct result appeared and
> prints the FREQUENCY of every OBSERVED outcome against the outcomes you declared
> `ACCEPTABLE`, `ACCEPTABLE_INTERESTING` or `FORBIDDEN`.**
>
> **It finds interleavings a unit test never will — and it still proves nothing about
> what it did not observe.**

Two sentences. The first is why you use it. The second is why you must never present
a green jcstress run as a proof.

---

## The two honesty rules — state these out loud before you run anything

These are not caveats bolted onto the end. They are the reason this topic is ELITE
rather than "here is a testing tool."

### Honesty rule 1 — a `FORBIDDEN` outcome with zero observations is not a proof

If you declare an outcome `FORBIDDEN` and jcstress reports that it never appeared,
the only sentence you are entitled to say is:

> "That outcome was **not observed** on this machine, on this JDK, on this run."

You are **not** entitled to say "that outcome is impossible", "the code is correct",
or "we proved it can't happen." jcstress samples a space it cannot enumerate. The
space includes every interleaving of two threads, every reordering the compiler is
licensed to perform, every reordering the CPU is licensed to perform, every cache
state, every scheduling decision the OS makes, and the JIT's decision about whether
to compile the method at all this run.

**The proof is the happens-before argument.** You write down, in words, which
happens-before edge (Topic 86) makes the bad outcome unreachable: a monitor
unlock→lock edge, a volatile write→read edge, a final-field freeze, a
`Thread.start`/`join` edge. That argument is the proof.

**jcstress is how you catch the argument being wrong.** It is a falsifier, not a
verifier. A `FORBIDDEN` outcome that *does* appear is conclusive: your argument is
wrong, right now, with a counterexample. A `FORBIDDEN` outcome that does not appear
is a failure to falsify, which is a much weaker thing.

Compare it to how you already think about tests: a failing test proves a bug exists;
a passing test does not prove no bug exists. jcstress is the same asymmetry, turned
up to a scale where the asymmetry actually bites.

### Honesty rule 2 — x86 is Total Store Order; aarch64 is not, and you are probably on aarch64

This one has a direct consequence for you, personally, today.

**x86 and x86-64 implement a memory model called Total Store Order (TSO).** In
practice, on x86, the hardware does not reorder a store with a later store, and does
not reorder a load with a later load. The only reordering the hardware performs is
store-then-load (a later load may be satisfied from cache before an earlier store
drains the store buffer). This means **a large class of memory-ordering bugs simply
does not manifest on x86 hardware**, even though the Java Memory Model plainly
permits them and even though the JIT may still perform the reordering in software.

**aarch64 — the ARM architecture your Apple Silicon Mac runs — is weakly ordered.**
Stores can become visible to other cores out of program order. Loads can be satisfied
out of program order. The hardware needs explicit barrier instructions
(`dmb`, `ldar`, `stlr`) to enforce what x86 gives you for free.

Which way does this cut for you?

| Situation | What it means |
|---|---|
| You are on an Apple Silicon Mac (M-series), so aarch64 | You are on the **more revealing** platform. Reordering bugs that a colleague on an x86 laptop cannot reproduce may show up for you. That is a gift, not a nuisance. |
| Your CI runs on x86 and production runs on AWS Graviton (aarch64) | This is a genuine, common, and expensive gap. Your CI is structurally less able to find these bugs than your production hardware is to hit them. |
| Your CI runs on aarch64 and production is x86 | The gap is the other way and less dangerous, but a "found on ARM only" bug is still a real bug: the JMM permits it, so a future JIT change can make it appear on x86 too. |
| You are on an Intel Mac | You are on the less revealing platform. Get an aarch64 runner into the loop before you trust a clean result. |

**What you must never say:** "run it on ARM and you'll see the bug." You will not
know that. The JIT might not have compiled the method. The two threads might not have
overlapped in the four-instruction window that matters. The barrier the JIT emitted
might have been conservative. Say instead: "this platform is *capable* of exposing
it; a clean run is weaker evidence on x86 than on aarch64."

Check which you are on:

```bash
uname -m          # arm64 on Apple Silicon, x86_64 on Intel
java --version    # record this too; it is part of the result
```

Record both numbers next to every jcstress result you keep. A result without its
platform and JDK version is not a result.

---

## The bridge from what you know

### The closest thing you have: property-based testing

You met property-based testing at Topic 64 (and you may know `fast-check` from the
TypeScript side). The shape is familiar:

```ts
import fc from 'fast-check';

test('order total is never negative', () => {
  fc.assert(
    fc.property(fc.array(fc.integer({ min: 0, max: 1000 })), (lineTotals) => {
      return computeOrderTotal(lineTotals) >= 0;
    }),
    { numRuns: 10_000 },
  );
});
```

`fast-check` generates thousands of inputs, runs your property against each, and
shrinks any counterexample it finds. It does not enumerate the input space; it samples
it, cleverly. A pass means "not falsified in 10,000 samples."

That last sentence is exactly jcstress's epistemics. **Verdict: PARTIAL analogue —
and a genuinely useful one.**

**What transfers:**

- You declare the *property* (jcstress: the set of acceptable outcomes), not the
  expected value of one run.
- The tool searches, you do not enumerate.
- A pass is a failure to falsify, not a proof.
- A counterexample is conclusive and precious.

**What does not transfer, and this is the whole reason jcstress exists:**

`fast-check` searches a space you can describe: the space of *inputs*. You can write
`fc.integer({min: 0, max: 1000})` because you know what an input is.

jcstress searches a space you **cannot** describe: the space of *interleavings and
memory-visibility outcomes*. There is no generator for "the store buffer drained
between these two instructions on core 3 but not on core 7 while C2 had hoisted the
load out of the loop." You cannot write that down. You can only run the shape
millions of times, in several JIT configurations, and count what came out.

### What has NO analogue in your Node experience at all

Nothing in a single-threaded event loop can produce this class of bug, so nothing in
your Node toolchain addresses it.

Your run-to-completion guarantee (Topic 84) means a callback is never interrupted
mid-way. Two callbacks never observe each other's half-written state, because there
is no "meanwhile." Worker threads have isolated heaps and communicate by copying, so
they are message passing, not shared memory.

The result: **JavaScript has essentially no data races, so JavaScript has no memory
model to test against, so JavaScript has no jcstress.** There is no tool you have used
that this is like. `SharedArrayBuffer` plus `Atomics` is the one corner of JS with a
real memory model, and almost nobody writes it.

**Verdict: NO ANALOGUE.** This is new machinery for a new class of problem.

### The instinct you must actively unlearn

In Node, "I ran it 100,000 times and it always worked" is decent evidence, because
the number of distinct execution shapes is small and you probably covered them.

In Java, "I ran it 100,000 times on 100 threads and it always worked" is close to no
evidence at all. Here is why, spelled out, because this is the single most common
senior-level mistake in the whole phase:

1. **You tested one machine.** Specifically, one memory model. If it is x86, whole
   categories of reordering were unreachable by construction.
2. **You tested one JIT state.** Your loop may have run entirely in the interpreter,
   or in C1, or been compiled to C2 halfway through. The interpreter reorders nothing.
   C2 reorders aggressively. You do not know which one you measured.
3. **You tested one thread-arrival pattern.** 100 threads started in a loop do not
   arrive at the critical section simultaneously; they arrive in a staircase, because
   `Thread.start()` costs real time. The racy window is typically two to five
   instructions wide. Your threads mostly missed it.
4. **You tested a machine that was doing other things.** The scheduler's decisions
   were shaped by your browser, your IDE and your Docker daemon.
5. **You had luck.** That is not a joke. It is the actual technical situation.

jcstress attacks every one of those five. That is what it is for.

---

## What is this?

**jcstress** — the Java Concurrency Stress tests — is an OpenJDK project. It is a
harness for writing *very small* concurrency experiments and grading their outcomes
statistically. It was built by the OpenJDK team to test the JVM's own memory-model
implementation; you can use the same harness on your own code.

It is not a JUnit extension. It is not something you run in your normal test suite.
It builds its own self-contained jar and you run that jar directly, and a serious run
takes minutes to hours because it is deliberately trying to be unlucky on your behalf.

### The vocabulary — six annotations and one result object

| Thing | What it is |
|---|---|
| `@JCStressTest` | Marks the class as a test. jcstress's annotation processor generates a runner for it at build time. |
| `@State` | Marks the object holding the mutable state under test. Every field you race on lives here. jcstress allocates *many* of these and pads them. |
| `@Actor` | A method run by one thread, exactly once per state instance. Two or more actors race. |
| `@Arbiter` | A method run *after* every actor has finished, once, to read the final state into the result object. |
| `@Outcome` | Declares a result value you expect, with an `Expect` grade and a description. |
| `@Description` | Free text describing the test. Shows up in the report. |
| `XX_Result` | A pre-generated, padded carrier class with public fields `r1`, `r2`, .... `I_Result` holds one `int`; `II_Result` holds two; `ZZ_Result` holds two `boolean`s; `LL_Result` holds two references. From `org.openjdk.jcstress.infra.results`. |

### The four grades

| `Expect` value | Meaning | Effect on the run |
|---|---|---|
| `ACCEPTABLE` | A legal, boring outcome. | Fine. |
| `ACCEPTABLE_INTERESTING` | A legal outcome that surprises people, or that you specifically want to see appear. | Fine, but highlighted in the report. |
| `FORBIDDEN` | An outcome your happens-before argument says cannot occur. | If it appears **even once**, the test FAILS. |
| `UNKNOWN` | You did not declare this outcome at all. | The test FAILS. jcstress refuses to let you silently ignore an outcome you never thought about. |

That last row matters more than it looks. jcstress will not let you write a test that
passes because you forgot a case. Every observed result must be classified by you.

### The two test modes

- **`Mode.Continuous`** (the default): actors race; the arbiter reads the outcome.
  This is what you will use 95% of the time.
- **`Mode.Termination`**: one actor spins waiting for something; a `@Signal` method
  tries to make it stop. The test grades whether the actor *terminated*. This is the
  correct shape for the non-`volatile` stop-flag bug from Topic 86 — the failure mode
  there is "the loop never ends", which cannot be expressed as a result value.

---

## Why does it matter?

Three reasons, in increasing order of career relevance.

### 1. It is the only way to falsify a memory-model claim about your own code

You have spent Topics 86 to 98 learning to write happens-before arguments. Those
arguments are the actual engineering artefact. But an argument you never tried to
break is an opinion. jcstress is how you try to break it, at a scale where trying is
meaningful.

### 2. It changes what you are allowed to say in a code review

Before this topic, the strongest thing you could say about a critical section was
"I believe this is correct because of the monitor edge on line 40." After this topic
you can say: "I believe this is correct because of the monitor edge on line 40, and
here is a jcstress test that declares the interleaving-visible outcome `FORBIDDEN` and
did not observe it in N iterations across interpreted, C1 and C2 modes on aarch64.
That is not a proof, but it is what falsification looks like when it fails."

That is a different level of engineer. It is also, precisely, the difference the
interview is probing (see the interview section).

### 3. `orderflow`'s two contended writes are worth this level of care

Inventory decrement and wallet debit. A visibility bug in either one produces a
business outcome, not a stack trace:

- **Inventory:** you sell stock you do not have. The customer is charged, the
  warehouse has nothing to pick, and you find out from a support ticket days later.
- **Wallet:** you debit twice, or you debit and lose the record of the debit. Both
  are money. Both end up in a reconciliation report that does not balance, which is
  the most expensive kind of bug to investigate because the evidence is a number, not
  an exception.

Neither produces a log line at the moment it happens. That is the defining property
of a memory-visibility bug and the reason this class of bug survives to production.

---

## Machine-level reality

This section is what separates "I've heard of jcstress" from "I know why it works."

### The generated runner

jcstress ships an annotation processor. At build time, for each `@JCStressTest` class,
it generates a runner class. You never see this class unless you look for it in
`target/generated-sources`, and looking at one once is a genuinely good use of ten
minutes.

The generated runner does roughly this:

1. **Allocate an array of `@State` instances** — thousands of them, not one. Each
   actor thread walks the same array.
2. **Pad each state instance.** jcstress inserts padding fields so that two adjacent
   state objects do not share a cache line (Topic 96). Without this, false sharing
   between unrelated iterations would distort the timing so badly that the interesting
   window closes.
3. **Start N actor threads**, one per `@Actor` method, and hold them at a barrier.
4. **Release the barrier.** Each thread runs a tight loop: for each index `i` in its
   slice, call its actor method on `states[i]`.
5. **Join.** Then run the `@Arbiter` for each index, writing into a
   correspondingly-padded result object.
6. **Tally.** Each distinct result tuple is counted into a histogram.

Two design decisions in there are worth naming:

**Why an array of states rather than one?** Because setting up a rendezvous for a
single shared object costs far more than the racy window itself. By handing each
thread a long run of independent state objects, the threads settle into a steady state
where they are genuinely executing concurrently, and their relative progress drifts
naturally across the array. That drift is the search: sometimes actor 1 is ahead,
sometimes actor 2 is, sometimes they are on the same index at the same nanosecond.

**Why padding?** Two reasons. Between state objects, padding prevents false sharing
from serialising the threads. Inside the result objects, padding prevents the tally
from perturbing what it measures.

### Why it forks multiple JVMs

This is the part most people miss, and it is the strongest single argument for the
tool.

A concurrency bug can be introduced at four different layers:

| Layer | What it can do | How to isolate it |
|---|---|---|
| Your source | An actual missing lock or missing `volatile` | Read the code |
| The interpreter | Nothing. It executes bytecode in order. | Run with `-Xint` |
| C1 (the fast, low-optimisation JIT) | Modest reordering, some hoisting | Run with `-XX:TieredStopAtLevel=1` |
| C2 (the aggressive JIT) | Load hoisting out of loops, store sinking, redundant-load elimination, full reassociation | Run normally with a long warm-up |
| The CPU | Store buffering (all), store-store and load-load reordering (aarch64, not x86) | Run on the architecture in question |

jcstress **forks a fresh JVM for each configuration** and runs the same test in each.
That is why a serious run takes minutes: it is not one long run, it is several
independent runs under different compilation regimes, plus repetitions to get the
statistics to settle.

This directly explains something you already saw. Topic 86's drill — the non-`volatile`
stop flag — terminates under `-Xint` and hangs under normal execution. That gap is not
mysterious once you know that the interpreter re-reads the field every iteration while
C2 is licensed to hoist the read out of the loop entirely, because nothing in the
program establishes that another thread might change it. jcstress systematises exactly
that comparison.

There is a second reason for forking: a fresh JVM has a fresh, unpolluted profile.
Topic 74 taught you that a polluted call site permanently degrades compiled code. If
jcstress ran all its tests in one JVM, earlier tests would shape the JIT's decisions
for later ones and you would not know which result belonged to which condition.

### How the outcome ids are formed

The `id` on an `@Outcome` is a string. For a multi-field result it is the fields
joined with `", "` — comma, space — in field order `r1`, `r2`, ....

```java
@Outcome(id = "0, 1", expect = FORBIDDEN, desc = "...")
```

means `r1 == 0 && r2 == 1`.

**The ordering is a real trap.** Write `"1, 0"` when you meant `"0, 1"` and jcstress
does not tell you that you made a typo; it reports the actual outcome as `UNKNOWN` and
fails the test, and you spend twenty minutes thinking you have found a JVM bug. The
`id` also accepts a regular expression, which is occasionally useful and occasionally
the cause of an outcome being swallowed by an over-broad pattern.

### The `@Arbiter`'s happens-before guarantee

The arbiter runs after all actors have finished, and jcstress establishes proper
happens-before edges between actor completion and arbiter execution. So the arbiter
sees the *final* state, not a racy view of it.

This matters when you reason about which outcomes are even declarable. In the plain
`units++` test below, the arbiter cannot observe `0`: both actors ran to completion,
each performed at least one increment, and the arbiter has an edge from both. The
observable outcomes are `1` (one increment lost to a read-modify-write interleaving)
and `2` (both landed). If you find yourself declaring an outcome the arbiter
structurally cannot see, you have misunderstood the harness.

### What jcstress does not do

Be clear about the boundaries:

- **It does not model-check.** There is no exhaustive state-space exploration, no
  symbolic reasoning, no SMT solver. It is stochastic search. (Tools that *do* model
  check the JMM exist in research; none of them scale to your service.)
- **It does not test your application.** It tests a shape you extracted from your
  application. Extracting the shape faithfully is your job and is where the real skill
  is.
- **It does not measure performance.** The `FREQ` column is a frequency of outcomes,
  not a throughput number. Reading it as performance is a category error.
- **It does not scale to large critical sections.** If your actor does a HashMap
  lookup, a log call and a database round trip, the two threads will essentially never
  overlap in the window that matters, and every run will report the boring outcome.

---

## Concurrency trace

Before any correct code, here is the bug, step by step, on two threads.

### The scenario

`orderflow` reserves inventory when an order is placed. The reservation is recorded on
a small object shared between the order-placement path and the fulfilment path:

```java
// The version that ships. Both fields are plain: not volatile, not final.
public class InventoryReservation {
    int  reservedUnits;     // plain field
    boolean reserved;       // plain field

    // Called by the order-placement thread
    void reserve(int units) {
        reservedUnits = units;      // (W1) write the quantity
        reserved = true;            // (W2) then flip the flag
    }
}
```

And the fulfilment path reads it:

```java
    // Called by the fulfilment thread
    int unitsToPick() {
        if (reserved) {             // (R1) read the flag
            return reservedUnits;   // (R2) then read the quantity
        }
        return -1;                  // not yet reserved
    }
```

The author's reasoning was: "I write the quantity first and set the flag second, so
anyone who sees the flag must see the quantity." That reasoning is **wrong**, and it
is wrong for two independent reasons, either of which is sufficient.

### The interleaving

Two threads. `T1` is the order-placement thread. `T2` is the fulfilment worker.
Initial state: `reservedUnits = 0`, `reserved = false`.

| Step | T1 — order placement (`reserve(5)`) | T2 — fulfilment (`unitsToPick()`) |
|---|---|---|
| 1 | About to execute `reservedUnits = 5` (W1) | — |
| 2 | The JIT has sunk W1 below W2, **or** the store buffer has not drained W1 to memory yet. Either way, W1 is not visible to T2. | — |
| 3 | Executes `reserved = true` (W2). This store becomes visible to T2. | — |
| 4 | — | Executes `if (reserved)` (R1). Reads `true`. |
| 5 | — | Executes `return reservedUnits` (R2). Reads **0** — the value from before W1, because W1 has not become visible. |
| 6 | W1 finally becomes visible to T2. Too late. | Returns 0. |

There is a second, entirely separate route to the same outcome, on the *reader* side:

| Step | T1 — order placement | T2 — fulfilment |
|---|---|---|
| 1 | — | The JIT has hoisted R2 above R1 (it is allowed to: nothing in T2's code says the two reads are ordered), **or** aarch64 hardware satisfies R2 out of order from a cache line it already owns. T2 reads `reservedUnits` → **0**. |
| 2 | Executes W1: `reservedUnits = 5` | — |
| 3 | Executes W2: `reserved = true` | — |
| 4 | — | Executes R1: reads `reserved` → `true` |
| 5 | — | Returns the value it read at step 1: **0** |

Both routes end in the same place, and neither requires the two threads to be
"interleaved" in the naive sense of the word. Nothing was interrupted mid-statement.
The statements executed atomically. They just did not become visible in the order they
were written, and **the Java Memory Model explicitly permits that**, because you never
established a happens-before edge between the write of `reservedUnits` and the read of
it.

### Outcome, in business terms

An order is marked reserved with **zero units held**.

Downstream, the picking system receives a reservation for 0 units and picks nothing.
The customer's card is already charged. The order sits in `RESERVED` state with an
empty pick list. Nobody is paged, because nothing threw. Inventory is not
double-counted, so no reconciliation job flags it. The first signal is a customer
emailing three days later asking where their order is.

Now multiply by 5 million order lines. If the window is hit once in ten million, you
have a handful of these per week, spread across your customer base, with no
correlation to any deploy. That is the actual shape of this bug in production, and it
is why "we ran it 100,000 times and it was fine" is not a defence.

**Frequency estimate:** none. I will not give you one. It depends on the JIT's
decisions, the architecture, the cache state and the load pattern. That unknowability
is exactly the reason jcstress exists.

---

## Example 1 — minimal

The smallest useful jcstress test: does a plain `units++` lose an increment?

You already know from Topic 87 that `i++` is a read-modify-write and therefore not
atomic. This is how you *see* that, rather than assert it.

### Getting a project

jcstress needs its annotation processor on the compile path and its own runner in the
output jar. The maintained way to get that is the archetype:

```bash
mkdir -p ~/java-lab/99 && cd ~/java-lab/99

mvn archetype:generate \
  -DinteractiveMode=false \
  -DarchetypeGroupId=org.openjdk.jcstress \
  -DarchetypeArtifactId=jcstress-java-test-archetype \
  -DgroupId=com.orderflow \
  -DartifactId=orderflow-jcstress \
  -Dversion=1.0
```

> **Version note.** I have deliberately not pinned a jcstress version here. The
> archetype and the `jcstress-core` dependency move independently of the JDK, and a
> version I quote today will be stale. Open the jcstress project page
> (`github.com/openjdk/jcstress`) and take the current version from its README, then
> pin it in your POM. If the archetype coordinates have changed, the README is also
> the place that will say so. Do not copy a version number out of a teaching document.

Then confirm what you actually got:

```bash
cd orderflow-jcstress
grep -A2 jcstress pom.xml     # see the version the archetype chose
java --version
uname -m
```

Write those three things down. They are part of every result you record.

### The test

`src/main/java/com/orderflow/PlainCounterIncrement.java`:

```java
package com.orderflow;

import org.openjdk.jcstress.annotations.Actor;
import org.openjdk.jcstress.annotations.Arbiter;
import org.openjdk.jcstress.annotations.Description;
import org.openjdk.jcstress.annotations.JCStressTest;
import org.openjdk.jcstress.annotations.Outcome;
import org.openjdk.jcstress.annotations.State;
import org.openjdk.jcstress.infra.results.I_Result;

import static org.openjdk.jcstress.annotations.Expect.ACCEPTABLE;
import static org.openjdk.jcstress.annotations.Expect.ACCEPTABLE_INTERESTING;

@JCStressTest
@Description("Two plain increments of a shared int. Read-modify-write is not atomic.")
@Outcome(id = "2", expect = ACCEPTABLE,
         desc = "Both increments landed. The threads did not overlap in the window.")
@Outcome(id = "1", expect = ACCEPTABLE_INTERESTING,
         desc = "One increment was LOST. Both actors read the same value, "
              + "both incremented it, both wrote the same result back.")
@State
public class PlainCounterIncrement {

    int units;

    @Actor
    public void reserveOne() {
        units++;
    }

    @Actor
    public void reserveTwo() {
        units++;
    }

    @Arbiter
    public void readFinal(I_Result r) {
        r.r1 = units;
    }
}
```

Read what each part is doing:

| Line | Why it is there |
|---|---|
| `@State` on the class | The class *is* the state. jcstress will allocate thousands of these. |
| `int units` — no `volatile`, no `final` | The subject of the test. If you make it `volatile`, the lost update still happens, which is Topic 87's point and a worthwhile second run. |
| Two `@Actor` methods | Two threads. jcstress decides scheduling; you do not. |
| `@Arbiter` taking `I_Result` | Runs after both actors, with happens-before edges from both, so it reads the settled value. |
| `id = "1"` graded `ACCEPTABLE_INTERESTING` | It is legal — that is the whole complaint — but it is the outcome you are hunting, so you want the report to shout about it. |
| No `id = "0"` declared | The arbiter cannot see `0`. If it somehow did, the test would fail as `UNKNOWN`, which is correct behaviour: an outcome I did not think about must not pass silently. |

### Build and run

```bash
mvn clean verify
java -jar target/jcstress.jar -t PlainCounterIncrement
```

`-t` takes a regular expression matched against the test class name. Without it,
jcstress runs everything, which for a real project is a long lunch.

Useful flags — but **confirm them against your version**, because the CLI has changed
across releases:

```bash
java -jar target/jcstress.jar -h            # do this once, first
java -jar target/jcstress.jar -l            # list the tests it found
java -jar target/jcstress.jar -t PlainCounter -m quick     # shorter run
java -jar target/jcstress.jar -t PlainCounter -v           # verbose per-config output
java -jar target/jcstress.jar -t PlainCounter -r results/  # where the HTML report goes
```

### Reading the outcome table

jcstress prints a table per test configuration. Here is its **column structure**:

```
RESULT      SAMPLES     FREQ       EXPECT  DESCRIPTION
     1          <n>      xxx%  INTERESTING  One increment was LOST.
     2          <n>      xxx%   ACCEPTABLE  Both increments landed.
```

*illustration of the format, not captured output*

I have no JVM, so `<n>` and `xxx%` are placeholders. I am not going to invent sample
counts, because a plausible-looking number would teach you to expect a particular
frequency, and the frequency is exactly the thing that varies by machine, JDK and
architecture.

| Column | What it is |
|---|---|
| `RESULT` | The result tuple, formatted the same way as your `@Outcome` `id`. |
| `SAMPLES` | How many times this exact outcome was observed across the whole run. |
| `FREQ` | That count as a percentage of all observations. **Not a performance number.** |
| `EXPECT` | The grade you assigned. `UNKNOWN` here means you did not declare it. |
| `DESCRIPTION` | Your `desc` text, echoed back. Write these for a reader who is not you. |

### How to read what you get

| What you see | What it means |
|---|---|
| Both `1` and `2` appear, `2` far more common | The expected shape. You have observed a lost update directly. Note the platform and JDK next to it. |
| Only `2` appears; `1` has zero samples | The window was not hit on this run. That is **not** evidence `units++` is atomic. Try `-m stress`, close other applications, and re-read Honesty rule 1. |
| A result you did not declare, marked `UNKNOWN` | The test fails, correctly. Work out how that outcome is reachable before you declare it — the thinking is the point. |
| The whole run errors out at build time | The annotation processor did not run. Check that `mvn clean verify` was used (not `mvn compile`) and that `target/jcstress.jar` exists. |

---

## Example 2 — production scenario (on the project spine)

Now the real one: the `InventoryReservation` publication bug from the trace above,
encoded as a jcstress test, then fixed three ways with the argument for each.

### The broken version, as a test

`src/main/java/com/orderflow/ReservationPublicationPlain.java`:

```java
package com.orderflow;

import org.openjdk.jcstress.annotations.Actor;
import org.openjdk.jcstress.annotations.Description;
import org.openjdk.jcstress.annotations.JCStressTest;
import org.openjdk.jcstress.annotations.Outcome;
import org.openjdk.jcstress.annotations.State;
import org.openjdk.jcstress.infra.results.II_Result;

import static org.openjdk.jcstress.annotations.Expect.ACCEPTABLE;
import static org.openjdk.jcstress.annotations.Expect.ACCEPTABLE_INTERESTING;

@JCStressTest
@Description("orderflow inventory reservation published through two PLAIN fields. "
           + "The fulfilment reader can see the flag without the quantity.")
// r1 = the flag the reader saw, as 0/1.  r2 = the quantity the reader saw.
@Outcome(id = "0, 0", expect = ACCEPTABLE,
         desc = "Reader ran entirely before the writer. Nothing reserved yet. Correct.")
@Outcome(id = "1, 5", expect = ACCEPTABLE,
         desc = "Reader saw a fully published reservation. Correct.")
@Outcome(id = "0, 5", expect = ACCEPTABLE,
         desc = "Reader saw the quantity but not the flag. Harmless here: "
              + "the reader ignores the quantity when the flag is false.")
@Outcome(id = "1, 0", expect = ACCEPTABLE_INTERESTING,
         desc = "OVERSELL. Flag visible, quantity not. The order is marked reserved "
              + "with zero units held and the warehouse picks nothing.")
@State
public class ReservationPublicationPlain {

    int     reservedUnits;   // plain
    boolean reserved;        // plain

    @Actor
    public void orderPlacement() {
        reservedUnits = 5;   // W1
        reserved = true;     // W2
    }

    @Actor
    public void fulfilment(II_Result r) {
        r.r1 = reserved ? 1 : 0;   // R1
        r.r2 = reservedUnits;      // R2
    }
}
```

Two things to notice about how this is written.

**There is no `@Arbiter`.** The reader *is* an actor, and it writes straight into the
result. That is the correct shape when what you care about is what a concurrent reader
observed, not what the final state settled to. An arbiter would tell you the state
ended up consistent — which it always does — and would tell you nothing about the
window.

**`1, 0` is graded `ACCEPTABLE_INTERESTING`, not `FORBIDDEN`.** This is deliberate and
it is a point of craft. The outcome is *legal* under the JMM: I have not established
any edge that rules it out, so declaring it `FORBIDDEN` would be declaring my own code
correct by fiat, and the test would fail for the right reason but with the wrong
label. `ACCEPTABLE_INTERESTING` says: this is permitted, and I want the report to shout
when it happens. **`FORBIDDEN` belongs on the fixed version**, where I have an argument
for it.

### The fixed versions

#### Fix 1 — `volatile` on the flag

`src/main/java/com/orderflow/ReservationPublicationVolatile.java`:

```java
package com.orderflow;

import org.openjdk.jcstress.annotations.Actor;
import org.openjdk.jcstress.annotations.Description;
import org.openjdk.jcstress.annotations.JCStressTest;
import org.openjdk.jcstress.annotations.Outcome;
import org.openjdk.jcstress.annotations.State;
import org.openjdk.jcstress.infra.results.II_Result;

import static org.openjdk.jcstress.annotations.Expect.ACCEPTABLE;
import static org.openjdk.jcstress.annotations.Expect.FORBIDDEN;

@JCStressTest
@Description("Same reservation, published through a VOLATILE flag. "
           + "The volatile write/read pair is the happens-before edge.")
@Outcome(id = "0, 0", expect = ACCEPTABLE, desc = "Reader ran before the writer.")
@Outcome(id = "1, 5", expect = ACCEPTABLE, desc = "Reader saw the full reservation.")
@Outcome(id = "0, 5", expect = ACCEPTABLE,
         desc = "Reader raced the flag but read the quantity late. Harmless.")
@Outcome(id = "1, 0", expect = FORBIDDEN,
         desc = "OVERSELL. Ruled out by the volatile write -> volatile read edge: "
              + "everything before the volatile write is visible to any thread that "
              + "reads true from it.")
@State
public class ReservationPublicationVolatile {

    int              reservedUnits;   // still plain
    volatile boolean reserved;        // the edge lives here

    @Actor
    public void orderPlacement() {
        reservedUnits = 5;   // W1 — ordinary write
        reserved = true;     // W2 — VOLATILE write: releases everything above it
    }

    @Actor
    public void fulfilment(II_Result r) {
        r.r1 = reserved ? 1 : 0;   // R1 — VOLATILE read: acquires
        r.r2 = reservedUnits;      // R2 — ordinary read
    }
}
```

**Write the happens-before argument down before you run it.** That argument is the
proof; the run is the falsification attempt.

> `reservedUnits = 5` precedes `reserved = true` in program order on the writing
> thread. `reserved` is `volatile`, so the write to it is a *release*: every action
> that precedes it in program order is ordered before it, and the JIT may not sink an
> ordinary store below it. On the reading thread, `reserved` is read as an *acquire*:
> the JIT may not hoist the subsequent ordinary load above it, and the hardware barrier
> the JIT emits prevents the CPU from doing so either. Therefore if the reader observes
> `reserved == true`, the volatile read has synchronised-with the volatile write, which
> gives a happens-before edge, which makes `reservedUnits = 5` visible. `1, 0` is
> unreachable.

Notice that the argument names the writer side *and* the reader side. Both halves are
needed. A `volatile` write with a plain read gives you nothing.

#### Fix 2 — `synchronized` on both sides

```java
    int     reservedUnits;
    boolean reserved;

    @Actor
    public void orderPlacement() {
        synchronized (this) {
            reservedUnits = 5;
            reserved = true;
        }
    }

    @Actor
    public void fulfilment(II_Result r) {
        synchronized (this) {
            r.r1 = reserved ? 1 : 0;
            r.r2 = reservedUnits;
        }
    }
}
```

Argument: the monitor unlock at the end of the writer's block happens-before any
subsequent monitor acquire of the same monitor. Whichever thread runs second sees
everything the first one did. `1, 0` is unreachable, and so is `0, 5` — the reader now
sees a consistent snapshot or nothing, never a torn one.

That last part is the difference worth noticing. `volatile` fixes *visibility*.
`synchronized` fixes visibility **and** gives you an atomic snapshot of both fields.
If your invariant spans two fields — and this one does — mutual exclusion is the more
honest tool, and the outcome table will show it: the `0, 5` row disappears.

> One forward-looking caveat you will need in two topics' time: `synchronized` here is
> correct, but on a virtual thread a `synchronized` block held across a blocking call
> is the pinning problem (Topic 101), and `ReentrantLock` is the fix. Correctness first;
> Topic 101 deals with the scheduling consequence.

#### Fix 3 — publish an immutable record through a volatile reference

```java
    record Reservation(int units, boolean reserved) { }

    volatile Reservation state = new Reservation(0, false);

    @Actor
    public void orderPlacement() {
        state = new Reservation(5, true);   // one volatile write, one object
    }

    @Actor
    public void fulfilment(II_Result r) {
        Reservation s = state;               // one volatile read
        r.r1 = s.reserved() ? 1 : 0;
        r.r2 = s.units();
    }
}
```

Argument: a record's components are `final` fields, so the constructor's freeze action
(Topic 88) guarantees any thread that sees the reference sees the correctly-initialised
fields. The reference itself is published through a volatile write, giving the edge.
And because both fields travel inside one object, the reader gets an atomic snapshot
without a lock at all.

This is the version I would actually ship. It removes the two-field invariant rather
than defending it, which is a design fix rather than a synchronisation fix. It also
costs one small allocation per reservation, which at `orderflow`'s order rate is
irrelevant — but say so out loud in the PR rather than letting someone else raise it.

### Running the pair and comparing

```bash
mvn clean verify
java -jar target/jcstress.jar -t ReservationPublication
```

The regex matches all three classes, so you get all their tables in one report.

**What to compare, in this order:**

1. Does `1, 0` appear at all in the **plain** version's table?
2. Does the **volatile** version's `FORBIDDEN` row appear? (It must not. If it does,
   your argument is wrong — go find out why, that is the most valuable outcome
   available to you today.)
3. Does the `0, 5` row disappear in the `synchronized` and record versions? That is
   the atomicity difference made visible.

**And the sentence you are allowed to write in the PR:**

> "On aarch64 / JDK `<your version>` / jcstress `<your version>`, the plain-field
> version was observed producing `1, 0`. The volatile version declares `1, 0`
> `FORBIDDEN` on the argument that the volatile write/read pair supplies the
> happens-before edge, and `1, 0` was not observed. That is a failure to falsify, not
> a proof; the proof is the argument in the `@Outcome` description."

If `1, 0` was *not* observed in the plain version either, the honest sentence is
different and still useful:

> "The plain-field version's `1, 0` outcome was not observed on this machine. The
> outcome is nonetheless permitted by the JMM and this platform is capable of
> producing it; the code is incorrect regardless of whether this run caught it."

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — concluding the code is thread-safe because a 100-thread loop passed

**Wrong:**

```java
@Test
void inventoryDecrementIsThreadSafe() throws Exception {
    Inventory inventory = new Inventory(100_000);
    ExecutorService pool = Executors.newFixedThreadPool(100);
    CountDownLatch done = new CountDownLatch(100_000);

    for (int i = 0; i < 100_000; i++) {
        pool.submit(() -> { inventory.decrement(1); done.countDown(); });
    }
    done.await();
    assertEquals(0, inventory.available());   // passes. Every time. For months.
}
```

**Exact symptom:** the test is green in CI on every commit for eight months. Then
production, running on different hardware under a different load shape, produces a
handful of orders per week whose reserved quantity does not match the order's line
total. There is no exception, no error metric, no correlation with a deploy. The
finance team notices before engineering does, from a stock-reconciliation variance.

**Root cause:** five things at once, and you can only fix them by changing tools.
(1) One machine, therefore one memory model — if it is x86, entire reordering classes
were unreachable. (2) One JIT state, unknown and unrecorded. (3) A thread-arrival
staircase: 100 threads submitted in a loop do not converge on a two-instruction window.
(4) A noisy machine whose scheduler decisions were shaped by everything else running.
(5) Luck, which is a technical fact here and not a joke.

**Fix:** two steps, in this order, and the order is the point.

1. **Write the happens-before argument in prose.** Name the edge. If you cannot name
   it, the code is wrong and no test is required to establish that.
2. **Encode the invariant as a jcstress test** with the bad outcome declared
   `FORBIDDEN`, and run it across JIT modes on the architecture that matters. Keep the
   loop test too — it is a fine smoke test — but stop presenting it as evidence of
   thread safety.

The sentence to retire from your vocabulary: "it passed under load, so it is
thread-safe." The replacement: "here is the edge that makes it correct, and here is
the falsification attempt that failed."

---

### Trap 2 — reading zero `FORBIDDEN` observations as a proof

**Wrong:** a PR description that says "jcstress passes, so the lock-free path is
proven correct."

**Exact symptom:** delayed, and this is what makes it dangerous. The immediate symptom
is a merged PR and a confident team. The eventual symptom is the same production
variance as Trap 1, arriving six months later on a new instance type, after an
upgrade to a JDK whose C2 makes a different inlining decision — at which point nobody
looks at the lock-free path, because "that one has a jcstress test."

**Root cause:** treating a stochastic search's failure to find a counterexample as a
universal claim. jcstress explored some interleavings under some compilation regimes on
one architecture. It cannot enumerate the space and does not claim to.

**Fix:** change what the artefact *is*. The test is not the proof. The proof is the
happens-before argument, and it belongs **in the `desc` field of the `FORBIDDEN`
outcome**, where the next person reads it:

```java
@Outcome(id = "1, 0", expect = FORBIDDEN,
         desc = "Ruled out by the volatile write->read edge on `reserved`: the release "
              + "store orders W1 before W2, the acquire load orders R1 before R2, and "
              + "observing true from R1 synchronises-with W2.")
```

Now the test failing tells you the argument is wrong, and the test passing tells the
reader what the argument *was*. Also write, in the PR: the JDK version, `uname -m`,
and the jcstress mode you ran. A result without its conditions is not a result.

---

### Trap 3 — running only on x86 (or only on aarch64) and generalising

**Wrong:** CI runs on x86 GitHub runners. Production runs on AWS Graviton. jcstress
runs in CI only.

**Exact symptom, direction one:** a clean CI history, then a memory-visibility bug in
production that nobody can reproduce locally on their x86 laptop or in CI. Attempts to
reproduce burn a week. Someone eventually says "it only happens on the ARM nodes" and
is not believed, because that sounds like superstition.

**Exact symptom, direction two — the one you will hit personally:** you are on an
Apple Silicon Mac. You run a jcstress test locally, see an interesting outcome, raise
it, and a colleague on an x86 workstation cannot reproduce it and closes your ticket.

**Root cause:** x86/x86-64 implements Total Store Order. The hardware will not reorder
store-store or load-load. So a program with a genuine JMM-level defect can execute
correctly on x86 forever — not by luck, but structurally, because the hardware provides
ordering the language did not require. aarch64 is weakly ordered and provides no such
accident. The defect is present in both cases; only one architecture is willing to show
it to you.

**Fix:**

- Run jcstress on **both** architectures. On a Mac, an x86 container under emulation is
  not a substitute — emulation typically enforces stronger ordering than the emulated
  hardware would. Use a real x86 runner.
- Treat the **aarch64** result as the more informative one. A `FORBIDDEN` observation
  on aarch64 is a hard bug. A clean x86 run is weak evidence.
- Put an aarch64 runner in CI if production is aarch64. This is a five-line CI change
  and it is the single highest-leverage thing in this section.
- And the discipline: **never say a bug "will" reproduce on a given architecture.** Say
  "this architecture is capable of exposing it." You do not control the JIT's decisions,
  the scheduler, or whether the window was hit.

---

### Trap 4 — actors that do too much work

**Wrong:**

```java
@Actor
public void orderPlacement() {
    Map<String, Integer> skuIndex = buildSkuIndex();   // allocates, hashes
    log.debug("reserving {}", skuIndex.size());        // formats, maybe I/O
    reservedUnits = skuIndex.get("SKU-4471");
    reserved = true;
}
```

**Exact symptom:** the outcome table shows one row — the boring, fully-ordered one — at
100%, on every run and every mode. You conclude the code is fine. Alternatively, the
run takes twenty minutes per configuration and you stop running it.

**Root cause:** the racy window is a handful of instructions wide. jcstress finds
interesting interleavings because the two actors are executing *the same few
instructions at the same time*, thousands of times, and their relative drift eventually
lands them in the window. Put a `HashMap` construction and a log call in front, and
each actor spends microseconds in unrelated work; the odds of both threads being inside
the four-instruction window simultaneously collapse. You have also introduced
synchronisation you did not intend — logging frameworks and `HashMap` resizing both
contain barriers and locks, which can order the very thing you were trying to leave
unordered.

**Fix:** the actor body is *only* the statements under test. Everything else moves into
the `@State` object's fields and constructor, which jcstress runs before the actors
start:

```java
@State
public class ReservationPublicationPlain {
    // Pre-computed in the state, not in the actor.
    static final int UNITS = 5;

    int     reservedUnits;
    boolean reserved;

    @Actor
    public void orderPlacement() {
        reservedUnits = UNITS;   // two statements. That is the whole actor.
        reserved = true;
    }
}
```

The skill here is **extraction**: reducing a 200-line service method to the two field
accesses whose ordering is actually in question. That extraction is the engineering.
The tool is easy.

---

### Trap 5 — mutable state outside the `@State` object

**Wrong:**

```java
@JCStressTest
@State
public class BadlyScopedTest {

    static int sharedCounter;        // STATIC. Shared across ALL state instances.

    @Actor public void a() { sharedCounter++; }
    @Actor public void b() { sharedCounter++; }
    @Arbiter public void arb(I_Result r) { r.r1 = sharedCounter; }
}
```

**Exact symptom:** the outcome table is full of `UNKNOWN` rows with large values —
`847`, `1203`, whatever the counter happened to reach — and the test fails immediately.
Or, if you were unlucky enough to declare a wide regex `id`, the test *passes* while
measuring nothing.

**Root cause:** jcstress allocates thousands of `@State` instances and runs many
iterations. A `static` field is shared across every one of them, so the counter
accumulates across the entire run instead of being a fresh two-thread experiment each
time. You are no longer measuring an interleaving; you are measuring the total
iteration count, badly.

**Fix:** every mutable field involved in the race is an **instance field of the `@State`
class**. `static final` constants are fine. `static` mutable state is never fine.

If you genuinely need shared state across iterations — you almost never do — jcstress
supports separating the `@State` object from the test class, so the state has an
explicit lifecycle. Read the jcstress samples in the archetype-generated project before
reaching for that; there is a `samples` package in the generated sources, and it is the
best documentation the tool has.

---

## Hands-on proof

Everything below is a command **you** run. I have no JVM, so I will not print output
and call it real. What I can give you precisely is what to run, what to look for, and
what each possible result means.

### Setup and machine hygiene

```bash
cd ~/java-lab/99/orderflow-jcstress

java --version
uname -m
sysctl -n hw.ncpu                 # macOS: logical core count
sysctl -n machdep.cpu.brand_string 2>/dev/null || echo "(Apple Silicon: see About This Mac)"
```

**Record all four.** Then quiet the machine:

- Close the browser. Close Docker Desktop. Close IntelliJ if you can.
- Plug in the laptop. On battery, macOS will park cores and change frequency scaling,
  which changes the interleavings you can reach.
- Do not run jcstress and expect to use the machine for anything else. It will saturate
  every core by design.

On Apple Silicon there is an extra wrinkle worth knowing: the cores are **asymmetric**
(performance cores and efficiency cores). Two actor threads scheduled onto a P-core and
an E-core run at very different rates, which changes the drift pattern and therefore
which windows you reach. This is not a problem — it is arguably an advantage, because
it explores relative-progress ratios that a symmetric machine does not. But it does
mean your frequencies will not match a colleague's, and you should not expect them to.

### Proof 1 — see the generated runner

```bash
mvn clean verify
find target/generated-sources -name '*PlainCounterIncrement*'
```

Open one of the generated files and read it. Look for:

- The **array of state objects** being allocated.
- The **padding** fields, usually with names that make their purpose obvious.
- The **tight actor loop** over a slice of the array.
- The **barrier** the threads are held at before release.

**How to read it:** you are confirming, from source you did not write, that the harness
does what this document claims: many states, padded, two threads racing over the same
array, tallied afterwards. Ten minutes here removes the tool's mystique permanently.

### Proof 2 — the interpreter reorders nothing

Run the plain publication test in each compilation regime and compare. jcstress does
this for you across its forks, but doing it by hand once makes the mechanism concrete:

```bash
java -jar target/jcstress.jar -t ReservationPublicationPlain \
     -jvmArgs "-Xint"

java -jar target/jcstress.jar -t ReservationPublicationPlain \
     -jvmArgs "-XX:TieredStopAtLevel=1"

java -jar target/jcstress.jar -t ReservationPublicationPlain
```

> `-jvmArgs` is the flag on the versions I know; confirm with `-h` on yours.

**What to look for:** whether the `1, 0` row appears in each of the three, and at what
frequency.

**How to read it:**

| What you see | What it means |
|---|---|
| `1, 0` absent under `-Xint`, present under normal execution | The clean signal. The interpreter does not reorder; C2 does. You have just seen the compiler's reordering licence in action. |
| `1, 0` present under `-Xint` too | Then the hardware produced it, not the compiler. Much more likely on aarch64 than x86. Either way, a real observation. |
| `1, 0` absent in all three | Not observed on this run. Says nothing about legality. See Honesty rule 1, and try `-m stress`. |
| Every mode looks identical and boring | Check the actors are actually as small as Trap 4 requires, and that you built with `mvn verify` rather than `mvn compile`. |

### Proof 3 — the `UNKNOWN` guard rail

Deliberately delete one `@Outcome` line from the plain test — say, `"0, 5"` — rebuild
and re-run.

**What to look for:** an `UNKNOWN` row in the table and a failed test.

**How to read it:** this is the property that makes jcstress trustworthy. A test
framework that let an unclassified outcome pass would let you ship a green test that
observed the bug and shrugged. jcstress forces you to have thought about every outcome
it saw. Put the line back.

### Proof 4 — the mode ladder

```bash
java -jar target/jcstress.jar -t ReservationPublicationPlain -m quick
java -jar target/jcstress.jar -t ReservationPublicationPlain -m default
java -jar target/jcstress.jar -t ReservationPublicationPlain -m stress
```

> Mode names vary by version. `-h` lists the ones your build accepts.

**What to look for:** the change in `SAMPLES` and whether rare outcomes appear at
higher modes that were absent at `quick`.

**How to read it:** if a rare outcome appears only at `-m stress`, that is a direct,
personal demonstration of Honesty rule 1 — you had "proof" of absence at `quick` and it
was wrong. That is a lesson worth the wall-clock time. Keep the result and the
timestamps.

### Proof 5 — the HTML report

```bash
java -jar target/jcstress.jar -t ReservationPublication -r results/
open results/index.html
```

**What to look for:** the per-configuration breakdown. The report shows each test
across each JVM configuration separately, rather than the console's summary.

**How to read it:** a test that is clean in three configurations and dirty in the fourth
is telling you *which layer* introduced the problem. That is diagnostic information the
console summary flattens away.

---

## Failure drill

**Mandatory.** Do not read the analysis until you have produced both halves yourself
and written down what you saw. The point is not the knowledge. The point is that you
personally watched a 100-thread loop test lie to you.

### The scenario

You will test the *same* invariant twice: once with a conventional multi-threaded JUnit
test, and once with jcstress. Then you will compare what each was capable of telling
you.

The invariant: **a fulfilment worker must never observe `reserved == true` with
`reservedUnits == 0`.**

### Part A — the test that lies

`src/test/java/com/orderflow/ReservationLoopTest.java` (in a normal Maven project, not
the jcstress one):

```java
package com.orderflow;

import org.junit.jupiter.api.RepeatedTest;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;
import static org.junit.jupiter.api.Assertions.assertEquals;

class ReservationLoopTest {

    static final class Reservation {
        int     reservedUnits;   // plain
        boolean reserved;        // plain

        void reserve(int units) {
            reservedUnits = units;
            reserved = true;
        }

        int observe() {
            if (reserved) return reservedUnits;
            return -1;
        }
    }

    @RepeatedTest(20)
    void fulfilmentNeverSeesAReservedZero() throws Exception {
        final int rounds = 100_000;
        AtomicInteger oversells = new AtomicInteger();

        ExecutorService pool = Executors.newFixedThreadPool(2);
        for (int i = 0; i < rounds; i++) {
            Reservation r = new Reservation();
            CountDownLatch start = new CountDownLatch(1);
            CountDownLatch done  = new CountDownLatch(2);

            pool.submit(() -> {
                await(start);
                r.reserve(5);
                done.countDown();
            });
            pool.submit(() -> {
                await(start);
                if (r.observe() == 0) oversells.incrementAndGet();
                done.countDown();
            });

            start.countDown();
            done.await();
        }
        pool.shutdown();
        assertEquals(0, oversells.get(),
            "observed a reservation with the flag set and zero units");
    }

    private static void await(CountDownLatch l) {
        try { l.await(); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
    }
}
```

Run it:

```bash
mvn test -Dtest=ReservationLoopTest
```

**Record, before you go on:**

1. Did it pass? (Write down pass or fail, and how many of the 20 repetitions.)
2. How long did it take?
3. `java --version` and `uname -m`.

Note what this test does *right*, because it is better than most: it uses a start latch
so the two threads are released together rather than in a staircase, and it creates a
fresh object per round so there is no accidental happens-before carried over. It is a
conscientious version of the wrong approach.

Also notice the thing it cannot do, no matter how conscientious: **the `CountDownLatch`
itself creates happens-before edges.** `start.countDown()` and `start.await()` are a
synchronisation point. The very mechanism used to make the threads start together
partially orders the memory they are about to race on. You are fighting the tool.

### Part B — the same invariant in jcstress

Use `ReservationPublicationPlain` from Example 2 above, verbatim.

```bash
cd ~/java-lab/99/orderflow-jcstress
mvn clean verify
java -jar target/jcstress.jar -t ReservationPublicationPlain -m default
```

**Record:**

4. Which rows appeared in the table.
5. The `SAMPLES` column for each row — the actual numbers from *your* run.
6. Whether `1, 0` appeared, and in which configurations.
7. Wall-clock time.

### Part C — the architecture dimension

If you have access to an x86 machine or an x86 CI runner, run Part B there too and
record the same four items.

If you have only your Mac, run it on your Mac and **write the honest note**: "aarch64
only; the x86 result is unknown." That note is the correct output of the exercise when
you lack the hardware. Do not guess at the other platform's numbers.

### How to read what you got

| What you see | What it means |
|---|---|
| Part A passes, Part B shows `1, 0` | **The drill has fired.** You have a test that says "thread-safe" and a tool that produced a counterexample to the same claim on the same machine in the same session. Write down how long each took. |
| Part A passes, Part B does not show `1, 0` either | Not observed today. The code is still wrong — the JMM permits the outcome and you have no edge. Escalate to `-m stress`, quiet the machine further, and re-read Honesty rule 1. A clean run is not a defence. |
| Part A **fails** | Unusual and excellent. Your machine's scheduling happened to hit the window with the latch's help. Record it: you have a loop test that caught a real bug, which is a good day, and it changes nothing about the argument — the loop test is still not a proof of the negative. |
| Part B shows `1, 0` only in the C2 configuration | The compiler introduced the reordering. Confirm with the `-Xint` run from Proof 2. |
| Part B shows `1, 0` in `-Xint` too | The hardware did it. Note your architecture next to that result; on aarch64 this is entirely expected, on x86 it is worth a second look at your test. |

### Now fix it and re-run

Swap in `ReservationPublicationVolatile`, with `1, 0` declared `FORBIDDEN`, and run it
in the same session with the same flags.

```bash
java -jar target/jcstress.jar -t ReservationPublication -m default
```

**What the fix proves:** nothing, by itself — and that sentence is the entire lesson of
this topic.

What you now have is:

1. **An argument** (the volatile release/acquire edge) that rules the outcome out.
2. **A falsification attempt** against that argument, which failed to falsify it, under
   named conditions.
3. **A control**: the same experiment against the unfixed code, in the same session, on
   the same machine, which produced the outcome (or did not — record which).

That triple is what "verified" means in concurrency. It is strictly weaker than a proof
and enormously stronger than a green loop test. Being able to state that distinction
precisely, without hedging into mush and without overclaiming, is the thing this topic
exists to teach you.

---

## Measurement

### The standing rule, restated

A naive `System.nanoTime()` loop is the **wrong** way to measure anything on the JVM.
It measures JIT warm-up, dead-code elimination of your unused result, on-stack
replacement, and whatever the machine was doing at the time. Topic 77 (JMH) is where
you learn to do it properly. That rule applies to every doc in this phase and this one
is no exception.

### But: jcstress is not a benchmark, so do not measure it as one

This deserves a paragraph of its own, because the mistake is easy.

The `FREQ` column looks like a performance metric. It is not. It is *the fraction of
observations that produced this outcome*. A frequency of 0.001% for the interesting
outcome does not mean the bug is rare in production; it means the bug was rare **under
jcstress's specific stress pattern on this machine**. Production has a different
pattern, different cores, different cache pressure and different load. The frequency
does not transfer.

Two rules:

- **Never** put a jcstress `FREQ` into a risk assessment as a probability.
- **Never** compare `FREQ` between two implementations and call one "faster". If you
  want throughput numbers for a lock-free versus locked implementation, that is JMH
  (Topic 77), with `@Threads`, and it is a different tool for a different question.

### What is worth measuring about a jcstress run

| Thing | How | Why |
|---|---|---|
| Wall-clock time per mode | `time java -jar target/jcstress.jar -t X -m quick` etc. | You need to know what a full run costs before you propose putting one in CI. |
| Which configurations were run | `-v`, and the per-config sections in the HTML report | A test that only ran in one config is a test you over-read. |
| Whether the interesting outcome appears at all in `quick` | Compare `quick` and `stress` | Tells you the minimum mode that is honest for this test. |
| CPU saturation during the run | Activity Monitor, or `top -l 1 -s 0` | If jcstress is not saturating your cores, something is throttling it and your results are weaker. |

### The things you must record with every result

Make this a checklist and paste it into the PR. A result without these is not a result.

```
jcstress result — <test class name>
  jcstress version : <from pom.xml>
  JDK              : <java --version, full string>
  Architecture     : <uname -m>            # arm64 or x86_64 — this matters
  CPU              : <brand string / core count, note P/E asymmetry if Apple Silicon>
  Mode             : <-m value>
  Configurations   : <how many forks/JIT modes ran>
  Wall clock       : <duration>
  Machine state    : <what else was running; "quiesced" if nothing>
  Outcomes         : <the table, copied verbatim>
  Claim            : "<outcome> was NOT OBSERVED under these conditions."
                     NOT "<outcome> is impossible."
```

That last two lines are the discipline. Write the claim in those exact words until it
becomes automatic.

### Connecting to the Topic 65 baseline

jcstress does not touch the `orderflow` baseline and should not. It runs a synthetic
two-thread shape, not your service. There is no p99 here.

The connection runs the other way: **when the Topic 65 load test surfaces a correctness
anomaly** — a reservation quantity that does not match the order, a wallet balance that
does not reconcile — jcstress is how you turn that anomaly into a reproducible,
minimised experiment. The workflow is:

1. Load test produces an anomaly at some rate.
2. You identify the two fields and the two code paths involved.
3. You extract those into a jcstress test — three or four lines per actor.
4. You write the happens-before argument, or discover you cannot.
5. You fix, and re-run both the jcstress test and the Topic 65 baseline.

Step 3 is the skill. Steps 1, 2, 4 and 5 are process.

### Should this be in CI?

Honest answer: **usually not in the per-commit pipeline.** A meaningful jcstress run
takes minutes per test and saturates the machine, which makes it a bad neighbour on a
shared runner and a bad fit for a pre-merge gate.

What works in practice:

- A **nightly** job running the full jcstress suite, on the architecture that matches
  production, failing loudly on any `FORBIDDEN` or `UNKNOWN`.
- A **pre-merge** run scoped to the tests touching the changed critical section, at
  `-m quick`, understood by everyone as a smoke test rather than a gate.
- The tests **committed to the repository regardless**, because their real value is
  that the `@Outcome` descriptions carry the happens-before argument to the next reader.

That last point is underrated. Six months from now, someone will look at
`ReservationPublicationVolatile` and read, in the `FORBIDDEN` description, exactly why
the `volatile` is load-bearing. That is worth more than the run.

---

## Practice exercises

Write real files, run them, and record the platform and JDK alongside every result.

### 1 — easy

Build the archetype project and write **two** tests:

**(a)** `PlainCounterIncrement` from Example 1, verbatim. Run it and paste the table.

**(b)** The same test with `units` declared `volatile`. Before you run it, write down
your prediction: does `volatile` prevent the lost update? Then run it.

Requirements:

- Paste both outcome tables, with `java --version` and `uname -m` above each.
- Answer in one paragraph: why does `volatile` not fix this? Name what `volatile`
  guarantees and what it does not, and connect it to Topic 87.
- Then make it correct a third way, with `AtomicInteger.incrementAndGet()` (Topic 95),
  declare the lost-update outcome `FORBIDDEN`, and run it. State the argument for the
  `FORBIDDEN` in the `desc` field.

### 2 — medium (combines Topics 01–98)

Take three bugs you already met earlier in the curriculum and encode each as a jcstress
test. For each one: write the happens-before argument first, then the test, then run it,
then record the table.

**(a) Topic 87 — `volatile i++` loses an increment.** Two actors, one `volatile int`.
Grade the lost update `ACCEPTABLE_INTERESTING` and explain in the `desc` why it is legal
despite the `volatile`.

**(b) Topic 88 — unsafe publication of a partially-constructed object.** One actor
writes `order = new Order(sku, 5)` to a plain field where `Order`'s fields are **not**
final; the other actor reads the reference and, if non-null, reads `order.quantity()`.
Declare the "non-null reference, default-valued field" outcome. Then run the *same*
test with `Order`'s fields made `final` and the bad outcome declared `FORBIDDEN`, and
write the freeze-action argument.

**(c) Topic 92 — check-then-act on a `ConcurrentHashMap` idempotency cache.** Two
actors each do `if (!cache.containsKey(key)) { cache.put(key, ORDER); charged++; }`.
The result is the number of charges. Declare `2` — the double charge — with a
description that says, in business terms, that the customer's wallet was debited twice.
Then fix it with `putIfAbsent` and declare `2` `FORBIDDEN`.

Deliverables:

- Three test classes, three prose happens-before arguments, three outcome tables.
- One paragraph per test: what the loop-test version of this would have told you, and
  why it would have been less informative.
- For (c) specifically: is `2` genuinely `FORBIDDEN` after the fix, or merely
  `ACCEPTABLE` and not observed? Justify. (This is the sharpest question in the
  exercise; think about what `putIfAbsent`'s atomicity actually guarantees and what it
  does not.)

### 3 — hard

The full workflow, on the `orderflow` spine.

**Part A — extract.** Take the real inventory-decrement critical section from your
`orderflow` service — the one from Topic 52. Reduce it to the smallest pair of actors
that preserves the invariant under test. Write down what you removed and, for each
removal, one sentence on why removing it does not change the memory-ordering question.
This extraction step is the actual engineering; the rest is mechanics.

**Part B — argue.** Before writing a line of test code, write the happens-before
argument for the current implementation in prose. Name the edge. If your implementation
is `synchronized`, name the monitor and both the release and the acquire. If it is a
conditional atomic UPDATE at the database level, say honestly that the JMM argument does
not apply because the invariant is enforced outside the JVM — and then say what, if
anything, jcstress can usefully test about the Java side.

**Part C — falsify.** Encode it. Declare the oversell outcome `FORBIDDEN` with your Part
B argument in the `desc`. Run it at `-m default` and `-m stress`. Record everything on
the Measurement checklist.

**Part D — break it three ways.** Produce three broken variants and predict, *before
running each*, whether jcstress will observe the bad outcome:

1. Remove the `synchronized` from the **reader** only, keeping it on the writer.
2. Replace `synchronized` with a `volatile` flag but leave the two-field invariant
   spanning the flag and the quantity.
3. Keep both `synchronized` blocks but move one field's write outside the block.

For each: your prediction, then the actual table, then one sentence on why you were
right or wrong. **Being wrong here is the most valuable outcome in the exercise** —
write it up rather than quietly correcting it.

**Part E — the honest write-up.** Produce the PR description you would actually post.
It must contain: the argument, the conditions, the observed table, the control, and the
sentence "this is a failure to falsify, not a proof." Then add one paragraph on what
you would need to be *confident* rather than merely unfalsified — and be honest that
for most real code the answer is "a simpler design", not "a longer jcstress run".

---

## Interview questions

### Q1 — "You tested it with 100 threads in a loop and it passed. What does that prove?"

This is the staple. It is asked to find out whether you understand the epistemics or
just the API.

**Mid-level answer:** "It shows the code works under concurrent load. It's not exhaustive
— there could still be edge cases we didn't hit."

**Senior answer:** "It proves the code did not fail on that machine, in that JIT state,
with that thread-arrival pattern, on that architecture, on that run. That's much weaker
than it sounds, for five specific reasons. The threads don't converge — starting 100
threads in a loop is a staircase, and the racy window is a few instructions wide, so
they mostly miss it. The JIT state is unknown: the loop may have run interpreted, or in
C1, or been compiled to C2 halfway through, and only C2 does the aggressive reordering.
The architecture matters — if that machine was x86, Total Store Order means whole classes
of store-store and load-load reordering were structurally unreachable, so the test could
not have found them even in principle. The machine was noisy. And there's luck, which is
a real technical factor and not a joke. What I'd actually do is write the happens-before
argument — name the edge — and then use jcstress to try to falsify it, which is the only
tool that deliberately hunts interleavings and grades outcome frequencies. And even then
a clean jcstress run is a failure to falsify, not a proof."

**What separates them:** the mid-level answer knows the test is incomplete. The senior
answer knows *which specific mechanisms* make it incomplete, names TSO by name, and —
critically — does not then overclaim for jcstress either. The trap in this question is
that many candidates answer it by proposing jcstress as the thing that *would* prove it.
That is the same mistake with a better tool.

**Follow-up:** "So what would prove it?" The answer is a happens-before argument, plus
the observation that most real code is easier to make obviously correct than to prove
correct — which is an argument for design simplicity, and is the answer they are hoping
for.

---

### Q2 — "Your jcstress test declares an outcome FORBIDDEN and it never appeared. What can you conclude?"

**Mid-level answer:** "That the outcome can't happen — the synchronisation is working."

**Senior answer:** "That it was not observed on that machine, that JDK, that
architecture, that run. Nothing stronger. jcstress samples a space it cannot enumerate:
every interleaving, every compiler reordering, every hardware reordering, every cache
state. A `FORBIDDEN` that *does* appear is conclusive — I have a counterexample and my
argument is wrong. A `FORBIDDEN` that doesn't appear is a failure to falsify, which is
much weaker. The proof is the happens-before argument, which is why I put it in the
outcome's description field: the test failing tells me the argument is wrong, and the
test passing tells the next reader what the argument was."

**What separates them:** the asymmetry between falsification and verification, stated
cleanly, plus the practical move of putting the argument in the `desc`. That second part
signals someone who has actually maintained these tests rather than written one.

**Follow-up:** "Then why run it at all?" Because being unable to falsify a claim you
seriously tried to falsify is real evidence, and because the control case — running the
unfixed version in the same session — is what makes the result interpretable.

---

### Q3 — "Why does jcstress fork multiple JVMs?"

**Mid-level answer:** "To get more runs and better statistics."

**Senior answer:** "Because a concurrency bug can be introduced at four layers and they
need to be separable. The interpreter reorders nothing. C1 does modest reordering. C2
hoists loads out of loops, sinks stores, and eliminates redundant loads — the aggressive
stuff. And the CPU does its own reordering on top. Forking a fresh JVM per compilation
configuration is how you find out which layer produced the outcome, which is diagnostic
information you cannot get from one long run. It's also why the Topic 86 stop-flag drill
terminates under `-Xint` and hangs normally: that gap is C2's hoisting, made visible.
There's a second reason too — a fresh JVM has an unpolluted profile, so earlier tests
don't shape the JIT's inlining decisions for later ones."

**What separates them:** naming the four layers, connecting it to a bug they have
personally reproduced, and knowing about profile pollution.

**Follow-up:** "How would you use that to debug a real report?" Run `-Xint` first: if the
bug survives interpretation, it is the hardware or a genuine logic race, not the
compiler.

---

### Q4 — "Your CI runs on x86 and production runs on Graviton. Does that matter for concurrency testing?"

This one is increasingly common and it is a good discriminator.

**Mid-level answer:** "It might, if there's anything architecture-specific. We'd want to
test on the same architecture ideally."

**Senior answer:** "Yes, and specifically in one direction. x86 implements Total Store
Order: the hardware won't reorder store-store or load-load, only store-then-load. So a
program with a genuine memory-model defect can run correctly on x86 indefinitely — not
by luck, but structurally, because the hardware supplies ordering the language never
required. aarch64 is weakly ordered and gives you no such accident. So a clean x86 CI run
is structurally weaker evidence than a clean aarch64 run, and the gap is exactly the
wrong way round for us: our least revealing platform is gating our most revealing one.
I'd put an aarch64 runner in the nightly jcstress job — it's a small CI change and it's
the highest-leverage thing available. I'd also be careful not to overclaim: I wouldn't
tell anyone the bug *will* reproduce on ARM, because I don't control the JIT's decisions
or whether the window gets hit. The right phrasing is that the platform is capable of
exposing it."

**What separates them:** naming TSO, stating which direction the risk runs, proposing a
concrete cheap fix, and refusing to overclaim about reproduction. That last refusal is
what a strong interviewer is listening for.

**Follow-up:** "Can you emulate x86 on ARM to test?" Emulation usually enforces stronger
ordering than the emulated hardware would, so it is not a substitute. Use real hardware.

---

### Q5 — "When is jcstress the wrong tool?"

**Mid-level answer:** "When you're not doing concurrency, or for performance testing."

**Senior answer:** "Several cases. First, when the critical section is large — jcstress
finds interleavings because two threads execute the same few instructions
simultaneously, thousands of times. If the actor does a map lookup and a log call, the
threads never overlap in the window and every run reports the boring outcome, which
looks like a pass. Second, when the failure mode isn't a value — a livelock or a
starvation problem isn't expressible as a result tuple, though `Mode.Termination`
handles the hang case. Third, for performance: `FREQ` is an outcome frequency, not a
throughput number, and reading it as one is a category error — that's JMH with
`@Threads`. Fourth, and most importantly: when the honest fix is a simpler design.
If I'm writing a jcstress test to defend a clever lock-free structure in application
code, the real question is why that structure exists rather than a `ConcurrentHashMap`
or a database constraint. jcstress is the right tool for the JDK's own internals and for
the small number of genuinely hot, genuinely custom critical sections in an application.
It's the wrong tool for justifying complexity."

**What separates them:** the last point. Anyone can list technical limitations. Treating
"should this code exist" as part of the tool-selection question is a judgment answer, and
judgment is what an ELITE-level question is probing.

**Follow-up:** "Give me an example from your own codebase." Have one ready. For
`orderflow` the honest answer is that inventory decrement should be a single conditional
`UPDATE` in the database (Topic 52), which makes the entire JMM question disappear, and
jcstress is then only relevant to the in-memory caching layer in front of it.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. jcstress refuses to pass a test that observed an outcome you did not declare. Argue
   the case *against* that design: what would be gained by a mode that let unclassified
   outcomes through, and why do you think the authors rejected it?

2. The `@Arbiter` runs with happens-before edges from every actor, so it always sees a
   settled state. Given that, when is an arbiter-based test the right shape and when is
   an actor-writes-the-result test the right shape? State the rule, not an example.

3. You have a `FORBIDDEN` outcome that appeared exactly once in a run of many millions.
   Your colleague says "that's a fluke, re-run it." Construct the strongest version of
   their argument, then demolish it.

4. Suppose the JIT were changed so that it never reordered anything, ever. Would jcstress
   still be able to find memory-visibility bugs? Under what conditions, on which
   architectures, and what would that tell you about where the bug "lives"?

5. `FREQ` is not a probability of the bug occurring in production. Explain why, in terms
   of what jcstress does to the machine, and then explain what — if anything — you *can*
   legitimately infer from a very high versus a very low frequency.

6. You extracted a 200-line service method down to a four-line actor pair. What class of
   error does that extraction introduce, and how would you catch it? (Hint: the test can
   be perfectly correct about a shape that is not the shape your service has.)

7. Argue this position and then argue against yourself: "if a piece of application code
   needs a jcstress test to be trusted, that code should be deleted and replaced with
   something obviously correct, even at a throughput cost."

---

## Quick reference card

### The two sentences

- **A `FORBIDDEN` with zero samples means:** "not observed on this machine, this JDK,
  this run." It does **not** mean "impossible".
- **x86 is Total Store Order and hides store-store and load-load reordering bugs that
  aarch64 exposes.** Record `uname -m` with every result. Never say a bug *will*
  reproduce.

### Annotations

```java
@JCStressTest                     // marks the test; default Mode.Continuous
@JCStressTest(Mode.Termination)   // for "does the loop ever stop" tests
@State                            // the object holding the raced fields
@Actor                            // one method, one thread, once per state instance
@Arbiter                          // runs after all actors, with HB edges from them
@Signal                           // Termination mode: tries to stop the spinning actor
@Outcome(id = "...", expect = ..., desc = "...")
@Description("...")
```

### Grades

| Grade | Use it when |
|---|---|
| `ACCEPTABLE` | Legal and boring. |
| `ACCEPTABLE_INTERESTING` | Legal, surprising, and the thing you are hunting. Use this on the *broken* version. |
| `FORBIDDEN` | Your happens-before argument rules it out. Use this on the *fixed* version, and put the argument in `desc`. |
| `UNKNOWN` | You never write this. jcstress assigns it to anything you failed to declare, and fails the test. |

### Result carriers

```
I_Result     one int          fields: r1
II_Result    two ints                 r1, r2
III_Result   three ints               r1, r2, r3
Z_Result     one boolean              r1
ZZ_Result    two booleans             r1, r2
J_Result     one long                 r1
L_Result     one reference            r1
LL_Result    two references           r1, r2
```

Outcome `id` = fields joined by `", "` in order. `"1, 0"` is not `"0, 1"`.

### Commands

```bash
mvn archetype:generate -DarchetypeGroupId=org.openjdk.jcstress \
    -DarchetypeArtifactId=jcstress-java-test-archetype ...   # pin the CURRENT version
mvn clean verify                                  # must be verify; builds target/jcstress.jar
java -jar target/jcstress.jar -h                  # do this FIRST; flags vary by version
java -jar target/jcstress.jar -l                  # list discovered tests
java -jar target/jcstress.jar -t <regex>          # run matching tests
java -jar target/jcstress.jar -t X -m quick       # short run
java -jar target/jcstress.jar -t X -m stress      # long run; rare outcomes surface here
java -jar target/jcstress.jar -t X -v             # per-configuration detail
java -jar target/jcstress.jar -t X -r results/    # HTML report location
java -jar target/jcstress.jar -t X -jvmArgs "-Xint"                    # interpreter only
java -jar target/jcstress.jar -t X -jvmArgs "-XX:TieredStopAtLevel=1"  # C1 only
```

### Reading the table

```
RESULT      SAMPLES     FREQ       EXPECT  DESCRIPTION
<tuple>         <n>     xxx%   ACCEPTABLE  <your desc>
```

*illustration of the format, not captured output*

`FREQ` is an outcome frequency. It is **not** throughput and **not** a production
probability.

### Record with every result

```
jcstress version | JDK version | uname -m | CPU + core count | mode | forks
wall clock | machine state | the table | "NOT OBSERVED under these conditions"
```

### Gotchas checklist

- [ ] Actors contain only the statements under test. No logging, no map lookups.
- [ ] All mutable fields are instance fields of the `@State` object. No `static` mutables.
- [ ] `@Outcome` ids are in `r1, r2` order, with a comma and a space.
- [ ] `FORBIDDEN` on the fixed version, `ACCEPTABLE_INTERESTING` on the broken one.
- [ ] The happens-before argument lives in the `FORBIDDEN` outcome's `desc`.
- [ ] Built with `mvn clean verify`, not `mvn compile`.
- [ ] Machine quiesced, plugged in, browser closed.
- [ ] `uname -m` and `java --version` recorded next to the table.
- [ ] Never write "proves". Write "was not observed under these conditions".

---

## When would I use this at work?

**1. A production anomaly with no exception attached.**

Reconciliation shows a handful of orders per week whose reserved quantity does not
match the line total. No stack trace, no error metric, no deploy correlation. The
classic shape of a visibility bug. You identify the two fields and the two threads
involved, extract them into a four-line jcstress test, and either observe the bad
outcome or confirm you cannot construct a happens-before argument for the current code.
Either way you have moved from "we think something's racy" to a specific, minimised,
runnable artefact — which is the difference between a theory and a ticket.

**2. Reviewing a PR that introduces a lock-free fast path.**

Someone replaces a `synchronized` block with an `AtomicReference` and a CAS loop, citing
contention. Your review comment is not "are you sure?" — it is: "what happens-before edge
makes the reader's view consistent, and can you add a jcstress test that declares the
inconsistent view `FORBIDDEN` with that argument in the description?" That comment does
three things: it forces the argument to exist, it leaves the argument in the repository
for the next reader, and it puts the burden of evidence where it belongs. It is also a
much better review comment than a vague concern, because it is actionable.

**3. Before an architecture migration.**

Your service is moving from x86 to Graviton, or your team is adopting Apple Silicon
laptops while CI stays on x86. This is the moment to run the concurrency-sensitive parts
of the codebase under jcstress on **both** architectures, and to add an aarch64 nightly
runner. The point is not that you expect to find something; it is that the migration
changes which class of latent bugs the hardware is willing to reveal, and you would much
rather find out during the migration than three weeks after it.

---

## Connected topics

**Prerequisites — you cannot use this tool without these:**

- **84 — Threads vs the event loop:** preemption between any two bytecodes is the reason
  interleavings exist at all.
- **85 — `synchronized` and monitors:** the unlock→lock edge is the most common argument
  you will encode in a `FORBIDDEN` description.
- **86 — JMM I, happens-before:** the argument. jcstress without this is button-pushing.
- **87 — `volatile` and CPU barriers:** the release/acquire pair, and why `volatile i++`
  is still a race.
- **88 — `final` fields and safe publication:** the freeze action, and the record-based
  fix in Example 2.
- **92 — Concurrent collections:** check-then-act does not compose; exercise 2(c).
- **95 — CAS and atomics:** what you are usually testing when you write a jcstress test
  for application code.
- **96 — False sharing:** why jcstress pads its state and result objects.
- **98 — Bug taxonomy:** tells you whether jcstress is even the right tool for the
  symptom you have.

**Also relevant from earlier phases:**

- **64 — Property-based testing:** the honest partial analogue; same epistemics, a
  describable search space instead of an undescribable one.
- **73 — Safepoints:** why a thread can be paused in places you did not expect.
- **74 — JIT deoptimisation and profile pollution:** why jcstress forks a fresh JVM per
  configuration rather than running everything in one.
- **77 — JMH:** the tool for the performance question jcstress does not answer. If you
  want throughput, go there.

**This unlocks:**

- **100 — ForkJoinPool:** the work-stealing deque's correctness rests on exactly the kind
  of argument you now know how to falsify.
- **101 — Virtual threads:** unmounting and remounting a continuation across a blocking
  call is a memory-visibility question as well as a scheduling one. `Thread.currentThread()`
  identity holding across a mount is a design property, not an accident.
- **102 — Structured concurrency:** a structured scope's join point is a happens-before
  edge, and that is precisely what makes the fan-out results safe to read without extra
  synchronisation.
- **107 — Loom vs reactive:** both models still sit on the same memory model; neither
  makes visibility bugs impossible.

---

*Java baseline 21; nothing in this topic is version-sensitive at the language level.
jcstress itself versions independently of the JDK — pin the current release from
`github.com/openjdk/jcstress` rather than any version quoted in this document, and run
`-h` before trusting a flag from here. The two honesty rules — zero observations is not
a proof, and x86's Total Store Order hides what aarch64 exposes — do not change with
any version, and are the parts of this topic worth memorising verbatim.*
