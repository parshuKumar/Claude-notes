# 78 — Profiling: async-profiler, JFR, Flame Graphs, Wall-Clock vs CPU

## Phase: 8 — JVM Internals
## Category: ELITE
## Java baseline: 21  |  Notes features from: 21 / 25
## Project spine: this is where you stop guessing what `orderflow` is doing. You profile the running containerised service at the Topic 65 baseline — 100k products, 1M orders, 5M order lines, k6 driving 70% catalogue read / 20% order read / 10% order placement — produce a flame graph, and write a top-three-costs summary you would defend in a design review. Then you do it again in wall-clock mode and find out that the answer changed, and why.

---

## R0 — READ THIS BEFORE ANY OTHER LINE IN THIS DOCUMENT

**I do not have a JVM, a profiler, or a running `orderflow`. There is not a single
captured profiler output in this document, and there is no flame graph image.**

Specifically, you will not find here:

- a flame graph, rendered or described as if I had looked at one,
- a percentage attributed to any method ("Jackson was 34% of CPU"),
- JFR output, `jfr summary` counts, event counts, or overhead figures,
- a sample count, a stack depth, a duration, or a recording size.

**Why this matters more here than almost anywhere:** the entire skill of this topic is
*reading a picture you generated yourself and drawing a conclusion from it*. If I hand
you a conclusion, you never learn to read the picture. Worse, a fabricated percentage
about `orderflow` would send you looking for a bottleneck that does not exist in your
build, and you would find something that looks close enough and stop.

What you get instead, everywhere:

- the **exact command with exact flags**,
- **WHAT TO LOOK FOR** in the output you generate,
- a **"what you see" → "what it means"** table covering both plausible and surprising
  outcomes, because the surprising outcome is where the learning is.

### The labelled exceptions

To read a profiler's output you must know its shape. In three places I show the
**structure** of a format — field names and their order — with every value replaced by
`<n>`, `<name>` or `xxx`, each carrying the inline label:

> *illustration of the format, not captured output*

Placeholders only. No plausible-looking numbers, ever.

### Spec-level facts I state plainly, each with a confirming command

1. **JVMTI's stack-walking entry points require the target thread to be at a safepoint.**
   This is what makes classic JVMTI sampling profilers safepoint-biased. Confirm the
   *effect* by running the same workload under two profilers and comparing (Proof 6).
2. **async-profiler walks Java stacks from a signal handler via `AsyncGetCallTrace`, not
   at a safepoint.** Confirm: `./asprof --version` and the tool's own documentation; the
   observable consequence is Proof 6's disagreement.
3. **JFR is built into the JVM and is event-based, not sample-based at its core** —
   sampling is one event type among hundreds. Confirm: `jfr summary <file>.jfr` lists
   every event type in the recording; `jfr metadata <file>.jfr` prints their field
   definitions.
4. **CPU-mode profiling produces zero samples for a blocked thread.** Confirm: Example 1,
   which is four lines of code and takes two minutes.

### THE RULE

> **If your profiler output disagrees with anything in this document, YOUR OUTPUT IS THE
> TRUTH.** Not mine, not a conference talk's, not the blog post whose flame graph you
> half-remember. And when two profilers disagree with *each other*, that disagreement is
> a finding — Trap 2 is entirely about what it means.

---

## Mechanical statement

Read this three times.

> **Most Java profilers sample only at safepoints, and therefore over-report code near
> safepoint polls.**
>
> A sampling profiler must stop a thread and read its stack. The classic way to do that
> is JVMTI — `GetStackTrace` / `GetAllStackTraces` — and those entry points need the
> thread to be at a **safepoint** (Topic 73). Compiled code polls for safepoints at
> **method returns and non-counted-loop back-edges**, and not in between. So every sample
> lands at one of a limited set of locations in the code. Work that happens *between*
> poll points is invisible; the method that happens to sit *at* a poll point absorbs the
> attribution for its neighbours.
>
> The result is not noise. It is **bias** — a systematic, reproducible error that points
> at the wrong method with total confidence, and that you cannot detect by taking more
> samples.
>
> **async-profiler is not safepoint-biased.** It uses two mechanisms neither of which
> needs a safepoint: an OS signal (from `perf_event_open` on Linux, or an interval timer)
> to interrupt the thread wherever it is, and HotSpot's `AsyncGetCallTrace` to walk the
> Java stack from inside that signal handler. The sample lands where the thread actually
> was.
>
> **And the second axis, which is independent of the first and matters more often:**
>
> **CPU profiling shows where CYCLES go. Wall-clock profiling shows where TIME goes.**
>
> A CPU profile samples a thread only while it is executing on a core. A thread blocked
> on a socket read contributes **zero samples**, no matter how long it blocks. So for a
> service that spends most of a request waiting on Postgres, a CPU profile is a
> high-resolution picture of the 8% of the request that is not the problem — and it will
> name a real method, with a real percentage, and be entirely useless.
>
> A wall-clock profile samples every thread at a fixed interval regardless of state, so
> blocked time appears as frames like `SocketInputStream.read` or `Unsafe.park`. It
> answers "where does the latency come from". It is also much noisier and requires thread
> filtering, because two hundred idle pool threads are still threads.
>
> **Therefore: before you read a flame graph, you must know which of the two questions it
> answers. A profile is not "the truth about the program". It is the answer to one
> specific question, and the most common profiling mistake in the industry is reading a
> CPU profile as if it were a latency profile.**

The one-line version:

*"CPU profiling tells you what the processor is busy with. Wall-clock profiling tells you
what the user is waiting for. For an I/O-bound service those are different pictures, and
only one of them is about your p99."*

---

## The bridge from what you know

### Chrome's flame chart is an HONEST ANALOGUE — and you should trust it

This is the rare case where your existing intuition transfers almost intact. Say the
three properties out loud, because they are identical:

| Property | Chrome DevTools Performance | Java flame graph |
|---|---|---|
| **Width = time** | A wider block spent more time | A wider frame appeared in more samples, which is proportional to time |
| **Stacking = call depth** | A block sitting on another was called by it | A frame sitting on another was called by it |
| **Read bottom-up** | The bottom is the entry point; the top is what was executing | The bottom is the thread's root frame; the top is the method on the CPU |
| **Plateaus at the top matter** | A wide flat block at the top is where the time went | Same. Wide plateaus at the top are your hot leaves |
| **Narrow towers are usually noise** | Yes | Yes |

Everything you know about scanning a Chrome performance recording — look for the wide
plateau, follow it down to find who called it, ignore the thin spires — works. You are
not learning a new visual language. **That is genuinely most of the skill, and you already
have it.**

### The four differences that will bite you

**Difference 1 — flame *graph* versus flame *chart*, and the x-axis is not time.**

This is the one that confuses everyone coming from DevTools, so get it straight now.

- Chrome shows you a **flame chart**: the x-axis is **wall-clock time, left to right**.
  You can see that method A ran, then method B ran, then A ran again. Order is preserved.
- A classic Java **flame graph** is **aggregated**: every occurrence of the same stack
  across the whole recording is merged into one frame, and the frames at each level are
  sorted **alphabetically** so that identical stacks line up. The x-axis has **no
  temporal meaning at all**. Left-to-right position tells you nothing except "the letter
  it starts with".

Consequences you must internalise:

- **You cannot see phases.** A startup phase and a steady-state phase are merged into one
  picture. If you want phases, either record them separately or use a time-ordered view.
- **Width is total inclusive time across the whole recording**, not "one call took this
  long".
- **Position is meaningless.** Two adjacent frames have no relationship beyond sharing a
  parent.

Modern tooling blurs this: async-profiler and JFR viewers can also produce time-ordered
views, and some flame-graph HTML supports switching. **Know which one you are looking at.**
The command in Hands-on Proof 4 tells you.

**Difference 2 — many threads, not one.**

Chrome profiles the main thread. There is one call stack, and it is the whole story.

`orderflow` under the Topic 65 load has, at minimum: a Tomcat request-thread pool, the
HikariCP housekeeper, G1's GC worker threads, JIT compiler threads, a Kafka consumer
thread, and scheduled-task threads. A flame graph merges **all of them** unless you tell
it not to.

That merge is often exactly what you *don't* want. A profile showing "40% in
`Unsafe.park`" may mean the entire idle thread pool doing nothing at all. Thread
filtering and per-thread splitting are not advanced options in Java profiling; they are
the first thing you reach for.

**Difference 3 — safepoint bias, which has no browser equivalent.**

V8's sampler interrupts on a timer and reads the stack. There is no safepoint concept,
so there is no safepoint bias, so you have never had to distrust a profiler on structural
grounds. In Java you do, and it is the single most important thing this topic teaches you
that DevTools never could.

**Difference 4 — CPU versus wall-clock is a decision you must make.**

Chrome's performance panel gives you both at once, implicitly: you see the main thread's
activity and you see the idle gaps, and network waits appear on their own track. Java
profilers make you *choose a mode up front*, and choosing wrong produces an answer that
is precise, confident, and about the wrong thing.

### What transfers, summarised

| Transfers cleanly | Does not transfer |
|---|---|
| Width = time, stacking = depth, read bottom-up | The x-axis being chronological |
| Look for wide top plateaus | One thread being the whole story |
| Ignore thin spires | Trusting the profiler's attribution structurally |
| "Who called this?" by looking down | Getting CPU and blocked time in one view for free |
| Self time versus total time | — you already have this, and it is important here too |

**Verdict: HONEST ANALOGUE for the picture, not for the pipeline that produces it.**

---

## What is this?

**Profiling is answering "where does my program spend its time" by statistical sampling
rather than by measurement.**

The mechanism is simple and worth stating plainly, because once you see it you can
predict every limitation:

1. Something interrupts a thread at intervals.
2. Something walks that thread's call stack and records it as a list of frames.
3. Steps 1–2 repeat thousands of times.
4. A tool counts how many recorded stacks contain each frame.
5. Frame counts, divided by total samples, are reported as percentages.

**Everything that can go wrong with a profiler is a consequence of one of those five
steps.** *When* you interrupt (safepoint or signal) is bias. *Which threads* you
interrupt (running only, or all) is the CPU/wall-clock distinction. *How well* you can
walk the stack (inlined frames, native frames) is accuracy. *How many* samples you take
is resolution.

### The three tools you will actually use

**1. async-profiler.** An external, open-source, low-overhead sampling profiler. On Linux
it uses `perf_event_open` for CPU-cycle-based sampling and `AsyncGetCallTrace` for Java
stack walking. It supports several event types — CPU cycles, wall-clock, allocation
(`alloc`), lock contention (`lock`) — and emits flame graphs directly as self-contained
HTML. **This is your primary tool for a development or staging investigation.**

**2. JFR (JDK Flight Recorder).** Built into the JVM, no external dependency, designed to
be safe to run continuously in production. It is **event-based**, not merely a sampler:
method-execution samples are one event type among hundreds, alongside GC events,
safepoint events, thread-park events, allocation events, socket-read events, JIT
compilation events, and exception events. That breadth is its superpower — a JFR
recording lets you correlate a latency spike with a GC pause with a safepoint with a
socket read, from one file. **This is your production tool.**

**3. A thread dump.** `jcmd <pid> Thread.print`. Not a profiler, but the zeroth tool: for
a service that is stuck, three thread dumps thirty seconds apart will often tell you more
than an hour of profiling. Costs a global safepoint each time (Topic 73), so do not put
it in a loop.

### What each one is NOT

| Tool | Is not |
|---|---|
| async-profiler | Something you leave running in production indefinitely; something that works without kernel permissions in a locked-down container without configuration |
| JFR | A flame-graph tool by itself (you convert its output); a substitute for async-profiler's allocation precision |
| Thread dumps | A profiler. Three samples is not a distribution, and taking many is a safepoint storm |
| Any of them | A benchmark. They tell you where time goes, not what an alternative implementation would cost. That is Topic 77 |
| Any of them | A memory-leak tool. Retention is Topic 79 |

### The two axes, as a decision table

| | **CPU mode** | **Wall-clock mode** |
|---|---|---|
| Samples | Only threads on a core | All threads, running or blocked |
| Answers | "What is the processor busy with?" | "What is the request waiting for?" |
| Right for | CPU-bound work: serialisation, parsing, computation, GC-heavy paths, a service pegged at 100% CPU | I/O-bound work: database calls, HTTP calls, lock contention, queue waits, thread-pool starvation |
| Wrong for | A service at 15% CPU with a 200 ms p99 | Finding which computation to optimise |
| Noise | Low. Idle threads contribute nothing | **High.** Idle pool threads dominate unless filtered |
| `orderflow` at baseline | Names your serialiser or your hash lookups | Names Postgres, the connection pool, or a lock |

**Neither mode is more correct. They answer different questions, and the failure drill in
this document makes you feel that in your hands.**

---

## Why does it matter?

### Because optimisation without a profile is guessing, and guessing is usually wrong

Every engineer has an intuition about where their program spends time. That intuition is
reliably wrong, and it is wrong in a specific direction: **you suspect the code you find
interesting, and the time is in code you find boring.** It is in serialisation, in
reflection, in logging, in the connection pool, in a regex someone wrote in 2021, in an
N+1 query, in `toString()` on an object with 40 fields being called by a log statement at
DEBUG level that is never printed.

A profile is the correction to that bias. It is also the only defensible input to the
question "what should we optimise", and it is the step that Topic 77's Rule 5 says must
come *before* any microbenchmark.

### Because a CPU profile of an I/O-bound service is a beautifully-rendered lie

This is the specific failure this document exists to prevent, and it is extremely common.

`orderflow` at the Topic 65 baseline is a normal Spring Boot service in front of
Postgres. A large fraction of a `GET /orders/{id}` request is spent waiting: for a
connection from HikariCP, for Postgres to plan and execute, for bytes to come back over a
socket. During all of that, the request thread is **blocked and off-CPU**.

A CPU profile samples nothing during that time. So it reports, with total confidence and
a real percentage, that the service's time goes to whatever it *does* do on-CPU — very
likely JSON serialisation or the mapping layer. An engineer optimises JSON serialisation.
The p99 does not move, because JSON serialisation was never the problem; it was merely
the largest thing in the small part of the request that the tool could see.

**You will do this once. The failure drill makes you do it deliberately, so that the once
is under controlled conditions and costs you an hour instead of a sprint.**

### Because a safepoint-biased profiler will name a specific innocent method

And it will name it consistently, run after run, which is *exactly* what makes it
convincing. The mid-level engineer sees `HashMap.get` at 30% and starts replacing hash
maps. The senior engineer sees `HashMap.get` at 30% and says "that is a suspicious
result, what profiler produced it, and does async-profiler agree?"

Reproducibility is not validity. A biased instrument gives the same wrong answer every
time.

### Because it is the first step of the three-step performance loop

Say this shape in an interview:

1. **Topic 78 (here)** — profile the running service under the Topic 65 baseline load, in
   the right mode, and find where the time goes.
2. **Topic 77** — microbenchmark the specific method the profile named, with error bars.
3. **Topic 65** — re-run the load baseline and prove the endpoint percentiles moved by
   more than the ±10% gate tolerance.

Skip step 1 and you optimise the wrong thing. Skip step 2 and you cannot say how much you
gained. Skip step 3 and you cannot prove you gained anything.

### Because "can we profile production?" is a real and answerable question

Most engineers believe profiling is a development-only activity. It is not. JFR was
designed to be on continuously, and the correct professional answer to "what is the
overhead" is not a number you read somewhere — it is *"I measured it on our service by
running the Topic 65 baseline with and without the recording and comparing p99."* That
answer, and the fact that you can produce it, is a senior-level differentiator.

---

## Machine-level reality

Four mechanisms: how a stack gets walked, why that walk can be biased, JFR's event model,
and how a flame graph is actually built out of text.

### 1. `AsyncGetCallTrace` versus JVMTI — the whole basis of the bias

**The JVMTI path (the biased one).**

JVMTI is the JVM Tool Interface — the official, documented, supported C API for tooling.
Its stack-walking functions are `GetStackTrace` (one thread) and `GetAllStackTraces` (all
threads). Both have a constraint that is easy to skim past and is the whole story:

**they need the target thread to be at a safepoint.**

Recall Topic 73: a thread can only stop at a point where the JVM has a precise map of
which registers and stack slots hold object references. Compiled code polls for a
safepoint request at **method returns** and at **the back-edges of non-counted loops** —
and nowhere else. C2 may elide the poll in a counted `int` loop entirely.

So a JVMTI-based sampling profiler cannot sample "wherever the thread is". It can only
sample where the thread is *allowed to stop*. Three consequences:

| Consequence | What it does to your flame graph |
|---|---|
| Samples cluster at safepoint polls | Methods that sit at or immediately after a poll point absorb attribution that belongs to their neighbours |
| Long stretches between polls are invisible | A tight counted loop over a large array — exactly the kind of code you most want to find — can be **entirely unsampled** |
| `GetAllStackTraces` requests a **global safepoint** | The profiler stops the whole application to take a sample. That is an observer effect measured in milliseconds per sample (Topic 73), and it perturbs the very timing you are measuring |

The bias is **systematic**. Taking ten times as many samples gives you ten times as many
samples of the same wrong locations, with tighter confidence intervals around a wrong
answer. **This is the reason you cannot fix a biased profiler by profiling for longer**,
and it is the single most misunderstood thing about Java profiling.

**The `AsyncGetCallTrace` path (the unbiased one).**

`AsyncGetCallTrace` — AGCT — is a HotSpot-internal entry point, not part of the JVMTI
specification, that walks a Java thread's stack **from inside an asynchronous signal
handler**, without requiring a safepoint. It has existed for a long time and is used by
essentially every serious modern Java profiler.

async-profiler's CPU mode works like this:

1. Ask the kernel, via `perf_event_open`, to deliver a signal to a thread every N CPU
   cycles (or every N nanoseconds of CPU time). Where `perf` is unavailable, fall back to
   an interval timer (`itimer`).
2. The signal arrives **wherever the thread happens to be** — mid-loop, mid-method, in
   inlined code, anywhere.
3. Inside the signal handler, call AGCT to reconstruct the Java stack.
4. Record the frame list.

Because the interrupt is asynchronous and the walk does not need a safepoint, there is no
safepoint bias.

**AGCT's honest limitations, which you will see in your output:**

| Limitation | How it appears | What to do |
|---|---|---|
| The thread may be in a state where AGCT cannot walk (mid-transition, in some native code, during class loading) | AGCT returns an error; async-profiler reports these as **failed samples** or `[unknown_Java]` frames | A small number is normal. A large plateau of `[unknown_Java]` means something structural — see Trap 4 |
| Inlined frames need debug metadata to attribute correctly | Missing intermediate frames; attribution to the caller instead of the inlined callee | **`-XX:+UnlockDiagnosticVMOptions -XX:+DebugNonSafepoints`** — see below |
| Native (C/C++) frames need frame pointers to walk | Broken or truncated native stacks | **`-XX:+PreserveFramePointer`** |
| It is not a specified API | Behaviour can change between JDK builds | Check your profiler's release notes against your JDK |

**The two flags that matter, and why:**

```bash
# Emit debug information at NON-safepoint locations, so a sample taken between
# poll points can be attributed to the right (possibly inlined) method.
-XX:+UnlockDiagnosticVMOptions -XX:+DebugNonSafepoints

# Keep the frame pointer register available so native stacks can be walked.
# Costs a register; a small, real, and usually acceptable throughput cost.
-XX:+PreserveFramePointer
```

`DebugNonSafepoints` is the important one and it is under-known. Without it, C2 emits
debug information only at safepoints — which means that even though async-profiler
*sampled* you at a non-safepoint location, it may not be able to *name* that location
precisely, and attribution slides to the nearest point that does have metadata. **You get
a partially safepoint-flavoured answer from an unbiased sampler**, which is the worst of
both worlds because you think you are safe.

> **Flagged uncertainty.** Some JDK builds and some profiler versions enable
> `DebugNonSafepoints` implicitly when a profiler attaches. I am not confident enough
> about which combinations do this to have you rely on it. **Set it explicitly.** Confirm
> what is actually on with:
>
> ```bash
> jcmd <pid> VM.flags -all | grep -E "DebugNonSafepoints|PreserveFramePointer"
> ```

> **Flagged uncertainty, second.** There has been long-running work on a specified,
> supported successor to AGCT (an asynchronous stack-trace VM API). I am **not** confident
> about its current status, its exact name in a shipped JDK, or whether your JDK 25 build
> has it. Do not take a flag or an API name for it from this document. Check the JEP
> index at `https://openjdk.org/jeps/0` and your profiler's documentation.

### 2. JFR's event model — why it is a different kind of tool

JFR is not "a profiler with extra features". It is an **event recorder** that happens to
include a sampling event.

**The model:**

- The JVM and the JDK libraries are instrumented with hundreds of **event types**.
- Each event type has a **schema**: named, typed fields. `jdk.GCPhasePause` has a
  duration and a name. `jdk.SocketRead` has a host, a port, a byte count and a duration.
  `jdk.ExecutionSample` has a thread, a state, and a stack trace. `jdk.ObjectAllocationSample`
  has a class and a weight.
- Events are written to a **thread-local buffer**, flushed to a global buffer, then to
  disk. This design is why the overhead is low: the hot path is a buffer append.
- Events fall into three shapes: **duration** events (start and end), **instant** events
  (a point in time), and **sample** events (periodic).
- **Settings** — the `default` and `profile` templates — decide which events are enabled
  and at what threshold. `default` is intended for continuous production use; `profile`
  enables more and samples more often.

**Why this matters practically:** a JFR recording lets you answer questions no flame graph
can. "Was the p99 spike a GC pause, a safepoint, or a slow socket read?" is one query
over three event types in one file. That correlation ability is JFR's actual value, and
people who treat it as "a worse async-profiler" miss the point entirely.

**The method-sampling event.** `jdk.ExecutionSample` is JFR's periodic Java-stack sample.
Two things to know and one to check:

- It samples threads it considers to be **executing Java code**, which makes it
  CPU-profile-shaped by default. There is a companion `jdk.NativeMethodSample` for threads
  in native code.
- Its stack-walking accuracy benefits from `DebugNonSafepoints` the same way
  async-profiler's does.

> **Flagged uncertainty.** The precise sampling mechanism JFR's execution sampler uses,
> and therefore its exact bias characteristics, have changed across JDK releases and I am
> not going to characterise them from memory. What I am confident of: JFR's sampler is
> **not** the classic `GetAllStackTraces`-at-a-global-safepoint design, and it does not
> stop the world per sample. Whether it is fully free of safepoint-flavoured bias on your
> specific JDK build is something to establish empirically — **run Proof 6, which compares
> JFR against async-profiler on the same workload.** If they agree, you have corroboration.
> If they disagree, that disagreement is the finding.

**Inspecting a recording is entirely command-line, and you should learn these:**

```bash
jfr summary recording.jfr                  # every event type present, with counts
jfr metadata recording.jfr                 # the schema: field names and types
jfr print --events jdk.ExecutionSample recording.jfr | head -60
jfr print --events jdk.GCPhasePause,jdk.SafepointBegin recording.jfr
jfr print --events jdk.SocketRead --stack-depth 20 recording.jfr
```

The structure of a printed event:

```
jdk.ExecutionSample {
  startTime = <timestamp>
  sampledThread = "<thread-name>" (javaThreadId = <n>)
  state = "<STATE>"
  stackTrace = [
    <fully.qualified.Class>.<method>(<descriptor>) line: <n>
    <fully.qualified.Class>.<method>(<descriptor>) line: <n>
    ...
  ]
}
```

*illustration of the format, not captured output — field names and nesting only*

**WHAT TO LOOK FOR** in that structure: the `state` field. It is the field that tells you
whether this sample represents on-CPU work or something else, and it is the field that
makes a JFR recording partly answer the wall-clock question even though the sampler is
CPU-shaped.

### 3. How a flame graph is actually built — the folded-stack format

Demystifying this removes all the magic. A flame graph is generated from a plain text
file where **each line is one distinct stack, semicolon-separated from root to leaf,
followed by a count**:

```
<thread-or-root>;<frame1>;<frame2>;<frame3> <n>
<thread-or-root>;<frame1>;<frame2>;<frame4> <n>
<thread-or-root>;<frame1>;<frame5> <n>
```

*illustration of the format, not captured output — structure only; `<n>` is a sample count*

That is the whole intermediate representation. To build the picture:

1. Group lines by their common prefix. Frames sharing a prefix become one merged frame.
2. At each depth, **sort the frames alphabetically** and lay them left to right.
3. Frame width = sum of the counts of every line passing through it.
4. Colour is (in the classic scheme) random within a hue family, chosen only to make
   adjacent frames distinguishable — **colour carries no meaning** unless the tool says
   it does (async-profiler does colour Java / native / kernel / inlined frames
   differently, which is genuinely useful).

Two facts fall straight out of this construction and they are the two most misread things
about flame graphs:

- **The x-axis is alphabetical, not chronological.** Alphabetical sorting is what makes
  identical stacks from different points in time merge into one wide frame. Merging is
  the entire point, and time ordering is what it costs.
- **Frame width is inclusive time** (this frame and everything above it). The **self**
  time of a frame is its width minus the total width of its children — visually, the part
  of the frame with nothing sitting on it. A wide frame with a wide child is not itself
  expensive; a wide frame with nothing above it is.

You can produce the folded format yourself and read it as text, which is often faster than
squinting at a picture:

```bash
./asprof -d 30 -o collapsed -f /tmp/orderflow.collapsed <pid>

# The twenty hottest distinct stacks, by sample count:
sort -k2 -n -r /tmp/orderflow.collapsed | head -20

# Total samples, so percentages are computable:
awk '{s+=$NF} END {print s}' /tmp/orderflow.collapsed

# Everything mentioning your own package, aggregated:
grep -o 'com\.orderflow\.[A-Za-z0-9_.$]*' /tmp/orderflow.collapsed | sort | uniq -c | sort -rn | head -20
```

**WHAT TO LOOK FOR:** whether the hottest stacks are in your code, in a framework, in the
JDK, or in the kernel. That single question, answered from the text, is often 80% of the
investigation and takes thirty seconds.

### 4. Why CPU mode produces zero samples for a blocked thread — mechanically

In CPU mode, async-profiler asks the kernel for a signal every N **CPU cycles consumed by
this thread** (a `perf` counter), or every N nanoseconds of **thread CPU time** (an
itimer). A thread blocked in a `read(2)` syscall waiting for Postgres is **not consuming
cycles and not accruing CPU time**. The counter does not advance. The signal is never
delivered. The thread contributes nothing.

This is not a bug, an approximation, or a tuning issue. It is what CPU profiling *means*.

In wall-clock mode, the profiler instead maintains its own timer and, on each tick, walks
the stacks of a set of threads **regardless of their state**. A parked thread yields a
stack ending in `Unsafe.park` or `SocketInputStream.read`, and that is precisely the
information you wanted.

The cost of wall-clock mode falls directly out of that mechanism: **every idle thread
produces a sample on every tick.** Two hundred idle Tomcat workers parked in
`ThreadPoolExecutor.getTask` will produce the widest tower in your flame graph and it will
mean nothing whatsoever. **Thread filtering is not optional in wall-clock mode.** This is
Trap 3, and everybody hits it once.

---

## Example 1 — minimal

**The goal:** feel the CPU-versus-wall-clock difference in two minutes, with code so
simple that there is no possible ambiguity about the right answer.

### The program

```java
package com.orderflow.demo;

import java.util.concurrent.TimeUnit;

/**
 * Two costs, deliberately unequal and deliberately of different KINDS.
 *   - recalculatePrices(): pure CPU, a short computation.
 *   - callPaymentGateway(): pure blocking, no CPU at all.
 * A correct wall-clock profile must attribute most of the time to the second.
 * A correct CPU profile must attribute nearly all of its samples to the first.
 * Both profilers are right. They are answering different questions.
 */
public final class ProfilerModeDemo {

    public static void main(String[] args) throws InterruptedException {
        while (!Thread.currentThread().isInterrupted()) {
            recalculatePrices();
            callPaymentGateway();
        }
    }

    /** CPU-bound: a real computation over an order's lines. */
    private static long recalculatePrices() {
        long acc = 0;
        for (int i = 0; i < 2_000_000; i++) {
            acc += (i * 31L) ^ (acc >>> 7);
        }
        return acc;
    }

    /** Blocking: stands in for a Postgres round trip or a payment-gateway call. */
    private static void callPaymentGateway() throws InterruptedException {
        TimeUnit.MILLISECONDS.sleep(50);
    }
}
```

Note the loop in `recalculatePrices` is a **counted `int` loop** (Topic 73). That is
deliberate: it is exactly the shape whose safepoint poll C2 may elide, which makes it the
shape a safepoint-biased profiler is worst at sampling. You are building the trap on
purpose.

### Run it and profile both ways

```bash
javac -d out ProfilerModeDemo.java
java -XX:+UnlockDiagnosticVMOptions -XX:+DebugNonSafepoints -XX:+PreserveFramePointer \
     -cp out com.orderflow.demo.ProfilerModeDemo &
PID=$!

# CPU mode: samples only while on a core.
./asprof -e cpu  -d 20 -f /tmp/demo-cpu.html  $PID

# Wall-clock mode: samples all threads regardless of state.
./asprof -e wall -d 20 -t -f /tmp/demo-wall.html $PID

kill $PID
```

Open both HTML files in a browser.

### WHAT TO LOOK FOR

Do not look for numbers. Look for **which method is the wide frame** in each picture.

| What you see | What it means |
|---|---|
| `demo-cpu.html`: `recalculatePrices` is essentially the entire graph; `callPaymentGateway` is absent or a sliver | **Correct.** The sleeping thread consumes no cycles and is not sampled. This is CPU profiling working exactly as designed. |
| `demo-wall.html`: `callPaymentGateway` (leading into `Thread.sleep` / `park`) is the dominant frame | **Correct, and the point of the exercise.** Most of the *elapsed* time is spent sleeping, and only wall-clock mode can see that. |
| Both pictures identical | Something is wrong: either wall-clock mode did not engage, or the sleep is far shorter than you think. Check that you passed `-e wall`, and check the sleep duration. |
| Wall-clock graph dominated by `main` doing nothing recognisable | You are probably seeing other JVM threads. Add `-t` to split by thread and read only the `main` tower. |
| CPU graph shows `recalculatePrices` but attributes it to the wrong line, or shows an `[unknown_Java]` chunk | Missing `DebugNonSafepoints`. Restart with the flag and compare. **This comparison is itself a useful proof.** |
| Wall-clock graph shows kernel frames beneath the sleep | Expected and good — you are seeing the syscall. Colour normally distinguishes these. |

### The sentence to write down

> *"Both profiles are correct. `recalculatePrices` is where the CPU goes;
> `callPaymentGateway` is where the time goes. If I want to reduce CPU cost I optimise the
> first. If I want to reduce latency I attack the second. Choosing the wrong mode means
> optimising the first while the user waits on the second."*

That sentence, applied to a real service, is the entire value of this topic.

### The variation that makes the point sharper

Change the sleep to 1 ms and the loop to 20 million iterations, and re-run both. Now the
program is genuinely CPU-bound and the two profiles will **agree**. That agreement is
itself informative: **when CPU and wall-clock profiles agree, your service is CPU-bound;
when they disagree, the size of the disagreement is your blocked time.** That is a
diagnostic technique, not just a curiosity, and it is what the failure drill formalises.

---

## Example 2 — production scenario (on the project spine)

### The constraints, taken from the Topic 65 baseline

| Thing | Value |
|---|---|
| Service | `orderflow`, containerised, behind k6 |
| Dataset | 100k products, 1M orders, **5M order lines** (≈5 lines/order) |
| Container | 2 vCPU, 2 GiB memory limit |
| Heap | `-Xmx1200m -Xms1200m` |
| Collector | G1, confirmed with `jcmd <pid> VM.flags -all` |
| Load mix | 70% catalogue read, 20% order read, 10% order placement |
| Arrival model | **Open model** (constant arrival rate), not a fixed VU loop — Topic 65 |
| Baseline artefacts | `/docs/java/baselines/<date>-run-01/` — p50/p95/p99/p999 per endpoint |
| Gate rule | a re-run must land within ±10% on every recorded percentile |

### The situation

`GET /orders/{id}` has a p99 you consider too high. Nobody knows why. The team's
hypotheses, in the order they were shouted in the channel:

1. "Jackson serialisation — those order responses are big."
2. "It's GC, we should switch to ZGC."
3. "The database. It's always the database."

All three are plausible. All three are guesses. Your job is to replace them with a
picture.

### Step 0 — get the profiler into the container, and be honest about permissions

This is the step every tutorial skips and every real investigation trips over.

```bash
# What are we attaching to? PID 1 in the container, usually.
docker exec orderflow jcmd -l

# Is perf usable in here? This is the gate for CPU-mode sampling on Linux.
docker exec orderflow cat /proc/sys/kernel/perf_event_paranoid
docker exec orderflow cat /proc/sys/kernel/kptr_restrict
```

| What you see | What it means | What to do |
|---|---|---|
| `perf_event_paranoid` is 2 or 3 | The kernel will not let an unprivileged process open perf events. CPU-cycle sampling will fail or fall back. | Run the profiling container with `--cap-add SYS_ADMIN` (or `--privileged` in a lab), or lower `perf_event_paranoid` **on the host** to 1 for the duration of the investigation. In managed Kubernetes you may not be able to; then use JFR, which needs no kernel permissions. |
| async-profiler reports it fell back to `itimer` | You are getting wall-clock-of-CPU-time sampling rather than true cycle sampling. Usable, slightly less precise, and it will not see kernel frames. | Note it in your write-up. Do not silently compare an itimer profile against a perf profile. |
| The profiler cannot attach at all | Usually a namespace or a user mismatch — the profiler must run as the same user, in the same PID namespace. | Run async-profiler **inside** the container (`docker exec`), or use JFR via `jcmd`, which works over the JVM's own attach mechanism. |

**Restart `orderflow` with the profiling-support flags before the baseline run.** These
belong in your load-test profile permanently:

```yaml
# docker-compose.yml, the "load" profile (Topic 43)
environment:
  JAVA_TOOL_OPTIONS: >-
    -Xms1200m -Xmx1200m -XX:+UseG1GC
    -XX:+UnlockDiagnosticVMOptions
    -XX:+DebugNonSafepoints
    -XX:+PreserveFramePointer
    -XX:StartFlightRecording=name=baseline,settings=profile,filename=/dumps/orderflow.jfr,dumponexit=true
```

**A necessary warning about `PreserveFramePointer`:** it reserves a register and has a
real, small throughput cost. If it is on during your profiling run but off in your
recorded baseline, you are profiling a slightly different program. The clean approach is
to **re-record the Topic 65 baseline with these flags on**, and use that as the reference
for this investigation. Note the flag set in `environment.md` alongside the numbers, the
way Topic 65 requires.

### Step 1 — reach steady state, then profile

Profiling a JVM in its first thirty seconds tells you about class loading and JIT
compilation, not about your service (Topic 74).

```bash
# 1. Start the Topic 65 load and let it run until the JIT has settled.
k6 run --out json=/tmp/k6.json loadtest/baseline.js &

# 2. Wait for steady state. "Settled" is a condition you verify, not a number you assume:
docker exec orderflow jcmd 1 Compiler.queue      # queue should be near-empty
docker exec orderflow jcmd 1 VM.uptime
watch -n 5 'docker exec orderflow jcmd 1 GC.heap_info'   # stable pattern, not still growing
```

**WHAT TO LOOK FOR before you profile:** a near-empty compiler queue, a stable young-GC
rhythm, and k6 reporting steady throughput and latency. If any of those is still moving,
you are profiling a warm-up.

### Step 2 — the CPU profile

```bash
docker exec orderflow /opt/async-profiler/bin/asprof \
    -e cpu -d 60 -o flamegraph \
    -f /dumps/orderflow-cpu.html 1

docker exec orderflow /opt/async-profiler/bin/asprof \
    -e cpu -d 60 -o collapsed \
    -f /dumps/orderflow-cpu.collapsed 1
```

Take both formats every time: the HTML to look at, the collapsed file to grep and to
keep. The collapsed file is small, diffable, and archivable next to your baseline.

**WHAT TO LOOK FOR, in this order:**

1. **Total samples**, from the collapsed file: `awk '{s+=$NF} END {print s}'`. If it is
   tiny for a 60-second run, sampling is not working — check the perf permissions.
2. **The widest top-level towers.** These are threads or thread groups. Identify each:
   Tomcat workers, GC threads, compiler threads, the Kafka consumer, the HikariCP
   housekeeper.
3. **The widest plateaus at the top of the request-thread tower.** These are your on-CPU
   leaves.
4. **Whether `com.orderflow` code appears at all**, and where.

| What you see in the CPU flame graph | What it means | What you do next |
|---|---|---|
| A wide plateau under Jackson / `ObjectMapper` / `SerializerProvider` | Serialisation genuinely dominates the *CPU* portion of the request | Note it, but **do not act yet** — you do not know what fraction of the request is CPU at all. Go to Step 3 |
| A wide plateau under Hibernate's `AbstractEntityPersister` / `ResultSetProcessor` | Entity hydration. Frequently a symptom of returning entities instead of projections (Topic 47) or of an N+1 (Topic 50) | Count queries with Hibernate `Statistics` before optimising anything |
| A wide plateau in G1 GC threads | Allocation pressure (Topics 68, 70, 71) | Cross-check with `-Xlog:gc*` and with an `alloc` profile |
| A wide plateau in a `Pattern`/regex frame | A regex on a hot path — very common and very cheap to fix | Find it, cache the compiled `Pattern`, re-measure |
| A wide plateau in a logging framework | A log statement doing work at a level that is not being emitted, or a `toString()` on a large object graph | Guard it or use parameterised logging (Topic 120) |
| The request threads barely appear at all | **The most important outcome, and the one people misread.** Your request threads are mostly off-CPU, i.e. blocked. The CPU profile is showing you the small remainder | Go to Step 3 immediately. This is the finding |
| A large `[unknown_Java]` plateau | Stack walking is failing — see Trap 4 | Add `DebugNonSafepoints`, check the JDK/profiler version pairing |
| Compiler threads dominate | You are still warming up | Wait longer and re-profile |

### Step 3 — the wall-clock profile, with thread filtering

```bash
# -t splits by thread. Without filtering, the idle pool will dominate.
docker exec orderflow /opt/async-profiler/bin/asprof \
    -e wall -t -d 60 -o flamegraph \
    -f /dumps/orderflow-wall.html 1

# Keep the collapsed form too, and filter to the request threads.
docker exec orderflow /opt/async-profiler/bin/asprof \
    -e wall -t -d 60 -o collapsed \
    -f /dumps/orderflow-wall.collapsed 1

grep '^http-nio' /dumps/orderflow-wall.collapsed > /dumps/orderflow-wall-requests.collapsed
```

> **Flagged detail.** async-profiler has include/exclude options for filtering frames and
> threads, and the exact spelling of those options varies by version. Rather than have you
> copy a flag that may not exist in your build, **check your own help output**:
>
> ```bash
> ./asprof --help 2>&1 | grep -iE "thread|include|exclude|filter"
> ```
>
> Grepping the collapsed file by thread-name prefix, as above, always works and needs no
> flag.

**WHAT TO LOOK FOR — and this is the comparison that produces the answer:**

| What you see in wall-clock, versus CPU | What it means |
|---|---|
| A dominant tower under `HikariDataSource.getConnection` / `HikariPool.getConnection` | **Connection-pool starvation** (Topics 55, 109). Requests are waiting for a connection, not for the database. Check pool size against concurrency, and look for long transactions holding connections |
| A dominant tower under `SocketInputStream.read` / `NioSocketImpl` beneath a JDBC frame | **Waiting on Postgres.** Now the question is *why* — slow query, N+1, missing index. Move to `pg_stat_statements` and Hibernate `Statistics` |
| Many narrow `SocketRead` towers rather than one wide one | Lots of small round trips: the classic N+1 shape (Topic 50). **This is what an N+1 looks like in a wall-clock flame graph**, and recognising it on sight is worth the whole exercise |
| A dominant tower under `Unsafe.park` in `ThreadPoolExecutor.getTask` | Idle pool threads. **Filter them out** — this is noise, not a finding (Trap 3) |
| A dominant tower under a `synchronized` block or `ReentrantLock.lock` | Lock contention (Topics 85, 94). Cross-check with `-e lock` mode and with JFR's `jdk.JavaMonitorEnter` events |
| Jackson still dominant in wall-clock too | Then serialisation really is the problem, and the team's first hypothesis was right. Now microbenchmark it (Topic 77) |
| Wall-clock and CPU graphs look the same | The service is CPU-bound. Genuinely possible at 2 vCPU under heavy load. Check container CPU throttling (Topic 82) before concluding |
| A tower in `G1` / `SafepointSynchronize` frames | Correlate with `-Xlog:safepoint*` (Topic 73). A wall-clock profile can show you the *effect* of stop-the-world pauses on request threads |

### Step 4 — the written top-three, which is the actual deliverable

A flame graph is not a deliverable. **A short written summary that someone can act on
is.** Write it in this shape, filling in your own observations:

```markdown
# orderflow profile — GET /orders/{id} at the Topic 65 baseline

Recorded: <date>. Duration: 60s at steady state, k6 baseline scenario.
JVM flags: <paste from environment.md>. Profiler: <asprof --version>.
Artefacts: /dumps/orderflow-cpu.html, /dumps/orderflow-wall.html, both .collapsed files.

## Top three costs — WALL-CLOCK (what the user waits for)
1. <frame path>  — <n>% of request-thread samples. Interpretation: <...>
2. <frame path>  — <n>%. Interpretation: <...>
3. <frame path>  — <n>%. Interpretation: <...>

## Top three costs — CPU (what the processor is busy with)
1. <frame path>  — <n>% of CPU samples. Interpretation: <...>
2. ...
3. ...

## Why the two lists differ
<One paragraph. If wall-clock is dominated by socket reads and CPU by Jackson, say so
explicitly: "the service is I/O-bound; the CPU profile describes the ~X% of the request
that is not waiting.">

## Recommendation
<What to do, in priority order, with the expected effect on p99 and how you will verify
it against /docs/java/baselines/.>

## What I am NOT recommending, and why
<The hypotheses this profile rules out. Naming what you have excluded is as valuable as
naming what you found — it stops the team relitigating it next week.>
```

### The senior move: the numbers only mean something as fractions of the request

A flame graph gives you percentages **of samples**, not of request latency. Before you
act, connect the two:

- **Wall-clock, filtered to request threads**, is approximately a breakdown of request
  latency — because the request thread's wall time *is* the request. This is the profile
  whose percentages you may multiply by p99 to get a milliseconds estimate.
- **CPU percentages are fractions of CPU time**, which may be a small fraction of the
  request. Multiplying them by p99 is meaningless, and it is the arithmetic error behind
  most wasted optimisation work.

So the sentence to put in the design review is:

> *"Wall-clock says `<n>`% of request-thread time is in `<frame>`. Our p99 baseline for
> this endpoint is `<n>` ms, so that is roughly `<n>` ms of the p99. Our target is a `<n>`
> ms reduction, so this is worth attacking. Jackson is `<n>`% of CPU, but CPU is only
> `<n>`% of the request, so optimising it has a ceiling of about `<n>` ms — below what our
> ±10% gate can even resolve."*

That paragraph is the whole topic, applied.

---

## Wrong approach → exact symptom → root cause → fix

Five traps. Each is stated as an observation you will actually make.

---

### Trap 1 — a CPU profile of an I/O-bound service names an irrelevant method

**Wrong approach.** You attach async-profiler in its default mode, get a flame graph,
see a wide plateau under Jackson's serialiser, and open a ticket titled "Optimise order
serialisation — 40% of CPU".

**Exact symptom — three observations that together are conclusive:**

1. The flame graph's request-thread tower is **narrow relative to the whole picture**, and
   idle/GC/compiler threads make up much of the rest.
2. The container's CPU utilisation, measured independently, is **low** while p99 is high:

   ```bash
   docker stats --no-stream orderflow
   docker exec orderflow cat /sys/fs/cgroup/cpu.stat   # nr_throttled, usage_usec
   ```

   Low CPU with high latency means the time is not being spent computing.
3. **The arithmetic does not close.** Add up the CPU samples in the request thread,
   convert to milliseconds using the profile duration and the thread count, and compare to
   `p99 × requests-in-window`. If CPU time accounts for a small fraction of the elapsed
   request time, the rest is blocked time that the profile cannot see.

**Root cause.** CPU-mode sampling is driven by cycles or CPU time consumed by the thread.
A thread blocked in a `read(2)` on the Postgres socket consumes neither. It contributes
**zero samples** for the entire blocking period. So the profile is a faithful, precise,
high-resolution description of a small minority of the request — and Jackson genuinely is
40% *of that minority*.

The percentage is not wrong. **The denominator is.**

**Fix.**

```bash
# Wall-clock mode, split by thread, then read only the request threads.
./asprof -e wall -t -d 60 -o collapsed -f /tmp/wall.collapsed <pid>
grep '^http-nio' /tmp/wall.collapsed | sort -k2 -nr | head -20
```

And the habit that prevents the trap permanently: **always take both profiles, and always
compare them.** The difference between the two is your blocked time. If they agree,
you are CPU-bound and the CPU profile was the right tool all along.

| What you see when you compare | What it means |
|---|---|
| Wall-clock dominated by socket reads; CPU dominated by Jackson | I/O-bound. Attack the I/O. Jackson's ceiling is small |
| Both dominated by the same frames | CPU-bound. The CPU profile was correct; proceed |
| Wall-clock dominated by `getConnection`, not by socket reads | Pool starvation, not database slowness. Completely different fix (Topics 55, 109) |
| Wall-clock dominated by `park` in the request threads with no obvious cause | Look for a lock (Topics 85, 94) or an executor handoff |

---

### Trap 2 — `HashMap.get` is "hot", from a safepoint-biased profiler

**Wrong approach.** You use a JVMTI-based sampling profiler — the sampler in a
general-purpose Java IDE or monitoring UI, or an older APM's sampling mode — and it
reports a large percentage in `HashMap.get`, or in some similarly tiny, universally-called
JDK method. You start replacing hash maps.

**Exact symptom.** Four things, and the fourth is the decisive one:

1. The named method is **small and universally called** — `HashMap.get`, `ArrayList.get`,
   `String.equals`, `Integer.valueOf`. Real hotspots are usually *your* code or a
   framework's, not a three-line JDK method.
2. The attribution is **stable across runs**, which people misread as validation.
3. The profiler's own documentation or settings mention JVMTI, or "sampling" without
   mentioning `AsyncGetCallTrace` or perf events.
4. **async-profiler, run on the same workload, disagrees.** Not "differs in the third
   decimal" — names a different method.

**Root cause.** JVMTI stack walking requires the thread to be at a safepoint. Samples can
therefore only land at safepoint polls: method returns and non-counted-loop back-edges.
`HashMap.get` is short and is called from everywhere; its **return** is a poll point that
an enormous number of code paths pass through. Meanwhile the work you actually care about
— a counted loop over an array whose poll C2 elided (Topic 73) — has **no poll point at
all** and is sampled never.

So the profiler over-reports the method next to the poll and under-reports the code that
cannot be polled. Both errors point you away from the truth, in the same direction, every
time.

**Prove it — this is Proof 6 and it takes ten minutes:**

```bash
# Same workload, two profilers, side by side.
./asprof -e cpu -d 60 -o collapsed -f /tmp/agct.collapsed <pid>

jcmd <pid> JFR.start name=cmp settings=profile duration=60s filename=/tmp/cmp.jfr
# ...wait...
jfr print --events jdk.ExecutionSample /tmp/cmp.jfr > /tmp/jfr-samples.txt
```

| What you see | What it means |
|---|---|
| The two tools name the same top frames | Corroboration. Trust the result |
| async-profiler names your loop; the JVMTI tool names `HashMap.get` | **Safepoint bias confirmed.** Believe async-profiler and stop using the other tool for this question |
| async-profiler shows a large `[unknown_Java]` fraction | Its own stack walking is failing — fix that first (Trap 4) before comparing |
| Both name `HashMap.get` | Then it may genuinely be hot. Verify by finding the *caller*: look down from that frame. A real `HashMap` hotspot has an identifiable caller doing an identifiable amount of lookup work, and you can then microbenchmark it (Topic 77) |

**Fix.** Use a profiler that is not safepoint-biased, set `DebugNonSafepoints`, and adopt
the rule: **when a profiler names a tiny JDK method, corroborate before acting.**

The interview-ready version: *"`HashMap.get` at 30% is a suspicious result, not a
finding. It's the shape safepoint bias produces, because `HashMap.get`'s return is a poll
point that half the codebase passes through. I'd re-run under async-profiler and see
whether the answer survives."*

---

### Trap 3 — the wall-clock profile is 90% idle threads

**Wrong approach.** You switch to wall-clock mode to fix Trap 1, take a flame graph, and
find that the widest tower by far is `ThreadPoolExecutor.getTask` →
`LinkedBlockingQueue.take` → `Unsafe.park`. You conclude the application spends 90% of its
time waiting on a queue.

**Exact symptom.** The dominant stack ends in `Unsafe.park` beneath `getTask`,
`AbstractQueuedSynchronizer.acquire`, or an event-loop select. The width is enormous and
suspiciously round. And the tell: **the width scales with your pool size, not with your
load.** Double the Tomcat `max-threads` and the tower gets wider while p99 does not change.

**Root cause.** Wall-clock mode samples **every** thread on every tick, regardless of
state. `orderflow` at the baseline has a large Tomcat pool, most of which is idle at any
instant, plus the Hikari housekeeper, plus scheduled executors, plus a Kafka consumer
polling. Every one of them contributes a sample per tick, every tick.

Idle threads are not a finding. They are the definition of a thread pool that is not
saturated — which is *good news*.

**Fix — filter, always.**

```bash
# Split by thread, then keep only the request threads.
./asprof -e wall -t -d 60 -o collapsed -f /tmp/wall.collapsed <pid>
grep '^http-nio' /tmp/wall.collapsed > /tmp/wall-req.collapsed

# Percentages within the request threads only:
awk '{s+=$NF} END {print "request-thread samples:", s}' /tmp/wall-req.collapsed
sort -k2 -nr /tmp/wall-req.collapsed | head -20
```

Then render a flame graph from the filtered file, or read it as text.

**The subtler version of this trap, which is worth knowing:** if the request-thread tower
*itself* is dominated by `park` in `getTask`, that means your request threads are idle —
which means the bottleneck is upstream of them (the acceptor, the connection limit, or the
load generator itself). Topic 65 warns about the load generator saturating; this is what
that looks like from the inside.

| What you see after filtering | What it means |
|---|---|
| Request threads mostly parked in `getTask` | The pool is not saturated. Your bottleneck is upstream — check k6's own CPU and the acceptor/connection limits |
| Request threads mostly in socket reads to Postgres | The expected I/O-bound shape. Proceed to the query investigation |
| Request threads mostly in `getConnection` | Pool starvation (Topics 55, 109) |
| Request threads mostly on-CPU in your code | CPU-bound after all — the CPU profile is now the right tool |

---

### Trap 4 — profiling without `DebugNonSafepoints`, and reading fiction

**Wrong approach.** Attach async-profiler to a default-flags JVM, get a flame graph, and
read the frames at face value.

**Exact symptom.** One or more of:

1. A wide `[unknown_Java]` or `[unknown]` plateau.
2. **Frames you know exist are missing.** You know `OrderMapper.toResponse` calls
   `LineMapper.toLine` in a loop, and `LineMapper.toLine` does not appear anywhere.
3. A caller is wide and has **no children**, so it appears to have enormous self time —
   even though you can read its source and see that it delegates everything.
4. `lambda$...$0` frames appearing without their enclosing context (Topic 76 — those are
   real synthetic method names, so their presence is normal; their *isolation* is not).

**Root cause.** C2 aggressively inlines (Topic 75). By default, HotSpot records the debug
metadata needed to map a machine-code address back to a Java method **only at safepoints**.
async-profiler samples at arbitrary instructions — which is the whole point — but at an
arbitrary instruction inside an inlined region, without non-safepoint debug info, the
runtime cannot say precisely which inlined method that address belongs to. Attribution
falls back to whatever it can determine, usually the outermost frame, and inlined callees
vanish.

The net effect: **an unbiased sampler producing safepoint-flavoured attribution.** You
believe you have escaped Trap 2 and you have not.

**Fix.**

```bash
-XX:+UnlockDiagnosticVMOptions -XX:+DebugNonSafepoints
-XX:+PreserveFramePointer          # for native frame walking
```

Verify they are on:

```bash
jcmd <pid> VM.flags -all | grep -E "DebugNonSafepoints|PreserveFramePointer"
```

**And do the A/B**, because seeing the difference is what makes you set the flags forever:
profile the same workload with and without `DebugNonSafepoints` and diff the collapsed
files.

```bash
diff <(cut -d' ' -f1 /tmp/without.collapsed | sort -u) \
     <(cut -d' ' -f1 /tmp/with.collapsed    | sort -u) | head -40
```

**WHAT TO LOOK FOR:** stacks that exist only in the `with` version. Those are the inlined
frames you were blind to.

| What you see | What it means |
|---|---|
| Many more distinct stacks with the flag on, naming methods you recognise | The flag was essential. Keep it in your load profile permanently |
| Essentially no difference | Your hot code was not heavily inlined, or was already attributed correctly. Fine — you have now confirmed it rather than assumed it |
| `[unknown_Java]` persists even with the flag | Something else: a JDK/profiler version mismatch, heavy native code, or sampling during class loading. Check `asprof --version` against your JDK and look for a `--all-user`/native-stack option in your build's help |

---

### Trap 5 — using thread dumps in a loop as a "poor man's profiler"

**Wrong approach.** No profiler available in the environment, so you write:

```bash
# DO NOT DO THIS on a production JVM.
for i in $(seq 1 200); do
  jcmd <pid> Thread.print >> /tmp/dumps.txt
  sleep 0.1
done
```

...and aggregate the top frames. It is a genuinely old and genuinely popular technique.

**Exact symptom.** Two, and the second is the serious one:

1. The aggregated result is **safepoint-biased for the same reason as Trap 2** —
   `Thread.print` walks stacks and requires threads to be at safepoints.
2. **Your p99 gets worse while you are measuring.** Each `Thread.print` requests a
   **global safepoint**: every application thread stops. At ten dumps per second on a
   service with hundreds of threads, you have added a stop-the-world event ten times a
   second. Check it:

   ```bash
   # In the JVM's own log, with -Xlog:safepoint*
   grep -c "Safepoint" /logs/safepoint.log
   grep -i "ThreadDump\|PrintThreads" /logs/safepoint.log | head
   ```

**Root cause.** Thread dumps are a safepoint operation (Topic 73, Trap 4 there:
*"a monitoring agent taking thread dumps in a loop is requesting a global safepoint every
ten seconds"*). Doing it deliberately, in a loop, at high frequency, is that failure mode
as a methodology.

**Fix.** Use a real profiler. If the environment genuinely will not permit async-profiler,
**use JFR** — it is in the JVM, needs no external binary and no kernel permissions:

```bash
jcmd <pid> JFR.start name=investigate settings=profile duration=120s filename=/dumps/x.jfr
jcmd <pid> JFR.check
jcmd <pid> JFR.dump name=investigate filename=/dumps/x.jfr
```

**When thread dumps ARE the right tool:** when the service is **stuck**, not slow. Three
dumps thirty seconds apart, compared by hand, will find a deadlock (Topic 94), a pool
exhaustion (Topic 109), or a thread leak (Topic 98) faster than any profiler. That is a
diagnostic, not a sampling campaign, and three dumps is three safepoints — acceptable.

```bash
for i in 1 2 3; do
  jcmd <pid> Thread.print > /dumps/threads-$i.txt
  sleep 30
done
grep -A5 "Found one Java-level deadlock" /dumps/threads-*.txt
grep -c "waiting to lock" /dumps/threads-*.txt
```

---

## Hands-on proof

Eight exercises. Commands plus what to look for.

### Setup — establish your tool versions. Do not take versions from this document.

```bash
java -version
jcmd -l
jfr --version 2>/dev/null || echo "jfr is part of the JDK; check \$JAVA_HOME/bin/jfr"

# async-profiler: install it, then ask IT what version it is.
ls /opt/async-profiler
/opt/async-profiler/bin/asprof --version
/opt/async-profiler/bin/asprof --help | head -40
```

> **I am not going to name an async-profiler version or a download URL.** The project
> releases regularly, the binary name changed historically (`profiler.sh` → `asprof`), and
> the option set differs across versions. **`--help` on your build is the authority.**
> If your build has `profiler.sh` rather than `asprof`, use that; the options in this
> document are the stable core ones (`-e`, `-d`, `-f`, `-o`, `-t`).

### Proof 1 — list the event types your build supports

```bash
./asprof list <pid>
```

**WHAT TO LOOK FOR:** which of `cpu`, `wall`, `alloc`, `lock`, `itimer`, `ctimer` and the
perf hardware events are available. If `cpu` is missing but `itimer` is present, perf
events are unavailable in this environment — that is your kernel-permissions answer,
found in one command.

### Proof 2 — CPU versus wall-clock, on the minimal example

Run Example 1. This is the proof that everything else in the document rests on. Do not
skip it because it looks trivial.

### Proof 3 — read a collapsed file as text, without a picture

```bash
./asprof -e cpu -d 30 -o collapsed -f /tmp/p.collapsed <pid>
wc -l /tmp/p.collapsed                                   # distinct stacks
awk '{s+=$NF} END {print s}' /tmp/p.collapsed            # total samples
sort -k2 -nr /tmp/p.collapsed | head -10                 # hottest stacks
```

**WHAT TO LOOK FOR:** that you can answer "where is the time" from the text alone. The
picture is a convenience; the data is a text file. Engineers who can read the text are
much faster in an incident, when nobody wants to open a browser against a jump host.

### Proof 4 — flame graph versus time-ordered view

```bash
./asprof -e cpu -d 30 -o flamegraph -f /tmp/p-flame.html <pid>
# Many builds also support a time-ordered rendering. Check yours:
./asprof --help 2>&1 | grep -iE "timeline|chart|order"
```

**WHAT TO LOOK FOR:** open the flame graph and try to determine *when* something happened.
You cannot. Confirming that inability, deliberately, is what fixes the DevTools instinct
that the x-axis is time.

### Proof 5 — allocation profiling, which is a different question again

```bash
./asprof -e alloc -d 60 -o flamegraph -f /tmp/alloc.html <pid>
```

**WHAT TO LOOK FOR:** which call paths allocate. This is a third axis alongside CPU and
wall-clock, and it is the profile that connects to Topics 68 and 70.

| What you see | What it means |
|---|---|
| Allocation dominated by your DTO mapping | Normal for a REST service. Compare against the GC log's allocation rate to see whether it matters |
| Allocation dominated by Hibernate entity hydration | You are loading entities you do not need. Projections (Topic 47) |
| Allocation dominated by `byte[]` in the HTTP or JDBC layer | Buffer churn. Check sizes and pooling; also check Topic 80 for the direct-buffer counterpart |
| Huge single allocations | Humongous-object territory under G1 (Topic 71). Cross-check `-Xlog:gc+humongous=debug` |

### Proof 6 — two profilers on the same workload (the bias check)

Run async-profiler and JFR against the same steady-state load, and compare the top frames.
Agreement is corroboration. Disagreement is a finding — and Trap 2 is the interpretation.

### Proof 7 — drive JFR entirely from `jcmd`

```bash
jcmd <pid> JFR.start name=inv settings=profile filename=/dumps/inv.jfr
jcmd <pid> JFR.check
jcmd <pid> JFR.dump name=inv filename=/dumps/inv-snapshot.jfr
jcmd <pid> JFR.stop name=inv

jfr summary /dumps/inv.jfr
jfr metadata /dumps/inv.jfr | head -60
jfr print --events jdk.ExecutionSample --stack-depth 12 /dumps/inv.jfr | head -80
jfr print --events jdk.GCPhasePause /dumps/inv.jfr | head -40
jfr print --events jdk.JavaMonitorEnter /dumps/inv.jfr | head -40
jfr print --events jdk.SocketRead /dumps/inv.jfr | head -40
```

**WHAT TO LOOK FOR in `jfr summary`:** the list of event types and their counts. This is
where JFR's breadth becomes obvious, and where you realise you can correlate a latency
spike against GC, safepoints, socket reads and lock contention **from one file**.

| Event type | The question it answers | Related topic |
|---|---|---|
| `jdk.ExecutionSample` | Where is Java code executing | This topic |
| `jdk.GCPhasePause` | GC pause durations and phases | 71, 72 |
| `jdk.SafepointBegin` / `jdk.SafepointEnd` | Stop-the-world including TTSP | 73 |
| `jdk.JavaMonitorEnter` | Which monitors are contended, and by whom | 85, 94 |
| `jdk.ThreadPark` | Where threads block on `LockSupport` | 89, 90 |
| `jdk.SocketRead` / `jdk.SocketWrite` | Network waits, with host and duration | 55, 109 |
| `jdk.ObjectAllocationSample` | Allocation hot paths | 68, 70 |
| `jdk.Compilation` / `jdk.Deoptimization` | JIT activity and deopts | 74 |
| `jdk.NativeMemoryUsage` (where available) | Native footprint | 80 |
| `jdk.VirtualThreadPinned` (where available) | Loom pinning | 101 |

> **Flagged uncertainty.** Event availability and names vary by JDK version, and some of
> the rows above may not exist on your build. **`jfr metadata <file>` is the authority for
> your JDK.** Do not assume an event exists because it is listed here.

### Proof 8 — turn a JFR recording into a flame graph

JFR does not render flame graphs itself. Two routes:

```bash
# Route A: async-profiler ships a converter (name varies by version — check your build).
ls /opt/async-profiler/lib /opt/async-profiler/bin | grep -i conv
java -cp /opt/async-profiler/lib/converter.jar jfr2flame /dumps/inv.jfr /dumps/inv.html

# Route B: extract folded stacks yourself from `jfr print` and render with any
# flame-graph tool that consumes the collapsed format.
```

> **Flagged.** The converter's jar name, main class and CLI have changed across
> async-profiler releases. **Check `ls` and `--help` on your build.** The concept —
> "JFR records events; a converter turns `jdk.ExecutionSample` events into folded stacks;
> a renderer turns folded stacks into SVG/HTML" — is stable; the invocation is not.

### Proof 9 — measure the profiler's own overhead, honestly

**This is the answer to "is it safe in production", and it is the answer nobody gives.**

```bash
# 1. Run the Topic 65 baseline with no profiling. Record percentiles.
k6 run loadtest/baseline.js --out json=/tmp/k6-clean.json

# 2. Same load, with JFR's default settings running continuously.
docker exec orderflow jcmd 1 JFR.start name=ovh settings=default filename=/dumps/ovh.jfr
k6 run loadtest/baseline.js --out json=/tmp/k6-jfr-default.json

# 3. Same load, with JFR's profile settings.
docker exec orderflow jcmd 1 JFR.start name=ovh2 settings=profile filename=/dumps/ovh2.jfr
k6 run loadtest/baseline.js --out json=/tmp/k6-jfr-profile.json

# 4. Same load, with async-profiler in CPU mode for the run's duration.
```

**WHAT TO LOOK FOR:** the p99 delta across the four runs, compared against the ±10% gate
tolerance from Topic 65.

| What you see | What it means |
|---|---|
| `settings=default` p99 indistinguishable from clean, within run-to-run variation | JFR's default template is safe to leave on continuously for this service. **You can now say so with evidence.** |
| `settings=profile` measurably worse than `default` | Expected — it enables more events and samples more often. Use it for windows, not continuously |
| async-profiler CPU mode measurably worse | Note the magnitude. It is usually acceptable for a 60-second window; it is not something to leave running |
| Everything within noise, including `profile` | Good news, and now it is a measurement rather than a vendor claim |
| Wildly variable results across runs | Your baseline is not reproducible; fix that first (Topic 65's ±10% gate rule) before drawing any conclusion about overhead |

**Write the result into `/docs/java/baselines/`.** "JFR default costs us `<n>`% of p99 on
this service" is a fact your team will reuse for years, and it is the sort of thing that
gets you listened to in an architecture review.

---

## Failure drill

> **From the master plan, Section G:** *Profile `orderflow` at baseline. Capture a flame
> graph, CPU versus wall-clock. **The two modes answer different questions.***

The deliverable is not a picture. It is **two written top-three lists and a paragraph
explaining why they differ.**

### Step 0 — preconditions, and do not proceed without them

- [ ] The Topic 65 baseline is reproducible within ±10% (the gate rule).
- [ ] `orderflow` runs with `-XX:+UnlockDiagnosticVMOptions -XX:+DebugNonSafepoints
      -XX:+PreserveFramePointer`, and the baseline was **re-recorded** with those flags on.
- [ ] async-profiler is installed in the container and `asprof --version` works.
- [ ] `perf_event_paranoid` allows CPU sampling, **or** you have accepted the itimer
      fallback and written that down.
- [ ] You have `/dumps` mounted so artefacts survive the container.

### Step 1 — reach and verify steady state

```bash
k6 run loadtest/baseline.js &
sleep 120
docker exec orderflow jcmd 1 Compiler.queue
docker exec orderflow jcmd 1 VM.uptime
docker exec orderflow jcmd 1 GC.heap_info
```

**WHAT TO LOOK FOR:** a near-empty compile queue and a stable GC rhythm. Write down the
uptime at which you started profiling — it goes in the report.

### Step 2 — the CPU profile

```bash
docker exec orderflow /opt/async-profiler/bin/asprof \
  -e cpu -d 60 -o flamegraph -f /dumps/drill-cpu.html 1
docker exec orderflow /opt/async-profiler/bin/asprof \
  -e cpu -d 60 -o collapsed  -f /dumps/drill-cpu.collapsed 1
```

### Step 3 — write the CPU top-three BEFORE looking at wall-clock

This ordering is the drill. **Commit to an answer from the CPU profile alone**, the way
you would if you did not know wall-clock mode existed — which is the state most engineers
are in.

Write `/dumps/drill-cpu-top3.md`:

```markdown
# CPU profile — top three (written before seeing the wall-clock profile)
Total samples: <n>. Duration 60s. Steady state at uptime <n>s.

1. <frame> — <n>% of request-thread samples. Why I think it is expensive: <...>
2. <frame> — <n>%. <...>
3. <frame> — <n>%. <...>

## My recommendation, based on CPU alone
<What you would do if this were the only profile you had.>

## My prediction for the wall-clock profile
<Write this down. Being wrong here is the most instructive outcome of the drill.>
```

The prediction matters. A prediction you record and then falsify teaches you something a
prediction you make silently never will.

### Step 4 — the wall-clock profile

```bash
docker exec orderflow /opt/async-profiler/bin/asprof \
  -e wall -t -d 60 -o flamegraph -f /dumps/drill-wall.html 1
docker exec orderflow /opt/async-profiler/bin/asprof \
  -e wall -t -d 60 -o collapsed  -f /dumps/drill-wall.collapsed 1

grep '^http-nio' /dumps/drill-wall.collapsed > /dumps/drill-wall-req.collapsed
awk '{s+=$NF} END {print "request-thread samples:", s}' /dumps/drill-wall-req.collapsed
sort -k2 -nr /dumps/drill-wall-req.collapsed | head -20
```

**Filter before reading.** If you skip the filter you will hit Trap 3 and conclude the
service spends its life in `LinkedBlockingQueue.take`.

### Step 5 — write the wall-clock top-three, and the explanation

Write `/dumps/drill-wall-top3.md` in the same shape, then the deliverable:

```markdown
# Why the two answers differ

## The two lists, side by side
| Rank | CPU profile | Wall-clock profile (request threads only) |
|---|---|---|
| 1 | <frame> <n>% | <frame> <n>% |
| 2 | <frame> <n>% | <frame> <n>% |
| 3 | <frame> <n>% | <frame> <n>% |

## The mechanism
CPU mode samples a thread only while it is consuming cycles. Our request threads spend
<n>% of their wall time blocked in <where>, and during all of that they produce zero CPU
samples. So the CPU profile describes only the on-CPU remainder of the request, and its
percentages have CPU time as their denominator — not request latency.

Wall-clock mode samples every thread on every tick regardless of state, so blocked time
appears as <frames>. Filtered to request threads, those percentages approximate a
breakdown of request latency.

## What my prediction got wrong, and why
<...>

## What each profile is good for, on THIS service
- CPU profile: <...>
- Wall-clock profile: <...>

## Recommendation, and how I will verify it
<Action, expected p99 effect in ms computed from the wall-clock share, and the plan to
re-run the Topic 65 baseline and compare against /docs/java/baselines/.>

## Ruled out
<Hypotheses this profiling excludes, with the evidence.>
```

### Step 6 — optional but strongly recommended: the corroboration run

```bash
docker exec orderflow jcmd 1 JFR.start name=drill settings=profile duration=60s filename=/dumps/drill.jfr
# ...wait...
jfr summary /dumps/drill.jfr
jfr print --events jdk.ExecutionSample --stack-depth 12 /dumps/drill.jfr | head -60
jfr print --events jdk.SocketRead /dumps/drill.jfr | head -40
jfr print --events jdk.JavaMonitorEnter /dumps/drill.jfr | head -40
```

**WHAT TO LOOK FOR:** whether JFR's `jdk.ExecutionSample` top frames agree with
async-profiler's CPU profile, and whether `jdk.SocketRead` durations corroborate the
wall-clock picture. Two independent tools agreeing is what turns a profile into evidence.

### What the drill proves

Four things. State each in one sentence:

1. **A profile answers one question, and you chose which one when you picked the mode.**
2. **For an I/O-bound service, the CPU profile's top method is usually not worth
   optimising** — not because the profile is wrong, but because its denominator is not
   request latency.
3. **Wall-clock mode is unusable without thread filtering**, and the idle pool is not a
   finding.
4. **The gap between the two profiles is your blocked time**, which makes the comparison
   itself a measurement rather than just a sanity check.

---

## Measurement

Profiling *is* measurement, so this section is about the discipline around it: how to
make a profile reproducible, how to quote it honestly, what is safe in production, and how
to connect a percentage to a millisecond.

### The five rules

**Rule 1 — a profile is only meaningful with its conditions recorded.**

A flame graph with no metadata is an unfalsifiable picture. Every profile you keep must be
accompanied by:

```bash
{
  echo "=== when ==="; date -Is
  echo "=== jvm ==="; docker exec orderflow java -version 2>&1
  echo "=== flags ==="; docker exec orderflow jcmd 1 VM.flags -all
  echo "=== uptime at profile start ==="; docker exec orderflow jcmd 1 VM.uptime
  echo "=== container ==="; docker inspect orderflow --format '{{.HostConfig.Memory}} {{.HostConfig.NanoCpus}}'
  echo "=== profiler ==="; docker exec orderflow /opt/async-profiler/bin/asprof --version
  echo "=== load ==="; echo "k6 scenario: baseline.js, open model, <n> arrivals/s"
  echo "=== dataset ==="; echo "100k products / 1M orders / 5M order lines"
} > /dumps/profile-metadata.txt 2>&1
```

Same discipline as Topic 65's `environment.md`, and for the same reason: a number without
its conditions is not reusable, and six months from now you will not remember.

**Rule 2 — profile at steady state, and prove you were at steady state.**

A JVM's first minute is class loading and JIT compilation (Topics 67, 74). A profile taken
then is a profile of startup, and it will point at `ClassLoader`, the compiler threads, and
static initialisers.

Evidence of steady state, all three:

```bash
docker exec orderflow jcmd 1 Compiler.queue     # near-empty
docker exec orderflow jcmd 1 GC.heap_info       # stable pattern across samples
# plus: k6's own throughput and latency flat for at least a minute
```

**Rule 3 — always take both modes, and treat the difference as data.**

Not "take CPU, and if it looks wrong take wall-clock". Take both, every time. The
comparison is cheap and the difference *is* your blocked-time measurement:

| CPU and wall-clock... | Conclusion |
|---|---|
| Agree | CPU-bound. Optimise computation. Check container CPU throttling first (Topic 82) |
| Disagree, wall-clock dominated by socket reads | I/O-bound. Attack queries, N+1s, round-trip counts (Topics 50, 55) |
| Disagree, wall-clock dominated by `getConnection` | Pool-bound. Pool size, transaction duration, nested `REQUIRES_NEW` (Topics 55, 109) |
| Disagree, wall-clock dominated by lock frames | Contention-bound. Topics 85, 94; corroborate with `-e lock` and `jdk.JavaMonitorEnter` |
| Disagree, wall-clock dominated by park in the request pool | Not saturated. The bottleneck is upstream — including possibly your load generator |

**Rule 4 — convert percentages to milliseconds before recommending anything.**

A flame-graph percentage is a fraction of samples. To make a decision you need a fraction
of *request latency*:

1. Take the **wall-clock profile filtered to request threads**. Its percentages
   approximate the breakdown of request-thread wall time, which is request latency.
2. Multiply by the endpoint's p99 from `/docs/java/baselines/`.
3. Compare the result to your improvement target and to the ±10% gate tolerance.

**Never do this arithmetic with a CPU profile's percentages.** CPU percentages have CPU
time as the denominator. Multiplying them by p99 systematically overstates the opportunity
— often by an order of magnitude — and it is the single most common analytical error in
this area.

**Rule 5 — corroborate before you act on a surprising result.**

If the profile names something you did not expect, run a second, independent tool before
you spend a week on it:

| Surprise | Corroborate with |
|---|---|
| A tiny JDK method is hot | async-profiler versus JFR (Proof 6) |
| GC threads dominate | `-Xlog:gc*`, allocation profile (`-e alloc`), Topics 70/71 |
| A method you cannot explain | `javap -c` (Topic 76) and `-XX:+PrintInlining` (Topic 75) |
| Socket reads dominate | Hibernate `Statistics` query counts (Topic 50), `pg_stat_statements` |
| Lock frames dominate | `-e lock`, JFR `jdk.JavaMonitorEnter`, a thread dump |
| A stall no profile explains | `-Xlog:safepoint*` — TTSP (Topic 73) is invisible to profilers |

### What is safe to run in production

| Practice | Verdict | Why |
|---|---|---|
| JFR with `settings=default`, continuously, `dumponexit=true` | **Yes**, after you have measured the overhead on your own service (Proof 9) | Designed for it. A recording already running when an incident starts is worth more than any tool you can attach afterwards |
| JFR with `settings=profile` for a bounded window | **Yes**, with `duration=` set so it cannot be forgotten | More events, more samples, more overhead |
| async-profiler in CPU mode for a 30–60s window on one instance | **Usually yes** — and take the instance out of the load-balancer rotation if you can | Needs the binary present and, on Linux, perf permissions |
| async-profiler in wall-clock mode in production | **Careful.** More samples, more threads, more overhead. Short windows only | The mode itself is fine; the sample volume is the issue |
| `-XX:+PreserveFramePointer` permanently in production | **A deliberate trade.** Small throughput cost, big observability gain | Decide it once, write it down, and keep it consistent between your baseline and your profiles |
| `-XX:+DebugNonSafepoints` permanently | **Generally yes** — it affects the debug metadata C2 emits, not the generated code's speed | Confirm on your own service with Proof 9 rather than trusting me |
| Thread dumps in a loop | **No.** Trap 5 | Every dump is a global safepoint |
| Attaching a full JVMTI profiler to production | **No** | Safepoint bias plus a real observer effect |
| Profiling the instance that is currently on fire | **Yes, with JFR**, if a recording is already running — dump it | This is the whole argument for continuous JFR |

### Building profiling into the Topic 65 workflow permanently

Make it a standing part of the baseline rather than a thing you do in a crisis:

```bash
#!/usr/bin/env bash
# scripts/profile-baseline.sh — run alongside every baseline capture
set -euo pipefail
OUT="/docs/java/baselines/$(date +%F)-run-01/profiles"
mkdir -p "$OUT"

docker exec orderflow jcmd 1 VM.flags -all         > "$OUT/vm-flags.txt"
docker exec orderflow jcmd 1 VM.uptime             > "$OUT/uptime-at-start.txt"

docker exec orderflow jcmd 1 JFR.start name=baseline settings=profile \
    duration=180s filename=/dumps/baseline.jfr

for MODE in cpu wall alloc; do
  docker exec orderflow /opt/async-profiler/bin/asprof \
      -e "$MODE" -d 60 -o collapsed -f "/dumps/baseline-$MODE.collapsed" 1
  docker exec orderflow /opt/async-profiler/bin/asprof \
      -e "$MODE" -d 60 -o flamegraph -f "/dumps/baseline-$MODE.html" 1
done

docker cp orderflow:/dumps/. "$OUT/"
echo "artefacts in $OUT"
```

**Why this is worth doing every time:** a profile from six months ago is how you answer
"when did this get slow, and what changed". Collapsed files are small text; commit them.
Diffing two collapsed files across releases is a genuine regression-detection technique
and almost nobody does it.

```bash
# Frames present now that were not present at the last baseline:
comm -13 <(cut -d' ' -f1 old/baseline-cpu.collapsed | sort -u) \
         <(cut -d' ' -f1 new/baseline-cpu.collapsed | sort -u) | head -40
```

### What profiling cannot tell you

Know the tool's blind spots as precisely as its strengths:

| Blind spot | What to use instead |
|---|---|
| Time-to-safepoint stalls | `-Xlog:safepoint*` (Topic 73). A profiler is *stopped* during a safepoint; it cannot see the stall that stopped it |
| Native memory growth with a healthy heap | NMT, `jcmd VM.native_memory` (Topic 80) |
| Retention — why memory is not being freed | Heap dump and MAT dominator tree (Topic 79) |
| Whether an alternative implementation would be faster | JMH (Topic 77) |
| Whether the system-level percentiles moved | The Topic 65 load baseline |
| Cross-service latency attribution | Distributed tracing (Topic 119) |
| Very rare events (a 1-in-10,000 slow request) | Sampling misses them almost by definition. Use tracing or JFR duration events with a threshold |

That last row deserves emphasis: **a sampling profiler is bad at rare events.** If your
p999 is the problem but your p50 is fine, sampling will show you the p50's shape. JFR's
threshold-based duration events (record only events longer than X) and distributed tracing
are the right tools for the tail.

---

## Practice exercises

### 1 — Easy: build your flame-graph reading sheet

**Goal:** be able to read a flame graph fluently, and know your own environment's
constraints before you need them in an incident.

**Do this:**

1. Run Example 1 and generate both profiles.
2. In the CPU flame graph, identify by pointing at the picture: the root frame, a frame
   with large **self** time, a frame with large **total** but small self time, and the
   widest top plateau.
3. Generate the collapsed file for the same run and find the same four things in the text.
4. Write down the answers to these, from your own commands:
   - Which async-profiler events does your build support? (`asprof list <pid>`)
   - Is `perf_event_paranoid` permissive enough for CPU sampling here?
   - Which JFR event types exist on your JDK? (`jfr metadata`)
   - Are `DebugNonSafepoints` and `PreserveFramePointer` on? (`jcmd VM.flags -all`)

**Deliverable:** a one-page reference with those four answers plus a five-line "how to read
a flame graph" you wrote yourself, in your own words.

**You have succeeded when:** you can say, without hesitating, why the x-axis is not time.

---

### 2 — Medium: the profile-driven audit
*(combines Topics 01, 11, 18, 21, 25, 47, 50, 68, 70, 71, 73, 76, 77)*

**Goal:** connect a profile to the specific curriculum claims it can confirm or refute.

**Do this against `orderflow` under the Topic 65 load:**

1. Take CPU, wall-clock and allocation profiles at steady state.
2. From the **allocation** profile, find the top three allocating call paths. For each,
   name the mechanism from an earlier topic: boxing (01), string building (18), a
   capturing lambda (21), entity hydration (47), collection resizing (11). Confirm at
   least one with `javap -c` (Topic 76).
3. From the **CPU** profile, pick the hottest method that is your own code. Microbenchmark
   it with JMH (Topic 77) — correctly, with `@Fork(3)`, a baseline and error bars.
4. From the **wall-clock** profile, count the distinct socket-read call paths into
   Postgres for a single `GET /orders/{id}`. Cross-check against Hibernate `Statistics`
   query counts (Topic 50). **If the numbers disagree, work out which one is lying and
   why.**
5. Take a GC log for the same window (`-Xlog:gc*`) and compute the allocation rate (Topic
   68/70). Compare it against the allocation profile's top paths: do the paths the
   profiler names plausibly account for the rate the GC log implies?
6. Take a safepoint log (`-Xlog:safepoint*`, Topic 73) for the same window and check
   whether any stop-the-world event is large enough to appear in your p99. **A profiler
   cannot see TTSP; this is the cross-check that covers its blind spot.**

**Deliverable:** a markdown report with a section per tool, and a final table mapping each
observation to the topic that explains it. Include at least one case where two tools
**disagreed**, and your resolution.

**You have succeeded when:** you can explain why the tools disagreed, rather than picking
the one you liked.

---

### 3 — Hard: production simulation — find a regression you did not plant
*(against the Topic 65 baseline; combines Topics 50, 55, 65, 71, 73, 77, 78, 109)*

**Setup — have someone else do this, or do it and then wait a week so you forget.** Have a
colleague introduce **one** of the following into `orderflow`, without telling you which:

- (a) an N+1 on the order-lines fetch (`FetchType.EAGER` on an association, Topic 49);
- (b) a 200 ms blocking HTTP call inside a `@Transactional` method (Topic 55);
- (c) a regex compiled per request in the catalogue filter;
- (d) a `synchronized` block around the inventory-decrement path (Topic 85);
- (e) a `static final` list that grows per request, driving allocation and promotion
      (Topics 68, 70);
- (f) a counted `int` loop over a large array in a scheduled job, causing TTSP (Topic 73).

**Your task, in this order, and each step timed:**

1. Re-run the Topic 65 baseline. Confirm from `/docs/java/baselines/` **which percentile
   on which endpoint** has regressed beyond ±10%. If nothing has, the change is invisible
   to your load test — a finding in itself.
2. **Predict**, from the shape of the regression alone (p50 versus p99 versus p999, which
   endpoints, error rate), which class of cause it is. Write the prediction down.
3. Take CPU and wall-clock profiles. See whether the prediction survives.
4. If neither profile explains it, escalate to the tools profilers cannot replace:
   `-Xlog:gc*` (Topics 70, 71), `-Xlog:safepoint*` (Topic 73), Hibernate `Statistics`
   (Topic 50), HikariCP pool metrics (Topic 109), a thread dump.
5. Name the change. Then fix it, re-run the baseline, and confirm the percentiles return
   to within ±10%.

**Deliverable:** an incident-style write-up: symptom, hypotheses considered and how each
was excluded, the tool that produced the decisive evidence, the fix, and verification
against the baseline. Include a section titled **"what I looked at first and why it was
wrong"** — that section is worth more than the rest.

**You have succeeded when:** for cause (f), you correctly conclude that **no profiler will
find it**, and go to the safepoint log. That is the case that separates people who own a
profiler from people who understand observability.

---

## Interview questions

### Q1 — "The profiler says `HashMap.get` is hot. Do you believe it?"

**MID-LEVEL answer:** *"If the profiler says so, probably — we must be doing a lot of map
lookups. I'd look for hot maps and consider a faster structure or caching the lookups."*

**SENIOR answer:** *"My first reaction is suspicion, and it's structural rather than about
this particular codebase.*

*Most classic Java profilers use JVMTI to walk stacks, and JVMTI needs the thread at a
safepoint. Compiled code only polls for safepoints at method returns and non-counted-loop
back-edges. `HashMap.get` is short and called from everywhere, so its return is a poll
point that an enormous number of code paths pass through — and it absorbs attribution
from the code around it. Meanwhile the thing I probably care about, a counted loop over an
array whose poll C2 elided, is sampled never. So safepoint bias produces exactly this
shape: a tiny, universal JDK method at a big percentage.*

*So I'd re-run with async-profiler, which samples via perf events or a signal and walks the
stack with `AsyncGetCallTrace`, so it isn't safepoint-biased — and I'd make sure
`DebugNonSafepoints` is on, because without it inlined frames get misattributed and I'd get
safepoint-flavoured results from an unbiased sampler.*

*Two more things before I acted even if the result survived. First: is this a CPU profile
of an I/O-bound service? If the service is mostly waiting on Postgres, `HashMap.get` might
genuinely be 30% of CPU while CPU is 8% of the request — a true statement about an
irrelevant denominator. Second: I'd look at the caller. A real `HashMap` hotspot has an
identifiable caller doing an identifiable amount of lookup work, and if I can't name it,
the attribution is probably an artefact."*

**What separates them:** naming the mechanism (JVMTI needs a safepoint; polls are at
returns and back-edges), naming the fix (AGCT plus `DebugNonSafepoints`), and then
*independently* raising the denominator problem. The mid-level answer trusts the tool; the
senior answer knows what class of tool it is.

**Follow-up:** *"async-profiler agrees. Now do you believe it?"*
→ *"Now it's a real finding, so I'd look down the flame graph to find the caller and up to
see whether it's a leaf. Then I'd check whether it's worth fixing: what fraction of request
latency is it, from a wall-clock profile? And I'd suspect the map's keys before the map —
an expensive `hashCode` or a poor hash distribution causing collisions is a much more
likely root cause than `HashMap` being slow (Topics 12, 13)."*

---

### Q2 — "We're I/O bound on Postgres. Which profiling mode, and what would you expect to see?"

**MID-LEVEL answer:** *"I'd profile it and look at the flame graph to see what's slow."*

**SENIOR answer:** *"Wall-clock mode, filtered to the request threads. CPU mode would show
me nothing useful, because a thread blocked in a socket read isn't consuming cycles and so
produces zero samples — the CPU profile would describe only the small on-CPU remainder of
the request.*

*In wall-clock I'd expect a dominant tower under the JDBC socket read, and the shape of
that tower is diagnostic. One wide read means a genuinely slow query — go to
`pg_stat_statements` and `EXPLAIN ANALYZE`. Many narrow reads means lots of round trips,
which is the N+1 signature, and I'd confirm it by counting queries with Hibernate
`Statistics` rather than by reading logs. If instead the tower is under
`HikariPool.getConnection`, that's not database slowness at all — it's pool starvation, and
the usual causes are an undersized pool, a long transaction holding a connection, or an
HTTP call inside a transaction.*

*Two things I'd be careful about. Wall-clock mode samples every thread regardless of state,
so the idle Tomcat pool will be the widest thing in the picture and mean nothing — I filter
by thread name first. And I'd take the CPU profile too, because the difference between the
two is a measurement of blocked time, not just a sanity check."*

**What separates them:** picking the mode with a mechanical justification, predicting the
*shape* and mapping shapes to distinct root causes, and pre-empting the idle-thread trap.

**Follow-up:** *"The wall-clock profile is 90% `LinkedBlockingQueue.take`. What now?"*
→ *"That's idle pool threads, not a finding. I filter to `http-nio` threads and re-read. If
after filtering the *request* threads are also parked in `getTask`, then the pool isn't
saturated and my bottleneck is upstream — the acceptor, a connection limit, or the load
generator itself. Topic 65 warns that a saturated load generator under-reports; this is what
that looks like from inside the JVM."*

---

### Q3 — "Can we profile production? What's the overhead?"

**MID-LEVEL answer:** *"JFR is low overhead, around 1%, so it's fine to leave on."*

The number is the problem. It is a number they read, not one they measured, and a good
interviewer will ask where it came from.

**SENIOR answer:** *"Yes, and I'd answer the overhead question with a measurement rather
than a figure from a slide.*

*The design intent is that JFR's default settings template is safe for continuous
production use — it's built into the JVM, it writes to thread-local buffers, and it needs
no external binary or kernel permissions. But the actual overhead depends on the workload,
the event set and the sampling interval, so I'd measure it on our service: run the Topic 65
baseline clean, then with `settings=default`, then with `settings=profile`, and compare
p99 against our ±10% gate tolerance. That gives me a number I can defend, specific to us,
and I'd record it in the baselines directory.*

*Operationally, I'd have a JFR recording running continuously with `dumponexit=true` and a
bounded size, because a recording that is already running when an incident starts is worth
more than any tool I can attach afterwards. For a deeper look I'd use `settings=profile`
for a bounded window with `duration=` set, so it can't be left on by accident.*

*async-profiler in production is a narrower proposition: it needs the binary present and,
for CPU-cycle sampling, permissive `perf_event_paranoid`, which many managed Kubernetes
environments won't give you. When it's available I'd use it for a 30–60 second window on
one instance, ideally out of the load-balancer rotation.*

*And the thing I would not do is take thread dumps in a loop as a poor man's profiler.
Every dump is a global safepoint, so at ten a second you've added a stop-the-world event
ten times a second and made the latency you're investigating worse."*

**What separates them:** replacing a memorised percentage with a measurement plan, knowing
JFR's operational shape (continuous default plus bounded profile windows), knowing the
container permission constraint, and knowing the anti-pattern.

**Follow-up:** *"Your measurement shows `settings=profile` costs 8% of p99. Ship it?"*
→ *"Not continuously. 8% is a real cost and it's close to our ±10% gate, which means it
would also contaminate baseline comparisons. I'd run `default` continuously and switch to
`profile` for bounded windows during an investigation. And I'd note the 8% in the baselines
directory so nobody re-measures it next year."*

---

### Q4 — "Walk me through this flame graph." (You are shown one.)

**MID-LEVEL answer:** *"The wide bars are where the time goes, so this method here is the
bottleneck."*

**SENIOR answer:** *"Before I read it, four questions about how it was made, because the
answers change the interpretation completely.*

*One: CPU or wall-clock? That determines whether percentages are of CPU time or of elapsed
time, and therefore whether I'm allowed to multiply them by p99.*

*Two: which threads are in it? If it's everything, GC threads, compiler threads and an idle
pool are diluting the request threads.*

*Three: was it taken at steady state? If it includes the first minute, I'm looking at class
loading and JIT compilation.*

*Four: is it an aggregated flame graph or a time-ordered chart? On an aggregated graph the
x-axis is alphabetical, so left-to-right position means nothing and I can't see phases.*

*Then I read it bottom-up. Bottom frames are thread roots. I look for wide plateaus at the
top, because a frame with nothing above it is where the CPU actually was — a wide frame
with a wide child isn't itself expensive, it's just on the path. Then I follow the widest
plateau downwards to find who called it, which is usually where the fix lives: the leaf is
where the cost is, the caller is where the decision was.*

*I'd also look for what's missing. If I know a method is on this path and it isn't in the
graph, either it was inlined and I'm missing `DebugNonSafepoints`, or the stack walk is
failing — and a large `[unknown_Java]` plateau tells me the same thing."*

**What separates them:** interrogating the provenance before the content, the
inclusive-versus-self distinction, and looking for absent frames. The last one is rare and
very senior.

**Follow-up:** *"What if the widest frame is `Unsafe.park`?"*
→ *"Then this is a wall-clock profile and I'm probably looking at idle threads. I'd filter
to the request threads. If it survives filtering, I look at what's *below* `park`: a
`ThreadPoolExecutor.getTask` means idle; an `AbstractQueuedSynchronizer` under a lock means
contention; a `HikariPool.getConnection` means pool starvation. `park` is not an answer —
its caller is."*

---

### Q5 — "Our p999 has 2-second spikes. No profile shows anything. What now?"

**MID-LEVEL answer:** *"Profile for longer to catch the spike, or increase the sampling
rate."*

**SENIOR answer:** *"More sampling won't help, and it's worth saying why: a sampling
profiler is stopped during a stop-the-world pause, so the very stall I'm chasing is the
thing it structurally cannot observe. Sampling is also bad at rare events by construction
— it shows me the shape of the common case.*

*So I'd go to the tools that record events rather than sample:*

*First, the safepoint log — `-Xlog:safepoint*`. Total stop-the-world time is time-to-
safepoint plus time-at-safepoint, and only the second is in the GC log. A 5 ms GC pause can
sit inside a 900 ms stop-the-world event caused by one thread in a counted `int` loop
failing to reach a poll for the whole loop. That's Topic 73, it hits every endpoint at the
same instant including ones doing no work, and no profiler will ever show it.*

*Second, the GC log itself — `-Xlog:gc*` — for evacuation failure, humongous allocations,
or a concurrent cycle losing the race.*

*Third, a JFR recording, because it records duration events with thresholds. A slow socket
read or a long monitor-enter is a recorded event with a duration, not something I have to
be lucky enough to sample.*

*Fourth, outside the JVM: cgroup CPU throttling, which produces exactly this shape in a
container with a CPU quota, and swapping. `cat /sys/fs/cgroup/cpu.stat` and `vmstat`.*

*The general principle is that spikes are event problems and profilers are distribution
tools. Match the instrument to the shape of the question."*

**What separates them:** knowing the profiler's structural blind spot, naming TTSP
specifically, and reaching for event-based rather than sample-based instruments. Naming the
container-throttling possibility shows they think past the JVM.

**Follow-up:** *"The safepoint log shows a large 'reaching safepoint' time. Who is
responsible?"*
→ *"The application, or the OS. I'd turn on `-XX:+SafepointTimeout
-XX:SafepointTimeoutDelay=500`, which names the straggler thread. Then I'd look for a
counted `int` loop over a large array, a big `System.arraycopy`, a large array allocation
being zeroed, or a JNI critical section — and if none of those, cgroup throttling or
swapping stopping the thread from running at all."*

---

## Mental model checkpoint

1. **State the two independent axes along which profilers differ, and give the failure
   mode of getting each one wrong.**
   *(Mechanical statement. Axis 1: safepoint-biased versus not — wrong answer, named
   confidently. Axis 2: CPU versus wall-clock — right answer to the wrong question.)*

2. **Why does taking ten times as many samples not fix a safepoint-biased profiler?**
   *(Machine-level reality §1. Bias is systematic, not random. More samples tighten the
   confidence interval around the wrong answer.)*

3. **In a flame graph, what does the x-axis mean? What does frame width mean? Where is
   self time?**
   *(Machine-level reality §3. X-axis is alphabetical, not time. Width is inclusive
   samples. Self time is the part of a frame with nothing stacked on it.)*

4. **Your service is at 15% CPU with a 200 ms p99. You take a CPU profile and it names a
   method at 40%. What is that 40% a fraction of, and what may you not do with it?**
   *(Trap 1. A fraction of CPU time, not of latency. You may not multiply it by p99.)*

5. **Why is a wall-clock profile 90% `Unsafe.park` on a healthy service, and what do you
   do about it?**
   *(Trap 3. Idle pool threads are sampled every tick. Filter by thread name.)*

6. **Name one thing a profiler structurally cannot see, and the tool that sees it.**
   *(TTSP → `-Xlog:safepoint*`, Topic 73. Also acceptable: retention → heap dump, Topic 79;
   native memory → NMT, Topic 80.)*

7. **What does `-XX:+DebugNonSafepoints` do, and what goes wrong without it even when you
   are using an unbiased sampler?**
   *(Machine-level reality §1. It makes C2 emit debug metadata at non-safepoint locations.
   Without it, inlined frames are misattributed, so an unbiased sampler produces
   safepoint-flavoured attribution — the worst case, because you think you are safe.)*

---

## Quick reference card

### async-profiler

```bash
./asprof --version
./asprof list <pid>                                   # supported events
./asprof -e cpu   -d 60 -f out.html   <pid>           # CPU flame graph
./asprof -e wall  -d 60 -t -f out.html <pid>          # wall-clock, split by thread
./asprof -e alloc -d 60 -f alloc.html <pid>           # allocation
./asprof -e lock  -d 60 -f lock.html  <pid>           # contention
./asprof -e cpu -d 60 -o collapsed -f out.collapsed <pid>   # folded stacks (grep-able)
./asprof --help | grep -iE "thread|include|exclude"   # filtering options on YOUR build
```

### JFR

```bash
# At startup
-XX:StartFlightRecording=name=base,settings=default,filename=/dumps/x.jfr,dumponexit=true
-XX:StartFlightRecording=name=deep,settings=profile,duration=120s,filename=/dumps/d.jfr

# At runtime
jcmd <pid> JFR.start name=inv settings=profile duration=60s filename=/dumps/inv.jfr
jcmd <pid> JFR.check
jcmd <pid> JFR.dump  name=inv filename=/dumps/snap.jfr
jcmd <pid> JFR.stop  name=inv

# Reading
jfr summary  x.jfr
jfr metadata x.jfr
jfr print --events jdk.ExecutionSample --stack-depth 12 x.jfr
jfr print --events jdk.GCPhasePause,jdk.SafepointBegin x.jfr
jfr print --events jdk.SocketRead,jdk.JavaMonitorEnter x.jfr
```

### JVM flags that make profiling accurate

```bash
-XX:+UnlockDiagnosticVMOptions -XX:+DebugNonSafepoints   # inlined-frame attribution
-XX:+PreserveFramePointer                               # native stack walking
# verify:
jcmd <pid> VM.flags -all | grep -E "DebugNonSafepoints|PreserveFramePointer"
```

### Environment checks before you profile

```bash
cat /proc/sys/kernel/perf_event_paranoid    # <=1 for unprivileged perf sampling
cat /proc/sys/kernel/kptr_restrict
jcmd <pid> Compiler.queue                   # steady state?
jcmd <pid> VM.uptime
docker stats --no-stream <container>
cat /sys/fs/cgroup/cpu.stat                 # nr_throttled (Topic 82)
```

### Reading a collapsed file

```
<thread-or-root>;<frame>;<frame>;<frame> <n>
```
*illustration of the format, not captured output — structure only*

```bash
awk '{s+=$NF} END {print s}' p.collapsed             # total samples
sort -k2 -nr p.collapsed | head -20                  # hottest stacks
grep '^http-nio' p.collapsed > req.collapsed         # filter to request threads
grep -o 'com\.orderflow\.[A-Za-z0-9_.$]*' p.collapsed | sort | uniq -c | sort -rn | head
comm -13 <(cut -d' ' -f1 old.collapsed | sort -u) \
         <(cut -d' ' -f1 new.collapsed | sort -u)    # new frames since last baseline
```

### Mode selection

| Symptom | Mode | Then |
|---|---|---|
| CPU pegged at 100% | `cpu` | Optimise the hot leaf; check container throttling first |
| High latency, low CPU | `wall` + thread filter | Find the blocking frame |
| Heap growing, GC frequent | `alloc` | Cross-check `-Xlog:gc*` (Topics 70, 71) |
| Threads blocked on each other | `lock` + JFR `jdk.JavaMonitorEnter` | Topics 85, 94 |
| Periodic global stall | **Not a profiler.** `-Xlog:safepoint*` | Topic 73 |
| Memory not released | **Not a profiler.** Heap dump + MAT | Topic 79 |
| RSS grows, heap flat | **Not a profiler.** NMT | Topic 80 |

### Gotchas checklist

- [ ] Did I take **both** CPU and wall-clock, and compare them?
- [ ] Did I filter wall-clock to the request threads?
- [ ] Was the JVM at steady state (empty compile queue, stable GC)?
- [ ] Are `DebugNonSafepoints` and `PreserveFramePointer` on?
- [ ] Is there a large `[unknown_Java]` plateau I am ignoring?
- [ ] Am I about to multiply a **CPU** percentage by p99? (Do not.)
- [ ] Did I record the metadata — flags, JDK, profiler version, load, dataset?
- [ ] Did I corroborate a surprising result with a second tool?
- [ ] Did I keep the collapsed file, not just the HTML?
- [ ] Am I sure this is not a safepoint problem the profiler cannot see?

---

## When would I use this at work?

**1. The first hour of a latency investigation, instead of the first day of guessing.**
A ticket says "checkout is slow". The channel fills with hypotheses. You take a wall-clock
profile at steady state, filter to request threads, and post two sentences with a picture
attached: *"78% of request-thread time is in a socket read to Postgres, in many small
reads rather than a few large ones — that's an N+1 shape. Confirming with Hibernate
`Statistics` now."* You have converted an argument into an investigation in under an hour,
and you have excluded three hypotheses along the way.

**2. Deciding whether an optimisation is worth doing — before doing it.**
Someone proposes a two-week rewrite of the serialisation layer. You profile, find that
serialisation is a modest share of CPU and CPU is a small share of the request, compute the
ceiling in milliseconds against the p99 baseline, and show that a perfect implementation
moves p99 by less than the load test can resolve. Two weeks returned to the roadmap. This
is the same senior move as Topic 77's, one step earlier in the chain — and profiling is
what makes the microbenchmark worth running at all.

**3. Making production observable before you need it.**
Turn on a continuous JFR recording with `dumponexit`, measure its overhead against your own
baseline so the number is yours rather than a vendor's, and store collapsed profiles
alongside every load-test baseline. Then, when the incident happens at 02:00, you have a
recording from before the incident and a profile from last month to diff against. The work
that makes an incident short is done months earlier, and this is most of it.

---

## Connected topics

**Prerequisites:**

- **21 / 22 — Lambdas and method references:** the `lambda$method$0` frames you will see
  everywhere in a flame graph are real synthetic method names (Topic 76 shows you why).
  Recognising them stops you hunting for a method that does not exist in your source.
- **25 — Parallel streams:** `ForkJoinPool.commonPool` frames in a profile are the visible
  evidence of that topic's warnings, and their thread names tell you who is using the
  shared pool.
- **47 / 49 / 50 — Spring Data, lazy associations, N+1:** the many-narrow-socket-reads
  shape in a wall-clock profile *is* an N+1. This topic is where you learn to recognise it
  by sight; Topic 50 is where you prove it with a query counter.
- **55 / 109 — Isolation and the connection pool:** a `HikariPool.getConnection` tower is
  pool starvation, not slowness. Completely different fix, and only wall-clock mode shows
  it to you.
- **65 — The load-testing gate:** you profile at the baseline, at steady state, and you
  convert profile percentages to milliseconds using the baseline's percentiles. Without
  Topic 65 this topic has nothing to profile.
- **66 / 67 — JVM architecture and class loading:** why a profile taken in the first minute
  is a profile of class loading.
- **68 / 70 — TLABs, allocation, GC fundamentals:** the allocation profile's counterpart.
  Allocation rate from the GC log; allocation *paths* from `-e alloc`.
- **71 / 72 — G1 and low-pause collectors:** GC threads appearing in a profile, and the
  correlation between GC events and request-thread stalls.
- **73 — Safepoints and TTSP:** **the most important connection in this document.** It
  explains the bias mechanism, and it is the blind spot no profiler covers. A stall that no
  profile explains is very often a safepoint story.
- **74 — Tiered compilation:** why steady state matters, and why compiler threads in your
  profile mean you profiled too early.
- **75 — Escape analysis and inlining:** why inlined frames vanish without
  `DebugNonSafepoints`, and why a caller can appear to have impossible self time.
- **76 — Reading bytecode:** when a profile names a method you cannot explain, `javap -c`
  tells you what the compiler emitted there.
- **81 — Instrumentation agents:** an APM agent rewrites bytecode at class load, changes
  method sizes and can change inlining decisions. That is a real observer effect, and it is
  worth knowing before you conclude a release made things slower.
- **82 — Containers and JVM tuning:** CPU throttling produces latency spikes that look like
  nothing in a profile. `cat /sys/fs/cgroup/cpu.stat` before you blame the code.

**This unlocks:**

- **77 — JMH:** the step *after* this one. Profile to choose what to benchmark; benchmark
  to quantify what the profile named; re-run the Topic 65 baseline to prove it moved.
- **79 — Memory leaks and heap dumps:** the sibling. A profiler tells you where time goes;
  a heap dump tells you what is retained. They answer different questions and neither
  substitutes for the other.
- **80 — Off-heap and NMT:** when a profile shows nothing and RSS is still growing, the
  problem is not in the heap and not in the CPU.
- **90 — `ExecutorService` and pool sizing:** wall-clock profiles of pool threads are how
  you see queueing, starvation and rejection in practice rather than in theory.
- **95 / 96 — Atomics and false sharing:** `-e lock` mode and hardware counters are how you
  see contention; the flame graph shows you where it lives.
- **101 — Virtual threads:** profiling changes shape entirely when threads are cheap and
  numerous. Pinning shows up as a JFR event; wall-clock profiling of millions of virtual
  threads is a different problem.
- **118 — Micrometer metrics:** profiles are for investigation, metrics are for detection.
  A profile tells you why; a metric tells you when. You need both, and confusing them
  leads to cardinality disasters.
- **119 — Distributed tracing:** the cross-service counterpart. A trace tells you which
  service; a profile tells you where inside it.
- **129 — Capacity, cost and latency budgets:** the wall-clock breakdown of a request is the
  input to a latency budget, and CPU-share-per-request is the input to a capacity model.

---

*Java baseline 21, running on JDK 25. Several things in this document are deliberately
hedged rather than asserted, each with a settling command given inline: the exact bias
characteristics of JFR's execution sampler on your specific JDK build (compare against
async-profiler — Proof 6), the status and naming of any specified successor to
`AsyncGetCallTrace` (check the JEP index at openjdk.org/jeps/0), async-profiler's version,
binary name and option spellings (`--version` and `--help` on your build), the JFR-to-flame-
graph converter's jar name and main class (`ls` your async-profiler installation), which
JFR event types exist on your JDK (`jfr metadata`), and whether any JDK or profiler
combination enables `DebugNonSafepoints` implicitly (set it explicitly and verify with
`jcmd VM.flags -all`).*

*Everything else here — that JVMTI stack walking requires a safepoint and is therefore
biased, that async-profiler samples via signals and walks stacks with `AsyncGetCallTrace`
without a safepoint, that CPU mode produces zero samples for a blocked thread, that JFR is
event-based with a typed schema you can print, and that a flame graph is built from folded
stacks sorted alphabetically so its x-axis is not time — is documented behaviour with a
confirming command above.*

*No profiler output was captured for this document. There is no flame graph here, no
percentage attributed to any method, no sample count and no overhead figure. Every number
you quote from this topic must come from your own run, with the metadata attached.*
