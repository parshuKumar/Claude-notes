# 82 — JVM Tuning and Container Awareness

## Phase: 8 — JVM Internals
## Category: ELITE
## Java baseline: 21  |  Notes features from: 21 / 25
## Project spine: decides `orderflow`'s heap size, collector and CPU allocation from measured numbers instead of rules of thumb — and explains why the same image that meets the Topic 65 baseline on your laptop is a different runtime entirely inside a `--cpus=0.5 --memory=512m` container.

---

## Mechanical statement

Read this twice. Everything below is elaboration.

> A modern JVM **reads the cgroup limits of the container it is running in** and derives
> two numbers from them: **`Runtime.availableProcessors()`** and the **default maximum
> heap size**.
>
> That processor count is then used to size **almost everything else the runtime builds
> for itself**: the number of parallel GC threads, the number of concurrent GC threads,
> whether the machine counts as "server class" and therefore **which collector is
> selected**, the parallelism of `ForkJoinPool.commonPool()`, the parallelism of the
> virtual-thread scheduler, the number of JIT compiler threads, and the default thread
> counts of most libraries you did not configure.
>
> **A CPU quota is therefore not a throttle. It silently reconfigures the runtime.**
>
> `--cpus=0.5` makes `availableProcessors()` return **1**. The common ForkJoinPool's
> parallelism is `availableProcessors() - 1`, floored at zero — so it has **no worker
> threads at all**, and every parallel stream in your process runs **serially on the
> calling thread**. Nothing throws. Nothing warns. Your `.parallel()` calls are still
> there in the source, and they do nothing.

The second half of the statement is the memory half:

> The default max heap is a **percentage of the container limit**, not the limit. Setting
> `-Xmx` equal to the container limit means metaspace, the code cache, thread stacks, GC
> structures, direct buffers and every native library are **unfunded by construction** —
> and the kernel kills you for it with no `OutOfMemoryError`, no heap dump, and exit code
> 137. That is Topic 80's RSS equation, and this document is where you turn it into
> numbers.

---

## The bridge from what you know

You know cgroups. You know CPU quota and CPU shares. You know what an OOMKill is, what
exit code 137 means, and what `resources.limits` does in a pod spec. **None of that is
what this document teaches.** What is new is the specific chain from those limits into the
JVM's own internal sizing, and how far that chain reaches.

### The PARTIAL analogue: `--max-old-space-size`

You have almost certainly written this line, or reviewed it:

```dockerfile
ENV NODE_OPTIONS="--max-old-space-size=1536"
```

You wrote it because Node would not size its heap from the container limit for you. V8's
default old-space size has historically been derived from the machine's total physical
memory, which inside a container is the **host's** memory, not the cgroup limit. So a Node
service in a 2 GB container on a 64 GB node would happily let V8 grow toward a limit
derived from 64 GB, and get OOMKilled.

**What transfers exactly:**

| Fact | Node | Java |
|---|---|---|
| The runtime must be told, or must infer, the container's memory limit | you tell it, manually | the JVM **infers** it from cgroups |
| Getting it wrong ends in an OOMKill, not an in-process error | yes | yes |
| The managed heap is only part of RSS | yes (external, buffers, native modules) | yes (Topic 80's equation) |
| The default is dangerous if you leave it alone | yes | **yes, differently** — see below |

> **Honest uncertainty, one line:** Node's container awareness has changed across
> versions and I will not assert what your Node build does today. It does not matter for
> this document; the point is the habit you already have.

### Where Java is *more* dangerous than Node, and why

Node's failure mode was **obvious**: no automatic sizing, so everyone learned to set the
flag, and the flag is in every Dockerfile you have ever read.

Java's failure mode is **subtle**, because the JVM *does* size itself automatically. It
reads the cgroup limit and picks a number. It just picks a number that is frequently wrong
for the situation — because it is designed to be safe for a machine that might be running
several processes, and your container exists to run exactly one JVM.

**A wrong automatic answer is more dangerous than no answer at all**, because nobody goes
looking for it. `-Xmx` absent from the command line reads as "we're using the defaults,
that's fine". It is not fine, and Trap 4 is what it costs.

### What has NO analogue: the processor-count cascade

This is the part with no Node equivalent, and it is the payload of the document.

In Node, `os.cpus().length` is **just a number you can read**. If your code chooses to
read it and size a worker pool from it, that is your decision, in your code, visible in
your diff. Nothing in the runtime reconfigures itself based on it. Node has one event loop
thread whether the box has 1 core or 128. The libuv thread pool defaults to 4 threads,
fixed, regardless of core count, until you set `UV_THREADPOOL_SIZE`.

In Java, `Runtime.availableProcessors()` is an **input to the runtime's own construction**.
It is consulted before your first line of code runs, by machinery you did not write and
cannot see, and it changes:

- how many threads collect your garbage,
- **which collector is chosen at all**,
- how many threads compile your hot methods,
- how many threads exist in the JVM-global work-stealing pool that every parallel stream
  and every `CompletableFuture.supplyAsync(...)` without an explicit executor uses,
- how many carrier threads schedule your virtual threads,
- and the defaults of most libraries that size a pool.

**Verdict: NO ANALOGUE.** Say exactly this in an interview: *"In Node, core count is a
number my code may choose to read. In the JVM, it is a construction parameter for the
runtime itself — so a CPU quota does not just slow the process down, it builds a different
process."*

### The pod spec you already know, read through this lens

```yaml
resources:
  requests:
    cpu: "500m"          # -> cgroup cpu.weight (v2) / cpu.shares (v1). A SCHEDULING HINT.
    memory: "512Mi"      # -> scheduling only. Not a limit.
  limits:
    cpu: "1"             # -> cgroup cpu.max quota. THE JVM READS THIS.
    memory: "1Gi"        # -> cgroup memory.max. THE JVM READS THIS, and the kernel enforces it.
```

You already knew every line. The new fact is the third column. **`limits.cpu` is an input
to the JVM's own architecture**, and `requests.cpu` — on modern JDKs — usually is not. Which
means the most common Kubernetes configuration of all, *requests set, limits omitted*, has a
specific and surprising consequence for a JVM. That is Trap 3.

---

## What is this?

"Container awareness" is a set of behaviours, on by default, in which the JVM inspects the
cgroup filesystem instead of asking the kernel about the whole machine.

```
-XX:+UseContainerSupport      (default: ON)
-XX:-UseContainerSupport      (turn it OFF - almost never what you want)
```

When it is on, the JVM reads, roughly:

| What the JVM wants to know | cgroup v1 file | cgroup v2 file |
|---|---|---|
| memory limit | `memory/memory.limit_in_bytes` | `memory.max` |
| memory usage | `memory/memory.usage_in_bytes` | `memory.current` |
| swap limit | `memory/memory.memsw.limit_in_bytes` | `memory.swap.max` |
| CPU quota | `cpu/cpu.cfs_quota_us` | `cpu.max` (first field) |
| CPU period | `cpu/cpu.cfs_period_us` | `cpu.max` (second field) |
| CPU shares/weight | `cpu/cpu.shares` | `cpu.weight` |
| CPU set (pinning) | `cpuset/cpuset.cpus` | `cpuset.cpus.effective` |

Two important cgroup-v2 details:

- **`cpu.max` is a single file with two fields**, `<quota> <period>`, and the quota may be
  the literal string `max`, meaning "no quota". So an unlimited container reads `max 100000`
  and the JVM concludes there is no CPU limit.
- **`memory.max` may also read `max`**, meaning no limit — in which case the JVM falls back
  to the host's physical memory, which is the whole problem Node has.

Cgroup v2 (the "unified hierarchy") is the default on modern distributions and on modern
Kubernetes. Support for it landed in the JDK during the 15/16 era and is present in 21 and
25; both v1 and v2 are handled.

> **Honest uncertainty, one line:** I am confident both hierarchies are supported on 21 and
> 25, and confident about the file names above; I am **not** going to assert which
> hierarchy your host uses or how your runtime maps a pod spec onto it. **Settle it in ten
> seconds inside the container:** `stat -fc %T /sys/fs/cgroup` — `cgroup2fs` means v2,
> `tmpfs` means v1 — and then `cat /sys/fs/cgroup/cpu.max /sys/fs/cgroup/memory.max` (v2)
> or the v1 equivalents.

### The one command that shows you what the JVM concluded

```bash
java -Xlog:os+container=trace -version
```

This prints the JVM's container detection, decision by decision: which hierarchy it found,
which files it read, what it parsed out of them, and what it decided the memory limit and
active processor count are. **This is the single most useful command in this document.**
When a container behaves differently from your laptop, this is the first thing to run, and
it usually ends the investigation.

Its calmer sibling:

```bash
java -XshowSettings:system -version
```

which prints an "Operating System Metrics" block (Linux only) with the provider, the
effective CPU quota, period and shares, the memory limit, and more.

### The two derived numbers

**1. Active processor count → `Runtime.availableProcessors()`.**

The JVM computes an *active processor count* by taking the most restrictive of the
constraints it can see:

- the CPU **affinity mask** (`sched_getaffinity`) — how many CPUs this process may run on
  at all, which is what `cpuset`/`--cpuset-cpus` sets;
- the CPU **quota**, as `ceil(quota / period)` — what `--cpus=N` and `limits.cpu` set;
- historically, a value derived from CPU **shares**.

**The `ceil` is not a detail. It is the whole drill.** `--cpus=0.5` produces
`quota=50000, period=100000`, so `0.5`, and `ceil(0.5) = 1`. The JVM cannot report a
fractional processor; `availableProcessors()` returns an `int` and its minimum useful value
is 1. So half a CPU and one CPU look **identical** to every piece of JVM machinery
downstream — while the scheduler gives the half-CPU container half the wall-clock CPU time.
That mismatch is a large part of why small containers behave badly.

`--cpus=1.5` gives `ceil(1.5) = 2`. `--cpus=2.1` gives `3`. The rounding is always up.

> **Honest uncertainty, one line, and the brief told me to be careful here:** the treatment
> of `cpu.shares` in the processor-count calculation has **changed across JDK releases** —
> older JDKs derived a count from shares (with `-XX:+PreferContainerQuotaForCPUCount`
> deciding precedence when both were set), and more recent JDKs stopped using shares for
> this purpose. **I am not certain which behaviour your JDK build has.** Settle it, do not
> trust me: run `java -Xlog:os+container=trace -version` inside a container with **only**
> `requests.cpu` set (shares, no quota) and read what it says the active processor count is,
> and run `java -XX:+PrintFlagsFinal -version | grep -i ActiveProcessorCount` to see whether
> the override flag is at its default.

The override:

```
-XX:ActiveProcessorCount=<n>
```

This wins over every derivation. It is the flag you reach for when the JVM's answer is
wrong for your workload — and, importantly, it is the flag that lets you say
"`limits.cpu: 500m` for the scheduler, but treat this as 2 processors for pool sizing"
when you have measured that the workload is I/O-bound and benefits from it.

**2. Default maximum heap → a percentage of the memory limit.**

The modern flags are percentage-based:

```
-XX:MaxRAMPercentage=<double>       # max heap as a % of the detected memory limit
-XX:InitialRAMPercentage=<double>   # initial heap as a % of the detected memory limit
-XX:MinRAMPercentage=<double>       # used INSTEAD of MaxRAMPercentage for small memory limits
-XX:MaxRAM=<bytes>                  # override what the JVM believes "total memory" is
```

The commonly-cited defaults are **`MaxRAMPercentage` = 25.0**, **`InitialRAMPercentage`
≈ 1.5625** (that is 1/64), and **`MinRAMPercentage` = 50.0**, where `MinRAMPercentage`
applies instead of `MaxRAMPercentage` when the detected physical memory is below a
threshold in the region of 256 MB.

> **Honest uncertainty, one line, and the brief told me to be especially careful here:**
> those three numbers match my understanding of long-standing HotSpot behaviour, and 25% is
> the figure quoted everywhere — but defaults move between releases, `MinRAMPercentage`'s
> threshold in particular is a detail I would not stake a production heap size on, and
> `-XX:MaxRAM` interacts with all of them. **Settle it on your JDK in five seconds:**
> ```bash
> java -XX:+PrintFlagsFinal -version | grep -i ramp
> java -XX:+PrintFlagsFinal -version | grep -iE "MaxHeapSize|InitialHeapSize|MaxRAM"
> ```
> and on a live process, `jcmd <pid> VM.flags -all | grep -iE "ramp|HeapSize"`, which also
> shows each flag's **origin** — `default`, `command line`, `ergonomic`, or `environment`.
> The origin column is the part people miss, and it answers "did I set this or did the JVM?"

**Take the 25% figure seriously as a problem, not as a fact to memorise.** In a 512 MB
container, 25% is 128 MB of heap — for a Spring Boot application with Hibernate, an
entity graph, a connection pool and a Tomcat thread pool. In a 2 GB container it is 512 MB,
which for `orderflow` at the Topic 65 dataset may or may not be enough, and you will not
know until you measure the live set (Topic 70).

The default is a **conservative multi-tenant default**. Your container is not multi-tenant.
It exists to run one JVM. **You should set the number, from measurement.**

### The legacy form you will meet in old configs

```
-XX:MaxRAMFraction=2        # heap = totalMemory / 2. Deprecated/obsolete.
-XX:MaxRAMFraction=4        # heap = totalMemory / 4.
```

Fractions are integers, so they could only express 1/1, 1/2, 1/3, 1/4… — there was no way
to say "70%". That is why the percentage flags exist. If you find `MaxRAMFraction` in a
Dockerfile, it is a fossil from the Java 8 container era; replace it, and check whether the
JVM is even honouring it any more.

---

## Why does it matter?

**1. It is the difference between "the container is slow" and "the container built a
different JVM".**

A CPU quota does not merely make each instruction take longer. It changes how many GC
threads exist, which collector runs, and whether parallel work is parallel. Those are
structural differences, and "just give it more CPU" only accidentally addresses them.

**2. The `.parallel()` that does nothing is invisible.**

There is no warning, no log line, no exception. Code that says `parallel` runs serially.
Code that says `supplyAsync` runs somewhere you did not expect. This is the most
surprising fact in Phase 8 for people arriving from a language with no shared-memory
parallelism, and it is silent.

**3. The heap default is wrong in both directions, expensively.**

Too small: constant GC, terrible p99, and an `OutOfMemoryError: Java heap space` in a
container with 70% of its memory unused. Too large (`-Xmx` = the limit): OOMKilled with a
healthy heap graph, no dump, exit 137. Both are common. Both are configuration.

**4. It is where Phase 8 becomes a decision instead of a diagnosis.**

Topics 68–80 taught you to measure allocation rate, live set, GC behaviour, and native
footprint. This is the topic where those measurements become a heap size, a collector, a
CPU request and a pod spec. It is the bridge from "I understand the JVM" to "I sized the
fleet", which is Topic 129.

**5. It is asked in every senior interview, in one of three forms.**

"Heap is at 40%, why did Kubernetes OOMKill the pod?" "How would you size the JVM for a
2 GB / 1 CPU container?" "Why is our service slower in Kubernetes than on the build
agent?" All three are this document.

---

## Machine-level reality

Everything here is checkable. Where I am not certain, I say so in one line and give the
command.

### The chain, in full: quota → processor count → everything

This diagram is the payload. Commit the **shape** to memory, not the arithmetic.

```
     Kubernetes pod spec:  limits.cpu: "500m"
                 |
                 v
     container runtime writes cgroup:
        v2:  /sys/fs/cgroup/cpu.max            ->  "50000 100000"
        v1:  cpu.cfs_quota_us = 50000, cpu.cfs_period_us = 100000
                 |
                 v
     JVM (UseContainerSupport=on) reads it at startup
        active_processors = ceil(quota / period) = ceil(0.5) = 1
        ... also bounded by sched_getaffinity() and cpuset
                 |
                 v
     Runtime.availableProcessors()  ==  1
                 |
     +-----------+-----------+-----------+-----------+-----------+-----------+
     |           |           |           |           |           |           |
     v           v           v           v           v           v           v
 Parallel    Concurrent  "server-    ForkJoinPool  virtual-   JIT compiler  library
 GC threads  GC threads   class?"    .commonPool() thread      threads      defaults
 (Parallel-  (ConcGC-     decision   parallelism   scheduler  (CICompiler-  (Tomcat,
  GCThreads)  Threads)      |        = n - 1 = 0   parallelism  Count)       Netty,
     |           |          v            |         = n = 1       |          Reactor,
     |           |    COLLECTOR          |            |          |          Jackson
     |           |    SELECTION          |            |          |          pools...)
     |           |          |            |            |          |             |
     v           v          v            v            v          v             v
  GC pause   concurrent  SerialGC   EVERY parallel  one carrier  slower    pools sized
  duration     cycle     instead    stream runs     thread for   warm-up   for a machine
  and CPU      keeps up   of G1     SERIALLY, on    ALL virtual  (Topic     that does not
  cost         or not                the CALLER     threads       74)       exist
                                      thread
```

Every arrow in that diagram is a real derivation the JVM performs before your `main` runs.
Take them one at a time.

### Link 1 — GC thread counts

`-XX:ParallelGCThreads` is derived from the active processor count with a well-known
heuristic: for counts up to 8, it equals the processor count; above 8, it grows more slowly
(roughly `8 + (n - 8) * 5/8`). `-XX:ConcGCThreads` (G1's concurrent-marking threads) is
derived from `ParallelGCThreads`, roughly a quarter of it, rounded up.

> **Honest uncertainty, one line:** the `8 + (n-8)*5/8` shape is the long-standing HotSpot
> heuristic and I am reasonably confident it still holds, but the exact formula and the
> `ConcGCThreads` ratio vary by collector and release. **Settle it:**
> `java -XX:+PrintFlagsFinal -version | grep -iE "ParallelGCThreads|ConcGCThreads"`, and on
> a container, run the same command **inside** the container so the derivation happens under
> the real quota.

**Why you care in both directions:**

- **Too few** (quota of 1 on a large-heap service): young collections are single-threaded,
  so pause duration scales with live set with no parallelism to hide it. G1's concurrent
  cycle may not keep up with the allocation rate, and you get evacuation failures and full
  GCs — Topic 71's failure shapes, caused by a CPU limit.
- **Too many** (no quota, on a 64-core node, with a small CPU share): the JVM sees 64
  processors and creates GC threads to match. Those threads all become runnable during a
  stop-the-world pause, in a cgroup whose CPU weight entitles it to a fraction of a core.
  The kernel throttles them. **A pause that should take milliseconds takes hundreds**, and
  it looks exactly like a GC problem, because it is one — caused by a missing limit. This
  is Trap 3, and it is the single most under-diagnosed Java-in-Kubernetes issue.

### Link 2 — collector selection, and the SerialGC surprise

The JVM's ergonomics classify the machine. Historically, the JVM selects G1 only on a
**"server-class machine"**, defined as at least **2 available processors** and at least
about **1792 MB** of memory. Below either threshold, it selects **SerialGC**.

Read that against the drill's configuration: `--cpus=0.5 --memory=512m` gives 1 processor
and 512 MB. **Both thresholds fail.** The JVM selects SerialGC — a stop-the-world,
single-threaded collector — and nobody told you.

For a small CLI utility, that is the correct choice. For a Spring Boot service under load,
it means every young collection is a stop-the-world pause with no parallelism, and it will
show up in your p99 as a shape you will not recognise if you assumed G1.

> **Honest uncertainty, one line:** 2 processors and ~1792 MB are the long-standing
> server-class thresholds; I am confident about the shape of the rule and reasonably
> confident about those numbers, but this is exactly the kind of ergonomic detail that
> moves between releases. **Settle it:**
> ```bash
> java -XX:+PrintFlagsFinal -version | grep -E "UseSerialGC|UseG1GC|UseParallelGC|UseZGC"
> ```
> run **inside** the target container, and read which one is `true` and whether its origin
> is `ergonomic`. Also `-Xlog:gc+init=debug` at startup, which prints the selected collector
> and its thread counts explicitly.

**The lesson generalises beyond this one threshold:** *the JVM makes ergonomic decisions
based on what it thinks the machine is, and a container changes what it thinks the machine
is.* Always print the flags from inside the container. Never from your laptop.

### Link 3 — the common ForkJoinPool, and the silent serialisation

This is the doc's headline cascade, and the mechanism is exact.

`ForkJoinPool.commonPool()` has a parallelism of `availableProcessors() - 1`, with a floor
of **zero**. With `availableProcessors() == 1`, the common pool has **parallelism 0**, which
means it has **no worker threads**. A `ForkJoinPool` with parallelism 0 executes submitted
tasks **in the calling thread**.

Consequences, in order of how much they will surprise you:

**a) Every parallel stream in the JVM becomes serial.**

```java
// In orderflow's catalogue pricing path:
List<PricedProduct> priced = products.parallelStream()
        .map(pricingEngine::price)
        .toList();
```

Under a CPU quota that yields 1 processor, this runs entirely on the calling request thread.
It produces the correct answer. It is not faster. It is **slower** than the sequential
version, because you pay the stream's splitting and combining machinery for zero
parallelism. And there is no diagnostic anywhere that says so.

This is Topic 25's material — the common pool is JVM-global shared state — with a new and
worse cause: **the pool is empty because of a pod spec**.

**b) `CompletableFuture.supplyAsync(...)` without an executor changes behaviour.**

The JDK anticipates the parallelism-0 case. `CompletableFuture`'s default async executor is
the common pool **only when the common pool's parallelism is greater than 1**; otherwise it
falls back to an executor that creates **a new thread per task**.

So under a CPU quota of 1, the code that was submitting work to a bounded, shared,
work-stealing pool is now **creating an unbounded number of raw threads** — one per
`supplyAsync` call. Each with a stack (native memory, Topic 80). Under the Topic 65 load
that is a thread explosion, and its symptom is `OutOfMemoryError: unable to create native
thread` (Topic 98's shape) or an OOMKill from thread-stack memory.

**The same pod spec produces two opposite failures in two different APIs.** Parallel streams
silently under-use the machine; `CompletableFuture` silently creates unbounded threads.
Neither is documented anywhere your team will read.

**c) The override, and why it is a real option.**

```
-Djava.util.concurrent.ForkJoinPool.common.parallelism=<n>
```

This sets the common pool's parallelism explicitly, independent of the processor count. It
is legitimate when you have measured that the workload benefits — but understand what you
are doing: you are running N runnable threads inside a cgroup entitled to a fraction of a
core, so they will be throttled. **More threads than CPU entitlement does not create CPU.**
It can help for a pool that is mostly blocked and hurt for one that is mostly running.

**d) The real fix is usually neither.** Pass an explicit executor sized for the workload,
which is Topic 91's discipline, and stop depending on a JVM-global pool whose size is
decided by a pod spec written by someone in a different team.

### Link 4 — the virtual-thread scheduler

`[JAVA 21]` Virtual threads run on a dedicated `ForkJoinPool` of **carrier threads**, whose
parallelism defaults to `availableProcessors()`.

```
-Djdk.virtualThreadScheduler.parallelism=<n>
-Djdk.virtualThreadScheduler.maxPoolSize=<n>
```

With `availableProcessors() == 1`, you have **one carrier thread** for every virtual thread
in the process. Virtual threads are still enormously useful in that configuration — the
point of them is to raise the ceiling on *blocked* operations, and a blocked virtual thread
unmounts from its carrier and costs nothing. But:

- any **CPU-bound** virtual-thread work is now fully serialised behind one carrier;
- any **pinning** (a `synchronized` block held across a blocking call, or a native frame)
  now blocks the **only** carrier there is, and the entire process stops making progress.

Topic 101's pinning drill has a much sharper failure with a CPU quota of 1 than on your
8-core laptop. **The drill you will run in Phase 9 has a different severity depending on a
number in a YAML file**, and this document is why.

### Link 5 — JIT compiler threads

The number of compiler threads (`-XX:CICompilerCount`) is derived from the processor count
too. Fewer compiler threads means the queue of methods awaiting C1 and C2 compilation
drains more slowly, which means **a longer warm-up** — more requests served by interpreted
or C1 code before C2 catches up (Topic 74).

There is also a floor: tiered compilation needs at least one C1 and one C2 thread, so
`CICompilerCount` has a minimum of 2 when tiered compilation is enabled. Setting
`ActiveProcessorCount=1` and then trying to force `CICompilerCount=1` will be rejected.

**The operational consequence:** a service that scales out to many small pods pays the
warm-up cost on **every pod**, with fewer compiler threads on each. The first N requests
after each scale-up event are slow. If your autoscaler is aggressive, a meaningful fraction
of your traffic is permanently running in warm-up. That is a Topic 74 fact with a Topic 129
cost, and it is one of the strongest honest arguments for *fewer, larger* pods rather than
*more, smaller* ones for a JIT-compiled runtime.

### Link 6 — library defaults

Almost every library that sizes a pool calls `Runtime.availableProcessors()`. Netty's
default `EventLoopGroup` size (Topic 103), Tomcat's and Reactor's defaults, and many
others. None of them will tell you what they chose. `jcmd <pid> Thread.print` and counting
by name prefix is how you find out what actually got built.

### The memory half: heap is not the footprint

You built this equation in Topic 80. Here it is as a **budget**, which is how you use it.

```
container memory limit
  - metaspace          (native; UNBOUNDED by default; a Spring app loads thousands of classes)
  - code cache         (native; grows with compiled methods; grows more with an agent, Topic 81)
  - thread stacks      (native; -Xss x live thread count; bound the POOLS, not the stack)
  - GC structures      (native; card table, remembered sets, mark bitmaps; scales with heap)
  - direct buffers     (native; MaxDirectMemorySize defaults from max heap, not the limit)
  - native libraries   (JDBC, compression, crypto; INVISIBLE to NMT)
  - allocator slack    (glibc arenas; RSS - NMT total is the clue)
  - headroom           (page cache, sidecars, the kernel's opinion)
  ================================================================
  = the number -Xmx may be
```

**Setting `-Xmx` to the container limit means every line above it is unfunded.** That is
Trap 1, and it is the most common JVM-in-a-container mistake in the industry.

### Sizing the heap honestly, from measurement

The inputs are two numbers you already know how to measure, from Topic 70:

- **Live set** — how many bytes are actually reachable after a full collection.
- **Allocation rate** — MB/s of new objects.

The procedure:

1. **Measure the live set.** Run the Topic 65 baseline, then either
   `jcmd <pid> GC.run` followed by `jcmd <pid> GC.heap_info` (the used figure right after a
   full collection is a decent proxy), or read the post-collection occupancy from
   `-Xlog:gc`. A heap dump analysed in MAT gives you a more careful number.
2. **Set the heap to a multiple of the live set.** A working starting point is 2×–4× for a
   throughput-oriented G1 configuration; low-pause collectors want **more** headroom, not
   less, because they collect concurrently with your allocation and must not lose the race.
   **This is a starting point for an experiment, not a rule.**
3. **Subtract the native terms** from the container limit and check the heap you chose
   fits. If it does not, either the container is too small or the native terms need
   bounding.
4. **Bound every native term explicitly** — `MaxMetaspaceSize`, `ReservedCodeCacheSize`,
   `MaxDirectMemorySize`, `-Xss` plus a bounded thread pool.
5. **Re-run the Topic 65 baseline** and compare p50/p95/p99 and throughput. **Repeat.**

**The 32 GB capacity fact (Topic 69).** Compressed ordinary object pointers store references
as 32-bit values below roughly 32 GB of heap. Above that boundary the JVM switches to full
64-bit references and **every reference in your heap gets bigger**. A 33 GB heap can hold
*fewer* objects than a 31 GB one. So: never size a heap into the ~32–48 GB band. Either stay
comfortably below the boundary or go far enough above it that the extra capacity outweighs
the loss. Confirm which side you are on with
`jcmd <pid> VM.flags | grep UseCompressedOops`.

### `-XX:+AlwaysPreTouch` and the "fail in CI, not at 3am" trade

```
-XX:+AlwaysPreTouch
```

Commits and touches every page of the heap at startup, so committed memory and resident
memory agree from the first second. It costs startup time — noticeably, on a large heap.

In a container with a hard limit, that trade is often worth taking, because it converts
"OOMKilled at 3am under peak load, when the heap finally grew into its committed size" into
"the pod fails to start in CI". **A deterministic early failure is worth real startup
seconds.** It also makes RSS graphs boring and readable, which has its own value.

### The flags that make failures loud instead of silent

```
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/dumps    # evidence (Topic 79)
-XX:+ExitOnOutOfMemoryError                                 # die, let the orchestrator restart
-XX:NativeMemoryTracking=summary                            # attribution (Topic 80)
-Xlog:gc*:file=/logs/gc.log:time,uptime,level,tags:filecount=5,filesize=20m
-Xlog:os+container=trace                                    # what the JVM concluded about the container
```

`-XX:+ExitOnOutOfMemoryError` deserves a note. Without it, a JVM that throws
`OutOfMemoryError` on one thread may limp on in a half-broken state — the readiness probe
still passes, traffic keeps arriving, and every request fails in a novel way. In
Kubernetes, dying cleanly and being restarted is almost always better than limping. **Pair
it with the heap dump flag** so you die *and* leave evidence.

### `[JAVA 25]` A note on AOT, and what I will not guess

Project Leyden's ahead-of-time work — class loading and linking done in advance, plus
ahead-of-time method profiling — is landing across JDK 24 and 25, and it changes the
startup picture in a way that competes directly with native-image (Topic 83). It is
adjacent to this document because startup and warm-up are CPU-quota-sensitive.

> **Honest uncertainty, stated plainly because the brief demands it:** I know the *shape* —
> a training run records an AOT configuration, a second step creates a cache, and
> subsequent runs load it. **I am not confident enough in the exact flag spellings or in
> which capability landed in which release to have you type them from this document.**
> Do not take flag names from me here. Go to the JEP index at **openjdk.org/jeps/0**, find
> the ahead-of-time JEPs for your JDK, and cross-check with
> `java -XX:+PrintFlagsFinal -version | grep -i aot` on the build you actually run.
> Topic 122 (AppCDS and layered jars) and Topic 83 both revisit this with the same caveat.

What I will assert: **for a long-running service, a startup optimisation that keeps the JIT
is a better trade than one that removes it.** That is the argument in Topic 83, and it
applies to AppCDS today with no uncertainty at all.

---

## Example 1 — minimal

Forty lines that print everything the JVM decided about your container, plus the cascade.
Run it in Docker with different flags and the whole document becomes observable.

**`ContainerFacts.java`:**

```java
import java.lang.management.ManagementFactory;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ForkJoinPool;
import java.util.stream.IntStream;

public class ContainerFacts {

    public static void main(String[] args) throws Exception {
        Runtime rt = Runtime.getRuntime();
        long mb = 1024 * 1024;

        System.out.println("=== what the JVM thinks the machine is ===");
        System.out.println("availableProcessors()       = " + rt.availableProcessors());
        System.out.println("Runtime.maxMemory()   (MB)  = " + rt.maxMemory() / mb);
        System.out.println("Runtime.totalMemory() (MB)  = " + rt.totalMemory() / mb);

        System.out.println();
        System.out.println("=== the cascade ===");
        System.out.println("commonPool parallelism      = " + ForkJoinPool.getCommonPoolParallelism());
        System.out.println("commonPool pool size        = " + ForkJoinPool.commonPool().getPoolSize());

        System.out.println("garbage collectors in use:");
        ManagementFactory.getGarbageCollectorMXBeans()
                .forEach(gc -> System.out.println("    " + gc.getName()));

        System.out.println("memory pools:");
        ManagementFactory.getMemoryPoolMXBeans()
                .forEach(p -> System.out.println("    " + p.getName()
                        + "  max(MB)=" + (p.getUsage().getMax() < 0 ? "undefined"
                                          : p.getUsage().getMax() / mb)));

        System.out.println();
        System.out.println("=== is parallelism real? ===");

        // Which threads actually execute a parallel stream?
        String streamThreads = IntStream.range(0, 64)
                .parallel()
                .mapToObj(i -> Thread.currentThread().getName())
                .distinct()
                .sorted()
                .reduce((a, b) -> a + ", " + b)
                .orElse("none");
        System.out.println("parallel stream ran on       : " + streamThreads);

        // Where does an un-executored CompletableFuture actually run?
        String cfThread = CompletableFuture
                .supplyAsync(() -> Thread.currentThread().getName())
                .get();
        System.out.println("supplyAsync ran on           : " + cfThread);

        System.out.println();
        System.out.println("=== virtual threads ===");
        String vtCarrier = Thread.ofVirtual().start(() -> { }).toString();
        System.out.println("a virtual thread             : " + vtCarrier);
        System.out.println("(carrier parallelism defaults to availableProcessors())");
    }
}
```

**Run it four ways and read the difference. This is the whole document in four commands.**

```bash
javac ContainerFacts.java

# 1. no limits at all - what your host looks like
java ContainerFacts

# 2. the drill's configuration
docker run --rm --cpus=0.5 --memory=512m -v "$PWD":/app -w /app eclipse-temurin:21 \
  java ContainerFacts

# 3. one full CPU, more memory
docker run --rm --cpus=1 --memory=2g -v "$PWD":/app -w /app eclipse-temurin:21 \
  java ContainerFacts

# 4. two CPUs - crosses the server-class threshold
docker run --rm --cpus=2 --memory=2g -v "$PWD":/app -w /app eclipse-temurin:21 \
  java ContainerFacts
```

| What you see | What it means |
|---|---|
| `availableProcessors() = 1` under `--cpus=0.5` | `ceil(0.5) = 1`. **Half a CPU and one CPU are indistinguishable to the JVM** — while the scheduler treats them very differently. |
| `commonPool parallelism = 0` in run 2 | `availableProcessors() - 1`, floored at zero. **The common pool has no worker threads.** |
| `parallel stream ran on : main` in run 2 | **The headline finding.** A `parallel()` stream executed entirely on the calling thread. It is not parallel. Nothing warned you. |
| `parallel stream ran on : main, ForkJoinPool.commonPool-worker-1, ...` in run 4 | Real parallelism. Note that `main` appears in both — the calling thread always participates. |
| `supplyAsync ran on : ForkJoinPool.commonPool-worker-N` in run 4 | The normal case. |
| `supplyAsync ran on` something that is **not** a common-pool worker in run 2 | The `CompletableFuture` fallback: with common-pool parallelism ≤ 1, the JDK uses a thread-per-task executor instead. **One thread per call, unbounded.** Look at the thread name carefully. |
| `Copy` / `MarkSweepCompact` in the collector list in run 2 | **SerialGC was selected.** The container failed the server-class test. Nobody told you. |
| `G1 Young Generation` / `G1 Concurrent GC` in run 4 | G1 was selected, because 2 processors and 2 GB pass the thresholds. |
| `Runtime.maxMemory()` ≈ 25% of the `--memory` value | The `MaxRAMPercentage` default, visible. Compare across runs 2, 3 and 4 and the percentage should hold. |
| `Runtime.maxMemory()` is close to the **host's** memory, not the container's | Container support is off or detection failed. Run `-Xlog:os+container=trace` immediately. |

**Now add the diagnostic that explains all of it:**

```bash
docker run --rm --cpus=0.5 --memory=512m eclipse-temurin:21 \
  java -Xlog:os+container=trace -XshowSettings:system -version
```

Read the trace lines about the memory limit, the CPU quota, the CPU period and the
resulting active processor count. **That output is the JVM showing its working**, and it is
the first thing to reach for whenever a container behaves unlike your laptop.

**Then prove you can override every decision:**

```bash
docker run --rm --cpus=0.5 --memory=512m -v "$PWD":/app -w /app eclipse-temurin:21 \
  java -XX:ActiveProcessorCount=4 -XX:MaxRAMPercentage=75 -XX:+UseG1GC ContainerFacts
```

| What you see | What it means |
|---|---|
| `availableProcessors() = 4`, `commonPool parallelism = 3` | `ActiveProcessorCount` wins over the derivation. The cascade follows it. |
| The parallel stream now uses several worker threads | Parallelism restored — **but you have 4 runnable threads in a cgroup entitled to half a core.** They will be throttled. More threads is not more CPU. |
| `Runtime.maxMemory()` ≈ 75% of 512 MB | `MaxRAMPercentage` honoured. Whether 75% is *correct* here is a separate, measured question. |
| G1 in the collector list despite the small container | Explicit flags beat ergonomics. Whether G1 is *better* than SerialGC on half a core is, again, a measured question — and often the answer is no. |

**That last row is the point of the whole example.** You now know how to override every
decision the JVM made. Overriding them correctly requires the Topic 65 baseline, and that
is the rest of this document.

---

## Example 2 — production scenario (on the project spine)

### The constraints

`orderflow` at the recorded Topic 65 baseline:

- **Recorded baseline configuration:** container `--memory=2g --cpus=2`, `-Xmx1g`, G1,
  `MaxDirectMemorySize` and `MaxMetaspaceSize` bounded from Topic 80's work, NMT summary on.
- **Dataset:** 100k products, 1M orders, 5M order lines, with a few hot products.
- **Load:** the recorded k6 mix — 70% catalogue read, 20% order read, 10% order placement,
  open-model arrival rate.
- **Recorded numbers:** p50/p95/p99/p999 and throughput per endpoint, in
  `/docs/java/baselines/`, re-runnable within ±10%.

### The change that ships

A platform-wide cost-reduction initiative. The message in the channel is reasonable:
"Several services are over-provisioned. We are adding CPU limits and reducing memory
requests across the estate. No application changes required."

```yaml
# before - what produced the recorded baseline
resources:
  requests: { cpu: "2",   memory: "2Gi" }
  limits:   { cpu: "2",   memory: "2Gi" }

# after - the cost-reduction change
resources:
  requests: { cpu: "250m", memory: "512Mi" }
  limits:   { cpu: "500m", memory: "512Mi" }
```

No Dockerfile change. No JVM flags — the image never had any; the recorded baseline's
`-Xmx1g` came from a `JAVA_TOOL_OPTIONS` value in a values file that this change also
removed, on the grounds that "the JVM sizes itself now".

That last sentence is true and is the problem.

### What the JVM builds instead

Everything below follows mechanically from `limits.cpu: 500m` and `limits.memory: 512Mi`.

1. `cpu.max` reads `50000 100000`. `ceil(0.5) = 1`. **`availableProcessors()` returns 1.**
2. 1 processor and 512 MB fail the server-class test. **SerialGC is selected.**
3. `MaxRAMPercentage` at its default gives a max heap around a quarter of 512 MB. For a
   Spring Boot service with Hibernate, an entity graph over 5 M order lines, a Hikari pool
   and a Tomcat thread pool, **that is a very small heap.**
4. `ParallelGCThreads` and `ConcGCThreads` are sized for 1 processor — and with SerialGC
   selected, the question is moot: collection is single-threaded by definition.
5. **`ForkJoinPool.commonPool()` parallelism is 0.** Every parallel stream in the process
   runs on its calling thread.
6. `CompletableFuture.supplyAsync(...)` without an explicit executor falls back to a
   thread-per-task executor. Every such call creates a thread.
7. The virtual-thread scheduler, if `spring.threads.virtual.enabled` is on, has **one
   carrier thread**.
8. `CICompilerCount` is at its floor. Warm-up is slower, so more requests are served by
   interpreted or C1 code after every pod start (Topic 74).
9. Metaspace is still unbounded by default, and the code cache is still unbounded by
   default. In a 512 MB container, with a heap of roughly 128 MB, **those two native terms
   are now a large fraction of the budget** (Topic 80).

### What you observe, in the order you actually meet it

1. **Pods start.** Readiness passes, eventually — slower than before, because warm-up is
   slower and startup is CPU-bound.
2. **The catalogue endpoint gets much worse.** `GET /products` uses a parallel stream in
   the pricing path. p50 and p99 both rise. Nothing about the code changed.
3. **Order placement p99 collapses.** GC pauses are stop-the-world and single-threaded,
   against a live set that did not shrink just because the heap did. The heap is small, so
   collections are frequent. **Frequent single-threaded pauses is the worst possible
   combination**, and it is exactly what the ergonomics chose.
4. **The thread count climbs.** The `CompletableFuture` fan-in used for the payment
   callback creates one thread per call now. `jcmd 1 Thread.print | grep -c '^"'` grows
   with request rate.
5. **Pods OOMKill.** Exit 137. The heap dashboard shows the heap near its (small) maximum,
   so someone concludes "we need more heap" — but the kill is thread stacks plus metaspace
   plus code cache plus a 128 MB heap in a 512 MB box.
6. **Some pods instead throw `OutOfMemoryError: Java heap space`**, because the live set
   genuinely does not fit in the derived heap. **Two different memory failures in the same
   deployment**, which is maximally confusing and sends two people down two paths.
7. **The load-test gate fails catastrophically** — not by 10%, by a lot. Someone says "the
   load test is broken".
8. **Nobody connects any of it to the pod spec**, because the change was described as
   "reducing over-provisioning" and reviewed by people thinking about cost, not about
   `availableProcessors()`.

### The correct diagnosis, in the order you should do it

**Step 1 — ask the JVM what it thinks it is running on.** Before any theory:

```bash
kubectl exec -it <pod> -- java -Xlog:os+container=trace -XshowSettings:system -version
kubectl exec -it <pod> -- jcmd 1 VM.flags -all > flags-after.txt
```

Diff `flags-after.txt` against the same capture from the baseline configuration. **The
diff is the incident.** You will see `MaxHeapSize` changed with origin `ergonomic`,
`UseSerialGC` flipped to `true` with origin `ergonomic`, `ParallelGCThreads` changed, and
`CICompilerCount` changed — **none of which anyone wrote anywhere.**

**Step 2 — confirm the cascade is live, not theoretical.** Add a startup log line to
`orderflow` permanently. Three lines of code, and it would have made this incident a
five-minute investigation:

```java
package com.orderflow.config;

import org.springframework.boot.context.event.ApplicationReadyEvent;
import org.springframework.context.event.EventListener;
import org.springframework.stereotype.Component;

import java.lang.management.ManagementFactory;
import java.util.concurrent.ForkJoinPool;
import java.util.stream.Collectors;

@Component
public class RuntimeFactsLogger {

    private static final org.slf4j.Logger log =
            org.slf4j.LoggerFactory.getLogger(RuntimeFactsLogger.class);

    @EventListener(ApplicationReadyEvent.class)
    public void logRuntimeFacts() {
        Runtime rt = Runtime.getRuntime();
        String collectors = ManagementFactory.getGarbageCollectorMXBeans().stream()
                .map(java.lang.management.GarbageCollectorMXBean::getName)
                .collect(Collectors.joining(","));

        log.info("runtime_facts availableProcessors={} maxHeapMB={} collectors=[{}] "
                 + "commonPoolParallelism={} vtSchedulerParallelism={}",
                rt.availableProcessors(),
                rt.maxMemory() / (1024 * 1024),
                collectors,
                ForkJoinPool.getCommonPoolParallelism(),
                System.getProperty("jdk.virtualThreadScheduler.parallelism", "default"));
    }
}
```

**Emit these as metrics too, not just as a log line.** `availableProcessors` and
`maxHeapMB` as gauges on a dashboard, next to the pod's CPU limit, means the next time
someone changes a pod spec you see the runtime change on a graph. This is cheap and almost
nobody does it.

**Step 3 — separate the four failures.** There are four distinct problems in this incident
and they have four distinct fixes. Do not fix them as one thing.

| Symptom | Mechanism | Fix |
|---|---|---|
| Catalogue endpoint slower | common-pool parallelism 0; parallel streams serialised | explicit executor, or more CPU, or stop using parallel streams here |
| Order p99 collapsed | SerialGC + small heap + unchanged live set | explicit collector, explicit heap from measured live set |
| Thread count climbing | `CompletableFuture` fallback to thread-per-task | explicit executor everywhere (Topic 91) |
| OOMKill | heap + metaspace + code cache + thread stacks > 512 MB | bound every native term; size the container from the budget |

**Step 4 — decide what the container should actually be.** This is the engineering, and it
is a measurement, not a preference.

### The fix, in four layers

**Layer 1 — make the runtime explicit, so ergonomics can never surprise you again.**

```
-XX:MaxRAMPercentage=<measured>
-XX:+UseG1GC
-XX:MaxMetaspaceSize=<measured>m
-XX:ReservedCodeCacheSize=<measured>m
-XX:MaxDirectMemorySize=<measured>m
-Xss<measured>k
-XX:+ExitOnOutOfMemoryError
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/dumps
-XX:NativeMemoryTracking=summary
-Xlog:gc*:file=/logs/gc.log:time,uptime,level,tags:filecount=5,filesize=20m
```

Every `<measured>` is a placeholder for a number **you** produce from the Topic 65 run and
Topic 80's NMT work. I am not giving you numbers; I have no JVM and any figure I invented
would be worse than useless, because it would look authoritative.

**Note what is not in that list: `-Xmx`.** Prefer `MaxRAMPercentage` in a container, because
it survives a change to the memory limit. `-Xmx1g` in a 2 GB container becomes `-Xmx1g` in a
512 MB container the moment someone edits a values file — which is a different, worse version
of this same incident. A percentage tracks the limit automatically.

**Layer 2 — decide the CPU allocation deliberately, and state the reasoning.**

Three defensible positions, and the choice depends on measurement:

- **`limits.cpu: 2` (restore the baseline).** The service was sized for 2 CPUs and its
  recorded baseline is at 2 CPUs. If it meets its latency SLO at 2 CPUs and the cost is
  acceptable, this is the honest answer and "we measured it" is the justification.
- **`limits.cpu: 1`, with `-XX:ActiveProcessorCount=2`.** Legitimate when you have measured
  that the workload is mostly **blocked** (waiting on Postgres, which for `orderflow` is
  most of a request) rather than mostly running. You get a 2-processor cascade — G1 selected,
  a common pool with workers, two carriers — inside a 1-CPU entitlement. **This is a real
  technique, and it must be measured, not assumed**, because you are now running more
  runnable threads than your CPU entitlement.
- **No CPU limit at all, only a request.** Increasingly recommended for latency-sensitive
  JVM services, because CFS quota throttling produces latency spikes that look exactly like
  GC pauses. **But:** with no quota, the JVM sees the node's CPU count and sizes GC threads
  and pools for the whole node — which is Trap 3. So this option is only safe **with an
  explicit `-XX:ActiveProcessorCount`**. The two go together; picking one without the other
  is worse than either baseline.

**Layer 3 — size the memory from the budget, not from a percentage you liked.**

1. Measure the live set at the Topic 65 baseline (Topic 70).
2. Choose a heap as a multiple of it, and be prepared to defend the multiple.
3. Measure the native terms with NMT (Topic 80) and bound each one.
4. Add them up. Add headroom. **That sum is the container limit**, not the other way round.
5. Express the heap as `MaxRAMPercentage` of that limit so the two stay coupled.

**Layer 4 — re-run the baseline, and record the whole configuration with it.**

A baseline that does not name its container limits, its JVM flags, its collector and its
`availableProcessors()` is not reproducible. Add those to `/docs/java/baselines/` as fields,
not as prose. The Topic 65 gate rule — re-run within ±10% — is only meaningful if the
configuration is part of the record.

### The budget line this produces

| Term | Bounded by | Number from |
|---|---|---|
| Java heap | `MaxRAMPercentage` | measured live set × multiple (Topic 70) |
| Metaspace | `MaxMetaspaceSize` | NMT `Class` committed at steady state + headroom (Topic 80) |
| Code cache | `ReservedCodeCacheSize` | NMT `Code` committed at steady state + headroom |
| Thread stacks | `-Xss` × bounded pools | `Thread.print` count × `-Xss`; bound the **pools** |
| Direct buffers | `MaxDirectMemorySize` | `BufferPoolMXBean` at peak + headroom |
| GC structures | implied by heap and collector | NMT `GC` committed |
| Native libraries | audit | `RSS − NMT total` (Topic 80) |
| Headroom | judgement | state it explicitly, do not leave it implicit |
| **Container limit** | **the sum** | — |
| CPU limit | measured | latency at the Topic 65 baseline vs cost (Topic 129) |
| `availableProcessors()` | `ActiveProcessorCount` | **a decision, not an accident** |

**That table is the deliverable.** It is also, almost verbatim, the input to Topic 129's
capacity and cost model.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — setting `-Xmx` equal to the container memory limit

**Wrong:** `limits.memory: 1Gi` with `-Xmx1g`, on the reasoning that the container exists to
run the JVM, so the JVM should get all of it.

**Exact symptom, precisely:**

- Pods restart intermittently, correlated with load but not with any particular endpoint.
- `kubectl describe pod` shows `Last State: Terminated`, `Reason: OOMKilled`,
  `Exit Code: 137`.
- **The last application log line is an ordinary request completing.** No exception, no
  stack trace, no shutdown-hook output.
- The heap dashboard reads well below 100% at the moment of death and does not spike first.
- No heap dump on disk despite `-XX:+HeapDumpOnOutOfMemoryError` being set.
- `cat /sys/fs/cgroup/memory.events` in a surviving pod shows a non-zero `oom_kill` counter.
- The failure gets **more** frequent after a release that loads more classes, or after
  attaching an APM agent (Topic 81), with no change to the heap graph at all.

**Root cause:** RSS is heap **plus** metaspace, code cache, thread stacks, GC structures,
direct buffers, JVM internals and native libraries. Setting `-Xmx` to the whole limit makes
every one of those terms over budget by construction. The kill is `SIGKILL` from the cgroup
OOM killer, which the JVM cannot catch — hence no `OutOfMemoryError`, no dump, no hook.

**Fix — in this order:**

1. **Stop using `-Xmx` in a container. Use `-XX:MaxRAMPercentage`.** It tracks the limit, so
   a values-file edit cannot silently invert your safety margin.
2. **Enable NMT** and measure every native term at steady state under the Topic 65 load.
3. **Bound each one explicitly**: `MaxMetaspaceSize`, `ReservedCodeCacheSize`,
   `MaxDirectMemorySize`, `-Xss` plus bounded pools.
4. **Build the budget table** from the previous section and set the container limit to the
   sum, plus stated headroom.
5. **Consider `-XX:+AlwaysPreTouch`** so the failure, if it comes, arrives at startup in CI
   rather than at peak load at 3am.
6. **Re-run the Topic 65 baseline** and confirm the smaller heap did not cost p99.

**The rule to carry:** *heap is a term in the equation, not the equation.* (Topic 80 said
this. It is the most important sentence in Phase 8 and it earns the repetition.)

---

### Trap 2 — a CPU quota that silently serialises every parallel operation

**Wrong:** treating `limits.cpu` as a throttle that makes things proportionally slower.

**Exact symptom, precisely:**

- An endpoint that uses a parallel stream is **slower than its own sequential version would
  be**, not merely un-accelerated. The stream machinery costs something and buys nothing.
- `jcmd <pid> Thread.print | grep -c "ForkJoinPool.commonPool-worker"` returns **zero**.
  The pool exists; it has no workers.
- A thread dump taken during the slow endpoint shows the work happening on
  `http-nio-8080-exec-*` threads — **the request threads themselves** — not on pool workers.
- CPU utilisation is at or near the quota ceiling with a modest request rate, and
  `cat /sys/fs/cgroup/cpu.stat` shows a rising `nr_throttled` and `throttled_usec`.
- Separately and confusingly, the **total thread count climbs with request rate**, because
  `CompletableFuture.supplyAsync` without an executor has fallen back to a thread-per-task
  executor.
- The same image on a developer laptop, or on the CI runner, behaves perfectly. **The bug
  only exists where the quota exists.**

**Root cause:** `availableProcessors()` returns `ceil(quota/period)`, which is 1 for any
quota at or below one CPU. `ForkJoinPool.commonPool()`'s parallelism is
`availableProcessors() - 1` floored at zero, so it has no worker threads and executes
submitted tasks in the caller. Meanwhile `CompletableFuture` detects common-pool parallelism
≤ 1 and substitutes a thread-per-task executor, producing the opposite failure in the same
process.

**Fix — in this order:**

1. **Prove it first**, with `ForkJoinPool.getCommonPoolParallelism()` and a thread dump. Do
   not tune on a hypothesis.
2. **Pass explicit executors.** Topic 91's rule — never submit work to a pool you did not
   size — makes the whole class of problem go away. This is the real fix.
3. **Question the parallel stream at all.** Topic 25's preconditions: splittable source,
   enough elements times enough per-element cost, no shared mutable state. For a request
   handler on a 1-CPU container, a parallel stream is almost never right even when the pool
   has workers.
4. **If you genuinely need parallelism on a fractional CPU**, set
   `-XX:ActiveProcessorCount` deliberately and measure — but understand you are creating
   runnable threads that the CFS quota will throttle.
5. **Re-run the Topic 65 baseline** and check the catalogue endpoint specifically, since it
   is the one that uses the parallel path.

**The rule to carry:** *a CPU quota does not slow the runtime down; it builds a smaller
one.*

---

### Trap 3 — GC threads sized from a quota nobody thought about

**Wrong:** setting `requests.cpu` without `limits.cpu` (the most common Kubernetes idiom)
and assuming the JVM sizes itself from the request.

**Exact symptom, precisely:**

- Very long stop-the-world pauses — far longer than the heap size or live set justifies.
- `-Xlog:gc*` shows pauses whose **duration varies enormously run to run** for collections
  doing similar amounts of work.
- `-Xlog:safepoint` (Topic 73) shows a large gap between reaching the safepoint and the
  operation completing — the threads are runnable but not running.
- `cat /sys/fs/cgroup/cpu.stat` shows large `nr_throttled` and `throttled_usec`, and the
  throttling **spikes during GC pauses**.
- `java -XX:+PrintFlagsFinal -version | grep ParallelGCThreads`, run **inside the pod**,
  returns a number that matches the **node's** core count, not anything in your pod spec.
- Pauses get **worse when the cluster gets busier**, because the CPU weight entitles you to
  less when there is contention. The service's own load did not change.
- Moving the pod to a bigger node makes it worse, not better.

**Root cause:** `requests.cpu` becomes a cgroup **weight/share**, which is a scheduling
hint under contention. It is **not a quota**, and on modern JDKs it is not used to derive
the processor count. With no quota, the JVM concludes it has the whole node — so on a
64-core node it creates GC threads sized for 64 processors. During a stop-the-world pause,
all of them become runnable inside a cgroup whose weight entitles it to a fraction of a
core. The kernel time-slices them. A pause that should be milliseconds becomes hundreds of
milliseconds of scheduling.

**This is the inverse of Trap 2 and it comes from the same missing number.** Trap 2 is too
few threads because of a quota. Trap 3 is too many threads because of no quota. Both are
"the JVM sized itself from something that was not the truth".

**Fix — in this order:**

1. **Always set `-XX:ActiveProcessorCount` explicitly** if you do not set a CPU limit. This
   is the single most valuable line in this trap. It decouples the JVM's internal sizing
   from whether an SRE happened to set a limit.
2. **Or set a CPU limit** — accepting that CFS throttling has its own latency cost, which
   is a real and much-argued trade-off.
3. **Verify by printing the flags inside the pod**, never on your laptop:
   `kubectl exec <pod> -- jcmd 1 VM.flags -all | grep -iE "GCThreads|ActiveProcessor"`.
4. **Cap the GC threads directly** as a belt-and-braces measure:
   `-XX:ParallelGCThreads=<n> -XX:ConcGCThreads=<n>`.
5. **Watch `cpu.stat` throttling as a first-class metric.** If `nr_throttled` correlates
   with your p99 spikes, you have found the cause of a whole class of "mystery GC pause".

**The rule to carry:** *requests are a scheduling hint; limits are what the JVM reads. If
you set neither, the JVM believes it owns the node.*

---

### Trap 4 — trusting the default heap percentage in a single-purpose container

**Wrong:** shipping with no heap flags at all, on the reasoning that "the JVM is container
aware now".

**Exact symptom, precisely — and note there are two opposite ones:**

**Symptom A, the heap is too small:**

- `OutOfMemoryError: Java heap space` **inside a container with most of its memory unused**.
  `container_memory_working_set_bytes` sits well below the limit while the JVM dies.
- `-Xlog:gc` shows near-continuous collection: back-to-back young collections, then full
  collections, with the post-collection occupancy barely dropping. The collector is running
  constantly and reclaiming little, because the live set nearly fills the heap.
- CPU is high with low throughput. The service is spending its time collecting.
- p99 is dominated by GC pauses, and raising the **container** limit does nothing, because
  the heap is a percentage and 25% of a bigger number is still a percentage.
- `jcmd <pid> VM.flags -all | grep MaxHeapSize` shows origin `ergonomic` — **nobody chose
  this number.**

**Symptom B, the derived heap plus native terms exceed the limit:** the OOMKill from Trap 1,
arriving without anyone having set `-Xmx` at all.

**Root cause:** the default is a **multi-tenant** default, designed to be safe on a machine
that might be running several processes. Your container runs one JVM and exists for no other
purpose. A default that leaves three quarters of the container to other processes is
answering a question you did not ask.

**Fix — in this order:**

1. **Measure the live set** at the Topic 65 baseline (Topic 70). This is the input. Without
   it every heap number is a guess.
2. **Set `MaxRAMPercentage` from the budget table**, not from a blog post. For a container
   whose only job is one JVM, the number is usually much higher than the default — but "much
   higher" must be validated against the native terms with NMT, not assumed.
3. **Confirm the value took effect from inside the container**:
   `jcmd 1 VM.flags -all | grep -iE "MaxHeapSize|RAMPercentage"` and check the **origin**
   column now reads `command line`.
4. **Re-run the Topic 65 baseline** and compare p50/p95/p99, GC frequency and GC pause
   duration. A bigger heap usually means fewer, longer pauses; whether that is better
   depends on your latency budget, which is Topic 72's argument.
5. **Record the number and its justification** in `/docs/java/baselines/`. "We set 70%
   because the measured live set is X and the native terms sum to Y" is a sentence that
   survives a handover. "70% because that's what we always use" is not.

**The rule to carry:** *the JVM's automatic heap size is a safe default for a machine you
do not own. You own this machine.*

---

### Trap 5 — the collector you never chose

**Wrong:** assuming the collector you tuned in Topics 71 and 72 is the one running in
production.

**Exact symptom, precisely:**

- GC log lines do not have the shape you expect. You tuned G1 and configured
  `MaxGCPauseMillis`, but the log has no G1 phase names, no concurrent cycle, and no
  region-based accounting.
- `-XX:MaxGCPauseMillis` appears to have no effect whatsoever, because it is a G1/ZGC
  concept and you are not running either.
- `ManagementFactory.getGarbageCollectorMXBeans()` returns collector names you do not
  recognise (SerialGC's beans are named differently from G1's).
- Pause durations scale linearly with the live set and show **no concurrent phase at all**.
- The same image on a bigger container behaves completely differently — because it crosses
  the server-class threshold and gets a different collector.
- Your Topic 71 and 72 drills, re-run in this container, produce output that does not match
  what those documents describe. **The documents are not wrong; the collector is different.**

**Root cause:** collector selection is **ergonomic**. On a machine the JVM does not classify
as server-class — fewer than about 2 available processors, or less than about 1792 MB — it
selects SerialGC. A small container fails both tests. Nothing in your configuration said
"SerialGC"; nothing warned you; the log line naming the collector is only there if you asked
for it with `-Xlog:gc+init`.

**Fix — in this order:**

1. **Always select the collector explicitly in production.** `-XX:+UseG1GC`,
   `-XX:+UseZGC`, `-XX:+UseSerialGC` — whichever you measured. Ergonomics are a default for
   people who have not measured; you have.
2. **Log the selection at startup**: `-Xlog:gc+init=debug` prints the collector and its
   thread counts, and it costs nothing.
3. **Verify from inside the container**, always:
   `jcmd 1 VM.flags -all | grep -E "UseSerialGC|UseG1GC|UseParallelGC|UseZGC"` — and read
   the origin column.
4. **Ask whether SerialGC was actually the right answer.** On a genuinely 1-CPU container
   with a small heap, SerialGC may well beat G1 — G1's concurrent threads and write barriers
   cost CPU you do not have. **Ergonomics being surprising does not make ergonomics wrong.**
   Measure both against the Topic 65 baseline before overriding.
5. **Re-run your Topic 71/72 drills** under the production container configuration, not
   under your laptop's, so the numbers you learned from are the numbers that apply.

**The rule to carry:** *print the collector from inside the container, or you are tuning a
JVM you are not running.*

---

## Hands-on proof

Every command below is one **you** run. I have no JVM and no container runtime, so I print
no output and claim nothing as captured. What follows is the exact command, what to look
for, and how to read each result you might get.

### Setup

```bash
mkdir -p ~/java-lab/82 && cd ~/java-lab/82
java --version
docker --version
stat -fc %T /sys/fs/cgroup    # cgroup2fs => v2, tmpfs => v1 (on Linux; on macOS this is the VM)
```

### Proof 0 — what the JVM concluded, in its own words

```bash
docker run --rm --cpus=0.5 --memory=512m eclipse-temurin:21 \
  java -Xlog:os+container=trace -version 2>&1 | head -40
```

| What you see | What it means |
|---|---|
| Trace lines naming `/sys/fs/cgroup/...` paths and parsed values | Container detection is working. **Read the parsed CPU quota, period and memory limit.** |
| A line stating the active processor count | This is the number the whole cascade depends on. Compare it to your `--cpus` value and confirm the `ceil`. |
| `Container Support: disabled` or host-sized values | `UseContainerSupport` is off, or you are not in a container the JVM recognises. Everything downstream will be wrong. |
| Nothing at all | You are not on Linux — on macOS/Windows the JVM runs inside a Linux VM and the numbers may reflect the VM, not your `--cpus` flag. **Verify on a real Linux host before trusting any of this.** |

### Proof 1 — the `ceil`, demonstrated

```bash
for c in 0.25 0.5 1 1.5 2 2.5 4; do
  printf "cpus=%-5s -> " "$c"
  docker run --rm --cpus=$c eclipse-temurin:21 \
    jshell -q -s <<< 'System.out.println(Runtime.getRuntime().availableProcessors());' \
    2>/dev/null | tail -1
done
```

(If `jshell` is awkward in your image, use the compiled `ContainerFacts` from Example 1
instead — same result, fewer moving parts.)

| What you see | What it means |
|---|---|
| 0.25 → 1, 0.5 → 1, 1 → 1 | **`ceil` in action.** A quarter CPU and a full CPU are the same number to the JVM. |
| 1.5 → 2, 2.5 → 3 | Always rounded up. Never down, never fractional. |
| Every value → your host's core count | Container detection failed, or your Docker Desktop VM is not applying the quota. Go back to Proof 0. |

### Proof 2 — the flag defaults on **your** JDK, settled

```bash
java -XX:+PrintFlagsFinal -version | grep -i ramp
java -XX:+PrintFlagsFinal -version | grep -i ActiveProcessor
java -XX:+PrintFlagsFinal -version | grep -iE "MaxHeapSize|InitialHeapSize|MaxRAM "
java -XX:+PrintFlagsFinal -version | grep -iE "ParallelGCThreads|ConcGCThreads|CICompilerCount"
java -XX:+PrintFlagsFinal -version | grep -E "UseSerialGC|UseG1GC|UseParallelGC|UseZGC"
```

*Illustration of the format, not captured output.* A `PrintFlagsFinal` line looks like this
shape — I am showing you the **columns**, with `<n>` where a number would be:

```
     uintx MaxHeapSize                         = <n>          {product} {ergonomic}
   double MaxRAMPercentage                     = <n>          {product} {default}
   double InitialRAMPercentage                 = <n>          {product} {default}
     uint ActiveProcessorCount                 = <n>          {product} {default}
     bool UseG1GC                              = true         {product} {ergonomic}
```

**How to read those columns, which is the actual skill:**

| Column | Meaning | What to do with it |
|---|---|---|
| type (`uintx`, `bool`, `double`) | the flag's type | tells you the syntax: `-XX:Flag=<n>` vs `-XX:+Flag` |
| name | the flag | |
| `= <value>` | the **effective** value after all ergonomics | this is what is actually in force |
| `{product}` / `{diagnostic}` / `{experimental}` | flag category | `diagnostic` needs `-XX:+UnlockDiagnosticVMOptions`; `experimental` needs `-XX:+UnlockExperimentalVMOptions` |
| `{default}` | nobody set it | it is the JVM's built-in default |
| `{ergonomic}` | **the JVM chose this from the machine** | **the column that matters for this document.** An `ergonomic` value changes when the container changes. |
| `{command line}` | you set it | it will not change when the container changes |
| `{environment}` | set via `JAVA_TOOL_OPTIONS` or similar | find out who set it, and where |

**Then run the identical command inside the target container** and diff the two outputs.
Every line that differs is a decision the container made for you.

```bash
java -XX:+PrintFlagsFinal -version > flags-host.txt
docker run --rm --cpus=0.5 --memory=512m eclipse-temurin:21 \
  java -XX:+PrintFlagsFinal -version > flags-container.txt
diff flags-host.txt flags-container.txt
```

**That diff is the single most instructive artefact in this document.** Read every line of
it.

### Proof 3 — the common pool has no workers

```bash
docker run --rm --cpus=0.5 --memory=512m -v "$PWD":/app -w /app eclipse-temurin:21 \
  java ContainerFacts
```

| What you see | What it means |
|---|---|
| `commonPool parallelism = 0` | **Confirmed.** `availableProcessors() - 1`, floored at zero. |
| `parallel stream ran on : main` | **The headline.** `parallel()` executed on the calling thread. Nothing is parallel. |
| `supplyAsync ran on` a thread whose name is not a common-pool worker | The `CompletableFuture` thread-per-task fallback. **One thread per call.** |
| `commonPool parallelism = 1` | You are at `--cpus=2`. Cross-check with `availableProcessors()`. |

### Proof 4 — the collector you did not choose

```bash
for m in 512m 1g 2g; do
  for c in 0.5 1 2; do
    printf "mem=%-5s cpus=%-4s -> " "$m" "$c"
    docker run --rm --cpus=$c --memory=$m eclipse-temurin:21 \
      java -XX:+PrintFlagsFinal -version 2>/dev/null \
      | grep -E "UseSerialGC|UseG1GC" | grep true | awk '{print $2}' | tr '\n' ' '
    echo
  done
done
```

| What you see | What it means |
|---|---|
| `UseSerialGC` true at small memory and/or 1 processor | **The server-class threshold, mapped.** You have just measured where the boundary is on your JDK, which is better than trusting my numbers. |
| `UseG1GC` true from 2 processors and ~2 GB upward | Confirms the shape of the rule. |
| The boundary is somewhere I did not predict | **Your measurement wins.** Write down where it actually is for your JDK and base image. |

### Proof 5 — throttling, which is the thing quotas actually do

Inside a running container under load:

```bash
cat /sys/fs/cgroup/cpu.stat          # v2
# nr_periods, nr_throttled, throttled_usec

cat /sys/fs/cgroup/cpu/cpu.stat      # v1
```

| What you see | What it means |
|---|---|
| `nr_throttled` rising steadily under load | The CFS quota is actively stopping your threads. **Correlate this with your p99 spikes** — it is a very common hidden cause. |
| `throttled_usec` large relative to wall-clock time | A substantial fraction of your latency is scheduler enforcement, not your code. |
| `nr_throttled` spiking during GC pauses specifically | **Trap 3.** Too many GC threads for the entitlement. |
| `nr_throttled` at zero | Either no quota, or you are not near it. Rule throttling out and move on. |

### Proof 6 — the live process, from inside the pod

```bash
kubectl exec -it <pod> -- jcmd 1 VM.flags -all > flags-prod.txt
kubectl exec -it <pod> -- jcmd 1 VM.command_line
kubectl exec -it <pod> -- jcmd 1 GC.heap_info
kubectl exec -it <pod> -- jcmd 1 Thread.print | grep -c '^"'
kubectl exec -it <pod> -- jcmd 1 VM.native_memory summary
```

| What you see | What it means |
|---|---|
| `VM.command_line` shows flags you did not expect | Something is injecting `JAVA_TOOL_OPTIONS` — a ConfigMap, an operator, a base image `ENTRYPOINT`. **Find it.** |
| Flags with origin `ergonomic` that matter (heap, collector, GC threads) | These will change if the pod spec changes. Make them explicit. |
| Thread count much higher than your configured pools | The `CompletableFuture` fallback, or a thread leak (Topic 98). Group the dump by thread-name prefix. |
| `GC.heap_info` used ≈ max | Trap 4 Symptom A. The heap is too small for the live set. |
| NMT `Class` or `Code` committed large relative to the container limit | The native terms are eating your budget (Topic 80). |

---

## Failure drill

**Mandatory.** Do not read the interpretation tables until you have produced the numbers
yourself. The point is not the knowledge — it is the memory of watching `parallel()` run on
`main`, and of finding `UseSerialGC = true {ergonomic}` in a flag dump nobody wrote.

### The scenario, exactly as assigned

Run `orderflow` in a container with `--cpus=0.5 --memory=512m` and **no JVM flags at all**.
Capture `Runtime.availableProcessors()`, the selected collector, the default max heap, and
the OOMKill. Then fix it with explicit flags and re-run the Topic 65 baseline.

### Part A — the standalone version, to learn the instruments

Run `ContainerFacts` from Example 1 at `--cpus=0.5 --memory=512m`, and capture, before
anything else:

```bash
docker run --rm --cpus=0.5 --memory=512m -v "$PWD":/app -w /app eclipse-temurin:21 \
  java -Xlog:os+container=trace -Xlog:gc+init=debug ContainerFacts 2>&1 | tee facts-drill.txt
```

Write down, in a notes file, not in your head:

1. `availableProcessors()`.
2. `Runtime.maxMemory()` in MB, and it as a percentage of 512.
3. The collector names from the MXBeans.
4. `ForkJoinPool.getCommonPoolParallelism()`.
5. Which thread(s) the parallel stream ran on.
6. Which thread the `supplyAsync` ran on.
7. From `-Xlog:os+container=trace`: the parsed CPU quota, CPU period and memory limit.

### Part B — the real drill, on `orderflow`

**1. Confirm the baseline is still valid.** Re-run the recorded Topic 65 scenario against
the baseline configuration and confirm it lands within ±10%. **If it does not, stop.** Every
number you take today would be measured against a drifted reference. This is the Topic 65
gate rule, and this is the moment it exists for.

**2. Run `orderflow` with no JVM flags in a starved container.**

```bash
docker run --rm --name orderflow-drill \
  --cpus=0.5 --memory=512m \
  --network orderflow-net \
  -e SPRING_PROFILES_ACTIVE=load \
  -p 8080:8080 \
  orderflow:baseline
```

**No `JAVA_TOOL_OPTIONS`. No `-Xmx`. No collector. Nothing.** That is the point.

**3. Capture the four assigned facts before load starts.**

```bash
# a. availableProcessors, and everything downstream of it
docker exec orderflow-drill jcmd 1 VM.flags -all > drill-flags.txt
docker exec orderflow-drill jcmd 1 VM.command_line

# b. the collector
grep -E "UseSerialGC|UseG1GC|UseParallelGC|UseZGC" drill-flags.txt | grep true

# c. the derived max heap, and its origin
grep -iE "MaxHeapSize|RAMPercentage" drill-flags.txt

# d. GC thread counts and compiler threads
grep -iE "ParallelGCThreads|ConcGCThreads|CICompilerCount|ActiveProcessorCount" drill-flags.txt

# e. what the JVM concluded about the container - restart with this if you did not include it
#    -Xlog:os+container=trace
```

Hit the application's own runtime-facts endpoint (add `RuntimeFactsLogger` from Example 2 if
you have not) and record `availableProcessors`, `maxHeapMB`, `collectors` and
`commonPoolParallelism` from the log line.

**4. Start the recorded Topic 65 k6 scenario, unmodified.** Same script, same arrival rate,
same dataset, same mix. **Do not reduce the load to "be fair to the small container".** The
whole point is to run the recorded scenario against a differently-configured runtime.

**5. Sample continuously while it runs.**

```bash
while docker ps --format '{{.Names}}' | grep -q orderflow-drill; do
  date +%s
  docker exec orderflow-drill jcmd 1 GC.heap_info 2>/dev/null | head -3
  docker exec orderflow-drill jcmd 1 Thread.print 2>/dev/null | grep -c '^"'
  docker exec orderflow-drill cat /sys/fs/cgroup/cpu.stat 2>/dev/null
  docker exec orderflow-drill grep VmRSS /proc/1/status 2>/dev/null
  sleep 10
done
echo "container exited"
docker inspect orderflow-drill --format '{{.State.OOMKilled}} {{.State.ExitCode}}' 2>/dev/null
```

**6. When it dies, capture the cause.**

```bash
docker inspect orderflow-drill --format '{{.State.OOMKilled}} {{.State.ExitCode}}'
docker logs orderflow-drill --tail 100
```

### Part C — what to write down before reading on

Nine facts and two sentences.

1. `availableProcessors()`.
2. The collector, and its **origin** from the flag dump.
3. `MaxHeapSize`, its origin, and it as a percentage of 512 MB.
4. `ParallelGCThreads`, `ConcGCThreads`, `CICompilerCount`.
5. `commonPoolParallelism`.
6. Peak thread count during the run.
7. Peak RSS during the run.
8. Whether the container was OOMKilled, and the exit code.
9. p50/p95/p99 for each endpoint from k6, **and how far outside the ±10% gate they are**.
10. One sentence: *which of the four assigned facts surprised you most, and why?*
11. One sentence: *did the service die of a heap `OutOfMemoryError` or of an OOMKill, and
    how do you know?*

### Part D — how to read it

| What you see | What it means |
|---|---|
| `availableProcessors() = 1` | **The `ceil`.** Half a CPU is one processor to the JVM. |
| `UseSerialGC = true {ergonomic}` | **The surprise most people do not predict.** The container failed the server-class test. You are not running the collector you tuned in Topics 71 and 72. |
| `MaxHeapSize` ≈ a quarter of 512 MB, origin `ergonomic` | The `MaxRAMPercentage` default, on a container whose only job is this JVM. Compare to your measured live set from Topic 70. |
| `commonPoolParallelism = 0` | **Every parallel stream in `orderflow` is now serial.** Confirm with a thread dump during the catalogue endpoint. |
| Thread count climbing with request rate | The `CompletableFuture` thread-per-task fallback, or a pool you did not bound. Group the dump by name prefix to tell which. |
| Exit code 137, `OOMKilled: true` | **The kernel killed it.** No heap dump exists. No exception was thrown. Say why out loud. |
| `OutOfMemoryError: Java heap space` in the logs instead | The **other** failure: the live set does not fit in the derived heap. Both are possible from this configuration, and which you get depends on how the two pressures race. |
| **Both**, on different runs | Entirely expected. Two independent memory failures from one pod spec. Note it; it is the reason this incident is confusing in production. |
| p99 far outside the gate, not marginally | Confirms the point: this is a structurally different runtime, not a slower one. |
| `nr_throttled` large in `cpu.stat` | The quota is actively stopping your threads. Part of your p99 is scheduler enforcement. |
| The container survives the whole run | Your live set or class count is smaller than mine assumed. **That is a fine result** — record the numbers, and increase the dataset or the load until you find the edge. Finding *where* the edge is beats being told it exists. |

### Part E — fix it, and re-run the baseline

Apply the four-layer fix, with **numbers you measured**, not numbers I invented:

1. Explicit collector, chosen after comparing SerialGC and G1 at this container size against
   the baseline. **Do not assume G1 wins on half a CPU.**
2. `-XX:MaxRAMPercentage` from the budget table, after bounding `MaxMetaspaceSize`,
   `ReservedCodeCacheSize`, `MaxDirectMemorySize` and `-Xss` from NMT.
3. `-XX:ActiveProcessorCount` set deliberately, with a written justification — or a CPU
   limit raised to what the measurement says the service needs.
4. `-XX:+ExitOnOutOfMemoryError`, `-XX:+HeapDumpOnOutOfMemoryError`,
   `-XX:NativeMemoryTracking=summary`, GC logging to a file, and `-Xlog:gc+init=debug`.

Then run the **identical** k6 scenario again.

**What the fix proves — and this is the point of the drill, not the fix:**

- The **same image**, the **same load**, the **same dataset**, and the difference is entirely
  in what the JVM was told about the machine. **Configuration built a different runtime.**
- You can now say, with numbers, what `orderflow` actually needs: this many CPUs for this
  latency, this much heap for this live set, this much container memory for this native
  footprint. That is not tuning. That is a **capacity statement**, and it goes straight into
  Topic 129.
- If the fixed 512 MB / 0.5 CPU configuration **still** cannot meet the baseline, that is
  also a finding, and a valuable one: *the service does not fit in this container, and here
  is the smallest one it does fit in.* Cost-reduction conversations need that number, and
  almost nobody has it.

Carry one sentence out of this drill: *a CPU quota is not a throttle; it is a construction
parameter for the runtime.*

---

## Measurement

### The instrument for each claim

Every claim in this document maps to an instrument that could falsify it. That mapping is
the difference between engineering and folklore.

| Claim | Instrument that makes it falsifiable |
|---|---|
| "The JVM read the cgroup limits" | `java -Xlog:os+container=trace -version`, inside the container |
| "`availableProcessors()` is derived from the quota" | run `ContainerFacts` at several `--cpus` values and observe the `ceil` |
| "That number sized the GC threads" | `jcmd <pid> VM.flags -all \| grep GCThreads`, with the origin column |
| "It selected the collector" | `jcmd <pid> VM.flags -all \| grep -E "UseSerialGC\|UseG1GC"` plus `-Xlog:gc+init=debug` |
| "The common pool has no workers" | `ForkJoinPool.getCommonPoolParallelism()` and a thread dump during the operation |
| "The parallel stream is serial" | print `Thread.currentThread().getName()` from inside the stream |
| "`supplyAsync` fell back to thread-per-task" | the thread name in the same test, plus a rising thread count under load |
| "The default heap is a percentage" | `jcmd <pid> VM.flags -all \| grep -iE "MaxHeapSize\|RAMPercentage"` at two container sizes |
| "The heap is too small for the live set" | post-collection occupancy from `-Xlog:gc`, or `GC.run` + `GC.heap_info` |
| "It was the kernel, not the JVM" | `docker inspect --format '{{.State.OOMKilled}} {{.State.ExitCode}}'`; `memory.events` `oom_kill` |
| "The native terms broke the budget" | `jcmd VM.native_memory baseline` then `summary.diff` (Topic 80) |
| "The quota is throttling us" | `/sys/fs/cgroup/cpu.stat`: `nr_throttled`, `throttled_usec`, correlated with p99 |
| "The fix worked" | **the recorded Topic 65 p50/p95/p99, re-run identically** |

That last row is the one people skip. **A tuning change that is not compared against the
recorded baseline is a preference, not a result.** The Topic 65 gate exists precisely so
that every Phase 8 change can be checked against the same scenario, the same dataset and the
same arrival rate.

### Startup-time decomposition — and why it belongs here

Container CPU allocation is one of the largest inputs to startup time, because startup is
CPU-bound: class loading, verification, bean instantiation and JIT compilation all want
CPU, and a quota of 0.5 gives them half of one core's worth.

The four slices, with the instrument for each:

| Slice | What happens | How to measure it | What a CPU quota does to it |
|---|---|---|---|
| **1. JVM init** | VM creation, heap reservation, core classes, agent `premain`s (Topic 81) | `<m> − <n>` from Spring's `Started ... in <n> seconds (process running for <m>)` | pre-touch and heap reservation are slower; agents are slower |
| **2. Class loading** | finding, verifying, defining thousands of classes | `-Xlog:class+load=info` line count; how many came from a CDS archive | verification is CPU-bound; a quota hits this hard |
| **3. Context refresh** | bean definitions, `@Conditional` evaluation, post-processors | Boot's `--debug` condition-evaluation report | CPU-bound |
| **4. Bean instantiation** | constructing singletons, EntityManagerFactory, pool warm-up | `BufferingApplicationStartup` + `/actuator/startup`, sorted by duration | CPU-bound, plus any I/O wait |

Wire slice 4 up permanently — it is three lines and it pays for itself the first time:

```java
SpringApplication app = new SpringApplication(OrderflowApplication.class);
app.setApplicationStartup(new BufferingApplicationStartup(4096));
app.run(args);
```
```bash
curl -s localhost:8080/actuator/startup | jq '.timeline.events
  | sort_by(.duration) | reverse | .[0:20]
  | map({name: .startupStep.name, tags: .startupStep.tags, duration})'
```

**The rule:** *measure the split before you optimise.* Slices 1 and 2 are attacked with
AppCDS or the JDK 25 AOT cache (Topic 122); slices 3 and 4 with lazy initialisation and
fewer auto-configurations. Reaching for native-image (Topic 83) before doing this
decomposition is the single most expensive mistake in this area, and it gets its own trap
in the next document.

### The standing rule: a naive `System.nanoTime()` measurement is wrong

You will be tempted to answer "is `MaxRAMPercentage=70` faster than the default?" like this:

```java
// DO NOT DO THIS. Every number it produces is meaningless.
long start = System.nanoTime();
for (int i = 0; i < 1_000_000; i++) {
    orderService.placeOrder(command);
}
System.out.println((System.nanoTime() - start) / 1_000_000 + " ns/op");
```

Four independent reasons this lies, and you cannot tell which one is lying:

1. **Warm-up state.** The loop begins interpreted, is on-stack-replaced mid-flight, and ends
   in C2 code. **Under a CPU quota this is far worse than usual**, because there are fewer
   compiler threads, so the transition takes longer and a larger fraction of your loop runs
   in slow code (Topic 74).
2. **You are measuring GC, in a ratio you did not choose.** Heap size changes collection
   frequency, so the number of collections that land inside your timed window is a function
   of the very variable you are testing. The measurement and the treatment are entangled.
3. **CFS throttling is invisible to `nanoTime`.** Wall-clock time includes the periods when
   the kernel stopped your threads to enforce the quota. You cannot tell how much of your
   elapsed time was throttling without reading `cpu.stat` separately.
4. **Dead-code elimination and constant folding.** If the result is unused, C2 may prove the
   work has no observable effect and remove it.

**Topic 77 is the full treatment.** The correct shape is a JMH harness with
`@State(Scope.Benchmark)`, `Blackhole.consume`, explicit warm-up iterations and at least
`@Fork(3)` — and for *this* question specifically, each heap or CPU configuration must be a
**separate container**, because these are JVM- and cgroup-level settings that cannot be
toggled inside a fork.

And the honest framing to lead with, before any benchmark: **the right instrument for a
tuning question is the recorded Topic 65 scenario, not a microbenchmark.** Heap size,
collector choice and CPU allocation change GC frequency, pause duration, warm-up rate and
throttling — effects that only compose correctly under a realistic arrival-rate load against
a realistic dataset. A microbenchmark measures one of them and silently omits the rest.

### What to graph in production, permanently

| Metric | Why |
|---|---|
| `availableProcessors()` **as a gauge** | so a pod-spec change shows up on a graph. Almost nobody does this. **Do it.** |
| `jvm_memory_max_bytes{area="heap"}` | the derived heap; a change here means ergonomics moved |
| `jvm_memory_used_bytes{area="heap"}` with the max as a line | the classic; keep it |
| `process_resident_memory_bytes` **and** `container_memory_working_set_bytes` | **the numbers that get you killed** (Topic 80) |
| the container memory limit as a horizontal line on the same panel | the moment RSS approaches it is the moment you page |
| `container_cpu_cfs_throttled_periods_total` / `..._seconds_total` | **throttling, correlated with p99.** A hugely under-used signal. |
| `jvm_gc_pause_seconds` by cause, and collection count | frequency and duration are different problems |
| `jvm_threads_live_threads` | the `CompletableFuture` fallback and thread leaks both show here |
| `jvm_memory_used_bytes{area="nonheap"}` | metaspace + code cache; the native terms in the budget |
| the selected collector, as a label on an info metric | so you never again wonder which one is running |

**Put `availableProcessors` and the CPU limit on the same panel.** The next time someone
edits a pod spec, the runtime change is visible on a dashboard instead of arriving as a
mysterious p99 regression a week later.

---

## Practice exercises

### 1 — Easy: map the ergonomic boundaries on your own JDK

Do not take any threshold in this document on trust. Measure them.

Write a script that runs a trivial JVM across a grid of `--cpus` values (0.25, 0.5, 1, 1.5,
2, 3, 4) crossed with `--memory` values (256m, 512m, 1g, 1792m, 2g, 4g), and for each cell
records:

- `Runtime.availableProcessors()`
- `Runtime.maxMemory()`, and as a percentage of the container limit
- the selected collector
- `ParallelGCThreads`, `ConcGCThreads`, `CICompilerCount`
- `ForkJoinPool.getCommonPoolParallelism()`

Produce a table. Then answer, in writing:

1. Exactly where is the SerialGC/G1 boundary on **your** JDK and base image — in processors
   and in megabytes?
2. Is the heap percentage constant across every cell, or does it change at small memory
   limits? (If it changes, you have just found `MinRAMPercentage` empirically.)
3. At which `--cpus` value does the common pool first get a worker thread?
4. Which single cell would you pick for a service that is mostly waiting on a database, and
   why?

**Why this exercise:** it replaces every hedged number in this document with a measured one,
on the exact JDK you run, in twenty minutes. That table is worth more than the document.

### 2 — Medium: build `orderflow`'s memory budget from measurement (combines Topics 68, 69, 70, 79, 80, 81)

Produce a one-page budget for `orderflow` at the Topic 65 baseline, with every number
sourced.

**A. Live set.** Run the baseline. Force a full collection and read post-collection
occupancy, or take a heap dump and read the retained size in MAT. State your method and its
error bars. *(Topics 70, 79.)*

**B. Object-level sanity check.** Pick the three largest object types from a class
histogram. Using Topic 69's layout rules, compute the expected per-instance size and
multiply by the instance count. Does it roughly agree with the histogram's byte count? If
not, explain the gap. *(Topic 69.)*

**C. Allocation rate.** From `-Xlog:gc`, compute MB/s of allocation at the baseline arrival
rate. State what young-collection frequency that implies for a given eden size. *(Topics 68,
70.)*

**D. Native terms.** NMT summary at steady state: `Class`, `Code`, `Thread`, `GC`, `Other`.
Bound each with a flag and justify the headroom you added. *(Topic 80.)*

**E. Agent overhead.** Repeat D with the OTel agent attached. Report the delta in `Class`
and `Code`, and add it to the budget as an explicit line item. *(Topic 81.)*

**F. The compressed-oops check.** State whether your chosen heap keeps you below the ~32 GB
boundary, and confirm with `jcmd <pid> VM.flags | grep UseCompressedOops`. Explain in two
sentences why this matters for a future scale-up conversation. *(Topic 69.)*

**G. The budget.** Sum everything. Choose a container memory limit. Express the heap as
`MaxRAMPercentage` of that limit. State the headroom explicitly as a number and a
justification.

**H. Validation.** Run the baseline with your budget. Report p50/p95/p99 against the
recorded numbers, RSS against the limit, and whether NMT's committed total explains RSS.

**Success criterion:** someone who disagrees with your headroom can argue with the number
rather than with your judgement, because every other line is measured.

### 3 — Hard: production simulation against the `orderflow` baseline

You are the engineer who owns `orderflow`. A cost-reduction programme has landed the pod
spec from Example 2. You have the recorded Topic 65 baseline and one week.

**Produce the following.**

1. **The incident timeline**, reconstructed from what a real on-call would see: which
   symptom appears first, which alert fires, and which two symptoms are most likely to be
   misdiagnosed as each other.

2. **A reproduction** of at least three of the four failures from Example 2 — serialised
   parallel work, GC pause collapse, thread-count growth, and OOMKill — locally, at
   `--cpus=0.5 --memory=512m`, with evidence for each.

3. **A four-way collector comparison at the constrained size.** SerialGC, ParallelGC, G1 and
   ZGC, each at the same heap, against the recorded k6 scenario. Report p50/p95/p99/p999,
   throughput and CPU. **Be prepared for SerialGC to win**, and if it does, say so — the
   interesting engineering is explaining why a collector everyone dismisses is correct on
   half a core. *(Topic 72's argument, applied.)*

4. **A CPU-allocation experiment.** Three arms: (a) `limits.cpu: 0.5`, no
   `ActiveProcessorCount`; (b) `limits.cpu: 0.5` with `-XX:ActiveProcessorCount=2`; (c) no
   CPU limit with `-XX:ActiveProcessorCount=2`. Report p99 and `cpu.stat` throttling for
   each. State which you would ship and why, including what arm (c) risks that the others do
   not.

5. **The smallest container that meets the SLO.** Search the space and find it. This is the
   number the cost programme actually wants, and nobody else will produce it. Express it as
   CPU, memory, heap percentage, collector and `ActiveProcessorCount`.

6. **A cost model.** Using the Topic 65 throughput per instance, compute instances needed at
   the target request rate, for the original spec and for your recommended spec. Convert to
   a monthly cost delta. **State the break-even honestly**, including cases where the
   cost-reduction programme is right and you should accept a smaller container.

7. **A one-page written response to the cost programme**, addressed to someone who does not
   know what `availableProcessors()` is. It must contain: what actually happened, what it
   costs in latency and in incidents, what you recommend, what that costs, and what you need
   from them. **This artefact is the exercise.** Everything above it is the evidence.

8. **A guardrail** so it cannot recur: a startup assertion in `orderflow` that fails fast if
   `availableProcessors()` is below a threshold, or if the collector is not the expected one,
   or if `MaxHeapSize`'s origin is `ergonomic`. Ten lines of code that convert a silent
   misconfiguration into a refused startup.

**The trap in this exercise:** it is very tempting to conclude "the cost programme was
wrong". It may not have been. `orderflow` may genuinely have been over-provisioned, and the
correct answer may be a smaller container **with explicit flags** — which costs less than the
original *and* meets the SLO. **The engineering position is not "give me my CPUs back"; it is
"here is the smallest configuration that meets the SLO, and here is the evidence".** That
distinction is most of the difference between a senior engineer and a loud one.

---

## Interview questions

### Q1 — "The heap is at 40%. Why did Kubernetes OOMKill the pod?"

**Mid-level answer:** "Something outside the heap is using memory — maybe a native library
or direct buffers. I'd increase the memory limit and see if it stops happening."

**Senior answer:** "Because the heap is one term in RSS, and the cgroup limit applies to RSS.
The full set is committed heap, metaspace and compressed class space, the JIT code cache,
thread stacks, GC data structures like the card table and remembered sets, direct and mapped
byte buffers, JVM internals, and any native library's own `malloc`. Several of those are
unbounded by default — metaspace and direct memory in particular — and metaspace grows with
class count, so an APM agent or a lot of generated proxies moves it.

The kill is `SIGKILL` from the cgroup OOM killer, which the JVM cannot catch, so there is no
`OutOfMemoryError`, no heap dump, and no shutdown hook. **The absence of evidence is the
diagnostic signature** — an OOM with a stack trace is a heap problem; an OOM with nothing but
exit code 137 is a native one.

Concretely: confirm with `kubectl describe pod` showing `OOMKilled` and exit 137, or a
non-zero `oom_kill` in the cgroup's `memory.events`. Then redeploy with
`-XX:NativeMemoryTracking=summary`, take a `VM.native_memory baseline` early and a
`summary.diff` after the growth, and see which category moved. If NMT's committed total is
close to RSS, the JVM allocated it and the diff names it; if RSS is far above NMT's total,
it is a native library or allocator fragmentation and I go to `pmap`.

And the specific thing I would check first in a container: whether `-Xmx` was set equal to
the memory limit. That is the most common single cause, and the fix is to use
`-XX:MaxRAMPercentage` instead — both because it leaves room for the other terms and because
it tracks the limit if someone edits the pod spec. I would bound metaspace, the code cache,
direct memory and the thread pools explicitly before I raised the limit, because raising the
limit on an unbounded term just buys time."

**What separates them:** the mid answer names one possible cause and reaches for the limit.
The senior answer gives the **equation**, explains **why there is no evidence**, gives a
**ranked diagnostic procedure**, names the **most likely single cause in a container**, and
proposes bounding before resizing.

**Follow-up the interviewer asks:** "You said `MaxRAMPercentage` over `-Xmx`. Give me the
operational reason, not the technical one." (Because a pod spec is edited by people who are
not thinking about the JVM. `-Xmx1g` in a 2 GB container is 50%; the same flag in a 512 MB
container after a cost-reduction edit is 200%. A percentage cannot be inverted by someone
else's YAML change.)

---

### Q2 — "How would you size the JVM for a 2 GB / 1 CPU container?"

**Mid-level answer:** "I'd set `-Xmx` to about 1 GB, half the container, and use G1 since
it's the default. Maybe `-XX:MaxRAMPercentage=50`."

**Senior answer:** "I would not answer that from a rule of thumb, and I would say so — but I
can tell you exactly what I would measure and in what order, and I can tell you the things
about that specific shape of container that I would check before anything else.

**Memory.** The input is the measured live set — how many bytes are reachable after a full
collection under realistic load — which I get from our load-test baseline. Heap is then a
multiple of that; 2× to 4× is a starting point for G1, and a low-pause collector needs more
headroom rather than less because it collects concurrently with allocation. Then I subtract
the native terms from the container limit: metaspace, code cache, thread stacks, GC
structures, direct buffers, native libraries. I measure those with NMT rather than guessing,
bound each with a flag, and check the heap I chose fits in what is left. If it does not,
either the container is too small or one of the native terms needs bounding. I express the
result as `MaxRAMPercentage`, not `-Xmx`, so it tracks the limit.

**CPU is the more interesting half of the question, and it is the half people skip.** One CPU
means `availableProcessors()` returns 1, and that number sizes far more than people expect.
It decides GC thread counts. It decides whether the JVM classifies this as a server-class
machine — below roughly two processors it selects **SerialGC**, not G1, and nobody gets told.
It makes `ForkJoinPool.commonPool()`'s parallelism zero, so every parallel stream in the
process runs on the calling thread, and `CompletableFuture.supplyAsync` without an explicit
executor falls back to creating a thread per task. It gives the virtual-thread scheduler one
carrier. And it reduces the JIT compiler threads, so warm-up is slower on every pod start.

So for this container I would: print the flags from **inside** it and read the origin column
so I know what is ergonomic and what is mine; select the collector explicitly after
comparing SerialGC and G1 at this size against the baseline, because SerialGC genuinely may
win on one core; set `ActiveProcessorCount` deliberately if the workload is I/O-bound enough
to justify more pool parallelism than the entitlement; make sure no code path depends on the
common pool; and add `-XX:+ExitOnOutOfMemoryError` with a heap dump so a failure leaves
evidence.

Then I would re-run the recorded baseline and compare p50/p95/p99, and record the whole
configuration alongside the numbers — because a baseline that does not name its container
limits, collector and processor count is not reproducible."

**What separates them:** the mid answer gives a number. The senior answer **refuses to give a
number without a measurement**, then demonstrates that the CPU half of the question is the
more consequential half, names the specific cascade, and finishes with reproducibility. The
SerialGC point in particular is the one that makes an interviewer sit up, because most
candidates do not know the collector is chosen for them.

**Follow-up the interviewer asks:** "You said SerialGC might win. Defend that." (On one core
there is no parallelism for a parallel collector to exploit, and G1's concurrent marking
threads and write barriers cost CPU that a 1-CPU container does not have. G1 buys shorter
pauses by spending CPU concurrently; if there is no spare CPU, you pay the cost and get
little of the benefit. It is an empirical question and I would measure both — but the
assumption that G1 is always better is exactly the kind of thing a container invalidates.)

---

### Q3 — "Our service is slower in Kubernetes than on the build agent. Same image. Where do you look?"

**Mid-level answer:** "The build agent probably has more CPU and memory. And there is network
latency to the database in the cluster that we don't have locally. I'd compare resource usage
between the two."

**Senior answer:** "Both of those are worth checking, but I would start somewhere more
specific: **the same image is not the same runtime.** The JVM makes ergonomic decisions from
what it believes the machine is, and a cgroup changes what it believes.

My first command is `jcmd 1 VM.flags -all` inside the pod, and the same on the build agent,
and I diff them. I am reading the **origin** column: anything marked `ergonomic` is a number
the JVM chose from the machine, and those are exactly the ones that differ. In my experience
the diff usually shows a different `MaxHeapSize`, different `ParallelGCThreads` and
`ConcGCThreads`, a different `CICompilerCount`, and — the one that surprises people — a
**different collector**, because a small container fails the server-class test and gets
SerialGC while the build agent gets G1.

Second, `Runtime.availableProcessors()` in both places. If the pod has a CPU limit at or below
one core, that returns 1, and the whole cascade follows: the common ForkJoinPool has
parallelism zero, so parallel streams run on the calling thread, and `CompletableFuture`
without an explicit executor falls back to a thread per task.

Third, `-Xlog:os+container=trace -version` inside the pod, which prints the JVM's container
detection decision by decision, and usually ends the investigation on its own.

Fourth, `/sys/fs/cgroup/cpu.stat`. If `nr_throttled` is large, part of the latency is the CFS
quota stopping threads, which looks exactly like a GC pause on a dashboard and is not one.

And fifth, warm-up: fewer compiler threads means a longer ramp to peak, so if we scale out to
many small pods, a meaningful fraction of traffic may permanently be served by
not-yet-optimised code. That is a real argument for fewer, larger pods for a JIT runtime, and
it is one people rarely make."

**What separates them:** the mid answer compares resources. The senior answer knows the
container **reconfigures the runtime**, names the exact command that reveals it, reads the
**origin column** — which is the specific expert move here — and adds two causes (throttling,
warm-up) that do not appear on any resource graph.

**Follow-up the interviewer asks:** "The flag diff shows the collector changed. Is that
necessarily the cause of the slowdown?" (No. It is a difference, not a diagnosis. I would
force the same collector in both places and re-run; if the gap persists, the collector was a
red herring and I move to the next difference. Naming a difference is the start of the
investigation, not the end of it.)

---

### Q4 — "We set `requests.cpu` but no `limits.cpu`, because we heard CPU limits hurt latency. Is that right?"

**Mid-level answer:** "Yes, CPU limits cause throttling and that hurts p99. Removing the limit
lets the pod burst, so it should be faster."

**Senior answer:** "The throttling argument is real — CFS quota enforcement produces latency
spikes that look like GC pauses on a dashboard and are not — so the instinct is defensible.
But applied to a JVM without one extra step, it creates a different and often worse problem.

`requests.cpu` becomes a cgroup weight or share. It is a scheduling hint under contention, not
a quota, and on modern JDKs it is not what the JVM derives its processor count from. So with
no limit set, the JVM concludes it has the **whole node**. On a 64-core node it will size GC
threads, the common ForkJoinPool, the virtual-thread scheduler and the compiler threads for
64 processors — inside a cgroup that may be entitled to a fraction of one core under
contention.

The symptom is very long stop-the-world pauses that are far longer than the heap size
justifies, with `-Xlog:safepoint` showing time spent after reaching the safepoint, and
`cpu.stat` throttling spiking during GC. All those GC threads become runnable at once and the
kernel time-slices them. And it gets **worse** when the cluster gets busier, and worse when
you move to a bigger node, which is exactly the opposite of what people expect.

So: dropping the CPU limit is a reasonable choice, **but only together with an explicit
`-XX:ActiveProcessorCount`.** That decouples the JVM's internal sizing from whether anyone
happened to set a limit. Without it you have replaced a known throttling cost with an unknown
one.

I would also say the two options are testable rather than arguable. Three arms against the
recorded load-test baseline: limit set; limit set with an explicit processor count; no limit
with an explicit processor count. Report p99 and `cpu.stat` throttling for each. That
converts a Kubernetes-philosophy argument into a measurement, which is usually the right move
when two reasonable engineers disagree."

**What separates them:** the mid answer repeats correct received wisdom without knowing what
it breaks. The senior answer knows **why** the advice exists, knows the **specific JVM
consequence** of following it naively, names the **one flag** that makes it safe, and
converts the disagreement into an experiment.

**Follow-up the interviewer asks:** "How would you catch this in review, before it ships?"
(A policy check: any JVM workload with no `limits.cpu` must set `-XX:ActiveProcessorCount`.
And an application-level startup assertion that logs `availableProcessors()` and refuses to
start if it is wildly different from the expected value — ten lines that turn a silent
misconfiguration into a refused deployment.)

---

### Q5 — "Someone added `.parallel()` to a stream and it got slower in production but faster locally. Explain."

**Mid-level answer:** "Parallel streams have overhead — splitting and merging cost something,
and if the work per element is small the overhead dominates. Production probably has a
different data size."

**Senior answer:** "That is one real cause, and Topic 25's preconditions cover it: you need a
splittable source, enough elements times enough per-element cost to amortise the fork/join
machinery, no shared mutable state, and an associative reduction. If any of those fail,
parallel is slower everywhere — but it would be slower locally too, so it does not explain
the local/production split.

The container explanation does. `parallelStream()` uses `ForkJoinPool.commonPool()`, whose
parallelism is `availableProcessors() - 1` with a floor of zero. In production, if the pod has
a CPU limit at or below one core, `availableProcessors()` returns 1 — the derivation is
`ceil(quota/period)`, so even 0.25 of a CPU yields 1 — and the common pool's parallelism is
**zero**. A ForkJoinPool with parallelism zero runs submitted tasks **in the calling thread**.

So in production the stream is completely serial *and* still pays for the splitting, the
task objects and the combining. It is strictly slower than the sequential version. Locally,
on an 8-core laptop, the pool has seven workers and it genuinely parallelises. Same code,
same data, opposite result, and **nothing anywhere logs a warning.**

I would confirm it in two commands: `ForkJoinPool.getCommonPoolParallelism()` from inside the
pod, and a thread dump during the slow endpoint showing the work on `http-nio-*` request
threads rather than on `commonPool-worker-*` threads.

There is a second-order effect worth mentioning too: with common-pool parallelism at or below
one, `CompletableFuture.supplyAsync` without an explicit executor stops using the common pool
and creates a **thread per task** instead. So the same pod spec makes one API silently serial
and another silently unbounded. If the service is also seeing thread-count growth, that is
probably why.

The fix is not to tune the common pool. It is to stop using a JVM-global pool whose size is
decided by a pod spec someone else wrote — pass an explicit executor, sized for the workload.
And honestly, for a request handler on a small container, I would question whether the
parallel stream should be there at all."

**What separates them:** the mid answer knows the generic overhead argument. The senior answer
**explains the local/production asymmetry specifically**, gives the exact arithmetic
(`ceil`, minus one, floor at zero), names the two-command confirmation, adds the
`CompletableFuture` second-order effect that ties in an unrelated-looking symptom, and ends
with the design fix rather than a tuning knob.

**Follow-up the interviewer asks:** "Could you just set
`-Djava.util.concurrent.ForkJoinPool.common.parallelism`?" (You can, and it will create worker
threads. But you now have N runnable threads inside a cgroup entitled to a fraction of a core,
so the kernel throttles them; more threads does not create more CPU. It can help when the
tasks are mostly blocked and hurt when they are mostly running. It is a measurement, not a
fix, and the real fix is an explicit executor.)

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. `availableProcessors()` returns `ceil(quota/period)`, so 0.25 CPU and 1.0 CPU both report
   1. Design a better contract for the JVM to expose. What would it return, what would GC
   thread sizing do with it, and name one thing your design would make worse.

2. `ForkJoinPool.commonPool()` has parallelism `availableProcessors() - 1`. Derive **why the
   minus one** from what the pool is for, without recalling it from the text. Then explain
   why the floor is zero rather than one, and what the JDK does with a zero-parallelism pool.

3. `requests.cpu` becomes a weight and `limits.cpu` becomes a quota. Construct the scenario
   in which **removing** a CPU limit makes p99 dramatically worse for a JVM, and name the two
   observations that would distinguish it from an ordinary GC problem.

4. The JVM selects SerialGC below the server-class thresholds. Argue that this is the
   **correct** default for a 1-CPU 512 MB container, using Topic 71 and 72's material. Then
   argue the opposite. Which measurement decides it?

5. Topic 69 says compressed oops stop working above roughly 32 GB of heap. Topic 70 says heap
   size follows from the live set. Combine them into a rule for a service whose live set is
   growing 20% a quarter, and name the moment you would have to make a decision.

6. `-XX:+AlwaysPreTouch` trades startup time for a predictable RSS. In a Kubernetes
   deployment with a rolling update and a readiness probe, describe precisely who pays that
   cost and when — and construct the case where it makes an outage **more** likely rather
   than less.

7. A colleague proposes `-XX:ActiveProcessorCount=4` on a pod with `limits.cpu: 500m`,
   because "the service is I/O-bound and we want the pools sized properly". Give the
   strongest argument for and the strongest argument against, then name the single experiment
   that settles it and the metric it would turn on.

---

## Quick reference card

### JVM flags

| Flag | What it does | When to set it |
|---|---|---|
| `-XX:+UseContainerSupport` | read cgroup limits (default **on**) | leave on; turning it off is almost always wrong |
| `-XX:ActiveProcessorCount=<n>` | **override** the derived processor count | when there is no CPU limit, or when the workload is measurably I/O-bound |
| `-XX:MaxRAMPercentage=<d>` | max heap as a % of the detected limit | **prefer over `-Xmx` in a container** — it tracks the limit |
| `-XX:InitialRAMPercentage=<d>` | initial heap as a % of the limit | set equal to max to avoid heap resizing under load |
| `-XX:MinRAMPercentage=<d>` | used instead of max for small limits | know it exists; verify its threshold on your JDK |
| `-XX:MaxRAM=<bytes>` | override what the JVM believes total memory is | rarely; when detection is wrong and you cannot fix it |
| `-Xmx` / `-Xms` | absolute heap sizes | outside containers, or when you deliberately want a fixed number |
| `-XX:+UseG1GC` / `-XX:+UseSerialGC` / `-XX:+UseZGC` | **select the collector explicitly** | **always in production.** Ergonomics are for people who have not measured. |
| `-XX:ParallelGCThreads=<n>` | stop-the-world GC worker threads | when the derived count does not match your entitlement |
| `-XX:ConcGCThreads=<n>` | concurrent GC threads | same |
| `-XX:MaxMetaspaceSize=<n>m` | bounds metaspace | **always in a container** — unbounded by default (Topic 80) |
| `-XX:ReservedCodeCacheSize=<n>m` | bounds the JIT code cache | when NMT `Code` is a large term, or an agent is attached |
| `-XX:MaxDirectMemorySize=<n>m` | bounds direct buffers | **always in a container** — default derives from max heap |
| `-Xss<n>k` | per-thread stack size | when NMT `Thread` is large; bound the pools first |
| `-XX:+AlwaysPreTouch` | commit and touch the heap at startup | when you want RSS predictable from second one; costs startup |
| `-XX:+ExitOnOutOfMemoryError` | die rather than limp | usually yes in Kubernetes — let the orchestrator restart |
| `-XX:+HeapDumpOnOutOfMemoryError` `-XX:HeapDumpPath=/dumps` | leave evidence | always, with a mounted volume (Topic 79) |
| `-XX:NativeMemoryTracking=summary` | per-category native accounting | **always** when sizing a container (Topic 80) |
| `-Djava.util.concurrent.ForkJoinPool.common.parallelism=<n>` | common-pool size, independent of CPU count | only with measurement; the real fix is an explicit executor |
| `-Djdk.virtualThreadScheduler.parallelism=<n>` | carrier threads for virtual threads | when the derived count is wrong (Topic 101) |

### Diagnostic commands

| Command | What it answers |
|---|---|
| `java -Xlog:os+container=trace -version` | **what the JVM concluded about the container** — the first command to run |
| `java -XshowSettings:system -version` | the OS metrics block: provider, quota, period, shares, memory limit |
| `java -XX:+PrintFlagsFinal -version \| grep -i ramp` | the RAM-percentage defaults on **your** JDK |
| `java -XX:+PrintFlagsFinal -version \| grep -i ActiveProcessor` | the processor-count override's default |
| `jcmd <pid> VM.flags -all` | **every flag's value AND ORIGIN** on the live process — the money command |
| `jcmd <pid> VM.command_line` | the actual command line, including injected `JAVA_TOOL_OPTIONS` |
| `jcmd <pid> GC.heap_info` | heap used, committed, per-region detail |
| `jcmd <pid> GC.run` then `GC.heap_info` | a rough live-set reading (Topic 70) |
| `jcmd <pid> Thread.print \| grep -c '^"'` | live thread count; group by name prefix to find the pool |
| `jcmd <pid> VM.native_memory summary` / `summary.diff` | the native terms in the budget (Topic 80) |
| `-Xlog:gc+init=debug` | **the selected collector and its thread counts, at startup** |
| `-Xlog:gc*:file=/logs/gc.log:time,uptime,level,tags` | the GC log (Topic 71) |
| `-Xlog:safepoint` | time-to-safepoint vs pause duration (Topic 73) |
| `stat -fc %T /sys/fs/cgroup` | `cgroup2fs` = v2, `tmpfs` = v1 |
| `cat /sys/fs/cgroup/cpu.max` | v2 quota and period, as the kernel sees them |
| `cat /sys/fs/cgroup/memory.max` | v2 memory limit |
| `cat /sys/fs/cgroup/cpu.stat` | **`nr_throttled` and `throttled_usec` — throttling, correlated with p99** |
| `cat /sys/fs/cgroup/memory.events` | `oom_kill` counter — proof the kernel killed you |
| `docker inspect <id> --format '{{.State.OOMKilled}} {{.State.ExitCode}}'` | `true` / `137` |
| `kubectl exec <pod> -- jcmd 1 VM.flags -all` | **run every flag check from INSIDE the container** |

### The cascade — say it from memory

```
cgroup cpu quota
  -> availableProcessors() = ceil(quota / period), also bounded by affinity/cpuset
     -> ParallelGCThreads, ConcGCThreads
     -> server-class test -> COLLECTOR SELECTION (SerialGC below ~2 cpu / ~1792 MB)
     -> ForkJoinPool.commonPool() parallelism = n - 1, floor 0
          -> parallel streams
          -> CompletableFuture.supplyAsync (falls back to thread-per-task when <= 1)
     -> virtual-thread scheduler parallelism = n
     -> CICompilerCount -> warm-up rate
     -> most library pool defaults
```

### The memory budget — say it from memory

```
container limit
  - metaspace - code cache - thread stacks - GC structures
  - direct buffers - native libraries - allocator slack - headroom
  = the heap you may have, expressed as MaxRAMPercentage
```

### Gotchas checklist

- [ ] `availableProcessors()` is `ceil(quota/period)` — 0.25 CPU and 1 CPU both give **1**.
- [ ] Common-pool parallelism is `n - 1`, floor **0** — at 1 CPU, parallel streams are serial.
- [ ] `CompletableFuture.supplyAsync` falls back to **thread-per-task** when common-pool
      parallelism ≤ 1.
- [ ] Below ~2 processors or ~1792 MB the JVM ergonomically selects **SerialGC**.
- [ ] `requests.cpu` is a **weight**, not a quota. No limit means the JVM believes it owns the
      node.
- [ ] No CPU limit **requires** `-XX:ActiveProcessorCount`, or GC threads size for the node.
- [ ] `MaxRAMPercentage` defaults to a **conservative multi-tenant** value; your container is
      not multi-tenant.
- [ ] Prefer `MaxRAMPercentage` over `-Xmx` in a container — it survives a pod-spec edit.
- [ ] `-Xmx` = container limit means every native term is unfunded (Topic 80).
- [ ] Metaspace, the code cache and direct memory are **unbounded by default**.
- [ ] `MaxDirectMemorySize` defaults from **max heap**, not the container limit.
- [ ] Never size a heap into the ~32–48 GB band (compressed oops, Topic 69).
- [ ] Print flags from **inside** the container. A laptop dump tells you nothing.
- [ ] Read the **origin** column — `ergonomic` values change when the container changes.
- [ ] Correlate `cpu.stat` `nr_throttled` with p99 before blaming GC.
- [ ] Record the container limits, collector and `availableProcessors()` **in the baseline**.

---

## When would I use this at work?

**1. The cost-reduction ticket.**
Someone proposes halving the CPU and memory of every JVM service. Instead of arguing from
principle, you run the recorded baseline at the proposed size, produce a table of p50/p95/p99
and throughput, find the **smallest container that still meets the SLO**, and bring that
number to the conversation. Sometimes it vindicates the proposal and you save money;
sometimes it shows the proposal costs three extra instances and saves nothing. Either way you
are the person with the number, and the discussion takes an hour instead of a quarter.

**2. Reviewing a pod spec — for a JVM specifically.**
You have reviewed hundreds of pod specs. Now you read four extra things: does `limits.cpu`
round down to one processor and does the code use parallel streams or un-executored
`CompletableFuture`s; is there a CPU limit at all, and if not is `ActiveProcessorCount` set;
is `-Xmx` present and equal to the memory limit; are metaspace, code cache and direct memory
bounded. Four questions, thirty seconds, and they catch the majority of JVM-in-Kubernetes
incidents before they ship.

**3. The "it's slower in Kubernetes" investigation.**
Someone reports that the same image behaves differently in the cluster. You run
`jcmd 1 VM.flags -all` in both places and diff, reading the origin column. In ten minutes you
have either found the ergonomic difference — a different collector, a different heap, a
different GC thread count — or ruled the whole class out and moved on to the network. That is
the difference between a targeted investigation and a week of guessing, and it is entirely
mechanical once you know the flags exist.

---

## Connected topics

**Prerequisites:**

- **65 — the load-test gate.** Every sizing decision here is validated against the recorded
  p50/p95/p99 and throughput. Without that baseline, tuning is preference. And the baseline
  must record the container limits, the collector and `availableProcessors()`, or it is not
  reproducible.
- **68 — memory areas.** Metaspace is native and outside the heap, and unbounded by default.
  That is a line in the budget, and the reason a class-heavy application dies in a small
  container with a healthy heap.
- **69 — object layout and compressed oops.** The ~32 GB cliff is a hard capacity fact that
  constrains heap sizing at the top end, and object-layout arithmetic is how you sanity-check
  a live-set measurement.
- **70 — GC fundamentals.** The **live set is the input to heap sizing** — the single most
  important sentence in this document's memory half. Allocation rate is the input to
  collection frequency, which is the input to p99.
- **71 — G1 in depth.** What the collector actually does with the threads a CPU quota gives
  it, and what evacuation failure looks like when the concurrent cycle cannot keep up because
  it has too few threads.
- **72 — ZGC and Shenandoah.** Collector choice follows a latency budget. This document adds
  the constraint that collector choice may have already been made for you, ergonomically,
  by the container's size.
- **73 — safepoints and time-to-safepoint.** Under CFS throttling, the time to *reach* a
  safepoint can dominate the pause, and `-Xlog:safepoint` is how you separate them.
- **74 — tiered compilation and warm-up.** Compiler thread count is derived from the
  processor count, so a CPU quota lengthens warm-up on every pod start — the strongest
  technical argument for fewer, larger pods.
- **79 — memory leaks and heap dumps.** How you measure the live set carefully, and what to
  do when the heap genuinely is too small because something is retaining.
- **80 — off-heap memory and NMT.** **The direct predecessor.** That document gives you the
  RSS equation; this one turns it into a budget with a container limit at the bottom.
  `summary.diff` is the shared instrument and `-Xmx` = the limit is the shared trap.
- **81 — instrumentation agents.** An agent adds metaspace and code cache to the budget, and
  more compilation work to a compiler thread pool that a CPU quota already shrank.

**This unlocks:**

- **83 — GraalVM native-image.** The next document, and the alternative answer to "our
  startup and footprint are too big for these containers". Native image changes which terms
  exist in this budget entirely — no metaspace, no code cache, no compiler threads — at the
  cost of the JIT. Read Topic 83 with this document's budget table in hand.
- **100 — `ForkJoinPool` and work stealing.** The common pool whose parallelism this document
  showed you is derived from the container. Topic 100 is what that pool is actually for and
  why blocking in it is catastrophic.
- **101 — virtual threads.** The scheduler's carrier count is `availableProcessors()`.
  Everything about pinning is more severe when there is one carrier, so the severity of Topic
  101's drill is set by a number in a YAML file.
- **103 — NIO and Netty's event loop.** Netty's default `EventLoopGroup` size comes from the
  same processor count, so a CPU quota decides how many event loops multiplex your
  connections.
- **119 — OpenTelemetry.** The agent's cost lands in the budget you built here, and the
  exporter's threads land in the thread count.
- **121 — Actuator and Kubernetes probes.** Slower startup under a CPU quota interacts
  directly with `initialDelaySeconds` and with liveness probes that restart a pod which was
  merely still warming up.
- **122 — layered jars, AppCDS and startup.** The startup decomposition in this document's
  Measurement section is Topic 122's core technique, and AppCDS is the cheapest of the
  startup fixes.
- **129 — capacity, cost and latency budgets.** **The direct consumer.** The budget table and
  the "smallest container that meets the SLO" number from the hard exercise are literally the
  inputs to that model. "How many pods" is downstream of "how many bytes and how many
  processors per pod, and which of them are unbounded".

---

*Java baseline 21, running on JDK 25. Five things in this document are deliberately hedged
rather than asserted: the exact defaults of `MaxRAMPercentage`, `InitialRAMPercentage` and
`MinRAMPercentage` on your JDK build (and `MinRAMPercentage`'s threshold in particular); the
exact server-class thresholds that select SerialGC over G1; the current treatment of
`cpu.shares` in the active-processor-count derivation, which has changed across releases; the
precise `ParallelGCThreads`/`ConcGCThreads` heuristics; and anything about Project Leyden's
ahead-of-time cache, where I have given you the JEP index rather than a flag name I might
misremember. Every one of them has a command attached — `java -XX:+PrintFlagsFinal -version`,
`jcmd <pid> VM.flags -all`, `java -Xlog:os+container=trace -version`,
`java -XshowSettings:system -version` — and the easy exercise turns all of them into measured
numbers on your own JDK in twenty minutes. Everything else — that `availableProcessors()` is
`ceil(quota/period)`, that the common pool's parallelism is that number minus one with a floor
of zero, that a zero-parallelism ForkJoinPool runs tasks in the calling thread, that
`CompletableFuture` falls back to thread-per-task below parallelism 2, that collector
selection is ergonomic, and that heap is one term in RSS while the cgroup limit applies to all
of it — is specified or long-stable behaviour, and will still be true the next time a pod spec
change quietly builds you a different runtime.*
