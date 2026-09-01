# 75 — JIT II: Inlining, Escape Analysis, Scalar Replacement, Lock Elision

## Phase: 8 — JVM Internals
## Category: ELITE
## Java baseline: 21  |  Notes features from: 21 / 25
## Project spine: this is why an `orderflow` pricing method that allocates six objects per order line can allocate zero at steady state — and why one added log statement can silently bring all six back.

---

## Before anything else — what is and is not in this document

**I do not have a JVM. I have never run `PrintInlining`, async-profiler, or JMH on your
machine. Nothing in this document is captured output, and I will never present anything
as if it were.**

Specifically, you will not find here:

- a `PrintInlining` transcript presented as something I ran,
- an allocation profile, a flame graph, or a `B/op` figure,
- an allocation rate, a bytes-per-operation number, a GC frequency, or a p99,
- "escape analysis saved 40% of allocations in this example" or any figure of that shape.

**Why the rule bites hardest here:** the entire skill of this topic is *refusing to
believe an optimisation happened until you have evidence*. A fabricated `PrintInlining`
line showing your method inlined would teach you the exact habit this document exists to
destroy.

What you get instead, everywhere:

- the **exact command with exact flags**,
- **WHAT TO LOOK FOR** in the output *you* generate,
- a **"what you see" → "what it means"** table covering the plausible outcomes *and* the
  surprising ones, because the surprising outcome is where the learning is.

### The labelled exception

To read compiler output you must know its shape. In two places I show the **line
structure** of `PrintInlining` and the **column structure** of a JMH `-prof gc` table,
with every value replaced by `<n>`, `<name>` or `xxx`, carrying the inline label:

> *illustration of the format, not captured output*

Placeholders only. If you find a digit in one of those blocks that is not part of a field
name, it is a bug in this document.

### The flag defaults question, answered honestly

This topic is full of numeric thresholds — `MaxInlineSize`, `FreqInlineSize`,
`MaxInlineLevel`, `EliminateAllocationArraySizeLimit`. Commonly-cited values are given
below **as things to confirm, not as facts to quote**. They have changed across releases
and they differ between builds. Every one of them is printable in ten seconds:

```bash
java -XX:+PrintFlagsFinal -version | grep -Ei 'inline|escape|eliminate'
```

**Never quote a threshold you did not print.** An interviewer who knows this area will
ask "on which JDK?", and "I printed it on mine, it was X" is a strictly better answer
than a memorised constant.

### Three flags that may not exist on your JDK, and why

`-XX:+PrintEscapeAnalysis` and `-XX:+PrintEliminateAllocations` are declared in HotSpot as
non-product flags in mainline. On a release JDK they are likely to be rejected outright
with `Unrecognized VM option`. **That is expected, it is not your mistake, and I am
telling you in advance so you do not spend an afternoon on it.** Try them once so you know
which world you are in; then use the instruments that do work on a release build — the
allocation profiler and JMH's `gc.alloc.rate.norm`. Same for `-XX:+PrintOptoAssembly`,
which needs a debug build, and `-XX:+PrintAssembly`, which needs an `hsdis` plugin that is
not bundled.

---

## Mechanical statement

> **Escape analysis proves an object cannot be observed outside its allocating method.
> C2 then scalar-replaces it: the object is never allocated, and its fields become
> registers or stack slots. All of it depends on inlining succeeding first.**

Unpack that into four mechanical claims, each of which you will confirm yourself:

1. **Inlining is the enabler, not a nice-to-have.** C2 optimises one compilation unit. A
   call it cannot see through is an opaque wall: the object you passed across it might do
   anything, so it must exist. Inlining pulls the callee's body into the caller's graph,
   and only then is there a single graph in which "this object is never observed" is
   provable.

2. **Escape analysis is a proof, not a heuristic.** C2 assigns every allocation an escape
   state — `NoEscape`, `ArgEscape`, `GlobalEscape`. The proof must hold on *every* path
   through the compiled graph. One path that stores the reference into a field poisons the
   allocation on all paths.

3. **Scalar replacement is what pays.** Proving non-escape by itself buys nothing. The
   payoff is C2 deleting the allocation and replacing the object with its individual
   fields held in registers. No header, no TLAB bump, no GC pressure, no reference at all.

4. **Lock elision is the same proof applied to monitors.** If no other thread can ever see
   the object, no other thread can ever contend for its monitor, so the CAS on its mark
   word (Topic 85) is provably uncontended and can simply be removed.

And one mechanical claim about *you*:

5. **None of this is true until you measure it.** All four are speculative compiler
   behaviours in tier 4 only (Topic 74). They do not happen in the interpreter, they do
   not happen in C1, they can be lost on recompilation, and they are defeated by changes
   that look completely innocent in a diff.

---

## The bridge from what you know

### PARTIAL — the concept exists in V8; the *observability* does not

This is the rare Phase 8 topic where TurboFan does something recognisably similar. V8's
optimising compiler does perform escape analysis and can avoid materialising short-lived
objects. So the *idea* is not new to you.

Everything around the idea is new:

| In Node, you | In Java, you |
|---|---|
| have no flag to switch escape analysis off | have `-XX:-DoEscapeAnalysis` — a true A/B control |
| have no per-allocation-site profiler | have async-profiler `-e alloc` and JMH `gc.alloc.rate.norm` |
| cannot see inlining decisions without a debug build of V8 | have `-XX:+PrintInlining` on a stock JDK |
| have one heap, one thread, one opaque GC | have a TLAB, generations, a live set, and a collector you chose |
| never think about it because you cannot act on it | are expected to *act on it*, in review, with evidence |

**The honest verdict: the mechanism transfers, the practice does not.** You have never had
to defend a claim about escape analysis with data, because you have never had data. From
this topic on you do.

### Scalar replacement: no observable JS analogue

Scalar replacement is an *observable state* in Java — you can watch bytes-per-operation
drop to zero and then watch it come back. In Node there is no instrument that shows you
the difference between "V8 materialised this object" and "V8 did not". You have no
before/after. **A thing you cannot measure is not a thing you can engineer**, which is why
this has never been part of your job and is about to become part of it.

### Lock elision: NO ANALOGUE, and the reason is structural

Lock elision removes a `synchronized` region on an object no other thread can see.
JavaScript, in the runtime you use, has no locks, because it has no shared mutable heap
across threads. There is nothing to elide. **Do not go looking for the analogy.** The
whole optimisation exists to clean up after a library — `StringBuffer`, `Vector`,
`Collections.synchronizedList` — that was written to be thread-safe and is being used in a
context where thread safety is irrelevant. Your ecosystem never produced those libraries
because it never had the problem.

### What actually transfers

- **"Don't optimise what you haven't measured"** transfers unchanged, and is more load
  bearing here than anywhere.
- **The idea that a compiler's decisions are shaped by code size** transfers — V8's
  inlining budget is real too, and you may have hit it without knowing.
- **The habit of writing small functions** transfers, and here it has a mechanical payoff
  rather than an aesthetic one.

---

## What is this?

### Inlining, precisely

Inlining replaces a call with the callee's body. Two things happen, and only one of them
is the one people talk about:

1. **You save the call.** Real but small — a few nanoseconds of frame setup and an
   indirect branch.
2. **You merge two graphs into one.** This is the whole game. Constant folding, dead-code
   elimination, null-check elimination, loop optimisations, and — the subject of this
   document — escape analysis can now reason across the former call boundary.

C2 decides per call site, using the profile from tier 3 (Topic 74). The decision inputs
that matter:

| Input | Effect |
|---|---|
| Is the receiver type known? | A monomorphic site can be inlined. Polymorphic (3+ types) generally cannot, and you lose everything downstream. |
| How big is the callee, in **bytecodes**? | Two budgets — one for cold callees, a much larger one for hot ones. |
| How deep are we already? | There is a maximum inlining depth. |
| How big has the caller become? | There is a total budget; a caller that has already swallowed a lot stops swallowing. |
| Is the callee already compiled and large? | A large existing compiled body discourages inlining. |

**The unit is bytecodes, not lines and not source characters.** A five-line method with
three string concatenations and a boxed compare can be surprisingly large. Topic 76's
`javap -c` is how you find out; it is one command and it settles the argument.

### Escape analysis, precisely

For each allocation in the compiled graph, C2 asks: *can a reference to this object be
reached from anywhere outside this compilation?* The answer is one of three states:

| State | Meaning | Scalar replacement? | Lock elision? |
|---|---|---|---|
| `NoEscape` | The object never leaves the method. No field store, no return, no exception carrying it, no call receiving it that C2 could not see into. | **Yes** | Yes |
| `ArgEscape` | It is passed to a method, but analysis shows it does not escape the *thread* — it does not get stored anywhere globally reachable. | No | In principle yes — verify on your build rather than relying on this |
| `GlobalEscape` | It is stored in a static or instance field, returned, thrown, put in a collection, or handed to a call C2 could not see through. | No | No |

**`GlobalEscape` is contagious in the direction that hurts.** If an object is
`GlobalEscape`, everything reachable from it is too.

### Scalar replacement, precisely

Given a `NoEscape` allocation, C2 deletes it. The object's fields become independent
values — "scalars" — living in registers or stack slots, exactly as if you had written
local variables instead of an object.

The consequences, in order of how much they matter for `orderflow`:

1. **No allocation.** Not a cheap allocation — *no* allocation. The TLAB pointer (Topic
   68) does not move.
2. **No GC pressure from that site.** Nothing to trace, nothing to copy, nothing to
   promote. Young-collection frequency is driven by allocation rate; this reduces it at
   source.
3. **No object header.** 12 or 16 bytes you never paid (Topic 69). `[JAVA 25]` compact
   object headers change the header size and therefore the `B/op` you will measure — they
   do **not** change whether scalar replacement happens.
4. **Better downstream optimisation.** Fields in registers can be constant-folded and
   propagated in ways fields in memory cannot.

**Important limits, all of which you can hit accidentally:**

- It is **all-or-nothing per allocation site**. There is no partial credit.
- Historically, an object created in two branches and merged before use defeated it.
  Recent JDKs improved handling of these "allocation merges" in C2. **I am not going to
  state which release fixed which case** — check the release notes for your JDK and, more
  usefully, measure. The engineering conclusion is unchanged either way: merges are
  fragile, so if it matters, verify.
- Arrays can be scalar-replaced only when the length is a **compile-time constant** and
  below `EliminateAllocationArraySizeLimit`. Print the value; do not quote mine.
- It happens in **C2 only**. Interpreted and C1-compiled executions allocate normally.
  Which means: **during warm-up, your "zero allocation" method allocates.** Topic 74 is
  the reason; this is one of its most concrete consequences.

### Lock elision, precisely

If an object is proven not to escape the thread, `synchronized` on it cannot possibly be
contended, so C2 removes the lock. Related, and separate: **lock coarsening** merges
adjacent `synchronized` regions on the same object into one, cutting the number of atomic
operations.

The canonical shape:

```java
String describeOrderStatus(OrderStatus status, long orderId) {
    StringBuffer sb = new StringBuffer();      // StringBuffer: every method synchronized
    sb.append("order ").append(orderId).append(" is ").append(status);
    return sb.toString();                      // sb itself never escapes
}
```

`sb` is `NoEscape`. Four `synchronized` method calls, all provably uncontended, all
eligible for removal. **This does not make `StringBuffer` a good choice** — use
`StringBuilder` — but it explains why legacy code full of `StringBuffer` is not as slow as
theory suggests, and it is the cleanest demonstration of the mechanism.

---

## Why does it matter?

**For `orderflow` specifically.** At the Topic 65 baseline the catalogue-read path runs at
70% of the load mix, and the pricing calculation runs per line on every order read and
every placement. If each price calculation allocates a handful of small carrier objects
and those allocations survive to the heap, you are generating garbage proportional to
traffic. Topic 68 taught you that allocation rate drives young-collection *frequency*.
Topic 71 taught you what that does to p99. Scalar replacement, when it works, removes the
allocation at source — better than tuning the collector to cope with it.

**For your code review habits.** Once you know inlining has a size budget, "this method is
getting long" becomes a mechanical argument rather than a taste argument. A method that
grows past the hot-inlining budget stops being inlined, and its caller silently loses
escape analysis. **The diff that causes this looks like a diff that adds logging.**

**For your credibility.** "The JIT will optimise that away" is the single most common
unfalsifiable claim in Java performance discussions. Being the person who says *"probably,
and here is the allocation profile with and without `-XX:-DoEscapeAnalysis`"* is a
visible, repeatable difference in a design review. This document exists to make you that
person.

**For interviews.** "Prove that allocation was eliminated" is a standard senior screen. The
mid-level answer is "escape analysis removes it". The senior answer is a procedure.

---

## Machine-level reality

### The inlining thresholds — names, meanings, and how to print them

| Flag | What it governs | Commonly cited default — **confirm, do not quote** |
|---|---|---|
| `MaxInlineSize` | Max bytecode size of a **cold** (not-hot) callee that may still be inlined | 35 |
| `FreqInlineSize` | Max bytecode size of a **hot** callee | 325 |
| `MaxTrivialSize` | Below this, treated as trivial and inlined almost unconditionally | 6 |
| `MaxInlineLevel` | Maximum inlining **depth** | 15 in recent JDKs; it was smaller historically |
| `InlineSmallCode` | Discourages inlining a callee whose existing compiled code exceeds this many bytes | build-dependent |
| `DontCompileHugeMethods` / `HugeMethodLimit` | A method above the huge limit is **never compiled at all** — it runs interpreted forever | on / 8000 bytecodes |

```bash
# Print YOUR values. Ten seconds. Do this before reading further.
java -XX:+PrintFlagsFinal -version | grep -E 'MaxInlineSize|FreqInlineSize|MaxTrivialSize|MaxInlineLevel|InlineSmallCode|HugeMethodLimit|DontCompileHugeMethods'
```

**The number that ruins afternoons is `FreqInlineSize`.** A hot method at 320 bytecodes
inlines. The same method at 330 does not. Nothing in the source says "330". Nothing in
your IDE warns you. The only signal is a performance regression and, if you look,
`PrintInlining` saying `too big`.

**Measure a method's real bytecode size:**

```bash
javap -c -p com.orderflow.pricing.PricingEngine | grep -A400 'applyPromotions'
# The largest bytecode offset shown before the method ends is its size.
# Compare that number against the FreqInlineSize you just printed.
```

### The escape flags

| Flag | Default | Use |
|---|---|---|
| `-XX:+DoEscapeAnalysis` | on | **The A/B control.** `-XX:-DoEscapeAnalysis` disables the analysis and therefore everything built on it. |
| `-XX:+EliminateAllocations` | on | Scalar replacement specifically. Turning this off alone leaves the analysis running for lock elision — a *finer* control than the one above. |
| `-XX:+EliminateLocks` | on | Lock elision. |
| `-XX:+EliminateNestedLocks` | on | Coarsening of nested locks on the same object. |
| `-XX:EliminateAllocationArraySizeLimit` | 64 | Max constant array length eligible for scalar replacement. Print it. |
| `-XX:+PrintEscapeAnalysis` | — | **Likely rejected on a release JDK** (non-product flag). Try it once. |
| `-XX:+PrintEliminateAllocations` | — | Same. |

**The A/B discipline for the whole topic:**

```bash
# A — defaults. Escape analysis on.
java -XX:+UnlockDiagnosticVMOptions -XX:+PrintInlining <app> 2>&1 | tee inlining-A.txt

# B — the control. Same binary, same load, one flag different.
java -XX:-DoEscapeAnalysis <app>

# C — the finer control: analysis still runs, but no scalar replacement.
java -XX:-EliminateAllocations <app>
```

If A and B produce the same allocation profile, **escape analysis was not doing anything
for you** and every claim to the contrary is wrong. That is the single most valuable
experiment in this document, and it is one flag.

### How lock elision interacts with the mark word (Topic 85)

Topic 85 established: `synchronized` on an uncontended object is a CAS on the object's
mark word, plus a reverse CAS on exit. Cheap, but not free — a CAS is an atomic
read-modify-write on a cache line, and on aarch64 it carries ordering costs.

Lock elision removes those CASes entirely for a `NoEscape` object. The argument is a
proof, not an optimisation gamble:

1. The object is allocated in this compilation.
2. No reference to it reaches any field, any return, any thrown exception, or any call C2
   cannot see into.
3. Therefore no other thread can obtain a reference to it.
4. Therefore no other thread can ever execute `monitorenter` on it.
5. Therefore the mutual-exclusion property is vacuously satisfied with no instructions.

**The subtlety worth carrying into Topic 86/87 territory:** an elided lock also emits no
barriers. That is *safe* precisely because there is no second thread to be visible to. It
is also why you can never reason "my `synchronized` block gave me a happens-before edge"
about a lock on a thread-confined object — there is nothing on the other side of the edge.
The edge is real only when the object is shared, and if it is shared, the lock is not
elided. The two cases never overlap.

### Where escape analysis runs in the pipeline

Roughly, and enough to reason with:

```
bytecode
  -> C2 parses to an IR graph
  -> INLINING happens during parsing: callee bodies are pulled into the graph
  -> ... optimisation passes ...
  -> ESCAPE ANALYSIS runs on the resulting graph
  -> allocations proven NoEscape are SCALAR REPLACED
  -> locks on non-escaping objects are ELIMINATED
  -> ... more passes, now with fields in registers ...
  -> machine code
```

Two consequences you must be able to state:

1. **Inlining happens first, so inlining failure is upstream of everything.** No amount of
   escape-analysis flag-twiddling recovers an optimisation lost to a call C2 could not see
   through.
2. **The analysis sees the graph, not your source.** A method that "obviously" does not
   leak the object can still be `GlobalEscape` because an inlined callee — three levels
   down, in a library you did not write — stored it somewhere.

### What makes an object escape, in practice

The list you should be able to recite in a review:

- Assignment to a **static** field. (The drill.)
- Assignment to an **instance** field of an object that itself escapes.
- **Returning** it.
- **Throwing** it, or storing it in an exception you throw.
- Adding it to a **collection** that escapes.
- Passing it to a method C2 **could not inline** (too big, polymorphic, native).
- Passing it to `Blackhole.consume` in a JMH benchmark. **Yes, really. Trap 3.**
- Being the receiver of a **synchronized** block that C2 could not prove non-escaping —
  circular, but the practical form is: if it escapes, the lock stays.
- Registering it as a listener, a callback, or a `Thread`. (This overlaps precisely with
  Topic 88's "publishing `this` from a constructor".)

---

## Example 1 — minimal

A price-per-line calculation. Small, real domain, no `foo`.

```java
package com.orderflow.lab.ea;

/**
 * A small value carrier. Two longs, nothing else.
 * If C2 can prove an instance never escapes, this allocation can vanish entirely.
 */
record LineTotal(long netCents, long taxCents) {
    long grossCents() { return netCents + taxCents; }
}

public final class ScalarReplacementProbe {

    // The escape hatch used in phase two of the drill. Static = GlobalEscape.
    public static LineTotal lastComputed;

    /** Phase 1: nothing escapes. The LineTotal is a candidate for scalar replacement. */
    static long grossForLine(long unitPriceCents, int quantity, int taxBasisPoints) {
        long net = unitPriceCents * quantity;
        long tax = net * taxBasisPoints / 10_000L;
        LineTotal total = new LineTotal(net, tax);   // <- the allocation under test
        return total.grossCents();                   // <- and it dies right here
    }

    /** Phase 2: identical arithmetic, one added line. The object now escapes globally. */
    static long grossForLineLeaking(long unitPriceCents, int quantity, int taxBasisPoints) {
        long net = unitPriceCents * quantity;
        long tax = net * taxBasisPoints / 10_000L;
        LineTotal total = new LineTotal(net, tax);
        lastComputed = total;                        // <- GlobalEscape. One line.
        return total.grossCents();
    }

    public static void main(String[] args) {
        boolean leak = args.length > 0 && args[0].equals("leak");
        long iterations = args.length > 1 ? Long.parseLong(args[1]) : 2_000_000_000L;

        long sink = 0;
        for (long i = 0; i < iterations; i++) {
            int qty = (int) (i & 7) + 1;
            sink += leak
                ? grossForLineLeaking(1999L, qty, 2000)
                : grossForLine(1999L, qty, 2000);
        }
        System.out.println("checksum " + sink);      // keeps the loop alive
    }
}
```

Compile and get the bytecode size of the method under test — you will need it later:

```bash
mkdir -p ~/java-lab/75 && cd ~/java-lab/75
# put the file at com/orderflow/lab/ea/ScalarReplacementProbe.java
javac -d out com/orderflow/lab/ea/ScalarReplacementProbe.java

javap -c -p -cp out com.orderflow.lab.ea.ScalarReplacementProbe | sed -n '/grossForLine(/,/^$/p'
```

**WHAT TO LOOK FOR in the bytecode:** a `new` opcode, a `dup`, an `invokespecial` to the
record's constructor. **The `new` is there in both phases and always will be** — javac does
not do this optimisation, and Topic 76 already told you javac barely optimises at all.
Whether the `new` survives to machine code is a C2 question, and bytecode cannot answer it.
**This is the first thing to internalise: `javap` proves the allocation is requested, not
that it happens.**

### The four runs

```bash
# A - defaults: escape analysis and scalar replacement on, no leak.
java -cp out com.orderflow.lab.ea.ScalarReplacementProbe

# B - the A/B CONTROL: same code, escape analysis off.
java -XX:-DoEscapeAnalysis -cp out com.orderflow.lab.ea.ScalarReplacementProbe

# C - the finer control: analysis runs, scalar replacement off.
java -XX:-EliminateAllocations -cp out com.orderflow.lab.ea.ScalarReplacementProbe

# D - defaults, but the reference is stored in a static field.
java -cp out com.orderflow.lab.ea.ScalarReplacementProbe leak
```

Under an allocation profiler (next section gives the command), the shape you are testing
for is:

| Run | What you are asking |
|---|---|
| A | Does `LineTotal` appear as an allocation site at all? |
| B | Does it appear *now*, when it did not in A? If yes, A's absence was escape analysis. |
| C | Same question, isolating scalar replacement from the analysis itself. |
| D | Does it reappear with defaults, purely because of one field store? |

**If A and B look the same, you have learned something real:** either your loop was not
compiled by C2 (warm-up), or the profiler is not sampling this site, or the object was
escaping for a reason you have not found. All three are worth chasing. **Do not "fix" it
by concluding the optimisation happened.**

---

## Example 2 — production scenario (on the project spine)

### The constraints, taken from the Topic 65 baseline

| Thing | Value |
|---|---|
| Service | `orderflow`, containerised |
| Dataset | 100k products, 1M orders, 5M order lines |
| Container | 2 vCPU, 2 GiB memory limit |
| Heap | `-Xmx1200m -Xms1200m`, G1 (confirm with `jcmd VM.flags -all`) |
| Load mix | 70% catalogue read, 20% order read, 10% order placement |
| Baseline | `/docs/java/baselines/` — p50/p95/p99/p999 per endpoint |

### The code that has been running for months

```java
package com.orderflow.pricing;

/** Immutable carrier. Allocated per line, per request, and never stored anywhere. */
record PriceBreakdown(long listCents, long discountCents, long taxCents) {
    long payableCents() { return listCents - discountCents + taxCents; }
}

@Component
public class PricingEngine {

    private final PromotionTable promotions;   // immutable, loaded at startup

    public PricingEngine(PromotionTable promotions) { this.promotions = promotions; }

    /** Called once per order line. At the baseline mix this is one of the hottest
     *  methods in the service. */
    public long payableCents(long productId, long unitPriceCents, int quantity, int taxBp) {
        long list = unitPriceCents * quantity;
        long discount = discountFor(productId, list);
        long tax = (list - discount) * taxBp / 10_000L;
        return new PriceBreakdown(list, discount, tax).payableCents();
    }

    /** Small, hot, monomorphic. Bytecode size well under FreqInlineSize. */
    private long discountFor(long productId, long listCents) {
        int bp = promotions.basisPointsFor(productId);
        return bp == 0 ? 0L : listCents * bp / 10_000L;
    }
}
```

At the baseline this is fine. `PriceBreakdown` is `NoEscape`: constructed, one method
called on it, discarded. `discountFor` is small and inlines. The whole thing collapses into
arithmetic on registers.

### The change

A ticket asks for promotion observability: *"we cannot tell which promotion applied to a
line."* Someone adds logging and defensive checks inside `discountFor`.

```java
    private long discountFor(long productId, long listCents) {
        int bp = promotions.basisPointsFor(productId);
        if (bp < 0 || bp > 10_000) {
            log.warn("promotion basis points out of range: product={} bp={} list={}",
                     productId, bp, listCents);
            bp = 0;
        }
        Promotion applied = promotions.lookup(productId);
        if (applied != null && log.isDebugEnabled()) {
            log.debug("promotion {} applied to product {} on list {} giving {} bp",
                      applied.code(), productId, listCents, bp);
        }
        if (bp == 0) {
            return 0L;
        }
        long discount = listCents * bp / 10_000L;
        if (discount > listCents) {
            log.warn("discount exceeds list: product={} discount={} list={}",
                     productId, discount, listCents);
            discount = listCents;
        }
        return discount;
    }
```

**Every line of this is defensible.** Guarded logging, range checks, a clamp. It would pass
review in any team. Nobody claimed a performance benefit and nobody claimed a performance
cost.

### What you observe, in the order you observe it

1. Functional tests pass. The change ships.
2. `orderflow`'s p99 on `GET /orders` and `GET /products` drifts above the Topic 65
   baseline. Not a cliff — a drift, maybe 10–25%. **Nobody is paged.**
3. Young-collection frequency in `-Xlog:gc*` is up. Pause *durations* are unchanged.
4. Allocation rate is up. If you have the Topic 78 dashboard, this is the first honest
   signal.
5. A heap dump (Topic 79) shows **nothing wrong** — no leak, live set unchanged. The
   objects are dying young exactly as designed. There are just far more of them.
6. Everyone concludes "more traffic" or "the dataset grew". Nobody connects it to a
   logging change three weeks earlier.

### Why the p99 moved

The added code pushed `discountFor` past the hot-inlining bytecode budget. Note what got
big: **the string concatenation machinery in the `log.warn` calls, and the boxing of
`productId`, `bp`, `listCents` and `discount` into `Object[]` for the varargs**. Topic 01
told you boxing is an allocation; Topic 76 told you what SLF4J's parameterised logging
actually compiles to. Both bills come due here.

Then the chain:

```
discountFor grows past FreqInlineSize
  -> C2 will not inline it into payableCents
  -> payableCents's graph now contains an opaque call
  -> C2 cannot prove what happens to values crossing that call
  -> escape analysis on PriceBreakdown becomes weaker or fails outright
  -> PriceBreakdown is allocated for real, per line, per request
  -> plus the boxed varargs arrays allocated on paths that never log
  -> allocation rate up -> young collections more frequent -> p99 drift
```

**The lesson is the shape, not the numbers.** A method that is not the hot one got bigger and
its *caller* lost an optimisation. No profiler line says "you lost escape analysis".

### The diagnosis, as commands

```bash
# 1. Rule out GC as a cause rather than a symptom. Twenty seconds.
#    Frequency up, durations flat => allocation rate, not the collector.
grep -E 'Pause Young' /var/log/orderflow/gc.log | tail -50

# 2. Rule out safepoints (Topic 73). Also twenty seconds.
grep -E 'safepoint' /var/log/orderflow/safepoint.log | tail -20

# 3. Where are the allocations coming from? THIS is the instrument for this topic.
./asprof -e alloc -d 60 -f /tmp/alloc-after.html $(pgrep -f orderflow)

# 4. Compare against a profile from the previous release. If you do not have one,
#    take one now and start keeping them - this is the argument for doing so.
./asprof -e alloc -d 60 -f /tmp/alloc-before.html <pid of previous-release pod>

# 5. Confirm the inlining hypothesis directly.
#    PrintInlining is very high volume under load: log to a file, one pod only.
java -XX:+UnlockDiagnosticVMOptions -XX:+PrintInlining \
     -XX:+LogCompilation -XX:LogFile=/var/log/orderflow/compile.log \
     -jar orderflow.jar
grep -n 'discountFor' /var/log/orderflow/compile.log | head -40

# 6. Get the method's actual bytecode size and compare it against the printed threshold.
javap -c -p -cp orderflow.jar com.orderflow.pricing.PricingEngine \
  | sed -n '/discountFor/,/^$/p' | tail -5
java -XX:+PrintFlagsFinal -version | grep FreqInlineSize
```

| What you see | What it means |
|---|---|
| `discountFor` with `too big` or `hot method too big` | **Confirmed.** The method exceeded the budget. Fix by shrinking it. |
| `discountFor` inlined, but `PriceBreakdown` still in the alloc profile | Inlining is not the story. Something else makes it escape — check what `PromotionTable.lookup` does with it, and check for a merge. |
| No mention of `discountFor` at all in the compile log | It was never compiled hot enough during your window, or you filtered wrong. Run longer; check the class name spelling. |
| Allocation profile dominated by `Object[]` and `String` rather than `PriceBreakdown` | The varargs boxing and concatenation are the bigger half. **Fix that first** — it is cheaper and more certain than chasing scalar replacement. |
| GC pause *durations* also up, not just frequency | Something else changed as well (live set, region sizing). Do not attribute everything to this. |

### The fixes, in the order you should consider them

1. **Extract the cold paths.** Keep `discountFor` tiny; move each logging/clamping branch
   into a separate `private` method that is called only on the rare path. C2 does not
   inline the cold callee, and does not need to — the hot path shrinks back under budget.
   **This is the fix, and it is a five-minute refactor.**
2. **Guard the varargs.** `if (log.isWarnEnabled())` around the warn calls removes the
   boxing on the normal path. Correct even independently of this topic.
3. **Re-measure.** Alloc profile, then the Topic 65 load profile against the recorded
   baseline. If p99 does not return, your hypothesis was wrong and you should say so.
4. **Only then** consider structural changes — returning a `long` instead of a record, and
   so on. **Do not start here**; you will make the code worse for an undemonstrated benefit.
5. **Put allocation rate on the dashboard.** This is the durable fix; the rest is the incident.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — asserting "the JIT removed that allocation", with no evidence

**Wrong approach.** In review: *"That record is fine, escape analysis will scalar-replace
it."* Nobody objects, because it is plausible and nobody can cheaply disprove it.

**Exact symptom.** Nothing, for months. Then allocation rate is 3× what the capacity model
(Topic 129) assumed, young collections are frequent, p99 misses the baseline, and every
individual code review that got you here was approved.

**Root cause.** Escape analysis is a **proof over the compiled graph**, and the compiled
graph is not the source. Whether the proof succeeds depends on inlining, on what a library
method three frames down does with the reference, on whether the site is monomorphic, and
on whether C2 compiled it at all. **None of that is visible in the diff.** The claim is not
false — it is *unfalsifiable as stated*, which is worse.

**Fix.** Make the claim falsifiable, in one of two ways, and never accept it otherwise:

```bash
# The cheap one, in a benchmark (Topic 77). Read gc.alloc.rate.norm - bytes per op.
mvn clean verify
java -jar target/benchmarks.jar PricingBench -prof gc -f 3 -wi 5 -i 5

# The one that works on the real service.
./asprof -e alloc -d 60 -f /tmp/alloc.html $(pgrep -f orderflow)
```

**The review comment that works:** *"Plausible — can you attach the `-prof gc` numbers with
and without `-XX:-DoEscapeAnalysis`? If they are identical, the optimisation is not
happening and we should know that."* It is polite, specific, and cheap to satisfy.

---

### Trap 2 — a method just over the threshold silently kills escape analysis in its caller

**Wrong approach.** Growing a hot small method organically — a null check here, a log line
there, a metric increment. Each change is a few bytecodes. No single change is wrong.

**Exact symptom.** A performance regression with **no corresponding change to the hot
method**. The hot method is untouched in the diff; the regression is real. Allocation rate
in the caller goes up. Profilers show the caller allocating objects it "obviously" does not
need. This is Example 2's incident.

**Root cause.** `FreqInlineSize` is a hard cliff, measured in bytecodes. The behaviour on
either side is completely different, and the source gives no hint where the line is. Then:
no inline → opaque call → escape analysis fails in the *caller*.

**Fix.**

```bash
# 1. Measure the callee's real bytecode size.
javap -c -p -cp target/classes com.orderflow.pricing.PricingEngine | sed -n '/discountFor/,/^$/p' | tail -5

# 2. Print the actual threshold on this JDK.
java -XX:+PrintFlagsFinal -version | grep -E 'FreqInlineSize|MaxInlineSize'

# 3. Confirm the decision the compiler actually made.
java -XX:+UnlockDiagnosticVMOptions -XX:+PrintInlining -jar app.jar 2>&1 | grep discountFor
```

Then **extract the cold paths into separate methods**. Structure the hot method as: compute,
and on the rare branch call out to a big cold method. That is the shape C2 wants and it is
also, independently, the shape a human wants.

**Do not** raise `FreqInlineSize` globally: you would change the budget for every method in
the JVM to fix one, trading code-cache footprint (Topic 74's Trap 4) for it. For a single
method there is `-XX:CompileCommand=inline,com/orderflow/pricing/PricingEngine.discountFor`
— last resort, commented in the manifest, with an expiry date.

---

### Trap 3 — `Blackhole` defeating the optimisation you were measuring

**Wrong approach.** A JMH benchmark written the way every tutorial writes one:

```java
@Benchmark
public void measureBreakdown(Blackhole bh) {
    PriceBreakdown b = new PriceBreakdown(1999L, 200L, 360L);
    bh.consume(b);              // <- the object is now handed to a method C2 must respect
}
```

**Exact symptom.** `-prof gc` reports a non-zero `gc.alloc.rate.norm` — roughly one object
per operation. You conclude scalar replacement does not work on records, or does not work
on your JDK, and you go and rewrite production code to avoid an allocation that production
was never making.

**Root cause.** `Blackhole.consume` exists specifically to stop C2 eliminating the value.
Handing the *reference* to it makes the object escape. **You have measured the cost of an
object you forced into existence.** The benchmark is not measuring your production code; it
is measuring a scenario your production code does not contain.

**Fix — consume a scalar derived from the object, not the object:**

```java
@Benchmark
public long measureBreakdownPayable() {
    // Return a primitive. The object stays NoEscape; the long is what escapes.
    return new PriceBreakdown(1999L, 200L, 360L).payableCents();
}
```

Then build the A/B, because a single number proves nothing:

```bash
# The measurement.
java -jar target/benchmarks.jar PricingBench -prof gc -f 3 -wi 5 -i 5

# The CONTROL. Same benchmark, escape analysis off.
java -jar target/benchmarks.jar PricingBench -prof gc -f 3 -wi 5 -i 5 \
     -jvmArgs "-XX:-DoEscapeAnalysis"
```

| What you see | What it means |
|---|---|
| `gc.alloc.rate.norm` near zero in the main run, non-zero with `-XX:-DoEscapeAnalysis` | **This is the proof.** The delta is the optimisation. |
| Non-zero and roughly equal in both runs | Not scalar-replaced. Find the escape — start with what your benchmark does with the value. |
| Near zero in **both** runs | Suspicious. Something is eliminating the whole computation. Check that the benchmark returns a value derived from the inputs and that `@State` inputs are not compile-time constants. |
| A number that is not a multiple of the object's real size | The profiler is sampling and/or other allocations are mixed in. Isolate the benchmark further. |

**The general principle, worth more than the trap:** *any* mechanism you add to a benchmark
to stop the compiler optimising something is itself a change to what the compiler sees.
Topic 77 is the full treatment. Sharpest form: **a `Blackhole` is an escape route.**

---

### Trap 4 — the reference escaping to a field, in code that looks harmless

**Wrong approach.** Adding "just a little" state. A last-value cache. A metrics sample. A
debug hook. A `ThreadLocal` "current calculation" for logging context.

```java
    private PriceBreakdown lastBreakdown;   // "for the /debug endpoint"

    public long payableCents(...) {
        PriceBreakdown b = new PriceBreakdown(list, discount, tax);
        this.lastBreakdown = b;             // GlobalEscape. One line, whole optimisation gone.
        return b.payableCents();
    }
```

**Exact symptom.** Allocation rate rises in exact proportion to request rate. The alloc
profile names `PriceBreakdown` at a site that previously did not appear. Nothing else
changed. A heap dump shows almost none of these objects retained — **because they die
immediately anyway**, which is precisely why the whole thing is so easy to miss: the field
is overwritten on the very next call, so there is no leak, only garbage.

**Root cause.** `GlobalEscape`. The proof requires that *no* path stores the reference
anywhere reachable from outside. A single field store on any path kills it for all paths.
It does not matter that the field is private, that it is immediately overwritten, or that
nobody reads it.

**Fix.**

- **Store the scalars, not the object**, if you must store anything:
  `this.lastPayableCents = payable;` — a `long` field does not keep an object alive and does
  not make one escape.
- Better: **emit a metric or a log event** rather than holding state. Topic 118 is where
  that belongs.
- Better still: **delete it.** "For the debug endpoint" is a real requirement served well by
  structured logging and badly by a mutable field on a hot singleton — which is also,
  incidentally, a data race (Topics 86–88).
- Verify with the alloc profile before and after — this is the drill.

---

### Trap 5 — expecting the optimisation during warm-up, in staging, or under a CPU quota

**Wrong approach.** Running the experiment for a few seconds, in an environment that is not
the production one, and concluding either "it works" or "it does not".

**Exact symptom.** Three flavours, all common:

- The benchmark shows allocation that "should not be there" — because the loop ran mostly
  interpreted or C1-compiled and never reached C2.
- Staging shows zero allocation and production shows plenty — different load, different
  profile, a polymorphic call site in production that is monomorphic in staging (Topic 74).
- A container at `--cpus=0.5` behaves differently from the developer laptop, because
  compiler threads are starved and tier-4 compilation is delayed (Topic 82).

**Root cause.** **Escape analysis is a C2 optimisation.** No C2, no scalar replacement.
Reaching C2 requires enough executions, enough time, and enough CPU to run the compiler
threads. All three vary by environment. And C2's decisions are driven by a *profile*, which
is a property of the traffic, not of the code.

**Fix.**

```bash
# Confirm the method reached tier 4 at all before drawing any conclusion.
java -XX:+PrintCompilation -jar app.jar 2>&1 | grep -E 'payableCents|discountFor'
#   Look for the tier number in the compilation lines. Topic 74 taught you to read these.

# The interpreter control: everything allocates. If your "optimised" run does not
# differ from this, nothing was optimised.
java -Xint -jar app.jar

# The C1 control: compiled, but no C2, therefore no escape analysis.
java -XX:TieredStopAtLevel=1 -jar app.jar
```

And structurally: **run the experiment against the containerised service under the Topic 65
load profile**, for long enough to be at steady state, with the same CPU limit as
production. A measurement taken in a different compilation regime than production is not a
measurement of production.

---

## Hands-on proof

### Setup

```bash
mkdir -p ~/java-lab/75 && cd ~/java-lab/75
java --version
uname -m                # aarch64 on Apple Silicon; record it, results are per-architecture
```

Get async-profiler from its releases page. **Record JDK version, async-profiler version and
`uname -m` alongside every result** — a profile without those three is not reportable.

### Proof 1 — establish your thresholds

```bash
java -XX:+PrintFlagsFinal -version | grep -E \
  'MaxInlineSize|FreqInlineSize|MaxTrivialSize|MaxInlineLevel|InlineSmallCode|HugeMethodLimit'

java -XX:+PrintFlagsFinal -version | grep -E \
  'DoEscapeAnalysis|EliminateAllocations|EliminateLocks|EliminateNestedLocks|EliminateAllocationArraySizeLimit'
```

**WHAT TO LOOK FOR:** each line gives type, name, value and origin. `{default}` means the
JDK's default; anything else means something set it. **These are your numbers; mine do not
exist.**

### Proof 2 — find out whether the diagnostic flags exist on your build

```bash
java -XX:+UnlockDiagnosticVMOptions -XX:+PrintEscapeAnalysis -version
java -XX:+UnlockDiagnosticVMOptions -XX:+PrintEliminateAllocations -version
```

| What you see | What it means |
|---|---|
| `Unrecognized VM option` | Expected on a release JDK. These are non-product flags. **Use the allocation profiler instead** — it works everywhere and answers the question you actually have. |
| It starts and prints analysis output | You are on a debug/fastdebug build. Useful, and rarer than people think. |


### Proof 3 — see an inlining decision, and learn the line format

```bash
java -XX:+UnlockDiagnosticVMOptions -XX:+PrintInlining \
     -cp out com.orderflow.lab.ea.ScalarReplacementProbe 2>&1 | head -80
```

The line structure is roughly:

```
  @ <bci>  <class>::<method> (<n> bytes)   <decision or reason>
```

*illustration of the format, not captured output*

| Field | Meaning |
|---|---|
| `@ <bci>` | Bytecode index of the call site in the caller |
| `<class>::<method>` | The callee |
| `(<n> bytes)` | **The callee's bytecode size.** Compare against the thresholds from Proof 1 |
| decision / reason | `inline (hot)`, `too big`, `hot method too big`, `not inlineable`, `virtual call`, `callee is too large`, `inlining too deep`, `no static binding`, `executed < MinInliningThreshold times`, and others |

Indentation shows nesting depth. Topic 74 gave the short version of this table; the addition
here is that **`(<n> bytes)` is the number the whole topic turns on**.

### Proof 4 — the allocation profile, with the A/B control

Run the probe under the profiler in both configurations:

```bash
# A - defaults.
java -XX:+UnlockDiagnosticVMOptions -XX:+DebugNonSafepoints \
     -cp out com.orderflow.lab.ea.ScalarReplacementProbe &
PID=$!
./asprof -e alloc -d 30 -f /tmp/75-A.html $PID
wait $PID

# B - the control.
java -XX:-DoEscapeAnalysis -XX:+UnlockDiagnosticVMOptions -XX:+DebugNonSafepoints \
     -cp out com.orderflow.lab.ea.ScalarReplacementProbe &
PID=$!
./asprof -e alloc -d 30 -f /tmp/75-B.html $PID
wait $PID
```

`-XX:+DebugNonSafepoints` improves stack attribution in compiled frames (Topic 78).

**WHAT TO LOOK FOR:** open both HTML flame graphs and search for `LineTotal`.

| What you see | What it means |
|---|---|
| `LineTotal` absent in A, present in B | **This is the result you came for.** The allocation exists only when escape analysis is disabled. |
| Present in both, similar weight | Not scalar-replaced. Check that the loop reached C2 (`-XX:+PrintCompilation`), and re-read the method for an escape you missed. |
| Absent in both | Suspicious. Either the profiler is not sampling (raise the rate with `--alloc 1k`), or the whole loop was eliminated. Check that the checksum is printed and depends on the inputs. |
| A dominates with `Long`/`Integer` boxes you did not write | Autoboxing from Topic 01, probably in your loop scaffolding. Fix the scaffolding — you are measuring the wrong thing. |

**A caution about the instrument.** async-profiler's alloc mode is **sampled**, driven off
TLAB events: it gives *relative* weight of allocation sites, not exact object counts. Right
tool for "which sites allocate"; wrong tool for "how many bytes per operation" — use `-prof gc`.

### Proof 5 — bytes per operation, the sharpest instrument

```java
package com.orderflow.bench;

import org.openjdk.jmh.annotations.*;
import java.util.concurrent.TimeUnit;

@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@State(Scope.Thread)
@Fork(3) @Warmup(iterations = 5) @Measurement(iterations = 5)
public class ScalarReplacementBench {

    // Non-final, non-constant: keeps javac and C2 from folding the whole thing away.
    @Param({"1999"}) long unitPriceCents;
    @Param({"3"})    int quantity;
    @Param({"2000"}) int taxBasisPoints;

    record LineTotal(long netCents, long taxCents) {
        long grossCents() { return netCents + taxCents; }
    }

    public static LineTotal escaped;   // used by the leaking benchmark below

    @Benchmark
    public long nonEscaping() {
        long net = unitPriceCents * quantity;
        long tax = net * taxBasisPoints / 10_000L;
        return new LineTotal(net, tax).grossCents();      // returns a PRIMITIVE
    }

    @Benchmark
    public long escaping() {
        long net = unitPriceCents * quantity;
        long tax = net * taxBasisPoints / 10_000L;
        LineTotal t = new LineTotal(net, tax);
        escaped = t;                                       // GlobalEscape
        return t.grossCents();
    }
}
```

```bash
mvn clean verify
java -jar target/benchmarks.jar ScalarReplacementBench -prof gc -f 3 -wi 5 -i 5
java -jar target/benchmarks.jar ScalarReplacementBench -prof gc -f 3 -wi 5 -i 5 \
     -jvmArgs "-XX:-DoEscapeAnalysis"
```

The `-prof gc` output includes rows whose column structure is:

```
Benchmark                                  Mode  Cnt  Score  Error   Units
<name>:·gc.alloc.rate                      avgt  <n>  <n>    <n>     MB/sec
<name>:·gc.alloc.rate.norm                 avgt  <n>  <n>    <n>     B/op
<name>:·gc.count                           avgt  <n>  <n>            counts
```

*illustration of the format, not captured output*

**`gc.alloc.rate.norm` is the number.** It is bytes allocated per benchmark operation.

| What you see | What it means |
|---|---|
| `nonEscaping` at or near `0` B/op | Scalar replacement happened. This is the proof, and it is unambiguous in a way a flame graph is not. |
| `escaping` at the object's real size in B/op | Expected. Compare against `jol` for the exact size — 24 or 32 bytes on a classic layout, less with `[JAVA 25]` compact headers. |
| `nonEscaping` non-zero and equal to `escaping` | Not scalar-replaced. Find the escape. |
| Both zero even with `-XX:-DoEscapeAnalysis` | The benchmark is broken — something folded the computation. `@Param` values are not constants; check you did not make them `static final`. |
| Any B/op that is not a multiple of 8 | You are averaging over something else too. Isolate. |

### Proof 6 — lock elision, using the mark word from Topic 85

```java
@Benchmark
public String synchronizedBuffer() {
    StringBuffer sb = new StringBuffer();     // synchronized methods, NoEscape object
    sb.append("order ").append(unitPriceCents).append(" x ").append(quantity);
    return sb.toString();
}

@Benchmark
public String unsynchronizedBuilder() {
    StringBuilder sb = new StringBuilder();   // no locks at all
    sb.append("order ").append(unitPriceCents).append(" x ").append(quantity);
    return sb.toString();
}
```

```bash
java -jar target/benchmarks.jar 'Bench.(synchronizedBuffer|unsynchronizedBuilder)' -f 3
java -jar target/benchmarks.jar 'Bench.(synchronizedBuffer|unsynchronizedBuilder)' -f 3 \
     -jvmArgs "-XX:-EliminateLocks"
```

| What you see | What it means |
|---|---|
| The two are close by default, and `synchronizedBuffer` gets worse with `-XX:-EliminateLocks` | **Lock elision demonstrated.** The delta is the CASes you got back. |
| No difference with the flag | Either the locks were not elided anyway (check the object escapes), or the benchmark is dominated by the `toString` allocation, which is real and does escape. |
| `synchronizedBuffer` slower in both | Fine and unsurprising — `StringBuffer` has other costs. **Do not conclude lock elision is fake from this**; the flag comparison is the experiment, not the cross-benchmark comparison. |

Note that `toString()` returns a `String` that **escapes** — so this benchmark can never
reach zero allocation, and it is not trying to. It isolates the *lock*, not the allocation.

---

## Failure drill

### The assignment, restated from the master plan

> Write a method allocating a `Point`-like value object; verify zero allocation with
> async-profiler's alloc mode; then store the reference in a static field and watch
> allocation reappear. Use `-XX:-DoEscapeAnalysis` as the A/B control.

**What the drill proves:** scalar replacement is real, it is measurable, and it is
destroyed by a one-line change that no reviewer would flag.

### Step 0 — the control, before anything else

You cannot detect a change you have no baseline for.

```bash
# Learn what your tooling prints on a case you already understand.
java -XX:+PrintFlagsFinal -version | grep -E 'DoEscapeAnalysis|EliminateAllocations'
./asprof --version
java --version && uname -m
```

Write those four facts at the top of your notes file. They belong in the result.

### Step 1 — the standalone reproduction first

Use `ScalarReplacementProbe` from Example 1 verbatim. **Build the standalone case before
touching `orderflow`.** If you cannot demonstrate it in twenty lines, you will not be able
to interpret it inside a Spring service.

```bash
javac -d out com/orderflow/lab/ea/ScalarReplacementProbe.java

# Run 1: no escape, defaults.
java -XX:+UnlockDiagnosticVMOptions -XX:+DebugNonSafepoints \
     -cp out com.orderflow.lab.ea.ScalarReplacementProbe & PID=$!
./asprof -e alloc -d 30 -f /tmp/drill-1-noescape.html $PID ; kill $PID

# Run 2: no escape, escape analysis OFF. THE CONTROL.
java -XX:-DoEscapeAnalysis -XX:+UnlockDiagnosticVMOptions -XX:+DebugNonSafepoints \
     -cp out com.orderflow.lab.ea.ScalarReplacementProbe & PID=$!
./asprof -e alloc -d 30 -f /tmp/drill-2-noescape-noea.html $PID ; kill $PID

# Run 3: static field store, defaults. THE BREAKAGE.
java -XX:+UnlockDiagnosticVMOptions -XX:+DebugNonSafepoints \
     -cp out com.orderflow.lab.ea.ScalarReplacementProbe leak & PID=$!
./asprof -e alloc -d 30 -f /tmp/drill-3-leak.html $PID ; kill $PID
```

### Step 2 — capture the same thing in exact numbers

Flame graphs show *where*; JMH `-prof gc` shows *how much*. Do both. Use
`ScalarReplacementBench` from Proof 5:

```bash
java -jar target/benchmarks.jar ScalarReplacementBench -prof gc -f 3 -wi 5 -i 5 \
  | tee /tmp/drill-bench-default.txt

java -jar target/benchmarks.jar ScalarReplacementBench -prof gc -f 3 -wi 5 -i 5 \
     -jvmArgs "-XX:-DoEscapeAnalysis" | tee /tmp/drill-bench-noea.txt
```

### Step 3 — put it inside `orderflow`

The standalone case is the mechanism. The service is the point.

1. Add a feature-flagged field to `PricingEngine`:
   ```java
   // Flagged so you can flip it under load without a redeploy.
   public static volatile PriceBreakdown lastBreakdown;

   public long payableCents(long productId, long unitPriceCents, int quantity, int taxBp) {
       long list = unitPriceCents * quantity;
       long discount = discountFor(productId, list);
       long tax = (list - discount) * taxBp / 10_000L;
       PriceBreakdown b = new PriceBreakdown(list, discount, tax);
       if (CAPTURE_LAST_BREAKDOWN) {   // static final boolean, set per deployment
           lastBreakdown = b;
       }
       return b.payableCents();
   }
   ```
   **Note the deliberate subtlety:** even guarded by a branch, the store is on *a* path
   through the method, so C2 must assume the object may escape. **A guarded escape is still
   an escape.** Verify this yourself — it is the most surprising result in the drill, and
   the one worth reporting.

2. Deploy two pods differing only in `CAPTURE_LAST_BREAKDOWN`.

### Step 4 — run it under load, unchanged

```bash
# The UNCHANGED Topic 65 load profile. Do not add an endpoint for the flag;
# the point is that a field store on a hot path degrades endpoints nobody touched.
k6 run --vus 200 --duration 15m load/orderflow-baseline.js

# Profile both pods over the same window.
./asprof -e alloc -d 120 -f /tmp/drill-orderflow-off.html $(pgrep -f 'orderflow.*pod-a')
./asprof -e alloc -d 120 -f /tmp/drill-orderflow-on.html  $(pgrep -f 'orderflow.*pod-b')

# GC frequency, same window, both pods.
grep -c 'Pause Young' /var/log/orderflow/pod-a/gc.log
grep -c 'Pause Young' /var/log/orderflow/pod-b/gc.log
```

### Step 5 — what to capture

| Artefact | Command | Why it is in the report |
|---|---|---|
| JDK, async-profiler and arch versions | `java --version`, `./asprof --version`, `uname -m` | A result without these is not reportable |
| Printed thresholds and escape flags | `-XX:+PrintFlagsFinal` | Proves you did not quote a constant |
| Three alloc flame graphs (standalone) | `asprof -e alloc` | The *where* |
| Two JMH `-prof gc` runs | `-prof gc` with and without `-XX:-DoEscapeAnalysis` | The *how much* — the strongest evidence |
| Two `orderflow` alloc profiles | `asprof -e alloc` on both pods | The mechanism, in the real service |
| Young-collection counts, both pods, same window | `grep -c 'Pause Young'` | Connects allocation rate to GC frequency (Topic 68) |
| p50/p95/p99 vs `/docs/java/baselines/` | k6 summary | Connects GC frequency to user-visible latency |
| `PrintInlining` for `payableCents` and `discountFor` | `-XX:+PrintInlining` | Rules inlining in or out as a confounder |

### Step 6 — how to read it

| What you see | What it means |
|---|---|
| `LineTotal`/`PriceBreakdown` absent by default, present with `-XX:-DoEscapeAnalysis`, present with the static store | **Full result.** All three legs. Write it up. |
| Absent by default and *also* absent with the static store | The store did not happen (check your flag), or the site did not reach C2 during the window. Extend the run and check with `-XX:+PrintCompilation`. |
| Present in every configuration | Something else makes it escape unconditionally. Read the method again; check whether an inlined callee stores it. |
| Alloc profiles differ but young-collection counts do not | Your allocation rate change is too small relative to everything else `orderflow` allocates. **Report that honestly** — it is a real and useful finding: this optimisation is not always material. |
| p99 unchanged despite a clear allocation difference | Also a real finding. The system is not allocation-bound at this load. **Say so.** Resisting the urge to claim a latency win you did not observe is the entire professional point of the drill. |

### Step 7 — the fix, and the honest write-up

Remove the field store. Re-run. Confirm you return to the baseline profile.

Then write three sentences you would put in a PR description:

1. What you measured, with which instrument, on which JDK and architecture.
2. What changed between A and B, and what did **not** change.
3. What you are *not* claiming. (If p99 did not move, say so.)

**That third sentence is the deliverable** — it is what separates this from a blog post.

---

## Measurement

### The instruments for this topic, in order of authority

| Question | Instrument | Authority |
|---|---|---|
| How many bytes does one operation allocate? | **JMH `-prof gc` → `gc.alloc.rate.norm`** | Highest. Exact, per-operation, comparable across runs |
| Which sites allocate, in a running service? | **async-profiler `-e alloc`** | High for *relative* weight; it is sampled, so not exact counts |
| Did this call get inlined, and why not? | `-XX:+UnlockDiagnosticVMOptions -XX:+PrintInlining` | Definitive for the decision; very high volume |
| How big is this method, in bytecodes? | `javap -c -p` | Definitive |
| Did escape analysis matter at all? | `-XX:-DoEscapeAnalysis` A/B | **The control that makes every other number mean something** |
| Is scalar replacement specifically the mechanism? | `-XX:-EliminateAllocations` | Separates scalar replacement from the analysis |
| Is lock elision the mechanism? | `-XX:-EliminateLocks` | Same idea for monitors |
| What is this object's real size? | JOL (`org.openjdk.jol`) | Definitive; needed to interpret B/op |
| Did the method even reach C2? | `-XX:+PrintCompilation` | Prerequisite for every claim above |

### Why a naive `System.nanoTime()` measurement is WRONG here

This is the standing rule of the curriculum (Topic 77), and this topic is where it is
sharpest, because the naive harness does not merely give a noisy answer — **it changes the
answer.**

```java
// EVERY LINE OF THIS IS WRONG. It is here so you can name the errors.
long start = System.nanoTime();
for (int i = 0; i < 10_000_000; i++) {
    grossForLine(1999L, 3, 2000);       // result unused
}
long elapsed = System.nanoTime() - start;
System.out.println(elapsed / 10_000_000 + " ns/op");
```

Name the failures:

1. **Dead-code elimination.** The result is unused, so C2 may delete the call entirely. You
   time an empty loop. The "optimisation" you are celebrating is the removal of your
   benchmark.
2. **Constant folding.** All arguments are compile-time constants. C2 may compute the
   result once and hoist it out of the loop.
3. **No warm-up.** The first executions are interpreted, then C1. **Escape analysis is a C2
   optimisation and does not exist in either.** You are averaging three different programs.
4. **One JVM, shared profile.** If you measure the escaping and non-escaping variants in
   the same process, the second inherits a profile polluted by the first (Topic 74). They
   may converge on the same compiled code and appear identical.
5. **It measures time, and the quantity of interest is bytes.** Even a perfect timing
   harness would be answering the wrong question. Allocation rate matters via *GC
   frequency*, which a microbenchmark's wall clock hides. **`gc.alloc.rate.norm` is the
   quantity; time is a proxy for it at best.**
6. **On-stack replacement.** A long-running loop gets OSR-compiled, which is a different
   compilation with different characteristics from the steady-state method compilation you
   care about.

**The correct harness is JMH with `-prof gc`, `@Fork(3)`, real warm-up, and a returned
value derived from non-constant `@Param` inputs.** And even then, the number means nothing
without the `-XX:-DoEscapeAnalysis` control run beside it.

### The one thing JMH cannot tell you

JMH measures a method. It cannot tell you whether the change matters to `orderflow` at the
Topic 65 baseline. A 24-byte-per-operation saving is enormous in a tight loop and
irrelevant if the request also allocates a JSON response, a Hibernate entity graph, and a
few dozen strings.

**So the sequence is always:**

1. Alloc profile of the running service → find the *top* allocation sites.
2. JMH → quantify the candidate fix in B/op.
3. `-XX:-DoEscapeAnalysis` control → prove the mechanism is what you think.
4. Topic 65 load profile against the recorded baseline → prove the system-level effect,
   **or honestly report that there is none.**

Skipping step 4 is how teams end up with unreadable code that optimises nothing.

### What to track continuously in production

| Number | Where from | Why |
|---|---|---|
| **Allocation rate** (MB/s) | Micrometer `jvm.gc.memory.allocated`, or GC log deltas | The leading indicator for this entire topic. A step change at a deploy is the signal |
| **Young-collection frequency** | `-Xlog:gc*`, or `jvm.gc.pause` count | Allocation rate's consequence; what actually moves p99 |
| **p99 per endpoint vs baseline** | k6 / your APM, compared to `/docs/java/baselines/` | The only number the business has |
| **Code cache used** | `jvm.memory.used{area="nonheap"}` | Aggressive inlining costs code cache; Topic 74's Trap 4 |

**Allocation rate belongs on the dashboard permanently.** It is cheap, it is a leading
indicator, and almost nobody graphs it.

---

## Practice exercises

### 1 — Easy: build your own escape-analysis fact sheet

Produce a one-page file, `~/java-lab/75/facts.md`, containing **only values you printed**:

- `java --version`, `uname -m`.
- Printed values of `MaxInlineSize`, `FreqInlineSize`, `MaxTrivialSize`, `MaxInlineLevel`,
  `InlineSmallCode`, `HugeMethodLimit`.
- Printed values of `DoEscapeAnalysis`, `EliminateAllocations`, `EliminateLocks`,
  `EliminateNestedLocks`, `EliminateAllocationArraySizeLimit`.
- Whether `-XX:+PrintEscapeAnalysis` is accepted or rejected on your build.
- The bytecode sizes, from `javap -c -p`, of three real methods in `orderflow` that you
  believe are hot — and, for each, whether it is under `MaxInlineSize`, between the two
  thresholds, or over `FreqInlineSize`.

**Success criterion:** you can answer "what is `FreqInlineSize` on your JDK" without
guessing, and you have found at least one real `orderflow` method sitting close to it.

### 2 — Medium: the audit (combines Topics 01, 17, 21, 27, 68, 69, 74, 76, 77, 78)

Take the top ten allocation sites from an alloc profile of `orderflow` at the Topic 65
baseline. For each, produce a row:

| Site | Object | Size (JOL) | Escapes? | Why | Cheapest fix | Would it matter? |

Rules that make this an exercise rather than a listing:

- **"Escapes?"** must be justified from the code path, not guessed. Where you are unsure,
  write "unknown — would need `-prof gc` on an isolated benchmark".
- **"Size"** must come from JOL, not from arithmetic in your head. Note whether your JDK
  uses compact headers.
- **"Would it matter?"** must reference the site's share of total allocation from the
  profile. Most rows should say **no**. If none of your rows say no, you are not being
  honest.
- At least one row must be a **boxing** site (Topic 01) and at least one must be a
  **lambda capture** (Topic 21).

**Success criterion:** you can name the *two* sites worth acting on and defend why the
other eight are not, in a design review, without anyone being able to say "but did you
measure?".

### 3 — Hard: production simulation — quantify the inlining cliff end to end

Reproduce Example 2's incident deliberately, and measure every link in the chain.

1. Find the real bytecode size of `PricingEngine.discountFor` in your build.
2. **Grow it deliberately**, in increments, until `PrintInlining` flips from inlined to
   `too big`. Record the size at which it flips and compare against your printed
   `FreqInlineSize`. Growing it with realistic code (guarded logging, range checks) is
   better than padding, because you want to know how many *plausible* lines it takes.
3. At each increment, capture: `PrintInlining` decision, JMH `gc.alloc.rate.norm` for a
   benchmark of `payableCents`, and an alloc profile of the running service.
4. Run the unchanged Topic 65 load profile at the pre-flip and post-flip builds. Record
   p50/p95/p99 per endpoint against `/docs/java/baselines/`, and young-collection counts.
5. Apply the **cold-path extraction** fix. Confirm `PrintInlining` flips back, B/op returns,
   and p99 returns — **or report that p99 never moved**, with the allocation numbers that
   show why not.
6. Write it up as a one-page incident note with a "what I am not claiming" section.

**Success criterion:** you can state, with your own numbers, how many bytecodes of
plausible-looking code it took to lose the optimisation, and whether losing it was
observable at the system level on your hardware. **Either answer is a pass. Fabricating the
one you expected is the only failure.**

---

## Interview questions

### Q1 — "Prove that allocation was eliminated."

**MID-LEVEL.** "Escape analysis removes objects that don't escape the method, so the JIT
scalar-replaces it and there's no allocation."

**SENIOR.** "I'd measure it, because the claim is unfalsifiable as stated. Three steps.
First, a JMH benchmark of the method that returns a primitive derived from the object —
not a `Blackhole.consume` of the object itself, because that forces it to escape and I'd be
measuring a scenario that doesn't exist in production. Second, run it with `-prof gc` and
read `gc.alloc.rate.norm`: bytes per operation. Third — and this is the part people skip —
run the identical benchmark with `-XX:-DoEscapeAnalysis` as the control. If B/op is near
zero in the first and equals the object's JOL size in the second, the delta *is* the
optimisation and I can put both numbers in the PR. If they're the same, escape analysis
wasn't doing anything and I should stop repeating that it was. On the running service I'd
use async-profiler's alloc mode instead, understanding it's sampled and gives relative
weight rather than exact counts."

**What separates them:** the senior answer contains a **control**. The mid-level answer
contains a mechanism. A mechanism explains how something could be true; a control
establishes whether it is. The senior answer also volunteers the failure mode of its own
instrument — `Blackhole` and sampling — which is the tell that the person has actually run
this rather than read about it.

**Follow-up:** *"Your `-prof gc` shows zero bytes per op even with `-XX:-DoEscapeAnalysis`.
What happened?"* → Something eliminated the whole computation. Most likely the inputs are
compile-time constants and got folded, or the result isn't returned. Make the inputs
`@Param` or `@State` fields, return the value, and re-run. **Zero in both arms means the
benchmark is broken, never that the optimisation is twice as good.**

---

### Q2 — "A method got 30 lines longer and an unrelated method got slower. Explain."

**MID-LEVEL.** "Bigger methods are slower to execute, and it probably fell out of some
cache."

**SENIOR.** "Most likely the bigger method crossed the inlining budget. C2 has two budgets
in bytecodes — a small one for cold callees and a much bigger one for hot ones,
`MaxInlineSize` and `FreqInlineSize`; I'd print the actual values rather than quote them.
Once the callee is too big, it isn't inlined, and the *caller* now has an opaque call in
its graph. That's what makes the caller slower: not the call overhead, which is small, but
everything the caller loses because it can no longer see through it — constant propagation,
null-check elimination, and specifically escape analysis. Objects that were being
scalar-replaced in the caller now get allocated for real, so allocation rate rises and
young collections get more frequent. I'd confirm with `PrintInlining` grepped for that
method, look for `too big`, and check the real size with `javap -c`. The fix is to extract
the cold branches into separate methods so the hot path shrinks back under budget — not to
raise `FreqInlineSize` globally, which changes the budget for every method in the JVM and
trades code cache for it."

**What separates them:** the senior answer identifies that **the loss is in the caller, not
the callee**, and names the specific downstream optimisation. That inversion is the whole
insight of the topic, and it is what makes the bug findable — you go looking at the diff of
a method that is not the slow one.

**Follow-up:** *"How would you prevent this class of regression?"* → Alloc rate on the
dashboard with an alert on step changes at deploy; a JMH benchmark with a `-prof gc`
assertion in CI for the two or three genuinely hot methods; and, culturally, a review norm
that hot methods stay small and cold logic lives in extracted methods. **Not** a rule
banning logging.

---

### Q3 — "What is lock elision, and when would you rely on it?"

**MID-LEVEL.** "The JIT removes locks it decides aren't needed, so `synchronized` is cheap
now."

**SENIOR.** "If escape analysis proves an object can't be seen by another thread, then no
other thread can ever execute `monitorenter` on it, so mutual exclusion is vacuously
satisfied and C2 removes the lock — the CAS on the mark word and the reverse CAS on exit.
Related but distinct is lock coarsening, which merges adjacent synchronized regions on the
same object. The classic beneficiary is a local `StringBuffer` or a
`Collections.synchronizedList` that never leaves the method. As for relying on it: I don't.
I'd never write `synchronized` code and defend it on the grounds that it'll be elided,
because elision requires C2, it requires the non-escape proof to survive whatever the code
looks like next quarter, and it's not visible in the source. Where it genuinely matters is
explaining why legacy code full of `StringBuffer` isn't as catastrophic as theory suggests
— it's an explanation, not a design tool. If I'm writing new code I use `StringBuilder`,
and the question doesn't arise."

**What separates them:** the senior answer distinguishes **explaining an observation** from
**depending on a behaviour**, and refuses to do the second. It also gets the mechanism
right — vacuous mutual exclusion, not "the JIT decided the lock wasn't needed", which is
the version that leads people to believe the JIT removes locks on shared objects.

**Follow-up:** *"Does an elided lock still give a happens-before edge?"* → The question
doesn't arise. If the object is provably invisible to other threads there is no thread on
the far side of the edge; if it *is* shared, the lock isn't elided. The cases are disjoint.

---

### Q4 — "Your benchmark shows the allocation is still happening. Walk me through it."

**MID-LEVEL.** "So escape analysis didn't work — maybe records aren't supported, or my JDK
is too old."

**SENIOR.** "I'd go through a fixed list before concluding anything about the JVM. One: did
the method reach C2 at all? Escape analysis is C2-only, so if I under-warmed or ran with
`TieredStopAtLevel=1` there's nothing to find. `-XX:+PrintCompilation` settles it. Two: is
my harness forcing the escape? `Blackhole.consume(object)` does exactly that — the fix is to
return a primitive derived from the object. Three: is something in the method escaping the
reference that I didn't notice — a field store on a rare branch, a throw carrying it, a
call I assumed was inlined but wasn't? A guarded store on one path still counts as an
escape on all paths. Four: is the allocation site actually the one I think? Sampling
profilers attribute stacks in compiled frames imperfectly without
`-XX:+DebugNonSafepoints`. Five: is it a merge — the object constructed in two branches and
used after? That has historically defeated scalar replacement and recent JDKs improved it,
so I'd check my release's notes rather than assume. Only after all five would I say
anything about the JVM's capabilities, and I'd say it with the `-XX:-DoEscapeAnalysis`
control run beside it."

**What separates them:** an ordered checklist that starts with **"is my instrument
lying?"** rather than "is the platform broken?". Blaming the runtime is almost always
wrong, and the senior answer knows the specific ways their own measurement betrays them.

**Follow-up:** *"Which of those five is most common in practice?"* → The `Blackhole` one, in
benchmarks; the guarded field store, in production code. The first wastes a day; the second
ships.

---

### Q5 — "Should we make all our small carrier types records to get scalar replacement?"

**MID-LEVEL.** "Yes, records are immutable value types so the JIT handles them better."

**SENIOR.** "Records are a good default for carriers for reasons that have nothing to do
with this — generated `equals`/`hashCode`, a canonical constructor, and final fields, which
give safe publication for free, that's Topic 88. But 'we'll get scalar replacement' isn't
a reason, because a record is an ordinary object at runtime; it isn't a value type, it has
a header, and it's scalar-replaced under exactly the same conditions as any other class:
it must be provably non-escaping in C2-compiled code. Making a class a record does not
change its escape state. What *does* change the escape state is where you store the
reference — and a record you put in a list or return from a public method escapes just like
anything else. So my answer is: use records because they're the right carrier, keep hot
methods small so inlining succeeds, avoid storing carriers in fields on hot paths, and then
measure whether any of it moved allocation rate at the Topic 65 baseline. If it didn't, we
have cleaner code and no performance claim, which is a perfectly good outcome to report."

**What separates them:** the senior answer **decouples a good practice from a false
justification**. It also names what a record does and does not change at runtime — the
mid-level answer contains a real misconception ("value types") that will cause wrong
decisions later. And it ends by proposing to check whether it mattered, including the
possibility that it did not.

**Follow-up:** *"What would change that answer?"* → Value classes from Project Valhalla, if
and when they land, are the feature that would make "make it a value type" a real
performance argument, because a value class has no identity and can be flattened rather
than requiring a proof. **I'd point at the JEP index rather than predicting flags or dates.**

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. Escape analysis runs **after** inlining, on the merged graph. Derive from that alone why
   a change to a method you never call directly can change the allocation rate of a method
   you do — and name the one command that confirms it.

2. A field store on a **rarely-taken branch** still makes the object `GlobalEscape`.
   Explain why C2 has no choice about this, and then explain why "but the branch is almost
   never taken" is not an argument the compiler can use.

3. Scalar replacement is a **C2-only** optimisation. Derive the consequence for: (a) the
   first thousand requests after a deploy, (b) a benchmark with two warm-up iterations,
   (c) a pod running at `--cpus=0.5`. Give the mechanism for each, not just the outcome.

4. `Blackhole.consume(obj)` makes `obj` escape. Explain why the *purpose* of `Blackhole`
   makes this unavoidable rather than a bug in JMH, and design the benchmark shape that
   measures a non-escaping allocation correctly.

5. Lock elision on a `NoEscape` object emits **no memory barriers**. Explain why that is
   safe, and then explain why this means you can never use a lock on a thread-confined
   object to establish a happens-before edge with another thread.

6. You disable escape analysis with `-XX:-DoEscapeAnalysis` and the allocation profile does
   not change. **List everything that rules in and everything it rules out**, and name what
   you would check first.

7. Your alloc profile shows a clear difference between two builds, but p99 against the
   Topic 65 baseline is identical. State what you would write in the PR description — and
   why writing "improves latency" would be a professional failure rather than a
   simplification.

---

## Quick reference card

### The model

```
javac emits `new`. ALWAYS. javac does not do this optimisation. (Topic 76)

C2 pipeline:
  parse bytecode -> INLINE callee bodies into the caller's graph
                 -> optimisation passes
                 -> ESCAPE ANALYSIS on the merged graph
                 -> NoEscape? -> SCALAR REPLACE (no allocation; fields in registers)
                             -> ELIMINATE LOCKS (mark-word CAS removed)
                 -> more passes, now with fields as scalars
                 -> machine code

escape states:  NoEscape     -> scalar replacement + lock elision
                ArgEscape    -> no scalar replacement; does not escape the thread
                GlobalEscape -> nothing. Field store, return, throw, collection,
                                uninlined call, Blackhole.consume

INLINING IS UPSTREAM OF ALL OF IT.
No inline -> opaque call -> no proof -> real allocation, IN THE CALLER.

C2 ONLY. Not the interpreter. Not C1. So: warm-up matters, staging lies,
CPU quota matters.
```

### Flags

| Flag | Use |
|---|---|
| `-XX:+PrintFlagsFinal -version` | **Print thresholds. Never quote one you did not print.** |
| `-XX:+UnlockDiagnosticVMOptions -XX:+PrintInlining` | The inlining decision, per call site, with callee byte size |
| `-XX:-DoEscapeAnalysis` | **The A/B control for the whole topic** |
| `-XX:-EliminateAllocations` | Scalar replacement off, analysis still on — the finer control |
| `-XX:-EliminateLocks` | Lock elision off |
| `-XX:+DebugNonSafepoints` | Better profiler stack attribution in compiled frames |
| `-XX:+PrintCompilation` | Did it reach C2 at all? Prerequisite for every claim |
| `-Xint` | Interpreter only. The "everything allocates" control |
| `-XX:TieredStopAtLevel=1` | C1 only. Compiled, no escape analysis |
| `-XX:CompileCommand=inline,<class>.<method>` | Force one inline. Last resort; comment it; expire it |
| `-XX:+PrintEscapeAnalysis`, `-XX:+PrintEliminateAllocations` | **Likely rejected on a release JDK.** Non-product flags |

### Reading `PrintInlining` reasons

| Reason | Meaning | What to do |
|---|---|---|
| `inline (hot)` | Inlined. Downstream optimisation is possible | Nothing |
| `too big` / `hot method too big` | Exceeded the bytecode budget | **Extract cold paths.** The central lever of this topic |
| `virtual call` | Polymorphic call site | Topic 74 — profile pollution, seen from the inlining side |
| `not inlineable` | Native, or otherwise ineligible | Usually nothing to do |
| `callee is too large` / `inlining too deep` | Depth or accumulated-size budget | Restructure, or accept |
| `no static binding` | Receiver type undetermined | Make the site monomorphic if you can |
| `executed < MinInliningThreshold times` | Not hot enough yet | Warm it more, or accept — it is cold for a reason |

### Diagnostic commands

```bash
# What are my thresholds and escape flags?
java -XX:+PrintFlagsFinal -version | grep -Ei 'inline|escape|eliminate'

# How big is this method, really?
javap -c -p -cp target/classes com.orderflow.pricing.PricingEngine | sed -n '/discountFor/,/^$/p' | tail -5

# Was it inlined, and why not?
java -XX:+UnlockDiagnosticVMOptions -XX:+PrintInlining -jar app.jar 2>&1 | grep discountFor

# Bytes per operation - the sharpest instrument.
java -jar target/benchmarks.jar Bench -prof gc -f 3 -wi 5 -i 5

# The control that makes it mean something.
java -jar target/benchmarks.jar Bench -prof gc -f 3 -wi 5 -i 5 -jvmArgs "-XX:-DoEscapeAnalysis"

# Which sites allocate, in the running service?
./asprof -e alloc -d 60 -f /tmp/alloc.html $(pgrep -f orderflow)

# Object size, exactly. Needed to interpret B/op.
java -cp jol-cli.jar org.openjdk.jol.Main internals com.orderflow.pricing.PriceBreakdown

# Did it reach C2?
java -XX:+PrintCompilation -jar app.jar 2>&1 | grep payableCents
```

### Gotchas checklist

- [ ] Never claim scalar replacement without the `-XX:-DoEscapeAnalysis` control.
- [ ] `Blackhole.consume(obj)` makes `obj` escape. Return a primitive instead.
- [ ] A field store on **any** path, including a guarded one, is `GlobalEscape`.
- [ ] `javap` shows `new` regardless. Bytecode cannot answer this question.
- [ ] Escape analysis is **C2 only**. Confirm the method reached tier 4 first.
- [ ] The bytecode budget is in **bytecodes**. SLF4J varargs boxing is bigger than it looks.
- [ ] Do not raise `FreqInlineSize` globally; scalar-replaced arrays need a constant length.
- [ ] `-prof gc` gives bytes; async-profiler alloc gives relative weight. Different questions.
- [ ] `[JAVA 25]` compact headers change B/op, not whether the optimisation happens; and if
      p99 did not move, **say p99 did not move.**

---

## When would I use this at work?

**1. Reviewing a PR that adds logging or validation to a hot method.**

You have a ten-second question that catches a whole regression class: *"what is this
method's bytecode size now, and what is `FreqInlineSize` on our JDK?"* Two commands answer
it. If it crossed the line, the fix is a five-minute extraction of the cold branches, and
you have converted "this method is getting long" from a taste argument into a mechanical
one. Over a year this catches more silent latency drift than any load test, because the
diffs that cause it look completely benign and never get a second look.

**2. Adjudicating an allocation-rate argument without a rewrite.**

Somebody proposes restructuring a pricing or serialisation path "to reduce allocations".
You can settle it in an afternoon: alloc-profile the running service to see whether the
site is even in the top ten, JMH the candidate with `-prof gc` to get B/op, run the
`-XX:-DoEscapeAnalysis` control to confirm the mechanism, then run the Topic 65 load
profile against the recorded baseline to see whether it moves p99. **Most of the time the
honest answer is "this is real and it doesn't matter here"**, and being the person who can
say that with evidence saves the team a week of code churn.

**3. Explaining a post-deploy latency drift nobody can attribute.**

The signature — allocation rate up, young-collection frequency up, pause durations flat,
heap dump clean, no obvious change to the slow endpoint — is one you now recognise
immediately. Everyone else on the call is looking at the endpoint that got slower. You go
and look at the diff of the *small method it calls*, which is where the change actually
was. Being the person who knows to look one frame down is a durable, visible difference in
an incident review.

---

## Connected topics

**Prerequisites:**

- **01 — Boxing**: every autobox is an allocation. The SLF4J varargs boxing in Example 2 is
  half the regression, and it is a Topic 01 bug wearing a Topic 75 costume.
- **17 — Immutability and `final`**: small immutable carriers are exactly the objects this
  optimisation targets. `final` fields also matter for a reason this topic does not cover —
  that is Topic 88.
- **21 — Lambdas**: a capturing lambda allocates. Whether that allocation survives is this
  document's question, and non-capturing lambdas are a singleton so the question never
  arises.
- **27 — Records**: the right carrier, for reasons that are not performance. A record is an
  ordinary object at runtime; being a record changes nothing about its escape state.
- **65 — The load baseline**: the only place a system-level claim can be settled.
  `/docs/java/baselines/`.
- **68 — TLAB and allocation**: what an allocation actually costs, and why allocation rate
  drives young-collection frequency. Scalar replacement removes the allocation at source,
  which is strictly better than making it cheaper.
- **69 — Object layout and headers**: why an object costs 16 bytes rather than 8, which is
  what `gc.alloc.rate.norm` is counting. `[JAVA 25]` compact headers change the number.
- **70–71 — GC fundamentals and G1**: allocation rate → collection frequency → p99. The
  chain that makes this topic matter to anyone outside the JVM team.
- **73 — Safepoints**: the other thing C2 decides about your code. Rule it out before
  blaming allocation.
- **74 — JIT I**: tiers, profiles, deoptimization, and the fact that all of this is C2-only.
  **This document is Topic 74's payoff**; without it, "the compiler will fix it" is faith.
- **76 — Bytecode and `javap`**: how you measure a method's real size, and the proof that
  javac does none of this.
- **77 — JMH**: the measurement authority. `-prof gc` is the sharpest instrument in this
  document, and the `Blackhole` trap is a JMH trap.
- **78 — Profiling**: async-profiler's alloc mode, and why `-XX:+DebugNonSafepoints`
  matters for attribution in compiled frames.
- **79 — Heap dumps**: what a *clean* heap dump means during this incident — garbage, not a
  leak — and why that misleads people.
- **82 — Containers**: CPU quota starves compiler threads, delaying tier 4, which delays
  every optimisation in this document.
- **85 — `synchronized` and the mark word**: the CAS that lock elision removes, and why
  removing it is a proof rather than a gamble.

**This unlocks:**

- **86–87 — The JMM**: the compiler's licence to reorder and eliminate is the same licence
  used here. An elided lock emits no barriers, and understanding why that is safe is
  understanding what a barrier is for.
- **88 — Final fields and safe publication**: the constructor freeze, and the counterpart
  question — not "can this object be optimised away" but "can another thread see it half
  built". Topic 17's promised payoff.
- **92 — Concurrent collections**: putting an object into a `ConcurrentHashMap` is a
  textbook `GlobalEscape`, and also a safe publication. Two topics, one line of code.
- **94 — Explicit locks**: `ReentrantLock` is an object with fields, not a monitor on a
  mark word — so lock elision does not apply to it. Worth knowing before you assume the
  JIT will clean up after you.
- **99 — jcstress**: the tool for the questions this document cannot answer, because
  "correct" and "fast" need different instruments.
- **101 — Virtual threads**: a virtual thread's stack is a heap object, which changes what
  "escapes" means for objects captured across a park. Worth revisiting this topic after it.
- **122 — Docker and startup**: warm-up is when none of these optimisations exist yet, which
  is why the first requests after a container start are slow.
- **129 — Capacity and cost**: allocation rate is an input to the capacity model. An
  optimisation that is real but immaterial does not change the model, and saying so is the
  job.

---

*Java baseline 21, running on JDK 25. Three things here are deliberately hedged rather than
asserted: the default values of the inlining thresholds on your build (print them), whether
`-XX:+PrintEscapeAnalysis` exists on your JDK (a non-product flag in mainline; likely
rejected), and which release improved C2's handling of allocation merges (check your JDK's
release notes, not a teaching document). Each has a command that settles it. **No allocation
figure, bytes-per-operation number, inlining transcript, GC frequency or latency percentile
in this document was measured — I have no JVM.** The `PrintInlining` line and the `-prof gc`
table are format illustrations with `<n>` placeholders, labelled as such. What will still be
true at 2am: escape analysis proves an object cannot be observed outside its allocating
method; C2 then scalar-replaces it, so the object is never allocated and its fields become
registers; lock elision is the same proof applied to monitors; all of it requires inlining
first; none of it happens outside C2; and none of it is true in your code until you have run
the `-XX:-DoEscapeAnalysis` control and seen the difference.*
