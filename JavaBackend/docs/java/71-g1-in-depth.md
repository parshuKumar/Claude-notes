# 71 — G1 in Depth: Regions, Humongous Allocations, the Concurrent Cycle, and Reading `-Xlog:gc*`

## Phase: 8 — JVM Internals
## Category: ELITE
## Java baseline: 21  |  Notes features from: 21 / 25
## Project spine: this is how you name the cause of a `orderflow` p99 spike from evidence instead of guessing. Every claim in this document is checked against the Topic 65 baseline in `/docs/java/baselines/`.

---

## Mechanical statement

Read this three times. The rest of the document is an elaboration of it.

> **G1 divides the whole Java heap into equal-sized regions.** Not into a contiguous
> eden followed by a contiguous old gen — into a flat array of same-size blocks, each
> of which is *labelled* at any moment as Eden, Survivor, Old, Humongous, or Free.
> A region's label can change on every collection.
>
> **Each region carries a remembered set** — a record of "which locations outside this
> region point into it". That set is maintained by a **write barrier**: a few extra
> instructions the JIT emits after every reference-field store.
>
> **G1 collects a chosen subset of regions, called the collection set.** It chooses
> that subset to meet a *pause-time goal*, `-XX:MaxGCPauseMillis`. The goal is not a
> guarantee and not a limit. It is an input to a sizing decision. G1 meets the goal by
> **collecting fewer regions**, which means leaving more garbage behind.
>
> **Any single allocation of half a region or more is HUMONGOUS.** It does not go
> into a TLAB. It does not go into eden. It is placed in a run of **contiguous
> old-generation regions**, starting on a region boundary, with the tail of the last
> region wasted. The object is old the instant it is born.

Four consequences follow directly, and you should be able to derive each one:

1. A pause goal that is too aggressive produces **more** long pauses, not fewer.
2. A large `byte[]` in a hot path is an old-gen allocation with a fragmentation cost.
3. "GC pause" in the log is a number G1 chose to produce, not a number physics imposed.
4. The write barrier means **every reference store in your application costs G1
   something**, whether or not a collection ever happens.

---

## The bridge from what you know

### What transfers

You have a generational intuition already, even if nobody named it for you.

V8 has a generational collector. New objects go into a small "new space" (a semi-space
scavenger). Objects that survive two scavenges get promoted to "old space", which is
collected by a mark-compact collector that runs much less often and takes much longer.
The famous advice "don't create garbage in a hot loop" and the famous symptom "my Node
process pauses for 200 ms every so often" both come from that structure.

So this transfers cleanly:

- **Young objects mostly die young.** The weak generational hypothesis is true in both
  runtimes, and it is true for `orderflow` — a request allocates DTOs, Hibernate
  snapshots, `String`s, boxed `Long`s (Topic 01), and almost all of it is unreachable
  by the time the response is written.
- **Collecting a small young space is cheap because cost tracks the LIVE set, not the
  garbage.** Topic 70 established this. A scavenger copies survivors and then declares
  the whole space empty. Ten megabytes of dead objects cost nothing to "free".
- **Promotion is the expensive event.** In both runtimes, an object that survives long
  enough to be promoted starts costing you on every subsequent old-generation cycle.

Keep all of that. It is correct and it is load-bearing.

### NO TYPESCRIPT ANALOGUE.

Now stop transferring, because the next four things have no counterpart in Node at all,
and pretending otherwise will install a wrong model.

**1. Region-based heap layout has no analogue.**
V8's new space and old space are *contiguous address ranges* with fixed roles. G1's
heap is an array of interchangeable blocks whose roles are reassigned continuously.
There is no "the eden" in G1 — there is "the set of regions currently labelled Eden",
and that set is a different set after every collection. You cannot ask "how big is
eden" and get a fixed answer; you can only ask "how many regions are eden right now".

**2. Humongous allocation has no analogue.**
V8 has a large-object space, but it is invisible to you, untunable, and never something
you reason about at design time. In G1, the humongous threshold is a *number derived
from your heap size* that you can compute, that changes when you change `-Xmx`, and
that determines whether a particular `byte[]` in your code goes to eden or straight to
old gen. A JavaScript engineer has never had to think "is this array bigger than half a
region", because there are no regions and no threshold you can see.

**3. A pause-time GOAL has no analogue.**
You cannot tell V8 "please keep pauses under 50 ms". There is no such knob. There is no
control loop that adapts what it collects to hit a target you set. This is genuinely
new: **you now get to state a latency requirement to the collector, and the collector
will change its behaviour to try to satisfy you — including in ways that hurt you.**

**4. Collector CHOICE has no analogue.**
There is exactly one V8 GC. You do not choose it. You do not compare it against
alternatives. You do not have to know the trade-offs, because there is no trade to
make. Java gives you G1, Parallel, Serial, ZGC and Shenandoah, all in the same runtime,
all selectable with one flag — and Topic 72 makes you choose between them.

> **The theme of Topics 71 and 72, and you should hold it consciously:**
> **you have been promoted from a passenger to a driver.** Everything about memory
> management that was previously opaque and untunable is now visible, measurable and
> tunable. The corollary is the part people underestimate: **you can now get it
> wrong.** A Node engineer cannot misconfigure the GC. You can, and this document's
> Trap section is a catalogue of the specific ways.

| You know | G1 | Verdict |
|---|---|---|
| New space / old space, fixed contiguous ranges | Regions with mutable labels | **NO ANALOGUE** |
| "Objects die young" | Weak generational hypothesis | **HONEST ANALOGUE** |
| Large-object space (invisible) | Humongous regions (visible, computable, tunable) | **NO ANALOGUE** |
| `--max-old-space-size` | `-Xmx` / `-XX:MaxRAMPercentage` | **PARTIAL** — heap is only part of the JVM footprint (Topic 80) |
| — | `-XX:MaxGCPauseMillis` as a *goal* | **NO ANALOGUE** |
| — | Choosing a collector | **NO ANALOGUE** |
| GC log: nothing, unless you build Node with flags nobody uses | `-Xlog:gc*`, structured, always available | **NO ANALOGUE** |

---

## What is this?

G1 — "Garbage-First" — has been HotSpot's default collector on server-class machines
since JDK 9. "Server-class" is a real definition: at least 2 CPUs visible to the JVM
and at least 1792 MB of memory visible to the JVM. Below that, the JVM historically
selects Serial GC instead. **That threshold matters to you specifically**, because it
is exactly the situation Topic 82 drills: a container with `--cpus=0.5` may make the
JVM see one processor and silently pick a different collector than the one you assumed.

Never assume the collector. Print it:

```bash
java -XX:+PrintFlagsFinal -version | grep -E "UseG1GC|UseSerialGC|UseParallelGC|UseZGC|UseShenandoahGC"
```

or, on the running service:

```bash
jcmd <pid> VM.flags -all | grep -E "UseG1GC|UseSerialGC|UseParallelGC|UseZGC"
```

### The four things G1 does

**1. It divides the heap into regions.** Same size, typically between 1 MB and 32 MB,
chosen at startup so the heap contains roughly 2048 of them. Every region is one of:

| Label | Meaning |
|---|---|
| Free | unused, available to be relabelled |
| Eden | new allocations land here (via TLABs — Topic 68) |
| Survivor | survived at least one young collection, not yet promoted |
| Old | promoted, or allocated directly as humongous |
| Humongous (Starts / Continues) | holds one object ≥ half a region; accounted as Old |
| Archive (CDS) | class-data-sharing regions, mapped read-only, never collected |

**2. It collects young regions with an evacuating copy.** A young collection is
stop-the-world. G1 finds all live objects in the eden and survivor regions, copies them
into fresh survivor or old regions, and then marks the source regions Free. Objects are
*moved*. Nothing is compacted in place; everything survives by being copied elsewhere.

**3. It runs a concurrent marking cycle over the old generation** while the application
runs, so that it knows which old regions have the least live data.

**4. It then runs "mixed" collections** — young regions plus a handful of the emptiest
old regions — reclaiming old-generation space incrementally rather than in one giant
full GC. "Garbage-first" is the name of exactly this policy: collect the regions with
the most garbage first, because they give the most space back per unit of copying work.

### What a Full GC means in G1

A **Full GC** in G1 is the failure mode, not part of normal operation. It is a
single-threaded-by-default (multi-threaded since JDK 10) stop-the-world mark-sweep-
compact of the entire heap. If you see `Pause Full` in your log, something went wrong:
G1 could not keep up, or could not find contiguous space, or ran out of regions to
evacuate into. Treat every Full GC line as an incident to explain, never as noise.

---

## Why does it matter?

**1. Because the pause goal is the single most-misused JVM flag in existence.**

Every "JVM tuning" blog post says to set `-XX:MaxGCPauseMillis`. Almost none explain
that it is a goal G1 satisfies by doing *less work per pause*, and that doing less work
per pause means old regions accumulate faster than they are reclaimed. There is a knee
in this curve, and past the knee your p99 gets dramatically worse, not slightly worse.
You need to be able to look at a log and say "this pause was long because G1 could not
find space to copy into, and the reason it could not is that I told it to collect less".

**2. Because humongous allocation is invisible in every ordinary tool.**

Your APM shows heap usage. Your metrics show GC pause time. Neither shows you that a
128 KB response payload buffer in a hot endpoint is being allocated directly into old
gen, wasting the tail of a region each time, and driving concurrent cycles that would
otherwise never run. The only place this is visible is `-Xlog:gc+heap=debug` and
`gc+humongous=debug`. If you do not know to look, you will not find it.

**3. Because "GC pause" is not "stop-the-world duration".**

This is Topic 73's subject, and it is important enough to state twice. The number after
`Pause Young` is the time G1 spent doing GC work. It does not include the time spent
waiting for the last application thread to reach a safepoint. Those are two different
numbers with two different causes and two different fixes. Reading one as the other has
sent entire teams chasing GC tuning for a problem that was a counted `int` loop.

**4. Because on `orderflow` you have a latency budget and a recorded baseline.**

You have p50/p95/p99/p999 committed in `/docs/java/baselines/`. That means every claim
you make about GC is falsifiable. That is the whole reason Phase 8 sits after the
Topic 65 gate. You are not learning G1 in the abstract. You are learning to answer
"why did `POST /orders` p99 go from its baseline value to five times that between
14:02 and 14:04", with evidence.

---

## Machine-level reality

### Region sizing — the actual arithmetic

At JVM startup, if you have not set `-XX:G1HeapRegionSize`, G1 computes a region size
by targeting approximately 2048 regions across the heap, then **rounding down to a
power of two** and clamping to the allowed range.

Roughly: `regionSize = clamp(roundDownToPowerOfTwo(maxHeapSize / 2048), min, max)`.

The consequence is a step function, not a smooth curve:

| `-Xmx` | Heap / 2048 | Region size G1 will typically pick | Humongous threshold (half a region) |
|---|---|---|---|
| 512 MB | 256 KB | 1 MB (the floor) | 512 KB |
| 1 GB | 512 KB | 1 MB | 512 KB |
| 2 GB | 1 MB | 1 MB | 512 KB |
| 4 GB | 2 MB | 2 MB | 1 MB |
| 8 GB | 4 MB | 4 MB | 2 MB |
| 16 GB | 8 MB | 8 MB | 4 MB |
| 32 GB | 16 MB | 16 MB | 8 MB |
| 64 GB | 32 MB | 32 MB | 16 MB |

**Do not memorise this table. Print the real value.**

```bash
java -Xmx4g -XX:+PrintFlagsFinal -version | grep -i G1HeapRegionSize
```

The single most important operational fact in this table: **the humongous threshold
moves when you change `-Xmx`.** A 700 KB allocation is humongous on a 2 GB heap and
ordinary on an 8 GB heap. Which means a service that behaved perfectly in staging with
`-Xmx8g` can start doing humongous allocation in production with `-Xmx2g`, from
identical code. This is a real production failure shape and almost nobody predicts it.

> **Version note, flagged honestly:** the *maximum* permitted region size has been
> raised in newer JDKs (the historical cap was 32 MB; later builds allow larger, which
> matters only for very large heaps). **I am not certain of the exact JDK in which the
> higher cap landed, and I will not guess.** Settle it on your runtime with
> `java -XX:+PrintFlagsFinal -version | grep -i G1HeapRegionSize` and by checking
> whether an explicit larger `-XX:G1HeapRegionSize=64m` is accepted or warned about.
> Nothing else in this document depends on the answer.

### Humongous allocation — exactly what happens

An allocation request is humongous when its **total size in bytes, including the object
header and array-length word and 8-byte alignment padding (Topic 69), is ≥ 50% of a
region.**

When G1 sees one:

1. It finds a run of **contiguous free regions** large enough to hold the object.
2. It labels the first one **Humongous Starts (HS)** and the rest **Humongous
   Continues (HC)**.
3. It places the object at the start of the first region.
4. **The remaining space in the last region is wasted.** A 5 MB object on a 4 MB region
   size occupies two regions = 8 MB, and 3 MB of that is unusable by anything else.
5. The regions are accounted as **old generation**. The object never sees eden, never
   sees a survivor space, never gets an age, and cannot be collected by an ordinary
   young collection unless eager reclaim applies.

Three sharp consequences:

- **Contiguity is required.** If the heap has 400 MB free but no run of two adjacent
  free regions, a 5 MB humongous allocation fails and triggers a collection — possibly
  a Full GC to compact. Fragmentation, which G1's evacuating young collector normally
  makes a non-issue, comes back specifically for humongous objects.
- **`byte[]` of "about a megabyte" is the classic offender.** Serialisation buffers,
  file uploads, image payloads, gzip buffers, a `ByteArrayOutputStream` that grew, a
  JDBC driver's fetch buffer, a Kafka record batch. All of these are commonly sized in
  the hundreds of kilobytes to low megabytes — exactly the humongous zone on a small
  heap.
- **Historically, humongous regions were only freed at a concurrent cycle.** Modern G1
  has **eager reclaim**: during a young collection, G1 can reclaim a humongous region
  whose object it can cheaply prove is unreachable — in practice, objects with no
  incoming references recorded in remembered sets, which covers the common case of a
  `byte[]` referenced only from a stack local or a young object. The flag is
  `-XX:+G1EagerReclaimHumongousObjects` and it is on by default.

  **Do not assume eager reclaim will save you.** It applies to a restricted set of
  cases, and one incoming old-gen reference disqualifies the object. Confirm what your
  JVM actually did with `-Xlog:gc+humongous=debug` — the drill below shows exactly
  where to look.

### Remembered sets and the write barrier — what your application pays

G1 must be able to collect *some* regions without scanning the whole heap. To do that
it needs to know, for each region it plans to collect, every reference into that region
from anywhere else. That is what a **remembered set** (RSet) is.

The RSet is populated by a **write barrier**: extra instructions the JIT compiler emits
after every store of a reference into an object field. Conceptually:

```java
// what you wrote
order.setPayment(payment);

// what compiled code effectively does
order.payment = payment;                 // the store itself
// --- G1 post-write barrier, emitted by C1/C2 ---
// if (regionOf(order) != regionOf(payment) && payment != null) {
//     mark the card covering &order.payment as dirty;
//     if the card was not already dirty, enqueue it for the refinement threads;
// }
```

Three costs fall out of this, and each is measurable:

1. **Instruction cost on the mutator.** Every reference store in your application is a
   handful of extra instructions. This is why G1's raw throughput is lower than
   Parallel GC's for batch workloads: Parallel's barrier is far cheaper.
2. **Concurrent refinement threads.** Dirty cards are drained by background threads
   (`-XX:G1ConcRefinementThreads`) that update the actual RSets. Those threads burn CPU
   your application also wants — and in a container with a small CPU quota, that is a
   direct throughput tax (Topic 82).
3. **Memory.** RSets are a real data structure with a real footprint, which can be a
   meaningful fraction of the heap for reference-dense graphs. `-Xlog:gc+remset=debug`
   reports on them.

There is a **second** barrier — the SATB (snapshot-at-the-beginning) **pre**-write
barrier — that is active only during a concurrent marking cycle. It records the
*previous* value of a reference field before it is overwritten, so concurrent marking
sees a consistent snapshot of the graph as it was when marking started. The practical
consequence: **an object that becomes garbage after concurrent marking begins is not
reclaimed in that cycle.** It is "floating garbage" and waits for the next cycle. This
is why a service under a sudden allocation spike can show old-gen occupancy that does
not drop as much as you expected after a cycle completes.

### The concurrent cycle, phase by phase

G1 starts a concurrent cycle when old-generation occupancy crosses the **Initiating
Heap Occupancy Percent** threshold. `-XX:InitiatingHeapOccupancyPercent` defaults to
45, but by default G1 **adapts** it at runtime (`-XX:+G1UseAdaptiveIHOP`) based on
observed allocation and marking rates. So the effective trigger point is not 45% and is
not fixed; it is a control loop. Setting IHOP explicitly disables the adaptation.

The phases, in order, with which ones stop the world:

| Phase | STW? | What happens |
|---|---|---|
| **Concurrent Start** | **yes** — piggybacked on a young collection | Marks the roots. You will see this in the log as a young pause tagged `(Concurrent Start)`. |
| Concurrent Scan Root Regions | no | Scans survivor regions for references into old gen. **Must finish before the next young collection**, or the young collection waits. |
| Concurrent Mark | no | Traces the object graph. SATB barrier is active. |
| **Pause Remark** | **yes** | Finishes marking, drains SATB buffers, does reference processing and class unloading. Usually the longest STW phase of the cycle. |
| Concurrent Cleanup / **Pause Cleanup** | brief STW + concurrent | Computes per-region liveness, frees regions that are 100% garbage immediately, and sorts the rest by liveness for the mixed-collection candidate list. |
| **Mixed collections** | **yes**, several | Ordinary young collections that additionally include some old-gen candidate regions. Continues until the candidate list is exhausted or old-gen occupancy falls below a threshold. |

Two failure shapes come directly out of that table:

- **The cycle loses the race.** If your allocation rate is high enough that old gen
  fills before concurrent marking finishes, G1 has no choice: it does a Full GC. The
  log shows `Pause Full (G1 Compaction Pause)` or a concurrent-mode-related message.
  Fixes: bigger heap, lower IHOP so the cycle starts earlier, more concurrent GC
  threads, or — the real fix — allocate less.
- **Root region scan blocks the next young collection.** If your survivor spaces are
  large and the scan is slow, a young collection that would have taken a few
  milliseconds waits for it. This shows up as an unusually long young pause immediately
  after a `Concurrent Start`.

### Evacuation failure / to-space exhaustion

This is the mechanism behind the most important trap in the document, so understand it
precisely.

A young collection works by **copying** live objects out of the collection-set regions
into other regions. To do that, G1 needs somewhere to copy *to*: free regions, or space
in survivor/old regions.

If, partway through evacuation, G1 runs out of space to copy into:

1. It cannot abort — some objects have already been moved and their references updated.
2. It **fails the evacuation** for the remaining objects: they stay where they are, and
   their regions are relabelled Old in place, without compaction.
3. It has to fix up the object headers of everything it half-moved.
4. The pause becomes dramatically longer than the goal, because none of this work was
   in the plan.
5. Very often the next thing that happens is a **Full GC**, because the heap is now
   both full and fragmented.

In the log this appears as `Evacuation Failure`, `To-space Exhausted`, or a
`Pause Young (Normal) (G1 Evacuation Pause)` with an unexpectedly long duration
followed by `Pause Full`. `-Xlog:gc+heap=debug` shows the region counts that explain it.

**This is the mechanism by which an aggressive pause goal makes latency worse.** Small
goal → small collection set → less garbage reclaimed per pause → free regions decline →
eventually no to-space → evacuation failure → Full GC → a pause an order of magnitude
larger than the one you were trying to avoid.

### Where the pause time actually goes

A young pause is not one thing. The `gc+phases=debug` tag breaks it down. The parts
that matter:

| Sub-phase | What it is | What makes it long |
|---|---|---|
| Ext Root Scanning | Scanning thread stacks, statics, JNI handles | Very many threads (Topic 98), huge static structures |
| Update RS / Scan RS | Processing dirty cards and remembered sets | High reference-mutation rate; refinement threads falling behind |
| Object Copy | The actual evacuation | **A large live set** — this is the term Topic 70 told you dominates |
| Termination | Worker threads finishing and load-balancing | Too few or too many GC threads for the CPU quota |
| Reference Processing | Soft/weak/phantom references, finalizers | Caches built on `WeakReference`, `DirectByteBuffer` cleaners (Topic 80) |

Notice that **Object Copy scales with the live set, not with the amount of garbage**.
That is the mechanical restatement of Topic 70's central claim. If your young pauses
are growing, the first question is "did the surviving-object volume grow", not "did
allocation grow".

---

## Example 1 — minimal

The smallest program that makes humongous allocation visible. No framework, no Spring,
no database. One knob: the size of the array.

```java
package com.orderflow.lab.gc;

/**
 * Allocates arrays of a size given on the command line, forever, discarding them.
 *
 * Run it twice: once with a size safely below half a region, once just above.
 * The GC log tells you which one went to eden and which went straight to old gen.
 */
public final class HumongousProbe {

    public static void main(String[] args) throws Exception {
        int arrayBytes = Integer.parseInt(args[0]);       // e.g. 262144 or 600000
        int iterations = Integer.parseInt(args[1]);       // e.g. 200000

        // A field, so the JIT cannot prove the allocation is dead and remove it.
        // Topic 75 explains exactly why this line is necessary.
        for (int i = 0; i < iterations; i++) {
            sink = new byte[arrayBytes];
            if ((i & 0xFFF) == 0) {
                Thread.onSpinWait();
            }
        }
        System.out.println("done, last length = " + sink.length);
    }

    /** Static, therefore a GC root while it holds a value. Defeats scalar replacement. */
    private static byte[] sink;
}
```

Compile and run it with an explicitly pinned region size so the arithmetic is not a
mystery:

```bash
javac -d out HumongousProbe.java

# Region size pinned to 1 MB => humongous threshold is 512 KB.

# Run A: 256 KB arrays. Well under the threshold. Ordinary eden allocation.
java -Xmx1g -Xms1g -XX:+UseG1GC -XX:G1HeapRegionSize=1m \
  -Xlog:gc*,gc+heap=debug,gc+humongous=debug:file=gc-small.log:time,uptime,level,tags \
  -cp out com.orderflow.lab.gc.HumongousProbe 262144 200000

# Run B: 600 KB arrays. Over the 512 KB threshold. Every single one is humongous.
java -Xmx1g -Xms1g -XX:+UseG1GC -XX:G1HeapRegionSize=1m \
  -Xlog:gc*,gc+heap=debug,gc+humongous=debug:file=gc-humongous.log:time,uptime,level,tags \
  -cp out com.orderflow.lab.gc.HumongousProbe 600000 200000
```

**WHAT TO LOOK FOR.** Do not look at pause durations first — look at these three things:

1. Does `gc-humongous.log` contain lines tagged `gc,humongous` that `gc-small.log` does
   not? (`grep humongous gc-*.log | wc -l` on each.)
2. In the `gc+heap=debug` output, does the **Humongous** region count in Run B move
   away from zero while Run A's stays at zero?
3. Does Run B contain any `Pause Full` line at all? Does Run A?

| What you see | What it means |
|---|---|
| Run A: zero `humongous` lines; Run B: many | The threshold is exactly where the arithmetic said. You have directly observed the half-a-region rule. |
| Run B shows humongous regions allocated **and** reclaimed at young collections | Eager reclaim is working: your arrays are unreferenced from old gen, so G1 can free them cheaply. This is the *good* case and it is common for a pure `byte[]` with no incoming references. |
| Run B shows humongous region count climbing and only dropping after a `Concurrent Cleanup` | Eager reclaim did **not** apply. Something is holding a reference, or the object shape disqualified it. This is the case that hurts. |
| Run B contains `Pause Full` and Run A does not | Humongous allocation exhausted contiguous free regions. This is the drill's target failure, reproduced in a program with one array in it. |
| **Both** runs show humongous lines | Your region size is not what you think. Re-check with `-XX:+PrintFlagsFinal -version \| grep G1HeapRegionSize` — an explicit `G1HeapRegionSize` that is not a power of two, or is outside the permitted range, is silently adjusted. |
| Neither run shows anything and the log file is empty | The `-Xlog` syntax is wrong. Test the syntax alone first: `java -Xlog:gc -version` should print startup GC lines to stdout. |

Then repeat Run B with `-Xmx8g` and no explicit region size. On an 8 GB heap the region
size is typically 4 MB, the threshold 2 MB, and your 600 KB array is now completely
ordinary. **Same code. Same array. Different heap size. Entirely different allocation
path.** That is the fact to carry into the production scenario.

---

## Example 2 — production scenario (on the project spine)

### The constraints, taken from the Topic 65 baseline

| Thing | Value |
|---|---|
| Service | `orderflow`, containerised |
| Dataset | 100k products, 1M orders, 5M order lines |
| Container | 2 vCPU, 2 GiB memory limit |
| Heap | `-Xmx1200m -Xms1200m` (leaving room for metaspace, code cache, thread stacks, direct buffers — Topic 80) |
| Collector | G1 (verify it, do not assume it) |
| Region size | derived: 1200 MB / 2048 ≈ 586 KB → rounds to **1 MB** → **humongous threshold 512 KB** |
| Load mix | 70% catalogue read, 20% order read, 10% order placement |
| Latency budget | `GET /products` p99 within the recorded baseline; `POST /orders` p99 within the recorded baseline |
| Baseline artefacts | `/docs/java/baselines/` |

### The change that causes the incident

A perfectly reasonable feature lands: `GET /orders/{id}/invoice` returns a generated
PDF-ish payload. The implementation builds it in memory.

```java
package com.orderflow.orders;

import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/orders")
public class InvoiceController {

    private final InvoiceRenderer renderer;

    public InvoiceController(InvoiceRenderer renderer) { this.renderer = renderer; }

    @GetMapping(value = "/{id}/invoice", produces = MediaType.APPLICATION_PDF_VALUE)
    public ResponseEntity<byte[]> invoice(@PathVariable long id) {
        byte[] pdf = renderer.render(id);          // typically 600 KB - 900 KB
        return ResponseEntity.ok(pdf);
    }
}
```

```java
package com.orderflow.orders;

import java.io.ByteArrayOutputStream;
import org.springframework.stereotype.Component;

@Component
public class InvoiceRenderer {

    private final OrderRepository orders;

    public InvoiceRenderer(OrderRepository orders) { this.orders = orders; }

    public byte[] render(long orderId) {
        var order = orders.findWithLines(orderId);
        // Pre-size the buffer "for efficiency". This is the line that does the damage.
        var out = new ByteArrayOutputStream(1_048_576);
        writeHeader(out, order);
        for (OrderLine line : order.lines()) {
            writeLine(out, line);
        }
        writeFooter(out, order);
        return out.toByteArray();          // a SECOND array, also humongous
    }
    // ...
}
```

Read that code as a Java reviewer and it is fine. Read it as someone who knows the
region arithmetic and there are **three** humongous allocations per request:

1. `new ByteArrayOutputStream(1_048_576)` allocates a **1 MB `byte[]`** — that is
   1 048 576 bytes plus a 16-byte header, so it needs **two** 1 MB regions. One
   megabyte of the second region is wasted.
2. `out.toByteArray()` allocates a **second** array of the actual content size. At
   600–900 KB that is over the 512 KB threshold — humongous again.
3. Spring's `HttpMessageConverter` may copy the array again on the way out, depending
   on the converter and whether the response is compressed.

### What you observe, in the order you observe it

This is the sequence, and its shape is the diagnosis:

1. **Invoice traffic is a small fraction of requests** — say a few per second against a
   much larger catalogue-read volume. Nobody suspects it.
2. **`GET /products` p99 degrades**, even though `GET /products` did not change and
   touches none of the new code.
3. **Old-generation occupancy rises steadily** even during periods when order volume is
   flat, because every invoice request puts one to three objects directly into old gen.
4. **Concurrent cycles start running frequently**, which they previously did rarely.
5. **Eventually, `Pause Full` appears**, taking far longer than any pause in the
   baseline, and the error rate spikes as requests time out.
6. If you look only at `Pause Young` durations, **they look fine**, right up until the
   Full GC. This is what makes the incident confusing.

### Why `GET /products` — code you did not touch — got slower

Because GC is a **global**, shared resource. This is the single most important
structural difference from your Node mental model: in Node, a slow endpoint blocks the
loop and you can see it. In Java, a memory-abusive endpoint degrades **every other
endpoint in the process**, through a mechanism that does not appear in any per-endpoint
metric. The invoice endpoint's p99 might look acceptable while it destroys the p99 of
the endpoint next to it.

Write that down. It is the reason GC is an application-architecture concern in Java and
not an infrastructure concern.

### The diagnosis, as commands

```bash
# 1. Confirm the collector and the region size on the RUNNING container.
jcmd $(pgrep -f orderflow) VM.flags -all | grep -E "UseG1GC|G1HeapRegionSize|MaxGCPauseMillis|InitiatingHeapOccupancyPercent"

# 2. Confirm the current region breakdown.
jcmd $(pgrep -f orderflow) GC.heap_info

# 3. Restart with the full log and re-run the Topic 65 load profile unchanged.
#    Log to a file with rotation so a long run does not fill the container's disk.
-Xlog:gc*,gc+heap=debug,gc+humongous=debug:file=/var/log/orderflow/gc.log:time,uptime,level,tags:filecount=5,filesize=50m

# 4. Count humongous events.
grep -c 'humongous' /var/log/orderflow/gc.log

# 5. Find every Full GC and print the 20 lines before each.
grep -n 'Pause Full' /var/log/orderflow/gc.log | cut -d: -f1 | while read n; do
  sed -n "$((n>20 ? n-20 : 1)),${n}p" /var/log/orderflow/gc.log; echo '-----'; done
```

**WHAT TO LOOK FOR:** whether humongous-tagged lines appear at roughly the rate of
invoice requests, and whether the region counts immediately before a `Pause Full` show
few or zero free regions with a nonzero humongous count.

| What you see | What it means |
|---|---|
| Humongous events at roughly the invoice request rate | Confirmed. One endpoint's buffer sizing is driving old-gen growth for the whole service. |
| Humongous events at a *much higher* rate than invoice requests | There is a second source. Look for JDBC fetch buffers, Kafka batches, compression buffers, Jackson's internal byte buffers. `-Xlog:gc+humongous=debug` plus an allocation profile (Topic 78, async-profiler `-e alloc`) names the allocation site. |
| Humongous events but old gen does **not** grow | Eager reclaim is doing its job — the arrays have no old-gen references. Your problem is elsewhere; keep looking. This is a genuinely surprising and genuinely common outcome. |
| No humongous events, but old gen grows anyway | Not a humongous problem. This is promotion (Topic 68) or a retention leak (Topic 79). Different fix entirely. Do not tune G1. |
| `Pause Full` preceded by `To-space exhausted` | Evacuation failure. G1 had nowhere to copy to. Region-count evidence is right there in the `gc+heap=debug` lines. |

### The fixes, in the order you should consider them

**Fix 1 — do not allocate the object (best).** Stream the response instead of buffering
it. `StreamingResponseBody`, or write to the `HttpServletResponse` output stream
directly, using a small reusable buffer. The humongous allocation disappears entirely,
along with three copies of the payload.

```java
@GetMapping(value = "/{id}/invoice", produces = MediaType.APPLICATION_PDF_VALUE)
public ResponseEntity<StreamingResponseBody> invoice(@PathVariable long id) {
    StreamingResponseBody body = out -> renderer.renderTo(id, out);   // 8 KB chunks
    return ResponseEntity.ok(body);
}
```

**Fix 2 — make the allocation small enough not to be humongous.** Remove the
`1_048_576` pre-size; let `ByteArrayOutputStream` grow from its default. This is worse
than Fix 1 because growth reallocates and copies, and a large enough invoice is still
humongous. It is a mitigation, not a fix.

**Fix 3 — change the region size so the object is no longer humongous.**
`-XX:G1HeapRegionSize=4m` raises the threshold to 2 MB, and a 1 MB array becomes an
ordinary eden allocation.

**This is the tempting fix and you should understand its cost before you reach for it.**
Bigger regions mean: fewer regions in the heap (a 1200 MB heap at 4 MB regions has only
300 regions, versus 1200), so the collection set is a much coarser instrument and G1
has far less freedom to hit a pause goal; more space wasted at the tail of partially
used regions; and coarser remembered-set granularity. On a small heap, forcing large
regions can measurably hurt pause behaviour. **Measure against the baseline, do not
assume.**

**Fix 4 — raise the heap.** Going from `-Xmx1200m` to `-Xmx4g` moves the region size to
4 MB by itself and gives more headroom. It also requires raising the container memory
limit, or you get OOM-killed (Topic 82). This is the fix that costs money and is
sometimes still correct.

> **The rule to carry:** when a fix is "change a GC flag" and an alternative fix is
> "allocate less", the second one is almost always right. GC flags redistribute pain.
> Allocating less removes it. Topic 70's arithmetic is why: the collector's cost is a
> function of your allocation rate and live set, and only you control those.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — setting `MaxGCPauseMillis=50` and getting worse latency

**Wrong:**

```bash
java -XX:+UseG1GC -XX:MaxGCPauseMillis=50 -Xmx1200m -jar orderflow.jar
```

The reasoning is seductive: "our budget is tight, so tell G1 to keep pauses under
50 ms."

**Exact symptom:** the *average* pause in the log does fall, and if you only watch a
mean-pause dashboard you will conclude the change worked. Meanwhile:

- p999 for `POST /orders` gets **worse** than the recorded baseline, not better.
- The GC log shows young pauses becoming more frequent and smaller.
- The number of **free regions** reported in `gc+heap=debug` trends downward over
  minutes.
- Eventually: `To-space exhausted` and/or `Evacuation Failure`, then `Pause Full`,
  whose duration dwarfs anything in the baseline.
- The failure appears minutes to hours after the change, not immediately, so it gets
  attributed to something else deployed in between.

**Root cause:** `MaxGCPauseMillis` is a **goal**, not a limit. G1's control loop meets
it by choosing a **smaller collection set** — evacuating fewer regions per pause. Fewer
regions evacuated means less memory reclaimed per pause. If the reclaim rate falls
below your allocation rate, free regions decline monotonically. When G1 needs somewhere
to copy survivors and there is nowhere, evacuation fails, and the recovery is a Full GC.

You did not make pauses shorter. You made G1 do less work each time, and it eventually
had to do all the deferred work at once.

**Fix:**

1. **Revert to the default (200 ms) first** and re-establish the baseline. Prove the
   flag was the cause before theorising.
2. If pauses genuinely need to be shorter, the lever is not the goal — it is the **live
   set** and the **allocation rate**. Reduce survivors, reduce promotion, reduce
   allocation. Topic 70's arithmetic, Topic 78's allocation profile.
3. If the live set is irreducible and the budget is genuinely sub-10 ms, you have a
   **collector-choice** problem, not a tuning problem. That is Topic 72.
4. Watch **free region count** and **Full GC count**, not mean pause. A tuning change
   that lowers mean pause and lowers free regions is a change that is about to hurt you.

**How to confirm the diagnosis in one command:**

```bash
grep -E 'Pause Full|To-space|Evacuation Failure' /var/log/orderflow/gc.log | wc -l
```

Non-zero here after the change and zero before it is your evidence.

---

### Trap 2 — humongous `byte[]` payloads that old gen never reclaims

**Wrong:**

```java
@Service
public class ProductImageCache {
    private final Map<Long, byte[]> thumbnails = new ConcurrentHashMap<>();

    public byte[] thumbnail(long productId) {
        return thumbnails.computeIfAbsent(productId, id -> loadBytes(id));  // ~700 KB each
    }
}
```

**Exact symptom:** old-generation occupancy climbs and never comes down between
concurrent cycles. `-Xlog:gc+humongous=debug` shows humongous regions being allocated
and **not** eagerly reclaimed. Heap utilisation after a concurrent cycle is far above
what your live-set estimate predicted. Eventually the concurrent cycle cannot keep up
and a Full GC appears. On a 1200 MB heap with 1 MB regions, a few hundred cached
thumbnails is most of the old generation.

**Root cause:** two independent mechanisms compounding.

1. Each 700 KB array exceeds the 512 KB humongous threshold, so it is allocated
   directly into old-generation regions and wastes the tail of its last region.
2. Each is **strongly referenced from a long-lived `ConcurrentHashMap`**, which means it
   is genuinely live and eager reclaim can never apply. This is not a GC problem at
   all — it is a **retention** problem (Topic 79) wearing a GC costume.

**Fix:**

- Bound the cache. Caffeine with a **weight-based** eviction policy sized in bytes, not
  entries, because the entries are wildly different sizes.
- Better: do not cache large binaries on the Java heap at all. Put them in Redis, on
  disk, or behind a CDN. Heap is the most expensive place in your system to store bytes
  that never need to be Java objects.
- If they must be in-process, consider off-heap (Topic 80) so they are not the
  collector's problem — while noting that off-heap moves the failure from an
  `OutOfMemoryError` to a container OOM kill, which is worse to debug.
- Confirm with a heap dump and MAT's dominator tree (Topic 79), not with GC flags.

---

### Trap 3 — reading "GC pause" as the whole stop-the-world duration

**Wrong:** the log says the young pause was some small number of milliseconds, so you
conclude GC is not your latency problem and move on to the database.

**Exact symptom:** your service's p99 shows spikes of a magnitude that nothing in the
GC log accounts for. Every GC pause in the log is small. Yet request latency histograms
show a stall, and — the giveaway — the stall hits **requests that were already in
flight on threads doing no GC work at all**, across unrelated endpoints, all at the
same instant.

**Root cause:** the pause duration in a `gc` log line is the time spent doing garbage
collection work at the safepoint. It **does not include time-to-safepoint (TTSP)** —
the time between "the JVM decided to stop the world" and "the last application thread
actually stopped". If one thread is in a long counted `int` loop, or a giant
`System.arraycopy`, or waiting on a page that has been swapped out, every other thread
sits stopped waiting for it. GC gets blamed for a delay it did not cause.

**Fix:** measure the two separately.

```bash
-Xlog:safepoint*:file=/var/log/orderflow/safepoint.log:time,uptime,level,tags
```

**WHAT TO LOOK FOR:** in the safepoint log, the fields that report the time spent
*reaching* the safepoint versus the time spent *at* it. When the reaching time is the
larger of the two, GC is not your problem. Topic 73 is the entire treatment; do not
tune G1 until you have ruled TTSP out.

---

### Trap 4 — tuning flags copied from a blog post

**Wrong:**

```bash
-XX:+UseG1GC -XX:MaxGCPauseMillis=100 -XX:InitiatingHeapOccupancyPercent=35 \
-XX:G1NewSizePercent=10 -XX:G1MaxNewSizePercent=40 -XX:ParallelGCThreads=8 \
-XX:ConcGCThreads=4 -XX:G1HeapRegionSize=16m -XX:+ParallelRefProcEnabled
```

Nine flags, applied together, from a post about a different service with a different
heap size on different hardware.

**Exact symptom:** it is impossible to say. That is the symptom. Some of these help,
some hurt, and because they were applied together you cannot attribute any change to
any flag. On a 2 vCPU container, `-XX:ParallelGCThreads=8` alone is actively harmful:
eight GC threads contending for two cores means context-switching overhead and longer
pauses, not shorter. `-XX:G1HeapRegionSize=16m` on a 1200 MB heap gives you 75 regions
total, which cripples G1's ability to size a collection set at all.

**Root cause:** GC flags are not additive and they are not portable. Every one of them
interacts with heap size, live set, allocation rate, CPU quota and the other flags.
Setting `InitiatingHeapOccupancyPercent` explicitly *also silently disables* adaptive
IHOP, which is a second change you did not intend to make.

**Fix:**

1. **Start from defaults.** G1's defaults are the product of a great deal of tuning
   against a great many workloads. Your first job is to reproduce the Topic 65 baseline
   with defaults and record it.
2. **Change one flag at a time**, re-run the identical load profile, and compare
   against the recorded baseline percentiles. If the change is inside the run-to-run
   noise of your baseline, it is not a change.
3. **Set only what you can justify in one sentence**, and write that sentence in a
   comment next to the flag in your deployment manifest.
4. The flags that are usually worth setting: `-Xms` equal to `-Xmx` (avoids heap
   resizing pauses and makes region sizing deterministic), an explicit collector so
   nobody is surprised by a container-driven default, and `-Xlog:gc*` to a rotating
   file so the evidence exists when you need it.

---

### Trap 5 — assuming the collector you configured is the collector you got

**Wrong:** the deployment sets `-XX:MaxGCPauseMillis=200` and everyone assumes G1,
because G1 is the default.

**Exact symptom:** GC logs contain no G1-specific tags at all. `Pause Young` lines look
structurally different from what this document describes. Region-related flags appear
to have no effect. Worst case: the service is on Serial GC in production and nobody
noticed, because the pauses were tolerable at low traffic and only became a problem at
peak.

**Root cause:** the JVM chooses a default collector from the machine it thinks it is
on. In a container with a small CPU quota or a small memory limit, the JVM may see
fewer than 2 processors or less than 1792 MB and select **Serial GC**. Meanwhile
`-XX:MaxGCPauseMillis` is accepted without complaint because it is a valid flag — it is
just being ignored by a collector that has no pause goal.

**Fix:** never infer the collector. Print it, at startup, in the application's own logs:

```java
import java.lang.management.ManagementFactory;

@Component
class RuntimeFacts implements ApplicationRunner {
    private static final Logger log = LoggerFactory.getLogger(RuntimeFacts.class);

    @Override public void run(ApplicationArguments args) {
        ManagementFactory.getGarbageCollectorMXBeans()
            .forEach(b -> log.info("gc collector = {}", b.getName()));
        log.info("availableProcessors = {}", Runtime.getRuntime().availableProcessors());
        log.info("maxMemory MB = {}", Runtime.getRuntime().maxMemory() / (1024 * 1024));
    }
}
```

and confirm from outside with:

```bash
jcmd <pid> VM.flags -all | grep -E "UseG1GC|UseSerialGC|UseParallelGC|UseZGC|UseShenandoahGC"
```

**WHAT TO LOOK FOR:** the collector MXBean names. G1 reports two beans whose names
identify the young and old collectors as G1's. Serial and Parallel report different
names. Read the names your JVM actually prints — do not match them against a list you
remembered.

---

## Hands-on proof

Every command here is one **you** run. I have no JVM and will not print output and call
it captured. What follows is the exact command, what to look for, and how to read every
plausible result — including the ones that contradict the expected answer.

### Setup

```bash
mkdir -p ~/java-lab/71 && cd ~/java-lab/71
java --version                 # expect 21 or 25; record which
mkdir -p logs
```

### Proof 1 — establish your region size and humongous threshold

```bash
for H in 512m 1g 2g 4g 8g 16g; do
  printf "%-6s " "$H"
  java -Xmx$H -XX:+UseG1GC -XX:+PrintFlagsFinal -version 2>/dev/null \
    | awk '/G1HeapRegionSize/ {print $4}'
done
```

**WHAT TO LOOK FOR:** the step pattern, and where the steps land for your JDK.

| What you see | What it means |
|---|---|
| Values increase in powers of two as the heap grows, hitting a floor at small heaps | Expected. You have derived the humongous threshold for every heap size you might deploy with. Halve each value and write it down. |
| The value is `0` | On some JDKs the *flag's default* prints as 0, meaning "compute it at startup". Get the real runtime value from a running JVM instead: `jcmd <pid> VM.flags -all \| grep G1HeapRegionSize`. |
| The value does not change as the heap grows | You have an explicit `G1HeapRegionSize` set somewhere — a `JAVA_TOOL_OPTIONS` env var, a `_JAVA_OPTIONS`, or a wrapper script. Check with `env \| grep -i java`. |
| A warning about the region size being adjusted | You asked for a non-power-of-two or out-of-range value. The JVM silently picks the nearest legal one and tells you. Believe the warning, not your flag. |

### Proof 2 — see the region breakdown of a live JVM

```bash
jcmd <pid> GC.heap_info
```

**WHAT TO LOOK FOR:** the total region count, the region size, and the counts of young,
survivor and humongous regions.

| What you see | What it means |
|---|---|
| Total regions ≈ 2048 | Default sizing. G1 has plenty of granularity to work with. |
| Total regions in the low hundreds | Either a small heap or a forced large region size. G1's collection-set sizing is coarse here; a pause goal is a blunt instrument. |
| Nonzero humongous count | You have humongous objects live right now. Get an allocation profile (Topic 78) filtered to large arrays to find where they come from. |
| Humongous count that never returns to zero across many minutes | Something long-lived is holding them. That is Topic 79, not Topic 71. |

### Proof 3 — read a young-collection log line, field by field

You must be able to parse these without help. Here is the **shape**:

```
[2026-08-29T14:02:11.417+0000][612.041s][info][gc] GC(318) Pause Young (Normal) (G1 Evacuation Pause) 918M->274M(1200M) 23.115ms
```

***Illustration of the format, not captured output. The numbers are placeholders chosen
to show the field positions.***

Field by field:

| Field | Meaning |
|---|---|
| `[2026-08-29T14:02:11.417+0000]` | Wall-clock timestamp. Present because you passed `time` in the `-Xlog` decorators. This is how you correlate with your load generator's timeline and your APM. |
| `[612.041s]` | Uptime. Present because you passed `uptime`. Better than wall clock for measuring intervals between collections. |
| `[info]` | Log level. `debug` lines appear only for tags you explicitly raised. |
| `[gc]` | Tag set. `gc,heap`, `gc,humongous`, `gc,phases` etc. identify which subsystem emitted the line. **Grep on the tag, not on the message text.** |
| `GC(318)` | Collection sequence number. All lines belonging to one collection share it — this is how you group a multi-line collection back together. |
| `Pause Young (Normal)` | The pause type. See the table below. |
| `(G1 Evacuation Pause)` | The cause. This is the field that names *why* the collection happened. |
| `918M->274M(1200M)` | Heap used **before** → **after** (total heap). |
| `23.115ms` | Duration of the GC work at the safepoint. **Not** the stop-the-world duration; TTSP is separate (Topic 73). |

The pause types you must recognise:

| Pause type / cause | What it means | Is it a problem? |
|---|---|---|
| `Pause Young (Normal) (G1 Evacuation Pause)` | Ordinary young collection: eden filled. | No. This is G1 working. |
| `Pause Young (Concurrent Start) (G1 Humongous Allocation)` | A young collection that also starts a concurrent cycle, triggered by a humongous allocation. | **Yes — this is your humongous smoking gun.** |
| `Pause Young (Concurrent Start) (G1 Evacuation Pause)` | Concurrent cycle starting because old-gen occupancy crossed IHOP. | Normal, unless very frequent. |
| `Pause Young (Prepare Mixed)` | The young collection immediately before mixed collections begin. | No. |
| `Pause Young (Mixed)` | Young regions plus some old candidate regions. | No — this is how old gen gets reclaimed without a Full GC. |
| `Pause Remark` | STW end of concurrent marking. | Normal; watch its duration if you have many weak references. |
| `Pause Cleanup` | Brief STW liveness accounting. | Normal, usually very short. |
| `Pause Full (G1 Compaction Pause)` | **Failure.** Full heap, single compacting collection. | **Yes. Always. Explain every one.** |
| `To-space exhausted` / `Evacuation Failure` | Nowhere to copy survivors to. | **Yes.** Trap 1's mechanism, caught in the act. |
| `(System.gc())` as the cause | Something called `System.gc()`. | **Yes** — find it. A library, a JMX console, or `Runtime.getRuntime().gc()` in your own code. `-XX:+DisableExplicitGC` suppresses it, but find the caller first. |
| `(Metadata GC Threshold)` | Metaspace, not heap. | Different problem: class loading. Topics 67 and 80. |
| `(G1 Periodic Collection)` | G1 collecting while idle to return memory to the OS. | Normal if `G1PeriodicGCInterval` is set; unexpected otherwise. |

### Proof 4 — group a whole collection back together

```bash
grep 'GC(318)' /var/log/orderflow/gc.log
```

**WHAT TO LOOK FOR:** every line for that collection, including the `gc,heap` region
counts and, if `gc+phases=debug` is on, the sub-phase breakdown. This single command is
how you go from "a long pause happened" to "the long pause was in Object Copy, and the
region counts show why".

| What you see | What it means |
|---|---|
| A `gc,phases` line dominated by Object Copy | Large live set. Topic 70's arithmetic. Reduce survivors or promotion, not flags. |
| A `gc,phases` line dominated by Scan RS / Update RS | Very high reference-mutation rate; refinement threads behind. Look at what is writing references at that rate. |
| A `gc,phases` line dominated by Ext Root Scanning | Very many threads, or very large static structures. Topic 98 (thread count), Topic 79 (static retention). |
| A `gc,heap` line showing zero or near-zero free regions | You are one allocation away from evacuation failure. This is the state Trap 1 produces. |
| Only one line for the collection | You logged at `info` with no `debug` tags. Add `gc+heap=debug,gc+phases=debug` and re-run. |

### Proof 5 — measure allocation rate from the log, correctly

You need this number for every conversation in Phases 8 and 9. Here is the formula.

For two **consecutive young collections** `n` and `n+1`:

```
allocated_between = (heap_used_before_collection[n+1]) - (heap_used_after_collection[n])
elapsed           = (uptime_at[n+1]) - (uptime_at[n])
allocation_rate   = allocated_between / elapsed          // MB/s
```

In words: the heap grew from "what was left after the last collection" to "what was
there when the next one started", over that interval. That growth is what your
application allocated.

```bash
# Extract the fields; do the arithmetic yourself on YOUR log.
grep 'Pause Young' /var/log/orderflow/gc.log \
  | sed -E 's/.*\[([0-9.]+)s\].*GC\(([0-9]+)\).*[^0-9]([0-9]+)M->([0-9]+)M\(([0-9]+)M\).*/\1 \2 \3 \4 \5/'
# columns: uptime, gcId, usedBefore, usedAfter, heapTotal
```

**I am deliberately not showing you a worked example with numbers**, because any numbers
I invented would be fiction and you would remember them. Run it on your own log and do
the subtraction. Then sanity-check the result: if your allocation rate comes out
implausibly high or low, the most common cause is mixing young and mixed collections in
the same subtraction, or a concurrent cycle having freed memory in between.

Do the same for **promotion rate**, which Topic 68 told you matters more:

```
promoted_between = (old_gen_used_after[n+1]) - (old_gen_used_after[n])
promotion_rate   = promoted_between / elapsed
```

Old-generation occupancy comes from the `gc+heap=debug` lines, not the summary line.

**WHAT TO LOOK FOR:** allocation rate that is high but stable with a flat promotion rate
is healthy — that is objects dying young, exactly as they should. Allocation rate that
is unremarkable with a **rising** promotion rate is the dangerous shape: your live set
is growing, and old-gen pressure and eventual Full GCs are the consequence.

### Proof 6 — confirm eager reclaim of humongous objects

```bash
java -Xmx1g -Xms1g -XX:+UseG1GC -XX:G1HeapRegionSize=1m \
  -Xlog:gc*,gc+humongous=debug:file=logs/eager-on.log:time,uptime,level,tags \
  -cp out com.orderflow.lab.gc.HumongousProbe 600000 200000

java -Xmx1g -Xms1g -XX:+UseG1GC -XX:G1HeapRegionSize=1m \
  -XX:-G1EagerReclaimHumongousObjects \
  -Xlog:gc*,gc+humongous=debug:file=logs/eager-off.log:time,uptime,level,tags \
  -cp out com.orderflow.lab.gc.HumongousProbe 600000 200000

grep -c 'Pause Full' logs/eager-on.log logs/eager-off.log
grep -c 'humongous'  logs/eager-on.log logs/eager-off.log
```

**WHAT TO LOOK FOR:** the difference in Full GC counts and in how quickly humongous
regions are returned to the free list.

| What you see | What it means |
|---|---|
| Eager-on has fewer or zero Full GCs; eager-off has more | Eager reclaim is doing real work for this allocation shape. This is the common case for a `byte[]` with no incoming old-gen references. |
| Both are identical | Your objects were not eligible for eager reclaim either way. Something references them from old gen, or the shape disqualifies them. Look at what holds the reference. |
| Eager-off is *better* | Genuinely surprising, and worth investigating rather than dismissing: eager reclaim has a bookkeeping cost, and for a workload where it almost never succeeds, that cost is pure overhead. Confirm by re-running both several times — a single run's difference may be noise. |

---

## Failure drill

**Mandatory.** Do not read the "how to read it" table until you have produced the
failure yourself and written down what you saw. The point is not the knowledge; it is
the memory of the moment your service stopped for seconds and you could name why.

### The assignment, restated from the master plan

> Drive `orderflow` allocation rate up and allocate large `byte[]` payloads sized just
> over half the G1 region size. Capture
> `-Xlog:gc*,gc+heap=debug,gc+humongous=debug:file=gc.log:time,uptime,level,tags`.
> Identify the multi-second pause's cause. Fix by region sizing or by not allocating the
> humongous object.

### Step 0 — establish the control

Before you break anything:

```bash
# Re-run the Topic 65 baseline load profile, unchanged, with GC logging on.
docker compose up -d
java -Xmx1200m -Xms1200m -XX:+UseG1GC \
  -Xlog:gc*,gc+heap=debug,gc+humongous=debug:file=/var/log/orderflow/gc-control.log:time,uptime,level,tags \
  -jar orderflow.jar

k6 run --out json=logs/control.json load/baseline.js
```

Record, in your own notes: p50, p95, p99, p999, throughput, error rate, Full GC count,
humongous line count. **If these are not within ±10% of the committed baseline in
`/docs/java/baselines/`, stop.** The Topic 65 gate rule applies: a drill against an
unstable baseline proves nothing.

### Step 1 — compute the threshold you are about to cross

```bash
jcmd $(pgrep -f orderflow) VM.flags -all | grep G1HeapRegionSize
```

Halve that number. That is your humongous threshold. Your payload size will be that
number plus a comfortable margin — say the threshold plus 20% — so there is no
ambiguity about which side of the line you are on.

On a 1200 MB heap with 1 MB regions, the threshold is 512 KB and your payload is
**640 000 bytes**.

### Step 2 — add the humongous allocation to a hot endpoint

Add an endpoint to `orderflow` that allocates a humongous array per request and keeps
it alive just long enough to be promoted-ish. Reference it from a bounded structure so
that eager reclaim cannot trivially save you — this is the realistic case, and it is
the one that produces the failure.

```java
package com.orderflow.lab;

import java.util.ArrayDeque;
import java.util.Deque;
import org.springframework.web.bind.annotation.*;

/**
 * DRILL CODE. Never merge this. It exists to make humongous allocation visible
 * under the Topic 65 load profile.
 */
@RestController
@RequestMapping("/lab")
public class HumongousEndpoint {

    /** Sized from the measured region size: threshold (512 KB) + 25%. */
    private static final int PAYLOAD_BYTES = 640_000;

    /** Holds the last N payloads, so some survive long enough to matter. */
    private final Deque<byte[]> recent = new ArrayDeque<>();
    private static final int KEEP = 64;

    @GetMapping("/humongous")
    public synchronized int humongous() {
        byte[] payload = new byte[PAYLOAD_BYTES];
        payload[0] = 1;
        payload[PAYLOAD_BYTES - 1] = 1;
        recent.addLast(payload);
        while (recent.size() > KEEP) {
            recent.removeFirst();
        }
        return recent.size();
    }
}
```

> The `synchronized` is deliberate and is *not* the point of this drill — it keeps the
> deque correct without introducing a second variable. Topic 85 will have opinions
> about it. Note that at `KEEP = 64` and 640 KB per payload you are pinning roughly
> 40 MB of humongous regions live at all times, on a 1200 MB heap.

### Step 3 — drive it under load, with the full log

```bash
java -Xmx1200m -Xms1200m -XX:+UseG1GC \
  -Xlog:gc*,gc+heap=debug,gc+humongous=debug:file=/var/log/orderflow/gc-drill.log:time,uptime,level,tags:filecount=5,filesize=50m \
  -jar orderflow.jar
```

Extend the k6 script so a meaningful share of requests hit `/lab/humongous` while the
rest of the Topic 65 mix continues unchanged:

```javascript
// load/humongous-drill.js — the Topic 65 mix plus a humongous stream.
import http from 'k6/http';

export const options = {
  scenarios: {
    baseline_mix: {
      executor: 'constant-arrival-rate',
      rate: 400, timeUnit: '1s', duration: '10m',
      preAllocatedVUs: 200, maxVUs: 600,
      exec: 'baselineMix',
    },
    humongous: {
      executor: 'constant-arrival-rate',
      rate: 40, timeUnit: '1s', duration: '10m',
      preAllocatedVUs: 50, maxVUs: 200,
      exec: 'humongous',
    },
  },
  // Open-model arrival rate, per the Topic 65 gate: fixed-VU loops under-report the
  // tail because a stalled VU stops issuing requests. That is coordinated omission.
};

export function baselineMix() { /* the recorded 70/20/10 mix, unchanged */ }
export function humongous()   { http.get('http://localhost:8080/lab/humongous'); }
```

Adjust the rates to your own recorded baseline. **Do not change the baseline mix** —
the whole point is that a new, low-rate endpoint degrades the endpoints that did not
change.

### Step 4 — what to capture

Write down, before reading anything else:

1. `grep -c humongous /var/log/orderflow/gc-drill.log`
2. `grep -c 'Pause Full' /var/log/orderflow/gc-drill.log`
3. `grep -cE 'To-space exhausted|Evacuation Failure' /var/log/orderflow/gc-drill.log`
4. The **longest** pause duration in the log, and its `GC(n)` id and cause string.
5. The full multi-line group for that collection: `grep 'GC(<n>)' gc-drill.log`
6. p50/p95/p99/p999 for **`GET /products`** — the endpoint you did not touch — from k6.
7. The free-region count in the `gc,heap` lines immediately before the longest pause.

### Step 5 — how to read it

| What you see | What it means |
|---|---|
| Many lines tagged `gc,humongous`, a nonzero humongous region count, and a `Pause Full` whose cause mentions humongous allocation | **The drill has fired.** You have a named cause for a multi-second pause, derived from evidence. Write the sentence: "the pause at uptime T was a Full GC caused by humongous allocation exhausting contiguous free regions". |
| `To-space exhausted` immediately before the long pause | Evacuation failure. G1 had nowhere to copy survivors. Note whether the preceding `gc,heap` line shows free regions at or near zero — that is the direct evidence. |
| Humongous lines present, but **no** Full GC and pauses within baseline | Eager reclaim is handling them, or your `KEEP` is too small to matter on this heap size. Raise `KEEP`, or reduce `-Xmx` so the same payload is a larger fraction of the heap. Do not conclude "humongous is harmless" — conclude "on this heap, at this retention, it was". |
| `GET /products` p99 degrades while `/lab/humongous` p99 looks acceptable | **The most important observation in the drill.** GC is a shared resource. A low-traffic endpoint degraded a high-traffic one through a mechanism that appears in neither endpoint's own metrics. |
| Pauses grow but there are **no** humongous lines | You are below the threshold. Re-check the region size with `jcmd VM.flags -all` — it may not be what you computed, particularly if `-Xms` differs from `-Xmx` or a wrapper script sets flags. |
| The service gets OOM-killed by the container instead of pausing | You are hitting Topic 82's problem, not Topic 71's. Heap plus metaspace plus code cache plus thread stacks plus direct buffers exceeded the container limit. Raise the limit or lower `-Xmx`, then re-run. |
| Everything looks identical to the control | Confirm the endpoint is actually being hit (`grep humongous` in the access log), and confirm the array size is above the threshold. An array that is exactly at 50% of a region *is* humongous; one at 49% is not, and header plus padding can put you either side of the line. |

### Step 6 — fix it two ways and compare

**Fix A — change the region size so the allocation is no longer humongous.**

```bash
java -Xmx1200m -Xms1200m -XX:+UseG1GC -XX:G1HeapRegionSize=2m \
  -Xlog:gc*,gc+heap=debug,gc+humongous=debug:file=/var/log/orderflow/gc-fixA.log:time,uptime,level,tags \
  -jar orderflow.jar
```

At 2 MB regions the threshold is 1 MB and 640 000 bytes is ordinary. Re-run the
identical load. Capture the same seven items.

**WHAT TO LOOK FOR:** humongous count should go to zero. But also look at the *young*
pause durations and the total region count — you have halved the number of regions in
the heap, which reduces G1's granularity. **If young pauses got worse while humongous
went away, that is the trade, and you must report both halves.**

**Fix B — do not allocate the humongous object.**

```java
    /** Pool of reusable buffers, each below the humongous threshold. */
    private static final int CHUNK = 64 * 1024;

    @GetMapping("/humongous")
    public int fixed() {
        // Whatever the payload was for, do it in chunks that fit in eden and die there.
        byte[] chunk = new byte[CHUNK];
        // ... process in CHUNK-sized pieces, streaming rather than buffering ...
        return CHUNK;
    }
```

Re-run. Capture the same seven items.

### What the fix proves

The comparison is the point, not either fix on its own.

- **Control**: baseline percentiles, zero humongous lines, zero Full GCs.
- **Drill**: humongous lines, Full GC, degraded p99 on an endpoint you never touched.
- **Fix A**: humongous gone, at the cost of a coarser heap — and you can now *state
  that cost with evidence* rather than asserting it.
- **Fix B**: humongous gone with no GC configuration change at all.

Carry one sentence out of this drill:

> *A GC flag redistributes the cost. Allocating less removes it. Both are legitimate,
> and only one of them is free.*

And carry one more, which is the interview-grade version:

> *I can name a pause's cause from the log. `Pause Full` with a humongous-allocation
> cause and near-zero free regions in the preceding heap line is not a guess — it is a
> reading.*

---

## Measurement

### The instrument for this topic

For Topics 71 and 72, **the instrument is the GC log plus a percentile comparison
against the recorded Topic 65 baseline.** Not a microbenchmark. Not a stopwatch. Not
"it feels faster".

There is a reason for that. GC behaviour is an emergent property of allocation rate,
live-set size, object lifetime distribution, heap size, CPU quota and the collector's
control loop. None of those are reproduced by a microbenchmark. A JMH benchmark of
`new byte[640_000]` measures allocation, which is a pointer bump (Topic 68) and is
nearly free. It tells you nothing about what the collector does with the resulting heap
state twenty seconds later.

So the measurement protocol is:

1. **Identical load profile** — the same k6 script, the same arrival rate, the same
   dataset, the same cache state. Restart Postgres or re-warm it identically.
2. **Long enough to include several concurrent cycles.** A two-minute run can miss the
   failure entirely. Ten minutes minimum; longer if your baseline shows infrequent
   concurrent cycles.
3. **Change exactly one thing.**
4. **Compare percentiles, not means.** p99 and p999 are where GC lives. A mean can
   improve while p999 doubles.
5. **Compare Full GC count and free-region floor**, which are leading indicators of a
   failure your percentiles have not shown you yet.
6. **Repeat each configuration at least three times** so you know your run-to-run noise.
   A change smaller than your noise is not a change.

### Why a naive `System.nanoTime()` loop is WRONG here

You will be tempted to write this:

```java
// DO NOT DO THIS. Every number it produces is meaningless.
long start = System.nanoTime();
for (int i = 0; i < 1_000_000; i++) {
    byte[] b = new byte[640_000];
}
System.out.println((System.nanoTime() - start) / 1_000_000 + " ns/alloc");
```

Four independent reasons it lies, and you cannot tell which one is lying:

1. **Dead-code elimination.** `b` is never used. C2 can prove the allocation has no
   observable effect and delete it entirely. You measure an empty loop. (Topic 75 is the
   full treatment: this is escape analysis plus dead-code elimination, and it is exactly
   the optimisation that makes naive allocation benchmarks report absurd numbers.)
2. **Constant folding.** The size is a compile-time constant and the loop bound is
   known. C2 has enormous freedom here.
3. **On-stack replacement (OSR).** The loop begins interpreted, is compiled *while
   running*, and is swapped mid-flight. Your average blends interpreted, C1 and C2
   execution in a ratio that depends on the loop count you happened to choose.
   (Topic 74.)
4. **Cold JIT.** The first thousands of iterations run interpreted. If your loop is
   short, that is most of your measurement.

And a fifth reason specific to this topic: **you are trying to measure the collector,
and the collector is not in the loop.** The cost of GC is paid at collection time, on
GC threads, asynchronously to your loop. A timer around allocations measures allocation,
not collection. Even a *correct* JMH benchmark of allocation would answer the wrong
question.

### Where JMH *is* the right tool in this topic

Only for one thing: comparing the cost of a specific allocation shape. For example,
"does pre-sizing this `ByteArrayOutputStream` actually help, given that it makes the
allocation humongous?" That is a legitimate JMH question, and JMH's allocation profiler
answers it directly.

```java
package com.orderflow.bench;

import org.openjdk.jmh.annotations.*;
import org.openjdk.jmh.infra.Blackhole;
import java.io.ByteArrayOutputStream;
import java.util.concurrent.TimeUnit;

@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.MICROSECONDS)
@Warmup(iterations = 5, time = 1)
@Measurement(iterations = 10, time = 1)
@Fork(3)                                  // 3 separate JVMs: defeats profile pollution
@State(Scope.Benchmark)
public class InvoiceBufferBenchmark {

    @Param({"32768", "262144", "1048576"})   // below, near, and above a 512 KB threshold
    public int preSize;

    private byte[] chunk;

    @Setup(Level.Trial)
    public void setUp() { chunk = new byte[8192]; }

    @Benchmark
    public void buildInvoice(Blackhole bh) {
        var out = new ByteArrayOutputStream(preSize);
        for (int i = 0; i < 80; i++) {
            out.write(chunk, 0, chunk.length);
        }
        bh.consume(out.toByteArray());       // consumed, so it cannot be eliminated
    }
}
```

```bash
./mvnw -Pbench clean verify
java -jar target/benchmarks.jar InvoiceBufferBenchmark \
     -prof gc -rf json -rff invoice-bench.json
```

| Annotation / flag | What it defends against |
|---|---|
| `@State(Scope.Benchmark)` | Holds state in a field JMH controls, so C2 cannot constant-fold it. |
| `Blackhole.consume(...)` | Defeats dead-code elimination — the reason the naive loop lies. |
| `@Warmup(iterations = 5)` | Lets C2 compile and reach steady state before recording. |
| `@Fork(3)` | Three separate JVMs. Exposes run-to-run variance and defeats profile pollution between `@Param` values (Topic 74). |
| `@Param` | Runs each size as a separate benchmark, so the JIT does not see one call site with three shapes. |
| `-prof gc` | **The important one.** Reports allocation rate and normalised bytes-per-operation from JMH's own GC instrumentation. This is what turns a timing benchmark into an allocation measurement. |

**WHAT TO LOOK FOR:** the bytes-per-operation figure from `-prof gc`, not the time. If
the 1 MB pre-size allocates more bytes per operation than the 32 KB one, the "efficient"
pre-size is costing you memory *and* pushing you over the humongous threshold. Report
the confidence interval JMH prints, never the point estimate — and if the intervals
overlap, you measured nothing.

Topic 77 is the full treatment of JMH. Do not write a benchmark you intend to act on
until you have read it.

### The three numbers to track continuously, in production

Not during a drill — permanently, on a dashboard:

| Number | Where from | Why |
|---|---|---|
| Full GC count | GC log, or the `jvm.gc.pause` meter with the appropriate cause/action tags | Should be zero. Any nonzero value is an incident. |
| Allocation rate (MB/s) | Derived from consecutive young collections, per the formula above; or Micrometer's `jvm.gc.memory.allocated` counter | The input to every capacity conversation. |
| Promotion rate (MB/s) | Old-gen occupancy delta across collections; or `jvm.gc.memory.promoted` | Topic 68's point: this is what drives old-gen pressure, not allocation. |

Topic 118 covers exporting these as metrics without blowing up cardinality.

---

## Practice exercises

### 1 — Easy: build your own region-size and threshold table

For heap sizes 256m, 512m, 1g, 2g, 4g, 8g, 16g, 32g, 64g:

1. Print the actual `G1HeapRegionSize` your JDK chooses.
2. Compute the humongous threshold (half of it).
3. Compute the total number of regions.

Produce a table with those three columns. Then answer, in one sentence each:

- Why does the region size stop growing at the small end?
- A service allocates 700 KB response buffers. At which of those heap sizes is that
  allocation humongous? What does that tell you about deploying the same jar to a
  staging environment with a smaller heap?
- If you set `-XX:G1HeapRegionSize=32m` on a 1 GB heap, how many regions do you have?
  Why is that bad for a pause goal?

### 2 — Medium: the audit (combines Topics 01, 11, 18, 21, 25, 68, 70)

This class runs on the `orderflow` catalogue-read path at 400 rps. Find **six** defects.
Four are allocation-behaviour defects relevant to G1; two are from earlier topics. For
each: name the topic, state the **observable** symptom in the GC log or in a latency
percentile, and write the fix.

```java
package com.orderflow.catalog;

import java.util.*;
import java.util.stream.Collectors;

@Service
public class CatalogExportService {

    private static final Map<Long, byte[]> RENDERED = new HashMap<>();

    private final ProductRepository products;

    public CatalogExportService(ProductRepository products) { this.products = products; }

    public String exportCsv(List<Long> productIds) {
        String csv = "";
        for (Long id : productIds) {
            Product p = products.findById(id).orElseThrow();
            csv = csv + p.sku() + "," + p.name() + "," + p.priceMinor() + "\n";
        }

        byte[] bytes = new byte[1_048_576];
        System.arraycopy(csv.getBytes(), 0, bytes, 0, csv.getBytes().length);
        RENDERED.put(productIds.get(0), bytes);

        Map<Long, Long> priceIndex = productIds.parallelStream()
            .collect(Collectors.toMap(id -> id, id -> products.priceOf(id)));

        return csv;
    }
}
```

Hints, in the order to think about them: one defect makes the heap grow without bound
and is a Topic 79 shape; one is a humongous allocation from this topic; one is a
Topic 18 quadratic-allocation problem that will dominate your allocation rate; one is a
Topic 01 boxing problem in a map; one is a Topic 25 problem that will make this worse
under load in a way that has nothing to do with memory; one is a redundant computation
that doubles the allocation of the line it is on.

For the humongous one specifically: state the heap size at which it stops being
humongous, and say whether raising the heap is an acceptable fix here.

### 3 — Hard: production simulation — name the cause from evidence alone

**Part A — reproduce and record.** Run the Topic 65 baseline with full GC logging.
Confirm you are within ±10% of the committed baseline. Record: p50/p95/p99/p999,
throughput, error rate, Full GC count, humongous line count, allocation rate (using the
formula above), promotion rate, and the minimum free-region count observed.

**Part B — introduce three independent problems, one at a time.** Between each, restore
the baseline and confirm you are back within ±10%.

1. The humongous endpoint from the failure drill.
2. `-XX:MaxGCPauseMillis=25` with no other change.
3. An unbounded `static ConcurrentHashMap<Long, Order>` populated on every order read
   (this is Topic 79's drill, borrowed).

For each, record the same nine numbers.

**Part C — the blind test.** This is the exercise that actually teaches the topic.
Hand the three GC logs to someone else — or to yourself in a week — **with the labels
removed**. For each log, write down:

- The pause type and cause of the longest pause.
- Whether the problem is allocation rate, live set, humongous allocation, or
  configuration.
- The single command you would run next to confirm.
- The fix you would propose, and its cost.

Then check your answers against your Part B notes. **Any log you cannot diagnose from
the log alone is a gap in your reading skill, not in your tuning skill.** That
distinction is the entire point of Phase 8.

**Part D — the cost of the tuning fix.** For problem (1), apply the region-size fix
(`-XX:G1HeapRegionSize=2m`) and measure. Then apply the allocation fix (stream instead
of buffer) and measure. Produce a table comparing them on: p99, p999, throughput, Full
GC count, young-pause p99, total region count, and lines of application code changed.

Then answer: under what circumstances would you ship the flag change instead of the
code change? Name at least one circumstance where that is the correct engineering
decision, and be honest about it.

**Part E — argue against yourself.** You will conclude that the allocation fix is
better. Make the strongest possible case that the region-size flag is the right answer
for `orderflow` specifically. Then say what would have to be true about the team, the
release process, or the incident timeline for that case to win.

---

## Interview questions

### Q1 — "Our p99 has 2-second spikes. Walk me through diagnosing it."

**MID-LEVEL answer:** "I'd look at the GC logs and see if there are long pauses. If
there are, I'd increase the heap size, or lower `MaxGCPauseMillis` so pauses get
shorter. I'd also check if there's a memory leak."

**SENIOR answer:** "First I'd establish whether it's GC at all, because a 2-second spike
has at least five candidate causes and GC is only one. My order:

**1. Correlate.** Do the spikes line up in time with GC pauses, or not? I need a GC log
with wall-clock timestamps — `-Xlog:gc*:file=gc.log:time,uptime,level,tags` — and the
request latency histogram on the same clock. If the long GC pauses do not coincide with
the latency spikes, GC is exonerated in one step and I've saved a day.

**2. Separate GC pause from stop-the-world.** The duration in a `Pause Young` line is GC
work at the safepoint. It excludes time-to-safepoint. I've seen a reported 5 ms pause
hiding a 900 ms TTSP from one thread in a counted `int` loop. So I turn on
`-Xlog:safepoint*` and compare the time spent *reaching* the safepoint against the time
spent *at* it. That is a completely different bug with a completely different fix.

**3. If it is GC, name the pause type from the log.** `Pause Full` means something
failed — G1 shouldn't do full collections in steady state. `To-space exhausted` means
evacuation failure: G1 had nowhere to copy survivors. A `Concurrent Start` caused by
humongous allocation means someone is allocating objects over half a region. Those are
three different problems with three different fixes, and the log distinguishes them
without any guessing.

**4. Get the two numbers that drive everything.** Allocation rate and live set. I derive
allocation rate from consecutive young collections — heap-used-before at collection
n+1 minus heap-used-after at collection n, over the elapsed interval — and promotion
rate from the old-gen deltas. High allocation with flat promotion is healthy; objects
are dying young. Rising promotion is the dangerous shape, and it means the fix is
retention, not flags.

**5. Only then consider configuration.** And I'd be suspicious of anyone who reached
for flags at step 1. Specifically I'd check whether someone has already 'tuned'
`MaxGCPauseMillis` down, because that's a goal G1 meets by collecting fewer regions,
which starves it of free regions and produces exactly the multi-second Full GC that
brought us here.

The five non-GC candidates I'd hold in mind the whole time: safepoint TTSP, connection
pool exhaustion — which for us is Topic 55's transaction-holding-a-connection problem —
a downstream dependency's own tail, container CPU throttling, and disk or page-cache
stalls. And if we have coordinated omission in our load test, our recorded p99 may
have been wrong all along."

**What separates them:** the mid answer treats GC as the assumption and flags as the
fix. The senior answer treats GC as a **hypothesis to falsify first**, distinguishes GC
pause from stop-the-world, names specific log evidence for specific causes, derives
quantities rather than reading dashboards, and holds a list of alternative causes. The
step that most reliably impresses is step 2 — very few candidates volunteer that TTSP
is a separate number.

**Interviewer's follow-up:** *"You said you'd correlate. What if the spikes are every
30 seconds, exactly?"* — Something periodic. A scheduled job, a metrics scrape, a cache
refresh, a `System.gc()` from a JMX console or an RMI distributed-GC timer. Grep the GC
log for `System.gc()` as a cause. `-XX:+DisableExplicitGC` suppresses it, but I'd find
the caller first, because a library calling `System.gc()` on a timer is telling you
something about the library.

---

### Q2 — "What actually happens when you set `-XX:MaxGCPauseMillis=50`?"

**MID-LEVEL answer:** "It tells G1 to keep GC pauses under 50 milliseconds. It's a
target — G1 tries to hit it but can't always."

**SENIOR answer:** "It's an input to G1's control loop, and the mechanism by which G1
responds is the part that matters: **it shrinks the collection set.** G1 keeps a model
of how long it takes to evacuate a region, and each pause it chooses how many regions to
put in the collection set so the predicted time fits the goal. A smaller goal means
fewer regions per pause, which means less memory reclaimed per pause. It also shrinks
the young generation, so collections become more frequent.

That gives you two failure modes. First, more frequent pauses can mean **more total GC
overhead**, because each pause has fixed costs — root scanning, termination — that you
now pay more often. Second, and much worse: if the reclaim rate falls below the
allocation rate, free regions decline over time. When G1 goes to evacuate and there's
nowhere to copy to, you get evacuation failure — to-space exhaustion — and the recovery
is a Full GC that is orders of magnitude longer than the goal you set. So an aggressive
pause goal can be the direct cause of the multi-second pause it was supposed to prevent.

The leading indicator is the free-region count in `-Xlog:gc+heap=debug`, not the pause
duration. A change that lowers your mean pause and lowers your free-region floor is a
change that's about to hurt you.

Practically: I leave it at the default of 200 unless I have a measured reason. If my
latency budget genuinely needs sub-10-ms pauses, that's a collector-choice question —
ZGC or Shenandoah — not a G1 tuning question, and it comes with its own throughput
cost."

**What separates them:** "it's a target" is the trivia. "G1 meets it by collecting fewer
regions, and that can starve it into a Full GC" is the mechanism. Naming free-region
count as the leading indicator is the operational maturity, and escalating from "tune
G1" to "choose a different collector" is the judgment.

**Interviewer's follow-up:** *"So what do you set it to?"* — The default, until the
baseline says otherwise. And I'd rather change the application's allocation behaviour
than the goal, because a flag redistributes the cost and allocating less removes it.

---

### Q3 — "What is a humongous allocation and why should I care?"

**MID-LEVEL answer:** "It's a large object that goes straight to old generation instead
of eden. G1 handles them specially."

**SENIOR answer:** "It's any single allocation whose total size — including the header,
the array-length word, and 8-byte alignment padding — is at least **half a G1 region**.
Not 'large' in the abstract: exactly half a region, and the region size is derived from
your heap size, so **the threshold moves when you change `-Xmx`**.

Three consequences I care about:

**It goes into contiguous old-gen regions.** Not eden, not a TLAB. It's old the instant
it's born, it never gets an age, and the tail of the last region it occupies is wasted —
a 1 MB array on a 1 MB region size takes two regions because of the header.

**It reintroduces fragmentation.** G1's evacuating young collector normally makes
fragmentation a non-issue, because everything is copied. But a humongous object needs a
*contiguous run* of free regions. You can have plenty of free memory and still fail to
place one, and the recovery is a compacting Full GC.

**Reclaim is conditional.** Modern G1 has eager reclaim — it can free a humongous region
during a young collection if it can cheaply prove the object is unreachable, which
covers a `byte[]` referenced only from a stack local. But one incoming reference from
old gen disqualifies it, and then it waits for a concurrent cycle. I never assume eager
reclaim saved me; I check `-Xlog:gc+humongous=debug`.

The reason I actually care in production: this is invisible in every ordinary tool. APM
shows heap usage, metrics show pause time, neither shows you that a 700 KB response
buffer in one low-traffic endpoint is driving old-gen growth for the whole process. The
only place it's visible is the GC log with humongous debug on.

And the sharpest version of the trap: the same jar, same code, same payload, deployed
with a smaller heap, starts doing humongous allocation it wasn't doing before. That's a
production incident with no code change, and almost nobody predicts it."

**What separates them:** the exact definition ("half a region", including header and
padding), the fact that the threshold is heap-size-dependent, contiguity and
fragmentation, the conditionality of eager reclaim, and the operational point that it's
invisible without the specific log tag. The "same jar, smaller heap" observation is the
one that shows production experience.

**Interviewer's follow-up:** *"How would you find them in a running service?"* —
`-Xlog:gc+humongous=debug` to identify that it's happening and at what rate, then
async-profiler in allocation mode (`-e alloc`) filtered to large arrays to find the
allocation site. The GC log tells you *that*; the allocation profile tells you *where*.

---

### Q4 — "Walk me through a G1 concurrent cycle. Which phases stop the world?"

**MID-LEVEL answer:** "G1 marks concurrently while the app runs, so it doesn't pause.
There's an initial mark and a remark that are stop-the-world, and then it collects."

**SENIOR answer:** "The cycle starts when old-gen occupancy crosses the IHOP threshold —
default 45%, but adaptive by default, so the real trigger is a control loop rather than
a fixed number. Setting `InitiatingHeapOccupancyPercent` explicitly turns the adaptation
off, which is a second change people don't realise they're making.

The phases: **Concurrent Start** is stop-the-world, and it's piggybacked on a young
collection, so you see it in the log as a young pause tagged `(Concurrent Start)` — the
root marking is essentially free because you were stopping anyway. Then **root region
scanning** runs concurrently over the survivor regions, and it must complete before the
next young collection can start; if it's slow, you get a mysteriously long young pause
right after the concurrent start. Then **concurrent marking** traces the graph.
**Remark** is stop-the-world and usually the longest STW phase of the cycle — it drains
the SATB buffers, does reference processing and class unloading. **Cleanup** has a brief
STW part for liveness accounting; regions that are 100% garbage get freed right there
without any evacuation. Then **mixed collections** run — ordinary young collections that
also include the emptiest old regions, which is how old gen gets reclaimed
incrementally instead of in one Full GC.

Two things I watch. First, **SATB means floating garbage**: anything that becomes
unreachable after marking starts is not reclaimed in that cycle, so occupancy after a
cycle can be higher than your live-set estimate predicts. That's expected, not a bug.
Second, **the cycle can lose the race** — if allocation fills old gen before marking
completes, G1 has no choice but a Full GC. The fixes are a bigger heap, an earlier IHOP,
more concurrent GC threads, or the real fix, which is allocating less."

**What separates them:** knowing initial mark is piggybacked on a young collection;
knowing root region scan blocks the next young collection; knowing Remark rather than
initial mark is usually the long STW phase; understanding SATB and floating garbage; and
naming "the cycle loses the race" as the specific failure with its specific fixes.

**Interviewer's follow-up:** *"You said Remark can be long. What makes it long?"* —
Reference processing. Soft, weak and phantom references, and the `Cleaner`s behind
`DirectByteBuffer` (Topic 80). A cache built on `WeakReference` can make Remark
disproportionately expensive. `-XX:+ParallelRefProcEnabled` parallelises it, and
`-Xlog:gc+ref=debug` tells you whether that's actually where the time goes — which you
check before setting the flag, not after.

---

### Q5 — "You inherit a service with fifteen GC flags. What do you do?"

**MID-LEVEL answer:** "I'd research each flag and see if it makes sense, then remove the
ones that don't."

**SENIOR answer:** "I'd delete almost all of them, and I'd do it in a way that's safe to
revert.

The reasoning: GC flags are not additive and they're not portable. Every one interacts
with heap size, live set, allocation rate, CPU quota and the other flags. A set of
fifteen was almost certainly accumulated one incident at a time, each addition tested
against a different workload on different hardware, and nobody has ever tested the
combination. Some of them are actively fighting each other — `ParallelGCThreads=8` on a
2-vCPU container means eight GC threads contending for two cores, which makes pauses
longer, not shorter. `InitiatingHeapOccupancyPercent` set explicitly has silently
disabled adaptive IHOP.

The process: first, capture the current behaviour as a baseline under the real load
profile — percentiles, throughput, Full GC count, allocation rate, promotion rate. Then
strip to a minimal set: `-Xms` equal to `-Xmx`, an explicit collector so nobody is
surprised by a container-driven default, and GC logging to a rotating file. Re-run the
identical load. Compare.

Most of the time the stripped configuration is as good or better, because G1's defaults
are the product of far more tuning than any of us have done. Where it's worse, I add
back **one** flag, re-measure, and write a one-line justification in the deployment
manifest next to it. A flag without a written justification and a measurement is
technical debt with a performance cost.

The one thing I'd check before any of this: what collector are we actually getting? In
a container with a small CPU quota the JVM can select Serial GC, and then half those G1
flags are being silently ignored. `jcmd <pid> VM.flags -all` settles it in one command."

**What separates them:** treating flag removal as the default action rather than flag
research; understanding that flags interact; naming a specific self-defeating
combination; insisting on a baseline before and after; requiring written justification;
and checking that the collector is even the one the flags apply to.

**Interviewer's follow-up:** *"What if the team won't let you remove them?"* — Then I'd
remove them in a canary, one deployment at a time, with the baseline comparison as the
evidence. And I'd ask whoever added each flag what measurement motivated it. Usually
nobody remembers, and that answer is itself the argument.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. G1 meets a pause goal by shrinking the collection set. Derive, from that one fact
   alone, why an aggressive pause goal can cause a **longer** pause than the default —
   without recalling the term "evacuation failure" from the text.

2. The humongous threshold is half a region, and the region size is derived from the
   heap size. A service is deployed to staging with `-Xmx8g` and to a cost-optimised
   production cluster with `-Xmx2g`. Predict a class of bug that appears only in
   production, with no code difference. Then design a startup assertion that would have
   caught it.

3. G1's write barrier costs your application instructions on **every reference store**,
   whether or not a collection ever happens. Given that, explain why G1 has lower raw
   throughput than Parallel GC on a batch job — and why that is still the right trade
   for `orderflow`.

4. SATB marking means an object that dies after marking starts is not reclaimed in that
   cycle. Construct a workload where this makes old-gen occupancy look like a leak when
   it is not. What single measurement distinguishes the two?

5. A young pause's cost is dominated by Object Copy, and Object Copy scales with the
   live set. Two services have identical allocation rates. One has young pauses ten
   times longer. Without seeing either service, name three things that could differ, in
   order of likelihood.

6. Eager reclaim can free a humongous region during a young collection, but only when
   the object has no incoming references from outside the young generation. Explain why
   putting a large `byte[]` into a `ConcurrentHashMap` field on a singleton bean defeats
   it, and why putting the same array in a local variable does not.

7. You are asked to reduce p99 by 30%. You may change the collector, the heap size, the
   flags, or the application. Rank those four levers by expected effect **and** by risk,
   and defend the ordering. Then say what evidence would change your ranking.

---

## Quick reference card

### JVM flags — G1

| Flag | Default | What it does | Set it? |
|---|---|---|---|
| `-XX:+UseG1GC` | on for server-class machines | Selects G1 | **Yes, explicitly** — never rely on the machine-class heuristic in a container |
| `-Xms` / `-Xmx` | derived from `MaxRAMPercentage` | Initial / maximum heap | **Yes, and set them equal** — avoids resizing pauses, makes region sizing deterministic |
| `-XX:MaxGCPauseMillis` | 200 | Pause-time **goal** | Only with a measurement. Lowering it shrinks the collection set. |
| `-XX:G1HeapRegionSize` | computed (~heap/2048, power of two) | Region size; halve it for the humongous threshold | Only to move the humongous threshold, and only after measuring the cost |
| `-XX:InitiatingHeapOccupancyPercent` | 45, **adaptive by default** | When a concurrent cycle starts | Rarely — setting it disables the adaptation |
| `-XX:+G1UseAdaptiveIHOP` | on | Adapt the IHOP trigger at runtime | Leave on |
| `-XX:+G1EagerReclaimHumongousObjects` | on | Free unreferenced humongous regions at young collections | Leave on; flip it off only as an A/B control in a drill |
| `-XX:G1NewSizePercent` / `-XX:G1MaxNewSizePercent` | 5 / 60 | Young-gen bounds as a % of heap | Almost never — this fights the pause-goal control loop |
| `-XX:ParallelGCThreads` | derived from visible CPUs | STW GC worker threads | Only when the container's CPU quota misleads the JVM (Topic 82) |
| `-XX:ConcGCThreads` | ~1/4 of `ParallelGCThreads` | Concurrent marking threads | Only when concurrent cycles are losing the race |
| `-XX:G1ConcRefinementThreads` | derived | Threads draining dirty cards into RSets | Rarely; check `gc+remset=debug` first |
| `-XX:+ParallelRefProcEnabled` | varies by version | Parallelise reference processing in Remark | Only if `gc+ref=debug` shows Remark dominated by reference processing |
| `-XX:+DisableExplicitGC` | off | Turns `System.gc()` into a no-op | Only after you've found the caller |
| `-XX:+HeapDumpOnOutOfMemoryError` | off | Dump on OOM | **Always, in production.** Topic 79. |

> **Version note, one line:** G1 remains the default collector on server-class machines
> on both JDK 21 and JDK 25; the region-size cap and some internal defaults have moved
> across versions, and JDK 22 introduced region pinning for G1 (JEP 423) which changes
> JNI-critical behaviour. **Settle any default on your runtime with
> `java -XX:+PrintFlagsFinal -version` or `jcmd <pid> VM.flags -all` rather than trusting
> any document, including this one.**

### Diagnostic commands

```bash
# What collector and what flags am I actually running?
jcmd <pid> VM.flags -all
java -XX:+PrintFlagsFinal -version | grep -E "UseG1GC|G1HeapRegionSize|MaxGCPauseMillis"

# Region breakdown, right now.
jcmd <pid> GC.heap_info

# Class histogram (live objects only — note this triggers a full GC).
jcmd <pid> GC.class_histogram

# The full GC log, to a rotating file, with the decorators you need.
-Xlog:gc*,gc+heap=debug,gc+humongous=debug:file=gc.log:time,uptime,level,tags:filecount=5,filesize=50m

# Add phase breakdown when you need to know WHERE a pause went.
-Xlog:gc*,gc+phases=debug,gc+heap=debug:file=gc.log:time,uptime,level,tags

# Remembered-set and reference-processing detail.
-Xlog:gc+remset=debug,gc+ref=debug:file=gc-detail.log:time,uptime,level,tags

# Separate GC pause from stop-the-world duration. Topic 73.
-Xlog:safepoint*:file=safepoint.log:time,uptime,level,tags

# Turn logging on at runtime, without a restart.
jcmd <pid> VM.log output=/tmp/gc-live.log what='gc*,gc+heap=debug' decorators='time,uptime,level,tags'
jcmd <pid> VM.log disable

# Triage a log you were handed.
grep -c 'Pause Full'                    gc.log
grep -cE 'To-space exhausted|Evacuation Failure' gc.log
grep -c 'humongous'                     gc.log
grep    'System.gc()'                   gc.log
grep    'GC(1234)'                      gc.log     # every line of one collection
```

### How to read a G1 GC log line — field guide

```
[<wall-clock>][<uptime>s][<level>][<tags>] GC(<id>) <PauseType> (<Cause>) <before>-><after>(<total>) <duration>ms
```

***Illustration of the format, not captured output.***

| Position | Field | How you use it |
|---|---|---|
| 1 | wall clock | correlate with the load generator and APM |
| 2 | uptime | compute intervals between collections; the denominator of allocation rate |
| 3 | level | `info` for summaries, `debug` for the tags you raised |
| 4 | tags | **grep on this**, not on message text |
| 5 | `GC(n)` | groups all lines of one collection |
| 6 | pause type | Young / Young (Concurrent Start) / Young (Mixed) / Remark / Cleanup / **Full** |
| 7 | cause | **the diagnostic field** — evacuation pause, humongous allocation, metadata threshold, `System.gc()`, periodic |
| 8 | `before->after(total)` | how much was reclaimed; `after` is the live set plus floating garbage |
| 9 | duration | GC work at the safepoint. **Not** stop-the-world. TTSP is separate. |

Reading order when you open an unfamiliar log:

1. `grep -c 'Pause Full'` — if nonzero, start there and read backwards.
2. `grep -cE 'To-space exhausted|Evacuation Failure'` — nonzero means the heap ran out
   of copy space; the region counts before it tell you why.
3. `grep -c humongous` — nonzero means read the allocation sites next.
4. Longest pause → `grep 'GC(n)'` for the whole group → `gc,phases` for where the time
   went → `gc,heap` for the region state that explains it.
5. Allocation rate and promotion rate from consecutive young collections.
6. Only now consider flags.

### Gotchas checklist

- [ ] Print the collector. Never infer it. Containers change the default.
- [ ] Compute the humongous threshold for your `-Xmx` and write it down.
- [ ] `MaxGCPauseMillis` is a goal met by collecting **less**, not a limit.
- [ ] `Pause Full` is always an incident. Explain every one.
- [ ] GC pause ≠ stop-the-world. TTSP is a separate number (Topic 73).
- [ ] `-Xms` = `-Xmx` in a container. Resizing pauses are avoidable.
- [ ] Change one flag at a time and compare against the recorded baseline.
- [ ] A flag change smaller than your run-to-run noise is not a change.
- [ ] Allocating less beats every flag. Flags redistribute; code removes.
- [ ] Old-gen growth without humongous lines is retention (Topic 79), not G1.

---

## When would I use this at work?

**1. A p99 regression appears after a release that "only added an endpoint".**
You correlate the GC log with the latency histogram, see `Pause Full` with a humongous
allocation cause, grep the humongous lines, and connect them to the new endpoint's
response buffer. Ninety seconds to a diagnosis that otherwise gets attributed to the
database because the database is always the first suspect. The fix — stream the response
instead of buffering it — is a five-line change that would never have been found by
tuning.

**2. Reviewing a deployment manifest that adds `-XX:MaxGCPauseMillis=50`.**
You ask one question: what measurement motivated it, and what is the free-region floor
under load? If there is no answer, you block the change and explain the mechanism —
that G1 meets the goal by shrinking the collection set, and that a starved collection
set produces exactly the multi-second pause the flag was meant to prevent. This is a
30-second review comment that prevents a 3am page.

**3. Capacity planning for a cost-optimised cluster.**
Finance wants to halve the memory limit on every pod. You can state, with numbers from
your own service, what happens: the region size halves, the humongous threshold halves,
these three allocation sites in the codebase cross it, concurrent cycles get more
frequent, and here is the measured p99 at each candidate heap size. That is a
capacity-model conversation, not an opinion, and it is what Phase 12's capacity-model
artefact is made of.

---

## Connected topics

**Prerequisites:**

- **01 — Boxing and the Integer cache**: every boxed value is an allocation. `Map<Long,
  Long>` counters and `List<Integer>` are allocation-rate problems, and allocation rate
  is the input to everything in this document.
- **21 — Lambdas and `invokedynamic`**: a capturing lambda allocates per evaluation. In
  a hot stream pipeline that is a measurable contribution to the allocation rate you
  derive from the GC log.
- **25 — Parallel streams and the common ForkJoinPool**: parallel work multiplies
  allocation across threads, and the common pool's size is derived from
  `availableProcessors()`, which containers change (Topic 82).
- **65 — The load-testing gate**: the recorded baseline in `/docs/java/baselines/` is
  the control for every measurement here. Without it, "faster" is an opinion.
- **68 — Heap generations and TLABs**: allocation is a pointer bump in a TLAB inside
  eden — **except** for humongous allocation, which is the exception this document is
  about. Promotion is the mechanism that fills old gen and drives concurrent cycles.
- **69 — Object layout and compressed oops**: the humongous threshold is measured
  against the object's *real* size — header, array-length word, and alignment padding
  included. A 1 048 576-byte array is more than 1 MB on the heap.
- **70 — GC fundamentals**: cost tracks the live set, not the garbage. Every pause-time
  observation in this document is an instance of that principle.

**This unlocks:**

- **72 — ZGC and Shenandoah**: what to do when G1's pauses are genuinely irreducible.
  Read it immediately after this one; the choice only makes sense once you know what G1
  does and what it costs.
- **73 — Safepoints and TTSP**: the number the GC log does **not** contain. Read this
  before you conclude any pause was GC's fault.
- **74 — JIT and tiered compilation**: the write barrier is code C1 and C2 emit. GC
  behaviour and compilation are the same story told from two ends.
- **75 — Escape analysis**: an object that is scalar-replaced never reaches the heap at
  all — the cheapest possible allocation-rate reduction, and the most fragile.
- **77 — JMH**: the only correct way to measure the allocation-shape questions this
  topic raises, and the reason the naive `nanoTime` loop above is fiction.
- **78 — Profiling**: async-profiler in allocation mode names the *site* that the GC
  log only tells you exists. GC log = that it happens; allocation profile = where.
- **79 — Heap dumps and MAT**: when old gen grows and there are no humongous lines, the
  answer is retention, and the dominator tree is the tool.
- **80 — Off-heap memory**: moving large buffers off-heap removes them from G1's
  problem — and creates a new failure mode where the container OOM-kills you with a
  healthy heap.
- **82 — Containers and cgroups**: the CPU quota sizes `ParallelGCThreads` and
  `ConcGCThreads`; the memory limit sizes the heap and therefore the region size and
  therefore the humongous threshold. This document's entire arithmetic is downstream of
  Topic 82's settings.
- **83 — GraalVM native image**: a different memory-management story entirely, with no
  JIT and a different collector. The contrast sharpens both.
- **96 — False sharing**: GC worker threads and refinement threads contend for cache
  lines too. The same hardware reality, one level down.
- **101 — Virtual threads**: thousands of virtual threads change the root-scanning cost
  of every pause, and pinning interacts with safepoints. Ext Root Scanning in the phase
  breakdown is where you will see it.

---

*Java baseline 21, running on JDK 25. Three things in this document are deliberately
hedged rather than asserted: the maximum permitted `G1HeapRegionSize` on your specific
JDK, the precise conditions under which eager reclaim of humongous regions succeeds on
your build, and the exact JDK in which the higher region-size cap landed. Each has a
command in the Hands-on section that settles it on your machine in under a minute. No
pause duration, throughput figure or percentile in this document was measured — every
number you act on must come from your own log against your own baseline. What has been
stable since JDK 9 and will still be true at 2am: regions, the half-a-region humongous
rule, the pause goal being met by shrinking the collection set, and `Pause Full` being
a failure rather than a feature.*
