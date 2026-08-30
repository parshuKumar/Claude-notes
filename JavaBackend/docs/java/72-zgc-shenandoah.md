# 72 — Low-Pause Collectors: ZGC (Generational), Shenandoah, and Choosing One

## Phase: 8 — JVM Internals
## Category: ELITE
## Java baseline: 21  |  Notes features from: 21 / 25
## Project spine: this is how you decide, with evidence from the Topic 65 baseline, whether `orderflow` should run on G1 or on a concurrent low-pause collector — and how you recognise the far more common case where the answer is "neither; your pauses were never the problem".

---

## Mechanical statement

Read this three times. Every section below is an elaboration of it.

> **ZGC stores metadata bits inside the object reference itself — a "coloured
> pointer" — and compiles a LOAD BARRIER into every read of a reference field.**
> The barrier is a few instructions that run on the mutator thread, on every
> reference load, forever. It inspects the colour of the pointer it just loaded. If
> the colour is stale — the object has been moved, or has not yet been marked — the
> barrier fixes it up *there and then*, on the application's thread, and hands your
> code a correct reference.
>
> **That is the whole trick.** Because a thread that loads a reference repairs it
> itself, the collector can move objects around **while the application is running**.
> It never needs a stop-the-world phase to update references.
>
> **The consequence:** ZGC's stop-the-world pauses do not contain marking, do not
> contain copying, and do not contain reference fixup. They contain only root-set
> work. **Pause time therefore becomes independent of heap size and of live-set
> size** — the two quantities that dominate every G1 pause.
>
> **The price is paid in two currencies, both of them continuous:**
> **throughput**, because every reference load in your application now costs extra
> instructions; and **CPU and memory headroom**, because collection runs concurrently
> with allocation and must finish before you run out of room.
>
> **Shenandoah reaches the same destination by a different road:** a load-reference
> barrier and a forwarding pointer, rather than coloured pointers and multi-mapped
> memory. Same trade shape. Different implementation, different vendor availability,
> different flags.

Five consequences follow directly, and you should be able to derive each one:

1. If your p99 is dominated by something that is not a GC pause, a collector whose
   only advantage is shorter pauses **buys you nothing and costs you throughput**.
2. A concurrent collector can *lose a race* it never had to run before: allocation
   during collection. The failure mode is not a long pause — it is an **allocation
   stall**, where application threads block waiting for free memory.
3. Because pause time is decoupled from heap size, a very large heap becomes viable
   in a way it never was under G1. That is ZGC's real headline feature, not "0.5 ms".
4. Coloured pointers need all 64 bits of a reference, so **compressed oops are off**
   under ZGC. Every reference in your heap gets bigger. On a small heap that is a
   material footprint cost.
5. **The safepoint still exists.** ZGC's pauses are short *at* the safepoint. The time
   to *reach* the safepoint is unchanged, and it is a separate number. Topic 73.

---

## The bridge from what you know

### NO TYPESCRIPT ANALOGUE.

State this plainly before anything else, because half of this document is about a
decision you have never been allowed to make.

**There is exactly one V8 garbage collector.** You do not select it. You do not
compare it against alternatives. You do not have a flag that swaps its fundamental
algorithm. You have never had to weigh pause time against throughput, because the
choice was made for you by the runtime authors and never surfaced.

Concretely, four things in this document have **no counterpart whatsoever** in Node:

**1. Collector choice has no analogue.**
`-XX:+UseG1GC` versus `-XX:+UseZGC` versus `-XX:+UseShenandoahGC` versus
`-XX:+UseParallelGC` is a real architectural decision with real, measurable, opposite
consequences. There is no `--gc=` flag in Node that does anything comparable. This is
not "Java has more knobs" — it is a different category of engineering decision that
has simply never been on your desk.

**2. A pause-time GOAL has no analogue.**
Topic 71 already made this point for G1's `MaxGCPauseMillis`. ZGC sharpens it: ZGC
does not take a goal at all, because it does not need one. Its pauses are structurally
short. Understanding *why one collector takes a goal and the other does not* is the
whole architectural story.

**3. Region-based, concurrently-relocating collection has no analogue.**
V8 compacts, and it does so in a stop-the-world pause. Nothing in V8 moves objects
while your JavaScript is running and repairs the references as you touch them. The
concept of a *barrier your compiled code executes on every reference load* does not
exist in your mental model of JavaScript at all.

**4. Trading throughput for latency, deliberately and measurably, has no analogue.**
This is the mature version of the point. You are not choosing a "better" collector.
You are choosing **which resource to spend**. ZGC spends CPU cycles and memory to buy
predictable latency. Parallel GC spends latency to buy throughput. G1 sits between
them. Node gives you one point on that curve and no way to move along it.

| You know | Java's low-pause collectors | Verdict |
|---|---|---|
| One GC, not selectable | G1 / Parallel / Serial / ZGC / Shenandoah, one flag apart | **NO ANALOGUE** |
| GC pauses are a fact of life you cannot influence | Pause behaviour is an engineering variable you choose | **NO ANALOGUE** |
| Write barriers (V8 does have these, invisibly) | G1 write barrier, ZGC load barrier, Shenandoah LRB | **PARTIAL** — the concept exists in V8's remembered sets, but you have never seen its cost, and a *load* barrier is a different and heavier animal than a *write* barrier |
| "Objects die young" | Weak generational hypothesis — still true, which is why generational ZGC exists | **HONEST ANALOGUE** |
| `--max-old-space-size` | `-Xmx`, `-XX:SoftMaxHeapSize`, `-XX:MaxRAMPercentage` | **PARTIAL** — and heap is only part of the JVM footprint (Topic 80) |
| Latency budgets, percentiles, tail latency | Identical concepts | **HONEST ANALOGUE** — and this is your strongest transferable asset in this topic |

### What DOES transfer, and it is the most important thing you own

Your system-design background. You already think in **latency budgets** and
**percentiles**. You already know that a p99 is a composition of every stage in a
request's path, and that improving a stage that contributes 2% of the budget improves
the budget by at most 2%.

That is exactly — *exactly* — the reasoning this topic requires. The single most
common mistake engineers make with ZGC is adopting it without asking what fraction
of p99 the GC pause actually was. You already have the instinct that answers this. Use
it, and you will out-reason most Java engineers on this specific decision.

---

## What is this?

### The problem all of this solves

G1 is a **mostly stop-the-world** collector. Marking runs concurrently, but every
actual reclamation — every young collection, every mixed collection — is a pause in
which G1 copies live objects and updates the references that point to them. That
copying work scales with the **live set**: the more surviving data, the longer the
pause.

That is a structural limit. You can shrink G1's pauses by collecting fewer regions
(Topic 71's Trap 1, and it backfires), or by having a smaller live set, but you cannot
make a pause that copies 400 MB of live objects take half a millisecond. The work is
the work.

**The low-pause collectors remove the copying from the pause entirely.** They relocate
objects while your application threads are running. To do that safely, they need a way
to guarantee that any thread which touches a reference to a moved object gets the new
address, not the old one. That guarantee is a **barrier**: compiler-emitted code that
runs on your thread as part of the memory access.

### ZGC in one paragraph

ZGC is a concurrent, region-based, compacting collector. It uses **coloured pointers**:
a 64-bit object reference in which some bits are not address bits but metadata —
"is this object marked", "has this object been relocated". Every time compiled code
loads a reference from a field, a **load barrier** checks those bits. If they indicate
the pointer is stale, the barrier does the work needed (marks the object, or looks up
its new address in a forwarding table and *rewrites the field it just read from* —
"self-healing") and then returns a good pointer. Marking, relocation and reference
fixup all happen concurrently. Pauses contain only root scanning and phase
transitions.

### Shenandoah in one paragraph

Shenandoah is also a concurrent, region-based, compacting collector, developed at
Red Hat. Rather than colouring the pointer, it uses a **forwarding pointer** in the
object header plus a **load-reference barrier (LRB)**: on loading a reference,
compiled code checks whether the object has been evacuated and, if so, follows the
forwarding pointer and self-heals the source field. The observable trade is the same
as ZGC's: pauses independent of heap size, throughput cost on every reference load.

### The honest comparison table

This is the table you should be able to reproduce from memory in an interview.

| | Serial | Parallel | G1 | ZGC | Shenandoah |
|---|---|---|---|---|---|
| Pause scales with | live set | live set | live set (per collection set) | **root set only** | **root set only** |
| Compaction | full, STW | full, STW | evacuating, STW | **concurrent** | **concurrent** |
| Barrier on the mutator | none meaningful | cheap write barrier | write barrier (card marking + SATB pre-barrier) | **load barrier on every reference load** | **load-reference barrier** |
| Generational | yes | yes | yes | yes, from JDK 21 as opt-in; default in JDK 23+ | experimental generational mode in recent JDKs — verify on your build |
| Compressed oops | yes | yes | yes | **no** | yes |
| Typical strength | tiny heaps, single CPU | batch throughput | general-purpose default | very large heaps, tight tail-latency budgets | tight tail-latency budgets |
| Typical weakness | everything else | pause length | pauses grow with live set | throughput, footprint, headroom | throughput, vendor availability |
| Available in every JDK build? | yes | yes | yes | yes | **NO — check your vendor** |

That last row is a trap that catches people in production. **Shenandoah is not
present in every JDK distribution.** Oracle's own JDK builds have historically not
shipped it; OpenJDK builds from Red Hat, Adoptium/Temurin and others do. If
`-XX:+UseShenandoahGC` produces "Unrecognized VM option", the collector is not in
your binary and no flag will conjure it. Verify before you design around it:

```bash
java -XX:+UseShenandoahGC -version
java -XX:+PrintFlagsFinal -version | grep -iE "UseZGC|UseShenandoahGC|ZGenerational"
```

### Version reality, stated carefully

This is the part of the topic where a confidently wrong sentence will cost you an
interview, so read the hedges as seriously as the claims.

- **JDK 21 (your baseline).** `-XX:+UseZGC` selects ZGC. On JDK 21 that alone gives
  you the **non-generational** ZGC; **generational ZGC is opt-in** with
  `-XX:+ZGenerational` (JEP 439 was integrated in 21 as a separate mode). If you
  benchmark ZGC on JDK 21 and forget `-XX:+ZGenerational`, you are benchmarking a
  collector that treats every object as old — and on a request-serving workload where
  almost everything dies young, that is a materially different and usually worse
  collector.
- **JDK 23+.** Generational became ZGC's **default** mode (JEP 474), so
  `-XX:+UseZGC` alone gives you generational ZGC and `-XX:-ZGenerational` was the way
  back.
- **JDK 24+.** The non-generational mode was **removed** (JEP 490), and
  `-XX:+ZGenerational` became obsolete — passing it may produce a warning.
- **JDK 25 (your runtime).** Generational, and the `ZGenerational` flag is not
  something you should be setting.

> **Flagged uncertainty, one line:** I am confident about the *pattern* above — opt-in
> on 21, default on 23+, non-generational removed on 24+ — and about the direction of
> travel. I am **not** going to assert the precise warning text, the exact deprecation
> semantics, or the behaviour of every vendor build. **Settle it on your runtime:**
>
> ```bash
> java -XX:+UseZGC -XX:+PrintFlagsFinal -version | grep -i ZGenerational
> jcmd <pid> VM.flags -all | grep -iE "UseZGC|ZGenerational|SoftMaxHeapSize"
> ```
>
> Read what your JVM prints. Do not trust this document, or any blog post, over the
> flag dump from the binary you are actually deploying.

**Why generational matters so much that it changed the default:** the weak
generational hypothesis is still true. Most `orderflow` objects — request DTOs,
Hibernate snapshots, boxed `Long`s (Topic 01), lambda captures (Topic 21) — die within
one request. A non-generational collector must trace the *entire* live set on every
cycle to discover that. A generational one collects a small young space cheaply and
frequently, and touches the old generation rarely. Non-generational ZGC bought
low pauses at a significant throughput cost that generational ZGC substantially
reduces. **This is why a 2019 blog post about ZGC's throughput is not evidence about
the ZGC you will run.**

---

## Why does it matter?

**1. Because "sub-millisecond pauses" is a marketing claim that answers a question you
probably do not have.**

The number is real. It is also frequently irrelevant. If `orderflow`'s `POST /orders`
p99 is composed of a database round trip, an inventory check, a wallet debit and a
call to a payment gateway, and the gateway alone contributes hundreds of milliseconds,
then eliminating a 40 ms GC pause that occurs in a fraction of requests changes p99 by
approximately nothing — while the load barrier taxes **every single reference load in
the entire service, forever.** That is a losing trade, and it is the single most
common way this topic is got wrong.

**2. Because concurrent collection introduces a failure mode you have never seen.**

Under G1, the failure mode is a long pause. Under ZGC, the failure mode is an
**allocation stall**: the collector is running, your threads want memory, there is
none free yet, and they *block*. Your latency graph shows a stall that no GC pause
accounts for, because it was not a pause — it was your own thread waiting for a page.
The fix is not a pause-goal flag; it is headroom, CPU, or a lower allocation rate.
You have to recognise a symptom you have never met.

**3. Because collector choice is a capacity-planning decision, not a preference.**

ZGC needs CPU to run its concurrent work. In a container with `--cpus=2` (Topic 82),
the concurrent GC threads compete with the request-handling threads for the same
quota. On a generously-provisioned host, ZGC's concurrent work is close to free
because there are spare cores. In a tightly-packed Kubernetes cluster, **it is not
free, and the cost lands directly on your throughput**. The same flag has opposite
economics in two deployments.

**4. Because this is exactly the decision a senior engineer is asked to defend.**

"Should we move to ZGC?" is a real question you will be asked. The mid-level answer is
a preference. The senior answer is a latency budget, a measurement, and a stated cost.
This document exists to make you able to give the second one.

**5. Because on `orderflow` you have a recorded baseline, which makes every claim
falsifiable.**

`/docs/java/baselines/` has p50/p95/p99/p999, throughput and error rate under a
defined load profile against a defined dataset. That is the difference between an
engineering decision and an opinion.

---

## Machine-level reality

### Coloured pointers — what is actually in the 64 bits

A Java object reference on a 64-bit JVM is a 64-bit word. Ordinary collectors use it
as an address (or, with compressed oops, a shifted 32-bit offset from a heap base).

ZGC uses some of those bits as **metadata about the state of the referenced object**.
Conceptually, a ZGC reference is:

```
 63                                                     0
 +---------------+---------------------------------------+
 | colour bits   |            address bits               |
 +---------------+---------------------------------------+
      ^                        ^
      |                        |
      |                        the actual location of the object
      |
      "Marked0" / "Marked1" / "Remapped" / "Finalizable" — the object's
      state with respect to the current GC cycle
```

*Illustration of the concept, not a bit-exact layout of any specific JDK.* The exact
bit positions, the number of colour bits, and their meanings have changed between ZGC
versions — notably between the non-generational and generational implementations.
**Do not memorise a bit layout.** Memorise the mechanism: *the pointer carries state,
and the barrier reads that state.*

Three hard consequences of colouring the pointer:

- **ZGC requires a 64-bit platform.** There are no spare bits in a 32-bit reference.
- **Compressed oops are disabled.** Topic 69 taught you that below roughly 32 GB of
  heap, HotSpot stores references as 32-bit shifted values, halving the cost of every
  reference field. Under ZGC you do not get that. Every reference field in every
  object is 8 bytes. **On a small heap this is a real, measurable footprint increase**,
  and it is one of the honest reasons ZGC can be *worse* on a 1–2 GB heap.
- **Some tools get confused.** Anything that reads raw pointer values — old native
  debuggers, some homegrown JNI code — may not expect coloured bits.

### Multi-mapped memory — the classic trick, and an honest caveat

The original (non-generational) ZGC used a beautiful piece of operating-system
abstraction to make coloured pointers dereferenceable without masking.

The idea: map the **same physical memory** at **several different virtual addresses**
— one "view" per colour. Then a coloured pointer, used directly as an address, lands
in the view corresponding to its colour, and still reaches the correct physical bytes.
No masking instruction is needed on the fast path.

```
   physical page P
        ^   ^   ^
        |   |   |
   +----+   |   +----+
   |        |        |
 view       view     view
 "Marked0"  "Marked1" "Remapped"
 (virtual)  (virtual) (virtual)
```

*Illustration of the mechanism, not a memory map from any particular JVM.*

The visible operational consequence: **`top`, `ps` and naive container memory
accounting could report virtual memory sizes far larger than the real heap**, because
the same physical pages appeared at multiple virtual addresses. Engineers regularly
opened tickets about "ZGC using 3x the memory" when RSS was fine and only VSZ was
inflated.

> **Flagged uncertainty, honestly:** multi-mapping is a documented property of the
> original ZGC implementation. **The generational ZGC implementation changed how
> coloured pointers are handled** — my understanding is that it moved away from
> relying on multi-mapping in favour of masking in the barrier, but **I am not
> confident enough in the details on your specific JDK to state it as fact.** What
> you should take away and can rely on: *if you see an alarming virtual-memory figure
> under ZGC, check RSS before you panic, and check what your JDK version actually
> does.* Settle it with:
>
> ```bash
> cat /proc/<pid>/status | grep -E "VmSize|VmRSS"
> jcmd <pid> VM.native_memory summary     # requires -XX:NativeMemoryTracking=summary
> ```
>
> Topic 80 is the full treatment of the heap-versus-footprint question.

### The load barrier — what your code actually pays

This is the cost centre. Understand it precisely.

```java
// what you wrote
Product product = orderLine.product();

// what compiled code effectively does under ZGC
// 1. the ordinary field load
//    tmp = *(orderLine + offset_of_product)
// 2. --- ZGC LOAD BARRIER, emitted by C1/C2 on every reference load ---
//    if (colour_bits_of(tmp) != current_good_colour) {
//        tmp = slow_path(tmp);            // mark it, or follow the forwarding table
//        *(orderLine + offset_of_product) = tmp;   // SELF-HEALING: fix the source field
//    }
// 3. product = tmp
```

*Illustration of the barrier's structure, not generated assembly.*

Four things to notice, because each one is an exam answer:

1. **It is on LOADS, not stores.** G1's barrier fires when you write a reference. ZGC's
   fires when you *read* one. Reference reads are far more common than reference
   writes in almost every program. **This is the structural reason ZGC costs more
   throughput than G1, and G1 costs more than Parallel.**
2. **The fast path is short** — a test and a well-predicted branch — but it is not
   free, and it is in the hottest code you have: field access, array element access,
   iterating a collection, walking a Hibernate entity graph.
3. **Self-healing** means the *second* load of the same field is fast again. So the
   barrier's cost is uneven: high just after a relocation phase, lower as fields get
   healed. This makes ZGC's throughput cost vary over the GC cycle, which makes it
   harder to measure and easier to misreport.
4. **The barrier is why relocation can be concurrent.** Everything ZGC buys you comes
   from this one mechanism. You cannot have the benefit without the cost.

### Where ZGC's pauses actually are

ZGC does still stop the world. Its pauses are short **because of what is in them**,
not because pausing was abolished.

| Pause | What is in it | Scales with |
|---|---|---|
| **Mark Start** | Flip the "good colour", scan thread-local roots, activate barriers | number of threads / root set |
| **Mark End** | Terminate marking, handle references | root set, weak reference count |
| **Relocate Start** | Flip colours again, scan roots, begin concurrent relocation | root set |

None of them contains marking the heap. None contains copying objects. None contains
updating references. That is the entire reason they do not grow with heap size.

**What they DO scale with: the root set.** Thread stacks are roots. Ten thousand
platform threads make root scanning slower than a hundred. That is one of the places
Topic 101 (virtual threads) and Topic 98 (thread counts) meet this topic. Concurrent
thread-stack processing moved most of that work out of the pause in modern JDKs, which
is precisely what took ZGC from "a few milliseconds" to "sub-millisecond" territory.

And — say it again, because it is the thing everyone forgets — **the safepoint
protocol is unchanged.** A sub-millisecond pause is a sub-millisecond pause *once
every thread has arrived*. If one thread takes 900 ms to reach the safepoint, your
application stopped for 900 ms and ZGC's log will still report a tiny pause. **Topic
73. Read it before you conclude anything from a ZGC log.**

### Regions, page sizes, and large objects

ZGC divides the heap into regions it calls **ZPages**, in a small number of size
classes rather than G1's single uniform region size. Small pages hold small objects;
medium pages hold medium ones; large pages hold a single large object each.

This has a genuinely useful consequence for you: **ZGC does not have G1's humongous
allocation problem in the same shape.** Topic 71's entire drill — a `byte[]` just over
half a region going straight to old gen and fragmenting the heap — does not translate
directly. ZGC allocates the large object in a page sized for it.

That does **not** mean large allocations are free. Large pages are not relocated in
the same way, and a workload allocating many large objects still stresses the
allocator. But if your G1 problem was specifically humongous fragmentation, ZGC
addresses it structurally rather than by tuning.

> **Flagged uncertainty:** I am not going to assert exact ZPage size-class boundaries
> for your JDK, because they are implementation detail that has moved. Confirm the
> shape of your heap with `-Xlog:gc+heap=debug` under ZGC and with `jcmd <pid>
> GC.heap_info`, and read what your runtime prints.

### Soft max heap size, uncommitting, and headroom

ZGC has a flag G1 does not, and it matters for containers:

- **`-XX:SoftMaxHeapSize`** — a soft target. ZGC tries to keep heap usage below it,
  collecting more eagerly to do so, but is allowed to exceed it up to `-Xmx` rather
  than throwing `OutOfMemoryError`. This is the flag for "I want to normally use 1 GB
  but I would rather burst than die".
- **`-XX:+ZUncommit` / `-XX:ZUncommitDelay`** — ZGC can return unused heap memory to
  the operating system. Useful for bursty services and for bin-packing. Also a source
  of confusion when RSS drops and someone thinks memory leaked out of the process.

Both are worth knowing because they are levers G1 does not offer you, and both are
worth **printing rather than assuming**:

```bash
jcmd <pid> VM.flags -all | grep -iE "SoftMaxHeapSize|ZUncommit|ZCollectionInterval"
```

### Shenandoah's mechanism, briefly but correctly

Shenandoah's barrier is a **load-reference barrier**: on loading a reference, check
whether the object has been evacuated; if so, follow the forwarding pointer stored in
the object's header and heal the source field. Historically Shenandoah used a
"Brooks pointer" — an extra word prepended to every object — which cost footprint on
every object. Later versions moved the forwarding information into the existing object
header, removing that per-object word.

Shenandoah has modes, selected with `-XX:ShenandoahGCMode=`:

| Mode | What it is |
|---|---|
| `satb` | Snapshot-at-the-beginning marking. The default. |
| `iu` | Incremental update marking. Different floating-garbage characteristics. |
| `passive` | Diagnostic: no concurrent work, degenerate collections only. For debugging, never production. |
| `generational` | A generational mode added in recent JDKs — **experimental; verify availability and status on your build before planning around it.** |

Plus a heuristics selector, `-XX:ShenandoahGCHeuristics=` (`adaptive` by default;
`static`, `compact`, `aggressive` also exist — `aggressive` is a stress-test setting,
not a production one).

**Shenandoah's degenerate and full collections** are its equivalent of G1's evacuation
failure: if the concurrent cycle cannot keep up with allocation, Shenandoah falls back
to a stop-the-world "degenerated" collection, and in the worst case a full GC. Seeing
`Pause Degenerated GC` in a Shenandoah log means the same thing as `Pause Full` in a
G1 log: **the concurrent collector lost a race, and this is an incident to explain.**

### The one number that decides everything: how much of your p99 is GC?

Before any of the machinery above matters, you need this fraction. It is computable
from artefacts you already have.

```
GC contribution to p99  ≈  (pause duration at the relevant percentile)
                            as a share of (end-to-end p99 from the load generator)
```

More usefully, in practice:

1. From the GC log, get the distribution of pause durations over the run.
2. From the load generator, get the end-to-end percentiles.
3. Ask: **if every GC pause became zero, what is the best possible p99?**

That last question is the whole topic in one sentence. If the answer is "p99 improves
by 3%", then ZGC's ceiling is a 3% improvement, and its floor is a throughput
regression on every request. **You have just made the decision, with arithmetic,
before running anything.**

---

## Example 1 — minimal

The smallest experiment that makes the trade visible: a program with a large, mostly
live heap and a steady allocation rate, run under both collectors.

The point is **not** to produce a benchmark number. It is to see, with your own eyes,
that (a) G1's pause durations grow with the live set while ZGC's do not, and (b) the
same program does less total work per second under ZGC.

```java
package com.orderflow.lab.gc;

import java.util.ArrayList;
import java.util.List;
import java.util.Random;

/**
 * A deliberately GC-hostile shape: a LARGE LIVE SET that must be traced and copied,
 * plus a steady stream of short-lived garbage on top of it.
 *
 * args[0] = live set size in MB (approximately)
 * args[1] = seconds to run
 *
 * This is not a benchmark. It is a demonstration rig. The number it prints is a
 * crude work counter used only to compare two runs of the SAME program under two
 * collectors; it is not a throughput measurement and must never be quoted as one.
 */
public final class LiveSetPressure {

    /** Each entry is a small object graph, so the live set has real reference density. */
    record Node(long id, byte[] payload, Node next) { }

    public static void main(String[] args) {
        int liveMb = Integer.parseInt(args[0]);
        int seconds = Integer.parseInt(args[1]);

        // Build a large live set: many objects that stay reachable for the whole run.
        // Reference density matters: a live set of byte[] is much cheaper to trace
        // than a live set of linked objects. Topic 70's point, made concrete.
        List<Node> live = new ArrayList<>();
        int perNode = 4096;
        long targetBytes = (long) liveMb * 1024 * 1024;
        long built = 0;
        Node chain = null;
        long id = 0;
        while (built < targetBytes) {
            chain = new Node(id++, new byte[perNode], chain);
            if ((id & 0x3F) == 0) {          // keep every 64th chain head reachable
                live.add(chain);
                chain = null;
            }
            built += perNode + 64;
        }
        System.out.println("live set built: roughly " + liveMb + " MB in " + id + " nodes");

        // Now churn: allocate short-lived garbage while the live set stays live.
        Random random = new Random(42);
        long deadline = System.currentTimeMillis() + seconds * 1000L;
        long work = 0;
        while (System.currentTimeMillis() < deadline) {
            // Reference-heavy work, so the LOAD BARRIER cost is exercised, not just
            // allocation. This is the part a byte[]-only rig would miss entirely.
            List<Node> sample = live;
            for (int i = 0; i < 1000; i++) {
                Node n = sample.get(random.nextInt(sample.size()));
                while (n != null) {                 // pointer chasing: loads, loads, loads
                    work += n.id();
                    n = n.next();
                }
            }
            byte[] garbage = new byte[8192];        // steady young allocation
            garbage[0] = (byte) work;
            work += garbage.length;
        }
        System.out.println("work counter = " + work);
        System.out.println("live set still reachable: " + live.size() + " chains");
    }
}
```

Run it under both collectors, with **identical heap settings**, logging to separate
files:

```bash
javac -d out LiveSetPressure.java

# G1 — the control.
java -Xms4g -Xmx4g -XX:+UseG1GC \
  -Xlog:gc*,gc+heap=debug:file=logs/g1.log:time,uptime,level,tags \
  -cp out com.orderflow.lab.gc.LiveSetPressure 2000 120

# ZGC, generational. On JDK 21 the second flag is REQUIRED to get generational mode.
# On JDK 23+ generational is the default and the flag should be dropped.
java -Xms4g -Xmx4g -XX:+UseZGC -XX:+ZGenerational \
  -Xlog:gc*,gc+heap=debug:file=logs/zgc.log:time,uptime,level,tags \
  -cp out com.orderflow.lab.gc.LiveSetPressure 2000 120

# Shenandoah, IF your JDK build has it.
java -Xms4g -Xmx4g -XX:+UseShenandoahGC \
  -Xlog:gc*:file=logs/shen.log:time,uptime,level,tags \
  -cp out com.orderflow.lab.gc.LiveSetPressure 2000 120
```

**WHAT TO LOOK FOR.** Not the work counter first. These, in order:

1. **The longest pause in each log.** Not the mean. Sort them.
2. **Whether the longest pause changes when you re-run with `2000` replaced by `4000`
   MB of live set** (and the heap raised to match). This is the decisive observation:
   G1's pauses should grow with the live set; ZGC's should not.
3. Only then, the work counter — and only as a *same-program, two-configuration*
   comparison, never as a throughput figure to quote.

| What you see | What it means |
|---|---|
| G1's longest pause grows substantially when you double the live set; ZGC's does not | **You have observed the entire point of this topic.** Pause independence from live-set size is ZGC's product, and you just watched it. |
| Both collectors' longest pauses grow with the live set | Suspect your ZGC run is non-generational, or the pause you are reading includes time-to-safepoint from something else in the program. Check `-XX:+ZGenerational` was applied (`jcmd VM.flags -all`) and read Topic 73 before concluding anything. |
| The work counter is lower under ZGC | Expected, and it is the **cost side of the trade** made visible: the load barrier on every one of those pointer-chasing loads. Do not report a percentage — this rig is not a benchmark. Report the direction. |
| The work counter is *higher* under ZGC | Possible and worth understanding rather than dismissing. If G1 was spending a large share of wall-clock time in pauses, removing them can more than pay for the barrier. This is exactly the workload shape where ZGC wins outright. |
| ZGC log contains `Allocation Stall` lines | **The important failure signal.** Your allocation rate outran the collector at this heap size. Raise `-Xmx`, give it more CPU, or allocate less. This is the ZGC-specific failure mode; see Trap 2. |
| ZGC's pauses are not sub-millisecond at all | Check for TTSP (Topic 73), check thread count (root scanning), and check whether you are on the non-generational mode. Also check whether the machine is CPU-starved — a concurrent collector with no spare CPU behaves badly. |
| `-XX:+UseShenandoahGC` fails to start | Your JDK build does not include Shenandoah. This is a distribution fact, not a configuration error. |

Now vary **one** thing at a time: heap size, live-set size, `-XX:ActiveProcessorCount`.
Watch which collector's behaviour is sensitive to which variable. That sensitivity map
*is* the collector-choice decision, and you cannot get it from a document.

---

## Example 2 — production scenario (on the project spine)

### The constraints, taken from the Topic 65 baseline

| Thing | Value |
|---|---|
| Service | `orderflow`, containerised |
| Dataset | 100k products, 1M orders, 5M order lines |
| Container | 2 vCPU, 2 GiB memory limit |
| Heap | `-Xmx1200m -Xms1200m` |
| Collector at baseline | G1 (verified with `jcmd VM.flags -all`, not assumed) |
| Load mix | 70% catalogue read, 20% order read, 10% order placement |
| Downstream | payment gateway call inside `POST /orders`, a real network hop |
| Baseline artefacts | `/docs/java/baselines/` — p50/p95/p99/p999, throughput, error rate |

### The proposal that lands in your review queue

A colleague opens a pull request against the deployment manifest:

```diff
   env:
     - name: JAVA_OPTS
-      value: "-Xms1200m -Xmx1200m -XX:+UseG1GC"
+      value: "-Xms1200m -Xmx1200m -XX:+UseZGC"
```

The description says: *"ZGC gives sub-millisecond pauses. Our p99 on `POST /orders` is
too high and GC is the obvious suspect. This should fix it."*

Every word of that is plausible. Most of it is wrong. Here is how you review it.

### Step 1 — ask what fraction of p99 is GC, before anything else

This is one command against artefacts that already exist.

```bash
# From the baseline GC log, the pause distribution:
grep -E 'Pause Young|Pause Full|Pause Remark' /var/log/orderflow/gc-baseline.log \
  | sed -E 's/.*\) ([0-9.]+)ms$/\1/' | sort -n | tail -20

# From the recorded k6 baseline, the end-to-end percentiles for POST /orders.
cat docs/java/baselines/baseline-summary.json
```

**WHAT TO LOOK FOR:** the longest GC pauses, expressed as a fraction of the recorded
p99 for the endpoint in question.

Suppose — and you must get the real answer from your own artefacts — that `POST
/orders` p99 is dominated by the payment-gateway round trip, and the longest GC pause
in the entire run is a small fraction of it. Then:

> **The best possible outcome of this change is that p99 improves by that fraction.
> The likely outcome is that p99 gets worse, because every request now pays the load
> barrier and the container has only 2 vCPU to give the concurrent collector.**

You have reviewed the PR without deploying anything. Write that in the comment.

### Step 2 — name the *specific* costs this deployment will pay

Not generic costs. These four, tied to this manifest:

**Cost 1 — no compressed oops on a 1200 MB heap.**
Topic 69: below ~32 GB, HotSpot stores references as 32-bit shifted values. ZGC turns
that off. Every reference field in every `Order`, `OrderLine`, `Product` and Hibernate
proxy on the heap gets bigger. On a heap this small, and with an entity graph this
reference-dense, that is a **direct reduction in how much of your dataset fits**. Your
live set grows without a single line of code changing.

**Cost 2 — concurrent GC threads compete for 2 vCPU.**
ZGC does its work concurrently, which means *on CPUs your request threads want*. At 2
vCPU with a container quota, "concurrent" does not mean "free" — it means "sharing".
Topic 82's arithmetic applies directly: `Runtime.availableProcessors()` drives GC
thread counts, and a small quota gives the collector very little room.

**Cost 3 — headroom.**
G1 collects when eden fills; a pause reclaims memory synchronously. ZGC must *finish*
a concurrent cycle before you run out. At `-Xmx1200m` with an allocation rate measured
at the Topic 65 baseline, there may not be enough headroom, and the symptom is
**allocation stalls** — worse than the pauses you were trying to remove, and harder to
recognise because they do not appear as pauses.

**Cost 4 — on JDK 21, this flag combination is non-generational.**
The manifest says `-XX:+UseZGC` with no `-XX:+ZGenerational`. On JDK 21 that is the
non-generational collector: every cycle traces the whole heap. For a request-serving
workload where almost everything dies within a request, this is close to a worst case.
**The PR as written would deploy the wrong ZGC.** Confirm what the running container
actually got:

```bash
jcmd $(pgrep -f orderflow) VM.flags -all | grep -iE "UseZGC|ZGenerational"
```

### Step 3 — the review comment you actually write

> I am not blocking ZGC as an idea, I am blocking this change as written. Three
> things before I can approve:
>
> 1. **What share of `POST /orders` p99 is GC pause?** From the baseline log the
>    longest pause is *X* and p99 is *Y*. If GC is a small share of *Y*, the ceiling
>    on this change is small and the floor is a throughput regression on every request.
>    Put the arithmetic in the PR description.
> 2. **On JDK 21 this deploys non-generational ZGC.** Add `-XX:+ZGenerational`, or
>    say explicitly why not. Then confirm with `jcmd VM.flags -all` in the canary,
>    because I do not want to review a flag, I want to review what the JVM chose.
> 3. **Run the Topic 65 profile under both, three times each**, and post p50/p95/p99/
>    p999 **and throughput** for both. If p99 improves and throughput drops, that is a
>    trade we can discuss with numbers. Right now it is a preference.
>
> Also: check `-Xlog:safepoint*` first. If our latency spikes are time-to-safepoint
> rather than GC work, neither collector fixes them and we would ship a throughput
> regression for nothing.

### Step 4 — the case where the answer is YES

Be fair to the proposal. There are real `orderflow` shapes where ZGC wins:

- **The nightly reconciliation job.** It loads a very large live set into memory to
  reconcile payments against orders. Under G1 its pauses grow with that live set,
  because Object Copy scales with survivors (Topic 71). Under ZGC they do not. If that
  job shares a JVM with the API — which is itself a design smell, but common — ZGC
  protects the API's tail latency from the job's live set.
- **A much larger heap.** If `orderflow` grows to hold a large in-process cache and
  runs with `-Xmx32g`, G1's pauses on that live set become genuinely hard to control,
  and ZGC's pause independence is exactly the product you want. **ZGC's headline
  feature is large heaps, and a 1200 MB container is the opposite of its home turf.**
- **A hard latency SLO with a small GC-share budget.** If the payment gateway is
  removed from the synchronous path (Topic 115's outbox, Topic 111's async patterns)
  and `POST /orders` p99 becomes database-bound and short, then a 40 ms G1 pause is
  suddenly a large share of the budget and the arithmetic flips.

**Notice that in two of those three cases, the thing that changed was not the
collector — it was the workload.** That is the senior insight: collector choice is
downstream of latency budget and live-set size, and both of those are properties of
your system, not of the JVM.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — adopting ZGC when p99 is dominated by a downstream call

**Wrong:**

```bash
# "Our p99 is bad. ZGC has sub-millisecond pauses. Ship it."
java -Xms1200m -Xmx1200m -XX:+UseZGC -jar orderflow.jar
```

The reasoning is entirely reasonable and entirely wrong: p99 is high, GC causes
latency, this collector has less of it.

**Exact symptom:**

- `POST /orders` p99 is essentially unchanged from the recorded baseline — well inside
  run-to-run noise.
- **Throughput at the same arrival rate drops**, and at the top of the load profile
  the error rate rises because requests start timing out.
- CPU utilisation in the container is higher for the same request rate.
- The GC log shows beautiful, tiny pauses. Everyone points at it as evidence the
  change worked, while the k6 output says it did not.
- p50 gets *slightly worse* across every endpoint, including read-only catalogue
  endpoints that allocate almost nothing. **That last detail is the giveaway**: a
  uniform, small regression on every endpoint is the signature of a per-operation tax,
  not of a GC event.

**Root cause:** two independent facts, both of which you could have established before
deploying.

1. **GC pause was never a meaningful share of p99.** If the payment gateway
   contributes hundreds of milliseconds to the tail, removing tens of milliseconds of
   pause from a small fraction of requests moves p99 by a small amount at most. The
   collector was optimising a term that was not dominant. This is Amdahl's law, and
   you already knew it from system design — you just did not apply it to the JVM.
2. **The load barrier is a tax on every reference load in the process.** Not on
   allocation. Not on GC. On *reads*. Hibernate entity graph traversal, `HashMap`
   lookups, iterating an `ArrayList<OrderLine>`, every field access on every DTO. That
   is why the regression is uniform and why it shows up in p50.

**Fix:**

1. **Revert.** Re-establish the baseline and confirm you are within ±10%.
2. **Compute the GC share of p99 first**, from the existing log and the existing
   baseline. Put the number in the ticket.
3. **Attack the actual dominant term.** If it is the gateway call, that is Topic 111
   (timeouts, retries, circuit breaking) and Topic 115 (get it off the synchronous
   path), not a GC topic at all.
4. **If you still want the collector change, justify it against a specific budget**:
   "GC contributes *N* of our *M* ms p99 budget; the SLO requires *M'*; the only lever
   that reaches *M'* is pause elimination." That sentence is the only acceptable
   justification, and it requires two measurements you can actually produce.

**How to confirm the diagnosis in one command:**

```bash
# Is the regression uniform across endpoints, including ones that barely allocate?
# A uniform p50 shift on a read-only endpoint is a per-operation tax, not a GC event.
jq '.metrics | to_entries[] | select(.key|startswith("http_req_duration"))' k6-zgc.json
```

---

### Trap 2 — not giving ZGC enough heap headroom

**Wrong:**

```bash
# Same heap that worked fine under G1. "It's the same amount of memory."
java -Xms1200m -Xmx1200m -XX:+UseZGC -XX:+ZGenerational -jar orderflow.jar
```

**Exact symptom:** the one you must learn to recognise, because it looks like nothing
you have seen before.

- Latency spikes appear, but **no GC pause in the log accounts for them.** Every pause
  is sub-millisecond. The GC log looks perfect.
- The spikes correlate with allocation-heavy endpoints — order placement, the invoice
  path — rather than with any GC event.
- The ZGC log contains lines about **allocation stalls**. Grep for them; they are the
  entire diagnosis.
- Under sustained load the service degrades progressively rather than spiking: as
  headroom shrinks, more threads stall, throughput falls, queues build, and eventually
  requests time out.
- If it gets bad enough, ZGC does a stop-the-world collection as a last resort and you
  see a pause that makes a mockery of "sub-millisecond".

**Root cause:** a concurrent collector is in a **race with your allocation rate**.
G1 stops the world to collect, so it cannot lose that race in the same way — it just
pauses longer. ZGC collects while you allocate. If the collector cannot free memory as
fast as you consume it, application threads have nowhere to allocate into and **block**.

Three things made the race harder than it was under G1, and all three are consequences
of choices in this exact command line:

1. **Compressed oops are off**, so the same object graph occupies more of that
   1200 MB. Your effective heap shrank without your changing `-Xmx`.
2. **Concurrent GC threads need CPU**, and there are 2 vCPU total. Starve the
   collector of CPU and it loses the race sooner.
3. **The heap is small in absolute terms.** ZGC's design point is large heaps; the
   headroom that a concurrent collector needs is a *fraction* of the heap, and a
   fraction of a small number is a small number.

**Fix:**

1. **Grep for the stall lines first** — do not theorise:
   ```bash
   grep -i 'allocation stall' /var/log/orderflow/gc.log | wc -l
   ```
   Non-zero is the diagnosis. Zero means look elsewhere (start with Topic 73).
2. **Give it headroom.** Raise `-Xmx`, which for a container means raising the memory
   limit too, which means Topic 82's arithmetic (heap is not the footprint) and a
   conversation about cost.
3. **Use `-XX:SoftMaxHeapSize`** to express "normally stay here, burst if you must",
   rather than choosing between a small hard limit and a wasteful large one.
4. **Give it CPU.** Concurrent collection is not free in a CPU-quota'd container.
5. **Reduce allocation rate.** Topic 78's allocation profile, Topic 01's boxing, Topic
   75's escape analysis. **This is the fix that also helps under every other
   collector**, which is a strong signal that it is the right one.
6. **Or conclude that this container is not ZGC's habitat** and go back to G1. That is
   a legitimate, senior conclusion — not a failure.

---

### Trap 3 — benchmarking ZGC on JDK 21 without `-XX:+ZGenerational`

**Wrong:**

```bash
# "I A/B tested G1 against ZGC on our JDK 21 build. ZGC lost. ZGC is bad for us."
java -Xms4g -Xmx4g -XX:+UseZGC -jar orderflow.jar
```

**Exact symptom:**

- ZGC's throughput is dramatically worse than G1's, not marginally worse.
- The ZGC log shows GC cycles that are frequent and that appear to touch the whole
  heap every time — the cycle count and duration do not behave like a young collector
  at all.
- Heap occupancy behaviour looks nothing like G1's sawtooth of cheap young collections.
- The result is so bad that everyone concludes the collector is unusable, and the
  evaluation is closed.

**Root cause:** on JDK 21, `-XX:+UseZGC` alone selects the **non-generational** mode.
Every cycle traces the entire live set, because there is no young generation to collect
cheaply. For `orderflow`, where the overwhelming majority of objects die inside one
request, this is close to the worst possible match between collector and workload.

You did not measure "ZGC". You measured a mode that no longer exists in current JDKs.

**Fix:**

1. **Add the flag on JDK 21:** `-XX:+UseZGC -XX:+ZGenerational`.
2. **Verify what you actually got, every time:**
   ```bash
   jcmd $(pgrep -f orderflow) VM.flags -all | grep -iE "UseZGC|ZGenerational"
   ```
3. **On JDK 23+, drop the flag** — generational is the default there, and on JDK 24+
   the non-generational mode is gone and the flag is obsolete.
4. **Re-run the comparison** and report both results, labelled by mode. "Non-
   generational ZGC lost badly; generational ZGC was competitive" is a much more
   useful finding than "ZGC lost", and it is the honest one.
5. **Generalise the lesson:** any collector comparison must state the JDK version and
   the full flag set, because collector defaults move between versions. A benchmark
   without a `VM.flags -all` dump attached is an anecdote.

---

### Trap 4 — reading a tiny GC pause as proof that the JVM did not stall

**Wrong:** the ZGC log shows sub-millisecond pauses, so GC is exonerated and everyone
moves on to blame the database.

**Exact symptom:** p999 shows stalls of hundreds of milliseconds. No pause in the GC
log is anywhere near that long. The stalls hit **requests already in flight on threads
doing no GC work**, across unrelated endpoints, at the same instant. The database's own
metrics show nothing.

**Root cause:** ZGC's pause is short **at the safepoint**. It does not include the time
to *reach* the safepoint. If one thread is in a long counted `int` loop over a large
array, every other thread waits for it, and the GC log reports only the tiny part it
measures.

**This trap is exactly as real under ZGC as under G1.** Switching to a low-pause
collector does **not** fix time-to-safepoint, and people are frequently more confused
by it under ZGC precisely because the GC log looks so clean.

**Fix:**

```bash
-Xlog:safepoint*:file=/var/log/orderflow/safepoint.log:time,uptime,level,tags
```

**WHAT TO LOOK FOR:** the field reporting time spent *reaching* the safepoint versus
time spent *at* it. When the reaching time dominates, no collector change will help you.
**Topic 73 is the entire treatment, and it is the next document you should read.**

---

### Trap 5 — assuming Shenandoah is available, and assuming what "generational" means

**Wrong:** a design document specifies Shenandoah, with generational mode, because a
comparison table said it had the best characteristics.

**Exact symptom, one of three:**

- The JVM refuses to start: `Unrecognized VM option 'UseShenandoahGC'`. Your vendor's
  build does not include it.
- The JVM starts but `-XX:ShenandoahGCMode=generational` is rejected, or accepted with
  an experimental-feature warning, or requires `-XX:+UnlockExperimentalVMOptions`.
- Everything starts, and months later a JDK upgrade changes the default mode or the
  experimental status, and behaviour shifts with no code change.

**Root cause:** collector availability is a property of the **JDK distribution**, not
of the Java version, and the maturity status of a mode is a property of the **JDK
release**. Neither is something you can read off a comparison table.

**Fix:**

1. **Verify availability in the exact base image you deploy**, in CI, as a startup
   assertion:
   ```bash
   java -XX:+UseShenandoahGC -version || echo "Shenandoah not present in this build"
   java -XX:+PrintFlagsFinal -version | grep -iE "UseShenandoahGC|ShenandoahGCMode"
   ```
2. **Log the collector from inside the application at startup**, so the fact is in your
   own logs and not only in a deployment manifest:
   ```java
   import java.lang.management.ManagementFactory;

   @Component
   class CollectorFacts implements ApplicationRunner {
       private static final Logger log = LoggerFactory.getLogger(CollectorFacts.class);

       @Override public void run(ApplicationArguments args) {
           ManagementFactory.getGarbageCollectorMXBeans()
               .forEach(b -> log.info("gc collector bean = {}", b.getName()));
           log.info("availableProcessors = {}", Runtime.getRuntime().availableProcessors());
           log.info("maxMemory MB = {}", Runtime.getRuntime().maxMemory() / (1024 * 1024));
       }
   }
   ```
3. **Never plan a production architecture around an experimental mode** without an
   explicit, written decision that says so and names who owns the risk.
4. **Pin the base image**, because "we upgraded the JDK" is a collector change.

---

## Hands-on proof

Every command here is one **you** run. I have no JVM and will not print output and call
it captured. What follows is the exact command, what to look for, and how to read every
plausible result — including the ones that contradict the expected answer.

### Setup

```bash
mkdir -p ~/java-lab/72 && cd ~/java-lab/72
java --version                 # record it: the version determines ZGC's default mode
mkdir -p logs
```

### Proof 1 — establish which collectors exist in YOUR JDK

```bash
for GC in UseSerialGC UseParallelGC UseG1GC UseZGC UseShenandoahGC; do
  printf "%-18s " "$GC"
  if java -XX:+$GC -version >/dev/null 2>&1; then echo "available"; else echo "NOT AVAILABLE"; fi
done
```

**WHAT TO LOOK FOR:** which of the five your binary actually contains.

| What you see | What it means |
|---|---|
| All five available | An OpenJDK build that includes Shenandoah (Temurin, Red Hat and others typically do). |
| All but Shenandoah | Very common. Oracle's JDK builds have historically not shipped Shenandoah. This is a distribution decision, not a bug, and no flag changes it. |
| ZGC unavailable | Either a very old JDK, a 32-bit JVM (ZGC requires 64-bit), or an unusual platform. Check `java -version` and the architecture. |
| Everything "available" but the JVM warns | Read the warning. Experimental options may need `-XX:+UnlockExperimentalVMOptions`, and that is itself information about maturity. |

### Proof 2 — establish ZGC's mode on your runtime

```bash
java -XX:+UseZGC -XX:+PrintFlagsFinal -version 2>/dev/null | grep -i ZGenerational
java -XX:+UseZGC -XX:+ZGenerational -version 2>&1 | head -5
```

**WHAT TO LOOK FOR:** whether `ZGenerational` exists as a flag, what its default value
is, and whether passing it produces a warning.

| What you see | What it means |
|---|---|
| `ZGenerational` present and defaulting to `false` | You are on a JDK where generational ZGC is **opt-in**. JDK 21 behaves this way. **You must pass the flag** for any meaningful evaluation. |
| `ZGenerational` present and defaulting to `true` | Generational is the default. JDK 23-era behaviour. Do not pass the flag. |
| The flag does not exist, or passing it warns that it is obsolete | The non-generational mode has been removed. JDK 24+ behaviour. `-XX:+UseZGC` is all you need. |
| No output from the grep at all | Either the flag is not present, or your `PrintFlagsFinal` invocation failed. Test the mechanism first: `java -XX:+PrintFlagsFinal -version \| head`. |

**Write down what your runtime says.** Every ZGC measurement you take from here is
meaningless without it, and every ZGC claim you make in an interview should be
prefaced by the version.

### Proof 3 — read a ZGC log line, field by field

You must be able to parse these unaided. Here is the **shape** of a ZGC cycle
summary line:

```
[2026-08-29T14:02:11.417+0000][612.041s][info][gc] GC(318) Major Collection (Allocation Rate) 1420M(35%)->612M(15%) 0.412s
```

***Illustration of the format, not captured output. The numbers are placeholders chosen
to show the field positions.***

And the shape of a pause line inside that cycle:

```
[2026-08-29T14:02:11.418+0000][612.042s][info][gc,phases] GC(318) Y: Pause Mark Start 0.021ms
```

***Illustration of the format, not captured output.***

Field by field:

| Field | Meaning |
|---|---|
| `[2026-08-29T...]` | Wall clock. Present because you passed `time` in the `-Xlog` decorators. Correlate with k6 and your APM. |
| `[612.041s]` | Uptime. Present because you passed `uptime`. Use this for intervals. |
| `[info]` | Level. `debug` lines appear only for tags you explicitly raised. |
| `[gc]` / `[gc,phases]` | Tag set. **Grep on the tag, not the message text.** |
| `GC(318)` | Cycle id. All lines of one cycle share it — this is how you reassemble a cycle. |
| `Y:` / `O:` | **Generational ZGC only:** young or old collection. Its presence is itself proof you are running generational mode. |
| `Major Collection` / `Minor Collection` | Which generation this cycle addressed. |
| `(Allocation Rate)` | **The cause.** The diagnostic field: why this cycle started. |
| `1420M(35%)->612M(15%)` | Heap used before and after, with percentage of capacity. |
| `0.412s` | **Cycle** duration — the wall time of the concurrent cycle. **This is NOT a pause.** Confusing it with a pause is the most common ZGC log-reading error. |
| `Pause Mark Start 0.021ms` | An actual stop-the-world pause. These are the numbers that should be tiny. |

**The single most important reading rule for ZGC logs:** *cycle duration is not pause
duration.* A ZGC cycle can take hundreds of milliseconds of wall time while stopping
your application for a fraction of a millisecond, three times. Someone who reads the
cycle duration as a pause will report catastrophic pauses that never happened.

The lines you must be able to recognise:

| Line / cause | What it means | Is it a problem? |
|---|---|---|
| `Pause Mark Start` / `Mark End` / `Relocate Start` | ZGC's three stop-the-world points | No — these should be tiny. If they are not, suspect root-set size or TTSP (Topic 73). |
| Cause `(Allocation Rate)` | Heuristics started a cycle because of how fast you are allocating | Normal. This is the common trigger. |
| Cause `(Warmup)` | Early cycles while heuristics calibrate | Normal, at startup only. |
| Cause `(Proactive)` | ZGC collecting while relatively idle to keep the heap tidy | Usually fine; a flag controls it. |
| Cause `(Metadata GC Threshold)` | Metaspace, not heap | Different problem: class loading. Topics 67 and 80. |
| Cause `(System.gc())` | Something called `System.gc()` | **Find the caller.** A library, a JMX console, an RMI distributed-GC timer. |
| **`Allocation Stall`** | **An application thread blocked waiting for memory** | **Yes. Always. This is ZGC's characteristic failure and Trap 2's smoking gun.** |
| A stop-the-world collection / OOM after stalls | The concurrent collector lost the race decisively | **Yes.** Headroom, CPU, or allocation rate. |
| `Y:` and `O:` prefixes absent on a JDK where you expected generational | You are running non-generational mode | Trap 3. Check the flags. |

And for Shenandoah:

| Line | What it means | Is it a problem? |
|---|---|---|
| `Pause Init Mark` / `Final Mark` / `Init Update Refs` / `Final Update Refs` | Shenandoah's short stop-the-world points | No — should be short. |
| `Concurrent marking` / `evacuation` / `update references` | The concurrent phases | Normal. |
| **`Pause Degenerated GC`** | The concurrent cycle could not keep up and fell back to stop-the-world | **Yes** — the direct analogue of G1's evacuation failure. |
| **`Pause Full`** | Last-resort full collection | **Yes. Always an incident.** |
| `Cancelling GC: Allocation Failure` | Allocation outran the collector | **Yes** — headroom or allocation rate. |

### Proof 4 — confirm compressed oops are off under ZGC

This is a fact with real footprint consequences, and it takes one command to verify.

```bash
java -Xmx8g -XX:+UseG1GC  -XX:+PrintFlagsFinal -version | grep -i UseCompressedOops
java -Xmx8g -XX:+UseZGC   -XX:+PrintFlagsFinal -version | grep -i UseCompressedOops
```

**WHAT TO LOOK FOR:** the value differing between the two runs at the same heap size.

| What you see | What it means |
|---|---|
| `true` under G1, `false` under ZGC | Confirmed. Coloured pointers need the full 64 bits. Every reference field in your heap is 8 bytes under ZGC. Topic 69's arithmetic now applies with the larger number. |
| `false` under both | Your heap is above the compressed-oops ceiling (roughly 32 GB) even under G1, so there is nothing to lose. |
| `true` under both | Surprising. Re-check that `-XX:+UseZGC` was actually applied — `PrintFlagsFinal` prints the final values, so also grep for `UseZGC` in the same output. |

Then quantify it for *your* object graph with JOL (Topic 69):

```bash
java -XX:+UseZGC -jar jol-cli.jar internals com.orderflow.orders.OrderLine
java -XX:+UseG1GC -jar jol-cli.jar internals com.orderflow.orders.OrderLine
```

**WHAT TO LOOK FOR:** the size difference per instance, multiplied by the number of
instances in your live set. That product is the footprint cost of the collector change,
and it is a number you can put in a capacity model.

### Proof 5 — watch pause independence from heap size directly

This is the demonstration that justifies the whole collector.

```bash
for H in 2g 4g 8g 16g; do
  echo "=== heap $H ==="
  java -Xms$H -Xmx$H -XX:+UseG1GC \
    -Xlog:gc:file=logs/g1-$H.log:uptime,level,tags \
    -cp out com.orderflow.lab.gc.LiveSetPressure $(( ${H%g} * 400 )) 90
  java -Xms$H -Xmx$H -XX:+UseZGC -XX:+ZGenerational \
    -Xlog:gc,gc+phases:file=logs/zgc-$H.log:uptime,level,tags \
    -cp out com.orderflow.lab.gc.LiveSetPressure $(( ${H%g} * 400 )) 90
done

# Longest pause per configuration. Extract, sort, take the tail. Do it on YOUR logs.
for f in logs/g1-*.log;  do echo "$f"; grep -oE '[0-9]+\.[0-9]+ms' "$f" | sort -n | tail -1; done
for f in logs/zgc-*.log; do echo "$f"; grep 'Pause' "$f" | grep -oE '[0-9]+\.[0-9]+ms' | sort -n | tail -1; done
```

**WHAT TO LOOK FOR:** the *trend* of the longest pause as the heap and live set grow.

| What you see | What it means |
|---|---|
| G1's longest pause grows with the live set; ZGC's stays roughly flat | **The core claim of this topic, observed rather than believed.** This is the graph to put in a design document. |
| Both grow | Something else is in your pause. Suspect TTSP (Topic 73), an enormous thread count inflating root scanning, or a machine that is CPU-starved. Investigate before drawing conclusions. |
| ZGC's pauses grow only at the largest heap | Look at what scales: root set, weak references, or the machine running out of CPU for concurrent work. Not heap size itself. |
| ZGC's logs contain allocation stalls at the smaller heaps | Headroom. This is the *other* side of the trade and it belongs in your report alongside the pause graph. |

### Proof 6 — measure the barrier cost honestly, in isolation

The load barrier taxes reference **loads**. A benchmark that only allocates will not
show it. A benchmark that chases pointers will.

```java
package com.orderflow.bench;

import org.openjdk.jmh.annotations.*;
import org.openjdk.jmh.infra.Blackhole;
import java.util.concurrent.TimeUnit;

/**
 * Pointer chasing: the operation a LOAD barrier taxes.
 *
 * Run this whole benchmark under G1 and under ZGC as SEPARATE JVM invocations
 * (-jvmArgs), never as two @Params in one JVM. The collector is a JVM-wide property.
 */
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@Warmup(iterations = 5, time = 1)
@Measurement(iterations = 10, time = 1)
@Fork(3)
@State(Scope.Benchmark)
public class ReferenceLoadBenchmark {

    static final class Node { Node next; long value; }

    @Param({"1000", "100000"})     // fits in cache vs. does not
    public int chainLength;

    private Node head;

    @Setup(Level.Trial)
    public void setUp() {
        Node current = null;
        for (int i = 0; i < chainLength; i++) {
            Node n = new Node();
            n.value = i;
            n.next = current;
            current = n;
        }
        head = current;
    }

    @Benchmark
    public long walkChain() {
        long sum = 0;
        for (Node n = head; n != null; n = n.next) {   // one reference LOAD per hop
            sum += n.value;
        }
        return sum;                                     // returned, so not eliminated
    }
}
```

```bash
java -jar target/benchmarks.jar ReferenceLoadBenchmark \
  -jvmArgs "-Xms4g -Xmx4g -XX:+UseG1GC" -rf json -rff bench-g1.json

java -jar target/benchmarks.jar ReferenceLoadBenchmark \
  -jvmArgs "-Xms4g -Xmx4g -XX:+UseZGC -XX:+ZGenerational" -rf json -rff bench-zgc.json
```

**WHAT TO LOOK FOR:** the **confidence intervals**, not the point estimates. If they
overlap, you measured nothing.

| What you see | What it means |
|---|---|
| ZGC's time per operation is higher, intervals do not overlap | The load barrier's cost, isolated and measured on your hardware. **This is the number to quote in the design document — yours, not a vendor's.** |
| Intervals overlap | You have not shown a difference. Either the barrier's cost is below your noise floor for this shape, or your run is too short. More forks, more iterations. |
| The difference is enormous | Check that both runs used the same heap size, and check whether GC ran during measurement — add `-prof gc` and look at the cycle counts. A benchmark that triggers collections is measuring collections. |
| ZGC is *faster* | Possible if the G1 run was doing collections during measurement. Look at `-prof gc` for both. Then re-run with a larger heap so neither collects during the measurement window. |

**Then state the honest caveat in your report:** this microbenchmark measures the
barrier on a pathological pointer-chasing loop. Your service's real barrier cost
depends on its ratio of reference loads to other work, and the only way to know it is
the end-to-end Topic 65 comparison in the drill below. **The microbenchmark bounds the
cost; the load test measures it.**

---

## Failure drill

**Mandatory.** Do not read the "how to read it" tables until you have produced the
result yourself and written down what you saw.

### The assignment, restated from the master plan

> Run the Topic 65 baseline under G1, then `-XX:+UseZGC`. Compare p99, p999 and
> throughput. **Explain any case where ZGC is worse.**

That last sentence is the assignment. The comparison is easy. **Explaining a
regression is the skill**, and if ZGC comes out worse on your service, you have not
failed the drill — you have completed it.

### Step 0 — establish the control

```bash
docker compose up -d
# Confirm the dataset is the Topic 65 dataset. Same rows, same skew, same cache state.

java -Xms1200m -Xmx1200m -XX:+UseG1GC \
  -Xlog:gc*,gc+heap=debug:file=/var/log/orderflow/gc-g1.log:time,uptime,level,tags:filecount=5,filesize=50m \
  -Xlog:safepoint*:file=/var/log/orderflow/safepoint-g1.log:time,uptime,level,tags \
  -jar orderflow.jar

k6 run --out json=logs/g1.json load/baseline.js
```

Record, in your own notes:

1. p50 / p95 / p99 / p999 **per endpoint**, not aggregate.
2. Throughput (requests/second actually completed) and error rate.
3. Longest GC pause, and the pause distribution.
4. Full GC count.
5. Allocation rate, derived per Topic 71's formula.
6. Live set — heap occupancy after a collection late in the run.
7. Container CPU utilisation at the load plateau.
8. **The safepoint log's reaching-time versus at-time split.** You will need this to
   rule out Topic 73 before you attribute anything to the collector.

**If these are not within ±10% of `/docs/java/baselines/`, stop.** The Topic 65 gate
rule applies. A collector comparison against an unstable baseline is noise.

### Step 1 — compute the ceiling before you run anything

From the control's GC pause distribution and end-to-end percentiles:

> **If every GC pause became exactly zero, what would p99 be?**

Write that number down. **It is the absolute best ZGC can do.** If it is within your
run-to-run noise, you have predicted the outcome of the drill before running it — and
predicting a result correctly, from a model, is worth more than the measurement.

Do this per endpoint. `GET /products` and `POST /orders` will have very different
answers, because one is short and cheap and the other includes a network hop.

### Step 2 — run under ZGC, correctly

```bash
# On JDK 21: the ZGenerational flag is REQUIRED. On JDK 23+, drop it.
java -Xms1200m -Xmx1200m -XX:+UseZGC -XX:+ZGenerational \
  -Xlog:gc*,gc+heap=debug:file=/var/log/orderflow/gc-zgc.log:time,uptime,level,tags:filecount=5,filesize=50m \
  -Xlog:safepoint*:file=/var/log/orderflow/safepoint-zgc.log:time,uptime,level,tags \
  -jar orderflow.jar

# FIRST, before load: confirm what the JVM actually chose.
jcmd $(pgrep -f orderflow) VM.flags -all | grep -iE "UseZGC|ZGenerational|UseCompressedOops|SoftMaxHeapSize"

k6 run --out json=logs/zgc.json load/baseline.js
```

**Do not change anything else.** Same heap, same container limits, same dataset, same
load profile, same duration. One variable.

Record the same eight items. Then run each configuration **at least three times** so
you know your noise floor. A difference smaller than your noise is not a difference.

### Step 3 — the third configuration, if available

```bash
java -Xms1200m -Xmx1200m -XX:+UseShenandoahGC \
  -Xlog:gc*:file=/var/log/orderflow/gc-shen.log:time,uptime,level,tags \
  -jar orderflow.jar
```

If Shenandoah is not in your JDK build, **record that as a finding**. "We cannot deploy
Shenandoah because our base image does not contain it" is a real constraint that
belongs in the report.

### Step 4 — what to capture, before interpreting anything

```bash
# Pause distributions.
grep -oE '[0-9]+\.[0-9]+ms' /var/log/orderflow/gc-g1.log  | sort -n | tail -20
grep 'Pause' /var/log/orderflow/gc-zgc.log | grep -oE '[0-9]+\.[0-9]+ms' | sort -n | tail -20

# ZGC's characteristic failure.
grep -ci 'allocation stall' /var/log/orderflow/gc-zgc.log

# G1's characteristic failure.
grep -cE 'Pause Full|To-space exhausted|Evacuation Failure' /var/log/orderflow/gc-g1.log

# Shenandoah's characteristic failure.
grep -cE 'Pause Degenerated|Pause Full|Cancelling GC' /var/log/orderflow/gc-shen.log

# Rule out TTSP in BOTH before attributing anything to the collector.
grep -i 'safepoint' /var/log/orderflow/safepoint-g1.log  | tail -20
grep -i 'safepoint' /var/log/orderflow/safepoint-zgc.log | tail -20
```

### Step 5 — how to read it

| What you see | What it means |
|---|---|
| ZGC's longest pause is dramatically shorter, **and** p99 barely moves | **The expected result for `orderflow`, and the whole lesson.** GC pause was not a meaningful share of p99. You have just demonstrated Amdahl's law on the JVM. Write that sentence down. |
| p99 unchanged **and** throughput lower under ZGC | **ZGC is worse here, and you can explain why:** the load barrier taxes every reference load, the container has 2 vCPU for concurrent GC work to share, and compressed oops are off so the live set grew. Three named, mechanical causes. This is the drill's target answer. |
| p999 improves meaningfully while p99 does not | Interesting and legitimate. p999 is where the rarest, longest G1 pauses live. If your SLO is written on p999 — and for a payments path it might be — this is a real win. **Report it as a p999 win with a throughput cost, not as "ZGC is better".** |
| ZGC log contains allocation stalls | Trap 2. You did not give it headroom. **Do not report the run as a ZGC result** — report it as a misconfiguration, fix the headroom, and re-run. A stalled run measures your heap sizing, not the collector. |
| Throughput drops so far that error rate rises | The barrier plus the CPU competition exceeded what a 2 vCPU container can absorb. This is a capacity finding: ZGC's concurrent work is not free when there is no spare CPU. Topic 82. |
| ZGC pauses are *not* short | Read the safepoint log before anything else. If reaching time dominates, this is Topic 73 and no collector fixes it. Also check thread count: root scanning scales with threads. |
| Both collectors show the same p99 spikes at the same timestamps | Something outside the JVM. Database, network, container CPU throttling, a noisy neighbour. Correlate with `docker stats` and the Postgres logs. **Two collectors agreeing is strong evidence the JVM is not the cause.** |
| ZGC wins on p99 *and* throughput | Genuinely possible if G1 was spending a large fraction of wall-clock time in pauses on your live set. Verify by summing G1's pause time over the run and expressing it as a percentage of run duration. If that percentage is large, the win is real and explicable. |
| Results differ wildly between the three repeats of one configuration | Your noise floor is too high to conclude anything. Look for interference: other containers, the load generator saturating, a shared database, thermal throttling on a laptop. Fix the harness before you fix the JVM. |

### Step 6 — write the explanation, which is the actual deliverable

Produce a short document with:

1. **The prediction** you made in Step 1, before running.
2. **The measured result**, three runs per configuration, with the noise floor stated.
3. **The mechanism** behind any regression. Not "ZGC was slower" — *"throughput fell
   because the load barrier executes on every reference load, and `orderflow`'s
   catalogue read path is dominated by Hibernate entity graph traversal, which is
   almost entirely reference loads."*
4. **The condition under which the decision would flip.** "If the payment gateway
   moves off the synchronous path and p99 becomes database-bound, GC pause becomes a
   larger share of the budget and this should be re-evaluated." That sentence is what
   separates a measurement from an engineering decision.
5. **A recommendation with a cost.** Not "stay on G1" — *"stay on G1; revisit if heap
   exceeds N GB or if p999 becomes the binding SLO."*

Carry two sentences out of this drill:

> *A collector that removes a term contributing 3% of p99 cannot improve p99 by more
> than 3%, and it will tax 100% of my reference loads to do it.*

> *"ZGC was worse" is not a criticism of ZGC. It is a statement about my workload, my
> heap size, and my CPU quota — and I can name which of the three did it.*

---

## Measurement

### The instrument for this topic

**The GC log plus a percentile comparison against the recorded Topic 65 baseline.**
Not a microbenchmark. Not a stopwatch. Not "it feels smoother".

The reason is structural. Collector behaviour is an emergent property of allocation
rate, live-set size, object lifetime distribution, reference density, heap size, CPU
quota and the collector's own control loop. **None of those are reproduced by a
microbenchmark.** A JMH benchmark of `new Order()` measures a pointer bump (Topic 68)
and tells you nothing about what the collector does with the resulting heap state
twenty seconds later.

The one place JMH belongs in this topic is Proof 6: isolating the **barrier cost** on
a reference-load-heavy operation. That is a legitimate microbenchmark question because
the barrier is a per-operation cost, and per-operation costs are exactly what JMH
measures well. Everything else is a load test.

### The protocol

1. **Identical everything except the collector.** Same heap, same container limits,
   same dataset, same cache state, same load script, same duration.
2. **Long enough to include many GC cycles.** A two-minute run under ZGC may not
   include a single old-generation cycle. Ten minutes minimum.
3. **Warm the JVM before measuring.** The first minute is interpreted and C1 code
   (Topic 74). A collector comparison that includes warm-up is comparing warm-ups.
4. **Three runs per configuration**, minimum, to establish the noise floor.
5. **Compare percentiles AND throughput.** A collector change that improves p99 and
   costs throughput is a trade, and reporting only half of it is misreporting.
6. **Compare per-endpoint, not aggregate.** The aggregate hides the most informative
   result: that a cheap read endpoint and an expensive write endpoint respond
   differently to the same change.
7. **Attach `jcmd <pid> VM.flags -all` output to every result.** A collector
   measurement without the flag dump is an anecdote.

### Why a naive `System.nanoTime()` loop is WRONG here

You will be tempted to write this:

```java
// DO NOT DO THIS. Every number it produces is meaningless.
long start = System.nanoTime();
for (int i = 0; i < 10_000_000; i++) {
    Order order = new Order(i, "PENDING");
}
System.out.println((System.nanoTime() - start) + " ns for 10M allocations");
```

Five independent reasons it lies, and you cannot tell which one is lying:

1. **Dead-code elimination.** `order` is never used. C2 can prove the allocation has no
   observable effect and delete it. You measure an empty loop. (Topic 75.)
2. **Escape analysis and scalar replacement.** Even if not eliminated outright, the
   object may never be allocated on the heap at all — its fields become registers.
   (Topic 75, and this is the *point* of Topic 75.)
3. **Constant folding.** Loop bound and constructor arguments are compile-time known.
   C2 has enormous freedom.
4. **On-stack replacement.** The loop starts interpreted, gets compiled while running,
   and is swapped mid-flight. Your average blends interpreter, C1 and C2 in a ratio
   determined by the loop count you happened to pick. (Topic 74.)
5. **Cold JIT.** The first thousands of iterations run in the interpreter. On a short
   loop that is most of your measurement.

And two reasons specific to *this* topic, which are the important ones:

6. **You are trying to measure a collector, and the collector is not in the loop.**
   GC cost is paid asynchronously, on GC threads, at collection time. A timer around
   allocations measures allocation. Even a perfectly correct JMH benchmark of
   allocation answers the wrong question.
7. **The load barrier is on reference LOADS, and your loop has none.** A loop that
   allocates and discards exercises exactly the operation ZGC does *not* tax. You
   would conclude the barrier is free. It is not; you just did not touch it.

**Topic 77 is the full treatment of JMH. Do not write a benchmark you intend to act on
until you have read it.**

### A correct JMH harness sketch, for the one question JMH can answer here

```java
package com.orderflow.bench;

import org.openjdk.jmh.annotations.*;
import org.openjdk.jmh.infra.Blackhole;
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.TimeUnit;

/**
 * The question: what does the load barrier cost on OUR object graph shape?
 *
 * Run under each collector as a SEPARATE JVM (-jvmArgs). The collector is a
 * process-wide property; it cannot be a @Param.
 */
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.MICROSECONDS)
@Warmup(iterations = 5, time = 1)
@Measurement(iterations = 10, time = 1)
@Fork(3)
@State(Scope.Benchmark)
public class OrderGraphTraversalBenchmark {

    /** A realistic orderflow shape: an order with lines, each referencing a product. */
    record Product(long id, String sku, long priceMinor) { }
    record OrderLine(long id, Product product, int quantity) { }
    record OrderGraph(long id, List<OrderLine> lines) { }

    @Param({"10", "200"})
    public int linesPerOrder;

    private List<OrderGraph> orders;

    @Setup(Level.Trial)
    public void setUp() {
        orders = new ArrayList<>();
        for (int o = 0; o < 5_000; o++) {
            List<OrderLine> lines = new ArrayList<>(linesPerOrder);
            for (int l = 0; l < linesPerOrder; l++) {
                lines.add(new OrderLine(l, new Product(l, "SKU-" + l, 100L * l), 1 + (l % 3)));
            }
            orders.add(new OrderGraph(o, lines));
        }
    }

    /** Pure traversal: reference loads, which is exactly what a load barrier taxes. */
    @Benchmark
    public long totalAcrossAllOrders() {
        long total = 0;
        for (OrderGraph order : orders) {
            for (OrderLine line : order.lines()) {
                total += line.product().priceMinor() * line.quantity();
            }
        }
        return total;
    }
}
```

| Annotation / flag | What it defends against |
|---|---|
| `@State(Scope.Benchmark)` | Holds state in a field JMH controls, so C2 cannot constant-fold the graph away. |
| Returning `total` | Defeats dead-code elimination without a `Blackhole` — a returned value is consumed by JMH. |
| `@Warmup(iterations = 5)` | Lets C2 compile and reach steady state before recording. Without it you measure warm-up (Topic 74). |
| `@Measurement(iterations = 10)` | Enough samples for a confidence interval that means something. |
| `@Fork(3)` | Three separate JVMs. Exposes run-to-run variance and defeats profile pollution between `@Param` values (Topic 74). |
| `@Param` | Each shape is a separate benchmark, so one call site does not see two profiles. |
| `-jvmArgs "-XX:+UseZGC ..."` | **The collector must vary across JVM invocations, never inside one.** |
| `-prof gc` | Reports allocation rate and bytes per operation from JMH's own instrumentation. |

```bash
java -jar target/benchmarks.jar OrderGraphTraversalBenchmark \
  -jvmArgs "-Xms4g -Xmx4g -XX:+UseG1GC" -prof gc -rf json -rff g1.json
java -jar target/benchmarks.jar OrderGraphTraversalBenchmark \
  -jvmArgs "-Xms4g -Xmx4g -XX:+UseZGC -XX:+ZGenerational" -prof gc -rf json -rff zgc.json
```

**WHAT TO LOOK FOR:** the confidence intervals. Report them, never the point estimate.
If they overlap, you measured nothing and saying otherwise is dishonest.

### The numbers to track continuously in production

| Number | Where from | Why |
|---|---|---|
| Longest pause, per interval | GC log, or `jvm.gc.pause` max | Under ZGC this should be tiny and boring. A change in it is a signal. |
| **Allocation stall count** | `grep -ci 'allocation stall'` on the GC log | **ZGC-specific. Should be zero. Any non-zero value is a headroom incident.** |
| Degenerated / Full GC count | GC log | Shenandoah's and G1's equivalents. Should be zero. |
| Cycle duration and cycle frequency | GC log | Under ZGC, rising cycle frequency is the leading indicator of the headroom problem, well before stalls appear. |
| Allocation rate (MB/s) | Micrometer `jvm.gc.memory.allocated`, or derived from the log | The input to every capacity conversation, and the number that decides whether a concurrent collector can win its race. |
| Live set | Heap occupancy after a cycle | Drives G1's pauses; drives ZGC's headroom requirement. |
| Container CPU utilisation | cgroup metrics | Concurrent GC competes for it. A collector change that raises CPU is spending your quota. |

Topic 118 covers exporting these without blowing up metric cardinality.

---

## Practice exercises

### 1 — easy: build the collector-availability and default matrix for your environment

For each JDK you can lay hands on — your local install, the CI image, and the exact
production base image:

1. Record the version string and the vendor.
2. Determine which of Serial, Parallel, G1, ZGC and Shenandoah are available.
3. Determine the **default** collector at `-Xmx1200m` with 2 CPUs visible, using
   `-XX:ActiveProcessorCount=2`.
4. Determine whether `ZGenerational` exists, and its default value.
5. Determine whether `UseCompressedOops` differs between G1 and ZGC at 8 GB.

Produce one table. Then answer in one sentence each:

- Which of these facts would change if someone bumped the base image by one JDK
  release, and how would you find out before it reached production?
- Your production base image and your laptop disagree about the ZGC default mode. Which
  measurements taken on your laptop are now invalid?
- Why is `jcmd <pid> VM.flags -all` a better source of truth than the deployment
  manifest?

### 2 — medium: the collector-choice audit (combines Topics 01, 21, 25, 65, 68, 69, 70, 71)

Below is a deployment manifest fragment and a description of a service. Find **six**
defects. Four are collector- or memory-related; two are from earlier topics. For each:
name the topic, state the **observable** symptom in a GC log or a latency percentile,
and write the fix.

```yaml
# orderflow-deployment.yaml (fragment)
resources:
  limits:
    cpu: "0.5"
    memory: "2Gi"
env:
  - name: JAVA_OPTS
    value: >-
      -XX:+UseZGC
      -Xmx2g
      -XX:MaxGCPauseMillis=10
      -XX:ParallelGCThreads=8
      -XX:+UseCompressedOops
```

And in the application, on the catalogue read path at 400 rps:

```java
@Service
public class ProductPricingService {

    private final Map<Long, Long> priceCache = new HashMap<>();

    public List<PricedProduct> priceAll(List<Product> products) {
        return products.parallelStream()
            .map(p -> new PricedProduct(p, priceCache.computeIfAbsent(p.id(), this::lookupPrice)))
            .collect(Collectors.toList());
    }

    private Long lookupPrice(Long productId) { /* database round trip */ }
}
```

Hints, in the order to think about them: one defect makes the container OOM-kill
itself; one flag is silently ignored by the collector that was chosen; one flag is
actively harmful given the CPU limit; one flag cannot do what it says under this
collector; one defect is a Topic 25 problem that gets worse when the CPU quota is
small; one is a Topic 01 boxing problem that inflates allocation rate on the hottest
path in the service.

For the collector defects specifically: state what the JVM will actually do with each
flag, and how you would prove it in one command.

### 3 — hard: production simulation — the collector decision, defended

**Part A — reproduce and record.** Run the Topic 65 baseline under G1 with full GC and
safepoint logging. Confirm ±10% against `/docs/java/baselines/`. Record all eight items
from the drill's Step 0, three times, and state your noise floor.

**Part B — predict.** Before running anything else, compute and write down:

1. The best possible p99 per endpoint if all GC pauses became zero.
2. The expected direction of the throughput change under ZGC, and the mechanism.
3. The expected footprint change from losing compressed oops, estimated with JOL on
   your three hottest entity classes multiplied by your measured live set.

**Part C — measure.** Run the identical profile under: G1 (control), generational ZGC,
non-generational ZGC (if your JDK still has it), and Shenandoah (if available). Three
runs each. Record the same eight items.

**Part D — the surprise.** For whichever configuration performed worst, find the
mechanism, not the label. Use: the GC log (stalls, degenerated collections, cycle
frequency), the safepoint log (rule out TTSP), `docker stats` (CPU saturation), and an
allocation profile (Topic 78). **Write one paragraph naming the mechanism.**

**Part E — change the workload, not the collector.** Now make ZGC's case stronger by
changing the system rather than the flag:

1. Raise the container to 4 vCPU and 8 GiB, heap to 6 GB. Re-run G1 and ZGC.
2. Add a large in-process Caffeine cache of products so the live set grows
   substantially. Re-run both.

**WHAT TO LOOK FOR:** whether the ranking flips, and at what point. **The heap size or
live-set size at which ZGC starts winning is the single most valuable number you will
produce in this exercise**, because it is the trigger condition for a future decision.

**Part F — write the decision record.** One page, in the form Phase 12 will demand:

- Context: the latency budget and where it goes.
- Decision: which collector, with the flag set.
- Evidence: the percentiles, the throughput, the noise floor, the flag dumps.
- Consequences: what you gave up.
- **Trigger for revisiting:** the specific measurable condition from Part E.

**Part G — argue against yourself.** You will most likely conclude "stay on G1". Make
the strongest possible case for ZGC on `orderflow` as it exists today. Then state what
would have to be true about the traffic, the SLO, or the infrastructure for that case
to win. If you cannot construct a serious opposing case, you have not understood the
trade.

---

## Interview questions

### Q1 — "When would you NOT use ZGC?"

**MID-LEVEL answer:** "ZGC is for large heaps. If your heap is small, G1 is fine. Also
ZGC uses more memory and CPU, so if you're resource-constrained you'd stick with G1."

**SENIOR answer:** "Most of the time, honestly — and I'd start by inverting the
question, because the interesting part is what has to be true for ZGC to *win*.

**I would not use ZGC when GC pause is not a meaningful share of my latency budget.**
That's the first thing I'd check and it settles most cases. If our `POST /orders` p99
is dominated by a payment-gateway round trip, then even eliminating GC entirely moves
p99 by a small percentage — while the load barrier taxes every reference load in the
process, forever. That's Amdahl's law applied to the JVM, and it's the most common way
this decision gets made wrong.

**I would not use ZGC on a small, CPU-constrained container.** Two specific mechanisms.
First, coloured pointers need all 64 bits, so compressed oops are off — every
reference field in the heap gets bigger, which on a 1.2 GB heap with a reference-dense
Hibernate entity graph is a real reduction in effective capacity. Second, ZGC does its
work concurrently, which means on CPUs my request threads want. On two vCPU,
'concurrent' means 'competing'.

**I would not use ZGC when I can't give it headroom.** A concurrent collector races
your allocation rate. Lose that race and you get allocation stalls — application
threads blocking on memory. That's worse than the pauses you were removing and much
harder to recognise, because the GC log still looks perfect.

**I would not use ZGC for a throughput-bound batch job.** If nobody is waiting on a
tail latency, pause time doesn't buy anything and the barrier cost is pure loss.
Parallel GC is often the right answer there and people forget it exists.

**And I would not use ZGC to fix a stall that GC didn't cause.** Before any collector
change I check `-Xlog:safepoint*`, because a reported five-millisecond pause can hide a
900-millisecond time-to-safepoint from one thread in a counted loop. ZGC doesn't fix
that. Nothing about the collector fixes that.

Where I *would* use it: large heaps where G1's pauses grow with the live set, a hard
tail-latency SLO where GC is a measurable share of the budget, or a service with a big
in-process cache. And I'd want the measurement, not the argument — the Topic 65 load
profile under both, three runs each, percentiles and throughput both reported."

**What separates them:** the mid answer knows the rule of thumb. The senior answer
inverts the question, names the *mechanism* of each cost (barrier on loads, compressed
oops off, CPU competition, headroom race), names a failure mode the mid answer has
never heard of (allocation stalls), rules out an unrelated cause first (TTSP), and
insists on measurement over argument. The step that most reliably impresses is
computing the ceiling — "the best possible improvement is X%" — before running anything.

**Interviewer's follow-up:** *"You said compressed oops are off. Does that actually
matter?"* — It depends on reference density, and it is measurable rather than
arguable. I'd run JOL on our three hottest entity classes under both collectors, take
the per-instance delta, and multiply by the live set measured from the GC log. On a
graph of orders, lines and products — which is mostly references — that product is
significant on a small heap. On a heap that is mostly primitive arrays, it is not.
Measure, do not assume.

---

### Q2 — "We switched to ZGC and throughput dropped. Explain what happened."

**MID-LEVEL answer:** "ZGC has more overhead than G1 because it does more work
concurrently. That overhead shows up as reduced throughput. You could tune it, or give
it more CPU."

**SENIOR answer:** "There are three candidate mechanisms and they leave different
fingerprints, so I'd distinguish them rather than guess.

**Mechanism one: the load barrier.** ZGC compiles a barrier into every reference load
— not every store, every *load*. That's field access, array element access, iterating a
collection, walking a Hibernate entity graph. The fingerprint is a **uniform, modest
regression across every endpoint, visible in p50, including endpoints that allocate
almost nothing.** A read-only catalogue endpoint getting slower is the signature,
because there's no GC event to blame. If I see that shape, it's the barrier.

**Mechanism two: CPU competition.** ZGC's marking and relocation run concurrently,
which in a CPU-quota'd container means they run on the same cores as the request
threads. The fingerprint is different: throughput falls **when GC cycles are active**
and recovers between them, and container CPU utilisation is higher for the same
request rate. I'd correlate cycle timestamps from the GC log against the throughput
time series.

**Mechanism three: lost compressed oops.** Below roughly 32 GB, G1 stores references
as 32-bit shifted values; ZGC can't, because coloured pointers need the full word. So
the same object graph occupies more heap, the live set grows, cycles get more frequent,
and cache locality worsens. The fingerprint is a larger live set for identical data,
which I can see directly by comparing heap occupancy after a cycle between the two runs.

Then two things I'd check before accepting any of them:

**Was it actually generational ZGC?** On JDK 21, `-XX:+UseZGC` alone gives you the
non-generational mode, which traces the whole heap every cycle. On a request-serving
workload where everything dies young, that's close to a worst case, and it's a
misconfiguration rather than a property of ZGC. `jcmd VM.flags -all` settles it.

**Were there allocation stalls?** If the collector lost the race against allocation,
threads were blocking on memory. That's a headroom problem, not a throughput property,
and the run should be discarded and repeated with more heap.

The honest framing for the write-up: this isn't ZGC being 'worse'. It's a trade we
executed. We spent throughput to buy pause predictability. The question is whether we
needed what we bought — and if p99 didn't improve, we paid for something we didn't
need."

**What separates them:** the mid answer says "overhead" without naming it. The senior
answer names three distinct mechanisms, gives each a **distinguishing fingerprint in
observable data**, checks for a misconfiguration and a failure mode before accepting
any of them, and reframes the result as an executed trade rather than a defect.

**Interviewer's follow-up:** *"How would you separate the barrier cost from the CPU
competition?"* — Run the traversal microbenchmark under both collectors with a heap
large enough that no collection happens during measurement. That isolates the barrier
with the concurrent work removed. Then compare the delta against the end-to-end load
test delta. The gap between the two is the CPU-competition component. Two measurements,
one subtraction.

---

### Q3 — "Our p99 has 2-second spikes and we're already on ZGC with sub-millisecond pauses. Diagnose it."

**MID-LEVEL answer:** "If ZGC's pauses are sub-millisecond then it's not GC. I'd look
at the database, or the network, or check whether there's a slow query."

**SENIOR answer:** "The mid answer's conclusion might well be right, but it skips the
two JVM-level causes that produce exactly this shape, and both are things a clean ZGC
log actively hides.

**First: time-to-safepoint.** ZGC's pause number is the time spent *at* the safepoint.
It excludes the time spent *reaching* it. If one thread is in a long counted `int` loop
over a large array, or a huge `System.arraycopy`, or waiting on a swapped-out page,
every other thread sits stopped until it arrives. The GC log reports the tiny part it
measures and says nothing about the rest. The tell is that the stall hits requests
already in flight on threads doing no GC work, across unrelated endpoints, at the same
instant. `-Xlog:safepoint*` reports the reaching time and the at-time as separate
fields. That's the first command I'd run, and it's a two-minute check.

**Second: allocation stalls.** ZGC is concurrent, so it races your allocation rate. If
it loses, application threads block waiting for free memory. That's not a pause and it
never appears as one — but it stalls requests. `grep -i 'allocation stall'` on the GC
log finds it immediately. If it's non-zero, the fix is headroom, CPU, or a lower
allocation rate — never a pause-goal flag, which ZGC doesn't even have.

**Third, and this one is not the JVM's fault at all: container CPU throttling.** If the
cgroup quota is exhausted, every thread in the container is descheduled for the rest of
the period. That looks exactly like a stop-the-world pause from inside the application
and appears in no JVM log whatsoever. I'd check the cgroup throttling counters —
`nr_throttled` and `throttled_time` — against the spike timestamps. Adding ZGC's
concurrent GC threads to a tight quota can *cause* this, which makes it a
collector-adjacent problem even though the JVM is innocent.

Only after those three would I go outside: database, downstream calls, connection pool
exhaustion — which for us is Topic 55's transaction-holding-a-connection problem — DNS,
and the load generator's own coordinated omission.

The ordering matters because the first three are cheap, fast, and rule out the
possibility that we chase the database for a week over a counted loop."

**What separates them:** the mid answer accepts the log at face value. The senior
answer knows the log *excludes* TTSP, knows ZGC's characteristic non-pause failure
mode, names an external cause that mimics a pause perfectly and appears in no JVM log,
and orders the checks by cost. The cgroup-throttling point is the one that marks
someone who has actually debugged this in Kubernetes.

**Interviewer's follow-up:** *"What if the safepoint log shows reaching time is the
whole problem?"* — Then it's Topic 73 and no collector helps. I'd find the thread with
`-XX:+SafepointTimeout -XX:SafepointTimeoutDelay=<ms>`, which prints the threads that
didn't arrive in time, and look for a counted `int` loop over a large array, a big
`System.arraycopy`, or a JNI critical section. The fix is in the application code, not
the JVM configuration.

---

### Q4 — "How do you choose a garbage collector?"

**MID-LEVEL answer:** "It depends on the application. G1 is a good default. If you need
low pauses, use ZGC or Shenandoah. If you need throughput, use Parallel GC. For small
apps, Serial GC."

**SENIOR answer:** "I start from the latency budget and the live set, not from the
collector list, because those two numbers determine the answer and everything else is
detail.

**Step one: what is the latency budget, and where does it currently go?** From the
recorded load-test baseline and the GC log I compute the share of p99 that is GC pause.
If that share is small, the collector question is closed — I go work on the dominant
term instead. This one calculation resolves most 'should we switch collectors'
conversations in ten minutes.

**Step two: how big is the live set, and is it growing?** G1's pauses scale with the
live set, because Object Copy scales with survivors. If the live set is a few hundred
megabytes, G1's pauses are manageable and tunable. If it is tens of gigabytes — a large
in-process cache, a big graph — G1's pauses become structurally hard to control, and
that is precisely the problem ZGC was built to solve. **Large heap is ZGC's actual
headline feature; the pause number is the symptom of it.**

**Step three: what resources do I have to spend?** A concurrent collector spends CPU
and memory headroom. In a container with a small CPU quota, 'concurrent' means
'competing with request threads'. If I cannot give the collector CPU and headroom, I
should not choose one that needs them.

**Step four: what does my binary actually contain?** Shenandoah is not in every JDK
distribution. ZGC's default mode changed across JDK releases — opt-in generational on
21, default on 23, non-generational removed on 24. Any collector plan that isn't
verified against the exact base image is a plan for a JVM I'm not running.

**Step five: measure, one variable at a time, three runs each, percentiles and
throughput both.** With the flag dump attached to every result.

My defaults, stated plainly: G1 unless I have a reason. Parallel for pure batch
throughput where nobody is waiting. ZGC when the heap is large or the tail-latency SLO
is tight *and* I have measured that GC is a meaningful share of it. Serial only for
tiny, single-CPU workloads — and I'd want to know if a container accidentally selected
it for me, which happens more often than people think."

**What separates them:** the mid answer is a lookup table. The senior answer is a
decision procedure that starts with two measured quantities, uses the collector list
only at the end, includes a step about what the binary actually contains, and states
the defaults with the conditions attached. Naming "large heap, not low pause" as ZGC's
real feature is the sentence that shows genuine understanding.

**Interviewer's follow-up:** *"What if you can't run a load test?"* — Then I make the
decision reversible and instrument it. Deploy the change to a canary with the GC log
and the safepoint log on, compare against production traffic rather than synthetic
load, and define the rollback trigger before deploying. A collector change is a
one-flag rollback, which makes it one of the safest experiments available — provided
you decided in advance what evidence would trigger the rollback.

---

### Q5 — "Explain coloured pointers and load barriers to someone who has only used G1."

**MID-LEVEL answer:** "ZGC stores metadata in the pointer, and there's a barrier that
checks it when you read a reference. That's how it can move objects without stopping
the application."

**SENIOR answer:** "The framing I'd use is: **G1 asks the collector to fix references
during a pause; ZGC asks the application to fix them as it runs.**

Under G1, when a young collection copies a live object, every reference to that object
must be updated to the new address before the application resumes. That's why the
copying is inside the pause, and it's why the pause grows with the live set — the work
is proportional to how much survives.

ZGC removes that requirement by putting state into the reference itself. Some bits of
a 64-bit reference aren't address bits, they're colour: whether the object has been
marked, whether it has been relocated in the current cycle. Then C1 and C2 compile a
**load barrier** into every reference load. The barrier tests the colour. On the fast
path — colour matches the current good colour — it costs a test and a well-predicted
branch. On the slow path, the barrier does the work itself: marks the object, or looks
up its new address in a forwarding table, and then **writes the corrected reference
back into the field it just read from**. That's the self-healing property, and it's
what makes the amortised cost tolerable: the second read of the same field is fast
again.

Because every thread repairs the references it touches, the collector can relocate
objects while the application runs. It never needs a stop-the-world phase to fix
references. So the pauses that remain contain only root-set work — scanning thread
stacks and flipping the colour — and that's why pause time is independent of heap size
and live-set size.

Three consequences I'd want the listener to take away.

**It requires 64-bit and it costs compressed oops.** Below about 32 GB, G1 stores
references as 32-bit shifted values. Coloured pointers need the whole word, so ZGC
gives that up, and every reference field gets bigger.

**The cost is on loads, not stores.** G1's write barrier fires when you write a
reference; ZGC's fires when you read one. Reads are vastly more common. That is the
structural reason ZGC costs more throughput than G1, and it is why the regression shows
up uniformly across every endpoint rather than at GC time.

**Shenandoah reaches the same place differently** — a load-reference barrier and a
forwarding pointer in the object header rather than colours in the pointer — with the
same trade shape and different vendor availability.

And the caveat I'd put at the end: none of this touches time-to-safepoint. The pauses
are short once every thread has arrived. Getting them to arrive is a separate problem
with a separate log."

**What separates them:** the one-sentence reframing ("the application fixes references
as it runs"), naming self-healing and why it matters for amortisation, deriving
"pause independent of heap size" from the mechanism rather than quoting it, connecting
it to the compressed-oops loss, identifying loads-versus-stores as the structural
throughput reason, and closing with the TTSP caveat that shows the model is complete
rather than memorised.

**Interviewer's follow-up:** *"What was multi-mapped memory for?"* — In the original
non-generational ZGC, the same physical pages were mapped at several virtual addresses,
one per colour, so a coloured pointer could be dereferenced directly without masking
the colour bits off. The visible side effect was alarming virtual-memory figures in
`top` while RSS was fine. My understanding is that the generational implementation
changed this, and I'd check what my specific JDK does rather than assert it — the
practical rule I'd give an on-call engineer is: under ZGC, check RSS, not VSZ.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. ZGC's pause contains root scanning but not marking, copying or reference fixup.
   Derive from that one fact alone why pause time is independent of heap size — and
   then name the one quantity ZGC's pauses **do** scale with, and one application-level
   change that would make it worse.

2. The load barrier fires on reference **loads**; G1's write barrier fires on reference
   **stores**. Without recalling any benchmark, predict which of these two workloads
   suffers more from switching G1 to ZGC, and why: (a) a service that walks a large
   Hibernate entity graph per request, (b) a service that allocates many short-lived
   `byte[]` buffers and does arithmetic on them.

3. Compressed oops are disabled under ZGC. Construct a scenario where switching to ZGC
   makes GC *more* frequent, purely as a consequence of that one fact, with no change
   to the application.

4. A concurrent collector races your allocation rate. Explain why G1 cannot fail in the
   same way, and what G1 does instead when it is under the equivalent pressure. Then
   say which of the two failure modes is easier to diagnose from a dashboard, and why
   that matters operationally.

5. Your p99 improves by a large margin under ZGC but your p50 gets slightly worse
   across every endpoint, including one that performs no allocation at all. Explain
   both halves with one mechanism each, and say which mechanism you would expect to
   dominate the *cost* side of the trade at 10x the current traffic.

6. On JDK 21, `-XX:+UseZGC` without `-XX:+ZGenerational` gives non-generational ZGC.
   Predict, from the weak generational hypothesis alone, how a request-serving workload
   behaves under that mode compared to the generational one — and name the specific
   line in the GC log that would let you tell which mode you were running.

7. You are asked to reduce p999 by half. You may change the collector, the heap size,
   the container CPU limit, or the application. Rank those four levers by expected
   effect **and** by risk, and defend the ordering. Then state the single measurement
   that would most change your ranking.

---

## Quick reference card

### JVM flags — ZGC

| Flag | Default | What it does | Set it? |
|---|---|---|---|
| `-XX:+UseZGC` | off | Selects ZGC | Only with a measured justification |
| `-XX:+ZGenerational` | **version-dependent** — opt-in on JDK 21, default JDK 23+, removed JDK 24+ | Selects generational mode | **On JDK 21, yes, always.** Verify with `VM.flags -all`. |
| `-Xms` / `-Xmx` | derived from `MaxRAMPercentage` | Heap bounds | **Yes, and set them equal.** ZGC needs headroom; give it deliberately. |
| `-XX:SoftMaxHeapSize` | `-Xmx` | Soft target; ZGC collects harder to stay below it but may exceed it | Useful for bursty services in fixed containers |
| `-XX:+ZUncommit` | on | Return unused heap to the OS | Leave on unless you need stable RSS for bin-packing |
| `-XX:ZUncommitDelay` | version-dependent | How long memory must be unused before uncommitting | Rarely |
| `-XX:ZCollectionInterval` | off | Force a cycle every N seconds | Diagnostic only |
| `-XX:ConcGCThreads` | derived from CPUs | Concurrent GC worker threads | Only when the container's CPU quota misleads the JVM (Topic 82) |
| `-XX:+UseCompressedOops` | **forced off under ZGC** | 32-bit shifted references | Cannot be enabled under ZGC. Do not try. |

### JVM flags — Shenandoah

| Flag | Default | What it does | Set it? |
|---|---|---|---|
| `-XX:+UseShenandoahGC` | off, **and absent from some JDK builds** | Selects Shenandoah | Verify availability in your exact base image first |
| `-XX:ShenandoahGCMode` | `satb` | `satb` / `iu` / `passive` / `generational` (experimental in recent JDKs) | Leave default unless you have measured a reason |
| `-XX:ShenandoahGCHeuristics` | `adaptive` | `adaptive` / `static` / `compact` / `aggressive` | Leave default. `aggressive` is a stress mode, not a production setting. |

### JVM flags — comparison and control

| Flag | Why it is here |
|---|---|
| `-XX:+UseG1GC` | The control for every experiment in this topic |
| `-XX:+UseParallelGC` | The throughput baseline. If your job is batch and nobody waits, measure this too. |
| `-XX:ActiveProcessorCount=N` | Simulate a CPU quota without changing the container. Topic 82. |
| `-XX:+HeapDumpOnOutOfMemoryError` | **Always, in production.** Topic 79. |
| `-XX:+PrintFlagsFinal` | The only trustworthy source of a default |

### Diagnostic commands

```bash
# What collector, what mode, what flags am I ACTUALLY running?
jcmd <pid> VM.flags -all | grep -iE "UseG1GC|UseZGC|ZGenerational|UseShenandoahGC|UseCompressedOops|SoftMaxHeapSize"
java -XX:+PrintFlagsFinal -version | grep -iE "UseZGC|ZGenerational|UseCompressedOops"

# Which collectors does this binary even contain?
for GC in UseSerialGC UseParallelGC UseG1GC UseZGC UseShenandoahGC; do
  printf "%-18s " "$GC"
  java -XX:+$GC -version >/dev/null 2>&1 && echo available || echo "NOT AVAILABLE"
done

# Collector beans from inside a running JVM (names identify the collector).
jcmd <pid> GC.heap_info

# Full GC log to a rotating file, with the decorators you need.
-Xlog:gc*,gc+heap=debug:file=gc.log:time,uptime,level,tags:filecount=5,filesize=50m

# The ZGC-specific failure. Should be zero.
grep -ci 'allocation stall' gc.log

# The Shenandoah-specific failure. Should be zero.
grep -cE 'Pause Degenerated|Pause Full|Cancelling GC' gc.log

# The G1-specific failure. Should be zero.
grep -cE 'Pause Full|To-space exhausted|Evacuation Failure' gc.log

# Rule out time-to-safepoint BEFORE blaming or crediting any collector. Topic 73.
-Xlog:safepoint*:file=safepoint.log:time,uptime,level,tags

# Turn logging on at runtime without a restart.
jcmd <pid> VM.log output=/tmp/gc-live.log what='gc*' decorators='time,uptime,level,tags'
jcmd <pid> VM.log disable

# Is the container throttling the CPU? A pause the JVM never took.
cat /sys/fs/cgroup/cpu.stat            # nr_throttled, throttled_usec
```

### How to read a ZGC log line — field guide

```
[<wall-clock>][<uptime>s][<level>][<tags>] GC(<id>) <Y|O>: <Minor|Major> Collection (<Cause>) <before>M(<pct>%)-><after>M(<pct>%) <cycle-duration>
[<wall-clock>][<uptime>s][<level>][gc,phases] GC(<id>) <Y|O>: Pause <PhaseName> <duration>ms
```

***Illustration of the format, not captured output.***

| Position | Field | How you use it |
|---|---|---|
| 1 | wall clock | correlate with k6 and the APM |
| 2 | uptime | intervals between cycles; the denominator of allocation rate |
| 3 | level | `info` for summaries, `debug` for tags you raised |
| 4 | tags | **grep on this**, not on message text |
| 5 | `GC(n)` | groups all lines of one cycle |
| 6 | `Y:` / `O:` | young or old — **its presence proves generational mode is active** |
| 7 | Minor / Major | which generation this cycle addressed |
| 8 | cause | **the diagnostic field** — Allocation Rate, Warmup, Proactive, Metadata GC Threshold, `System.gc()` |
| 9 | `before->after(pct)` | heap used before and after; `after` is live set plus floating garbage |
| 10 | **cycle duration** | **wall time of the concurrent cycle. NOT a pause.** The single most misread field in a ZGC log. |
| 11 | `Pause <phase> <ms>` | an actual stop-the-world pause. These should be tiny. |

Reading order when you open an unfamiliar ZGC log:

1. `grep -ci 'allocation stall'` — non-zero means headroom, and nothing else matters
   until it is fixed.
2. Look for `Y:` / `O:` prefixes — they tell you which ZGC you are running.
3. Longest `Pause` line — should be tiny. If not, go to the safepoint log.
4. Cycle **frequency** over time — rising frequency is the leading indicator of the
   headroom problem, well before stalls appear.
5. Cycle **causes** — a change in the mix of causes is a change in workload.
6. Only now consider flags.

### Gotchas checklist

- [ ] Print the collector and the mode. Never infer them. `jcmd VM.flags -all`.
- [ ] On JDK 21, `-XX:+UseZGC` alone is **non-generational**. Add `-XX:+ZGenerational`.
- [ ] Compressed oops are off under ZGC. Recompute your live-set footprint.
- [ ] Cycle duration is not pause duration. Do not report one as the other.
- [ ] Allocation stalls are ZGC's failure mode. Grep for them in every run.
- [ ] Shenandoah may not exist in your JDK build. Verify in the exact base image.
- [ ] Concurrent GC needs CPU. In a quota'd container, concurrent means competing.
- [ ] Compute the GC share of p99 **before** proposing a collector change.
- [ ] Rule out time-to-safepoint (Topic 73) before crediting or blaming any collector.
- [ ] Report percentiles **and** throughput. Half a trade is a misreport.
- [ ] Three runs per configuration. A change inside your noise floor is not a change.
- [ ] Attach the flag dump to every result. A measurement without it is an anecdote.

---

## When would I use this at work?

**1. Reviewing a pull request that changes the collector flag.**
Someone proposes `-XX:+UseZGC` to fix a p99 problem. You ask one question — what share
of p99 is GC pause? — and compute the ceiling from artefacts that already exist. If the
ceiling is small, you have prevented a throughput regression with a five-minute review
comment. If the ceiling is large, you have just turned a preference into a justified
experiment with a defined success criterion. Either way the PR gets better and the
conversation stops being about opinions.

**2. Capacity planning when the service's heap needs to grow.**
Product wants a large in-process product catalogue cache to cut database load. That
raises the live set substantially. You can state, with your own numbers, what happens
under G1: Object Copy scales with survivors, so young pauses grow, and past some live
set the pause becomes a real share of the latency budget. **You can name the live-set
size at which the collector decision flips** — from Part E of the hard exercise — and
that number turns "should we cache more" from an argument into a plan with a trigger
condition. This is the shape of a Phase 12 capacity model.

**3. An incident where the GC log looks perfect and the service is still stalling.**
On call, p999 spikes, ZGC's pauses are sub-millisecond, and everyone is about to spend
the night on the database. You run three commands: grep for allocation stalls, check
the safepoint log's reaching time versus at-time, and check the cgroup throttling
counters. Each takes under a minute and each rules out a whole class of cause. You know
that a clean GC log under a concurrent collector is *specifically weak evidence*,
because the two most common JVM stalls — TTSP and allocation stalls — do not appear in
it as pauses. That knowledge is the difference between a 20-minute diagnosis and a
wasted night.

---

## Connected topics

**Prerequisites:**

- **01 — Boxing and the Integer cache**: every boxed value is an allocation, and
  allocation rate is the quantity a concurrent collector races against. `Map<Long,
  Long>` counters on a hot path are a ZGC headroom problem as well as a G1 one.
- **21 — Lambdas and `invokedynamic`**: capturing lambdas allocate per evaluation and
  add reference fields; both the allocation rate and the reference density feed
  directly into this topic's trade.
- **25 — Parallel streams and the common ForkJoinPool**: parallel work multiplies
  allocation across threads and competes with concurrent GC threads for the same CPU
  quota. Under ZGC in a small container these two contend directly.
- **65 — The load-testing gate**: the recorded baseline in `/docs/java/baselines/` is
  the control for every measurement here. Without it, "ZGC is better" is an opinion.
- **68 — TLABs and promotion**: promotion rate determines how much work an old-
  generation cycle has to do, which determines whether a concurrent collector can win
  its race. Generational ZGC exists because this topic's arithmetic is still true.
- **69 — Object layout and compressed oops**: ZGC disables compressed oops. Topic 69's
  per-object arithmetic is how you quantify what that costs on your live set.
- **70 — GC fundamentals**: cost tracks the live set, not the garbage. The whole
  collector-choice decision is a statement about live set and latency budget.
- **71 — G1 in depth**: the control. Every claim here is relative to G1's regions,
  pause goal, evacuation failure and write barrier. Read 71 first; this document is
  its direct sequel and does not repeat it.

**This unlocks:**

- **73 — Safepoints and time-to-safepoint**: the number no GC log contains. **Read it
  next.** A sub-millisecond ZGC pause tells you nothing about how long your application
  was actually stopped, and under a low-pause collector that gap is the most common
  source of confusion.
- **74 — JIT and tiered compilation**: the load barrier is code C1 and C2 emit. The
  collector's cost and the compiler's output are the same story told from two ends, and
  warm-up interacts with every collector comparison you will run.
- **75 — Escape analysis**: an object that is scalar-replaced never reaches the heap,
  never needs a barrier, and never enters the collector's race. The cheapest possible
  form of GC tuning is not allocating.
- **77 — JMH**: the only correct way to measure the barrier cost, and the reason the
  naive `nanoTime` loop above is fiction.
- **78 — Profiling**: async-profiler's allocation mode names the *sites* driving the
  allocation rate that a concurrent collector must keep up with. The GC log says the
  rate; the profile says where.
- **79 — Heap dumps and MAT**: when the live set is too large for any collector to make
  cheap, the answer is retention, and the dominator tree is the tool.
- **80 — Off-heap memory**: ZGC's multi-mapping historically inflated virtual memory
  figures, and heap is only part of the footprint. Native memory tracking is how you
  tell an inflated VSZ from a real RSS problem.
- **82 — Containers and cgroup CPU sizing**: the CPU quota sizes `ConcGCThreads` and
  determines whether "concurrent" means "free" or "competing". The memory limit
  determines whether ZGC has headroom. **This topic's economics are entirely downstream
  of Topic 82's settings.**
- **83 — GraalVM native image**: no JIT, a different collector, a different footprint
  story. The contrast sharpens what the JIT-plus-concurrent-collector combination
  actually buys.
- **85 — `synchronized` and the mark word**: Shenandoah's historical forwarding-pointer
  design and the object header's contents are the same real estate. Header bits are a
  scarce, contested resource.
- **96 — False sharing**: concurrent GC threads and application threads share caches.
  The barrier's cost is partly a cache-behaviour story, one level below the flag.
- **101 — Virtual threads**: thousands of threads change the root-set size, which is
  the one quantity ZGC's pauses still scale with. Root scanning is where virtual
  threads and low-pause collectors meet.

---

*Java baseline 21, running on JDK 25. Four things in this document are deliberately
hedged rather than asserted: whether generational ZGC still relies on multi-mapped
memory, the exact ZPage size-class boundaries on your JDK, the precise experimental
status of generational Shenandoah on your build, and the exact deprecation semantics of
`-XX:+ZGenerational` across JDK 22 to 25. Each has a command in the Hands-on section
that settles it on your machine in under a minute. No pause duration, throughput
figure, percentile or speedup in this document was measured — every number you act on
must come from your own log and your own baseline. What has been stable and will still
be true at 2am: coloured pointers and load barriers make relocation concurrent, that
concurrency is paid for in throughput on every reference load, a concurrent collector
can lose a race that a stop-the-world collector cannot, and a collector that shortens
pauses cannot improve a p99 that pauses were not causing.*
