# 100 — ForkJoinPool and Work Stealing

## Phase: 9 — Concurrency
## Category: DIFFERENTIATOR
## Java baseline: 21  |  Notes features from: 21 (the pool has been in the JDK since 7; the constructor gained saturation and spare controls in 9, and JDK 21 makes it the substrate for the virtual-thread scheduler)
## Project spine: the `orderflow` nightly revenue-by-product rollup over 5M order lines — the one genuinely CPU-bound, splittable workload in the system. Also the diagnosis for why the Topic 91 fan-out degraded unrelated work when it ran on the common pool.

---

## Mechanical statement

Read this twice. Everything below is elaboration.

> **Each worker thread in a `ForkJoinPool` owns its own double-ended queue (a
> "deque").**
>
> **It pushes new subtasks onto its own end, and pops from its own end — LIFO —
> because the task it just created is the one whose data is still hot in L1 cache.**
>
> **When a worker runs out of its own work, it becomes a thief: it steals from the
> TAIL of another worker's deque — the oldest task there, which in a
> divide-and-conquer split is the largest remaining chunk.**
>
> **The whole design assumes tasks that are CPU-bound, recursively splitting, and
> non-blocking. A worker that parks in a blocking call cannot steal, cannot help, and
> cannot be replaced — because the pool has no way to know it is blocked.**

That last sentence is the failure mode, and it is the source of the bugs in Topics 25
and 91 that you have already met.

---

## The bridge from what you know

### NO TYPESCRIPT ANALOGUE — and the reason is structural, not incidental

I am not going to build you a bridge here, because building one would install a model
you would have to demolish later. Work stealing has no equivalent anywhere in your
Node experience, and it is worth being precise about *why*, because the reason is the
same reason the JVM has a memory model and Node does not.

**Your event loop has one queue.** Callbacks are appended and drained in order. There
is nothing to steal from, because there is only one place work lives, and only one
thread to take it. The macrotask queue, the microtask queue and the timer heap are all
single-consumer structures.

**`worker_threads` do not share a heap.** Each worker is a separate V8 isolate with
its own heap. Passing work between them means either structured-cloning the payload
across the boundary or moving a `SharedArrayBuffer`. So a "steal" in Node would not be
a pointer handoff — it would be a serialisation. That cost is orders of magnitude
larger than the cost of the work being stolen for any task small enough to be worth
stealing. **The economics of stealing do not exist in a shared-nothing runtime.**

**Pool libraries like Piscina are not work stealing.** They maintain one shared task
queue and hand tasks to whichever worker is free — a *shared queue with a dispatcher*.
That is a fundamentally different structure. A shared queue has one contention point
that every worker hits on every task. Work stealing has N private queues that are
uncontended in the common case and only contended on the rare steal. That difference is
the entire performance argument for the design, and it only makes sense when all the
workers can reach the same objects for free.

**What you should take from this section:** when you read Java code that submits to a
`ForkJoinPool`, do not reach for an intuition from Node. You do not have one. Read the
mechanics below instead. This is one of the few places in the whole curriculum where
your existing model is not partially right — it is silent.

### The one thing that does transfer: "don't block the loop"

There is a *discipline* that transfers, even though the mechanism does not.

You already know, viscerally, that you must not do `fs.readFileSync` or a synchronous
`crypto.pbkdf2Sync` on the main thread of a Node server, because it stops everything.
You do not have to think about it; the reflex is installed.

Java's common `ForkJoinPool` deserves exactly that reflex, for a structurally similar
reason: it is a **small, shared, JVM-global resource that everything quietly uses**, and
blocking in it stops unrelated work. The difference is that Node made this obvious to
you — there is one thread and you know its name — while Java hides it. Nothing in
`list.parallelStream().map(...)` tells you that you just submitted to a JVM-wide pool
sized to `cores - 1` that a library you have never heard of is also using.

**Verdict: NO ANALOGUE for the mechanism. A transferable reflex for the hazard.**

---

## What is this?

`ForkJoinPool` is a thread pool with a different internal structure and a different
intended workload from `ThreadPoolExecutor` (Topic 90).

`ThreadPoolExecutor` has **one shared queue**. Every worker takes from it. Every take
contends on the same lock. That is fine when tasks are chunky and independent — a
request handler, a background job — because the per-task overhead is small relative to
the task.

`ForkJoinPool` has **one deque per worker**. It is built for a specific shape of
computation: a task that splits itself into smaller tasks, recursively, until the
pieces are small enough to compute directly, then combines the results back up. Merge
sort. Tree traversal. Aggregating five million order lines by product.

For that shape, the per-task overhead matters enormously — you might create tens of
thousands of tasks — so the design goes to considerable lengths to make the common case
(a worker running its own tasks) completely uncontended.

### Where you have already been using it without knowing

Three places, and this is why the topic matters even if you never write a
`RecursiveTask` yourself:

1. **`stream.parallel()`** (Topic 25) runs on `ForkJoinPool.commonPool()`.
2. **`CompletableFuture`'s `*Async` methods with no executor argument** (Topic 91)
   run on `ForkJoinPool.commonPool()`.
3. **The virtual-thread scheduler** (Topic 101) is *a* `ForkJoinPool` — a dedicated one,
   in FIFO mode, not the common pool. You will meet it next topic.

Points 1 and 2 share one JVM-global pool sized to roughly `cores - 1`. Point 3 is
separate. Confusing those two pools is a very common error, so hold the distinction
from the start.

### The task types

```java
RecursiveTask<V>      // splits, computes, RETURNS a value.   compute() returns V
RecursiveAction       // splits, computes, returns nothing.   compute() returns void
CountedCompleter<T>   // for computations that complete asynchronously; used internally
                      // by the streams implementation. You will rarely write one.
ForkJoinTask<V>       // the base class. fork(), join(), invoke(), invokeAll()
```

The methods you actually use:

| Method | What it does |
|---|---|
| `fork()` | Push this task onto the **current worker's own deque** and return immediately. If called from outside a worker, it goes to the common pool's submission queue. |
| `join()` | Wait for this task's result — but "wait" here does not mean park. See Machine-level reality; the calling worker *helps*. |
| `invoke()` | `fork()` plus `join()`, but optimised: runs it directly if it can. |
| `invokeAll(a, b)` | Fork all but the first, run the first directly, then join. The correct idiom for a two-way split. |
| `compute()` | The method you write. The recursion lives here. |

---

## Why does it matter?

### 1. It explains two bugs you already produced

Topic 25's drill: you blocked inside `ForkJoinPool.commonPool()` and watched an
unrelated parallel stream's latency collapse. Topic 91's drill: you ran blocking JDBC
on the common pool and watched unrelated work stall.

You know *that* those happen. This topic is where you learn *why*, precisely enough to
predict the next instance before you cause it. The answer is one sentence: **a parked
worker cannot steal.** Everything else follows.

### 2. It is the substrate for virtual threads

Topic 101 is the most important topic in this phase for you, and its scheduler is a
`ForkJoinPool`. The carrier-starvation failure you are about to study is a
work-stealing pool whose workers have been rendered unable to steal. You cannot
understand pinning properly without understanding what a carrier thread is being
prevented from doing, and that is this topic.

### 3. It is a genuine interview differentiator

"ForkJoinPool is just a thread pool" is a mid-level answer. The senior answer names the
deque, the LIFO/FIFO asymmetry, the reason for each direction, and the workload
assumption that makes it all valid. That answer takes twenty seconds and it changes the
temperature of the room, because it is specific and it is mechanical.

### 4. `orderflow` has exactly one workload where it is right, and several where it is wrong

The nightly revenue rollup over 5M order lines: CPU-bound, splittable, in-memory,
non-blocking. Correct use.

The order-detail fan-out (product + inventory + payment status): three blocking network
calls. **Wrong use**, catastrophically so, and it is the drill from Topic 91.

Being able to tell those apart on sight, from the workload shape rather than from a rule
of thumb, is the skill.

---

## Machine-level reality

### The deque, and the two ends

Each worker owns a `WorkQueue`. Internally it is a circular array plus two indices:

```
                 base                                    top
                  |                                       |
      +-----+-----+-----+-----+-----+-----+-----+-----+-----+
      |     |     | T4  | T3  | T2  | T1  | T0  |     |     |
      +-----+-----+-----+-----+-----+-----+-----+-----+-----+
                  ^                                 ^
            THIEVES take here                OWNER pushes and pops here
            (oldest task, biggest chunk)     (newest task, hottest cache)
```

- **`top`** is the owner's end. `push` increments it; `pop` decrements it. The owner is
  the only thread that touches `top`, so in the uncontended case there is no CAS on the
  push at all — just a store and a memory fence for visibility.
- **`base`** is the thieves' end. A thief reads `base`, reads the slot, and CASes `base`
  forward to claim the task. Multiple thieves may race; the CAS decides.
- Both indices are accessed through `VarHandle`s with explicit ordering semantics, and
  the array slots are read and written with acquire/release ordering. This is a lock-free
  structure, and its correctness rests on exactly the kind of happens-before argument you
  learned to falsify in Topic 99.

The array slots are also **padded relative to each other in the sense that matters**: the
`WorkQueue` objects themselves are laid out to reduce false sharing between the owner's
hot `top` field and the thieves' hot `base` field, because those two are written by
different cores and would otherwise ping-pong a cache line on every push (Topic 96).

### Why the owner takes LIFO

Two independent reasons, and both are worth being able to state.

**Cache locality.** In a recursive split, the task you just pushed operates on the data
you were just touching. Its inputs are in L1. Popping it immediately means the next
computation reads memory that is already in the nearest cache. Taking the *oldest* task
instead would mean jumping back to data evicted several thousand instructions ago.

**Stack discipline.** LIFO on the deque mirrors the call stack of the sequential version
of the same algorithm. A worker executing its own deque LIFO is, in effect, running the
recursion depth-first — which means the number of live, un-completed tasks stays
proportional to the recursion depth rather than to the total task count. That is a memory
bound, and it is why a divide-and-conquer FJP job does not allocate a million live task
objects even when it creates a million tasks.

### Why the thief takes FIFO — from the tail

A steal is expensive. It is a contended CAS, plus a cache-line transfer of the task
object from the victim's core to the thief's core, plus the cold-cache cost of operating
on data the thief has never touched.

So you want to steal *rarely* and steal *big*.

In a divide-and-conquer split, the task at the **base** of the deque is the one pushed
earliest, which means it is highest in the recursion tree, which means it represents the
**largest remaining subtree of work**. Stealing it transfers a large chunk in one
operation, and the thief can then work on it privately for a long time before needing to
steal again.

Stealing from the top instead would grab a leaf: a tiny amount of work, and you would be
back stealing again microseconds later. The asymmetry — LIFO local, FIFO steal — is
therefore not an arbitrary choice. It is two different optimisation targets on two ends
of the same structure.

There is a second, subtler benefit: because the owner and the thief operate on opposite
ends, they only contend when the deque is nearly empty. A deque with a few tasks in it is
accessed concurrently with no contention whatsoever.

### Submission queues — how external work gets in

When a thread that is *not* an FJP worker submits a task (your request thread calling
`commonPool().submit(...)`, or a parallel stream started from a Tomcat thread), it cannot
push onto a worker's deque — it does not own one.

Instead, the pool maintains a set of **submission queues** in the same shared array as
the worker queues. The submitting thread picks one by hashing a thread-local probe value.
Two different external submitters therefore usually land in different submission queues,
which keeps external submission from becoming a single contention point.

Workers then take from submission queues the same way they steal: FIFO, from the base.

This is why `getQueuedSubmissionCount()` and `getQueuedTaskCount()` are two different
numbers, and why the distinction matters when you diagnose a stall. Submissions piling up
means external work is arriving faster than workers can start it. Queued tasks piling up
means work is being *created* by running tasks faster than it is being completed.

### `join()` is not a park — it is "help"

This is the single most elegant thing in the design and the thing most people do not
know.

When a worker calls `task.join()` on a task that has not completed:

1. It first checks whether the task is already done. If so, return.
2. If the task is still on **its own deque**, it pops it and runs it directly. No
   blocking at all. In the common case of a well-written `compute()`, this is what
   happens.
3. If the task was **stolen by someone else**, the joining worker tries to help: it
   looks for the thief and runs tasks from the thief's deque, working down the chain of
   "who stole from whom" so it makes progress on the very subtree it is waiting for.
4. Only if all of that fails does it compensate — possibly adding a spare thread — and
   actually block.

The consequence: **an FJP worker waiting on a `join()` is still doing useful work.** It
does not sit idle. That property is what makes deep recursive splitting affordable in
the first place; in a conventional pool, a worker blocked on a subtask's result is a
worker removed from the pool, and a recursive algorithm would deadlock itself
immediately at any depth greater than the pool size.

**And here is the boundary.** The worker can help with `ForkJoinTask`s. It cannot help
with a socket read. It cannot help with `DriverManager.getConnection`. It cannot help
with `Thread.sleep`. Those park the OS thread in the kernel, and from the pool's point of
view the worker has simply stopped responding: it is not running a task it can be
credited for, it is not scanning for victims, and — critically — **the pool cannot tell
the difference between "this worker is deep in a long CPU-bound computation" and "this
worker is blocked in a syscall."** So it does not compensate.

### `ManagedBlocker` — the sanctioned escape hatch

For the cases that genuinely must block, `ForkJoinPool` offers a way for the blocking
code to *tell the pool what it is about to do*:

```java
public static interface ManagedBlocker {
    boolean block() throws InterruptedException;   // do the blocking; true if no more needed
    boolean isReleasable();                         // true if blocking is unnecessary now
}
```

You call `ForkJoinPool.managedBlock(blocker)`. If the calling thread is an FJP worker,
the pool may start or unpark a **spare** worker first, so that target parallelism is
maintained while this one is parked. When the block returns, the spare is retired or
parked.

Two facts about it that people get wrong:

- **Compensation is bounded.** The common pool has a maximum spare count (the
  `java.util.concurrent.ForkJoinPool.common.maximumSpares` system property, with a
  default in the low hundreds). It is a safety valve, not a licence to block freely.
  Fifty concurrent managed blocks will exhaust it and you are back where you started,
  with a much more confusing thread dump.
- **The JDK uses it internally.** `CompletableFuture`'s blocking `join()`/`get()` paths
  use this mechanism when called on an FJP worker, which is why joining a future inside
  a parallel stream is less immediately catastrophic than a raw JDBC call. That is a
  reason not to panic, not a reason to do it.

`ManagedBlocker` is the right tool when a CPU-bound computation must occasionally wait
on a bounded resource — a `Semaphore` permit, a bounded queue slot. It is the wrong tool
for "I want to do I/O in a parallel stream." The right tool for that is a different
executor, or — from next topic — virtual threads.

### The common pool: sizing, ownership, and container reality

```java
ForkJoinPool.commonPool()
```

- **Target parallelism is `availableProcessors() - 1`.** The reasoning: the submitting
  thread itself participates in the computation, so the pool provides the other N-1.
- **On a machine where that leaves nothing, submitted tasks run in the calling thread.**
  Your "parallel" code becomes sequential, silently, with no warning and no error. Print
  `ForkJoinPool.commonPool().getParallelism()` and check for yourself rather than trusting
  arithmetic — including mine.
- **Its threads are daemon threads.** They do not keep the JVM alive, and you cannot
  shut the pool down. `commonPool().shutdown()` is a documented no-op.
- **It is JVM-global.** Your code, the JDK's stream implementation, and any library on
  your classpath all share it.
- **It is configurable only at startup**, via system properties:

```bash
-Djava.util.concurrent.ForkJoinPool.common.parallelism=8
-Djava.util.concurrent.ForkJoinPool.common.maximumSpares=256
-Djava.util.concurrent.ForkJoinPool.common.threadFactory=...
-Djava.util.concurrent.ForkJoinPool.common.exceptionHandler=...
```

**Container reality (Topic 82).** `availableProcessors()` respects the cgroup CPU quota
on a modern JDK. So:

| Container setting | Likely `availableProcessors()` | Consequence for the common pool |
|---|---|---|
| `--cpus=8` | 8 | parallelism 7. Fine. |
| `--cpus=2` | 2 | parallelism 1. One worker. Your "parallel" stream has one helper. |
| `--cpus=0.5` | 1 | parallelism 0. Tasks run in the caller thread. Fully sequential. |

That last row is the one that ships. A service that was tuned on a developer laptop with
10 cores and deployed with a conservative CPU request behaves completely differently, and
nothing anywhere says so. **Print the parallelism at startup and log it.** It is one line
and it has saved more debugging time than most monitoring.

### The virtual-thread scheduler is a different ForkJoinPool

Hold this clearly, because Topic 101 depends on it:

| | Common pool | Virtual-thread scheduler |
|---|---|---|
| Instance | `ForkJoinPool.commonPool()` | A separate, dedicated `ForkJoinPool` |
| Mode | LIFO local (async mode off) | **FIFO** (async mode on) |
| Default parallelism | `availableProcessors() - 1` | `availableProcessors()` |
| Used by | parallel streams, default `CompletableFuture` async | mounting virtual threads onto carriers |
| Tunable via | `...ForkJoinPool.common.parallelism` | `jdk.virtualThreadScheduler.parallelism`, `jdk.virtualThreadScheduler.maxPoolSize` |

Why FIFO for virtual threads? Because there is no recursive-split structure to exploit.
Virtual threads are independent units of work with no parent-child data locality, and
FIFO gives fairer latency: a virtual thread that became runnable first gets mounted
first. LIFO would give better cache behaviour and much worse tail latency, which is the
wrong trade for a request-serving workload.

> Verify the property names against your JDK's documentation before relying on them; the
> `jdk.virtualThreadScheduler.*` properties are documented as tuning knobs rather than
> stable API, and I would not build automation on them without checking.

---

## Concurrency trace

Before any correct code: here is the failure, step by step, on two workers.

### The scenario

`orderflow` has two things running in the same JVM:

- **The nightly revenue rollup.** A `RecursiveTask` over 5M in-memory order lines,
  splitting until each chunk is 50,000 lines, summing revenue per product. Pure CPU.
  Submitted to the common pool.
- **The order-detail endpoint's fan-out** (Topic 91). Someone "just added" a
  `CompletableFuture.supplyAsync(() -> paymentClient.status(orderId))` — with **no
  executor argument**, so it goes to the common pool — and the payment gateway is
  currently taking 800 ms per call.

The container is `--cpus=4`. So `availableProcessors()` is 4, and the common pool has
target parallelism **3**. Three workers.

### The interleaving

| Step | Worker-1 (common pool) | Worker-2 (common pool) |
|---|---|---|
| 1 | Owns a deque holding `[rollup-chunk-A, rollup-chunk-B, rollup-chunk-C, rollup-chunk-D]`. Pops D (LIFO) and starts summing. | Deque empty. Scans for a victim. |
| 2 | Still computing D. | Finds Worker-1. Steals from the **base**: takes `rollup-chunk-A`, the largest remaining subtree. CAS on `base` succeeds. |
| 3 | Finishes D, pops C, continues. Cache is warm; no contention with Worker-2. | Splitting A into sub-chunks, pushing them onto its own deque. Working productively. |
| 4 | **A request thread submits `paymentClient.status(...)` to the common pool's submission queue.** | — |
| 5 | Finishes C. Before popping B, scans and picks up the submission — external submissions are taken FIFO. | Still working on A's subtree. |
| 6 | Enters `paymentClient.status(...)`. This is an HTTP call. It reaches `socketRead0`, a **native method**, and the OS thread parks in the kernel. | Still working. |
| 7 | **Parked.** It is not running a task. It is not scanning for victims. It cannot steal. It cannot help. It will be there for 800 ms. | Finishes A's subtree. Deque empty. Scans for a victim. |
| 8 | Still parked. Its deque still holds `rollup-chunk-B`. | Finds Worker-1's deque, steals B. Continues. |
| 9 | Still parked. The pool's `getActiveThreadCount()` still counts it — **the pool has no way to know it is blocked in a syscall rather than deep in a long computation.** No spare is started. No compensation happens. | Finishes B. Deque empty. Nothing left to steal. Parks as an idle worker. |
| 10 | Two more payment-status submissions arrive and are picked up by Worker-2 and Worker-3 as they go idle. Both enter `socketRead0`. | — |
| 11 | **All three workers are parked in the kernel on socket reads.** Effective parallelism: zero. | — |
| 12 | A different part of the service starts a parallel stream over the product catalogue. It submits to the common pool. | Nothing takes it. It sits in a submission queue. |
| 13 | The machine's CPU utilisation is roughly 5%. Four cores, nothing running. | — |

### Outcome, in business terms

Three concurrent payment-status lookups — an entirely ordinary load level — have
stopped **every** parallel computation in the JVM.

The nightly revenue rollup, which has nothing to do with payments and touches no
network, is frozen mid-way. The catalogue endpoint that uses a parallel stream to build
its response is queued and not running. Against the Topic 65 baseline, the catalogue
read p99 goes from its recorded value to a timeout, and the error rate climbs — for an
endpoint whose code path did not change and does not touch the payment gateway at all.

**And the CPU is idle.** That is the diagnostic signature and the thing that makes this
bug so confusing on a dashboard: throughput has collapsed, latency has exploded, and
every resource-utilisation graph says the service is doing nothing. The instinct is to
look for a lock or a database problem. There is neither. There are three threads sitting
in `socketRead0` and a global pool that no longer has anybody in it.

An on-call engineer who has read this trace runs one command:

```bash
jcmd <pid> Thread.print | grep -A5 'ForkJoinPool.commonPool-worker'
```

and sees three workers in `socketRead0`. Diagnosis complete, in about ninety seconds.

---

## Example 1 — minimal

The smallest correct `RecursiveTask`: sum the line totals of an order batch.

```java
package com.orderflow.lab;

import java.util.concurrent.RecursiveTask;

/** Sums minor-unit line totals over a slice of an array. CPU-bound, no I/O. */
public class LineTotalSum extends RecursiveTask<Long> {

    /** Below this many elements, computing directly beats splitting. */
    private static final int SEQUENTIAL_THRESHOLD = 10_000;

    private final long[] lineTotalsMinor;
    private final int from;   // inclusive
    private final int to;     // exclusive

    public LineTotalSum(long[] lineTotalsMinor, int from, int to) {
        this.lineTotalsMinor = lineTotalsMinor;
        this.from = from;
        this.to = to;
    }

    @Override
    protected Long compute() {
        int length = to - from;

        if (length <= SEQUENTIAL_THRESHOLD) {
            long sum = 0;
            for (int i = from; i < to; i++) {
                sum += lineTotalsMinor[i];
            }
            return sum;
        }

        int mid = from + (length / 2);

        LineTotalSum left  = new LineTotalSum(lineTotalsMinor, from, mid);
        LineTotalSum right = new LineTotalSum(lineTotalsMinor, mid, to);

        left.fork();                     // push LEFT onto MY deque; returns immediately
        long rightSum = right.compute(); // compute RIGHT on THIS thread, right now
        long leftSum  = left.join();     // pop LEFT back off my deque (usually), or help

        return leftSum + rightSum;
    }
}
```

Running it:

```java
long[] lineTotals = loadLineTotals();          // e.g. 5_000_000 entries

// Do NOT use the common pool for anything you care about. Own your pool.
ForkJoinPool pool = new ForkJoinPool(Runtime.getRuntime().availableProcessors());
try {
    long total = pool.invoke(new LineTotalSum(lineTotals, 0, lineTotals.length));
    System.out.println("total minor units = " + total);
} finally {
    pool.shutdown();
}
```

### The three lines that matter

```java
left.fork();
long rightSum = right.compute();
long leftSum  = left.join();
```

This specific ordering is the idiom, and it is not arbitrary.

- `left.fork()` pushes `left` onto **this worker's own deque**, at `top`. Cheap: a store
  and a fence, no CAS.
- `right.compute()` runs on **this thread, immediately**. No task object is submitted;
  there is no scheduling decision, no queue traffic. The current thread keeps working on
  data that is already in its cache.
- `left.join()` then, in the overwhelmingly common case, **pops `left` straight back off
  this worker's own deque and runs it here** — because nobody stole it. Total cost of the
  "parallel" split when the pool is busy elsewhere: two pushes and two pops, all local,
  all uncontended.

If another worker *did* steal `left`, `join()` helps rather than idles, as described
above. Either way this thread never sits doing nothing.

`ForkJoinTask.invokeAll(left, right)` does exactly this pattern for you and is the more
idiomatic form for a two-way split:

```java
    invokeAll(left, right);
    return left.join() + right.join();
```

### The honest caveat about the threshold

`SEQUENTIAL_THRESHOLD = 10_000` is a guess. It is *always* a guess until you measure it.

Below the threshold, the cost of creating a `ForkJoinTask` object, pushing it, popping
it, and CASing its completion status exceeds the cost of the arithmetic it wraps. Split
too finely and the parallel version is **slower than the sequential one** — often
dramatically, because you have added millions of allocations to a loop that was
previously doing nothing but adds.

Finding the right threshold is a JMH job (Topic 77). Do not time it with a
`System.nanoTime()` loop; you will measure JIT warm-up and get an answer that is wrong by
a factor you cannot estimate.

---

## Example 2 — production scenario (on the project spine)

Two workloads from `orderflow`, one where FJP is right and one where it is
catastrophically wrong. The point of putting them side by side is that the code looks
similar and the outcomes could not be more different.

### The workload that belongs on a ForkJoinPool

The nightly revenue-by-product rollup. 5M order lines, already loaded into memory in
chunks by the batch job. For each line: look up a per-product margin from a pre-built
array, multiply, accumulate into a per-product total. No I/O in the inner loop. Pure
arithmetic over arrays.

```java
package com.orderflow.reporting;

import java.util.concurrent.ForkJoinPool;
import java.util.concurrent.RecursiveTask;

/**
 * Revenue rollup by product over a chunk of order lines.
 * CPU-bound, splittable, non-blocking. This is what ForkJoinPool is for.
 */
public final class RevenueRollupTask extends RecursiveTask<long[]> {

    private static final int SEQUENTIAL_THRESHOLD = 50_000;   // measured, see below

    private final int[]  productIds;        // parallel arrays: index i is one order line
    private final long[] lineTotalsMinor;
    private final int[]  marginBasisPoints; // indexed by productId
    private final int    productCount;
    private final int    from;
    private final int    to;

    public RevenueRollupTask(int[] productIds,
                             long[] lineTotalsMinor,
                             int[] marginBasisPoints,
                             int productCount,
                             int from,
                             int to) {
        this.productIds        = productIds;
        this.lineTotalsMinor   = lineTotalsMinor;
        this.marginBasisPoints = marginBasisPoints;
        this.productCount      = productCount;
        this.from              = from;
        this.to                = to;
    }

    @Override
    protected long[] compute() {
        int length = to - from;

        if (length <= SEQUENTIAL_THRESHOLD) {
            long[] revenueByProduct = new long[productCount];
            for (int i = from; i < to; i++) {
                int productId = productIds[i];
                revenueByProduct[productId] +=
                        lineTotalsMinor[i] * marginBasisPoints[productId] / 10_000L;
            }
            return revenueByProduct;
        }

        int mid = from + (length / 2);

        RevenueRollupTask left = new RevenueRollupTask(
                productIds, lineTotalsMinor, marginBasisPoints, productCount, from, mid);
        RevenueRollupTask right = new RevenueRollupTask(
                productIds, lineTotalsMinor, marginBasisPoints, productCount, mid, to);

        left.fork();
        long[] rightTotals = right.compute();
        long[] leftTotals  = left.join();

        // Combine. Associative, no shared mutable state, no lock.
        for (int p = 0; p < productCount; p++) {
            leftTotals[p] += rightTotals[p];
        }
        return leftTotals;
    }
}
```

And the caller, with the two decisions that matter:

```java
package com.orderflow.reporting;

import org.springframework.stereotype.Component;
import jakarta.annotation.PreDestroy;
import java.util.concurrent.ForkJoinPool;

@Component
public class RevenueRollupRunner {

    /**
     * DECISION 1: a DEDICATED pool, not commonPool().
     * The rollup is a long, CPU-saturating job. Running it on the common pool would
     * mean every parallel stream and every default-executor CompletableFuture in the
     * JVM queues behind an hour of batch work.
     *
     * DECISION 2: parallelism sized from the CONTAINER, and logged.
     */
    private final ForkJoinPool rollupPool;

    public RevenueRollupRunner() {
        int cores = Runtime.getRuntime().availableProcessors();
        // Leave one core for request-serving threads. On a 1-core container this is 1,
        // not 0 -- a ForkJoinPool constructed with 0 parallelism is not what you want.
        int parallelism = Math.max(1, cores - 1);
        this.rollupPool = new ForkJoinPool(parallelism);

        System.out.printf("rollupPool parallelism=%d (availableProcessors=%d, "
                        + "commonPool parallelism=%d)%n",
                parallelism, cores, ForkJoinPool.commonPool().getParallelism());
    }

    public long[] run(OrderLineChunk chunk, int[] marginBasisPoints, int productCount) {
        return rollupPool.invoke(new RevenueRollupTask(
                chunk.productIds(), chunk.lineTotalsMinor(),
                marginBasisPoints, productCount,
                0, chunk.size()));
    }

    @PreDestroy
    public void shutdown() {
        rollupPool.shutdown();
    }
}
```

Why this one is correct, checked against the four preconditions:

| Precondition | Met? |
|---|---|
| CPU-bound, no blocking in `compute()` | Yes. Array reads and arithmetic only. |
| Recursively splittable with cheap splits | Yes. Index arithmetic; no copying. |
| Associative combine, no shared mutable state | Yes. Each task returns its own array; the parent adds them. |
| Enough total work to amortise task overhead | Yes at 5M lines with a 50k threshold — about 100 leaf tasks. Verify with JMH. |

One allocation note worth flagging in review: each leaf allocates a `long[productCount]`.
At 100k products that is 800 KB per leaf task, and with ~100 leaves you have allocated
80 MB of short-lived arrays. That is a Topic 68 conversation, not a Topic 100 one, but it
is exactly the kind of thing a reviewer should catch — and the fix (accumulate into a
pre-allocated per-worker array, or roll up into a `ConcurrentHashMap` with `merge`) is a
real design choice with real trade-offs.

### The workload that must NOT go on a ForkJoinPool

The order-detail fan-out. This is Topic 91's endpoint: `GET /orders/{id}` needs the
order, its product details, its inventory status, and its payment status. Three of those
are network calls.

**The version that ships and takes the service down:**

```java
// WRONG. Every supplyAsync with no executor goes to ForkJoinPool.commonPool().
public OrderDetail loadWrong(long orderId) {
    var product   = CompletableFuture.supplyAsync(() -> productClient.byOrder(orderId));
    var inventory = CompletableFuture.supplyAsync(() -> inventoryClient.byOrder(orderId));
    var payment   = CompletableFuture.supplyAsync(() -> paymentClient.status(orderId));

    return CompletableFuture.allOf(product, inventory, payment)
            .thenApply(v -> new OrderDetail(
                    product.join(), inventory.join(), payment.join()))
            .join();
}
```

Three blocking HTTP calls, three common-pool workers parked in `socketRead0`, on a pool
with `cores - 1` workers. On a 4-core container: **one request saturates the entire
JVM's parallel-computation capacity.** This is the trace above, and it is why the
Topic 91 drill produced the collapse it did.

**The correct version, pre-Loom:**

```java
package com.orderflow.orders;

import org.springframework.stereotype.Service;
import java.util.concurrent.*;

@Service
public class OrderDetailService {

    /**
     * A DEDICATED, BOUNDED pool for blocking downstream calls.
     * Sizing is Little's Law from Topic 90, not a round number:
     *   in-flight = target throughput x latency
     *   200 rps x 0.08 s ~= 16, so 24 gives headroom.
     * The queue is bounded, so overload is rejected rather than absorbed.
     */
    private final ExecutorService downstreamPool = new ThreadPoolExecutor(
            24, 24,
            60, TimeUnit.SECONDS,
            new ArrayBlockingQueue<>(200),
            new ThreadPoolExecutor.CallerRunsPolicy());

    public OrderDetail load(long orderId) {
        var product   = CompletableFuture.supplyAsync(
                () -> productClient.byOrder(orderId), downstreamPool);
        var inventory = CompletableFuture.supplyAsync(
                () -> inventoryClient.byOrder(orderId), downstreamPool);
        var payment   = CompletableFuture.supplyAsync(
                () -> paymentClient.status(orderId), downstreamPool);

        return CompletableFuture.allOf(product, inventory, payment)
                .thenApply(v -> new OrderDetail(
                        product.join(), inventory.join(), payment.join()))
                .join();
    }
}
```

Three changes, each fixing a distinct hazard:

1. **An explicit executor on every `supplyAsync`.** The common pool is now untouched by
   this path. The rule to internalise: *if a `CompletableFuture` method name ends in
   `Async` and you did not pass an executor, you submitted to the common pool.*
2. **A bounded queue with a rejection policy.** Overload becomes a visible error rather
   than an invisible latency climb (Topic 90).
3. **A size derived from Little's Law**, not from a hunch.

> **Two forward pointers, both important.**
>
> Topic 101 replaces this whole pool with virtual threads, and the blocking style becomes
> the *right* style rather than a compromise. But note what virtual threads do **not**
> fix here: `allOf` still has no cancellation semantics, so if `paymentClient` fails, the
> product and inventory calls keep running and their results are discarded. That is a
> leaked in-flight request against a downstream you are already stressing. Topic 102 fixes
> exactly that with a structured scope.

### Where `ManagedBlocker` genuinely belongs

One legitimate case in `orderflow`: the rollup must occasionally consult a bounded,
rate-limited pricing service for products whose margin is not in the pre-built array. It
is CPU-bound work that occasionally waits on a `Semaphore` permit.

```java
package com.orderflow.reporting;

import java.util.concurrent.ForkJoinPool;
import java.util.concurrent.Semaphore;

/** Acquire a rate-limit permit without silently removing a worker from the pool. */
final class PermitBlocker implements ForkJoinPool.ManagedBlocker {

    private final Semaphore permits;
    private boolean acquired;

    PermitBlocker(Semaphore permits) { this.permits = permits; }

    @Override
    public boolean block() throws InterruptedException {
        if (!acquired) {
            permits.acquire();     // this is where we park
            acquired = true;
        }
        return true;               // no further blocking needed
    }

    @Override
    public boolean isReleasable() {
        return acquired || (acquired = permits.tryAcquire());
    }
}

// Use:
PermitBlocker blocker = new PermitBlocker(pricingPermits);
ForkJoinPool.managedBlock(blocker);   // pool may start a spare to keep parallelism
try {
    margin = pricingClient.marginFor(productId);
} finally {
    pricingPermits.release();
}
```

**And the honest assessment:** this is correct and it is still a smell. If the rollup
needs to make network calls, the right design is to fetch every missing margin *before*
the parallel phase starts, so the parallel phase stays pure. `ManagedBlocker` is what you
reach for when you cannot restructure — a bounded wait inside an otherwise CPU-bound
computation — not a general permission slip to do I/O in a `ForkJoinPool`.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — blocking inside the common ForkJoinPool

**Wrong:**

```java
CompletableFuture.supplyAsync(() -> paymentClient.status(orderId))   // no executor
// or
orderIds.parallelStream().map(id -> jdbc.queryForObject(...))        // JDBC in a stream
```

**Exact symptom:** the sharpest signature in this whole document.

- Throughput on **unrelated** endpoints collapses.
- p99 on those endpoints goes from its Topic 65 baseline to timeout.
- **CPU utilisation is near zero.** Every dashboard says the service is idle.
- `jcmd <pid> Thread.print` shows workers named `ForkJoinPool.commonPool-worker-N`
  parked in `socketRead0`, `Unsafe.park`, or a JDBC driver frame.
- The number of such workers equals `availableProcessors() - 1`, exactly.
- Restarting the service fixes it for a few minutes.

That combination — collapsed throughput plus idle CPU plus exactly `cores - 1` blocked
workers with the same name prefix — is diagnostic. Nothing else looks like it.

**Root cause:** a parked worker cannot steal, cannot help, and cannot be compensated for,
because the pool cannot distinguish "blocked in a syscall" from "deep in a long
computation". The common pool is JVM-global and sized to `cores - 1`, so a small number
of blocking tasks removes all parallel-computation capacity from the entire process.

**Fix:**

1. **Never submit blocking work to the common pool.** Practically: never call an `*Async`
   `CompletableFuture` method without an executor argument. Enforce it — ErrorProne and
   ArchUnit can both fail the build on the no-executor overloads, and that is a
   five-minute change worth making today.
2. **Use a dedicated, bounded executor for blocking work**, sized by Little's Law
   (Topic 90).
3. **From Topic 101**, use virtual threads for the blocking calls — which removes the
   sizing question rather than answering it.
4. If a bounded wait genuinely must happen inside a CPU-bound FJP task, wrap it in a
   `ManagedBlocker` and understand that compensation is capped.

---

### Trap 2 — assuming `.parallel()` gives you a private pool

**Wrong:**

```java
// Two unrelated jobs, in the same JVM, both "parallel".
List<ProductRevenue> revenue  = lines.parallelStream()....collect(...);
List<StockAlert>     restocks = skus.parallelStream()....collect(...);
```

**Exact symptom:** the latencies of two entirely unrelated jobs correlate perfectly on a
dashboard. Whenever the batch job runs, the API's parallel-stream-backed endpoint slows
down by roughly the same factor, and vice versa. Neither team's code changed. Adding
threads to your own executors does nothing, because neither job is using your executors.

There is a nastier variant: a **library** you depend on uses a parallel stream
internally. Your batch job saturates the common pool and a library call that has always
taken 2 ms starts taking 400 ms, with no line of your code in the stack trace to explain
it.

**Root cause:** `parallelStream()` and `stream().parallel()` both execute on
`ForkJoinPool.commonPool()`. There is one of those per JVM. It is shared with the JDK and
with every library on your classpath, and it is sized to `cores - 1` regardless of how
many independent jobs want it.

**Fix:** submit the stream from inside your own pool.

```java
ForkJoinPool rollupPool = new ForkJoinPool(Math.max(1, cores - 1));
List<ProductRevenue> revenue = rollupPool.submit(
        () -> lines.parallelStream().map(...).collect(...)
).join();
```

**And here is the honest caveat**, which you should carry into any interview where you
give this answer: this works because the stream implementation executes on the
`ForkJoinPool` that is current for the calling thread, and a task submitted to
`rollupPool` runs on one of its workers. That behaviour is **widely relied upon and is
not a specified guarantee of the Streams API.** It has been stable for many years and
is the standard workaround, but it is an implementation detail, not a contract. If you
need a guarantee, do not use parallel streams — write the `RecursiveTask` explicitly and
submit it to a pool you own, as in Example 2.

---

### Trap 3 — the container makes your parallel code sequential, silently

**Wrong:** nothing in the code. This trap is purely operational.

```yaml
resources:
  requests: { cpu: "500m" }
  limits:   { cpu: "500m" }
```

**Exact symptom:** a batch job that took 4 minutes on a developer laptop takes 40 minutes
in the cluster. Profiling shows one thread doing all the work. There is no error, no
warning, no log line. The code is identical. Scaling the deployment to more replicas does
not help, because each replica is equally crippled. Someone eventually says "parallel
streams don't work in Kubernetes", which is wrong but is a rational conclusion from the
evidence available.

**Root cause:** `availableProcessors()` respects the cgroup CPU quota on a modern JDK. At
`cpu: 500m`, it reports 1. The common pool's target parallelism is `availableProcessors()
- 1`, which leaves nothing, and the JDK falls back to running submitted tasks in the
calling thread. Your parallel stream is a sequential stream with extra allocation.

**Fix:**

1. **Log it at startup.** One line, and it turns a week of confusion into a glance:

```java
@PostConstruct
void logPoolSizing() {
    log.info("availableProcessors={} commonPoolParallelism={} rollupPoolParallelism={}",
             Runtime.getRuntime().availableProcessors(),
             ForkJoinPool.commonPool().getParallelism(),
             rollupPool.getParallelism());
}
```

2. **Give CPU-bound workloads a CPU request that reflects the parallelism they need.**
   Fractional CPU limits and parallel computation are in direct conflict; pick one.
3. **Size your own pools explicitly with a floor**, as in Example 2's
   `Math.max(1, cores - 1)`. Never construct a `ForkJoinPool` with a computed parallelism
   that can reach zero.
4. Cross-reference Topic 82: this is the same container-awareness story that affects
   heap sizing and collector selection, and it is worth auditing all three at once.

---

### Trap 4 — forking both halves and joining in the wrong order

**Wrong:**

```java
    left.fork();
    right.fork();
    return left.join() + right.join();   // joins LEFT first
```

**Exact symptom:** the parallel version scales far worse than expected — say 1.6x on 8
cores instead of 6x. `getStealCount()` on the pool is enormous relative to the number of
leaf tasks. A profiler shows significant time in FJP's scanning and helping paths rather
than in your `compute()` body. It is not *broken*; it is just much slower than the same
algorithm written the idiomatic way, which makes it very easy to conclude "parallelism
doesn't help here" and revert.

**Root cause:** two problems stacked.

First, `left.fork(); right.fork();` pushes **both** tasks onto the deque and leaves the
current thread with nothing to do but immediately go looking for work again — including
possibly stealing back what it just pushed, through a much more expensive path than a
local pop.

Second, and worse: `left` was pushed **first**, so it sits *below* `right` on the deque.
`left.join()` therefore cannot pop it — `pop` only takes from `top`, and `right` is on
top. So the joining worker has to take the slow path: search, help, or wait. Meanwhile
`right` sits on top of the deque available for anyone to pop, which is fine, but you have
inverted the cheap operation and the expensive one.

**Fix:** fork one, compute the other on the current thread, join the forked one.

```java
    left.fork();
    R rightResult = right.compute();
    R leftResult  = left.join();       // pops from the top; usually free
```

Or use the library idiom, which does this for you and is what you should write:

```java
    invokeAll(left, right);
    return left.join() + right.join();
```

If you have already written `fork(); fork();` for some reason, join in **reverse** order
— `right.join()` then `left.join()` — so the join order matches the LIFO deque order.
But the fork-one-compute-one form is better, and it is what every JDK example uses.

---

### Trap 5 — splitting too finely

**Wrong:**

```java
    if (to - from <= 1) {                     // split all the way to single elements
        return lineTotalsMinor[from];
    }
```

**Exact symptom:** the parallel version is *slower* than a plain `for` loop, often by
3x or more. An allocation profile (Topic 75) shows millions of `RecursiveTask` instances
dominating the allocation rate. GC logs show a sharp rise in young collections
(Topic 68). CPU is at 100% across all cores, which makes it look like the parallelism is
working, but throughput is worse than one thread.

**Root cause:** each `ForkJoinTask` is a heap object with a status field that is updated
by CAS on completion. Push, pop and completion each cost tens of nanoseconds. If the leaf
work is a single `long` addition — about a nanosecond — you have added roughly a hundred
times the overhead of the work you are trying to parallelise.

**Fix:** a sequential threshold, chosen so the leaf does enough work to dwarf the task
overhead. A useful starting heuristic is "each leaf should run for at least tens of
microseconds", which for simple arithmetic means tens of thousands of elements.

Then **measure it**, with JMH (Topic 77), across a few threshold values. Do not time it
with a `System.nanoTime()` loop; you will measure JIT warm-up and dead-code elimination
and get a number that is confidently wrong.

And apply the prior question first: **is there enough total work to justify parallelism
at all?** Below roughly 10,000 elements of cheap per-element work, a sequential loop
usually wins outright, and the correct fix is to delete the `ForkJoinTask`.

---

## Hands-on proof

Everything below is a command **you** run. I have no JVM; I will not print output and
claim it is real. What follows is what to run, what to look for, and what each possible
result means.

### Setup

```bash
mkdir -p ~/java-lab/100 && cd ~/java-lab/100
java --version
uname -m
sysctl -n hw.ncpu 2>/dev/null || nproc
```

### Proof 1 — what parallelism do you actually have?

`PoolFacts.java`:

```java
import java.util.concurrent.ForkJoinPool;

public class PoolFacts {
    public static void main(String[] args) {
        ForkJoinPool common = ForkJoinPool.commonPool();
        System.out.println("availableProcessors  = "
                + Runtime.getRuntime().availableProcessors());
        System.out.println("common parallelism   = " + common.getParallelism());
        System.out.println("common poolSize      = " + common.getPoolSize());
        System.out.println("common toString      = " + common);
    }
}
```

```bash
java PoolFacts.java
java -Djava.util.concurrent.ForkJoinPool.common.parallelism=1 PoolFacts.java
```

**What to look for:** the relationship between `availableProcessors` and
`common parallelism`, and whether the system property actually took effect.

| What you see | What it means |
|---|---|
| `parallelism` is `availableProcessors - 1` | The documented default. Note the number; it is the ceiling on all parallel-stream work in this JVM. |
| `parallelism` is 1 while `availableProcessors` is 2 | Expected. You have one helper thread. Any "parallelism" claim about this JVM should be stated as "two threads including the caller". |
| The system property changed the value | Confirms the property name is right for your JDK, which is worth knowing before you put it in a Dockerfile. |
| `poolSize` is 0 | Also expected on a fresh JVM. FJP creates workers lazily on first submission. |

### Proof 2 — the same thing inside a container

This is the proof that matters operationally.

`Dockerfile`:

```dockerfile
FROM eclipse-temurin:21-jdk
COPY PoolFacts.java /app/PoolFacts.java
WORKDIR /app
CMD ["java", "PoolFacts.java"]
```

```bash
docker build -t poolfacts .
docker run --rm --cpus=8   poolfacts
docker run --rm --cpus=2   poolfacts
docker run --rm --cpus=1   poolfacts
docker run --rm --cpus=0.5 poolfacts
```

**What to look for:** the `availableProcessors` line at each CPU limit, and what happens
to `common parallelism` as it falls.

**How to read it:** at `--cpus=0.5`, if `availableProcessors` reports 1, then every
parallel stream in that container is running with no helper threads. Whatever it reports
on your JDK, **write the numbers down** — this is the table you show the platform team
when they propose fractional CPU limits for a service that does batch work.

### Proof 3 — watch stealing happen

`StealWatch.java`:

```java
import java.util.concurrent.*;

public class StealWatch {

    static class Chunk extends RecursiveTask<Long> {
        final long from, to, threshold;
        Chunk(long from, long to, long threshold) {
            this.from = from; this.to = to; this.threshold = threshold;
        }
        protected Long compute() {
            if (to - from <= threshold) {
                long acc = 0;
                for (long i = from; i < to; i++) acc += (i * 31) % 1_000_003L;
                return acc;
            }
            long mid = from + (to - from) / 2;
            Chunk left = new Chunk(from, mid, threshold);
            Chunk right = new Chunk(mid, to, threshold);
            left.fork();
            long r = right.compute();
            return left.join() + r;
        }
    }

    public static void main(String[] args) {
        int parallelism = Integer.parseInt(args[0]);
        long threshold  = Long.parseLong(args[1]);

        ForkJoinPool pool = new ForkJoinPool(parallelism);
        long result = pool.invoke(new Chunk(0, 200_000_000L, threshold));

        System.out.println("result       = " + result);
        System.out.println("parallelism  = " + pool.getParallelism());
        System.out.println("poolSize     = " + pool.getPoolSize());
        System.out.println("stealCount   = " + pool.getStealCount());
        pool.shutdown();
    }
}
```

```bash
java StealWatch.java 1 1000000       # one worker: nobody to steal from
java StealWatch.java 8 1000000       # eight workers, coarse tasks
java StealWatch.java 8 1000          # eight workers, very fine tasks
```

**What to look for:** `stealCount` across the three runs.

| What you see | What it means |
|---|---|
| `stealCount` near zero at parallelism 1 | Correct: with one worker there is nothing to steal from. This is your control. |
| `stealCount` is modest at parallelism 8 with a coarse threshold | The intended behaviour. A handful of steals distributed a lot of work, exactly as the FIFO-from-base design intends. |
| `stealCount` is enormous at the fine threshold | Task overhead is dominating. Compare wall-clock times: the fine-grained run will probably be *slower* despite the same total arithmetic. This is Trap 5, measured. |
| Times are close across all three | Your machine may be thermally throttling, or another process is competing. Re-run on a quiet machine before drawing conclusions. |

> These are wall-clock comparisons and therefore not trustworthy as performance numbers.
> They are fine for comparing `stealCount`, which is a counter, not a timing. For the
> timing question, use JMH (Topic 77).

### Proof 4 — the thread dump signature of a blocked worker

`BlockedWorker.java`:

```java
import java.util.concurrent.*;
import java.util.stream.IntStream;

public class BlockedWorker {
    public static void main(String[] args) throws Exception {
        System.out.println("pid = " + ProcessHandle.current().pid());
        System.out.println("common parallelism = "
                + ForkJoinPool.commonPool().getParallelism());

        // Occupy every common-pool worker with a long block.
        int workers = ForkJoinPool.commonPool().getParallelism();
        for (int i = 0; i < workers; i++) {
            CompletableFuture.runAsync(() -> {          // NO executor: common pool
                try { Thread.sleep(120_000); } catch (InterruptedException ignored) { }
            });
        }

        Thread.sleep(2_000);

        // Now try to do unrelated parallel work.
        long start = System.currentTimeMillis();
        long sum = IntStream.range(0, 20_000_000).parallel().asLongStream().sum();
        System.out.println("unrelated parallel sum took "
                + (System.currentTimeMillis() - start) + " ms, sum=" + sum);
    }
}
```

```bash
java BlockedWorker.java &
sleep 5
jcmd $(jcmd -l | grep BlockedWorker | cut -d' ' -f1) Thread.print \
  | grep -A6 'ForkJoinPool.commonPool-worker'
```

**What to look for:**

1. In the thread dump: how many `ForkJoinPool.commonPool-worker-N` threads there are, and
   what frame each is in.
2. Whether the "unrelated parallel sum" line prints promptly or is delayed.

**How to read it:**

| What you see | What it means |
|---|---|
| Every common-pool worker is in `Thread.sleep` / `Unsafe.park` | The trace from earlier in this document, reproduced. Those workers cannot steal. |
| The unrelated sum still finishes quickly | Expected, and instructive: the **calling thread** participates in a parallel stream, so with the pool fully blocked the stream degrades to roughly sequential rather than hanging. That is the "silently sequential" failure, not a deadlock. |
| The unrelated sum takes far longer than a sequential baseline | You have measured the degradation directly. Run it once without the blocking futures to get the baseline. |
| Fewer worker threads than `parallelism` | Workers are created lazily. Submit more work first. |

The lesson to write down: **the common-pool blocking failure usually presents as silent
sequential degradation, not as a hang.** That is why it is so hard to spot on a dashboard
and why the thread dump is the diagnostic, not the metrics.

### Proof 5 — the virtual-thread scheduler is a different pool

A one-line confirmation of the distinction that matters for Topic 101:

```java
public class SchedulerIdentity {
    public static void main(String[] args) throws Exception {
        System.out.println("common pool worker name pattern:");
        ForkJoinPool.commonPool().submit(
            () -> System.out.println("  " + Thread.currentThread())).get();

        System.out.println("virtual thread carrier:");
        Thread.ofVirtual().start(() ->
            System.out.println("  " + Thread.currentThread())).join();
    }
}
```

**What to look for:** the thread names and descriptions printed by each.

**How to read it:** the common-pool worker will name the common pool. The virtual thread
will describe itself as a virtual thread, not as a carrier — the carrier is deliberately
not exposed through `Thread.currentThread()`. That is not an omission; it is the design
decision that makes `Thread.currentThread()` identity stable across mounts, and it is the
first thing Topic 101 explains.

---

## Failure drill

**Mandatory.** Do not read the analysis until you have produced the failure yourself and
written down what you saw.

### The scenario

You are going to instrument the `orderflow` revenue rollup, run it correctly, and then
have a colleague "just add a quick lookup" that blocks — and watch the pool's own
counters tell you exactly what happened.

The point of this drill, as distinct from Topics 25 and 91, is that you will diagnose it
**from the pool's metrics alone**, before looking at a thread dump. Learning to read
`stealCount` / `activeThreadCount` / `queuedTaskCount` together is a skill you will use
on a real incident.

### Setup

`src/main/java/com/orderflow/lab/RollupDrill.java`:

```java
package com.orderflow.lab;

import java.util.concurrent.*;

public class RollupDrill {

    /** Set to true for the BROKEN run. */
    static volatile boolean blockInLeaf = false;

    /** Simulates the "quick" payment-gateway lookup someone added to the leaf. */
    static void slowLookup() {
        try { Thread.sleep(50); } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }

    static class Rollup extends RecursiveTask<Long> {
        final int from, to;
        Rollup(int from, int to) { this.from = from; this.to = to; }

        protected Long compute() {
            if (to - from <= 50_000) {
                if (blockInLeaf) slowLookup();          // <-- the defect
                long acc = 0;
                for (int i = from; i < to; i++) acc += ((long) i * 7919L) % 1_000_003L;
                return acc;
            }
            int mid = from + (to - from) / 2;
            Rollup left = new Rollup(from, mid);
            Rollup right = new Rollup(mid, to);
            left.fork();
            long r = right.compute();
            return left.join() + r;
        }
    }

    public static void main(String[] args) throws Exception {
        blockInLeaf = args.length > 0 && args[0].equals("blocking");

        int parallelism = Math.max(1, Runtime.getRuntime().availableProcessors() - 1);
        ForkJoinPool pool = new ForkJoinPool(parallelism);

        System.out.printf("pid=%d mode=%s parallelism=%d%n",
                ProcessHandle.current().pid(),
                blockInLeaf ? "BLOCKING" : "CLEAN",
                pool.getParallelism());

        // Sampler: prints the pool's own counters every 500 ms.
        Thread sampler = Thread.ofPlatform().daemon().start(() -> {
            while (true) {
                System.out.printf(
                    "t=%6d  poolSize=%2d  active=%2d  running=%2d  "
                  + "queuedTasks=%5d  queuedSubmissions=%3d  steals=%6d%n",
                    System.currentTimeMillis() % 1_000_000,
                    pool.getPoolSize(),
                    pool.getActiveThreadCount(),
                    pool.getRunningThreadCount(),
                    pool.getQueuedTaskCount(),
                    pool.getQueuedSubmissionCount(),
                    pool.getStealCount());
                try { Thread.sleep(500); } catch (InterruptedException e) { return; }
            }
        });

        long start = System.currentTimeMillis();
        long result = pool.invoke(new Rollup(0, 20_000_000));
        long elapsed = System.currentTimeMillis() - start;

        System.out.printf("RESULT=%d ELAPSED_MS=%d FINAL_STEALS=%d%n",
                result, elapsed, pool.getStealCount());
        pool.shutdown();
    }
}
```

### Commands

```bash
# Run A -- the control. CPU-bound leaves, no blocking.
java RollupDrill.java clean 2>&1 | tee run-a.log

# Run B -- the defect. Same computation, one Thread.sleep(50) per leaf.
java RollupDrill.java blocking 2>&1 | tee run-b.log

# While Run B is going, in another terminal:
jcmd <pid> Thread.print | grep -A6 'ForkJoinPool-1-worker'
```

### What to capture

Write down, for each run, before reading further:

1. `parallelism` and `poolSize` (final value).
2. `active` and `running` over time — do they diverge?
3. `queuedTasks` over time — does it climb, plateau, or stay near zero?
4. `steals` over time — does it keep increasing, or flatline?
5. `ELAPSED_MS`.
6. From the thread dump during Run B: how many workers, and in what frame.
7. CPU utilisation during each run, from Activity Monitor or `top`.

### How to read it

| What you see | What it means |
|---|---|
| Run A: `active` ≈ `running` ≈ `parallelism`, `steals` rising steadily, `queuedTasks` low, CPU near 100% | The healthy shape. Workers are busy, stealing is distributing work, nothing is backing up. This is your baseline for every future FJP incident. |
| Run B: `running` drops well below `active`, and stays there | **The drill has fired.** `getRunningThreadCount()` estimates workers *not blocked*; `getActiveThreadCount()` estimates workers not idle. The gap between them is your blocked workers. This is the single most useful number pair on the class. |
| Run B: `steals` flatlines while `queuedTasks` climbs | Work is being created and nobody is taking it. A parked worker cannot steal — the mechanical statement, observed as two counters. |
| Run B: CPU utilisation is low while elapsed time is high | The signature. Collapsed throughput with idle CPU means threads are parked, not contended. If CPU were high you would be looking at a lock or a livelock instead (Topic 98). |
| Run B thread dump: workers in `Thread.sleep` / `Unsafe.park` | Confirmation. In a real incident, substitute `socketRead0` or a JDBC driver frame. |
| Run B: `poolSize` grows above `parallelism` | Interesting and worth investigating. FJP can add spares in some circumstances; note the exact numbers and compare with the `ManagedBlocker` fix below, where growth is the *expected* behaviour. |
| Run A and Run B look identical | Check `blockInLeaf` is actually true (print it) and that the leaf threshold means you have more leaves than workers. With 20M elements and a 50k threshold you get ~400 leaves; that is plenty. |

### Now fix it — two ways, and only one is right

#### Fix 1 — `ManagedBlocker` (the escape hatch)

Replace `slowLookup()` with:

```java
    static void slowLookupManaged() throws InterruptedException {
        ForkJoinPool.managedBlock(new ForkJoinPool.ManagedBlocker() {
            boolean done;
            public boolean block() throws InterruptedException {
                if (!done) { Thread.sleep(50); done = true; }
                return true;
            }
            public boolean isReleasable() { return done; }
        });
    }
```

Re-run and compare. **What to look for:** does `poolSize` now grow above `parallelism`?
Does `ELAPSED_MS` improve? Does `running` stay closer to `parallelism`?

**How to read it:** if `poolSize` grows, you have watched compensation happen — the pool
started spares because you told it you were about to block. That is the mechanism working
as designed. Note also what it cost: you now have more OS threads than cores, all of them
sleeping, which is exactly the thread-per-blocking-operation model that Topic 90 taught
you to size carefully. `ManagedBlocker` did not make blocking free. It made it visible to
the pool.

#### Fix 2 — move the blocking work out entirely (the right answer)

Restructure so the leaf is pure:

```java
// BEFORE the parallel phase, on a dedicated bounded executor:
Map<Integer, Integer> margins = prefetchMargins(missingProductIds, downstreamPool);

// The parallel phase now reads only from arrays. No blocking anywhere in compute().
```

Re-run. **What to look for:** `running` back at `parallelism`, `steals` rising steadily,
`queuedTasks` low, CPU near 100%, and `ELAPSED_MS` close to Run A's.

### What the fix proves

Three things, in order of importance:

1. **The pool's own counters diagnose this without a thread dump.** `running` diverging
   from `active`, with `steals` flat and `queuedTasks` climbing, is a complete diagnosis.
   You can put those four numbers on a Micrometer gauge and alert on the divergence, which
   turns a mystery incident into a dashboard.

2. **`ManagedBlocker` is compensation, not a fix.** It kept the pool responsive by adding
   threads. It did not make the blocking cheap, and it is capped. Reaching for it should
   feel like reaching for a bounded resource, because it is one.

3. **The real fix is a workload boundary, not a tuning knob.** Blocking work and
   CPU-bound work belong on different executors. That statement survives the arrival of
   virtual threads in the next topic — virtual threads change what "the blocking executor"
   costs, not whether the boundary should exist.

---

## Measurement

### The standing rule

A naive `System.nanoTime()` loop is the **wrong** way to measure JVM performance. It
measures JIT warm-up, dead-code elimination of results you never use, on-stack
replacement, and whatever else the machine was doing. Topic 77 (JMH) is where you learn
to do this properly, and every threshold decision in this topic is a JMH job.

The wall-clock comparisons in the Hands-on and Failure Drill sections above are fine for
comparing **counters** (`stealCount` is a counter, not a timing) and for producing an
obvious order-of-magnitude difference. They are not fine for choosing a threshold or for
claiming a speedup. Do not put them in a document with the word "benchmark" in the title.

### The pool's own instrumentation

`ForkJoinPool` exposes more useful introspection than any other JDK executor. Learn all
six of these; together they are a complete picture.

| Method | What it tells you | What a bad value looks like |
|---|---|---|
| `getParallelism()` | Target parallelism. Fixed at construction. | Lower than you expect — check `availableProcessors()` and your container limits. |
| `getPoolSize()` | Current worker count, including spares. | Persistently above `parallelism` means compensation is happening — something is blocking. |
| `getActiveThreadCount()` | Estimated workers that are not idle. | — |
| `getRunningThreadCount()` | Estimated workers that are **not blocked**. | **`running` well below `active` is the blocked-worker signature.** This is the pair to alert on. |
| `getQueuedTaskCount()` | Tasks sitting in worker deques. | Climbing while `steals` is flat: work is being created faster than it is taken, because takers are parked. |
| `getQueuedSubmissionCount()` | External submissions not yet started. | Non-zero and climbing: external work is arriving faster than workers can pick it up. |
| `getStealCount()` | Cumulative steals since construction. | Flat during a busy period: nobody can steal. Enormous relative to leaf count: your threshold is too small (Trap 5). |

Wire them into Micrometer (Topic 118) and you have a dashboard that diagnoses every
failure in this document:

```java
@Bean
MeterBinder forkJoinPoolMetrics(ForkJoinPool rollupPool) {
    return registry -> {
        Gauge.builder("orderflow.fjp.parallelism", rollupPool,
                ForkJoinPool::getParallelism).register(registry);
        Gauge.builder("orderflow.fjp.pool.size", rollupPool,
                ForkJoinPool::getPoolSize).register(registry);
        Gauge.builder("orderflow.fjp.threads.active", rollupPool,
                ForkJoinPool::getActiveThreadCount).register(registry);
        Gauge.builder("orderflow.fjp.threads.running", rollupPool,
                ForkJoinPool::getRunningThreadCount).register(registry);
        Gauge.builder("orderflow.fjp.tasks.queued", rollupPool,
                ForkJoinPool::getQueuedTaskCount).register(registry);
        Gauge.builder("orderflow.fjp.submissions.queued", rollupPool,
                ForkJoinPool::getQueuedSubmissionCount).register(registry);
        FunctionCounter.builder("orderflow.fjp.steals", rollupPool,
                ForkJoinPool::getStealCount).register(registry);
    };
}
```

The alert that is worth having: **`threads.running` below 50% of `parallelism` for more
than 30 seconds while `tasks.queued` is above zero.** That fires on exactly the failure
in this document and on very little else.

### JFR

For an incident where you do not have the gauges, JFR (Topics 78, 81) gives you the same
picture from a recording:

```bash
java -XX:StartFlightRecording=duration=120s,filename=fjp.jfr,settings=profile -jar orderflow.jar

jfr summary fjp.jfr
jfr print --events jdk.ThreadPark      fjp.jfr | head -100
jfr print --events jdk.SocketRead      fjp.jfr | head -100
jfr print --events jdk.ExecutionSample fjp.jfr | head -100
```

**What to look for:**

- `jdk.ThreadPark` events whose thread name matches `ForkJoinPool*worker*`. Park duration
  and stack trace together tell you what the worker was waiting on.
- `jdk.SocketRead` events on FJP worker threads. **Any of these is a bug** — an FJP
  worker should never be doing socket I/O.
- `jdk.ExecutionSample` counts on FJP workers. If they are low while wall-clock time is
  high, the workers are parked and not sampled.

That third bullet is a good general habit: comparing execution-sample density against
wall-clock time is how you distinguish "slow because busy" from "slow because waiting",
and it is the CPU-vs-wall-clock distinction from Topic 78.

### Against the Topic 65 baseline

The rollup is a batch job and does not appear in your recorded p50/p95/p99. Its
interaction with them is the measurement that matters:

1. Record the baseline with the rollup **not running**. You already have this from
   Topic 65.
2. Run the k6 load with the rollup running on the **common pool**. Record p50/p95/p99 for
   the catalogue read and order read endpoints.
3. Run the same load with the rollup on a **dedicated pool** sized `cores - 1`. Record
   again.
4. Run it a third time with the dedicated pool sized to `cores / 2`, so the rollup takes
   longer but leaves more CPU for request serving.

**What to compare:** the p99 delta on the catalogue endpoint across runs 1 to 4, and the
rollup's own wall-clock duration.

**How to read it:** run 2 should show the worst request-latency degradation. Runs 3 and 4
trade rollup duration against request p99 — and that trade is a *product* decision, not a
technical one. Someone has to say whether the rollup finishing in 20 minutes instead of
12 is acceptable in exchange for the API keeping its latency SLO. Presenting that as a
table with real numbers, rather than as an opinion, is the senior move.

### Choosing the sequential threshold, properly

```java
@State(Scope.Benchmark)
public class RollupThresholdBenchmark {

    @Param({"1000", "10000", "50000", "200000", "1000000"})
    int threshold;

    long[] lineTotals;
    ForkJoinPool pool;

    @Setup public void setup() { /* build 5M-element arrays, construct pool */ }

    @Benchmark
    public long rollup() {
        return pool.invoke(new RevenueRollupTask(/* ... */ threshold /* ... */));
    }

    @Benchmark
    public long sequentialBaseline() { /* plain for loop over the same data */ }
}
```

**The baseline benchmark is not optional.** A parallel result without the sequential
number next to it is meaningless — you cannot tell a 6x speedup from a 0.6x slowdown
without it, and Trap 5 produces the latter routinely.

---

## Practice exercises

### 1 — easy

Build `StealWatch` from Proof 3 and answer three questions with numbers you produced.

**(a)** Run it at parallelism 1, 2, 4 and 8 with a fixed coarse threshold. Record
`stealCount` and elapsed time at each. Plot or tabulate them. Does `stealCount` grow with
parallelism, and why would you expect it to?

**(b)** Fix parallelism at 8 and sweep the threshold across 1,000 / 10,000 / 100,000 /
1,000,000. Record `stealCount` and elapsed time. Identify the threshold where elapsed time
stops improving, and the threshold where `stealCount` explodes.

**(c)** Rewrite `compute()` to use `left.fork(); right.fork(); left.join() + right.join()`
— Trap 4's wrong order. Re-run at parallelism 8 with your best threshold from (b). Report
the change in `stealCount` and elapsed time, and explain the change in terms of which end
of the deque `join()` can reach.

### 2 — medium (combines Topics 01–98)

Build the `orderflow` revenue rollup properly, applying things from across the curriculum.

**Part A.** Implement `RevenueRollupTask` from Example 2 over 5,000,000 synthetic order
lines across 100,000 products, with realistic skew (a few hot products holding a large
share of the lines).

**Part B.** Implement it three more ways and compare all four:

1. A plain sequential `for` loop.
2. `lines.parallelStream()` with a `Collectors.groupingBy` and a downstream summing
   collector (Topic 24). Note which pool it runs on.
3. The `RecursiveTask` on the **common** pool.
4. The `RecursiveTask` on a **dedicated** pool.

**Part C.** For each version, answer with evidence, not opinion:

- What is its **allocation rate**? Run each with `-Xlog:gc` and count young collections
  (Topic 68). Version 1 should allocate almost nothing; version 2 boxes every key
  (Topic 01) and allocates a `Map`; versions 3 and 4 allocate one array per leaf.
- Does version 2's `groupingBy` use a boxed `Integer` key? What would `EnumMap` or a
  plain `long[]` indexed by product id cost instead (Topics 16 and 01)?
- Which versions share the common pool, and what would happen if this job ran while the
  Topic 65 load test was in flight?

**Part D.** Now break version 4 deliberately: add a `Thread.sleep(20)` inside the leaf,
as in the drill. Predict what happens to `getRunningThreadCount()` and `getStealCount()`
**before** running it, then run it and compare your prediction to reality.

**Part E.** One paragraph: given your numbers, which version would you actually ship, and
what would have to change about the workload for your answer to flip?

### 3 — hard

The full production question, on the spine.

**Part A — establish the interaction.** With `orderflow` running under the Topic 65 k6
load, run the revenue rollup on the **common pool**. Record p50/p95/p99 and error rate for
the catalogue read, order read and order placement endpoints, and compare each against the
recorded baseline. Also record the rollup's wall-clock duration and the common pool's
counters throughout.

**Part B — isolate.** Move the rollup to a dedicated pool. Re-run the same load. Produce a
table with one row per endpoint and columns for baseline / common-pool / dedicated-pool
p99. State the size of the improvement in milliseconds and as a percentage of the SLO
budget, not just as a ratio.

**Part C — find the knee.** Sweep the dedicated pool's parallelism: `cores`, `cores - 1`,
`cores / 2`, `2`. For each, record both the rollup duration and the catalogue p99. Plot
one against the other. Identify the knee.

**Part D — the capacity argument.** Write the paragraph you would put in an RFC. It must
contain: the latency budget you are protecting, the rollup completion deadline the business
needs, the parallelism you chose, and the specific evidence for that choice. It must also
name what you would do if the two requirements became incompatible — more CPU, a longer
rollup window, an incremental rollup, or moving the job out of the request-serving JVM
entirely. Argue for one and give the strongest case against yourself.

**Part E — the container check.** Re-run Part B inside the container with the CPU limit
your production deployment actually uses. If `availableProcessors()` in that container
differs from your development machine, redo Part C's sweep inside the container. Report
whether your Part D conclusion survives contact with the real CPU limit. **This part is
where most people's analysis falls apart, which is why it is last.**

---

## Interview questions

### Q1 — "What's the difference between a ForkJoinPool and a fixed thread pool?"

**Mid-level answer:** "ForkJoinPool is designed for recursive tasks. It uses work
stealing so idle threads can take work from busy threads, which balances the load
better."

**Senior answer:** "The structural difference is the queue. `ThreadPoolExecutor` has one
shared queue, so every worker contends on the same lock for every task — fine when tasks
are chunky and independent. `ForkJoinPool` gives each worker its own deque. The owner
pushes and pops its own end LIFO, which is uncontended and cache-friendly, because the
subtask it just created operates on data still in L1. Idle workers steal from the *base*
of another worker's deque — the oldest task, which in a recursive split is the largest
remaining subtree, so one expensive steal transfers a lot of work. The other big
difference is `join()`: an FJP worker waiting on a subtask doesn't park, it helps — it
runs tasks from its own deque or from the thief's, so it makes progress on the thing it's
waiting for. That's what makes deep recursive splitting affordable; in a fixed pool a
worker blocked on a subtask is a worker removed from the pool, and the recursion would
deadlock itself at any depth beyond the pool size. All of it assumes CPU-bound,
non-blocking tasks — a worker parked in a syscall can't steal and can't help, and the pool
can't tell that's happened."

**What separates them:** the mid-level answer knows the feature list. The senior answer
names the deque, explains *both* directions with a distinct reason for each, knows about
helping, and closes with the workload assumption — which is the thing that actually
matters in production.

**Follow-up:** "So when would you choose a fixed pool?" For anything blocking, and for
independent chunky tasks where the work-stealing machinery is pure overhead.

---

### Q2 — "Why does a thief steal from the opposite end of the deque?"

**Mid-level answer:** "To avoid contending with the owner, since they're working on
different ends."

**Senior answer:** "That's one reason and it's real — the owner and the thief only
contend when the deque is nearly empty. But the bigger reason is *task size*. In a
divide-and-conquer split, the task at the base was pushed earliest, so it's highest in
the recursion tree, so it represents the largest remaining subtree. A steal is expensive
— a contended CAS plus a cache-line transfer plus operating on data the thief has never
touched — so you want to steal rarely and steal big. Taking from the base gets you a
large chunk you can then work on privately for a long time. Stealing from the top would
grab a leaf and you'd be back stealing microseconds later. The LIFO-local side is
optimising for cache locality; the FIFO-steal side is optimising for the amount of work
moved per steal. Two ends, two different objectives."

**What separates them:** giving the contention reason *and* the task-size reason, and
being explicit that they are two different optimisation targets. Most candidates give the
first and stop.

**Follow-up:** "What would you measure to see whether stealing is working?"
`getStealCount()` relative to the number of leaf tasks. Very high means the threshold is
too fine; flat during a busy period means workers are parked.

---

### Q3 — "Someone put a JDBC call inside a parallel stream. What happens?"

**Mid-level answer:** "It'll block the pool threads and slow things down. You should use
a separate thread pool for I/O."

**Senior answer:** "The specific failure is worse than 'slow'. Parallel streams run on
`ForkJoinPool.commonPool()`, which is JVM-global and sized to `availableProcessors() - 1`.
A JDBC call parks the OS thread in a socket read, and a parked worker can't steal, can't
help, and — critically — can't be compensated for, because the pool has no way to
distinguish a worker blocked in a syscall from a worker deep in a long computation. So on
a 4-core container, three concurrent JDBC calls remove *all* parallel-computation capacity
from the entire JVM, including code that has nothing to do with this feature and including
any library that uses a parallel stream internally. The signature is distinctive:
throughput collapses, latency explodes, and CPU sits near idle — which sends people
looking for a lock, and there isn't one. `jcmd Thread.print` shows exactly `cores - 1`
threads named `ForkJoinPool.commonPool-worker-N` in `socketRead0`. The fix is a dedicated
bounded executor sized by Little's Law, or virtual threads. And I'd enforce it — ErrorProne
or ArchUnit can fail the build on the no-executor `supplyAsync` overloads."

**What separates them:** the exact symptom triple (collapsed throughput, exploded latency,
idle CPU), the specific diagnostic command and what it shows, the "JVM-global including
libraries" scope, and proposing enforcement rather than a convention.

**Follow-up:** "What about `ManagedBlocker`?" It exists, the JDK uses it internally for
`CompletableFuture.join()`, compensation is capped by `maximumSpares`, and it is the right
tool for a bounded wait inside a CPU-bound computation — not a licence to do I/O in the
pool.

---

### Q4 — "How would you size a ForkJoinPool?"

**Mid-level answer:** "Set it to the number of cores, since the tasks are CPU-bound."

**Senior answer:** "For genuinely CPU-bound work, parallelism equal to the cores you're
willing to give it — which is not necessarily all of them. If the JVM is also serving
requests, saturating every core with batch work destroys your request latency, so I'd size
it to leave headroom and I'd derive the number from a measured trade-off: sweep the
parallelism, plot batch duration against request p99, and pick the knee. That's a
capacity decision with a product input, not a technical constant.

The thing I'd check first, though, is `availableProcessors()` *in the container*. It
respects the cgroup CPU quota, so a fractional CPU limit reports 1, the common pool's
parallelism goes to zero, and submitted tasks run in the calling thread — your parallel
code is silently sequential with no warning. I log `availableProcessors()` and every pool's
parallelism at startup for exactly that reason.

And I would not use the common pool for anything I care about. It's shared with every
library on the classpath and can't be shut down."

**What separates them:** treating sizing as a measured trade-off rather than a formula,
knowing the container behaviour and that it fails silently, and the operational habit of
logging it.

**Follow-up:** "What if it's not purely CPU-bound?" Then it should not be on a
`ForkJoinPool` at all; split the workload, or use `ManagedBlocker` for a genuinely bounded
wait — and from Java 21 the blocking half belongs on virtual threads.

---

### Q5 — "Java 21 runs virtual threads on a ForkJoinPool. Is that the common pool?"

This one separates people who read the release notes from people who read the source.

**Mid-level answer:** "I think so — virtual threads are scheduled onto carrier threads by
a ForkJoinPool."

**Senior answer:** "It's a `ForkJoinPool`, but a **separate, dedicated one**, not the
common pool — and it runs in FIFO mode rather than the common pool's LIFO-local mode. Its
default parallelism is `availableProcessors()`, not `availableProcessors() - 1`. The FIFO
choice makes sense: virtual threads are independent units of work with no parent-child
data locality to exploit, so LIFO would buy nothing and would hurt tail latency badly — a
virtual thread that became runnable first should get mounted first. Keeping the two pools
separate also matters, because otherwise a long parallel stream could starve request
handling. The connection to this topic is that pinning is exactly a work-stealing failure:
a pinned virtual thread renders its carrier unable to unmount, which is the same 'a parked
worker can't steal' problem, and with only `availableProcessors()` carriers it starves the
whole JVM."

**What separates them:** knowing it is a different pool, knowing the mode differs, having
a reason for the mode, and connecting pinning back to the work-stealing failure mode
rather than treating it as an unrelated fact.

**Follow-up:** "Can you tune it?" There are `jdk.virtualThreadScheduler.parallelism` and
`jdk.virtualThreadScheduler.maxPoolSize` system properties — check your JDK's
documentation, they are documented as tuning knobs rather than stable API, and I would not
build automation on them without verifying against the version I'm running.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. The owner pops LIFO and the thief steals FIFO. Construct the workload for which this
   asymmetry is actively *harmful* — where you would rather the owner took FIFO. What
   property of divide-and-conquer does your workload lack?

2. An FJP worker calling `join()` on a stolen task goes and helps the thief. Why is that
   safe? Specifically: what stops the helping from recursing indefinitely, and what
   stops it from running a task the joining thread should not be running?

3. The pool cannot distinguish a worker blocked in `socketRead0` from a worker two
   minutes into a long computation. Design a mechanism that *would* let it tell the
   difference. Then explain why the JDK does not do that.

4. Fractional CPU limits make `availableProcessors()` return 1 and the common pool's
   parallelism zero, and tasks then run in the caller's thread. Argue that this is the
   *right* fallback. Then argue it should throw instead. Which do you actually believe,
   and would your answer change if you were writing the JDK rather than using it?

5. Work stealing gives good load balance with almost no coordination. What is the price?
   Name at least two costs that a centralised scheduler would not pay, and say which
   workloads make those costs dominant.

6. The virtual-thread scheduler uses FIFO; the common pool uses LIFO-local. Given only
   that fact, what can you infer about the two workloads they were each designed for?
   Answer from the scheduling discipline alone, without using anything else you know
   about virtual threads.

7. You have a task that is 95% CPU-bound arithmetic and 5% a bounded `Semaphore` wait.
   Argue for `ManagedBlocker`, then argue for restructuring to remove the wait from the
   parallel phase. What information would you need to decide, and how would you get it?

---

## Quick reference card

### The mechanism in five lines

```
Each worker owns a deque.
Owner pushes/pops its own end   -> LIFO -> cache locality, stack-shaped memory use.
Thief steals from the other end -> FIFO -> the OLDEST task = the BIGGEST subtree.
join() does not park: the worker helps -- runs its own tasks or the thief's.
A worker parked in a syscall can steal nothing, help nothing, and is not compensated.
```

### Task API

```java
class Sum extends RecursiveTask<Long> {         // returns a value
    protected Long compute() {
        if (small enough) return computeDirectly();
        Sum left = ..., right = ...;
        left.fork();                  // push onto MY deque
        long r = right.compute();     // run on THIS thread
        return left.join() + r;       // usually a local pop
    }
}
class Action extends RecursiveAction { }        // returns void
invokeAll(left, right);                         // the idiomatic two-way split
```

### The common pool

```java
ForkJoinPool.commonPool()
  parallelism  = availableProcessors() - 1   // print it; do not assume
  threads      = daemon, named ForkJoinPool.commonPool-worker-N
  shutdown()   = documented no-op
  used by      = parallel streams, CompletableFuture *Async with no executor
```

```bash
-Djava.util.concurrent.ForkJoinPool.common.parallelism=8
-Djava.util.concurrent.ForkJoinPool.common.maximumSpares=256
```

### Introspection — the six numbers

```java
pool.getParallelism()            // target
pool.getPoolSize()               // actual workers incl. spares
pool.getActiveThreadCount()      // not idle
pool.getRunningThreadCount()     // NOT BLOCKED  <-- compare with active
pool.getQueuedTaskCount()        // in worker deques
pool.getQueuedSubmissionCount()  // external, not yet started
pool.getStealCount()             // cumulative
```

**The blocked-worker signature:** `running` << `active`, `steals` flat, `queuedTasks`
climbing, CPU idle.

### Diagnostic commands

```bash
jcmd <pid> Thread.print | grep -A6 'ForkJoinPool.commonPool-worker'
jcmd <pid> Thread.print | grep -c 'ForkJoinPool'
jfr print --events jdk.ThreadPark  recording.jfr | grep -A10 ForkJoinPool
jfr print --events jdk.SocketRead  recording.jfr | grep -A10 ForkJoinPool   # any hit = bug
docker run --rm --cpus=0.5 <image> java PoolFacts    # container parallelism check
```

### Two pools, not one

| | Common pool | Virtual-thread scheduler |
|---|---|---|
| Mode | LIFO local | FIFO |
| Default parallelism | `availableProcessors() - 1` | `availableProcessors()` |
| Users | parallel streams, default async CF | virtual thread mounting |
| Tuning property | `...ForkJoinPool.common.parallelism` | `jdk.virtualThreadScheduler.parallelism` |

### Gotchas checklist

- [ ] Never submit blocking work to the common pool. Never call `*Async` without an executor.
- [ ] `parallelStream()` uses the common pool. So does a library's internal parallel stream.
- [ ] `Math.max(1, cores - 1)` when constructing a pool. Never a computed zero.
- [ ] Log `availableProcessors()` and every pool's parallelism at startup.
- [ ] `fork()` one, `compute()` the other, `join()` the forked one. Or `invokeAll`.
- [ ] Sequential threshold measured with JMH, with a sequential baseline benchmark next to it.
- [ ] `ManagedBlocker` for bounded waits only, and know that compensation is capped.
- [ ] Alert on `running` << `active` with `queuedTasks` > 0.
- [ ] Fractional CPU limits and parallel computation are in direct conflict.

---

## When would I use this at work?

**1. Diagnosing "the service is slow and the CPU is idle".**

This is the highest-value use, and it takes ninety seconds once you know the shape. A
service whose throughput has collapsed while CPU sits near zero is not contended and is
not GC-bound — it is *parked*. One `jcmd Thread.print`, grep for `ForkJoinPool`, and you
either find your workers in `socketRead0` or you rule the whole class out and move on.
Most engineers spend an hour on GC logs and lock analysis before getting here.

**2. Reviewing a PR that adds `.parallel()` or `supplyAsync(...)`.**

Two questions, both mechanical, both answerable in the review. First: does the task block?
If yes, this belongs on a dedicated executor, or on virtual threads, and the PR is
incorrect as written. Second: was an executor passed? If the method name ends in `Async`
and there is no executor argument, that is the common pool, and the author almost
certainly did not intend to touch a JVM-global resource. These are the two most valuable
review comments in the whole concurrency phase because they are cheap to make and the bug
they prevent is expensive and hard to attribute.

**3. Capacity-planning a batch job that shares a JVM with request serving.**

Someone wants the nightly rollup to run in-process. The question is not "will it work" —
it will — but "what does it cost the p99 of everything else, and is that price
acceptable?" You answer it by sweeping the pool parallelism, plotting batch duration
against request p99, finding the knee, and presenting the trade-off as a table. That
conversation — a technical measurement handed to a product decision — is exactly what a
senior engineer is expected to produce, and it is the shape of Topic 129's capacity work.

---

## Connected topics

**Prerequisites:**

- **25 — Parallel streams:** where you first blocked the common pool. This topic is the
  mechanism behind that drill.
- **84 — Threads vs the event loop:** why a parked OS thread is genuinely gone, unlike a
  yielded coroutine.
- **90 — Executors and pool sizing:** `ThreadPoolExecutor`'s shared queue is the contrast
  that makes the deque design legible. Little's Law comes from here.
- **91 — `CompletableFuture`:** the `*Async`-without-an-executor trap, in its natural
  habitat.
- **95 — CAS and atomics:** the deque's `base` index is a CAS target; steal contention is
  a cache-line ping-pong.
- **96 — False sharing:** why the owner's `top` and the thief's `base` must not share a
  cache line.

**Also relevant:**

- **82 — Container awareness:** `availableProcessors()` under a cgroup quota, which is
  Trap 3 and the most common operational surprise in this topic.
- **78 — Profiling:** CPU time versus wall-clock time is how you tell "busy" from
  "parked".
- **77 — JMH:** the only honest way to pick a sequential threshold.
- **98 — Bug taxonomy:** collapsed throughput with idle CPU is a distinct diagnostic
  category, and this topic supplies one of its two main causes.
- **99 — jcstress:** the deque is lock-free, and its correctness is exactly the kind of
  happens-before argument you now know how to try to falsify.

**This unlocks:**

- **101 — Virtual threads:** the scheduler is a `ForkJoinPool` in FIFO mode, sized from
  `availableProcessors()`. Pinning is "a parked worker cannot steal", applied to carriers.
  You will not understand carrier starvation without this topic.
- **102 — Structured concurrency:** a `StructuredTaskScope`'s forks are scheduled by that
  same machinery, and the scope's join point is what finally gives fan-out proper
  cancellation.
- **105 — Backpressure:** an FJP has no backpressure at all. Its deques are effectively
  unbounded. That is fine for a bounded recursive split and dangerous for a stream of
  external submissions.
- **107 — Loom vs reactive:** both models sit on top of schedulers like this one; the
  question is which programming model you write against.
- **118 — Metrics:** the six-gauge dashboard from the Measurement section.

---

*Java baseline 21. `ForkJoinPool` has been in the JDK since 7 and its public API is
stable; the constructor gained saturation, spare and keep-alive controls in 9. The one
genuinely new fact at 21 is that the virtual-thread scheduler is a separate `ForkJoinPool`
in FIFO mode — the tuning properties for it are documented as knobs rather than stable
API, so verify them against your JDK rather than trusting a name from this document.
Everything about deques, LIFO-local and FIFO-steal is unchanged and is unlikely to
change.*
