# 101 — Virtual Threads (Loom): Carriers, Scheduling, Pinning

## Phase: 9 — Concurrency
## Category: ELITE
## Java baseline: 21  |  Notes features from: 21 / 25 (virtual threads are final in 21; JDK 24 changed the `synchronized` pinning story and removed the `jdk.tracePinnedThreads` property — both are flagged in place below)
## Project spine: switch `orderflow` to `spring.threads.virtual.enabled=true` and re-run the Topic 65 baseline at matched arrival rates. The deliverable is a before/after table, not an opinion.

---

## Mechanical statement

Read this three times. Every other paragraph in this document is elaboration on it.

> **A virtual thread's stack lives on the heap as a continuation — an object holding
> the frames that would otherwise sit in a fixed-size OS thread stack.**
>
> **To run, the virtual thread is MOUNTED onto a carrier: a real platform thread from a
> `ForkJoinPool` sized, by default, to `availableProcessors()`.**
>
> **On a blocking call it UNMOUNTS: the JDK copies the continuation's live stack frames
> off the carrier and onto the heap, and the carrier immediately picks up another
> virtual thread. When the blocking operation completes, the virtual thread is
> resubmitted, its frames are copied back onto some carrier, and it resumes on the line
> after the blocking call.**
>
> **It cannot unmount while there is a native frame on its stack, and — on some JDK
> versions — while it is inside a `synchronized` block. That is PINNING. A pinned
> virtual thread holds its carrier for the whole blocking duration. With only
> `availableProcessors()` carriers, a handful of pinned threads starves every other
> virtual thread in the JVM — while the CPU sits idle.**

That last sentence is the failure this document is built around. It is Topic 100's
"a parked worker cannot steal", relocated one level up: **a pinned virtual thread is a
carrier that cannot be reused.**

---

## The bridge from what you know

### This is the most important bridge in the curriculum for you. Read it slowly.

You already solved this problem. Node solved it in 2009. Java solved it in 2023. **You
arrived at the same destination by opposite routes**, and holding both routes in your
head at once is what makes you dangerous in a design review.

**Here is the problem both runtimes were solving.** A server handling 20,000 concurrent
requests, each of which spends 95% of its time waiting on a database, a payment gateway
or another service. The waiting is nearly free — it is a socket with no bytes on it. The
*expensive* part is what you had to allocate in order to be allowed to wait: in
classical Java, one OS thread per in-flight request, each with a stack reservation
(commonly around 1 MB of virtual address space, committed lazily), each visible to the
kernel scheduler, each costing a context switch when it wakes.

**Node's route: make the I/O non-blocking, and keep one thread.**

There is exactly one thread. A read is registered with `epoll` and returns immediately.
The "rest of the function" — everything after the `await` — is captured as a callback.
libuv's loop wakes when the socket is readable and invokes that callback. `async`/`await`
is syntax over the same thing: the TypeScript compiler (or V8 directly) rewrites your
function into a **state machine**, so the local variables that were live across the
`await` are stored in a heap-allocated closure object and the function resumes at the
right label.

Note what got moved to the heap in Node: **the continuation of your function**. Node
just made *you* (and the compiler) do the moving, and made the fact visible in your
source with a keyword.

**Loom's route: keep the blocking imperative code, and make the THREAD cheap.**

The code stays exactly as it was:

```java
Payment result = paymentGateway.charge(order);   // blocks. Looks like it always did.
inventory.reserve(order);                        // blocks.
```

There is no `await`, no callback, no colour on the function, no state machine in your
source. What changed is underneath: `paymentGateway.charge` eventually reaches a socket
read inside the JDK. On a virtual thread, that socket read does not park an OS thread.
It **registers with the platform poller, copies your live stack frames onto the heap as
a continuation, and releases the carrier**. Some carrier will resume you later.

Note what got moved to the heap in Java: **the continuation of your function**. Java
made the *runtime* do the moving, and made the fact invisible in your source.

### Same destination, opposite direction

| | Node / TypeScript | Java 21 virtual threads |
|---|---|---|
| What changed | The **I/O** became non-blocking | The **thread** became cheap |
| What you write | `async`/`await`, promise chains, explicit asynchrony | Ordinary blocking, sequential code |
| Where the continuation lives | Heap, as a closure/state machine | Heap, as a `Continuation` holding stack frames |
| Who creates it | The compiler, at your `await` points | The JDK runtime, at any blocking point |
| Is it visible in source? | Yes — that is the point of `async` | No — that is the point of Loom |
| Function colouring | Yes. `async` infects callers. | **No.** A method does not know or care. |
| Stack traces | Broken across `await` unless the engine stitches them | Whole. One trace, one logical task. |
| Debugger | Steps across await points awkwardly | Steps normally. Breakpoints work. |
| Thread-per-request | Impossible | The **recommended** model again |
| Cost of 100k in-flight | 100k closures | 100k continuations |
| CPU parallelism | 1 thread (plus worker_threads) | `availableProcessors()` carriers |
| Backpressure | None built in | **None built in** (Topics 105, 107) |

**The single sentence to carry:** *Node made the I/O asynchronous so one thread could
serve many requests; Loom made the thread cheap so thread-per-request could serve many
requests. Both end up with a heap-allocated continuation per in-flight operation. Only
one of them made you rewrite your code.*

### The corollary that matters most

Because Loom does not colour functions, **the entire existing Java ecosystem became
scalable without being rewritten.** JDBC, `HttpClient`, `InputStream`, Hibernate,
`Thread.sleep`, `BlockingQueue.take()` — all of it unmounts correctly, because the JDK's
blocking primitives were retrofitted to be continuation-aware.

That is a genuinely different outcome from Node, where the transition from callbacks to
promises to `async`/`await` required rewriting libraries three times, and where a
synchronous library is permanently unusable on the main thread. In Java, the same
`OrderRepository` you wrote in Topic 47 runs unmodified and now scales to a hundred
thousand concurrent callers. **That is the payload of this topic.**

### What virtual threads do NOT do

This list is the difference between a mid-level answer and a senior one. Say all of it.

**1. They do not add CPU.** Carriers are `availableProcessors()` platform threads. If
your work is CPU-bound, you have exactly the parallelism you had before, plus scheduling
overhead. A million virtual threads doing arithmetic finish no sooner than
`availableProcessors()` platform threads doing the same arithmetic — slightly later, in
fact.

**2. The database does not get faster.** Postgres still executes the same query in the
same time. Virtual threads raise the number of *concurrent waiters*, not the *service
rate* of anything you are waiting on.

**3. If the bottleneck is a 20-connection HikariCP pool, you have moved the queue, not
removed it.** This is the most common disappointment. Before: 200 Tomcat threads queue
for 20 connections; 180 wait. After: 20,000 virtual threads queue for 20 connections;
19,980 wait — now inside `HikariPool.getConnection` instead of inside the Tomcat
acceptor. Same throughput, worse latency distribution, more memory, and a `connection is
not available` timeout storm instead of a clean accept-queue backlog. **Topic 109 is
where this becomes concrete.** The correct sequence is: find the real constraint first,
then decide whether thread count was ever it.

**4. They do not give you backpressure.** Nothing about a virtual thread signals to an
upstream producer that you are behind. Unbounded virtual-thread creation is unbounded
concurrency against a downstream that has a finite capacity. You still need a
`Semaphore`, a bounded queue, or a rate limiter. **This is the residual that Topic 107's
recommendation turns on, and it is the one thing reactive still uniquely provides
(Topic 105).**

**5. They must NOT be pooled.** A thread pool exists to amortise the cost of creating a
thread. Creating a virtual thread costs roughly an object allocation. Pooling them
reintroduces exactly the ceiling you adopted them to remove, *and* it breaks the
`ThreadLocal` hygiene assumptions, *and* it makes structured concurrency (Topic 102)
impossible. `Executors.newVirtualThreadPerTaskExecutor()` is not a pool despite living
on the same factory class — it creates one new virtual thread per submitted task. Trap 1
below is a whole section on people getting this wrong.

**6. They do not improve single-request latency.** One request that takes 300 ms takes
300 ms. Virtual threads change the shape of the *concurrency* curve, which shows up in
p99 under load and in the throughput ceiling — not in the p50 of an idle system. If your
Topic 65 baseline p50 does not move after the switch, that is the expected result, not a
failed migration.

**Verdict: STRONG ANALOGUE for the problem and the destination. INVERTED analogue for
the mechanism. Your Node instincts about "cheap concurrency for I/O" transfer whole; your
Node instincts about "don't block" transfer only to the carrier, not to the virtual
thread.**

---

## What is this?

A **virtual thread** is a `java.lang.Thread` — same class, same API, `Thread.sleep`,
`Thread.currentThread()`, interruption, `ThreadLocal` — whose execution is scheduled by
the JVM rather than by the operating system.

```java
Thread.ofVirtual().name("orderflow-vt-", 0).start(() -> handle(request));

Thread vt = Thread.startVirtualThread(() -> handle(request));

try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    for (var order : orders) {
        executor.submit(() -> enrich(order));   // one NEW virtual thread each
    }
}   // close() waits for all submitted tasks — this is an AutoCloseable ExecutorService
```

Three participants, and you must be able to name each one separately:

| Term | What it is | How many |
|---|---|---|
| **Virtual thread** | A `Thread` whose stack is a heap object. Cheap. Created per task. | Millions, potentially |
| **Carrier** | A real platform thread that a virtual thread is temporarily mounted on. | `availableProcessors()` by default |
| **Scheduler** | The `ForkJoinPool` (in FIFO mode) that decides which virtual thread mounts on which carrier next. | One, JVM-wide, **not** the common pool |

A **platform thread** is the old thing: a 1:1 wrapper over an OS thread. Still exists,
still the right choice for a long-running dedicated loop, still what carriers are.

### The three properties that follow from "the stack is on the heap"

1. **The stack grows and shrinks.** A platform thread reserves its stack up front. A
   virtual thread's stack is a chain of *stack chunks* on the heap that grows as you
   recurse and is collected when the thread ends. A virtual thread that is three frames
   deep costs about what three frames cost.
2. **They are garbage-collected, not destroyed.** A finished virtual thread is
   unreachable and gets collected. There is no `destroy()`, no OS resource to leak, and
   crucially **no reason to reuse one**.
3. **They are invisible to the OS.** `top -H` shows carriers, not virtual threads. This
   changes your diagnostics, and Hands-on proof 4 below is how you get the picture back.

### What is deliberately not there

- **No thread priorities.** `setPriority` is a no-op.
- **No thread groups** in any meaningful sense.
- **Not daemon-configurable** — virtual threads are always daemon threads. They will not
  keep the JVM alive.
- **`Thread.stop`/`suspend`/`resume`** throw `UnsupportedOperationException`.
- **No `ThreadLocal` prohibition, but a strong discouragement.** They work; see Trap 4
  for why "works" is not the same as "advisable".

---

## Why does it matter?

**1. It is the default programming model for new Java services from 21 onward.** Not a
niche tuning option. `spring.threads.virtual.enabled=true` is one line in
`application.yaml`, and it changes the threading model of your entire request path. You
will be asked about it in every senior loop from now on.

**2. It resolves an argument you have been living inside for a decade.** "Threads are
expensive, so you need an event loop" was true and is now substantially false in Java.
Being able to say precisely *which part* became false — thread creation and stack
memory — and which part did not — CPU parallelism, downstream capacity, backpressure —
is the whole of Topic 107.

**3. The failure mode is invisible on every dashboard you own.** Carrier starvation
presents as: throughput collapsed, latency exploded, CPU near idle, no locks in the
thread dump, no GC pressure. Every instinct you have points at the database. The
database is fine. This document exists so that you spend ninety seconds on this instead
of an afternoon.

**4. `orderflow` is exactly the workload virtual threads were built for — and exactly
the workload where the Hikari trap fires.** 70% catalogue reads (short DB call), 20%
order reads (fan-out), 10% order placement (DB write plus a slow payment gateway). High
blocking-I/O concurrency: the ideal case. A 20-connection pool in front of it: the
classic disappointment. You are going to measure both.

---

## Machine-level reality

### The continuation

Underneath `Thread.ofVirtual()` sits `jdk.internal.vm.Continuation`, a delimited
continuation primitive with two operations:

```
Continuation.run()    — start or resume executing, on the CURRENT carrier
Continuation.yield()  — stop executing; copy the live frames off the stack onto the heap
```

`yield` is what "unmount" means mechanically. It walks the stack from the continuation's
entry point down to the current frame, copies those frames into a **stack chunk** object
on the heap (`jdk.internal.vm.StackChunk`), and returns control to whatever called
`run()` — which is the carrier's mount loop. The carrier then goes back to the scheduler
for more work.

`run` is what "mount" means: it copies frames from the heap chunk back onto the carrier's
real stack and jumps to the saved instruction pointer.

**Lazy copying.** The implementation does not necessarily copy the entire stack on every
mount and unmount; it copies what it must, and the return barrier means frames are often
restored incrementally as the thread returns into them. Two consequences you can reason
about:

- A **deep** stack makes unmounting more expensive. A virtual thread that blocks 200
  frames deep costs more to unmount than one that blocks 5 frames deep. Deep reactive-
  style or deeply-recursive code is not free here.
- A virtual thread that blocks and resumes **frequently** pays this cost repeatedly. A
  tight loop of tiny blocking operations is the worst shape; one long block is the best.

**This is heap.** Topic 68's heap — same TLABs, same generations, same collector. Stack
chunks are objects. A million parked virtual threads is a million live object graphs the
collector must trace. That is the real memory cost, and it is measured in the heap, not
in RSS-minus-heap. It is also why ZGC or Shenandoah (Topic 72) and virtual threads are
often discussed together: many live, long-lived, medium-sized objects is a shape that
rewards a concurrent collector.

### The scheduler is a ForkJoinPool — but not the common one

This is Topic 100, one level up. Hold the distinction; it is a favourite interview
follow-up.

| | Common pool | Virtual-thread scheduler |
|---|---|---|
| Instance | `ForkJoinPool.commonPool()` | A separate, dedicated `ForkJoinPool` |
| Mode | LIFO local (async mode off) | **FIFO** (async mode on) |
| Default parallelism | `availableProcessors() - 1` | `availableProcessors()` |
| Used by | parallel streams, default `CompletableFuture` async | mounting virtual threads onto carriers |
| Tuning | `...ForkJoinPool.common.parallelism` | `jdk.virtualThreadScheduler.parallelism`, `jdk.virtualThreadScheduler.maxPoolSize` |

**Why FIFO?** Because there is no parent-child data locality to exploit. Virtual threads
are independent requests, not recursive splits. FIFO means the virtual thread that became
runnable first gets mounted first, which is the right latency behaviour for
request serving. LIFO would give better cache reuse and much worse tail latency.

> **Do not build automation on `jdk.virtualThreadScheduler.*`.** These are documented as
> tuning knobs, not stable API, and I am not going to assert their exact semantics across
> JDK versions. Read your JDK's own documentation before setting them, and print
> the value you actually got rather than the value you set.

**Container reality — this is Topic 82 wearing a different hat.** The carrier count comes
from `availableProcessors()`, which respects the cgroup CPU quota on a modern JDK:

| Container setting | Likely `availableProcessors()` | Carriers | Consequence |
|---|---|---|---|
| `--cpus=8` | 8 | 8 | Fine |
| `--cpus=2` | 2 | 2 | **Two** pinned virtual threads stop the JVM |
| `--cpus=0.5` | 1 | 1 | **One** pinned virtual thread stops the JVM |

That last row is not hypothetical. A conservative Kubernetes CPU request plus one
`synchronized` block across a slow call is a complete outage, and this is exactly the
drill below. **Log `availableProcessors()` at startup.** One line.

### Where the unmount actually happens

The JDK's blocking primitives were retrofitted to yield the continuation rather than
park the OS thread. The ones that matter to `orderflow`:

| Operation | Unmounts? |
|---|---|
| `Thread.sleep` | Yes |
| Socket read/write via `java.net.Socket`, `SocketChannel`, `HttpClient` | Yes |
| `BlockingQueue.take/put`, `LinkedBlockingQueue` etc. | Yes |
| `ReentrantLock.lock`, `Condition.await`, `Semaphore.acquire` (all AQS) | Yes |
| `CompletableFuture.get/join` | Yes |
| `Object.wait()` | Version-dependent; see the pinning note below |
| `synchronized` block entry / holding across a block | **Version-dependent. This is the whole story.** |
| **File I/O** (`FileInputStream`, most `java.io` file paths) | **Often no** — many file operations go through native frames or an internal pool. Do not assume. |
| Any JNI / native method that blocks | **No. Pins, on every JDK.** |
| A class initializer (`<clinit>`) that blocks | **No. Pins, on every JDK.** |

### Pinning, stated exactly

A virtual thread is **pinned** when it cannot yield. It then behaves like a platform
thread: the blocking call holds the carrier for its full duration.

Two causes that pin on **every** JDK version:

1. **A native frame on the stack.** The JVM cannot copy native frames to the heap; it
   does not own them. Any JNI call that then blocks pins.
2. **A class initializer in progress.** Blocking inside `<clinit>` pins.

And one cause that is **version-dependent, and you must check your own JDK**:

3. **Being inside a `synchronized` block or method.** On JDK 21 through 23, the object
   monitor implementation ties monitor ownership to the carrier thread, so a virtual
   thread inside `synchronized` cannot unmount. **JDK 24 shipped work to remove this
   restriction for most cases** (the "synchronize virtual threads without pinning"
   change). On JDK 24/25 you may therefore see no pinning at all in the drill below.

> **I am not going to tell you which outcome you will get.** It depends on your exact JDK
> build, and the change has edge cases. The drill below is deliberately designed so that
> **both outcomes teach you something**, and the diagnostic step — confirming with
> `-Djdk.tracePinnedThreads=full` on 21–23, or the JFR `jdk.VirtualThreadPinned` event on
> any version — is the part you are actually learning. Run `java --version` first and
> write the version at the top of your notes.

**Why `synchronized` was hard and `ReentrantLock` was not.** A `synchronized` monitor's
ownership is recorded against the platform thread in the JVM's monitor machinery — deep
in the runtime, entangled with the interpreter, the JIT's lock elision and biased-locking
history, and native frames. `ReentrantLock` (Topic 94) is *ordinary Java code*: AQS is a
Java class holding an `int` and a queue of `Thread` references, and parking goes through
`LockSupport.park`, which the JDK taught to yield a continuation. **That is why the fix
for pinning on JDK 21 is "replace `synchronized` with `ReentrantLock`" — the lock that is
written in Java can participate in a Java-level scheduling mechanism, and the one written
in the runtime could not.**

### `ThreadLocal` on virtual threads — the arithmetic

`ThreadLocal` works. Each virtual thread has its own `ThreadLocalMap`. That is exactly
the problem:

```
platform threads:   200 threads  x  1 context object  =    200 contexts
virtual threads:  50,000 threads x  1 context object  = 50,000 contexts
```

Nothing leaked. The retention is *correct* — each live thread holds its own value, and
when the thread dies the map dies with it. But you multiplied a per-thread cost by a
number that grew by two orders of magnitude. If that context object holds a parsed JWT, a
tenant record and a `SecurityContext` (Topic 56 — `SecurityContextHolder` is
`ThreadLocal`-backed), you have just moved several hundred megabytes into the heap
without changing a line of your own code.

This is also the exact reason `ScopedValue` exists, and it is Topic 102.

Note the contrast with Topic 79's `ThreadLocal` leak: there, a *pooled* platform thread
outlived the request and retained a value nobody removed. Here nothing leaks — the cost
is simply multiplied. **Two different failures, same API, and you should be able to
distinguish them on sight.**

### Thread identity and diagnostics

```java
Thread t = Thread.currentThread();
t.isVirtual()      // true / false — the single most useful line in this whole document
t.getName()        // EMPTY STRING by default for virtual threads. Name them.
t.threadId()       // stable long id (Java 19+); getId() is deprecated
```

The default `toString()` of an unnamed virtual thread looks roughly like
`VirtualThread[#31]/runnable@ForkJoinPool-1-worker-3` — the part after the `@` is the
**carrier**, and that is how you see mounting in a log line. *(Illustration of the
format, not captured output — the exact shape varies by JDK version, so read your own.)*

Two habits worth installing now:

```java
// Name them at creation. An unnamed virtual thread in a log is a wasted diagnostic.
Thread.ofVirtual().name("orderflow-callback-", counter.getAndIncrement()).start(task);

// Print isVirtual() plus the carrier at any point you care about:
log.info("thread={} virtual={}", Thread.currentThread(), Thread.currentThread().isVirtual());
```

---

## Concurrency trace

Before any correct code: the failure, step by step. **This is the headline trace of the
topic.**

### The scenario

`orderflow` runs in a container with `--cpus=2`. So `availableProcessors()` is 2 and the
virtual-thread scheduler has **two carriers**. (Two is chosen so the trace fits in two
columns and stays readable. On an 8-core box the same thing happens with 8 pinned
threads; the arithmetic scales, the mechanism does not change.)

The service is on `spring.threads.virtual.enabled=true`. Every HTTP request gets its own
virtual thread.

Someone added an idempotency guard to payment submission, six months ago, when the
service was on platform threads. It looked completely reasonable then:

```java
// PaymentService.java  -- the defect
public PaymentResult submit(Order order) {
    synchronized (this) {                                  // <-- guard
        if (alreadySubmitted(order.id())) {
            return PaymentResult.duplicate(order.id());
        }
        PaymentResult result = gateway.charge(order);      // <-- 800 ms HTTP call
        markSubmitted(order.id(), result);                 //     INSIDE the monitor
        return result;
    }
}
```

The payment gateway is currently taking 800 ms per call — not an outage, just a slow
afternoon. Load is the Topic 65 mix: 10% of requests place an order.

> **JDK caveat, stated once and applying to the whole trace.** This trace describes the
> JDK 21–23 behaviour of `synchronized`. On JDK 24/25 the monitor may no longer pin, and
> steps 3 onward would not happen for `synchronized` specifically. **The trace is still
> exactly correct on every JDK if the blocking call sits behind a native frame** — a JNI
> payment SDK, a native crypto library, some file-I/O paths. Read it as the shape of
> carrier starvation, not as a claim about your build.

### The interleaving

| Step | Carrier-1 (`ForkJoinPool-1-worker-1`) | Carrier-2 (`ForkJoinPool-1-worker-2`) |
|---|---|---|
| 1 | Idle. Takes VT-1 (order placement) from the scheduler queue and mounts it. | Idle. Takes VT-2 (order placement) and mounts it. |
| 2 | VT-1 enters `synchronized (this)`. Acquires the monitor. | VT-2 reaches `synchronized (this)`. Monitor is held by VT-1. **Blocks on monitor entry.** |
| 3 | VT-1 calls `gateway.charge(order)`. Reaches a socket read. The JDK is ready to yield the continuation — **but there is a monitor held by this thread, so it cannot.** | VT-2 is blocked on monitor entry — also inside the monitor machinery, also unable to yield. |
| 4 | **PINNED.** Carrier-1 is stuck inside `gateway.charge` for 800 ms. It cannot mount another virtual thread. | **PINNED.** Carrier-2 is stuck waiting for the monitor. It cannot mount another virtual thread. |
| 5 | Still pinned. 40 catalogue-read virtual threads become runnable — their DB responses arrived, their continuations are on the heap and ready. | Still pinned. |
| 6 | Still pinned. **Those 40 virtual threads sit in the scheduler's FIFO queue. There is no carrier to mount them on.** | Still pinned. |
| 7 | Still pinned. Tomcat accepts more connections and creates more virtual threads. Each one runs zero instructions. | Still pinned. |
| 8 | Still pinned. A background virtual thread that only refreshes an in-memory cache — no I/O, no locks, ten microseconds of work — is also queued and does not run. | Still pinned. |
| 9 | Still pinned. Scheduler queue depth is now in the thousands. Heap is climbing: every queued virtual thread is a live continuation. | Still pinned. |
| 10 | **CPU utilisation across both cores: near zero.** Two threads are asleep in the kernel on a socket read and a monitor. Nothing else in the JVM can execute. | — |
| 11 | At t=800 ms, `charge` returns. VT-1 marks the payment, exits `synchronized`, unmounts on its next block, and Carrier-1 frees. | Monitor released; VT-2 acquires it, enters `charge`, and pins Carrier-2 for another 800 ms. |
| 12 | Carrier-1 mounts one of the thousands of queued virtual threads. It runs for microseconds and completes. Then the next order-placement virtual thread reaches `synchronized` and pins Carrier-1 again. | Still pinned. |
| 13 | The system is now serving roughly **one order placement per 800 ms, serialised**, with a brief burst of catalogue reads between pins. | — |

### Outcome, in business terms

The service has an effective concurrency of **two**, on a machine that was serving
thousands of concurrent requests an hour ago.

- **Catalogue reads** — 70% of traffic, code path completely unchanged, touches no
  payment code, holds no locks — go from their recorded Topic 65 p99 to timeouts. The
  error rate on `/products` climbs to whatever your gateway timeout produces.
- **Order placement** is serialised at one per gateway round-trip. Throughput on the
  business-critical path collapses to roughly 1.25 per second.
- **Heap climbs** because every queued virtual thread is a live continuation nobody can
  run. If the arrival rate holds, this ends in `OutOfMemoryError` — a memory symptom
  whose actual cause is a scheduling bug.
- **CPU sits near idle.** Both cores are doing nothing.
- **The Kubernetes liveness probe** (Topic 121) is itself served by a virtual thread that
  cannot be mounted. It times out. **Kubernetes restarts the pod.** The restart drops all
  in-flight work, the load rebalances onto the remaining pods, and they pin too. This is
  how a slow third party becomes a cluster-wide restart loop.

**And the dashboard says the service is idle.** That is the diagnostic signature, and it
is why this bug takes people hours: throughput collapsed, latency exploded, CPU flat,
GC quiet, no `BLOCKED` platform threads worth mentioning, database perfectly healthy.
Every instinct points downstream. The cause is two lines of six-month-old code that were
correct on platform threads.

An engineer who has read this trace runs one command and is done:

```bash
# JDK 21-23:
# (re-run the service with) -Djdk.tracePinnedThreads=full

# Any version, and the one to use in production:
jcmd <pid> JFR.start name=pin settings=profile
# ... 60 seconds of load ...
jcmd <pid> JFR.dump name=pin filename=pin.jfr
jfr print --events jdk.VirtualThreadPinned pin.jfr | head -60
```

---

## Example 1 — minimal

The smallest thing that shows mounting, unmounting and carrier identity. Save as
`CarrierProbe.java` and run it directly with `java CarrierProbe.java`.

```java
package com.orderflow.lab;

import java.util.concurrent.CountDownLatch;

public class CarrierProbe {

    static String where(String label) {
        Thread t = Thread.currentThread();
        return "%-14s | virtual=%-5s | %s".formatted(label, t.isVirtual(), t);
    }

    public static void main(String[] args) throws Exception {
        System.out.println("availableProcessors = "
                + Runtime.getRuntime().availableProcessors());
        System.out.println(where("main"));

        var latch = new CountDownLatch(3);

        for (int i = 0; i < 3; i++) {
            final int id = i;
            Thread.ofVirtual().name("probe-", id).start(() -> {
                System.out.println(where("before sleep " + id));
                try {
                    Thread.sleep(200);          // UNMOUNTS here
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
                System.out.println(where("after sleep " + id));
                latch.countDown();
            });
        }

        latch.await();
    }
}
```

**What to look for:**

1. `main` reports `virtual=false`.
2. The three probes report `virtual=true`.
3. The `toString()` includes a carrier name after an `@`.
4. **Compare the carrier in "before sleep N" with the carrier in "after sleep N".** They
   may differ. That difference *is* the unmount and remount: the continuation came off
   one carrier and went back onto whichever one was free.

| What you see | What it means |
|---|---|
| Carrier name changes across the sleep for at least one probe | You have observed unmount/remount directly. The virtual thread's identity (`probe-N`) is stable; the carrier underneath it is not. |
| Carrier name is the same before and after for all three | Also fine and common — with few threads and idle carriers, the same one is often free again. Raise the loop to 200 probes and it will change. |
| The virtual thread name is empty | You forgot `.name(...)`. Do not ship unnamed virtual threads. |
| `virtual=false` inside the lambda | You used `Thread.ofPlatform()` or a plain `new Thread(...)`. Check the factory. |

**Now the second half of the minimal example — the one that matters.** Add a
`synchronized` block around the sleep and see whether your JDK pins:

```java
    static final Object MONITOR = new Object();

    // Replace the body of the virtual thread with this:
    synchronized (MONITOR) {
        Thread.sleep(200);      // blocking INSIDE a monitor
    }
```

Run with a small carrier count so the effect is visible:

```bash
java -Djdk.virtualThreadScheduler.parallelism=1 \
     -Djdk.virtualThreadScheduler.maxPoolSize=1 \
     -Djdk.tracePinnedThreads=full \
     CarrierProbe.java
```

**What to look for:** total wall-clock time for the three probes, and any pinning output.

| What you see | What it means |
|---|---|
| Roughly 600 ms total (3 x 200 ms, serialised) **and** pinning output on stderr | Your JDK pins on `synchronized`. This is the JDK 21–23 behaviour. The drill below will reproduce fully. |
| Roughly 200 ms total, no pinning output | The monitor did **not** pin. Either your JDK includes the JDK 24+ change, or the monitor was uncontended in a way that avoided it. **The correct reading is "this JDK fixed it" — then confirm with the JFR event rather than concluding from a timing.** |
| No output from `jdk.tracePinnedThreads` at all, on any run | On JDK 24+ this system property was removed. Its silence is not evidence of no pinning. Use JFR. |
| Roughly 600 ms but no pinning output on JDK 21–23 | Look again at whether the property name is spelled correctly and whether stderr is being captured. Then use JFR to settle it. |

> **This is the version-dependence in miniature, and it is the honest state of the world
> in 2026: you cannot know the answer for your JDK without running it.** Write down
> `java --version` and your result. You will need both for the drill.

---

## Example 2 — production scenario (on the project spine)

### The constraints

`orderflow` at the Topic 65 baseline:

- Containerised, Postgres + Redis alongside, k6 driving an **open-model** arrival rate.
- Dataset: 100k products, 1M orders, 5M order lines.
- Mix: 70% catalogue read, 20% order read, 10% order placement.
- Recorded p50/p95/p99/p999 per endpoint, committed under `/docs/java/baselines/`.
- HikariCP `maximum-pool-size` at whatever Topic 65 recorded — write the number down now,
  because it is the protagonist of the second half of this section.
- Order placement calls an external payment gateway.

### Step 1 — the switch, and everything that has to be checked with it

```yaml
# application-load.yaml -- committed next to the Topic 65 baseline artefacts
spring:
  threads:
    virtual:
      enabled: true          # THE switch. One line.
```

That property makes Spring Boot use a virtual-thread-per-task executor for the servlet
container's request handling, for `@Async`, and for several scheduling and messaging
integration points. **It does not change your own executors.** Anything you constructed
with `Executors.newFixedThreadPool(...)` is still a fixed platform pool.

So the switch is one line and the audit is not. Here is the audit, and it is the actual
work of this migration:

```java
@Configuration
class OrderflowThreadingConfig {

    /**
     * Log the world at startup. Three lines that have saved more incident time than
     * most monitoring. Topic 82 taught you to distrust container CPU assumptions.
     */
    @Bean
    ApplicationRunner threadingFacts() {
        return args -> {
            log.info("availableProcessors={}", Runtime.getRuntime().availableProcessors());
            log.info("virtual-threads-enabled={}", virtualThreadsEnabled);
            log.info("carriers(default)={}  -- verify against your JDK's scheduler docs",
                     Runtime.getRuntime().availableProcessors());
        };
    }

    /**
     * The payment-notification pool from Topic 90 -- now virtual-thread-per-task,
     * with an EXPLICIT concurrency limit, because virtual threads give you no
     * backpressure (Topic 105) and the notification endpoint is fragile.
     */
    @Bean(destroyMethod = "close")
    ExecutorService notificationExecutor() {
        return Executors.newVirtualThreadPerTaskExecutor();
    }

    /** The limiter is a SEPARATE concern from the executor. Topic 97's bulkhead. */
    @Bean
    Semaphore notificationBulkhead(
            @Value("${orderflow.notifications.max-in-flight:64}") int permits) {
        return new Semaphore(permits);
    }
}
```

The audit checklist, in the order you should perform it:

| # | Check | Why | Where it goes wrong |
|---|---|---|---|
| 1 | Every `synchronized` on a path that does I/O | Pinning (version-dependent) | Idempotency guards, lazy-init caches, `Collections.synchronizedMap` |
| 2 | Every `ThreadLocal` on the request path | Multiplied by thread count now | `SecurityContextHolder` (T56), MDC (T120), tenant context |
| 3 | Every fixed-size executor still in the code | You did not migrate it; it may now be the bottleneck | `@Async` custom `TaskExecutor`, batch pools |
| 4 | HikariCP `maximum-pool-size` | **The queue moved here** (T109) | Every single time |
| 5 | Any JNI / native library on a blocking path | Pins on **every** JDK | Native crypto, native payment SDKs |
| 6 | Any unbounded virtual-thread creation | No backpressure | `submit()` in a loop with no limiter |
| 7 | Downstream rate limits and connection pools | You just raised concurrency against them | The payment gateway's own limit |

### Step 2 — the code that is now wrong, and the code that replaces it

**The idempotency guard from the trace, fixed.**

```java
// BEFORE -- correct on platform threads, a carrier-starvation bomb on virtual threads
public PaymentResult submit(Order order) {
    synchronized (this) {
        if (alreadySubmitted(order.id())) return PaymentResult.duplicate(order.id());
        PaymentResult result = gateway.charge(order);   // 800 ms, inside the monitor
        markSubmitted(order.id(), result);
        return result;
    }
}
```

Two things are wrong, and they are independent:

1. The lock is `synchronized`, which may pin (Topic 85 is the mechanism; Topic 94 is the
   replacement).
2. **A slow network call is inside a mutual-exclusion region at all.** That was already
   a design defect on platform threads — it serialised every payment in the JVM. Virtual
   threads did not create this bug; they raised its blast radius from "payments are slow"
   to "the JVM stops".

Fix the second problem and the first mostly evaporates:

```java
// AFTER -- per-order lock, ReentrantLock, and the gateway call OUTSIDE the critical
// section. Note the ordering: reserve intent, call, then record.
@Service
public class PaymentService {

    private final ConcurrentHashMap<OrderId, ReentrantLock> locks = new ConcurrentHashMap<>();
    private final PaymentGateway gateway;
    private final PaymentRepository payments;

    public PaymentResult submit(Order order) {
        // Per-ORDER exclusion, not per-SERVICE. Two different orders never contend.
        ReentrantLock lock = locks.computeIfAbsent(order.id(), k -> new ReentrantLock());

        if (!lock.tryLock()) {
            // Someone else is submitting this exact order right now.
            return PaymentResult.inProgress(order.id());
        }
        try {
            // 1. Cheap, local, inside the lock: has this order already been charged?
            Optional<Payment> existing = payments.findByOrderId(order.id());
            if (existing.isPresent()) {
                return PaymentResult.duplicate(order.id());
            }
            // 2. Record INTENT durably before calling out, so a crash is recoverable.
            payments.saveIntent(order.id());
        } finally {
            lock.unlock();              // <-- released BEFORE the network call
            locks.remove(order.id());
        }

        // 3. The slow call, holding NO lock. This unmounts cleanly on every JDK.
        PaymentResult result = gateway.charge(order);

        // 4. Record the outcome. Idempotent by (orderId) unique constraint.
        payments.recordOutcome(order.id(), result);
        return result;
    }
}
```

**Read what changed, in the right order of importance:**

1. The gateway call holds no lock. The virtual thread unmounts at the socket read and the
   carrier goes and serves 5,000 catalogue reads. **This is the actual fix.**
2. Exclusion is per-order, so two different orders never contend at all.
3. `ReentrantLock` instead of `synchronized`, which removes the version-dependent pinning
   question entirely — on JDK 21 it is the fix, on JDK 25 it is belt-and-braces at zero
   cost.
4. `tryLock()` rather than `lock()`, so a duplicate submission returns a defined result
   instead of queueing (Topic 94's Fix 2).
5. The real idempotency guarantee is a **unique constraint on `payments(order_id)`** in
   the database, not the lock. The lock is an optimisation that avoids the wasted gateway
   call; the constraint is the correctness. Any in-memory guard is per-JVM and you run
   more than one pod.

**The `ThreadLocal` audit, concretely.** `SecurityContextHolder` is `ThreadLocal`-backed
(Topic 56). At 200 platform threads that was 200 security contexts. At 30,000 concurrent
virtual threads it is 30,000. Measure it rather than guess:

```bash
jcmd <pid> GC.class_histogram | grep -iE 'SecurityContext|Authentication|TenantContext'
```

**What to look for:** instance count roughly tracking your concurrent request count, and
whether the retained size is a meaningful fraction of your heap. If it is, Topic 102's
`ScopedValue` is the structural answer.

### Step 3 — the honest caveat about what you will measure

Before you run anything, write down your prediction. Then run it. The most likely
outcomes, in order of probability:

| Outcome | What it means | What to do next |
|---|---|---|
| p50 unchanged, p99 improves at high arrival rates, throughput ceiling rises | The expected win. Thread count was a real constraint above some arrival rate. | Record the crossover arrival rate. That number is the whole justification. |
| **Nothing changes at any arrival rate** | **The most likely outcome for `orderflow` as specified.** The bottleneck is the connection pool or the database, not threads. | Go to Topic 109. Do not tune the scheduler. |
| Throughput drops and latency worsens | Something pins, or you kept an executor that is now the constraint, or you removed the accept-queue backpressure that was protecting the database. | Run the JFR pinned event first, then check pool metrics. |
| Heap use climbs noticeably | Continuations plus multiplied `ThreadLocal`s. Expected, and it is a real cost. | Class histogram; decide whether it is contexts or continuations. |
| Errors change shape: fewer connection-refused, more pool-timeout | **You moved the queue.** Textbook. | Topic 109. This is a good finding, not a failure. |

The second and fifth rows are the ones people do not write down honestly, and they are
the most valuable outputs of the whole exercise.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — pooling virtual threads

**Wrong approach.**

```java
// Seen in real code, written by someone being careful:
ExecutorService executor = Executors.newFixedThreadPool(
        200, Thread.ofVirtual().factory());     // "a pool of virtual threads"

// Or the Spring version:
@Bean
TaskExecutor orderExecutor() {
    var ex = new ThreadPoolTaskExecutor();
    ex.setVirtualThreads(true);                 // depends on Spring version; the idea is the same
    ex.setCorePoolSize(200);
    ex.setQueueCapacity(1000);
    return ex;
}
```

**Exact symptom.** Throughput identical to the platform-thread version. Under load,
`getQueueSize()` climbs and p99 climbs with it while CPU stays low. A thread dump shows
exactly 200 virtual threads and a queue behind them. **The migration produced no measurable
benefit whatsoever, and the team concludes "virtual threads are overhyped".**

**Root cause.** `newFixedThreadPool` is a `ThreadPoolExecutor`: it creates *N* threads and
reuses them, feeding them from a shared queue. Handing it a virtual thread factory changes
what kind of thread it creates and changes nothing else. You have **capped concurrency at
200** — the exact ceiling you adopted virtual threads to remove — and added continuation
overhead on top. A pool exists to amortise thread *creation*; virtual thread creation is
about as expensive as an object allocation, so there is nothing to amortise.

Worse, you have reintroduced the Topic 79 leak shape: a pooled thread outlives the task,
so a `ThreadLocal` nobody removed is retained until the pool recycles the thread — which
now never happens, because virtual threads in a pool are long-lived.

**Fix.**

```java
// One new virtual thread per task. NOT a pool, despite the class it comes from.
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    for (Order order : batch) {
        executor.submit(() -> enrich(order));
    }
}   // close() blocks until every submitted task finishes
```

**And when you genuinely need to limit concurrency** — because the downstream is fragile,
not because threads are expensive — **limit it explicitly with a `Semaphore`, not with a
thread count:**

```java
private final Semaphore gatewayPermits = new Semaphore(64);   // Topic 97 bulkhead

PaymentResult charge(Order order) throws InterruptedException {
    gatewayPermits.acquire();          // AQS -> unmounts cleanly. No carrier held.
    try {
        return gateway.charge(order);
    } finally {
        gatewayPermits.release();
    }
}
```

**Say this out loud, because it is the reframing the whole topic turns on:** with platform
threads, pool size was doing two jobs — bounding memory *and* bounding downstream
concurrency — and you could not separate them. Virtual threads make memory bounding
unnecessary and force you to state the downstream limit explicitly. **That is not extra
work; it is work you were doing badly by accident.**

**Observable:** thread count under load. Fixed pool: exactly 200, flat. Per-task executor:
tracks in-flight requests.

### Trap 2 — pinning on `synchronized` or a native frame

**Wrong approach.** Any `synchronized` region, or any JNI call, that contains a blocking
operation. It does not have to look dangerous:

```java
// A lazily-initialised cache. Utterly ordinary.
private Map<String, PricingRule> rules;
public synchronized PricingRule ruleFor(String sku) {
    if (rules == null) {
        rules = pricingClient.fetchAll();       // network call. Inside synchronized.
    }
    return rules.get(sku);
}

// Collections.synchronizedMap with a compute function that does I/O:
synchronizedMap.computeIfAbsent(sku, k -> catalogClient.lookup(k));

// A native payment SDK -- pins on EVERY JDK version:
nativePaymentSdk.charge(order);   // JNI, blocks in native code
```

**Exact symptom.** Throughput collapses. p99 goes to timeouts on endpoints that have
nothing to do with the affected code. **CPU utilisation is near zero.** Heap climbs. The
liveness probe times out and Kubernetes restarts the pod. Platform-thread dumps look
almost empty.

**Root cause.** The virtual thread cannot yield, so the carrier is held for the whole
blocking duration. With `availableProcessors()` carriers, that many concurrent pins is a
complete stall. See the Concurrency trace.

**Fix, in order of preference:**

1. **Move the blocking call out of the exclusion region.** This is almost always
   possible, and it was a design defect regardless of thread type.
2. **Replace `synchronized` with `ReentrantLock`.** AQS parks via `LockSupport.park`,
   which yields the continuation.
3. **For native frames, there is no fix at the lock level.** Isolate the native call on
   a small, explicitly-sized **platform** thread pool and have the virtual thread wait on
   the result. You are deliberately paying for platform threads exactly where you must.
4. **Upgrade the JDK** — for `synchronized` specifically, JDK 24+ removes most of this.
   Verify on your build; do not assume in either direction.

**Observable:** the JFR `jdk.VirtualThreadPinned` event (any version), or
`-Djdk.tracePinnedThreads=full` output on JDK 21–23. Plus the CPU-idle-while-slow
signature.

### Trap 3 — virtual threads for CPU-bound work

**Wrong approach.**

```java
// The nightly revenue rollup from Topic 100, "modernised":
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    for (var chunk : partition(orderLines, 50_000)) {
        executor.submit(() -> sumRevenue(chunk));    // pure arithmetic
    }
}
```

**Exact symptom.** Identical or *slightly worse* wall-clock time than the
`ForkJoinPool` version. CPU pegged at 100% — correctly. No error, no warning, nothing
anywhere says you gained nothing.

**Root cause.** There are `availableProcessors()` carriers. CPU-bound work never blocks,
so it never unmounts; each carrier runs one task to completion, exactly like a platform
thread pool of the same size. You added continuation setup, scheduling and the FIFO
queue for zero benefit. **Virtual threads raise the number of concurrent *waiters*, not
the amount of concurrent *computation*.**

**Fix.** CPU-bound, splittable work goes on a `ForkJoinPool` (Topic 100) or a fixed
platform pool sized near core count (Topic 90). The decision rule is one question:
**does this task spend most of its time waiting on something outside the JVM?** Yes →
virtual threads. No → platform threads sized to cores.

**Observable:** wall-clock time versus the `ForkJoinPool` baseline, measured with JMH
(Topic 77), plus CPU utilisation at 100% in both cases.

### Trap 4 — `ThreadLocal` on virtual threads, multiplying memory by thread count

**Wrong approach.** Keeping the request-context pattern that worked at 200 threads.

```java
public class TenantContext {
    private static final ThreadLocal<Tenant> CURRENT = new ThreadLocal<>();
    public static void set(Tenant t) { CURRENT.set(t); }
    public static Tenant get() { return CURRENT.get(); }
    public static void clear() { CURRENT.remove(); }
}
```

Plus `SecurityContextHolder` (Topic 56), plus MDC (Topic 120), plus whatever else the
filter chain populates.

**Exact symptom.** Heap use climbs roughly linearly with concurrent request count after
the switch. GC frequency rises. A class histogram shows tens of thousands of context
objects. Under a burst you get an `OutOfMemoryError` that looks nothing like a leak,
because nothing is retained past the request — there are simply a hundred times more
live requests.

**Root cause.** `ThreadLocal` cost is per-thread. You multiplied the thread count by
two orders of magnitude and the per-thread cost came along.

**Fix.**

1. **Measure first.** `jcmd <pid> GC.class_histogram | grep -i context`. If it is 40 MB,
   note it and move on. If it is 800 MB, act.
2. **Shrink what you store.** Store a tenant *id* (a `long`), not a hydrated `Tenant`
   with its config map.
3. **The structural fix is `ScopedValue`** — immutable, lexically scoped, and it does not
   allocate a per-thread map. **Topic 102.**
4. **Never** carry a `ThreadLocal` across a pooled boundary — that is Topic 79's leak,
   and it is a different, worse bug.

**Observable:** class histogram counts before and after, at matched arrival rates.

### Trap 5 — `CompletableFuture.allOf` leaking a still-running sibling

**Wrong approach.** The Topic 91 fan-out for the order-detail endpoint:

```java
public OrderDetail detail(OrderId id) {
    var product  = CompletableFuture.supplyAsync(() -> productClient.get(id),  executor);
    var stock    = CompletableFuture.supplyAsync(() -> inventoryClient.get(id), executor);
    var payment  = CompletableFuture.supplyAsync(() -> paymentClient.status(id), executor);

    CompletableFuture.allOf(product, stock, payment).join();   // <-- here

    return new OrderDetail(product.join(), stock.join(), payment.join());
}
```

**Exact symptom.** The payment gateway returns a 500 in 20 ms. `allOf().join()` throws
`CompletionException` almost immediately and the endpoint returns a 502 — good. But the
product call and the inventory call **keep running to completion**, holding connections,
consuming a downstream rate-limit budget, and possibly writing a cache entry for a
request that already failed. Under load you see: high downstream call volume that does
not match your served-request volume, and connection-pool pressure that no endpoint's
latency explains.

Worse: if `productClient.get` is the one that hangs for 30 seconds, `allOf` waits 30
seconds even though `payment` failed at 20 ms. You have taken the **maximum** latency of
all branches when you needed the **first failure**.

**Root cause.** `CompletableFuture` has **no cancellation semantics that propagate to
running work**. `allOf` is a completion combinator, not a lifetime scope. Calling
`cancel(true)` on a `CompletableFuture` does not interrupt the thread executing the
supplier — it only completes the future exceptionally. There is no parent-child
relationship anywhere in the API; three independent futures happen to be referenced by
one local variable.

**Fix.** This is exactly what a `StructuredTaskScope` fixes: subtask lifetimes bound to a
lexical block, first failure cancels the siblings, nothing outlives the scope.
**Topic 102 is the fix, and it is deliberately the next document.** The interim
mitigation on plain `CompletableFuture` is a timeout per branch
(`orTimeout(...)`) plus a cancellation token you check inside each supplier — which is
manual, easy to get wrong, and precisely the boilerplate structured concurrency removes.

**Observable:** count downstream calls (Micrometer, Topic 118) and compare with served
requests. A ratio above 1.0 during an error burst is orphaned work.

### Trap 6 — assuming the switch improved anything, without measuring

**Wrong approach.** Setting `spring.threads.virtual.enabled=true`, seeing the service
still work, and writing "migrated to virtual threads, improved scalability" in the PR
description.

**Exact symptom.** None. That is the trap. Six months later someone asks what it bought
and there is no answer.

**Root cause.** Thread count was probably never your constraint. `orderflow` at the
Topic 65 baseline is far more likely bounded by the connection pool, the database, or
the payment gateway.

**Fix.** Run the baseline at matched arrival rates, before and after, and record the
table. **If the numbers do not move, say so in the PR.** "No measurable change at our
current arrival rates; the constraint is the connection pool, see Topic 109. Adopting
virtual threads anyway because it removes the thread-count ceiling ahead of the Q3 traffic
increase, at no measured cost." That paragraph is a senior engineer's paragraph. "Improved
scalability" is not.

**Observable:** the before/after table. That is the entire deliverable of the hard
exercise.

---

## Hands-on proof

### Setup

```bash
java --version        # WRITE THIS DOWN. Everything below depends on it.
```

Everything here runs against `orderflow` under the Topic 65 harness, except Proof 1 and
Proof 2, which are standalone.

### Proof 1 — how many carriers do you actually have?

```java
public class SchedulerFacts {
    public static void main(String[] args) {
        System.out.println("availableProcessors = "
                + Runtime.getRuntime().availableProcessors());
        System.out.println("common pool parallelism = "
                + java.util.concurrent.ForkJoinPool.commonPool().getParallelism());

        // Read the carrier name out of a virtual thread's own toString().
        Thread.startVirtualThread(() ->
                System.out.println("virtual thread reports: " + Thread.currentThread()));

        try { Thread.sleep(200); } catch (InterruptedException ignored) { }
    }
}
```

```bash
java SchedulerFacts.java
docker run --rm --cpus=0.5  -v "$PWD":/w -w /w eclipse-temurin:25 java SchedulerFacts.java
docker run --rm --cpus=2    -v "$PWD":/w -w /w eclipse-temurin:25 java SchedulerFacts.java
```

**What to look for:** `availableProcessors()` in each container, and the carrier name in
the virtual thread's `toString()`.

| What you see | What it means |
|---|---|
| `availableProcessors` matches the `--cpus` value (rounded up) | Container awareness is working (Topic 82). Your carrier count is this number. |
| `availableProcessors = 1` under `--cpus=0.5` | **One carrier.** A single pinned virtual thread stops the entire JVM. Write this on the wall. |
| The virtual thread's `toString()` names a `ForkJoinPool` worker | Confirms the scheduler is an FJP and shows you the carrier-name prefix to grep for later. |
| `availableProcessors` equals the host core count inside a limited container | Your JDK or container runtime is not applying the quota. Fix this before anything else — every pool in the JVM is mis-sized. |

### Proof 2 — the thread-name probe, inside your own code

The single most useful instrument in this topic. Put it temporarily at the top of a
handler and at every point where you suspect a thread switch:

```java
log.info("stage={} thread={} virtual={} carrier-hint={}",
         stage,
         Thread.currentThread().getName(),
         Thread.currentThread().isVirtual(),
         Thread.currentThread().toString());
```

**What to look for:** `virtual=true` on your request path after the switch, and whether
`toString()`'s carrier hint changes across a blocking call.

| What you see | What it means |
|---|---|
| `virtual=true`, name like `tomcat-handler-N` or empty | Requests are on virtual threads. The switch took effect. |
| `virtual=false`, name `http-nio-8080-exec-N` | **The switch did not take effect on this path.** Check the property spelling, the active profile, and whether this path goes through a custom executor you never migrated. |
| `virtual=true` at stage A and `virtual=false` at stage B | You hand off to a platform executor somewhere between them. Find it — that is where your concurrency is capped. |
| Carrier hint changes across a DB call | Correct unmount/remount. The blocking call yielded. |
| Carrier hint identical across a long blocking call, repeatedly | Suspicious. Candidate pinning. Go to Proof 3. |

### Proof 3 — pinning diagnostics

**On JDK 21–23:**

```bash
java -Djdk.tracePinnedThreads=full -jar orderflow.jar
# or: -Djdk.tracePinnedThreads=short
```

The output, when it fires, has roughly this shape:

```
Thread[#42,ForkJoinPool-1-worker-3,5,CarrierThreads]
    java.base/java.lang.VirtualThread$VThreadContinuation.onPinned(...)
    ...
    com.orderflow.payments.PaymentService.submit(PaymentService.java:37) <== monitors:1
    ...
```

*(Illustration of the format, not captured output. The exact frames and layout vary by
JDK version — read yours.)*

The `<== monitors:1` marker is the payload: it names the frame that holds the monitor.

**On any JDK version, and the one to use in production — the JFR event:**

```bash
# Start a recording against a running service:
jcmd <pid> JFR.start name=pin settings=profile
# ... drive load for 60 s ...
jcmd <pid> JFR.dump name=pin filename=pin.jfr
jcmd <pid> JFR.stop name=pin

jfr summary pin.jfr | grep -i virtual
jfr print --events jdk.VirtualThreadPinned pin.jfr | head -80
```

| What you see | What it means |
|---|---|
| `jdk.VirtualThreadPinned` events with a stack trace naming your code | **Confirmed pinning.** The stack names the frame. Fix per Trap 2. |
| No pinned events at all, under load that reproduces the slowdown | On this JDK, this path does not pin. Your slowdown has another cause — go and look at the connection pool (Topic 109) and the downstream. |
| `jfr summary` does not list the event | Your recording settings may exclude it, or the event has a **duration threshold** below which it is not recorded. Check your settings file and lower the threshold if you need short pins; do not conclude "no pinning" from an event you did not enable. |
| Events present but all attributed to a third-party library | Still yours to fix. Isolate the call onto a platform executor (Trap 2, fix 3), or replace the library. |
| Pinned events on JDK 24/25 in a `synchronized` block | The 24+ change does not cover every case. This is exactly why you verify instead of assuming. |

> **Version note, stated plainly:** `-Djdk.tracePinnedThreads` exists on JDK 21–23 and was
> **removed in JDK 24** along with most of the pinning it diagnosed. Silence from it on a
> newer JDK means the property is gone, not that your code is clean. **The JFR event is
> the version-independent instrument. Use it.**

### Proof 4 — see your virtual threads at all

`jstack` and `top -H` show carriers, not virtual threads. To get the picture back:

```bash
# The virtual-thread-aware dump. JSON, one entry per virtual thread, grouped by
# the structured-concurrency scope where one exists.
jcmd <pid> Thread.dump_to_file -format=json /tmp/threads.json

# How many virtual threads exist right now:
grep -c '"isVirtual": true' /tmp/threads.json

# What are they all doing? Group by the top application frame:
grep -A3 '"isVirtual": true' /tmp/threads.json | grep 'com.orderflow' \
  | sort | uniq -c | sort -rn | head -20
```

| What you see | What it means |
|---|---|
| Virtual-thread count roughly matches in-flight requests | Healthy. Thread-per-request is doing what it says. |
| Thousands of virtual threads all in `HikariPool.getConnection` | **You moved the queue** (Topic 109). Threads are not the constraint; connections are. |
| Thousands of virtual threads all parked in the same application frame | Whatever that frame calls is your bottleneck. One command, one answer. |
| Virtual-thread count far above in-flight requests, and growing | A leak — something creates virtual threads that never finish. Look for a missing timeout. |
| The plain `jcmd Thread.print` shows a handful of carriers and looks idle | Expected and important: **the old tool now under-reports by design.** This is why you need the JSON dump. |

### Proof 5 — the memory cost, honestly

```java
public class ContinuationCost {
    public static void main(String[] args) throws Exception {
        int n = Integer.parseInt(args[0]);
        var latch = new java.util.concurrent.CountDownLatch(1);

        for (int i = 0; i < n; i++) {
            Thread.ofVirtual().start(() -> {
                try { latch.await(); } catch (InterruptedException ignored) { }
            });
        }
        System.gc();
        Thread.sleep(500);
        Runtime r = Runtime.getRuntime();
        System.out.printf("threads=%d heapUsedMB=%d%n",
                n, (r.totalMemory() - r.freeMemory()) / (1024 * 1024));
        latch.countDown();
    }
}
```

```bash
for n in 1000 10000 100000 500000; do java -Xmx2g ContinuationCost.java $n; done
```

**What to look for:** heap used as a function of thread count, and the slope.

| What you see | What it means |
|---|---|
| Heap grows roughly linearly with thread count, at a modest per-thread cost | The expected shape. Divide to get your per-continuation cost **on this JDK with this stack depth**. Do not quote anybody else's number, including one you read online. |
| The per-thread cost is much larger than you expected | Your parked threads have deep stacks. Continuation cost scales with live frame count. |
| `OutOfMemoryError` well before you expected | Same reason, plus whatever each task captured in its lambda. |
| A platform-thread version of the same program dies at a few thousand | The comparison that justifies the whole feature. Run it: `Thread.ofPlatform()` and watch `unable to create native thread` (Topic 98). |

### Proof 6 — is `synchronized` still a problem on YOUR JDK?

This is Proof 6 from Topic 94, and it is worth repeating here because it is the single
fact that changes this topic's advice:

```bash
# 1. Write down: java --version
# 2. Run the minimal Example 1 variant with the synchronized block, at parallelism=1.
# 3. On JDK 21-23: add -Djdk.tracePinnedThreads=full
# 4. On any JDK: take a JFR recording and check jdk.VirtualThreadPinned.
# 5. Record the answer in your notes as a FACT ABOUT YOUR BUILD, not a general truth.
```

**There is no general answer to quote.** That is the honest state of the ecosystem, and
saying so in an interview — "it depends on the JDK, and JDK 24 changed it; I'd check
before advising either way" — is a stronger answer than either confident claim.

---

## Failure drill

**Mandatory.** Do not read the analysis until you have produced the result yourself and
written down what you saw. **This drill is designed so that both possible outcomes are
informative.** Your JDK version determines which one you get.

### The scenario

Reproduce the Concurrency trace: N virtual threads pinned inside `synchronized` across a
slow payment-gateway call, with only `availableProcessors()` carriers, and watch every
other virtual thread in the JVM stop while the CPU sits idle.

### Part A — standalone, so you learn the instrument

`src/main/java/com/orderflow/lab/PinDrill.java`:

```java
package com.orderflow.lab;

import java.util.concurrent.CountDownLatch;
import java.util.concurrent.atomic.AtomicLong;

public class PinDrill {

    static final Object GATEWAY_MONITOR = new Object();
    static final java.util.concurrent.locks.ReentrantLock GATEWAY_LOCK =
            new java.util.concurrent.locks.ReentrantLock();

    /** Stands in for the 800 ms payment-gateway HTTP call. */
    static void slowGatewayCall() {
        try { Thread.sleep(800); } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }

    /** Mode: "synchronized" | "reentrantlock" | "nolock" */
    static String mode;

    static void submitPayment() {
        switch (mode) {
            case "synchronized" -> {
                synchronized (GATEWAY_MONITOR) { slowGatewayCall(); }
            }
            case "reentrantlock" -> {
                GATEWAY_LOCK.lock();
                try { slowGatewayCall(); } finally { GATEWAY_LOCK.unlock(); }
            }
            default -> slowGatewayCall();
        }
    }

    public static void main(String[] args) throws Exception {
        mode = args.length > 0 ? args[0] : "synchronized";
        int payments  = 8;        // the pinning threads
        int background = 2000;    // the innocent bystanders

        System.out.printf("pid=%d mode=%s processors=%d%n",
                ProcessHandle.current().pid(), mode,
                Runtime.getRuntime().availableProcessors());

        AtomicLong backgroundCompleted = new AtomicLong();
        CountDownLatch done = new CountDownLatch(payments + background);
        long start = System.currentTimeMillis();

        // Sampler: how much innocent work is getting through?
        Thread sampler = Thread.ofPlatform().daemon().start(() -> {
            while (true) {
                System.out.printf("t=%5dms  backgroundCompleted=%5d%n",
                        System.currentTimeMillis() - start, backgroundCompleted.get());
                try { Thread.sleep(500); } catch (InterruptedException e) { return; }
            }
        });

        // The pinning threads.
        for (int i = 0; i < payments; i++) {
            Thread.ofVirtual().name("payment-", i).start(() -> {
                submitPayment();
                done.countDown();
            });
        }

        Thread.sleep(50);   // let the payments grab their locks first

        // The innocent bystanders: tiny, CPU-only, no locks, no I/O.
        for (int i = 0; i < background; i++) {
            Thread.ofVirtual().name("catalogue-", i).start(() -> {
                long acc = 0;
                for (int j = 0; j < 10_000; j++) acc += j * 7919L;
                if (acc != 0) backgroundCompleted.incrementAndGet();
                done.countDown();
            });
        }

        done.await();
        System.out.printf("TOTAL_ELAPSED_MS=%d backgroundCompleted=%d%n",
                System.currentTimeMillis() - start, backgroundCompleted.get());
    }
}
```

### Commands

```bash
java --version   # RECORD THIS FIRST

# Constrain carriers so the effect is unmissable. Two carriers, eight pinning threads.
export VT="-Djdk.virtualThreadScheduler.parallelism=2 -Djdk.virtualThreadScheduler.maxPoolSize=2"

# Run 1 -- the defect.
java $VT -Djdk.tracePinnedThreads=full PinDrill.java synchronized 2>&1 | tee run-sync.log

# Run 2 -- the fix.
java $VT PinDrill.java reentrantlock 2>&1 | tee run-lock.log

# Run 3 -- the control: no lock at all.
java $VT PinDrill.java nolock 2>&1 | tee run-nolock.log

# During Run 1, in another terminal:
jcmd <pid> Thread.dump_to_file -format=json /tmp/pin.json
grep -c '"isVirtual": true' /tmp/pin.json
top -H -p <pid>          # Linux; Activity Monitor on macOS
```

And the version-independent confirmation, which you should do regardless of what Run 1
printed:

```bash
java $VT -XX:StartFlightRecording=duration=30s,filename=pin.jfr,settings=profile \
     PinDrill.java synchronized
jfr summary pin.jfr | grep -i -E 'VirtualThread|Pinned'
jfr print --events jdk.VirtualThreadPinned pin.jfr | head -60
```

### What to capture, before reading on

Write these down for all three runs:

1. `java --version`.
2. `TOTAL_ELAPSED_MS`.
3. The `backgroundCompleted` curve over time — does it rise smoothly, or does it flatline
   and then jump?
4. Any `jdk.tracePinnedThreads` output, and which frame carries `<== monitors:1`.
5. Whether `jfr print --events jdk.VirtualThreadPinned` produced any events.
6. CPU utilisation during the run.
7. The virtual thread count from the JSON dump.

**Now write one sentence predicting what Run 2 will change and why.** Then read on.

### How to read it — and this table covers BOTH JDK outcomes

| What you see | What it means |
|---|---|
| **Run 1:** `backgroundCompleted` flat near zero for ~800 ms at a time, then jumps; total elapsed several seconds; CPU near idle; pinned events present | **Your JDK pins on `synchronized`.** You have reproduced carrier starvation exactly as in the trace. 2,000 tiny CPU tasks — no locks, no I/O — could not run because two carriers were held by threads that were doing nothing but waiting on a socket. |
| **Run 1:** `backgroundCompleted` rises smoothly; total elapsed close to Run 3; **no pinned events in JFR** | **Your JDK does not pin on `synchronized` for this case — JDK 24+ fixed it.** This is a real and correct result, not a failed drill. The right reading is: "on this build, monitors no longer hold the carrier." Confirm it with the JFR event rather than concluding from the timing alone, then continue with Part B, which pins on **every** JDK. |
| **Run 1:** slow, but no pinned events anywhere | Suspect something else: your carriers are busy for another reason, or the JFR event has a duration threshold above your pin length. Lower the threshold and re-run before concluding. |
| **Run 2 (`reentrantlock`):** total elapsed roughly `8 x 800 ms` for the payments, but `backgroundCompleted` rises smoothly throughout | **The fix, working.** The lock still serialises the eight payments — that is the application's own logic and `ReentrantLock` does not change it. What changed is that the waiting threads **unmount**, so the carriers stay free and the 2,000 background tasks complete promptly. **Serialisation of the guarded work is intended. Starvation of everything else was not.** |
| **Run 3 (`nolock`):** all eight payments in roughly 800 ms total, background finishes promptly | The upper bound. This is what removing the lock entirely buys, and it is why the production fix in Example 2 moves the network call out of the critical section rather than just swapping the lock type. |
| Run 1 and Run 2 are identical | Either your JDK does not pin (see row 2), or `$VT` did not apply. Print `availableProcessors()` and check the parallelism actually took effect. |
| Background tasks complete but very slowly in all runs | Your background task may be too large. It should be microseconds. Reduce the inner loop. |

### Part B — the version-independent version of the same drill

Because `synchronized` pinning is a moving target, do this second half regardless of what
Part A showed. A **native frame** pins on every JDK, with no exceptions.

Replace `slowGatewayCall()` with a blocking call that goes through a native frame that
does not yield. The most portable version for a lab:

```java
    /**
     * A blocking file operation on a slow path. Many java.io file operations do NOT
     * unmount. VERIFY which ones on your JDK rather than trusting this comment --
     * that verification IS the exercise.
     */
    static void slowNativeCall() {
        try (var in = new java.io.FileInputStream("/dev/urandom")) {
            byte[] buf = new byte[1 << 20];
            for (int i = 0; i < 200; i++) in.read(buf);
        } catch (Exception e) { throw new RuntimeException(e); }
    }
```

**What to look for:** whether `jdk.VirtualThreadPinned` fires for this path on your JDK,
and whether the background tasks stall.

| What you see | What it means |
|---|---|
| Pinned events fire and background stalls | Confirmed: this path holds the carrier. This is the failure mode that **no JDK upgrade removes**, and it is why the "isolate native calls on a platform executor" fix in Trap 2 is permanent advice. |
| No pinned events and background runs freely | This particular path unmounts on your JDK. Good to know — record it. Try a genuine JNI library if you have one on the classpath. |
| Inconsistent between runs | File I/O paths differ by operating system and by JDK. This is exactly why "check, do not assume" is the rule. |

### Part C — on `orderflow`, under the Topic 65 baseline

1. Put the `synchronized`-across-the-gateway-call version of `PaymentService.submit`
   back into the service (a branch, not `main`).
2. Set `spring.threads.virtual.enabled=true`.
3. Constrain the container: `--cpus=2`.
4. Run the Topic 65 k6 scenario at the **recorded baseline arrival rate**.
5. Record p50/p95/p99 for **the catalogue read endpoint** — the one that touches no
   payment code at all.
6. Then deploy the Example 2 fix and re-run identically.

**What to look for:** the catalogue read p99, on an endpoint whose code did not change.

**How to read it:** if the catalogue p99 degrades badly in step 5 and recovers in step 6,
you have demonstrated the entire thesis of this document — that a lock held across a
network call in one corner of the service destroys the latency of unrelated endpoints,
and that the blast radius is set by the carrier count, not by the lock. If it does not
degrade, check the JDK version and go back to Part B.

### What the drill proves

Three things, in order of importance:

1. **Carrier starvation is a global failure with a local cause.** Two lines of code in a
   payments class stopped a catalogue endpoint. Nothing in the catalogue code, the
   catalogue metrics, or the database explains it. The only diagnostic that finds it
   quickly is the pinned event.

2. **CPU-idle-plus-collapsed-throughput is a distinct diagnostic category.** You met it
   in Topic 100 as blocked FJP workers and again here as pinned carriers. Same signature,
   one level up. When you see it, do not look at GC or locks — look for threads parked
   somewhere they cannot be replaced.

3. **The version-dependence is the lesson, not an inconvenience.** Whichever result you
   got, the durable skill is: run `java --version`, run the JFR event, and let the machine
   tell you. An engineer who says "synchronized pins virtual threads" in 2026 is quoting a
   fact with an expiry date. An engineer who says "it did on 21, JDK 24 changed it, here
   is how I'd check on your build" is doing the job.

---

## Measurement

### The standing rule

A naive `System.nanoTime()` loop is the **wrong** way to measure JVM performance. It
measures JIT warm-up, dead-code elimination, on-stack replacement, and whatever else the
machine was doing. **Topic 77 (JMH)** is where you learn to do this properly.

The wall-clock comparisons in the drill are fine because they produce an
order-of-magnitude difference in a *counter* (`backgroundCompleted`) and a stall you can
see with your eyes. They are not fine for "virtual threads gave us 12% more throughput."
That claim needs the Topic 65 harness, matched arrival rates, and multiple runs.

**And the specific trap for this topic:** virtual threads change *concurrency behaviour*,
which is invisible in a microbenchmark. JMH will happily tell you that creating a virtual
thread is faster than creating a platform thread. That is true and nearly useless. The
number that matters is throughput and tail latency **at a given arrival rate**, from the
Topic 65 harness.

### The instrument for each claim

| Claim you want to make | The instrument | What invalidates it |
|---|---|---|
| "We are on virtual threads" | `Thread.currentThread().isVirtual()` logged on the request path | A custom executor on the path that you never migrated |
| "Nothing pins" | JFR `jdk.VirtualThreadPinned` under load | An event threshold above your pin duration; load that does not exercise the path |
| "We have N carriers" | `availableProcessors()` logged at startup, inside the container | Measuring on your laptop instead of in the container |
| "Throughput improved" | Topic 65 harness, matched open-model arrival rate, before/after | Closed-model load (coordinated omission); different dataset; warm vs cold cache |
| "Memory cost is acceptable" | `jcmd GC.class_histogram` plus heap-used at matched concurrency | Measuring at idle |
| "The pool is now the bottleneck" | HikariCP metrics: `hikaricp.connections.pending`, acquire timing (Topic 109) | Not looking, and blaming threads |
| "Blocking work is off the event loop" | BlockHound (Topic 103) — for the WebFlux module, not for virtual threads | Assuming BlockHound covers virtual threads; it targets non-blocking schedulers |

**A note on BlockHound and virtual threads.** BlockHound detects blocking calls on
threads it has been told are non-blocking — Reactor's event loops and `parallel()`
scheduler. **Blocking on a virtual thread is correct and expected**, so BlockHound is not
the instrument here. It becomes relevant again in Topics 105, 106 and 108, where
`orderflow` has a WebFlux module whose event loops must never block. Do not install
BlockHound expecting it to find pinning; it will not, and pinning is a different question.

### Against the Topic 65 baseline — the rules

This is the spine deliverable. Follow it exactly or the numbers are not comparable.

1. **Same dataset.** 100k products, 1M orders, 5M order lines, same seed, same skew.
2. **Same JVM flags** — heap size, collector, container limits. Record them.
3. **Same arrival rate, open model.** Not "as fast as it can go". A closed-loop VU test
   hides latency degradation behind reduced offered load (coordinated omission), which is
   exactly the effect you are trying to measure.
4. **Same warm-up.** Discard the first N seconds identically in both runs. JIT and page
   cache both matter.
5. **Run each configuration at least three times** and report the spread, not one number.
6. **Sweep the arrival rate.** A single arrival rate cannot show you a ceiling. Run at
   0.5x, 1x, 2x and 4x the recorded baseline rate. **The interesting result is where the
   two curves diverge, if they diverge at all.**

Record this table and commit it next to the Topic 65 baseline:

| Endpoint | Arrival rate | Platform p50 | Platform p95 | Platform p99 | Virtual p50 | Virtual p95 | Virtual p99 | Errors P | Errors V |
|---|---|---|---|---|---|---|---|---|---|
| `GET /products` | 0.5x / 1x / 2x / 4x | | | | | | | | |
| `GET /orders/{id}` | 0.5x / 1x / 2x / 4x | | | | | | | | |
| `POST /orders` | 0.5x / 1x / 2x / 4x | | | | | | | | |

Plus, for each run: peak heap used, peak thread count (platform and virtual separately),
CPU utilisation, and `hikaricp.connections.pending` peak.

**How to read it:**

- **p50 identical, p99 better at 2x and 4x** — the classic and expected win. Thread count
  was a ceiling above 1x. Report the arrival rate where the curves diverge; that is the
  headroom you bought.
- **Identical everywhere** — thread count was never the constraint. This is a completely
  legitimate finding and the most likely one for `orderflow`. **Write it down as the
  finding.** Then go to Topic 109 and find the real constraint.
- **Virtual worse at 4x** — look for pinning, then for `hikaricp.connections.pending`. You
  probably removed a queue that was protecting something.
- **Errors change shape** — fewer connection refusals, more pool timeouts — is the "moved
  the queue" signature and belongs in the writeup verbatim.

### What to graph permanently in production

```java
@Bean
MeterBinder virtualThreadMetrics() {
    return registry -> {
        Gauge.builder("orderflow.jvm.available.processors",
                Runtime.getRuntime(), Runtime::availableProcessors).register(registry);
        // Thread counts by kind, from ThreadMXBean plus your own counter for
        // virtual threads (the MXBean does not count them).
    };
}
```

And enable the JFR pinned event continuously in production with a conservative recording
(Topics 78, 81). It is cheap, it fires rarely, and when it fires it is the answer.

---

## Practice exercises

### 1 — easy: build the facts table for your own environment

Produce a one-page table of facts about **your** setup, all from commands you ran:

1. `java --version`.
2. `availableProcessors()` on your laptop, under `--cpus=2`, and under `--cpus=0.5`.
3. The default carrier count, read out of a virtual thread's `toString()`.
4. Whether `synchronized` pins on your JDK, proven with the JFR event and not with a
   timing.
5. Approximate heap cost per parked virtual thread at a shallow stack depth, from Proof 5.
6. What `jcmd Thread.print` shows versus `jcmd Thread.dump_to_file -format=json` when
   10,000 virtual threads are parked.

**The deliverable is the table, with the command that produced each row next to it.** Any
row you cannot attribute to a command you ran does not go in the table.

### 2 — medium: the audit (combines Topics 01–100)

Audit `orderflow` for virtual-thread readiness and produce a written report. For each
finding: the file and line, the topic it comes from, the symptom it would produce, and
the fix.

Find at least:

1. Every `synchronized` block or method that contains a blocking call (Topic 85). For
   each: does it need mutual exclusion at all, and can the blocking call move out?
2. Every `ThreadLocal` on the request path (Topics 56, 79, 120), with the retained-size
   arithmetic at 20,000 concurrent requests.
3. Every fixed-size executor still in the codebase (Topic 90). For each: was the size
   bounding *memory* or bounding *downstream concurrency*? Only the second is still a
   real requirement.
4. Every `CompletableFuture.*Async` call with no executor argument (Topics 91, 100) —
   those go to the common pool, which is **not** the virtual-thread scheduler.
5. The `allOf` fan-out (Topic 91) and what it leaks when one branch fails (Trap 5).
6. Every place a `@Transactional` method makes an external call (Topic 55) — virtual
   threads make it *easier* to have thousands of these in flight, each holding a
   connection. **The Topic 55 bug gets worse, not better.**
7. HikariCP `maximum-pool-size` and the arithmetic of what happens when 20,000 virtual
   threads want a connection (Topic 109).

**The report should end with a one-paragraph recommendation and a named risk.** That
paragraph is a rehearsal for Topic 107.

### 3 — hard: the full Topic 65 baseline re-run

**This is the spine deliverable and the main assessment for this topic.**

Produce a decision-quality before/after comparison of `orderflow` on platform threads
versus virtual threads.

**Method:**

1. Confirm you can reproduce the Topic 65 baseline within ±10%. If you cannot, stop —
   the gate rule says nothing downstream is valid.
2. Record `java --version`, container CPU and memory limits, JVM flags, collector,
   HikariCP pool size, and the k6 scenario file hash. All of it goes in the report.
3. Run the full scenario mix at **four arrival rates** (0.5x, 1x, 2x, 4x of baseline), on
   platform threads. Three repetitions each.
4. Flip `spring.threads.virtual.enabled=true`. Change **nothing else**. Repeat step 3.
5. For each virtual-thread run, capture a JFR recording and check
   `jdk.VirtualThreadPinned`. **A run with pinning events is not a valid comparison** —
   fix the pinning and re-run.
6. Capture, for every run: p50/p95/p99/p999 per endpoint, throughput, error rate and
   error *shape*, peak heap, CPU utilisation, virtual-thread count, and
   `hikaricp.connections.pending`.
7. Then do a **third** configuration: virtual threads plus a HikariCP pool sized from the
   Topic 109 arithmetic. This is the run that tells you whether the pool was the real
   constraint.

**Deliverable — a document of at most two pages containing:**

- The table from the Measurement section, filled in.
- One graph: throughput versus arrival rate, three curves.
- One graph: p99 versus arrival rate, three curves.
- **A statement of what the constraint actually is**, defended by the data.
- A recommendation with a named risk and a rollback trigger.
- An explicit "what we did not measure" section.

**The grading criterion is honesty, not improvement.** A report that says "no measurable
change; the constraint is the connection pool; we adopted virtual threads anyway because
it removes a ceiling ahead of Q3 at no measured cost, and here is the rollback trigger" is
a **better** report than one claiming a speedup it cannot defend. **Keep this document.
Topic 107 consumes it.**

---

## Interview questions

### Q1 — "What is a virtual thread, and how is it different from a platform thread?"

**MID-LEVEL.** "It's a lightweight thread managed by the JVM instead of the OS. You can
create millions of them, so you don't need thread pools any more. They're good for I/O."

**SENIOR.** "A virtual thread is a `java.lang.Thread` whose stack lives on the heap as a
continuation rather than in a fixed OS thread stack. To run, it mounts onto a carrier — a
platform thread from a dedicated `ForkJoinPool` in FIFO mode, sized by default to
`availableProcessors()`. When it hits a blocking call in the JDK, it unmounts: the live
frames are copied onto the heap and the carrier picks up another virtual thread.

The practical consequence is that thread-per-request becomes viable again at concurrency
levels that used to require an event loop, without changing the programming model —
blocking code, whole stack traces, working debuggers, no function colouring.

What it does *not* change: there is no additional CPU, the database doesn't get faster,
and there's no backpressure. If the real constraint is a 20-connection pool, virtual
threads just move the queue from the servlet container into `getConnection`. And they must
not be pooled — creation is roughly an allocation, so pooling reintroduces the exact
ceiling you removed."

**What separates them.** The mid answer describes the feature. The senior answer names the
mechanism (continuation, mount, unmount, carrier, FJP), names the limits, and pre-empts
the most common disappointment. The phrase "moved the queue" is worth a lot on its own.

**Follow-up:** *"You said no backpressure. What would you add?"* → A `Semaphore` as a
bulkhead sized to the downstream's real capacity, plus a bounded queue and a rejection
policy at the ingress. And note that bounding concurrency is now an **explicit** decision
rather than an accident of pool size — which is an improvement in clarity, not a
regression.

### Q2 — "What is pinning, and how would you find it in a running service?"

**MID-LEVEL.** "It's when a virtual thread gets stuck to a carrier thread. It happens with
`synchronized`. You should use `ReentrantLock` instead."

**SENIOR.** "Pinning is when a virtual thread cannot unmount, so its blocking call holds
the carrier for the full duration. Two causes pin on every JDK: a native frame on the
stack, and blocking inside a class initializer. A third — being inside a `synchronized`
block — pinned on JDK 21 through 23, and JDK 24 shipped work removing it for most cases.
So my first question is always which JDK you're on, and I'd verify rather than assume in
either direction.

The reason `synchronized` was the hard case and `ReentrantLock` was not: monitor ownership
is tracked in the runtime against the platform thread, whereas AQS is ordinary Java code
that parks through `LockSupport`, which the JDK taught to yield a continuation.

To find it: on 21–23, `-Djdk.tracePinnedThreads=full` gives you the stack with a
`monitors:` marker on the offending frame. On any version — and the only one I'd run in
production — the JFR event `jdk.VirtualThreadPinned`. Watch the event's duration threshold;
short pins may not be recorded by default, and absence of events isn't absence of pinning
until you've checked that.

The signature to recognise without any tooling is collapsed throughput with **idle CPU**.
That rules out contention and GC immediately, and it means threads are parked somewhere
they can't be replaced."

**What separates them.** Naming all three pinning causes rather than just `synchronized`;
refusing to assert the JDK behaviour without checking; naming the diagnostic *and* its
failure mode (the threshold); and naming the CPU-idle signature.

**Follow-up:** *"Your `synchronized` block is inside a third-party library you can't
change."* → Isolate that call onto a small, explicitly-sized platform-thread executor and
have the virtual thread wait on the result. You pay for platform threads exactly where you
must, with a bound you chose. That is also the only available answer for a JNI frame,
which no JDK upgrade fixes.

### Q3 — "We switched to virtual threads and nothing got faster. What happened?"

**MID-LEVEL.** "Maybe there's pinning somewhere, or the code is CPU-bound so virtual
threads don't help."

**SENIOR.** "Most likely nothing is wrong — thread count was never the constraint. That's
the common case, and it's worth saying before reaching for a bug.

I'd work through it in order. First, confirm the switch actually took effect on the path
under test: log `Thread.currentThread().isVirtual()` in the handler. A custom executor
anywhere on the path silently keeps you on platform threads.

Second, check for pinning with the JFR event — if we pinned, we'd be *worse*, not
unchanged, so 'unchanged' actually argues against pinning.

Third, and most likely: find the real constraint. Take a virtual-thread-aware dump —
`jcmd Thread.dump_to_file -format=json` — and see what those threads are all doing. If
they're all in `HikariPool.getConnection`, the pool is the constraint and we moved the
queue rather than removing it. If they're all waiting on the payment gateway, the gateway
is the constraint and no amount of local concurrency changes that.

Fourth: how was the load generated? If it was a fixed number of virtual users in a closed
loop, the test itself limits concurrency, and you cannot observe a concurrency ceiling with
a test that imposes its own. It needs to be an open-model arrival rate.

And I'd want the honest sentence in the writeup: 'no measurable change at current arrival
rates; here is the constraint we found; we're keeping the change because it removes a
ceiling ahead of projected growth at no measured cost.'"

**What separates them.** Starting from "probably nothing is wrong", naming the load-
generator methodology error, and using the argument that pinning would make it worse
rather than neutral. The senior answer also produces a *finding* rather than a fix.

**Follow-up:** *"How would you size the connection pool then?"* → From Little's Law
against the database's actual concurrency capability, not from the thread count. Topic 109,
and the answer is usually much smaller than people expect.

### Q4 — "Should we use `Executors.newFixedThreadPool(1000, virtualFactory)`?"

**MID-LEVEL.** "That gives you a pool of a thousand virtual threads, which is more than
200, so it should scale better."

**SENIOR.** "No — that's a contradiction. `newFixedThreadPool` is a `ThreadPoolExecutor`:
it creates N threads, reuses them, and feeds them from a shared queue. Giving it a virtual
thread factory changes what kind of thread it creates and nothing else. You've capped
concurrency at 1000 and added continuation overhead for no benefit.

Pools exist to amortise thread creation. Virtual thread creation is roughly an object
allocation, so there is nothing to amortise. Pooling also reintroduces the Topic 79 leak
shape — a `ThreadLocal` nobody removed is retained for the life of a now-immortal thread.

The right construct is `Executors.newVirtualThreadPerTaskExecutor()`, which creates one new
virtual thread per submitted task and, being `AutoCloseable`, waits for all of them on
close.

If the goal was to limit concurrency against a fragile downstream, that's a legitimate
requirement and the answer is a `Semaphore` sized to that downstream's real capacity —
which unmounts cleanly, unlike a thread-count cap. The important shift is that pool size
used to be doing two jobs at once: bounding memory and bounding downstream load. Virtual
threads make the first unnecessary and force you to state the second explicitly."

**What separates them.** Recognising it as a category error rather than a tuning question,
and articulating the "pool size was doing two jobs" reframing, which is the real insight.

**Follow-up:** *"Where would you put the `Semaphore`?"* → Around the downstream call, not
around the whole request, so a request waiting for a permit still holds no carrier and
still releases its other resources. And I'd expose the permit count as a metric so the
bulkhead is visible when it saturates.

### Q5 — "Does this mean we should rewrite our WebFlux service?"

**MID-LEVEL.** "Probably — virtual threads give you the same scalability with much simpler
code."

**SENIOR.** "For the thread-economy motivation, largely yes: virtual threads give you the
concurrency with readable stack traces, working debuggers, and no function colouring.

But two things reactive gives you that Loom does not. Backpressure — `request(n)` is a
demand signal travelling upstream, and nothing in Loom has an equivalent. And stream
composition — operators over an asynchronous sequence with a defined contract.

So the technical criterion is the workload. High-concurrency request/response with
blocking I/O: virtual threads, easily. Streaming with a genuinely slow consumer that must
signal a fast producer: reactive still wins.

And for an *existing* WebFlux codebase, the migration cost is usually the deciding factor,
not the technical merits. A working reactive service is not a bug. I'd want a measured
reason, not an aesthetic one.

What I'd actually propose for a mixed service is a boundary: virtual threads for the
request/response core, and a small reactive module for the genuinely streaming ingestion
path — benchmarked separately so the decision has data behind it."

**What separates them.** Conceding the thread-economy point immediately, then naming
backpressure and composition precisely, then treating migration cost as first-class. This
is Topic 107's centrepiece question and you should be able to give this answer cold.

**Follow-up:** *"Give me the one metric that would change your mind."* → Whether the
ingestion path has a producer that can outrun the consumer and cannot be slowed. If yes,
that path needs demand propagation and virtual threads will not supply it. If the producer
is a synchronous HTTP caller — which is already backpressured by the fact that it is
waiting — the argument for reactive is much weaker.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. Node moved the continuation to the heap and made you write `await`. Loom moved the
   continuation to the heap and made the runtime do it. Name one concrete thing that Node's
   approach gives you that Loom's does not, and one that Loom gives you that Node's does
   not. Be specific — "simpler" is not an answer.

2. A virtual thread unmounts at a blocking call. Construct a workload where this makes
   things measurably **worse** than platform threads, and say what property of the workload
   causes it.

3. The virtual-thread scheduler uses FIFO; the common pool uses LIFO-local. Given only
   that, what can you infer about the workloads each was designed for? Now argue that
   virtual threads should have used LIFO, and say what it would cost.

4. Pinning is defined as "cannot unmount". Design an alternative implementation of
   `synchronized` that would never pin. Then explain why it took the JDK until version 24
   to ship something along those lines — what makes monitors harder than `ReentrantLock`?

5. `ThreadLocal` on 200 platform threads costs 200 contexts; on 50,000 virtual threads it
   costs 50,000. Nothing leaked. Explain why this is nevertheless a bug worth fixing, and
   distinguish it precisely from the Topic 79 `ThreadLocal` leak.

6. You have a service where every request does exactly one 5 ms database query and nothing
   else, against a pool of 20 connections. Predict the effect of switching to virtual
   threads on p50, p99 and throughput, and justify each prediction from the mechanism.
   Then say what measurement would falsify you.

7. Virtual threads must not be pooled. Construct the strongest possible argument *for*
   pooling them. Then rebut it. What would have to be true about the JVM for the argument
   to win?

---

## Quick reference card

### The mechanism in five lines

```
The stack is a heap object (a continuation).
Mount   = copy frames onto a carrier and jump to the saved IP.
Unmount = copy live frames off the carrier onto the heap; carrier takes other work.
Carriers = a dedicated ForkJoinPool in FIFO mode, sized availableProcessors().
Pinned  = cannot unmount -> holds the carrier -> N pins stops the JVM, CPU idle.
```

### Creation

```java
Thread.startVirtualThread(runnable);
Thread.ofVirtual().name("orderflow-vt-", 0).start(runnable);
Thread.ofVirtual().factory();                        // a ThreadFactory
Executors.newVirtualThreadPerTaskExecutor();         // NOT a pool. One thread per task.

// Spring Boot:
spring.threads.virtual.enabled=true
```

### Never

```java
Executors.newFixedThreadPool(n, Thread.ofVirtual().factory());   // caps concurrency
Executors.newCachedThreadPool(Thread.ofVirtual().factory());     // pointless
// synchronized around any blocking call  -> version-dependent pinning
// ThreadLocal on the request path        -> multiplied by thread count
// virtual threads for CPU-bound work     -> zero gain, extra overhead
```

### Pins on every JDK

```
- a native (JNI) frame on the stack
- blocking inside a class initializer (<clinit>)
- some file-I/O paths -- verify on YOUR JDK, do not assume
```

### Pins version-dependently

```
- synchronized: pins on 21-23; JDK 24 removed most of it.
  VERIFY on your build. Never assert either way from memory.
```

### Diagnostics

```bash
java --version                                    # ALWAYS FIRST

# JDK 21-23 only (removed in 24):
-Djdk.tracePinnedThreads=full

# Any version -- the real instrument:
jcmd <pid> JFR.start name=pin settings=profile
jcmd <pid> JFR.dump  name=pin filename=pin.jfr
jfr print --events jdk.VirtualThreadPinned pin.jfr

# See virtual threads at all (jstack and top -H show only carriers):
jcmd <pid> Thread.dump_to_file -format=json /tmp/threads.json
grep -c '"isVirtual": true' /tmp/threads.json

# In code:
Thread.currentThread().isVirtual()
```

### Tuning knobs (verify against your JDK; not stable API)

```bash
-Djdk.virtualThreadScheduler.parallelism=N
-Djdk.virtualThreadScheduler.maxPoolSize=N
```

### The signature

```
Throughput collapsed + latency exploded + CPU IDLE + no lock contention + DB healthy
  -> threads are parked where they cannot be replaced.
  -> Topic 100 if it is FJP workers. THIS TOPIC if it is carriers.
```

### Gotchas checklist

- [ ] Never pool virtual threads. `newVirtualThreadPerTaskExecutor` only.
- [ ] Limit downstream concurrency with a `Semaphore`, not a thread count.
- [ ] Audit every `synchronized` that contains a blocking call.
- [ ] Audit every `ThreadLocal` on the request path; plan for `ScopedValue` (T102).
- [ ] Name your virtual threads. An unnamed one in a log is a wasted diagnostic.
- [ ] Log `availableProcessors()` at startup, inside the container.
- [ ] Check HikariCP pool size before claiming any improvement (T109).
- [ ] Enable the JFR pinned event in production; it is cheap and it is the answer.
- [ ] `jstack` under-reports by design now. Use `Thread.dump_to_file -format=json`.
- [ ] Measure at matched, open-model arrival rates against the Topic 65 baseline.

---

## When would I use this at work?

**1. Diagnosing "the service is slow and the CPU is idle" on a Java 21+ service.**

Ninety seconds once you know the shape. Collapsed throughput with idle CPU is not
contention and not GC — it is threads parked where they cannot be replaced. Take the JSON
thread dump, see what the virtual threads are all waiting on, and check the JFR pinned
event. You will either find a pin, find the connection pool, or rule the whole class out
and move on. Most engineers spend an hour on GC logs first.

**2. Reviewing a PR that touches concurrency on a virtual-thread service.**

Three questions, all mechanical, all answerable in the review. Does this `synchronized`
block contain a blocking call? Is this executor a fixed pool that now caps concurrency for
no reason? Does this new `ThreadLocal` get multiplied by the request count? Those three
comments cost thirty seconds each and prevent the three most expensive bugs in this
document.

**3. Deciding whether to adopt virtual threads at all — and being honest about the answer.**

The valuable output is usually "we measured, thread count was not the constraint, the pool
is, and here is what we're doing about that instead." Presenting a negative result with the
data behind it is a senior move that builds far more credibility than a claimed speedup
nobody can reproduce. It is also the exact shape of the Topic 107 deliverable and of the
Phase 12 design-document work.

---

## Connected topics

**Prerequisites:**

- **56 — Spring Security filter chain:** `SecurityContextHolder` is `ThreadLocal`-backed.
  That fact is now multiplied by your virtual-thread count.
- **68 — Heap, generations, TLAB:** continuations and stack chunks live here. The memory
  cost of a million parked threads is a heap cost, and the collector must trace it.
- **73 — Safepoints:** virtual threads change what a thread dump means and how threads
  reach a safepoint; the interaction is worth understanding before you read one.
- **79 — Memory leaks:** the `ThreadLocal`-on-a-pooled-thread leak. Distinguish it
  precisely from this topic's multiplied-cost problem — same API, different failure.
- **84 — Threads vs the event loop:** where the "a parked OS thread is genuinely gone"
  intuition came from. Virtual threads are the reply to it.
- **85 — `synchronized` and monitors:** the mechanism behind pinning. Monitor ownership
  lives in the runtime, which is why it was the hard case.
- **90 — Executors and pool sizing:** the model this topic partially retires. Little's Law
  still applies — to the downstream, not to the threads.
- **94 — `ReentrantLock`:** the fix for pinning, and the explanation of *why* it is the
  fix: AQS is Java code that parks through `LockSupport`.
- **100 — `ForkJoinPool`:** the scheduler is one of these, in FIFO mode, sized from
  `availableProcessors()`. "A parked worker cannot steal" is "a pinned carrier cannot be
  reused", one level up.

**Also relevant:**

- **82 — JVM tuning in containers:** `availableProcessors()` under a cgroup quota sets your
  carrier count. `--cpus=0.5` means one carrier means one pin is an outage.
- **91 — `CompletableFuture`:** the `allOf` orphan-sibling problem (Trap 5), which
  Topic 102 fixes properly.
- **97 — Coordination primitives:** `Semaphore` as the bulkhead that replaces pool size as
  your concurrency limit.
- **98 — Bug taxonomy:** collapsed throughput with idle CPU is a named category, and this
  topic supplies its most modern cause.
- **65 — The load baseline:** the numbers you compare against. Without it, none of this is
  measurable.

**This unlocks:**

- **102 — Structured concurrency and `ScopedValue`:** the fix for Trap 5's orphaned
  siblings and for Trap 4's multiplied `ThreadLocal`. Read it next; it is the other half
  of this document.
- **105 — Backpressure:** the thing virtual threads do **not** give you, and the reason
  reactive is not obsolete.
- **106 — WebFlux vs MVC:** the alternative threading model, benchmarked against the
  numbers you just produced.
- **107 — The Loom-vs-reactive decision:** consumes the hard exercise's report directly.
  Do not skip the exercise; Topic 107's deliverable depends on it.
- **109 — HikariCP:** where the queue moved. Almost certainly your real constraint.
- **119 — Tracing across virtual threads:** context propagation when there are 50,000
  threads and no pool to attach to.
- **120 — MDC:** the logging half of the same context problem.

---

*Java baseline 21. Virtual threads were finalised in Java 21 (JEP 444) and the API in this
document is stable. Two things in it are explicitly version-dependent and are flagged in
place: `synchronized` pinning, which JDK 24 substantially removed, and the
`jdk.tracePinnedThreads` system property, which JDK 24 removed along with it. The JFR
`jdk.VirtualThreadPinned` event is the version-independent instrument and is what you
should build on. The `jdk.virtualThreadScheduler.*` properties are documented tuning knobs
rather than stable API — verify them against your JDK rather than trusting a name from this
document. Run `java --version` before you trust any claim here about pinning, including
mine.*
