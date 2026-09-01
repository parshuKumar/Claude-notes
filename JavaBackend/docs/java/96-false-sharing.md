# 96 — False Sharing, Cache Lines and `@Contended`

## Phase: 9 — Concurrency
## Category: ELITE
## Java baseline: 21  |  Notes features from: 21 / 25
## Project spine: `orderflow`'s per-endpoint request counters — the small array of `long`s that every request increments. This topic is where you find out that a metrics object nobody has thought about since it was written can set the throughput ceiling of the whole service.

---

## Before anything else — what is and is not in this document

**I do not have a JVM, a profiler, or your hardware. Nothing in this document is
captured output.** There is no measurement here, and I will never present one.

Specifically, you will not find:

- a JMH table with `ns/op` or `ops/s` figures,
- a cache-miss count, a cache-miss *rate*, or any `perf` counter value,
- a "padding made it 4.7× faster" claim,
- a scalability curve with numbers on its axes,
- a JOL layout dump with real offsets in it.

That last one matters more here than anywhere else in Phase 9, because **object layout
is exactly the thing you must verify on your own JVM**. Header size, field ordering and
alignment depend on your JDK version, your heap size (compressed oops on or off), and
whether `[JAVA 25]` compact object headers are enabled. A layout I invented would be
wrong on at least one of those axes and you would build padding arithmetic on top of it.

What you get instead, every time:

- the **exact command with exact flags**,
- **WHAT TO LOOK FOR** in your own output,
- a **"what you see → what it means"** table covering the expected result and the
  surprising ones.

### Architecture facts I will state, each with a confirming command

These are properties of hardware and of the JDK source, not measurements:

- **x86-64 cache lines are 64 bytes.** Confirm: `getconf LEVEL1_DCACHE_LINESIZE`.
- **Apple Silicon reports a 128-byte cache line.** Confirm: `sysctl hw.cachelinesize`.
  This is the single most important number in the document for you, because you are on a
  Mac and every padding example you will read online assumes 64.
- **`LongAdder`'s inner `Cell` class is annotated `@jdk.internal.vm.annotation.Contended`.**
  Confirm by reading the JDK source, command in Proof 3.
- **`@Contended` is inert on application classes unless you pass `-XX:-RestrictContended`.**
  Confirm with JOL, Proof 4 — and this is the trap that makes people conclude "padding
  doesn't help".

If a command on your machine disagrees with any of these, the command is right.

### This topic is over-diagnosed, and I am going to keep saying so

False sharing is the most satisfying explanation in concurrency: invisible, physical,
solved by a clever trick. That is exactly why it gets blamed for scalability problems
that are actually lock contention, allocation pressure, GC, or a saturated downstream.
**Roughly nine times in ten, the wall you have hit is not false sharing.** This document
teaches you to construct it, measure it, and fix it — and, just as hard, to *rule it out*
in five minutes so you can go and find the real cause.

---

## Mechanical statement

Read this twice. Everything below is elaboration.

> **Cache coherence operates on 64-byte lines — 128 bytes on Apple Silicon. Two
> independent variables that land in the same line make every write by one core
> invalidate the other core's copy. The variables are logically independent and
> physically contended.**
>
> A CPU never reads or writes memory one byte at a time. It moves whole **cache lines**
> between memory and its private caches. The coherence protocol (MESI and its variants)
> tracks ownership **per line**, not per variable: to write any byte of a line, a core
> must hold that entire line in **Modified** state, which requires **invalidating every
> other core's copy of it**.
>
> So if `ordersPlaced` and `refundsProcessed` are adjacent `long` fields in the same
> object, they sit in one line. Core 0 incrementing `ordersPlaced` and Core 1
> incrementing `refundsProcessed` are not sharing data in any sense your program can
> see — and they are fighting over the same line, one round trip at a time, at a cost of
> tens to hundreds of cycles per exchange.
>
> The fix is **space**: put the two variables in different lines, by padding or by
> `@Contended`. You are trading memory for the absence of coherence traffic.

Three consequences you will use:

1. **The symptom is a scalability wall, not a slowdown.** Single-threaded throughput is
   unaffected. Two threads are a bit slower than one thread should predict. At eight
   threads throughput has stopped improving; at sixteen it is *going down*. Any
   measurement at one thread will tell you everything is fine.
2. **It is invisible in every artefact you normally read.** Not in a thread dump — the
   threads are `RUNNABLE` and working. Not in a flame graph — the time is inside the
   increment instruction, attributed to your own method. Not in GC logs. Not in lock
   contention metrics, because there is no lock.
3. **The fix has a cost.** Padding a 16-byte object to 128 bytes is an 8× memory
   multiplier on that object. Do it to a hot, small, per-thread structure and you win. Do
   it to a million-instance domain object and you have traded a coherence problem for a
   cache-capacity problem and a GC problem.

---

## The bridge from what you know

### `NO TYPESCRIPT ANALOGUE.`

This is not a partial analogue, a rhyme, or a thing you can approach sideways. There is
nothing in the JavaScript runtime, in Node, in NestJS, or in your entire mental model of
how programs execute that corresponds to false sharing.

**Why there is nothing:** false sharing requires **two cores writing to memory at the
same instant**. Your V8 isolate has one thread. It runs on one core at a time. There is
no second core with a private cache holding a stale copy of anything the main thread
owns, because there is no second core executing your code. The MESI protocol is running
underneath you constantly — it is a property of the hardware, not of the runtime — but
your program cannot cause a coherence conflict with itself.

You have never had this bug. You could not have had it. There is no analogy, no
approximation, and no "it's a bit like…". Reaching for one would install a wrong model
that costs more to remove than the comfort is worth.

**The two near-misses, and why they are near-misses:**

- **`worker_threads`** genuinely run on multiple cores. But each worker has its own V8
  isolate and its own heap, and they communicate by *copying* through
  `postMessage`. There is no shared object whose adjacent fields two workers write. The
  isolation that makes workers safe is the same isolation that makes false sharing
  impossible.
- **`SharedArrayBuffer` + `Atomics`** is the one place JavaScript has real shared memory
  across real threads — and here false sharing **can genuinely occur**. Two workers doing
  `Atomics.add(view, 0, 1)` and `Atomics.add(view, 1, 1)` on adjacent `Int32Array` slots
  are contending on one cache line for exactly the reasons this document describes. If
  you have written `SharedArrayBuffer` code, you have the hardware model already and this
  document is telling you something you could have hit. Almost nobody has written that
  code. If you have not, treat this section as: **nothing transfers.**

### What *does* transfer — from Phase 8, not from Node

Your bridge into this topic is Topic 69, not TypeScript:

| From Topic 69 (object layout) | Used here as |
|---|---|
| An object is a header plus fields, laid out at fixed offsets | the offsets are what decide which line a field lands in |
| Header = mark word + class word; compressed oops shrink references to 4 bytes | changes the offset of your first field, which shifts everything |
| Objects are 8-byte aligned; the JVM inserts padding to reach a multiple of 8 | you are going to insert *much more* padding, deliberately, for a different reason |
| Field ordering is chosen by the JVM (roughly longs/doubles, then ints, then shorts, then refs), **not** by declaration order | which is why naively declaring `long p1..p7` between two fields does not put them there |
| JOL is how you see the truth | the primary instrument in this entire document |

And from Topic 11: you already accepted that `ArrayList` beats `LinkedList` because of
cache locality. That was the *good* side of cache lines — one fetch brings you the next
several elements. **This document is the bad side of the identical mechanism.** Locality
helps when one core reads many nearby values; it hurts when many cores write nearby
values. Same hardware, same line, opposite sign.

---

## What is this?

**False sharing** is when two threads on two cores write to two *different* variables
that happen to occupy the same cache line, and pay the full cost of contention even
though they share nothing logically.

The word "false" is doing precise work. **True sharing** is two threads writing the same
variable — that is a real data race and you fix it with a lock, an atomic, or by not
sharing. **False sharing** is two threads writing different variables and getting the
performance of true sharing anyway, with none of the correctness problems. Your program
is correct. It is just slow, and it gets slower as you add cores, which is the opposite
of what everyone expects.

### The three places it actually happens

**1. Adjacent fields in a shared object.** The textbook case:

```java
class Stats {
    long ordersPlaced;      // written by request threads
    long refundsProcessed;  // written by refund workers
}
```

Two `long`s, 16 bytes, guaranteed in one cache line on any machine. This is the case
everyone teaches, and it is the least common in real code — because most such fields are
`AtomicLong`s or are behind a lock, which moves the problem rather than removing it.

**2. Adjacent elements in a shared array.** Far more common in practice:

```java
final AtomicLongArray perEndpointCounters = new AtomicLongArray(8);
// thread handling /orders   -> index 0
// thread handling /products -> index 1
```

Eight `long`s is 64 bytes. On x86-64 that is one line; on Apple Silicon it is half of
one. Every counter in that array is fighting every other counter, and the array looks
completely innocent. **This is the shape you will meet in `orderflow`.**

**3. A hot object adjacent in the heap to another hot object.** Two separately-allocated
small objects can land in the same line because the allocator bumped a TLAB pointer and
put them next to each other (Topic 68). You have no control over this and no way to
predict it. It is the reason `@Contended` pads on *both* sides of a field rather than
just after it.

### The counter-example that keeps you honest

```java
class Order {
    long id;
    long customerId;
    BigDecimal total;
}
```

Adjacent fields, one cache line, and **no false sharing whatsoever** — because one
thread owns each `Order` and no two cores write it concurrently. False sharing needs
*concurrent writes from different cores to the same line*. Adjacency alone is not the
bug; adjacency plus concurrent multi-core writes is.

Say the precondition out loud every time, because it is the test that rules the
diagnosis in or out in ten seconds:

> **Is this line written by more than one core at a time?** If no, you do not have false
> sharing, however adjacent the fields are.

---

## Why does it matter?

**1. It is the explanation for a scalability wall that has no other explanation.**

You have a service. You add cores. Throughput improves to four threads, plateaus at
eight, and *declines* at sixteen. There is no lock in the hot path — you checked. GC is
quiet. The downstream is not saturated. Allocation rate is flat. CPU utilisation is
**high**, which makes it look like you are compute-bound, except you are doing less work
per second than you were at eight threads.

That combination — high CPU, falling throughput, no lock, no GC — is one of the few
signatures that points here. It is also the signature of a contended CAS loop (Topic 95),
and the two are physically the same phenomenon: cache-line ownership ping-pong. The
difference is only whether the contending threads are hitting the same variable or
different variables in the same line.

**2. It explains a design decision inside the JDK that you would otherwise find bizarre.**

Topic 95 taught you that `LongAdder` beats `AtomicLong` under contention by giving each
contending thread its own `Cell`. **That only works because `Cell` is `@Contended`.**
Without the padding, N cells in an array would occupy `N × 16` bytes — several per cache
line — and the threads would contend on lines instead of on one variable. The striping
would buy you almost nothing. `LongAdder`'s entire performance argument rests on a
padding annotation. So does `ConcurrentHashMap`'s `CounterCell`, which is how `size()`
scales (Topic 92).

If you understand this topic, those two classes stop being magic.

**3. Because it is over-diagnosed, being able to rule it out is worth more than being
able to find it.**

The senior move is not "I found false sharing". It is: *"here are the four explanations
for a scalability wall, here is the five-minute check that eliminates three of them, and
here is the JMH curve that would distinguish the fourth."* Padding on a hunch is a
category of engineering failure — it costs memory, it obfuscates the code, it survives
in the codebase forever, and it usually does nothing. Trap 1 is that failure.

---

## Machine-level reality

### The memory hierarchy, and the unit that matters

A modern core does not talk to RAM. It talks to its own L1 data cache, which talks to L2,
which talks to a shared L3 (on x86) or a system-level cache (on Apple Silicon), which
talks to memory. Each level is bigger and slower.

The unit of transfer between every level is the **cache line**. Not a byte, not a word —
a line. You read one byte, the hardware fetches 64 (or 128) bytes containing it. You
write one byte, the hardware must own all 64 (or 128).

Access costs rise by roughly an order of magnitude per level: an L1 hit is a few cycles,
L2 a couple of dozen, L3/SLC tens, main memory a couple of hundred. The one that generates
this document sits between the last two: **fetching a line from another core that holds it
Modified costs tens of cycles plus protocol traffic — and under contention that happens on
every single write, not once.**

### MESI: the four states, and the one that hurts

Every cache line, in every core's cache, is in one of four states:

| State | Meaning | Can I read? | Can I write? |
|---|---|---|---|
| **M** — Modified | I have the only copy, and it differs from memory | yes | yes |
| **E** — Exclusive | I have the only copy, and it matches memory | yes | yes (transitions to M) |
| **S** — Shared | Several cores have this line, all matching memory | yes | **no** — must upgrade first |
| **I** — Invalid | My copy is stale; I do not have this line | no | no |

The rule that produces false sharing is short:

> **To write any byte of a line, a core must hold that line in M or E. Getting there
> requires every other core holding it in S or M to transition to I.**

That transition is a **Request For Ownership** (RFO). It is a message on the interconnect.
The other core must acknowledge, and if it held the line in M it must first write back or
forward the data. This is not instantaneous and it does not scale: with N cores writing
the same line, each write requires invalidating N−1 copies.

### The ping-pong, step by step

Two cores, one line, two logically independent variables in it.

| Step | Core 0 (writes `ordersPlaced`) | Core 1 (writes `refundsProcessed`) | Line state in each cache |
|---|---|---|---|
| 1 | reads the line → **E** | — | C0: E · C1: I |
| 2 | writes `ordersPlaced` → **M** | — | C0: M · C1: I |
| 3 | — | wants to write → RFO | C0 must give it up |
| 4 | line invalidated | receives line in **M** | C0: **I** · C1: M |
| 5 | wants to write again → RFO | line invalidated | C0: M · C1: **I** |
| 6 | writes | wants to write → RFO | and around again, forever |

Each core's write **stalls waiting for the line**. Neither core is doing anything wrong.
Neither variable is shared. The line is shared, and the line is the unit of coherence.

Now notice the crucial asymmetry: **if both cores were only reading, both would hold the
line in S simultaneously and nothing would happen.** Read-mostly data never false-shares.
It is writes — and only writes — that force exclusivity. That is why the fix for a
read-heavy structure is never padding, and why Topic 94's `StampedLock` optimistic read
(which performs no stores) is fast for exactly this reason.

### Cache-line geometry, and how to compute padding

You need three numbers before you can pad anything.

**Number 1 — the line size.** Confirm it, do not assume it:

```bash
# macOS (Intel or Apple Silicon):
sysctl hw.cachelinesize

# Linux:
getconf LEVEL1_DCACHE_LINESIZE
cat /sys/devices/system/cpu/cpu0/cache/index0/coherency_line_size
```

x86-64 reports **64**. Apple Silicon reports **128**. If you pad for 64 bytes on an M-series
Mac, **you have padded half a line and the false sharing remains**. Every blog post you
will find assumes 64. This is Trap 5 and it is the one that will actually get you.

**Number 2 — the object header size.** From Topic 69, and it depends on your JVM:

| Configuration | Header size |
|---|---|
| Compressed oops (default, heap < 32 GB), JDK 21 | 12 bytes (8-byte mark + 4-byte compressed class) |
| No compressed oops (`-XX:-UseCompressedOops` or heap ≥ 32 GB) | 16 bytes |
| `[JAVA 25]` `-XX:+UseCompactObjectHeaders` | **8 bytes** — and this changes every offset below it |

**Do not compute this. Read it with JOL.** The command is in Proof 1. A header size you
assumed is a padding calculation that is silently wrong.

**Number 3 — the JVM's field ordering.** This is the one that defeats naive padding. The
JVM does **not** lay out fields in declaration order. It groups them, broadly largest
first — `long`/`double`, then `int`/`float`, then `short`/`char`, then `byte`/`boolean`,
then references — to minimise alignment gaps. So this:

```java
class Naive {
    volatile long ordersPlaced;
    long p1, p2, p3, p4, p5, p6, p7;   // "padding"
    volatile long refundsProcessed;
}
```

…gives the JVM nine `long` fields and complete freedom to place `ordersPlaced` and
`refundsProcessed` adjacent to each other with all seven padding fields after them. **The
padding is real memory and does nothing.** Trap 3.

**The arithmetic, once you have the three numbers.** To give one `long` field a line to
itself on a 128-byte machine:

```
line size                        128
header                            -12   (JOL-confirmed, JDK 21 compressed oops)
the field itself                   -8
                                 ----
padding needed after the field     108  -> 14 longs (112 bytes), rounding up
padding needed before the field    116  -> 15 longs, if the object might be adjacent
                                          in the heap to another hot object
```

Which is why `@Contended`'s default padding is **128 bytes** (`-XX:ContendedPaddingWidth`,
default 128) and why it pads on **both sides**: it is defending against 64-byte lines with
an adjacent-line prefetcher pulling pairs, and against heap neighbours you cannot see.
Two lines of padding is not paranoia; it is the honest answer to "I do not know what is
next to me".

### The layout trick that actually works without `@Contended`

Field *reordering* is per-class. The JVM lays out a superclass's fields before a
subclass's. So you force the ordering by using an inheritance chain — the idiom the LMAX
Disruptor made famous:

```java
abstract class LhsPadding      { byte p00,p01,p02,p03,p04,p05,p06,p07,
                                      p08,p09,p10,p11,p12,p13,p14,p15,
                                      /* ...to 128 bytes... */ p120; }
abstract class Value extends LhsPadding { volatile long value; }
abstract class RhsPadding extends Value { byte q00,q01,/* ...to 128 bytes... */ q120; }
public final class PaddedCounter extends RhsPadding { /* accessors */ }
```

Superclass fields first, so `value` is guaranteed to have the left padding before it and
the right padding after it, and no reordering can interleave them. It is ugly, it is
portable, it needs no flags, and **it is what you ship when you cannot use `@Contended`.**

**Verify it with JOL. Always.** The whole point is a layout claim, and a layout claim you
have not checked is a guess.

### `@Contended` — what it is, and why it usually does nothing

```java
import jdk.internal.vm.annotation.Contended;

public class Stats {
    @Contended volatile long ordersPlaced;
    @Contended volatile long refundsProcessed;
}
```

Three facts, stated plainly, because every one of them trips people:

1. **The annotation is `jdk.internal.vm.annotation.Contended`.** There was a
   `sun.misc.Contended` before JDK 9; it is gone. There is **no public, supported,
   application-facing equivalent** in Java 21 or 25.
2. **It is not exported to application code.** `java.base` does not export
   `jdk.internal.vm.annotation` to unnamed modules. To *compile* against it you need:
   ```
   javac --add-exports java.base/jdk.internal.vm.annotation=ALL-UNNAMED ...
   ```
   and to run without warnings, the same flag on `java`. Your build tool must carry it,
   and it will not survive a dependency upgrade unnoticed.
3. **It is inert on non-boot-classpath classes unless you disable the restriction.** The
   JVM honours `@Contended` only on trusted classes by default. For your class you need:
   ```
   -XX:-RestrictContended
   ```
   **Without that flag the annotation compiles, runs, and does absolutely nothing.** JOL
   shows no padding. Your benchmark shows no improvement. You conclude that false sharing
   was not the problem. That conclusion is wrong and it is Trap 2 — the most expensive
   trap in this document, because it produces a *confidently wrong negative result*.

**The honest recommendation for application code: use manual padding via the inheritance
idiom, or restructure so the contention does not exist.** `@Contended` is the right tool
inside the JDK, where the classes are trusted and the flags are unnecessary. In your
service it is two JVM flags, an `--add-exports`, a non-public API, and a silent failure
mode. That is a poor trade for something a superclass can do portably.

**Where you should absolutely know it exists:** reading JDK source. `LongAdder.Cell`,
`ConcurrentHashMap.CounterCell`, `ForkJoinPool`'s internals and `Thread`'s
`threadLocalRandomProbe` are all `@Contended`, and if you do not know what the annotation
means those classes are unreadable.

### `[JAVA 25]` compact object headers change the arithmetic

`-XX:+UseCompactObjectHeaders` reduces the header from 12 bytes to 8. Every field offset
below it shifts by 4 bytes, which means **every padding calculation in every article
written before JDK 24 is off by four bytes on a JVM with this flag on**. It does not
change any concept here. It changes one number, and that number is an input to your
padding arithmetic.

**Do not compute it. Settle it with JOL**, on the exact JVM and flags you will deploy.
That is Proof 1, and it takes ninety seconds.

---

## Concurrency trace

**Before any correct code.** This is `orderflow`'s request-counter object under the Topic
65 baseline, traced at the cache-line level.

**The setup.** Someone added per-endpoint request counting. It is four lines, it has no
locks, it uses `AtomicLong` so it is obviously thread-safe, and it passed review in
ninety seconds:

```java
@Component
public class EndpointCounters {
    final AtomicLong orders   = new AtomicLong();   // POST /orders
    final AtomicLong products = new AtomicLong();   // GET  /products
    final AtomicLong wallets  = new AtomicLong();   // GET  /wallets
    final AtomicLong payments = new AtomicLong();   // POST /payments
}
```

Four `AtomicLong` objects, allocated one after another in the same TLAB (Topic 68), so
they land **consecutively in the heap**. Each is a 12-byte header plus an 8-byte `long`,
padded to 24 bytes by 8-byte object alignment. Four of them is 96 bytes.

On Apple Silicon, with a 128-byte line, **all four fit in one or two cache lines.** On
x86-64 with 64-byte lines, at least two of them share a line, and which two depends on
where the first one happened to be allocated.

Four request threads, on four cores, each serving a different endpoint. They share
nothing logically: four endpoints, four counters, four cores.

| Step | Core 0 — `http-exec-3` (POST /orders) | Core 1 — `http-exec-9` (GET /products) | Cache-line state / outcome |
|---|---|---|---|
| 1 | `orders.incrementAndGet()` → needs line L in **M** | — | C0: L=**M** · C1: L=I |
| 2 | `lock xadd` executes; ~cycles, no contention | — | counter incremented. Fast. |
| 3 | — | `products.incrementAndGet()` → `products` is in **the same line L** → RFO | C0 must relinquish |
| 4 | **stalls**: line invalidated mid-flight | acquires L in **M** | C0: **I** · C1: M |
| 5 | next request arrives → `orders.incrementAndGet()` → RFO | **stalls** | C1: **I** · C0: M |
| 6 | writes | next request → RFO | ping-pong established |
| 7 | Cores 2 and 3 join with `wallets` and `payments` | | **four cores serialised on one line** |
| 8 | each increment now costs an interconnect round trip instead of an L1 hit | | the counters have become a **global serialisation point** |
| 9 | throughput at 4 threads is barely above 2 threads | | scaling has stopped |
| 10 | ops team adds pods; each pod's CPU is high, so it looks compute-bound | | **more cores per pod makes each pod slower** |
| 11 | `POST /orders` p99 begins to include line-acquisition stalls on every request | | latency rises with no code change and no traffic change |

**Outcome, in business terms.**

`orderflow` stops scaling at four cores. The Topic 65 baseline was recorded on a 2-core
container; when the team moves to 8-core nodes to handle Black Friday, throughput per pod
improves by far less than 4× and p99 gets *worse*. The capacity model (Topic 129) predicted
linear scaling and was wrong, so the Black Friday provisioning is undersized, and the
team's response is to add more pods — which costs money and does not fix the per-pod
ceiling.

CPU utilisation is high on every pod, so every dashboard says "we are compute-bound, we
need bigger instances". Bigger instances have **more cores**, which makes it worse. The
investigation looks at the database (fine), the downstream (fine), GC (fine) and locks
(there are none in the path). Three engineers spend a week. The cause is four counters
that no product feature depends on, occupying one cache line, incremented by every request.

**And the sentence that makes it hurt:** the counters were added *to help diagnose a
performance problem*.

### The same ten seconds, with the counters on separate lines

One change: each counter is padded to occupy a cache line alone.

| Step | Core 0 — `http-exec-3` | Core 1 — `http-exec-9` | Cache-line state / outcome |
|---|---|---|---|
| 1 | `orders` increment → line L0 in **M** | `products` increment → line L1 in **M** | two different lines, both Modified, **simultaneously** |
| 2 | writes, L1-hit speed | writes, L1-hit speed | no RFO, no invalidation, no interconnect traffic |
| 3 | next request → L0 still **M**, still owned | next request → L1 still **M** | **each core keeps its line indefinitely** |
| 4 | Cores 2 and 3 join on L2 and L3 | | four cores, four lines, zero coherence traffic |
| 5 | throughput scales with cores | | the wall is gone |

**Outcome, in business terms.** Per-pod throughput scales with core count as the capacity
model predicted. The Black Friday provisioning is right. The cost is **memory**: four
counters went from 96 bytes to roughly 512 bytes. That is 416 wasted bytes, once, for the
lifetime of the process, in exchange for the service's ability to use the cores you are
paying for.

**Read the trade in the right direction.** You did not make the counters faster. You made
them *stop making everything else slower*. And you spent memory to do it — which is only a
good trade because there are four of these objects, not four million. Trap 4 is what
happens when you make the same trade on four million.

---

## Example 1 — minimal

Two counters, two threads, one object. The whole phenomenon in thirty lines.

```java
public final class TwoCounters {

    // Both fields in one object. On any machine these are adjacent
    // and therefore in the same cache line.
    public static final class Shared {
        volatile long a;    // written only by thread A
        volatile long b;    // written only by thread B
    }

    // The fix: each counter alone in a line, via the inheritance idiom
    // so the JVM cannot reorder the padding away.
    abstract static class Lhs   { long p1,p2,p3,p4,p5,p6,p7,p8,
                                       p9,p10,p11,p12,p13,p14,p15; }   // 120 bytes
    abstract static class Val extends Lhs { volatile long value; }
    public static final class Padded extends Val {
        long q1,q2,q3,q4,q5,q6,q7,q8,q9,q10,q11,q12,q13,q14,q15;       // 120 bytes
    }
}
```

**What to notice:**

- **`volatile` is doing two jobs and you must keep them apart.** It gives visibility and
  ordering (Topic 87) — which is why the writes actually reach memory and can contend at
  all. It does **not** cause false sharing; plain `long` fields written by two cores
  contend identically. `volatile` makes the effect *reliable and measurable* by preventing
  the JIT from keeping the value in a register, which is exactly why every false-sharing
  demonstration uses it. Without it, C2 may hoist the increment out of the loop entirely
  and you measure nothing (Topic 75).
- **15 `long`s, not 7.** `15 × 8 = 120` bytes, which with the field and header covers a
  128-byte line. Seven would be right for x86-64 and **wrong on your Mac**.
- **The inheritance chain is not stylistic.** It is the only portable way to guarantee the
  padding surrounds the field rather than being grouped after it by the JVM's size-ordered
  field layout.
- **The numbers here are a starting point, not a result.** `Padded`'s actual layout is
  what JOL says it is on your JVM with your flags. Proof 1.

### The benchmark that would demonstrate it — and the one that would not

```java
// WRONG. This measures nothing useful and will produce a confident, false number.
long t0 = System.nanoTime();
for (int i = 0; i < 100_000_000; i++) { shared.a++; }
System.out.println(System.nanoTime() - t0);
```

Four independent reasons, all from Topic 77: no warm-up, so most of it runs interpreted;
C2 may prove the result unused and delete the loop; **it is single-threaded, and false
sharing does not exist with one thread**; and one run gives no variance so you cannot tell
signal from noise.

**False sharing is a multi-thread scalability phenomenon. It can only be measured as a
curve across thread counts.** A single number at any thread count is not evidence. The
correct harness is in the Measurement section, and its defining feature is
`@Threads({1,2,4,8,16,32,64})` — because **the shape of the curve is the result**.

---

## Example 2 — production scenario (on the project spine)

### The constraints, stated before any code

| Constraint | Value | Source |
|---|---|---|
| Request rate | 400 rps baseline, 4,000 rps Black Friday target | Topic 65 + the capacity model |
| Counters required | one per endpoint (4), plus a global total | the metrics requirement |
| Container | 8 vCPU on the target nodes | infrastructure |
| Counter read frequency | once per 15 s, by the Micrometer scrape | Topic 118 |
| Counter write frequency | once per request, per endpoint | by definition |
| Accuracy requirement | exact totals; these feed billing reconciliation | product |

**Read the third and fourth rows together.** These counters are written 4,000 times a
second and read four times a minute. That write/read ratio — roughly 15,000:1 — is the
single most important fact in the design, and it points at exactly one answer.

### The code that ships and caps the service at four cores

```java
@Component
public class EndpointCounters {

    private final AtomicLong orders   = new AtomicLong();
    private final AtomicLong products = new AtomicLong();
    private final AtomicLong wallets  = new AtomicLong();
    private final AtomicLong payments = new AtomicLong();

    public void recordOrder()    { orders.incrementAndGet(); }
    public void recordProduct()  { products.incrementAndGet(); }
    public void recordWallet()   { wallets.incrementAndGet(); }
    public void recordPayment()  { payments.incrementAndGet(); }

    public long orders() { return orders.get(); }
    // ...
}
```

This has **two** problems stacked on each other, and separating them is the skill:

1. **True sharing** on each individual counter. Every request thread hitting `/orders`
   CASes the *same* `AtomicLong`. That is Topic 95's contended-CAS problem, and it is the
   larger of the two effects.
2. **False sharing** between the four counters, because they are four small objects
   allocated together and landing in one or two lines.

Fixing only the second one leaves the first. Fixing only the first one — as you are about
to see — fixes both, which is why the correct answer here is not padding at all.

### Fix 1 — the right answer: `LongAdder`

```java
@Component
public class EndpointCounters {

    private final LongAdder orders   = new LongAdder();
    private final LongAdder products = new LongAdder();
    private final LongAdder wallets  = new LongAdder();
    private final LongAdder payments = new LongAdder();

    public void recordOrder()   { orders.increment(); }
    public long orders()        { return orders.sum(); }
    // ...
}
```

**Why this is the answer, and why it belongs in a false-sharing document:**

`LongAdder` (Topic 95) keeps a base value plus an array of `Cell`s. Under contention each
thread hashes to its own `Cell` and increments that, so threads stop fighting over one
variable — that fixes problem 1. And **`Cell` is `@Contended`**, so the cells are padded
apart and threads stop fighting over one *line* — that fixes problem 2, using exactly the
mechanism this document teaches, written by people who could legitimately use the
annotation.

`sum()` walks the cells and adds them up. That costs more than `AtomicLong.get()`, and it
is not atomic with respect to concurrent updates — a `sum()` taken during writes is a
value that was never simultaneously true. For a counter read four times a minute and
written 4,000 times a second, both costs are irrelevant. **The write/read ratio chose the
data structure.**

**This is the senior answer to the whole topic:** you did not pad anything. You picked a
structure whose author already did, correctly, with flags you cannot use.

### Fix 2 — when you genuinely must pad: the per-shard array

`LongAdder` is right for counters. It is not right for everything. Suppose `orderflow`
shards its in-memory inventory reservation state by SKU hash, one shard per core, and each
shard has a mutable `reservedUnits` count that only its owning thread writes:

```java
// The obvious version. Every shard's counter shares lines with its neighbours.
final long[] reservedUnits = new long[SHARD_COUNT];
```

Eight `long`s is 64 bytes — one whole line on x86-64, half a line on Apple Silicon.
Perfectly partitioned logically; completely contended physically.

Two portable fixes:

```java
// (a) Stride the array so each live slot sits in its own line.
//     LINE = 128 on Apple Silicon, 64 on x86-64. CONFIRM with sysctl.
private static final int LINE  = 128;
private static final int SLOTS = LINE / Long.BYTES;      // 16 longs per line
private final long[] reservedUnits = new long[SHARD_COUNT * SLOTS];

private long read(int shard)            { return reservedUnits[shard * SLOTS]; }
private void write(int shard, long v)   { reservedUnits[shard * SLOTS] = v; }
```

```java
// (b) One padded object per shard, via the inheritance idiom.
//     Prefer this when the shard has several fields, not just one counter.
private final PaddedShardState[] shards = new PaddedShardState[SHARD_COUNT];
```

**(a) is cheaper and simpler for a single value.** Note that `long[]` elements are
guaranteed contiguous — the striding arithmetic is sound in a way it would not be for an
array of objects, where the elements are *references* and the objects themselves may be
anywhere.

**(b) is what you want the moment a shard has more than one hot field**, because those
fields also need to be in the shard's own line and not in a neighbour's.

**And the mandatory step before either:** measure. The scalability curve at
`@Threads({1,2,4,8,16,32,64})` before the change, and the same curve after. If the curves
are the same shape, **revert the padding** — you have spent memory and readability on
nothing, and the wall is somewhere else. That is not a formality. It is Trap 1, and it is
the most common outcome of the first time anyone tries this.

### Fix 3 — the fix that beats both: do not share the line at all

The strongest version of this fix removes the shared object rather than padding it:

```java
// Per-thread accumulation, merged only on read. No shared writes at all.
private final ThreadLocal<long[]> localCounts = ThreadLocal.withInitial(() -> new long[4]);
```

Zero coherence traffic, because no two cores write the same line — each thread's array is
written only by that thread. It is also the shape with the worst hazards: `ThreadLocal` on
a pooled thread is a leak if never removed (Topic 79), the merge-on-read needs a registry
of live thread-local arrays, and you have rebuilt `LongAdder` badly.

**Which is the point.** `LongAdder` *is* this design, done properly, by the JDK. Reaching
for `ThreadLocal` here is a signal you should be reaching for `LongAdder` instead. The
principle generalises: **the best fix for false sharing is usually to stop sharing, and
the second best is to use a JDK class whose author already solved it.** Padding your own
fields is third.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — padding on a hunch, without measuring

**Wrong approach.** Throughput plateaus at eight threads. Someone recalls a talk about
false sharing. Padding gets added to three classes in the hot path. The PR says "fixes
scalability wall".

**Exact symptom.** One of three, and you cannot tell which without a measurement:

1. **No change at all.** The wall is exactly where it was. The padding is now permanent,
   unexplained, and will be copied to the next class by someone who assumes it was needed.
2. **A small improvement**, indistinguishable from noise, which gets reported as a win
   because nobody computed an error bar.
3. **It gets worse.** The objects are now 8× bigger. Fewer fit in L2. The working set no
   longer fits in cache, so you have traded a coherence problem for a **capacity**
   problem. Allocation rate rises, Eden fills faster, GC frequency rises (Topic 68).

**Root cause.** False sharing is one of at least five explanations for a scalability wall
and, in an application service, rarely the right one. The others:

| Candidate | The cheap check that rules it in or out |
|---|---|
| Lock contention | JFR `jdk.JavaMonitorEnter` / `jdk.ThreadPark` events; `BLOCKED` threads in a dump |
| Contended CAS on one variable (Topic 95) | is there a single hot `AtomicLong`/`AtomicReference`? that is **true** sharing |
| GC | `-Xlog:gc*` — is GC time growing with thread count? |
| A saturated downstream (DB, pool, provider) | HikariCP pool metrics (Topic 109); downstream p99 |
| **False sharing** | **only after the above four are eliminated**, and then only by the JMH curve |

**Fix.** Never pad without: (a) a scalability curve across `@Threads({1,2,4,8,16,32,64})`
showing the wall, (b) elimination of the four cheaper candidates, and (c) the same curve
after padding, showing the wall move. If (c) shows nothing, **revert**. A reverted
optimisation is a successful experiment; a kept one that did nothing is technical debt
with a plausible-sounding comment on it.

### Trap 2 — `@Contended` silently doing nothing

**Wrong approach.**

```java
import jdk.internal.vm.annotation.Contended;

public class Stats {
    @Contended volatile long ordersPlaced;
    @Contended volatile long refundsProcessed;
}
```

Compiled with `--add-exports`, deployed, benchmarked. Result: no improvement. Conclusion:
"false sharing wasn't the problem."

**Exact symptom.** The benchmark curve is unchanged. JOL's layout for `Stats` shows the
two `long`s **adjacent, at consecutive offsets, with no gap**. There is no warning, no
error, and no log line anywhere saying the annotation was ignored.

**Root cause.** The JVM honours `@Contended` only on trusted classes (boot classpath /
`java.base`) unless you pass **`-XX:-RestrictContended`**. This is deliberate: the
annotation multiplies object size, and letting arbitrary code inflate every instance by
256 bytes is not a default anyone wants. Your class is not trusted, so the annotation is
metadata with no effect.

**The reason this trap is the most expensive one here:** it produces a *false negative*.
You have not merely failed to fix the problem — you have gathered evidence that the
problem does not exist, and you will not look here again.

**Fix.** Three steps, in order:

```bash
# 1. Compile with the export.
javac --add-exports java.base/jdk.internal.vm.annotation=ALL-UNNAMED Stats.java

# 2. Run with BOTH flags. The second one is the one everybody forgets.
java --add-exports java.base/jdk.internal.vm.annotation=ALL-UNNAMED \
     -XX:-RestrictContended \
     -jar app.jar

# 3. VERIFY with JOL that padding actually appeared. Never skip this.
java --add-exports java.base/jdk.internal.vm.annotation=ALL-UNNAMED \
     -XX:-RestrictContended \
     -cp .:jol-cli.jar org.openjdk.jol.Main internals Stats
```

**WHAT TO LOOK FOR** in step 3: large gaps in the layout, tens of bytes wide, before and
after each annotated field, and an instance size in the hundreds of bytes.

**And the honest recommendation, again:** for application code, use the inheritance-based
manual padding instead. It needs no flags, cannot silently do nothing, survives a JDK
upgrade, and is visible to the next reader. Reserve `@Contended` knowledge for reading JDK
source.

### Trap 3 — padding fields the JVM reorders away

**Wrong approach.**

```java
class Counters {
    volatile long a;
    long p1, p2, p3, p4, p5, p6, p7;   // "padding between a and b"
    volatile long b;
}
```

**Exact symptom.** The object is 64+ bytes bigger, the code has seven mystery fields, and
the benchmark curve is identical. JOL shows `a` and `b` at **adjacent offsets**, with all
seven padding fields sitting after `b`.

**Root cause.** The JVM lays out fields grouped by type and size, not in declaration
order. Nine `long`s are nine `long`s; the layout algorithm has no idea that `p1..p7` were
meant to be a spacer, and it is free to place them anywhere. A future JDK is free to
change the ordering again.

**Fix.** Force the order with the class hierarchy, since **superclass fields are always
laid out before subclass fields**:

```java
abstract class Lhs { long p1,p2,p3,p4,p5,p6,p7,p8,p9,p10,p11,p12,p13,p14,p15; }
abstract class A extends Lhs { volatile long a; }
abstract class Mid extends A { long q1,q2,q3,q4,q5,q6,q7,q8,q9,q10,q11,q12,q13,q14,q15; }
abstract class B extends Mid { volatile long b; }
public final class Counters extends B { long r1,/* ... */ r15; }
```

Then **confirm with JOL**. The entire fix is a claim about offsets; a claim about offsets
that you have not read out of JOL is a guess with extra steps.

### Trap 4 — padding an object that has millions of instances

**Wrong approach.** False sharing is found and fixed on a shard-state object. The same
padding is then applied to `OrderLine`, because it is "in the hot path".

**Exact symptom.** Throughput is unchanged or slightly worse. Then, over the next few
hours: allocation rate up several-fold, Eden filling faster, young-GC frequency up,
promotion rate up, old-gen growing, and eventually longer G1 pauses (Topic 71). With 5M
order lines in `orderflow`, an object grown from 48 bytes to 256 bytes is **an extra
gigabyte of live heap**.

**Root cause.** Padding trades memory for coherence. That trade is excellent for a handful
of long-lived, per-core, write-hot objects, and catastrophic for a numerous, short-lived,
single-threaded-per-instance data object. `OrderLine` is written by exactly one thread and
never contended, so the padding bought nothing and cost everything.

**Fix.** Apply the precondition test before padding anything:

> **Is this specific object written concurrently by more than one core?** And: **how many
> instances exist?**

Padding is for the few-instance, many-writer case. If either half fails, do not pad.
`LongAdder`'s `Cell` array has one entry per contending thread — a handful. That is the
shape.

### Trap 5 — padding 64 bytes on Apple Silicon

**Wrong approach.** Every article and every Stack Overflow answer says 64 bytes, so:

```java
long p1, p2, p3, p4, p5, p6, p7;   // 56 bytes + 8-byte field = 64. Done.
```

**Exact symptom.** The most confusing one in this document, because it is
**machine-dependent**. On the x86-64 Linux CI runner the fix works and the curve improves.
On the developer's M-series Mac the curve is unchanged. Two engineers get opposite results
from the same commit and each concludes the other measured wrong.

Or, worse, the reverse: it is developed and verified on the Mac with 128-byte padding,
someone "optimises" the memory back down to 64 for the x86 production fleet, and the fix
silently regresses for anyone still on a Mac.

**Root cause.** `sysctl hw.cachelinesize` reports **128** on Apple Silicon and **64** on
x86-64. 64 bytes of padding on a 128-byte-line machine leaves both variables in the same
line. **The fix is exactly half of a fix, which is no fix.**

**Fix.** Derive the padding from the platform rather than hard-coding it, and document
both numbers:

```java
// Confirmed with `sysctl hw.cachelinesize` (128 on Apple Silicon)
// and `getconf LEVEL1_DCACHE_LINESIZE` (64 on x86-64).
// We pad for the LARGER of the two: over-padding costs bytes, under-padding costs
// the entire fix. -XX:ContendedPaddingWidth defaults to 128 for the same reason.
private static final int CACHE_LINE = 128;
```

**Always pad for the largest line size any of your targets uses**, and put the two
commands in a comment so the next reader can check rather than guess. And run the
scalability curve on **both** architectures if you deploy to both — this is one of very
few optimisations that is genuinely not portable.

---

## Hands-on proof

No JVM ran here. These are the commands; the outputs are yours.

### Setup

```bash
sysctl hw.cachelinesize          # macOS: expect 128 on Apple Silicon, 64 on Intel
sysctl -n machdep.cpu.brand_string hw.physicalcpu hw.logicalcpu
java -version
```

**WHAT TO LOOK FOR:** write down the cache line size. Every padding number in the rest of
this document is derived from it. On Apple Silicon also note that the P-cores and E-cores
differ in cache sizes, which is one more reason your scalability curve will not look like
one from an x86 server.

Get JOL, which is the primary instrument here:

```bash
# from Maven Central; or add org.openjdk.jol:jol-core as a test dependency
curl -sLO https://repo1.maven.org/maven2/org/openjdk/jol/jol-cli/0.17/jol-cli-0.17.jar
```

### Proof 1 — read your own object layout, do not calculate it

```java
// Layout.java
public class Layout {
    public static class Naive {
        volatile long a;
        long p1,p2,p3,p4,p5,p6,p7;
        volatile long b;
    }
    abstract static class Lhs { long l1,l2,l3,l4,l5,l6,l7,l8,l9,l10,l11,l12,l13,l14,l15; }
    abstract static class A extends Lhs { volatile long a; }
    abstract static class Mid extends A { long m1,m2,m3,m4,m5,m6,m7,m8,m9,m10,m11,m12,m13,m14,m15; }
    abstract static class B extends Mid { volatile long b; }
    public static class Padded extends B { long r1,r2,r3,r4,r5,r6,r7,r8,r9,r10,r11,r12,r13,r14,r15; }
}
```

```bash
javac Layout.java
java -cp .:jol-cli-0.17.jar org.openjdk.jol.Main internals 'Layout$Naive'
java -cp .:jol-cli-0.17.jar org.openjdk.jol.Main internals 'Layout$Padded'

# and again with the JDK 25 flag, to see the arithmetic move:
java -XX:+UseCompactObjectHeaders -cp .:jol-cli-0.17.jar org.openjdk.jol.Main internals 'Layout$Naive'
```

**WHAT TO LOOK FOR:** the `OFFSET` column for fields `a` and `b`, and the reported
`Instance size`.

| What you see | What it means |
|---|---|
| In `Naive`: `a` and `b` at offsets differing by 8 | **The seven padding fields did nothing.** Trap 3, confirmed on your own JVM. Look at where `p1..p7` actually landed. |
| In `Padded`: `a` and `b` at offsets differing by ≥ your cache line size | The inheritance idiom worked. This is the layout you can rely on. |
| Instance size of `Padded` in the hundreds of bytes | The cost of the fix, stated in bytes. Multiply by instance count before you ship it. |
| Header size 12 vs 8 bytes with/without `UseCompactObjectHeaders` | `[JAVA 25]`'s effect on your arithmetic, measured rather than assumed. |
| Field ordering differs from your declaration order | Expected. This is the whole reason Trap 3 exists. |

**Do this before writing any padding.** It converts the topic from folklore into
arithmetic you can check.

### Proof 2 — the scalability curve, which is the only real evidence

```java
@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.SECONDS)
@State(Scope.Benchmark)
@Warmup(iterations = 5, time = 1)
@Measurement(iterations = 10, time = 1)
@Fork(3)
@Threads({1, 2, 4, 8, 16, 32, 64})
public class FalseSharingBench {

    @Param({"NAIVE", "PADDED"})
    public String layout;

    Object[] slots;                    // one slot per thread, either naive or padded

    @Setup(Level.Trial)
    public void setup() { /* allocate SLOTS according to layout and thread count */ }

    @State(Scope.Thread)
    public static class Idx { int i; @Setup public void s() { i = nextIndex(); } }

    @Benchmark
    public void increment(Idx idx) {
        // each thread increments ONLY its own slot; no logical sharing at all
        bump(slots[idx.i]);
    }
}
```

```bash
mvn -q clean package
java -jar target/benchmarks.jar FalseSharingBench -rf json -rff /tmp/fs.json
```

**WHAT TO LOOK FOR** — and read this as a *shape*, not as numbers:

| What the curve does | What it means |
|---|---|
| `NAIVE` rises to ~2 threads then flattens or **declines**; `PADDED` keeps rising | **False sharing, demonstrated.** The declining segment is the ping-pong: more cores, more invalidation traffic, less work. |
| Both curves flatten at the same thread count | Not false sharing. Something else is the wall — go back to Trap 1's table. |
| Both curves identical at every thread count | Either the sharing is not happening (check JOL — did the padding apply?) or the increment is not the bottleneck. |
| `PADDED` *worse* than `NAIVE` at 1 thread | Expected and fine. Bigger objects, worse cache density. It is the multi-thread end of the curve that matters. |
| Error bars overlapping between variants | **You have no result.** Raise `@Fork`, or accept that they are equivalent here. |
| `NAIVE` flat from 1 thread onward | Suspect the JIT deleted the work (Topic 75) — add a `Blackhole` and re-check. |

**The single most important thing about this proof:** the result is the *difference in
slope*, across thread counts, between two variants measured in the same run. It is not a
number. Anyone quoting a single "false sharing costs 4×" figure without a curve and a
thread count has not measured this.

On Apple Silicon the P/E-core split introduces variance that looks like a plateau; when in
doubt, run the same benchmark in a Linux container and compare shapes.

### Proof 3 — confirm `LongAdder`'s `Cell` is `@Contended`

```bash
unzip -p "$JAVA_HOME/lib/src.zip" \
  java.base/java/util/concurrent/atomic/Striped64.java | grep -n -B 4 "static final class Cell"

unzip -p "$JAVA_HOME/lib/src.zip" \
  java.base/java/util/concurrent/ConcurrentHashMap.java | grep -n -B 3 "static final class CounterCell"
```

**WHAT TO LOOK FOR:** the `@jdk.internal.vm.annotation.Contended` annotation immediately
above each class declaration.

| What you see | What it means |
|---|---|
| `@jdk.internal.vm.annotation.Contended static final class Cell` | Confirmed. Topic 95's `LongAdder` scaling depends on this document's mechanism. |
| The same on `CounterCell` in `ConcurrentHashMap` | Confirmed. Topic 92's `size()` striping depends on it too. |
| No `src.zip` in your JDK | Read it on the OpenJDK GitHub mirror for your exact tag instead. Do not take it on trust. |

### Proof 4 — watch `@Contended` do nothing, then do something

```bash
javac --add-exports java.base/jdk.internal.vm.annotation=ALL-UNNAMED Stats.java

# (a) WITHOUT the flag — the annotation is inert:
java --add-exports java.base/jdk.internal.vm.annotation=ALL-UNNAMED \
     -cp .:jol-cli-0.17.jar org.openjdk.jol.Main internals Stats

# (b) WITH the flag:
java --add-exports java.base/jdk.internal.vm.annotation=ALL-UNNAMED \
     -XX:-RestrictContended \
     -cp .:jol-cli-0.17.jar org.openjdk.jol.Main internals Stats

# and to see the padding width itself:
java -XX:+PrintFlagsFinal -version | grep -i contended
```

| What you see | What it means |
|---|---|
| (a) fields adjacent, small instance size; (b) large gaps, large instance size | **Trap 2, demonstrated.** The difference between the two runs is one flag and it is the difference between a fix and a no-op. |
| Both identical | Your JDK differs from my description. Believe JOL, and check `PrintFlagsFinal` for `RestrictContended`. |
| `ContendedPaddingWidth = 128` | The JDK's own answer to "how much padding is enough" — two 64-byte lines, or one Apple Silicon line. Use the same number. |

### Proof 5 — hardware counters, and the honest note about macOS

On Linux, the direct evidence for coherence traffic is a hardware counter:

```bash
# Linux only:
perf stat -e cache-misses,cache-references,LLC-load-misses,LLC-store-misses -- java -jar bench.jar
perf c2c record -- java -jar bench.jar && perf c2c report     # purpose-built for false sharing
java -jar target/benchmarks.jar FalseSharingBench -prof perfnorm
```

`perf c2c` ("cache to cache") exists specifically to find false sharing: it reports HITM
events — a load that hit a line **modified in another core's cache** — and attributes them
to source lines and offsets. It is the definitive tool.

**On macOS, none of this is available, and I am not going to pretend otherwise.** `perf` is
a Linux kernel subsystem; it does not exist on Darwin. Apple Silicon does not expose these
counters to userspace in any equivalent form. There is no `perf c2c` for macOS, JMH's
`-prof perfnorm` and `-prof perfasm` **will not work**, and any command claiming to do this
on your Mac would be fabricated.

Your three honest options:

| Option | What it gives you | What it costs |
|---|---|---|
| **The JMH scalability curve** (Proof 2) | The shape of the wall, and whether padding moves it. **Sufficient to decide.** | You infer the mechanism rather than observing it. |
| Run the benchmark in a **Linux container** (`--cap-add=PERFMON`, or a Linux CI runner) with `perf c2c` | Direct HITM evidence attributed to a field offset | Different hardware from production if production is ARM; container overhead |
| `xcrun xctrace record --template 'Time Profiler'` | Where CPU time goes on macOS | **Not cache misses.** It will show time inside your increment, which you already knew. |

**Do not let the missing tool stop you.** The curve in Proof 2 is a complete decision
procedure: if padding moves the wall and JOL confirms the padding applied, you have your
answer. `perf c2c` tells you *why* with more authority; it does not tell you anything
different.

---

## Failure drill

**The drill:** make `orderflow`'s per-endpoint counters false-share on purpose, prove the
scalability wall with a curve, fix it, prove the fix — **and then run the negative control
that most engineers skip**, which is the half of this drill that makes you good at it.

### Part A — build the wall

Replace `EndpointCounters` with an array-backed version, deliberately unstrided:

```java
@Component
public class EndpointCounters {
    // 8 longs = 64 bytes = one x86 line, half an Apple Silicon line.
    private final AtomicLongArray counts = new AtomicLongArray(8);
    public void record(int endpointOrdinal) { counts.incrementAndGet(endpointOrdinal); }
    public long get(int endpointOrdinal)    { return counts.get(endpointOrdinal); }
}
```

Wire `record(...)` into a Spring interceptor so every request increments exactly one slot,
chosen by endpoint. **No two endpoints share a slot.** There is zero logical sharing, and
that is the point of the drill.

### Part B — measure the wall, with a curve

Do **not** measure this by running k6 once and looking at throughput. Extract the counter
into a JMH benchmark, because you need the thread-count axis:

```bash
java -jar target/benchmarks.jar CounterBench -p layout=NAIVE -rf json -rff /tmp/naive.json
```

with `@Threads({1,2,4,8,16,32,64})`.

**WHAT TO CAPTURE:** throughput at each thread count, with error bars. Plot it. **The plot
is the artefact** — it is what you will show someone, and it is what a single number can
never be.

| What the curve shows | What to conclude |
|---|---|
| rises then flattens around the core count | normal saturation, not necessarily false sharing |
| rises then **declines** as threads increase | strongly suggestive: more parallelism producing less work is the coherence signature |
| flat from one thread | the JIT ate your benchmark; add a `Blackhole` |

### Part C — write down your diagnosis *before* fixing anything

In your own words, in one paragraph:

1. Which of Trap 1's five candidates have you eliminated, and with which command?
2. What is the cache line size on this machine, and how many counters fit in one?
3. What do you predict the padded curve will look like — specifically, at which thread
   count do you expect the two curves to diverge?

**Committing to a prediction before you measure is the entire discipline.** Without it you
will find whatever you were hoping for.

### Part D — fix it, two ways, and compare

```java
// Fix 1 — stride the array. LINE from sysctl; 16 longs per 128-byte line.
private static final int STRIDE = 128 / Long.BYTES;
private final AtomicLongArray counts = new AtomicLongArray(8 * STRIDE);
public void record(int i) { counts.incrementAndGet(i * STRIDE); }
```

```java
// Fix 2 — LongAdder per endpoint. Fixes true sharing AND false sharing.
private final LongAdder[] counts =
        IntStream.range(0, 8).mapToObj(i -> new LongAdder()).toArray(LongAdder[]::new);
public void record(int i) { counts[i].increment(); }
```

Re-run the curve for each. Then answer:

- Did Fix 1 move the wall? By how much, and at which thread count did the curves diverge?
- Did Fix 2 do better than Fix 1? **It should**, because Fix 1 only addresses false
  sharing while Fix 2 also removes the single-variable CAS contention (Topic 95). If Fix 2
  is *not* better, that tells you the true-sharing component was negligible — which is
  itself a finding worth writing down.
- What did each cost in memory? Compute it from JOL, not from arithmetic.

### Part E — the negative control (do not skip this)

Now pad something that is **not** false-sharing. Take `OrderLine` — 5M instances,
single-threaded per instance — and apply the same padding. Re-run the Topic 65 k6 baseline.

**WHAT TO LOOK FOR:**

```bash
# heap and GC before and after:
jcmd <pid> GC.heap_info
grep -c "Pause Young" /tmp/gc.log
curl -s localhost:8080/actuator/metrics/jvm.gc.pause
```

| Observation | What it means |
|---|---|
| throughput unchanged or slightly worse | Padding did not help, because there was no contention to relieve |
| live heap up substantially, young-GC frequency up | **The cost of the trade, with nothing bought.** Trap 4, felt rather than read. |
| p99 worse | GC pauses now dominate what padding was supposed to improve |

**Then revert it.** The lesson is not "padding is bad". It is: **padding is a targeted
trade that requires the precondition to hold, and you now have first-hand evidence of
what it costs when the precondition does not.**

### What the drill proves

- Logical independence does not imply physical independence. Eight counters, eight
  endpoints, eight threads — one cache line.
- The symptom is a *curve*, not a number, and it is invisible at one thread.
- The fix must be verified twice: **JOL** (did the layout change?) and **the curve** (did
  it matter?). Either alone is insufficient.
- The same fix applied without the precondition is a pure cost. You measured that too.

---

## Measurement

### The instrument for each claim

| Claim you want to make | The right instrument | The wrong instrument |
|---|---|---|
| "these two fields are in the same cache line" | **JOL** `internals` — read the offsets | arithmetic from a blog post's header size |
| "my padding actually applied" | **JOL** again, after the change | the fact that you wrote the fields |
| "this is a scalability wall" | JMH `@Threads({1,2,4,8,16,32,64})`, throughput vs threads | a single-threaded benchmark |
| "padding fixed it" | the same curve, both variants, one run, with error bars | a before/after number from two separate runs |
| "it is coherence traffic specifically" | `perf c2c` / `-prof perfnorm` — **Linux only** | anything on macOS; there is no equivalent |
| "it is not lock contention instead" | JFR `jdk.JavaMonitorEnter` and `jdk.ThreadPark` | absence of `synchronized` in the file you read |
| "the padding cost is acceptable" | JOL instance size × instance count, plus `-Xlog:gc*` | intuition |
| "throughput changed in production" | Topic 65's k6 baseline, re-run | the JMH number, which is a microbenchmark |

### JOL — the commands you will actually type

```bash
# layout of one class, with your real deployment flags:
java <your-prod-flags> -cp app.jar:jol-cli-0.17.jar \
     org.openjdk.jol.Main internals com.orderflow.metrics.EndpointCounters

# estimate layouts across VM modes without switching JVMs:
java -cp app.jar:jol-cli-0.17.jar org.openjdk.jol.Main estimates com.orderflow.Order

# what does this object graph actually retain?
java -cp app.jar:jol-cli-0.17.jar org.openjdk.jol.Main footprint com.orderflow.Order
```

**WHAT TO LOOK FOR:** the `OFFSET`/`SIZE`/`TYPE`/`DESCRIPTION` columns, the alignment-gap
rows, and `Instance size`. Two fields whose offsets differ by less than your cache line
size **can** false-share; whether they **do** depends on whether two cores write them
concurrently.

Run JOL with your **production** flags. Compressed oops, `UseCompactObjectHeaders` and heap
size all change the answer, and a layout from a default `java` invocation may not be the
layout you deploy.

### JMH — the harness shape that matters

The defining features, restated because getting any of them wrong invalidates the run:

```java
@BenchmarkMode(Mode.Throughput)          // throughput, not average time: you want the slope
@Threads({1, 2, 4, 8, 16, 32, 64})       // THE axis. Without it there is no experiment.
@Fork(3)                                 // separate JVMs: JIT state varies between runs
@Warmup(iterations = 5, time = 1)        // C2 must have compiled the hot loop
@Measurement(iterations = 10, time = 1)
@State(Scope.Benchmark)                  // shared object: that is the point
```

- `@Param` both variants so they run in **one** invocation. Comparing two separate runs
  compares two JIT histories and two thermal states as much as two layouts.
- Use a `Blackhole` or return the value, or C2 may delete the work (Topic 75).
- Report the **JSON** (`-rf json`) and plot it. The deliverable is a chart.
- On Apple Silicon, thread counts above the P-core count schedule onto E-cores and the
  curve bends for a reason that has nothing to do with your code. Note the P/E split from
  `sysctl hw.perflevel0.physicalcpu` when you interpret the tail of the curve.

### The standing rule: a naive `System.nanoTime()` loop is wrong

Restated once more, because this topic tempts it more than any other:

```java
// WRONG. Produces a number. The number is false.
long t0 = System.nanoTime();
for (int i = 0; i < 100_000_000; i++) counters.a++;
System.out.println(System.nanoTime() - t0);
```

No warm-up; dead-code elimination; **single-threaded, so the phenomenon under study cannot
occur**; no variance. Topic 77. Use JMH.

### JFR — for ruling the alternatives out

JFR does **not** have a false-sharing event. Nothing in it will say "false sharing". What
it does brilliantly is eliminate the cheaper explanations, which is Trap 1's table:

```bash
java -XX:StartFlightRecording=duration=120s,filename=/tmp/fs.jfr,settings=profile -jar orderflow.jar

jfr summary /tmp/fs.jfr
jfr print --events jdk.JavaMonitorEnter /tmp/fs.jfr | head -40    # lock contention?
jfr print --events jdk.ThreadPark       /tmp/fs.jfr | head -40    # j.u.c blocking?
jfr print --events jdk.GCPhasePause     /tmp/fs.jfr | head -40    # GC?
jfr print --events jdk.ExecutionSample  /tmp/fs.jfr | head -40    # where is CPU going?
```

| What you see | What it means |
|---|---|
| Substantial `JavaMonitorEnter` durations | Lock contention. **Fix that first**; it is bigger and easier. |
| Many `ThreadPark` events in `j.u.c` frames | Blocking on a lock or queue. Not this topic. |
| Negligible monitor/park/GC events, high CPU, flat throughput | The residue in which false sharing is a live candidate. **Now** go to the JMH curve. |
| `ExecutionSample` concentrated in your increment method | Consistent with false sharing, and also with simply doing a lot of increments. Not evidence on its own. |

### Micrometer — what to watch in production (forward-ref Topic 118)

There is no metric for false sharing. What you can watch is the *shape* that makes you
suspect it:

- **throughput per pod against vCPU count**, as a dashboard panel. A service whose
  throughput does not rise when you give it cores is telling you something, and this topic
  is one of the four candidate explanations.
- **CPU utilisation against throughput.** Rising CPU with flat throughput is the signature.
- Standard RED metrics per endpoint, so you can see whether the plateau is uniform (a
  shared resource, like a cache line) or per-endpoint (a downstream).

And the counters themselves: **use `LongAdder` or Micrometer's own `Counter`**, which is
`LongAdder`-backed under the hood, and this entire class of problem never arises in your
metrics layer.

---

## Practice exercises

### 1 — Easy: read your own layouts and build the reference

Using JOL, produce a small table for your machine covering: an empty object; an object
with one `long`; the `Naive` class from Proof 1; the `Padded` class; and each of those
again under `-XX:+UseCompactObjectHeaders`. Record header size, field offsets and instance
size.

Then answer in one line each: how many `long` fields fit in one cache line on this machine
alongside a header? How much padding does one `long` need to sit alone in a line? By how
much did compact headers change that?

**Why this is worth it:** you now have the three numbers from "Machine-level reality"
measured rather than assumed, for your actual machine, and you will never again copy a
padding constant from an article.

### 2 — Medium: the counter shoot-out (combines 01, 68, 69, 87, 92, 95)

Build four implementations of `orderflow`'s per-endpoint request counter:

1. `AtomicLong[]` — unstrided.
2. `AtomicLongArray` — unstrided.
3. `AtomicLongArray` — strided by cache line.
4. `LongAdder[]`.

Requirements:

- JMH, `@Threads({1,2,4,8,16,32,64})`, `@Fork(3)`, all four as `@Param` in one run.
- JOL output for each, showing where the counters actually sit.
- A read benchmark too: `sum()`/`get()` at the 15-second scrape frequency, because Fix 1's
  argument rests on the write/read ratio and you should verify the read cost you are
  accepting.
- `-prof gc` on all four, since (1) allocates four objects and (4) allocates cells lazily.
- A written recommendation naming which you would ship for `orderflow` **and the load at
  which your answer would change**.

**The trap deliberately built into this exercise:** (1) and (2) may perform differently
even though both are unstrided, because `AtomicLong[]` is an array of *references* to
separately allocated objects whose heap positions you do not control, while
`AtomicLongArray` is one contiguous `long[]` whose positions you do. Explain what you
observe. Either result teaches you something.

### 3 — Hard: production simulation on the `orderflow` baseline

1. Run the Failure drill's Parts A–E end to end against the running service under the
   Topic 65 k6 profile, at 2, 4 and 8 vCPU (`docker run --cpus=...`). Produce a
   throughput-versus-vCPU chart for the naive and fixed versions. **The chart is the
   deliverable.**
2. Extend to the shard case: shard the in-memory inventory reservation cache by SKU hash
   with one mutable state object per shard, deliberately unpadded. Measure. Pad. Measure
   again. State what you would ship and why, including the memory cost in bytes.
3. Write a one-page decision note in the shape Topic 131 will ask for: the symptom, the
   four candidate causes, the command that eliminated each, the experiment that
   discriminated the survivor, the fix, the cost in memory, and **the condition under which
   you would revert it**.
4. Run the whole thing once more in a Linux container with `perf c2c` and compare its
   verdict with your curve's. If they disagree, the container's hardware differs from your
   Mac's — say how, and which one resembles production.

**The deliverable is the decision note.** Part 4 is the check that your reasoning from the
curve alone was sound — and if you cannot run it, the note must say so, which is itself the
professional behaviour this document is teaching.

---

## Interview questions

### Q1 — "Two threads increment two different counters and it doesn't scale. Why?"

**MID-LEVEL ANSWER.** "If they're different variables there shouldn't be contention. Maybe
there's a lock somewhere, or the JIT isn't optimising it. I'd check whether the counters
are `volatile`, since `volatile` writes are slower."

**SENIOR ANSWER.** "Most likely false sharing, but I'd rule out the cheaper explanations
first because this is over-diagnosed.

The mechanism: cache coherence works on lines, not variables — 64 bytes on x86-64, 128 on
Apple Silicon. To write any byte of a line a core must hold it exclusively, which
invalidates every other core's copy. Two adjacent `long` fields are in the same line, so
each core's write kicks the line away from the other. Logically independent, physically
contended, and it gets *worse* with more cores, which is the giveaway.

Before believing that I'd check four things: monitor contention in JFR, whether it's
actually one hot variable rather than two — that's true sharing and a different fix — GC
time growing with thread count, and downstream saturation. If all four are clean, I'd
build a JMH harness at `@Threads({1,2,4,8,16,32,64})` with padded and unpadded variants as
`@Param`s in one run. **The evidence is the divergence in the two curves, not a single
number.**

The fix depends on the shape. For counters, `LongAdder` — its `Cell` is `@Contended`, so
it solves both the true and the false sharing, and I get it for free. For a per-shard
struct I'd pad via the superclass idiom and confirm with JOL. I wouldn't use `@Contended`
directly in application code: it needs `--add-exports` and `-XX:-RestrictContended`, and
without the second flag it silently does nothing, which produces a false negative."

**What separates them.** The mid answer has no mechanism and reaches for `volatile`, which
is a correctness construct, as a performance explanation. The senior answer names the
hardware mechanism, names the platform-specific line size, **volunteers that the diagnosis
is over-applied and gives the elimination order**, insists the evidence is a curve, and
knows the annotation's silent-failure mode. The scepticism is the senior part.

**Follow-up:** *"How would you prove it on a Mac?"* — I can't get cache counters there;
`perf` and `perf c2c` are Linux-only and Apple Silicon doesn't expose the equivalent. So
the JMH curve is my evidence, plus JOL confirming the layout changed. If I needed hardware
proof I'd run the same benchmark in a Linux container with `perf c2c` and note that the
hardware differs from my laptop.

### Q2 — "Why is `LongAdder` faster than `AtomicLong` under contention?"

**MID-LEVEL ANSWER.** "`LongAdder` splits the value across multiple cells so threads don't
all hit the same variable. You call `sum()` to read it. `AtomicLong` uses CAS which retries
under contention."

**SENIOR ANSWER.** "Two mechanisms, and the second is the one people miss.

First: striping. `AtomicLong.incrementAndGet` is a `lock xadd` — or a CAS retry loop — on
one memory location, so N threads serialise on it and throughput *falls* as N rises.
`LongAdder` gives each contending thread its own `Cell`, chosen by a per-thread probe, so
the threads mostly stop colliding.

Second, and this is the part that makes the first one work: **`Cell` is annotated
`@jdk.internal.vm.annotation.Contended`.** A `Cell` is a header plus one `long` — about 24
bytes. Without padding, five or six cells fit in a 128-byte line, so the threads would
stop contending on a *variable* and start contending on a *line*, and the striping would
buy almost nothing. The annotation pads each cell into its own line. `ConcurrentHashMap`'s
`CounterCell` does the same thing for `size()`.

The cost is on the read side: `sum()` walks the cell array, so it is O(cells) rather than
O(1), and it is not atomic — a sum taken during concurrent updates is a value that was
never simultaneously true. For a counter written thousands of times a second and read once
per metrics scrape, that is exactly the right trade. For something read on every request,
it is not, and I'd stay with `AtomicLong`."

**What separates them.** The mid answer knows the striping. The senior answer knows that
the striping **only works because of the padding**, connects it to `ConcurrentHashMap`, and
states the cost that decides when *not* to use it. That "why does the obvious mechanism
actually work" layer is the ELITE marker.

### Q3 — "How much padding, and how do you know?"

**MID-LEVEL ANSWER.** "64 bytes, so seven `long` fields after the value. That's the cache
line size."

**SENIOR ANSWER.** "It's derived from three numbers, and I read all three rather than
assuming them.

The line size: `sysctl hw.cachelinesize` on macOS, `getconf LEVEL1_DCACHE_LINESIZE` on
Linux. **128 on Apple Silicon, 64 on x86-64.** If I'd padded 64 on my Mac I'd have padded
half a line and fixed nothing — and it would have worked on the x86 CI runner, which is the
worst kind of bug.

The header size: JOL, on my deployment flags. 12 bytes with compressed oops, 16 without,
**8 with `[JAVA 25]`'s `UseCompactObjectHeaders`** — which shifts every offset and
invalidates every pre-JDK-24 padding article.

And the field ordering, which is the one that catches people: the JVM lays fields out by
size, not declaration order, so `long a; long p1..p7; long b;` gives you nine longs it can
arrange however it likes, and JOL will show `a` and `b` adjacent with the padding after
them. I force the order with a superclass chain — superclass fields are laid out first —
and then confirm with JOL, because the whole fix is a claim about offsets.

I pad both sides, and I pad for the largest line size across my deployment targets.
That's why `-XX:ContendedPaddingWidth` defaults to 128 rather than 64: it defends against
the adjacent-line prefetcher and against heap neighbours you can't see."

**What separates them.** The mid answer has one number, no verification, and would be
wrong on the interviewer's Mac. The senior answer treats padding as **three measured
inputs plus a layout guarantee**, knows that naive padding is reordered away, and knows why
the JDK's own default is double the line size.

### Q4 — "When would you *not* fix false sharing?"

**MID-LEVEL ANSWER.** "If it's not causing a problem, I suppose. Or if the code gets too
ugly."

**SENIOR ANSWER.** "Four situations, and three of them are common.

**When the precondition doesn't hold.** False sharing needs concurrent writes from
different cores to the same line. Adjacent fields on an object owned by one thread — an
`Order`, an `OrderLine` — cannot false-share however adjacent they are. That's most objects.

**When there are too many instances.** Padding trades memory for coherence. On a handful of
per-core structures that's free. On `orderflow`'s 5M order lines, padding a 48-byte object
to 256 bytes adds about a gigabyte of live heap, which buys longer GC pauses and a worse
p99 — I'd have traded a coherence problem I didn't have for a GC problem I now do.

**When it isn't the bottleneck.** If lock contention, GC or a saturated downstream is the
wall, padding changes nothing and I've added a permanent mystery to the code. I want the
JMH curve before and after; if the curves match, I revert.

**When a JDK class already solves it.** For counters, `LongAdder`. For a concurrent map's
size, `ConcurrentHashMap`. Hand-rolled padding is the third-best answer — behind not
sharing at all, and behind using a class whose author could legitimately use `@Contended`.

The general form: padding is a targeted trade with a precondition, not a hardening
measure. Applying it defensively is a cost with no benefit."

**What separates them.** The senior answer has a **precondition test** and a cost model,
and it names the specific `orderflow` object where the trade goes wrong with the heap
figure attached. It also ranks the fixes, putting "don't share" and "use the JDK class"
above the clever one — which is the judgment the ELITE tier is testing.

**Follow-up:** *"So when have you seen it genuinely matter?"* — In per-core or per-thread
structures where the whole design premise is independence: sharded counters, ring-buffer
head and tail cursors, per-worker queue state. Anywhere the code says "these threads never
touch each other's data", the hardware may disagree, and that is exactly the shape where
padding earns its memory.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. Two threads **read** the same cache line continuously and never write it. Throughput
   scales linearly with cores. Explain why, in terms of the MESI states, and then state
   the exact change that would destroy that scaling.

2. `-XX:ContendedPaddingWidth` defaults to 128 even on machines with 64-byte cache lines.
   Give two independent reasons the JDK chose double the line size, and say what you would
   lose by setting it to 64.

3. `LongAdder` fixes both true and false sharing. `@Contended` fixes only false sharing.
   Construct a scenario where `@Contended` on an `AtomicLong` field makes a program
   measurably faster, and a second where it makes no difference at all — and name the
   property that distinguishes them.

4. The JVM lays out fields by size, not declaration order, which defeats naive padding.
   Argue that this ordering policy is correct. Then say what the language would need to add
   for padding to be expressible without the superclass trick.

5. You pad an object and the benchmark gets *worse* at one thread but *better* at sixteen.
   Explain both halves with one mechanism, and say which half decides whether you ship it.

6. Topic 11 said `ArrayList` beats `LinkedList` because of cache locality. This topic says
   locality causes false sharing. Reconcile them precisely: what property of the access
   pattern flips locality from an advantage to a cost?

7. `SharedArrayBuffer` + `Atomics` in Node can genuinely false-share. Given that, why does
   this document still open with `NO TYPESCRIPT ANALOGUE`, and what would have to be true
   of your Node background for that header to be wrong?

---

## Quick reference card

### The mechanism in four lines

```
Coherence unit  = the cache line (64 B x86-64 / 128 B Apple Silicon), never the variable
To write a line = hold it Modified = invalidate every other core's copy (an RFO)
False sharing   = two INDEPENDENT variables in one line, written by two cores
Read-only lines never conflict: many cores may hold one line Shared simultaneously
```

### The precondition test — run it before anything else

```
1. Is this line written concurrently by more than one core?     no -> not false sharing
2. Have you eliminated locks / true-sharing CAS / GC / downstream? no -> fix those first
3. Do you have a JMH curve across @Threads showing a wall?      no -> you have no evidence
4. How many instances of this object exist?                     many -> do not pad
```

### The three numbers, and where to read them

| Number | Command | Typical |
|---|---|---|
| cache line size | `sysctl hw.cachelinesize` / `getconf LEVEL1_DCACHE_LINESIZE` | **128** Apple Silicon, **64** x86-64 |
| object header size | JOL `internals` | 12 (compressed oops) · 16 (no COOPs) · **8** `[JAVA 25]` compact |
| actual field offsets | JOL `internals` | **never** what declaration order suggests |

### The fixes, ranked

| Rank | Fix | When |
|---|---|---|
| 1 | **Don't share** — per-thread state, merged on read | whenever the design allows |
| 2 | **Use a JDK class that already pads** — `LongAdder`, `ConcurrentHashMap` | counters, map sizes |
| 3 | **Manual padding via superclass chain** | per-shard/per-core structs; portable, no flags |
| 4 | **Stride an array** (`index * LINE / 8`) | one primitive per logical owner |
| 5 | `@Contended` + `--add-exports` + `-XX:-RestrictContended` | JDK code; rarely worth it in yours |

### Gotchas checklist

- [ ] Line size confirmed from `sysctl`, not from an article. **128 on your Mac.**
- [ ] Header size read from JOL with production flags, not calculated.
- [ ] Padding verified in JOL — the offsets actually moved.
- [ ] Padding surrounds the field on **both** sides.
- [ ] Superclass chain used, so field reordering cannot undo it.
- [ ] `-XX:-RestrictContended` present if `@Contended` is used at all.
- [ ] A JMH curve across `@Threads({1,2,4,8,16,32,64})` exists, before and after.
- [ ] Lock contention, true sharing, GC and downstream eliminated first.
- [ ] Instance count checked — you are not padding a million-instance object.
- [ ] `perf c2c` claims are Linux-only and labelled as such.

---

## When would I use this at work?

**1. The scale-up that did not scale.**

The team moves `orderflow` from 2-vCPU to 8-vCPU nodes and per-pod throughput barely moves.
Everyone concludes the app "isn't parallel enough" and starts planning a rewrite. You spend
an afternoon: JFR rules out locks and GC, the pool metrics rule out the database, and a JMH
curve on the metrics path shows throughput *declining* past four threads. JOL shows four
counters in one line. The fix is one import — `LongAdder` — and the rewrite is cancelled.
**The value is not the fix; it is that you produced a curve instead of an opinion, in an
argument that was otherwise going to be settled by seniority.**

**2. Reviewing a PR that adds padding.**

Someone submits a change padding three classes, citing false sharing. You ask three
questions: *which of these objects is written by more than one core at a time; how many
instances of each exist; and where is the before/after curve?* If the answers are "all of
them", "about five million", and "no curve", the PR gets a clear explanation of Trap 4 and
a request for the measurement. **You have prevented a gigabyte of heap and a permanently
confusing class**, and the author learns the precondition test rather than being told no.

**3. Designing a per-shard structure, before it exists.**

You are partitioning the inventory reservation cache by SKU hash, one shard per core, so
that shards never touch each other. In the design review you say: "logically independent,
but the shard states will be adjacent in memory, so they'll share lines and we'll get the
contention we just designed away." You put a padded shard object and a JOL assertion in the
design **and a benchmark task in the same ticket**. Cost: half a day. Cost of finding it six
months later under Black Friday load: a week and a bad weekend. This is the one case where
padding preemptively is right — because the precondition is *known* to hold from the design
itself, not guessed at afterwards.

---

## Connected topics

**Prerequisites:**

- **69 — object layout, headers and compressed oops.** The direct feeder. Header size,
  field ordering by type, 8-byte alignment and JOL are all from there; this topic is what
  happens when you take that layout seriously across two cores. `[JAVA 25]` compact object
  headers change one input to every padding calculation here.
- **68 — heap generations and TLABs.** Why four separately-allocated `AtomicLong`s end up
  adjacent in memory: they were bump-allocated in the same TLAB. It is also why padding a
  numerous object raises allocation rate and promotion — Trap 4's cost.
- **11 — list implementations and cache reality.** Cache locality's good side. Same
  hardware, opposite sign: locality helps one core reading nearby data, and hurts many
  cores writing it.
- **85 / 87 — monitors and `volatile`.** Why the writes reach memory at all. A `volatile`
  write is a store the JIT may not keep in a register, which is what makes the coherence
  traffic real — and why every demonstration of this effect uses `volatile` even though
  `volatile` is not the cause.
- **95 — CAS, atomics and `LongAdder`.** The closest neighbour. A contended CAS loop is
  cache-line ping-pong on **one** variable; false sharing is the same ping-pong on **two**.
  `LongAdder` fixes both, and it fixes the second one with this document's mechanism.
- **77 — JMH.** Non-negotiable here. This effect is only visible as a curve across thread
  counts, and every naive measurement of it is wrong in at least four ways.
- **75 — escape analysis and dead-code elimination.** Why a benchmark of an increment can
  measure nothing at all, and why a `Blackhole` is mandatory.

**This unlocks:**

- **92 — `ConcurrentHashMap`.** `CounterCell` is `@Contended` for exactly the reason
  `LongAdder`'s `Cell` is. `size()`'s striped counter is a false-sharing fix you have now
  read the source of.
- **94 — explicit locks.** Why a `ReadWriteLock`'s *read* path does not scale: acquiring a
  read lock is an atomic **write** to the shared state word, so readers who exclude nobody
  logically still serialise physically. `StampedLock`'s optimistic read performs no stores
  at all, which is why it scales — and that is this topic's mechanism explaining that one's
  design.
- **93 — blocking queues.** `LinkedBlockingQueue` splits `head` and `last` under separate
  locks so producers and consumers do not contend; whether they contend *physically*
  depends on whether those fields share a line. The same question the Disruptor answers by
  padding its cursors.
- **98 — the concurrency bug taxonomy.** False sharing is the one entry in the performance
  taxonomy that produces **no** distinguishing thread-dump signature: the threads are
  `RUNNABLE` and busy. Knowing that is what stops you from taking a dump and concluding
  nothing is wrong.
- **100 — `ForkJoinPool`.** Per-worker deque state is padded for this reason; work stealing
  is a design whose entire premise is per-worker independence, which the hardware only
  honours if the layout does.
- **101 — virtual threads.** Millions of virtual threads on a handful of carriers changes
  *which* structures are per-core and which are per-task. Padding a per-carrier structure is
  still right; padding anything per-virtual-thread is Trap 4 at a scale you have not seen
  before.
- **129 — capacity and cost models.** The direct business consequence: a capacity model that
  assumes throughput scales with vCPU is wrong for a service with a coherence wall, and this
  is one of the few reasons a model can be wrong in the *optimistic* direction.
- **118 — Micrometer metrics.** Where the counters in this document actually live. Micrometer's
  `Counter` is `LongAdder`-backed, so using it correctly makes this whole problem someone
  else's — which is the outcome you want.

---

*Java baseline 21, running on JDK 25. Three things here are deliberately hedged rather than
asserted: your machine's exact object layout, which only JOL can tell you and which changes
under `[JAVA 25]` compact object headers; the magnitude of any false-sharing effect, which
is a curve on your hardware and never a number I could supply; and whether cache counters
are available to you at all — they are not on macOS, `perf c2c` is Linux-only, and no
command I could write would change that. Everything else — that coherence works on lines of
64 bytes on x86-64 and 128 on Apple Silicon, that writing a line requires invalidating every
other copy, that `LongAdder`'s `Cell` is `@Contended`, and that `@Contended` does nothing in
your code without `-XX:-RestrictContended` — is stable, checkable, and the difference between
finding this bug in an afternoon and rewriting a service that never needed rewriting.*
