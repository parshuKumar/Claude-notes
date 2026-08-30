# 89 — `wait`/`notify`, Guarded Blocks, and Spurious Wakeups

## Phase: 9 — Concurrency
## Category: DIFFERENTIATOR
## Java baseline: 21  |  Notes features from: 21 (with a JDK 24/25 virtual-thread note you must verify locally)
## Project spine: the `orderflow` reservation hand-off buffer — the queue that sits between the HTTP request threads that accept an order and the background workers that reserve inventory. You will build it with `wait`/`notify` in this topic, break it two ways, and then delete it in Topic 93 in favour of `BlockingQueue`.

---

## Mechanical statement

Read this until you can recite it. Everything else is elaboration.

> **`wait()` atomically releases the monitor and parks the calling thread. On waking,
> it must re-acquire that monitor before it can return.** The JLS explicitly permits
> **spurious wakeups** — a thread may return from `wait()` with nobody having called
> `notify()`, without an interrupt, and without the timeout having elapsed.
>
> **Therefore the condition you are waiting on must be re-tested in a `while` loop.
> An `if` is a bug by specification, not a bug by bad luck.**

Three sub-facts fall straight out of that sentence, and each one is a production bug
if you get it wrong:

1. **You must hold the monitor to call `wait()`.** Calling it without holding the
   monitor throws `IllegalMonitorStateException` immediately — this is the one failure
   in this topic that is loud.
2. **`wait()` releases exactly one monitor: the one you called it on.** Any *other*
   lock the thread holds stays held. That is how a thread parked in `wait()` can still
   deadlock the rest of the system.
3. **Returning from `wait()` tells you nothing.** It is not a message. It carries no
   payload. It does not mean "the thing you wanted happened". It means "you are
   runnable again; go and look at the shared state yourself."

That third point is the one that catches engineers arriving from Promises, and it is
worth naming now: **`notify()` is not `resolve()`.** A resolved Promise carries a
value to exactly the callbacks registered on it. `notify()` carries nothing to an
unspecified one of possibly several waiters.

---

## The bridge from what you know

### What you already have that is close

You have written this exact shape in Node, many times:

```ts
// A promise you can resolve from elsewhere — a "deferred".
let releaseSlot!: () => void;
const slotAvailable = new Promise<void>(res => { releaseSlot = res; });

async function reserve(): Promise<void> {
  if (freeSlots === 0) {
    await slotAvailable;      // park this logical task
  }
  freeSlots--;
}
```

That is a guarded wait. You blocked a *task* until a condition became true, and
someone else made it true and signalled. The intent is identical to
`wait`/`notify`.

**Verdict: PARTIAL analogue — and every one of the four differences below is a bug
you will write once.**

### Where it breaks — four differences, all load-bearing

**1. `await` yields the event loop. `wait()` parks an OS thread.**

`await slotAvailable` costs you a continuation on the heap and lets the single thread
go and do a thousand other things. `buffer.wait()` costs you a real OS thread, its
stack (typically 1 MB of reserved address space on a 64-bit JVM, sized by `-Xss`), and
a trip through the kernel scheduler. Ten thousand awaiting tasks in Node is a normal
Tuesday. Ten thousand threads in `wait()` is a JVM you cannot afford — which is the
entire reason Topics 90 and 101 exist.

**2. A resolved Promise stays resolved. A `notify()` that nobody is waiting for is lost forever.**

This is the single most important difference in the topic.

```ts
releaseSlot();                     // resolve NOW
await slotAvailable;               // later — still returns immediately. The value persists.
```

```java
synchronized (buffer) { buffer.notify(); }   // nobody is waiting: this does nothing at all
// ...later...
synchronized (buffer) { buffer.wait(); }     // parks forever. The signal is gone.
```

A Promise is a **latch with memory**. `notify()` is an **edge**. If no thread is in
the wait set at the instant you call it, the signal evaporates. There is no queue of
pending notifications. This is the *missed signal* / *lost wakeup* bug, and it is the
headline trace in this document.

**3. `await` never returns spuriously. `wait()` may.**

There is no equivalent in JavaScript of "your `await` resumed for no reason and the
value is not there". The JLS permits it. The `Object.wait` javadoc names it. This is
why you write `while`, not `if`.

**4. Promises target their own callbacks. `notify()` targets an arbitrary waiter.**

`resolve()` on a specific Promise wakes exactly the continuations registered on that
Promise. `notify()` on a monitor wakes **one unspecified thread** from that monitor's
wait set — and if your producers and your consumers wait on the *same* monitor, it may
wake a thread whose condition is still false. That thread re-checks, sees false, and
waits again. Meanwhile the thread that could have made progress was never woken. The
system stalls with no exception, no error and no CPU usage.

### The verdict table

| You know | Java | Verdict |
|---|---|---|
| `await promise` | `obj.wait()` inside `synchronized (obj)` | **PARTIAL** — same intent, different cost, different memory semantics |
| `resolve(v)` | `obj.notify()` | **PARTIAL** — a Promise remembers; `notify()` is an edge that can be missed |
| `Promise.all` fan-in | `notifyAll()` + a counter | **PARTIAL** — you must maintain the counter yourself |
| A Promise carries a value | `notify()` carries nothing | **NO ANALOGUE** — you must read shared state after waking |
| An `await` never resumes without a reason | spurious wakeups are legal | **NO ANALOGUE** — this is why `while` is mandatory |
| Node warns on unhandled rejection | nothing warns on a lost wakeup | **NO ANALOGUE** — silence is the failure mode |

### Building on Topics 85–88 — the vocabulary you already have

You already know from Topic 85 that every object has a **monitor** and that
`synchronized` acquires it. You know from Topic 86 that a monitor **unlock
happens-before** a subsequent lock of the same monitor. You know from Topic 87 that
visibility is not automatic.

`wait`/`notify` is built entirely out of those three facts. `wait()` performs a monitor
**unlock**, publishing everything the waiting thread wrote; re-acquiring on wake
performs a monitor **lock**, so the woken thread sees everything every previous holder
wrote. **Therefore guarded state needs no `volatile`.** Adding `volatile` to a field
only ever touched inside `synchronized (thisMonitor)` is noise, and it tells a
reviewer you do not know why the code is correct.

---

## What is this?

A **guarded block** is the general pattern:

> "Do not proceed until some shared condition is true. While it is false, give up the
> CPU and the lock, and let someone else make it true."

Java's oldest implementation of that pattern is three methods on `java.lang.Object` —
which means every object in the language has them:

```java
public final void wait() throws InterruptedException;
public final void wait(long timeoutMillis) throws InterruptedException;
public final void wait(long timeoutMillis, int nanos) throws InterruptedException;
public final void notify();
public final void notifyAll();
```

They are on `Object` because in Java's original design every object could be a lock,
and a lock is useless for coordination unless you can also wait on it.

### The correct skeleton — memorise this shape

```java
// WAITER
synchronized (lock) {
    while (!conditionIsTrue()) {   // WHILE. Never if.
        lock.wait();               // releases `lock`, parks, re-acquires before returning
    }
    // Here: you hold `lock` AND the condition was true when you last checked it,
    // and nothing could have changed it since, because you have held `lock` throughout.
    doTheThing();
}

// SIGNALLER
synchronized (lock) {
    makeConditionTrue();
    lock.notifyAll();              // notifyAll by default. notify only after an argument.
}
```

Four rules, and they are not negotiable:

| Rule | Why |
|---|---|
| Wait inside `synchronized` on the same object you call `wait()` on | Otherwise `IllegalMonitorStateException` |
| Always `while`, never `if` | Spurious wakeups are permitted by the spec |
| Change the state **and** signal inside the same `synchronized` block | Otherwise the state change and the signal can interleave with a waiter's check — that is the lost wakeup |
| Prefer `notifyAll()` | `notify()` is only correct under conditions most code does not meet, listed below |

### `notify()` vs `notifyAll()` — the exact rule

`notify()` wakes **one** thread from the wait set. Which one is unspecified — it is
not "the longest waiting", it is not FIFO, and you must not depend on it.

`notify()` is only safe when **all three** of these hold:

1. **Uniform waiters.** Every thread in the wait set is waiting for the *same*
   condition, so waking any one of them makes progress.
2. **One-in, one-out.** Each signal enables exactly one waiter to proceed.
3. **No conditional consumption.** A woken thread that finds its condition true always
   consumes the thing that was signalled — it never wakes, decides not to proceed, and
   waits again, having absorbed a signal it did not use.

A bounded buffer with both producers and consumers waiting on the same monitor
**violates rule 1**. That is the second headline trap in this document, and it
deadlocks a live system with zero CPU usage.

`notifyAll()` is O(waiters) in wakeups and creates a thundering herd where all but one
immediately re-park. That cost is real but it is measured in microseconds. Correctness
first. If a profiler shows the herd is your bottleneck — which it will not, at
`orderflow`'s scale — the fix is `ReentrantLock` with two `Condition` objects
(Topic 94), not `notify()`.

---

## Why does it matter?

You will almost never write `wait`/`notify` in application code. So why is this a
DIFFERENTIATOR topic rather than a footnote?

**1. Everything above it is built on it.** `ArrayBlockingQueue`, `LinkedBlockingQueue`,
`CountDownLatch`, `Semaphore`, `ThreadPoolExecutor`'s worker parking, HikariCP's
connection borrow, Hibernate's session-factory init latches — all of them are guarded
blocks. `Condition.await`/`signal` (Topic 94) is the same mechanism with a separate
wait set per condition. If you do not understand the primitive, every one of those is
magic, and magic cannot be debugged at 2am.

**2. It is the shape of every "our service just stopped" incident.** A thread parked
in `Object.wait()` consumes no CPU, produces no log line, and throws nothing. The
service looks healthy to a naive liveness probe. Throughput is zero. The only evidence
is a thread dump, and reading a thread dump means recognising `in Object.wait()` and
knowing what it implies about the wait set.

**3. It is where the "silent failure" instinct gets installed.** Coming from Node, your
diagnostic reflex is "find the exception, read the stack". Java concurrency's
characteristic failure is the *absence* of an event. Nothing threw. Nothing logged.
Work simply stopped happening. That reflex change is worth more than the API.

**4. Interviewers use it as a filter.** "Why `while` and not `if`?" separates people
who have read the spec from people who have copied a pattern. It takes ten seconds to
ask and it is very hard to bluff.

---

## Machine-level reality

### Two sets, not one

Every Java object header can be inflated into a **monitor** (Topic 85: the mark word
CASes to a thin lock when uncontended, and inflates to a fat `ObjectMonitor` when
contended or when `wait()` is called). A fat monitor holds **two distinct queues**:

| Queue | HotSpot name | Who is in it | What gets them out |
|---|---|---|---|
| **Entry set** | `_EntryList` / `_cxq` | Threads blocked trying to *acquire* the monitor | The owner releasing it |
| **Wait set** | `_WaitSet` | Threads that called `wait()` on it | `notify` / `notifyAll` / timeout / interrupt |

This two-queue structure is the whole mechanism. Draw it once and you never confuse
`BLOCKED` with `WAITING` again.

**Thread state mapping — this is what you read in a thread dump:**

| Java thread state | Where the thread is |
|---|---|
| `BLOCKED (on object monitor)` | In the **entry set**. Trying to acquire. Someone else owns it. |
| `WAITING (on object monitor)` | In the **wait set**, after `wait()` with no timeout. |
| `TIMED_WAITING (on object monitor)` | In the **wait set**, after `wait(millis)`. |

A thread that is notified moves from the wait set to the **entry set** — it is now
`BLOCKED`, competing for the lock like any other acquirer. It does not jump the queue.
That transition is why `wait()` can return "late": being notified is not the same as
running.

### The exact sequence `wait()` performs

Conceptually, `obj.wait()` does this, and the first two steps are atomic with respect
to `notify()`:

1. Assert the current thread owns `obj`'s monitor. If not, throw
   `IllegalMonitorStateException`.
2. **Save the recursion count.** If you entered `synchronized` three times nested,
   `wait()` remembers 3.
3. Add this thread to `_WaitSet`.
4. **Fully release the monitor** — all three levels, not one. Any thread in the entry
   set can now acquire it.
5. `park()` the thread — a `pthread_cond_wait` / futex wait on POSIX, which takes it
   off the run queue entirely. Zero CPU.
6. On wake (notify, notifyAll, timeout, interrupt, or spurious): move to the entry set
   and **re-acquire the monitor, restoring the saved recursion count**.
7. Only then return from `wait()`.

**Step 4 is the reason `wait()` cannot be replaced by `Thread.sleep()`.** `sleep()`
holds every lock it has. A thread sleeping inside `synchronized` guarantees nobody can
change the condition it is sleeping for. That is a deadlock you built by hand.

**Step 6 is the reason `wait()` can be slow to return under contention** — a notified
thread still has to win the lock.

### `park`/`unpark` — the layer underneath

`wait()` and `notify()` ultimately sit on the same primitive that
`java.util.concurrent` uses directly:

```java
java.util.concurrent.locks.LockSupport.park(Object blocker);
java.util.concurrent.locks.LockSupport.unpark(Thread t);
```

`park`/`unpark` has one property that `wait`/`notify` does not, and it is worth
knowing because it explains a lot of `j.u.c` code:

> **`unpark` has a one-permit memory.** An `unpark` delivered *before* the target
> parks makes the subsequent `park` return immediately. `notify` has no such memory.

That is exactly the missed-signal hazard, solved one layer down. `j.u.c` is built on
`park`/`unpark` rather than `wait`/`notify` partly for that reason, and partly because
`park` is not tied to a monitor at all.

`park` is also permitted to return spuriously. Same rule: loop and re-check.

### What a parked thread looks like in a thread dump

*Illustration of the FORMAT, not captured output. Every `<n>` is a placeholder for a
value you will see on your own machine.*

```
"orderflow-reservation-worker-<n>" #<n> [<n>] prio=5 os_prio=0 cpu=<n>ms
    tid=0x<n> nid=0x<n> in Object.wait()  [0x<n>]
   java.lang.Thread.State: WAITING (on object monitor)
        at java.base/java.lang.Object.wait0(Native Method)
        - waiting on <0x<n>> (a com.orderflow.inventory.ReservationBuffer)
        at java.base/java.lang.Object.wait(Object.java:<n>)
        at com.orderflow.inventory.ReservationBuffer.take(ReservationBuffer.java:<n>)
        - locked <0x<n>> (a com.orderflow.inventory.ReservationBuffer)
        at com.orderflow.inventory.ReservationWorker.run(ReservationWorker.java:<n>)
```

**The four fields that matter, and how to read them:**

| Field | What it tells you |
|---|---|
| `java.lang.Thread.State: WAITING (on object monitor)` | This thread is in a **wait set**. It is not stuck on a lock; it is waiting for a condition. |
| `- waiting on <0x…> (a com.orderflow…ReservationBuffer)` | **The monitor identity.** This hex address is how you match waiters to signallers. |
| `- locked <0x…> (a …ReservationBuffer)` **below** the `waiting on` line | Standard and initially confusing: the dump records the frame that owns the monitor. The thread has *released* it while parked; it will re-acquire before returning. Do not read this as "holding a lock while waiting". |
| `cpu=<n>ms` not increasing between two dumps | Confirms it is genuinely parked, not spinning. |

**The diagnostic move:** take two dumps 30 seconds apart. If the same threads are
`WAITING` on the same `0x` address in both, and no thread anywhere is running your
signalling code, you have a lost wakeup. That is a two-minute diagnosis once you know
the shape.

### Where spurious wakeups actually come from

They are not folklore. Three real sources:

1. **POSIX permits it.** `pthread_cond_wait` may return without a signal — classically
   because a process signal interrupts the futex wait. The JDK passes that property
   through rather than paying to hide it.
2. **`notifyAll` plus a lost race.** A is notified, moves to the entry set; before A
   acquires the lock, B acquires it and consumes the item; A then acquires and finds
   the condition false. Indistinguishable from a spurious wakeup, and *far* more
   common in practice than the OS-level kind.
3. **Interruption handling and internal JVM bookkeeping** can surface as an early
   return in some implementations.

Cause 2 alone means the `while` loop is not defence against an exotic kernel event —
it is the normal, expected path in any multi-consumer buffer.

**Confirm the specification claim yourself:**

```bash
# The JLS/Javadoc statement, straight from the JDK source shipped with your JVM.
unzip -p "$JAVA_HOME/lib/src.zip" java.base/java/lang/Object.java \
  | grep -n -A4 -i "spurious"
```

**What to look for:** the javadoc paragraph on `wait` describing a thread waking
"without being notified, interrupted, or timing out", and the sentence recommending
that waits always occur in loops. If `src.zip` is absent, your JDK was installed
without sources — install them, or read the same text at the `java.lang.Object`
javadoc for your exact JDK version. Do not take my word for it; this is the one fact
in the topic that the entire `while` rule rests on.

### Virtual threads — one line, and verify it

Through **JDK 21**, a virtual thread that blocks in `Object.wait()` or on
`synchronized` entry **pins its carrier thread**: the carrier cannot be reused while
the virtual thread is parked. **JEP 491, delivered in JDK 24**, changed `synchronized`
and `Object.wait()` so a virtual thread unmounts instead of pinning.

I am not going to assert which behaviour your JDK has. Settle it:

```bash
java --version
# Then, on JDK 21-23 specifically, this flag reports pinning events:
java -Djdk.tracePinnedThreads=full YourApp
# On JDK 21+, the JFR event is the durable answer (see Measurement below):
jfr summary recording.jfr | grep -i pinned
```

**What to look for:** on JDK 21–23 you will see stack traces headed by a pinned-frame
report naming `Object.wait` or a `synchronized` frame. On JDK 24+ those events should
be absent for `synchronized`/`wait` while still appearing for native frames. This is
Topic 101's subject in full; here it is a one-line caveat with a command attached.

---

## Concurrency trace

**The headline: the missed signal (lost wakeup).**

Setup: `orderflow` accepts an order on an HTTP request thread and hands the
reservation off to a background worker through a shared buffer. Someone wrote the
producer so that the state change and the signal are in **separate** `synchronized`
blocks — which looks harmless, and reads fine in review.

```java
// PRODUCER — the version with the bug
public void submit(Reservation r) {
    synchronized (this) {
        queue.add(r);                 // state change ... block ends, lock released
    }
    synchronized (this) {
        notify();                     // ... signal in a SEPARATE block
    }
}

// CONSUMER — also with the classic `if` bug, to show both at once
public Reservation take() throws InterruptedException {
    synchronized (this) {
        if (queue.isEmpty()) {        // IF, not WHILE
            wait();
        }
        return queue.poll();          // may return null
    }
}
```

There is exactly one item to reserve: SKU-4471, one unit, for order `ORD-88213`.

| Step | Thread A — HTTP request thread (`submit`) | Thread B — reservation worker (`take`) | Shared state / outcome |
|---|---|---|---|
| 1 | — | enters `synchronized (this)`, **owns monitor** | queue: `[]` |
| 2 | — | `queue.isEmpty()` → **true** | queue: `[]` |
| 3 | — | about to call `wait()` — **preempted here** (Topic 84: the scheduler may suspend between any two bytecodes) | B owns the monitor, is not yet in the wait set |
| 4 | blocks entering `synchronized (this)` — B owns it | — | A is in the **entry set** |
| 5 | — | calls `wait()`: releases monitor, enters **wait set** | wait set: `{B}` |
| 6 | acquires monitor, `queue.add(ORD-88213)` | parked | queue: `[ORD-88213]` |
| 7 | **exits the first `synchronized` block — releases the monitor** | parked | wait set: `{B}`, queue: `[ORD-88213]` |
| 8 | — | **spurious wakeup.** B leaves the wait set, re-acquires the monitor, returns from `wait()` | B owns the monitor again |
| 9 | blocks entering the **second** `synchronized` block — B owns it | `if` already evaluated at step 2. **Not re-checked.** Falls straight through | — |
| 10 | — | `queue.poll()` → `ORD-88213`, exits block | queue: `[]`, B holds `ORD-88213` |
| 11 | acquires monitor, calls `notify()` — **wait set is empty** | working on the reservation | **the signal evaporates** |
| 12 | second order `ORD-88214` arrives, `queue.add`, `notify()` — but B is still busy and not waiting | still working | **that signal evaporates too** |
| 13 | — | B finishes, loops, enters `take()`, `queue.isEmpty()` → **false**, polls `ORD-88214` | recovers by luck: the item was already there |
| 14 | no more orders arrive for 40 seconds | enters `take()`, `queue.isEmpty()` → true, calls `wait()` | wait set: `{B}` |
| 15 | order `ORD-88215` arrives: `queue.add` in block 1, lock released, **thread A is descheduled before block 2** | parked | queue: `[ORD-88215]`, nobody signalled |
| 16 | A is descheduled long enough that the request times out and the thread is recycled by Tomcat **before reaching the `notify()`** | parked | **signal never sent** |
| 17 | — | still parked. Zero CPU. No log line. No exception. | **`ORD-88215` is accepted, the customer sees "order placed", and the inventory is never reserved.** Stock is oversold at the next replenishment reconciliation, and the order sits in `PENDING_RESERVATION` until a human notices. |

Read step 3 and step 9 again. Two separate defects fired in one trace:

- **Steps 3–11 are the `if` bug.** B checked the condition, was preempted, woke
  spuriously, and never re-checked. Here it got lucky — the item happened to be there.
  Change the ordering slightly and `queue.poll()` returns `null`, and you get a
  `NullPointerException` in a worker thread with a stack trace that makes no sense,
  because the queue "obviously" was non-empty.
- **Steps 15–17 are the lost wakeup.** The state change and the signal were in
  different critical sections, so a waiter could slot in between them. The correct
  code puts `add` and `notifyAll` in the *same* `synchronized` block, which makes that
  interleaving impossible.

**The business outcome:** the customer is told the order is placed. Inventory is never
decremented. The oversell surfaces days later during reconciliation, by which time you
have shipped stock you do not have.

**Now — and only now — here is the correct code.**

```java
public void submit(Reservation r) {
    synchronized (this) {
        queue.add(r);
        notifyAll();          // SAME critical section as the state change
    }
}

public Reservation take() throws InterruptedException {
    synchronized (this) {
        while (queue.isEmpty()) {   // WHILE
            wait();
        }
        return queue.poll();        // cannot be null: we hold the lock and just checked
    }
}
```

Two edits. Both are one word. Neither produces a test failure if you get it wrong.

---

## Example 1 — minimal

A one-shot readiness gate: the `orderflow` context finishes loading the product
pricing table, and worker threads must not price anything until it is loaded.

```java
package com.orderflow.pricing;

/**
 * Minimal guarded block. One boolean, many waiters, one signaller.
 * This is the simplest correct use of wait/notify that exists.
 */
public final class PricingTableGate {

    private final Object lock = new Object();   // a private lock object, not `this`
    private boolean loaded = false;

    /** Called by every worker before it prices anything. Blocks until loaded. */
    public void awaitLoaded() throws InterruptedException {
        synchronized (lock) {
            while (!loaded) {          // WHILE: re-checks after every wake, spurious or not
                lock.wait();
            }
        }
        // Past this point `loaded` is true and can never become false again.
        // Note we do NOT hold the lock here — that is deliberate and safe only because
        // this gate is one-shot. See "Why a private lock object" below.
    }

    /** Called once, by the startup thread, when the table is in memory. */
    public void markLoaded() {
        synchronized (lock) {
            loaded = true;
            lock.notifyAll();          // ALL: every worker's condition just became true
        }
    }
}
```

**Why every line is the way it is:**

| Line | Reason |
|---|---|
| `private final Object lock` | See below — a dedicated lock object, never `this` |
| `boolean loaded` is **not** `volatile` | It is only read and written while holding `lock`. The monitor gives the happens-before edge (Topic 86). `volatile` here would be redundant and would mislead a reviewer into thinking the field is read outside the lock. |
| `while (!loaded)` | Spurious wakeups. Also: with many waiters, `notifyAll` wakes all of them and they re-acquire the lock one at a time — every one of them must re-check. |
| `lock.wait()` not `lock.wait(5000)` | This gate is one-shot and the signal is guaranteed by startup ordering. If it were not, a timeout plus a diagnostic log is the right call — see Trap 5. |
| `notifyAll()` not `notify()` | Rule 1 (uniform waiters) holds here, so `notify()` would be *correct* — but it would wake exactly one worker and leave the rest parked forever, because `markLoaded` is only ever called once. **This is the clearest example there is of `notify()` being individually correct and collectively catastrophic.** |
| `throws InterruptedException` | Not swallowed. Propagated. See Trap 4. |

### Why a private lock object rather than `this`

```java
synchronized (this) { ... }   // the monitor is PUBLIC API whether you meant it or not
```

If you synchronise on `this`, any code that can reach your object can do
`synchronized (gate) { ... }`, hold your monitor indefinitely, and hang your workers.
Worse, someone else's `gate.notifyAll()` becomes a spurious wakeup you did not plan
for — harmless with a `while` loop, and *fatal* with an `if`.

A `private final Object lock` makes the monitor unreachable from outside the class.
It costs 16 bytes. Use it.

**Run it:**

```java
public static void main(String[] args) throws Exception {
    PricingTableGate gate = new PricingTableGate();

    for (int i = 0; i < 4; i++) {
        int id = i;
        Thread.ofPlatform().name("pricing-worker-" + id).start(() -> {
            try {
                gate.awaitLoaded();
                System.out.println("worker " + id + " released");
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });
    }

    Thread.sleep(500);                 // let all four reach wait()
    System.out.println("loading table...");
    gate.markLoaded();
}
```

**What to look for:** four `released` lines, all after `loading table...`. Now change
`notifyAll()` to `notify()` and run again: **you should see exactly one `released`
line and the JVM should not exit**, because three non-daemon threads are still parked.
That is the experiment; the `Thread.sleep(500)` is a teaching crutch and would be a
defect in real code.

---

## Example 2 — production scenario (on the project spine)

### The constraints

These are the givens of the scenario, taken from your Topic 65 load profile. They are
requirements and budgets, not measurements.

| Constraint | Value |
|---|---|
| Container | `--cpus=4 --memory=2g` |
| Dataset | 100k products, 1M orders, 5M order lines |
| Load | k6, 400 rps sustained on `POST /orders` for 15 minutes, plus a 3-minute spike to 1200 rps |
| Tomcat request threads | 200 (Boot default) |
| HikariCP | `maximumPoolSize=20`, `connectionTimeout=30s` |
| SLO | `POST /orders` p99 under 400 ms; error budget 0.1% |
| Reservation work | 3 background workers, each reservation takes ~15 ms of DB work |
| Buffer capacity | 500 pending reservations, chosen in Topic 93 from the latency budget |

The design: the HTTP thread validates the order, writes the `PENDING_RESERVATION` row,
commits, and hands the reservation to a bounded in-memory buffer. Three worker threads
drain it and perform the inventory decrement. The buffer must be **bounded** — an
unbounded one is Topic 90's OOM drill, and this is exactly the same mistake wearing a
different hat.

### The implementation

```java
package com.orderflow.inventory;

import java.util.ArrayDeque;
import java.util.Deque;
import java.util.Objects;

/**
 * A bounded hand-off buffer between order-intake threads (producers) and
 * reservation workers (consumers), written with wait/notify.
 *
 * PRODUCTION NOTE: you would not ship this. java.util.concurrent.ArrayBlockingQueue
 * does the same job, with two condition queues instead of one and a decade of
 * hardening. It is written out here so that when you read ArrayBlockingQueue's source
 * in Topic 93 you recognise every line. See "Why you delete this" at the end.
 */
public final class ReservationBuffer {

    private final Object lock = new Object();
    private final Deque<Reservation> items = new ArrayDeque<>();
    private final int capacity;
    private boolean shuttingDown = false;

    // Metrics-facing counters. Guarded by `lock` like everything else.
    private long offeredCount;
    private long rejectedCount;
    private long peakDepth;

    public ReservationBuffer(int capacity) {
        if (capacity <= 0) throw new IllegalArgumentException("capacity must be positive");
        this.capacity = capacity;
    }

    /**
     * Called on a Tomcat request thread. MUST NOT block for long: a blocked request
     * thread is 1/200th of the service's entire request capacity.
     *
     * @return true if accepted; false if the buffer was full within the deadline.
     */
    public boolean offer(Reservation r, long timeoutMillis) throws InterruptedException {
        Objects.requireNonNull(r, "reservation");
        // Deadline arithmetic done ONCE, outside the loop, in nanos.
        // System.currentTimeMillis() is wall-clock and can jump backwards on an NTP step.
        final long deadlineNanos = System.nanoTime() + timeoutMillis * 1_000_000L;

        synchronized (lock) {
            offeredCount++;
            while (items.size() == capacity && !shuttingDown) {
                long remainingNanos = deadlineNanos - System.nanoTime();
                if (remainingNanos <= 0) {
                    rejectedCount++;
                    return false;              // backpressure, surfaced as a 503
                }
                // wait(millis, nanos) — round UP so we never busy-spin on a sub-ms remainder
                long millis = remainingNanos / 1_000_000L;
                int nanos = (int) (remainingNanos % 1_000_000L);
                lock.wait(millis, nanos);
            }
            if (shuttingDown) {
                rejectedCount++;
                return false;
            }
            items.addLast(r);
            if (items.size() > peakDepth) peakDepth = items.size();
            lock.notifyAll();                  // SAME block as the state change
            return true;
        }
    }

    /** Called on a reservation worker thread. Blocks until work exists or shutdown. */
    public Reservation take() throws InterruptedException {
        synchronized (lock) {
            while (items.isEmpty()) {
                if (shuttingDown) return null;     // poison-free shutdown signal
                lock.wait();
            }
            Reservation r = items.removeFirst();
            lock.notifyAll();                      // a producer may be waiting for space
            return r;
        }
    }

    /** Called from the Spring @PreDestroy hook. Wakes everyone so nobody parks forever. */
    public void shutdown() {
        synchronized (lock) {
            shuttingDown = true;
            lock.notifyAll();
        }
    }

    // ---- observability: cheap snapshot for Micrometer gauges (Topic 118) ----

    public record Snapshot(int depth, int capacity, long offered, long rejected, long peak) {}

    public Snapshot snapshot() {
        synchronized (lock) {
            return new Snapshot(items.size(), capacity, offeredCount, rejectedCount, peakDepth);
        }
    }
}
```

### The decisions worth defending in review

**Why `offer` has a timeout and `take` does not.** A producer is a Tomcat request
thread. Blocking one indefinitely burns 1/200th of the service's request capacity and
converts a downstream slowdown into a full outage — the exact shape of Topic 55's
connection-pool drill. So the producer gets a deadline and returns `false`, which the
controller maps to HTTP 503 with `Retry-After`. A consumer is a dedicated worker whose
only job is to wait, so blocking it costs nothing.

**Why `notifyAll` in `take` too.** After removing an item there is space, and a
producer may be parked waiting for it. Forgetting this signal is a real bug: producers
sit on their timeout, return 503, and you shed load with an empty buffer.

**Why both producers and consumers wait on the *same* monitor — and why that is the
weakness.** One monitor means exactly one wait set, so a producer's `notifyAll` also
wakes every waiting producer, which re-checks and re-parks. That is wasted wakeups.
Worse, it makes `notify()` **actively unsafe** here — see Trap 2. `ReentrantLock` with
`notFull`/`notEmpty` `Condition`s (Topic 94) gives two separate wait sets and lets you
use `signal()` safely. That is exactly what `ArrayBlockingQueue` does.

**Why the timeout uses `nanoTime`, recomputed inside the loop.** `currentTimeMillis`
is wall-clock and can move backwards on an NTP step, so a deadline built from it may
never expire; `nanoTime` is monotonic. And every wake — spurious or real — consumes an
unknown slice of the budget, so passing the *original* `timeoutMillis` on each
iteration would let a thread that woke ten times block for ten times its stated
timeout. This "recompute the remaining time" loop is the standard shape for every
timed guarded block in the JDK, and it is the part people get wrong.

**Why counters are inside the lock.** They are guarded state like any other. Making
them `AtomicLong` would be *more* code and *less* consistent — `snapshot()` would then
return values from different instants.

### Wiring it into `orderflow`

```java
package com.orderflow.inventory;

import jakarta.annotation.PreDestroy;
import org.springframework.stereotype.Component;

@Component
public class ReservationWorkerPool {

    private static final org.slf4j.Logger log =
            org.slf4j.LoggerFactory.getLogger(ReservationWorkerPool.class);

    private final ReservationBuffer buffer;
    private final InventoryService inventory;
    private final java.util.List<Thread> workers = new java.util.ArrayList<>();

    public ReservationWorkerPool(ReservationBuffer buffer, InventoryService inventory) {
        this.buffer = buffer;
        this.inventory = inventory;
        for (int i = 0; i < 3; i++) {
            Thread t = Thread.ofPlatform()
                    .name("orderflow-reservation-" + i)
                    .daemon(false)                 // we shut it down explicitly; see below
                    .unstarted(this::drainLoop);
            workers.add(t);
            t.start();
        }
    }

    private void drainLoop() {
        try {
            for (;;) {
                Reservation r = buffer.take();
                if (r == null) return;                     // shutdown
                // NEVER let an exception kill the worker thread. A dead worker is a
                // silent capacity loss: the buffer fills, producers 503, and nothing
                // in your logs says "one of three workers died".
                try { inventory.reserve(r); }
                catch (RuntimeException e) { log.error("reserve failed {}", r.orderId(), e); }
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();            // restore the flag, then exit
        }
    }

    @PreDestroy
    public void stop() throws InterruptedException {
        buffer.shutdown();
        for (Thread t : workers) {
            t.join(5_000);
            if (t.isAlive()) log.warn("worker {} did not stop within 5s", t.getName());
        }
    }
}
```

The `try/catch` inside the loop is not defensive noise. A worker thread that dies from
an uncaught exception disappears silently: `Thread.run` returns, the JVM prints to
`System.err` via the default uncaught-exception handler if you are lucky, and your
throughput drops by a third with no alert. Topic 90 has the pooled-executor version of
exactly this failure, and it is worse there.

### Why you delete this

Everything above is roughly 90 lines to reimplement four lines:

```java
BlockingQueue<Reservation> buffer = new ArrayBlockingQueue<>(500);
boolean accepted = buffer.offer(r, 50, TimeUnit.MILLISECONDS);   // producer
Reservation r = buffer.take();                                   // consumer
```

Topic 93 is where you make that swap. The reason to have written it once is that when
you read `ArrayBlockingQueue`'s source you will recognise the deadline loop, the two
conditions, and the signal-in-the-same-critical-section rule — because you have now
made all three mistakes yourself.

---

## Wrong approach → exact symptom → root cause → fix

### Trap 1 — `if` instead of `while` (broken by specification)

**Wrong:**

```java
synchronized (lock) {
    if (items.isEmpty()) {
        lock.wait();
    }
    return items.removeFirst();     // ArrayDeque throws NoSuchElementException when empty
}
```

**Exact symptom, and it has three faces depending on the collection:**

- With `ArrayDeque.removeFirst()`: `java.util.NoSuchElementException` from a worker
  thread, at a rate of roughly one per hundred thousand operations, with a stack trace
  in code that "obviously" just checked the queue was non-empty.
- With `Deque.poll()`: a `NullPointerException` several frames later, wherever the
  `null` is finally dereferenced — the worst variant, because the stack trace does not
  point at the concurrency bug at all.
- With `int count--` style state: a counter that drifts negative over hours. Nothing
  throws. You find it in a dashboard.

All three are **load-dependent and machine-dependent**. They pass CI. They pass a
100-thread stress loop on your laptop. They appear in production under real
concurrency, which is exactly why Topic 99 (jcstress) exists.

**Root cause:** two independent mechanisms, either of which is sufficient:

1. The JLS permits `wait()` to return spuriously. Your `if` treated the return as
   proof the condition changed.
2. Far more commonly: `notifyAll()` woke five consumers, one item was added, the
   first consumer to re-acquire the lock took it, and the other four returned from
   `wait()` into an empty queue.

**Fix:** `while`. That is the entire fix.

```java
synchronized (lock) {
    while (items.isEmpty()) {
        lock.wait();
    }
    return items.removeFirst();
}
```

**How to make it un-regressable:** ban the pattern in CI rather than in review. An
ErrorProne check (`WaitNotInLoop`) and SpotBugs' `WA_NOT_IN_LOOP` both detect it:

```bash
mvn -q com.github.spotbugs:spotbugs-maven-plugin:check -Dspotbugs.includeFilterFile=spotbugs-concurrency.xml
```

**What to look for:** a `WA_NOT_IN_LOOP` finding naming the class and line. If
SpotBugs reports nothing and you know the pattern is present, your build is running
SpotBugs on the wrong source set — check that `target/classes` actually contains the
class.

---

### Trap 2 — `notify()` waking the wrong waiter, deadlocking a mixed producer/consumer set

**Wrong:**

```java
// Producers and consumers wait on the SAME monitor, and this uses notify().
public boolean offer(Reservation r) throws InterruptedException {
    synchronized (lock) {
        while (items.size() == capacity) lock.wait();
        items.addLast(r);
        lock.notify();                          // <-- the defect
        return true;
    }
}

public Reservation take() throws InterruptedException {
    synchronized (lock) {
        while (items.isEmpty()) lock.wait();
        Reservation r = items.removeFirst();
        lock.notify();                          // <-- and here
        return r;
    }
}
```

**Exact symptom:** the service reaches a state where **throughput is exactly zero**,
CPU is near idle, no exception is thrown, no log line appears, and the liveness probe
passes because the HTTP port still accepts connections. Request threads pile up until
Tomcat's 200 are exhausted, and then new connections queue in the accept backlog. From
the outside this looks like a hung database.

**How to confirm it in 60 seconds:**

```bash
jcmd $(jcmd -l | grep -i orderflow | cut -d' ' -f1) Thread.print > dump1.txt
sleep 30
jcmd $(jcmd -l | grep -i orderflow | cut -d' ' -f1) Thread.print > dump2.txt
grep -c "ReservationBuffer" dump1.txt dump2.txt
grep -A3 "in Object.wait()" dump1.txt | grep "waiting on"
```

| What you see | What it means |
|---|---|
| The same threads `WAITING (on object monitor)` on the same `0x…` address in both dumps, with `cpu=` unchanged | Nothing is making progress on that monitor. Confirmed stall, not slowness. |
| Both producers and consumers `waiting on` the **same** hex address | **This is the diagnosis.** One wait set holds two kinds of waiter, so `notify()` can wake the useless kind. |
| A "Found one Java-level deadlock" section | This is **not** that. `notify` starvation is not a lock cycle; `Thread.print` will not flag it. You must recognise it by eye. |
| Consumers waiting, buffer depth gauge > 0 | Definitive: there is work available and nobody woke the thread that could do it. |

**Root cause, stated exactly:** `notify()` wakes **one arbitrary** thread from the
monitor's single wait set. With producers and consumers sharing that wait set, a
consumer's "there is now space" signal can wake **another consumer**, which re-checks
`items.isEmpty()`, finds it still true, and parks again — absorbing the signal. The
producer that was waiting for space is never woken. Now the buffer is full and no
producer proceeds; consumers are waiting for items and no consumer proceeds. Total
stall. Rule 1 (uniform waiters) was violated.

**Fix A — always correct, slightly wasteful:**

```java
lock.notifyAll();
```

**Fix B — the production answer, from Topic 94:** two wait sets, one per condition.

```java
private final ReentrantLock lock = new ReentrantLock();
private final Condition notFull  = lock.newCondition();
private final Condition notEmpty = lock.newCondition();

public boolean offer(Reservation r) throws InterruptedException {
    lock.lock();
    try {
        while (items.size() == capacity) notFull.await();
        items.addLast(r);
        notEmpty.signal();          // signal() is SAFE here: only consumers wait on notEmpty
        return true;
    } finally { lock.unlock(); }
}
```

`signal()` is now correct because the wait set it targets contains only threads
waiting for the same condition. That is exactly what `ArrayBlockingQueue` does — read
its source in Topic 93 and you will find these two `Condition` fields.

---

### Trap 3 — signalling outside the critical section that changes the state

**Wrong:**

```java
public void submit(Reservation r) {
    synchronized (lock) {
        items.addLast(r);
    }
    synchronized (lock) {          // separate block
        lock.notifyAll();
    }
}
```

Also wrong, and much more common because it looks like a performance optimisation:

```java
public void submit(Reservation r) {
    synchronized (lock) { items.addLast(r); }
    // "notify outside the lock so the woken thread doesn't immediately block"
    lock.notifyAll();              // IllegalMonitorStateException — you do not hold it
}
```

**Exact symptoms — two different ones:**

- The second form throws `java.lang.IllegalMonitorStateException: current thread is
  not owner`, immediately and every time. Loud. Trivially caught in a smoke test.
- The first form is **silent**. Under low load it works. Under production concurrency
  a consumer slots into the gap between the two blocks, checks the condition against
  state that is *already updated*, and parks — after which the `notifyAll` fires into
  an empty wait set. Result: a permanently parked consumer with items in the buffer.
  This is steps 15–17 of the Concurrency trace.

**Root cause:** the check-and-park in the consumer and the change-and-signal in the
producer must be **mutually exclusive** as units. Splitting the producer into two
critical sections opens a window in which a consumer can observe the new state and
still park, because the signal has not been sent yet. `notify()` has no memory, so a
signal into an empty wait set is discarded.

**Fix:** one critical section containing both the state change and the signal.

```java
public void submit(Reservation r) {
    synchronized (lock) {
        items.addLast(r);
        lock.notifyAll();
    }
}
```

**The "notify outside the lock is faster" argument, answered:** the concern is real —
a notified thread must acquire the lock you still hold, so it wakes and immediately
blocks. HotSpot already mitigates this internally, and the cost is a microsecond-scale
context switch against a 15 ms database operation. You are trading a guaranteed
correctness property for an unmeasurable win. Do not.

---

### Trap 4 — swallowing `InterruptedException`

**Wrong:**

```java
try {
    lock.wait();
} catch (InterruptedException e) {
    // nothing to do here
}
```

**Exact symptom:** `@PreDestroy` runs, `executor.shutdownNow()` interrupts the
workers, and **the workers keep running**. The container's `SIGTERM` grace period
(30 s on the k8s default) elapses, Kubernetes sends `SIGKILL`, and in-flight
reservations are lost mid-write. In the logs you see the shutdown starting and never
completing. Under `kubectl describe pod` the container's last state shows a non-zero
exit and `Reason: Error` rather than a clean stop.

**Root cause:** catching `InterruptedException` **clears the thread's interrupt
flag**. That is the key fact and it is not obvious. Swallowing the exception destroys
the only record that anyone asked the thread to stop, so every subsequent
interrupt-aware call (`wait`, `sleep`, `take`, `join`, `Future.get`) behaves as if
nothing happened.

**Fix — one of exactly two, always:**

```java
// 1. Propagate it. Best, when your signature allows.
public Reservation take() throws InterruptedException { ... lock.wait(); ... }

// 2. Restore the flag and stop what you are doing. When you cannot propagate.
try {
    lock.wait();
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();   // restore the flag
    return null;                          // and actually abandon the work
}
```

There is no third option. "Log it and continue the loop" is Topic 09's swallowed
exception with a thread attached: you have logged that shutdown was requested and then
ignored the request.

---

### Trap 5 — waiting forever with no timeout and no diagnostics

**Wrong:**

```java
synchronized (lock) {
    while (!paymentConfirmed) {
        lock.wait();               // no timeout, no logging, no metric
    }
}
```

**Exact symptom:** a request thread parked indefinitely. Tomcat's thread count climbs
to 200 over an hour, then every endpoint returns connection timeouts — including
health endpoints that touch nothing. Your dashboards show a service that is "up" with
zero throughput. Nothing in the logs marks the transition.

**Root cause:** an unbounded wait converts any missed signal, any dropped webhook, any
upstream crash into a permanent thread leak. A guarded block with no time bound is a
guarded block with no failure mode — and code without a failure mode does not have
zero failures, it has invisible ones.

**Fix — bound it, and make the bound observable:**

```java
private static final long BUDGET_NANOS = TimeUnit.SECONDS.toNanos(5);

boolean awaitPaymentConfirmation() throws InterruptedException {
    final long deadline = System.nanoTime() + BUDGET_NANOS;
    synchronized (lock) {
        while (!paymentConfirmed) {
            long remaining = deadline - System.nanoTime();
            if (remaining <= 0) {
                registry.counter("orderflow.payment.confirmation.timeout").increment();
                log.warn("payment confirmation not received within 5s");
                return false;
            }
            lock.wait(remaining / 1_000_000L, (int) (remaining % 1_000_000L));
        }
        return true;
    }
}
```

Note the return value. `wait(long)` does **not** tell you whether it returned because
of a notify or because of the timeout — that information is simply not available from
the API. You must derive it by re-checking the condition and comparing against your
own deadline. Every timed guarded block in the JDK does exactly this.

---

## Hands-on proof

Everything here is a command you run. I have no JVM and will not print output and
claim it is real. What follows is precisely what to run, what to look for, and how to
interpret each possible result.

### Setup

```bash
mkdir -p ~/java-lab/89 && cd ~/java-lab/89
java --version           # expect 21 or 25
```

### Proof 1 — `wait()` without the monitor throws immediately

`NoMonitor.java`:

```java
public class NoMonitor {
    public static void main(String[] args) throws Exception {
        Object lock = new Object();
        lock.wait();          // no synchronized block
    }
}
```

```bash
java NoMonitor.java
```

**What to look for:**

| What you see | What it means |
|---|---|
| `java.lang.IllegalMonitorStateException: current thread is not owner` | Correct. The JVM verifies monitor ownership at the point of the call. This is the one loud failure in the whole topic. |
| The program hangs | You accidentally wrapped it in `synchronized`. Re-read your file. |
| No exception, immediate exit | Impossible on a compliant JVM. Check you are running the file you edited. |

**Why bother:** it proves ownership is checked at runtime, not at compile time. `javac`
will happily compile a `wait()` you have no right to call.

### Proof 2 — `wait()` releases the lock; `sleep()` does not

`ReleaseProof.java`:

```java
public class ReleaseProof {
    private static final Object lock = new Object();

    public static void main(String[] args) throws Exception {
        boolean useSleep = args.length > 0 && args[0].equals("sleep");

        Thread holder = Thread.ofPlatform().name("holder").start(() -> {
            synchronized (lock) {
                try { if (useSleep) Thread.sleep(3000); else lock.wait(3000); }
                catch (InterruptedException e) { Thread.currentThread().interrupt(); }
            }
        });
        Thread.sleep(300);                    // let holder get inside the block

        long start = System.nanoTime();
        Thread contender = Thread.ofPlatform().name("contender")
                .start(() -> { synchronized (lock) { /* just get in */ } });
        contender.join();

        System.out.println("mode=" + (useSleep ? "sleep" : "wait")
                + " contenderWaitedMillis=" + (System.nanoTime() - start) / 1_000_000L);
        holder.join();
    }
}
```

```bash
java ReleaseProof.java wait
java ReleaseProof.java sleep
```

**What to look for:** the `contenderWaitedMillis` value in each mode.

| What you see | What it means |
|---|---|
| `mode=wait` with a small `contenderWaitedMillis` (single- or low double-digit) | `wait()` released the monitor. The contender entered while the holder was still parked. **This is the mechanism, demonstrated.** |
| `mode=sleep` with `contenderWaitedMillis` close to the remaining 2700 ms | `sleep()` held the monitor for its whole duration. The contender was in the entry set the entire time. |
| Both modes small | You are not actually contending — check that `Thread.sleep(300)` is long enough for `holder` to enter the block on your machine, and that both threads use the same `lock` object. |
| Both modes large | The contender started before the holder entered. Increase the initial `Thread.sleep`. |

> These are wall-clock numbers from a deliberately crude harness, and they are fine for
> this purpose because the effect is three orders of magnitude larger than the noise.
> Do not carry this technique into anything where the difference is small — that is
> Topic 77's subject, and the Measurement section below says so again.

### Proof 3 — see both queues in a thread dump

`TwoQueues.java`:

```java
public class TwoQueues {
    private static final Object lock = new Object();

    public static void main(String[] args) throws Exception {
        for (int i = 0; i < 2; i++)                    // two threads → the WAIT SET
            Thread.ofPlatform().name("waiter-" + i).start(() -> {
                synchronized (lock) {
                    try { while (true) lock.wait(); }
                    catch (InterruptedException e) { Thread.currentThread().interrupt(); }
                }
            });
        Thread.sleep(500);

        Thread.ofPlatform().name("hog").start(() -> {  // holds the monitor forever
            synchronized (lock) {
                try { Thread.sleep(Long.MAX_VALUE); }
                catch (InterruptedException e) { Thread.currentThread().interrupt(); }
            }
        });
        Thread.sleep(300);
        for (int i = 0; i < 2; i++)                    // two threads → the ENTRY SET
            Thread.ofPlatform().name("blocked-" + i).start(() -> {
                synchronized (lock) { System.out.println("never printed"); }
            });

        System.out.println("pid=" + ProcessHandle.current().pid() + " — now take a dump");
        Thread.sleep(600_000);
    }
}
```

```bash
java TwoQueues.java &
# use the pid it prints
jcmd <pid> Thread.print > dump.txt
grep -E '"(waiter|hog|blocked)-?[0-9]*"' -A4 dump.txt
```

**What to look for:** the `Thread.State` line for each named thread, and the
`waiting on` / `waiting to lock` markers.

| What you see | What it means |
|---|---|
| `waiter-0`, `waiter-1`: `WAITING (on object monitor)` with `- waiting on <0x…>` | They are in the **wait set**. They called `wait()`. |
| `blocked-0`, `blocked-1`: `BLOCKED (on object monitor)` with `- waiting to lock <0x…>` | They are in the **entry set**. They never called `wait()`; they cannot get in. |
| `hog`: `TIMED_WAITING (sleeping)` with `- locked <0x…>` | It owns the monitor and is asleep inside it. This is the thread to blame. |
| All five report the **same** `0x…` address | Correct — one monitor, two queues. Matching the address is how you connect a blocked thread to its blocker in a real dump. |
| `waiting on` and `waiting to lock` use different wording | This is the distinction to memorise. `waiting on` = wait set. `waiting to lock` = entry set. |

Then send a `notifyAll` and take a second dump: the two waiters transition to
`BLOCKED` on the same monitor, because `hog` still owns it. **Being notified is not
being scheduled** — that transition is the proof.

### Proof 4 — reproduce the lost wakeup deterministically

Race conditions are not reproducible by luck. Force the interleaving with a
`CountDownLatch` (Topic 97) as a synthetic scheduler.

`LostWakeup.java`:

```java
import java.util.ArrayDeque;
import java.util.Deque;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;

public class LostWakeup {
    private static final Object lock = new Object();
    private static final Deque<String> items = new ArrayDeque<>();

    // Latches that let us force the exact interleaving from the Concurrency trace.
    private static final CountDownLatch consumerHoldsLock  = new CountDownLatch(1);
    private static final CountDownLatch producerFinishedAdd = new CountDownLatch(1);

    public static void main(String[] args) throws Exception {
        Thread consumer = Thread.ofPlatform().name("consumer").start(() -> {
            synchronized (lock) {
                consumerHoldsLock.countDown();                 // step 1-2: condition checked
                try {
                    producerFinishedAdd.await();               // step 3: pause BEFORE wait()
                    if (items.isEmpty()) lock.wait(2000);      // step 5: park (bounded so the demo ends)
                } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
                System.out.println("consumer woke; queue size = " + items.size());
            }
        });

        consumerHoldsLock.await();

        Thread producer = Thread.ofPlatform().name("producer").start(() -> {
            synchronized (lock) { items.addLast("ORD-88215"); }   // block 1: state change
            producerFinishedAdd.countDown();                      // release the consumer
            try { TimeUnit.MILLISECONDS.sleep(50); } catch (InterruptedException ignored) {}
            synchronized (lock) { lock.notifyAll(); }             // block 2: signal, too late
            System.out.println("producer signalled");
        });

        consumer.join();
        producer.join();
    }
}
```

Note the consumer cannot enter `wait()` until the producer has already added the item
and released the lock — but the producer's `notifyAll` is 50 ms behind, in a separate
block. The consumer parks with the item already present.

```bash
java LostWakeup.java
```

**What to look for:**

| What you see | What it means |
|---|---|
| `consumer woke; queue size = 1` printed after roughly 2 seconds | **The drill fired.** The consumer waited its full 2-second timeout while an item sat in the queue. Change `wait(2000)` to `wait()` and it parks forever. |
| The same line printed within ~50 ms | The consumer was notified in time on your machine. Increase the producer's sleep to widen the window. |
| `producer signalled` printed before the consumer line | Expected — the producer's signal arrived while the consumer was parked, so it did wake it. The point stands: had the producer's thread been descheduled, killed, or slow between the two blocks, the signal never arrives. |
| Rewrite `submit` with `addLast` and `notifyAll` in ONE block, re-run | The consumer never parks at all: it finds the queue non-empty on its first check. **That is the fix, proven.** |

### Proof 5 — the specification claim, from your own JDK

```bash
unzip -p "$JAVA_HOME/lib/src.zip" java.base/java/lang/Object.java | grep -n -B2 -A8 -i "spurious"
```

**What to look for:** the sentence in the `wait` javadoc describing a wake without
notification, interruption or timeout, and the accompanying recommendation to wait in
a loop.

| What you see | What it means |
|---|---|
| A javadoc paragraph naming spurious wakeups | Confirmed from primary source on your JDK. This is the authority for the `while` rule. |
| `unzip: cannot find` | Your JDK ships without sources. `sdk install java <version>` distributions usually include `src.zip`; otherwise read `java.lang.Object`'s javadoc for your exact version. |
| Nothing matches | Check `$JAVA_HOME` actually points at a JDK, not a JRE: `echo $JAVA_HOME && ls $JAVA_HOME/lib/src.zip`. |

---

## Failure drill

**Mandatory. Produce the failure yourself and write down what you saw before reading
the fix.** The knowledge is not the point; the memory of a service with zero
throughput and zero errors is the point.

### The scenario

`orderflow` is running under the Topic 65 load profile. The reservation buffer uses
`notify()` and a shared monitor for producers and consumers. You will drive it until
producers and consumers are simultaneously parked, and the service goes to zero
throughput while the health check stays green.

### Setup — the deliberately broken buffer

`src/main/java/com/orderflow/inventory/BrokenReservationBuffer.java`:

```java
package com.orderflow.inventory;

import java.util.ArrayDeque;
import java.util.Deque;

/** DELIBERATELY BROKEN. Uses notify() with a mixed producer/consumer wait set. */
public final class BrokenReservationBuffer {

    private final Object lock = new Object();
    private final Deque<Reservation> items = new ArrayDeque<>();
    private final int capacity;

    public BrokenReservationBuffer(int capacity) { this.capacity = capacity; }

    public void offer(Reservation r) throws InterruptedException {
        synchronized (lock) {
            while (items.size() == capacity) lock.wait();
            items.addLast(r);
            lock.notify();                 // <-- DEFECT: may wake another producer
        }
    }

    public Reservation take() throws InterruptedException {
        synchronized (lock) {
            while (items.isEmpty()) lock.wait();
            Reservation r = items.removeFirst();
            lock.notify();                 // <-- DEFECT: may wake another consumer
            return r;
        }
    }

    public int depth() { synchronized (lock) { return items.size(); } }
}
```

Configure a **small capacity** — 4, not 500. A small buffer makes both sides wait
often, which is what you need to hit the interleaving quickly. Use 8 producer threads
and 4 consumer threads so both wait sets are populated.

`src/main/java/com/orderflow/inventory/DrillRunner.java`:

```java
package com.orderflow.inventory;

import java.util.concurrent.atomic.AtomicLong;

public class DrillRunner {
    static final AtomicLong produced = new AtomicLong(), consumed = new AtomicLong();

    public static void main(String[] args) throws Exception {
        BrokenReservationBuffer buffer = new BrokenReservationBuffer(4);

        for (int i = 0; i < 8; i++) {                       // 8 producers
            int id = i;
            Thread.ofPlatform().name("producer-" + id).daemon(true).start(() -> {
                try { for (long n = 0; ; n++) {
                    buffer.offer(new Reservation("ORD-" + id + "-" + n, "SKU-4471", 1));
                    produced.incrementAndGet();
                } } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
            });
        }
        for (int i = 0; i < 4; i++) {                       // 4 consumers
            Thread.ofPlatform().name("consumer-" + i).daemon(true).start(() -> {
                try { for (;;) {
                    buffer.take();
                    consumed.incrementAndGet();
                    Thread.sleep(15);                       // the ~15ms inventory decrement
                } } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
            });
        }

        System.out.println("pid=" + ProcessHandle.current().pid());
        long lp = 0, lc = 0;
        for (;;) {
            Thread.sleep(1000);
            long p = produced.get(), c = consumed.get();
            System.out.printf("depth=%d producedDelta=%d consumedDelta=%d%n",
                    buffer.depth(), p - lp, c - lc);
            lp = p; lc = c;
        }
    }
}
```

### Commands

```bash
# 1. Run it. Note the pid it prints.
java -Xss512k src/main/java/com/orderflow/inventory/DrillRunner.java

# 2. When both delta columns read 0 for several consecutive seconds, dump.
jcmd <pid> Thread.print > stall1.txt
sleep 30
jcmd <pid> Thread.print > stall2.txt

# 3. Count who is where.
grep -c 'WAITING (on object monitor)' stall1.txt
grep -B6 'Object.wait' stall1.txt | grep -oE '"(producer|consumer)-[0-9]+"' | sort | uniq -c

# 4. Confirm nothing is moving between the two dumps.
diff <(grep -oE '"[a-z]+-[0-9]+".*cpu=[0-9]+ms' stall1.txt) \
     <(grep -oE '"[a-z]+-[0-9]+".*cpu=[0-9]+ms' stall2.txt)

# 5. Confirm the JVM is idle, not spinning.
top -pid <pid> -l 2      # macOS;  on Linux:  top -H -p <pid> -n 2 -b
```

### What to capture

Write these six things down before reading on:

1. The `depth=` value at the moment both deltas hit zero.
2. How many `producer-*` threads are `WAITING (on object monitor)`.
3. How many `consumer-*` threads are `WAITING (on object monitor)`.
4. Whether they all report the **same** `0x…` monitor address.
5. Whether `cpu=` changed for any of them between `stall1.txt` and `stall2.txt`.
6. The process CPU percentage from step 5's `top`.

### How to read it

| What you see | What it means |
|---|---|
| `producedDelta=0 consumedDelta=0` for 10+ consecutive seconds, `depth` frozen at some value | **The drill has fired.** Total stall. |
| Producers **and** consumers both in `WAITING (on object monitor)` on the same `0x…` | The diagnosis. One wait set, two kinds of waiter, `notify()` woke the wrong kind and the signal was absorbed. |
| `depth=4` (full) with consumers parked | Consumers are waiting for items while the buffer is *full*. Physically impossible unless a signal was lost — this single line is the proof. |
| `depth=0` with producers parked | The mirror image: producers waiting for space in an *empty* buffer. Equally conclusive. |
| `cpu=` identical across both dumps for every parked thread | Genuinely parked. Not a livelock (Topic 98's contrast case — livelock burns CPU). |
| Process CPU near 0% | Confirms the same. A stalled `notify()` system is indistinguishable from an idle one to any monitor that only watches CPU. |
| No "Found one Java-level deadlock" section | Expected and important. `Thread.print` detects **lock cycles**. This is not a lock cycle — it is a lost signal. Your tooling will not name it for you. That is why you must recognise the shape. |
| It never stalls | Widen the window: reduce capacity to 2, raise producers to 16, or add `Thread.sleep(1)` between `items.addLast` and `notify` (temporarily splitting the critical section) to make the race far more likely. The bug is real; you are fighting probability. |

### Now fix it — two ways, then a recommendation

**Fix 1 — `notifyAll()`.** Change both `lock.notify()` calls to `lock.notifyAll()`.
Re-run. The deltas should stay non-zero indefinitely.

*What it costs:* every signal wakes all 12 threads; 11 of them re-check, find their
condition false, and re-park. At `orderflow`'s scale — 400 rps against a 15 ms
operation — that is invisible. At a million operations per second it would matter.

**Fix 2 — two `Condition`s on a `ReentrantLock`.** Apply Trap 2's Fix B verbatim.
Two wait sets, one per condition, so `signal()` becomes safe again because rule 1
(uniform waiters) now holds *per condition*. Note `while` is still mandatory —
`Condition.await` is permitted to return spuriously too, and `signal` still races with
a competing consumer. This is Topic 94's subject in full.

**Fix 3 — the one you actually ship.** Delete the class:

```java
BlockingQueue<Reservation> buffer = new ArrayBlockingQueue<>(500);
```

`ArrayBlockingQueue` is Fix 2, written by Doug Lea, with a decade of production
hardening and a jcstress suite behind it. Topic 93.

### What the fix proves

Three things, and say them in this order in an interview:

1. **`notify()` requires a uniform wait set.** Mixing producers and consumers on one
   monitor breaks that precondition, and the failure is a total stall with no error.
2. **The absence of an event is a failure mode.** Nothing threw. Nothing logged. CPU
   was idle. Your instinct from Node — "find the exception" — has nothing to find.
3. **`notifyAll` is the correct default.** The cost is measurable and small; the cost
   of `notify()` being wrong is unbounded.

---

## Measurement

You will be asked "is `notifyAll` expensive?" and "how do I know my workers are
waiting rather than working?" Here is how to answer both honestly.

### The standing rule, restated

```java
// WRONG. Every number this produces is untrustworthy.
long start = System.nanoTime();
for (int i = 0; i < 1_000_000; i++) { buffer.offer(r); buffer.take(); }
System.out.println((System.nanoTime() - start) / 1_000_000 + " ns/op");
```

Four independent reasons, and you cannot tell which one is lying: dead-code
elimination (the results are unused), on-stack replacement (the loop is compiled
mid-flight, so your average blends interpreted, C1 and C2 execution), cold-JIT warmup
in the first thousands of iterations, and — specific to concurrency — the fact that
this measures *your machine's scheduler on this particular run*, which is the single
least reproducible thing in the system. This is Topic 77's whole subject. Treat any
blog post quoting `notify` vs `notifyAll` overhead from a `nanoTime` loop as fiction.

### JMH with `@Threads` — the correct harness

```java
package com.orderflow.bench;

import org.openjdk.jmh.annotations.*;
import java.util.ArrayDeque;
import java.util.Deque;
import java.util.concurrent.TimeUnit;

@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.SECONDS)
@Warmup(iterations = 5, time = 2)
@Measurement(iterations = 10, time = 2)
@Fork(3)
@State(Scope.Benchmark)
public class NotifyStrategyBenchmark {

    @Param({"1", "4", "16", "64"})
    public int waiters;                    // the variable that actually matters

    private final Object lock = new Object();
    private final Deque<Integer> items = new ArrayDeque<>();

    @Setup(Level.Iteration)
    public void reset() { synchronized (lock) { items.clear(); } }

    @Benchmark
    @Group("all")   @GroupThreads(1)
    public void produceNotifyAll() {
        synchronized (lock) { items.addLast(1); lock.notifyAll(); }
    }

    @Benchmark
    @Group("all")   @GroupThreads(4)
    public Integer consumeNotifyAll() throws InterruptedException {
        synchronized (lock) {
            while (items.isEmpty()) lock.wait();
            return items.removeFirst();
        }
    }
}
```

| Annotation | What it defends against |
|---|---|
| `@Fork(3)` | Three separate JVMs. Defeats profile pollution and exposes run-to-run variance. A single fork can be silently wrong. |
| `@Warmup(iterations = 5, time = 2)` | Lets C2 compile and lock inflation settle before anything is recorded. Concurrency benchmarks need *longer* warmup than single-threaded ones because the lock must inflate. |
| `@Group` / `@GroupThreads` | Runs producers and consumers **concurrently** in fixed ratios. A plain `@Threads(N)` running one method is not a producer-consumer benchmark at all. |
| `@Param({"1","4","16","64"})` | The whole question is how the cost scales with wait-set size. One data point answers nothing. |
| `@State(Scope.Benchmark)` | Shared state across threads — which is the point here, unlike most benchmarks. |

```bash
./mvnw -Pbench clean verify
java -jar target/benchmarks.jar NotifyStrategyBenchmark -rf json -rff notify-bench.json
```

**What to look for:** the throughput number **and the `±` error margin JMH prints
beside it**, at each `waiters` value. If the margins for `notify` and `notifyAll`
overlap at your real wait-set size, you have measured nothing and the correctness
argument decides it. Report the interval, never the point estimate.

**The honest interpretation, stated before you run it:** you are comparing a
microsecond-scale wakeup cost against a 15 ms database operation in the real system.
Even a 10× difference in wakeup cost is roughly one ten-thousandth of the request
budget. If your measurement says otherwise, suspect the benchmark first.

### `jcmd` — is anyone actually waiting?

```bash
PID=$(jcmd -l | grep -i orderflow | cut -d' ' -f1)

# One-shot census of thread states
jcmd $PID Thread.print | grep -c 'WAITING (on object monitor)'
jcmd $PID Thread.print | grep -c 'BLOCKED (on object monitor)'
jcmd $PID Thread.print | grep -c 'RUNNABLE'

# Which monitors, and how many threads each
jcmd $PID Thread.print | grep -oE '(waiting on|waiting to lock) <0x[0-9a-f]+>' | sort | uniq -c | sort -rn
```

| What you see | What it means |
|---|---|
| A high `WAITING (on object monitor)` count with low throughput | Threads are parked in wait sets. Either genuinely no work, or a lost wakeup. Compare against your buffer-depth gauge to tell which. |
| A high `BLOCKED` count | Lock **contention**, not waiting. Different problem, Topic 85's drill. |
| One monitor address dominating the `uniq -c` output | Your hot lock. That address is what you take to JFR. |
| `WAITING` count high **and** buffer depth > 0 | **Lost wakeup, confirmed.** Work exists and nobody was woken. |

### JFR — the durable version

```bash
java -XX:StartFlightRecording=duration=120s,filename=orderflow-89.jfr,settings=profile \
     -jar target/orderflow.jar

jfr summary orderflow-89.jfr
jfr print --events jdk.JavaMonitorWait  orderflow-89.jfr | head -80
jfr print --events jdk.JavaMonitorEnter orderflow-89.jfr | head -80
```

*Illustration of the event FORMAT, not captured output. `<n>` are placeholders.*

```
jdk.JavaMonitorWait {
  startTime = <timestamp>
  monitorClass = com.orderflow.inventory.ReservationBuffer
  notifier = "orderflow-http-<n>"
  timeout = <n> ms
  timedOut = <true|false>
  duration = <n> ms
  eventThread = "orderflow-reservation-<n>"
  stackTrace = [ ... ]
}
```

**The fields that matter:**

| Field | How to use it |
|---|---|
| `monitorClass` | Which of your objects is the contention/wait point. Group by this first. |
| `duration` | How long the thread was parked. A p99 near your timeout means starvation. |
| `timedOut` | **The single most diagnostic field in this topic.** A high `timedOut=true` rate means signals are being missed or are too slow. |
| `notifier` | The thread that woke it — `null` if it timed out or woke spuriously. A `null` notifier with `timedOut=false` is a spurious wakeup, observed. |
| `jdk.JavaMonitorEnter.duration` | Time spent in the **entry set**. This is contention, not waiting. Keep the two separate in your head and in your dashboards. |

**Note the threshold.** By default JFR only records monitor events above a duration
threshold (in the `profile` settings, typically low tens of milliseconds). A flood of
short waits will not appear. Lower it deliberately when hunting:

```bash
jfr print --events jdk.JavaMonitorWait orderflow-89.jfr | grep -c "timedOut = true"
# and to capture short waits, use a custom settings file with
#   <event name="jdk.JavaMonitorWait"><setting name="threshold">1 ms</setting></event>
```

### Micrometer metrics — what to actually put on a dashboard

Forward-reference to Topic 118. Four series, and no more:

```java
@Component
public class ReservationBufferMetrics {

    private final Timer waitTimer;

    public ReservationBufferMetrics(MeterRegistry registry, ReservationBuffer buffer) {
        // 1. DEPTH — the leading indicator. Rising depth means consumers are behind.
        registry.gauge("orderflow.reservation.buffer.depth", buffer, b -> b.snapshot().depth());

        // 2. SATURATION — depth / capacity. Alert on THIS, not on depth: it survives
        //    a capacity change without you having to move the threshold.
        registry.gauge("orderflow.reservation.buffer.saturation", buffer,
                b -> (double) b.snapshot().depth() / b.snapshot().capacity());

        // 3. REJECTIONS — backpressure actually applied. SHOULD be non-zero under
        //    overload. Zero rejections under overload means something is queueing
        //    without bound, which is Topic 90's failure.
        registry.gauge("orderflow.reservation.buffer.rejected", buffer,
                b -> b.snapshot().rejected());

        // 4. WAIT TIME — how long producers park. This is latency the customer feels.
        this.waitTimer = Timer.builder("orderflow.reservation.buffer.wait")
                .publishPercentiles(0.5, 0.95, 0.99).register(registry);
    }

    public Timer waitTimer() { return waitTimer; }
}
```

**The one rule that matters:** do **not** tag any of these with the order ID or SKU.
That is Topic 118's cardinality drill — 100k products becomes 100k time series and
kills the scrape.

**How to read the dashboard:** depth rising while `consumed` throughput stays flat is
the signature of consumers falling behind. Depth rising while throughput is **zero**
and CPU is idle is the signature of this topic's failure — a lost wakeup.

---

## Practice exercises

Write real files, run them, and record what you observed — including "nothing
happened", which is a legitimate and common result in this topic.

### 1 — easy

Build a `BoundedSkuBuffer` for `orderflow`: capacity 4, holding `String` SKUs, with
`put(String)` and `String take()`, both blocking, both using `wait`/`notify` on a
private lock object.

Requirements:

- Use `while`, `notifyAll`, and a single critical section per method.
- Drive it with 2 producers and 1 consumer for 5 seconds, counting puts and takes.
- Print the count each second. Confirm producers block when the buffer is full — prove
  it by making the consumer sleep 200 ms per item and showing the produced-per-second
  rate converges to 5.

Then make **one** change at a time and record the exact symptom of each:

1. `while` → `if`. Run it 50 times in a loop from a shell script. Report how many runs
   throw and what they throw. (If none do, widen the window: add more consumers.)
2. `notifyAll` → `notify`. Report whether it stalls and how long you waited before
   deciding.
3. Move `notifyAll` into a second `synchronized` block. Report what changes.

Deliverable: a table with three rows — change, symptom observed, how long it took you
to notice.

### 2 — medium (combines Topics 01–88)

Build a `PriceRefreshCoordinator` for `orderflow` with the following contract, and use
material from at least five earlier topics.

**Contract:** a single background thread refreshes the price of every product in a
`Map<Long, Money>` every 30 seconds. Request threads must (a) never see a
half-refreshed map, (b) never block for more than 50 ms waiting for a refresh, and
(c) get the *previous* snapshot if a refresh is in progress and has exceeded 50 ms.

Requirements, each tied to a topic you already have:

- **Topic 01:** prices are `long` minor units, never `double`. Comment on why
  `Map<Long, Long>` is an allocation-rate concern at 100k products.
- **Topic 13:** the key is a `ProductId` record, not a `Long`. Justify in one sentence
  about `equals`/`hashCode` why a record is safe and a mutable class would not be.
- **Topic 17 / 88:** publish each snapshot as an immutable `Map.copyOf` held in a
  `final` field. Name which of the five safe-publication idioms you used, and why it
  removes the need for `volatile` on the map's *contents*.
- **Topic 87:** the *reference* to the current snapshot **is** `volatile`. One line on
  why the field needs it and the contents do not.
- **Topic 89:** the "wait up to 50 ms" path is a timed guarded block with a `nanoTime`
  deadline recomputed inside the loop.
- **Topic 26:** the read path returns `Optional<Money>` and never calls `get()`.

Then answer in prose: (a) which contract clause silently breaks if you use `if`
instead of `while`, and what an on-call engineer would see; (b) you could implement
this with no `wait`/`notify` at all — a `volatile` snapshot reference and readers that
take whatever is current. What does the guarded block buy? Argue both sides, pick one.

### 3 — hard (production simulation on `orderflow` under load)

Run the full drill against the real service under the Topic 65 load profile, and
produce a written diagnosis of the kind you would attach to an incident ticket.

**Part A — instrument.** Add the four Micrometer series from the Measurement section
to your `orderflow` reservation buffer. Verify they appear at `/actuator/prometheus`.
Confirm no series is tagged with an order ID or SKU.

**Part B — break it.** Swap in `BrokenReservationBuffer` (capacity 4, `notify()`).
Start the k6 load profile: 400 rps on `POST /orders` for 15 minutes with a 3-minute
spike to 1200 rps.

**Part C — capture.** During the stall, capture all of:

- two `jcmd <pid> Thread.print` dumps 30 seconds apart
- a 120-second JFR recording including `jdk.JavaMonitorWait` with the threshold
  lowered to 1 ms
- the four Micrometer series over the whole window
- the k6 summary (request rate, error rate, p99)

**Part D — diagnose from evidence only.** Write the diagnosis as if you did not
already know the answer. It must answer:

1. At what buffer depth did throughput reach zero, and was the buffer full or empty?
2. How many threads were in the wait set, split by producer and consumer, and on which
   monitor address?
3. What is the `timedOut=true` rate in `jdk.JavaMonitorWait`, and what does that rate
   imply about how many signals were lost versus merely delayed?
4. Which single metric would have alerted you fastest, and what threshold would you
   set? Justify the threshold against the SLO (`POST /orders` p99 under 400 ms),
   not against a round number.
5. **Would a liveness probe have caught this?** Answer honestly, then say what probe
   *would* have — and connect it to Topic 121's liveness-versus-readiness distinction.

**Part E — fix and re-measure.** Apply `notifyAll`, re-run, and confirm throughput
recovers. Then apply the two-`Condition` version and re-run. Compare:

- throughput at the 400 rps steady state
- p99 of `orderflow.reservation.buffer.wait`
- the `jdk.JavaMonitorWait` event count in each configuration

**Part F — the honest conclusion.** Given your numbers, is the two-`Condition` version
worth the extra 20 lines over `notifyAll` **in this system**? State the condition
under which your answer flips — a specific throughput, a specific wait-set size, or a
specific latency budget. "The difference was inside the error margin, here are the
margins" is a better answer than a confident wrong one.

---

## Interview questions

### Q1 — "Why must `wait()` be in a `while` loop and not an `if`?"

**Mid-level answer:** "Because of spurious wakeups — a thread can wake up without
being notified, so you have to re-check the condition."

**Senior answer:** "Two reasons, and the second is the one that actually bites. First,
the JLS and the `Object.wait` javadoc explicitly permit spurious wakeups, so an `if`
is broken by specification regardless of how the JVM behaves — that alone settles it.
But the far more common cause in practice is a *stolen* wakeup: `notifyAll` wakes five
consumers, one item was added, the first consumer to re-acquire the monitor takes it,
and the other four return from `wait()` into an empty queue. From their point of view
that is indistinguishable from a spurious wakeup and it happens constantly under load.
The deeper principle is that returning from `wait()` carries no information — it is
not a message, it has no payload, it just means 'you are runnable, go look at the
state yourself'. `while` is what encodes that. And the symptom of getting it wrong is
a `NoSuchElementException` or an NPE at a rate of about one in a hundred thousand,
which passes CI and appears in production."

**What separates them:** the mid-level answer recites the spec clause. The senior
answer names the *common* cause, explains why the return value is uninformative, and
states the observable symptom and why testing misses it.

**Follow-up:** "Does `Condition.await` have the same requirement?" — yes, identical,
and for the same two reasons. Also: "Does `park()`?" — yes; `LockSupport.park` is
permitted to return spuriously too.

---

### Q2 — "When is `notify()` safe, and when does it deadlock?"

**Mid-level answer:** "`notifyAll` is safer. `notify` only wakes one thread so you
might miss someone."

**Senior answer:** "`notify()` requires three preconditions simultaneously. One:
uniform waiters — every thread in the wait set is waiting for the same condition, so
waking any of them makes progress. Two: one-in, one-out — each signal enables exactly
one waiter. Three: no conditional consumption — a woken thread whose condition is true
always consumes the thing signalled, rather than waking, deciding not to proceed, and
re-parking having absorbed the signal.

The classic violation is a bounded buffer where producers and consumers wait on the
same monitor. There is one wait set, so a consumer's 'space is available' signal can
wake another consumer, which re-checks `isEmpty()`, finds it still true, and parks
again — absorbing the signal a producer needed. Now the buffer is full with producers
parked and consumers parked, throughput is exactly zero, CPU is idle, and nothing
threw. Critically, `jcmd Thread.print` will **not** report it as a deadlock, because
it is not a lock cycle — you have to recognise the shape yourself: waiters on both
sides of the same monitor address, and a depth gauge that contradicts what they are
waiting for.

`notifyAll` fixes it at the cost of a thundering herd. The real answer is
`ReentrantLock` with two `Condition`s, which gives you a separate wait set per
condition and makes `signal()` safe again — which is exactly what
`ArrayBlockingQueue` does."

**What separates them:** naming the three preconditions rather than saying "notifyAll
is safer", constructing the deadlock, and — the strongest signal — knowing that the
standard deadlock detector will not flag it.

**Follow-up:** "How would you find this in a running production system?" They want:
two thread dumps 30 seconds apart, matching monitor addresses, unchanged `cpu=` values,
and a queue-depth metric that contradicts the wait condition.

---

### Q3 — "What's the difference between `wait()` and `sleep()`?"

**Mid-level answer:** "`wait` releases the lock and `sleep` doesn't. `wait` is on
`Object`, `sleep` is on `Thread`."

**Senior answer:** "Those are the facts; the consequence is what matters. `sleep()`
holds every monitor the thread owns. So a thread that sleeps inside a `synchronized`
block guarantees that nobody can change the state it is sleeping for — that is a
deadlock you constructed by hand, and it is the reason 'just add a sleep and retry'
is not a fix for a coordination problem, it is a way of making it intermittent.

`wait()` fully releases the monitor — including nested acquisitions, restoring the
recursion count on re-entry — which is what allows another thread to acquire it and
change the condition. It also requires that you own the monitor, or you get
`IllegalMonitorStateException` immediately, whereas `sleep()` can be called anywhere.

In a thread dump they look different and you should be able to tell them apart at a
glance: `WAITING (on object monitor)` with a `- waiting on <0x…>` line versus
`TIMED_WAITING (sleeping)` with a `- locked <0x…>` line and no `waiting on`. If I see
`TIMED_WAITING (sleeping)` with a `locked` line, I know immediately that thread is
holding a lock it should not be."

**What separates them:** turning the API difference into a diagnostic skill and a
design rule, and knowing that the recursion count is preserved.

**Follow-up:** "What about `Thread.onSpinWait()`?" — a hint to the CPU that this is a
spin-wait loop (`PAUSE` on x86), for very short waits where parking costs more than
spinning. Not a substitute for a guarded block.

---

### Q4 — "We use `wait`/`notify` for our producer-consumer queue. Is that fine?"

**Mid-level answer:** "It works, but `BlockingQueue` would be simpler."

**Senior answer:** "It is almost certainly a defect waiting to happen, and I would
raise it in review rather than accept it. Three questions I would ask, in order.

First: is the wait in a `while` loop? If it is an `if`, it is broken by spec and I can
stop there. Second: are the state change and the signal in the same critical section?
If the producer adds in one `synchronized` block and notifies in another, a consumer
can slot into the gap, see the updated state, and still park — and the signal fires
into an empty wait set and evaporates. Third: is it `notify` or `notifyAll` with mixed
waiters? If it is `notify` on a shared monitor, it will stall.

Those three questions are the entire review. If it passes all three it is correct —
and I would still replace it with `ArrayBlockingQueue`, because that class is these
same 90 lines written by Doug Lea with two conditions instead of one, jcstress
coverage, and fifteen years of production. The 90 lines we wrote have none of that,
and every future maintainer has to re-derive the three questions.

The reason to *understand* it is that `BlockingQueue`, `CountDownLatch`, `Semaphore`,
HikariCP's connection borrow and `ThreadPoolExecutor`'s worker parking are all guarded
blocks. When one of them shows up in a thread dump I need to know what the wait set
means."

**What separates them:** having a concrete three-question review checklist, and
separating "understand it" from "write it".

**Follow-up:** "What does `BlockingQueue` give you that a correct `wait`/`notify`
implementation does not?" — two condition queues rather than one shared wait set,
timed and interruptible variants, `drainTo` for batching, weakly-consistent iteration,
and correctness that someone else maintains.

---

### Q5 — "A thread dump shows 40 threads in `Object.wait()`. Is that a problem?"

**Mid-level answer:** "It means they're waiting for something. Probably a bottleneck."

**Senior answer:** "By itself, no — it is the *normal* state of an idle worker pool.
Forty threads in `Object.wait()` on a `LinkedBlockingQueue`'s `notEmpty` condition just
means there is no work. Alerting on it would page you every quiet night.

What makes it a problem is a **contradiction between the wait and the state**. So I
would correlate three things. One: the monitor address — are they all on the same one,
and which class is it? Two: the corresponding depth metric — if 40 threads are waiting
for items and the queue-depth gauge says 500, that is conclusive: work exists and
nobody was woken. Three: `cpu=` across two dumps 30 seconds apart — unchanged means
genuinely parked, so it is not a livelock.

I would also separate `WAITING (on object monitor)` from `BLOCKED (on object
monitor)`, because they are different problems with different fixes. `BLOCKED` means
the entry set — that is lock contention, and JFR's `jdk.JavaMonitorEnter` events with
their duration distribution is where I would go. `WAITING` means the wait set — that
is a coordination question, and `jdk.JavaMonitorWait` with the `timedOut` field is
where I would go. Conflating the two sends you to the wrong fix."

**What separates them:** refusing to treat a thread state as a symptom on its own,
naming the correlating evidence, and cleanly separating contention from coordination.

**Follow-up:** "What would you alert on instead?" — queue saturation as a fraction of
capacity, rejection rate, and the p99 of producer wait time against the SLO. Never
raw thread state.

---

## Mental model checkpoint

Reason these out. Do not look them up.

1. `notify()` has no memory: a signal sent when the wait set is empty is discarded.
   `LockSupport.unpark()` has a one-permit memory: an unpark before the park makes the
   park return immediately. Why did the designers make these different, and what would
   break if `notify()` had a permit?

2. `wait()` restores the monitor's recursion count on re-entry, so a thread that
   entered `synchronized` three times deep comes back three deep. Why is that
   necessary rather than merely convenient? Construct the bug that would exist if it
   released one level and re-acquired one level.

3. The correct pattern requires the state change and the signal to be in the same
   critical section. But the *waiter* checks the condition and calls `wait()` in the
   same critical section too. Explain, in terms of Topic 85's monitor and Topic 86's
   happens-before edges, why those two facts together make the lost wakeup impossible
   — and identify precisely which one you would have to remove to reintroduce it.

4. You are told that on some hypothetical JVM, `wait()` provably never returns
   spuriously — the vendor guarantees it. Would you then use `if`? Give the strongest
   argument for yes, then defeat it.

5. `wait`/`notify` gives one wait set per monitor. `ReentrantLock` gives many
   conditions per lock. Java could have given `Object` multiple wait sets from the
   start. What would that have cost, given that every object in the language has a
   header and a potential monitor?

6. A colleague replaces every `notify()` in the codebase with `notifyAll()` and says
   "now it is definitely correct". Under what circumstances is that statement false —
   that is, when does `notifyAll` fail to fix a `notify` bug?

7. Your reservation buffer is correct. You switch `orderflow` to virtual threads
   (Topic 101) and the workers now block in `Object.wait()` on a virtual thread. What
   changes about the cost model, what changes about the *correctness* argument, and
   which JDK version is the hinge? Say what you would verify before deploying.

---

## Quick reference card

### The three methods

```java
obj.wait()                 // park until notify/interrupt/spurious. Must hold obj's monitor.
obj.wait(millis)           // ...or until the timeout elapses. 0 means "no timeout".
obj.wait(millis, nanos)    // finer granularity; nanos is 0..999999
obj.notify()               // wake ONE arbitrary thread from obj's wait set
obj.notifyAll()            // wake ALL threads in obj's wait set
```

All five are `final` on `Object`. You cannot override them.

### The mandatory shape

```java
// WAITER                                    // SIGNALLER
synchronized (lock) {                        synchronized (lock) {
    while (!condition()) {                       makeConditionTrue();
        lock.wait();                             lock.notifyAll();
    }                                        }
    proceed();
}
```

### Wait set vs entry set

| | Wait set | Entry set |
|---|---|---|
| How you get in | called `wait()` | tried to acquire a held monitor |
| Thread state | `WAITING` / `TIMED_WAITING (on object monitor)` | `BLOCKED (on object monitor)` |
| Dump marker | `- waiting on <0x…>` | `- waiting to lock <0x…>` |
| How you get out | notify / notifyAll / timeout / interrupt / spurious | the owner releases |
| JFR event | `jdk.JavaMonitorWait` | `jdk.JavaMonitorEnter` |
| The problem it indicates | coordination | contention |

### `notify()` is safe only if ALL THREE hold

- [ ] Uniform waiters — everyone waits for the same condition
- [ ] One-in, one-out — one signal enables exactly one waiter
- [ ] No conditional consumption — a woken thread never absorbs a signal it does not use

Otherwise `notifyAll()`, or two `Condition`s on a `ReentrantLock`.

### Diagnostic commands

```bash
jcmd -l                                              # find the pid
jcmd <pid> Thread.print > d1.txt                     # thread dump
jcmd <pid> Thread.print | grep -c 'WAITING (on object monitor)'
jcmd <pid> Thread.print | grep -oE '(waiting on|waiting to lock) <0x[0-9a-f]+>' | sort | uniq -c
jstack -l <pid>                                      # equivalent; -l adds j.u.c lock info
java -XX:StartFlightRecording=duration=120s,filename=r.jfr,settings=profile -jar app.jar
jfr print --events jdk.JavaMonitorWait r.jfr | grep -c 'timedOut = true'
unzip -p "$JAVA_HOME/lib/src.zip" java.base/java/lang/Object.java | grep -i -A6 spurious
```

### Gotchas checklist

- [ ] `while`, never `if` — broken by specification otherwise
- [ ] State change and signal in the **same** `synchronized` block
- [ ] `notifyAll()` unless you can defend all three `notify()` preconditions
- [ ] `wait()` on a `private final Object lock`, never on `this`
- [ ] Never `Thread.sleep()` inside `synchronized` while waiting for a condition
- [ ] Never swallow `InterruptedException` — propagate or restore the flag and stop
- [ ] Timed waits recompute the remaining deadline **inside** the loop, using `nanoTime`
- [ ] `wait(long)` cannot tell you why it returned — re-check the condition and your own deadline
- [ ] Guarded state touched only under the monitor needs no `volatile`
- [ ] A worker loop catches `RuntimeException` per item, or one bad item kills the worker silently
- [ ] `Thread.print` does **not** detect lost wakeups. Only a lock cycle is reported.

---

## When would I use this at work?

**1. Reading a thread dump during a "the service stopped responding" incident.**
This is the most common way the topic pays for itself, and it pays within minutes. You
see 40 threads `WAITING (on object monitor)` on one address, you check the queue-depth
gauge, you see it is non-zero, and you know within two minutes that a signal was lost
rather than that the database is slow. Without this topic you spend an hour on the
database.

**2. Reviewing someone's hand-rolled coordination code.**
Somebody writes a "simple" gate, latch or buffer because "we only need 20 lines". You
ask the three questions — `while` or `if`, same critical section, `notify` or
`notifyAll` — and you either find the bug or you recommend `ArrayBlockingQueue` /
`CountDownLatch` / `Semaphore` and explain what those give you that 20 lines cannot.
This costs you 90 seconds per review.

**3. Debugging a library, not your own code.**
A Hikari connection borrow hangs. A Kafka consumer's `poll` never returns. A Spring
context startup latch never counts down. All of these appear in a thread dump as a
guarded block in someone else's code, and reading the frame tells you which condition
is not becoming true. You cannot fix what you cannot read, and this is the topic that
makes those frames readable.

---

## Connected topics

**Prerequisites:**

- **84 — Threads vs the event loop:** the preemption fact. The scheduler can suspend a
  thread between the condition check and the `wait()` call, and that single gap is the
  lost wakeup.
- **85 — `synchronized` and monitors:** the monitor, the mark word, and lock inflation.
  `wait()` forces inflation, because a thin lock has nowhere to store a wait set.
- **86 — The JMM:** the monitor unlock→lock happens-before edge is what makes the
  guarded state visible to the woken thread with no `volatile` anywhere.
- **87 — `volatile`:** why guarded state does *not* need it, and why knowing that is a
  signal you understand the code rather than sprinkling keywords.
- **88 — Safe publication:** the snapshot object in Exercise 2 is a `final`-field
  publication; this topic gives you the coordination and Topic 88 gives you the
  visibility.
- **08 — Exceptions:** `InterruptedException` is checked for a reason. Trap 4 is
  Topic 09's swallowed exception with a thread attached.

**This unlocks:**

- **90 — `ExecutorService`:** worker threads park in a guarded block on the task
  queue. Everything in this topic is why `ThreadPoolExecutor` looks the way it does.
- **91 — `CompletableFuture`:** `join()` is a guarded block that blocks a real OS
  thread. That is the sentence that separates it from `await`.
- **92 — `ConcurrentHashMap`:** the atomic-operations-do-not-compose lesson is the
  same lesson as check-then-act around `wait()`, in a different costume.
- **93 — `BlockingQueue`:** the class you should have used. `ArrayBlockingQueue`'s
  source is this document's Example 2 with two `Condition`s. Read it side by side.
- **94 — Explicit locks:** `Condition.await`/`signal` is `wait`/`notify` with multiple
  wait sets per lock, plus `awaitNanos`, `awaitUninterruptibly` and fairness. The
  correct destination for every mixed-waiter buffer.
- **97 — Coordination primitives:** `CountDownLatch` is Example 1 with a counter.
  `Semaphore` is a permit-counting guarded block. `CyclicBarrier` is a reusable one.
- **98 — Bug taxonomy:** a lost wakeup is *not* a deadlock and `Thread.print` will not
  flag it. This is where you learn to tell the four hang shapes apart from a dump.
- **99 — jcstress:** how you would actually *prove* the buffer correct rather than
  running it 50 times and hoping.
- **101 — Virtual threads:** whether `Object.wait()` pins the carrier, and the JDK
  version at which that changed.
- **105 — Backpressure:** the bounded buffer's capacity is a backpressure policy. This
  topic gives you the mechanism; 93 gives you the number; 105 gives you the theory.
- **118 — Metrics:** the four buffer series, and why none of them may be tagged with a
  product ID.
- **121 — Health probes:** a lost-wakeup stall keeps the liveness probe green. That
  distinction is the entire point of readiness.

---

*Java baseline 21, running on JDK 25. Two things here are deliberately hedged rather
than asserted, and each has a command that settles it on your machine in under a
minute: (1) whether your JDK still pins a carrier thread when a virtual thread enters
`Object.wait()` — the behaviour changed in JDK 24 under JEP 491, so run
`java --version` and check the JFR pinned event before assuming either way; and (2) the
exact JFR default duration threshold for `jdk.JavaMonitorWait` in the `profile`
settings, which varies by JDK and which you should lower explicitly when hunting short
waits. Everything else — the mechanical statement, the two-queue structure, the three
`notify()` preconditions, and the drill — has been stable since Java 1.0 and will
still be true when you next read a thread dump at 2am.*
