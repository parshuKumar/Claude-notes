# 77 — JMH: Why Every Naive Benchmark Is Wrong

## Phase: 8 — JVM Internals
## Category: ELITE
## Java baseline: 21  |  Notes features from: 21 / 25
## Project spine: this is the measurement authority for the entire curriculum. Every "is A faster than B" claim made about `orderflow` — boxing in the pricing path (Topic 01), `ArrayList` versus `LinkedList` (Topic 11), string building in the reconciliation export (Topic 18), lambda allocation (Topic 21), `.parallel()` on the catalogue filter (Topic 25), `AtomicLong` versus `LongAdder` on the order counter (Topic 95) — is settled here or it is not settled at all. Topic 65 measures the whole system under load. This topic measures one method. Both are needed, and confusing them is its own failure mode.

---

## R0 — READ THIS BEFORE ANY OTHER LINE IN THIS DOCUMENT

**I do not have a JVM. I have never run JMH on your machine. Nothing in this document
is captured benchmark output, and I will never present anything as if it were.**

Specifically, and without exception, you will not find in this document:

- a JMH results table with numbers in it,
- any `ns/op`, `us/op`, `ms/op`, `ops/s` or `B/op` figure presented as a measurement,
- a throughput figure, a latency figure, or an "X is 3.4× faster than Y" claim,
- an error bar, a standard deviation, or a confidence interval with digits in it.

**Why this rule is stricter here than anywhere else in the curriculum:** roughly fifteen
other documents point at this one and say "settle it with JMH". If I invent a plausible
number here, I do not merely mislead you once — I put a fabricated constant into the
foundation of every performance argument you will make for the next year. A wrong
number is worse than no number, because a wrong number ends the investigation.

What you get instead, everywhere:

- the **exact command with exact flags**,
- **WHAT TO LOOK FOR** in your own output,
- a **"what you see" → "what it means"** table that covers the plausible outcomes *and*
  the surprising ones, because the surprising outcome is the one that teaches you
  something.

### The one labelled exception, and its limits

To read a results table you have to know what the columns are. So in exactly two places
I show the **column structure** of JMH output — the field names and their order — with
every value replaced by `<n>`, `<unit>` or `xxx`. Each such block carries the inline
label:

> *illustration of the format, not captured output*

Placeholders only. Never a plausible-looking number. If you find a digit in one of those
blocks that is not part of a field name, it is a bug in this document and you should
distrust it.

### The three things I state as fact

These are spec-level or documented-behaviour facts, and each comes with a command that
confirms it on your machine rather than asking you to trust me:

1. **JMH forks a fresh JVM process for each trial.** Confirm: run any benchmark and watch
   the run log print `# Fork: 1 of N` and re-print the full JVM banner each time; confirm
   independently with `jps -l` during a run, or `ps` for a second `java` process.
2. **JMH generates source code at build time.** Confirm: build the project and then
   `find target/generated-sources -name '*_jmhTest.java'` and read the file. Your
   benchmark method is not what runs; the generated harness around it is.
3. **`Blackhole` exists to defeat dead-code elimination, and `@State` exists to defeat
   constant folding.** Confirm: run the drill in this document, which produces both
   failures deliberately and then removes them one at a time.

### THE RULE, and it is absolute

> **If your JMH output disagrees with anything in this document, YOUR OUTPUT IS THE
> TRUTH.**
>
> Not mine. Not a conference talk's. Not a blog post from 2016 that benchmarked on
> JDK 8 with two forks and a laptop on battery power. The entire point of learning JMH
> is that you stop needing to believe anyone — *including me* — about which of two
> implementations is faster on the hardware you actually deploy to.

---

## Mechanical statement

Read this three times. Everything below is elaboration.

> **A `System.nanoTime()` loop does not measure your code. It measures dead-code
> elimination, constant folding, on-stack replacement and cold-JIT state, in unknown
> proportions.**
>
> Four separate machines are working against you at once:
>
> - **Dead-code elimination.** If nothing observes the result of your computation, C2 is
>   permitted to delete the computation. It frequently does. You then time an empty loop.
> - **Constant folding.** If your inputs are constants that C2 can see through — a
>   `static final`, a literal, a value it has proved never changes — C2 computes the
>   answer once, at compile time, and your "benchmark" returns a pre-computed constant.
> - **On-stack replacement (OSR).** A long-running loop gets compiled *while it is still
>   executing*, by a different path than a normal method compilation, producing code that
>   is often shaped differently from the code that runs in production. You measure the
>   OSR version. Production runs the other one.
> - **Cold JIT.** The first hundreds or thousands of iterations run interpreted, then
>   under C1, then under C2. Averaging across those three regimes produces a number that
>   describes none of them.
>
> **And a fifth, which only appears when you compare two things:** **profile pollution.**
> C2 optimises a call site against the profile it has observed. Run implementation A a
> million times and then implementation B in the same JVM, and B is compiled against a
> profile that A wrote. Reorder the two cases and the answer changes. That is not noise;
> it is a different compilation.
>
> **JMH is the answer to all five, and it answers each one with a specific mechanism:**
>
> - It **forks a fresh JVM per trial** (`@Fork`), so no profile, no code cache and no
>   heap state carries between cases. Multiple forks expose between-JVM variance, which
>   is real and which a single fork hides completely.
> - It **warms up** (`@Warmup`) until the compiler has settled, and discards those
>   iterations rather than averaging them in.
> - It **consumes every result through a `Blackhole`**, an object the compiler cannot see
>   through, so the work cannot be proved dead.
> - It **holds inputs in `@State` objects** read through fields, so they are not JIT-time
>   constants and cannot be folded.
> - It **generates the measurement loop itself**, so the loop is not something you can
>   accidentally get wrong, and it reports **statistics with error bars**, not a single
>   number.
>
> **Therefore: the output of a correct JMH run is a distribution, not a number. And the
> only honest way to state a JMH result is with its error.**

The one-line version, which you should be able to say without preparation:

*"Three nanoseconds is roughly the cost of nothing, which is exactly what a dead-code-
eliminated loop measures."*

---

## The bridge from what you know

### `console.time` is your habit, and it is ACTIVELY WRONG in Java

Let me be blunt, because softening this does you no favours.

This is what you do today, and it is fine in Node for the thing you use it for:

```ts
// orderTotals.ts — your normal reflex
console.time('total');
for (let i = 0; i < 1_000_000; i++) {
  computeOrderTotal(lines);
}
console.timeEnd('total');
```

The direct Java transliteration is this, and **you must not write it**:

```java
// DO NOT DO THIS. This is the anti-pattern this entire document exists to kill.
long start = System.nanoTime();
for (int i = 0; i < 1_000_000; i++) {
    computeOrderTotal(lines);
}
long elapsed = System.nanoTime() - start;
System.out.println((elapsed / 1_000_000.0) + " ns/op");
```

It compiles. It runs. It prints a number. The number is meaningless, and — this is the
part that catches people — **it is meaningless in a direction that flatters whatever you
were hoping to prove**, because the more trivially-eliminable your code is, the faster it
appears.

Why the same code is *less* wrong in Node:

| | Node / V8 | JVM |
|---|---|---|
| Warm-up profile | Real, but shallower; Sparkplug/Maglev/TurboFan tiers exist and matter less for typical script-level code | Deep. Interpreter → C1 → C2, with profile-guided speculation and deoptimization (Topic 74) |
| Cross-case profile pollution | Present, and also under-appreciated | **Present and severe.** A monomorphic call site that becomes bimorphic is permanently slower (Topic 74) |
| Standard tool with process isolation | None in the standard toolchain; `benchmark.js` and `tinybench` do statistics but run in one process | **JMH, written by the people who write the JIT**, forks per trial by default |
| Dead-code elimination in practice | TurboFan does it; you have probably been bitten and not noticed | Aggressive, and the first thing that will ruin your benchmark |

### The PARTIAL analogue — and you have probably already been fooled by it

V8 also optimises away unobserved work. If you have ever written this:

```js
console.time('hash');
for (let i = 0; i < 5_000_000; i++) {
  hashSku('SKU-' + i);          // return value discarded
}
console.timeEnd('hash');
```

...and been pleased with the result, there is a real chance TurboFan noticed that
`hashSku` is pure and its result is unused, and removed some or all of the calls. You
have likely written misleading JavaScript benchmarks without ever finding out, because
nothing in the Node toolchain tells you.

**So the transferable insight is genuine: "the compiler removes work you do not
observe."** You already believe that, or you will in a minute. What does *not* transfer
is any sense of *how much* machinery is between your source and the executed code on the
JVM, and how many independent ways it can invalidate a timing loop.

**Verdict: PARTIAL ANALOGUE.** The instinct transfers. The magnitude does not, and the
tooling does not exist on the Node side, so you have no habit of process isolation to
carry over.

### The four Java lies, named

You will meet all four in the failure drill. Learn the names now, because naming the
failure is how you diagnose it in ten seconds instead of an afternoon.

**Lie 1 — dead-code elimination (DCE).**
Your loop computes something and throws it away. C2 proves the computation has no
observable effect and deletes it. Symptom: an impossibly small number, often below one
nanosecond per operation, and often *identical* across implementations that obviously
differ.

**Lie 2 — constant folding.**
Your inputs are `static final`, or literals, or values C2 has proved constant. The whole
call collapses to a constant at compile time. Symptom: two implementations that must
differ report the same number; changing the algorithm changes nothing.

**Lie 3 — on-stack replacement (OSR).**
Your benchmark loop runs for a long time inside one method invocation. The JVM cannot
wait for the method to be re-entered, so it compiles the loop *in place* and jumps into
the compiled version mid-flight. OSR-compiled code is real compiled code but is
generated under different constraints from a normally-compiled method — loop-invariant
state is already live in the frame, some optimisations are unavailable, and inlining
decisions can differ. Symptom: your benchmark's answer disagrees with production, and
the disagreement is stable and reproducible, which makes it very convincing and very
wrong.

**Lie 4 — cold JIT.**
Your loop starts interpreted. Somewhere in the first thousands of iterations it becomes
C1 code, then C2 code. Your average spans all three. Symptom: the number depends on the
iteration count. Double the iterations and the per-operation cost drops — which people
misread as "it gets faster when it's hot", concluding something true for a wrong reason,
and then reporting the average anyway.

**And the fifth, which is not a lie about one measurement but about the comparison:
profile pollution.**
Two implementations, one JVM, run in sequence. The second one is compiled against a call
site and a class hierarchy that the first one shaped. Symptom, and it is the exact
symptom to look for: **swap the order of the two cases and the answer changes.** This is
Topic 74's mechanism appearing in your measurement, and it is precisely why
`@Fork(3)` is not paranoia.

### What DOES transfer from your Node background

- **p50 versus p99 thinking** from Topic 65. You already know an average hides the tail.
  JMH's `@BenchmarkMode(Mode.SampleTime)` gives you percentiles for a single method the
  same way k6 gives them for an endpoint.
- **The instinct to isolate.** You would not benchmark an Express route by timing it
  inside the app with other routes serving traffic. Forking is the same instinct,
  applied to the JIT.
- **Suspicion of a number that is too good.** You have this. Use it. On the JVM, "too
  good" is the single most common benchmark outcome.

---

## What is this?

**JMH is the Java Microbenchmark Harness.** It is a build-time code generator plus a
runtime harness, developed and maintained by the OpenJDK team — the same people who
write HotSpot's compilers. That provenance is the reason to use it rather than a
hand-rolled timer: the tool is written by the people who wrote the optimisations that
would otherwise ruin your measurement.

Three plain statements about what it is:

**1. It is a code generator, not a timing library.**
You annotate a method with `@Benchmark`. An annotation processor
(`jmh-generator-annprocess`) runs during compilation and *writes a new Java source file*
containing the measurement loop, the state plumbing, the blackhole wiring, the iteration
control and the result accounting. That generated class is what runs. Your method is
called from inside it. You can and should read the generated file — the command is in
the Hands-on section, and reading it once demystifies the entire tool.

**2. It runs your benchmark in separate JVM processes.**
The `main` process is a coordinator. Each *fork* is a fresh `java` process launched with
a controlled command line, which runs the warmup and measurement iterations and streams
results back. This is what makes cross-case profile pollution structurally impossible
rather than merely unlikely.

**3. It reports a distribution.**
Not "12 ns". A mean, an error at a stated confidence level, and — depending on the mode —
percentiles. The error is not decoration. It is the part of the output that tells you
whether your result means anything.

### What it is NOT

| It is not | Because |
|---|---|
| A profiler | It tells you how long a method takes. It does not tell you where the time goes inside your service. That is Topic 78. |
| A load test | It exercises one method with synthetic inputs. It says nothing about connection pools, GC under sustained pressure, or tail latency at concurrency. That is Topic 65. |
| A way to benchmark a Spring service method | A `@Transactional` service method's cost is dominated by a network round trip to Postgres. Microbenchmarking it measures your database's mood. |
| A substitute for a production measurement | JMH answers "which of these two implementations is cheaper per call". It does not answer "does that matter", which is an Amdahl question and requires Topic 78. |
| Trustworthy across machines | A JMH number is a fact about one machine, one JDK build, one set of JVM flags, one CPU governor and one thermal state. |

### The three questions JMH can answer well

1. **"Which of these two implementations of the same pure function is cheaper per call?"**
   This is its home ground.
2. **"How does this data structure's cost scale with size and with thread count?"**
   `@Param` and `@Threads` make this a first-class workflow.
3. **"How much does this allocate?"** `-prof gc` gives you normalised allocation per
   operation, which is often the number you actually needed (Topics 68, 70).

### The three questions it answers badly, and what to use instead

| Question | Wrong tool | Right tool |
|---|---|---|
| "Why is `GET /orders` slow?" | JMH | Topic 78 — profiler, wall-clock mode |
| "Can we handle 2000 rps?" | JMH | Topic 65 — k6 against the containerised stack |
| "Why does RSS keep growing?" | JMH | Topics 79 and 80 — heap dump, NMT |

---

## Why does it matter?

### Because roughly fifteen other topics in this curriculum defer to it

Go and count. Topic 01 says boxing costs allocation "but I'd confirm with JMH". Topic 11
says `LinkedList` loses to `ArrayList` and that the claim needs a real benchmark. Topic
18 says the problem is not `+` but `+` inside a loop. Topic 21 says a capturing lambda
allocates. Topic 25 says only measure before reaching for `.parallel()`. Topic 95 says
`LongAdder` beats `AtomicLong` above some thread count — and *which* thread count is
exactly the sort of question that has no universal answer and must be measured on the
hardware you deploy on. Topic 96 says false sharing is real and needs JMH plus
`@Threads`. Topic 101 says virtual threads change the calculus for blocking code.

Every one of those is a hypothesis until you run this tool. **This document is where the
curriculum's performance claims stop being assertions.** That is why it is the longest
one in Phase 8, and why R0 above is written as strictly as it is.

### Because the wrong number is worse than no number

An engineer with no number says "I don't know, let me measure." An engineer with a
fabricated number rewrites 400 lines of readable code into an unreadable optimisation
that is measurably nothing, ships it, and blocks the next person's PR with it for three
years. Bad measurement is not neutral; it actively produces worse systems and worse
teams.

### Because it is the difference between two very different engineers in an interview

Mid: *"I timed it in a loop and it took 3 nanoseconds."*

Senior: *"Three nanoseconds is about the cost of nothing — a couple of cycles — which is
what an empty loop measures. Something eliminated the work. I'd port it to JMH with a
`Blackhole` and `@State`, run three forks, and look at the error bars before believing
any comparison. And before doing any of that I'd ask whether this method is even on the
critical path, because if it's 0.1% of the request I'm optimising the wrong thing."*

The gap between those two answers is not knowledge of an API. It is a whole disposition
toward evidence.

### Because "faster" is a claim about a distribution, and most people report a point

The single most valuable habit this topic gives you is refusing to say "A is faster than
B" and saying instead "A is faster than B **by X ± Y**, and Y overlaps zero, so I am not
going to act on it." Reporting the error is the professional standard, and almost nobody
does it.

### Because of what happens after the benchmark

This is the part nobody teaches. You benchmark two implementations of the order-total
calculation. One wins per call. You ship it. **The endpoint's p99 does not move**,
because the calculation was a rounding error against a Postgres round trip. You have
spent a week and made the code worse.

The correct sequence is:

1. **Topic 78** — profile the running service, find where the time actually goes.
2. **Topic 77 (here)** — microbenchmark the specific method the profile named.
3. **Topic 65** — re-run the load baseline to prove the system-level number moved.

A JMH result on its own justifies nothing. It is the middle step, and it is worthless
without the first and last. Say this in an interview.

---

## Machine-level reality

This section is the part that turns JMH from a magic incantation into a tool you
understand. Three mechanisms: the generated harness, forking, and the blackhole.

### 1. The generated harness — what actually runs

You write this:

```java
@Benchmark
public long totalMinorUnits(OrderLineState state) {
    long sum = 0;
    for (OrderLine line : state.lines) {
        sum += line.unitPriceMinor() * line.quantity();
    }
    return sum;
}
```

At build time, `jmh-generator-annprocess` reads that annotation and writes a new source
file next to your compiled classes. **Go and read it.** This command is the single most
useful thing in this section:

```bash
mvn -q clean verify
find target/generated-sources -name '*_jmhTest.java' | head
# then open one:
find target/generated-sources -name '*_jmhTest.java' -exec sed -n '1,200p' {} \; | less
```

**WHAT TO LOOK FOR in the generated file:**

| Look for | What it tells you |
|---|---|
| A method per `@Benchmark` per mode, named like `totalMinorUnits_thrpt_jmhStub` | JMH generates a *separate* loop per benchmark mode. Throughput and average-time are different generated code. |
| A `while (!control.isDone) { ... }` loop | The measurement loop is generated, not written by you. You cannot get it wrong. |
| `l_blackhole1_1.consume(l_orderbench0_0.totalMinorUnits(l_orderlinestate1_1))` | Your return value is fed to a `Blackhole` **automatically**. Returning a value is the implicit blackhole. |
| `operations++` next to the call | This is what becomes the throughput numerator. |
| `res.allOps`, `res.measuredOps` | JMH accounts for warmup and measurement operations separately. |
| Setup/teardown calls guarded by `if (!state.readyTrial)` style flags | `@Setup(Level.Trial)` really does run once, and JMH generates the guard. |
| A `RawResults` object and a `long realTime` field | Where `@BenchmarkMode(Mode.SingleShotTime)` and `AuxCounters` get their data. |

> **Flagged uncertainty.** The exact identifier names, the exact loop shape, and the file
> naming convention vary between JMH versions. Do not memorise names from this table —
> memorise the *categories* (a generated loop, an automatic blackhole call, an operation
> counter, setup guards) and find them in your own generated file. That is the whole
> point of the command.

Once you have read that file, one thing becomes obvious and stays with you: **JMH is not
doing anything you could not do by hand. It is doing a dozen things you would forget to
do by hand, every time, correctly.**

### 2. Forking — and why it defeats profile pollution

**The spec-level fact:** JMH launches a **new JVM process** for each fork. Confirm it
yourself:

```bash
# Terminal 1 — run a benchmark with several forks
java -jar target/benchmarks.jar OrderTotalBenchmark -f 3

# Terminal 2 — during the run, watch a second java process appear and disappear
watch -n 1 'jps -l'
```

**WHAT TO LOOK FOR:** in terminal 2, a second `java` process whose command line contains
the forked-JVM arguments, appearing and being replaced three times. In terminal 1, three
blocks each beginning `# Fork: 1 of 3`, `# Fork: 2 of 3`, `# Fork: 3 of 3`, each
re-printing the JVM banner.

**Why a fresh process, specifically:**

| What resets on a fork | Why it matters |
|---|---|
| **Call-site profiles** | Topic 74. A call site that saw only `LongMinorUnitTotaller` for a million calls is monomorphic and inlined. Run `BigDecimalTotaller` next in the same JVM and that site becomes bimorphic, deoptimizes, and recompiles — permanently worse. A new process has never seen either. |
| **Class hierarchy analysis (CHA)** | C2 devirtualises a call when only one implementation of an interface is *loaded*. Load a second implementation and every such optimisation is invalidated. Loading order therefore changes compiled code. |
| **Code cache contents and compilation queue state** | Compilation is asynchronous. A JVM that has already compiled 4000 methods behaves differently from one that has compiled 40. |
| **Heap shape, GC generation state, TLAB sizing** | Topic 68. A heap that has done 200 young collections has different eden occupancy and different TLAB sizes from a fresh one. |
| **Metaspace, string table, interned constants** | Topic 18. |
| **JIT compiler tuning decisions that adapt over time** | Inlining budgets and tier thresholds are adaptive. |

**Why more than one fork.** A single fork gives you the variance *within* one JVM. That
is the small variance. The large variance is *between* JVMs: different code layout in
the code cache, different compilation orderings, different address-space layout
randomisation and therefore different cache-set conflicts. Runs of the same benchmark in
different JVMs can differ by more than the effect you are trying to measure.

**Therefore: with `-f 1` you cannot distinguish "A is faster than B" from "this JVM
happened to lay out A well".** Three forks is the working minimum. Five is better when a
decision rides on it.

There is a `-f 0` mode that runs in the harness JVM with no fork at all. It exists for
debugging your benchmark under a debugger and for nothing else. **Any number produced
with `-f 0` is not a result.** JMH itself prints a warning when you use it. Read the
warning.

### 3. `Blackhole` — the object the compiler cannot see through

Dead-code elimination is legal because C2 can prove the result is unobserved. A
`Blackhole` exists to make that proof impossible.

There are two implementations, and which one you get depends on your JMH version and JVM:

**Implementation A — the classic "volatile arithmetic" blackhole.** `Blackhole.consume`
compares the value against a `volatile` field in a way that is arranged never to be true
in practice, but which the compiler cannot prove. Because a `volatile` read cannot be
elided and the branch cannot be folded, the value must be materialised. It is genuinely
clever and it has a genuine, small, non-zero cost — which is why you never compare a
benchmark that uses one `consume` call against one that uses four.

**Implementation B — compiler blackholes.** Newer JMH versions can ask HotSpot itself to
treat a method as a blackhole, via `-XX:+UnlockDiagnosticVMOptions` and a
`-XX:CompileCommand=blackhole,...` directive. The compiler then knows the arguments are
consumed and emits nothing at all, which is both cheaper and more reliable.

**How to find out which one you are getting — this is the command:**

```bash
java -jar target/benchmarks.jar OrderTotalBenchmark -f 1 -wi 1 -i 1 2>&1 | head -40
```

**WHAT TO LOOK FOR:** JMH prints a line in the run banner about blackhole mode. Read
your own banner. If it names a compiler blackhole, good. If it says it fell back to the
classic mode, that is also fine — just be aware there is a small fixed cost per
`consume`.

> **Flagged uncertainty.** I am confident that compiler blackholes exist and are
> negotiated at startup via a `CompileCommand`. I am **not** confident enough about the
> exact banner wording, the exact flag spelling, or which JMH version made it the
> default to have you copy it from this document. Run the command above and read your
> own banner. That is the authority.

**The rule that follows:** consume everything you compute, and consume the *same number
of things* in every case you are comparing. A benchmark where variant A consumes one
value and variant B consumes three is measuring blackhole calls as much as it is
measuring your code.

### 4. `@State` — why your inputs must live in a field

C2 constant-folds through `static final` fields whose values it can see. It also folds
through values it has proved constant by profiling. Either way, a benchmark whose input
is a constant is a benchmark of "return a precomputed answer".

A `@State` object is instantiated by the generated harness and passed as a parameter.
Its fields are read through a reference the compiler cannot prove constant. That is the
entire mechanism, and it is why the fix for constant folding is structural rather than a
flag.

The three scopes:

| Scope | One instance per | Use it for |
|---|---|---|
| `Scope.Benchmark` | The whole benchmark, shared by all threads | Shared read-only input data; the contended structure in a concurrency benchmark (Topics 95, 96) |
| `Scope.Thread` | Each thread | Per-thread scratch space; anything you must not share |
| `Scope.Group` | Each `@Group` of threads | Asymmetric benchmarks — producer/consumer, reader/writer (Topic 94) |

**The trap that lives here:** with `Scope.Benchmark` and `@Threads(N)`, your state is
shared and therefore contended. That is correct and desirable for a concurrency
benchmark and catastrophic for a single-threaded one where you did not intend sharing.

### 5. On-stack replacement, and how JMH mostly (not entirely) avoids it

The generated measurement loop is a method that JMH invokes repeatedly — once per
iteration — with many operations inside each invocation. Because that method is entered
many times across warmup, it accumulates invocation counts and gets compiled through the
normal path, not only through OSR. Your `@Benchmark` method is a separate method that is
invoked millions of times and is therefore very definitely compiled normally.

**Be precise about what this does and does not guarantee.** JMH makes the *benchmark
method* a normally-compiled method, which is the important part. It does not magically
abolish OSR everywhere; if you write your own long loop *inside* the benchmark method,
that inner loop can still be OSR-compiled, and if the loop count is large you are back to
measuring OSR code.

**The practical rule:** if your benchmark method contains a loop over a large collection,
you are benchmarking a loop, and you should know that. Prefer `@Param` over a hard-coded
large N, and prefer benchmarking the unit of work your production code actually performs
in one call. `orderflow` computes a total over ~5 order lines per order, not 5 million,
so the honest benchmark loops over ~5.

**How to check what got compiled:**

```bash
java -jar target/benchmarks.jar OrderTotalBenchmark -f 1 \
  -jvmArgs "-XX:+UnlockDiagnosticVMOptions -XX:+PrintCompilation" 2>&1 | grep -i "totalMinorUnits"
```

**WHAT TO LOOK FOR:** compilation lines for your benchmark method. A `%` in the
`PrintCompilation` flags column marks an **OSR compilation**. Seeing `%` against your own
method's inner loop is the signal that you have written a loop benchmark.

| What you see | What it means |
|---|---|
| Your `@Benchmark` method compiled at level 4, no `%` | Normal C2 compilation. This is what you want. |
| A `%` (OSR) entry for your method | You have a long-running loop inside the benchmark method. Legitimate if you meant it; a bug if you did not. |
| Repeated `made not entrant` lines for your method during *measurement* | Deoptimization is happening mid-measurement (Topic 74). Your warmup was too short, or your benchmark's behaviour changes over time. Investigate before believing the number. |
| No compilation line for your method at all | It was inlined into the generated stub, which is normal and fine. Look for the stub instead. |

### 6. Statistics — what the error actually is

JMH's default reporting gives a **mean** and an **error at 99.9% confidence**, computed
across iterations (and across forks, which is the important part). The confidence
interval is computed from the sample of iteration results, so:

- **More iterations narrows it.** More forks narrows it in the way that matters, because
  it samples the between-JVM variance rather than the within-JVM variance.
- **A wide error means your benchmark is unstable**, not that your machine is bad. Common
  causes: a laptop on battery, thermal throttling, another process, a GC storm inside the
  benchmark, an unstable data structure state.
- **Two results whose error bars overlap are not distinguishable.** Full stop. Not
  "probably the same". Not distinguishable by this experiment. You may run more forks
  and try again, or accept that the difference is below your measurement floor.

`Mode.SampleTime` gives you percentiles instead — p50, p90, p99, p999, max — which is
the right mode when you care about the tail of a single operation rather than its mean.

---

## Example 1 — minimal

**The question:** summing the quantities on an order's lines. Does it matter whether we
hold them as `List<Integer>` or `int[]`? This is Topic 01 (boxing) and Topic 11
(`ArrayList` and cache behaviour) meeting Topic 77 for the first time.

### Step 1 — the wrong benchmark, written exactly as you would write it today

```java
package com.orderflow.bench;

import java.util.ArrayList;
import java.util.List;

/**
 * THIS CLASS IS THE ANTI-PATTERN. It is here to be broken, not to be copied.
 */
public class NaiveQuantityBenchmark {

    // Lie 2 waiting to happen: a static final the compiler can see through.
    private static final int LINE_COUNT = 8;

    public static void main(String[] args) {
        List<Integer> boxed = new ArrayList<>();
        int[] primitive = new int[LINE_COUNT];
        for (int i = 0; i < LINE_COUNT; i++) {
            boxed.add(i + 1);
            primitive[i] = i + 1;
        }

        long t0 = System.nanoTime();
        for (int i = 0; i < 10_000_000; i++) {
            sumBoxed(boxed);                 // Lie 1: result discarded
        }
        long t1 = System.nanoTime();
        System.out.println("boxed:     " + (t1 - t0) / 10_000_000.0 + " ns/op");

        long t2 = System.nanoTime();
        for (int i = 0; i < 10_000_000; i++) {
            sumPrimitive(primitive);         // Lie 1 again, plus Lie 5:
        }                                    // this case runs in a JVM the first case warmed
        long t3 = System.nanoTime();
        System.out.println("primitive: " + (t3 - t2) / 10_000_000.0 + " ns/op");
    }

    static int sumBoxed(List<Integer> lines) {
        int total = 0;
        for (Integer q : lines) {
            total += q;                      // unboxing per element (Topic 01)
        }
        return total;
    }

    static int sumPrimitive(int[] lines) {
        int total = 0;
        for (int q : lines) {
            total += q;
        }
        return total;
    }
}
```

Count the defects — there are five, and they are the five from the mechanical statement:

1. **Dead-code elimination.** Neither `sumBoxed` nor `sumPrimitive` has its result used.
2. **Constant folding.** `LINE_COUNT` is `static final`; the array contents never change
   and C2 may prove it.
3. **On-stack replacement.** The ten-million-iteration loop is inside `main`, entered
   once. It is an OSR compilation by construction.
4. **Cold JIT.** The first case pays all the warm-up cost. The second case starts warm.
5. **Profile pollution and ordering.** Both cases run in one JVM, in a fixed order. The
   `Iterable` call site in `sumBoxed` and the array loop in `sumPrimitive` share nothing
   directly, but the JVM's compilation queue, code cache and GC state are shared, and
   `sumPrimitive` starts with a JVM that has already done ten million iterations of work.

### Step 2 — run it and predict the shape of the answer

```bash
javac -d out NaiveQuantityBenchmark.java
java -cp out com.orderflow.bench.NaiveQuantityBenchmark
```

I am not going to tell you what number you will get. I will tell you **what to look
for**, and what each outcome means:

| What you see | What it means |
|---|---|
| A per-op figure well under one nanosecond for either case | Dead-code elimination. Sub-nanosecond means fewer than about three cycles on a typical modern core — you cannot iterate eight elements in three cycles. The loop was removed. |
| The two cases reporting the same number to two significant figures | Both were eliminated, or both were folded to a constant. The measurement is not distinguishing them because there is nothing left to distinguish. |
| The boxed case *faster* than the primitive case | Almost certainly ordering: the second case is measuring something the first case's warm-up already paid for, or the first case ran while the JIT was still compiling. Swap the two blocks and re-run — if the answer follows the *position* rather than the *code*, you have proved profile/ordering contamination. |
| A number that halves when you double the iteration count | Cold-JIT averaging (Lie 4). You are averaging interpreted, C1 and C2 regimes. |
| A number that changes by a factor of several between consecutive runs of the same binary | Between-JVM variance, which is exactly what `@Fork(3)` exists to sample and this program cannot see at all. |
| A plausible-looking, stable number that you are tempted to believe | **This is the dangerous outcome.** Stability is not validity. Run it under `-Xint` (below); if the ratio between the two cases changes dramatically, the JIT was doing something to your measurement. |

**The confirmatory experiment that costs nothing:**

```bash
# Interpreter only: no C1, no C2, therefore no DCE, no folding, no OSR.
java -Xint -cp out com.orderflow.bench.NaiveQuantityBenchmark
```

**WHAT TO LOOK FOR:** the interpreted run will be far slower in absolute terms — that is
expected and uninteresting. **The interesting thing is the ratio between the two cases.**
If the boxed/primitive ratio under `-Xint` is wildly different from the ratio under the
default, then the JIT was materially reshaping at least one of the two cases, and your
default-mode numbers were describing compiler behaviour rather than algorithmic cost.

### Step 3 — the same question, correctly

```java
package com.orderflow.bench;

import org.openjdk.jmh.annotations.*;
import org.openjdk.jmh.infra.Blackhole;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.TimeUnit;

@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@State(Scope.Benchmark)
@Warmup(iterations = 5, time = 1, timeUnit = TimeUnit.SECONDS)
@Measurement(iterations = 5, time = 1, timeUnit = TimeUnit.SECONDS)
@Fork(3)
public class QuantitySumBenchmark {

    /** Realistic: orderflow's dataset averages ~5 lines per order. Not 5 million. */
    @Param({"3", "8", "40"})
    public int lineCount;

    private List<Integer> boxed;
    private int[] primitive;

    @Setup(Level.Trial)
    public void setUp() {
        boxed = new ArrayList<>(lineCount);
        primitive = new int[lineCount];
        for (int i = 0; i < lineCount; i++) {
            int quantity = 1 + (i % 7);
            boxed.add(quantity);
            primitive[i] = quantity;
        }
    }

    /** The baseline. Measures the harness floor: loop, state read, blackhole. */
    @Benchmark
    public void baseline(Blackhole bh) {
        bh.consume(primitive);
    }

    @Benchmark
    public int sumBoxed() {
        int total = 0;
        for (Integer q : boxed) {
            total += q;
        }
        return total;                 // returned -> implicit blackhole
    }

    @Benchmark
    public int sumPrimitive() {
        int total = 0;
        for (int q : primitive) {
            total += q;
        }
        return total;
    }
}
```

Every difference from the naive version is a defence, and you should be able to name
which lie each one defeats:

| Element | Defends against |
|---|---|
| `@State(Scope.Benchmark)` + instance fields | **Constant folding.** `boxed` and `primitive` are read through a non-constant reference. |
| `return total` / `bh.consume(...)` | **Dead-code elimination.** The result is observed. |
| `@Warmup(iterations = 5, time = 1s)` | **Cold JIT.** Five seconds of execution before anything is recorded. |
| `@Fork(3)` | **Profile pollution and between-JVM variance.** Three independent processes; no case can contaminate another. |
| `@Measurement(iterations = 5)` | Gives the statistics something to compute an error from. One iteration has no error. |
| JMH's generated loop | **OSR.** The benchmark method is invoked millions of times and compiled through the normal path. |
| The `baseline` benchmark | The measurement floor. Any result close to it is at the noise floor and means nothing. |
| `@Param` at realistic sizes | Stops you answering a question nobody asked (`lineCount = 1_000_000`). |

### Step 4 — run it

```bash
mvn -q clean verify
java -jar target/benchmarks.jar QuantitySumBenchmark
```

**WHAT TO LOOK FOR, in this order:**

1. **The `baseline` row.** Everything else is measured against it. If `sumPrimitive` is
   within the error of `baseline` at `lineCount = 3`, then at three elements the work is
   below your measurement floor and you have learned something real: *at this size, it
   does not matter.*
2. **The error column on every row.** Wide errors invalidate comparisons before you make
   them.
3. **The trend across `@Param` values.** A per-element cost difference should scale with
   `lineCount`. If the gap is constant across 3, 8 and 40, it is a fixed cost (an
   allocation, a virtual call), not a per-element cost.
4. **The gap between this answer and the naive one.** That gap is the entire lesson of
   this topic, and the failure drill makes you write it down.

| What you see | What it means |
|---|---|
| `sumBoxed` slower than `sumPrimitive`, gap growing with `lineCount` | The expected result: per-element unboxing and pointer-chasing cost (Topics 01, 11). The gap's *slope* is the real finding. |
| `sumBoxed` and `sumPrimitive` indistinguishable at `lineCount = 3` | Also expected, and more useful. At the size `orderflow` actually uses, the difference is below the floor. **Do not rewrite the code.** |
| `sumBoxed` *faster* | Surprising. Check `-prof gc` first: if `sumPrimitive` is allocating and `sumBoxed` is not, your `@Setup` is doing something you did not intend. Then check that both consume the same number of values. |
| Error bars wider than the effect at every `@Param` | Your machine is not stable enough for this measurement. Close everything, disable turbo/thermal-variable modes if you can, pin CPU governor to performance, raise forks, and re-run. If it persists, the effect is smaller than your floor. |
| Fork-to-fork means that differ by more than the within-fork error | Real between-JVM variance. This is exactly what `@Fork(3)` was for; report the aggregate error, not one fork. |

Add the allocation profiler, because for this particular question allocation is half the
story:

```bash
java -jar target/benchmarks.jar QuantitySumBenchmark -prof gc
```

**WHAT TO LOOK FOR:** the `gc.alloc.rate.norm` row, which is **bytes allocated per
operation, normalised**. This is the single most useful number in JMH's whole profiler
suite and it is the one people ignore.

| What you see in `gc.alloc.rate.norm` | What it means |
|---|---|
| `sumPrimitive` at zero bytes per op | Correct. Summing an `int[]` allocates nothing. |
| `sumBoxed` at zero bytes per op | Surprising but plausible: the `Integer` values are from the `-128..127` cache (Topic 01), so iteration allocates nothing — only the `Iterator` might, and escape analysis (Topic 75) may scalar-replace it. **This is a genuine finding**: the boxing cost here is indirection, not allocation. |
| `sumBoxed` allocating a fixed number of bytes per op regardless of `lineCount` | The `Iterator` object, not scalar-replaced. Confirm with `-XX:-DoEscapeAnalysis` as an A/B (Topic 75). |
| Either benchmark allocating a large amount per op | Your `@Setup` level is wrong — you are probably rebuilding the data per invocation. Check for `Level.Invocation`. |

---

## Example 2 — production scenario (on the project spine)

### The constraints, taken from the Topic 65 baseline

| Thing | Value |
|---|---|
| Service | `orderflow`, containerised |
| Dataset | 100k products, 1M orders, **5M order lines** (≈5 lines per order) |
| Container | 2 vCPU, 2 GiB memory limit |
| Heap | `-Xmx1200m -Xms1200m` |
| Collector | G1, verified with `jcmd <pid> VM.flags -all` |
| Load mix | 70% catalogue read, 20% order read, 10% order placement |
| Baseline artefacts | `/docs/java/baselines/<date>-run-01/` — p50/p95/p99/p999 per endpoint |
| Gate rule | re-running the baseline must land within ±10% on every recorded percentile |

### The question that arrives from a real code review

A senior engineer reviews the order-total path and writes:

> *"We're doing `BigDecimal` arithmetic per order line for money. `BigDecimal` allocates
> on every operation and does software decimal arithmetic. We should hold money as `long`
> minor units and only convert to `BigDecimal` at the API boundary. This is a hot path —
> `POST /orders` is 10% of our traffic and `GET /orders/{id}` is 20%."*

This is a **good** review comment. It names a mechanism, it names a hot path, and it
proposes a specific change. It is also, as stated, an unfalsifiable claim. Your job is to
turn it into a number with an error bar, and then — this is the senior part — to decide
whether the number justifies the change.

### The two implementations

```java
package com.orderflow.pricing;

import java.math.BigDecimal;
import java.math.RoundingMode;
import java.util.List;

public final class OrderTotals {

    private OrderTotals() {}

    /** Current production implementation. Money as BigDecimal throughout. */
    public static BigDecimal totalDecimal(List<OrderLine> lines, BigDecimal taxRate) {
        BigDecimal subtotal = BigDecimal.ZERO;
        for (OrderLine line : lines) {
            BigDecimal lineTotal = line.unitPrice()
                    .multiply(BigDecimal.valueOf(line.quantity()));
            if (line.discountPercent() > 0) {
                BigDecimal factor = BigDecimal.valueOf(100 - line.discountPercent())
                        .divide(BigDecimal.valueOf(100), 6, RoundingMode.HALF_UP);
                lineTotal = lineTotal.multiply(factor);
            }
            subtotal = subtotal.add(lineTotal);
        }
        BigDecimal tax = subtotal.multiply(taxRate).setScale(2, RoundingMode.HALF_UP);
        return subtotal.add(tax).setScale(2, RoundingMode.HALF_UP);
    }

    /** Proposed implementation. Money as long minor units (cents). */
    public static long totalMinorUnits(List<OrderLine> lines, int taxBasisPoints) {
        long subtotal = 0L;
        for (OrderLine line : lines) {
            long lineTotal = line.unitPriceMinor() * line.quantity();
            int discount = line.discountPercent();
            if (discount > 0) {
                // integer maths with explicit half-up rounding at the last step
                lineTotal = Math.round(lineTotal * (100 - discount) / 100.0d);
            }
            subtotal += lineTotal;
        }
        long tax = Math.round(subtotal * taxBasisPoints / 10_000.0d);
        return subtotal + tax;
    }
}
```

Two things to notice before benchmarking anything, because a benchmark of two things
that are not equivalent is worse than no benchmark:

1. **They do not necessarily produce the same answer.** `BigDecimal` with an explicit
   `RoundingMode` and `double`-mediated integer rounding round differently at ties. Money
   correctness dominates performance. **Write a property-based test (Topic 64) asserting
   the two agree across the full input domain before you benchmark either.** If they
   disagree, the benchmark is irrelevant — you have a correctness decision to make first.
2. **The `double` in the proposed version is a smell for money.** A production version
   would use pure integer arithmetic for the discount. I have left it as written because
   this is what the PR actually contained, and the review should catch it.

### The harness

```java
package com.orderflow.bench;

import com.orderflow.pricing.OrderLine;
import com.orderflow.pricing.OrderTotals;
import org.openjdk.jmh.annotations.*;
import org.openjdk.jmh.infra.Blackhole;

import java.math.BigDecimal;
import java.util.ArrayList;
import java.util.List;
import java.util.Random;
import java.util.concurrent.TimeUnit;

@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@State(Scope.Benchmark)
@Warmup(iterations = 5, time = 1, timeUnit = TimeUnit.SECONDS)
@Measurement(iterations = 10, time = 1, timeUnit = TimeUnit.SECONDS)
@Fork(value = 3, jvmArgs = {"-Xms1200m", "-Xmx1200m", "-XX:+UseG1GC"})
@Threads(1)
public class OrderTotalBenchmark {

    /**
     * From the Topic 65 dataset: 5M order lines over 1M orders = 5 lines/order mean.
     * 1 is the floor case, 5 the mean, 25 the tail (bulk B2B orders).
     * There is no value here above 25 because orderflow does not have orders above 25
     * lines in any meaningful volume, and benchmarking a size that does not occur
     * answers a question nobody asked.
     */
    @Param({"1", "5", "25"})
    public int lineCount;

    private List<OrderLine> lines;
    private BigDecimal taxRate;
    private int taxBasisPoints;

    @Setup(Level.Trial)
    public void setUp() {
        // Fixed seed: the benchmark must be reproducible run to run.
        Random rnd = new Random(20260828L);
        lines = new ArrayList<>(lineCount);
        for (int i = 0; i < lineCount; i++) {
            long unitPriceMinor = 199L + rnd.nextInt(48_000);   // 1.99 .. 480.99
            int quantity = 1 + rnd.nextInt(4);
            // 70% of lines have no discount, matching the seeded dataset's skew
            int discountPercent = rnd.nextInt(10) < 7 ? 0 : (5 + rnd.nextInt(20));
            lines.add(new OrderLine(unitPriceMinor, quantity, discountPercent));
        }
        taxRate = new BigDecimal("0.20");
        taxBasisPoints = 2000;
    }

    /** MANDATORY. The measurement floor. Never omit this. */
    @Benchmark
    public void baseline(Blackhole bh) {
        bh.consume(lines);
        bh.consume(taxBasisPoints);
    }

    @Benchmark
    public BigDecimal decimal() {
        return OrderTotals.totalDecimal(lines, taxRate);
    }

    @Benchmark
    public long minorUnits() {
        return OrderTotals.totalMinorUnits(lines, taxBasisPoints);
    }
}
```

### Running it, with the flags that matter

```bash
# 1. Build. The annotation processor runs here; if it does not, nothing works.
mvn -q clean verify
ls -l target/benchmarks.jar

# 2. Sanity run: does the harness work at all? Not a result.
java -jar target/benchmarks.jar OrderTotalBenchmark -f 1 -wi 1 -i 1 -r 1s

# 3. The real run, results to a machine-readable file.
java -jar target/benchmarks.jar OrderTotalBenchmark \
     -rf json -rff /docs/java/baselines/bench/order-total.json

# 4. The allocation question, which is half the review comment's claim.
java -jar target/benchmarks.jar OrderTotalBenchmark -prof gc

# 5. If you are on Linux and want to see the generated assembly for the hot loop.
#    Requires hsdis; if it is not installed JMH will say so.
java -jar target/benchmarks.jar OrderTotalBenchmark.minorUnits \
     -f 1 -prof perfasm
```

### WHAT TO LOOK FOR — and the decision each outcome drives

**First, the `baseline` row.** If `minorUnits` at `lineCount = 1` is within the error of
`baseline`, then at one line the entire computation is below the harness floor.

**Second, `gc.alloc.rate.norm`.** The review comment's mechanism was "`BigDecimal`
allocates". This row is where that claim lives or dies.

| What you see | What it means | What you do |
|---|---|---|
| `decimal` allocates a substantial and `lineCount`-proportional number of bytes per op; `minorUnits` allocates zero | The mechanism in the review comment is **confirmed**. Now the question is magnitude, not mechanism. | Proceed to the Amdahl step below. Do not ship yet. |
| Both allocate zero | Escape analysis (Topic 75) scalar-replaced the intermediate `BigDecimal`s. Genuinely possible for small `lineCount` with good inlining. | Re-run with `-jvmArgs -XX:-DoEscapeAnalysis` as the control. If `decimal` then allocates, EA is carrying it — and EA is fragile: a slightly larger method in production may not inline and the allocation returns. Note this in the PR as a risk, not a result. |
| `decimal` allocates a *constant* amount regardless of `lineCount` | Only the final `setScale` results escaped; the loop intermediates were scalar-replaced. | Interesting and worth reporting. The optimisation opportunity is smaller than the reviewer thought. |
| `minorUnits` allocates non-zero | Something boxed. Look for an `Integer`/`Long` autobox (Topic 01) or a lambda capture (Topic 21). Check with `javap -c` (Topic 76). | Fix the benchmark or the code, then re-measure. |

**Third, the time difference, expressed the only acceptable way.**

Write it in your PR comment as: *"`minorUnits` is faster than `decimal` by **X ± Y ns per
call** at `lineCount = 5`."* Both numbers, from your own run. Never the mean alone.

| What you see | What it means | What you do |
|---|---|---|
| A clear gap that scales with `lineCount`, errors well separated | A real per-element cost difference. | Go to Amdahl. |
| A gap smaller than the combined errors | Not distinguishable by this experiment. | Raise to `-f 5` and re-run once. If still overlapping, report "no measurable difference" — which is a result, and a valuable one. |
| A large gap at `lineCount = 25` and none at `lineCount = 1` | Fixed costs dominate small orders. Since 5 is the mean, the mean case is what you should quote. | Quote the `lineCount = 5` row as the headline; show all three. |
| `decimal` *faster* | Surprising. Most likely your `minorUnits` implementation is doing `double` division and hitting a slow path, or the `Math.round` is not being inlined. | Read `-prof perfasm` for `minorUnits`, or check `-XX:+PrintInlining` (Topic 75). Do not report a surprising result without investigating it. |
| Results differ substantially between forks | Between-JVM variance is dominating. | This is the value of `@Fork(3)`. Report the aggregate error. If it is huge, your machine is unsuitable — a shared CI runner is a classic cause. |

### The step that makes this a senior answer: Amdahl against the baseline

Suppose your run establishes a saving of **S nanoseconds per order-total computation** at
`lineCount = 5`, with a clean error bar. Do not ship anything yet. Do this arithmetic,
using **your own** `S` and **your own** baseline numbers:

1. **How many total computations per request?** `GET /orders/{id}` computes one total.
   `POST /orders` computes one. So: one.
2. **What is the endpoint's measured p50 from `/docs/java/baselines/`?** Call it `P`
   milliseconds. Look it up; do not guess it.
3. **Compute `S / (P × 1_000_000)` as a fraction.** That is the maximum possible
   improvement to p50 from this change, assuming the saving is entirely on the critical
   path and nothing else changes.
4. **Compare that fraction to the ±10% gate tolerance** from Topic 65.

If the fraction is far below your gate tolerance, **the change cannot be validated by
your own load test**, which means you cannot prove it helped, which means you should not
make it for performance reasons. You might still make it for correctness reasons —
integer money arithmetic has real advantages — but then say so, and stop citing
performance.

**Write this conclusion in the PR. It is the most valuable thing in the review:**

> *"Confirmed the mechanism: `decimal` allocates ~`<n>` B/op at 5 lines and `minorUnits`
> allocates zero (`-prof gc`, 3 forks, error `± <n>`). The time saving is `<n> ± <n>` ns
> per call. Our `GET /orders/{id}` p50 baseline is `<n>` ms, so this is `<n>`% of one
> request — roughly two orders of magnitude below the ±10% tolerance our load-test gate
> can resolve. I cannot demonstrate this change at the system level, so I do not think it
> is justified as a performance change. If we want it for money-correctness reasons —
> and I think there is a case — let us make that argument instead, and I will pair on
> the rounding-equivalence property test."*

That paragraph is what separates a senior engineer from a mid engineer, and it contains
one benchmark, one profile and one piece of division.

### The follow-up the reviewer will raise, and the correct answer

> *"But the allocation rate matters even if the latency doesn't — we're reducing GC
> pressure."*

This is a fair point and it has its own measurement, which is **not** JMH. Allocation
rate at the system level is a Topic 68/70 measurement, taken from GC logs under the
Topic 65 load:

```bash
# On the running container, with -Xlog:gc*:file=/logs/gc.log:time,uptime,level,tags
# derive allocation rate from eden occupancy between young collections (Topic 68).
docker exec orderflow jcmd 1 GC.heap_info
```

`gc.alloc.rate.norm` from JMH tells you bytes per call. Multiplying it by your measured
request rate gives you an *estimate* of the contribution to system allocation rate.
Compare that estimate to the total allocation rate you measured in Topic 70. If the
order-total path is 0.4% of your allocation rate, the GC argument is also unsupported —
and now you have said so with two independent measurements instead of an opinion.

---

## Wrong approach → exact symptom → root cause → fix

Five traps. Each is stated as something you will actually observe, not as a principle.

---

### Trap 1 — the `nanoTime` loop that reports an absurd number because the loop was eliminated

**Wrong approach.**

```java
long t0 = System.nanoTime();
for (int i = 0; i < 100_000_000; i++) {
    OrderTotals.totalMinorUnits(lines, 2000);   // result discarded
}
System.out.println((System.nanoTime() - t0) / 100_000_000.0 + " ns/op");
```

**Exact symptom.** The program prints a per-operation figure **below one nanosecond** —
often around a quarter of a nanosecond, sometimes reported as zero. And crucially:
**the number does not change when you make the method do more work.** Add another five
lines to the order and re-run: the figure is the same.

That second observation is the diagnostic. A real measurement responds to a change in the
work. A dead-code-eliminated one does not, because there is no work.

**Root cause.** `totalMinorUnits` is a pure function whose result is discarded. C2 inlines
it, observes that nothing in the method has an effect outside itself and that its return
value is unused, and deletes the whole thing. What remains is
`for (int i = 0; i < 100_000_000; i++) {}` — and C2 will very likely delete that too,
because an empty counted loop with no side effects is itself dead. You are timing two
`nanoTime` calls and a print.

**Prove it in thirty seconds:**

```bash
# If DCE is the cause, the interpreter (which performs no optimisation) will
# report a per-op cost many orders of magnitude larger.
java -Xint  -cp out com.orderflow.bench.NaiveQuantityBenchmark
java -cp out com.orderflow.bench.NaiveQuantityBenchmark
```

| What you see | What it means |
|---|---|
| `-Xint` figure enormously larger than the default figure | Expected regardless — the interpreter is 10–100× slower. Not by itself proof. |
| Default figure **does not respond** to adding work; `-Xint` figure **does** | **Confirmed DCE.** The optimised build removed the work; the interpreter could not. |
| Both figures respond proportionally to added work | DCE is *not* your problem here. Look at constant folding (Trap 2) or accept that the JIT is genuinely that fast and port to JMH to find out. |

**Fix.** Observe the result. In JMH that means returning it, or passing it to a
`Blackhole`:

```java
@Benchmark
public long minorUnits() {
    return OrderTotals.totalMinorUnits(lines, taxBasisPoints);   // implicit blackhole
}

// or, when a benchmark produces several values:
@Benchmark
public void bothVariants(Blackhole bh) {
    bh.consume(OrderTotals.totalMinorUnits(lines, taxBasisPoints));
    bh.consume(OrderTotals.totalDecimal(lines, taxRate));
}
```

**The rule to carry away:** *any benchmark result you do not consume, the compiler is
entitled to delete — and it will.*

---

### Trap 2 — a benchmark whose result changes when you reorder the cases in one JVM

**Wrong approach.** Two implementations, one `main`, one JVM:

```java
public static void main(String[] args) {
    timeIt("decimal",    () -> OrderTotals.totalDecimal(lines, taxRate));
    timeIt("minorUnits", () -> OrderTotals.totalMinorUnits(lines, 2000));
}
```

**Exact symptom.** You run it: `minorUnits` wins. A colleague, tidying up, swaps the two
lines. Now `decimal` wins. **The answer follows the position in the file, not the code.**

This is the exact symptom to look for, and it is worth deliberately provoking once,
because after you have seen it you will never again trust a single-JVM comparison.

**Root cause — two mechanisms, both from Topic 74.**

1. **Warm-up asymmetry.** The first case pays for JIT compilation of shared machinery —
   the `timeIt` harness, the lambda call sites, `List.iterator()`, the boxing paths. By
   the time the second case runs, all of that is compiled. The second position is
   structurally advantaged.
2. **Profile pollution, which is the deeper one.** `timeIt` takes a `Supplier`. On the
   first call, that call site sees exactly one implementation and C2 compiles it
   **monomorphically** — the target is inlined directly, with no dispatch. The second
   case introduces a second implementation at the same call site. C2's speculation is
   violated, an uncommon trap fires, the method is deoptimized and recompiled
   **bimorphically** — now with a type check and a branch on every call, forever. The
   second case runs in permanently worse code. Or the first case does, depending on which
   deoptimization happens when.

Same mechanism as Topic 74's drill, appearing inside your measurement.

**Prove it:**

```bash
java -XX:+UnlockDiagnosticVMOptions -XX:+PrintCompilation -XX:+TraceDeoptimization \
     -cp out com.orderflow.bench.SingleJvmComparison 2>&1 | grep -i -E "timeIt|deopt|not entrant"
```

| What you see | What it means |
|---|---|
| `made not entrant` on the harness method between your two cases | **Deoptimization confirmed.** The second case is running recompiled code. Your comparison is invalid. |
| A `TraceDeoptimization` entry citing a class-check or bimorphic reason | Profile pollution, exactly as described. |
| No deopt, but the answer still follows position | Warm-up asymmetry alone. Still invalid, and still fixed the same way. |
| No deopt and the answer follows the *code* when you swap | You got lucky in this instance. It is still not a safe methodology; the next benchmark will not be lucky. |

**Fix.** `@Fork(3)`. Not `@Fork(1)` — the point of three is that you also sample
between-JVM variance, which is the variance a single fork cannot see.

```java
@Fork(3)   // three fresh JVM processes per benchmark method
```

JMH runs **each `@Benchmark` method in its own forked JVM**, so `decimal` and
`minorUnits` never share a process at all. The pollution is not mitigated; it is
structurally impossible.

**The rule to carry away:** *if reordering changes the answer, you measured the JVM's
history, not your code.*

---

### Trap 3 — one warmup iteration

**Wrong approach.**

```java
@Warmup(iterations = 1, time = 100, timeUnit = TimeUnit.MILLISECONDS)
@Measurement(iterations = 1, time = 1, timeUnit = TimeUnit.SECONDS)
@Fork(1)
```

Or, more commonly, this on the command line because someone was in a hurry:

```bash
java -jar target/benchmarks.jar OrderTotalBenchmark -wi 1 -i 1 -f 1
```

**Exact symptom.** Three observable things, and you should look for all three:

1. JMH prints the individual iteration results during the run. **The first measurement
   iteration is dramatically slower than it would be later** — but with `-i 1` there is
   no later, so you never see it.
2. **Running it twice gives materially different answers**, in a way that a well-warmed
   benchmark does not.
3. With `-i 1` there is **no error bar to compute**, so JMH reports the error as `NaN` or
   omits it. *An error of `NaN` is the tool telling you the result is not a measurement.*

**Root cause.** 100 ms of warm-up is not enough for HotSpot to reach steady state. The
method may still be in the interpreter or in C1. C2 compilation happens on a background
thread; a compilation queued at 90 ms may install at 300 ms — inside your measurement
window. You are averaging across compilation tiers, and possibly across the installation
of a better compiled version mid-iteration.

**Prove it:**

```bash
# Watch the iteration-by-iteration numbers with generous warmup and measurement counts.
java -jar target/benchmarks.jar OrderTotalBenchmark -f 1 -wi 10 -i 10 -r 1s
```

**WHAT TO LOOK FOR:** JMH prints each iteration as `# Warmup Iteration   1: <n> ns/op`
and `Iteration   1: <n> ns/op`. Read the *sequence*, not the summary.

| What you see in the iteration sequence | What it means |
|---|---|
| Warm-up iterations 1–3 much slower, then flat from 4 onward | Normal HotSpot warm-up. Your `@Warmup` count must cover the sloped part with room to spare. |
| Still trending downward at the last warm-up iteration | **Not warm.** Increase `-wi` until the last few warm-up iterations are flat, then add a couple more. |
| Measurement iterations trending in either direction | Something is changing during measurement: a data structure growing, a cache filling, GC ramping, or a late C2 install. Diagnose before believing the mean. |
| One measurement iteration far off the others | Usually a GC pause landing inside the iteration. Run `-prof gc` and see whether the benchmark allocates enough to trigger collections. |
| Error reported as `NaN` | You used `-i 1`. There is no sample to compute an error from. This is not a result. |

**Fix.** Warm up until it is flat and then some. For a small pure method, five one-second
iterations is usually generous. For anything that touches a large data structure, or that
has a class-loading or first-call cost, more. And **look at the sequence** rather than
trusting a fixed number of iterations you copied from somewhere.

```java
@Warmup(iterations = 5, time = 1, timeUnit = TimeUnit.SECONDS)
@Measurement(iterations = 10, time = 1, timeUnit = TimeUnit.SECONDS)
@Fork(3)
```

**The rule to carry away:** *warm-up is not a ritual number; it is a condition you verify
by reading the iteration sequence.*

---

### Trap 4 — the input is a constant, so the benchmark measures returning a constant

**Wrong approach.**

```java
public class TaxBenchmark {
    private static final long SUBTOTAL = 12_499L;      // static final
    private static final int  TAX_BP   = 2000;

    @Benchmark
    public long tax() {
        return Math.round(SUBTOTAL * TAX_BP / 10_000.0d);
    }
}
```

**Exact symptom.** The result is at or below the harness floor — indistinguishable from
the `baseline` benchmark — and **it does not change when you change the arithmetic.**
Replace `Math.round` with a much more expensive computation and the number stays flat.

A second, sharper symptom: **two implementations that are obviously different report
identical figures.** If a `BigDecimal` path and a `long` path report the same cost to
three significant figures, neither is running.

**Root cause.** `static final` primitives are compile-time constants: `javac` may inline
them into the bytecode (Topic 76 — you can see this with `javap -c`), and C2 constant-
folds the entire expression regardless. The generated code returns a literal. There is no
arithmetic at run time.

This is not restricted to `static final`. C2 will also fold values it has *proved*
constant — a field never written after `<clinit>`, or a value the profile shows is always
the same.

**Prove it:**

```bash
# javac's half: is the constant already inlined into the bytecode?
javap -c -p target/classes/com/orderflow/bench/TaxBenchmark.class | sed -n '1,60p'
```

| What you see in `javap -c` | What it means |
|---|---|
| `ldc2_w` / `sipush` pushing your literal value where you wrote the field name | `javac` inlined the compile-time constant. C2 will fold the rest. |
| `getstatic` reading the field | `javac` did not inline it (non-primitive, or not a compile-time constant), but C2 can still fold it if it can prove constancy. |

**Fix.** Move every input into a `@State` object and read it through the state reference:

```java
@State(Scope.Benchmark)
public class TaxBenchmark {
    private long subtotal;
    private int  taxBasisPoints;

    @Setup(Level.Trial)
    public void setUp() {
        // computed at runtime; the compiler cannot see through this
        subtotal = 12_000L + new Random(20260828L).nextInt(1_000);
        taxBasisPoints = 2000;
    }

    @Benchmark
    public long tax() {
        return Math.round(subtotal * taxBasisPoints / 10_000.0d);
    }
}
```

**The rule to carry away:** *if the compiler can compute your answer, it will, and you
will time the answer rather than the computation.* `@State` is not boilerplate. It is the
defence.

---

### Trap 5 — reporting the mean, acting on a difference inside the error bars

**Wrong approach.** The output shows one variant with a slightly lower mean. Someone
writes in Slack: *"minor units is 4% faster, merging the rewrite."*

**Exact symptom — and this one is a reading failure, not a coding failure.** Look at the
error column. If the two intervals overlap, there is no established difference. The exact
thing to look for:

```
Benchmark                        (lineCount)  Mode  Cnt  Score   Error   Units
OrderTotalBenchmark.decimal                5  avgt   30  <n>   ± <n>   ns/op
OrderTotalBenchmark.minorUnits             5  avgt   30  <n>   ± <n>   ns/op
```

*illustration of the format, not captured output — the column structure only*

The columns, left to right:

| Column | Meaning | How you use it |
|---|---|---|
| `Benchmark` | Fully-qualified class + method | Identifies the case |
| `(lineCount)` | One column per `@Param` | **Read across params** — a fixed gap and a scaling gap mean different things |
| `Mode` | `avgt`, `thrpt`, `sample`, `ss` | `avgt` is time per op (lower is better); `thrpt` is ops per unit time (higher is better). **Mixing these up is a classic and embarrassing error.** |
| `Cnt` | Number of measurement iterations aggregated **across all forks** | With `-f 3 -i 10` you expect 30. If it is smaller, a fork failed — read the log. |
| `Score` | The mean | Never quote this alone |
| `Error` | Half-width of the confidence interval, by default 99.9% | **The column that decides whether you have a result** |
| `Units` | `ns/op`, `us/op`, `ops/s`, `B/op` for the GC profiler | Check it. `@OutputTimeUnit` sets it. |

**Root cause.** Confusing "a difference in the sample means" with "a difference between
the underlying values". A 4% gap with a ±6% error is a coin flip that landed one way.

**Fix — a three-step discipline:**

1. **Compare intervals, not means.** `scoreA + errorA` versus `scoreB − errorB`. If they
   overlap, say "not distinguishable".
2. **If they overlap and the decision matters, buy resolution**: more forks (`-f 5`), more
   iterations (`-i 20`), a quieter machine. Then re-run once, and accept that answer.
   Re-running until you like the result is p-hacking, and it is as dishonest in
   engineering as it is in science.
3. **Report both numbers, always.** "X ± Y" in every PR comment, every Slack message,
   every design doc. Make it a habit and you will be the person whose numbers people
   trust.

Also: **quote the run conditions with the number.** The `Cnt`, the fork count, the JDK
build, the machine, the flags. JMH prints all of it in the banner — capture the whole
output, not the last five lines.

**The rule to carry away:** *"faster" without an error is an opinion with a decimal point
in it.*

---

## Hands-on proof

Nine short exercises. Each one is a command plus what to look for. Run them in order;
each builds on the previous.

### Setup — get a JMH project, verify your versions

**Never invent tool versions.** Find out what you have:

```bash
java -version                      # JDK: expect 25 for this curriculum
mvn -version
mvn -q dependency:tree | grep -i jmh    # once the project exists
java -jar target/benchmarks.jar -h | head -5   # JMH prints its own version
```

Create a benchmark project with the official archetype:

```bash
mvn archetype:generate \
  -DinteractiveMode=false \
  -DarchetypeGroupId=org.openjdk.jmh \
  -DarchetypeArtifactId=jmh-java-benchmark-archetype \
  -DgroupId=com.orderflow.bench \
  -DartifactId=orderflow-bench \
  -Dversion=1.0-SNAPSHOT
```

> **Version note, flagged.** The archetype accepts `-DarchetypeVersion=<x.y>`. I am not
> going to tell you which version to pin, because JMH releases regularly and any number I
> write here will be wrong by the time you read it. Omit the parameter to take the latest
> released archetype, then look at the generated `pom.xml` to see which JMH version it
> chose, and pin it deliberately in your own repo. **Check what you got:**
>
> ```bash
> grep -A2 jmh.version orderflow-bench/pom.xml
> ```

Then set the `maven.compiler.release` to match your JDK, and confirm the annotation
processor is wired:

```bash
grep -n "jmh-generator-annprocess" orderflow-bench/pom.xml
```

**WHAT TO LOOK FOR:** a `provided`-scope dependency on `jmh-generator-annprocess`. If it
is missing, `@Benchmark` methods are silently ignored and your run will report "no
matching benchmarks". That specific error message almost always means the annotation
processor did not run.

### Proof 1 — the annotation processor really generates code

```bash
cd orderflow-bench
mvn -q clean verify
find target/generated-sources -name '*_jmhTest.java'
```

**WHAT TO LOOK FOR:** one generated file per benchmark class. Open one and find: the
`while (!control.isDone)` loop, the `blackhole.consume(...)` call around your method
invocation, and the `@Setup` guard flags. **This is the moment JMH stops being magic.**

| What you see | What it means |
|---|---|
| Generated files present, containing your method name | Correct setup. |
| `target/generated-sources` empty or missing | The annotation processor did not run. Check the `provided` dependency and that you are not using `-proc:none`. |
| A generated file per benchmark *mode* | Expected — throughput and average-time need different loops. |

### Proof 2 — forking is a real process

```bash
# Terminal 1
java -jar target/benchmarks.jar OrderTotalBenchmark -f 3 -wi 2 -i 2 -r 2s
# Terminal 2, while it runs
watch -n 1 'ps -ef | grep -c "[j]ava"'
```

**WHAT TO LOOK FOR:** the java process count rising to two and falling back to one, three
times. Plus, in terminal 1, three `# Fork: n of 3` headers each re-printing the JVM
banner.

### Proof 3 — see dead-code elimination happen and then stop happening

```java
@State(Scope.Benchmark)
public class DceProof {
    private long subtotal;

    @Setup public void setUp() { subtotal = 12_499L; }

    /** No consumption. Expect this to be at the floor. */
    @Benchmark
    public void discarded() {
        Math.round(subtotal * 2000 / 10_000.0d);
    }

    /** Consumed. Expect this to cost something. */
    @Benchmark
    public long returned() {
        return Math.round(subtotal * 2000 / 10_000.0d);
    }

    /** Consumed explicitly. Should match `returned` closely. */
    @Benchmark
    public void blackholed(Blackhole bh) {
        bh.consume(Math.round(subtotal * 2000 / 10_000.0d));
    }
}
```

```bash
java -jar target/benchmarks.jar DceProof -f 3
```

**WHAT TO LOOK FOR:** whether `discarded` is materially cheaper than `returned`.

| What you see | What it means |
|---|---|
| `discarded` far cheaper than `returned`/`blackholed` | **DCE demonstrated.** This is the whole document in one table row. |
| `discarded` ≈ `returned` | Either the computation is too cheap for the difference to show above the floor, or JMH's own generated code prevented elimination in this shape. Make the computation more expensive (a loop over the order lines) and re-run. |
| `blackholed` noticeably more expensive than `returned` | You are seeing the cost of the classic (non-compiler) `Blackhole`. Check the banner for blackhole mode. This is why you keep the number of `consume` calls equal across compared variants. |
| JMH prints a warning about the `discarded` benchmark | Newer JMH versions detect some dead-code patterns and warn. **Read the warnings**; they are written by people who have seen every mistake you are about to make. |

### Proof 4 — see constant folding, and defeat it

Run the Trap 4 pair: the `static final` version and the `@State` version, in the same
run. **WHAT TO LOOK FOR:** the `static final` version at the floor and unresponsive to
changes in the arithmetic; the `@State` version responsive.

### Proof 5 — establish your harness floor

Always have this benchmark in every class you write:

```java
@Benchmark
public void baseline(Blackhole bh) {
    bh.consume(lines);
}
```

```bash
java -jar target/benchmarks.jar '.*baseline.*' -f 3
```

**WHAT TO LOOK FOR:** a small, stable figure with a tight error. **This is your
measurement floor on this machine.** Write it down. Any benchmark result near it is not a
measurement of your code; it is a measurement of JMH.

### Proof 6 — the allocation profiler

```bash
java -jar target/benchmarks.jar OrderTotalBenchmark -prof gc
```

**WHAT TO LOOK FOR:** the `gc.alloc.rate.norm` rows — bytes per operation, normalised.

| What you see | What it means |
|---|---|
| A clean whole number like 16, 24, 32, 40 | Object headers and alignment (Topic 69). You can often identify the exact allocation from the size. |
| Zero for something you expected to allocate | Escape analysis / scalar replacement (Topic 75). Confirm with `-jvmArgs -XX:-DoEscapeAnalysis`. |
| A figure that scales linearly with a `@Param` | Per-element allocation. |
| A large figure independent of everything | Your `@Setup` is at the wrong `Level`, or the benchmark allocates a fresh collection each call. |

### Proof 7 — list the available profilers on your build

```bash
java -jar target/benchmarks.jar -lprof
```

**WHAT TO LOOK FOR:** which profilers are available and which report themselves as
unsupported on your platform. `gc` is available everywhere. `perfasm`, `perfnorm` and
`perf` need Linux and the `perf` tool; `perfasm` also needs `hsdis`. `stack` is a crude
sampling profiler that is useful for a sanity check but is not Topic 78's tool.

### Proof 8 — the modes, and why they answer different questions

```bash
java -jar target/benchmarks.jar OrderTotalBenchmark -bm avgt    # ns per op
java -jar target/benchmarks.jar OrderTotalBenchmark -bm thrpt   # ops per second
java -jar target/benchmarks.jar OrderTotalBenchmark -bm sample  # percentiles
java -jar target/benchmarks.jar OrderTotalBenchmark -bm ss      # single shot, cold
```

**WHAT TO LOOK FOR:**

| Mode | What it reports | When it is the right mode |
|---|---|---|
| `avgt` (AverageTime) | Mean time per operation | The default choice for comparing two implementations |
| `thrpt` (Throughput) | Operations per time unit | When you care about aggregate rate, especially with `@Threads` |
| `sample` (SampleTime) | p50/p90/p99/p999/max of individual operations | **When the tail matters** — the microbenchmark analogue of Topic 65's percentiles |
| `ss` (SingleShotTime) | One operation, cold | **Cold-start cost.** The right mode for asking "how long does the first call take", which is the question Topic 83 cares about |

`sample` mode is under-used and worth knowing. If two implementations have the same mean
but one has a much worse p999 — because it occasionally resizes, or occasionally
allocates enough to trigger a young collection — `avgt` will never show you that and
`sample` will.

### Proof 9 — `@Threads`, and the scaling question

```bash
java -jar target/benchmarks.jar OrderTotalBenchmark -t 1
java -jar target/benchmarks.jar OrderTotalBenchmark -t 4
java -jar target/benchmarks.jar OrderTotalBenchmark -t max
```

**WHAT TO LOOK FOR:** whether per-operation cost stays flat as threads increase.

| What you see | What it means |
|---|---|
| Flat per-op cost as threads rise | No shared state, no contention. Expected for a pure function over `Scope.Benchmark` read-only data. |
| Per-op cost rising with thread count | Contention, false sharing, or memory bandwidth saturation. This is exactly the shape Topics 95 and 96 are about. |
| Throughput rising sub-linearly then plateauing | Normal: you ran out of cores. Check `Runtime.availableProcessors()` inside a container (Topic 82) — it may be lower than you think. |
| Wildly unstable results at high thread counts | Your machine has fewer usable cores than `-t max` assumes, or the benchmark JVM is competing with the OS. Pin the count explicitly. |

---

## Failure drill

> **From the master plan, Section G:** *Naive `nanoTime` loop, then JMH. Capture both
> numbers. **The gap is the lesson.***

This drill has one deliverable: a short written explanation of why two numbers for the
same code differ, naming the mechanism for each part of the gap. Not the numbers
themselves — the explanation.

### The question under test

`orderflow`'s reconciliation export (Topic 73's job, Topic 18's string handling) builds a
CSV line per order. Two implementations:

```java
package com.orderflow.reporting;

public final class CsvLine {

    private CsvLine() {}

    /** What the code does today. */
    public static String withConcat(long orderId, String sku, int qty, long amountMinor) {
        return orderId + "," + sku + "," + qty + "," + amountMinor + "\n";
    }

    /** The proposed "optimisation". */
    public static String withBuilder(long orderId, String sku, int qty, long amountMinor) {
        StringBuilder sb = new StringBuilder(48);
        sb.append(orderId).append(',')
          .append(sku).append(',')
          .append(qty).append(',')
          .append(amountMinor).append('\n');
        return sb.toString();
    }
}
```

This is a deliberately good choice for the drill, because Topic 18 already told you the
expected answer (`+` on a single expression compiles to one `invokedynamic` bound to
`StringConcatFactory`, and is not obviously worse than a builder) and Topic 76 already
told you how to see that in `javap -c`. So you arrive with a prediction, which is the
right way to run an experiment.

### Step 0 — record your environment, before anything else

```bash
mkdir -p /docs/java/baselines/bench/77-drill
{
  java -version
  echo "---"
  uname -a
  echo "---"
  nproc 2>/dev/null || sysctl -n hw.ncpu
  echo "---"
  java -XX:+PrintFlagsFinal -version | grep -E "UseG1GC|UseZGC|MaxHeapSize|CICompilerCount"
} > /docs/java/baselines/bench/77-drill/environment.txt 2>&1
```

**Why first:** a benchmark number without its environment is not reusable and not
comparable. This is the same discipline as Topic 65's `environment.md`.

### Step 1 — write the naive benchmark, and make it as convincing as you can

Write it the way a competent engineer in a hurry writes it. Do not sabotage it — the
point is that a *reasonable* attempt is still wrong.

```java
package com.orderflow.bench;

import com.orderflow.reporting.CsvLine;

public class NaiveCsvBenchmark {

    private static final int ITERATIONS = 20_000_000;

    public static void main(String[] args) {
        String sku = "SKU-00042817";

        // A gesture at warm-up, which is what most people do.
        for (int i = 0; i < 100_000; i++) {
            CsvLine.withConcat(i, sku, 3, 12_499L);
            CsvLine.withBuilder(i, sku, 3, 12_499L);
        }

        long t0 = System.nanoTime();
        for (int i = 0; i < ITERATIONS; i++) {
            CsvLine.withConcat(i, sku, 3, 12_499L);
        }
        long t1 = System.nanoTime();

        long t2 = System.nanoTime();
        for (int i = 0; i < ITERATIONS; i++) {
            CsvLine.withBuilder(i, sku, 3, 12_499L);
        }
        long t3 = System.nanoTime();

        System.out.println("concat : " + (t1 - t0) / (double) ITERATIONS + " ns/op");
        System.out.println("builder: " + (t3 - t2) / (double) ITERATIONS + " ns/op");
    }
}
```

Run it and **write both numbers down** in the drill directory:

```bash
javac -d out $(find src -name '*.java')
java -cp out com.orderflow.bench.NaiveCsvBenchmark \
  | tee /docs/java/baselines/bench/77-drill/naive-run-1.txt
```

### Step 2 — establish that the naive number is absurd, with three checks

**Check A — does it respond to more work?**

Change `withConcat` to append a fifth field and re-run. A real measurement gets slower. If
it does not, work is being eliminated.

**Check B — does it respond to reordering?**

Swap the two timed blocks. Re-run.

```bash
java -cp out com.orderflow.bench.NaiveCsvBenchmarkSwapped \
  | tee /docs/java/baselines/bench/77-drill/naive-run-2-swapped.txt
```

**WHAT TO LOOK FOR:** whether the winner follows the code or the position.

**Check C — does the ratio survive the interpreter?**

```bash
java -Xint -cp out com.orderflow.bench.NaiveCsvBenchmark \
  | tee /docs/java/baselines/bench/77-drill/naive-run-3-xint.txt
```

**WHAT TO LOOK FOR:** the *ratio* between the two cases, not the absolute figures.

| Observation | Mechanism it implicates |
|---|---|
| Number does not change when you add a field | Dead-code elimination (both results are discarded) |
| Winner follows position, not code | Profile pollution / warm-up asymmetry |
| Ratio under `-Xint` very different from the ratio under the default | The JIT was reshaping one case substantially — you were measuring compilation, not code |
| Per-op figure below ~1 ns for a method that builds a String | Impossible. A `String` of ~30 characters cannot be allocated and populated in under a nanosecond. **Something was removed.** |
| The number changes materially between consecutive identical runs | Between-JVM variance, entirely invisible to this program |

A useful sanity anchor for the "is this absurd" judgement — and it is the only quantitative
anchor in this document, given as an order of magnitude rather than a measurement: on a
modern server core, one clock cycle is on the order of a third of a nanosecond, an L1 hit
is a few cycles, and a main-memory miss is on the order of a hundred nanoseconds. So a
figure below one nanosecond means "fewer than about three cycles", and no `String`
allocation happens in three cycles. That is how you know without measuring anything that
the number is a lie.

### Step 3 — port it to JMH, changing nothing about the code under test

```java
package com.orderflow.bench;

import com.orderflow.reporting.CsvLine;
import org.openjdk.jmh.annotations.*;
import org.openjdk.jmh.infra.Blackhole;

import java.util.concurrent.TimeUnit;

@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@State(Scope.Benchmark)
@Warmup(iterations = 5, time = 1, timeUnit = TimeUnit.SECONDS)
@Measurement(iterations = 10, time = 1, timeUnit = TimeUnit.SECONDS)
@Fork(3)
@Threads(1)
public class CsvLineBenchmark {

    private long orderId;
    private String sku;
    private int quantity;
    private long amountMinor;

    @Setup(Level.Trial)
    public void setUp() {
        // Runtime-computed: not constant-foldable.
        java.util.Random rnd = new java.util.Random(20260828L);
        orderId = 900_000L + rnd.nextInt(100_000);
        sku = "SKU-" + String.format("%08d", rnd.nextInt(100_000));
        quantity = 1 + rnd.nextInt(4);
        amountMinor = 199L + rnd.nextInt(48_000);
    }

    @Benchmark
    public void baseline(Blackhole bh) {
        bh.consume(orderId);
        bh.consume(sku);
    }

    @Benchmark
    public String concat() {
        return CsvLine.withConcat(orderId, sku, quantity, amountMinor);
    }

    @Benchmark
    public String builder() {
        return CsvLine.withBuilder(orderId, sku, quantity, amountMinor);
    }
}
```

```bash
mvn -q clean verify
java -jar target/benchmarks.jar CsvLineBenchmark -prof gc \
  | tee /docs/java/baselines/bench/77-drill/jmh-run-1.txt
```

### Step 4 — read the JMH output, in this order

1. **`baseline`.** Your floor.
2. **Errors.** Do `concat` and `builder` overlap?
3. **`gc.alloc.rate.norm`.** Both allocate a `String` and a backing array, so both should
   be non-zero. If the two differ, the difference is where the interesting story is.
4. **The banner.** JDK build, forks, threads, blackhole mode, JVM args. Keep it.

| What you see | What it means |
|---|---|
| `concat` and `builder` indistinguishable | The expected result given Topic 18. `+` on one expression compiles to `invokedynamic`/`StringConcatFactory`, which is at least as good as a manual builder. **The proposed "optimisation" is not one.** |
| `builder` clearly faster | Possible if the `StringConcatFactory` strategy on your JDK does something the pre-sized builder avoids. Investigate with `javap -c` (Topic 76) and `-prof gc` before accepting it. |
| `concat` clearly faster | Also plausible — `makeConcatWithConstants` can size the result exactly. |
| Both several times *more* expensive per op than the naive run reported | **This is the drill's payload.** The naive figure was smaller because part of the work was not being done. |
| Both roughly equal to the naive figures | Then the naive benchmark was not eliminated, and the drill's lesson is narrower: you still could not have *known* that without JMH, and the ordering check in Step 2B is the part that matters. Report honestly. |

### Step 5 — the deliverable: explain the gap

Write `/docs/java/baselines/bench/77-drill/gap-analysis.md`. Structure it exactly like
this, filling in your own numbers:

```markdown
# Topic 77 drill — the gap between the naive and JMH numbers

## Environment
(paste environment.txt)

## Numbers
| Measurement | concat | builder |
|---|---|---|
| Naive nanoTime loop        | <n> ns/op | <n> ns/op |
| Naive, cases swapped       | <n> ns/op | <n> ns/op |
| JMH avgt, 3 forks          | <n> ± <n> ns/op | <n> ± <n> ns/op |
| JMH gc.alloc.rate.norm     | <n> B/op | <n> B/op |
| JMH baseline (floor)       | <n> ± <n> ns/op | — |

## Accounting for the gap
For each component, state whether it applied here and what evidence says so.

1. Dead-code elimination — did the naive number respond to added work? (Step 2A)
2. Constant folding — were the inputs constants in the naive version? Were they in JMH?
3. Cold JIT — was 100k iterations of warm-up enough? What did the JMH warm-up
   sequence look like before it flattened?
4. On-stack replacement — the naive loop was entered once. Was there a `%` entry in
   PrintCompilation? (Step 6, optional)
5. Profile pollution / ordering — did the winner follow position? (Step 2B)

## Conclusion
One paragraph. Which implementation, if either, is faster, by how much, with what
error, on what machine. And then the sentence that matters:
"Against our GET /orders p50 baseline of <n> ms, this difference is <n>% of one
request, which is below what our load-test gate can resolve."
```

### Step 6 — optional but instructive: catch the OSR

```bash
java -XX:+UnlockDiagnosticVMOptions -XX:+PrintCompilation \
     -cp out com.orderflow.bench.NaiveCsvBenchmark 2>&1 \
  | grep -E "NaiveCsvBenchmark|CsvLine" \
  | tee /docs/java/baselines/bench/77-drill/naive-compilation.txt
```

**WHAT TO LOOK FOR:** a `%` in the flags column against `main`. That is an OSR
compilation of your timing loop — Lie 3, caught in the act.

### What the drill proves

Four things, and you should be able to state each in one sentence:

1. **The naive number and the JMH number describe different things.** One describes what
   the compiler did to a loop; the other describes the cost of an operation.
2. **The naive method cannot detect its own failure.** It produces a number in every case,
   including the cases where it measured nothing. The only way to catch it is to run the
   checks in Step 2 — which nobody does, which is why hand-rolled benchmarks persist.
3. **Ordering is a variable in a single-JVM comparison.** Once you have seen the winner
   follow the position, `@Fork(3)` stops being a cargo-culted annotation.
4. **The correct engineering conclusion is often "this does not matter."** That is not a
   failed experiment. It is a successful one that saved a rewrite.

---

## Measurement

**This section is the core of the document.** Everything above explains why naive
measurement fails. This is the correct harness, annotation by annotation, with the
question *"what does this defend against?"* answered for every single one — followed by
the five rules that turn a benchmark into evidence.

### The complete harness, for a real `orderflow` question

**The question:** the catalogue endpoint is 70% of `orderflow`'s traffic. After Postgres
returns a page of products, the service applies an in-memory availability filter and a
relevance sort before serialising. Two candidate implementations exist. Which is cheaper
per call, at the page sizes we actually serve?

```java
package com.orderflow.bench;

import com.orderflow.catalog.Product;
import com.orderflow.catalog.ProductRanker;
import org.openjdk.jmh.annotations.*;
import org.openjdk.jmh.infra.Blackhole;
import org.openjdk.jmh.runner.Runner;
import org.openjdk.jmh.runner.RunnerException;
import org.openjdk.jmh.runner.options.Options;
import org.openjdk.jmh.runner.options.OptionsBuilder;

import java.util.ArrayList;
import java.util.List;
import java.util.Random;
import java.util.concurrent.TimeUnit;

@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.MICROSECONDS)
@State(Scope.Benchmark)
@Warmup(iterations = 5, time = 1, timeUnit = TimeUnit.SECONDS)
@Measurement(iterations = 10, time = 1, timeUnit = TimeUnit.SECONDS)
@Fork(value = 3, jvmArgs = {
        "-Xms1200m", "-Xmx1200m",           // match the container heap from Topic 65
        "-XX:+UseG1GC",                     // match the production collector
        "-XX:+AlwaysPreTouch"               // remove first-touch page faults from the measurement
})
@Threads(1)
public class ProductRankingBenchmark {

    /**
     * Page sizes orderflow actually serves, from the Topic 65 k6 scenario:
     * default page 20, "load more" page 50, admin export page 200.
     * Nothing here is a round number chosen for aesthetics.
     */
    @Param({"20", "50", "200"})
    public int pageSize;

    /** Matches the seeded dataset's skew: a few hot products, a long tail. */
    @Param({"0.10", "0.60"})
    public double outOfStockFraction;

    private List<Product> page;
    private ProductRanker streamRanker;
    private ProductRanker loopRanker;

    @Setup(Level.Trial)
    public void setUpTrial() {
        Random rnd = new Random(20260828L);          // fixed seed: reproducible
        page = new ArrayList<>(pageSize);
        for (int i = 0; i < pageSize; i++) {
            page.add(new Product(
                    "SKU-" + String.format("%08d", rnd.nextInt(100_000)),
                    199L + rnd.nextInt(48_000),
                    rnd.nextDouble() >= outOfStockFraction,
                    rnd.nextDouble()
            ));
        }
        streamRanker = ProductRanker.streamBased();
        loopRanker = ProductRanker.loopBased();
    }

    /** MANDATORY BASELINE. The harness floor on this machine, at this @Param. */
    @Benchmark
    public void baseline(Blackhole bh) {
        bh.consume(page);
        bh.consume(pageSize);
    }

    @Benchmark
    public List<Product> streamPipeline() {
        return streamRanker.rank(page);
    }

    @Benchmark
    public List<Product> handWrittenLoop() {
        return loopRanker.rank(page);
    }

    /** Programmatic runner: lets CI run this with exactly the options we intend. */
    public static void main(String[] args) throws RunnerException {
        Options opt = new OptionsBuilder()
                .include(ProductRankingBenchmark.class.getSimpleName())
                .resultFormat(org.openjdk.jmh.results.format.ResultFormatType.JSON)
                .result("target/product-ranking.json")
                .addProfiler("gc")
                .build();
        new Runner(opt).run();
    }
}
```

### Every annotation, and what it DEFENDS AGAINST

This table is the reason this section exists. Learn it as a list of defences, not as a
list of options.

| Annotation | What it does | **What it defends against** | What goes wrong without it |
|---|---|---|---|
| `@Benchmark` | Marks the method for the annotation processor | Nothing directly — it is the trigger | Without it the processor generates nothing and JMH reports "no matching benchmarks" |
| `@BenchmarkMode(Mode.AverageTime)` | Reports mean time per operation | **Answering the wrong question.** `Throughput` and `AverageTime` are reciprocals but read oppositely — lower is better for one, higher for the other | Someone reads a `thrpt` table as if it were `avgt` and concludes the slower variant won. This happens in real code reviews. |
| `@OutputTimeUnit(TimeUnit.MICROSECONDS)` | Fixes the reported unit | **Unit confusion and false precision.** A 200-element sort reported in `ns/op` is a six-digit number nobody can compare at a glance | Cross-`@Param` comparison becomes error-prone; results in JSON are ambiguous |
| `@State(Scope.Benchmark)` | One instance, shared by all threads, fields read through a reference | **Constant folding (Lie 2).** The compiler cannot prove field values constant | C2 folds the computation to a literal; every variant reports the same figure |
| `@Setup(Level.Trial)` | Runs once per fork, before warm-up | **Setup cost leaking into the measurement**, and non-determinism | `Level.Invocation` puts data construction inside the timed region and you benchmark your own test fixture |
| `@TearDown` | Cleanup, at the matching level | Resource leaks across iterations | Not needed here; needed when a benchmark opens files, sockets or pools |
| `Blackhole` parameter / returning a value | Makes the result observable | **Dead-code elimination (Lie 1).** The compiler cannot prove the work unobserved | The benchmark measures an empty loop and reports a sub-nanosecond figure |
| `@Warmup(iterations = 5, time = 1s)` | Five discarded seconds of execution | **Cold JIT (Lie 4).** Lets HotSpot reach steady state before recording | You average interpreter, C1 and C2 into one meaningless number |
| `@Measurement(iterations = 10, time = 1s)` | Ten recorded one-second iterations | **A result with no error.** Ten samples give the statistics something to work with | With `-i 1` the error is `NaN` and you have a point, not a measurement |
| `@Fork(3)` | Three separate JVM processes per benchmark method | **Profile pollution (Lie 5) AND between-JVM variance.** Nothing carries between cases; three JVMs sample layout and compilation-order luck | With one fork you cannot tell "A is faster" from "this JVM laid out A well"; with zero forks the cases contaminate each other outright |
| `jvmArgs` on `@Fork` | Applies production flags to every forked JVM | **Measuring a JVM you do not deploy.** Your service runs G1 in a 1200 MB heap; a default-flag benchmark JVM may pick a different collector and a different heap | Your benchmark's GC behaviour, escape analysis and TLAB sizing differ from production |
| `-XX:+AlwaysPreTouch` in `jvmArgs` | Faults in the whole heap at startup | **First-touch page faults appearing as latency spikes** in early iterations | Warm-up looks noisy for reasons that have nothing to do with your code |
| `@Threads(1)` | One benchmark thread | **Accidental contention, and accidental absence of it.** Being explicit makes the thread count part of the recorded result | `@Threads(Threads.MAX)` on a `Scope.Benchmark` mutable state silently measures contention you did not intend |
| `@Param({...})` | Runs the whole matrix of values | **Benchmarking one arbitrary size and generalising.** Also exposes whether a cost is fixed or per-element | You report a result at N=1,000,000 for a system that serves N=20 |
| The `baseline` benchmark | Measures the harness itself | **Reporting noise as a result.** Anything near the floor is unmeasurable | You confidently report a difference that is smaller than JMH's own overhead |

Two more you will need soon, listed here because they belong with the defences:

| Annotation | What it defends against |
|---|---|
| `@CompilerControl(CompilerControl.Mode.DONT_INLINE)` | **Inlining collapsing the thing you meant to measure.** When you want to measure a call, and the compiler inlines it away, the call disappears. Use sparingly and deliberately — it also makes the benchmark unrepresentative of production, where inlining does happen. |
| `@Group` / `@GroupThreads` | **Symmetric benchmarks hiding asymmetric contention.** Producer/consumer and reader/writer workloads need different thread counts per role (Topics 94, 95, 96). |

### The five rules

These are not style preferences. Each one is the fix for a specific way engineers get
benchmarks wrong, and each has cost real teams real weeks.

---

**Rule 1 — Always benchmark a baseline.**

Every benchmark class gets a `baseline` method that does the minimum possible: consume
the state, return. That figure is your **measurement floor** on that machine, with those
flags, at that `@Param`.

Why it is mandatory: without it you have no idea whether a difference you are looking at
is above the noise. The floor moves with your machine, your JDK, your blackhole mode and
your thread count. It is not a constant you can look up; it is something you measure
alongside every result.

Second, related discipline: **benchmark the thing you are replacing, not just the
replacement.** "The new implementation takes X" is not a result. "The new implementation
takes X, the old one takes Y, both ± their errors, floor is Z" is.

---

**Rule 2 — Report error bars, never the mean alone.**

The professional format, which you should use in PRs, Slack, and design docs:

> *"`handWrittenLoop` is faster than `streamPipeline` by `<n> ± <n>` µs/op at
> `pageSize=50` (JMH `avgt`, 3 forks × 10 iterations, JDK `<build>`, `<machine>`)."*

If the intervals overlap, the sentence is:

> *"No measurable difference between `handWrittenLoop` and `streamPipeline` at
> `pageSize=50` (both `<n> ± <n>` µs/op, 3 forks). The difference, if any, is below our
> measurement resolution."*

That second sentence is a *result*. Say it with the same confidence as the first.

Never do these:
- Quote a mean with no error.
- Quote a percentage improvement computed from two means whose errors you did not check.
- Re-run until the answer is the one you wanted, then report that run.
- Compare a `Score` from one JMH invocation to a `Score` from a different invocation on a
  different day without checking that the baseline moved by less than the effect.

---

**Rule 3 — Never compare across machines.**

A JMH number is a fact about **one machine, one JDK build, one set of flags, one CPU
frequency-governor state, one thermal condition**. It is not a property of the code.

The failure modes this rule prevents, all of which are common:

| The mistake | Why it produces a wrong answer |
|---|---|
| Running variant A on your laptop and variant B on the CI runner | Different core counts, different cache sizes, different governors. Any difference you see is confounded. |
| Comparing today's run against a number in a wiki page from last year | Different JDK, different flags, probably different hardware. |
| Benchmarking on a shared CI runner at all | Noisy neighbours. Errors will be huge, and worse, they will be *asymmetrically* huge depending on what else ran. |
| Benchmarking a laptop on battery | Aggressive frequency scaling. The measurement changes based on your power settings. |
| Benchmarking with a browser, an IDE and a Docker daemon running | Cache pollution and scheduler interference. |

**The rule that follows:** if two numbers are to be compared, they must come from **the
same JMH invocation, or two invocations on the same machine in the same session with the
baseline benchmark confirming the floor has not moved.** Put both variants in one
benchmark class. That is the whole reason JMH lets you.

For `orderflow` specifically: benchmark on a machine whose CPU generation matches your
production node type as closely as you can get, and **record the machine in the result
file**. If your production nodes are ARM and you benchmark on x86, say so loudly in the
PR; the answer may not transfer at all.

---

**Rule 4 — Benchmark at the sizes your system actually uses.**

`orderflow` serves 20-, 50- and 200-product pages and computes totals over ~5 order
lines. A benchmark at N=1,000,000 answers a question your system never asks, and it
answers it in a regime — one where the data no longer fits in L2, where allocation
triggers collections, where branch prediction behaves differently — that has nothing to
do with your production behaviour.

Get the real distribution from the Topic 65 dataset:

```sql
-- run against the seeded orderflow Postgres
SELECT percentile_disc(0.5)  WITHIN GROUP (ORDER BY line_count) AS p50,
       percentile_disc(0.95) WITHIN GROUP (ORDER BY line_count) AS p95,
       percentile_disc(0.99) WITHIN GROUP (ORDER BY line_count) AS p99,
       max(line_count) AS max
FROM (SELECT order_id, count(*) AS line_count FROM order_line GROUP BY order_id) t;
```

Then set `@Param` to those values. Your benchmark now describes your system.

---

**Rule 5 — A JMH result is the middle step, never the whole argument.**

The complete sequence, and you should say it in exactly this shape in an interview:

1. **Topic 78 — profile the running service under the Topic 65 load.** Find out where the
   time actually goes. Do this *first*. A microbenchmark chosen without a profile is a
   guess wearing a lab coat.
2. **Topic 77 — this document.** Microbenchmark the specific method the profile named, in
   isolation, with error bars.
3. **Topic 65 — re-run the load baseline.** Prove the system-level percentiles moved by
   more than the ±10% gate tolerance.

If step 3 cannot detect your change, you have not made a performance improvement. You have
made a change.

### Getting the JMH project into `orderflow`

The archetype creates a standalone project, which is the right starting point. For
`orderflow` you then want a `orderflow-bench` module that depends on the service's
production code but is **never** on the production classpath:

```xml
<!-- orderflow-bench/pom.xml — the shape, not a complete file -->
<dependencies>
  <dependency>
    <groupId>com.orderflow</groupId>
    <artifactId>orderflow-core</artifactId>
    <version>${project.version}</version>
  </dependency>
  <dependency>
    <groupId>org.openjdk.jmh</groupId>
    <artifactId>jmh-core</artifactId>
    <version>${jmh.version}</version>
  </dependency>
  <dependency>
    <groupId>org.openjdk.jmh</groupId>
    <artifactId>jmh-generator-annprocess</artifactId>
    <version>${jmh.version}</version>
    <scope>provided</scope>
  </dependency>
</dependencies>
```

Two things to check, because both are common failure modes (Topics 31, 32):

```bash
# 1. The annotation processor is on the compile path but NOT on the runtime path.
mvn -q dependency:tree -Dincludes=org.openjdk.jmh

# 2. The benchmark module is not accidentally a dependency of the service.
mvn -q dependency:tree -pl orderflow-app | grep -i bench   # expect no output
```

### What is safe to run in CI, and what is not

| Practice | Verdict |
|---|---|
| Running JMH on a shared CI runner and failing the build on a regression threshold | **No.** The noise floor on shared runners is larger than most regressions. You will get flaky builds and the team will disable the check. |
| Running JMH on a dedicated, pinned, otherwise-idle benchmark machine, nightly, storing JSON, and alerting on a trend across many runs | **Yes.** This is the only version of "benchmark in CI" that works. |
| Running JMH on a developer laptop during a code review, and pasting the full output including the banner | **Yes**, and it is the highest-value habit in this document. |
| Attaching JMH to a running production service | **Not a thing.** JMH runs benchmarks; it does not observe a service. You want Topic 78. |
| Using JMH's `stack` profiler as your profiler | **No.** It is a crude sanity check. Topic 78's tools are the real answer. |

---

## Practice exercises

### 1 — Easy: build your own defence sheet, and find your floor

**Goal:** internalise the annotation-to-defence mapping, and learn your own machine's
measurement floor so that you can recognise a meaningless result on sight.

**Do this:**

1. Create the JMH project with the archetype command from Hands-on Setup. Record the JMH
   version your archetype chose.
2. Write a single class with exactly four benchmarks against `orderflow`'s `Product`:
   - `baseline` — consume the state and return.
   - `hashCodeOnly` — return `product.sku().hashCode()`.
   - `equalsOnly` — return `product.equals(otherProduct)`.
   - `toStringFull` — return `product.toString()`.
3. Run with `-f 3 -prof gc`.
4. Write down, in a file you keep: your floor from `baseline`, and the
   `gc.alloc.rate.norm` for each of the four.
5. Now deliberately break it: remove the `return` from `hashCodeOnly` so the result is
   discarded, and change `@State` fields to `static final`. Re-run.

**Deliverable:** a one-page table with two columns — *annotation removed* and *observed
symptom* — with a row for each of `@State`, the return value, `@Fork(3)`, and a
sufficient `@Warmup`. Written from your own runs, not from this document.

**You have succeeded when:** you can look at any benchmark result and immediately ask
"what is the floor here?" without being prompted.

---

### 2 — Medium: settle the `ArrayList` versus `LinkedList` claim properly
*(combines Topics 01, 06, 10, 11, 12, 21, 23, 25, 68, 69, 74, 75, 76)*

Topic 11 makes a strong claim: `LinkedList` loses to `ArrayList` even for middle
insertion at realistic sizes, because of cache lines and pointer chasing rather than
Big-O. **That claim is exactly the kind of thing that must not be taken on faith, and
this document exists so you can settle it yourself.**

**Do this:**

1. Write `ListInsertionBenchmark` with `@Param` sizes drawn from `orderflow` reality —
   the order-line list (`@Param({"5", "25"})`) and the catalogue page
   (`@Param({"20", "200"})`) — plus one deliberately large size (`@Param({"10000"})`) so
   you can see where, if anywhere, the crossover is.
2. Benchmark four operations on both `ArrayList` and `LinkedList`:
   - insert at the middle index,
   - `get` at the middle index,
   - full iteration summing a field,
   - `contains` for an element that is not present.
3. Use `@Setup(Level.Invocation)` for the *insertion* benchmarks only — and **write down
   why** that is necessary here and dangerous everywhere else. (Hint: an insertion
   mutates the structure, so without per-invocation reset the list grows during the
   measurement and you benchmark an ever-larger list. `Level.Invocation` puts the reset
   inside the timed region, which biases the result; JMH warns about this explicitly.
   Read the warning and account for it — the honest alternative is to use
   `Mode.SingleShotTime` with a pre-built list, or to amortise a fixed number of
   insertions per invocation with `@OperationsPerInvocation`.)
4. Run with `-prof gc`. Explain any allocation difference in terms of Topic 69's object
   layout — a `LinkedList` node is a separate object with a header and three references.
5. Cross-check with `javap -c` (Topic 76) that your iteration benchmark is doing what you
   think — that the enhanced `for` loop over a `List` is calling `iterator()` and not
   something else.
6. Check with `-XX:+PrintInlining` or `-jvmArgs -XX:-DoEscapeAnalysis` (Topic 75) whether
   the iterator is being scalar-replaced.

**Deliverable:** a markdown file that either **confirms or refutes Topic 11's claim on
your hardware**, with error bars, at each `@Param` size, and a paragraph explaining the
mechanism you believe is responsible. Include the case where the claim is *wrong*, if you
find one — a real result that contradicts the curriculum is worth more than one that
agrees with it.

**You have succeeded when:** you can defend the answer against someone quoting Big-O at
you, using your own numbers and a cache-line argument.

---

### 3 — Hard: the production simulation — profile, benchmark, prove
*(against the Topic 65 baseline; combines Topics 25, 50, 65, 68, 70, 71, 74, 77, 78)*

**The scenario.** `orderflow` is running under the Topic 65 load in its container (2
vCPU, 2 GiB, `-Xmx1200m`, G1). Product management wants the catalogue endpoint's p99
reduced by 20%. A colleague proposes adding `.parallel()` to the in-memory ranking
pipeline, citing Topic 25's stream material. You are the reviewer.

**Do this, in this order, and do not skip step 1:**

1. **Profile first.** Run the Topic 65 load and take a wall-clock profile of the catalogue
   path (Topic 78). Determine what fraction of the p99 is CPU in the ranking pipeline
   versus waiting on Postgres versus serialisation versus GC. **Write the fraction down
   before you benchmark anything.**
2. **Compute the Amdahl ceiling.** If ranking is F% of the p99, then even making it
   infinitely fast improves p99 by at most F%. Compare F to the requested 20%. If F < 20,
   the proposal cannot succeed *even if it works perfectly*, and that is your review
   comment. Write it before doing any more work.
3. **Benchmark anyway, correctly**, because the review needs evidence and because you
   want to know what `.parallel()` actually does here. Write `RankingParallelBenchmark`:
   - `@Param` page sizes 20, 50, 200 (the real ones).
   - Three variants: sequential stream, parallel stream, hand-written loop.
   - `@Fork(3)`, `jvmArgs` matching the container (`-Xmx1200m -XX:+UseG1GC`).
   - **Critically:** add a fork variant with `-XX:ActiveProcessorCount=2` to match the
     container's CPU allocation (Topic 82). `ForkJoinPool.commonPool()` is sized from
     `availableProcessors() - 1`, so in a 2-vCPU container the common pool has **one**
     worker thread, and `.parallel()` therefore has almost nothing to parallelise onto.
     This is the finding that will decide the review.
   - `-prof gc` to see the splitting and merging allocation.
4. **Run `@Threads(2)` as well**, because in production the ranking runs concurrently
   across request threads. A parallel stream that helps at one thread can be catastrophic
   at N, because every request contends for the same common pool (Topic 25's drill).
5. **If, and only if, the benchmark shows a real win**, implement it and re-run the Topic
   65 baseline. Compare against `/docs/java/baselines/` on every percentile, and check
   that you have not regressed something else — GC pause count and allocation rate
   (Topics 70, 71) are the usual casualties of a parallel stream.
6. **Write the review.**

**Deliverable:** a PR review comment containing, in order: the profile fraction, the
Amdahl ceiling, the benchmark result with error bars at each page size and thread count,
the `ActiveProcessorCount=2` result, and a recommendation. The recommendation may well be
"no", and if it is, it must also say **what would work instead** — which, given the
profile, is probably a Topic 50 query fix or a Topic 110 cache, not a stream change.

**You have succeeded when:** the review would be persuasive to someone who disagrees with
you, because it contains measurements rather than assertions.

---

## Interview questions

### Q1 — "I timed it in a loop and it took 3 nanoseconds. Nice, right?"

**MID-LEVEL answer:** *"Yes, that's fast. Maybe add a warm-up loop first to let the JIT
kick in."*

Not wrong about warm-up, but it accepts the number. That is the failure.

**SENIOR answer:** *"Three nanoseconds is about the cost of nothing — roughly ten clock
cycles on a modern core, less than an L2 hit. Unless the method is a field read, that
number means the work was eliminated. The first thing I'd do is add work to the method
and re-run: a real measurement responds, a dead-code-eliminated one doesn't. Then I'd
check whether the inputs are `static final`, because constant folding produces the same
symptom.*

*Then I'd port it to JMH. Not because JMH is a better timer — `nanoTime` is fine — but
because JMH defends against the four specific things that break a hand-rolled loop:
dead-code elimination, constant folding, on-stack replacement, and measuring across
compilation tiers. It consumes results through a `Blackhole`, holds inputs in a `@State`
object, warms up until the compiler settles, and forks a fresh JVM per trial.*

*And I'd ask the question before all of that: what fraction of a request is this method?
If it's 0.1%, the correct outcome is that we don't optimise it, whatever the number
says."*

**What separates them:** the mid-level engineer treats warm-up as the only problem. The
senior engineer knows the number is *impossible* and can say why from first principles —
cycle counts — before touching a tool. And they end on Amdahl, not on the benchmark.

**Follow-up:** *"You add the `Blackhole` and the number becomes 40 ns. Is that the answer
now?"*
→ *"It's a candidate. I'd still want three forks, because a single JVM can't distinguish
the code from the JVM's compilation luck; I'd want the error bar; I'd want a `baseline`
benchmark to know the floor; and I'd want `-prof gc`, because if this thing allocates,
the per-call time isn't the whole cost — the GC pressure shows up somewhere else
entirely."*

---

### Q2 — "Why does JMH fork a new JVM? And why would you ever need more than one fork?"

**MID-LEVEL answer:** *"To get a clean environment so previous runs don't affect the
result."*

Directionally right, mechanically empty. It does not survive a follow-up.

**SENIOR answer:** *"Two different reasons, and the second one is the one people miss.*

*The first is profile pollution. C2 compiles against the profile it observed. If I run
implementation A a million times, the shared call sites in my harness become monomorphic
and get inlined. Then B runs at the same call sites, the speculation is violated, an
uncommon trap fires, and the method is deoptimized and recompiled bimorphically — with a
type check on every call from then on. So B runs in permanently worse code, or A does,
depending on ordering. The observable symptom is that swapping the two cases changes the
answer. A fresh process has no profile, no class-hierarchy assumptions, no code cache and
no heap history.*

*The second reason is between-JVM variance, and it's why one fork isn't enough. Two JVMs
running identical bytecode can differ measurably because of code layout in the code cache,
compilation ordering, and address-space randomisation affecting cache-set conflicts. One
fork samples the variance within a JVM, which is small. Three forks sample the variance
between JVMs, which is often larger than the effect you're chasing. With `-f 1` you cannot
distinguish 'A is faster' from 'this JVM happened to lay out A well'."*

**What separates them:** naming deoptimization and class-hierarchy analysis as the
mechanism, and — the real differentiator — knowing that multiple forks exist to sample a
*different source of variance*, not merely to run the same thing again.

**Follow-up:** *"JMH has `-f 0`. When would you use it?"*
→ *"Only to attach a debugger to a benchmark, because with no fork it runs in the harness
JVM. Any number produced with `-f 0` is not a result, and JMH warns you about it. I'd
never quote one."*

---

### Q3 — "Your benchmark says the new implementation is 4% faster. Do we ship the rewrite?"

**MID-LEVEL answer:** *"4% is a decent win, so yes — assuming the tests pass."*

**SENIOR answer:** *"Three questions before I'd answer.*

*One: what's the error? If it's ±6%, there's no established difference and 4% is a coin
flip. I'd compare the intervals, not the means. If they overlap I'd raise to five forks
and run it once more — once, not until I like the answer.*

*Two: 4% of what? If it's 4% of a method that's 0.2% of the request, we're talking about
eight parts per hundred thousand. Our Topic 65 load-test gate resolves ±10% at the
endpoint level, so we couldn't demonstrate this change even if it were real. I'd want the
profile fraction before the benchmark, not after.*

*Three: what does the rewrite cost? If it turns readable code into an unreadable
optimisation, and the win is unmeasurable at the system level, then the change has a
definite cost and an undetectable benefit. That's a bad trade regardless of the
benchmark.*

*So my answer is probably no — and the useful thing I'd add to the PR is what I think
would actually move the number, which given our profile is the N+1 on the order-lines
query, not this."*

**What separates them:** the mid-level engineer treats a percentage as a fact. The senior
engineer treats it as an estimate with an uncertainty, contextualises it against the
system, and prices the maintenance cost of the change. Note that the senior answer ends
by redirecting to the real problem.

**Follow-up:** *"What if the win were 40% instead of 4%?"*
→ *"Same three questions, different answers. 40% with a ±3% error is a real effect. But
40% of a method that's 0.2% of the request is still 0.08% of the request. The magnitude
of the microbenchmark win never changes the Amdahl arithmetic — it just makes it more
tempting to skip it."*

---

### Q4 — "How would you benchmark whether `ArrayList` or `LinkedList` is better for our order-line list?"

**MID-LEVEL answer:** *"Write a loop that inserts a million elements into each and time
both."*

**SENIOR answer:** *"First I'd push back on the question: our orders average five lines,
with a p99 around twenty-five. At those sizes almost nothing distinguishes the two, and
the interesting answer is likely 'it doesn't matter, use `ArrayList` because it's the
default and it's simpler.' So I'd benchmark at the sizes we actually have — 5, 25, and
one large size like 10,000 purely to find where a crossover exists, if it does.*

*Then the mechanics. `@Param` for the sizes. Both implementations in one class so they
share a JMH invocation, because comparing across invocations or machines isn't valid.
`@Fork(3)`. A `baseline` benchmark so I know the floor, which at N=5 I'd expect the whole
operation to be close to. `-prof gc`, because a `LinkedList` node is a separate heap
object with a header and three references, and the allocation difference may matter more
than the time difference for GC pressure.*

*The insertion benchmark is the tricky one: inserting mutates the list, so without a reset
the structure grows during measurement. `@Setup(Level.Invocation)` resets it but puts the
reset inside the timed region, which biases the result — JMH warns about exactly this. The
cleaner options are `Mode.SingleShotTime` on a pre-built list, or a fixed number of
insertions per invocation with `@OperationsPerInvocation`.*

*And I'd expect `ArrayList` to win even for middle insertion at realistic sizes, because
`System.arraycopy` on a contiguous block is fast and a `LinkedList` traversal is a chain
of likely cache misses. But that's a prediction, and the point of the benchmark is that
I might be wrong on this hardware."*

**What separates them:** questioning the premise, choosing sizes from the production
dataset, knowing the `Level.Invocation` trap for mutating benchmarks, and having a
mechanistic prediction that the benchmark can falsify.

**Follow-up:** *"Your benchmark says they're identical at N=25. What do you write in the
PR?"*
→ *"'No measurable difference at our production sizes (both `<n> ± <n>` ns/op, 3 forks,
floor `<n>`). Use `ArrayList` — same performance, simpler, better `get`, and less
allocation per element.' The performance result is neutral, so the decision falls to the
non-performance criteria, which is exactly how it should be."*

---

### Q5 — "Can you use JMH to benchmark our `OrderService.placeOrder` method?"

**MID-LEVEL answer:** *"Yes — spin up the Spring context in `@Setup` and call the method
in the benchmark."*

This is a trap and the trap is subtle, because the answer is technically possible.

**SENIOR answer:** *"You can make it run, but it wouldn't be a microbenchmark and I
wouldn't trust the number. `placeOrder` is `@Transactional`: it takes a HikariCP
connection, does several Postgres round trips, debits a wallet with optimistic locking,
and publishes an event. The dominant term is network and database time. JMH would give
me a precise, well-warmed, three-fork measurement of my database's mood on the day I ran
it.*

*Worse, it would be actively misleading in a specific way. JMH runs a tight loop with no
think time and no concurrency model, so it exercises the connection pool completely
unlike production. Contention behaviour — which is where the real problems are, per
Topics 52 and 55 — would look nothing like reality.*

*The right tool for a service method under realistic conditions is Topic 65: k6 against
the containerised stack with an open-model arrival rate, measuring percentiles per
endpoint. If I then want to know why it's slow, that's Topic 78 — profile it under that
load. And if the profile names a specific pure computation inside `placeOrder`, *that* is
what I'd bring to JMH.*

*There's one legitimate exception: `Mode.SingleShotTime` on a cold Spring context is a
reasonable way to measure application startup cost, which is a real question for Topic 83
and for scale-to-zero deployments. But that's a different question from throughput."*

**What separates them:** knowing the tool's domain, and — the part that impresses — being
specific about *how* the wrong tool would mislead, rather than just saying "use a load
test". The `SingleShotTime` exception shows genuine familiarity rather than a memorised
rule.

**Follow-up:** *"So what's the smallest thing in `placeOrder` you would microbenchmark?"*
→ *"Whatever the wall-clock profile says is CPU-bound and significant. Realistically:
order-total computation, the validation pass, DTO mapping, or JSON serialisation. Each of
those is a pure function of in-memory data, which is JMH's home ground. And I'd only
benchmark the one the profile named — picking one without a profile is guessing."*

---

## Mental model checkpoint

Answer these without looking anything up. If you cannot, re-read the named section.

1. **Name the four ways a naive `nanoTime` loop lies, plus the fifth thing that goes
   wrong only when you compare two implementations.** For each, name the JMH mechanism
   that defends against it.
   *(Mechanical statement; Measurement — the defence table.)*

2. **You see a benchmark reporting 0.4 ns/op. Without running anything, why do you know
   it is wrong?** What single change to the code under test would confirm it in thirty
   seconds?
   *(Trap 1. The cycle-count argument, then "add work and see if the number responds".)*

3. **Why does JMH fork more than once, given that one fork already gives a clean JVM?**
   *(Machine-level reality §2. Between-JVM variance is a different source of variance
   from within-JVM variance, and it is frequently larger than the effect.)*

4. **What exactly does `@State` defend against, and why is it a structural fix rather
   than a flag?**
   *(Machine-level reality §4. Constant folding. The fix is that field reads through a
   non-constant reference cannot be folded — there is no flag that makes C2 stop folding
   constants, nor should there be.)*

5. **Two benchmark results have overlapping error bars. State the correct conclusion in
   one sentence, and state the two things you must not do.**
   *(Rule 2. "Not distinguishable by this experiment." Do not quote the means as a
   difference; do not re-run until you get the answer you want.)*

6. **Your JMH result says the new implementation saves 30 ns per call. Your `GET /orders`
   p50 baseline is in `/docs/java/baselines/`. Write the two-step arithmetic that decides
   whether to ship it.**
   *(Example 2, Amdahl step. Saving divided by request latency, compared against the
   ±10% gate tolerance.)*

7. **Someone asks you to benchmark a `@Transactional` Spring service method with JMH.
   Give the one-sentence refusal and name the correct tool.**
   *(Q5. It measures the database, not the code; use Topic 65 for the system and Topic 78
   to find what inside it is worth microbenchmarking.)*

---

## Quick reference card

### Annotations, as defences

| Annotation | Defends against |
|---|---|
| `@State(Scope.Benchmark\|Thread\|Group)` | Constant folding |
| `Blackhole` param / returning a value | Dead-code elimination |
| `@Warmup(iterations, time)` | Cold JIT / tier averaging |
| `@Measurement(iterations, time)` | A result with no error bar |
| `@Fork(3)` | Profile pollution + between-JVM variance |
| `@Fork(jvmArgs = {...})` | Measuring a JVM you do not deploy |
| `@BenchmarkMode` | Answering the wrong question |
| `@OutputTimeUnit` | Unit confusion |
| `@Setup(Level.Trial\|Iteration\|Invocation)` | Fixture cost inside the timed region |
| `@Param({...})` | Generalising from one arbitrary size |
| `@Threads(n)` | Unrecorded, accidental concurrency |
| `@Group` / `@GroupThreads` | Symmetric benchmarks hiding asymmetric contention |
| `@CompilerControl(DONT_INLINE)` | Inlining erasing the call you meant to measure |
| `@OperationsPerInvocation(n)` | Amortising a fixed batch inside one invocation |
| A `baseline` benchmark | Reporting noise as a result |

### Command line

```bash
# Create a project (omit archetypeVersion to take latest; then pin what you got)
mvn archetype:generate -DinteractiveMode=false \
  -DarchetypeGroupId=org.openjdk.jmh \
  -DarchetypeArtifactId=jmh-java-benchmark-archetype \
  -DgroupId=com.orderflow.bench -DartifactId=orderflow-bench -Dversion=1.0-SNAPSHOT

mvn -q clean verify                       # runs the annotation processor
java -jar target/benchmarks.jar -h        # help, and JMH's own version
java -jar target/benchmarks.jar -l        # list benchmarks
java -jar target/benchmarks.jar -lprof    # list available profilers

java -jar target/benchmarks.jar OrderTotalBenchmark          # run by regex
java -jar target/benchmarks.jar '.*Total.*' -f 3 -wi 5 -i 10 # explicit forks/iterations
java -jar target/benchmarks.jar X -p lineCount=5,25          # override @Param
java -jar target/benchmarks.jar X -bm sample                 # percentiles, not mean
java -jar target/benchmarks.jar X -t 4                       # threads
java -jar target/benchmarks.jar X -prof gc                   # allocation per op
java -jar target/benchmarks.jar X -prof perfasm              # Linux + hsdis
java -jar target/benchmarks.jar X -rf json -rff result.json  # machine-readable
java -jar target/benchmarks.jar X -jvmArgs "-Xmx1200m -XX:+UseG1GC"
java -jar target/benchmarks.jar X -f 0                       # DEBUG ONLY. Not a result.
```

### How to read a results table — the columns

```
Benchmark                       (param)  Mode  Cnt   Score   Error  Units
com.orderflow.bench.X.baseline       xxx  avgt   <n>   <n>  ± <n>  ns/op
com.orderflow.bench.X.variantA       xxx  avgt   <n>   <n>  ± <n>  ns/op
com.orderflow.bench.X.variantA:gc.alloc.rate.norm
                                     xxx  avgt   <n>   <n>  ± <n>   B/op
```

*illustration of the format, not captured output — column structure only, all values are
placeholders*

Reading order: **Units → Mode → Cnt → Error → Score.** Never Score first.

- **Units and Mode together** tell you which direction is better. `avgt` + `ns/op`: lower
  is better. `thrpt` + `ops/s`: higher is better.
- **Cnt** should equal forks × measurement iterations. If it is lower, a fork died.
- **Error** decides whether you have a result at all.
- **`:gc.alloc.rate.norm`** is bytes per operation. Often the most useful row on the page.
- **Score** is last, because it is meaningless without the other four.

### Diagnostic flags for a suspicious benchmark

```bash
-jvmArgs "-XX:+UnlockDiagnosticVMOptions -XX:+PrintCompilation"   # % marks OSR
-jvmArgs "-XX:+UnlockDiagnosticVMOptions -XX:+PrintInlining"      # Topic 75
-jvmArgs "-XX:-DoEscapeAnalysis"                                  # A/B control, Topic 75
-jvmArgs "-Xint"                                                  # no JIT at all
-jvmArgs "-XX:ActiveProcessorCount=2"                             # container reality, Topic 82
-jvmArgs "-Xlog:gc*"                                              # GC inside the benchmark, Topic 71
```

### Gotchas checklist

- [ ] Is there a `baseline` benchmark, and is my result meaningfully above it?
- [ ] Are the error bars separated, or am I about to report noise?
- [ ] Am I on at least three forks?
- [ ] Did I look at the warm-up iteration *sequence*, or just trust the count?
- [ ] Are all inputs in `@State`, with nothing `static final`?
- [ ] Is every computed value consumed, and the same number of values per variant?
- [ ] Are my `@Param` values the sizes production actually uses?
- [ ] Do the forked JVM args match the production container (heap, collector, CPU count)?
- [ ] Are both variants in the same JMH invocation, on the same machine?
- [ ] Did I run `-prof gc`? (Allocation is often the real answer.)
- [ ] Did I capture the whole output including the banner, not just the last table?
- [ ] Did I check the profile *first* (Topic 78) to know this method matters at all?
- [ ] Will I re-run the Topic 65 baseline to prove the system-level number moved?

---

## When would I use this at work?

**1. Ending a code-review argument with evidence instead of seniority.**
Someone blocks a PR claiming a stream pipeline is slower than a loop, or that a
`BigDecimal` is unacceptable in a hot path. You write a fifteen-line JMH class, run it
with three forks and `-prof gc`, and paste the table with the error bars and the
baseline row. Half the time it confirms the reviewer and half the time it refutes them,
and either way the discussion ends in a measurement. Teams that settle arguments this way
develop a different culture from teams that settle them by tenure — and you can be the
person who starts that.

**2. Sizing a library or data-structure decision before you commit to it.**
Caffeine versus a `ConcurrentHashMap` for the product cache. `AtomicLong` versus
`LongAdder` for the order counter (Topic 95). Jackson versus a hand-rolled writer for the
reconciliation export. These are decisions you make once and live with for years, and
they are exactly the shape JMH answers well: a pure operation, a realistic input, a
scaling question via `@Param` and `@Threads`. The half day you spend measuring is
insurance against a rewrite two years later.

**3. Validating — or, more often, killing — a performance proposal before anyone builds
it.**
The most valuable thing this tool does is stop work. Someone proposes a two-week
optimisation. You profile (Topic 78), find it is 1% of the request, microbenchmark the
best case, and show that even a perfect implementation moves p99 by less than your load
test can resolve. That is two weeks returned to the roadmap, delivered as a table rather
than an opinion — and it is the single most senior thing in this document. Optimisation
that cannot be demonstrated is not optimisation; it is decoration with a maintenance cost.

---

## Connected topics

**Prerequisites — the claims this topic exists to settle:**

- **01 — Boxing and the `Integer` cache:** "boxing allocates unless it hits the cache" is
  measurable with `-prof gc`, and `gc.alloc.rate.norm` is the number that settles it.
- **06 — Type erasure:** why a generic benchmark's `checkcast`s and bridge methods are
  part of what you are measuring, whether you meant them to be or not.
- **11 — `ArrayList` versus `LinkedList`:** the curriculum's strongest performance claim,
  and the medium exercise here is where you either confirm or refute it on your hardware.
- **18 — Strings, the pool and `StringBuilder`:** the failure drill's subject. `+` on one
  expression compiles to `invokedynamic`; the claim that a builder beats it is a
  hypothesis until you run this.
- **21 — Lambdas:** non-capturing lambdas are cached, capturing ones allocate. `-prof gc`
  is where you see the difference, in bytes per operation.
- **25 — Parallel streams:** the three preconditions are all measurable, and the hard
  exercise here is a parallel-stream proposal killed with `-XX:ActiveProcessorCount=2`.
- **65 — The load-testing gate:** the system-level counterpart. JMH measures a method;
  Topic 65 measures the service. Neither substitutes for the other, and the ±10% gate
  tolerance is what decides whether a JMH win is shippable.
- **68 — TLABs and allocation:** why allocation is a pointer bump and why the interesting
  number is bytes per operation rather than allocations per second.
- **69 — Object layout:** why `gc.alloc.rate.norm` values are multiples of 8, and how to
  identify which object a 24- or 40-byte figure corresponds to.
- **70 — GC fundamentals:** allocation rate and live set. Multiply `gc.alloc.rate.norm`
  by your production request rate to estimate a benchmark's system-level GC contribution.
- **71 / 72 — G1 and the low-pause collectors:** why `@Fork(jvmArgs = ...)` must name the
  collector you deploy, and why a GC pause inside a measurement iteration shows up as one
  wild outlier in the sequence.
- **73 — Safepoints and TTSP:** why a benchmark iteration can be interrupted by a global
  pause that has nothing to do with your code, and why you read the iteration sequence
  rather than only the mean.
- **74 — Tiered compilation and deoptimization:** the mechanism behind warm-up and behind
  profile pollution. This is the single most load-bearing prerequisite for Topic 77;
  `@Fork(3)` is Topic 74's drill applied as a defence.
- **75 — Escape analysis and inlining:** why a benchmark can report zero allocation for
  code that obviously allocates, and why `-XX:-DoEscapeAnalysis` is the A/B control.
- **76 — Reading bytecode:** the tool for "what did the language do". JMH is the tool for
  "what does it cost". Two questions, two tools — and `javap -c` is how you check that
  your benchmark is measuring what you think.
- **80 — Off-heap and NMT:** JMH's `gc` profiler measures heap allocation only. Direct
  buffers do not appear in it; that gap is Topic 80's territory.
- **82 — Containers and JVM tuning:** the reason `-XX:ActiveProcessorCount` belongs in
  your `@Fork` args. A benchmark on a 16-core laptop describing a 2-vCPU container is
  measuring a machine you do not run.

**This unlocks:**

- **78 — Profiling and flame graphs:** the step that comes *before* this one. Profile to
  choose what to benchmark; benchmark to quantify what the profile named.
- **90 — `ExecutorService` and pool sizing:** queue and rejection-policy choices are
  `@Threads` + `@Group` benchmarks, and pool sizing is a measurement, not a formula.
- **95 — Atomics and `LongAdder`:** the canonical `@Threads` scaling benchmark. "Lock-free
  is not the same as scalable" is a claim you settle here, at several thread counts.
- **96 — False sharing:** only demonstrable with JMH plus `@Threads` plus `perfnorm`
  hardware counters. There is no other way to see it.
- **101 — Virtual threads:** the blocking-versus-non-blocking calculus changes entirely,
  and the pinning cost is measurable with `@Threads` and the right benchmark shape.
- **118 — Metrics and Micrometer:** the production counterpart. A timer in production and
  a JMH benchmark measure different things; knowing which question each answers stops you
  reporting one as the other.
- **129 — Capacity, cost and latency budgets:** `gc.alloc.rate.norm` times request rate
  feeds a capacity model. This is where a microbenchmark number legitimately becomes a
  business number.

---

*Java baseline 21, running on JDK 25. Four things in this document are deliberately
hedged rather than asserted, each with a settling command given inline: the exact naming
and shape of JMH's generated harness source (`find target/generated-sources -name
'*_jmhTest.java'`), whether your JMH build negotiates compiler blackholes or falls back
to the classic volatile-arithmetic implementation (read the run banner), the current JMH
and archetype versions (`java -jar target/benchmarks.jar -h`, and `grep jmh.version
pom.xml` after generation), and which profilers your platform supports
(`-lprof`). The extent to which JMH's generated measurement loop avoids OSR is stated as
"mostly, for the benchmark method" rather than "entirely", because a loop you write
inside a benchmark method can still be OSR-compiled — verify with `-XX:+PrintCompilation`
and look for `%`.*

*Everything else here — that JMH forks a fresh JVM process per trial, that the annotation
processor generates the harness at build time, that `Blackhole` defeats dead-code
elimination and `@State` defeats constant folding, and the meaning of every column in the
results table — is documented behaviour, and every one of them has a command above that
confirms it on your machine.*

*No number in this document was measured. There are no benchmark results here, and the
two format illustrations contain placeholders only. Every figure you will ever quote from
this topic must come from your own run, on your own hardware, with the banner attached.*
