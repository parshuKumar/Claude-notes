# 73 — Safepoints and Time-to-Safepoint

## Phase: 8 — JVM Internals
## Category: ELITE
## Java baseline: 21  |  Notes features from: 21 / 25
## Project spine: this is how you explain an `orderflow` p999 spike that **no GC pause in the log accounts for** — the stall that hits every endpoint at the same instant, including ones doing no work, and that survives every collector change you throw at it.

---

## Mechanical statement

Read this three times. Everything below is an elaboration of it.

> **Many JVM operations require every application thread to be stopped
> simultaneously.** Not "mostly stopped". Every one. Garbage collection is the famous
> example, but it is far from the only one: thread dumps, heap dumps, class
> redefinition by an agent, deoptimization, and several kinds of internal bookkeeping
> all need a moment where the entire heap and every thread stack is stable.
>
> **A thread cannot be stopped anywhere.** It can only be stopped at a **safepoint** —
> a point in the generated code where the JVM has a precise map of which registers and
> stack slots hold object references. Stop a thread halfway through computing an
> address and the collector cannot tell a pointer from an integer.
>
> **So the JVM asks, and waits.** It sets a flag; each thread notices the flag at its
> next **safepoint poll** and parks itself. Compiled code polls **at method returns
> and at the back-edges of non-counted loops.**
>
> **A counted `int` loop may have its poll ELIDED.** C2 reasons that a loop with an
> `int` induction variable and a known bound terminates in a bounded number of
> iterations, so it removes the per-iteration poll as an optimisation. If that loop
> runs over five million array elements, the thread does not check the flag for the
> entire duration of the loop.
>
> **Therefore ONE thread can delay EVERY other thread in the JVM for an unbounded
> length of time.** Every other thread has already stopped. They are sitting at their
> safepoints, waiting for the last straggler. Your whole application is frozen, and
> the freeze is caused by a thread that is doing nothing wrong.
>
> **And that delay does not appear in the GC pause figure.** The number the GC log
> prints is time spent doing GC work *at* the safepoint. The time spent *reaching* the
> safepoint is a different number, in a different log, that almost nobody turns on.

Five consequences follow directly, and you should be able to derive each one:

1. A reported 5 ms GC pause can sit inside a 900 ms stop-the-world event.
2. Switching to a low-pause collector (Topic 72) does **not** fix this, and can make it
   more confusing, because the GC log then looks flawless.
3. The stall is **global**: it hits requests on threads that were doing nothing related
   to the cause, across every endpoint, at the same instant.
4. **Non-GC operations pause you too.** A monitoring agent taking a thread dump every
   ten seconds is requesting a global safepoint every ten seconds.
5. You will never find this by staring at a GC log, because the evidence is not in it.
   You need `-Xlog:safepoint*`, and you need to know it exists.

---

## The bridge from what you know

### NO TYPESCRIPT ANALOGUE.

This is the single cleanest "no analogue" in the entire curriculum, and it is worth
being precise about why.

**Node has one JavaScript thread.** When V8 wants to garbage collect, it does not need
to coordinate with anybody. There is nobody to coordinate with. The single thread
reaches a point between two operations, V8 does its work, and execution resumes. There
is no flag to set, no other thread to wait for, no straggler, no unbounded wait.

**The entire concept of "stopping other threads" does not exist in your model.** And
because it does not exist, neither does the failure:

- There is no such thing as a poll point in JavaScript, because there is nobody polling
  for.
- There is no such thing as a thread that delays a collection, because there is one
  thread and it *is* the collection when the collection runs.
- There is no such thing as an operation that stops your code so that a *different*
  subsystem can inspect a *different* thread's stack.

The closest thing you have is the folk knowledge that **a long synchronous function
blocks the event loop.** That intuition is worth keeping for one reason and dangerous
for another.

**Keep this part:** "one piece of code that does not yield can freeze everything else"
is structurally the same complaint. Your instinct that a tight CPU-bound loop is
antisocial is correct and transfers.

**Discard this part, immediately:** in Node, the blocked work is *your own queued
work*, and it is blocked because the loop is busy. In Java, the other threads are
**already stopped and idle**, doing nothing, burning no CPU, waiting on one thread
they have no relationship with. Your service is not busy. It is frozen. And it will
stay frozen even on a 64-core machine with 63 idle cores, because the JVM will not
proceed until the last thread arrives.

That difference matters practically: in Node you can see the blocking function on a CPU
profile, because it is burning CPU. In Java, a CPU profile during a TTSP stall shows
**almost nothing**, because almost every thread is parked. The evidence is somewhere
else entirely.

| You know | Java | Verdict |
|---|---|---|
| One thread, no coordination needed for GC | Global safepoint across all threads | **NO ANALOGUE** |
| A long sync function blocks the event loop | A long counted loop delays every thread | **PARTIAL** — the shape rhymes, the mechanism and the diagnosis are completely different |
| "The process is busy" explains a stall | The process may be **idle** during the stall | **NO ANALOGUE** |
| Profilers show you the blocking work | A CPU profile during a TTSP stall shows almost nothing | **NO ANALOGUE** |
| Thread dumps are free | `jcmd Thread.print` requests a **global safepoint** | **NO ANALOGUE** |

### What DOES transfer

Your latency-budget thinking, again. You already know that a p999 spike is a
*different question* from a p50 regression, and that rare, correlated, cross-endpoint
stalls point to a shared resource rather than to any individual code path. That
instinct is exactly right here, and it is what will make you suspect a global mechanism
instead of chasing the endpoint that happened to be slowest.

---

## What is this?

### The precise definition

A **safepoint** is a point in a thread's execution at which the thread's state is
completely known to the JVM: every object reference held in a register or a stack slot
is identified in a data structure called an **oop map**, generated by the compiler for
that exact program point.

Why the JVM needs this: a garbage collector that moves objects must find and update
every reference to every moved object. References live in thread stacks and registers.
The collector cannot distinguish a reference from a `long` that happens to look like an
address unless the compiler told it, per program point, which slots are references.
Generating oop maps for *every* instruction would be enormous, so the compiler
generates them only at chosen points — the safepoints.

A **global safepoint** (also called a stop-the-world pause, or a VM operation) is when
**all** Java threads are simultaneously at a safepoint, so that the JVM can perform an
operation requiring a stable view of the whole world.

**Time-to-safepoint (TTSP)** is the interval between "the JVM requested a safepoint"
and "the last thread arrived at one". It is a property of your *application code*, not
of the collector.

### The operations that require a global safepoint

This list matters, because GC gets blamed for all of them.

| Operation | Triggered by |
|---|---|
| Garbage collection (young, mixed, full, and the STW phases of concurrent collectors) | Allocation, IHOP, `System.gc()` |
| **Thread dump** | `jcmd <pid> Thread.print`, `jstack`, a monitoring agent, a `kill -3` |
| **Heap dump** | `jcmd GC.heap_dump`, `-XX:+HeapDumpOnOutOfMemoryError`, a profiler |
| **Class redefinition / retransformation** | A `-javaagent` (APM, tracer, mocking library) — Topic 81 |
| Deoptimization of compiled code | A C2 speculation being violated — Topic 74 |
| Making a compiled method not-entrant | Class loading invalidating a CHA assumption — Topic 74 |
| Revoking biased locks | **Historical.** Biased locking was disabled by default in JDK 15 and removed later; on JDK 21 this is no longer a cause. Older war stories about it are still instructive. |
| Enabling or disabling method-level instrumentation | JVMTI agents, JFR settings changes |
| `Thread.getAllStackTraces()`, `GetAllStackTraces` JVMTI | Your own monitoring code, or a library's |
| Inline-cache buffer full | Normal JIT operation; appears as an internal cause in the safepoint log |
| Metaspace / code-cache housekeeping | Class loading and JIT compilation |
| Periodic "cleanup" operations | The JVM's own timers |

**Two things to take from that table.** First, **a thread dump is not free.** A
monitoring agent that dumps threads every ten seconds is requesting a global safepoint
every ten seconds, and if your TTSP is bad, that agent is causing stalls all by itself.
Second, **an APM agent that redefines classes at runtime pauses you**, which is one of
the reasons "the service got slower after we added the agent" is a real and
under-diagnosed phenomenon (Topic 81).

### Where the polls are

This is the part to memorise, because it is the whole diagnosis.

**Interpreted code** checks for a safepoint request very frequently — effectively at
bytecode dispatch. An interpreted thread reaches a safepoint almost immediately. This
is why the problem is a *compiled-code* problem: the JIT removes the checks.

**Compiled code (C1 and C2) polls at:**

| Location | Why |
|---|---|
| **Method return** | Bounded work between calls in the common case; cheap to place |
| **Back-edge of a non-counted loop** | A loop whose trip count the compiler cannot bound could run forever, so it must be interruptible |
| **Method entry (in some cases)** and at call sites | The call itself is a natural boundary |

**Compiled code does NOT poll at:**

| Location | Why |
|---|---|
| **The back-edge of a counted `int` loop** (subject to loop strip mining, below) | C2 reasons the loop terminates in a bounded number of iterations, so the poll is removed as an optimisation. **This is the trap.** |
| Inside an intrinsic such as `System.arraycopy` or `Arrays.fill` | These compile to tight machine loops with no poll |
| Inside a JNI critical section (`GetPrimitiveArrayCritical`) | The native code holds a direct pointer into the heap; the JVM cannot move anything |

**Threads in native code are already "safe".** A thread blocked in a socket read or a
JDBC call has transitioned to a native thread state; the JVM knows it holds no
in-flight references it can disturb, so it does **not** wait for it. This is why a
service that spends most of its time waiting on the database usually has excellent
TTSP — until one thread starts doing CPU-bound array work.

### What a "counted loop" means, exactly

C2 recognises a **counted loop**: an induction variable of type `int`, incremented by a
loop-invariant stride, compared against a loop-invariant bound. Something of this shape:

```java
for (int i = 0; i < lines.length; i++) { ... }
```

That is a counted loop. C2 knows the trip count is bounded by `int` range, applies a
family of optimisations to it — unrolling, vectorisation, range-check elimination — and
historically **removed the safepoint poll from the back-edge**, on the reasoning that
the loop cannot run forever.

The reasoning is sound about *termination* and wrong about *time*. A loop with two
billion iterations terminates. It also runs for a long time, and during it the thread
never checks the flag.

### Loop strip mining — the modern mitigation, and why you must verify it

Modern HotSpot has **loop strip mining**. C2 transforms a counted loop into a nested
pair: an outer loop that carries a safepoint poll, and an inner loop of at most
`-XX:LoopStripMiningIter` iterations that does not. Conceptually:

```java
// what you wrote
for (int i = 0; i < 5_000_000; i++) { total += prices[i]; }

// what C2 may generate, conceptually
for (int outer = 0; outer < 5_000_000; outer += STRIP) {
    int limit = Math.min(outer + STRIP, 5_000_000);
    for (int i = outer; i < limit; i++) { total += prices[i]; }   // no poll inside
    // --- safepoint poll here, on the OUTER back-edge ---
}
```

*Illustration of the transformation, not generated code.*

This bounds TTSP to roughly the time taken by `LoopStripMiningIter` iterations, rather
than the whole loop.

> **Version-dependent behaviour, flagged honestly and importantly.** Loop strip mining
> is controlled by `-XX:+UseCountedLoopSafepoints` and `-XX:LoopStripMiningIter`. My
> understanding is that on a modern JDK with G1, ZGC or Shenandoah, counted-loop
> safepoints are enabled and strip mining is active by default — which means **the
> classic dramatic counted-loop TTSP demonstration may be substantially muted on your
> JDK 21 or 25 runtime.** I am not going to assert the exact defaults for your build.
> **This directly affects the drill below, and the drill's result table explicitly
> covers the "I saw no difference" outcome.** Settle it before you start:
>
> ```bash
> java -XX:+UseG1GC -XX:+PrintFlagsFinal -version | grep -iE "UseCountedLoopSafepoints|LoopStripMiningIter"
> java -XX:+UseZGC -XX:+PrintFlagsFinal -version | grep -iE "UseCountedLoopSafepoints|LoopStripMiningIter"
> jcmd <pid> VM.flags -all | grep -iE "UseCountedLoopSafepoints|LoopStripMiningIter"
> ```
>
> Read what your JVM prints. Then run the drill with the flag explicitly **off** as the
> control, so you can see the mechanism whether or not your default protects you.

### The `int` versus `long` counter question

The classic advice is: change the loop counter from `int` to `long` and the poll comes
back, because a `long` loop was historically not recognised as a counted loop.

That advice was correct for a long time and is now **partially obsolete**. C2 gained
the ability to transform long-counted loops — typically by splitting them into an outer
long loop and inner `int`-counted loops, which then get the same treatment as any other
counted loop, including strip mining.

> **Flagged uncertainty, one line:** I am confident that (a) `int`-counted loops are
> the classic elision case, (b) `long` counters historically kept their polls, and (c)
> modern C2 handles long counted loops in a way that narrows the difference. **I am not
> confident about the exact behaviour on your JDK.** The drill is designed to make you
> measure it rather than believe me, and the result table covers the case where `int`
> and `long` behave identically.

**This matters more than it looks.** A senior engineer who says "change it to `long`
and the poll comes back" is quoting a blog post from 2015. A senior engineer who says
"that was the classic fix; on a modern JDK strip mining may already bound it, so I'd
measure with `-Xlog:safepoint*` before and after" is describing reality.

---

## Why does it matter?

**1. Because it is the single most misattributed latency problem in Java.**

The sequence is always the same. p999 spikes. Someone opens the GC log. The pauses look
fine. GC is "ruled out". The team spends a week on the database, the connection pool,
the network, and the load balancer. The cause was a thread in a counted loop over a
large array, and the evidence was in a log nobody enabled.

**2. Because the symptom is uniquely misleading.**

The stall is **global and correlated**. Every endpoint spikes at the same instant,
including ones that touch no shared code. That pattern normally points at a shared
downstream — a database, a cache, a network. Here the shared resource is *the JVM's own
safepoint protocol*, which is not on anybody's architecture diagram.

**3. Because a CPU profile does not show it.**

During a TTSP stall, almost every thread is **parked**. They burn no CPU. A CPU profile
of the interval shows a nearly empty flame graph and one thread doing array arithmetic
that looks entirely reasonable. Topic 78 covers wall-clock versus CPU profiling; this
is one of the cases where the distinction is the whole answer.

**4. Because it is unaffected by every GC lever you have.**

Bigger heap: no effect. Smaller pause goal: no effect. ZGC: no effect — and worse, ZGC
makes the GC log look so clean that people conclude the JVM is innocent. Topic 72's
Trap 4 exists specifically to warn about this.

**5. Because non-GC safepoint operations are in your production stack right now.**

Your APM agent. Your `/actuator/threaddump` endpoint. Your profiler. Your Kubernetes
liveness probe if someone wired it to a diagnostic endpoint. Each of those requests a
global safepoint, and each of them pays your TTSP bill.

**6. Because on `orderflow` you have a recorded baseline, which makes this falsifiable.**

You have p999 in `/docs/java/baselines/`. You can produce a stall, measure it, name it,
fix it, and re-measure. That is the difference between knowing this exists and being
able to diagnose it at 3am.

---

## Machine-level reality

### The polling instruction

A safepoint poll in compiled code is not a branch. Branches cost prediction resources
and instruction bytes in the hottest code in the process. HotSpot uses something
cheaper and cleverer.

**The polling page trick.** The JVM reserves a page of memory — the *polling page*.
Compiled code emits a **load from that page** at each poll site. On x86-64 the
instruction is a test against the page address; conceptually:

```
    ; ... loop body ...
    test  DWORD PTR [poll_page_address], eax      ; the safepoint poll
    ; ... loop back-edge ...
```

*Illustration of the mechanism, not disassembly from a specific JVM.*

While no safepoint is requested, that page is **readable**, the load succeeds, costs a
cache hit and nothing else, and execution continues. There is no branch to mispredict.

To request a safepoint, the JVM **removes read permission from the page**
(`mprotect(PROT_NONE)`). Now the next thread to execute a poll takes a **SIGSEGV**. The
JVM installs a signal handler that recognises the faulting address as the polling page,
and parks the thread at the safepoint.

This is genuinely elegant: **the fast path costs one load and no branch, and the slow
path is delivered by the memory-protection hardware.**

### Thread-local handshakes — polls became per-thread

Since JEP 312 ("Thread-Local Handshakes", JDK 10), each thread has its **own** polling
address stored in its thread-local storage. The poll becomes: load the poll address from
TLS, then test against it. That extra load is the cost; the benefit is enormous.

Because each thread has its own poll address, the JVM can stop **one** thread without
stopping all of them. That is a *handshake* rather than a global safepoint, and it is
why many operations that used to require a global stop-the-world — some kinds of stack
walking, some deoptimization, revoking a single biased lock back when that existed — no
longer do.

**What this means for you:** the JVM has been steadily moving work *out* of global
safepoints. That is good, and it does not eliminate this topic — GC still needs a
global safepoint for its stop-the-world phases, and TTSP still gates it.

### Why the poll cannot go just anywhere

You might reasonably ask: why not poll every few instructions and bound TTSP tightly?

Because each poll site requires an **oop map** — a compiler-generated table saying which
registers and stack slots hold references at that exact point. Oop maps cost memory in
the code cache (Topic 74) and constrain the optimiser: the compiler must keep references
in identifiable locations across a poll, which limits register allocation and
reordering. Polls are not free, and putting them in the innermost loop of hot numeric
code is exactly where they cost the most.

**So the JVM is making a deliberate trade: peak throughput in tight loops, against
bounded time-to-safepoint.** Loop strip mining is the compromise. Understanding that
this is a *trade the compiler made on your behalf* is the mature version of this topic.

### What else makes TTSP long

Counted loops are the famous cause. They are not the only one, and an interview answer
that names only them is incomplete.

| Cause | Mechanism | How you would spot it |
|---|---|---|
| **Counted `int` loop over a large array** | Poll elided or strip-mined with a large strip | The classic. Look for CPU-bound array work in the application. |
| **Large `System.arraycopy` / `Arrays.fill` / `Arrays.sort`** | Intrinsics compile to tight machine loops with no poll inside | Copying or clearing multi-megabyte arrays. Common in serialization and buffer code. |
| **Zeroing a huge new array** | `new byte[100_000_000]` must be zeroed; that zeroing is uninterruptible | Correlates with large allocations (Topic 71's humongous discussion) |
| **JNI critical sections** | `GetPrimitiveArrayCritical` pins the heap; the JVM cannot proceed | Native libraries, some compression and crypto code |
| **Page faults / swapping** | The thread is descheduled by the OS mid-poll; it cannot arrive until it is scheduled again | Host memory pressure. Check the machine, not the JVM. |
| **CPU starvation / cgroup throttling** | The thread is runnable but not scheduled | `cat /sys/fs/cgroup/cpu.stat` — `nr_throttled`, `throttled_usec`. Topic 82. |
| **Noisy neighbour / steal time** | The hypervisor descheduled your vCPU | `vmstat` steal column |
| **An enormous number of threads** | The *at-safepoint* work grows with thread count even when TTSP is fine | Topic 98, Topic 101 |

**Note the last four are not your code's fault at all.** TTSP is a property of the whole
system: application code, JVM, container, kernel, hypervisor. When TTSP is long and you
cannot find a loop, look down the stack rather than harder at your code.

### The two numbers, and why the split is the whole diagnosis

Every safepoint event has (at least) two durations that you must never conflate:

| Number | What it measures | Whose fault it usually is |
|---|---|---|
| **Time to reach the safepoint (TTSP)** | From "safepoint requested" to "last thread parked" | **Your application code**, or the OS/container |
| **Time at the safepoint** | The VM operation itself — GC work, dumping threads, redefining classes | The collector, or whatever requested the operation |

**Total stop-the-world duration = reaching + at.** The GC log prints only the second
one. Your users experience the sum.

Say this in an interview and watch the room change: *"GC pause and stop-the-world
duration are different numbers, and only one of them is in the GC log."*

### The safepoint log, and what its fields mean

The relevant logging is `-Xlog:safepoint*`. The shape of a summary line:

```
[2026-08-29T14:02:11.417+0000][612.041s][info][safepoint] Safepoint "G1CollectForAllocation", Time since last: 481027 ns, Reaching safepoint: 912440000 ns, At safepoint: 5117000 ns, Total: 917557000 ns
```

***Illustration of the format, not captured output. The numbers are placeholders chosen
to show the field positions and the SHAPE of the problem — a large "Reaching" beside a
small "At".***

Field by field:

| Field | Meaning | How you use it |
|---|---|---|
| `[2026-08-29T...]` | Wall clock (from the `time` decorator) | Correlate with k6 and the APM |
| `[612.041s]` | Uptime (from `uptime`) | Intervals between safepoints |
| `[safepoint]` | Tag set | **Grep on this** |
| `"G1CollectForAllocation"` | **The VM operation name** | The diagnostic field: *why* a safepoint was requested. GC? A thread dump? Deoptimization? |
| `Time since last` | Gap since the previous safepoint | Very small values mean safepoints are frequent — an agent or a pathological workload |
| **`Reaching safepoint`** | **TTSP** | **The number nobody looks at. If it dominates, your problem is not GC.** |
| **`At safepoint`** | The VM operation's own duration | This is roughly what the GC log reports for a GC operation |
| `Total` | Reaching + At | **What your users actually experienced** |

> **Version note, flagged:** the exact field names, units and ordering of
> `-Xlog:safepoint` output have varied across JDK releases, and there are additional
> tags (`safepoint+stats`, `safepoint+cleanup`) with their own formats. The legacy
> `-XX:+PrintSafepointStatistics` flag was **removed** in favour of unified logging.
> **Do not memorise the exact line; memorise that the reaching time and the at time are
> reported separately, and go find them in your own output.** Confirm what your JDK
> emits:
>
> ```bash
> java -Xlog:safepoint -version
> java -Xlog:safepoint*=debug -version | head -40
> ```

### The other flags that matter

```bash
# Print the threads that failed to reach a safepoint within N milliseconds.
# This is the flag that NAMES THE CULPRIT THREAD.
-XX:+SafepointTimeout -XX:SafepointTimeoutDelay=500

# Nuclear option for a reproducible test rig: abort the VM on a slow safepoint,
# producing a full crash log with all thread stacks. NEVER in production.
-XX:+AbortVMOnSafepointTimeout

# Force safepoint polls in counted loops (and control the strip size).
# Use these as an A/B CONTROL in the drill, not as a production setting
# without measurement.
-XX:+UseCountedLoopSafepoints
-XX:LoopStripMiningIter=1000

# The JVM periodically forces a safepoint even with nothing to do.
-XX:GuaranteedSafepointInterval=1000
```

> **Flagged uncertainty:** `GuaranteedSafepointInterval`'s default and its precise role
> have changed as thread-local handshakes absorbed work that used to need a global
> safepoint. I am not asserting its value on your JDK. `-XX:+PrintFlagsFinal -version |
> grep -i GuaranteedSafepointInterval` settles it, and
> `-XX:+UnlockDiagnosticVMOptions` may be required for some of these.

**`-XX:+SafepointTimeout` is the flag most engineers have never heard of and it is the
one that turns a mystery into a name.** When a thread takes too long, the JVM prints
which thread it was waiting for. That is your culprit, identified by the JVM itself.

---

## Example 1 — minimal

The smallest program that produces the failure: one thread in a long counted `int` loop,
another thread forcing frequent safepoints.

```java
package com.orderflow.lab.safepoint;

import java.util.concurrent.CountDownLatch;

/**
 * Two threads:
 *   WORKER    - a long counted int loop over a large array. Candidate straggler.
 *   DISTURBER - allocates hard, forcing frequent GCs, each of which requests a
 *               GLOBAL SAFEPOINT and therefore has to wait for WORKER.
 *
 * The DISTURBER's own latency is the observable: it measures how long each of its
 * short units of work took. When WORKER delays a safepoint, DISTURBER stalls even
 * though DISTURBER is doing nothing wrong and is not in a loop of its own.
 *
 * args[0] = "int" or "long"  - the type of the WORKER's loop counter
 * args[1] = array length      - e.g. 500000000
 *
 * IMPORTANT: the numbers this prints are a crude in-process observation, useful only
 * to compare two runs of THIS program. They are not a benchmark and must never be
 * quoted as one. The authoritative evidence is -Xlog:safepoint*.
 */
public final class SafepointStraggler {

    /** Static sink: prevents dead-code elimination of the loop's result (Topic 75). */
    public static volatile long sink;
    public static volatile boolean running = true;

    public static void main(String[] args) throws Exception {
        String counterType = args[0];
        int length = Integer.parseInt(args[1]);

        System.out.println("allocating array of " + length + " ints...");
        int[] data = new int[length];
        for (int i = 0; i < length; i++) {
            data[i] = i & 0xFF;
        }
        System.out.println("array ready");

        CountDownLatch started = new CountDownLatch(1);

        Thread worker = new Thread(() -> {
            started.countDown();
            while (running) {
                if (counterType.equals("int")) {
                    sink += sumWithIntCounter(data);
                } else {
                    sink += sumWithLongCounter(data);
                }
            }
        }, "worker");

        Thread disturber = new Thread(() -> {
            long worstMillis = 0;
            long iterations = 0;
            // Allocate steadily. Each young collection needs a global safepoint,
            // and therefore has to wait for `worker` to arrive at one.
            while (running) {
                long t0 = System.nanoTime();
                for (int i = 0; i < 200; i++) {
                    sink += new byte[64 * 1024].length;     // steady young allocation
                }
                long elapsedMillis = (System.nanoTime() - t0) / 1_000_000;
                if (elapsedMillis > worstMillis) {
                    worstMillis = elapsedMillis;
                    System.out.println("disturber: new worst unit of work = "
                        + worstMillis + " ms  (iteration " + iterations + ")");
                }
                iterations++;
            }
        }, "disturber");

        worker.setDaemon(true);
        disturber.setDaemon(true);
        worker.start();
        started.await();
        disturber.start();

        Thread.sleep(60_000);
        running = false;
        System.out.println("done; sink = " + sink);
    }

    /**
     * COUNTED int loop. This is the shape whose safepoint poll C2 may elide,
     * or strip-mine with an outer poll depending on your JDK's defaults.
     */
    private static long sumWithIntCounter(int[] data) {
        long total = 0;
        for (int i = 0; i < data.length; i++) {
            total += data[i];
        }
        return total;
    }

    /**
     * The classic "fix": a long induction variable. Historically NOT recognised as a
     * counted loop, so the back-edge poll survived. Modern C2 handles long counted
     * loops too, so the difference may be small or absent on your JDK. MEASURE IT.
     */
    private static long sumWithLongCounter(int[] data) {
        long total = 0;
        for (long i = 0; i < data.length; i++) {
            total += data[(int) i];
        }
        return total;
    }
}
```

Run it four ways, changing exactly one thing each time:

```bash
javac -d out SafepointStraggler.java

# A: int counter, JDK defaults.
java -Xms2g -Xmx2g -XX:+UseG1GC \
  -Xlog:safepoint*:file=logs/sp-int-default.log:time,uptime,level,tags \
  -Xlog:gc:file=logs/gc-int-default.log:time,uptime,level,tags \
  -cp out com.orderflow.lab.safepoint.SafepointStraggler int 500000000

# B: int counter, counted-loop safepoints explicitly DISABLED.
#    This is the CONTROL that exposes the mechanism even if your default protects you.
java -Xms2g -Xmx2g -XX:+UseG1GC -XX:-UseCountedLoopSafepoints \
  -Xlog:safepoint*:file=logs/sp-int-nopoll.log:time,uptime,level,tags \
  -cp out com.orderflow.lab.safepoint.SafepointStraggler int 500000000

# C: int counter, counted-loop safepoints ON with a small strip.
java -Xms2g -Xmx2g -XX:+UseG1GC -XX:+UseCountedLoopSafepoints -XX:LoopStripMiningIter=1000 \
  -Xlog:safepoint*:file=logs/sp-int-strip.log:time,uptime,level,tags \
  -cp out com.orderflow.lab.safepoint.SafepointStraggler int 500000000

# D: long counter, JDK defaults. The classic "fix".
java -Xms2g -Xmx2g -XX:+UseG1GC \
  -Xlog:safepoint*:file=logs/sp-long-default.log:time,uptime,level,tags \
  -cp out com.orderflow.lab.safepoint.SafepointStraggler long 500000000
```

**WHAT TO LOOK FOR.** In the safepoint logs, for each run, the largest **reaching**
time and the largest **at** time. Extract them and compare the four runs.

```bash
for f in logs/sp-*.log; do
  echo "=== $f"
  grep -o 'Reaching safepoint: [0-9]* ns' "$f" | grep -o '[0-9]*' | sort -n | tail -1
  grep -o 'At safepoint: [0-9]* ns'       "$f" | grep -o '[0-9]*' | sort -n | tail -1
done
```

*(Adapt the field names to whatever your JDK actually prints — check with
`java -Xlog:safepoint -version` first. Do not assume this grep matches your output.)*

| What you see | What it means |
|---|---|
| Run B has a far larger maximum **reaching** time than run A | **The mechanism, demonstrated.** With counted-loop safepoints off, the poll is elided and one thread delays every other for the length of the loop. Your default was protecting you; now you have seen what it protects you from. |
| Runs A and B are similar | Your JDK's default may already differ from what the flag name suggests, or the loop is being optimised in a way that keeps polls. Check `jcmd VM.flags -all` for the actual value and re-read the version note above. **This is a legitimate result, not a failed experiment.** |
| Reaching time is large and At time is small, in any run | **The signature of the entire topic.** The application stopped for a long time and GC did almost none of it. Write down the ratio. |
| Run C (small strip) has much smaller reaching times than run B | Strip mining working exactly as designed: the outer loop's poll bounds TTSP. |
| Run D (`long`) is no better than run A | Modern C2 handles long counted loops. **The classic advice is dated, and you have just proven it on your own runtime** — that is a more valuable finding than confirming the folklore. |
| Run D is much better than run B | The classic behaviour survives on your JDK for this shape. Note it, and note that it is a JDK-specific observation, not a law. |
| The disturber's "worst unit of work" tracks the maximum reaching time | Cause and effect connected: a thread doing nothing wrong stalled because of a thread it has no relationship with. |
| No safepoint log at all | Test the syntax alone: `java -Xlog:safepoint -version` should print lines. Then fix your file/decorator spec. |
| Everything is fast in every run | The array may not be large enough, or the loop may be vectorised so aggressively that it completes quickly. Raise the array length until one pass takes a visible amount of time, then retry. |

**The point of running all four is not to get one number.** It is to build the map:
which flag changes which number, and by how much, on *your* runtime. That map is what
you will use at 3am.

---

## Example 2 — production scenario (on the project spine)

### The constraints, taken from the Topic 65 baseline

| Thing | Value |
|---|---|
| Service | `orderflow`, containerised |
| Dataset | 100k products, 1M orders, **5M order lines** |
| Container | 2 vCPU, 2 GiB memory limit |
| Heap | `-Xmx1200m -Xms1200m` |
| Collector | G1, verified with `jcmd VM.flags -all` |
| Load mix | 70% catalogue read, 20% order read, 10% order placement |
| Baseline artefacts | `/docs/java/baselines/` — p50/p95/p99/p999 per endpoint |

### The change that causes the incident

Finance asks for a daily revenue reconciliation. Someone implements it as a scheduled
job **inside the same JVM as the API** — which is common, expedient, and the root of
the whole story.

```java
package com.orderflow.reporting;

import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

@Component
public class RevenueReconciliationJob {

    private final OrderLineExtractor extractor;

    public RevenueReconciliationJob(OrderLineExtractor extractor) {
        this.extractor = extractor;
    }

    @Scheduled(cron = "0 */5 * * * *")     // every five minutes
    public void reconcile() {
        // Pull the day's order-line amounts into a primitive array.
        // Primitive, deliberately: someone read Topic 01 and avoided boxing.
        long[] amounts = extractor.todaysLineAmountsMinor();   // several million entries

        long total = 0;
        // A textbook-perfect loop. Primitive array, int counter, no allocation,
        // no boxing, no stream overhead. Reviewed and approved by everyone.
        for (int i = 0; i < amounts.length; i++) {
            total += amounts[i];
        }

        long refunds = 0;
        for (int i = 0; i < amounts.length; i++) {
            if (amounts[i] < 0) {
                refunds += amounts[i];
            }
        }

        reportRepository.save(new RevenueReport(total, refunds));
    }
}
```

Read that as a Java reviewer and it is **exemplary**. Primitive array, no boxing, no
allocation in the loop, no stream, no lambda. It is the code you would hold up as a
good example after Topic 01.

Read it as someone who knows safepoints and it is a **stop-the-world generator that
fires every five minutes**.

### What you observe, in the order you observe it

1. **Every five minutes, p999 spikes across every endpoint** — `GET /products`,
   `GET /orders`, `POST /orders`. All of them. At the same instant.
2. **The GC log accounts for none of it.** Pause durations are normal. There are no
   Full GCs, no evacuation failures, no humongous allocation.
3. **The database shows nothing.** No slow queries at those timestamps. Connection pool
   utilisation is normal.
4. **The CPU profile of the spike window is nearly empty.** Almost every thread is
   parked. The only thing running is a loop summing a `long[]`, which looks innocuous.
5. **Someone notices the five-minute periodicity** and connects it to the cron
   expression — and this is usually the moment the penny drops, days in.
6. **Someone "fixes" it by moving to ZGC.** The GC log becomes even cleaner. The spikes
   are unchanged. Now the team is more confused than before, because the collector with
   sub-millisecond pauses did not help.

### Why every endpoint spiked

Because the safepoint is **global**. When G1 decides to do a young collection — which
it does regularly, driven by the API's own allocation — it requests a safepoint. Every
request-handling thread parks promptly. The reconciliation thread is in a counted loop
over several million elements and does not check the flag until it reaches a poll.

**Every parked thread waits.** The requests they were serving are frozen. Those requests
have nothing to do with reconciliation, revenue, or the job. They are collateral.

This is the structural fact to internalise: **in Java, one thread's uninterruptible
loop freezes the entire process, and the freeze appears in no per-endpoint metric,
because no endpoint caused it.**

### The diagnosis, as commands

```bash
# 1. Confirm the collector and the counted-loop safepoint flags on the RUNNING container.
jcmd $(pgrep -f orderflow) VM.flags -all \
  | grep -iE "UseG1GC|UseZGC|UseCountedLoopSafepoints|LoopStripMiningIter|GuaranteedSafepointInterval"

# 2. Restart with safepoint logging alongside GC logging, on the same clock.
-Xlog:gc*:file=/var/log/orderflow/gc.log:time,uptime,level,tags:filecount=5,filesize=50m
-Xlog:safepoint*:file=/var/log/orderflow/safepoint.log:time,uptime,level,tags:filecount=5,filesize=50m

# 3. Add the flag that NAMES the straggler thread.
-XX:+SafepointTimeout -XX:SafepointTimeoutDelay=500

# 4. Re-run the Topic 65 load profile, unchanged, for long enough to span
#    at least three cron firings.

# 5. Find the worst safepoints and look at the SPLIT.
grep -i 'safepoint' /var/log/orderflow/safepoint.log | tail -50

# 6. Correlate the timestamps with the cron schedule and with k6's spike timestamps.
```

**WHAT TO LOOK FOR:** safepoint events whose **reaching** time is large while the **at**
time is small, occurring on a five-minute cadence, with `SafepointTimeout` naming the
scheduler thread.

| What you see | What it means |
|---|---|
| Large reaching time, small at time, every five minutes, and `SafepointTimeout` names the scheduling thread | **Diagnosis complete.** You can write the sentence: "the stall at 14:05:00 was a 900 ms time-to-safepoint caused by the reconciliation job's counted loop; GC contributed 5 ms of it." |
| Large reaching time but `SafepointTimeout` names a request-handling thread | A different straggler. Look for large `System.arraycopy`, `Arrays.sort` on a big array, a large `new byte[]` being zeroed, or JNI. Same mechanism, different code. |
| Large reaching time on **all** threads simultaneously | Not a loop. The threads are not being scheduled. Check cgroup throttling (`/sys/fs/cgroup/cpu.stat`), host memory pressure and swap, and hypervisor steal time. Topic 82. |
| Reaching time is small and **at** time is large | The opposite problem, and a genuinely useful finding: the VM operation itself is slow. If the operation is a GC, that is Topic 71. If it is a thread dump or class redefinition, look at your agents (Topic 81). |
| Safepoints occurring far more often than GCs | Something else is requesting them. Read the **operation name** field. A monitoring agent taking thread dumps, JFR, a profiler, or deoptimization storms (Topic 74). |
| Nothing correlates with the spikes at all | Widen the net: the load generator's own coordinated omission, the network, the database's checkpointing, or an external dependency. But you have now *ruled out* the JVM cheaply and correctly, which is a real result. |

### The fixes, in the order you should consider them

**Fix 1 — move the job out of the request-serving JVM (best).**
The reconciliation job has no business sharing a JVM with a latency-sensitive API. Run
it as a separate pod, a separate deployment, a Kubernetes `CronJob`. **It cannot delay
a safepoint in a JVM it is not in.** This fixes the safepoint problem, the CPU
contention problem, the heap-pressure problem and the deployment-coupling problem in
one move. It is more work than a flag and it is the right answer.

**Fix 2 — do the aggregation in the database.**
`SELECT SUM(amount_minor) FROM order_line WHERE ...`. There is no reason to move
several million rows into the JVM to add them up. This removes the loop, the array, the
allocation and the safepoint problem simultaneously, and it is almost certainly faster.
**The best fix for a JVM problem is frequently not a JVM change.**

**Fix 3 — break the loop so it polls.**
If the computation genuinely must happen in-process, restructure it so safepoint polls
occur. Chunking the work through a method call gives you polls at method returns:

```java
private static final int CHUNK = 32_768;

long total = 0;
for (int start = 0; start < amounts.length; start += CHUNK) {
    int end = Math.min(start + CHUNK, amounts.length);
    total += sumChunk(amounts, start, end);      // method RETURN is a poll site
}

/** Small enough to be inlined-friendly; the return provides the poll. */
private static long sumChunk(long[] amounts, int start, int end) {
    long sum = 0;
    for (int i = start; i < end; i++) {
        sum += amounts[i];
    }
    return sum;
}
```

> **Caveat, stated honestly:** if C2 inlines `sumChunk` into the caller and then fuses
> the loops back together, you may have achieved nothing. **This fix must be verified
> with `-Xlog:safepoint*`, not assumed.** That verification requirement is the whole
> reason this topic insists on measurement.

**Fix 4 — turn on counted-loop safepoints explicitly.**
`-XX:+UseCountedLoopSafepoints -XX:LoopStripMiningIter=1000`. This bounds TTSP for
every counted loop in the process at some throughput cost in tight numeric loops.

**This is the tempting fix and you should know its cost.** It is a **global** change
affecting every hot loop in the service to fix one badly-placed job. If your default
already enables it, this fix is a no-op and you will have "fixed" it by accident while
the real cause remains. **Verify the before-and-after with the safepoint log, and
verify the throughput cost against the Topic 65 baseline.**

**Fix 5 — change the counter to `long`.**
The classic advice. **Verify it on your JDK before relying on it**, per the version note
above. If it works, it works; if modern C2 treats the long loop the same way, you have
changed a type for nothing and left the bug in place.

> **The rule to carry:** when the fix is "change a JVM flag" and the alternative is
> "do not run this work here", the second is almost always right. A flag bounds the
> damage globally. Moving the work removes it.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — a reported 5 ms GC pause hiding a 900 ms time-to-safepoint

**Wrong:** open the GC log, see that all pauses are small, conclude GC is not the
problem, and move on to the database.

**Exact symptom:**

- p999 shows stalls far larger than anything in the GC log.
- The stalls hit **requests already in flight on threads doing no GC work**, across
  unrelated endpoints, **at the same instant**.
- A CPU profile of the stall window is nearly empty — almost every thread parked.
- The database, the connection pool and the downstream services all show nothing at
  those timestamps.
- Switching collectors changes nothing except making the GC log look even better.

**Root cause:** the number in a `Pause Young` line is the time spent doing GC work **at**
the safepoint. It excludes the time spent **reaching** it. Total stop-the-world is
reaching plus at, and only the second half is in the GC log. A thread in a counted loop,
a large `System.arraycopy`, a huge array being zeroed, or one that has been swapped out
can add hundreds of milliseconds that GC gets blamed for and did not cause.

**Fix:**

1. **Turn on the log that has the missing number**, on the same clock as the GC log:
   ```bash
   -Xlog:safepoint*:file=/var/log/orderflow/safepoint.log:time,uptime,level,tags
   ```
2. **Compare reaching time against at time** for the worst events. If reaching
   dominates, GC is exonerated and no collector change will help.
3. **Name the straggler** with `-XX:+SafepointTimeout -XX:SafepointTimeoutDelay=500`.
   The JVM prints which thread it waited for.
4. **Fix the code**, not the collector.

**How to confirm the diagnosis in one command:**

```bash
# Do the largest safepoint totals come from reaching, or from the operation itself?
grep -i 'Reaching safepoint' /var/log/orderflow/safepoint.log | sort -t: -k2 -n | tail -10
```

---

### Trap 2 — a counted `int` loop whose poll is elided, freezing every other thread

**Wrong:**

```java
// Reviewed, approved, and held up as an example of allocation-free Java.
long total = 0;
for (int i = 0; i < orderLineAmounts.length; i++) {   // several million elements
    total += orderLineAmounts[i];
}
```

**Exact symptom:**

- Latency spikes correlate with **when this code runs**, not with load.
- If it runs on a schedule, the spikes are **periodic and suspiciously round** — every
  five minutes, every hour, on the hour.
- The spikes are **global**: every endpoint, every thread, one instant.
- `-Xlog:safepoint*` shows large reaching times; `-XX:+SafepointTimeout` names the
  thread running this loop.
- The loop itself does not appear slow. It completes in a time nobody would question.
  **Its problem is not its duration; it is its uninterruptibility.**

**Root cause:** an `int`-counted loop with a loop-invariant bound is a **counted loop**.
C2 knows it terminates, so the back-edge safepoint poll is a candidate for elision, or
is strip-mined with a strip large enough to still be significant. During the
uninterruptible stretch, the thread does not observe the safepoint request. Every other
thread has already parked and is waiting.

**Fix:**

1. **Confirm the mechanism before fixing** — `-Xlog:safepoint*` and
   `-XX:+SafepointTimeout` naming this thread. Do not fix a loop on suspicion.
2. **Move the work out of the latency-sensitive JVM**, or push it into the database.
   Best fix; least JVM cleverness.
3. **Chunk the loop through a method boundary** so returns provide poll sites — and
   **verify with the safepoint log**, because inlining can undo it.
4. **`-XX:+UseCountedLoopSafepoints -XX:LoopStripMiningIter=<n>`** as a global bound,
   measured against the Topic 65 baseline for its throughput cost.
5. **Try a `long` counter** — and measure, because modern C2 may treat it the same way.
6. **Do not** reach for a bigger heap, a different collector, or a pause goal. None of
   them touch this.

---

### Trap 3 — blaming GC for a pause GC did not cause

**Wrong:** "Our p99 is bad. The GC log has pauses in it. Let's tune GC." Two weeks of
heap sizing, pause goals, collector comparisons and region tuning follow.

**Exact symptom:**

- Every GC tuning change produces results **inside the run-to-run noise**. Nothing
  helps and nothing hurts, which is itself the clue.
- The spikes persist unchanged across G1, ZGC and different heap sizes.
- Someone eventually notices the spikes correlate with something entirely non-GC: a
  monitoring agent's polling interval, a deployment, a scheduled job, a thread dump
  endpoint being scraped.
- The GC pause **counts** and **durations** are stable across the whole period,
  including during the spikes.

**Root cause:** two independent errors compounding.

1. **GC was assumed rather than tested.** The presence of GC pauses in a log is not
   evidence that GC caused a spike. It is evidence that GC happened, which it always
   does.
2. **The stall came from a different safepoint operation, or from outside the JVM
   entirely.** Thread dumps, class redefinition by an APM agent (Topic 81),
   deoptimization storms (Topic 74), cgroup CPU throttling (Topic 82), or host swapping
   all produce stalls that look exactly like GC pauses from the application's point of
   view — and appear in no GC log.

**Fix:**

1. **Correlate before theorising.** Put the GC log, the safepoint log and the request
   latency histogram on the same clock. If the long GC pauses do not coincide with the
   latency spikes, GC is exonerated in one step.
2. **Read the safepoint log's operation-name field.** It tells you *what kind* of
   safepoint stopped you. If your stalls are `ThreadDump` operations, your problem is a
   monitoring agent, not a collector.
3. **Count the safepoints.** If safepoints vastly outnumber GCs, something else is
   requesting them, and the operation name says what.
4. **Check outside the JVM:**
   ```bash
   cat /sys/fs/cgroup/cpu.stat        # nr_throttled, throttled_usec
   vmstat 1 10                        # si/so (swap), st (steal)
   ```
5. **Adopt the discipline permanently:** GC is a *hypothesis to falsify*, never the
   default explanation.

---

### Trap 4 — a monitoring agent taking thread dumps in a loop

**Wrong:** a dashboard, a health check, or a homegrown "thread monitor" calls
`Thread.getAllStackTraces()` or hits `/actuator/threaddump` every few seconds. It is
"just observability", so nobody reviews it.

**Exact symptom:**

- Latency spikes on a **very regular** cadence matching a polling interval — every 10
  seconds, every 15, every 30.
- The GC log shows no pause at those timestamps.
- The safepoint log shows safepoints at exactly that cadence with an operation name
  indicating a thread dump or stack-trace collection, **not** a collection.
- The effect is **worse in a JVM with many threads**, because the at-safepoint work
  scales with thread count — a thread-per-request service with hundreds of threads pays
  much more per dump than a small one.
- Removing the monitoring makes the spikes vanish, which is the moment everyone is
  surprised.

**Root cause:** a thread dump requires a **global safepoint**. It is not a free
read-only operation. Every dump stops the world twice over: once to get everyone to a
safepoint (your TTSP bill), and again for the time spent walking every thread's stack.
Doing it on a timer means paying that bill on a timer.

**Fix:**

1. **Read the operation name in the safepoint log.** That is the whole diagnosis, and
   it takes one grep.
2. **Reduce the frequency dramatically**, or make dumps on-demand rather than periodic.
   A thread dump is a diagnostic tool, not a metric.
3. **Use a sampling profiler that is not safepoint-based.** async-profiler uses
   `AsyncGetCallTrace` and perf events rather than safepoints (Topic 78). JFR's
   overhead profile is designed for continuous production use.
4. **Reduce the thread count** if it is large for no reason (Topic 98), because at-
   safepoint cost scales with it. Virtual threads change this arithmetic (Topic 101).
5. **Review observability code with the same rigour as request-path code.** "It is only
   monitoring" is how this gets shipped.

---

### Trap 5 — assuming a low-pause collector solves it

**Wrong:** the GC log shows pauses, so migrate to ZGC. Sub-millisecond pauses will fix
the stalls.

**Exact symptom:**

- The GC log becomes beautiful. Every pause is a fraction of a millisecond.
- **The p999 stalls are completely unchanged.**
- The team is now *more* confused, because the obvious suspect has been eliminated with
  no improvement, and there is no remaining hypothesis.
- Throughput has dropped, because the load barrier taxes every reference load (Topic
  72), so the migration was a pure loss.

**Root cause:** ZGC's pause is short **at** the safepoint. It does not change the
safepoint protocol, the poll placement, or how long a thread takes to arrive. **Time-to-
safepoint is a property of the application code and the operating system, not of the
collector.** Every collector — Serial, Parallel, G1, ZGC, Shenandoah — waits for the
last thread.

Worse: ZGC's clean log actively **hides** the problem, because the number people look
at gets smaller while the number that mattered is unchanged and still not printed.

**Fix:**

1. **Measure TTSP before any collector change.** Make it a checklist item: no collector
   migration proposal is complete without a safepoint-log analysis.
2. **If reaching time dominates, do not change the collector.** Fix the code.
3. **If you already migrated**, evaluate honestly whether to revert: you are paying the
   barrier cost for a benefit you did not receive.
4. **Put this in the runbook**: "A clean GC log under a low-pause collector is
   *specifically weak* evidence that the JVM is innocent, because the two most common
   JVM stalls — TTSP and allocation stalls — do not appear in it as pauses."

---

## Hands-on proof

Every command here is one **you** run. I have no JVM and will not print output and call
it captured. What follows is the exact command, what to look for, and how to read every
plausible result — including the ones that contradict the expected answer.

### Setup

```bash
mkdir -p ~/java-lab/73 && cd ~/java-lab/73
java --version                 # record it; safepoint log format varies by version
mkdir -p logs out
```

### Proof 1 — find out what your JDK actually prints

Before anything else, learn your own log format. Do not use mine.

```bash
java -Xlog:safepoint -version
java -Xlog:safepoint*=debug -version 2>&1 | head -40
java -Xlog:help | grep -i safepoint
```

**WHAT TO LOOK FOR:** the exact field names and units your JDK emits.

| What you see | What it means |
|---|---|
| Lines with separately reported reaching and at durations | Good. Note the exact field names and adapt every grep in this document to them. |
| Only a total duration | Raise the level: try `-Xlog:safepoint*=debug` or the `safepoint+stats` tag. |
| Nothing at all | Your `-Xlog` syntax is wrong. `java -Xlog:gc -version` should print GC lines; get that working first. |
| A format different from anything described here | **Expected and fine.** The format has moved across releases. The concepts — an operation name, a reaching time, an at time — are stable. Find them in your output. |

### Proof 2 — establish your counted-loop safepoint defaults

```bash
for GC in UseSerialGC UseParallelGC UseG1GC UseZGC; do
  printf "%-16s " "$GC"
  java -XX:+$GC -XX:+PrintFlagsFinal -version 2>/dev/null \
    | awk '/UseCountedLoopSafepoints|LoopStripMiningIter/ {printf "%s=%s  ", $2, $4}'
  echo
done
```

**WHAT TO LOOK FOR:** whether the values differ by collector.

| What you see | What it means |
|---|---|
| `UseCountedLoopSafepoints` true under the low-pause collectors | Those collectors need bounded TTSP more than Parallel does, and the JVM sets the flag accordingly. Your default is protecting you — and you now know the mechanism it protects you from. |
| Values identical across collectors | Simpler world. Note the value and move on. |
| `LoopStripMiningIter` is a large number | TTSP is bounded, but by the time taken for that many iterations — which for expensive loop bodies is not a small bound. **A bound is not the same as a small bound.** |
| The flags do not exist | An older or unusual JVM. Record what you have; the mechanism is still worth understanding. |

### Proof 3 — see a safepoint that is not a GC

Prove to yourself that non-GC operations stop the world.

```bash
# Terminal 1: start any long-running Java process with safepoint logging.
java -Xms1g -Xmx1g -XX:+UseG1GC \
  -Xlog:safepoint*:file=logs/sp-threaddump.log:time,uptime,level,tags \
  -cp out com.orderflow.lab.safepoint.SafepointStraggler long 100000000

# Terminal 2: request thread dumps in a loop and watch the safepoint log grow.
for i in $(seq 1 20); do jcmd $(pgrep -f SafepointStraggler) Thread.print > /dev/null; done

# Now count safepoints by operation name.
grep -oE 'Safepoint "[A-Za-z]+"' logs/sp-threaddump.log | sort | uniq -c | sort -rn
```

**WHAT TO LOOK FOR:** safepoint operations whose names are clearly **not** garbage
collection.

| What you see | What it means |
|---|---|
| Safepoint operations named for thread dumping appearing 20 times | **Confirmed: a thread dump is a stop-the-world event.** Every APM agent that does this on a timer is stopping your service on a timer. |
| A wide variety of operation names | Good — you are seeing the real breadth of VM operations. Read them. Deoptimization, inline-cache handling, cleanup and GC all appear. |
| Only GC-related names | Your dumps may not have landed. Check `jcmd` returned successfully and that you targeted the right pid. |
| Enormous numbers of safepoints for operations you did not request | Something is doing it. Deoptimization storms (Topic 74), JFR, an agent. The operation name is the diagnosis. |

### Proof 4 — name the straggler thread with `SafepointTimeout`

This is the highest-value flag in the topic.

```bash
java -Xms2g -Xmx2g -XX:+UseG1GC -XX:-UseCountedLoopSafepoints \
  -XX:+SafepointTimeout -XX:SafepointTimeoutDelay=200 \
  -Xlog:safepoint*:file=logs/sp-timeout.log:time,uptime,level,tags \
  -cp out com.orderflow.lab.safepoint.SafepointStraggler int 500000000 \
  2>&1 | tee logs/stdout-timeout.log

grep -i -A5 'safepoint' logs/stdout-timeout.log | head -40
```

**WHAT TO LOOK FOR:** JVM output naming the thread or threads that failed to reach the
safepoint within the delay.

| What you see | What it means |
|---|---|
| A message naming the `worker` thread | **The JVM identified your culprit for you.** In production this is the single fastest path from "we have stalls" to "this thread, this code". |
| Nothing printed | Either no safepoint exceeded the delay — lower `SafepointTimeoutDelay` — or the flag needs `-XX:+UnlockDiagnosticVMOptions` on your build. Try adding it. |
| It names a thread you did not expect | Follow it. A JDBC driver, a serialization library, a compression codec doing a big `arraycopy`. Same mechanism, code you did not write. |
| It names many threads at once | Not a straggler problem. The threads are not being scheduled. Look at cgroup throttling, swap and steal time. |

### Proof 5 — prove it is global, from inside the application

The most convincing demonstration is a thread that measures its **own** stalls while
doing nothing but sleeping in small increments.

```java
package com.orderflow.lab.safepoint;

/**
 * A "jitter detector". It sleeps for 1 ms in a loop and measures how much longer
 * than 1 ms each iteration actually took.
 *
 * It allocates nothing, computes nothing, and holds no locks. Any large excursion it
 * records is time during which THE WHOLE JVM WAS STOPPED - because there is nothing
 * else that could have delayed it.
 *
 * The numbers it prints are observations from YOUR run. They are not measurements
 * of the JVM in general and must not be quoted as such.
 */
public final class JvmJitterDetector implements Runnable {

    private final long thresholdMillis;

    public JvmJitterDetector(long thresholdMillis) { this.thresholdMillis = thresholdMillis; }

    @Override public void run() {
        long previous = System.nanoTime();
        while (!Thread.currentThread().isInterrupted()) {
            try { Thread.sleep(1); } catch (InterruptedException e) { return; }
            long now = System.nanoTime();
            long excursionMillis = ((now - previous) / 1_000_000) - 1;
            if (excursionMillis >= thresholdMillis) {
                System.out.println("JVM-WIDE STALL: " + excursionMillis
                    + " ms at uptime " + (now / 1_000_000_000) + "s");
            }
            previous = now;
        }
    }

    /** Wire this into orderflow behind a profile so it never runs in production. */
    public static Thread startDaemon(long thresholdMillis) {
        Thread t = new Thread(new JvmJitterDetector(thresholdMillis), "jvm-jitter-detector");
        t.setDaemon(true);
        t.start();
        return t;
    }
}
```

Run it inside the straggler program, or wire it into `orderflow` behind a Spring
profile, and correlate its output timestamps against the safepoint log.

**WHAT TO LOOK FOR:** stall excursions in the detector that line up with large reaching
times in the safepoint log.

| What you see | What it means |
|---|---|
| Detector excursions match safepoint totals in time and magnitude | **Cause and effect, joined.** A thread doing literally nothing was stopped for that long. This is the demonstration that convinces a sceptical room. |
| Detector excursions with no matching safepoint event | Not a safepoint. OS scheduling, cgroup throttling, swap, or steal. Look outside the JVM. |
| Detector excursions match safepoint **at** times, not reaching times | The VM operation itself is slow. If it is GC, Topic 71. If it is a thread dump with thousands of threads, Topic 98. |
| No excursions at all under load | Good news, and worth recording as a baseline fact about the service: TTSP is not currently a problem here. Re-check after any change that adds CPU-bound in-process work. |

> **Caveat on this detector, stated honestly:** `Thread.sleep(1)` has its own OS-level
> jitter, and on a loaded or virtualised machine you will see excursions that have
> nothing to do with safepoints. It is a *correlation* tool, not a measurement
> instrument. The safepoint log is the authority; the detector tells you when to look
> at it.

---

## Failure drill

**Mandatory.** Do not read the "how to read it" table until you have produced the result
yourself and written down what you saw.

### The assignment, restated from the master plan

> Run a long counted `int` loop over a large array in one thread while another thread
> triggers frequent GCs. Capture `-Xlog:safepoint*` and identify TTSP dominating the
> pause. Change the loop counter to `long` and re-measure.

**Read the last sentence carefully.** The assignment says *re-measure*, not *fix*. On a
modern JDK the `long` change may make no difference at all. **Finding that out and being
able to explain it is a better outcome than reproducing folklore.**

### Step 0 — establish the control and learn your log format

```bash
java -Xlog:safepoint -version          # learn the field names your JDK prints
java -XX:+PrintFlagsFinal -version | grep -iE "UseCountedLoopSafepoints|LoopStripMiningIter"
```

Write both down. Every command below must be adapted to them.

Then run the Topic 65 baseline on `orderflow`, unchanged, with **both** logs on the
same clock:

```bash
docker compose up -d
java -Xms1200m -Xmx1200m -XX:+UseG1GC \
  -Xlog:gc*:file=/var/log/orderflow/gc-control.log:time,uptime,level,tags \
  -Xlog:safepoint*:file=/var/log/orderflow/sp-control.log:time,uptime,level,tags \
  -jar orderflow.jar

k6 run --out json=logs/control.json load/baseline.js
```

Record: p50/p95/p99/p999 per endpoint, throughput, error rate, **maximum reaching
time**, **maximum at time**, and the safepoint count broken down by operation name.

**If the percentiles are not within ±10% of `/docs/java/baselines/`, stop.** The Topic
65 gate rule applies.

### Step 1 — build the straggler, standalone first

Run `SafepointStraggler` from Example 1 in all four configurations (A/B/C/D). Record the
maximum reaching time for each. **Do this standalone before touching `orderflow`,** so
you know the mechanism works on your JDK before you introduce it into a system with
many other variables.

If configuration B (counted-loop safepoints off) does not show a large reaching time,
**stop and investigate before proceeding.** Raise the array size until one pass takes a
long time. If it still does not reproduce, note it and continue to Step 2 anyway with
the flag explicitly disabled — the `orderflow` version has more competing threads and
more frequent safepoint requests, and often shows it more clearly.

### Step 2 — put the straggler inside `orderflow`

```java
package com.orderflow.lab;

import java.util.concurrent.atomic.AtomicLong;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

/**
 * DRILL CODE. Never merge this.
 *
 * Simulates the revenue-reconciliation job from Example 2: a counted int loop over a
 * large primitive array, running on a schedule, in the same JVM as the API.
 */
@Component
public class ReconciliationStraggler {

    /** Sized to resemble a day of order lines from the Topic 65 dataset. */
    private static final int LINES = 50_000_000;

    private final long[] amountsMinor = new long[LINES];
    private final AtomicLong lastTotal = new AtomicLong();

    public ReconciliationStraggler() {
        for (int i = 0; i < LINES; i++) {
            amountsMinor[i] = (i % 997) * 13L;
        }
    }

    @Scheduled(fixedDelay = 20_000)
    public void reconcile() {
        long total = 0;
        // COUNTED INT LOOP. This is the drill's subject.
        for (int i = 0; i < amountsMinor.length; i++) {
            total += amountsMinor[i];
        }
        lastTotal.set(total);
    }
}
```

> Note the heap implication: a `long[]` of 50 million entries is roughly 400 MB. On a
> 1200 MB heap that is a large live set on its own, which will *also* affect GC pause
> durations. **That is a confound and you must account for it** — which is why you
> compare the *reaching* time, not the total, and why the `at` time is expected to grow
> too. If you cannot afford the memory, shrink the array and raise the loop's repeat
> count instead.

### Step 3 — run it under load, with both logs

```bash
java -Xms1200m -Xmx1200m -XX:+UseG1GC -XX:-UseCountedLoopSafepoints \
  -XX:+SafepointTimeout -XX:SafepointTimeoutDelay=200 \
  -Xlog:gc*:file=/var/log/orderflow/gc-drill.log:time,uptime,level,tags:filecount=5,filesize=50m \
  -Xlog:safepoint*:file=/var/log/orderflow/sp-drill.log:time,uptime,level,tags:filecount=5,filesize=50m \
  -jar orderflow.jar 2>&1 | tee logs/stdout-drill.log

# Run the UNCHANGED Topic 65 load profile. Do not add an endpoint for the job -
# the whole point is that a background thread degrades endpoints it never touches.
k6 run --out json=logs/drill.json load/baseline.js
```

### Step 4 — what to capture

Write these down **before** reading any interpretation:

1. Maximum **reaching** time in `sp-drill.log`, and its safepoint operation name.
2. Maximum **at** time in `sp-drill.log`.
3. **The ratio** of the two for the worst events.
4. The longest pause reported in `gc-drill.log` for the same timestamps.
5. Whether `SafepointTimeout` named the scheduler thread in `stdout-drill.log`.
6. p999 for **`GET /products`** — the endpoint you did not touch — versus control.
7. The 20-second periodicity: do the p999 spikes match `fixedDelay`?
8. Safepoint counts by operation name, drill versus control.

```bash
# Adapt these greps to YOUR JDK's field names, learned in Step 0.
grep -o 'Reaching safepoint: [0-9]*' /var/log/orderflow/sp-drill.log | grep -o '[0-9]*' | sort -n | tail -5
grep -o 'At safepoint: [0-9]*'       /var/log/orderflow/sp-drill.log | grep -o '[0-9]*' | sort -n | tail -5
grep -oE 'Safepoint "[A-Za-z]+"'     /var/log/orderflow/sp-drill.log | sort | uniq -c | sort -rn
grep -i -A5 'safepoint' logs/stdout-drill.log | head -40
```

### Step 5 — how to read it

| What you see | What it means |
|---|---|
| Maximum reaching time is far larger than maximum at time, and `SafepointTimeout` names the scheduler thread | **The drill has fired.** Write the sentence: "the stall at uptime T was a *N* ms time-to-safepoint caused by the reconciliation thread's counted loop; the GC operation itself took *M* ms, and only *M* appears in the GC log." |
| p999 for `GET /products` degrades on a 20-second cadence | **The most important observation in the drill.** A background thread you never called degraded an endpoint it never touched, through a mechanism absent from both endpoints' metrics. |
| The GC log's longest pause is much smaller than the observed stall | **This is the entire topic in one comparison.** The GC log is not lying; it is answering a narrower question than you asked. |
| Reaching and at times are both large | The 400 MB array grew your live set, so GC pauses grew too (Topic 71: Object Copy scales with the live set). **Both effects are real.** Separate them by shrinking the array and repeating the loop more often. |
| Reaching times are unremarkable with defaults, large with `-XX:-UseCountedLoopSafepoints` | Your JDK's defaults protect you, and you have proven both the mechanism and the mitigation. **Record the default value** — it is the reason your production service may be fine, and the reason a JDK downgrade or an unusual flag set could break it. |
| No difference in any configuration | Check that the job is running (log the total), that the array is genuinely large, and that `jcmd VM.flags -all` shows the flag you passed. If it still does not reproduce, the honest conclusion is "on this JDK and this shape, the poll is present" — and that is a finding, not a failure. |
| The whole service is slow, not spiky | You have overwhelmed the container. A 400 MB array on a 1200 MB heap plus load may simply not fit. Shrink the array. |
| `SafepointTimeout` names a Hibernate or JDBC thread instead | Excellent, and follow it. A driver doing a large `arraycopy` on a big result set is the same mechanism in code you did not write, and it is a more realistic production finding than the synthetic job. |

### Step 6 — the `long` counter, re-measured

Change the loop counter to `long` and re-run **with identical flags**:

```java
        long total = 0;
        for (long i = 0; i < amountsMinor.length; i++) {
            total += amountsMinor[(int) i];
        }
```

Capture the same eight items.

| What you see | What it means |
|---|---|
| Reaching times fall substantially with the `long` counter | The classic behaviour holds on your JDK for this shape. Note the JDK version alongside the finding — it is a version-specific observation, not a law. |
| No difference | **Modern C2 handles long counted loops.** The folklore is dated on your runtime. **This is the more valuable result**, and being able to say it in an interview — with your own evidence — puts you ahead of people quoting a 2015 blog post. |
| The `long` version is *slower* overall | Expected and unrelated to safepoints: the `(int) i` cast and the wider induction variable can inhibit optimisations such as range-check elimination and vectorisation. **Note that the "fix" has a throughput cost**, which is exactly why it is not a good fix. |
| Reaching times get worse | Investigate rather than dismiss. Look at whether the loop is still being transformed the same way, and whether the array traversal pattern changed. |

### Step 7 — fix it properly and compare

Apply, in order, and measure each:

**Fix A — move the work out of the JVM.** Run the job as a separate process against the
same database. Re-run the load profile. **Expect the reaching times to return to
control levels and `GET /products` p999 to return to baseline.**

**Fix B — do it in SQL.** `SELECT SUM(amount_minor) FROM order_line WHERE ...`. Re-run.

**Fix C — chunk the loop through a method boundary.** Re-run **and verify with the
safepoint log**, because inlining may fuse the loops back together and undo it.

**Fix D — `-XX:+UseCountedLoopSafepoints -XX:LoopStripMiningIter=1000`.** Re-run, and
**also measure throughput against the Topic 65 baseline**, because this is a global
change affecting every hot loop in the service.

Produce a table comparing: maximum reaching time, maximum at time, `GET /products`
p999, throughput, and lines of application code changed.

### What the fix proves

- **Control:** baseline percentiles, small reaching times.
- **Drill:** large reaching times, degraded p999 on an endpoint you never touched, a GC
  log that explains none of it.
- **Fix A/B:** the problem disappears with no JVM configuration change at all.
- **Fix C:** works only if verified — inlining can silently undo it.
- **Fix D:** bounds the damage globally, at a throughput cost you can now state.

Carry three sentences out of this drill:

> *GC pause and stop-the-world duration are different numbers, and only one of them is
> in the GC log.*

> *One thread that does not poll can stop every other thread, and none of them will
> appear busy while it happens.*

> *The best fix for a safepoint problem is usually not a JVM flag. It is not running
> that work in that JVM.*

---

## Measurement

### The instrument for this topic

**`-Xlog:safepoint*`, correlated with the request latency histogram on the same clock.**
Nothing else answers the question.

Not a GC log — it omits the number you need. Not a CPU profile — during the stall
almost every thread is parked and the flame graph is nearly empty. Not a microbenchmark
— TTSP is a *whole-JVM, multi-thread* property that a single-threaded benchmark cannot
produce. Not an APM dashboard — unless it specifically exposes safepoint statistics,
which most do not.

The measurement protocol:

1. **Both logs, same decorators, same clock.** `time,uptime,level,tags` on the GC log
   and the safepoint log, so you can join them.
2. **Compare reaching against at**, per event, for the worst events. The ratio is the
   diagnosis.
3. **Read the operation name.** GC, thread dump, deoptimization and class redefinition
   are four different problems with four different owners.
4. **Add `-XX:+SafepointTimeout -XX:SafepointTimeoutDelay=<ms>`** to name the straggler.
   Set the delay a little above your normal reaching time so it only fires on outliers.
5. **Correlate with the load generator's timestamps.** A stall that does not coincide
   with a safepoint is not a safepoint problem.
6. **Change one thing at a time**, re-run the identical load profile, and compare
   against the recorded Topic 65 baseline.
7. **Attach `jcmd <pid> VM.flags -all`** to every result, because `UseCountedLoopSafepoints`
   and `LoopStripMiningIter` change the answer.

### Why a naive `System.nanoTime()` loop is WRONG here

You will be tempted to write this:

```java
// DO NOT DO THIS. It cannot measure what you are trying to measure.
long start = System.nanoTime();
for (int i = 0; i < data.length; i++) {
    total += data[i];
}
System.out.println((System.nanoTime() - start) / 1_000_000 + " ms for the loop");
```

Four reasons the general technique lies, all of which apply here:

1. **Dead-code elimination.** If `total` is not observed, C2 can delete the loop
   entirely. You measure nothing. (Topic 75.)
2. **Constant folding.** A known bound and a known array can give C2 enormous freedom.
3. **On-stack replacement.** The loop starts interpreted, is compiled while running, and
   is swapped mid-flight. Your number blends interpreter, C1 and C2 execution in
   proportions set by the loop count you happened to choose — **and note that the
   interpreted portion polls frequently, so an OSR-blended measurement has completely
   different safepoint behaviour from steady-state compiled code.** (Topic 74.)
4. **Cold JIT.** The first iterations are interpreted, which is precisely the state in
   which the bug does not occur.

And three reasons specific to **this** topic, which are the decisive ones:

5. **You are measuring the wrong thread.** TTSP hurts *other* threads. The straggler
   itself experiences no stall at all — it is the one everybody is waiting for. A timer
   inside the loop measures the loop, not the damage.
6. **You need a safepoint to be requested during the loop.** A single-threaded
   benchmark with no allocation pressure may never request one. **No safepoint, no
   symptom** — you would conclude the problem does not exist.
7. **The quantity you want has a name and a log.** Reaching time is reported directly
   by the JVM. Reconstructing it from wall-clock arithmetic is strictly worse than
   reading it.

**Topic 77 is the full treatment of JMH. Do not write a benchmark you intend to act on
until you have read it.**

### Where JMH *is* the right tool here

One narrow question: **what does adding safepoint polls cost in throughput?** That is a
per-operation cost, which is what JMH measures well. It is the number you need to decide
whether `-XX:+UseCountedLoopSafepoints` is an acceptable global fix.

```java
package com.orderflow.bench;

import org.openjdk.jmh.annotations.*;
import org.openjdk.jmh.infra.Blackhole;
import java.util.concurrent.TimeUnit;

/**
 * The question: what does counted-loop safepoint polling cost this loop shape?
 *
 * The FLAG must vary across JVM invocations (-jvmArgs), never as a @Param, because
 * it is a JVM-wide compilation policy.
 */
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.MICROSECONDS)
@Warmup(iterations = 5, time = 1)
@Measurement(iterations = 10, time = 1)
@Fork(3)
@State(Scope.Benchmark)
public class CountedLoopBenchmark {

    @Param({"1000000", "50000000"})
    public int length;

    private long[] amountsMinor;

    @Setup(Level.Trial)
    public void setUp() {
        amountsMinor = new long[length];
        for (int i = 0; i < length; i++) {
            amountsMinor[i] = (i % 997) * 13L;
        }
    }

    @Benchmark
    public long sumWithIntCounter() {
        long total = 0;
        for (int i = 0; i < amountsMinor.length; i++) {
            total += amountsMinor[i];
        }
        return total;                       // returned, so not eliminated
    }

    @Benchmark
    public long sumWithLongCounter() {
        long total = 0;
        for (long i = 0; i < amountsMinor.length; i++) {
            total += amountsMinor[(int) i];
        }
        return total;
    }
}
```

```bash
java -jar target/benchmarks.jar CountedLoopBenchmark \
  -jvmArgs "-XX:-UseCountedLoopSafepoints" -rf json -rff no-poll.json
java -jar target/benchmarks.jar CountedLoopBenchmark \
  -jvmArgs "-XX:+UseCountedLoopSafepoints -XX:LoopStripMiningIter=1000" -rf json -rff strip-1000.json
java -jar target/benchmarks.jar CountedLoopBenchmark \
  -jvmArgs "-XX:+UseCountedLoopSafepoints -XX:LoopStripMiningIter=100" -rf json -rff strip-100.json
```

| Annotation / flag | What it defends against |
|---|---|
| `@State(Scope.Benchmark)` | Array held in a JMH-controlled field, so C2 cannot constant-fold it away. |
| Returning `total` | Defeats dead-code elimination without needing a `Blackhole`. |
| `@Warmup(iterations = 5)` | Reaches steady-state compiled code. Without it you measure OSR and the interpreter — which have *different safepoint behaviour*, so the benchmark would answer a different question. |
| `@Fork(3)` | Separate JVMs; exposes run-to-run variance and defeats profile pollution between the two `@Benchmark` methods (Topic 74). |
| `@Param` | Separate benchmarks per size, so one call site does not see two shapes. |
| `-jvmArgs` | **The safepoint policy is JVM-wide. It must vary across invocations, never inside one.** |

**WHAT TO LOOK FOR:** the confidence intervals. If they overlap, the polling cost is
below your noise floor for this shape — which is itself a strong argument for enabling
it. Report the intervals, never the point estimate.

**And state the caveat:** this measures the *cost* of the fix. It says nothing about
the *benefit*, which only the safepoint log under real multi-threaded load can show.
**Cost from JMH, benefit from the load test. Two instruments, two questions.**

### The numbers to track continuously in production

| Number | Where from | Why |
|---|---|---|
| **Maximum reaching time per interval** | `-Xlog:safepoint*`, parsed | **The number nobody tracks and everybody needs.** A rising trend is a code change nobody attributed. |
| Maximum at time per interval | Same log | Separates VM-operation cost from arrival cost. |
| Safepoint count by operation name | Same log | A change in the mix means a change in what is stopping you — an agent, a profiler, a deoptimization storm. |
| Safepoint frequency | Same log | Very frequent safepoints multiply even a small reaching time into real latency. |
| Total stop-the-world time as a share of wall clock | Reaching + at, summed | The honest "how much of the time is my JVM frozen" figure. |
| cgroup throttling counters | `/sys/fs/cgroup/cpu.stat` | The non-JVM cause that mimics this perfectly. Topic 82. |

Topic 118 covers exporting these without blowing up metric cardinality. Note that most
APM tools do **not** expose reaching time; you will likely have to parse the log
yourself, and doing so is a genuinely differentiating piece of platform work.

---

## Practice exercises

### 1 — easy: build your safepoint fact sheet

For your JDK, and for the exact production base image:

1. Print the safepoint log format: `java -Xlog:safepoint -version` and
   `java -Xlog:safepoint*=debug -version`. Record the exact field names.
2. Print `UseCountedLoopSafepoints` and `LoopStripMiningIter` under Serial, Parallel,
   G1 and ZGC. Record any differences.
3. Print `GuaranteedSafepointInterval` and note whether it requires
   `-XX:+UnlockDiagnosticVMOptions`.
4. Run a trivial program and take 20 thread dumps against it while logging safepoints.
   Count safepoint operations by name.

Produce one fact sheet. Then answer in one sentence each:

- Which safepoint operations in your list are **not** garbage collection?
- Your monitoring agent takes a thread dump every 10 seconds. Using only your fact
  sheet, what is the maximum stop-the-world cost per minute that policy could impose,
  and what additional number would you need to compute the actual cost?
- Why is `jcmd <pid> VM.flags -all` more trustworthy than the deployment manifest for
  `UseCountedLoopSafepoints`?

### 2 — medium: the audit (combines Topics 01, 11, 18, 25, 68, 70, 71, 72)

This class runs inside the `orderflow` JVM. Find **six** defects. Three are
safepoint/TTSP defects from this topic; three are from earlier topics. For each: name
the topic, state the **observable** symptom in a log or a latency percentile, and write
the fix.

```java
package com.orderflow.reporting;

import java.util.*;
import java.util.stream.Collectors;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

@Component
public class DailySettlementJob {

    private static final Map<Long, List<Long>> SETTLED = new HashMap<>();

    private final OrderLineRepository lines;

    public DailySettlementJob(OrderLineRepository lines) { this.lines = lines; }

    @Scheduled(fixedRate = 30_000)
    public void settle() {
        long[] amounts = lines.allAmountsMinorToday();        // several million entries

        long gross = 0;
        for (int i = 0; i < amounts.length; i++) {
            gross += amounts[i];
        }

        byte[] exportBuffer = new byte[8 * 1024 * 1024];
        System.arraycopy(encode(amounts), 0, exportBuffer, 0, amounts.length * 8);

        Map<Long, Long> byMerchant = Arrays.stream(amounts).boxed()
            .parallel()
            .collect(Collectors.groupingBy(a -> a % 100, Collectors.summingLong(a -> a)));

        String report = "";
        for (Map.Entry<Long, Long> e : byMerchant.entrySet()) {
            report = report + e.getKey() + ":" + e.getValue() + "\n";
        }

        SETTLED.computeIfAbsent(System.currentTimeMillis(), k -> new ArrayList<>())
               .add(gross);
    }
}
```

Hints, in the order to think about them: one is the counted-loop TTSP defect from this
topic; one is a very large uninterruptible intrinsic that is *also* a TTSP defect; one
allocates an object that is humongous on a 1200 MB heap (Topic 71) **and** must be
zeroed, which is a third TTSP defect; one is a Topic 25 problem that puts blocking-ish
work on the common ForkJoinPool; one is a Topic 01 boxing problem that inflates
allocation rate; one is a Topic 18 quadratic string problem; and one is a Topic 79
unbounded-static-map retention shape.

(Yes, that is more than six. Find all of them and rank them by expected impact on
p999 — the ranking is the exercise.)

For the TTSP defects specifically: state which `-Xlog:safepoint*` field would show each
one, and what `-XX:+SafepointTimeout` would print.

### 3 — hard: production simulation — separate the two numbers under real load

**Part A — reproduce and record.** Run the Topic 65 baseline with GC and safepoint
logging on the same clock. Confirm ±10%. Record per endpoint: p50/p95/p99/p999,
throughput, error rate, maximum reaching time, maximum at time, and safepoint counts by
operation name. Three runs; state your noise floor.

**Part B — introduce three independent stall sources, one at a time.** Restore the
baseline between each and confirm ±10%.

1. The counted-loop reconciliation job from the failure drill.
2. A "thread monitor" that calls `Thread.getAllStackTraces()` every two seconds.
3. Container CPU throttling: re-run with `--cpus=0.6` and no other change.

For each, record the same numbers **plus** the cgroup throttling counters.

**Part C — the blind test.** This is the exercise that actually teaches the topic. Hand
the three pairs of logs (GC + safepoint) to someone else — or to yourself in a week —
**with the labels removed**. For each pair, write down:

- Whether the dominant cost is reaching time or at time.
- The safepoint operation name responsible.
- Whether the cause is inside the JVM or outside it.
- The single command you would run next to confirm.
- The fix, and its cost.

Then check against your Part B notes. **Any pair you cannot diagnose from the logs alone
is a gap in your reading skill, not your tuning skill.**

**Part D — the collector red herring.** Take source (1) and re-run it under ZGC
(generational — see Topic 72's version note). Record the same numbers.

**WHAT TO LOOK FOR:** that the GC log improves dramatically while p999 does not move.
Write one paragraph explaining, to a colleague who proposed the ZGC migration, exactly
why it did not help — in terms of the safepoint protocol, not in terms of collectors.

**Part E — the cost of the global fix.** Apply `-XX:+UseCountedLoopSafepoints
-XX:LoopStripMiningIter=1000` and measure. Then apply the *code* fix (move the job out
of the JVM) and measure. Produce a table comparing: maximum reaching time, `GET
/products` p999, throughput, and lines of application code changed.

Then answer: under what circumstances would you ship the flag instead of the code
change? Name at least one circumstance where that is the correct engineering decision,
and be honest about it.

**Part F — argue against yourself.** You will conclude the code fix is better. Make the
strongest possible case for the flag on `orderflow` specifically. Then state what would
have to be true about the team, the release process or the incident timeline for that
case to win.

---

## Interview questions

### Q1 — "Our p99 has 2-second spikes. The GC log shows nothing over 20 ms. Diagnose it."

**MID-LEVEL answer:** "If the GC log is clean then it's not GC. I'd look at the
database, check for slow queries, look at the connection pool, and check the network to
our downstream services."

**SENIOR answer:** "The mid answer's conclusion might be right, but it skips the number
the GC log does not contain, and that number is the most common cause of exactly this
shape.

**A GC pause and a stop-the-world pause are different durations.** The number in a
`Pause Young` line is time spent doing GC work *at* the safepoint. It excludes
**time-to-safepoint** — the interval between the JVM requesting a safepoint and the
last application thread arriving at one. A reported 5 ms pause can sit inside a 900 ms
stop-the-world event, and the GC log will never mention the other 895 ms.

So my first command is:

```
-Xlog:safepoint*:file=safepoint.log:time,uptime,level,tags
```

and I compare reaching time against at time for the worst events. If reaching
dominates, GC is exonerated in one step and I have saved a week.

**Then I read the operation name field**, because not every safepoint is a collection.
If our stalls are thread-dump operations, the cause is a monitoring agent polling
`Thread.getAllStackTraces()`. If they are class-redefinition operations, it is an APM
agent instrumenting classes at runtime. Those are three different owners.

**Then I name the culprit thread** with `-XX:+SafepointTimeout
-XX:SafepointTimeoutDelay=500`. The JVM prints which thread it waited for. That takes
me from 'we have stalls' to 'this thread, this code' without guessing.

**The classic causes of a long reaching time**, in the order I'd check: a counted `int`
loop over a large array with its poll elided; a large `System.arraycopy` or `Arrays.fill`
or a huge array being zeroed, which are intrinsics with no poll inside; a JNI critical
section; and — the ones that are not our code's fault at all — page faults, cgroup CPU
throttling, and hypervisor steal time.

**Two corroborating signals** I'd want. First, the stalls should hit requests already
in flight on threads doing no related work, across unrelated endpoints, at the same
instant — that global-and-correlated shape is the fingerprint. Second, a CPU profile of
the stall window should be nearly *empty*, because almost every thread is parked. A
nearly-empty profile during a stall is very strong evidence for a safepoint problem and
almost nobody looks for it.

Only after all that would I go to the database — and I'd hold coordinated omission in
the load test in mind the whole time, because our recorded p99 might have been wrong
from the start."

**What separates them:** the mid answer treats the GC log as complete. The senior answer
knows it is *structurally incomplete*, names the missing number, gives the exact flag,
knows that safepoints are requested by non-GC operations and reads the operation name,
knows the flag that names the straggler, has a ranked list of TTSP causes that includes
non-JVM ones, and offers a corroborating signal (the empty CPU profile) that
demonstrates the model is real rather than memorised.

**Interviewer's follow-up:** *"What if the reaching time is fine and the at time is
large?"* — Then the VM operation itself is slow, which is a different problem. If it is
a GC operation, that is a live-set and collector question — Object Copy scales with
survivors. If it is a thread dump in a JVM with thousands of threads, the at-safepoint
work scales with thread count, and the fix is fewer threads or fewer dumps. Either way
I now know which half to work on, which is the entire value of the split.

---

### Q2 — "Why can one thread stop all the others? Walk me through the mechanism."

**MID-LEVEL answer:** "The JVM needs all threads stopped for GC, so it signals them and
waits. If one thread is busy in a loop, the others wait for it."

**SENIOR answer:** "The reason it *has* to wait is the interesting part, so I'll start
there.

A moving collector must find and update every reference to every object it moves.
References live in registers and stack slots. The JVM cannot tell a reference from a
`long` that happens to look like an address — unless the compiler recorded, for that
exact program point, which slots hold references. That record is an **oop map**, and
generating one for every instruction would be prohibitive in code-cache space and would
constrain register allocation everywhere. So the compiler generates them only at chosen
points: **safepoints**. A thread can only be stopped where an oop map exists.

**The mechanism is a polling page.** Compiled code emits a load from a reserved page at
each poll site. Normally the page is readable, the load costs a cache hit, and there is
no branch to mispredict. To request a safepoint, the JVM removes read permission from
the page with `mprotect`. The next poll faults, a signal handler recognises the address,
and parks the thread. The fast path is one load; the slow path is delivered by the MMU.
Since thread-local handshakes in JDK 10 each thread has its own poll address in
thread-local storage, so the JVM can also stop a single thread without stopping
everybody — which moved a lot of work out of global safepoints.

**Where the polls are is the operational fact.** Compiled code polls at method returns
and at the back-edges of *non-counted* loops. It does **not** poll inside a counted
`int` loop, because C2 knows a counted loop terminates and removes the poll as an
optimisation. That reasoning is correct about termination and silent about *time*: a
loop over five million elements terminates, and takes a long time doing it.

**So one thread in such a loop delays everybody.** All the other threads have already
parked. They are idle, burning no CPU, waiting. That is why a CPU profile during the
stall is nearly empty and why the stall hits endpoints with no relationship to the loop.

**The modern mitigation is loop strip mining**: C2 splits a counted loop into an outer
loop carrying a poll and an inner loop of at most `LoopStripMiningIter` iterations. That
bounds TTSP — to the time taken by that many iterations, which is a bound and not
necessarily a small one. Whether it is on by default depends on the JDK and the
collector, so I check `jcmd VM.flags -all` rather than assume.

And the old advice — change the counter to `long` to restore the poll — was correct for
a long time and is now version-dependent, because C2 gained a long-counted-loop
transformation. I'd measure it with `-Xlog:safepoint*` before and after rather than
quoting the folklore."

**What separates them:** starting from *why* an arbitrary stop is impossible (oop maps),
naming the polling-page and page-protection mechanism, knowing thread-local handshakes
changed the picture, naming the exact poll locations and the counted-loop exception,
knowing about strip mining as the modern mitigation, and treating the `long`-counter
advice as version-dependent rather than as a law. Mentioning the empty CPU profile as a
consequence shows the model is joined up.

**Interviewer's follow-up:** *"Why not just poll more often and bound TTSP tightly?"* —
Because each poll site needs an oop map, which costs code-cache space, and because the
compiler must keep references in identifiable locations across a poll, which constrains
register allocation and reordering in exactly the hottest loops. The JVM is trading peak
throughput in tight loops against bounded time-to-safepoint. Strip mining is the
compromise, and `LoopStripMiningIter` is the dial on that trade.

---

### Q3 — "You're on ZGC with sub-millisecond pauses and you still have 500 ms stalls. What now?"

**MID-LEVEL answer:** "If ZGC's pauses are that short, the JVM isn't the problem. I'd
look outside it — the database, the network, or the infrastructure."

**SENIOR answer:** "It's the right instinct applied one step too early, because there
are two JVM-level causes that a clean ZGC log actively hides — and 'clean ZGC log' is
*specifically weak* evidence for that reason.

**First: time-to-safepoint.** ZGC's pause is short *at* the safepoint. The safepoint
protocol is unchanged: every thread must still arrive, and the JVM still waits for the
last one. A counted loop, a large `arraycopy`, a huge array being zeroed, or a swapped-
out page delays everybody exactly as it would under G1. The GC log reports the tiny part
it measures. `-Xlog:safepoint*` reports the reaching time separately, and
`-XX:+SafepointTimeout` names the thread. That's my first check and it takes two
minutes.

**Second: allocation stalls.** ZGC is concurrent, so it races the allocation rate. If it
loses, application threads block waiting for free memory. That is not a pause and never
appears as one — but requests stall. `grep -i 'allocation stall'` on the GC log finds it
immediately. The fix is headroom, CPU, or a lower allocation rate; there is no pause-
goal flag to reach for, because ZGC doesn't have one.

**Third, and it's not the JVM's fault at all: cgroup CPU throttling.** If the container
exhausts its quota, every thread is descheduled for the rest of the period. From inside
the application that is indistinguishable from a stop-the-world pause, and it appears in
no JVM log whatsoever. `cat /sys/fs/cgroup/cpu.stat` gives `nr_throttled` and
`throttled_usec`. And note that ZGC's own concurrent GC threads consume that quota, so
migrating to ZGC in a tight container can *cause* this — collector-adjacent even though
the JVM is innocent.

Only after those three would I go outside. And I'd add a note for the team: the
migration to ZGC may have been a pure loss here — we're paying the load barrier's
throughput cost on every reference load for a benefit we didn't receive, because pause
duration was never our problem."

**What separates them:** knowing that a clean GC log under a concurrent collector is
weak evidence rather than strong evidence, naming both JVM-level hidden causes with
their exact greps, naming a non-JVM cause that mimics a pause perfectly and appears in
no log, and closing the loop on whether the collector migration was worth its cost. The
cgroup-throttling point marks someone who has debugged this in Kubernetes.

**Interviewer's follow-up:** *"How would you confirm cgroup throttling caused a specific
spike?"* — Sample `nr_throttled` and `throttled_usec` on a short interval and correlate
the deltas against the spike timestamps. The corroborating signal is that the safepoint
log shows *many* threads with long reaching times simultaneously, rather than one
straggler — because nobody was scheduled, not because one thread was in a loop. One
straggler means a loop; everybody straggling means the scheduler.

---

### Q4 — "Is `jcmd Thread.print` safe to run on a production JVM?"

**MID-LEVEL answer:** "Yes, it's a read-only diagnostic. It just prints thread stacks.
It might be a bit slow on a big JVM but it doesn't change anything."

**SENIOR answer:** "It's safe in the sense that it changes no state, and it is
absolutely *not* free — it requires a **global safepoint**.

Two costs. First, you pay **time-to-safepoint**: every thread must reach a poll point,
and if one is in a counted loop or a big `arraycopy`, everybody waits for it. That bill
is exactly the same one GC pays, and it is a property of your application, so a JVM with
a TTSP problem has a thread-dump problem too. Second, you pay the **at-safepoint** cost
of walking every thread's stack, which **scales with thread count**. On a
thread-per-request service with hundreds of threads and deep Spring and Hibernate
stacks, that is not trivial.

So: one dump during an incident, absolutely, take it — the information is worth the
stall. **A dump every ten seconds from a monitoring agent is a self-inflicted latency
source**, and it is one I have seen mistaken for GC, because it produces regular,
correlated, cross-endpoint stalls that no GC log explains. The tell is in the safepoint
log's operation-name field: the safepoints are thread-dump operations, not collections,
and they arrive on a suspiciously round cadence matching a polling interval.

The generalisation worth stating: **several things people think of as read-only
observability are stop-the-world operations.** Thread dumps, heap dumps, and any
JVMTI-based stack sampling. And an APM agent that redefines classes at runtime triggers
safepoints too, which is one honest reason 'the service got slower after we added the
agent' is a real phenomenon rather than a superstition.

The safe alternatives for continuous use: JFR, whose overhead profile is designed to be
left on in production, and async-profiler, which uses `AsyncGetCallTrace` and perf
events rather than safepoints — which also makes it free of the safepoint sampling bias
that makes many Java profilers lie about where time goes."

**What separates them:** knowing a thread dump is a global safepoint at all, splitting
the cost into TTSP and at-safepoint work, knowing the at-safepoint cost scales with
thread count, connecting periodic dumps to a specific misdiagnosis, generalising to
other "read-only" operations including agent class redefinition, and naming
non-safepoint-based alternatives with the reason they are better.

**Interviewer's follow-up:** *"How would you prove an agent was causing your stalls?"* —
Count safepoints by operation name in `-Xlog:safepoint*` and compare the cadence against
the agent's polling interval. Then the decisive test: run the identical load profile
with the agent removed, and compare maximum reaching time, safepoint counts by
operation, and p999. One variable, one measurement, three numbers.

---

### Q5 — "A team wants to set `-XX:+UseCountedLoopSafepoints` globally. Do you approve it?"

**MID-LEVEL answer:** "It bounds time-to-safepoint, which sounds good, but it adds
overhead to loops. I'd test it and see if the overhead is acceptable."

**SENIOR answer:** "I'd approve it only after three questions, and I'd be suspicious of
the motivation behind it.

**Question one: is it already on?** On a modern JDK with a low-pause collector I believe
counted-loop safepoints are enabled by default, and I would check with `jcmd VM.flags
-all` rather than assume. If it is already on, the team is proposing a no-op and
believes it will fix something — which means the real cause is unidentified and they
are about to declare victory over nothing. That is the worst outcome available and the
most likely one.

**Question two: what does the safepoint log say?** If the maximum reaching time is
already small, there is nothing to fix and this is a throughput cost for no benefit.
If it is large, I want the straggler named with `-XX:+SafepointTimeout` before we
change a global compilation policy to work around one piece of code.

**Question three: what is the actual fix?** This flag is a **global** change affecting
every hot loop in the process, proposed to fix — usually — one badly-placed job. If the
cause is a reconciliation loop running in the API's JVM, the right fix is to move that
job out of the JVM, or push the aggregation into the database. Those fixes remove the
problem; the flag bounds it everywhere at a cost everywhere. **A flag redistributes the
cost; not running the work removes it.**

If after all that the flag is still the right answer — say the offending loop is inside
a third-party library we cannot change — then yes, with conditions: measure the
throughput cost with JMH on the specific loop shapes that matter, measure the end-to-end
effect against the recorded load-test baseline, tune `LoopStripMiningIter` rather than
accepting the default blindly, and write a one-line justification next to the flag in
the manifest. A flag without a written justification and a measurement is technical debt
with a performance cost.

And I'd note the reverse risk: someone later 'cleans up' the flag, and the stalls come
back months after the change that removed it. That is why the justification comment
matters more than the flag."

**What separates them:** checking whether the flag is already on *first* — which
reframes the whole request — insisting the straggler be named before a global policy
change, preferring the code fix over the flag with a stated principle, defining the
conditions under which the flag is nevertheless correct, and anticipating the future
cleanup that reintroduces the bug.

**Interviewer's follow-up:** *"What if `LoopStripMiningIter` is already large — say a
few thousand?"* — Then TTSP is bounded by the time taken for that many iterations, which
for a cheap loop body is microseconds and for an expensive one — a loop body that
touches memory sparsely and misses cache every iteration — could still be
milliseconds. **A bound is not the same as a small bound.** I'd measure the reaching
time rather than reason about the iteration count, because the iteration count is not
the quantity that matters; the wall time is.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. A moving collector needs oop maps to distinguish references from integers in
   registers and stack slots. Derive from that single requirement why a thread cannot
   be stopped at an arbitrary instruction — and then explain why the JVM does not simply
   generate an oop map for every instruction.

2. The safepoint poll is a load from a page whose permissions the JVM revokes. Explain
   why this is cheaper than a conditional branch, and name one situation in which the
   page-protection approach would perform *worse* than a branch.

3. A counted `int` loop's poll may be elided because C2 knows the loop terminates.
   State precisely why "terminates" and "terminates soon" are different properties, and
   construct a loop with a small iteration count that would still produce a long TTSP.

4. During a TTSP stall, a CPU profile of the JVM is nearly empty. Explain why, and then
   explain why this makes CPU profiling the *wrong* tool for this problem and what you
   would use instead.

5. A thread blocked in a JDBC socket read does **not** delay a safepoint. A thread in a
   counted loop does. Explain the difference in terms of what the JVM knows about each
   thread's state — and then predict how a service's TTSP profile changes when it moves
   from blocking JDBC to a fully non-blocking stack.

6. Switching from G1 to ZGC does not change TTSP. Explain why, in terms of what each
   collector changes and what it does not. Then explain why the migration can make the
   problem *harder* to diagnose rather than easier.

7. You have a service where the maximum reaching time is small but safepoints occur
   very frequently. Explain how that combination can still produce a serious latency
   problem, and name the two numbers you would multiply together to quantify it.

---

## Quick reference card

### JVM flags — safepoints

| Flag | Default | What it does | Set it? |
|---|---|---|---|
| `-Xlog:safepoint*` | off | Logs every safepoint with reaching and at durations | **Yes, in production**, to a rotating file. This is the topic's instrument. |
| `-Xlog:safepoint*=debug` | off | More detail, more tags | When triaging |
| `-XX:+SafepointTimeout` | off | Report threads that fail to reach a safepoint in time | **Yes when triaging** — this is what names the culprit |
| `-XX:SafepointTimeoutDelay=<ms>` | version-dependent | The threshold for the above | Set just above your normal reaching time so it only fires on outliers |
| `-XX:+AbortVMOnSafepointTimeout` | off | Crash the VM on a slow safepoint, producing full stacks | **Never in production.** A reproducible test rig only. |
| `-XX:+UseCountedLoopSafepoints` | **version- and collector-dependent** | Emit polls in counted loops (via strip mining) | Verify the current value before proposing it. Measure the throughput cost. |
| `-XX:LoopStripMiningIter=<n>` | version-dependent | Inner-loop iterations between outer-loop polls | Only with a measurement. A bound is not necessarily a small bound. |
| `-XX:GuaranteedSafepointInterval=<ms>` | version-dependent | Force a periodic safepoint even with nothing to do | Rarely; diagnostic |
| `-XX:+UnlockDiagnosticVMOptions` | off | Required for some of the above on some builds | As needed |
| `-XX:+PrintFlagsFinal` | — | The only trustworthy source of a default | Always, when the default matters |

> **Version note, one line:** the legacy `-XX:+PrintSafepointStatistics` and
> `-XX:PrintSafepointStatisticsCount` flags were **removed** in favour of unified
> logging; if a blog post tells you to use them, the post predates JDK 9. Defaults for
> `UseCountedLoopSafepoints`, `LoopStripMiningIter`, `SafepointTimeoutDelay` and
> `GuaranteedSafepointInterval` have moved across releases and can vary by collector.
> **Settle every one of them with `java -XX:+PrintFlagsFinal -version` or
> `jcmd <pid> VM.flags -all` on your own runtime rather than trusting any document,
> including this one.**

### Diagnostic commands

```bash
# The instrument. Both logs, same decorators, same clock, rotating.
-Xlog:safepoint*:file=safepoint.log:time,uptime,level,tags:filecount=5,filesize=50m
-Xlog:gc*:file=gc.log:time,uptime,level,tags:filecount=5,filesize=50m

# Learn YOUR JDK's format before writing any grep.
java -Xlog:safepoint -version
java -Xlog:safepoint*=debug -version | head -40
java -Xlog:help | grep -i safepoint

# Name the straggler thread. The highest-value flag in this topic.
-XX:+SafepointTimeout -XX:SafepointTimeoutDelay=500

# What are the actual flag values on the running JVM?
jcmd <pid> VM.flags -all | grep -iE "CountedLoopSafepoints|LoopStripMining|SafepointTimeout|GuaranteedSafepointInterval"
java -XX:+PrintFlagsFinal -version | grep -iE "CountedLoopSafepoints|LoopStripMining"

# Triage a safepoint log you were handed (adapt field names to your JDK).
grep -oE 'Safepoint "[A-Za-z]+"' safepoint.log | sort | uniq -c | sort -rn   # by operation
grep -o 'Reaching safepoint: [0-9]*' safepoint.log | grep -o '[0-9]*' | sort -n | tail -10
grep -o 'At safepoint: [0-9]*'       safepoint.log | grep -o '[0-9]*' | sort -n | tail -10
wc -l safepoint.log                                                          # how often?

# Turn logging on at runtime without a restart.
jcmd <pid> VM.log output=/tmp/sp-live.log what='safepoint*' decorators='time,uptime,level,tags'
jcmd <pid> VM.log disable

# A thread dump - remembering that this ITSELF requests a global safepoint.
jcmd <pid> Thread.print

# The non-JVM causes that mimic this exactly.
cat /sys/fs/cgroup/cpu.stat      # nr_throttled, throttled_usec  (Topic 82)
vmstat 1 10                      # si/so = swapping, st = hypervisor steal
```

### How to read a safepoint log line — field guide

```
[<wall-clock>][<uptime>s][<level>][safepoint] Safepoint "<OperationName>", Time since last: <ns>, Reaching safepoint: <ns>, At safepoint: <ns>, Total: <ns>
```

***Illustration of the format, not captured output. Field names and units vary by JDK
release — confirm yours with `java -Xlog:safepoint -version`.***

| Position | Field | How you use it |
|---|---|---|
| 1 | wall clock | correlate with k6, the APM, and the GC log |
| 2 | uptime | intervals between safepoints |
| 3 | level | `info` for summaries, `debug` for detail |
| 4 | tag | **grep on this**, not on message text |
| 5 | **operation name** | **the diagnostic field** — GC? thread dump? deoptimization? class redefinition? |
| 6 | time since last | very small values mean safepoints are frequent; look for an agent |
| 7 | **reaching safepoint** | **TTSP. Your application's fault (or the OS's). The number nobody looks at.** |
| 8 | **at safepoint** | the VM operation's own cost. Roughly what the GC log reports for a GC. |
| 9 | total | **what your users experienced.** Reaching + at. |

Reading order when you open an unfamiliar safepoint log:

1. Count safepoints by **operation name**. If most are not GC, your problem is not GC.
2. Sort by **total** and look at the worst ten. For each, is reaching or at dominant?
3. If **reaching** dominates: application code or the OS. Name the thread with
   `SafepointTimeout`; look for counted loops, big `arraycopy`, huge array zeroing, JNI
   critical sections; then check cgroup throttling and swap.
4. If **at** dominates: the VM operation. GC (Topic 71), a thread dump with many threads
   (Topic 98), class redefinition by an agent (Topic 81).
5. If **frequency** is the problem: multiply frequency by total. A small stall a hundred
   times a second is a large stall.
6. Only now correlate with the GC log — and only to explain the `at` component.

### Gotchas checklist

- [ ] GC pause and stop-the-world duration are different numbers. Only one is logged.
- [ ] Turn `-Xlog:safepoint*` on in production, rotating, permanently.
- [ ] Learn **your** JDK's safepoint log format before writing a grep.
- [ ] Read the operation name. Not every safepoint is a collection.
- [ ] `-XX:+SafepointTimeout` names the straggler. Use it early.
- [ ] A thread dump is a global safepoint. So is a heap dump. So is class redefinition.
- [ ] A counted `int` loop may not poll. Verify `UseCountedLoopSafepoints` on your JVM.
- [ ] The `long`-counter fix is version-dependent. Measure it; do not quote folklore.
- [ ] A low-pause collector does **not** fix TTSP, and makes it harder to see.
- [ ] During a TTSP stall the CPU profile is nearly empty. That absence is evidence.
- [ ] Many threads straggling at once means the scheduler, not a loop.
- [ ] Check cgroup throttling and swap before blaming the JVM.
- [ ] Frequency times duration is the real cost. A small stall, often, is a big problem.

---

## When would I use this at work?

**1. A p999 incident where the GC log is clean and everyone is about to blame the
database.**
You run two commands — turn on `-Xlog:safepoint*` and `-XX:+SafepointTimeout` — and
within one load-test cycle you have either exonerated the JVM or named the exact thread
and code that stalled it. The alternative is a week of database investigation that finds
nothing, because the database was never involved. This is the single highest-leverage
thing in this document, and the reason is simple: almost nobody on your team knows the
number exists.

**2. Reviewing a background job that someone wants to run inside the API's JVM.**
A scheduled reconciliation, an export, a cache warmer, a report. You ask two questions:
does it contain a long loop over a large array, and does it need to be in this JVM?
Both questions take thirty seconds and both prevent a class of incident that is
notoriously hard to diagnose after the fact. **"Not in this JVM" is a design principle
you can now justify mechanically**, rather than as a preference for tidiness.

**3. Evaluating observability tooling before it goes to production.**
An APM agent, a thread-dump-based monitor, a profiler. You know that thread dumps and
class redefinition are stop-the-world operations, and that at-safepoint cost scales with
thread count. So you run the Topic 65 load profile with and without the tool, compare
maximum reaching time, safepoint counts by operation name, and p999, and you make the
overhead a **measured** number in the adoption decision rather than a vendor claim.
That is the difference between "the agent adds about 3%" as marketing and as a fact
about your service.

---

## Connected topics

**Prerequisites:**

- **01 — Boxing and the Integer cache**: allocation rate drives collection frequency,
  and every collection requests a safepoint. A high allocation rate multiplies even a
  small TTSP into a real latency problem — frequency times duration.
- **21 — Lambdas and `invokedynamic`**: lambda linkage and the resulting compilation
  activity contribute safepoint operations you did not write. The operation-name field
  is where you notice them.
- **25 — Parallel streams and the common ForkJoinPool**: a parallel stream over a large
  array runs counted loops on *several* threads at once, multiplying the chance that one
  of them is the straggler when a safepoint is requested.
- **65 — The load-testing gate**: the recorded baseline is the control. TTSP effects are
  visible in p999 and almost invisible in p50, so you need percentiles you trust.
- **68 — TLABs and promotion**: allocation drives young collections, which drive
  safepoint requests. Reducing allocation reduces how often you pay your TTSP bill even
  if you never fix the straggler.
- **70 — GC fundamentals**: cost tracks the live set. That is the `at` half of the
  stop-the-world duration; this topic is about the `reaching` half.
- **71 — G1 in depth**: G1's Trap 3 raised this topic and deferred it here. Everything
  about reading a GC log applies, plus the crucial fact that the GC log's duration field
  is only half the story.
- **72 — ZGC and Shenandoah**: low-pause collectors do **not** change the safepoint
  protocol. Read 72's Trap 4 alongside this document; a clean ZGC log is specifically
  weak evidence that the JVM is innocent.

**This unlocks:**

- **74 — JIT and tiered compilation**: safepoint polls are code C1 and C2 emit, and
  their placement is a compiler decision. Deoptimization is itself a safepoint
  operation, so a deoptimization storm shows up in this log. Counted-loop recognition,
  strip mining and OSR are all C2 concepts.
- **75 — Escape analysis and inlining**: inlining changes where method-return poll sites
  are, which is why the "chunk the loop through a method call" fix must be **verified**
  rather than assumed — C2 may inline your chunk method and fuse the loops back.
- **76 — Reading bytecode with `javap -c`**: shows what the *language* did. It does
  **not** show safepoint polls, which are emitted by the JIT, not by javac. A useful
  reminder that bytecode and machine code answer different questions.
- **77 — JMH**: the correct way to measure the *cost* of enabling counted-loop
  safepoints. Note that a single-threaded benchmark cannot measure the *benefit*, which
  needs multi-threaded load.
- **78 — Profiling**: most Java profilers sample at safepoints and are therefore
  safepoint-biased; async-profiler is not. This topic explains *why* that bias exists,
  and why a nearly-empty CPU profile during a stall is meaningful evidence.
- **79 — Heap dumps**: taking one is a stop-the-world operation. Worth knowing before
  you take a dump on a latency-sensitive production service during an incident.
- **81 — Instrumentation agents**: class redefinition requires a safepoint, and agents
  change method sizes in ways that affect inlining and therefore poll placement. The
  observer effect is real and this is one of its mechanisms.
- **82 — Containers and cgroup CPU sizing**: CPU throttling produces stalls that mimic
  safepoint pauses perfectly and appear in no JVM log. Many threads straggling at once
  is the fingerprint. Always check the cgroup counters.
- **83 — GraalVM native image**: a different execution model with different safepoint
  characteristics and no JIT. Worth contrasting once you understand the JIT's role here.
- **85 — `synchronized` and monitors**: biased-lock revocation was historically a
  frequent non-GC safepoint operation. Biased locking is gone on modern JDKs, but the
  war stories explain a lot of older tuning advice you will encounter.
- **96 — False sharing**: the polling page and thread-local poll addresses are memory
  the hardware caches. The same hardware reality, one level down.
- **101 — Virtual threads**: thousands of threads change the at-safepoint cost of stack
  scanning and thread dumps, and virtual-thread pinning interacts with safepoint
  behaviour. Root scanning is where thread count and stop-the-world duration meet.

---

*Java baseline 21, running on JDK 25. Four things in this document are deliberately
hedged rather than asserted: the default value of `UseCountedLoopSafepoints` and
`LoopStripMiningIter` on your JDK and collector, whether a `long` loop counter still
restores the back-edge poll on your JDK, the exact field names and units in your
`-Xlog:safepoint` output, and the current default and role of
`GuaranteedSafepointInterval`. Each has a command in the Hands-on section that settles
it on your machine in under a minute, and the failure drill's result table explicitly
covers the outcome where your JDK's defaults protect you. No pause duration, TTSP
figure, percentile or throughput number in this document was measured — every number you
act on must come from your own logs against your own baseline. What has been stable and
will still be true at 2am: a global safepoint waits for the last thread; compiled code
polls at method returns and non-counted-loop back-edges; the GC log reports only the
time spent at the safepoint, never the time spent reaching it; and no collector change
has ever fixed a time-to-safepoint problem.*
