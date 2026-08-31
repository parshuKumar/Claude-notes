# 70 — GC Fundamentals: Reachability, Roots, Allocation Math, Throughput vs Latency

## Phase: 8 — JVM Internals
## Category: DIFFERENTIATOR
## Java baseline: 21  |  Notes features from: 21 / 25
## Project spine: measure `orderflow`'s **allocation rate** and **live set** at the Topic 65 baseline load. Those two numbers are the input to every collector, heap-sizing and container-limit decision in the rest of Phase 8. Baselines live in `/docs/java/baselines/`.

---

## Mechanical statement

Read this three times. The rest of the document is an elaboration of it.

> **A garbage collector does not find garbage. It finds the living, and then declares
> everything else free.** It starts from a fixed set of **GC roots** — thread stacks,
> static fields, JNI handles, thread-locals, held monitors, and a few JVM-internal
> structures — and traverses every strong reference it can reach. What it reaches is
> *live*. What it does not reach is, by definition, garbage. The collector never looks
> at a dead object. It does not know dead objects exist.
>
> **"Unreachable" is a graph property. "Unused" is a human intention. They are not the
> same thing, and the gap between them is the definition of a Java memory leak.** A
> `static Map` holding an `Order` that will never be read again is reachable from a root,
> therefore live, therefore retained forever. Nothing is broken. The collector is doing
> exactly its job.
>
> **Therefore the cost of a collection scales with the LIVE SET, not with the garbage.**
> A copying young collection visits the survivors, copies them, and reclaims the whole
> space in one step. Ten megabytes of dead objects in eden cost *nothing* — they are not
> visited, not swept, not touched. Ten megabytes of *live* objects cost a traversal and a
> copy, on every single collection, forever.
>
> **Which means the two numbers that describe your service to a collector are its
> allocation rate (MB/s) and its live set (MB).** Not its request rate. Not its heap
> size. Not its flags. Everything else in Phase 8 is downstream of those two numbers, and
> you can measure both from your own GC log with subtraction.

Five consequences follow directly, and you should be able to derive each one:

1. **Doubling the rate at which you create short-lived objects costs almost nothing.**
   It makes collections more *frequent*, not more *expensive*.
2. **Doubling your live set costs on every collection, forever.** This is the asymmetry
   that decides where optimisation effort goes.
3. **Object pooling to "reduce GC pressure" usually makes GC worse**, because a pooled
   object survives by construction, and survival is the thing that costs.
4. **A leak in Java is never a failure to free. It is a reference you forgot you held.**
   The diagnostic question is never "what leaked" but "what still points at it".
5. **Old-generation objects pointing at young-generation objects have to be found on
   every young collection.** That is what card tables and remembered sets are for, and it
   is why the write barrier costs you instructions on every reference store.

---

## The bridge from what you know

### HONEST ANALOGUE — and this is one of the strongest in the whole curriculum

**Reachability and GC roots work the same way in V8.** Not "similarly". The same.

V8 is a tracing collector. It starts from roots — the global object, the execution
stacks, handle scopes held by native code, and a few internal structures — and marks
everything reachable through strong references. Everything unreached is swept or, in the
young generation, simply abandoned when the survivors are evacuated. Objects are live
because something points at them, and for no other reason.

Everything you already believe about JavaScript memory transfers directly:

- **A closure capturing a variable keeps it alive.** In Java, an inner class capturing
  its enclosing instance, or a lambda capturing a local (Topic 21), does exactly the same
  thing for exactly the same reason. Topic 79's classic leak shapes are the Java spellings
  of leaks you have already debugged in Node.
- **A global array you push to and never splice from is a leak.** In Java it is a
  `static List` field, and it is Topic 79's first drill. Same shape, same cause.
- **An event listener you add and never remove is a leak**, because the emitter holds a
  reference to your handler and the handler closes over your object. Java: a listener
  registered on a long-lived bean, an unremoved `PropertyChangeListener`, a cache with no
  eviction. **Identical mechanism.**
- **`WeakMap` and `WeakRef` exist because sometimes you want a reference that does not
  keep the target alive.** Java has `WeakReference`, `WeakHashMap`, `SoftReference`,
  `PhantomReference` and `Cleaner`, with more precise semantics and more ways to misuse
  them — but the *idea* is one you already have.
- **Cycles are fine.** Both runtimes trace rather than reference-count, so a cycle of
  objects unreachable from any root is collected without special handling. If your
  intuition ever says "circular reference leak", that intuition came from CPython or
  from old Internet Explorer, and it is wrong in both V8 and the JVM.

Keep all of that. It is correct and it is load-bearing. **You are not learning
reachability from scratch; you already understand it. What you are learning is what to do
with it now that you can see it.**

### What is genuinely new: the cost model, and the ability to measure it

Three things you have never had:

**1. GC cost scales with the LIVE SET, not the garbage — and you can prove it.**

You probably half-know this from Node folklore ("don't retain things"). What you have
never had is the ability to *demonstrate* it, because V8 gives you no way to hold
allocation rate constant while varying the live set and observe the collector's response.
The JVM does, and this document's Example 1 is exactly that experiment. **Moving from
"I've heard retention is bad" to "I measured that doubling the live set doubled young-
collection duration at constant allocation rate" is the whole difference between folklore
and engineering.**

**2. You can measure both numbers, from a log, with subtraction.**

Node has no GC log worth the name. The JVM emits a structured, timestamped record of
every collection, and two subtractions on consecutive lines give you allocation rate and
promotion rate. The Measurement section of this document is a formula you apply to your
own output. **There is no equivalent in your previous career.** This is the single most
practically valuable thing in Phase 8's early topics.

**3. You can enumerate the roots.**

In V8 the root set is an implementation detail you read about in blog posts. In the JVM
it is a list you can recite, each entry maps to a category of real bug, and one of them —
the JIT-emitted **oop map** at each safepoint — is the mechanical bridge to Topic 73.
"Where are the roots" is a question with a concrete answer, and heap-dump tools (Topic 79)
will show you the *path from a root* to any object you select.

| You know | Java | Verdict |
|---|---|---|
| Tracing GC from roots | Tracing GC from roots | **HONEST ANALOGUE** — the concept is identical |
| Closures retain captured variables | Lambdas/inner classes retain captured references | **HONEST ANALOGUE** |
| Cycles are collected fine | Cycles are collected fine | **HONEST ANALOGUE** |
| `WeakMap`, `WeakRef`, `FinalizationRegistry` | `WeakHashMap`, `WeakReference`, `Cleaner` | **PARTIAL** — same idea, four reference strengths instead of one, and `SoftReference` has no JS counterpart |
| "Retaining objects is bad" as folklore | **Cost ∝ live set**, measurable in MB and ms | **NO ANALOGUE** — you gain the measurement, and the obligation to take it |
| No GC log | `-Xlog:gc*`, structured and timestamped | **NO ANALOGUE** |
| One collector, no choice | G1 / Parallel / Serial / ZGC / Shenandoah | **NO ANALOGUE** — Topic 72 makes you choose |
| Heap size is one flag you set once | Heap size, live set, headroom and collector as a **joint** decision | **NO ANALOGUE** |

> **The theme, stated plainly:** your reachability intuition is correct and transfers
> whole. What you are adding is a *cost model* and a *ruler*. From here on, "we have a GC
> problem" is not a diagnosis — it is a claim with two numbers attached, or it is nothing.

---

## What is this?

### Reachability, precisely

An object is **live** if there exists a chain of **strong** references from at least one
**GC root** to that object. Otherwise it is **garbage**, and the collector may reclaim it
at a time of its choosing.

That is the entire definition. Three things people expect to be in it and are not:

- **Whether your code will ever use the object again.** The collector cannot know that
  and does not try.
- **Whether the object is "finished", "closed", or "done".** Calling `close()` on a
  resource does not make it unreachable.
- **Whether you set the variable to `null`.** Nulling a *local* usually does nothing,
  because the local was about to go out of scope anyway (and the JIT may have already
  stopped tracking it — see the oop-map discussion below). Nulling a *field* on a
  long-lived object is a real and often necessary act.

### The GC roots — the list you should be able to recite

| Root category | Concretely, in `orderflow` | The bug shape it produces |
|---|---|---|
| **Thread stacks** — locals and operand-stack entries in every frame of every live thread | The `Order` your controller method is holding right now | A thread parked forever holds its whole stack live (Topic 90) |
| **Static fields** of every loaded class | `static final Map<Long, Product> CACHE` | The unbounded static cache — Topic 79's first drill |
| **Thread objects and their `ThreadLocal` maps** | A request context stored in a `ThreadLocal` on a pooled thread | The subtlest leak in Java — Topic 79's second drill |
| **JNI local and global references** | Anything a native library holds | Invisible in a normal heap dump's Java view |
| **Monitors currently held** — objects being `synchronized` on | The inventory lock (Topic 85) | Rarely a leak; matters for root-scan cost |
| **Class loaders and loaded classes** | Spring's `ApplicationContext`, Hibernate's session factory | A class loader is a root for everything it loaded — Topic 67, and the reason redeploy leaks exist |
| **The interned-string table** | `String.intern()` results (Topic 18) | Interning moves a leak; it does not remove one |
| **JVMTI / agent references** | Your APM agent, a debugger, a profiler | Topic 81 |
| **`Reference` objects pending processing** | Anything queued to a `ReferenceQueue` or `Cleaner` | Delays reclamation by at least one cycle |

**The one that matters most for Phase 8**, because it connects to two other topics:

> For **compiled** code, the JIT emits an **oop map** at every safepoint: a table saying
> "at this program counter, stack slot 3 and register `rbx` hold object references, and
> everything else does not". Without it the collector could not tell an `int` on the stack
> from a pointer. This is why the world must stop at a *safepoint* rather than at an
> arbitrary instruction, and it is the mechanical link to Topic 73. It is also why
> "nulling a local to help GC" is largely folklore: **the compiler already knows the local
> is dead and simply omits it from the oop map.**

### Tracing, not reference counting

Java collectors trace. They do not maintain a per-object reference count. Consequences:

- **Cycles are collected**, with no special machinery.
- **No cost is paid on reference assignment for counting** — though a *write barrier* is
  paid for generational bookkeeping, which is a different thing (see below).
- **Reclamation is not deterministic.** An object becomes unreachable at some instant;
  it is reclaimed whenever the collector next runs and gets to it. Anything that needs
  deterministic release needs `try-with-resources` (Topic 08), not the collector.

### The three collection algorithms, and what each costs

| Algorithm | How it works | Cost is proportional to |
|---|---|---|
| **Mark-sweep** | Mark live objects, then sweep the rest into free lists | Marking ∝ **live**; sweeping ∝ **heap size**; leaves fragmentation |
| **Mark-compact** | Mark, then slide live objects together | ∝ **live** (marking and moving), plus updating every reference to a moved object |
| **Copying / evacuating** | Copy live objects to a fresh space; declare the old space entirely free | ∝ **live only.** The dead are never touched. No fragmentation. Costs 2× the space |

**Young collections in every generational HotSpot collector are copying collections.**
That is where the central claim comes from and why it is so strong: *the dead cost
literally nothing*, because reclaiming them is one pointer assignment that marks the whole
region free.

### Throughput, latency, footprint — pick two

Every collector trades along three axes, and there is no configuration that wins all
three:

| Axis | Definition | Measured as |
|---|---|---|
| **Throughput** | Fraction of CPU time spent running your code rather than the collector | `1 − (GC CPU time / total CPU time)` |
| **Latency** | How long any single pause stops the application | pause duration distribution — **p99 and max, never mean** |
| **Footprint** | How much memory the whole thing needs to run well | RSS, and the heap headroom required to avoid failure |

- **Parallel GC** maximises throughput and accepts long pauses. Right for batch.
- **G1** targets a pause goal at a modest throughput cost. The default, and right for
  most services.
- **ZGC / Shenandoah** make pauses nearly independent of heap size, paid for in throughput
  (concurrent barriers) and footprint (headroom). Topic 72.
- **Serial GC** minimises footprint and complexity. Right for tiny containers — and it is
  what a JVM may silently pick when the container makes it see one CPU (Topic 82).

**Latency and pause time are not the same thing.** If `orderflow`'s p99 is dominated by a
Postgres query, moving from 40 ms pauses to 0.5 ms pauses changes p99 by almost nothing.
Collector choice follows a *latency budget*, which requires knowing where the latency
currently goes (Topic 78).

---

## Why does it matter?

**1. Because "we have a GC problem" is the most common misdiagnosis in Java production
work.**

The symptom — latency spikes — is shared by GC pauses, time-to-safepoint (Topic 73),
connection-pool exhaustion (Topic 55), CPU throttling in the container (Topic 82),
downstream tail latency, and page-cache stalls. **The only way to tell them apart is
evidence**, and the evidence for the GC hypothesis is exactly the two numbers this
document teaches you to derive. Being able to say "our allocation rate is X MB/s and our
live set is Y MB, so a young collection should take roughly Z, and the log agrees" is how
you *exonerate* GC in five minutes and go look at the real cause.

**2. Because the live-set/garbage asymmetry redirects optimisation effort.**

Teams spend weeks removing allocations from hot paths. Sometimes that is right — a high
allocation rate does cost CPU, does churn TLABs, and does increase collection *frequency*.
But if your problem is long pauses and rising old-gen occupancy, **allocation is not your
problem and removing it will not help.** The correct effort goes to retention: what is
live, why, and for how long. Knowing which of the two situations you are in, from
measurement, is the difference between a week well spent and a week wasted.

**3. Because it is the reason object pooling is usually wrong in Java.**

This is genuinely counter-intuitive coming from any other runtime. Pooling converts an
object that would have died in eden — free — into one that lives across collections,
gets copied into survivor space repeatedly, and eventually gets promoted to old gen where
it is expensive forever. **Pooling does not reduce GC pressure. It converts cheap garbage
into expensive live data.** Trap 2 is the full treatment.

**4. Because heap sizing is arithmetic on these two numbers.**

"How much heap does `orderflow` need?" is not answered by a rule of thumb or by the
container limit. It is: measured live set, plus enough headroom that the collector never
runs out of space to copy into, plus room for the allocation rate to be absorbed between
collections. Topic 82 turns that into container flags. **You cannot do that arithmetic
without the measurements in this document.**

**5. Because every remaining topic in Phase 8 assumes these two numbers exist.**

Topic 71's collection-set sizing, Topic 72's collector comparison, Topic 79's leak hunt,
Topic 82's container tuning — every one of them starts with "what is your allocation rate
and live set". This is the topic that produces them.

---

## Machine-level reality

### Root enumeration — what actually happens at the start of a collection

A young collection begins with the world stopped at a safepoint (Topic 73), and the first
phase is **root scanning**. Concretely:

1. **Every thread's stack is walked frame by frame.** For interpreted frames the JVM
   knows the layout from the method's metadata. For compiled frames it reads the **oop
   map** the JIT emitted for that exact program counter. Each identified reference becomes
   a root.
2. **Every loaded class's static fields are scanned.** This is why class-loader leaks are
   so total: a retained loader keeps every class it loaded, which keeps every static field
   in those classes, which keeps everything those point at.
3. **JNI global references, JVMTI references, held monitors, and the interned-string
   table** are enumerated.
4. **Remembered-set / dirty-card entries are added as roots for the young generation** —
   see the next section. This is usually the largest of the four in a big-heap service.

**WHAT TO LOOK FOR** in a phase breakdown (`-Xlog:gc+phases=debug`):

| What you see | What it means |
|---|---|
| Ext Root Scanning is a large share of the pause | Many threads (each stack is walked), or very large static structures. Topic 90 for thread count, Topic 79 for statics. **Thousands of virtual threads change this materially — Topic 101.** |
| Scan RS / Update RS is a large share | High reference-mutation rate: lots of old→young pointer stores. See the card table below. |
| Object Copy dominates | The normal case, and it means the **live set** is what costs. Topic 68's survivor sizing and this document's retention story. |
| Termination / other is large | Worker threads finishing at very different times; often a CPU-quota problem (Topic 82). |

### The card table and the write barrier — why old→young references cost

Here is the problem generational collection creates for itself.

A young collection wants to collect only the young generation. To do that it needs every
root that points *into* the young generation. Thread stacks and statics are easy. But an
**old-generation object may hold a reference to a young-generation object** — you assigned
a fresh `OrderLine` into a long-lived `Order`'s list, say. That old object is a root for
this collection.

Finding those by scanning the whole old generation would make the young collection cost
proportional to the *old* generation, destroying the entire point of generational GC.

So the JVM maintains a summary, and pays for it on every reference store:

> **The heap is divided into "cards" — small fixed-size ranges, classically 512 bytes.
> After every store of a reference into an object field, the JIT emits a few extra
> instructions that mark the card containing that field as "dirty". At the next young
> collection, the collector scans only the dirty cards for old→young references.**

Those few extra instructions are the **write barrier**, and they are the standing tax
generational GC charges your application. Note carefully what triggers it:

```java
order.setStatus(OrderStatus.PAID);   // reference store  → write barrier
order.setTotalMinorUnits(4999L);     // primitive store  → NO write barrier
lines.add(line);                     // reference store into the array → write barrier
```

**Reference stores cost; primitive stores do not.** That is a small but real argument for
primitive representations (Topic 69) beyond memory.

**G1 generalises this into per-region remembered sets** (Topic 71): rather than one global
card table, each region records which locations outside it point into it, maintained by
concurrent *refinement threads* draining a queue of dirty cards. That is more precise and
more expensive, and it is why G1 has lower raw throughput than Parallel GC.

**The design consequence you can act on:** a data structure where old objects constantly
acquire references to new objects — a long-lived cache whose values are frequently
replaced, a long-lived list you keep appending to — generates a high dirty-card rate and
makes every young collection more expensive. **Rebuilding a small structure is sometimes
cheaper than mutating a large long-lived one**, which is the opposite of the intuition
most people bring.

### Floating garbage — why heap-used-after-GC is not the live set

Concurrent collectors mark while the application runs. To stay correct, they use a
snapshot discipline (SATB in G1: an object reachable when marking *started* is treated as
live for this cycle, even if it dies during it). So:

```
heap_used_after_collection  =  live_set  +  floating_garbage  +  TLAB_waste  +  fragmentation
```

**Every term on the right except the first is unknown and varies.** This is why estimating
the live set requires the *floor* over many collections rather than a single reading, and
why a single "heap used after GC" number is a bad estimate.

It is also a diagnostic trap in its own right: a rising old-gen occupancy over a few
cycles can be floating garbage from a slow concurrent cycle rather than a leak. The
distinguishing measurement is whether the **floor after full or mixed collections, over a
long run**, is flat or climbing.

### Reference strengths — the four, and where each is a trap

| Strength | Cleared when | Correct use | The trap |
|---|---|---|---|
| **Strong** (normal) | Never — it is the reference | Everything | The leak: you still hold it |
| **Soft** | At the collector's discretion, "before OOM" | Memory-sensitive caches, in theory | **Almost always wrong** — see Trap 5. Softs are cleared *late*, so they hold the live set at maximum and cause the pressure they were meant to relieve |
| **Weak** | As soon as no strong reference remains | Canonicalising maps, listener registries, metadata keyed by object identity | `WeakHashMap` keys are weak but **values are strong**, and a value referencing its own key pins the entry forever |
| **Phantom** | After the object is finalizable; you cannot dereference it | Post-mortem cleanup with `Cleaner` | Needs an active `ReferenceQueue` drain; a stalled `Cleaner` is a native-memory leak (Topic 80) |

**Reference processing happens inside a pause** (G1's Remark phase). A service with a very
large number of soft/weak references can have Remark dominated by reference processing —
visible with `-Xlog:gc+ref=debug`, and one of the few legitimate uses of
`-XX:+ParallelRefProcEnabled`.

> `finalize()` is deprecated for removal and should be treated as unavailable. It made
> objects survive an extra cycle, could resurrect them, and ran on an unbounded queue with
> no timing guarantee. Use `try-with-resources` for deterministic release and `Cleaner`
> for a native-resource safety net. Never both as the primary mechanism.

### The two numbers, and what they predict

```
allocation_rate  (MB/s)  — how fast the application produces new objects
live_set         (MB)    — how much is reachable at steady state
```

From those, with eden size, you can predict the collector's behaviour before you run it:

```
young_collection_frequency   ≈  eden_size / allocation_rate         // collections per second
young_collection_duration    ≈  k × surviving_bytes                 // k is a copy cost per byte
gc_cpu_overhead              ≈  frequency × duration × parallelism
```

**Note which term the live set appears in.** Allocation rate sets the *frequency*; the
live set sets the *duration*. That is the asymmetry, expressed as arithmetic:

- **Halving allocation rate** halves the number of collections. Each still costs the same.
  Modest CPU win, no effect on pause *duration*, so **no effect on p99**.
- **Halving the live set** halves the duration of every collection **and** reduces old-gen
  pressure, concurrent-cycle frequency, and full-GC risk. **This is where p99 lives.**

---

## Example 1 — minimal

The point of this example is to **prove the central claim**: at identical allocation rate,
a larger live set makes every collection more expensive.

### The probe

```java
package com.orderflow.lab.gc;

import java.util.ArrayDeque;
import java.util.Deque;

/**
 * Holds allocation rate CONSTANT and varies the LIVE SET.
 *
 * args[0] = retained objects (the live set)
 * args[1] = total allocations to perform
 *
 * Every run allocates the same number of identical objects. The ONLY difference
 * between runs is how many of them are still reachable at any moment.
 */
public final class LiveSetProbe {

    /** ~1 KB of payload plus a reference, so the live set is easy to reason about. */
    record Payload(long id, byte[] blob) { }

    public static void main(String[] args) throws Exception {
        int  retained = Integer.parseInt(args[0]);
        long total    = Long.parseLong(args[1]);

        // A bounded ring: exactly `retained` objects are reachable at any instant.
        Deque<Payload> live = new ArrayDeque<>(Math.max(retained, 1));

        for (long i = 0; i < total; i++) {
            Payload p = new Payload(i, new byte[1024]);
            live.addLast(p);
            while (live.size() > retained) {
                live.removeFirst();      // becomes unreachable => garbage
            }
        }

        // Keep the ring reachable to the very end so it is unambiguously in the live set.
        System.out.println("done, live=" + live.size());
    }
}
```

### Run it — three arms, one variable

```bash
javac -d out --release 21 src/main/java/com/orderflow/lab/gc/LiveSetProbe.java

# Identical heap, identical collector, identical total allocation.
# The ONLY difference is the first argument: the live set.
COMMON="-Xms512m -Xmx512m -XX:+UseG1GC"
LOG="file=logs/liveset-ARM.log:time,uptime,level,tags"

java $COMMON -Xlog:gc*,gc+heap=debug:${LOG/ARM/small}  -cp out com.orderflow.lab.gc.LiveSetProbe      1000 2000000
java $COMMON -Xlog:gc*,gc+heap=debug:${LOG/ARM/medium} -cp out com.orderflow.lab.gc.LiveSetProbe    50000 2000000
java $COMMON -Xlog:gc*,gc+heap=debug:${LOG/ARM/large}  -cp out com.orderflow.lab.gc.LiveSetProbe   200000 2000000
```

### What to extract

```bash
for arm in small medium large; do
  echo "=== $arm ==="
  # How many young collections happened?
  grep -c 'Pause Young' logs/liveset-$arm.log
  # The pause durations, so you can look at the distribution rather than a mean.
  grep 'Pause Young' logs/liveset-$arm.log | grep -oE '[0-9]+\.[0-9]+ms$' | sort -n | tail -5
  # Did anything worse than a young collection happen?
  grep -cE 'Pause Full|To-space exhausted|Evacuation Failure' logs/liveset-$arm.log
done
```

**WHAT TO LOOK FOR:** three things, in this order.

1. **The number of young collections should be roughly the same across all three arms.**
   Same total allocation, same eden — same frequency. This is the control that proves you
   really did hold allocation rate constant.
2. **The pause durations should rise with the live set.** This is the claim.
3. **The largest arm may show evacuation failure or a Full GC.** That is the live set
   approaching the heap, which is a different failure and worth seeing.

| What you see | What it means |
|---|---|
| Collection counts roughly equal, durations rise with retention | **The central claim, demonstrated on your machine.** Cost tracks the live set. Write the three numbers down. |
| Collection counts differ substantially between arms | Your arms are not comparable — the larger live set left less room in eden, changing frequency too. Reduce the retained sizes, or raise `-Xmx`, and re-run. |
| Durations do not rise | Your live set is too small relative to the heap for the effect to clear the noise. Increase the spread — try 100, 100 000, 400 000 — and re-run each arm three times. |
| The large arm throws `OutOfMemoryError` | 200 000 × ~1 KB exceeds a 512 MB heap once you count headers and the deque (Topic 69). Lower it, or raise the heap. **Predict the number before adjusting it.** |
| The large arm shows Full GCs and enormous pauses | The live set is a large fraction of the heap, so the collector has almost no room to work. This is the shape of a real production incident. |
| All arms show a Full GC at the very end | Often the JVM shutting down. Ignore collections after the "done" line. |

### The variant that completes the lesson

Now hold the **live set** constant and vary the **allocation rate**: run with `retained =
50000` and `total` of 500 000, 2 000 000, and 8 000 000.

**WHAT TO LOOK FOR:** collection *count* should scale roughly with total allocation, while
each collection's *duration* should stay roughly flat.

| What you see | What it means |
|---|---|
| Count scales, duration flat | **The other half of the asymmetry.** Allocation rate buys you frequency; the live set buys you duration. Together these two experiments are the whole mechanical statement. |
| Duration also rises with allocation rate | Some of what you allocated is surviving — check the promotion rate. At very high allocation rates, objects can be promoted prematurely because survivor space overflows (Topic 68's Trap 4). |
| Total wall time barely changes | Allocation really is nearly free. This is the number that kills "allocation is expensive" as a general claim. |

**This pair of experiments is worth more than the rest of the document.** Do them before
you read on.

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
| Baseline artefacts | `/docs/java/baselines/` |

### The task, which is the spine assignment

> Measure `orderflow`'s allocation rate and live set at baseline load, and write both into
> `/docs/java/baselines/`.

This is not a warm-up. **These two numbers are cited by Topics 71, 72, 79, 82 and 90.**
Get them right once, record the method, and re-measure whenever the service changes
materially.

### Step 1 — get a log you can do arithmetic on

The decorators matter as much as the tags. Without `uptime` you have no denominator.

```bash
# In the container's JAVA_TOOL_OPTIONS or the Dockerfile's ENTRYPOINT.
-Xms1200m -Xmx1200m -XX:+UseG1GC \
-Xlog:gc*,gc+heap=debug,gc+phases=debug:file=/var/log/orderflow/gc.log:time,uptime,level,tags:filecount=5,filesize=50m
```

| Decorator | Why you need it |
|---|---|
| `time` | Wall clock — correlate with k6's timeline and your APM |
| `uptime` | **The denominator of every rate.** Without it you cannot compute MB/s |
| `level` | Distinguish `info` summaries from the `debug` lines you asked for |
| `tags` | **Grep on tags, not on message text** — message text changes between JDKs |

`gc+heap=debug` is what gives you per-generation occupancy. **The summary line's
`before->after` is whole-heap and cannot tell you promotion.**

### Step 2 — run the baseline, unchanged

```bash
# Confirm what you are actually running BEFORE generating load.
jcmd $(pgrep -f orderflow) VM.flags -all | grep -E "UseG1GC|MaxHeapSize|InitialHeapSize"

# The UNCHANGED Topic 65 load profile. Do not tune anything for this measurement.
k6 run --out json=results/baseline-gc.json load/orderflow-baseline.js
```

**Ten minutes minimum.** A two-minute run can miss every concurrent cycle and give you a
live-set estimate that is pure floating garbage. If your baseline shows infrequent
concurrent cycles, run longer.

### Step 3 — apply the formulas

They are in the Measurement section below, in full, with sanity checks. Apply them to
**your** log. Do not look for numbers in this document; there are none, deliberately.

### Step 4 — what the answers mean for `orderflow` specifically

Once you have the two numbers, here is how to read them against this service's shape.

**On allocation rate.** `orderflow` at baseline allocates from four main sources, and you
should be able to name them before you profile:

| Source | Why it allocates | Topic |
|---|---|---|
| Hibernate entity hydration + **snapshots** | The persistence context keeps a snapshot of every loaded entity for dirty checking — **roughly doubling** the memory per loaded entity | 48 |
| JSON serialisation | Buffers, intermediate strings, `byte[]` | 44, 69 |
| DTO mapping | One object per row, per request | 47, 49 |
| Boxing on the counting/aggregation paths | `Long`/`Integer` per value (Topic 69: 24 and 16 bytes) | 01, 69 |

If your measured allocation rate is high **and** promotion is flat, that is *healthy* and
needs no action. Say so explicitly in the baseline document, because someone will
otherwise try to "fix" it.

**On the live set.** At steady state `orderflow`'s live set should be roughly: the Spring
context and Hibernate metadata (fixed, tens of MB), plus any caches, plus the in-flight
request working set — which is *concurrency × per-request retention*. The last term is the
one that grows under load and the one that matters:

```
in_flight_retention  ≈  concurrent_requests  ×  bytes_retained_per_request
```

**A `GET /orders/{id}` that loads an order with 500 lines retains all 500 hydrated
entities plus their snapshots for the duration of the transaction.** At 400 rps with even
a modest latency, that is a live set that scales with your load — which means **the live
set is not a constant, and a heap that is comfortable at 200 rps may not be at 600.** This
is the single most important `orderflow`-specific insight in this topic, and it connects
directly to Topic 49 (fetch strategy) and Topic 50 (N+1).

### Step 5 — write the artefact

```
/docs/java/baselines/gc-baseline.md

  Measured on:        <date>, <JDK version and vendor>, <image tag>, <git sha>
  Container:          2 vCPU, 2 GiB
  Flags:              -Xms1200m -Xmx1200m -XX:+UseG1GC   (confirmed via jcmd VM.flags -all)
  Load:               unchanged Topic 65 profile, <N> minutes, open model
  Method:             consecutive-young-collection subtraction (Topic 70 formula)

  allocation_rate:    <MB/s>   (median across <N> intervals; IQR <..>)
  promotion_rate:     <MB/s>   (median; note whether it is flat or climbing)
  live_set_floor:     <MB>     (floor of old-gen occupancy after mixed collections)
  gc_cpu_overhead:    <%>      (sum of pause durations x parallelism / wall clock)
  young_collections:  <count>, median duration <ms>, p99 <ms>
  full_collections:   <count>  -- MUST be zero; any nonzero value is an incident

  Interpretation:     <"high allocation, flat promotion => healthy" or whatever is true>
  Re-measure when:    heap size, collector, container limits, JDK, dataset size,
                      or the load profile changes; or a release adds a cache
```

### The change that would cause an incident

Now the counterfactual, so you can see what these numbers protect you from.

1. A release adds an in-process product cache (Topic 69's Example 2) — **live set up by
   100 MB, permanently.**
2. The same release adds an `open-session-in-view` workaround (Topic 49) so a lazily-loaded
   association can be serialised. The persistence context now lives for the whole request
   rather than the transaction, so **per-request retention roughly doubles**, and the
   in-flight live set scales with concurrency.
3. Allocation rate barely changes. Every dashboard that tracks allocation says "fine".
4. The live set rises. Old-gen occupancy crosses G1's initiating threshold sooner; concurrent
   cycles run more often; mixed collections have more to copy; pause durations rise.
5. At the traffic peak, the live set plus in-flight retention leaves G1 too few free regions
   to evacuate into. **Evacuation failure, then `Pause Full`, then a multi-second stall on
   every endpoint** — including the catalogue reads that touch none of the changed code.
6. The postmortem says "GC issue" and someone proposes a bigger heap or a different
   collector. Both are downstream of the actual cause, and one of them (Topic 71's Trap 1)
   makes it worse.

**The diagnosis is one subtraction against your recorded baseline:** allocation rate
unchanged, live set up 3×. That sentence names the cause and rules out the collector in
under a minute — but only if the baseline exists.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — treating a high allocation rate as the problem when the LIVE SET drives full GCs

**Wrong approach.** `orderflow` starts showing multi-second pauses at peak. Someone
profiles allocation (Topic 78) and finds the service allocating at a high MB/s rate,
dominated by DTO mapping and Hibernate hydration. The conclusion is obvious and wrong:

> "We're allocating far too much. Let's cut allocations on the hot path — reuse DTOs,
> avoid streams, pool buffers. That'll fix the GC."

A sprint goes into it.

**Exact symptom.**

- Allocation rate drops meaningfully. Everyone is pleased.
- **Pause durations do not change.** p99 is exactly where it was.
- Young collections are *less frequent*, so total GC CPU drops slightly — a real but small
  throughput win that nobody asked for.
- `Pause Full` events continue at the same rate, at the same times, under the same load.
- Old-generation occupancy after mixed collections is unchanged.
- Morale drops, and the team concludes GC tuning is voodoo.

**Root cause.** They optimised the wrong term of the model. From the Machine-level section:

```
frequency ≈ eden_size / allocation_rate      ← what they changed
duration  ≈ k × surviving_bytes              ← what actually hurt
```

Their pauses were long because a lot of data was **surviving** — and none of the DTO
churn was surviving. **They removed garbage that was already free.** The Full GCs came
from old-gen occupancy, which is fed by promotion, which is fed by retention, which they
never touched.

**Fix.**

1. **Measure both numbers before choosing a target.** The formula is in the Measurement
   section. Allocation rate high + promotion flat = healthy; do not optimise it.
2. **Find the promotion source**, not the allocation source. Promotion rate from the
   `gc+heap=debug` old-gen deltas; then a heap dump at peak and MAT's dominator tree
   (Topic 79) to see *what* is old and *why*.
3. **For `orderflow` specifically**, the usual answers are: an unbounded or unevicted
   cache; a persistence context held too long (open-session-in-view, or a long
   `@Transactional` method); an oversized page from an endpoint with no pagination limit;
   or a request-scoped structure escaping to a longer scope (Topic 38).
4. **Then re-measure.** If the live set drops and pauses drop with it, you have confirmed
   the model.

**The one-sentence rule:** *allocation rate buys you collection frequency; the live set
buys you collection duration. Pauses are a duration problem.*

---

### Trap 2 — pooling objects that should have died young

**Wrong approach.** Following Trap 1's failed sprint, someone escalates: "if allocation is
the problem, stop allocating. Pool the objects." An object pool goes in for `OrderLineDto`
— the most-allocated type on the read path.

```java
// DO NOT DO THIS. It converts free garbage into expensive live data.
@Component
public class OrderLineDtoPool {
    private final ArrayBlockingQueue<OrderLineDto> pool = new ArrayBlockingQueue<>(50_000);

    public OrderLineDto acquire() {
        OrderLineDto dto = pool.poll();
        return dto != null ? dto : new OrderLineDto();
    }
    public void release(OrderLineDto dto) { dto.reset(); pool.offer(dto); }
}
```

**Exact symptom.**

- Allocation rate drops. The dashboard the team was watching improves.
- **Old-generation occupancy rises immediately and permanently** by roughly the pool's
  full size, and never comes back down.
- Young-collection durations **increase**, because the pool itself is now part of the live
  set being traced, and because every pooled object handed out and returned is a reference
  store that dirties cards.
- Concurrent cycles become more frequent; mixed collections take longer.
- p99 gets **worse** than before the pool was introduced.
- Occasionally, a subtle correctness bug: a pooled DTO returned to the pool while a
  response is still being serialised, producing fields from another request in a response.
  Under load. Intermittently. This is far worse than the performance problem.

**Root cause.** This is the central asymmetry, weaponised against you.

```
Without the pool: allocate in a TLAB (a pointer bump, ~10 instructions, Topic 68),
                  die in eden, cost of reclamation = ZERO. Never copied, never traced.

With the pool:    allocate once, then live forever. Traced on every collection.
                  Copied through survivor spaces. Promoted to old gen. Then it is part
                  of the live set that every concurrent cycle marks and every mixed
                  collection may have to copy. Plus a write barrier on every hand-out.
```

**Pooling did not remove a cost. It converted the cheapest possible object lifecycle into
the most expensive one.** Java allocation is a pointer bump; that is the fact that makes
pooling — which is correct in C++ and in a manual-memory runtime — actively wrong here.

**Fix.**

1. **Delete the pool.** Measure before and after against the recorded baseline.
2. **Pool only things whose cost is not allocation**: database connections (setup is a TCP
   handshake and authentication), threads (an OS resource, Topic 90), off-heap buffers
   (Topic 80 — native memory really does have a malloc-shaped cost), and very large arrays
   whose allocation is humongous under G1 (Topic 71).
3. **The test for whether pooling is legitimate:** *is the expensive part of this object's
   creation something other than the memory allocation?* If yes, pool it. If no, do not.
4. If the DTO churn genuinely costs CPU, the fix is to **allocate less** — project fewer
   columns (Topic 47), page the endpoint, stream the response — not to reuse more.

**What the fix proves:** the cheapest object is the one that dies in eden. Anything that
extends an object's life makes it more expensive, and "reuse" is a synonym for "extend its
life".

---

### Trap 3 — "unreachable" versus "unused": the leak that is not a bug in anything

**Wrong approach.** `orderflow` adds an idempotency guard so a retried `POST /orders` does
not double-charge a wallet (a real requirement — Topic 92 and Topic 115 revisit it):

```java
// DO NOT DO THIS. Correct logic, unbounded retention.
@Service
public class IdempotencyGuard {
    private static final Map<String, OrderResult> SEEN = new ConcurrentHashMap<>();

    public OrderResult once(String idempotencyKey, Supplier<OrderResult> work) {
        return SEEN.computeIfAbsent(idempotencyKey, k -> work.get());
    }
}
```

**Exact symptom.**

- Behaviour is **completely correct**. Every test passes. Retries are deduplicated. There
  is no bug in the logic, and code review will not catch it.
- Memory grows slowly and monotonically. On a service that restarts weekly, it may never
  be noticed at all — which is why this shape survives to production.
- Old-generation occupancy floor climbs over hours: **flat within an hour, clearly rising
  over a day.** Distinguishing that from a large working set requires a long run.
- Concurrent cycles get more frequent, then mixed collections lengthen, then eventually
  `OutOfMemoryError: Java heap space` — often days or weeks after the deploy that caused
  it, which destroys the correlation everyone would use to find it.
- A heap dump shows `ConcurrentHashMap$Node` and `OrderResult` dominating retained size,
  with a path to GC root through a **static field**.

**Root cause.** `SEEN` is `static`, so it is a GC root. Every entry is reachable, so every
entry is live, so nothing is ever collected. **The collector is behaving perfectly.** The
objects are unreachable-from-the-programmer's-intent and reachable-from-a-root, and only
the second one is a fact about the program.

**This is what "unreachable is not unused" means, and it is the definition of a Java leak:
a live root chain to a stale object.**

**Fix.**

| Fix | When |
|---|---|
| A bounded cache with eviction — **Caffeine** with `maximumSize` and `expireAfterWrite` | The default answer. An idempotency key is only interesting for minutes |
| Move it out of the process — Redis with a TTL (Topic 110) | Multi-instance deployments, where an in-process map is **also wrong for correctness**: a retry hitting a different pod sees nothing |
| A database uniqueness constraint on the idempotency key | The most robust option, and it survives restarts |
| `WeakHashMap` | **Almost never here** — nothing else holds the `String` key, so entries would be collected immediately and the guard would not work |

**The general rule to take away:** *any `static` collection that only ever grows is a leak.
Any cache without an eviction policy is an unbounded collection.* Make "what evicts this?"
a standing question in code review — it takes five seconds and catches the most common
production memory bug in Java.

---

### Trap 4 — doubling the heap to "fix" GC, and making pauses worse

**Wrong approach.** Pauses are too long. The container has room. Someone doubles the heap:
`-Xmx1200m` becomes `-Xmx2400m`, and the container limit goes to 4 GiB. It is the obvious
move, it is free, and it is often exactly wrong.

**Exact symptom.**

- Young collections become **less frequent** — good, and it is what people notice first.
- Young collections become **longer**, because a bigger heap means a bigger eden (young
  gen is sized as a fraction of heap), so each collection covers more allocation and has
  more survivors to copy.
- **p99 gets worse even though total GC CPU went down.** Mean improves; the tail
  regresses. If you were watching averages you would report a win.
- Concurrent cycles are rarer but each does more work; mixed collections take longer.
- If it was already close to failing, the eventual `Pause Full` is **much** longer, because
  a full collection's cost scales with the heap it has to compact.
- On a container, the JVM may now be sized past what cgroups allow once you add metaspace,
  code cache, thread stacks and direct buffers — and the pod gets **OOM-killed with a
  healthy heap** (Topic 80, Topic 82).

**Root cause.** Heap size does not reduce the cost of a collection. It changes *how often*
collections happen and *how much* each one covers. **The live set is unchanged, so the
per-object tracing and copying work is unchanged — it is just batched into fewer, bigger
pauses.** For a throughput-oriented batch job that is a straightforward win. For a
latency-sensitive service measured at p99, it is a regression.

The exception that makes people believe the myth: if the heap was so small that the
collector was running constantly and failing to reclaim enough, more heap genuinely helps
enormously. **The two situations look identical from a latency graph and completely
different in the GC log.**

**Fix.**

1. **Read the log before changing the number.** Which situation are you in?
   ```bash
   grep -c 'Pause Full' gc.log                                  # nonzero => real pressure
   grep -cE 'To-space exhausted|Evacuation Failure' gc.log      # nonzero => real pressure
   grep -c 'Pause Young' gc.log                                 # very high => thrashing
   ```
   | What you see | What it means | Does more heap help? |
   |---|---|---|
   | Full GCs, or evacuation failures | The collector genuinely cannot keep up | **Yes** — and also fix the retention |
   | Young collections every few hundred ms, short, no Full GCs | Healthy, high allocation rate | **No.** More heap makes each pause longer |
   | Long young pauses, no Full GCs, high old-gen floor | Large live set | **No.** Reduce the live set; heap does not help the duration |
   | Rising old-gen floor over hours | A leak (Trap 3) | **No.** More heap postpones the OOM and lengthens the eventual Full GC |
2. **Size the heap from the measured live set**, not from the container limit:
   `heap ≈ live_set × (2 to 3) + room for eden` — then validate under load, and remember
   that heap is only part of the process footprint (Topic 82).
3. **Set `-Xms` equal to `-Xmx`** in a container, so you never pay resizing pauses and the
   region sizing is deterministic (Topic 71).
4. **Change one thing and compare against the recorded baseline**, on percentiles, with at
   least three runs per arm.

---

### Trap 5 — a `SoftReference` cache that "self-tunes", and holds the live set at maximum

**Wrong approach.** After Trap 3, someone learns about soft references and reaches the
appealing conclusion:

> "Let's make the cache values `SoftReference`s. Then the GC clears them automatically when
> memory is tight. A self-tuning cache with no eviction policy to get wrong."

```java
// DO NOT DO THIS. The semantics are not what the name suggests.
private final Map<Long, SoftReference<ProductSummary>> cache = new ConcurrentHashMap<>();
```

**Exact symptom.** Genuinely confusing, which is why this trap survives so long:

- Heap usage sits **near the maximum, permanently.** Every graph is alarming and nothing
  is actually wrong yet.
- Old-generation occupancy is high, so concurrent cycles run near-continuously and mixed
  collections are expensive. **GC CPU overhead is high at all times.**
- Under a load spike, the collector clears a large batch of softs at once — a big,
  expensive reference-processing phase inside a pause — and **cache hit rate collapses to
  near zero simultaneously**, so every subsequent request goes to the database. **The
  latency spike and the database spike arrive together**, which makes the database look
  like the cause.
- The map itself keeps growing: the `SoftReference` objects and the `Node`s are strongly
  held even after the referents are cleared. **You now have a leak of empty wrappers.**
- Behaviour is wildly different between machines and JDKs, because clearing policy depends
  on free heap and on `-XX:SoftRefLRUPolicyMSPerMB`, which almost nobody sets deliberately.

**Root cause.** Soft references are cleared **late — at the collector's discretion, when
memory is tight.** So the design's steady state is "hold everything until the heap is
nearly full". That is precisely the state in which every collection is most expensive: the
live set is at its maximum, concurrent cycles run constantly, and there is minimal headroom
for evacuation. **The mechanism intended to relieve memory pressure guarantees the service
runs permanently under memory pressure**, and it removes the collector's headroom exactly
when a load spike needs it.

Two secondary problems make it worse: the clearing is *bulk* (correlated cache misses
rather than a smooth eviction curve), and the map entries themselves are strong, so you
must sweep cleared references yourself or the map grows without bound.

**Fix.**

1. **Use a real cache with a real policy.** Caffeine with `maximumSize` or
   `maximumWeight`, plus `expireAfterWrite` or `expireAfterAccess`. A **bounded** cache
   makes the live set a number you chose, and a chosen live set is the entire point of this
   document.
2. **Size the bound from measurement:** per-entry deep size (Topic 69) × entry count, as a
   deliberate fraction of heap after the live-set floor and headroom.
3. **Track hit rate.** A cache without a hit-rate metric cannot be tuned or justified.
   Caffeine exposes stats; wire them to Micrometer (Topic 118).
4. **Reserve weak references for their actual use** — canonicalising maps and
   identity-keyed metadata where the *key's* lifetime is controlled elsewhere. Reserve
   soft references for very nearly nothing.
5. If you must keep softs, **at minimum drain a `ReferenceQueue`** to remove dead entries,
   and know what `-XX:SoftRefLRUPolicyMSPerMB` is set to.

**What the fix proves:** a bounded cache is not a worse version of a self-tuning one. **A
bounded live set is the goal, not a compromise** — it is what makes collection cost
predictable, which is what makes p99 predictable.

---

## Hands-on proof

### Setup

```bash
mkdir -p ~/jvm-lab/logs && cd ~/jvm-lab
java -version
javac -d out --release 21 src/main/java/com/orderflow/lab/gc/*.java
```

### Proof 1 — confirm what collector you actually have

```bash
java -XX:+PrintFlagsFinal -version | grep -E "UseG1GC|UseSerialGC|UseParallelGC|UseZGC|UseShenandoahGC"
jcmd $(pgrep -f orderflow) VM.flags -all | grep -E "UseG1GC|UseSerialGC|UseZGC|MaxHeapSize"
```

**WHAT TO LOOK FOR:** exactly one collector flag `true`, and a heap size you recognise.

| What you see | What it means |
|---|---|
| `UseG1GC = true` | The default on server-class machines. Topic 71 is your reading. |
| `UseSerialGC = true` **in a container** | The JVM sees fewer than 2 CPUs or less than ~1792 MB. **A container CPU limit did this** (Topic 82), and it will dominate every measurement you take. Fix it before proceeding. |
| `UseZGC = true` | Someone chose it. Topic 72. The log format differs — learn it before writing greps. |
| Heap much smaller than the container limit | The default `MaxRAMPercentage` (25%) applied. Topic 82. |

### Proof 2 — watch reachability directly

```java
package com.orderflow.lab.gc;

import java.lang.ref.WeakReference;

public class ReachabilityProbe {
    public static void main(String[] args) throws Exception {
        Object strong = new Object();
        WeakReference<Object> weak = new WeakReference<>(strong);

        System.gc();                          // a REQUEST, not a command
        Thread.sleep(200);
        System.out.println("still strongly held: " + (weak.get() != null));

        strong = null;                        // the last strong reference is gone

        System.gc();
        Thread.sleep(200);
        System.out.println("after dropping the strong ref: " + (weak.get() != null));
    }
}
```

**WHAT TO LOOK FOR:** the two printed booleans.

| What you see | What it means |
|---|---|
| `true` then `false` | Reachability demonstrated in six lines. The object died because nothing pointed at it, and for no other reason. |
| `true` then `true` | `System.gc()` did nothing (it is a request), or `-XX:+DisableExplicitGC` is set, or the JIT kept the local alive in a way you did not expect. **This is a useful failure**: it proves `System.gc()` is unreliable, which is why Topic 69's Measurement section forbids heap-differencing. |
| `false` then `false` | The compiler determined `strong` was dead after the `WeakReference` constructor — the local was not in the oop map. **This is the oop-map fact from Machine-level reality, observed.** Add a use of `strong` after the first `System.gc()` to keep it alive. |

### Proof 3 — find what is keeping something alive

This is the skill that Topic 79 turns into a full workflow, and it is worth touching now.

```bash
# On the running service, at peak load.
jcmd $(pgrep -f orderflow) GC.class_histogram | head -30
```

**WHAT TO LOOK FOR:** the classes at the top by byte total.

| What you see | What it means |
|---|---|
| Domain entities dominate | Normal for a persistence-heavy service under load — probably in-flight requests. Check whether the count scales with concurrency. |
| `java.lang.Long`, `java.lang.Integer`, `HashMap$Node` dominate | A boxed-collection representation problem. Topic 69's Trap 2. |
| `byte[]` and `String` dominate | Serialisation buffers or a text-heavy cache. Check for humongous allocations too (Topic 71). |
| A class you do not recognise dominates | Read the path to GC root in a heap dump. It is usually a framework holding something you did not intend. |
| Counts that keep rising across repeated runs of this command | The Trap 3 shape. Take a heap dump before it OOMs. |

> **`GC.class_histogram` triggers a full collection** so it can report live objects only.
> That is a stop-the-world pause on a running service. On a load-test instance, fine. On
> production, know what you are asking for before you press enter.

### Proof 4 — see the write barrier's cost, indirectly

You cannot see the barrier instructions without disassembly (Topic 76 gets close), but you
can see the work they generate.

```bash
java -Xms512m -Xmx512m -XX:+UseG1GC \
  -Xlog:gc*,gc+phases=debug,gc+remset=debug:file=logs/remset.log:time,uptime,level,tags \
  -cp out com.orderflow.lab.gc.LiveSetProbe 50000 2000000

grep -iE 'remset|Scan RS|Update RS|refinement' logs/remset.log | head -30
```

**WHAT TO LOOK FOR:** the share of pause time in remembered-set work.

| What you see | What it means |
|---|---|
| Scan RS / Update RS a small share | Normal. Your old→young reference-mutation rate is low. |
| Scan RS / Update RS a large share | Lots of old objects acquiring references to new objects. A long-lived mutable structure — a cache whose values are replaced, or a growing list on a long-lived object. |
| Refinement threads mentioned as behind or falling behind | The dirty-card queue is not being drained fast enough. Often a CPU-quota problem (Topic 82) before it is a tuning problem. |
| No such lines at all | Your JDK uses different tag names, or the tag is not enabled. **Learn your JDK's actual field names before writing any grep** — this is the standing rule for every log in Phase 8. |

### Proof 5 — measure GC CPU overhead

```bash
# Total time spent in GC pauses, versus wall-clock time of the run.
grep -oE '[0-9]+\.[0-9]+ms$' logs/liveset-medium.log | awk '{s+=$1} END {print "total pause ms:", s}'
grep -oE '\[[0-9]+\.[0-9]+s\]' logs/liveset-medium.log | tail -1
```

```
gc_pause_overhead_percent  =  (sum of all pause durations)  /  (wall clock of the run)  × 100
```

**WHAT TO LOOK FOR:** the percentage, and its interpretation.

| What you see | What it means |
|---|---|
| Under a few percent | Healthy for a latency-sensitive service. |
| Ten percent or more | You are giving away a meaningful share of your machine. Look at live set first, then heap size, then collector. |
| Very low percentage but bad p99 | **GC is not your latency problem.** Go look at safepoints (Topic 73), pool exhaustion (Topic 55), the downstream, or CPU throttling (Topic 82). This is the single most valuable use of this measurement. |

> This measures *pause* overhead, not total GC CPU. Concurrent collectors do most of their
> work off the pause, on GC threads, competing with your application for cores — which on a
> 2 vCPU container is a real cost that this number does not show. For total CPU, compare
> process CPU time against application-thread CPU time, or use JFR.

---

## Failure drill

**Mandatory.** The assignment for this topic is the spine measurement, and the failure
mode you are drilling is **misattribution**: concluding "GC problem" from a latency graph
without the two numbers.

### The assignment

> Measure `orderflow`'s allocation rate and live set at baseline load. Then deliberately
> change **one** of them, predict the effect on the other and on p99, and verify.

### Step 0 — establish the control

```bash
# The gate rule from Topic 65 applies: if you cannot reproduce the baseline
# within +/-10%, nothing downstream is measurable. Stop and fix that first.
k6 run --out json=results/control.json load/orderflow-baseline.js
```

Confirm your recorded p50/p95/p99/p999 against `/docs/java/baselines/`.

### Step 1 — measure both numbers at baseline

Apply the Measurement formulas to a ten-minute run. Record them in
`/docs/java/baselines/gc-baseline.md` using the template from Example 2.

### Step 2 — arm A: change the allocation rate, hold the live set

Add an endpoint-level change that allocates substantially more **short-lived** data per
request without retaining any of it. A realistic one for `orderflow`:

```java
// In the catalogue read path. Allocates, then discards. Nothing survives the method.
private String buildEtag(ProductSummary p) {
    // Deliberately wasteful: several intermediate Strings and a byte[] per request.
    String basis = p.id() + "|" + p.sku() + "|" + p.updatedAt() + "|" + p.price();
    return Integer.toHexString(basis.hashCode()) + "-" + basis.length();
}
```

**PREDICT BEFORE RUNNING.** Write these down:

| Quantity | Your prediction | Reason |
|---|---|---|
| Allocation rate | | |
| Young-collection frequency | | |
| Young-collection **duration** | | |
| Promotion rate | | |
| Old-gen floor | | |
| p99 | | |

### Step 3 — arm B: change the live set, hold the allocation rate

Add retention without adding much allocation — a bounded cache populated at startup, so
the objects exist either way and the only change is that they stay reachable:

```java
@Component
public class ProductCache {
    private final Map<Long, ProductSummary> byId = new ConcurrentHashMap<>();

    @PostConstruct
    void warm() {
        // Loads once at startup: a one-off allocation, then permanent retention.
        productRepository.findAllSummaries().forEach(p -> byId.put(p.id(), p));
    }
}
```

**PREDICT BEFORE RUNNING**, using the same table.

### Step 4 — run each arm, one at a time, identically

```bash
for arm in control allocation liveset; do
  # deploy the arm, then:
  docker compose up -d orderflow
  sleep 60                                  # let the JIT warm up (Topic 74) and caches fill
  k6 run --out json=results/$arm.json load/orderflow-baseline.js   # UNCHANGED profile
  docker compose logs orderflow > logs/app-$arm.log
  cp /var/log/orderflow/gc.log logs/gc-$arm.log
  docker compose down
done
```

**Ten minutes per arm, three repetitions per arm.** A difference smaller than your
run-to-run spread is not a difference.

### Step 5 — what to capture

For each arm: allocation rate, promotion rate, live-set floor, young-collection count,
median and p99 young-collection duration, Full GC count, and the endpoint p50/p95/p99/p999.

```bash
for arm in control allocation liveset; do
  echo "=== $arm ==="
  grep -c 'Pause Young'                          logs/gc-$arm.log
  grep -cE 'Pause Full|Evacuation Failure'       logs/gc-$arm.log
  grep 'Pause Young' logs/gc-$arm.log | grep -oE '[0-9]+\.[0-9]+ms$' | sort -n | tail -3
  # Then the formulas, by hand, on YOUR fields.
done
```

### Step 6 — how to read it

| What you see | What it means |
|---|---|
| Arm A: allocation rate up, collection **count** up, **duration flat**, p99 unchanged | **The expected result and the point of the drill.** Allocation rate is a frequency lever, not a latency lever. |
| Arm A: duration also up | Some of the "short-lived" data survived. Check the promotion rate — you may have crossed a survivor-space threshold and triggered premature promotion (Topic 68). |
| Arm B: allocation rate unchanged, live set up, **duration up**, p99 up | **The other expected result.** The live set is the latency lever. |
| Arm B: p99 unchanged despite a larger live set | Your live set is still small relative to the heap, so the extra copying is below the noise floor. Increase the cache size — or accept the more valuable finding: **at this scale, the live set was not the constraint.** |
| Arm B shows Full GCs the control did not | You crossed into the region where the collector has too little headroom. This is the incident shape from Example 2, reproduced deliberately. |
| Neither arm moves p99 at all | **The most valuable outcome.** GC is not `orderflow`'s latency bottleneck at this baseline. Now go find what is — Topic 78, and Topics 55, 73 and 82 as candidates. Write this up; it saves the next person a week. |

### What the drill proves

1. **The two numbers are independent levers with different effects**, and you have now
   moved each one separately and watched the difference.
2. **A latency graph cannot tell you which one you are looking at.** Only the log can.
3. **"We have a GC problem" is a hypothesis that must be falsified before it is acted on.**
   You now own the two measurements that falsify it in five minutes.

---

## Measurement

### The instrument for this topic

**The GC log, plus subtraction.** Not a profiler, not a benchmark, not a dashboard.

A profiler tells you where CPU goes. A benchmark tells you how fast a method is. Neither
tells you the two numbers that describe your service to a collector. **Those come from a
structured log and arithmetic, and there is no tool that will do the arithmetic for you
correctly, because the correctness depends on which collections you subtract across.**

### Get the log right first

```bash
-Xlog:gc*,gc+heap=debug,gc+phases=debug:file=/var/log/orderflow/gc.log:time,uptime,level,tags:filecount=5,filesize=50m
```

Before writing a single grep, **learn your own JDK's field names**:

```bash
head -40 /var/log/orderflow/gc.log
grep 'Pause Young' /var/log/orderflow/gc.log | head -3
grep 'gc,heap'     /var/log/orderflow/gc.log | head -6
```

**Message text changes between JDK versions and between collectors.** ZGC's log looks
nothing like G1's. Every grep in this document is a starting point to adapt, not a command
to copy.

### The shape of the line you are extracting from

```
[<wall-clock>][<uptime>s][info][gc] GC(<id>) Pause Young (<Type>) (<Cause>) <before>M-><after>M(<total>M) <duration>ms
```

***Illustration of the format, not captured output. All values are placeholders showing
field positions.***

| Field | Role in the formulas below |
|---|---|
| `<uptime>` | **The denominator of every rate.** Seconds since JVM start |
| `<id>` | Groups all log lines belonging to one collection |
| `<before>` | Heap used when this collection started |
| `<after>` | Heap used when it finished — live set **plus floating garbage plus TLAB waste** |
| `<total>` | Current heap capacity |
| `<duration>` | GC work at the safepoint — **not** stop-the-world duration (Topic 73) |

Old-generation occupancy is **not** on this line. It comes from the `gc,heap` debug lines.

### FORMULA 1 — allocation rate

For two **consecutive young collections** `n` and `n+1`:

```
allocated_between  =  heap_used_BEFORE[n+1]  -  heap_used_AFTER[n]        // MB
elapsed            =  uptime_at[n+1]  -  uptime_at[n]                     // seconds
allocation_rate    =  allocated_between / elapsed                         // MB/s
```

In words: **the heap grew from "what was left after the last collection" to "what was there
when the next one started". That growth is what your application allocated.**

Do this for **many** consecutive pairs across your run and take the **median**, not the
mean. Report the interquartile range alongside it. A single pair is a sample of one and
tells you nothing about a service whose load varies.

### FORMULA 2 — promotion rate

Using **old-generation** occupancy after each collection, from the `gc,heap` debug lines:

```
promoted_between   =  old_used_AFTER[n+1]  -  old_used_AFTER[n]           // MB
promotion_rate     =  promoted_between / elapsed                          // MB/s
```

**This is the number that predicts old-gen pressure and eventual Full GCs.** Topic 68's
central point: promotion, not allocation, drives old-gen behaviour.

### FORMULA 3 — the live set

No single log line gives you this. You need a floor over time:

```
live_set  ≈  the FLOOR of old-generation occupancy immediately after MIXED or FULL
             collections, sampled over a long run at steady state
```

Why the floor and why only after mixed/full collections: a young collection does not touch
old gen, so old-gen occupancy after a young collection includes everything ever promoted
and not yet reclaimed. Only a mixed or full collection actually reclaims old regions, and
even then `heap_used_after` includes floating garbage. **The floor across many such
collections is the best estimate available from a log.**

Two independent methods, and you should agree them:

```bash
# 1. From the log, over a long run at steady state.
grep -E 'Pause Full|Pause Young \(Mixed\)' /var/log/orderflow/gc.log | tail -50

# 2. Force a full collection and read the heap.
#    NOTE: this pauses the service. Load-test instance only, NEVER production traffic.
jcmd <pid> GC.run
jcmd <pid> GC.heap_info
```

### FORMULA 4 — derived quantities you can predict and then check

```
young_collection_frequency  ≈  eden_size / allocation_rate           // collections per second
gc_pause_overhead_percent   =  sum(pause_durations) / wall_clock × 100
heap_needed                 ≈  live_set × (2 to 3) + eden headroom   // then validate under load
```

**The point of the first line is that it is falsifiable.** Compute the predicted frequency
from your measured allocation rate and eden size, then count the collections in the log. If
they disagree by more than a modest factor, one of your inputs is wrong — most often eden
size, because under G1 the young generation is adaptive and changes size at runtime.

### Extracting the fields

```bash
# Young collections: uptime, gc id, used-before, used-after, heap total.
# ADAPT THIS REGEX TO YOUR JDK'S ACTUAL FORMAT. Check with `head` first.
grep 'Pause Young' /var/log/orderflow/gc.log \
  | sed -E 's/.*\[([0-9.]+)s\].*GC\(([0-9]+)\).*[^0-9]([0-9]+)M->([0-9]+)M\(([0-9]+)M\).*/\1 \2 \3 \4 \5/'
# columns: uptime gcId usedBefore usedAfter heapTotal

# Old-generation occupancy per collection, from the gc,heap debug lines.
grep -E 'gc,heap' /var/log/orderflow/gc.log | grep -iE 'old|Old regions'
```

**I am deliberately not showing you a worked example with numbers, because any numbers I
invented would be fiction and you would remember them.** Run it on your own log and do the
subtraction.

### Sanity checks — run all four, every time

| Check | If it fails |
|---|---|
| Did you mix **young** with **mixed/full** collections in one subtraction? | Mixed and full collections reclaim old gen, so the promotion delta goes negative. Subtract only across consecutive collections of the same kind. |
| Did a concurrent cycle free memory during the interval? | `allocated_between` under-reports. Prefer intervals with no concurrent activity, or take the median over many samples. |
| Is the allocation rate implausible? | Cross-check against a second source: Micrometer's `jvm.gc.memory.allocated` counter, or JMH's `-prof gc` on a representative operation. **Two independent methods agreeing is the standard.** |
| Is the promotion rate negative over a long window? | Old gen shrank, so a mixed or full collection ran. That is information, not an error. |
| Was the JVM warmed up? | The first minute after start is JIT warm-up (Topic 74) and cache filling. **Discard it**, or you are measuring startup. |

### WHAT TO LOOK FOR — reading the two numbers together

| Shape | What it means | What to do |
|---|---|---|
| High allocation rate, **flat** promotion | **Healthy.** Objects dying young, exactly as designed. | **Nothing.** Record it in the baseline and write "this is fine" next to it, explicitly, so nobody optimises it. |
| Modest allocation, **rising** promotion | **The dangerous shape.** The live set is growing. | Old-gen pressure → concurrent cycles → Full GC. The fix is retention (Trap 3, Topic 79), not flags. |
| Both high and stable | A genuinely large working set. | Size the heap from the measured live set plus headroom (Topic 82). Ask whether the working set is necessary. |
| Promotion spikes at a regular interval | Something periodic: a scheduled job, a cache refresh, a metrics scrape building a large intermediate. | Correlate on wall-clock timestamps. Topic 73 covers the periodic-safepoint variant. |
| Live-set floor **flat** over hours | A working set. | Size for it. |
| Live-set floor **climbing** over hours | **A leak.** | Heap dump, MAT dominator tree, path to GC root. Topic 79. |
| Live-set floor climbing then plateauing | A **bounded** leak — the `ThreadLocal`-on-a-pool shape. Retention grows to pool size × context size and stops. | Topic 79's second drill. Often misdiagnosed as "not a leak" because the graph flattens. |

### Why a naive `System.nanoTime()` measurement is WRONG here

You will be tempted to measure "GC cost" by timing application work:

```java
// DO NOT DO THIS. It cannot measure what you want.
long start = System.nanoTime();
for (int i = 0; i < 1_000_000; i++) {
    process(new Order());
}
System.out.println("ns/op = " + (System.nanoTime() - start) / 1_000_000);
```

Five independent reasons it lies, and you cannot tell which one is lying:

1. **Dead-code elimination.** If the result is unused, C2 can prove the work has no
   observable effect and delete it. You measure an empty loop. Topic 75.
2. **Escape analysis and scalar replacement.** The `Order` may never be allocated at all,
   so you are not measuring allocation. Topic 75 again.
3. **On-stack replacement and cold JIT.** The loop begins interpreted and is swapped to
   compiled code mid-flight. Your average blends interpreted, C1 and C2 execution in a
   ratio determined by the loop count you happened to pick. Topic 74.
4. **GC is not in the loop.** Collection happens on GC threads, asynchronously, triggered
   by eden filling. **A timer around your code measures your code, not the collector.**
   Some collection cost lands inside your measured window and some does not, in a
   proportion you cannot know.
5. **One run, one JVM.** No warm-up, no forks, no variance estimate. Topic 77.

**And the reason specific to this topic:** GC cost is an emergent property of allocation
rate, live set, object lifetime distribution, heap size, CPU quota and the collector's
control loop. **None of those are reproduced by a loop in a `main` method.** The instrument
is a realistic load against a real heap, observed through the log.

Topic 77 is the full treatment. Read it before you write any benchmark you intend to act on.

### Where JMH *is* the right tool here

For one narrow question: **the allocation cost of a specific operation**, in bytes per
operation. That is a legitimate microbenchmark and JMH's `-prof gc` answers it directly.

```java
package com.orderflow.bench;

import org.openjdk.jmh.annotations.*;
import org.openjdk.jmh.infra.Blackhole;

import java.util.List;
import java.util.concurrent.TimeUnit;

@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.MICROSECONDS)
@Warmup(iterations = 5, time = 1)
@Measurement(iterations = 10, time = 1)
@Fork(3)                                   // 3 separate JVMs: defeats profile pollution
@State(Scope.Benchmark)
public class OrderMappingAllocationBenchmark {

    @Param({"10", "100", "500"})
    public int lineCount;

    private Order order;
    private OrderMapper mapper;

    @Setup(Level.Trial)
    public void setUp() {
        order  = TestOrders.withLines(lineCount);   // built once, not measured
        mapper = new OrderMapper();
    }

    @Benchmark
    public void mapToDto(Blackhole bh) {
        List<OrderLineDto> dtos = mapper.toLineDtos(order);
        bh.consume(dtos);                            // consumed: cannot be eliminated
    }
}
```

```bash
./mvnw -Pbench clean verify
java -jar target/benchmarks.jar OrderMappingAllocationBenchmark -prof gc -rf json -rff mapping.json
```

| Annotation / flag | What it defends against |
|---|---|
| `@State(Scope.Benchmark)` | Holds inputs in fields JMH controls, so C2 cannot constant-fold them |
| `Blackhole.consume(...)` | Defeats dead-code elimination — **and note it also defeats scalar replacement**, which is exactly the subtlety Topic 75 is about |
| `@Warmup(iterations = 5)` | Reaches steady-state compiled code before recording |
| `@Fork(3)` | Three JVMs; exposes variance and stops one `@Param` polluting another's profile (Topic 74) |
| `@Param` | Separate benchmarks per size, so one call site does not see three shapes |
| `-prof gc` | **The one that matters here.** Normalised bytes allocated per operation |

**WHAT TO LOOK FOR:** the bytes-per-operation figure, and its scaling.

| What you see | What it means |
|---|---|
| Bytes/op scales linearly with `lineCount` | Expected. Multiply by your request rate to get this endpoint's contribution to the service allocation rate. |
| Bytes/op has a large constant term | Fixed per-call overhead — a pre-sized buffer, a boxed key, a defensive copy. Often the cheapest thing to remove. |
| Bytes/op is near zero | Escape analysis eliminated the allocations (Topic 75) — **or the `Blackhole` did not prevent it and the work was eliminated**. Verify with `-prof gc` on a variant that stores the result. |
| Times overlap within their confidence intervals | You measured nothing on the time axis. The allocation axis may still be informative. |

Report the confidence interval JMH prints, never the point estimate.

### The four numbers to track continuously, in production

Not during a drill — permanently, on a dashboard (Topic 118 covers exporting these without
blowing up metric cardinality):

| Number | Where from | Why |
|---|---|---|
| **Allocation rate (MB/s)** | The formula above, or Micrometer's `jvm.gc.memory.allocated` | The input to every capacity conversation |
| **Promotion rate (MB/s)** | Old-gen deltas, or `jvm.gc.memory.promoted` | Predicts old-gen pressure before it becomes a Full GC |
| **Live-set floor (MB)** | Old-gen occupancy after mixed/full collections | Flat = working set; climbing = leak. **The single best leak detector you can have on a dashboard** |
| **Full GC count** | GC log, or `jvm.gc.pause` with cause tags | Should be zero. Any nonzero value is an incident to explain |

---

## Practice exercises

### 1 — Easy: predict the collector's behaviour, then check

Using `LiveSetProbe` from Example 1, and **before running anything**, predict:

1. With `-Xmx512m` and a live set of ~1 MB, how many young collections will 2 000 000
   allocations of ~1 KB produce? Show your working: total allocated bytes ÷ eden size. You
   will need eden size — get it from `jcmd <pid> GC.heap_info` or from the `gc+heap=debug`
   lines, and note that under G1 it is adaptive.
2. If you double the total allocations, what happens to collection count? To duration?
3. If you double the retained count, what happens to each?
4. At what retained count do you expect the first `Pause Full`? Justify it against `-Xmx`
   using Topic 69's object-size arithmetic — remember each `Payload` is a record header
   plus a `long` plus a reference, **plus** a separate `byte[1024]` with its own 16-byte
   header.
5. Run all four. Write down every gap between prediction and result, and the specific
   assumption that was wrong.

Then answer: **which prediction were you most confident about and most wrong about?** That
one is the belief worth correcting.

### 2 — Medium: the audit (combines Topics 01, 12, 15, 17, 21, 25, 38, 48, 49, 65, 68, 69)

Audit `orderflow` for retention. The deliverable is a ranked list with numbers.

1. **Enumerate every GC root in the codebase.** Grep for `static` non-final collections,
   `static` mutable fields, `ThreadLocal` declarations, and registered listeners. For each:
   what bounds it? If nothing does, it is Trap 3.
2. **Find every cache.** Spring `@Cacheable` (Topic 110), `ConcurrentHashMap` fields,
   `LinkedHashMap` LRUs (Topic 15), Hibernate's second-level cache (Topic 51). For each:
   what is the maximum entry count, what is the per-entry deep size (Topic 69), and what is
   the product? **Sum the products.** That is your cache contribution to the live set.
3. **Find every `ThreadLocal`.** For each: is it removed in a `finally`? On a pooled thread
   an unremoved entry lives as long as the thread does — which is forever. Compute the
   bounded-but-permanent retention: pool size × context deep size.
4. **Estimate per-request retention.** For `GET /orders/{id}` with 500 lines: how many
   entities are hydrated, what does each cost including Hibernate's dirty-checking snapshot
   (Topic 48 — roughly double), and how long is the persistence context held? Multiply by
   peak concurrency.
5. **Find the longest-lived scope violation.** Anything request-scoped captured by a
   singleton (Topic 38), any lambda capturing something large (Topic 21), any non-static
   inner class capturing its enclosing instance.
6. **Predict the live set** from steps 2–5, then **measure it** with Formula 3 and explain
   every gap. The gaps are where your model of the service is wrong, and that is the
   valuable output.
7. Write the findings as: site, retained bytes at peak, bound (or "none"), risk, fix,
   effort. **Sort by retained bytes.** Draw a line where the saving stops being worth the
   diff, and defend the line.

### 3 — Hard: production simulation — separate the two numbers under real load

Prove you can diagnose a GC problem from evidence, and prove you can exonerate GC.

1. **Re-run the Topic 65 baseline** and confirm ±10%. Gate rule.
2. **Record allocation rate, promotion rate, live-set floor, GC CPU overhead, and per-endpoint
   percentiles.** This is your control.
3. **Have someone else** (or a script with a random seed you do not read) introduce
   **exactly one** of these four changes:
   - (a) a large increase in short-lived allocation on the catalogue path;
   - (b) an unbounded static cache that grows with traffic;
   - (c) a bounded but very large cache populated at startup;
   - (d) **no change at all**, but a 300 ms artificial delay in the payment gateway stub.
4. **Run the unchanged baseline load for ten minutes** and capture the GC log and the k6
   results.
5. **Diagnose which change was made, from the evidence alone**, before looking at the diff.
   Write down your reasoning as a decision path:
   - allocation rate up, live set flat, duration flat → (a)
   - allocation flat, live-set floor **climbing** → (b)
   - allocation flat, live-set floor **high but flat**, durations up from the first minute → (c)
   - **all GC numbers unchanged, p99 up** → (d), and the whole point is that this is a GC
     *exoneration*: you must be able to say "it is not GC" with the same confidence you say
     "it is".
6. **Check the diff.** If you were wrong, identify precisely which observation you
   over-weighted.
7. **Repeat with a different change.** Do this until you get it right twice in a row.
8. **Write the runbook**: "GC-suspected latency incident — the five commands, in order, and
   what each one rules in or out." Commit it to `/docs/java/baselines/`. That runbook is a
   real production artefact, and it is the kind of thing Phase 12's deliverables are made
   of.

**Case (d) is the most important one in the exercise.** Most engineers can eventually find
a GC problem. Very few can look at a latency spike and say, with evidence, "GC is fine,
stop looking here" — and that skill saves far more time than the other one.

---

## Interview questions

### Q1 — "What does the garbage collector do?"

**MID-LEVEL answer:** "It automatically frees memory that's no longer being used, so you
don't have to manage memory manually like in C. It runs periodically and cleans up objects
that aren't referenced anymore."

**SENIOR answer:** "Mechanically, it does the opposite of what that description suggests:
**it never looks at garbage at all.** It starts from a fixed root set — thread stacks,
static fields, JNI handles, thread-locals, held monitors, a few JVM-internal structures —
traverses every strong reference reachable from those, and marks what it reaches as live.
Everything it did not reach is free by definition. **The dead are never visited.**

That has one consequence that matters more than everything else: **cost scales with the
live set, not with the garbage.** A young collection is a copying collection — it copies
the survivors out and declares the whole space free in one step. Ten megabytes of dead
objects in eden cost literally nothing. Ten megabytes of live objects cost a traversal and
a copy, on every collection, forever.

So the two numbers that describe a service to a collector are **allocation rate** and
**live set**, and they are independent levers. Allocation rate sets how *often* collections
happen; the live set sets how *long* each one takes. Pause time is a duration problem, so
p99 is a live-set problem.

The second consequence is the definition of a leak. **'Unreachable' is a graph property;
'unused' is a human intention.** A `static Map` holding an order you will never read again
is reachable, therefore live, therefore retained forever — and nothing is broken. Java
doesn't leak, it **retains**. So the diagnostic question is never 'what leaked' but 'what
still points at it', which is why the heap-dump tooling is built around path-to-GC-root and
dominator trees rather than around allocation sites."

**What separates them:** the mid answer describes the *purpose*. The senior answer describes
the *mechanism*, derives the cost model from it, converts that into two measurable
quantities, and lands on the definition of a leak as a corollary rather than as a separate
fact. The strongest single sentence is "the dead are never visited" — it is short, it is
mechanically true, and everything else follows from it.

**Interviewer's follow-up:** *"So allocation is free?"* — Nearly. Allocation is a pointer
bump in a thread-local buffer in eden, on the order of ten instructions, with no lock and no
free-list search. It is not literally free — it costs CPU, it churns TLABs, and it drives
collection frequency — but it is *radically* cheaper than people from a malloc background
assume. **Retention is what costs.** That inversion is the single most useful thing to carry
from JVM memory management into design decisions.

---

### Q2 — "Our GC pauses got worse after we doubled the heap. Why?"

**MID-LEVEL answer:** "A bigger heap means more memory to scan, so collections take longer.
We should probably lower the pause target, or switch to a low-pause collector."

**SENIOR answer:** "The general mechanism is that **heap size doesn't change the cost of a
collection, it changes how much each collection covers.** Young gen is sized as a fraction
of heap, so doubling the heap roughly doubles eden. Collections become less frequent and
each one covers twice as much allocation and has more survivors to copy. Total GC CPU
usually goes *down* — so the mean improves — while individual pauses get *longer*, so p99
gets worse. If you were watching averages, you'd have reported a win.

There are three more specific mechanisms I'd check, in this order.

**One — did the heap cross about 32 GB?** If so, compressed oops turned off silently. Every
reference went from 4 bytes to 8 and every header from 12 to 16, so the live set grew with
no code change, and since cost tracks the live set every collection got more expensive.
`jcmd <pid> VM.flags -all | grep CompressedOops` settles it in one command, and the
fingerprint in a class histogram is identical instance counts with larger byte totals.

**Two — was the heap previously *too small*?** If the collector was thrashing — constant
young collections, Full GCs, evacuation failures — then more heap genuinely helps a lot.
Those two situations look identical on a latency graph and completely different in the log:
`grep -c 'Pause Full'` and `grep -cE 'To-space exhausted|Evacuation Failure'` tell them
apart in two seconds.

**Three — did the container limit move with it?** Heap is only part of the process
footprint. Metaspace, code cache, thread stacks, direct buffers and GC structures are all
outside `-Xmx`, so a bigger heap in an unchanged container is how you get OOM-killed with a
perfectly healthy heap.

**What I'd actually do:** compare the GC logs before and after under the same load, and look
at three things — the live-set floor (old-gen occupancy after mixed collections), the
`gc+phases=debug` breakdown to see whether Object Copy grew, and the Full GC count. If the
live set jumped without a data change, it's compressed oops. If Object Copy grew
proportionally with eden, it's the general mechanism and the fix is to size the heap from the
measured live set rather than from what the container allows.

**And before any of that I'd separate GC pause from stop-the-world duration** with
`-Xlog:safepoint*`. The duration in a pause line is GC work at the safepoint; the time to
*reach* the safepoint is a separate number with a completely different cause, and I've seen
teams tune GC for weeks on a problem that was a counted loop."

**What separates them:** the mid answer has one model and reaches for a flag. The senior
answer explains the *general* mechanism correctly, holds three competing specific hypotheses,
names the exact command that discriminates each, knows the compressed-oops cliff exists, and
refuses to attribute a pause to GC before separating GC time from time-to-safepoint.

**Interviewer's follow-up:** *"How would you size the heap properly, then?"* — From the
measured live set, not from the container limit. Measure the live-set floor under
representative load — old-gen occupancy after mixed collections over a long run — then size
the heap at roughly two to three times that, so the collector has headroom to evacuate into
and eden has room to absorb the allocation rate. Then validate under load and check the
*total* process footprint against the container limit, not just the heap. And set `-Xms` equal
to `-Xmx` in a container so you never pay resizing pauses.

---

### Q3 — "How would you find a memory leak in a Java service?"

**MID-LEVEL answer:** "I'd look at the heap usage graph, and if it keeps growing I'd take a
heap dump and look at which classes are using the most memory. Probably there's a collection
that keeps growing."

**SENIOR answer:** "First I'd confirm it *is* a leak, because three different things produce
a rising memory graph and only one of them is a leak.

**The distinguishing measurement is the live-set floor**: old-generation occupancy
immediately after mixed or full collections, over hours. Heap-used sawtooths — that graph
tells you nothing. The floor is the signal.

- **Floor flat over hours** → not a leak. It's a working set, and the fix is heap sizing or
  reducing retention per request.
- **Floor climbing steadily over hours** → a leak.
- **Floor climbing then plateauing** → a *bounded* leak, which is the sneakiest shape. The
  classic is a `ThreadLocal` never removed on a fixed thread pool: retention grows to pool
  size × context size and then stops. It gets dismissed as 'not a leak' because the graph
  flattens, and it's permanently holding memory it shouldn't.

Once I know it's a leak: heap dump, ideally at peak and ideally with
`-XX:+HeapDumpOnOutOfMemoryError` already configured so I get one automatically. Then Eclipse
MAT, and specifically the **dominator tree** rather than the class histogram. A histogram
tells me there are twelve million `String`s, which I already knew and can't act on. The
dominator tree answers 'if this object were freed, how much would go with it', which is the
question I actually have. Then **path to GC root** on the largest dominator — that names the
root category, and the root category names the bug.

The four shapes I'd expect, in rough order of frequency: an unbounded static collection or a
cache with no eviction; a listener or callback registered and never removed; a `ThreadLocal`
on a pooled thread; and a non-static inner class or lambda capturing an enclosing instance
that outlives what it should.

**And I'd want two heap dumps, twenty minutes apart, and diff them.** One dump tells me
what's big. Two dumps tell me what's *growing*, which is a much sharper question — a large
constant structure looks identical to a leak in a single dump."

**What separates them:** the mid answer conflates 'memory is high' with 'memory is leaking'.
The senior answer **names the measurement that distinguishes three different causes**, knows
why the dominator tree beats a histogram, knows the bounded-leak shape that gets dismissed,
and asks for two dumps rather than one.

**Interviewer's follow-up:** *"The dump is 8 GB and MAT won't open it on your laptop."* —
Run MAT's headless parser on a machine with the memory, which produces a much smaller index
plus the leak-suspect report I can open locally. Or take the dump with
`jcmd GC.heap_dump` which writes live objects only. Or, if I can't get a dump at all, fall
back to `jcmd GC.class_histogram` sampled repeatedly and diffed — much weaker evidence, but
it shows growth by class and is usually enough to narrow the search. And I'd fix the process
problem too: if we can't get a dump from production, that's a gap in our tooling that will
cost us again.

---

### Q4 — "Should we pool objects to reduce GC pressure?"

**MID-LEVEL answer:** "It depends — pooling avoids allocation, so it can help if you're
creating a lot of objects. But it adds complexity, so I'd only do it for expensive objects."

**SENIOR answer:** "For plain Java objects, almost certainly not, and the reason is the
opposite of the intuition.

**Allocation in Java is a pointer bump in a thread-local buffer** — around ten instructions,
no lock, no free-list search. **Reclaiming an object that dies in eden costs zero**, because
a young collection copies out the survivors and declares the whole space free; the dead are
never visited.

So pooling takes an object with the cheapest possible lifecycle and gives it the most
expensive one. A pooled object survives by construction: it gets traced on every collection,
copied through survivor spaces, promoted to old gen, and from then on it's part of the live
set that every concurrent cycle marks and every mixed collection may relocate. **You've
converted free garbage into permanent live data**, and since cost tracks the live set, you've
made every collection more expensive — forever.

There's a second cost people miss: handing objects in and out of a pool means storing
references into long-lived objects, which **dirties cards and drives up remembered-set work
on every young collection.** So you pay on both axes.

And there's a correctness cost that's worse than either. A pooled object returned while it's
still in use produces cross-request data leakage — fields from one user's response appearing
in another's. Intermittent, load-dependent, and a security incident rather than a performance
bug.

**The test I'd apply: is the expensive part of this object's creation something other than
the memory allocation?** If yes, pool it — database connections, because setup is a TCP
handshake and authentication. Threads, because they're an OS resource. Off-heap buffers,
because native memory really does have a malloc-shaped cost and a `DirectByteBuffer`'s
lifecycle is tied to a `Cleaner`. Very large arrays that would be humongous allocations under
G1. If no — and for a DTO or a value object the answer is no — don't pool it.

If allocation genuinely is costing us CPU, the fix is to **allocate less**: project fewer
columns, page the endpoint, stream the response instead of buffering it. Not to reuse more."

**What separates them:** the mid answer treats pooling as a normal tool with a cost/benefit.
The senior answer knows the JVM's allocation path is a pointer bump, derives that pooling
*inverts* the cost model, adds the write-barrier cost and the correctness hazard, and gives a
crisp decision rule that correctly admits connections and threads while excluding DTOs.

**Interviewer's follow-up:** *"We measured, and the pool did reduce allocation rate."* — I'd
believe that and say it's the wrong metric. Allocation rate buys collection *frequency*; the
live set buys collection *duration*. If our problem was pause time, reducing allocation rate
was never going to fix it — and the pool increased the live set, which is the term that
drives duration. I'd want the before/after on **pause durations and old-gen occupancy**, not
on allocation rate, and I'd expect those to have got worse.

---

### Q5 — "How do you decide between G1, ZGC, and Parallel GC?"

**MID-LEVEL answer:** "ZGC has much lower pause times, so for a latency-sensitive service
you'd use ZGC. G1 is the default and works for most things. Parallel is older."

**SENIOR answer:** "The choice follows from a **latency budget** and two measurements, not
from a ranking of collectors.

**The measurements are allocation rate and live set.** They tell me how much work the
collector has to do and how much headroom it needs.

**The latency budget is the part people skip.** If `orderflow`'s p99 is 180 ms and 150 ms of
that is a Postgres query plus a payment-gateway call, then moving from G1's 40 ms pauses to
ZGC's sub-millisecond pauses buys me at most a few percent — and costs me throughput via load
barriers, and costs me footprint via the headroom ZGC needs to relocate concurrently. On a
2 vCPU container that throughput cost is not hypothetical; concurrent GC threads compete with
request-handling threads for the same two cores. **Low pause time is not the same as low
latency, and if the bottleneck is elsewhere, changing collectors is a regression.**

So my decision path:

**Parallel GC** if it's a batch job and I care about total throughput and wall-clock
completion. Longest pauses, best throughput, smallest overhead. Nobody is watching p99 on a
nightly report.

**G1** as the default for a service. It targets a pause goal at a modest throughput cost, it
handles a wide range of heap sizes, and it's what everything is tuned and documented against.
I'd stay here unless I have a measured reason to move.

**ZGC or Shenandoah** when I have a *measured* pause problem that dominates my latency
budget — typically a large heap where G1's pauses scale with the live set and I need them not
to. Generational ZGC is the version to use, because most objects still die young and the
non-generational version gave that up.

**Serial GC** is a real answer for a small container, and it's also what the JVM may silently
*choose* for me if the container makes it see fewer than two CPUs — which is Topic 82's trap
and the first thing I'd check before trusting any measurement.

**And whatever I pick, I'd A/B it against the recorded load baseline** — same k6 profile, same
dataset, ten minutes minimum, three runs per arm, comparing p99 and p999 and throughput, not
means. I'd also specifically look for the case where the low-pause collector is *worse*,
because on a CPU-constrained container that happens and it's the finding people don't expect."

**What separates them:** the mid answer ranks collectors. The senior answer **derives the
choice from a latency budget and two measurements**, knows that pause time and latency are
different quantities, knows the throughput and footprint costs of concurrent collection, knows
the container can override the choice silently, and insists on an A/B against a recorded
baseline including the possibility of a negative result.

**Interviewer's follow-up:** *"You switched to ZGC and p99 got worse. Explain."* — Most likely
CPU. ZGC does its work concurrently, on GC threads, competing with application threads. On a
2 vCPU container that's a large fraction of the machine, and the load barriers cost throughput
on every reference load. So request service time rises even though pauses fell. I'd check GC
thread CPU, check whether `ActiveProcessorCount` matches reality, and consider whether the
honest answer is that this service's pauses were never the problem. The second possibility is
insufficient headroom — ZGC needs room to relocate concurrently, and if allocation outpaces
relocation you get allocation stalls, which look like latency spikes but are a different
mechanism.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. A young collection copies survivors out and declares the whole space free. **Derive, from
   that one fact alone**, why doubling short-lived allocation costs almost nothing while
   doubling the live set costs on every collection — and then state which of those two
   changes moves p99.

2. Object pooling reduces allocation rate and increases the live set. Using the cost model
   from question 1, predict what pooling does to young-collection *frequency*, young-collection
   *duration*, promotion rate, and p99. Then name the one category of object for which
   pooling is still correct, and say what makes it different.

3. "Unreachable is not unused." Construct an `orderflow` example where every line of code is
   correct, every test passes, and the service runs out of memory in three weeks. Then name
   the single code-review question that would have caught it.

4. Old-generation objects that point at young-generation objects must be found on every young
   collection. Explain why, explain what mechanism finds them, and explain what your
   application pays for that mechanism **even when no collection ever happens**. Then design a
   data structure change that reduces the cost, and say what it trades away.

5. Two services have identical allocation rates and identical heaps. One has young pauses ten
   times longer than the other. Name three things that could differ, in order of likelihood,
   and the single log line that distinguishes them.

6. `heap_used_after_collection` is not the live set. Name three things included in it that are
   not live data, and describe the procedure that gives you a defensible live-set number
   anyway. Then explain why that procedure needs a *long* run rather than a careful one.

7. Your p99 has 2-second spikes. Your GC log shows no pause over 20 ms and your GC pause
   overhead is under 1%. **List the five things you would investigate next, in order**, and
   for each say what single command or measurement would rule it in or out.

---

## Quick reference card

### The model

```
live      = reachable from a GC root by a chain of strong references
garbage   = everything else -- and the collector NEVER VISITS IT
leak      = a live root chain to an object you will never use again

allocation_rate (MB/s)  -> sets collection FREQUENCY
live_set        (MB)    -> sets collection DURATION  -> this is where p99 lives

young_collection_frequency  ~=  eden_size / allocation_rate
young_collection_duration   ~=  k x surviving_bytes
gc_pause_overhead_%         =   sum(pause_durations) / wall_clock x 100
heap_needed                 ~=  live_set x (2 to 3) + eden headroom
```

### The formulas — apply them to YOUR log

```
// Between two CONSECUTIVE YOUNG collections n and n+1:
allocated_between  =  heap_used_BEFORE[n+1] - heap_used_AFTER[n]
elapsed            =  uptime_at[n+1] - uptime_at[n]
allocation_rate    =  allocated_between / elapsed                 // MB/s

// Old-gen occupancy from the gc+heap=debug lines, NOT the summary line:
promoted_between   =  old_used_AFTER[n+1] - old_used_AFTER[n]
promotion_rate     =  promoted_between / elapsed                  // MB/s

// The live set has no single line. Take the FLOOR:
live_set  ~=  floor of old-gen occupancy after MIXED or FULL collections,
              sampled over a LONG run at steady state
```

Take the **median over many intervals**, report the spread, and run all four sanity checks.

### The GC roots

```
thread stacks (locals + operand stacks, via JIT oop maps at safepoints)
static fields of every loaded class
Thread objects and their ThreadLocal maps
JNI local and global references
monitors currently held
class loaders and the classes they loaded
the interned string table
JVMTI / agent references
Reference objects pending processing
```

### Diagnostic commands

```bash
# What collector, what heap, right now?
jcmd <pid> VM.flags -all | grep -E "UseG1GC|UseZGC|UseSerialGC|MaxHeapSize|InitialHeapSize"
java -XX:+PrintFlagsFinal -version | grep -E "UseG1GC|UseZGC|MaxRAMPercentage"

# The log. Decorators matter as much as tags -- `uptime` is the denominator of every rate.
-Xlog:gc*,gc+heap=debug,gc+phases=debug:file=gc.log:time,uptime,level,tags:filecount=5,filesize=50m

# Learn YOUR JDK's format BEFORE writing any grep.
head -40 gc.log
grep 'Pause Young' gc.log | head -3
grep 'gc,heap'     gc.log | head -6

# Triage a log you were handed, in this order.
grep -c 'Pause Full'                              gc.log   # must be zero
grep -cE 'To-space exhausted|Evacuation Failure'  gc.log   # nonzero => no copy space
grep -c 'Pause Young'                             gc.log   # very high => thrashing
grep    'System.gc()'                             gc.log   # find the caller
grep    'GC(1234)'                                gc.log   # every line of one collection

# Live objects, right now. NOTE: triggers a full collection.
jcmd <pid> GC.class_histogram | head -30
jcmd <pid> GC.heap_info

# Force a collection to read the floor. Load-test instances only.
jcmd <pid> GC.run

# Always on, in production.
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/dumps

# Turn logging on at runtime, no restart.
jcmd <pid> VM.log output=/tmp/gc-live.log what='gc*,gc+heap=debug' decorators='time,uptime,level,tags'
jcmd <pid> VM.log disable

# Separate GC pause from stop-the-world duration. Do this BEFORE blaming GC.
-Xlog:safepoint*:file=safepoint.log:time,uptime,level,tags
```

> **Version note, one line:** G1 remains the default on server-class machines on both JDK 21
> and JDK 25, and generational ZGC became the ZGC default in a recent release — but log
> formats, tag names and defaults have moved across versions. **Settle every default on your
> own runtime with `jcmd <pid> VM.flags -all` or `java -XX:+PrintFlagsFinal -version`, and
> learn your log's field names with `head` before writing a grep. Do not trust any document,
> including this one.**

### Reading the two numbers together

| Allocation | Promotion / live set | Verdict |
|---|---|---|
| High | Flat | **Healthy.** Objects dying young. Do nothing; write "this is fine" in the baseline |
| Modest | Rising | **Dangerous.** Live set growing → old-gen pressure → Full GC. Fix retention, not flags |
| High | High and stable | Genuinely large working set. Size the heap from it; question whether it's necessary |
| Any | Floor climbing over hours | **Leak.** Heap dump, dominator tree, path to root (Topic 79) |
| Any | Floor climbs then plateaus | **Bounded leak** — the `ThreadLocal`-on-a-pool shape. Often wrongly dismissed |

### Gotchas checklist

- [ ] Print the collector. Never infer it. Containers change the default (Topic 82).
- [ ] `uptime` in the decorators, or you have no denominator for any rate.
- [ ] Learn your JDK's log field names before writing a grep. They move between versions.
- [ ] Cost tracks the **live set**, not the garbage. The dead are never visited.
- [ ] Allocation rate = frequency. Live set = duration. **p99 is a duration problem.**
- [ ] `heap_used_after_GC` ≠ live set. Floating garbage, TLAB waste, fragmentation.
- [ ] Take the **floor** over a long run, and only after mixed/full collections.
- [ ] Median across many intervals, never a single pair. Report the spread.
- [ ] Discard the first minute — that is JIT warm-up (Topic 74), not steady state.
- [ ] Don't mix young with mixed/full collections in one subtraction.
- [ ] Pooling plain objects converts free garbage into permanent live data.
- [ ] Any `static` collection that only grows is a leak. Ask "what evicts this?" in review.
- [ ] `SoftReference` caches hold the live set at maximum. Use a bounded cache.
- [ ] More heap does not make a collection cheaper — it makes them rarer and longer.
- [ ] GC pause ≠ stop-the-world. Check `-Xlog:safepoint*` before blaming GC (Topic 73).
- [ ] Zero Full GCs is the target. Every one is an incident to explain.

---

## When would I use this at work?

**1. A latency incident where someone has already declared it a GC problem.**
You pull the GC log, compute pause overhead in one command, and find it is under 1%. That
single number **exonerates GC in about two minutes** and redirects the whole investigation to
safepoints, the connection pool, the downstream, or CPU throttling. The ability to say "it is
not GC, here is the evidence, stop looking here" is worth more hours than any tuning skill,
because the default failure mode of a Java latency incident is three engineers tuning flags
for a day on a problem that was never GC.

**2. Reviewing a PR that adds a cache, a registry, or a `static` collection.**
One question: **what bounds this?** If the answer is "nothing", you have found the most common
production memory bug in Java before it shipped. If the answer is "a size limit", the follow-up
is "what is the per-entry deep size and what does that do to the live-set floor" — which is
Topic 69's measurement feeding this topic's cost model. This is a two-comment review that
prevents an OOM three weeks after the deploy, when nobody will connect it to this change.

**3. Capacity planning and container sizing.**
Someone asks how much memory the pods need, or finance asks to halve it. You answer with a
measured live set, measured allocation rate, and the headroom the collector needs — and you
can state what happens at each candidate size, including which one starts producing Full GCs.
That is a capacity model, not an opinion, and it is exactly the artefact Phase 12 asks you to
produce. It also makes the reverse case: when someone wants to double the heap "to be safe",
you can show that it will make p99 worse and the container footprint riskier.

---

## Connected topics

**Prerequisites:**

- **01 — Boxing and the Integer cache**: every boxed value is an allocation, and boxed
  collections are a live-set problem as well as an allocation-rate one. Topic 69 gives the
  bytes; this topic gives the consequence.
- **12 — HashMap internals**: `Node` objects are part of your live set, and a map that grows
  without eviction is Trap 3's shape. The table itself is a large array that survives.
- **15 — LinkedHashMap and LRU**: the simplest bounded cache, and bounding is what makes the
  live set a number you chose.
- **17 — Immutability and safe publication**: immutable objects are often short-lived and die
  in eden — the cheapest possible lifecycle.
- **21 — Lambdas and `invokedynamic`**: a capturing lambda holds its captures alive. A
  non-capturing one is a singleton. That distinction is a retention question.
- **25 — Parallel streams**: parallel work multiplies allocation across threads, and the
  common pool's threads are GC roots with stacks of their own.
- **38 — Bean scopes**: a request-scoped object captured by a singleton is promoted to the
  singleton's lifetime. That is a scope bug and a retention bug simultaneously.
- **48 — Hibernate persistence context**: the snapshot for dirty checking roughly **doubles**
  the memory per loaded entity, and the context lives as long as the transaction. This is
  `orderflow`'s single largest per-request retention term.
- **49 — Lazy associations and open-session-in-view**: OSIV extends the persistence context to
  the whole request, multiplying in-flight retention by request duration.
- **65 — The load-testing gate**: the recorded baseline is the control for every measurement
  here. Without it, "the live set grew" is an observation without a comparison.
- **66 — JVM architecture**: what is per-thread (stacks — roots) and what is shared (the heap
  — what gets collected).
- **68 — Heap generations and TLABs**: allocation is a pointer bump; promotion is the
  expensive event. This topic supplies the *why*: the collector's cost model.
- **69 — Object layout and compressed oops**: converts an object *count* into an allocation
  *rate* in MB/s and a live set in MB. Crossing the compressed-oops threshold is a live-set
  event with no code change.

**This unlocks:**

- **71 — G1 in depth**: G1 meets a pause goal by shrinking the collection set, and the pause
  cost it is managing is exactly the live-set copying cost this topic defines. Card tables
  generalise into per-region remembered sets.
- **72 — ZGC and Shenandoah**: making pause time independent of live-set size is precisely
  what these collectors buy, and what they charge for it in throughput and footprint.
- **73 — Safepoints and TTSP**: root scanning needs the world stopped at a safepoint with valid
  oop maps. And a pause the GC log does not account for is not a GC pause.
- **74 — JIT and tiered compilation**: the JIT emits the oop maps that make root scanning
  possible, and the write barriers that maintain the card table.
- **75 — Escape analysis**: a scalar-replaced object is never allocated at all — the cheapest
  possible reduction in allocation rate, and the only one that is genuinely free.
- **77 — JMH**: the only correct way to measure the allocation cost of a specific operation,
  and the reason the timing loop above is fiction.
- **78 — Profiling**: async-profiler's allocation mode names the *sites*; this topic gives you
  the *rate*. You need both to prioritise.
- **79 — Memory leaks and heap dumps**: the direct continuation. Reachability is the concept;
  the dominator tree and path-to-GC-root are the tools.
- **80 — Off-heap memory**: moving data out of the collector's graph removes it from the live
  set entirely — and creates a footprint problem the heap graph will not show you.
- **82 — JVM tuning and containers**: heap sizing from the measured live set, GC thread counts
  from the CPU quota, and the collector the JVM silently picks when the container lies about
  the machine.
- **83 — GraalVM native image**: a build-time heap and a different collector; the contrast
  sharpens what HotSpot's model is for.
- **85 — `synchronized` and monitors**: held monitors are GC roots, and the mark word that
  holds lock state is the same word that holds the GC age.
- **90 — Executors and pool sizing**: every thread is a root with a stack, and a pooled thread
  outliving a request is the `ThreadLocal` leak shape.
- **96 — False sharing**: GC worker threads contend for cache lines like any other threads.
- **101 — Virtual threads**: millions of stacks change root-scanning cost materially, and a
  virtual thread's stack is itself a heap object in your live set.

---

*Java baseline 21, running on JDK 25. Three things in this document are deliberately hedged
rather than asserted: your JDK's exact GC log field names and tag spellings, the current
default collector inside your specific container, and the precise card size and
remembered-set implementation on your build. Each has a command in the Hands-on section that
settles it in under a minute. **No allocation rate, live-set size, pause duration or
percentile in this document was measured — the formulas are given precisely so that every
number you act on comes from your own log against your own baseline.** The one log-line
layout shown is a labelled illustration of the format with placeholder values, not captured
output. What has been true since generational collection was invented and will still be true
at 2am: the collector traces from roots, never visits the dead, and charges you for what
lives.*
