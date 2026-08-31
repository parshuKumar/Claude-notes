# 74 — JIT I: The Interpreter, C1/C2 Tiered Compilation, Profiling, Deoptimization, and Warm-Up

## Phase: 8 — JVM Internals
## Category: ELITE
## Java baseline: 21  |  Notes features from: 21 / 25
## Project spine: this is how you explain why `orderflow`'s first thousand requests after a deploy miss the Topic 65 baseline by a wide margin, and why a feature flag enabled for 2% of traffic can permanently slow down the 98% that never touches it. Baselines live in `/docs/java/baselines/`.

---

## Mechanical statement

Read this three times. The rest of the document is an elaboration of it.

> **Every Java method starts interpreted.** The interpreter walks bytecode one instruction
> at a time. It is slow — roughly an order of magnitude slower than compiled code — and it
> is also where the JVM *learns* about your program.
>
> **Two counters per method drive everything: an invocation counter and a back-edge
> counter.** The invocation counter increments on entry. The back-edge counter increments
> every time control jumps backwards — that is, on every loop iteration. When either
> crosses a threshold, the method is queued for compilation.
>
> **Compilation happens in tiers.** Tier 0 is the interpreter. Tiers 1, 2 and 3 are **C1**,
> a fast compiler producing decent code — tier 3 being the important one, because it emits
> code that *keeps profiling*. Tier 4 is **C2**, a slow compiler producing aggressively
> optimised code. The normal path for a hot method is **0 → 3 → 4**: interpret, then run
> profiled C1 code while gathering a type profile, then compile with C2 using that profile.
>
> **C2 does not optimise your code. It optimises your code given the profile it observed.**
> If a call site only ever saw one receiver type, C2 inlines that implementation directly
> and *speculates* that it will keep being the only one. If a branch was never taken, C2
> may not emit code for it at all. These are assumptions, not proofs.
>
> **Each speculation is protected by an uncommon trap.** A cheap runtime check that, if it
> fails, transfers execution out of the compiled frame and **back into the interpreter**,
> reconstructing the interpreter's state from the compiled frame. That transfer is
> **deoptimization**. The compiled method is marked *not entrant*, and the method is
> recompiled later with the wider profile — producing **permanently slower code, because
> the wider profile permits fewer optimisations.**

Five consequences follow directly, and you should be able to derive each one:

1. **The first thousand-odd executions of any code path are slow**, and the slowness is
   not proportional to load — it is proportional to how many times *that specific method*
   has run. This is warm-up, and it is why the minute after a deploy looks like an outage.
2. **A benchmark that runs each case once measures the interpreter.** A benchmark that
   runs a loop without warm-up measures a blend of interpreter, C1 and C2 in an unknown
   ratio. Both numbers are meaningless. This is Topic 77's entire reason for existing.
3. **A rarely-used code path can permanently degrade a hot one.** Introduce a second
   implementation at a call site that has been monomorphic for a million calls, and C2
   deoptimizes, recompiles less aggressively, and never gets the original speed back.
   That is **profile pollution**.
4. **Compiled code lives in a fixed-size region called the code cache.** If it fills, the
   compiler shuts down and everything runs interpreted from then on — a cliff-edge,
   permanent, order-of-magnitude regression with no GC signal and no obvious cause.
5. **You can see all of it.** `-XX:+PrintCompilation` shows every compilation and every
   deoptimization. `-XX:+TraceDeoptimization` names the reason. `jcmd Compiler.codecache`
   shows the cache. None of this is guesswork.

---

## The bridge from what you know

### HONEST ANALOGUE — and this is the strongest one in the entire curriculum

**You already understand this.** Not partially. Not by analogy. V8 does the same thing, for
the same reasons, with the same failure modes, and if you have ever heard the advice *"don't
change an object's shape in a hot loop"* then you have already been taught profile pollution
— you just were not told the name.

Here is the mapping, and it is close enough to be genuinely load-bearing:

| V8 | HotSpot | Same? |
|---|---|---|
| **Ignition** — bytecode interpreter, gathers feedback | **Tier 0** — bytecode interpreter, increments counters and gathers profile | **Yes** |
| **Sparkplug** — fast non-optimising baseline compiler | **Tier 1–3 (C1)** — fast compiler; tier 3 keeps profiling | **Yes, closely** |
| **Maglev** — mid-tier optimising compiler | *No exact counterpart*; C1 tier 3 plus C2's queue fills a similar niche | **Roughly** |
| **TurboFan** — slow, aggressive, speculative optimiser | **Tier 4 (C2)** — slow, aggressive, speculative optimiser | **Yes** |
| **Feedback vectors** recording observed types per call site | **Method data objects (MDO)** recording observed receiver types per call site | **Yes** |
| Speculating on **hidden classes / maps** | Speculating on **receiver classes and the class hierarchy** | **Yes** |
| **Deoptimization / bailout** to the interpreter when a map check fails | **Uncommon trap → deoptimization** to the interpreter when a class check fails | **Yes** |
| "Don't change an object's shape in a hot loop" | **Profile pollution at a call site** | **Yes — same phenomenon** |
| **OSR** (loop entry into optimised code) | **OSR** (on-stack replacement at a loop back-edge) | **Yes, same name even** |
| `--trace-opt` / `--trace-deopt` | `-XX:+PrintCompilation` / `-XX:+TraceDeoptimization` | **Yes** |
| `%NeverOptimizeFunction`, `--no-opt` | `-XX:TieredStopAtLevel=1`, `-Xint` | **Yes** |
| Warm-up before benchmarking | Warm-up before benchmarking | **Yes** — and you have been bitten by this in Node |

**Use this heavily.** Every time something in this document feels foreign, translate it back:
"C2 deoptimizing because a call site became polymorphic" is "TurboFan bailing out because
the map check failed". If you have ever profiled a Node service and seen a function
deoptimize because someone passed a differently-shaped object, you have already lived this
topic. The mental model transfers whole.

### Where the analogue stops being exact — the Java specifics you must get right

Four places where "same idea" is not "same detail", and the details are what interviews and
incidents turn on.

**1. The tier structure is explicit, numbered, and controllable.**
V8's pipeline is largely opaque and its heuristics are not yours to configure. HotSpot's
tiers are **0 through 4**, each has a name and a compiler, the thresholds are printable
flags, and `-XX:TieredStopAtLevel=N` caps the pipeline. You will be asked "what are the
tiers" in an interview and expected to answer with numbers.

**2. The promotion trigger is two explicit counters, not a heuristic you cannot name.**
**Invocation counter** and **back-edge counter**. The second one is why a method called
*once* containing a hot loop still gets compiled — via OSR. Being able to say "invocation
counter for method-level compilation, back-edge counter for loop-level, and back-edges are
what trigger OSR" is a senior answer.

**3. Java has a class hierarchy and *class hierarchy analysis*.**
V8 speculates on hidden classes, which are an emergent property of how you built the object.
HotSpot speculates on something stronger: if only one class implementing an interface is
currently *loaded*, C2 can treat that call as monomorphic and inline it **without even a
type check** — protected instead by a dependency that invalidates the compiled code if a
second implementation is ever loaded. **Class loading can deoptimize your code**, which has
no V8 counterpart at all, and it is why a lazily-initialised Spring bean can perturb a hot
path that has nothing to do with it.

**4. Deoptimization has named reasons and named actions, and you can read them.**
`-XX:+TraceDeoptimization` prints a *reason* (`class_check`, `unstable_if`, `bimorphic`,
`null_check`, `unloaded`, ...) and an *action* (`reinterpret`, `make_not_entrant`,
`make_not_compilable`, ...). That is far more diagnostic detail than `--trace-deopt` gives
you, and reading it is the skill this document's failure drill builds.

| You know | Java | Verdict |
|---|---|---|
| Tiered compilation with speculation | Tiered compilation with speculation | **HONEST ANALOGUE** |
| Deopt when a shape assumption breaks | Uncommon trap when a class assumption breaks | **HONEST ANALOGUE** |
| "Don't change shape in a hot loop" | Profile pollution | **HONEST ANALOGUE** — same phenomenon, better tooling |
| Warm-up matters for benchmarks | Warm-up matters for benchmarks | **HONEST ANALOGUE** — and JMH exists because of it |
| Opaque pipeline, few knobs | Tiers 0–4, printable thresholds, `TieredStopAtLevel` | **PARTIAL** — you gain control and the ability to misuse it |
| Hidden-class speculation | **Class-hierarchy analysis** — speculation invalidated by *class loading* | **PARTIAL** — Java's is stronger and has a failure mode JS lacks |
| `--trace-deopt` | `PrintCompilation` + `TraceDeoptimization` with named reasons | **PARTIAL** — far more detail |
| — | A fixed-size **code cache** that can fill and disable the compiler | **NO ANALOGUE** |

> **The theme:** you are not learning a new concept here. You are learning the Java spelling
> of a concept you already have, plus one genuinely new failure mode (the code cache) and
> one genuinely new mechanism (class-hierarchy-analysis-driven deopt). **Lean on the V8
> model hard, then be precise about the four differences.**

---

## What is this?

### The tiers, by number

| Tier | Compiler | Profiling | When it is used |
|---|---|---|---|
| **0** | Interpreter | Full profiling | Every method starts here |
| **1** | C1 | **None** | Trivial methods C2 could not improve — compiled once and left alone |
| **2** | C1 | Limited (counters only) | When the **C2 queue is long**: get to compiled code fast, profile lightly, upgrade later |
| **3** | C1 | **Full** | **The normal warm path.** Compiled, but still collecting the type profile C2 needs |
| **4** | C2 | None (it consumes the profile) | **The steady state for hot methods** |

The common path is **0 → 3 → 4**. The variants exist for edge cases: a trivial getter goes
0 → 1 and stops, because C2 has nothing to add. A method that becomes hot while the C2 queue
is backed up may go 0 → 2 → 3 → 4, trading profile quality for getting out of the interpreter
sooner.

### The counters

```
invocation_counter  — incremented on method entry
back_edge_counter   — incremented on every backward jump (i.e. every loop iteration)
```

Roughly: cross a tier-3 threshold and the method is queued for C1-with-profiling; keep going
and cross a tier-4 threshold and it is queued for C2. The tier-4 decision also considers the
profile's maturity, not just the raw count.

**The default thresholds are in the hundreds to low tens of thousands.** I am not quoting
exact numbers, because they differ by JDK and are adjusted dynamically based on compiler
queue length. **Print your own — verify, don't trust me:**

```bash
java -XX:+PrintFlagsFinal -version | grep -E "Tier[0-9]+(Invocation|Compile|BackEdge)Threshold|CompileThreshold|TieredCompilation"
```

**The number to carry away is the order of magnitude, and it is the practically important
fact:** a method needs to run **thousands of times** before C2 sees it. Not tens. Not
millions. That single fact explains warm-up, explains why microbenchmarks need warm-up
iterations, and explains why a code path exercised only during a nightly job is effectively
never optimised.

### What C2 actually does with the profile

C2 is a **speculative** optimiser. The profile tells it what has happened so far, and it
generates code that is fast *if that keeps being true*:

| Observation in the profile | What C2 does | The speculation |
|---|---|---|
| This call site saw exactly one receiver class | **Inline the target directly**, guarded by a class check — or with no check at all if class-hierarchy analysis says only one implementation is loaded | "it will keep being that class" |
| This call site saw two receiver classes | Inline **both**, with a type switch (bimorphic inlining) | "it will keep being one of those two" |
| This call site saw three or more | **Give up on inlining.** Emit a virtual call through the vtable or itable | — |
| This branch was never taken | Emit an **uncommon trap** instead of the branch body | "it will keep not being taken" |
| This value was never null | Skip the null check; rely on the hardware trap | "it will keep being non-null" |
| This field was always the same constant | Constant-fold it | "it will not change" |
| This loop's bounds are provable | Eliminate the array bounds check | (this one is a proof, not a speculation) |

**The inlining decisions are the load-bearing ones**, because inlining is what enables
everything else — escape analysis, constant folding, dead-code elimination. Topic 75 is
entirely about that consequence. **A call site that goes polymorphic does not just lose the
inline; it loses every optimisation that depended on the inline.**

### Deoptimization

When a speculation is violated, the guard fails and the JVM must transfer execution from an
optimised compiled frame back into the interpreter — **mid-method**, with the interpreter's
stack, locals and operand stack reconstructed from the compiled frame's contents and the
debug information the compiler recorded for that program counter.

That reconstruction is why deoptimization is *possible at all*, and it is not cheap. The
important cost, though, is not the transfer:

> **The transfer happens once. The recompilation is forever.** The compiled method is marked
> **not entrant** (no new calls will enter it), the wider profile is recorded, and when the
> method next gets hot it is recompiled with **fewer permitted speculations**. That version
> is slower, and it is the version you keep.

### On-stack replacement (OSR)

A method called once but containing a million-iteration loop would never be compiled by
invocation counting alone. The back-edge counter fixes that: when it crosses a threshold, the
JVM compiles a special **OSR version** of the method entered at that loop's bytecode index,
and **swaps the running frame over to it mid-execution**.

Three practical consequences:

1. **A long loop in `main` gets compiled while it runs**, which is why a naive benchmark's
   average blends interpreted and compiled execution in a ratio determined by the loop count
   you happened to choose.
2. **OSR-compiled code is often slightly worse** than a normally-compiled version, because
   it is entered mid-method with a fixed state and the compiler has less freedom.
3. `PrintCompilation` marks OSR compilations with `%`, and that is how you spot a benchmark
   measuring OSR code rather than steady-state code.

### The code cache

Compiled code lives in the **code cache**, a fixed-size region of native memory reserved at
startup (`-XX:ReservedCodeCacheSize`). It is not the heap and `-Xmx` does not bound it.

Modern JVMs segment it into three parts: non-method code (stubs, interpreter), profiled
nmethods (C1 tier 2/3 code), and non-profiled nmethods (C1 tier 1 and C2 tier 4 code).

**If it fills, the compiler stops.** The JVM logs a warning along the lines of "CodeCache is
full. Compiler has been disabled." and from that moment everything not already compiled runs
interpreted — an order-of-magnitude regression that persists until restart. Trap 4.

### `[JAVA 25]` Ahead-of-time class loading and method profiling

Recent JDKs add an **AOT cache** — a file produced by a training run that can carry class
loading, linking and, in the newest versions, **method profiles** across JVM restarts, so a
freshly started JVM begins with some of the knowledge a warmed one had.

> **One-line version note, and it is a genuine uncertainty:** I am **not** asserting which
> AOT capabilities are present, enabled, or production-ready on your specific JDK 25 build —
> the feature has arrived in stages across releases and the flag surface has moved. **Settle
> it with `java -XX:+PrintFlagsFinal -version | grep -i -E "AOT|CDS"` and your vendor's
> release notes.** The reason it belongs in this topic: **it attacks warm-up directly**, and
> warm-up is Trap 3's whole subject. On JDK 21, AppCDS (`-XX:SharedArchiveFile`) is the
> available lever and it addresses class loading only, not method profiles.

---

## Why does it matter?

**1. Because "the first minute after deploy is slow" is a JIT fact, and people spend money
on it instead.**

The symptom is universal: a rolling deploy, and every freshly started pod misses its latency
SLO for a period, then recovers. The instinctive fixes — bigger instances, more replicas,
"warming up" the load balancer by guessing — all cost money and none of them address the
cause, which is that **a method needs thousands of executions before C2 compiles it**. The
correct fixes are readiness-gated warm-up, AppCDS or an AOT cache, and a deploy strategy that
does not send full traffic to a cold JVM. Knowing this is the difference between a
configuration change and a hardware purchase.

**2. Because profile pollution is a real, load-bearing production hazard that almost nobody
looks for.**

A feature flag enabled for 2% of traffic introduces a second implementation at a call site
that has been monomorphic for months. C2 deoptimizes and recompiles bimorphically or gives
up on inlining entirely. **The 98% of traffic that never touches the new code gets slower —
permanently, until restart.** The p99 regression correlates with the flag rollout, but the
affected endpoints have nothing to do with the feature, so nobody connects them. This is one
of the hardest classes of performance regression to diagnose without knowing the mechanism.

**3. Because every benchmark you will ever write is wrong without this.**

`System.nanoTime()` around a loop measures dead-code elimination, constant folding, OSR, and
cold-JIT state in unknown proportions. This is not a small correction — it is routinely an
order of magnitude, and sometimes it reports zero because the work was deleted. Topic 77 is
the full treatment; **this topic is why Topic 77 exists.**

**4. Because the code cache is a cliff, not a slope.**

An application with a lot of code — a large Spring app with many auto-configurations, an
agent rewriting classes (Topic 81), lots of lambdas and generated proxies (Topic 40) — can
fill the code cache. When it does, performance does not degrade gradually. It falls off a
cliff, permanently, with no GC signal and no memory alarm. `jcmd <pid> Compiler.codecache` is
a ten-second check that almost nobody runs.

**5. Because it is the prerequisite for Topic 75, which is where the money is.**

Escape analysis, scalar replacement and lock elision all depend on inlining succeeding first.
**Inlining decisions come from the profile this topic describes.** If you do not understand
why a call site is monomorphic and what makes it stop being monomorphic, you cannot reason
about why an allocation was or was not eliminated.

---

## Machine-level reality

### Where the profile lives

Each method has a **MethodData** object (an "MDO") allocated when profiling begins. It holds,
per bytecode index that needs it:

- **Receiver type profile** at virtual and interface call sites — the observed classes and
  their counts. **The number of distinct receivers it can record is bounded** by
  `TypeProfileWidth`, typically 2. That is the mechanical reason "monomorphic and bimorphic
  are special; three or more is just polymorphic".
- **Branch taken/not-taken counts**, which is what lets C2 conclude a branch is never taken.
- **Null-seen flags** per null check.
- **Invocation and back-edge counters.**

```bash
java -XX:+PrintFlagsFinal -version | grep -E "TypeProfileWidth|ProfileTraps|MethodProfileWidth"
```

**WHAT TO LOOK FOR:** `TypeProfileWidth`. If it is 2, a call site that sees three receiver
types has literally nowhere to record the third, which is why C2 gives up rather than
inlining trimorphically.

### Call-site states, and what each costs

| State | Receivers seen | What C2 emits | Cost |
|---|---|---|---|
| **Monomorphic** | 1 | The target **inlined**, guarded by a class check — or **no check at all** under class-hierarchy analysis if only one implementation is loaded | Nearly free, and it unlocks every downstream optimisation |
| **Bimorphic** | 2 | **Both** targets inlined behind a two-way type switch | Cheap, but code size doubles and inlining budget is consumed faster |
| **Polymorphic / megamorphic** | 3+ | A **virtual call** through the vtable (class) or itable (interface). **No inlining.** | An indirect branch the CPU may mispredict — and, far worse, **everything downstream of the inline is lost** |

**The last column is the point.** A megamorphic call site costs you a handful of nanoseconds
directly. It costs you far more indirectly, because the caller can no longer see into the
callee, so escape analysis cannot prove non-escape (Topic 75), constants cannot be folded,
and branches cannot be eliminated. **The direct cost is small and the indirect cost is
unbounded.**

Interface calls (`invokeinterface`) are somewhat more expensive than class calls
(`invokevirtual`) when megamorphic, because itable lookup is more work than a vtable index.
Both are dwarfed by the lost inlining.

### Class-hierarchy analysis — the speculation with no V8 counterpart

If `PaymentGateway` is an interface and **only** `StripeGateway` is currently loaded, C2 can
compile `gateway.charge(...)` as a **direct, unguarded, inlined call**. Not "inlined with a
class check" — inlined with *nothing*, because the JVM knows there is nothing else it could
be.

The safety mechanism is a **dependency**: the compiled method records that it assumed a
single implementation. **If a second implementation is ever loaded, every compiled method
carrying that dependency is invalidated and marked not entrant.**

Three consequences that surprise people:

1. **Class loading can deoptimize code that has nothing to do with the class being loaded.**
2. **A lazily-initialised Spring bean, a reflective instantiation, or a proxy generated on
   first use** (Topic 40) can therefore perturb a hot path at an arbitrary moment.
3. **Behaviour differs between environments** where different sets of classes get loaded.
   Your staging environment may never load the second implementation and therefore never
   reproduce production's performance.

### Deoptimization reasons you will actually see

`-XX:+TraceDeoptimization` (needs `-XX:+UnlockDiagnosticVMOptions`) prints a reason and an
action per event. The ones worth recognising:

| Reason | Means | Typical cause in a service |
|---|---|---|
| `class_check` | A receiver was not the speculated class | **The classic profile pollution event.** A second implementation appeared at a call site |
| `bimorphic` / `bimorphic_or_optimized` | A call site that had two types saw a third | A third implementation, or a mock leaking into a shared JVM |
| `unstable_if` | A branch C2 assumed was never taken was taken | An error path finally executed; a null finally appeared; a feature flag flipped |
| `null_check` | A value assumed non-null was null | First null on a path that had never seen one |
| `unloaded` / `uninitialized` | A class referenced in compiled code was not yet loaded/initialised | Lazy initialisation, first use of a rare code path |
| `unreached` | Execution reached code C2 believed unreachable | Same family as `unstable_if` |
| `predicate` / `loop_limit_check` | A loop-optimisation guard failed | Loop bounds changed shape at runtime |
| `constraint` / `speculate_class_check` | A type speculation from profile data failed | Generic code specialised on an observed type |

And the actions:

| Action | Means |
|---|---|
| `none` | Deoptimize this once; the compiled code stays valid for other paths |
| `maybe_recompile` | Deoptimize and consider recompiling with the wider profile |
| `reinterpret` | Fall back to the interpreter for this execution |
| `make_not_entrant` | **No new calls enter the compiled version.** Existing frames finish; the method will be recompiled |
| `make_not_compilable` | **Give up on compiling this method at this tier.** Repeated failures; the method may stay interpreted |

**`make_not_compilable` is the one to fear.** It means the JVM has repeatedly tried to
optimise a method, been wrong every time, and stopped trying. That method is slow forever.

### The code cache, in detail

```bash
jcmd <pid> Compiler.codecache
jcmd <pid> Compiler.codelist        # every compiled method, with tier
java -XX:+PrintFlagsFinal -version | grep -E "ReservedCodeCacheSize|InitialCodeCacheSize|SegmentedCodeCache|UseCodeCacheFlushing"
```

**WHAT TO LOOK FOR:** free space per segment, and whether flushing has occurred.

| What you see | What it means |
|---|---|
| Plenty free in every segment | Healthy. Re-check after a long soak, not at startup. |
| The **profiled nmethods** segment nearly full | Many methods at tier 2/3. Common in large Spring apps with agents. C2 code has nowhere to be promoted from. |
| The **non-profiled nmethods** segment nearly full | Lots of C2 code. A very large hot codebase, or heavy lambda/proxy generation. |
| Any "CodeCache is full" message in the logs | **Incident.** The compiler is disabled or has been. Everything not already compiled is interpreted. Restart is the immediate fix; raising `ReservedCodeCacheSize` is the real one. |
| Frequent code-cache flushing / "sweeper" activity | The cache is under pressure and methods are being evicted and recompiled. Wasteful, and a leading indicator of the previous row. |

**Common causes of code-cache pressure**, in rough order: a very large application with many
frameworks; a `-javaagent` rewriting classes (Topic 81) which both increases method count and
increases method sizes; heavy use of lambdas and `invokedynamic` (Topic 21), each of which
generates a class; dynamic proxies and CGLIB subclasses (Topic 40); and heavy reflection.

### The flags that matter, and the ones that are cargo cult

| Flag | What it does | Use it? |
|---|---|---|
| `-XX:+PrintCompilation` | Log every compilation and deopt event | **Yes, in a drill.** Never permanently — it is very chatty |
| `-XX:+UnlockDiagnosticVMOptions -XX:+PrintInlining` | Show inlining decisions and why they failed | **Yes, in a drill.** Topic 75's main instrument |
| `-XX:+UnlockDiagnosticVMOptions -XX:+TraceDeoptimization` | Name the reason and action for every deopt | **Yes, in a drill** |
| `-XX:+LogCompilation -XX:LogFile=...` | Structured XML of every compiler decision, for JITWatch | **Yes**, when `PrintCompilation` is not enough |
| `-XX:TieredStopAtLevel=1` | Cap at C1. Fast warm-up, **no C2, permanently lower peak throughput** | **Only** for short-lived processes: CLI tools, tests, build steps. **Never** for a long-running service — Trap 5 |
| `-XX:-TieredCompilation` | Disable tiering; C2 only, with a much higher threshold | Almost never. Slower warm-up for a marginal steady-state gain |
| `-Xint` | Interpreter only | **A/B control in a drill only.** Order of magnitude slower |
| `-Xcomp` | Compile everything immediately, no profile | **Never in production.** Terrible code (no profile) and a very slow start. Useful only to prove a point |
| `-XX:ReservedCodeCacheSize=<n>` | Size the code cache | **Yes**, if `Compiler.codecache` shows pressure |
| `-XX:CICompilerCount=<n>` | Number of compiler threads | Only when the CPU quota misleads the JVM (Topic 82) |
| `-XX:-Inline` | Disable inlining entirely | **A/B control only.** Devastating in production |

---

## Example 1 — minimal

The point of this example is to **see a deoptimization happen**, with your own eyes, in a
program small enough that nothing else is going on.

### The probe

```java
package com.orderflow.lab.jit;

/**
 * Warms a call site to MONOMORPHIC, then introduces a second implementation
 * and keeps calling. The call site becomes bimorphic and the compiled code
 * that assumed one type is invalidated.
 *
 * args[0] = iterations in the monomorphic phase
 * args[1] = iterations in the polluted phase
 */
public final class CallSitePollution {

    interface PriceRule {
        long apply(long minorUnits);
    }

    static final class StandardRule implements PriceRule {
        @Override public long apply(long m) { return m; }
    }

    static final class PromotionalRule implements PriceRule {
        @Override public long apply(long m) { return m - (m / 10); }
    }

    /** The hot call site. Deliberately tiny so the whole story is about dispatch. */
    static long total(PriceRule rule, long iterations) {
        long acc = 0;
        for (long i = 0; i < iterations; i++) {
            acc += rule.apply(i & 0xFFFF);     // <-- THE CALL SITE
        }
        return acc;
    }

    public static void main(String[] args) {
        long warm = Long.parseLong(args[0]);
        long dirty = Long.parseLong(args[1]);

        PriceRule standard = new StandardRule();

        // Phase 1: the call site sees exactly ONE receiver type, many times.
        System.out.println("phase 1 (monomorphic) start");
        long a = total(standard, warm);
        System.out.println("phase 1 done: " + a);

        // Phase 2: a SECOND implementation appears at the same call site.
        // Note: merely LOADING this class is enough to invalidate a
        // class-hierarchy-analysis-based inline, before it is even called.
        PriceRule promo = new PromotionalRule();
        System.out.println("phase 2 (polluted) start");
        long b = total(standard, dirty) + total(promo, dirty);
        System.out.println("phase 2 done: " + b);
    }
}
```

### Run it and watch the compiler

```bash
javac -d out --release 21 src/main/java/com/orderflow/lab/jit/CallSitePollution.java

java -XX:+UnlockDiagnosticVMOptions \
     -XX:+PrintCompilation \
     -XX:+TraceDeoptimization \
     -cp out com.orderflow.lab.jit.CallSitePollution 50000000 50000000 \
     2>&1 | tee logs/pollution.log
```

### The shape of `PrintCompilation` output

```
<timestamp_ms>  <compile_id> <attributes> <tier> <fully.qualified.Method>(<descriptor>) @<osr_bci> (<bytecode_size> bytes)
<timestamp_ms>  <compile_id> <attributes> <tier> <fully.qualified.Method>   made not entrant
<timestamp_ms>  <compile_id>              <tier> <fully.qualified.Method>   made zombie
```

***Illustration of the format, not captured output. Every value is a placeholder chosen to
show the field positions.***

Field by field:

| Field | Meaning |
|---|---|
| `<timestamp_ms>` | Milliseconds since JVM start. **How you correlate compilation with a phase of your program.** |
| `<compile_id>` | Sequence number for this compilation task. The same method compiled at a higher tier gets a **new** id. |
| `<attributes>` | Flags — see the table below. **`%` is the one that matters most for benchmarks.** |
| `<tier>` | 1–4. **The number you are watching.** 4 means C2 got it. |
| `<Method>` | Class and method. |
| `@<osr_bci>` | Present only for OSR compilations: the bytecode index of the loop that triggered it. |
| `(<n> bytes)` | Bytecode size of the method. **Compare this against `MaxInlineSize` and `FreqInlineSize` — Topic 75.** |

The attribute characters:

| Char | Meaning |
|---|---|
| `%` | **On-stack replacement.** Compiled because of a loop's back-edge counter, entered mid-method |
| `s` | `synchronized` method |
| `!` | Has exception handlers |
| `b` | Blocking compilation — the application thread waited for it |
| `n` | Native method wrapper |
| `made not entrant` | **The deoptimization event.** No new calls enter this compiled version |
| `made zombie` | The not-entrant code has no live frames left and can be reclaimed from the code cache |

### What to look for

```bash
# The whole story of one method.
grep 'CallSitePollution::total' logs/pollution.log

# Every deoptimization, with the reason.
grep -iE 'not entrant|DEOPT|Uncommon trap|reason=' logs/pollution.log | head -40

# Which tier did the hot method reach, and when?
grep 'CallSitePollution::total' logs/pollution.log | awk '{print $1, $2, $4}'
```

**WHAT TO LOOK FOR:** the sequence of events around the phase-2 boundary.

| What you see | What it means |
|---|---|
| `total` compiled at tier 3, then tier 4, all during phase 1 | The normal warm-up path: profiled C1, then C2. **This is the baseline behaviour to recognise.** |
| A `made not entrant` for `total` shortly after "phase 2 start" | **The deoptimization, caught in the act.** The speculation that the call site was monomorphic was violated. |
| A `TraceDeoptimization` line with reason `class_check` or `bimorphic` | The reason, named. **This is the evidence you would cite in a postmortem.** |
| `total` recompiled at tier 4 again with a **new compile id** after the deopt | Recompilation with the wider profile. That new version is the one you keep, and it is slower. |
| A `%` next to a compilation of `total` | **OSR.** The loop got hot before the method did. Expected here, since `total` is called only a few times. |
| A deopt with reason `unloaded` before phase 2's calls even begin | **Class-hierarchy analysis.** Merely *loading* `PromotionalRule` invalidated an inline that assumed one implementation — before a single call through it. This is the Java-specific mechanism with no V8 counterpart. |
| No deopt at all | Possible: the call site may have been compiled bimorphically from the start if the JIT saw both types during warm-up, or the method may be small enough that dispatch is not the interesting cost. Increase phase-1 iterations, or check that phase 1 really completed before phase 2 began. |
| Enormous volumes of unrelated compilation output | Normal. `PrintCompilation` logs the whole JVM including the JDK's own classes. **Grep for your class.** |

### The A/B controls that make the result mean something

One run tells you what happened. These three tell you *why*.

```bash
# CONTROL 1: interpreter only. No compilation, no deopt, no speculation.
java -Xint -cp out com.orderflow.lab.jit.CallSitePollution 5000000 5000000

# CONTROL 2: cap at C1. Compiled, but no C2 speculation, so nothing to deoptimize.
java -XX:TieredStopAtLevel=1 -XX:+PrintCompilation \
     -cp out com.orderflow.lab.jit.CallSitePollution 50000000 50000000 | grep 'total'

# CONTROL 3: never let the call site be monomorphic - alternate types from the start.
#            (Modify main to interleave both rules in phase 1.)
```

| Control | What it isolates |
|---|---|
| `-Xint` | Everything is interpreted, so any speed difference between phases is *not* compilation. If phase 2 is still slower here, your slowdown is algorithmic, not JIT |
| `TieredStopAtLevel=1` | C1 does not speculate on receiver types the way C2 does, so **there should be no `made not entrant` for this reason**. If the phase-2 slowdown disappears, C2 speculation was the cause |
| Polluted from the start | The call site is bimorphic from its first compilation, so there is nothing to deoptimize — **and this is what the "fixed" version's steady state looks like** |

**Do not skip the controls.** A single run showing "phase 2 is slower" is consistent with at
least four explanations, and the controls eliminate three of them.

---

## Example 2 — production scenario (on the project spine)

### The constraints, taken from the Topic 65 baseline

| Thing | Value |
|---|---|
| Service | `orderflow`, containerised |
| Dataset | 100k products, 1M orders, 5M order lines |
| Container | 2 vCPU, 2 GiB memory limit |
| Heap | `-Xmx1200m -Xms1200m` |
| Collector | G1 (verify it, do not assume it) |
| Load mix | 70% catalogue read, 20% order read, 10% order placement |
| Arrival rate | open model, roughly 400 requests per second total |
| Latency budget | endpoint p99s within the recorded baseline |
| Deploy | rolling, 3 replicas, k8s readiness probe (Topic 121) |
| Baseline artefacts | `/docs/java/baselines/` |

### The code that has been running for months

The catalogue read path — 70% of all traffic — prices each product through an interface:

```java
package com.orderflow.catalog;

public interface PriceRule {
    /** Returns the effective price in minor units for this product and customer tier. */
    long effectivePrice(ProductSummary product, CustomerTier tier);
}

@Component
public class StandardPriceRule implements PriceRule {
    @Override
    public long effectivePrice(ProductSummary product, CustomerTier tier) {
        return product.listPriceMinorUnits();
    }
}
```

```java
@Service
public class CatalogQueryService {

    private final PriceRule priceRule;   // one implementation exists; injected by type

    public List<ProductView> page(int page, int size, CustomerTier tier) {
        return productRepository.findPage(page, size).stream()
            .map(p -> new ProductView(
                    p.id(), p.sku(), p.name(),
                    priceRule.effectivePrice(p, tier)))   // <-- THE HOT CALL SITE
            .toList();
    }
}
```

**Note the shape of the hot path.** `page(...)` is called ~280 times per second, and each
call invokes `effectivePrice` once per product in the page — say 50 — so **the call site
executes roughly 14 000 times per second.** It reaches C2 within seconds of startup and stays
there. Under class-hierarchy analysis, with only one `PriceRule` implementation loaded, C2
inlines `effectivePrice` **with no type check at all**, and — because the inline succeeded —
can then see that `ProductView`'s construction and the lambda's captures are local, enabling
the optimisations Topic 75 covers.

### The change

A promotions feature ships behind a flag. It adds a second implementation:

```java
@Component
@Primary                       // selected when the flag is on
public class PromotionalPriceRule implements PriceRule {

    private final PromotionRepository promotions;

    @Override
    public long effectivePrice(ProductSummary product, CustomerTier tier) {
        return promotions.bestActiveFor(product.id(), tier)
                .map(promo -> promo.applyTo(product.listPriceMinorUnits()))
                .orElse(product.listPriceMinorUnits());
    }
}
```

The rollout is careful and, by every normal standard, correct:

- Behind a feature flag.
- Enabled for **2% of traffic** in a canary.
- Load tested in staging, where it looked fine.
- Reviewed, tested, monitored.

### What you observe, in the order you observe it

1. **The flag is enabled for 2%.** Within a minute or two, on the pods serving the canary,
   **p99 on `GET /products` rises — for all traffic on those pods, not just the 2%.**
2. `GET /orders` p99 rises too, though it does not call `PriceRule` at all. (It shares the
   JSON serialisation and DTO mapping code that used to be inlined into the same compiled
   frames.)
3. **GC metrics are unchanged.** Allocation rate, live-set floor, pause durations: all
   normal. This is the observation that should immediately point away from Topic 70 and
   toward this one.
4. **CPU per request rises.** Modestly, but measurably, and uniformly across endpoints.
5. **Rolling back the flag does not fully fix it.** The pods stay degraded until they are
   restarted, because the deoptimized-and-recompiled code is the code they now have.
6. **Staging never reproduced it**, because staging's load never warmed the call site to C2
   in the first place, so there was nothing to deoptimize.

**Point 5 is the tell**, and it is the one that turns a confusing incident into a diagnosis:
*a performance regression that a rollback does not fix, but a restart does, is a JIT state
problem.* Nothing else has that signature.

### Why the 98% got slower

The call site went from **monomorphic** to **bimorphic**:

```
BEFORE:  one implementation loaded
         -> class-hierarchy analysis: direct call, no type check
         -> effectivePrice INLINED into the stream pipeline
         -> constants folded, allocation of intermediate objects eliminated (Topic 75)

AFTER:   two implementations loaded
         -> the CHA dependency is invalidated: every compiled method that assumed
            a single implementation is MADE NOT ENTRANT
         -> recompiled with a type check, bimorphic at best
         -> PromotionalPriceRule's body is large (a repository call, an Optional chain)
            so it likely exceeds the inlining size budget
         -> once the callee is not inlined, the caller loses everything downstream:
            no constant folding across the boundary, no escape analysis on objects
            passed into it (Topic 75), and a real call with real argument setup
```

**The 2% of traffic caused the deoptimization. The 98% pays for it**, because the compiled
code is shared — there is one compiled version of `page(...)` per JVM, not one per request.

### The diagnosis, as commands

```bash
# 1. Rule out GC first. It takes twenty seconds and it is the wrong tree 90% of the time.
grep -c 'Pause Full' /var/log/orderflow/gc.log
grep -oE '[0-9]+\.[0-9]+ms$' /var/log/orderflow/gc.log | awk '{s+=$1} END {print "pause ms:", s}'
#    Unchanged from baseline? Then it is not GC. Move on.

# 2. Rule out safepoints (Topic 73). Also twenty seconds.
grep -iE 'safepoint' /var/log/orderflow/safepoint.log | tail -20

# 3. Confirm the compiler is the story. Restart ONE pod with compilation logging.
#    PrintCompilation is chatty; log to a file and rotate.
-XX:+UnlockDiagnosticVMOptions -XX:+PrintCompilation -XX:+TraceDeoptimization \
-XX:+LogCompilation -XX:LogFile=/var/log/orderflow/compile.log

# 4. Watch the hot call site's compilation history across the flag flip.
grep -E 'CatalogQueryService::page|PriceRule::effectivePrice|StandardPriceRule' \
     /var/log/orderflow/compile.log

# 5. Find the deoptimizations and their reasons.
grep -iE 'made not entrant|Uncommon trap|reason=' /var/log/orderflow/compile.log | tail -50

# 6. Confirm inlining stopped. This is Topic 75's instrument, used here as evidence.
-XX:+UnlockDiagnosticVMOptions -XX:+PrintInlining
#    then look for "too big", "not inlineable", or a virtual call at that site.

# 7. Check the code cache while you are here - it is ten seconds and rules out Trap 4.
jcmd $(pgrep -f orderflow) Compiler.codecache
```

**WHAT TO LOOK FOR**, in order:

| What you see | What it means |
|---|---|
| GC and safepoint numbers unchanged, p99 up | **GC and safepoints are exonerated.** This is the most valuable early result. |
| `made not entrant` on `CatalogQueryService::page` timed with the flag flip | **The smoking gun.** Correlate the timestamp with the flag's audit log. |
| Deopt reason `class_check`, `bimorphic`, or an invalidated CHA dependency | Profile pollution confirmed, with a named mechanism. |
| A recompilation at tier 4 with a **new compile id** shortly after | The slower, wider-profile version. This is what the pod runs now. |
| `PrintInlining` says the callee is "too big" or shows a virtual call | The inline was lost, and with it everything downstream (Topic 75). |
| Deopts continuing at a steady rate rather than once | Something is repeatedly violating a speculation — a **deopt storm**, much worse than a one-off. Look for `make_not_compilable`. |
| Code cache nearly full | A *different* problem (Trap 4) that also produces uniform slowdown. Rule it in or out. |
| No deopt, no inlining change, p99 still up | JIT is exonerated too. Go to Topic 78 and profile — and check the obvious: is the new code path simply doing more work (a database call per product = an N+1, Topic 50)? |

**That last row matters.** `PromotionalPriceRule` calls a repository per product. If that is
not cached, the 2% of traffic is doing an N+1 query — which is a Topic 50 problem, not a
Topic 74 problem, and would produce a much larger regression on the 2% and none on the 98%.
**The distribution of the regression across traffic is what separates the two hypotheses:**
JIT pollution hurts everyone; a slow new code path hurts only its own traffic.

### The fixes, in the order you should consider them

| Fix | What it does | Cost |
|---|---|---|
| **Accept it.** Measure the regression; if it is within budget, ship the feature | Bimorphic inlining is often fine — two implementations still inline | Requires having measured, which is the whole point |
| **Make the new implementation small enough to inline.** Move the repository lookup out; pass a pre-resolved promotion into a tiny method | Restores bimorphic inlining and everything downstream | A design change, but usually a good one anyway |
| **Split the call site.** Resolve the rule once per request rather than per product, or branch on the flag at the top of the method so each branch has its own monomorphic site | Each site sees one type again | More code; the branch must be predictable |
| **Cache the promotion lookup** so the new path is not doing per-product I/O | Fixes the real cost if the regression is on the 2%, not the 98% | Topic 110, and a cache is a live-set decision (Topic 70) |
| **Remove the interface** if there will only ever be one implementation | `final` classes and non-virtual calls cannot be polluted | Only when the abstraction was never earning its keep |
| **Restart the pods after rolling back a flag** | Clears the deoptimized state | Operational awareness, not a code fix — but knowing to do it is worth a lot during an incident |

> **What you would put in the postmortem:** "A feature flag enabled for 2% of traffic
> introduced a second implementation at a call site that had been monomorphic since startup.
> C2's class-hierarchy-analysis-based inline was invalidated; the method was deoptimized and
> recompiled without the inline, which also disabled the downstream optimisations that
> depended on it. The regression affected 100% of traffic on the affected pods because
> compiled code is per-JVM, not per-request. Rollback did not restore performance; restart
> did. Action: measure inlining impact for new implementations at hot call sites, and treat
> 'a rollback that does not fix it but a restart does' as a JIT-state signature."

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — benchmarking without warm-up

**Wrong approach.** Two implementations of `orderflow`'s order-total calculation need
comparing. Someone writes the obvious benchmark:

```java
// DO NOT DO THIS. Every number it produces is meaningless.
public static void main(String[] args) {
    List<OrderLine> lines = TestOrders.lines(500);

    long start = System.nanoTime();
    for (int i = 0; i < 1_000_000; i++) {
        long total = OrderTotals.calculateV1(lines);
    }
    System.out.println("V1: " + (System.nanoTime() - start) / 1_000_000 + " ns/op");

    start = System.nanoTime();
    for (int i = 0; i < 1_000_000; i++) {
        long total = OrderTotals.calculateV2(lines);
    }
    System.out.println("V2: " + (System.nanoTime() - start) / 1_000_000 + " ns/op");
}
```

**Exact symptom.** One or more of these, and you cannot tell which:

- One or both results are **absurdly fast** — a small number of nanoseconds for work that
  obviously cannot be done that fast.
- **V2 looks faster than V1 purely because it ran second**, after the JVM had warmed up
  shared code — `List` iteration, `BigDecimal` arithmetic, the JDK internals both use.
- **Reversing the order of the two blocks reverses the result.** This is the diagnostic
  that proves the benchmark is measuring order, not code.
- Results vary by large factors between runs on an idle machine.
- The result does not reproduce anything you see in production.

**Root cause.** Four independent lies, and they compound:

1. **Cold JIT.** The first thousands of iterations run interpreted — roughly 10× slower. If
   your loop is short, that is most of your measurement.
2. **OSR.** The loop is compiled *while running* and the frame swapped mid-flight, so the
   average blends interpreted, C1 and C2 execution in a ratio determined by the loop count
   you happened to pick.
3. **Dead-code elimination.** `total` is never used. C2 can prove the call has no observable
   effect and delete the whole loop. **You measured an empty loop**, which is the source of
   the absurdly-fast numbers.
4. **Profile pollution between the two cases.** They share a JVM. Code common to both is
   compiled with a profile polluted by both, so **neither measurement reflects what happens
   when only one is deployed** — which is the situation you are trying to predict.

**Fix.** Use JMH. Not "consider using"; use it. The correct harness is in the Measurement
section, and the annotations that matter here are:

| Annotation | Defends against |
|---|---|
| `@Warmup(iterations = 5)` | Cold JIT and OSR — reach steady-state compiled code before recording |
| `@Fork(3)` | **Profile pollution.** Three separate JVMs; each `@Benchmark` gets a fresh compiler state |
| `Blackhole.consume(...)` | Dead-code elimination |
| `@State(Scope.Benchmark)` | Constant folding of inputs |

**And then check what you compiled**, because JMH's defences are not automatic proof:

```bash
java -jar target/benchmarks.jar OrderTotalBenchmark -prof perfasm      # Linux
java -jar target/benchmarks.jar OrderTotalBenchmark -prof gc
```

**The gap between the naive number and the JMH number is the lesson**, and Topic 77 makes
producing that gap a drill in its own right.

---

### Trap 2 — a call site monomorphic for a million calls goes bimorphic and stays degraded

**Wrong approach.** This is Example 2, stated as a general trap because it recurs in many
disguises. The common shapes:

- A feature flag introducing a second implementation of an interface at a hot call site.
- A test suite that runs all tests in one JVM, so a mock implementation pollutes the profile
  of code that in production only ever sees the real one.
- A strategy pattern where a rarely-used strategy is registered at startup, defeating
  class-hierarchy analysis for a strategy used 99.99% of the time.
- Migrating from one implementation to another with both live during the migration —
  and forgetting to remove the old one afterwards.

**Exact symptom.**

- **A uniform, modest slowdown across endpoints**, including endpoints that never touch the
  changed code.
- **GC metrics completely unchanged.** Allocation rate, live set, pauses: all normal.
- **CPU per request up**, latency up, throughput down.
- **A rollback does not fix it. A restart does.** This is the signature.
- Timing correlates with a deploy, a flag flip, or a class being loaded for the first time —
  which can be an arbitrary moment, long after the deploy, when some lazily-initialised bean
  is first touched.
- **Reproduces in production and not in staging**, because staging never warmed the call
  site to C2.

**Root cause.** C2 speculated. The speculation was violated. The recompiled version is
permitted fewer optimisations, and — critically — **losing the inline loses everything
downstream of the inline**: escape analysis, constant folding, branch elimination. The direct
cost of a virtual call is nanoseconds; the indirect cost is unbounded.

**Fix.**

1. **Confirm before acting.** `-XX:+PrintCompilation -XX:+TraceDeoptimization`, find the
   `made not entrant` and the reason. Do not act on the hypothesis alone; a slow new code
   path (an N+1 query) produces a superficially similar graph with a completely different
   distribution across traffic.
2. **Measure whether it actually matters.** Bimorphic call sites still inline. If the
   regression is within budget, the correct action is to record it and ship.
3. **Keep the new implementation small.** A callee that exceeds the inlining size budget
   turns a bimorphic site from "inlined twice" into "not inlined at all". Topic 75's
   `MaxInlineSize` / `FreqInlineSize` discussion is directly actionable here.
4. **Split the call site** so each site sees one type — branch on the flag once at the top,
   not per element.
5. **In tests, use `@Fork` and separate JVMs** for anything performance-sensitive. A shared
   test JVM is a profile-pollution machine.
6. **Consider `final`.** A `final` class or method cannot be overridden, so the call is not
   virtual and cannot be polluted. Do not sprinkle `final` everywhere as a performance
   measure — C2 already handles the common case via CHA — but for a genuinely hot,
   genuinely single-implementation type it is a legitimate and clear signal.

**What the fix proves:** compiled code is a shared, per-JVM resource with global state. **A
change that affects 2% of requests can degrade 100% of them**, and that is only intuitive
once you know the mechanism.

---

### Trap 3 — "the first thousand requests after deploy are slow", fixed with a bigger instance

**Wrong approach.** After every rolling deploy, `orderflow` shows a latency spike. p99 on
freshly started pods is far outside the Topic 65 baseline for the first minute or two, then
recovers to normal. The k8s readiness probe passes immediately, because it checks that the
HTTP port is open (Topic 121's trap).

The team's diagnosis: "the pods are resource-starved at startup." The fix: **double the CPU
and memory limits.**

**Exact symptom.**

- The spike is **smaller but still there** after the resource increase. Money spent, problem
  not solved.
- The spike duration correlates with **request count**, not with elapsed time. A pod that
  receives no traffic for five minutes and then gets traffic still spikes.
- **A pod that has served traffic does not spike**, no matter how long it has been up.
- Adding *more* replicas makes it **worse in aggregate**, because more cold JVMs enter the
  pool at once.
- GC metrics show normal young collections. There is no memory pressure.
- CPU is high during the spike — the interpreter and the compiler threads are both working.

**Root cause.** **Warm-up.** Every method starts interpreted. A method needs to run thousands
of times before C2 compiles it. Until then the pod is running an order of magnitude slower on
those paths. **The clock that matters is executions, not seconds**, which is exactly why more
CPU only partially helps: it speeds up the interpreter and the compiler threads, but it does
not reduce the number of executions required.

The readiness probe compounds it: the pod is declared ready and receives its full share of
traffic while still interpreted.

**Fix.** In increasing order of investment:

| Fix | What it addresses |
|---|---|
| **Warm up before declaring readiness.** On startup, exercise the hot paths — a few thousand synthetic calls through the real code paths — and only then let the readiness probe pass (Topic 121) | The core problem: get the executions done before real traffic arrives |
| **Slow-start / connection ramping at the load balancer.** Send a new pod a small share of traffic that increases over a minute | Spreads the cold period across fewer users |
| **Deploy more gradually.** One pod at a time, with a wait between | Never have a large fraction of the fleet cold at once |
| **AppCDS** (`-XX:SharedArchiveFile`) — a class-data-sharing archive from a training run | **Class loading**, which is a large part of Spring startup. **Does not address method profiles** |
| **`[JAVA 25]` AOT cache** — carries linkage and, in newer versions, method profiles across restarts | Attacks warm-up directly. **Verify what your build supports before planning around it** |
| **GraalVM native image** (Topic 83) | Eliminates warm-up entirely — and gives up C2's peak throughput, which for a 24/7 service is usually the wrong trade |
| **`-XX:TieredStopAtLevel=1`** | **Do not do this for a service.** See Trap 5 |

**Prove it before you fix it**, in one experiment:

```bash
# Start a pod, DO NOT send it traffic, wait 5 minutes, then send the baseline load.
# If the spike still happens, it is executions, not elapsed time. That is warm-up.
```

**What the fix proves:** JVM performance is a function of how much a specific code path has
executed, not of how long the process has been alive or how much CPU it has. That is a
genuinely different mental model from anything in Node, where V8's tiers warm up so much
faster that it rarely surfaces as an operational concern.

---

### Trap 4 — the code cache fills and the compiler shuts down

**Wrong approach.** Nothing is done wrong, exactly. `orderflow` grows: more auto-configuration,
more Hibernate entities, more Spring proxies (Topic 40), an APM agent instrumenting everything
(Topic 81), and a lot of lambdas (Topic 21), each of which generates a class. Nobody ever
looks at `-XX:ReservedCodeCacheSize`.

**Exact symptom.** Distinctive once you have seen it, baffling if you have not:

- **Performance falls off a cliff.** Not a gradual slope — a step change to roughly an order
  of magnitude slower, at some point during a long-running process's life.
- **Every endpoint is affected equally.**
- **GC is completely normal.** No memory alarms. Heap looks healthy.
- **CPU is pegged**, because everything is interpreting.
- **A restart fixes it completely**, and it comes back hours or days later.
- Sometimes a log line — "CodeCache is full. Compiler has been disabled." — which is easy to
  miss in a busy log and is often the only direct evidence.
- Before the cliff, there may have been a period of **degraded and unstable** performance as
  the code-cache sweeper flushed and methods were recompiled repeatedly.

**Root cause.** The code cache is fixed-size native memory reserved at startup, unrelated to
`-Xmx`. When it fills, the JVM disables the compiler. **From that moment, anything not
already compiled runs interpreted, forever.**

**Fix.**

1. **Check it — this takes ten seconds and almost nobody does it:**
   ```bash
   jcmd $(pgrep -f orderflow) Compiler.codecache
   java -XX:+PrintFlagsFinal -version | grep -E "ReservedCodeCacheSize|SegmentedCodeCache|UseCodeCacheFlushing"
   ```
   Do it **after a long soak under load**, not at startup. At startup it is always fine.
2. **Alert on it.** `jvm.memory.used{area="nonheap",id="CodeCache"}` — or the segmented
   equivalents — via Micrometer (Topic 118). **This should be on every JVM dashboard and
   almost never is.**
3. **Raise `-XX:ReservedCodeCacheSize`** if the soak shows pressure. It is native memory, so
   it counts against the container limit (Topic 82) — raise the container limit with it or
   you have traded a JIT problem for an OOM kill.
4. **Reduce the code**: fewer agents, fewer proxies, less reflection. Usually not practical,
   but worth knowing which of these are contributing.
5. **Confirm `-XX:+UseCodeCacheFlushing`** is on, so the sweeper can reclaim cold methods
   rather than the cache simply filling.

**What the fix proves:** the JVM has native-memory regions outside `-Xmx` that can be
exhausted independently, with symptoms that look nothing like a memory problem. Topic 80 is
the general treatment; the code cache is the one that masquerades as a CPU problem.

---

### Trap 5 — `-XX:TieredStopAtLevel=1` cargo-culted into a long-running service

**Wrong approach.** Someone reads a blog post — a true one, about CLI tools or build steps —
saying `-XX:TieredStopAtLevel=1` dramatically improves startup time. It goes into
`orderflow`'s `JAVA_TOOL_OPTIONS` to fix the Trap 3 deploy spike.

**Exact symptom.**

- **The deploy spike genuinely improves.** The change appears to work, which is why it
  survives review.
- **Steady-state throughput drops substantially and permanently.** Requests per CPU-second
  fall. The service needs more replicas for the same load.
- **p99 at steady state is worse** than the recorded Topic 65 baseline, on every endpoint.
- **No deoptimizations at all** in `PrintCompilation`, and **no tier-4 compilations** —
  which is the direct evidence.
- Six months later nobody remembers the flag is there, and the service is quietly costing
  substantially more to run than it needs to.

**Root cause.** `TieredStopAtLevel=1` caps compilation at C1. C1 is fast to compile and
produces reasonable code, but it does **no profiling and no speculative optimisation**: no
profile-guided inlining, no escape analysis, no scalar replacement (Topic 75), no aggressive
loop optimisation. **You have permanently traded peak throughput for warm-up time.**

That trade is *correct* for a process that lives for seconds — a CLI tool, a Maven plugin, a
short test JVM — where C2 would never pay for itself. It is *wrong* for a process that lives
for weeks.

**Fix.**

1. **Remove the flag.** Then fix warm-up the right way (Trap 3): readiness-gated warm-up,
   load-balancer slow start, AppCDS or an AOT cache.
2. **A/B it against the recorded baseline**, both arms, three runs each, comparing steady-state
   p99 and throughput as well as the deploy spike. **Report both numbers.** The flag improves
   one and damages the other, and a decision that only looks at one is not a decision.
3. **Audit `JAVA_TOOL_OPTIONS` and every Dockerfile** for other flags nobody can justify.
   ```bash
   jcmd $(pgrep -f orderflow) VM.flags -all | tr ' ' '\n' | grep -E '^-XX' | sort
   ```
   For each flag, ask: what measurement motivated it, and when was that measurement taken?
   A flag with no answer is a liability.

**The general rule:** *a flag that improves startup at the cost of steady state is correct
for short-lived processes and wrong for services. Know which one you are running, and never
copy a flag without knowing which case the source was talking about.*

---

## Hands-on proof

### Setup

```bash
mkdir -p ~/jvm-lab/logs && cd ~/jvm-lab
java -version
javac -d out --release 21 src/main/java/com/orderflow/lab/jit/*.java
```

### Proof 1 — establish your JIT configuration

```bash
java -XX:+PrintFlagsFinal -version | grep -E "TieredCompilation|TieredStopAtLevel|CICompilerCount|ReservedCodeCacheSize|SegmentedCodeCache|TypeProfileWidth|MaxInlineSize|FreqInlineSize|MaxInlineLevel"
java -XX:+PrintFlagsFinal -version | grep -E "Tier[0-9]+(Invocation|Compile|BackEdge)Threshold"
```

**WHAT TO LOOK FOR:** write these down before doing anything else.

| What you see | What it means |
|---|---|
| `TieredCompilation = true`, `TieredStopAtLevel = 4` | The normal configuration. Everything in this document applies. |
| `TieredStopAtLevel` less than 4 | **Someone capped your compiler.** Find out who and why before measuring anything — Trap 5. |
| `CICompilerCount` of 2 or fewer | You are on a small machine or a CPU-limited container. Compilation will be slow, which makes warm-up longer. Topic 82. |
| `TypeProfileWidth = 2` | Confirms why bimorphic is the boundary: there is nowhere to record a third type. |
| Threshold values in the hundreds/thousands | The order of magnitude that explains warm-up. **Your numbers, not mine.** |

### Proof 2 — watch a method climb the tiers

```bash
java -XX:+PrintCompilation -cp out com.orderflow.lab.jit.CallSitePollution 20000000 1 \
  2>&1 | grep -E 'CallSitePollution' | head -20
```

**WHAT TO LOOK FOR:** the tier column across successive lines for the same method.

| What you see | What it means |
|---|---|
| Tier 3, then tier 4, for the same method | **The normal path**, observed. Profiled C1 first, then C2. |
| Tier 1 only, and no further compilation | The method is trivial — C2 has nothing to add, so the JVM stops at C1. Common for getters. |
| Tier 2 appearing | The C2 queue was backed up, so the JVM used lightly-profiled C1 to escape the interpreter sooner. More common on a busy or CPU-limited machine. |
| A `%` in the attributes | **OSR.** The loop got hot before the method's invocation count did. |
| No compilation at all for your method | It did not run enough times. Increase the iteration count — **and note how many you needed.** That number is your warm-up threshold, felt rather than read. |

### Proof 3 — prove warm-up exists, with the crudest possible instrument

Deliberately using the *wrong* tool, to see the effect the right tool hides:

```java
package com.orderflow.lab.jit;

public class WarmupShape {
    static long work(long n) {
        long acc = 0;
        for (long i = 0; i < 1000; i++) acc += (i * n) % 7;
        return acc;
    }

    public static void main(String[] args) {
        // Print per-batch timing. NOT a benchmark - a SHAPE, which is all we want.
        for (int batch = 0; batch < 40; batch++) {
            long t0 = System.nanoTime();
            long sink = 0;
            for (int i = 0; i < 10_000; i++) sink += work(i);
            long ns = System.nanoTime() - t0;
            System.out.println(batch + " " + ns / 10_000 + " ns/call  (sink=" + sink + ")");
        }
    }
}
```

**WHAT TO LOOK FOR:** the *shape* of the sequence, never the absolute values.

| What you see | What it means |
|---|---|
| Early batches much slower, then a step down, then flat | **Warm-up, visible.** The step is a tier transition. This is the shape JMH's warm-up iterations discard. |
| Two distinct steps | Tier 0 → 3 → 4, both visible. |
| Noisy with no clear trend | Your machine is busy, or the work is too small relative to timing overhead. Increase the inner loop. |
| Instantly flat and implausibly fast | **Dead-code elimination got you anyway**, despite the `sink`. Proof that you cannot outsmart C2 with a hand-rolled harness — which is Topic 77's thesis. |

**This is not a benchmark and must never be reported as one.** It is a demonstration that the
first N executions are different from the rest. Every number it prints is contaminated by
exactly the four problems in Trap 1.

### Proof 4 — see a deoptimization, named

```bash
java -XX:+UnlockDiagnosticVMOptions -XX:+PrintCompilation -XX:+TraceDeoptimization \
  -cp out com.orderflow.lab.jit.CallSitePollution 50000000 5000000 2>&1 \
  | grep -iE 'not entrant|Uncommon trap|DEOPT|reason' | head -30
```

**WHAT TO LOOK FOR:** a reason string.

| What you see | What it means |
|---|---|
| `class_check` | The receiver was not the speculated class. **The canonical profile-pollution deopt.** |
| `bimorphic` | A site that had two types saw a third. |
| `unstable_if` | A branch believed never-taken was taken. Often the first execution of an error path. |
| `unloaded` or `uninitialized` | Compiled code referenced a class not yet loaded. Very common at startup — mostly benign. |
| `null_check` | First null on a path that had never seen one. |
| `make_not_entrant` as the action | The compiled version is retired; recompilation follows. |
| `make_not_compilable` | **Bad.** The JVM has given up compiling this method. It will stay slow. |
| Nothing at all | Your JDK spells it differently, or the flag needs `-XX:+UnlockDiagnosticVMOptions` first. **Learn your JDK's actual output before writing greps** — this is the standing rule for every log in Phase 8. |

### Proof 5 — check the code cache

```bash
jcmd $(pgrep -f orderflow) Compiler.codecache
jcmd $(pgrep -f orderflow) Compiler.codelist | wc -l      # how many compiled methods
```

**WHAT TO LOOK FOR:** free space per segment, **after a soak under load, not at startup**.

| What you see | What it means |
|---|---|
| Ample free in all segments after a long soak | Healthy. Record the numbers in the baseline so you have a trend. |
| One segment nearly full | Read the Trap 4 table for which segment means what. |
| Very high compiled-method count | A large application, an agent, heavy lambda or proxy use. Not a problem by itself — a reason to watch the cache. |
| Evidence of flushing / sweeper activity | Under pressure. Leading indicator of the cliff. |

### Proof 6 — the `-Xint` control

```bash
time java       -cp out com.orderflow.lab.jit.CallSitePollution 5000000 5000000
time java -Xint -cp out com.orderflow.lab.jit.CallSitePollution 5000000 5000000
```

**WHAT TO LOOK FOR:** the ratio.

| What you see | What it means |
|---|---|
| `-Xint` is roughly an order of magnitude slower | **The value of the JIT, measured on your machine in one command.** This is the number to quote when someone asks what the JIT is worth. |
| The ratio is small | Your workload is dominated by something the JIT cannot help — I/O, allocation, or a syscall. That is itself a useful finding. |
| `-Xint` is faster | Essentially impossible for compute; if you see it, you are measuring startup rather than steady state. Increase the workload. |

---

## Failure drill

**Mandatory.** Do not read "how to read it" until you have produced the deoptimization
yourself and written down what you saw.

### The assignment, restated from the master plan

> Make a call site monomorphic, warm it, then introduce a second implementation. Capture
> `-XX:+UnlockDiagnosticVMOptions -XX:+PrintCompilation -XX:+TraceDeoptimization` and find
> the deopt. Measure the before/after with JMH forks.

### Step 0 — establish the control and learn your JDK's output

```bash
java -version
java -XX:+PrintFlagsFinal -version | grep -E "TieredStopAtLevel|TypeProfileWidth|MaxInlineSize|FreqInlineSize"

# Learn what YOUR JDK prints before writing a single grep.
java -XX:+UnlockDiagnosticVMOptions -XX:+PrintCompilation -XX:+TraceDeoptimization \
  -version 2>&1 | head -30
```

Write down: your tier thresholds, `TypeProfileWidth`, and the **exact spelling** of the
deoptimization lines your JDK emits. Every grep below is a starting point to adapt.

### Step 1 — build the standalone reproduction first

Use `CallSitePollution` from Example 1. **Standalone before service** — if you cannot see the
deopt in a 40-line program, you will not find it in Spring.

```bash
java -XX:+UnlockDiagnosticVMOptions -XX:+PrintCompilation -XX:+TraceDeoptimization \
  -cp out com.orderflow.lab.jit.CallSitePollution 50000000 20000000 \
  2>&1 | tee logs/drill-standalone.log

grep -E 'CallSitePollution' logs/drill-standalone.log
grep -iE 'not entrant|Uncommon trap|reason=' logs/drill-standalone.log
```

**Do not proceed until you have found a `made not entrant` for `total` and a named reason.**
If you cannot, increase the phase-1 iteration count, verify phase 1 completed before phase 2
began (the print statements tell you), and check `TieredStopAtLevel` is 4.

### Step 2 — put it inside `orderflow`

Implement Example 2's `PriceRule` / `StandardPriceRule` / `PromotionalPriceRule`, with the
call site inside the catalogue read path — the endpoint carrying 70% of baseline traffic.

**Critically: make the second implementation loadable but not initially loaded**, so you can
introduce it at a controlled moment:

```java
@Service
public class CatalogQueryService {

    private volatile PriceRule priceRule = new StandardPriceRule();

    /** Flipped by an admin endpoint or a scheduled task DURING the load run. */
    public void enablePromotions(PriceRule promotional) {
        this.priceRule = promotional;
    }
}
```

> **Why `volatile`:** the field is written by one thread and read by request threads. Without
> it, visibility is not guaranteed (Topic 86). It also stops the JIT constant-folding the
> field, which would change what you are measuring. **Note that this is itself a JIT-relevant
> decision** — a `static final` field would be folded to a constant and the whole experiment
> would behave differently.

### Step 3 — run it under load, with compilation logging

```bash
# Log to a file. PrintCompilation on a Spring app under load is very high volume.
docker compose up -d orderflow  # with:
#   -XX:+UnlockDiagnosticVMOptions -XX:+PrintCompilation -XX:+TraceDeoptimization
#   -XX:+LogCompilation -XX:LogFile=/var/log/orderflow/compile.log
#   -Xlog:gc*:file=/var/log/orderflow/gc.log:time,uptime,level,tags

# Phase 1: warm the call site with the UNCHANGED Topic 65 load profile.
k6 run --duration 5m load/orderflow-baseline.js

# Confirm the call site reached tier 4 BEFORE polluting it.
grep -E 'CatalogQueryService::page|StandardPriceRule::effectivePrice' \
     /var/log/orderflow/compile.log | tail -20

# Phase 2: flip the implementation WHILE the same load continues.
k6 run --duration 5m load/orderflow-baseline.js &
sleep 60
curl -X POST localhost:8080/internal/admin/enable-promotions
wait
```

**The load profile must not change between phases.** The entire point is that the *code* got
slower, not that the *work* got harder.

### Step 4 — what to capture

Capture all six. The last two are what turn an observation into evidence.

1. The **timestamp** of the flip.
2. Every compilation of the hot method, **with tier and compile id**, before and after.
3. The `made not entrant` event and its **timestamp relative to the flip**.
4. The **deoptimization reason** from `TraceDeoptimization`.
5. **The GC log across the same window** — to prove GC did not change, which is what
   exonerates Topic 70 and makes your JIT diagnosis credible.
6. **The k6 percentiles per endpoint, before and after the flip**, including endpoints that
   do not call `PriceRule` at all.

```bash
# Adapt the field names to YOUR JDK, learned in Step 0.
grep -E 'CatalogQueryService::page' /var/log/orderflow/compile.log | awk '{print $1, $2, $4}'
grep -iE 'not entrant|Uncommon trap|reason=' /var/log/orderflow/compile.log | tail -40
grep -c 'Pause Full' /var/log/orderflow/gc.log
```

### Step 5 — how to read it

| What you see | What it means |
|---|---|
| Tier 4 before the flip, `made not entrant` within seconds after, tier 4 again with a new compile id | **The full deoptimization/recompilation cycle under production-shaped load.** This is the drill's target result. |
| Deopt reason `class_check` or an invalidated CHA dependency | Profile pollution, named. **Cite this exact string in your write-up.** |
| p99 up on endpoints that never call `PriceRule` | **The most important observation in the drill.** Compiled code is per-JVM, so pollution is not confined to the polluted path. |
| GC numbers flat across the window | GC exonerated. Without this, your diagnosis is a guess. |
| p99 does not recover when you flip the implementation back | **The signature.** Rollback does not restore JIT state; only a restart does. Verify by restarting the pod. |
| No deopt, but p99 up | The new implementation is simply doing more work — likely an N+1 (Topic 50). Check the distribution: does it hurt only the promotional traffic, or everyone? **JIT pollution hurts everyone.** |
| Deopts continuing at a steady rate | A **deopt storm**. Something violates a speculation repeatedly. Look for `make_not_compilable` and for a value oscillating between types. |
| Compile log so large it fills the disk | Expected. Rotate it, or capture a narrow window around the flip. |

### Step 6 — measure it properly, with JMH forks

The load test tells you the *service* got slower. JMH tells you *how much* the call site
itself costs in each state, isolated from everything else.

The harness is in the Measurement section. The essential design:

- **Three separate benchmarks**: monomorphic, bimorphic, megamorphic (three implementations).
- **`@Fork(3)`** so each runs in fresh JVMs and cannot pollute the others. **A single fork
  would make all three bimorphic or worse and the experiment would measure nothing** — which
  is itself the most instructive possible failure of this drill.
- **`-prof perfasm` on Linux**, to see whether the call was inlined or is a real virtual call.

**Run it once with `@Fork(1)` deliberately**, and compare. The convergence of the three
results under a single fork is the mechanism, demonstrated inside the measuring instrument.

### What the drill proves

1. **Deoptimization is real, observable, and nameable.** You have the reason string.
2. **The cost is not the deopt — it is the recompiled code you keep.** Rollback does not fix
   it; restart does.
3. **Compiled code is shared state at JVM scope.** A change affecting 2% of requests degraded
   100% of them.
4. **Your measuring instrument is subject to the same effect.** A single-fork JMH run cannot
   measure monomorphic dispatch, because running the other benchmarks pollutes it. That is
   why `@Fork(3)` is not a nicety.

---

## Measurement

### The instrument for this topic

**Two instruments, and they answer different questions.**

| Question | Instrument |
|---|---|
| "What did the compiler decide, and when?" | `-XX:+PrintCompilation`, `-XX:+TraceDeoptimization`, `-XX:+PrintInlining`, `-XX:+LogCompilation` + JITWatch |
| "How much does that cost, in nanoseconds per operation?" | **JMH, with `@Fork(3)`** |

You need both. The compilation log tells you *what happened*; only a benchmark tells you
*whether it matters*. **A deoptimization you cannot measure the cost of is a curiosity, not a
finding.**

### Why a naive `System.nanoTime()` measurement is WRONG here

This is the topic where the answer is most emphatic, because **the thing you are trying to
measure is the thing that breaks the measurement.**

```java
// DO NOT DO THIS. It is wrong in four independent ways, and you cannot tell which.
long start = System.nanoTime();
for (int i = 0; i < 1_000_000; i++) {
    result = rule.apply(i);
}
System.out.println((System.nanoTime() - start) / 1_000_000 + " ns/op");
```

1. **Cold JIT.** The first thousands of iterations run interpreted, roughly 10× slower. If
   the loop is short, that is most of your measurement.
2. **On-stack replacement.** The loop is compiled *while running* and swapped mid-flight, so
   the average blends tier 0, tier 3 and tier 4 execution in a ratio determined by the loop
   count you happened to choose. **Change the loop count and the answer changes**, which is
   the diagnostic that proves the harness is broken.
3. **Dead-code elimination.** If `result` is unused, C2 deletes the loop. You measure
   nothing, and "nothing" times out at a few nanoseconds — the source of every absurd
   benchmark number ever posted.
4. **Constant folding.** `i` is a known sequence and `rule` may be a constant; C2 has
   enormous freedom.

**And the reason specific to this topic, which is the killer:** the compiled code's quality
depends on the profile, and the profile depends on **everything else that ran in this JVM
first**. Running case A then case B in one `main` gives B a profile polluted by A. **The
second measurement is not a measurement of B; it is a measurement of B-after-A**, and that is
not a configuration you will ever deploy.

Topic 77 is the full treatment. **Read it before you write any benchmark you intend to act
on.**

### The correct harness

```java
package com.orderflow.bench;

import com.orderflow.catalog.*;
import org.openjdk.jmh.annotations.*;
import org.openjdk.jmh.infra.Blackhole;

import java.util.concurrent.TimeUnit;

/**
 * Measures the cost of a call site in three profile states.
 *
 * @Fork(3) IS THE POINT OF THIS BENCHMARK. With a single fork, running the
 * bimorphic and megamorphic benchmarks would pollute the profile of the
 * monomorphic one, and all three would converge -- measuring nothing.
 */
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@Warmup(iterations = 10, time = 1)          // generous: we must reach C2 steady state
@Measurement(iterations = 10, time = 1)
@Fork(3)                                    // three fresh JVMs per benchmark method
@State(Scope.Benchmark)
public class CallSiteShapeBenchmark {

    private PriceRule[] mono;
    private PriceRule[] bi;
    private PriceRule[] mega;
    private ProductSummary product;
    private CustomerTier tier;

    @Setup(Level.Trial)
    public void setUp() {
        product = TestProducts.one();
        tier    = CustomerTier.STANDARD;

        // Same array length everywhere, so loop overhead is identical across
        // benchmarks and only the RECEIVER TYPE MIX differs.
        mono = fill(new PriceRule[]{ new StandardPriceRule() });
        bi   = fill(new PriceRule[]{ new StandardPriceRule(), new PromotionalPriceRule() });
        mega = fill(new PriceRule[]{ new StandardPriceRule(), new PromotionalPriceRule(),
                                     new ClearancePriceRule(), new StaffPriceRule() });
    }

    private static PriceRule[] fill(PriceRule[] kinds) {
        PriceRule[] a = new PriceRule[1024];
        for (int i = 0; i < a.length; i++) a[i] = kinds[i % kinds.length];
        return a;
    }

    @Benchmark
    public void monomorphic(Blackhole bh) {
        for (PriceRule r : mono) bh.consume(r.effectivePrice(product, tier));
    }

    @Benchmark
    public void bimorphic(Blackhole bh) {
        for (PriceRule r : bi) bh.consume(r.effectivePrice(product, tier));
    }

    @Benchmark
    public void megamorphic(Blackhole bh) {
        for (PriceRule r : mega) bh.consume(r.effectivePrice(product, tier));
    }
}
```

```bash
./mvnw -Pbench clean verify

# The real run.
java -jar target/benchmarks.jar CallSiteShapeBenchmark -rf json -rff callsite.json

# See whether the call was actually inlined. Linux; needs perf and hsdis.
java -jar target/benchmarks.jar CallSiteShapeBenchmark.monomorphic -prof perfasm

# The deliberate mistake, for comparison. Run this SECOND and explain the difference.
java -jar target/benchmarks.jar CallSiteShapeBenchmark -f 1
```

| Annotation / flag | What it defends against |
|---|---|
| `@Fork(3)` | **Profile pollution between benchmark methods.** The single most important line in this harness for this topic |
| `@Warmup(iterations = 10)` | Cold JIT and OSR. Generous, because we specifically need steady-state C2 code |
| `Blackhole.consume(...)` | Dead-code elimination. **Note it also inhibits some optimisations — Topic 75's trap** |
| `@State(Scope.Benchmark)` | Inputs live in fields JMH controls, so C2 cannot constant-fold them |
| Identical array lengths | Loop overhead is constant across the three, so the difference is dispatch and inlining |
| `-prof perfasm` | Shows the generated assembly — the only direct proof of whether the call was inlined |
| `-f 1` (the mistake) | Demonstrates the failure this benchmark exists to avoid |

**WHAT TO LOOK FOR:** three numbers, their ordering, and the single-fork comparison.

| What you see | What it means |
|---|---|
| monomorphic < bimorphic < megamorphic, clearly separated | **The expected result.** Now you can quantify what a polluted call site costs *in your workload*. |
| monomorphic ≈ bimorphic, megamorphic clearly worse | Also common and correct: bimorphic inlining works well. **The cliff is at three types, and this result names where the cliff is.** |
| All three roughly equal | Either the callee body dominates dispatch cost (likely if `effectivePrice` does real work), or something defeated the experiment. Check `-prof perfasm`. **A callee whose body dominates is a legitimate finding: dispatch does not matter here.** |
| Confidence intervals overlap | **You measured nothing.** Increase iterations, reduce machine noise, add forks. Never report a point estimate whose interval overlaps its comparator. |
| **With `-f 1`, all three converge** | **The mechanism, demonstrated inside the instrument.** One JVM means one profile means one shape for all three call sites. This is why `@Fork(3)` exists, and it is the most memorable result in this document. |
| `perfasm` shows an inlined body for monomorphic and a `call` for megamorphic | Direct assembly-level confirmation. This is as conclusive as it gets. |

### Reading `PrintInlining`, briefly (Topic 75 is the full treatment)

```bash
java -XX:+UnlockDiagnosticVMOptions -XX:+PrintInlining \
     -cp out com.orderflow.lab.jit.CallSitePollution 50000000 5000000 2>&1 \
  | grep -A2 -B2 'effectivePrice'
```

| Message you see | What it means |
|---|---|
| `inline (hot)` | Inlined. Everything downstream is possible. |
| `too big` / `hot method too big` | The callee's bytecode exceeds the size budget. **Topic 75's central lever** — shrink the method. |
| `not inlineable` | Native, abstract with multiple implementations, or otherwise ineligible. |
| `virtual call` | The call site is polymorphic. **This is the profile-pollution outcome, seen from the inlining side.** |
| `callee is too large` / `inlining too deep` | Depth or budget limits (`MaxInlineLevel`). |
| `no static binding` | The receiver type could not be determined. |

### The numbers to track continuously, in production

Not during a drill — permanently:

| Number | Where from | Why |
|---|---|---|
| **Code cache used / free**, per segment | Micrometer `jvm.memory.used{area="nonheap"}`, or `jcmd Compiler.codecache` | Trap 4 is a cliff. **This should be on every JVM dashboard and almost never is** |
| **Compilation queue length / compiler thread CPU** | JFR, or `jcmd Compiler.queue` | A persistently long queue means warm-up is being delayed, often by a CPU quota |
| **Time-to-baseline-p99 after a pod starts** | Your own deploy dashboard | Quantifies warm-up. Makes Trap 3 a number instead of an anecdote |
| **Deoptimization rate** | JFR's compilation and deoptimization events | A steady rate is a deopt storm; a spike correlates with a flag flip or a class first loaded |

JFR (Topic 78) is the right permanent instrument here — low enough overhead to leave on, and
it records compilation and deoptimization events without `PrintCompilation`'s volume.

---

## Practice exercises

### 1 — Easy: build your own tier and threshold fact sheet

Produce a one-page reference, measured on **your** runtime:

1. Print every tier threshold flag. Record the values.
2. Using `CallSitePollution`, find empirically how many invocations it takes for `total` to
   reach tier 3, and then tier 4. Compare against the printed thresholds and explain any gap
   (hint: the thresholds are adjusted based on compiler queue length, and OSR uses back-edge
   counters, not invocation counters).
3. Time the same workload under: default, `-XX:TieredStopAtLevel=1`, `-XX:-TieredCompilation`,
   `-Xint`, and `-Xcomp`. Rank them and **explain each result mechanically** — especially why
   `-Xcomp` is slower than the default, which surprises most people.
4. Print `TypeProfileWidth`, `MaxInlineSize`, `FreqInlineSize`, `MaxInlineLevel`,
   `ReservedCodeCacheSize`. Write one sentence per flag on what it controls.
5. Run `jcmd <pid> Compiler.codecache` on a freshly started `orderflow` and again after a
   ten-minute load run. Record both.

Commit it to `/docs/java/baselines/jit-facts-<jdk-version>.md`. **It expires when the JDK
changes.** Say so in the file.

### 2 — Medium: the audit (combines Topics 01, 21, 22, 25, 40, 42, 65, 68, 69, 70)

Audit `orderflow` for JIT hazards. Deliverable: a ranked list with evidence.

1. **Find every interface with more than one implementation that is called on a hot path.**
   Start from the k6 scenario mix: what does the 70% catalogue path actually call? For each
   interface: how many implementations are *loaded* in production, and how many are loaded in
   staging? **A difference between the two is a reproducibility hazard.**
2. **Find every `@Bean` that is lazily initialised** (`@Lazy`, or created on first use). Each
   is a class-load event at an arbitrary moment that can invalidate CHA-based inlines. List
   them and note which are on hot paths.
3. **Measure warm-up.** Start a pod, send no traffic for five minutes, then run the baseline
   load and record time-to-baseline-p99. Repeat sending traffic immediately. **The difference
   between the two is the elapsed-time component; the similarity is the executions
   component.**
4. **Check the code cache after a soak.** Record usage per segment against
   `ReservedCodeCacheSize`. Count compiled methods with `Compiler.codelist`.
5. **Audit every JIT-related flag** in `JAVA_TOOL_OPTIONS`, the Dockerfile, and the k8s
   manifest. For each: what measurement motivated it, and when? Flag anything unjustified —
   especially `TieredStopAtLevel`.
6. **Count the lambdas and dynamic proxies.** Every lambda call site generates a class at
   runtime (Topic 21); every `@Transactional`/`@Cacheable` bean gets a CGLIB subclass
   (Topic 40). Both contribute to code-cache pressure and to method count. **Do not conclude
   anything alarming — the point is to know the magnitude.**
7. Write findings as: hazard, evidence, likelihood, impact, fix, effort. **Sort by impact ×
   likelihood.** Then draw a line where investigating stops being worth it and defend the
   line.

### 3 — Hard: production simulation — quantify profile pollution end to end

1. **Re-run the Topic 65 baseline** and confirm ±10%. Gate rule; nothing downstream is
   measurable otherwise.
2. **Instrument** with `-XX:+LogCompilation` to a rotating file, plus GC and safepoint logs on
   the same clock.
3. **Establish the warm steady state**: run baseline load for ten minutes, confirm the hot
   call sites reached tier 4, and record per-endpoint p50/p95/p99/p999.
4. **Predict, before flipping anything**, and write it down:
   - Which methods will be made not entrant?
   - What deoptimization reason will appear?
   - What will happen to p99 on `GET /products` (the polluted path)?
   - What will happen to p99 on `GET /orders` (an unpolluted path)?
   - What will happen to allocation rate and live set? (Topic 70 — and the correct
     prediction is "nothing much", which is what exonerates GC.)
   - Will rolling back the flag restore performance?
5. **Flip the implementation mid-run** and continue the unchanged load for ten more minutes.
6. **Capture everything** from Step 4 of the failure drill.
7. **Roll back the flag without restarting** and run ten more minutes. **Then restart and run
   ten more.** Three-phase result: polluted, rolled-back, restarted.
8. **Compare against every prediction and explain every gap.** A prediction you can explain
   afterwards is worth more than one that happened to be right.
9. **Now do it properly with JMH**: quantify the per-call cost of monomorphic vs bimorphic vs
   megamorphic dispatch for this specific interface, with `@Fork(3)`, and reconcile the
   microbenchmark delta with the end-to-end p99 delta. **They will not match, and explaining
   why is the hardest and most valuable part of this exercise** — the microbenchmark measures
   dispatch in isolation; the service measures dispatch plus every optimisation that was lost
   downstream of the inline, minus the fraction of request time that is actually Postgres.
10. **Write the postmortem** you would file: timeline, evidence with exact log lines, root
    cause with the named deopt reason, the distinguishing observation (rollback vs restart),
    blast radius (why unpolluted endpoints regressed), fix, and the detection you would add.
    Commit it to `/docs/java/baselines/`.

**A legitimate and valuable outcome is "the regression was within noise".** Bimorphic
inlining often costs very little. Reporting that honestly — with the JMH numbers and the
confidence intervals that support it — is a stronger result than manufacturing a dramatic
one, and it is the finding that stops the team from over-engineering around a non-problem.

---

## Interview questions

### Q1 — "Why is our service slow for the first minute after a deploy?"

**MID-LEVEL answer:** "The JVM needs to warm up — classes get loaded, caches fill, connection
pools initialise. It settles down after a bit. We could pre-warm the caches or give the pods
more resources."

**SENIOR answer:** "There are three separate warm-up mechanisms and they have different fixes,
so I'd want to know which dominates before spending anything.

**One — JIT warm-up, which is usually the biggest.** Every method starts interpreted, roughly
an order of magnitude slower than compiled code. A method needs to run **thousands of times**
before C2 compiles it — the thresholds are printable flags and they're in that range. The
critical detail is that **the clock is executions, not elapsed time.** A pod that sits idle
for ten minutes is exactly as cold as one that just started. That's the test I'd run first:
start a pod, send no traffic for five minutes, then load it. If it still spikes, it's
executions, and no amount of CPU or waiting fixes it.

**Two — class loading.** Spring loads thousands of classes, and each is loaded, verified,
linked and initialised lazily on first use. AppCDS with a class-data-sharing archive from a
training run addresses this specifically. On newer JDKs the AOT cache goes further and can
carry method profiles across restarts — I'd check what our build actually supports rather
than assume.

**Three — everything else.** Connection pools sizing up, caches empty, DNS cold, TLS sessions
not resumed. Real, usually smaller, and easy to fix with an explicit warm-up.

**What I'd actually do:** gate the readiness probe on warm-up. On startup, exercise the hot
paths a few thousand times through the real code — not a health-check stub, the actual query
path — and only then let the probe pass. That directly attacks the executions problem.
Alongside that, load-balancer slow-start so a new pod ramps up rather than taking a full share
immediately, and deploying one pod at a time so we never have a large fraction of the fleet
cold at once.

**What I'd specifically not do is add `-XX:TieredStopAtLevel=1`.** It genuinely improves
warm-up, because it caps compilation at C1 — but it also means C2 never runs, so we
permanently give up profile-guided inlining, escape analysis and every speculative
optimisation. That's the right trade for a CLI tool that lives for two seconds and the wrong
one for a service that lives for weeks. Every blog post recommending it is talking about the
first case."

**What separates them:** the mid answer names warm-up and stops. The senior answer
**decomposes it into three mechanisms with three different fixes**, identifies the one
diagnostic experiment that discriminates them, knows the clock is executions rather than time,
proposes readiness-gated warm-up rather than more hardware, and pre-empts the most common
wrong fix with a mechanical explanation of why it is wrong.

**Interviewer's follow-up:** *"How would you measure whether the warm-up actually worked?"* —
Time-to-baseline-p99 after a pod starts, as a tracked metric on the deploy dashboard. Start a
pod, send the recorded Topic 65 baseline load, and measure how long until p99 is within 10% of
the baseline. That's a number I can put on a graph and watch across releases, and it turns
"deploys feel bad" into something with a trend line. I'd also check `PrintCompilation` on one
pod to confirm the hot methods are actually reaching tier 4 during the warm-up phase rather
than during real traffic.

---

### Q2 — "What is deoptimization and when does it happen?"

**MID-LEVEL answer:** "It's when the JIT undoes an optimisation because an assumption turned
out to be wrong, and the code goes back to being interpreted. It happens when the runtime
behaviour changes from what the compiler expected."

**SENIOR answer:** "C2 is a **speculative** optimiser: it compiles based on the profile it
observed, not on what's provably true. If a call site only ever saw one receiver class, it
inlines that implementation directly. If a branch was never taken, it may not emit code for
it. Those are assumptions.

Each assumption is protected by an **uncommon trap** — a cheap check that, when it fails,
transfers execution out of the compiled frame back into the interpreter, reconstructing the
interpreter's locals and operand stack from the compiled frame using debug info the compiler
recorded for that program counter. That transfer is deoptimization.

**The important part is what it costs, and it's not the transfer.** The transfer happens once.
The compiled method is marked **not entrant**, the wider profile is recorded, and the method
is recompiled later with **fewer permitted speculations**. That slower version is the one you
keep, until restart.

The reasons I'd expect to see, and I'd read them with `-XX:+TraceDeoptimization`:
`class_check` when a receiver isn't the speculated class — the classic profile-pollution
event; `unstable_if` when a branch believed never-taken is taken; `null_check` on the first
null on a path that had never seen one; `unloaded` when compiled code references a class not
yet loaded, which is common and mostly benign at startup. And the actions matter too —
`make_not_entrant` means recompile, but **`make_not_compilable` means the JVM has given up on
this method entirely**, and that method is slow forever.

**The Java-specific mechanism I'd want to mention is class-hierarchy analysis.** If only one
implementation of an interface is loaded, C2 can inline the call with **no type check at all**,
protected by a dependency instead. If a second implementation is ever *loaded* — not even
called, just loaded — every compiled method carrying that dependency is invalidated. So a
lazily-initialised bean, or a reflective instantiation, can deoptimize a hot path that has
nothing to do with it, at an arbitrary moment. That has no counterpart in V8's hidden-class
speculation, and it's the reason production and staging can behave differently with identical
code.

**The operational signature** I'd teach a team: a performance regression that a rollback does
not fix but a restart does. Nothing else has that shape."

**What separates them:** the mid answer describes deoptimization as an event. The senior
answer explains the **mechanism** (uncommon traps and frame reconstruction), identifies that
**the recompilation, not the transfer, is the cost**, names specific reasons and actions,
raises class-hierarchy analysis as a Java-specific mechanism with a surprising trigger, and
supplies an operational signature the team can use without understanding any of the above.

**Interviewer's follow-up:** *"Is deoptimization bad?"* — No, it's what makes speculation
safe, and speculation is where most of C2's advantage comes from. Without the ability to
deoptimize, C2 could only apply optimisations it could *prove*, which would be far fewer. A
one-off deopt at startup is completely normal and I'd expect thousands of them. **What's bad
is a deopt storm** — a steady rate rather than a one-off, meaning something keeps violating a
speculation — and `make_not_compilable`, meaning the JVM has stopped trying.

---

### Q3 — "A call site has been monomorphic for a million calls. We add a second
implementation. What happens?"

**MID-LEVEL answer:** "The JIT has to handle both types now, so the call becomes a virtual
call instead of an inlined one. It'll be a bit slower for that call site."

**SENIOR answer:** "Three things happen, and the third is much bigger than the first two.

**First, the existing compiled code is invalidated.** If C2 had used class-hierarchy analysis
— only one implementation loaded, so inline with no type check — then merely *loading* the
second class invalidates that dependency and every method carrying it is made not entrant.
If it was inlined with a class check from the profile, the first call with the new type
triggers a `class_check` deopt.

**Second, it's recompiled with the wider profile.** Two types is still fine — C2 does
**bimorphic inlining**, inlining both bodies behind a type switch. Three or more and it gives
up and emits a real virtual call, because the profile can only record a bounded number of
receiver types — `TypeProfileWidth`, typically 2, which is the mechanical reason bimorphic is
the boundary rather than some arbitrary policy.

**Third — and this is the one that actually costs — losing the inline loses everything
downstream of the inline.** The direct cost of a virtual call is a few nanoseconds and an
indirect branch the CPU will usually predict correctly. But once the caller can't see into the
callee, escape analysis can't prove that objects passed in don't escape, so allocations that
were being scalar-replaced come back. Constants can't be folded across the boundary. Branches
can't be eliminated. **The direct cost is small and bounded; the indirect cost is unbounded**,
and it's the reason a 2%-of-traffic feature flag can measurably slow down the 98%.

**And it's not confined to the polluted path.** Compiled code is per-JVM, not per-request.
There's one compiled version of the calling method, so every request through it gets the
degraded version. Endpoints that never touch the new implementation get slower too, if they
share code that was inlined into the same frames.

**How I'd verify rather than assert:** `-XX:+PrintCompilation -XX:+TraceDeoptimization` to
find the `made not entrant` and its reason, `-XX:+PrintInlining` to confirm the inline was
lost and see the message — 'too big' versus 'virtual call' point at different fixes — and JMH
with `@Fork(3)` to quantify it. **`@Fork(3)` is not optional there**: with one fork, running
the bimorphic benchmark pollutes the monomorphic one and all the results converge, so you
measure nothing.

**And the fix isn't necessarily to prevent it.** Bimorphic inlining often costs almost
nothing. If I measure it and it's within budget, we ship the feature. If it isn't, the highest
-leverage fix is usually to make the new implementation **small enough to inline** — because a
bimorphic site where both callees inline is very different from one where neither does."

**What separates them:** the mid answer identifies the virtual call. The senior answer knows
the invalidation mechanism, knows bimorphic inlining exists and why the boundary is at two,
identifies that **the lost inline matters far more than the lost direct call**, understands
the blast radius is the whole JVM, names the verification tools including the `@Fork(3)`
subtlety, and refuses to prescribe a fix without a measurement.

**Interviewer's follow-up:** *"How would you prevent it?"* — Usually I wouldn't; I'd measure
first, because interfaces with two implementations are normal and good design. If measurement
says it matters: keep the callee small enough to inline, split the call site so each sees one
type — branch on the flag once at the top rather than per element — or, for a genuinely hot
type that genuinely has one implementation, make it `final` so the call isn't virtual at all.
What I would *not* do is remove interfaces across the codebase as a performance measure; C2's
class-hierarchy analysis already handles the single-implementation case, and I'd be trading
real design quality for a benefit I haven't measured.

---

### Q4 — "Walk me through the tiers. Why does the JVM have both C1 and C2?"

**MID-LEVEL answer:** "C1 is the client compiler and C2 is the server compiler. C1 compiles
fast with fewer optimisations, C2 compiles slower with better optimisations. Tiered
compilation uses both."

**SENIOR answer:** "**Tiers 0 through 4.** Tier 0 is the interpreter. Tiers 1, 2 and 3 are all
C1, differing in how much profiling they do. Tier 4 is C2.

The reason there are three C1 tiers is the interesting part. **Tier 3 is C1 with full
profiling** — that's the normal warm path, and it exists because C2 *needs* a profile and
somebody has to collect it. The interpreter collects it too, but the interpreter is an order
of magnitude slower, so you want to get out of it quickly while still gathering the data C2
will consume. **Tier 1 is C1 with no profiling**, used for trivial methods where C2 has
nothing to add — profiling them would be pure overhead. **Tier 2 is C1 with limited profiling**,
used when the C2 queue is backed up: get to compiled code fast, profile lightly, upgrade
later.

So the normal path is **0 → 3 → 4**: interpret, then profiled C1 while collecting the type
profile, then C2 consuming it.

**Why both compilers exist** is a genuine engineering trade. C2 takes a long time to compile —
it does global value numbering, aggressive inlining, escape analysis, loop transformations —
and while it's compiling, your method is still running slowly. If C2 were the only compiler,
every method would either be interpreted or wait for an expensive compilation. C1 fills that
gap: fast compilation, decent code, immediate benefit, and it collects the data that makes
C2's output good.

**The promotion trigger is two counters**: an invocation counter incremented on method entry,
and a back-edge counter incremented on every backward jump. The second is why a method called
*once* containing a hot loop still gets compiled — via **on-stack replacement**, where the JVM
compiles a version entered at that loop's bytecode index and swaps the running frame over
mid-execution. That's also why a naive benchmark's numbers depend on the loop count you chose:
you're measuring a blend of interpreted, C1 and C2 execution in a ratio you didn't control.

**And I'd know the failure modes.** `-XX:TieredStopAtLevel=1` caps at C1 — right for a CLI
tool, wrong for a service, and I've seen it cargo-culted from the first case into the second.
`-Xcomp` compiles everything immediately with no profile, which produces *worse* code than the
default because speculation is what makes C2 good. And the code cache is fixed-size native
memory; if it fills, the compiler shuts down and everything runs interpreted permanently.
`jcmd Compiler.codecache` is a ten-second check that almost nobody has on a dashboard."

**What separates them:** the mid answer names the two compilers. The senior answer knows all
five tiers and **why the three C1 tiers differ**, explains the engineering trade that makes
two compilers necessary, names both counters and connects the back-edge counter to OSR and
therefore to benchmark unreliability, and closes with three specific operational failure
modes.

**Interviewer's follow-up:** *"When would you turn tiered compilation off?"* — Essentially
never. `-XX:-TieredCompilation` means C2 only with a much higher threshold, so you get a
longer, slower warm-up for a steady-state gain that is marginal at best. The historical
argument was that tier-3 profiling code is slower than pure C1 code and pollutes the picture,
but that's been well-tuned for many releases. If someone proposes it, I'd want an A/B against
the recorded baseline showing steady-state throughput *and* warm-up time, three runs each —
and I'd expect it to lose on warm-up and roughly tie on steady state, which makes it a
straightforward no.

---

### Q5 — "Our p99 got worse and nothing changed. GC is normal. Where do you look?"

**MID-LEVEL answer:** "I'd profile the application and see what's slow. Maybe check the
database, or if there's more traffic than usual."

**SENIOR answer:** "'Nothing changed' is almost never true, so my first job is to find what
did — but I'd triage in an order that eliminates whole categories fast.

**First, exonerate GC properly rather than glancing at a dashboard.** Compute GC pause
overhead — sum of pause durations over wall clock. If it's under a percent, GC is not the
story and I stop looking there. I'd also check the live-set floor against our recorded
baseline, because a growing live set is the shape that precedes GC problems.

**Second, safepoints.** GC pause and stop-the-world duration are different numbers.
`-Xlog:safepoint*` separates time spent *reaching* the safepoint from time spent *at* it, and
a long time-to-safepoint is invisible in GC pause time. That's Topic 73, and it's the single
most common thing a GC-focused investigation misses.

**Third, and this is where 'nothing changed' usually breaks down: JIT state.** Things that
change compiled code without changing source:
- **A feature flag flipped**, introducing a second implementation at a hot call site.
  Profile pollution: everything gets slower, including endpoints that don't touch the new
  code, because compiled code is per-JVM.
- **A class loaded for the first time** — a lazily-initialised bean, a reflective
  instantiation — invalidating a class-hierarchy-analysis-based inline.
- **The code cache filling**, which shuts the compiler down permanently. That one produces a
  uniform, cliff-edge, order-of-magnitude regression with no memory signal at all, and
  `jcmd Compiler.codecache` rules it in or out in ten seconds.

**The diagnostic that separates JIT from everything else: does a rollback fix it, and does a
restart?** A regression that survives rollback but dies on restart is JIT state. Nothing else
has that signature.

**Fourth, the environment.** Container CPU throttling — check cgroup throttle counters, not
CPU utilisation, because a throttled container looks under-utilised. A noisy neighbour. A
downstream dependency's own tail. Connection pool saturation, which for us is a transaction
holding a connection across an HTTP call.

**And I'd insist on the recorded baseline throughout.** Without Topic 65's numbers, 'p99 got
worse' is an impression. With them, it's a delta I can attribute.

**Concretely, the first five commands:** GC pause overhead from the log; safepoint log tail;
`jcmd Compiler.codecache`; `jcmd VM.flags -all` diffed against what we think we deployed; and
the deploy and feature-flag audit logs correlated against the time the p99 moved."

**What separates them:** the mid answer starts profiling. The senior answer **triages in an
order that eliminates categories cheaply**, knows GC pause and stop-the-world are different
numbers, holds JIT state as a first-class hypothesis with three specific mechanisms, supplies
the rollback-versus-restart discriminator, and grounds everything in a recorded baseline. The
single strongest move is the code-cache check: it is ten seconds, it is almost never done, and
when it hits, it explains everything.

**Interviewer's follow-up:** *"You find the code cache is full. Now what?"* — Immediate:
restart the affected instances, which restores compilation. Then raise
`-XX:ReservedCodeCacheSize` — but it's native memory outside `-Xmx`, so I have to raise the
container memory limit with it or I've traded a JIT problem for an OOM kill. Then find out why
it filled: a large application plus an APM agent rewriting classes, heavy lambda and dynamic
proxy generation, lots of reflection. And I'd add the metric to the dashboard with an alert
well before the cliff, because this is a failure mode you can see coming and almost nobody
watches for.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. A method needs thousands of executions before C2 compiles it, and the counter is
   executions rather than elapsed time. **Derive from that alone** why adding CPU to a pod
   only partially fixes the post-deploy latency spike, and design the one experiment that
   proves elapsed time is not the variable.

2. `TypeProfileWidth` is typically 2. Explain why that single number is the mechanical reason
   "monomorphic and bimorphic are special, three or more is just polymorphic" — and predict
   what would change if it were 4.

3. C2 can inline a call with **no type check at all** when class-hierarchy analysis says one
   implementation is loaded. Explain what protects that inline, what invalidates it, and why
   this means a class being *loaded* — never called — can slow down a hot path. Then explain
   why staging may never reproduce it.

4. Losing an inline costs far more than the virtual call it replaces. **Name three specific
   optimisations that become impossible once the caller cannot see into the callee**, and for
   each say what the observable consequence is. (Topic 75 is the full treatment; do this
   without it.)

5. A benchmark with one JVM fork measures each case with a profile polluted by every other
   case. Construct the specific scenario where `@Fork(1)` makes two genuinely different
   implementations appear identical, and explain why increasing warm-up iterations does **not**
   fix it.

6. Compiled code is per-JVM, not per-request. Derive from that why a feature flag enabled for
   2% of traffic can degrade 100% of it — and then derive the operational signature that
   distinguishes this from "the new code path is just slow".

7. Your p99 regressed. Rolling back the change does not fix it; restarting the pods does.
   **List everything that signature rules IN and everything it rules OUT**, and name the one
   command you would run first.

---

## Quick reference card

### The model

```
tier 0 = interpreter          (profiles; ~10x slower than compiled)
tier 1 = C1, no profiling     (trivial methods; C2 has nothing to add)
tier 2 = C1, limited profiling (used when the C2 queue is backed up)
tier 3 = C1, full profiling   (THE NORMAL WARM PATH)
tier 4 = C2                   (consumes the profile; speculates; steady state)

normal path:  0 -> 3 -> 4

invocation counter  -> method-level compilation
back-edge counter   -> loop-level compilation -> ON-STACK REPLACEMENT (OSR)

call site: 1 receiver  = monomorphic -> inlined (maybe with NO check, via CHA)
           2 receivers = bimorphic   -> both inlined behind a type switch
           3+          = polymorphic -> virtual call, NO INLINING, and everything
                                        downstream of the inline is lost

speculation violated -> uncommon trap -> DEOPTIMIZE to the interpreter
                     -> compiled code "made not entrant"
                     -> recompiled with a WIDER profile = PERMANENTLY SLOWER
```

### Flags

| Flag | Use |
|---|---|
| `-XX:+PrintCompilation` | Every compilation and deopt event. **Drill only** — very chatty |
| `-XX:+UnlockDiagnosticVMOptions -XX:+TraceDeoptimization` | The **reason** for each deopt |
| `-XX:+UnlockDiagnosticVMOptions -XX:+PrintInlining` | Inlining decisions and why they failed. Topic 75 |
| `-XX:+LogCompilation -XX:LogFile=...` | Structured XML for JITWatch |
| `-XX:TieredStopAtLevel=1` | Cap at C1. **CLI tools and tests only. Never a service** |
| `-XX:-TieredCompilation` | C2 only, higher threshold. Almost never |
| `-Xint` | Interpreter only. **A/B control** |
| `-Xcomp` | Compile immediately, no profile. **Worse code.** Demonstration only |
| `-XX:ReservedCodeCacheSize=<n>` | Size the code cache. Native memory — raise the container limit too |
| `-XX:CICompilerCount=<n>` | Compiler threads. Only when the CPU quota misleads the JVM (Topic 82) |
| `-XX:-Inline` | Disable inlining. **A/B control only** |
| `-XX:SharedArchiveFile=<f>` | AppCDS — attacks class loading, not method profiles |

> **Version note, one line:** JDK 25 adds ahead-of-time capabilities aimed at warm-up
> (class loading and linking, and in newer versions method profiles), but **I am not
> asserting which are present, default, or production-ready on your build** — settle it with
> `java -XX:+PrintFlagsFinal -version | grep -i -E "AOT|CDS"` and your vendor's release notes.
> On JDK 21, AppCDS is the available lever and covers class loading only.

### Reading `PrintCompilation`

```
<timestamp_ms>  <compile_id> <attrs> <tier> <Class::method> @<osr_bci> (<n> bytes)
<timestamp_ms>  <compile_id> <attrs> <tier> <Class::method>   made not entrant
```

***Illustration of the format, not captured output. All values are placeholders.***

| Attribute | Meaning |
|---|---|
| `%` | **OSR** — compiled from a loop back-edge, entered mid-method |
| `s` | `synchronized` method |
| `!` | Has exception handlers |
| `b` | Blocking — an application thread waited for the compilation |
| `n` | Native method wrapper |
| `made not entrant` | **The deopt event.** No new calls enter this version |
| `made zombie` | No live frames remain; reclaimable from the code cache |

### Deopt reasons worth recognising

| Reason | Cause |
|---|---|
| `class_check` | Receiver was not the speculated class — **classic profile pollution** |
| `bimorphic` | A two-type site saw a third |
| `unstable_if` | A branch believed never-taken was taken |
| `null_check` | First null on a never-null path |
| `unloaded` / `uninitialized` | Referenced class not yet loaded. Common and mostly benign at startup |
| `predicate` / `loop_limit_check` | A loop-optimisation guard failed |

| Action | Meaning |
|---|---|
| `reinterpret` | Fall back to the interpreter for this execution |
| `make_not_entrant` | Retire the compiled version; recompile later |
| `make_not_compilable` | **Give up.** This method may stay slow forever |

### Diagnostic commands

```bash
# What is my JIT configuration?
java -XX:+PrintFlagsFinal -version | grep -E "TieredCompilation|TieredStopAtLevel|CICompilerCount|TypeProfileWidth|MaxInlineSize|FreqInlineSize|ReservedCodeCacheSize"
jcmd <pid> VM.flags -all | tr ' ' '\n' | grep -E '^-XX' | sort

# Watch compilation, in a drill.
-XX:+UnlockDiagnosticVMOptions -XX:+PrintCompilation -XX:+TraceDeoptimization
-XX:+UnlockDiagnosticVMOptions -XX:+PrintInlining
-XX:+LogCompilation -XX:LogFile=/var/log/orderflow/compile.log

# The code cache. Run this AFTER a soak, not at startup. Ten seconds, rarely done.
jcmd <pid> Compiler.codecache
jcmd <pid> Compiler.codelist | wc -l
jcmd <pid> Compiler.queue

# Triage a compile log.
grep -E 'MyClass::myMethod'                    compile.log
grep -iE 'made not entrant|Uncommon trap|reason=' compile.log | tail -50
grep -c 'made not entrant'                     compile.log
grep '%'                                       compile.log | head   # OSR compilations

# A/B controls.
java -Xint ...                        # no compilation at all
java -XX:TieredStopAtLevel=1 ...      # C1 only, no C2 speculation
java -XX:-Inline ...                  # no inlining

# The right way to measure the cost.
java -jar target/benchmarks.jar MyBenchmark -f 3 -prof perfasm -prof gc
```

### Gotchas checklist

- [ ] Every method starts interpreted. The clock is **executions**, not elapsed time.
- [ ] Thousands of executions before C2. Print your own thresholds.
- [ ] `%` in `PrintCompilation` means OSR — your benchmark may be measuring OSR code.
- [ ] A naive timing loop measures dead-code elimination, OSR and cold JIT in unknown ratio.
- [ ] `@Fork(3)`, always. One fork means one profile means every case pollutes every other.
- [ ] Compiled code is **per-JVM**. A 2% flag can degrade 100% of traffic.
- [ ] Class **loading** can deoptimize code that never touches that class (CHA).
- [ ] Rollback doesn't fix it but restart does ⇒ **JIT state**. Nothing else has that shape.
- [ ] Losing the inline costs far more than the virtual call. Topic 75.
- [ ] `TieredStopAtLevel=1` is right for CLI tools and wrong for services.
- [ ] `-Xcomp` produces **worse** code than the default. Speculation needs a profile.
- [ ] The code cache is native memory outside `-Xmx`, it can fill, and then the compiler stops.
- [ ] Check `Compiler.codecache` **after a soak**. At startup it is always fine.
- [ ] Staging may never warm a call site to C2, so it may never reproduce a pollution bug.
- [ ] Exonerate GC and safepoints first. Both are cheaper to rule out than JIT is to rule in.

---

## When would I use this at work?

**1. Diagnosing a p99 regression that a rollback did not fix.**
That single signature — rollback no, restart yes — points at JIT state, and almost nobody on
the team will know that. You pull `PrintCompilation` on one pod, find the `made not entrant`
timed with a flag flip, name the deopt reason, and turn a multi-day mystery into a
half-day fix with a specific mechanism in the postmortem. This is the highest-value use of
the topic, and it comes up more often than people realise because feature flags introducing
second implementations at hot call sites is an extremely common pattern.

**2. Fixing post-deploy latency spikes without buying hardware.**
The team is about to double instance sizes to fix a spike that is warm-up. You run the
five-minute-idle experiment, show the spike is executions rather than elapsed time, and
propose readiness-gated warm-up plus load-balancer slow start. That is a configuration change
against a hardware purchase, and you can quantify the result as time-to-baseline-p99 on the
deploy dashboard. It also stops the `-XX:TieredStopAtLevel=1` suggestion that would have
traded a startup win for a permanent throughput loss nobody would have measured.

**3. Reviewing any benchmark anyone brings you.**
"We measured it and V2 is 30% faster" is a claim you can now interrogate in three questions:
was it JMH, how many forks, and what does `-prof gc` say. Most hand-rolled benchmarks fail the
first question and most single-fork JMH runs fail the second. Catching this before a
representation change ships on the strength of a wrong number saves a rewrite — and the
rewrite is usually the expensive part, not the measurement.

---

## Connected topics

**Prerequisites:**

- **01 — Boxing and the Integer cache**: `Integer.valueOf` is a method call that must be
  inlined before boxing can be scalar-replaced. Whether it is depends on everything here.
- **21 — Lambdas and `invokedynamic`**: a lambda call site is linked at runtime by
  `LambdaMetafactory` and generates a class. That affects the code cache, method count, and
  whether the call site is monomorphic — a `Function` field assigned two different lambdas is
  a polymorphic call site by construction.
- **22 — Method references**: same mechanism, same consequences.
- **25 — Parallel streams**: parallel pipelines multiply call sites across threads, and the
  common pool's shared code is compiled once for all users of it.
- **40 — Proxying, JDK dynamic proxies and CGLIB**: every proxy is a generated class and a
  level of indirection the JIT must see through. Proxies add methods to the code cache and
  can turn a direct call into a virtual one.
- **42 — Auto-configuration**: conditional beans mean **different classes are loaded in
  different environments**, which means different class-hierarchy-analysis outcomes, which
  means staging and production can genuinely have different compiled code.
- **65 — The load-testing gate**: the recorded baseline is what makes "p99 regressed" a
  measurement instead of an impression, and the warm-up drill requires a reproducible load.
- **66 — JVM architecture**: the execution engine that starts interpreting and promotes hot
  methods. This topic is that sentence, in full.
- **67 — Class loading**: lazy initialisation is what makes class-hierarchy-analysis
  invalidation happen at unpredictable moments.
- **68 / 69 / 70 — Memory areas, object layout, GC fundamentals**: the JIT emits the write
  barriers that maintain the card table and the oop maps that make root scanning possible.
  Exonerating GC is step one of every JIT investigation.
- **73 — Safepoints**: deoptimization, and installing new compiled code, are safepoint
  operations. Compiled code polls for safepoints; where those polls go is a compiler decision.

**This unlocks:**

- **75 — Escape analysis, inlining, scalar replacement, lock elision**: **read it
  immediately after this one.** Every optimisation there depends on inlining succeeding, and
  inlining depends on the profile described here. Topic 75 is the payoff of Topic 74.
- **76 — Bytecode and `javap -c`**: `javap` shows what *javac* emitted, which is nearly
  nothing. This topic shows what actually runs. **Two different questions, two different
  tools** — and confusing them is a common error.
- **77 — JMH**: the full treatment of why every naive benchmark lies. This topic supplies
  three of the four reasons: cold JIT, OSR, and profile pollution across cases.
- **78 — Profiling**: async-profiler shows where time goes in *compiled* code, and a flame
  graph of a cold JVM is a flame graph of the interpreter. JFR records compilation and
  deoptimization events at low enough overhead to leave on in production.
- **79 — Heap dumps**: unrelated to the JIT, except that JIT-state regressions are frequently
  *misdiagnosed* as memory problems because both produce "everything got slower".
- **80 — Off-heap memory**: the code cache is native memory outside `-Xmx`, and a full code
  cache is a native-memory failure with CPU symptoms.
- **81 — Instrumentation agents**: an agent rewrites bytecode at class load, changing method
  sizes and therefore **inlining decisions**. That is a genuine observer effect: attaching a
  profiler can change what you are profiling.
- **82 — JVM tuning and containers**: the CPU quota sizes `CICompilerCount`, so a
  CPU-throttled container compiles more slowly and warms up for longer. It can also change
  the collector and `availableProcessors()`.
- **83 — GraalVM native image**: no JIT at all — closed-world AOT compilation. Fast startup,
  no warm-up, and **no profile-guided peak throughput**. The contrast is the clearest possible
  statement of what C2 is worth.
- **85 — `synchronized` and monitors**: lock elision and lock coarsening are C2 optimisations
  that depend on inlining and escape analysis. Topic 75 bridges them.
- **96 — False sharing**: C2's register allocation and field access patterns interact with
  cache-line behaviour; the same hardware reality, one level down.
- **101 — Virtual threads**: mounting and unmounting interacts with compiled frames, and a
  `synchronized` block pinning a carrier thread is partly a story about what the compiler did
  with the monitor.

---

*Java baseline 21, running on JDK 25. Four things in this document are deliberately hedged
rather than asserted: your JDK's exact tier threshold defaults, the exact spelling of its
`PrintCompilation` and `TraceDeoptimization` output, which ahead-of-time / AOT-cache
capabilities are present and enabled on your JDK 25 build, and your platform's default
`ReservedCodeCacheSize`. Each has a command in the Hands-on section that settles it in under a
minute. **No timing, throughput figure, speedup ratio, compilation log line or deoptimization
trace in this document was captured from a running JVM** — the two output layouts shown are
labelled illustrations of the format with placeholder values, and every number you act on must
come from your own run against your own baseline. What has been stable since tiered
compilation became the default and will still be true at 2am: methods start interpreted,
counters promote them, C2 speculates on what it observed, and a violated speculation costs you
the recompilation rather than the trap.*
