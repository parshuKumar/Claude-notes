# 90 — `ExecutorService`: Pool Sizing, Queue Choice, Rejection, Shutdown

## Phase: 9 — Concurrency
## Category: ELITE
## Java baseline: 21  |  Notes features from: 21 (`ExecutorService.close()` arrived in 19; `newVirtualThreadPerTaskExecutor` in 21)
## Project spine: the `orderflow` async payment-notification pool — the executor that fires the "payment confirmed" webhook and email after an order commits. You will size it from Little's Law, bound its queue from a latency budget, choose its rejection policy on purpose, and shut it down correctly. Then you will remove the bound and watch the JVM die.

---

## Mechanical statement

Read this until you can recite it, because it is counter-intuitive and almost everyone
gets it backwards.

> **`ThreadPoolExecutor` creates threads up to `corePoolSize`. After that it QUEUES.
> It only grows toward `maximumPoolSize` when the queue is FULL.**
>
> Therefore: **with an unbounded queue, `maximumPoolSize` is dead configuration.** The
> queue never fills, so the pool never grows past `corePoolSize`, and the queue
> absorbs unbounded work.

Say the consequence out loud, because it is the whole topic:

> **An unbounded queue converts a throughput problem into a latency-and-memory
> problem, silently.** Throughput stays exactly flat at whatever the core threads can
> manage. Queue depth grows without bound. Latency grows without bound. Memory grows
> without bound. Nothing is rejected, nothing is logged, no error rate moves — until
> the heap is gone and the process dies.

And the corollary that most people find backwards:

> **A `maximumPoolSize` of 200 with an unbounded queue gives you a pool of
> `corePoolSize`. A `maximumPoolSize` of 200 with a queue of capacity 1 gives you a
> pool of 200 almost immediately.** The queue is not a buffer *in front of* the growth
> policy; it is the *trigger* for it.

`Executors.newFixedThreadPool(n)` is `new ThreadPoolExecutor(n, n, 0L, MILLISECONDS,
new LinkedBlockingQueue<>())` — an unbounded queue by construction. This is not a
footnote. It is the default that ships in every Java codebase on earth.

---

## The bridge from what you know

### The thing you already have — and it is a genuine analogue

Node is single-threaded for **your** JavaScript. It is not single-threaded for
everything. libuv keeps a real thread pool for the operations that cannot be done with
epoll:

```bash
UV_THREADPOOL_SIZE=4    # the default. Yes, four.
```

That pool serves `fs.*`, `dns.lookup`, `zlib`, and `crypto.pbkdf2`. And you have
already been bitten by it, or you have read the blog post: fire sixteen concurrent
`crypto.pbkdf2` calls with a pool of four and the last four complete at roughly four
times the latency of the first four. Throughput is capped at four; the rest queue.

**That is exactly this topic.** A fixed number of worker threads, an unbounded queue in
front of them, and a latency that grows linearly with backlog while throughput stays
pinned.

**Verdict: PARTIAL — and a good one.** What transfers: the pool-plus-queue shape, the
"latency grows, throughput does not" signature, and the instinct that the pool size is
a capacity decision. What does not: libuv's pool is one global, opaque, environment-
variable-configured thing you cannot instrument. `ThreadPoolExecutor` is your object.
You choose the queue, the bound, the rejection policy, the thread factory, and the
shutdown semantics — and you are responsible for every one of those choices.

### Little's Law — you already own this

You know this from system design. I am not going to re-derive it.

> **L = λW.** In-flight items = arrival rate × time in system.

Applied here:

> **threads ≈ throughput × latency**

At 100 tasks/second with a mean task latency of 150 ms, you need
`100 × 0.15 = 15` concurrent workers to keep up. That is the sizing calculation, and
it is the same one you have done a hundred times for queue workers and connection
pools.

**Three adjustments Java forces on you that your Node instincts do not cover:**

1. **CPU-bound is a different formula.** For work that is genuinely burning CPU, more
   threads than cores makes it *slower* — context switches and cache pollution, with
   no additional parallelism available. The size is `N_cpu` (or `N_cpu + 1`), full
   stop. Little's Law applies to *waiting*, and CPU-bound work is not waiting.
2. **A thread is expensive here in a way a Promise is not.** A platform thread reserves
   stack address space (1 MB default on 64-bit Linux, tunable with `-Xss`) and is an
   OS scheduling entity. 240 threads is a design decision; 240 pending Promises is
   Tuesday. This asymmetry is exactly what virtual threads (Topic 101) remove.
3. **You must pick a queue bound, and Java will not pick a sane one for you.** Node's
   libuv queue is unbounded too, but Node's failure mode is a busy event loop. Java's
   failure mode is `OutOfMemoryError`, several minutes after the queue started growing,
   with no warning in between.

### What has no analogue at all

| You know | Java | Verdict |
|---|---|---|
| `UV_THREADPOOL_SIZE` | `corePoolSize` / `maximumPoolSize` | **PARTIAL** — same idea, vastly more control and more rope |
| Little's Law for sizing workers | Little's Law for sizing pools | **HONEST ANALOGUE** — use it directly |
| An unbounded internal queue you cannot see | `LinkedBlockingQueue` with `Integer.MAX_VALUE` capacity | **PARTIAL** — in Java it is yours to bound, which means it is your fault |
| — | The core-then-queue-then-max growth rule | **NO ANALOGUE** — nothing in Node behaves this way and the rule is counter-intuitive |
| — | Rejection policies (`Abort`/`CallerRuns`/`Discard`/`DiscardOldest`) | **NO ANALOGUE** — you have never had to choose what happens when the buffer is full |
| Unhandled rejection warning | `submit()` silently captures the exception in the `Future` | **NO ANALOGUE** — Node at least warns you. Java does not. |
| Process exit when the event loop drains | Non-daemon pool threads keep the JVM alive forever | **NO ANALOGUE** — an executor you never shut down is a process that never exits |

That last row deserves a sentence on its own. In Node, when there is nothing left to
do, the process exits. In Java, a single non-daemon pool thread will keep the JVM
running indefinitely after `main` returns. "The application finished but the process
did not exit" is a first-week Java experience and this is why.

---

## What is this?

`ExecutorService` is an interface for "give me work, I will run it on some thread". The
implementation you will use for essentially everything is `ThreadPoolExecutor`.

```java
new ThreadPoolExecutor(
    int corePoolSize,                    // threads kept alive even when idle
    int maximumPoolSize,                 // hard ceiling on threads
    long keepAliveTime, TimeUnit unit,   // how long a NON-core idle thread survives
    BlockingQueue<Runnable> workQueue,   // where tasks wait. THE decision.
    ThreadFactory threadFactory,         // names, daemon flag, uncaught handler
    RejectedExecutionHandler handler);   // what happens when full. ALSO the decision.
```

Seven parameters. Two of them — the queue and the rejection handler — are where every
production incident in this topic lives, and they are the two that the `Executors.*`
factory methods choose for you without telling you.

### The `execute()` algorithm, exactly

This is the mechanical statement in code. It is worth memorising the shape:

```
execute(task):
  1. if (workerCount < corePoolSize)
         → start a NEW thread with this task as its first job.        DONE.
  2. else if (pool is RUNNING and workQueue.offer(task) succeeds)
         → the task is now QUEUED. No new thread.                     DONE.
  3. else  (the queue REFUSED the task — i.e. it is bounded and full)
         if (workerCount < maximumPoolSize)
             → start a NEW thread with this task as its first job.    DONE.
         else
             → REJECT: handler.rejectedExecution(task, this).
```

Read step 2 and step 3 together. **The only path to a thread beyond `corePoolSize` is
`workQueue.offer` returning `false`.** An unbounded `LinkedBlockingQueue` never returns
`false`. Therefore step 3 never runs. Therefore `maximumPoolSize` is unreachable and
the rejection handler is unreachable.

That is why an unbounded queue disables *both* your growth policy and your backpressure
policy in one move.

### The queue choices, and what each one means

| Queue | Bounded? | Growth behaviour | When it is right |
|---|---|---|---|
| `LinkedBlockingQueue<>()` | **No** (`Integer.MAX_VALUE`) | Pool never exceeds core; queue grows to OOM | Essentially never in a service. This is the default and it is the trap. |
| `LinkedBlockingQueue<>(capacity)` | Yes | Fills, then pool grows to max, then rejects | **The default you should reach for.** Split put/take locks give good throughput. |
| `ArrayBlockingQueue<>(capacity)` | Yes | Same | When you want a fixed pre-allocated array and one lock — slightly lower throughput, slightly more predictable memory. |
| `SynchronousQueue<>()` | Capacity **zero** | `offer` fails unless a worker is *right now* waiting to take. So the pool grows to max immediately | Hand-off pools where you want threads created eagerly. This is what `newCachedThreadPool` uses — with `maximumPoolSize = Integer.MAX_VALUE`. |
| `PriorityBlockingQueue<>()` | **No** | Same trap as `LinkedBlockingQueue` | Rarely. And note it is unbounded, so it carries the same defect. |
| `DelayedWorkQueue` (internal) | **No** | Used by `ScheduledThreadPoolExecutor`; core-only | You do not choose this; you inherit it by using a scheduled pool. |

Topic 93 covers these classes properly. Here you only need one sentence per row and the
knowledge that **bounded is the default position, and unbounded needs a written
justification**.

### The four rejection policies

When step 3 fails, `RejectedExecutionHandler` runs **on the submitting thread**.

| Policy | What it does | The consequence you must accept |
|---|---|---|
| `AbortPolicy` (**default**) | Throws `RejectedExecutionException` | The submitter gets an exception. If the submitter is a Tomcat request thread, that becomes a 500 unless you catch it. **Loud, which is good.** |
| `CallerRunsPolicy` | Runs the task **on the submitting thread**, synchronously | Real backpressure: the submitter is now busy and cannot submit again. But if the submitter is a request thread, you have just converted an async path into a synchronous one and your request latency inherits the task latency. |
| `DiscardPolicy` | Silently drops the task. Returns normally. | **Almost never acceptable.** You have chosen to lose work with no record. For a payment notification this means the customer is never told. |
| `DiscardOldestPolicy` | Drops the head of the queue, then retries `execute` | You lose the *oldest* work — the task that has already waited longest. Occasionally right for "latest value wins" telemetry. Never right for anything transactional. |

**The one you write yourself.** In practice the right handler for `orderflow` is a
custom one that increments a metric, logs at WARN with the task's identity, and then
falls back to a durable path — writing the notification to the outbox table (Topic 115)
so a relay picks it up. Rejection is not an error; it is the system correctly declining
work it cannot do in time. Losing the work is the error.

### The `Executors.*` factory methods, and what each one hides

```java
Executors.newFixedThreadPool(n)
// = new ThreadPoolExecutor(n, n, 0L, MILLISECONDS, new LinkedBlockingQueue<>())
//   UNBOUNDED QUEUE. maximumPoolSize == corePoolSize, so it is moot anyway.
//   Under overload: latency and memory grow without bound. Nothing sheds.

Executors.newCachedThreadPool()
// = new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60L, SECONDS, new SynchronousQueue<>())
//   UNBOUNDED THREADS. Zero-capacity queue means every burst creates threads.
//   Under overload: OutOfMemoryError: unable to create native thread.

Executors.newSingleThreadExecutor()
// = a wrapper around ThreadPoolExecutor(1, 1, 0L, MILLISECONDS, new LinkedBlockingQueue<>())
//   UNBOUNDED QUEUE, and it is deliberately non-reconfigurable.

Executors.newScheduledThreadPool(n)
// = ScheduledThreadPoolExecutor(n) with an internal UNBOUNDED DelayedWorkQueue.
//   Also: it never grows past core, whatever you set for max.

Executors.newVirtualThreadPerTaskExecutor()   // Java 21
//   No pool at all. A new virtual thread per task. There is no queue and no
//   rejection — which means there is no backpressure either. Topic 101.
```

**The rule:** in application code, construct `ThreadPoolExecutor` explicitly. Every
`Executors.*` factory makes at least one unbounded choice on your behalf, and the point
of this topic is that those choices are yours.

You do not have to take my word for any of the above — read them yourself:

```bash
unzip -p "$JAVA_HOME/lib/src.zip" java.base/java/util/concurrent/Executors.java \
  | grep -n -A6 'public static ExecutorService newFixedThreadPool(int nThreads)'
```

**What to look for:** the `new LinkedBlockingQueue<Runnable>()` with no capacity
argument. That single missing argument is this entire topic.

---

## Why does it matter?

**1. It is the most common serious production defect in Java services.** Not a subtle
one — a common one. `Executors.newFixedThreadPool(200)` appears in essentially every
codebase, and the person who wrote it believed they had configured a bound. They
configured a *thread* bound. They left the *work* unbounded.

**2. The failure signature is uniquely confusing.** Throughput is flat. CPU is normal.
The database is fine. Error rate is zero. Every dashboard is green, and latency is
climbing linearly. Then the process dies with an OOM whose heap dump is 90% queue
nodes. Without this topic you will spend the incident looking at the database.

**3. It is where "backpressure" stops being a metaphor.** You already know backpressure
from Node streams and from system design. This is the first place in Java where you
have to write the number down. The queue capacity **is** the policy.

**4. Every higher-level async thing in Spring is one of these.** `@Async`,
`ThreadPoolTaskExecutor`, `spring.task.execution.*`, Kafka listener concurrency,
`WebClient`'s bounded elastic scheduler, `CompletableFuture.*Async`'s executor
argument — all of them are a `ThreadPoolExecutor` with defaults someone chose. If you
do not know the growth rule you cannot reason about any of them.

**5. Interviewers use "how do you size a thread pool?" as their single best filter.**
The mid-level answer is a number. The senior answer is a question back — "CPU-bound or
I/O-bound?" — followed by Little's Law and a statement about what happens when the
estimate is wrong. It is very hard to fake.

---

## Machine-level reality

### `ctl` — one `AtomicInteger` holding two things

`ThreadPoolExecutor`'s entire state machine lives in a single field:

```java
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));

private static final int COUNT_BITS = Integer.SIZE - 3;      // 29
private static final int COUNT_MASK = (1 << COUNT_BITS) - 1; // 2^29 - 1 workers max

private static final int RUNNING    = -1 << COUNT_BITS;
private static final int SHUTDOWN   =  0 << COUNT_BITS;
private static final int STOP       =  1 << COUNT_BITS;
private static final int TIDYING    =  2 << COUNT_BITS;
private static final int TERMINATED =  3 << COUNT_BITS;
```

**The high 3 bits are the run state. The low 29 bits are the worker count.** Both are
updated with a single CAS on one word, which is what makes "check the state and
increment the count" atomic without a lock. This is Topic 95's CAS applied to a packing
trick, and it is worth recognising because the same pattern appears throughout
`java.util.concurrent`.

The run states are **monotonically increasing** and the transitions are one-way:

| State | Accepts new tasks? | Runs queued tasks? | Interrupts workers? |
|---|---|---|---|
| `RUNNING` | yes | yes | no |
| `SHUTDOWN` | **no** | **yes** — drains the queue | only idle ones |
| `STOP` | no | **no** — queue is drained and returned | **yes, all of them** |
| `TIDYING` | no | no | — all workers gone, `terminated()` about to run |
| `TERMINATED` | — | — | `terminated()` has returned; `awaitTermination` unblocks |

`shutdown()` moves `RUNNING → SHUTDOWN`. `shutdownNow()` moves to `STOP` **and returns
the `List<Runnable>` it drained from the queue**. That return value is the only record
of the work you abandoned — throwing it away is how notifications get lost during a
deploy.

### The `Worker` — and why it extends `AbstractQueuedSynchronizer`

```java
private final class Worker extends AbstractQueuedSynchronizer implements Runnable {
    final Thread thread;         // the actual OS thread, from the ThreadFactory
    Runnable firstTask;          // the task it was created to run, or null
    volatile long completedTasks;
}
```

The `AQS` inheritance is not decoration. `Worker` uses a **non-reentrant** lock whose
only purpose is to answer one question: *is this worker currently running a task, or is
it idle in `getTask()`?*

That matters because `shutdown()` must interrupt **idle** workers (so they wake from
`take()` and notice the state change) without interrupting workers **mid-task** (which
would abort in-flight work). `interruptIdleWorkers()` calls `w.tryLock()`: if it
succeeds, the worker is idle and safe to interrupt.

It is deliberately non-reentrant so that a task which calls back into the pool's own
control methods cannot accidentally re-acquire it. Read the class comment:

```bash
unzip -p "$JAVA_HOME/lib/src.zip" \
  java.base/java/util/concurrent/ThreadPoolExecutor.java | sed -n '/class Worker/,/^        }/p'
```

### The worker lifecycle — `runWorker` and `getTask`

```
runWorker(w):
    task = w.firstTask
    loop:
        if task == null:
            task = getTask()               // blocks here when idle
            if task == null:               // getTask decided this worker should die
                break
        w.lock()                           // mark "busy" so shutdown() won't interrupt me
        beforeExecute(w.thread, task)
        try   { task.run() }
        catch (Throwable t) { thrown = t; throw t }
        finally { afterExecute(task, thrown); w.unlock(); task = null; completedTasks++ }
    processWorkerExit(w, completedAbruptly)
```

**`getTask()` is where the keep-alive policy actually lives:**

```java
boolean timed = allowCoreThreadTimeOut || workerCount > corePoolSize;

Runnable r = timed
        ? workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS)   // may return null → worker dies
        : workQueue.take();                                     // blocks forever → worker lives
```

Three facts fall out of those two lines, and each is a real-world consequence:

1. **Core threads block in `take()` forever by default.** They never time out. That is
   why an executor you never shut down keeps the JVM alive — `take()` on a non-daemon
   thread is an eternal parked state.
2. **`keepAliveTime` only applies above core.** Setting it on a `newFixedThreadPool`
   (where core == max) does nothing, which is why the factory passes `0L`.
3. **`allowCoreThreadTimeOut(true)` flips core threads to the timed path**, so an idle
   pool shrinks to zero. Useful for a rarely-used pool in a memory-constrained
   container. Note it requires a non-zero `keepAliveTime` or it throws.

`processWorkerExit` then decides whether to replace the dead worker — which is how the
pool self-heals after a task throws, and also how a task that throws *every* time
produces an endless churn of thread creation.

### What a pool looks like in a thread dump

*Illustration of the FORMAT, not captured output. `<n>` are placeholders.*

```
"orderflow-notify-7" #<n> [<n>] prio=5 os_prio=0 cpu=<n>ms tid=0x<n> nid=0x<n>
    waiting on condition  [0x<n>]
   java.lang.Thread.State: WAITING (parking)
        at jdk.internal.misc.Unsafe.park(Native Method)
        - parking to wait for  <0x<n>> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
        at java.base/java.util.concurrent.locks.LockSupport.park(LockSupport.java:<n>)
        at java.base/java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(...)
        at java.base/java.util.concurrent.LinkedBlockingQueue.take(LinkedBlockingQueue.java:<n>)
        at java.base/java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:<n>)
        at java.base/java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:<n>)
        at java.base/java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:<n>)
        at java.base/java.lang.Thread.runWith(Thread.java:<n>)
```

**How to read it — four fields:**

| Field | What it tells you |
|---|---|
| The **thread name prefix** (`orderflow-notify-`) | Which pool. **If this reads `pool-3-thread-7` you have not set a `ThreadFactory`, and you have just lost the ability to diagnose anything from a dump.** Naming your threads is the cheapest observability win in Java. |
| `LinkedBlockingQueue.take` + `getTask` + `runWorker` in the stack | This worker is **idle**, waiting for work. Not stuck. Not slow. Idle. |
| A stack that ends in **your** code instead of `getTask` | This worker is **busy**. Count these to get true active-thread count from a dump. |
| `cpu=<n>ms` compared across two dumps | Unchanged on a "busy" worker means it is blocked on I/O, not computing. |

**The single most useful `grep` in this topic:**

```bash
jcmd <pid> Thread.print | grep -c 'orderflow-notify'                       # pool size
jcmd <pid> Thread.print | grep -A9 'orderflow-notify' | grep -c 'getTask'  # idle workers
```

Busy = total − idle. If total equals your `corePoolSize` while your `maximumPoolSize`
is far higher, the queue is not full — which under load means it is unbounded.

### Where the memory goes

A `LinkedBlockingQueue` allocates one `Node` per task: an object header plus two
references. On a 64-bit JVM with compressed oops that is roughly 24 bytes per node,
**plus the task object itself and everything it retains**.

That last clause is the one that kills you. A queued `Runnable` is usually a lambda
capturing an `Order`, which retains its `OrderLine`s, which retain `Product`s. A
notification task in `orderflow` can easily retain several kilobytes. At 400 rps with
consumers keeping up at 213/s, the queue grows by ~187 tasks/second. At 3 KB retained
each, that is roughly 560 KB/s — which in a 2 GB container with a 1 GB heap gives you
on the order of half an hour before the heap is gone.

**Nothing in that half hour looks like an error.** That is the drill.

---

## Concurrency trace

**The headline: the unbounded queue that never triggers growth.**

Setup: `orderflow`'s payment-notification pool, configured the way it is in most
codebases:

```java
// The version that ships
ExecutorService notifier = Executors.newFixedThreadPool(16);
// Someone later "increases capacity for the sale":
//   ThreadPoolExecutor tpe = (ThreadPoolExecutor) notifier;
//   tpe.setMaximumPoolSize(64);      // <-- has no effect whatsoever
```

Traffic: the 3-minute k6 spike to 1200 rps. 25% of orders trigger an immediate
notification, so ~300 notification tasks/second arrive. The notifier downstream has a
published SLO of 80 ms p50 / 600 ms p99; call the mean 150 ms. With 16 workers the pool
completes `16 / 0.15 ≈ 107` tasks/second.

Arrival 300/s. Service 107/s. The gap is 193 tasks/second, forever.

| Step | Thread A — Tomcat request thread (submitter) | Thread B — `orderflow-notify-3` (pool worker) | Shared state / outcome |
|---|---|---|---|
| 1 | `notifier.execute(task)` — `workerCount` is 0 | — | step 1 of `execute`: **new thread created**. `poolSize=1` |
| 2 | ... 15 more submissions ... | — | `poolSize=16` — core reached |
| 3 | `notifier.execute(task)` — `workerCount == corePoolSize` | busy in `task.run()` | falls to step 2: `workQueue.offer(task)` |
| 4 | — | — | `LinkedBlockingQueue` capacity is `Integer.MAX_VALUE`. **`offer` returns `true`.** `queueDepth=1` |
| 5 | `execute` returns. **Step 3 is never reached.** | — | `maximumPoolSize` is never consulted. `poolSize` stays 16 |
| 6 | 300 submissions/second continue | 107 completions/second | `queueDepth` grows by ~193/second |
| 7 | after 60 s | still 16 workers | `queueDepth ≈ 11,600`. Wait time for a newly queued task ≈ 11,600 / 107 ≈ **108 seconds** |
| 8 | after 60 s | still 16 workers | **`execute()` still returns in nanoseconds.** No exception. No rejection. `orderflow`'s own p99 is unaffected — the async submission is instant |
| 9 | operator checks dashboards | — | CPU normal, DB fine, error rate 0.00%, `POST /orders` p99 within SLO. **Everything is green.** |
| 10 | after 5 minutes | 16 workers | `queueDepth ≈ 58,000`. Each queued task retains an `Order` graph of ~3 KB → **~170 MB of live heap that is invisible to every dashboard** |
| 11 | — | — | G1 young collections rise; survivors are promoted because the queue is a GC root and the tasks are long-lived. Old gen grows. |
| 12 | after ~25 minutes | 16 workers | `java.lang.OutOfMemoryError: Java heap space` on some **unrelated** thread — whichever one happened to allocate next |
| 13 | — | workers die | Kubernetes restarts the pod. **The queue is in memory. Every one of those 58,000+ notifications is gone.** |
| 14 | — | — | **Business outcome:** ~58,000 customers were charged and never told. Support tickets arrive as "did my payment go through?"; a proportion of those customers pay a second time through a different route. Meanwhile the tasks that *did* run late — steps 7–11 — sent "payment confirmed" emails up to two minutes after the customer had already received the automatic "we could not confirm your payment" retry notice. Contradictory emails, in that order. |

Read step 5 again. **`maximumPoolSize = 64` was configured, deployed, code-reviewed and
believed — and it was never read by anything**, because `offer` never returned `false`.

Read step 9 again. That is the reason this is an ELITE topic. **The failure is
invisible to every standard signal.** Latency of the endpoint is fine. Error rate is
zero. CPU is normal. The only two signals that would have shown it are queue depth and
heap occupancy, and neither is on a default dashboard.

### The second trace: the shutdown that loses work

Same pool, deploy time. `SIGTERM` arrives.

| Step | Thread A — Spring shutdown hook | Thread B — `orderflow-notify-3` | Shared state / outcome |
|---|---|---|---|
| 1 | `notifier.shutdownNow()` | mid-`task.run()`, HTTP POST in flight | `ctl` → `STOP`; queue **drained**, 4,200 tasks returned as a `List<Runnable>` |
| 2 | ignores the return value | — | **the only record of 4,200 pending notifications is discarded** |
| 3 | — | receives `interrupt()`; the HTTP client throws `InterruptedIOException` | the in-flight notification is abandoned mid-request |
| 4 | returns immediately (`shutdownNow` does not wait) | dying | Spring proceeds to close the datasource |
| 5 | JVM exits ~200 ms later | — | **Business outcome:** 4,201 customers charged, never notified. It happens on **every deploy**, which is why nobody correlates it with deploys — it looks like a steady low-level background rate of "missing confirmation" tickets. |

**Now — and only now — here is the correct configuration.**

```java
ThreadPoolExecutor notifier = new ThreadPoolExecutor(
        16, 32,                                     // core, max — Little's Law, below
        60L, TimeUnit.SECONDS,
        new LinkedBlockingQueue<>(400),             // BOUNDED — a latency budget, below
        threadFactory, rejectionHandler);           // named threads, durable fallback

// shutdown, correctly:
notifier.shutdown();                                 // stop accepting; drain the queue
if (!notifier.awaitTermination(20, TimeUnit.SECONDS)) {
    List<Runnable> abandoned = notifier.shutdownNow();
    log.error("abandoned {} notifications at shutdown", abandoned.size());
    persistToOutbox(abandoned);                      // do NOT throw the list away
    notifier.awaitTermination(5, TimeUnit.SECONDS);
}
```

Two bounds and one honest shutdown. Both bounds are derived below rather than guessed.

---

## Example 1 — minimal

The growth rule, made visible. Run this before you read anything else in this section.

```java
import java.util.concurrent.*;

public class GrowthRule {

    static ThreadPoolExecutor pool(BlockingQueue<Runnable> q) {
        return new ThreadPoolExecutor(
                2, 10,                              // core 2, max 10
                60L, TimeUnit.SECONDS, q,
                r -> { Thread t = new Thread(r); t.setName("w-" + t.threadId()); return t; },
                new ThreadPoolExecutor.AbortPolicy());
    }

    public static void main(String[] args) throws Exception {
        boolean unbounded = args.length > 0 && args[0].equals("unbounded");
        ThreadPoolExecutor p = pool(unbounded
                ? new LinkedBlockingQueue<>()        // Integer.MAX_VALUE capacity
                : new LinkedBlockingQueue<>(2));     // capacity 2

        int accepted = 0, rejected = 0;
        for (int i = 0; i < 30; i++) {
            try {
                p.execute(() -> {                    // each task occupies a thread for 3s
                    try { Thread.sleep(3000); } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                    }
                });
                accepted++;
            } catch (RejectedExecutionException e) {
                rejected++;
            }
            System.out.printf("submitted=%2d  poolSize=%2d  active=%2d  queue=%2d  rejected=%d%n",
                    i + 1, p.getPoolSize(), p.getActiveCount(), p.getQueue().size(), rejected);
        }
        System.out.println("accepted=" + accepted + " rejected=" + rejected);
        p.shutdown();
        p.awaitTermination(1, TimeUnit.MINUTES);
    }
}
```

```bash
java GrowthRule.java bounded
java GrowthRule.java unbounded
```

**What to look for:** the `poolSize` column, and the submission number at which it
changes.

| What you see | What it means |
|---|---|
| **bounded:** `poolSize` goes 1, 2, 2, 2 (queue fills to 2), then 3, 4, 5 … up to 10 | **The rule, demonstrated.** The pool grew only *after* the queue reported full. Submissions 3 and 4 went to the queue; submission 5 was the first to trigger growth. |
| **bounded:** `rejected` becomes non-zero once `poolSize=10` and `queue=2` | Correct. Capacity is `max + queueCapacity = 12` concurrent-or-pending tasks. The 13th is refused. **This is backpressure.** |
| **unbounded:** `poolSize` reaches 2 and **stays at 2** for all 30 submissions | `maximumPoolSize = 10` is dead configuration. The queue absorbed 28 tasks. |
| **unbounded:** `rejected = 0`, `queue = 28` | Nothing was refused. The system has accepted work it will take 45 seconds to do, and has told the submitter everything is fine. |
| **unbounded:** it never rejects no matter how many you submit | Correct, and that is the point. There is no number of submissions at which this configuration pushes back. |

Sit with the contrast for a moment. The *only* difference between the two runs is the
integer `2` passed to a queue constructor. That one argument decides whether the pool
has a capacity limit at all.

---

## Example 2 — production scenario (on the project spine)

### The constraints

These are the givens of the scenario — requirements, budgets and published SLOs, not
measurements.

| Constraint | Value |
|---|---|
| Container | `--cpus=4 --memory=2g`, heap `-XX:MaxRAMPercentage=60` (~1.2 GB) |
| Dataset | 100k products, 1M orders, 5M order lines |
| Load | k6: 400 rps sustained on `POST /orders` for 15 min, plus a 3-min spike to 1200 rps |
| Notification fan-out | 25% of orders trigger an immediate notification → **100/s steady, 300/s at spike** |
| Notifier downstream SLO | 80 ms p50, 600 ms p99. Treat the **mean as 150 ms** |
| `orderflow` SLO | `POST /orders` p99 under 400 ms; error budget 0.1% |
| Max acceptable notification delay | **2 seconds** — beyond that the customer's retry email has already gone out |
| Tomcat request threads | 200 |
| HikariCP | `maximumPoolSize=20` |

### Step 1 — size the pool with Little's Law

The work is **I/O-bound**: an HTTP POST to the notifier plus a small JSON
serialisation. The thread is waiting, not computing, for ~95% of the task.

```
L = λ × W
  = 100 tasks/second × 0.150 seconds
  = 15 concurrent workers, at steady state
```

**`corePoolSize = 16.`** Rounded up, with one thread of slack.

For the spike:

```
L = 300 × 0.150 = 45 concurrent workers
```

45 threads is affordable in memory terms — 45 MB of reserved stack — but it is *not*
free: 45 threads on 4 cores means real context-switch pressure, and it competes with
the 200 Tomcat threads and the 20 Hikari connections for the same 4 cores. And the
spike is 3 minutes out of 18.

**`maximumPoolSize = 32.`** A deliberate decision to serve roughly two-thirds of spike
demand from threads and shed the rest. This is a capacity choice, and it should be
written down as one.

**Why not use the CPU-bound formula.** For CPU-bound work you would use
`N_threads = N_cpu × U_cpu × (1 + W/C)` and land near `N_cpu`. Here `W/C` — wait time
over compute time — is roughly 19, which is exactly why Little's Law is the right tool
and why 16 threads on 4 cores is sensible rather than absurd.

**The senior move:** state the assumption you are least sure of. Here it is the mean of
150 ms. If the notifier's real mean is 400 ms, you need 40 threads at steady state, not
16, and the pool will queue permanently. So the mean latency of the downstream call
must be a **monitored metric with an alert**, not a number in a comment. That is the
difference between sizing a pool and guessing at one.

### Step 2 — derive the queue bound from a latency budget

This is the part almost nobody does, and it is what makes the number defensible in
review.

```
Service rate at maximum pool = maximumPoolSize / meanTaskLatency
                             = 32 / 0.150
                             ≈ 213 tasks/second

Maximum acceptable queue wait = 2 seconds        (the product constraint above)

Queue capacity = service rate × acceptable wait
               = 213 × 2
               ≈ 426  →  round DOWN to 400
```

**`queueCapacity = 400.`**

Now say what that number means, because this is the sentence to reuse in an interview:

> "The queue holds 400 tasks. At full pool that is under two seconds of buffered work.
> So a task that is accepted is *guaranteed* to start within the customer's tolerance,
> and a task that would have waited longer is refused instead of accepted-and-delayed.
> **The capacity is the latency budget, expressed as a number.**"

Compare with the unbounded default: a task accepted at step 7 of the trace waits 108
seconds. It was accepted with the same silent success as one that waits 3 ms. An
unbounded queue does not decline work; it lies about when the work will happen.

### Step 3 — write the executor

```java
package com.orderflow.payments;

import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.binder.jvm.ExecutorServiceMetrics;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;

@Configuration
public class NotificationExecutorConfig {

    /** Derived above. Kept as constants so the reasoning is next to the numbers. */
    static final int  CORE  = 16;     // Little's Law: 100/s x 150ms
    static final int  MAX   = 32;     // two-thirds of spike demand; the rest sheds
    static final int  QUEUE = 400;    // 2s latency budget at 213 tasks/s service rate
    static final long KEEPALIVE_SECONDS = 60;

    @Bean(destroyMethod = "")   // we shut it down ourselves — see NotificationLifecycle
    public ThreadPoolExecutor paymentNotificationExecutor(MeterRegistry registry,
                                                          OutboxRepository outbox) {

        ThreadFactory factory = new ThreadFactory() {
            private final AtomicInteger seq = new AtomicInteger();
            @Override public Thread newThread(Runnable r) {
                Thread t = new Thread(r, "orderflow-notify-" + seq.incrementAndGet());
                t.setDaemon(false);                 // deliberate: see shutdown discussion
                t.setUncaughtExceptionHandler((thread, ex) ->
                        LoggerFactory.getLogger("orderflow.notify")
                                     .error("uncaught in {}", thread.getName(), ex));
                return t;
            }
        };

        ThreadPoolExecutor executor = new ThreadPoolExecutor(
                CORE, MAX, KEEPALIVE_SECONDS, TimeUnit.SECONDS,
                new LinkedBlockingQueue<>(QUEUE),
                factory,
                new OutboxFallbackRejectionHandler(outbox, registry)) {

            /**
             * submit() captures a task's exception inside the Future. If nobody calls
             * Future.get(), that exception is never seen by anyone. This override is
             * how you get it back. See Trap 4 — it is the single most valuable
             * fifteen lines in this file.
             */
            @Override
            protected void afterExecute(Runnable r, Throwable t) {
                super.afterExecute(r, t);
                if (t == null && r instanceof Future<?> f && f.isDone()) {
                    try { f.get(); }
                    catch (CancellationException ce) { t = ce; }
                    catch (ExecutionException ee)    { t = ee.getCause(); }
                    catch (InterruptedException ie)  { Thread.currentThread().interrupt(); }
                }
                if (t != null) {
                    LoggerFactory.getLogger("orderflow.notify")
                                 .error("notification task failed", t);
                    registry.counter("orderflow.notification.failed").increment();
                }
            }
        };

        // Bind the standard executor metrics. See Measurement.
        return ExecutorServiceMetrics.monitor(
                registry, executor, "paymentNotification") instanceof ThreadPoolExecutor tpe
                ? tpe : executor;
    }
}
```

> **Note on the last three lines.** `ExecutorServiceMetrics.monitor` returns a wrapped
> `ExecutorService`, not a `ThreadPoolExecutor`, in current Micrometer versions. The
> cleaner shape is to register the metrics as a side effect and return the raw
> executor. I have written it defensively here because the exact return type has varied
> across Micrometer versions — **check yours** with
> `mvn dependency:tree | grep micrometer` and read the method signature in your IDE.
> This is a genuine uncertainty, flagged rather than papered over.

### Step 4 — the rejection handler that does not lose work

```java
package com.orderflow.payments;

import io.micrometer.core.instrument.MeterRegistry;
import java.util.concurrent.*;

/**
 * Rejection is NOT an error. It is the pool correctly declining work it cannot
 * complete inside the latency budget. Losing the work IS an error.
 */
public class OutboxFallbackRejectionHandler implements RejectedExecutionHandler {

    private static final org.slf4j.Logger log =
            org.slf4j.LoggerFactory.getLogger(OutboxFallbackRejectionHandler.class);

    private final OutboxRepository outbox;
    private final io.micrometer.core.instrument.Counter rejected;

    public OutboxFallbackRejectionHandler(OutboxRepository outbox, MeterRegistry registry) {
        this.outbox = outbox;
        this.rejected = registry.counter("orderflow.notification.rejected");
    }

    @Override
    public void rejectedExecution(Runnable r, ThreadPoolExecutor executor) {
        rejected.increment();

        if (executor.isShutdown()) {
            // Shutting down. Persist and return — never run work on the shutdown path.
            if (r instanceof NotificationTask nt) outbox.enqueue(nt.notification());
            return;
        }

        if (r instanceof NotificationTask nt) {
            // Durable fallback: the relay (Topic 115) will pick this up within 30s.
            outbox.enqueue(nt.notification());
            log.warn("notification pool saturated; order {} deferred to outbox "
                            + "(queue={}, active={}, pool={})",
                    nt.orderId(), executor.getQueue().size(),
                    executor.getActiveCount(), executor.getPoolSize());
        } else {
            log.error("rejected an unrecognised task type: {}", r.getClass().getName());
        }
    }
}
```

**Why not `CallerRunsPolicy` here.** It is the tempting answer because it is real
backpressure — and it would be correct if the submitter were a background thread. But
the submitter here is a **Tomcat request thread**. `CallerRunsPolicy` would run a 150 ms
(p99: 600 ms) HTTP call inline on the request thread, adding it directly to
`POST /orders` latency and consuming 1/200th of the service's request capacity for the
duration. Under the spike, 193 request threads per second would each be captured for
150 ms — which is `193 × 0.15 ≈ 29` permanently-occupied request threads, climbing.
That is Topic 55's connection-pool collapse in a different costume.

**The rule:** `CallerRunsPolicy` is excellent when the caller is a load generator, a
batch driver, or another pool you are happy to slow down. It is dangerous when the
caller is a request thread, because it converts your async path into a synchronous one
precisely when the system is already overloaded.

### Step 5 — shut it down honestly

```java
package com.orderflow.payments;

import jakarta.annotation.PreDestroy;
import org.springframework.stereotype.Component;
import java.util.List;
import java.util.concurrent.*;

@Component
public class NotificationLifecycle {

    private static final org.slf4j.Logger log =
            org.slf4j.LoggerFactory.getLogger(NotificationLifecycle.class);

    private final ThreadPoolExecutor executor;
    private final OutboxRepository outbox;

    public NotificationLifecycle(ThreadPoolExecutor executor, OutboxRepository outbox) {
        this.executor = executor;
        this.outbox = outbox;
    }

    @PreDestroy
    public void shutdown() throws InterruptedException {
        // 1. Stop accepting. Already-queued tasks WILL still run.
        executor.shutdown();

        // 2. Give them a bounded amount of time. 20s fits inside the k8s
        //    terminationGracePeriodSeconds of 30, with room for the rest of the
        //    context to close. Never wait unbounded here.
        if (executor.awaitTermination(20, TimeUnit.SECONDS)) {
            log.info("notification executor drained cleanly");
            return;
        }

        // 3. Time is up. Interrupt, and CAPTURE what we abandoned.
        List<Runnable> abandoned = executor.shutdownNow();
        log.error("notification executor did not drain in 20s; abandoning {} tasks", abandoned.size());

        // 4. The abandoned list is the ONLY record of that work. Persist it.
        for (Runnable r : abandoned) {
            if (r instanceof NotificationTask nt) outbox.enqueue(nt.notification());
        }

        // 5. Give interrupted tasks a moment to unwind so we log accurately.
        if (!executor.awaitTermination(5, TimeUnit.SECONDS)) {
            log.error("notification executor still has live threads after shutdownNow");
        }
    }
}
```

**Five steps, and each one exists because of a specific failure:**

| Step | The failure it prevents |
|---|---|
| `shutdown()` before `shutdownNow()` | `shutdownNow()` alone discards the entire queue on every deploy — trace 2, step 1 |
| `awaitTermination` with a **bounded** timeout | An unbounded wait hangs the shutdown until `SIGKILL`, which is worse than a clean abandon |
| 20 s chosen against the k8s grace period | A timeout longer than `terminationGracePeriodSeconds` is a timeout that never fires |
| Capturing the `shutdownNow()` return value | It is the only record of abandoned work. Discarding it is trace 2, step 2 |
| The final 5 s wait | So the "still has live threads" log line is true rather than premature |

**Why the threads are non-daemon.** A daemon thread does not keep the JVM alive, so at
shutdown the JVM can exit with tasks half-done and no record. Non-daemon threads force
you to shut the pool down explicitly — which is exactly the discipline you want. The
cost is that forgetting to shut it down hangs the process, which is Trap 2, and which
is a *loud* failure. Prefer a loud failure to a silent one.

**Java 19+ alternative.** `ExecutorService` implements `AutoCloseable`:

```java
try (ExecutorService pool = Executors.newFixedThreadPool(4)) {
    pool.submit(task);
}   // close() = shutdown() + awaitTermination(indefinitely) + shutdownNow() on interrupt
```

Excellent for a batch job with a lexical scope. **Wrong for a long-lived Spring bean**,
because `close()` waits without a bound, so a stuck task hangs your shutdown until the
container kills you. Use the explicit five-step version for anything that outlives a
method.

### Step 6 — the Spring trap you must check

If you use `@Async` or `ThreadPoolTaskExecutor` instead of a raw
`ThreadPoolExecutor`, you inherit Spring's defaults — and **Spring's default queue
capacity is `Integer.MAX_VALUE`**, which reproduces the exact defect in this document.
Boot's `spring.task.execution.pool.*` properties have historically defaulted to
`core-size: 8`, `max-size: Integer.MAX_VALUE`, `queue-capacity: Integer.MAX_VALUE`.

Do not take my word for the current values on your Boot version — settle it:

```bash
curl -s localhost:8080/actuator/configprops | jq '.contexts[].beans
  | with_entries(select(.key | test("TaskExecutionProperties")))'
```

**What to look for:** the `queueCapacity` value.

| What you see | What it means |
|---|---|
| `"queueCapacity": 2147483647` | Unbounded. **Your `@Async` executor has this document's defect.** Set it explicitly. |
| `"queueCapacity": <a real number>` | Someone configured it. Check the number is derived from a latency budget and not from taste. |
| `"maxSize": 2147483647` with an unbounded queue | Doubly moot: max is unreachable *and* unbounded. |

The fix in `application-load.yml`:

```yaml
spring:
  task:
    execution:
      pool:
        core-size: 16
        max-size: 32
        queue-capacity: 400          # the number you derived, not a round one
        keep-alive: 60s
      shutdown:
        await-termination: true
        await-termination-period: 20s
```

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — `newFixedThreadPool`'s unbounded queue makes `maximumPoolSize` dead configuration

**Wrong:**

```java
ExecutorService notifier = Executors.newFixedThreadPool(16);
// or, equally wrong and more insidious because it looks configured:
new ThreadPoolExecutor(16, 64, 60, SECONDS, new LinkedBlockingQueue<>());
```

**Exact symptom — and the reason this is an ELITE topic is that the symptom is
absence:**

- Throughput **flat**, exactly at `corePoolSize / meanTaskLatency`, and it does not
  move no matter how much load you add.
- Queue depth and task-wait p99 climb **together, linearly**, forever.
- `execute()` / `submit()` latency stays at nanoseconds. The submitting endpoint's own
  p99 is unaffected.
- Error rate **zero**. Rejection count **zero**. CPU normal. Database normal.
- Heap old-gen grows steadily; GC frequency rises; then
  `java.lang.OutOfMemoryError: Java heap space` on an arbitrary unrelated thread.
- A heap dump's dominator tree is topped by `LinkedBlockingQueue` retaining
  `LinkedBlockingQueue$Node` retaining your task lambdas.

**Root cause:** `execute()` reaches step 2, `workQueue.offer` returns `true` because
capacity is `Integer.MAX_VALUE`, and step 3 — the only path to `maximumPoolSize` and
the only path to the rejection handler — is never reached. You disabled both your
growth policy and your backpressure policy with one omitted constructor argument.

**Fix:**

```java
new ThreadPoolExecutor(16, 32, 60, SECONDS,
        new LinkedBlockingQueue<>(400),        // <-- the whole fix
        namedThreadFactory, outboxFallbackHandler);
```

**How to prevent the regression:** ban the factory methods in the build.

```xml
<!-- Checkstyle: fail the build on Executors.newFixedThreadPool / newCachedThreadPool -->
<module name="IllegalMethodCall">
  <property name="methodNames"
            value="newFixedThreadPool,newCachedThreadPool,newSingleThreadExecutor"/>
</module>
```

```bash
mvn -q checkstyle:check
```

**What to look for:** a build failure naming the file and line. If the build passes and
you know the call is there, confirm Checkstyle is bound to a phase that runs:
`mvn help:effective-pom | grep -A5 checkstyle`.

---

### Trap 2 — an executor that is never shut down (thread leak → `OutOfMemoryError: unable to create native thread`)

**Wrong:**

```java
@RestController
public class OrderController {
    @PostMapping("/orders")
    public OrderResponse place(@RequestBody PlaceOrderRequest req) {
        // A NEW executor per request. Never shut down.
        ExecutorService pool = Executors.newFixedThreadPool(4);
        Future<Void> f = pool.submit(() -> { notifier.send(req); return null; });
        return orders.place(req);
        // pool is now unreachable — but its 4 threads are GC ROOTS and live forever.
    }
}
```

**Exact symptom, in this order:**

1. Thread count climbs monotonically and never falls. Visible in
   `jcmd <pid> Thread.print | grep -c '^"'`, in JFR's thread statistics, and in
   `/actuator/metrics/jvm.threads.live`.
2. Native memory (RSS) grows far faster than heap — each thread reserves stack address
   space outside the heap. Your container is OOMKilled by the kernel *or* the JVM
   throws, depending on which limit you hit first.
3. Eventually: `java.lang.OutOfMemoryError: unable to create native thread` — and note
   this is a **completely different message** from heap exhaustion, with a completely
   different cause. Increasing `-Xmx` makes it *worse*, because it leaves less room for
   thread stacks inside the container limit.
4. Or, in a container: no Java exception at all. The kernel OOM-killer terminates the
   process and `kubectl describe pod` shows `Reason: OOMKilled` with exit code 137.

**Root cause, precisely:** a `ThreadPoolExecutor`'s core threads block in
`workQueue.take()` forever — `getTask()` uses the untimed path when
`workerCount <= corePoolSize` and `allowCoreThreadTimeOut` is false. A running thread
is a GC root. Therefore the executor and its queue and every task in it are
**strongly reachable from the thread itself**, and unreachability from your code is
irrelevant. Garbage collection cannot help you. There is no finaliser that saves you.

**Fix — three, in order of preference:**

```java
// 1. BEST: one shared, named, bounded, Spring-managed executor. Inject it.
//    A pool is infrastructure, not a local variable.

// 2. If it genuinely must be per-scope, use try-with-resources (Java 19+):
try (ExecutorService pool = Executors.newFixedThreadPool(4)) {
    pool.submit(task);
}   // close() blocks until done — acceptable ONLY where that is what you want

// 3. Explicit finally, pre-19 style:
ExecutorService pool = new ThreadPoolExecutor(...);
try { pool.submit(task); }
finally { pool.shutdown(); }
```

**How to detect it before production:** assert on thread count in an integration test.

```java
@Test
void doesNotLeakThreadsUnderRepeatedRequests() {
    int before = Thread.activeCount();
    for (int i = 0; i < 500; i++) controller.place(sampleRequest());
    await().atMost(Duration.ofSeconds(10))
           .untilAsserted(() -> assertThat(Thread.activeCount()).isLessThan(before + 20));
}
```

This is Topic 98's drill in full. It is listed here because the *cause* is an executor
lifecycle mistake, and this is the topic that owns executor lifecycles.

---

### Trap 3 — `newCachedThreadPool` creating unbounded threads

**Wrong:**

```java
ExecutorService pool = Executors.newCachedThreadPool();
// = ThreadPoolExecutor(0, Integer.MAX_VALUE, 60L, SECONDS, new SynchronousQueue<>())
```

**Exact symptom:** the mirror image of Trap 1, and it arrives far faster. Under a burst,
thread count spikes to roughly the number of concurrent submissions — in `orderflow`'s
spike, up to 300 threads created within a second or two. Then:

- Context-switch rate explodes (visible in `vmstat 1`'s `cs` column, or
  `pidstat -w -p <pid> 1`). Throughput *falls* as you add load.
- p99 becomes wildly unstable — the classic sawtooth, because 300 threads are
  time-slicing 4 cores.
- If the burst is large enough:
  `java.lang.OutOfMemoryError: unable to create native thread`.
- On recovery, thread count decays over 60 seconds as idle threads time out — so a
  post-hoc thread dump taken two minutes later shows **nothing wrong**. This is why the
  incident is so often mis-diagnosed.

**Root cause:** `SynchronousQueue` has **capacity zero**. `offer` succeeds only if a
worker is at that instant blocked in `take()`. Under a burst there is never an idle
worker, so `offer` always returns `false`, so `execute` always reaches step 3, and step
3 compares against `maximumPoolSize = Integer.MAX_VALUE`. Every single submission
creates a thread. `newCachedThreadPool` is not "cached"; it is "unlimited, with a
60-second decay".

**Where it *is* the right choice:** short-lived, bursty, genuinely-bounded-by-something-
else work — a CLI tool, a test fixture, a fan-out whose width you control. Never a
server endpoint whose concurrency is decided by the internet.

**Fix:** the same bounded `ThreadPoolExecutor`. If you specifically want the eager
thread creation that `SynchronousQueue` gives you, keep the queue and bound the threads:

```java
new ThreadPoolExecutor(0, 32, 60L, TimeUnit.SECONDS,
        new SynchronousQueue<>(), factory, handler);
// Eager growth (a thread per burst item) but a hard ceiling of 32,
// and rejection above that. Occasionally exactly right.
```

---

### Trap 4 — `submit()` silently swallows the task's exception

**Wrong:**

```java
executor.submit(() -> {
    notifier.send(order);          // throws HttpServerErrorException
});
// The Future is discarded. Nobody ever calls get(). Nothing is logged. Anywhere.
```

**Exact symptom:** notifications stop for a subset of orders. **There is no stack trace
in any log.** The worker thread does not die. The pool keeps running. Metrics show tasks
completing normally, because from the pool's point of view the task *did* complete —
the `FutureTask` caught the throwable and stored it as the future's outcome. You find
this from a business number: a support ticket, or a "notifications sent" count that is
lower than the order count.

Coming from Node, this is the difference that will cost you the most. Node prints an
`UnhandledPromiseRejection` warning — and since Node 15 it terminates the process by
default. Java prints nothing and carries on.

**Root cause:** `submit()` wraps your task in a `FutureTask`. `FutureTask.run()`
catches `Throwable` and stores it via `setException`. It is then re-thrown **only** from
`Future.get()`. Since `runWorker`'s `try/catch` never sees a throwable, its
`afterExecute(task, null)` hook is called with a `null` throwable and the default
uncaught-exception handler is never involved.

`execute()` does **not** do this: an exception from an `execute`d `Runnable` propagates
out of `runWorker`, kills the worker (which the pool then replaces), and reaches the
thread's `UncaughtExceptionHandler`.

| | `execute(Runnable)` | `submit(Runnable/Callable)` |
|---|---|---|
| Returns | `void` | `Future<T>` |
| On exception | propagates → worker dies → `UncaughtExceptionHandler` fires → pool replaces the worker | **captured in the `Future`. Silent.** |
| `afterExecute`'s `Throwable` arg | the exception | **`null`** |
| Default visibility | a stack trace on stderr | **nothing** |

**Fix — pick one, and do it once for the whole pool:**

```java
// 1. Override afterExecute to unwrap the Future. Covers BOTH submit and execute.
//    This is the version in Example 2 above and it is the one to use.
@Override protected void afterExecute(Runnable r, Throwable t) {
    super.afterExecute(r, t);
    if (t == null && r instanceof Future<?> f && f.isDone()) {
        try { f.get(); }
        catch (CancellationException ce) { t = ce; }
        catch (ExecutionException ee)    { t = ee.getCause(); }
        catch (InterruptedException ie)  { Thread.currentThread().interrupt(); }
    }
    if (t != null) log.error("task failed", t);
}

// 2. Or make every task self-contained: catch inside the task itself.
executor.execute(() -> {
    try { notifier.send(order); }
    catch (RuntimeException e) { log.error("notify failed for {}", order.id(), e); }
});

// 3. Or use CompletableFuture and attach a handler — but see Topic 91, which has
//    exactly the same silent-swallow problem if you never join.
```

**Note the `f.isDone()` guard.** Without it, `afterExecute` can be reached with a task
that is not a completed `Future`, and `get()` would block. Also note: this trick does
not work for `ScheduledThreadPoolExecutor`'s periodic tasks, which are never "done".

---

### Trap 5 — choosing the rejection policy by accident

**Wrong (three variants, all common):**

```java
// A. Default AbortPolicy, and nobody catches RejectedExecutionException.
executor.execute(task);                    // throws straight into a Tomcat request thread

// B. DiscardPolicy, "so we don't get errors".
new ThreadPoolExecutor(..., new ThreadPoolExecutor.DiscardPolicy());

// C. CallerRunsPolicy on a pool fed by request threads, "because it's backpressure".
new ThreadPoolExecutor(..., new ThreadPoolExecutor.CallerRunsPolicy());
```

**Exact symptoms, one per variant:**

- **A.** `POST /orders` starts returning HTTP 500 with
  `java.util.concurrent.RejectedExecutionException: Task ... rejected from
  java.util.concurrent.ThreadPoolExecutor@... [Running, pool size = 32, active threads
  = 32, queued tasks = 400, completed tasks = <n>]` in the log. The order **was already
  committed** — you have a 500 on a request that succeeded, so the client retries and
  you get a duplicate order. The error budget burns on a path that actually worked.
- **B.** Absolutely nothing. No log, no metric, no exception. Notifications simply do
  not arrive for some customers. You discover it weeks later from support volume. **This
  is the worst outcome in this entire document**, because there is no evidence at all.
- **C.** `POST /orders` p99 jumps from ~200 ms to ~800 ms as soon as the queue fills,
  and Tomcat's busy-thread count climbs. Under sustained overload, request threads are
  consumed running notification tasks, the accept backlog grows, and endpoints that
  touch nothing at all start timing out. You have propagated backpressure to exactly
  the place that could least absorb it.

**Root cause:** the handler runs **on the submitting thread**, synchronously, inside
`execute()`. So the policy is not "what the pool does" — it is "what happens to your
caller". Choosing it without knowing who the caller is means choosing blind.

**Fix — decide from two questions:**

| Question | If yes | If no |
|---|---|---|
| Is the caller a request thread (or anything latency-critical)? | Never `CallerRunsPolicy` | `CallerRunsPolicy` is excellent — real backpressure, no extra machinery |
| Is the work safe to lose? | `DiscardPolicy` is defensible (telemetry, cache warming) | You need a **durable fallback**: outbox, retry topic, or a rejection that the caller handles explicitly |

For `orderflow`'s notifications: caller is a request thread → not CallerRuns; work is a
customer-facing confirmation → not Discard. Therefore a custom handler with an outbox
fallback, which is what Example 2 does.

**And whichever you choose, this is non-negotiable:**

```java
registry.counter("orderflow.notification.rejected").increment();
```

**A rejection count of zero under overload does not mean the system is healthy. It means
you have no bound.** Put that sentence on the dashboard.

---

## Hands-on proof

Everything below is a command you run. I have no JVM and will not print output and call
it real. What follows is exactly what to run, what to look for, and how to read every
plausible result.

### Setup

```bash
mkdir -p ~/java-lab/90 && cd ~/java-lab/90
java --version         # expect 21 or 25
```

### Proof 1 — read the factory methods in the JDK source

```bash
unzip -p "$JAVA_HOME/lib/src.zip" java.base/java/util/concurrent/Executors.java \
  | grep -n -A8 -E 'public static ExecutorService new(Fixed|Cached|SingleThread)'
```

**What to look for:** the queue argument in each `new ThreadPoolExecutor(...)` call.

| What you see | What it means |
|---|---|
| `newFixedThreadPool`: `new LinkedBlockingQueue<Runnable>()` — **no capacity argument** | Unbounded. Confirmed from primary source, on your JDK. |
| `newCachedThreadPool`: `Integer.MAX_VALUE` as the second argument and `new SynchronousQueue<Runnable>()` | Unbounded **threads**. Confirmed. |
| `newSingleThreadExecutor`: wrapped in a delegating class | The wrapper is why you cannot cast it back to `ThreadPoolExecutor` and reconfigure it. Deliberate. |
| `unzip: cannot find` | Your JDK ships without sources. Read the same code at the OpenJDK repository for your exact version tag. |

This is a two-minute exercise and it converts "I read that fixed pools have unbounded
queues" into "I have seen the constructor argument". Do it.

### Proof 2 — the growth rule, instrumented

Run `GrowthRule.java` from Example 1 in both modes. Then extend it: add a
`ScheduledExecutorService` that prints the pool's four numbers every 200 ms while a
background thread submits at a fixed rate.

```java
ScheduledExecutorService probe = Executors.newSingleThreadScheduledExecutor(
        r -> { Thread t = new Thread(r, "probe"); t.setDaemon(true); return t; });

probe.scheduleAtFixedRate(() -> System.out.printf(
        "t=%5d pool=%2d active=%2d queue=%5d completed=%d%n",
        System.currentTimeMillis() % 100000,
        p.getPoolSize(), p.getActiveCount(), p.getQueue().size(), p.getCompletedTaskCount()),
        0, 200, TimeUnit.MILLISECONDS);
```

**What to look for:** the relationship between `queue` and `pool` over time.

| What you see | What it means |
|---|---|
| `queue` reaches capacity **before** `pool` starts rising | **The rule.** Queue-full is the trigger for growth, not a buffer in front of it. |
| `pool` pinned at core while `queue` rises without limit | Unbounded queue. `maximumPoolSize` is dead configuration. |
| `completed` rising at a constant rate while `queue` rises | **Throughput is flat and latency is climbing** — the exact drill signature, in four numbers. |
| `active` < `pool` with a non-empty queue | Impossible in a correct pool. If you see it, your tasks are finishing between the two `get` calls — these getters are not an atomic snapshot. |

> **Note:** `getPoolSize()`, `getActiveCount()` and `getQueue().size()` are read
> independently and are **not** a consistent snapshot. For teaching, fine. For a metric,
> use `ExecutorServiceMetrics` (Measurement section), which has the same limitation but
> at least applies it consistently.

### Proof 3 — count pool threads and separate idle from busy

```bash
PID=$(jcmd -l | grep -i orderflow | cut -d' ' -f1)

# total threads in the pool
jcmd $PID Thread.print | grep -c '"orderflow-notify-'

# idle ones (parked in getTask)
jcmd $PID Thread.print | grep -A10 '"orderflow-notify-' | grep -c 'ThreadPoolExecutor.getTask'

# anything unnamed — a pool somebody forgot to give a ThreadFactory
jcmd $PID Thread.print | grep -cE '"pool-[0-9]+-thread-[0-9]+"'
```

| What you see | What it means |
|---|---|
| total == `corePoolSize` while under heavy load | The queue is not full. Either load is below capacity, or the queue is unbounded. |
| total == `maximumPoolSize` and idle == 0 | Saturated. The next rejection is imminent. Correct behaviour if you bounded it. |
| idle == total under load | Workers are waiting for work that is not arriving. Look upstream, not at the pool. |
| The `pool-N-thread-M` count is non-zero | **Someone created an executor without a `ThreadFactory`.** You cannot tell which pool those threads belong to. Find it and name it — this is a five-minute fix that pays for itself in the next incident. |

### Proof 4 — prove the queue is what is retaining heap

```bash
jcmd $PID GC.heap_info
jcmd $PID GC.class_histogram | head -25
```

**What to look for** in the histogram: the instance count for
`java.util.concurrent.LinkedBlockingQueue$Node` and for your task class.

| What you see | What it means |
|---|---|
| `LinkedBlockingQueue$Node` in the top few rows with a count in the tens of thousands | Your queue depth, confirmed independently of any metric. |
| Node count roughly equal to your task-lambda count | Confirms each queued task is retaining its captured state. Multiply by the retained size to get the real cost. |
| Node count near zero while old gen is full | The leak is elsewhere. Go to a real heap dump (Topic 79). |

Then the definitive version:

```bash
jcmd $PID GC.heap_dump /tmp/orderflow-90.hprof
# open in Eclipse MAT -> Dominator Tree, or:
jhat /tmp/orderflow-90.hprof   # deprecated but present on some JDKs
```

**What to look for** in MAT's dominator tree: whether `LinkedBlockingQueue` (or your
`ThreadPoolExecutor`) is the top dominator by retained heap. **Retained**, not shallow —
that is Topic 79's distinction and it is the whole point.

### Proof 5 — the `submit()` swallow, demonstrated

`Swallow.java`:

```java
import java.util.concurrent.*;

public class Swallow {
    public static void main(String[] args) throws Exception {
        ExecutorService p = Executors.newFixedThreadPool(1, r -> {
            Thread t = new Thread(r, "worker");
            t.setUncaughtExceptionHandler((th, e) ->
                    System.out.println("UNCAUGHT HANDLER FIRED: " + e));
            return t;
        });

        System.out.println("--- via submit ---");
        p.submit(() -> { throw new IllegalStateException("notify failed"); });
        Thread.sleep(300);

        System.out.println("--- via execute ---");
        p.execute(() -> { throw new IllegalStateException("notify failed"); });
        Thread.sleep(300);

        p.shutdown();
    }
}
```

```bash
java Swallow.java
```

**What to look for:** which of the two sections prints anything at all.

| What you see | What it means |
|---|---|
| `--- via submit ---` followed by **nothing** | **The trap, demonstrated.** `FutureTask` captured the exception. No handler fired. No stack trace. In production this is a notification that silently never happened. |
| `--- via execute ---` followed by `UNCAUGHT HANDLER FIRED: java.lang.IllegalStateException: notify failed` | `execute` lets it propagate. The worker died and the pool replaced it. |
| Both sections silent | Your `ThreadFactory` is not being used — check you passed it to the factory method and that you are not on a wrapped executor. |
| Both sections print | You are on an executor that overrides `afterExecute` (or a framework wrapper that does). Good — but confirm it, do not assume it. |

Run this once. It is thirty seconds and it will change how you write async code in Java.

---

## Failure drill

**Mandatory. Produce the failure yourself. Write down what you saw before reading the
fix.** The point is the memory of a dashboard that was entirely green while the service
was dying.

### The scenario

`orderflow` under the Topic 65 load profile with the payment-notification pool
configured as `Executors.newFixedThreadPool(16)`. You will drive it into unbounded queue
growth, watch queue depth and task-wait p99 climb together while throughput stays
exactly flat, and take it to `OutOfMemoryError`. Then you will bound the queue and watch
latency stabilise **and errors appear** — and you will argue that the errors are the
correct behaviour.

### Setup

**1. Constrain the heap so the drill finishes in minutes rather than hours.**

```yaml
# docker-compose.load.yml
services:
  orderflow:
    deploy:
      resources:
        limits: { cpus: "4", memory: 2g }
    environment:
      JAVA_TOOL_OPTIONS: >-
        -Xmx512m
        -XX:+HeapDumpOnOutOfMemoryError
        -XX:HeapDumpPath=/dumps/orderflow-oom.hprof
        -Xlog:gc*:file=/dumps/gc.log:time,uptime,level,tags
        -XX:StartFlightRecording=duration=30m,filename=/dumps/drill90.jfr,settings=profile
```

`-Xmx512m` is deliberately small. You are compressing a 25-minute failure into a
5-minute one. Say so in your write-up — it is a legitimate technique and pretending the
heap was production-sized would be dishonest.

**2. Make the notifier slow and the tasks fat**, so the queue grows and each node
retains real memory.

```java
@Service
public class PaymentNotifier {

    // Simulates the downstream notifier's 150ms mean.
    public void send(Notification n) {
        try { Thread.sleep(150); } catch (InterruptedException e) {
            Thread.currentThread().interrupt(); return;
        }
        // ...real HTTP POST would go here
    }
}

record NotificationTask(Order order, PaymentNotifier notifier) implements Runnable {
    // Capturing the whole Order (with its OrderLines and Products) is DELIBERATE.
    // It is also what most real code does.
    @Override public void run() { notifier.send(Notification.from(order)); }
}
```

**3. The broken executor:**

```java
@Bean
public ExecutorService paymentNotificationExecutor() {
    return Executors.newFixedThreadPool(16);      // DELIBERATELY BROKEN
}
```

**4. Expose the four numbers**, or you will have nothing to look at:

```java
@Bean
ApplicationRunner queueProbe(ExecutorService ex, MeterRegistry reg) {
    ThreadPoolExecutor tpe = (ThreadPoolExecutor) ex;
    reg.gauge("orderflow.notify.queue.depth", tpe, t -> t.getQueue().size());
    reg.gauge("orderflow.notify.pool.size",   tpe, ThreadPoolExecutor::getPoolSize);
    reg.gauge("orderflow.notify.active",      tpe, ThreadPoolExecutor::getActiveCount);
    reg.gauge("orderflow.notify.completed",   tpe, t -> (double) t.getCompletedTaskCount());
    return args -> {};
}
```

### Commands

```bash
# 1. Start the service
docker compose -f docker-compose.load.yml up -d

# 2. Start the load: 400 rps for 15 minutes
k6 run --vus 200 --duration 15m load/place-order.js

# 3. Sample the pool every 5 seconds into a CSV you can plot
PID=$(docker exec orderflow jcmd -l | grep -i orderflow | cut -d' ' -f1)
while true; do
  echo -n "$(date +%s),"
  curl -s localhost:8080/actuator/metrics/orderflow.notify.queue.depth | jq -r '.measurements[0].value' | tr -d '\n'
  echo -n ","
  curl -s localhost:8080/actuator/metrics/orderflow.notify.completed | jq -r '.measurements[0].value'
  sleep 5
done | tee /tmp/queue-depth.csv

# 4. Every 2 minutes, snapshot the pool from the JVM's side
docker exec orderflow jcmd $PID Thread.print | grep -c 'orderflow-notify\|pool-.*-thread'
docker exec orderflow jcmd $PID GC.class_histogram | grep -E 'LinkedBlockingQueue|NotificationTask'
docker exec orderflow jcmd $PID GC.heap_info

# 5. When it dies
docker logs orderflow 2>&1 | grep -i 'OutOfMemoryError'
grep -c 'Pause Full' /dumps/gc.log
```

### What to capture

Write these eight things down before reading further:

1. `poolSize` at t=1min, t=5min, t=10min. (Prediction: it never changes.)
2. `queue.depth` at the same three points.
3. The **derivative** of `completed` — tasks/second — at the same three points.
4. Task-wait p99, computed as `queue.depth / (completed-rate)`.
5. Old-gen occupancy from `GC.heap_info` at the same three points.
6. The count of `Pause Full` entries in `gc.log` over the last 3 minutes before death.
7. The exact `OutOfMemoryError` message and which thread threw it.
8. The top three entries by retained size in the heap dump's dominator tree.

### How to read it

| What you see | What it means |
|---|---|
| `poolSize` is 16 at every sample, from t=0 to death | **The mechanical statement, confirmed under load.** The queue never filled, so the pool never grew. |
| `queue.depth` climbing at a constant ~187/second | Arrival minus service. The slope *is* `λ − μ`. |
| The `completed` derivative flat at roughly 107/second throughout | **Throughput is capped and does not respond to load.** Adding VUs changes nothing except the slope of the queue. |
| Computed task-wait p99 climbing linearly, crossing 2 s within ~15 seconds of the spike | Your latency budget was blown 25 minutes before anything else noticed. |
| `POST /orders` p99 **unchanged and within SLO** the whole time | The submission is asynchronous and instant. **This is why nobody notices.** Say this out loud. |
| HTTP error rate exactly 0.00% | Nothing was rejected. **A zero rejection rate under overload is a symptom, not a success.** |
| Old gen rising monotonically; `Pause Full` entries appearing near the end | The queue is a GC root; queued tasks survive every young collection and get promoted. |
| `GC.class_histogram` showing `LinkedBlockingQueue$Node` counts matching your queue-depth gauge | Two independent sources agreeing. This is what "confirmed" means. |
| `java.lang.OutOfMemoryError: Java heap space`, thrown on an **unrelated** thread (a Tomcat worker, a Hikari housekeeper) | Expected and important. The OOM lands on whichever thread allocates next, so **the stack trace does not point at the bug.** |
| Dominator tree topped by `ThreadPoolExecutor` → `LinkedBlockingQueue` → `Node[]` → your tasks | The proof. Retained heap, not shallow heap. |
| It does not OOM within 15 minutes | Lower `-Xmx` to 256m, raise the sleep in `PaymentNotifier.send` to 300 ms, or make the captured `Order` graph fatter. You are compressing a real timescale; adjust until it fits your patience. |

### Now fix it

Change exactly one bean:

```java
@Bean
public ThreadPoolExecutor paymentNotificationExecutor(MeterRegistry registry,
                                                      OutboxRepository outbox) {
    return new ThreadPoolExecutor(
            16, 32, 60L, TimeUnit.SECONDS,
            new LinkedBlockingQueue<>(400),                       // BOUNDED
            namedFactory("orderflow-notify-"),
            new OutboxFallbackRejectionHandler(outbox, registry));  // and a real policy
}
```

Re-run the identical load and capture the same eight things.

| What you see after the fix | What it means |
|---|---|
| `queue.depth` rises to 400 and **stops** | The bound is doing its job. Queue wait is now bounded at ~2 s by construction. |
| `poolSize` climbs from 16 to 32 once depth hits 400 | **Step 3 of `execute` is now reachable.** `maximumPoolSize` is live configuration again. |
| `completed` rate rises from ~107/s to ~213/s | You gained real throughput — because the pool was finally allowed to grow. |
| `orderflow.notification.rejected` becomes **non-zero** during the spike | **This is the fix working, not the fix failing.** The system is declining work it cannot do inside the budget, and recording that it did so. |
| Old-gen occupancy flat | The queue's memory is now bounded by construction: `400 × retained-size-per-task`. You can compute the worst case in advance, which you could not before. |
| No OOM for the full 15 minutes | The failure mode has changed from "dies silently after 25 minutes" to "sheds load visibly during a 3-minute spike". |
| The outbox table gains rows during the spike, and drains after | **No work was lost.** Rejection shed *latency*, not *work*. This is the distinction to defend in review. |

### What the fix proves

Four things. Say them in this order:

1. **The queue bound is the growth trigger.** `maximumPoolSize` went from dead
   configuration to live configuration by changing one constructor argument.
2. **The queue bound is the latency budget.** 400 at 213/s is under 2 seconds, by
   arithmetic, not by hope.
3. **Errors under overload are correct.** A zero rejection rate under overload means no
   bound exists. The dashboard panel to build is `rejected`, and its correct value under
   a spike is *not zero*.
4. **Shedding latency is not the same as losing work.** The rejection handler is where
   that distinction lives, and it is the part everyone skips.

---

## Measurement

### The standing rule

```java
// WRONG. Every number this produces is untrustworthy.
long start = System.nanoTime();
for (int i = 0; i < 100_000; i++) executor.execute(NOOP);
System.out.println((System.nanoTime() - start) / 100_000 + " ns/submit");
```

Four independent reasons and you cannot tell which is lying: dead-code elimination (a
no-op task is provably useless and C2 may delete the body), on-stack replacement (the
loop is compiled mid-flight, so the average blends interpreted, C1 and C2 execution),
cold JIT for the first thousands of iterations, and — fatal here — the fact that you are
measuring *submission* while the interesting cost is *scheduling and contention on the
queue*, which depends entirely on how many threads are submitting. This is Topic 77's
subject in full.

### JMH with `@Threads` — the correct harness

Pool-related benchmarks are meaningless single-threaded. The variable that matters is
submitter count.

```java
package com.orderflow.bench;

import org.openjdk.jmh.annotations.*;
import java.util.concurrent.*;

@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.SECONDS)
@Warmup(iterations = 5, time = 2)
@Measurement(iterations = 10, time = 2)
@Fork(3)
@State(Scope.Benchmark)
public class QueueChoiceBenchmark {

    @Param({"linked-bounded", "array-bounded", "synchronous"})
    public String queueType;

    private ThreadPoolExecutor pool;

    @Setup(Level.Trial)
    public void setUp() {
        BlockingQueue<Runnable> q = switch (queueType) {
            case "linked-bounded" -> new LinkedBlockingQueue<>(400);
            case "array-bounded"  -> new ArrayBlockingQueue<>(400);
            case "synchronous"    -> new SynchronousQueue<>();
            default -> throw new IllegalArgumentException(queueType);
        };
        pool = new ThreadPoolExecutor(16, 32, 60, TimeUnit.SECONDS, q,
                Executors.defaultThreadFactory(),
                new ThreadPoolExecutor.CallerRunsPolicy());   // never lose a task mid-benchmark
        pool.prestartAllCoreThreads();                        // remove warmup skew
    }

    @TearDown(Level.Trial)
    public void tearDown() throws InterruptedException {
        pool.shutdown();
        pool.awaitTermination(30, TimeUnit.SECONDS);
    }

    @Benchmark
    @Threads(16)                     // 16 concurrent SUBMITTERS — the variable that matters
    public void submitTrivialTask() {
        pool.execute(() -> Blackhole.consumeCPU(200));   // ~fixed tiny CPU cost per task
    }
}
```

| Annotation | What it defends against |
|---|---|
| `@Fork(3)` | Three separate JVMs. Defeats profile pollution; exposes run-to-run variance. A single fork can be silently wrong. |
| `@Threads(16)` | The whole question is contention on the queue's put lock. One submitter measures nothing relevant. |
| `@Param` over queue types | This is a *comparison*; a single configuration answers no question. |
| `prestartAllCoreThreads()` | Otherwise the first iterations include thread creation, which is orders of magnitude more expensive than a submit. |
| `Blackhole.consumeCPU(200)` | A fixed, non-eliminable amount of work. A truly empty task lets the JIT delete the body and you measure nothing. |
| `CallerRunsPolicy` in the harness only | Guarantees no task is dropped, so the throughput number counts every task. **Not** the policy you would ship. |

```bash
./mvnw -Pbench clean verify
java -jar target/benchmarks.jar QueueChoiceBenchmark -rf json -rff queue-bench.json
```

**What to look for:** the throughput and **the `±` error margin** for each `queueType`.
If the margins overlap, you have measured nothing and the choice should be made on
semantics (bounded vs unbounded, capacity choice) rather than speed. Report the interval,
never the point estimate.

**The honest interpretation, before you run it:** you are comparing queue lock
contention — nanoseconds to low microseconds — against a 150 ms downstream HTTP call.
Even a 5× difference between queue implementations is under one ten-thousandth of the
task's cost. **Choose the queue for its bound, not for its throughput.** If your
measurement suggests otherwise, suspect the benchmark before you suspect the queue.

### Micrometer — the four numbers, and one binder that gives you them

```java
// Micrometer's standard binder. Do this rather than hand-rolling gauges.
ExecutorServiceMetrics.monitor(registry, executor, "paymentNotification",
        Tags.of("service", "orderflow"));
```

That registers a standard set. The four that matter:

| Metric | What it tells you | Alert on it? |
|---|---|---|
| `executor.queued` (gauge) | **Queue depth.** The leading indicator: it moves minutes before latency does. | Yes — but as a fraction of capacity, see below |
| `executor.active` (gauge) | Busy workers. `active == poolSize == max` means saturated. | As a saturation ratio |
| `executor.pool.size` (gauge) | Current thread count. **Pinned at core under load ⇒ the queue is not filling ⇒ suspect unbounded.** | No — use it for diagnosis |
| `executor.rejected` (counter) | Backpressure applied. **Should be non-zero under overload.** | Yes, on a *sustained* rate |
| `executor.seconds` (timer) | Task execution time — how long the task ran once started | Yes, p99 against the downstream SLO |

**The one it does not give you, and which you must add:** *queue wait time* — how long a
task sat before starting. Task latency as the customer experiences it is
`queueWait + executionTime`, and `executor.seconds` only measures the second half.

```java
public void submitNotification(Order order) {
    long enqueuedNanos = System.nanoTime();
    executor.execute(() -> {
        registry.timer("orderflow.notification.queue.wait")
                .record(System.nanoTime() - enqueuedNanos, TimeUnit.NANOSECONDS);
        notifier.send(Notification.from(order));
    });
}
```

**Two rules for these series:**

1. **Alert on saturation, not on depth.** `executor.queued / capacity > 0.8` survives a
   capacity change; `executor.queued > 320` does not. Derive the threshold from the
   latency budget: 80% of 400 at 213/s is 1.5 s of buffered work, which is inside your
   2-second constraint with margin.
2. **Never tag any of these with an order ID, SKU or customer ID.** 100k products means
   100k time series and a dead Prometheus. That is Topic 118's cardinality drill.

*Illustration of the exposition FORMAT, not captured output. `<n>` are placeholders.*

```
# HELP executor_queued_tasks The approximate number of tasks that are queued for execution
# TYPE executor_queued_tasks gauge
executor_queued_tasks{name="paymentNotification",service="orderflow",} <n>.0
# TYPE executor_pool_size_threads gauge
executor_pool_size_threads{name="paymentNotification",service="orderflow",} <n>.0
# TYPE executor_rejected_tasks_total counter
executor_rejected_tasks_total{name="paymentNotification",service="orderflow",} <n>.0
```

Note the word **approximate** in Micrometer's own help text for `queued`. `getQueue().size()`
on a `LinkedBlockingQueue` is an atomic read of a counter, so it is exact for that class —
but the binder is written against `BlockingQueue` in general, where `size()` may be a
traversal. Do not build a correctness argument on it.

### `jcmd` and JFR

```bash
PID=$(jcmd -l | grep -i orderflow | cut -d' ' -f1)

jcmd $PID Thread.print | grep -c '"orderflow-notify-'                        # pool size
jcmd $PID Thread.print | grep -A10 '"orderflow-notify-' | grep -c 'getTask'  # idle
jcmd $PID GC.class_histogram | grep -E 'LinkedBlockingQueue\$Node'           # queue depth, independently
jcmd $PID VM.native_memory summary | grep -A3 Thread                         # stack memory (needs -XX:NativeMemoryTracking=summary)
```

```bash
java -XX:StartFlightRecording=duration=300s,filename=pool.jfr,settings=profile -jar orderflow.jar

jfr summary pool.jfr
jfr print --events jdk.ThreadStart,jdk.ThreadEnd pool.jfr | head -40
jfr print --events jdk.JavaThreadStatistics pool.jfr | tail -20
jfr print --events jdk.ThreadPark pool.jfr | head -40
```

| Event | What to read from it |
|---|---|
| `jdk.JavaThreadStatistics` | `activeCount` and `peakCount` over time. **A monotonically rising `activeCount` is the thread-leak signature from Trap 2.** |
| `jdk.ThreadStart` / `jdk.ThreadEnd` | Thread churn. A high start rate with a matching end rate means a `newCachedThreadPool`-style pool thrashing. A high start rate with no ends is a leak. |
| `jdk.ThreadPark` with a `LinkedBlockingQueue.take` stack | Idle workers. Volume here is normal for a quiet pool. |
| `jdk.ObjectAllocationSample` filtered to your task class | Where the queued objects come from, if the class histogram is ambiguous. |

---

## Practice exercises

### 1 — easy

Take `GrowthRule.java` from Example 1 and turn it into a table you can defend.

For each of the following six configurations, **predict** the final `poolSize`, final
`queue` depth, and `rejected` count for 30 submissions of a 3-second task — write the
prediction down first — then run it and record the actual.

| # | core | max | queue |
|---|---|---|---|
| 1 | 2 | 10 | `LinkedBlockingQueue<>()` |
| 2 | 2 | 10 | `LinkedBlockingQueue<>(2)` |
| 3 | 2 | 10 | `LinkedBlockingQueue<>(100)` |
| 4 | 2 | 10 | `SynchronousQueue<>()` |
| 5 | 0 | 10 | `SynchronousQueue<>()` |
| 6 | 10 | 10 | `LinkedBlockingQueue<>(2)` |

Then answer in one sentence each:

- Which row has the highest throughput, and why is it not the one with the biggest queue?
- Which row would you ship for a payment notification, and what is the one number you
  would need from product to choose the queue capacity?
- Row 6 has core == max. What does `keepAliveTime` do there, and what would
  `allowCoreThreadTimeOut(true)` change?

### 2 — medium (combines Topics 01–89)

Build a `ProductImportExecutor` for `orderflow`: a bulk CSV import of 100,000 products,
parsed and upserted in batches of 500.

Requirements, each tied to a topic you already have:

- **Topic 90:** an explicit `ThreadPoolExecutor` with a bounded queue. Derive both the
  pool size and the queue capacity in a comment, showing the arithmetic. The importer
  is **CPU-bound during parse** and **I/O-bound during upsert** — decide whether that is
  one pool or two, and defend it.
- **Topic 53:** batched inserts with `SEQUENCE` + pooled optimiser, not `IDENTITY`. State
  in a comment why `IDENTITY` would make the batch size irrelevant.
- **Topic 55:** the transaction boundary is **per batch**, not per import. State what
  would happen to HikariCP's 20 connections if you used one transaction for the whole
  import and 16 threads.
- **Topic 79:** the import sets an MDC correlation ID. Show the `try/finally
  MDC.clear()` on the *pooled* thread, and explain in one sentence why forgetting it
  leaks the previous import's ID into the next one's logs — permanently, because pooled
  threads are reused.
- **Topic 89:** the main thread waits for completion with `awaitTermination`, not with a
  sleep-and-poll loop. State which guarantee `awaitTermination` gives you that polling
  `getActiveCount() == 0` does not.
- **Topic 01:** quantities are `int`, prices are `long` minor units. No boxing in the
  per-row hot path — say how you would confirm that with an allocation profile.
- **Topic 13:** the dedupe set is keyed by a `Sku` record. One sentence on why.

Then break it three ways and record the exact symptom of each:

1. Replace the bounded queue with `Executors.newFixedThreadPool(8)` and import 1,000,000
   rows instead of 100,000, with `-Xmx256m`.
2. Change `execute` to `submit` and make one batch throw. Report what you see in the
   logs. (Prediction: nothing.)
3. Remove the `MDC.clear()` and run two imports back to back. Report what appears in the
   second import's log lines.

### 3 — hard (production simulation on `orderflow` under load)

Run the full failure drill and produce a document of the kind you would attach to an
incident ticket and a capacity review.

**Part A — baseline.** With the correctly-bounded pool, run the Topic 65 load profile.
Record: `POST /orders` p50/p95/p99, notification queue depth over time, `executor.active`,
`executor.rejected`, and the notification end-to-end latency (submit → sent). This is
your control.

**Part B — break it.** Swap to `Executors.newFixedThreadPool(16)` with `-Xmx512m`.
Run the identical profile. Capture everything from the Failure drill's "What to capture"
list, plus the heap dump.

**Part C — the capacity model.** From your baseline numbers, compute:

1. The mean notification latency `W` your service actually observes (not my assumed
   150 ms).
2. The required `corePoolSize` from Little's Law at 100 tasks/s and at 300 tasks/s.
3. The queue capacity implied by a 2-second budget at your measured service rate.
4. The **worst-case heap** of a full queue: `capacity × retained size per task`. Get the
   retained size from MAT, not from arithmetic — the difference between shallow and
   retained is the point.
5. The point at which the honest answer stops being "a bigger pool" and becomes
   "virtual threads" (Topic 101) or "push it to Kafka" (Topic 115). State the specific
   number that flips it.

**Part D — the rejection policy argument.** Implement all four standard policies plus
your outbox-fallback handler. Run the spike against each. For each, report: `POST /orders`
p99, HTTP error rate, notifications lost, and notifications delayed beyond 2 seconds.
Then write two paragraphs recommending one, **including what you are giving up**.

**Part E — the shutdown drill.** With load running at 400 rps, `docker compose stop`
the service (which sends `SIGTERM` with a 10-second default timeout). Count the
notifications that were queued, sent, and lost. Then implement the five-step shutdown
and repeat. Report the difference and state whether the k8s
`terminationGracePeriodSeconds` needs to change.

**Part F — the honest conclusion.** Answer: **could the drill have been detected by
alerting before the OOM?** Name the single metric, the threshold, and the lead time it
would have given you. If the answer is "only queue depth would have caught it", say what
that implies about every executor in your service that has no queue-depth metric — and
then go and count how many that is.

---

## Interview questions

### Q1 — "How do you size a thread pool?"

This is the staple. The senior signal is that you **ask a question back before
answering**.

**Mid-level answer:** "Usually number of cores times two. Or 200 — that's what we use."

**Senior answer:** "First: is the work CPU-bound or I/O-bound? They have different
formulas and getting it backwards is a factor-of-ten error.

For **I/O-bound** work I use Little's Law, which I already reason with for queues:
`threads ≈ throughput × latency`. For `orderflow`'s payment notifications at 100 tasks
per second with a 150 ms mean, that is 15 concurrent workers, so I'd set core to 16.
For the 3× spike it computes to 45, and I'd deliberately set max to 32 and shed the
rest, because 45 threads on a 4-core container competing with 200 Tomcat threads is
real scheduling pressure and the spike is 3 minutes out of 18.

For **CPU-bound** work Little's Law does not apply, because the thread is not waiting —
it is computing, and there is no additional parallelism to buy. The size is the core
count, maybe cores plus one to cover a page fault. More threads makes it slower via
context switches and cache pollution.

Then the two things people skip. One: I bound the queue, and I derive the capacity from
a latency budget rather than picking a round number. At a max pool of 32 and a 150 ms
task, the service rate is about 213 per second; a 2-second budget gives a capacity of
about 400. Two: I choose the rejection policy deliberately, and I make sure rejection is
counted, because **a rejection rate of zero under overload means there is no bound**.

Finally, I'd say which assumption I am least sure of. Here it is the 150 ms mean. If the
downstream's real mean is 400 ms, my pool is a third of the size it needs to be and it
will queue permanently. So that latency is a monitored metric with an alert, not a
number in a comment."

**What separates them:** asking CPU-vs-I/O first; using Little's Law by name and
correctly; deriving the *queue* bound as well as the thread count; naming the rejection
policy as a decision; and identifying the weakest assumption and monitoring it.

**Follow-up:** "Your pool is 16 and max is 64 and it never goes above 16 under heavy
load. Why?" — the mechanical statement. Unbounded queue.

---

### Q2 — "What happens under overload?"

**Mid-level answer:** "It gets slow and eventually you get errors."

**Senior answer:** "That depends entirely on whether the queue is bounded, and the two
outcomes are qualitatively different.

**Unbounded queue:** throughput goes flat at `corePoolSize / taskLatency` and stays
there. `maximumPoolSize` is never reached because the growth check only fires when
`offer` returns false. Queue depth and task-wait latency climb together, linearly,
without limit. Crucially the *submitting* endpoint's latency does not move — submission
is still nanoseconds — so the p99 you page on looks fine. Error rate stays at zero
because nothing is ever rejected. Memory grows, because each queued task retains
whatever it captured. Then you get an `OutOfMemoryError` on some unrelated thread,
twenty-odd minutes later, and the stack trace does not point at the queue. **Every
standard dashboard is green throughout.** That is what makes it dangerous.

**Bounded queue:** the queue fills to capacity, the pool grows to max, and then the
rejection handler starts firing. Latency is now bounded by construction — a task is
either started within `capacity / serviceRate` or it is refused. You get errors, and the
errors are correct: the system is declining work it cannot do inside the budget. The
question then becomes what the handler does — and for anything customer-facing it must
be durable, an outbox or a retry topic, so that you shed *latency* rather than *work*.

The design principle: **you always choose. The only question is whether you choose
deliberately or by omitting a constructor argument.**"

**What separates them:** treating "what happens" as branching on the queue bound;
naming that the submitting endpoint's latency does not move; and stating that errors
under overload are the correct behaviour.

**Follow-up:** "What would you alert on?" — queue saturation as a fraction of capacity,
and a sustained non-zero rejection rate. Not queue depth in absolute terms.

---

### Q3 — "Is `Executors.newFixedThreadPool(200)` fine?"

**Mid-level answer:** "Sure, 200 threads should handle a lot of load."

**Senior answer:** "Two problems, and the second is the one that ends the service.

First, where did 200 come from? If the work is I/O-bound at 50 ms and you want 1000
requests per second, Little's Law says 50 in flight, not 200. If it is CPU-bound on a
4-core container, 200 threads makes it slower — you have added context switches and
cache pollution and bought no parallelism. Either way 200 is a number somebody typed.

Second and worse: `newFixedThreadPool` uses `new LinkedBlockingQueue<>()` with no
capacity, so the queue is `Integer.MAX_VALUE`. That means there is no backpressure at
all. Under overload it accepts every task, throughput stays flat at 200 divided by task
latency, wait time grows without bound, and the queued tasks retain heap until you OOM.
Also, since core equals max, `maximumPoolSize` was never going to do anything anyway.

I'd replace it with an explicit `ThreadPoolExecutor`: a size from Little's Law, a
bounded queue whose capacity comes from a latency budget, a named `ThreadFactory` so
thread dumps are readable, and a rejection handler that writes to the outbox. And I'd
add a Checkstyle rule banning the `Executors.*` factories so it does not come back."

**What separates them:** attacking the *queue* rather than only the number, and knowing
that core == max makes max moot regardless.

**Follow-up:** "So use `newCachedThreadPool`?" — no; that is unbounded in the other
direction. `SynchronousQueue` plus `Integer.MAX_VALUE` max means every burst creates
threads until `OutOfMemoryError: unable to create native thread`.

---

### Q4 — "Walk me through shutting an executor down."

**Mid-level answer:** "Call `shutdown()`, then `awaitTermination()`."

**Senior answer:** "Five steps, and each one exists because of a specific failure.

`shutdown()` first — it stops accepting new tasks but **still runs everything already
queued**. Then `awaitTermination` with a **bounded** timeout; I pick it against the
Kubernetes `terminationGracePeriodSeconds`, so 20 seconds inside a 30-second grace
period, leaving room for the rest of the context to close. An unbounded wait here means
one stuck task hangs the pod until `SIGKILL`, which is strictly worse.

If that times out, `shutdownNow()` — which moves to STOP, interrupts the workers, and
**returns the list of tasks it drained from the queue**. That return value is the only
record of the work you are abandoning, and throwing it away is how notifications
silently disappear on every deploy. I persist it to the outbox. Then a short second
`awaitTermination` so my 'still running' log line is accurate rather than premature.

Two more things. Tasks must actually respond to interruption — a task that swallows
`InterruptedException` will not stop, and `shutdownNow` becomes advisory. And since Java
19 `ExecutorService` is `AutoCloseable`, so try-with-resources works nicely for a
batch job with a lexical scope — but `close()` waits indefinitely, so it is the wrong
tool for a long-lived bean."

**What separates them:** knowing `shutdownNow` returns the drained list and that it
matters; bounding the wait against the platform's grace period; and knowing that
interruption is cooperative.

**Follow-up:** "Your pool threads are daemon threads. Does any of this change?" — yes:
the JVM can now exit with tasks half-done and no record. Non-daemon plus an explicit
shutdown is the safer default because forgetting is *loud*.

---

### Q5 — "This code has no logs and notifications are missing. Where do you look?"

```java
executor.submit(() -> notifier.send(order));
```

**Mid-level answer:** "Maybe the notifier service is down. I'd check its logs."

**Senior answer:** "Before I look downstream I'd check whether the exception is being
swallowed here, because this specific line does that by construction.

`submit()` wraps the task in a `FutureTask`. `FutureTask.run()` catches `Throwable` and
stores it in the future. It is re-thrown only from `Future.get()`. Nobody holds this
future, so nobody ever calls `get()`. The exception is therefore never seen: the worker
does not die, the `UncaughtExceptionHandler` never fires, `afterExecute` is called with
a null throwable, and nothing is logged anywhere. From the pool's point of view the task
completed successfully.

This is the difference from Node that catches me most: an unhandled promise rejection at
least prints a warning and by default terminates the process. Java prints nothing.

Two fixes. The narrow one: change `submit` to `execute`, so the exception propagates,
the worker dies, the handler fires and the pool replaces the worker. The proper one:
override `afterExecute` on the pool to unwrap the future and log — that covers both
`submit` and `execute` for every task, once, and it is fifteen lines.

I'd confirm it in thirty seconds with a two-line reproduction: submit a throwing task
and an executed throwing task to the same pool with an uncaught-exception handler
attached, and see which one prints."

**What separates them:** knowing the mechanism (`FutureTask.setException`), naming the
`afterExecute` fix, and having a reproduction rather than a theory.

**Follow-up:** "Does the same problem exist with `CompletableFuture`?" — yes, and worse.
Topic 91: a future nobody joins discards its exception entirely.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. `ThreadPoolExecutor` queues before it grows. The intuitive design would be the
   opposite — grow to max, then queue. Argue for the actual design: what does
   "queue first" buy, and for what kind of workload is it the right default?

2. `SynchronousQueue` has capacity zero, so `offer` fails unless a consumer is waiting.
   Explain, from the `execute()` algorithm alone, why that makes `newCachedThreadPool`
   grow a thread per submission — and predict what a `SynchronousQueue` with
   `maximumPoolSize = 4` does under a burst of 100.

3. Your pool is correctly sized at 16 for a 150 ms downstream. The downstream degrades
   to 600 ms. Nothing in your configuration changed. Describe, step by step, what
   happens to queue depth, pool size, rejection rate and customer-visible latency — and
   identify the earliest metric that would move.

4. An unbounded queue makes `maximumPoolSize` unreachable. Name a configuration where
   `corePoolSize` is unreachable, and say what it is good for.

5. `CallerRunsPolicy` is described as backpressure. It is also described as dangerous.
   Both are true. State precisely the property of the *caller* that decides which one it
   is, and give one example of each.

6. You bound the queue at 400 and the rejection rate under the spike is 8%. A colleague
   says "just raise the capacity to 4000 and the rejections go away". They are right that
   the rejections go away. Explain what actually happened to the work, and why the fix
   is worse than the problem.

7. Virtual threads (Topic 101) make a thread cheap, so `newVirtualThreadPerTaskExecutor`
   has no pool and no queue. That removes every problem in this document. What does it
   remove that you actually **wanted**, and what would you put back?

---

## Quick reference card

### The growth rule

```
execute(task):
  workerCount < corePoolSize   → NEW THREAD
  else queue.offer(task) == true → QUEUED             ← unbounded queue always lands here
  else workerCount < maximumPoolSize → NEW THREAD
  else → REJECT (handler runs ON THE CALLING THREAD)
```

### Constructor, annotated

```java
new ThreadPoolExecutor(
    16,                                  // core: Little's Law, threads ≈ λ × W
    32,                                  // max: only reachable if the queue can be FULL
    60L, TimeUnit.SECONDS,               // keepAlive: applies ABOVE core only
    new LinkedBlockingQueue<>(400),      // capacity = serviceRate × latencyBudget
    namedThreadFactory,                  // ALWAYS. Unnamed threads are undiagnosable.
    rejectionHandlerWithDurableFallback  // ALWAYS. And count the rejections.
);
```

### Sizing

| Work type | Formula | Typical result on 4 cores |
|---|---|---|
| I/O-bound | `threads ≈ throughput × latency` (Little's Law) | 16 for 100/s × 150 ms |
| CPU-bound | `N_cpu × U_cpu × (1 + W/C)` → ≈ `N_cpu` | 4, maybe 5 |
| Mixed | Two pools. Do not average them. | — |

Queue capacity = `serviceRate × acceptableWait`, where
`serviceRate = maximumPoolSize / meanTaskLatency`.

### `Executors.*` — what each one hides

| Factory | Hidden defect |
|---|---|
| `newFixedThreadPool(n)` | Unbounded `LinkedBlockingQueue`. Max == core, so max is moot too. |
| `newCachedThreadPool()` | `maximumPoolSize = Integer.MAX_VALUE` + `SynchronousQueue` ⇒ unbounded threads. |
| `newSingleThreadExecutor()` | Unbounded queue, and non-reconfigurable by design. |
| `newScheduledThreadPool(n)` | Unbounded internal `DelayedWorkQueue`; never grows past core. |
| `newVirtualThreadPerTaskExecutor()` | No pool, no queue ⇒ **no backpressure**. Topic 101. |

### Rejection policies

| Policy | Runs on | Use when |
|---|---|---|
| `AbortPolicy` (default) | caller (throws) | The caller can handle a `RejectedExecutionException` meaningfully |
| `CallerRunsPolicy` | caller (runs it) | The caller is a batch driver or another pool — **never a request thread** |
| `DiscardPolicy` | caller (no-op) | The work is genuinely disposable and you have written that down |
| `DiscardOldestPolicy` | caller | Latest-value-wins telemetry only |
| **Custom + durable fallback** | caller | **Anything customer-facing.** Shed latency, not work. |

### Shutdown

```java
executor.shutdown();                                       // 1. stop accepting; drain
if (!executor.awaitTermination(20, TimeUnit.SECONDS)) {     // 2. bounded, < grace period
    List<Runnable> lost = executor.shutdownNow();           // 3. interrupt + DRAIN
    persist(lost);                                          // 4. the only record. Keep it.
    executor.awaitTermination(5, TimeUnit.SECONDS);         // 5. so your log is honest
}
```

### Diagnostic commands

```bash
jcmd -l
jcmd <pid> Thread.print | grep -c '"orderflow-notify-'                       # pool size
jcmd <pid> Thread.print | grep -A10 '"orderflow-notify-' | grep -c getTask   # idle workers
jcmd <pid> Thread.print | grep -cE '"pool-[0-9]+-thread-[0-9]+"'             # unnamed pools
jcmd <pid> GC.class_histogram | grep 'LinkedBlockingQueue\$Node'             # queue depth
jcmd <pid> GC.heap_info
jcmd <pid> GC.heap_dump /tmp/pool.hprof
curl -s localhost:8080/actuator/metrics/executor.queued
curl -s localhost:8080/actuator/configprops | jq '..|.queueCapacity? // empty'
unzip -p "$JAVA_HOME/lib/src.zip" java.base/java/util/concurrent/Executors.java | grep -A8 newFixedThreadPool
```

### Gotchas checklist

- [ ] Never `Executors.newFixedThreadPool` / `newCachedThreadPool` in a service
- [ ] The queue is **bounded**, and the capacity is derived from a latency budget
- [ ] `maximumPoolSize` is meaningless unless the queue can be full
- [ ] A `ThreadFactory` with real names — unnamed threads are undiagnosable
- [ ] The rejection handler is chosen on purpose and **counts** its rejections
- [ ] A zero rejection rate under overload means no bound exists
- [ ] `execute` for fire-and-forget; if you use `submit`, override `afterExecute`
- [ ] Every executor has an owner and a shutdown path
- [ ] `shutdownNow()`'s return value is persisted, never discarded
- [ ] `awaitTermination` is bounded against the k8s grace period
- [ ] `ThreadLocal`/MDC is cleared in a `finally` — pooled threads outlive tasks (Topic 79)
- [ ] Spring's `ThreadPoolTaskExecutor` defaults to an unbounded queue — check `configprops`
- [ ] Queue depth is on a dashboard. If it is not, this failure is invisible.

---

## When would I use this at work?

**1. The first week on any Java codebase.** Grep for `Executors.new` and
`ThreadPoolTaskExecutor`, and check the queue capacity of every hit. In most codebases
this takes twenty minutes and finds at least one unbounded queue with no queue-depth
metric. It is the highest-value twenty minutes you will spend, and it is a very good
first contribution.

**2. During a "the service is slow but nothing is broken" incident.** Flat throughput,
climbing latency, zero errors, normal CPU and a healthy database is a *signature*, not a
mystery. You go straight to queue depth. Without this topic that incident is an hour of
looking at the database; with it, it is five minutes.

**3. In a capacity review, before the traffic arrives.** Product says the sale will be
3× normal volume. You compute `λ × W` for each pool, check each queue capacity against
its latency budget, and produce a table of what will shed and when. That converts "we
think it will hold" into "these three pools shed above 240 rps and here is what happens
to the work they shed" — which is the difference between an engineer and a senior
engineer in that meeting.

---

## Connected topics

**Prerequisites:**

- **84 — Threads vs the event loop:** a pool thread is a real OS thread with a real
  stack. That cost is why pools exist at all.
- **85 — `synchronized` and monitors:** `Worker` extends AQS and the queue uses locks;
  contention on the queue's put lock is a real (if usually irrelevant) cost.
- **86–87 — The JMM:** `ctl` is a `volatile`-semantics `AtomicInteger`; submitting a task
  establishes a happens-before edge to its execution, which is why a task safely sees
  everything the submitter wrote before `execute()`.
- **88 — Safe publication:** the reason the previous line is true, and the reason you do
  not need to synchronise the objects you pass into a task.
- **89 — `wait`/`notify`:** `getTask()` blocks in `workQueue.take()`, which is a guarded
  block. The pool's idle state is Topic 89's wait set, and the thread dump reads exactly
  as that topic taught you.
- **25 — Parallel streams:** the common `ForkJoinPool` is a pool you did not configure
  and cannot bound. Same lesson, less control.
- **55 — Transactions and the connection pool:** the same "bounded resource plus a
  queue" shape. A pool of 20 connections with a 30-second borrow timeout is this
  document with different nouns.
- **13 — `equals`/`hashCode`:** relevant the moment you key a dedupe or an in-flight map
  by a task identity.

**This unlocks:**

- **91 — `CompletableFuture`:** every `*Async` method takes an executor. This topic is
  how you choose it, and why passing none is a defect.
- **92 — Concurrent collections:** what to use for the pool's own shared state.
- **93 — `BlockingQueue`:** the queue you just bounded, in full — and where the capacity
  number comes from.
- **94 — Explicit locks:** `Worker`'s AQS lock, and `ReentrantLock` + `Condition` as the
  machinery inside every bounded queue.
- **95 — CAS and `LongAdder`:** `ctl` is a CAS loop; task counters at high throughput are
  where `LongAdder` earns its keep.
- **97 — `Semaphore` as a bulkhead:** the alternative to a pool when you want to limit
  *concurrency* rather than own *threads* — and the right tool once virtual threads make
  the thread itself free.
- **98 — Bug taxonomy:** the thread-leak drill (`new ExecutorService` per request) is
  Trap 2 in full.
- **100 — `ForkJoinPool`:** a different pool with a different growth model — work
  stealing, LIFO local, FIFO steal — and why blocking in it is catastrophic.
- **101 — Virtual threads:** removes the thread-cost constraint that motivates pooling,
  and makes **pooling them an anti-pattern**. But it also removes your backpressure, so
  read Topic 97 alongside it.
- **102 — Structured concurrency:** subtask lifetimes bound to a scope, which is the
  answer to "who owns this executor" at the language level.
- **105 — Backpressure:** the theory behind the number you wrote in the queue
  constructor.
- **109 — HikariCP:** a bounded pool with a queue and a rejection timeout. Everything
  here applies, with connections instead of threads.
- **111 — Bulkheads:** why the notification pool must be a *separate* pool from
  everything else, so its saturation cannot take down order placement.
- **115 — The outbox pattern:** where rejected work goes so that shedding latency does
  not mean losing work.
- **116 — Idempotency:** why a rejected-then-retried notification must not double-send.
- **118 — Metrics:** `ExecutorServiceMetrics`, and the cardinality rule that forbids
  tagging any of it with an order ID.
- **119 — Tracing:** a plain `ExecutorService` orphans the trace context; propagation
  across a pool boundary is explicit work.
- **79 — `ThreadLocal` leaks:** pooled threads outlive tasks, so an uncleaned
  `ThreadLocal` or MDC entry is retained for the life of the pool.

---

*Java baseline 21, running on JDK 25. Three things here are deliberately hedged rather
than asserted, and each has a command that settles it on your machine in minutes:
(1) Spring Boot's current default `queue-capacity` for `spring.task.execution` — read it
from `/actuator/configprops` rather than trusting any document, including this one;
(2) the exact return type and side effects of `ExecutorServiceMetrics.monitor` in your
Micrometer version, which has varied across releases; and (3) the precise per-`Node`
overhead of `LinkedBlockingQueue` on your JVM, which depends on compressed oops and
alignment — measure it with JOL rather than trusting my ~24 bytes. Everything else — the
growth rule, the `ctl` packing, the `execute()` algorithm, the factory-method defaults,
and the drill — has been stable since Java 5 and is verifiable from `src.zip` in under
five minutes.*
