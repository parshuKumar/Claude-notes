# 93 — `BlockingQueue` and Producer-Consumer

## Phase: 9 — Concurrency
## Category: DIFFERENTIATOR
## Java baseline: 21  |  Notes features from: 21 / 25
## Project spine: `orderflow`'s payment-notification queue — the hand-off between the request thread that places an order and the background workers that push notifications to the payment provider. This topic is where you decide, by choosing a number, what your service does when that downstream gets slow.

---

## Before anything else — what is and is not in this document

**I do not have a JVM. I have not run `orderflow`, k6, JMH, JFR or `jcmd` on your
machine. Nothing in this document is captured output, and I will never present
anything as if it were.**

Specifically, you will not find here:

- a queue-depth number presented as something I observed,
- a JMH table with `ops/s` or `ns/op` figures in it,
- a heap-dump histogram with byte counts,
- a p99 latency figure attributed to a run,
- a thread count, a GC pause, or a throughput number of any kind offered as measured.

Every claim that could only be settled by running something is presented as:

- the **exact command with exact flags**,
- **WHAT TO LOOK FOR** in your own output,
- a **"what you see → what it means"** table that covers the expected result *and* the
  surprising ones, because the surprising one is where the learning is.

### The one labelled exception

To teach you to *read* a thread dump, I show the **structure** of a `jcmd <pid>
Thread.print` section — the frame ordering and the `parking to wait for` line — with
every value replaced by `<tid>`, `0x...` or `<n>`. Each such block carries the inline
label:

> *illustration of the format, not captured output*

Placeholders only. Never a plausible-looking number dressed as evidence.
---

## Mechanical statement

Read this twice. Everything else in the document is elaboration.

> A `BlockingQueue` is a queue with two extra operations: **`put`, which blocks the
> producer while the queue is full**, and **`take`, which blocks the consumer while the
> queue is empty**. Those two blocking behaviours are the entire point. They are not a
> convenience over `poll` in a loop; they are the mechanism by which a fast producer is
> forced to slow down to the speed of a slow consumer.
>
> **A bounded queue's capacity IS your backpressure policy, expressed as a number: how
> much work you are willing to buffer before pushing back on the producer.**
>
> Three implementations, three physical shapes:
>
> - **`ArrayBlockingQueue`** — a fixed circular array, **one** `ReentrantLock`, two
>   `Condition`s (`notEmpty`, `notFull`). Capacity is mandatory. Producers and consumers
>   contend on the *same* lock.
> - **`LinkedBlockingQueue`** — a linked list with **two separate locks**, `putLock` and
>   `takeLock`, and an `AtomicInteger count` joining them. A producer and a consumer can
>   proceed genuinely in parallel. Capacity is **optional, and defaults to
>   `Integer.MAX_VALUE`** — which is to say, unbounded.
> - **`SynchronousQueue`** — **capacity zero**. It holds nothing. A `put` parks until a
>   `take` arrives, and then the element passes directly from one thread to the other.
>   It is a rendezvous, not a buffer.

Two consequences you will use for the rest of your career:

1. **The default is the bug.** `new LinkedBlockingQueue<>()` compiles, reads as
   deliberate, and is unbounded. So is `Executors.newFixedThreadPool(n)`, which uses one
   internally. An unbounded queue does not remove your capacity limit; it converts a
   *throughput* problem, which sheds load and stays up, into a *memory* problem, which
   does not.
2. **Choosing `put` vs `offer` vs `offer(timeout)` vs `add` is choosing your failure
   mode.** They are not four spellings of the same thing. One blocks, one silently
   drops, one waits then sheds, one throws. If you did not choose deliberately, the
   failure mode chose you.

---

## The bridge from what you know

### `PARTIAL ANALOGUE` — Node streams' `highWaterMark`

This is a genuine bridge, and it is the best one in Phase 9. Take it, then unlearn two
things precisely.

You already know this shape:

```js
// Node — you have written this
const ok = writable.write(chunk);
if (!ok) {
  readable.pause();
  writable.once('drain', () => readable.resume());
}
```

`highWaterMark` is the number of bytes (or objects, in object mode) a writable stream
will buffer internally before `write()` starts returning `false`. That `false` is the
stream telling you: *I am full, stop producing*. You respond by pausing the source. When
the buffer drains below the mark, `'drain'` fires and you resume.

**What transfers, completely:**

- The idea that a buffer size is a **policy decision**, not an implementation detail.
- The idea that the correct response to a full buffer is to **stop the producer**, not to
  grow the buffer.
- The idea that a pipeline runs at the speed of its slowest stage, and the buffer only
  smooths bursts — it never raises the sustained rate. You know this from Little's Law
  already; a buffer changes `L`, not `μ`.
- The vocabulary. `highWaterMark` ≈ capacity. `write() === false` ≈ `offer()` returning
  `false`. `'drain'` ≈ the `notFull` condition being signalled.

**What you must unlearn — two things, both important:**

**1. `highWaterMark` is advisory. A `BlockingQueue` bound is enforced.**

When Node's `write()` returns `false`, **the chunk was still accepted**. The buffer grew
past the high-water mark. Nothing stopped you. If you ignore the return value and keep
writing, the buffer grows without limit and you get a Node heap OOM — which is exactly
why the `write()`-returns-false pattern is a discipline you must follow rather than a
constraint the runtime imposes.

`BlockingQueue` does not ask politely. `put()` on a full `ArrayBlockingQueue` **parks the
calling thread**. `offer()` **refuses the element and returns `false`** — the element is
not in the queue. There is no path by which a bounded queue exceeds its bound. The
enforcement is real, and it happens whether or not you wrote the handling code.

That difference has a direct consequence: in Node, ignoring backpressure produces a
memory bug. In Java with a bounded queue, ignoring backpressure produces a *blocked
thread*, which is a different and usually better failure — but it is still a failure, and
Trap 4 below is about exactly which thread you blocked.

**2. Node has one thread. Java's producer and consumer are separate OS threads.**

In Node, the producer and consumer are callbacks on the same event loop. They never run
simultaneously. `stream.write()` and the drain handler are strictly interleaved, and the
buffer is an ordinary array touched by one thread.

In Java, `put` and `take` execute genuinely at the same instant on different cores. The
queue's internal state — head, tail, count — is shared mutable memory. Everything from
Topics 85 through 92 applies to it. What `BlockingQueue` gives you is that **all of that
is already handled correctly inside the class**, and the happens-before edge is part of
the contract:

> Actions in a thread prior to placing an object into a `BlockingQueue`
> *happen-before* actions subsequent to the access of that element from the
> queue in another thread.

That sentence, from the `BlockingQueue` javadoc, is the reason you can put a mutable
`PaymentNotification` object into a queue and the consumer sees fully-constructed fields
without any `volatile` of your own. It is Topic 86's happens-before edge, provided to you
as an API guarantee. **A plain `ArrayDeque` shared between threads gives you none of
it** — not the mutual exclusion, and not the visibility.
---

## What is this?

`java.util.concurrent.BlockingQueue<E>` extends `Queue<E>` and adds waiting versions of
insert and remove.

### The four ways to insert, and the four ways to remove

This table is the most practically important thing on the page. Every one of these is on
`BlockingQueue`; they differ *only* in what happens when the operation cannot proceed
right now.

| | Throws exception | Returns a special value | Blocks forever | Blocks with a deadline |
|---|---|---|---|---|
| **Insert** | `add(e)` → `IllegalStateException("Queue full")` | `offer(e)` → `false` | `put(e)` | `offer(e, t, unit)` → `false` on timeout |
| **Remove** | `remove()` → `NoSuchElementException` | `poll()` → `null` | `take()` | `poll(t, unit)` → `null` on timeout |
| **Examine** | `element()` → `NoSuchElementException` | `peek()` → `null` | *not offered* | *not offered* |

Read the insert row as four different products:

- **`add`** — "this must never be full; if it is, that is a bug, crash loudly." Correct for
  a queue you have proven cannot fill. Wrong everywhere else, because
  `IllegalStateException` is an unchecked exception that will propagate out of a request
  thread and become a 500.
- **`offer`** — "if there is no room, drop it and tell me." **Load shedding.** Correct when
  the item is genuinely droppable (a metric sample, a cache warm-up hint) and a
  catastrophe when it is not (a payment notification). The single most common bug in this
  whole topic is calling `offer` and ignoring the `boolean`.
- **`put`** — "if there is no room, I will wait." **Backpressure by blocking.** Correct
  when the calling thread is a dedicated feeder whose slowdown is the desired signal.
  Dangerous when the calling thread is a request thread — see Trap 4.
- **`offer(e, timeout, unit)`** — "wait up to this long, then shed." **Backpressure with a
  latency budget.** This is the one you want most often in a request path, because it
  bounds how long a customer waits before you tell them the truth.

There is no `add(e, timeout)` and no blocking `peek`. Both absences are deliberate.

### The implementations you should know

**`ArrayBlockingQueue<E>`** — bounded, capacity fixed at construction and mandatory.
Backed by a plain `Object[]` used as a circular buffer with `putIndex`/`takeIndex`.
One `ReentrantLock`, optionally fair. Zero allocation per element after construction.
**This is your default for a hand-off between threads.**

**`LinkedBlockingQueue<E>`** — optionally bounded; **unbounded if you use the
no-argument constructor**. Singly-linked list of `Node` objects; one allocation per
element inserted, one dead `Node` per element removed. Two locks, so a producer and a
consumer do not contend with each other — which is why it usually has higher throughput
than `ArrayBlockingQueue` under a two-sided load. **Always pass a capacity.**

**`SynchronousQueue<E>`** — capacity zero. `isEmpty()` is always `true`, `size()` is
always `0`, `peek()` always returns `null`, and iterating it yields nothing. `put` parks
until a consumer arrives to take the element directly. This is what
`Executors.newCachedThreadPool()` uses, and it is *why* that pool creates a new thread
per task when all threads are busy: the offer to the queue fails immediately, so the
executor's next step is to grow the pool.

**`LinkedTransferQueue<E>`** — unbounded, and a superset of the above semantics. Its
`transfer(e)` method behaves like `SynchronousQueue.put` — it waits for a consumer —
while `put(e)` returns immediately. Genuinely useful; rarely the first thing to reach
for, because it is unbounded and you now know what that means.

**`PriorityBlockingQueue<E>`** — **unbounded**, ordered by a `Comparator`. The
unboundedness is not optional and there is no constructor that bounds it. If you need
priority *and* a bound, you are building it yourself with a `ReentrantLock` and a
`Semaphore`, or you are using a bounded queue per priority class. Its `Iterator` does
**not** traverse in priority order — only `poll`/`take` respect ordering.

**`DelayQueue<E extends Delayed>`** — unbounded; an element becomes takeable only when
its delay expires. This is how you build a retry-with-backoff queue in-process, and it is
what Topic 111's retry scheduling looks like without a scheduler.

**`ConcurrentLinkedQueue<E>`** — *not* a `BlockingQueue`. Non-blocking, unbounded,
lock-free (Michael-Scott algorithm). No `put`, no `take`. Its `size()` is O(n) and
approximate. Use it when you want a queue and specifically do **not** want blocking; do
not use it as a work queue, because you will end up writing a spin loop over `poll()`,
which is worse than blocking in every dimension.

---

## Why does it matter?

Three reasons, in ascending order of how much they will cost you.

**1. It is the only correct way to hand work between threads.**

Topic 89 taught you `wait`/`notify` and told you never to ship it. This is what you ship
instead. A `BlockingQueue` is a correct bounded buffer written by people who got the
`while`-loop condition check, the `signalAll` versus `signal` choice, the interruption
semantics and the memory-model edge right. You will not beat it, and code review should
reject any attempt.

**2. The capacity number is a design decision that shows up in your SLO.**

Little's Law, which you already use: `L = λW`. In a queue, the time an item waits is
`W = L / μ` where `L` is the queue depth and `μ` is the consumer's service rate. So:

> **queue capacity ÷ consumer throughput = the worst-case added latency for an item that
> enters a full queue.**

That is not a metaphor. If your payment-notification consumers drain 200 items per second
and your queue holds 10,000, then an item entering a full queue waits **50 seconds**
before it is processed. If your product promise is "the customer sees the notification
within 5 seconds", your queue capacity is wrong by a factor of ten and no amount of
tuning elsewhere fixes it. The capacity is 1,000, and you derived it from the latency
budget rather than from a round number that looked safe.

This is the sentence that separates senior from mid in the interview: **"what capacity
should this queue have?" is answered with a division, not with an adjective.**

**3. Unbounded is not "no policy" — it is the worst policy, chosen by accident.**

An unbounded queue does not mean "we can handle any load". It means: when arrival rate
exceeds service rate, we will buffer the difference in heap, indefinitely, while latency
grows without bound and throughput stays flat, until the JVM dies. You have traded a
**bounded, observable, recoverable** failure (rejections, visible in a counter, handled by
a retry) for an **unbounded, invisible, fatal** one (heap growth, then `OutOfMemoryError`,
then a pod restart that loses everything in the queue).

Topic 90 made this argument about `ThreadPoolExecutor`. This topic makes it about the
queue itself, because you will also use queues that no executor owns.

---

## Machine-level reality

### `ArrayBlockingQueue` — one lock, two conditions

The fields, in essence:

```java
final Object[] items;
int takeIndex;
int putIndex;
int count;
final ReentrantLock lock;
private final Condition notEmpty;   // lock.newCondition()
private final Condition notFull;    // lock.newCondition()
```

Note what is *not* there: no `volatile`, no `AtomicInteger`. Every field is plain, because
every access is inside `lock`. The lock provides both mutual exclusion and the
happens-before edge, exactly as Topic 94 described.

`put(e)` in outline:

```java
lock.lockInterruptibly();
try {
    while (count == items.length)   // WHILE, not if — Topic 89's spurious-wakeup rule
        notFull.await();            // releases the lock and parks
    items[putIndex] = e;
    if (++putIndex == items.length) putIndex = 0;   // circular wrap
    count++;
    notEmpty.signal();              // wake exactly one waiting consumer
} finally {
    lock.unlock();
}
```

Three details worth carrying:

- **`await()` releases the lock.** That is the whole reason a `Condition` exists — a parked
  producer holds nothing, so consumers can proceed and make room.
- **`signal()`, not `signalAll()`.** Safe here precisely because producers and consumers
  wait on *different* `Condition` objects; the wrong-waiter problem that `wait`/`notify` on
  one monitor cannot avoid is designed out.
- **One lock for both sides.** A producer and a consumer cannot proceed simultaneously —
  the structural reason `ArrayBlockingQueue` has lower peak throughput than
  `LinkedBlockingQueue`, and lower, steadier latency with no allocation.

### `LinkedBlockingQueue` — two locks, and the cascading signal

```java
private final int capacity;             // Integer.MAX_VALUE from the no-arg constructor
private final AtomicInteger count = new AtomicInteger();
transient Node<E> head;
private transient Node<E> last;
private final ReentrantLock takeLock;
private final Condition notEmpty;
private final ReentrantLock putLock;
private final Condition notFull;
```

The head and tail of the list are guarded by *different* locks. That is the entire design.
A producer appending at `last` never touches `head`; a consumer removing at `head` never
touches `last`. They meet only at `count`, which is why `count` must be an
`AtomicInteger` rather than a plain `int`.

The `put` path does something that looks strange until you see why:

```java
int c;
putLock.lockInterruptibly();
try {
    while (count.get() == capacity) notFull.await();
    enqueue(new Node<E>(e));                 // ALLOCATION, per element
    c = count.getAndIncrement();
    if (c + 1 < capacity) notFull.signal();  // <-- the cascading signal
} finally {
    putLock.unlock();
}
if (c == 0) signalNotEmpty();                // take the OTHER lock, briefly
```

- **`if (c + 1 < capacity) notFull.signal()`** — a producer that just inserted, and sees
  there is still room, wakes *another producer*. This is the "cascading notify": rather
  than one thread waking everybody, each waking thread wakes at most one more, so the
  wakeups propagate down the queue as capacity allows. It keeps the wakeup count
  proportional to available space rather than to waiter count.
- **`if (c == 0) signalNotEmpty()`** — only when the queue transitioned from empty do
  producers need to acquire `takeLock` to signal. In steady state, with a non-empty queue,
  a producer never touches the consumer's lock at all. **That is where the throughput comes
  from.**
- **`new Node<E>(e)`** — every insert allocates. At 5,000 notifications per second that is
  5,000 short-lived objects per second added to your allocation rate. Topic 68 tells you
  these die in Eden and are cheap; Topic 79 tells you that if the queue grows unboundedly
  they are promoted and are not.

### `SynchronousQueue` — no capacity, direct hand-off

There is no array and no list of elements. There is a **stack of waiting threads**
(`TransferStack`, the default, unfair) or a **queue of waiting threads**
(`TransferQueue`, when constructed with `fair = true`). Each waiting thread publishes a
node describing what it wants: to hand an item over, or to receive one.

The transfer algorithm, conceptually:

1. Look at the head node.
2. If it is empty, or holds a request of the **same** kind as mine (I want to put, it
   wants to put), push my node and park.
3. If it holds a request of the **complementary** kind, CAS my item into it, unpark that
   thread, and we both return. The element passes from my stack frame to yours. **It is
   never stored in the queue, because there is no storage.**

Consequences you will actually hit:

- `size()` is always `0`; `isEmpty()` always `true`; `peek()` always `null`.
- `offer(e)` with no consumer *currently parked* returns **`false` immediately** and
  discards the element. That is Trap 2, and it is silent.
- A depth gauge on a `SynchronousQueue` reports `0` forever. A flat-zero queue-depth panel
  is not evidence of health — check the implementation first.

### Where each queue op leaves you in a thread dump

You need this to read the Failure drill, and it is the same table you built in Topic 94
applied to queues.

| Operation | Blocking mechanism | Thread state | The frame you will see |
|---|---|---|---|
| `take()` on empty | `Condition.await()` → `LockSupport.park` | **`WAITING (parking)`** | `AbstractQueuedSynchronizer$ConditionObject.await` under `ArrayBlockingQueue.take` |
| `put()` on full | `Condition.await()` → `LockSupport.park` | **`WAITING (parking)`** | `...ConditionObject.await` under `ArrayBlockingQueue.put` |
| `poll(t, unit)` | `Condition.awaitNanos()` → `parkNanos` | **`TIMED_WAITING (parking)`** | `...ConditionObject.awaitNanos` |
| `offer(e, t, unit)` | `Condition.awaitNanos()` | **`TIMED_WAITING (parking)`** | same |
| contending for the queue's `ReentrantLock` | AQS acquire | **`WAITING (parking)`** | `AbstractQueuedSynchronizer.acquire` — note: *not* `BLOCKED` |
| `SynchronousQueue.put` with no taker | `LockSupport.park` directly | **`WAITING (parking)`** | `SynchronousQueue$TransferStack.transfer` |

**The trap in reading these:** the `parking to wait for <0x...>` line names a
`ConditionObject`, *not* your queue — two different queues produce identical text. The only
way to know **which** queue is to read the frame above it (`ArrayBlockingQueue.take` versus
`LinkedBlockingQueue.take`) and, for two queues of the same type, to have named your
threads. **Name your threads.** It is the cheapest diagnostic investment in this phase.

A corollary you will need at 3am: a healthy idle consumer and a hung pipeline are **the
same `Thread.State` in the same frame**. The dump cannot separate them; the depth gauge
can. Parked in `take()` with depth 0 is health. Parked in `take()` with depth at capacity
is an outage.

### `[JAVA 25]` note

Compact object headers (`-XX:+UseCompactObjectHeaders`, Topic 69) shrink the object
header, which changes the per-element footprint of `LinkedBlockingQueue`'s `Node`
objects. It changes the arithmetic of "how much heap does a 100,000-element queue cost",
not the semantics of anything here. Settle the actual number with JOL rather than with a
calculation; the command is in the Measurement section.

---

## Concurrency trace

**Before any correct code.** This is `orderflow`'s payment-notification pipeline at the
Topic 65 baseline, with the default that everybody writes.

**The setup.** When an order is placed, the request thread enqueues a
`PaymentNotification` for the background pool to POST to the payment provider. The queue
is created with the constructor that looks obviously correct:

```java
private final BlockingQueue<PaymentNotification> queue = new LinkedBlockingQueue<>();
```

At baseline: **400 orders per second** arriving. The notification pool has **8 consumer
threads**, and each POST to the provider normally takes **20 ms**, so the pool drains
`8 / 0.020 = 400` items per second. Arrival rate equals service rate exactly. This has
worked for months.

At **T+0** the payment provider's p99 degrades: each POST now takes **400 ms** instead of
20 ms. Nothing is down. Nothing throws. The provider is simply slow.

Consumer throughput is now `8 / 0.400 = 20` items per second. Arrivals stay at 400.

| Step | Thread A — `http-nio-8080-exec-*` (order placement, 400/s) | Thread B — `payment-notify-1..8` (consumers, 8 threads) | Queue depth / outcome |
|---|---|---|---|
| 1 | `queue.put(n)` — returns in nanoseconds, never blocks | `queue.take()` → POST, 20 ms | depth ≈ 0 · **steady state, healthy** |
| 2 | still 400/s | provider slows to 400 ms per POST | depth ≈ 0 · the only visible change is the consumer's latency |
| 3 | 400 puts in this second | 20 takes in this second | **depth = 380** after one second |
| 4 | 400 puts | 20 takes | depth = 760 after two seconds |
| 5 | …60 seconds pass… | …consumers keep draining at 20/s… | **depth ≈ 22,800.** `put` has still never blocked. No log line. No metric alarm, because nobody gauged the depth. |
| 6 | HTTP p99 for `POST /orders` **unchanged** — the request thread returns instantly | each `take` returns an item enqueued **19 minutes ago** | **The queue has become a time machine.** Notifications are being sent for orders the customer placed and then abandoned. |
| 7 | …10 minutes… 240,000 items enqueued… | 12,000 drained | depth ≈ 228,000 · each `Node` + `PaymentNotification` retained in heap |
| 8 | GC begins promoting the queue's contents out of Eden — they are no longer short-lived | consumers slow further as GC steals CPU | **Old-gen fills. GC time climbs. Throughput of the whole JVM drops.** |
| 9 | request latency finally rises — not from the queue, from **GC pauses** | consumer throughput falls below 20/s, worsening the ratio | The feedback loop is now positive. Depth accelerates. |
| 10 | `queue.put(n)` throws `OutOfMemoryError: Java heap space` on a request thread | consumers die with the same error | **JVM dead. Every item in the queue is lost.** |
| 11 | k8s restarts the pod | — | The 228,000 undelivered notifications existed only in heap. **They are gone.** |

**Outcome, in business terms.**

For nineteen minutes, `orderflow` reported itself completely healthy. `POST /orders`
returned 201 with a p99 inside the Topic 65 baseline. The dashboard was green. No alert
fired, because the only failing thing — notification delivery — was measured nowhere.

Then the pod died and **228,000 payment notifications were destroyed**. The payment
provider was never told about those orders. Reconciliation the next morning shows
228,000 orders in `orderflow` with no corresponding provider record: money taken with no
notification, or notifications the provider will now never receive, depending on which
side of the transaction each order was on. The recovery is a manual reconciliation job,
written under pressure, against a provider API with a rate limit.

The failure was not the provider slowing down. **Providers slow down; that is a
Tuesday.** The failure was that a single missing constructor argument converted a
recoverable downstream slowdown into permanent data loss. Nothing in the code review
would have looked wrong.

### The same fifty seconds, with `new LinkedBlockingQueue<>(1000)`

Same slowdown, same load, one number changed. Producers now call
`offer(n, 200, MILLISECONDS)`.

| Step | Thread A — request thread | Thread B — consumers | Queue depth / outcome |
|---|---|---|---|
| 1 | `offer(n, 200ms)` → `true` immediately | `take()` → POST, 20 ms | depth ≈ 0 · healthy |
| 2 | 400/s | provider slows to 400 ms | depth climbing at 380/s |
| 3 | ~2.6 s later | 20/s | **depth = 1000. The queue is full.** |
| 4 | `offer(n, 200ms)` → `Condition.awaitNanos` → **`TIMED_WAITING`** | draining at 20/s | request thread now waits — for at most 200 ms |
| 5 | 200 ms elapses, room did not appear → returns **`false`** | — | **The producer is told.** |
| 6 | increments `notifications.rejected` counter, writes the notification to the outbox table, returns 201 | — | The order still succeeds. The notification is durable. |
| 7 | steady state: ~20/s enter the queue, ~380/s go to the outbox | 20/s drained | **Heap flat. Depth pinned at 1000. Nothing lost.** |
| 8 | `notifications.rejected` alert fires at T+3 s | — | **A human knows within seconds, not nineteen minutes.** |
| 9 | provider recovers; consumers return to 20 ms | 400/s | queue drains, rejections stop, outbox replays |

**Outcome, in business terms.** Zero notifications lost. The failure was visible in three
seconds instead of invisible for nineteen minutes. `POST /orders` latency rose by at most
200 ms during the incident — a real cost, deliberately chosen, and bounded by a number
you can point at in a design document. The JVM never came close to OOM.

**Read the difference between those two tables carefully.** The bounded version *fails
more*. It has a rejection counter that goes up, an alert that fires, and 200 ms of extra
latency on the request path. That is not a regression. **The unbounded version failed
exactly as much; it simply hid the failure until the failure was fatal and irreversible.**

---

## Example 1 — minimal

A single producer, a single consumer, one bounded queue. This is the whole pattern.

```java
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.BlockingQueue;

public final class MinimalHandoff {

    // Capacity is mandatory on ArrayBlockingQueue. That is a feature:
    // the API refuses to let you not decide.
    private static final BlockingQueue<Integer> STOCK_DECREMENTS =
            new ArrayBlockingQueue<>(100);

    public static void main(String[] args) throws InterruptedException {

        Thread consumer = new Thread(() -> {
            try {
                while (true) {
                    Integer productId = STOCK_DECREMENTS.take();   // parks when empty
                    if (productId == POISON) break;                // see below
                    applyDecrement(productId);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();   // restore the flag. Always.
            }
        }, "inventory-applier");

        consumer.start();

        for (int productId = 1; productId <= 1_000; productId++) {
            STOCK_DECREMENTS.put(productId);          // parks when full
        }
        STOCK_DECREMENTS.put(POISON);

        consumer.join();
    }

    private static final Integer POISON = Integer.valueOf(-1);

    private static void applyDecrement(int productId) { /* ... */ }
}
```

**What to notice, line by line:**

- **`new ArrayBlockingQueue<>(100)`** — the capacity is a constructor *requirement*.
  Contrast with `new LinkedBlockingQueue<>()`, which compiles and is unbounded. The API
  is telling you which one it thinks is the default.
- **`take()` parks the consumer when empty.** No spin loop, no `sleep(10)`. Zero CPU while
  waiting, woken by the producer's `signal()` in microseconds.
- **`put()` parks the producer when full.** With 1,000 items and capacity 100 the producer
  *will* block. That blocking is the backpressure: the loop runs at the consumer's speed
  and the footprint stays at 100 items however much the producer wants to send.
- **The poison pill.** There is no `queue.close()`. You enqueue a sentinel the consumer
  recognises — **N pills for N consumers**, since each is taken by exactly one. The
  alternative is interruption, which requires every consumer to handle
  `InterruptedException` correctly (Trap 3).
- **`Thread.currentThread().interrupt()`** in the catch. `take()` clears the interrupt flag
  when it throws. If you do not restore it, code further up the stack — including
  `ExecutorService.shutdownNow()` machinery and any library you call — cannot tell that an
  interrupt happened. This is the most commonly omitted line in Java concurrency.

### The variation that looks equivalent and is not

```java
// WRONG in three separate ways
while (true) {
    Integer id = queue.poll();          // 1. returns null immediately when empty
    if (id != null) applyDecrement(id);
    Thread.sleep(10);                   // 2. adds up to 10ms latency to every item
}                                       // 3. burns a wakeup 100x/second doing nothing
```

This is the shape a Node developer writes first, because polling feels natural when you
have never had a thread that can genuinely sleep and be woken. It is worse in every
dimension: higher latency, more CPU, more scheduler churn, and — with `poll()`'s `null`
return — an `if` that a future maintainer will convert to something that NPEs. **`take()`
is not a convenience over this. It is the correct version of it.**

---

## Example 2 — production scenario (on the project spine)

### The constraints, stated before any code

This is the part most engineers skip, and it is the part the interview is actually about.

| Constraint | Value | Where it comes from |
|---|---|---|
| Arrival rate | 400 notifications/s | Topic 65 baseline: 400 orders/s, one notification each |
| Consumer service time (healthy) | 20 ms per POST | measured against the provider at baseline |
| Consumer service time (degraded) | up to 2 s | the provider's own published p99.9 |
| Notification delivery SLO | 95% within 5 s of order placement | product requirement |
| Order placement p99 SLO | 250 ms | Topic 65 baseline, must not regress |
| Notification durability | **must not be lost** | it is money |

**Deriving the pool size** (Topic 90's Little's Law): to sustain 400/s at 20 ms each,
in-flight work is `400 × 0.020 = 8`, so **8 consumer threads** at health. At 400 ms you
would need 160 threads — which you will not provision, because the provider would not
survive it. Degradation *will* produce a backlog. The design question is what happens then.

**Deriving the capacity.** The SLO says 5 seconds. At the healthy drain rate of 400/s,
five seconds of buffer is `400 × 5 = 2000` items. Round down to **1000** to leave headroom
for the drain rate falling: at 1000 capacity and a drain rate of 200/s (a 10× provider
slowdown), the oldest item in a full queue is 5 seconds old — exactly at the SLO
boundary. **Capacity 1000 is not a round number. It is `SLO × throughput`, with the
throughput chosen at the worst rate you are willing to still call "working".**

**Deriving the offer timeout.** The order p99 budget is 250 ms and the rest of the request
uses most of it. 200 ms would eat the budget. Use **50 ms**: long enough to ride out a
momentary consumer stall, short enough to be invisible in the latency histogram.

### The code that ships and loses 228,000 notifications

```java
@Service
public class PaymentNotificationService {

    // The default constructor. Unbounded. This single line is the whole bug.
    private final BlockingQueue<PaymentNotification> queue = new LinkedBlockingQueue<>();

    private final ExecutorService consumers = Executors.newFixedThreadPool(8);

    @PostConstruct
    void start() {
        for (int i = 0; i < 8; i++) {
            consumers.submit(this::drainLoop);
        }
    }

    // Called from the request thread, inside order placement.
    public void notifyAsync(PaymentNotification n) {
        queue.put(n);            // never blocks, because the queue is never full
    }

    private void drainLoop() {
        while (true) {
            try {
                PaymentNotification n = queue.take();
                gateway.post(n);                   // 20 ms healthy, 400 ms degraded
            } catch (Exception e) {
                log.error("notification failed", e);   // and Trap 3 lives here
            }
        }
    }
}
```

Every line of that passes review. It has a bounded pool, it uses an `ExecutorService`, it
logs its errors, it does not block the request thread. It is the trace above.

### Fix 1 — bound the queue and choose the failure mode explicitly

```java
@Service
public class PaymentNotificationService {

    private static final int CAPACITY = 1_000;          // = 5s SLO × 200/s worst drain
    private static final long OFFER_TIMEOUT_MS = 50;    // = 20% of the p99 budget

    private final BlockingQueue<PaymentNotification> queue =
            new ArrayBlockingQueue<>(CAPACITY);

    private final NotificationOutbox outbox;     // durable, Topic 115
    private final MeterRegistry meters;
    private final Counter rejected;

    private final ExecutorService consumers = Executors.newFixedThreadPool(
            8, r -> Thread.ofPlatform().name("payment-notify-", 0).unstarted(r));

    PaymentNotificationService(NotificationOutbox outbox, MeterRegistry meters) {
        this.outbox = outbox;
        this.meters = meters;
        this.rejected = Counter.builder("orderflow.notifications.rejected")
                .description("notifications that could not be queued within the timeout")
                .register(meters);
        // Topic 118: the gauge is not optional. An unmeasured queue is an invisible queue.
        Gauge.builder("orderflow.notifications.queue.depth", queue, Collection::size)
                .register(meters);
        Gauge.builder("orderflow.notifications.queue.remaining",
                        queue, BlockingQueue::remainingCapacity)
                .register(meters);
    }

    /** Called on the request thread. Must not block for longer than OFFER_TIMEOUT_MS. */
    public void notifyAsync(PaymentNotification n) throws InterruptedException {
        boolean queued = queue.offer(n, OFFER_TIMEOUT_MS, TimeUnit.MILLISECONDS);
        if (!queued) {
            rejected.increment();
            outbox.persist(n);      // durable fallback — the item is NOT dropped
        }
    }

    private void drainLoop() {
        while (!Thread.currentThread().isInterrupted()) {
            PaymentNotification n = null;
            try {
                n = queue.take();
                gateway.post(n);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();     // restore and exit the loop
                return;
            } catch (RuntimeException e) {
                log.error("notification {} failed permanently", n == null ? "?" : n.id(), e);
                if (n != null) outbox.persist(n);       // never silently drop
            }
        }
    }

    @PreDestroy
    void stop() throws InterruptedException {
        consumers.shutdownNow();                        // interrupts threads parked in take()
        if (!consumers.awaitTermination(10, TimeUnit.SECONDS)) {
            log.warn("notification consumers did not stop within 10s");
        }
    }
}
```

**What changed, and why each change is load-bearing:**

| Change | What it buys |
|---|---|
| `ArrayBlockingQueue<>(1000)` | The bound is enforced and the constructor made you state it. Heap footprint is now `O(1000)`, not `O(unbounded)`. |
| `offer(n, 50ms, MS)` instead of `put(n)` | The request thread can wait, but only for a budget you chose. It can never wait forever. |
| The `if (!queued)` branch | **The failure has a handler.** This is the difference between backpressure and data loss. |
| `outbox.persist(n)` | The overflow path is *durable*. Shedding to a database is not shedding; it is deferring. Topic 115. |
| Two `Gauge`s | Depth and remaining capacity. Without these the queue is invisible, which was the real fault in the original. |
| A `Counter` on rejections | The alert. Rate-of-rejections is the signal that the downstream is degraded — often before the downstream's own monitoring notices. |
| Named threads (`payment-notify-0..7`) | So the thread dump in the Failure drill is readable. |
| `catch (InterruptedException)` → restore + `return` | `shutdownNow()` now actually works. See Trap 3. |
| `catch (RuntimeException)` separately | A poisoned message no longer kills the consumer thread silently. See Trap 5. |

### Fix 3 — the one that is usually right at scale: do not own the queue

At 400/s with a durability requirement, an in-process queue is an odd place for money to
live. The version that ships in a mature system writes to the outbox **always**, and a
separate poller drains the outbox to the provider:

```java
@Transactional
public Order placeOrder(PlaceOrderCommand cmd) {
    Order order = // ... inventory decrement, wallet debit ...
    outbox.persist(PaymentNotification.of(order));   // same transaction as the order
    return order;
}
```

Now the notification commits atomically with the order, survives a `kill -9`, and the
in-memory queue's only job is the poller's local work buffer — where a bounded queue with
`put()` is *exactly* right, because the poller is a dedicated feeder whose slowdown is the
desired signal.

**The general rule:** an in-memory queue is a *rate-smoothing* device, not a *durability*
device. If losing its contents on a pod restart is unacceptable, the queue is the wrong
storage and no capacity number fixes that.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — `new LinkedBlockingQueue<>()` — the queue is your OOM

**Wrong approach**

```java
private final BlockingQueue<PaymentNotification> queue = new LinkedBlockingQueue<>();
// or, equivalently and even more invisibly:
private final ExecutorService pool = Executors.newFixedThreadPool(8);
```

**Exact symptom** — in this order, over minutes to hours:

1. Everything is fine. Producer latency is *excellent*, because `put` never blocks.
2. Old-gen occupancy climbs monotonically across GC cycles in `-Xlog:gc*`. It does not
   sawtooth back down.
3. GC frequency and duration climb. Application throughput falls for no visible reason.
4. `java.lang.OutOfMemoryError: Java heap space`, thrown from wherever the JVM happened to
   be — often nowhere near the queue.
5. In the heap dump (Topic 79), the dominator tree is topped by
   `java.util.concurrent.LinkedBlockingQueue` or `LinkedBlockingQueue$Node`, retaining
   millions of your domain objects.

**Root cause.** The no-argument constructor sets `capacity = Integer.MAX_VALUE`. There is
no warning, no deprecation, no lint rule in a default setup.
`Executors.newFixedThreadPool` and `newSingleThreadExecutor` both construct one
internally, so a pool you never wrote a queue for still has an unbounded one.

**Fix.** Pass a capacity, derived from a latency budget. Then make it impossible to
regress:

```java
// ArrayBlockingQueue has no unbounded constructor. Prefer it for exactly this reason.
new ArrayBlockingQueue<>(1_000)

// If you need LinkedBlockingQueue's two-lock throughput, pass the bound explicitly:
new LinkedBlockingQueue<>(1_000)

// And ban the executor factory methods in the codebase:
new ThreadPoolExecutor(8, 8, 0L, TimeUnit.MILLISECONDS,
        new ArrayBlockingQueue<>(1_000),
        namedFactory("payment-notify-"),
        new ThreadPoolExecutor.CallerRunsPolicy());
```

Add an ArchUnit or Checkstyle rule forbidding `Executors.newFixedThreadPool` and the
no-arg `LinkedBlockingQueue` constructor. **A rule, not a review comment** — review catches
it four times in five, and the fifth is an outage.

### Trap 2 — `SynchronousQueue` misunderstood as having capacity

**Wrong approach**

```java
private final BlockingQueue<PaymentNotification> handoff = new SynchronousQueue<>();

public void notifyAsync(PaymentNotification n) {
    handoff.offer(n);        // return value ignored
}
```

Somebody chose `SynchronousQueue` after reading that it is "the fastest queue" — which is
true in the narrow sense that a direct hand-off has no storage overhead — and then used
it as if it buffered.

**Exact symptom.** Notifications are delivered *sometimes*. Under low load, most get
through. Under any load, the delivery rate is a fraction of the enqueue rate. **Nothing
throws. Nothing logs. No metric moves.** The queue-depth gauge, if there is one, reads
`0` — correctly, and uselessly. The bug is usually found weeks later by the payment
provider's reconciliation, not by you.

A second shape of the same misunderstanding:

```java
new ThreadPoolExecutor(8, 8, 60L, SECONDS, new SynchronousQueue<>())
```

with `corePoolSize == maximumPoolSize`. Here the symptom is the opposite and much
louder: `RejectedExecutionException` under any burst, because the queue can never hold a
task and the pool can never grow past 8. `newCachedThreadPool` uses a `SynchronousQueue`
precisely *because* its `maximumPoolSize` is `Integer.MAX_VALUE` — the queue's refusal is
the growth signal. Pair a `SynchronousQueue` with a fixed maximum and you have built a
hard rejection at exactly `maximumPoolSize` concurrent tasks.

**Root cause.** `SynchronousQueue` has capacity **zero**. `offer(e)` with no consumer
*currently parked in `take()`* returns `false` immediately and discards the element. It is
a rendezvous point, not a buffer, and the `BlockingQueue` interface it implements makes
that indistinguishable at the call site.

**Fix.** Two, depending on intent:

- If you wanted buffering: use `ArrayBlockingQueue(n)`. That was always the answer.
- If you genuinely want a rendezvous — hand this to a consumer *right now* or do not
  proceed — keep `SynchronousQueue` and use the blocking form with a deadline, and handle
  the false:

```java
if (!handoff.offer(n, 50, TimeUnit.MILLISECONDS)) {
    rejected.increment();
    outbox.persist(n);
}
```

**And never ignore the `boolean` returned by `offer`** — treat an ignored return value as
a build error. It is the same bug as ignoring Node's `stream.write()`, and it fails the
same way.

### Trap 3 — swallowing `InterruptedException` in the drain loop

**Wrong approach**

```java
private void drainLoop() {
    while (true) {
        try {
            gateway.post(queue.take());
        } catch (Exception e) {
            log.error("notification failed", e);   // catches InterruptedException too
        }
    }
}
```

**Exact symptom.** Deployment hangs. `@PreDestroy` runs `shutdownNow()`,
`awaitTermination(10s)` returns `false`, the container's graceful shutdown period expires,
and Kubernetes sends `SIGKILL`. In the logs you see one `InterruptedException` stack trace
per consumer thread at shutdown, logged at ERROR, and then nothing. Rolling deploys take
`terminationGracePeriodSeconds` every time. Everyone assumes that is normal.

Worse variant: the thread does not exit and the executor is never garbage collected, so
each redeploy of the application context inside a long-lived JVM leaks 8 threads. That is
Topic 98's thread leak arriving by a different road.

**Root cause.** `shutdownNow()` works by calling `Thread.interrupt()` on each worker. A
thread parked in `take()` responds by throwing `InterruptedException` **and clearing the
interrupt flag**. Catching it and continuing the `while (true)` loop means the thread
immediately re-enters `take()` and parks again, with the flag now cleared. The interrupt
has been consumed and discarded. There is no second interrupt.

**Fix.** Handle `InterruptedException` distinctly from every other exception, restore the
flag, and leave:

```java
private void drainLoop() {
    while (!Thread.currentThread().isInterrupted()) {
        PaymentNotification n = null;
        try {
            n = queue.take();
            gateway.post(n);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();   // restore, so callers up-stack can see it
            return;                               // and exit. This is the cooperative contract.
        } catch (RuntimeException e) {
            log.error("notification {} failed", n, e);
            if (n != null) outbox.persist(n);
        }
    }
}
```

**The rule, memorised:** *catch `InterruptedException` separately, and either propagate it
or restore the flag and return. Never `catch (Exception)` around a blocking call.*

### Trap 4 — `put()` from a request thread, with a long-blocking consumer

**Wrong approach**

```java
// bounded queue — good. put() from the Tomcat thread — not good.
public void notifyAsync(PaymentNotification n) throws InterruptedException {
    queue.put(n);                 // blocks indefinitely when full
}
```

This is the fix for Trap 1 applied without thinking about *which thread* is now doing the
waiting.

**Exact symptom.** The provider slows. Within seconds the queue fills. Then:

- `POST /orders` p99 goes from 250 ms to *seconds*, then to the load balancer's timeout.
- **`GET /products` also starts timing out** — an endpoint that touches neither the queue
  nor the payment provider.
- A thread dump shows a large fraction of `http-nio-8080-exec-*` threads in
  `WAITING (parking)` inside `ArrayBlockingQueue.put`.
- Tomcat's `busy threads` metric pins at `maxThreads`. New connections queue in the
  accept backlog; eventually the LB returns 503.
- Heap is flat. GC is fine. CPU is low. **Everything that usually indicates a problem
  looks healthy.**

**Root cause.** You bounded the queue, so backpressure works — it propagated upstream
exactly as designed. But upstream is the shared Tomcat request-thread pool, which is a
**global resource**. Blocking it for one slow dependency converts a single-dependency
degradation into a whole-service outage. This is the failure that Topic 111's bulkhead
pattern exists to prevent, and it is the same shape as Topic 55's "HTTP call inside a
transaction pins a connection".

**Fix.** Bound the *wait*, not just the queue, and give the overflow somewhere to go:

```java
boolean queued = queue.offer(n, 50, TimeUnit.MILLISECONDS);
if (!queued) { rejected.increment(); outbox.persist(n); }
```

**And the general principle**, which is worth more than the fix: *backpressure must
terminate somewhere.* If every stage blocks the stage before it, the pressure eventually
reaches your request threads, and a request thread pool is not a place to store load. The
last stage before a shared resource must **shed** — reject, degrade, or persist — rather
than block. `put()` is correct only when the calling thread is a dedicated feeder whose
sole job is feeding that queue.

## Hands-on proof

No JVM ran here. These are the commands; the outputs are yours.

### Setup

```bash
java -version && mkdir -p /tmp/bq && cd /tmp/bq
```

### Proof 1 — settle the unbounded default from your own JDK

Do not take my word for `Integer.MAX_VALUE`. Read it in your own JDK's source
(`unzip -p "$JAVA_HOME/lib/src.zip" java.base/java/util/concurrent/LinkedBlockingQueue.java`
— look for the no-arg constructor delegating to `this(Integer.MAX_VALUE)`), then confirm
it behaviourally, which is the proof that matters:

```java
// Cap.java
import java.util.concurrent.*;
public class Cap {
    public static void main(String[] a) {
        System.out.println("LBQ()      remaining = " + new LinkedBlockingQueue<>().remainingCapacity());
        System.out.println("LBQ(1000)  remaining = " + new LinkedBlockingQueue<>(1000).remainingCapacity());
        System.out.println("ABQ(1000)  remaining = " + new ArrayBlockingQueue<>(1000).remainingCapacity());
        System.out.println("SyncQ      remaining = " + new SynchronousQueue<>().remainingCapacity());
        System.out.println("SyncQ      size      = " + new SynchronousQueue<>().size());
        System.out.println("PBQ        remaining = " + new PriorityBlockingQueue<>().remainingCapacity());
    }
}
```

```bash
java Cap.java
```

| What you see | What it means |
|---|---|
| `LBQ() remaining = 2147483647` | Confirmed unbounded. That is `Integer.MAX_VALUE`. Your "fixed thread pool" has this queue. |
| `SyncQ remaining = 0` and `size = 0` | Confirmed zero capacity. A depth gauge on this reports 0 forever. |
| `PBQ remaining = 2147483647` | `PriorityBlockingQueue` is unbounded and **has no bounded constructor**. If you need priority plus a bound you must build it. |
| Anything else | Read the source you just unzipped; your JDK differs from my description and the source is authoritative. |

### Proof 2 — see `put` actually block

```java
// Block.java
import java.util.concurrent.*;
public class Block {
    public static void main(String[] a) throws Exception {
        BlockingQueue<Integer> q = new ArrayBlockingQueue<>(3);
        Thread.currentThread().setName("producer");
        System.out.println("pid " + ProcessHandle.current().pid());
        for (int i = 0; ; i++) {
            System.out.println("putting " + i + " (depth=" + q.size() + ")");
            q.put(i);                 // will park at i == 3
        }
    }
}
```

```bash
java Block.java
```

**WHAT TO LOOK FOR:** output stops after `putting 3`. The process does not exit and does
not consume CPU. In a second terminal, using the printed pid:

```bash
jcmd <pid> Thread.print | grep -A 12 '"producer"'
```

**Structure of what you will find** — *illustration of the format, not captured output*:

```
"producer" #<n> prio=5 os_prio=<n> cpu=<n>ms elapsed=<n>s tid=0x... nid=0x... waiting on condition  [0x...]
   java.lang.Thread.State: WAITING (parking)
        at jdk.internal.misc.Unsafe.park(java.base@<ver>/Native Method)
        - parking to wait for  <0x...> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
        at java.util.concurrent.locks.LockSupport.park(...)
        at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(...)
        at java.util.concurrent.ArrayBlockingQueue.put(...)
        at Block.main(Block.java:<n>)
```

| What you see | What it means |
|---|---|
| `WAITING (parking)` with `ConditionObject` in the `parking to wait for` line | A blocking-queue operation, or any `Condition`. **The object named is not your queue** — it is the internal condition. |
| The frame `ArrayBlockingQueue.put` | *This* is how you know which queue and which direction. Always read two frames up from `ConditionObject.await`. |
| `cpu=` a small, non-increasing value across two dumps | Confirms the thread is parked, not spinning. Compare with a livelock (Topic 98), where cpu climbs. |
| `BLOCKED` instead of `WAITING` | You are looking at a `synchronized` monitor, not a queue. Different problem, Topic 85. |

Now do the same with `take()` on an empty queue and confirm the frame reads
`ArrayBlockingQueue.take`. That pair — `put` on full, `take` on empty — is 90% of what you
will see in production dumps of a queue-based system.

### Proof 3 — `SynchronousQueue.offer` returns false with no consumer

```java
// Sync.java
import java.util.concurrent.*;
public class Sync {
    public static void main(String[] a) throws Exception {
        SynchronousQueue<String> q = new SynchronousQueue<>();
        System.out.println("offer with no consumer      -> " + q.offer("n1"));
        System.out.println("size after that offer       -> " + q.size());
        new Thread(() -> { try { System.out.println("consumer took: " + q.take()); }
                           catch (InterruptedException e) { } }, "consumer").start();
        Thread.sleep(200);                                 // let the consumer park in take()
        System.out.println("offer with a parked consumer-> " + q.offer("n2"));
    }
}
```

```bash
java Sync.java
```

| What you see | What it means |
|---|---|
| First offer `false`, size `0`, second offer `true` | Confirmed: capacity zero, hand-off only. `n1` **was destroyed** and nothing told you. |
| Both offers `true` | You have a consumer parked earlier than you thought — re-check your `sleep`. |
| `consumer took: n2` and never `n1` | The visible proof of Trap 2's silent data loss. |

The `Thread.sleep(200)` there is a teaching device, not a pattern. Never coordinate real
code with sleeps — Topic 97 gives you `CountDownLatch` for exactly this.

## Failure drill

**The point of this drill:** the bound does not merely *prevent* a failure. It **selects**
which failure you get. You are going to produce three completely different outcomes from
the same overload by changing one constructor and one method call.

### The scenario

`orderflow`'s payment-notification path, at the Topic 65 baseline, with the consumer
deliberately slowed to simulate a degraded provider.

```java
// In the notification consumer, add a configurable delay:
private void drainLoop() {
    while (!Thread.currentThread().isInterrupted()) {
        try {
            PaymentNotification n = queue.take();
            Thread.sleep(consumerDelayMs);      // simulated provider latency
            counters.delivered.increment();
        } catch (InterruptedException e) { Thread.currentThread().interrupt(); return; }
    }
}
```

Set `consumerDelayMs = 400` (a 20× provider slowdown) and run the standard k6 profile at
400 rps for ten minutes. Do it three times.

### Before you start — instrument, or the drill teaches nothing

```java
Gauge.builder("orderflow.notifications.queue.depth", queue, Collection::size).register(meters);
Counter rejected  = meters.counter("orderflow.notifications.rejected");
Counter delivered = meters.counter("orderflow.notifications.delivered");
```

And run the JVM with GC logging on, so you can see the heap story:

```bash
java -Xmx1g -Xlog:gc*:file=/tmp/bq-gc.log:time,uptime,level,tags \
     -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp/bq-oom.hprof \
     -jar orderflow.jar
```

`-Xmx1g` is deliberate: a small heap makes Run A finish in minutes rather than hours. The
failure is identical at 8 GB — it just takes longer, which is what makes it dangerous.

### Run A — unbounded, `put()`

```java
new LinkedBlockingQueue<>()      // and notifyAsync calls queue.put(n)
```

**Capture, in this order:**

```bash
# every 10s while the run proceeds:
curl -s localhost:8080/actuator/metrics/orderflow.notifications.queue.depth
curl -s localhost:8080/actuator/metrics/jvm.memory.used?tag=area:heap

# once, mid-run:
jcmd <pid> Thread.print -l > /tmp/runA-threads.txt

# after the OOM:
ls -la /tmp/bq-oom.hprof
grep -c "Pause Full" /tmp/bq-gc.log
```

**WHAT TO LOOK FOR:**

| Observation | What it means |
|---|---|
| `POST /orders` p99 unchanged for most of the run | The producer never blocks. **The service is lying about its health.** |
| queue depth rising roughly linearly at (arrival − drain) per second | The definition of an unbounded queue absorbing an imbalance. |
| heap used rising monotonically, never sawtoothing back | The queue's contents are being promoted out of Eden. Topic 68's lifetime argument, live. |
| `Pause Full` count rising near the end | The collector is trying and failing to reclaim a live set that is genuinely live. |
| `OutOfMemoryError: Java heap space` and a heap dump on disk | Run A's conclusion. |
| In MAT (Topic 79): `LinkedBlockingQueue` dominating the retained-size tree | **Write this screenshot down.** It is the single most convincing artefact you will produce in Phase 9. |

### Run B — bounded 1000, `put()`

```java
new ArrayBlockingQueue<>(1000)   // notifyAsync still calls queue.put(n)
```

**Capture:**

```bash
jcmd <pid> Thread.print -l > /tmp/runB-threads.txt
grep -c "ArrayBlockingQueue.put" /tmp/runB-threads.txt
grep -c "http-nio" /tmp/runB-threads.txt
```

**WHAT TO LOOK FOR:**

| Observation | What it means |
|---|---|
| queue depth rises to 1000 and **stops** | The bound is enforced. Heap is flat. Run A's failure is gone. |
| `POST /orders` p99 climbs sharply once depth hits 1000 | Backpressure propagated to the producer. **This is correct behaviour.** |
| Many `http-nio-8080-exec-*` threads in `WAITING (parking)` at `ArrayBlockingQueue.put` | Trap 4, reproduced. Count them against Tomcat's `maxThreads`. |
| **`GET /products` latency also degrading** | The critical observation. An unrelated endpoint is failing because request threads are a shared resource. |
| Eventually 503s from the load balancer with the JVM completely healthy | You traded an OOM for an outage. **Better** — it is recoverable and the data survives — but not yet right. |

### Run C — bounded 1000, `offer(timeout)` + outbox

```java
new ArrayBlockingQueue<>(1000)
// notifyAsync: offer(n, 50, MILLISECONDS); on false -> rejected.increment(); outbox.persist(n);
```

**WHAT TO LOOK FOR:**

| Observation | What it means |
|---|---|
| depth pinned at 1000, heap flat | Bound working. |
| `POST /orders` p99 rises by ≤ 50 ms and no more | The wait is bounded by a number you chose from the SLO. |
| `orderflow.notifications.rejected` climbing at ≈ (arrival − drain) per second | **The overflow is now a number on a dashboard.** This is the metric you alert on. |
| `GET /products` unaffected | Blast radius contained. |
| rows accumulating in the outbox table | Nothing lost. |
| after you restore `consumerDelayMs = 20`: depth drains, rejections stop, outbox replays | Full recovery with zero manual intervention. |

### Write this down before reading on

One sentence each, in your own words: (1) in Run A, what was the *first* observable
signal, and how long before the OOM did it appear? (2) in Run B, name the resource that
was exhausted — it was not the queue. (3) in Run C, why is a rising `rejected` counter a
*success* condition? (4) which single number would make Run C's rejections start earlier,
and what would that buy?

### What the fix proves

- An unbounded queue is not "no limit". It is a limit set to *available heap*, discovered
  at the worst moment, with total data loss as the failure mode.
- A bound converts an OOM into blocking. That is progress, not a solution.
- A bound *plus a bounded wait* plus *somewhere for the overflow to go* converts blocking
  into a visible, attributable, recoverable rejection.
- **The capacity number and the timeout number are both derived from the SLO.** If you
  cannot state where a queue's capacity came from, it did not come from anywhere.

---

## Measurement

### The instrument for each claim

| Claim you want to make | The right instrument | The wrong instrument |
|---|---|---|
| "the queue is backing up" | Micrometer gauge on `size()` / `remainingCapacity()` (Topic 118) | logs; a heap dump; intuition |
| "we are shedding load" | a `Counter` incremented in the `offer`-returned-false branch | absence of errors — shedding is silent by default |
| "consumers are parked, not working" | `jcmd <pid> Thread.print`, count frames at `.take` | CPU graphs — a parked thread uses none, and neither does a healthy idle one |
| "the queue is causing GC pressure" | `-Xlog:gc*` old-gen occupancy trend + a heap dump dominator tree | allocation counters alone |
| "implementation X is faster here" | JMH `@Group` producer/consumer harness with `-prof gc` | `System.nanoTime()` around a loop — Topic 77 |
| "a thread has been stuck for minutes" | **three** thread dumps ten seconds apart, diffed | one dump — it shows state, not whether state is changing |
| "the consumer thread died" | a gauge on live consumer count + an `UncaughtExceptionHandler` | nothing; this failure is silent by construction |

### Micrometer — the two gauges and one counter you always add

Forward-reference to Topic 118; this is the minimum.

```java
Gauge.builder("orderflow.notifications.queue.depth", queue, Collection::size)
     .description("current items buffered")
     .register(registry);

Gauge.builder("orderflow.notifications.queue.remaining", queue, BlockingQueue::remainingCapacity)
     .description("headroom before shedding begins")
     .register(registry);

Counter rejected = Counter.builder("orderflow.notifications.rejected")
     .description("items that could not be queued within the offer timeout")
     .register(registry);
```

**Why `remainingCapacity` and not just depth:** depth alone is meaningless without the
bound, and the bound lives in a constructor no dashboard can see. Panel
`depth / (depth + remaining)` as utilisation; alert on **>0.8 for one minute** and on **any
non-zero rejection rate**.

**Two warnings** (Topic 118 in full): never tag these with an order or product id, and
note that `LinkedBlockingQueue.size()` is an `AtomicInteger.get()` — cheap — while
`ConcurrentLinkedQueue.size()` is **O(n)**, so gauging the latter is itself a cost.

If your executor owns the queue you get this for one line —
`ExecutorServiceMetrics.monitor(registry, executor, "payment-notify")`, giving
`executor.queued`, `executor.queue.remaining`, `executor.active`, `executor.completed`.
Its absence is exactly what made Run A invisible.

### `jcmd Thread.print` — the commands you will actually type

```bash
# take three, ten seconds apart. One dump gives state; three give change.
for i in 1 2 3; do jcmd <pid> Thread.print -l > /tmp/d$i.txt; sleep 10; done

# state histogram — the first thing to look at, always:
grep "java.lang.Thread.State" /tmp/d1.txt | sort | uniq -c | sort -rn

# who is parked on a queue, and in which direction:
grep -E "BlockingQueue\.(put|take|offer|poll)" /tmp/d1.txt | sort | uniq -c

# are the request threads the ones blocked? (Trap 4's signature)
grep -B 6 "ArrayBlockingQueue.put" /tmp/d1.txt | grep '"http-nio'

# did anything move between dumps?
diff <(grep -A1 '^"' /tmp/d1.txt) <(grep -A1 '^"' /tmp/d3.txt) | head -40
```

| What you see | What it means |
|---|---|
| N threads at `...take` and depth is 0 | Healthy idle consumers. Not a problem. Ignore. |
| N threads at `...take` and depth is at capacity | **A hang.** The consumers are parked but there is work. Signal lost, or the drain loop is not the thing parked. Read the full stacks. |
| Threads at `...put` whose names are `http-nio-*` | Trap 4. Your request pool is absorbing backpressure. |
| Threads at `...put` whose names are your feeder threads | Working as designed. |
| Identical stacks across all three dumps, `cpu=` not increasing | Genuinely stuck, not slow. |
| Identical stacks, `cpu=` increasing fast | Not stuck — spinning. Livelock, Topic 98. |
| Zero `BLOCKED` threads | Expected. Queues use `Condition`s, which park. `BLOCKED` means `synchronized`. |

### JFR — for the questions a dump cannot answer

```bash
java -XX:StartFlightRecording=duration=120s,filename=/tmp/bq.jfr,settings=profile \
     -jar orderflow.jar

jfr summary /tmp/bq.jfr
jfr print --events jdk.JavaMonitorWait /tmp/bq.jfr | head -60
jfr print --events jdk.ThreadPark      /tmp/bq.jfr | head -60
jfr print --events jdk.ObjectAllocationSample /tmp/bq.jfr | grep -i node | head
```

**WHAT TO LOOK FOR:** `jdk.ThreadPark` is the event for `LockSupport.park`, which is what
every blocking-queue wait becomes. Aggregate by the parked-on class and by thread name.
A large total parked *duration* on your producer threads is Trap 4 quantified in
milliseconds. `jdk.ObjectAllocationSample` showing `LinkedBlockingQueue$Node` high in the
profile is the allocation cost of the linked design, measured rather than assumed.

### The standing rule: a naive `System.nanoTime()` loop is wrong

Restated because it applies to every "which queue is faster" question you will be asked:

```java
// WRONG. Do not do this. It will produce a number, and the number will be false.
long t0 = System.nanoTime();
for (int i = 0; i < 1_000_000; i++) { q.put(i); q.take(); }
System.out.println((System.nanoTime() - t0) / 1_000_000);
```

Four independent reasons it lies, all from Topic 77: the first ~10,000 iterations run
interpreted and then in C1 before C2 compiles anything; C2 may eliminate the loop
entirely once it proves the results unused; a single-threaded put-then-take never
exercises the contention that is the *entire* difference between these implementations;
and one run gives you no variance, so you cannot tell a 5% difference from noise. **Use
JMH with `@Group`. Always.** Forward-reference: Topic 77.

### `perf` — and the honest note about macOS

On Linux you would corroborate a contention hypothesis with hardware counters:

```bash
# Linux only:
perf stat -e cache-misses,cache-references,context-switches,cpu-migrations \
     -p <pid> -- sleep 30
```

**On macOS this is unavailable.** `perf` is a Linux kernel facility; it does not exist on
Darwin, and Apple Silicon does not expose the same counters to userspace. There is no
equivalent command and I will not invent one. Your options:

- Run the JVM in a Linux container (`--privileged` or `--cap-add=PERFMON`) and run `perf`
  inside it — the standard approach, and what your CI can do.
- Use `xcrun xctrace record --template 'Time Profiler'` for CPU sampling on macOS, which
  answers "where is time going" but **not** "how many cache misses".
- For queue work specifically, JFR's `jdk.ThreadPark` durations and JMH's `-prof gc` answer
  almost everything you need without hardware counters at all.

Topic 96 needs cache counters far more than this topic does, and repeats this note.

---

## Practice exercises

### 1 — Easy: build the four-quadrant reference from your own JVM

Write a single class that, for `ArrayBlockingQueue(2)` filled to capacity, calls each of
`add`, `offer`, `offer(1ms)` and (in a separate thread you then interrupt) `put`, and
prints exactly what each one did. Do the same for the empty case with `remove`, `poll`,
`poll(1ms)` and `take`.

**Deliverable:** the table from the "What is this?" section, regenerated from your own
output, with the exact exception types and messages your JDK produces.

**Why it is worth the twenty minutes:** you will remember `IllegalStateException("Queue
full")` because you caused it, and you will never again call `add` on a bounded queue by
accident.

### 2 — Medium: a metered, bounded, shutdown-clean pipeline

Combines Topics 01 (boxing), 13 (`equals`/`hashCode`), 39 (constructor injection),
79 (retention), 85 (monitors), 87 (`volatile`), 90 (pool sizing), 92 (`ConcurrentHashMap`).

Build `InventoryDecrementPipeline`:

- A bounded `ArrayBlockingQueue<Decrement>` where `Decrement` is a **record** of
  `(String sku, int qty, long orderId)`. Do not use `Integer` keys anywhere in the hot
  path — justify that choice in a comment referencing Topic 01's boxing allocation.
- A `ThreadPoolExecutor` with a named thread factory, `corePoolSize` derived from Little's
  Law against a stated arrival rate and service time. Write the arithmetic in a comment.
- A `ConcurrentHashMap<String, LongAdder>` accumulating per-SKU decrements, updated with
  `computeIfAbsent` — not `get`-then-`put` (Topic 92's check-then-act).
- Micrometer gauges for depth and remaining capacity, and a counter for rejections.
- `@PreDestroy` that calls `shutdown()`, waits, then `shutdownNow()`, and **proves** all
  consumer threads exited by asserting on `awaitTermination` returning `true`.
- A JUnit test that submits 100,000 decrements from 8 producer threads against a
  capacity-64 queue and asserts the final per-SKU totals are exactly correct.

**The assertion that makes it a real exercise:** after shutdown, assert that
`Thread.getAllStackTraces()` contains **zero** threads whose name starts with your
prefix. If that assertion fails you have Trap 3, and you will have found it yourself.

### 3 — Hard: production simulation on the `orderflow` baseline

Run the three-configuration drill above under the real k6 profile, then extend it:

1. Add a **fourth** configuration: `LinkedBlockingQueue(1000)` instead of
   `ArrayBlockingQueue(1000)`, everything else identical to Run C. Compare producer-side
   latency distribution and allocation rate (`-prof gc` on a JMH extraction, or JFR's
   `ObjectAllocationSample` in situ). State which you would ship for `orderflow` and why —
   including the case for the one you rejected.
2. Add a **priority** requirement: notifications for orders above £500 must be delivered
   first. `PriorityBlockingQueue` is unbounded. Design a bounded priority scheme and write
   it. (Two credible answers: a `Semaphore` with `CAPACITY` permits guarding an unbounded
   `PriorityBlockingQueue`, or two `ArrayBlockingQueue`s with a consumer that polls high
   before low. Implement one, and write down the starvation risk of the second — Topic 98.)
3. Produce a one-page capacity note in the shape Topic 129 will ask for: arrival rate,
   service time, derived pool size, derived capacity **with the arithmetic shown**, offer
   timeout, overflow destination, alert thresholds, and behaviour at 2×, 5× and 20× load.

**The deliverable is the capacity note.** The code is how you check the note is true —
artefact first, code as verification, which is Phase 12's thesis arriving early.

---

## Interview questions

### Q1 — "What capacity should this queue have?"

**MID-LEVEL ANSWER.** "Depends on the workload — maybe 1,000 or 10,000. You want it big
enough to handle bursts but not so big you run out of memory. We'd tune it if we saw
problems."

**SENIOR ANSWER.** "It's a division, and it comes from the SLO. The notification promise
is 95% delivered within 5 seconds. The consumers drain at 8 threads over a 20 ms POST, so
400 per second healthy. But capacity has to be sized for the *degraded* drain rate, not
the healthy one — the queue only fills when the consumer is slow. If I'm willing to call
200 per second 'still working', then five seconds of buffer at 200 per second is 1,000
items. So capacity is 1,000, and the number in the constructor is
`SLO_SECONDS × WORST_ACCEPTABLE_DRAIN_RATE`.

Then two more numbers fall out. The offer timeout: my order p99 budget is 250 ms and the
queue must not eat more than a fifth of it, so 50 ms. And the alert threshold: utilisation
above 80% for a minute, plus any non-zero rejection rate.

And I'd say what happens when it's full, because that's the actual decision. Rejecting to
a durable outbox, because these are payment notifications and dropping them is a
reconciliation incident. If they were metrics samples I'd drop them with `offer` and not
think about it again."

**What separates them.** The mid answer treats capacity as a tuning parameter to be
adjusted empirically. The senior answer treats it as **a derived quantity with units** —
it is a time budget multiplied by a rate, and it can be wrong in a way you can prove on a
whiteboard before deploying. The senior answer also volunteers the overflow policy without
being asked, because a capacity without an overflow policy is half a design.

**Follow-up:** *"Your consumer's drain rate isn't constant. What then?"* — Then size for
the worst rate you will still call working, and let the rejection counter be your alarm
that reality left that band. And consider whether the queue should be adaptive at all:
usually not, because a queue that grows under pressure is an unbounded queue with extra
steps.

### Q2 — "`ArrayBlockingQueue` or `LinkedBlockingQueue`?"

**MID-LEVEL ANSWER.** "`LinkedBlockingQueue` is usually faster because it's linked, so it
doesn't have a fixed size and doesn't need to copy anything. I'd use that."

**SENIOR ANSWER.** "Different lock structures, and that's the whole answer.
`ArrayBlockingQueue` has one `ReentrantLock` with two `Condition`s, so a producer and a
consumer serialise against each other. `LinkedBlockingQueue` has separate `putLock` and
`takeLock` with an `AtomicInteger` count joining them, so both sides genuinely run in
parallel — that's where its throughput advantage comes from on a two-sided workload. It
pays for that with an allocation per element and a dead node per removal, so it adds to
your allocation rate.

For `orderflow`'s notification queue at 400 per second, both are far below the throughput
where the difference shows. I'd pick `ArrayBlockingQueue` for a reason that isn't
performance at all: **its constructor requires a capacity.** `LinkedBlockingQueue`'s
no-arg constructor is unbounded, and that default has taken down more services than the
lock design has ever sped up. Choosing the API that refuses to let me forget the bound is
worth more than the throughput delta at my load.

If I were at a hundred thousand per second I'd JMH both with a `@Group` producer/consumer
harness on the real hardware and let the curve decide, and I'd look at
`gc.alloc.rate.norm` alongside throughput."

**What separates them.** "Linked is faster" is a folk belief with a plausible-sounding
reason that is not the actual reason. The senior answer names the mechanism (two locks),
names the cost (allocation per node), and then — crucially — argues that at this load the
mechanism does not matter and a **safety property of the API** decides it instead. That
last move, choosing on API safety when performance is a tie, is a senior instinct.

**Follow-up:** *"When would the allocation actually matter?"* — When the queue is deep and
long-lived, because then the nodes are promoted out of Eden rather than dying there, and
Topic 68's argument flips from "allocation is cheap" to "retention is expensive".

### Q3 — "The service is hung. Walk me through diagnosing it."

**MID-LEVEL ANSWER.** "I'd check the logs, then look at CPU and memory. If nothing shows
up, restart the pod and see if it comes back."

**SENIOR ANSWER.** "First: do not restart. Restarting destroys the only evidence and
guarantees the same incident next week. Take the pod out of the load balancer if you can —
Topic 121's distinction between liveness and readiness exists precisely for this — and
leave it running.

Then three dumps, ten seconds apart: `jcmd <pid> Thread.print -l`. One dump tells me
state; three tell me whether the state is *changing*, which is the difference between
stuck and merely slow.

Then the state histogram. Four shapes, four diagnoses:

- `BLOCKED` threads in a cycle, with a `Found one Java-level deadlock` section — deadlock.
- Threads `RUNNABLE` at high CPU with the same stack in all three dumps and no progress —
  livelock.
- Threads `WAITING (parking)` in `ArrayBlockingQueue.take` — could be perfectly healthy
  idle consumers. **The dump alone cannot tell me.** I cross-reference the queue-depth
  gauge: parked in `take` with depth zero is health; parked in `take` with depth at
  capacity is a hang.
- Threads `WAITING (parking)` in `ArrayBlockingQueue.put`, and their names are
  `http-nio-*` — that is backpressure that has reached my request pool, which is why an
  unrelated endpoint is also down.

For this service the fourth shape is the one I'd expect, because I know the notification
queue is bounded and the request thread calls into it. And I'd confirm it in one grep:
`grep -B 6 'BlockingQueue.put' dump.txt | grep http-nio`."

**What separates them.** The mid answer has one hypothesis ("something is wrong") and one
tool (restart). The senior answer has a **decision procedure** with four branches, names
the specific evidence that selects each branch, and — most tellingly — knows that one
piece of evidence (`WAITING` in `take`) is **ambiguous** and names the second signal
needed to disambiguate it. Knowing which of your signals is ambiguous is the whole skill.

**Follow-up:** *"You found threads parked in `put`. Now what?"* — Short term, that's a
downstream problem, so I'd check the consumer's dependency before touching the queue.
Long term, `put` from a request thread is the design bug; it becomes `offer(timeout)` with
a durable overflow, which converts the outage into a rejection metric.

### Q4 — "How do you shut down a producer-consumer pipeline cleanly?"

**MID-LEVEL ANSWER.** "Call `shutdown()` on the executor and then `awaitTermination`. Maybe
`shutdownNow()` if it doesn't stop."

**SENIOR ANSWER.** "Two mechanisms, and you pick by whether in-flight work must complete.

**Poison pill** when you must drain: enqueue one sentinel per consumer — one, because each
pill is taken by exactly one consumer — and have the drain loop `break` on it. Consumers
finish everything ahead of the pill first. It's ordered, it's graceful, and it fails if a
producer is still running, so you have to stop producers first.

**Interruption** when you must stop now: `shutdownNow()` interrupts every worker, and a
thread parked in `take()` throws `InterruptedException`. That only works if every consumer
handles it correctly — restore the flag with `Thread.currentThread().interrupt()` and
return. The commonest bug in Java is `catch (Exception e) { log.error(...); }` around a
blocking call, which eats the interrupt, clears the flag and loops back into `take()`.
The thread is now unstoppable and your rolling deploy takes the full grace period every
time.

In production I'd do both: `shutdown()`, `awaitTermination` with a budget, then
`shutdownNow()`, then `awaitTermination` again, and **log loudly if the second one also
returns false** — because that means I have a thread that ignores interruption and I need
to know which one. And I'd assert in a test that no threads with my name prefix survive
shutdown, so the bug can't come back."

**What separates them.** The senior answer knows shutdown is a **contract requiring
cooperation from the consumer code**, not a method call that works by itself. And it names
the specific, extremely common line of code that breaks it.

**Follow-up:** *"What if the consumer is in the middle of a 30-second HTTP call when you
interrupt?"* — Interruption is cooperative and most blocking I/O does not respond to it;
a socket read will not throw. That is an argument for a timeout on the HTTP client rather
than for a different shutdown strategy, and it is why `awaitTermination` needs a budget
longer than your longest downstream timeout — or an accepted policy of killing in-flight
work.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. `ArrayBlockingQueue` uses one lock and two `Condition`s;
   `LinkedBlockingQueue` uses two locks and two `Condition`s. Explain why the array-backed
   design *cannot* use two locks, in terms of the data structure rather than the API.

2. `LinkedBlockingQueue.put` contains
   `if (c + 1 < capacity) notFull.signal()` — a producer waking another producer. Why is
   this necessary at all, given that a *consumer* also signals `notFull` when it removes an
   element? What goes wrong without it?

3. A bounded queue's capacity is a backpressure policy. Little's Law says a queue changes
   `L`, not `μ`. Reconcile those two statements: if the buffer cannot increase throughput,
   what exactly is it buying, and under what arrival pattern does it buy nothing at all?

4. `SynchronousQueue.size()` always returns `0`. Argue that this is the correct design.
   Then make the strongest case that it should have thrown
   `UnsupportedOperationException` instead, and say which failure mode each choice
   produces in a monitoring dashboard.

5. A thread parked in `take()` and a thread parked in `put()` are in the same
   `Thread.State`. Name the piece of evidence outside the thread dump that distinguishes a
   healthy idle consumer from a hung one, and explain why no amount of dump-reading can
   substitute for it.

6. Topic 90 says `ThreadPoolExecutor` grows past `corePoolSize` only when the queue is
   full. Combine that with this topic: describe the behaviour of a pool with
   `core=2, max=100, queue=LinkedBlockingQueue()` under a 10× load spike, and then the
   behaviour of `core=2, max=100, queue=SynchronousQueue()` under the same spike. One of
   them never creates a third thread. Say which and why.

---

## Quick reference card

### The methods, by failure mode

| Intent | Insert | Remove |
|---|---|---|
| "this cannot be full/empty; crash if it is" | `add(e)` → `IllegalStateException` | `remove()` → `NoSuchElementException` |
| "shed immediately if it cannot proceed" | `offer(e)` → `false` | `poll()` → `null` |
| "wait as long as it takes" | `put(e)` | `take()` |
| "wait up to a budget, then shed" | `offer(e, t, u)` → `false` | `poll(t, u)` → `null` |
| "look without removing" | — | `peek()` → `null` if empty |
| "move many at once" | — | `drainTo(c)` / `drainTo(c, max)` |

**Never ignore the `boolean` from `offer` or the `null` from `poll`.**

### The implementations, at a glance

| | `ArrayBlockingQueue` | `LinkedBlockingQueue` | `SynchronousQueue` | `LinkedTransferQueue` | `PriorityBlockingQueue` |
|---|---|---|---|---|---|
| Bounded | **always** (mandatory) | optional — **unbounded by default** | capacity 0 | no | **no, and cannot be** |
| Locks | one `ReentrantLock` | `putLock` + `takeLock` | none (CAS on a transfer stack/queue) | none (CAS) | one `ReentrantLock` |
| Allocation per element | none | one `Node` | none | one `Node` | array growth, amortised |
| FIFO | yes | yes | fair mode only | yes | **no** — priority order |
| Optional fairness | yes | no | yes | no | n/a |
| `size()` cost | O(1) | O(1) (`AtomicInteger`) | always 0 | **O(n)** | O(1) |
| Typical use | thread hand-off with a bound | high-throughput two-sided | `newCachedThreadPool`; rendezvous | when you want both semantics | scheduling by priority |

### The three numbers you must be able to justify

```
capacity       = SLO_seconds x worst_acceptable_drain_rate
offer_timeout  = a stated fraction of the caller's latency budget
pool_size      = arrival_rate x service_time        (Little's Law, Topic 90)
```

If you cannot say where each number came from, it came from nowhere.

### Gotchas checklist

- [ ] No `new LinkedBlockingQueue<>()` anywhere. No `Executors.newFixedThreadPool`.
- [ ] Every `offer` return value is checked.
- [ ] `put` is called only from dedicated feeder threads, never from a request thread.
- [ ] `InterruptedException` is caught **separately**, the flag restored, the loop exited.
- [ ] `RuntimeException` inside the drain loop cannot kill the consumer.
- [ ] Depth and remaining-capacity gauges exist; a rejection counter exists; both alert.
- [ ] Threads are named.
- [ ] Shutdown is tested — an assertion that zero named threads survive.
- [ ] The overflow has a destination, and that destination is durable if the data is.
- [ ] `SynchronousQueue` is used only where a rendezvous is intended, never as a buffer.

---

## When would I use this at work?

**1. Reviewing a PR that adds a background worker.**

Someone adds `@Async` processing with a `LinkedBlockingQueue` and a fixed pool. The diff
is clean. You ask two questions, and neither is a style comment: *"what is the queue's
capacity, and where did that number come from?"* and *"what happens to an item when the
queue is full?"* If the answers are "the default" and "I hadn't thought about it", you
have prevented Run A. The whole conversation takes ninety seconds and it is the highest
value-per-second review you will do all week.

**2. Sizing a pipeline before it exists.**

Product wants order-confirmation emails. You do not start with code. You write four lines:
arrival 400/s, provider p99 200 ms, SLO 30 s, durability required. Little's Law gives 80
consumer threads — too many, so you renegotiate the SLO or batch the sends. Capacity falls
out as `30 × drain_rate`; overflow goes to the outbox because durability was required.
**The design finished before anyone opened an editor**, and every number in the code is
traceable to a line in a requirement.

**3. The 3am page where the queue is the story.**

Notification delivery latency alert fires. You check the depth gauge: pinned at capacity.
Rejection counter: climbing. Consumer thread count: still 8, so nobody died. Consumer
latency: 20× normal. In ninety seconds you know it is a downstream degradation, not your
service, and your service is behaving exactly as designed — shedding to the outbox and
telling you about it. You page the provider's on-call instead of restarting your own pods,
and when the provider recovers the outbox replays with zero data loss. **You did not fix
anything. The design fixed it, and the instrumentation let you prove that in ninety
seconds instead of arguing about it for an hour.**

---

## Connected topics

**Prerequisites:**

- **89 — `wait`/`notify` and guarded blocks.** The mechanism inside every blocking queue:
  the `while`-loop condition check, the spurious-wakeup rule, and the wrong-waiter problem
  designed out by giving producers and consumers separate `Condition`s.
- **86 / 87 — the JMM and `volatile`.** The `BlockingQueue` contract's happens-before
  guarantee is why a mutable object handed through a queue is safely published. Without
  Topic 86 that guarantee reads as a formality; with it, it is the reason you do not need
  a single `volatile` of your own on the payload.
- **90 — executors and pool sizing.** The queue is `ThreadPoolExecutor`'s second
  constructor argument, and the executor's growth rule (*queue first, grow only on
  rejection*) is unintelligible without knowing what each queue does when it cannot accept
  work. Topic 90 sized the pool; this topic sizes the buffer in front of it.
- **94 — explicit locks and AQS.** `Condition` is an AQS construct; `await` is `park` with
  the lock released. The BLOCKED-vs-WAITING table you built there is what makes the thread
  dumps in this document readable.
- **92 — `ConcurrentHashMap`.** Both are `java.util.concurrent` collections whose per-op
  atomicity does not compose. `if (queue.size() < 100) queue.put(x)` races exactly like
  `containsKey`-then-`put` does, and for exactly the same reason.
- **65 — the load-testing gate.** The drill is meaningless without the recorded baseline.
- **68 / 79 — heap generations and leak hunting.** Run A's failure is a *retention*
  failure: the queue's contents stop being short-lived and start being promoted. The MAT
  dominator tree is what turns "we OOMed" into "the notification queue OOMed us".

**This unlocks:**

- **96 — false sharing.** Why `ArrayBlockingQueue`'s adjacent `putIndex`/`takeIndex`
  fields, and any queue's head and tail pointers, can contend physically even when
  producer and consumer never touch the same logical data. The next reason a queue can be
  slower than its lock structure predicts.
- **97 — coordination primitives.** A `Semaphore` is the other way to express "at most N
  in flight". A `BlockingQueue` bounds *buffered work*; a `Semaphore` bounds *concurrent
  work*. Knowing which of those two your constraint actually is, is the whole design
  decision in Topic 111's bulkhead.
- **98 — the concurrency bug taxonomy.** Trap 3's swallowed interrupt is a thread leak;
  Trap 5's dying consumer is a slow-motion starvation; Trap 4's parked request threads are
  the shape people misdiagnose as deadlock. Topic 98 gives you the decision table that
  keeps those four apart.
- **101 — virtual threads.** Changes the calculus of Trap 4 substantially: a virtual
  thread parked in `put` costs a heap continuation rather than an OS thread, so blocking a
  "request thread" is far cheaper. It does **not** change the argument for bounding the
  queue — heap is still finite, and the downstream's capacity has not moved.
- **105 — backpressure in Reactor.** `BlockingQueue` is backpressure by *blocking a
  thread*; `request(n)` is a demand signal travelling upstream with no thread blocked.
  `onBackpressureBuffer` / `Drop` / `Latest` are this document's `put`, `offer` and "keep
  only the newest" — the same three policies as an operator.
- **109 — the HikariCP pool deadlock.** A connection pool is a bounded queue of
  connections with `poll(timeout)` semantics; `connectionTimeout` is this document's offer
  timeout under a different name, and pool exhaustion is Run B's failure with a database
  connection as the resource.
- **111 — bulkheads and resilience.** Trap 4's fix generalised: isolate a fragile
  downstream behind its own bounded resource so its degradation cannot consume the shared
  request pool.
- **115 — the outbox pattern.** The durable destination that Fix 1's overflow branch
  writes to, and the argument in Fix 3 that a notification which must not be lost has no
  business living only in heap.
- **118 — Micrometer metrics.** The depth gauge, the remaining-capacity gauge and the
  rejection counter, done properly — including why neither may ever be tagged with an
  order id.

---

*Java baseline 21, running on JDK 25. Two things here are deliberately hedged rather than
asserted: the relative throughput of `ArrayBlockingQueue` versus `LinkedBlockingQueue` on
your hardware, which only your own JMH `@Group` run can settle; and the per-element cost of
a linked queue under `[JAVA 25]` compact object headers, which JOL settles in one command.
Everything else — that the no-arg `LinkedBlockingQueue` constructor is unbounded, that
`SynchronousQueue` has capacity zero, that `put` and `take` produce `WAITING (parking)` on
a `ConditionObject`, and that a queue's capacity is a latency budget times a drain rate —
is stable platform behaviour, and it is what you will need at 3am when the depth gauge is
pinned and someone is asking whether to restart the pod.*
