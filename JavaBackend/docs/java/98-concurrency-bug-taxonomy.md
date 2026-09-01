# 98 — Concurrency Bug Taxonomy: Races, Deadlock, Livelock, Starvation, Thread Leaks

## Phase: 9 — Concurrency
## Category: ELITE
## Java baseline: 21  |  Notes features from: 21 / 25
## Project spine: `orderflow` at the Topic 65 baseline, hung. This is the diagnostic capstone of Phase 9: given a service that has stopped responding, you name which of five bugs it is and prove it — from evidence, in minutes, without restarting the pod.

---

## Before anything else — what is and is not in this document

**I do not have a JVM. I have not run `orderflow`, `jcmd`, JFR, or k6. Nothing here is
captured output.**

You will not find:

- a real thread dump I took,
- a thread count, a heap figure, a latency number, or a GC pause presented as measured,
- a "we saw 4,000 threads" claim,
- any number offered as evidence of anything.

Every claim that requires running something is given as the **exact command**, **WHAT TO
LOOK FOR**, and a **"what you see → what it means"** table.

### The one labelled exception, and why this document needs it more than any other

**Learning to read the deadlock section of a thread dump is the point of this topic.** You
cannot learn to read a format you have never seen. So this document shows the **structure**
of `jcmd <pid> Thread.print` output — including the `Found one Java-level deadlock:` header,
the `waiting to lock monitor ... which is held by` lines, and the frame ordering — with
every value replaced by `<tid>`, `0x...`, `<n>` or a name from the `orderflow` domain.

Each such block carries the inline label:

> *illustration of the format, not captured output*

**Placeholders only. Never a plausible-looking number.** The shape is real — it is the JVM's
documented output format, stable for many years. The values are not, and you will fill them
in from your own machine in the Hands-on proof.

### What I will assert without a command

That the JVM detects and reports monitor and `Lock` deadlocks but never breaks one; that
`ThreadMXBean.findDeadlockedThreads()` cannot see a cycle through a `Semaphore` or a
connection pool; that `OutOfMemoryError: unable to create native thread` means OS thread
exhaustion rather than heap exhaustion. These are stable platform behaviours, each with a
confirming command below.

---

## Mechanical statement

Read this twice. Everything below is elaboration.

> **Deadlock requires all four Coffman conditions to hold simultaneously:**
>
> 1. **Mutual exclusion** — a resource is held exclusively; only one thread at a time.
> 2. **Hold and wait** — a thread holding one resource waits for another.
> 3. **No preemption** — a resource cannot be forcibly taken from its holder.
> 4. **Circular wait** — a cycle of threads, each waiting for a resource the next holds.
>
> **You break deadlock by removing exactly one.** All four must hold; remove any single one
> and deadlock becomes impossible. **Lock ordering removes circular wait**, which is why it
> is the primary fix. `tryLock(timeout)` removes no-preemption. Taking all locks at once, or
> none, removes hold-and-wait. Immutable data removes mutual exclusion.
>
> And the sentence this entire document exists to install:
>
> **"The service is hung" is not a diagnosis. It is four different diagnoses that produce
> the same customer-visible symptom, and one thread dump separates them.**

| Symptom | Diagnosis | Thread state signature |
|---|---|---|
| threads stuck in a cycle, CPU idle | **deadlock** | `BLOCKED` + `Found one Java-level deadlock` |
| 100% CPU, no progress | **livelock** | `RUNNABLE`, same stacks, `cpu=` climbing |
| one consumer never scheduled | **starvation** | one thread `WAITING`/`RUNNABLE` for ever while peers progress |
| thread count only grows | **thread leak** | thread count monotonic, then `unable to create native thread` |

That table is the whole topic. The rest of the document teaches you to fill in each row
with evidence rather than with a guess.

---

## The bridge from what you know

### `NO TYPESCRIPT ANALOGUE.`

Say it plainly, because reaching here costs more than it buys.

**A single-threaded runtime cannot deadlock.** Node's event loop gives you
run-to-completion: once a synchronous function starts, nothing else in the process runs
until it returns. Two callbacks cannot be half-finished at the same moment, so there is no
state in which each holds something the other needs. You have never written a mutex, so you
have never had a lock cycle. You cannot have had one.

The same argument removes three of the other four bugs:

| Bug | Why it cannot happen in Node | Why it happens in Java |
|---|---|---|
| **Data race** | one thread; no two writers to one variable at one instant | shared heap, preemption between any two bytecodes |
| **Deadlock** | no locks, no hold-and-wait | two threads, two locks, opposite orders |
| **Livelock** | no retry-against-another-thread; nothing to collide with | two threads politely backing off in lockstep |
| **Starvation** | the loop is FIFO; a queued callback always runs eventually | preemptive scheduling, lock barging, thread-priority effects |
| **Thread leak** | **this one has a cousin** — see below | unreferenced pools whose non-daemon threads never exit |

**Verdict: NO ANALOGUE for four of the five. Do not carry the event loop into this
document.**

### The one thing that genuinely rhymes, and its exact limits

You *have* had a resource leak: an event listener added per request and never removed, an
open handle never closed, a `setInterval` never cleared. Node's process eventually fails
with `EMFILE: too many open files` or grows until the container OOM-kills it.

**A Java thread leak is that shape, with two differences that change the diagnosis
completely:**

1. **A leaked thread has a 1 MB stack reservation and an OS-level kernel object**, not just
   a heap entry. You run out of a *system* resource long before you run out of heap, and the
   error message says so: **`java.lang.OutOfMemoryError: unable to create native thread`** —
   an `OutOfMemoryError` that has nothing to do with your heap. Adding `-Xmx` makes it
   **worse**, because heap and thread stacks compete for the same process address space.
2. **A non-daemon thread keeps the JVM alive.** A leaked `ExecutorService` nobody shut down
   holds threads that prevent process exit. Your application "finishes" and the process
   hangs. Node has no equivalent — an unreferenced object is simply collected.

So: bring your leak intuition, and rename the resource. **Everything else in this document
is new.**

### What you must actively unlearn

| Habit | Safe in Node because | A production incident in Java because |
|---|---|---|
| "if it's hung, restart it" | restarting is cheap and the state is small | **restarting destroys the only evidence.** The same incident recurs next week. |
| "high CPU means it's working hard" | mostly true | livelock is 100% CPU and zero progress |
| "no errors in the log means no bug" | mostly true | deadlock, livelock and starvation log **nothing at all** |
| "the profiler will show me" | it usually does | a deadlocked thread uses no CPU and appears nowhere in a flame graph |
| "`OutOfMemoryError` means increase the heap" | not a Node concept | `unable to create native thread` gets *worse* with a bigger heap |

---

## What is this?

Five distinct failure classes. They are not degrees of one thing; they have different
causes, different evidence, and different fixes.

### 1. Data race

Two threads access the same variable, at least one writes, and there is no happens-before
edge (Topic 86). The outcome depends on interleaving.

**What it looks like:** wrong numbers. An oversold product. A wallet debited twice. A
counter that is slightly low. It does **not** hang and it does **not** throw. The system
stays up and produces incorrect results, which is worse.

**How you find it:** you do not find it in a thread dump. You find it by reasoning about
happens-before, by jcstress (Topic 99), or — most often — by a customer noticing.

**This is the one bug in this document with no runtime diagnostic**, which is why Topics
86–88 and 99 exist and why this document's decision table has four rows rather than five.

### 2. Deadlock

Two or more threads each hold a resource and wait for one another's, forming a cycle.
**Permanent.** No timeout, no exception, no recovery.

**What it looks like:** requests stop completing. CPU drops. Latency dashboards go *flat*,
not up, because requests that never complete never record a latency. The load balancer
returns 503; the application logs are silent.

**How you find it:** `jcmd <pid> Thread.print`. The JVM finds it for you and prints
`Found one Java-level deadlock:` with the threads and monitors named. **This is the only bug
in the taxonomy the JVM diagnoses on your behalf** — and only for intrinsic monitors and
`java.util.concurrent.locks.Lock` implementations that report an owner.

### 3. Livelock

Threads are running — genuinely executing instructions — but no work completes. Each
thread's action causes another to retry, and they cycle forever.

**What it looks like:** 100% CPU with zero throughput. The opposite CPU signature to
deadlock, and the same business outcome.

**Where it comes from:** almost always a retry loop without randomised backoff. Two threads
take locks in opposite orders, both use `tryLock(50ms)`, both time out, both release, both
retry **at the same instant**, and collide again. Topic 94's `tryLock` fix converts a
deadlock into this if you forget the jitter.

**How you find it:** three thread dumps ten seconds apart showing the same threads
`RUNNABLE` in the same code region, with `cpu=` climbing fast, and a business counter that
is not moving.

### 4. Starvation

A thread never gets a resource it needs, while others get it repeatedly. It is not stuck —
it is *skipped*.

**Where it comes from:** non-fair locks and semaphores letting arriving threads barge past
queued ones (Topics 94, 97); a priority queue whose high-priority stream never empties; a
pool whose long tasks monopolise every worker; unbounded `ReadWriteLock` readers blocking a
writer for ever.

**What it looks like:** the hardest symptom here, because **the system appears to work**.
Throughput normal, p50 normal, p99.9 catastrophic — or one specific customer's orders never
process. It reads as "an intermittent slow request" and is usually closed as unreproducible.

**How you find it:** per-thread or per-queue age metrics — the **oldest** waiting item, not
the average. Never an aggregate.

### 5. Thread leak

Threads are created and never terminate. The count rises monotonically until the OS refuses
to create another.

**Where it comes from:** an `ExecutorService` per request; `new Thread()` per task in a
loop; a pool never shut down on context reload; a non-daemon thread waiting on a queue
nobody feeds.

**What it looks like:** slow degradation over hours or days — more context switching, more
scheduler pressure, rising memory outside the heap (Topic 80) — then
`java.lang.OutOfMemoryError: unable to create native thread` and process death.

**How you find it:** a thread-count gauge over time. The easiest bug here to detect in
advance, and one of the easiest to miss without that one metric.

### The symptom → diagnosis → fix decision table

**This is the most important table in Phase 9.** Learn it. It is what "the service is hung"
gets replaced with.

| Symptom you observe | Likely diagnosis | The diagnostic that CONFIRMS it | The fix |
|---|---|---|---|
| **Hung threads in a cycle; CPU idle; latency flat; logs silent** | **Deadlock** | `jcmd <pid> Thread.print` prints **`Found one Java-level deadlock:`**; threads are **`BLOCKED`** (monitors) or `WAITING (parking)` (`Lock`s), each `waiting to lock` a monitor `held by` the next | **Break one Coffman condition.** Global lock ordering (removes circular wait) is primary; `tryLock(timeout)` + jitter (removes no-preemption) is the safety net; take fewer locks is best |
| **100% CPU, no progress; throughput flat at zero** | **Livelock** | Three dumps 10 s apart: same threads **`RUNNABLE`** in the same region, `cpu=` **climbing**; a business counter flat; JFR shows execution samples in the retry loop | **Randomised exponential backoff** on retries, plus a **retry budget** that gives up and sheds. Never a bare retry loop |
| **Everything looks fine; p50 normal; one consumer/tenant/queue never progresses; p99.9 terrible** | **Starvation** | Age of the **oldest** waiting item, not the average; per-thread CPU (`top -H`) showing one thread at ~0 while peers work; a fair-vs-non-fair lock or an unbounded priority stream | **Fairness** (`new ReentrantLock(true)`, `new Semaphore(n, true)`), **ageing** (promote by wait time), or **partition** the resource so classes cannot starve each other |
| **Thread count only grows; degradation over hours; eventually `OutOfMemoryError: unable to create native thread`** | **Thread leak** | `jvm.threads.live` gauge monotonic; `jcmd <pid> Thread.print \| grep -c '^"'` rising across dumps; many identical stacks with distinct names; JFR `jdk.ThreadStart` far exceeding `jdk.ThreadEnd` | **One shared, bounded, named pool**, created once, shut down in `@PreDestroy`. Never `new ExecutorService` per request. Ban the factory methods with an ArchUnit rule |

**And the fifth row that is not a hang at all:**

| **Wrong results; no hang; no exception; system healthy** | **Data race** | **No runtime diagnostic exists.** A happens-before argument (Topic 86), jcstress (Topic 99), or a customer complaint | Establish the missing edge: a lock, a `volatile`, an atomic, or an immutable design |

---

## Why does it matter?

**1. Because the wrong diagnosis costs you the incident.**

All four hangs present identically to the customer and to the load balancer: requests stop
completing. If your first move is "restart it", you have destroyed the evidence, and the
same incident recurs on the same schedule until someone gets lucky. Teams that cannot tell
these apart do not fix them; they add pods, add alerts, and normalise a weekly restart.

**2. Because three of the four log nothing.**

A deadlock throws no exception. A livelock throws no exception. Starvation throws no
exception. Your error rate stays at zero while the service is completely down, because
requests that never complete never produce an error. **The observability you rely on is
structurally blind to this entire class of failure**, and the only instruments that see it
are the ones in this document.

**3. Because the four Coffman conditions generalise beyond locks, and that is the senior
insight.**

Deadlock is not a `synchronized` phenomenon. It is a property of any system with exclusive
resources, hold-and-wait, no preemption, and a cycle. Topic 109's HikariCP pool deadlock is
**the same four conditions with database connections as the resource** — ten threads each
holding one connection and each waiting for a second from a pool of ten. No lock is
involved, `findDeadlockedThreads()` returns `null`, and the fix is still "break one
condition": raise the pool above the maximum simultaneous holdings, or stop holding one
connection while acquiring another.

Once you see connections, permits, pool threads and file handles as resources, you diagnose
a class of outage that most engineers experience as bad luck.

---

## Machine-level reality

### The four Coffman conditions, and exactly what each fix removes

| Condition | What it means | How you remove it | What that costs |
|---|---|---|---|
| **Mutual exclusion** | the resource cannot be shared | make the data **immutable** (Topic 88), or give each thread its own copy | a redesign; not always possible |
| **Hold and wait** | a holder waits for more | acquire **all** resources atomically, or none; release everything before waiting | needs a global acquire point; can reduce concurrency badly |
| **No preemption** | the resource cannot be taken back | **`tryLock(timeout)`** — the thread gives up voluntarily and releases what it holds | converts deadlock into **livelock** unless you add jitter |
| **Circular wait** | a cycle exists in the wait-for graph | **impose a global lock order** — every thread takes locks in the same total order | you must define and enforce the order; review cannot be the enforcement |

**Removing exactly one is sufficient.** That is the entire theory, and it is why the
interview answer "how do you prevent deadlock" is not a list of tips: it is "name the four
conditions, then say which one you are removing and how".

**Lock ordering is the primary fix** because it costs the least at runtime: no timeouts, no
retries, no reduction in concurrency, no redesign. Its cost is entirely at design time —
someone must define the order and make it hard to violate.

### How the JVM finds a deadlock, and what it cannot find

The JVM builds a **wait-for graph**: for each blocked thread, which resource it waits on and
which thread owns that resource. A cycle in that graph is a deadlock. `jcmd Thread.print`
runs this analysis on every dump, and `ThreadMXBean.findDeadlockedThreads()` exposes it
programmatically.

**It can see** intrinsic monitors (`synchronized`), because the JVM knows the owner from the
object header, and `Lock` implementations that report an owner via
`AbstractOwnableSynchronizer` — `ReentrantLock`, `ReentrantReadWriteLock`.
(`findDeadlockedThreads()` covers both; `findMonitorDeadlockedThreads()` covers monitors only.)

**It cannot see — and this list separates you from someone who trusts the tool:**

| Invisible to the detector | Why |
|---|---|
| **`Semaphore`** cycles | no owner is recorded; there is no "who holds this permit" |
| **`CountDownLatch`** that never opens | no owner, no cycle — just a permanent wait |
| **`StampedLock`** self-deadlock | records no owner at all (Topic 94) |
| **Connection-pool exhaustion** (Topic 109) | a pool is not a monitor; `getConnection` is a timed wait on a queue |
| **`BlockingQueue`** full/empty deadlock | a `Condition` wait, with no owner |
| **Livelock, starvation, thread leaks** | not cycles at all; nothing to detect |

**`findDeadlockedThreads()` returning `null` is not evidence that nothing is stuck.** Say
that out loud. It is the single most common false conclusion in this area.

**And the JVM never breaks a deadlock.** It reports; it does not act. Postgres detects
deadlocks and kills a victim, so your application sees a retryable error; the JVM's threads
park and stay parked until the process dies.

### The exact difference between BLOCKED, WAITING and TIMED_WAITING

This is the load-bearing table of Phase 9 and the reason you can diagnose in minutes.

| State | What put it there | What it implies **here** |
|---|---|---|
| **`RUNNABLE`** | executing — **or** inside a native call the JVM cannot see, including **socket reads** | **Not proof of CPU use.** A thread blocked on `SocketInputStream.read` is `RUNNABLE` at 0% CPU. Cross-check with `top -H`. `RUNNABLE` + high CPU + no progress = **livelock**. |
| **`BLOCKED`** | waiting to enter or re-enter a `synchronized` block — **intrinsic monitors only** | Monitor contention. Several `BLOCKED` on one monitor = your hot lock. `BLOCKED` in a **cycle** = deadlock, and the JVM will say so. |
| **`WAITING`** | `Object.wait()`, `Thread.join()`, and **`LockSupport.park()` — every `j.u.c` construct**: `ReentrantLock`, `Semaphore`, `CountDownLatch`, `BlockingQueue` | **Can be permanent.** A `ReentrantLock` deadlock lands here, **not** in `BLOCKED`. So does a healthy idle worker. `WAITING` alone means nothing — you must read the stack and correlate with a metric. |
| **`TIMED_WAITING`** | `sleep`, `wait(n)`, `join(n)`, `parkNanos`, `tryLock(t,u)`, `poll(t,u)`, `offer(t,u)` | **This thread will wake up.** It cannot be part of a permanent deadlock. That is exactly why `tryLock(timeout)` works as a fix — and why it can produce livelock instead. |
| **`NEW` / `TERMINATED`** | not started / finished | Many retained `TERMINATED` threads is Topic 79, not this document. |

**Three rules follow, and they are what you will actually use.** `synchronized` produces
`BLOCKED`; `j.u.c` produces `WAITING (parking)` — so a dump with no `BLOCKED` threads may
simply mean the codebase uses `ReentrantLock`. `RUNNABLE` does not mean running — corroborate
with per-thread CPU (`top -H -p <pid>`, convert the tid to hex, match `nid=0x...`). And
`TIMED_WAITING` is a promise to return, whereas deadlock is a property of threads that will
not.

### `OutOfMemoryError: unable to create native thread` — what is actually exhausted

Not the heap. The JVM asked the OS for a thread and was refused. Four causes, each checked
with a command rather than guessed: the per-user process limit (`ulimit -u`), the container's
`cat /sys/fs/cgroup/pids.max`, the system-wide `cat /proc/sys/kernel/threads-max`, and
address space — each platform thread reserves `-Xss` (default ~1 MB), so 4,000 threads is
**4 GB of reservation** that is not heap.

**The counter-intuitive consequence:** raising `-Xmx` makes this *worse*, because heap and
thread stacks compete for the same address space and the same cgroup limit. Someone who sees
`OutOfMemoryError` and raises the heap has made the failure arrive sooner. **Recognising the
message text is the whole diagnosis.**

**Virtual threads (Topic 101) change the arithmetic, not the lesson.** A virtual thread's
stack is a heap continuation, so you will hit `Java heap space` instead — the same bug with a
different message, and the same fix: no unbounded concurrency per request.

---

## Concurrency trace

**Before any correct code.** Four traces, because there are four failures. Read them as a
set: the point is that they produce the same customer symptom from completely different
mechanisms.

### Trace 1 — Deadlock

`orderflow` guards each product's stock and each user's wallet with a `ReentrantLock` from a
registry. `OrderPlacementService.place()` locks **inventory then wallet** — you check stock
before you take money. `RefundService.refund()` locks **wallet then inventory** — you give
the money back, then restock. Both are individually reasonable. Neither author knew about
the other.

| Step | Thread A — `http-nio-8080-exec-17` (order placement, order 8812, SKU-1001, user 55) | Thread B — `refund-worker-3` (refund of order 4471, SKU-1001, user 55) | Locks held / outcome |
|---|---|---|---|
| 1 | `inventoryLock("SKU-1001").lock()` → CAS succeeds | — | A: {inventory} |
| 2 | reads `available = 12` | `walletLock(55).lock()` → CAS succeeds | A: {inventory} · B: {wallet} |
| 3 | computes `11`, not yet written | credits wallet by £40.00 | A: {inventory} · B: {wallet} |
| 4 | `walletLock(55).lock()` → held by B → `park` | — | **A is `WAITING (parking)`** |
| 5 | parked | `inventoryLock("SKU-1001").lock()` → held by A → `park` | **B is `WAITING (parking)`** |
| 6 | parked | parked | **Circular wait closed. Deadlock.** All four Coffman conditions hold at this instant. |
| 7 | parked | parked | No exception. No log line. No timeout. `cpu=` for both stops increasing. |
| 8 | request thread 18 for SKU-1001 parks behind A | refund worker 4 parks behind B | two growing piles of identical stacks |
| 9 | …threads 19–200 arrive | …workers 5–8 arrive | **Tomcat's 200-thread pool is fully consumed** |
| 10 | — | — | `GET /products`, which touches **neither** lock, returns 503 |

**Outcome, in business terms.** Order 8812 never completes and never fails; the card
authorisation succeeded before the lock was taken, so money is reserved against an order
that does not exist. Refund 4471 is half-applied — wallet credited, stock never returned, so
that unit is permanently unsellable. Within ninety seconds every endpoint returns 503,
including the catalogue. **The latency dashboard does not spike; it goes flat**, because
requests that never complete never record a latency. The error-rate graph shows 503s from
the load balancer, not the application, so the application logs are silent. The Topic 65 p99
is not violated — it is simply no longer being measured.

### Trace 2 — Livelock (a separate failure, from the deadlock fix applied carelessly)

Same two services. Someone fixed Trace 1 with `tryLock(50ms)` — correct, per Topic 94 — and
then wrote the retry loop with a fixed delay and no jitter.

```java
while (true) {
    if (invLock.tryLock(50, MILLISECONDS)) {
        try {
            if (walletLock.tryLock(50, MILLISECONDS)) { try { place(); return; } finally { walletLock.unlock(); } }
        } finally { invLock.unlock(); }
    }
    Thread.sleep(10);        // fixed delay. No randomness. This is the bug.
}
```

| Step | Thread A — order placement | Thread B — refund worker | CPU / outcome |
|---|---|---|---|
| 1 | `invLock.tryLock(50ms)` → **true** | `walletLock.tryLock(50ms)` → **true** | A: {inv} · B: {wallet} · both `RUNNABLE` |
| 2 | `walletLock.tryLock(50ms)` → held by B → parks with a deadline | `invLock.tryLock(50ms)` → held by A → parks with a deadline | both `TIMED_WAITING` — **not** deadlock; both will wake |
| 3 | 50 ms elapses → returns **false** | 50 ms elapses → returns **false** | both wake **at the same instant**, because both started at the same instant |
| 4 | `finally` → `invLock.unlock()` | `finally` → `walletLock.unlock()` | both locks free · **the cycle broke, exactly as designed** |
| 5 | `sleep(10)` | `sleep(10)` | both sleep the **same fixed** 10 ms |
| 6 | wakes, `tryLock(inv)` → true | wakes, `tryLock(wallet)` → true | **the same collision, perfectly re-synchronised** |
| 7 | parks 50 ms, fails, releases, sleeps 10 ms | identical | one full cycle = 60 ms, zero progress |
| 8 | …repeats, ~16 cycles per second, per thread pair… | | `cpu=` for both threads **climbing steadily** |
| 9 | request threads 18–200 join, each pairing off the same way | refund workers 4–8 join | **CPU utilisation approaches 100%** |
| 10 | throughput of order placement: **zero** | throughput of refunds: **zero** | **Livelock.** Every thread is `RUNNABLE`. Nothing completes. |

**Outcome, in business terms.** Identical to Trace 1 from the customer's seat — no orders,
no refunds, 503s across the service — and **the opposite from the operator's seat**. CPU is
pinned at 100%, so every dashboard says "we are compute-bound, scale up". Autoscaling adds
pods. The new pods have the same code, immediately livelock, and add their own CPU load.
`jcmd Thread.print` shows **no** deadlock section, so the on-call engineer — who has learned
to grep for `Found one Java-level deadlock` — concludes that locking is not the problem and
starts looking at the database.

The distinguishing evidence is precisely two things: **`cpu=` climbing across dumps** (in a
deadlock it is frozen), and **`TIMED_WAITING`/`RUNNABLE` rather than `WAITING`**.

**And the fix is one line:** randomise the backoff.

```java
Thread.sleep(ThreadLocalRandom.current().nextLong(5, 50));   // jitter breaks the lockstep
```

Plus a **retry budget** — after N attempts, give up and return an error — because an
unbounded retry loop is a livelock waiting for the right conditions. This is Topic 111's
retry-with-jitter argument, and it is the same mechanism that stops a retry storm.

### Trace 3 — Starvation (a separate failure again, with no hang at all)

`orderflow` processes payment callbacks from a `PriorityBlockingQueue`, ordered so that
premium-tier merchants are handled before standard-tier. Four consumer threads. This works
for months.

Then a large premium merchant is onboarded and its callback volume alone exceeds the
consumers' drain rate.

| Step | Threads A1–A4 — `callback-worker-1..4` | Standard-tier callback for merchant 7781 | Queue state / outcome |
|---|---|---|---|
| 1 | `take()` → highest priority available | enqueued at priority 5 (standard) | queue: premium items present · 7781 is behind them |
| 2 | drain premium items at 400/s | still queued | 7781 not yet reached |
| 3 | premium arrivals now 500/s, drain 400/s | still queued | **premium backlog now grows faster than it drains** |
| 4 | every `take()` returns a premium item | still queued | 7781 will **never** be reached while that holds |
| 5 | …one hour… | still queued | the consumers are **100% busy and 100% healthy** |
| 6 | throughput: 400/s, normal | still queued | **every aggregate metric is green** |
| 7 | p50 callback latency: 12 ms | age of 7781: **3,600,000 ms** | the mean is fine because 7781 is one item among millions |
| 8 | …six hours… | still queued | merchant 7781's orders sit unconfirmed |
| 9 | ops asked to investigate "a slow merchant" | still queued | the queue-depth gauge looks *normal* — it is bounded and stable |
| 10 | — | still queued | **Starvation.** No thread is stuck. No cycle exists. Nothing is hung. |

**Outcome, in business terms.** `orderflow` reports itself completely healthy: throughput at
baseline, p50 and p95 inside the Topic 65 numbers, queue depth stable, CPU normal, zero
errors. Meanwhile **an entire class of customer is receiving no service at all**. Merchant
7781 raises a support ticket; it is closed as "unable to reproduce", because any test
callback sent by support is *also* standard-tier and *also* never processed — which nobody
notices, because the test is asynchronous and the engineer moves on.

The failure surfaces commercially, weeks later, when a standard-tier merchant churns.

**Why every instrument you own missed it:** starvation is invisible to **averages**.
p50 is fine, p95 is fine, even p99 may be fine when the starved class is 0.1% of volume. The
one metric that shows it is the **age of the oldest item in the queue**, which almost nobody
gauges — and a per-class breakdown, because the aggregate is exactly what hides it.

**The fixes, and the trade each makes:**

| Fix | Mechanism | Cost |
|---|---|---|
| **Ageing** | raise an item's effective priority with its wait time | the priority order is no longer strict; premium latency rises slightly |
| **Partition** | separate queues and separate consumers per tier | premium can no longer borrow standard's capacity when idle |
| **Fair lock/semaphore** | `new Semaphore(n, true)` where the contention is on a permit | a handoff per acquisition; lower throughput |
| **Reserve capacity** | one consumer that only ever takes standard-tier | that consumer idles when standard is empty |

**Partitioning is usually right**, because it makes the guarantee structural rather than
probabilistic: a starved class cannot exist if the classes do not share a queue.

### Trace 4 — Thread leak (the assigned drill, traced)

`orderflow`'s order-detail endpoint fans out three lookups in parallel. The author needed an
executor, and created one where it was needed:

```java
@GetMapping("/orders/{id}")
public OrderDetail get(@PathVariable long id) {
    ExecutorService pool = Executors.newFixedThreadPool(4);      // per request. This is the bug.
    var product = pool.submit(() -> productService.find(id));
    var inventory = pool.submit(() -> inventoryService.find(id));
    var payment = pool.submit(() -> paymentService.find(id));
    return new OrderDetail(product.get(), inventory.get(), payment.get());
    // no shutdown(). And it would still leak if it were in a finally on the exception path only.
}
```

At the Topic 65 baseline: **400 requests per second**.

| Step | Request threads (400/s) | Leaked pool threads | Thread count / outcome |
|---|---|---|---|
| 1 | request arrives → `newFixedThreadPool(4)` | 4 threads created, named `pool-1-thread-1..4` | **+4 threads** |
| 2 | three tasks submitted; the fourth thread never runs anything | 4 threads alive | the response is **correct**; the endpoint works |
| 3 | request returns 200; `pool` goes out of scope | **4 threads still alive** — a running thread is a GC root | `ExecutorService` is unreachable and **uncollectable** |
| 4 | 400 requests in second 1 | +1,600 threads | ~1,600 |
| 5 | 10 seconds | +16,000 threads | ~16,000 · context-switch rate climbing |
| 6 | 30 seconds | +48,000 | scheduler pressure; every thread's timeslice shrinks; p99 rising |
| 7 | ~60 s | ~96,000 threads × ~1 MB stack reservation | **RSS far exceeds `-Xmx`** (Topic 80) — heap is fine |
| 8 | — | thread creation begins to fail | `ulimit -u` or the cgroup `pids.max` reached |
| 9 | a request thread calls `newFixedThreadPool` | OS refuses | **`java.lang.OutOfMemoryError: unable to create native thread`** |
| 10 | the error propagates as a 500 from a **random** endpoint | — | the failing endpoint is whichever ran next — usually **not** the leaking one |
| 11 | someone raises `-Xmx` "because it's an OutOfMemoryError" | — | **strictly worse** — heap and stacks compete for the same address space |
| 12 | pod OOM-killed or JVM dies | — | restart; the clock resets; recurs in an hour |

**Outcome, in business terms.** `orderflow` restarts roughly hourly under baseline load. Each
restart drops in-flight requests and empties every in-memory queue (Topic 93's lesson,
arriving from a different direction). The team adds pods, which halves the per-pod rate and
doubles the time to failure, so the incident becomes "every two hours" and is downgraded
from "outage" to "known issue". A restart-loop alert is muted. Someone raises the heap,
which makes it worse, and that change is not connected to the regression because the error
message says `OutOfMemoryError` and the heap graph looks fine.

**The one metric that would have caught it on day one:** `jvm.threads.live`, gauged. It rises
monotonically and never falls. Nothing else in the entire observability stack shows this
clearly — not heap, not GC, not latency, not error rate.

---

## Example 1 — minimal

The four bugs, each in the fewest lines that produce them, so you can run them.

```java
public final class FourBugs {

    // 1. DEADLOCK — two locks, two orders. Runs for ever.
    static final Object INVENTORY = new Object();
    static final Object WALLET    = new Object();

    static void deadlock() {
        new Thread(() -> { synchronized (INVENTORY) { sleep(50);
                           synchronized (WALLET) { } } }, "order-placement").start();
        new Thread(() -> { synchronized (WALLET)    { sleep(50);
                           synchronized (INVENTORY) { } } }, "refund-worker").start();
    }

    // 2. LIVELOCK — both back off politely, in lockstep, for ever. 100% CPU.
    static final AtomicBoolean invFree = new AtomicBoolean(true);
    static final AtomicBoolean walFree = new AtomicBoolean(true);

    static void livelock() {
        Runnable polite = () -> {
            while (true) {
                if (invFree.compareAndSet(true, false)) {
                    if (walFree.compareAndSet(true, false)) { /* work */ return; }
                    invFree.set(true);          // "you first" — and so does the other thread
                }
                sleep(10);                       // FIXED delay: the lockstep never breaks
            }
        };
        new Thread(polite, "polite-a").start();
        new Thread(polite, "polite-b").start();
    }

    // 3. STARVATION — a non-fair lock plus one greedy thread.
    static final ReentrantLock LOCK = new ReentrantLock(/* fair = */ false);

    static void starvation() {
        for (int i = 0; i < 4; i++)
            new Thread(() -> { while (true) { LOCK.lock(); try { spin(1_000_000); }
                                              finally { LOCK.unlock(); } } }, "greedy-" + i).start();
        new Thread(() -> { LOCK.lock();                    // may wait a very long time
                           try { System.out.println("polite finally ran"); }
                           finally { LOCK.unlock(); } }, "polite").start();
    }

    // 4. THREAD LEAK — an executor per iteration, never shut down.
    static void threadLeak() {
        while (true) {
            ExecutorService pool = Executors.newFixedThreadPool(4);
            pool.submit(() -> {});
            // no shutdown(): the 4 threads live for ever, and a running thread is a GC root
        }
    }
}
```

**What to notice, and it is the same point four times:**

- **Not one of these throws.** Not one logs. All four produce a program that appears to be
  running.
- **`deadlock()` and `livelock()` are indistinguishable to a user** and opposite in CPU.
- **`starvation()` is not stuck at all.** The greedy threads work perfectly. Non-fair
  `ReentrantLock` lets an arriving thread barge past queued waiters, so `polite` may wait
  arbitrarily long. Change `false` to `true` and it runs promptly — at a throughput cost.
- **`threadLeak()` fails with an `OutOfMemoryError` that has nothing to do with the heap.**

**Run these before reading on.** Twenty minutes with `jcmd` against these four processes
teaches more than any amount of reading — you will see the four dump shapes yourself.

---

## Example 2 — production scenario (on the project spine)

### The scenario, stated as it will actually arrive

03:12. `orderflow` p99 alert fires. By the time you are online:

- error rate from the load balancer: rising 503s
- error rate from the application: **zero**
- application logs: nothing unusual since 03:04
- CPU: **you have not looked yet, and this is the fork in the road**
- database: healthy, connections idle
- the payment provider: green
- Slack: "should we restart it?"

**Your first decision is not technical.** It is: **do not restart.** A restart clears the
symptom, destroys the evidence, and guarantees you are back here next week with the same
zero information. Instead, remove the pod from the load balancer (Topic 121's readiness
endpoint — liveness must **not** be what kills it) and leave it running.

### The five-minute procedure

```bash
# 0. Identify the JVM.
jcmd -l

# 1. THREE dumps, ten seconds apart. One dump gives state; three give CHANGE.
for i in 1 2 3; do jcmd <pid> Thread.print -l > /tmp/d$i.txt; sleep 10; done

# 2. Did the JVM already diagnose it for you?
grep -A 40 "Found one Java-level deadlock" /tmp/d1.txt

# 3. State histogram — the shape of the problem in one line.
grep "java.lang.Thread.State" /tmp/d1.txt | sort | uniq -c | sort -rn

# 4. Thread count, across all three dumps. Rising = leak.
for f in /tmp/d1.txt /tmp/d2.txt /tmp/d3.txt; do echo -n "$f: "; grep -c '^"' $f; done

# 5. Is CPU being burned? (livelock's signature)
top -H -p <pid> -b -n 1 | head -25

# 6. Are the stacks CHANGING?
diff <(grep -A1 '^"' /tmp/d1.txt) <(grep -A1 '^"' /tmp/d3.txt) | head -40

# 7. What is holding what?
grep -E "waiting to lock|locked <0x" /tmp/d1.txt | sort | uniq -c | sort -rn
```

**Then apply the decision table.** Each branch has a distinct signature:

| What steps 2–6 show | Diagnosis | Next command |
|---|---|---|
| Step 2 prints a deadlock section | **deadlock** | read it — it names the threads and monitors |
| Step 3: many `BLOCKED` on one monitor, step 2 silent | monitor contention, not deadlock | Topic 85 — find the hot lock |
| Step 3: many `WAITING (parking)`, step 2 silent, step 5 CPU low | a `j.u.c` deadlock, a permit leak (Topic 97), or a pool exhaustion (Topic 109) | read the `parking to wait for` class |
| Step 5: threads at high CPU, step 6 shows the same stacks | **livelock** | JFR execution samples to find the retry loop |
| Step 4: counts rising across dumps | **thread leak** | group thread names: `grep -o '^"[^"]*"' d1.txt \| sed 's/-[0-9]*"/"/' \| sort \| uniq -c` |
| Everything normal, one queue's oldest item is hours old | **starvation** | per-class age metrics |

### The dump section you must be able to read

When step 2 hits, this is the shape.

> *illustration of the format, not captured output*

```
Found one Java-level deadlock:
=============================
"http-nio-8080-exec-17":
  waiting to lock monitor 0x... (object 0x..., a com.orderflow.wallet.Wallet),
  which is held by "refund-worker-3"
"refund-worker-3":
  waiting to lock monitor 0x... (object 0x..., a com.orderflow.inventory.Inventory),
  which is held by "http-nio-8080-exec-17"

Java stack information for the threads listed above:
===================================================
"http-nio-8080-exec-17":
        at com.orderflow.orders.OrderPlacementService.place(OrderPlacementService.java:<n>)
        - waiting to lock <0x...> (a com.orderflow.wallet.Wallet)
        - locked <0x...> (a com.orderflow.inventory.Inventory)
        at com.orderflow.orders.OrderController.create(OrderController.java:<n>)
        ...
"refund-worker-3":
        at com.orderflow.refunds.RefundService.refund(RefundService.java:<n>)
        - waiting to lock <0x...> (a com.orderflow.inventory.Inventory)
        - locked <0x...> (a com.orderflow.wallet.Wallet)
        ...

Found 1 deadlock.
```

**How to read it — four questions, in this order:**

1. **Which threads?** The quoted names. This is why naming your threads is the cheapest
   diagnostic investment you will ever make: `refund-worker-3` tells you the subsystem;
   `Thread-47` tells you nothing.
2. **Which objects?** The types in parentheses — `Wallet`, `Inventory`. **These are your
   domain classes**, which means the answer to "where in the code" is already on the screen.
3. **Which order did each take them in?** Read the `locked` and `waiting to lock` lines
   *within each thread's stack*. Thread 17 **locked** `Inventory` and **waits for** `Wallet`.
   Worker 3 **locked** `Wallet` and **waits for** `Inventory`. **That is the circular wait,
   printed.**
4. **What is the global order that would fix it?** Any total order applied consistently.
   Pick one — e.g. always by class name, or always inventory-before-wallet — and enforce it
   at a single acquire point.

### The fix, expressed as breaking one condition

```java
/**
 * Acquires both locks in a globally-consistent order, breaking CIRCULAR WAIT (Coffman #4).
 * Every path in the codebase that needs both locks MUST come through here.
 */
public <T> T withInventoryAndWallet(String sku, long userId, Supplier<T> work) {
    Lock inv = inventoryLocks.computeIfAbsent(sku, k -> new ReentrantLock());
    Lock wal = walletLocks.computeIfAbsent(userId, k -> new ReentrantLock());

    // The ORDER is the fix. Inventory before wallet, always, everywhere.
    inv.lock();
    try {
        wal.lock();
        try {
            return work.get();
        } finally { wal.unlock(); }
    } finally { inv.unlock(); }
}
```

Three details that make this production code rather than an illustration:

- **`computeIfAbsent`, not `get`-then-`put`** (Topic 92). A racing `get`/`put` pair hands two
  threads two *different* `ReentrantLock` objects for the same SKU, which silently removes
  all mutual exclusion — a data race introduced by the deadlock fix.
- **One acquire point.** Enforce it with ArchUnit: no class outside this one may call
  `lock()` on either registry. **Review is not enforcement**; the fix must be structural, or
  the next author reintroduces the cycle.
- **`lock()` outside the `try`, nothing between them** (Topic 94). If `lock()` throws you
  must not unlock a lock you never took.

### And the safety net, with the livelock avoided

```java
if (!inv.tryLock(50, MILLISECONDS)) throw new LockAcquisitionTimeout("inventory " + sku);
try {
    if (!wal.tryLock(50, MILLISECONDS)) throw new LockAcquisitionTimeout("wallet " + userId);
    try { return work.get(); } finally { wal.unlock(); }
} finally { inv.unlock(); }
```

**This breaks no-preemption (Coffman #3)** and converts a permanent hang into an error you
can count. **It is the secondary fix, not the primary one**, for exactly the reason Trace 2
demonstrates: under sustained contention, retrying this without jitter produces livelock. If
you retry at all:

```java
Thread.sleep(ThreadLocalRandom.current().nextLong(5, 50));   // jitter, always
if (++attempts > 3) { shed.increment(); throw new LockAcquisitionTimeout(...); }  // a budget
```

**Ordering removes the cycle. `tryLock` bounds the damage when someone reintroduces it.**
Ship both, in that priority.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — treating "the service is hung" as one diagnosis

**Wrong approach.** The pager fires, the service is unresponsive, and the response is a
restart — or a single hypothesis ("must be a deadlock") pursued to the exclusion of the other
three.

**Exact symptom.** The incident is "resolved" by the restart and recurs on a schedule. Over
weeks, the postmortems say "transient", "resource contention" and "under investigation".
Someone builds a cron job to restart the service nightly. The actual bug is never found,
because **the evidence is destroyed within sixty seconds of every occurrence**.

The subtler version: an engineer greps for `Found one Java-level deadlock`, finds nothing,
and concludes locking is fine. That grep is only sensitive to **one** of the four failures —
and only to the subset the JVM can see. A `Semaphore` cycle, a livelock, a starved consumer
and a thread leak all produce empty output from it.

**Root cause.** Four mechanisms produce one customer symptom. Without a decision procedure,
you are pattern-matching on the symptom, and the symptom does not discriminate.

**Fix.** The procedure from Example 2, written down and rehearsed *before* the incident:
**do not restart** (readiness out, process alive); three dumps ten seconds apart; grep for
the deadlock section — one branch; the state histogram — `BLOCKED` versus `WAITING` versus
`RUNNABLE`; per-thread CPU — livelock's discriminator; thread count across the three dumps —
the leak's discriminator; and a diff of dump 1 against dump 3 — stuck versus slow.

**Put this in the runbook and rehearse it in a game day.** At 03:12 you will not invent it.

### Trap 2 — `new ExecutorService` per request

**Wrong approach.** Trace 4. An executor created inside a request handler, method or loop.

Three variants, all common, all the same bug: `Executors.newFixedThreadPool(4)` inside a
request handler; `CompletableFuture.supplyAsync(task, Executors.newCachedThreadPool())` per
call; and `new Thread(task).start()` per item in a loop.

**Exact symptom.** In this order:

1. Everything works. Responses are correct. Tests pass.
2. `jvm.threads.live` rises monotonically and never falls.
3. Context-switch rate climbs; p99 degrades with no code change and no traffic change.
4. **RSS far exceeds `-Xmx`** — each platform thread reserves ~1 MB of stack outside the
   heap (Topic 80).
5. `java.lang.OutOfMemoryError: unable to create native thread`, thrown from **a random
   endpoint** — whichever code happened to ask for a thread next, usually not the leaking one.
6. Someone raises `-Xmx` because the message says `OutOfMemoryError`. **It gets worse.**
7. Restart. Recurs on a fixed schedule proportional to request rate.

**Root cause, in two parts:**

- **A running thread is a GC root.** The `ExecutorService` becomes unreachable, but its
  threads are alive and therefore uncollectable. `Executors.newFixedThreadPool` creates
  **non-daemon** threads, which additionally keep the JVM from exiting.
- **`shutdown()` is not optional and not automatic.** There is no finaliser, no
  `AutoCloseable` behaviour in older code paths, and no warning. Even
  `newFixedThreadPool(4)` with only three tasks leaves the fourth thread alive for ever.

**Fix.** One shared, bounded, named pool, created once and shut down deliberately:

```java
@Bean(destroyMethod = "shutdown")
ExecutorService orderDetailPool(MeterRegistry meters) {
    ThreadPoolExecutor pool = new ThreadPoolExecutor(
            8, 8, 0L, TimeUnit.MILLISECONDS,
            new ArrayBlockingQueue<>(1_000),                 // bounded (Topic 93)
            Thread.ofPlatform().name("order-detail-", 0).factory(),
            new ThreadPoolExecutor.CallerRunsPolicy());
    ExecutorServiceMetrics.monitor(meters, pool, "order-detail");   // Topic 118
    return pool;
}
```

and, so it cannot come back:

```java
// ArchUnit — a RULE, not a review comment.
noClasses().should().callMethod(Executors.class, "newFixedThreadPool", int.class)
           .because("pools are @Beans with a lifecycle, never created per request");
```

**And the gauge that catches every future variant:** alert when `jvm.threads.live` rises
monotonically over 30 minutes. That single alert catches this bug, its `new Thread()`
cousin, and the "pool never shut down on context reload" variant.

### Trap 3 — fixing deadlock with `tryLock` and no jitter

**Wrong approach.** Trace 2. `tryLock(timeout)` added, retry loop written with a fixed delay.

**Exact symptom.** The deadlock is gone — genuinely, the fix worked — and is replaced by
100% CPU with zero throughput. Every diagnostic points the wrong way: high CPU reads as
compute-bound, so autoscaling adds pods, which livelock immediately and add load. `jcmd`
shows **no** deadlock section, so the engineer who learned to grep for it concludes locking
is fine.

**Root cause.** `tryLock` breaks no-preemption, which is a valid way to break deadlock. But
two threads that fail, release and retry **on the same schedule** re-synchronise perfectly
and collide again. A fixed backoff does not desynchronise them — **it guarantees they stay
synchronised**.

**Fix.** Three things, all of them:

```java
Thread.sleep(ThreadLocalRandom.current().nextLong(5, 50));      // 1. jitter
if (++attempts > 3) { shed.increment(); throw new ...; }        // 2. a retry budget
// 3. and fix the ORDERING, so the retry path is a safety net rather than the mechanism
```

**Say the general rule:** *every retry needs randomised backoff and a budget.* This is
Topic 111's retry-storm argument at thread scale, and it is the same failure at both scales.

### Trap 4 — trusting `findDeadlockedThreads()` to mean "not stuck"

**Wrong approach.** A health check or an incident procedure whose deadlock test is:

```java
long[] ids = ManagementFactory.getThreadMXBean().findDeadlockedThreads();
if (ids == null) { /* "no deadlock, look elsewhere" */ }
```

**Exact symptom.** The service is completely hung. Threads are parked. The check returns
`null`, the runbook says "not a deadlock", and the investigation moves to the database — which
is healthy — and then to the network, which is also healthy. Hours pass.

**Root cause.** The detector builds a wait-for graph from **resources that report an owner**:
intrinsic monitors and `AbstractOwnableSynchronizer`-based `Lock`s. It cannot see a cycle
through anything ownerless:

- **`Semaphore`** — no owner (Topic 97).
- **`CountDownLatch`** that never opens — no owner, no cycle, just a permanent wait.
- **A connection pool** — `getConnection` is a timed wait on a queue, not a lock. **Topic
  109's pool deadlock returns `null` here**, and it is a genuine deadlock by the Coffman
  definition.
- **`StampedLock`** — records no owner at all.
- **`BlockingQueue`** — a `Condition` wait.

**Fix.** Treat the detector as a **positive-only** signal. If it fires, you have a deadlock.
If it is silent, you have learned **nothing**. Continue with the state histogram, and read
the `parking to wait for` class name:

```bash
grep -oE 'a java\.util\.concurrent\.[A-Za-z$]*' /tmp/d1.txt | sort | uniq -c | sort -rn
```

| Class in the parking line | Consult |
|---|---|
| `ReentrantLock$NonfairSync` | Topic 94 — and the detector *should* have caught it; check for `StampedLock` too |
| `Semaphore$NonfairSync` | Topic 97 — at capacity, or a permit leak. Check the permits gauge. |
| `CountDownLatch$Sync` | Topic 97 — a task died before its `countDown()` |
| `...ConditionObject` under a `BlockingQueue` frame | Topic 93 — check the depth gauge before concluding anything |
| `HikariPool` / `getConnection` frames | **Topic 109** — the pool deadlock. The four Coffman conditions with connections as the resource. |

**Ship the programmatic detector anyway** — it is cheap and catches the case it can catch —
but log it as `deadlockDetected=true` rather than treating silence as `false`.

### Trap 5 — diagnosing starvation from aggregate metrics

**Wrong approach.** The dashboard shows throughput, p50, p95, p99 and queue depth, all
aggregate, all healthy. Conclusion: the system is fine.

**Exact symptom.** Trace 3. A support ticket from one customer whose work never completes,
closed as unreproducible because every aggregate is green and any test the engineer runs is
also starved — asynchronously, so the engineer never sees it fail.

**Root cause.** Starvation is a **distributional** failure, and aggregates are exactly the
transformation that destroys distributional information. If the starved class is 0.1% of
volume, it cannot move p99. Queue depth is stable, because items arrive and depart at equal
rates — just not the *same* items.

**Fix.** Measure the things that survive aggregation:

```java
// The age of the OLDEST waiting item. Not the average. This is the starvation detector.
Gauge.builder("orderflow.callbacks.oldest.age.seconds", queue,
              q -> q.peekOldest().map(i -> secondsSince(i.enqueuedAt())).orElse(0.0))
     .register(meters);

// And per-class, because the aggregate is what hides it.
Gauge.builder("orderflow.callbacks.oldest.age.seconds", queue, q -> oldestAge(q, "standard"))
     .tag("tier", "standard").register(meters);
```

**Alert on the maximum age crossing a threshold, per class.** Then remove the possibility
structurally: partition the queue per tier so classes cannot starve each other, or add
ageing so waiting time raises effective priority. **Partitioning is usually right**, because
it makes the guarantee structural rather than statistical.

**The general principle, worth more than the fix:** *any metric that answers "is anyone being
ignored" must be a maximum or a per-class breakdown. A mean cannot answer it, by
construction.*

---

## Hands-on proof

No JVM ran here. These are the commands; the outputs are yours.

### Setup

```bash
java -version && mkdir -p /tmp/tax && cd /tmp/tax
ulimit -u                 # your per-user process/thread limit — note this number
sysctl -n hw.logicalcpu   # macOS core count
```

### Proof 1 — produce a real deadlock and read the dump

Save `FourBugs.java` from Example 1, calling only `deadlock()` from `main`, then:

```bash
java FourBugs.java &
jcmd <pid> Thread.print > /tmp/deadlock.txt
grep -A 40 "Found one Java-level deadlock" /tmp/deadlock.txt
```

**WHAT TO LOOK FOR:** the header, the two thread names, the two monitor lines with
`which is held by`, and — in the stack section — a `locked <0x...>` and a
`waiting to lock <0x...>` per thread, in **opposite** orders.

| What you see | What it means |
|---|---|
| `Found one Java-level deadlock:` and `Found 1 deadlock.` | The JVM diagnosed it for you. This is the only bug in the taxonomy it will. |
| Both threads `BLOCKED (on object monitor)` | `synchronized`. Convert the example to `ReentrantLock` and re-run: they become `WAITING (parking)` and the detector **still finds it**, because `ReentrantLock` reports an owner. |
| `cpu=` values identical across two dumps ten seconds apart | Deadlock burns no CPU. **This is the discriminator against livelock.** |
| Nothing found | Your threads have not reached step 4 of Trace 1 yet — raise the `sleep`, or take another dump. |

Then confirm the programmatic view:

```bash
jcmd <pid> Thread.print | grep -c "Found one Java-level deadlock"
```

### Proof 2 — livelock, and how it differs

Run `livelock()` and take two dumps ten seconds apart.

```bash
java FourBugs.java &        # with livelock() in main
top -H -p <pid> -b -n 1 | head -20     # Linux;  on macOS: top -pid <pid> -stats pid,cpu
for i in 1 2; do jcmd <pid> Thread.print > /tmp/ll$i.txt; sleep 10; done
grep -A 4 '"polite-a"' /tmp/ll1.txt /tmp/ll2.txt
grep -c "Found one Java-level deadlock" /tmp/ll1.txt
```

| What you see | What it means |
|---|---|
| `RUNNABLE` or `TIMED_WAITING (sleeping)`, **never** a stable `WAITING` | Not deadlock. These threads are running. |
| `Found one Java-level deadlock` count = **0** | **Critical.** The detector is silent and the service is still completely stuck. Trap 4, demonstrated. |
| `cpu=` **increasing substantially** between the two dumps | The livelock discriminator. Compare with Proof 1, where it is frozen. |
| The same stack frames in both dumps | Same region, no progress. Combined with rising CPU, that is livelock and nothing else. |

**Now fix it in one line** — replace the fixed `sleep(10)` with
`sleep(ThreadLocalRandom.current().nextLong(5, 50))` — and watch it complete. **That one
line is the difference between an outage and a working system**, and running it yourself is
what makes you remember.

### Proof 3 — the thread leak and its distinctive error

```bash
# Small heap and small stacks so this fails in seconds rather than minutes.
java -Xmx256m -Xss256k FourBugs.java &     # with threadLeak() in main

# watch the count rise:
while true; do jcmd <pid> Thread.print | grep -c '^"'; sleep 2; done

# and the JVM's own view:
jcmd <pid> VM.native_memory summary        # needs -XX:NativeMemoryTracking=summary
jcmd <pid> GC.heap_info                    # heap is FINE. That is the point.
```

| What you see | What it means |
|---|---|
| Thread count rising monotonically, never falling | The leak, made visible. This is the one metric that shows it. |
| `java.lang.OutOfMemoryError: unable to create native thread` | **Not a heap error.** The OS refused a thread. |
| `GC.heap_info` showing plenty of free heap at the moment of the OOM | The proof that raising `-Xmx` is the wrong response. |
| NMT `Thread` category dwarfing `Java Heap` | Where the memory actually went (Topic 80). |
| A *different* error — `Java heap space` | You are on virtual threads, or your `-Xss` is small enough that heap ran out first. Same bug, different message. |

**Now do the instructive variant:** add `pool.shutdown()` and re-run. The count stabilises.
Then remove it but make the threads **daemon** and observe that the JVM will now exit — but
still leaks while running. **Daemon-ness is about process exit; it is not a fix for a leak.**

### Proof 4 — settle what the detector can and cannot see

```java
// Undetectable.java — a genuine deadlock the JVM will not report.
import java.util.concurrent.*;
import java.lang.management.*;
public class Undetectable {
    static final Semaphore A = new Semaphore(1), B = new Semaphore(1);
    public static void main(String[] x) throws Exception {
        System.out.println("pid " + ProcessHandle.current().pid());
        new Thread(() -> { try { A.acquire(); Thread.sleep(100); B.acquire(); }
                           catch (Exception e) {} }, "sem-a").start();
        new Thread(() -> { try { B.acquire(); Thread.sleep(100); A.acquire(); }
                           catch (Exception e) {} }, "sem-b").start();
        Thread.sleep(2000);
        long[] d = ManagementFactory.getThreadMXBean().findDeadlockedThreads();
        System.out.println("findDeadlockedThreads() -> " + (d == null ? "null" : d.length));
        Thread.currentThread().join();
    }
}
```

```bash
java Undetectable.java &
jcmd <pid> Thread.print | grep -c "Found one Java-level deadlock"
jcmd <pid> Thread.print | grep -A 6 '"sem-a"'
```

| What you see | What it means |
|---|---|
| `findDeadlockedThreads() -> null` while both threads are permanently stuck | **Trap 4, proved on your own JVM.** Silence is not evidence of health. |
| No `Found one Java-level deadlock` section | The dump's automatic analysis is equally blind. |
| Both threads `WAITING (parking)` on `Semaphore$NonfairSync` | The evidence you *do* get. Reading the parked-on class is the skill. |
| A non-null result | Your JDK reports semaphore ownership — surprising; check the version and believe the JVM, not me. |

**This is the single most important proof in the document**, because it is the one that
stops you from ever writing "no deadlock detected, so it isn't a locking problem" in an
incident channel.

---

## Failure drill

**The assigned drill:** create a thread leak by constructing an `ExecutorService` per
request. Watch the thread count grow until
`java.lang.OutOfMemoryError: unable to create native thread`. Then fix it, and — the half
that makes you good at this — **prove that every other diagnostic in your toolkit was
silent** while it happened.

### Part A — instrument first, or the drill teaches nothing

```java
// The single most valuable gauge in this document.
Gauge.builder("jvm.threads.live", () -> ManagementFactory.getThreadMXBean().getThreadCount())
     .register(meters);
// (Spring Boot's JvmThreadMetrics binder registers jvm.threads.live for you — verify it is on.)
```

```bash
java -Xmx512m -Xss512k \
     -XX:NativeMemoryTracking=summary \
     -XX:StartFlightRecording=duration=600s,filename=/tmp/leak.jfr,settings=profile \
     -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp/leak.hprof \
     -jar orderflow.jar
```

`-Xss512k` is deliberate: smaller stacks mean more threads before failure, making the
thread-count curve long enough to watch. The failure is identical at the default 1 MB.

### Part B — break it

Change `orderflow`'s order-detail endpoint to Trace 4's version — an
`Executors.newFixedThreadPool(4)` created inside the handler, never shut down. Run the
Topic 65 k6 profile at the baseline rate.

**Capture, every 30 seconds:**

```bash
curl -s localhost:8080/actuator/metrics/jvm.threads.live
jcmd <pid> Thread.print | grep -c '^"'

# thread names, grouped — the leak's fingerprint:
jcmd <pid> Thread.print | grep -oE '^"[^"]+"' | sed -E 's/-[0-9]+"/-N"/' | sort | uniq -c | sort -rn | head

# heap and native memory side by side:
jcmd <pid> GC.heap_info
jcmd <pid> VM.native_memory summary | grep -A 3 "Thread"
ps -o rss= -p <pid>          # RSS versus -Xmx
```

### Part C — what to look for, and what stays silent

| Observation | What it means |
|---|---|
| `jvm.threads.live` rising ~1,600/s and **never falling** | The leak. **The only clear signal in the entire stack.** |
| `pool-N-thread-M` names dominating the grouped output | Names identify the culprit immediately — which is why `Executors`' default naming is a gift here and why naming your own pools matters everywhere else |
| `GC.heap_info`: heap **healthy** throughout | Every heap-based alert stays green |
| `-Xlog:gc*`: **normal** | GC is not the problem and will never tell you there is one |
| RSS climbing far past `-Xmx`; NMT `Thread` category large | Where the memory went — Topic 80 |
| p99 degrading with **no** code or traffic change | Scheduler pressure from tens of thousands of runnable threads |
| **`grep -c "Found one Java-level deadlock"` = 0** | Nothing is deadlocked, and it is completely down |
| **`grep -c BLOCKED` ≈ 0** | Nothing is blocked either |
| Error rate: **zero** until the very end | The service is failing and reporting success |
| Finally: `java.lang.OutOfMemoryError: unable to create native thread` | Not a heap error. Check `ulimit -u` and the cgroup `pids.max`. |
| The OOM thrown from an endpoint that does **not** create pools | Whichever code asked for a thread next. The stack trace points away from the bug. |

**Write down, before fixing anything:** which of your existing dashboards would have caught
this, and how long before the OOM? For most services the honest answer is *none, and never*.
That is the drill's real finding.

### Part D — the wrong fix, run deliberately

Raise `-Xmx` from 512m to 2g and re-run. **Predict the result first.**

| Observation | What it means |
|---|---|
| Time to failure **unchanged or shorter** | Heap and thread stacks compete for the same address space and the same cgroup limit |
| RSS reaching the container limit sooner; possibly an OOMKill instead of a JVM error | You changed the failure's *shape* and not its cause |

**This is why recognising the message text matters.** `OutOfMemoryError` has at least four
distinct meanings and only one of them is fixed by more heap.

### Part E — fix, and prove the fix

Replace with one shared pool: a `@Bean` with `destroyMethod = "shutdown"`, bounded queue,
named threads, `ExecutorServiceMetrics.monitor(...)`. Re-run.

```bash
# thread count should be FLAT:
for i in $(seq 1 20); do curl -s localhost:8080/actuator/metrics/jvm.threads.live \
  | grep -o '"value":[0-9.]*'; sleep 15; done

# JFR: starts should now roughly equal ends
jfr print --events jdk.ThreadStart /tmp/leak.jfr | grep -c "jdk.ThreadStart"
jfr print --events jdk.ThreadEnd   /tmp/leak.jfr | grep -c "jdk.ThreadEnd"
```

| Observation | What it means |
|---|---|
| Thread count flat across 20 samples under sustained load | Fixed |
| `jdk.ThreadStart` ≈ `jdk.ThreadEnd` | The JFR form of the same proof, and the one you can run in production |
| `jdk.ThreadStart` ≫ `jdk.ThreadEnd` | Still leaking somewhere else — this is how you find the second one |

### Part F — add the alert that makes it permanent

```
alert: ThreadCountMonotonic
expr:  min_over_time(jvm_threads_live[30m]) > min_over_time(jvm_threads_live[30m] offset 30m)
for:   1h
```

A thread count whose *minimum* keeps rising is a leak, regardless of source. This one alert
catches the per-request executor, the `new Thread()` in a loop, the pool never shut down on
context reload, and the third-party library that starts a thread per connection.

### What the drill proves

- A thread leak is invisible to **every** heap-based diagnostic, and the error message
  actively misdirects you.
- The deadlock detector, `BLOCKED` counts, GC logs, error rates and latency percentiles were
  **all silent or misleading** throughout.
- **One gauge** — `jvm.threads.live` — separates this from the other three failures in
  seconds.
- The obvious fix for the error message makes it worse, and you have now watched that
  happen rather than been told about it.

---

## Measurement

### The instrument for each claim

| Claim you want to make | The right instrument | The wrong instrument |
|---|---|---|
| "it is deadlocked" | `jcmd Thread.print` → `Found one Java-level deadlock` **or** the state histogram plus the wait-for reasoning | `findDeadlockedThreads()` returning null (positive-only signal) |
| "it is livelocked" | **three** dumps 10 s apart: same stacks, `cpu=` climbing, business counter flat | one dump; CPU alone; a flame graph alone |
| "a consumer is starved" | **maximum** age of the oldest waiting item, **per class** | p50/p95/p99; queue depth; any mean |
| "threads are leaking" | `jvm.threads.live` over time; JFR `jdk.ThreadStart` vs `jdk.ThreadEnd` | heap metrics; GC logs; error rate |
| "it is a race, not a hang" | happens-before reasoning; jcstress (Topic 99) | a load test that passed |
| "it is stuck, not slow" | diff dump 1 against dump 3 | a single dump |
| "this thread is using CPU" | `top -H -p <pid>` matched to `nid=0x...` | the `RUNNABLE` state, which lies about socket reads |

### `jcmd Thread.print` — the commands, in the order you type them

```bash
# 1. three dumps: state, then whether state is CHANGING
for i in 1 2 3; do jcmd <pid> Thread.print -l > /tmp/d$i.txt; sleep 10; done

# 2. the JVM's own diagnosis
grep -A 40 "Found one Java-level deadlock" /tmp/d1.txt

# 3. the shape of the problem, in one line
grep "java.lang.Thread.State" /tmp/d1.txt | sort | uniq -c | sort -rn

# 4. leak check
for f in /tmp/d?.txt; do echo -n "$f "; grep -c '^"' $f; done

# 5. thread-name families — a leak's fingerprint
grep -oE '^"[^"]+"' /tmp/d1.txt | sed -E 's/[-0-9]+"$/-N"/' | sort | uniq -c | sort -rn | head

# 6. who holds what (monitors)
grep -E "waiting to lock|- locked <0x" /tmp/d1.txt | sort | uniq -c | sort -rn | head

# 7. what are j.u.c waiters parked on
grep -oE 'a java\.util\.concurrent\.[A-Za-z$.]*' /tmp/d1.txt | sort | uniq -c | sort -rn

# 8. stuck or slow?
diff <(grep -A1 '^"' /tmp/d1.txt) <(grep -A1 '^"' /tmp/d3.txt) | head -40

# 9. per-thread CPU, matched to the dump by hex native id
top -H -p <pid> -b -n 1 | head -25          # Linux
printf '%x\n' <linux-tid>                    # then grep nid=0x<that> in the dump
```

**On macOS**, `top -H` is not available in that form. Use
`top -pid <pid> -stats pid,command,cpu` for the process, and prefer JFR's
`jdk.ExecutionSample` for per-thread attribution — portable, and the better tool anyway.

### The programmatic detector — ship this

```java
@Component
public class DeadlockHealthIndicator implements HealthIndicator {
    private final ThreadMXBean threads = ManagementFactory.getThreadMXBean();

    @Override public Health health() {
        long[] ids = threads.findDeadlockedThreads();      // monitors AND ownable Locks
        if (ids == null) {
            // NOT "healthy". Only "no DETECTABLE deadlock". Semaphores, latches,
            // connection pools and livelock are all invisible here.
            return Health.up().withDetail("deadlock.detectable", false).build();
        }
        ThreadInfo[] info = threads.getThreadInfo(ids, true, true);
        return Health.down()
                .withDetail("deadlockedThreads",
                        Arrays.stream(info).map(ThreadInfo::getThreadName).toList())
                .withDetail("lockedOn",
                        Arrays.stream(info).map(i -> String.valueOf(i.getLockInfo())).toList())
                .build();
    }
}
```

**Two disciplines around it:** wire it to **readiness**, never liveness (Topic 121), so a
deadlocked pod leaves the load balancer and stays alive to be dumped; and log the detail at
ERROR so the evidence survives if the pod is later killed.

### JFR — for the questions a dump cannot answer

```bash
java -XX:StartFlightRecording=duration=300s,filename=/tmp/tax.jfr,settings=profile -jar orderflow.jar

jfr summary /tmp/tax.jfr
jfr print --events jdk.ThreadStart,jdk.ThreadEnd /tmp/tax.jfr | head -40   # leak
jfr print --events jdk.JavaMonitorEnter /tmp/tax.jfr | head -40            # monitor contention
jfr print --events jdk.ThreadPark /tmp/tax.jfr | head -40                  # j.u.c waits
jfr print --events jdk.ExecutionSample /tmp/tax.jfr | head -40             # livelock's hot loop
```

| Event | Answers |
|---|---|
| `jdk.ThreadStart` far exceeding `jdk.ThreadEnd` | **thread leak**, quantified, in production, without a heap dump |
| `jdk.JavaMonitorEnter` with long durations | monitor contention (Topic 85) — a hot lock, not necessarily a deadlock |
| `jdk.ThreadPark` aggregated by parked-on class | which `j.u.c` construct is absorbing the wait — the Topic 97 triage |
| `jdk.ExecutionSample` concentrated in a retry loop | **livelock's positive signature**: CPU is being burned *here* |
| All four quiet while the service is down | look at the downstream, not the JVM |

### Micrometer — the four gauges that make this diagnosable in advance

Forward-reference Topic 118; this is the minimum that makes all four bugs visible before an
incident:

```java
// 1. THREAD LEAK — the single highest-value gauge in this document
jvm.threads.live            // Spring Boot's JvmThreadMetrics; verify it is registered

// 2. STARVATION — a maximum, per class. Never a mean.
orderflow.queue.oldest.age.seconds{tier="standard"}

// 3. DEADLOCK/HANG — in-flight requests that never complete
orderflow.requests.inflight  // rising while throughput falls = requests are not returning

// 4. LIVELOCK — CPU per unit of work, not CPU alone
process.cpu.usage / orderflow.orders.completed   // a ratio that explodes is livelock
```

**The fourth is the subtle one and worth building.** CPU alone cannot distinguish "busy"
from "livelocked"; CPU **per completed unit of work** can, and it is the only dashboard panel
that makes livelock visible without a thread dump.

### The standing rule: a naive `System.nanoTime()` loop is wrong

Every "is this fix faster" question in this document goes through JMH, never a hand-rolled
timing loop: no warm-up means you measure the interpreter, C2 may delete uncontended locking
entirely (Topic 75), single-threaded runs cannot exhibit any bug here, and one run has no
variance. Topic 77.

### `perf` — and the honest note about macOS

```bash
# Linux only — livelock's system-level fingerprint:
perf stat -e context-switches,cpu-migrations,task-clock -p <pid> -- sleep 30
```

A context-switch rate far above your request rate, with throughput at zero, corroborates
livelock or a thread-count explosion. **On macOS this is unavailable** — `perf` is a Linux
kernel facility that does not exist on Darwin, and I will not invent an equivalent. Use JFR's
`jdk.ExecutionSample` and `jdk.ThreadPark`, which answer the same questions portably.

---

## Practice exercises

### 1 — Easy: build the four-dump reference card

Run all four bugs from Example 1 as separate processes. For each, capture two dumps ten
seconds apart and record the `Thread.State` histogram, whether `Found one Java-level
deadlock` appears, whether `cpu=` changes between dumps, and the thread count.

**Deliverable:** a one-page table — *four symptoms, four dump signatures* — from your own
JVM. Pin it where you will find it at 3am. **The `cpu=` column is the one you will use
most**: it separates deadlock from livelock in five seconds.

### 2 — Medium: the diagnostic harness (combines 79, 85, 87, 90, 92, 93, 94, 97)

Build `orderflow`'s `/actuator/concurrency-report` endpoint that produces, on demand:

- the thread-state histogram,
- `findDeadlockedThreads()` output **with an explicit note that null is not evidence**,
- thread count grouped by name prefix, with a rolling 30-minute minimum for leak detection,
- for each registered `BlockingQueue`: depth, remaining capacity, and **oldest-item age**,
- for each registered `Semaphore`: available permits, queue length, and configured maximum,
- the CPU-per-completed-order ratio.

Then write four integration tests that trigger each bug in a Testcontainers-backed
`orderflow` and **assert the report correctly names the diagnosis**. The starvation test is
the hard one: assert that the oldest standard-tier item's age exceeds a threshold while p99
stays inside the Topic 65 baseline — that assertion *is* the definition of starvation, and
writing it is the exercise.

### 3 — Hard: production simulation and the runbook

1. Run the assigned Failure drill (Parts A–F) end to end under the k6 profile. Produce the
   thread-count-over-time chart for the leaking and fixed versions.
2. Reproduce Trace 1's deadlock in `orderflow` under load. Capture the dump section. Fix by
   global ordering. Then **deliberately reintroduce it** with `tryLock` and a fixed backoff,
   producing Trace 2's livelock. Capture the dumps and CPU for both. **Two failures, one
   root cause, opposite signatures** — this is the exercise's core.
3. Reproduce Trace 3's starvation with a `PriorityBlockingQueue` and a saturating
   high-priority stream. Show that every aggregate metric stays green. Add oldest-item age,
   partition the queue, re-measure.
4. Write the **runbook page**: "The service is not responding." Four branches, each with the
   exact command, the exact string to grep, the confirming evidence, and the fix. One page.
   Then hand it to a colleague and have them run an incident you trigger without telling
   them which of the four it is.

**Point 4 is the deliverable.** If the runbook does not let someone else diagnose your
incident without you, it is not finished — and that is the standard Phase 12 will hold you
to for every artefact.

---

## Interview questions

### Q1 — "The service is hung. Walk me through diagnosing it."

**MID-LEVEL ANSWER.** "I'd check the logs and the CPU and memory graphs. Then I'd take a
thread dump and look for a deadlock. If there's no deadlock I'd restart it and see whether
it comes back."

**SENIOR ANSWER.** "First, and before any tooling: **do not restart.** That destroys the only
evidence and guarantees we're here again next week. I'd pull the pod out of the load balancer
via readiness — Topic 121's distinction between liveness and readiness exists exactly for
this — and leave the process alive.

Then three thread dumps, ten seconds apart. One dump gives state; three tell me whether the
state is *changing*, which is the difference between stuck and slow.

Then I branch, because 'hung' is four different diagnoses:

- **`Found one Java-level deadlock`** in the dump — deadlock. The section names the threads,
  the monitors and the acquisition orders, so the fix is already visible.
- **Threads `RUNNABLE`, same stacks in all three dumps, `cpu=` climbing, throughput at
  zero** — livelock. Almost always a retry loop with no jitter.
- **Everything looks healthy but one class of work never completes** — starvation. Aggregates
  hide it; I'd look at the age of the oldest item per class.
- **Thread count rising across the three dumps** — a thread leak, heading for
  `OutOfMemoryError: unable to create native thread`.

Two things I'd say explicitly. **`findDeadlockedThreads()` returning null is not evidence
that nothing is stuck** — it only sees monitors and ownable `Lock`s, so a `Semaphore` cycle,
a `CountDownLatch` that never opens, or a HikariCP pool exhaustion all come back null and are
genuine deadlocks by the Coffman definition. And **`RUNNABLE` doesn't mean running** — a
thread blocked on a socket read is `RUNNABLE` at zero CPU, so I corroborate with per-thread
CPU rather than trusting the state.

For this service I'd expect the fourth branch first, because I know we fan out per request,
and I'd check `jvm.threads.live` before I even opened the dump."

**What separates them.** The mid answer has one hypothesis and one tool, and its final step
destroys the evidence. The senior answer opens with evidence preservation, has a **four-way
decision procedure** with a named discriminator for each branch, knows which of its
instruments is **unreliable in the negative direction**, and knows that a thread state can
lie. Naming the ambiguity in your own evidence is the strongest signal in this answer.

**Follow-up:** *"You found nothing in the dump and CPU is low."* — Then the threads are
waiting on something ownerless. I'd read the `parking to wait for` class: a `Semaphore` means
a bulkhead at capacity or a permit leak, and the permits gauge tells me which; a
`BlockingQueue` condition means an empty or full queue, and the depth gauge tells me which;
Hikari frames mean the pool deadlock. Each has a second signal, and the dump alone never
resolves any of them.

### Q2 — "How do you prevent deadlock?"

**MID-LEVEL ANSWER.** "Always take locks in the same order. And use `tryLock` with a timeout
so threads don't wait forever. Keep critical sections short."

**SENIOR ANSWER.** "Deadlock needs all four Coffman conditions **simultaneously**: mutual
exclusion, hold-and-wait, no preemption, and circular wait. So prevention is: **remove
exactly one.** Which one is the design decision.

**Circular wait** is the one I break by default, with a **global lock ordering** — every path
takes locks in the same total order. It costs nothing at runtime: no timeouts, no retries, no
lost concurrency. The whole cost is at design time, and the trap is enforcement: I'd route
every multi-lock acquisition through a single helper and add an ArchUnit rule forbidding
direct `lock()` calls outside it, because review catches this four times in five and the
fifth is an outage.

**No preemption** is the safety net: `tryLock(timeout)`, so a thread gives up and releases
what it holds. Important caveat — that converts a hang into an error only if you add
**randomised backoff and a retry budget**. Without jitter, both threads fail, release and
retry on the same schedule, collide again, and you have traded deadlock for livelock: 100%
CPU, zero throughput, and no deadlock section in the dump to tell you what happened.

**Hold-and-wait** you break by acquiring everything atomically or nothing — a global acquire
point, which usually costs too much concurrency.

**Mutual exclusion** you break with immutability, and that is the strongest fix because it
removes the possibility rather than managing it. Where `orderflow` can use an atomic
conditional UPDATE in the database instead of two JVM locks, that is better than any ordering
scheme.

And I'd generalise it: this isn't about `synchronized`. **Topic 109's HikariCP pool deadlock
is the same four conditions with connections as the resource** — ten threads each holding one
connection and waiting for a second from a pool of ten. `findDeadlockedThreads()` returns
null, no lock is involved, and the fix is still 'break one condition': don't hold a connection
while acquiring another, or size the pool above the maximum simultaneous holdings."

**What separates them.** The mid answer is a list of tips with no theory, and it recommends
`tryLock` without the caveat that turns it into a different outage. The senior answer states
the theory (**all four**, remove **one**), ranks the fixes by cost, names the enforcement
problem, and — the strongest move — generalises past locks to connection pools, which is
where the expensive version of this bug actually lives.

**Follow-up:** *"Your ordering is by SKU, and now you need two SKUs."* — Then I need a total
order over SKUs, not just over lock *types*: sort the keys and lock in sorted order. That's a
standard technique and it is exactly what a bank transfer between two accounts requires. And
I'd note the identity trap: if two threads get different `ReentrantLock` objects for the same
SKU because the registry used `get`-then-`put` instead of `computeIfAbsent`, the ordering is
correct and the mutual exclusion is gone.

### Q3 — "What's the difference between deadlock and livelock, and how do you tell them apart in production?"

**MID-LEVEL ANSWER.** "Deadlock means threads are stuck waiting for each other. Livelock means
they're doing something but not making progress. Livelock is rarer."

**SENIOR ANSWER.** "Same business outcome, opposite mechanisms, opposite evidence.

Deadlock: threads are **parked**. `BLOCKED` for `synchronized`, `WAITING (parking)` for
`j.u.c`. CPU drops. `cpu=` in the dump stops increasing. The JVM detects it and prints
`Found one Java-level deadlock` with the threads and monitors named. It is permanent — the
JVM reports it and never breaks it, unlike Postgres, which kills a victim.

Livelock: threads are **running**. `RUNNABLE` or `TIMED_WAITING`, CPU near 100%, `cpu=`
climbing fast across dumps, identical stacks each time, and a business counter that isn't
moving. There is **no** deadlock section, which misleads anyone whose procedure is to grep for
one.

So the discriminator is two things I can check in about thirty seconds: **is `cpu=` changing
between two dumps**, and **is the state `RUNNABLE` or parked**. Deadlock is frozen and quiet;
livelock is frantic and equally stuck.

The reason it matters operationally is that livelock's high CPU actively misleads: it reads as
compute-bound, so autoscaling adds pods, which livelock immediately and add load. That's a
feedback loop deadlock doesn't have.

In practice livelock is usually a deadlock fix applied without jitter — `tryLock` with a fixed
backoff, so both threads desynchronise and then re-synchronise perfectly on the next attempt.
The fix is randomised backoff plus a retry budget, and the deeper fix is to make ordering the
primary mechanism and `tryLock` only the safety net."

**What separates them.** The senior answer gives a **thirty-second field test** rather than a
definition, knows the deadlock-detector's silence is expected in one case, and — the detail
that shows real experience — knows livelock's high CPU causes autoscaling to make it worse.

### Q4 — "You get `OutOfMemoryError: unable to create native thread`. What now?"

**MID-LEVEL ANSWER.** "That's an out-of-memory error, so I'd increase the heap with `-Xmx`
and add memory to the container."

**SENIOR ANSWER.** "That message specifically means the **OS refused to create a thread** —
it has nothing to do with the heap, and **raising `-Xmx` makes it worse**, because heap and
thread stacks compete for the same process address space and the same container memory limit.
That's the trap in the message text, and it's why I read the message rather than the class
name.

Four candidate causes, and I check rather than guess: the per-user process limit
(`ulimit -u`), the container's `pids.max` cgroup, the system-wide `threads-max`, and address
space — each platform thread reserves about 1 MB of stack by default, so forty thousand
threads is roughly forty gigabytes of reservation.

But the real question is **why do we have that many threads**, and that's almost always a
leak: an `ExecutorService` created per request and never shut down, a `new Thread()` in a
loop, or a pool not shut down on context reload. A running thread is a GC root, so the
executor is unreachable and uncollectable, and if the threads are non-daemon they also stop
the JVM exiting.

I'd confirm with `jvm.threads.live` over time — a count that only rises — and group the thread
names in a dump, which usually names the culprit immediately because the default
`pool-N-thread-M` naming tells you which factory made them. JFR's `jdk.ThreadStart` versus
`jdk.ThreadEnd` gives the same answer in production without a dump.

Fix: one shared, bounded, named pool as a `@Bean` with `destroyMethod = "shutdown"`, an
ArchUnit rule banning `Executors.newFixedThreadPool` outside configuration, and an alert on a
monotonically rising thread count. That last one catches every future variant, including ones
in libraries.

One nuance: on virtual threads this failure changes shape. You won't exhaust OS threads, so
you'll get `Java heap space` instead — same bug, different message, same fix."

**What separates them.** The mid answer applies the obvious remedy suggested by the exception
class and makes the problem worse. The senior answer knows the message text is the diagnosis,
knows the four resource limits and how to check each, treats the count as a **symptom of a
leak** rather than a limit to raise, and knows how the failure re-shapes under virtual threads.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. All four Coffman conditions must hold for deadlock, and breaking one is sufficient. Lock
   ordering breaks circular wait; `tryLock` breaks no-preemption. Which condition does making
   your data **immutable** break, and why is that fix qualitatively different from the other
   two rather than merely better?

2. The JVM detects deadlocks and reports them but never breaks one; Postgres detects them and
   kills a victim. Argue that the JVM's choice is correct. Then make the strongest case
   against yourself.

3. `findDeadlockedThreads()` cannot see a cycle through a `Semaphore`, because a semaphore
   records no owner. Design the smallest change to `Semaphore` that would make such cycles
   detectable, and say precisely what that change would cost — including what it would break
   about the class's existing uses.

4. Deadlock and livelock produce the same customer symptom from opposite CPU signatures.
   Construct a scenario in which a system **alternates** between the two, and explain what an
   operator watching only CPU would conclude.

5. Starvation is invisible to means and percentiles when the starved class is a small
   fraction of volume. State the general property a metric must have to detect "is anyone
   being ignored", and name one other production problem the same property would catch.

6. A leaked thread is uncollectable because a running thread is a GC root. Explain why making
   the threads **daemon** changes the JVM's exit behaviour but does not fix the leak — and
   describe a case where daemon threads make the leak *harder* to notice.

7. Topic 109's HikariCP pool deadlock satisfies all four Coffman conditions with connections
   as the resource, yet no JVM tool reports a deadlock. Which of the four conditions is
   cheapest to break there, and why is the answer different from the JVM-lock case?

---

## Quick reference card

### The decision table — four symptoms, four diagnostics, four fixes

| Symptom | Diagnosis | Confirm with | Fix |
|---|---|---|---|
| hung threads in a cycle, CPU **idle**, latency flat, logs silent | **deadlock** | `Found one Java-level deadlock`; `BLOCKED` (monitors) or `WAITING (parking)` (`Lock`s); `cpu=` frozen | break one Coffman condition — **lock ordering** first, `tryLock`+jitter second |
| **100% CPU**, zero throughput | **livelock** | 3 dumps: `RUNNABLE`, same stacks, `cpu=` **climbing**; no deadlock section | **randomised backoff + a retry budget** |
| all aggregates healthy, one class never progresses | **starvation** | **max** age of oldest item, **per class**; `top -H` showing one idle thread among busy peers | fairness, ageing, or **partition** the resource |
| thread count only **grows**, then `unable to create native thread` | **thread leak** | `jvm.threads.live` monotonic; `jdk.ThreadStart` ≫ `jdk.ThreadEnd`; name-prefix grouping | one shared bounded named pool, shut down in `@PreDestroy` |
| **wrong results, no hang** | **data race** | no runtime diagnostic — happens-before reasoning, jcstress | establish the missing edge |

### The four Coffman conditions and what removes each

```
mutual exclusion  -> immutable data, or per-thread copies        (strongest, hardest)
hold and wait     -> acquire all-or-nothing at one point         (costs concurrency)
no preemption     -> tryLock(timeout) + JITTER + a budget        (safety net; livelock risk)
circular wait     -> a global lock order, enforced structurally  (PRIMARY FIX)
```

### Thread states — the diagnostic table

| State | Cause | Implies |
|---|---|---|
| `RUNNABLE` | executing, **or** in a native call (socket reads!) | not proof of CPU; + high CPU + no progress = **livelock** |
| `BLOCKED` | `synchronized` only | monitor contention; in a cycle = deadlock |
| `WAITING` | `wait()`, `join()`, **all `j.u.c` parking** | **can be permanent**; also what a healthy idle worker looks like |
| `TIMED_WAITING` | `sleep`, `*(t,u)` variants | **will wake up**; cannot be a permanent deadlock |

### Triage commands

```bash
for i in 1 2 3; do jcmd <pid> Thread.print -l > /tmp/d$i.txt; sleep 10; done
grep -A 40 "Found one Java-level deadlock" /tmp/d1.txt
grep "java.lang.Thread.State" /tmp/d1.txt | sort | uniq -c | sort -rn
for f in /tmp/d?.txt; do echo -n "$f "; grep -c '^"' $f; done      # leak
grep -oE '^"[^"]+"' /tmp/d1.txt | sed -E 's/[-0-9]+"$/-N"/' | sort | uniq -c | sort -rn
grep -oE 'a java\.util\.concurrent\.[A-Za-z$.]*' /tmp/d1.txt | sort | uniq -c
diff <(grep -A1 '^"' /tmp/d1.txt) <(grep -A1 '^"' /tmp/d3.txt)     # stuck vs slow
```

### The four gauges that make all of this visible in advance

```
jvm.threads.live                              -> thread leak
orderflow.queue.oldest.age.seconds{tier=...}  -> starvation (a MAX, per class)
orderflow.requests.inflight                   -> hangs of any kind
process.cpu.usage / orders.completed          -> livelock (CPU per unit of work)
```

### Gotchas checklist

- [ ] **Never restart before dumping.** Readiness out, process alive.
- [ ] Three dumps, not one.
- [ ] `findDeadlockedThreads() == null` is **not** evidence of health.
- [ ] `RUNNABLE` is not evidence of CPU use; corroborate with `top -H` or JFR.
- [ ] No `Executors.newFixedThreadPool` outside a `@Bean`; ArchUnit-enforced.
- [ ] Every pool is bounded, named, metered, and shut down in `@PreDestroy`.
- [ ] Every retry has randomised jitter **and** a budget.
- [ ] Multi-lock acquisition goes through one ordered helper; direct calls are banned.
- [ ] Lock registries use `computeIfAbsent`, never `get`-then-`put`.
- [ ] Starvation metrics are maxima, per class — never means.
- [ ] `OutOfMemoryError: unable to create native thread` is **not** a heap problem.

---

## When would I use this at work?

**1. The 3am page where the answer is "which of the four".**

Requests stop completing, error rate from the LB only, application logs silent. You pull the
pod from the load balancer, take three dumps, and run four greps. In under two minutes you
know whether it is deadlock, livelock, a `j.u.c` wait, or a thread leak — and, just as
importantly, you know that the deadlock detector's silence proved nothing. **The value is not
the fix. It is that you did not restart the pod**, which is what happens on most teams and
which guarantees the same incident next week with the same zero information.

**2. Writing the runbook before the incident.**

One page: "The service is not responding." Four branches, each with the exact command, the
exact string to grep, the confirming evidence, and the fix. Then a game day where a colleague
diagnoses an incident you triggered without telling them which. **The runbook is the
deliverable, and its test is that it works without you in the room.** This is the single
highest-leverage document an on-call rotation can own, and almost nobody writes it.

**3. Reviewing a PR that adds concurrency.**

Someone adds a second lock, or a retry loop, or an executor. Three questions, none of them
style comments: *"is there another path that takes these two locks, and in what order?"*,
*"does that retry have jitter and a budget?"*, and *"who shuts this pool down?"* Each maps to
one row of the decision table and each catches a specific outage before it ships. **The
questions come from the taxonomy**, which is exactly why having the taxonomy in your head —
rather than a general instinct that concurrency is risky — is what makes the review useful.

---

## Connected topics

**Prerequisites:**

- **85 — `synchronized` and intrinsic monitors.** Why `BLOCKED` exists, why the JVM knows a
  monitor's owner, and therefore why deadlock detection works at all for `synchronized` and
  not for a `Semaphore`.
- **86 / 87 / 88 — the JMM, `volatile` and safe publication.** The data-race row of the
  taxonomy in full. This document says a race has no runtime diagnostic; those topics say
  what to reason about instead, and Topic 88 gives the immutability fix that removes
  Coffman's mutual exclusion outright.
- **90 — executors and pool sizing.** The thread leak's home. Bounded pools, named threads,
  and the `shutdown` → `awaitTermination` → `shutdownNow` sequence, without which the drill's
  fix does not hold.
- **93 — blocking queues.** Where two of the four hangs actually live in a service: a
  producer parked in `put` on a full queue, and consumers parked in `take` — indistinguishable
  from health without the depth gauge.
- **94 — explicit locks and AQS.** The deadlock in depth: the wallet/inventory lock ordering,
  the `tryLock` safety net, and the BLOCKED-versus-WAITING table this document depends on.
  Topic 94's Part D drill produces the livelock this document traces.
- **97 — coordination primitives.** Two of the hardest cases here: a permit leak is a
  permanent hang that is **not** a deadlock and that the detector cannot see, and non-fair
  semaphores are a principal source of starvation.
- **92 — `ConcurrentHashMap`.** `computeIfAbsent` for the lock registry. A `get`-then-`put`
  race hands two threads two different locks for the same key and silently deletes the mutual
  exclusion your ordering fix depends on.
- **79 — memory leaks and heap dumps.** The sibling failure, and the `ThreadLocal`-on-a-
  pooled-thread leak specifically: the same "nobody removes it" shape with retention rather
  than threads as the resource.
- **80 — off-heap and native memory.** Why RSS exceeds `-Xmx` during a thread leak, and why
  `jcmd VM.native_memory` shows the Thread category dwarfing the heap.
- **65 — the load-testing gate.** The drill needs the baseline; "it degraded" requires a
  number it degraded from.

**This unlocks:**

- **99 — jcstress.** How you would *prove* a fix correct rather than fail to reproduce a bug
  and call it fixed. The data-race row of the taxonomy gets its real tool here — and the
  reminder that a forbidden outcome with zero observations means "not observed", not
  "impossible".
- **101 — virtual threads.** Re-shapes three of the four bugs. Thread leaks become heap
  exhaustion rather than native-thread exhaustion; pinning is a **new** starvation mechanism
  where a handful of pinned virtual threads starve every carrier; and `synchronized`-versus-
  `ReentrantLock` becomes a scheduling question rather than a performance one.
- **102 — structured concurrency.** The structural fix for the thread leak: subtask lifetimes
  bound to a lexical scope means there are no orphan tasks by construction, and `ScopedValue`
  removes the `ThreadLocal` leak from Topic 79.
- **109 — the HikariCP pool deadlock.** **The same four Coffman conditions with database
  connections as the resource.** Ten threads each holding one connection and each waiting for
  a second from a pool of ten. No JVM lock is involved, `findDeadlockedThreads()` returns
  `null`, and the fix is still "break exactly one condition". This is the most valuable
  forward reference in the document, because it is where the expensive version of this bug
  actually happens in production Java.
- **111 — bulkheads, retries and circuit breakers.** Retry-with-jitter at the network scale is
  the same fix as retry-with-jitter at the lock scale, and for the same reason: fixed
  intervals re-synchronise contenders. Bulkheads keep one subsystem's hang from consuming the
  shared request pool, which is what turns a two-thread deadlock into a whole-service outage.
- **118 — Micrometer metrics.** The four gauges, done properly — especially the two nobody
  builds: oldest-item age per class, and CPU per unit of completed work.
- **121 — liveness versus readiness probes.** The operational precondition for everything
  here: a hung pod must leave the load balancer and **stay alive** so it can be dumped. A
  liveness probe that restarts it destroys the evidence automatically, forever, which is the
  most common reason these bugs go undiagnosed for months.
- **131 — design-doc authorship.** The runbook from Exercise 3 is the artefact, and its test
  is that a colleague can run your incident without you.

---

*Java baseline 21, running on JDK 25. One thing here is deliberately hedged rather than
asserted: exactly which synchronizers `findDeadlockedThreads()` covers on your JDK — it
covers monitors and ownable `Lock`s, and Proof 4 settles the `Semaphore` case on your own
machine in two minutes. Everything else — the four Coffman conditions, that breaking one is
sufficient, that the JVM reports deadlocks and never breaks one, that `RUNNABLE` does not
mean running, that `unable to create native thread` is not a heap problem, and that "the
service is hung" is four diagnoses rather than one — has been stable for the entire life of
the platform. It will still be true the next time a service goes quiet at 3am, and the only
question that will matter then is whether you take the dump before someone restarts the pod.*
