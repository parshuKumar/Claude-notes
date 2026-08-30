# 68 — Memory Areas: Generations, Eden/Survivor, TLABs, Promotion, Metaspace

## Phase: 8 — JVM Internals
## Category: DIFFERENTIATOR
## Java baseline: 21  |  Notes features from: 21 / 25
## Project spine: this is how you answer "where does `orderflow`'s memory actually go, and which part of it is expensive" — with an allocation rate and a promotion rate derived from your own GC log, against the Topic 65 baseline in `/docs/java/baselines/`.

---

## Mechanical statement

Read this three times. The rest of the document is an elaboration of it.

> **Allocation is a pointer bump inside a thread-local allocation buffer.** Each
> application thread owns a private slice of **eden** — a Thread-Local Allocation Buffer,
> or TLAB. To allocate, the thread compares its current position against the end of its
> slice, moves the position forward by the object's size, and writes the header. No lock.
> No compare-and-swap. No free-list search. Roughly ten instructions.
>
> **The young generation is eden plus two survivor spaces.** A young collection copies
> the *live* objects out of eden and the in-use survivor space into the other survivor
> space, and then declares everything it copied out of to be free. Dead objects cost
> nothing. They are not visited, not swept, not touched.
>
> **Objects that survive enough copies are promoted to the old generation.** Each object
> carries an age in its header, incremented on each copy. When the age reaches the
> tenuring threshold — or when the survivor space cannot hold everything — the object is
> copied into old gen instead. Old gen is collected rarely, expensively, and by a
> different algorithm.
>
> **Class metadata lives in native metaspace, outside the heap.** It is not bounded by
> `-Xmx`, not collected by young collections, and not visible in a heap graph. It has its
> own `OutOfMemoryError`, its own flag, and its own `jcmd` subcommand.

Four consequences follow directly, and you should be able to derive each one:

1. **"Object allocation is expensive" is false.** Allocation is a pointer bump.
   **Object *retention* is expensive**, because survival is what triggers copying.
2. Therefore **pooling an object that would have died in eden makes it more expensive**,
   not less: you converted a free death into a copy, then into a promotion, then into
   old-gen pressure.
3. **Doubling the rate at which you create short-lived garbage costs almost nothing.
   Doubling your live set costs on every single collection.**
4. **Heap is not the process footprint.** `-Xmx` bounds one of at least six regions the
   JVM commits. Sizing `-Xmx` from your container's memory limit is how you get
   OOM-killed with a healthy heap.

---

## The bridge from what you know

### What transfers — and it is more than you expect

V8 is generational too, and the shape of what it does is genuinely similar. Hold on to
all of this, because it is correct:

- **V8 has a young generation** ("new space") collected by a copying scavenger, and an
  **old generation** ("old space") collected by a mark-compact collector that runs far
  less often and takes far longer. Same two-tier structure, same reason: most objects die
  young.
- **V8 promotes after surviving a small number of scavenges.** So the concept "an object
  that lives long enough graduates to the expensive space" is already in your head.
- **V8 allocates by bumping a pointer** in a linear allocation buffer, not by searching a
  free list. The "allocation is cheap in a managed runtime" intuition is right and it
  transfers directly.
- **The 200-millisecond Node pause you have seen** is an old-space collection. You have
  already experienced the cost asymmetry between the two generations; you just could not
  see it.

That is a genuine **HONEST ANALOGUE** for the generational model. Do not unlearn it.

### `--max-old-space-size` ≈ `-Xmx` — PARTIAL, and the gap is the whole topic

You know one memory knob:

```bash
node --max-old-space-size=1024 server.js
```

One number. One meaning. Node gives you almost nothing else, and it gives you almost
nothing to look at either.

The JVM gives you, on the same axis:

| Node | JVM | What is new |
|---|---|---|
| `--max-old-space-size` | `-Xmx` / `-XX:MaxRAMPercentage` | Roughly the same idea, and the closest thing to a direct translation |
| — | `-Xms` | Initial heap. Setting it equal to `-Xmx` removes resizing pauses. |
| — | `-Xmn`, `-XX:NewRatio` | Explicit young-generation sizing |
| — | `-XX:SurvivorRatio` | The eden-to-survivor split |
| — | `-XX:MaxTenuringThreshold` | How many copies before promotion |
| — | `-XX:+UseTLAB`, `-XX:TLABSize` | Per-thread allocation buffers |
| — | `-XX:MaxMetaspaceSize` | A **separate, native** region for class metadata |
| — | `-XX:ReservedCodeCacheSize` | Another separate region, for JIT output |
| — | `-XX:MaxDirectMemorySize` | Another, for off-heap buffers |
| — | `-Xss` × thread count | Another, and it scales with concurrency |
| — | `-XX:+UseG1GC` / `UseZGC` / `UseParallelGC` | **A choice of collector**, each with different sizing behaviour |
| — | `-Xlog:gc*` | A structured, always-available log of everything above |

**Verdict: PARTIAL.** Transfer the intuition that one number caps the main object store.
Then unlearn two things:

**First: heap is not the footprint.** In Node, `--max-old-space-size` plus a modest fixed
overhead is roughly your process size, and you can size a container by adding a bit. In
the JVM that arithmetic is wrong and will get you OOM-killed. The process commits, at
minimum:

```
RSS  ≈  heap (-Xmx)
     +  metaspace + compressed class space   (class metadata — Topic 67 made these)
     +  code cache                            (JIT output — Topic 74)
     +  thread stacks  (-Xss  ×  thread count)
     +  direct byte buffers / FFM arenas      (Topic 80)
     +  GC-internal structures                (remembered sets, card tables, mark bitmaps)
     +  the JVM's own native allocations and malloc arenas
```

Every one of those is real, and several of them scale with things you change for other
reasons — thread count, class count, concurrency. Topic 80 is the full treatment and the
tool (`-XX:NativeMemoryTracking`). For now, carry the sentence: **`-Xmx` is a budget line
item, not the budget.**

**Second: you now have visibility, and therefore responsibility.** You have never been
able to ask V8 "what was my allocation rate over the last minute" or "how many bytes
survived that collection". You can ask the JVM both, from a log it writes for free, and
the Measurement section gives you the formula. A Node engineer cannot mis-tune the GC. You
can — and Topic 71's trap section is a catalogue of how.

### NO ANALOGUE — the three genuinely new things

**1. Thread-local allocation has no analogue.** V8's bump pointer works because a
JavaScript isolate has one mutator thread. The heap is not shared, so there is nothing to
contend on. The JVM's heap is shared by every thread in the process, so a naive bump
pointer would need an atomic operation per allocation. TLABs exist *specifically* to make
allocation lock-free on a shared heap, and the consequences — TLAB sizing, refill, waste,
and why a thread with a low allocation rate still holds a chunk of eden — are entirely new
territory.

**2. Metaspace has no analogue.** V8 has no separately-sized, separately-collected,
separately-failing region for "the things classes are made of". Java does, it lives in
*native* memory, and it fails with an error most Java developers respond to by raising
`-Xmx`, which cannot possibly help.

**3. An explicit, tunable tenuring threshold has no analogue.** You cannot tell V8 "let
objects survive twelve scavenges before promoting them". You can tell the JVM exactly
that, watch an age distribution table in the log, and get it wrong.

| You know | Java | Verdict |
|---|---|---|
| New space / old space, copying scavenger | Young gen (eden + 2 survivors) / old gen | **HONEST ANALOGUE** |
| "Objects die young" | The weak generational hypothesis | **HONEST ANALOGUE** |
| Bump-pointer allocation in a linear buffer | Bump-pointer allocation in a TLAB | **PARTIAL** — same instruction sequence, but the *thread-local* part is new |
| `--max-old-space-size` | `-Xmx` | **PARTIAL** — and heap is not the footprint |
| One number, one knob | Generations, survivor ratios, tenuring, TLABs, collector choice | **NO ANALOGUE** |
| — | Metaspace (native, separately bounded, separately fatal) | **NO ANALOGUE** |
| — | An age table you can read in the log | **NO ANALOGUE** |
| `process.memoryUsage()` returns four opaque numbers | `-Xlog:gc*`, `jcmd GC.heap_info`, `VM.metaspace`, NMT | **NO ANALOGUE** — you have been promoted from passenger to driver |

---

## What is this?

The Java heap, under a generational collector, is divided by **object age**.

```
                                  Java heap  (bounded by -Xmx)
  +---------------------------------------------+-----------------------------------+
  |            YOUNG GENERATION                 |         OLD GENERATION            |
  |  +---------------------+--------+--------+  |                                   |
  |  |        Eden         |   S0   |   S1   |  |  promoted objects live here       |
  |  |  (new objects)      | surv.  | surv.  |  |  collected rarely, expensively    |
  |  +---------------------+--------+--------+  |                                   |
  +---------------------------------------------+-----------------------------------+

  Outside the heap, in NATIVE memory, not bounded by -Xmx:
  +----------------+  +------------------------+  +-------------+  +---------------+
  |   Metaspace    |  | Compressed class space |  | Code cache  |  | Thread stacks |
  | class metadata |  |  Klass pointers        |  | JIT output  |  | -Xss each     |
  +----------------+  +------------------------+  +-------------+  +---------------+
```

*Illustration of the structure. G1 implements this with regions rather than contiguous
spaces — see the note below — so treat the picture as a model of roles, not of addresses.*

### The life of an object, precisely

1. **`new Order()` executes.** The thread checks whether its TLAB has room. If yes: bump
   the pointer, write the header (Topic 69), zero the fields, done. Around ten
   instructions, no synchronisation.
2. If the TLAB has no room, one of two things happens: the thread **retires** the TLAB and
   requests a new one from eden (an atomic operation, but amortised over thousands of
   allocations), or — for an object too large to be worth a new TLAB — it allocates
   **directly in eden** with an atomic bump of the shared top pointer.
3. **Eden fills.** A young collection is triggered. This is stop-the-world.
4. The collector finds every **live** object in eden and the in-use survivor space by
   tracing from GC roots (Topic 70), and **copies** them into the other survivor space.
   Their age is incremented.
5. **Everything not copied is garbage, and is never touched.** The spaces are simply
   declared free. This is why a young collection's cost tracks the *live* set, not the
   amount of garbage.
6. Objects whose age reaches the **tenuring threshold**, or that do not fit in the target
   survivor space, are **promoted** to old gen instead of copied to a survivor.
7. Old gen fills over time. It is collected by a different, more expensive algorithm — a
   concurrent cycle plus mixed collections under G1 (Topic 71), or a full compacting
   collection under Serial/Parallel.

The whole design rests on the **weak generational hypothesis**: most objects die very
young. For `orderflow` this is overwhelmingly true — a request allocates DTOs, Hibernate
snapshots (Topic 48), `String`s, boxed `Long`s (Topic 01) and framework plumbing, and
almost all of it is unreachable by the time the response is written.

### A note on G1, so the vocabulary stays honest

G1 — your likely default — does not lay the heap out as three contiguous blocks. It uses
**equal-sized regions**, each *labelled* Eden, Survivor, Old, Humongous or Free, with
labels reassigned on every collection (Topic 71). So "eden" in G1 means "the set of
regions currently labelled Eden", and its size changes continuously as G1 pursues its
pause-time goal.

Everything in this document about TLABs, ages, survival, promotion and metaspace applies
identically under G1. What does **not** apply cleanly is the fixed-ratio sizing
vocabulary — `-Xmn`, `-XX:NewRatio`, `-XX:SurvivorRatio` — which comes from the
Serial/Parallel world. Under G1, young sizing is adaptive and driven by the pause goal.
Setting those flags either does nothing useful or disables the adaptation. That is Trap 3.

Do not take my word for which flags your collector honours:

```bash
java -XX:+UseG1GC -XX:+PrintFlagsFinal -version | grep -E 'NewRatio|SurvivorRatio|NewSize|MaxTenuringThreshold|UseTLAB'
jcmd <pid> VM.flags -all
```

---

## Why does it matter?

**1. Because it inverts the optimisation instinct you brought with you.**

Every performance article you have read in JavaScript says "avoid allocating in hot
loops". In the JVM that advice is half right and dangerously incomplete. Allocation is a
pointer bump; ten million short-lived allocations per second is a workload the JVM was
designed for. What costs is **survival**. An engineer who "optimises" by pooling objects,
caching aggressively, or reusing buffers is frequently *converting cheap deaths into
expensive survivals* — and the measurement they use to justify it (a flat allocation-rate
number) does not show the harm.

This is the single most valuable inversion in Phase 8, and it is why the mechanical
statement puts it first.

**2. Because promotion rate, not allocation rate, predicts your Full GCs.**

You can have a very high allocation rate and a perfectly healthy service, as long as the
objects die in eden. You can have a modest allocation rate and be in serious trouble, if a
steady trickle survives into old gen. The GC log gives you both numbers and the formulas
are in the Measurement section. Getting into the habit of quoting *two* numbers instead of
one is what separates a GC conversation that goes somewhere from one that goes in circles.

**3. Because metaspace failures are misdiagnosed by almost everybody.**

`java.lang.OutOfMemoryError: Metaspace` means class metadata ran out of *native* memory.
The reflex — raise `-Xmx` — is not merely useless; it makes the process footprint bigger
and brings the container OOM kill closer. You need to know that metaspace is a different
region with a different flag before you meet it, because you will meet it at 3am.

**4. Because on `orderflow` you have a baseline, so every claim here is falsifiable.**

You have p50/p95/p99/p999, throughput and error rate committed in `/docs/java/baselines/`,
and a load generator that reproduces them. That means "we reduced allocation and it got
faster" is a testable statement rather than a story. The drill in this document produces
two workloads with **identical allocation rates** and different lifetimes, so the only
variable is survival.

---

## Machine-level reality

### TLABs — sizing, refill, and waste

A TLAB is a contiguous chunk of eden handed to one thread. While a thread allocates inside
its TLAB, allocation is:

```
  // Conceptually, what compiled allocation code does in the fast path:
  //   1. load  this thread's current TLAB top pointer
  //   2. add   the object's aligned size            <- the "pointer bump"
  //   3. cmp   against the TLAB end pointer
  //   4. jump  to the slow path if it would overflow
  //   5. store the new top pointer
  //   6. write the mark word and class word         <- Topic 69
  //   7. zero the fields
```

*Illustration of the instruction sequence, not disassembly of a real compilation.* The
important property is what is **absent**: no lock, no CAS, no free-list traversal, no
size-class lookup. That is why the "allocation is expensive" instinct from
manual-memory-management languages does not apply here.

**Sizing.** TLAB size is adaptive by default. Roughly, the JVM divides eden among the
threads that are actually allocating, weighted by each thread's observed allocation rate,
and resizes at each collection. `-XX:+ResizeTLAB` (on by default) controls the adaptation;
`-XX:TLABSize` forces a fixed size (`0` means adaptive). You almost never set these.

**The consequences of adaptive sizing that do bite you:**

- A thread that allocates *rarely* still gets a TLAB. With very many threads, the sum of
  mostly-empty TLABs is a real fraction of eden. This is one of several reasons Topic 98's
  "one thread pool per subsystem, all unbounded" pattern is expensive.
- A **burst** of new threads (a thread-per-request model, or a pool that just grew) each
  demands a TLAB, which can trigger a young collection earlier than the allocation volume
  alone would suggest.

**Refill and waste.** When the remaining space in a TLAB is too small for the next
allocation, the JVM must choose:

| Situation | What happens | Cost |
|---|---|---|
| Remaining space is small (below the waste threshold) | Retire the TLAB, **abandoning the remainder** as waste, and get a fresh one | One atomic operation, plus the wasted bytes |
| Remaining space is large but the object is bigger | Allocate the object **directly in eden**, outside any TLAB, via an atomic bump of the shared top pointer | One atomic operation per allocation — the "slow path" |
| The object is very large | May be allocated directly in old gen (or, under G1, as a humongous object — Topic 71) | Skips the young generation entirely |

`-XX:TLABWasteTargetPercent` (default 1) expresses the target waste as a percentage of
eden, and `-XX:TLABRefillWasteFraction` (default 64) bounds the waste per refill to about
1/64 of the TLAB. The practical reading: **the JVM is willing to throw away a small slice
of eden to keep allocation lock-free**, and that trade is almost always correct.

Where it stops being correct: a workload allocating many objects that are large relative
to the TLAB. Each one takes the slow path, each slow path is an atomic operation on a
shared pointer, and under high thread counts that pointer becomes a contention point. The
symptom is CPU time in allocation with a normal-looking GC log.

Observe TLAB behaviour directly:

```bash
# Unified logging tag for TLAB detail. Verify availability first:
java -Xlog:help | grep -i tlab
java -Xlog:gc+tlab=trace,gc+tlab+start=trace:file=tlab.log -jar orderflow.jar
```

**One-line honesty note:** the exact TLAB log tags and their level names have shifted
across JDK versions, and `-XX:+PrintTLAB` from the pre-JDK-9 era no longer exists. Settle
what your build supports with `java -Xlog:help` before concluding TLABs are inactive.

### Why survivor copying, not allocation, is the cost

Here is the accounting that makes the whole topic click.

A young collection does approximately this much work:

```
  work  ≈  root scanning
        +  (bytes of LIVE data)  ×  (cost to copy a byte + cost to update references to it)
        +  fixed per-collection overhead
```

Notice what is **not** in that expression: the amount of garbage. Dead objects are never
visited. A young generation containing 900 MB of garbage and 10 MB of live data costs the
same as one containing 10 MB of garbage and 10 MB of live data — the collection just
happens less often in the second case.

Now trace what happens to one object under three lifetimes, at a tenuring threshold of 15:

| Lifetime | Copies | Ends up | Ongoing cost |
|---|---|---|---|
| Dies in eden before the first collection | **0** | nowhere — never touched | **zero** |
| Survives 3 collections, then dies | 3 | freed from a survivor space | 3 copies |
| Survives 15 collections | 15 | promoted to old gen | 15 copies **plus** it now participates in every old-gen marking cycle, and it costs remembered-set maintenance if it points at young objects |

The third row is the expensive one, and it is the row that **object pooling puts you in
deliberately**.

That is the mechanical justification for the mechanical statement. Say it in your own
words before moving on: *allocation is free; copying is the cost; promotion is the
compounding cost.*

### Ages, the tenuring threshold, and the age table

Every object header (Topic 69) reserves a few bits for an **age** — the number of young
collections it has survived. The header has room for a small maximum, which is why
`-XX:MaxTenuringThreshold` cannot exceed 15.

Promotion happens when either:

- the object's age reaches the tenuring threshold; **or**
- the target survivor space cannot hold everything that survived. Then objects are
  promoted regardless of age. This is **premature promotion**, and it is Trap 4.

The JVM also *adapts* the effective threshold. It aims to keep the survivor space around
`-XX:TargetSurvivorRatio` percent full (default 50). If survivors overflow that target, the
JVM lowers the effective threshold so more objects tenure sooner. So the threshold you set
is a maximum, not a setting.

The diagnostic that makes all of this visible is the **age table**:

```
Desired survivor size 8388608 bytes, new threshold 7 (max 15)
- age   1:    4194304 bytes,    4194304 total
- age   2:    1048576 bytes,    5242880 total
- age   3:     262144 bytes,    5505024 total
- age   7:      65536 bytes,    5570560 total
```

***Illustration of the format, not captured output. The byte values are placeholders
chosen to show the field positions.***

Enable it:

```bash
-Xlog:gc+age=trace:file=age.log:time,uptime,level,tags
```

**How to read the shape** — and shape is all you should read, never my placeholder
numbers:

| Shape you see | What it means |
|---|---|
| A steep decline: most bytes at age 1, very little beyond age 2 or 3 | Healthy. Objects are dying young exactly as the hypothesis predicts. |
| "new threshold" reported well below "max" | The JVM has *lowered* the effective threshold because survivors were overflowing. Premature promotion is happening. Trap 4. |
| A fat tail — significant bytes at high ages | Something is genuinely medium-lived: a cache with a TTL, a session, a batch accumulating results. Those bytes will be promoted. Expect old-gen growth and plan for it. |
| Bytes concentrated at the *maximum* age | Objects are reaching the threshold and tenuring on schedule. Fine if the volume is small; a promotion-rate problem if it is not. |
| The table is absent from your log | The tag is not enabled, or your collector does not emit it. Check `java -Xlog:help`. |

### Promotion, and the write barrier it costs you

Promotion is not just a copy. An object in old gen that holds a reference to a young
object creates an **old-to-young reference**, and the collector must find those without
scanning all of old gen. It does this with a **card table** (Serial/Parallel) or
**remembered sets** (G1), maintained by a **write barrier**: extra instructions the JIT
emits after every reference-field store.

Topic 71 covers G1's version in detail. The point for this document is the accounting:

> Promoting an object does not just cost the copy. It adds a long-lived participant to
> every old-gen marking cycle, and every reference *from* it *into* young generation costs
> barrier work on your application threads for as long as it lives.

This is the second reason pooling backfires. A pool is, by construction, a long-lived
old-gen object holding references to objects it hands out — which is exactly the
old-to-young reference pattern the barrier exists to track.

### Metaspace and compressed class space

When Topic 67's class loading defines a class, the resulting metadata — the `Klass`
structure, the method bytecode, the runtime constant pool — is allocated in **metaspace**,
which is native memory, in chunks, **per class loader**.

Two separate regions, and the distinction matters because they have separate errors:

| Region | Holds | Flag | Error message |
|---|---|---|---|
| **Metaspace** | Class metadata generally | `-XX:MaxMetaspaceSize` (unlimited by default) | `OutOfMemoryError: Metaspace` |
| **Compressed class space** | The `Klass` pointers, when compressed class pointers are enabled | `-XX:CompressedClassSpaceSize` (commonly 1 GB reserved) | `OutOfMemoryError: Compressed class space` |

Facts that follow, and each one is a mistake somebody makes:

- **`-Xmx` does not bound metaspace.** Raising it cannot fix a metaspace OOM. It makes the
  footprint larger and the container OOM kill nearer.
- **Metaspace is freed per class loader, not per class** (Topic 67). Reclaiming it requires
  a whole loader to become unreachable.
- **Metaspace pressure triggers heap collections.** `-XX:MetaspaceSize` sets the initial
  high-water mark at which a collection is triggered to try to unload classes. In a GC log
  this appears with the cause `Metadata GC Threshold` — a GC that has nothing to do with
  your heap.
- **In a container, cap it.** An unbounded metaspace turns a class-loader leak into an OOM
  *kill* — exit code 137, no heap dump, no error in your log. Capping it converts the same
  bug into an `OutOfMemoryError` you can dump and diagnose. That trade is almost always
  worth making.

```bash
jcmd <pid> VM.metaspace            # totals plus a per-classloader breakdown
jcmd <pid> GC.heap_info            # heap only: young/old/humongous, region counts under G1
```

### The full footprint — the sentence that prevents an OOM kill

Say this out loud once:

> **`-Xmx` bounds the Java heap. The container limit bounds everything.**

Everything, for a JVM, means at least:

| Component | Scales with | Bounded by |
|---|---|---|
| Java heap | your live set + allocation headroom | `-Xmx` / `MaxRAMPercentage` |
| Metaspace + compressed class space | number of classes and class loaders | `MaxMetaspaceSize` / `CompressedClassSpaceSize` (uncapped by default) |
| Code cache | how much code gets JIT-compiled | `ReservedCodeCacheSize` |
| Thread stacks | thread count × `-Xss` | nothing, until you run out |
| Direct byte buffers / FFM arenas | NIO, Netty, some drivers | `MaxDirectMemorySize` (defaults to about `-Xmx` if unset) |
| GC internal structures | heap size and reference density | implicit |
| JVM native + malloc arenas | everything else | `MALLOC_ARENA_MAX` on glibc |

Topic 80 gives you Native Memory Tracking to attribute the gap between heap and RSS.
Topic 82 gives you the container arithmetic. This document gives you the sentence.

---

## Example 1 — minimal

Two workloads with **identical allocation volume and identical allocation rate**. The only
difference is how long each object lives. This is the entire topic in one program.

```java
package com.orderflow.lab.mem;

import java.util.ArrayDeque;
import java.util.Deque;

/**
 * Allocates the SAME number of bytes per second in both modes.
 *
 *   churn : every object becomes unreachable immediately.  Dies in eden.
 *   retain: the last RETAINED objects are kept alive in a bounded deque, so a
 *           steady trickle survives young collections and eventually promotes.
 *
 * Run both with IDENTICAL JVM flags. The only variable is object lifetime.
 */
public final class LifetimeProbe {

    /** 4 KB each: comfortably below any TLAB or humongous threshold. */
    private static final int OBJECT_BYTES = 4096;

    /** Held so the JIT cannot prove the allocation is dead (Topic 75). */
    private static byte[] sink;

    public static void main(String[] args) throws Exception {
        String mode        = args[0];                       // "churn" or "retain"
        int    perSecond   = Integer.parseInt(args[1]);     // e.g. 50000
        int    seconds     = Integer.parseInt(args[2]);     // e.g. 300
        int    retained    = Integer.parseInt(args[3]);     // e.g. 20000 (retain mode only)

        Deque<byte[]> live = new ArrayDeque<>();
        long intervalNanos = 1_000_000_000L / perSecond;
        long next = System.nanoTime();

        for (long i = 0; i < (long) perSecond * seconds; i++) {
            byte[] o = new byte[OBJECT_BYTES];
            o[0] = 1;                                        // touch it, so it is real
            if ("retain".equals(mode)) {
                live.addLast(o);
                while (live.size() > retained) { live.removeFirst(); }
            } else {
                sink = o;                                    // immediately overwritten
            }

            // Crude pacing so BOTH modes allocate at the same rate.
            next += intervalNanos;
            long delay = next - System.nanoTime();
            if (delay > 0) { java.util.concurrent.locks.LockSupport.parkNanos(delay); }
        }
        System.out.println("done, retained=" + live.size());
    }
}
```

> The pacing loop is deliberately crude, and it is **not** a benchmark. It exists only to
> equalise allocation rate between the two modes. Topic 77 explains why a hand-written
> timing loop is never a measurement; here it is a throttle, which is a different job.

Run both with identical flags:

```bash
javac -d out LifetimeProbe.java

FLAGS="-Xmx512m -Xms512m -XX:+UseG1GC"
LOG="gc*,gc+heap=debug,gc+age=trace"

java $FLAGS -Xlog:$LOG:file=logs/churn.log:time,uptime,level,tags \
  -cp out com.orderflow.lab.mem.LifetimeProbe churn 50000 300 0

java $FLAGS -Xlog:$LOG:file=logs/retain.log:time,uptime,level,tags \
  -cp out com.orderflow.lab.mem.LifetimeProbe retain 50000 300 20000
```

Note the retain-mode arithmetic: 20,000 objects × 4 KB = about **80 MB permanently live**
on a 512 MB heap. That is the live set you have introduced. Allocation rate is unchanged.

**WHAT TO LOOK FOR.** Four counts and one shape, in this order:

```bash
grep -c 'Pause Young'                  logs/churn.log logs/retain.log
grep -c 'Pause Full'                   logs/churn.log logs/retain.log
grep -c 'Concurrent'                   logs/churn.log logs/retain.log
grep -A8 'Desired survivor size'       logs/retain.log | head -40
```

| What you see | What it means |
|---|---|
| Similar young-collection **counts** in both, but longer pause **durations** in retain mode | The expected core result. Same allocation rate means eden fills at the same rate, so collections happen equally often. Longer pauses mean more live data to copy. **Cost tracks the live set.** |
| The retain log's `before->after` heap values leave much more behind after each collection | That residue is your live set plus what has been promoted. In churn mode `after` should return to a low, roughly constant value. |
| Concurrent cycles (or, on Parallel/Serial, full collections) appear in retain mode and not in churn mode | Old gen is filling, because objects are being promoted. This is the promotion consequence, visible directly. |
| The age table in retain mode shows a fat tail; churn mode's is steeply declining or absent | The distributional evidence for the same conclusion. |
| Both logs look nearly identical | Your retained set is too small relative to the heap. Raise `retained` until 20–40% of the heap is live, or drop `-Xmx`. Do **not** conclude "lifetime doesn't matter" — conclude "at this ratio, on this heap, it didn't". |
| Retain mode throws `OutOfMemoryError` | You retained more than the heap holds. That is a correct outcome and a useful one: it is the difference between a leak and a large live set (Topics 70, 79) — one grows without bound, the other plateaus. |
| Churn mode has **more** young collections than retain mode | Genuinely surprising, and worth chasing. Most likely retain mode's live set shrank the effective eden (G1 will grow old gen at eden's expense), which changes the collection *interval*. Check region counts in the `gc,heap` lines before concluding your pacing is broken. |

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
| Arrival rate | open-model, roughly 400 requests per second total |
| Latency budget | endpoint p99s within the recorded baseline |
| Baseline artefacts | `/docs/java/baselines/` |

### The change that causes the incident

An engineer profiles `orderflow` under load (Topic 78) and finds that a large share of
allocation comes from `OrderLineDto` objects created on the `GET /orders/{id}` path —
unsurprising, since a large order has many lines and the endpoint builds one DTO per line.

The conclusion drawn is the one almost everybody draws coming from a
manual-memory-management or JavaScript background:

> "We're allocating millions of DTOs per minute. That's obviously GC pressure. Let's pool
> them."

```java
package com.orderflow.orders;

import java.util.ArrayDeque;
import java.util.Deque;
import java.util.List;
import org.springframework.stereotype.Component;

/**
 * "Reduces GC pressure by reusing DTOs instead of allocating them."
 *
 * Every sentence in that comment is wrong, and the code review approved it because
 * the reasoning sounds like every performance article the reviewer has ever read.
 */
@Component
public class OrderLineDtoPool {

    private static final int CAPACITY = 100_000;

    private final Deque<OrderLineDto> pool = new ArrayDeque<>(CAPACITY);

    public synchronized OrderLineDto borrow() {
        OrderLineDto dto = pool.pollFirst();
        return dto != null ? dto : new OrderLineDto();
    }

    public synchronized void giveBack(List<OrderLineDto> used) {
        for (OrderLineDto dto : used) {
            dto.reset();
            if (pool.size() < CAPACITY) { pool.addLast(dto); }
        }
    }
}
```

### What you observe, in the order you observe it

1. **Allocation rate falls.** The metric the change was made to improve does improve. The
   dashboard shows fewer bytes allocated per second. Everybody is pleased.
2. **Young-collection frequency falls too** — eden fills more slowly. This looks like
   further confirmation.
3. **Young-collection pause duration rises.** Each collection now has more live data to
   copy, because pooled DTOs are alive by construction.
4. **Promotion rate rises sharply.** Pooled objects survive every young collection and
   reach the tenuring threshold on schedule. Within minutes, 100,000 DTOs plus their
   fields are in old gen.
5. **Old-generation occupancy climbs and does not fall.** The pool is a live root chain
   (Topic 70): those objects are reachable, so they are not garbage, so no collector will
   ever free them.
6. **Concurrent cycles become frequent, then a `Pause Full` appears** (Topic 71), and p99
   for **every** endpoint — including `GET /products`, which touches none of this code —
   degrades past the baseline.
7. The engineer, looking at a dashboard that shows a *reduced* allocation rate, concludes
   the problem must be something else entirely.

**This is the trap the mechanical statement warns about, in its natural habitat.** The
change did exactly what it claimed. It reduced allocation. Allocation was never the cost.

### The arithmetic that explains it

Before the change: each `OrderLineDto` was allocated with a pointer bump, lived for the
duration of one request, and was dead before the next young collection. Cost of copying:
**zero**. Cost of promotion: **zero**. Cost of old-gen participation: **zero**.

After the change: up to 100,000 DTOs are permanently reachable. Assume — and you will
measure this properly in Topic 69 rather than trusting arithmetic — that each DTO plus its
referenced `String`s and `BigDecimal`s is on the order of a couple of hundred bytes. That
is tens of megabytes of permanently live data on a 1200 MB heap, plus:

- every one of those objects is **copied** on each young collection until it tenures;
- once promoted, every one participates in **every** concurrent marking cycle, forever;
- the pool's references into freshly-borrowed objects create **old-to-young references**,
  which cost write-barrier work on your request threads (Topic 71) on every store.

You traded a free death for a permanent tax.

### The diagnosis, as commands

```bash
# 1. Confirm collector, heap and flags on the RUNNING container. Never assume.
jcmd $(pgrep -f orderflow) VM.flags -all | grep -E 'UseG1GC|MaxHeapSize|InitialHeapSize|MaxTenuringThreshold'

# 2. Heap and region breakdown right now.
jcmd $(pgrep -f orderflow) GC.heap_info

# 3. Restart with the log that shows promotion and ages, and re-run the SAME k6 profile.
-Xlog:gc*,gc+heap=debug,gc+age=trace:file=/var/log/orderflow/gc.log:time,uptime,level,tags:filecount=5,filesize=50m

# 4. Old-gen occupancy after each collection. Rising = promotion.
grep 'Old regions\|Pause Young' /var/log/orderflow/gc.log | tail -60

# 5. The age distribution.
grep -A12 'Desired survivor size' /var/log/orderflow/gc.log | tail -40
```

**WHAT TO LOOK FOR:** the *pair* of numbers — allocation rate and promotion rate — derived
using the formulas in the Measurement section. One number alone cannot distinguish these
outcomes:

| What you see | What it means |
|---|---|
| Allocation rate **down**, promotion rate **up**, old-gen occupancy climbing | Confirmed. You reduced the cheap thing and increased the expensive one. Remove the pool. |
| Allocation rate down, promotion rate flat, pauses unchanged | The pool is genuinely harmless here — probably because the pooled objects were already going to be promoted, or the pool is tiny relative to the heap. Say so honestly and move on; not every pool is a mistake. |
| Allocation rate down, promotion rate up, but p99 **improved** | Possible and worth understanding rather than dismissing: if allocation was so extreme that collections were back-to-back, reducing it can win even at a promotion cost. Report both halves and state the heap size at which the trade flips. |
| Age table shows a fat tail exactly where the pool size predicts | Direct evidence tying the pool to the promotion. This is the screenshot for the postmortem. |
| Old gen grows but the age table has no tail | Not tenuring-by-age — this is **premature promotion** from survivor overflow (Trap 4), or direct old-gen allocation of large objects (Topic 71 humongous). Different fix. |
| Old gen grows and there is no pool | You are looking at a retention leak (Topic 79), not this topic. Take a heap dump. |

### The fix, and the correct version of the original instinct

**Fix 1 — delete the pool.** The DTOs were dying in eden for free. Restore that.

**Fix 2 — if allocation genuinely is the problem, allocate less rather than reusing more.**
There is a real and correct version of the engineer's instinct, and it is worth stating so
the lesson is not "never optimise allocation":

```java
// The DTOs existed because the endpoint returned entities' worth of data.
// The real question is: does the caller need every field of every line?
public interface OrderLineSummary {          // a Spring Data projection, Topic 47
    long productId();
    int  quantity();
    long unitPriceMinor();
}
```

A projection allocates fewer, smaller objects because it fetches fewer columns — and it
also removes Hibernate's per-entity snapshot (Topic 48), which is often the larger cost.
That reduces allocation **without** creating a live set. That is the shape of a correct
allocation optimisation.

**Fix 3 — when pooling is legitimately correct.** Pooling is right when the object is
genuinely expensive to *create*, not merely to allocate: a database connection, a thread, a
compiled regex, a `Cipher`, a large direct buffer whose native memory must be reserved
(Topic 80). The test is not "is this allocated often" — it is **"does creating this do
work beyond the pointer bump?"**

> **The rule to carry:** pool things that are expensive to *construct*. Never pool things
> that are merely *numerous*. Numerous-and-cheap is the case the JVM's allocator was
> designed for, and pooling converts it into the case the JVM is worst at.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — "allocation is expensive", so pool everything

**Wrong:** the `OrderLineDtoPool` above, or any of its cousins — a reusable
`StringBuilder` held in a field, a pooled DTO, a "recycled" event object, a
`ThreadLocal<byte[]>` scratch buffer that is larger than it needs to be.

**Exact symptom:** allocation rate falls (the metric you were watching improves), young
collections become **less frequent but longer**, promotion rate rises, old-gen occupancy
climbs monotonically, concurrent cycles get more frequent, and eventually a `Pause Full`
appears. p99 degrades across **all** endpoints, including ones that never touch the pooled
type. The dashboard that motivated the change shows an improvement throughout.

**Root cause:** allocation in a TLAB is a pointer bump — roughly ten instructions with no
synchronisation. A young collection's cost is proportional to the **live** set, because
dead objects are never visited. Pooling takes objects that would have been free to
collect and makes them permanently live, which converts zero copies into repeated copies
and then into promotion and permanent old-gen participation. You optimised the cheap axis
by degrading the expensive one.

**Fix:**

1. Delete the pool. Measure against the baseline before and after — this is the sort of
   change that must be justified with numbers in both directions.
2. If allocation genuinely dominates, reduce it at the source: fetch fewer columns
   (projections, Topic 47), avoid boxing in hot paths (Topic 01), avoid string
   concatenation in loops (Topic 18), avoid materialising intermediate collections
   (Topic 23).
3. Pool only what is expensive to **construct**: connections, threads, compiled patterns,
   direct buffers.
4. When you must hold state, bound it and give it a TTL — a `Caffeine` cache with a size
   or weight bound is a *managed* live set, which is a completely different thing from an
   unbounded one.

**How to confirm the diagnosis in one command:**

```bash
grep -A12 'Desired survivor size' /var/log/orderflow/gc.log | tail -40
```

A fat tail in the age distribution that appeared with the pool and disappears without it
is your evidence.

---

### Trap 2 — sizing `-Xmx` from the container memory limit

**Wrong:**

```yaml
# k8s
resources:
  limits:
    memory: 2Gi
```
```bash
java -Xmx2g -jar orderflow.jar     # "the container has 2 GiB, so give the heap 2 GiB"
```

**Exact symptom:** the pod is killed. Not an `OutOfMemoryError` — a **kill**. Specifically:

- exit code **137** (128 + SIGKILL), pod status `OOMKilled`;
- **no heap dump**, even with `-XX:+HeapDumpOnOutOfMemoryError`, because the JVM never got
  to run any code;
- **nothing in the application log** — the last line is an ordinary request;
- the heap graph in your monitoring looks **healthy** right up to the kill, often well
  under 100% of `-Xmx`;
- it correlates with traffic (more threads, more classes, more direct buffers) rather than
  with heap growth.

**Root cause:** `-Xmx` bounds the Java heap. The container limit bounds the **process**.
Between them sit metaspace, compressed class space, the code cache, one stack per thread,
direct byte buffers, GC-internal structures and the JVM's own native allocations. Setting
`-Xmx` to the whole limit guarantees that as soon as any of those grows — and they all
grow as the service warms up and thread count rises — the process exceeds the cgroup limit
and the kernel kills it.

**Fix:**

```bash
java -XX:MaxRAMPercentage=60 \
     -XX:MaxMetaspaceSize=256m \
     -XX:ReservedCodeCacheSize=192m \
     -XX:MaxDirectMemorySize=256m \
     -Xss512k \
     -XX:NativeMemoryTracking=summary \
     -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/dumps \
     -jar orderflow.jar
```

Every one of those is a deliberate decision, not a cargo-cult copy:

- `MaxRAMPercentage` reads the **cgroup** limit rather than the host's memory, so it is the
  container-correct way to express "a fraction of what I'm allowed". The default is 25%,
  which is far too low for a container that exists to run one JVM — that is Topic 82's
  argument.
- Capping metaspace converts a class-loader leak (Topic 67) from an undiagnosable OOM kill
  into an `OutOfMemoryError` with a heap dump.
- Capping direct memory does the same for Topic 80's failure shape.
- `-Xss512k` matters because the cost is per thread, and Topic 98's thread counts make it
  a real number.

Then **attribute the gap**, do not guess at it:

```bash
jcmd <pid> VM.native_memory summary
```

**WHAT TO LOOK FOR:** the difference between the JVM's own committed total and the RSS the
operating system reports for the same process. Topic 80 is the full treatment. The
percentage-of-limit number to start from is a starting point to be measured away from, not
a rule.

---

### Trap 3 — tuning young generation with flags your collector ignores

**Wrong:**

```bash
java -XX:+UseG1GC -Xmn512m -XX:SurvivorRatio=4 -XX:NewRatio=3 -jar orderflow.jar
```

copied from a blog post written for Parallel GC in 2013.

**Exact symptom:** the *most* common outcome is that nothing measurable changes, which is
its own problem: you now believe you have tuned something and you have not, and the belief
survives into the next incident. The less common but worse outcome is that pause behaviour
gets *worse* and more variable, because pinning the young generation prevents G1's control
loop from resizing it to meet the pause goal.

The tell: `jcmd <pid> VM.flags -all` shows the flags set, and `GC.heap_info` shows a young
generation that does not respond to load the way the log's collection intervals suggest it
should.

**Root cause:** `-Xmn`, `-XX:NewRatio` and `-XX:SurvivorRatio` come from the fixed-space,
contiguous-generation model of Serial and Parallel GC. G1's heap is regions with mutable
labels, and its young size is an output of the pause-goal control loop, not an input.
Forcing a fixed young size does not make G1 behave like Parallel; it makes G1 behave like
G1 with one hand tied.

**Fix:**

1. **Start from defaults.** Reproduce the Topic 65 baseline with the collector's defaults
   and record it.
2. **Verify which flags your collector actually honours**, on your JDK, rather than
   trusting any document:

```bash
java -XX:+UseG1GC -XX:+PrintFlagsFinal -version | grep -E 'NewRatio|SurvivorRatio|NewSize|MaxNewSize|MaxTenuringThreshold'
java -XX:+UseParallelGC -XX:+PrintFlagsFinal -version | grep -E 'NewRatio|SurvivorRatio'
# diff the two outputs; the differences are the collector-specific story
```

3. **Change one flag at a time** and compare percentiles against the recorded baseline. A
   change inside your run-to-run noise is not a change.
4. If you genuinely need a bigger young generation under G1, the knobs are
   `-XX:G1NewSizePercent` / `-XX:G1MaxNewSizePercent` — and Topic 71's Trap 4 explains why
   reaching for them is usually the wrong move.

**One-line honesty note:** exactly which of these flags G1 reads, ignores, or reinterprets
has varied across JDK versions, and I will not assert a per-version answer. `-XX:+PrintFlagsFinal`
and a controlled A/B against your baseline settle it for your runtime in ten minutes.

---

### Trap 4 — premature promotion from an undersized survivor space

**Wrong:** either an explicitly tiny survivor space, or — much more commonly — a workload
whose surviving volume per collection simply exceeds what the survivor space can hold.
Nobody chooses this; you arrive at it by adding a moderately-lived cache, raising
concurrency, or increasing request payload size.

**Exact symptom:** old-generation occupancy grows steadily even though nothing in your
application is obviously long-lived. Concurrent cycles run far more often than your live
set should require. Crucially, the **age table shows almost nothing above age 1 or 2**, and
the reported "new threshold" is well below "max". You are not tenuring objects because they
are old; you are tenuring them because there is nowhere else to put them.

The second tell is that a lot of what gets promoted is **garbage by the next cycle** — old
gen grows fast and then a concurrent cycle reclaims a large fraction of it. That is the
signature of promoting things that were about to die.

**Root cause:** when survivors do not fit in the target survivor space, the collector
promotes them regardless of age. Objects that would have died at age 3 in a survivor space
are now old-gen objects, where they can only be reclaimed by a much more expensive
mechanism.

**Fix:**

1. **Confirm it from the age table first.** A low effective threshold plus a thin age
   distribution is the diagnosis; anything else is guessing.
2. **Reduce what survives.** This is the real fix, and it is an application change: a
   smaller working set per request, fewer retained intermediates, a bounded cache.
3. **Give the young generation more room** so more objects die before the collection
   reaches them. Under G1 this means a larger heap or letting the adaptive sizing work,
   not pinning `-Xmn`.
4. Only after the above, and only with a measurement, consider adjusting the tenuring
   threshold — and understand that raising it keeps objects in survivor spaces longer,
   which means *more copying*, not less. There is no free direction here.

**How to confirm the fix worked:**

```bash
grep -A12 'Desired survivor size' /var/log/orderflow/gc.log | tail -40
```

A fix has worked when the reported "new threshold" rises back toward "max" and old-gen
growth flattens — not when a pause number moved.

---

### Trap 5 — reading heap-used-after-GC as "the live set"

**Wrong:** exporting `Runtime.getRuntime().totalMemory() - freeMemory()` as a "memory
used" metric, or taking the `after` number from a single GC log line as the live set, and
alerting on it.

**Exact symptom:** an alert that fires constantly and means nothing. The graph is a sawtooth
that varies by hundreds of megabytes between adjacent samples depending purely on when the
sample landed relative to a collection. Nobody trusts it, so it gets muted, so a real leak
goes unnoticed for weeks.

The secondary symptom is worse: a capacity decision — "we need 4 GB because that's what the
graph shows" — made from a number that is mostly garbage-not-yet-collected.

**Root cause:** heap-used at an arbitrary instant is live data **plus** garbage that has
not been collected yet. Even immediately after a young collection, the `after` figure is
live young data plus everything in old gen — including old-gen garbage awaiting a
concurrent cycle, plus **floating garbage** (objects that died after concurrent marking
started and cannot be reclaimed until the next cycle — Topic 71).

**Fix:** measure the live set the way Topic 70 defines it — from the **floor** of
old-generation occupancy immediately after a full or mixed collection, sampled over a long
run, not from any single reading:

```bash
# Old-gen occupancy after collections. The FLOOR over a long run approximates the live set.
grep -E 'Pause Full|Pause Young \(Mixed\)' /var/log/orderflow/gc.log | tail -50
```

And when you need a defensible number for capacity planning, take a heap dump and let a
tool compute retained sizes (Topic 79) rather than reading a gauge. For a dashboard, export
the collector's own accounting — allocation, promotion and post-collection occupancy as
separate series (Topic 118) — rather than one ambiguous "memory used" line.

---

## Hands-on proof

Every command here is one **you** run. I have no JVM and will not print output and call it
captured. What follows is the exact command, what to look for, and how to read every
plausible result — including the ones that contradict the expected answer.

### Setup

```bash
mkdir -p ~/java-lab/68/logs && cd ~/java-lab/68
java --version                 # expect 21 or 25; record which
java -Xlog:help | grep -iE 'tlab|age|heap|metaspace'    # what tags does YOUR build have?
```

### Proof 1 — establish what your JVM actually chose

```bash
java -XX:+PrintFlagsFinal -version | grep -E \
 'MaxHeapSize|InitialHeapSize|NewSize|MaxNewSize|SurvivorRatio|MaxTenuringThreshold|TargetSurvivorRatio|UseTLAB|TLABSize|ResizeTLAB|MaxMetaspaceSize|CompressedClassSpaceSize|UseCompressedOops|UseG1GC|UseParallelGC|UseSerialGC|UseZGC'
```

**WHAT TO LOOK FOR:** which collector was chosen, what the heap defaults to, and whether
metaspace is bounded.

| What you see | What it means |
|---|---|
| `UseG1GC = true` and a heap around a quarter of your machine's memory | The normal default on a server-class machine. Note the quarter: `MaxRAMPercentage` defaults to 25, which is wrong for a dedicated container (Topic 82). |
| `UseSerialGC = true` | The JVM decided the machine is not server-class — fewer than two visible CPUs, or a small memory limit. Extremely common in a container with a small CPU quota, and it silently changes everything in this document's tuning vocabulary. |
| `MaxMetaspaceSize` is a very large number (effectively unlimited) | The default. In a container, cap it — Trap 2. |
| `UseTLAB = true` | Expected. If it is false, someone disabled it; allocation is now an atomic operation per object and throughput will show it. |
| `MaxTenuringThreshold = 15` | The usual default. Remember it is a *maximum*; the effective threshold adapts. |

Then do the same on the **running** service, because environment variables and wrapper
scripts change things:

```bash
jcmd $(pgrep -f orderflow) VM.flags -all
```

### Proof 2 — watch eden fill and drain

```bash
java -Xmx512m -Xms512m -XX:+UseG1GC \
  -Xlog:gc*,gc+heap=debug:file=logs/basic.log:time,uptime,level,tags \
  -cp out com.orderflow.lab.mem.LifetimeProbe churn 50000 120 0

grep 'Pause Young' logs/basic.log | tail -20
```

**WHAT TO LOOK FOR:** the `before->after(total)` triple on each young collection, and the
uptime interval between consecutive collections.

| What you see | What it means |
|---|---|
| `after` returns to roughly the same low value every time | Nothing is surviving. This is the healthy churn shape and the baseline for every comparison you will make. |
| `after` creeps upward collection by collection | Something is being promoted or retained. That upward drift *is* the promotion signal, and its slope is the promotion rate. |
| The interval between collections is roughly constant | Constant allocation rate — which your pacing loop was designed to produce. If it is not constant, your pacing is being outrun; lower the rate. |
| The interval shrinks over time | Effective eden is shrinking, usually because old gen is growing at its expense. A second-order symptom of promotion. |

### Proof 3 — see TLAB behaviour

```bash
java -Xlog:help | grep -i tlab            # confirm the tag exists on your build first
java -Xmx512m -XX:+UseG1GC \
  -Xlog:gc+tlab=trace:file=logs/tlab.log \
  -cp out com.orderflow.lab.mem.LifetimeProbe churn 50000 60 0
wc -l logs/tlab.log
```

**WHAT TO LOOK FOR:** whether allocations are being served from TLABs at all, and how much
space is being wasted at refill.

| What you see | What it means |
|---|---|
| Lines reporting TLAB refills with a small waste fraction | Normal. The JVM is trading a little eden for lock-free allocation, which is the right trade. |
| A high proportion of allocations reported outside TLABs | Your objects are large relative to the TLAB. Each one takes the slow path — an atomic operation on a shared pointer. Under high thread counts that becomes contention. Consider whether the objects need to be that large (a pre-sized buffer, a big array — and see Topic 71 on humongous objects). |
| The log is empty and the tag exists | TLAB logging may be at a different level on your build, or disabled in product builds. Do not conclude TLABs are off; confirm with `-XX:+PrintFlagsFinal -version \| grep UseTLAB`. |
| Very many small TLABs and many threads | Each thread holds a chunk of eden whether it allocates or not. With hundreds of threads this is a measurable slice of your young generation (Topic 98). |

Then run the A/B control that proves TLABs matter:

```bash
# A: normal.  B: TLABs disabled — every allocation becomes an atomic operation.
java -Xmx512m -XX:+UseTLAB  -cp out com.orderflow.lab.mem.LifetimeProbe churn 200000 60 0
java -Xmx512m -XX:-UseTLAB  -cp out com.orderflow.lab.mem.LifetimeProbe churn 200000 60 0
```

**WHAT TO LOOK FOR:** whether the run *completes on schedule* in both cases — the pacing
loop will start falling behind if allocation gets expensive enough. **Do not** report a
wall-clock difference as a benchmark result; it is not one (see the Measurement section).
`-XX:-UseTLAB` is a diagnostic control, never a production setting.

### Proof 4 — read the age table

```bash
java -Xmx512m -Xms512m -XX:+UseG1GC \
  -Xlog:gc+age=trace:file=logs/age.log:time,uptime,level,tags \
  -cp out com.orderflow.lab.mem.LifetimeProbe retain 50000 180 20000

grep -A15 'Desired survivor size' logs/age.log | tail -60
```

**WHAT TO LOOK FOR:** the reported threshold versus the max, and the shape of the
distribution. Use the table in the Machine-level reality section to read it.

### Proof 5 — metaspace is not the heap

```java
package com.orderflow.lab.mem;

import java.lang.invoke.MethodHandles;
import java.util.ArrayList;
import java.util.List;

/**
 * Defines classes until metaspace is exhausted. Watch the HEAP stay healthy.
 * Run with a small metaspace cap so it fails quickly and safely.
 */
public final class MetaspaceProbe {
    static final List<Class<?>> KEEP = new ArrayList<>();   // retain so nothing unloads

    public static void main(String[] args) throws Exception {
        byte[] template = java.nio.file.Files.readAllBytes(
                java.nio.file.Path.of(args[0]));            // any small .class file
        for (int i = 0; ; i++) {
            KEEP.add(MethodHandles.lookup().defineHiddenClass(template, true).lookupClass());
            if ((i & 0x3FF) == 0) {
                Runtime r = Runtime.getRuntime();
                System.out.printf("defined=%d heapUsedMB=%d%n",
                        i, (r.totalMemory() - r.freeMemory()) / (1024 * 1024));
            }
        }
    }
}
```

```bash
java -Xmx512m -XX:MaxMetaspaceSize=64m \
  -Xlog:gc*:file=logs/meta.log:time,uptime,level,tags \
  -cp out com.orderflow.lab.mem.MetaspaceProbe out/com/orderflow/lab/mem/LifetimeProbe.class
```

**WHAT TO LOOK FOR:** the exact error message, and the heap usage figure at the moment it
is thrown.

| What you see | What it means |
|---|---|
| `OutOfMemoryError: Metaspace` while heap usage is low | **The point of the proof.** The heap was never the constraint. Anyone who responds to this error by raising `-Xmx` has diagnosed the wrong region. |
| `OutOfMemoryError: Compressed class space` instead | You hit the separate class-pointer region first. Different flag: `-XX:CompressedClassSpaceSize`. Same lesson. |
| `Metadata GC Threshold` lines in the GC log before the error | Metaspace pressure triggering heap collections in the hope of unloading classes. A GC caused by something that is not the heap — worth seeing once. |
| No error, it runs forever | Your `MaxMetaspaceSize` did not take effect, or hidden classes are being unloaded because `KEEP` is not retaining them. Verify with `jcmd <pid> VM.metaspace` in another terminal. |
| The process is killed by the container instead | You ran it without a metaspace cap, in a container. That is Trap 2, reproduced by accident — and a useful accident. |

Now watch it from outside while it runs:

```bash
jcmd <pid> VM.metaspace | head -30
jcmd <pid> GC.heap_info
```

The contrast between those two outputs is the proof.

### Proof 6 — heap versus footprint on the real service

```bash
# Start orderflow with native memory tracking on.
java -Xmx1200m -Xms1200m -XX:NativeMemoryTracking=summary -jar orderflow.jar

# Then, under the Topic 65 load profile:
jcmd $(pgrep -f orderflow) VM.native_memory summary
ps -o pid,rss,vsz -p $(pgrep -f orderflow)
cat /sys/fs/cgroup/memory.current 2>/dev/null || cat /sys/fs/cgroup/memory/memory.usage_in_bytes
```

**WHAT TO LOOK FOR:** the gap between `-Xmx` and RSS, and which NMT category accounts for
it.

| What you see | What it means |
|---|---|
| RSS meaningfully above heap, with the gap attributed across class metadata, code, threads and GC | Normal and expected. Now you can size the container from evidence rather than from a rule of thumb. |
| Thread category large | Thread count × stack size. Lower `-Xss` or lower the thread count (Topic 98). |
| Class category large and growing | Class-loader leak (Topic 67) or heavy runtime class generation. |
| A large gap NMT does **not** account for | Native allocations outside the JVM's own tracking — a native library, glibc malloc arenas, or direct buffers depending on your JDK's accounting. Topic 80. |
| RSS close to the cgroup limit while heap is at 50% | You are on the path to Trap 2's OOM kill. Act before it happens. |

---

## Failure drill

**Mandatory.** Do not read the "how to read it" table until you have produced the result
yourself and written down what you saw. The point is not the knowledge; it is the memory of
watching two identical allocation rates produce completely different collector behaviour.

### The assignment, restated from the master plan

> Allocate short-lived vs long-lived objects at the **same rate** under **identical
> flags**, and compare promotion in `-Xlog:gc*`.

The equality of the allocation rate is the entire experimental design. If the rates differ,
you have proved nothing.

### Step 0 — establish the control

```bash
docker compose up -d
java -Xmx1200m -Xms1200m -XX:+UseG1GC \
  -Xlog:gc*,gc+heap=debug,gc+age=trace:file=/var/log/orderflow/gc-control.log:time,uptime,level,tags \
  -jar orderflow.jar

k6 run --out json=logs/control.json load/baseline.js
```

Record p50/p95/p99/p999, throughput, error rate, allocation rate, promotion rate, Full GC
count, and the age-table shape. **If these are not within ±10% of the committed baseline in
`/docs/java/baselines/`, stop.** The Topic 65 gate rule applies.

### Step 1 — add two endpoints that allocate identically

```java
package com.orderflow.lab;

import java.util.ArrayDeque;
import java.util.Deque;
import org.springframework.web.bind.annotation.*;

/**
 * DRILL CODE. Never merge this.
 *
 * Both endpoints allocate EXACTLY the same bytes per request. The only
 * difference is whether the objects remain reachable afterwards.
 */
@RestController
@RequestMapping("/lab/lifetime")
public class LifetimeEndpoints {

    private static final int CHUNK   = 4096;    // bytes per object
    private static final int PER_REQ = 64;      // objects per request  -> 256 KB / request
    private static final int KEEP    = 40_000;  // retained objects     -> ~160 MB live

    private final Deque<byte[]> retained = new ArrayDeque<>();

    /** Short-lived: every object is unreachable before the response is written. */
    @GetMapping("/short")
    public int shortLived() {
        int checksum = 0;
        for (int i = 0; i < PER_REQ; i++) {
            byte[] b = new byte[CHUNK];
            b[0] = 1; b[CHUNK - 1] = 1;
            checksum += b[0];                    // touch it so it is not trivially dead
        }
        return checksum;
    }

    /** Long-lived: identical allocation, but a bounded set stays reachable. */
    @GetMapping("/long")
    public synchronized int longLived() {
        int checksum = 0;
        for (int i = 0; i < PER_REQ; i++) {
            byte[] b = new byte[CHUNK];
            b[0] = 1; b[CHUNK - 1] = 1;
            checksum += b[0];
            retained.addLast(b);
            while (retained.size() > KEEP) { retained.removeFirst(); }
        }
        return checksum;
    }
}
```

> Two deliberate choices. `CHUNK` is 4 KB — small enough to stay well below any humongous
> threshold (Topic 71) so that G1's special large-object path is not a confounding
> variable. The `synchronized` on the second method keeps the deque correct without adding
> a second variable; Topic 85 will have opinions about it, and it is not what this drill
> measures.

### Step 2 — run each arm, one at a time, with identical flags

```bash
FLAGS="-Xmx1200m -Xms1200m -XX:+UseG1GC"
LOG="gc*,gc+heap=debug,gc+age=trace"

# Arm A — short-lived
java $FLAGS -Xlog:$LOG:file=/var/log/orderflow/gc-short.log:time,uptime,level,tags -jar orderflow.jar
k6 run --out json=logs/short.json load/lifetime-drill.js   # baseline mix + /lab/lifetime/short

# Arm B — long-lived. IDENTICAL flags, IDENTICAL k6 rates, only the path differs.
java $FLAGS -Xlog:$LOG:file=/var/log/orderflow/gc-long.log:time,uptime,level,tags -jar orderflow.jar
k6 run --out json=logs/long.json  load/lifetime-drill.js   # baseline mix + /lab/lifetime/long
```

```javascript
// load/lifetime-drill.js — the Topic 65 mix, plus ONE lifetime endpoint at a fixed rate.
import http from 'k6/http';

const LIFETIME_PATH = __ENV.LIFETIME_PATH;   // '/lab/lifetime/short' or '/lab/lifetime/long'

export const options = {
  scenarios: {
    baseline_mix: {
      executor: 'constant-arrival-rate',
      rate: 400, timeUnit: '1s', duration: '15m',
      preAllocatedVUs: 200, maxVUs: 600, exec: 'baselineMix',
    },
    lifetime: {
      executor: 'constant-arrival-rate',
      rate: 100, timeUnit: '1s', duration: '15m',
      preAllocatedVUs: 50, maxVUs: 200, exec: 'lifetime',
    },
  },
  // Open-model arrival rate per the Topic 65 gate: a fixed-VU loop under-reports the
  // tail because a stalled VU stops issuing requests. That is coordinated omission.
};

export function baselineMix() { /* the recorded 70/20/10 mix, unchanged */ }
export function lifetime()    { http.get(`http://localhost:8080${LIFETIME_PATH}`); }
```

**Fifteen minutes minimum per arm.** A short run can miss the concurrent cycles entirely,
and the whole point is what happens after old gen starts filling.

### Step 3 — what to capture

Write down, before reading anything else, for **each** arm:

1. **Allocation rate**, using the formula in the Measurement section. *These two numbers
   must be approximately equal.* If they are not, the experiment is invalid — fix the rates
   and re-run.
2. **Promotion rate**, using the formula in the Measurement section.
3. Young-collection **count** and the **interval** between them.
4. Young-collection **pause duration**: median and maximum.
5. Concurrent-cycle count; `Pause Full` count.
6. Old-gen occupancy at the end of the run.
7. The **age table shape** — where the bytes sit.
8. p50/p95/p99/p999 for **`GET /products`** — the endpoint you did not touch.

```bash
grep -c 'Pause Young'  /var/log/orderflow/gc-short.log /var/log/orderflow/gc-long.log
grep -c 'Pause Full'   /var/log/orderflow/gc-short.log /var/log/orderflow/gc-long.log
grep -c 'Concurrent Start' /var/log/orderflow/gc-short.log /var/log/orderflow/gc-long.log
grep -A12 'Desired survivor size' /var/log/orderflow/gc-long.log | tail -40
```

### How to read it

| What you see | What it means |
|---|---|
| Allocation rates approximately equal; promotion rate much higher in the long arm | **The drill has fired.** This is the result the whole document exists to produce. Write the sentence: "at an identical allocation rate, object lifetime alone changed promotion rate by a factor of N and moved p99 on an endpoint I did not touch." |
| Young-collection counts similar, pause durations higher in the long arm | Correct and mechanically important: same eden fill rate means same collection frequency; more live data means more copying per collection. **Cost tracks the live set.** |
| The long arm has concurrent cycles (or full collections) that the short arm does not | Old gen is filling from promotion. This is the compounding cost, visible. |
| Age table: short arm steeply declining, long arm with a fat tail or a lowered threshold | The distributional evidence for the same conclusion. If the long arm's *threshold* dropped, you also reproduced Trap 4 by accident — note it. |
| `GET /products` p99 degrades in the long arm only | The most important observation in the drill. GC is a **shared** resource. An endpoint that allocated no more than its twin degraded a completely unrelated endpoint. |
| Allocation rates are **not** equal | Invalid experiment. The usual cause is that the long arm's endpoint is slower (the `synchronized` block, or the deque work), so k6's open model delivered the same *request* rate but the service completed fewer. Check throughput, not just the configured rate, and equalise before drawing any conclusion. |
| The long arm throws `OutOfMemoryError` | `KEEP` is too large for a 1200 MB heap alongside the baseline working set. Lower it. The error is also a useful datum: it is the difference between a bounded live set and an unbounded one. |
| Both arms look identical | `KEEP` is too small to matter, or the retained bytes are being reclaimed because your deque is not actually retaining. Verify with `GC.heap_info` mid-run that old-gen occupancy differs. Do **not** conclude "lifetime doesn't matter" — conclude "at this retention, on this heap, it didn't", and raise `KEEP`. |
| The long arm is **faster** on some percentile | Possible: fewer, larger objects surviving can occasionally reduce collection frequency enough to win on a metric. Report it honestly, then check whether the win survives a longer run and three repetitions. Most such wins do not. |

### Step 4 — the second half of the drill

Now run a **third** arm that keeps the long-lived retention but **halves the allocation
rate** (drop `PER_REQ` to 32). Capture the same eight items.

The comparison you are building:

- Short arm vs long arm: **same allocation, different lifetime** → the cost is lifetime.
- Long arm vs half-rate long arm: **same lifetime, different allocation** → how little the
  allocation rate mattered.

If the second comparison shows a much smaller effect than the first, you have proved the
mechanical statement experimentally, on your own service, with your own numbers.

### What the drill proves

- Allocation is nearly free; **survival is the cost**.
- Promotion rate, not allocation rate, predicts old-gen pressure and concurrent-cycle
  frequency.
- GC is a process-global shared resource: one endpoint's retention degrades every other
  endpoint's latency, through a mechanism that appears in neither endpoint's own metrics.
- A dashboard showing "allocation rate" alone cannot distinguish a healthy service from
  one about to have an incident.

Carry one sentence out of this drill:

> *I made two workloads allocate identically and behave completely differently, and I can
> name the number that separates them: promotion rate.*

---

## Measurement

### The instrument for this topic

**The GC log, plus a percentile comparison against the recorded Topic 65 baseline.** Not a
microbenchmark. Not a stopwatch. Not `System.currentTimeMillis()` around a loop.

GC behaviour is an emergent property of allocation rate, live-set size, object lifetime
distribution, heap size, CPU quota and the collector's control loop. A microbenchmark
reproduces none of those.

### The two formulas — apply them to your own log

These are the numbers every conversation in Phases 8 and 9 starts from. **I am deliberately
not showing a worked example with numbers, because any numbers I invented would be fiction
and you would remember them.** Run these on your own log and do the arithmetic.

**Allocation rate.** For two *consecutive young collections* `n` and `n+1`:

```
allocated_between  =  heap_used_BEFORE_collection[n+1]  -  heap_used_AFTER_collection[n]
elapsed            =  uptime_at[n+1]  -  uptime_at[n]
allocation_rate    =  allocated_between / elapsed                    // MB per second
```

In words: the heap grew from "what was left after the last collection" to "what was there
when the next one started". That growth is what your application allocated in between.

**Promotion rate.** Using **old-generation** occupancy after each collection:

```
promoted_between   =  old_used_AFTER[n+1]  -  old_used_AFTER[n]
promotion_rate     =  promoted_between / elapsed                     // MB per second
```

Old-gen occupancy comes from the `gc+heap=debug` lines, **not** from the summary line. The
summary line's `before->after` is whole-heap.

**Extracting the fields:**

```bash
# Young collections: uptime, gc id, used-before, used-after, heap total.
grep 'Pause Young' /var/log/orderflow/gc.log \
  | sed -E 's/.*\[([0-9.]+)s\].*GC\(([0-9]+)\).*[^0-9]([0-9]+)M->([0-9]+)M\(([0-9]+)M\).*/\1 \2 \3 \4 \5/'

# Old-generation occupancy per collection, from the gc,heap debug lines.
grep -E 'gc,heap.*Old regions|gc,heap.*old' /var/log/orderflow/gc.log
```

**Sanity checks on the result** — do these every time, because both formulas have a
characteristic failure mode:

| Sanity check | If it fails |
|---|---|
| Did you mix **young** and **mixed/full** collections in one subtraction? | Mixed and full collections reclaim old gen, so the promotion delta goes negative. Compute promotion only across consecutive collections of the same kind, or across a window that starts and ends just after a mixed collection. |
| Did a concurrent cycle free memory in between? | Then `allocated_between` under-reports. Prefer intervals with no concurrent activity, or use many samples and take the mode rather than the mean. |
| Is your allocation rate implausibly high or low? | Compare against a second source: Micrometer's `jvm.gc.memory.allocated` counter, or JMH's `-prof gc` on a representative operation. Two independent methods agreeing is the standard. |
| Is your promotion rate negative over a long window? | Old gen shrank, which means a mixed or full collection ran. That is information, not an error. |

**WHAT TO LOOK FOR** once you have both numbers:

| Shape | What it means |
|---|---|
| High allocation rate, **flat** promotion rate | **Healthy.** Objects are dying young exactly as designed. A high allocation rate on its own is not a problem and does not need fixing. |
| Modest allocation rate, **rising** promotion rate | **The dangerous shape.** Your live set is growing. Expect old-gen pressure, then concurrent cycles, then a Full GC. The fix is retention (Topics 70, 79), not flags. |
| Both high and stable | You have a genuinely large working set. Size the heap from the measured live set plus headroom (Topic 82), and consider whether the working set is necessary. |
| Promotion rate spikes at a regular interval | Something periodic: a scheduled job, a cache refresh, a metrics scrape building a large intermediate. Correlate on the wall-clock timestamps. |

### Estimating the live set

The live set is the number you need for heap sizing, and no single log line gives it to you.

```
live_set  ≈  the FLOOR of old-generation occupancy immediately after full or mixed
             collections, sampled over a long run at steady state
```

Two independent ways to get it, and you should agree them:

```bash
# 1. From the log, over a long run.
grep -E 'Pause Full|Pause Young \(Mixed\)' /var/log/orderflow/gc.log | tail -50

# 2. Force a full collection and read the heap. NOTE: this pauses your service.
#    Never on production traffic; fine on a load-test instance.
jcmd <pid> GC.run
jcmd <pid> GC.heap_info
```

**WHAT TO LOOK FOR:** a floor that is roughly flat over tens of minutes. A floor that
climbs steadily over hours is a **leak** (Topic 79); a floor that is flat but large is a
**working set**, and the fix for those two is completely different.

### Why a naive `System.nanoTime()` measurement is WRONG here

You will be tempted to write this:

```java
// DO NOT DO THIS. Every number it produces is meaningless.
long t0 = System.nanoTime();
for (int i = 0; i < 10_000_000; i++) {
    Object o = new Object();
}
System.out.println((System.nanoTime() - t0) / 10_000_000.0 + " ns per allocation");
```

Five independent reasons it lies, and you cannot tell which one is dominating:

1. **Dead-code elimination.** `o` is never used. C2 can prove the allocation has no
   observable effect and delete it entirely. You measure an empty loop (Topic 75).
2. **Escape analysis and scalar replacement.** Even if the object is used, if it provably
   never escapes the method, C2 may never allocate it at all — its fields become registers.
   Your "allocation benchmark" then contains no allocations (Topic 75).
3. **On-stack replacement.** The loop starts interpreted and is compiled *while running*,
   then swapped mid-flight. Your average blends interpreted, C1 and C2 execution in a ratio
   that depends on the loop count you happened to pick (Topic 74).
4. **Cold JIT.** The first thousands of iterations run interpreted. For a short loop, that
   is most of your measurement.
5. **The collector is not in your loop.** This is the reason specific to *this* topic, and
   it is the important one. **The cost of memory management is not paid at allocation
   time.** It is paid at collection time, on GC threads, asynchronously. A timer around
   allocations measures the pointer bump — which is genuinely nearly free, which is exactly
   the point — and tells you nothing about the copying, promotion and old-gen work that
   your allocation *shape* causes twenty seconds later.

Topic 77 is the full treatment of why hand-rolled timing is wrong in general. The
generation-specific point is sharper: **even a perfectly correct JMH benchmark of
allocation would answer the wrong question.** The right instrument is a GC log from a
realistic run.

### Where JMH *is* the right tool here

Exactly one class of question: **how many bytes does one operation allocate?** JMH's
allocation profiler answers that directly and reliably.

```java
package com.orderflow.bench;

import org.openjdk.jmh.annotations.*;
import org.openjdk.jmh.infra.Blackhole;
import java.util.concurrent.TimeUnit;

@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.MICROSECONDS)
@Warmup(iterations = 5, time = 1)
@Measurement(iterations = 10, time = 1)
@Fork(3)
@State(Scope.Benchmark)
public class OrderLineMappingBenchmark {

    private Order order;

    @Setup(Level.Trial)
    public void setUp() { order = TestOrders.withLines(50); }

    @Benchmark
    public void mapToDtos(Blackhole bh) {
        bh.consume(OrderMapper.toDtos(order));       // consumed: cannot be eliminated
    }

    @Benchmark
    public void mapToProjection(Blackhole bh) {
        bh.consume(OrderMapper.toSummaries(order));  // the lower-allocation alternative
    }
}
```

```bash
java -jar target/benchmarks.jar OrderLineMappingBenchmark -prof gc -rf json -rff mapping.json
```

| Annotation / flag | What it defends against |
|---|---|
| `@State(Scope.Benchmark)` | Holds state in a field JMH controls, so C2 cannot constant-fold it |
| `Blackhole.consume(...)` | Defeats dead-code elimination — reason 1 above |
| `@Warmup(iterations = 5)` | Lets C2 compile and reach steady state before recording |
| `@Fork(3)` | Three separate JVMs; exposes run-to-run variance and defeats profile pollution (Topic 74) |
| `-prof gc` | **The important one.** Reports normalised bytes allocated per operation. This turns a timing benchmark into an allocation measurement. |

**WHAT TO LOOK FOR:** the bytes-per-operation figure, not the time. And report the
confidence interval JMH prints, never the point estimate — if two intervals overlap, you
measured nothing.

Then multiply: bytes-per-operation × operations-per-second from your baseline gives an
*independent* estimate of allocation rate, which you cross-check against the GC-log
formula. Two methods agreeing is how you earn the right to state a number.

### The three numbers to track continuously, in production

Not during a drill — permanently, on a dashboard (Topic 118 covers exporting them without
cardinality problems):

| Number | Where from | Why |
|---|---|---|
| Allocation rate (MB/s) | GC-log formula, or Micrometer `jvm.gc.memory.allocated` | The input to every capacity conversation |
| Promotion rate (MB/s) | GC-log formula, or Micrometer `jvm.gc.memory.promoted` | **The one that predicts trouble.** Rising promotion is the leading indicator. |
| Post-collection old-gen floor | `gc+heap=debug` after mixed/full collections | Your live set. Flat = working set; climbing = leak. |

Metaspace committed and RSS-minus-heap belong on the same dashboard, for Trap 2's reasons.

---

## Practice exercises

### 1 — Easy: derive the young-collection frequency before you measure it

Using only the Topic 65 baseline and one measurement:

1. Measure `orderflow`'s allocation rate under the baseline load, using the formula above.
2. Read your effective eden size: `jcmd <pid> GC.heap_info` (under G1, count the Eden
   regions and multiply by the region size).
3. **Predict** the interval between young collections: `eden_size / allocation_rate`.
4. Now measure the actual interval from the uptimes of consecutive `Pause Young` lines.

Report predicted versus actual, and then answer in one sentence each:

- If actual is much shorter than predicted, name two mechanisms that could explain it.
- If actual is much longer, name two more.
- You double `-Xmx`. Predict what happens to (a) collection *frequency*, (b) each
  collection's *duration*, and (c) total GC *overhead* as a percentage of wall time. Then
  measure it and say which prediction you got wrong.
- What single measurement would tell you whether doubling the heap was worth the memory it
  cost?

### 2 — Medium: the audit (combines Topics 01, 11, 12, 15, 18, 23, 48, 66)

This class runs on the `orderflow` order-read path at 80 rps. It contains **six** defects.
Three are allocation-and-lifetime defects from this topic; three are from earlier topics.
For each: name the topic, state the **observable** symptom in the GC log or in a latency
percentile, and write the fix.

```java
package com.orderflow.orders;

import java.util.*;
import java.util.stream.Collectors;

@Service
public class OrderSummaryService {

    /** "Avoids re-computing summaries." */
    private static final Map<Long, OrderSummary> CACHE = new HashMap<>();

    /** "Reuses buffers to reduce GC pressure." */
    private static final ThreadLocal<StringBuilder> BUFFER =
            ThreadLocal.withInitial(() -> new StringBuilder(1_048_576));

    private final OrderRepository orders;

    public OrderSummaryService(OrderRepository orders) { this.orders = orders; }

    public OrderSummary summarise(Long orderId) {
        OrderSummary hit = CACHE.get(orderId);
        if (hit != null) { return hit; }

        Order order = orders.findWithLines(orderId).orElseThrow();

        Map<Long, Long> unitsByProduct = new HashMap<>();
        for (OrderLine line : order.getLines()) {
            Long current = unitsByProduct.get(line.getProductId());
            unitsByProduct.put(line.getProductId(),
                    current == null ? line.getQuantity() : current + line.getQuantity());
        }

        String description = "";
        for (OrderLine line : order.getLines()) {
            description = description + line.getSku() + " x" + line.getQuantity() + "; ";
        }

        StringBuilder sb = BUFFER.get();
        sb.setLength(0);
        sb.append(description);

        List<Long> productIds = order.getLines().stream()
                .map(OrderLine::getProductId)
                .collect(Collectors.toList());

        OrderSummary summary = new OrderSummary(orderId, sb.toString(), productIds, unitsByProduct);
        CACHE.put(orderId, summary);
        return summary;
    }
}
```

Hints, in the order to think about them: one defect creates an unbounded live set and is
the classic Topic 79 shape; one pins a large buffer per thread forever, so its cost is
thread-count multiplied (and it is the exact "reuse to reduce GC pressure" instinct this
document argues against); one is a Topic 18 quadratic-allocation problem that will dominate
your allocation rate; one is a Topic 01 boxing problem in a counter map; one is a Topic 12
concurrency hazard that will silently corrupt the structure under load; and one loads full
entities where a projection would do, dragging Hibernate snapshots (Topic 48) into the
live set.

For each of the three lifetime defects specifically, predict — before you run it — whether
it raises **allocation rate**, **promotion rate**, or **both**. Then measure and report
where you were wrong.

### 3 — Hard: production simulation — build a memory model of `orderflow`

**Part A — measure the model's inputs.** Run the Topic 65 baseline and derive:

1. Allocation rate (MB/s), by the GC-log formula.
2. Promotion rate (MB/s), by the GC-log formula.
3. Live set (MB), from the post-collection old-gen floor over a 30-minute run.
4. Effective eden size and observed young-collection interval.
5. Metaspace committed, at steady state.
6. Code cache used, at steady state (`jcmd <pid> Compiler.codecache`).
7. Thread count and `-Xss`, giving total stack reservation.
8. RSS, and the NMT breakdown that explains RSS minus heap.

Cross-check (1) independently with a JMH `-prof gc` measurement of your dominant endpoint
multiplied by its request rate. Report both numbers and the discrepancy.

**Part B — build the model.** Write down, as arithmetic you can defend:

```
required_heap      =  live_set  ×  headroom_factor
required_footprint =  required_heap + metaspace + code_cache
                      + (threads × Xss) + direct_memory + gc_overhead + jvm_native
container_limit    =  required_footprint × safety_factor
```

State your chosen `headroom_factor` and `safety_factor` **and the reason for each**. A
number without a reason is a guess with a decimal point.

**Part C — falsify the model.** Deploy `orderflow` with the heap and container limit your
model produced. Re-run the baseline. Then, one at a time:

1. Halve the container limit and keep the JVM flags. Predict the failure mode before you
   run it, then run it and record what actually happened (`OutOfMemoryError`? OOM kill?
   thrashing? nothing?).
2. Double the heap without changing the container limit. Predict, then measure, the effect
   on collection frequency, pause duration, total GC overhead, and p99.
3. Triple the thread pool size. Predict, then measure, the effect on RSS and on the number
   of TLABs.

**Part D — the blind test.** Produce three GC logs — baseline, the pooling change from
Example 2, and an unbounded-cache change (borrowed from Topic 79's drill) — and strip the
labels. A week later, for each log, write down: allocation rate, promotion rate, whether
the live set is bounded, which of the three configurations it is, and the single command
you would run next. Check against your notes.

**Part E — argue against yourself.** You will conclude that object pooling is almost always
wrong. Make the strongest possible case for pooling in `orderflow` specifically — there is
a real one, and it involves an object that is expensive to *construct* rather than to
allocate. Name the object, name the measurement that would justify the pool, and name the
measurement that would kill it.

---

## Interview questions

### Q1 — "Is object allocation expensive in Java?"

**MID-LEVEL answer:** "It's not free — you're creating an object on the heap, and that adds
GC pressure. Best practice is to avoid unnecessary object creation in hot loops, and to
reuse objects where you can."

**SENIOR answer:** "Allocation itself is close to free, and the instinct to avoid it is
usually optimising the wrong axis.

Mechanically: each thread has a **thread-local allocation buffer** — a private slice of
eden. Allocating means comparing a pointer against the buffer's end, bumping it by the
object's aligned size, and writing the header. Roughly ten instructions, no lock, no CAS,
no free-list search. It's cheaper than `malloc`.

What actually costs is **survival**. A young collection copies the live objects out and
declares everything else free — dead objects are never visited. So the cost of a collection
is proportional to the **live set**, not to the garbage. An object that dies in eden costs
literally nothing to collect. One that survives gets copied on every collection until it
tenures, and once it's in old gen it participates in every marking cycle for as long as it
lives.

That inverts the usual advice in a specific and important way: **pooling objects to 'reduce
GC pressure' usually makes things worse.** You've converted free deaths into repeated
copies and then into permanent old-gen residents — and the dashboard you used to justify
the change, allocation rate, goes down, so it looks like it worked.

The version of the instinct that *is* correct: pool things that are expensive to
**construct** — connections, threads, compiled patterns, direct buffers. Never pool things
that are merely numerous.

Concretely, I'd quote two numbers from the GC log rather than one. Allocation rate and
promotion rate. High allocation with flat promotion is a healthy service. Modest allocation
with rising promotion is a service about to have an incident."

**What separates them:** the mid answer repeats received wisdom that is directionally
wrong. The senior answer gives the mechanism (TLAB, pointer bump), states the cost model
(live set, not garbage), derives the counter-intuitive consequence (pooling backfires),
gives the correct boundary condition (expensive to construct), and names the two numbers
that make it measurable. The detail that most reliably impresses is noticing that the
metric used to justify pooling improves while the service degrades.

**Interviewer's follow-up:** *"So allocation rate doesn't matter at all?"* — It matters as
the *denominator*: allocation rate determines how often eden fills, so it sets collection
frequency. It just doesn't determine each collection's cost. Halving allocation halves the
number of pauses; halving the live set shortens every pause. Which one to attack depends on
whether the log shows frequency or duration as the problem.

---

### Q2 — "We pooled our DTOs to reduce GC pressure and latency got worse. Why?"

**MID-LEVEL answer:** "Maybe the pool has synchronisation overhead, or it's not sized
correctly. I'd check for contention on the pool."

**SENIOR answer:** "Contention is a real secondary cost and worth checking, but the primary
cause is more fundamental: **you made the objects live.**

Before the pool, a DTO was allocated with a pointer bump and was unreachable by the end of
the request. If it never survived a young collection, its collection cost was exactly zero
— dead objects aren't visited. After the pool, every pooled instance is reachable from a
long-lived pool object, by construction. So on every young collection, each pooled DTO is
**live data that must be copied**. They survive to the tenuring threshold and get promoted,
and now they participate in every old-gen marking cycle forever.

There's a third cost people miss. A pool is an old-gen object holding references to objects
it hands out to young code, which creates **old-to-young references**. Those are tracked by
a write barrier — extra instructions the JIT emits after every reference store — and under
G1 they populate remembered sets that have to be maintained and scanned. So you've added a
per-store tax on your application threads too.

The reason it survives code review is that the metric improves. Allocation rate genuinely
falls, which is what the change claimed to do. The metrics that get worse are promotion
rate, young-pause duration and old-gen occupancy — and if you're only watching allocation
rate, you'll conclude the regression came from somewhere else.

How I'd confirm it: derive allocation rate and promotion rate from consecutive young
collections in the GC log, with and without the pool, under an identical load profile. And
look at the age distribution — `-Xlog:gc+age=trace`. The pool shows up as a fat tail
exactly where the pool size predicts.

Then I'd remove the pool, and if allocation genuinely is a problem, attack it at the source
— fetch fewer columns with a projection so we build fewer and smaller objects, rather than
recycling the ones we build."

**What separates them:** the mid answer looks for a bug in the pool. The senior answer
identifies that the *pool itself is the bug*, gives three distinct mechanisms including the
write barrier, explains why the change passed review, and names the specific log evidence.

**Interviewer's follow-up:** *"Is pooling ever right?"* — Yes, when construction is
expensive rather than allocation: database connections, threads, compiled `Pattern`s,
`Cipher` instances, large direct buffers whose native memory must be reserved. The test is
"does creating this do work beyond the pointer bump?" — not "do we create a lot of them?"

---

### Q3 — "We set `-Xmx` to the container's memory limit and the pod keeps getting OOM-killed. Explain."

**MID-LEVEL answer:** "The heap is filling up. We probably have a memory leak, or we need a
bigger container. I'd take a heap dump and look for what's growing."

**SENIOR answer:** "The symptom rules out a heap problem before I look at anything. An
OOM **kill** is the kernel's cgroup enforcement — exit code 137, pod status OOMKilled — and
it produces no heap dump, no `OutOfMemoryError`, and nothing in the application log. If the
heap were the problem, the JVM would have thrown `OutOfMemoryError: Java heap space` and I'd
have a dump. Getting killed instead means the *process* exceeded the limit, not the heap.

`-Xmx` bounds the Java heap. It bounds nothing else. The process also commits metaspace and
compressed class space for class metadata, the code cache for JIT output, one stack per
thread at `-Xss` each, direct byte buffers, GC-internal structures like remembered sets and
mark bitmaps, and the JVM's own native allocations plus glibc malloc arenas. Set `-Xmx` to
the whole container limit and the first time any of those grows — and they all grow as the
service warms up and thread count rises — you're over the limit.

The characteristic signature is exactly what they described: it correlates with traffic
rather than with heap growth, the heap graph looks healthy right up to the kill, and it
happens minutes after startup rather than at a memory high-water mark.

What I'd do. First, `-XX:NativeMemoryTracking=summary` and `jcmd VM.native_memory summary`
to attribute the gap between heap and RSS, rather than guessing which region it is. Then
use `-XX:MaxRAMPercentage` instead of `-Xmx` so the JVM reads the **cgroup** limit rather
than the host's memory — and raise it well above the 25% default, which is far too
conservative for a container that exists to run one JVM. Then explicitly cap the regions
that are unbounded by default: `MaxMetaspaceSize`, `ReservedCodeCacheSize`,
`MaxDirectMemorySize`, and a sensible `-Xss`.

That last part is the bit I'd argue for hardest. Capping metaspace and direct memory
doesn't prevent the bug — it converts an undiagnosable OOM kill into an `OutOfMemoryError`
with a heap dump and a stack trace. That trade is almost always worth making in a
container."

**What separates them:** reading the *kill* as evidence that excludes the heap; enumerating
the actual footprint components; knowing `MaxRAMPercentage` is cgroup-aware and that its
default is wrong for containers; and the judgment call about capping regions to convert an
invisible failure into a diagnosable one.

**Interviewer's follow-up:** *"What percentage would you set?"* — I'd measure rather than
pick. Native memory tracking gives the non-heap total for our actual workload; heap comes
from the measured live set plus headroom; the container limit is their sum plus a safety
factor. If forced to start somewhere before measuring, something in the 50–70% range is a
common starting point for a service-shaped JVM — but it's a starting point to be measured
away from, not an answer.

---

### Q4 — "Walk me through what happens to an object from `new` to being collected."

**MID-LEVEL answer:** "It gets created on the heap in the young generation. When the young
generation fills up, GC runs and removes anything that's not referenced. If the object
survives long enough it moves to old generation, which is collected less often."

**SENIOR answer:** "I'll do it in the JVM's own vocabulary, because the details are where
the tuning decisions live.

**Allocation.** The thread checks its **TLAB** — a private slice of eden. If there's room,
it bumps a pointer by the object's 8-byte-aligned size and writes the mark word and class
word. About ten instructions, no synchronisation. If there isn't room, either it retires
the TLAB and takes a new one — wasting the remainder, deliberately, to keep the fast path
lock-free — or, for a large object, it allocates directly in eden with an atomic bump of
the shared top pointer.

**First collection.** Eden fills; a young collection is triggered and stops the world. The
collector traces from **GC roots** — thread stacks, statics, JNI handles, thread-locals —
and **copies** every live object out of eden and the in-use survivor space into the other
survivor space, incrementing each object's age. Then it declares the source spaces free.
Dead objects are never visited, which is why collection cost tracks the live set rather
than the garbage.

**Ageing.** Each subsequent young collection copies the object again and increments its
age. The age lives in a few bits of the object header, which is why the maximum tenuring
threshold is 15.

**Promotion.** The object is promoted to old gen either when its age reaches the tenuring
threshold, or — and this is the case that surprises people — when the target survivor space
can't hold everything that survived. That second path is **premature promotion**, and you
diagnose it from `-Xlog:gc+age=trace`: if the reported effective threshold is well below
the maximum and the age distribution is thin, you're promoting for lack of space, not
because objects are old.

**Old gen.** Under G1 it's reclaimed by a concurrent marking cycle followed by mixed
collections. A promoted object also costs you on an ongoing basis: it participates in every
marking cycle, and any reference from it into the young generation costs write-barrier work
on your application threads.

**Death.** 'Collected' just means the collector observed it was unreachable at some
collection. There's no destructor and no deterministic moment. And unreachable isn't the
same as unused — a live root chain to a stale object is a leak, which is Topic 70's point.

Two things I'd add that aren't in the standard answer. Some objects never go through this
at all: if C2 can prove an object doesn't escape its method, it's **scalar-replaced** and
never allocated. And large objects can skip the young generation entirely — under G1,
anything at least half a region is **humongous** and is allocated directly into old gen."

**What separates them:** the vocabulary (TLAB, mark word, ages, tenuring threshold,
survivor overflow), the cost model, the diagnostic for premature promotion, and the two
exceptions — scalar replacement and humongous allocation — that show the mental model is a
real one rather than a memorised diagram.

**Interviewer's follow-up:** *"What if the survivor space is too small?"* — Premature
promotion: objects that would have died at age 3 end up in old gen, so old gen fills with
things that were about to be garbage, and you pay a concurrent cycle to clean up what a
young collection would have handled free. I'd confirm from the age table before touching
any flag, and the real fix is usually to reduce what survives, not to resize a space.

---

### Q5 — "You get `OutOfMemoryError: Metaspace`. What do you do?"

**MID-LEVEL answer:** "That's a memory error, so I'd increase the heap with `-Xmx` and
restart. If it keeps happening I'd look for a leak."

**SENIOR answer:** "`-Xmx` cannot help — it doesn't bound metaspace at all, and raising it
makes the process footprint larger, which brings a container OOM kill closer. So the reflex
is not just ineffective, it's actively harmful.

Metaspace is **native** memory holding class metadata: `Klass` structures, method bytecode,
runtime constant pools. It's allocated in chunks **per class loader**, and it's only
reclaimed when an entire class loader becomes unreachable — not per class.

So the question is always the same: are we loading more classes than we expect, or failing
to unload the ones we should?

Diagnosis: `jcmd <pid> VM.metaspace` gives totals and a per-classloader breakdown —
that breakdown usually names the culprit immediately. `jcmd <pid> VM.classloaders` shows
whether the loader count is growing over time, which is the leak signature.
`-Xlog:class+load=info` shows what's being loaded and `-Xlog:class+unload=info` shows what
is — or isn't — being unloaded. Zero unload lines over a long run with heavy dynamic class
generation is a finding.

The usual causes: repeated redeployments into a long-lived container; a library generating
a class per instance rather than reusing one, which some serialization, mocking and
expression libraries do; a `ThreadLocal` on a pooled thread retaining an object from a
discarded loader, so the loader can never be collected; or just genuinely more classes than
the default sizing anticipated.

Fixes in order: fix the loader leak if there is one — that's Topic 79's dominator-tree
investigation with a `ClassLoader` as the root. If it's genuinely a large application,
raise `MaxMetaspaceSize` to a measured value. And if you're generating classes at runtime,
find the reuse knob.

The operational point I'd make regardless: in a container I set `MaxMetaspaceSize`
explicitly, even though the default is unlimited. An unbounded metaspace turns a class
loader leak into an OOM kill with no dump and no error. Capping it turns the same bug into
an `OutOfMemoryError` I can actually diagnose. I'd rather have a diagnosable failure than a
slightly later one."

**What separates them:** knowing immediately that `-Xmx` is the wrong region and saying
*why* it's harmful; the per-loader reclamation rule; naming three specific `jcmd`
subcommands; the realistic causes; and the deliberate decision to cap an unlimited default
to make failures diagnosable.

**Interviewer's follow-up:** *"What's compressed class space and why is there a separate
error for it?"* — When compressed class pointers are on, `Klass` pointers are stored as
narrow offsets into a dedicated reserved region, sized by `-XX:CompressedClassSpaceSize`. It
can be exhausted independently of the rest of metaspace, so it has its own message. That
also ties into Topic 69: the class word in every object header is narrow precisely because
that region exists.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. A young collection copies live objects and never visits dead ones. Derive from that one
   fact why doubling your rate of short-lived allocation costs almost nothing, while
   doubling your live set costs on **every** collection. Then state what each of those two
   changes does to collection *frequency* versus collection *duration*.

2. TLABs make allocation lock-free by giving each thread a private slice of eden. Derive
   two distinct costs of that design. Then say what happens to both costs when you go from
   16 threads to 2,000 (Topic 98 is the thread-count story; Topic 101 makes it 2,000,000).

3. Pooling an object makes it live. Construct the general rule that separates objects worth
   pooling from objects that must not be pooled, in one sentence, without using the words
   "expensive" or "cheap".

4. Premature promotion tenures objects because there is nowhere to put them, not because
   they are old. Design a diagnostic that distinguishes premature promotion from
   promotion-by-age using only the GC log. Then say which one raising the heap fixes and
   which one it does not.

5. Metaspace is reclaimed per class loader, not per class. Explain why that design choice
   makes class unloading nearly all-or-nothing, and construct a scenario where one small
   retained object prevents tens of megabytes of metaspace from being freed.

6. `-Xmx` bounds the heap; the container limit bounds the process. Given a fixed container
   limit, describe the trade-off curve as you raise `-Xmx` from very low to very high. There
   is a failure at each end. Name both, and name the observable that tells you which end you
   are approaching.

7. You are told to reduce `orderflow`'s p99 by 30%, and you may change allocation
   behaviour, the live set, the heap size, or the collector. Rank those by expected effect
   **and** by risk, defend the ordering, and say what evidence would change your ranking.

---

## Quick reference card

### JVM flags — heap, generations, TLABs, metaspace

| Flag | Default | What it does | Set it? |
|---|---|---|---|
| `-Xms` / `-Xmx` | derived from `MaxRAMPercentage` | Initial / maximum heap | **Yes, and set them equal** — avoids resizing pauses and makes sizing deterministic |
| `-XX:MaxRAMPercentage` | 25 | Max heap as a % of the **cgroup** limit | **Yes in containers** — 25% is far too low for a dedicated JVM (Topic 82) |
| `-XX:InitialRAMPercentage` | varies | Initial heap as a % of available memory | With `MaxRAMPercentage`, as the container-native `-Xms` |
| `-Xmn` / `-XX:NewSize` / `-XX:MaxNewSize` | derived | Young generation size | **Almost never** — and under G1 it fights the pause-goal control loop (Trap 3) |
| `-XX:NewRatio` | 2 | old:young size ratio | Serial/Parallel vocabulary; verify G1 honours it before believing it does |
| `-XX:SurvivorRatio` | 8 | eden:survivor ratio | Rarely, and only from an age-table diagnosis |
| `-XX:MaxTenuringThreshold` | 15 | Maximum age before promotion | Rarely — it is a max, and the effective value adapts |
| `-XX:TargetSurvivorRatio` | 50 | Target survivor-space occupancy driving adaptation | Almost never |
| `-XX:+UseTLAB` | on | Thread-local allocation buffers | Leave on. `-XX:-UseTLAB` is an A/B control in a drill, never production. |
| `-XX:TLABSize` | 0 (adaptive) | Fixed TLAB size | Almost never |
| `-XX:+ResizeTLAB` | on | Adapt TLAB size per thread at each collection | Leave on |
| `-XX:MaxMetaspaceSize` | effectively unlimited | Caps class metadata | **Yes in containers** — converts an OOM kill into a diagnosable `OutOfMemoryError` |
| `-XX:MetaspaceSize` | small, adaptive | High-water mark that triggers a collection to unload classes | Rarely; raise it if you see frequent `Metadata GC Threshold` collections at startup |
| `-XX:CompressedClassSpaceSize` | commonly 1 GB reserved | The narrow-`Klass`-pointer region | Only after seeing its specific OOM |
| `-XX:ReservedCodeCacheSize` | version-dependent | JIT output (Topic 74) | Cap it in a container so it is in your footprint model |
| `-XX:MaxDirectMemorySize` | approximately `-Xmx` if unset | Direct byte buffers (Topic 80) | **Yes in containers** — the bound most people never set |
| `-Xss` | platform-dependent, often ~1 MB | Per-thread stack | Yes when thread counts are high; the cost is per thread |
| `-XX:NativeMemoryTracking=summary` | off | Enables `jcmd VM.native_memory` | Yes while investigating footprint; small overhead |
| `-XX:+HeapDumpOnOutOfMemoryError` | off | Dump on heap OOM | **Always, in production** (Topic 79) |
| `-XX:PretenureSizeThreshold` | 0 | Allocate objects above this size directly in old gen | Serial/Parallel only; **G1 ignores it** and uses the humongous rule instead (Topic 71) |

> **Version note, one line:** the generational model, TLAB mechanics and metaspace design
> are unchanged between JDK 21 and 25; what has moved across versions is the *default*
> collector selection heuristics, some metaspace and code-cache defaults, and which of the
> Serial/Parallel sizing flags G1 honours. **Settle every default on your runtime** with
> `java -XX:+PrintFlagsFinal -version` or `jcmd <pid> VM.flags -all` rather than trusting
> any document, including this one.

### Diagnostic commands

```bash
# What did the JVM actually choose?
java -XX:+PrintFlagsFinal -version | grep -E 'HeapSize|NewSize|SurvivorRatio|Tenuring|TLAB|Metaspace|UseG1GC|UseSerialGC'
jcmd <pid> VM.flags -all

# Heap and generation breakdown, right now.
jcmd <pid> GC.heap_info

# Metaspace, with the per-classloader breakdown that names a leak.
jcmd <pid> VM.metaspace
jcmd <pid> VM.classloaders

# Code cache (Topic 74) and native footprint (Topic 80).
jcmd <pid> Compiler.codecache
jcmd <pid> VM.native_memory summary          # needs -XX:NativeMemoryTracking=summary

# The logs that matter for this topic.
-Xlog:gc*,gc+heap=debug:file=gc.log:time,uptime,level,tags:filecount=5,filesize=50m
-Xlog:gc+age=trace:file=age.log:time,uptime,level,tags       # the age distribution
-Xlog:gc+tlab=trace:file=tlab.log                            # verify with java -Xlog:help first

# Force a collection and read the floor. NEVER on production traffic.
jcmd <pid> GC.run && jcmd <pid> GC.heap_info

# Triage.
grep -c 'Pause Young'  gc.log
grep -c 'Pause Full'   gc.log
grep    'Metadata GC Threshold' gc.log        # a GC caused by CLASS LOADING, not the heap
grep -A12 'Desired survivor size' gc.log | tail -40
```

### The two formulas — write these on a card

```
allocation_rate = (used_BEFORE[n+1] - used_AFTER[n]) / (uptime[n+1] - uptime[n])
promotion_rate  = (old_AFTER[n+1]   - old_AFTER[n])  / (uptime[n+1] - uptime[n])
live_set        ≈ the FLOOR of old-gen occupancy after mixed/full collections, over a long run
```

Consecutive **young** collections only. Old-gen figures come from `gc+heap=debug`, not the
summary line.

### Gotchas checklist

- [ ] Allocation is a pointer bump. **Retention** is the cost.
- [ ] Never pool what is merely numerous. Pool what is expensive to **construct**.
- [ ] Quote **two** numbers: allocation rate *and* promotion rate. One alone is not a diagnosis.
- [ ] `-Xmx` is a budget line item, not the budget. Use `MaxRAMPercentage` in containers.
- [ ] Cap metaspace, code cache and direct memory so failures are diagnosable, not fatal.
- [ ] `OutOfMemoryError: Metaspace` is not a heap problem. `-Xmx` cannot help.
- [ ] Heap-used at an instant is live data **plus** uncollected garbage. It is not the live set.
- [ ] Under G1, `-Xmn` / `NewRatio` / `SurvivorRatio` / `PretenureSizeThreshold` mostly do not do what the blog post said.
- [ ] An age table with a lowered effective threshold means **premature promotion**, not old objects.
- [ ] More threads means more TLABs means less usable eden — and more stacks in your footprint.
- [ ] A `Metadata GC Threshold` collection is class loading (Topic 67), not your heap.
- [ ] Verify the collector. In a small container the JVM may have chosen Serial GC.

---

## When would I use this at work?

**1. Reviewing a PR titled "reduce GC pressure by reusing objects".**
You ask two questions: is this object expensive to *construct*, or merely numerous? And
what happens to promotion rate? If the answers are "numerous" and "it goes up", you have
prevented a regression that would otherwise have been diagnosed as "the database got
slower", because the p99 damage lands on endpoints that never touch the pooled type. A
five-minute review comment that saves a week.

**2. Sizing a container before the service exists.**
Product asks whether `orderflow` fits in a 1 GiB pod. You do not guess. You measure the
live set from the post-collection floor, add headroom, add metaspace, add the code cache,
add threads times `-Xss`, add direct memory, and produce an arithmetic answer with each
term justified. Then you say what would have to change for the answer to flip. That is the
capacity-model artefact Phase 12 asks for, in miniature.

**3. Triaging a p99 regression with two numbers instead of an argument.**
Someone says "GC is bad, let's tune it". You pull allocation rate and promotion rate from
the log before and after the suspect release. If allocation rose and promotion did not, GC
is exonerated in one step and you have saved a day of flag-fiddling. If promotion rose, you
have the *direction* of the fix — retention, not configuration — before anyone has proposed
a flag. Being the person who arrives with two derived numbers rather than an opinion is
most of what "senior" means in this conversation.

---

## Connected topics

**Prerequisites:**

- **01 — Boxing and the `Integer` cache**: every boxed value is a heap allocation. A
  `Map<Long, Long>` counter in a hot path is an allocation-rate contributor you can now
  quantify, and `List<Integer>` versus `int[]` is a live-set decision (Topic 69 gives the
  bytes).
- **11 — List implementations**: `ArrayList` growth reallocates and copies; `LinkedList`
  allocates a node per element, all of which become live data if the list is retained.
- **12 — HashMap internals**: every entry is a `Node` object, and a resize allocates a new
  table. A large long-lived map is a large long-lived *object graph*, not one object.
- **15 — LinkedHashMap and LRU**: a bounded cache is a **managed** live set. That is the
  correct shape of "keep things alive on purpose".
- **18 — Strings**: `+` in a loop is the classic allocation-rate defect; `intern()` moves
  strings into a structure that keeps them alive, converting an allocation question into a
  retention question.
- **48 — Hibernate persistence context**: the dirty-checking snapshot roughly doubles the
  memory per loaded entity and keeps it alive for the transaction's duration. Loading
  100,000 entities to update one field is a live-set decision, and now you can price it.
- **66 — JVM architecture**: the runtime data areas. This document is the detailed treatment
  of two of them — the heap and metaspace — plus the allocation mechanism.
- **67 — Class loading**: what fills metaspace, and why it is reclaimed per class loader.

**This unlocks:**

- **69 — Object layout and compressed oops**: how many bytes each of those objects actually
  occupies. You cannot convert "40,000 retained DTOs" into megabytes without it.
- **70 — GC fundamentals**: reachability and roots — *why* an object survives, as opposed to
  this document's *what it costs* when it does.
- **71 — G1 in depth**: the collector that implements everything here with regions, plus
  humongous allocation, which is the exception to "allocation is a pointer bump in eden".
- **72 — ZGC and Shenandoah**: what changes when the collector relocates objects
  concurrently, and why generational ZGC re-added a young generation.
- **73 — Safepoints**: every young collection is a safepoint operation. Its reported
  duration excludes the time to *reach* the safepoint.
- **74 — JIT**: the code cache is another non-heap region in your footprint, and the write
  barriers that make promotion costly are code C1 and C2 emit.
- **75 — Escape analysis**: the allocation that never happens. The cheapest possible
  allocation-rate reduction, and the most fragile.
- **77 — JMH**: the only correct way to measure bytes-per-operation, and the reason the
  naive allocation loop above is fiction.
- **78 — Profiling**: async-profiler in allocation mode names the *site*. The GC log tells
  you that allocation happened; the profile tells you where.
- **79 — Heap dumps and MAT**: when the live set grows without bound, the dominator tree
  names the retaining path. This document tells you the live set is the cost; that one tells
  you what is in it.
- **80 — Off-heap memory**: the rest of the footprint, and the tool (NMT) that attributes it.
- **82 — Containers and cgroups**: `MaxRAMPercentage`, CPU quota effects on GC thread counts,
  and the OOM-kill arithmetic Trap 2 introduced.
- **98 — Thread pools**: thread count drives TLAB count, stack reservation and root-scanning
  cost. Three separate memory effects from one configuration value.
- **101 — Virtual threads**: continuations are stored **on the heap**, so millions of virtual
  threads become a heap and live-set question rather than a stack-reservation one. That
  changes every number in your footprint model.

---

*Java baseline 21, running on JDK 25. Two things in this document are deliberately hedged
rather than asserted: exactly which Serial/Parallel sizing flags G1 honours on your JDK, and
the precise TLAB logging tags available in your build. Each has a command in the Hands-on
section that settles it on your machine in under a minute. No allocation rate, pause
duration, promotion figure or object size in this document was measured — every number you
act on must come from your own log against your own baseline. The two structural
illustrations (the heap diagram and the age table) are labelled as illustrations with
placeholder values. What has been stable for two decades and will still be true at 3am:
allocation is a pointer bump, collection cost tracks the live set, promotion is the
compounding cost, and `-Xmx` is not the process footprint.*
