# 84 — Threads and Lifecycle vs the Node Event Loop

## Phase: 9 — Concurrency
## Category: DIFFERENTIATOR
## Java baseline: 21  |  Notes features from: 21, 25
## Project spine: `orderflow` already serves thread-per-request under Tomcat. This topic makes that fact *visible*: every singleton bean field is shared mutable state reachable from every request thread simultaneously. Nothing new is added to the service; what is added is your ability to see the hazard that has been there since Topic 44.

---

## Mechanical statement

Read this three times. Every other section is an elaboration of it.

> **A Java thread is an OS thread, scheduled preemptively by the kernel. The
> scheduler can suspend it between any two bytecodes — including between reading a
> counter and writing it back.**
>
> There is no point in your Java program that is guaranteed to run to completion.
> Not a method. Not a statement. Not `stock = stock - 1`.
>
> Every thread in one JVM shares one heap. Any object reachable from two threads is a
> correctness question, whether or not you thought of it as shared.

That is the whole phase in three sentences. Topics 85 through 102 exist because those
three sentences are true.

---

## The bridge from what you know

### NO TYPESCRIPT ANALOGUE.

This block is not a formality. Read it slowly, because the thing it removes is the
foundation of every concurrency intuition you currently have.

**Your model today:** JavaScript has one thread running one call stack. When a
callback starts, it runs to the end. Nothing else runs in between. The event loop
does not interrupt you. It picks up the next task only when your stack is empty.

That property has a name: **run-to-completion**.

Run-to-completion is why you have never written a lock. It is why `count++` is safe
in Node. It is why you can read a field, do some arithmetic, and write the field back
without a thought, and it is *always* correct. It is why "data race" is a word you
have read but never debugged.

**Java has no such property.** None. At any point:

```java
inventory.stock = inventory.stock - 1;
```

the operating system may take the CPU away from your thread and give it to another
thread which runs the same line. When your thread resumes, it writes back a value it
computed before the other thread's write existed. One decrement is lost.

Not "rarely". Not "under exotic conditions". This is the normal behaviour of a
preemptively scheduled system, and it is what your load test at Topic 65 has been
generating conditions for all along.

**Do not carry the run-to-completion assumption into Java.** It is not a small
mismatch you can patch with care. It is the absence of the guarantee your entire
mental model rests on. Everything in Phase 9 is the machinery Java gives you to
*manually re-create* small islands of the guarantee you used to get for free.

### `worker_threads` — PARTIAL, and the gap is the whole phase

You may reach for `worker_threads` as the analogue. It is a real analogue, and it is
partial in exactly one way that matters more than everything it gets right.

| Node `worker_threads` | Java threads | Verdict |
|---|---|---|
| A worker is a real OS thread | A Java thread is a real OS thread | **HONEST** — same underlying object |
| Each worker gets its **own V8 isolate and its own heap** | All threads share **one heap** | **NO ANALOGUE** — and this is the entire difference |
| Communication is `postMessage` — structured-clone **copying** | Communication is by writing to a shared object | **NO ANALOGUE** |
| A worker cannot observe another worker's object mid-mutation, because it cannot observe another worker's objects at all | Any thread can observe any reachable object at any instant, including mid-mutation | **NO ANALOGUE** |
| `SharedArrayBuffer` is the deliberate opt-in to sharing | Sharing is the default; isolation is what you must build | **INVERTED** |

Say this out loud once, because it is the load-bearing sentence of Phase 9:

> **Node workers share nothing by default and share by explicit opt-in. Java threads
> share everything by default and isolate by explicit effort. That single inversion
> generates every remaining topic in this phase.**

Your `worker_threads` experience gives you the right instinct for *scheduling* — many
things running truly at once, on many cores. It gives you no instinct at all for
*shared mutable state*, because Node never let you have any.

### The one honest analogue in the whole phase — and it is not in this topic

There is exactly one place where JavaScript has a genuine memory model:
`SharedArrayBuffer` plus `Atomics`. `Atomics.store`, `Atomics.load`, `Atomics.wait`,
`Atomics.notify` exist because once two workers share raw memory, JavaScript needs
the same answers Java needs: which write does a read see, and in what order.

Most JavaScript engineers have never touched it. If you have, that experience
transfers almost exactly, and Topics 86 and 87 will feel like recognition rather than
learning. If you have not, do not go and learn it first — you will meet the same ideas
here, in the language you are actually being interviewed on.

For *this* topic, it does not help. `SharedArrayBuffer` is about visibility;
Topic 84 is about **preemption**, and JavaScript has no preemption at all.

### What does transfer

| You know | Java | Verdict |
|---|---|---|
| A worker has a lifecycle: created, running, exited | `Thread` has states: `NEW`, `RUNNABLE`, `BLOCKED`, `WAITING`, `TIMED_WAITING`, `TERMINATED` | **PARTIAL** — Java's set is finer, and `BLOCKED` vs `WAITING` is a real diagnostic distinction |
| `worker.terminate()` kills a worker | `Thread.stop()` was removed because it cannot be done safely | **NO ANALOGUE** — see Trap 2 |
| An unhandled rejection can kill the process | An uncaught exception kills **only that thread**, silently | **NO ANALOGUE** — and this is a common production surprise |
| `AsyncLocalStorage` follows the async context | `ThreadLocal` is bound to a thread and does not follow work across threads | **PARTIAL** — Topic 79 for the leak, Topic 102 for `ScopedValue` |
| Node's process stays alive while a handle is open | The JVM stays alive while any **non-daemon** thread is alive | **PARTIAL** — same shape, different rule |
| `os.cpus().length` and `cluster` — scale by processes | `Runtime.availableProcessors()` — scale by threads inside one process | **PARTIAL** — Java scales *up*, Node scales *out* |

---

## What is this?

### A thread, precisely

A **thread** is an independent path of execution through your program. It has:

- its own **program counter** — which instruction it is on,
- its own **stack** — its chain of method frames, local variables and operand stacks,
- **shared access to the heap** — every object, every static field, every array,
- an identity in the OS scheduler, which decides when it runs.

Threads in one JVM share: the heap, the metaspace, loaded classes, static fields, and
open file descriptors. They do not share stacks. That split — private stack, shared
heap — is the single most important structural fact in this phase.

**Local variables of primitive type are safe.** They live on the thread's own stack.
No other thread can reach them.

**Anything reachable through a reference is not safe by default**, because the object
lives on the shared heap, and the reference may have been handed to another thread.

### Creating one

There are four shapes you will see. Three of them you should not write in
application code, and you need to recognise all four.

```java
// 1. Subclass Thread. Legacy. Couples your logic to the threading mechanism.
class Decrementer extends Thread {
    @Override public void run() { /* work */ }
}
new Decrementer().start();

// 2. Runnable + Thread. The classic form. Still what you see in older code.
Thread t = new Thread(() -> { /* work */ }, "inventory-decrementer");
t.start();

// 3. Java 21 factory / builder. Explicit, names threads properly, and is the
//    same API that produces virtual threads (Topic 101).
Thread t = Thread.ofPlatform()
                 .name("inventory-decrementer-", 0)   // prefix + starting counter
                 .daemon(false)
                 .start(() -> { /* work */ });

// 4. What you will ACTUALLY use in orderflow: an executor (Topic 90).
ExecutorService pool = Executors.newFixedThreadPool(8);
pool.submit(() -> { /* work */ });
```

You will write form 4 in production and form 3 in labs. Forms 1 and 2 are here so you
can read a 2011 codebase without stopping.

### `start()` versus `run()` — the first trap

```java
Thread t = new Thread(() -> System.out.println(Thread.currentThread().getName()));

t.start();   // creates an OS thread; prints "Thread-0"
t.run();     // NO new thread; prints "main". It is a plain method call.
```

`run()` is an ordinary method on an ordinary object. Calling it does exactly what
calling any method does: it runs on your current thread. `start()` is the native call
that asks the OS for a thread and arranges for `run()` to be its entry point.

This compiles, runs, produces no warning, and gives you a completely sequential
program that you believe is concurrent. It is Trap 1 below.

### The six states, and what each one means diagnostically

`Thread.State` has exactly six values. You will read these in thread dumps for the
rest of your career, so learn them as **diagnoses**, not as vocabulary.

| State | Meaning | What it tells you in a dump |
|---|---|---|
| `NEW` | Constructed, `start()` not yet called | Almost never seen in a dump |
| `RUNNABLE` | Eligible to run. **May or may not be on a CPU right now** | Either doing work, or blocked in a syscall the JVM cannot see (socket read, file read). Not proof of CPU burn |
| `BLOCKED` | Waiting to acquire an **intrinsic monitor** — i.e. waiting to enter a `synchronized` block | Lock contention. Topic 85. The dump names the monitor and often the owner |
| `WAITING` | Parked indefinitely: `Object.wait()`, `Thread.join()`, `LockSupport.park()` | Waiting for another thread to signal. Topics 89, 94 |
| `TIMED_WAITING` | Same, with a deadline: `sleep`, `wait(ms)`, `parkNanos`, `poll(timeout)` | Usually benign — pool workers idling on a queue |
| `TERMINATED` | `run()` returned or threw | Gone |

**The distinction that matters most:** `RUNNABLE` in Java does **not** mean "burning
CPU". A thread blocked in `read()` on a socket is `RUNNABLE` from the JVM's point of
view, because the JVM does not model kernel-level blocking as a Java state. This is
why a thread dump showing 200 `RUNNABLE` threads is not evidence of CPU saturation,
and why you need Topic 78's profiler in **wall-clock mode** to tell the difference.

**The second distinction:** `BLOCKED` means `synchronized`, specifically. A thread
waiting on a `ReentrantLock` shows as `WAITING` (it is parked), not `BLOCKED`. That
one fact lets you tell at a glance which locking mechanism a hung service is using.

### Daemon versus non-daemon

```java
Thread t = Thread.ofPlatform().daemon(true).start(runnable);
```

The JVM exits when the last **non-daemon** thread finishes. Daemon threads are killed
abruptly at that point — no `finally` blocks, no shutdown hooks of their own.

- Application worker that must finish its unit of work → **non-daemon**.
- Background metrics flusher, cache refresher, heartbeat → **daemon**, so it never
  keeps a dying JVM alive.

`main` is non-daemon. Threads created by `Executors.defaultThreadFactory()` are
non-daemon, which is why a forgotten `ExecutorService` keeps your JVM running forever
after `main` returns. You will meet that as a symptom in Topic 90.

### Interruption — Java's only cooperative cancellation

Java has **no** way to forcibly stop a thread. What it has is a per-thread boolean
called the **interrupt flag**, plus a convention.

```java
t.interrupt();                          // sets the flag on t
Thread.currentThread().isInterrupted(); // reads it, does NOT clear it
Thread.interrupted();                   // reads it AND clears it — static, easy to misuse
```

Two behaviours follow:

1. If the target thread is **blocked in an interruptible method** (`sleep`, `wait`,
   `join`, `BlockingQueue.take`, `Lock.lockInterruptibly`, most NIO channel ops), that
   method throws `InterruptedException` **and clears the flag**.
2. If the target thread is **running normally**, nothing happens automatically. The
   flag is set, and it is that thread's job to check it.

So a cancellable loop looks like this:

```java
while (!Thread.currentThread().isInterrupted()) {
    OrderLine line = queue.take();     // may throw InterruptedException
    process(line);
}
```

And the catch block has exactly one correct default shape:

```java
try {
    queue.take();
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();   // RESTORE the flag you just consumed
    throw new OrderProcessingAbortedException("interrupted while draining", e);
}
```

Why restore it? Because `InterruptedException` **cleared** the flag when it was
thrown. If you swallow the exception without restoring, the cancellation signal is
destroyed. Your caller — often a thread pool trying to shut down — sees a thread that
was never asked to stop. That is Trap 4, and it is the most common concurrency bug in
enterprise Java by volume.

### What is *not* interruptible

Interruption does not unblock:

- a plain `synchronized` acquisition (that is why `ReentrantLock.lockInterruptibly()`
  exists — Topic 94),
- a blocking read on a plain `InputStream` from a `Socket`,
- native code.

So "interrupt it" is not a general cancellation mechanism. It is a *cooperative
protocol* that works only where the code you are interrupting participates.

### Uncaught exceptions kill one thread, quietly

```java
Thread.ofPlatform().start(() -> {
    throw new IllegalStateException("inventory reload failed");
});
```

That thread dies. The JVM keeps running. By default the exception is printed to
`System.err` by the default `ThreadGroup` handler — which, in a container with
structured JSON logging, frequently means it lands somewhere nobody is looking.

In Node an unhandled rejection can be configured to kill the process, and by default
in modern Node it does. In Java the default is the opposite: the failure is *local*
and *silent enough to miss*.

Always set a handler:

```java
Thread.setDefaultUncaughtExceptionHandler((thread, throwable) ->
        log.error("uncaught in thread {}", thread.getName(), throwable));
```

And when you use an `ExecutorService`, know that `submit()` **captures** the exception
into the returned `Future` and never prints it at all. If nobody calls `get()`, the
exception is gone entirely. `execute()` propagates it to the handler; `submit()` does
not. That asymmetry is Topic 90's problem and a favourite interview question.

### Thread names are an operational feature, not decoration

```java
Thread.ofPlatform().name("orderflow-inventory-", 0).factory();
```

Every thread dump, every flame graph, every JFR event is keyed on the thread name. A
service whose threads are called `Thread-14` and `pool-3-thread-7` costs you real
minutes at 3am. Name every pool you create after the work it does.

Tomcat already does this: `http-nio-8080-exec-N`. When you see that prefix in a dump,
you are looking at a request thread in `orderflow`.

### `[JAVA 25]` A one-line note on headers

JDK 25 ships **compact object headers** (`-XX:+UseCompactObjectHeaders`, production
in 25) which shrink the object header from 96 to 64 bits and change the mark word's
internal field layout — the same mark word Topic 85 uses for lock state. Verify what
your JVM is actually doing before quoting any layout:

```bash
java -XX:+PrintFlagsFinal -version | grep -i CompactObjectHeaders
```

Nothing in *this* topic depends on it. Topic 85 says more.

---

## Why does it matter?

Three concrete consequences, all of which are already live in `orderflow` today.

**1. Every field of every singleton Spring bean is shared mutable state.**

`orderflow`'s `InventoryService` is a singleton. Tomcat runs up to 200 request threads
by default. If that service holds any mutable field — a cache, a counter, a
`SimpleDateFormat`, an accumulated `StringBuilder` — then 200 threads can be inside
that object at the same instant. You did not write a thread. You did not think about
concurrency. You have it anyway, because Spring MVC is thread-per-request and you
wrote a field.

This is the single largest category of production Java bug for engineers arriving from
Node, and it is invisible in code review unless you are specifically looking.

**2. Your load test is a race-condition generator.**

Topic 65's k6 profile is 10% order placement against 100k products with a few hot SKUs.
"A few hot products" means many concurrent threads hitting the same inventory row and
the same in-memory cache entry. That is the exact shape that turns a theoretical
interleaving into an observed one. You built a race detector and called it a load test.

**3. Thread count is a resource with a hard ceiling.**

Each platform thread reserves stack space — 1 MB by default on 64-bit HotSpot
(`-Xss`), reserved as virtual address space and committed as touched. At a few
thousand threads you are into real memory and real scheduler overhead; at some
OS-dependent limit you get:

```
java.lang.OutOfMemoryError: unable to create native thread: possibly out of memory or process/resource limits reached
```

which is an OOM message that has nothing to do with your heap. Topic 98 drills it;
Topic 101 removes the ceiling for blocking I/O.

---

## Machine-level reality

### A Java thread *is* an OS thread

`Thread.start()` ends in a native call. On Linux the JVM calls `pthread_create`, which
calls `clone(2)` with the flags that share the address space. On macOS it is
`pthread_create` over Mach threads. The kernel scheduler now owns your thread. HotSpot
does not schedule Java threads; it creates them and gets out of the way.

Consequences that follow directly:

- **The kernel decides when your thread runs, and can stop it at any instruction
  boundary.** A timer interrupt fires, the scheduler runs, your thread is descheduled.
  It does not ask the JVM. It does not wait for a convenient point.
- **Creation costs a syscall plus a stack reservation.** Order of magnitude: tens of
  microseconds and 1 MB of reserved address space. This is why pools exist.
- **A context switch costs a scheduler pass plus cache and TLB disruption.** The
  direct cost is a few microseconds; the indirect cost — a cold L1/L2 for the incoming
  thread — is frequently larger and does not show up in any timer you own.

### "Between any two bytecodes" is literal

Take the line you will write a hundred times:

```java
stock = stock - 1;
```

For an `int` field on `this`, `javac` emits roughly:

```
aload_0            // push 'this'
aload_0            // push 'this' again
getfield  stock    // READ the field  <-- (1)
iconst_1
isub               // compute stock-1 <-- (2)
putfield  stock    // WRITE the field <-- (3)
```

Three separately observable steps. The scheduler may deschedule your thread between
(1) and (3). While you are off the CPU, another thread may run all of (1), (2), (3).
When you resume, you execute (3) with the value you computed *before* the other
thread's write. Their decrement is erased.

This is a **lost update**, and it is the canonical shape of a data race. You will
reproduce it in the Concurrency trace below and in the Failure drill.

You are not guessing at this. Topic 76 taught you to read the disassembly; you can
confirm the three instructions with `javap -c` in the Hands-on proof.

### Even a single field write is not always atomic

JLS §17.7: reads and writes of `long` and `double` are **not guaranteed atomic**
unless the field is `volatile`. A 64-bit value may be written as two 32-bit halves, and
a concurrent reader may observe one old half and one new half — a value that was never
written by anyone. This is called **word tearing**.

In practice, 64-bit HotSpot implements `long`/`double` accesses atomically, so you are
unlikely to observe it on your Mac. **That is not a licence to rely on it**: it is not
a specification guarantee, it does not hold on all VMs, and interviewers ask precisely
because the practical answer and the spec answer differ.

Reference writes *are* atomic — you never see half a pointer. That does not make them
safe: you can see a fully-formed reference to a half-initialised object, which is
Topic 88's entire subject.

### The stack: private, and where safety comes from

```
Thread A                     Thread B
+-------------+              +-------------+
| frame: main |              | frame: run  |
|  int i      |   PRIVATE    |  int j      |   PRIVATE
|  ref o -----|--+        +--|--- ref o    |
+-------------+  |        |  +-------------+
                 v        v
        +----------------------------+
        |          HEAP              |   SHARED
        |   Inventory { stock=7 }    |
        +----------------------------+
```

`i` and `j` cannot race. The `Inventory` object can. Every concurrency bug in this
phase lives on the right-hand side of that diagram.

### The stack size, concretely

```bash
java -XX:+PrintFlagsFinal -version | grep -i ThreadStackSize
```

Default is typically 1 MB on 64-bit platforms (`-Xss1m`). That is *reserved* virtual
address space; physical pages are committed as the stack grows. So 1,000 threads
reserve ~1 GB of address space but may commit far less. Address space is cheap on
64-bit; the real limits you hit first are scheduler overhead and the OS thread limit.

`-Xss` is worth knowing for the opposite reason too: deep recursion throws
`StackOverflowError`, and raising `-Xss` is the diagnostic knob.

### Parking: how a thread stops without spinning

When a thread must wait, the JVM does not spin forever. It **parks**:

- `LockSupport.park()` / `unpark(thread)` are the JDK-level primitives.
- Under them, HotSpot uses a per-thread `Parker`. On Linux that is implemented with a
  **futex** (`futex(FUTEX_WAIT)` / `FUTEX_WAKE`) — a fast path entirely in userspace
  when uncontended, dropping into the kernel only when a thread must actually sleep.
  On macOS it is a pthread mutex plus condition variable.

The point for you: **parking and unparking a thread involves the kernel.** The cost is
in the microseconds, not the nanoseconds. That is the specific reason contended locks
are expensive and uncontended locks are not — Topic 85 develops this in full.

### Safepoints — the JVM's own preemption points

You met these at Topic 73. Restating the part that matters here: the JVM sometimes
needs *all* Java threads to stop at a known-good point (a GC, a deoptimisation, a
biased-lock revocation historically, `Thread.print`). It does this by setting a global
flag that threads poll at method returns and loop back-edges.

Two consequences relevant now:

1. A **thread dump is taken at a safepoint**. What you read is a consistent snapshot,
   not a live view, and taking it stops your service briefly.
2. A counted `int` loop with no allocation may contain **no safepoint poll at all** in
   compiled code, so it cannot be stopped. That is the time-to-safepoint problem from
   Topic 73 — and it is the same mechanism that lets a JIT-compiled loop never
   re-read a field, which is Topic 86's bug.

Notice that safepoints are the JVM's *own* preemption mechanism, layered on top of the
OS's. Both exist. Neither gives you run-to-completion.

### x86-TSO versus aarch64 — say this once now, in full at Topic 87

You are on macOS, and most likely on Apple Silicon, which is **aarch64**.

- **x86-64 implements Total Store Order (TSO).** The hardware forbids reordering loads
  with loads, stores with stores, and loads with earlier stores in program order. The
  only reordering the hardware permits is a **later load moving ahead of an earlier
  store to a different address**, which is a consequence of the store buffer.
- **aarch64 has a substantially weaker model.** It permits load-load, load-store,
  store-store and store-load reordering unless you use explicit barriers or the
  acquire/release load and store instructions.

**Which way this cuts for you:** memory-ordering bugs that x86 hardware silently hides
can become observable on Apple Silicon. That makes your laptop a *better* test
environment for Topics 87 and 88 than a typical x86 CI runner.

**And the honest limit:** it does not make them *certain*. Whether a particular
reordering is observed on a particular run depends on the JIT's output, the core
layout, what else the machine is doing, and luck. I will never tell you a race "will"
reproduce. I will tell you what to run and how to read what comes back.

One important exception, which you should hold onto through Topics 86 and 87:

> **The Topic 86 stop-flag bug is a *compiler* bug, not a hardware one.** The JIT
> hoists the field read out of the loop. That happens identically on x86 and aarch64.
> Architecture is irrelevant there. Architecture becomes relevant at Topic 87 and 88,
> where the reordering can also come from the CPU.

---

## Concurrency trace

**This is the most valuable thing in this document.** Read it line by line before you
read any correct code. Every bug in Phase 9 is a specific interleaving; if you can
write the interleaving, you understand the bug, and if you cannot, you are guessing.

### The scenario

`orderflow` holds a per-SKU in-memory reservation counter in a singleton
`InventoryService`, in front of the database, to shed obviously-impossible orders
before they reach Postgres. `SKU-4471` is one of Topic 65's hot products. It has
**1 unit** left.

Two Tomcat request threads arrive at the same instant, both placing an order for one
unit of `SKU-4471`.

The code under trace:

```java
// InventoryService — a Spring singleton. Shared by every request thread.
private final Map<String, Integer> stockBySku = new HashMap<>();

public boolean tryReserve(String sku) {
    Integer available = stockBySku.get(sku);   // READ
    if (available == null || available < 1) {
        return false;
    }
    stockBySku.put(sku, available - 1);        // COMPUTE and WRITE
    return true;
}
```

### The interleaving

| Step | Thread A — `http-nio-8080-exec-3` (order #90210) | Thread B — `http-nio-8080-exec-7` (order #90211) | Shared state / what is visible |
|---|---|---|---|
| 1 | Enters `tryReserve("SKU-4471")` | — | `stockBySku["SKU-4471"] = 1` |
| 2 | `getfield` / map `get` → reads **1** into a local | — | map still `= 1` |
| 3 | Evaluates `available < 1` → **false**, so it proceeds | — | map still `= 1` |
| 4 | **Scheduler preempts A here.** A is descheduled between the read and the write. Its local `available = 1` is on A's private stack, frozen | — | map still `= 1`. A's intention to write 0 exists nowhere in shared memory |
| 5 | — | Enters `tryReserve("SKU-4471")` | map `= 1` |
| 6 | — | map `get` → reads **1**. This read is *correct*: the map really does still say 1 | map `= 1` |
| 7 | — | Evaluates `available < 1` → **false**, proceeds | map `= 1` |
| 8 | — | `put("SKU-4471", 1 - 1)` → writes **0** | map `= 0` |
| 9 | — | Returns `true`. Order #90211 is accepted. B continues: debits wallet, writes order row | map `= 0`, one order committed |
| 10 | **A resumes**, on whatever core is free. It does *not* re-read the map — it already has `available = 1` in a local | — | map `= 0` |
| 11 | `put("SKU-4471", 1 - 1)` → writes **0** over B's 0 | — | map `= 0`. B's decrement has been overwritten by an identical value, so the corruption is invisible in the data |
| 12 | Returns `true`. Order #90210 is accepted | — | map `= 0`, **two** orders committed |
| **13** | **OUTCOME: two customers were each told they successfully bought the last unit of `SKU-4471`. One unit exists. `orderflow` has oversold. One of those two customers will be refunded, apologised to, and possibly lost — and the stock count in memory reads a perfectly plausible `0`, so nothing in the data looks wrong.** | | |

### What to take from step 11

The map ends at `0`, which is the correct-looking value. Two decrements happened; one
was lost; the final number is indistinguishable from the correct one. **The data
carries no evidence of the bug.** You find it from the business: two orders, one unit.

This is why concurrency bugs are expensive. They do not fail loudly. They produce
plausible state and an angry customer six hours later.

### The three separate defects in eleven lines

Do not let the fix collapse into "add `synchronized`". There are three independent
problems, and they map to three different topics:

1. **Check-then-act is not atomic.** Read, decide, write is three operations with two
   gaps. Topic 85 (`synchronized`), Topic 95 (CAS), Topic 92 (`compute`).
2. **`HashMap` is not thread-safe at all.** Concurrent `put` during a resize can
   corrupt the internal table. Historically, on Java 7, this could produce an infinite
   loop in `get` — a permanently spinning CPU. On Java 8 the resize is not circular,
   but concurrent mutation still loses entries and can produce arbitrary garbage.
   Topic 92.
3. **There is no happens-before edge**, so even a *correct* sequence of operations
   gives no guarantee that thread B ever sees thread A's write. Topics 86 and 87.

Problem 3 is the one your intuition will not generate on its own, and it is why
Topics 86–88 are `ELITE`.

---

## Example 1 — minimal

### The lost update, in the smallest program that shows it

```java
public class LostUpdate {

    // Shared, mutable, unguarded. The entire bug lives here.
    static int unitsReserved = 0;

    public static void main(String[] args) throws InterruptedException {

        final int threads = 8;
        final int perThread = 100_000;

        Thread[] workers = new Thread[threads];
        for (int i = 0; i < threads; i++) {
            workers[i] = Thread.ofPlatform()
                    .name("reserver-" + i)
                    .unstarted(() -> {
                        for (int n = 0; n < perThread; n++) {
                            unitsReserved++;          // read, add, write
                        }
                    });
        }

        for (Thread t : workers) t.start();
        for (Thread t : workers) t.join();            // wait for all to finish

        int expected = threads * perThread;
        System.out.println("expected  = " + expected);
        System.out.println("actual    = " + unitsReserved);
        System.out.println("lost      = " + (expected - unitsReserved));
    }
}
```

```bash
java LostUpdate.java
```

**What to look for:** the value of `lost`.

| What you see | What it means |
|---|---|
| `lost` is greater than 0, and a different number on each run | The expected result. You have directly observed lost updates. The non-determinism is the signature of a race |
| `lost` is 0 on this run | It happens. The interleaving that loses an update is *permitted*, not *mandatory*. Run it ten more times; raise `perThread` to 10 million; run with `-XX:-TieredCompilation` and with `-Xint` and compare. A single clean run proves nothing at all |
| `lost` is a large fraction of `expected` | Normal at high thread counts. Each thread spends most of its time working from a stale local value |
| An exception | Something else is wrong — `unitsReserved` is a plain `static int`, no exception should be possible here |

**The `join()` matters.** `Thread.join()` is one of the happens-before edges you will
memorise in Topic 86: everything the joined thread did is guaranteed visible to you
after `join()` returns. Without it, `main` could print a value from before the workers
even started, and you would be measuring nothing.

### The same program, showing the preemption directly

```java
public class PreemptionWindow {

    static int stock = 1;
    static int accepted = 0;

    public static void main(String[] args) throws InterruptedException {

        Runnable reserve = () -> {
            int available = stock;                    // READ
            Thread.onSpinWait();                      // widen the window, no sleep
            if (available >= 1) {
                stock = available - 1;                // WRITE
                synchronized (PreemptionWindow.class) { accepted++; }
            }
        };

        for (int trial = 0; trial < 10_000; trial++) {
            stock = 1;
            accepted = 0;
            Thread a = Thread.ofPlatform().unstarted(reserve);
            Thread b = Thread.ofPlatform().unstarted(reserve);
            a.start(); b.start();
            a.join();  b.join();
            if (accepted == 2) {
                System.out.println("OVERSOLD on trial " + trial
                        + ": accepted=" + accepted + ", stock=" + stock);
                return;
            }
        }
        System.out.println("no oversell observed in 10000 trials");
    }
}
```

**What to look for:** whether the program prints an `OVERSOLD` line, and on which
trial.

| What you see | What it means |
|---|---|
| `OVERSOLD on trial <n>` | You have reproduced the exact interleaving from the Concurrency trace, in isolation. Note that `n` varies wildly between runs |
| `no oversell observed in 10000 trials` | The window is narrow — two threads must be preempted inside a three-instruction sequence. Raise the trial count, add more threads per trial, or run under load. **This is not evidence the code is safe.** It is evidence that a test loop is a bad way to find races, which is exactly the argument for jcstress (Topic 99) |
| It hangs | Check your `join()` calls. Nothing here should block |

Note the `synchronized` on `accepted++`: I am guarding the *measurement* so the count
of accepted orders is itself trustworthy. Guarding your instrumentation while leaving
the subject unguarded is a standard technique, and it is worth being explicit about
why: if both the subject and the instrument are racy, you cannot tell which one lied.

---

## Example 2 — production scenario (on the project spine)

### The constraints

From Topic 65's recorded baseline:

- `orderflow` runs containerised: app + Postgres + Redis, under `docker compose`.
- Dataset: 100k products, 1M orders, 5M order lines, with a small hot set.
- Load: k6, open model, 70% catalogue read / 20% order read / 10% order placement.
- Tomcat default `server.tomcat.threads.max=200`.

### The code that looks right and oversells

This is written the way a strong Node engineer writes it on day one of Java. Nothing
in it is stupid. It is idiomatic, readable, tested, and wrong.

```java
package com.orderflow.inventory;

import org.springframework.stereotype.Service;
import java.util.HashMap;
import java.util.Map;

@Service                                  // SINGLETON. One instance. 200 threads.
public class InventoryCache {

    /** Hot-SKU reservation counters, in front of Postgres to shed impossible orders. */
    private final Map<String, Integer> availableBySku = new HashMap<>();

    /** Incremented for observability. */
    private long reservationAttempts = 0;
    private long reservationRejections = 0;

    public void warmUp(Map<String, Integer> snapshot) {
        availableBySku.putAll(snapshot);
    }

    public boolean tryReserve(String sku, int quantity) {
        reservationAttempts++;                                  // race #1

        Integer available = availableBySku.get(sku);            // race #2 (read)
        if (available == null || available < quantity) {
            reservationRejections++;                            // race #1
            return false;
        }

        availableBySku.put(sku, available - quantity);          // race #2 (write)
        return true;
    }

    public long attempts()   { return reservationAttempts; }
    public long rejections() { return reservationRejections; }
}
```

And its caller:

```java
package com.orderflow.orders;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class OrderPlacementService {

    private final InventoryCache inventoryCache;
    private final InventoryRepository inventoryRepository;
    private final WalletService walletService;
    private final OrderRepository orderRepository;

    public OrderPlacementService(InventoryCache inventoryCache,
                                 InventoryRepository inventoryRepository,
                                 WalletService walletService,
                                 OrderRepository orderRepository) {
        this.inventoryCache = inventoryCache;
        this.inventoryRepository = inventoryRepository;
        this.walletService = walletService;
        this.orderRepository = orderRepository;
    }

    @Transactional
    public OrderId place(PlaceOrderCommand cmd) {

        // Fast rejection path, in memory, before we touch the database.
        if (!inventoryCache.tryReserve(cmd.sku(), cmd.quantity())) {
            throw new OutOfStockException(cmd.sku());
        }

        inventoryRepository.decrement(cmd.sku(), cmd.quantity());
        walletService.debit(cmd.customerId(), cmd.total());
        return orderRepository.save(Order.from(cmd)).id();
    }
}
```

### What actually happens under the Topic 65 load

Walk it against the trace above.

**Failure 1 — oversell on hot SKUs.** With a few hot products and an open-model
arrival rate, many of the 200 request threads sit on the same SKU key. The
read-decide-write window in `tryReserve` is a handful of nanoseconds wide, but at
Topic 65's request rate you get millions of chances per minute. Some fraction of them
land inside the window. The in-memory guard lets through more orders than there is
stock, and the real damage then happens in `inventoryRepository.decrement`, where you
either drive stock negative or take a database error at high volume.

**Failure 2 — the metrics are wrong, and wrong in a way that hides Failure 1.**
`reservationAttempts++` is a `long` read-modify-write with no guard. Under 200 threads
it loses a large fraction of its increments. Your rejection *rate* — the one number
that would have told you the guard was misbehaving — is computed from two independently
corrupted counters. The dashboard is confidently wrong.

**Failure 3 — the `HashMap` itself can break.** `availableBySku` is a plain `HashMap`
mutated by 200 threads. Concurrent `put` during a resize is undefined behaviour: lost
entries, entries appearing under the wrong key, and a `get` that returns something
absurd. If a hot SKU's entry is lost, `available == null` and every subsequent order
for the best-selling product in the catalogue is rejected as out of stock, while the
database says there are 40,000 units. That is a revenue outage caused by a cache.

**Failure 4 — the fast path is not actually consistent with the slow path.** Even if
you fix 1–3, an in-memory counter and a database row are two sources of truth that
drift on every restart, every deploy and every replica. That is a *design* problem,
not a concurrency problem, and no amount of `synchronized` fixes it.

### The fix, in the order you should apply it

The instinct is to wrap `tryReserve` in `synchronized`. Resist that for sixty seconds
and think about the four failures separately.

**Step 1 — delete the metrics race.** Counters that are only ever incremented and
occasionally read are exactly what `LongAdder` is for (Topic 95).

```java
private final LongAdder reservationAttempts   = new LongAdder();
private final LongAdder reservationRejections = new LongAdder();
// ...
reservationAttempts.increment();
// ...
public long attempts() { return reservationAttempts.sum(); }
```

**Step 2 — make the map safe and the operation atomic.** `ConcurrentHashMap.compute`
performs the read-decide-write as one atomic operation on that key, and it locks only
that key's bin, not the whole map (Topic 92).

```java
private final ConcurrentHashMap<String, Integer> availableBySku = new ConcurrentHashMap<>();

public boolean tryReserve(String sku, int quantity) {
    reservationAttempts.increment();

    // compute() is atomic per key. The lambda may be retried; it must be
    // side-effect-free apart from returning the new value.
    Integer result = availableBySku.compute(sku, (k, available) -> {
        if (available == null || available < quantity) {
            return available;                       // unchanged
        }
        return available - quantity;                // reserved
    });

    boolean reserved = result != null
            && result.intValue() == availableBySku.getOrDefault(sku, -1);
    // ^ see below: this is still not right. Read on.
    ...
}
```

Stop. That last expression is wrong, and I have left it in deliberately because it is
the mistake you will make. `compute` tells you the *new* value; it does not tell you
whether you were the thread that changed it. Re-reading the map afterwards is another
check-then-act race. **Atomic operations do not compose into atomic sequences.** That
sentence is Topic 92's mastery line, and you have just watched it bite.

The correct shape carries the decision out of the lambda:

```java
public boolean tryReserve(String sku, int quantity) {
    reservationAttempts.increment();

    // A one-element holder is the standard trick for getting a boolean out of
    // a compute() lambda. It is only touched by the thread inside the atomic
    // section, so it is not itself shared.
    boolean[] reserved = { false };

    availableBySku.compute(sku, (k, available) -> {
        if (available == null || available < quantity) {
            return available;
        }
        reserved[0] = true;
        return available - quantity;
    });

    if (!reserved[0]) {
        reservationRejections.increment();
    }
    return reserved[0];
}
```

**Step 3 — recognise that steps 1 and 2 fixed the *symptom*, not the design.** The
authoritative fix for oversell in `orderflow` is not in memory at all. It is the
atomic conditional UPDATE you built at Topic 52:

```sql
UPDATE inventory
   SET available = available - :qty
 WHERE sku = :sku
   AND available >= :qty
```

which returns 0 rows when there is not enough stock, in one round trip, with the
database's own concurrency control doing the work. The in-memory cache should be a
*hint* that sheds obviously-hopeless load, and it must be allowed to be wrong in the
safe direction (letting through an order that the database then rejects), never in the
unsafe direction.

**This is the senior judgement in the whole example:** the correct concurrency fix is
frequently to *move the invariant somewhere that already enforces it*, not to add a
lock. Topic 85 will give you the lock. Knowing when not to reach for it is the
difference between mid and senior.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — calling `run()` instead of `start()`

**Wrong:**

```java
Thread worker = new Thread(() -> reprocessFailedPayments());
worker.run();                               // <-- no new thread
```

**Exact symptom:** the program is correct, deterministic and slow. Timings are exactly
sequential — eight "parallel" jobs take eight times one job. `Thread.currentThread()
.getName()` inside the task prints `main` (or the caller's name), never `Thread-0`. A
thread dump taken while it is "running in parallel" shows a single thread with all the
work on its stack. No exception, no warning, no compiler complaint.

**Root cause:** `run()` is an ordinary instance method. Only `start()` performs the
native call that asks the OS for a thread. This compiles because `Runnable.run()` is
public.

**Fix:** call `start()`. Then stop writing raw threads and use an `ExecutorService`
(Topic 90), where the API gives you no way to make this mistake — `submit()` has no
"accidentally run it here" sibling.

**How to catch it in review:** any bare `.run()` on something named like a thread or a
task is a defect until proven otherwise.

---

### Trap 2 — `Thread.stop()` and `Thread.suspend()`

**Wrong:**

```java
worker.stop();       // "cancel the job"
worker.suspend();    // "pause the job"
```

**Exact symptom:** on a modern JDK, an `UnsupportedOperationException` at the call
site. On a JDK old enough to still perform the operation, something far worse: silent
data corruption with no exception at all.

Verify what your JDK does rather than trusting me:

```java
public class DegradedApis {
    public static void main(String[] args) throws Exception {
        Thread t = Thread.ofPlatform().start(() -> {
            while (true) { Thread.onSpinWait(); }
        });
        Thread.sleep(100);
        try { t.stop(); }    catch (Throwable e) { System.out.println("stop():    " + e); }
        try { t.suspend(); } catch (Throwable e) { System.out.println("suspend(): " + e); }
        System.exit(0);
    }
}
```

```bash
java --version
java DegradedApis.java
```

| What you see | What it means |
|---|---|
| `UnsupportedOperationException` for both | Your JDK has degraded these methods to throw. This is the modern behaviour and it is the right one |
| A compiler warning about deprecation and removal | Expected — they have been deprecated for removal for years |
| Anything else | Note it down; you are on an older or unusual JDK, and the numbers you get in later drills may differ |

**Root cause — and this is the part interviewers actually want.** `Thread.stop()`
worked by throwing a `ThreadDeath` error at the target thread at an *arbitrary*
instruction. That means:

- It could fire in the middle of a multi-field update, leaving an object in a state
  that violates its own invariants — for example, an `Order` with a `total` updated but
  its `lines` not yet appended.
- It **released every monitor the thread held**, immediately. Other threads then
  entered `synchronized` blocks and observed the broken invariant, believing themselves
  correctly protected. The corruption spreads silently to code that did everything
  right.
- There was no way to write a `finally` block that could cope, because the error could
  arrive inside the `finally` block too.

`suspend()` was worse in a different way: it stopped the thread **while holding its
locks**. Any other thread needing those locks blocked forever. That is a deadlock with
no cycle in it, which is why it does not show up in a thread dump's deadlock section.

**Fix:** cooperative cancellation. `interrupt()`, plus a task that checks
`Thread.currentThread().isInterrupted()` at safe points and unwinds cleanly. If your
task cannot be cancelled cooperatively — a blocking read on a socket, say — the fix is
a timeout on the socket, not a way to kill the thread.

---

### Trap 3 — swallowing `InterruptedException`

**Wrong:**

```java
try {
    OrderLine line = queue.poll(1, TimeUnit.SECONDS);
    process(line);
} catch (InterruptedException e) {
    log.warn("interrupted");          // and carry on round the loop
}
```

**Exact symptom:** the service will not shut down. `docker compose down` hangs for the
full grace period and Docker sends `SIGKILL`. In Kubernetes, the pod sits in
`Terminating` for `terminationGracePeriodSeconds` on every deploy, and a rolling update
that should take 30 seconds takes ten minutes. `ExecutorService.awaitTermination` times
out. Nothing logs an error, because every layer thinks it did its job. The only
evidence is the word "interrupted" in the logs, repeatedly, from a thread that never
stopped.

**Root cause:** throwing `InterruptedException` **clears** the interrupt flag. Catching
it without restoring the flag destroys the only record that cancellation was requested.
Your loop condition, your pool's shutdown check, and every wrapper above you now see a
thread that was never asked to stop.

**Fix — one of exactly two options, and you must pick deliberately:**

```java
// Option A: you cannot handle it here. Restore the flag and propagate.
catch (InterruptedException e) {
    Thread.currentThread().interrupt();
    throw new OrderDrainAbortedException(e);
}

// Option B: you ARE the top of the thread and cancellation means "return".
catch (InterruptedException e) {
    Thread.currentThread().interrupt();   // still restore it, for anything above you
    return;                               // and exit the loop
}
```

Option A is the default. Option B is only correct in a `Runnable` you own end to end.
There is no third option where you log and continue.

**How to catch it in review:** grep the codebase.

```bash
grep -rn "catch (InterruptedException" --include=*.java src/ \
  | while IFS= read -r hit; do echo "CHECK: $hit"; done
```

Then confirm each one either rethrows or calls `Thread.currentThread().interrupt()`.
This is a five-minute audit that finds real bugs in almost every Java codebase.

---

### Trap 4 — "the other thread will see it eventually"

**Wrong:**

```java
private boolean cacheWarm = false;              // no volatile, no lock

public void warmUp() {
    loadCatalogue();
    cacheWarm = true;                           // writer thread
}

public void awaitWarm() {
    while (!cacheWarm) {                        // reader thread
        Thread.onSpinWait();
    }
}
```

**Exact symptom:** `awaitWarm()` never returns. The service starts, reports healthy,
and one thread spins at 100% of one core forever. `jcmd <pid> Thread.print` shows that
thread as `RUNNABLE`, sitting on the `while` line, with no lock held and nothing to
wait for. It looks like an infinite loop bug, and everyone reads the code and concludes
the loop is obviously correct because `cacheWarm` is obviously set.

The tell: **it works in your IDE and hangs under load**, or it works with a debugger
attached and hangs without one. Both of those change JIT behaviour.

**Root cause:** there is no happens-before edge between the write and the read.
Without one, the JIT is permitted to hoist the read of `cacheWarm` out of the loop —
compiling it once into a register and testing the register forever. The write may also
sit in the writer core's store buffer. Neither is a bug in the JVM. Both are legal
compilations of a program that made no ordering demand.

**Say the sentence that fixes your model:**

> **"Eventually" is not in the specification.** There is no clause in the JLS that
> promises a write becomes visible after some interval. Absent a happens-before edge,
> a thread may read a stale value *forever*.

**Fix:** establish the edge. `volatile boolean cacheWarm` is the minimum. A
`CountDownLatch` is better, because it expresses the intent and does not spin.

This is Topic 86, and it is the drill for that topic. I am naming it here so that when
you reach it you already have the symptom in your memory.

---

### Trap 5 — a thread per request, or a pool per request

**Wrong:**

```java
@PostMapping("/orders")
public OrderResponse place(@RequestBody PlaceOrderRequest req) {
    ExecutorService pool = Executors.newFixedThreadPool(4);   // per request!
    var f1 = pool.submit(() -> pricing.quote(req));
    var f2 = pool.submit(() -> fraud.score(req));
    // ... and no shutdown()
    return combine(f1.get(), f2.get());
}
```

**Exact symptom:** thread count grows monotonically and never falls. `jcmd <pid>
Thread.print | grep -c 'java.lang.Thread.State'` climbs with every request. Latency
degrades gradually over hours as the scheduler manages tens of thousands of threads.
Eventually:

```
java.lang.OutOfMemoryError: unable to create native thread: possibly out of memory or process/resource limits reached
```

which is an `OutOfMemoryError` that a heap dump will not explain, because the heap is
fine. In a container with a low `pids` limit, you get a different error at a lower
count.

**Root cause:** `Executors.newFixedThreadPool` creates **non-daemon** threads that live
until `shutdown()` is called. Every request leaks four of them. `Future.get()` succeeded
so the code appeared to work.

**Fix:** one pool, created once, owned by a Spring bean with a `@PreDestroy` that shuts
it down. Pools are infrastructure, not local variables. Topic 90 covers sizing, bounded
queues and rejection policy; Topic 98 drills this exact leak.

```java
@Bean(destroyMethod = "shutdown")
ExecutorService orderFanOutExecutor() {
    return new ThreadPoolExecutor(
            8, 8, 0L, TimeUnit.MILLISECONDS,
            new ArrayBlockingQueue<>(256),                 // BOUNDED
            Thread.ofPlatform().name("orderflow-fanout-", 0).factory(),
            new ThreadPoolExecutor.CallerRunsPolicy());
}
```

---

### Trap 6 — assuming an uncaught exception is loud

**Wrong:**

```java
executor.submit(() -> {
    reconcileWallets();          // throws on bad data
});
```

**Exact symptom:** wallets stop reconciling. There is no stack trace anywhere. No error
metric fires. The thread is alive and healthy and processing the next task. You find out
from a finance report a week later.

**Root cause:** `ExecutorService.submit()` wraps the task in a `FutureTask`, which
**captures** any thrown exception into the `Future`. If nobody calls `get()`, the
exception is never observed and never printed. `execute()` behaves differently: it
propagates to the thread's uncaught exception handler.

**Fix — three layers, all of them:**

```java
// 1. Global net.
Thread.setDefaultUncaughtExceptionHandler(
        (t, e) -> log.error("uncaught in {}", t.getName(), e));

// 2. Use execute() for fire-and-forget work, so the handler actually sees it.
executor.execute(() -> reconcileWallets());

// 3. If you must use submit(), OBSERVE the future.
executor.submit(() -> reconcileWallets())
        .exceptionally(...)   // for CompletableFuture; for Future, check get()
```

For scheduled tasks this is even sharper: a `ScheduledExecutorService` task that throws
is **not rescheduled**. Your periodic job silently stops forever after its first
failure. Always wrap the body in a try/catch that logs and returns.

---

## Hands-on proof

Everything below is a command you run. I do not have a JVM and I will not print output
and call it real. What I give you is the exact command, what to look for, and a table
mapping what you see to what it means.

### Setup

```bash
mkdir -p ~/java-lab/84 && cd ~/java-lab/84
java --version                  # expect 21 or 25
uname -m                        # arm64 on Apple Silicon, x86_64 on Intel
```

Record the `uname -m` result. It matters from Topic 87 onward.

### Proof 1 — `stock--` really is three bytecodes

`Decrement.java`:

```java
public class Decrement {
    int stock;
    void reserveOne() {
        stock = stock - 1;
    }
    static long total;
    static void addTo(long n) {
        total += n;
    }
}
```

```bash
javac Decrement.java
javap -c Decrement.class
```

**What to look for** in `reserveOne`:

- `getfield  #<n>  // Field stock:I` — the read
- `iconst_1`, `isub` — the compute
- `putfield #<n>  // Field stock:I` — the write

and in `addTo`:

- `getstatic`, `lload`, `ladd`, `putstatic` — the same shape on a `long`

| What you see | What it means |
|---|---|
| One `getfield` and one `putfield`, with arithmetic between | Confirmed. The scheduler can preempt at either boundary. This is the mechanical basis of the whole trace above |
| A single instruction doing the whole thing | Does not exist in the JVM instruction set. There is no atomic decrement bytecode. Atomicity, when you get it, comes from `synchronized`, `volatile` plus CAS, or an `Atomic*` class |
| `javap` refuses to run | You passed the `.java` file instead of the `.class`. Compile first |

Add `-v` for the constant pool if you want to resolve the `#<n>` indices. Topic 76 has
the full reading guide.

### Proof 2 — a thread is an OS thread

```bash
# Terminal 1: a JVM that just sleeps, with a couple of named threads.
cat > Sleeper.java <<'EOF'
public class Sleeper {
    public static void main(String[] args) throws Exception {
        for (int i = 0; i < 4; i++) {
            Thread.ofPlatform().name("orderflow-worker-" + i).start(() -> {
                try { Thread.sleep(Long.MAX_VALUE); } catch (InterruptedException e) { }
            });
        }
        Thread.sleep(Long.MAX_VALUE);
    }
}
EOF
java Sleeper.java &
JVMPID=$!
echo "pid=$JVMPID"
```

Now count OS-level threads for that process:

```bash
# macOS
ps -M $JVMPID | wc -l

# Linux
ls /proc/$JVMPID/task | wc -l
```

And ask the JVM what it thinks:

```bash
jcmd $JVMPID Thread.print | grep -c 'java.lang.Thread.State'
```

| What you see | What it means |
|---|---|
| The OS thread count is comfortably larger than 5 | Correct. Your four workers plus `main` are real OS threads, and the JVM adds its own: GC workers, the JIT compiler threads (`C1 CompilerThread`, `C2 CompilerThread`), `VM Thread`, `Reference Handler`, `Finalizer`, `Signal Dispatcher`, JFR threads |
| The two numbers do not match exactly | Expected. `Thread.print` lists only *Java* threads; the OS sees internal VM threads too |
| The OS count is 1 | You are looking at the wrong pid, or `java Sleeper.java` launched a child process. Use `jcmd -l` to find the real JVM pid |

**Why this proof matters:** it removes any lingering idea that "Java thread" is a
lightweight framework abstraction. It is a kernel object, and the kernel is what
preempts it.

Clean up: `kill $JVMPID`.

### Proof 3 — read a thread dump

```bash
jcmd -l                                  # list JVMs, find your pid
jcmd <pid> Thread.print > dump.txt
# or
jstack <pid> > dump.txt
```

Here is the **shape** of a dump section, so you can read one. The angle-bracket
placeholders are where real values go.

```
"http-nio-8080-exec-7" #<n> [<native-id>] daemon prio=5 os_prio=<n> cpu=<n>ms elapsed=<n>s tid=0x<addr> nid=0x<hex> waiting for monitor entry  [0x<addr>]
   java.lang.Thread.State: BLOCKED (on object monitor)
        at com.orderflow.inventory.InventoryCache.tryReserve(InventoryCache.java:<n>)
        - waiting to lock <0x<addr>> (a com.orderflow.inventory.InventoryCache)
        at com.orderflow.orders.OrderPlacementService.place(OrderPlacementService.java:<n>)
        ...

   Locked ownable synchronizers:
        - <none>
```

*Illustration of the format, not captured output.*

How to read it, field by field:

| Field | Meaning |
|---|---|
| `"http-nio-8080-exec-7"` | Thread name. This is why you name your pools |
| `daemon` | Present only for daemon threads |
| `tid` / `nid` | JVM thread id and native OS thread id. `nid` in hex maps to what `top -H` shows |
| `java.lang.Thread.State: BLOCKED (on object monitor)` | The state. `BLOCKED` here means it wants a `synchronized` monitor |
| `- waiting to lock <0x...>` | The monitor's identity hash. **Search the dump for the same address with `- locked`** to find the owner |
| `- locked <0x...>` | This thread holds that monitor |
| `Locked ownable synchronizers` | `ReentrantLock` and friends show here, not in the stack lines |

**The single most useful technique:** grep the dump for one monitor address. One thread
shows `- locked` on it; every other thread showing `- waiting to lock` on it is blocked
behind that one thread. That is your contention, named and counted, in one command.

```bash
grep -n '0x00000007xxxxxxxx' dump.txt      # substitute a real address from YOUR dump
```

And to see the distribution of states at a glance:

```bash
grep 'java.lang.Thread.State' dump.txt | sort | uniq -c | sort -rn
```

| What you see | What it means |
|---|---|
| Most threads `TIMED_WAITING` on a queue `poll` | Idle pool workers. Healthy |
| Many threads `BLOCKED` on one monitor address | Lock contention. Topic 85 is the whole answer |
| Many threads `RUNNABLE` deep in `SocketInputStream.read` | Blocked on I/O, not on CPU. Look downstream, not at your code |
| A `Found one Java-level deadlock` section at the end | The JVM detected a monitor cycle for you. Topic 94 |
| Thread count far above your configured pool sizes | A thread leak. Trap 5 |

### Proof 4 — stack size is real and tunable

```bash
java -XX:+PrintFlagsFinal -version | grep -i ThreadStackSize
```

Then observe it changing behaviour:

```java
public class Deep {
    static int depth = 0;
    static void recurse() { depth++; recurse(); }
    public static void main(String[] args) {
        try { recurse(); } catch (StackOverflowError e) {
            System.out.println("depth = " + depth);
        }
    }
}
```

```bash
java -Xss256k Deep.java
java -Xss4m   Deep.java
```

**What to look for:** the `depth` number should be substantially larger with `-Xss4m`.

| What you see | What it means |
|---|---|
| Depth scales roughly with `-Xss` | Confirmed: each thread's stack is a real, sized allocation. 1,000 threads at 1 MB each is 1 GB of reserved address space |
| Depth barely changes | Check the flag actually applied; some launchers override `-Xss` from `JAVA_TOOL_OPTIONS` |
| Wildly different numbers between runs at the same `-Xss` | Normal — JIT inlining changes frame sizes as the method gets compiled |

### Proof 5 — watch states change live

```bash
jcmd <pid> Thread.print | grep -A1 'orderflow-worker'
```

Run it while your test program is sleeping, then while it is spinning, then while it is
blocked on a `synchronized` block held by someone else. Three runs, three different
`Thread.State` lines for the same thread. That is the mapping from "what my code is
doing" to "what the dump says" and it is worth ten minutes.

---

## Failure drill

**Mandatory.** Do not read past the "How to read it" table until you have produced the
failure yourself and written down the numbers.

Topic 84 has no drill in the master plan's Section G — the drills start at 85. This one
exists because the Concurrency trace above is only knowledge until you have watched the
number come out wrong on your own machine. It is deliberately the *unfixed* version of
Topic 85's drill: here you produce the oversell, and at Topic 85 you fix it with
`synchronized` and measure what the fix costs.

### The scenario

The `orderflow` in-memory inventory cache from Example 2, driven by the Topic 65 load
generator, oversells hot SKUs. You will quantify the oversell, then confirm the
mechanism with a thread dump and JFR.

### Setup

Add a drill-only endpoint behind a profile so it cannot reach production.

`src/main/java/com/orderflow/lab/OversellDrillController.java`:

```java
package com.orderflow.lab;

import org.springframework.context.annotation.Profile;
import org.springframework.web.bind.annotation.*;
import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.atomic.AtomicLong;

@Profile("drill")
@RestController
@RequestMapping("/drill/oversell")
public class OversellDrillController {

    /** Deliberately unsafe. Plain HashMap, plain long counters. */
    private final Map<String, Integer> availableBySku = new HashMap<>();

    /** Deliberately unsafe counter — this is race #1 from Example 2. */
    private long unsafeAccepted = 0;

    /** Safe counter, so the MEASUREMENT is trustworthy even though the subject is not. */
    private final AtomicLong safeAccepted = new AtomicLong();

    @PostMapping("/reset")
    public synchronized Map<String, Object> reset(@RequestParam int stock) {
        availableBySku.clear();
        availableBySku.put("SKU-4471", stock);
        unsafeAccepted = 0;
        safeAccepted.set(0);
        return Map.of("sku", "SKU-4471", "stock", stock);
    }

    /** The endpoint k6 hammers. No synchronisation anywhere on the hot path. */
    @PostMapping("/reserve")
    public Map<String, Object> reserve() {
        Integer available = availableBySku.get("SKU-4471");     // READ
        boolean ok = available != null && available >= 1;
        if (ok) {
            availableBySku.put("SKU-4471", available - 1);      // WRITE
            unsafeAccepted++;                                   // racy
            safeAccepted.incrementAndGet();                     // trustworthy
        }
        return Map.of("reserved", ok);
    }

    @GetMapping("/state")
    public Map<String, Object> state() {
        return Map.of(
                "remainingInMap", availableBySku.get("SKU-4471"),
                "unsafeAccepted", unsafeAccepted,
                "safeAccepted",   safeAccepted.get());
    }
}
```

`k6/oversell.js`:

```javascript
import http from 'k6/http';
import { check } from 'k6';

export const options = {
  scenarios: {
    burst: {
      executor: 'constant-arrival-rate',   // OPEN model — see Topic 65
      rate: 2000,
      timeUnit: '1s',
      duration: '20s',
      preAllocatedVUs: 300,
      maxVUs: 600,
    },
  },
};

export default function () {
  const res = http.post('http://localhost:8080/drill/oversell/reserve');
  check(res, { 'status 200': (r) => r.status === 200 });
}
```

### Commands

```bash
# 1. Start orderflow with the drill profile and a flight recording.
./mvnw spring-boot:run \
  -Dspring-boot.run.profiles=load,drill \
  -Dspring-boot.run.jvmArguments="-XX:StartFlightRecording=name=oversell,filename=oversell.jfr,settings=profile,dumponexit=true"

# 2. In another terminal: seed exactly 1000 units.
curl -s -XPOST 'http://localhost:8080/drill/oversell/reset?stock=1000' | jq

# 3. Fire the load.
k6 run k6/oversell.js

# 4. Read the damage.
curl -s http://localhost:8080/drill/oversell/state | jq

# 5. While step 3 is still running, in a third terminal:
jcmd $(jcmd -l | grep orderflow | cut -d' ' -f1) Thread.print > during-load.txt
```

### What to capture

Write these five numbers down **before** reading the table:

1. `safeAccepted` — how many reservations the service believed it granted.
2. `remainingInMap` — what the map says is left.
3. `unsafeAccepted` — the racy counter's value.
4. `safeAccepted + remainingInMap` — should equal 1000 if nothing was lost.
5. From `during-load.txt`: `grep -c 'http-nio' during-load.txt` and the distribution
   from `grep 'java.lang.Thread.State' during-load.txt | sort | uniq -c`.

### How to read it

| What you see | What it means |
|---|---|
| `safeAccepted` > 1000 | **The drill has fired.** You seeded 1,000 units and granted more than 1,000 reservations. In `orderflow` terms: you have sold stock that does not exist. Note the exact overage — it is your contention rate made visible |
| `safeAccepted + remainingInMap` ≠ 1000 | Confirms lost updates. Every missing unit is one decrement that a concurrent thread overwrote |
| `unsafeAccepted` < `safeAccepted` | The racy `long` counter lost increments that the `AtomicLong` did not. Two counters, same events, different answers. **This is why you cannot trust an unguarded metric to tell you about a concurrency problem** |
| `remainingInMap` is `null` | The `HashMap` lost the key under concurrent mutation. Every subsequent request for the hot SKU is now rejected as out-of-stock while the database is full. This is Failure 3 from Example 2, and it is a revenue outage |
| `safeAccepted` is exactly 1000 and nothing is lost | The race did not land on this run. Do not conclude the code is safe. Raise `rate` to 5000, extend `duration`, and re-run. If it still does not fire, add `Thread.onSpinWait()` between the read and the write to widen the window — that is legitimate, because it changes only the *probability* of an interleaving the spec already permits, not its legality |
| Response times climb but no oversell | You may be rate-limited upstream, or Tomcat's thread pool is smaller than you think. Check `server.tomcat.threads.max` |
| In the dump: many `http-nio-8080-exec-*` threads, mostly `RUNNABLE` | Expected for this drill — there is no lock, so nobody is `BLOCKED`. **The absence of `BLOCKED` threads is the point.** A race with no lock produces no contention signal at all. Nothing in a thread dump will ever tell you about this bug |

That last row is the drill's real lesson. Compare it against Topic 85, where the same
workload *with* a `synchronized` block produces a dump full of `BLOCKED` threads and a
JFR event stream you can measure. **Adding a lock converts an invisible correctness bug
into a visible performance cost.** That trade is the entire argument for locking, and
you should be able to state it in an interview in one sentence.

### Now fix it, and re-measure

Three fixes, in increasing order of how much they actually solve.

**Fix 1 — `synchronized` on the method.** Correct, and the subject of Topic 85. It
serialises every reservation through one monitor. Re-run the drill: `safeAccepted`
should be exactly 1000, and your dump should now show `BLOCKED` threads. Record the
throughput drop.

```java
@PostMapping("/reserve")
public synchronized Map<String, Object> reserve() { /* unchanged body */ }
```

**Fix 2 — `ConcurrentHashMap.compute`.** Correct, and locks only the one hot bin
rather than the whole endpoint. Re-run: `safeAccepted` should be exactly 1000, and
throughput should be higher than Fix 1 because reservations for *different* SKUs no
longer contend with each other. In this drill there is only one SKU, so the difference
will be small — which is itself a useful finding, and worth writing down honestly.

**Fix 3 — move the invariant to the database.** The atomic conditional UPDATE from
Topic 52. This is the one that actually holds across restarts, replicas and deploys.

Write one paragraph for each fix answering: what did throughput do, what did p99 do,
and what class of failure does this fix *not* address?

---

## Measurement

### The standing rule

> **A naive `System.nanoTime()` loop is not a measurement. It is a number.**

```java
// WRONG. Every figure this produces is meaningless.
long start = System.nanoTime();
for (int i = 0; i < 10_000_000; i++) {
    inventory.tryReserve("SKU-4471", 1);
}
System.out.println((System.nanoTime() - start) / 10_000_000 + " ns/op");
```

Four independent reasons, and you cannot tell which one is lying:

1. **Dead-code elimination.** The return value is unused. C2 proves the call has no
   observable effect and deletes the loop body.
2. **On-stack replacement.** The loop starts interpreted and is replaced with compiled
   code mid-flight. Your average blends interpreter, C1 and C2 in a ratio decided by
   the loop count you happened to pick.
3. **Profile pollution.** Benchmarking implementation A first makes the call site
   monomorphic; benchmarking B afterwards in the same JVM measures a megamorphic site.
   Both numbers are wrong, in opposite directions. Topic 74.
4. **In a concurrency benchmark specifically:** thread start-up cost, scheduler warm-up
   and cache-line migration all land inside your window.

Topic 77 is the full treatment. Forward-reference it now and never quote a `nanoTime`
number in a design discussion.

### JMH with `@Threads` — the correct shape for contention

Contention is a *function of thread count*. A single-threaded benchmark of a
`synchronized` method measures the uncontended fast path and tells you nothing about
production. The parameter you vary is the number of threads.

```java
package com.orderflow.bench;

import org.openjdk.jmh.annotations.*;
import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.TimeUnit;

@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.SECONDS)
@Warmup(iterations = 5, time = 1)
@Measurement(iterations = 10, time = 1)
@Fork(value = 3)                          // separate JVMs: no profile pollution
@State(Scope.Benchmark)                   // ONE instance shared by all threads
public class ReservationContentionBenchmark {

    @Param({"1", "2", "4", "8", "16", "32"})
    int threadHint;                       // documentation only; see the run command

    private final Map<String, Integer> plainMap = new HashMap<>();
    private final Map<String, Integer> concurrentMap = new ConcurrentHashMap<>();
    private final Object lock = new Object();

    @Setup(Level.Iteration)
    public void seed() {
        plainMap.put("SKU-4471", Integer.MAX_VALUE);
        concurrentMap.put("SKU-4471", Integer.MAX_VALUE);
    }

    /** Baseline: uncontended-shaped work with no sharing at all. */
    @Benchmark
    public int localOnly() {
        int available = Integer.MAX_VALUE;
        return available - 1;
    }

    /** Correct via a single monitor. Measures pure lock contention. */
    @Benchmark
    @Threads(Threads.MAX)
    public int synchronizedReserve() {
        synchronized (lock) {
            int available = plainMap.get("SKU-4471");
            plainMap.put("SKU-4471", available - 1);
            return available;
        }
    }

    /** Correct via per-key atomicity. Measures striped contention. */
    @Benchmark
    @Threads(Threads.MAX)
    public int chmReserve() {
        return concurrentMap.compute("SKU-4471", (k, v) -> v - 1);
    }
}
```

Run it:

```bash
# build the benchmark jar (jmh-maven-archetype layout)
mvn clean verify

# all benchmarks, default threads
java -jar target/benchmarks.jar ReservationContentionBenchmark

# sweep thread counts explicitly — this is the measurement that matters
java -jar target/benchmarks.jar ReservationContentionBenchmark \
     -t 1 -t 2 -t 4 -t 8 -t 16 -t 32 \
     -rf json -rff contention.json

# add a GC profiler to see allocation, and perfasm if you have perf
java -jar target/benchmarks.jar ReservationContentionBenchmark -prof gc
```

**What to look for:** how throughput changes as `-t` rises.

| What you see | What it means |
|---|---|
| `localOnly` scales close to linearly with threads | Your machine really does have that many usable cores, and the harness is sound. This is your control |
| `synchronizedReserve` throughput is flat or falls as threads rise | Correct and expected. A single monitor serialises everything; adding threads adds contention, not throughput. **Falling** throughput means you are paying park/unpark on top of serialisation |
| `chmReserve` also flat with one hot key | Also expected — one key is one bin, so `ConcurrentHashMap` gives you no striping benefit here. Add a `@Param` over key count to see the striping appear |
| Enormous variance between forks | Look at the per-fork numbers, not the mean. High variance in a contention benchmark usually means scheduler noise or thermal throttling on a laptop. Close everything else and re-run |
| A number that looks too good | Suspect dead-code elimination. Return a value from every benchmark method, or consume it with a `Blackhole` |

### JFR — the events that matter for this topic

```bash
# Start recording at launch
java -XX:StartFlightRecording=name=orderflow,filename=of.jfr,settings=profile,dumponexit=true -jar app.jar

# Or attach to a running JVM
jcmd <pid> JFR.start name=orderflow settings=profile filename=of.jfr
jcmd <pid> JFR.dump  name=orderflow filename=of.jfr
jcmd <pid> JFR.stop  name=orderflow

# Read it
jfr summary of.jfr
jfr print --events jdk.ThreadStart,jdk.ThreadEnd of.jfr | head -50
jfr print --events jdk.JavaThreadStatistics of.jfr
```

| Event | What it answers |
|---|---|
| `jdk.ThreadStart` / `jdk.ThreadEnd` | Are threads being created per request? Count starts over the run |
| `jdk.JavaThreadStatistics` | Live and peak thread count over time — the leak signal from Trap 5 |
| `jdk.ThreadSleep` | Someone put a `sleep` on a request path |
| `jdk.ThreadPark` | Threads parked in `LockSupport` — `ReentrantLock`, queues, `CompletableFuture` |
| `jdk.JavaMonitorEnter` | **`synchronized` contention.** Topic 85's core measurement — noted here so you know where it lives |
| `jdk.ExecutionSample` | Where CPU actually goes. Topic 78 |

`jfr summary` first, always. It gives you the event counts, which tells you which
`jfr print` is worth running. Printing `jdk.ExecutionSample` on a 60-second profile
recording without a filter will bury your terminal.

### The two things to measure for *this* topic

1. **Thread count over time.** `jdk.JavaThreadStatistics`, or in a pinch:
   ```bash
   while true; do
     echo "$(date +%s) $(jcmd <pid> Thread.print | grep -c 'java.lang.Thread.State')"
     sleep 5
   done
   ```
   A monotonically rising number is a leak. A stable number is a bounded pool. That is
   the whole diagnosis.

2. **Correctness under load, not latency under load.** The drill above measures
   `safeAccepted + remainingInMap` against the seed. **For concurrency work, the first
   measurement is always an invariant, not a percentile.** A fast wrong answer is not a
   result. Establish correctness, then measure the cost of correctness — that ordering
   is the difference between engineering and benchmarking.

---

## Practice exercises

Write real files. Run them. Write down what you saw, including when it was boring.

### 1 — Easy: build the state table yourself

Write one program that drives a single thread into **five** of the six `Thread.State`
values and prints the observed state from a second, observing thread. `NEW` and
`TERMINATED` are easy. For the others you need:

- `RUNNABLE` — a spin loop with `Thread.onSpinWait()`.
- `BLOCKED` — a `synchronized` block whose monitor the main thread already holds.
- `WAITING` — `Object.wait()` with no timeout, or `LockSupport.park()`.
- `TIMED_WAITING` — `Thread.sleep(10_000)`.

Requirements:

- Print the observed state as `worker.getState()` from the main thread.
- For each state, also take a thread dump (`jcmd <pid> Thread.print`) and record how
  that state appears in the dump text — the exact wording differs from the enum name.
- Answer in writing: **why does a thread blocked reading a socket show as `RUNNABLE`?**
  What does that mean for anyone diagnosing "high CPU" from a thread dump alone?

### 2 — Medium: the audit (combines Topics 01–83)

Below is a fragment from an `orderflow` background service. It contains **seven**
distinct defects drawn from this topic and from Phases 1, 5 and 8. Find them all. For
each, state the **exact symptom an on-call engineer observes** — not "it's bad
practice" — and then rewrite the class correctly.

```java
package com.orderflow.settlement;

import org.springframework.stereotype.Service;
import java.text.SimpleDateFormat;
import java.util.*;
import java.util.concurrent.*;

@Service
public class SettlementService {

    private final SimpleDateFormat stamp = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
    private final Map<Long, Long> settledAmountByWallet = new HashMap<>();
    private long totalSettled = 0;
    private boolean running = true;

    public void startWorker() {
        Thread worker = new Thread(() -> {
            while (running) {
                try {
                    List<Payment> batch = fetchPending();
                    for (Payment p : batch) {
                        settle(p);
                    }
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    // nothing to do here
                }
            }
        });
        worker.run();
    }

    public void stopWorker() {
        running = false;
    }

    private void settle(Payment p) {
        Long current = settledAmountByWallet.get(p.walletId());
        settledAmountByWallet.put(p.walletId(),
                current == null ? p.amount() : current + p.amount());
        totalSettled += p.amount();
        audit("settled " + p.id() + " at " + stamp.format(new Date()));
    }

    public void settleAllAsync(List<Payment> payments) {
        ExecutorService pool = Executors.newFixedThreadPool(8);
        for (Payment p : payments) {
            pool.submit(() -> settle(p));
        }
    }

    private void audit(String message) { /* writes to a log */ }
    private List<Payment> fetchPending() { return List.of(); }
}
```

Hints, in the order the defects appear conceptually:

- One defect means the "background worker" is not in the background at all.
- One defect means `stopWorker()` may never stop anything, ever. Name the topic
  number where that is proven.
- One defect makes shutdown hang for the full Kubernetes grace period.
- Two defects are shared mutable state with no guard — one of them is a `Map`, and one
  of them is a `long` that also loses updates.
- One defect is a class from Phase 1 that is documented as not thread-safe and will
  produce garbage dates or an exception under concurrency. Name it and name its
  replacement.
- One defect leaks threads and swallows every exception the tasks throw.

Deliverable: the corrected class, plus a one-line justification per change. If two
defects have the same fix, say so.

### 3 — Hard: production simulation on `orderflow` under load

**Part A — reproduce.** Run the Failure drill above at Topic 65's load profile. Record
`safeAccepted`, `remainingInMap`, `unsafeAccepted` and the overage.

**Part B — characterise.** Sweep the arrival rate: 500, 1000, 2000, 4000 requests per
second, seeding 1,000 units each time. Plot overage against rate. Answer: **is the
relationship linear?** Then explain *why* the shape is what it is, in terms of how many
nanoseconds wide the read-write window is versus how many requests arrive per
nanosecond.

**Part C — the counter that lies.** Compute `(safeAccepted - unsafeAccepted) /
safeAccepted` at each rate. That is the fraction of increments the unguarded `long`
counter lost. Write two sentences you could put in a postmortem explaining why the
service's own metrics under-reported the incident.

**Part D — fix three ways and measure.** Apply `synchronized`, then
`ConcurrentHashMap.compute`, then the Topic 52 atomic conditional UPDATE. For each,
record throughput and p99 from k6, and confirm `safeAccepted + remainingInMap == 1000`
exactly. Use JMH for the in-JVM comparison and k6 for the end-to-end numbers, and say
in one sentence why you needed both.

**Part E — argue the other side.** Under what circumstances would you ship the
*unsynchronised* in-memory cache deliberately? There is a real answer involving the
word "hint", the phrase "fails safe", and the fact that the database enforces the
invariant anyway. Make the strongest case for it, then state the one condition that
makes it indefensible.

**Part F — the honest limit.** Your drill oversold. Does that prove your fixed version
is correct? Write down why not, in one sentence, and name the topic that gives you a
better tool.

---

## Interview questions

### Q1 — "You come from Node. What is the single biggest difference when you write concurrent Java?"

**Mid-level answer:** "Java has real threads, so you can use multiple cores, and you
have to be careful about shared data. Node is single-threaded so you don't."

**Senior answer:** "The event loop guarantees run-to-completion: once a callback
starts, nothing else runs until my stack unwinds. That guarantee is why I've never
written a lock in JavaScript. Java has no such guarantee — the OS scheduler can suspend
a thread between any two bytecodes, including between reading a field and writing it
back. And Java threads share one heap, where `worker_threads` have isolated heaps and
communicate by copying. So in Node, sharing is an explicit opt-in via
`SharedArrayBuffer`; in Java, sharing is the default and isolation is the work.

Practically, that means the first thing I look at in an unfamiliar Java service is
mutable state on singleton beans, because Spring MVC is thread-per-request and every
field on a singleton is touched by every request thread at once. That's the bug class
that catches people coming from Node, and it doesn't look like a concurrency bug in
review — it looks like an ordinary field."

**What separates them:** naming *run-to-completion* as the thing that is missing rather
than "you have to be careful"; naming the shared-heap-versus-isolated-heap inversion;
and converting it into a concrete review heuristic rather than a general warning.

**Follow-up the interviewer asks:** "Where in a Spring app does that bite first?" They
want singleton beans with mutable fields, and bonus points for naming a specific
non-thread-safe class — `SimpleDateFormat` is the classic.

---

### Q2 — "This background worker never stops. Walk me through why."

```java
private boolean running = true;
void start() { new Thread(() -> { while (running) { poll(); } }).start(); }
void stop()  { running = false; }
```

**Mid-level answer:** "`running` isn't `volatile`, so the other thread might not see
the change. Adding `volatile` fixes it."

**Senior answer:** "Two separate things are wrong and it's worth separating them.

The visibility problem is the one you named, but I'd be precise about the mechanism:
without a happens-before edge, the JIT is entitled to hoist the read of `running` out
of the loop, because in a single-threaded reading of that code nothing inside the loop
modifies it. So the compiled loop tests a register that was loaded once. It's not that
the write is slow to propagate — it's that the read no longer happens at all. That's
why the failure is total and permanent rather than delayed, and it's why 'eventually'
is not in the specification.

The way I'd prove it rather than assert it: run the same program with `-Xint`. In the
interpreter there is no hoisting, so it terminates. Run it normally with a warmed loop
and it hangs. That gap is the memory model, demonstrated in two commands.

`volatile` does fix it. But if I'm writing this fresh I'd rather not have a spin loop
at all — a `CountDownLatch`, or an interruptible `BlockingQueue.take()` with proper
`InterruptedException` handling, expresses the intent and doesn't burn a core."

**What separates them:** the mid-level answer knows the fix; the senior answer names
*hoisting* as the mechanism, offers the `-Xint` A/B as proof, distinguishes "slow to
propagate" from "never read again", and then questions the design rather than only
patching it.

**Follow-up:** "If `poll()` were a `synchronized` method, would the loop still hang?"
This is genuinely subtle. Entering a monitor is a happens-before edge, so in practice
the hoist is prevented and it may well terminate — but relying on that is relying on an
implementation accident rather than a guarantee you asked for. The answer they want is
"it might work and I would not ship it."

---

### Q3 — "What does `Thread.interrupt()` actually do?"

**Mid-level answer:** "It stops the thread."

**Senior answer:** "It sets a boolean flag on the target thread. That's all it does
directly. Two things follow.

If the target is blocked in an interruptible method — `sleep`, `wait`, `join`,
`BlockingQueue.take`, `lockInterruptibly`, most NIO channel operations — that method
throws `InterruptedException` and **clears** the flag. If the target is running normal
code, nothing happens automatically; it's that thread's job to check
`isInterrupted()`.

The clearing is where the bugs are. If you catch `InterruptedException` and don't
either rethrow or call `Thread.currentThread().interrupt()`, you've destroyed the
cancellation signal. The symptom is a service that won't shut down: `awaitTermination`
times out, the pod sits in `Terminating` for the whole grace period, and every deploy
takes ten minutes. It's a five-minute grep to audit a codebase for it.

And interruption is cooperative, not forceful. It won't unblock a `synchronized`
acquisition or a plain socket read. If cancellation genuinely has to work, the answer
is usually a timeout on the resource, not a way to kill the thread."

**What separates them:** knowing it is a flag rather than an action; knowing the
exception *clears* the flag; naming the shutdown-hang symptom; and knowing the limits
of what interruption can unblock.

**Follow-up:** "Why was `Thread.stop()` removed?" See Q4.

---

### Q4 — "Why can't you just kill a thread in Java?"

**Mid-level answer:** "`Thread.stop()` is deprecated because it's unsafe."

**Senior answer:** "Because there is no safe point at which to do it, and the failure
mode is silent corruption rather than an error.

`stop()` threw a `ThreadDeath` error at the target at an arbitrary bytecode. So it
could land in the middle of a multi-field update — an `Order` whose total was updated
but whose lines weren't. Worse, it **released every monitor the thread held**
immediately. So another thread would enter a `synchronized` block, correctly believing
itself protected, and read a broken invariant. The corruption propagates into code
that did nothing wrong, and it's unrecoverable because you cannot write a `finally`
that copes — the error can arrive inside the `finally` too.

`suspend()` was unsafe in the opposite direction: it stopped the thread *holding* its
locks, so anyone needing those locks blocked forever. That's a deadlock with no cycle,
so the JVM's deadlock detector doesn't report it.

Both were degraded to throw `UnsupportedOperationException` in JDK 20. The replacement
is cooperative cancellation: `interrupt()` plus code that checks. And if the work
genuinely can't be interrupted, the honest engineering answer is a timeout at the
resource — a socket read timeout, a statement timeout — rather than a bigger hammer at
the thread level."

**What separates them:** the *specific* mechanism (monitors released mid-invariant, and
corruption spreading to correct code), the distinction between `stop` and `suspend`'s
failure modes, and the fact that suspend-deadlock is invisible to the deadlock
detector.

**Follow-up:** "How would you cancel a thread blocked on a JDBC query?" The answer is
`Statement.setQueryTimeout` or a connection-level timeout — not thread manipulation.
They are checking whether you solve problems at the right layer.

---

### Q5 — "How many threads should this service run?"

**Mid-level answer:** "Number of cores, or number of cores times two."

**Senior answer:** "That rule is for CPU-bound work, and `orderflow` isn't CPU-bound —
it's mostly waiting on Postgres.

For an I/O-bound service I'd start from Little's Law: threads needed ≈ target
throughput × average latency. At 1,000 requests per second and 50 ms average service
time, that's about 50 concurrently in flight. Then I'd check that against the real
constraint, which is almost never threads — it's the connection pool. If Hikari is
sized at 20, then 200 Tomcat threads just means 180 threads queueing for a connection,
which converts a capacity problem into a latency problem and makes the thread dump
harder to read.

I'd also be explicit about what a thread costs: about 1 MB of reserved stack by
default, a `pthread_create` to make, and scheduler plus cache-pressure cost to run.
Thousands of them is real overhead, and at some OS-dependent limit you get
`OutOfMemoryError: unable to create native thread`, which is an OOM a heap dump won't
explain.

And I'd say what changes the answer entirely: virtual threads. On JDK 21+ with
`spring.threads.virtual.enabled=true`, the thread count stops being the constraint for
blocking I/O — but the downstream capacity doesn't change, so the queue just moves. If
the bottleneck was a 20-connection pool, virtual threads don't make it faster; they
make it a different shape. That's Topic 101."

**What separates them:** Little's Law rather than a folk ratio, identifying the
connection pool as the real constraint, quoting the per-thread cost with the specific
OOM message, and knowing that virtual threads relocate the bottleneck rather than
removing it.

**Follow-up:** "Your thread dump shows 200 threads all `RUNNABLE`. Is the CPU
saturated?" No — `RUNNABLE` includes threads blocked in kernel I/O. You need a
wall-clock profiler (Topic 78) or `top -H` to tell.

---

## Mental model checkpoint

Reason these out. Do not look them up. Write your answers down; several of them are
questions you will be asked in an interview almost verbatim.

1. The event loop guarantees run-to-completion. Java guarantees nothing of the kind.
   Name one class of bug that Node therefore *cannot* have, and one class of bug that
   Node *can* have and Java's preemption makes no worse. What does that tell you about
   which of your existing debugging instincts still transfer?

2. `stock = stock - 1` compiles to `getfield`, arithmetic, `putfield`. If the JVM added
   a single atomic-decrement bytecode tomorrow, would that fix the oversell in the
   Concurrency trace? Answer carefully — there are two independent problems and only
   one of them is atomicity.

3. A thread blocked reading from a socket is `RUNNABLE`. A thread waiting for a
   `synchronized` monitor is `BLOCKED`. A thread waiting on a `ReentrantLock` is
   `WAITING`. Explain why those three are in three different states, in terms of *who*
   is doing the waiting — the kernel, the JVM, or the JDK library.

4. The Failure drill produces a thread dump with **no** `BLOCKED` threads, because
   there is no lock. Adding `synchronized` creates `BLOCKED` threads. Argue that this
   makes the service *more* observable even though it also makes it slower. Then argue
   the opposite: name a situation where the lock's contention signal actively misleads
   you about where the real problem is.

5. `Thread.stop()` released the thread's monitors on the way out. Suppose it had
   instead *kept* them held forever. Would that have been safer or more dangerous?
   Justify your answer in terms of which failure is easier to diagnose.

6. Node scales out with `cluster` — one process per core, nothing shared. Java scales
   up — one process, many threads, everything shared. Both approaches ship real systems.
   State the operational cost of each honestly, including one thing the Node model makes
   *harder* that Java's makes easy.

7. You are reviewing a PR that adds a `private final Map<String, BigDecimal>
   priceCache = new HashMap<>();` field to a `@Service` class. The author says "it's
   read-only after startup, so it's fine". Under what precise conditions is that
   author correct? Name the topic number that supplies the guarantee they are implicitly
   relying on, and what would break it.

---

## Quick reference card

### Happens-before edges — the complete list

You will memorise this at Topic 86. It is here so you have it from the start, and so
you can see how few of them there are.

| Edge | The rule |
|---|---|
| **Program order** | Within a single thread, each action happens-before every action that comes later in program order |
| **Monitor lock** | An unlock of a monitor happens-before every subsequent lock of *that same* monitor |
| **Volatile** | A write to a `volatile` field happens-before every subsequent read of *that same* field |
| **Thread start** | `Thread.start()` happens-before any action in the started thread |
| **Thread join** | Every action in a thread happens-before another thread returns from `join()` on it |
| **Thread termination** | Every action in a thread happens-before any other thread detects it has terminated (`isAlive()` returning false, `join()` returning) |
| **Interruption** | A call to `interrupt()` happens-before the interrupted thread detects the interrupt |
| **Final fields** | The end of a constructor happens-before the freeze of its `final` fields — Topic 88 |
| **Default values** | The write of the default value (zero/null/false) to a field happens-before the first action in any thread |
| **Transitivity** | If A happens-before B and B happens-before C, then A happens-before C |

The library edges you will actually use follow from these: `ExecutorService.submit`
happens-before the task runs, task completion happens-before `Future.get()` returns,
`CountDownLatch.countDown()` happens-before `await()` returns, and putting into a
`BlockingQueue` happens-before taking that element out.

**In this topic you used exactly two:** `Thread.start()` and `Thread.join()`. Both of
the minimal examples above are correct *only* because of those edges.

### Flags

| Flag | What it does | When you use it |
|---|---|---|
| `-Xss<size>` | Per-thread stack size (default typically 1 MB on 64-bit) | Deep recursion; or reducing per-thread footprint at very high thread counts |
| `-XX:+PrintFlagsFinal -version` | Print every flag's effective value | Always, before quoting any default |
| `-Xint` | Interpreter only, no JIT | The A/B control for Topic 86. Also useful here: races behave differently without compiler optimisation |
| `-XX:-TieredCompilation` | C2 only, no C1 tier | Reduces the warm-up phase in experiments |
| `-XX:+UnlockDiagnosticVMOptions` | Enables diagnostic flags | Prerequisite for several of the above |
| `-XX:StartFlightRecording=...` | Start JFR at launch | Every drill from here on |
| `-XX:+UseCompactObjectHeaders` | `[JAVA 25]` 64-bit object headers | Verify before quoting mark-word layout — Topic 85 |
| `-Djdk.tracePinnedThreads=full` | Report virtual-thread pinning | Topic 101 |

### Diagnostic commands

| Command | What it answers |
|---|---|
| `jcmd -l` | Which JVMs are running, and their pids |
| `jcmd <pid> Thread.print` | Full thread dump with monitor ownership. **Taken at a safepoint** |
| `jstack <pid>` | The same thing, older entry point |
| `jcmd <pid> Thread.print \| grep 'java.lang.Thread.State' \| sort \| uniq -c` | State distribution in one line |
| `jcmd <pid> Thread.print \| grep -c 'java.lang.Thread.State'` | Live Java thread count — run repeatedly to spot a leak |
| `jcmd <pid> VM.flags` | The flags this JVM is actually running with |
| `jcmd <pid> VM.uptime` | How long it has been up, for correlating with load |
| `jcmd <pid> JFR.start settings=profile filename=x.jfr` | Begin a flight recording on a live JVM |
| `jfr summary x.jfr` | Event counts. **Always run this first** |
| `jfr print --events jdk.ThreadStart x.jfr` | Thread creation timeline |
| `jfr print --events jdk.JavaThreadStatistics x.jfr` | Live and peak thread counts |
| `top -H -p <pid>` (Linux) | Per-thread CPU. Match `nid` from the dump (hex) to the OS tid (decimal) |
| `ps -M <pid>` (macOS) | Per-thread view on macOS |
| `javap -c <Class>.class` | Prove that a statement is several bytecodes |

### The reading checklist for a thread dump

- [ ] What is the state distribution? (`sort | uniq -c`)
- [ ] Is there a `Found one Java-level deadlock` section? (Read it first if so)
- [ ] Are many threads `BLOCKED` on **one** monitor address? Find the `- locked` owner
- [ ] Are threads `RUNNABLE` deep in socket or file I/O? That is downstream, not CPU
- [ ] Is the thread count higher than the sum of your configured pool sizes?
- [ ] Are threads named after their job, or are they `pool-3-thread-7`?

### Gotchas checklist

- [ ] `start()`, never `run()`.
- [ ] Every `catch (InterruptedException)` either rethrows or restores the flag.
- [ ] `Thread.stop` / `suspend` / `resume` are degraded to throw. Never reach for them.
- [ ] Uncaught exceptions kill one thread silently. Set a default handler.
- [ ] `submit()` swallows exceptions into the `Future`; `execute()` does not.
- [ ] A `ScheduledExecutorService` task that throws is never rescheduled again.
- [ ] Pools are beans with a `@PreDestroy`, never local variables.
- [ ] Non-daemon threads keep the JVM alive after `main` returns.
- [ ] Every mutable field on a singleton bean is shared by every request thread.
- [ ] `RUNNABLE` does not mean "on a CPU".
- [ ] `BLOCKED` means `synchronized`; `WAITING` means parked.
- [ ] "It will see it eventually" is not in the specification.

---

## When would I use this at work?

**1. Reviewing a pull request that adds a field to a Spring `@Service`.**

This is the highest-frequency application of Topic 84 and it costs you nothing once the
pattern is in your eye. Every new mutable field on a singleton bean is a question:
*which threads touch this?* If the answer is "all 200 request threads", the field needs
to be immutable, thread-confined, or guarded — and the author almost never realised
there was a question. You will catch `SimpleDateFormat`, `HashMap` caches, `long`
counters and accumulating `StringBuilder`s, and each one is a real incident you
prevented.

**2. Triaging a service that will not shut down.**

The pod sits in `Terminating` on every deploy, and the platform team is asking why the
rolling update takes ten minutes. You go straight to `jcmd <pid> Thread.print`, look
for non-daemon threads that are still `RUNNABLE` or `TIMED_WAITING`, and then grep the
source for `catch (InterruptedException`. Nine times out of ten it is a swallowed
interrupt in a polling loop. Fifteen minutes, and the fix is two lines.

**3. Sizing a service in a design review, before it exists.**

Someone proposes thread-per-request at 5,000 concurrent connections. You can say, from
first principles: that is 5,000 platform threads at ~1 MB reserved stack each, plus
scheduler overhead, and the real ceiling is not threads but the 20-connection database
pool behind them — so the design is queueing 4,980 threads to look busy. Then you can
offer the two real options: bound the concurrency with a semaphore and shed load
honestly, or move to virtual threads and accept that the queue relocates to the pool.
Being able to make that argument with numbers rather than opinions is what a senior
engineer is paid for.

---

## Connected topics

**Prerequisites — what you are standing on:**

- **17 — Immutability, `final`, safe publication.** You learned there that `final` has
  memory-model meaning. Topic 88 is the promised payoff, and Topic 84 is why it was
  needed: without threads, `final` would be nothing but "cannot reassign".
- **66 — JVM architecture.** Per-thread stack and PC register versus shared heap and
  metaspace. That diagram is the structural basis of everything above.
- **68 — TLABs.** Each thread gets its own allocation buffer, which is the JVM
  already applying thread-confinement as an optimisation. Now you know why it had to.
- **69 — Object layout and the mark word.** Topic 85 uses that same mark word to store
  lock state.
- **73 — Safepoints.** Thread dumps are taken at safepoints; a counted loop may have no
  safepoint poll, which is the same mechanism that lets Topic 86's loop never re-read a
  field.
- **74–75 — The JIT.** The compiler's licence to hoist a read out of a loop *is* the
  Topic 86 bug. Deoptimisation and escape analysis are why `-Xint` behaves differently.
- **77 — JMH.** Why your `nanoTime` loop is not a measurement.
- **78 — Profiling.** Wall-clock versus CPU mode is exactly the `RUNNABLE`-does-not-mean-
  running distinction.
- **65 — The load gate.** Every drill in this phase runs against that baseline.

**This unlocks:**

- **85 — `synchronized` and monitors.** The first tool that gives you back a slice of
  run-to-completion, and the mark word that makes it cheap when uncontended.
- **86 — The JMM I: happens-before.** Why even *correctly ordered* operations give no
  guarantee without an edge. The stop-flag drill is Trap 4 above, proven.
- **87 — The JMM II: `volatile` and barriers.** What the edge costs in CPU instructions,
  and why `volatile i++` still loses increments.
- **88 — `final` fields and safe publication.** Why a reader can see a non-null
  reference to a half-built object, and the freeze that prevents it.
- **89 — `wait`/`notify`.** The coordination primitive built on the monitor.
- **90 — Executors.** Where you stop creating threads by hand. Pool sizing, bounded
  queues, rejection policy, and the shutdown sequence that Trap 3 breaks.
- **92 — `ConcurrentHashMap`.** The correct home for Example 2's map, and the lesson
  that atomic operations do not compose.
- **94 — Explicit locks.** `tryLock` with a timeout, interruptibility, and why a
  `ReentrantLock` waiter shows as `WAITING` rather than `BLOCKED`.
- **95 — Atomics and `LongAdder`.** The correct home for Example 2's counters.
- **98 — Bug taxonomy.** Given a symptom, name the class of bug. The thread-leak drill
  is Trap 5 above.
- **99 — jcstress.** The honest answer to "how do I know my fix is correct", and the
  reason a passing test loop proves nothing.
- **101 — Virtual threads.** What happens when a thread stops being an OS thread, and
  which of the facts above survive that change. (Most of them do. Preemption and the
  shared heap both survive.)
- **102 — Structured concurrency.** Making a thread's lifetime a lexical property, so
  Trap 5 becomes unwriteable.

---

*Java baseline 21. The `Thread.ofPlatform()` / `ofVirtual()` builder API is Java 21.
`Thread.stop`/`suspend`/`resume` were degraded to throw `UnsupportedOperationException`
in JDK 20 — verify on your JDK with the snippet in Trap 2 rather than trusting any
document, including this one. `[JAVA 25]` compact object headers change the object
header layout but nothing in this topic depends on it. Everything else here has been
true since Java 5 and will still be true in a decade, because it is a statement about
operating systems, not about Java.*
